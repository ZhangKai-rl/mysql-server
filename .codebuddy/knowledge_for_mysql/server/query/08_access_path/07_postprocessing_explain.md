# 08.7 后处理与 EXPLAIN 反解

> AccessPath 树建好之后、执行之前，还有一层"后处理"：把优化期的临时表示（谓词位图、`nullptr` 的临时表）翻译成可执行的最终形态；同时 `EXPLAIN` 要能把这棵树打印成人能读的样子。本篇讲这两件事，核心是一个遍历机制 `WalkAccessPaths`。

## 目录

- [WalkAccessPaths：遍历机制](#walkaccesspaths遍历机制)
- [后处理的代表](#后处理的代表)
- [EXPLAIN 反解](#explain-反解)

---

## WalkAccessPaths：遍历机制

> `sql/join_optimizer/walk_access_paths.h`。它是所有"后处理"的公共底座。

### 为什么用模板

```cpp
template <class AccessPathPtr, class Func, class JoinPtr>
void WalkAccessPaths(AccessPathPtr path, JoinPtr join,
                     WalkAccessPathPolicy cross_query_blocks, Func &&func,
                     bool post_order_traversal = false) {
```

三个模板参数：`AccessPathPtr`（`AccessPath*` 或 `const AccessPath*`）、`JoinPtr`（`JOIN*` 或 `const JOIN*` 或 `nullptr`）、`Func`（回调）。

★ 用模板而不是虚函数/函数指针的原因：回调会被**内联**到遍历里，大量后处理遍历（explain、找 rowid、收 filesort……）都在热路径上，模板避免了每次回调的间接跳转。

### 三种策略（policy）

```cpp
enum class WalkAccessPathPolicy {
  STOP_AT_MATERIALIZATION,   // 不跨 MATERIALIZE/STREAM（留在当前 query block）
  ENTIRE_QUERY_BLOCK,        // 跨整个 query block（需要 join 参数跟踪）
  ENTIRE_TREE                // 跨整棵树
};
```

### pre-order 与 post-order

- **pre-order**（默认）：先调 `func(path)`，若返回 `true` 则**不下降**到子节点（剪枝）
- **post-order**：先遍历子节点再调 `func`，此时子节点已遍历完，`func` 的返回值被忽略（注释明说 "it is too late to skip them"）

### `join` 参数：跨 query block 的跟踪

AccessPath 本身**不含"属于哪个 query block"的信息**。但 `MATERIALIZE` / `STREAM` 节点会**切换 query block**（物化的子查询是另一个 block）。所以：

```cpp
/// The `join` parameter signifies what query block `path` is part of, since
/// that is not implicit from the path itself. The function will track this as
/// it changes throughout the tree (in MATERIALIZE or STREAM access paths).
```

`WalkAccessPaths` 在遍历过程中跟踪 query block 的变化，给回调传正确的 `JOIN*`。如果 policy 不是 `ENTIRE_QUERY_BLOCK`，这个参数只用于回调、可以传 `nullptr`。

### switch 遍历每个类型的孩子

```cpp
switch (path->type) {
  case AccessPath::TABLE_SCAN: ... case AccessPath::UNQUALIFIED_COUNT:
    // No children.
    break;
  case AccessPath::NESTED_LOOP_JOIN:
    WalkAccessPaths(path->nested_loop_join().outer, ...);
    WalkAccessPaths(path->nested_loop_join().inner, ...);
    break;
  ...
}
```

★ 这里体现了 AccessPath 设计的一个代价：**每种类型的 child 结构不同，遍历必须写一个 44 分支的 switch**。任何新增类型都要同步改这个 switch（这正是 `08_extension.md` 里"新增类型要动的地方"之一）。

---

## 后处理的代表

建树之后到执行之前的"最终化"，典型两件事（细节见 [`../07_optimize/physical/18_hypergraph_advanced.md`](../07_optimize/physical/18_hypergraph_advanced.md)）：

| 后处理 | 做什么 |
|---|---|
| `ExpandFilterAccessPaths` | 把 `filter_predicates` / `delayed_predicates` 这些**位图**翻译成真正的 `FILTER` 节点（见 `05_parameterization.md`），摆脱对 "predicates 数组" 的依赖 |
| `FinalizePlanForQueryBlock` | 建临时表、建 Filesort、把 Item 改写指向临时表、`push_to_engines()`——**只能调一次**（破坏性操作） |

两者都基于 `WalkAccessPaths` 做遍历。

---

## EXPLAIN 反解

> `sql/join_optimizer/explain_access_path.cc`。把 AccessPath 树翻译成 `EXPLAIN FORMAT=TREE` / `FORMAT=JSON` 的输出。

### ExplainChild 结构

```cpp
/// This structure encapsulates the information needed to create a Json object
/// for a child access path.
struct ExplainChild {
  AccessPath *path;
  // Normally blank. If not blank, a heading for this iterator
  // saying what kind of role it has to the parent ...
  string heading;
};
```

`ExplainChild` 是"一个子路径 + 一个可选的角色标题"——比如 hash join 的左侧被标成 "Left side"，让 JSON 输出里能区分"这个子是 build 侧还是 probe 侧"。

### 输出从哪来

- **`FORMAT=TREE`**：`--><--` 缩进树（`ExplainAccessPath` 生成）
- **`FORMAT=JSON`**：`json_dom` 构造（`ExplainChild` → JSON 对象）
- **`EXPLAIN ANALYZE`**：额外挂 `actual_time` / `rows`（从 `RowIterator` 的 timing 信息回填，见 `00_structure.md` 里 `iterator` 反向指针的用途）

★ `AccessPath::iterator` 那个反向指针（`RowIterator *iterator`）在 EXPLAIN ANALYZE 里的作用就是：**执行后反查每个节点实际跑了多久、多少行**。这是"计划（静态）与执行（动态）桥接"的关键一环。

### 与 `opt_explain_*` 的分工

- `explain_access_path.cc`：hypergraph 的 AccessPath → JSON/tree
- `sql/opt_explain.cc` / `opt_explain_traditional.cc`：旧优化器的 `QEP_TAB` → 传统 `EXPLAIN` 表格 / JSON

两条路径在 `EXPLAIN` 命令层面汇合（`EXPLAIN FORMAT=TREE` 时旧优化器也走 AccessPath，因为旧优化器最终也产出了 AccessPath 树）。
