# 08.4 工厂函数与 cost 三个字段的语义

> 本篇讲两件事：① AccessPath 是怎么被**创建**的（`New*` 工厂函数族）；② 那三个代价字段 `cost` / `init_cost` / `init_once_cost` 到底各是什么意思。代价语义是读 `EXPLAIN ANALYZE` 和 optimizer trace 时最容易混淆的地方。

## 目录

- [cost 三个字段的语义](#cost-三个字段的语义)
- [rescan_cost 的设计](#rescan_cost-的设计)
- [过滤前后的两套值](#过滤前后的两套值)
- [工厂函数：统一的创建模式](#工厂函数统一的创建模式)
- [几个特殊的工厂](#几个特殊的工厂)

---

## cost 三个字段的语义

> 源码 `access_path.h` 的字段注释非常精确，逐条翻译如下。

### `cost`：读完整路径一次的期望代价

```cpp
/// Expected cost to read all of this access path once; -1.0 for unknown.
double cost{-1.0};
```

**把整个路径从头到尾读一遍**的期望代价。`-1.0` 表示未知。

### `init_cost`：初始化代价（读第一行的代价）

```cpp
/// Expected cost to initialize this access path; ie., cost to read
/// k out of N rows would be init_cost + (k/N) * (cost - init_cost).
double init_cost{-1.0};
```

读 **k/N 行**的代价公式：

```
cost(k 行) = init_cost + (k / N) * (cost - init_cost)
```

★ 关键点（注释里专门强调）：**EXPLAIN 打印的是"读第一行"的代价（就是 `init_cost`）**，因为它对用户更直观、也更容易在 `EXPLAIN ANALYZE` 里测量。但内部计算时用的是"纯初始化代价"。

也就是说：你在 `EXPLAIN` 里看到的 cost 不是 `cost` 字段，而是 `init_cost` 字段（读第一行的代价）。这是一个**展示层与内部层的差异**。

### `init_once_cost`：每个 query block 只付一次的初始化

```cpp
/// Of init_cost, how much of the initialization needs only to be done
/// once per query block. (This is a cost, not a proportion.)
double init_once_cost{0.0};
```

`init_cost` 里"**每个 query block 只需付一次**"的那部分。典型例子：**materialized table with `rematerialize=false`**——第二次 `Init()` 是 no-op。

★ 注释明确说了两个边界：

1. **不用于建模缓存效应**（"We do not intend to use this field to model cache effects"）
2. **不打印在 EXPLAIN 里，只在 optimizer trace**（"This is currently not printed in EXPLAIN, only optimizer trace"）

### 三个字段的典型取值

| 节点 | cost | init_cost | init_once_cost |
|---|---|---|---|
| 全表扫描 | 读全表 | 读第一行（≈0） | 0 |
| 物化（rematerialize=false） | 第一次物化 + 读 | 物化 + 读第一行 | **≈物化代价**（第二次 Init 免了物化） |
| const 表 | 0 | 0 | 0 |

---

## rescan_cost 的设计

```cpp
/// Return the cost of scanning the given path for the second time
/// (or later) in the given query block.
double rescan_cost() const { return cost - init_once_cost; }
```

第二次及以后扫描的代价 = `cost - init_once_cost`。

★ 为什么存 `init_once_cost` 而不是直接存 `rescan_cost`？注释给了理由：

> This is really the interesting metric, not init_once_cost in itself, but since nearly all paths have zero init_once_cost, storing that instead allows us to skip a lot of repeated `path->init_once_cost = path->init_cost` calls in the code.

即：**绝大多数节点的 `init_once_cost` 是 0**，存它（而不是 `rescan_cost`）能让代码少写很多次 `init_once_cost = init_cost` 的赋值。这是一个"存罕见值、算常见值"的空间/代码优化。

---

## 过滤前后的两套值

```cpp
/// If no filter, identical to num_output_rows, cost, respectively.
/// init_cost is always the same (filters have zero initialization cost).
double num_output_rows_before_filter{kUnknownRowCount},
    cost_before_filter{-1.0};
```

一个节点如果带 filter，会同时记两套值：

| 字段 | 含义 |
|---|---|
| `num_output_rows` / `cost` | **过滤后**的行数/代价 |
| `num_output_rows_before_filter` / `cost_before_filter` | **过滤前**的行数/代价 |

★ `init_cost` 永远是同一个（**filter 的初始化代价为 0**）。

这也呼应了 `01_construction_classic.md` 里 `SetCostOnTableAccessPath` 的 `is_after_filter` 参数——那个参数就是在决定"把值填进过滤前还是过滤后的字段"。

---

## 工厂函数：统一的创建模式

`New*` 工厂函数族（`NewTableScanAccessPath`、`NewRefAccessPath`、`NewIndexScanAccessPath` …）是 AccessPath 唯一的创建入口，模式完全统一：

```cpp
inline AccessPath *NewTableScanAccessPath(THD *thd, TABLE *table,
                                          bool count_examined_rows) {
  AccessPath *path = new (thd->mem_root) AccessPath;   // ① placement new 到 MEM_ROOT
  path->type = AccessPath::TABLE_SCAN;                  // ② 设置类型标签
  path->count_examined_rows = count_examined_rows;      // ③ 通用字段
  path->table_scan().table = table;                     // ④ 填 union 专用字段
  return path;
}
```

四步：

1. **`new (thd->mem_root) AccessPath`**：placement new 到 `THD::mem_root`，所以**不能 delete**（MEM_ROOT 是块式分配，最后统一 `ClearForReuse`）
2. 设 `type` 标签
3. 填通用字段（`count_examined_rows` 几乎每个都有）
4. 通过类型断言访问器（`table_scan()`）填 union 专用字段

★ 工厂函数的共同特征：

- 全是 `inline`（定义在 `.h`，编译期内联，创建节点零函数调用开销）
- 大部分**不填 cost**（默认 `-1.0` 未知，由后续的代价估算步骤填）
- `count_examined_rows` 控制"这个节点读的行是否计入 `EXPLAIN ANALYZE` 的 rows examined"

---

## 几个特殊的工厂

### NewConstTableAccessPath：零代价特例

```cpp
inline AccessPath *NewConstTableAccessPath(THD *thd, TABLE *table,
                                           Index_lookup *ref,
                                           bool count_examined_rows) {
  ...
  path->set_num_output_rows(1.0);
  path->cost = 0.0;
  path->init_cost = 0.0;
  path->init_once_cost = 0.0;
  ...
}
```

const 表是唯一在工厂里**直接填全 cost** 的：输出 1 行、三个代价全 0。因为 const 表在优化期就已经求值了，执行期读它确实是零代价。

### NewMRRAccessPath：bka_path 延迟填充

```cpp
inline AccessPath *NewMRRAccessPath(THD *thd, TABLE *table, Index_lookup *ref,
                                    int mrr_flags) {
  ...
  // This will be filled in when the BKA iterator is created.
  path->mrr().bka_path = nullptr;
  ...
}
```

`mrr().bka_path` 在工厂里置 `nullptr`，注释明说"**will be filled in when the BKA iterator is created**"——因为 MRR 节点在 BKA join 里是"内表的一部分"，它要指向 BKA 的外表路径，这个指针只能在 BKA 迭代器创建时才知道。

### NewFollowTailAccessPath：递归 CTE

`FOLLOW_TAIL` 节点读"临时表的尾部"（新追加部分），是 `WITH RECURSIVE` 自引用的关键（见 `01_construction_classic.md`）。
