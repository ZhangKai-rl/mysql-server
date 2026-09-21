# 08.0 AccessPath 结构：44 种类型 + 参数 union

> 定义在 `sql/join_optimizer/access_path.h`。本篇只讲**结构**（这棵树长什么样、每个节点带什么参数）。构建过程见 `01_construction_classic.md` / `02_construction_hypergraph.md`，与 RowIterator 的对应见 `06_rowiterator.md`。

## 目录

- [关键成员](#关键成员)
- [44 种类型](#44-种类型)
- [参数 union（代表几个）](#参数-union代表几个)
- [重要类型详解](#重要类型详解)

---

## 关键成员

| 成员 | 说明 |
|------|------|
| `enum Type : uint8_t` | 节点类型（44 种，见下），用 uint8 省空间 |
| `enum Safety` | `SAFE / SAFE_IF_SCANNED_ONCE / UNSAFE`：取 rowid 是否安全 |
| `RowIterator *iterator` | **反向指针**：指向已实例化的迭代器 |
| `double cost` | 读完整路径一次的期望代价，-1.0 未知 |
| `double init_cost` | 初始化代价 |
| `double init_once_cost` | 每 query block 只付一次的 init |
| `num_output_rows()` | 期望输出行数（`kUnknownRowCount = -1.0`） |
| `union {...} u` | 类型专用参数（匿名 union） |

★ 三个代价字段（`cost` / `init_cost` / `init_once_cost`）的区别是理解执行器代价语义的关键，详见 `04_factory_cost.md`。

---

## 44 种类型

按语义分五类：

| 分类 | 类型 |
|------|------|
| **基础表访问（17）** | `TABLE_SCAN, INDEX_SCAN, REF, REF_OR_NULL, EQ_REF, PUSHED_JOIN_REF, FULL_TEXT_SEARCH, CONST_TABLE, MRR, FOLLOW_TAIL, INDEX_RANGE_SCAN, INDEX_MERGE, ROWID_INTERSECTION, ROWID_UNION, INDEX_SKIP_SCAN, GROUP_INDEX_SKIP_SCAN, DYNAMIC_INDEX_RANGE_SCAN` |
| **非特定表（6）** | `TABLE_VALUE_CONSTRUCTOR, FAKE_SINGLE_ROW, ZERO_ROWS, ZERO_ROWS_AGGREGATED, MATERIALIZED_TABLE_FUNCTION, UNQUALIFIED_COUNT` |
| **连接（4）** | `NESTED_LOOP_JOIN, NESTED_LOOP_SEMIJOIN_WITH_DUPLICATE_REMOVAL, BKA_JOIN, HASH_JOIN` |
| **复合（15）** | `FILTER, SORT, AGGREGATE, TEMPTABLE_AGGREGATE, LIMIT_OFFSET, STREAM, MATERIALIZE, MATERIALIZE_INFORMATION_SCHEMA_TABLE, APPEND, WINDOW, WEEDOUT, REMOVE_DUPLICATES, REMOVE_DUPLICATES_ON_INDEX, ALTERNATIVE, CACHE_INVALIDATOR` |
| **修改表（2）** | `DELETE_ROWS, UPDATE_ROWS` |

---

## 参数 union（代表几个）

| 类型 | union 成员 | 关键字段 |
|------|-----------|---------|
| `table_scan` | `TableScanParameters` | `TABLE *table` |
| `index_range_scan` | `IndexRangeScanParameters` | `used_key_part, ranges, num_ranges, index` |
| `nested_loop_join` | `NestedLoopJoinParameters` | `outer, inner, join_type, join_predicate` |
| `hash_join` | `HashJoinParameters` | `outer, inner, join_predicate, allow_spill_to_disk` |
| `filter` | `FilterParameters` | `child, condition` |
| `sort` | `SortParameters` | `child, filesort, order, limit` |
| `aggregate` | `AggregateParameters` | `child, rollup` |

每个类型有带 `assert(type == ...)` 的访问器（如 `table_scan()`、`nested_loop_join()`），**用断言保证取参数时类型正确**——这是"union + 类型标签 + 断言访问器"的经典 C++ 模式。

---

## 重要类型详解

以下挑复杂/易踩坑的几个讲透。

### MATERIALIZE（最重要）

参数：`table_path`（读回物化表的路径，只允许 `TABLE_SCAN`/`REF`/`EQ_REF`/`ALTERNATIVE`/`INDEX_SCAN`）、`param`（`MaterializePathParameters*`）、`subquery_cost`。

`MaterializePathParameters`：`query_blocks[]`（每个物化块）、`invalidators`、`rematerialize`、`limit_rows`、`reject_multiple_rows`、`ref_slice`。

**迭代器藏在 `.cc` 里**：`MaterializeIterator` 定义在 `composite_iterators.cc`，**不导出到任何头文件**——只能经 `materialize_iterator::CreateIterator()` 构造。`TemptableAggregateIterator` 同理。这是 8.0 的封装习惯：复合迭代器不暴露结构。

**Init 流程**：建表 → 共享 CTE 复用判断 → **查 invalidators 的 generation 是否变化决定是否 rematerialize** → `ha_delete_all_rows` → 递归走 `MaterializeRecursive()` 否则逐 block 物化 → 更新 generation → 初始化 table_iterator。

**工厂里两处"静默改写"（易踩坑）**：

1. `rematerialize=true` 时 `invalidators` 被**强制置 nullptr**，并把 `safe_for_rowid` 设为 `SAFE_IF_SCANNED_ONCE`——因为每次都重物化，失效机制失去了意义。
2. 对 INTERSECT/EXCEPT，`limit_rows` 被**强制成 `HA_POS_ERROR`**——去重由 TableScanIterator 完成，LIMIT 不能下推。

### CACHE_INVALIDATOR（缓存失效）

参数：`child`、`name`（仅用于 EXPLAIN，取表别名）。迭代器每次 `Init()`/`Read()`/`SetNullRowFlag()` 都 `++m_generation`，物化方比对 generation 判断依赖是否变化。

**两条硬约束**：

1. **创建顺序**：invalidator 必须在物化路径**左侧**先建好迭代器——`MaterializeIterator` 构造函数里 `assert(invalidator_path->iterator != nullptr)`。
2. **仅旧优化器创建**：`NewInvalidatorAccessPathForTable()`，用于 LATERAL。**hypergraph 优化器从不创建 CACHE_INVALIDATOR**——LATERAL 改用 `rematerialize` + `parameter_tables`，hash join 用 `join->hash_table_generation` 代替（源码有 TODO 明确说明这是旧优化器 invalidator 的替代方案）。

### ALTERNATIVE（优化期建、执行期选）

参数：`table_scan_path`、`child`（ref 路径）、`used_ref`。迭代器在 **`Init()` 时二选一**，之后 `Read()` 纯转发。

**为什么需要它**：`<expr> IN (SELECT...)` 被改写成 ref 访问后，若 `<expr>` 求值为 NULL 则改写**非法**，需退回全表扫描——靠 `used_ref->cond_guards[]` 判定。

**陷阱**：切换分支时必须 `ha_index_or_rnd_end()` 重置 handler（用 `m_last_iterator_inited` 记录），否则上一个迭代器的索引/扫描状态会残留。

### WINDOW

参数：`child`、`window`、`temp_table`、`needs_buffering`。迭代器有流式（`WindowIterator`）与缓冲（`BufferingWindowIterator`）两种。

**两个细节**：

1. `temp_table` 在工厂里被置 nullptr，真正的临时表要到 **finalize 阶段**才建。
2. 窗口路径后面挂物化时必须 `copy_items=false`——拷贝已由窗口迭代器自己完成，再拷一次会重复求值。

### STREAM（"不落盘的 MATERIALIZE"）

参数类似 MATERIALIZE。当优化器以为要物化（read_set/上游 Item 都按临时表布局设好了），实际只需单趟顺序扫时，用它替换 MATERIALIZE：纯转发 + `copy_funcs`。

**它会伪造 rowid**：`provide_rowid` 为真时把自增 `m_row_number` memcpy 进 `table()->file->ref`——这样上层 WEEDOUT 仍能拿到"rowid"。该字段由 `FindTablesToGetRowidFor` 在拿到 weedout 父节点后才置位。

### 去重三兄弟

| 类型 | 参数 | 语义 | 陷阱 |
|------|------|------|------|
| **WEEDOUT** | `SJ_TMP_TABLE*` | semi-join 的"inner join + 事后按外层 rowid 去重" | `is_confluent`（全字段去重）被直接退化成 **LIMIT 1**；工厂里 `tables_to_get_rowid_for` 置 0 并注明"必须由调用方处理" |
| **REMOVE_DUPLICATES** | `group_items` | 删**相邻**重复行 | **输入必须已按分组有序**（只删相邻）；`semijoin_group_size == 0` 时退化成 LIMIT 1 |
| **REMOVE_DUPLICATES_ON_INDEX** | — | LooseScan 的索引级去重 | 头文件明确"**只在非 hypergraph 优化器里使用**"，hypergraph 用 `REMOVE_DUPLICATES` + 排序表达同样语义 |

### APPEND（UNION ALL 的串联）

参数：`Mem_root_array<AppendPathParameters>* children`。工厂构造时就把 cost/init_cost/init_once_cost/行数**按子节点求和**。

迭代器顺序串联：前一个 EOF 后才 `Init` 下一个，`Read()` 用**尾递归**跳到下一个。

**陷阱**：切换子迭代器前后必须显式 `EndPSIBatchModeIfStarted()` / 重新 `StartPSIBatchMode()`——**PFS batch 模式不能跨迭代器边界延续**。
