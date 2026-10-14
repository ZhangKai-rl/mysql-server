# 20 视图的解析与优化：view 是怎么被打开、解析、合并或物化的

> 基于 MySQL 8.0.39。本篇覆盖 view 从"打开表时发现它是个 view"到"被合并进外层查询或物化成临时表"的全过程：`open_and_read_view` / `parse_view_definition` / `Query_block::merge_derived` 的三级优先级、`ALGORITHM` 三值、可更新视图与 `WITH CHECK OPTION`。
>
> **边界**：派生表（derived table）的 merge 12 步细节见 [`logical/04_logical_join.md`](logical/04_logical_join.md)；CTE 与物化见 [`../runtime/08_materialization.md`](../runtime/08_materialization.md)；条件下推到派生表见 [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md)。**本篇写 view 独有的一条路**，但它与 derived 共用 `merge_derived()` 这个事实是理解的关键。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [一、view 的打开：从 open_table 到 parse_view_definition](#一view-的打开从-open_table-到-parse_view_definition)
- [二、三级优先级：ALGORITHM > hint > switch+heuristic](#二三级优先级algorithm--hint--switchheuristic)
- [三、技术约束：is_mergeable 与 CTE 的特殊规则](#三技术约束is_mergeable-与-cte-的特殊规则)
- [四、合并之后：表映射、条件下推、列替换](#四合并之后表映射条件下推列替换)
- [五、物化路径：view 作为临时表](#五物化路径view-作为临时表)
- [六、可更新视图与 WITH CHECK OPTION](#六可更新视图与-with-check-option)
- [七、完整调用栈](#七完整调用栈)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 一句话定位

在 MySQL 8.0 的源码里，**view 就是"一个带名字、定义存在数据字典里的派生表"**：

- 它的定义（一段 SQL 文本）从 DD 读出来，**重新解析**成一个 `Query_expression`，挂在 `Table_ref::derived` 上；
- 之后走的是与派生表**完全相同**的 `merge_derived()` / 物化逻辑。

所以 `Query_block::merge_derived()` 的 trace 节点会区分输出：

```cpp
Opt_trace_object trace_derived(trace,
                               derived_table->is_view() ? "view" : "derived");
```

★ 这个"共用"是理解 view 优化的第一要点：**没有单独的 view 优化器**，只有"从 DD 读定义 + 重新解析"这一步是 view 独有的。

**★ 源码对"具名 vs 匿名"的精确定义**（`sql/table.h:3061`，★ derived 就是"未具名的视图"）：

```cpp
  /// Return true if this represents a named view
  bool is_view() const { return view != nullptr; }

  /// Return true if this represents a derived table (an unnamed view)
  bool is_derived() const { return derived != nullptr && view == nullptr; }

  /// Return true if this represents a named view or a derived table
  bool is_view_or_derived() const { return derived != nullptr; }
```
> ★★ 关键：**没有 `bool is_view` 成员**——"是不是 view"就是 `LEX *view` 指针非空。`is_view_or_derived()` 只看 `derived` 而不看 `view`，因为**view 必然也有 `derived`**。
> ★ 两者的唯一差别就是 `LEX *view`（`table.h:3616`）：derived 只有 `Query_expression *derived`，view 额外持有一个独立 LEX。

**★ "view 不产生 TABLE" 的直接证据**（`table.h:3109`）：

```cpp
  /**
    @returns true for a table that is a placeholder, ie a derived table,
    a view, a recursive reference, a table function or a schema table.
    A table is also considered to be a placeholder if it does not have a
    TABLE object for some other reason.
  */
  bool is_placeholder() const {
    return is_view_or_derived() || is_recursive_reference() ||
           is_table_function() || schema_table || table == nullptr;
  }
```

**★ `Table_ref` 上的 view 相关成员全表**（⚠️ 8.0.39 已改名 `Table_ref`，`TABLE_LIST` 只是 typedef 兼容层）：

| 成员 | 位置 | 说明 |
|---|---|---|
| `Query_expression *derived` | `table.h:3616`（private） | view 与 derived **都有** |
| `LEX *view` | `table.h:3644`（private） | ★ **只有 view 有**——唯一区别 |
| `Common_table_expr *m_common_table_expr` | `table.h:3619` | CTE 引用 |
| `Field_translator *field_translation` | 公开 | ★ 列替换表（见第四节） |
| `Table_ref *merge_underlying_list` | 公开 | ★ **只在 MERGE 之后填充** |
| `mem_root_deque<Table_ref*> *view_tables` | 公开 | ★ "全量表"（含子查询里的表），与上面不同 |
| `Table_ref *belong_to_view` / `referencing_view` | 公开 | 最外层 view / 直接引用者（递归检测用） |
| `Item *check_option` | `table.h:3690` | WITH CHECK OPTION 条件 |
| `LEX_STRING select_stmt` | `table.h:3692` | ★ 从 DD 读出的**定义原文** |
| `ulonglong algorithm` | `table.h:3711` | 用户声明的 ALGORITHM（**不是枚举**，源码标了 `@todo`） |
| `ulonglong with_check` | `table.h:3713` | CHECK OPTION 声明值 |
| `enum_view_algorithm effective_algorithm` | `table.h:3717`（private） | ★ **实际生效**的算法 |
| `dd::enum_table_type required_type` | `table.h:3791` | 拒绝"要求 BASE TABLE 却是 VIEW" |
| `bool view_no_explain` | `table.h:3879` | 底层表权限不足 → EXPLAIN 被拒 |

```cpp
/* table.h:3132-3148 —— effective_algorithm 只能通过这四个接口读写 */
  bool is_merged() const { return effective_algorithm == VIEW_ALGORITHM_MERGE; }
  void set_merged() {
    assert(effective_algorithm == VIEW_ALGORITHM_UNDEFINED);
    effective_algorithm = VIEW_ALGORITHM_MERGE;
  }
  bool uses_materialization() const {
    return effective_algorithm == VIEW_ALGORITHM_TEMPTABLE;
  }
  void set_uses_materialization() {
    // @todo We should do this only once, but currently we cannot:
    // assert(effective_algorithm == VIEW_ALGORITHM_UNDEFINED);
    assert(effective_algorithm != VIEW_ALGORITHM_MERGE);
    effective_algorithm = VIEW_ALGORITHM_TEMPTABLE;
  }
```

### 一个反直觉的开场

```sql
CREATE ALGORITHM=MERGE VIEW v AS SELECT a, b FROM t WHERE a > 10;
SELECT * FROM v WHERE a > 20;
```

直觉上"MERGE 视图"一定能合并。但 `merge_derived()` 的**第一道**判断不是 ALGORITHM，而是：

```cpp
// Check whether the outer query allows merged views
if ((master_query_expression() == lex->unit && !lex->can_use_merged()) ||
    lex->can_not_use_merged())
  return false;
```

**外层查询不允许合并时，ALGORITHM=MERGE 也不合并。** 这类"外层上下文否决内层指令"的情形在 view 处理里有好几处，本篇会把它们列全。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | view 用 `.frm` 存储，`open_new_frm()` 读定义；merge 与 temptable 两种算法 |
| 8.0.0 | 数据字典（DD）化：view 定义存 `mysql.tables` 的 `view_definition`，`open_and_read_view()` 取代旧的 frm 路径 |
| 8.0.x | `derived_merge`（optimizer_switch）与 `/*+ MERGE()/NO_MERGE() */` hint 引入，形成本篇第二章的三级优先级 |
| 8.0.14+ | CTE 引入后，`is_mergeable()` 增加"多次引用的 CTE + 非确定性"约束 |
| 8.0.22+ | `derived_condition_pushdown`：条件下推到派生表（对 view 同样适用） |
| 8.0.31+ | view 内的 set operation 走新的 `Query_term` 树（见 [`19_set_operation.md`](19_set_operation.md)） |

**★ 源码取证的演进证据（带行号）**：

```cpp
/* ① sql/sql_resolver.cc:3638 —— ★ 5.6 及以前无条件合并 view 的 ORDER BY */
     Up to version 5.6 included, ORDER BY was unconditionally merged.
     Currently we only merge in the simple case above, which ensures
     backward compatibility for most reasonable use cases.
```
> ★ 注释完整段落（`sql_resolver.cc:3631-3650`）：现在只在"外层允许排序 + 只引用这一个 view + 不在 set operation 里 + 未分组/未聚合/自身未排序"时才把 `ORDER BY` 提到外层。

```cpp
/* ② sql/dd/impl/tables/tables.cc:130 —— ★ 8.0 起视图定义就是 mysql.tables 的一批列 */
  m_target_def.add_field(FIELD_VIEW_DEFINITION, "FIELD_VIEW_DEFINITION",
                         "view_definition LONGBLOB");
  m_target_def.add_field(FIELD_VIEW_DEFINITION_UTF8,
                         "FIELD_VIEW_DEFINITION_UTF8",
                         "view_definition_utf8 LONGTEXT");
  m_target_def.add_field(FIELD_VIEW_CHECK_OPTION, "FIELD_VIEW_CHECK_OPTION",
                         "view_check_option ENUM('NONE', 'LOCAL', 'CASCADED')");
  m_target_def.add_field(FIELD_VIEW_IS_UPDATABLE, "FIELD_VIEW_IS_UPDATABLE",
                         "view_is_updatable ENUM('NO', 'YES')");
  m_target_def.add_field(FIELD_VIEW_ALGORITHM, "FIELD_VIEW_ALGORITHM",
                         "view_algorithm ENUM('UNDEFINED', 'TEMPTABLE', 'MERGE')");
  m_target_def.add_field(FIELD_VIEW_SECURITY_TYPE, "FIELD_VIEW_SECURITY_TYPE",
                         "view_security_type ENUM('DEFAULT', 'INVOKER', 'DEFINER')");
  m_target_def.add_field(FIELD_VIEW_DEFINER, "FIELD_VIEW_DEFINER",
                         "view_definer VARCHAR(288)");
  m_target_def.add_field(FIELD_VIEW_COLUMN_NAMES, "FIELD_VIEW_COLUMN_NAMES",
                         "view_column_names LONGTEXT");
```
> ★⚠️ **真名核查**：8.0.39 **不存在 `mysql.view` 系统表**——`sql/dd/impl/tables/` 下只有 `view_routine_usage.cc` / `view_table_usage.cc`（依赖表）。

```cpp
/* ③ sql/dd/upgrade_57/table.cc:710 —— ★ 5.7 .frm → 8.0 DD 的迁移表 */
static const int REQUIRED_VIEW_PARAMETERS = 12;
/*
  Table of VIEW .frm field descriptors
  Note that one should NOT change the order for this,
  as it's used by parse().
*/
static File_option view_parameters[] = {
    {{STRING_WITH_LEN("query")}, ..., FILE_OPTIONS_ESTRING},
    {{STRING_WITH_LEN("updatable")}, ..., FILE_OPTIONS_ULONGLONG},
    {{STRING_WITH_LEN("algorithm")}, ..., FILE_OPTIONS_ULONGLONG},
    {{STRING_WITH_LEN("definer_user")}, ...}, {{STRING_WITH_LEN("definer_host")}, ...},
    {{STRING_WITH_LEN("suid")}, ..., FILE_OPTIONS_ULONGLONG},
    {{STRING_WITH_LEN("with_check_option")}, ..., FILE_OPTIONS_ULONGLONG},
    {{STRING_WITH_LEN("timestamp")}, ..., FILE_OPTIONS_TIMESTAMP},
    {{STRING_WITH_LEN("source")}, ..., FILE_OPTIONS_ESTRING},
    {{STRING_WITH_LEN("client_cs_name")}, ...}, {{STRING_WITH_LEN("connection_cl_name")}, ...},
    {{STRING_WITH_LEN("view_body_utf8")}, ..., FILE_OPTIONS_ESTRING},
};
```
> ★ 这条是"5.x 用 .frm 存视图"最直接的源码证据。升级先建 `create_unlinked_view()`（无列无依赖），再用 `ALTER VIEW` 补列与依赖。

**★ SQL 层与 DD 层是两套 algorithm 枚举**（⚠️ 值域不同，靠转换函数衔接）：

```cpp
/* sql/table.h:2479 —— SQL 层：0-based plain enum */
enum enum_view_algorithm {
  VIEW_ALGORITHM_UNDEFINED = 0,
  VIEW_ALGORITHM_TEMPTABLE = 1,
  VIEW_ALGORITHM_MERGE = 2
};
/* sql/dd/types/view.h:52 —— DD 层：1-based，且名字也略不同 */
  enum enum_algorithm { VA_UNDEFINED = 1,
                        VA_TEMPORARY_TABLE,   /* ★ SQL 侧叫 TEMPTABLE，DD 侧叫 TEMPORARY_TABLE */
                        VA_MERGE };
```
```cpp
/* sql/dd/dd_view.cc:126 —— "old/new" 命名本身就是 8.0 迁移痕迹 */
static enum enum_view_algorithm dd_get_old_view_algorithm_type(
    dd::View::enum_algorithm type) { ... }
static dd::View::enum_algorithm dd_get_new_view_algorithm_type(
    enum enum_view_algorithm type) { ... }
```
> ★ 两个转换函数都是**全 switch + `assert(false)`** 写法——加枚举值必须同步改这两处，否则 debug 构建直接 abort。

**★ 三个"枚举"的真名核查**（⚠️ 只有 algorithm 是真枚举）：

```cpp
/* sql/table.h:2485 —— SUID 与 CHECK 都是 #define 宏，不是枚举 */
#define VIEW_SUID_INVOKER 0
#define VIEW_SUID_DEFINER 1
#define VIEW_SUID_DEFAULT 2        /* ★★ DEFAULT=2 是"第三态"，既非 INVOKER 也非 DEFINER */

/* view WITH CHECK OPTION parameter options */
#define VIEW_CHECK_NONE 0
#define VIEW_CHECK_LOCAL 1
#define VIEW_CHECK_CASCADED 2

/* result of view WITH CHECK OPTION parameter check */
#define VIEW_CHECK_OK 0
#define VIEW_CHECK_ERROR 1
#define VIEW_CHECK_SKIP 2
```
> ★ `VIEW_SUID_DEFAULT` 的后果：`SHOW CREATE VIEW` 只用布尔真值判断（`if (table->view_suid)`），所以 DEFAULT(2) 与 DEFINER(1) **都打印 `SQL SECURITY DEFINER`**（`sql_show.cc:2665`）。
```cpp
/* sql/sql_lex.h:274 —— CREATE/ALTER/OR REPLACE 三态，8.0 已升级为 enum class */
enum class enum_view_create_mode {
  VIEW_CREATE_NEW,        // check that there are not such VIEW/table
  VIEW_ALTER,             // check that VIEW with such name exists
  VIEW_CREATE_OR_REPLACE  // check only that there are not such table
};
```

---

## 理论基础

### 视图的两种物理实现

这是数据库教科书里的经典二分：

| | **查询改写（view merging / query rewrite）** | **物化（materialization）** |
|---|---|---|
| 做法 | 把 view 的定义展开进外层查询，view 消失 | 先执行 view 的查询，结果存进临时表，外层再读它 |
| 优化机会 | **外层条件可以下推进 view 内部**，与 view 内部的条件合并 | 无；外层只能对临时表做过滤 |
| 索引利用 | view 内部的索引仍可用 | 临时表只有自动生成的键 |
| 适用范围 | 受限（聚合、DISTINCT、UNION、LIMIT 等都不能合并） | 任意 view |

MySQL 的 `merge_derived()` 属于前者，`uses_materialization()` 属于后者。

**为什么 MySQL 优先合并**：合并后 view 内部的索引可以被外层的等值谓词直接利用（`ref` 访问），这是数量级的差异。物化则必须读满整个 view 的结果。

**★ 源码实证："merge 优先、物化是 fallback"**（`sql_resolver.cc:1288`，★ 分派顺序本身就是答案）：

```cpp
  for (Table_ref *tl = get_table_list(); tl; tl = tl->next_local) {
    if (!tl->is_view_or_derived() && !tl->is_table_function()) continue;
    if (tl->is_merged() || tl->uses_materialization()) continue;   /* 已决定过 */
    assert(!tl->is_merged() && !tl->uses_materialization());

    if (tl->resolve_derived(thd, apply_semijoin)) return true;     /* ① prepare */
    if (!thd->lex->is_view_context_analysis()) {
      if (tl->is_mergeable() && merge_derived(thd, tl))
        return true;                                               /* ② ★ 能合并就合并 */
    }
    if (tl->is_merged()) continue;
    if (tl->is_table_function()) {
      if (tl->setup_table_function(thd)) return true;
    } else if (tl->table == nullptr && tl->setup_materialized_derived(thd)) {
      return true;                                                 /* ③ ★ 剩下的一律物化 */
    }
    materialized_derived_table_count++;
  }
```
> ★★ **`merge_derived()` 内部第 1 级判定的完整代码**（`sql_resolver.cc:3486`）：
> ```cpp
>     if (derived_table->algorithm == VIEW_ALGORITHM_TEMPTABLE ||
>         !derived_query_expression->is_mergeable())
>       return false;                        /* 第 1 级：ALGORITHM=TEMPTABLE 或技术不可合并 */
>
>     if (derived_table->algorithm == VIEW_ALGORITHM_UNDEFINED) {
>       const bool merge_heuristic =
>           (derived_table->is_view() || allow_merge_derived) &&
>           derived_query_expression->merge_heuristic(thd->lex);
>       if (!hint_table_state(thd, derived_table, DERIVED_MERGE_HINT_ENUM,
>                             merge_heuristic ? OPTIMIZER_SWITCH_DERIVED_MERGE : 0))
>         return false;                      /* 第 2/3 级：hint 覆盖 derived_merge 开关 */
>     }
> ```
> ★ 另一个否决条件（`sql_resolver.cc:3518`）：`leaf_table_count + derived_query_block->leaf_table_count - 1 > MAX_TABLES`(64) 也不可合并。

**★ 多次引用的非确定性 CTE 强制物化**（`table.cc:6484`，★ 真名就是 `is_mergeable()`）：

```cpp
/**
  @returns true if view or derived table is mergeable, based on
  technical constraints.
*/
bool Table_ref::is_mergeable() const {
  if (!is_view_or_derived() || algorithm == VIEW_ALGORITHM_TEMPTABLE)
    return false;
  /*
    If the table's content is non-deterministic and the query references it
    multiple times, merging it has the risk of creating different contents.
  */
  Common_table_expr *cte = common_table_expr();
  if (cte != nullptr && cte->references.size() >= 2 &&
      derived->uncacheable & UNCACHEABLE_RAND)
    return false;            /* ★★ "是 CTE" AND "引用 >=2 次" AND "UNCACHEABLE_RAND" */
  return derived->is_mergeable();
}
```

### 可更新视图与 WITH CHECK OPTION

SQL 标准允许对某些视图做 DML，并要求 `WITH CHECK OPTION` 保证"通过视图插入/更新的行，仍能被该视图看到"。两种作用域：

| | 语义 |
|---|---|
| `WITH LOCAL CHECK OPTION` | 只检查**本视图**的条件 |
| `WITH CASCADED CHECK OPTION` | 检查本视图**及其所有底层视图**的条件 |

MySQL 在 DD 里存的是枚举 `NONE` / `LOCAL` / `CASCADED`。

### 他库对比

| 库 | view 实现 | 差异 |
|---|---|---|
| **MySQL** | merge（改写）或 temptable（物化），**规则驱动**，有 `ALGORITHM` 提示 | 有"物化"这一非标准的物理形态暴露给用户 |
| **PostgreSQL** | 只有查询改写（view 是重写规则；8.3+ 优化器可下推）；物化视图是**独立对象**（`MATERIALIZED VIEW`，需手动 `REFRESH`） | PG 把"物化"做成显式对象，而不是优化器的隐式选择 |
| **Oracle** | 查询改写，且优化器有复杂的 view merging / predicate pushing / group-by placement | Oracle 的改写能力远强于 MySQL |
| **SQL Server** | 查询改写 + 索引视图（indexed view，自动维护） | 索引视图是 MySQL 完全没有的能力 |

★ 关键差异：**MySQL 把"是否物化"暴露成了一个用户可见的 `ALGORITHM` 选项**，而 PG/Oracle 把它完全交给优化器。

> ⚠️ **诚实标注**：对 `sql/sql_view.cc`、`sql/sql_derived.cc`、`sql/sql_base.cc` 三个文件执行 `PostgreSQL|Postgres|Oracle|SQL Server|other databases` 检索 —— **0 命中**。8.0.39 源码注释里**不存在**任何与其他数据库视图实现的对比文字。上表是他库知识，本篇可引用的"MySQL 特有行为"源码证据只有上面那段 `merge_derived()` 的优先级注释。

**★ 一句可引用的官方表述**（`sql/table.h:3065`，★ "派生表就是未具名的视图"）：

```cpp
  /// Return true if this represents a derived table (an unnamed view)
  bool is_derived() const { return derived != nullptr && view == nullptr; }
```

---

## 核心实现

### 一、view 的打开：从 open_table 到 parse_view_definition

> `sql/sql_base.cc` 的 `open_table()` 与 `sql/sql_view.cc`。

#### 1.1 主流程

```cpp
if (table_list->is_view() || share->is_view) {
  bool view_open_result = true;

  // ① merge 表（MRG_MYISAM）的子表是 view → 不支持
  if (table_list->parent_l)
    my_error(ER_WRONG_MRG_TABLE, MYF(0));

  // ② 版本校验：确认"期望是 view 时确实是 view"
  else if (check_and_update_table_version(thd, table_list, share))
    ;

  // ③ CREATE TABLE 与已有 view 同名时的特例
  else if (table_list->open_strategy == Table_ref::OPEN_FOR_CREATE) {
    /*
      The LEX object is used by the executor and other parts of the code to
      detect the presence of a view. As this is OPEN_FOR_CREATE we skip the
      call to open_and_read_view(), which creates the LEX object, and create
      a dummy LEX object.
      For SP and PS, LEX objects are created at the time of statement prepare
      and open_table() is called for every execute after that. Skip creation
      of LEX objects if it is already present.
    */
    if (!table_list->is_view()) return add_view_place_holder(thd, table_list);
    return false;
  }

  // ④ 真正读 view 定义
  else {
    view_open_result = open_and_read_view(thd, share, table_list);
  }

  release_table_share(share);
  mysql_mutex_unlock(&LOCK_open);

  if (view_open_result) return true;

  // ⑤ 解析 view 定义成查询树
  if (parse_view_definition(thd, table_list)) return true;

  assert(table_list->is_view());
  return false;
}
```

#### 1.2 两个核心函数

| 函数 | 位置 | 职责 |
|---|---|---|
| `open_and_read_view()` | `sql/sql_view.cc` | 从 DD 读出 view 的定义、algorithm、check option、definer、字符集等元数据，填进 `Table_ref`，并**创建一个 LEX 对象** |
| `parse_view_definition()` | `sql/sql_view.cc` | 把 view 的 SQL 文本**重新解析**成 `Query_expression`，挂到 `Table_ref::derived` |

#### 1.3 ★ 每个 view 有自己的 LEX

第三条分支的注释透露了一个重要事实：**view 会创建自己的 `LEX` 对象**。这与存储过程类似——嵌套的查询需要独立的解析器上下文。

由此带来的复杂性（源码注释自己承认的）：

- SP/PS 场景下，LEX 在 prepare 时创建，而 `open_table()` 每次 execute 都调，所以要判断"LEX 已存在则跳过"；
- `CREATE TABLE` 与已存在 view 同名时（`OPEN_FOR_CREATE`），不读定义但要**造一个 dummy LEX**，因为"执行器等地方靠 LEX 是否存在来判断这是不是个 view"。

★ 最后这一点是典型的"**用对象存在性代替布尔标志**"的设计，代价是所有相关代码都得假定"有 LEX 不一定有真定义"。

**★★ `parse_view_definition()`：独立 LEX 的四步创建**（`sql/sql_view.cc:1253`）：

```cpp
  // A parsed view requires its own LEX object
  LEX *const old_lex = thd->lex;                       /* ① 保存父 LEX */
  LEX *const view_lex = (LEX *)new (thd->mem_root) st_lex_local;
  if (!view_lex) return true;                          /* ② new 一个全新 LEX（语句 mem_root） */

  bool result = false;
  Make_view_tracker view_tracker(thd, view_ref, &result);

  thd->lex = view_lex;                                 /* ③ 全局切换 —— 之后所有 parse/resolve 都作用在这个独立上下文 */

  view_ref->set_view_query(view_lex);                  /* ④ 挂到 Table_ref 上，从此 is_view() == true */
```
```cpp
  LEX_CSTRING current_db_name_saved = thd->db();
  Parser_state parser_state;
  if ((result = parser_state.init(thd, view_ref->select_stmt.str,
                                  view_ref->select_stmt.length)))
    return true;    /* ★ 解析的输入就是 DD 里读出的 select_stmt 原文 */
  /*
    Use view db name as thread default database, in order to ensure
    that the view is parsed and prepared correctly.
  */
  mysql_mutex_lock(&thd->LOCK_thd_data);
  thd->reset_db({view_ref->db, view_ref->db_length});  /* ★★ 连当前库都要临时切到 view 所属 schema */
  mysql_mutex_unlock(&thd->LOCK_thd_data);

  lex_start(thd);

  Query_block *const view_query_block = view_lex->query_block;
  // Correctly mark unit for explain
  view_lex->unit->explain_marker = CTX_DERIVED;        /* ★ 在 EXPLAIN 里 view 与 derived 共用同一类标记 */
  ...
    // Parse the query text of the view
    result = parse_sql(thd, &parser_state, view_ref->view_creation_ctx);
  }    /* ★ 用的是 view 自己的字符集/collation 上下文，不是当前会话的 */
```
> ★★ 三个"临时切换"必须成对恢复：`thd->lex`（由 `Make_view_tracker` 析构恢复）、`thd->db`、`sql_mode`（`Sql_mode_parse_guard`）。

**★ `Make_view_tracker`：RAII 保证上下文还原 + 失败回滚**（`sql/sql_view.cc:1058`）：

```cpp
/// RAII class to ease error handling in parse_view_definition()
class Make_view_tracker {
 public:
  Make_view_tracker(THD *thd, Table_ref *view_ref, bool *result)
      : thd(thd), old_lex(thd->lex), view_ref(view_ref), result(result) {}
  ~Make_view_tracker() {
    if (thd->lex != old_lex) {
      lex_end(thd->lex);   // Terminate processing of view LEX
      thd->lex = old_lex;  // Needed for prepare_security
    }                      /* ★ 解析完必须换回父 LEX，否则整个语句上下文被污染 */
    if (*result && view_ref->is_view()) {
      delete view_ref->view_query();
      view_ref->set_view_query(nullptr);  // view_ref is no longer a VIEW
    }                      /* ★ 失败回滚：删掉 LEX 并置空 → is_view() 自动变 false */
  }
```

**★★ 子 LEX 与父 LEX 怎么关联**（`merge_query_blocks()`，`sql_view.cc:1181`，★ 不是树形指针链）：

```cpp
/**
  Merge a view query expression into the parent expression.
  Update all LEX pointers inside the view expression to point to the parent LEX.
*/
void merge_query_blocks(LEX *view_lex, LEX *parent_lex) {
  for (Query_block *select = view_lex->all_query_blocks_list; select != nullptr;
       select = select->next_select_in_list())
    select->parent_lex = parent_lex;
}
```
> ★★ **子 LEX 不是通过"父指针"挂上去的**，而是把子 LEX 里**每个 `Query_block` 的 `parent_lex` 全部改写指向父 LEX**。这是 view 嵌套解析的关键机制。

**★ 底层表被"就地插入"父 LEX 的全局表链**（`sql_view.cc:1450`，★ 这是 `open_tables()` 下一轮能打开它们的原因）：

```cpp
  if (view_tables && !(view_ref->prelocking_placeholder && ...)) {
    if (view_ref->next_global) {
      view_tables_tail->next_global = view_ref->next_global;
      view_ref->next_global->prev_global = &view_tables_tail->next_global;
    } else {
      old_lex->query_tables_last = &view_tables_tail->next_global;
    }
    view_tables->prev_global = &view_ref->next_global;
    view_ref->next_global = view_tables;   /* ★ 紧跟在 view 之后插入 */
  }
```
```cpp
  for (Table_ref *tbl = view_tables; tbl; tbl = (view_tables_tail = tbl)->next_global) {
    tbl->open_type = OT_BASE_ONLY;
    tbl->belong_to_view = top_view;        /* ★ 指向最外层 view（多层嵌套只记最外层） */
    tbl->referencing_view = view_ref;      /* ★ 指向直接引用者 */
    tbl->grant.privilege = 0;              /* ★ 权限清零，按用户上下文重算 */
    view_ref->view_tables->push_back(tbl);
  }
```
> ★ 注释解释了为什么必须"紧跟在 view 之后"而不是追加到表链尾部：*"It is important for UPDATE/INSERT/DELETE checks to have these tables just after view instead of at tail of list, to be able to check that table is unique."*

**★ 不确定性从子 LEX 冒泡到父 LEX**（`sql_view.cc:1470`）：

```cpp
  old_lex->set_stmt_unsafe_flags(view_lex->get_stmt_unsafe_flags());
  ...
  old_lex->safe_to_cache_query &= view_lex->safe_to_cache_query;
```
```cpp
  const bool view_is_mergeable =
      view_ref->algorithm != VIEW_ALGORITHM_TEMPTABLE &&
      view_lex->unit->is_mergeable();      /* ★ 打开阶段的第一次可合并性判定 */
  ...
  if (view_is_mergeable) {
    view_lex->sql_command = old_lex->sql_command;   /* ★ 只有可合并（因而是可更新）的视图才继承父语句的 DML 命令字 */
    view_lex->duplicates = old_lex->duplicates;
  }
```

**★ 两个核心函数的签名**（`sql/sql_view.h:62`）：

```cpp
bool open_and_read_view(THD *thd, TABLE_SHARE *share, Table_ref *view_ref);
bool parse_view_definition(THD *thd, Table_ref *view_ref);

/*
  Check if view is updatable.
  @retval true   View is updatable.
  @retval false  Otherwise.
*/
bool is_updatable_view(THD *thd, Table_ref *view);
```
> ⚠️ **真名核查**：运行期判定用 `Table_ref::is_updatable()`（`table.h:3154`）；`is_updatable_view()` 是 CREATE/ALTER 期用的**自由函数**（`sql_view.cc:785`）。两个同名概念，别混。
> ⚠️ **`mysql_make_view()` 不存在**（全仓仅 `sql_base.cc:143` 一行注释提到）；`update_view_metadata()` 不在 `sql_view.cc`，在 `sql/dd_sql_view.cc:719`。
> ★ `open_and_read_view()` 只读定义（`dd::read_view` 从 DD 读），`parse_view_definition()` 才建 LEX 并 parse——**两者紧挨着成对出现**。

#### 1.4 view 不产生 TABLE

`sql_base.cc` 里有 `assert(table_list->is_view() || table_list->table == nullptr)` —— view 的 `Table_ref::table` 始终为空（除非被物化）。这与派生表一致：**view/derived 不是存储实体**。

### 二、三级优先级：ALGORITHM > hint > switch+heuristic

> `Query_block::merge_derived()`，`sql/sql_resolver.cc`。

#### 2.1 源码注释直接给出了优先级

```cpp
/*
  Check whether derived table is mergeable, and directives allow merging;
  priority order is:
  - ALGORITHM says MERGE or TEMPTABLE
  - hint specifies MERGE or NO_MERGE (=materialization)
  - optimizer_switch's derived_merge is ON and heuristic suggests merge
*/
```

#### 2.2 逐条展开

```cpp
// 第 0 道：外层查询是否允许合并（★ 连 ALGORITHM=MERGE 都否决得了）
if ((master_query_expression() == lex->unit && !lex->can_use_merged()) ||
    lex->can_not_use_merged())
  return false;

// 第 1 级：ALGORITHM=TEMPTABLE 或技术上不可合并 → 直接放弃
if (derived_table->algorithm == VIEW_ALGORITHM_TEMPTABLE ||
    !derived_query_expression->is_mergeable())
  return false;

// 第 2、3 级：ALGORITHM 未指定时，看 hint，再看 switch + 启发式
if (derived_table->algorithm == VIEW_ALGORITHM_UNDEFINED) {
  const bool merge_heuristic =
      (derived_table->is_view() || allow_merge_derived) &&
      derived_query_expression->merge_heuristic(thd->lex);
  if (!hint_table_state(thd, derived_table, DERIVED_MERGE_HINT_ENUM,
                        merge_heuristic ? OPTIMIZER_SWITCH_DERIVED_MERGE : 0))
    return false;
}
```

★ **一个容易漏掉的细节**：

```cpp
(derived_table->is_view() || allow_merge_derived)
```

**view 不受 `allow_merge_derived` 的限制，派生表才受。** 也就是说某些场景下 view 比等价的派生表**更容易**被合并。这是 view 与 derived 在合并决策上的实质差异（尽管共用同一个函数）。

`hint_table_state()` 的调用方式也值得注意：它把"启发式结果"与"optimizer_switch 位"合成一个参数传入，即 **hint 优先于启发式与 switch**；没有 hint 时，看启发式是否建议合并，若建议则进一步看 `derived_merge` 开关。

#### 2.3 之后的其它否决条件

```cpp
// STRAIGHT_JOIN 下，含 semi-join nest 的 query block 不能合并进来
if ((active_options() & SELECT_STRAIGHT_JOIN) &&
    (derived_query_block->has_sj_nests || derived_query_block->has_aj_nests))
  return false;

// 合并后叶子表数不能超过 MAX_TABLES
if (leaf_table_count + derived_query_block->leaf_table_count - 1 > MAX_TABLES)
  return false;

derived_table->set_merged();
```

`MAX_TABLES`（64）这个限制很实际：**合并会把 view 内部的叶子表"摊平"进外层**，表数会涨；超过 64 就合不了，只能物化。这也是多表 view 容易退化成临时表的原因之一。

### 三、技术约束：is_mergeable 与 CTE 的特殊规则

> `Table_ref::is_mergeable()`，`sql/table.cc`。

```cpp
bool Table_ref::is_mergeable() const {
  if (!is_view_or_derived() || algorithm == VIEW_ALGORITHM_TEMPTABLE)
    return false;
  /*
    If the table's content is non-deterministic and the query references it
    multiple times, merging it has the risk of creating different contents.
  */
  Common_table_expr *cte = common_table_expr();
  if (cte != nullptr && cte->references.size() >= 2 &&
      derived->uncacheable & UNCACHEABLE_RAND)
    return false;
  return derived->is_mergeable();
}
```

两层判定：

1. **`Table_ref::is_mergeable()`**：ALGORITHM 层 + CTE 层
2. **`Query_expression::is_mergeable()`**：查询结构层（聚合、DISTINCT、UNION、LIMIT、窗口等，见 [`logical/04_logical_join.md`](logical/04_logical_join.md)）

★ CTE 那条规则是个很细的正确性考虑：**一个被引用多次的非确定性 CTE，如果每处都合并展开，可能得到不同的内容**（例如含 `RAND()`）。所以强制物化，保证多处引用看到同一份结果。

这解释了为什么"CTE 里用了 RAND() 时，MySQL 的行为会和普通派生表不一样"。

### 四、合并之后：表映射、条件下推、列替换

合并（`set_merged()`）之后要做的事（细节见 [`logical/04_logical_join.md`](logical/04_logical_join.md)）：

| 动作 | 说明 |
|---|---|
| 叶子表搬移 | view 内部的叶子表被搬进外层的 `leaf_tables`，view 的 `Table_ref` 变成"nest" |
| 条件下推 | 外层的 WHERE 条件下推进 view 内部（8.0.22+ 的 `derived_condition_pushdown` 更激进） |
| 列替换 | 外层对 `v.a` 的引用替换成 `t.a`（`Item_field` 换指） |
| 表数更新 | `leaf_table_count` 增加 |

★ 合并带来的**性能收益主要来自条件下推**：`SELECT * FROM v WHERE a > 20` 里 `a > 20` 会被推进 view 内部与 `a > 10` 合并，变成 `a > 20`，进而可能走索引。

物化路径下没有任何下推，view 必须被完整执行。

**★★ 表映射：叶子链上"视图"这个节点被子查询的叶子链整体替换**（`sql_resolver.cc:3562`）：

```cpp
  // Set up permanent list of underlying tables of a merged view
  derived_table->merge_underlying_list = derived_query_block->get_table_list();
                                             /* ① 表映射第 1 步 */

  // Replace derived table in leaf table list with underlying tables:
  for (Table_ref **tl = &leaf_tables; *tl; tl = &(*tl)->next_leaf) {
    if (*tl == derived_table) {
      for (Table_ref *leaf = derived_query_block->leaf_tables; leaf;
           leaf = leaf->next_leaf) {
        leaf->dep_tables <<= table_adjust;   /* ★ 依赖位图整体左移 */
        if (leaf->next_leaf == nullptr) {
          leaf->next_leaf = (*tl)->next_leaf;
          break;
        }
      }
      *tl = derived_query_block->leaf_tables;  /* ★★ 视图节点被子查询叶子链整体替换 */
      break;
    }
  }

  leaf_table_count += (derived_query_block->leaf_table_count - 1);
  ...
  // Remove tables from old query block:
  derived_query_block->leaf_tables = nullptr;
  derived_query_block->leaf_table_count = 0;
  derived_query_block->m_table_list.clear();  /* ★ 子 query block 被"掏空" */
```
```cpp
/* table.cc:4372 —— merge_underlying_tables：建 nest */
bool Table_ref::merge_underlying_tables(Query_block *select) {
  assert(nested_join->m_tables.empty());
  for (Table_ref *tl : select->m_table_nest) {
    tl->embedding = this;                 /* ★ 底层表的"包裹 nest"改指向视图 */
    tl->join_list = &nested_join->m_tables;
    nested_join->m_tables.push_back(tl);
  }
  return false;
}
```
> ★ ★ 真名核查：**不存在 `replace_leaf_tables()`** —— 真实实现就是上面内联的叶子链替换 + `remap_tables(thd)` + `fix_tables_after_pullout(...)`。

**★ 合并后的收尾三件事**（`sql_resolver.cc:3619`）：

```cpp
  // Exclude the derived table query expression from query graph.
  derived_query_expression->exclude_level();

  // Don't try to access it:
  derived_table->set_derived_query_expression((Query_expression *)1);
                                             /* ★★ "哨兵 1"：非 0 但不可用 */

  // Merge subquery's name resolution contexts into parent's
  merge_contexts(derived_query_block);
  repoint_contexts_of_join_nests(derived_query_block->m_table_nest);

  // Leaf tables have been shuffled, so update table numbers for them
  remap_tables(thd);                          /* ★ 真名就是 remap_tables() */

  // Update table info of referenced expressions after query block is merged
  fix_tables_after_pullout(this, derived_query_block, derived_table,
                           table_adjust,
                           derived_query_expression->m_lateral_deps);
```
```cpp
/* table.h:3251 —— 同一个"哨兵 1"套路也用在 view 上 */
  /**
    Set the LEX object of a view (will also define this as a view).
    @note: The value 1 is used to indicate a view but without a valid
           query object. Use only if the LEX object is not going to
           be used in later processing.
  */
  void set_view_query(LEX *lex) { view = lex; }

  /// Return the valid LEX object for a view.
  LEX *view_query() const {
    assert(view != nullptr && view != (LEX *)1);   /* ★ 哨兵 1 在此被显式排除 */
    return view;
  }
```

**★ 条件下推的真实实现就是 `and_conds()`**（`table.cc:4500`）：

```cpp
/**
  Merge WHERE condition of view or derived table into outer query.

  If the derived table is on the inner side of an outer join, its WHERE
  condition is merged into the respective join operation's join condition,
  otherwise the WHERE condition is merged with the derived table's
  join condition.
*/
bool Table_ref::merge_where(THD *thd) {
  Item *const condition =
      derived_query_expression()->first_query_block()->where_cond();
  if (!condition) return false;

  /*
    Save the WHERE condition separately. This is needed because it is already
    resolved, so we need to explicitly update used tables information after
    merging this derived table into the outer query.
  */
  derived_where_cond = condition;
  ...
  set_join_cond(and_conds(join_cond(), condition));   /* ★★ view 的 WHERE 被 AND 进 nest 的 join condition */
  if (!join_cond()) return true;
  return false;
}
```

**★★ 列替换的本质：`Field_translator[]`**（`table.cc:4529`，⚠️ **`Item_direct_view_ref` 不存在**）：

```cpp
/**
  Create field translation for merged derived table/view.
*/
bool Table_ref::create_field_translation(THD *thd) {
  Query_block *select = derived->first_query_block();
  uint field_count = 0;
  ...
  Field_translator *transl = (Field_translator *)thd->stmt_arena->alloc(
      select->num_visible_fields() * sizeof(Field_translator));
  if (!transl) return true;

  for (Item *item : select->visible_fields()) {
    /*
      Notice that all items keep their nullability here.
      All items are later wrapped within Item_direct_view objects.
      ...
    */            /* ⚠️ 注释里的 Item_direct_view 在 8.0.39 代码中已不存在，是历史残留 */
    transl[field_count].name = m_derived_column_names
                                   ? (*m_derived_column_names)[field_count].str
                                   : item->item_name.ptr();
    transl[field_count++].item = item;    /* ★★ 视图第 i 列 → 子 SELECT 第 i 个 Item（共享 Item 树，不拷贝） */
  }
  field_translation = transl;
  field_translation_end = transl + field_count;
  return false;
}
```
> ★★ **`Item_direct_view_ref` 类在 8.0.39 不存在**（全仓仅 `table.cc:4555` 注释与 mysql-test 结果文件残留）。当前实现是 `Field_translator{item, name}` 数组 + `Item_ref`（`Item::REF_ITEM`）间接层。

**★ 合并后顺带确定可更新性**（`sql_resolver.cc:3540`，★ OR / AND 的不对称）：

```cpp
  /**
    A view is updatable if any underlying table is updatable.
    A view is insertable-into if all underlying tables are insertable.
    A view is not updatable nor insertable if it contains an outer join
    @see mysql_register_view()
  */
  if (derived_table->is_view()) {
    bool updatable = false;
    bool insertable = true;
    bool outer_joined = false;
    for (Table_ref *tr = derived_table->merge_underlying_list; tr;
         tr = tr->next_local) {
      updatable |= tr->is_updatable();       /* ★ 可更新 = 任一层可更新（OR） */
      insertable &= tr->is_insertable();     /* ★ 可插入 = 全部可插入（AND） */
      outer_joined |= tr->is_inner_table_of_outer_join();
    }
    updatable &= !outer_joined;              /* ★ outer join 内侧 → 两者都否 */
    insertable &= !outer_joined;
    if (updatable) derived_table->set_updatable();
    if (insertable) derived_table->set_insertable();
  }
```

### 五、物化路径：view 作为临时表

若 `merge_derived()` 返回 false，`Table_ref::uses_materialization()` 为真，view 会在 prepare 期走：

```cpp
} else if (tl->table == nullptr && tl->setup_materialized_derived(thd)) {
```

即把 view 的 `Query_expression` 当作派生表物化（建临时表、优化、执行、填充）。相关细节见 [`../runtime/08_materialization.md`](../runtime/08_materialization.md)。

一个有用的判据（优化期能否把它当常量）：

```cpp
bool Table_ref::materializable_is_const() const {
  const Query_expression *unit = derived_query_expression();
  return unit->query_result()->estimated_rowcount <= 1 &&
         (unit->first_query_block()->active_options() &
          OPTION_NO_SUBQUERY_DURING_OPTIMIZATION) == 0;
}
```

估算行数 ≤ 1 的物化 view 可以在优化期直接求值。

**★ 物化三件套**（`sql/sql_derived.cc`，★ 建临时表 → 真正建 → 填充）：

```cpp
/* ① :820 setup_materialized_derived_tmp_table —— 造 TABLE 骨架 */
  assert(is_view_or_derived() && !is_merged() && table == nullptr);
  Opt_trace_object trace_derived(trace, is_view() ? "view" : "derived");
  trace_derived.add_utf8_table(this)
      .add("select#", derived->first_query_block()->select_number)
      .add("materialized", true);          /* ★ trace 里 "materialized": true 是"走物化"的判据 */

  set_uses_materialization();
  // From resolver POV, columns of this table are readonly
  set_readonly();                          /* ★ 物化视图的列在 resolver 眼里一律只读 */

  if (m_common_table_expr && m_common_table_expr->tmp_tables.size() > 0) {
    trace_derived.add("reusing_tmp_table", true);
    table = m_common_table_expr->clone_tmp_table(thd, this);   /* ★ CTE 共享 */
  }
  if (table == nullptr) {
    ulonglong create_options =
        derived->first_query_block()->active_options() | TMP_TABLE_ALL_COLUMNS;
    if (m_derived_column_names) {          /* ★ CREATE VIEW v(c1,c2) 在这里"临时偷换"列名 */
      swap_column_names_of_unit_and_tmp_table(*derived->get_unit_column_types(),
                                              *m_derived_column_names);
    }
    bool is_distinct = derived->can_materialize_directly_into_result() &&
                       derived->has_top_level_distinct();
    bool rc = derived_result->create_result_table(
        thd, *derived->get_unit_column_types(), is_distinct, create_options,
        alias, false, false);              /* ★ 临时表用 alias 命名 */
    ...
    table->s->tmp_table = NON_TRANSACTIONAL_TMP_TABLE;
```
```cpp
/* ② :1636 create_materialized_table —— 真正在 SE 里建出来 */
  // @todo: Be able to assert !table->is_created() as well
  assert((is_table_function() || derived_query_expression()) &&
         uses_materialization() && table);
  ...
  if (table->is_created() ||
      (query_block->join != nullptr &&
       (query_block->join->const_table_map & map()))) {
    return false;                          /* ★ 已建 / const 表 → 不重复建 */
  }
  if (instantiate_tmp_table(thd, table)) return true;
  table->file->ha_extra(HA_EXTRA_IGNORE_DUP_KEY);
```
```cpp
/* ③ :1698 materialize_derived —— 填充 */
  assert(is_view_or_derived() && uses_materialization());
  assert(table && table->is_created() && !table->materialized);
                                           /* ★★ 两条 assert 锁死调用时序：禁止在 prepare 阶段调用 */
  Derived_refs_iterator it(this);
  while (TABLE *t = it.get_next())
    if (t->materialized) {
      table->materialized = true;
      table->set_not_started();
      return false;                        /* ★ CTE 多次引用只填一次 */
    }
  if (unit->is_recursive()) {
    assert(table->s->primary_key == MAX_KEY);   /* ★ 递归 CTE 的不变量 */
  }
  ...
  bool res = unit->execute(thd);
  ...
  table->materialized = true;
  table->set_not_started();                /* ★ 否则 read_system/read_const 会忘记读行 */
```
> ★ 一个源码自认的坑（`:1612`）：*"The hypergraph optimizer ... will get confused and crash if it has already been materialized"* —— 所以 hypergraph 下 const 表不在优化期提前物化。

**★ `CHECK TABLE` 强制视图走物化**（`sql/sql_admin.cc:873`，★ 即使 `ALGORITHM=MERGE`）：

```cpp
      /*
        Views are always treated as materialized views, including creation
        of temporary table descriptor.
      */
      if (!open_error && table->is_view()) {
        open_error = table->resolve_derived(thd, false);
        if (!open_error) open_error = table->setup_materialized_derived(thd);
      }
```

### 六、可更新视图与 WITH CHECK OPTION

#### 6.1 两个时点

| 时点 | 函数 | 做什么 |
|---|---|---|
| prepare | `Table_ref::prepare_check_option(thd, is_cascaded)` | 把 view 的条件构造成一个 `Item` 求值表达式 |
| 执行（DML 每一行） | `Table_ref::view_check_option(thd)` | 对当前行求值，返回 `VIEW_CHECK_OK` / `VIEW_CHECK_SKIP` / `VIEW_CHECK_ERROR` |

`view_check_option()` 的实现：

```cpp
int Table_ref::view_check_option(THD *thd) const {
  if (check_option && check_option->val_int() == 0) {
    const Table_ref *main_view = top_table();
    my_error(ER_VIEW_CHECK_FAILED, MYF(0), main_view->db, main_view->table_name);
    return VIEW_CHECK_ERROR;
  }
  return VIEW_CHECK_SKIP;   // 见下方说明
}
```

返回值语义（DML 侧的 switch）：

- `VIEW_CHECK_SKIP` —— **跳过这一行**（local check 时不满足就跳过，不算错）
- `VIEW_CHECK_ERROR` —— 报错 `ER_VIEW_CHECK_FAILED`
- `VIEW_CHECK_OK` —— 通过

这个三态返回值是 `LOCAL` 与 `CASCADED` 语义差异的载体：`LOCAL` 允许"不满足就跳过该行"，`CASCADED` 则更严格。

**★ 三态的语义差异只在最后一行**（`table.cc:4736`）：

```cpp
int Table_ref::view_check_option(THD *thd) const {
  if (check_option && check_option->val_int() == 0) {
    const Table_ref *main_view = top_table();      /* ★ 报错报的是最外层视图名 */
    my_error(ER_VIEW_CHECK_FAILED, MYF(0), main_view->db,
             main_view->table_name);
    if (thd->lex->is_ignore()) return (VIEW_CHECK_SKIP);  /* ★ IGNORE → "失败但继续" */
    return (VIEW_CHECK_ERROR);                             /* ★ 否则"失败即中止" */
  }
  return (VIEW_CHECK_OK);
}
```
> ⚠️ **两个常量组别混用**：**选项**三值 `VIEW_CHECK_NONE/LOCAL/CASCADED`（`table.h:2490`，宏不是枚举）vs **返回值**三态 `VIEW_CHECK_OK/ERROR/SKIP`（`table.h:2494`）。
> ★ `is_ignore()` 决定 SKIP vs ERROR——所以 `INSERT IGNORE` / `UPDATE IGNORE` 下违反 CHECK OPTION 只跳过该行。

**★ `prepare_check_option()`：CASCADED 会"传染"，LOCAL 不会**（`table.cc:4607`）：

```cpp
bool Table_ref::prepare_check_option(THD *thd, bool is_cascaded) {
  assert(is_view());
  /*
    True if conditions of underlying views should be treated as WITH CASCADED
    CHECK OPTION
  */
  is_cascaded |= (with_check == VIEW_CHECK_CASCADED);   /* ★★ CASCADED 沿递归向下传染 */

  for (Table_ref *tbl = merge_underlying_list; tbl; tbl = tbl->next_local) {
    if (tbl->is_view() && tbl->prepare_check_option(thd, is_cascaded))
      return true;
  }

  if (!check_option_processed) {
    if ((with_check || is_cascaded) &&
        merge_join_conditions(thd, this, &check_option))
      return true;      /* ★ 把本视图的 WHERE + join condition 收集成 check_option */

    for (Table_ref *tbl = merge_underlying_list; tbl; tbl = tbl->next_local) {
      if (tbl->check_option &&
          !(check_option = and_conds(check_option, tbl->check_option)))
        return true;    /* ★ 底层视图的 check_option 用 AND 串起来 */
    }
    check_option_processed = true;
  }
  ...
```

#### 6.2 调用点分布

`view_check_option()` 在 `sql_insert.cc`、`sql_update.cc`、`sql_load.cc` 里被调用——即 **INSERT / UPDATE / LOAD DATA 都可能触发 CHECK OPTION 检查**。

**★ 精确调用点**（★ UPDATE 只在"行真的变了"时才校验）：

```cpp
/* sql/sql_update.cc:915 */
      if (is_row_changed) {                /* ★★ 只在行真的变化时校验 */
        /*
          Default function and default expression values are filled before
          evaluating the view check option. Check option on view using table(s)
          with default function and default expression breaks otherwise.
          It is safe to not invoke CHECK OPTION for VIEW if records are same.
          In this case the row is coming from the view and thus should satisfy
          the CHECK OPTION.
        */
        int check_result = table_list->view_check_option(thd);
        if (check_result != VIEW_CHECK_OK) {
          if (check_result == VIEW_CHECK_SKIP) continue;
          else if (check_result == VIEW_CHECK_ERROR) { error = 1; break; }
        }
```
```cpp
/* sql/sql_insert.cc:608 —— 在写行之后、CHECK 约束之前 */
      const int check_result = table_list->view_check_option(thd);
      if (check_result == VIEW_CHECK_SKIP) continue;
      else if (check_result == VIEW_CHECK_ERROR) { has_error = true; break; }
```
> ★ 其余：`sql_insert.cc:1225`（prepare）/`:1997`/`:2366`；`sql_update.cc:2465`/`2761`；`sql_load.cc:278`（prepare）/`:856`/`1095`/`1276`；`sql_resolver.cc:1800`。

**★ `WITH CHECK OPTION` 语法：4 个分支**（`sql_yacc.yy:17555`，⚠️ 缺省是 **NONE** 不是 CASCADED）：

```yacc
view_check_option:
          %empty                          { $$= VIEW_CHECK_NONE; }
        | WITH CHECK_SYM OPTION           { $$= VIEW_CHECK_CASCADED; }   /* ★ 裸写 = CASCADED */
        | WITH CASCADED CHECK_SYM OPTION  { $$= VIEW_CHECK_CASCADED; }
        | WITH LOCAL_SYM CHECK_SYM OPTION { $$= VIEW_CHECK_LOCAL; }
        ;
```

**★ ALGORITHM 三值产生式**（`sql_yacc.yy:17447`）：

```yacc
view_algorithm:
          ALGORITHM_SYM EQ UNDEFINED_SYM
          { Lex->create_view_algorithm= VIEW_ALGORITHM_UNDEFINED; }
        | ALGORITHM_SYM EQ MERGE_SYM
          { Lex->create_view_algorithm= VIEW_ALGORITHM_MERGE; }
        | ALGORITHM_SYM EQ TEMPTABLE_SYM
          { Lex->create_view_algorithm= VIEW_ALGORITHM_TEMPTABLE; }
        ;
```

#### 6.3 可更新性

**★ 六条限制**（`is_updatable_view()`，`sql_view.cc:785`）：

```cpp
  bool can_be_merged =
      lex->unit->is_mergeable() && view->algorithm != VIEW_ALGORITHM_TEMPTABLE;
                                     /* ① 不可合并 / ALGORITHM=TEMPTABLE → 不可更新 */
  if (!dd::get_dictionary()->is_system_view_name(view->db, view->table_name) &&
      (updatable_view = can_be_merged)) {
                                     /* ② I_S 系统视图一律不可更新 */
      updatable |= !((tbl->is_view() && !tbl->updatable_view) || tbl->schema_table);
                                     /* ③ 底层有不可更新视图 / schema 表 → 该表不可更新 */
      outer_joined |= tbl->is_inner_table_of_outer_join();
    }
    updatable &= !outer_joined;      /* ④ outer join 内侧 → 不可更新 */
      for (Item *item : lex->query_block->visible_fields()) {
        Item_field *item_field = item->field_for_view_update();
        if (item_field && !item_field->table_ref->schema_table) {
          view_has_updatable_column = true; break; }
      }
      updatable &= view_has_updatable_column;
                                     /* ⑤ 至少要有一列能映射到真实 Field（聚合/表达式列不行） */
  if (updatable_view && ... &&
      find_table_in_global_list(lex->query_tables->next_global, ...)) {
    updatable_view = false;          /* ⑥ 目标表又在子查询里被引用 → 不可更新 */
  }
```
> ⚠️ 第 ⑥ 条**只能抓最简单的情况**——源码注释自认：*"this test can catch only very simple cases ... all other will be detected before updating commands execution. (it is more optimisation then real check)"*

**★ 带 LIMIT 更新视图必须有键 —— `updatable_views_with_limit` 兜底**（⚠️ **该变量 8.0.39 仍在**）：

```cpp
/* sql/sql_view.cc:1982 */
        /*
          Keys or all fields of underlying tables are not found => we have
          to check variable updatable_views_with_limit to decide should we
          issue an error or just a warning
        */
        if (thd->variables.updatable_views_with_limit) {
          /* update allowed, but issue warning */
          push_warning(thd, Sql_condition::SL_NOTE, ER_WARN_VIEW_WITHOUT_KEY,
                       ER_THD(thd, ER_WARN_VIEW_WITHOUT_KEY));
          return false;
        }
        /* prohibit update */
```
```cpp
/* sql/sys_vars.cc:5066 —— 默认 true，且 HINT_UPDATEABLE */
// why ENUM and not BOOL ?          /* ★ 源码原话 */
static const char *updatable_views_with_limit_names[] = {"NO", "YES", nullptr};
static Sys_var_enum Sys_updatable_views_with_limit(
    "updatable_views_with_limit",
    "YES = Don't issue an error message (warning only) if a VIEW without "
    "presence of a key of the underlying table is used in queries with a "
    "LIMIT clause for updating. NO = Prohibit update of a VIEW, which "
    "does not contain a key of the underlying table and the query uses "
    "a LIMIT clause (usually get from GUI tools)",
    HINT_UPDATEABLE SESSION_VAR(updatable_views_with_limit),
    CMD_LINE(REQUIRED_ARG), updatable_views_with_limit_names, DEFAULT(true));
```
> ★ `check_key_in_view()`（`sql_view.cc:1890`）的 Doxygen 列出了三条满足之一即可：① 含底层可更新表的主键；② 含唯一键且列全 NOT NULL；③ 含底层表的**所有列**。

DD 里存 `view_is_updatable`（`YES` / `NO`），在 CREATE VIEW 时计算。不可更新的 view 做 DML 会报 `ER_NON_UPDATABLE_TABLE`。判定与 merge 能力无关（merge view 与 temptable view 都可能可更新）。

### 七、完整调用栈

```
open_table()                                     sql/sql_base.cc
  └─ [is_view 分支]
       ├─ check_and_update_table_version()
       ├─ open_and_read_view()                   sql/sql_view.cc
       │     └─ 从 DD 读定义 + 建 LEX + 填 Table_ref（algorithm / check_option / definer…）
       └─ parse_view_definition()                sql/sql_view.cc
             └─ 重新解析 SQL → Query_expression，挂到 Table_ref::derived

Query_block::prepare()
  └─ setup_tables()
       ├─ 合并路径：Query_block::merge_derived(thd, view_ref)   sql_resolver.cc
       │     ├─ 外层允许合并？（can_use_merged / can_not_use_merged）
       │     ├─ 三级优先级（ALGORITHM > hint > switch+heuristic）
       │     ├─ is_mergeable（含 CTE+RAND 规则）
       │     ├─ STRAIGHT_JOIN / MAX_TABLES 检查
       │     └─ set_merged() → 搬叶子表、下推条件、替换列
       └─ 物化路径：Table_ref::setup_materialized_derived(thd)
             └─ 建临时表 + optimize + 填充

DML 执行期
  └─ Table_ref::view_check_option(thd)           sql/table.cc
        └─ 用 prepare_check_option 构造的 Item 求值 → OK / SKIP / ERROR
```

---

## ★ 本机制里的工程实现技法

### 1. view 与 derived 共用 `merge_derived()`，用 trace 节点名区分

```cpp
Opt_trace_object trace_derived(trace,
                               derived_table->is_view() ? "view" : "derived");
```

同一个函数服务两个语义相近的对象，只在**可观测性输出**上区分。这是"复用优先"的设计，代价是两者共享了大部分（但不完全相同的）判定路径——例如 `allow_merge_derived` 那条 `is_view() ||` 就是为 view 开的口子。

### 2. 用 LEX 的存在性代替"是否是 view"的布尔

`OPEN_FOR_CREATE` 分支里那句注释说得很清楚：*"The LEX object is used by the executor and other parts of the code to detect the presence of a view"*。于是不得不在"其实没有 view 定义"的情况下造一个 dummy LEX。

这类"**对象存在性承载语义**"的做法在 MySQL 里很常见（参见 [`16_join_object_model.md`](16_join_object_model.md) 里 `where_cond` 的非法初值），优点是省一个字段，缺点是语义隐式化。

### 3. 三级优先级写进注释而不是配置

`merge_derived` 的优先级（ALGORITHM > hint > switch+heuristic）是**硬编码的 if/else 顺序**，只是把顺序写成了注释。这意味着改动优先级必须改代码顺序，且没有集中配置。

### 4. 三态返回值表达语义差异

`view_check_option()` 返回 `OK` / `SKIP` / `ERROR` 三态，把 `LOCAL` 与 `CASCADED` 的行为差异编码进返回值，让调用方（INSERT/UPDATE/LOAD）用统一的 switch 处理。这是一个"**用返回值多态代替参数分支**"的技巧。

### 5. `assert` 表达不变量

`assert(table_list->is_view() || table_list->table == nullptr)` —— view 永远没有 TABLE。这条不变量散落在 `sql_base.cc` 多处，用 assert 保证。

### 6. 多次引用的非确定性 CTE 强制物化

`is_mergeable()` 里那条 CTE + `UNCACHEABLE_RAND` 规则，是一个**正确性优先于性能**的典型：宁可放弃合并带来的优化机会，也要保证多处引用看到同一份结果。

---

## 可观测性

### optimizer trace

| 节点 | 内容 |
|---|---|
| `view`（或 `derived`） | `merge_derived` 的包裹节点，含表名、`select#` |
| `merged` / `materialized` | 最终选择 |
| `derived_condition_pushdown` | 条件下推是否成功（8.0.22+） |

★ 判断"view 为什么没合并"时，直接看这个节点——它是 `merge_derived` 所有 return false 分支的唯一出口记录。

**★ 完整节点名清单**（★ 找 view 根节点要先钻进两个外层包裹）：

| 节点名 | 位置 | 含义 |
|---|---|---|
| `join_preparation` | `sql_resolver.cc:216` | ★ 外层包裹（prepare 阶段） |
| `join_optimization` | `sql_optimizer.cc:359` | ★ 外层包裹（optimize 阶段） |
| **`view`** | `sql_derived.cc:831` / `merge_derived` | ★ 具名视图 |
| **`derived`** | 同上 | ★ 匿名派生表 |
| `table_function` | `sql_derived.cc:953` | JSON_TABLE 等（既非 view 也非 derived） |
| `condition_pushdown_to_derived` | `sql_derived.cc:1049` | 条件下推 |
| `pushdown_to_query_blocks` | `sql_derived.cc:1074` | 数组节点 |
| `pushing_past_window_functions` | `sql_derived.cc:1285` | 越不过窗口函数 |
| `pushing_past_group_by` | `sql_derived.cc:1326` | 越不过 GROUP BY |

```cpp
/* sql/sql_derived.cc:829 —— ★ 唯一的区分点 */
  Opt_trace_object trace_derived(trace, is_view() ? "view" : "derived");
  trace_derived.add_utf8_table(this)
      .add("select#", derived->first_query_block()->select_number)
      .add("materialized", true);
```
> ⚠️ **`sql/sql_view.cc` 里没有任何 `Opt_trace_object`**——视图打开阶段不写 trace，trace 只在 `sql_derived.cc` 的合并/物化阶段产生。

**★ EXPLAIN 倒排的源码依据**（⚠️ `select_type` 的 `DERIVED` **仍在**）：

```cpp
/* sql/sql_lex.cc:4322 —— DERIVED 的判定条件是 linkage，与"是否被 merge"无关 */
  } else if (this == master_query_expression()->first_query_block()) {
    if (linkage == DERIVED_TABLE_TYPE)
      return enum_explain_type::EXPLAIN_DERIVED;
    else
      return enum_explain_type::EXPLAIN_SUBQUERY;
  }
```
```cpp
/* sql/sql_lex.h:1283 —— 8.0.39 依然保留 EXPLAIN_DERIVED */
enum class enum_explain_type {
  EXPLAIN_NONE = 0, EXPLAIN_PRIMARY, EXPLAIN_SIMPLE,
  EXPLAIN_DERIVED,          /* ★ 仍在 */
  EXPLAIN_SUBQUERY, EXPLAIN_UNION, EXPLAIN_INTERSECT, EXPLAIN_EXCEPT,
  EXPLAIN_UNION_RESULT, EXPLAIN_INTERSECT_RESULT, EXPLAIN_EXCEPT_RESULT,
  EXPLAIN_UNARY_RESULT,
  EXPLAIN_MATERIALIZED,     /* ★ 给 semi-join 物化用，不是给物化派生表/视图用 */
  EXPLAIN_total
};
```
```cpp
/* sql/opt_explain.cc:1456 —— ★★ 视图被物化后，传统格式里连视图名都不显示 */
  if (tr->query_block_id() && tr->is_view_or_derived() &&
      !fmt->is_hierarchical()) {
    /* Derived table name generation */
    len = snprintf(namebuf, len - 1, "<derived%u>",
                   tr->query_block_id_for_explain());
    return func(namebuf, len);       /* ★ 显示 <derivedN> 而非视图名 */
  } else {
    return func(tr->alias, strlen(tr->alias));
  }
```
> ★ 一个刻意的历史例外（`opt_explain.cc:779`）：
> ```cpp
>     if (query_block->type() != enum_explain_type::EXPLAIN_DERIVED)
>       fmt->entry()->is_cacheable = query_block->is_cacheable();
> ```
> 即 **DERIVED 行不打 `UNCACHEABLE` 前缀**。

**★ EXPLAIN 对视图的权限闸门**（`sql/opt_explain.cc:1856` + `sql_view.cc:1388`）：

```cpp
/* opt_explain.cc:1856 */
static bool check_acl_for_explain(const Table_ref *table_list) {
  for (const Table_ref *tbl = table_list; tbl; tbl = tbl->next_global) {
    if (tbl->is_view() && tbl->view_no_explain) {
      my_error(ER_VIEW_NO_EXPLAIN, MYF(0));
      return true;
    }
  }
  return false;
}
```
```cpp
/* sql_view.cc:1382 —— ★ 缺底层表权限时"打标"而非立刻报错 */
    if (check_table_access(thd, SELECT_ACL, view_tables, false, UINT_MAX, true) ||
        check_table_access(thd, SHOW_VIEW_ACL, &view_no_suid, false, UINT_MAX, true))
      view_ref->view_no_explain = true;
```
> ★ 注释解释了为什么只扫顶层表链就够：*"views contained in merge-able views have been merged and brought up in the top list of tables"* —— **合并掉的视图会被"提上来"，只有物化的才作为独立节点存在**。

**★ 相关对象（DD / I_S）**：

```cpp
/* sql/dd/impl/system_views/views.cc:47 —— ★ I_S.VIEWS 是定义在 mysql.tables 上的系统视图 SQL */
  m_target_def.add_field(
      FIELD_VIEW_DEFINITION, "VIEW_DEFINITION",
      "IF(CAN_ACCESS_VIEW(sch.name, vw.name, vw.view_definer, vw.options)=TRUE,"
      "   vw.view_definition_utf8, '')");   /* ★ 无权限返回空串，不报错 */
  m_target_def.add_field(
      FIELD_SECURITY_TYPE, "SECURITY_TYPE",
      "IF (vw.view_security_type='DEFAULT', 'DEFINER', vw.view_security_type)");
                                            /* ★ DD 三值 → I_S 只暴露两值 */
  m_target_def.add_from("mysql.tables vw");
  m_target_def.add_where("AND vw.type = 'VIEW'");
```
```cpp
/* sql/sql_show.cc:2714 —— ★ SHOW CREATE VIEW 输出的是"重新 print 的 AST"，不是原文 */
  table->view_query()->unit->print(
      thd, buff,
      enum_query_type(QT_TO_ARGUMENT_CHARSET | QT_NO_DEFAULT_DB |
                      QT_HIDE_ROLLUP_FUNCTIONS));
```
> ★ 且输出用**建视图时记下的 client charset**（`sql_show.cc:1200`），不是当前会话的。

### EXPLAIN 倒排

| 你看到 | 说明 |
|---|---|
| 没有 view 名字、直接出现基表 | 合并成功 |
| `<derived2>` 或 view 名作为表名 | 物化路径 |
| `Using temporary` 出现在 view 相关 | 物化 |
| 条件下推成功 | 外层谓词出现在 view 内部的访问路径上 |

### 相关对象

| 对象 | 位置 |
|---|---|
| view 的定义 / algorithm / check option | `mysql.tables`（DD），`information_schema.views` 可读 |
| `Table_ref::effective_algorithm` | 运行期：UNDEFINED / MERGE / TEMPTABLE |
| `Table_ref::is_merged()` | `effective_algorithm == VIEW_ALGORITHM_MERGE` |

### 常用验证

```sql
-- 看 view 的 algorithm 是怎么被记下来的
SELECT TABLE_NAME, VIEW_DEFINITION, CHECK_OPTION, IS_UPDATABLE
FROM information_schema.views WHERE TABLE_NAME = 'v';

-- 看是否被合并（trace）
SET optimizer_trace='enabled=on';
SELECT * FROM v WHERE ...;
SELECT * FROM information_schema.optimizer_trace\G
```

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 让更多 view 能合并 | `Query_expression::is_mergeable()` 的条件（目前聚合/DISTINCT/UNION/LIMIT 都不能合并） |
| 增强条件下推 | `derived_condition_pushdown` 相关（8.0.22+） |
| 新增一种 ALGORITHM | `enum_view_algorithm` + `sql_yacc.yy` 的 `view_algorithm` 规则 + `merge_derived` 的分支 + DD 的枚举字段 |
| 改 view 的检查语义 | `prepare_check_option` / `view_check_option` |
| 让 view 参与代价决策 | 需要把"合并 vs 物化"改成代价比较，当前是纯规则 |

### 已知缺陷

- **`MAX_TABLES`（64）限制**：多个大 view 合并后表数超限即退化物化。
- **合并判定是纯规则**：不知道"合并后能省多少"，只判断"能不能"。
- **view 不能合并的常见情形很多**（聚合、DISTINCT、GROUP BY、UNION、LIMIT、窗口、外连接语义等），且分散在 `is_mergeable()` 与 `merge_derived()` 的多个 return false 分支里，没有集中清单。
- **嵌套 view 的合并限制**：源码 TODO 提到 `can_use_merged()` 目前会避免合并"包含在其它 view 里的 view"，这是一个已知的能力缺口。
- **`OPEN_FOR_CREATE` 的 dummy LEX** 是历史遗留的语义隐式化。

**★ 源码自认的 TODO / 缺陷（逐个带行号）**：

```cpp
/* ① sql/sql_view.cc:601 —— CREATE VIEW 的 prepare 代码被复制了两份 */
    /*
      @todo - the following code is duplicated in mysql_test_create_view.
              ensure that we have a single preparation function for create view.
    */
```
```cpp
/* ② sql/sql_view.cc:977 —— ALTER VIEW 不支持 CASCADE / RESTRICT */
    /*
      TODO: read dependence list, too, to process cascade/restrict
      TODO: special cascade/restrict procedure for alter?
    */
```
```cpp
/* ③ sql/sql_view.cc:1003 —— ★ 视图体 UTF8 打印有 bug，只能靠"拒绝建这种视图"兜底 */
  /*
    Our parser allows incorrect invalid UTF8 characters in literals.
    Due to this and due to some bugs in view body printing our UTF8
    version of view body might contain invalid characters. ...
    This is a temporary workaround to be removed once we stop accepting
    invalid UTF8 in literals and fix bugs in view body printing.
  */
```
```cpp
/* ④ sql/sql_view.cc:1910 —— ★ 可更新视图不支持 UNION，作者自己都在质疑调用点 */
  /*
    we do not support updatable UNIONs in VIEW, so we can check just limit of
    LEX::query_block.
    But why call this function from INSERT when we explicitly ignore it?
  */
```
```cpp
/* ⑤ sql/sql_derived.cc:924 —— 物化派生表读权限全量标记，未按引用裁剪 */
    // Set all selected fields to be read:
    // @todo Do not set fields that are not referenced from outer query
```
```cpp
/* ⑥ sql/sql_derived.cc:1269 —— 条件下推无法越过窗口函数 */
  @todo
  Introduce another condition (like WHERE and HAVING) which can be
  used to filter after window function execution.
```
```cpp
/* ⑦ sql/sql_derived.cc:1639 —— create_materialized_table 的 assert 不完整 */
  // @todo: Be able to assert !table->is_created() as well
```
```cpp
/* ⑧ sql/sql_derived.cc:1612 —— ★ hypergraph 与"提前物化 const 表"互斥 */
  // The hypergraph optimizer doesn't care about const tables, though, so it
  // prefers to do this at execution time (in fact, it will get confused and
  // crash if it has already been materialized).
```
```cpp
/* ⑨ sql/table.h:3711 —— algorithm 字段不是枚举，源码自己标了 todo */
      @todo Replace with an enum
  ulonglong algorithm{0};
```
```cpp
/* ⑩ sql/aggregate_check.h:489 —— ★ 合并后的视图无法做函数依赖分析 */
The above is fine for materialized views. For merged views, we cannot study the
query expression of the view, it has been merged (and scattered), so we use a
different rule: ...
```

**★ assert 表达的不变量**（分散收集）：

```cpp
  assert(is_view_or_derived() && !is_merged() && table == nullptr);   // sql_derived.cc:825
  assert((is_table_function() || derived_query_expression()) &&
         uses_materialization() && table);                            // sql_derived.cc:1640
  assert(is_view_or_derived() && uses_materialization());             // sql_derived.cc:1700
  assert(table && table->is_created() && !table->materialized);       // sql_derived.cc:1701
    assert(table->s->primary_key == MAX_KEY);                         // sql_derived.cc:1722（递归 CTE）
  assert(is_view());                                                  // table.cc:4610
  assert(view != nullptr && view != (LEX *)1);                        // table.h:3253（哨兵 1 被排除）
      assert(is_merged());  // Cannot be a materialized view          // table.h:3216
  assert(!view_ref->is_updatable());                                  // sql_view.cc:1636
```

### 社区边界澄清

- **没有物化视图（MATERIALIZED VIEW）对象**：社区版 MySQL 没有 PG/Oracle 那种"显式物化视图 + REFRESH"的对象。"物化"只是优化器对 view/derived 的一种内部实现，用户无法创建持久化的物化视图。
- **没有索引视图（indexed view）**：SQL Server 有，MySQL 没有。
- **没有视图的增量维护**。
- **view 的 algorithm 是 MySQL 特有的用户可见选项**，不要把 ALGORITHM 当作 SQL 标准能力。

**★ 语法层铁证：`CREATE MATERIALIZED VIEW` 不存在**

```
$ grep -c "MATERIALIZED" sql/sql_yacc.yy   →  0
```
> ⚠️ 全仓库检索 `materialized view` 命中的只是"**已经被物化的**视图/派生表"这种**运行时状态描述**，没有任何"物化视图对象"的语义：
> ```cpp
>   if (!is_merged())  // Base table or materialized view
>                      // sql/table.cc:6517 —— ★ 这里的 "materialized view" 指"走了 TEMPTABLE 的视图"
> ```
> ```cpp
>   // Find a materialized view inside another view.
>                      // sql/sql_union.cc:2014 —— 视图里嵌了个被物化的派生表
> ```
> ★ 但注意一个反直觉点（`sql/sql_admin.cc:873`）：**`CHECK TABLE` 强制视图走物化**，即使 `ALGORITHM=MERGE`。

**★ 扩展点：若要给 MySQL 加"真正的物化视图"，源码可证要动这些地方**：

| # | 要改 | 位置 | 说明 |
|---|---|---|---|
| 1 | 新增 token + 产生式 | `sql/sql_yacc.yy` | 目前 `MATERIALIZED` 0 命中，`view_algorithm` 只有三支（`:17447`） |
| 2 | SQL 层枚举扩容 | `enum enum_view_algorithm`（`table.h:2479`） | `is_mergeable()` 用 `algorithm == VIEW_ALGORITHM_TEMPTABLE` 短路（`table.cc:6489`） |
| 3 | **DD 层枚举同步扩容** | `dd::View::enum_algorithm`（`dd/types/view.h:52`）+ `dd_get_old/new_view_algorithm_type()`（`dd_view.cc:126-166`） | ★ 两个转换函数是**全 switch + `assert(false)`**，漏改则 debug 构建直接 abort |
| 4 | 持久化 schema 加列 | `sql/dd/impl/tables/tables.cc:130-155` | 纯静态 `add_field`；**没有** refresh 时间戳 / last_refreshed / staleness 字段 |
| 5 | I_S 同步 | `system_views/views.h:37` 的 `enum_fields` + `views.cc:38` 的 `add_field` | I_S.VIEWS 只是对 `mysql.tables` 的投影 |
| 6 | 复用现有物化骨架 | `setup_materialized_derived_tmp_table`（`sql_derived.cc:820`）→ `create_materialized_table`（`:1636`）→ `materialize_derived`（`:1698`） | ★ 已是一套完整的"建临时表 → 填充 → 只读"流水线；`sql_admin.cc:877` 证明"强制某视图走物化"只需两行 |
| 7 | 观测点可复用 | `sql_derived.cc:831` 的节点名二选一、`opt_explain.cc:1461` 的 `<derivedN>` 命名 | — |

> ⚠️ 以下为**基于结构的推断**（源码未直接给出）：需新增刷新调度子系统（`sql/` 下本次检索未发现任何 view refresh 逻辑）；需把临时表语义换成持久化表（涉及 `TMP_TABLE_ALL_COLUMNS` 与 `table->s->tmp_table = NON_TRANSACTIONAL_TMP_TABLE`，`sql_derived.cc:890`）；权限与 `view_no_explain`（`sql_view.cc:1388`）语义需重新定义。

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| `mysql_make_view()` 是打开视图的主函数 | **不存在** → 全仓仅 `sql_base.cc:143` 一行注释提及 |
| `update_view_metadata()` 在 `sql_view.cc` | 在 **`sql/dd_sql_view.cc:719`**（I_S 视图元数据刷新，与 view 打开无关） |
| `Item_direct_view_ref` 是列替换的实现类 | **不存在** → 仅 `table.cc:4555` 注释残留；真实机制是 `Field_translator[]` + `Item_ref` |
| 有 `bool is_view` 成员 | **没有** → `is_view()` = `view != nullptr` |
| `Table_ref::view_check_option` 是字段 | 是**成员函数**（`table.cc:4736`）；字段叫 `with_check` 与 `check_option` |
| `WITH CHECK OPTION` 缺省是 CASCADED | 缺省是 **`VIEW_CHECK_NONE`**；裸 `WITH CHECK OPTION` 才映射到 CASCADED |
| `enum_view_suid` / `enum_view_check_option` 是枚举 | **都是 `#define` 宏**；只有 `enum_view_algorithm` 是真枚举 |
| `SQL SECURITY` 缺省是 INVOKER | 缺省是 **`VIEW_SUID_DEFAULT = 2`**（第三态）；`SHOW CREATE VIEW` 按布尔真值打印 → DEFINER |
| SQL 层与 DD 层用同一套 algorithm 枚举 | **两套**：SQL 层 0-based（UNDEFINED=0），DD 层 1-based（`VA_UNDEFINED=1`），靠转换函数衔接 |
| `replace_leaf_tables()` | **不存在** → 真实实现是内联叶子链替换 + `remap_tables()` + `fix_tables_after_pullout()` |
| 8.0 的 `select_type` 已取消 `DERIVED` | **还在**（`sql_lex.h:1287`）；`EXPLAIN_MATERIALIZED` 是给 semi-join 用的 |
| 还有 `mysql.view` 系统表 | **不存在** → 视图定义存在 `mysql.tables` 的 `view_*` 列里 |
| `updatable_views_with_limit` 已移除 | **仍在**（`sys_vars.cc:5066`，默认 true，`HINT_UPDATEABLE`） |
| `open_and_read_view` 只有一个调用点 | **3 个**：`sql_base.cc:3290`、`sql_base.cc:3772`、`sql_view.cc:295`（DDL 期用 decoy） |
| view/derived 是两个独立成员 | 共用 `derived`；view 额外持有 `LEX *view`，`is_derived()` = "unnamed view" |
| 源码注释会对比 PG/Oracle 视图 | **grep 零命中**（`sql_view.cc` / `sql_derived.cc` / `sql_base.cc`） |
| `sql_view.cc` 写 optimizer trace | **不写** → trace 只在 `sql_derived.cc` 的合并/物化阶段产生 |

---

## 参考

**论文 / 理论**

- 《Query Rewrite for Views》(SQL 标准与经典 view merging 讨论) —— 视图查询改写的理论基础
- SQL:2016 Feature F711/F712 —— 可更新视图与 `WITH CHECK OPTION` 的标准定义

**官方文档**

- MySQL 8.0 Reference Manual, "CREATE VIEW Statement"（ALGORITHM / WITH CHECK OPTION / 可更新性判定规则）
- MySQL 8.0 Reference Manual, "Updatable and Insertable Views"
- MySQL 8.0 Reference Manual, "Optimizing Derived Tables, View References, and Common Table Expressions"

**相关文档**

- [`logical/04_logical_join.md`](logical/04_logical_join.md) —— derived merge 的 12 步细节（view 共用）
- [`../runtime/08_materialization.md`](../runtime/08_materialization.md) —— 物化路径
- [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md) —— 条件下推到派生表
- [`19_set_operation.md`](19_set_operation.md) —— view 内的 set operation
- [`../05_contextualize.md`](../05_contextualize.md) —— view 定义的解析与 contextualize
- [`16_join_object_model.md`](16_join_object_model.md) —— `Table_ref` 与 `Table_ref::derived`
