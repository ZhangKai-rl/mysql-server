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

#### 2.4 COUNT(expr) 的常量化条件

```
if (conds == nullptr                        // 没有 WHERE
    && !arg0->is_nullable()                 // COUNT 的参数不可能为 NULL
    && !inner_tables                        // 没有外连接的 inner table
    && tables_filled)                       // 表都已填充
```

四条件全部满足，`COUNT(expr)` 才能换成"表的笛卡尔积行数"。**`arg0->is_nullable()` 这一条**是语义正确性的关键：`COUNT(expr)` 不计 NULL，只有参数非空时才等于行数。这也是 `COUNT(*)` 通常比 `COUNT(col)` 快的原因之一——`COUNT(*)` 的参数是常量 `1`，天然非空。

另有一条全文检索特例：单表 + WHERE 是单个 `MATCH` + 引擎能提供 FTS 结果行数 + 参数非空 → **优化期直接跑全文检索**拿到真实 count 并替换。

---

### 三、optimize_distinct_group_order 逐段剖析

> 位于 `sql/sql_optimizer.cc`。这是本篇的核心，函数约 180 行，但每一段都在改一个全局标志。

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

**为什么是这个顺序？** 因为它按"成本递增 + 输出属性递减"排列：

- ② 最便宜（只读索引端点），但只适用于极窄的查询形状；
- ① 次之（一次索引扫描，但读了全部行），且输出有序；
- ③ 需要物化全部行，但能处理任意分组键；
- ④ 最贵（全量排序 + 流式聚合），只有在临时表被显式禁止时才选。

注意 ④ 在成本上通常**不低于** ③，它只是"临时表被禁用"时的兜底。这正是 `SQL_BIG_RESULT` 可能让查询变慢的原因。

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

---

### 七、与 ROLLUP / 窗口 / HAVING / ONLY_FULL_GROUP_BY 的交互

| 特性 | 优化期的影响 |
|---|---|
| **ROLLUP** | `JOIN::optimize_rollup` 强制 `allow_group_via_temp_table = false`，ROLLUP 只能流式聚合；同时 `optimize_distinct_group_order` 里 `query_block->olap == ROLLUP_TYPE` 会阻止"GROUP BY 顺带删 ORDER BY" |
| **窗口函数** | `m_windows_sort` 为真时不允许用 GROUP BY 的顺序去消 ORDER BY（窗口会重排）；窗口函数强制 `need_tmp_before_win`，即必须先物化 |
| **HAVING** | 在 `optimize_distinct_group_order` 之前被 `optimize_cond` 处理；HAVING 引用聚合函数时不能下推到 WHERE（见 [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md)） |
| **ONLY_FULL_GROUP_BY** | 在 `Query_block::prepare` 的 `Aggregate_check` 里检查（`aggregate_check.h`，WL#2489），早于优化期。优化期看到的是**已通过检查**的查询 |
| **DISTINCT + ORDER BY** | 若 ORDER BY 的列不在 SELECT list 中，`create_order_from_distinct` 的 `all_order_fields_used` 为 false，导致改写被拒，只能走临时表去重 |

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
