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

### ★ 判据一：谁决定用 hypergraph？

不是 `JOIN::optimize()` 自己决定的，而是在 **`Sql_cmd_dml::prepare()` 开表之后、进 prepare 之前**一次性从 `optimizer_switch` 快照到 `LEX` 上：

```cpp
// [O5] sql/sql_select.cc —— Sql_cmd_dml::prepare()
  if (open_tables_for_query(thd, lex->query_tables, ...)) { /* ... */ }
  DEBUG_SYNC(thd, "after_open_tables");

  // ★ 分派判据：整条语句在此刻冻结"用哪个优化器"，之后全程不再看 optimizer_switch
  lex->set_using_hypergraph_optimizer(
      thd->optimizer_switch_flag(OPTIMIZER_SWITCH_HYPERGRAPH_OPTIMIZER));

  {
    if (prepare_inner(thd)) goto err;                  // ← Query_block::prepare()
  }
```

```cpp
// [O7] sql/sql_lex.h —— 注释原文即"可安全地在任何阶段查询"
  /**
    Whether the currently-running query should be (attempted) executed in
    the hypergraph optimizer. This will not change after the query is
    done parsing, so you can use it in any query phase to e.g. figure out
    whether to inhibit some transformation that the hypergraph optimizer
    does not properly understand yet.
   */
  bool using_hypergraph_optimizer() const { return m_using_hypergraph_optimizer; }
```

⚠️ **一处必须纠正的常见说法**：8.0.39 里 `hypergraph_optimizer` **仍是实验特性**——开关默认 off，且 `check_optimizer_switch()` 写着：只有 `#ifdef WITH_HYPERGRAPH_OPTIMIZER`（debug/实验构建）才允许打开，打开时 `push_warning(ER_WARN_HYPERGRAPH_EXPERIMENTAL)`；非该构建直接报 `ER_HYPERGRAPH_NOT_SUPPORTED_YET`（"use in non-debug builds"）。

```cpp
// sql/sys_vars.cc
  } else if (!current_hypergraph_optimizer && want_hypergraph_optimizer) {
#ifdef WITH_HYPERGRAPH_OPTIMIZER
    push_warning(thd, Sql_condition::SL_WARNING, ER_WARN_DEPRECATED_SYNTAX,
                 ER_THD(thd, ER_WARN_HYPERGRAPH_EXPERIMENTAL));
    return false;                                      // Allow, with a warning.
#else
    my_error(ER_HYPERGRAPH_NOT_SUPPORTED_YET, MYF(0), "use in non-debug builds");
    return true;
#endif
  }
```

### ★ 判据二：`JOIN::optimize()` 里的硬分叉

分叉点在阶段 1 之后——**两条路径共享阶段 0/1**（派生表优化 + `optimize_cond` + 分区裁剪 + 聚合常量化），然后彻底分开：

```cpp
// [O1]
  if (thd->lex->using_hypergraph_optimizer()) {
    // 去掉 IN→EXISTS 加进去的条件，以便"带 in2exists / 不带 in2exists"各规划一次
    Item *where_cond_no_in2exists  = remove_in2exists_conds(where_cond);
    Item *having_cond_no_in2exists = remove_in2exists_conds(having_cond);

    // ★ hypergraph 入口：直接产出一棵 AccessPath
    m_root_access_path = FindBestQueryPlan(thd, query_block, trace_ptr);
    if (finalize_access_paths && m_root_access_path != nullptr) {
      if (FinalizePlanForQueryBlock(thd, query_block)) return true;
    }

    // 若本层被 IN→EXISTS 改写过，外层可能想改走物化 → 再规划一份"无 in2exists"的备着
    if (where_cond != where_cond_no_in2exists ||
        having_cond != having_cond_no_in2exists) {
      where_cond  = where_cond_no_in2exists;
      having_cond = having_cond_no_in2exists;
      m_root_access_path_no_in2exists = FindBestQueryPlan(thd, query_block, trace_ptr);
    }

    if (m_root_access_path == nullptr) return true;
    set_plan_state(PLAN_READY);
    return false;                                      // ★ 到此 return，绝不进入旧路径
  }

  // ----------------------------------------------------------------------------
  //       All of this is never called for the hypergraph join optimizer!
  // ----------------------------------------------------------------------------
  assert(!thd->lex->using_hypergraph_optimizer());
```

### 路径 A：旧优化器 —— 一维数组 ⇒ 只能左深

产出是**线性的 `QEP_TAB[]` 数组**（`best_ref[]` 已按最优顺序排好），再由 `ConnectJoins()` 从左往右**折叠成一棵左深树**：

```cpp
// [O8] —— 左深树的构造：path 永远是"已折叠好的左子树"，新表永远在右边
AccessPath *ConnectJoins(plan_idx upper_first_idx, plan_idx first_idx,
                         plan_idx last_idx, QEP_TAB *qep_tabs, THD *thd, ...) {
  AccessPath *path = nullptr;
  for (plan_idx i = first_idx; i < last_idx;) {
    Substructure substructure = FindSubstructure(...);
    if (substructure == Substructure::OUTER_JOIN ||
        substructure == Substructure::SEMIJOIN) {
      // 外连接/半连接是一个子树（可能多表），递归下去再把返回的根接回来
      AccessPath *subtree_path = ConnectJoins(first_idx, i, substructure_end, ...);
    }
    AccessPath *table_path = GetTableAccessPath(thd, qep_tab, qep_tabs);

    if (path == nullptr) {
      path = table_path;                               // 第一张表：树根
    } else {
      if (is_bka) {
        path = CreateBKAAccessPath(thd, ..., path, left_tables,
                                   table_path, right_tables, ..., JoinType::INNER);
      } else if (replace_with_hash_join) {
        path = CreateHashJoinAccessPath(thd, qep_tab, path, left_tables,
                                        table_path, right_tables, JoinType::INNER, ...);
      } else {
        path = CreateNestedLoopAccessPath(             // ★ 左深：新表永远做 inner
            thd, path, table_path, JoinType::INNER, ...);
        SetCostOnNestedLoopAccessPath(*thd->cost_model(), qep_tab->position(), path);
      }
    }
  }
  return path;
}
```

★ **"只支持左深树"的机械原因**：`POSITION`/`QEP_TAB` 是**一维数组**，`best_positions[i]` 描述的是"前 i+1 张表的前缀"，没有任何结构能表达"两个各自已成型的子树再 join"。`ConnectJoins()` 里唯一能把子树放左边的情况是外连接/半连接的 nest 递归——那是**语法结构决定的 nest**，不是代价搜索选出来的 bushy。

### 路径 B：hypergraph —— DP 状态即 AccessPath 本身

```cpp
// [O3] sql/join_optimizer/join_optimizer.cc
AccessPath *FindBestQueryPlan(THD *thd, Query_block *query_block, string *trace) {
  if (CheckSupportedQuery(thd)) return nullptr;        // 不支持的查询直接退回

  // 1) 把 join 结构编译成超图：表=节点，连接谓词=边
  JoinHypergraph graph(thd->mem_root, query_block);
  if (MakeJoinHypergraph(thd, trace, &graph, &where_is_always_false)) return nullptr;

  FindSargablePredicates(thd, trace, &graph);
  CacheCostInfoForJoinConditions(thd, query_block, &graph, trace);

  // 2) 收集 interesting orders（ORDER BY / GROUP BY / semi-join / 窗口）
  LogicalOrderings orderings(thd);
  BuildInterestingOrders(thd, &graph, query_block, &orderings, ...);

  // 3) ★ 真正的 join 枚举：DPhyp 的 csg-cmp-pair（连通子图 + 连通补图对）
  CostingReceiver receiver(thd, query_block, graph, &orderings, ...);
  if (graph.edges.empty()) {
    receiver.FoundSingleNode(0);                       // 单表快路径
  } else if (EnumerateAllConnectedPartitions(graph.graph, &receiver) && ...) {
    SimplifyQueryGraph(thd, ..., &graph, trace);
    /* ... 用简化后的超图重跑一次 ... */
  }

  // 4) root_candidates = Pareto 前沿（代价/行数/init_cost/顺序性各有所长）
  Prealloced_array<AccessPath *, 4> root_candidates = receiver.root_candidates();

  // 5) 后置算子逐层往上叠，每步仍是"多候选 + ProposeAccessPath 淘汰"
  /* 5a FILTER → 5b GROUP BY（能不排序就流式聚合）→ 5c HAVING
     → 5d WINDOW → 5e DISTINCT+ORDER BY → 5f LIMIT → 5g DELETE/UPDATE */

  // 6) 所有 metric 到此只剩代价有意义：取 min(cost)
  AccessPath *root_path =
      *std::min_element(root_candidates.begin(), root_candidates.end(),
                        [](const AccessPath *a, const AccessPath *b) {
                          return a->cost < b->cost;
                        });
  return root_path;
}
```

★ `CostingReceiver::ProposeNestedLoopJoin()` 的签名就暴露了 bushy 能力——**左、右都是任意 `NodeMap` 子集**，而不是"已定前缀 + 一张新表"：

```cpp
void CostingReceiver::ProposeNestedLoopJoin(
    NodeMap left, NodeMap right, AccessPath *left_path, AccessPath *right_path, ...) {
  if (Overlaps(left_path->parameter_tables, right)) return;  // 外表不能引用内表的值
  AccessPath join_path;
  join_path.type = AccessPath::NESTED_LOOP_JOIN;
  join_path.nested_loop_join().outer = left_path;      // ← 任意形状的子树
  join_path.nested_loop_join().inner = right_path;     // ← 任意形状的子树
  ...
}
```

⇒ **DP 状态即 AccessPath 本身**：每枚举一个连通子集就存一棵 AccessPath；两子集 join 时直接把两棵子树挂到新节点上。所以 `(A⋈B)⋈(C⋈D)` 是**自然产生**的形状，不需要任何"bushy 开关"。

### 两条路径对比

```
                    JOIN::optimize()  阶段 0/1 共享
                              │
              thd->lex->using_hypergraph_optimizer()   [O7] 由 optimizer_switch 快照
                    ┌─────────┴──────────┐
                  no│                    │yes
                    ▼                    ▼
 ┌──────────────────────────────┐  ┌────────────────────────────────────────┐
 │ 旧优化器（greedy）             │  │ hypergraph（DPhyp）                      │
 │ make_join_plan()      [O1]   │  │ FindBestQueryPlan()             [O3]    │
 │  ├ update_ref_and_keys       │  │  ├ MakeJoinHypergraph()  → 超图          │
 │  ├ extract_const_tables      │  │  ├ BuildInterestingOrders()             │
 │  ├ estimate_rowcount()       │  │  ├ EnumerateAllConnectedPartitions()    │
 │  └ choose_table_order() [O6] │  │  │    └ CostingReceiver                 │
 │      └ greedy_search()       │  │  │        ProposeHashJoin/NL/BKA        │
 │      → best_positions[]      │  │  └ 后置算子：FILTER/AGG/WINDOW/          │
 │ get_best_combination()       │  │       SORT/DISTINCT/LIMIT              │
 │  → JOIN_TAB[]（线性前缀）      │  │  → root_candidates，取 min(cost)         │
 │ alloc_qep() → QEP_TAB[]      │  │ FinalizePlanForQueryBlock()             │
 │ create_access_paths()  [O8]  │  │                                        │
 │  └ ConnectJoins() 事后翻译    │  │ 产出：AccessPath 树（原生，bushy 可达）   │
 │ 产出：AccessPath 树（左深）    │  │                                        │
 └──────────────┬───────────────┘  └───────────────────┬────────────────────┘
                └──────────────┬───────────────────────┘
                               ▼                  JOIN::m_root_access_path
                               ▼        CreateIteratorFromAccessPath() → 执行（两路径共用）

旧路径能表达的 join 形状（左深）        hypergraph 能表达的（bushy）
        NLJ                                    HASH_JOIN
        /  \                                   /        \
     NLJ    t3                            HASH_JOIN   HASH_JOIN
     /  \                                 /     \     /     \
   t1    t2                              t1     t2   t3     t4
（新表只能挂在右边）                  （两侧都是独立成型的子树）
```

> AccessPath 节点本体见 [`08_access_path.md`](../08_access_path.md)；代价公式见 [`01_cost_model.md`](01_cost_model.md)；连接顺序搜索算法见 [`physical/06_join_order.md`](physical/06_join_order.md)——本篇只在接口处导航。
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

### ★ `JOIN::optimize()` 源码骨架（按源码自带的四段组织）

> 源码坐标：`[O1] sql/sql_optimizer.cc` · `[O5] sql/sql_select.cc` · `[O6] sql/sql_planner.cc` · `[O8] sql/sql_executor.cc`

源码的 doxygen 注释把流程分为 `Logical transformations` → `cost-based optimization (见 JOIN::make_join_plan)` → `Post-join order optimization` → `Code generation`。**与本篇"四阶段框架"的差异**：源码没有独立的「②初始优化分析」阶段——统计/const 表抽取被算进 `make_join_plan()` 前半段；源码的 `Code generation` 覆盖本篇的④ + AccessPath 生成。

**阶段 0：入口与短路**

```cpp
// [O1] —— JOIN::optimize 入口
bool JOIN::optimize(bool finalize_access_paths) {
  if (optimized) return false;                       // 防重入：EXPLAIN 会走两遍
  THD_STAGE_INFO(thd, stage_optimizing);

  Opt_trace_object trace_optimize(trace, "join_optimization");   // trace 顶层节点
  Opt_trace_array trace_steps(trace, "steps");

  if (has_windows && Window::setup_windows2(thd, &m_windows)) return true;
  if (query_block->olap == ROLLUP_TYPE && optimize_rollup()) return true;
  if (query_block->get_optimizable_conditions(thd, &where_cond, &having_cond))
    return true;                                     // 取 WHERE/HAVING 到 JOIN

  set_optimized();

  // ★ 递归优化本层用到的所有派生表 / 视图 / 表函数（① 是递归的）
  for (Table_ref *tl = query_block->leaf_tables; tl; tl = tl->next_leaf) {
    if (tl->is_view_or_derived()) {
      if (tl->optimize_derived(thd)) return true;    // → 内层 Query_block::optimize()
    } else if (tl->is_table_function()) { /* ... */ }
  }

  // hypergraph 专用：把内层子查询也先优化掉，并禁掉 const 表与优化期子查询求值
  if (thd->lex->using_hypergraph_optimizer()) {
    for (Query_expression *unit = query_block->first_inner_query_expression();
         unit; unit = unit->next_query_expression()) {
      if (!unit->is_optimized() && unit->optimize(thd, nullptr, false, false))
        return true;
    }
    query_block->add_active_options(OPTION_NO_CONST_TABLES |
                                    OPTION_NO_SUBQUERY_DURING_OPTIMIZATION);
  }

  // ⚡ 零结果短路：不进入任何搜索，直接造 ZERO_ROWS 的 AccessPath
  if (query_expression()->select_limit_cnt == 0 && !calc_found_rows) {
    zero_result_cause = "Zero limit";
    best_rowcount = 0;
    create_access_paths_for_zero_rows();
    goto setup_subq_exit;
  }
```

**阶段 1：① 逻辑优化（纯规则，每次 execute 重做）**

```cpp
  // WHERE：equality_propagation → constant_propagation → trivial_condition_removal
  if (where_cond || query_block->outer_join) {
    if (optimize_cond(thd, &where_cond, &cond_equal, &query_block->m_table_nest,
                      &query_block->cond_value)) return true;
    if (query_block->cond_value == Item::COND_FALSE) {
      zero_result_cause = "Impossible WHERE";        // 常量折叠出 FALSE
      create_access_paths_for_zero_rows();
      goto setup_subq_exit;
    }
  }
  if (having_cond) { /* 同样三步，但 join_list == nullptr ⇒ 不做等值传播 */ }

  if (query_block->partitioned_table_count && prune_table_partitions()) return true;

  // 隐式分组下把 COUNT(*)/MIN()/MAX() 常量化
  if (tables_list && implicit_grouping && ...) {
    aggregate_evaluated outcome;
    if (optimize_aggregated_query(thd, query_block, *fields, where_cond, &outcome))
      return true;
    switch (outcome) {
      case AGGR_REGULAR:  break;                     // 没优化掉
      case AGGR_DELAYED:  select_count = true; break;// 引擎支持 HA_COUNT_ROWS_INSTANT
      case AGGR_COMPLETE:                            // 全部求值完毕 → 只造一行
        zero_result_cause = "Select tables optimized away";
        m_root_access_path = attach_access_paths_for_having_and_limit(
            NewFakeSingleRowAccessPath(thd, true));
        goto setup_subq_exit;
      case AGGR_EMPTY:
        zero_result_cause = "No matching min/max row";
        create_access_paths_for_zero_rows();
        goto setup_subq_exit;
    }
  }

  if (tables_list == nullptr) {                      // 无表可查（SELECT 1）
    if (make_tmp_tables_info()) return true;
    set_plan_state(NO_TABLES);
    create_access_paths();
    return false;
  }

  if ((where_cond || !group_list.empty() || !order.empty()) &&
      substitute_gc(thd, query_block, where_cond, group_list.order, order.order))
    count_field_types(...);                          // 生成列替换
```

`optimize_cond()` 是①的核心，三步改写名字直接写进 optimizer trace：

```cpp
bool optimize_cond(THD *thd, Item **cond, COND_EQUAL **cond_equal,
                   mem_root_deque<Table_ref *> *join_list,
                   Item::cond_result *cond_value) {
  if (join_list) {                                   // HAVING 不做等值传播
    step_wrapper.add_alnum("transformation", "equality_propagation");
    // a) x=y AND y=z ⇒ 多重等值 MEP {x,y,z}；b) 有常量则代入
    if (build_equal_items(thd, *cond, cond, nullptr, true, join_list, cond_equal))
      return true;
  }
  if (*cond) {                                       // field = const 传播
    step_wrapper.add_alnum("transformation", "constant_propagation");
    if (propagate_cond_constants(thd, nullptr, *cond, *cond)) return true;
  }
  if (*cond) {                                       // 去恒真/恒假
    step_wrapper.add_alnum("transformation", "trivial_condition_removal");
    if (remove_eq_conds(thd, *cond, cond, cond_value)) return true;
  }
  return false;
}
```

**阶段 2：② 初始分析 + ③ 物理优化 —— `make_join_plan()`（旧路径心脏）**

```cpp
bool JOIN::make_join_plan() {
  if (init_planner_arrays()) return true;            // join_tab / best_ref / positions

  if (query_block->outer_join || query_block->is_recursive()) {
    if (propagate_dependencies()) return true;       // 外连接/递归 CTE 的顺序依赖
    init_key_dependencies();
  }

  // ② 生成 Key_use[]（ref 访问的基础）与 sargable 谓词
  if (where_cond || query_block->outer_join) {
    if (update_ref_and_keys(thd, &keyuse_array, join_tab, tables, where_cond,
                            ~query_block->outer_join, query_block, &sargables))
      return true;
  }

  // ③ 半连接表外提（只做一次）
  if (!query_block->sj_pullout_done && !query_block->sj_nests.empty() &&
      pull_out_semijoin_tables(this)) return true;
  query_block->sj_pullout_done = true;

  // ② const 表：0/1 行的表现在就读出来，列替换成常量
  if (!(query_block->active_options() & OPTION_NO_CONST_TABLES)) {
    if (extract_const_tables()) return true;
    if (extract_func_dependent_tables()) return true;
  }

  // ② 首次行数估算：每表 fanout + range 分析（test_quick_select）
  if (estimate_rowcount()) return true;

  if (sj_nests) {
    set_semijoin_embedding();
    query_block->update_semijoin_strategies(thd);    // 五策略候选
  }

  // ③ 选 join 顺序：贪心 + 限深 DFS
  if (Optimize_table_order(thd, this, nullptr).choose_table_order()) return true;

  if (query_expression()->item && decide_subquery_strategy()) return true;
  refine_best_rowcount();

  // ③ 由最优顺序回填 JOIN_TAB
  if (get_best_combination()) return true;
  ...
}
```

> ⚠️ 该函数的 doxygen 写的是 `@return true if success`，**与实际相反**（true = 出错）。以调用点 `if (make_join_plan()) { ... return true; }` 为准。

`choose_table_order()` 是"贪心 or 直连"的分派点：

```cpp
// [O6] sql/sql_planner.cc
bool Optimize_table_order::choose_table_order() {
  if (join->const_tables == join->tables) { got_final_plan = true; return false; }

  const bool straight_join =
      join->query_block->active_options() & SELECT_STRAIGHT_JOIN;
  if (!straight_join)                                // 启发式预排序：按访问行数升序
    merge_sort(join->best_ref + join->const_tables,
               join->best_ref + join->tables, Join_tab_compare_default());

  if (straight_join)
    optimize_straight_join(join_tables);             // 不搜索，按书写顺序
  else {
    if (greedy_search(join_tables)) return true;     // ← 贪心 + 限深 DFS
  }
  got_final_plan = true;
  if (fix_semijoin_strategies()) return true;        // 定 semi-join 策略并算最终代价
  return false;
}
```

`greedy_search()` 的骨架——**每次只往"当前前缀"尾部追加一张表，这就是左深树的由来**：

```cpp
bool Optimize_table_order::greedy_search(table_map remaining_tables) {
  uint idx = join->const_tables;                     // 当前前缀长度
  do {
    join->best_read = DBL_MAX;                       // 全局上界 = 分支限界
    if (best_extension_by_limited_search(remaining_tables, idx, search_depth))
      return true;                                   // 限深 DFS，深到 search_depth 就回溯

    if (size_remain <= search_depth || use_best_so_far) return false;

    best_pos = join->best_positions[idx];            // 取最优扩展的第一张表，钉死
    join->positions[idx] = best_pos;
    /* ... 继续下一轮 ... */
  } while (/* 还有表没放进前缀 */);
}
```

**阶段 4：③ 收尾 + ④ 计划改进（旧路径专属）**

```cpp
  // ----------------------------------------------------------------------------
  //       All of this is never called for the hypergraph join optimizer!
  // ----------------------------------------------------------------------------
  assert(!thd->lex->using_hypergraph_optimizer());

  if (make_join_plan()) return true;                 // ← 阶段 2
  if (zero_result_cause != nullptr) { create_access_paths_for_zero_rows();
                                      goto setup_subq_exit; }

  // 【①】等价类展开成"星形等值"——依赖已定的 join 顺序
  if (where_cond) {
    where_cond = substitute_for_best_equal_field(thd, where_cond, cond_equal, map2table);
    where_cond->update_used_tables();
  }
  if (init_ref_access()) return true;                // ref 访问的 key 拷贝/绑定
  update_depend_map();

  // 读 const 表后再判一次：可能出现"读 const 表后 WHERE 不可能满足"
  if (make_join_query_block(this, where_cond)) {
    zero_result_cause = "Impossible WHERE noticed after reading const tables";
    create_access_paths_for_zero_rows();
    goto setup_subq_exit;
  }

  // ④ ORDER BY / DISTINCT / GROUP BY：能否用索引消掉排序
  if (optimize_distinct_group_order()) return true;

  // ④ join buffer 选择：BNL / BKA / 不用
  for (uint i = const_tables; i < tables; i++) {
    JOIN_TAB *const tab = best_ref[i];
    if (!tab->position()) continue;
    if (setup_join_buffering(tab, this, no_jbuf_after)) return true;
    if (tab->use_join_cache() != JOIN_CACHE::ALG_NONE) simple_sort = false;
  }
  if (!simple_sort) simple_order = simple_group = false;

  // ④ 是否需要临时表（ROLLUP / GROUP BY / DISTINCT / ORDER BY / 窗口）
  if (!plan_is_const()) {
    if ((!group_list.empty() && !simple_group) ||
        (!has_windows && (select_distinct ||
                          (!order.empty() && !simple_order) || ...)))
      need_tmp_before_win = true;
  }
```

**阶段 5：代码生成 —— QEP_TAB[] → AccessPath 树**

```cpp
  if (alloc_qep(tables)) return (error = 1);         // QEP_TAB[n+1]，末尾存 op_type
  if (!plan_is_const()) {
    test_skip_sort();                                // 索引能否直接给出 GROUP BY/ORDER BY 顺序
    if (finalize_table_conditions(thd)) return true; // 每表的最终条件 + 外连接守卫
  }
  if (make_join_readinfo(this, no_jbuf_after)) return true;   // 读方式 + ICP 下推
  if (make_tmp_tables_info()) return true;                    // 临时表 / filesort

  create_access_paths();                             // ← 代码生成 [O8]
  if (push_to_engines()) return true;
  set_plan_state(PLAN_READY);
  return false;

setup_subq_exit:                                     // 所有"零结果"出口的汇合点
  assert(zero_result_cause != nullptr);
  Opt_trace_object(trace, "empty_result").add_alnum("cause", zero_result_cause);
  if (!qep_tab && best_ref) { if (alloc_qep(tables)) return true; unplug_join_tabs(); }
  set_plan_state(ZERO_RESULT);
  return false;
}
```

```cpp
// [O8] —— 旧路径的"翻译层"
void JOIN::create_access_paths() {
  AccessPath *path = create_root_access_path_for_join();
  path = attach_access_paths_for_having_and_limit(path);
  path = attach_access_path_for_update_or_delete(path);
  m_root_access_path = path;
}
```

#### ★ `JOIN::optimize()` 完整流程图

```
                      JOIN::optimize(finalize_access_paths)              [O1]
                                     │
  ┌──────────────────────────────────▼───────────────────────────────────────────┐
  │ 阶段 0  预处理                                                                │
  │   count_field_types · setup_windows2 · optimize_rollup                        │
  │   get_optimizable_conditions（取 WHERE/HAVING 到 JOIN）                        │
  │   └─ optimize_derived()  ──递归──▶ 内层 Query_block::optimize()   【①递归】    │
  │   ⚡ 短路：Zero limit → create_access_paths_for_zero_rows() → setup_subq_exit   │
  └──────────────────────────────────┬───────────────────────────────────────────┘
                                     ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段 1  【① 逻辑优化 —— 纯规则，不比代价】                                     │
  │   optimize_cond(WHERE)   build_equal_items → propagate_cond_constants         │
  │                          → remove_eq_conds                                    │
  │   optimize_cond(HAVING)  （不做等值传播）                                     │
  │   prune_table_partitions() · optimize_aggregated_query() · substitute_gc()     │
  │   ⚡ 短路：Impossible WHERE/HAVING · Select tables optimized away              │
  └──────────────────────────────────┬───────────────────────────────────────────┘
                                     ▼
                    ┌──── thd->lex->using_hypergraph_optimizer() ? ────┐
                 no │                                                  │ yes
                    ▼                                                  ▼
  ┌────────────────────────────────────────┐   ┌──────────────────────────────────┐
  │ 阶段 2+4  【②③ 旧路径 greedy】          │   │ 阶段 3  【③ 新路径 hypergraph】     │
  │ JOIN::make_join_plan()                 │   │ remove_in2exists_conds()          │
  │  ② update_ref_and_keys → Key_use[]     │   │ FindBestQueryPlan()  [O3]         │
  │  ② extract_const_tables                │   │   MakeJoinHypergraph()            │
  │  ② estimate_rowcount()  ← range 分析   │   │   EnumerateAllConnectedPartitions │
  │  ③ choose_table_order()      [O6]      │   │     → CostingReceiver             │
  │      greedy_search()                   │   │       ProposeHashJoin/NL/BKA      │
  │  ③ get_best_combination() → JOIN_TAB[] │   │   + GROUP BY/HAVING/WINDOW/       │
  └───────────────┬────────────────────────┘   │     DISTINCT/ORDER BY/LIMIT      │
                  ▼                            │   → root_candidates 取 min(cost)  │
  ┌────────────────────────────────────────┐   │ FinalizePlanForQueryBlock()       │
  │ 阶段 4  【① 收尾改写 + ④ 计划改进】       │   └──────────────────────────────────┘
  │   substitute_for_best_equal_field      │
  │   init_ref_access · make_join_query_block│
  │   optimize_distinct_group_order        │
  │   setup_join_buffering · alloc_qep     │
  │   make_join_readinfo · make_tmp_tables_info│
  └───────────────┬────────────────────────┘
                  ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段 5  代码生成                                                              │
  │   create_access_paths()  [O8]  QEP_TAB[] ──ConnectJoins()──▶ AccessPath 树     │
  │   push_to_engines()  ·  set_plan_state(PLAN_READY)                            │
  └──────────────────────────────────┬───────────────────────────────────────────┘
                                     ▼
                         JOIN::m_root_access_path   ← 两条路径的汇合点
```

**逻辑/物理的落点一句话总结**：**阶段 1 与阶段 4 的前两行是逻辑优化（规则、保语义）**；**阶段 2 的 `estimate_rowcount` / `choose_table_order` 与阶段 3 全部是物理优化（代价、不保等价）**；**阶段 5 是代码生成**——这正是本篇"先规则、后代价"在源码里的物理位置。
#### `Query_block::prepare()` 骨架

> 源码坐标：`[O2] sql/sql_resolver.cc`

这一步做什么：**名字解析 + 结构性逻辑改写**（视图/派生表展开与合并、子查询展平、连接简化）。这些改写**改变的是查询块的结构本身**，因此必须最先做完。

```cpp
// [O2]
bool Query_block::prepare(THD *thd, mem_root_deque<Item *> *insert_field_list) {
  /* ---------- 名字解析（R：Resolver） ---------- */
  if (setup_tables(thd, get_table_list(), false)) return true;

  // ① 视图 / 派生表 / 表函数：展开 → 尝试合并 → 否则准备物化
  if ((derived_table_count || table_func_count) &&
      resolve_placeholder_tables(thd, true)) return true;

  if (setup_fields(thd, thd->want_privilege, /*allow_sum_func=*/true,
                   /*split_sum_funcs=*/true, ...)) return true;
  if (setup_conds(thd)) return true;                   // WHERE / ON 的 fix_fields
  if (group_list.elements && setup_group(thd)) return true;

  /* ---------- ① 结构性改写 ---------- */
  // ★ 子查询展平：IN/ANY/ALL/EXISTS → semi-join 或派生表
  if (has_sj_candidates() && flatten_subqueries(thd)) return true;
  set_sj_candidates(nullptr);

  // ① 局部变换（只由最外层 query block 发起，向下递归）
  if (/* 最外层条件 */ && !skip_local_transforms) {
    if (apply_local_transforms(thd, true)) return true;
  }
  return false;
}
```

**改写点 1：`resolve_placeholder_tables()` —— 视图/派生表展开与合并**

```cpp
for (Table_ref *tl = get_table_list(); tl; tl = tl->next_local) {
  if (!tl->is_view_or_derived() && !tl->is_table_function()) continue;
  if (tl->resolve_derived(thd, apply_semijoin)) return true;   // ① 递归 prepare 内层

  // ★ 合并：把派生表/视图的查询块"摊平"进外层（derived_merge 开关控制）
  if (tl->is_mergeable() && merge_derived(thd, tl)) return true;
  if (tl->is_merged()) continue;

  // 没合并掉的：准备物化
  if (tl->table == nullptr && tl->setup_materialized_derived(thd)) return true;
  materialized_derived_table_count++;
}
```

**改写点 2：`apply_local_transforms()` —— 连接简化 + 分区裁剪 + 条件下推**

```cpp
bool Query_block::apply_local_transforms(THD *thd, bool prune) {
  assert(first_execution);                             // ★ 只做一次（PS 首次 prepare）

  for (Query_expression *unit = first_inner_query_expression(); unit;
       unit = unit->next_query_expression())
    for (auto qt : unit->query_terms<>())              // 先递归到所有内层
      if (qt->query_block()->apply_local_transforms(thd, true)) return true;

  // ★ 外连接 → 内连接（WHERE 里有拒 NULL 谓词时合法）
  if (simplify_joins(thd, &m_table_nest, true, false, &m_where_cond)) return true;
  if (record_join_nest_info(&m_table_nest)) return true;
  build_bitmap_for_nested_joins(&m_table_nest, 0);

  // ONLY_FULL_GROUP_BY 检查必须放在 simplify_joins 之后（要用到 nest 的 used_tables）
  if ((is_distinct() || is_grouped()) &&
      (thd->variables.sql_mode & MODE_ONLY_FULL_GROUP_BY) &&
      check_only_full_group_by(thd)) return true;

  // ★ 分区裁剪（常量条件）—— 放在改写之后最有效
  if (partitioned_table_count && prune) {
    for (Table_ref *tbl = leaf_tables; tbl; tbl = tbl->next_leaf) {
      if (prune_partitions(thd, tbl->table, this,
                           tbl->join_cond() ? tbl->join_cond() : m_where_cond))
        return true;
      if (tbl->table->all_partitions_pruned_away &&
          !tbl->is_inner_table_of_outer_join())
        set_empty_query();
    }
  }

  // ★ 条件下推到物化派生表（自顶向下，可多级下推）
  if (outer_query_block() == nullptr && push_conditions_to_derived_tables(thd))
    return true;
  return false;
}
```

> ⚠️ **一处需要修正的常见说法**：**等值传播（`build_equal_items`）不在 prepare 期**。它唯一的 WHERE 调用点在 `optimize_cond()`（属 `JOIN::optimize()`）。prepare 期做的是**结构类**改写（合并/展平/简化/裁剪/下推），optimize 期做的是**谓词类**改写（等值传播/常量传播/去恒真恒假）——因为后者依赖的常量要等 const 表读出、依赖的等价类要等 join 顺序定下来。

#### 为什么这些必须在 `JOIN::optimize()` 之前

1. **物理优化的输入是"表集合 + 谓词集合"**。派生表合并会**改变 leaf table 的数量与身份**，semi-join 展平会**凭空多出 `sj_nests` 与 5 种候选策略**。若先跑 `make_join_plan()` 再去合并，搜索空间的对象集合本身就是错的。
2. **改写是单向、不可逆的**：`merge_derived()` 后原 `Table_ref` 被标记 `is_merged()`，`flatten_subqueries()` 后 `set_sj_candidates(nullptr)`。源码甚至用 `assert(first_execution)` 把 `apply_local_transforms()` 锁死为"只做一次"。
3. **简化能缩小搜索空间**：`simplify_joins()` 转内连接后 `propagate_dependencies()` 的依赖约束变少；`prune_partitions()` 让 `estimate_rowcount()` 拿到更小的 `stats.records`。**先剪枝再搜索**比"搜索时再判断合法性"便宜得多。
4. **超图把合法性编译进图**：hypergraph 的 `MakeJoinHypergraph()` 直接读简化后的 `m_table_nest` 与 `sj_nests` 建超边。若 prepare 没把外连接语义固化，超图无从编码"哪些重排合法"——这正是 DPhyp 相对旧优化器"在搜索循环里反复检查"的关键差别。

```
Query_block::prepare()                                    [O2]  sql/sql_resolver.cc
│
├─ setup_tables / setup_fields / setup_conds / setup_group / setup_order   （名字解析）
│
├─ resolve_placeholder_tables()          ← 视图 / 派生表 / 表函数
│     ├─ tl->resolve_derived()           ← 递归 prepare 内层 query block
│     ├─ merge_derived()                 ← 【逻辑】合并进外层（derived_merge）
│     └─ setup_materialized_derived()    ← 否则准备物化
│
├─ flatten_subqueries()                  ← 【逻辑】IN/ANY/ALL/EXISTS → semi-join / 派生表
│
└─ apply_local_transforms()              ← assert(first_execution)：只做一次
      ├─ 递归内层 query block
      ├─ simplify_joins()                ← 【逻辑】外连接 → 内连接
      ├─ prune_partitions()              ← 【逻辑】分区裁剪（常量条件）
      └─ push_conditions_to_derived_tables()  ← 【逻辑】条件下推
                                    │
                                    ▼
                        JOIN::optimize()（此时逻辑形态已定）
```
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


### 示例 SQL

```sql
SELECT o.cust_id, SUM(l.amount) AS total
FROM   orders   o
JOIN   lineitem l ON l.order_id = o.id
WHERE  o.status = 'PAID'
  AND  l.amount > 100
GROUP  BY o.cust_id
ORDER  BY total DESC
LIMIT  10;
```

假设：`orders.id` 主键、`lineitem.order_id` 上有索引、`orders.status` 上有索引。

### 1) prepare 阶段发生了什么

| # | 改写 | 函数 | 结果 |
|---|---|---|---|
| 1 | 无派生表/视图/子查询 → 本例**不做**合并与展平 | `resolve_placeholder_tables` / `flatten_subqueries` | 只有两张 leaf table |
| 2 | 名字解析 | `setup_tables` → `setup_fields` → `setup_conds` → `setup_group` → `setup_order` | `SUM(l.amount)` 被 `split_sum_func` 拆出隐藏 item；`ORDER BY total` 解析到该隐藏 item |
| 3 | 外连接简化 | `simplify_joins` | 本例是 INNER JOIN，无转换；但 `m_table_nest` 在此填好 |
| 4 | ONLY_FULL_GROUP_BY | `check_only_full_group_by` | `o.cust_id` 在 GROUP BY 中，合法 |
| 5 | 分区裁剪 / 条件下推 | `prune_partitions` / `push_conditions_to_derived_tables` | 非分区表、无物化派生表，均跳过 |

⇒ 本例 prepare 期几乎没有"结构性"改写——这恰好说明 prepare 的改写是**查询形态驱动**的，不是每条 SQL 都会触发。

### 2) `JOIN::optimize()` 各阶段的关键产物

```cpp
/* 阶段 0 */
get_optimizable_conditions(&where_cond, &having_cond)
   // where_cond = o.status='PAID' AND l.amount>100 AND l.order_id=o.id
/* 阶段 1 【逻辑】 */
optimize_cond(&where_cond, ...)
   ├ build_equal_items        → 等值传播：MEP {l.order_id, o.id}（连接条件）
   ├ propagate_cond_constants → 'PAID' / 100 已是常量，无传播
   └ remove_eq_conds          → 无恒真恒假，cond_value = COND_OK
/* 阶段 2 【②③ 旧路径】 */
make_join_plan()
   ├ update_ref_and_keys      → Key_use[]: o.status / l.order_id / l.amount
   ├ extract_const_tables     → 无表是 0/1 行 → const_tables = 0
   ├ estimate_rowcount        → orders≈12 万（status='PAID' 后≈3 万），lineitem≈200 万
   │                            对两个谓词各做一次 range 分析
   ├ choose_table_order       → greedy_search：前缀 [orders] then [orders, lineitem]
   └ get_best_combination     → JOIN_TAB[0]=orders(range on status), JOIN_TAB[1]=lineitem(ref)
/* 阶段 4 【④】 */
substitute_for_best_equal_field(...)     // 等价类选 o.id 作为先取字段
optimize_distinct_group_order()          // GROUP BY 无法用索引序 → simple_group=false
                                         // ORDER BY total 是聚合 → simple_order=false
setup_join_buffering(...)                // lineitem 走 ref；可用 hash join 则替换
need_tmp_before_win = true               // 需要 GROUP BY 临时表
finalize_table_conditions(...)           // ICP：l.amount>100 下推到 lineitem 的索引扫描
/* 阶段 5 */
create_access_paths()                    // QEP_TAB[] → ConnectJoins() → AccessPath 树
```

optimizer trace 的骨架正是这些字符串（均为源码中的字面量）：

```json
"join_optimization": {
  "steps": [
    { "condition_processing": { "condition": "WHERE",
        "steps": [ { "transformation": "equality_propagation"   },
                   { "transformation": "constant_propagation"   },
                   { "transformation": "trivial_condition_removal" } ] } },
    { "rows_estimation": [                        // ← 阶段 2 的 estimate_rowcount
        { "table": "`orders`",   "range_analysis": { "table_scan": {...}, ... } },
        { "table": "`lineitem`", "range_analysis": { ... } } ] },
    { "considered_execution_plans": [             // ← greedy_search（flag = GREEDY_SEARCH）
        { "plan_prefix": ["`orders`"],            // ← 每次只往尾部加一张表 = 左深
          "best_access_path": { "considered_access_paths": [...] } },
        { "plan_prefix": ["`orders`","`lineitem`"], ... } ] },
    { "optimizing_distinct_group_by_order_by": {...} },   // ← 阶段 4
    { "finalizing_table_conditions": [...] },
    { "refine_plan": [...] }
  ]
}
```

开启 hypergraph 时阶段 2/4 被整段替换，trace 变成（字符串同样取自 `[O3]`）：

```
join_optimization → steps → ... →
  "join_optimizer": [
    "Enumerating subplans:",
    "Found sets {orders} and {lineitem}, connected by condition (l.order_id = o.id)",
    "Applying aggregation for GROUP BY",
    "Applying LIMIT",
    "Final cost is 12345.6."
  ]
```

### 3) 两条路径产出的 AccessPath 树形状

**旧路径**（`create_access_paths()` 事后翻译，左深）：

```
LIMIT_OFFSET (10)                       // attach_access_paths_for_having_and_limit
└─ SORT (filesort, ORDER BY total DESC) // make_tmp_tables_info 建的 filesort
   └─ MATERIALIZE / TEMPTABLE_AGGREGATE // GROUP BY 临时表（simple_group=false ⇒ 要物化）
      └─ NESTED_LOOP_JOIN (INNER)
         ├─ outer: INDEX_RANGE_SCAN orders (status='PAID')
         └─ inner: REF lineitem (order_id = o.id)   ← ICP 把 amount>100 压在索引扫描上
```

**hypergraph**（`FindBestQueryPlan()` 原生产出，可 bushy）：

```
LIMIT_OFFSET (10)                       // ApplyDistinctAndOrder 里与 ORDER BY 合并处理
└─ SORT (ORDER BY total DESC)           // ApplyDistinctAndOrder
   └─ AGGREGATE (streaming, GROUP BY)   // CreateStreamingAggregationPath
      └─ HASH_JOIN (INNER, build=orders, probe=lineitem)   // ProposeHashJoin
         ├─ build: INDEX_RANGE_SCAN orders (status='PAID')
         └─ probe: INDEX_RANGE_SCAN lineitem (amount > 100)
```

★ **两条的关键差别**：旧路径的 `GROUP BY` 必须落进临时表（`TEMPTABLE_AGGREGATE`/`MATERIALIZE`），因为左深树的输出顺序无法保证按 `o.cust_id` 有序；hypergraph 在枚举时就知道每个候选的 `ordering_state`，会主动插入"sort-ahead"（先按 `cust_id` 排序再流式聚合），或干脆挑一个已有序的候选，从而**省掉一次物化**。

⇒ 这正是"DP 状态带 interesting order + bushy 可达"带来的实际收益。细节见 [`physical/09_hypergraph.md`](physical/09_hypergraph.md)。
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

