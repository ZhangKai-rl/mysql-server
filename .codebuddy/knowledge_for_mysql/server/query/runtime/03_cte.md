# 03 CTE 与递归 CTE：WITH 的解析与执行（算法级）

> 基于 MySQL 8.0.39 源码。本篇覆盖：`WITH` 如何解析与存储、非递归 CTE 为何就是 derived table、merge 与 materialize 的决策链、**物化表的自动建索引**、递归 CTE 的 seed + 收敛循环、共享物化，以及递归的陷阱与性能特征。
>
> **边界**：本篇讲 CTE 的执行期实现。优化器侧"CTE 为何不能 merge、与 derived 的结构差异"见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)；物化用的临时表本身见 [`07_temptable.md`](07_temptable.md)。

## 目录

- [先有个整体印象](#先有个整体印象)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、WITH 的解析与存储](#一with-的解析与存储)
- [二、非递归 CTE：就是 derived table](#二非递归-cte就是-derived-table)
- [三、物化表的自动建索引](#三物化表的自动建索引)
- [四、递归 CTE 的核心算法](#四递归-cte-的核心算法)
- [五、共享物化与重扫](#五共享物化与重扫)
- [六、递归的限制、陷阱与性能特征](#六递归的限制陷阱与性能特征)
- [七、深潜补充：克隆表、校验清单、递归的三个细节](#七深潜补充克隆表校验清单递归的三个细节)
- [参考](#参考)

---

## 先有个整体印象

> ⚠️ **版本事实**：8.0.39 **没有** `sql/sql_cte.h/.cc`。CTE 逻辑已分散到 `sql_derived.cc`、`sql_union.cc`、`sql_parse.cc`、`parse_tree_nodes.cc`。后解析数据结构叫 `Common_table_expr`。

CTE 的本质：**非递归 CTE 在解析层就被做成了 derived table**（复用 merge/materialize 全部机制），差别只在"多次引用共享一次物化"；**递归 CTE** 则多出"seed + 递归步收敛循环"，核心是 `FollowTailIterator` 读"上一轮新增的行"。

```
非递归 CTE ──▶ 就是 derived table（merge 或 materialize）
递归 CTE   ──▶ MaterializeIterator::MaterializeRecursive：
                 1. seed（anchor）物化一次
                 2. 递归块反复物化，直到不再新增行（收敛）
```

---

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 为什么非递归 CTE 要走 derived table 这条路

CTE 和 derived table（`FROM (SELECT ...)`）在语义上几乎等价。MySQL 的选择是**不另起炉灶**——`find_common_table_expr` 找到 CTE 定义后，直接用它构造一个 `PT_subquery` 并标记为 derived table，从而完整复用已有的 `merge_derived` / `materialize_derived` 机制。

**收益**：优化器的 merge 决策、物化、临时表管理、共享引用全部白拿，代码量极小。
**代价**：CTE 也一并继承了 derived table 的全部限制（不是所有 CTE 都能 merge）。

**★ 源码证据：一句注释 + 三行代码**（`sql/sql_parse.cc:5774`）：

```cpp
  // We imitate derived tables as much as possible.
  assert(parsing_place == CTX_NONE && linkage != GLOBAL_OPTIONS_TYPE);
  parsing_place = CTX_DERIVED;
  node->m_is_derived_table = true;
```
```cpp
  tl->is_alias = true;
  Query_expression *node_query_expression =
      node->value()->master_query_expression();
  *table_name = Table_ident(node_query_expression);
  assert(table_name->is_derived_table());       /* ★ 命中 CTE 后 Table_ident 被改写成派生表 */
```
> ★★ `find_common_table_expr()` 命中后直接把 `Table_ident` **改写成 `Table_ident(Query_expression)`**（一个派生表），并 `assert(is_derived_table())`。从此以后解析器眼里就只有派生表，不再有"CTE"这个概念。
> ★ 递归引用则先塞一个 `"(select 0)"` 的 dummy 派生表占位：
> ```cpp
>   if (tl->is_recursive_reference()) {
>     LEX_CSTRING dummy_subq = {STRING_WITH_LEN("(select 0)")};
>     if (reparse_common_table_expr(thd, dummy_subq.str, dummy_subq.length, 0, &node))
>       return true;
>   } else if (cte->make_subquery_node(thd, &node))
>     return true;
> ```
> ★ 查找顺序：**先 CTE 再真实表**（由 `add_table_to_list()` 调用，`(table_options & TL_OPTION_ALIAS) == 0` 时才查）；作用域沿 `outer_query_block()` **由内向外冒泡**。
```cpp
  do {
    unit = select->master_query_expression();
    if (!(wc = unit->m_with_clause)) continue;
    if (wc->lookup(tl, &cte)) return true;
    /* If no match in the WITH clause of 'select', maybe this is a subquery, so
       look up in the outer query's WITH clause: */
  } while (cte == nullptr && (select = select->outer_query_block()));
```
> ★ 一个容易被忽略的机制：**同一 CTE 被引用第二次起，必须按保存的原始文本重新解析一份独立的 PT 树**（`sql_parse.cc:5713`）：
> ```cpp
> bool PT_common_table_expr::make_subquery_node(THD *thd, PT_subquery **node) {
>   if (m_postparse.references.size() >= 2) {
>     // m_subq_node was already attached elsewhere, make new node:
>     return reparse_common_table_expr(thd, m_subq_text.str, m_subq_text.length,
>                                      m_subq_text_offset, node);
>   }
>   *node = m_subq_node;
>   return false;
> }
> ```
> 这也是语法产生式为什么要保存 `subq_text` + `subq_text_offset` 的原因。

#### 2. merge 还是 materialize：一场经典的权衡

非递归 CTE（derived table）有两种执行方式：

| | **merge**（合并进外层查询） | **materialize**（物化成临时表） |
|---|---|---|
| 做法 | 把 CTE 的定义**展开**进外层 WHERE/JOIN，当成一个整体优化 | 先执行 CTE 把结果写进临时表，再当普通表访问 |
| 优点 | 外层谓词可**下推**进 CTE、能用一个全局 join order、可能避免物化 | 只执行一次（多次引用共享）、阻断优化爆炸 |
| 缺点 | 复杂 CTE 展开后会拖垮 join 搜索空间 | 谓词下推被阻断、额外写临时表、可能因降级到磁盘而变慢 |

MySQL 默认**倾向 merge**（`derived_merge` 默认 ON），只要结构允许就合并。这与 PostgreSQL 12 之前"CTE 总是物化"形成鲜明对比（见他库对比）。

**★ merge 的一个副作用：WITH 子句会消失**（`sql/iterators/composite_iterators.cc:900`，★ 官方注释讲得很清楚）：

```cpp
  if (m_cte != nullptr) {
    // This is needed in a special case. Consider:
    // SELECT FROM ot WHERE EXISTS(WITH RECURSIVE cte (...)
    //                             SELECT * FROM cte)
    // and assume that the CTE is outer-correlated. When EXISTS is
    // evaluated, Query_expression::ClearForExecution() calls
    // clear_correlated_query_blocks(), which scans the WITH clause and clears
    // the CTE, including its references to itself in its recursive definition.
    // But, if the query expression owning WITH is merged up, e.g. like this:
    // FROM ot SEMIJOIN cte ON TRUE,
    // then there is no Query_expression anymore, so its WITH clause is
    // not reached. But this "lateral CTE" still needs comprehensive resetting.
    // That's done here.
    if (m_cte->clear_all_references()) return true;
  }
```
> ★★ CTE 被 semijoin merge 掉后 **`Query_expression` 不存在了**，`clear_correlated_query_blocks()` 扫不到它的 WITH——只能由 `MaterializeIterator` 直接调 `m_cte->clear_all_references()` 兜底。这是"merge 省了一层，但生命周期管理要另找地方"的典型代价。

**★ 共享与否的运行期判定**（`composite_iterators.cc:841`，⚠️ **没有条件变量、没有读写锁、没有引用计数**）：

```cpp
  const bool use_shared_cte_materialization =
      !table()->materialized && m_cte != nullptr && !m_rematerialize &&
      any_of(m_cte->tmp_tables.begin(), m_cte->tmp_tables.end(),
             [](const Table_ref *table_ref) {
               return table_ref->table != nullptr &&
                      table_ref->table->materialized;
             });

  if (use_shared_cte_materialization) {
    for (Invalidator &invalidator : m_invalidators) {
      invalidator.generation_at_last_materialize =
          invalidator.iterator->generation();
    }
    table()->materialized = true;    /* ★ 直接置位跳过写表 */
  }
```
> ★ "同步"就是三步：① 扫 `tmp_tables` 找任一已 `materialized` 的 clone；② 拉齐 invalidator generation；③ 自己也置 `materialized=true`。**单线程执行**，读者只是 clone 出的不同 `TABLE` 对象。
> ★ `rematerialize` 与 `use_shared_cte_materialization` **互斥**——相关 CTE 每次 `Init()` 都要重算，因此不能共享。

#### 3. 递归 CTE 为什么必须是"迭代 + 临时表"，而不是真递归

递归 CTE（`WITH RECURSIVE`）在语义上是**不动点迭代**：从 anchor 出发反复应用递归规则，直到不再产生新行。

MySQL 的实现是**执行期的迭代循环**（`MaterializeRecursive` 里的 `do...while`），不是调用栈意义上的递归：

- 每一轮把新产生的行**追加**进同一张物化临时表
- 递归引用用一个特殊迭代器 `FollowTailIterator` 只读"上一轮新增的行"
- 收敛条件：某一轮没有新增任何行

**为什么用迭代而不是递归**：迭代让"已算出的行"天然落在一张表里，递归引用就是一次顺序扫描；同时深度由数据决定而非调用栈，避免了栈溢出。代价是**每轮都要重新扫描新增行**，复杂度随数据规模上升（见性能特征）。

**★★★ 官方对"非标准但等价"的自述**（`sql/iterators/composite_iterators.cc:979`，★ 本节最权威的一段）：

```cpp
/**
  Recursive materialization happens much like regular materialization,
  but some steps are repeated multiple times. Our general strategy is:

    1. Materialize all non-recursive query blocks, once.

    2. Materialize all recursive query blocks in turn.

    3. Repeat #2 until no query block writes any more rows (ie., we have
       converged) -- for UNION DISTINCT queries, rows removed by deduplication
       do not count. Each materialization sees only rows that were newly added
       since the previous iteration; see FollowTailIterator for more details
       on the implementation.

  Note that the result table is written to while other iterators are still
  reading from it; again, see FollowTailIterator. This means that each run
  of #2 can potentially run many actual CTE iterations -- possibly the entire
  query to completion if we have only one query block.

  This is not how the SQL standard specifies recursive CTE execution
  (it assumes building up the new result set from scratch for each iteration,
  using the previous iteration's results), but it is equivalent, and more
  efficient for the class of queries we support, since we don't need to
  re-create the same rows over and over again.
 */
```
> ★★ MySQL 用的是**增量 delta 迭代**（semi-naive 风格，每轮只看上一轮新增的行），而 SQL 标准描述的是"每轮从上一轮完整结果集重建"。MySQL 明说"**不等同标准写法，但等价，且更高效，因为不必反复重建同样的行**"。

**★ 收敛循环本体**（`composite_iterators.cc:1080`）：

```cpp
  // First, materialize all non-recursive query blocks.
  for (const materialize_iterator::QueryBlock &query_block :
       m_query_blocks_to_materialize) {
    if (!query_block.is_recursive_reference) {
      if (MaterializeQueryBlock(query_block, &stored_rows)) return true;
    }
  }

  // Then, materialize all recursive query blocks until we converge.
  ha_rows last_stored_rows;
  do {
    last_stored_rows = stored_rows;
    for (const materialize_iterator::QueryBlock &query_block :
         m_query_blocks_to_materialize) {
      if (query_block.is_recursive_reference) {
        if (MaterializeQueryBlock(query_block, &stored_rows)) return true;
      }
    }
    ...
  } while (stored_rows > last_stored_rows);   /* ★★ 固定点（fixpoint）判据 */
```

**★ 迭代算法对 join order 有硬约束**（`sql/sql_optimizer.cc:5495`，★ 这是"必须迭代+临时表"的强力注脚）：

```cpp
    tab->dependent = tl->dep_tables;  // Initialize table dependencies
    if (query_block->is_recursive()) {
      if (query_block->recursive_reference != tl)
        // Recursive reference must go first
        tab->dependent |= query_block->recursive_reference->map();
      else {
        // Recursive reference mustn't use any index
        table->covering_keys.clear_all();
        table->keys_in_use_for_group_by.clear_all();
        table->keys_in_use_for_order_by.clear_all();
      }
    }
```
> ★★ 递归表**强制最左**，且**所有索引位图被清空**——每轮只能全表尾随扫描。hypergraph 优化器同样如此（`join_optimizer.cc:415` 的 `forced_leftmost_table` 注释：*"they work by continuously tailing new records, which wouldn't work if we were to scan them multiple times or put them in a hash table"*）。

#### 4. 为什么没有环检测：一个明确的设计取舍

如果递归产生环（如 `SELECT n+1 FROM cte WHERE ...` 永远能产生新行），MySQL **不会**检测"这一行是不是已经出现过"。它只靠 `cte_max_recursion_depth`（默认 1000）在**轮数**上兜底。

**这是个权衡**：

- **检测环的代价**：每产生一行都要在整张已物化表里查找是否已存在（O(n) 甚至需要额外索引），对合法的深递归是纯浪费
- **不检测的代价**：有环时只能跑到深度上限然后报错，浪费了一千轮的计算

MySQL 选择后者——把"写对递归终止条件"的责任交给用户。（PostgreSQL 同样不做环检测。）

**顺带的坑**：因为不检测重复行，如果递归逻辑本身会重复产生已有行（且不是环），MySQL 会**一直产生直到深度上限**——即使逻辑上"早该停了"。

**★ 源码里能取证的"防环"其实有三处**（★ 不是"完全不管"）：

**① 解析期禁止前向引用 = 结构性防环**（`sql/sql_parse.cc:5831`）：

```cpp
      if (el->is(right_bound)) {
        /*
          We meet right_bound.
          If not RECURSIVE:
          we must stop the search in this WITH clause;
          indeed right_bound must not reference itself or any CTE defined after it
          in the WITH list (forward references are forbidden, preventing any
          cycle).
          If RECURSIVE:
          If right_bound matches 'tl', it is a recursive reference.
        */
        if (!m_recursive) break;  // Prevent forward reference.
        in_self = true;           // Accept a recursive reference.
      }
```
> ★★ `preventing any cycle` —— 禁前向引用就是环检测的替代方案。RECURSIVE 下还有第二道 `if (in_self) break;`（`sql_parse.cc:5844`）：走到自己之后仍没匹配就停止，**递归 CTE 也只能引用"自己"或"自己左边已定义的 CTE"**。

**② 互递归不支持**（`mysql-test/include/with_recursive.inc:6`）：

```
# Mutual recursion unsupported; cycles must have one node only
--error ER_NO_SUCH_TABLE
with recursive qn as (select * from qn2),
               qn2 as (select * from qn)
select * from qn;
```
> ⚠️ **没有专门错误码**，报 `ER_NO_SUCH_TABLE`。

**③ 单节点自环交给深度上限兜底**（`sql/iterators/basic_row_iterators.cc:399`，★ 唯一的运行期检查点）：

```cpp
  if (m_read_rows == m_end_of_current_iteration) {
    // We have started a new iteration. Check to see if we have passed the
    // user-set limit.
    if (++m_recursive_iteration_count >
        thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;
    }
    m_end_of_current_iteration = *m_stored_rows;
```
> ★ 注意检查点在 `FollowTailIterator::Read()`，不是 UNION 侧。

**★ "为什么不只发 warning"的官方解释**（`composite_iterators.cc:1023`，★ 也是"没有 MAXRECURSION hint"的证据）：

```cpp
    If we only raised warnings: it will not interrupt an infinite recursion,
    a MAX_RECURSION hint (if we featured one) may interrupt; but then the
    warnings won't be seen, as the interruption will raise an error. So
    warnings are useless.
    Instead, we send a truncation error: it is visible, indicates the
    source of the problem, and is consistent with the INSERT case above.

    Usually, truncation in SELECT triggers an error only in
    strict mode; but if we don't send an error we get a runaway query;
    and as WITH RECURSIVE is a new feature we don't have to carry the
    permissiveness of the past, so we send an error even if in non-strict
    mode.
```
> ★★ 即：**没有 SQL:2011 的 `CYCLE` 子句，也没有 SQL Server 的 `MAXRECURSION` hint**，只有"截断报错 + 深度上限"两道防线。且递归物化期间**强制开启严格模式**（`Strict_error_handler`）。

### 理论溯源

递归 CTE 来自 **SQL:1999 标准**的递归查询（recursive query），理论基础是**最小不动点（least fixed point, LFP）**语义：

```
结果 = 反复应用递归规则，直到结果集不再增长（到达不动点）
```

这也是为什么 SQL 标准要求递归 CTE 必须是 `UNION`（或 `UNION ALL`）+ anchor/递归两段结构——它在形式上是标准的 LFP 迭代。

`WITH RECURSIVE` 的这个结构约束在 MySQL 里有硬校验（见后面"结构校验"）。

**★ 源码里直接引用标准的 5 处证据**（★ "理论溯源"的硬证据）：

| 源码位置 | 原文 |
|---|---|
| `sql/sql_union.cc:825` | `// Per SQL2011.`（递归块禁用聚合/窗口函数） |
| `sql/sql_optimizer.cc:5322` | `SQL2011 forbids: WITH RECURSIVE rec AS (... UNION ALL SELECT ... FROM tbl LEFT JOIN rec ON...)` |
| `sql/sql_union.cc:786` | `holder->set_nullable(true); // Always nullable, per SQL standard.` |
| `composite_iterators.h:485` | `there can be at most one recursive reference in a join list, as per the SQL standard` |
| `sql/sql_derived.cc:358` | `In SQL standard terms, the CTE must be "expandable" except that we allow it to have more than one recursive SELECT.` |

```cpp
/* sql/sql_union.cc:797 —— ★ 递归块不参与类型推导（这是"LFP 由 anchor 定种子"的代码体现） */
      if (sl->recursive_reference) {
        /*
          Recursive query blocks don't determine output types of the result.
          The only thing to check could be if the recursive query block has a
          type which can't be cast to the output type of the result.
          But in MySQL, all types can be cast to each other (at least during
          resolution; an error may reported when trying to actually insert, for
          example an INT into a POINT). So no further compatibility check is
          needed here.
        */
      } else {
        auto it = sl->visible_fields().begin();
        auto tp = types.begin();
        for (; it != sl->visible_fields().end() && tp != types.end(); ++it, ++tp) {
          if ((down_cast<Item_type_holder *>(*tp))->join_types(thd, *it))
            return true;
        }
      }
```
> ★★ 递归块的字段**根本不参与 `Item_type_holder::join_types()`**——注释原话："Recursive query blocks don't determine output types of the result."

**★ `RECURSIVE_SYM` 是 SQL-1999 保留字**（`sql/sql_yacc.yy:1252`）：

```cpp
%token RECURSIVE_SYM 915 /* SQL-1999-R */
```
> ★ 这印证了 CTE 能力来自 **SQL-1999 标准**，而非某个 MySQL 小版本。

### 算法与数据结构

- **`Common_table_expr`**：CTE 的运行时结构，三个关键成员——`references`（定义之外的非递归引用）、`recursive`（是否递归）、`tmp_tables`（共享物化的所有 `Table_ref`）
- **`FollowTailIterator`**：递归 CTE 的核心迭代器，读"上一轮新增的行"，且**不让底层游标真正撞 EOF**
- **`MaterializeIterator`**：物化的通用算子，`is_recursive()` 时走 `MaterializeRecursive()`
- **`Derived_key`**：物化表上自动建立的"可用键"描述（见下一节）

**★★ `Common_table_expr`：准确说是"两个数组 + 一个 bool"**（`sql/table.h:4332`，⚠️ 常见说法"三个数组"是错的）：

```cpp
/**
  After parsing, a Common Table Expression is accessed through a
  Table_ref. This class contains all information about the CTE which the
  Table_ref needs.

  @note that before and during parsing, the CTE is described by a
  PT_common_table_expr.
*/
class Common_table_expr {
 public:
  Common_table_expr(MEM_ROOT *mem_root)
      : references(mem_root), recursive(false), tmp_tables(mem_root) {}
  TABLE *clone_tmp_table(THD *thd, Table_ref *tl);
  bool substitute_recursive_reference(THD *thd, Query_block *sl);
  /// Remove one table reference.
  void remove_table(Table_ref *tr);
  /// Empties the materialized CTE and informs all of its clones.
  bool clear_all_references();
  /**
     All references to this CTE in the statement, except those inside the
     query expression defining this CTE.
     In other words, all non-recursive references.
  */
  Mem_root_array<Table_ref *> references;
  /// True if it's a recursive CTE
  bool recursive;
  /**
    List of all TABLE_LISTSs reading/writing to the tmp table created to
    materialize this CTE. Due to shared materialization, only the first one
    has a TABLE generated by create_tmp_table(); other ones have a TABLE
    generated by open_table_from_share().
  */
  Mem_root_array<Table_ref *> tmp_tables;
  /// Name of the WITH block. Used only for EXPLAIN FORMAT=tree.
  LEX_STRING name;
};
```
> ⚠️ 真名核查三条：① **`Common_table_expr` 在 `sql/table.h:4340`**，不在 `sql/sql_lex.h`（后者只有 `Common_table_expr_parser_state`）；② **不存在 `With_clause` 类**，只有 parse-tree 期的 `PT_with_clause`；③ **不存在 `recursive_references` 数组**，"三个数组"的错觉来自 `Query_block::recursive_reference`（单指针）与 `Query_expression::first_recursive`（单指针）。
> ★ `references`（外部非递归引用）与 `tmp_tables`（所有读/写该临时表的 `Table_ref`，含递归 clone）**分两个数组**是有原因的——见 `clear_all_references()`。

**★ `FollowTailIterator`：三个计数器 + 一个裸指针**（`sql/iterators/basic_row_iterators.h:439`，⚠️ 不在 `composite_iterators.h`）：

```cpp
 private:
  bool m_inited = false;
  uchar *const m_record;
  const double m_expected_rows;
  ha_rows *const m_examined_rows;
  ha_rows m_read_rows;                    /* 已读行数 */
  ha_rows m_end_of_current_iteration;     /* 本轮迭代结束边界 = 进入本轮时表里的行数 */
  unsigned m_recursive_iteration_count;   /* 迭代轮数 */

  // Points into MaterializeIterator's data; set by BeginMaterialization() only.
  ha_rows *m_stored_rows = nullptr;       /* ★ 指向 MaterializeIterator 栈上 stored_rows 的裸指针 */
```
> ★★ "读上一轮新增的行"的**全部机制**就是这三个计数器的对比——**不是靠 handler 游标位置**。`m_stored_rows` 指向 `MaterializeIterator::Init()` 的**栈上局部变量** `stored_rows`（`composite_iterators.cc:1056` 通过 `set_stored_rows_pointer(&stored_rows)` 共享）。

**★ 为什么"不让游标真撞 EOF"**（`basic_row_iterators.cc:317`，★ 一段极详细的注释）：

```cpp
int FollowTailIterator::Read() {
  if (m_read_rows == *m_stored_rows) {
    /*
      Return EOF without even checking if there are more rows
      (there isn't), so that we can continue reading when there are.
      There are two underlying reasons why we need to do this,
      depending on the storage engine in use:

      1. For both MEMORY and InnoDB, when they report EOF,
         the scan stays blocked at EOF forever even if new rows
         are inserted later. (InnoDB has a supremum record, and
         MEMORY increments info->current_record unconditionally.)

      2. Specific to MEMORY, inserting records that are deduplicated
         away can corrupt cursors that hit EOF. ...
         - write 'A'  →  - write 'A': dup error, leaves "deleted record" at #1
         - read: finds 'A' at #0  → read: skips deleted #1, moves to EOF
         - even if we save the read position, it's "after #1"
         - write 'B' takes the place of deleted record, i.e. writes at #1
         - reposition at saved position → still after #1, so misses 'B'.
       ...
     */
    return -1;
  }
```
> ★★ 这段注释解释了为什么必须"数行"而不是"记位置"：**MEMORY 引擎下被去重掉的写入会留下 deleted record，重新定位会漏行**。

**★ `materialize_iterator::QueryBlock`**（`composite_iterators.h:427`）：

```cpp
  /// If copy_items is true, used for copying the Field objects
  Temp_table_param *temp_table_param;

  // Whether this query block is a recursive reference back to the
  // output of the materialization.
  bool is_recursive_reference = false;

  // If is_recursive_reference is true, contains the FollowTailIterator
  // in the query block (there can be at most one recursive reference
  // in a join list, as per the SQL standard, so there should be exactly one).
  // Used for informing the iterators about various shared state in the
  // materialization (including coordinating rematerializations).
  FollowTailIterator *recursive_reader = nullptr;
```

**★ `Derived_refs_iterator`**（`sql/table.h:4369`，★ 遍历某 CTE 的所有 TABLE 克隆）：

```cpp
  TABLE *get_next() {
    const Common_table_expr *cte = start->common_table_expr();
    m_is_first = ref_idx == 0;
    // Derived tables and views have a single reference.
    if (cte == nullptr) {
      return ref_idx++ == 0 ? start->table : nullptr;
    }
    /*
      CTEs may have multiple references. Return the next one, but notice that
      some references may have been deleted.
    */
    while (ref_idx < cte->tmp_tables.size()) {
      TABLE *table = cte->tmp_tables[ref_idx++]->table;
      if (table != nullptr) return table;
    }
    return nullptr;
  }
```

### 他库对比与演进动机

| | MySQL 8.0 | PostgreSQL | SQL Server |
|---|---|---|---|
| 非递归 CTE 默认 | **倾向 merge**（`derived_merge=ON`） | **12 之前总是物化**（著名的 optimization fence，谓词下推被阻断）；12 起可 `NOT MATERIALIZED` | 通常展开（类似 merge） |
| 递归实现 | 迭代 + 临时表 + `FollowTailIterator` | 迭代 + tuplestore（WorkTable Scan） | 迭代 + 栈式 spool |
| 环检测 | ❌ 无（靠深度上限） | ❌ 无 | ❌ 无 |
| 递归深度限制 | `cte_max_recursion_depth`（默认 1000） | 无限制（靠 `max_stack_depth`/内存） | `MAXRECURSION`（默认 100，可用 hint 改到 32767） |

> **PG 的教训值得记住**：PG 12 之前"CTE 总是物化"导致大量性能问题（外层谓词下推不进 CTE，CTE 被全量执行），社区为此呼吁多年，12 才允许 `NOT MATERIALIZED`。MySQL 从一开始就走 merge 优先，避开了这个坑——但代价是复杂 CTE 展开后可能拖慢 join 搜索。

**★ 源码里唯一的他库直接引用：Oracle 的 ORA-32042**（`sql/sql_parse.cc:5860`）：

```cpp
      if (in_self && tl->query_block->outer_query_block() !=
                         m_most_inner_in_parsing->query_block) {
        /*
          SQL2011 says a recursive CTE cannot contain a subquery
          referencing the CTE, except if this subquery is a derived table
          like:
          WITH RECURSIVE qn AS (non-rec-SELECT UNION ALL
          SELECT * FROM (SELECT * FROM qn) AS dt)
          However, we don't allow this, as:
          - it simplifies detection and substitution correct recursive
          references (they're all on "level 0" of the UNION)
          - it's not limiting the user so much (in most cases, he can just
          merge his DT up manually, as the DT cannot contain aggregation).
          - Oracle bans it:
          with qn (a) as (
          select 123 from dual
          union all
          select 1+a from (select * from qn) where a<130) select * from qn
          ORA-32042: recursive WITH clause must reference itself directly in one
          of the UNION ALL branches.
          ...
        */
        my_error(ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE, MYF(0),
                 el->name().str);
        return true;
      }
```
> ★★ 递归引用必须与 UNION 处在**同一层**（level 0），放在子查询/派生表里即报错。理由三条，第三条是"Oracle 也禁"。

**★ 没有 `MAXRECURSION` hint 的源码证据**（`composite_iterators.cc:1024`）：

```cpp
    a MAX_RECURSION hint (if we featured one) may interrupt
```
> ★ 括号里的 `if we featured one` 说明 **MySQL 8.0.39 没有 SQL Server 风格的 `OPTION (MAXRECURSION n)`**，只有 `cte_max_recursion_depth`（且它是 `HINT_UPDATEABLE`，可以用 `SET_VAR` hint 改）。

**★ 执行语义与标准的差异**（已在「权衡三」给出原文，此处只记结论）：MySQL 用**增量 delta 迭代**，标准描述的是"每轮从上一轮完整结果集重建"——**不等价于标准写法，但等价且更高效**。

> ⚠️ **说明**：8.0.39 源码中 grep `PostgreSQL|SQL Server|DB2` 在 CTE 相关文件里**零命中**，可用的他库证据只有上面这条 Oracle + 多处 `per SQL2011` / `as per the SQL standard` 引用。

### 版本演进与能力清单

**引入历史：8.0 首次引入，5.x 完全没有。**

源码最硬的证据是一处注释直称 CTE 为"新特性"（`composite_iterators.cc` 的递归物化里，讲为什么递归 CTE 的截断**不继承旧 SELECT 的宽松行为**）：

```cpp
// ...as WITH RECURSIVE is a new feature we don't have to carry the
// permissiveness of the past, so we send an error even if in non-strict mode.
```

"新特性、不必背负历史包袱"——这句话同时证明了两点：CTE 是后加的、且设计者有意让它更严格。开发按三个工作日志推进（测试文件头可证）：

| 工作日志 | 内容 |
|---|---|
| `WL#883` | 非递归 `WITH`（`with_non_recursive.inc` 首行） |
| `WL#3634` | 递归 CTE（`with_recursive_bugs.test` 反复出现） |
| `WL#9248` | 递归所需的 InnoDB 临时表配套改动 |

`cte_max_recursion_depth` 由 `Bug#26136509` 引入（测试文件头明写 "ADD A MAX_RECURSION VARIABLE TO LIMIT RECURSION IN CTES"）。

⚠️ **诚实标注**：具体到 8.0.14 / 8.0.19 / 8.0.22 这些小版本的精确改动，**本源码树没有版本号标注**（`8.0.` 正则搜 `with_*` 测试 0 命中），需结合官方 Release Notes。源码能证明的是"能力落点"，不能证明"哪个小版本引入"。

**关键纠偏：非递归 CTE 不是"总是物化"。**

这是最容易误解的一点。MySQL 从 CTE 诞生起就走 derived table 的 merge/materialize 决策链——**非递归 CTE 默认倾向 merge，`derived_merge` 开关和 `MERGE/NO_MERGE` hint 都对它生效**。`Table_ref::is_mergeable()` 里甚至专门为 CTE 写了分支：

```cpp
// If the table's content is non-deterministic and the query references it
// multiple times, merging it has the risk of creating different contents.
Common_table_expr *cte = common_table_expr();
if (cte != nullptr && cte->references.size() >= 2 &&
    derived->uncacheable & UNCACHEABLE_RAND)
  return false;   // 多次引用 + RAND 才禁止 merge
```

只有**递归 CTE 必须物化**（迭代 + 临时表，`derived_merge` 不参与）。

**8.0 后期可源码佐证的能力演进**（版本号需外部 changelog 交叉定位）：

- **EXPLAIN 专门显示 CTE**：`Common_table_expr::name` 字段注释 "Used only for EXPLAIN FORMAT=tree"；`explain_access_path.cc` 打印 "Materialize CTE / recursive CTE / union CTE"；`with_explain.test` 明确"多次引用只在第一处展开、其余显示 `<derivedN>`，让用户看懂是单次物化"。
- **hypergraph 优化器改变 CTE 计划**：测试里大量 `--skip_if_hypergraph # Chooses a different plan`，是 8.0.22 引入 hypergraph 后 CTE 执行路径变化的源码可见痕迹。

**语法能力清单（8.0.39 逐一核实）**：

| 能力 | 支持 | 说明 |
|---|---|---|
| 普通 CTE / 多 CTE（逗号分隔） | ✅ | `with_list: with_list ',' common_table_expr` |
| `WITH RECURSIVE` | ✅ | 与普通 WITH 同一条 `with_clause` 规则，`RECURSIVE_SYM` 只是可选关键字 |
| `WITH RECURSIVE` 但无递归成员 | ✅ | 合法（测试注释明说 "ok to have the RECURSIVE word without any recursive member"） |
| WITH 前置 / 内嵌（子查询里再 WITH） | ✅ | 内嵌时前向引用仍报 `ER_NO_SUCH_TABLE` |
| CTE 引用 CTE | ✅ | 前向引用禁止（`ER_NO_SUCH_TABLE`） |
| 列名覆盖 `WITH cte(col1,col2)` | ✅ | 空列名表 `()` 报 parse error；列数不符报 `ER_VIEW_WRONG_LIST` |
| 同一 CTE 多次引用 `FROM cte a, cte b` | ✅ | 解析期 `reparse_common_table_expr` 克隆子树，执行期共享物化、各自游标 |
| `SEARCH DEPTH FIRST / BREADTH FIRST` | ❌ | `lex.h` 无 SEARCH/DEPTH/BREADTH 关键字 |
| `CYCLE` 子句 | ❌ | `lex.h` 无 CYCLE 关键字；深度优先要 `STRAIGHT_JOIN`+`ORDER BY PATH` 手写 |
| 互递归（mutual recursion） | ❌ | 报 `ER_NO_SUCH_TABLE`，"cycles must have one node only" |
| `WITH ... UPDATE/DELETE` | ✅ | `update_stmt`/`delete_stmt` 以 `opt_with_clause` 开头 |
| `WITH ... INSERT/REPLACE` | ❌ | `insert_stmt`/`replace_stmt` 直接以 `INSERT_SYM`/`REPLACE_SYM` 开头，无 `opt_with_clause` |

**"多次引用"限制的真正边界**：非递归 CTE 可随意多次引用（克隆 + 共享物化）；但**递归块内**递归表只能被引用**一次**、且不能出现在子查询里（`ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE`，落点 `Table_ref::set_recursive_reference()`）。这与 SQL:1999 的约束一致。

**演进遗留的 TODO（源码自述未完成方向）**：

1. 共享物化的 lateral 失效检查目前只判断"是否写入了新行"，`composite_iterators.cc` 注释：*"TODO: It would be better ... to check the actual column values instead of just whether we've seen any new rows"*——暗示未来可能朝**值级环检测**演进。（LATERAL 机制的系统讲解见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md#六lateral让派生表看见同层兄弟表)）
2. `MAXRECURSION` 类 hint 未实现：*"a MAX_RECURSION hint (if we featured one) may interrupt"*——"if we featured one" 明示 MySQL 没有 SQL Server 那种 `OPTION (MAXRECURSION n)`，只有 `cte_max_recursion_depth`（可经 `SET_VAR` hint 改）。
3. EXPLAIN 打印位置：*"TODO(sgunders): Consider printing CTE query plans on the top level ... instead?"*

**★ `cte_max_recursion_depth` 变量定义**（`sql/sys_vars.cc:1012`）：

```cpp
static Sys_var_uint Sys_cte_max_recursion_depth(
    "cte_max_recursion_depth",
    "Abort a recursive common table expression "
    "if it does more than this number of iterations.",
    HINT_UPDATEABLE SESSION_VAR(cte_max_recursion_depth),
    CMD_LINE(REQUIRED_ARG), VALID_RANGE(0, UINT_MAX32), DEFAULT(1000),
    BLOCK_SIZE(1));
```
> ★ 默认 **1000**，范围 `0 .. UINT_MAX32`，**SESSION 级且 `HINT_UPDATEABLE`**（支持 `/*+ SET_VAR(cte_max_recursion_depth=...) */`）。成员声明在 `sql/system_variables.h:240`。

**★ `WITH RECURSIVE` 语法产生式**（`sql/sql_yacc.yy:12413`，完整）：

```yacc
with_clause:
          WITH with_list
          {
            $$= NEW_PTN PT_with_clause($2, false);
          }
        | WITH RECURSIVE_SYM with_list
          {
            $$= NEW_PTN PT_with_clause($3, true);
          }
        ;

with_list:
          with_list ',' common_table_expr
          {
            if ($1->push_back($3)) MYSQL_YYABORT;
          }
        | common_table_expr
          {
            $$= NEW_PTN PT_with_list(YYTHD->mem_root);
            if ($$ == NULL || $$->push_back($1)) MYSQL_YYABORT;
          }
        ;

common_table_expr:
          ident opt_derived_column_list AS table_subquery
          {
            LEX_STRING subq_text;
            subq_text.length= @4.cpp.length();
            subq_text.str= YYTHD->strmake(@4.cpp.start, subq_text.length);
            if (subq_text.str == NULL) MYSQL_YYABORT;
            uint subq_text_offset= @4.cpp.start - YYLIP->get_cpp_buf();
            $$= NEW_PTN PT_common_table_expr($1, subq_text, subq_text_offset,
                                             $4, &$2, YYTHD->mem_root);
            if ($$ == NULL) MYSQL_YYABORT;
          }
        ;
```
> ★ `subq_text` + `subq_text_offset` 保存**原始文本片段**——正是 `make_subquery_node()` 在第二次引用时重新解析的输入。
> ★ 重名校验在 `PT_with_list::push_back()`（`parse_tree_nodes.cc:1916`）→ `ER_NONUNIQ_TABLE`。

**★ CTE 相关错误码全集**（`share/messages_to_clients.txt`，⚠️ 真名在 `share/` 不是 `sql/share/`）：

| 错误码 | 消息原文 | 位置 | 触发点 |
|---|---|---|---|
| `ER_CTE_RECURSIVE_REQUIRES_UNION` | `Recursive Common Table Expression '%s' should contain a UNION` | 8156 | `sql_derived.cc:328` |
| `ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST` | `Recursive Common Table Expression '%s' should have one or more non-recursive query blocks followed by one or more recursive ones` | 8159 | `sql_derived.cc:415`/`423` |
| `ER_CTE_RECURSIVE_FORBIDS_AGGREGATION` | `...can contain neither aggregation nor window functions in recursive query block` | 8162 | `sql_union.cc:826` |
| `ER_CTE_RECURSIVE_FORBIDDEN_JOIN_ORDER` | `...the recursive table must neither be in the right argument of a LEFT JOIN, nor be forced to be non-first with join order hints` | 8165 | `sql_optimizer.cc:5331`、`join_optimizer.cc:3064` |
| `ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE` | `...the recursive table must be referenced only once, and not in any subquery` | 8168 | `sql_parse.cc:5884`/`5908` |
| `ER_CTE_MAX_RECURSION_DEPTH` | `Recursive query aborted after %u iterations. Try increasing @@cte_max_recursion_depth to a larger value.` | 8346 | `basic_row_iterators.cc:404` |
| `ER_CTE_RECURSIVE_NOT_UNION` | `Recursive table reference in EXCEPT or INTERSECT operand is not allowed.` | 9810 | `sql_derived.cc:367` |
| `ER_SWITCH_TMP_ENGINE` | `'%s' requires @@internal_tmp_disk_storage_engine=InnoDB` | 8171 | CTE 临时表不支持 MyISAM |

**★ 相关系统变量（溢出/临时表）**：

| 变量 | 文件:行 | 默认值 | 注释原文 |
|---|---|---|---|
| `tmp_table_size` | `sql/sys_vars.cc:5297` | **16 MiB** | `"If an internal in-memory temporary table in the MEMORY or TempTable storage engine exceeds this size, MySQL will automatically convert it to an on-disk table"` |
| `max_heap_table_size` | `sql/sys_vars.cc:3033` | **16 MiB** | `"Don't allow creation of heap tables bigger than this"` |
| `internal_tmp_mem_storage_engine` | `sql/sys_vars.cc:5352` | **`TMP_TABLE_TEMPTABLE`** | `"The default storage engine for in-memory internal temporary tables."` |
| `temptable_max_ram` | `sql/sys_vars.cc:5360` | **1 GiB** | `"Maximum amount of memory ... the TempTable storage engine is allowed to allocate ... before starting to store data on disk."` |

---

## 一、WITH 的解析与存储

### 1.1 WITH 挂到 Query_expression

```cpp
bool PT_with_clause::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  pc->select->master_query_expression()->m_with_clause = this;  // 挂到所属 QE
  return false;
}
```

WITH 是 `<query expression>` 的前缀，所以存到 `Query_expression::m_with_clause`（不是 query block）。

### 1.2 Common_table_expr：三个数组是理解执行的关键

```cpp
class Common_table_expr {
  Mem_root_array<Table_ref *> references;  // 定义之外的非递归引用
  bool recursive;                          // 是否递归 CTE
  Mem_root_array<Table_ref *> tmp_tables;  // 共享物化的所有 Table_ref
};
```

- `references`：CTE 定义**之外**的引用。
- `recursive`：标记为递归。
- `tmp_tables`：共享物化时指向同一临时表的所有 Table_ref（首个持有真正 create 出来的 TABLE）。

### 1.3 名字查找：find_common_table_expr

从当前 query block 沿 `master_query_expression()->m_with_clause` 逐层向外查找（支持 CTE 作用域嵌套）。找到后：

- **非递归引用**：用 CTE 定义构造 `PT_subquery`，标 `node->m_is_derived_table = true`——**复用派生表机制**。
- **递归引用**：塞入 `(select 0)` 的 dummy 定义占位（`reparse_common_table_expr`），先蒙混过 name resolution。

`make_subquery_node` 保证同一 CTE 被多次引用时各自有独立子树（`references.size() >= 2` 时重新 parse 一份）。

**★ `PT_with_clause::contextualize()` 只有三行**（`sql/parse_tree_nodes.cc:1945`）：

```cpp
bool PT_with_clause::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true; /* purecov: inspected */
  // WITH complements a query expression (a unit).
  pc->select->master_query_expression()->m_with_clause = this;
  return false;
}
```
> ★ WITH **不做任何 CTE 定义的解析/分派**——只是把自己挂到 `Query_expression::m_with_clause` 上。真正的逐个 CTE 展开是**惰性的**：名字被引用时才由 `find_common_table_expr` 触发。

**★ `PT_common_table_expr` 与 `PT_with_clause` 的类定义**（`sql/parse_tree_nodes.h:245` / `:336`）：

```cpp
class PT_common_table_expr : public Parse_tree_node {
 public:
  bool make_subquery_node(THD *thd, PT_subquery **node);
  /**
    @param tl  Table reference to match
    @param in_self  If this is a recursive reference
    @param[out]  found Is set to true/false if matches or not
    @returns true if error
  */
  bool match_table_ref(Table_ref *tl, bool in_self, bool *found);
  bool is(const Common_table_expr *other) const { return other == &m_postparse; }
 private:
  LEX_STRING m_name;
  /// Raw text of query expression (including parentheses)
  const LEX_STRING m_subq_text;
  uint m_subq_text_offset;
  PT_subquery *const m_subq_node;
  const Create_col_name_list m_column_names;
  /**
    A Table_ref representing a CTE needs access to the WITH list
    element it derives from. However, in order to:
    - limit the members which Table_ref can access
    - avoid including this header file everywhere Table_ref needs to
    access these members, these members are relocated into a separate inferior
    object whose declaration is in table.h, like that of Table_ref. It's
    the "postparse" part. Table_ref accesses this inferior object only.
  */
  Common_table_expr m_postparse;
};
```
> ★★ 这段注释就是"**为什么 `Common_table_expr` 定义在 `table.h` 而不是 `sql_lex.h`**"的答案——为了让 `Table_ref` 不必 include `parse_tree_nodes.h`。
```cpp
class PT_with_clause : public Parse_tree_node {
 public:
  /**
    Call this to record in the WITH clause that we are contextualizing the
    CTE definition inserted in table reference 'tl'.
  */
  const Table_ref *enter_parsing_definition(Table_ref *tl) {
    auto old = m_most_inner_in_parsing;
    m_most_inner_in_parsing = tl;
    return old;
  }
  void leave_parsing_definition(const Table_ref *old) {
    m_most_inner_in_parsing = old;
  }
 private:
  const PT_with_list *const m_list;
  /// True if the user has specified the RECURSIVE keyword.
  const bool m_recursive;
  /**
    The innermost CTE reference which we're parsing at the
    moment. Used to detect forward references, loops and recursiveness.
  */
  const Table_ref *m_most_inner_in_parsing;
};
```
> ★ `m_most_inner_in_parsing` 是一个**栈式的保存/恢复指针**（`enter_parsing_definition` 返回旧值供 `leave_parsing_definition` 恢复）——这就是前向引用/自引用/环检测的**唯一机制**。

**★ `match_table_ref()`：递归标记与"只能引用一次"**（`sql_parse.cc:5894` + `sql/table.cc:7289`）：

```cpp
  if (tl->table_name_length == m_name.length &&
      !memcmp(tl->table_name, m_name.str, m_name.length)) {
    *found = true;
    if (in_self) {
      m_postparse.recursive = true;
      if (tl->set_recursive_reference()) {     /* ★ 返回 true = 该块已有递归引用 */
        my_error(ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE, MYF(0), name().str);
        return true;
      }
    } else {
      if (m_postparse.references.push_back(tl)) return true;
      if (m_column_names.size()) tl->set_derived_column_names(&m_column_names);
    }
    tl->set_common_table_expr(&m_postparse);
  }
```
```cpp
bool Table_ref::set_recursive_reference() {
  if (query_block->recursive_reference != nullptr) return true;
  query_block->recursive_reference = this;
  m_is_recursive_reference = true;
  return false;
}
```
> ⚠️ **真名核查**：`TABLE_LIST` 在 8.0.39 已统一改名 **`Table_ref`**；**不存在 `Table_ref::is_cte`**，用 `common_table_expr()` 返回非空判断。

### 1.4 递归/前向引用判定：PT_with_clause::lookup

`m_most_inner_in_parsing` 记录当前正在 contextualize 的 CTE。在 WITH 列表从左到右扫描，一旦越过"正在解析的那个 CTE"（right_bound），再往后就是前向引用——被禁止，除非声明 `RECURSIVE` 且匹配的是它自己（自引用 = 递归引用）。

**★ 完整实现**（`sql/sql_parse.cc:5813`，⚠️ 不在 `parse_tree_nodes.cc`）：

```cpp
bool PT_with_clause::lookup(Table_ref *tl, PT_common_table_expr **found) {
  *found = nullptr;
  assert(tl->query_block != nullptr);
  /*
    If right_bound!=NULL, it means we are currently parsing the
    definition of CTE 'right_bound' and this definition contains 'tl'.
  */
  const Common_table_expr *right_bound =
      m_most_inner_in_parsing ? m_most_inner_in_parsing->common_table_expr()
                              : nullptr;
  bool in_self = false;
  for (auto el : m_list->elements()) {
    // Search for a CTE named like 'tl', in this list, from left to right.
    if (el->is(right_bound)) {
      /*
        We meet right_bound.
        If not RECURSIVE:
        we must stop the search in this WITH clause;
        indeed right_bound must not reference itself or any CTE defined after it
        in the WITH list (forward references are forbidden, preventing any
        cycle).
        If RECURSIVE:
        If right_bound matches 'tl', it is a recursive reference.
      */
      if (!m_recursive) break;  // Prevent forward reference.
      in_self = true;           // Accept a recursive reference.
    }
    bool match;
    if (el->match_table_ref(tl, in_self, &match)) return true;
    if (!match) {
      if (in_self) break;       // ★ RECURSIVE 下也不能前向引用
      continue;
    }
    if (in_self && tl->query_block->outer_query_block() !=
                       m_most_inner_in_parsing->query_block) {
      /*
        SQL2011 says a recursive CTE cannot contain a subquery
        referencing the CTE, except if this subquery is a derived table ...
        However, we don't allow this, as:
        - it simplifies detection and substitution correct recursive
        references (they're all on "level 0" of the UNION)
        - it's not limiting the user so much ...
        - Oracle bans it: ... ORA-32042: recursive WITH clause must reference
          itself directly in one of the UNION ALL branches.
      */
      my_error(ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE, MYF(0),
               el->name().str);
      return true;
    }
    *found = el;
    break;
  }
  return false;
}
```
> ★★ 两条 `break` 的分工：① `if (!m_recursive) break;`——非 RECURSIVE 时**既不能自引用，也不能引用右边的 CTE**（前向引用禁止 ⇒ 结构性防环）；② `if (in_self) break;`——RECURSIVE 下**只能引用"自己"或"自己左边已定义的 CTE"**，仍不允许前向引用。
> ★ 递归引用必须与 UNION 处在**同一层**（level 0）——`outer_query_block() != m_most_inner_in_parsing->query_block` 即报错。
> ⚠️ 真名核查：`find_common_table_expr()` 在 **`sql/sql_parse.cc:5735`**（不是 `sql_resolver.cc`）；`lookup()` 同在 `sql_parse.cc:5813`，只有 `contextualize()` 在 `parse_tree_nodes.cc:1945`。

---

## 二、非递归 CTE：就是 derived table

**核心结论**：非递归 CTE 引用在解析层被做成了 derived table，完整复用 `merge_derived`（合并）或 `materialize_derived`（物化）。

- **merge 判定**：`Query_expression::is_mergeable()`——无 set-op、无 GROUP BY/HAVING/DISTINCT/LIMIT/窗口 才可 merge
- **与普通 derived table 的差别**：①所有引用共享一个 `Common_table_expr`；②多次引用时物化一次、共享

### merge 的完整决策链（优先级从高到低）

源码注释把优先级写得很清楚：

```cpp
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
```

**逐段解读**：

1. **`ALGORITHM=TEMPTABLE` 或结构不可 merge** ⇒ 直接物化（显式指令 + 结构限制最高优先级）
2. **`ALGORITHM=UNDEFINED`**（默认）时看两条：
   - `merge_heuristic`：启发式（视图或允许 merge 的 derived，且查询表达式本身适合合并）
   - 再经 **hint（`MERGE`/`NO_MERGE`）** 与 **`optimizer_switch=derived_merge`** 共同决定
3. 另有约束：`STRAIGHT_JOIN` 时不能 merge 含 semi-join nest 的查询块（会破坏连接顺序语义）

### 物化的执行入口

- `resolve_derived()`：prepare 期解析
- `create_materialized_table()` + `materialize_derived()`：真正建表并填充

**★ 真正的执行入口是 `MaterializeIterator::Init()`**（`composite_iterators.cc:824`，⚠️ 8.0.39 **没有** `Query_expression::materialize()` / `execute_materialization()`）：

```cpp
template <typename Profiler>
bool MaterializeIterator<Profiler>::Init() {
  if (!table()->materialized && table()->pos_in_table_list != nullptr &&
      table()->pos_in_table_list->is_view_or_derived()) {
    // Create the table if it's the very first time.
    if (table()->pos_in_table_list->create_materialized_table(thd())) {
      return true;
    }
  }
  /* ... use_shared_cte_materialization 判定（见 7.3） ... */
  if (table()->materialized) { /* ... 只重扫，或按 invalidator 决定是否 rematerialize ... */ }
  table()->set_not_started();
  if (!table()->is_created()) {
    if (instantiate_tmp_table(thd(), table())) return true;
    empty_record(table());
  } else {
    table()->file->ha_index_or_rnd_end();
    table()->file->ha_delete_all_rows();     /* ★ 重跑先清空 */
  }
  if (m_query_expression != nullptr)
    if (m_query_expression->clear_correlated_query_blocks()) return true;
  if (m_cte != nullptr) {
    if (m_cte->clear_all_references()) return true;    /* ★ lateral CTE 兜底 */
  }
  ...
  ha_rows stored_rows = 0;

  if (m_query_expression != nullptr && m_query_expression->is_recursive()) {
    if (MaterializeRecursive()) return true;           /* ★★ 递归分派点 */
  } else {
    for (const materialize_iterator::QueryBlock &query_block :
         m_query_blocks_to_materialize) {
      if (MaterializeQueryBlock(query_block, &stored_rows)) return true;
      ...
    }
  }
  end_unique_index.rollback();
  table()->materialized = true;
  ...
  m_profiler.IncrementNumRows(stored_rows);
  return m_table_iterator->Init();
}
```
> ★★ 上行链路：`Query_expression::execute()` → `ExecuteIteratorQuery()`（`sql_union.cc:1672/1813`）→ 根迭代器 `Init()`。
> ★ 递归分派只看 `m_query_expression->is_recursive()`（即 `first_recursive != nullptr`）。
> ★ `stored_rows` 是 **`Init()` 的栈上局部变量**，而 `FollowTailIterator::m_stored_rows` 正是指向它的裸指针。
> ⚠️ 真名核查：`MaterializeIterator` 是**模板类**，定义与实现**全在 `composite_iterators.cc:620-1470`**，`composite_iterators.h` 里**没有**它的定义（只有 `FollowTailIterator` 的前向声明）。

**★ 递归 reader 的回填**（`access_path.cc:1000`，★ `MaterializeIterator` 自己不 new `FollowTailIterator`）：

```cpp
          if (to.is_recursive_reference) {
            // Find the recursive reference to ourselves; there should be
            // exactly one, as per the standard.
            RowIterator *recursive_reader = FindSingleIteratorOfType(
                from.subquery_path, AccessPath::FOLLOW_TAIL);
            if (recursive_reader == nullptr) {
              // The recursive reference was optimized away, e.g. due to an
              // impossible WHERE condition, so we're not a recursive
              // reference after all.
              to.is_recursive_reference = false;
            } else {
              to.recursive_reader =
                  down_cast<FollowTailIterator *>(recursive_reader);
            }
          }
```
> ★ 若递归引用被优化掉（如 `WHERE FALSE`），`is_recursive_reference` 回退为 false——一个容错细节。

---

## 三、物化表的自动建索引

物化的 derived/CTE 表**不是**一张只能全表扫描的笨表——MySQL 会根据外层对它的等值引用**自动建键**，让后续访问能走 ref。

`update_derived_keys` 的注释说明了用途：

```cpp
/**
  @brief
  Update derived table's list of possible keys

  @details
  This function creates/extends a list of possible keys for this derived
  table/view. For each table used by a value from the 'values' array the
  corresponding possible key is extended to include the 'field'.
  If there is no such possible key, then it is created. field's
  part_of_key bitmaps are updated accordingly.
  @see add_derived_key
*/
```

真正建键的 `add_derived_key`：

```cpp
static bool add_derived_key(THD *thd, List<Derived_key> &derived_key_list,
                            Field *field, table_map ref_by_tbl) {
  uint key = 0;
  Derived_key *entry = nullptr;
  List_iterator<Derived_key> ki(derived_key_list);

  /* Search for already existing possible key. */
  while ((entry = ki++)) {
    key++;
    if (ref_by_tbl) {
      /* Search for the entry for the specified table.*/
      if (entry->referenced_by & ref_by_tbl) break;
    } else {
      /*
        Search for the special entry that should contain fields referred
        from any table.
      */
      if (!entry->referenced_by) break;
    }
  }
  /* Add new possible key if nothing is found. */
  if (!entry) {
    key++;
    entry = new (thd->mem_root) Derived_key();
    if (!entry) return true;
    entry->referenced_by = ref_by_tbl;
    entry->used_fields.clear_all();
    if (derived_key_list.push_back(entry, thd->mem_root)) return true;
  }
  /* Don't create keys longer than REF access can use. */
  if (entry->used_fields.bits_set() < MAX_REF_PARTS) {
    field->part_of_key.set_bit(key - 1);
    field->set_flag(PART_KEY_FLAG);
    entry->used_fields.set_bit(field->field_index());
    entry->key_part_count++;
  }
  return false;
}
```

**机制要点**：

- **按"被哪个表引用"分组建键**：`referenced_by` 是表位图，对每个引用它的表单独维护一个键（`ref_by_tbl` 为 0 时是"任意表都能用"的通用键）
- **键不能无限长**：`used_fields.bits_set() < MAX_REF_PARTS`——超过 ref 访问能用的键部分数就不再加列，因为建了也用不上
- **三类字段跳过**（`update_derived_keys` 开头）：CREATE VIEW 的上下文分析、BLOB 字段、零长字段

**为什么重要**：如果没有这个机制，物化后的 CTE 每次被外层访问都要全表扫描，物化路径会比 merge 慢得多。有了它，`WHERE cte.col = t.col` 这类引用可以在物化表上走 ref。

> 这也解释了为什么"能 merge 就 merge"虽然是默认，但物化路径并没有被放弃——物化 + 自动建索引在某些情况下（尤其是多次引用共享时）反而更优。

---

## 四、递归 CTE 的核心算法

### 4.1 结构校验

遍历 UNION 的所有 query block，校验 SQL 标准约束：

- 必须是 UNION（`ER_CTE_RECURSIVE_REQUIRES_UNION`）
- 递归块必须在 UNION 下、禁止右嵌套 UNION
- 递归成员不允许 ORDER BY/LIMIT/DISTINCT
- 非递归块（anchor）必须排在递归块之前
- 至少要有 anchor

最后 `derived->first_recursive = last_non_recursive->next_query_block()` 指向第一个递归块。

**★ 主校验函数完整原文**（`sql/sql_derived.cc:324`，★ 逐条对应）：

```cpp
  if (m_common_table_expr && m_common_table_expr->recursive &&
      !derived->is_recursive()) {
    // Ensure it's UNION.
    if (!derived->is_union()) {
      my_error(ER_CTE_RECURSIVE_REQUIRES_UNION, MYF(0), alias);
      return true;
    }
    if (derived->global_parameters()->is_ordered()) {
      /*
        ORDER BY applied to the UNION causes the use of the union tmp
        table. The fake_query_block would want to sort that table, which isn't
        going to work as the table is incomplete when fake_query_block first
        reads it. Workaround: put ORDER BY in the top query.
        Another reason: allowing
        ORDER BY <condition using fulltext> would make the UNION tmp table be
        of MyISAM engine which recursive CTEs don't support.
        LIMIT is allowed and will stop the row generation after N rows.
        However, without ORDER BY the CTE's content is ordered in an
        unpredictable way, so LIMIT theoretically returns an unpredictable
        subset of rows. Users are on their own.
      */
      my_error(ER_NOT_SUPPORTED_YET, MYF(0),
               "ORDER BY over UNION in recursive Common Table Expression");
      return true;
    }
    /*
      Should be:
      SELECT1 UNION [DISTINCT | ALL] ... SELECTN
      where SELECT1 is non-recursive, and all non-recursive SELECTs are before
      all recursive SELECTs.
      In SQL standard terms, the CTE must be "expandable" except that we allow
      it to have more than one recursive SELECT.
    */
    bool previous_is_recursive = false;
    Query_block *last_non_recursive = nullptr;
    for (Query_block *sl = derived->first_query_block(); sl;
         sl = sl->next_query_block()) {
      if (sl->is_recursive()) {
        if (sl->parent()->term_type() != QT_UNION) {
          my_error(ER_CTE_RECURSIVE_NOT_UNION, MYF(0)); return true;
        } else if (sl->parent()->parent() != nullptr) {
          my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                   "right nested recursive query blocks, in Common Table Expression");
          return true;
        }
        if (sl->is_ordered() || sl->has_limit() || sl->is_distinct()) {
          my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                   "ORDER BY / LIMIT / SELECT DISTINCT"
                   " in recursive query block of Common Table Expression");
          return true;
        }
        if (sl == derived->last_distinct() && sl->next_query_block()) {
          /*
            Consider anchor UNION ALL rec1 UNION DISTINCT rec2 UNION ALL rec3:
            after execution of rec2 we must turn off the duplicate-checking
            index; it will thus not contain the keys of rows of rec3, so it
            becomes permanently unusable. ...
          */
          my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                   "recursive query blocks with UNION DISTINCT then UNION ALL, "
                   "in recursive Common Table Expression");
          return true;
        }
      } else {
        if (previous_is_recursive) {
          my_error(ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST, MYF(0), alias);
          return true;
        }
        last_non_recursive = sl;
      }
      previous_is_recursive = sl->is_recursive();
    }
    if (last_non_recursive == nullptr) {
      my_error(ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST, MYF(0), alias);
      return true;
    }
    derived->first_recursive = last_non_recursive->next_query_block();
    assert(derived->is_recursive());
  }
```

**★ 另两处校验**：

```cpp
/* sql/sql_union.cc:802 —— 聚合/窗口函数禁用 */
    if (sl->recursive_reference &&
        (sl->is_grouped() || sl->m_windows.elements > 0)) {
      // Per SQL2011.
      my_error(ER_CTE_RECURSIVE_FORBIDS_AGGREGATION, MYF(0),
               derived_table->alias);
      return true;
    }
```
```cpp
/* sql/sql_optimizer.cc:5318 —— LEFT JOIN 右操作数 / join order hint 禁用 */
  if (query_block->outer_join || query_block->is_recursive()) {
    if (propagate_dependencies()) {
      /*
        Catch illegal join order.
        SQL2011 forbids:
        WITH RECURSIVE rec AS (... UNION ALL SELECT ... FROM tbl LEFT JOIN rec ON...)
        MySQL also forbids the same query with STRAIGHT_JOIN instead of LEFT
        JOIN, because the algorithm of with-recursive imposes that "rec" be
        first in plan, i.e. "tbl" depends on "rec", but STRAIGHT_JOIN imposes
        the opposite dependency.
      */
      assert(query_block->is_recursive());
      my_error(ER_CTE_RECURSIVE_FORBIDDEN_JOIN_ORDER, MYF(0),
               query_block->recursive_reference->alias);
      return true;
    }
    init_key_dependencies();
  }
```

**★ 完整校验清单（14 条）**：

| # | 校验 | 错误码 | 位置 |
|---|---|---|---|
| 1 | 必须是 UNION | `ER_CTE_RECURSIVE_REQUIRES_UNION` | `sql_derived.cc:328` |
| 2 | UNION 上不能有 ORDER BY | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:348` |
| 3 | 不能在 EXCEPT/INTERSECT 操作数里 | `ER_CTE_RECURSIVE_NOT_UNION` | `sql_derived.cc:367` |
| 4 | 不能右嵌套 UNION | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:376` |
| 5 | 递归块不能有 ORDER BY/LIMIT/SELECT DISTINCT | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:390` |
| 6 | 不能 UNION DISTINCT 后接 UNION ALL | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:407` |
| 7 | anchor 必须全在递归块之前 | `ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST` | `sql_derived.cc:415` |
| 8 | 至少一个 anchor | 同上 | `sql_derived.cc:423` |
| 9 | 递归块不能有聚合/窗口 | `ER_CTE_RECURSIVE_FORBIDS_AGGREGATION` | `sql_union.cc:826` |
| 10 | 不能在 LEFT JOIN 右操作数 / 被 hint 强制非首位 | `ER_CTE_RECURSIVE_FORBIDDEN_JOIN_ORDER` | `sql_optimizer.cc:5331`、`join_optimizer.cc:3064` |
| 11 | 递归表只引用一次且不在子查询 | `ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE` | `sql_parse.cc:5884`/`5908` |
| 12 | WITH 列表内同名 | `ER_NONUNIQ_TABLE` | `parse_tree_nodes.cc:1921` |
| 13 | 互递归 | **无专门码**，报 `ER_NO_SUCH_TABLE` | `with_recursive.inc:6` |
| 14 | 深度超限 | `ER_CTE_MAX_RECURSION_DEPTH` | `basic_row_iterators.cc:404` |

> ★ 一个易被误读的点：**UNION 级别的 `LIMIT` 是允许的**（注释："LIMIT is allowed and will stop the row generation after N rows"），这正是 `limit_rows` 下推的用途；被拒的是**递归块内**的 ORDER BY/LIMIT/DISTINCT。

### 4.2 prepare 阶段：建临时表 + 递归引用替换

```cpp
if (sl == first_recursive) {
  derived_table->setup_materialized_derived_tmp_table(thd);   // anchor 类型确定后建表
}
if (sl->recursive_reference)
  derived_table->common_table_expr()->substitute_recursive_reference(thd, sl);
```

`substitute_recursive_reference`：递归引用此前是 `(select 0)` 的 dummy derived table，这里克隆一个指向物化临时表的 TABLE 句柄，摘掉 dummy 的 query expression，变成"直接读临时表"的引用。

**★ 完整实现**（`sql/sql_derived.cc:231`，⚠️ 真名不是 `replace_recursive_references`）：

```cpp
bool Common_table_expr::substitute_recursive_reference(THD *thd,
                                                       Query_block *sl) {
  Table_ref *tl = sl->recursive_reference;
  assert(tl != nullptr && tl->table == nullptr);
  TABLE *t = clone_tmp_table(thd, tl);          /* ① 克隆共享临时表的 TABLE */
  if (t == nullptr) return true;
  // Eliminate the dummy unit:
  tl->derived_query_expression()->exclude_tree();  /* ② 摘掉 dummy 整棵树 */
  tl->set_derived_query_expression(nullptr);
  tl->set_privileges(SELECT_ACL);                  /* ③ 权限降为 SELECT */
  return false;
}
```
```cpp
/* sql/sql_union.cc:741 —— ★ 调用顺序极其关键 */
    if (sl == first_recursive) {
      // create_result_table() depends on current_query_block()
      save_query_block.restore();
      /*
        All next query blocks will read the temporary table, which we must
        thus create now:
      */
      if (derived_table->setup_materialized_derived_tmp_table(thd))
        return true;                    /* ★ 先建表（列类型只由前面的 anchor 决定） */
      thd->lex->set_current_query_block(sl);
    }
    if (sl->recursive_reference)       // Make tmp table known to query block:
      derived_table->common_table_expr()->substitute_recursive_reference(thd, sl);
                                       /* ★ 再替换递归引用 */
    if (sl->prepare(thd, insert_field_list)) return true;   /* ★ 最后才 prepare */
```

### 4.3 运行期：seed + 收敛循环

`MaterializeIterator` 检测到 `is_recursive()` 时走 `MaterializeRecursive()`，算法注释写明：

```
1. 物化所有非递归 query block（seed/anchor），一次。
2. 物化所有递归 query block。
3. 重复 2 直到没有 block 再写入任何行（收敛）。
   每次物化只看到上一轮新增的行（FollowTailIterator）。
```

核心循环：

```cpp
ha_rows stored_rows = 0;
// 把总行数指针交给每个递归 reader（FollowTailIterator）
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (qb.is_recursive_reference)
    qb.recursive_reader->set_stored_rows_pointer(&stored_rows);

// 第一步：物化 seed
for (const QueryBlock &qb : m_query_blocks_to_materialize)
  if (!qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);

// 第二步+第三步：反复物化递归块直到收敛
ha_rows last_stored_rows;
do {
  last_stored_rows = stored_rows;
  for (const QueryBlock &qb : m_query_blocks_to_materialize)
    if (qb.is_recursive_reference) MaterializeQueryBlock(qb, &stored_rows);
} while (stored_rows > last_stored_rows);   // 不再新增即收敛
```

### 4.4 FollowTailIterator：读"上一轮新增的行"

```cpp
int FollowTailIterator::Read() {
  if (m_read_rows == *m_stored_rows) {
    return -1;   // 已读到当前总行数 → EOF；关键是不让底层 cursor 真正撞 EOF，
                 // 否则 MEMORY/InnoDB 的 scan 会永久卡在 EOF，看不到后续插入的新行
  }
  if (m_read_rows == m_end_of_current_iteration) {
    if (++m_recursive_iteration_count > thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;   // 超深度报错
    }
    m_end_of_current_iteration = *m_stored_rows;  // 本轮终点 = 当前总行数
  }
  int err = table()->file->ha_rnd_next(m_record);  // 顺序读下一行
  if (err) return HandleError(err);
  ++m_read_rows;
  return 0;
}
```

精髓：

- `m_stored_rows` 指向物化器的总行数计数器。
- 每个递归步开始时 `m_end_of_current_iteration = *m_stored_rows` 记下"本步之前已有多少行"。
- 递归引用只顺序扫描 `[上一轮终点, 当前总数)` 区间，即**上一轮新产生**的行。
- 扫描位置追上总行数时**返回 EOF 而不推进底层游标**（关键！），从而下一轮写入新行后还能继续读。

### 4.5 深度限制

`cte_max_recursion_depth` 默认 **1000**，SESSION 级。

```cpp
/* sql/sys_vars.cc:1012 */
static Sys_var_uint Sys_cte_max_recursion_depth(
    "cte_max_recursion_depth",
    "Abort a recursive common table expression "
    "if it does more than this number of iterations.",
    HINT_UPDATEABLE SESSION_VAR(cte_max_recursion_depth),
    CMD_LINE(REQUIRED_ARG), VALID_RANGE(0, UINT_MAX32), DEFAULT(1000),
    BLOCK_SIZE(1));
```
> ★ 范围 `0 .. UINT_MAX32`；**`HINT_UPDATEABLE`** ⇒ 可用 `/*+ SET_VAR(cte_max_recursion_depth=N) */`。

**★ 唯一的运行期检查点**（`basic_row_iterators.cc:399`，⚠️ 在 `FollowTailIterator::Read()` 不是 UNION 侧）：

```cpp
  if (m_read_rows == m_end_of_current_iteration) {
    // We have started a new iteration. Check to see if we have passed the
    // user-set limit.
    if (++m_recursive_iteration_count >
        thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;
    }
    m_end_of_current_iteration = *m_stored_rows;
```
> ★ 计数是"**轮数**"不是行数，且**先自增再比较**。因为"one run can encompass many iterations"，所以不能按 `Init()` 次数算。
> ★ 报错消息会带回实际迭代数：`"Recursive query aborted after %u iterations. Try increasing @@cte_max_recursion_depth to a larger value."`

---

## 五、共享物化与重扫

> **视角声明**：`clone_tmp_table()` 与 `DELAYED_OPEN` 的**机制权威**在 [`08_materialization.md`](08_materialization.md)「共享物化：clone_tmp_table() 与 DELAYED_OPEN」（物化是第一主语，含完整源码逐段）。本篇只从 **CTE 视角**保留"多引用只物化一次"与"读者同步"这两条使用侧结论，不重复那里的机制剖析。

### 5.1 多引用只物化一次：clone_tmp_table

同一 CTE 的第二个及后续引用不再 `create_tmp_table`，而是 `open_table_from_share` 基于同一个 `TABLE_SHARE` 克隆一个新的 TABLE 句柄——**共享底层数据、各自独立游标**。第一个是写者，克隆是读者。

`setup_materialized_derived_tmp_table`：`tmp_tables.size() > 0` 时复用，否则首次真正创建。

**★ `clone_tmp_table()` 完整实现**（`sql/sql_derived.cc:170`，⚠️ 是成员函数不是 `sql/table.cc` 的自由函数）：

```cpp
TABLE *Common_table_expr::clone_tmp_table(THD *thd, Table_ref *tl) {
  assert(tl->common_table_expr() == this);
#ifndef NDEBUG
  /*
    We're adding a clone; if another clone has been opened before, it was not
    aware of the new one, so perhaps the storage engine has not set up the
    necessary logic to share data among clones. Check that no clone is open:
  */
  Derived_refs_iterator it(tmp_tables[0]);
  while (TABLE *t = it.get_next()) assert(!t->is_created() && !t->materialized);
#endif
  TABLE *first = tmp_tables[0]->table;
  // Allocate clone on the memory root of the TABLE_SHARE.
  TABLE *t = static_cast<TABLE *>(first->s->mem_root.Alloc(sizeof(TABLE)));
  if (!t) return nullptr;
  if (open_table_from_share(thd, first->s, tl->alias,
                            /*
                              Pass db_stat == 0 to delay opening of table in SE,
                              as table is not instantiated in SE yet.
                            */
                            0,
                            /* We need record[1] for this TABLE instance. */
                            EXTRA_RECORD |
                                /*
                                  Use DELAYED_OPEN to have its own record[0]
                                  (necessary because db_stat is 0).
                                  Otherwise it would be shared with 'first'
                                  and thus a write to tmp table would modify
                                  the row just read by readers.
                                */
                                DELAYED_OPEN,
                            0, t, false, nullptr))
    return nullptr;
  assert(t->s == first->s && t != first && t->file != first->file);
  t->s->increment_ref_count();
  t->s->tmp_handler_count++;
  // In case this clone is used to fill the materialized table:
  bitmap_set_all(t->write_set);
  t->reginfo.lock_type = TL_WRITE;
  t->copy_blobs = true;
  tl->table = t;
  t->pos_in_table_list = tl;
  // If initial CTE table has a hash key, set up a hash key for all clones too.
  if (first->hash_field) t->hash_field = t->file->field[0];
  t->hidden_field_count = first->hidden_field_count;
  t->set_not_started();
  if (tmp_tables.push_back(tl)) return nullptr;
  return t;
}
```
> ★★ **三个关键参数**（7.1 段）：`db_stat = 0`（延迟打开 SE）、`EXTRA_RECORD`（要 `record[1]`，用于 `check_unique_constraint` 的双缓冲）、`DELAYED_OPEN`（**必须有独立的 `record[0]`**）。
> ★★ **为什么不能共用同一个 `TABLE`**：注释原话——否则 `record[0]` 与写入者共享，"**写一行会改掉读者刚读到的那一行**"。这不是并发问题，而是**单线程下读写交错**的行缓冲冲突。
> ★ clone 共享 `TABLE_SHARE`（`t->s == first->s`），但各有独立 `handler`（`t->file != first->file`）→ 各有独立游标。

**★ 调用点**（`sql/sql_derived.cc:841`，★ optimizer trace 里会出现 `reusing_tmp_table`）：

```cpp
  if (m_common_table_expr && m_common_table_expr->tmp_tables.size() > 0) {
    trace_derived.add("reusing_tmp_table", true);
    table = m_common_table_expr->clone_tmp_table(thd, this);
    if (table == nullptr) return true;
    derived_result->table = table;
  }
```

**★ `rematerialize` 的真身**（`sql/join_optimizer/materialize_path_parameters.h:80`，⚠️ 不是 `Common_table_expr::m_rematerialize`）：

```cpp
  /**
     True if rematerializing on every Init() call (e.g., because we
     have a dependency on a value from outside the query block).
  */
  bool rematerialize;

  /**
     Used for when pushing LIMIT down to MaterializeIterator; this is
     more efficient than having a LimitOffsetIterator above the
     MaterializeIterator, since we can stop materializing when there are
     enough rows. (This is especially important for recursive CTEs.)
     Note that we cannot have a LimitOffsetIterator _below_ the
     MaterializeIterator, as that would count wrong if we have deduplication,
     and would not work at all for recursive CTEs.
     Set to HA_POS_ERROR for no limit.
  */
  ha_rows limit_rows;
```
```cpp
/* composite_iterators.cc:671 —— 成员 Doxygen */
  /// If we are materializing a CTE, points to it (otherwise nullptr).
  /// Used so that we see if some other iterator already materialized the table,
  /// avoiding duplicate work.
  Common_table_expr *m_cte;
  ...
  /// If true, we need to materialize anew for each Init() (because the contents
  /// of the table will depend on some outer non-constant value).
  const bool m_rematerialize;
```

### 5.2 rematerialize 参数

`MaterializeIterator::Init`：

- 非相关 CTE `rematerialize=false`：多次引用共享一次物化，之后只重扫。
- 相关（lateral/correlated）CTE 或依赖外层值的派生表 `rematerialize=true`：每次 `Init()` 清空重建。

`use_shared_cte_materialization` 判定：`m_cte != nullptr && !m_rematerialize` 且 `tmp_tables` 里已有 `materialized==true` 的表，则复用不再物化。

**★ `rematerialize` 的三种来源**（★ 不是 `Common_table_expr` 的成员）：

```cpp
/* sql/join_optimizer/materialize_path_parameters.h:40 —— 路径参数 */
struct MaterializePathParameters {
  ...
  /// If materializing a CTE, points to it (see m_cte), otherwise nullptr.
  Common_table_expr *cte;
  /// The query expression we are materializing.
  Query_expression *unit;
  /**
     True if rematerializing on every Init() call (e.g., because we
     have a dependency on a value from outside the query block).
  */
  bool rematerialize;
  ...
  ha_rows limit_rows;
};
```
```cpp
/* composite_iterators.cc:823 —— 构造时从 path_params 拷进 const 成员 */
      m_cte(path_params->cte),
      m_query_expression(path_params->unit),
      m_rematerialize(path_params->rematerialize),
```
```cpp
/* access_path.cc:1010 —— ★ create_iterators() 对 uncacheable 的 Query_block 恒设 true */
      // If we're invalidated, the join also needs to invalidate all of its
      // own materialization operations, but it will automatically do so by
      // virtue of the Query_block being marked as uncachable
      // (create_iterators() always sets rematerialize=true for such cases).
```
> ★ 与 invalidator generation 的交互（`composite_iterators.cc:863`）：即使 `m_rematerialize == false`，只要 lateral 表的 generation 变了也要重算：
> ```cpp
>       for (const Invalidator &invalidator : m_invalidators) {
>         if (invalidator.iterator->generation() !=
>             invalidator.generation_at_last_materialize) {
>           rematerialize = true;
>           break;
>         }
>       }
> ```
> ★ 源码 TODO 承认这里粗糙：`"TODO: It would be better, although probably much harder, to check the actual column values instead of just whether we've seen any new rows"`。

### 5.3 FOLLOW_TAIL 访问路径

```cpp
if (range_scan != nullptr)      path = range_scan;
else if (table_ref->is_recursive_reference()) path = NewFollowTailAccessPath(...);  // 递归引用
else                            path = NewTableScanAccessPath(...);
```

`AccessPath::FOLLOW_TAIL` → `FollowTailIterator`。

**★ `create_table_access_path` 的完整决策**（`sql/sql_executor.cc:4674`，★ 注意是**硬编码不是代价比较**）：

```cpp
AccessPath *create_table_access_path(THD *thd, TABLE *table,
                                     AccessPath *range_scan,
                                     Table_ref *table_ref, POSITION *position,
                                     bool count_examined_rows) {
  AccessPath *path;
  if (range_scan != nullptr) {
    range_scan->count_examined_rows = count_examined_rows;
    path = range_scan;
  } else if (table_ref != nullptr && table_ref->is_recursive_reference()) {
    path = NewFollowTailAccessPath(thd, table, count_examined_rows);
  } else {
    path = NewTableScanAccessPath(thd, table, count_examined_rows);
  }
  if (position != nullptr) {
    SetCostOnTableAccessPath(*thd->cost_model(), position,
                             /*is_after_filter=*/false, path);
  }
  return path;
}
```
> ★ 只要 `is_recursive_reference()` 为真就**无条件**选 `FOLLOW_TAIL`；但由于 `sql_optimizer.cc:5500` 已清空该表所有索引位图，`range_scan` 几乎不可能出现。

**★ `FOLLOW_TAIL` 是叶子路径，不带任何索引信息**（`sql/join_optimizer/access_path.h:209`，⚠️ 该文件在 `sql/join_optimizer/` 不在 `sql/`）：

```cpp
  enum Type : uint8_t {
    // Basic access paths (those with no children, at least nominally).
    TABLE_SCAN,
    INDEX_SCAN,
    ...
    MRR,
    FOLLOW_TAIL,
    INDEX_RANGE_SCAN,
    ...
  };
```
```cpp
    struct {
      TABLE *table;
    } follow_tail;      /* ★ 参数结构体只有 TABLE* 一个字段 */
```
```cpp
inline AccessPath *NewFollowTailAccessPath(THD *thd, TABLE *table,
                                           bool count_examined_rows) {
  AccessPath *path = new (thd->mem_root) AccessPath;
  path->type = AccessPath::FOLLOW_TAIL;
  path->count_examined_rows = count_examined_rows;
  path->follow_tail().table = table;
  return path;
}
```
```cpp
/* access_path.cc:498 —— 构造点 */
      case AccessPath::FOLLOW_TAIL: {
        const auto &param = path->follow_tail();
        iterator = NewIterator<FollowTailIterator>(
            thd, mem_root, param.table, path->num_output_rows(), examined_rows);
        break;
      }
```

**★ `sql/iterators/` 里与 CTE 相关的迭代器完整清单**：

| 迭代器 | 文件:行 | 与 CTE 的关系 |
|---|---|---|
| `FollowTailIterator` | `basic_row_iterators.h:439`（实现 `.cc:294-441`） | **递归 CTE 专用**：读自引用临时表的"尾巴" |
| `MaterializeIterator<Profiler>` | `composite_iterators.cc:620`（模板，`.h` 无定义） | CTE / 派生表 / 集合操作 / 排序 的物化入口 |
| `TableScanIterator` | `basic_row_iterators.h` | 非递归 CTE 引用、以及 `m_table_iterator` |
| `StreamingIterator` | `composite_iterators.cc:600` | 物化被"省略"时的替身 |
| `CacheInvalidatorIterator` | `composite_iterators.cc` | lateral / 相关 CTE 的 generation 失效检测 |
| ~~`CteScanIterator`~~ | — | ⚠️ **不存在** |
| `MaterializedTableFunctionIterator` | `composite_iterators.h:613` | 与 CTE 无关（JSON_TABLE） |

---

## 六、递归的限制、陷阱与性能特征

### 6.1 UNION 与 UNION ALL 的语义差异（最容易踩的坑）

递归 CTE 中 `UNION` 与 `UNION ALL` 的差别比普通查询更关键：

- **`UNION ALL`**：每轮新产生的行全部追加，**不做去重**——可能产生重复行，也可能因此**永不收敛**
- **`UNION`（DISTINCT）**：每轮产生的新行会与已有行**去重**——这既是防重复的保障，也是**提前收敛**的常见原因

因为 MySQL 不做环检测，`UNION`（去重）实际上是用户手上**唯一的"防环"手段**：只要某一轮产生的行全都已存在，去重后新增为 0，循环立即收敛。反之用 `UNION ALL` 且递归逻辑会重复产生相同行时，会一直跑到 `cte_max_recursion_depth` 才报错。

因此源码对 `UNION DISTINCT` 与 `UNION ALL` 的**混用顺序**有硬校验：

```cpp
        if (sl == derived->last_distinct() && sl->next_query_block()) {
          // 例：anchor UNION ALL rec1 UNION DISTINCT rec2 UNION ALL rec3
          my_error(ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST, ...
                   " UNION DISTINCT then UNION ALL, in recursive "
```

即：**一旦出现 `UNION DISTINCT`，它之后不允许再跟 `UNION ALL`**（去重后再追加会破坏语义）。

**★ 去重的两种实现路径**（`sql/sql_union.cc:142`，★ `Query_result_union::create_result_table`）：

```cpp
  if (unit != nullptr && op == nullptr) {
    if (unit->is_recursive()) {
      /*
        If the UNIQUE key specified for UNION DISTINCT were a primary key in
        InnoDB, rows would be returned by the scan in an order depending on
        their columns' values, not in insertion order.
      */
      tmp_table_param.can_use_pk_for_unique = false;      /* ★★ 递归 CTE 特判 */
    }
    ...
  }
  ...
  } else if (op->has_mixed_distinct_operators()) {
    // If we have mixed UNION DISTINCT / UNION ALL, we can't use an unique
    // index to deduplicate, as we need to be able to turn off deduplication
    // checking when we get to the UNION ALL part. ...
    tmp_table_param.force_hash_field_for_unique = true;
  }

  if (!(table = create_tmp_table(thd_arg, &tmp_table_param, visible_fields,
                                 nullptr, is_union_distinct, true, options,
                                 HA_POS_ERROR, table_alias)))
    return true;
  if (create_table) {
    table->file->ha_extra(HA_EXTRA_IGNORE_DUP_KEY);      /* ★ 重复行被静默忽略 */
    if (table->hash_field) table->file->ha_index_init(0, false);
  }
```
> ★★ 两条路径：① `is_union_distinct` → 建唯一索引 + `HA_EXTRA_IGNORE_DUP_KEY`（重复行在 `ha_write_row()` 时被静默丢弃）；② 混合 DISTINCT/ALL 或走 hash 字段 → `force_hash_field_for_unique` + `check_unique_constraint()` 手工比对。
> ★★ **`can_use_pk_for_unique = false` 是递归 CTE 的关键特判**：否则 InnoDB 的 PK 会让扫描按**列值顺序**而非**插入顺序**返回，`FollowTailIterator` 的"尾随"语义就崩了。

**★ 每块的去重开关**（`sql/sql_union.cc:1382`）：

```cpp
    bool activate_deduplication =
        idx <= m_last_distinct ||
        term_type() != QT_UNION; /* always for INTERSECT and EXCEPT */
    ...
    param.disable_deduplication_by_hash_field =
        (has_mixed_distinct_operators() && !activate_deduplication);
```
> ⚠️ **真名核查**：DISTINCT 覆盖范围标志是 `Query_term_set_op::m_last_distinct` / `m_first_distinct`（`sql/query_term.h:415`），**不是** `Query_expression::m_union_distinct`（8.0.39 无此成员）。

**★⚠️ 一处需要纠偏的常见说法**：文档/资料常说"`UNION ALL` 会永不收敛、`UNION` 才安全"。**源码不支持"UNION ALL 会导致不收敛"这一因果**——收敛判据是 `stored_rows > last_stored_rows`（全局行数净增长），与是否去重无关；真正被拒的是：
- 递归块**内**的 `ORDER BY`/`LIMIT`/`SELECT DISTINCT`（`ER_NOT_SUPPORTED_YET`，`sql_derived.cc:390`）
- `UNION DISTINCT` 后接 `UNION ALL`（`sql_derived.cc:407`）

去重**帮助**收敛（去重掉的行不计入 `stored_rows`，见 `composite_iterators.cc:987` 注释），而不是"防止不收敛"的唯一手段。真正导致跑飞的是**列类型被 anchor 截断**（见 6.2）。

### 6.2 列类型由 anchor 决定

递归 CTE 的列类型完全由**非递归块（anchor）**决定——递归块产出的值必须能装进 anchor 定义的类型。

这带来一个隐蔽问题：若 anchor 定义 `VARCHAR(10)` 而递归部分产出更长的值，**非严格模式下会截断**，而截断后的值可能永远无法收敛 ⇒ 无限递归。所以 MySQL 在递归物化时**强制严格模式**（`Strict_error_handler`），即使 session 不是严格模式也要报错。

**★ 三处源码证据**：

**① 递归块不参与类型推导**（`sql/sql_union.cc:797`，★ 最硬的证据）：

```cpp
      if (sl->recursive_reference) {
        /*
          Recursive query blocks don't determine output types of the result.
          The only thing to check could be if the recursive query block has a
          type which can't be cast to the output type of the result.
          But in MySQL, all types can be cast to each other (at least during
          resolution; an error may reported when trying to actually insert, for
          example an INT into a POINT). So no further compatibility check is
          needed here.
        */
      } else {
        auto it = sl->visible_fields().begin();
        auto tp = types.begin();
        for (; it != sl->visible_fields().end() && tp != types.end(); ++it, ++tp) {
          if ((down_cast<Item_type_holder *>(*tp))->join_types(thd, *it))
            return true;
        }
      }
```
> ★★ `if (sl->recursive_reference) { /* 空体 */ } else { join_types(...) }` —— 递归块的字段**根本不参与 `Item_type_holder::join_types()`**。注释原话："Recursive query blocks don't determine output types of the result."

**② 递归 CTE 所有列恒为 nullable**（`sql/sql_union.cc:785`）：

```cpp
        auto holder = new Item_type_holder(thd, item_tmp);
        if (!holder) return true;
        if (is_recursive()) {
          holder->set_nullable(true);  // Always nullable, per SQL standard.
```
> ★ 这是"由 anchor 定类型"的直接后果，也是 SQL 标准的要求。

**③ 截断 → 无限递归的权威解释**（`composite_iterators.cc:1006`）：

```cpp
  /*
    For RECURSIVE, beginners will forget that:
    - the CTE's column types are defined by the non-recursive member
    - which implies that recursive member's selected expressions are cast to
    the non-recursive member's type.
    That will cause silent truncation and possibly an infinite recursion due
    to a condition like: 'LENGTH(growing_col) < const', or,
    'growing_col < const',
    which is always satisfied due to truncation.
    ...
    Usually, truncation in SELECT triggers an error only in
    strict mode; but if we don't send an error we get a runaway query;
    and as WITH RECURSIVE is a new feature we don't have to carry the
    permissiveness of the past, so we send an error even if in non-strict
    mode.
  */
  Strict_error_handler strict_handler(
      Strict_error_handler::ENABLE_SET_SELECT_STRICT_ERROR_HANDLER);
```
> ★★ 注意这段注释纠正了"UNION ALL 导致不收敛"的说法：**真正导致跑飞的是类型截断**，不是 UNION 语义。

### 6.3 性能特征：为什么递归 CTE 容易慢

| 因素 | 影响 |
|---|---|
| **每轮全量扫描新增行** | 递归引用读的是 `[上轮终点, 当前总数)`，随结果集增大，每轮扫描量递增 |
| **物化表上的查找无索引**（除非触发自动建键且是等值引用） | 递归步内部的连接/过滤往往退化为扫描 |
| **递归深度 = 迭代轮数** | 每一轮都是一次完整的子查询执行 + 物化写入 |
| **可能降级到磁盘** | 结果集超过 `temptable_max_ram` 后转 InnoDB（见 [07_temptable.md](07_temptable.md)），递归写入代价陡增 |

**实践含义**：递归 CTE 适合"深度小、每轮增量小"的场景（如组织树展开、路径枚举）。深度大或每轮产生大量行时，复杂度会快速恶化。

**★ 源码第一手证据：递归引用被强制去索引 + 强制最左**（`sql/sql_optimizer.cc:5495`）：

```cpp
    tab->dependent = tl->dep_tables;  // Initialize table dependencies
    if (query_block->is_recursive()) {
      if (query_block->recursive_reference != tl)
        // Recursive reference must go first
        tab->dependent |= query_block->recursive_reference->map();
      else {
        // Recursive reference mustn't use any index
        table->covering_keys.clear_all();
        table->keys_in_use_for_group_by.clear_all();
        table->keys_in_use_for_order_by.clear_all();
      }
    }
```
> ★★ 递归表**强制最左**，且 `covering_keys` 被**全部清空**——每轮迭代对 CTE 只能**全表尾随扫描**，代价随累计行数线性增长。这是"递归 CTE 容易慢"最硬的证据。

**★ hypergraph 优化器同样的约束**（`sql/join_optimizer/join_optimizer.cc:415`）：

```cpp
  /// A map of tables that can never be on the right side of any join,
  /// ie., they have to be leftmost in the tree. This only affects recursive
  /// table references (ie., when WITH RECURSIVE is in use); they work by
  /// continuously tailing new records, which wouldn't work if we were to
  /// scan them multiple times or put them in a hash table. Naturally,
  /// there must be zero or one bit here; the common case is zero.
  NodeMap forced_leftmost_table = 0;
```
> ★ 递归表**既不能被多次扫描，也不能进 hash join 的 hash 表**。

**★ 另一条：递归查询块内不允许引入 tmp table**（`sql/sql_select.cc:4920`）：

```cpp
  /*
    tmp tables are a layer between the nested loop and the derived table's
    result, WITH RECURSIVE cannot work with them. This should not happen, as a
    recursive query cannot have clauses which use a tmp table (GROUP BY,
    etc).
  */
  assert(!query_block->is_recursive() || !tmp_tables);
```

**★ LIMIT 下推对递归 CTE 尤其重要**（`materialize_path_parameters.h:83`）：

```cpp
     Used for when pushing LIMIT down to MaterializeIterator; this is
     more efficient than having a LimitOffsetIterator above the
     MaterializeIterator, since we can stop materializing when there are
     enough rows. (This is especially important for recursive CTEs.)
     Note that we cannot have a LimitOffsetIterator _below_ the
     MaterializeIterator, as that would count wrong if we have deduplication,
     and would not work at all for recursive CTEs.
```

### 6.4 本版能力边界

- **无环检测**（见设计思想一节）
- **无 `MAXRECURSION` 式的 hint**：`cte_max_recursion_depth` 是唯一控制手段（源码里仅注释提及过这类 hint，未实现）
- **递归块内禁止 ORDER BY / LIMIT / DISTINCT**（破坏逐轮追加语义）

**★ 完整限制清单（11 条，源码注释取证）**：

| 限制 | 源码证据 |
|---|---|
| UNION 上不允许 ORDER BY | `sql_derived.cc:331-352`（理由：fake_query_block 会在表不完整时排序） |
| 递归块不允许 ORDER BY / LIMIT / SELECT DISTINCT | `sql_derived.cc:381-394` |
| 递归块不允许聚合 / 窗口函数 | `sql_union.cc:802-829`（`// Per SQL2011.`） |
| 递归表只能引用一次、不能在子查询里 | `sql_parse.cc:5847-5887` |
| 递归表不能在 LEFT JOIN 右操作数 | `sql_optimizer.cc:5322-5329` |
| 不允许右嵌套 UNION | `sql_derived.cc:369-380` |
| 不允许 UNION DISTINCT 后接 UNION ALL | `sql_derived.cc:395-412` |
| **不支持 MyISAM**（溢出盘必须 InnoDB） | `sql_derived.cc:337-339`：`"allowing ORDER BY <condition using fulltext> would make the UNION tmp table be of MyISAM engine which recursive CTEs don't support"` |
| 临时表不能用 MySQL 自建 PK | `sql_derived.cc:190`（`can_use_pk_for_unique = false`） |
| **互递归不支持** | `with_recursive.inc:6`，报 `ER_NO_SUCH_TABLE` |
| 递归成员列会被截断 → **即使非严格模式也报错** | `composite_iterators.cc:1006-1038` |

> ★ 注意 **UNION 级别的 `LIMIT` 是允许的**（`sql_derived.cc:340`："LIMIT is allowed and will stop the row generation after N rows"），被拒的是**递归块内**的 LIMIT。

**★ EXPLAIN 里的 CTE 显示**（⚠️ 真名核查：**没有** `Follow tail` 字符串）：

```cpp
/* sql/join_optimizer/explain_access_path.cc:1060 */
    case AccessPath::FOLLOW_TAIL:
      description =
          string("Scan new records on ") + path->follow_tail().table->alias;
      error |= AddMemberToObject<Json_string>(obj, "access_type",
                                              "scan_new_records");
```
```cpp
/* :335 —— CTE 物化行文案 */
      if (param->cte->recursive) {
        str = "Materialize recursive CTE " + to_string(param->cte->name);
      } else {
        if (is_set_operation) str = "Materialize union CTE " + ...;
        else                  str = "Materialize CTE " + ...;
        if (param->cte->tmp_tables.size() > 1) {
          str += " if needed";
          if (!explain_cte_now) str += " (query plan printed elsewhere)";
        }
      }
```
```cpp
/* :468 —— 递归块标题 */
    if (query_block.is_recursive_reference) {
      if (this_heading.empty()) this_heading = "Repeat until convergence";
      else this_heading += ", repeat until convergence";
    }
```
| 显示 | 字符串 |
|---|---|
| 递归 CTE 物化 | `Materialize recursive CTE <name>` |
| 非递归 CTE 物化 | `Materialize CTE <name>` / `Materialize union CTE <name>`（多引用加 ` if needed`） |
| 递归引用读取 | **`Scan new records on <alias>`**（JSON：`access_type=scan_new_records`） |
| 递归块标题 | `Repeat until convergence` |

```cpp
/* sql/opt_explain.cc:1250 —— TRADITIONAL/JSON 的 recursive 标记 */
  if (query_block->is_recursive()) {
    /*
      This will add the "recursive" word to:
      - the block of the JOIN, in JSON format
      - the first table of the JOIN, in TRADITIONAL format.
    */
    if (push_extra(ET_RECURSIVE)) return true;
  }
```

---

## 七、深潜补充：克隆表、校验清单、递归的三个细节

### 7.1 clone_tmp_table 的关键参数

```cpp
open_table_from_share(..., db_stat=0, EXTRA_RECORD | DELAYED_OPEN);
```

注释原文明确：**否则写者写行会覆盖读者刚读的行**——`db_stat=0` + `EXTRA_RECORD` 保证克隆表有独立的 record buffer，读者和写者互不干扰。

其余：`t->reginfo.lock_type = TL_WRITE`；`hash_field` 一并拷贝。

**★ 三个参数的精确含义**（`sql/sql_derived.cc:190`）：

```cpp
  if (open_table_from_share(thd, first->s, tl->alias,
                            /*
                              Pass db_stat == 0 to delay opening of table in SE,
                              as table is not instantiated in SE yet.
                            */
                            0,
                            /* We need record[1] for this TABLE instance. */
                            EXTRA_RECORD |
                                /*
                                  Use DELAYED_OPEN to have its own record[0]
                                  (necessary because db_stat is 0).
                                  Otherwise it would be shared with 'first'
                                  and thus a write to tmp table would modify
                                  the row just read by readers.
                                */
                                DELAYED_OPEN,
                            0, t, false, nullptr))
    return nullptr;
  assert(t->s == first->s && t != first && t->file != first->file);
  t->s->increment_ref_count();
  t->s->tmp_handler_count++;

  // In case this clone is used to fill the materialized table:
  bitmap_set_all(t->write_set);
  t->reginfo.lock_type = TL_WRITE;
  t->copy_blobs = true;
  tl->table = t;
  t->pos_in_table_list = tl;
  if (first->hash_field) t->hash_field = t->file->field[0];
  t->hidden_field_count = first->hidden_field_count;
  t->set_not_started();
  if (tmp_tables.push_back(tl)) return nullptr;
```
> ★ `db_stat = 0` —— 延迟在存储引擎层打开（临时表此刻还没实例化）。
> ★ `EXTRA_RECORD` —— 需要 `record[1]`（`check_unique_constraint()` 用双缓冲）。
> ★ `DELAYED_OPEN` —— **必须有独立的 `record[0]`**，否则与写者共享，"写一行会改掉读者刚读到的那一行"。
> ★ 三个断言式事实：`t->s == first->s`（共享 TABLE_SHARE）、`t != first`（不同 TABLE 对象）、`t->file != first->file`（**各有独立 handler ⇒ 独立游标**）。

**★ 共享机制的总设计文档**（`sql/sql_derived.cc:69`，★ 极长但极有价值，此处摘三种场景）：

```cpp
   How sharing of a single tmp table works
   =======================================

   There are several scenarios.
   (1) Non-recursive CTE referenced only once: nothing special.
   (2) Non-recursive CTE referenced more than once:
   - multiple TABLEs, one TABLE_SHARE.
   - The first ref in setup_materialized_derived() calls
   create_tmp_table(); others call open_table_from_share().
   - The first ref in create_derived() calls instantiate_tmp_table()
   (which calls handler::create() then open_tmp_table()); others call
   open_tmp_table(). open_tmp_table() calls handler::open().
   - The first ref in materialize_derived() evaluates the subquery and does
   all writes to the tmp table.
   - Finally all refs set up a read access method (table scan, index scan,
   index lookup, etc) and do reads, possibly interlaced (example: a
   nested-loop join of two references to the CTE).
   - The storage engine (MEMORY or InnoDB) must be informed of the uses above;
   this is done by having TABLE_SHARE::ref_count>=2 for every handler::open()
   call.
   (3) Recursive CTE, referenced once or more than once:
   All of (2) applies, where the set of refs is the non-recursive
   ones ... Additionally:
   - recursive refs do not call setup_materialized_derived(),
   create_derived(), materialize_derived().
   - right after a non-recursive ref has been in setup_materialized_derived(),
   its recursive refs are replaced with clones of that ref, made with
   open_table_from_share().
   ...
   - The storage engine (MEMORY or InnoDB) must be informed of the uses above;
   this is done by having TABLE_SHARE::ref_count>=2 for every handler::open()
   call.
   - The Server code handling tmp table creation must also be informed:
   see how Query_result_union::create_result_table() disables PK promotion.
```
> ★★ "对临时表的要求"一节（`sql_derived.cc:132-157`）还列出 6 条硬约束，核心是：**表扫描必须记住两次读之间的位置、必须返回插入顺序的行、必须返回稳定的 `position()`**。这些正是 `FollowTailIterator` 与 MEMORY/InnoDB 交互的契约。

### 7.2 结构校验的完整清单

除"必须有 UNION"之外，还有 5 条容易被忽略的禁止项：

| 禁止项 | 原因 |
|--------|------|
| ORDER BY over UNION | 会引入 MyISAM 临时表（递归不能用） |
| 右嵌套 UNION | — |
| 递归块内有 ORDER BY/LIMIT/DISTINCT | 破坏逐轮追加语义 |
| `last_distinct` 后跟 ALL | 去重语义冲突（见 6.1） |
| 非递归块必须在最前 | — |

**★ 补全为 14 条（含错误码与位置）**：

| # | 校验 | 错误码 | 位置 |
|---|---|---|---|
| 1 | 必须是 UNION | `ER_CTE_RECURSIVE_REQUIRES_UNION` | `sql_derived.cc:328` |
| 2 | UNION 上不能有 ORDER BY | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:348` |
| 3 | 不在 EXCEPT/INTERSECT 操作数 | `ER_CTE_RECURSIVE_NOT_UNION` | `sql_derived.cc:367` |
| 4 | 不右嵌套 UNION | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:376` |
| 5 | 递归块无 ORDER BY/LIMIT/DISTINCT | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:390` |
| 6 | 无 DISTINCT 后接 ALL | `ER_NOT_SUPPORTED_YET` | `sql_derived.cc:407` |
| 7 | anchor 全在递归块前 | `ER_CTE_RECURSIVE_REQUIRES_NONRECURSIVE_FIRST` | `sql_derived.cc:415` |
| 8 | 至少一个 anchor | 同上 | `sql_derived.cc:423` |
| 9 | 递归块无聚合/窗口 | `ER_CTE_RECURSIVE_FORBIDS_AGGREGATION` | `sql_union.cc:826` |
| 10 | 不在 LEFT JOIN 右操作数 / 不被 hint 强制非首位 | `ER_CTE_RECURSIVE_FORBIDDEN_JOIN_ORDER` | `sql_optimizer.cc:5331`、`join_optimizer.cc:3064` |
| 11 | 递归表只引用一次且不在子查询 | `ER_CTE_RECURSIVE_REQUIRES_SINGLE_REFERENCE` | `sql_parse.cc:5884`/`5908` |
| 12 | WITH 列表内同名 | `ER_NONUNIQ_TABLE` | `parse_tree_nodes.cc:1921` |
| 13 | 互递归 | **无专门码** → `ER_NO_SUCH_TABLE` | `with_recursive.inc:6` |
| 14 | 深度超限 | `ER_CTE_MAX_RECURSION_DEPTH` | `basic_row_iterators.cc:404` |

**★ 落点：三行代码确定"第一个递归块是谁"**（`sql_derived.cc:424`）：

```cpp
    derived->first_recursive = last_non_recursive->next_query_block();
    assert(derived->is_recursive());
```
> ★ 从此 `Query_expression::is_recursive()`（= `first_recursive != nullptr`）为真——这是后续所有递归分派的开关。

### 7.3 共享物化的读者同步

`use_shared_cte_materialization` 判定后只置 `table->materialized = true`；`Table_ref::materialize_derived` 经 **`Derived_refs_iterator`** 把所有**克隆**都标记为已物化（不只是主表）。

### 7.4 递归的三个细节

1. **强制严格模式**（`Strict_error_handler`）：列类型由 **anchor 成员**决定，若非严格模式下截断（如 anchor 是 `VARCHAR(10)`，递归部分产出更长值），会导致**无限递归**（值永不收敛）——所以非严格模式也要报错。
2. **深度计数是"轮数"不是行数**：`m_recursive_iteration_count` 在 `Init` 归零，**先自增再比较**。`cte_max_recursion_depth` 默认 **1000**。

**★ 第 2 点完整代码**（`basic_row_iterators.cc:399`，★ 见「4.5 深度限制」）：

```cpp
  if (m_read_rows == m_end_of_current_iteration) {
    // We have started a new iteration. Check to see if we have passed the
    // user-set limit.
    if (++m_recursive_iteration_count >
        thd()->variables.cte_max_recursion_depth) {
      my_error(ER_CTE_MAX_RECURSION_DEPTH, MYF(0), m_recursive_iteration_count);
      return 1;
    }
    m_end_of_current_iteration = *m_stored_rows;
#ifdef ENABLED_DEBUG_SYNC
    if (m_recursive_iteration_count == 4) {
      DEBUG_SYNC(thd(), "in_WITH_RECURSIVE");
    }
#endif
  }
```
> ★ 计数发生在**每次读到上一轮末尾（即新一轮开始）**时，不是每次 `Init()`。因为"one run can encompass many iterations"（一次 MaterializeQueryBlock 可能跑完整个查询）。

**★ 第 3 点：`Init()` 用 `file->inited` 作"是否首次"的信号**（`basic_row_iterators.cc:326`）：

```cpp
  const bool first_init = !table()->file->inited;
  if (first_init) {
    /*
      Before starting a new WITH RECURSIVE execution,
      MaterializeIterator::Init() does ha_index_or_rnd_end() on all read
      cursors of recursive members, which sets file->inited = false, so we can
      use that as a signal.
    */
    if (!table()->is_created()) {
      // Recursive references always refer to a temporary table,
      // which do not exist at resolution time; thus, we need to
      // connect to it on first run here.
      if (open_tmp_table(table())) return true;
    }
    int error = table()->file->ha_rnd_init(true);
    ...
    m_read_rows = 0;
    m_recursive_iteration_count = 0;
    m_end_of_current_iteration = 0;
  } else {
    // Just continue where we left off last time.   /* ★★ 游标不回退 */
  }
```

**★ 递归物化期间强制严格模式**（`composite_iterators.cc:1039`）：

```cpp
  Strict_error_handler strict_handler(
      Strict_error_handler::ENABLE_SET_SELECT_STRICT_ERROR_HANDLER);
```
3. **本版无 `MAX_RECURSION` hint**——只在注释里被提及，实际只能靠系统变量。

### 7.5 溢出转 InnoDB 与相关 CTE 重置

- **溢出**：`MaterializeQueryBlock` 中 `create_ondisk_from_heap` 后要调 `recursive_reader->RepositionCursorAfterSpillToDisk()`——因为引擎换了，游标必须重定位。
- **相关 CTE 重置**：`m_cte->clear_all_references()` 清空所有 `tmp_tables` 并回绕游标；必须让 `materialized = false`，否则写者会跳过物化。

**★ 溢出后通知所有 reader 重新定位**（`composite_iterators.cc:1389`）：

```cpp
      if (create_ondisk_from_heap(thd(), t, error,
                                  /*insert_last_record=*/true,
                                  /*ignore_last_dup=*/true, &is_duplicate))
        return true;
      // Table's engine changed; index is not initialized anymore.
      if (t->hash_field) t->file->ha_index_init(0, false);
      ...
      // Inform each reader that the table has changed under their feet,
      // so they'll need to reposition themselves.
      for (const materialize_iterator::QueryBlock &query_b :
           m_query_blocks_to_materialize) {
        if (query_b.is_recursive_reference) {
          query_b.recursive_reader->RepositionCursorAfterSpillToDisk();
        }
      }
```
```cpp
/* basic_row_iterators.cc:429 */
bool FollowTailIterator::RepositionCursorAfterSpillToDisk() {
  if (!m_inited) {
    // Spill-to-disk happened before we got to read a single row,
    // so the table has not been initialized yet. It will start
    // at the first row when we actually get to Init(), which is fine.
    return false;
  }
  return reposition_innodb_cursor(table(), m_read_rows);
}
```

**★ 溢出时递归引用游标的特殊处理**（`sql/sql_tmp_table.cc:2765`，★ 解释了 `tmp_tables` 顺序为何重要）：

```cpp
      if (table->pos_in_table_list &&
          table->pos_in_table_list->is_recursive_reference() &&
          table->file->inited) {
        /*
          Due to the last condition, this is guaranteed to be a recursive
          reference belonging to the unit which 'wtable' materializes, and not
          to the unit of another non-recursive reference (indeed, this other
          reference will re-use the rows of 'wtable', i.e. not execute its
          unit).
          This reference has opened a cursor.
          In the 'tmp_tables' list, 'wtable' is always before such recursive
          reference, as setup_materialized_derived_tmp_table() runs before
          substitute_recursive_reference(). So, we know the disk-based rows
          already exist at this point.
        */
        assert(rows_on_disk);
        (void)table->file->ha_rnd_end();
        rec_ref_w_open_cursor = true;
      }
```
> ★ ★ "写者表永远排在递归读者克隆之前"——因为 `setup_materialized_derived_tmp_table()` 先于 `substitute_recursive_reference()`。

**★ `clear_all_references()`：为什么 `references` 与 `tmp_tables` 要分两个数组**（`sql/sql_union.cc:1589`）：

```cpp
bool Common_table_expr::clear_all_references() {
  bool reset_tables = false;
  for (Table_ref *tl : references) {           /* ① 非递归克隆：一写者 + N 读者 */
    if (tl->table &&
        tl->derived_query_expression()->uncacheable & UNCACHEABLE_DEPENDENT) {
      reset_tables = true;
      if (tl->derived_query_expression()->query_result()->reset()) return true;
    }
    /*
      This loop has found all non-recursive clones; one writer and N
      readers.
    */
  }
  if (!reset_tables) return false;
  for (Table_ref *tl : tmp_tables) {           /* ② 递归克隆：只有读者 */
    if (tl->is_derived()) continue;  // handled above
    if (tl->table->empty_result_table()) return true;
    // This loop has found all recursive clones (only readers).
  }
  /*
    Above, emptying all clones is necessary, to rewind every handler (cursor) to
    the table's start. Setting materialized=false on all is also important or
    the writer would skip materialization, see loop at start of
    Table_ref::materialize_derived()). There is one "recursive table"
    which we don't find here: it's the UNION DISTINCT tmp table. It's reset in
    unit::execute() of the unit which is the body of the CTE.
  */
  return false;
}
```
> ★★ "**一写者 + N 读者**"的原文注释就在第 1597-1600 行。只有 `uncacheable & UNCACHEABLE_DEPENDENT`（相关/依赖外层值）才需要重置。

**★ 触发点（`sql/sql_union.cc:1619`）**：

```cpp
/**
  Empties all correlated query blocks defined within the query expression;
  that is, correlated CTEs defined in the expression's WITH clause, and
  correlated derived tables.
 */
bool Query_expression::clear_correlated_query_blocks() {
  ...
  if (!m_with_clause) return false;
  for (auto el : m_with_clause->m_list->elements()) {
    Common_table_expr &cte = el->m_postparse;
    if (cte.clear_all_references()) return true;
  }
  return false;
}
```
> ★ 注意它直接访问 private 成员 `el->m_postparse`——靠 `PT_common_table_expr` / `PT_with_clause` 里的 `friend bool Query_expression::clear_correlated_query_blocks();` 授权。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → WITH (Common Table Expressions)*
- *MySQL 8.0 Reference Manual → Recursive Common Table Expressions*（`cte_max_recursion_depth`）

**理论**
- SQL:1999 标准的递归查询（recursive query）——最小不动点（least fixed point）语义，是 `WITH RECURSIVE` 必须写成 anchor + UNION 递归块的根本原因

**相关文档**
- 优化器侧（CTE 为何不能 merge、与 derived 的差异）见 [`../07_optimize/logical/04_logical_join.md`](../07_optimize/logical/04_logical_join.md)
- 物化用的内部临时表（TempTable / MEMORY 引擎、降级链）见 [`07_temptable.md`](07_temptable.md)
