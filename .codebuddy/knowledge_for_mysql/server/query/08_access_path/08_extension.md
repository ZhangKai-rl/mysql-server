# 08.8 扩展指南：新增一个 AccessPath 类型要动哪些地方

> 本篇是"新增类型"的改动清单。不是方法论，而是把 AccessPath 这个 IR 的**扩展点**摊开——让你知道 44 种类型之外再加一种，会牵动哪些文件、哪些约定。

## 新增一个类型要动的六处

| # | 位置 | 改动 |
|---|---|---|
| 1 | `sql/join_optimizer/access_path.h` | `enum Type` 加一个枚举值；union 加一个参数结构体；加 `assert(type == ...)` 的访问器 |
| 2 | 同文件 | 加一个 `NewXxxAccessPath()` 工厂函数 |
| 3 | `sql/join_optimizer/walk_access_paths.h` | `WalkAccessPaths` 的 switch 加分支（遍历子节点） |
| 4 | `sql/join_optimizer/access_path.cc` | `CreateIteratorFromAccessPath` 的 switch 加分支（建迭代器） |
| 5 | `sql/join_optimizer/explain_access_path.cc` | `ExplainAccessPath` 加输出逻辑（TREE + JSON） |
| 6 | `sql/iterators/` | 新迭代器类的实现 |

## 三处容易漏掉的约定

### 1. WalkAccessPaths 的 switch 是"必须同步"的

`walk_access_paths.h` 里那个 44 分支的 switch 是**手写的**，新增类型漏改这里会导致"遍历不到新节点的子节点"——而且是静默的（不会编译报错，只是后处理/EXPLAIN 看不到子节点）。

### 2. cost 三字段要么填全、要么留 -1.0

工厂函数大部分**不填 cost**（默认 `-1.0` 未知），由后续代价估算填。但 `NewConstTableAccessPath` 这类是**填全**的（全 0）。新增工厂时要明确：你的节点是"代价未知、后填"还是"已知、工厂填全"。

### 3. `count_examined_rows` 几乎每个工厂都有

这个 bool 控制"节点读的行是否计入 `EXPLAIN ANALYZE` 的 rows examined"。漏掉它会导致 `EXPLAIN ANALYZE` 的行数统计不对（少计或多计）。

## 一个对照：为什么旧优化器要改的地方更多

如果你是在**旧优化器**路径上加类型（而不是 hypergraph），还要额外改：

- `ConnectJoins`（`01_construction_classic.md`）——旧优化器的翻译逻辑
- 可能涉及 `QEP_TAB` 的相关字段（如果类型对应某种表访问）

而 hypergraph 路径下，`Propose*` 函数直接 new 出 AccessPath，改动更集中。

## 参考资料

- 新增类型最完整的参考：`GROUP_INDEX_SKIP_SCAN` 的引入（`sql/range_optimizer/group_index_skip_scan_plan.cc` 产出的类型，涉及 range 优化器 → AccessPath → 迭代器的全链路）
- 迭代器侧：`sql/iterators/composite_iterators.cc`（复合迭代器的集中地）
