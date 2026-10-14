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
- [**★ 决策时序总纲：36 个决策点在代码里落在哪一环**](#决策时序总纲36-个决策点在代码里落在哪一环)
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
| 25 | **join 算法**（NLJ / BNL / BKA / HashJoin） | 规则 + 代价 | `setup_join_buffering` | `JOIN_CACHE` 类型 / HashJoin | `block_nested_loop` / `batched_key_access` / ~~`hash_join`~~（**已失效**，无任何消费点）/ `mrr` / `mrr_cost_based` / `BNL`/`BKA`/`NO_BNL` hint |
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

**★★ `SJ_OPT_*` 的真身**（⚠️ 是 **`#define` 宏，不是 enum**，`sql/sql_select.h:309`；且真名是 `DUPS_WEEDOUT` 不是 `DUPLICATE_WEEDOUT`）：

```cpp
#define SJ_OPT_NONE 0
#define SJ_OPT_DUPS_WEEDOUT 1
#define SJ_OPT_LOOSE_SCAN 2
#define SJ_OPT_FIRST_MATCH 3
#define SJ_OPT_MATERIALIZE_LOOKUP 4
#define SJ_OPT_MATERIALIZE_SCAN 5

/* ★ "是否物化"靠序值比较 —— 所以物化两策略必须排在最后 */
inline bool sj_is_materialize_strategy(uint strategy) {
  return strategy >= SJ_OPT_MATERIALIZE_LOOKUP;
}
```

**★★ 状态载体 `POSITION` 的四个字段**（`sql/sql_select.h:446`，★ 注释明说要"从右往左读"）：

```cpp
  /**
    Current optimization state: Semi-join strategy to be used for this
    and preceding join tables.

    Join optimizer sets this for the *last* join_tab in the
    duplicate-generating range. That is, in order to interpret this field,
    one needs to traverse join->[best_]positions array from right to left.
    When you see a join table with sj_strategy!= SJ_OPT_NONE, some other
    field (depending on the strategy) tells how many preceding positions
    this applies to. The values of covered_preceding_positions->sj_strategy
    must be ignored.
  */
  uint sj_strategy;
  uint n_sj_tables;              // 本策略覆盖几张表（fix_semijoin_strategies 后有效）

  /**
    Bitmap of semi-join inner tables that are in the join prefix and for
    which there's no provision yet for how to eliminate semi-join duplicates
    which they produce.
  */
  table_map dups_producing_tables;   /* ★★ "还没人负责去重"的表集合 —— 见下 */

  /* LooseScan strategy members */
  uint first_loosescan_table;
```

**★★★ 纠正一个普遍误解：五个策略不是"同台比价"**

`advance_sj_state()`（`sql/sql_planner.cc:4105`）里，**只有三处**出现真正的 `cost < best_cost` 比较。FirstMatch 与 LooseScan 是**无条件先占坑**：

```cpp
/* ---- FirstMatch（sql_planner.cc:4261）---- */
          (void)semijoin_firstmatch_loosescan_access_paths(
              pos->first_firstmatch_table, idx, remaining_tables, false,
              &rowcount, &cost);
          /*
            We don't yet know what are the other strategies, so pick FirstMatch.
            ... Instead, we will re-calculate the alternate POSITIONs after
            we've picked the best QEP.
          */
          sj_strategy = SJ_OPT_FIRST_MATCH;    /* ★★ 直接赋值，没有任何 cost 比较 */
          best_cost = cost;
          best_rowcount = rowcount;

/* ---- LooseScan（sql_planner.cc:4344）---- */
      if (semijoin_firstmatch_loosescan_access_paths(pos->first_loosescan_table,
                                                     idx, remaining_tables,
                                                     true, &rowcount, &cost)) {
        /*
          We don't yet have any other strategies that could handle this
          semi-join nest ... so unconditionally pick the LooseScan.
        */
        sj_strategy = SJ_OPT_LOOSE_SCAN;       /* ★★ 同样无条件；但前提是上面那个函数返回 true */
        best_cost = cost; best_rowcount = rowcount;
      }
```

而**后置的三个策略**才是真正的比价，且都带一个 `|| pos->dups_producing_tables` 的"强制接管"条件：

```cpp
/* ---- MaterializeLookup（:4461）---- */
    if (cost < best_cost || pos->dups_producing_tables) {
      sj_strategy = SJ_OPT_MATERIALIZE_LOOKUP;
      best_cost = cost; best_rowcount = rowcount;
      pos->dups_producing_tables &= ~emb_sj_nest->sj_inner_tables;
    }

/* ---- MaterializeScan 第二阶段（:4510）---- */
    if (cost < best_cost || pos->dups_producing_tables) {
      sj_strategy = SJ_OPT_MATERIALIZE_SCAN;
      ...
    }

/* ---- Duplicate Weedout（:4564）★ 唯一还要额外检查 optimizer_switch 的策略 ---- */
      if ((cost < best_cost &&
           join->positions[pos->first_dupsweedout_table]
                   .table->emb_sj_nest->nested_join->sj_enabled_strategies &
               OPTIMIZER_SWITCH_DUPSWEEDOUT) ||
          pos->dups_producing_tables) {
        sj_strategy = SJ_OPT_DUPS_WEEDOUT;
        ...
      }
```

> ★★★ **"`|| pos->dups_producing_tables`" 是整个决策的兜底语义**：Weedout 是最后一个被考虑的策略（它需要最多的表在前缀里），所以**不能留下"没人负责去重"的表**——一旦还有 `dups_producing_tables`，无论代价多高都必须接管。源码注释原文：*"we can't leave duplicate-producing tables not handled by any strategy"*。
> ★ **Weedout 是唯一在比价之外还要查 `OPTIMIZER_SWITCH_DUPSWEEDOUT` 的**——其余四个的开关检查都在更早的位置（`sj_enabled_strategies` 位图，见决策 15）。

**★★ MaterializeScan 是两阶段的**（★ 这是"为什么 trace 里先看到 `choice: deferred`"的原因）：

```cpp
  const int sjm_strategy = semijoin_order_allows_materialization(
      join, remaining_tables, new_join_tab, idx);
  if (sjm_strategy == SJ_OPT_MATERIALIZE_SCAN) {
    /*
      We cannot evaluate this option now. This is because we cannot
      account for fanout of sj-inner tables yet:
        ntX  SJM-SCAN(it1 ... itN) | ot1 ... otN  |
                                   ^(1)           ^(2)
      we're now at position (1). SJM temptable in general has multiple
      records, so at point (1) we'll get the fanout from sj-inner tables.
      The final join result will not contain any semi-join produced fanout...
      The simple way to model this is to remove SJM-SCAN(...) fanout once
      we reach the point #2.
    */
    ...
    pos->sjm_scan_need_tables = emb_sj_nest->sj_inner_tables |
                                emb_sj_nest->nested_join->sj_depends_on;
    pos->sjm_scan_last_inner = idx;
    Opt_trace_object(trace)
        .add_alnum("strategy", "MaterializeScan")
        .add_alnum("choice", "deferred");       /* ★★ 第一阶段只记账，不算代价 */
  }
```

**★★ `dups_producing_tables` 的记账规则**（★ 只有 FM/LS 能"消账"）：

```cpp
  if (emb_sj_nest) pos->dups_producing_tables |= emb_sj_nest->sj_inner_tables;
  pos->dups_producing_tables &= ~handled_by_fm_or_ls;   /* ★ FM 或 LS 选中后消掉 */
```

**★★ LooseScan 的 8 条准入**（`sql_planner.cc:4355`，★ 这是"为什么 LooseScan 经常不生效"的完整答案）：

```cpp
    /*
      We may consider the LooseScan strategy if
      1a. The next table is an SJ-inner table, and
      1b. LooseScan is enabled for this SJ nest, and
      2. We have no more than 64 IN expressions (must fit in bitmap), and
      3. It is the first table from that semijoin, and
      4. We're not within a semi-join range, except
      new_join_tab->emb_sj_nest (which we've just entered, see #3), and
      5. All non-IN-equality correlation references from this sj-nest are
      bound, and
      6. But some of the IN-equalities aren't (so this can't be handled by
      FirstMatch strategy), and
      7. There are equalities (including maybe semi-join ones) which can be
      handled with an index of this table, and
      8. Not a derived table/view. (a temporary restriction)
    */
    if (emb_sj_nest &&                                            // (1a)
        emb_sj_nest->nested_join->sj_enabled_strategies &
            OPTIMIZER_SWITCH_LOOSE_SCAN &&                        // (1b)
        emb_sj_nest->nested_join->sj_inner_exprs.size() <= 64 &&  // (2)
        ((remaining_tables_incl & emb_sj_nest->sj_inner_tables) ==
         emb_sj_nest->sj_inner_tables) &&                         // (3)
        pos->dups_producing_tables == 0 &&                        // (4)
        !(remaining_tables_incl &
          emb_sj_nest->nested_join->sj_corr_tables) &&            // (5)
        (remaining_tables_incl &
         emb_sj_nest->nested_join->sj_depends_on) &&              // (6)
        new_join_tab->keyuse() != nullptr &&                      // (7)
        !new_join_tab->table_ref->uses_materialization())         // (8)
```

> ★★ **第 6 条最反直觉**：LooseScan 要求 **IN 等值的外层依赖表还没进前缀**（`sj_depends_on` 与 `remaining_tables_incl` 有交集）。这与 FirstMatch 的第 3 条**恰好相反**（FM 要求所有依赖表**已**在前缀）。所以**同一个 join 前缀不可能同时满足 FM 与 LS**——这解释了二者的天然分工。
> ★ 第 7 条 `new_join_tab->keyuse() != nullptr`：LS 的驱动表必须有可用索引，否则直接出局。

**★★ "先选后重算"的后半段：`fix_semijoin_strategies()`**（`sql_planner.cc:3360`，★ 从右往左扫，把"倒序记录"翻转成"正序可读"）：

```cpp
/**
  @details
    Fix semi-join strategies for the picked join order. ... What we do
    here is switch join's semi-join strategy description from backward-based
    to forwards based.

    Suppose the join optimization is underway:
    1) ot1  it1  it2
                 sjX  -- looking at (ot1, it1, it2) join prefix, we decide
                         to use semi-join strategy sjX.
    2) ot1  it1  it2  ot2
                 sjX  sjY -- Having added table ot2, we now may consider
                             another semi-join strategy and decide to use a
                             different strategy sjY. Note that the record
                             of sjX has remained under it2. That is
                             necessary because we need to be able to get
                             back to (ot1, it1, it2) join prefix.
      what makes things even worse is that there are cases where the choice
      of sjY changes the way we should access it2.
    3) [ot1  it1  it2  ot2  ot3]
                  sjX  sjY  -- This means that after join optimization is
                               finished, semi-join info should be read
                               right-to-left (while nearly all plan refinement
                               functions, EXPLAIN, etc proceed from left to
                               right)
    This function does the needed reversal...
*/
bool Optimize_table_order::fix_semijoin_strategies() {
  for (uint tableno = join->tables - 1;
       tableno != join->const_tables - 1; tableno--) {     /* ★★ 从右往左 */
    POSITION *const pos = join->best_positions + tableno;
    if ((handled_tables & pos->table->table_ref->map()) ||
        pos->sj_strategy == SJ_OPT_NONE) { ... continue; }

    uint first = 0;
    if (pos->sj_strategy == SJ_OPT_MATERIALIZE_LOOKUP) {
      Table_ref *const sjm_nest = pos->table->emb_sj_nest;
      const uint table_count = my_count_bits(sjm_nest->sj_inner_tables);
      /*
        This memcpy() copies a partial QEP produced by
        optimize_semijoin_nests_for_materialization() (source) into the final
        top-level QEP (target), in order to re-use the source plan for
        to-be-materialized inner tables.
        It is however possible that the source QEP had picked some semijoin
        strategy (noted SJY), different from materialization. ... Which is why
        sj_strategy of each table of the duplicate-generating range then
        becomes temporarily unreliable.
      */
      memcpy(pos - table_count + 1, sjm_nest->nested_join->sjm.positions,
             sizeof(POSITION) * table_count);    /* ★★ 把物化子计划整段拷回主计划 */
      first = tableno - table_count + 1;
      join->best_positions[first].n_sj_tables = table_count;
      join->best_positions[first].sj_strategy = SJ_OPT_MATERIALIZE_LOOKUP;
      ...
    } else if (pos->sj_strategy == SJ_OPT_FIRST_MATCH) {
      first = pos->first_firstmatch_table;
      // Recalculate final access paths for this semi-join strategy
      double rowcount, cost;
      (void)semijoin_firstmatch_loosescan_access_paths(
          first, tableno, remaining_tables, false, &rowcount, &cost);  /* ★★ 重算 */
      if (pos->table->emb_sj_nest->is_aj_nest()) {
        /*
          Antijoin doesn't use the execution logic of FirstMatch. So we
          won't set it up ...
        */
        pos->sj_strategy = SJ_OPT_NONE;
      } else {
        join->best_positions[first].sj_strategy = SJ_OPT_FIRST_MATCH;
        join->best_positions[first].n_sj_tables = tableno - first + 1;
      }
    } else if (pos->sj_strategy == SJ_OPT_LOOSE_SCAN) { ... }
      else if (pos->sj_strategy == SJ_OPT_DUPS_WEEDOUT) { ... }

    for (uint i = first; i <= tableno; i++) {
      /* Eliminate stale strategies. */
      if (i != first) join->best_positions[i].sj_strategy = SJ_OPT_NONE;  /* ★ 区间内其余清 0 */
      handled_tables |= join->best_positions[i].table->table_ref->map();
    }
    remaining_tables |= pos->table->table_ref->map();
  }
}
```

> ★★ **`semijoin_firstmatch_loosescan_access_paths()` 一个对象服务两次调用**——靠成员 `got_final_plan` 区分写哪里：搜索期写局部 `join->positions`（且只写 rowcount/cost，access path **丢弃**），定案后写 `join->best_positions` 并真正生成 access path。这就是"重算"的物理含义。

**★ 物化代价的三个判据**（`calculate_materialization_costs()`，`sql_optimizer.cc:10954`）：

```cpp
    distinct_rowcount = min(mat_rowcount, rows);   /* ★ 去重后行数 = min(物化前行数, 各相关表 rows 之积) */
  ...
  const uint rowlen = get_tmp_table_rec_length(*inner_expr_list);
  /* ★★ 内存 / 磁盘临时表的判定阈值 */
  if (rowlen * distinct_rowcount < join->thd->variables.max_heap_table_size)
    tmp_table_type = Cost_model_server::MEMORY_TMPTABLE;
  else
    tmp_table_type = Cost_model_server::DISK_TMPTABLE;

  mat_cost += cost_model->tmptable_create_cost(tmp_table_type);
  mat_cost += cost_model->tmptable_readwrite_cost(tmp_table_type, mat_rowcount, 0.0);
```

**★ 哪些 nest 值得物化**（`optimize_semijoin_nests_for_materialization()`，`sql_optimizer.cc:6600`）：

```cpp
    if (sj_nest->nested_join->sj_enabled_strategies &
        OPTIMIZER_SWITCH_MATERIALIZATION) {           /* ① optimizer_switch materialization=on */
      if (sj_nest->nested_join->sj_corr_tables) continue;   /* ② 非平凡相关 → 放弃物化 */
      semijoin_types_allow_materialization(sj_nest);        /* ③ 类型检查 → scan_allowed / lookup_allowed */
      if (!sj_nest->nested_join->sjm.scan_allowed &&
          !sj_nest->nested_join->sjm.lookup_allowed)
        continue;                                           /* ④ 两者都不可用 → 放弃 */
      if (Optimize_table_order(join->thd, join, sj_nest).choose_table_order())
        return true;    /* ★★ 递归：为这个 nest 单独跑一次 join order 优化 */
      ...
      memcpy(sj_nest->nested_join->sjm.positions,
             join->best_positions + join->const_tables,
             sizeof(POSITION) * n_tables);   /* ★ 保存子计划，供 fix_semijoin_strategies 回拷 */
    }
```

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

**★★ 输入侧：哪些条件能生成 Key_use**（★ 决策 17 的候选从哪来）。真正的闸门是 `add_key_field()`（`sql/sql_optimizer.cc:7090`），它有三个硬准入：

```cpp
    table_map used_tables = 0;
    bool optimizable = false;
    for (uint i = 0; i < num_values; i++) {
      used_tables |= (value[i])->used_tables();
      if (!((value[i])->used_tables() & (tl->map() | RAND_TABLE_BIT)))
        optimizable = true;     /* ★ 准入 1：比较值不能引用本表、也不能是 RAND 表 */
    }
    if (!optimizable) return false;
    ...
      Key_map possible_keys = field->key_start;
      possible_keys.intersect(tl->table->keys_in_use_for_query);  /* ★ 准入 2：受 keys_in_use_for_query 约束 */
      stat[0].keys().merge(possible_keys);
    ...
      /*
        We can't always use indexes when comparing a string index to a number.
        ... eq_func is NEVER true when num_values > 1
      */
      if (!eq_func) return false;   /* ★★ 准入 3：只有等值类（=, <=>, IS NULL）才生成 Key_use */

      /* Check if the field and value are comparable in the index. */
      if (!comparable_in_index(cond, field, Field::itRAW, cond->functype(),
                               *value) ||
          (field->cmp_type() == STRING_RESULT &&
           field->match_collation_to_optimize_range() &&
           field->charset() != cond->compare_collation())) {
        warn_index_not_applicable(stat->join()->thd, field, possible_keys);
        return false;
      }
```

> ★★★ **第 3 条是最容易被忽略的**：**范围谓词（`>`、`<`、`BETWEEN`、`LIKE`）根本不生成 Key_use**，它们只进 `sargables` 数组交给 range 优化器。所以 `ref` 候选**只来自等值条件**——这是"为什么有时候明明有索引却说 no key"的一半答案（另一半是下面的 collation 检查）。
> ★ **`BETWEEN low AND high` 在 `low == high` 时退化成等值**（`add_key_fields()` 里 `equal_func = true; num_values = 1;`），因而能进 Key_use。
> ★ `OR` 分支的 Key_use 必须 `merge_key_fields()`（交集存活），所以 `a=1 OR b=2` 通常产生不了可用 ref。

**★ 三个开关的作用点**（`use_invisible_indexes` / `use_index_extensions` 的真实落点）：

```cpp
/* sql/table.cc:497 —— use_invisible_indexes */
Key_map TABLE_SHARE::usable_indexes(const THD *thd) const {
  Key_map usable_indexes(keys_in_use);
  if (!thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES))
    usable_indexes.intersect(visible_indexes);
  return usable_indexes;
}

/* sql/field.cc:9756 —— use_index_extensions：决定"主键扩展列"算不算索引一部分 */
bool Field::is_part_of_actual_key(THD *thd, uint cur_index,
                                  KEY *cur_index_info) const {
  return thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS) &&
                 !(cur_index_info->flags & HA_NOSAME)
             ? part_of_key.is_set(cur_index)
             : part_of_key_not_extended.is_set(cur_index);
}

/* sql/sql_select.cc:5416 —— use_index_extensions：影响"实际 keypart 数" */
uint actual_key_parts(const KEY *key_info) {
  return current_thd->optimizer_switch_flag(
             OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS)
             ? key_info->actual_key_parts      /* ★ 含 InnoDB 主键扩展部分 */
             : key_info->user_defined_key_parts;
}
```

**★★ 二次 range 重估：`can_switch_from_ref_to_range()` 的 6 条准入**（`sql/sql_optimizer.cc:2883`，★ 文档里常说"range 与 ref 比价"，实际是一条**启发式**，不是比价）：

```cpp
/**
  A helper function to check whether it's better to use range than ref.

  @details
  Heuristic: Switch from 'ref' to 'range' access if 'range'
  access can utilize more keyparts than 'ref' access. Conditions
  for doing switching:
  1) Range access is possible
  2) 'ref' access and 'range' access uses the same index
  3) Used parts of key shouldn't have nullable parts & ref_or_null isn't used.
  4) 'ref' access depends on a constant, not a value read from a table earlier
     in the join sequence.
     Rationale: ... if 'ref' depends on a value from another table, the join
     condition is not used to limit the rows read by 'range' access ...
     We therefore trust the cost based choice made by best_access_path()
     instead of forcing a heuristic choice here.
  5) 'range' access uses more keyparts than 'ref' access
  6) ORDER BY might make range better than table scan:
     Check possibility of range scan even if it was previously deemed unviable
  ...
  @todo: This decision should rather be made in best_access_path()
*/
static bool can_switch_from_ref_to_range(THD *thd, JOIN_TAB *tab,
                                         enum_order ordering,
                                         bool recheck_range) {
  if ((tab->range_scan() &&                                            // 1)
       tab->position()->key->key == used_index(tab->range_scan())) ||  // 2)
      recheck_range) {
    ...
    if (!maybe_null &&     // 3)
        !dep_map)          // 4)
    {
      if (recheck_range)   // 6)
      {
        ... test_quick_select(...) ...
        if (length < get_max_used_key_length(range_scan)) {
          destroy(tab->range_scan());
          tab->set_range_scan(range_scan);
          return true;
        }
      } else
        return length < get_max_used_key_length(tab->range_scan());   // 5)
    }
  }
  return false;
}
```

> ★★★ **判据是"range 用到的 key 长度 > ref 用到的长度"，不是代价**。第 4 条的注释尤其重要：若 ref 依赖前序表的值（而不是常量），**放弃启发式、信任 `best_access_path()` 的代价选择**——因为此时 range 与 ref 用的是条件的不同部分，可比性不成立。

---

### 六、深挖 C：join 算法的选择

> 由 `setup_join_buffering` 完成（`sql/sql_optimizer.cc`），发生在 `optimize_distinct_group_order` 之后。

MySQL 8.0 的 join 算法只有四个（**没有 merge join**，见 [`../09_executor_iterator.md`](../09_executor_iterator.md)）：

| 算法 | 触发条件 | 开关 |
|---|---|---|
| **NLJ**（嵌套循环，无缓冲） | 默认 | — |
| **BNL**（块嵌套循环，join buffer） | 内层无索引可用且 `block_nested_loop=on` | `block_nested_loop`、`join_buffer_size` |
| **BKA**（批量键访问，MRR 优化） | 内层有索引 + `batched_key_access=on` + MRR 可用 | `batched_key_access`、`mrr`、`mrr_cost_based`、`read_rnd_buffer_size` |
| **HashJoin** | 等值连接条件、无索引可用（8.0.18+） | ⚠️ `hash_join` 开关**已失效**（见下） |

决策要点：

1. **HashJoin 取代了 BNL 的默认地位**：理论上 hash join 在各维度不劣于 BNL（见 [`../runtime/05_join_buffer.md`](../runtime/05_join_buffer.md)），所以 8.0 里**执行器把 `OT_BNL` 一律改写成 hash join**——判定就一行，只读结构体成员：

   ```cpp
   static bool UseHashJoin(QEP_TAB *qep_tab) {
     return qep_tab->op_type == QEP_TAB::OT_BNL;
   }
   ```

   > ⚠️ **8.0.39 已无法再退回 BNL**。`OPTIMIZER_SWITCH_HASH_JOIN` 在整个 `sql/` 下**只有两处出现**——`sql_const.h` 的常量定义（`1ULL << 21`）与 `sys_vars.cc` 里 `optimizer_switch` 的默认位掩码——**没有任何消费点**。经典优化器侧走上面的 `UseHashJoin()`，hypergraph 侧由 `ProposeHashJoin` 直接提出候选，**两条路径都不读这个开关**。旧资料与本文早前版本写的"hash_join=off 退回 BNL"**已不成立**。同理 `HASH_JOIN` / `NO_HASH_JOIN` hint 也无代码消费（见 [`11_optimizer_hints.md`](11_optimizer_hints.md)）。
2. **BKA 与 HashJoin 不冲突**：BKA 只适用于**内层有索引**的场景（把随机索引查找变成按 rowid 排序的顺序查找，即 DS-MRR）；HashJoin 用于**内层无索引**。二者适用面互补。
3. **`mrr_cost_based`** 决定 MRR 是"代价驱动"还是"总是用"。
4. `no_jbuf_after` 参数保证"某些策略之后不能再加 join buffer"（例如 semi-join 的 FirstMatch 之后），这是跨决策的约束。

**★★ `setup_join_buffering()` 全文**（`sql/sql_optimizer.cc:3451`，★ 上面那张表的真身）。

先看**算法枚举**（⚠️ 真名在 `sql/sql_join_buffer.h`，不是 `sql_join_cache.h`；**只有三个值**）：

```cpp
class JOIN_CACHE {
 public:
  /** Bits describing cache's type @sa setup_join_buffering() */
  enum enum_join_cache_type { ALG_NONE = 0, ALG_BNL = 1, ALG_BKA = 2 };
};
```

**两段开关读取 + 初值**（★ hint 与 optimizer_switch 在这里汇合）：

```cpp
static bool setup_join_buffering(JOIN_TAB *tab, JOIN *join, uint no_jbuf_after) {
  const bool bnl_on = hint_table_state(join->thd, tab->table_ref, BNL_HINT_ENUM,
                                       OPTIMIZER_SWITCH_BNL);   /* ★ BNL hint / block_nested_loop */
  const bool bka_on = hint_table_state(join->thd, tab->table_ref, BKA_HINT_ENUM,
                                       OPTIMIZER_SWITCH_BKA);   /* ★ BKA hint / batched_key_access */
  ...
  // Set preliminary join cache setting based on decision from greedy search
  if (!join->select_count)
    tab->set_use_join_cache(tab->position()->use_join_buffer
                                ? JOIN_CACHE::ALG_BNL
                                : JOIN_CACHE::ALG_NONE);   /* ★ 初值来自贪心搜索期已做过的代价比较 */
  if (tableno == join->const_tables) { ... return false; }  /* 第一个非 const 表永不缓冲 */
  if (!(bnl_on || bka_on)) goto no_join_cache;              /* ★ 两个开关都关 → 放弃 */
```

**八条失败分支**（★ 全部 `goto no_join_cache`，这是"为什么加了 hint 却没用上 join buffer"的排查清单）：

```cpp
  if (tab->use_quick == QS_DYNAMIC_RANGE) goto no_join_cache;      // ① dynamic range
  if (tableno > no_jbuf_after) goto no_join_cache;                 // ② 硬边界（semi-join/force index 传进来）
  /* ③ outer join nest 首个 inner 表没缓冲 → 后面的 inner 表也不许 */
  if (tab->first_inner() != NO_PLAN_IDX && tab->first_inner() != tab->idx() &&
      !join->best_ref[tab->first_inner()]->use_join_cache())
    goto no_join_cache;
  /* ④ 上层 nest 没缓冲 */
  if (tab->first_upper() != NO_PLAN_IDX &&
      !join->best_ref[tab->first_upper()]->use_join_cache())
    goto no_join_cache;
  if (tab->table()->pos_in_table_list->is_table_function() && tab->dependent)
    goto no_join_cache;                                            // ⑤ 表函数 + dependent

  switch (tab_sj_strategy) {                                       // ⑥ 按 semi-join 策略分派
    case SJ_OPT_FIRST_MATCH:
      /* Use join cache with FirstMatch only when semi-join contains only one
         table. (lifted for secondary engine) */
      if (!(... using_secondary_storage_engine())) {
        if (!tab->is_single_inner_of_semi_join()) goto no_join_cache;
      }
      break;
    case SJ_OPT_LOOSE_SCAN:
      /* No join buffering if this semijoin nest is handled by loosescan */
      goto no_join_cache;
    case SJ_OPT_MATERIALIZE_LOOKUP:
    case SJ_OPT_MATERIALIZE_SCAN:
      /* 物化 nest 的首张表不能缓冲，其余可以 */
      if (tab->first_sj_inner() == tab->idx()) goto no_join_cache;
      break;
    case SJ_OPT_DUPS_WEEDOUT:
    case SJ_OPT_NONE:
      break;    /* ★ Weedout 允许与常规 join 相同的缓冲 */
  }

  /* ⑦ FirstMatch + outer join 争用同一个 match flag */
  if (tab_sj_strategy == SJ_OPT_FIRST_MATCH &&
      tab->is_inner_table_of_outer_join())
    goto no_join_cache;
  /* ⑧ 表落在 LATERAL 派生表的依赖区间内 */
  if (join->deps_of_remaining_lateral_derived_tables &
      (tab->prefix_tables() & ~tab->added_tables()))
    goto no_join_cache;
```

> ★★ **第 ⑦ 条的注释是全篇最精辟的一段**（解释了"一个缓存行只有一个 match flag"）：outer join 需要一个 flag 标记"是否要 NULL 补齐"，FirstMatch 也需要一个 flag 标记"是否已找到第一个匹配"——**若"sj inner table"正好等于"first inner"，两者抢同一个 flag，只能放弃**。

**★ 真正的 BNL / BKA 分派**（★ 按 `tab->type()` 一刀切）：

```cpp
  switch (tab->type()) {
    case JT_ALL:
    case JT_INDEX_SCAN:
    case JT_RANGE:
    case JT_INDEX_MERGE:
      if (!bnl_on) goto no_join_cache;         /* ★ 全表/索引扫/range/index_merge → 只能 BNL */
      if (!join->select_count) tab->set_use_join_cache(JOIN_CACHE::ALG_BNL);
      return false;                            /* → 出口：BNL（后续必被改写成 hash join） */

    case JT_SYSTEM:
    case JT_CONST:
    case JT_REF:
    case JT_EQ_REF:
      if (!bka_on) goto no_join_cache;         /* ★ ref/eq_ref/const/system → 只能 BKA */
      if (tab->table_ref->uses_materialization()) goto no_join_cache;  // a: 物化派生表未实例化
      if (tab->has_guarded_conds()) goto no_join_cache;                // b: guarded conds
      if (tab->table()->covering_keys.is_set(tab->ref().key))
        join_cache_flags |= HA_MRR_INDEX_ONLY;
      rows = tab->table()->file->multi_range_read_info(   /* ★ 问引擎：能不能做 MRR */
          tab->ref().key, 10, 20, &bufsz, &join_cache_flags, &cost);
      /*
        Cannot use BKA if
        1. MRR scan cannot be performed, or
        2. MRR default implementation is used, or
        3. HA_MRR_NO_ASSOCIATION flag is set
      */
      if ((rows == HA_POS_ERROR) ||                        // 1
          (join_cache_flags & HA_MRR_USE_DEFAULT_IMPL) ||  // 2
          (join_cache_flags & HA_MRR_NO_ASSOCIATION))      // 3
        goto no_join_cache;
      tab->set_use_join_cache(JOIN_CACHE::ALG_BKA);        /* → 出口：BKA */
      tab->join_cache_flags = join_cache_flags;
      return false;
    default:;
  }
no_join_cache:
  revise_cache_usage(tab);
  tab->set_use_join_cache(JOIN_CACHE::ALG_NONE);           /* → 出口：无缓冲 */
```

> ★★★ **BNL 与 BKA 的分界就是 `tab->type()`**：`ALL/INDEX_SCAN/RANGE/INDEX_MERGE` 归 BNL，`SYSTEM/CONST/REF/EQ_REF` 归 BKA。**这两个集合不相交**，所以同一张表不可能既 BNL 又 BKA。
> ★ BKA 的失败分支 c 要**问引擎**（`multi_range_read_info`）——引擎说"只能用默认 MRR 实现"就没收益，直接放弃。

**★★ `UseHashJoin()` 的完整上下文**（⚠️ 真身在 **`sql/sql_executor.cc:2188`**，不是 `sql_select.cc`）：

```cpp
static bool UseHashJoin(QEP_TAB *qep_tab) {
  return qep_tab->op_type == QEP_TAB::OT_BNL;   /* ★ 就一行；所有前置条件都在调用点外面判 */
}

/* sql/sql_executor.h:409 —— OT_BNL 的注释本身就是证据 */
    // Block-nested loop (rewritten to hash join).
    OT_BNL,
```

```cpp
/* op_type 的来源是 setup_join_buffering 的结果（sql_select.cc:3249） */
void QEP_TAB::init_join_cache(JOIN_TAB *join_tab) {
  switch (join_tab->use_join_cache()) {
    case JOIN_CACHE::ALG_BNL: op_type = QEP_TAB::OT_BNL; break;
    case JOIN_CACHE::ALG_BKA: op_type = QEP_TAB::OT_BKA; break;
    default: assert(0);
  }
}
```

```cpp
/* 改写成 hash join 迭代器（sql_executor.cc:2726） */
    const bool replace_with_hash_join =
        UseHashJoin(qep_tab) && !QueryMixesOuterBKAAndBNL(qep_tab->join());
```

**★ 唯一的否决条件 `QueryMixesOuterBKAAndBNL()`**（★ 这是"为什么 BKA 会连带关掉 hash join"的答案）：

```cpp
// Having a non-BKA join on the right side of an outer BKA join causes problems
// for the matched-row signaling from MultiRangeRowIterator to BKAIterator;
// rows could be found just fine, but not go through the join filter (and thus
// not be marked as matched in BKAIterator), creating extra NULLs.
//
// The only way this can happen is when we get a hash join on the inside of an
// outer BKA join ... If this happens, we simply turn off both BKA and hash
// join handling for the query; it is a very rare situation, and the slowdown
// should be acceptable. (Only turning off BKA helps somewhat, but
// MultiRangeRowIterator also cannot be on the inside of a hash join, so we
// need to turn off BNL as well.)
static bool QueryMixesOuterBKAAndBNL(JOIN *join) {
  bool has_outer_bka = false, has_bnl = false;
  for (uint i = join->const_tables; i < join->primary_tables; ++i) {
    QEP_TAB *qep_tab = &join->qep_tab[i];
    if (UseHashJoin(qep_tab)) has_bnl = true;                  /* ★ 复用 UseHashJoin */
    else if (qep_tab->op_type == QEP_TAB::OT_BKA &&
             qep_tab->last_inner() != NO_PLAN_IDX) has_outer_bka = true;
  }
  return has_bnl && has_outer_bka;                             /* ★★ 同时出现 → 双双禁用 */
}
```

> ★★ 所以：**一个 query 里不能同时有"外连接 BKA"和"hash join"**——触发时两者**一起关掉**，退回纯 NLJ。这是一个非常隐晦的行为突变源。

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
| 全局变慢 | 有人关了 `block_nested_loop`（~~`hash_join` 已失效，关了也没用~~）或 `condition_fanout_filter` |
| 某些查询突然用不了索引 | `range_optimizer_max_mem_size` 太小导致长 IN list 降级 |
| 索引不可见却没生效 | `use_invisible_indexes` |
| 排序变多 | `prefer_ordering_index=off` |

---

## ★★ 决策时序总纲：36 个决策点在代码里落在哪一环

前面三张表按"阶段"分组，这里给出**可对照源码行号的完整调用序列**——排查"为什么这个决策没生效"时，先定位它在这条链的哪一环。

### `JOIN::optimize()` 骨架（`sql/sql_optimizer.cc:338`）

```
optimize()
 ├─ 370  Window::setup_windows2()                      ← 决策 9
 ├─ 404  tl->optimize_derived()（递归各 derived/view）
 ├─ 422  【分岔】using_hypergraph_optimizer() → 先递归优化所有子查询
 ├─ 474  optimize_cond(WHERE)      ★ 等值/常量传播（决策 13、6 的 optimize 版）
 │        └─ cond_value == COND_FALSE → goto "Impossible WHERE"       ← 出口②
 ├─ 488  optimize_cond(HAVING)     → having_value == FALSE           ← 出口③
 ├─ 502  prune_table_partitions()  ★ 决策 10（optimize 期版，可裁非常量条件）
 ├─ 517  optimize_aggregated_query()  ★ 决策 12
 │        ├─ AGGR_COMPLETE → goto "Select tables optimized away"     ← 出口④
 │        └─ AGGR_EMPTY    → goto "No matching min/max row"          ← 出口⑤
 ├─ 600  substitute_gc()           ★ 决策 11
 ├─ 610  ★★★ 分岔：hypergraph
 │        ├─ 622 FindBestQueryPlan()   ← join order + 访问路径一次搞定
 │        ├─ 624 FinalizePlanForQueryBlock()
 │        └─ 678 return false          ★★ 直接返回，根本不进 make_join_plan()
 ├─ 696  make_join_plan()           ★★ 决策 14/15/16/19 的宿主（见下）
 ├─ 737  substitute_for_best_equal_field(WHERE)   ★ 决策 23
 ├─ 756  substitute_for_best_equal_field(join cond)
 ├─ 770  init_ref_access()
 ├─ 781  make_join_query_block()    → "Impossible WHERE noticed after reading const tables" ← 出口⑧
 ├─ 796  optimize_distinct_group_order()   ★ 决策 26
 ├─ 861  replace_index_subquery()   ← 唯一键子查询短路                 ← 出口⑩
 ├─ 897  setup_join_buffering()     ★★ 决策 25/33（逐表）
 ├─ 975  need_tmp_before_win 的 (1)..(8) 判定   ★ 决策 32
 ├─ 1015 test_skip_sort()           ★★ 决策 27/28
 ├─ 1017 finalize_table_conditions() ★ 决策 29
 ├─ 1020 make_join_readinfo()       → 内部调 push_index_cond()  ★ 决策 30
 ├─ 1023 make_tmp_tables_info()     ★★ 决策 32
 ├─ 1037 create_access_paths()      ★ 决策 35
 ├─ 1064 push_to_engines()          ★ 决策 31
 └─ 1067 set_plan_state(PLAN_READY)
```

> ★★ **8 个 `goto setup_subq_exit` 提前出口**（470/484/498/562/569/707/786/838）+ hypergraph 在 678 提前 return。**任何一个出口都意味着后面所有决策不再发生**——这是"看起来优化器什么都没做"的第一排查点。

### `JOIN::make_join_plan()` 骨架（`sql/sql_optimizer.cc:5307`）

```
make_join_plan()
 ├─ 5314 init_planner_arrays()        建 JOIN_TAB[]/best_ref[]/positions[]，并 fetch_number_of_rows()
 ├─ 5319 propagate_dependencies()     外连接依赖传播（递归 CTE 的非法顺序在此报 ER_CTE_RECURSIVE_*）
 ├─ 5343 update_ref_and_keys()        ★★ 决策 15：生成 keyuse_array（ref 候选）
 ├─ 5354 pull_out_semijoin_tables()   ★ 决策 16（表上提）
 ├─ 5362 extract_const_tables()       ★★ 决策 14：0/1 行 → const
 ├─ 5365 extract_func_dependent_tables()  ★ 决策 14 第二轮（函数依赖）
 ├─ 5368 update_sargable_from_const()
 ├─ 5371 estimate_rowcount()          ★ fanout / 选择性估算
 ├─ 5379 apply_join_order_hints()     ★★ 决策 14 可覆盖：JOIN_ORDER/PREFIX/SUFFIX hint
 │        └─ 前置：!(active_options() & SELECT_STRAIGHT_JOIN)  ★ STRAIGHT_JOIN 下静默跳过
 ├─ 5383 update_semijoin_strategies() ★★ 决策 19 可覆盖：写 sj_enabled_strategies 位图
 ├─ 5386 optimize_keyuse()            keyuse 去重/淘汰
 ├─ 5390 optimize_semijoin_nests_for_materialization()  ★ 物化代价预估
 ├─ 5394 choose_table_order()         ★★★ 决策 16/17/19 的真正搜索
 │        └─ best_extension_by_limited_search() → advance_sj_state()（每加一张表调一次）
 ├─ 5401 decide_subquery_strategy()   ★★ 决策 1 的 optimize 期落点
 ├─ 5403 refine_best_rowcount()
 ├─ 5408 get_best_combination()       由 best_positions 生成 QEP_TAB
 └─ 5420 const_table_map != found_const_table_map → zero_result_cause（出口⑦）
```

> ★★ **决策 15（Key_use 生成）在决策 14（const 表）之后但在决策 16（join order）之前**——这个顺序是硬性的：`update_ref_and_keys()` 需要 read_partitions 已定，而 `choose_table_order()` 需要 keyuse_array。

### ★ 决策时序的三条铁律

1. **逻辑改写必须最早**（prepare 期）：`simplify_joins()`（决策 3/4）在 `sql_resolver.cc:766`，而依赖它的 `check_only_full_group_by()`（决策 8）在 `:773`——源码注释原文解释：*"check_only_full_group_by() is dependent on processing done by simplify_joins() (for example it uses the value of Query_block::outer_join)"*。
2. **条件下推到 derived（决策 5）排在所有 local transforms 之后**，且**只由最外层 block 触发**（`outer_query_block() == nullptr`）。注释原文点出代价：*"a pushed-down condition cannot help to convert LEFT JOIN to inner join inside a derived table's definition"*。
3. **`substitute_for_best_equal_field()`（决策 23）在 `make_join_plan()` 之后**——因为"等价类用哪个字段作代表"依赖 join 顺序。这是唯一一个"逻辑改写却发生在 optimize 中段"的例外。

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
| `hash_join` | Hash Join ⚠️ **已失效**：`sql/` 下仅常量定义与默认值，无任何消费点 |
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
| 强制 HashJoin 或禁用 | ⚠️ **用 `block_nested_loop` 开关 / `BNL` / `NO_BNL` hint**，**不是** `hash_join`——后者无消费点。`BNL` hint 有真实消费点（`hint_table_state(..., BNL_HINT_ENUM, OPTIMIZER_SWITCH_BNL)`），它决定是否走 `OT_BNL`，进而决定是否 hash join |
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

**★ 硬编码常量的真身**（⚠️ 在 **`sql/item.h:98-131`**，不在 `sql_optimizer.h` / `sql_planner.h`；且**只有 3 个默认选择率**，都不是精确分数）：

```cpp
/* sql/item.h:102 —— 「拍脑袋」的来源，无统计时全部走这些值 */
constexpr const float COND_FILTER_ALLPASS{1.0f};
/// Default filtering effect for the inequality predicate like "t1.a < 10"
constexpr const float COND_FILTER_INEQUALITY{0.3333f};
/// Default filtering effect for the between predicate
constexpr const float COND_FILTER_BETWEEN{0.1111f};
/// Filtering effect for the equality predicate like "t1.a = 10"
constexpr const float COND_FILTER_EQUALITY{0.1f};
/// Filtering effect for "t1.a is null"
constexpr const float COND_FILTER_ISNULL{0.05f};
/* 两个"陈旧"哨兵值 */
/// Filtering effect which is unknown at the moment
constexpr const float COND_FILTER_STALE{-1.0f};
/// Filtering effect of a stale filter which is not covering
constexpr const float COND_FILTER_STALE_NO_CONST{-2.0f};
```

> ★ 三者关系：`BETWEEN`(0.1111) ≈ `INEQUALITY`²(0.3333²)——即"两个不等式独立相乘"。这就是**列独立假设**在代码里的具体形态。
> ★ `COND_FILTER_STALE` / `_STALE_NO_CONST` 是**负数哨兵**，用于标记"读 const 表后过滤系数失效、需要重算"。

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `TABLE_LIST::embedding` / `nested_join` | 8.0 已重命名 **`Table_ref`**；字段仍在 `sql/table.h:3772/3774` |
| NULL-rejecting 判定叫 `simplify_join` | **不存在** → `Query_block::simplify_joins()`；NULL-rejecting 的真身是 **`Item::not_null_tables()`** |
| 决策 5 叫 `derived_condition_pushdown` / `push_condition_to_derived` | **两个都不存在**（前者只是 switch 名）→ **`Query_block::push_conditions_to_derived_tables()`** + **`Table_ref::can_push_condition_to_derived()`**（在 **`sql/sql_derived.cc:1028`**） |
| 决策 6 `resolve_const_item()` 是 prepare 期常量折叠 | **错**。它在 `sql/item.cc:9317`，只在 optimize 期的 `propagate_cond_constants()` 里调用；prepare 期的折叠真名是 **`simplify_const_condition()`**（`sql_resolver.cc:934`） |
| 决策 8 的类叫 `Aggregate_check` | **不存在** → **`Distinct_check`** 与 **`Group_check`**（`sql/aggregate_check.h:530/560`），共同基类 `Item_tree_walker` |
| 决策 10 裁剪叫 `prune_table_partitions()`（prepare 期） | 那是 **optimize 期**（`sql_optimizer.cc:2801`）。prepare 期是直接调 **`prune_partitions()`**（实现在 `sql/range_optimizer/partition_pruning.cc:251`） |
| `decide_subquery_strategy()` 属 prepare 期 | **错** → `JOIN::decide_subquery_strategy()`（`sql_optimizer.cc:11061`），在 `make_join_plan()` 的 5401 行 |
| `JOIN::optimize_aggregated_query()` | **不存在**。是**自由函数** `optimize_aggregated_query()`，在 **`sql/opt_sum.cc:276`** |
| 聚合"三态" | 实为**四态**：多一个 `AGGR_EMPTY`（MIN/MAX 走索引找不到匹配行） |
| `JOIN::make_const_table_list()` | **不存在**（0 命中）→ `extract_const_tables()` + `extract_func_dependent_tables()` + `mark_const_table()` |
| `SJ_OPT_*` 是 enum | 是 **`#define` 宏**（`sql/sql_select.h:309`）；且真名是 `SJ_OPT_DUPS_WEEDOUT` 不是 `DUPLICATE_WEEDOUT` |
| 五个 semi-join 策略同台比价 | **误解**。FM 与 LS **无条件先占坑**（不做 `cost < best_cost`）；真正比价的只有 MatLookup / MatScan / Weedout 三处 |
| `fix_semijoin_strategies_for_picked_join_order()` | **旧名**，注释里仍在用 → 真名 **`Optimize_table_order::fix_semijoin_strategies()`**（`sql_planner.cc:3360`） |
| condition filter 常量在 `sql_optimizer.h` | 在 **`sql/item.h:98-131`**；只有 3 个默认选择率（0.1 / 0.3333 / 0.1111） |
| `make_tmp_tables_info()` 在 `sql_tmp_table.cc` | 在 **`sql/sql_select.cc:4354`**；`sql_tmp_table.cc` 只管 `create_tmp_table()` |
| `QEP_TAB::OT_TMP_TABLE` | **不存在** → 真实 8 值：`OT_NONE / OT_AGGREGATE / OT_MATERIALIZE / OT_AGGREGATE_THEN_MATERIALIZE / OT_AGGREGATE_INTO_TMP_TABLE / OT_WINDOWING_FUNCTION / OT_BNL / OT_BKA` |
| `op_type` 由 `make_tmp_tables_info()` 直接赋值 | **不是** → 它在 4496/4639/4849 行调 **`setup_tmptable_write_func()`**（`sql/sql_executor.cc:509`）赋值；`OT_BNL/OT_BKA` 由 `QEP_TAB::init_join_cache()` 赋值 |
| `UseHashJoin()` 在 `sql_select.cc` | 真身在 **`sql/sql_executor.cc:2188`**（static）；文档引用的那一行**逐字正确** |
| `UseHashJoin()` 内部会看 optimizer_switch / 内存 | **不会**。函数体只有一行 return；所有前置条件在调用点外面 |
| `hash_join` 关掉能退回 BNL | **已不成立**。`OPTIMIZER_SWITCH_HASH_JOIN` 全 `sql/` 仅 **2 处**：`sql_const.h:220` 定义、`sys_vars.cc:209` 默认掩码，**无任何消费点** |
| `test_if_skip_sort_order()` 是纯规则 | **错**。它在 2453 行调 **`test_if_cheaper_ordering()`** 做真正的代价比较（`index_scan_time < read_time`） |
| 决策 26 是"纯规则、代价不参与" | **部分证伪**。主干确实纯规则，但 1548/1563 两次调 `test_if_skip_sort_order()` → 间接引入代价 |
| `JOIN_CACHE` 有 BNL/BKA/BNL_BKA 多算法 | **只有三个**：`ALG_NONE / ALG_BNL / ALG_BKA`（`sql/sql_join_buffer.h:30`，⚠️ 不是 `sql_join_cache.h`） |
| `CreateIteratorFromAccessPath()` 是递归的 | **是显式栈迭代**（`Mem_root_array<IteratorToBeCreated> todo` + while），注释说为省栈空间 |
| ICP 的条件是 5~6 条 | 注释列 **0~7 共 8 条**，实际 if 把 2/3/4/5/6/7 折进一行 |
| `sql/secondary_engine/` 目录 | **不存在**。只有 `sql/join_optimizer/secondary_engine_costing_flags.h`；阈值比较在 **`retry_with_secondary_engine()`**（`sql/sql_select.cc:916`） |
| `secondary_engine_cost_threshold` 是 `cost > 阈值` | 是 **`cost <= 阈值 → 不用`**（即严格大于才走），见 `sql_select.cc:961` |
| hypergraph 走 `setup_join_buffering()` | **不走**。hypergraph 直接构造 `AccessPath::HASH_JOIN`，只复用 `JOIN::push_to_engines()`（`finalize_plan.cc:760`） |
| `sql/opt_range.cc` | 8.0 已拆为 **`sql/range_optimizer/`** 目录 |

**★ 三个"结构性事实"**

1. **`dups_producing_tables` 是 semi-join 决策的真正兜底**。五个策略里只有 FirstMatch 与 LooseScan 能"消账"（`&= ~handled_by_fm_or_ls`），后三个比价分支都带 `|| pos->dups_producing_tables` 的强制接管条件。注释原文：*"we can't leave duplicate-producing tables not handled by any strategy"*。所以**Weedout 不是"最后才考虑的备胎"，而是"必须有人收底的保底"**——这解释了为什么复杂查询里 Weedout 出现频率远高于直觉。

2. **BNL 与 BKA 的分界就是 `tab->type()`，二者集合不相交**：`ALL/INDEX_SCAN/RANGE/INDEX_MERGE` → BNL；`SYSTEM/CONST/REF/EQ_REF` → BKA。而 BNL 在执行器里**必然**被改写成 hash join（`OT_BNL` 的注释原文 *"rewritten to hash join"*）。所以**"8.0 用 hash join 还是 BKA"本质上等价于"内层有没有索引"**——不需要任何开关参与。唯一的否决是 `QueryMixesOuterBKAAndBNL()`：**同一 query 里外连接 BKA 与 hash join 不能共存，触发时两者一起关掉**。

3. **`can_switch_from_ref_to_range()` 是启发式不是代价比较**——判据是"range 用到的 key 长度 > ref 用到的长度"。而且第 4 条准入（ref 依赖前序表的值时）明确**放弃启发式、交回 `best_access_path()` 的代价选择**，注释解释了理由：此时两者用的是条件的不同部分，可比性不成立。这条注释是理解"规则与代价边界模糊"的绝佳样本——**连作者自己都在代码里承认边界不清**（`@todo: This decision should rather be made in best_access_path()`）。

> 注：`sql/sql_planner.cc`、`sql/sql_optimizer.cc`、`sql/sql_select.cc` 等文件中的部分 `/* 【注】... */` 与 `// ★` 注释为本工作区人工批注，非 Oracle 上游原文。

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
