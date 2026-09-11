# 08 AccessPath：统一的物理计划树

> 本篇覆盖：第 ④ 层 IR —— AccessPath 树。它是 8.0 新旧优化器的**统一物理计划 IR**，也是"解耦优化器与执行器"的关键。

## 目录

- [设计思想与理论基础](#设计思想与理论基础)
- [先有个整体印象](#先有个整体印象)
- [一、结构：44 种类型 + 参数 union](#一结构44-种类型--参数-union)
- [二、两条创建路径](#二两条创建路径)
- [三、工厂函数](#三工厂函数)
- [四、与 RowIterator 的 1:1 关系](#四与-rowiterator-的-11-关系)
- [五、cost 字段的语义](#五cost-字段的语义)
- [五.5 参数化：parameter_tables](#五5-参数化parameter_tables)
- [六、后处理](#六后处理)
- [七、EXPLAIN 输出从哪来](#七explain-输出从哪来)
- [九、扩展指南：如何新增一个 AccessPath 类型](#九扩展指南如何新增一个-accesspath-类型)
- [核心调用栈](#核心调用栈)

---

## 设计思想与理论基础

### 多层中间表示（IR）：编译思想的数据库版

整个 SQL 管线（01~07 篇）其实是**一层层中间表示（IR）的递进**，与编译器的 IR 分层同构：

| 层 | IR | 关键属性 |
|---|---|---|
| ① | Parse Tree（`PT_*`） | 纯语法、上下文无关 |
| ② | 逻辑查询树（`Query_block` + `Item`） | 语义完整、**未优化** |
| ③ | 优化后的逻辑树（同一批对象，状态变了） | 逻辑改写完成 |
| ④ | **AccessPath 树** | **物理计划**：选了什么索引、什么 join 方式 |

AccessPath 是**物理 IR**——它不再表达"用户要什么"，而是表达"怎么做"。引入它，是为了在"优化"和"执行"之间划一条清晰的界。

### 术语对照：逻辑计划 vs 物理计划（= 查询计划）

| 术语 | 同义词 | 对应 IR | 表达 |
|------|--------|---------|------|
| **逻辑计划** | logical plan | QE/QB/Query_term + Item 树（②③ 层） | "**用户要什么**"（`SELECT a FROM t WHERE b>1`） |
| **物理计划** | physical plan / **查询计划** / query plan / **执行计划** | AccessPath 树（④ 层，本篇） | "**怎么做**"（主键扫描还是 range？hash join 还是 NLJ？） |
| **EXPLAIN 输出** | 执行计划的可视化 | AccessPath 树的可读呈现 | 上面的"怎么做"用树/表格画出来 |

> 日常说的"看查询计划"、"EXPLAIN 看执行计划"，看的都是 **AccessPath 树**——它是优化器（07_optimize）的最终产物，也是执行器（08_executor）的唯一输入。本篇第七节"EXPLAIN 输出从哪来"讲的正是"EXPLAIN 如何把这棵树打印出来"。

### 为什么 AccessPath 是"纯数据结构"

AccessPath 被设计成**纯数据结构**（`struct`，可平凡析构，≤144 字节，MEM_ROOT 分配），刻意不携带行为。这是解耦的关键：

- 优化器（新旧两个）只负责**产出**这棵树，`create_access_paths()`（旧）或 hypergraph（新）填满参数
- 执行器只负责**消费**这棵树，`CreateIteratorFromAccessPath` 把它翻译成 RowIterator

两者之间只通过 AccessPath 的结构定义通信，**谁都不依赖谁的内部状态**。这就是"计划树与迭代器分离"的经典做法——计划是数据，执行是行为。

### 8.0.22 的动机：为 hypergraph 铺路

AccessPath 是 8.0.20 起逐步引入、8.0.22 稳定的。深层动机是 **hypergraph 优化器**（`07_optimize/physical/09_hypergraph.md`）：它要支持 bushy tree、多表 join 的任意形状，旧的 `QEP_TAB` 链（只支持左深树）表达不了。所以需要一个**通用的物理计划结构**，让新旧优化器都往这上面吐——**先有统一 IR，才能并行跑两个优化器**。

---

## 先有个整体印象

8.0.22 之前，执行计划揉在 `JOIN`/`QEP_TAB` 里，执行器直接读它。8.0 重构后引入 **AccessPath**，作为**纯数据结构的计划树**：

```
旧优化器：QEP_TAB 链 ──create_access_paths()──┐
                                              ├──▶ AccessPath 树 ──▶ RowIterator 树
新优化器：直接产出 ───────────────────────────┘
```

**为什么需要这一层**：让"优化器怎么产生计划"和"执行器怎么执行计划"解耦 —— 执行器只认 AccessPath，不关心它来自哪个优化器。阿里云那篇说的"AccessPath 是解耦执行器和优化器的 Plan Tree"就是这个意思。

**关键设计**：

1. AccessPath 是**纯数据结构**（`struct`，可平凡析构，≤144 字节，MEM_ROOT 分配）
2. 与 RowIterator **1:1 对应**（`CreateIteratorFromAccessPath` 翻译）
3. 一个路径节点可以**被多个上游节点共享**（`static_assert` 说明）

---

## 一、结构：44 种类型 + 参数 union

定义：`sql/join_optimizer/access_path.h:193`。

### 1.1 关键成员

| 成员 | 行号 | 说明 |
|------|------|------|
| `enum Type : uint8_t` | `:209-266` | 类型（44 种，见下） |
| `enum Safety` | `:275-291` | `SAFE / SAFE_IF_SCANNED_ONCE / UNSAFE`（取 rowid 是否安全） |
| `RowIterator *iterator` | `:361` | **反向指针**：指向已实例化的迭代器 |
| `double cost` | `:364` | 读完整路径一次的期望代价，-1.0 未知 |
| `double init_cost` | `:373` | 初始化代价 |
| `double init_once_cost` | `:385` | 每 query block 只付一次的 init |
| `num_output_rows()` | `:849` | 期望输出行数（`kUnknownRowCount=-1.0`） |
| `union {...} u` | `:863-1217` | 类型专用参数 |

### 1.2 44 种类型（`access_path.h:209-266`）

| 分类 | 类型 |
|------|------|
| 基础表访问（17） | `TABLE_SCAN, INDEX_SCAN, REF, REF_OR_NULL, EQ_REF, PUSHED_JOIN_REF, FULL_TEXT_SEARCH, CONST_TABLE, MRR, FOLLOW_TAIL, INDEX_RANGE_SCAN, INDEX_MERGE, ROWID_INTERSECTION, ROWID_UNION, INDEX_SKIP_SCAN, GROUP_INDEX_SKIP_SCAN, DYNAMIC_INDEX_RANGE_SCAN` |
| 非特定表（6） | `TABLE_VALUE_CONSTRUCTOR, FAKE_SINGLE_ROW, ZERO_ROWS, ZERO_ROWS_AGGREGATED, MATERIALIZED_TABLE_FUNCTION, UNQUALIFIED_COUNT` |
| 连接（4） | `NESTED_LOOP_JOIN, NESTED_LOOP_SEMIJOIN_WITH_DUPLICATE_REMOVAL, BKA_JOIN, HASH_JOIN` |
| 复合（15） | `FILTER, SORT, AGGREGATE, TEMPTABLE_AGGREGATE, LIMIT_OFFSET, STREAM, MATERIALIZE, MATERIALIZE_INFORMATION_SCHEMA_TABLE, APPEND, WINDOW, WEEDOUT, REMOVE_DUPLICATES, REMOVE_DUPLICATES_ON_INDEX, ALTERNATIVE, CACHE_INVALIDATOR` |
| 修改表（2） | `DELETE_ROWS, UPDATE_ROWS` |

### 1.3 参数 union（代表几个）

| 类型 | union 成员 | 关键字段 |
|------|-----------|---------|
| `table_scan` | `:864-866` | `TABLE *table` |
| `index_range_scan` | `:915-957` | `used_key_part, ranges, num_ranges, index` |
| `nested_loop_join` | `:1068-1087` | `outer, inner, join_type, join_predicate` |
| `hash_join` | `:1052-1059` | `outer, inner, join_predicate, allow_spill_to_disk` |
| `filter` | `:1095-1108` | `child, condition` |
| `sort` | `:1109-1122` | `child, filesort, order, limit` |
| `aggregate` | `:1123-1126` | `child, rollup` |

每个类型有带 `assert(type == ...)` 的访问器（`:496-847`，如 `table_scan()`、`nested_loop_join()`）。

### 1.4 重要类型详解（复杂/易踩坑的几个）

#### MATERIALIZE（最重要）

参数（`access_path.h:1152-1163`）：`table_path`（读回物化表的路径，只允许 TABLE_SCAN/REF/EQ_REF/ALTERNATIVE/INDEX_SCAN）、`param`（`MaterializePathParameters*`）、`subquery_cost`。

`MaterializePathParameters`（`materialize_path_parameters.h:40-100`）：`query_blocks[]`（每个物化块）、`invalidators`、`rematerialize`、`limit_rows`、`reject_multiple_rows`、`ref_slice`。`QueryBlock`（42-57）含 `subquery_path`/`disable_deduplication_by_hash_field`/`copy_items`。

**迭代器藏在 .cc 里**：`MaterializeIterator` 定义在 `composite_iterators.cc:620`，**不导出到任何头文件**——只能经 `materialize_iterator::CreateIterator()`（`composite_iterators.h:511`）构造。`TemptableAggregateIterator`（`.cc:1589`）同理（`:538`）。这是 8.0 的封装习惯：复合迭代器不暴露结构。

**Init 流程**（`composite_iterators.cc:824-977`）：建表 → 共享 CTE 复用判断 → **查 invalidators 的 generation 是否变化决定是否 rematerialize**（:872）→ `ha_delete_all_rows` → 递归走 `MaterializeRecursive()` 否则逐 block 物化 → 更新 generation → 初始化 table_iterator。

**工厂里两处"静默改写"**（易踩坑）：
1. `rematerialize=true` 时 `invalidators` 被**强制置 nullptr**（`access_path.h:1556-1562`），并把 `safe_for_rowid` 设为 `SAFE_IF_SCANNED_ONCE`（:1588）——因为每次都重物化，失效机制失去了意义。
2. 对 INTERSECT/EXCEPT（`table->is_union_or_table()` 为假）`limit_rows` 被**强制成 `HA_POS_ERROR`**（:1568-1573）——去重由 TableScanIterator 完成，LIMIT 不能下推。

#### CACHE_INVALIDATOR（缓存失效）

参数（`access_path.h:1203-1206`）：`child`、`name`（仅用于 EXPLAIN，取表别名）。迭代器 `composite_iterators.h:392`：每次 `Init()`/`Read()`/`SetNullRowFlag()` 都 `++m_generation`（:401-414），物化方比对 generation 判断依赖是否变化。

**两条硬约束**：
1. **创建顺序**：invalidator 必须在物化路径**左侧**先建好迭代器——`MaterializeIterator` 构造函数里 `assert(invalidator_path->iterator != nullptr)`（`composite_iterators.cc:804`）。
2. **仅旧优化器创建**：`sql_executor.cc:809 NewInvalidatorAccessPathForTable()`，用于 LATERAL（`:2806`）。**超图优化器从不创建 CACHE_INVALIDATOR**——LATERAL 改用 `rematerialize` + `parameter_tables`，hash join 用 `join->hash_table_generation` 代替（`access_path.cc:807-818` 的 TODO 明确说明这是旧优化器 invalidator 的替代方案）。

#### ALTERNATIVE（优化期建、执行期选）

参数（`access_path.h:1196-1202`）：`table_scan_path`、`child`（ref 路径）、`used_ref`。迭代器 `ref_row_iterators.h:263`：**`Init()` 时二选一**，之后 `Read()` 纯转发。

**为什么需要它**：`<expr> IN (SELECT...)` 被改写成 ref 访问后，若 `<expr>` 求值为 NULL 则改写**非法**，需退回全表扫描——靠 `used_ref->cond_guards[]` 判定（`sql_executor.cc:3770-3802` 长注释）。

**陷阱**：切换分支时必须 `ha_index_or_rnd_end()` 重置 handler（`ref_row_iterators.cc:793-796`，用 `m_last_iterator_inited` 记录），否则上一个迭代器的索引/扫描状态会残留。

#### WINDOW

参数（`access_path.h:1172-1179`）：`child`、`window`、`temp_table`、`needs_buffering`。迭代器 `window_iterators.h:94`（`WindowIterator` 流式）/ `:204`（`BufferingWindowIterator` 缓冲）。

**两个细节**：
1. `temp_table` 在工厂里被置 nullptr（`access_path.h:1636`），真正的临时表要到 **finalize 阶段**才建（`finalize_plan.cc:501`）。
2. 窗口路径后面挂物化时必须 `copy_items=false`（`join_optimizer.cc:5738`）——拷贝已由窗口迭代器自己完成，再拷一次会重复求值。

#### STREAM（"不落盘的 MATERIALIZE"）

参数（`access_path.h:1144-1151`）。迭代器 `composite_iterators.h:558`。当优化器以为要物化（read_set/上游 Item 都按临时表布局设好了），实际只需单趟顺序扫时，用它替换 MATERIALIZE：纯转发 + `copy_funcs`。

**它会伪造 rowid**：`provide_rowid` 为真时把自增 `m_row_number` memcpy 进 `table()->file->ref`（`composite_iterators.cc:1571-1574`）——这样上层 WEEDOUT 仍能拿到"rowid"。该字段由 `FindTablesToGetRowidFor` 在拿到 weedout 父节点后才置位（`access_path.h:1525-1526`）。

#### 去重三兄弟

| 类型 | 参数 | 语义 | 陷阱 |
|------|------|------|------|
| **WEEDOUT** | `:1180-1184`（`SJ_TMP_TABLE*`） | semi-join 的"inner join + 事后按外层 rowid 去重" | `is_confluent`（全字段去重）被直接退化成 **LIMIT 1**（`sql_executor.cc:1218-1226`）；工厂里 `tables_to_get_rowid_for` 置 0 并注明"必须由调用方处理" |
| **REMOVE_DUPLICATES** | `:1185-1189`（`group_items`） | 删**相邻**重复行 | **输入必须已按分组有序**（只删相邻）；`semijoin_group_size == 0` 时退化成 LIMIT 1 |
| **REMOVE_DUPLICATES_ON_INDEX** | `:1190-1195` | LooseScan 的索引级去重 | 头文件 720-722 明确"**只在非超图优化器里使用**"，超图用 `REMOVE_DUPLICATES` + 排序表达同样语义 |

#### APPEND（UNION ALL 的串联）

参数（`access_path.h:1169-1171`）：`Mem_root_array<AppendPathParameters>* children`。工厂 `:1609` 构造时就把 cost/init_cost/init_once_cost/行数**按子节点求和**（:1613-1624）。

迭代器 `composite_iterators.h:859`：顺序串联，前一个 EOF 后才 `Init` 下一个，`Read()` 用**尾递归**（`.cc:2260 return Read();`）跳到下一个。

**陷阱**：切换子迭代器前后必须显式 `EndPSIBatchModeIfStarted()` / 重新 `StartPSIBatchMode()`（:2250-2259）——**PFS batch 模式不能跨迭代器边界延续**。

---

## 二、两条创建路径

### 2.1 旧优化器：QEP_TAB → AccessPath

```cpp
// sql/sql_executor.cc:2960
void JOIN::create_access_paths() {
  assert(m_root_access_path == nullptr);
  AccessPath *path = create_root_access_path_for_join();      // :2963
  path = attach_access_paths_for_having_and_limit(path);      // :2964
  path = attach_access_path_for_update_or_delete(path);       // :2965
  m_root_access_path = path;
}
```

`create_root_access_path_for_join()`（`:2970`）→ **`ConnectJoins()`（`:2356`）** 是 QEP_TAB 链翻译的核心：

- `FindSubstructure()`（`:2421`）识别 INNER/OUTER/SEMIJOIN 子结构
- 单表走 `GetTableAccessPath()`（`:2714` → `create_table_access_path` `:4690`）
- BKA 包 `NewMRRAccessPath`，LooseScan 用 `NewRemoveDuplicatesOnIndexAccessPath`

`create_table_access_path()`（`sql_executor.cc:4690`）：有 range_scan 就用 range_scan，否则递归引用用 FOLLOW_TAIL，否则 TABLE_SCAN；再用 `POSITION` 填成本。

### 2.2 新优化器：直接产出

`FindBestQueryPlan()` 直接构造 AccessPath（见 07 篇）。

### 2.3 集合操作层

`Query_expression::create_access_paths()`（`sql_union.cc:1420`）：`is_simple()` 直接取 `first_query_block()->join->root_access_path()`；否则各分支物化后 `NewAppendAccessPath`（UNION ALL）。

---

## 三、工厂函数

全部在 `access_path.h:1241` 起（inline，`new (thd->mem_root) AccessPath` + 设 type/字段）：

`NewTableScanAccessPath`(:1241)、`NewIndexScanAccessPath`(:1250)、`NewRefAccessPath`(:1263)、`NewEQRefAccessPath`(:1288)、`NewFullTextSearchAccessPath`(:1312)、`NewConstTableAccessPath`(:1328)、`NewMRRAccessPath`(:1343)、`NewFilterAccessPath`(:1415)、`NewSortAccessPath`(:1427)、`NewAggregateAccessPath`(:1430)、`NewTemptableAggregateAccessPath`(:1439)、`NewLimitOffsetAccessPath`(:1452)、`NewMaterializeAccessPath`(:1546)、`NewAppendAccessPath`(:1609)、`NewWindowAccessPath`(:1628)、`NewWeedoutAccessPath`(:1643)、`NewUpdateRowsAccessPath`(:1701) 等。

> 注意：`NESTED_LOOP_JOIN` / `BKA_JOIN` / `HASH_JOIN` / `INDEX_RANGE_SCAN` / `INDEX_MERGE` **没有 inline 工厂**，由 `sql_executor.cc` 的 `CreateNestedLoopAccessPath`(:792)、`CreateBKAAccessPath`(:836)、`CreateHashJoinAccessPath`(:1964) 及 range_optimizer 直接构造。

---

## 四、与 RowIterator 的 1:1 关系

```cpp
// access_path.h:357-361
/// If an iterator has been instantiated for this access path, points to the
/// iterator. Used for constructing iterators that need to talk to each other
/// (e.g. for recursive CTEs, or BKA join), and also for locating timing
/// information in EXPLAIN ANALYZE queries.
RowIterator *iterator = nullptr;
```

- 赋值点：`access_path.cc:1192`（`CreateIteratorFromAccessPath` 尾部 `path->iterator = iterator.get()`）
- `IteratorsAreNeeded()`（`sql_optimizer.cc:11463`）：只有二级引擎用外部执行器时才跳过迭代器创建，否则 AccessPath 必翻译成 RowIterator

---

## 五、cost 字段的语义

| 字段 | 含义 |
|------|------|
| `cost` | 读完整路径一次的期望代价（`-1.0` 未知） |
| `init_cost` | 初始化代价；读 k/N 行 = `init_cost + (k/N)*(cost - init_cost)` |
| `init_once_cost` | init 中每 query block 只付一次的部分（多数为 0，不进 EXPLAIN） |
| `num_output_rows()` | 期望输出行数 |

估算来源（旧优化器）`sql_executor.cc:1862 SetCostOnTableAccessPath`：

```cpp
double num_rows_after_filtering = pos->rows_fetched * pos->filter_effect;  // :1865
double cost = pos->read_cost
              + cost_model.row_evaluate_cost(num_rows_after_filtering);   // :1874
```

join 代价由 `SetCostOnNestedLoopAccessPath`(:1885) / `SetCostOnHashJoinAccessPath`(:1921) 从 `POSITION` 推算。

---

## 五.5 参数化：parameter_tables（prepared stmt / LATERAL / 相关子查询）

**定义**（`access_path.h:463-484`）：`hypergraph::NodeMap parameter_tables{0}`——非零位图表示"本路径每行求值时，哪些其他表的已连接行必须已载入"，即该路径必须做 NLJ 的内侧。

```cpp
// 注释原文要点（access_path.h:463-483）：
// - LATERAL 依赖表是最明显的例子
// - 更常见：引用其他表的连接条件下推到索引查找（ref access）
// - 这些 bit 会传播，直到 parameter_tables 归零前不能做 hash join 内侧
// - 特例：允许置 RAND_TABLE_BIT，表示该路径完全不可缓存（依赖不确定值/外引用）
```

**赋值点**（谁设它）：

| 位置 | 场景 |
|------|------|
| `join_optimizer.cc:2036-2038` | **REF/EQ_REF**：sargable 谓词另一侧的表并入（`:1823-1824` 累积） |
| `:2264-2271` | **表函数**（JSON_TABLE）：参数 = 函数 used_tables；含外引用/不确定 → 加 `RAND_TABLE_BIT` 保证永不进 hash 表 |
| `:2298-2301` | **LATERAL 派生表**：`parameter_tables = m_lateral_deps` 转节点位图 |
| `:3360-3362` | **HASH_JOIN**：`(left|right 的参数并集) & ~(left|right)`——子图内部参数互相消化，剩余向上传播 |
| `:3849-3851` | **NESTED_LOOP_JOIN**：同上公式 |

> **相关子查询不用 parameter_tables**：它用 `uncacheable & UNCACHEABLE_DEPENDENT` 标记，hypergraph 据此设 `rematerialize = true`（`:2289-2294`）——每次 Init 重物化。递归 CTE 也不用它（`forced_leftmost_table` 强制 FOLLOW_TAIL 最左）。

**消费点**（谁读它）：

1. **ProposeHashJoin 前置拒绝**（`:3266-3271`）：`Overlaps(left->parameter_tables, right) || Overlaps(right->parameter_tables, left | RAND_TABLE_BIT)` → 直接返回。注释原文："**Parameterizations must be resolved by nested loop**"（参数化必须由 NLJ 消化）。
2. **hash join init_once_cost 公式**（`:3479-3506`）：`right->parameter_tables > 0` 时 `reuse_buffer_probability = 0.0`——hash 表**完全不可复用**，每次 Init 重建。
3. **DisallowParameterizedJoinPath**（`:2865-2899`）：join 后仍有未消化参数、且某侧本可立即消化却推迟 → 禁止该 join——**强制参数化 ref 计划左深**（Postgres 技巧）。
4. **CompareAccessPaths**（`:4150-4157`）：parameter_tables 按**子集偏序**比较（参数更少者优）；参数化路径的 ordering_state 清零参与比较（`:4159-4166`）。
5. **执行期**（`access_path.cc:796-818`）：hash join 的 `hash_table_generation` 指针仅在 `parameter_tables == 0` 时传入——参数化 hash join 每次 Init 强制重建。

**对代价的关键影响**：

- **NLJ 内表每外行重执行**（`:4012-4020`）：`first_loop_cost = inner.cost × min(1, ceil(outer_rows))`；`subsequent_loops_cost = inner.rescan_cost() × max(0, outer_rows-1)`。参数化 ref 的 `init_cost=0`，`rescan_cost` 即单次 seek 成本，逐外行累加。
- **`init_once_cost` / `rescan_cost()` 分离**（`access_path.h:367-393`）：`rescan_cost() = cost - init_once_cost`——第一次 Init 和后续 Init 的代价分开建模，参数化正是靠这个让"内表重扫"变贵。

**例子走查：LATERAL**：

```sql
SELECT * FROM t1 JOIN LATERAL (SELECT * FROM t2 WHERE t2.x = t1.x) d;
```

1. 解析：`m_lateral_deps` 先置伪 bit，resolver 处理外引用后变为 t1 的实际 map
2. 建路径：materialize 路径的 `parameter_tables = {t1}`（`:2298-2301`），`rematerialize = true`
3. 枚举：`{t1} HJ {d}` 被 3266 拒绝（右参数依赖左）；`{t1} NLJ {d}` 通过，NLJ 后 parameter_tables 归零
4. 执行：MaterializeIterator 每次 Init 重物化重扫——正是 `rescan_cost` 模型在优化期已计入的部分

> **向量扩展的参照意义**：原生机制 = "路径级参数声明 + join 层消化传播 + init/rescan 代价分离"。新增参数源（如 ANN 查询向量依赖外表）应仿照：建路径设 parameter_tables、join 构造处按 3266/3835 模式检查、代价按 4012/3479 模式计入 rescan。

## 六、后处理

| 函数 | 位置 | 作用 |
|------|------|------|
| `FinalizeMaterializedSubqueries` | `access_path.cc:308` | 物化 FILTER 中的子查询 |
| `FindTablesToGetRowidFor` | `:1198` | 为 HASH_JOIN/BKA/WEEDOUT/SORT 计算需要 rowid 的表 |
| `ExpandSingleFilterAccessPath` | `:1353` | 展开单个节点：equijoin 谓词、filter_predicates → 显式 FILTER |
| `ExpandFilterAccessPaths` | `:1463` | 遍历整棵树展开（FILTER 下推） |
| `FinalizePlanForQueryBlock` | `finalize_plan.cc:656` | 新优化器最终化 |

> ⚠️ 没有 `apply_some_access_paths` 这个函数（全库无匹配），对应的是 `ExpandFilterAccessPaths` + `FinalizePlanForQueryBlock`。

---

## 七、EXPLAIN 输出从哪来

**不在 `access_path.cc`**，而在 **`sql/join_optimizer/explain_access_path.cc`**：

- `SetObjectMembers()`（`:934`）：巨型 `switch (path->type)`，生成 operation 描述
- `ExplainAccessPath()`（`:1665`）：递归构建 JSON 树
- `PrintQueryPlan()`（`:1727`）：生成最终 JSON 或树形文本

传统表格格式的 EXPLAIN 则遍历 `join->qep_tab`（`opt_explain.cc:1250 shallow_explain`），不直接走 AccessPath。

---

## 核心调用栈

```
JOIN::optimize()
 ├─ [旧] create_access_paths()              sql_executor.cc:2960
 │        └─ ConnectJoins()                 sql_executor.cc:2356（QEP_TAB → 树）
 ├─ [新] FindBestQueryPlan()                join_optimizer.cc:6392（直接产出）
 └─ Query_expression::create_access_paths() sql_union.cc:1420（集合操作层）
      └─ CreateIteratorFromAccessPath()     access_path.cc:379（→ 迭代器，见 09 篇）
```

---


## 九、扩展指南：如何新增一个 AccessPath 类型

> 以新增 `AccessPath::VECTOR_SEARCH` 为例。这是"读懂体系"到"动手扩展"的桥——新增任何物理算子都走这 6 步。

### 9.1 六步改动清单

| # | 改动点 | 位置 | 做什么 |
|---|--------|------|--------|
| 1 | **类型枚举** | `access_path.h:209-266` | 在 `enum Type` 加 `VECTOR_SEARCH` |
| 2 | **参数 union 成员** | `access_path.h:863-1217` | 加 `vector_search{...}`（对应你的 `VectorAccessPathParameters`） |
| 3 | **工厂函数** | 仿 `NewMaterializeAccessPath`（`access_path.h:1546`） | 加 `NewVectorSearchAccessPath(...)`，内部设 type/cost/行数 |
| 4 | **迭代器分派** | `access_path.cc:379` `CreateIteratorFromAccessPath` | 加 `case AccessPath::VECTOR_SEARCH:` 返回 `VectorSearchIterator` |
| 5 | **EXPLAIN 输出** | `explain_access_path.cc`（TREE/JSON）+ `opt_explain_traditional.cc`（Traditional） | 加 operation 描述与 JSON 成员 |
| 6 | **代价与参数化** | 优化器侧 | 设 `cost`/`num_output_rows`/**`parameter_tables`** → 才能进 tournament |

### 9.2 第 6 步最关键：parameter_tables

新增的算子如果**依赖外表**（如相关子查询里的查询向量来自外层行），必须正确设 `parameter_tables`（见 5.5 节），否则：

- 会被 `ProposeHashJoin` 拒绝（"Parameterizations must be resolved by nested loop"，`join_optimizer.cc:3266-3271`）
- 或被 `DisallowParameterizedJoinPath`（`:2865-2899`）判为不合法
- hash join 的 `init_once_cost` 会算成"buffer 可复用"（`:3479-3506`），代价**低估**

正确做法仿照 REF / LATERAL（`:2036-2038` / `:2298-2301`）：建路径时把依赖的外表位图写进 `parameter_tables`，让 join 层（HJ `:3360-3362` / NLJ `:3849-3851`）按 `(left|right) & ~(left|right)` 传播消化。

### 9.3 需要留意的既有约定（踩坑清单）

1. **复合迭代器不导出头文件**：`MaterializeIterator`（`composite_iterators.cc:620`）/`TemptableAggregateIterator`（`:1589`）只定义在 .cc，要经 `xxx_iterator::CreateIterator()` 构造。新增复合算子建议沿用这个习惯。
2. **profiler 双实例化**：物化类迭代器要按 `EXPLAIN ANALYZE` 与否实例化 `<IteratorProfilerImpl>` 或 `<DummyIteratorProfiler>` 两个模板版本（`composite_iterators.cc:1935-1955`），并用 `SetOverrideProfiler()` 让"物化 + 扫描"耗时合并显示。
3. **`safe_for_rowid` / `immediate_update_delete` 等标志要正确设置**：它们参与 `CompareAccessPaths`（`:4180`/`:4188`）的支配判定，设错会让路径被错误淘汰。
4. **新增类型若支持"参数化"**：要同步考虑 `ordering_state` 归零规则（`:4165-4166`）与 sort-ahead 禁用（`:4792-4794`）。
5. **旧/新优化器双路径**：若两条创建路径都要支持，旧优化器在 `sql_executor.cc`（`ConnectJoins` 系）、新优化器在 `join_optimizer.cc`（`Propose*` 系）各加一处；若只支持其中之一（如 CACHE_INVALIDATOR 仅旧优化器），要在文档写明。

### 9.4 一个最小可走的验证路径

```
1. enum 加 VECTOR_SEARCH
2. union 加参数（先只放 child + 查询向量 Item*）
3. 工厂函数（cost 先填 0，行数填估算值）
4. CreateIteratorFromAccessPath 加 case → 返回一个临时占位迭代器
5. EXPLAIN FORMAT=TREE 里能看到新节点的 operation 字符串
6. 再回填真实代价与 parameter_tables
```

先跑通 1~5（能出现在 EXPLAIN 里），再补代价——这样能隔离"结构问题"和"代价问题"。

## 参考

**论文**
- **Graefe《Volcano》(1990)** —— AccessPath 与 RowIterator 的 1:1 对应，即"查询计划 ↔ 迭代器"的经典设计

**官方文档**
- *MySQL 8.0 Reference Manual → EXPLAIN Output Format*（AccessPath 是 `EXPLAIN FORMAT=TREE` 的数据源）
- *MySQL 8.0 Reference Manual → Optimizing Queries with EXPLAIN*

