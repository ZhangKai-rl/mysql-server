# MySQL 优化器（Optimizer）

> 优化器是查询处理中最大的子系统之一，独立成此目录（作为 query 主链第 5 步）。本目录按**逻辑优化 → 物理优化 → 计划改进**三层组织，全部算法级。
>
> **上下游**：`../06_resolver_prepare.md`（②→③）→ **本目录**（③→④，产出 AccessPath）→ `../08_access_path.md`（④ AccessPath 树）→ `../09_executor_iterator.md`（⑤ 迭代器）。执行期专题见 `../runtime/`。

## 目录结构

```
optimizer/
├── README.md              本文件（导航，按阶段分组）
├── 00_overview.md         总览：四阶段框架、JOIN::optimize 全流程
├── 01_cost_model.md       代价模型 + 统计信息（②③ 公共基础）
├── logical/               ① 逻辑优化（规则驱动，等价变换）
│   ├── 02_subquery.md
│   ├── 03_semijoin.md
│   ├── 04_logical_join.md
│   └── 05_logical_predicate.md
├── physical/              ②③ 物理优化（代价驱动）
│   ├── 06_join_order.md
│   ├── 07_access_method.md
│   ├── 08_range_optimizer.md
│   └── 09_hypergraph.md
├── 10_plan_refinement.md  ④ 计划改进（单篇，留在根）
├── 11_optimizer_hints.md  横切：`/*+ */` 干预优化器
├── 12_partition_pruning.md  横切：分区裁剪（归约到 range 分析）
├── 13_functional_mv_index.md  横切：函数索引 + 多值索引
├── 14_plan_stability.md   横切：计划稳定性与优化器能力横向对照
├── 15_groupby_distinct_order.md  专项：聚合/分组/排序的优化期决策（①前后）
├── 16_join_object_model.md       专项：优化器对象模型与生命周期（对象视角）
├── 17_optimizer_decisions.md     横切：优化器决策全景（36 个决策点索引）
├── physical/18_hypergraph_advanced.md  ② 续：CSE / 谓词归位 / 计划最终化 / 新代价模型 / 二级引擎
├── 19_set_operation.md          专项：set operation 的优化侧（UNION/INTERSECT/EXCEPT）
├── 20_view_resolution.md        专项：VIEW 的解析与优化（merge vs materialize）
├── 21_collation_index_usability.md  横切：collation/类型聚合与索引可用性
├── 22_optimizer_worklog_timeline.md  横切：优化器 worklog 与版本演进时间线
└── 23_optimizer_trace_internals.md   横切：optimizer_trace 的实现原理
```

> 编号**全局连续**（02-23），不因分目录而重置——这样各篇之间"见 03 篇"这类交叉引用始终有效。
>
> 15 之后的分组：**专项**（针对某一个具体机制深挖）、**横切**（跨阶段，用于索引与排查）、**② 续**（hypergraph 高阶，是 09 的延续故留在 physical/）。

## 为什么独立成子目录

优化器不是一个函数，而是一条**多阶段流水线**（对应内核月报 2024/04 的框架）：

```
SQL 逻辑查询树（prepare 之后）
        │
        ▼ ① 逻辑优化（规则驱动，等价变换）
        │    子查询改写 / 外连接消除 / 等值传播 / 常量折叠 / 谓词下推 / 视图合并
        ▼ ② 初始优化分析
        │    各表访问路径的扫描行数+代价、Const Table 检测
        ▼ ③ 物理优化（代价驱动）
        │    访问路径/访问方式、Join Order、Join 方式（NL/HashJoin）选择
        ▼ ④ 计划改进（Plan Refinement）
        │    Ordering index、ICP 下推引擎、条件最终化
        ▼ AccessPath 树 → RowIterator 树
```

> 之前把优化器混在 `query/` 平层的 `07_optimizer_classic.md`/`07_optimizer_hypergraph.md` 里，只讲了 ②③，丢了 ①④ 和统计信息。本目录补齐。

## 文件索引

### 总览与代价基础

| 文件 | 阶段 | 核心内容 |
|------|------|----------|
| [00_overview.md](00_overview.md) | 总览 | 四阶段框架、两条优化路径对比、全流程映射 |
| [01_cost_model.md](01_cost_model.md) | ②③ 前提 | 代价模型 + 统计信息（采样、NDV 外推、直方图、过滤系数） |

### ① 逻辑优化（规则驱动，等价变换）

| 文件 | 核心内容 |
|------|----------|
| [02_subquery.md](logical/02_subquery.md) | 子查询改写：策略状态机、IN2EXISTS、anti-join、标量→derived |
| [03_semijoin.md](logical/03_semijoin.md) | semi-join 独立成章：五策略、改写/优化/执行三阶段 |
| [04_logical_join.md](logical/04_logical_join.md) | 连接简化：simplify_joins、外连接转内连接、merge_derived |
| [05_logical_predicate.md](logical/05_logical_predicate.md) | 谓词优化：等价类、常量传播、谓词下推、ICP 判定 |

### ②③ 物理优化（代价驱动）

> 这是优化器最"重"的部分，拆成四篇：搜索顺序、访问方法、range 专线、hypergraph。

| 文件 | 核心内容 |
|------|----------|
| [06_join_order.md](physical/06_join_order.md) | 表连接顺序：choose_table_order、best_extension、剪枝 |
| [07_access_method.md](physical/07_access_method.md) | **访问方法选择**（1058 行）：const 检测、Key_use、`find_best_ref` 代价决斗、★**`Index_lookup` 键值绑定（第八章）**——四个互不相干的 `ref` 术语澄清、`store_key` 与**常量固化**（`key_copy[i]==NULL` ⇒ 执行期跳过，EXPLAIN `ref` 列显示 `const`）、★`cond_guards`（NOT IN 的"可开关条件"，失效时由 `AlternativeIterator` **退化成全表扫描**，Extra 显示 `Full scan on NULL key`）、REF_OR_NULL 的**两阶段查找**（`null_ref_key` 单字节 ⇒ 最多一个 ref_or_null keypart）、`keypart_hash`（8.0，物化临时表 hash 去重，与 hash join 无关）、★ICP 导致同 key 结果可变 ⇒ 禁用 eq_ref 行缓存、EXPLAIN `key_len`/`ref` 两列的来源 |
| [08_range_optimizer.md](physical/08_range_optimizer.md) | range 优化：SEL_TREE/SEL_ARG、六种 range 访问、index merge 代价 |
| [09_hypergraph.md](physical/09_hypergraph.md) | hypergraph：JoinHypergraph 构建、DPhyp、Pareto 前沿 |
| [18_hypergraph_advanced.md](physical/18_hypergraph_advanced.md) | hypergraph 高阶：CSE、谓词归位、计划最终化、二级引擎 |

### ④ 计划改进

| 文件 | 核心内容 |
|------|----------|
| [10_plan_refinement.md](10_plan_refinement.md) | Ordering index、条件最终化、join buffering、临时表 |
| [21_collation_index_usability.md](21_collation_index_usability.md) | collation / 类型聚合与索引可用性 |
| [23_optimizer_trace_internals.md](23_optimizer_trace_internals.md) | optimizer_trace 实现原理 |
| [22_optimizer_worklog_timeline.md](22_optimizer_worklog_timeline.md) | Worklog 与版本演进时间线 |
| [19_set_operation.md](19_set_operation.md) | set operation 的优化侧 |
| [20_view_resolution.md](20_view_resolution.md) | VIEW 的解析与优化（merge vs materialize） |
| [17_optimizer_decisions.md](17_optimizer_decisions.md) | 优化器决策全景（36 个决策点横向索引） |
| [16_join_object_model.md](16_join_object_model.md) | 优化器对象模型与生命周期（对象视角） |
| [15_groupby_distinct_order.md](15_groupby_distinct_order.md) | GROUP BY / DISTINCT / ORDER BY 的优化期决策 |

### 横切（不属四阶段任何单一阶段）

| 文件 | 核心内容 |
|------|----------|
| [11_optimizer_hints.md](11_optimizer_hints.md) | Optimizer Hints：双语法器、四层对象树 |
| [12_partition_pruning.md](12_partition_pruning.md) | 分区裁剪：归约到 range 分析 |
| [13_functional_mv_index.md](13_functional_mv_index.md) | 函数索引 + 多值索引 |
| [14_plan_stability.md](14_plan_stability.md) | 计划稳定性与优化器能力横向对照 |

## 逻辑优化 vs 物理优化（一句话界定）

| | 逻辑优化 | 物理优化 |
|---|---------|---------|
| 驱动 | **规则**（等价变换，保语义） | **代价**（Cost-based） |
| 做什么 | 把查询改写成"更利于优化"的等价形式 | 在等价形式里选"代价最小"的执行方式 |
| 典型动作 | semi-join 拍平、外连接转内连接、等值传播、谓词下推 | join order 搜索、访问方法选择、join 算法选择 |
| MySQL 里 | prepare 阶段改写 + `optimize_cond`/`simplify_joins` | `make_join_plan`（greedy/DPhyp + cost model） |

## 参考（内核月报）

> 各篇末尾的「参考」章节列出该篇的论文 / 官方文档 / 月报出处。

- 2024/04《MySQL 查询优化分析 - 基础概念》（四阶段框架）
- 2024/06《连接消除》（外连接/内连接/半连接消除原理）
- 2024/06《Semijoin 丛林小道全览》（五种策略 + JOIN ORDER 矩阵）
- 2022/10《统计信息采集》、2021/06《Semi-join 优化与执行逻辑》、2021/06《Range (Min-Max Tree) 结构分析》
