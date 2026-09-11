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
└── 13_functional_mv_index.md  横切：函数索引 + 多值索引
```

> 编号**全局连续**（02-13），不因分目录而重置——这样各篇之间"见 03 篇"这类交叉引用始终有效。

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
| [00_overview.md](00_overview.md) | 总览 | 四阶段框架、**两条优化路径对比**（greedy vs hypergraph）、`JOIN::optimize` 全流程映射表、各篇导航与推荐阅读顺序、贯穿式例子 |
| [01_cost_model.md](01_cost_model.md) | ②③ 前提 | 代价模型（**total_cost 不含 mem_cost**）+ 统计信息：InnoDB 分层采样与 NDV 外推公式、rec_per_key（legacy 整数版除 2 的历史遗留）、index dive 三阶段、直方图 GEE 与桶内插值、条件过滤系数表、15 项兜底估算 |

### ① 逻辑优化（规则驱动，等价变换）

| 文件 | 核心内容 |
|------|----------|
| [02_subquery.md](logical/02_subquery.md) | **子查询改写**：Subquery_strategy 状态机（8 态 + 两阶段决策）、resolve_subquery 半连接 16 条件、IN2EXISTS 三分支（HAVING/WHERE/常量）、物化代价公式 `subq_executions` 累乘、anti-join 的 value_transform 四态 + NULL 三值陷阱、MIN/MAX 方向反转（`l_op()` + ALL invert）、标量→derived + 相关谓词提 GROUP BY、flatten_subqueries 四步 + 恒假删除 |
| [03_semijoin.md](logical/03_semijoin.md) | **semi-join 独立成章**：语义、Notation(ot/ct/nt/it)、五策略详解、改写/优化/执行三阶段、QEP_TAB 序列、JOIN ORDER 矩阵 |
| [04_logical_join.md](logical/04_logical_join.md) | **连接简化**：simplify_joins 两次递归（含位图清零陷阱）、外连接转内连接判定 + 形式化证明、not_null_tables 三条规则与 8 条判例、Pass 2 扁平化充要条件（SJ 可溶/AJ 不可溶）、merge_derived 12 步（**merge_where 合进 ON 的关键设计**）、递归 CTE 不 merge 三重原因 |
| [05_logical_predicate.md](logical/05_logical_predicate.md) | **谓词优化**：Item_equal 等价类四 case 合并 + 跨层写时复制 copyfl、**⚠8.0 无通用算术折叠器**（`1+2` 不变 `3`）、`resolve_const_item` 是唯一物化点、常量传播两轮、make_cond_for_table 的 AND 可拆/OR 全或无、外连接 trig_cond 两种守卫、ICP 8 项判定 + 零代价影响 |

### ②③ 物理优化（代价驱动）

> 这是优化器最"重"的部分，拆成四篇：搜索顺序、访问方法、range 专线、hypergraph。

| 文件 | 核心内容 |
|------|----------|
| [06_join_order.md](physical/06_join_order.md) | **表连接顺序**：`choose_table_order` 入口（`best_ref` 预排序 + straight 分流）、`best_extension` 主循环（`best_ref[]` 遍历 + swap）、`set_prefix_join_cost` 公式、三种剪枝精确条件（cost 的 `found_plan_with_allowed_sj` / heuristic 的 Pareto 基准 / eq_ref 的 `almost_equal` 10%）、POSITION 全字段（POD 约束）、condition filtering 闭环、hypergraph 分流 |
| [07_access_method.md](physical/07_access_method.md) | **访问方法选择**：const 表检测（不动点循环+唯一键）、Key_use 数组（笛卡尔展开+多等值 O(n²)）、find_best_ref（fanout 三级来源）、ref/range/scan 四条启发式短路 + 代价决斗、二次 range 重估 |
| [08_range_optimizer.md](physical/08_range_optimizer.md) | **range 优化独立成篇**：SEL_TREE/SEL_ARG 区间森林（红黑树+链表+next_key_part 三重结构）、key_and/key_or、六种 range 访问（range/skip scan/MIN-MAX/ROR-intersection/union/index merge）、**index merge 代价估算**（选择率乘数 `n_k/n_{k-1}` 连乘、贪心搜索、去重率、非相关性假设的 bad case 根因）、index dive vs 统计、**长 IN list 字面量专题**（DNF 展开 O(N log N)、`in_vector` 排序不显式去重、dive limit 边界、无 inlist2join、8MB 内存闸降级）、QUICK_RANGE→执行 |
| [09_hypergraph.md](physical/09_hypergraph.md) | **hypergraph**：JoinHypergraph 构建（**CD-C 算法 + TES 膨胀**）、DPhyp 五函数（Solve/EmitCsg/EnumerateCsgRec/EnumerateCmpRec/TryConnecting）、CostingReceiver 的 Pareto 前沿、图简化重跑（打破左深树，支持 bushy tree） |

### ④ 计划改进

| 文件 | 核心内容 |
|------|----------|
| [10_plan_refinement.md](10_plan_refinement.md) | Ordering index 选择（**test_if_order_by_key 返回方向非 keypart 数**、select_limit 三重缩放、多表反向保护）、finalize_table_conditions 删冗余谓词、make_join_readinfo 分派、**两层下推的区别**、join buffering、临时表七阶段 |

### 横切（不属四阶段任何单一阶段）

| 文件 | 核心内容 |
|------|----------|
| [11_optimizer_hints.md](11_optimizer_hints.md) | **Optimizer Hints**：双语法器（`sql_hints.yy`）+ 四层对象树（global/qb/table/key）、三类映射（开关/枚举/复杂）、JOIN_ORDER 转 `dependent` 链式依赖、INDEX 转索引位图集合运算、SET_VAR/MAX_EXECUTION_TIME、8.0 已废弃的 hint |
| [12_partition_pruning.md](12_partition_pruning.md) | **分区裁剪**：归约到 range 分析（复用 `get_mm_tree`→SEL_TREE）、`find_used_partitions` 遍历 SEL_ARG 树、RANGE/LIST 二分 vs HASH/KEY 单点精确·范围枚举、子分区递归、read/lock_partitions 位图→handler→EXPLAIN |
| [13_functional_mv_index.md](13_functional_mv_index.md) | **函数索引 + 多值索引**：隐藏生成列（`HT_HIDDEN_SQL`）、GC substitution（`get_gc_for_expr` 表达式等价匹配）、MEMBER OF 转等值 range、多值索引的唯一记录过滤器去重 |

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
