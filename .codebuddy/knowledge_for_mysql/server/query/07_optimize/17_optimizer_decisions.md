# 17 优化器决策全景：每一个决定是怎么做出来的

> 基于 MySQL 8.0.39。本篇从**决策**视角横向贯穿优化器：把 `prepare` 与 `optimize` 里所有"二选一 / 多选一"的判决定点列成表，标明每个决策的**性质（规则 / 代价 / 硬编码）**、**判据函数**、**产出**、**能否被覆盖**，并深挖三个最典型的决策。
>
> **边界**：本篇是**横向索引 + 三个深挖**。每个决策的算法细节不在这里：
> - 子查询与 semi-join 改写决策 → [`logical/02_subquery.md`](logical/02_subquery.md)、[`logical/03_semijoin.md`](logical/03_semijoin.md)
> - 连接简化 / derived merge → [`logical/04_logical_join.md`](logical/04_logical_join.md)
> - 谓词改写 → [`logical/05_logical_predicate.md`](logical/05_logical_predicate.md)
> - join order 搜索 → [`physical/06_join_order.md`](physical/06_join_order.md)
> - 访问方法代价 → [`physical/07_access_method.md`](physical/07_access_method.md)
> - range 优化 → [`physical/08_range_optimizer.md`](physical/08_range_optimizer.md)
> - 排序/临时表 → [`10_plan_refinement.md`](10_plan_refinement.md)、[`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)
> - hint 的语法与对象树 → [`11_optimizer_hints.md`](11_optimizer_hints.md)
> - 代价与统计 → [`01_cost_model.md`](01_cost_model.md)

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [一、决策全景表（prepare 期）](#一决策全景表prepare-期)
  - [二、决策全景表（optimize 期：规划）](#二决策全景表optimize-期规划)
  - [三、决策全景表（optimize 期：收尾与代码生成）](#三决策全景表optimize-期收尾与代码生成)
  - [四、深挖 A：semi-join 执行策略的选择](#四深挖-asemi-join-执行策略的选择)
  - [五、深挖 B：访问方法的选择](#五深挖-b访问方法的选择)
  - [六、深挖 C：join 算法的选择](#六深挖-cjoin-算法的选择)
  - [七、决策的三种失败模式](#七决策的三种失败模式)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 什么是"决策点"

优化器不是一条流水线，而是一串**分叉**。每个分叉处它必须在若干等价（或近似等价）的方案里选一个：

- 这个子查询要不要转 semi-join？
- 转了之后用哪种执行策略？
- 这张表用 ref 还是 range 还是全表扫？
- 两张表用 NLJ 还是 HashJoin？
- GROUP BY 用索引还是临时表？

现有各篇是按**算法**组织的（讲 `best_access_path` 怎么算代价），本篇按**决策**组织（讲"在哪个分叉、依据什么、选了什么、能不能改"）。两者的关系是：算法篇回答"怎么算"，本篇回答"在哪算、算完怎么用"。

### 决策的三种性质

这是理解 MySQL 优化器行为的关键分类：

| 性质 | 依据 | 特点 | 典型 |
|---|---|---|---|
| **规则**（RBO） | 满足条件就改/就选，不比代价 | 可预测，但对数据分布不敏感 | 外连接转内连接、derived merge、等值传播 |
| **代价**（CBO） | 比谁便宜 | 依赖统计，统计不准就选错 | ref vs range vs scan、join order |
| **硬编码常量** | 拍的常数 | 完全没有依据，纯兜底 | 无统计时的选择率 `0.1` / `0.3333` / `0.1111` |

★ 关键认知：**MySQL 是三者的混合体**，而且分类边界常常不按"直觉"划分。例如：

- **子查询改写是纯规则的**（满足条件就转 semi-join），但**转了之后用哪种执行策略是代价的**；
- **join order 搜索是代价的**，但**剪枝是规则的**（`pruned_by_heuristic` 会砍掉代价更优的计划，源码注释明说 "may miss the optimal QEPs"）；
- **GROUP BY 的实现选择是纯规则的**（见 [`15_groupby_distinct_order.md`](15_groupby_distinct_order.md)），代价模型完全不参与。

这解释了为什么"优化器不听话"往往不是代价算错，而是**那个决策根本就没走代价**。

### 可覆盖性谱系

当优化器选错时，DBA 能干预的强度分四档：

```
最强  optimizer hint（/*+ ... */）    → 针对单个 query block / 表 / 索引
  ↑   optimizer_switch                → 会话/全局级的开关
  ↑   系统变量（代价常量、内存上限）    → 影响代价计算的输入
最弱  不可覆盖                         → 只能改 SQL 写法或改数据分布
```

★ 注意一个反直觉事实：**hint 也不是万能的**。很多决策（如 GROUP BY 实现、semi-join 准入 16 条件）没有任何 hint 能覆盖，只能靠改写法。这也是 [`14_plan_stability.md`](14_plan_stability.md) 里"社区版无 plan cache / 无 SPM"的直接后果。

---

## 理论基础

### 决策点为什么分布成三段

```
prepare（Query_block::prepare）
   └─ 逻辑改写类决策：改的是查询结构本身，必须早做，因为会影响后续所有估算
optimize 前段（optimize_cond / optimize_aggregated_query）
   └─ 不需要代价的化简：常量化、分区裁剪、生成列替换
optimize 中段（make_join_plan）
   └─ 需要代价的决策：join order、访问方法、semi-join 策略、join 算法
optimize 后段（计划改进 / 代码生成）
   └─ 依赖前段结论的收尾决策：排序、临时表、ICP、下推
```

这个分布不是随意的，而是由**依赖关系**决定的：

- 逻辑改写必须最早，因为它改变"表与条件"的形态，后续所有估算都基于改写后的形态；
- 代价决策必须在统计信息就绪、且候选空间确定之后；
- 收尾决策必须在 join order 定下来之后（例如"能否用索引免排序"要看第一张表是谁）。

**一个重要的例外**：`substitute_for_best_equal_field`（等价类展开成星形等值）属于逻辑改写，却发生在 `make_join_plan` **之后**——因为"哪个字段作为等价类的代表"依赖 join 顺序。源码注释明确指出了这一点。

### 规则与代价的边界为什么模糊

理论上应该是"规则生成候选、代价挑最优"（Cascades 范式）。但 MySQL 里：

1. **规则自己做了本该由代价做的主**（例如 semi-join 准入的 16 条里，有些是纯性能启发式）
2. **代价被规则提前剪掉**（`pruned_by_heuristic`）
3. **有些决策干脆不比代价**（GROUP BY、join 算法里的部分分支）

结果是：**同一个查询，改数据分布可能变计划，改写法也可能变计划，而且两者的原因不同**。定位时要先分清"这个决策是规则还是代价"，否则会往错的方向查。

### 他库对比

| 库 | 决策组织方式 |
|---|---|
| **Cascades 系**（SQL Server、Calcite、Orca） | 统一的规则引擎 + 代价搜索，决策点由规则声明，"什么时候用代价"是框架决定的 |
| **PostgreSQL** | 逻辑改写（`transformStmt`/`preprocess`）与代价规划（`Planner`）分离较干净；分组实现（`HashAgg` vs `GroupAgg`）是**代价决策** |
| **MySQL** | 决策散落在 `prepare` + `optimize` 的各函数里，规则/代价/硬编码混用，无统一框架 |

这就是 [`14_plan_stability.md`](14_plan_stability.md) 里"MySQL 优化器能力边界"的根源：没有框架，就无法系统地做"多计划比较 / 计划回归检测"。

---

## 核心实现

### 一、决策全景表（prepare 期）

| # | 决策 | 性质 | 判据（函数） | 产出 | 可覆盖 |
|---|---|---|---|---|---|
| 1 | 子查询：转 semi-join / anti-join / IN→EXISTS / 物化 / 转 derived | 规则 + 代价 | `resolve_subquery`（16 条准入 + 物化代价公式） | `Subquery_strategy` | `semijoin` / `materialization` / `subquery_materialization_cost_based` / `subquery_to_derived` |
| 2 | derived / view：merge 还是物化 | 规则 | `merge_derived`（12 步） | merged 或 materialized | `derived_merge` |
| 3 | 外连接能否转内连接 | 规则 | `simplify_joins`（NULL-rejecting 判定） | 改写后的 join nest | 不可覆盖（改写法） |
| 4 | join nest 扁平化 | 规则 | `simplify_joins` Pass 2 | Table_ref 树 | 不可覆盖 |
| 5 | 能否下推条件到 derived | 规则 | `derived_condition_pushdown` 相关 | 下推的 WHERE | `derived_condition_pushdown` |
| 6 | 常量折叠（能否求值成常量） | 规则 | `resolve_const_item` | 常量 Item | 不可覆盖 |
| 7 | 类型聚合（比较/运算的结果类型） | 规则 | `Aggregate_type` / `agg_result_type` | 每个 Item 的 `data_type` | 不可覆盖（改写法） |
| 8 | ONLY_FULL_GROUP_BY 合法性 | 规则 | `Aggregate_check`（WL#2489） | 报错或放行 | sql_mode |
| 9 | 窗口函数能否合并/排序复用 | 规则 | `setup_windows1/2` | Window 对象 | 不可覆盖 |
| 10 | 分区裁剪（prepare 期） | 规则 | `prune_table_partitions` | 分区位图 | 不可覆盖 |
| 11 | 生成列能否替换表达式 | 规则 | `substitute_gc` | 替换后的 Item | 不可覆盖 |

### 二、决策全景表（optimize 期：规划）

| # | 决策 | 性质 | 判据（函数） | 产出 | 可覆盖 |
|---|---|---|---|---|---|
| 12 | 聚合能否常量化（整个 JOIN 消掉） | 规则 | `optimize_aggregated_query`（三态） | `AGGR_COMPLETE` / `AGGR_DELAYED` / `AGGR_REGULAR` | 不可覆盖 |
| 13 | 条件化简（等值传播/常量传播/去恒真恒假） | 规则 | `optimize_cond` | 改写后的 WHERE/HAVING | 不可覆盖 |
| 14 | 哪些表是 const 表 | 规则 + 引擎 | `extract_const_tables` | `const_tables` | 不可覆盖 |
| 15 | 生成哪些 Key_use（可用的索引查找弧） | 规则 | `update_ref_and_keys` | `keyuse_array` | `use_index_extensions`、`use_invisible_indexes` |
| 16 | **join order** | 代价（+规则剪枝） | `choose_table_order` / `best_extension_by_limited_search` | `best_ref` 顺序、`best_positions` | `STRAIGHT_JOIN`、`JOIN_ORDER`/`JOIN_PREFIX` hint |
| 17 | **每张表的访问方法**（const/ref/eq_ref/range/scan/skip scan/index merge） | 代价（+规则短路） | `best_access_path` | `JOIN_TAB::type()`、`POSITION` | `FORCE/USE/IGNORE INDEX`、`MRR`、`NO_RANGE_OPTIMIZATION`、`skip_scan` |
| 18 | 是否做二次 range 重估 | 规则 | `can_switch_from_ref_to_range` | 重估后的 range 计划 | 不可覆盖 |
| 19 | **semi-join 执行策略**（6 选 1） | 规则 + 代价 | `advance_sj_state` / `fix_semijoin_strategies` | `POSITION::sj_strategy` | `firstmatch` / `loosescan` / `duplicateweedout` / `materialization` |
| 20 | 是否用 index merge | 代价 | range 优化器（索引合并的贪心搜索） | `QUICK_INDEX_MERGE_SELECT` | `index_merge` / `index_merge_union` / `index_merge_sort_union` / `index_merge_intersection` |
| 21 | 是否用 range / skip scan | 代价 | `get_best_disjunct_quick` 等 | `QUICK_RANGE_SELECT` / `IndexSkipScanIterator` | `skip_scan`、`range_optimizer_max_mem_size` |
| 22 | condition filtering（过滤系数） | 硬编码 + 统计 | `calculate_condition_filter` | `POSITION::filter_effect` | `condition_fanout_filter` |
| 23 | 等价类代表字段（星形等值展开） | 规则（依赖 join order） | `substitute_for_best_equal_field` | 改写后的等值条件 | 不可覆盖 |
| 24 | LooseScan / skip scan 的候选键补充 | 规则 | `add_loose_index_scan_and_skip_scan_keys` | 扩充的 `Key_map` | `loosescan` / `skip_scan` |

### 三、决策全景表（optimize 期：收尾与代码生成）

| # | 决策 | 性质 | 判据（函数） | 产出 | 可覆盖 |
|---|---|---|---|---|---|
| 25 | **join 算法**（NLJ / BNL / BKA / HashJoin） | 规则 + 代价 | `setup_join_buffering` | `JOIN_CACHE` 类型 / HashJoin | `block_nested_loop` / `batched_key_access` / `hash_join` / `mrr` / `mrr_cost_based` / `BNL`/`BKA`/`NO_BNL` hint |
| 26 | DISTINCT / GROUP BY / ORDER BY 的化简与改写 | 规则 | `optimize_distinct_group_order` | 一组标志 | 不可覆盖（见 15 篇） |
| 27 | 能否用索引顺序免排序（GROUP BY 或 ORDER BY 二选一） | 规则 | `test_skip_sort` / `test_if_skip_sort_order` | `m_ordered_index_usage` | `prefer_ordering_index`、`SQL_BIG_RESULT` |
| 28 | filesort 还是索引（ORDER BY 场景的代价比较） | 代价 | `test_if_cheaper_ordering` | `skip_sort_order` | `prefer_ordering_index` |
| 29 | 删掉冗余谓词 | 规则 | `finalize_table_conditions` | 精简后的条件 | 不可覆盖 |
| 30 | 是否 ICP（索引条件下推） | 规则（8 项判定） | `push_index_cond` | 推给引擎的条件 | `index_condition_pushdown` |
| 31 | 是否引擎条件下推 | 规则 | `push_to_engines` | 推给引擎的条件 | `engine_condition_pushdown` |
| 32 | 是否要临时表、建什么键、用什么 op_type | 规则 | `make_tmp_tables_info` | `QEP_TAB::op_type`、临时表 | `SQL_BIG_RESULT` / `SQL_SMALL_RESULT`、`tmp_table_size` |
| 33 | 是否 join buffering 及 buffer 大小 | 规则 + 代价 | `setup_join_buffering` | `JOIN_CACHE` 参数 | `join_buffer_size` |
| 34 | 是否用二级引擎（卸载） | 代价 + 接口 | `secondary_engine_cost_threshold` 对比 | 二级引擎计划或回退 | `secondary_engine_cost_threshold`、SQL 的 `USE SECONDARY ENGINE` 子句 |
| 35 | AccessPath 的构造形态 | 规则 | `create_access_paths` | AccessPath 树 | 不可覆盖 |
| 36 | 迭代器类型（含 materialize 的惰性/即时） | 规则 | `CreateIteratorFromAccessPath` | RowIterator 树 | 不可覆盖 |

★ 表里的"可覆盖"列是最实用的一列：它直接告诉你"优化器选错时我能怎么办"。

---

### 四、深挖 A：semi-join 执行策略的选择

> 这是本篇最值得深挖的决策，因为它**同时**用到规则与代价，且有一个很特别的实现妥协。

#### 4.1 六态枚举

`POSITION::sj_strategy` 取值（`sql/sql_select.h`）：

| 值 | 策略 | 执行器载体 |
|---|---|---|
| `SJ_OPT_NONE` | 未启用（还在 nest 内） | — |
| `SJ_OPT_DUPLICATE_WEEDOUT` | 用临时表唯一键去重 | `SJ_TMP_TABLE` |
| `SJ_OPT_LOOSE_SCAN` | 索引跳扫去重 | 索引扫描 + 分组跳过 |
| `SJ_OPT_FIRST_MATCH` | 找到第一个匹配就返回 | `return_tab` 短路 |
| `SJ_OPT_MATERIALIZE_LOOKUP` | 物化后查表去重 | 物化临时表 + 索引查找 |
| `SJ_OPT_MATERIALIZE_SCAN` | 物化后扫描去重 | 物化临时表 + 全扫 |

注意 `MATERIALIZE_LOOKUP` 与 `MATERIALIZE_SCAN` 是**两种**策略（是否能在物化表上建索引做查找），这解释了为什么常说"五策略"却看到 6 个枚举值。

#### 4.2 决策发生在 join order 搜索的每一步

`advance_sj_state()` 被 `best_extension_by_limited_search` 在**每加入一张表时**调用一次。也就是说：**semi-join 策略不是单独决定的，而是与 join order 一起被搜索出来的**——不同的前缀长度会激活不同的策略。

这带来一个关键约束（源码注释原文）：

```
We don't yet know what are the other strategies, so pick FirstMatch.

We ought to save the alternate POSITIONs produced by
semijoin_firstmatch_loosescan_access_paths() but the problem is that
providing save space uses too much space.
Instead, we will re-calculate the alternate POSITIONs after we've
picked the best QEP.
```

**这就是那个实现妥协**：搜索过程中，为了省内存，只保留"当前最优策略"，不保留各策略的备选 `POSITION`。等 join order 定下来后，再由 `fix_semijoin_strategies()` **重新计算一遍**访问路径、行数与代价。

后果：

- 搜索期与最终期的计算**必须完全一致**，否则计划会自相矛盾（这也是这个函数历史上出过 bug 的地方）；
- 无法做"跨策略的全局最优"比较——每一步只比当前这一步。

#### 4.3 三个策略的准入条件（节选）

| 策略 | 准入（规则部分） | 代价部分 |
|---|---|---|
| **FirstMatch** | 所有依赖的外层表必须已在 join 前缀中；多个 semi-join nest 的表若交织则合并成一次 FirstMatch | `semijoin_firstmatch_loosescan_access_paths()` 算 rowcount + cost |
| **LooseScan** | 依赖的外层表**不能**在前缀中；表序必须严格为：LooseScan 驱动表 → 同 nest 的其余表 → 外层依赖表；**不允许其它 semi-join 的表插进来** | 同上 |
| **Materialize** | `optimize_semijoin_nests_for_materialization` 先决定哪些 nest 值得物化；`calculate_materialization_costs` 算代价 | 物化代价 vs 不物化 |
| **Duplicate Weedout** | 兜底策略，几乎总能做 | 去重行数估算 |

`advance_sj_state` 里对 LooseScan 的那段注释值得引用：

```
LooseScan strategy can't handle interleaving between tables from
the semi-join that LooseScan is handling and any other tables.
```

一旦检测到交织，直接把 `first_loosescan_table` 置 `MAX_TABLES`（即放弃 LooseScan）。这解释了"为什么 LooseScan 经常不生效"——它对 join 顺序的要求太严。

#### 4.4 可观测

optimizer trace 里对应节点是 `semijoin_strategy_choice`，每个策略一个子节点，含 `strategy` / `cost` / `rows` / `chosen` 四个字段。想知道"为什么选了 FirstMatch 而不是 LooseScan"，看这一节就能看到 LooseScan 是被准入条件否掉、还是被代价否掉。

---

### 五、深挖 B：访问方法的选择

> 算法细节见 [`physical/07_access_method.md`](physical/07_access_method.md)，这里只讲"决策"层面。

`best_access_path` 的结构是**"四条规则短路 + 一次代价决斗"**：

```
1. const 表检测（唯一键等值）        → 规则，直接判定，不算代价
2. 无可用 Key_use 且无 range         → 规则，直接全表扫
3. 启发式短路（如 found_const / 强制索引 / 极小表） → 规则
4. range 分析（get_mm_tree → SEL_TREE）→ 代价（可能回退到全表扫）
5. ref vs range vs scan 的代价决斗   → 代价
```

其中的关键点是：**range 是被单独评估、然后拿回来与 ref/scan 比价的**（`can_switch_from_ref_to_range` 负责二次重估）。也就是说 range 优化器不是"另一个决策"，而是"给决策 5 提供一个候选"。

**为什么这个决策容易选错**：

- 它依赖 `records_in_range`（index dive）或 `rec_per_key`（统计估算）得到行数；两者都可能失真（见 [`01_cost_model.md`](01_cost_model.md)）；
- 它对多列条件做了**列独立的假设**，联合分布相关时严重失准；
- 长 IN list 会触发内存闸（`range_optimizer_max_mem_size`）降级，此时 range 候选直接消失，于是退化成全表扫——这类"突然变慢"的现象根因在这里。

---

### 六、深挖 C：join 算法的选择

> 由 `setup_join_buffering` 完成（`sql/sql_optimizer.cc`），发生在 `optimize_distinct_group_order` 之后。

MySQL 8.0 的 join 算法只有四个（**没有 merge join**，见 [`../09_executor_iterator.md`](../09_executor_iterator.md)）：

| 算法 | 触发条件 | 开关 |
|---|---|---|
| **NLJ**（嵌套循环，无缓冲） | 默认 | — |
| **BNL**（块嵌套循环，join buffer） | 内层无索引可用且 `block_nested_loop=on` | `block_nested_loop`、`join_buffer_size` |
| **BKA**（批量键访问，MRR 优化） | 内层有索引 + `batched_key_access=on` + MRR 可用 | `batched_key_access`、`mrr`、`mrr_cost_based`、`read_rnd_buffer_size` |
| **HashJoin** | 等值连接条件、无索引可用（8.0.18+） | `hash_join` |

决策要点：

1. **HashJoin 取代了 BNL 的默认地位**：理论上 hash join 在各维度不劣于 BNL（见 [`../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md)），所以 8.0 里 BNL 基本只剩"hash_join=off"时的退路。
2. **BKA 与 HashJoin 不冲突**：BKA 只适用于**内层有索引**的场景（把随机索引查找变成按 rowid 排序的顺序查找，即 DS-MRR）；HashJoin 用于**内层无索引**。二者适用面互补。
3. **`mrr_cost_based`** 决定 MRR 是"代价驱动"还是"总是用"。
4. `no_jbuf_after` 参数保证"某些策略之后不能再加 join buffer"（例如 semi-join 的 FirstMatch 之后），这是跨决策的约束。

---

### 七、决策的三种失败模式

定位"优化器选错了"时，先判断属于哪一类：

#### 模式 1：代价算错（输入失真）

| 症状 | 根因 | 查什么 |
|---|---|---|
| 选了全表扫而不是索引 | 行数估算失真（dive vs 统计） | `EXPLAIN FORMAT=json` 的 `rows_examined_per_scan`、`optimizer trace` 的 `rows_estimation` |
| 行数估算差几个数量级 | 无统计 / 统计过期 / 列相关 | `ANALYZE TABLE`、`SHOW INDEX`（`Cardinality`）、直方图 |
| 突然变慢（计划突变） | 统计自动重算触发（变更超 10%） | `innodb_stats_auto_recalc`、持久统计表 |

#### 模式 2：规则边界（根本没走代价）

| 症状 | 根因 | 查什么 |
|---|---|---|
| semi-join 没生效 | 16 条准入不满足 | trace 里的 `semijoin_strategy_choice`、`transformation` 节点 |
| derived 没 merge | 12 步里某步不满足（如含聚合、UNION、LIMIT） | trace 的 `derived_merge` / `materialization` 节点 |
| GROUP BY 走了临时表 | 规则判定，代价不参与 | 15 篇第五章判定图 |
| 外连接没转成内连接 | 没有 NULL-rejecting 谓词 | [`logical/04_logical_join.md`](logical/04_logical_join.md) |

★ 这类问题的特征是：**改数据、刷新统计都没用**，只有改写法才有用。

#### 模式 3：开关/变量被误配

| 症状 | 可能原因 |
|---|---|
| 全局变慢 | 有人关了 `hash_join` 或 `condition_fanout_filter` |
| 某些查询突然用不了索引 | `range_optimizer_max_mem_size` 太小导致长 IN list 降级 |
| 索引不可见却没生效 | `use_invisible_indexes` |
| 排序变多 | `prefer_ordering_index=off` |

---

## ★ 本机制里的工程实现技法

### 1. "先选了再说，事后重算"：semi-join 策略的省内存妥协

`advance_sj_state` 明知"后面还有别的策略可能更优"，仍然先选 FirstMatch，理由是**保存备选 `POSITION` 太占空间**。等 join order 定下来后再用 `fix_semijoin_strategies()` 重算。

代价是：搜索期与最终期的计算必须严格一致，且无法做跨策略的全局比较。这是一个典型的"**用重算换内存**"的取舍。

### 2. `optimizer_switch` 的 flagset 实现

`optimizer_switch` 是一个 `Sys_var_flagset`：一个字符串到 bitmask 的映射表（`optimizer_switch_names[]` 数组），**数组顺序必须与 `sql_const.h` 里的 `#define` 顺序一致**（源码注释用大写写了 `BEWARE!`）。

这是 C 时代"位标志 + 名称表"双份定义的经典隐患：新增开关必须同时改两处，顺序错了就是静默的行为错乱。

### 3. 决策点的"双重入口"：hint 与 switch

同一个决策往往有**两个覆盖通道**：

- `optimizer_switch`（会话级，影响所有查询）
- optimizer hint（`/*+ ... */`，只影响指定 query block）

实现上，hint 最终也是去改这些标志（在 `opt_hints.cc` 里），但作用范围被限定到 query block / 表级。见 [`11_optimizer_hints.md`](11_optimizer_hints.md)。

### 4. 用 `MAX_TABLES` 作为"无效值"哨兵

`first_firstmatch_table = MAX_TABLES`、`first_loosescan_table = MAX_TABLES` —— 用"超出范围的下标"表示"未启用"，而不是引入一个额外的 bool。好处是判定式只需一次比较，坏处是含义要靠注释才能读懂。

### 5. 决策的可追溯性靠 trace 而不是日志

MySQL 为每个决策在 optimizer trace 里留了节点（`semijoin_strategy_choice`、`rows_estimation`、`considered_access_paths`、`optimizing_distinct_group_by_order_by`…）。这是"无框架"优化器唯一的自证手段——因为没有统一规则引擎，只能靠手工埋点。

---

## 可观测性

### optimizer_switch 完整清单（26 项）

> 按源码 `optimizer_switch_names[]` 的声明顺序。默认全 on 的标 ✅，默认 off 的标 ❌（以 8.0.39 为准，逐项见 `sys_vars.cc` 的默认值定义）。

| 开关 | 影响的决策 |
|---|---|
| `index_merge` | 是否考虑 index merge（总闸） |
| `index_merge_union` | UNION 型 index merge |
| `index_merge_sort_union` | sort-union 型 index merge |
| `index_merge_intersection` | 交集型 index merge |
| `engine_condition_pushdown` | 条件下推到引擎 |
| `index_condition_pushdown` | ICP |
| `mrr` | MRR（多范围读） |
| `mrr_cost_based` | MRR 是否代价驱动 |
| `block_nested_loop` | BNL |
| `batched_key_access` | BKA |
| `materialization` | 半连接/子查询物化 |
| `semijoin` | semi-join 转换总闸 |
| `loosescan` | LooseScan 策略 |
| `firstmatch` | FirstMatch 策略 |
| `duplicateweedout` | Duplicate Weedout 策略 |
| `subquery_materialization_cost_based` | 物化是否代价驱动 |
| `use_index_extensions` | 索引扩展（二级索引隐含主键列） |
| `condition_fanout_filter` | condition filtering |
| `derived_merge` | derived/view 合并 |
| `use_invisible_indexes` | 是否使用 `INVISIBLE` 索引 |
| `skip_scan` | skip scan |
| `hash_join` | Hash Join |
| `subquery_to_derived` | 标量/IN 子查询转 derived |
| `prefer_ordering_index` | 优先用有序索引免排序 |
| `hypergraph_optimizer` | hypergraph 优化器（**源码注释明说"故意不写进文档"**） |
| `derived_condition_pushdown` | 条件下推到派生表 |

### 倒排速查：我想干预什么 → 用哪个开关/hint

| 目标 | 手段 |
|---|---|
| 强制 join 顺序 | `STRAIGHT_JOIN` / `JOIN_ORDER` / `JOIN_PREFIX` / `JOIN_SUFFIX` hint |
| 强制用/不用某索引 | `USE/FORCE/IGNORE INDEX` / `INDEX` / `NO_INDEX` hint |
| 禁用 semi-join | `optimizer_switch='semijoin=off'` / `NO_SEMIJOIN` hint |
| 只留某一种 semi-join 策略 | `firstmatch` / `loosescan` / `duplicateweedout` / `materialization` |
| 强制/禁用物化 | `materialization` / `subquery_materialization_cost_based` |
| 阻止 derived 合并 | `derived_merge=off` / `NO_MERGE` hint / 加 `LIMIT`（规则自然阻断） |
| 强制 HashJoin 或禁用 | `hash_join` / `BNL` / `NO_BNL` hint |
| 让 ORDER BY 用索引 | `prefer_ordering_index=on` |
| GROUP BY 不用临时表 | `SQL_BIG_RESULT` / `SQL_SMALL_RESULT`（见 15 篇第四章的反向逻辑警告） |
| 控制 range 优化内存 | `range_optimizer_max_mem_size`（变量，非 switch） |
| 改代价常量 | `mysql.server_cost` / `mysql.engine_cost` 表（见 [`01_cost_model.md`](01_cost_model.md)） |

---

## Misc

### 扩展点：新增一个决策要动哪些地方

| 步骤 | 要动的地方 |
|---|---|
| 1 决策逻辑 | 在 `prepare` 或 `optimize` 的合适阶段插入（注意依赖关系：能否在 join order 之前？） |
| 2 开关（可选） | `optimizer_switch_names[]` **和** `sql_const.h` 的 `#define`（顺序必须一致） |
| 3 hint（可选） | `sql/sql_hints.yy` + `opt_hints.h/cc` 的对象树（见 [`11_optimizer_hints.md`](11_optimizer_hints.md)） |
| 4 trace 埋点 | 否则无法诊断 |
| 5 EXPLAIN 表达 | `opt_explain*` 与 `explain_extra` |
| 6 MTR 回归 | 计划变更必须同步更新 `.result` |

### 已知缺陷与源码 TODO

- `test_skip_sort` 里那句 `TODO: Explain the allow_group_via_temp_table part of the test below` 仍未解决。
- `advance_sj_state` 的"先选后重算"是已知妥协（注释自陈），历史上因此出过计划不一致的 bug。
- 多个决策依赖**硬编码常量**（`COND_FILTER_EQUALITY = 0.1` 等），无统计时全靠拍。
- `optimizer_switch_names[]` 与 `sql_const.h` 的顺序耦合是长期隐患（源码注释自己加了 `BEWARE!`）。

### 社区边界澄清

以下内容**社区版 MySQL 没有**，不要与云厂商/分支特性混淆：

- **没有计划缓存 / plan cache**：每条语句重新做全部决策（见 [`../12_prepared_statement.md`](../12_prepared_statement.md)）
- **没有 SPM / 计划基线**：无法把"历史好计划"固定下来（Oracle 有，MySQL 只有 hint 这种手工手段）
- **没有多列统计 / 扩展统计**：决策依赖列独立假设（PolarDB 有 dependency 统计，社区没有）
- **没有 `inlist2join`**（Percona 有，社区没有）
- **没有 table elimination**（MariaDB 有，社区没有）
- **没有自适应优化 / 执行期计划修正**：决策全部在优化期一次性做定，执行期不反悔（HeatWave 的部分能力不在此列）
- **`hypergraph_optimizer` 是实验特性**：源码注释明说"故意不写进文档"，且默认 off

---

## 参考

**论文 / 理论**

- Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979) —— 代价决策的范式来源：选择性估算 + 左深树 DP
- Graefe《The Cascades Framework for Query Optimization》(1995) —— "规则生成候选 + 代价挑选"的标准分工；MySQL **没有**采用这个框架，这正是本篇"决策散落"的根源
- Moerkotte & Neumann《Dynamic Programming Strikes Back》(SIGMOD 2008) —— hypergraph 的枚举；semi-join 在其中是一等公民，与本篇"6 态策略模拟"形成对照
- **Simmen, Shekita, Malkemus《Fundamental Techniques for Order Optimization》(EDBT 1996)** —— 对应决策 26/27/28（DISTINCT/GROUP BY/ORDER BY 与排序选择）：sort 下推、用函数依赖推导顺序。MySQL 未实现
- **Neumann & Moerkotte《A Combined Framework for Grouping and Order Optimization》(VLDB 2004)** —— 对应决策 26/27/32：该论文主张分组与顺序必须联合推理；MySQL 把两者拆成两个互不通信的判定，这是多个反直觉现象的根源
- **Levy, Mumick, Sagiv《Query Optimization by Predicate Move-Around》(VLDB 1994)** —— 对应决策 29/30/31（删冗余谓词、ICP、引擎条件下推）：完整的谓词移动是先上提再下推，MySQL 只有下推

**WorkLog**

- WL#4389 "Transform EXISTS subqueries to semi-join" —— 8.0.16 起 EXISTS 也能转 semi-join，扩展了决策 1 的适用范围
- WL#1110 —— 子查询物化引擎的引入（决策 1、19 的物化分支）
- WL#2489 "better only_full_group_by" —— 决策 8 的标准特性 T301

**官方文档**

- MySQL 8.0 Reference Manual, "Optimizer Hints" / "Switchable Optimizations" / "Controlling the Query Optimizer"

**相关文档**

- [`00_overview.md`](00_overview.md) —— 四阶段框架与两条优化路径
- [`logical/02_subquery.md`](logical/02_subquery.md)、[`logical/03_semijoin.md`](logical/03_semijoin.md) —— 决策 1、19 的算法细节
- [`physical/06_join_order.md`](physical/06_join_order.md)、[`physical/07_access_method.md`](physical/07_access_method.md) —— 决策 16、17 的算法细节
- [`14_plan_stability.md`](14_plan_stability.md) —— 计划稳定性与能力边界
- [`11_optimizer_hints.md`](11_optimizer_hints.md) —— hint 的语法与对象树
- [`01_cost_model.md`](01_cost_model.md) —— 代价与统计（模式 1 类失败的根因）
