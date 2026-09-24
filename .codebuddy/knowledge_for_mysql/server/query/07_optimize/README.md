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
| [18_hypergraph_advanced.md](physical/18_hypergraph_advanced.md) | **hypergraph 高阶模块**：**CSE 公共子表达式消除**（`(a AND b) OR (a AND c)` → `a AND (b OR c)`，★动机是"让谓词可独立下推"而非提速；`AlwaysPresent` 三条规则、`OrGroupWithSomeRemoved` 的恒真传播、无代价阈值、只在 hypergraph 生效）、**谓词位图→Item 树的归位**（`ApplyPredicatesForBaseTable` / `ApplyDelayedPredicatesAfterJoin` 多等值只应用一次 / `ExpandFilterAccessPaths`）、`FinalizePlanForQueryBlock`（**只能调一次**）、**新代价模型**（`constexpr` 全是 0.1 vs 旧的可配置 `server_cost` 表）、`replace_item`、★ **二级引擎**（`SecondaryEngineFlag` bitmask 免遍历、`IteratorsAreNeeded` 三条规则、社区无可用引擎） |

### ④ 计划改进

| 文件 | 核心内容 |
|------|----------|
| [10_plan_refinement.md](10_plan_refinement.md) | Ordering index 选择（**test_if_order_by_key 返回方向非 keypart 数**、select_limit 三重缩放、多表反向保护）、finalize_table_conditions 删冗余谓词、make_join_readinfo 分派、**两层下推的区别**、join buffering、临时表七阶段 |
| [21_collation_index_usability.md](21_collation_index_usability.md) | **collation / 类型聚合与索引可用性**：★ **两条独立链**（字符集聚合 vs 类型聚合，后者才是索引失效元凶）、`DTCollation` 七档 derivation 强度体系（低枚举值=高强度）、`agg_item_charsets` 两步（aggregate + **就地永久替换**）、★ **`only_consts` 使字段永不被包转换器**（故字符集不同通常不毁索引）、`agg_cmp_type` 提升到 REAL 域才毁索引、**NO PAD vs PAD SPACE**（CHAR 去尾空格导致"索引返回假候选行"，优化器必须保守）、前缀索引与索引长度限制 |
| [23_optimizer_trace_internals.md](23_optimizer_trace_internals.md) | **optimizer_trace 实现原理**：★ 为什么 context 持"当前结构"指针（深栈埋点 vs 层层传参的 backtrace 例子）、`Opt_trace_struct` 的 **RAII 栈**（构造入栈/析构出栈）、★ **`unlikely(is_started())` 零成本开关**、`add()` 系列重载 + **feature 位掩码过滤**（`MISC` 不可禁用）、`start/end` 生命周期与 OFFSET/LIMIT、★ **I_S 填充里的 SUID 安全洞**（`fill_optimizer_trace_info`）、`end_marker`/`one_line` 的 JSON 序列化 |
| [22_optimizer_worklog_timeline.md](22_optimizer_worklog_timeline.md) | **Worklog 与版本演进**：5.6→8.0.39 优化器/解析器时间线（**两条并行主线**：物理计划表达方式、优化器算法）、**WL 清单按主题分类**（WL#5257 trace / WL#2489 only_full_group_by / WL#5561 semijoin / WL#1110 物化 / WL#4389 EXISTS→SJ / WL#5800 / WL#12108 / WL#7384 / WL#6059，★ 全部编号来自本仓库源码注释）、**源码"活化石"清单**（WL#6570 在 6 个文件的散落位置与含义）、**四组经典 bad case → 源码根因 → 深入哪篇** |
| [19_set_operation.md](19_set_operation.md) | **set operation 的优化侧**：`Query_term` 四类型、`Query_expression::optimize()` 逐段（代价简单相加、**递归 CTE 靠 `query_result()` 跨 block 传行数**、相关子查询行数保护）、★ **`create_access_paths()` 的 `streaming_allowed` 三条件**（顶层 ORDER BY 是"遗留决定"、INSERT-SELECT 的 **Halloween Problem** 防护）、**混合 UNION ALL/DISTINCT 的两种策略**（`m_last_distinct` 切分点）、`setup_materialize_set_op` 的 `activate_deduplication`、★ **INTERSECT/EXCEPT 的临时表计数器算法**（EXCEPT 递减 / INTERSECT DISTINCT 的 N-1 / INTERSECT ALL 的 `HalfCounter` 与 2^32 上限） |
| [20_view_resolution.md](20_view_resolution.md) | **VIEW 的解析与优化**：`open_and_read_view` + `parse_view_definition`（★ 每个 view 有自己的 LEX、`OPEN_FOR_CREATE` 的 dummy LEX）、★ **`merge_derived` 的三级优先级**（ALGORITHM > hint > switch+heuristic，源码注释原文）、★ **view 不受 `allow_merge_derived` 限制**（比等价派生表更容易合并）、`is_mergeable` 的 CTE+RAND 规则、`MAX_TABLES` 限制、可更新视图与 **CHECK OPTION 的三态返回值**（OK/SKIP/ERROR） |
| [17_optimizer_decisions.md](17_optimizer_decisions.md) | **优化器决策全景（横向索引）**：**36 个决策点三张总表**（prepare 期 11 / optimize 规划期 13 / 收尾与代码生成 12），每个标性质（规则 / 代价 / 硬编码）、判据函数、产出、**可覆盖性**；深挖三个决策——**semi-join 执行策略**（`SJ_OPT_*` 六态、`advance_sj_state` 的"先选后重算"省内存妥协）、访问方法、join 算法；**决策的三种失败模式**（代价失真 / 规则边界 / 开关误配）与诊断入口；**`optimizer_switch` 完整 26 项清单** + 倒排速查 |
| [16_join_object_model.md](16_join_object_model.md) | **优化器对象模型与生命周期（对象视角）**：`JOIN` 字段分组剖析（表序/条件/分组排序/计划产物）、**`JOIN_TAB` / `QEP_TAB` / `QEP_shared` 的继承与分工**（`QEP_shared_owner` 半公开封装）、**六个指针数组的生命周期**（`join_tab`/`best_ref`/`map2table`/`positions`/`best_positions`/`qep_tab` 谁分配、谁重排、**谁被主动置空**）、`Temp_table_param` 母版+拷贝、`ORDER_with_src` 溯源、★ **`ref_items` 切片机制**（`REF_SLICE_*` 与 RAII 切换）、`TABLE` 上的优化器成员（含 `reginfo.qep_tab` **双向绑定**）、从 `lex_start` 到 `JOIN::destroy` 的完整生命周期与内存归属、`cleanup` vs `destroy` |
| [15_groupby_distinct_order.md](15_groupby_distinct_order.md) | **GROUP BY / DISTINCT / ORDER BY 的优化期决策**：`optimize_aggregated_query`（聚合常量化三态、`HA_STATS_RECORDS_IS_EXACT` vs `HA_COUNT_ROWS_INSTANT` 双路径）、`optimize_distinct_group_order` 逐段（消序 / 唯一索引消组 / **DISTINCT→GROUP BY 改写** / 子序列删 ORDER BY）、`test_skip_sort`（GROUP BY 优先于 ORDER BY、**`SQL_BIG_RESULT` 的反向逻辑**、LooseScan 冲突保护）、**四种 GROUP BY 实现的判定链**、`make_tmp_tables_info` 的 `need_tmp_before_win` / `allow_group_via_temp_table` / `is_agg_loose_index_scan` |

### 横切（不属四阶段任何单一阶段）

| 文件 | 核心内容 |
|------|----------|
| [11_optimizer_hints.md](11_optimizer_hints.md) | **Optimizer Hints**：双语法器（`sql_hints.yy`）+ 四层对象树（global/qb/table/key）、三类映射（开关/枚举/复杂）、JOIN_ORDER 转 `dependent` 链式依赖、INDEX 转索引位图集合运算、SET_VAR/MAX_EXECUTION_TIME、8.0 已废弃的 hint |
| [12_partition_pruning.md](12_partition_pruning.md) | **分区裁剪**：归约到 range 分析（复用 `get_mm_tree`→SEL_TREE）、`find_used_partitions` 遍历 SEL_ARG 树、RANGE/LIST 二分 vs HASH/KEY 单点精确·范围枚举、子分区递归、read/lock_partitions 位图→handler→EXPLAIN |
| [13_functional_mv_index.md](13_functional_mv_index.md) | **函数索引 + 多值索引**：隐藏生成列（`HT_HIDDEN_SQL`）、GC substitution（`get_gc_for_expr` 表达式等价匹配）、MEMBER OF 转等值 range、多值索引的唯一记录过滤器去重 |
| [14_plan_stability.md](14_plan_stability.md) | **计划稳定性与优化器能力横向对照**（★ 不是实现剖析，是"有什么/缺什么"的对照）：**计划稳定性 / GROUP BY / 下推**三个维度，对照 **Oracle（SPM 计划基线）**、HyPer、Presto；计划回归的成因（统计刷新/数据量/版本/参数）与社区版能力边界（无 SPM、无 plan cache） |

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
