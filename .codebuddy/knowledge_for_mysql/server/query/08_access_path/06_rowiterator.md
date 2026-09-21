# 08.6 与 RowIterator 的 1:1 关系：CreateIteratorFromAccessPath

> AccessPath 是"计划"（数据），RowIterator 是"执行"（行为）。`CreateIteratorFromAccessPath` 是两者之间的唯一桥梁。本篇讲这个翻译过程本身（显式栈机制）；每个迭代器的行为细节见 [`../09_executor_iterator.md`](../09_executor_iterator.md)。

## 目录

- [1:1 映射](#11-映射)
- [为什么用显式栈而不是递归](#为什么用显式栈而不是递归)
- [IteratorToBeCreated 与两阶段模式](#iteratortobecreated-与两阶段模式)
- [NewIterator 模板与 examined_rows 绑定](#newiterator-模板与-examined_rows-绑定)
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

## 翻译的边界：物化子查询惰性翻译

`CreateIteratorFromAccessPath` 只翻译**一个 query block** 的树。跨 block 的 `MATERIALIZE` 节点，其内部子查询树由 `MaterializeIterator` 在 **`Init()` 时惰性翻译**（因为子查询的迭代器要到物化时才需要）。所以 `path->iterator` 只回填"当前 block 根往下"的部分，物化子查询的迭代器是执行期才建的。

这正是"计划树一次性生成、执行树惰性实例化"的体现——计划（AccessPath）是完整的，但执行（RowIterator）按需创建。
