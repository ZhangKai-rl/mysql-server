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

### 六、ORDER BY / LIMIT 的处理与 fake query block

- 顶层 `ORDER BY` 由 **fake query block** 承载：它是一个额外的 `Query_block`，其 JOIN 的迭代器负责 LIMIT/OFFSET 与排序（源码注释："If there's a fake_query_block, its JOIN's iterator already handles LIMIT/OFFSET, so we don't do it again here"）。
- LIMIT/ORDER BY 可以**下推到 operand**：`pushdown_limit_order_by`（见 [`../05_contextualize.md`](../05_contextualize.md)）。
- 但 **INTERSECT / EXCEPT 的下推受限**：源码注释写明 *"Logic here presumes that query expressions that only add limit (not order by) will have been pushed down"*，并且只有 `cand->has_limit()` 时才允许（`level == 0 || cand->has_limit()`）。原因是集合运算的语义不允许在 operand 上提前截断（会影响交/差的结果）。

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

## 参考

**论文 / 理论**

- 《Halloween Problem》—— INSERT...SELECT 边扫边插的经典问题（`streaming_allowed` 第二条否定的根因）
- 多重集（multiset）语义 —— `INTERSECT ALL` / `EXCEPT ALL` 要求重复度语义，这是 MySQL 选择计数器算法而非哈希集合的根本原因

**官方文档**

- MySQL 8.0 Reference Manual, "INTERSECT / EXCEPT / UNION Clause"（8.0.31 起支持）
- MySQL 8.0 Reference Manual, "Optimizing UNION"

**相关文档**

- [`../05_contextualize.md`](../05_contextualize.md) —— `Query_term` 的解析、标签分配、`pushdown_limit_order_by`
- [`../runtime/08_materialization.md`](../runtime/08_materialization.md) —— 物化与临时表的运行期细节
- [`../09_executor_iterator.md`](../09_executor_iterator.md) —— `AppendIterator` / `MaterializeIterator` / `TableScanIterator`
- [`16_join_object_model.md`](16_join_object_model.md) —— `m_root_iterator` 归属于 unit（本篇的执行入口）
- [`../08_access_path/README.md`](../08_access_path/README.md) —— AccessPath 类型（`APPEND` / `MATERIALIZE` / `LIMIT_OFFSET`）
