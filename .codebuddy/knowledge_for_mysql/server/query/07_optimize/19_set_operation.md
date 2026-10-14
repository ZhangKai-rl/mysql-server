# 19 Set Operation 的优化侧：UNION / INTERSECT / EXCEPT 是怎么被优化和执行的

> 基于 MySQL 8.0.39。本篇覆盖 `Query_term` 树在**优化期**与**代码生成期**的处理：`Query_expression::optimize()` 如何递归、`create_access_paths()` 如何决定"流式追加 vs 物化去重"、`setup_materialize_set_op()` 如何决定哪些 operand 需要去重、INTERSECT / EXCEPT 如何用临时表计数器实现。
>
> **边界**：`Query_term` 的**解析**（8.0.31 重构、6 层 BNF 到查询树、`merge_descendants`、`Query_terms` 的游标分配、`validate_tree`/`label` 标签分配、`pushdown_limit_order_by` 下推）见 [`../05_contextualize.md`](../05_contextualize.md)。**本篇只写解析之后的优化与代码生成**。
> 物化与临时表的运行期细节见 [`../runtime/08_materialization.md`](../runtime/08_materialization.md)，迭代器见 [`../09_executor_iterator.md`](../09_executor_iterator.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、Query_term 的四个类型](#一query_term-的四个类型)
  - [二、Query_expression::optimize 逐段](#二query_expressionoptimize-逐段)
  - [三、create_access_paths：流式还是物化](#三create_access_paths流式还是物化)
  - [四、setup_materialize_set_op：谁需要去重](#四setup_materialize_set_op谁需要去重)
  - [五、INTERSECT / EXCEPT 的计数器算法](#五intersect--except-的计数器算法)
  - [六、ORDER BY / LIMIT 的处理与 fake query block](#六order-by--limit-的处理与-fake-query-block)
  - [七、完整调用栈](#七完整调用栈)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 优化期要回答的三个问题

一个 set operation（`UNION` / `INTERSECT` / `EXCEPT`）在优化期要决定：

1. **能不能流式（streaming）**：行能否直接从各 operand 流出去，而不必先落进临时表？
2. **哪些 operand 需要去重**：`UNION ALL` 不需要，`UNION DISTINCT` 需要；混合出现时怎么办？
3. **INTERSECT / EXCEPT 怎么算**：它们无法流式，必须物化，且需要一套计数机制。

三个问题的答案全部落在 `Query_expression::create_access_paths()` 与 `Query_term_set_op::setup_materialize_set_op()` 里。

### 一个反直觉的开场

```sql
CREATE TABLE t (a INT);
EXPLAIN FORMAT=TREE SELECT a FROM t UNION ALL SELECT a FROM t;
-- -> Append
--     -> Table scan on t
--     -> Table scan on t          ← 没有临时表

EXPLAIN FORMAT=TREE SELECT a FROM t UNION ALL SELECT a FROM t ORDER BY a;
-- -> Sort: t.a
--     -> Table scan on <union1,2>  ← ★ 加了 ORDER BY，UNION ALL 也变成临时表
```

第二条里 `UNION ALL` 明明不需要去重，却出现了临时表 `<union1,2>`。原因就在 `create_access_paths()` 开头那条判定：

```cpp
// If we're sorting, we currently put it in a real table no matter what.
// This is a legacy decision, because we used to not know whether filesort
// would want to refer to rows in the table after the sort (sort by row ID).
// We could probably be more intelligent here now.
```

**注释自己承认这是"遗留决定"并且"我们现在大概可以更聪明"**。这是理解 UNION 性能的一个关键点：任何顶层 `ORDER BY` 都会让整个 UNION 物化。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | 只有 UNION；`Query_expression` 直接持 `next_query_block()` 链表 |
| 8.0（早期） | `Query_term` 引入，与 `Query_expression::first_query_block()` 链表**并存**（两套表示，注释里承认是 `TODO` 要合并） |
| 8.0.31 | **INTERSECT / EXCEPT 进入社区版**；`Query_term_set_op` 派生出 `Query_term_union` / `Query_term_intersect` / `Query_term_except`；`Query_terms` 游标与标签机制成型 |
| 8.0.31+ | `pushdown_limit_order_by` 把 LIMIT/ORDER BY 下推到 operand；INTERSECT/EXCEPT 因语义限制只能在有 LIMIT 时下推 |

---

## 理论基础

### UNION ALL vs UNION DISTINCT：追加 vs 去重

| | 语义 | 物理实现 | 能否流式 |
|---|---|---|---|
| `UNION ALL` | 多集合并（保留重复） | 依次读完每个 operand 直接输出 | ✅ 能（追加即可） |
| `UNION DISTINCT` | 集合并（去重） | 需要知道"这行是不是已经出现过" | ❌ 不能（必须落表或用哈希） |

MySQL 的实现是：**所有 operand 都写进同一张临时表，用临时表的唯一键去重，最后统一扫描输出**。这与 PG 的 `Append` + `HashAggregate` 或 `Unique` 算子思路不同——MySQL 没有独立的 "distinct/unique" 算子，去重能力被放在**临时表**这个对象上。

### 为什么 MySQL 没有独立的去重算子

历史原因：临时表（`TABLE`）是 MySQL 最古老、最通用的中间结果载体，天然支持"唯一索引"这个去重手段。加一个 `Unique` 迭代器需要重新实现内存/磁盘双模的哈希表，8.0 的迭代器化改造没有做这件事。

后果是：**UNION DISTINCT 的代价里包含一次完整的物化**，即使行数很少。

### INTERSECT / EXCEPT 的三种算法选择

| 算法 | 适用 | MySQL 用不用 |
|---|---|---|
| **哈希半连接** | INTERSECT/EXCEPT，内存足够 | ❌ 不用（没有通用 hash set 算子） |
| **排序归并** | 两路有序输入 | ❌ 不用 |
| **计数器（multiset counting）** | 支持 `ALL` 语义（重复度） | ✅ 用 |

★ 关键在 `ALL` 语义：`INTERSECT ALL` 与 `EXCEPT ALL` 不是简单的集合运算，而是**多重集运算**——`{1,1,2} INTERSECT ALL {1,2}` 结果是 `{1,2}`（重复度取 min）。要正确表达重复度，光靠哈希去重不行，必须**计数**。

所以 MySQL 选了"临时表 + 每行一个计数器"，一套机制同时覆盖 `DISTINCT` 与 `ALL` 两种语义。这是本篇第五章的重点。

### 他库对比

| 库 | UNION DISTINCT | INTERSECT / EXCEPT |
|---|---|---|
| **MySQL 8.0** | 临时表唯一键 | 临时表 + 计数器（8.0.31+） |
| **PostgreSQL** | `Append` + `HashAggregate`/`Sort+Unique`，代价驱动 | `SetOp` 算子（`HashSetOp` / `SortedSetOp`），**代价驱动**选哈希还是排序 |
| **Oracle** | `HASH UNIQUE` / `SORT UNIQUE` | `HASH INTERSECT` / `INTERSECTION` |
| **SQL Server** | `Merge Join` / `Hash Match (Union)` | 同上，且支持 set-op 并行 |

MySQL 的特点仍然是**规则驱动**：`streaming_allowed` 三个条件一判就定，不比较"流式追加"与"物化去重"的代价。

---

## 核心实现

### 一、Query_term 的四个类型

> 定义于 `sql/query_term.h`。

```
Query_term                      基类：所有节点
 ├─ Query_block                 叶子（普通 SELECT）
 └─ Query_term_set_op           集合操作的基类
     ├─ Query_term_union        QT_UNION
     ├─ Query_term_intersect    QT_INTERSECT
     ├─ Query_term_except       QT_EXCEPT
     └─ Query_term_unary        一元包装（如只有 ORDER BY/LIMIT 的外壳）
```

`Query_term_set_op` 的关键成员：

| 成员 | 含义 |
|---|---|
| `m_children` | 各个 operand |
| `m_last_distinct` | **最后一个需要去重的 operand 下标**（决定混合 UNION ALL/DISTINCT 的策略） |
| `m_is_materialized` | 该 set-op 是否被要求物化（例如作为派生表时必须物化） |

★ `m_last_distinct` 是理解混合 UNION 的关键。举例：

```sql
SELECT a FROM t1              -- operand 0
UNION     SELECT a FROM t2    -- operand 1（DISTINCT）
UNION ALL SELECT a FROM t3    -- operand 2（ALL）
UNION ALL SELECT a FROM t4    -- operand 3（ALL）
```

这里 `m_last_distinct == 1`，即"下标 ≤ 1 的 operand 需要去重，后面的可以追加"。

**★★ `Query_term_type` 枚举全文**（`sql/query_term.h:88`）：

```cpp
/// Node type of the query term tree nodes
enum Query_term_type {
  /// Represents Query specification, table value constructor and explicit table
  QT_QUERY_BLOCK,
  /// Represents a query primary with parentesized query expression body with
  /// order by clause and/or limit/offset clause. If none of order by
  /// or limit is present, we collapse this level of parentheses.
  QT_UNARY,
  /// Represents the three set operations. Nodes are N-ary, i.e. a node can hold
  /// two or more operands.
  QT_INTERSECT,
  QT_EXCEPT,
  QT_UNION
};

/// Query term iterator template argument type: how to visit nodes in tree
enum Visit_order { QTC_POST_ORDER, QTC_PRE_ORDER };
/// Query term iterator template argument type: whether to visit leaf nodes
enum Visit_leaves { VL_VISIT_LEAVES, VL_SKIP_LEAVES };
```

> ⚠️ **不存在 `union_type` / `UT_UNION` / `UT_INTERSECT` / `UT_EXCEPT`**——全 `sql/` 目录 0 命中。真名就是 `Query_term_type`。
> ★ **层级是 3 层**：`Query_term`（抽象）→ `Query_term_set_op`（抽象，n-ary 公共基类）→ 4 个具体类。而 **`Query_block` 也直接继承 `Query_term`**，是第 5 种节点（`QT_QUERY_BLOCK`）——所以准确说是 **5 个节点类型、4 个 set-op 子类**。

**★★ `Query_term_set_op` 完整定义**（★ `m_last_distinct` 的载体，`sql/query_term.h:403`）：

```cpp
/// Common base class for n-ary set operations, including unary.
class Query_term_set_op : public Query_term {
  /// Replaces the old "fake" query block for post processing result set with
  /// ORDER BY, LIMIT.
  Query_block *m_block{nullptr};

 protected:
  Query_term_set_op(MEM_ROOT *mem_root) : m_children(mem_root) {}

 public:
  /// Tree structure. Cardinality is one for unary, two or more for UNION,
  /// EXCEPT, INTERSECT
  mem_root_deque<Query_term *> m_children;
  /**
     Index of last query expression which has <set-op> DISTINCT on its left. In
     a list of <set-op>ed blocks, UNION is left-associative; so UNION DISTINCT
     eliminates duplicates in all blocks up to the first one on its right
     included. Which is why we only need to remember that query block. Is -1
     for Unary.
  */
  int64_t m_last_distinct;
  /**
    Presently only needed by EXCEPT set operator: the index of the first
    DISTINCT set operand: minimum legal value is 1. If not DISTINCT, it should
    have the value std::numeric_limits<int64_t>::max(). The value is set
    in PT_set_operation::merge_descendants.
  */
  int64_t m_first_distinct;
  /**
    true if the result of this set operation is materialized. A priori true
    unless we have a pure UNION ALL.
  */
  bool m_is_materialized{true};

  Query_block *query_block() const override { return m_block; }
  bool set_block(Query_block *b) { assert(!m_block); if (b == nullptr) return true; m_block = b; return false; }
  size_t child_count() const override { return m_children.size(); }
  bool is_unary() const { return term_type() == QT_UNARY; }
  /**
    Check if this set operation has a mix of DISTINCT and ALL.
    @return true if so.  Always false for unary
   */
  bool has_mixed_distinct_operators();
  Mem_root_array<MaterializePathParameters::QueryBlock>
  setup_materialize_set_op(THD *thd, TABLE *dst_table, bool union_distinct_only,
                           bool calc_found_rows);
};
```

> ★★ **`m_block` 就是"旧 `fake_query_block` 的替代物"**——源码注释原文：*"Replaces the old \"fake\" query block for post processing result set with ORDER BY, LIMIT"*。旧版只有**一个** fake block 挂在 `Query_expression` 上，**新版每个非叶节点各有一个 companion block**，所以能支持任意多层嵌套的 `ORDER BY`/`LIMIT`。
> ⚠️ **注释与代码不一致**：`m_last_distinct` 的注释写"Is -1 for Unary"，但 `Query_term_unary` 构造函数里实际写的是 **`m_last_distinct = 0;`**（`query_term.h:566`）。

**★ `has_mixed_distinct_operators()`**（★ 混合判定的精确定义，`query_term.cc:189`）：

```cpp
bool Query_term_set_op::has_mixed_distinct_operators() {
  return (m_last_distinct > 0) &&
         (static_cast<size_t>(m_last_distinct) < (m_children.size() - 1));
}
```

> ★ 语义：`m_last_distinct > 0`（有 DISTINCT）**且** `m_last_distinct < N-1`（最后一个操作数不是 DISTINCT）。上面那个例子 `m_last_distinct=1`、`N=4`，`1 < 3` ⇒ 混合。

**★★★ `m_last_distinct` 的全部 13 处使用点**（★ 一个下标贯穿 prepare / optimize / 物化 / 打印四个阶段）：

| # | 位置 | 作用 |
|---|---|---|
| 1 | `sql_union.cc:204` | `m_last_operation_is_distinct = op->m_last_distinct > 0`（INTERSECT/EXCEPT 临时表是否"最后一步 DISTINCT"） |
| 2 | `sql_union.cc:356` | `const bool distinct = parent->m_last_distinct > 0`（建 set-op 临时表时是否唯一去重） |
| 3 | `sql_union.cc:720` | 决定 `m_is_materialized`（prepare 阶段） |
| 4 | `sql_union.cc:1281` | `m_last_distinct == 0 ? 0 : m_last_distinct + 1`（**流式起点**：最后一个 DISTINCT 之后才开始 append） |
| 5 | `sql_union.cc:1393` | `activate_deduplication = idx <= m_last_distinct \|\| term_type() != QT_UNION`（**谁需要去重**） |
| 6 | `sql_union.cc:1413` | `if (idx == m_last_distinct && idx > 0 && union_distinct_only) break`（**物化到此为止**） |
| 7 | `sql_union.cc:1497` | `m_last_distinct == 0`（**能否把 LIMIT 下推到流式路径**） |
| 8 | `query_term.cc:190` | `has_mixed_distinct_operators()` |
| 9 | `query_term.cc:204` | debugPrint |
| 10-11 | `sql_lex.h:851/859` | `last_distinct()` / `has_top_level_distinct()` |
| 12 | `sql_lex.cc:2604` | 反打印 SQL：`if (i+1 > qts->m_last_distinct) str->append("all ")` |
| 13-17 | `parse_tree_nodes.cc:1577-1649` | `merge_descendants()` 里计算/写入（含折叠逻辑） |

**★ INTERSECT 的 DISTINCT 优先级最高**（`parse_tree_nodes.cc:1647`）：

```cpp
  if (last_distinct > 0 && op == QT_INTERSECT)  // distinct always wins
    last_distinct = setop->m_children.size() - 1;
```

> ★ 只要 INTERSECT 里有**任意一个** DISTINCT，就把 `m_last_distinct` 推到最后一个下标——因为 INTERSECT 一旦去重，ALL 与否就不重要了。

### 二、Query_expression::optimize 逐段

> `sql/sql_union.cc`。这是 unit 级的递归入口。

#### 2.1 逐 query block 优化 + 代价累加

```cpp
for (Query_block *query_block = first_query_block(); query_block != nullptr;
     query_block = query_block->next_query_block()) {
  thd->lex->set_current_query_block(query_block);
  if (set_limit(thd, query_block)) return true;
  if (query_block->optimize(thd, finalize_access_paths)) return true;

  if (contributes_to_rowcount_estimate(query_block))
    estimated_rowcount += (query_block->is_implicitly_grouped() ||
                           query_block->join->group_optimized_away)
                              ? 1
                              : query_block->join->best_rowcount;

  estimated_cost += query_block->join->best_read;

  if (query_result() != nullptr) {
    query_result()->estimated_rowcount = estimated_rowcount;
    query_result()->estimated_cost = estimated_cost;
  }
}
```

三点值得注意：

- **每个 query block 单独优化**（各自的 JOIN 各自走一遍 `JOIN::optimize`），然后**代价是简单相加**。没有跨 operand 的联合优化。
- **隐式分组 / GROUP BY 被消掉**时行数按 1 算——这与 [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md) 里 `group_optimized_away` 呼应。
- **边优化边更新 `query_result()->estimated_rowcount`**，源码注释解释了原因：递归 CTE 优化自己的递归部分时，需要知道"非递归部分会产出多少行"（`Table_ref::fetch_number_of_rows()` 会来读这个值）。这是一处**优化期跨 query block 的信息传递**。

#### 2.2 相关子查询的行数保护

```cpp
if ((uncacheable & UNCACHEABLE_DEPENDENT) && estimated_rowcount <= 1) {
  estimated_rowcount = PLACEHOLDER_TABLE_ROW_ESTIMATE;
}
```

若 unit 依赖外层引用且估算行数 ≤ 1，**故意抬高估算**，避免被当成常量表。注释说明只测 `UNCACHEABLE_DEPENDENT` 一位（不测 `UNCACHEABLE_SIDEEFFECT`）。

#### 2.3 set-op 自身的优化

```cpp
if (!is_simple()) {
  if (optimize_set_operand(thd, this, query_term())) return true;
  if (set_limit(thd, query_term()->query_block())) return true;
  if (!is_union()) query_result()->set_limit(select_limit_cnt);
}
```

`optimize_set_operand` 负责 set-op 层面的优化（含 LIMIT/ORDER BY 处理）。注意最后一行：**非 UNION（即 INTERSECT/EXCEPT）才需要显式设置 limit**，因为它们的语义不允许随意截断。

**★ `optimize_set_operand()` 完整源码**（⚠️ 真名核查：**不存在** `Query_term::optimize()` / `Query_term_union::optimize()`——`query_term.cc` 全文无 `optimize`。真身是这个 **static 自由函数**，`sql/sql_union.cc:915`）：

```cpp
static bool optimize_set_operand(THD *thd, Query_expression *qe,
                                 Query_term *qt) {
  if (qt->term_type() == QT_QUERY_BLOCK) return false;  // done already
  Query_term_set_op *qts = down_cast<Query_term_set_op *>(qt);
  thd->lex->set_current_query_block(qts->query_block());

  // LIMIT is required for optimization
  if (qe->set_limit(thd, qts->query_block())) return true;

  if ((qts->is_unary() || qts->m_is_materialized) &&      /* ★ 纯 UNION ALL 的后处理块不 optimize */
      qts->query_block()->optimize(thd,
                                   /*finalize_access_paths=*/true))
    return true;
  for (Query_term *child : qts->m_children) {
    if (optimize_set_operand(thd, qe, child)) return true;
  }
  return false;
}
```

> ★★ **`m_is_materialized == false` 的后处理块根本不 optimize**——纯 UNION ALL 不需要，因为没人会去读那张临时表。

**★★ 相关子查询的行数保护**（⚠️ **不存在** `set_found_rows_for_union()`，真身是 `optimize()` 内联的一段，`sql_union.cc:1043`）：

```cpp
  if ((uncacheable & UNCACHEABLE_DEPENDENT) && estimated_rowcount <= 1) {
    /*
      This depends on outer references, so optimization cannot assume that all
      executions will always produce the same row. So, increase the counter to
      prevent that this table is replaced with a constant.
      Not testing all bits of "uncacheable", as if derived table sets user
      vars (UNCACHEABLE_SIDEEFFECT) the logic above doesn't apply.
    */
    estimated_rowcount = PLACEHOLDER_TABLE_ROW_ESTIMATE;
  }
```

> ★ 语义：一个依赖外层值的 unit 若估出 ≤ 1 行，不能让它被当成常量表优化掉——因为不同次执行可能产出不同行数。抬到 `PLACEHOLDER_TABLE_ROW_ESTIMATE` 阻止这个替换。

**★★ INTERSECT / EXCEPT 的右操作数不计入行数估计**（★ 这是一个很容易漏的优化，`sql_union.cc:945`）：

```cpp
static bool contributes_to_rowcount_estimate(Query_block *qb) {
  if (qb->parent() == nullptr) return true;
  Query_term *query_term = qb;
  // See if this query block is contained in a right side of an INTERSECT
  // or EXCEPT operation anywhere in tree. If so, we can ignore its count.
  Query_term_set_op *parent = query_term->parent();
  while (parent != nullptr) {
    const auto term_type = parent->term_type();
    if ((term_type == QT_EXCEPT || term_type == QT_INTERSECT) &&
        parent->m_children[0] != query_term)
      return false;    /* ★ 右操作数只会让结果变小，不加 */
    // Even if we are on the left side of an INTERSECT, EXCEPT, the set
    // operation itself could still be in a non-contributing side higher up,
    // so continue checking.
    query_term = parent;
    parent = parent->parent();
  }
  return true;
}
```

> ★★ 注意 while 循环：**即使在某个 INTERSECT/EXCEPT 的左边，整个 set-op 也可能处在更高层的非贡献侧**，所以要一路向上检查。

#### 2.4 三条代码生成路径

```cpp
if (thd->lex->m_sql_cmd != nullptr &&
    thd->lex->m_sql_cmd->using_secondary_storage_engine()) {
  create_access_paths(thd);                       // 二级引擎：不支持 unfinished materialization
} else if (estimated_rowcount <= 1 ||
           use_iterator(materialize_destination, query_term())) {
  create_access_paths(thd);                       // 常量表 或 无法直接物化
} else if (materialize_destination != nullptr &&
           can_materialize_directly_into_result()) {
  m_query_blocks_to_materialize = set_operation()->setup_materialize_set_op(
      thd, materialize_destination,
      /*union_distinct_only=*/false, calc_found_rows);   // ★ 直接物化到调用者的表
} else {
  assert(!is_recursive());
  create_access_paths(thd);
}
```

第二条路径的注释很关键：**常量表（estimated_rowcount ≤ 1）不走 unfinished materialization**，因为 `optimize_derived()` 想在优化期就跑一次这个查询拿到值，需要迭代器。

第三条是**优化**：如果调用者（例如外层派生表）已经准备好了一张目标表，就直接物化进去，省掉一次中间临时表。

### 三、create_access_paths：流式还是物化

> `sql/sql_union.cc`。这是本篇的核心函数。

#### 3.1 streaming_allowed 的两个否定条件

```cpp
bool streaming_allowed = true;
if (global_parameters()->order_list.size() != 0 ||
    (!is_simple() && set_operation()->m_is_materialized)) {
  // If we're sorting, we currently put it in a real table no matter what.
  // This is a legacy decision, because we used to not know whether filesort
  // would want to refer to rows in the table after the sort (sort by row ID).
  // We could probably be more intelligent here now.
  streaming_allowed = false;
} else if ((thd->lex->sql_command == SQLCOM_INSERT_SELECT ||
            thd->lex->sql_command == SQLCOM_REPLACE_SELECT) &&
           thd->lex->unit == this) {
  // If we're doing an INSERT or REPLACE, and we're not outputting to
  // a temporary table already (ie., we are the topmost unit), then we
  // don't want to insert any records before we're done scanning. Otherwise,
  // we would risk incorrect results and/or infinite loops, as we'd be seeing
  // our own records as they get inserted.
  // @todo Figure out if we can check for OPTION_BUFFER_RESULT instead;
  //       see bug #23022426.
  streaming_allowed = false;
}
```

| 条件 | 为什么禁止流式 |
|---|---|
| **顶层有 ORDER BY** | 遗留决定（注释自陈"大概可以更聪明"）；filesort 可能需要按 row ID 回表取行 |
| **`m_is_materialized`** | 调用方（如派生表）明确要求物化 |
| **INSERT ... SELECT / REPLACE ... SELECT 且是顶层 unit** | **正确性**：边扫边插会读到自己插入的行，导致结果错误甚至死循环。注释给了 bug 号 23022426 |

第三条是典型的"**自读自写（Halloween Problem）**"防护。

#### 3.2 混合 UNION ALL / DISTINCT 的两种策略

```
streaming_allowed == true：
    需要去重的 operand（下标 ≤ m_last_distinct） → 物化进临时表
    其余 UNION ALL 的 operand                    → 由 AppendIterator 流式追加
    最终：Materialize(去重部分) + Append(ALL 部分)

streaming_allowed == false：
    全部 operand → 一张临时表
    最终：Table scan on <union...>
```

源码注释明说：*"If we cannot stream, our strategy for mixed UNION ALL/DISTINCT becomes a bit different; see MaterializeIterator for details."*

**★★ `create_access_paths()` 全文**（`sql/sql_union.cc:1420`，本篇核心）：

```cpp
void Query_expression::create_access_paths(THD *thd) {
  if (is_simple()) {
    JOIN *join = first_query_block()->join;
    assert(join && join->is_optimized());
    m_root_access_path = join->root_access_path();
    return;
  }

  bool streaming_allowed = true;
  if (global_parameters()->order_list.size() != 0 ||
      (!is_simple() && set_operation()->m_is_materialized)) {
    streaming_allowed = false;
  } else if ((thd->lex->sql_command == SQLCOM_INSERT_SELECT ||
              thd->lex->sql_command == SQLCOM_REPLACE_SELECT) &&
             thd->lex->unit == this) {
    streaming_allowed = false;
  }

  ha_rows offset = global_parameters()->get_offset(thd);
  ha_rows limit = global_parameters()->get_limit(thd);
  if (limit + offset >= limit) limit += offset;
  else limit = HA_POS_ERROR;
  const bool calc_found_rows =
      (first_query_block()->active_options() & OPTION_FOUND_ROWS);

  Mem_root_array<AppendPathParameters> *union_all_sub_paths =
      new (thd->mem_root) Mem_root_array<AppendPathParameters>(thd->mem_root);

  if (!streaming_allowed || !is_simple()) {
    AppendPathParameters param;
    param.path = make_set_op_access_path(
        thd, /*parent*/ nullptr, m_query_term,
        streaming_allowed ? union_all_sub_paths : nullptr, calc_found_rows);
    /* ★★ 关键分叉：union_all_sub_paths 非空 ⇒ 内部会把「m_last_distinct 之后的 ALL 块」
       追加进这个数组，而不是塞进 Materialize */
    param.join = nullptr;
    if (!streaming_allowed) union_all_sub_paths->push_back(param);
  }

  assert(!union_all_sub_paths->empty());
  if (union_all_sub_paths->size() == 1) {
    m_root_access_path = (*union_all_sub_paths)[0].path;
  } else {
    assert(streaming_allowed);
    m_root_access_path = NewAppendAccessPath(thd, union_all_sub_paths);
  }

  if (streaming_allowed && (limit != HA_POS_ERROR || offset != 0) &&
      (is_simple() || set_operation()->m_last_distinct == 0)) {
    m_root_access_path = NewLimitOffsetAccessPath(
        thd, m_root_access_path, limit, offset, calc_found_rows,
        /*reject_multiple_rows=*/false, &send_records);
  }
}
```

> ★★ **LIMIT 下推只在"流式 + 纯 ALL"（`m_last_distinct == 0`）时发生在这一层**。一旦有 DISTINCT，输出行数未知，LIMIT 只能由物化路径或后处理 block 的 JOIN 处理——否则会重复截断。

**★ `make_set_op_access_path()`：流式的具体切分点**（`:1240`）：

```cpp
      if (union_all_subpaths != nullptr) {
        assert(parent == nullptr);
        TABLE *dest = qts->m_children[0]->setop_query_result_union()->table;
        size_t start_idx =
            qts->m_last_distinct == 0 ? 0 : qts->m_last_distinct + 1;
        for (size_t i = start_idx; i < qts->m_children.size(); ++i) {
          // append UNION ALL blocks that follow last UNION [DISTINCT]
          Query_term *const term = qts->m_children[i];
          ...
          param.path = NewStreamingAccessPath(thd, child_path, join,
                                              &join->tmp_table_param, dest,
                                              /*ref_slice=*/-1);
          /* ★ UNION ALL 块包一层 StreamingIterator：只拷贝字段，不真写临时表 */
          param.join = join;
          CopyBasicProperties(*join->root_access_path(), param.path);
          union_all_subpaths->push_back(param);
        }
      }
```

**★ UNION 独有的 LIMIT 下推到物化**（`:1259`）：

```cpp
        const bool push_limit_down =
            qt->term_type() == QT_UNION &&
            qts->query_block()->order_list.size() == 0 && !calc_found_rows;
        const ha_rows max_rows = push_limit_down
                                     ? qts->query_block()->get_limit(thd) +
                                           qts->query_block()->get_offset(thd)
                                     : HA_POS_ERROR;
        path = add_materialized_access_path(thd, qts, query_blocks, dest, max_rows);
```

> ★★ 只有 **UNION 且无 ORDER BY** 才能把 LIMIT 下推成 `MaterializeAccessPath` 的 `limit`（物化到 N 行就停）。**INTERSECT/EXCEPT 做不到**——因为不知道最终行数（注释：`For these set operations, we do not know enough to enforce the limit at materialize time`）。

**★★ `AppendIterator` 全文**（`composite_iterators.cc:2226`，★ UNION ALL 的流式载体）：

```cpp
int AppendIterator::Read() {
  if (m_current_iterator_index >= m_sub_iterators.size()) return -1;
  int err = m_sub_iterators[m_current_iterator_index]->Read();
  if (err != -1) return err;                       // 一行，或错误

  // EOF. Go to the next iterator.
  m_sub_iterators[m_current_iterator_index]->EndPSIBatchModeIfStarted();
  if (++m_current_iterator_index >= m_sub_iterators.size()) return -1;
  if (m_sub_iterators[m_current_iterator_index]->Init()) return 1;
  if (m_pfs_batch_mode_enabled) m_sub_iterators[m_current_iterator_index]->StartPSIBatchMode();
  return Read();                                   // ★ 尾递归切换下一个分支
}
```

> ★ 注意 **惰性 Init**：下一个分支只在"上一个读到 EOF"时才 `Init()`，不是一开始就全 Init。

**★ `StreamingIterator` 的"只拷贝不写"**（`composite_iterators.cc:1548`）：

```cpp
int StreamingIterator::Read() {
  m_join->set_ref_item_slice(m_input_slice);
  auto switch_to_output_slice = create_scope_guard([&] { ... });

  int error = m_subquery_iterator->Read();
  if (error != 0) return error;

  // Materialize items for this row.
  if (copy_funcs(m_temp_table_param, thd())) return 1;   /* ★ 仅此而已 —— 不写表 */
  ...
}
```

#### 3.3 最终组装

```cpp
if (union_all_sub_paths->size() == 1) {
  m_root_access_path = (*union_all_sub_paths)[0].path;
} else {
  assert(streaming_allowed);
  m_root_access_path = NewAppendAccessPath(thd, union_all_sub_paths);
}

// NOTE: If there's a fake_query_block, its JOIN's iterator already handles
// LIMIT/OFFSET, so we don't do it again here.
if (streaming_allowed && (limit != HA_POS_ERROR || offset != 0) &&
    (is_simple() || set_operation()->m_last_distinct == 0)) {
  m_root_access_path = NewLimitOffsetAccessPath(
      thd, m_root_access_path, limit, offset, calc_found_rows,
      /*reject_multiple_rows=*/false, &send_records);
}
```

LIMIT/OFFSET 的应用有一个前提条件：`is_simple() || m_last_distinct == 0`。也就是说：**只有全部是 UNION ALL（没有去重 operand）时，LIMIT 才能直接用 LimitOffset 节点截断**；一旦有去重，必须先物化去重完再截断（否则会截断掉需要参与去重的行）。

### 四、setup_materialize_set_op：谁需要去重

> `sql/query_term.cc`。

```cpp
int64 idx = -1;
for (Query_term *term : m_children) {
  ++idx;
  bool activate_deduplication =
      idx <= m_last_distinct ||
      term_type() != QT_UNION; /* always for INTERSECT and EXCEPT */
  JOIN *join = term->query_block()->join;
  AccessPath *child_path = join->root_access_path();
  ...
  if (idx == m_last_distinct && idx > 0 && union_distinct_only)
    // The rest will be done by appending.
    break;
}
```

判据只有两条：

1. `idx <= m_last_distinct` —— UNION 的前若干个 operand 需要去重
2. `term_type() != QT_UNION` —— **INTERSECT / EXCEPT 永远激活去重**（它们语义上就必须去重/计数）

`union_distinct_only` 参数的含义写在注释里：*"if true, materialize only UNION DISTINCT query blocks (any UNION ALL blocks are presumed handled higher up, by AppendIterator)"* —— 即"剩下的由 AppendIterator 处理"，配合 `break` 提前退出。

**★★ 完整源码**（⚠️ **不是自由函数**，是 `Query_term_set_op` 的成员函数，真身在 **`sql/sql_union.cc:1368`**，不在 `query_term.cc`）：

```cpp
Mem_root_array<MaterializePathParameters::QueryBlock>
Query_term_set_op::setup_materialize_set_op(THD *thd, TABLE *dst_table,
                                            bool union_distinct_only,
                                            bool calc_found_rows) {
  Mem_root_array<MaterializePathParameters::QueryBlock> query_blocks(thd->mem_root);

  int64 idx = -1;
  for (Query_term *term : m_children) {
    ++idx;
    bool activate_deduplication =
        idx <= m_last_distinct ||
        term_type() != QT_UNION; /* always for INTERSECT and EXCEPT */
    JOIN *join = term->query_block()->join;
    AccessPath *child_path = join->root_access_path();
    assert(join->is_optimized() && child_path != nullptr);

    if (term->term_type() != QT_QUERY_BLOCK)
      child_path = make_set_op_access_path(thd, nullptr, term, nullptr, calc_found_rows);

    MaterializePathParameters::QueryBlock param =
        term->query_block()->setup_materialize_query_block(child_path, dst_table);
    param.m_first_distinct = m_first_distinct;      /* ★ EXCEPT 用 */
    param.m_operand_idx = idx;                      /* ★★ 0 = 左侧（写行）；>0 = 右侧（只更新计数） */
    param.m_total_operands = m_children.size();     /* ★ INTERSECT DISTINCT 计数器初值 = N-1 */
    param.disable_deduplication_by_hash_field =
        (has_mixed_distinct_operators() && !activate_deduplication);
    query_blocks.push_back(std::move(param));

    if (idx == m_last_distinct && idx > 0 && union_distinct_only)
      // The rest will be done by appending.
      break;                                        /* ★ 流式策略：DISTINCT 段到此为止 */
  }
  return query_blocks;
}
```

> ★★ **`disable_deduplication_by_hash_field` 是"混合 UNION 不去重"的实现**：混合时 ALL 段的这个标志为真，`MaterializeQueryBlock()` 里就**连 `check_unique_constraint()` 都不调**（见 5.4）。前提是本表必须有 `hash_field`（`assert(doing_hash_deduplication())`）——因为唯一索引无法部分关闭。

**★★ "为什么不能只关掉唯一索引"的源码自陈**（`composite_iterators.h:452`）：

```cpp
  /// If true, unique constraint checking via hash key is disabled
  /// when materializing this query block (ie., we simply avoid calling
  /// check_unique_constraint() for each row). Used when materializing
  /// UNION DISTINCT and UNION ALL parts into the same table.
  /// We'd like to just use a unique constraint via unique index instead,
  /// but there might be other indexes on the destination table
  /// that we'd like to keep, and the implementation doesn't allow
  /// disabling only one index.
  ///
  /// If you use this on a query block, doing_hash_deduplication()
  /// must be true.
  bool disable_deduplication_by_hash_field = false;
```

> ★★ 这段话解释了一个关键设计约束：**handler 层的索引开关是"全有全无"的，不能只关一个索引**。所以混合 UNION 只能放弃唯一索引、改用哈希字段手工比对。

### 五、INTERSECT / EXCEPT 的计数器算法

> 实现在 `MaterializeIterator`（`sql/iterators/composite_iterators.cc`）。

#### 5.1 基本思想

所有 operand 的行都写进同一张临时表，**每行带一个计数器**（`read_counter`，在临时表里是一个额外的隐藏列）。不同运算对计数器的操作不同：

| 运算 | 左侧 operand（idx 0） | 右侧 operand | 最终扫描时 |
|---|---|---|---|
| `EXCEPT` | 写入行，计数设初始 | 每匹配一行，计数**递减** | 输出计数 > 0 的行 |
| `INTERSECT DISTINCT` | 写入行，计数 = N-1（N 为 operand 数） | 每匹配一行，计数递减 | 输出计数 == 0 的行 |
| `INTERSECT ALL` | 用 `subcounter[0]` 记录左侧出现次数 | 用 `subcounter[1]` 递增（不超过 [0]） | 输出 min(subcounter[0], subcounter[1]) |

源码注释里对 `INTERSECT DISTINCT` 的解释是：*"After we finish reading the left side, each row's counter contains N - 1, i.e. the number of operands intersected."*

#### 5.2 EXCEPT 的具体流程（源码注释原文）

```
EXCEPT    After we finish reading the left side, each row's counter
          contains 1 ... (右侧处理时递减)
```

右侧 operand 处理时：**不写行**（注释明写 `continue; // right hand side of EXCEPT or INTERSECT, never write`），只对已存在的行做计数更新。

#### 5.3 INTERSECT ALL 的限制

```cpp
HalfCounter c(read_counter());
if (static_cast<uint64_t>(c[0]) + 1 > std::numeric_limits<uint32_t>::max()) {
  my_error(ER_INTERSECT_ALL_MAX_DUPLICATES_EXCEEDED, MYF(0));
```

`INTERSECT ALL` 用**两个 32 位子计数器**打包成一个 `HalfCounter`（放在同一个 64 位计数器列里）。若某一行在左侧出现次数超过 `2^32`，直接报错 `ER_INTERSECT_ALL_MAX_DUPLICATES_EXCEEDED`。

而且源码注释明确写出这个算法的前提：*"NOTE: this only works correctly if we only ever have two blocks for INTERSECT ALL"*，并配了 `assert(query_block.m_operand_idx <= 1)`。也就是说 **INTERSECT ALL 目前只支持两个 operand**，更多 operand 的实现不完整。

#### 5.4 为什么不用哈希

`INTERSECT`/`EXCEPT` 天然可以用哈希集合实现（PG 的 `HashSetOp`）。MySQL 没这么做，原因与 UNION DISTINCT 相同：**没有通用的哈希集合算子**，而临时表 + 计数器这套机制同时覆盖了 `DISTINCT` 与 `ALL` 语义，代价是 INTERSECT/EXCEPT 一定物化。

**★★ 源码自陈：哈希做"行身份"，计数器做"集合语义"**（★ 这是 5.4 最准确的表述，`sql/table.h:1509`）：

```cpp
  // The next few members are used if this (temporary) file is used solely for
  // the materialization/computation of an INTERSECT or EXCEPT set operation
  // (in addition to hash_field above used to detect duplicate rows).  For
  // INTERSECT and EXCEPT, we always use the hash field and compute the shape
  // of the result set using m_set_counter. ... The materialized table has no
  // duplicate rows, relying instead of the embedded counter to produce the
  // correct number of duplicates with ALL semantics.
  ...
  /// A priori unlimited. We pass this on to TableScanIterator at construction
  /// time ... For these set operations, we do not know enough to enforce the
  /// limit at materialize time (as for UNION): only when reading the rows with
  /// TableScanIterator do we check the counters.
  ha_rows m_limit_rows{HA_POS_ERROR};

 private:
  Field_longlong *m_set_counter{nullptr};
  bool m_except{false};
  bool m_last_operation_is_distinct{false};

 public:
  bool is_union_or_table() const { return m_set_counter == nullptr; }  /* ★ 有无 set_counter 就是"UNION 路线 vs INTERSECT/EXCEPT 路线"的分水岭 */
  bool is_intersect() const { return m_set_counter != nullptr && !m_except; }
  bool is_except() const { return m_set_counter != nullptr && m_except; }
  bool is_distinct() const { return m_last_operation_is_distinct; }
```

> ★★★ 最精辟的一句：**"The materialized table has no duplicate rows"** —— 临时表里**从不存重复行**，重复度完全由计数器在**读取阶段**还原。所以 INTERSECT/EXCEPT 的物化表大小是"去重后"的，不是"操作数之和"。

**★★ `HalfCounter` 完整定义**（⚠️ 在 **`sql/iterators/basic_row_iterators.h:528`**，**不在** `composite_iterators.h`；是独立非模板类）：

```cpp
/**
  Auxiliary class to squeeze two 32 bits integers into a 64 bits one, cf.
  logic of INTERSECT ALL in
  MaterializeIterator<Profiler>::MaterializeQueryBlock.
  For INTERSECT ALL we need two counters: the number of duplicates in the left
  operand, and the number of matches seen (so far) from the right operand.
  Instead of adding another field to the temporary table, we subdivide the
  64 bits counter we already have. This imposes an implementation restriction
  on INTERSECT ALL: the resulting table must have <= uint32::max duplicates of
  any row.
 */
class HalfCounter {
  union {
    /// [0]: # of duplicates on left side of INTERSECT ALL
    /// [1]: # of duplicates on right side of INTERSECT ALL. Always <= [0].
    uint32_t m_value[2];      /* ★ 同一 64 位字段的两种视图 */
    uint64_t m_value64;
  } data;

 public:
  HalfCounter(uint64_t packed) { data.m_value64 = packed; }
  uint64_t value() const { return data.m_value64; }
  uint32_t &operator[](size_t idx) {
    assert(idx == 0 || idx == 1);
    return data.m_value[idx];
  }
};
```

> ★ "不新增一个字段"是刻意的：复用已有的 64 位计数器列，用 union 拆成两个 32 位。

**★★★ 计数器的读写技巧：`read_counter()` 临时掰字段指针**（`composite_iterators.cc:1137`）：

```cpp
  uchar *const set_counter_0 =
      t->set_counter() != nullptr ? t->set_counter()->field_ptr() : nullptr;
  uchar *const set_counter_1 = t->set_counter() != nullptr
                                   ? set_counter_0 + t->s->rec_buff_length
                                   : nullptr;
  /**
    Read the value of TABLE::m_set_counter from record[1]. The value can be
    found there after a call to check_unique_constraint if the row was found.
    Note that m_set_counter a priori points to record[0], which is used when
    writing and updating the counter.
   */
  auto read_counter = [t, set_counter_0, set_counter_1]() -> ulonglong {
    assert(t->record[1] - t->record[0] == set_counter_1 - set_counter_0);
    t->set_counter()->set_field_ptr(set_counter_1);   /* ★ 掰到 record[1] 读旧值 */
    ulonglong cnt = static_cast<ulonglong>(t->set_counter()->val_int());
    t->set_counter()->set_field_ptr(set_counter_0);   /* ★ 必须掰回 record[0] */
    return cnt;
  };
```

> ★★ `check_unique_constraint()` 把**命中的旧行放在 `record[1]`**，而计数器字段平时指向 `record[0]`（写/更新用）。所以读旧计数必须临时改 `field_ptr`——**读完必须掰回，否则后续 `store()` 会写错位置**。这是一个极易出 bug 的手法。

**★★★ INTERSECT DISTINCT 的"必须出现在所有操作数"判定**（★ 本篇最精妙的一处，`composite_iterators.cc:1343`）：

```cpp
          ulonglong cnt = read_counter();
          if (cnt == query_block.m_total_operands - query_block.m_operand_idx) {
            cnt = cnt - 1;
            t->set_counter()->store(cnt, true);
            error = t->file->ha_update_row(t->record[1], t->record[0]);
          }
```

> ★★★ 计数器必须**恰好等于期望值** `N - idx` 才递减。若某中间操作数缺失该行 ⇒ 计数器**偏高** ⇒ 后续永远不再等于期望值 ⇒ 永不到 0 ⇒ 最终被 `TableScanIterator` 丢弃。**用一次整数比较同时表达了"必须出现在所有 operand"和"只递减一次"两个语义**。

**★ 左侧（operand 0）写行段**（`composite_iterators.cc:1218-1293`，节选关键分支）：

```cpp
    } else if (query_block.m_operand_idx == 0) {
      if (t->is_except()) {
        // EXCEPT  After we finish reading the left side, each row's counter
        //         contains the number of duplicates seen of that row.
        if (check_unique_constraint(t)) {
          t->set_counter()->store(1, true);                  /* ★ 新行：计数 = 1 */
        } else {
          t->set_counter()->store(read_counter() + 1, true); /* ★ 重复行：只 +1，不写行 */
          error = t->file->ha_update_row(t->record[1], t->record[0]);
          if (!t->file->is_ignorable_error(error) &&
              spill_to_disk_and_retry_update_row(thd(), error)) return true;
          continue;
        }
      } else {
        assert(t->is_intersect());
        if (t->is_distinct()) {
          if (check_unique_constraint(t)) {
            t->set_counter()->store(query_block.m_total_operands - 1, true);
            /* ★ 初值 = 还需被多少个右侧操作数"点名" */
          } else {
            continue;  // already written and counter initialized
          }
        } else {
          // INTERSECT ALL：左侧建立 subcounter[0]
          if (check_unique_constraint(t)) {
            HalfCounter c(0);
            c[0] = 1;
            t->set_counter()->store(c.value(), true);
          } else {
            HalfCounter c(read_counter());
            if (static_cast<uint64_t>(c[0]) + 1 >
                std::numeric_limits<uint32_t>::max()) {
              my_error(ER_INTERSECT_ALL_MAX_DUPLICATES_EXCEEDED, MYF(0));
              return true;
            }
            c[0]++;
            t->set_counter()->store(c.value(), true);
            error = t->file->ha_update_row(t->record[1], t->record[0]);
            ... continue;
          }
        }
      }
    } else {   /* 右侧见下 */
```

**★ 右侧（operand > 0）**（`composite_iterators.cc:1294-1387`，节选）：

```cpp
      if (t->is_except()) {
        // EXCEPT  After this right side has been processed, the counter
        //         contains the number of duplicates not yet matched.
        if (check_unique_constraint(t)) continue;      /* ★ 右侧独有行：左侧没有，忽略 */
        ulonglong cnt = read_counter();
        if (cnt > 0) {
          if (query_block.m_operand_idx < query_block.m_first_distinct)
            t->set_counter()->store(cnt - 1, true);    /* ★ EXCEPT ALL：抵消一个 */
          else
            t->set_counter()->store(0, true);          /* ★ EXCEPT DISTINCT：整行清零 */
          error = t->file->ha_update_row(t->record[1], t->record[0]);
          ...
        }
      } else {
        assert(t->is_intersect());
        if (t->is_distinct()) { /* ... 见上面的"必须等于期望值"段 ... */ }
        else {
          assert(query_block.m_operand_idx <= 1);      /* ★ INTERSECT ALL 只允许二元 */
          if (check_unique_constraint(t)) continue;
          HalfCounter c(read_counter());
          const uint32_t left_side = c[0];
          if (c[1] + 1 <= left_side) {
            c[1]++;                                    /* ★ 右侧匹配数，封顶为左侧重复数 */
            t->set_counter()->store(c.value(), true);
            error = t->file->ha_update_row(t->record[1], t->record[0]);
            ...
          }  // else: already matched all occurences from left side table
        }
      }
      continue;  // right hand side of EXCEPT or INTERSECT, never write
```

**★★ 读取侧：把计数器还原成结果行**（`TableScanIterator::Read()`，`basic_row_iterators.cc:212`）：

```cpp
    while (true) {
      if (m_remaining_dups == 0) {
        while ((tmp = table()->file->ha_rnd_next(m_record))) {
          if (tmp == HA_ERR_RECORD_DELETED && !thd()->killed) continue;
          return HandleError(tmp);
        }
        const ulonglong cnt =
            static_cast<ulonglong>(table()->set_counter()->val_int());
        if (table()->is_except()) {
          if (table()->is_distinct()) {
            if (cnt >= 1) break;                    /* ★ EXCEPT DISTINCT：>0 ⇒ 恰好一行 */
          } else {
            m_remaining_dups = cnt;                 /* ★ EXCEPT ALL：展开成 cnt 行 */
          }
        } else {
          if (table()->is_distinct()) {
            if (cnt == 0) break;                    /* ★ INTERSECT DISTINCT：归零才算出现在所有 operand */
          } else {
            HalfCounter c(cnt);
            m_remaining_dups = std::min(c[0], c[1]); /* ★ INTERSECT ALL：min(左重复, 右匹配) */
          }
        }
      } else {
        --m_remaining_dups;  // return the same row once more.
        break;
      }
      // Skipping this row
    }
    if (++m_stored_rows > m_limit_rows) {
      return HandleError(HA_ERR_END_OF_FILE);        /* ★ INTERSECT/EXCEPT 的 LIMIT 只能在扫描期限流 */
    }
```

> ★★ **`m_remaining_dups` 是"重复度展开器"**：同一行连续返回 `m_remaining_dups` 次。这就把"表里只存一行 + 一个计数"还原成符合 ALL 语义的结果集。
> ⚠️ `MaterializeIterator::Init()` 里只对 `is_union_or_table()` 做 `stored_rows >= m_limit_rows` 的提前 break（`:946`），INTERSECT/EXCEPT 的注释明说 *"For INTERSECT and EXCEPT, this is done in TableScanIterator"*。

**★★ UNION DISTINCT 的去重 = 临时表上的一个唯一索引**（★ **"MySQL 没有独立去重算子"的最硬证据**）：

```cpp
/* sql/sql_tmp_table.cc:1329-1358 */
      param->keyinfo->flags = HA_NOSAME | HA_NULL_ARE_EQUAL;
      param->keyinfo->name = "<auto_distinct_key>";   /* ★ EXPLAIN/元数据里这个名字就是 UNION DISTINCT 的"去重算子" */
      for (unsigned i = param->hidden_field_count; i < share->fields;
           i++, key_part_info++) {
        key_part_info->init_from_field(table->field[i]);
        if (key_part_info->store_length > max_key_part_length) {
          unique_constraint_via_hash_field = true;   /* ★ B-tree 键太长 → 退化为哈希字段 */
          break;
        }
      }
```

```cpp
/* Query_result_union::create_result_table()，sql_union.cc:222 */
    table->file->ha_extra(HA_EXTRA_IGNORE_DUP_KEY);   /* ★★ dup-key 变成"可忽略错误" ⇒ 插入即去重 */
```

```cpp
/* MaterializeQueryBlock() 写行失败时，composite_iterators.cc:1415 */
    } else {
      // An ignorable error means duplicate key, ie. we deduplicated
      // away the row. This is seemingly separate from
      // check_unique_constraint(), which only checks hash indexes.
    }
```

**★ 何时退化为哈希字段**（四个条件，`sql_tmp_table.cc:900`）：

```cpp
  /**
    When true, enforces unique constraint (by adding a hidden hash_field and
    creating a key over this field) when:
    (1) unique key is too long, or
    (2) number of key parts in distinct key is too big, or
    (3) the caller has requested it.
    (4) we have INTERSECT or EXCEPT, i.e. not UNION.
  */
  bool unique_constraint_via_hash_field =
      param->m_operation != Temp_table_param::TTP_UNION_OR_TABLE;
```

```cpp
/* sql_union.cc:213 —— 混合 UNION 的情形 */
  } else if (op->has_mixed_distinct_operators()) {
    tmp_table_param.force_hash_field_for_unique = true;
  }
```

**★ 手工哈希比对 `check_unique_constraint()`（`sql_executor.cc:4020`）**：

```cpp
bool check_unique_constraint(TABLE *table) {
  ulonglong hash;
  if (!table->hash_field) return true;     /* ★ 无哈希字段 ⇒ 恒"未找到"，去重全交给唯一索引 */
  if (table->no_keyread) return true;

  if (table->group) hash = unique_hash_group(table->group);
  else              hash = unique_hash_fields(table);
  table->hash_field->store(hash, true);
  int res = table->file->ha_index_read_map(table->record[1],   /* ★ 命中的旧行落在 record[1] */
                                           table->hash_field->field_ptr(),
                                           HA_WHOLE_KEY, HA_READ_KEY_EXACT);
  while (res == 0) {
    if (!(table->group
              ? group_rec_cmp(table->group, table->record[0], table->record[1])
              : table_rec_cmp(table))) {
      return false;  // skip it
    }
    res = table->file->ha_index_next_same(      /* ★ 哈希冲突：逐字段比对同哈希的下一行 */
        table->record[1], table->hash_field->field_ptr(), sizeof(hash));
  }
  return true;
}
```

### 六、ORDER BY / LIMIT 的处理与 fake query block

- 顶层 `ORDER BY` 由 **fake query block** 承载：它是一个额外的 `Query_block`，其 JOIN 的迭代器负责 LIMIT/OFFSET 与排序（源码注释："If there's a fake_query_block, its JOIN's iterator already handles LIMIT/OFFSET, so we don't do it again here"）。
- LIMIT/ORDER BY 可以**下推到 operand**：`pushdown_limit_order_by`（见 [`../05_contextualize.md`](../05_contextualize.md)）。
- 但 **INTERSECT / EXCEPT 的下推受限**：源码注释写明 *"Logic here presumes that query expressions that only add limit (not order by) will have been pushed down"*，并且只有 `cand->has_limit()` 时才允许（`level == 0 || cand->has_limit()`）。原因是集合运算的语义不允许在 operand 上提前截断（会影响交/差的结果）。

**★★★ 但 `fake_query_block` 在 8.0.39 已经不存在了。**

全 `sql/` 目录里 `fake_query_block` 只剩**注释残留**（`sql_union.cc:1494/1745`、`sql_derived.cc:334`），没有成员、没有调用。源码自己写明：

```cpp
/* sql/query_term.h:183 —— ★ 源码原注释 */
  Each of these also contain a Query_block which will handle its order by
  and/or limit clauses. These are similar to the old so-called
  "fake_query_block" (which is now gone), and are not linked in with "next"
  pointers.
```

> ★★★ **"which is now gone"** —— 官方原文。旧版只有**一个** fake block 挂在 `Query_expression` 上，所以只能支持单层 `ORDER BY/LIMIT`；新版是**每个非叶节点各有一个 companion block**（`Query_term_set_op::m_block`），于是能支持任意多层嵌套：
>
> ```sql
> (((SELECT a,b FROM t) ORDER BY a LIMIT 5) ORDER BY -b LIMIT 3) ORDER BY a;
> ```
>
> 这正是 `Query_term` 类 Doxygen（`query_term.h:33`）开头举的例子——"Originally ... limited us to left-deep nesting of unions ... could not be supported"。

**★ 新的官方入口 `global_parameters()`**（`sql_lex.h:954`）：

```cpp
  /**
    Return the query block holding the top level ORDER BY, LIMIT and OFFSET.
    ...
    @return query block containing the global parameters
  */
  inline Query_block *global_parameters() const {
    return query_term()->query_block();   /* ★ 根节点的 companion block == 旧 fake_query_block 的角色 */
  }

  /* LIMIT clause runtime counters */
  ha_rows select_limit_cnt, offset_limit_cnt;
```

> ⚠️ `select_limit_cnt` 是**公有数据成员**不是方法，由 `Query_expression::set_limit(THD*, Query_block*)` 填充。

**★★ ORDER BY/LIMIT 的真正"搬运"在 `pushdown_limit_order_by()`**（★ 不在 prepare 里。`query_term.cc:74`）：

```cpp
    case QT_UNARY: {
      auto *unary = down_cast<Query_term_unary *>(this);
      Query_block *const this_block = unary->query_block();
      Query_block *const child_block = unary->m_children[0]->query_block();
      if (this_block->order_list.elements == 0 &&
          child_block->absorb_limit_of(this_block)) {
        // Eliminate this level by pushing down LIMIT/OFFSET, if any.
        // E.g.  (SELECT a, c FROM t1 ORDER BY a LIMIT 4) LIMIT 2
        //   to  SELECT a, c FROM t1 ORDER BY a LIMIT 2
        return unary->m_children[0]->pushdown_limit_order_by(parent);
      } else {
        //    (SELECT a, b AS c FROM t1) ORDER BY b+1 LIMIT 5
        // The above would be allowed, since we re-write to:
        //    SELECT a, b AS c FROM t1 ORDER BY b+1 LIMIT 5
        if (child_block->order_list.elements == 0 &&
            child_block->select_limit == nullptr) {
          child_block->order_list = this_block->order_list;   /* ★★ 整个 order_list 赋给内层 */
          child_block->absorb_limit_of(this_block);
          child_block->m_windows.prepend(&this_block->m_windows);
          ... 累加 n_sum_items / n_child_sum_items / n_scalar_subqueries ...
          return unary->m_children[0]->pushdown_limit_order_by(parent);  /* ★ 递归消更多层 */
        }
        ...
      }
    } break;
```

> ★★ 注意注释里的自陈缺陷：*"this makes name binding too lenient (non-std)"*——把外层 ORDER BY 下推后，`(SELECT a, b AS c FROM t1) ORDER BY b+1` 这种**本不该通过**的名字解析也会被允许。这是一个**为了消除嵌套层而故意放宽标准**的取舍。

**★ 冗余 ORDER BY 的消除**（`query_term.cc:43`）：

```cpp
std::pair<bool, bool> Query_term::redundant_order_by(Query_block *cand, int level) {
  if (query_block() == cand) {
    if (level == 0 || cand->has_limit()) return {true, false};   /* 不冗余 */
    return {true, true};                                          /* 冗余，可消除 */
  }
  auto op = down_cast<Query_term_set_op *>(this);
  for (Query_term *child : op->m_children) {
    auto result = child->redundant_order_by(cand, level + 1);
    if (result.first) return result;
  }
  return {false, false};
}
```

**★ `set_limit()`：LIMIT n OFFSET m 折叠成 n+m**（`sql_lex.cc:3751`）：

```cpp
bool Query_expression::set_limit(THD *thd, Query_block *provider) {
  offset_limit_cnt = provider->get_offset(thd);
  select_limit_cnt = provider->get_limit(thd);
  if (select_limit_cnt + offset_limit_cnt >= select_limit_cnt)
    select_limit_cnt += offset_limit_cnt;
  else
    select_limit_cnt = HA_POS_ERROR;   /* ★ 加法回绕 ⇒ 视为无上限 */
  return false;
}
```

**★ 树是在 contextualize 阶段建的，不在 prepare**（⚠️ **不存在** `Query_expression::push_query_term()`）：

```
PT_set_operation::contextualize_setop()   parse_tree_nodes.cc:1653
   ├─ new Query_term_union / _intersect / _except
   ├─ merge_descendants()                 ← N-ary 折叠 + 算 m_last_distinct / m_first_distinct
   ├─ setop->label_children()             ← 设置每个 child 的 sibling_idx
   └─ setop->set_block(qe->create_post_processing_block(setop))   ← companion block
Parse_context::finalize_query_expression()   parse_tree_node_base.cc:37
   ├─ top = top->pushdown_limit_order_by()   ← 消括号 / 下推 LIMIT / 建 m_parent 链
   ├─ select->master_query_expression()->set_query_term(top)      ← ★ 真名
   └─ top->validate_structure(nullptr)       ← 深度 > MAX_SELECT_NESTING 报错 + 给后处理块编号
```

> ★ `create_post_processing_block()` 里有一处细节：后处理块先编到 `select_number = INT_MAX`，**真正编号在 `validate_structure()` 里 `query_block()->renumber()`**——这样能保证"叶子 QB 编号小、顶层 set-op 块编号最大"（与 EXPLAIN 的 id 列顺序一致）。

**★ INTERSECT 优先级高于 UNION/EXCEPT 的实现**（⚠️ 8.0.39 的 yacc 里**没有** `query_term` 非终结符——三个 set-op **同在一个** `query_expression_body` 左递归产生式里）：

```yacc
%left UNION_SYM EXCEPT_SYM        /* ★ 先声明 = 优先级低 */
%left INTERSECT_SYM               /* ★ 后声明 = 优先级高 ⇒ INTERSECT 绑定更紧 */
```

```yacc
query_expression_body:
          query_primary
        | query_expression_parens %prec SUBQUERY_AS_EXPR
        | query_expression_body UNION_SYM union_option query_expression_body
        | query_expression_body EXCEPT_SYM union_option query_expression_body
        | query_expression_body INTERSECT_SYM union_option query_expression_body
        ;
```

### 七、完整调用栈

```
Query_expression::optimize()                      sql_union.cc
  ├─ for each Query_block:
  │     └─ Query_block::optimize() → JOIN::optimize()      ← 各 operand 独立优化
  │           （边算边累加 estimated_rowcount / estimated_cost）
  ├─ optimize_set_operand(thd, this, query_term())         ← set-op 层面优化
  ├─ set_limit()
  ├─ 三选一：
  │     ├─ create_access_paths()                           ← ① 常规路径
  │     ├─ setup_materialize_set_op(union_distinct_only=false)  ← ② 直接物化到调用者表
  │     └─ （unfinished materialization，仅 derived 优化期求值场景）
  └─ CreateIteratorFromAccessPath(m_root_access_path)     ← m_root_iterator（归 unit 所有）

Query_expression::create_access_paths()
  ├─ is_simple() → 直接取 join->root_access_path()
  ├─ streaming_allowed 判定（ORDER BY / m_is_materialized / INSERT-SELECT）
  ├─ make_set_op_access_path() → 递归构造 set-op 的 AccessPath
  │     └─ union_all_sub_paths 收集可流式的 operand
  ├─ NewAppendAccessPath()（多个 ALL operand 时）
  └─ NewLimitOffsetAccessPath()（仅当 m_last_distinct == 0 时）
```

---

## ★ 本机制里的工程实现技法

### 1. `m_last_distinct`：用一个下标表达"混合 UNION 的切分点"

`UNION DISTINCT` 与 `UNION ALL` 混合时，MySQL 不去构造复杂的计划，而是**记一个下标**："下标之前的物化去重，之后的流式追加"。这是一个把语义约束压缩进一个整数的做法，简洁但表达能力有限（例如 `UNION ALL` 在前、`UNION DISTINCT` 在后时，就退化成全部物化）。

### 2. 用 assert 表达算法前提

```cpp
assert(query_block.m_operand_idx <= 1);   // INTERSECT ALL 只支持两个 operand
```

`INTERSECT ALL` 的"只支持两个 operand"不是硬编码的限制检查，而是一个 `assert`——意味着**这个限制只在 debug 构建下被检查**。这是把"尚未实现的功能"用断言标记的典型做法（社区版常见）。

### 3. Halloween Problem 的防护写成了规则

`INSERT ... SELECT` 时禁止流式，本质上是防止"边扫边插读到自己插入的行"。MySQL 没有做通用的快照/版本隔离来解决，而是在计划生成层直接禁掉流式路径，并留了 `@todo` 指向 bug 23022426。

### 4. 优化期跨 query block 的信息传递

递归 CTE 需要"非递归部分的行数"来优化自己的递归部分。MySQL 的传递方式是**把估计值写进 `query_result()`**，让后面的优化器去读这个公共对象。源码里的 TODO 也承认：*"Communicate this in a different way when the query result goes away"*。

### 5. `HalfCounter`：把两个 32 位计数器打包进一个 64 位字段

`INTERSECT ALL` 需要两个计数（左侧出现次数、右侧匹配次数），MySQL 把它们打包成一个 64 位计数列（`HalfCounter`）。好处是临时表不需要加两列；代价是溢出检测与位操作散落在执行代码里。

### 6. 注释里的"自陈缺陷"

本模块有大量注释坦承当前实现的局限性（"legacy decision, we could probably be more intelligent"、"FIXME: find out if we can remove this exception"、"TODO(sgunders)"）。这是 8.0.31 新代码的典型特征：**先正确、后优化**，把已知的次优实现写进注释留给后人。

---

## 可观测性

### EXPLAIN 倒排

| 你看到 | 对应路径 |
|---|---|
| `-> Append` + 各 operand 独立 | `streaming_allowed == true` 且全部 UNION ALL |
| `-> Table scan on <union1,2>` | 物化路径（`streaming_allowed == false` 或有去重 operand） |
| `-> Materialize` + `-> Table scan on <union...>` | 混合路径：去重部分物化、ALL 部分由 AppendIterator 追加 |
| `-> Sort: t.a` 下有 `<union1,2>` | 顶层 ORDER BY 强制物化（开场那个例子） |
| INTERSECT / EXCEPT 的 `-> Table scan` | 计数器算法；扫描时按计数过滤 |

### 验证手段

```sql
-- 看 AccessPath 树（最直接）
EXPLAIN FORMAT=TREE SELECT a FROM t1 UNION SELECT a FROM t2;

-- 看是否真的建了临时表（对比两次执行）
FLUSH STATUS;
SELECT ... ;
SHOW STATUS LIKE 'Created_tmp_tables';
```

### DBUG 通道

`Query_expression::optimize` 里有 `DBUG_EXECUTE_IF("ast", ...)`，可以打印 `Query_term` 树的 ASCII 图与 AccessPath 计划：

```
SET debug='d,ast';
```

这在调试复杂嵌套 set-op 时非常有用（见 [`../05_contextualize.md`](../05_contextualize.md) 里 `Query_term` 的调试支持）。

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 让顶层 ORDER BY 也能流式 UNION | `create_access_paths` 的 `streaming_allowed` 判定（注释已指明这是遗留决定），需要确认 filesort 是否还会按 row ID 回表 |
| 加一个真正的 `Unique`/`Distinct` 迭代器（免物化的 UNION DISTINCT） | 新 AccessPath 类型 + 迭代器 + `create_access_paths` 的分派 |
| 让 `INTERSECT ALL` 支持多于两个 operand | `MaterializeIterator` 的计数器逻辑 + 去掉 `assert(m_operand_idx <= 1)` |
| 支持 set-op 的代价决策（流式 vs 物化） | 需要在 `create_access_paths` 引入代价比较，当前是纯规则 |
| 新增一种 set 运算 | `query_term.h` 加 `Query_term_*` 子类 + `QT_*` 枚举 + `create_access_paths` 的分派 + `MaterializeIterator` 的计数逻辑 |

### 已知缺陷

- **顶层 ORDER BY 强制物化 UNION ALL**（遗留决定，注释自陈）。
- **`INTERSECT ALL` 只支持两个 operand**，且单行的重复度上限 `2^32`（超出报 `ER_INTERSECT_ALL_MAX_DUPLICATES_EXCEEDED`）。
- **`Query_term` 与 `Query_expression::first_query_block()` 链表两套表示并存**，源码 TODO 计划合并。
- **`use_iterator` 对 INTERSECT/EXCEPT 有一个 FIXME**：*"In corner cases for transform of scalar subquery, it can happen that the destination table isn't ready ... so force double materialization. FIXME: find out if we can remove this exception."*
- **INSERT ... SELECT 禁止流式**是正确性防护，非性能考虑（bug 23022426）。

### 社区边界澄清

- **8.0.31 之前社区版没有 INTERSECT / EXCEPT**，只有 UNION。
- **没有 set-op 的并行执行**（PG 有 `Parallel Append`，MySQL 没有）。
- **没有 `Unique` 算子**：去重要靠临时表，这是 MySQL 与 PG/Oracle 在 set-op 实现上最大的结构性差异。
- **`UNION DISTINCT` 的代价不进入比较**：`streaming_allowed` 是规则，不是代价决策。

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `Query_term` 有 4 个子类 | **层级是 3 层**，`Query_block` 也直接继承 `Query_term` ⇒ 实际是 **5 个节点类型**。中间多了抽象基类 **`Query_term_set_op`** |
| `Query_term::optimize()` / 各子类的 `optimize()` | **不存在**（`query_term.cc` 全文无 `optimize`）。真身是 static 自由函数 **`optimize_set_operand()`**（`sql_union.cc:915`） |
| `Query_expression::set_found_rows_for_union()` | **不存在**。是 `optimize()` 内联的一段（`sql_union.cc:1043`） |
| `Query_expression::push_query_term()` | **不存在**。树在 **contextualize** 阶段就建好，真名是 `Query_expression::set_query_term()`（由 `finalize_query_expression()` 调用） |
| `Query_expression::change_to_union_with_tmp_table()` | **不存在**。等价物是 `create_tmp_table_for_set_op()` + `Query_term_set_op::open_result_tables()` |
| `Query_expression` 有 `union_distinct` / `m_union_type` | **都不存在**。QE 层只有 `Query_term *m_query_term`；访问器是 `last_distinct()` / `has_top_level_distinct()` / `set_operation()` |
| `Query_expression::query_blocks()` | **不存在**。遍历靠 `first_query_block()` + `next_query_block()` 链表，或 `query_terms<order,visit_leaves>()` 模板迭代器 |
| INTERSECT 优先级靠 yacc 的 `query_term` 非终结符 | **8.0.39 没有 `query_term` 非终结符**。三个 set-op 同在一个左递归产生式里，优先级靠 bison `%left` 的**声明顺序** |
| `Query_term::print()` 反打印 SQL | **不是成员**。是自由函数 **`print_set_operation()`**（`sql_lex.cc:2575`）；`Query_term_set_op::print()` 是 protected 的调试打印 minion |
| `m_last_distinct` 对 Unary 是 -1 | 注释写 -1，**代码实为 0**（`query_term.h:566`） |
| `MaterializeIterator` 声明在 `composite_iterators.h` | **在 .cc 里**（`composite_iterators.cc:619`）。头文件里只有 `materialize_iterator::QueryBlock` 与 `CreateIterator()` |
| `Query_expression::create_iterators()` / `set_iterators()` | **不存在**。真名 `create_access_paths(THD*)`（`void`）+ `force_create_iterators()` + 全局 `CreateIteratorFromAccessPath()` |
| `streaming_allowed` 只有一个否定条件 | **两个**（顶层 ORDER BY / `m_is_materialized`），另有**第三个**独立 else-if 分支（INSERT/REPLACE SELECT 顶层） |
| 混合 UNION ALL/DISTINCT 只有一种策略 | **两种**，取决于 `streaming_allowed` |
| `setup_materialize_set_op()` 是自由函数 / 在 `query_term.cc` | 是 **`Query_term_set_op` 的成员函数**，真身在 **`sql/sql_union.cc:1368`** |
| INTERSECT/EXCEPT 用哈希计数 | **行身份**用 `hash_field` 去重，**集合语义**用一个隐藏 64 位计数器 `TABLE::m_set_counter` + `ha_update_row()` 就地维护 |
| `HalfCounter` 在 `composite_iterators.h` | 在 **`sql/iterators/basic_row_iterators.h:528`**，独立非模板类 |
| 存在 `MaterializeUnionOrTable()` | **不存在**。只有 `MaterializeRecursive()` 与 `MaterializeQueryBlock()` |
| `fake_query_block` 承载 UNION 的 ORDER BY | **已不存在**（源码原注释 "which is now gone"）。现在是 **query term 树每个非叶节点的 companion `Query_block`** |
| UNION 的 ORDER BY 在 `prepare()` 里搬到 fake block | **不是**。真正的搬运在 **`Query_term::pushdown_limit_order_by()`**（contextualize 尾期） |
| `union_type` 枚举（`UT_UNION` 等） | **不存在**（全 `sql/` 0 命中）。真名 `Query_term_type` |
| `Query_expression::select_limit_cnt` 是方法 | 是**公有数据成员** |
| `Query_expression::optimize` 有 Halloween Problem 防护 | **不存在**（全 `sql/` 仅 `access_path.h:271` 注释提过一次）。等价防护是 `create_access_paths()` 里 INSERT/REPLACE SELECT 强制 `streaming_allowed = false` |
| `Query_expression::reset()` | **不存在**（在 `JOIN` 上）。QE 侧是 `ClearForExecution()` + `cleanup(bool)` |
| `is_recursive()` 靠 `m_recursive` 成员 | **没有 `m_recursive`**。就是 `first_recursive != nullptr` |
| EXPLAIN 的 `UNION RESULT` 由 `union_type` 决定 | 由 **`Query_block::type_str[]`**（`sql_lex.cc:144`）经 `Query_block::type()` → `setop2result()` 决定 |

**★ 四个"结构性事实"**

1. **一个 `m_last_distinct` 下标贯穿四个阶段**——prepare（决定 `m_is_materialized`、建表是否去重）、optimize（决定能否把 LIMIT 下推）、物化（谁要去重、何时 break）、打印（越过它就都打 `all`）。**这是本篇最值得记住的一处设计**：用一个整数而不是一组标志，把"UNION 左结合 + DISTINCT 作用于其左侧全部"这条语义压缩成了一个下标。

2. **INTERSECT/EXCEPT 的物化表里从不存重复行**。源码原话 *"The materialized table has no duplicate rows, relying instead of the embedded counter"*——重复度完全由计数器在**扫描阶段**展开（`TableScanIterator` 的 `m_remaining_dups`）。连锁后果：① 物化表大小是"去重后"的；② **LIMIT 只能在扫描期限流**，因为物化时不知道最终行数（对比 UNION 可以在物化时下推 LIMIT）。

3. **`read_counter()` 的"掰字段指针"是整个算法最脆弱的一环**。`check_unique_constraint()` 把命中的旧行放在 `record[1]`，而计数器字段平时指向 `record[0]`；读旧计数必须临时 `set_field_ptr(record[1])`，**读完必须掰回**。这个手法省了一个字段，代价是极易写错。

4. **"没有独立去重算子"是一个贯穿全文的约束**。UNION DISTINCT 的去重就是临时表上一个名为 `<auto_distinct_key>` 的 `HA_NOSAME` 唯一索引 + `HA_EXTRA_IGNORE_DUP_KEY`（dup-key 当可忽略错误吞掉）。而混合 UNION 因为"handler 的索引开关不能只关一个"，被迫放弃唯一索引、改用 `hash_field` 手工比对。**同一套临时表机制同时承担了去重、排序、物化、计数四件事**——这既是 MySQL 的简洁之处，也是它不能像 PG 那样用 `HashSetOp` 的原因。

> 注：`sql/sql_union.cc`、`sql/query_term.h` 等文件中的部分 `// note:` 与 `/* 【注】... */` 注释为本工作区人工批注，非 Oracle 上游原文。

---

## 参考

**论文 / 理论**

- 《Halloween Problem》—— INSERT...SELECT 边扫边插的经典问题（`streaming_allowed` 第二条否定的根因）。**⚠️ 注意它有两个完全不同的形态，且都不是 InnoDB 解决的**：

  | 形态 | 例子 | 谁解决 | 证据 |
  |---|---|---|---|
  | **A. UPDATE 型**（行在索引里前移被重复扫到） | `UPDATE t SET a=a+1 WHERE a<10` | **MySQL 层**，不是 InnoDB | `unsafe_key_update()`（`sql/sql_update.cc:1255`） |
  | **B. INSERT...SELECT 自引用型**（读到自己刚插的行） | `INSERT INTO t SELECT FROM t` | **优化器**（计划形状：流式 vs 物化） | `streaming_allowed = false`（`sql/sql_union.cc:1453`） |

  **★★★ InnoDB 两处 TODO 自陈"没有解决"**（8.0.39 仍在）：

  ```cpp
  /* storage/innobase/row/row0upd.cc:3171 */
      TODO! What to do to the 'Halloween problem', where an update
      moves the record forward in index so that it is again
      updated when the cursor arrives there? Solution: the
      read operation must check the undo record undo number when
      choosing records to update. MySQL solves now the problem
      externally!
  ```

  ```cpp
  /* storage/innobase/handler/ha_innodb.cc:9751 */
  TODO: currently InnoDB does not prevent the 'Halloween problem':
  in a searched update a single row can get updated several times
  if its index columns are updated!
  ```

  本篇引用的是 **B**（`INSERT ... SELECT` 自引用），不是 A。B 里引擎根本无从参与——它只看到"给我下一行"和"插入这一行"两个独立请求，**"要不要边扫边插"是执行形状的选择，正是优化器的活**。所以 `streaming_allowed = false` 让整个 UNION 的所有分支都必须先物化进临时表，**直接决定了 set operation 的执行形状**——这才是它出现在本篇的理由。

#### ★★ B 详解：`streaming_allowed = false` 之后发生了什么

`sql/sql_union.cc:1436` 起的完整判定：

```cpp
  bool streaming_allowed = true;
  if (global_parameters()->order_list.size() != 0 ||
      (!is_simple() && set_operation()->m_is_materialized)) {
    // If we're sorting, we currently put it in a real table no matter what.
    streaming_allowed = false;
  } else if ((thd->lex->sql_command == SQLCOM_INSERT_SELECT ||
              thd->lex->sql_command == SQLCOM_REPLACE_SELECT) &&
             thd->lex->unit == this) {
    // If we're doing an INSERT or REPLACE, and we're not outputting to
    // a temporary table already (ie., we are the topmost unit), then we
    // don't want to insert any records before we're done scanning. Otherwise,
    // we would risk incorrect results and/or infinite loops, as we'd be seeing
    // our own records as they get inserted.
    streaming_allowed = false;
  }
```

> ★★★ **注意注释里的 "we'd be seeing our own records as they get inserted"** —— MySQL 自己认为**不加这道防线就会看到自己插入的行**。这反过来说明：**MVCC 快照读不足以兜住这个场景**（至少在 `INSERT ... SELECT` 的加锁读语义下），必须由优化器在计划层切断。

**`streaming_allowed` 的两个消费点**（★ 一个布尔量决定两种完全不同的计划形状）：

```cpp
  // ① :1475 —— 决定 UNION ALL 的分支是"追加流式"还是"全部物化"
  if (!streaming_allowed || !is_simple()) {
    param.path = make_set_op_access_path(
        thd, nullptr, m_query_term,
        streaming_allowed ? union_all_sub_paths : nullptr, calc_found_rows);
    param.join = nullptr;
    if (!streaming_allowed) union_all_sub_paths->push_back(param);
    // else filled in by make_set_op_access_path
  }

  if (union_all_sub_paths->size() == 1) {
    m_root_access_path = (*union_all_sub_paths)[0].path;      /* 单一物化路径 */
  } else {
    // Just append all the UNION ALL sub-blocks.
    assert(streaming_allowed);
    m_root_access_path = NewAppendAccessPath(thd, union_all_sub_paths);  /* AppendIterator 流式 */
  }

  // ② :1496 —— 只有流式时才敢在顶层加 LIMIT/OFFSET
  if (streaming_allowed && (limit != HA_POS_ERROR || offset != 0) && ...) {
    m_root_access_path = NewLimitOffsetAccessPath(...);
  }
```

> ★ ★两处 `assert(streaming_allowed)` 是"流式 ⇒ 多分支"这个不变式的自我校验。**一旦 `streaming_allowed=false`，`union_all_sub_paths` 必为 1 个**，即整个 set operation 走单一物化路径。

#### ★★★ A 详解：UPDATE 型的四道防线（MySQL SQL 层）

**先说结论：单表 UPDATE 的主循环是边扫边改的，循环体内没有任何去重。防线在循环之前——对 iterator 的选择。**

```cpp
/* sql/sql_update.cc:872 —— 主循环本体（节选）*/
    while (true) {
      error = iterator->Read();                   /* ★ 读一行 */
      if (error || thd->killed) break;
      if (conds != nullptr) { ... if (skip_record) { unlock_row(); continue; } }
      ...
      store_record(table, record[1]);             /* before-image */
      ...
          error = table->file->ha_update_row(table->record[1], table->record[0]);
      /* ★★ 就地写回；没有任何"已处理 rowid 集合"、版本比对、游标回退 */
```

**★★ 防线 1（真正的防线）：`used_key_is_modified || order` → 先把 rowid 物化到 tempfile，再两阶段更新**

判据（`sql_update.cc:577-606`）：

```cpp
  bool used_key_is_modified = false;                    /* ★★ 全场唯一的 Halloween 判据 */
  used_index = get_index_for_order(&order_src, table, limit, range_scan,
                                   &need_sort, &reverse);
  if (need_sort) used_index = table->file->key_used_on_scan;
  if (used_index != MAX_KEY) {
    // Check if we are modifying a key that we are used to search with:
    used_key_is_modified = is_key_used(table, used_index, table->write_set);
  } else if (range_scan) {
    used_key_is_modified = (!unique_key_range(range_scan) &&
                            uses_index_on_fields(range_scan, table->write_set));
  }
  if (table->part_info)
    used_key_is_modified |= table->part_info->num_partitions_used() > 1 &&
                            partition_key_modified(table, table->write_set);
```

两阶段的实现（`sql_update.cc:645` 起）：

```cpp
    if (used_key_is_modified || order) {          /* ★★★ 入口 */
      /*
        We can't update table directly;  We must first search after all
        matching rows before updating the table!
      */
        /*
          We are doing a search on a key that is updated. In this case
          we go through the matching rows, save a pointer to them and
          update these in a separate loop based on the pointer. ...
          TODO: Find something less ugly.
        */
        table->prepare_for_position();
        ...
        while (!(error = iterator->Read()) && !thd->killed) {   /* ★ 阶段一：只扫描，不更新 */
          if (conds != nullptr) { ...skip... }
          table->file->position(table->record[0]);              /* ★ 取 rowid（InnoDB = 主键值） */
          if (my_b_write(tempfile, table->file->ref, table->file->ref_length)) { ... }
        }
        table->file->ha_index_or_rnd_end();                     /* ★★ 扫描彻底结束 */
        iterator.reset();

        iterator = NewIterator<SortFileIndirectIterator>(        /* ★ 换迭代器：读 rowid 文件 */
            thd, thd->mem_root, Mem_root_array<TABLE *>{table}, tempfile, ...);
        conds = nullptr;                                         /* WHERE 已在阶段一过滤完 */
    } else {
      // No ORDER BY or updated key underway, so we can use a regular read.
      iterator = init_table_iterator(...);                       /* ★ 边扫边改 */
    }
```

> ★★★ `SortFileIndirectIterator` 内部靠 `ha_rnd_pos()` 按 rowid 定位。InnoDB 的 `rnd_pos` 就是 `index_read(buf, pos, ref_length, HA_READ_KEY_EXACT)`（`ha_innodb.cc:10862`）——**按主键精确定位，与扫描游标无关**。所以更新导致的行搬家影响不到阶段二。

**★ 一个容易漏掉的豁免**：`get_index_for_order()` 对"唯一键等值/单行 range"直接返回 `MAX_KEY`（`sql_select.cc:5330`）：

```cpp
  if (range_scan && unique_key_range(range_scan)) {
    *need_sort = false;
    /*
      Returning of MAX_KEY here prevents updating of used_key_is_modified
      in mysql_update(). Use AccessPath "as is".
    */
    return MAX_KEY;
  }
```

> ★ 即 `UPDATE t SET a=1 WHERE pk=5` 不可能重复扫，**故意不设** `used_key_is_modified`，省掉物化。

**★ 防线 2：多表 UPDATE 默认全部物化**（比单表更强）

`Query_result_update::optimize()`（`sql_update.cc:2184`）给**每张被更新表**建一张临时表，首列是 rowid 且在其上建唯一键：

```cpp
    /* Make an unique key over the first field to avoid duplicated updates */
    group->direction = ORDER_ASC;
    group->item = &temp_fields.front();
    ...
    tmp_tables[cnt] = create_tmp_table(thd, tmp_param, temp_fields, group, ...);
```

只有 `safe_update_on_fly()` 放行的**最外层表**才 on-the-fly（`sql_update.cc:2036`，逐 join type 防守）：

```cpp
    case JT_REF:
    case JT_REF_OR_NULL:
      // If the key is updated, it may change from non-NULL-constant to NULL,
      // so with JT_REF_OR_NULL, the row would be read twice.
      return !is_key_used(table, join_tab->ref().key, table->write_set);
    case JT_RANGE:
    case JT_INDEX_MERGE:
      // When scanning a range, it's possible to read a row twice if it moves
      // within the range:
      if (uses_index_on_fields(join_tab->range_scan(), table->write_set))
        return false;
      [[fallthrough]];
    case JT_ALL:
      // If using the clustered key under the cover:
      if ((table->file->ha_table_flags() & HA_PRIMARY_KEY_IN_READ_INDEX) &&
          table->s->primary_key < MAX_KEY)
        return !is_key_used(table, table->s->primary_key, table->write_set);
      return true;
```

**★ 防线 3：非法语句直接报错**

- `unsafe_key_update()` → `ER_MULTI_UPDATE_KEY_CONFLICT`（同一基表经两个别名被更新 + 更新会搬行）
- `unique_table()` → `ER_UPDATE_TABLE_USED`（*"You can't specify target table '%-.192s' for update in FROM clause"*）

> ⚠️ `unsafe_key_update()` **只在 `update_table_count_local > 1` 时被调用**（`sql_update.cc:1713`）——**单表 UPDATE 根本不走它**。所以它不是"扫描防重"，而是"多别名语义冲突"检查。

#### ★★★ 三处"看起来像防线但其实不是"（证伪）

| 机制 | 为什么不是 |
|---|---|
| **`HA_READ_BEFORE_WRITE_REMOVAL`** | 是 NDB 的"去掉读、直接按 key 写"往返优化。**InnoDB 未声明该 flag**（`ha_innodb.cc:2973` 的 `m_int_table_flags` 里没有；`ha_innopart.h:51` 明说不支持），所以 `sql_update.cc:861` 与 `:1448` 两处**对 InnoDB 永远为假**。源码自己注释：`read_removal is only used by NDB storage engine` |
| **`try_semi_consistent_read` / `was_semi_consistent_read` / `unlock_row`** | 是 RC/RU 下"不加锁乐观读 → 命中 WHERE 则丢弃并加锁**重读同一行**"的**并发锁优化**（`handler.h:5645` 有权威注释）。它的语义是"重读"，不是"避免重复扫到" |
| **`btr_pcur` 的 cursor restoration** | 存/恢复的是**B-tree 物理记录位置**（`old_rec` + `m_rel_pos`），**不是行身份（PK 值 / undo 版本号）**；恢复失败就 `move_to_next`（`row0sel.cc:3443`）——**这正是重复访问的来源**。InnoDB TODO 缺的就是"读时检查 undo number" |

#### ★★ A 侧并没有被完全防住——官方测试自己在兜底

`mysql-test/include/mix2.inc:909-911` 与 `mysql-test/suite/innodb/t/innodb.test:970-972`：

```
# We need the a < 1000 test here to quard against the halloween problems
UPDATE t1 set a=a+100 where b between 2 and 3 and a < 1000;
```

> ★★★ 这个用例**故意加了 `a < 1000` 这个不会被新值再次落入的上界**来终止可能的重复更新。它为什么能漏网？按上面的判据逐条推：range scan 走的是 **b** 上的唯一索引 → `used_index` = b 的索引号 → `is_key_used(table, b_idx, write_set={a})` = **false**（a 不在 b 的索引列里）→ `used_key_is_modified == false` → 走边扫边改；而更新 `a`（**聚簇主键**）会让行在聚簇索引里物理搬家。
>
> ★★ 所以防线**只覆盖"扫描索引 == 被更新索引"这一条自反馈路径**，漏掉的是**交叉场景**：扫描走索引 X，而 UPDATE 让行在索引 Y / 聚簇索引中前移，导致扫描游标绕回来。
- 多重集（multiset）语义 —— `INTERSECT ALL` / `EXCEPT ALL` 要求重复度语义，这是 MySQL 选择计数器算法而非哈希集合的根本原因

**官方文档**

- MySQL 8.0 Reference Manual, "INTERSECT / EXCEPT / UNION Clause"（8.0.31 起支持）
- MySQL 8.0 Reference Manual, "Optimizing UNION"

**相关文档**

- [`../05_contextualize.md`](../05_contextualize.md) —— `Query_term` 的解析、标签分配、`pushdown_limit_order_by`
- [`../runtime/08_materialization.md`](../runtime/08_materialization.md) —— 物化与临时表的运行期细节
- [`../09_executor_iterator.md`](../09_executor_iterator.md) —— `AppendIterator` / `MaterializeIterator` / `TableScanIterator`
- [`16_join_object_model.md`](16_join_object_model.md) —— `m_root_iterator` 归属于 unit（本篇的执行入口）
- [`../08_access_path.md`](../08_access_path.md) —— AccessPath 类型（`APPEND` / `MATERIALIZE` / `LIMIT_OFFSET`）
