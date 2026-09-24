# 00 优化器总览：四阶段框架与两条优化路径

> 本篇回答三个基础问题：**优化器分几个阶段？逻辑优化和物理优化分别是什么？各篇怎么串起来？**

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [一、四阶段框架](#一四阶段框架)
- [二、逻辑优化 vs 物理优化](#二逻辑优化-vs-物理优化)
- [三、两条优化路径（greedy / hypergraph）](#三两条优化路径greedy--hypergraph)
- [四、JOIN::optimize 全流程映射](#四joinoptimize-全流程映射)
- [五、本目录各篇导航](#五本目录各篇导航)
- [六、一个贯穿式的例子](#六一个贯穿式的例子)

---

## 设计思想与理论基础

### CBO 与 RBO：MySQL 属于哪一类？

优化器按"**怎么选计划**"分两大阵营——这是理解优化器行为的第一层分类：

| | **RBO（Rule-Based，基于规则）** | **CBO（Cost-Based，基于代价）** |
|---|---|---|
| 决策依据 | 预定义的**规则/启发式**（"有索引就用索引"、"先做选择后做连接"） | **代价估算**（IO + CPU），选代价最小的 |
| 需要统计信息 | 不需要 | **需要**（行数、NDV、`rec_per_key`、直方图） |
| 优点 | 简单、可预测、优化本身开销小 | 能适应数据分布与数据量 |
| 缺点 | **不看数据**——数据倾斜时可能选出极差计划 | 依赖统计准确性；代价模型本身可能失真；优化有开销 |
| 代表 | Oracle 早期的 RULE 模式、早期 MySQL | System R 之后的所有现代优化器 |

**MySQL 的定位：以 CBO 为主，但保留了大量规则/启发式成分。** 这一点必须讲清楚，因为很多"优化器不听话"的现象正来自两类的混用：

**CBO 的部分**（看数据、算代价）：

- 代价模型（`Cost_model_table`：`page_read_cost` / `row_evaluate_cost` / `key_compare_cost`，见 `01_cost_model.md`）
- 访问方法选择：`ref` vs `range` vs 全表扫的比价（`best_access_path`）
- join order 的代价比较（`prefix_cost`）
- range 优化器对多种访问类型（range / skip scan / index merge）的比价

**规则与启发式的部分**（不看数据）：

- **逻辑改写几乎都是纯规则的**——子查询转 semi-join、IN→EXISTS、等值传播、常量传播、条件下推、外连接简化，都是"满足条件就改写"，不比较代价
- **join order 的剪枝是启发式的**——`prune_level` 的启发式剪枝使搜索**非穷举**，源码注释明说 "may miss the optimal QEPs"
- **无统计时的兜底常量**：`COND_FILTER_EQUALITY = 0.1`、`INEQUALITY = 0.3333`、`BETWEEN = 0.1111` —— 纯拍的
- **大量 `optimizer_switch` 开关**：本质是把规则/代价决策的**最终裁决权**暴露给 DBA 手动覆盖（这本身就是对"模型不够准"的承认）

**为什么现代优化器都是"规则 + 代价"的混合**：

> **规则负责生成等价的候选计划，代价负责从候选中挑最好的。**

纯 CBO 要枚举所有候选（指数级，太贵）；纯 RBO 不看数据（容易选差）。所以 Cascades 之后的主流架构都是"规则生成 + 代价选择"。MySQL 虽然**没有**实现 Cascades 的规则引擎，但整体遵循这个分工：

| 阶段 | 性质 | 做什么 |
|---|---|---|
| **逻辑优化** | 规则驱动 | 生成语义等价的候选（改写） |
| **物理优化** | 代价驱动 | 从候选中挑代价最低的实现 |

**历史脉络**：MySQL 早期 RBO 色彩更重（5.x 的 join order 就是贪心 + 固定规则，逻辑改写也更少）；8.0 持续 CBO 化——引入直方图、改进 condition filtering、用 estimator 计算选择率，都是为了让"代价"这部分更准、让"拍脑袋常量"用得更少。

---

### 优化器的理论谱系：System R → DPccp / DPhyp →（思想上的）Cascades

关系型查询优化器五十年的演进，join 枚举这条线是**三步走**，MySQL 恰好走到了第三步：

```
System R (1979)        DPccp (2006)            DPhyp (2008)
左深树 + 子集 DP   →   连通子图对，bushy 可达  →  + 超图，非内连接也合法
O(2^n)                 O(3^n)                   O(3^n) + 合法性编码进图
```

---

**1. System R 派：动态规划 + 代价模型（Selinger et al., SIGMOD 1979）**

《Access Path Selection in a Relational DBMS》奠定了现代优化器的范式，四个贡献：

- **代价模型**：用统计信息（基数、选择性）估算每个候选计划的 IO + CPU 代价
- **选择性估算**：假设各条件独立，`结果行数 = 输入行数 × 各过滤系数乘积`
- **自底向上动态规划**：按"已连接的表集合"递增（1 表 → 2 表 → … → n 表），每个子集**保留最优子计划**（memo），避免重复计算子问题
- **左深树假设**：每次只往已有前缀**尾部**加一张表，所以状态数是子集数 **O(2^n)**，而不是排列数 O(n!)
- **interesting orders（感兴趣的排序）**：最容易被忽略、影响却最深远的一条——**DP 状态不只是"表集合"，还要带上"输出是否按某组列有序"**。因为有序输出能省掉一次排序、或启用 merge join，所以同一个表集合要保留**多个不同 interesting order** 的最优计划

**MySQL 与 System R 的关系要分两层说**（这点常被含糊带过）：

| 层面 | 关系 |
|---|---|
| **思想上** | **继承**：代价模型、左深树、选择率估算这套范式直接沿用（见 `01_cost_model.md`） |
| **算法上** | **不是**：旧优化器**没有**实现 System R 的 DP——它不按子集建 memo 表，而是用"单路径 + 全局代价上界 + 可调深度"的 **DFS + 分支限界**（`join->positions[]` 是单条路径，不是 DP 表） |

⇒ 准确说法是"**思想源自 System R，但用贪心 + 剪枝取代了穷举 DP**"——牺牲最优性换规划时间。详见 `physical/06_join_order.md`。

---

**2. DPccp：从"只能左深"到"任意形状"（Moerkotte, 2006）**

System R 的左深假设把空间压到 O(2^n)，代价是 **bushy 计划完全不可达**——像 `(A⋈B)⋈(C⋈D)` 这种"两个分支各自先用选择性谓词缩小、再 join"的形状，在分析型查询里常常明显更优，却根本生成不出来。

Moerkotte 在《On the Correct and Complete Enumeration of the Core Search Space》里系统分析了"核心搜索空间"，给出 **DPccp**：

- 它枚举的不是"子集"，而是 **csg-cmp-pair（连通子图 + 连通补图对）**：
  - **csg**（connected subgraph）：一组通过连接谓词连通的表
  - **cmp**（complement）：csg 的补图中、与 csg 连通的那部分
  - 把 csg 的结果与 cmp 的结果 join，就得到一个候选
- **只枚举连通的组合** ⇒ 天然避免笛卡尔积（cross product），无需事后剔除
- **csg 与 cmp 都允许任意形状** ⇒ **bushy 可达**
- **复杂度**：csg-cmp-pair 的数量是 **O(3^n)**——比枚举所有子集对的 O(4^n) 好一个量级，但比左深 DP 的 O(2^n) 大一档。**这大一档就是"支持 bushy"要付的规划代价**

---

**3. DPhyp：DPccp + 超图（Moerkotte & Neumann, SIGMOD 2008）= MySQL 的 hypergraph**

《Dynamic Programming Strikes Back》在 DPccp 上再进一步：把查询表示成**超图**——关系是节点、连接谓词是边，而**外连接 / 反连接 / 半连接的重排序限制被编码成超边**。

意义在于：**"哪些连接顺序合法"这件事被一次性编译进图结构**，DPccp 的连通性枚举就自动只产生合法计划，不需要在搜索循环里反复检查"这个顺序会不会破坏外连接语义"。

MySQL 8.0 的 hypergraph 优化器就是 DPhyp 的实现（见 `physical/09_hypergraph.md`）。

> ⚠️ **常见误引**：这篇论文的出处是 **SIGMOD 2008**，不是 CIDR 2021。（CIDR 2021 是 Neumann 团队的另一批工作。）

---

**4. Cascades 派：规则 + 代价（Graefe, 1995）——思想上的另一条线**

《The Cascades Framework for Query Optimization》提出逻辑算子与物理算子分离：用**规则**做等价变换、用**代价**做选择。

MySQL 8.0 的 hypergraph 在**思想上**更接近这一派（逻辑枚举与代价驱动分离），但要说清楚：**它并没有实现 Cascades 的规则引擎**——MySQL 没有 Volcano/Cascades 那种可扩展的规则 + 物理属性框架，它的枚举核心仍是 DPhyp，改写逻辑则散落在 prepare/optimize 各阶段的具体函数里。

此外 hypergraph 还用到 **CD-C 冲突规则**（Moerkotte et al.）来表达外连接的重排序限制，少数无法折叠进超边的情况会在 `CostingReceiver` 里手工检查。

---

### 三代算法在 MySQL 源码中的落点

join 枚举经历三代算法。全景如下——**各代的详细源码落点见对应篇**（细节不在此重复）：

| 代 | 算法 | join 形状 | MySQL 落地 | 详见 |
|---|---|---|---|---|
| 一 | System R（Selinger 1979） | 左深树 | 旧优化器**思想上继承、算法上未采用**（DFS + 分支限界，无 DP 表） | [`physical/06_join_order.md`](physical/06_join_order.md) |
| 二 | DPccp（Moerkotte 2006） | bushy | **未实现**（全库搜索 `DPccp` 零命中） | — |
| 三 | DPhyp（Moerkotte & Neumann 2008） | bushy + **超图** | hypergraph 优化器直接实现（**非**经 DPccp） | [`physical/09_hypergraph.md`](physical/09_hypergraph.md) |

三点值得记住：

1. **MySQL 跳过了 DPccp**，直接实现 DPhyp——函数命名与分解完全按 DPhyp 的 csg / cmp / csg-cmp-pair，而不是 DPccp 那套无重复枚举的 partition 编号。
2. **旧优化器不是 System R 的 DP**：`join->positions` 分配大小是 `table_count`（**线性**）而非 2^N，没有按子集建 memo 表，是 DFS + 分支限界（`join->best_read` 作全局上界）。System R 真正留下的只有代价模型、选择率估算、左深树三个思想遗产。
3. **两条路径靠 `AccessPath` 汇合**：分叉在 `JOIN::optimize()` 的 `if (thd->lex->using_hypergraph_optimizer())`，合流在 `JOIN::m_root_access_path`——但**产出方式不同**：旧优化器是末端 `create_access_paths()` **事后翻译** QEP_TAB 数组，hypergraph 则**原生**以 AccessPath 作为 DP 状态。

### 为什么 MySQL 有两条优化路径

这是 MySQL 最大的历史包袱，也是最值得理解的一点：

| | 旧优化器（greedy） | 新优化器（hypergraph） |
|---|---|---|
| 起源 | 5.x 一路演进，源自 System R 左深树 | **8.0.22** 引入（`optimizer_switch` 的 `hypergraph_optimizer`，实验特性），源自 DPhyp 论文 |
| join 形状 | **只支持左深树** | 支持 bushy tree（任意形状） |
| 算法 | 贪心 + 剪枝 | `EnumerateAllConnectedPartitions`（DPhyp） |
| 默认 | 默认开启 | `optimizer_switch=hypergraph_optimizer` 开启 |

**为什么不能直接替换**：旧优化器经过几十年打补丁，承载了无数特殊 case（semi-join 策略、子查询物化决策、hint 语义），行为已被用户和测试依赖。hypergraph 是"另起炉灶"的新实现，需要长周期灰度切换——所以 8.0 是**两条并行**，靠 AccessPath（08 篇）这个统一 IR 汇合。

### 逻辑优化 vs 物理优化的分野

四阶段框架（下面「一」节）的骨架，本质是 **Cascades 思想的简化落地**：

- **逻辑优化** = 逻辑等价变换（规则），不涉及具体执行方式
- **物理优化** = 在等价逻辑表达式里，用代价挑具体实现（访问方法、join 顺序、join 方式）

"先规则、后代价"——规则负责缩小空间，代价负责挑出最优。这是本目录 24 篇组织逻辑的根源。

---

## 一、四阶段框架

内核月报 2024/04 把 MySQL 优化器分成四阶段。这是理解优化器的主线：

```
① 逻辑优化     对查询做逻辑等价变换：semi-join 拍平、子查询解关联、外连接转内连接、
               等值传播、常量折叠、谓词下推、视图/derived 合并。
               变换后查询有更优的执行计划、更多计划选择。

② 初始优化分析  对各表可能访问路径做扫描行数和代价分析，帮助后续选基表路径；
               同时分析出 Const Table（优化期间就执行并当常量用）。

③ 物理优化      对表访问路径（哪个索引）、访问方式（REF / RANGE / 各 range 类型）、
               Join Order 和 Join 方式（Nested Loop / Hash Join）做选择。

④ 计划改进      索引谓词下推引擎（ICP）、Ordering index 选择避免排序、访问方式微调。
```

**关键理解**：①是"把查询改写成更利于优化的样子"，③是"在改写后的形态里用代价挑执行方式"。**先规则后代价**。

### 四阶段的时间分布（重要）

| 阶段 | 发生在 | 特征 |
|---|---|---|
| ① 逻辑优化（大部分） | **prepare 期**（`Query_block::prepare`） | **永久变换**，`assert(first_execution)`——PS 只做一次 |
| ① 逻辑优化（谓词类） | **optimize 期**（`optimize_cond`） | **临时变换**，每次 execute 重做 |
| ② 初始分析 | optimize 期（`make_join_plan` 前段） | 统计 + const 表 |
| ③ 物理优化 | optimize 期（`make_join_plan`） | 代价搜索 |
| ④ 计划改进 | optimize 期（`make_join_plan` 之后） | 收尾 |

> 逻辑优化横跨 prepare 和 optimize 两个阶段，这是 MySQL 历史演进的痕迹。见 [04 篇](logical/04_logical_join.md)（prepare 期的连接简化）与 [05 篇](logical/05_logical_predicate.md)（optimize 期的谓词优化）。

---

## 二、逻辑优化 vs 物理优化

| | 逻辑优化（RBO，规则驱动） | 物理优化（CBO，代价驱动） |
|---|---|---|
| 输入 | 逻辑查询树（已 fix_fields） | 逻辑优化后的等价查询树 + 统计信息 |
| 输出 | 等价的、更优的查询树 | 物理执行计划（AccessPath） |
| 手段 | 等价变换规则 | 代价模型 + 搜索算法 |
| 保真 | **保语义**（必须等价） | 不必等价，只求代价最小 |
| 典型 | 子查询拍平、外连接转内、等值传播、谓词下推 | join order、访问方法、join 算法 |
| MySQL 代码 | prepare 改写 + `optimize_cond` + `simplify_joins` | `make_join_plan`（greedy / DPhyp） |

**代价从哪来**：物理优化的判据是代价模型，代价模型的输入是统计信息——所以 [01 篇](01_cost_model.md)（代价 + 统计）是 ②③ 的前提。

### 一个容易混淆的点：semi-join 属于①还是③？

**两者都有**：

- **①（prepare）**：`IN` 子查询被判为 semi-join 候选后**拍平**成 `t1 SEMI JOIN t2 ON ...`（语法结构改写）→ [02 篇](logical/02_subquery.md)、[03 篇](logical/03_semijoin.md)
- **③（optimize）**：在 join order 搜索中**选择** five 种执行策略之一（FirstMatch/LooseScan/DuplicateWeedout/Materialize）→ [03 篇 4 节](logical/03_semijoin.md)、[06 篇](physical/06_join_order.md)

---

## 三、两条优化路径（greedy / hypergraph）

8.0 有**两个**物理优化器，硬分叉（`sql_optimizer.cc:610`）：

| | 旧优化器（默认） | hypergraph（实验性） |
|---|---|---|
| 搜索算法 | 贪心 + 限深 DFS（7 表分界） | **DPhyp**（完全 DP） |
| 支持的 join 树 | **只支持左深树** | **支持 bushy tree** |
| 中间表示 | `POSITION` / `QEP_TAB` | 直接产出 AccessPath |
| semi-join | five 种策略模拟 | 一等公民（`RelationalExpression::SEMIJOIN`） |
| 表数上限 | 61（table_map 位图） | 61（NodeMap） |
| 状态 | 默认 | 默认 off，debug 构建才可用 |
| 篇章 | [06](physical/06_join_order.md) [07](physical/07_access_method.md) [08](physical/08_range_optimizer.md) | [09](physical/09_hypergraph.md) |

**汇合点**：两条路径都产出 **AccessPath 树**，后续（计划改进、代码生成、执行器）完全共用。

---

## 四、JOIN::optimize 全流程映射

`JOIN::optimize()`（`sql/sql_optimizer.cc:338`）：

| 行号 | 步骤 | 阶段 | 详见 |
|------|------|------|------|
| 363-370 | `count_field_types` / `setup_windows2` | 预处理 | |
| 404-420 | 优化派生表/视图/表函数 | ①（递归） | [04 篇](logical/04_logical_join.md) |
| **473-486** | WHERE `optimize_cond`（等值传播→常量传播→去恒真恒假） | **①** | [05 篇](logical/05_logical_predicate.md) |
| 487-500 | HAVING `optimize_cond` | ① | 同上 |
| 502-506 | `prune_table_partitions` 分区裁剪 | ① | [12 篇](12_partition_pruning.md) |
| 514-571 | `optimize_aggregated_query`（聚合常量化三态） | ① | [15 篇 2 章](15_groupby_distinct_order.md) |
| 600-606 | `substitute_gc`（生成列替换） | ① | [17 篇 决策 11](17_optimizer_decisions.md) |
| **610-679** | **hypergraph 分支**（若开启则到此 return） | ③ | [09 篇](physical/09_hypergraph.md) [18 篇](physical/18_hypergraph_advanced.md) |
| **696** | **`make_join_plan()`** | **②③** | [06](physical/06_join_order.md) [07](physical/07_access_method.md) [17 篇](17_optimizer_decisions.md) |
| 736-748 | `substitute_for_best_equal_field`（等价类展开成星形等值） | ①（需 join 顺序先定） | [05 篇 1.5](logical/05_logical_predicate.md) |
| 770-781 | `init_ref_access` / `make_join_query_block` | ③ 收尾 | [05 篇 3 节](logical/05_logical_predicate.md) |
| 796 | `optimize_distinct_group_order` | ① | [15 篇 3 章](15_groupby_distinct_order.md) |
| 894-916 | `setup_join_buffering`（BNL/BKA/HashJoin） | ③ | [10 篇](10_plan_refinement.md) [17 篇 决策 25](17_optimizer_decisions.md) |
| **1011-1023** | `alloc_qep` / `test_skip_sort` / `finalize_table_conditions` / `make_join_readinfo` / `make_tmp_tables_info` | **④ 计划改进** | [10 篇](10_plan_refinement.md) [15 篇 4/6 章](15_groupby_distinct_order.md) |
| **1037** | **`create_access_paths()`** | 代码生成 | [`../08_access_path.md`](../08_access_path.md) |
| 1064 | `push_to_engines`（引擎条件下推） | ④ | [10 篇](10_plan_refinement.md) |

> **注意**：逻辑优化（①）不是一次性完成的。它分布在 `JOIN::optimize` 前段（`optimize_cond`）+ prepare 阶段（semi-join/derived merge/连接简化）+ 部分在 join 顺序之后（`substitute_for_best_equal_field`，因为"最优字段"依赖 join 顺序）。

### prepare 阶段的逻辑优化（早于 JOIN::optimize）

```
Query_block::prepare
  ├─ resolve_subquery → flatten_subqueries     sql_resolver.cc:3811   [02 篇]
  ├─ resolve_placeholder_tables → merge_derived sql_resolver.cc:3462  [04 篇]
  └─ apply_local_transforms → simplify_joins    sql_resolver.cc:1951  [04 篇]
```

---

## 五、本目录各篇导航

> 本目录共 **24 篇**：根下 15 篇 + `logical/` 4 篇 + `physical/` 5 篇。按"主链阶段 → 专项 → 横切"三组导航。

```
optimizer/
├── 【基础】
│   ├── 00_overview.md        本篇：两阶段模型、两条优化路径、各篇导航
│   └── 01_cost_model.md      代价模型 + 统计信息（②③ 的前提）
├── 【① 逻辑优化】logical/
│   ├── 02_subquery.md          子查询改写决策树
│   ├── 03_semijoin.md          semi-join（五策略、改写/优化/执行）
│   ├── 04_logical_join.md      连接简化（外连接转内、derived/view/CTE merge）
│   └── 05_logical_predicate.md 谓词优化（等值传播、常量折叠、下推、ICP）
├── 【②③ 物理优化】physical/
│   ├── 06_join_order.md        join order 搜索（greedy + 限深 DFS + 剪枝）
│   ├── 07_access_method.md     访问方法选择（const 表 / ref / range / scan）
│   ├── 08_range_optimizer.md   range 优化（SEL_TREE 区间森林）
│   ├── 09_hypergraph.md        hypergraph（DPhyp 算法级）
│   └── 18_hypergraph_advanced.md  hypergraph 进阶（CSE、最终化、与经典优化器对照）
├── 【④ 计划改进】
│   └── 10_plan_refinement.md  计划改进阶段
├── 【专项：某类查询/对象的优化】
│   ├── 11_optimizer_hints.md   optimizer hint 体系
│   ├── 12_partition_pruning.md 分区裁剪
│   ├── 13_functional_mv_index.md 功能索引与多值索引
│   ├── 15_groupby_distinct_order.md GROUP BY / DISTINCT / ORDER BY
│   ├── 19_set_operation.md     UNION / INTERSECT / EXCEPT
│   ├── 20_view_resolution.md  视图解析与合并
│   └── 21_collation_index_usability.md collation 与索引可用性
└── 【横切：对象模型 / 决策集 / 演进 / 观测】
    ├── 14_plan_stability.md   计划稳定性（★ 外部知识对照：SPM/他库方案，非 8.0.39 实现）
    ├── 16_join_object_model.md JOIN/POSITION/QEP_TAB 对象模型
    ├── 17_optimizer_decisions.md 优化器决策全集（数十个 if 的"为什么"）
    ├── 22_optimizer_worklog_timeline.md WL 与版本演进时间线
    └── 23_optimizer_trace_internals.md optimizer trace 内部实现
```

**推荐阅读顺序**：

1. **入门主链**：00 → 01 → 06 → 07（框架 → 代价 → join order → 访问方法）
2. **深入物理优化**：08（range 是最复杂的访问方法）→ 09 → 18（hypergraph）
3. **深入逻辑优化**：03（semi-join）→ 04（连接简化）→ 05（谓词）
4. **查具体决策**：17（决策全集，按主题索引）、16（对象模型）
5. **排查与演进**：23（trace 怎么用）→ 22（什么版本改了什么）

---

## 六、一个贯穿式的例子

```sql
SELECT t1.a FROM t1 WHERE t1.a IN (SELECT t2.b FROM t2 WHERE t2.c > 5);
```

各阶段依次发生：

| # | 阶段 | 发生什么 | 详见 |
|---|---|---|---|
| 1 | **① 子查询改写**（prepare） | `IN` 子查询判为 semi-join 候选（16 条准入条件），拍平成 `t1 SEMI JOIN t2 ON t1.a=t2.b WHERE t2.c>5` | [02](logical/02_subquery.md) [03](logical/03_semijoin.md) |
| 2 | **① 连接简化**（prepare） | `simplify_joins` 检查能否转内连接（此处是 semi-join nest，不转）；若外层有拒 NULL 谓词则会转 | [04 篇 3 节](logical/04_logical_join.md) |
| 3 | **① 等值传播**（optimize） | `optimize_cond` → `build_equal_items` 把 `t1.a=t2.b` 建成等价类 `{t1.a, t2.b}` | [05 篇 1 节](logical/05_logical_predicate.md) |
| 4 | **② 初始分析** | `extract_const_tables` 检测 const 表；`update_ref_and_keys` 生成 Key_use；`estimate_rowcount` 首次 range 分析 | [07 篇](physical/07_access_method.md) |
| 5 | **③ 物理优化** | `make_join_plan`：greedy search 定 join 顺序、`best_access_path` 选访问方法（`t2.c>5` → range）、semi-join 策略选择 | [06](physical/06_join_order.md) [07](physical/07_access_method.md) [08](physical/08_range_optimizer.md) |
| 6 | **④ 计划改进** | `test_if_skip_sort` 看能否用索引顺序；`make_join_readinfo` → `push_index_cond` 把条件下推为 ICP | [10 篇](10_plan_refinement.md) |
| 7 | **代码生成** | `create_access_paths` 产出 AccessPath 树 → `CreateIteratorFromAccessPath` 生成迭代器 | [`../08_access_path.md`](../08_access_path.md)、[`../09_executor_iterator.md`](../09_executor_iterator.md) |

---


## 参考

**论文**
- **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979，System R)** —— 代价模型、选择性估算、左深树 DP。旧优化器的理论源头
- **Graefe《The Cascades Framework for Query Optimization》(1995)** —— 现代优化器框架（规则 + 代价）。MySQL 未采用，但是 hypergraph 重构的参照系

**官方文档**
- *MySQL 8.0 Reference Manual → Optimization*（ch.10）
- *MySQL 8.0 Reference Manual → Switchable Optimizations*（`optimizer_switch` 全部开关）

**内核月报**
- **2024/04《MySQL 查询优化分析 - 基础概念》** —— 本目录"四阶段框架"的直接来源

**其他**
- 知乎 - 从优化器综述论文学习System-R框架和Cascade框架： https://zhuanlan.zhihu.com/p/611439035
- 查询优化器：从 System R 到 Cascades: https://quant67.com/post/algorithms/60-query-optimizer/query-optimizer.html
- 知乎 - An Overview of Query Optimization in Relational Systems：https://zhuanlan.zhihu.com/p/463696413
- https://zhuanlan.zhihu.com/p/1943396105124574367
- https://zhuanlan.zhihu.com/p/632565261
- https://www.mirrorship.cn/zh-CN/blog/d/273649
- https://download.csdn.net/blog/column/2727851/132138500
- https://developer.aliyun.com/article/789923

