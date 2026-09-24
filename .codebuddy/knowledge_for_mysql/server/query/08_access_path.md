# 08 AccessPath：统一的物理计划树

> **一句话**：AccessPath 是优化器与执行器之间的**统一物理计划 IR**——优化器（新旧两个）只负责产出这棵树，执行器只负责消费它。它被设计成纯数据结构（`struct` ≤144 字节 / MEM_ROOT 分配 / union 参数），正是这种"计划是数据、执行是行为"的分离，让 MySQL 能在同一个执行器上并行跑两个优化器。

> **边界**：本篇讲 AccessPath **自身**——它的结构、怎么被构建、怎么被最终化、怎么翻译成 RowIterator、怎么被 EXPLAIN 打印。
> - 优化器侧的算法（join 顺序枚举、代价估算、DPhyp、CSE）见 [`07_optimize/`](07_optimize/)；
> - 迭代器**拿到行之后的行为**（每种 RowIterator 怎么读、怎么落盘）见 [`09_executor_iterator.md`](09_executor_iterator.md) 与 [`runtime/`](runtime/)；
> - 集合操作的语义背景（`m_last_distinct`、INTERSECT/EXCEPT 计数器算法）见 [`07_optimize/19_set_operation.md`](07_optimize/19_set_operation.md)；
> - hypergraph 的枚举与最终化概览视角见 [`07_optimize/physical/09_hypergraph.md`](07_optimize/physical/09_hypergraph.md) 与 [`18_hypergraph_advanced.md`](07_optimize/physical/18_hypergraph_advanced.md)。

## 目录

- [概述](#概述)
  - 是什么
  - 用途
  - 版本演进
- [理论基础](#理论基础)
  - 为什么需要一层物理 IR
  - 三个设计选择
  - 统一 IR 才可能并行跑两个优化器
- [核心实现](#核心实现)
  - 类型体系：44 种节点
  - 参数 union 与断言访问器
  - 重点类型详解
  - 代价三字段与过滤前后两套值
  - 参数化：parameter_tables 与四个谓词位图
  - 构建一：旧优化器（QEP_TAB → ConnectJoins）
  - 构建二：hypergraph（Propose 直出）
  - 构建三：集合操作层
  - 工厂函数：统一的创建协议
  - 后处理：从临时表示到可执行
  - 最终化：FinalizePlanForQueryBlock
  - 翻译成执行：AccessPath → RowIterator
- [可观测性](#可观测性)
- [Misc](#misc)
  - ★ 本机制里的工程实现技法
  - 扩展点：新增一个类型要动七处
  - 容易误读的点
  - 坑与已知缺陷
- [参考](#参考)

---

## 概述

### 是什么

AccessPath 是一棵**物理计划树**的节点类型，定义在 `sql/join_optimizer/access_path.h`。每个节点表达"这一步怎么做"（扫哪张表、走哪个索引、用什么 join 方式、要不要排序/物化/去重），而不是"用户要什么"。

整条 SQL 管线是一层层中间表示（IR）的递进，与编译器的 IR 分层同构：

| 层 | IR | 表达 |
|---|---|---|
| ① | Parse Tree（`PT_*`） | 纯语法、上下文无关 |
| ② | 逻辑查询树（`Query_block` + `Item`） | 语义完整、**未优化** |
| ③ | 优化后的逻辑树（同一批对象，状态变了） | 逻辑改写完成 |
| ④ | **AccessPath 树** | **物理计划**：选了什么索引、什么 join 方式 |

日常说的"看执行计划""EXPLAIN 看计划"，看的就是这第 ④ 层——它是优化器的最终产物，也是执行器的唯一输入。

### 用途

三个用途，按重要性：

1. **解耦优化与执行**：优化器不知道执行器怎么读行，执行器不知道计划是哪个优化器产出的。
2. **承载代价与行数估算**：每个节点带 `cost` / `init_cost` / `num_output_rows`，是优化器比价的依据，也是 EXPLAIN 展示的内容。
3. **作为后处理的操作对象**：临时表创建、Item 改写、谓词下推、条件下推引擎，全部在这棵树上做 walk 完成。

### 版本演进

| 版本 | 变化 | 为什么 |
|------|------|--------|
| 8.0.20 起 | AccessPath 逐步引入，替换原 `QEP_TAB` 链作为计划表示 | 旧结构表达不了 bushy tree（见「为什么需要」） |
| **8.0.22** | **hypergraph 优化器落地**（实验特性，`optimizer_switch` 的 `hypergraph_optimizer` 开关），AccessPath 成为新旧两个优化器的**共同产出** | 新优化器需要通用物理计划结构 |
| 8.0.22~8.0.3x | 类型持续扩充至 44 种；`EXPLAIN FORMAT=TREE` 成为默认可读格式；物化/窗口/CTE 等复合节点陆续归入 | 把原本散在 `JOIN`/`QEP_TAB` 上的执行策略收敛成显式节点 |
| 8.0.39（本库基线） | 44 种类型、`CreateIteratorFromAccessPath` 显式栈翻译、`FinalizePlanForQueryBlock` 一次性的计划最终化 | —— |
| 8.4 / 9.x | AccessPath 已是唯一的计划表示（旧优化器也产出它）；hypergraph 仍非默认 | 结构稳定，演进主要在优化器侧而非 IR 侧 |

> 演进的关键转折是 **8.0.22**：此前 AccessPath 只是"旧优化器换个写法"，此后它成为**两个优化器的公共契约**——这也是它必须设计成纯数据结构的原因。

---

## 理论基础

### 为什么需要一层物理 IR

旧的计划表示是 `QEP_TAB` 数组 + `POSITION` 数组，`JOIN` 对象在上面做搜索。它有三个无法绕开的局限：

1. **只能表达左深树**。`QEP_TAB` 是一条链，天然是"((t1 ⋈ t2) ⋈ t3) ⋈ t4" 的形状。而 bushy tree（`(t1 ⋈ t2) ⋈ (t3 ⋈ t4)`）在多表 join 中常常更优，旧结构**表达不出来**——不是算法不想选，是数据结构装不下。
2. **计划表示与执行状态混在一起**。`QEP_TAB` 既有"计划"（访问方法、索引）又有"执行态"（读到的行、filesort 状态），优化器改写它时容易踩到执行态。
3. **搜索与构造耦合**。`POSITION` 数组是"可变黑板"，优化器在它上面做贪心/穷举，搜索完再翻译成计划；多方改写同一个数组，难以做"探索性候选"（建了又丢弃）。

新设计把这三点都解开：**计划是不可变的值对象、与执行态彻底分离、可以廉价地建了又丢**。

### 三个设计选择

| 选择 | 做法 | 换来什么 | 代价 |
|---|---|---|---|
| **纯数据结构** | `struct`，可平凡析构，≤144 字节，不带任何虚函数/行为 | 优化器与执行器只通过结构定义通信；可 memcpy、可栈上构造 | 类型相关的行为必须外置（工厂、switch） |
| **MEM_ROOT 分配** | `new (thd->mem_root) AccessPath`，从不 `delete` | 分配极廉价、查询结束统一回收；探索性候选零成本丢弃 | 生命周期绑到查询，不能跨查询持有 |
| **union + 类型标签 + 断言访问器** | `enum Type` + 匿名 `union` + `table_scan()` 之类带 `assert(type == ...)` 的访问器 | 44 种类型共用一个结构体体积不膨胀 | 取参数前必须判类型；遍历/翻译要写 44 分支 switch |

★ 第三条的代价是**实打实的**：`WalkAccessPaths` 与 `CreateIteratorFromAccessPath` 各有一个 44 分支的手写 switch，新增类型漏改就是静默 bug（见「扩展点」）。

### 统一 IR 才可能并行跑两个优化器

这是整件事最深层的动机：**先有统一 IR，才能并行跑两个优化器**。

```
旧优化器：POSITION/QEP_TAB 搜索 ──create_access_paths() 翻译──┐
                                                              ├──▶ AccessPath 树 ──▶ RowIterator 树
新优化器：DPhyp 枚举，Propose 时直接产出 ─────────────────────┘
```

如果计划表示是 `QEP_TAB`，hypergraph 就必须先产出 `QEP_TAB`、再由执行器翻译——等于逼新优化器套旧模子。有了 AccessPath，两个优化器**各自产出同一结构**，执行器只认结构不认来源，`optimizer_switch` 一行切换即可对比两个优化器的计划。

这也解释了为什么 AccessPath 上会同时存在两套"风格不同"的字段：`Item` 树风格的 `condition`（旧优化器用）与位图风格的 `filter_predicates`（hypergraph 用）——它们服务于两个产出方，最终在后处理里被统一（见「后处理」）。

---

## 核心实现

### 类型体系：44 种节点

`enum Type : uint8_t`，按语义分五类：

| 分类 | 数量 | 类型 |
|---|---|---|
| **基础表访问** | 17 | `TABLE_SCAN, INDEX_SCAN, REF, REF_OR_NULL, EQ_REF, PUSHED_JOIN_REF, FULL_TEXT_SEARCH, CONST_TABLE, MRR, FOLLOW_TAIL, INDEX_RANGE_SCAN, INDEX_MERGE, ROWID_INTERSECTION, ROWID_UNION, INDEX_SKIP_SCAN, GROUP_INDEX_SKIP_SCAN, DYNAMIC_INDEX_RANGE_SCAN` |
| **非特定表** | 6 | `TABLE_VALUE_CONSTRUCTOR, FAKE_SINGLE_ROW, ZERO_ROWS, ZERO_ROWS_AGGREGATED, MATERIALIZED_TABLE_FUNCTION, UNQUALIFIED_COUNT` |
| **连接** | 4 | `NESTED_LOOP_JOIN, NESTED_LOOP_SEMIJOIN_WITH_DUPLICATE_REMOVAL, BKA_JOIN, HASH_JOIN` |
| **复合** | 15 | `FILTER, SORT, AGGREGATE, TEMPTABLE_AGGREGATE, LIMIT_OFFSET, STREAM, MATERIALIZE, MATERIALIZE_INFORMATION_SCHEMA_TABLE, APPEND, WINDOW, WEEDOUT, REMOVE_DUPLICATES, REMOVE_DUPLICATES_ON_INDEX, ALTERNATIVE, CACHE_INVALIDATOR` |
| **修改表** | 2 | `DELETE_ROWS, UPDATE_ROWS` |

**关键成员**（与类型无关，每个节点都有）：

| 成员 | 说明 |
|------|------|
| `enum Type : uint8_t` | 节点类型 |
| `enum Safety` | `SAFE / SAFE_IF_SCANNED_ONCE / UNSAFE`：取 rowid 是否安全 |
| `RowIterator *iterator` | **反向指针**：指向已实例化的迭代器（见「翻译成执行」） |
| `double cost` / `init_cost` / `init_once_cost` | 三个代价字段（见下节） |
| `num_output_rows()` | 期望输出行数（`kUnknownRowCount = -1.0`） |
| `union {...} u` | 类型专用参数 |
| `count_examined_rows` | 本节点读的行是否计入 `EXPLAIN ANALYZE` 的 rows examined |

### 参数 union 与断言访问器

| 类型 | union 成员 | 关键字段 |
|------|-----------|---------|
| `table_scan` | `TableScanParameters` | `TABLE *table` |
| `index_range_scan` | `IndexRangeScanParameters` | `used_key_part, ranges, num_ranges, index` |
| `nested_loop_join` | `NestedLoopJoinParameters` | `outer, inner, join_type, join_predicate` |
| `hash_join` | `HashJoinParameters` | `outer, inner, join_predicate, allow_spill_to_disk` |
| `filter` | `FilterParameters` | `child, condition` |
| `sort` | `SortParameters` | `child, filesort, order, limit` |
| `aggregate` | `AggregateParameters` | `child, rollup` |

访问器模式：

```cpp
// 每个类型有带断言的访问器，取参数时先校验类型标签
path->table_scan().table = table;      // 内含 assert(type == TABLE_SCAN)
```

这是"union + 类型标签 + 断言访问器"的经典 C++ 手法：空间上是 union（所有类型共享一段内存），语义上靠标签区分，debug 构建用 assert 兜住误取。

### 重点类型详解

挑复杂、易踩坑的七个讲透。

#### MATERIALIZE（最重要）

参数：`table_path`（读回物化表的路径，只允许 `TABLE_SCAN`/`REF`/`EQ_REF`/`ALTERNATIVE`/`INDEX_SCAN`）、`param`（`MaterializePathParameters*`）、`subquery_cost`。

`MaterializePathParameters`：`query_blocks[]`（每个物化块）、`invalidators`、`rematerialize`、`limit_rows`、`reject_multiple_rows`、`ref_slice`。

**迭代器藏在 `.cc` 里**：`MaterializeIterator` 定义在 `composite_iterators.cc`，**不导出到任何头文件**——只能经 `materialize_iterator::CreateIterator()` 构造。`TemptableAggregateIterator` 同理。8.0 的封装习惯：复合迭代器不暴露结构。

**Init 流程**：建表 → 共享 CTE 复用判断 → 查 invalidators 的 generation 是否变化决定要不要 rematerialize → `ha_delete_all_rows` → 递归走 `MaterializeRecursive()` 否则逐 block 物化 → 更新 generation → 初始化 table_iterator。

**工厂里两处"静默改写"**（易踩坑）：

1. `rematerialize=true` 时 `invalidators` 被**强制置 nullptr**，并把 `safe_for_rowid` 设为 `SAFE_IF_SCANNED_ONCE`——每次都重物化，失效机制失去意义。
2. 对 INTERSECT/EXCEPT，`limit_rows` 被**强制成 `HA_POS_ERROR`**——去重由 TableScanIterator 完成，LIMIT 不能下推。

#### CACHE_INVALIDATOR

参数：`child`、`name`（仅用于 EXPLAIN，取表别名）。迭代器每次 `Init()`/`Read()`/`SetNullRowFlag()` 都 `++m_generation`，物化方比对 generation 判断依赖是否变化。

**两条硬约束**：

1. **创建顺序**：invalidator 必须在物化路径**左侧**先建好迭代器——`MaterializeIterator` 构造函数里 `assert(invalidator_path->iterator != nullptr)`。
2. **仅旧优化器创建**（`NewInvalidatorAccessPathForTable()`，用于 LATERAL）。**hypergraph 优化器从不创建 CACHE_INVALIDATOR**——LATERAL 改用 `rematerialize` + `parameter_tables`，hash join 用 `join->hash_table_generation` 代替（源码 TODO 明确说明这是旧优化器 invalidator 的替代方案）。

#### ALTERNATIVE（优化期建、执行期选）

参数：`table_scan_path`、`child`（ref 路径）、`used_ref`。迭代器在 **`Init()` 时二选一**，之后 `Read()` 纯转发。

**为什么需要它**：`<expr> IN (SELECT...)` 被改写成 ref 访问后，若 `<expr>` 求值为 NULL 则改写**非法**，需退回全表扫描——靠 `used_ref->cond_guards[]` 判定。

**陷阱**：切换分支时必须 `ha_index_or_rnd_end()` 重置 handler（用 `m_last_iterator_inited` 记录），否则上一个迭代器的索引/扫描状态会残留。

#### WINDOW

参数：`child`、`window`、`temp_table`、`needs_buffering`。迭代器分流式（`WindowIterator`）与缓冲（`BufferingWindowIterator`）两种。

两个细节：① `temp_table` 在工厂里被置 nullptr，真正的临时表要到**最终化阶段**才建（见「最终化」）；② 窗口路径后面挂物化时必须 `copy_items=false`——拷贝已由窗口迭代器完成，再拷一次会重复求值。

#### STREAM（"不落盘的 MATERIALIZE"）

参数类似 MATERIALIZE。当优化器以为要物化（read_set/上游 Item 都按临时表布局设好了），实际只需单趟顺序扫时，用它替换 MATERIALIZE：纯转发 + `copy_funcs`。

**它会伪造 rowid**：`provide_rowid` 为真时把自增 `m_row_number` memcpy 进 `table()->file->ref`——这样上层 WEEDOUT 仍能拿到"rowid"。该字段由 `FindTablesToGetRowidFor` 在拿到 weedout 父节点后才置位。

#### 去重三兄弟

| 类型 | 参数 | 语义 | 陷阱 |
|------|------|------|------|
| **WEEDOUT** | `SJ_TMP_TABLE*` | semi-join 的"inner join + 事后按外层 rowid 去重" | `is_confluent`（全字段去重）被直接退化成 **LIMIT 1**；工厂里 `tables_to_get_rowid_for` 置 0 并注明"必须由调用方处理" |
| **REMOVE_DUPLICATES** | `group_items` | 删**相邻**重复行 | **输入必须已按分组有序**（只删相邻）；`semijoin_group_size == 0` 时退化成 LIMIT 1 |
| **REMOVE_DUPLICATES_ON_INDEX** | — | LooseScan 的索引级去重 | 头文件明确"**只在非 hypergraph 优化器里使用**"，hypergraph 用 `REMOVE_DUPLICATES` + 排序表达同样语义 |

#### APPEND（UNION ALL 的串联）

参数：`Mem_root_array<AppendPathParameters>* children`。工厂构造时就把 cost/init_cost/init_once_cost/行数**按子节点求和**。

迭代器顺序串联：前一个 EOF 后才 `Init` 下一个，`Read()` 用**尾递归**跳到下一个。

**陷阱**：切换子迭代器前后必须显式 `EndPSIBatchModeIfStarted()` / 重新 `StartPSIBatchMode()`——**PFS batch 模式不能跨迭代器边界延续**。

### 代价三字段与过滤前后两套值

> 这是读 `EXPLAIN ANALYZE` 与 optimizer trace 时最容易混淆的地方。

| 字段 | 源码定义 | 含义 |
|---|---|---|
| `cost` | `Expected cost to read all of this access path once; -1.0 for unknown` | 把整个路径**从头到尾读一遍**的期望代价 |
| `init_cost` | `cost to read k out of N rows would be init_cost + (k/N) * (cost - init_cost)` | **读第一行**的代价（初始化代价） |
| `init_once_cost` | `Of init_cost, how much of the initialization needs only to be done once per query block` | `init_cost` 里"每 query block 只付一次"的那部分 |

**★ EXPLAIN 打印的是 `init_cost`，不是 `cost`。** 源码注释专门解释：打印"读第一行"的代价对用户更直观、也更容易在 `EXPLAIN ANALYZE` 里测量。也就是说，你在 EXPLAIN 里看到的 cost 与内部比价用的 `cost` 不是同一个字段——这是**展示层与内部层的差异**。

`init_once_cost` 的两个边界（注释明说）：① **不用于建模缓存效应**；② **不打印在 EXPLAIN 里，只在 optimizer trace**。

**`rescan_cost` 为什么反过来存**：

```cpp
double rescan_cost() const { return cost - init_once_cost; }
```

真正有意义的指标是"第二次及以后扫的代价"，但代码存的是 `init_once_cost`。理由（注释原文）：绝大多数节点的 `init_once_cost` 是 0，存它而不是 `rescan_cost`，能让代码少写大量 `init_once_cost = init_cost` 的赋值——**"存罕见值、算常见值"**的空间/代码优化。

**过滤前后的两套值**：带 filter 的节点同时记 `num_output_rows` / `cost`（过滤后）与 `num_output_rows_before_filter` / `cost_before_filter`（过滤前）。`init_cost` 永远只有一份（filter 的初始化代价为 0）。

三个字段的典型取值：

| 节点 | cost | init_cost | init_once_cost |
|---|---|---|---|
| 全表扫描 | 读全表 | 读第一行（≈0） | 0 |
| 物化（`rematerialize=false`） | 第一次物化 + 读 | 物化 + 读第一行 | ≈物化代价（第二次 Init 免了物化） |
| const 表 | 0 | 0 | 0 |

### 参数化：parameter_tables 与四个谓词位图

这两个是 hypergraph 在规划期操作 AccessPath 的抓手（旧优化器用 `Item` 树，不用位图）。

#### parameter_tables

```cpp
/// If nonzero, a bitmap of other tables whose joined-in rows must already be
/// loaded when rows from this access path are evaluated; that is, this
/// access path must be put on the inner side of a nested-loop join ...
```

非零表示"这个节点求值时某些表的行必须已加载"，即**必须放在 nested-loop join 的内侧**。两个来源：

1. **LATERAL 的 dependent tables**——派生表引用了左侧表的列；
2. **下推的 join 条件**（更常见）——`t1.x = t2.x` 被下推成 t1 上的 ref access，则 t2 被设进 bitmap（t1 的索引查找要用 t2.x 的值）。

★ **最实质的约束**：`parameter_tables != 0` 的节点**不能做 hash join 的右侧**（build 侧）——build 侧要独立建哈希表，不依赖外层行。要等参数内联完（`parameter_tables == 0`）。

**`RAND_TABLE_BIT` 哨兵**：本不属于 NodeMap，被借用来表示"这个节点完全不可 cache"（依赖非确定性函数或外层引用），因此**永远不能做 hash join 右侧**。源码注释："Make sure the table function is never hashed, ever."

**与 CACHE_INVALIDATOR 的分工**：

| 机制 | 旧优化器 | hypergraph |
|---|---|---|
| LATERAL 依赖表达 | CACHE_INVALIDATOR（generation 计数） | `parameter_tables` + `rematerialize=true` |
| 缓存失效判定 | 比对 invalidator 的 generation | 每次重物化 |

#### 四个谓词位图

存在的理由（注释原文）：位图比创建 `Item` 对象便宜得多，且优化开始时所有条件都已知，所以规划期一律操作位图。

| 字段 | 含义 |
|---|---|
| `filter_predicates` | 这个节点要应用的 WHERE 谓词（1 位 = 应用；多位则 AND） |
| `applied_sargable_join_predicates` | 已通过 ref access 应用的 sargable join 谓词（**不再重复计算选择性**） |
| `delayed_predicates` | 触达了已 join 的表、但还不能应用的（引用其它表 / 不能下推到外连接 nullable 侧） |
| `subsumed_sargable_join_predicates` | 已应用且**完全覆盖** join 谓词（谓词冗余，不再当 filter） |

分工上两组对照：`filter_predicates`（还没用掉的）vs `applied_sargable_join_predicates`（已被索引吃掉、别重复计选择性）；`delayed_predicates`（想用但用不了）vs `subsumed_...`（用了且覆盖了）。

**位图只在规划期存在**。创建迭代器之前，`ExpandFilterAccessPaths` 把它们翻译成真正的 `FILTER` 节点，摆脱对 predicates 数组的依赖（这些 FILTER 约定 `materialize_subqueries=false`，因为绝大多数谓词里没有子查询）。

**内存复用**：`filter_predicates` 与 `applied_sargable_join_predicates` 共用同一块内存（后者是前者的引用别名）；`delayed_predicates` 与 `subsumed_sargable_join_predicates` 同理。原理是两者 bit 不重叠（谓词数组 N 个：bit 0..N-1 给前者，更高位给后者）。★ 为什么不用 union？`OverflowBitset` 有非平凡默认构造函数，union 不允许——所以用"两个 accessor 返回同一成员"的别名方式，语义独立、内存共用。

### 构建一：旧优化器（QEP_TAB → ConnectJoins）

入口：

```cpp
void JOIN::create_access_paths() {
  assert(m_root_access_path == nullptr);
  AccessPath *path = create_root_access_path_for_join();     // → ConnectJoins()
  path = attach_access_paths_for_having_and_limit(path);
  path = attach_access_path_for_update_or_delete(path);
  m_root_access_path = path;
}
```

`ConnectJoins` 是整个翻译过程的灵魂。它的签名有 11 个参数，其中 **3 个**是递归传递的 `vector`（`pending_conditions`、`pending_invalidators`、`pending_join_conditions`），另两个是 `qep_tab_map*` 与 `table_map*`——参数数量本身就说明了复杂度：

```cpp
AccessPath *ConnectJoins(plan_idx upper_first_idx, plan_idx first_idx,
                         plan_idx last_idx, QEP_TAB *qep_tabs, THD *thd,
                         CallingContext calling_context,
                         vector<PendingCondition> *pending_conditions,
                         vector<PendingInvalidator> *pending_invalidators,
                         vector<PendingCondition> *pending_join_conditions,
                         qep_tab_map *unhandled_duplicates,
                         table_map *conditions_depend_on_outer_tables);
```

#### 难题①：树形——左深树 + 三种嵌套子结构

MySQL 的 join 树一般是左深树，内层循环"从左边开始一张张往右接"。但表序列里会嵌着三类子结构，每种靠 `QEP_TAB` 上的下标标记：

| 子结构 | 怎么标记 | 处理 |
|---|---|---|
| **外连接** | 切片第一张表上设 `first_inner()` / `last_inner()` | 递归处理 `[first_inner, last_inner]` 作为外连接右臂 |
| **semi-join（FirstMatch）** | 表 N 设了"first match 指向表 M" | 递归处理 `[M+1, N]` 作为 semijoin 右侧 |
| **Duplicate Weedout** | `Substructure::WEEDOUT` | 包一层去重节点 |

★ **源码注释的警示**：不能用 `first_sj_inner()` 检测 semijoin，因为**它不随 join 顺序重排而更新**（优化器会重排表，但 `first_sj_inner()` 不跟着改）。所以检测逻辑绕了一圈用 `firstmatch_return`。这类"存了下标的地方，重排后还对吗"的坑在 `JOIN::optimize` 里到处都是。

外连接与 semijoin 可以嵌套，`FindSubstructure()` 必须挑**最外层**的结构递归，`calling_context` 参数就是为防止无限递归。

#### 难题②：条件什么时候能求值

概念上 SQL 的条件在**所有表 join 完之后**求值，但为了效率希望尽早求值：

| 上下文 | 条件何时求值 |
|---|---|
| 只有内连接 | 一旦条件涉及的表都读完了，立即求值 |
| **在外连接的右臂内部** | **必须延迟**——外连接会补 NULL 行，提前求值会错 |

延迟靠 `pending_conditions` 实现。★ **`pending_conditions` 是否为 `nullptr` 就是"我现在是不是在外连接右臂里"的标志**：`nullptr` → 可以立即求值；非 `nullptr` → 必须推迟。这是"用指针的空/非空承载上下文"的写法。

递归调用时四种传法各不相同：

```
SEMIJOIN + UseHashJoin      → 传 &subtree_pending_conditions（新向量）
                              要把 join conditions 上提到本层挂到 hash join 迭代器上
SEMIJOIN + 非 hash join     → 传 pending_conditions（继承父层）
OUTER_JOIN 且已在某个外连接内侧 → 继承 pending_conditions，继续推迟 WHERE
OUTER_JOIN 且不在外连接内侧  → 传新的 &subtree_pending_conditions，本层 ON 条件可就地挂
```

递归返回后，`PickOutConditionsForTableIndex()` 把"属于这一层的 ON 条件"挑出来，挂到刚建好的 join 节点**正上方**。

#### 难题③：anti-join 特判（NOT IN）

```cpp
if (qep_tab->table()->reginfo.not_exists_optimize) {
  join_type = (pending_conditions == nullptr) ? JoinType::ANTI : JoinType::OUTER;
}
```

`not_exists_optimize` 表示"找到一条就不用再找了"。但**只有当不在另一个外连接的右臂内时，才能启用 antijoin**——否则会让上层外连接本不该产生 NULL 行的地方产生 NULL 行。

更刁的特例：`NOT IN` 的 antijoin 会在同一张表上设 found 触发器，条件被包在 `not_null_compl` 与 `found` 里，照常处理会把所有输出行杀掉。所以要专门识别 `it->cond->item_name.ptr() == antijoin_null_cond`，从 pending 里**删掉**，并用它把 join 标记成"真正的 ANTI"。

★ 这类补丁是 `ConnectJoins` 最难读的部分——它的逻辑不是"怎么建树"，而是"哪些情况下不能按常规建树"。

#### 收口与单表路径

`FinishPendingOperations()` 做两件事：把积压的条件包成 `FILTER`（`PossiblyAttachFilter` 会合并相邻 filter 并判断能否省略），以及给 LooseScan 包一层 `REMOVE_DUPLICATES_ON_INDEX`。

单表的基础访问路径由 `GetTableAccessPath()` 按 `QEP_TAB::materialize_table` 分五路：

| 分支 | 产出 | 说明 |
|---|---|---|
| `MATERIALIZE_DERIVED` | `GetAccessPathForDerivedTable()` | 派生表：内部查询包一层物化 |
| `MATERIALIZE_TABLE_FUNCTION` | `NewMaterializedTableFunctionAccessPath()` | JSON_TABLE 等 |
| `MATERIALIZE_SEMIJOIN` | ★ **递归调用 `ConnectJoins()`** | 见下 |
| 非物化 + `schema_table->fill_table` | `NewMaterializeInformationSchemaTableAccessPath()` | I_S 表必须先填充再扫 |
| 普通表 | `qep_tab->access_path()` | 即 `create_table_access_path()` 的产出 |

★ `MATERIALIZE_SEMIJOIN` 分支最值得注意：它递归调用 `ConnectJoins()`，把同一 `JOIN` 对象里的两组表**当成两个独立的 join** 来建树（源码称之为 "virtual join"）。这个递归还要处理三件附带的事：weedout 收尾（`unhandled_duplicates` 补 WEEDOUT 节点）、NULL 过滤（给每个 nullable 的 `sj_inner_exprs` 加 `Item_func_isnotnull`——**对 `IN` 只是优化，对 `NOT IN` 是正确性必需**）、代价回填（`EstimateMaterializeCost()`）。

`create_table_access_path()` 三选一：有 range 计划就用它；否则若是 `WITH RECURSIVE` 自引用就用 `FOLLOW_TAIL`（读临时表尾部而非重扫）；否则全表扫。

#### 代价怎么从 POSITION 搬到 AccessPath

```cpp
void SetCostOnTableAccessPath(const Cost_model_server &cost_model,
                              const POSITION *pos, bool is_after_filter,
                              AccessPath *path) {
  double num_rows_after_filtering = pos->rows_fetched * pos->filter_effect;
  ...
  double cost = pos->read_cost + cost_model.row_evaluate_cost(num_rows_after_filtering);
  if (pos->prefix_rowcount <= 0.0) {
    path->cost = cost;
  } else {
    // Scale the estimated cost to being for one loop only, to match the measured costs.
    path->cost = cost * num_rows_after_filtering / pos->prefix_rowcount;
  }
}
```

两个要点：

1. **`num_output_rows` 有两态**：`is_after_filter=false` 记过滤前行数，`true` 记过滤后。所以 FILTER 节点**下方**记过滤前、上方记过滤后——读 `EXPLAIN FORMAT=JSON` 的 `rows_examined_per_scan` / `rows_produced_per_join` 时要分清（这就是上一节"过滤前后两套值"的写入点）。
2. **代价被缩放到"单次循环"**：`POSITION::read_cost` 是整个前缀的累计代价，而 AccessPath 记的是"这个节点每被调用一次花多少"。缩放是为了让估算值与 `EXPLAIN ANALYZE` 实测可比较。

（另有注释坦承"we don't try to adjust for the filtering here"——过滤不减代价，是已知的粗略处理。）

#### 包裹顺序

得到 `table_path` 后、接进 join 之前，依次包若干层，顺序固定：

```
① BKA        → NewMRRAccessPath()（随机索引查找换成按 rowid 排序的顺序查找，即 DS-MRR）
② 条件下推   → PossiblyAttachFilter(predicates_below_join)
               除非 condition_is_pushed_to_sort()（条件已推到排序前面）
③ LooseScan  → NewRemoveDuplicatesOnIndexAccessPath()（仅单表 LooseScan）
④ 失效器     → NewInvalidatorAccessPathForTable()（lateral derived table 的缓存失效）
⑤ 接进 join  → NestedLoop / HashJoin / ...
```

备注：② 为真时不挂 filter，只把依赖表并入 `conditions_depend_on_outer_tables`；③ 只有 `do_loosescan() && match_tab == i` 才在这里包（多表 LooseScan 由 `NestedLoopSemiJoinWithDuplicateRemovalIterator` 处理，本质是"semijoin 的 NestedLoop + RemoveDuplicatesOnIndex 合二为一"）；④ 若被失效的表属于**更高的外连接 nest**，不能立即发 invalidator（当前外连接可能还会补 NULL 行，那些行也要失效缓存），推进 `pending_invalidators` 等回到同一 nest——注释 "But if we deal with them later than that, it might be too late!" 说明时序很敏感。

**BNL 被换成 HashJoin**：`replace_with_hash_join` 的判定是 `UseHashJoin(qep_tab) && !QueryMixesOuterBKAAndBNL(join)`。换成 hash join 时 `ExtractJoinConditions()` 把等值与非等值连接条件**全部上提**到 `join_conditions` 挂到 hash join 迭代器上，留在 `predicates_below_join` 的仍是 filter。

#### 一个完整例子

```sql
SELECT * FROM t1 LEFT JOIN t2 ON t2.a = t1.a WHERE t1.b > 10;
```

```
1. FindSubstructure 在 t2 上发现 first_inner/last_inner → OUTER_JOIN 子结构
2. 递归：[t1] 先建 TABLE_SCAN(t1)
3. 递归：[t2] 建 TABLE_SCAN(t2)，用 DIRECTLY_UNDER_OUTER_JOIN 上下文
4. ON 条件 (t2.a = t1.a) 由 PickOutConditionsForTableIndex 挑出，挂到 join 节点上
5. WHERE t1.b > 10：
   - 不在外连接右臂内 → 读 t1 后立即求值（FILTER 挂在 TABLE_SCAN(t1) 上）
   - 在外连接右臂内   → 进 pending_conditions，由 FinishPendingOperations 挂到 join 之上
```

★ 第 5 步的两条路径就是"同样的 WHERE、位置不同"的全部由来——也是 EXPLAIN 里 `Filter:` 节点位置差异的根源。

### 构建二：hypergraph（Propose 直出）

| 维度 | 旧优化器 | hypergraph |
|---|---|---|
| 中间结构 | `QEP_TAB` 链（左深树） | `RelationalExpression` / `JoinHypergraph` |
| 产出方式 | `create_access_paths()` 把 QEP_TAB **翻译**成树 | **搜索的同时直接产出 AccessPath** |
| cost 何时填 | 翻译时从 `POSITION` 搬 | **Propose 的时候就填好** |
| 谓词表达 | `Item` 树 | 位图 |

★ 最本质的差异是**"搜索 = 构造"合二为一**：旧优化器先在 `POSITION` 数组上搜索、搜完才翻译；hypergraph 的 DPhyp 是自底向上递归构造候选，**每个子问题的解就是一个 AccessPath 子树**，所以"找到一个候选"和"产出一个 AccessPath"是同一件事。

#### Propose 模式：栈上值对象 → MEM_ROOT

```cpp
bool CostingReceiver::ProposeTableScan(TABLE *table, int node_idx, ...) {
  AccessPath path;                              // ① 栈上建值对象
  path.type = AccessPath::TABLE_SCAN;           // ② 填字段（像填 struct 一样）
  path.table_scan().table = table;
  path.count_examined_rows = true;
  path.ordering_state = 0;                      // ③ interesting order 状态
  path.num_output_rows_before_filter = num_output_rows;  // ④ 代价现在就填
  path.init_cost = path.init_once_cost = 0.0;
  path.cost_before_filter = path.cost = cost;
  // 需要长期持有时才搬上 MEM_ROOT：
  AccessPath *new_path = new (m_thd->mem_root) AccessPath(path);  // ⑤ 落盘
}
```

与旧优化器工厂对比：

| | 旧优化器工厂（`New*`） | hypergraph Propose |
|---|---|---|
| 创建方式 | 直接 `new (mem_root)` + 填字段 | 先栈上值对象，需要时再拷贝落盘 |
| cost | 大多留 `-1.0` 后填 | 现在就填好 |
| 探索性 | 无（工厂即最终） | **有**——栈上对象建了又丢弃，只有被采纳才落盘 |

"栈上值对象 → 落盘"是精髓：DPhyp 探索大量候选，很多是试探性的，若每个都 `new (mem_root)` 会污染 MEM_ROOT。所以先构造、比价，只有进入 Pareto 前沿的才搬上 MEM_ROOT。

#### ProposeTableScan 逐段

- **递归 CTE**：自引用用 `FOLLOW_TAIL`，且**强制它是最左表**（`forced_leftmost_table`）。
- **代价与行数**：`num_output_rows = table->file->stats.records`，`cost = table->file->table_scan_cost().total_cost()`；全表扫 `init_cost` 为 0。
- **单表 UPDATE/DELETE 就地修改**：`immediate_update_delete_table` 在 Propose 阶段就预判——**若表扫走聚簇键且修改了主键，就不能就地改**（会破坏扫描顺序），置 -1。
- **I_S 表**：包一层 `MATERIALIZE_INFORMATION_SCHEMA_TABLE`，并把内层的 `filter_predicates` / `delayed_predicates` **上移**到物化层（谓词要在物化后应用）；代价标 `// Rudimentary`（作者自承粗略）。
- **派生表 / 表函数**：包 MATERIALIZE，并写入 `parameter_tables`——表函数的 `used_tables()` 转 NodeMap；若依赖外层引用或非确定性（`OUTER_REF_TABLE_BIT | RAND_TABLE_BIT`），额外置 `RAND_TABLE_BIT`（"never hashed, ever"）。

#### ProposeRefAccess 逐段

逐个 keypart 匹配 sargable 谓词，两个关键点：

1. **前缀匹配规则**：`if (!matched_this_keypart) break;` —— 一旦某个 keypart 匹配不上，后面全放弃。这是索引最左前缀规则的体现。
2. **`parameter_tables` 的来源**：`parameter_tables |= other_side_tables` —— ref access 依赖的"等式另一边"的表就是 `parameter_tables`。这是"下推的 join 条件"这一来源的源码实锤。

剪枝：`if (parameter_tables != allowed_parameter_tables) return false;` —— 相同的 `parameter_tables` 只保留第一次（等价候选不重复提议）。

每个匹配的 keypart 记五元组 `KeypartForRef`：`field`、`condition`、`val`、`null_rejecting`、`used_tables`——后续 `NewRefAccessPath` 的原料。

> `FoundJoin` 如何把多个 Propose 出的单表 path 组合成 join 节点、Pareto 前沿如何淘汰候选，见 [`07_optimize/physical/09_hypergraph.md`](07_optimize/physical/09_hypergraph.md)。

### 构建三：集合操作层

集合操作在 `Query_expression`（unit）层面构建，与 query block 层面（`ConnectJoins`）是两条平行的构建路径。

```cpp
void Query_expression::create_access_paths(THD *thd) {
  if (is_simple()) {                       // ① 单查询块：直接复用 JOIN 的树
    m_root_access_path = first_query_block()->join->root_access_path();
    return;
  }
  ...
  if (union_all_sub_paths->size() == 1) {  // ② 只有一个子 path：直接取用
    m_root_access_path = (*union_all_sub_paths)[0].path;
  } else {                                 // 多个：APPEND 串联
    assert(streaming_allowed);
    m_root_access_path = NewAppendAccessPath(thd, union_all_sub_paths);
  }
  ...
}
```

**LIMIT/OFFSET 何时直接用节点**：

```cpp
// NOTE: If there's a fake_query_block, its JOIN's iterator already handles
// LIMIT/OFFSET, so we don't do it again here.
if (streaming_allowed && (limit != HA_POS_ERROR || offset != 0) &&
    (is_simple() || set_operation()->m_last_distinct == 0)) {
  m_root_access_path = NewLimitOffsetAccessPath(...);
}
```

判据两条：① `fake_query_block` 的 JOIN 已处理过，不重复；② `is_simple() || m_last_distinct == 0`——**只有全部是 UNION ALL（无去重 operand）时 LIMIT 才能直接截断**，一旦有去重必须先物化去重完再截断（否则截掉需要参与去重的行）。

**`streaming_allowed` 的两个否定条件**：

| 条件 | 触发 | 原因 |
|---|---|---|
| **A** | 有排序，或整个 set op 被要求物化（`!is_simple() && m_is_materialized`） | 注释自陈是**遗留决定**：排序总落真实表，因为历史上不确定 filesort 是否要在排序后按 row ID 引用表行（`We could probably be more intelligent here now`） |
| **B** | 顶层 INSERT/REPLACE SELECT | Halloween Problem：边扫边插会读到自己刚插入的行，导致错误结果甚至死循环（TODO 指向 bug #23022426，建议改用 `OPTION_BUFFER_RESULT` 判断） |

**混合 UNION 的两种策略**：

| 场景 | `union_all_sub_paths` 传入 | 结果 |
|---|---|---|
| 不流式 | `nullptr` | 整棵树走物化；返回单一物化 path 手动 push 进数组 |
| 流式但非简单 | 非空数组 | `make_set_op_access_path` 往里填：物化部分（UNION DISTINCT 块）+ 流式 UNION ALL 块 |

★ **"流式"的本质是"UNION DISTINCT 块先物化、UNION ALL 块用 AppendIterator 追加"**——不是完全不物化，而是"去重部分物化、追加部分不物化"。

`make_set_op_access_path` 递归遍历 `Query_term` 树：叶子（`Query_block`）用其 `root_access_path()`；`UNION` 走去重物化 + ALL 进数组；`INTERSECT`/`EXCEPT` 走物化 + 计数器列。`setup_materialize_set_op()` 决定哪些 operand 要去重：`idx <= m_last_distinct`（UNION 的前若干 operand）或 `term_type() != QT_UNION`（**INTERSECT/EXCEPT 永远激活去重**）。

### 工厂函数：统一的创建协议

`New*` 工厂族（`NewTableScanAccessPath`、`NewRefAccessPath`…）是 AccessPath 唯一的创建入口，模式完全统一：

```cpp
inline AccessPath *NewTableScanAccessPath(THD *thd, TABLE *table,
                                          bool count_examined_rows) {
  AccessPath *path = new (thd->mem_root) AccessPath;   // ① placement new 到 MEM_ROOT
  path->type = AccessPath::TABLE_SCAN;                  // ② 设类型标签
  path->count_examined_rows = count_examined_rows;      // ③ 通用字段
  path->table_scan().table = table;                     // ④ 通过断言访问器填 union
  return path;
}
```

共同特征：全是 `inline`（定义在 `.h`，创建节点零函数调用开销）；大部分**不填 cost**（默认 `-1.0`，由后续估算填）；`count_examined_rows` 控制该节点读的行是否计入 `EXPLAIN ANALYZE` 的 rows examined。

三个特殊工厂：

- **`NewConstTableAccessPath`**：唯一在工厂里填全 cost 的（输出 1 行、三个代价全 0）——const 表在优化期就求值了，执行期确实零代价。
- **`NewMRRAccessPath`**：`mrr().bka_path` 在工厂里置 nullptr，注释明说"will be filled in when the BKA iterator is created"——MRR 节点在 BKA join 里要指向外表路径，这个指针只能在 BKA 迭代器创建时才知道。
- **`NewFollowTailAccessPath`**：`WITH RECURSIVE` 自引用读临时表尾部。

### 后处理：从临时表示到可执行

建树之后、执行之前，还有一层"后处理"：把优化期的临时表示（谓词位图、`nullptr` 的临时表）翻译成可执行的最终形态。

#### WalkAccessPaths：所有后处理的公共底座

```cpp
template <class AccessPathPtr, class Func, class JoinPtr>
void WalkAccessPaths(AccessPathPtr path, JoinPtr join,
                     WalkAccessPathPolicy cross_query_blocks, Func &&func,
                     bool post_order_traversal = false);
```

★ **为什么用模板而不是虚函数/函数指针**：回调会被**内联**进遍历里。后处理遍历（EXPLAIN、找 rowid、收 filesort…）都在热路径上，模板避免了每次回调的间接跳转。

三种策略：

```cpp
enum class WalkAccessPathPolicy {
  STOP_AT_MATERIALIZATION,   // 不跨 MATERIALIZE/STREAM（留在当前 query block）
  ENTIRE_QUERY_BLOCK,        // 跨整个 query block（需要 join 参数跟踪）
  ENTIRE_TREE                // 跨整棵树
};
```

- **pre-order**（默认）：先调 `func(path)`，返回 `true` 则**不下降**到子节点（剪枝）
- **post-order**：先遍历子节点再调 `func`（此时"跳过"已太迟，返回值被忽略）

**`join` 参数的必要性**：AccessPath 本身不含"属于哪个 query block"的信息，但 `MATERIALIZE`/`STREAM` 会切换 query block（物化的子查询是另一个 block）。`WalkAccessPaths` 在遍历中跟踪这个变化，给回调传正确的 `JOIN*`。

★ **代价**：每种类型的 child 结构不同，遍历必须写一个 **44 分支的 switch**——新增类型漏改就遍历不到子节点（见「扩展点」）。

#### 两个代表性后处理

| 后处理 | 做什么 |
|---|---|
| `ExpandFilterAccessPaths` | 把 `filter_predicates` / `delayed_predicates` 位图翻译成真正的 `FILTER` 节点，摆脱对 predicates 数组的依赖 |
| `FinalizePlanForQueryBlock` | 建临时表、建 Filesort、把 Item 改写指向临时表、`push_to_engines()`——**只能调一次**（见下节） |

### 最终化：FinalizePlanForQueryBlock

`sql/join_optimizer/finalize_plan.cc`。这是 hypergraph 路径的"计划收尾"，也是**只能调用一次的破坏性操作**。

#### 四个前置分支

```cpp
bool FinalizePlanForQueryBlock(THD *thd, Query_block *query_block) {
  assert(query_block->join->needs_finalize);
  query_block->join->needs_finalize = false;          // ① 幂等保护
  AccessPath *const root_path = ...;
  if (root_path->type == AccessPath::EQ_REF) return false;   // ② 点查短路
  if (!IteratorsAreNeeded(thd, root_path)) return false;     // ③ 二级引擎短路
  thd->lex->set_current_query_block(query_block);     // ④ 切换当前 block
```

| 分支 | 说明 |
|---|---|
| ① `needs_finalize` 置 false | **幂等保护**：assert + 立即清标志，重复调用在 debug 构建会崩（这就是"只能调一次"的运行期体现） |
| ② `EQ_REF` 点查短路 | 单行 eq_ref 不需要临时表/聚合器/Item 改写 |
| ③ `IteratorsAreNeeded` 短路 | 二级引擎（外部执行器）不需要内部临时表、filesort、Item 改写——**最终化的一切产物都是给"内部执行"用的** |
| ④ 切换 `current_query_block` | 后续建临时表、改写 Item 都要工作在目标 block 上下文（THD 的很多操作依赖它） |

#### 第一步：合并相邻 FILTER

post-order 遍历，把叠放的 FILTER 合并成一个。三个要点：

1. **合并条件**：child 也是 FILTER **且**两者 `materialize_subqueries` 一致——`true` 的 FILTER 执行时可能触发子查询物化，与 `false` 合并会改变子查询求值时机。
2. **新建 Item 的"三步必修课"**：`quick_fix_field()` + `update_used_tables()` + **`apply_is_true()`**。比 CSE 那边的"两步"多一步——`apply_is_true()` 处理 WHERE 语义对未知值的 truthiness 要求（如 `AND` 遇 NULL 的行为）。**漏掉这一步会导致 NULL 语义错误**。
3. **post-order**：从叶子往根合并，保证连续叠放的 FILTER 被逐层吃掉。

#### 第二步：主遍历（post-order）

每到一个节点做四件事，顺序敏感：

**① `DelayedCreateTemporaryTable`——延迟建临时表**。物化节点的 `temp_table` 在建树时是 `nullptr`，这里才真正建 `TABLE`。"Delayed"的含义：建表需要知道"输入行有哪些列"（尤其聚合后列集变了），所以要等**子树处理完**——这正是用 post-order 的原因。三个状态变量跨节点传递：`last_window_temp_table` / `num_windows_seen`（多窗口共享/区分临时表）、`after_aggregation`（越过 AGGREGATE 后，后续节点面对的是聚合后的列集）。

**② `UpdateReferencesToMaterializedItems`——Item 改写**。把 `Item_field`（指向基表列）替换成指向临时表列，替换规则累积在 `applied_replacements` 里，供 AGGREGATE 与窗口分支使用。

**③ WINDOW 分支**：`FinalizeWindowPath()` 用 `original_fields`（进入该节点前的 fields 快照）+ 累积的 replacements 最终化窗口路径。

**④ `AddCachesAroundConstantConditionsInPath`**：在常量条件周围包 `Cached_item`（如 `WHERE b = const` 的 `b` 不必每行重求值）。每个节点都调，因为常量条件可能在任何位置。

#### AGGREGATE 分支的三条要害

```cpp
} else if (path->type == AccessPath::AGGREGATE) {
  for (Cached_item &ci : join->group_fields) { ... thd->change_item_tree(...) ... }
  // Set up aggregators, now that fields point into the right temporary table.
  const bool need_distinct = true;  // We don't support loose index scan yet.
  ...
  after_aggregation = true;
}
```

1. **用 `change_item_tree` 而不是直接赋值**：支持 PS 复用时回滚；`need_exact_match=true` 要求精确匹配（分组键错配是正确性问题）。
2. ★ **"We don't support loose index scan yet"——hypergraph 不支持 LIS**：`need_distinct` 硬编码 `true`，带 `DISTINCT` 的聚合一律走 `DISTINCT_AGGREGATOR`。经典优化器里 `GROUP_INDEX_SKIP_SCAN` 可以走"预计算分组"（`precomputed_group_by`），hypergraph 不行——这是两者计划性能差异的已知来源。
3. **`set_aggregator` 必须在 Item 改写之后**：聚合器要初始化"往哪里聚合"，而"哪里"是临时表列。顺序是：建临时表 → 改写 Item → **最后**设聚合器。这个顺序由 post-order + `after_aggregation` 状态机保证。

#### 完整调用栈

```
Query_block::optimize()（hypergraph 分支）
  └─ FindBestQueryPlan() → EnumerateAllConnectedPartitions()  产出带位图的 AccessPath 树
  └─ ExpandFilterAccessPaths()                    位图 → FILTER
  └─ FinalizePlanForQueryBlock()                  最终化
       ├─ 前置：needs_finalize / EQ_REF 短路 / IteratorsAreNeeded
       ├─ WalkAccessPaths(post-order)：合并相邻 FILTER
       ├─ WalkAccessPaths(post-order)：延迟建表 / Item 改写 / WINDOW / AGGREGATE / 常量缓存
       └─ push_to_engines()
  └─ CreateIteratorFromAccessPath()               翻译成迭代器（见下）
```

### 翻译成执行：AccessPath → RowIterator

`CreateIteratorFromAccessPath`（`sql/join_optimizer/access_path.cc`）是计划与执行之间的唯一桥梁。

| AccessPath | RowIterator |
|---|---|
| `TABLE_SCAN` | `TableScanIterator` |
| `REF` / `REF_OR_NULL` / `EQ_REF` / `PUSHED_JOIN_REF` | `RefIterator` / `RefOrNullIterator` / `EQRefIterator` / `PushedJoinRefIterator` |
| `INDEX_RANGE_SCAN` | `IndexRangeScanIterator` |
| `NESTED_LOOP_JOIN` | `NestedLoopIterator` |
| `HASH_JOIN` | `HashJoinIterator` |
| `SORT` / `FILTER` / `MATERIALIZE` | `SortingIterator` / `FilterIterator` / `MaterializeIterator` |

> 每种迭代器**拿到行之后的行为**见 [`09_executor_iterator.md`](09_executor_iterator.md)，本篇只讲翻译过程本身。

#### 为什么用显式栈而不是递归

源码注释：

```
The access path trees can be pretty deep, and the stack frames can be big
on certain compilers/setups, so instead of explicit recursion, we push jobs
onto a MEM_ROOT-backed stack. This uses a little more RAM ..., but reduces
the stack usage greatly.
```

**"用内存换栈空间"**：树深 × 单帧大小可能超过线程栈（MySQL 默认 256KB），深查询（几十层物化嵌套）递归翻译会栈溢出。

#### 两阶段：先推子、再重推自己

```cpp
unique_ptr_destroy_only<RowIterator> ret;
Mem_root_array<IteratorToBeCreated> todo(mem_root);
todo.push_back({top_path, top_join, top_eligible_for_batch_mode, &ret, {}});
while (!todo.empty()) {
  IteratorToBeCreated job = todo.back(); todo.pop_back();
  switch (path->type) { ... }
}
```

有子节点的迭代器走两阶段：

```
第一次弹出自己 → 发现还没建子节点 → 把各子节点 job 推入栈尾 → 把自己重新压栈
第二次弹出自己 → 子节点都已建好（在 children 里）→ 真正 new 出自己
```

★ 区分"第一次还是第二次"的手段：**看 `job.children` 是否已分配**。另一个细节：`children` 数组直接分配在 MEM_ROOT 上而不是 `todo` 栈里——`todo` 的 `push_back` 会 realloc（移动元素），而子迭代器指针要稳定指向这个数组。

#### NewIterator 模板与 examined_rows 绑定

`NewIterator<T>` 是统一工厂：`new (mem_root) T(...)` 并包成 `unique_ptr_destroy_only`（析构时不 delete，MEM_ROOT 统一回收）。

```cpp
ha_rows *examined_rows = nullptr;
if (path->count_examined_rows && join != nullptr) examined_rows = &join->examined_rows;
```

这就是 `count_examined_rows` 的消费点：迭代器每读一行 `++*examined_rows`——`EXPLAIN ANALYZE` 的 "rows examined" 就是这么累计的。

#### ★ PFS batch mode 是"监控埋点的批量"，不是"计算的批量"

它把 N 次 `start/end_table_io_wait` 计时合并成 1 个事件 + 一个行数计数，**与向量化/一次取多行毫无关系**——`RowIterator::Read()` 的契约始终是 `Read a single row`。把它写成"批处理执行"是常见错误。

三条传递规则（源码注释明文）：

| 规则 | 迭代器 | 行为 |
|---|---|---|
| ① 单子转发 | `FilterIterator`、`LimitOffsetIterator`、`SubqueryIterator`、**`TimingIterator`** | 直接转发给子（TimingIterator 转发意味着 EXPLAIN ANALYZE 的包装层不阻断传递） |
| ③ 多子忽略 + 自行处理 | `NestedLoopIterator` | 不重写 `StartPSIBatchMode()`（基类空实现 = 忽略），只在 `Init()`/读完后**给 inner 单独开关** |
| ③ 变体 | `AppendIterator` | 自己记住状态，转给"当前"那个子 |

**为什么只在最内层开**（注释）：收益最大、精度损失最不关键。老优化器用 `QEP_TAB::pfs_batch_update` 判定（非最内层 / `eq_ref` / `const` / 带子查询的条件 → 不开）；hypergraph 侧有对应的 `ShouldEnableBatchMode()`（注释说它 mirror 前者，多一条：多于一张表时由 join 迭代器在 probe 侧处理）。

**翻译期的资格传播**：`eligible_for_batch_mode` 沿 `IteratorToBeCreated` 往下传，双子节点时**只传给 inner**：

```cpp
todo->push_back({inner, join, inner_eligible_for_batch_mode, &job->children[1], {}});
todo->push_back({outer, join, false, &job->children[0], {}});   // outer 传 false
```

**计数准确性**：`examined_rows` 不受影响（迭代器自己累加）；PFS 侧行数也准（汇总后一次性报出），**变粗的只有事件粒度与时间分摊**。

#### 翻译的边界：什么被延迟，什么没有

> ⚠️ **纠正一个常见误传**："MATERIALIZE 的子查询迭代器在 `Init()` 时才惰性创建"——**源码反证这是错的**。MATERIALIZE 的每个 query block 子迭代器在翻译期就建好了（`job.children[i+1]`），`Init()` 里只是 `Init()` 它们。被延迟的是**物化动作**，不是迭代器构造。

真正被延迟的是三处：

1. **`Query_expression` 的根迭代器**（阶段级）：`Query_expression::optimize()` 有显式开关 `create_iterators`——派生表/视图只建 access path 不建 iterator（`unit->optimize(thd, table, /*create_iterators=*/false, ...)`），需要时再 `force_create_iterators()`。★ 为什么必须推迟：`If you use this function, make sure it's not called at prepare. Due to evaluation of LIMIT clause it can not be used at prepared stage.` 另外 const 表在 hypergraph 下必须推迟到执行期物化（"it will get confused and crash if it has already been materialized"）。
2. **`SortingIterator` 的结果迭代器**（类型未定型）：构造前不知道排序结果是否落盘、是否用 packed addons → **类型未定** → 只能在 `Init()` 里定。这是全库最硬的"延迟创建"证据。
3. **临时表 instantiate**：第一次 `Init()` 时才建表。

| | 两阶段（先子后父） | 惰性（推迟到执行期） |
|---|---|---|
| 作用域 | **同一次** `CreateIteratorFromAccessPath` 调用内部 | **跨阶段**：optimize → execute / 首次 `Init()` |
| 解决的问题 | 父迭代器构造需要子迭代器已存在（构造依赖） | 信息不足或阶段不允许：类型未定、表未 instantiate、LIMIT 不能 prepare 期求值 |

同理，**递归 CTE 也不是"延迟创建迭代器"**——递归引用是建好之后从树里找回来的（`FindSingleIteratorOfType(...FOLLOW_TAIL)`）。真正需要"边写边读"的是**表**，不是迭代器。

#### 关键 switch 分支要点

| 类型 | 要点 |
|---|---|
| `TABLE_SCAN` / `INDEX_SCAN` / `REF` | 按 `reverse` 选模板参数 |
| `EQ_REF` | **不传 expected_rows**（至多一行，无需 record buffer） |
| `NESTED_LOOP_JOIN` | `SetupJobsForChildren(outer, inner)` |
| `BKA_JOIN` | **特殊**：压子 job **之前**先设 `mrr_path->mrr().bka_path = path`，再从 `mrr_path->iterator->real_iterator()` 拿 `MultiRangeRowIterator*` |
| `HASH_JOIN` | **build = inner（右）、probe = outer（左）** |
| `FILTER` | 先 `FinalizeMaterializedSubqueries()` 再建 |
| `SORT` | 创建后把 `SortingIterator*` 回填到 `filesort->tables[0]->sorting_iterator` |
| `WINDOW` | 按 `needs_buffering` 选缓冲/非缓冲 |
| `DELETE_ROWS` / `UPDATE_ROWS` | **特殊**：压子 job 之前先调 `SetUpTablesForDelete` / `FinalizeOptimizationForUpdate`（子迭代器构造需要看到最终 read set） |

**创建顺序**：`SetupJobsForChildren` 把 **inner 先压、outer 后压**——栈是 LIFO，出栈顺序是 outer → inner，保证 "left before right"（物化路径的 invalidators 依赖此顺序）。

**MATERIALIZE 分支两处特殊**：① 子节点数是 N+1 且异构——`children[0]` 是"读临时表的迭代器"（必须是单表访问，有 assert 限定类型），`children[1..N]` 是各 query block 的迭代器且**每个可能属于不同的 JOIN**；② 必须等所有子节点就绪——递归 CTE 要通过 `path->iterator` **反向查找**子迭代器树里的 `FollowTailIterator`。

#### 反向指针 AccessPath::iterator

翻译完成后回填。两个用途：① **迭代器间通信**（递归 CTE 找 CTE 根迭代器、BKA 中 MRR 指向外表迭代器）；② **EXPLAIN ANALYZE 取 timing**（执行后反查每个节点实际耗时/行数）。这是"计划（静态）与执行（动态）桥接"的关键一环。

#### EXPLAIN ANALYZE 与翻译失败

`NewIterator<T>` 在 `is_explain_analyze` 时自动包一层 `TimingIterator<T>`，所以此时 `path->iterator` 指向 `TimingIterator<T>*`，取真实类型必须走 `real_iterator()`。⚠️ `MaterializeIterator` / `TemptableAggregateIterator` **不用** `TimingIterator`（会给出误导性测量，注释引 Bug#33834146），它们自己持有 profiler。

**翻译失败**：switch 无 default 分支 → 未覆盖类型导致 `iterator == nullptr` → 函数返回 nullptr。调用点只转布尔、不 `my_error`——**源码把它当作内部不变量破坏，不是面向用户的错误路径**。

---

## 可观测性

| 想看什么 | 手段 |
|---|---|
| 计划树的形状 | `EXPLAIN FORMAT=TREE`（`--><--` 缩进树，最直观）、`EXPLAIN FORMAT=JSON` |
| 每个节点的估算代价 | `EXPLAIN` 的 cost 列 / `FORMAT=JSON` 的 `query_cost`——**注意打印的是 `init_cost` 不是 `cost`** |
| 实际耗时与行数 | `EXPLAIN ANALYZE`（靠 `TimingIterator` 包装 + `iterator` 反向指针回填） |
| `init_once_cost` | 只在 **optimizer trace** 里（`SET optimizer_trace="enabled=on"`），EXPLAIN 不打印 |
| rows examined 的构成 | `EXPLAIN ANALYZE` 的 rows examined（由各节点 `count_examined_rows` 累加） |
| 切换优化器 | `SET optimizer_switch='hypergraph_optimizer=on|off'`（8.0.22+，实验特性） |
| 后处理做了什么 | optimizer trace 里的 `finalize_plan` / `expand_filter` 段落 |

**相关系统变量**：`optimizer_switch`（`hypergraph_optimizer`）、`optimizer_trace`、`optimizer_trace_features`。

★ **展示层与内部层的差异**要时刻记住：EXPLAIN 上的 cost 是"读第一行"的代价（`init_cost`），内部比价用的是"读全量"的 `cost`。同一个查询里，一个"全量代价很高但首行很快"的计划在 EXPLAIN 上会显得比另一个便宜。

---

## Misc

### ★ 本机制里的工程实现技法

1. **模板回调内联**：`WalkAccessPaths` 用三个模板参数（`AccessPathPtr` / `Func` / `JoinPtr`）而非函数指针，让后处理回调被内联进遍历——遍历是热路径，省掉间接跳转。代价是每个调用点实例化一份代码。
2. **union + 类型标签 + 断言访问器**：44 种类型共用一个 ≤144 字节的结构体；空间靠 union 压缩，安全靠 `assert(type == ...)` 在 debug 构建兜住。
3. **placement new + MEM_ROOT**：所有节点 `new (thd->mem_root)`，从不 delete；配合 `unique_ptr_destroy_only`（析构不 delete）让"自动指针语义"与"块式分配"共存。
4. **别名复用内存**：`applied_sargable_join_predicates()` 返回 `filter_predicates` 的引用——两个语义独立的位图共用一块内存（bit 不重叠）。不用 union 是因为 `OverflowBitset` 有非平凡默认构造函数。
5. **显式栈换栈空间**：深树翻译不用递归，用 MEM_ROOT 支撑的任务栈——用内存换调用栈安全。
6. **两阶段任务 + 状态判据**：`IteratorToBeCreated` 重压自己，用"`children` 是否已分配"区分第一次/第二次弹出，省掉显式的阶段字段。
7. **栈上值对象 → 落盘**：hypergraph 的 Propose 先在栈上构造 AccessPath 比价，只有进入 Pareto 前沿才拷贝到 MEM_ROOT——探索性候选零分配成本。

### 扩展点：新增一个 AccessPath 类型要动七处

| # | 位置 | 改动 |
|---|---|---|
| 1 | `sql/join_optimizer/access_path.h` | `enum Type` 加枚举值；union 加参数结构体；加带断言的访问器 |
| 2 | 同文件 | 加 `NewXxxAccessPath()` 工厂函数 |
| 3 | 同文件 | ★ **若新类型属于"基础访问路径"（无子节点）段，必须同步更新 `GetBasicTable()`**——源码 enum 注释明写 "When adding more paths to this section, also update GetBasicTable() to handle them" |
| 4 | `sql/join_optimizer/walk_access_paths.h` | `WalkAccessPaths` 的 44 分支 switch 加分支（遍历子节点） |
| 5 | `sql/join_optimizer/access_path.cc` | `CreateIteratorFromAccessPath` 的 switch 加分支 |
| 6 | `sql/join_optimizer/explain_access_path.cc` | `ExplainAccessPath` 加输出逻辑（TREE + JSON） |
| 7 | `sql/iterators/` | 新迭代器类的实现 |

**三处容易漏掉的约定**：

1. **第 3、4 项漏改是静默的**——不会编译报错，只是 `GetBasicTable()` 返回错、`WalkAccessPaths` 遍历不到新节点的子节点（后处理/EXPLAIN 都看不到）。
2. **cost 三字段要么填全、要么留 `-1.0`**：大部分工厂不填（后填），但 `NewConstTableAccessPath` 这类是填全的。新增时要明确属于哪种。
3. **`count_examined_rows` 几乎每个工厂都有**：漏掉会导致 `EXPLAIN ANALYZE` 的行数统计不对。

**旧优化器 vs hypergraph 的改动量差异**：在旧优化器路径上加类型还要改 `ConnectJoins`（翻译逻辑）与可能的 `QEP_TAB` 字段；hypergraph 下 `Propose*` 直接产出，改动更集中。

> 新增类型最完整的参考案例：`GROUP_INDEX_SKIP_SCAN` 的引入（range 优化器 → AccessPath → 迭代器全链路）。

### 容易误读的点

- **EXPLAIN 的 cost 是 `init_cost`**（读第一行），不是 `cost`（读全量）——展示层与内部层不同。
- **`rescan_cost = cost - init_once_cost`**，但代码存的是 `init_once_cost`（因为绝大多数为 0，省掉大量赋值）。
- **PFS batch mode 是监控埋点的批量，不是计算的批量**——`Read()` 永远只读一行。
- **MATERIALIZE 的子迭代器不是惰性的**——翻译期就建好，被延迟的是物化动作；真正的惰性在三处（`Query_expression` 根迭代器、`SortingIterator` 结果迭代器、临时表 instantiate）。
- **hypergraph 从不创建 `CACHE_INVALIDATOR`**——LATERAL 用 `parameter_tables` + `rematerialize`。
- **`REMOVE_DUPLICATES_ON_INDEX` 只在非 hypergraph 下使用**。
- **hypergraph 不支持 LooseScan**（`need_distinct` 硬编码 true，注释 "We don't support loose index scan yet"）。
- **`ConnectJoins` 有 3 个递归传递的 vector**（不是 4 个），另两个是 `qep_tab_map*` / `table_map*`。
- **`parameter_tables != 0` 不能做 hash join 的右侧**；`RAND_TABLE_BIT` 是"永远不能"的哨兵。

### 坑与已知缺陷

| 坑 | 说明 |
|---|---|
| 遍历 switch 与翻译 switch 是手写 44 分支 | 新增类型漏改 = 静默失效（无编译错误） |
| `first_sj_inner()` 不随 join 重排更新 | 检测 semijoin 必须绕用 `firstmatch_return` |
| 过滤不减代价 | `SetCostOnTableAccessPath` 注释自承 "we don't try to adjust for the filtering here" |
| 排序总落真实表 | `streaming_allowed` 条件 A 是遗留决定，注释自陈 `We could probably be more intelligent here now` |
| I_S 表物化的代价是 `Rudimentary` | `ProposeTableScan` 里作者自承粗略 |
| Halloween 防护用整条命令判定 | INSERT/REPLACE SELECT 一律不流式，TODO 指向 bug #23022426 建议改用 `OPTION_BUFFER_RESULT` |
| hypergraph 不支持 LIS | 带 DISTINCT 的聚合一律 `DISTINCT_AGGREGATOR`，是性能差异来源 |
| 翻译失败不报用户错误 | switch 无 default，返回 nullptr，调用点只转布尔——视为内部不变量破坏 |

---

## 参考

**源码**

- `sql/join_optimizer/access_path.h`——类型枚举、union、工厂函数、代价字段注释
- `sql/join_optimizer/access_path.cc`——`CreateIteratorFromAccessPath`
- `sql/join_optimizer/walk_access_paths.h`——遍历模板与三策略
- `sql/join_optimizer/finalize_plan.cc`——`FinalizePlanForQueryBlock`
- `sql/join_optimizer/explain_access_path.cc`——EXPLAIN 反解
- `sql/sql_executor.cc`——`ConnectJoins`、旧优化器翻译主链
- `sql/sql_union.cc` / `sql/query_term.cc`——集合操作层构建
- `sql/join_optimizer.cc`——`CostingReceiver::Propose*`
- `sql/iterators/composite_iterators.cc`——复合迭代器集中地

**相关文档**

- [`07_optimize/`](07_optimize/)——优化器算法（枚举、代价、DPhyp、CSE）
- [`09_executor_iterator.md`](09_executor_iterator.md)——RowIterator 体系与执行行为
- [`runtime/04_window_function.md`](runtime/04_window_function.md)——窗口函数执行期
- [`07_optimize/19_set_operation.md`](07_optimize/19_set_operation.md)——集合操作语义背景
- [`07_optimize/16_join_object_model.md`](07_optimize/16_join_object_model.md)——`JOIN::m_root_access_path` 挂载点
