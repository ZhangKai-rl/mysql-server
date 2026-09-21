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

#### 6.2 调用点分布

`view_check_option()` 在 `sql_insert.cc`、`sql_update.cc`、`sql_load.cc` 里被调用——即 **INSERT / UPDATE / LOAD DATA 都可能触发 CHECK OPTION 检查**。

#### 6.3 可更新性

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

### 社区边界澄清

- **没有物化视图（MATERIALIZED VIEW）对象**：社区版 MySQL 没有 PG/Oracle 那种"显式物化视图 + REFRESH"的对象。"物化"只是优化器对 view/derived 的一种内部实现，用户无法创建持久化的物化视图。
- **没有索引视图（indexed view）**：SQL Server 有，MySQL 没有。
- **没有视图的增量维护**。
- **view 的 algorithm 是 MySQL 特有的用户可见选项**，不要把 ALGORITHM 当作 SQL 标准能力。

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
