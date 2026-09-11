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

### 优化器的理论谱系：System R → Cascades → hypergraph

关系型查询优化器五十年的演进，有两条主线，MySQL 恰好两条都沾：

**1. System R 派：动态规划 + 代价模型（1979）**

Selinger et al. 的《Access Path Selection in a Relational DBMS》奠定了现代优化器的基石：

- **左深树 + 自底向上动态规划**：只考虑"连接顺序"，按表数递增枚举，复用子问题最优解
- **代价模型**：用统计信息（基数、选择性）估算每个候选计划的 IO + CPU 代价
- **选择性估算**：假设各条件独立，`结果行数 = 输入行数 × 各过滤系数乘积`

MySQL 的**旧优化器（greedy）就是 System R 的简化版**——代价模型直接沿用（`01_cost_model.md`），但把穷举 DP 换成带剪枝的贪心搜索（`physical/06_join_order.md`），牺牲最优性换速度。

**2. Cascades 派：规则 + 代价（1995）**

Graefe 的《The Cascades Framework for Query Optimization》提出更现代的架构：逻辑算子（logical algebra）与物理算子（physical algebra）分离，用**规则**做等价变换、用**代价**做选择。

MySQL 8.0 的 **hypergraph 优化器**（`physical/09_hypergraph.md`）在思想上更接近这一派——虽然它没完整实现 Cascades 的规则引擎，但吸收了"逻辑枚举与代价驱动分离"、以及 DPhyp / CD-C 这些更先进的 join 枚举算法。

### 为什么 MySQL 有两条优化路径

这是 MySQL 最大的历史包袱，也是最值得理解的一点：

| | 旧优化器（greedy） | 新优化器（hypergraph） |
|---|---|---|
| 起源 | 5.x 一路演进，源自 System R 左深树 | 8.0.20+ 引入，源自 DPhyp 论文 |
| join 形状 | **只支持左深树** | 支持 bushy tree（任意形状） |
| 算法 | 贪心 + 剪枝 | `EnumerateAllConnectedPartitions`（DPhyp） |
| 默认 | 默认开启 | `optimizer_switch=hypergraph_optimizer` 开启 |

**为什么不能直接替换**：旧优化器经过几十年打补丁，承载了无数特殊 case（semi-join 策略、子查询物化决策、hint 语义），行为已被用户和测试依赖。hypergraph 是"另起炉灶"的新实现，需要长周期灰度切换——所以 8.0 是**两条并行**，靠 AccessPath（08 篇）这个统一 IR 汇合。

### 逻辑优化 vs 物理优化的分野

四阶段框架（下面「一」节）的骨架，本质是 **Cascades 思想的简化落地**：

- **逻辑优化** = 逻辑等价变换（规则），不涉及具体执行方式
- **物理优化** = 在等价逻辑表达式里，用代价挑具体实现（访问方法、join 顺序、join 方式）

"先规则、后代价"——规则负责缩小空间，代价负责挑出最优。这是本目录 11 篇组织逻辑的根源。

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
| 502-506 | `prune_table_partitions` 分区裁剪 | ① | |
| 514-571 | `optimize_aggregated_query` | ① | |
| 600-606 | `substitute_gc`（生成列替换） | ① | |
| **610-679** | **hypergraph 分支**（若开启则到此 return） | ③ | [09 篇](physical/09_hypergraph.md) |
| **696** | **`make_join_plan()`** | **②③** | [06](physical/06_join_order.md) [07](physical/07_access_method.md) |
| 736-748 | `substitute_for_best_equal_field`（等价类展开成星形等值） | ①（需 join 顺序先定） | [05 篇 1.5](logical/05_logical_predicate.md) |
| 770-781 | `init_ref_access` / `make_join_query_block` | ③ 收尾 | [05 篇 3 节](logical/05_logical_predicate.md) |
| 796 | `optimize_distinct_group_order` | ① | |
| 894-916 | `setup_join_buffering`（BNL/BKA/HashJoin） | ③ | [10 篇](10_plan_refinement.md) |
| **1011-1023** | `alloc_qep` / `test_skip_sort` / `finalize_table_conditions` / `make_join_readinfo` / `make_tmp_tables_info` | **④ 计划改进** | [10 篇](10_plan_refinement.md) |
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

```
optimizer/
├── 00_overview.md        本篇
├── 01_cost_model.md      代价模型 + 统计信息（②③ 的前提）
├── logical/              ① 逻辑优化
│   ├── 02_subquery.md          子查询改写决策树
│   ├── 03_semijoin.md          semi-join（五策略、改写/优化/执行）
│   ├── 04_logical_join.md      连接简化（外连接转内、derived/view/CTE merge）
│   └── 05_logical_predicate.md 谓词优化（等值传播、常量折叠、下推、ICP）
├── physical/             ②③ 物理优化
│   ├── 06_join_order.md        join order 搜索（greedy + 限深 DFS + 剪枝）
│   ├── 07_access_method.md     访问方法选择（const 表 / ref / range / scan）
│   ├── 08_range_optimizer.md   range 优化（SEL_TREE 区间森林）
│   └── 09_hypergraph.md        hypergraph（DPhyp）
└── 10_plan_refinement.md ④ 计划改进
```

**推荐阅读顺序**：

1. **入门**：00 → 01 → 06 → 07（主链：框架 → 代价 → join order → 访问方法）
2. **深入物理优化**：08（range 是最复杂的访问方法）
3. **深入逻辑优化**：03（semi-join）→ 04（连接简化）→ 05（谓词）
4. **新特性**：09（hypergraph）

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

