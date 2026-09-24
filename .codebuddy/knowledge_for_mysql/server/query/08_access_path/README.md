# 08 AccessPath：统一的物理计划树

> 基于 MySQL 8.0.39。AccessPath 是**优化器与执行器之间的统一中间表示（IR）**，也是 8.0 重构的枢纽。它被设计成纯数据结构：新旧两个优化器都往它上面吐计划，执行器只认它、不关心计划来自哪个优化器。
>
> 本目录把 AccessPath 拆成多个子篇（因为它是"有自己一整套数据结构 + 被两个优化器复用的核心子系统"，规模与 `07_optimize/` 相当）。

## 目录

- [定位与设计思想](#定位与设计思想)
- [子篇索引](#子篇索引)
- [与其它篇的关系](#与其它篇的关系)

---

## 定位与设计思想

### 多层中间表示（IR）：编译思想的数据库版

整个 SQL 管线（01~07 篇）其实是**一层层中间表示（IR）的递进**，与编译器的 IR 分层同构：

| 层 | IR | 关键属性 |
|---|---|---|
| ① | Parse Tree（`PT_*`） | 纯语法、上下文无关 |
| ② | 逻辑查询树（`Query_block` + `Item`） | 语义完整、**未优化** |
| ③ | 优化后的逻辑树（同一批对象，状态变了） | 逻辑改写完成 |
| ④ | **AccessPath 树** | **物理计划**：选了什么索引、什么 join 方式 |

AccessPath 是**物理 IR**——它不再表达"用户要什么"，而是表达"怎么做"。引入它，是为了在"优化"和"执行"之间划一条清晰的界。

### 术语对照：逻辑计划 vs 物理计划

| 术语 | 同义词 | 对应 IR | 表达 |
|------|--------|---------|------|
| **逻辑计划** | logical plan | QE/QB/Query_term + Item 树（②③ 层） | "**用户要什么**" |
| **物理计划** | physical plan / 查询计划 / query plan / 执行计划 | AccessPath 树（④ 层） | "**怎么做**" |
| **EXPLAIN 输出** | 执行计划的可视化 | AccessPath 树的可读呈现 | 上面的"怎么做"用树/表格画出来 |

> 日常说的"看查询计划"、"EXPLAIN 看执行计划"，看的都是 **AccessPath 树**——它是优化器的最终产物，也是执行器的唯一输入。`07_postprocessing_explain.md` 讲"EXPLAIN 如何把这棵树打印出来"。

### 为什么 AccessPath 是"纯数据结构"

AccessPath 被设计成**纯数据结构**（`struct`，可平凡析构，≤144 字节，MEM_ROOT 分配），刻意不携带行为。这是解耦的关键：

- 优化器（新旧两个）只负责**产出**这棵树，`create_access_paths()`（旧）或 hypergraph（新）填满参数
- 执行器只负责**消费**这棵树，`CreateIteratorFromAccessPath` 把它翻译成 RowIterator

两者之间只通过 AccessPath 的结构定义通信，**谁都不依赖谁的内部状态**。这就是"计划树与迭代器分离"的经典做法——计划是数据，执行是行为。

### 8.0.22 的动机：为 hypergraph 铺路

AccessPath 是 8.0.20 起逐步引入、8.0.22 稳定的。深层动机是 **hypergraph 优化器**：它要支持 bushy tree、多表 join 的任意形状，旧的 `QEP_TAB` 链（只支持左深树）表达不了。所以需要一个**通用的物理计划结构**，让新旧优化器都往这上面吐——**先有统一 IR，才能并行跑两个优化器**。

### 整体印象

```
旧优化器：QEP_TAB 链 ──create_access_paths()──┐
                                              ├──▶ AccessPath 树 ──▶ RowIterator 树
新优化器：直接产出 ───────────────────────────┘
```

关键设计：

1. AccessPath 是**纯数据结构**（`struct`，可平凡析构，≤144 字节，MEM_ROOT 分配）
2. 与 RowIterator **1:1 对应**（`CreateIteratorFromAccessPath` 翻译）
3. 一个路径节点可以**被多个上游节点共享**

---

## 子篇索引

| 子篇 | 主题 | 状态 |
|---|---|---|
| [00_structure.md](00_structure.md) | 结构：44 种类型 + 参数 union + 重要类型详解（MATERIALIZE / CACHE_INVALIDATOR / ALTERNATIVE / WINDOW / STREAM / 去重三兄弟 / APPEND） | ✅ |
| [01_construction_classic.md](01_construction_classic.md) | 构建（旧优化器）：`ConnectJoins` / `GetTableAccessPath` / `create_table_access_path` / 包裹顺序 / `SetCostOnTableAccessPath` | ✅ |
| [02_construction_hypergraph.md](02_construction_hypergraph.md) | 构建（新优化器）：**Propose 模式**（栈上值对象 → MEM_ROOT）、`ProposeTableScan` / `ProposeRefAccess` 逐段、`parameter_tables` 写入点 | ✅ |
| [03_set_operation.md](03_set_operation.md) | 构建（集合操作层）：`create_access_paths` 完整控制流、两种策略、`setup_materialize_set_op` | ✅ |
| [04_factory_cost.md](04_factory_cost.md) | 工厂函数 + **cost/init_cost/init_once_cost 三字段语义** + `rescan_cost` 设计 | ✅ |
| [05_parameterization.md](05_parameterization.md) | 参数化：`parameter_tables`（含 `RAND_TABLE_BIT` 哨兵）+ **四个谓词位图** + 内存复用 | ✅ |
| [06_rowiterator.md](06_rowiterator.md) | ★ **AccessPath→RowIterator 翻译的唯一定位**（329 行，`09_executor_iterator` 第三章已并入此处）：1:1 映射、显式栈（为何不用递归）、两阶段与 `IteratorToBeCreated`、`NewIterator`+`count_examined_rows`、**batch mode 沿树传递**（★是监控埋点批量，非计算批量）、**翻译边界：什么被延迟什么没有**（★纠正"物化子查询惰性翻译"这一误传；真惰性在 `Query_expression` 根迭代器 / `SortingIterator` 结果迭代器 / 临时表 instantiate）、各 switch 分支要点、MATERIALIZE 特殊处理、EXPLAIN ANALYZE 的 `TimingIterator` 包装与翻译失败路径 | ✅ |
| [07_postprocessing_explain.md](07_postprocessing_explain.md) | 后处理 + `WalkAccessPaths` 模板 + EXPLAIN 反解（`ExplainChild`） | ✅ |
| [08_extension.md](08_extension.md) | 扩展指南：如何新增一个 AccessPath 类型（六处改动） | ✅ |
| [09_finalize.md](09_finalize.md) | **最终化**：`FinalizePlanForQueryBlock` 逐段（合并 FILTER、延迟建表、Item 改写、聚合器设置、hypergraph 不支持 LIS） | ✅ |

---

## 与其它篇的关系

| 篇 | 关系 |
|---|---|
| [`../07_optimize/`](../07_optimize/) | 优化器产出的就是 AccessPath；其中 `physical/18_hypergraph_advanced.md` 讲 hypergraph 侧的最终化 |
| [`../09_executor_iterator.md`](../09_executor_iterator.md) | 消费 AccessPath 的下半段：`CreateIteratorFromAccessPath` 翻译成 RowIterator |
| [`../07_optimize/16_join_object_model.md`](../07_optimize/16_join_object_model.md) | `JOIN::m_root_access_path` 是 AccessPath 树在 JOIN 上的挂载点 |
| [`../07_optimize/logical/03_semijoin.md`](../07_optimize/logical/03_semijoin.md) | 五种 semi-join 策略的 AccessPath 形态 |
| [`../07_optimize/19_set_operation.md`](../07_optimize/19_set_operation.md) | UNION/INTERSECT/EXCEPT 的 AccessPath（对应本目录 `03_set_operation.md`） |
