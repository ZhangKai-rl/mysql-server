# 15 GROUP BY / DISTINCT / ORDER BY 在优化期的处理

> 基于 MySQL 8.0.39。本篇讲透 `JOIN::optimize()` 流程里与"分组 / 去重 / 排序"有关的三步：`optimize_aggregated_query`、`optimize_distinct_group_order`、`test_skip_sort`，以及它们在 `make_tmp_tables_info` 里如何决定"要不要临时表、要不要排序、用哪种聚合器"。
>
> **边界**：本篇只讲**经典优化器**（`JOIN::optimize`）这条路径。
> - 索引免排序的判定算法本身（`test_if_order_by_key` / `test_if_cheaper_ordering` / `test_if_skip_sort_order`）见 [`10_plan_refinement.md`](10_plan_refinement.md)，本篇只取"被谁调、结果怎么用"；
> - `GROUP_INDEX_SKIP_SCAN`（MIN/MAX 与松索引扫描）的区间构造见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)，本篇讲它在优化期被识别和被否决的位置；
> - 聚合器与 ROLLUP 的运行期状态机见 [`../runtime/06_rollup.md`](../runtime/06_rollup.md)，本篇只写"优化期因 rollup 否决了什么"；
> - Filesort 见 [`../runtime/01_filesort.md`](../runtime/01_filesort.md)，临时表引擎见 [`../runtime/07_temptable.md`](../runtime/07_temptable.md)；
> - hypergraph 优化器**不走本篇任何一步**，见 Misc 的「路径边界」。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、主链路：三步在流程中的位置](#一主链路三步在流程中的位置)
  - [二、optimize_aggregated_query：聚合常量化](#二optimize_aggregated_query聚合常量化)
    - [2.5 MIN/MAX 的索引端点分支](#25-minmax-的索引端点分支优化期就把值读出来)
  - [三、optimize_distinct_group_order 逐段剖析](#三optimize_distinct_group_order-逐段剖析)
  - [四、test_skip_sort：能否用索引做 GROUP BY](#四test_skip_sort能否用索引做-group-by)
  - [五、四种 GROUP BY 实现的判定链](#五四种-group-by-实现的判定链)
  - [六、make_tmp_tables_info：临时表与排序的最终判定](#六make_tmp_tables_info临时表与排序的最终判定)
  - [七、与 ROLLUP / 窗口 / HAVING / ONLY_FULL_GROUP_BY 的交互](#七与-rollup--窗口--having--only_full_group_by-的交互)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

一个 `Query_block` 的 `JOIN::optimize()` 里，有三步专门负责"GROUP BY / DISTINCT / ORDER BY 到底怎么做"：

| 步骤 | 在流程中的位置 | 一句话职责 |
|---|---|---|
| `optimize_aggregated_query` | `make_join_plan()` **之前** | 无 GROUP BY 的聚合查询，能否**在优化期就把结果算出来**（`COUNT(*)` 用引擎精确行数、`MIN/MAX` 走索引端点），从而把整个 JOIN 消掉 |
| `optimize_distinct_group_order` | `make_join_plan()` **之后**（`make_join_query_block` 之后） | 消掉常量排序项；判定 GROUP BY / DISTINCT 能否整组消掉；能否把 DISTINCT **改写成 GROUP BY**；能否让 GROUP BY 顺带提供顺序从而删掉 ORDER BY |
| `test_skip_sort` | `alloc_qep` 之后、`make_join_readinfo` 之前 | 判定能否**用某个索引的顺序**直接满足 GROUP BY / ORDER BY，从而免掉排序 |

三步都是**规则驱动**的（不做代价比较），它们改的是 `JOIN` 上的一组标志：`grouped`、`select_distinct`、`simple_group`、`simple_order`、`skip_sort_order`、`streaming_aggregation`、`m_ordered_index_usage`、`need_tmp_before_win`、`tmp_table_param.allow_group_via_temp_table`。后面 `make_tmp_tables_info` 读这些标志决定物理实现。

### 一个反直觉的开场

同一张表、同一个 GROUP BY，加一个 `SQL_BIG_RESULT` 就把"索引扫描"变成"全表扫 + filesort"：

```sql
EXPLAIN SELECT b, SUM(c) FROM t1 GROUP BY b;
-- type=index, key=b, Extra 为空            ← 索引有序，流式聚合

EXPLAIN SELECT SQL_BIG_RESULT b, SUM(c) FROM t1 GROUP BY b;
-- type=ALL, key=NULL, Extra=Using filesort ← 禁止临时表后，退化为"先排序再流式聚合"
```

`SQL_BIG_RESULT` 的文档语义是"结果集很大，不要用带唯一键的临时表"。但它在 `test_skip_sort` 里的实际效果是**关掉了"用索引做 GROUP BY"的尝试**，而"用索引做 GROUP BY"恰恰是唯一能免掉排序的路径——于是反而被逼去 filesort。这就是一个"提示语义"与"实现判据"错位的典型案例，后面第四章会把这个判定式拆开讲。

### 为什么需要：顺序需求的传递

SQL 语义上三者的输出要求不同：

- `GROUP BY`：分组键相同的行必须**成簇**到达（不要求全局有序）
- `DISTINCT`：要求**全局去重**（实现上等价于按全部输出列分组）
- `ORDER BY`：要求**全局有序**

而逻辑执行顺序是固定的：

```
WHERE → GROUP BY(+聚合) → HAVING → 窗口函数 → DISTINCT → ORDER BY → LIMIT
```

关键性质是：**前一步的实现会吃掉后一步的需求**。如果 GROUP BY 恰好按 ORDER BY 的前缀产出顺序，ORDER BY 就可以整段删掉（`test_if_subpart` 判据）；反之，如果 GROUP BY 选择了哈希/临时表实现，输出顺序是哈希序，ORDER BY 就必须另起一次排序。

优化器的全部工作，就是在这条链上**为每个环节挑一种物理实现，并让"顺序"这种属性沿着链尽量往下传**。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | 聚合只有 `end_send_group` / `end_write_group` 两条老路径；`optimize_aggregated_query` 已有但只处理 `COUNT/MIN/MAX` |
| 5.7 | `ORDER_with_src` 之前，ORDER 无法溯源，EXPLAIN 分不清排序来自 ORDER BY 还是 GROUP BY |
| 8.0.14+ | 窗口函数进入 `JOIN::m_windows`；`m_windows_sort` / `need_tmp_before_win` 加入判定链，窗口函数成为"必须用临时表"的新来源 |
| 8.0.2x | 迭代器化后，聚合实现收敛为 `AggregateIterator`（流式）与 `TemptableAggregateIterator`（临时表）两种；`streaming_aggregation` 标志成为二者的开关 |
| 8.0.39 | `SQL_BIG_RESULT` / `with_json_agg` 参与 `test_skip_sort` 判定；`is_agg_loose_index_scan` 成为"LIS 已完成预计算"的识别手段 |

---

## 理论基础

### 论文谱系：顺序优化的三个里程碑

本篇涉及的 GROUP BY / DISTINCT / ORDER BY 决策，在学术界有非常清晰的演进线，MySQL 停在**第一个里程碑附近**：

| 里程碑 | 论文 | 贡献 | MySQL 的实现状态 |
|---|---|---|---|
| ① **interesting order** | Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979) | 提出"有趣顺序"：join order 搜索时为每个中间结果记录它能提供的顺序，从而让 `ORDER BY` / `GROUP BY` / `MERGE JOIN` 免费获得顺序 | ⚠️ **只实现了一个退化版本**：`m_ordered_index_usage` 是三态枚举（`VOID` / `GROUP_BY` / `ORDER_BY`），**一个表集合只保留一个顺序候选**，且 GROUP BY 与 ORDER BY 互斥二选一。System R 原意是为每个中间结果保存一个"顺序集合" |
| ② **order optimization 的完备化** | Simmen, Shekita, Malkemus《Fundamental Techniques for Order Optimization》(**EDBT 1996**) | DB2 的顺序优化技术：把 sort **下推**到 join 之下、最小化排序次数、**利用函数依赖推导顺序** | ❌ 未实现。MySQL 没有"排序下推"与"FD 推导顺序"（FD 推导只在 hypergraph 的 interesting orders 里有初步实现，见 [`physical/09_hypergraph.md`](physical/09_hypergraph.md)） |
| ③ **grouping 与 order 的统一框架** | Neumann & Moerkotte《A Combined Framework for Grouping and Order Optimization》(VLDB 2004) | ★ **首次给出完整、正确、高效的"顺序 + 分组"联合推理算法**。论文明确指出此前"no complete, correct, and efficient algorithm for ordering and grouping inference has been proposed" | ❌ 未实现。MySQL 的分组实现选择是**纯规则**（本篇第五章），代价模型完全不参与 |

★ 里程碑③是理解 MySQL 现状的关键参照物：Neumann-Moerkotte 那篇论文的价值恰恰在于**"分组"和"顺序"必须联合推理**（因为分组会改变输出顺序，顺序需求会反过来约束分组实现的选择）。MySQL 把这两件事拆成了两个互不通信的判定（`optimize_distinct_group_order` 与 `test_skip_sort`），这正是本篇里那些"反直觉"现象（如 `SQL_BIG_RESULT` 反而变慢）的根源。

**分组算法的分类**则来自 Graefe《Query Evaluation Techniques for Large Databases》(ACM Computing Surveys 1993) 的经典二分：基于排序的聚合 vs 基于哈希的聚合。MySQL 只实现了前者（外加"临时表唯一键"这种哈希形态，但选择它是规则而非代价）。

### 三种"分组实现"的本质：顺序 vs 哈希

分组在关系代数里只有两种物理实现，这是数据库教科书的经典分野：

| | **基于排序 / 有序输入**（sort-based / streaming） | **基于哈希**（hash-based） |
|---|---|---|
| 前提 | 输入按分组键**有序**（或成簇） | 无前提 |
| 做法 | 边读边比：当前行分组键 == 上一行则累加，否则切换新组 | 哈希表按分组键聚合 |
| 内存 | **O(1)**（只保存当前组的聚合状态） | O(分组数) |
| 输出顺序 | **分组键有序**（免费送给下游 ORDER BY） | 哈希序（下游要排序得另起） |
| 能否流式输出 | 能 | 必须读完才输出（除非分区） |

MySQL 的选择是：**优先基于排序，且排序尽量由索引免费提供**；哈希分组只在"拿不到顺序且允许临时表"时才用（虽然 `TemptableAggregateIterator` 的底层临时表用的是哈希索引或 B-tree 唯一键，本质上仍是哈希分组）。

**为什么 MySQL 不做哈希聚合的代价决策**？因为它在这里是**纯规则**的：能不能拿到顺序（索引或 filesort）是"能/不能"的二值判断，不是代价权衡。代价只在更前面（`make_join_plan` 选访问路径时）被考虑过一次。这是 MySQL 优化器"规则 + 代价混用"的又一例证——分组实现的选择被划进了规则侧。

### DISTINCT 即 GROUP BY：一个等价变换

`SELECT DISTINCT a, b` 等价于 `SELECT a, b GROUP BY a, b`（在没有聚合函数时）。MySQL 真的做了这个改写（第三章的 `create_order_from_distinct`），动机不是语义，而是**实现复用**：

- DISTINCT 只有"靠临时表的唯一键去重"或"靠排序后相邻去重"两条路；
- 一旦改写成 GROUP BY，就能复用"索引有序 → 流式去重"这条更便宜的路。

所以判定式里出现了一个看似奇怪的条件：**只有当"改写后的 GROUP BY 能用索引、或者没有 LIMIT"时，才做这个改写**。否则改写只会多出一个分组步骤。

### 他库对比

| 库 | 分组实现选择 | 与 MySQL 的差异 |
|---|---|---|
| PostgreSQL | `GroupAggregate`（需排序）vs `HashAggregate`，**由代价模型决定**，且 `HashAggregate` 会考虑 `work_mem` 溢出到磁盘的代价 | PG 是代价驱动，MySQL 是规则驱动；PG 有真正的 hash agg，MySQL 的"临时表聚合"是哈希索引临时表，代价模型里没有"分组数"这一项 |
| Oracle | `HASH GROUP BY` / `SORT GROUP BY`，代价驱动，有 `PGA` 预算 | 同上，且 Oracle 有 GROUP BY 并行 |
| MySQL | 规则驱动：优先索引有序 → 否则若允许临时表则临时表聚合 → 否则 filesort + 流式聚合 | 代价模型完全不参与这一步 |

**代价**：MySQL 无法表达"分组数很少时哈希分组更划算"这类权衡，也无法为"分组数很多导致临时表落盘"估算代价（`tmp_table_size` / `max_heap_table_size` 是硬阈值，不是代价项）。

---

## 核心实现

### 一、主链路：三步在流程中的位置

```
JOIN::optimize()
  ├─ optimize_cond()                    ← 谓词改写（等值传播、常量传播）
  ├─ prune_table_partitions()
  ├─ optimize_aggregated_query()        ← ① 聚合常量化（可能直接结束优化）
  │     └─ 若返回 AGGR_COMPLETE / AGGR_DELAYED：JOIN 被消掉，走常量结果
  ├─ substitute_gc()                    ← 生成列替换
  ├─ [hypergraph 分支：到此 return]
  ├─ make_join_plan()                   ← join order + 访问方法（代价驱动）
  ├─ substitute_for_best_equal_field()
  ├─ make_join_query_block()
  ├─ optimize_distinct_group_order()    ← ② 消序 / 消组 / DISTINCT→GROUP BY
  ├─ setup_join_buffering()
  ├─ alloc_qep()
  ├─ test_skip_sort()                   ← ③ 能否用索引顺序免排序
  ├─ finalize_table_conditions()
  ├─ make_join_readinfo()
  ├─ make_tmp_tables_info()             ← 读上面所有标志，定物理实现
  ├─ create_access_paths()
  └─ push_to_engines()
```

**为什么 ① 在 `make_join_plan` 之前、② 在之后？**

- ① 一旦成功，**整个 JOIN 就不存在了**（`SELECT COUNT(*) FROM t` 不需要真的去 join），所以必须尽早，省掉后面的全部搜索；
- ② 依赖 join order：判定"GROUP BY 能否用索引"要看**第一张表**是哪个（`best_ref[const_tables]`），这只有 join order 定了才知道。

这个"① 极早、② 较晚"的分布，是理解这三步的第一条线索。

---

### 二、optimize_aggregated_query：聚合常量化

> 位于 `sql/opt_sum.cc`，是本篇中唯一**在优化期就执行查询**的一步。

#### 2.1 三态决策

函数通过出参 `aggregate_evaluated *decision` 返回三态之一：

| 返回值 | 含义 | 后果 |
|---|---|---|
| `AGGR_REGULAR` | 放弃优化，常规执行 | 继续走 `make_join_plan` |
| `AGGR_COMPLETE` | **优化期已算出全部聚合值** | JOIN 被替换成一行常量结果 |
| `AGGR_DELAYED` | 聚合值可在**执行期**用引擎接口廉价取得（`ha_records()` 精确计数） | JOIN 仍保留，但聚合函数走 `end_send_count` 类快路径 |

三态的存在说明了两件事：**有些行数在优化期就能精确知道，有些必须等到执行期**（因为优化期还没打开表或表还没被填充）。

**枚举定义**（`sql/sql_select.h:784`）——★ 注意它其实是 **4 个值**，常说的"三态"只描述了 `decision` 的**赋值语句**，第 4 态 `AGGR_EMPTY` 是"源表为空"的**提前 return** 出口：

```cpp
// sql/sql_select.h:781
/* functions from opt_sum.cc */
bool is_simple_predicate(Item_func *func_item, Item **args, bool *inv_order);

enum aggregate_evaluated {
  AGGR_COMPLETE,  // All aggregates were evaluated
  AGGR_REGULAR,   // Aggregates not fully evaluated, regular execution required
  AGGR_DELAYED,   // Aggregates not fully evaluated, execute with ha_records()
  AGGR_EMPTY      // Source tables empty, aggregates are NULL or 0 (for COUNT)
};

bool optimize_aggregated_query(THD *thd, Query_block *select,
                               const mem_root_deque<Item *> &all_fields,
                               Item *conds, aggregate_evaluated *decision);
```

```cpp
// sql/opt_sum.cc:619 —— 结尾的三态赋值（AGGR_EMPTY 在 :552 提前 return 时已赋过）
*decision = aggr_impossible ? AGGR_REGULAR
                            : aggr_delayed ? AGGR_DELAYED : AGGR_COMPLETE;
```

**调用点与三态的落地**（`sql/sql_optimizer.cc:514`）：

```cpp
// sql/sql_optimizer.cc:508
/*
   Try to optimize count(*), min() and max() to const fields if
   there is implicit grouping (aggregate functions but no
   group_list). In this case, the result set shall only contain one
   row.
*/
if (tables_list && implicit_grouping &&
    !(query_block->active_options() & OPTION_NO_CONST_TABLES)) {
  aggregate_evaluated outcome;
  if (optimize_aggregated_query(thd, query_block, *fields, where_cond,
                                &outcome)) {
    error = 1;
    DBUG_PRINT("error", ("Error from optimize_aggregated_query"));
    return true;
  }
  switch (outcome) {
    case AGGR_REGULAR:
      // Query was not (fully) evaluated. Revert to regular optimization.
      break;
    case AGGR_DELAYED:
      // Query was not (fully) evaluated. Revert to regular optimization,
      // but indicate that storage engine supports HA_COUNT_ROWS_INSTANT.
      select_count = true;   // ★ 执行期走 UnqualifiedCountIterator
      break;
    case AGGR_COMPLETE: {
```

> ★ **勘误**：5.7 资料里常说的 `opt_sum_query()` 在 **8.0.39 已不存在**（全库 0 处 C++ 定义，只剩 mysql-test 注释/结果文本里的残留）。它的职责被**内联进 `optimize_aggregated_query()` 的 `case Item_sum::MIN_FUNC / MAX_FUNC` 分支**（`opt_sum.cc:458-596`）。同理，`min_max_in_range()` 不存在（真名 `maxmin_in_range()`）、`Item_sum_min_max` 不存在（是 `Item_sum_hybrid` + `Item_sum_min`/`Item_sum_max`）、`Item_sum::reset_and_add_if_needed()` 不存在（真名 `reset_and_add()`）。

`AGGR_DELAYED` 的执行期落地（`UnqualifiedCountIterator::Read`，`sql/iterators/ref_row_iterators.cc:846`）：

```cpp
int UnqualifiedCountIterator::Read() {
  if (!m_has_row) {
    return -1;
  }

  for (Item *item : *m_join->fields) {
    if (item->type() == Item::SUM_FUNC_ITEM &&
        down_cast<Item_sum *>(item)->sum_func() == Item_sum::COUNT_FUNC) {
      int error;
      ulonglong count = get_exact_record_count(m_join->qep_tab,
                                               m_join->primary_tables, &error);
      // 【注】这是 ref_row_iterators.cc:822 的同名 static 版本（签名不同，带 index 参数）
      if (error) return 1;

      down_cast<Item_sum_count *>(item)->make_const(
          static_cast<longlong>(count));
    }
  }

  // If we are outputting to a temporary table, we need to copy the results
  // into it here. It is also used for nonaggregated items, even when there are
  // no temporary tables involved.
  if (copy_funcs(&m_join->tmp_table_param, m_join->thd)) {
    return 1;
  }

  m_has_row = false;
  return 0;
}
```

#### 2.2 准入条件（逐条）

函数开头有三道硬否决，每条都对应一类语义陷阱：

```
1. WHERE 条件引用了外层引用或随机函数（OUTER_REF_TABLE_BIT | RAND_TABLE_BIT）
   → 直接放弃。理由写在注释里：子查询只优化一次却可能执行多次，
     "结果集是否为空"依赖外层行时，MIN() 这类依赖空集语义的函数不能被常量化。

2. 存在 semi-join / anti-join 的 nest
   → 直接放弃。因为 semi-join 会去重，改变行数语义。

3. 某个表是外连接的 inner table，且 WHERE 条件引用了它
   → 直接放弃。反例注释给的是：
     SELECT MAX(t1.a) FROM t1 LEFT JOIN t2 ON ... WHERE t2.field IS NULL
     —— 这个 WHERE 会把 LEFT JOIN 的补 NULL 行筛出来，行数语义被改变。
```

第三条是本函数最精妙的地方：**外连接的 inner table 在没有被 WHERE 引用时，行数等于 1（补 NULL 行不参与计数）还是 N？** MySQL 选择了保守——只要 WHERE 碰了它，就放弃优化。

**完整源码**（`sql/opt_sum.cc:311`）——注意**第三道否决不在顶层并列 if 里，而在 `for (tl = tables...)` 循环体内**：

```cpp
// sql/opt_sum.cc:309
  const table_map where_tables = conds ? conds->used_tables() : 0;
  /*
    A subquery is optimized once but executed possibly multiple times.
    If the value of the set function depends on the join's emptiness (like
    MIN() does), and the join's emptiness depends on the outer row or
    something nondeterministic, we cannot mark the set function as constant:
   */
  // ★ 硬否决 #1：WHERE 里出现外部相关引用 或 随机/不确定表达式
  if (where_tables & (OUTER_REF_TABLE_BIT | RAND_TABLE_BIT)) return false;

  // ★ 硬否决 #2：存在 semi-join / anti-join nest（会去重，改变行数语义）
  if (!select->sj_nests.empty())
    // Cannot optimize when there is a semijoin or antijoin
    return false;

  /*
    Analyze outer join dependencies, and, if possible, compute the number
    of returned rows.
  */
  for (Table_ref *tl = tables; tl; tl = tl->next_leaf) {
    // Don't replace expression on a table that is part of an outer join
    if (tl->is_inner_table_of_outer_join()) {
      inner_tables |= tl->map();

      /*
        We can't optimise LEFT JOIN in cases where the WHERE condition
        restricts the table that is used, like in:
          SELECT MAX(t1.a) FROM t1 LEFT JOIN t2 join-condition
          WHERE t2.field IS NULL;
      */
      // ★ 硬否决 #3（在循环体内）：外连接的 inner table 被 WHERE 条件引用
      if (tl->map() & where_tables) return false;
    } else
      used_tables |= tl->map();
```

#### 2.3 行数的两条来源：精确 vs 延迟

对每个叶子表：

```
if (引擎有 HA_STATS_RECORDS_IS_EXACT && 表已填充 && 没有 FORCE INDEX)
     row_count *= file->stats.records          ← 直接从统计信息拿，优化期可得
else
     delay_ha_records_to_exec_phase |= !(引擎有 HA_COUNT_ROWS_INSTANT) || FORCE INDEX
     have_exact_count = false
```

三个细节值得注意：

- **`FORCE INDEX` 会让两条路都失效**。注释解释是"用户明确指定了索引，那就别用 `stats.records` 走捷径"——这是对 hint 的尊重，代价是放弃优化。
- **schema 表（information_schema）与需要物化的派生表**在此时还没填充，`tables_filled` 为 false，因此它们的行数不可信。
- **混合引擎会拖垮整体**：`SELECT COUNT(*) FROM t_myisam, t_innodb` 里 MyISAM 有精确计数、InnoDB 没有，结果 `have_exact_count=false`，所有表都要在执行期真读（注释里就是这个例子）。

这正是"为什么 `COUNT(*)` 在 InnoDB 上慢"的源码级答案：InnoDB 不提供 `HA_STATS_RECORDS_IS_EXACT`，只能走 `ha_records()` 真扫，或者退化为 `end_send_count` 走索引。

**完整源码**（`sql/opt_sum.cc:351`）：

```cpp
// sql/opt_sum.cc:351
    /*
      If the storage manager of 'tl' gives exact row count as part of
      statistics (cheap), compute the total number of rows. If there are
      no outer table dependencies, this count may be used as the real count.
      Schema tables are filled after this function is invoked, so we can't
      get row count.
      Derived tables aren't filled yet, their number of rows are estimates.
      FORCE INDEX implies that user wants a specific index to be used. So skip
      using stats.records.
    */
    // ★ tables_filled：INFORMATION_SCHEMA 系统表 / 物化派生表 尚未填充 → 行数不可信
    tables_filled &= !(tl->schema_table || tl->uses_materialization());
    ulonglong table_flags = tl->table->file->ha_table_flags();
    if ((table_flags & HA_STATS_RECORDS_IS_EXACT) && tables_filled &&
        !tl->table->force_index) {
      error = tl->fetch_number_of_rows();
      if (error) {
        tl->table->file->print_error(error, MYF(ME_FATALERROR));
        return true;
      }
      row_count *= tl->table->file->stats.records;   // ★ 来源 A：优化期乘上 stats.records
    } else {
      /*
        Note: If at least one of the tables can't be optimized,
              then all tables will be read in the execution phase
              (i.e. end_send_count).

        Example: SELECT COUNT(*) FROM t_myisam, t_innodb;
      */
      // ★ 来源 B：不支持 HA_COUNT_ROWS_INSTANT 或用户 FORCE INDEX → 推迟到执行期再数
      delay_ha_records_to_exec_phase |=
          !(table_flags & HA_COUNT_ROWS_INSTANT) || tl->table->force_index;

      have_exact_count = false;
    }
```

以及"现算"路径 `get_exact_record_count()`（`opt_sum.cc:103`，专为 NDB 这类"有 `HA_COUNT_ROWS_INSTANT` 但没 `HA_STATS_RECORDS_IS_EXACT`"的引擎）：

```cpp
// sql/opt_sum.cc:94
/**
  Get exact count of rows in all tables. This is called, when at least one of
  the table handlers support HA_COUNT_ROWS_INSTANT, but not
  HA_STATS_RECORDS_IS_EXACT (NDB is one such storage engine).

    @param tables  List of tables
    @retval Product of number of rows in all tables. ULLONG_MAX for error.
*/
static ulonglong get_exact_record_count(Table_ref *tables) {
  ulonglong count = 1;
  for (Table_ref *tl = tables; tl; tl = tl->next_leaf) {
    ha_rows tmp = 0;
    int error = tl->table->file->ha_records(&tmp);
    if (error != 0) return ULLONG_MAX;   // 【注】失败用 ULLONG_MAX 表示，调用方据此判定
    count *= tmp;                        // 【注】笛卡尔积：多表 JOIN 的行数是各表行数相乘
  }
  return count;
}
```

> 注意**有两个同名 static `get_exact_record_count()`**：优化期版在 `opt_sum.cc:103`（签名 `Table_ref*`），执行期版在 `iterators/ref_row_iterators.cc:822`（签名 `QEP_TAB*, uint, int*`，会按访问类型选 `ha_records(&tmp)` 还是 `ha_records(&tmp, index)`）。

#### 2.4 COUNT(expr) 的常量化条件

```
if (conds == nullptr                        // 没有 WHERE
    && !arg0->is_nullable()                 // COUNT 的参数不可能为 NULL
    && !inner_tables                        // 没有外连接的 inner table
    && tables_filled)                       // 表都已填充
```

四条件全部满足，`COUNT(expr)` 才能换成"表的笛卡尔积行数"。**`arg0->is_nullable()` 这一条**是语义正确性的关键：`COUNT(expr)` 不计 NULL，只有参数非空时才等于行数。这也是 `COUNT(*)` 通常比 `COUNT(col)` 快的原因之一——`COUNT(*)` 的参数是常量 `1`，天然非空。

另有一条全文检索特例：单表 + WHERE 是单个 `MATCH` + 引擎能提供 FTS 结果行数 + 参数非空 → **优化期直接跑全文检索**拿到真实 count 并替换。

**完整源码**（`sql/opt_sum.cc:393`）：

```cpp
// sql/opt_sum.cc:393
        case Item_sum::COUNT_FUNC: {
          Item_sum_count *item_count = down_cast<Item_sum_count *>(item_sum);
          /*
            If the expr in COUNT(expr) can never be null we can change this
            to the number of rows in the tables if this number is exact and
            there are no outer joins.
          */
          // ★ 四个条件：①conds == nullptr ②参数非 nullable ③无 outer-join inner table ④表已填充
          if (conds == nullptr && !item_count->get_arg(0)->is_nullable() &&
              !inner_tables && tables_filled) {
            if (delay_ha_records_to_exec_phase) {
              aggr_delayed = true;      // ★ 走 AGGR_DELAYED：执行期用 ha_records()
            } else {
              if (!have_exact_count) {
                row_count = get_exact_record_count(tables);
                if (row_count == ULLONG_MAX) {
                  /*
                    Error from handler in counting rows. Don't optimize count()
                  */
                  aggr_impossible = true;
                  continue;
                }
                have_exact_count = true;  // count is now exact
              }
            }
          }
          /*
            For result count of full-text search: If
            1. it is a single table query,
            2. the WHERE condition is a single MATCH expression,
            3. the table engine can provide the row count from FTS result, and
            4. the expr in COUNT(expr) can not be NULL,
            we do the full-text search now, and replace with the actual count.

            Note: Item_func_match::init_search() will be called again later in
                  the optimization phase by init_fts_funcs(), but search will
                  still only be done once.
          */
          else if (tables->next_leaf == nullptr &&                       // 1
                   (func_type == Item_func::FT_FUNC ||
                    func_type == Item_func::MATCH_FUNC) &&               // 2
                   (tables->table->file->ha_table_flags() &
                    HA_CAN_FULLTEXT_EXT) &&                              // 3
                   !item_count->get_arg(0)->is_nullable())               // 4
          {
            Item_func_match *fts_item =
                func_type == Item_func::FT_FUNC
                    ? down_cast<Item_func_match *>(conds)
                    : down_cast<Item_func_match *>(
                          down_cast<Item_func_match_predicate *>(conds)
                              ->arguments()[0]);
            fts_item->get_master()->set_hints(nullptr, FT_NO_RANKING,
                                              HA_POS_ERROR, false);
            if (fts_item->init_search(thd)) break;  // ★ break 而非 aggr_impossible：保留常规执行
            row_count = fts_item->get_count();
            have_exact_count = true;
          } else
            aggr_impossible = true;

          // See comment above for get_exact_record_count()
          if (!thd->lex->is_explain() && !aggr_impossible && !aggr_delayed) {
            // ★ EXPLAIN 时不写常量值，只保留优化判定
            item_count->make_const((longlong)row_count);
            recalc_const_item = true;
          }
          break;
        }
```

```cpp
// sql/item_sum.h:1100 —— make_const 的两个重载
  void no_rows_in_result() override { count = 0; }
  void make_const(longlong count_arg) {
    count = count_arg;
    Item_sum::make_const();
  }

// sql/item_sum.h:594 —— 基类版本只置 forced_const
  void make_const() {
    // "forced_const" will make used_tables() return zero for this object
    forced_const = true;
  }
```

#### 2.5 MIN/MAX 的索引端点分支：优化期就把值读出来

上面讲的是 `optimize_aggregated_query` 的 **COUNT 半边**。同一个函数的另一半是 **MIN/MAX**——它走的是完全不同的机制，而且是全库**唯一一类"优化期就执行查询"**的优化：直接去索引端点读一行，把聚合函数换成常量，整个 JOIN 随之消失（`EXPLAIN` 里表现为 **"Select tables optimized away"**）。

> 这也是为什么 `SELECT MAX(id) FROM t` 秒回、而 `SELECT MAX(name) FROM t`（`name` 无索引）要全表扫。

**能优化的查询形状**——`opt_sum.cc:24` 的文件头注释就是一份规格说明：

```cpp
// sql/opt_sum.cc:24
/**
  @file

  Optimising of MIN(), MAX() and COUNT(*) queries without 'group by' clause
  by replacing the aggregate expression with a constant.

  Given a table with a compound key on columns (a,b,c), the following
  types of queries are optimised (assuming the table handler supports
  the required methods)

  @verbatim
  SELECT COUNT(*) FROM t1[,t2,t3,...]
  SELECT MIN(b) FROM t1 WHERE a=const
  SELECT MAX(c) FROM t1 WHERE a=const AND b=const
  SELECT MAX(b) FROM t1 WHERE a=const AND b<const
  SELECT MIN(b) FROM t1 WHERE a=const AND b>const
  SELECT MIN(b) FROM t1 WHERE a=const AND b BETWEEN const AND const
  SELECT MAX(b) FROM t1 WHERE a=const AND b BETWEEN const AND const
  @endverbatim

  Instead of '<' one can use '<=', '>', '>=' and '=' as well.
  Instead of 'a=const' the condition 'a IS NULL' can be used.

  If all selected fields are replaced then we will also remove all
  involved tables and return the answer without any join. Thus, the
  following query will be replaced with a row of two constants:
  @verbatim
  SELECT MAX(b), MIN(d) FROM t1,t2
    WHERE a=const AND b<const AND d>const
  @endverbatim
  (assuming a index for column d of table t2 is defined)
*/
```

```cpp
// sql/opt_sum.cc:86 —— 全文件只有这 3 个 static 前向声明（8.0.39 无 opt_sum_query / min_max_in_range）
static bool find_key_for_maxmin(bool max_fl, Index_lookup *ref,
                                Item_field *item_field, Item *cond,
                                uint *range_fl, uint *key_prefix_length);
static bool reckey_in_range(bool max_fl, Index_lookup *ref,
                            Item_field *item_field, Item *cond, uint range_fl,
                            uint prefix_len);
static bool maxmin_in_range(bool max_fl, Item_field *item_field, Item *cond);
```

**① 三道硬否决**（在函数开头，命中即放弃）：

```cpp
  const table_map where_tables = conds ? conds->used_tables() : 0;
  /*
    A subquery is optimized once but executed possibly multiple times.
    If the value of the set function depends on the join's emptiness (like
    MIN() does), and the join's emptiness depends on the outer row or
    something nondeterministic, we cannot mark the set function as constant:
   */
  if (where_tables & (OUTER_REF_TABLE_BIT | RAND_TABLE_BIT)) return false;

  if (!select->sj_nests.empty())
    // Cannot optimize when there is a semijoin or antijoin
    return false;
```

第三条针对外连接内表——注释给了反例：

```cpp
        We can't optimise LEFT JOIN in cases where the WHERE condition
        restricts the table that is used, like in:
          SELECT MAX(t1.a) FROM t1 LEFT JOIN t2 join-condition
          WHERE t2.field IS NULL;
```

**② `find_key_for_maxmin()`：能不能用索引端点**

先要求字段是某个键的一部分，然后逐个索引试：

```cpp
  if (!field->is_flag_set(PART_KEY_FLAG)) return false;  // Not key field
  ...
    /* Perform a check if index is not disabled by ALTER TABLE or IGNORE INDEX. */
    if (!table->keys_in_use_for_query.is_set(idx)) continue;
    ...
    for (part = keyinfo->key_part, part_end = part + actual_key_parts(keyinfo);
         part != part_end; part++, jdx++, ...) {
      if (!(table->file->index_flags(idx, jdx, false) & HA_READ_ORDER))
        return false;
      // Due to lack of time, currently only ASC keyparts are supported.
      if (part->key_part_flag & HA_REVERSE_SORT) break;
```

三个约束值得注意：

- 遍历段数用的是 **`actual_key_parts()`** 而非 `user_defined_key_parts()`——即**索引扩展参与进来**（见 [`physical/07_access_method.md`](physical/07_access_method.md) 4.4）。二级索引隐含主键列，所以 `SELECT MAX(id) FROM t WHERE a = 3`（`KEY(a)`）也能走端点。
- 索引必须支持 `HA_READ_ORDER`（有序是端点读的前提）。
- ⚠️ **`HA_REVERSE_SORT` 直接 `break`**——注释写着 *"Due to lack of time, currently only ASC keyparts are supported"*。降序索引的段不支持这条优化。
- 另有一条约束（源码注释）：**只有其余 keypart 全部被绑定时**才能用（`"find_key_for_maxmin only if all other keyparts are bound"`）。

**③ `is_simple_predicate()`：认哪些 WHERE 形态**

只有"字段与常量比较"才能转成索引的搜索键。三分支：

```cpp
    case 0:
      /* MULT_EQUAL_FUNC */
        Item_equal *item_equal = down_cast<Item_equal *>(func_item);
        args[0] = item_equal->get_first();
        if (item_equal->members() > 1) return false;   // 等值传播多于 2 个成员则放弃
    case 1:
      /* field IS NULL */
      item = func_item->arguments()[0];
      if (item->type() != Item::FIELD_ITEM) return false;
    case 2:
      /* 'field op const' or 'const op field' */
      ...
      if (!item->const_for_execution()) return false;
```

`inv_order` 出参标记 `const op field` 这种反序写法。多等值（`a = b = 3` 且成员 > 2）直接放弃。

**④ `matching_cond()`：递归检查 WHERE 是否"可控"**

递归遍历条件树，输出被绑定的 keypart 集合与端点是否开区间：

- 条件不涉及该表 → 直接 `true`（不影响）
- **`COND_OR_FUNC` → 直接 `false`**（OR 一律放弃，与 range 优化器不同）
- AND → 逐项递归
- 注释明说 `key_part_used` **由调用方初始化为 0**，因为它是递归累加的

**⑤ 真正去读：`get_index_min_value()` / `get_index_max_value()`**

```cpp
static int get_index_max_value(TABLE *table, Index_lookup *ref, uint range_fl) {
  return (ref->key_length
              ? table->file->ha_index_read_map(
                    table->record[0], ref->key_buff,
                    make_prev_keypart_map(ref->key_parts),
                    range_fl & NEAR_MAX ? HA_READ_BEFORE_KEY
                                        : HA_READ_PREFIX_LAST_OR_PREV)
              : table->file->ha_index_last(table->record[0]));
}
```

无搜索键（无 WHERE）时退化为 `ha_index_first` / `ha_index_last`——**一次 handler 调用拿到结果**。

★ 有搜索键时的两条路径值得拆开看，**读法枚举的选择就是"MAX 怎么绕开扫描"的答案**：

| 场景 | 路径 | 读法 |
|---|---|---|
| `SELECT MAX(b) FROM t`（无前缀） | `ref->key_length == 0` | `ha_index_last`——索引最后一条，O(1) |
| `SELECT MAX(b) FROM t WHERE a=1`（前缀有匹配） | `ha_index_read_map` + `make_prev_keypart_map(key_parts)` | `HA_READ_PREFIX_LAST_OR_PREV`——**前缀内最后一条**；若该前缀不存在，**再往前一条**（前缀前最后一条） |
| `SELECT MAX(b) FROM t WHERE a<5`（上界开区间） | 同上 | `range_fl & NEAR_MAX` → `HA_READ_BEFORE_KEY`——**严格小于 key 的最后一条** |

三个细节：

1. **`make_prev_keypart_map(ref->key_parts)`**——构造的 keypart 位图只覆盖**前缀列**（`a`），不含被取 MAX 的列（`b`）：搜索键是"前缀的最后一个可能值"，读方向负责在其中找最大 `b`。
2. ★ **`HA_READ_PREFIX_LAST_OR_PREV` 的 "OR_PREV" 是隐藏的关键**——它处理了"前缀值本身不存在"的情况：`WHERE a<=5` 时若没有 `a=5` 的行，会取 `a<5` 范围里最后一条。没有这个语义，`<=` 条件的 MAX 就漏答案了。
3. **`NEAR_MAX` 判定在读之前**——range 优化器把上界是开还是闭（`<` vs `<=`）编码进 `range_fl`，这里只负责选枚举：开区间（`<5`）必须用 `HA_READ_BEFORE_KEY` 严格小于，闭区间才能用 `PREFIX_LAST_OR_PREV`。

MIN 侧还有一段注释说明三条规则：

```
      1) Insert the minimum non-null values where the WHERE clause still
         matches, or
      2) a NULL value if there are only NULL values for key_part_k.
      3) Fail, producing a row of nulls
```

**⑥ ★ 为什么取到端点行后还要再判一次（`reckey_in_range`）**

这是最容易看漏的一步——索引端点读到的那一行**未必满足 WHERE**，所以必须回判：

```cpp
static bool reckey_in_range(bool max_fl, Index_lookup *ref,
                            Item_field *item_field, Item *cond, uint range_fl,
                            uint prefix_len) {
  if (key_cmp_if_same(item_field->field->table, ref->key_buff, ref->key, prefix_len))
    return true;                    // ① 常量前缀已不匹配 → 读到的行跑出范围了
  if (cond == nullptr) return false;
  // Constant conditions which were not evaluated earlier (in optimize_cond())
  // should be evaluated now.
  if (cond->const_for_execution()) return (cond->val_int() == 0);   // ② 常量条件现在才求值
  if (range_fl & (max_fl ? NO_MIN_RANGE : NO_MAX_RANGE)) return false;
  return maxmin_in_range(max_fl, item_field, cond);                 // ③ 用真实行再判范围条件
}
```

三个分支各有分工：①是"前缀失配"（`WHERE a = 3` 但端点行 `a != 3`，说明该前缀下无行）；②是优化期还没求值的常量条件现在补求；③`maxmin_in_range()` 递归检查落在 **MIN/MAX 字段本身**上的范围条件（如 `WHERE a = 3 AND id > 10` 里的 `id > 10`）。

任何一步判定"不满足"，这次 MIN/MAX 优化就作废，`decision` 退回 `AGGR_REGULAR` 走常规执行。

**⑦ 端点开闭：`NEAR_MIN` / `NEAR_MAX`**

`>` / `<` 这类严格比较会把端点设为**开区间**，读的时候改用 `HA_READ_AFTER_KEY` / `HA_READ_BEFORE_KEY`（`range_fl & NEAR_MAX ? HA_READ_BEFORE_KEY : HA_READ_PREFIX_LAST_OR_PREV`）。开区间标记在遍历条件时置位/清零（`*range_fl |= NEAR_MIN` / `&= ~NEAR_MIN`）。

**已知限制汇总**

| 限制 | 后果 |
|---|---|
| 字段不在任何索引里 | 全表扫聚合 |
| **降序 keypart**（`HA_REVERSE_SORT`） | 注释自认"暂不支持"（*"Due to lack of time"*） |
| **WHERE 里有 OR** | 一律放弃（`matching_cond` 直接 `false`） |
| 有 semijoin / antijoin | 放弃 |
| 子查询 + 依赖外层行 / 随机 | 放弃（优化一次执行多次，不能当常量） |
| 外连接内表且 WHERE 限制该表 | 放弃 |
| **前缀索引（partial key part）** | `part->length < part_field->key_length()` ⇒ `break` 换下一把索引 |

#### 2.5.1 MIN/MAX 分支完整源码（`optimize_aggregated_query`，`opt_sum.cc:458`）

这就是 5.7 时代 `opt_sum_query()` 干的那件事——**8.0 已内联进主函数**：

```cpp
// sql/opt_sum.cc:458
        case Item_sum::MIN_FUNC:
        case Item_sum::MAX_FUNC: {
          int is_max = (item_sum->sum_func() == Item_sum::MAX_FUNC);
          /*
            If MIN/MAX(expr) is the first part of a key or if all previous
            parts of the key is found in the COND, then we can use
            indexes to find the key.
          */
          Item *expr = item_sum->get_arg(0)->real_item();
          if (expr->type() == Item::FIELD_ITEM) {
            uchar key_buff[MAX_KEY_LENGTH];
            Index_lookup ref;
            uint range_fl, prefix_len;

            ref.key_buff = key_buff;
            Item_field *item_field = down_cast<Item_field *>(expr);
            Table_ref *tr = item_field->table_ref;
            TABLE *table = tr->table;
            if (table->const_table) {          // 已读过的常量表 → 放弃
              aggr_impossible = true;
              break;
            }
            assert(!table->file->inited);       // 这张表实例必须还没被访问过

            /*
              Look for a partial key that can be used for optimization.
              If we succeed, ref.key_length will contain the length of
              this key, while prefix_len will contain the length of
              the beginning of this key without field used in MIN/MAX().
              Type of range for the key part for this field will be
              returned in range_fl.
            */
            // ★ 第一步：找索引并构造搜索键（详见 2.5.2）
            if ((inner_tables & tr->map()) ||
                !find_key_for_maxmin(is_max, &ref, item_field, conds, &range_fl,
                                     &prefix_len)) {
              aggr_impossible = true;
              break;
            }
            if ((error = table->file->ha_index_init((uint)ref.key, true))) {
              table->file->print_error(error, MYF(0));
              table->set_keyread(false);
              return true;
            }

            /*
              Necessary columns to read from the index have been determined by
              find_key_for_maxmin(); they are the columns involved in
              'WHERE col=const' and the aggregated one.
            */
            assert(table->read_set == &table->def_read_set);
            assert(bitmap_is_clear_all(&table->tmp_set));
            table->read_set = &table->tmp_set;
            table->mark_columns_used_by_index_no_reset(ref.key, table->read_set,
                                                       ref.key_parts);
            // The aggregated column may or not be included in ref.key_parts.
            bitmap_set_bit(table->read_set, item_field->field->field_index());
            // ★ 第二步：真正去索引端点读一行（详见 2.5.5）
            error = is_max ? get_index_max_value(table, &ref, range_fl)
                           : get_index_min_value(table, &ref, item_field,
                                                 range_fl, prefix_len);

            /*
              Set table row status to "not started" unconditionally.  This will
              prepare the table for regular access in the join execution
              machinery if this optimization is aborted and cannot be used.
            */
            table->set_not_started();

            table->read_set = &table->def_read_set;
            bitmap_clear_all(&table->tmp_set);
            /* Verify that the read tuple indeed matches the search key */
            // ★ 第三步：回读校验（reckey_in_range 返回 true ⇒ 读到的行不在范围内）
            if (!error && reckey_in_range(is_max, &ref, item_field, conds,
                                          range_fl, prefix_len))
              error = HA_ERR_KEY_NOT_FOUND;
            table->set_keyread(false);
            table->file->ha_index_end();
            if (error) {
              if (error == HA_ERR_KEY_NOT_FOUND ||
                  error == HA_ERR_END_OF_FILE) {
                *decision = AGGR_EMPTY;        // ★★ 第四态：源表为空
                return false;                  // No rows matching WHERE
              }
              /* HA_ERR_LOCK_DEADLOCK or some other error */
              table->file->print_error(error, MYF(0));
              return true;
            }
            removed_tables |= tr->map();
          } else if (!expr->const_for_execution() || conds != nullptr ||
                     !have_exact_count) {
            /*
              We get here if the aggregate function is not based on a field.
              Example: "SELECT MAX(1) FROM table ..."
              ...
            */
            aggr_impossible = true;
            break;
          }
          // ★ 与 Item_sum 的交互点①：决定 SIMPLE / DISTINCT aggregator
          item_sum->set_aggregator(item_sum->has_with_distinct()
                                       ? Aggregator::DISTINCT_AGGREGATOR
                                       : Aggregator::SIMPLE_AGGREGATOR);
          /*
            If row_count == 0 and there are no outer joins, set to NULL,
            otherwise set to the constant value.
          */
          if (have_exact_count && row_count == 0 && !inner_tables) {
            item_sum->aggregator_clear();
            item->no_rows_in_result();          // ★ 交互点②：零行结果（MIN/MAX → NULL）
          } else
            item_sum->reset_and_add();          // ★ 交互点③：从 record[0] 取刚读到的那一行
          item_sum->make_const();               // ★ 交互点④：forced_const = true
          recalc_const_item = true;
          break;
        }
```

收尾还有一个容易漏的判据——**MIN/MAX 必须把所有被用到的表都"消掉"**，否则宁可放弃：

```cpp
// sql/opt_sum.cc:616
  /*
    With a where clause, only ignore searching in the tables if MIN/MAX
    optimisation replaced all used tables.
    We do not use replaced values in case of:
    SELECT MIN(key) FROM table_1, empty_table
    removed_tables is != 0 if we have used MIN() or MAX().
  */
  if (removed_tables && used_tables != removed_tables)
    aggr_impossible = true;  // We didn't remove all tables
```

#### 2.5.2 `find_key_for_maxmin()` 完整源码（`opt_sum.cc:969`）

```cpp
// sql/opt_sum.cc:969
static bool find_key_for_maxmin(bool max_fl, Index_lookup *ref,
                                Item_field *item_field, Item *cond,
                                uint *range_fl, uint *prefix_len) {
  Field *const field = item_field->field;

  if (!field->is_flag_set(PART_KEY_FLAG)) return false;  // Not key field

  DBUG_TRACE;

  TABLE *const table = field->table;
  uint idx = 0;

  KEY *keyinfo, *keyinfo_end;
  for (keyinfo = table->key_info, keyinfo_end = keyinfo + table->s->keys;
       keyinfo != keyinfo_end; keyinfo++, idx++) {
    KEY_PART_INFO *part, *part_end;
    key_part_map key_part_to_use = 0;
    /*
      Perform a check if index is not disabled by ALTER TABLE
      or IGNORE INDEX.
    */
    if (!table->keys_in_use_for_query.is_set(idx)) continue;
    uint jdx = 0;
    *prefix_len = 0;
    // ★ actual_key_parts()：索引扩展后的 keypart 数（二级索引隐含主键列）
    for (part = keyinfo->key_part, part_end = part + actual_key_parts(keyinfo);
         part != part_end;
         part++, jdx++, key_part_to_use = (key_part_to_use << 1) | 1) {
      // ★ 引擎必须支持有序读，否则无法走端点定位
      if (!(table->file->index_flags(idx, jdx, false) & HA_READ_ORDER))
        return false;
      // Due to lack of time, currently only ASC keyparts are supported.
      if (part->key_part_flag & HA_REVERSE_SORT) break;
      // 【注】是 break（换下一把索引继续找），不是 return false

      /* Check whether the index component is partial */
      Field *part_field = table->field[part->fieldnr - 1];
      if (part_field->is_flag_set(BLOB_FLAG) ||
          part->length < part_field->key_length())
        break;                                  // ★ 前缀索引不支持

      if (field->eq(part->field)) {
        ref->key = idx;
        ref->key_length = 0;
        ref->key_parts = 0;
        key_part_map key_part_used = 0;
        *range_fl = NO_MIN_RANGE | NO_MAX_RANGE;
        if (matching_cond(max_fl, ref, keyinfo, part, cond,
                          item_field->table_ref->map(), &key_part_used,
                          range_fl, prefix_len) &&
            !(key_part_to_use & ~key_part_used)) {
          // ★★ 聚合列之前的每个 keypart 都必须被等值 / IS NULL 覆盖
          if (!max_fl && key_part_used == key_part_to_use && part->null_bit) {
            /*
              The query is on this form:

              SELECT MIN(key_part_k)
              FROM t1
              WHERE key_part_1 = const and ... and key_part_k-1 = const

              If key_part_k is nullable, we want to find the first matching row
              where key_part_k is not null. The key buffer is now {const, ...,
              NULL}. This will be passed to the handler along with a flag
              indicating open interval. If a tuple is read that does not match
              these search criteria, an attempt will be made to read an exact
              match for the key buffer.
            */
            /* Set the first byte of key_part_k to 1, that means NULL */
            ref->key_buff[ref->key_length] = 1;
            ref->key_length += part->store_length;
            ref->key_parts++;
            assert(ref->key_parts == jdx + 1);
            *range_fl &= ~NO_MIN_RANGE;
            *range_fl |= NEAR_MIN;  // Open interval
          }
          /*
            The following test is false when the key in the key tree is
            converted (for example to upper case)
          */
          if (field->part_of_key.is_set(idx)) table->set_keyread(true);
          return true;
        }
      }
    }
  }
  return false;
}
```

#### 2.5.3 `is_simple_predicate()` 完整源码（`opt_sum.cc:635`）

★ 注意它是**按 `argument_count()` 分派**的（`MULT_EQUAL_FUNC` 的 `argument_count()` 是 0），且 **`case 3`（BETWEEN）没有 break，直接落到 `return true`**：

```cpp
// sql/opt_sum.cc:635
bool is_simple_predicate(Item_func *func_item, Item **args, bool *inv_order) {
  Item *item;
  *inv_order = false;
  switch (func_item->argument_count()) {
    case 0:
      /* MULT_EQUAL_FUNC */
      {
        Item_equal *item_equal = down_cast<Item_equal *>(func_item);
        args[0] = item_equal->get_first();
        // ★ 等值传播多于 2 个成员则放弃（a = b = const 这种）
        if (item_equal->members() > 1) return false;
        if (!(args[1] = item_equal->const_arg())) return false;
      }
      break;
    case 1:
      /* field IS NULL */
      item = func_item->arguments()[0];
      if (item->type() != Item::FIELD_ITEM) return false;
      args[0] = item;
      break;
    case 2:
      /* 'field op const' or 'const op field' */
      item = func_item->arguments()[0];
      if (item->type() == Item::FIELD_ITEM) {
        args[0] = item;
        item = func_item->arguments()[1];
        if (!item->const_for_execution()) return false;
        args[1] = item;
      } else if (item->const_for_execution()) {
        args[1] = item;
        item = func_item->arguments()[1];
        if (item->type() != Item::FIELD_ITEM) return false;
        args[0] = item;
        *inv_order = true;   // ★ 'const op field' ⇒ matching_cond 会翻转 less_fl
      } else
        return false;
      break;
    case 3:
      /* field BETWEEN const AND const */
      item = func_item->arguments()[0];
      if (item->type() != Item::FIELD_ITEM) return false;
      args[0] = item;
      for (int i = 1; i <= 2; i++) {
        item = func_item->arguments()[i];
        if (!item->const_for_execution()) return false;
        args[i] = item;
      }
      // 【注】case 3 没有 break，直接落到 return true
  }
  return true;
}
```

#### 2.5.4 `matching_cond()` 完整源码（`opt_sum.cc:740`）

它有三个身份：**①合法性检查 ②搜索键构造 ③开闭区间标记**。其中最精妙的是注释里承认的 **over-eager evaluation**（把常量写进 `Field` 对象本身，"边求值边收紧"）：

```cpp
// sql/opt_sum.cc:685（doxygen 摘录）
   As a side-effect, the key value to be used for looking up the MIN/MAX value
   is actually stored inside the Field object. An interesting feature is that
   the function will find the most restrictive endpoint by over-eager
   evaluation of the @c WHERE condition. It continually stores the current
   endpoint inside the Field object. For a query such as

   @code
   SELECT MIN(a) FROM t1 WHERE a > 3 AND a > 5;
   @endcode

   the algorithm will recurse over the conjunction, storing first a 3 in the
   field. In the next recursive invocation the expression a > 5 is evaluated
   as 3 > 5 (Due to the dual nature of Field objects as value carriers and
   field identifiers), which will obviously fail, leading to 5 being stored in
   the Field object.
```

```cpp
// sql/opt_sum.cc:740
static bool matching_cond(bool max_fl, Index_lookup *ref, KEY *keyinfo,
                          KEY_PART_INFO *field_part, Item *cond, table_map map,
                          key_part_map *key_part_used, uint *range_fl,
                          uint *prefix_len) {
  DBUG_TRACE;
  if (!cond) return true;

  if (!(cond->used_tables() & map)) {
    /* Condition doesn't restrict the used table */
    return true;
  }
  if (cond->type() == Item::COND_ITEM) {
    // ★★ OR 直接返回 false —— MIN/MAX 端点优化只支持合取式
    if (((Item_cond *)cond)->functype() == Item_func::COND_OR_FUNC)
      return false;

    /* AND */
    List_iterator_fast<Item> li(*((Item_cond *)cond)->argument_list());
    Item *item;
    while ((item = li++)) {
      if (!matching_cond(max_fl, ref, keyinfo, field_part, item, map,
                         key_part_used, range_fl, prefix_len))
        return false;
    }
    return true;
  }

  if (cond->type() != Item::FUNC_ITEM)
    return false;  // Not operator, can't optimize

  bool eq_type = false;          // =, <=> or IS NULL
  bool is_null_safe_eq = false;  // The operator is NULL safe, e.g. <=>
  bool noeq_type = false;        // < or >
  bool less_fl = false;          // < or <=
  bool is_null = false;          // IS NULL
  bool between = false;          // BETWEEN ... AND ...

  switch (((Item_func *)cond)->functype()) {
    case Item_func::ISNULL_FUNC:
      is_null = true;
      [[fallthrough]];
    case Item_func::EQ_FUNC:
      eq_type = true;
      break;
    case Item_func::EQUAL_FUNC:
      eq_type = is_null_safe_eq = true;
      break;
    case Item_func::LT_FUNC:
      noeq_type = true;
      [[fallthrough]];
    case Item_func::LE_FUNC:
      less_fl = true;
      break;
    case Item_func::GT_FUNC:
      noeq_type = true;
      [[fallthrough]];
    case Item_func::GE_FUNC:
      break;
    case Item_func::BETWEEN:
      between = true;
      // NOT BETWEEN is equivalent to OR and is therefore not a conjunction
      if (((Item_func_between *)cond)->negated) return false;
      break;
    case Item_func::MULT_EQUAL_FUNC:
      eq_type = true;
      break;
    default:
      return false;  // Can't optimize function
  }

  Item *args[3];
  bool inv;

  /* Test if this is a comparison of a field and constant */
  if (!is_simple_predicate(down_cast<Item_func *>(cond), args, &inv))
    return false;

  if (!is_null_safe_eq && !is_null &&
      (args[1]->is_null() || (between && args[2]->is_null())))
    return false;

  if (inv && !eq_type) less_fl = !less_fl;  // Convert '<' -> '>' (etc)

  /* Check if field is part of the tested partial key */
  uchar *key_ptr = ref->key_buff;
  KEY_PART_INFO *part;
  for (part = keyinfo->key_part;; key_ptr += part++->store_length)
  {
    if (part > field_part) return false;  // Field is beyond the tested parts
    if (part->field->eq(((Item_field *)args[0])->field))
      break;  // Found a part of the key for the field
  }

  bool is_field_part = part == field_part;
  // ★ 只有"聚合列所在 keypart"或"等值 / IS NULL 条件"才能推进；前缀必须等值
  if (!(is_field_part || eq_type)) return false;

  key_part_map org_key_part_used = *key_part_used;
  if (eq_type || between || max_fl == less_fl) {
    size_t length = (key_ptr - ref->key_buff) + part->store_length;
    if (ref->key_length < length) {
      /* Ultimately ref->key_length will contain the length of the search key */
      ref->key_length = length;
      ref->key_parts = (part - keyinfo->key_part) + 1;
    }
    if (!*prefix_len && part + 1 == field_part) *prefix_len = length;
    if (is_field_part && eq_type) *prefix_len = ref->key_length;

    // ★ key_part_used 累加：记录哪些 keypart 已被条件覆盖（find_key_for_maxmin 用它做全覆盖校验）
    *key_part_used |= (key_part_map)1 << (part - keyinfo->key_part);
  }

  if (org_key_part_used == *key_part_used &&
      /*
        The current search key is not being extended with a new key part.  This
        means that the a condition is added a key part for which there was a
        previous condition. We can only overwrite such key parts in some special
        cases, e.g. a > 2 AND a > 1 (here range_fl must be set to something). In
        all other cases the WHERE condition is always false anyway.
      */
      (eq_type || *range_fl == 0))
    return false;

  if (org_key_part_used != *key_part_used ||
      (is_field_part && (between || eq_type || max_fl == less_fl) &&
       !cond->val_int())) {
    /*
      It's the first predicate for this part or a predicate of the
      following form  that moves upper/lower bounds for max/min values:
      - field BETWEEN const AND const
      - field = const
      - field {<|<=} const, when searching for MAX
      - field {>|>=} const, when searching for MIN
    */

    if (is_null || (is_null_safe_eq && args[1]->is_null())) {
      if (!part->field->is_nullable()) return false;
      part->field->set_null();
      *key_ptr = (uchar)1;                    // ★ key_buff 的 NULL flag 字节
    } else {
      /* Update endpoints for MAX/MIN, see function comment. */
      Item *value = args[between && max_fl ? 2 : 1];
      type_conversion_status retval =
          value->save_in_field_no_warnings(part->field, true);
      if (!(retval == TYPE_OK ||
            (retval == TYPE_NOTE_TRUNCATED && part->field->is_text_key_type() &&
             part->field->charset()->pad_attribute == PAD_SPACE))) {
        return false;
      }

      if (part->null_bit) *key_ptr++ = (uchar)(part->field->is_null());
      part->field->get_key_image(key_ptr, part->length, Field::itRAW);
      // ★★ over-eager evaluation 的落点：常量真的被写进 Field，下一条合取项会拿它求值
    }
    if (is_field_part) {
      if (between || eq_type)
        *range_fl &= ~(NO_MAX_RANGE | NO_MIN_RANGE);
      else {
        *range_fl &= ~(max_fl ? NO_MAX_RANGE : NO_MIN_RANGE);
        if (noeq_type)
          *range_fl |= (max_fl ? NEAR_MAX : NEAR_MIN);    // ★ 严格不等 ⇒ 开区间
        else
          *range_fl &= ~(max_fl ? NEAR_MAX : NEAR_MIN);
      }
    }
  } else if (eq_type) {
    if ((!is_null && !cond->val_int()) || (is_null && !part->field->is_null()))
      return false;  // Impossible test
  } else if (is_field_part)
    *range_fl &= ~(max_fl ? NO_MIN_RANGE : NO_MAX_RANGE);
  return true;
}
```

#### 2.5.5 `get_index_min_value()` / `get_index_max_value()` 完整源码

★ 二者**不对称**：MIN 需要处理"该前缀下全是 NULL"的语义（多一次 `HA_READ_KEY_EXACT` 回退读），MAX 不需要——因为 NULL 排序在最前，读"最后一个"天然跳过 NULL。

```cpp
// sql/opt_sum.cc:114（doxygen 摘录：MIN 的三条规则）
    /*
      Use index to replace MIN/MAX functions with their values
      according to the following rules:

      1) Insert the minimum non-null values where the WHERE clause still
         matches, or
      2) a NULL value if there are only NULL values for key_part_k.
      3) Fail, producing a row of nulls

      Implementation: Read the smallest value using the search key. If
      the interval is open, read the next value after the search
      key. If read fails, and we're looking for a MIN() value for a
      nullable column, test if there is an exact match for the key.
    */
```

```cpp
// sql/opt_sum.cc:128
static int get_index_min_value(TABLE *table, Index_lookup *ref,
                               Item_field *item_field, uint range_fl,
                               uint prefix_len) {
  int error;

  if (!ref->key_length)
    error = table->file->ha_index_first(table->record[0]);   // ★ 无搜索键：索引第一条
  else {
    if (!(range_fl & NEAR_MIN))
      /*
         Closed interval: Either The MIN argument is non-nullable, or
         we have a >= predicate for the MIN argument.
      */
      error = table->file->ha_index_read_map(
          table->record[0], ref->key_buff,
          make_prev_keypart_map(ref->key_parts), HA_READ_KEY_OR_NEXT);
    else {
      /*
        Open interval: There are two cases:
        1) We have only MIN() and the argument column is nullable, or
        2) there is a > predicate on it, nullability is irrelevant.
        We need to scan the next bigger record first.
        Open interval is not used if the search key involves the last keypart,
        and it would not work.
      */
      assert(prefix_len < ref->key_length);
      error = table->file->ha_index_read_map(
          table->record[0], ref->key_buff,
          make_prev_keypart_map(ref->key_parts), HA_READ_AFTER_KEY);
      /*
         If the found record is outside the group formed by the search
         prefix, or there is no such record at all, check if all
         records in that group have NULL in the MIN argument
         column. If that is the case return that NULL.
      */
      if (item_field->field->is_nullable() && ref->key_buff[prefix_len] == 1 &&
          (error == HA_ERR_KEY_NOT_FOUND ||
           key_cmp_if_same(table, ref->key_buff, ref->key, prefix_len))) {
        assert(item_field->field->is_nullable());
        error = table->file->ha_index_read_map(
            table->record[0], ref->key_buff,
            make_prev_keypart_map(ref->key_parts), HA_READ_KEY_EXACT);
        // ★ 回退读"精确匹配"：key_buff 末位被 find_key_for_maxmin 置为 1（NULL）
      }
    }
  }
  return error;
}
```

```cpp
// sql/opt_sum.cc:212
static int get_index_max_value(TABLE *table, Index_lookup *ref, uint range_fl) {
  return (ref->key_length
              ? table->file->ha_index_read_map(
                    table->record[0], ref->key_buff,
                    make_prev_keypart_map(ref->key_parts),
                    range_fl & NEAR_MAX ? HA_READ_BEFORE_KEY
                                        : HA_READ_PREFIX_LAST_OR_PREV)
              : table->file->ha_index_last(table->record[0]));
}
```

#### 2.5.6 `reckey_in_range()` / `maxmin_in_range()` 完整源码

```cpp
// sql/opt_sum.cc:1067
static bool reckey_in_range(bool max_fl, Index_lookup *ref,
                            Item_field *item_field, Item *cond, uint range_fl,
                            uint prefix_len) {
  // ① 先检查读到的记录是否仍在"前缀常量组"内（防止越过前缀边界）
  if (key_cmp_if_same(item_field->field->table, ref->key_buff, ref->key,
                      prefix_len))
    return true;
  if (cond == nullptr) return false;
  // Constant conditions which were not evaluated earlier (in optimize_cond())
  // should be evaluated now.
  if (cond->const_for_execution()) return (cond->val_int() == 0);   // ② 常量条件现在补求
  if (range_fl & (max_fl ? NO_MIN_RANGE : NO_MAX_RANGE)) return false;
  // 【注】MIN 看 NO_MAX_RANGE / MAX 看 NO_MIN_RANGE：那一侧本来就无界，无需再校验
  return maxmin_in_range(max_fl, item_field, cond);                 // ③ 用真实行再判
}
```

```cpp
// sql/opt_sum.cc:1091
static bool maxmin_in_range(bool max_fl, Item_field *item_field, Item *cond) {
  /* If AND/OR condition */
  if (cond->type() == Item::COND_ITEM) {
    List_iterator_fast<Item> li(*down_cast<Item_cond *>(cond)->argument_list());
    Item *item;
    while ((item = li++)) {
      if (maxmin_in_range(max_fl, item_field, item)) return true;
    }
    return false;
  }

  // Check that condition actually references the specific table
  if ((cond->used_tables() & item_field->table_ref->map()) == 0) return false;
  bool less_fl = false;
  switch (down_cast<Item_func *>(cond)->functype()) {
    case Item_func::BETWEEN:
      return cond->val_int() == 0;
    case Item_func::LT_FUNC:
    case Item_func::LE_FUNC:
      less_fl = true;
      [[fallthrough]];
    case Item_func::GT_FUNC:
    case Item_func::GE_FUNC: {
      Item *item = down_cast<Item_func *>(cond)->arguments()[1];
      /* In case of 'const op item' we have to swap the operator */
      if (!item->const_for_execution()) less_fl = !less_fl;
      /*
        We only have to check the expression if we are using an expression like
        SELECT MAX(b) FROM t1 WHERE a=const AND b>const
        not for
        SELECT MAX(b) FROM t1 WHERE a=const AND b<const
      */
      if (max_fl != less_fl)
        return cond->val_int() == 0;  // Return true if condition is false
      return false;
    }
    case Item_func::EQ_FUNC:
    case Item_func::EQUAL_FUNC:
    case Item_func::MULT_EQUAL_FUNC:
    case Item_func::ISNULL_FUNC:
      break;   // ★ 等值条件已在 find_key_for_maxmin/matching_cond 阶段处理，无需再校验
    default:          // Keep compiler happy
      assert(false);  // Impossible
      break;
  }
  return false;
}
```

> ★ **为什么只校验"另一侧"**：`MAX(b) WHERE b>const` 的上界无约束，端点行必然满足；只有 `b<const` 这种"限制了 MAX 所在方向"的条件才需要回判——这就是 `max_fl != less_fl` 的含义。

---

### 三、optimize_distinct_group_order 逐段剖析

> 位于 `sql/sql_optimizer.cc:1456`。这是本篇的核心，函数约 185 行，但每一段都在改一个全局标志。
>
> ★ **勘误**：它返回 **`bool`**（true = 出错），不是 void；整个函数**没有** doxygen 注释块。

**完整源码**（`sql/sql_optimizer.cc:1456-1640`）：

```cpp
bool JOIN::optimize_distinct_group_order() {                          // :1456
  DBUG_TRACE;
  ASSERT_BEST_REF_IN_JOIN_ORDER(this);
  const bool windowing = m_windows.elements > 0;
  const bool may_trace = select_distinct || !group_list.empty() ||
                         !order.empty() || windowing ||
                         tmp_table_param.sum_func_count;
  Opt_trace_context *const trace = &thd->opt_trace;
  Opt_trace_disable_I_S trace_disabled(trace, !may_trace);
  Opt_trace_object wrapper(trace);
  Opt_trace_object trace_opt(trace, "optimizing_distinct_group_by_order_by");

  /* Optimize distinct away if possible */
  {
    ORDER *org_order = order.order;
    order = ORDER_with_src(                                           // :1470
        remove_const(order.order, where_cond, rollup_state == RollupState::NONE,
                     &simple_order, false),      // ★ 第 5 参 group_by = false（这是 ORDER BY）
        order.src, /*const_optimized=*/true);
    if (thd->is_error()) {
      error = 1;
      DBUG_PRINT("error", ("Error from remove_const"));
      return true;
    }

    /*
      If we are using ORDER BY NULL or ORDER BY const_expression,
      return result in any order (even if we are using a GROUP BY)
    */
    if (order.empty() && org_order) skip_sort_order = true;           // :1484
  }
  /*
     Check if we can optimize away GROUP BY/DISTINCT.
     We can do that if there are no aggregate functions, the
     fields in DISTINCT clause (if present) and/or columns in GROUP BY
     (if present) contain direct references to all key parts of
     an unique index (in whatever order) and if the key parts of the
     unique index cannot contain NULLs.
     Note that the unique keys for DISTINCT and GROUP BY should not
     be the same (as long as they are unique).

     The FROM clause must contain a single non-constant table.

     @todo Apart from the LIS test, every condition depends only on facts
     which can be known in Query_block::prepare(), possibly this block should
     move there.
  */

  JOIN_TAB *const tab = best_ref[const_tables];                       // :1503

  if (plan_is_single_table() && (!group_list.empty() || select_distinct) &&
      !tmp_table_param.sum_func_count &&
      (!tab->range_scan() ||
       tab->range_scan()->type != AccessPath::GROUP_INDEX_SKIP_SCAN)) {
    if (!group_list.empty() && rollup_state == RollupState::NONE &&
        list_contains_unique_index(tab, find_field_in_order_list,
                                   (void *)group_list.order)) {       // :1510
      /*
        We have found that grouping can be removed since groups correspond to
        only one row anyway.
      */
      group_list.clean();                                             // :1516
      grouped = false;
    }
    if (select_distinct &&
        list_contains_unique_index(tab, find_field_in_item_list, fields)) {
      select_distinct = false;
      trace_opt.add("distinct_is_on_unique", true)
          .add("removed_distinct", true);
    }
  }
  if (!(!group_list.empty() || tmp_table_param.sum_func_count || windowing) &&
      select_distinct && plan_is_single_table() &&
      rollup_state == RollupState::NONE) {                            // :1526
    int order_idx = -1, group_idx = -1;
    /*
      We are only using one table. In this case we change DISTINCT to a
      GROUP BY query if:
      - The GROUP BY can be done through indexes (no sort) and the ORDER
        BY only uses selected fields.
        (In this case we can later optimize away GROUP BY and ORDER BY)
      - We are scanning the whole table without LIMIT
        This can happen if:
        - We are using CALC_FOUND_ROWS
        - We are using an ORDER BY that can't be optimized away.
      - Selected expressions are not set functions (those cannot be put
      into GROUP BY).

      We don't want to use this optimization when we are using LIMIT
      because in this case we can just create a temporary table that
      holds LIMIT rows and stop when this table is full.
    */
    if (!order.empty()) {                                             // :1547
      skip_sort_order = test_if_skip_sort_order(   // ★ 第 1 次：ORDER BY 能否走索引
          tab, order, m_select_limit,
          true,  // no_changes
          &tab->table()->keys_in_use_for_order_by, &order_idx);
      count_field_types(query_block, &tmp_table_param, *fields, false, false);
    }
    ORDER *o;
    bool all_order_fields_used;
    if ((o = create_order_from_distinct(                              // :1556
             thd, ref_items[REF_SLICE_ACTIVE], order.order, fields,
             /*skip_aggregates=*/true,
             /*convert_bit_fields_to_long=*/true, &all_order_fields_used))) {
      group_list = ORDER_with_src(o, ESC_DISTINCT);                   // :1560
      const bool skip_group =
          skip_sort_order &&
          test_if_skip_sort_order(tab, group_list, m_select_limit,    // ★ 第 2 次
                                  true,  // no_changes
                                  &tab->table()->keys_in_use_for_group_by,
                                  &group_idx);
      count_field_types(query_block, &tmp_table_param, *fields, false, false);
      // ORDER BY and GROUP BY are using different indexes, can't skip sorting
      if (group_idx >= 0 && order_idx >= 0 && group_idx != order_idx)  // :1569
        skip_sort_order = false;
      if ((skip_group && all_order_fields_used) ||                     // :1571
          m_select_limit == HA_POS_ERROR ||
          (!order.empty() && !skip_sort_order)) {
        /*  Change DISTINCT to GROUP BY */
        select_distinct = false;
        /*
          group_list was created with ORDER BY clause as prefix and
          replaces it. So it must respect ordering. If there is no
          ORDER BY, GROUP BY need not have to provide order.
        */
        if (order.empty()) {                                           // :1581
          for (ORDER *group = group_list.order; group; group = group->next)
            group->direction = ORDER_NOT_RELEVANT;
        }
        if (all_order_fields_used && skip_sort_order && !order.empty()) {
          /*
            Force MySQL to read the table in sorted order to get result in
            ORDER BY order.
          */
          tmp_table_param.allow_group_via_temp_table = false;          // :1590
        }
        grouped = true;  // For end_write_group
        trace_opt.add("changed_distinct_to_group_by", true);
      } else
        group_list.clean();                                            // :1595 回滚
    } else if (thd->is_fatal_error())  // End of memory
      return true;
  }
  simple_group = false;                                                // :1599

  ORDER *old_group_list = group_list.order;
  group_list = ORDER_with_src(
      remove_const(group_list.order, where_cond,
                   rollup_state == RollupState::NONE, &simple_group, true),  // ★ group_by = true
      group_list.src, /*const_optimized=*/true);
  if (thd->is_error()) {
    error = 1;
    DBUG_PRINT("error", ("Error from remove_const"));
    return true;
  }
  if (old_group_list && group_list.empty()) select_distinct = false;   // :1611

  if (group_list.empty() && grouped) {                                 // :1613
    order.clean();  // The output has only one row
    simple_order = true;
    select_distinct = false;  // No need in distinct for 1 row
    group_optimized_away = true;      // ★★ 唯一置位点
  }

  calc_group_buffer(this, group_list.order);                           // :1620
  send_group_parts = tmp_table_param.group_parts; /* Save org parts */

  /*
     If ORDER BY is a prefix of GROUP BY and if windowing or ROLLUP
     doesn't change this order, ORDER BY can be removed and we can
     enforce GROUP BY to provide order.
     Also true if the result is one row.
  */
  if ((test_if_subpart(group_list.order, order.order) && !m_windows_sort &&
       query_block->olap != ROLLUP_TYPE) ||
      (group_list.empty() && tmp_table_param.sum_func_count)) {         // :1629
    if (!order.empty()) {
      order.clean();
      trace_opt.add("removed_order_by", true);
    }
    if (is_indexed_agg_distinct(this, nullptr)) streaming_aggregation = false; // :1636
  }

  return false;                                                        // :1639
}
```

**调用点**（`sql/sql_optimizer.cc:796`）：

```cpp
  error = -1; /* if goto err */                                        // :794

  if (optimize_distinct_group_order()) return true;                    // :796

  if ((query_block->active_options() & SELECT_NO_JOIN_CACHE) ||        // :798
      query_block->ftfunc_list->elements)
    no_jbuf_after = 0;

  /* Perform FULLTEXT search before all regular searches */            // :802
  if (query_block->has_ft_funcs() && optimize_fts_query()) return true; // :803
```

#### 3.1 前置：消掉 ORDER BY 里的常量项

```cpp
ORDER *org_order = order.order;
order = ORDER_with_src(
    remove_const(order.order, where_cond,
                 rollup_state == RollupState::NONE, &simple_order, false),
    order.src, /*const_optimized=*/true);
...
if (order.empty() && org_order) skip_sort_order = true;
```

`remove_const` 删掉"被 WHERE 常量化的排序项"（如 `WHERE a=5 ORDER BY a, b` → `ORDER BY b`）。注意 `ORDER_with_src` 的第二个参数 `ESC_ORDER_BY` / `const_optimized=true`——`ORDER_with_src` 给 ORDER 链表加了**溯源标签**，让 EXPLAIN 能区分"这个排序来自 ORDER BY / GROUP BY / DISTINCT"。这是 8.0 为可观测性做的改造。

若整条 ORDER BY 被消空而原来非空，说明"顺序无意义"，置 `skip_sort_order = true`。

**`ORDER_with_src` 的权威定义**（`sql/sql_optimizer.h:96`）——★ 注意 `src` 的类型是 **`Explain_sort_clause`**（定义在 `sql/opt_explain_format.h:426`），并**不存在**名为 `enum_order_src` 的枚举：

```cpp
// sql/sql_optimizer.h:89
/**
  Wrapper for ORDER* pointer to trace origins of ORDER list

  As far as ORDER is just a head object of ORDER expression
  chain, we need some wrapper object to associate flags with
  the whole ORDER list.
*/
class ORDER_with_src {
 public:
  ORDER *order;  ///< ORDER expression that we are wrapping with this class
  Explain_sort_clause src;  ///< origin of order list

 private:
  int flags;  ///< bitmap of Explain_sort_property
  // Status of const condition removal from the ORDER Expression
  bool m_const_optimized;

 public:
  ORDER_with_src() { clean(); }

  ORDER_with_src(ORDER *order_arg, Explain_sort_clause src_arg,
                 bool const_optimized_arg = false)
      : order(order_arg),
        src(src_arg),
        flags(order_arg ? ESP_EXISTS : ESP_none),
        m_const_optimized(const_optimized_arg) {}

  bool empty() const { return order == nullptr; }

  void clean() {      // ★ clean() 把 order/src/flags/m_const_optimized 一次全清
    order = nullptr;
    src = ESC_none;
    flags = ESP_none;
    m_const_optimized = false;
  }

  int get_flags() const {
    assert(order);
    return flags;
  }

  bool is_const_optimized() const { return m_const_optimized; }
};
```

```cpp
// sql/opt_explain_format.h:426
/**
  Enumeration of ORDER BY, GROUP BY and DISTINCT clauses for array indexing
  See Explain_format_flags::sorts
*/
enum Explain_sort_clause {
  ESC_none = 0,
  ESC_ORDER_BY = 1,
  ESC_GROUP_BY = 2,
  ESC_DISTINCT = 3,
  ESC_BUFFER_RESULT = 4,
  ESC_WINDOWING = 5,
  //-----------------
  ESC_MAX
};

/**
  Bit flags to explain GROUP BY, ORDER BY and DISTINCT clauses
*/
enum Explain_sort_property {
  ESP_none = 0,
  ESP_EXISTS = 1 << 0,        ///< Original query has this clause
  ESP_IS_SIMPLE = 1 << 1,     ///< Clause is effective for single JOIN_TAB only
  ESP_USING_FILESORT = 1 << 2, ///< Clause causes a filesort
  ESP_USING_TMPTABLE = 1 << 3, ///< Clause creates an intermediate table
  ESP_DUPS_REMOVAL = 1 << 4    ///< Duplicate removal for DISTINCT
};
```

初始来源在 `JOIN` 构造时打上（`sql_optimizer.cc:181`）：

```cpp
      order(select->order_list.first, ESC_ORDER_BY),                   // :181
      group_list(select->group_list.first, ESC_GROUP_BY),              // :182
  ...
  if (select->order_list.first) explain_flags.set(ESC_ORDER_BY, ESP_EXISTS);   // :195
  if (select->group_list.first) explain_flags.set(ESC_GROUP_BY, ESP_EXISTS);   // :196
  if (select->is_distinct()) explain_flags.set(ESC_DISTINCT, ESP_EXISTS);      // :197
```

> ★ 这就是为什么 `add_sorting_to_table()` 里那句 `explain_flags.set(sort_order->src, ESP_USING_FILESORT)` 能把 filesort **正确归因**到 ORDER BY / GROUP BY / DISTINCT 上——`src` 一路被 `ORDER_with_src` 带着走。

#### 3.2 单表 + 无聚合：整组消掉 GROUP BY

```cpp
if (plan_is_single_table() && (!group_list.empty() || select_distinct) &&
    !tmp_table_param.sum_func_count &&
    (!tab->range_scan() ||
     tab->range_scan()->type != AccessPath::GROUP_INDEX_SKIP_SCAN)) {
  if (!group_list.empty() && rollup_state == RollupState::NONE &&
      list_contains_unique_index(tab, find_field_in_order_list, group_list.order)) {
    group_list.clean();
    grouped = false;
  }
  if (select_distinct &&
      list_contains_unique_index(tab, find_field_in_item_list, fields)) {
    select_distinct = false;
  }
}
```

判据链条：

1. **必须是单表**（多表的"唯一索引"不构成全局唯一）；
2. **不能有聚合函数**（有聚合时分组是必需的，即使分组键唯一——因为要把多行聚成一行）；
3. **不能已经用了 `GROUP_INDEX_SKIP_SCAN`**（LIS/MIN-MAX 已经预计算好了，再消会丢失语义）；
4. `list_contains_unique_index`：**分组/去重的所有列合起来覆盖了某个唯一索引的全部 keypart，且这些 keypart 都 NOT NULL**。

第 4 条的"不可为 NULL"很关键：唯一索引允许多个 NULL，若列可空，两行 `(NULL)` 与 `(NULL)` 在唯一索引里不冲突，但在 GROUP BY 语义里是同一组。

被这个条件消掉后，`grouped = false`，`make_tmp_tables_info` 就不再建分组临时表。

**`list_contains_unique_index()` 完整源码**（`sql_optimizer.cc:10606`，★ 是 `static`，仅本文件可见；前向声明在 `:128`）：

```cpp
// sql/sql_optimizer.cc:10572（doxygen 摘录）
  Used in removing the GROUP BY/DISTINCT of the following types of
  statements:
  @code
    SELECT [DISTINCT] <unique_key_cols>... FROM <single_table_ref>
      [GROUP BY <unique_key_cols>,...]
  @endcode

    If (a,b,c is distinct)
    then <any combination of a,b,c>,{whatever} is also distinct

    This function checks if all the key parts of any of the unique keys
    of the table are referenced by a list : either the select list
    through find_field_in_item_list or GROUP BY list through
    find_field_in_order_list.
    If the above holds and the key parts cannot contain NULLs then we
    can safely remove the GROUP BY/DISTINCT,
    as no result set can be more distinct than an unique key.
```

```cpp
static bool list_contains_unique_index(JOIN_TAB *tab,
                                       bool (*find_func)(Field *, void *),
                                       void *data) {
  TABLE *table = tab->table();

  if (tab->is_inner_table_of_outer_join()) return false;   // ★ 外连接被驱动表直接放弃
  for (uint keynr = 0; keynr < table->s->keys; keynr++) {
    if (keynr == table->s->primary_key ||                  // ★ 主键
        (table->key_info[keynr].flags & HA_NOSAME)) {      // ★ 或 UNIQUE 索引
      KEY *keyinfo = table->key_info + keynr;
      KEY_PART_INFO *key_part, *key_part_end;

      for (key_part = keyinfo->key_part,
          key_part_end = key_part + keyinfo->user_defined_key_parts;
           key_part < key_part_end; key_part++) {
        // ★★ 每个 keypart 都必须 NOT NULL 且能在列表里找到
        if (key_part->field->is_nullable() || !find_func(key_part->field, data))
          break;
      }
      if (key_part == key_part_end) return true;           // ★ 完整走到尾 → 命中
    }
  }
  return false;
}

// sql/sql_optimizer.cc:10644 —— 在 ORDER 链里找
static bool find_field_in_order_list(Field *field, void *data) {
  ORDER *group = (ORDER *)data;
  bool part_found = false;
  for (ORDER *tmp_group = group; tmp_group; tmp_group = tmp_group->next) {
    const Item *item = (*tmp_group->item)->real_item();
    if (item->type() == Item::FIELD_ITEM &&
        down_cast<const Item_field *>(item)->field->eq(field)) {
      part_found = true;
      break;
    }
  }
  return part_found;
}

// sql/sql_optimizer.cc:10672 —— 在 SELECT list 里找（只看可见字段）
static bool find_field_in_item_list(Field *field, void *data) {
  mem_root_deque<Item *> *fields =
      reinterpret_cast<mem_root_deque<Item *> *>(data);
  bool part_found = false;

  for (const Item *item : VisibleFields(*fields)) {
    if (item->type() == Item::FIELD_ITEM &&
        down_cast<const Item_field *>(item)->field->eq(field)) {
      part_found = true;
      break;
    }
  }
  return part_found;
}
```

> ★ **最常见的"我有 unique key 但 GROUP BY 没被消掉"原因**：①索引不是 PK 也没有 `HA_NOSAME`；②keypart 里有可空列；③`find_field_in_*` 只认 **`Item::FIELD_ITEM` 且 `field->eq()` 成立**——表达式、函数、视图列引用都不算。

#### 3.3 DISTINCT → GROUP BY 改写

这是最复杂的一段，源码注释本身就写了动机：

```
We are only using one table. In this case we change DISTINCT to a
GROUP BY query if:
- The GROUP BY can be done through indexes (no sort) and the ORDER
  BY only uses selected fields.
- We are scanning the whole table without LIMIT
  (CALC_FOUND_ROWS / ORDER BY 无法被优化掉的情况)
- Selected expressions are not set functions

We don't want to use this optimization when we are using LIMIT
because in this case we can just create a temporary table that
holds LIMIT rows and stop when this table is full.
```

**"有 LIMIT 时不改写"** 是这段最反直觉的一点：有 LIMIT 时，临时表只要装 LIMIT 行就停，比"全表扫 + 分组"便宜。这是一个**隐含的代价判断**，写成了规则。

改写的执行步骤：

```
1. 若有 ORDER BY：先试 test_if_skip_sort_order(ORDER BY) → 得到 order_idx
2. create_order_from_distinct()：把 SELECT list 构造成一条 ORDER 链
   （skip_aggregates=true，聚合函数不能进 GROUP BY）
   → 得到 group_list（src = ESC_DISTINCT）
3. 试 test_if_skip_sort_order(改写后的 GROUP BY) → 得到 group_idx、skip_group
4. 若 group_idx >= 0 && order_idx >= 0 && group_idx != order_idx
   → skip_sort_order = false
   ★ ORDER BY 和 GROUP BY 要用不同的索引，则无法同时免排序
5. 满足 (skip_group && all_order_fields_used)
      || m_select_limit == HA_POS_ERROR   （无 LIMIT）
      || (!order.empty() && !skip_sort_order)
   → 真正改写：select_distinct = false; grouped = true
6. 若原来没有 ORDER BY：把 GROUP BY 各项的 direction 置为 ORDER_NOT_RELEVANT
   ★ 没有 ORDER BY 时，"改写出来的 GROUP BY"不需要提供顺序
7. 若 all_order_fields_used && skip_sort_order && 有 ORDER BY
   → tmp_table_param.allow_group_via_temp_table = false
   ★ 必须按索引顺序流式输出，禁止临时表聚合
8. 否则 group_list.clean()（改写白做，回滚）
```

第 6、7 两步是这段的精髓：**改写出来的 GROUP BY 是"为了去重"还是"为了提供顺序"，决定了它的 `direction` 和是否允许临时表**。同一个 `group_list`，因为来源不同（用户写的 vs DISTINCT 改写的），语义要求不同。

**`create_order_from_distinct()` 完整源码**（`sql_optimizer.cc:10687`；★ 它**没有** doxygen 注释块）：

```cpp
ORDER *create_order_from_distinct(THD *thd, Ref_item_array ref_item_array,
                                  ORDER *order_list,
                                  mem_root_deque<Item *> *fields,
                                  bool skip_aggregates,
                                  bool convert_bit_fields_to_long,
                                  bool *all_order_by_fields_used) {
  ORDER *group = nullptr, **prev = &group;

  *all_order_by_fields_used = true;

  // ★ 第一段：ORDER BY 中"已在 SELECT list 里"的项，作为 GROUP BY 的前缀
  for (ORDER *order = order_list; order; order = order->next) {
    if (order->in_field_list) {
      ORDER *ord = (ORDER *)thd->memdup((char *)order, sizeof(ORDER));
      if (!ord) return nullptr;
      *prev = ord;
      prev = &ord->next;
      (*ord->item)->marker = Item::MARKER_DISTINCT_GROUP;   // ★ 打标，防第二段重复加入
    } else
      *all_order_by_fields_used = false;                    // ★ ORDER BY 有不在 select list 的项
  }

  Mem_root_array<std::pair<Item *, ORDER *>> bit_fields_to_add(thd->mem_root);

  // ★ 第二段：把 SELECT list 的可见列补进 GROUP BY
  for (Item *&item : VisibleFields(*fields)) {
    if (!item->const_item() && (!skip_aggregates || !item->has_aggregation()) &&
        item->marker != Item::MARKER_DISTINCT_GROUP) {
      /*
        Don't put duplicate columns from the SELECT list into the
        GROUP BY list.
      */
      ORDER *ord_iter;
      for (ord_iter = group; ord_iter; ord_iter = ord_iter->next)
        if ((*ord_iter->item)->eq(item, true)) goto next_item;

      ORDER *ord = (ORDER *)thd->mem_calloc(sizeof(ORDER));
      if (!ord) return nullptr;

      if (item->type() == Item::FIELD_ITEM &&
          item->data_type() == MYSQL_TYPE_BIT && convert_bit_fields_to_long) {
        /*
          Because HEAP tables can't index BIT fields we need to use an
          additional hidden field for grouping because later it will be
          converted to a LONG field. Original field will remain of the
          BIT type and will be returned to a client.
          @note setup_ref_array() needs to account for the extra space.
          @note We need to defer the actual adding to after the loop,
            or we will invalidate the iterator to “fields”.
        */
        Item_field *new_item = new Item_field(thd, (Item_field *)item);
        ord->item = &item;  // Temporary; for the duplicate check above.
        bit_fields_to_add.push_back(std::make_pair(new_item, ord));
      } else if (ref_item_array.is_null()) {
        // No slices are in use, so just use the field from the list.
        ord->item = &item;
      } else {
        /*
          We have here only visible fields, so we can use simple indexing
          of ref_item_array (order in the array and in the list are same)
        */
        ord->item = &ref_item_array[0];
      }
      ord->direction = ORDER_ASC;
      *prev = ord;
      prev = &ord->next;
    }
  next_item:
    if (!ref_item_array.is_null()) {
      ref_item_array.pop_front();
    }
  }
  for (const auto &item_and_order : bit_fields_to_add) {
    item_and_order.second->item =
        thd->lex->current_query_block()->add_hidden_item(item_and_order.first);
    thd->lex->current_query_block()->hidden_items_from_optimization++;
  }
  *prev = nullptr;
  return group;
}
```

> ★ `all_order_fields_used` 的语义：**ORDER BY 的每一项是否都出现在 SELECT list 中**。只有它为 true，改写出的 GROUP BY 才可能顺带满足原 ORDER BY 的排序语义——这就是 `:1571`（决定是否改写）与 `:1585`（决定是否禁临时表）都要拿它当前置条件的原因。
>
> ★ `skip_aggregates=true` 时聚合函数**不进** GROUP BY——因为 `AGG(DISTINCT x)` 这类表达式本身不是分组键。

#### 3.4 消掉 GROUP BY 里的常量项，以及"整组消失"

```cpp
simple_group = false;
ORDER *old_group_list = group_list.order;
group_list = ORDER_with_src(
    remove_const(group_list.order, where_cond,
                 rollup_state == RollupState::NONE, &simple_group, true),
    group_list.src, /*const_optimized=*/true);
...
if (old_group_list && group_list.empty()) select_distinct = false;
if (group_list.empty() && grouped) {
  order.clean();
  simple_order = true;
  select_distinct = false;
  group_optimized_away = true;
}
```

若 GROUP BY 被消空但 `grouped` 仍为真（例如 `GROUP BY` 存在但有聚合函数、`SELECT COUNT(*) FROM t GROUP BY pk`），说明**结果只有一行**：

- ORDER BY 被删（一行无需排序）
- DISTINCT 被删（一行无需去重）
- `group_optimized_away = true`

**`JOIN::remove_const()` 主体源码**（`sql_optimizer.cc:10175`；只省略了 trace 字符串拼接与子查询求值两处细节，判定分支全部保留）——★ 它是 `JOIN` 的**成员函数**，不在 `sql_select.cc`（那里只有给 UPDATE/DELETE 用的 `simple_remove_const()`）；最后一个参数真名是 **`group_by`**，不是"是否保留顺序语义"，也不是 5.x 时代的 `split_sum_funcs`：

```cpp
// sql/sql_optimizer.cc:10158（doxygen）
/**
  Remove all constants and check if ORDER only contains simple
  expressions.

  simple_order is set to true if sort_order only uses fields from head table
  and the head table is not a LEFT JOIN table.

  @param first_order   List of GROUP BY or ORDER BY sub-clauses.
  @param cond          WHERE condition.
  @param change        If true, remove sub-clauses that need not be evaluated.
                       If this is not set, then only simple_order is calculated.
  @param[out] simple_order  Set to true if we are only using simple expressions.
  @param group_by      True if first_order represents a grouping operation.

  @returns new sort order, after const elimination (when change is true).
*/
```

```cpp
// sql/sql_optimizer.cc:10175
ORDER *JOIN::remove_const(ORDER *first_order, Item *cond, bool change,
                          bool *simple_order, bool group_by) {
  DBUG_TRACE;

  ASSERT_BEST_REF_IN_JOIN_ORDER(this);

  if (plan_is_const())
    return change ? nullptr : first_order;  // No need to sort

  Opt_trace_context *const trace = &thd->opt_trace;
  Opt_trace_disable_I_S trace_disabled(trace, first_order == nullptr);
  Opt_trace_object trace_simpl(
      trace, group_by ? "simplifying_group_by" : "simplifying_order_by");  // ★ group_by 效果①
  ...
  ORDER **prev_ptr = &first_order;
  *simple_order = !first_tab->join_cond();

  // De-optimization in conjunction with window functions
  if (group_by && m_windows.elements > 0) *simple_order = false;           // ★ group_by 效果②

  update_depend_map(first_order);

  for (ORDER *order = first_order; order; order = order->next) {
    Opt_trace_object trace_one_item(trace);
    trace_one_item.add("item", order->item[0]);
    table_map order_tables = order->item[0]->used_tables();

    if (order->item[0]->has_aggregation() || order->item[0]->has_wf() ||
        /*
          If the outer table of an outer join is const (either by itself or
          after applying WHERE condition), grouping on a field from such a
          table will be optimized away and filesort without temporary table
          will be used unless we prevent that now. Filesort is not fit to
          handle joins and the join condition is not applied. We can't detect
          the case without an expensive test, however, so we force temporary
          table for all queries containing more than one table, ROLLUP, and an
          outer join.
         */
        (primary_tables > 1 && rollup_state == RollupState::INITED &&
         query_block->outer_join)) {
      *simple_order = false;  // Must use a temporary table to sort
    } else if ((order_tables & not_const_tables) == 0 &&
               evaluate_during_optimization(order->item[0], query_block)) {
      if (order->item[0]->has_subquery()) { ... }
      trace_one_item.add("uses_only_constant_tables", true);
      continue;                                    // ★ 常量项被摘掉（不写回 prev_ptr）
    } else if (duplicate_order(first_order, order)) {
      /*
        If 'order' is a duplicate of an expression earlier in the
        ORDER/GROUP BY sequence, it can be removed from the ORDER BY
        or GROUP BY clause.
      */
      trace_one_item.add("duplicate_item", true);
      continue;
    } else if (order->in_field_list && order->item[0]->has_subquery()) {
      /*
        If the order item is a subquery that is also in the field
        list, a temp table should be used to avoid evaluating the
        subquery for each row both when a) creating a sort index and
        b) getting the value.
          Example: "SELECT (SELECT ... ) as a ... GROUP BY a;"
       */
      *simple_order = false;
    } else if (order_tables & (RAND_TABLE_BIT | OUTER_REF_TABLE_BIT)) {
      *simple_order = false;
    } else {
      if (cond != nullptr && check_field_is_const(cond, order->item[0])) {
        trace_one_item.add("equals_constant_in_where", true);   // ★ WHERE 里被常量化
        continue;
      }
      if ((ref = order_tables & (not_const_tables ^ first_table))) {
        if (!(order_tables & first_table) &&
            only_eq_ref_tables(this, first_order, ref, &cached_eq_ref_tables,
                               &eq_ref_tables)) {
          trace_one_item.add("eq_ref_to_preceding_items", true);
          continue;
        }
        *simple_order = false;  // Must do a temp table to sort
      }
    }
    if (change) *prev_ptr = order;  // use this entry
    prev_ptr = &order->next;
  }
  if (change) *prev_ptr = nullptr;
  if (prev_ptr == &first_order)  // Nothing to sort/group
    *simple_order = true;
  DBUG_PRINT("exit", ("simple_order: %d", (int)*simple_order));

  trace_each_item.end();
  trace_simpl.add("resulting_clause_is_simple", *simple_order);
  ...
  return first_order;
}
```

> ⚠️ **ROLLUP 下 `change` 恒为 false**：`optimize_rollup()` 在 `sql_optimizer.cc:373` 就把 `rollup_state` 置成 `INITED`，而两处调用传的是 `rollup_state == RollupState::NONE`。所以 ROLLUP 查询里 `remove_const` **只算 `simple_order`/`simple_group`、不改链表**——这是"ROLLUP 下 GROUP BY 项删不掉"的源码原因。

#### 3.5 GROUP BY 是 ORDER BY 的前缀 → 删掉 ORDER BY

```cpp
if ((test_if_subpart(group_list.order, order.order) && !m_windows_sort &&
     query_block->olap != ROLLUP_TYPE) ||
    (group_list.empty() && tmp_table_param.sum_func_count)) {
  if (!order.empty()) {
    order.clean();
    trace_opt.add("removed_order_by", true);
  }
  if (is_indexed_agg_distinct(this, nullptr)) streaming_aggregation = false;
}
```

`test_if_subpart` 判"ORDER BY 是 GROUP BY 的子序列"。两个排除条件：

- `m_windows_sort`：窗口函数会重排，GROUP BY 提供的顺序不能保证传到 ORDER BY；
- `query_block->olap == ROLLUP_TYPE`：ROLLUP 会插入汇总行，破坏顺序。

最后一行容易被忽略：`is_indexed_agg_distinct` 为真（即已经用了 LIS 这类"索引聚合去重"）时，把 `streaming_aggregation` 关掉——因为 LIS 已经完成了分组与去重，再叠加流式聚合是重复劳动。

**`test_if_subpart()` 完整源码**（`sql/sql_select.cc:3975`，★ 不在 `sql_optimizer.cc`；声明在 `sql_select.h:875`）：

```cpp
// sql/sql_select.cc:3968
/**
  Return 1 if second is a subpart of first argument.

  If first parts has different direction, change it to second part
  (group is sorted like order)
*/
bool test_if_subpart(ORDER *a, ORDER *b) {
  ORDER *first = a;
  ORDER *second = b;
  for (; first && second; first = first->next, second = second->next) {
    if ((*first->item)->eq(*second->item, true))
      continue;
    else
      return false;
  }
  // If the second argument is not subpart of the first return false
  if (second) return false;                       // ★ second 还有剩 → b 不是 a 的前缀
  // Else assign the direction of the second argument to the first
  for (; a && b; a = a->next, b = b->next) a->direction = b->direction;  // ★ 副作用
  return true;
}
```

> ★ 最后一行的**副作用**很关键：判定成功后会把 ORDER BY 的 `direction` 抄给 GROUP BY（注释："group is sorted like order"）。所以"GROUP BY 顺带提供顺序"这个动作本身就在 `test_if_subpart` 里完成，不额外写代码。

**`is_indexed_agg_distinct()` 完整源码**（`sql_optimizer.cc:8042`）：

```cpp
// sql/sql_optimizer.cc:8020（doxygen 摘录）
  Check for every COUNT(DISTINCT), AVG(DISTINCT) or
  SUM(DISTINCT). These can be resolved by Loose Index Scan as long
  as all the aggregate distinct functions refer to the same
  fields. Thus:

  SELECT AGGFN(DISTINCT a, b), AGGFN(DISTINCT b, a)... => can use LIS
  SELECT AGGFN(DISTINCT a),    AGGFN(DISTINCT a)   ... => can use LIS
  SELECT AGGFN(DISTINCT a, b), AGGFN(DISTINCT a)   ... => cannot use LIS
  SELECT AGGFN(DISTINCT a),    AGGFN(DISTINCT b)   ... => cannot use LIS
  etc.
```

```cpp
bool is_indexed_agg_distinct(JOIN *join,
                             mem_root_deque<Item_field *> *out_args) {
  Item_sum **sum_item_ptr;
  bool result = false;
  Field_map first_aggdistinct_fields;

  if (join->primary_tables > 1 ||             /* reference more than 1 table */
      join->select_distinct ||                /* or a DISTINCT */
      join->query_block->olap == ROLLUP_TYPE) /* Check (B3) for ROLLUP */
    return false;

  if (join->make_sum_func_list(*join->fields, true)) return false;

  for (sum_item_ptr = join->sum_funcs; *sum_item_ptr; sum_item_ptr++) {
    Item_sum *sum_item = *sum_item_ptr;
    Field_map cur_aggdistinct_fields;
    Item *expr;
    /* aggregate is not AGGFN(DISTINCT) or more than 1 argument to it */
    switch (sum_item->sum_func()) {
      case Item_sum::MIN_FUNC:
      case Item_sum::MAX_FUNC:
        continue;                              // ★ MIN/MAX 不参与 LIS 判定，直接跳过
      case Item_sum::COUNT_DISTINCT_FUNC:
        break;
      case Item_sum::AVG_DISTINCT_FUNC:
      case Item_sum::SUM_DISTINCT_FUNC:
        if (sum_item->argument_count() == 1) break;
        [[fallthrough]];
      default:
        return false;
    }

    for (uint i = 0; i < sum_item->argument_count(); i++) {
      expr = sum_item->get_arg(i);
      /* The AGGFN(DISTINCT) arg is not an attribute? */
      if (expr->real_item()->type() != Item::FIELD_ITEM) return false;

      Item_field *item = static_cast<Item_field *>(expr->real_item());
      if (out_args) out_args->push_back(item);

      cur_aggdistinct_fields.set_bit(item->field->field_index());
      result = true;
    }
    /*
      If there are multiple aggregate functions, make sure that they all
      refer to exactly the same set of columns.
    */
    if (first_aggdistinct_fields.is_clear_all())
      first_aggdistinct_fields.merge(cur_aggdistinct_fields);
    else if (first_aggdistinct_fields != cur_aggdistinct_fields)
      return false;                            // ★ 列集合不同 → 不能共用 LIS
  }

  return result;
}
```

---

### 四、test_skip_sort：能否用索引做 GROUP BY

> 位于 `sql/sql_optimizer.cc`，紧跟 `optimize_distinct_group_order` 之后。

#### 4.1 GROUP BY 优先于 ORDER BY

```cpp
if (!group_list.empty())                     // GROUP BY honoured first
{                                            // (DISTINCT 若可跳过已被改写成 GROUP BY)
    if (!(query_block->active_options() & SELECT_BIG_RESULT || with_json_agg) ||
        (tab->range_scan() && tab->range_scan()->type == GROUP_INDEX_SKIP_SCAN) ||
        contains_non_aggregated_fts()) {
      if (simple_group && !select_distinct) {
        const ha_rows limit = (need_tmp_before_win ? HA_POS_ERROR : m_select_limit);
        if (test_if_skip_sort_order(tab, group_list, limit, false,
                                    &tab->table()->keys_in_use_for_group_by, &dummy)) {
          m_ordered_index_usage = ORDERED_INDEX_GROUP_BY;
        }
      }
      ...
    }
}
else if (!order.empty() && (simple_order || skip_sort_order) && !m_windows_sort) {
    if ((skip_sort_order = test_if_skip_sort_order(tab, order, m_select_limit, false,
             &tab->table()->keys_in_use_for_order_by, &dummy)))
      m_ordered_index_usage = ORDERED_INDEX_ORDER_BY;
}
```

**这是一个 if / else if：GROUP BY 和 ORDER BY 只有一个能"用索引"。** 这与 PG 的 interesting order 思路不同——MySQL 不为同一个表集合保留多个候选顺序，`m_ordered_index_usage` 是三态枚举（`VOID` / `GROUP_BY` / `ORDER_BY`），只能选一个。

#### 4.2 SQL_BIG_RESULT 的反向逻辑

`!(SELECT_BIG_RESULT || with_json_agg) || (是 GROUP_INDEX_SKIP_SCAN) || (有非聚合的 FTS)` 这个判定式读起来别扭，展开语义是：

| 条件 | 能否尝试索引做 GROUP BY |
|---|---|
| 无 `SQL_BIG_RESULT`、无 JSON 聚合 | ✅ 尝试 |
| 有 `SQL_BIG_RESULT` 或 JSON 聚合 | ❌ 不尝试（**这就是开场那个反直觉例子的根因**） |
| 但已经选了 `GROUP_INDEX_SKIP_SCAN` | ✅ 仍然尝试（LIS 本身就是索引扫描，不该被禁用） |
| 或有非聚合的全文检索结果要在聚合后访问 | ✅ 尝试 |

`SQL_BIG_RESULT` 的**意图**是"别用临时表"，但实现上是"别用索引做 GROUP BY"——因为 MySQL 的 GROUP BY 只有两条免排序路径：索引有序、临时表唯一键。禁掉临时表后，若又不许用索引，就只剩 filesort。源码注释也承认这里别扭（后面那句 `TODO: Explain the allow_group_via_temp_table part of the test below`）。

#### 4.3 LooseScan 冲突保护

```cpp
if ((m_ordered_index_usage != ORDERED_INDEX_GROUP_BY) &&
    (tmp_table_param.allow_group_via_temp_table ||
     (tab->emb_sj_nest && tab->position()->sj_strategy == SJ_OPT_LOOSE_SCAN))) {
  need_tmp_before_win = true;
  simple_order = simple_group = false;       // Force tmp table without sort
}
```

若 GROUP BY 拿不到索引顺序，而表又处于 semi-join 的 **LooseScan** 策略中，必须强制走临时表。理由写在注释里：LooseScan 依赖"按索引顺序扫描并跳过重复"，若把排序压到 LooseScan 的表上会破坏它的去重语义。这是一个**跨优化的耦合**——semi-join 策略（见 [`logical/03_semijoin.md`](logical/03_semijoin.md)）会反过来约束 GROUP BY 的实现选择。

#### 4.4 输出：m_ordered_index_usage 三态

```
ORDERED_INDEX_VOID      → 没有索引能提供顺序
ORDERED_INDEX_GROUP_BY  → 索引顺序用于 GROUP BY
ORDERED_INDEX_ORDER_BY  → 索引顺序用于 ORDER BY
```

这个枚举的**消费者**是 `make_tmp_tables_info`——它在那里决定"还要不要加 filesort"。

#### 4.5 `test_skip_sort()` 完整源码（`sql_optimizer.cc:1642`）

```cpp
void JOIN::test_skip_sort() {                                         // :1642
  DBUG_TRACE;
  ASSERT_BEST_REF_IN_JOIN_ORDER(this);
  JOIN_TAB *const tab = best_ref[const_tables];

  assert(m_ordered_index_usage == ORDERED_INDEX_VOID);                 // :1647

  if (!group_list.empty())  // GROUP BY honoured first                 // :1649
                            // (DISTINCT was rewritten to GROUP BY if skippable)
  {
    /*
      If the SQL_BIG_RESULT option is set on the query block or a JSON
      aggregation function is used, check if it is possible to sort using index
      for GROUP BY and thus avoid materializing the row set to disk, unless:

      1. A group min-max optimization will be used, or
      2. Some non-aggregated full-text search results must be accessible after
         aggregation.
    */
    if (!(query_block->active_options() & SELECT_BIG_RESULT || with_json_agg) ||  // :1661
        (tab->range_scan() &&
         tab->range_scan()->type == AccessPath::GROUP_INDEX_SKIP_SCAN) ||
        contains_non_aggregated_fts()) {
      if (simple_group &&    // GROUP BY is possibly skippable
          !select_distinct)  // .. if not preceded by a DISTINCT
      {
        /*
          Calculate a possible 'limit' of table rows for 'GROUP BY':
          A specified 'LIMIT' is relative to the final resultset.
          'need_tmp' implies that there will be more postprocessing
          so the specified 'limit' should not be enforced yet.
         */
        const ha_rows limit =                                          // :1674
            (need_tmp_before_win ? HA_POS_ERROR : m_select_limit);
        int dummy;

        if (test_if_skip_sort_order(tab, group_list, limit, false,     // :1678
                                    &tab->table()->keys_in_use_for_group_by,
                                    &dummy)) {
          m_ordered_index_usage = ORDERED_INDEX_GROUP_BY;              // :1681
        }
      }

      /*
        If we are going to use semi-join LooseScan, it will depend
        on the selected index scan to be used.  If index is not used
        for the GROUP BY, we risk that sorting is put on the LooseScan
        table.  In order to avoid this, force use of temporary table.
        TODO: Explain the allow_group_via_temp_table part of the test below.
       */
      // ★★ TODO 的确切位置：:1690，紧跟其后的 if 就是它说的 "the test below"
      if ((m_ordered_index_usage != ORDERED_INDEX_GROUP_BY) &&
          (tmp_table_param.allow_group_via_temp_table ||
           (tab->emb_sj_nest &&
            tab->position()->sj_strategy == SJ_OPT_LOOSE_SCAN))) {     // :1692
        need_tmp_before_win = true;
        simple_order = simple_group = false;  // Force tmp table without sort  // :1697
      }
    }
  } else if (!order.empty() &&  // ORDER BY wo/ preceding GROUP BY    // :1700
             (simple_order ||
              skip_sort_order) &&  // which is possibly skippable,
             !m_windows_sort)      // and WFs will not shuffle rows
  {
    int dummy;
    if ((skip_sort_order = test_if_skip_sort_order(
             tab, order, m_select_limit, false,
             &tab->table()->keys_in_use_for_order_by, &dummy))) {
      m_ordered_index_usage = ORDERED_INDEX_ORDER_BY;                  // :1709
      /*
        Update plan cost if there is only one table. Multi-table/join scenarios
        are more complex and will not reflect updated costs after access change.
      */
      if (primary_tables == 1 && tab->table()->s->has_secondary_engine()) {
        best_read = qep_tab->position()->prefix_cost + sort_cost;
      }
    }
  }
}
```

**三态枚举的定义**（`sql/sql_optimizer.h:395`，★ 是 `class JOIN` 内的**匿名枚举成员**，不是文件级 enum）：

```cpp
  /*
    simple_xxxxx is set if ORDER/GROUP BY doesn't include any references
    to other tables than the first non-constant table in the JOIN.
    It's also set if ORDER/GROUP BY is empty.
    Used for deciding for or against using a temporary table to compute
    GROUP/ORDER BY.
  */
  bool simple_order{false};                                            // :387
  bool simple_group{false};                                            // :388

  /*
    m_ordered_index_usage is set if an ordered index access
    should be used instead of a filesort when computing
    ORDER/GROUP BY.
  */
  enum {
    ORDERED_INDEX_VOID,      // No ordered index avail.                 // :396
    ORDERED_INDEX_GROUP_BY,  // Use index for GROUP BY                  // :397
    ORDERED_INDEX_ORDER_BY   // Use index for ORDER BY                  // :398
  } m_ordered_index_usage{ORDERED_INDEX_VOID};                          // :399
```

> ⚠️ **易错点**：`:1661` 的判定式是 `if (!(A || B) || C || D)`——`A` = 有 `SELECT_BIG_RESULT`、`B` = 有 JSON 聚合。**整体取反**意味着「既非 BIG_RESULT 又无 JSON agg」才进内层；命中 `GROUP_INDEX_SKIP_SCAN`（C）或 `contains_non_aggregated_fts()`（D）也能进。但反过来，**若是 BIG_RESULT 且既非 C 也非 D，整个 GROUP BY 分支什么都不做——连 LooseScan 保护都不执行**。

`need_tmp_before_win` 的**全部置位点**（4 处，都在 `sql_optimizer.cc`，`make_tmp_tables_info` 只消费不置位）：

```cpp
// sql/sql_optimizer.cc:975
  if (rollup_state != RollupState::NONE &&  // (1)
      (select_distinct || has_windows || !order.empty()))
    need_tmp_before_win = true;                                    // :977

  /*
    If we have full-text columns involved in aggregation, we may need to
    materialize them. Materialization is needed if the result of a full-text
    search (the MATCH function) is accessed after aggregation, as the saving and
    loading of rows in AggregateIterator does not include FTS information. ...
   */
  if (!need_tmp_before_win && implicit_grouping &&
      contains_non_aggregated_fts()) {
    need_tmp_before_win = true;                                    // :990
  }

  if (!plan_is_const())
  {
    if ((!group_list.empty() && !simple_group) ||                       // (3)
        (!has_windows && (select_distinct ||                            // (4)
                          (!order.empty() && !simple_order) ||          // (5)
                          (!group_list.empty() && !order.empty()))) ||  // (6)
        ((query_block->active_options() & OPTION_BUFFER_RESULT) &&
         !has_windows &&
         !(query_expression()->derived_table &&
           query_expression()
               ->derived_table->uses_materialization())) ||     // (7)
        (has_windows && (primary_tables - const_tables) > 1 &&  // (8)
         m_windows[0]->needs_sorting() && !group_optimized_away))
      need_tmp_before_win = true;                                    // :1006
  }
```

以及 `optimize_rollup()` 的**全部 5 行**（`sql_optimizer.cc:11347`）：

```cpp
bool JOIN::optimize_rollup() {                                         // :11347
  tmp_table_param.allow_group_via_temp_table = false;                  // ★ ROLLUP 一律禁用临时表分组
  rollup_state = RollupState::INITED;
  tmp_table_param.group_parts = send_group_parts;
  return false;
}
```

---

### 五、四种 GROUP BY 实现的判定链

综合第三、四、六章的标志，最终落到的实现只有四种：

| # | 实现 | 触发条件 | 输出顺序 | 迭代器 |
|---|---|---|---|---|
| 1 | **索引有序流式聚合** | `m_ordered_index_usage == ORDERED_INDEX_GROUP_BY`（索引顺序满足分组键） | 分组键有序 | 表扫描/索引扫描 + `AggregateIterator` |
| 2 | **松索引扫描（LIS / MIN-MAX）** | range 优化器产出 `GROUP_INDEX_SKIP_SCAN` | 分组键有序 | `GroupIndexSkipScanIterator` |
| 3 | **临时表聚合** | `allow_group_via_temp_table == true` 且拿不到索引顺序 | 哈希序 | `MaterializeIterator` + `TemptableAggregateIterator` |
| 4 | **先 filesort 再流式聚合** | `allow_group_via_temp_table == false`（`SQL_BIG_RESULT` / rollup / 部分 UDF 聚合）且拿不到索引顺序 | 分组键有序 | `SortingIterator` + `AggregateIterator` |

判定顺序（ASCII）：

```
                 GROUP BY 存在？
                        │ no → 走 DISTINCT / ORDER BY 分支
                        ▼ yes
        range 优化器是否产出 GROUP_INDEX_SKIP_SCAN？
                        │ yes ────────────────► ② 松索引扫描（LIS / MIN-MAX）
                        ▼ no
        optimize_distinct_group_order：分组键覆盖唯一非空索引？
                        │ yes ────────────────► GROUP BY 整组消掉（无分组）
                        ▼ no
        test_skip_sort：索引顺序能否满足 GROUP BY？
                        │ yes ────────────────► ① 索引有序流式聚合
                        ▼ no
        allow_group_via_temp_table？
              ┌─────────┴─────────┐
            true                false
              ▼                   ▼
        ③ 临时表聚合        ④ filesort + 流式聚合
```

落到源码上，这条链的三个"岔路口"就是三个标志：

```cpp
// 岔路 1：range 优化器是否给了 LIS
tmp_table_param.precomputed_group_by = is_loose_index_scan(qep_tab[0].range_scan()) &&
                                       !is_agg_loose_index_scan(...);   // sql_select.cc:4413

// 岔路 2：索引顺序能否满足 GROUP BY（test_skip_sort 的产物）
m_ordered_index_usage == ORDERED_INDEX_GROUP_BY                          // sql_optimizer.h:397

// 岔路 3：允不允许临时表做分组
tmp_table_param.allow_group_via_temp_table                               // temp_table_param.h
```

**为什么是这个顺序？** 因为它按"成本递增 + 输出属性递减"排列：

- ② 最便宜（只读索引端点），但只适用于极窄的查询形状；
- ① 次之（一次索引扫描，但读了全部行），且输出有序；
- ③ 需要物化全部行，但能处理任意分组键；
- ④ 最贵（全量排序 + 流式聚合），只有在临时表被显式禁止时才选。

注意 ④ 在成本上通常**不低于** ③，它只是"临时表被禁用"时的兜底。这正是 `SQL_BIG_RESULT` 可能让查询变慢的原因。

#### 5.1 判定链的落点：三类标志如何变成一条实现

判定链不是一段连续代码，而是散在 `make_tmp_tables_info()` 里的几个 if。**把关键分支按第五章的四类实现排列**：

```cpp
// ② 松索引扫描（LIS / MIN-MAX）：range 优化器已经给了 GROUP_INDEX_SKIP_SCAN
// sql/sql_select.cc:4413
  if (qep_tab && qep_tab[0].range_scan() &&
      is_loose_index_scan(qep_tab[0].range_scan()))
    tmp_table_param.precomputed_group_by =
        !is_agg_loose_index_scan(qep_tab[0].range_scan());
  // ⇒ 后续所有"分组"步骤因 precomputed_group_by 被跳过

// ③ 临时表聚合：只有「非 simple_group」才把 group_list 交给临时表做分组键
// sql/sql_select.cc:4448
    ORDER_with_src tmp_group;
    if (!simple_group && !(test_flags & TEST_NO_KEY_GROUP) && !with_json_agg)
      tmp_group = group_list;                          // ★ 为空 ⇒ 临时表不带分组唯一键
    tmp_table_param.hidden_field_count = CountHiddenFields(*fields);
    if (create_intermediate_table(&qep_tab[curr_tmp_table], *fields, tmp_group,
                                  !group_list.empty() && simple_group))
      return true;
    exec_tmp_table = qep_tab[curr_tmp_table].table();
    if (exec_tmp_table->s->is_distinct) optimize_distinct();
    ...
    setup_tmptable_write_func(&qep_tab[curr_tmp_table], &trace_this_outer);
    // ⇒ op_type = OT_AGGREGATE_INTO_TMP_TABLE（临时表有 group 键 + 有 sum func）
    //        或 = OT_MATERIALIZE（只是物化，分组在后面）

// ① 索引有序流式聚合：m_ordered_index_usage 已由 test_skip_sort 置位，
//    这里表现为「不加 filesort」+「streaming_aggregation 仍为 true」
// sql/sql_select.cc:4758
    ORDER_with_src order_arg = group_list.empty() ? order : group_list;
    if (qep_tab &&
        m_ordered_index_usage != (group_list.empty()
                                      ? ORDERED_INDEX_ORDER_BY
                                      : ORDERED_INDEX_GROUP_BY) &&
        !m_windowing_steps) {
      // ... 走到这里说明索引没能提供顺序 ⇒ 加 filesort
      if (add_sorting_to_table(curr_tmp_table, &order_arg,
                               /*sort_before_group=*/false))
        return true;
    }

// ④ filesort + 流式聚合：allow_group_via_temp_table=false 使 create_tmp_table 丢掉 group，
//    于是 streaming_aggregation 生效，排序由上面的 add_sorting_to_table 提供
// sql/sql_executor.cc:519（op_type 决定最终迭代器，详见 6.4）
  } else if (join->streaming_aggregation && !tmp_tbl->precomputed_group_by) {
    tab->op_type = QEP_TAB::OT_AGGREGATE_THEN_MATERIALIZE;
  }
```

> ★ **四类实现的"开关位"速查**：`precomputed_group_by`（②）、`simple_group` + `allow_group_via_temp_table`（③）、`m_ordered_index_usage`（①）、`allow_group_via_temp_table=false` + `streaming_aggregation`（④）。`streaming_aggregation` 在 `sql_select.cc:4571` 与 `:4659` 建临时表时被清零——所以**一旦走了临时表，就不会再叠加流式聚合**。

---

### 六、make_tmp_tables_info：临时表与排序的最终判定

> 位于 `sql/sql_select.cc`。它是 `JOIN::optimize` 的最后一步之一，读前面所有标志决定物理实现。详细阶段划分见 [`10_plan_refinement.md`](10_plan_refinement.md)，这里只讲与分组/去重相关的判据。

#### 6.1 需要临时表的五个来源

`need_tmp_before_win` 为真的条件（在 `JOIN::optimize` 与 `optimize_distinct_group_order` 中被置位）：

| 来源 | 说明 |
|---|---|
| 拿不到索引顺序且允许临时表聚合 | 第四章 4.3 的 LooseScan 保护分支 |
| `implicit_grouping` 且有聚合函数 | 隐式分组（有聚合无 GROUP BY）需要一行结果 |
| 窗口函数相关 | `m_windows` 非空、或 `m_windowing_steps` 非空 |
| DISTINCT 未被消掉 | 需要临时表唯一键去重 |
| 之前的 `m_windows_sort` | 窗口排序 |

#### 6.2 allow_group_via_temp_table：一个会被多方关掉的开关

定义在 `Temp_table_param`，默认 `true`。被置 `false` 的位置：

| 置 false 的地方 | 原因 |
|---|---|
| `optimize_distinct_group_order` 的 DISTINCT→GROUP BY 分支 | 必须按索引顺序流式输出（第三章 3.3 第 7 步） |
| `JOIN::optimize_rollup` | **ROLLUP 强制禁止临时表聚合**，只能流式（见 [`../runtime/06_rollup.md`](../runtime/06_rollup.md)） |
| `make_tmp_tables_info` 里遍历聚合函数 | 某些聚合函数（含部分 UDF）不支持增量更新，注释写着 `with_distinct` 时也要关 |
| `Item_sum` 构造时 | 特定聚合函数天生不支持（如 `SUM(DISTINCT x)` 在某些形态下） |

这个开关的语义是"**能否用临时表的唯一键做增量聚合**"。关掉之后，分组只能靠"输入有序 + 流式聚合"，于是逼出一个 filesort。

#### 6.3 is_agg_loose_index_scan：识别 LIS 已预计算

`range_optimizer/path_helpers.h` 里的内联函数，判断一个 `AccessPath` 是否为 `GROUP_INDEX_SKIP_SCAN`。`make_tmp_tables_info` 用它计算：

```cpp
const bool precomputed_group_by = !is_agg_loose_index_scan(qep_tab[0].range_scan());
```

即：**若已经用了 LIS，分组与 MIN/MAX 已被预计算**，后续不再需要"分组"这一步骤。这是 LIS 与常规分组实现的接口点。

#### 6.3.1 两个判定函数的真实定义（`range_optimizer/path_helpers.h:41`）

```cpp
inline bool is_loose_index_scan(const AccessPath *path) {
  return path->type == AccessPath::INDEX_SKIP_SCAN ||
         path->type == AccessPath::GROUP_INDEX_SKIP_SCAN;
}

inline bool is_agg_loose_index_scan(const AccessPath *path) {
  switch (path->type) {
    case AccessPath::INDEX_SKIP_SCAN:
      return path->index_skip_scan().param->has_aggregate_function;
      break;
    case AccessPath::GROUP_INDEX_SKIP_SCAN:
      return path->group_index_skip_scan().param->have_agg_distinct;
      break;
    default:
      return false;
  }
}
```

> ★ 二者不是一回事：`is_loose_index_scan` = "是不是松散索引扫描"；`is_agg_loose_index_scan` = "这个松散扫描是不是**聚合型**的（MIN/MAX 已在扫描期算完）"。`precomputed_group_by = is_loose_index_scan && !is_agg_loose_index_scan`。

#### 6.3.2 `make_tmp_tables_info()` 里的两个置位点（`sql_select.cc:4413` / `:4591`）

```cpp
// sql/sql_select.cc:4413
  const bool has_group_by = this->grouped;

  /*
    The loose index scan access method guarantees that all grouping or
    duplicate row elimination (for distinct) is already performed
    during data retrieval, and that all MIN/MAX functions are already
    computed for each group. Thus all MIN/MAX functions should be
    treated as regular functions, and there is no need to perform
    grouping in the main execution loop.
    Notice that currently loose index scan is applicable only for
    single table queries, thus it is sufficient to test only the first
    join_tab element of the plan for its access method.
  */
  if (qep_tab && qep_tab[0].range_scan() &&
      is_loose_index_scan(qep_tab[0].range_scan()))
    tmp_table_param.precomputed_group_by =
        !is_agg_loose_index_scan(qep_tab[0].range_scan());
```

```cpp
// sql/sql_select.cc:4591 —— 建第二张分组表时无条件置 true
      /*
        If the access method is loose index scan then all MIN/MAX
        functions are precomputed, and should be treated as regular
        functions. See extended comment above.
      */
      if (qep_tab[0].range_scan() &&
          is_loose_index_scan(qep_tab[0].range_scan()))
        tmp_table_param.precomputed_group_by = true;
```

#### 6.3.3 `allow_group_via_temp_table`：置 false 与消费的完整链路

**置 false**（`count_field_types()` 遍历聚合函数，`sql_select.cc:3910`）：

```cpp
void count_field_types(const Query_block *query_block, Temp_table_param *param,
                       const mem_root_deque<Item *> &fields,
                       bool reset_with_sum_func, bool save_sum_fields) {
  DBUG_TRACE;

  param->sum_func_count = 0;
  param->func_count = fields.size();
  param->hidden_field_count = 0;
  param->outer_sum_func_count = 0;
  /*
    Loose index scan guarantees that all grouping is done and MIN/MAX
    functions are computed, so create_tmp_table() treats this as if
    save_sum_fields is set.
  */
  save_sum_fields |= param->precomputed_group_by;

  for (Item *field : fields) {
    Item *real = field->real_item();
    Item::Type real_type = real->type();

    if (real_type == Item::SUM_FUNC_ITEM && !real->m_is_window_function) {
      if (!field->const_item()) {
        Item_sum *sum_item = down_cast<Item_sum *>(field->real_item());
        if (sum_item->aggr_query_block == query_block) {
          // ★ 某些聚合函数（含 UDF）不支持增量更新 → 禁止临时表分组
          if (!sum_item->allow_group_via_temp_table)
            param->allow_group_via_temp_table = false;  // UDF SUM function
          param->sum_func_count++;

          // Add one column per argument.
          param->func_count += sum_item->argument_count();
        }
      } else if (save_sum_fields) { ... }
    } else if (real_type == Item::SUM_FUNC_ITEM) { ... }
    else { ... }
  }
}
```

**消费**（`create_tmp_table()`，`sql/sql_tmp_table.cc:885`——只有两行，但决定了整个临时表形态）：

```cpp
  if (!param->allow_group_via_temp_table)
    group = nullptr;  // Can't use group key      // ★ 直接丢弃 group 参数 → 临时表不再带分组唯一键

  if (group != nullptr) distinct = false;  // Can't use distinct      // ★ 有 group 就互斥掉 distinct
```

**结构定义**（`sql/temp_table_param.h:95` 摘录）：

```cpp
  /**
    Number of fields in the query that have aggregate functions. Note
    that the optimizer may choose to optimize away these fields by
    replacing them with constants, in which case sum_func_count will
    need to be updated.
    @see optimize_aggregated_query, count_field_types
  */
  uint sum_func_count;
  uint hidden_field_count;
  uint group_parts, group_length, group_null_parts;
  /**
    Whether we allow running GROUP BY processing into a temporary table,
    i.e., keeping many different aggregations going at once without
    having ordered input. This is usually the case, but is currently not
    supported for aggregation UDFs, aggregates with DISTINCT, or ROLLUP.

    Note that even if this is true, the optimizer may choose to not use
    a temporary table, as it is often more efficient to just read along
    an index.
   */
  bool allow_group_via_temp_table{true};
  ...
  /*
    True if GROUP BY and its aggregate functions are already computed
    by a table access method (e.g. by loose index scan). In this case
    query execution should not perform aggregation and should treat
    aggregate functions as normal functions.
  */
  bool precomputed_group_by;
```

#### 6.4 从标志到迭代器：`op_type` → `AccessPath` 的三步映射

第五章的"四种实现"在执行期由 `QEP_TAB::op_type` 决定，而 `op_type` 由两个函数算出来：

**① `setup_tmptable_write_func()`**（`sql/sql_executor.cc:519`）——决定**往临时表里怎么写**：

```cpp
void setup_tmptable_write_func(QEP_TAB *tab, Opt_trace_object *trace) {
  DBUG_TRACE;
  JOIN *join = tab->join();
  TABLE *table = tab->table();
  Temp_table_param *const tmp_tbl = tab->tmp_table_param;
  uint phase = tab->ref_item_slice;
  const char *description = nullptr;
  assert(table);

  if (table->group && tmp_tbl->sum_func_count &&
      !tmp_tbl->precomputed_group_by) {
    /*
      Note for MyISAM tmp tables: if uniques is true keys won't be
      created.
    */
    assert(phase < REF_SLICE_WIN_1);
    if (table->s->keys) {
      description = "continuously_update_group_row";
      // ★ ③ 临时表聚合：边扫边在临时表里更新组行 → TemptableAggregateIterator
      tab->op_type = QEP_TAB::OT_AGGREGATE_INTO_TMP_TABLE;
    }
  } else if (join->streaming_aggregation && !tmp_tbl->precomputed_group_by) {
    assert(phase < REF_SLICE_WIN_1);
    description = "write_group_row_when_complete";
    DBUG_PRINT("info", ("Using end_write_group"));
    // ★ ① 索引有序流式聚合 → AggregateIterator，再物化
    tab->op_type = QEP_TAB::OT_AGGREGATE_THEN_MATERIALIZE;

    for (Item_sum **func_ptr = join->sum_funcs; *func_ptr != nullptr;
         ++func_ptr) {
      tmp_tbl->items_to_copy->push_back(
          Func_ptr(*func_ptr, (*func_ptr)->get_result_field()));
    }
  } else {
    description = "write_all_rows";
    tab->op_type = (phase >= REF_SLICE_WIN_1 ? QEP_TAB::OT_WINDOWING_FUNCTION
                                             : QEP_TAB::OT_MATERIALIZE);
    if (tmp_tbl->precomputed_group_by) {
      for (Item_sum **func_ptr = join->sum_funcs; *func_ptr != nullptr;
           ++func_ptr) {
        tmp_tbl->items_to_copy->push_back(
            Func_ptr(*func_ptr, (*func_ptr)->get_result_field()));
      }
    }
  }
  if (description) trace->add_alnum("write_method", description);
}
```

**② `JOIN::get_end_select_func()`**（`sql_executor.cc:574`）——决定**最后一步是不是流式聚合**：

```cpp
QEP_TAB::enum_op_type JOIN::get_end_select_func() {
  DBUG_TRACE;
  /*
     Choose method for presenting result to user. Use end_send_group
     if the query requires grouping (has a GROUP BY clause and/or one or
     more aggregate functions). Use end_send if the query should not
     be grouped.
   */
  if (streaming_aggregation && !tmp_table_param.precomputed_group_by) {
    DBUG_PRINT("info", ("Using end_send_group"));
    return QEP_TAB::OT_AGGREGATE;      // ★ 收尾挂 AggregateIterator
  }
  DBUG_PRINT("info", ("Using end_send"));
  return QEP_TAB::OT_NONE;
}
```

**③ `JOIN::create_root_access_path_for_join()`**（`sql_executor.cc:2970`；★ 真实函数名，**不存在** `JOIN::create_iterators()`）——按 `op_type` 挂 `AccessPath`：

```cpp
  if (qep_tab->op_type == QEP_TAB::OT_AGGREGATE_THEN_MATERIALIZE) {
      // Aggregate as we go, with output into a temporary table.
      if (!qep_tab->tmp_table_param->precomputed_group_by) {
        path = NewAggregateAccessPath(thd, path,
                                      rollup_state != RollupState::NONE);   // :3056
        EstimateAggregateCost(path, query_block);
      }
  }
  ...
  } else if (qep_tab->op_type == QEP_TAB::OT_AGGREGATE_INTO_TMP_TABLE) {
      path = NewTemptableAggregateAccessPath(
          thd, path, qep_tab->tmp_table_param, qep_tab->table(), table_path,
          qep_tab->ref_item_slice);                                        // :3204
      if (qep_tab->having != nullptr) {
        path = NewFilterAccessPath(thd, path, qep_tab->having);
      }
  } else {
      assert(qep_tab->op_type == QEP_TAB::OT_MATERIALIZE ||
             qep_tab->op_type == QEP_TAB::OT_AGGREGATE_THEN_MATERIALIZE);

      // If we don't need the row IDs, and don't have some sort of deduplication
      // (e.g. for GROUP BY) on the table, filesort can take in the data
      // directly, without going through a temporary table.
      Filesort *first_sort = dup_filesort != nullptr ? dup_filesort : filesort;
      AccessPath *old_path = path;
      if (first_sort != nullptr && first_sort->using_addon_fields() &&
          !MaterializeIsDoingDeduplication(qep_tab->table())) {
        path = NewStreamingAccessPath(
            thd, path, /*join=*/this, qep_tab->tmp_table_param,
            qep_tab->table(), qep_tab->ref_item_slice);                    // :3233
        CopyBasicProperties(*old_path, path);
      } else {
        path = NewMaterializeAccessPath(...);                              // :3238
        EstimateMaterializeCost(thd, path);
      }
  }

  if (qep_tab->condition() != nullptr) {
    path = NewFilterAccessPath(thd, path, qep_tab->condition());
    qep_tab->mark_condition_as_pushed_to_sort();
  }

  if (limit_1_for_dup_filesort) {
    path = NewLimitOffsetAccessPath(thd, path, /*limit=*/1, ...);
  } else if (dup_filesort != nullptr) {
    path = NewSortAccessPath(thd, path, dup_filesort, dup_filesort_order,
                             /*count_examined_rows=*/true);                // :3263 DISTINCT 去重排序
  }
  if (filesort != nullptr) {
    path = NewSortAccessPath(thd, path, filesort, filesort_order,
                             /*count_examined_rows=*/true);                // :3267 ORDER/GROUP BY 排序
  }
```

以及最后"是否收尾再聚合一次"的判定（`:3275`）：

```cpp
  // See if we need to aggregate data in the final step. Note that we can
  // _not_ rely on streaming_aggregation, as it can be changed from false
  // to true during optimization, and depending on when it was set, it could
  // either mean to aggregate into a temporary table or aggregate on final
  // send.
  bool do_aggregate;
  if (primary_tables == 0 && tmp_tables == 0) {
    do_aggregate = (get_end_select_func() == QEP_TAB::OT_AGGREGATE);
  } else {
    // Note that tmp_table_param.precomputed_group_by can be set even if we
    // don't actually have any grouping (e.g., make_tmp_tables_info() does this
    // even if there are no temporary tables made).
    do_aggregate = (qep_tab[primary_tables + tmp_tables].op_type ==
                    QEP_TAB::OT_AGGREGATE) ||
                   ((grouped || group_optimized_away) &&
                    tmp_table_param.precomputed_group_by);
  }
  if (do_aggregate) {
    assert(streaming_aggregation || tmp_table_param.precomputed_group_by);
    ...
    if (!tmp_table_param.precomputed_group_by) {
      path =
          NewAggregateAccessPath(thd, path, rollup_state != RollupState::NONE);  // :3302
      EstimateAggregateCost(path, query_block);
    }
  }
```

> ★ **注意这条注释**：*"we can **not** rely on `streaming_aggregation`"*——因为它在优化期可能从 false 变 true（例如 `:1636` 的 `is_indexed_agg_distinct` 分支把它关掉），且**置位的时机不同含义不同**（之前置 = 聚合进临时表，之后置 = 最后一步聚合）。所以最终判定改用 `op_type`。

#### 6.5 真正造 `Filesort` 的地方：`JOIN::add_sorting_to_table()`（`sql_select.cc:4984`）

```cpp
bool JOIN::add_sorting_to_table(uint idx, ORDER_with_src *sort_order,
                                bool sort_before_group) {
  DBUG_TRACE;
  ASSERT_BEST_REF_IN_JOIN_ORDER(this);
  assert(!query_block->is_recursive());
  const enum join_type jt = qep_tab[idx].type();
  if (jt == JT_CONST || jt == JT_EQ_REF)
    return false;  // 1 single row: is already sorted

  // Weedout needs an underlying table to store refs from (it deduplicates
  // by row ID), so if this table is part of a weedout operation, we need
  // to force sorting by row IDs ...
  bool force_sort_rowids = false;
  for (plan_idx i = 0; i <= static_cast<plan_idx>(idx); ++i) {
    if (!qep_tab[i].starts_weedout()) {
      continue;
    }

    plan_idx weedout_end = NO_PLAN_IDX;  // Exclusive.
    for (uint j = i; j < primary_tables; ++j) {
      if (qep_tab[j].check_weed_out_table == qep_tab[i].flush_weedout_table) {
        weedout_end = j + 1;
        break;
      }
    }
    if (weedout_end != NO_PLAN_IDX &&
        weedout_end > static_cast<plan_idx>(idx)) {
      force_sort_rowids = true;
      break;
    }
  }

  explain_flags.set(sort_order->src, ESP_USING_FILESORT);   // ★★ 这里用 ORDER_with_src 的 src 归因
  QEP_TAB *const tab = &qep_tab[idx];
  bool keep_buffers = ...;

  {
    // Switch to the right slice if applicable, so that we fetch out the correct
    // items from order_arg.
    Switch_ref_item_slice slice_switch(this, tab->ref_item_slice);
    tab->filesort = new (thd->mem_root)
        Filesort(thd, {tab->table()}, keep_buffers, sort_order->order,
                 HA_POS_ERROR, /*remove_duplicates=*/false, force_sort_rowids,
                 /*unwrap_rollup=*/sort_before_group);
    tab->filesort_pushed_order = sort_order->order;
  }
  if (!tab->filesort) return true;
  Opt_trace_object trace_tmp(&thd->opt_trace, "filesort");
  trace_tmp.add_alnum("adding_sort_to_table",
                      tab->table() ? tab->table()->alias : "");

  return false;
}
```

> ★ `sort_before_group` 参数 = 第四章"④ 先 filesort 再流式聚合"的形态——它决定 `Filesort` 是否 `unwrap_rollup`（ROLLUP 的 group wrapper 要在分组**之前**解开）。

**在 `make_tmp_tables_info` 里怎么被调**（`sql_select.cc:4740` 附近）：

```cpp
    ORDER_with_src order_arg = group_list.empty() ? order : group_list;  // ★ GROUP BY 优先，ORDER BY 兜底
    if (qep_tab &&
        m_ordered_index_usage != (group_list.empty()
                                      ? ORDERED_INDEX_ORDER_BY
                                      : ORDERED_INDEX_GROUP_BY) &&
        // Windowing will change order, so it's too early to sort here
        !m_windowing_steps) {
      // Sort either first non-const table or the last tmp table
      QEP_TAB *const sort_tab = &qep_tab[curr_tmp_table];
      if (need_tmp_before_win && !materialize_join && !exec_tmp_table->group)
        explain_flags.set(order_arg.src, ESP_USING_TMPTABLE);

      if (add_sorting_to_table(curr_tmp_table, &order_arg,
                               /*sort_before_group=*/false))
        return true;
      /*
        filesort_limit:	 Return only this many rows from filesort().
        We can use select_limit_cnt only if we have no group_by and 1 table.
        This allows us to use Bounded_queue for queries like:
          "select * from t1 order by b desc limit 1;"
      */
      if (sort_tab->filesort)
        sort_tab->filesort->limit =
            (has_group_by || (primary_tables > curr_tmp_table + 1) ||
             calc_found_rows)
                ? m_select_limit
                : query_expression()->select_limit_cnt;
    }
```

> ★★ `m_ordered_index_usage != (...)` 这一行就是**第四章三态枚举的兑现点**：若索引已能提供所需顺序，就不加 filesort。

#### 6.6 DISTINCT 用临时表唯一键去重

`distinct_arg` 的构造（`create_intermediate_table()`，`sql_executor.cc:179`）：

```cpp
bool JOIN::create_intermediate_table(
    QEP_TAB *const tab, const mem_root_deque<Item *> &tmp_table_fields,
    ORDER_with_src &tmp_table_group, bool save_sum_fields) {
  DBUG_TRACE;
  THD_STAGE_INFO(thd, stage_creating_tmp_table);
  const bool windowing = m_windows.elements > 0;
  /*
    Pushing LIMIT to the temporary table creation is not applicable
    when there is ORDER BY or GROUP BY or aggregate/window functions, because
    in all these cases we need all result rows.
  */
  ha_rows tmp_rows_limit =
      ((order.empty() || skip_sort_order) && tmp_table_group.empty() &&
       !windowing && !query_block->with_sum_func)
          ? m_select_limit
          : HA_POS_ERROR;

  tab->tmp_table_param =
      new (thd->mem_root) Temp_table_param(thd->mem_root, tmp_table_param);
  tab->tmp_table_param->skip_create_table = true;

  bool distinct_arg =
      select_distinct &&
      // GROUP BY is absent or has been done in a previous step
      group_list.empty() &&
      // We can only do DISTINCT in last window's tmp table step
      (!windowing || (tab->tmp_table_param->m_window &&
                      tab->tmp_table_param->m_window->is_last()));

  TABLE *table =
      create_tmp_table(thd, tab->tmp_table_param, tmp_table_fields,
                       tmp_table_group.order, distinct_arg, save_sum_fields,
                       query_block->active_options(), tmp_rows_limit, "");
```

建 `<auto_distinct_key>` 唯一键（`create_tmp_table()`，`sql/sql_tmp_table.cc:1314`）：

```cpp
  } else if ((distinct ||
              param->m_operation != Temp_table_param::TTP_UNION_OR_TABLE) &&
             share->fields != param->hidden_field_count) {
    /*
      Create an unique key or an unique constraint over all columns
      that should be in the result.  In the temporary table, there are
      'param->hidden_field_count' extra columns, whose null bits are stored
      in the first 'hidden_null_pack_length' bytes of the row.
    */
    share->keys = 1;
    share->is_distinct =
        distinct || param->m_operation == Temp_table_param::TTP_INTERSECT ||
        param->m_operation == Temp_table_param::TTP_EXCEPT;

    if (!unique_constraint_via_hash_field) {
      param->keyinfo->table = table;
      param->keyinfo->is_visible = true;
      param->keyinfo->user_defined_key_parts =
          share->fields - param->hidden_field_count;   // ★ 唯一键只覆盖"非隐藏"列
      param->keyinfo->actual_key_parts = param->keyinfo->user_defined_key_parts;
      KEY_PART_INFO *key_part_info = share->mem_root.ArrayAlloc<KEY_PART_INFO>(
          param->keyinfo->user_defined_key_parts);
      if (key_part_info == nullptr) return nullptr;
      param->keyinfo->key_part = key_part_info;
      // ★★ HA_NULL_ARE_EQUAL：DISTINCT 语义下 NULL == NULL
      param->keyinfo->flags = HA_NOSAME | HA_NULL_ARE_EQUAL;
      param->keyinfo->actual_flags = param->keyinfo->flags;
      param->keyinfo->name = "<auto_distinct_key>";
      param->keyinfo->set_rec_per_key_array(nullptr, nullptr);
      param->keyinfo->set_in_memory_estimate(IN_MEMORY_ESTIMATE_UNKNOWN);

      /* Create a distinct key over the columns we are going to return */
      for (unsigned i = param->hidden_field_count; i < share->fields;
           i++, key_part_info++) {
        key_part_info->init_from_field(table->field[i]);
        if (key_part_info->store_length > max_key_part_length) {
          unique_constraint_via_hash_field = true;   // ★ 太长 → 退化成哈希列
          break;
        }
      }
      table->key_info = param->keyinfo;
      share->key_info = param->keyinfo;
      share->key_parts = param->keyinfo->user_defined_key_parts;
    }
  }

  /*
    To enforce unique constraint we need to add a field to hold key's hash
    A1) distinct key is too long
    A2) number of keyparts in distinct key is too big
    A3) caller cannot accept distinct via indexes (e.g. because it wants
        to turn off the checking at some point)
  */
  if (distinct) {
    if (distinct_key_length > max_key_length ||                   // 1
        (fieldnr - param->hidden_field_count) > max_key_parts ||  // 2
        param->force_hash_field_for_unique) {                     // 3
      unique_constraint_via_hash_field = true;
    }
  }
```

> ★ `HA_NULL_ARE_EQUAL` 是 DISTINCT 与 UNIQUE 索引的关键区别：唯一索引允许多个 NULL（所以 3.2 节才要求"所有 keypart NOT NULL"），而 DISTINCT 认为 `NULL` 与 `NULL` 相同。

迭代器模型下 DISTINCT 的**替代去重路径**（`sql_executor.cc:3097`）：

```cpp
    // The pre-iterator executor did duplicate removal by going into the
    // temporary table and actually deleting records, using a hash table for
    // smaller tables and an O(n²) algorithm for large tables. This kind of
    // deletion is not cleanly representable in the iterator model, so we do it
    // using a duplicate-removing filesort instead, which has a straight-up
    // O(n log n) cost.
    if (qep_tab->needs_duplicate_removal) {
      bool all_order_fields_used;
      ORDER *desired_order = this->order.order;
      if (desired_order == nullptr &&
          this->qep_tab[0].filesort_pushed_order != nullptr) {
        desired_order = this->qep_tab[0].filesort_pushed_order;
      }
      ...
      ORDER *order = create_order_from_distinct(
          thd, ref_items[qep_tab->ref_item_slice], desired_order, select_list,
          /*skip_aggregates=*/false, /*convert_bit_fields_to_long=*/false,
          &all_order_fields_used);
```

---

### 七、与 ROLLUP / 窗口 / HAVING / ONLY_FULL_GROUP_BY 的交互

| 特性 | 优化期的影响 |
|---|---|
| **ROLLUP** | `JOIN::optimize_rollup` 强制 `allow_group_via_temp_table = false`，ROLLUP 只能流式聚合；同时 `optimize_distinct_group_order` 里 `query_block->olap == ROLLUP_TYPE` 会阻止"GROUP BY 顺带删 ORDER BY" |
| **窗口函数** | `m_windows_sort` 为真时不允许用 GROUP BY 的顺序去消 ORDER BY（窗口会重排）；窗口函数强制 `need_tmp_before_win`，即必须先物化 |
| **HAVING** | 在 `optimize_distinct_group_order` 之前被 `optimize_cond` 处理；HAVING 引用聚合函数时不能下推到 WHERE（见 [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md)） |
| **ONLY_FULL_GROUP_BY** | 在 `Query_block::prepare` 的 `Group_check` / `Distinct_check` 里检查（`aggregate_check.h`，WL#2489），早于优化期。优化期看到的是**已通过检查**的查询（详见下节） |
| **DISTINCT + ORDER BY** | 若 ORDER BY 的列不在 SELECT list 中，`create_order_from_distinct` 的 `all_order_fields_used` 为 false，导致改写被拒，只能走临时表去重 |

#### ★ ONLY_FULL_GROUP_BY 的函数依赖推导（`Group_check`）

⚠️ **先纠正一个类名**：8.0.39 里**没有** `Aggregate_check` 这个类。`aggregate_check.h` 定义的是 **`Group_check`**（管 GROUP BY / 聚合）和 **`Distinct_check`**（管 DISTINCT），文档组名是 `AGGREGATE_CHECKS`。

入口与总开关（`check_only_full_group_by`）：

```cpp
  if (is_grouped()) {
    Group_check gc(this, &root);
    rc = gc.check_query(thd);
  }
  if (!rc && is_distinct()) {
    Distinct_check dc(this);
    rc = dc.check_query(thd);
  }
```

调用处带三重条件——**关闭 `ONLY_FULL_GROUP_BY` 后是"完全不检查"，不是"降级检查"**：

```cpp
  if ((is_distinct() || is_grouped()) &&
      (thd->variables.sql_mode & MODE_ONLY_FULL_GROUP_BY) &&
      check_only_full_group_by(thd)) return true;
```

##### 核心算法：不动点迭代

8.0 把 5.7"列是否出现在 GROUP BY 里"的朴素检查，重写成基于**函数依赖（FD）+ 唯一键闭包**的推导。算法注释原文：

```
   We build sets, named En, by induction.
   A "column" is defined as base table / view / derived table's column.

   E1 = {source columns} (=group columns, if this is a master Group_check;
   empty set otherwise).

   En is a set of columns of the result of the WHERE clause of 'select' which
   are functionally dependent on E1.

   Given En, build En+1:
    - let En+1= En
    - for each {pk/unique index of some table T} found in En, add T.* to En+1

   Then build En+2, by adding y, for each x=y in AND parts of WHERE/ON where
   x is in En+1 or is constant, and y is a column not in En+1.

   ...
   If En+2==En, this is the end of the induction. Otherwise we loop.
```

即每轮两步：**唯一键闭包**（En→En+1）+ **等值传播**（En+1→En+2），直到不再增长。

##### 三种 FD 来源

| 来源 | 规则 | 证据 |
|---|---|---|
| **唯一键 / 主键** | `{key 全部列} -> {T.*}` | 只认 `HA_NOSAME`；必须键的每一列都在 `fd` 里（`k == user_defined_key_parts`）；命中后**整表加入** `whole_tables_fd` |
| **WHERE/ON 等值** | `{T1.A} -> {T2.B}`（且反向） | `analyze_scalar_eq()`：`left_is_column && is_in_fd(left_item)` 或 `left_item->const_item()` 或 `is_outer_reference()` |
| **NULL 拒绝** | 让**可空**唯一索引也能用 | 见下 |

**"NULL 拒绝"这条很精妙**。注释给的例子：

```
While
 select a,b from t1 group by a;
is invalid (because two null values of 'a' go in same group),
 select a,b from t1 where 'a' is not null group by a;
is valid: 'a' can be treated as non-nullable in t1, so the unique index is
like "unique not null", so 'a' determines 'b'.
```

实现链条：`IS NOT NULL`（或 `>` `<` 等任何"NULL 时不为真"的条件）→ `not_null_tables()` → 把 `fd` 里对应的 Item 打上 `MARKER_FUNC_DEP_NOT_NULL` → 下一轮重新测唯一键 → 通过。

##### NFFD：传播的核心概念

外连接场景下，FD 能不能"穿过"外连接传播？源码引入了一个概念（注释明说是 **Guilhem 的发明**）：

```
Let's define a "NULL-friendly functional dependency" (NFFD) as a functional
dependency between two sets A and B, which has two properties:
- A is not empty
- if, in a row, all values of A are NULL, then all values of B are NULL.

All NFFDs in TW are also NFFDs in R.
```

传播总结：

```
All NULL-friendly functional dependencies propagate freely through join nests
all the way up to the result of the WHERE clause.
The same is true for ordinary functional dependencies except if there are weak
tables along the propagation path ...
```

以及哪些 FD **不是** NULL-friendly：

```
Summary: all functional dependencies which we have seen so far are
NULL-friendly, except those inferred from TW.A=constant in an outer join
condition or in a WHERE clause, and those about generated columns.
```

##### master / child：物化派生表怎么办

`Group_check` 成一个树：master 验证一个 query block；若引用了**物化**派生表/视图，就为它建一个 child `Group_check`，在底层 query block 里发现 FD 再回传。注释给了完整例子：

```
   For example:
     select d.a, d.b from (select t.a*2 as a, t.a as b from t) group by d.b;
   When we add d.b to 'fd', ... we create mat_gc, see that d.b is
   built from a column of t (t.b), we can say that "t.b is determined" ...
   Later, when we wonder if d.a is functionally dependent on d.b ...
   we conclude that d.a is indeed FD on d.b.
```

回传要满足三个条件（`group_in_fd == ~0ULL`、非 weak side、或源含非空列）。

##### 检查哪些位置

`check_query()` 检查**四类位置**：SELECT list、ORDER BY、HAVING、窗口的 PARTITION BY / ORDER BY。ORDER BY 里**已出现在 SELECT list 的跳过**。

报错时保留**旧错误码**换**新错误文本**（5.7 兼容性 hack）。

##### 聚合函数为什么"永远合法"

`is_in_fd()` 对聚合函数的判定：

```cpp
  if ((item->type() == Item::SUM_FUNC_ITEM && !item->m_is_window_function) || ...) {
    /*
      If all group expressions are FD on the source, this set function also is
      (one single value per group).
    */
    return group_in_fd == ~0ULL;
  }
```

**聚合函数的参数列从来不被检查**——所以 `SELECT COUNT(b) FROM t GROUP BY a` 里的 `b` 永远合法。注意 `m_is_window_function` 被**显式排除**（窗口函数不享受豁免）。

##### ★ 核心函数完整源码

> **勘误表**（8.0.39 实测）：
> - **不存在** `Aggregate_check` 类；真实是 **`Group_check`**（`aggregate_check.h:560`）与 **`Distinct_check`**（`aggregate_check.h:530`），二者都继承 `Item_tree_walker`
> - **不存在** `Group_check::find_fd_in_pk()`——PK/UNIQUE 的 FD 推导**内联在 `is_fd_on_source()` 的 `while(true)` 循环里**
> - **不存在** `find_fd_in_underlying()`——真名 `is_in_fd_of_underlying()`（`aggregate_check.cc:738`）
> - **不存在** `add_to_source_of_mask()`——真名 `add_to_source_of_mat_table()`
> - **不存在** `is_column_not_null()` 成员——真实是友元 `Item_ident::is_column_not_in_fd()` / `is_strong_side_column_not_in_fd()`
> - **不存在**成员 `weak_tables_fd`——weak 表以**形参** `table_map weak_tables` 逐层传递
> - `mat_gc` **不是成员**，是 `add_to_source_of_mat_table()` 的局部变量

**`Group_check` 类定义**（`sql/aggregate_check.h:560`）：

```cpp
/**
   Checks for queries which have GROUP BY or aggregate functions.
*/
class Group_check : public Item_tree_walker {
 public:
  Group_check(Query_block *select_arg, MEM_ROOT *root)
      : select(select_arg),
        search_in_underlying(false),
        non_null_in_source(false),
        table(nullptr),
        group_in_fd(~0ULL),        // ★ master：所有 GROUP 表达式默认视为已在 FD 中
        m_root(root),
        fd(root),
        whole_tables_fd(0),
        recheck_nullable_keys(0),
        mat_tables(root),
        failed_ident(nullptr) {}
  ...
  /// Query block which we are validating
  Query_block *const select;

  /**
     "Underlying" == expressions which are underlying in an identifier.
     The identifier can be a column of a view or derived table, both merged or
     materialized, or a generated column: all of those have an underlying
     expression.
     "Materialized table (mat table)" == view or derived table, materialized.
     If this is true, is_in_fd() will look for FDs in underlying expressions
     of columns.
  */
  bool search_in_underlying;

  /**
     This member is readable only if this is a child Group_check. Is true if
     one expression of the SELECT list of this->select is non-nullable.
  */
  bool non_null_in_source;

  Table_ref *const table;   // ★ child 时指向被物化的表；master 时为 nullptr

  /**
     Bit N is set if the N-th expression of GROUP BY is functionally dependent
     on source columns.
     ...
     Currently limited to 64 bits => max 64 GROUP expressions; should probably
     be MY_BITMAP.
  */
  ulonglong group_in_fd;

  MEM_ROOT *const m_root;

  /**
     Columns which are local to 'select' and functionally dependent on an
     initial set of "source" columns defined like this:
     - if !is_child(), the GROUP BY columns
     - if is_child(), columns of the result of the query expression under
     'table' which are themselves part of 'fd' of the parent Group_check.
  */
  Mem_root_array<Item_ident *> fd;      // ★★ En：已推定的函数依赖列集合
  /// Map of tables for which all columns can be considered part of 'fd'.
  table_map whole_tables_fd;
  /// Map of tables for which we discovered known-not-nullable columns.
  table_map recheck_nullable_keys;
  /// Children Group_checks of 'this'
  Mem_root_array<Group_check *> mat_tables;
  /// Identifier which triggered an error
  Item_ident *failed_ident;

  bool is_fd_on_source(Item *item);
  bool is_child() const { return table != nullptr; }
  ...
 private:
  /// Private ctor, for a Group_check to build a child Group_check
  Group_check(Query_block *select_arg, MEM_ROOT *root, Table_ref *table_arg)
      : ...,
        group_in_fd(0ULL),        // ★ child：起始为空，靠 master 逐步喂
        ...
```

**`Group_check::is_fd_on_source()` —— 不动点迭代主循环**（`aggregate_check.cc:336`）：

```cpp
bool Group_check::is_fd_on_source(Item *item) {
  if (is_in_fd(item)) return true;

  if (!is_child()) {
    if (fd.empty()) {
      /*
        We do a first attempt: is the column part of group columns? This
        test should be sufficient to accept any query accepted by
        only_full_group_by in 5.6, and avoids filling the "fd" list with
        add_to_fd() (and potentially add_to_source_of_mat_table()).
        It's just an optimization.
      */
      // ★★ 5.6 兼容快路径：先只比 GROUP BY 列表
      for (ORDER *grp = select->group_list.first; grp; grp = grp->next) {
        if ((*grp->item)->eq(item, false)) return true;
      }
      // It didn't suffice. Let's start the search for FDs: build E1.
      for (ORDER *grp = select->group_list.first; grp; grp = grp->next) {
        Item *const grp_it = *grp->item;
        add_to_fd(grp_it, local_column(grp_it));
      }
    }
  }

  if (select->olap != UNSPECIFIED_OLAP_TYPE) {
    /*
      - the syntactical transformation of ROLLUP is to make a union of
      queries, and in each such query, some group column references are
      replaced with a NULL literal.
      - functional dependencies should be recognized only after that
      transformation. But there cannot be a key-based or equality-based
      functional dependency on a NULL literal.
      Moreover, there are no FDs in a UNION.
      So if the query has ROLLUP, we can stop here. ...
    */
    return false;      // ★★ ROLLUP 完全放弃 FD 推导
  }

  // no need to search for keys in those tables:
  table_map tested_map_for_keys = whole_tables_fd;
  recheck_nullable_keys = 0;
  while (true) {
    // build En+1 —— 唯一键闭包
    const table_map last_whole_tables_fd = whole_tables_fd;
    for (uint j = 0; j < fd.size(); j++) {
      Item *const item2 = fd.at(j)->real_item();  // Go down view field
      if (item2->type() != Item::FIELD_ITEM) continue;

      Item_field *const item_field = down_cast<Item_field *>(item2);
      Table_ref *const tl = item_field->field->table->pos_in_table_list;

      if (tested_map_for_keys & tl->map()) continue;
      tested_map_for_keys |= tl->map();
      for (uint keyno = 0; keyno < tl->table->s->keys; keyno++) {
        KEY *const key_info = &tl->table->key_info[keyno];
        if (!(key_info->flags & HA_NOSAME)) continue;   // ★ 只认 UNIQUE / PK
        uint k;
        for (k = 0; k < key_info->user_defined_key_parts; k++) {
          const Field *const key_field = key_info->key_part[k].field;
          bool key_field_in_fd = false;
          for (uint l = 0; l < fd.size(); l++) {
            Item *const item3 = fd.at(l)->real_item();
            if (item3->type() != Item::FIELD_ITEM) continue;
            if (static_cast<Item_field *>(item3)->field == key_field &&
                // Not a nullable column, or can be treated as not nullable
                (!key_field->is_nullable() ||
                 item3->marker == Item::MARKER_FUNC_DEP_NOT_NULL)) {  // ★★ NULL 拒绝在这里生效
              key_field_in_fd = true;
              break;
            }
          }
          if (!key_field_in_fd) break;
        }
        if (k == key_info->user_defined_key_parts) {
          /*
            We just found that intersect(En,table.*) contains all columns of
            the key, so intersect(En,table.*) -> table.* in 'table'.
            This is key-based so is an NFFD, so it propagates to the result of
            the WHERE clause. Thus, intersect(En,table.*) -> table.* in this
            result, so En -> table.* in this result.
            We knew that E1 -> En in this result.
            So, E1 -> table.* there too. So we can add table.* to En+1:
          */
          add_to_fd(tl->map());       // ★ 整表所有列进入 FD
          break;
        }
      }
    }
    if (last_whole_tables_fd !=
            whole_tables_fd &&  // something new, check again
        is_in_fd(item))
      return true;

    // Build En+2 —— 等值传播
    uint last_fd = fd.size();

    find_fd_in_joined_table(select->m_current_table_nest);  // [OUTER] JOIN ON
    if (select->where_cond())                               // WHERE
      find_fd_in_cond(select->where_cond(), 0, false);

    table_map map_of_new_fds = recheck_nullable_keys;
    recheck_nullable_keys = 0;

    for (; last_fd < fd.size(); ++last_fd)
      map_of_new_fds |= fd.at(last_fd)->used_tables();

    map_of_new_fds &= ~PSEUDO_TABLE_BITS;
    if (map_of_new_fds != 0)  // something new, check again
    {
      if (is_in_fd(item)) return true;
      // Recheck keys only in tables with something new:
      tested_map_for_keys &= ~map_of_new_fds;
    } else {
      // If already searched in expressions underlying identifiers.
      if (search_in_underlying) return false;   // ★ 不动点：不再增长 ⇒ 结束

      // Otherwise, iterate once more and dig deeper.
      search_in_underlying = true;

      if (is_in_fd(item)) return true;
    }
  }  // while(true)
}
```

```cpp
// sql/aggregate_check.cc:295 —— 那段 "We build sets, named En, by induction" 算法注释的确切位置：
// 它是 is_fd_on_source() 的 doxygen（/\*\* 在 :295，正文 "We build sets" 在 :304）
```

**`Group_check::is_in_fd()`**（`aggregate_check.cc:678`）：

```cpp
bool Group_check::is_in_fd(Item *item) {
  if ((item->type() == Item::SUM_FUNC_ITEM && !item->m_is_window_function) ||
      (item->type() == Item_func::FUNC_ITEM &&
       (((Item_func *)item)->functype() == Item_func::GROUPING_FUNC))) {
    /*
      If all group expressions are FD on the source, this set function also is
      (one single value per group).
    */
    return group_in_fd == ~0ULL;    // ★★ 聚合函数的豁免就这一行
  }

  assert(local_column(item));
  Used_tables ut(select);
  (void)item->walk(&Item::used_tables_for_level, enum_walk::POSTFIX,
                   pointer_cast<uchar *>(&ut));
  if ((ut.used_tables & ~whole_tables_fd) == 0) {
    /*
      The item is a column from a table whose all columns are FD. ...
    */
    return true;                    // ★ 整表已在 FD 里 → 直接放行
  }
  for (uint j = 0; j < fd.size(); j++) {
    Item *const item2 = fd.at(j);
    if (item2->eq(item, false)) return true;
    /*
      Say that we have view:
      create view v1 as select i, 2*i as z from t1; and we do:
      select z from v1 group by i;
      ... For us to find this t1.i in "fd" we have to
      reach to real_item() of v1.i.
    */
    Item *const real_it2 = item2->real_item();
    if (real_it2 != item2 && real_it2->eq(item, false)) return true;
  }
  if (!search_in_underlying) return false;
  return is_in_fd_of_underlying(down_cast<Item_ident *>(item));
}
```

**`Group_check::analyze_scalar_eq()`**（`aggregate_check.cc:991`）——等值传播的核心：

```cpp
void Group_check::analyze_scalar_eq(Item *cond, Item *left_item,
                                    Item *right_item, table_map weak_tables,
                                    bool weak_side_upwards) {
  table_map left_tables = left_item->used_tables();
  table_map right_tables = right_item->used_tables();
  bool left_is_column = local_column(left_item);
  bool right_is_column = local_column(right_item);

  /*
    We look for something=col_not_FD.
    If there are weak tables, this column must be weak (something=strong gives
    us nothing, in an outer join condition).
  */
  if (right_is_column && (!weak_tables || (weak_tables & right_tables)) &&
      !is_in_fd(right_item)) {
  } else if (left_is_column && (!weak_tables || (weak_tables & left_tables)) &&
             !is_in_fd(left_item)) {
    // col_not_FD=something: change to something=col_not_FD
    std::swap(left_item, right_item);
    std::swap(left_tables, right_tables);
    std::swap(left_is_column, right_is_column);
  } else
    return;  // this equality brings nothing

  // right_item is a column not in fd, see if we can add it.

  if (left_is_column && (weak_tables & left_tables) && is_in_fd(left_item)) {
    // weak=weak: left->right, and this is NFFD
    add_to_fd(right_item, true);
    return;
  }

  const table_map strong_tables = (~weak_tables) & ~PSEUDO_TABLE_BITS;

  if ((left_is_column && (strong_tables & left_tables) &&
       is_in_fd(left_item)) ||
      left_item->const_item() || left_item->is_outer_reference()) {
    // strong_or_literal_or_outer_ref= right_item

    if (!weak_tables) {
      /*
        It cannot be an inner join, due to transformations done in
        simplify_joins(). So it is WHERE, so right_item is strong.
        This may be constant=right_item and thus not be an NFFD, but WHERE is
        exterior to join nests so propagation is not needed.
      */
      assert(!weak_side_upwards);  // cannot be inner join
      add_to_fd(right_item, true);
    } else {
      // Outer join. So condition must be deterministic.
      if (cond->is_non_deterministic()) return;

      /*
        FD will have DJS as source columns, where DJS is the set of strong
        columns referenced by "cond". FD has to propagate. It does if:
        - either there is no weak side upwards
        - or NULLness of DJS columns implies that "cond" is not true.
      */
      if (weak_side_upwards && !(strong_tables & cond->not_null_tables()))
        return;                     // ★★ not_null_tables() 消费点：保证 NFFD

      std::pair<Group_check *, table_map> p(this, strong_tables);
      if (!cond->walk(&Item::is_strong_side_column_not_in_fd, walk_options,
                      (uchar *)&p)) {
        add_to_fd(right_item, true);
      }
    }
  }
}
```

**`add_to_fd()` 与 `find_group_in_fd()`**（`aggregate_check.cc:491` / `:552`）——维护 `group_in_fd` 位图：

```cpp
void Group_check::add_to_fd(Item *item, bool local_column,
                            bool add_to_mat_table) {
  /*
    Because the "fd" list is limited to columns and because MySQL allows
    non-column expressions in GROUP BY (unlike the standard), we need this
    block _even_if_ this is not a column. For example:
      select d.s from
        (select b*3 as c, sum(a) as s from t1 group by b*3) as d
      group by d.c;
    ... In this corner case, a "source" can be a source _expression_, not
    column.
  */
  find_group_in_fd(item);

  if (!local_column) return;

  /*
    A column reference can later give more FDs, record it.
    ...
    Moreover, it is good to stick to the definition of what a
    functionally dependent column can be: it can be a view's column.
  */
  fd.push_back(down_cast<Item_ident *>(item));

  if (!add_to_mat_table) return;

  item = item->real_item();  // for merged view containing mat table
  if (item->type() == Item::FIELD_ITEM) {
    Item_field *const item_field = (Item_field *)item;
    Table_ref *const tl = item_field->field->table->pos_in_table_list;
    if (tl->uses_materialization() &&  // materialized table
        !tl->is_table_function())      // there's no underlying query expr
      add_to_source_of_mat_table(item_field, tl);   // ★ master → child 回传
  }
}
```

```cpp
void Group_check::find_group_in_fd(Item *item) {
  if (group_in_fd == ~0ULL) return;  // nothing to do
  if (select->is_grouped()) {
    /*
      See if we now have all of query expression's GROUP BY list; an
      implicitly grouped query has an empty group list.
    */
    bool missing = false;
    int j = 0;
    for (ORDER *grp = select->group_list.first; grp; ++j, grp = grp->next) {
      if (!(group_in_fd & (1ULL << j))) {
        Item *grp_item = *grp->item;
        if ((local_column(grp_item) &&
             (grp_item->used_tables() & ~whole_tables_fd) == 0) ||
            (item && grp_item->eq(item, false)))
          group_in_fd |= (1ULL << j);
        else
          missing = true;
      }
    }
    if (!missing) {
      /*
        All GROUP BY exprs are FD on the source. Turn all bits on, for easy
        testing.
      */
      group_in_fd = ~0ULL;    // ★ 一旦全满，聚合函数/窗口分组函数一律放行
    }
  }
}
```

**`Distinct_check::check_query()`**（`aggregate_check.cc:111`）——DISTINCT + ORDER BY 的"随机顺序"拒绝：

```cpp
/**
   Rejects the query if it has a combination of DISTINCT and ORDER BY which
   could lead to randomly ordered results. More precisely: if, in a query
   block 'sl', an ORDER BY expression
   - is not the same expression as one in the SELECT list of 'sl' (1)
   - and contains a column which:
   -- is of a table whose qualifying query block is 'sl'          (2)
   -- and is not in the SELECT list of 'sl'                       (3)
   then 'sl' should not have DISTINCT.

   @returns true if rejected (my_error() is called)
*/
bool Distinct_check::check_query(THD *thd) {
  uint number_in_list = 1;
  for (ORDER *order = select->order_list.first; order;
       ++number_in_list, order = order->next) {
    if (order->in_field_list)  // is in SELECT list
      continue;
    assert((*order->item)->fixed);
    uint counter;
    enum_resolution_type resolution;
    Item **res;
    /*
      Search if this expression is equal to one in the SELECT
      list. setup_order()/find_order_in_list() has already done so, but not
      perfectly, indeed at that time the expression was not fixed, which
      prevents recognition of certain equalities. Example:
        create view v select x*2 as b from ...;
        select distinct sin(b) as z from v order by sin(b);
      ...
      There is a limitation with subqueries: in this query
      SELECT (subquery) ... ORDER BY (subquery)
      we may not be able to recognize that both subqueries are the same ...
      It is because Item_subselect::eq() is Item::eq() which is too coarse ...
      Subqueries in ORDER BY are non-standard anyway.
    */
    if (find_item_in_list(thd, *order->item, &select->fields, &res, &counter,
                          &resolution)) {
      return true; /* purecov: inspected */
    }
    if (res != nullptr)  // is in SELECT list
      continue;
    /*
      [numbers refer to the function's comment]
      (1) is true. Check (2) and (3) inside walk().
    */
    if ((*order->item)
            ->walk(&Item::aggregate_check_distinct, walk_options,
                   (uchar *)this)) {
      if (failed_ident)
        my_error(ER_FIELD_IN_ORDER_NOT_SELECT, MYF(0), number_in_list,
                 failed_ident->full_name(), "DISTINCT");
      else
        my_error(ER_AGGREGATE_IN_ORDER_NOT_SELECT, MYF(0), number_in_list,
                 "DISTINCT");
      return true;
    }
  }
  return false;
}
```

> ★ 这段注释正是"已知边界 #6：`Item::eq()` 对含子查询的项不可靠"的出处——`Item_subselect::eq()` 退化成比较**名字**（子查询的原始文本），空白符不同就匹配不上。

##### 已知边界（源码自认）

1. **`group_in_fd` 只有 64 位** ⇒ GROUP BY 表达式上限 64 个（注释：`should probably be MY_BITMAP`）
2. **ROLLUP 完全放弃 FD 推导**——只保留"是否等于某个 GROUP BY 表达式"的检查
3. **HAVING 中的等值不推导 FD**：`It's possible to infer functional dependencies from equality conditions in HAVING, but we have not implemented it yet.`
4. **UNION / EXCEPT / INTERSECT / ROLLUP 的派生表无任何 FD**
5. **semi-join / antijoin nest 不参与 FD 发现**（等值已在别处复制）
6. **`Item::eq()` 对含子查询的项不可靠** ⇒ ORDER BY 里的相同子查询可能识别不出来而被误拒
7. **创建视图时不检查**（`simplify_joins()` 没跑），错误到视图被使用时才报
8. **5.6 兼容快路径**：先只比 GROUP BY 列表，命中就返回——注释明说这是为了让所有 5.6 接受的查询都还被接受

`ANY_VALUE()` 是官方逃生口——它让 walk 停在自身、不再下钻：

```cpp
bool Item_func_any_value::aggregate_check_group(uchar *arg) {
  Group_check *gc = reinterpret_cast<Group_check *>(arg);
  if (gc->is_stopped(this)) return false;
  gc->stop_at(this);
  return false;
}
```

---

## ★ 本机制里的工程实现技法

### 1. `ORDER_with_src`：把"来源"编码进数据结构

8.0 给 `ORDER` 链表包了一层 `ORDER_with_src`，带 `src`（`ESC_ORDER_BY` / `ESC_GROUP_BY` / `ESC_DISTINCT`）与 `const_optimized` 标志。这是一个典型的**为可观测性而做的数据结构设计**：排序项在优化期会被消、被改写、被合并，若不带来源标签，EXPLAIN 就无法解释"这个 filesort 到底是谁要求的"。

代价是每次改写都要手工带上 `src`（你能看到 `ORDER_with_src(o, ESC_DISTINCT)`、`ORDER_with_src(..., group_list.src, true)` 这类写法），漏传就丢失溯源。

### 2. `remove_const` 的双重身份

同一个 `remove_const` 函数，最后一个参数区分了两种用途：

- 用于 ORDER BY 时：`remove_const(order.order, where_cond, ..., &simple_order, false)`
- 用于 GROUP BY 时：`remove_const(group_list.order, where_cond, ..., &simple_group, true)`

最后一个 bool 是"是否要求保留顺序语义"。这导致同一个函数在两处产出不同的 `simple_*` 标志，而这两个标志后续又被 `test_skip_sort` 分别消费。**一个函数服务两个语义相近但不等价的需求**——这是 MySQL 里常见的"参数化复用"，也埋下了误用的可能。

### 3. 三个二态标志的耦合

`simple_group`、`simple_order`、`skip_sort_order` 三者互相影响：

```
skip_sort_order = true   →  test_skip_sort 里允许尝试 ORDER BY 索引
simple_group    = true   →  test_skip_sort 里允许尝试 GROUP BY 索引
simple_order/simple_group = false  →  强制临时表"不带排序"
```

它们不是独立的布尔，而是一个**三态状态机的投影**。读代码时若只盯着一个标志，很容易误判。

### 4. `count_field_types` 被反复调用

在 `optimize_distinct_group_order` 里，`count_field_types` 出现了两次（DISTINCT 改写的两个分支各一次）。原因是它**重算 `tmp_table_param` 的行宽与字段类型**——一旦 `fields` 或 `group_list` 变了，之前的估算就失效。这也意味着 `tmp_table_param` 在优化期是**可变状态**，不是一次性决定的。

### 5. 优化期执行查询

`optimize_aggregated_query` 是极少数**在优化期就真正读数据**的地方（`ha_records()`、全文检索计数）。这带来一个约束：它必须在事务/表打开之后，且必须处理"优化期拿到的值与执行期不一致"的风险（所以它只接受 `HA_STATS_RECORDS_IS_EXACT` 这种"保证精确"的来源）。

---

## 可观测性

### EXPLAIN Extra 倒排

| 你看到 | 对应哪条路径 | 定位方向 |
|---|---|---|
| Extra 为空（type=index） | ① 索引有序流式聚合 | `m_ordered_index_usage == ORDERED_INDEX_GROUP_BY` |
| `Using index for group-by` | ② 松索引扫描 | `GROUP_INDEX_SKIP_SCAN` |
| `Using temporary` | ③ 临时表聚合 | `need_tmp_before_win` 为真 |
| `Using temporary; Using filesort` | ③ + 排序，或 ④ | 看 `simple_group` / `simple_order` 是否被置 false |
| `Using filesort`（无 temporary） | ④ filesort + 流式聚合 | `allow_group_via_temp_table == false` |
| `Using index for group-by (scanning)` | LIS 的变体 | 见 [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) |

### optimizer trace 字段

`optimize_distinct_group_order` 里写入的 trace 节点：

| 字段 | 含义 |
|---|---|
| `optimizing_distinct_group_by_order_by` | 整个函数的包裹节点 |
| `distinct_is_on_unique` / `removed_distinct` | DISTINCT 被唯一索引消掉 |
| `changed_distinct_to_group_by` | DISTINCT 被改写成 GROUP BY |
| `removed_order_by` | ORDER BY 被子序列关系删掉 |

`test_skip_sort` 的判据则出现在 `test_if_skip_sort_order` 的 trace 里（详见 [`10_plan_refinement.md`](10_plan_refinement.md)）。

### 相关变量

| 变量 | 作用 |
|---|---|
| `tmp_table_size` / `max_heap_table_size` | 临时表从内存引擎转磁盘的**硬阈值**，不是代价项 |
| `SQL_BIG_RESULT` | 关掉"用索引做 GROUP BY"与临时表聚合 |
| `SQL_SMALL_RESULT` | 倾向于临时表聚合 |
| `ONLY_FULL_GROUP_BY`（sql_mode） | prepare 期检查，不属优化期 |

---

## Misc

### 扩展点：改这一块要动哪些地方

| 想做什么 | 要动的地方 |
|---|---|
| 新增一种分组实现 | `test_skip_sort` 的判定 + `make_tmp_tables_info` 的分派 + 一个新的 `RowIterator` + 对应 `AccessPath` 类型 + `explain_extra` 的输出 |
| 让 GROUP BY 支持代价决策 | 需要在 `test_skip_sort` 之后插入代价比较，但当前结构是布尔判定，改造面很大（这也是为什么 MySQL 至今没做） |
| 新增一个"禁止临时表聚合"的聚合函数 | 在该 `Item_sum` 子类构造里置 `allow_group_via_temp_table = false` |
| 让 DISTINCT 改写支持更多形状 | `create_order_from_distinct` + 第三章 3.3 的判定式 |
| 新增一个免排序的索引类型 | `keys_in_use_for_group_by` / `keys_in_use_for_order_by` 这两个 `Key_map` 的填充处 |

### 已知缺陷与源码 TODO

- `test_skip_sort` 里那句 `TODO: Explain the allow_group_via_temp_table part of the test below` 至今未解决——作者自己都承认这个判定式难以解释。
- `SQL_BIG_RESULT` 会让某些查询**变慢**（把索引扫描换成全表扫 + filesort），这是语义与实现错位的后果，社区无修正。
- `optimize_aggregated_query` 对外连接的处理是保守的（只要 WHERE 碰了 inner table 就放弃），理论上可以更精细。
- `JOIN::optimize` 里 `count_field_types` 反复调用是历史遗留，注释里有 `@todo WL#6570` 系列的清理计划。

### 社区边界澄清

以下内容**社区版 MySQL 没有**，不要与云厂商/分支特性混淆：

- **没有哈希聚合（HashAggregate）的代价选择**：临时表聚合用的是哈希索引临时表，但选择它是规则而非代价
- **没有分组并行**：`GROUP BY` 不会并行执行（InnoDB 的并行扫描只用于 `CHECK TABLE` 与 DDL、以及主键并行读，不用于 GROUP BY）
- **没有向量化聚合**：聚合是逐行 `Item_sum::add()` 调用，无批处理（HeatWave 有，社区版没有）
- **没有多列统计/扩展统计**：选择率估算假设列独立，见 [`01_cost_model.md`](01_cost_model.md)
- **没有 `COUNT(*)` 的精确计数缓存**：InnoDB 的 `stats.records` 是采样估计，不提供 `HA_STATS_RECORDS_IS_EXACT`

### 路径边界

hypergraph 优化器（`optimizer_switch='hypergraph_optimizer=on'`）在 `JOIN::optimize` 里**提前 return**，完全不执行本篇的 `optimize_distinct_group_order` / `test_skip_sort` / `make_tmp_tables_info`。它的等价机制在 `join_optimizer/finalize_plan.cc` 与 interesting orders 里（见 [`physical/09_hypergraph.md`](physical/09_hypergraph.md) 与 [`physical/18_hypergraph_advanced.md`](physical/18_hypergraph_advanced.md)）。`optimize_aggregated_query` 是**唯一两边共用**的一步。

---

## 参考

**论文 / 理论**

- **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979)** —— System R 的 interesting order 概念。MySQL 只实现了一个退化版本（`m_ordered_index_usage` 三态，一个表集合只保留一个顺序候选）
- **Simmen, Shekita, Malkemus《Fundamental Techniques for Order Optimization》(EDBT 1996)** —— DB2 的顺序优化：sort 下推、最小化排序次数、用函数依赖推导顺序。MySQL 未实现
- **Neumann & Moerkotte《A Combined Framework for Grouping and Order Optimization》(VLDB 2004)** ★ —— **顺序与分组的联合推理框架**，论文指出此前没有完整/正确/高效的算法。这是评判 MySQL 分组实现（纯规则）的参照系
- **Graefe《Query Evaluation Techniques for Large Databases》(ACM Computing Surveys 1993)** —— 排序与聚合的算法分类（sort-based vs hash-based aggregation）、谓词下推、物化策略。本篇第三章"四种实现"的理论出处
- **Graefe《Volcano—An Extensible and Parallel Query Evaluation System》(1990)** —— 流式聚合与排序的物理算子分类（对应 `AggregateIterator` / `SortingIterator`）
- **Graefe《Sorting and Database Systems》(Foundations and Trends in Databases, 2006)** —— 排序的专著；本篇"先 filesort 再流式聚合"这条路径的代价分析可对照此文
- （对照，非论文）PostgreSQL 的 `GroupAggregate` / `HashAggregate` 选择是**代价驱动**，实现见 PG 源码 `optimizer/plan/planner.c` 与 `optimizer/path/costsize.c`。列出它只是为了说明：MySQL 的"分组实现由规则决定"是设计取舍，不是唯一的做法

**关于 Loose Index Scan 的说明（重要）**

`GROUP_INDEX_SKIP_SCAN`（松索引扫描 / MIN-MAX）在 MySQL 与 Oracle（INDEX SKIP SCAN）里是**工程实践**，不是学术论文的产物。查不到对应的经典论文——它的本质是"利用索引有序性 + 分组键的跳跃式扫描 + 端点探测"，最接近的理论基础是 Selinger 1979 的 sargable predicate 与 Graefe 1993 的索引扫描分类。写文档/做分享时不要给它硬塞一个论文出处。

**WorkLog**

- WL#2489 "better only_full_group_by" —— `aggregate_check.h` 里实现的可选标准特性 T301，早于本篇的优化期
- WL#6570 "prepare once, execute many" —— 源码中散落的 `@todo`，涉及 `count_field_types` 反复调用等历史遗留

**官方文档**

- MySQL 8.0 Reference Manual, "Optimizing GROUP BY" / "DISTINCT Optimization" / "ORDER BY Optimization"
- MySQL 8.0 Reference Manual, "Loose Index Scan"

**相关文档**

- [`10_plan_refinement.md`](10_plan_refinement.md) —— 排序选择的判定算法
- [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md) —— `GROUP_INDEX_SKIP_SCAN` 的区间构造
- [`../runtime/06_rollup.md`](../runtime/06_rollup.md) —— ROLLUP 与聚合器的运行期状态机
- [`../runtime/07_temptable.md`](../runtime/07_temptable.md) —— 临时表引擎
- [`logical/03_semijoin.md`](logical/03_semijoin.md) —— LooseScan 策略与 GROUP BY 的耦合
- [`../13_item_expression.md`](../13_item_expression.md) —— `Item_sum` 的聚合状态机
