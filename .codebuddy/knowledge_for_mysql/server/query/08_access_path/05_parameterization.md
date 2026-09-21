# 08.5 参数化（parameter_tables）与谓词位图

> 本篇讲 AccessPath 上的两个"优化期元数据"：`parameter_tables`（参数化——这个节点求值时依赖哪些表的行）与四个谓词位图（哪些谓词在哪个节点被应用）。这两个字段是 hypergraph 优化器在规划期操作 AccessPath 的核心抓手，也是 `ExpandFilterAccessPaths` 最终把位图翻译成 `FILTER` 节点的依据。

## 目录

- [parameter_tables：参数化](#parameter_tables参数化)
- [四个谓词位图](#四个谓词位图)
- [内存复用：两个位图共用一块](#内存复用两个位图共用一块)

---

## parameter_tables：参数化

```cpp
/// If nonzero, a bitmap of other tables whose joined-in rows must already be
/// loaded when rows from this access path are evaluated; that is, this
/// access path must be put on the inner side of a nested-loop join (or
/// multiple such joins) where the outer side includes all of the given tables.
hypergraph::NodeMap parameter_tables{0};
```

### 语义

非零时表示：**这个节点求值时，某些表的行必须已经加载**。换句话说，这个节点必须放在 nested-loop join 的**内侧**，且外侧包含这些表。

### 两个来源

| 来源 | 说明 |
|---|---|
| **LATERAL 的 dependent tables** | 最"名正言顺"的情况：派生表引用了左侧表的列，必须先有左侧行 |
| **下推的 join 条件** | 更常见：`t1.x = t2.x` 被下推成 t1 上的 ref access，则 t2 被设进 bitmap（因为 t1 的索引查找要用 t2.x 的值） |

### 关键约束：不能做 hash join 右侧

```cpp
...we cannot be on the right side of a hash join until parameter_tables is zero again.
```

★ 这是参数化最实质的约束：**一个带参数的节点不能做 hash join 的右侧**（build 侧），因为 hash join 的 build 侧是独立物化/建哈希表的，不依赖外层行。所以要等"参数都内联完"（`parameter_tables == 0`）才能作为 hash join 右侧。

### RAND_TABLE_BIT 特殊值

```cpp
// As a special case, we allow setting RAND_TABLE_BIT ... it specifies that
// the access path is entirely noncachable, because it depends on something
// nondeterministic or an outer reference, and thus can never be on the right
// side of a hash join, ever.
```

`RAND_TABLE_BIT`（本不属于 NodeMap）被借用作哨兵：表示这个节点**完全不可 cache**（依赖非确定性函数或外层引用），因此**永远不能做 hash join 右侧**。

### DisallowParameterizedJoinPath

源码提到 `DisallowParameterizedJoinPath()` 是"禁止延迟"的**优化**——即某些情况下，与其让 `parameter_tables` 一直往下传播（把表推到后面去），不如**立即内联**（马上把依赖的表 join 进来），省掉传播的开销。

### 与 CACHE_INVALIDATOR 的分工

回顾 `00_structure.md` 里的 CACHE_INVALIDATOR：那是**旧优化器**处理 LATERAL 缓存失效的机制。hypergraph 优化器**不创建 CACHE_INVALIDATOR**，改用 `parameter_tables` + `rematerialize`（物化时每次重物化）。所以：

| 机制 | 旧优化器 | hypergraph |
|---|---|---|
| LATERAL 依赖表达 | CACHE_INVALIDATOR（generation 计数） | `parameter_tables` + `rematerialize=true` |
| 缓存失效判定 | 比对 invalidator 的 generation | 每次重物化 |

---

## 四个谓词位图

> 这些位图只存在于 **hypergraph 优化器的规划期**（旧优化器用 `Item` 树，不用位图）。它们的存在理由是源码注释明说的：

```
Since bit masks are much cheaper to deal with than creating Item objects,
and we don't invent new conditions during join optimization (all of them are
known when we begin optimization), we stick to manipulating bit masks during
optimization.
```

即：**规划期操作位图比创建 `Item` 对象便宜得多**，且所有条件在优化开始时都已知，所以用位图引用"predicates 数组"。

| 字段 | 含义 |
|---|---|
| `filter_predicates` | 这个节点要应用的 WHERE 谓词（1 位 = 应用；多位则 AND） |
| `applied_sargable_join_predicates` | 已通过 ref access 应用的 sargable join 谓词（**不再重复计算选择性**） |
| `delayed_predicates` | 触达了已 join 的表、但还不能应用的（引用其它表 / 不能下推到外连接 nullable 侧） |
| `subsumed_sargable_join_predicates` | 已应用且**完全覆盖** join 谓词（谓词冗余，不再当 filter） |

★ 四个位图的分工：

- `filter_predicates`（要应用）vs `applied_sargable_join_predicates`（已通过索引应用）——前者是"还没用掉的"，后者是"已经用索引吃掉、别重复计选择性的"
- `delayed_predicates`（推迟）vs `subsumed_sargable_join_predicates`（已 subsumed）——前者"想用但用不了"，后者"用了且覆盖了"

### 何时从位图变成真节点

```cpp
/// This is used during join optimization only; before iterators are created,
/// we will add FILTER access paths to represent these instead, removing the
/// dependency on the array.
```

**位图只在规划期存在**。创建迭代器之前，`ExpandFilterAccessPaths` 把这些位图翻译成真正的 `FILTER` 节点，从而摆脱对"predicates 数组"的依赖。

★ 一个细节：这些 `FILTER` 节点约定 `materialize_subqueries = false`（因为绝大多数谓词里没有子查询）。

---

## 内存复用：两个位图共用一块

```cpp
/// Since these refer to the same array as filter_predicates, they will never
/// overlap with filter_predicates, and so we can reuse the same memory using
/// an alias ... even though the meaning is entirely separate.
```

`filter_predicates` 和 `applied_sargable_join_predicates` 共用**同一块内存**（`applied_sargable_join_predicates()` 返回 `filter_predicates` 的引用）；同理 `delayed_predicates` 和 `subsumed_sargable_join_predicates` 共用。

原理：两者的 bit 不重叠——若 `predicates` 数组有 N 个 WHERE 谓词，则：

```
bit 0 .. N-1      → filter_predicates
bit N .. 更高位    → applied_sargable_join_predicates
```

★ 为什么不用 union？源码注释说了：`OverflowBitset` 是有**非平凡默认构造函数**的类，union 不允许。所以用"两个 accessor 返回同一个成员"的别名方式，语义上完全独立、内存上共用。
