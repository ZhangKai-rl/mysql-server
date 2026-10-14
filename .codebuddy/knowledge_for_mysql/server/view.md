# 视图

> **基于 MySQL 8.0.39 源码。**
>
> **为什么需要独立成篇**：视图是 SQL 层的**命名查询对象**，横跨语法、DD 持久化（`dd::View`）、权限上下文（definer / SQL SECURITY）、优化器的合并与物化四个层。目前全库只有优化器视角的 [`query/07_optimize/20_view_resolution.md`](query/07_optimize/20_view_resolution.md)（视图解析与合并算法），**"视图这个对象本身"没有宿主篇**——它的创建、可更新性判定、`WITH CHECK OPTION` 语义、与派生表的关系都散落在别处或缺失。
>
> **边界**：
> - **本篇讲**：视图的对象模型（`dd::View` 存什么）、创建与解析、**可更新视图的判定规则**、`WITH CHECK OPTION`（LOCAL / CASCADED）、视图合并 vs 物化的**决策与后果**、视图的权限上下文。
> - **不讲（指向别处）**：优化器侧的**合并/物化算法**（merge vs temptable 的实现、`VIEW_MERGE` 的限制）见 [`query/07_optimize/20_view_resolution.md`](query/07_optimize/20_view_resolution.md)；definer 与 SQL SECURITY 见 [`auth/definer.md`](auth/definer.md)；视图上的元数据锁（MDL）见 [`../infra/lock/transactional/mdl.md`](../infra/lock/transactional/mdl.md)；物化执行见 [`query/runtime/08_materialization.md`](query/runtime/08_materialization.md)。

## 目录

- [概述](#概述)
  - [是什么](#是什么)
  - [用途](#用途)
  - [版本演进](#版本演进)
- [理论基础](#理论基础)
  - [1. 为什么视图存"定义文本"而不是解析后的结构？](#1-为什么视图存定义文本而不是解析后的结构)
  - [2. 为什么可更新视图有那么多限制？](#2-为什么可更新视图有那么多限制)
  - [3. 为什么需要 WITH CHECK OPTION？](#3-为什么需要-with-check-option)
  - [4. 合并 vs 物化的权衡](#4-合并-vs-物化的权衡)
  - [5. 为什么社区没有物化视图？](#5-为什么社区没有物化视图)
- [核心实现](#核心实现)
  - [对象模型：dd::View 存什么](#对象模型ddview-存什么)
  - [创建路径：CREATE VIEW](#创建路径create-view)
  - [开表/解析：视图定义何时被重新解析](#开表解析视图定义何时被重新解析)
  - [★★★ 可更新性判定（本篇最实用）](#可更新性判定本篇最实用)
  - [CHECK OPTION 的执行](#check-option-的执行)
  - [与派生表的关系](#与派生表的关系)
- [可观测性](#可观测性)
  - [`I_S.VIEWS`](#isviews)
  - [`SHOW CREATE VIEW`](#show-create-view)
  - [EXPLAIN：怎么看出合并还是物化](#explain怎么看出合并还是物化)
- [Misc](#misc)
  - [★ 工程实现技法](#-工程实现技法)
  - [★★ 失效与退化场景](#-失效与退化场景)
  - [易混淆点](#易混淆点)
  - [一句话总结](#一句话总结)
- [参考](#参考)

---

## 概述

### 是什么

视图是**存了定义文本（而非数据）的命名查询**，在 DD 里以 `dd::View` 持久化（含定义、算法、CHECK OPTION、definer、字符集），查询时或被**合并**进外层查询块，或被**物化**成临时表。

★ 一句源码层面的定位（`sql/table.h:3062`）——**派生表就是"匿名视图"**：

```cpp
  /// Return true if this represents a named view
  bool is_view() const { return view != nullptr; }

  /// Return true if this represents a derived table (an unnamed view)
  bool is_derived() const { return derived != nullptr && view == nullptr; }

  /// Return true if this represents a named view or a derived table
  bool is_view_or_derived() const { return derived != nullptr; }
```

### 用途

逻辑抽象 / 权限隔离（只暴露部分列）/ 兼容旧 schema / 简化复杂查询。

★ 其中**权限隔离**是视图独有的——派生表做不到。源码证据见「与派生表的关系」一节：`prepare_security()` 只对视图生效，且 `definer` / `view_suid` / `with_check` 三个字段派生表恒为空。

### 版本演进

| 阶段 | 视图定义的存放 | 解析时机 |
|---|---|---|
| 5.x | `.frm` 文件 + `mysql.proc` | 开表时从 .frm 读 |
| **8.0**（本仓库 8.0.39） | **`dd::View`**（`mysql.tables` 表 + `mysql.view_table_usage` 等） | 开表时 `parse_view_definition()` 重新解析 |

**★★ 最重要的"社区没有 X"澄清：社区版至今没有物化视图。**

| 检索式 | 范围 | 命中数 | 结论 |
|---|---|---|---|
| `MATERIALIZED VIEW` / `materialized_view` / `REFRESH MATERIALIZED` | `sql/`（递归，大小写不敏感） | **8** | **全部是英文散文注释**（如 `sql/table.cc:6517` `// Base table or materialized view`），无一条是语法或实现 |
| `MATERIALIZED` | `sql/sql_yacc.yy` | **0** | **语法里没有该记号 ⇒ 不可能有 `CREATE MATERIALIZED VIEW`** |
| `MATERIALIZED` | `sql/lex.h`（关键字表） | **0** | **词法层没有该保留字** |

> ★★ 所以 8.0.39 **不存在任何物化视图基础设施**——无语法、无 `SQLCOM_*`、无 DD 表、无 refresh 机制。唯一的"物化"是**查询执行期把派生表/视图跑进临时表**，那是执行策略，不是持久对象。
>
> 社区替代品是 `CREATE TABLE ... AS SELECT`（`Query_result_create::create_table_for_query_block()`，`sql/sql_insert.cc:2835`）——**一次性快照，与源表无任何依赖或刷新关系**，需自己用定时任务 / 触发器维护。

**★ DD 版本 ≠ 服务器版本**（判断"这个行为属于哪个版本"时的常见坑）：

```cpp
/* sql/dd/dd_version.h:210 */
static const uint DD_VERSION = 80023;
static_assert(DD_VERSION <= MYSQL_VERSION_ID,
              "This release can not use a version number from the future");
```

---

## 理论基础

### 1. 为什么视图存"定义文本"而不是解析后的结构？

约束：底层表可能变更（加列、改类型），存文本才能在开表时重新解析并保持"视图跟随基表演进"；代价是**每次开表都要重新解析**。

```cpp
/* sql/sql_view.cc:1330 —— 重解析的输入就是定义文本 */
  if ((result = parser_state.init(thd, view_ref->select_stmt.str,
                                  view_ref->select_stmt.length)))
    return true;
  ...
    result = parse_sql(thd, &parser_state, view_ref->view_creation_ctx);
```

> ★ 关键：**视图没有"解析结果缓存"可复用**。每次 `open_table()` 命中 share 后都要走 `open_and_read_view() + parse_view_definition()`，除非是 PS/SP 重执行（`view_ref->is_view()` 已为真）。

而且**存的是规范化后的文本，不是用户原文**。两者分开保存：

```cpp
/* sql/sql_view.cc:919 —— 从 Item tree 反打印出两份文本 */
    lex->unit->print(thd, &view_query,
        enum_query_type(QT_TO_ARGUMENT_CHARSET | QT_HIDE_ROLLUP_FUNCTIONS));  /* 客户端字符集版 */
    lex->unit->print(thd, &is_query,
        enum_query_type(QT_TO_SYSTEM_CHARSET | QT_WITHOUT_INTRODUCERS));      /* UTF8 版，给 I_S.VIEWS 用 */
```

`SHOW CREATE VIEW` 也是反打印出来的（`sql_show.cc:2760`），不是原样吐文本：

```cpp
  table->view_query()->unit->print(
      thd, buff,
      enum_query_type(QT_TO_ARGUMENT_CHARSET | QT_NO_DEFAULT_DB |
                      QT_HIDE_ROLLUP_FUNCTIONS));
```

> ★ 所以 `CREATE VIEW v AS SELECT * FROM t1` 存进去的是 `select `t1`.`a` AS `a`, ... from `t1`` 这种展开形式。

### 2. 为什么可更新视图有那么多限制？

约束：把"对视图的 DML"翻译成"对基表的 DML"要求**行与基表行一一对应**——聚合、`DISTINCT`、`GROUP BY`、子查询、集合操作都会破坏这个映射。

★★ **所有限制几乎全部收敛到一个函数**（`sql/sql_lex.cc:3807`）：

```cpp
/**
  Return true if query expression can be merged into an outer query, based on
  technical constraints.
  Being mergeable also means that derived table/view is updatable.   /* ★★ "可合并 == 可更新"的官方表述 */

  A view/derived table is not mergeable if it is one of the following:
   - A set operation (implementation restriction).
   - An aggregated query, or has HAVING, or has DISTINCT
   - A table-less query (unimportant special case).
   - A query with a LIMIT
   - It has windows
*/
bool Query_expression::is_mergeable() const {
  if (is_set_operation()) return false;                    /* UNION / INTERSECT / EXCEPT */

  Query_block *const select = first_query_block();
  return !select->is_grouped() && select->having_cond() == nullptr &&
         !select->is_distinct() && select->has_tables() &&
         !select->has_limit() && !select->has_windows();
}
```

> ★★ **`is_grouped()` 的判定是 `group_list.elements > 0 || m_agg_func_used`**（`sql_lex.h:1434`）——即**只用了聚合函数而没写 GROUP BY 也算**，这条最容易踩。

### 3. 为什么需要 `WITH CHECK OPTION`？

约束：可更新视图可以插入"通过视图看不见的行"（插入后不满足视图的 WHERE）——CHECK OPTION 就是强制"插进去必须还看得见"。

**★ LOCAL vs CASCADED 的真正差别（源码实证，与直觉不同）**：

```cpp
/* sql/table.cc:4634 */
bool Table_ref::prepare_check_option(THD *thd, bool is_cascaded) {
  is_cascaded |= (with_check == VIEW_CHECK_CASCADED);   /* ★★ 唯一一行差别 */

  for (Table_ref *tbl = merge_underlying_list; tbl; tbl = tbl->next_local) {
    if (tbl->is_view() && tbl->prepare_check_option(thd, is_cascaded))
      return true;                                      /* ★ 把 is_cascaded 递归传下去 */
  }

  if (!check_option_processed) {
    if ((with_check || is_cascaded) &&                  /* ★★ CASCADED 时被父视图强制生成 */
        merge_join_conditions(thd, this, &check_option))
      return true;

    for (Table_ref *tbl = merge_underlying_list; tbl; tbl = tbl->next_local) {
      if (tbl->check_option &&
          !(check_option = and_conds(check_option, tbl->check_option)))
        return true;                                    /* ★★ LOCAL 也会累加上所有底层视图的条件！ */
    }
    check_option_processed = true;
  }
  ...
}
```

> ★★★ **"LOCAL 只检查本视图"是错的。** LOCAL 也会 AND 上所有**已声明** CHECK OPTION 的底层视图的条件。二者的真实差别只有一条：**CASCADED 会把 `is_cascaded=true` 向下递归传播，让"没自己声明 CHECK OPTION 的底层视图"也被强制生成条件**。
> ★ 另：**裸写 `WITH CHECK OPTION` 等价于 CASCADED**（`sql_yacc.yy:17557`）。

### 4. 合并 vs 物化的权衡

合并省掉临时表但受语法限制；物化通用但要建临时表。**★ 这个决策是优化器侧的算法（见 20 篇），本篇只给"提示不保证"的结构性证据**：

```cpp
/* sql/table.h:3131 —— 声明的算法 vs 实际生效的算法，是两个字段 */
  bool is_merged() const { return effective_algorithm == VIEW_ALGORITHM_MERGE; }
  bool uses_materialization() const {
    return effective_algorithm == VIEW_ALGORITHM_TEMPTABLE;
  }
 private:
  /// The view algorithm that is actually used, if this is a view.
  enum_view_algorithm effective_algorithm{VIEW_ALGORITHM_UNDEFINED};
```

> ★★ `algorithm`（public，DD 里那一份）与 `effective_algorithm`（private，实际生效）**分离**——这就是"提示不保证"的结构性证据。

### 5. 为什么社区没有物化视图？

见「版本演进」。物化视图需要增量刷新机制（Oracle 有 MV log、PG 有 `REFRESH MATERIALIZED VIEW`），社区选择不做——替代品是 `CREATE TABLE ... AS SELECT` + 定时重建 / 触发器维护汇总表。

> ★ 注意提纲第 3 条里"LOCAL vs CASCADED 的差别是'只检查本视图'还是'级联检查所有底层视图'"这个说法**被源码证伪**——LOCAL 也会累加上所有**已声明** CHECK OPTION 的底层视图条件。真实差别见上文。

---

## 核心实现

### 对象模型：`dd::View` 存什么

⚠️ **真名核查**：三个枚举是 `dd::View` 内部的 **unscoped enum**，且**从 1 开始计数**（与语法层的宏相差 1，靠 `dd_view.cc` 的换算函数对接）。

```cpp
/* sql/dd/types/view.h:46 */
  enum enum_check_option  // VIEW_CHECK_NONE, VIEW_CHECK_LOCAL, VIEW_CHECK_CASCADED
  { CO_NONE = 1, CO_LOCAL, CO_CASCADED };

  enum enum_algorithm  // VIEW_ALGORITHM_UNDEFINED, VIEW_ALGORITHM_TMPTABLE, MERGE
  { VA_UNDEFINED = 1, VA_TEMPORARY_TABLE, VA_MERGE };   /* ★ 是 VA_TEMPORARY_TABLE 不是 VA_TEMPTABLE */

  enum enum_security_type { ST_DEFAULT = 1, ST_INVOKER, ST_DEFINER };
```

**★ `dd::View` 的完整字段**（含注释）：

```cpp
  virtual Object_id client_collation_id() const = 0;      /* CHARACTER_SET_CLIENT，存该 charset 的默认 collation id */
  virtual Object_id connection_collation_id() const = 0;  /* COLLATION_CONNECTION */

  virtual const String_type &definition() const = 0;      /* ★ 定义文本（客户端字符集） */
  virtual const String_type &definition_utf8() const = 0; /* 同一定义的 UTF8 版，供 I_S.VIEWS */

  virtual enum_check_option check_option() const = 0;
  virtual bool is_updatable() const = 0;                  /* ★ 真名 is_updatable()，不是 updatable() */
  virtual void set_updatable(bool updatable) = 0;
  virtual enum_algorithm algorithm() const = 0;
  virtual enum_security_type security_type() const = 0;

  virtual const String_type &definer_user() const = 0;    /* ★ definer 拆成 user/host 两个，没有合并的 definer() */
  virtual const String_type &definer_host() const = 0;

  /**
    Explicit list of column names.
    It is a dictionary of string=>string, where the key is the number of the
    column ("1", "2", etc) and the value is the column's name.
  */
  virtual const Properties &column_names() const = 0;     /* CREATE VIEW v(a,b) 的显式列名 */

  virtual View_table *add_table() = 0;                    /* 视图依赖的基表 */
  virtual View_routine *add_routine() = 0;                /* 视图直接调用的存储函数 */
  View *clone() const override = 0;                       /* ★ TABLE_SHARE::view_object 就是这里 clone 的 */
```

**★ DD 物理列**（`sql/dd/impl/tables/tables.cc:130`）——所谓"typelib"在 8.0 只剩 DD 列的 SQL `ENUM`：

```cpp
  m_target_def.add_field(FIELD_VIEW_DEFINITION, ...,
                         "view_definition LONGBLOB");
  m_target_def.add_field(FIELD_VIEW_DEFINITION_UTF8, ...,
                         "view_definition_utf8 LONGTEXT");
  m_target_def.add_field(FIELD_VIEW_CHECK_OPTION, ...,
                         "view_check_option ENUM('NONE', 'LOCAL', 'CASCADED')");
  m_target_def.add_field(FIELD_VIEW_IS_UPDATABLE, ...,
                         "view_is_updatable ENUM('NO', 'YES')");
  m_target_def.add_field(FIELD_VIEW_ALGORITHM, ...,
                         "view_algorithm ENUM('UNDEFINED', 'TEMPTABLE', 'MERGE')");
  m_target_def.add_field(FIELD_VIEW_SECURITY_TYPE, ...,
                         "view_security_type ENUM('DEFAULT', 'INVOKER', 'DEFINER')");
  m_target_def.add_field(FIELD_VIEW_DEFINER, ..., "view_definer VARCHAR(288)");
  m_target_def.add_field(FIELD_VIEW_CLIENT_COLLATION_ID, ...,
                         "view_client_collation_id BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_VIEW_CONNECTION_COLLATION_ID, ...,
                         "view_connection_collation_id BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_VIEW_COLUMN_NAMES, ..., "view_column_names LONGTEXT");
```

**★ 语法层的三个"枚举"**（⚠️ **8.0.39 不存在 `view_algorithm`/`view_check_option`/`view_suid` 这三个 typelib**）：

```cpp
/* sql/table.h:2476 */
enum enum_view_algorithm {
  VIEW_ALGORITHM_UNDEFINED = 0, VIEW_ALGORITHM_TEMPTABLE = 1, VIEW_ALGORITHM_MERGE = 2
};
#define VIEW_SUID_INVOKER 0      /* ★ suid 是三个 #define，不是 enum */
#define VIEW_SUID_DEFINER 1
#define VIEW_SUID_DEFAULT 2
/* view WITH CHECK OPTION parameter options */
#define VIEW_CHECK_NONE 0        /* ★ check option 也是宏 */
#define VIEW_CHECK_LOCAL 1
#define VIEW_CHECK_CASCADED 2
/* result of view WITH CHECK OPTION parameter check */
#define VIEW_CHECK_OK 0          /* ★ 这是"检查结果"，别与上面三个混 */
#define VIEW_CHECK_ERROR 1
#define VIEW_CHECK_SKIP 2
```

**★ 两套枚举的换算**（`sql/dd/dd_view.cc:105`，共 6 个函数）：`dd_get_new_view_check_type()` / `dd_get_old_view_check_type()` / `dd_get_new_view_algorithm_type()` / `dd_get_old_view_algorithm_type()` / `dd_get_new_view_security_type()` / `dd_get_old_view_security_type()`。

### 创建路径：`CREATE VIEW`

**⚠️ 不存在 `Sql_cmd_create_view` 类，也不存在 `PT_create_view`**（全仓 0 命中）。CREATE 与 ALTER 共用 `sql_parse.cc` 的一个 case：

```cpp
/* sql/sql_parse.cc:4601 */
    case SQLCOM_CREATE_VIEW: {
      /*
        Note: SQLCOM_CREATE_VIEW also handles 'ALTER VIEW' commands
        as specified through the thd->lex->create_view_mode flag.
      */
      res = mysql_create_view(thd, first_table, thd->lex->create_view_mode);
      break;
    }
```

**★ 入口 `mysql_create_view()` 的骨架**（`sql/sql_view.cc:413`，★ Doxygen 自陈原子性）：

```cpp
/**
  Atomicity:
    The operation to create, alter and create_or_replace a view is
    atomic/crash-safe. ... In case of crash, there won't be any discrepancy
    between the data-dictionary table and the binary log.
*/
bool mysql_create_view(THD *thd, Table_ref *views, enum_view_create_mode mode) {
  Table_ref *view = lex->unlink_first_table(&link_to_local);  /* ★ 把视图名从表链摘出来，剩下的才是基表 */
  ...
  if (thd->locked_tables_mode) { my_error(ER_LOCK_OR_ACTIVE_TRANSACTION, MYF(0)); ... }
  if (create_view_precheck(thd, tables, view, mode)) goto err;
  ...
  if (open_tables_for_query(thd, lex->query_tables, 0)) goto err;
  ...
  /* ★ 关键：上下文分析模式，不真正执行 */
  lex->context_analysis_only |= CONTEXT_ANALYSIS_ONLY_VIEW;
  if (!unit->is_prepared()) {
    if (unit->prepare(thd, nullptr, nullptr, 0, 0)) goto err;  /* ★ 名字解析/列类型推导在这里 */
    make_valid_column_names(lex);
    if (check_duplicate_names(view->derived_column_names(), query_block->fields, true)) goto err;
    if (query_block->fields.size() > MAX_FIELDS) { my_error(ER_TOO_MANY_FIELDS, MYF(0)); goto err; }
  }
  ...
  if ((res = mysql_register_view(thd, view, mode))) goto err_with_rollback;  /* ★ 落 DD */
  tdc_remove_table(thd, TDC_RT_REMOVE_ALL, view->db, view->table_name, false); /* ★ 踢掉表定义缓存 */
  ...
  res = trans_commit_stmt(thd) || trans_commit(thd);
err_with_rollback:
  trans_rollback_stmt(thd);
  trans_rollback(thd);          /* ★ 语句级 + 全事务级双回滚（DD 缓存与磁盘同步） */
```

**★ 权限检查 `create_view_precheck()`**（`sql/sql_view.cc:317`）——它自己列的规则就是最好的注释：

```cpp
  /*
    Privilege check for view creation:
    - user has CREATE VIEW privilege on view table
    - user has DROP privilege in case of ALTER VIEW or CREATE OR REPLACE VIEW
    - user has some (SELECT/UPDATE/INSERT/DELETE) privileges on columns of
      underlying tables ...
    - for columns of underly tables used on top of SELECT list also will be
      checked that we have not more privileges on correspondent column of view
      table (i.e. user will not get some privileges by view creation)
  */
  if (!is_infoschema_db(view->db)) {
    if ((check_access(thd, CREATE_VIEW_ACL, view->db, ...) ||
         check_grant(thd, CREATE_VIEW_ACL, view, false, 1, false)) ||
        (mode != enum_view_create_mode::VIEW_CREATE_NEW &&
         (check_access(thd, DROP_ACL, view->db, ...) ||
          check_grant(thd, DROP_ACL, view, false, 1, false))))
      goto err;
  }
  ...
    if (check_some_access(thd, VIEW_ANY_ACL, tbl)) {   /* ★ VIEW_ANY_ACL = SELECT|UPDATE|INSERT|DELETE */
      my_error(ER_TABLEACCESS_DENIED_ERROR, MYF(0), "ANY", ...);
```

**★ 逐列取交集：视图列权限 = 基表列权限的交集**（★ 这是"不能通过建视图获得额外权限"的实现）：

```cpp
    Access_bitmask final_priv = VIEW_ANY_ACL;
    for (sl = query_block; sl; sl = sl->next_query_block()) {
      for (Item *item : sl->visible_fields()) {
        Item_field *fld = item->field_for_view_update();
        Access_bitmask priv = (get_column_grant(thd, &view->grant, view->db,
                                                view->table_name, item->item_name.ptr()) &
                               VIEW_ANY_ACL);
        if (fld && !fld->field->table->s->tmp_table) {
          final_priv &= fld->have_privileges;          /* ★★ 交集 */
          if (~fld->have_privileges & priv) report_item = item;
        }
      }
    }
```

**★★ `mysql_register_view()`：生成定义文本 + 推导属性 + 落 DD**（`sql/sql_view.cc:847`）

```cpp
  /* ★ 不可合并时，用户写的 MERGE 被悄悄改成 UNDEFINED —— 只告警不报错 */
  if (lex->create_view_algorithm == VIEW_ALGORITHM_MERGE && !can_be_merged) {
    push_warning(thd, Sql_condition::SL_WARNING, ER_WARN_VIEW_MERGE,
                 ER_THD(thd, ER_WARN_VIEW_MERGE));
    lex->create_view_algorithm = VIEW_ALGORITHM_UNDEFINED;
  }
  view->algorithm = lex->create_view_algorithm;
  view->definer.user = lex->definer->user;
  view->definer.host = lex->definer->host;
  view->view_suid = lex->create_view_suid;
  view->with_check = lex->create_view_check;

  view->updatable_view = is_updatable_view(thd, view);   /* ★★ 可更新性在这里现算，然后才写进 DD */

  view->view_creation_ctx = View_creation_ctx::create(thd);  /* ★ 视图列 charset/collation 的来源 */

  /* ★ 不可更新视图不能带 CHECK OPTION */
  if (view->with_check != VIEW_CHECK_NONE && !view->updatable_view) {
    my_error(ER_VIEW_NONUPD_CHECK, MYF(0), view->db, view->table_name);
    return true;
  }
  ...
  return dd::create_view(thd, *schema, view);           /* ★ 落库 */
```

**★ 写 DD 的现场**（`sql/dd/dd_view.cc:590`）：

```cpp
bool create_view(THD *thd, const dd::Schema &schema, Table_ref *view) {
  std::unique_ptr<dd::View> view_obj;
  if (dd::get_dictionary()->is_system_view_name(view->db, view->table_name, &hidden_system_view)) {
    view_obj.reset(schema.create_system_view(thd));      /* I_S 系统视图 */
    if (hidden_system_view)
      view_obj->set_hidden(dd::Abstract_table::HT_HIDDEN_SYSTEM);
  } else
    view_obj.reset(schema.create_view(thd));             /* ★ 普通视图 */
  if (fill_dd_view_definition(thd, view_obj.get(), view)) return true;
  return thd->dd_client()->store(view_obj.get());        /* ★ store() 落库 */
}

bool update_view(THD *thd, dd::View *new_view, Table_ref *view) {
  new_view->remove_children();                           /* ★ ALTER：先清 columns/tables/routines 三张子集合 */
  new_view->set_last_altered(...);
  if (fill_dd_view_definition(thd, new_view, view)) return true;
  return thd->dd_client()->update(new_view);
}
```

**★ 列类型推导**（`sql/dd/dd_view.cc:221`，★ 列的 collation 来自 `item->collation.collation`）：

```cpp
  for (Item *item : VisibleFields(*(thd->lex->unit->get_unit_column_types()))) {
    /* ★ 遍历的是 unit->get_unit_column_types()：单 QB 就是其可见列，UNION 时是各分支合并后的类型占位 item */
    ...
          tmp_field = new (thd->mem_root) Field_blob(
              item->max_length, item->is_nullable(), item->item_name.ptr(),
              item->collation.collation, true);      /* ★★ 列的 collation */
    ...
    Create_field *cr_field = new (thd->mem_root) Create_field(tmp_field, orig_field);
    ...
  return fill_dd_columns_from_create_fields(thd, view_obj, create_fields, file);
```

### 开表/解析：视图定义何时被重新解析

**⚠️ 不存在 `TABLE_SHARE::view_definition`**（8.0 已移除），也**不存在 `TABLE_SHARE::parse_view_definition`**。定义文本在 **`Table_ref::select_stmt`**；`TABLE_SHARE` 只剩 `bool is_view` + `const dd::View *view_object`（一份 clone）。

```cpp
/* sql/table.h:3689 */
  LEX_STRING select_stmt{nullptr, 0};  ///< text of (CREATE/SELECT) statement
  LEX_STRING source{nullptr, 0};       ///< source of CREATE VIEW（用户原始文本，binlog 用）
  LEX_USER definer;                    ///< definer of view
  /**
    @note: This field is currently not reliable when read from dictionary:
    If an underlying view is changed, updatable_view is not changed,
    due to lack of dependency checking in dictionary implementation.
    Prefer to use is_updatable() during preparation and optimization.
  */
  ulonglong updatable_view{0};         ///< VIEW can be updated
```

**★★ `open_table()` 对视图的特殊分支**（`sql/sql_base.cc:3244`，★ 这就是"何时重新解析"的答案）：

```cpp
  if (table_list->is_view() || share->is_view) {
    /*
      Note that there is no need to call TABLE_SHARE::has_old_version() as we
      do for regular tables, because view shares are always up to date.
    */
    if (table_list->parent_l) my_error(ER_WRONG_MRG_TABLE, MYF(0));
    else if (check_and_update_table_version(thd, table_list, share)) ;
    else if (table_list->open_strategy == Table_ref::OPEN_FOR_CREATE) {
      /* CREATE TABLE 与已有视图同名 → 跳过读定义，只建占位 LEX */
      if (!table_list->is_view()) return add_view_place_holder(thd, table_list);
      return false;
    } else {
      view_open_result = open_and_read_view(thd, share, table_list);   /* ① 读属性 */
    }
    release_table_share(share);
    mysql_mutex_unlock(&LOCK_open);
    if (view_open_result) return true;
    if (parse_view_definition(thd, table_list)) return true;           /* ② ★★ 真正重新解析 */
    assert(table_list->is_view());
    return false;      /* ★★ 视图不创建 TABLE 对象，直接返回 */
  }
```

**★ `dd::View` → `Table_ref` 的搬运**（`open_and_read_view()`，`sql_view.cc:1081`）：

```cpp
  if (view_ref->is_view()) {
    /* ★ PS/SP 重执行短路：已有 LEX 就不重读不重解析 */
    return false;
  }
  /* 检查循环引用 */
  for (Table_ref *precedent = view_ref->referencing_view; precedent;
       precedent = precedent->referencing_view) {
    if (... my_strcasecmp(system_charset_info, precedent->table_name, view_ref->table_name) == 0 ...) {
      my_error(ER_VIEW_RECURSIVE, MYF(0), top_view->db, top_view->table_name);
      return true;
    }
  }
  assert(share->view_object);
  if (dd::read_view(view_ref, *share->view_object, thd->mem_root)) return true;
  view_ref->view_creation_ctx = View_creation_ctx::create(thd, view_ref);  /* ★ 重建解析上下文 */
```

**★★★ `parse_view_definition()` 的核心**（`sql/sql_view.cc:1194`，★ 自由函数，不是成员）：

```cpp
  // A parsed view requires its own LEX object
  LEX *const old_lex = thd->lex;
  LEX *const view_lex = (LEX *)new (thd->mem_root) st_lex_local;   /* ★★ 每个视图一个独立 LEX */
  thd->lex = view_lex;
  view_ref->set_view_query(view_lex);                              /* ★★ 从此 is_view() 返回 true */

  if ((result = parser_state.init(thd, view_ref->select_stmt.str,
                                  view_ref->select_stmt.length))) return true;
  /* 临时把视图所在库当默认库，保证未限定名的表解析正确 */
  thd->reset_db({view_ref->db, view_ref->db_length});
  ...
    result = parse_sql(thd, &parser_state, view_ref->view_creation_ctx);  /* ★★★ 真正重解析 */
  ...
  /* 权限不足 → 不能 EXPLAIN */
    if (check_table_access(thd, SELECT_ACL, view_tables, false, UINT_MAX, true) ||
        check_table_access(thd, SHOW_VIEW_ACL, &view_no_suid, false, UINT_MAX, true))
      view_ref->view_no_explain = true;
  ...
  /* ★ SUID 切换现场 */
  if (view_ref->view_suid) {
    view_ref->view_sctx = new (thd->stmt_arena->mem_root) Security_context();  /* ★ 8.0 是 per-Table_ref，不是 THD 全局 */
    security_ctx = view_ref->view_sctx;
  } else {
    security_ctx = view_ref->security_ctx;                                     /* INVOKER：继承外层 */
  }
  for (Table_ref *tbl = view_tables; ...; tbl = tbl->next_global)
    tbl->security_ctx = security_ctx;                                          /* ★ 基表的 ctx 被换成 definer 的 */

  view_query_block->linkage = DERIVED_TABLE_TYPE;        /* ★ 视图的第一个 QB 被标成 DERIVED，与派生表合流 */

  // Updatability is not decided yet
  assert(!view_ref->is_updatable());                     /* ★★ 此刻还没定，要等 merge_derived() */

  view_ref->set_derived_query_expression(view_lex->unit); /* ★ 视图 LEX 的 unit 挂到 Table_ref::derived */
```

> ★★ **`view_ref->set_view_query(view_lex)` 这一行是"它从此是视图"的唯一开关**；而 `set_derived_query_expression()` 让视图在优化器侧与派生表完全同构。

**★ 表缓存与重新解析的关系**：视图**没有"解析结果缓存"**。缓存层面只有两处作用：

```cpp
/* sql/sql_view.cc:693 —— CREATE/ALTER VIEW 之后主动踢掉 share */
  /*
    View TABLE_SHARE must be removed from the table definition cache in order
    to make ALTER VIEW work properly. Otherwise, we would not be able to
    detect meta-data changes after ALTER VIEW.
  */
  tdc_remove_table(thd, TDC_RT_REMOVE_ALL, view->db, view->table_name, false);
```

> ★ 踢掉 share ⇒ 下次开表必然重新 `get_table_share()` → 重新 `clone()` `dd::View` → 重新 `parse_view_definition()`。

**★ definer 上下文的装载与查找**（`sql/table.cc`）：

```cpp
bool Table_ref::prepare_view_security_context(THD *thd) {
  if (view_suid) {
    if (acl_getroot(thd, view_sctx, definer.user.str, definer.host.str,
                    definer.host.str, thd->db().str)) {
      /* SHOW CREATE / SHOW FIELDS 对"definer 用户已不存在"降级为 warning，保证仍能看定义 */
      ...
    }
  }
}

Security_context *Table_ref::find_view_security_context(THD *thd) {
  Table_ref *upper_view = this;
  while (upper_view && !upper_view->view_suid) {        /* ★★ 非 SUID 视图一路往上爬 */
    upper_view = upper_view->referencing_view;
  }
  if (upper_view) return upper_view->view_sctx;         /* 用最近的 SUID 祖先的 definer */
  return thd->security_context();                       /* 整条链都没有 SUID → 用当前调用者 */
}
```

> ⚠️ 不存在 `THD::view_security_context`（那是 5.7）。8.0 是 **`Table_ref::view_sctx`** + `prepare_view_security_context()` / `find_view_security_context()` / `prepare_security()`。

### ★★★ 可更新性判定（本篇最实用）

**⚠️ 先纠正三处常见误解**：

| 说法 | 真相 |
|---|---|
| `Table_ref::is_updatable()` 是判定函数 | **纯 getter**，零计算（`sql/table.h:3154`） |
| `check_view_insertability()` 是总判定 | 是 **`static`**，只校验 INSERT 时"列能否写回基表"（`sql_insert.cc:808`） |
| `check_single_table()` 判定可更新性 | 只做"待改列是否落在同一张基表"（`sql/table.cc:4759`） |

```cpp
/* sql/table.h:3153 */
  bool is_updatable() const { return m_updatable; }        /* ★ 纯 getter */
  void set_updatable() { m_updatable = true; }             /* ★ 默认 false：不显式 set 就不可更新 */
```

**★★ 真正的判定：`is_updatable_view()`**（`sql/sql_view.cc:785`，★ CREATE 期算一次）：

```cpp
bool is_updatable_view(THD *thd, Table_ref *view) {
  bool can_be_merged =
      lex->unit->is_mergeable() && view->algorithm != VIEW_ALGORITHM_TEMPTABLE;  /* ① ② */

  if (!dd::get_dictionary()->is_system_view_name(view->db, view->table_name) &&  /* ③ 系统视图一律不可更新 */
      (updatable_view = can_be_merged)) {
    bool updatable = false;
    bool outer_joined = false;
    for (Table_ref *tbl = lex->query_block->get_table_list(); tbl; tbl = tbl->next_local) {
      updatable |= !((tbl->is_view() && !tbl->updatable_view) || tbl->schema_table);  /* ④ */
      outer_joined |= tbl->is_inner_table_of_outer_join();                            /* ⑤ */
    }
    updatable &= !outer_joined;

    if (updatable) {
      bool view_has_updatable_column = false;
      for (Item *item : lex->query_block->visible_fields()) {
        Item_field *item_field = item->field_for_view_update();                       /* ⑥ */
        if (item_field && !item_field->table_ref->schema_table) {
          view_has_updatable_column = true;
          break;
        }
      }
      updatable &= view_has_updatable_column;                                          /* ★ 至少要有一列能映射回基表列 */
    }
    if (!updatable) updatable_view = false;
  }

  /* ⑦ 主表同时出现在子查询里（只抓最简单场景） */
  if (updatable_view && !...is_set_operation() &&
      !(lex->query_block->get_table_list())->next_local &&
      find_table_in_global_list(lex->query_tables->next_global,
                                lex->query_tables->db, lex->query_tables->table_name)) {
    updatable_view = false;
  }
  return updatable_view;
}
```

**★ 逐项核实："使视图不可更新"的完整清单**

| 条件 | 机制 | 源码 |
|---|---|---|
| 聚合函数（含无 GROUP BY 的隐式聚合） | `is_mergeable()==false` | `sql_lex.cc:3811` + `sql_lex.h:1434`（`group_list.elements > 0 \|\| m_agg_func_used`） |
| `DISTINCT` | 同上 | `sql_lex.cc:3812` |
| `GROUP BY` | 同上 | 同上 |
| `HAVING` | 同上 | `select->having_cond() == nullptr` |
| 集合操作 UNION/INTERSECT/EXCEPT | 同上 | `sql_lex.cc:3808` |
| 视图体内的 `LIMIT` | 同上 | `!select->has_limit()` |
| 窗口函数 | 同上 | `!select->has_windows()` |
| `ALGORITHM=TEMPTABLE` | 同上 | `sql_view.cc:794` |
| 无表查询（`SELECT 1`） | 同上 | `select->has_tables()` |
| 表达式列（`a+1`、函数、常量） | `field_for_view_update()==nullptr` | `sql_view.cc:813` |
| 底层是 I_S / schema 表 | `tbl->schema_table` | `sql_view.cc:804/815` |
| 底层视图自身不可更新 | `(tbl->is_view() && !tbl->updatable_view)` | `sql_view.cc:804` |
| 底层表是外连接内表 | `updatable &= !outer_joined` | `sql_view.cc:805-807` |
| **多表 FROM（join）** | ⚠️ **不是**"不可更新"，而是**可更新但一次只能改一张基表** | `ER_VIEW_MULTIUPDATE`（`sql_update.cc:1745`）、`ER_VIEW_DELETE_MERGE_VIEW`（`sql_delete.cc:743`） |
| **`FROM` 里有派生表/CTE** | ⚠️ **无显式检查**——由"只有视图才 `set_updatable()`"兜底 | 见下 |

**★ prepare 期重算 `m_updatable`**（`sql/sql_resolver.cc:3537`，★ **只有视图走这段——这是"派生表不可被 DML"的最终落点**）：

```cpp
  /**
    A view is updatable if any underlying table is updatable.        /* ★ updatable 用 OR */
    A view is insertable-into if all underlying tables are insertable.  /* ★ insertable 用 AND */
    A view is not updatable nor insertable if it contains an outer join
  */
  if (derived_table->is_view()) {                                    /* ★★ 派生表永远进不来 */
    bool updatable = false, insertable = true, outer_joined = false;
    for (Table_ref *tr = derived_table->merge_underlying_list; tr; tr = tr->next_local) {
      updatable |= tr->is_updatable();
      insertable &= tr->is_insertable();
      outer_joined |= tr->is_inner_table_of_outer_join();
    }
    updatable &= !outer_joined;
    insertable &= !outer_joined;
    if (updatable) derived_table->set_updatable();
    if (insertable) derived_table->set_insertable();
  }
```

基表侧无条件置位（`sql/sql_base.cc:3468`）：`table_list->set_updatable(); table_list->set_insertable();`

**★ `updatable` 的三处落点**：

| 落点 | 字段/方法 | 语义 |
|---|---|---|
| 运行时（权威） | `Table_ref::m_updatable` + `is_updatable()` | prepare/optimize 期现算 |
| CREATE 期快照（**会过期**） | `Table_ref::updatable_view` | 写进 DD，注释明说"读自字典不可靠" |
| DD 持久 | `dd::View::is_updatable()` | `mysql.tables.view_is_updatable ENUM('NO','YES')` |

**★ 物化侧的第二道锁**（`sql/sql_derived.cc:820`）：

```cpp
  set_uses_materialization();
  // From resolver POV, columns of this table are readonly
  set_readonly();     /* ★ m_updatable = false; m_insertable = false; */
```

> ★ 所以 **`ALGORITHM=TEMPTABLE` 的视图不可更新**，运行时由 `set_readonly()` 二次保证。

### CHECK OPTION 的执行

**★ 语法层**（`sql/sql_yacc.yy:17555`，★ **裸写 `WITH CHECK OPTION` = CASCADED**）：

```cpp
view_check_option:
          %empty                          { $$= VIEW_CHECK_NONE; }
        | WITH CHECK_SYM OPTION           { $$= VIEW_CHECK_CASCADED; }   /* ★ 裸写 = CASCADED */
        | WITH CASCADED CHECK_SYM OPTION  { $$= VIEW_CHECK_CASCADED; }
        | WITH LOCAL_SYM CHECK_SYM OPTION { $$= VIEW_CHECK_LOCAL; }
        ;
```

落点 `Lex->create_view_check= $2;`（`sql_yacc.yy:17537`）→ `Table_ref::with_check` → `dd::View::check_option`。

**★ 条件树的构造**（`prepare_check_option()` 已在理论基础第 3 问给出全文）。补充 `merge_join_conditions()`——它把**视图的 WHERE + JOIN ON** 拷出来当检查条件：

```cpp
static bool merge_join_conditions(THD *thd, Table_ref *table, Item **pcond) {
  *pcond = nullptr;
  if (table->join_cond()) {
    if (!(*pcond = table->join_cond()->copy_andor_structure(thd))) return true;
  }
  if (!table->nested_join) return false;
  for (Table_ref *tbl : table->nested_join->m_tables) {
    if (tbl->is_view()) continue;                 /* 视图单独走 and_conds */
    Item *cond;
    if (merge_join_conditions(thd, tbl, &cond)) return true;
    if (cond && !(*pcond = and_conds(*pcond, cond))) return true;
  }
  return false;
}
```

**★ 执行期逐行校验**（`sql/table.cc:4726`）：

```cpp
int Table_ref::view_check_option(THD *thd) const {
  if (check_option && check_option->val_int() == 0) {          /* ★ 条件求值为假 */
    const Table_ref *main_view = top_table();
    my_error(ER_VIEW_CHECK_FAILED, MYF(0), main_view->db, main_view->table_name);
    if (thd->lex->is_ignore()) return (VIEW_CHECK_SKIP);       /* ★ IGNORE → 跳过本行继续 */
    return (VIEW_CHECK_ERROR);
  }
  return (VIEW_CHECK_OK);
}
```

**★ 全部调用点**：

| 位置 | 文件:行 |
|---|---|
| INSERT ... VALUES 主循环 | `sql_insert.cc:615` |
| INSERT 准备期（**只在视图被合并时**） | `sql_insert.cc:1222` |
| INSERT ... ON DUPLICATE KEY UPDATE | `sql_insert.cc:1986` |
| INSERT ... SELECT 的 `write_record` 前 | `sql_insert.cc:2364` |
| 单表 UPDATE | `sql_update.cc:920` |
| 多表 UPDATE（iterator） | `sql_update.cc:2464`（另有 pre-iterator `:2761`） |
| LOAD DATA | `sql_load.cc:278/856` |

> ★ **DELETE 没有 `view_check_option()` 调用**（已核实）——被删行必然已在视图可见集内。
> ★ `sql_insert.cc:1222` 的 `if (table_list->is_merged())` 意味着**物化视图根本不生成 check_option**——间接证明了不可更新。

**★ 相关错误码**：`ER_VIEW_CHECK_FAILED`、`ER_VIEW_NONUPD_CHECK`（不可更新视图带 CHECK OPTION）、`ER_NON_UPDATABLE_TABLE`、`ER_NON_INSERTABLE_TABLE`（INSERT 报的是这个）、`ER_VIEW_MULTIUPDATE`、`ER_VIEW_DELETE_MERGE_VIEW`、`ER_WARN_VIEW_MERGE`、`ER_WARN_VIEW_WITHOUT_KEY`。

### 与派生表的关系

**★ 判定三件套**（`sql/table.h:3061`，已在"是什么"给出）。核心：**`is_derived() == derived != nullptr && view == nullptr`**——派生表就是"匿名视图"。

**★★ 本质区别（源码证据）**：

**(a) 视图专属字段，派生表恒空**（`sql/table.h:3663`）：`security_ctx`、`view_sctx`、`definer`、`algorithm`、`view_suid`、`with_check`、`updatable_view`、`select_stmt`。

**(b) 权限上下文只有视图会构造**（`sql_view.cc:1580`）：

```cpp
    view_ref->view_sctx = new (thd->stmt_arena->mem_root) Security_context();  /* ★ 只有视图会 new */
    ...
      tbl->security_ctx = security_ctx;                 /* 传播到它引用的每张底层表 */
      sl->context.security_ctx = security_ctx;          /* 连名字解析上下文都换掉 */
```

**(c) `prepare_security()` 显式区分派生表**（`sql/table.cc:4938`）：

```cpp
bool Table_ref::prepare_security(THD *thd) {
  Security_context *save_security_ctx = thd->security_context();
  if (prepare_view_security_context(thd)) return true;
  thd->set_security_context(find_view_security_context(thd));   /* ★ 切换成 definer/invoker */
  for (Table_ref *tbl : *view_tables) {
    if (tbl->is_derived()) {
      tbl->grant.privilege = SELECT_ACL;                        /* ★★ 派生表只有 SELECT_ACL，无需查 ACL */
      continue;
    }
    fill_effective_table_privileges(thd, &tbl->grant, tbl->db, tbl->get_table_name());
  }
  thd->set_security_context(save_security_ctx);
```

**(d) DD 持久化只有视图有**（`dd_view.cc:602` `thd->dd_client()->store(view_obj.get())`）——**派生表没有任何 DD 落盘路径**。

**(e) 运行时 `set_updatable()` 只发给视图**（`sql_resolver.cc:3546` 的 `if (derived_table->is_view())`）。

> ★★★ 一句话：**视图 = 有名字 + 有 DD 持久化 + 有 definer/权限上下文的派生表**。优化器侧二者完全同构（都走 `merge_derived` / `setup_materialized_derived`），差别全在 SQL 层与 DD 层。

---

## 可观测性

### `I_S.VIEWS`

⚠️ 8.0 里 `I_S.VIEWS` **不再是 `sql_show.cc` 手填**，而是 **DD 系统视图**，定义在 `sql/dd/impl/system_views/views.cc:38`：

```cpp
  m_target_def.add_field(FIELD_VIEW_DEFINITION, "VIEW_DEFINITION",
      "IF(CAN_ACCESS_VIEW(sch.name, vw.name, vw.view_definer, vw.options)=TRUE,"
      "   vw.view_definition_utf8, '')");           /* ★ 没权限就返回空串，不报错 */
  m_target_def.add_field(FIELD_CHECK_OPTION, "CHECK_OPTION", "vw.view_check_option");
  m_target_def.add_field(FIELD_IS_UPDATABLE, "IS_UPDATABLE", "vw.view_is_updatable");
  m_target_def.add_field(FIELD_DEFINER, "DEFINER", "vw.view_definer");
  m_target_def.add_field(FIELD_SECURITY_TYPE, "SECURITY_TYPE",
      "IF (vw.view_security_type='DEFAULT', 'DEFINER', vw.view_security_type)");  /* ★ DEFAULT 归一化成 DEFINER */
  ...
  m_target_def.add_from("mysql.tables vw");
  m_target_def.add_where("CAN_ACCESS_TABLE(sch.name, vw.name)");
  m_target_def.add_where("AND vw.type = 'VIEW'");
```

**★ 四列的数据来源**：

| 列 | SQL 表达式 | 底层 DD 列 | 何时写入 |
|---|---|---|---|
| `CHECK_OPTION` | `vw.view_check_option` | `ENUM('NONE','LOCAL','CASCADED')` | CREATE/ALTER |
| `IS_UPDATABLE` | `vw.view_is_updatable` | `ENUM('NO','YES')` | **CREATE 期算一次** |
| `DEFINER` | `vw.view_definer` | `VARCHAR(288)` | `sql_view.cc:948-949` |
| `SECURITY_TYPE` | 归一化后的 `view_security_type` | `ENUM('DEFAULT','INVOKER','DEFINER')` | `sql_view.cc:950` |

**★★★ `IS_UPDATABLE` 是 DD 字段，不是运行时计算——而且它会过期**

```cpp
/* sql/table.h:3697 —— 官方警告 */
    @note: This field is currently not reliable when read from dictionary:
    If an underlying view is changed, updatable_view is not changed,
    due to lack of dependency checking in dictionary implementation.
    Prefer to use is_updatable() during preparation and optimization.
```

> ★★★ 即**底层视图变更后 `I_S.VIEWS.IS_UPDATABLE` 不会重算**。真正权威的是运行时 `Table_ref::is_updatable()`。这是排查"视图明明该可更新却说 NO"的第一嫌疑。

### `SHOW CREATE VIEW`

`PT_show_create_view`（`parse_tree_nodes.h:3302`）→ `sql_show.cc:1181`：

```cpp
  if (thd->lex->only_view && !table_list->is_view()) {
    my_error(ER_WRONG_OBJECT, MYF(0), table_list->db, table_list->table_name, "VIEW");
    goto exit;
  }
  if (table_list->is_view())
    buffer.set_charset(table_list->view_creation_ctx->get_client_cs());  /* ★ 用视图创建时的 client charset */
  if (table_list->is_view())
    view_store_create_info(thd, table_list, &buffer);
```

**★ 拼装**（`sql_show.cc:2714`，★ 算法段打印的是**声明的** algorithm，不是实际生效的）：

```cpp
static void append_algorithm(Table_ref *table, String *buff) {
  buff->append(STRING_WITH_LEN("ALGORITHM="));
  switch ((int8)table->algorithm) {        /* ★★ 是 algorithm，不是 effective_algorithm */
    case VIEW_ALGORITHM_UNDEFINED: buff->append(STRING_WITH_LEN("UNDEFINED ")); break;
    case VIEW_ALGORITHM_TEMPTABLE: buff->append(STRING_WITH_LEN("TEMPTABLE ")); break;
    case VIEW_ALGORITHM_MERGE:     buff->append(STRING_WITH_LEN("MERGE ")); break;
```

```cpp
  if (table->with_check != VIEW_CHECK_NONE) {
    if (table->with_check == VIEW_CHECK_LOCAL)
      buff->append(STRING_WITH_LEN(" WITH LOCAL CHECK OPTION"));
    else
      buff->append(STRING_WITH_LEN(" WITH CASCADED CHECK OPTION"));   /* ★ 只区分 LOCAL / 其它 */
  }
```

### EXPLAIN：怎么看出合并还是物化

**① optimizer_trace 里用 `"merged": true` / `"materialized": true` 区分**（且用 `"view"` / `"derived"` 区分类型）：

```cpp
/* sql/sql_resolver.cc:3524 —— 合并 */
  Opt_trace_object trace_derived(trace, derived_table->is_view() ? "view" : "derived");
  trace_derived.add_utf8_table(derived_table)
      .add("select#", derived_query_block->select_number)
      .add("merged", true);

/* sql/sql_derived.cc:831 —— 物化 */
  Opt_trace_object trace_derived(trace, is_view() ? "view" : "derived");
  trace_derived.add_utf8_table(this)
      .add("select#", derived->first_query_block()->select_number)
      .add("materialized", true);
```

**② 传统 EXPLAIN 出现 `<derived#N>` = 该视图/派生表被物化成独立查询块**（`sql/opt_explain.cc:1448`）：

```cpp
  if (tr->query_block_id() && tr->is_view_or_derived() && !fmt->is_hierarchical()) {
    len = snprintf(namebuf, len - 1, "<derived%u>", tr->query_block_id_for_explain());
    return func(namebuf, len);
  }
```

> ★ 判定法：**合并掉的视图名字会直接消失、换成底层基表名；物化的则显示为 `<derivedN>` 并多出一个 `select_type=DERIVED/MATERIALIZED` 的行**。

**③ `optimizer_switch` 的 `derived_merge`** 只在 `ALGORITHM=UNDEFINED` 时参与（见下"提示不保证"）。

**④ EXPLAIN 视图的权限门槛**：无 SHOW VIEW 权限时报 `ER_VIEW_NO_EXPLAIN`（`sql_view.cc:1394`、`opt_explain.cc:1877`）。

---

## Misc

### ★ 工程实现技法

1. **"声明的算法"与"实际生效的算法"用两个字段分离**——`algorithm`（public，DD 里那份）vs `effective_algorithm`（private）。这是"提示不保证"的结构性表达，而不是靠注释约定。

2. **每个视图一个独立 `LEX`**——`parse_view_definition()` 里 `new (thd->mem_root) st_lex_local`，临时换 `thd->lex`，解析完再换回。配合 `Make_view_tracker` 这个 RAII 在失败时清掉 `view_query()`，回滚"它已是视图"这一事实。

3. **视图的 QB 被标成 `DERIVED_TABLE_TYPE` 并 `set_derived_query_expression()`**——一次赋值让视图在优化器侧与派生表完全同构，省掉一整套并行代码路径。

4. **权限用"逐列取交集"实现**——`final_priv &= fld->have_privileges`，保证"不能通过建视图获得额外权限"。这是安全模型里很干净的一处设计。

5. **CHECK OPTION 用"自底向上 AND 成一棵 Item 树"**——prepare 期构造一次，执行期逐行 `val_int()`。两个标志 `check_option_processed` / `replace_filter_processed` 保证 PS 重执行不重复构造。

### ★★ 失效与退化场景

| 场景 | 表现 | 源码位置 |
|---|---|---|
| **基表列被删** | 视图仍在，但查询时报 `ER_BAD_FIELD_ERROR` / `ER_VIEW_INVALID` | 每次开表重新解析 ⇒ 不会有"缓存的旧定义" |
| **`I_S.VIEWS.IS_UPDATABLE` 过期** | 底层视图变了，值不重算 | `sql/table.h:3697` 官方警告 |
| **不可更新视图上 DML** | UPDATE/DELETE 报 `ER_NON_UPDATABLE_TABLE`；**INSERT 报 `ER_NON_INSERTABLE_TABLE`** | `sql_update.cc:1539`、`sql_insert.cc:257` |
| **`ALGORITHM=MERGE` 不可行** | 静默降级为 UNDEFINED + **warning**（不是 error） | `sql_view.cc:942-946` |
| **视图列含表达式** | `field_for_view_update()==nullptr` ⇒ 该列不可更新 | `sql_view.cc:813` |
| **CHECK OPTION 拒绝插入** | `ER_VIEW_CHECK_FAILED`；`IGNORE` 下退化为 `VIEW_CHECK_SKIP`（跳过本行） | `sql/table.cc:4736` |
| **join view 一次改多张基表** | UPDATE 报 `ER_VIEW_MULTIUPDATE`；DELETE 直接 `ER_VIEW_DELETE_MERGE_VIEW` | `sql_update.cc:1745`、`sql_delete.cc:743` |
| **definer 用户被删** | 报 `ER_NO_SUCH_USER`；但 `SHOW CREATE`/`SHOW FIELDS` 降级为 warning 仍可看定义 | `sql/table.cc:4871` |
| **视图含 LIMIT 的 UPDATE/DELETE** | 需要视图含基表主键或全部列；否则受 `updatable_views_with_limit` 控制（warning vs error） | `check_key_in_view()` `sql_view.cc:1890` |
| **无物化视图** | 复杂视图每次重算 | `sql_yacc.yy` 中 `MATERIALIZED` 零命中 |

### 易混淆点

**① 视图 vs 派生表**：优化器侧同构；差别在 SQL/DD 层——视图有 `definer`/`view_suid`/`with_check`/DD 持久化/权限上下文，派生表全无。且 **`set_updatable()` 只发给视图**，所以派生表永远不可被 DML。

**② `ALGORITHM=UNDEFINED/MERGE/TEMPTABLE` 只是提示，不保证**。四个退化点：

```cpp
/* (a) CREATE 期：MERGE 不可行 → 静默改 UNDEFINED */
  if (lex->create_view_algorithm == VIEW_ALGORITHM_MERGE && !can_be_merged) {
    push_warning(thd, Sql_condition::SL_WARNING, ER_WARN_VIEW_MERGE, ...);
    lex->create_view_algorithm = VIEW_ALGORITHM_UNDEFINED;
  }

/* (b) prepare 期：优先级 —— ALGORITHM > hint > optimizer_switch + heuristic */
  /*
    Check whether derived table is mergeable, and directives allow merging;
    priority order is:
    - ALGORITHM says MERGE or TEMPTABLE
    - hint specifies MERGE or NO_MERGE (=materialization)
    - optimizer_switch's derived_merge is ON and heuristic suggests merge
  */
  if (derived_table->algorithm == VIEW_ALGORITHM_TEMPTABLE ||
      !derived_query_expression->is_mergeable())
    return false;
  if (derived_table->algorithm == VIEW_ALGORITHM_UNDEFINED) {
    const bool merge_heuristic =
        (derived_table->is_view() || allow_merge_derived) &&
        derived_query_expression->merge_heuristic(thd->lex);
    if (!hint_table_state(thd, derived_table, DERIVED_MERGE_HINT_ENUM,
                          merge_heuristic ? OPTIMIZER_SWITCH_DERIVED_MERGE : 0))
      return false;
  }

/* (c) 外层查询禁止合并 */
  if ((master_query_expression() == lex->unit && !lex->can_use_merged()) ||
      lex->can_not_use_merged())
    return false;

/* (d) STRAIGHT_JOIN + semi-join nest，或表数超限 */
  if ((active_options() & SELECT_STRAIGHT_JOIN) &&
      (derived_query_block->has_sj_nests || derived_query_block->has_aj_nests))
    return false;
  if (leaf_table_count + derived_query_block->leaf_table_count - 1 > MAX_TABLES)
    return false;
```

> ★ 注意 `merge_heuristic()`：**SELECT list 里含依赖自身列的非恒定子查询时不建议合并**（`sql_lex.cc:3816`）——注释解释：这种写法通常就是用户想让它物化。

**③ 社区没有物化视图**（证据见「版本演进」）。

**④ `WITH CHECK OPTION` 裸写 = CASCADED**；且 **LOCAL 也会累加上所有"已声明" CHECK OPTION 的底层视图条件**——"LOCAL 只检查本视图"是错的。

**⑤ `updatable` 与 `insertable` 的聚合方式相反**：`updatable |= `（任一即可）、`insertable &= `（必须全部）。

### 一句话总结

视图是**存了规范化定义文本的命名查询**，DD 里以 `dd::View` 持久化；**每次开表都要 `parse_view_definition()` 重新解析**（除非 PS/SP 重执行），这正是"视图跟随基表演进"的代价与收益。它的**可更新性几乎全部由 `Query_expression::is_mergeable()` 的 7 个条件决定**（"可合并 == 可更新"），`WITH CHECK OPTION` 在 prepare 期自底向上 AND 成一棵条件树、执行期逐行 `val_int()` 校验；**在优化器侧它与派生表完全同构，唯一区别是它有名字、有 DD 持久化、有 definer 与权限上下文**。

---

## 参考

**官方文档**

- *MySQL 8.0 Reference Manual → 25.6 Using Views*
- *MySQL 8.0 Reference Manual → 15.1.23 CREATE VIEW Statement*（含可更新视图与 CHECK OPTION）

**相关文档**

- 优化器侧的视图解析与合并算法：[`query/07_optimize/20_view_resolution.md`](query/07_optimize/20_view_resolution.md)
- definer 与 SQL SECURITY：[`auth/definer.md`](auth/definer.md)
- 物化执行：[`query/runtime/08_materialization.md`](query/runtime/08_materialization.md)
- 视图上的 MDL：[`../infra/lock/transactional/mdl.md`](../infra/lock/transactional/mdl.md)
- 存储程序（另一类存储对象）：[`stored_program.md`](stored_program.md)
