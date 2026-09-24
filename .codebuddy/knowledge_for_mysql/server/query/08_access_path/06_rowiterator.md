# 08.6 与 RowIterator 的 1:1 关系：CreateIteratorFromAccessPath

> AccessPath 是"计划"（数据），RowIterator 是"执行"（行为）。`CreateIteratorFromAccessPath` 是两者之间的唯一桥梁。本篇讲这个翻译过程本身（显式栈机制）；每个迭代器的行为细节见 [`../09_executor_iterator.md`](../09_executor_iterator.md)。

## 目录

- [1:1 映射](#11-映射)
- [为什么用显式栈而不是递归](#为什么用显式栈而不是递归)
- [IteratorToBeCreated 与两阶段模式](#iteratortobecreated-与两阶段模式)
- [NewIterator 模板与 examined_rows 绑定](#newiterator-模板与-examined_rows-绑定)
- [batch mode 如何沿树传递](#batch-mode-如何沿树传递)
- [翻译的边界：什么被延迟，什么没有](#翻译的边界什么被延迟什么没有)
- [关键 switch 分支要点](#关键-switch-分支要点)
- [EXPLAIN ANALYZE 与翻译失败](#explain-analyze-与翻译失败)
- [反向指针 AccessPath::iterator](#反向指针-accesspathiterator)

---

## 1:1 映射

每个 AccessPath 类型几乎都对应一个 RowIterator 类型：

| AccessPath | RowIterator |
|---|---|
| `TABLE_SCAN` | `TableScanIterator` |
| `REF` / `REF_OR_NULL` / `EQ_REF` / `PUSHED_JOIN_REF` | `RefIterator` / `RefOrNullIterator` / `EQRefIterator` / `PushedJoinRefIterator` |
| `INDEX_RANGE_SCAN` | `IndexRangeScanIterator` |
| `NESTED_LOOP_JOIN` | `NestedLoopIterator` |
| `HASH_JOIN` | `HashJoinIterator` |
| `SORT` | `SortingIterator` |
| `FILTER` | `FilterIterator` |
| `MATERIALIZE` | `MaterializeIterator` |
| ... | ... |

设计动机：**计划树与执行树分离**。优化器只产出"做什么"（AccessPath），执行器决定"怎么做"（RowIterator），两者通过 1:1 映射解耦。

---

## 为什么用显式栈而不是递归

> `CreateIteratorFromAccessPath`，`sql/join_optimizer/access_path.cc`。

源码注释给了明确原因：

```
The access path trees can be pretty deep, and the stack frames can be big
on certain compilers/setups, so instead of explicit recursion, we push jobs
onto a MEM_ROOT-backed stack. This uses a little more RAM (the MEM_ROOT
typically lives to the end of the query), but reduces the stack usage greatly.
```

翻译：**AccessPath 树可能很深，而栈帧在某些编译器/配置下很大**。所以不用递归（递归会爆调用栈），改用 **MEM_ROOT 支撑的显式任务栈**——代价是多花一点内存（MEM_ROOT 活到查询结束），但极大降低了调用栈占用。

这是"用内存换栈空间"的经典取舍：树的深度 × 单帧大小可能超过系统线程栈（MySQL 线程栈默认 256KB），深查询（几十层的物化嵌套）递归翻译会栈溢出。

## IteratorToBeCreated 与两阶段模式

```cpp
unique_ptr_destroy_only<RowIterator> ret;
Mem_root_array<IteratorToBeCreated> todo(mem_root);
todo.push_back({top_path, top_join, top_eligible_for_batch_mode, &ret, {}});

while (!todo.empty()) {
  IteratorToBeCreated job = todo.back();
  todo.pop_back();
  ...
  switch (path->type) {
    case AccessPath::TABLE_SCAN:
      iterator = NewIterator<TableScanIterator>(...);   // 叶子：直接建
      break;
    // 有子节点的类型：先推子节点 job，再把自己重压栈
    ...
  }
}
```

### 任务结构

`IteratorToBeCreated` 五个字段：`path`（要翻译的 AccessPath）、`join`（所属 query block）、`eligible_for_batch_mode`、`destination`（迭代器建好后放哪）、`children`（子迭代器数组）。

### 两阶段：先推子、再重推自己

```cpp
// The general rule is that if an iterator requires any children, it will push
// jobs for their access paths at the end of the stack and then re-push
// itself. When the children are instantiated and we get back to the original
// iterator, we'll actually instantiate it.
```

有子节点的迭代器（join、filter、sort……）走两阶段：

```
第一次弹出自己 → 发现还没建子节点 → 把各子节点 job 推入栈尾 → 把自己重新压栈
第二次弹出自己 → 子节点都已建好（在 children 里）→ 真正 new 出自己
```

★ 区分"第一次还是第二次"的手段很巧妙：**看 `job.children` 是否已分配**（`Mem_root_array` 是否 `is_empty()` / 是否被创建过）。注释明说 "We distinguish between the two cases on basis of whether job.children has been allocated or not"。

★ 另一个细节：`children` 数组**直接分配在 MEM_ROOT 上**，而不是 `todo` 栈里——因为 `todo` 会 `push_back` 触发 realloc（移动元素），而子迭代器指针要稳定指向这个数组。

## NewIterator 模板与 examined_rows 绑定

### NewIterator<T>

```cpp
iterator = NewIterator<TableScanIterator>(thd, mem_root, param.table, ...);
```

`NewIterator<T>` 是统一工厂：`new (mem_root) T(...)` 并包成 `unique_ptr_destroy_only`（析构时不 delete，因为 MEM_ROOT 统一回收）。所有迭代器都这么建，保证一致的分配策略。

### examined_rows 绑定

```cpp
ha_rows *examined_rows = nullptr;
if (path->count_examined_rows && join != nullptr) {
  examined_rows = &join->examined_rows;
}
```

★ 这就是 `00_structure.md` 里 `count_examined_rows` 字段的**消费点**：翻译时若该节点标了 `count_examined_rows`，就把 `join->examined_rows` 的地址传给迭代器，迭代器每读一行就 `++*examined_rows`——`EXPLAIN ANALYZE` 的 "rows examined" 就是这么累计出来的。

### batch mode 传递

`eligible_for_batch_mode` 参数在翻译时沿树传递，决定迭代器能否进入 PFS batch mode（见 [`../09_executor_iterator.md`](../09_executor_iterator.md) 的 batch mode 章节）。`00_structure.md` 里 APPEND 节点"切换子迭代器前后必须重启 batch mode"的约束，就源于这个参数。

---

## 反向指针 AccessPath::iterator

```cpp
/// If an iterator has been instantiated for this access path, points to the
/// iterator. Used for constructing iterators that need to talk to each other
/// (e.g. for recursive CTEs, or BKA join), and also for locating timing
/// information in EXPLAIN ANALYZE queries.
RowIterator *iterator = nullptr;
```

翻译完成后回填 `path->iterator`。两个用途：

1. **迭代器间通信**：递归 CTE（`FOLLOW_TAIL` 要找 CTE 根的迭代器）、BKA join（MRR 节点要指向外表路径的迭代器）——这些"跨节点协作"靠反向指针找到对方
2. **EXPLAIN ANALYZE timing**：执行后反查每个节点实际耗时/行数

## batch mode 如何沿树传递

> ⚠️ **先定性：PFS batch mode 是「监控埋点的批量」，不是「数据计算的批量」。** 它把 N 次 `start/end_table_io_wait` 计时合并成 1 个事件 + 一个行数计数，**与向量化/一次取多行毫无关系**——`RowIterator::Read()` 的契约始终是 `Read a single row`。误把它写成"批处理执行"是常见错误。

接口只有两个虚函数，都带 `PSI` 前缀：

```cpp
virtual void StartPSIBatchMode() {}
virtual void EndPSIBatchModeIfStarted() {}
```

只有 `TableRowIterator` 真正实现，最终落到 handler 的 PSI 状态机：

```cpp
void TableRowIterator::StartPSIBatchMode() {
  m_table->file->start_psi_batch_mode();
}
```

### 三条传递规则（源码注释明文）

```cpp
    The rules for starting batch and ending mode are:

      1. If you are an iterator with exactly one child (FilterIterator etc.),
         forward any StartPSIBatchMode() calls to it.
      2. If you drive an iterator (read rows from it using a for loop
         or similar), use PFSBatchMode as described above.
      3. If you have multiple children, ignore the call and do your own
         handling of batch mode as appropriate. ...
```

所以**不是"父开启、子继承"那种简单继承**，而是两种机制并存：

| 规则 | 迭代器 | 行为 |
|---|---|---|
| **① 单子转发** | `FilterIterator`、`LimitOffsetIterator`、`SubqueryIterator`、以及 **`TimingIterator`** | 直接转发给子（TimingIterator 转发意味着 EXPLAIN ANALYZE 的包装层**不阻断**传递） |
| **③ 多子忽略 + 自行处理** | `NestedLoopIterator` | 不重写 `StartPSIBatchMode()`（基类空实现 = 忽略），只在 `Init()`/读完后**给 inner 单独开关** |
| **③ 变体** | `AppendIterator` | 自己记住状态，转给"当前"那个子 |

### 谁开、为什么

顶层根迭代器（`Query_expression` 执行入口）:`PFSBatchMode pfs_batch_mode(m_root_iterator.get())` ——这就是注释说的"call StartPSIBatchMode() on the root iterator, and it will trickle all the way down"。

`NestedLoopIterator` 每拿到一个 outer 行就给 inner 开、inner 扫完就关。

**为什么只在最内层开**：

```cpp
    The iterator takes care of activating performance schema batch mode on the
    right iterator if needed; this is typically only used if it is the innermost
    table in the entire join (where the gains from turning on batch mode is the
    largest, and the accuracy loss from turning it off are the least critical).
```

老优化器的判定（`QEP_TAB::pfs_batch_update`）：非最内层 / `eq_ref` / `const` / 带子查询的条件 → 不开。hypergraph 侧有对应函数 `ShouldEnableBatchMode()`，注释明说"mirrors `QEP_TAB::pfs_batch_update()`, with one addition: if there is more than one table, batch mode will be handled by the join iterators on the probe side"。

### 翻译期的"资格"传播

`eligible_for_batch_mode` 沿 `IteratorToBeCreated` 往下传，且**双子节点时只传给 inner**：

```cpp
todo->push_back(
    {inner, join, inner_eligible_for_batch_mode, &job->children[1], {}});
todo->push_back({outer, join, false, &job->children[0], {}});
```

★ 注意 outer 传的是 **false**——外层不该享受 batch mode（它不是最内层）。

### 与计数准确性

`examined_rows` **不受影响**（迭代器自己 `++*m_examined_rows`，与 PSI 无关）；PFS 侧行数也准（`m_psi_numrows` 汇总后一次性报出），**变粗的只有事件粒度与时间分摊**（N 个事件变 1 个，耗时被"均摊"）。

## 翻译的边界：什么被延迟，什么没有

> ⚠️ **纠正一个常见错误说法**："MATERIALIZE 的子查询迭代器在 `Init()` 时才惰性创建"——**源码反证，这是错的**。MATERIALIZE 的每个 query block 子迭代器在翻译期就建好了，作为 `job.children[i+1]`：
>
> ```cpp
> for (size_t i = 0; i < param->query_blocks.size(); ++i) {
>   todo.push_back({from.subquery_path, from.join,
>                   /*eligible_for_batch_mode=*/true, &job.children[i + 1], {}});
> }
> ...
> to.subquery_iterator = std::move(job.children[i + 1]);
> ```
>
> `Init()` 里只是 `Init()` 它们，不创建。被延迟的是**物化动作**（`Init()` 扫完子查询写临时表），不是迭代器构造。

真正被延迟的是这三处：

### ① `Query_expression` 的根迭代器（阶段级惰性）

`Query_expression::optimize()` 有个显式开关 `create_iterators`：

```cpp
if (create_iterators && IteratorsAreNeeded(thd, m_root_access_path)) {
  m_root_iterator = CreateIteratorFromAccessPath(...);
```

- **派生表/视图只建 access path，不建 iterator**：`unit->optimize(thd, table, /*create_iterators=*/false, /*finalize_access_paths=*/true)`
- 需要时再补建：`Query_expression::force_create_iterators()`

**为什么必须推迟**（两条硬注释）：

```cpp
  If you use this function, make sure it's not called at prepare.
  Due to evaluation of LIMIT clause it can not be used at prepared stage.
```

```cpp
  // If the table is const, materialize it now. The hypergraph optimizer
  // doesn't care about const tables, though, so it prefers to do this
  // at execution time (in fact, it will get confused and crash if it has
  // already been materialized).
```

### ② `SortingIterator` 的结果迭代器（类型未定型惰性）

这是全库最硬的"迭代器延迟创建"证据：

```cpp
  // The actual iterator of sorted records, populated in Init();
  // Read() only proxies to this. Always points to one of the members
  // in m_result_iterator_holder; the type can be different depending on
  // e.g. whether the sort result fit into memory or not, whether we are
  // using packed addons, etc..
  unique_ptr_destroy_only<RowIterator> m_result_iterator;
```

**原因很明确：构造前不知道排序结果是否落盘、是否用 packed addons → 类型未定 → 只能在 `Init()` 里定。**

### ③ 临时表 instantiate 延迟到第一次 `Init()`

```cpp
  if (!table()->materialized && table()->pos_in_table_list != nullptr &&
      table()->pos_in_table_list->is_view_or_derived()) {
    // Create the table if it's the very first time.
```

### 两阶段 ≠ 惰性（别混为一谈）

| | 两阶段（先子后父） | 惰性（推迟到执行期） |
|---|---|---|
| 作用域 | **同一次** `CreateIteratorFromAccessPath` 调用内部 | **跨阶段**：optimize → execute / 首次 `Init()` |
| 解决的问题 | 父迭代器构造需要子迭代器对象已存在（构造依赖） | 信息不足或阶段不允许：类型未定、表未 instantiate、LIMIT 不能 prepare 期求值 |
| 位置 | 只在 `access_path.cc` | 分散在 `sql_union.cc`、`sql_derived.cc`、`sorting_iterator.cc` |

同理，**递归 CTE 也不是"延迟创建迭代器"**——递归引用是**建好之后**从树里找回来的（`FindSingleIteratorOfType(from.subquery_path, AccessPath::FOLLOW_TAIL)`）。真正需要"边写边读"的是**表**，不是迭代器。

## 关键 switch 分支要点

| 类型 | 要点 |
|---|---|
| `TABLE_SCAN` / `INDEX_SCAN` / `REF` | 按 `reverse` 选模板参数 |
| `EQ_REF` | **不传 expected_rows**（至多一行，无需 record buffer） |
| `INDEX_RANGE_SCAN` | geometry / reverse / 正序三种 |
| `NESTED_LOOP_JOIN` | `SetupJobsForChildren(outer, inner)` → `NestedLoopIterator(children[0], children[1], join_type, pfs_batch_mode)` |
| `BKA_JOIN` | **特殊**：压子 job **之前**先设 `mrr_path->mrr().bka_path = path`，再从 `mrr_path->iterator->real_iterator()` 拿 `MultiRangeRowIterator*` |
| `HASH_JOIN` | **build = inner（右）、probe = outer（左）** |
| `FILTER` | 先 `FinalizeMaterializedSubqueries()` 再建 |
| `SORT` | 创建后把 `SortingIterator*` 回填到 `filesort->tables[0]->sorting_iterator` |
| `MATERIALIZE` | 见下 |
| `WINDOW` | 按 `needs_buffering` 选缓冲/非缓冲 |
| `DELETE_ROWS` / `UPDATE_ROWS` | **特殊**：压子 job **之前**先调 `SetUpTablesForDelete`/`FinalizeOptimizationForUpdate`（子迭代器构造需要看到最终 read set） |

**创建顺序**：`SetupJobsForChildren` 把 **inner 先压、outer 后压**——栈是 LIFO，出栈顺序是 outer → inner，保证 "left before right"（物化路径的 invalidators 依赖此顺序）。

### MATERIALIZE 分支的特殊处理

1. **子节点数是 N+1 且异构**：`children[0]` 是"物化后读临时表的迭代器"（必须是单表访问，有 assert 限定类型）；`children[1..N]` 是各 query block 的迭代器，且**每个可能属于不同的 JOIN**
2. **必须等所有子节点就绪**：`QueryBlock::recursive_reader` 需要通过 `path->iterator` **反向查找**子迭代器树里的 `FollowTailIterator`

## EXPLAIN ANALYZE 与翻译失败

**`NewIterator<T>` 会在 EXPLAIN ANALYZE 时自动包一层 `TimingIterator<T>`**：

```cpp
template <class RealIterator, class... Args>
unique_ptr_destroy_only<RowIterator> NewIterator(THD *thd, MEM_ROOT *mem_root,
                                                 Args &&... args) {
  if (thd->lex->is_explain_analyze) {
    return ... new (mem_root) TimingIterator<RealIterator>(...);
```

所以 `path->iterator` 在 EXPLAIN ANALYZE 下指向 `TimingIterator<T>*`，取真实类型必须走 `real_iterator()`。⚠️ `MaterializeIterator` / `TemptableAggregateIterator` **不用** `TimingIterator`（会给出误导性测量，注释引了 Bug#33834146），它们自己持有 profiler。

**翻译失败**：switch 无 default 分支 → 未覆盖类型导致 `iterator == nullptr` → 整个函数返回 nullptr。调用点只转布尔、不 `my_error`——**源码把它当作内部不变量破坏，不是面向用户的错误路径**。
