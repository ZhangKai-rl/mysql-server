# 05 join buffer：BNL / BKA / Hash Join 的缓冲机制（算法级）

> 本篇覆盖：join buffer 的三种实际载体、BNL 被 hash join 取代的真相、BKA 与 DS-MRR、hash join 的 build/probe/spill。
>
> ⚠️ **路径勘误（8.0.39 实测）**：本文涉及的实现文件**全部在 `sql/iterators/` 下**，不存在 `sql/hash_join_*.{h,cc}` 与 `sql/bka_iterator.{h,cc}`：
> ```
> sql/iterators/hash_join_buffer.{h,cc}    sql/iterators/hash_join_chunk.{h,cc}
> sql/iterators/hash_join_iterator.{h,cc}  sql/iterators/bka_iterator.{h,cc}
> sql/pack_rows.{h,cc}                     sql/immutable_string.h
> ```
> 另有一批符号在 8.0.39 **不存在**，见 §1 / §3 / §4 的勘误框。

## 目录

- [先有个整体印象](#先有个整体印象)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、关键版本事实：JOIN_CACHE 已删除](#一关键版本事实join_cache-已删除)
- [二、BNL：优化器标记，执行器改写为 hash join](#二bnl优化器标记执行器改写为-hash-join)
- [三、BKA：BKAIterator + MultiRangeRowIterator + DS-MRR](#三bkabkaiterator--multirangerowiterator--ds-mrr)
- [四、hash join 的 build / probe / spill](#四hash-join-的-build--probe--spill)
- [五、setup_join_buffering 决策链](#五setup_join_buffering-决策链)
- [六、深潜补充：chunk 格式、哈希表真相、DS-MRR 完整握手](#六深潜补充chunk-格式哈希表真相ds-mrr-完整握手)

---

## 先有个整体印象

join buffer 的经典动机没变：**避免"外层每进一行，内层表就全表扫一遍"的 O(N×M)**。外层批量读入 buffer，内层只扫一遍，逐行与 buffer 里所有外层行连接。

`join_buffer_size`（`sys_vars.cc:2348`）默认 **256KB**，最终作为 `max_memory_available` 传给 BKA 和 hash join 迭代器。

8.0.39 里 join buffer 的三种实际载体：

| 载体 | 位置 | 用途 |
|------|------|------|
| `HashJoinRowBuffer` | `sql/iterators/hash_join_buffer.h:119` | hash join 的 build 端内存哈希表 |
| `BKAIterator` 的 `MEM_ROOT m_mem_root` + `m_rows` | `sql/iterators/bka_iterator.h:82` | BKA 的外层行缓冲 |
| BKA 的 `HANDLER_BUFFER m_mrr_buffer` | `sql/iterators/bka_iterator.h:264` | 内层批量读的 rowid 排序缓冲区 |

> ★ 三者的**共同底座**是 `pack_rows::TableCollection`（`sql/pack_rows.h:57`）——hash join、BKA、流式聚合**共用同一套行打包/解包设施**。它的注释原文就写着 "hash join operation, **BKA join operation** or a streaming aggregation operation"。
>
> ★ 而且 `BufferRow` 根本不是独立结构：`using Key = std::string_view; using BufferRow = Key;`（`hash_join_buffer.h:102-105`）——**BKA 直接复用了 hash join 的行表示**。

```cpp
// sql/pack_rows.h:57 —— 真正的"外层行元信息容器"（不存在 class JoinBuffer）
namespace pack_rows {
/// A class that represents a field, which also holds a cached value of the
/// field's data type.
struct Column {
  explicit Column(Field *field);
  Field *const field;
  // The field type is used frequently, and caching it gains around 30% in some
  // of our microbenchmarks.
  const enum_field_types field_type;
};

struct Table {
  explicit Table(TABLE *table_arg);
  TABLE *table;
  Prealloced_array<Column, 8> columns;
  bool copy_null_flags{false};
  bool store_contents_of_null_rows{false};
};

/// A structure that contains a list of input tables for a hash join operation,
/// BKA join operation or a streaming aggregation operation, and some
/// pre-computed properties for the tables.
class TableCollection {
 public:
  const Prealloced_array<Table, 4> &tables() const { return m_tables; }
  table_map tables_bitmap() const { return m_tables_bitmap; }
  size_t ref_and_null_bytes_size() const { return m_ref_and_null_bytes_size; }
  bool has_blob_column() const { return m_has_blob_column; }
  ...
 private:
  Prealloced_array<Table, 4> m_tables{PSI_NOT_INSTRUMENTED};
  table_map m_tables_bitmap = 0;
  size_t m_ref_and_null_bytes_size = 0;
  /**
    Whether any of the tables has a BLOB/TEXT column. ... If this is set to
    false, we can pre-allocate any necessary buffers we need during the
    operation, and thus eliminate the need for recalculating the row size
    every time.
  */
  bool m_has_blob_column = false;
  ...
};
```

> ★ `has_blob_column()` 这个标志解释了一个反直觉现象：**有 BLOB/TEXT 列时无法预估行大小，必须每行重算**——所以 `ComputeRowSizeUpperBound()` 只在无 BLOB 时能用于预分配。

---

## 设计思想与理论基础

### 为什么需要 join buffer：NLJ 的 O(N×M) 困境

朴素嵌套循环（NLJ）的问题是：外层每进一行，内层表就要**完整扫一遍**。外层 N 行、内层 M 行 ⇒ 内层被扫 N 次，总代价 O(N×M)。当内层没有可用索引时只能如此。

join buffer 的思路是**批量化**：外层一次读入 buffer（一批），内层只扫一遍，与 buffer 里所有外层行比对，内层扫描次数从 N 降到 N/batch。

> 本质：**用内存换 IO**。这是 join buffer 家族所有技术的共同思想——**区别只在"换掉的究竟是什么"**。

### 三条路线，换掉的东西不同

| 技术 | 换掉什么 | 机制 | 比较次数 |
|---|---|---|---|
| **BNL**（Block Nested Loop） | 内层的**重复扫描** | 外层批量入 buffer，内层扫一遍 × 与 buffer 全部行比对 | **仍是 O(N×M)** |
| **BKA**（Batched Key Access） | **随机 IO** | 外层批量 → 收集 key → MRR 按 rowid 排序 → 批量索引查找 | O(N)（走索引） |
| **hash join** | **比较次数本身** | build 端建哈希表，probe 端 O(1) 探测 | **O(N+M)** |

**关键洞察**：BNL 只省了 IO，**没省比较**——仍要拿内层每一行去和 buffer 里所有行比对。hash join 则把比较次数从 O(N×M) 降到 O(N+M)。

所以 **hash join 在各维度上都不劣于 BNL**，这就是它取代 BNL 的理论原因。BKA 不与之冲突：它只适用于**内层有索引**的场景，价值是把"按外层顺序的随机索引查找"变成"按 rowid 排序后的顺序查找"（DS-MRR）。

### 演进：为什么 BNL 只剩一个标记

BNL 是"没有 hash join 的年代"的折中。8.0 引入 hash join 后，优化器仍产出 `ALG_BNL` 标记，但**执行器一律改写为 hash join**——BNL 退化为一个**优化器侧的遗留标记**，真正的"分块循环"语义由 hash join 的 `IN_MEMORY_WITH_HASH_TABLE_REFILL` 模式承载。

这也解释了为什么 `JOIN_CACHE` 会被删到只剩 33 行空壳枚举：**实现没了，标记还在**。

### 代价与限制

- **`join_buffer_size` 是双向约束**：太小 → 装不下 build 端，需多趟 refill（退化为"分块"）；太大 → 并发查询内存压力大（它是**每个 join 各自分配**，不是全局共享）
- **spill**：hash join 装不下时溢出到磁盘 chunk 文件，代价陡增
- **适用面**：hash join 只用于等值连接；BKA 要求内层有索引

### 他库对比

- PostgreSQL 从一开始就把 hash join 作为主力，并有 `work_mem` 控制 spill——与 MySQL 的 `join_buffer_size` 定位相似
- MySQL 直到 **8.0.18** 才有 hash join，此前长期只有 NLJ + BNL + BKA，这是它历史上 OLAP 能力弱的重要原因之一

---

## 一、关键版本事实：JOIN_CACHE 已删除

> ⚠️ `sql/sql_join_buffer.h` **全文只有 34 行**，唯一实体代码就是下面这个三值枚举（第 27-33 行；剩下的 26 行是 GPL 头 + include guard + `#endif`）。**没有任何 include、成员函数或数据成员**：

```cpp
// sql/sql_join_buffer.h:27
class JOIN_CACHE {
 public:
  /** Bits describing cache's type @sa setup_join_buffering() */
  enum enum_join_cache_type { ALG_NONE = 0, ALG_BNL = 1, ALG_BKA = 2 };
};
```

`JOIN_CACHE_BNL`/`JOIN_CACHE_BKA` 类、`sql_join_buffer.cc`、`join_records()`/`join_matching_records()`/`get_match_flag()` **全部不存在**（5.x 时代实现，8.0 迭代器化 + hash join 引入后删除）。所以：

- 优化器仍产出 `ALG_BNL` 标记，但**执行器一律改写为 hash join**。
- 经典 BNL 的"分块循环"语义由 hash join 的 `IN_MEMORY_WITH_HASH_TABLE_REFILL` 模式承载。

---

## 二、BNL：优化器标记，执行器改写为 hash join

### 2.1 优化器侧：JT_ALL → ALG_BNL

⚠️ **勘误**：`setup_join_buffering` **不是 `JOIN` 的成员函数**，而是 `sql/sql_optimizer.cc:3451` 的 **`static` 自由函数**（前向声明 `:149`）。它对全表扫描类访问路径打标记（`switch` 在 `:3621`）：

```cpp
  switch (tab->type()) {
    case JT_ALL:
    case JT_INDEX_SCAN:
    case JT_RANGE:
    case JT_INDEX_MERGE:
      if (!bnl_on) {
        assert(tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
        goto no_join_cache;
      }

      if (!join->select_count) tab->set_use_join_cache(JOIN_CACHE::ALG_BNL);
      return false;
    case JT_SYSTEM:
    case JT_CONST:
    case JT_REF:
    case JT_EQ_REF:
      if (!bka_on) {
        assert(tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
        goto no_join_cache;
      }

      /*
        Disable BKA for materializable derived tables/views as they aren't
        instantiated yet.
      */
      if (tab->table_ref->uses_materialization()) goto no_join_cache;

      /*
        Can't use BKA for subquery if dealing with a subquery that can
        turn a ref access into a "full scan on NULL key" table scan.
        ... @see Index_lookup::cond_guards ...
      */
      if (tab->has_guarded_conds()) goto no_join_cache;

      if (tab->table()->covering_keys.is_set(tab->ref().key))
        join_cache_flags |= HA_MRR_INDEX_ONLY;
      rows = tab->table()->file->multi_range_read_info(
          tab->ref().key, 10, 20, &bufsz, &join_cache_flags, &cost);
      /*
        Cannot use BKA if
        1. MRR scan cannot be performed, or
        2. MRR default implementation is used, or
        3. HA_MRR_NO_ASSOCIATION flag is set
      */
      if ((rows == HA_POS_ERROR) ||                        // 1
          (join_cache_flags & HA_MRR_USE_DEFAULT_IMPL) ||  // 2
          (join_cache_flags & HA_MRR_NO_ASSOCIATION))      // 3
        goto no_join_cache;

      tab->set_use_join_cache(JOIN_CACHE::ALG_BKA);

      tab->join_cache_flags = join_cache_flags;
      return false;
    default:;
  }

no_join_cache:
  revise_cache_usage(tab);
  tab->set_use_join_cache(JOIN_CACHE::ALG_NONE);
  return false;
```

> ★ `multi_range_read_info(key, 10, 20, ...)` 里的 **10/20 是传给 handler 的"每个 key 期望行数"上下界**——这就是后面 `rec_per_key` / `mrr_length_per_rec` 的来源。
> ★ `tab->join_cache_flags` 最终经 `NewMRRAccessPath(..., qep_tab->position()->table->join_cache_flags)`（`sql_executor.cc:2756`）流入 `MultiRangeRowIterator::m_mrr_flags`。

### 2.2 执行器侧：OT_BNL 改写为 hash join

`QEP_TAB::OT_BNL` 的语义已写成 "Block-nested loop (rewritten to hash join)"（`sql_executor.h:423`）：

```cpp
  // Operation between the previous QEP_TAB and this one.
  enum enum_op_type {
    OT_NONE,                        // Regular nested loop.
    OT_AGGREGATE,                   // Aggregate (GROUP BY).
    OT_MATERIALIZE,                 // Various temporary table operations,
    OT_AGGREGATE_THEN_MATERIALIZE,  //   used at the end of the join.
    OT_AGGREGATE_INTO_TMP_TABLE,
    OT_WINDOWING_FUNCTION,

    // Block-nested loop (rewritten to hash join).
    OT_BNL,

    // Batch key access.
    OT_BKA
  } op_type = OT_NONE;
```

`ConnectJoins`（`sql_executor.cc:2726`）：

```cpp
const bool replace_with_hash_join =
    UseHashJoin(qep_tab) && !QueryMixesOuterBKAAndBNL(qep_tab->join());
```

**`UseHashJoin()` / `UseBKA()` / `QueryMixesOuterBKAAndBNL()` 完整源码**（`sql_executor.cc:2188` / `:2197` / `:2221`）：

```cpp
static bool UseHashJoin(QEP_TAB *qep_tab) {
  return qep_tab->op_type == QEP_TAB::OT_BNL;
}

static bool UseBKA(QEP_TAB *qep_tab) {
  if (qep_tab->op_type != QEP_TAB::OT_BKA) {
    // Not BKA.
    return false;
  }

  // Similar to QueryMixesOuterBKAAndBNL(), if we have an outer join BKA
  // that contains multiple tables on the right side, we will not have a
  // left-deep tree, which we cannot handle at this point.
  if (qep_tab->last_inner() != NO_PLAN_IDX &&
      qep_tab->last_inner() != qep_tab->idx()) {
    // More than one table on the right side of an outer join, so not
    // left-deep.
    return false;
  }
  return true;
}

// Having a non-BKA join on the right side of an outer BKA join causes problems
// for the matched-row signaling from MultiRangeRowIterator to BKAIterator;
// rows could be found just fine, but not go through the join filter (and thus
// not be marked as matched in BKAIterator), creating extra NULLs.
//
// The only way this can happen is when we get a hash join on the inside of an
// outer BKA join (otherwise, the join tree will be left-deep). If this
// happens, we simply turn off both BKA and hash join handling for the query;
// it is a very rare situation, and the slowdown should be acceptable.
// (Only turning off BKA helps somewhat, but MultiRangeRowIterator also cannot
// be on the inside of a hash join, so we need to turn off BNL as well.)
static bool QueryMixesOuterBKAAndBNL(JOIN *join) {
  bool has_outer_bka = false;
  bool has_bnl = false;
  for (uint i = join->const_tables; i < join->primary_tables; ++i) {
    QEP_TAB *qep_tab = &join->qep_tab[i];
    if (UseHashJoin(qep_tab)) {
      has_bnl = true;
    } else if (qep_tab->op_type == QEP_TAB::OT_BKA &&
               qep_tab->last_inner() != NO_PLAN_IDX) {
      has_outer_bka = true;
    }
  }
  return has_bnl && has_outer_bka;
}
```

> ★★ 这段注释**直接点出了 BKA 的根本脆弱点**：BKA 靠 `MarkLastRowAsRead()` 把"匹配成功"从 MRR 回传给 BKAIterator。若外连接 BKA 的右侧子树里嵌了 hash join，MRR 匹配到的行可能被中间的 join filter 吃掉 ⇒ `MarkLastRowAsRead()` 不被调用 ⇒ 本该匹配的外层行被误判成"无匹配"而**多产 NULL 行**。解法很粗暴：**把整个查询的 BKA 和 BNL 一起关掉**。

`is_bka` 与 `replace_with_hash_join` 在 `ConnectJoins` 里**互斥**：

```cpp
      if (is_bka) {
        Index_lookup &ref = qep_tab->ref();
        table_path =
            NewMRRAccessPath(thd, qep_tab->table(), &ref,
                             qep_tab->position()->table->join_cache_flags);
        ...
      } else if (replace_with_hash_join) {
        ExtractJoinConditions(qep_tab, &predicates_below_join, &join_conditions);
      }
      ...
      if (is_bka) {
        path = CreateBKAAccessPath(thd, qep_tab->join(), path, left_tables,
                                   table_path, right_tables, qep_tab->table(),
                                   qep_tab->table_ref, &qep_tab->ref(),
                                   JoinType::INNER);
      } else if (replace_with_hash_join) {
        // The numerically lower QEP_TAB is often (if not always) the smaller
        // input, so use that as the build input.
        ...
        path = CreateHashJoinAccessPath(thd, qep_tab, path, left_tables,
                                        table_path, right_tables,
                                        JoinType::INNER, &join_conditions,
                                        conditions_depend_on_outer_tables);
        // Attach any remaining non-equi-join conditions as a filter after the join.
        path = PossiblyAttachFilter(path, join_conditions, thd,
                                    conditions_depend_on_outer_tables);
      } else {
        path = CreateNestedLoopAccessPath(
            thd, path, table_path, JoinType::INNER,
            qep_tab->pfs_batch_update(qep_tab->join()));
        ...
      }
```

> ★ 第三处判定（`:2616`）多了一个 `!right_side_depends_on_outer`：**hash join 子树若依赖外层表就不能替换**（外层行变化时 hash table 得重建），但 **BKA 没有这个限制**——它每批重建 range，天然支持相关子查询式的外依赖。

传统 EXPLAIN 里 `OT_BNL` 也直接显示成 "hash join"（`opt_explain.cc:1664`）。

### 2.3 退化两种情况

- **hash join 不允许 spill**（有 LIMIT 且无 group/order）→ 进入 `IN_MEMORY_WITH_HASH_TABLE_REFILL` 模式（即 BNL 骨架）。判定源码（`sql_executor.cc:2088`；⚠️ 不是 `:2076`）：

```cpp
  const JOIN *join = qep_tab->join();
  const bool has_grouping = join->implicit_grouping || join->grouped;
  const bool has_limit = join->m_select_limit != HA_POS_ERROR;
  const bool has_order_by = join->order.order != nullptr;

  // If we have a limit in the query, do not allow hash join to spill to
  // disk. The effect of this is that hash join will start producing
  // result rows a lot earlier, and thus hit the LIMIT a lot sooner.
  // Ideally, this should be decided during optimization.
  // There are however two situations where we always allow spill to disk,
  // and that is if we either have grouping or sorting in the query. In
  // those cases, the iterator above us will most likely consume the
  // entire result set anyways.
  const bool allow_spill_to_disk = !has_limit || has_grouping || has_order_by;
```

  → `path->hash_join().allow_spill_to_disk = allow_spill_to_disk;`（`sql_executor.cc:2157`）
  → 经 `CreateIteratorFromAccessPath`（`join_optimizer/access_path.cc:820`）传给 `HashJoinIterator` 的 `m_allow_spill_to_disk`
  → 另有两处来源：hypergraph 恒 `true`（`join_optimizer.cc:3369`）、NDB 恒 `false`（`ha_ndbcluster.cc:14323`）

- **外层 BKA + 内层 hash join 混用** → `QueryMixesOuterBKAAndBNL`（`:2221`）检测到后两者都关掉，整条查询退化成普通 nested loop（避免 matched-row 信号错乱）。

---

## 三、BKA：BKAIterator + MultiRangeRowIterator + DS-MRR

BKA 用两个迭代器协作（`sql/iterators/bka_iterator.h:27` 文件头注释）：`BKAIterator` 批量读外层行，`MultiRangeRowIterator` 用 MRR 批量读内层行。

**文件头注释（BKA 的设计说明书）**：

```cpp
/**
  Batch key access (BKA) is a join strategy that uses multi-range read (MRR)
  to get better read ordering on the table on the inner side. It reads
  a block of rows from the outer side, picks up the join keys (refs) from
  each row, and sends them all off in one big read request. The handler can
  then order these and read them in whatever order it would prefer. This is
  especially attractive when used with rotating media; the reads can then be
  ordered such that it does not require constant seeking (disk-sweep MRR,
  or DS-MRR).

  BKA is implemented with two iterators working in concert. The BKAIterator
  reads rows from the outer side into a buffer. When the buffer is full or we
  are out of rows, it then sets up the key ranges and hand it over to the
  MultiRangeRowIterator, which does the actual request, and reads rows from it
  until there are none left. For each inner row returned, MultiRangeRowIterator
  loads the appropriate outer row(s) from the buffer, doing the actual join.

  The reason for this split is twofold. First, it allows us to accurately time
  (for EXPLAIN ANALYZE) the actual table read. Second, and more importantly,
  we can have other iterators between the BKAIterator and MultiRangeRowIterator,
  in particular FilterIterator.
*/
```

> ★ 注意最后一句：**BKAIterator 与 MultiRangeRowIterator 之间允许夹 FilterIterator**。这正是后面 `QueryMixesOuterBKAAndBNL()` 要防的坑——过滤掉的行不会触发 `MarkLastRowAsRead()`。

```cpp
// sql/iterators/bka_iterator.h:82 —— 状态机只有 4 态
  enum class State {
    NEED_OUTER_ROWS,                  // 准备读外层行（一个 Read() 填满，无中间态）
    RETURNING_JOINED_ROWS,            // 从 MultiRangeRowIterator 吐连接结果
    RETURNING_NULL_COMPLEMENTED_ROWS, // outer/anti join 的 NULL 补行（在 BKAIterator 里做，
                                      //   否则会被中间的 FilterIterator 误过滤）
    END_OF_ROWS
  };
```

### 3.1 BKAIterator：批量读外层 + 内存预算

**构造函数**（`bka_iterator.cc:70`）——最后一行是整个内存切分的核心：

```cpp
BKAIterator::BKAIterator(THD *thd,
                         unique_ptr_destroy_only<RowIterator> outer_input,
                         const Prealloced_array<TABLE *, 4> &outer_input_tables,
                         unique_ptr_destroy_only<RowIterator> inner_input,
                         size_t max_memory_available,
                         size_t mrr_bytes_needed_for_single_inner_row,
                         float expected_inner_rows_per_outer_row,
                         bool store_rowids, table_map tables_to_get_rowid_for,
                         MultiRangeRowIterator *mrr_iterator, JoinType join_type)
    : RowIterator(thd),
      m_outer_input(std::move(outer_input)),
      m_inner_input(std::move(inner_input)),
      m_mem_root(key_memory_hash_join, 16384 /* 16 kB */),   // ★ 与 hash join 同款 MEM_ROOT
      m_rows(&m_mem_root),
      m_outer_input_tables(outer_input_tables, store_rowids,
                           tables_to_get_rowid_for,
                           /*tables_to_store_contents_of_null_rows_for=*/0),
      m_max_memory_available(max_memory_available),
      m_mrr_bytes_needed_for_single_inner_row(
          mrr_bytes_needed_for_single_inner_row),
      m_mrr_iterator(mrr_iterator),
      m_join_type(join_type) {
  assert(m_outer_input != nullptr);
  assert(m_inner_input != nullptr);

  m_mrr_bytes_needed_per_row =
      lrint(mrr_bytes_needed_for_single_inner_row *
            std::max(expected_inner_rows_per_outer_row, 1.0f));
  // ★★ 每个外层行要预留的 MRR 字节 = 单行内层行大小 × 每 key 期望匹配行数（至少 1）
}
```

> ★ `std::max(..., 1.0f)` 保证即便统计信息说"平均匹配不到 1 行"，也**至少按 1 行预留**，否则 MRR buffer 会算成 0。

```cpp
// bka_iterator.cc:101
bool BKAIterator::Init() {
  if (!m_outer_input_tables.has_blob_column()) {
    size_t upper_row_size =
        pack_rows::ComputeRowSizeUpperBound(m_outer_input_tables);
    if (m_outer_row_buffer.reserve(upper_row_size)) {
      my_error(ER_OUTOFMEMORY, MYF(0), upper_row_size);
      return true;
    }
    // ★ 只有无 BLOB/TEXT 才预分配；有 BLOB 时由 StoreFromTableBuffers 逐行 reserve
  }
  PrepareForRequestRowId(m_outer_input_tables.tables(),
                         m_outer_input_tables.tables_to_get_rowid_for());

  BeginNewBatch();
  m_end_of_outer_rows = false;
  m_has_row_from_previous_batch = false;

  return m_outer_input->Init();    // ★ 注意：这里不 Init 内层（range 还没确定）
}

// bka_iterator.cc:120 —— 每批只是"整块复用"MEM_ROOT，不释放物理块
void BKAIterator::BeginNewBatch() {
  m_mem_root.ClearForReuse();
  new (&m_rows) Mem_root_array<BufferRow>(&m_mem_root);
  m_bytes_used = 0;
  m_state = State::NEED_OUTER_ROWS;
}
```

**`ReadOuterRows()` 完整源码**（`bka_iterator.cc:127`）：

```cpp
int BKAIterator::ReadOuterRows() {
  for (;;) {
    if (m_has_row_from_previous_batch) {
      // The outer row will be in m_outer_row_buffer already. Load it back
      // into the global table buffers; MultiRangeRowIterator has loaded other
      // rows into them, and in case we are reading from a join, Read() may
      // not update all of the tables.
      m_has_row_from_previous_batch = false;
      LoadBufferRowIntoTableBuffers(
          m_outer_input_tables,
          hash_join_buffer::Key(m_outer_row_buffer.ptr(),
                                m_outer_row_buffer.length()));
    } else {
      int result = m_outer_input->Read();
      if (result == 1) {
        return 1;               // Error.
      }
      if (result == -1) {
        m_end_of_outer_rows = true;
        break;                  // EOF.
      }
      RequestRowId(m_outer_input_tables.tables(),
                   m_outer_input_tables.tables_to_get_rowid_for());

      // Save the contents of all columns marked for reading.
      if (StoreFromTableBuffers(m_outer_input_tables, &m_outer_row_buffer)) {
        return 1;
      }
    }

    // See if we have room for this row, and the associated number of MRR
    // rows, without going over our total RAM budget. (We ignore the budget
    // if the buffer is empty; at least a single row must be allowed at all
    // times.)
    const size_t row_size = m_outer_row_buffer.length();
    size_t total_bytes_needed_after_this_row =
        m_bytes_used + row_size +
        (m_mrr_bytes_needed_per_row + sizeof(m_rows[0])) * (m_rows.size() + 1);

    if (NeedMatchFlags(m_join_type)) {
      total_bytes_needed_after_this_row +=
          BytesNeededForMatchFlags(m_rows.size() + 1);
    }

    if (!m_rows.empty() &&
        total_bytes_needed_after_this_row > m_max_memory_available) {
      // Out of memory, so end the batch and send it.
      // This row will be dealt with in the next batch.
      m_has_row_from_previous_batch = true;
      break;
    }

    char *row = m_mem_root.ArrayAlloc<char>(row_size);
    if (row == nullptr) {
      return 1;
    }
    memcpy(row, m_outer_row_buffer.ptr(), row_size);

    m_rows.push_back(BufferRow(row, row_size));
    m_bytes_used += row_size;
  }

  // If we had no rows at all, we're done.
  if (m_rows.empty()) {
    assert(!m_has_row_from_previous_batch);
    m_state = State::END_OF_ROWS;
    return -1;
  }

  // Figure out how much RAM we need to allocate for the MRR row buffer,
  // given to the handler for holding inner rows.
  size_t mrr_buffer_size = m_mrr_bytes_needed_per_row * m_rows.size();
  if (m_bytes_used + mrr_buffer_size >= m_max_memory_available) {
    // Even if it will take us over budget, DS-MRR needs space for at least
    // one row to work.
    assert(m_rows.size() ==
           1);  // Otherwise, we would have stopped reading rows earlier.
    if (m_bytes_used + m_mrr_bytes_needed_for_single_inner_row >=
        m_max_memory_available) {
      mrr_buffer_size = m_mrr_bytes_needed_for_single_inner_row;
    } else {
      mrr_buffer_size = m_max_memory_available - m_bytes_used;
    }
  } else {
    // We're under budget. Heuristically, increase it to get some
    // extra headroom if the estimate is pessimistic.
    mrr_buffer_size = std::min(mrr_buffer_size * 2 + 16384,
                               m_max_memory_available - m_bytes_used);
  }
  assert(mrr_buffer_size >= m_mrr_bytes_needed_for_single_inner_row);

  // Ask the MRR iterator to do the actual read.
  m_mrr_iterator->set_rows(m_rows.begin(), m_rows.end());
  m_mrr_iterator->set_mrr_buffer(m_mem_root.ArrayAlloc<uchar>(mrr_buffer_size),
                                 mrr_buffer_size);
  if (NeedMatchFlags(m_join_type)) {
    const size_t bytes_needed = BytesNeededForMatchFlags(m_rows.size());
    m_mrr_iterator->set_match_flag_buffer(
        m_mem_root.ArrayAlloc<uchar>(bytes_needed));
  }
  if (m_inner_input->Init()) {    // ★★ 内层迭代器在这一刻才 Init
    return 1;
  }

  // Probe the rows we've got using MRR.
  m_state = State::RETURNING_JOINED_ROWS;
  m_mrr_iterator->SetNullRowFlag(false);
  return 0;
}
```

每读一行做**内存预算**，预留量 = 已用 + 本行 + `(每 key 的 MRR 字节 + 一个 BufferRow 指针) × (行数+1)` + match flag 位图。三个关键细节：

- `!m_rows.empty() &&` ——**缓冲区为空时无条件放行**，保证任何情况下至少能装 1 行（否则死锁）；
- 超预算**不丢行**，而是置 `m_has_row_from_previous_batch = true`，本批结束、该行留到下一批；
- `mrr_buffer_size` 的三分支：正常 = `per_row × rows`；超预算 = **至少保 1 行**（连 1 行都放不下就硬给，故意超预算）；有余量 = `min(2×mrr + 16384, 剩余)`。

**`Read()` 状态机**（`bka_iterator.cc:270`）：

```cpp
int BKAIterator::Read() {
  for (;;) {  // Termination condition within loop.
    switch (m_state) {
      case State::END_OF_ROWS:
        return -1;
      case State::NEED_OUTER_ROWS: {
        int err = ReadOuterRows();
        if (err != 0) return err;
        break;
      }
      case State::RETURNING_JOINED_ROWS: {
        int err = m_inner_input->Read();
        if (err != -1) {
          if (err == 0) {
            m_mrr_iterator->MarkLastRowAsRead();   // ★★ 唯一的"匹配回执"
            if (m_join_type == JoinType::ANTI) {
              break;
            }
          }
          return err;   // A row or an error; pass it through.
        }

        // No more joined rows in this batch. Go to the next batch -- but
        // if we're an outer join or antijoin, first create NULL-complemented
        // rows for the ones in this batch that we didn't match to anything.
        if (m_join_type == JoinType::OUTER || m_join_type == JoinType::ANTI) {
          m_state = State::RETURNING_NULL_COMPLEMENTED_ROWS;
          m_current_pos = m_rows.begin();
          m_mrr_iterator->SetNullRowFlag(true);
        } else {
          BatchFinished();
          break;
        }
      }
        [[fallthrough]];   // ★ 故意穿透：刚切到补行态就立刻产出
      case State::RETURNING_NULL_COMPLEMENTED_ROWS: {
        int err = MakeNullComplementedRow();
        if (err != -1) return err;

        BatchFinished();
        break;
      }
    }
  }
}
```

```cpp
// bka_iterator.cc:239
void BKAIterator::BatchFinished() {
  if (m_end_of_outer_rows) {
    m_state = State::END_OF_ROWS;
  } else {
    BeginNewBatch();
    assert(m_state == State::NEED_OUTER_ROWS);
  }
}

// bka_iterator.cc:248 —— match flag 位图的消费者
int BKAIterator::MakeNullComplementedRow() {
  // Find the next row that hasn't been matched to anything yet.
  while (m_current_pos != m_rows.end()) {
    if (m_mrr_iterator->RowHasBeenRead(m_current_pos)) {
      ++m_current_pos;
    } else {
      // Return a NULL-complemented row. (Our table already has the NULL flag set.)
      LoadIntoTableBuffers(m_outer_input_tables,
                           pointer_cast<const uchar *>(m_current_pos->data()));
      ++m_current_pos;
      return 0;
    }
  }

  // No more NULL-complemented rows to return.
  m_mrr_iterator->SetNullRowFlag(false);
  return -1;
}
```

### 3.2 MultiRangeRowIterator：MRR 回调生成 key range

**`Init()`**（`bka_iterator.cc:335`）——注册回调的那一刻：

```cpp
bool MultiRangeRowIterator::Init() {
  /*
    Prepare to iterate over keys from the join buffer and to get
    matching candidates obtained with MRR handler functions.
   */
  if (!m_file->inited) {
    const int error = m_file->ha_index_init(m_ref->key, true);
    if (error) {
      m_file->print_error(error, MYF(0));
      return error;
    }
  }
  RANGE_SEQ_IF seq_funcs = {MultiRangeRowIterator::MrrInitCallbackThunk,
                            MultiRangeRowIterator::MrrNextCallbackThunk,
                            nullptr};
  if (m_join_type == JoinType::SEMI || m_join_type == JoinType::ANTI) {
    seq_funcs.skip_record = MultiRangeRowIterator::MrrSkipRecordCallbackThunk;
    // ★ skip_record 只在 semi/anti 注册 —— 这是"同一外层行只匹配一次"的去重机制
  }
  if (m_match_flag_buffer != nullptr) {
    assert(NeedMatchFlags(m_join_type));
    // Reset all the match flags.
    memset(m_match_flag_buffer, 0,
           BytesNeededForMatchFlags(std::distance(m_begin, m_end)));
  } else {
    assert(!NeedMatchFlags(m_join_type));
  }

  /**
    We don't send a set of rows directly to MRR; instead, we give it a set
    of function pointers to iterate over the rows, and a pointer to ourselves.
    The handler will call our callbacks as follows:

     1. MrrInitCallback at the start, to initialize iteration.
     2. MrrNextCallback is called to yield ranges to scan, until it returns 1.
   */
  return m_file->multi_range_read_init(&seq_funcs, this,
                                       std::distance(m_begin, m_end),
                                       m_mrr_flags, &m_mrr_buffer);
}
```

**`MrrNextCallback()`**（`bka_iterator.cc:381`）——★ `range->ptr` 是整个 BKA 关联机制的枢纽：

```cpp
uint MultiRangeRowIterator::MrrNextCallback(KEY_MULTI_RANGE *range) {
  // Load the next row from the buffer, if there is one.
  //
  // NULL values will never match in a inner join. The optimizer will often
  // set up a NULL filter for inner joins, but not in all cases, so we must
  // skip such rows by checking impossible_null_ref(). Thus, we iterate
  // until we have a row that is not NULL-filtered. ...
  for (;;) {
    if (m_current_pos == m_end) {
      return 1;
    }

    LoadBufferRowIntoTableBuffers(m_outer_input_tables, *m_current_pos);
    // ★ 有副作用：把外层行还原进全局 record buffer

    construct_lookup(thd(), table(), m_ref);
    if (!m_ref->impossible_null_ref()) {
      break;
    }
    ++m_current_pos;   // ★ HA_MRR_NO_NULL_ENDPOINTS 的运行时兜底
  }

  // Set up a range consisting of a single key, so the only difference
  // between start and end is the flags. They signify that the range starts
  // at the row in question, and ends right after it (exclusive).

  range->range_flag = EQ_RANGE;
  range->ptr = const_cast<char *>(pointer_cast<const char *>(m_current_pos));
  // ★★ 把"外层行的 BufferRow 指针"塞进 range，handler 取回行时原样回传

  range->start_key.key = m_ref->key_buff;
  range->start_key.keypart_map = (1 << m_ref->key_parts) - 1;  // All keyparts.
  range->start_key.length = m_ref->key_length;
  range->start_key.flag = HA_READ_KEY_EXACT;

  range->end_key = range->start_key;
  range->end_key.flag = HA_READ_AFTER_KEY;

  ++m_current_pos;
  return 0;
}
```

> ★ 每个 range 是**单点等值区间**：start 与 end 除 flag 外完全相同（`HA_READ_KEY_EXACT` / `HA_READ_AFTER_KEY`）。BKA **不会为每个外层行单独发一次索引查找**——一次 `multi_range_read_init()` 把整批 range 交给 handler，handler 自行排序（DS-MRR）。

**`Read()`**（`bka_iterator.cc:431`）——用 `rec_ptr` 反查外层行，完成 join：

```cpp
int MultiRangeRowIterator::Read() {
  // Read a row from the MRR buffer. rec_ptr tells us which outer row
  // this corresponds to; it corresponds to range->ptr in MrrNextCallback(),
  // and points to the serialized outer row in BKAIterator's m_row array.
  BufferRow *rec_ptr = nullptr;
  do {
    int error =
        m_file->ha_multi_range_read_next(pointer_cast<char **>(&rec_ptr));
    if (error != 0) {
      return HandleError(error);
    }

    // NDB never calls mrr_funcs.skip_record(), so we need to recheck here.
    // See bug #30594210.
  } while (m_join_type == JoinType::SEMI && RowHasBeenRead(rec_ptr));

  LoadIntoTableBuffers(m_outer_input_tables,
                       pointer_cast<const uchar *>(rec_ptr->data()));
  // ★ 反序列化外层行回全局 record buffer，与刚读出的内层行拼成一条 join 结果

  m_last_row_returned = rec_ptr;   // 供 MarkLastRowAsRead() 用

  return 0;
}
```

> ⚠️ **`MrrSkipIndexTuple()` 是死声明**：`bka_iterator.h:377` 有声明，但 `.cc` 无定义、全仓库无调用，写文档/改代码都不要引用它。

### 3.3 DS-MRR：rowid 重排，磁盘顺序 sweep（`handler.cc:6766`）

**`DsMrr_impl::dsmrr_fill_buffer()` 完整源码**（`sql/handler.cc:6767`；⚠️ 不是 `:6766`）：

```cpp
int DsMrr_impl::dsmrr_fill_buffer() {
  char *range_info;
  int res = 0;
  DBUG_TRACE;
  assert(rowids_buf < rowids_buf_end);

  /*
    Set key_read to true since we only read fields from the index.
    This ensures that any virtual columns are read from index and are not
    attempted to be evaluated from base columns.
    (Do not use TABLE::set_keyread() since the MRR implementation operates
    with two handler objects, and set_keyread() would manipulate the keyread
    property of the wrong handler. ...)
  */
  bool table_keyread_save = table->key_read;
  table->key_read = true;

  rowids_buf_cur = rowids_buf;
  /*
    Do not use ha_multi_range_read_next() as it would call the engine's
    overridden multi_range_read_next() but the default implementation is wanted.
  */
  while ((rowids_buf_cur < rowids_buf_end) &&
         !(res = h2->handler::multi_range_read_next(&range_info))) {
    //                    ★★ 注意：用的是第二个 handler h2，且不走虚函数覆盖
    /* Put rowid, or {rowid, range_id} pair into the buffer */
    h2->position(table->record[0]);
    memcpy(rowids_buf_cur, h2->ref, h2->ref_length);
    rowids_buf_cur += h2->ref_length;

    if (is_mrr_assoc) {
      memcpy(rowids_buf_cur, &range_info, sizeof(void *));
      rowids_buf_cur += sizeof(void *);
    }
  }

  // Restore key_read since the next read operation might read complete rows
  table->key_read = table_keyread_save;

  if (res && res != HA_ERR_END_OF_FILE) return res;
  dsmrr_eof = (res == HA_ERR_END_OF_FILE);

  /* Sort the buffer contents by rowid */
  uint elem_size = h->ref_length + (int)is_mrr_assoc * sizeof(void *);
  assert((rowids_buf_cur - rowids_buf) % elem_size == 0);
  // ★ elem_size = ref_length + is_mrr_assoc * sizeof(void*)
  //   is_mrr_assoc = 有 HA_MRR_NO_ASSOCIATION 就为 0（即"不需要回传 range 关联"）

  // Store the handler in a thread local variable so that it is available in the
  // stateless comparator passed to qsort.
  thread_local const handler *current_handler;
  current_handler = h;

  qsort(rowids_buf, (rowids_buf_cur - rowids_buf) / elem_size, elem_size,
        [](const void *a, const void *b) {
          return current_handler->cmp_ref(
              static_cast<const unsigned char *>(a),
              static_cast<const unsigned char *>(b));
        });
  // ★★ 按 rowid 排序 —— BKA 的全部收益就在这一行

  rowids_buf_last = rowids_buf_cur;
  rowids_buf_cur = rowids_buf;
  return 0;
}
```

> ★ `thread_local const handler *current_handler` 是个巧妙的 workaround：`qsort` 的比较器是**无状态函数指针**，拿不到 `this`，所以用 thread-local 把 handler 塞进去。

`dsmrr_next`（`:6834`）按排序好的 rowid 用 `ha_rnd_pos` 回表。**BKA 的关键收益**：内层随机回表被重排成按 rowid 顺序的磁盘 sweep。

**握手全过程**（逐点实测）：

| 步 | 位置 | 动作 |
|---|---|---|
| 1 | `MultiRangeRowIterator::Init`（`bka_iterator.cc:335`） | 注册 `RANGE_SEQ_IF {MrrInitCallbackThunk, MrrNextCallbackThunk, skip_record}`，调 `multi_range_read_init(n_ranges, m_mrr_flags, &m_mrr_buffer)` |
| 2 | `DsMrr_impl::dsmrr_init`（`handler.cc:6550`） | 若引擎返回 `HA_MRR_USE_DEFAULT_IMPL \| HA_MRR_SORTED` 则**退回默认实现**（`:6561`） |
| 3 | 同上（`:6600`） | **克隆第二个 handler `h2`** 做索引扫描，并把 ICP 转移到它（`:6640`） |
| 4 | `dsmrr_fill_buffer`（`:6767`） | 收集 `{rowid, range_info}` 后 **qsort 按 rowid 排序** |

> ★ **为什么需要第二个 handler `h2`**：主 handler `h` 当前正用于**顺序读（rnd）**，而 MRR 需要独立的**索引扫描上下文**。所以 `dsmrr_init` 克隆一份 `h2` 专做索引扫描，收集完 rowid 后由 `h` 用 `ha_rnd_pos` 回表。上面 `dsmrr_fill_buffer` 里那句 `h2->handler::multi_range_read_next(...)` 用的是**显式限定调用**（`handler::` 而非虚调用），正是为了绕开引擎自己的 MRR 覆盖实现。

### 3.4 match flag 位图（替代旧 get_match_flag）

**完整源码**（`bka_iterator.cc:60` + `bka_iterator.h` 的 inline 方法）：

```cpp
// bka_iterator.cc:60
static bool NeedMatchFlags(JoinType join_type) {
  return join_type == JoinType::OUTER || join_type == JoinType::SEMI ||
         join_type == JoinType::ANTI;
  // ★ inner join 完全不需要（m_match_flag_buffer 恒为 nullptr）
}

static size_t BytesNeededForMatchFlags(size_t rows) {
  // One bit per row.
  return (rows + 7) / 8;      // ★ 紧凑位图，不是 bool 数组
}
```

```cpp
// bka_iterator.h:（MultiRangeRowIterator 的 inline 方法）
  /**
    Mark that the BKA iterator has seen the last row we returned from Read().
    (It could have been discarded by a FilterIterator before it reached them.)
    Will be a no-op for inner joins; see set_match_flag_buffer()..
   */
  void MarkLastRowAsRead() {
    if (m_match_flag_buffer != nullptr) {
      size_t row_number = std::distance(m_begin, m_last_row_returned);
      m_match_flag_buffer[row_number / 8] |= 1 << (row_number % 8);
    }
  }

  /**
    Check whether the given row has been marked as read
    (using MarkLastRowAsRead()) or not. Used internally when doing semijoins,
    and also by the BKAIterator when synthesizing NULL-complemented rows for
    outer joins or antijoins.
   */
  bool RowHasBeenRead(const hash_join_buffer::BufferRow *row) const {
    assert(m_match_flag_buffer != nullptr);
    size_t row_number = std::distance(m_begin, row);
    return m_match_flag_buffer[row_number / 8] & (1 << (row_number % 8));
  }

// bka_iterator.cc:427
bool MultiRangeRowIterator::MrrSkipRecord(char *range_info) {
  BufferRow *rec_ptr = pointer_cast<BufferRow *>(range_info);
  return RowHasBeenRead(rec_ptr);
  // ★ semijoin：若这行外层行**已经**被匹配过 ⇒ 返回 true ⇒ 跳过
}
```

> ★ **两个坑**：
> 1. `MarkLastRowAsRead()` 是 BKA 与 MRR 之间**唯一的回执通道**。若中间夹了 `FilterIterator` 把行过滤掉，MRR 端不会知道——这正是 `QueryMixesOuterBKAAndBNL()` 存在的原因。
> 2. NDB **不调** `skip_record` 回调，所以 `MultiRangeRowIterator::Read()` 里用 `do...while` 手工重做 semijoin 去重（bug #30594210）。

**行序列化格式**（`sql/pack_rows.cc`，每表）：`[NULL-row flag（仅可空表）] [NULL flags（null_bytes）] [各非 NULL 列的 pack 数据] [rowid（可选）]`

```cpp
// sql/pack_rows.cc:243
const uchar *LoadIntoTableBuffers(const TableCollection &tables,
                                  const uchar *ptr) {
  for (const Table &tbl : tables.tables()) {
    TABLE *table = tbl.table;

    const NullRowFlag null_row_flag = table->is_nullable()
                                          ? static_cast<NullRowFlag>(*ptr++)
                                          : NullRowFlag::kNotNull;
    ...
    // If the NULL row flag is set, it may override the NULL flags for the
    // columns. ... so clear the NULL row flag when restoring the row.
    table->reset_null_row();

    if (tbl.copy_null_flags) {
      memcpy(table->null_flags, ptr, table->s->null_bytes);
      ptr += table->s->null_bytes;
    }

    // Load all non-null column values.
    if (null_row_flag != NullRowFlag::kNullWithoutData) {
      for (const Column &column : tbl.columns) {
        if (!column.field->is_null()) {
          ptr = column.field->unpack(ptr);
        }
      }
    }
    ...
    if (tables.store_rowids() && ShouldCopyRowId(table)) {
      memcpy(table->file->ref, ptr, table->file->ref_length);
      ptr += table->file->ref_length;
    }
  }
  return ptr;
}
```

> ★ **注意：只存非 NULL 列**——但 `ComputeRowSizeUpperBound()` 把所有列都算进去（注释明说："we end up reserving a bit too much space ... That is more welcome than having to call Field::is_null() for every column in every row"）。这是**用内存换分支预测**的取舍。

---

## 四、hash join 的 build / probe / spill

### 4.1 三种模式 + 七态状态机

```cpp
// sql/iterators/hash_join_iterator.h:639
  // The "type" of hash join we are executing. We currently have three different
  // types of hash join:
  // - In memory: We do everything in memory without any refills of the hash
  //   table. Each input is read only once, and nothing is written to disk.
  // - Spill to disk: If the build input does not fit in memory, we write both
  //   inputs out to a set of chunk files. Both inputs are partitioned using a
  //   hash function over the join attribute, ensuring that matching rows can be
  //   found in the same set of chunk files. Each pair of chunk file is then
  //   processed as an in-memory hash join.
  // - In memory with hash table refill: This is enabled if we are not allowed
  //   to spill to disk, and the build input does not fit in memory. We read as
  //   much as possible from the build input into the hash table. We then read
  //   the entire probe input, probing for matching rows in the hash table.
  //   When the probe input returns EOF, the hash table is refilled with the
  //   rows that did not fit the first time. The entire probe input is read
  //   again, and this is repeated until the entire build input is consumed.
  enum class HashJoinType {
    IN_MEMORY,
    SPILL_TO_DISK,
    IN_MEMORY_WITH_HASH_TABLE_REFILL
  };
  HashJoinType m_hash_join_type{HashJoinType::IN_MEMORY};
```

```cpp
// sql/iterators/hash_join_iterator.h:478 —— ★ 真名是 State，不是 HashJoinIteratorState
  enum class State {
    // We are reading a row from the probe input, where the row comes from
    // the iterator.
    READING_ROW_FROM_PROBE_ITERATOR,
    // We are reading a row from the probe input, where the row comes from a
    // chunk file.
    READING_ROW_FROM_PROBE_CHUNK_FILE,
    // We are reading a row from the probe input, where the row comes from a
    // probe row saving file.
    READING_ROW_FROM_PROBE_ROW_SAVING_FILE,
    // The iterator is moving to the next pair of chunk files, where the chunk
    // file from the build input will be loaded into the hash table.
    LOADING_NEXT_CHUNK_PAIR,
    // We are reading the first row returned from the hash table lookup that
    // also passes extra conditions.
    READING_FIRST_ROW_FROM_HASH_TABLE,
    // We are reading the remaining rows returned from the hash table lookup.
    READING_FROM_HASH_TABLE,
    // No more rows, both inputs are empty.
    END_OF_ROWS
  };
```

> ★ 注意**没有** `enum class ProbeRow`、`ReadProbeRowFromDisk()`——都不存在。

### 4.2 build：StoreRow 插哈希表

**`HashJoinRowBuffer` 类定义**（`sql/iterators/hash_join_buffer.h:119`）：

```cpp
enum class StoreRowResult { ROW_STORED, BUFFER_FULL, FATAL_ERROR };

class HashJoinRowBuffer {
 public:
  StoreRowResult StoreRow(THD *thd, bool reject_duplicate_keys);
  std::optional<LinkedImmutableString> find(Key key) const;
  std::optional<LinkedImmutableString> first_row() const;
  LinkedImmutableString LastRowStored() const {
    assert(Initialized());
    return m_last_row_stored;
  }
  ...
 private:
  // The type of hash map in which the rows are stored.
  class HashMap;      // ★ 故意只在 .h 前向声明，避免第三方库头文件污染

  const std::vector<HashJoinCondition> m_join_conditions;
  const pack_rows::TableCollection m_tables;

  // The MEM_ROOT on which all of the hash table keys and values are allocated.
  // The actual hash map is on the regular heap.
  MEM_ROOT m_mem_root;

  // A MEM_ROOT used only for storing the final row (possibly both key and
  // value). The code assumes fairly deeply that inserting a row never fails, so
  // when m_mem_root goes full ..., we allocate the very last row on
  // this MEM_ROOT and the signal fullness so that we can start spilling to disk.
  MEM_ROOT m_overflow_mem_root;     // ★★ "最后一行专用"逃生区

  std::unique_ptr<HashMap> m_hash_map;
  String m_buffer;                  // 拼 join key 的暂存 buffer，只增不减
  size_t m_row_size_upper_bound;
  const size_t m_max_mem_available;
  LinkedImmutableString m_last_row_stored{nullptr};

  LinkedImmutableString StoreLinkedImmutableStringFromTableBuffers(
      LinkedImmutableString next_ptr, bool *full);
};
```

> ⚠️ **勘误（月报 2019/11 已过时）**：哈希表**不是** `std::unordered_multimap`（源码零命中），而是第三方库 `ankerl::unordered_dense` v4.4.0 的 `segmented_map`。真实定义在 `.cc:97`：

```cpp
// sql/iterators/hash_join_buffer.cc:33
#include <ankerl/unordered_dense.h>

// :51 —— 相等比较器（异质查找）
class KeyEquals {
 public:
  // This is a marker from C++17 that signals to the container that
  // operator() can be called with arguments of which one of the types
  // differs from the container's key type (ImmutableStringWithLength),
  // and thus enables map.find(Key). The type itself does not matter.
  using is_transparent = void;      // ★ 允许用 string_view 直接查，省一次 key 构造
  bool operator()(const Key &str1,
                  const ImmutableStringWithLength &other) const {
    return str1 == other.Decode();
  }
  bool operator()(const ImmutableStringWithLength &str1,
                  const ImmutableStringWithLength &str2) const {
    return str1 == str2;
  }
};

// :70 —— 哈希器
class KeyHasher {
 public:
  using is_transparent = void;
  // This is a marker telling ankerl::unordered_dense that the hash function has
  // good quality.
  using is_avalanching = void;      // ★ 告诉库"我的 hash 已雪崩"，省掉库自己的二次混合

  uint64_t operator()(Key key) const {
    return ankerl::unordered_dense::hash<Key>()(key);   // ★ 不是 xxHash64
  }
  uint64_t operator()(ImmutableStringWithLength key) const {
    return operator()(key.Decode());
  }
};

// :97
// A wrapper class around ankerl::unordered_dense::segmented_map, so that it can
// be forward-declared in the header file. This is done to limit the number of
// files that include directly or indirectly headers from the third-party
// library.
class HashJoinRowBuffer::HashMap
    : public ankerl::unordered_dense::segmented_map<ImmutableStringWithLength,
                                                    LinkedImmutableString,
                                                    KeyHasher, KeyEquals> {
 public:
  using ankerl::unordered_dense::segmented_map<ImmutableStringWithLength,
                                               LinkedImmutableString, KeyHasher,
                                               KeyEquals>::segmented_map;
};
```

> ★ **xxHash64（`MY_XXH64`）只用于 chunk 分区**，不用于哈希表内部。两者刻意不同（见 §6.2）。

**构造函数与 `Init()`**（`hash_join_buffer.cc:166` / `:184`）：

```cpp
HashJoinRowBuffer::HashJoinRowBuffer(
    TableCollection tables, std::vector<HashJoinCondition> join_conditions,
    size_t max_mem_available)
    : m_join_conditions(std::move(join_conditions)),
      m_tables(std::move(tables)),
      m_mem_root(key_memory_hash_join, 16384 /* 16 kB */),   // ★ 主块 16 kB
      m_overflow_mem_root(key_memory_hash_join, 256),        // ★ overflow 块 256 B
      m_hash_map(nullptr),
      m_max_mem_available(
          std::max<size_t>(max_mem_available, 16384 /* 16 kB */)) {  // ★ 下限 16 kB
  // Limit is being applied only after the first row.
  m_mem_root.set_max_capacity(0);
  // ★ 0 = 无限制，故意的：保证第一行一定插得进去（先插后查），
  //   真正的额度在 StoreRow() 里第一次 emplace 成功后才动态算出来
}

bool HashJoinRowBuffer::Init() {
  if (m_hash_map != nullptr) {
    // Reset the unique_ptr, so that the hash map destructors are called before
    // clearing the MEM_ROOT.
    m_hash_map.reset(nullptr);
    m_mem_root.Clear();
    m_mem_root.set_max_capacity(0);
    m_overflow_mem_root.ClearForReuse();
  }

  // NOTE: Will be ignored and re-calculated if there are any blobs in the table.
  m_row_size_upper_bound = ComputeRowSizeUpperBound(m_tables);

  m_hash_map.reset(new HashMap());
  if (m_hash_map == nullptr) {
    my_error(ER_OUTOFMEMORY, MYF(ME_FATALERROR), sizeof(*m_hash_map));
    return true;
  }

  m_last_row_stored = LinkedImmutableString{nullptr};
  return false;
}
```

**`StoreRow()` 完整源码**（`hash_join_buffer.cc:212`）：

```cpp
StoreRowResult HashJoinRowBuffer::StoreRow(THD *thd,
                                           bool reject_duplicate_keys) {
  bool full = false;

  // Make the key from the join conditions.
  m_buffer.length(0);                       // ★ m_buffer 只增不减
  for (const HashJoinCondition &hash_join_condition : m_join_conditions) {
    bool null_in_join_condition =
        hash_join_condition.join_condition()->append_join_key_for_hash_join(
            thd, m_tables.tables_bitmap(), hash_join_condition,
            m_join_conditions.size() > 1, &m_buffer);   // is_multi_column_key

    if (thd->is_error()) {
      return StoreRowResult::FATAL_ERROR;
    }

    if (null_in_join_condition) {
      // One of the components of the join key had a NULL value, and
      // that component was part of an equality predicate (=), *not* a
      // NULL-safe equality predicate, so it can never match a row in
      // the other table. There's no need to store the row in the hash
      // table. Skip it.
      return StoreRowResult::ROW_STORED;    // ★ "NULL 永不匹配"①：build 端丢行
    }
  }

  // Store the key in the MEM_ROOT. Note that we will only commit the memory
  // usage for it if the key was a new one (see the call to emplace() below)..
  const size_t required_key_bytes =
      ImmutableStringWithLength::RequiredBytesForEncode(m_buffer.length());
  ImmutableStringWithLength key;

  std::pair<char *, char *> block = m_mem_root.Peek();   // ★ 偷看剩余空间，不分配
  if (static_cast<size_t>(block.second - block.first) < required_key_bytes) {
    m_mem_root.ForceNewBlock(required_key_bytes);
    block = m_mem_root.Peek();
  }
  size_t bytes_to_commit = 0;
  if (static_cast<size_t>(block.second - block.first) >= required_key_bytes) {
    char *ptr = block.first;
    key = ImmutableStringWithLength::Encode(m_buffer.ptr(), m_buffer.length(),
                                            &ptr);
    bytes_to_commit = ptr - block.first;
    // ★ 先"写好"，等 emplace 成功才 RawCommit；key 重复则这 N 字节直接作废
  } else {
    char *ptr =
        pointer_cast<char *>(m_overflow_mem_root.Alloc(required_key_bytes));
    if (ptr == nullptr) return StoreRowResult::FATAL_ERROR;
    key = ImmutableStringWithLength::Encode(m_buffer.ptr(), m_buffer.length(),
                                            &ptr);
    // Keep bytes_to_commit == 0; the value is already committed.
  }

  std::pair<HashMap::iterator, bool> key_it_and_inserted;
  try {
    key_it_and_inserted =
        m_hash_map->emplace(key, LinkedImmutableString{nullptr});
  } catch (const std::overflow_error &) {
    // This can only happen if the hash function is extremely bad
    // (should never happen in practice).
    return StoreRowResult::FATAL_ERROR;
  }
  LinkedImmutableString next_ptr{nullptr};
  if (key_it_and_inserted.second) {         // ★ 新 key
    // We inserted an element, so the hash table may have grown.
    // Update the capacity available for the MEM_ROOT; our total may
    // have gone slightly over already, and if so, we will signal
    // that and immediately start spilling to disk.
    const size_t bytes_used =
        m_hash_map->bucket_count() * sizeof(HashMap::bucket_type) +
        m_hash_map->values().capacity() *
            sizeof(HashMap::value_container_type::value_type);
    // ★★ MEM_ROOT 容量是**动态重算**的：join_buffer_size 要先扣掉
    //    hash map 自身在**普通堆**上占的字节（桶数组 + 元素数组）
    if (bytes_used >= m_max_mem_available) {
      // 0 means no limit, so set the minimum possible limit.
      m_mem_root.set_max_capacity(1);       // ★ 额度被桶吃光 → 逼下一行走 overflow
      full = true;
    } else {
      m_mem_root.set_max_capacity(m_max_mem_available - bytes_used);
    }

    m_mem_root.RawCommit(bytes_to_commit);  // ★ 到这一刻才真正"落账"
  } else {                                  // ★ key 冲突
    if (reject_duplicate_keys) {
      return StoreRowResult::ROW_STORED;    // ★ semi/anti 无 extra cond：只留第一行
    }
    // Put the new value in the hash bucket, but keep track of what the old
    // one was; it will be our “next” pointer.
    next_ptr = key_it_and_inserted.first->second;   // ★ 冲突头插
  }

  // Save the contents of all columns marked for reading.
  m_last_row_stored = key_it_and_inserted.first->second =
      StoreLinkedImmutableStringFromTableBuffers(next_ptr, &full);
  if (m_last_row_stored == nullptr) {
    return StoreRowResult::FATAL_ERROR;
  } else if (full) {
    return StoreRowResult::BUFFER_FULL;
  } else {
    return StoreRowResult::ROW_STORED;
  }
}
```

> ★★ **"内存满了"的两个判据**：① 新 key 时 `bytes_used >= m_max_mem_available` → `set_max_capacity(1)`；② `StoreLinkedImmutableStringFromTableBuffers()` 走了 overflow 块 → `*full = true`。**溢出检查发生在插入之后**（注释明说），所以"至少能插进第一行"是不变式。

`find()`（`:322`）与 `LinkedImmutableString` 的链表结构：

```cpp
std::optional<LinkedImmutableString> HashJoinRowBuffer::find(Key key) const {
  const auto it = m_hash_map->find(key);   // 异质查找，无 key 构造开销
  if (it == m_hash_map->end()) return {};
  return it->second;                       // 只返回链表头；后续靠 Decode().next 遍历
}
```

```cpp
// sql/immutable_string.h:173 —— value 不存长度，存 zigzag-Varint 的"next 指针差"
/**
 * LinkedImmutableString is designed for storing rows (values) in hash join. It
 * does not need a length, since it is implicit from the contents; however,
 * since there might be multiple values with the same key, we simulate a
 * multimap by having a “next” pointer. (Normally, linked lists are a bad idea
 * due to pointer chasing, but here, we're doing so much work for each value
 * that the overhead disappears into the noise.)
 *
 * As the next pointer is usually be very close in memory to ourselves
 * (nearly all rows are stored in the same MEM_ROOT), we don't need to store
 * the entire pointer; instead, we store the difference between the start of
 * this string and the next pointer, as a zigzag-encoded Varint128. As with
 * the length in ImmutableStringWithLength, this typically saves 6–7 bytes
 * for each value. The special value of 0 means that there is no next pointer
 * (ie., it is nullptr), as that would otherwise be an infinite loop.
 */
```

> ★ 这是**用"指针差"代替 8 字节指针**的空间优化：因为绝大多数行都在同一个 MEM_ROOT 里，差值很小，zigzag-Varint 编码后通常只占 1-3 字节，**每行省 6~7 字节**。这就是"自己串链表模拟 multimap"而不是真的用 multimap 的原因。

### 4.3 probe：LookupProbeRowInHashTable（`hash_join_iterator.cc:848`）

```cpp
void HashJoinIterator::LookupProbeRowInHashTable() {
  if (m_join_conditions.empty()) {
    // Skip the call to find() in case we don't have any join conditions.
    m_current_row =
        m_row_buffer.first_row().value_or(LinkedImmutableString{nullptr});
    m_state = State::READING_FIRST_ROW_FROM_HASH_TABLE;
    return;
  }

  // Extract the join key from the probe input, and use that key as the lookup
  // key in the hash table.
  bool null_in_join_key = ConstructJoinKey(
      thd(), m_join_conditions, m_probe_input_tables.tables_bitmap(),
      &m_temporary_row_and_join_key_buffer);

  if (null_in_join_key) {
    if (m_join_type == JoinType::ANTI || m_join_type == JoinType::OUTER) {
      // SQL NULL was found, and we will never find a matching row in the hash
      // table. Let us indicate that, so that a null-complemented row is
      // returned.
      m_current_row = LinkedImmutableString{nullptr};   // ★ "NULL 永不匹配"③
      m_state = State::READING_FIRST_ROW_FROM_HASH_TABLE;
    } else {
      SetReadingProbeRowState();                        // ★ inner/semi 直接跳过
    }
    return;
  }

  hash_join_buffer::Key key{m_temporary_row_and_join_key_buffer.ptr(),
                            m_temporary_row_and_join_key_buffer.length()};

  m_current_row =
      m_row_buffer.find(key).value_or(LinkedImmutableString{nullptr});

  m_state = State::READING_FIRST_ROW_FROM_HASH_TABLE;
}
```

**`ReadNextJoinedRowFromHashTable()`**（`:944`）——沿链表取行 + 过 extra 条件：

```cpp
int HashJoinIterator::ReadNextJoinedRowFromHashTable() {
  int res;
  bool passes_extra_conditions = false;
  do {
    res = ReadJoinedRow();        // 把 m_current_row 解回 build 侧 record buffer
    assert(res == 0 || res == -1);

    // Evaluate any extra conditions that are attached to this iterator before
    // we return a row.
    if (res == 0) {
      passes_extra_conditions = JoinedRowPassesExtraConditions();
      // ★ 非等值条件（extra condition）在探测**之后**用 val_int() 求值
      if (thd()->is_error() || thd()->killed) return 1;

      if (!passes_extra_conditions) {
        m_current_row = m_current_row.Decode().next;
        // ★ 沿 next 走同 key 的下一行（伪 multimap 的迭代器）
      }
    }
  } while (res == 0 && !passes_extra_conditions);

  if (WriteProbeRowToDiskIfApplicable()) return 1;

  if (res == -1) {
    // ... antijoin and outer join should output the last row read from the
    // probe input together with a NULL-complemented row from the build input ...
    bool return_null_complemented_row = false;
    if ((on_disk_hash_join() && m_current_chunk == -1) ||
        m_write_to_probe_row_saving) {
      return_null_complemented_row = false;   // ★ on-disk 第一轮不能下结论
    } else if (m_join_type == JoinType::ANTI) {
      return_null_complemented_row = true;
    } else if (m_join_type == JoinType::OUTER &&
               m_state == State::READING_FIRST_ROW_FROM_HASH_TABLE &&
               !m_probe_row_match_flag) {     // ★ match flag 来自 chunk 文件前缀
      return_null_complemented_row = true;
    }

    SetReadingProbeRowState();

    if (return_null_complemented_row) {
      m_build_input->SetNullRowFlag(true);
      return 0;
    }
    return -1;
  }

  switch (m_join_type) {
    case JoinType::SEMI:
      SetReadingProbeRowState();      // 只吐第一个匹配，换下一个 probe 行
      break;
    case JoinType::ANTI:
      SetReadingProbeRowState();
      return -1;                      // 有匹配 ⇒ 不输出
    case JoinType::OUTER:
    case JoinType::INNER:
      m_state = State::READING_FROM_HASH_TABLE;   // 继续吐同 key 剩余行
      break;
    ...
  }

  m_current_row = m_current_row.Decode().next;
  return 0;
}

bool HashJoinIterator::JoinedRowPassesExtraConditions() const {
  if (m_extra_condition != nullptr) {
    return m_extra_condition->val_int() != 0;
  }
  return true;
}
```

### 4.4 spill：两条分流路径

**`BuildHashTable()` 里的分流点**（`hash_join_iterator.cc:410`）——这是 BUFFER_FULL 之后的关键分支：

```cpp
      case hash_join_buffer::StoreRowResult::BUFFER_FULL: {
        // The row buffer is full, so start spilling to disk (if allowed). Note
        // that the row buffer checks for OOM _after_ the row was inserted, so
        // we should always manage to insert at least one row.
        assert(!m_row_buffer.empty());

        // If we are not allowed to spill to disk, just go on to reading from
        // the probe iterator.
        if (!m_allow_spill_to_disk) {                  // ★ 分支 1：禁止 spill
          if (m_join_type != JoinType::INNER) {
            // Enable probe row saving, so that unmatched probe rows are written
            // to the probe row saving file. After the next refill of the hash
            // table, we will read rows from the probe row saving file, ensuring
            // that we only read unmatched probe rows.
            InitWritingToProbeRowSavingFile();
          }
          SetReadingProbeRowState();
          return false;
          // ★ 注意：此时 m_hash_join_type 仍是 IN_MEMORY，
          //   等 ReadRowFromProbeIterator() 里 probe 耗尽才切成 REFILL
        }

        if (InitializeChunkFiles(                       // ★ 分支 2：spill
                m_estimated_build_rows, m_row_buffer.size(), kMaxChunks,
                m_probe_input_tables, m_build_input_tables,
                /*include_match_flag_for_probe=*/m_join_type == JoinType::OUTER,
                &m_chunk_files_on_disk)) {
          return true;
        }

        // Write out the remaining rows from the build input out to chunk files.
        // ... we don't want to waste the rows that we already have stored in
        // memory. ... We never write out rows with NULL in condition ...
        if (WriteRowsToChunks(thd(), m_build_input.get(), m_build_input_tables,
                              m_join_conditions, kChunkPartitioningHashSeed,
                              &m_chunk_files_on_disk,
                              true /* write_to_build_chunks */,
                              false /* write_rows_with_null_in_join_key */,
                              m_tables_to_get_rowid_for,
                              &m_temporary_row_and_join_key_buffer)) {
          return true;
        }

        // Flush and position all chunk files from the build input at the beginning.
        for (ChunkPair &chunk_pair : m_chunk_files_on_disk) {
          if (chunk_pair.build_chunk.Rewind()) return true;
        }
        SetReadingProbeRowState();
        return false;
      }
```

**REFILL 模式的确立点**（`ReadRowFromProbeIterator`，`:673`）：

```cpp
  // The probe iterator is out of rows. We may be in three different situations
  // here (ordered from most common to less common):
  // 1. The build input is also empty, and the join is done. ...
  // 2. We have degraded into an on-disk hash join, and we will now start
  //    reading from chunk files on disk.
  // 3. The build input is not empty, and we have not degraded into an on-disk
  //    hash join (i.e. we were not allowed due to a LIMIT in the query),
  //    re-populate the hash table with the remaining rows from the build input.
  if (m_allow_spill_to_disk) {
    m_hash_join_type = HashJoinType::SPILL_TO_DISK;
    m_state = State::LOADING_NEXT_CHUNK_PAIR;
    return false;
  }

  m_hash_join_type = HashJoinType::IN_MEMORY_WITH_HASH_TABLE_REFILL;   // ★ 此处确立
  if (m_write_to_probe_row_saving) {
    // ... We must swap the probe row saving writing and probe row saving
    // reading file _before_ calling BuildHashTable, since BuildHashTable may
    // initialize (and thus clear) the probe row saving write file ...
    if (InitReadingFromProbeRowSavingFile()) return true;
  }

  if (BuildHashTable()) return true;      // ★ 重填 hash table

  switch (m_state) {
    case State::END_OF_ROWS: return false;
    case State::READING_ROW_FROM_PROBE_ITERATOR:
      return InitProbeIterator();          // ★ 重扫 probe 迭代器
    ...
  }
```

**`ReadNextHashJoinChunk()`**（`:565`）——on-disk 状态机：

```cpp
bool HashJoinIterator::ReadNextHashJoinChunk() {
  bool move_to_next_chunk = false;
  if (m_current_chunk == -1) {
    move_to_next_chunk = true;
  } else if (m_build_chunk_current_row >=
             m_chunk_files_on_disk[m_current_chunk].build_chunk.num_rows()) {
    move_to_next_chunk = true;                    // build chunk 读完了才换
  } else if (m_chunk_files_on_disk[m_current_chunk].probe_chunk.num_rows() == 0) {
    move_to_next_chunk = true;                    // probe chunk 是空的
  }

  if (move_to_next_chunk) {
    m_current_chunk++;
    m_build_chunk_current_row = 0;
    m_read_from_probe_row_saving = false;
  }

  if (m_current_chunk == static_cast<int>(m_chunk_files_on_disk.size())) {
    m_state = State::END_OF_ROWS;
    return false;
  }

  if (InitRowBuffer()) return true;               // 每对 chunk 都清空重建 hash table

  HashJoinChunk &build_chunk =
      m_chunk_files_on_disk[m_current_chunk].build_chunk;

  const bool reject_duplicate_keys = RejectDuplicateKeys();
  for (; m_build_chunk_current_row < build_chunk.num_rows();
       ++m_build_chunk_current_row) {
    if (build_chunk.LoadRowFromChunk(&m_temporary_row_and_join_key_buffer,
                                     /*matched=*/nullptr)) {
      return true;
    }

    hash_join_buffer::StoreRowResult store_row_result =
        m_row_buffer.StoreRow(thd(), reject_duplicate_keys);

    if (store_row_result == hash_join_buffer::StoreRowResult::BUFFER_FULL) {
      // ... the row buffer checks for OOM _after_ the row was inserted ...
      ++m_build_chunk_current_row;   // ★ 最后一行确实存进去了，手动 +1 再 break
      break;                         // ★ 不换 chunk 对：先 probe，回来继续装同一个 build chunk
    } else if (store_row_result == hash_join_buffer::StoreRowResult::FATAL_ERROR) {
      my_error(ER_OUTOFMEMORY, MYF(ME_FATALERROR), thd()->variables.join_buff_size);
      return true;
    }
  }

  // Prepare to do a lookup in the hash table for all rows from the probe chunk.
  if (m_chunk_files_on_disk[m_current_chunk].probe_chunk.Rewind()) return true;
  m_probe_chunk_current_row = 0;
  SetReadingProbeRowState();

  if (m_build_chunk_current_row < build_chunk.num_rows() &&
      m_join_type != JoinType::INNER) {
    // The build chunk did not fit into memory ... we can end up outputting the
    // same probe row twice if the probe phase finds a match in both iterations
    // through the hash table. By enabling probe row saving, unmatched probe rows
    // are written to a probe row saving file ...
    InitWritingToProbeRowSavingFile();
  } else {
    m_write_to_probe_row_saving = false;
  }

  return false;
}
```

**`Read()` 总调度**（`:1046`）：

```cpp
int HashJoinIterator::Read() {
  for (;;) {
    if (thd()->killed) { thd()->send_kill_message(); return 1; }

    switch (m_state) {
      case State::LOADING_NEXT_CHUNK_PAIR:
        if (ReadNextHashJoinChunk()) return 1;
        break;
      case State::READING_ROW_FROM_PROBE_ITERATOR:
        if (ReadRowFromProbeIterator()) return 1;
        break;
      case State::READING_ROW_FROM_PROBE_CHUNK_FILE:
        if (ReadRowFromProbeChunkFile()) return 1;
        break;
      case State::READING_ROW_FROM_PROBE_ROW_SAVING_FILE:
        if (ReadRowFromProbeRowSavingFile()) return 1;
        break;
      case State::READING_FIRST_ROW_FROM_HASH_TABLE:
      case State::READING_FROM_HASH_TABLE: {
        const int res = ReadNextJoinedRowFromHashTable();
        if (res == 0) return 0;          // A joined row is ready
        if (res == -1) continue;         // 读下一个 probe row
        assert(res == 1);
        return res;
      }
      case State::END_OF_ROWS:
        return -1;
    }
  }
}
```

**spill 步骤小结**：

1. build 内存满时 `InitializeChunkFiles` 创建 chunk 文件对（数量按"剩余行数 / 已装行数 × 0.9"估算，向上取到 2 的幂，上限 `kMaxChunks = 128`）。
2. 剩余 build 行按 **`MY_XXH64(key, kChunkPartitioningHashSeed)`** 分桶写 build chunk（与哈希表哈希刻意不同）。
3. probe 阶段：命中内存哈希表的同时，probe 行也按同样函数写 probe chunk（`WriteProbeRowToDiskIfApplicable`：inner/outer 全写；semi/anti 只写未命中行）。
4. probe 读完后 `LOADING_NEXT_CHUNK_PAIR`：把对应 build chunk 读进哈希表，probe chunk 回绕，做经典 hash join（同一桶的两边必在同一对 chunk 里）。若 build chunk 一次装不完，则**不换 chunk 对**，先 probe 一遍、回来继续装。

---

## 五、setup_join_buffering 决策链

`setup_join_buffering`（`sql_optimizer.cc:3451`）逐表判定，最终按访问类型分派：

| 内层访问类型 | 结论 |
|-------------|------|
| `JT_ALL/JT_INDEX_SCAN/JT_RANGE/JT_INDEX_MERGE` | `ALG_BNL`（执行时变 hash join） |
| `JT_REF/JT_EQ_REF` 等索引点查 | `ALG_BKA`（需 MRR 可用、非默认实现、有 association） |
| 其它 / 禁用条件 | `ALG_NONE`（普通 nested loop） |

**完整源码**（`sql/sql_optimizer.cc:3451-3689`；★ 是 `static` 自由函数，不是 `JOIN` 成员）：

```cpp
static bool setup_join_buffering(JOIN_TAB *tab, JOIN *join,
                                 uint no_jbuf_after) {
  ASSERT_BEST_REF_IN_JOIN_ORDER(join);
  Cost_estimate cost;
  ha_rows rows;
  uint bufsz = 4096;
  uint join_cache_flags = 0;
  const bool bnl_on = hint_table_state(join->thd, tab->table_ref, BNL_HINT_ENUM,
                                       OPTIMIZER_SWITCH_BNL);
  const bool bka_on = hint_table_state(join->thd, tab->table_ref, BKA_HINT_ENUM,
                                       OPTIMIZER_SWITCH_BKA);

  const uint tableno = tab->idx();
  const uint tab_sj_strategy = tab->get_sj_strategy();

  /*
    If all key_parts are null_rejecting, the MultiRangeRowIterator will
    eliminate all NULL values in the key set, such that
    HA_MRR_NO_NULL_ENDPOINTS can be promised.
  */
  const key_part_map keypart_map = make_prev_keypart_map(tab->ref().key_parts);
  if (tab->ref().null_rejecting == keypart_map) {
    join_cache_flags |= HA_MRR_NO_NULL_ENDPOINTS;    // ★ 由优化器主动承诺
  }

  // Set preliminary join cache setting based on decision from greedy search
  if (!join->select_count)
    tab->set_use_join_cache(tab->position()->use_join_buffer
                                ? JOIN_CACHE::ALG_BNL
                                : JOIN_CACHE::ALG_NONE);

  if (tableno == join->const_tables) {               // ★ 禁用①：第一张非 const 表
    assert(tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
    return false;
  }

  if (!(bnl_on || bka_on)) goto no_join_cache;       // ★ 禁用②：两开关都 off

  if (tab->use_quick == QS_DYNAMIC_RANGE) goto no_join_cache;   // ★ 禁用③

  /* No join buffering if prevented by no_jbuf_after */
  if (tableno > no_jbuf_after) goto no_join_cache;   // ★ 禁用④

  /*
    An inner table of an outer join nest must not use join buffering if
    the first inner table of that outer join nest does not use join buffering.
  */
  if (tab->first_inner() != NO_PLAN_IDX && tab->first_inner() != tab->idx() &&
      !join->best_ref[tab->first_inner()]->use_join_cache())
    goto no_join_cache;                              // ★ 禁用⑤
  if (tab->first_upper() != NO_PLAN_IDX &&
      !join->best_ref[tab->first_upper()]->use_join_cache())
    goto no_join_cache;                              // ★ 禁用⑥

  if (tab->table()->pos_in_table_list->is_table_function() && tab->dependent)
    goto no_join_cache;                              // ★ 禁用⑦：dependent 表函数

  switch (tab_sj_strategy) {
    case SJ_OPT_FIRST_MATCH:
      // Use join cache with FirstMatch semi-join strategy only when semi-join
      // contains only one table. ...
      if (!(... using_secondary_storage_engine())) {
        if (!tab->is_single_inner_of_semi_join()) {
          assert(tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
          goto no_join_cache;                        // ★ 禁用⑧
        }
      }
      break;

    case SJ_OPT_LOOSE_SCAN:
      /* No join buffering if this semijoin nest is handled by loosescan */
      assert(tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
      goto no_join_cache;                            // ★ 禁用⑨

    case SJ_OPT_MATERIALIZE_LOOKUP:
    case SJ_OPT_MATERIALIZE_SCAN:
      // The Materialize strategies reuse the join_tab belonging to the
      // first table that was materialized. ...
      if (tab->first_sj_inner() == tab->idx()) {
        assert(tab->use_join_cache() == JOIN_CACHE::ALG_NONE);
        goto no_join_cache;                          // ★ 禁用⑩
      }
      break;

    case SJ_OPT_DUPS_WEEDOUT:
    // This strategy allows the same join buffering as a regular join would.
    case SJ_OPT_NONE:
      break;
  }

  /*
    The following code prevents use of join buffering when there is an
    outer join operation and first match semi-join strategy is used, because:
    ... a row in a cache has only one "match flag" ...
  */
  if (!(... using_secondary_storage_engine())) {
    if (tab_sj_strategy == SJ_OPT_FIRST_MATCH &&
        tab->is_inner_table_of_outer_join())
      goto no_join_cache;                            // ★ 禁用⑪：match flag 冲突
  }

  if (join->deps_of_remaining_lateral_derived_tables &
      (tab->prefix_tables() & ~tab->added_tables())) {
    /*
      ... consider a plan t1-t2-dt where dt is LATERAL and depends only on t1 ...
      if t2 could do join buffering: we buffer many rows of t1, then read one
      row of t2, fetch row#1 of t1 from cache, then materialize "dt" ... then
      fetch row#2 of t1 from cache, rematerialize "dt": it's very inefficient.
      So we forbid join buffering on t2 ...
    */
    goto no_join_cache;                              // ★ 禁用⑫：lateral derived
  }

  switch (tab->type()) {          /* ... 见 §2.1 ... */ }

no_join_cache:
  revise_cache_usage(tab);
  tab->set_use_join_cache(JOIN_CACHE::ALG_NONE);
  return false;
}
```

**禁用条件速查（12 条，按出现顺序）**：

| # | 条件 | 行号 |
|---|---|---|
| ① | 第一张非 const 表（`tableno == join->const_tables`） | 3482 |
| ② | `bnl_on` 与 `bka_on` 都 off | 3487 |
| ③ | `use_quick == QS_DYNAMIC_RANGE`（range checked for each record） | 3493 |
| ④ | `tableno > no_jbuf_after` | 3496 |
| ⑤ | 外连接 nest 的**非首个** inner 表，但首个 inner 表没用 join buffer | 3503 |
| ⑥ | 外连接 nest 的**首个** inner 表，但嵌入 nest 没用 join buffer | 3511 |
| ⑦ | dependent 的**表函数** | 3515 |
| ⑧ | FirstMatch semijoin 且 nest 不止一张表（secondary engine 除外） | 3519 |
| ⑨ | **LOOSE_SCAN** semijoin | 3535 |
| ⑩ | MATERIALIZE_LOOKUP/SCAN 且本表是 `first_sj_inner` | 3540 |
| ⑪ | FirstMatch + 本表是 outer join 的 inner 表（**match flag 冲突**） | 3582 |
| ⑫ | **lateral derived table** 依赖区间 | 3589 |
| ⑬ | 内表是**物化派生表/视图**（尚未实例化） | 3641 |
| ⑭ | `has_guarded_conds()`（IN 子查询 NULL-key 全表扫描风险） | 3661 |
| ⑮ | MRR 三连否决：`rows == HA_POS_ERROR` / `HA_MRR_USE_DEFAULT_IMPL` / `HA_MRR_NO_ASSOCIATION` | 3673 |

> ★ 注意 `HA_MRR_NO_NULL_ENDPOINTS` 是**优化器主动承诺**的（:3471），判据是 `tab->ref().null_rejecting == make_prev_keypart_map(key_parts)`——即 ref 的**所有 keypart 都 null-rejecting**。做不到就只能靠运行时 `impossible_null_ref()` 兜底。

默认开关（`sys_vars.cc:196`）：`block_nested_loop=on`、`hash_join=on`、`batched_key_access=off`。

---


## 六、深潜补充：chunk 格式、哈希表真相、DS-MRR 完整握手

### 6.1 HashJoinRowBuffer 的真实类型（月报已过时）

> ⚠️ 月报说的 `std::unordered_multimap` **已不准确**。8.0.39 实际是 `ankerl::unordered_dense::segmented_map`（`hash_join_buffer.cc:97-106`）——分段连续存储，缓存友好且支持 MEM_ROOT 分配。

构造（`:166-178`）：MEM_ROOT 初始块 **16KB**、overflow 256B；`m_max_mem_available = max(join_buffer_size, 16384)`（即**不低于 16KB**）；初始 `set_max_capacity(0)`。

`StoreRow`（`:284-294`）按 `bucket_count * sizeof(bucket) + values.capacity() * sizeof(value)` 重算 MEM_ROOT 上限——**内存上限是动态重算的，不是一次性算好**。

### 6.2 chunk 文件格式（`hash_join_chunk.cc:92-123`）

```
[可选 bool match flag][size_t data_length][payload]
                                          ^ 由 StoreFromTableBuffers 按 read_set 紧凑序列化
```

`Init`（`:72-80`）用 `open_cached_file(mysql_tmpdir, TEMP_PREFIX, DISK_BUFFER_SIZE)`（`DISK_BUFFER_SIZE = IO_SIZE*16 = 64KB`）；`Rewind` 用 `reinit_io_cache`。

**`HashJoinChunk` 的三个函数完整源码**（`sql/iterators/hash_join_chunk.cc:72` / `:92` / `:125`）：

```cpp
bool HashJoinChunk::Init(const TableCollection &tables, bool uses_match_flags) {
  m_tables = tables;
  m_file.file_key = key_file_hash_join;      // PSI 统计 key
  m_num_rows = 0;
  m_uses_match_flags = uses_match_flags;     // ★ 只有 OUTER join 的 probe chunk 为 true
  close_cached_file(&m_file);
  return open_cached_file(&m_file, mysql_tmpdir, TEMP_PREFIX, DISK_BUFFER_SIZE,
                          MYF(MY_WME));
}

bool HashJoinChunk::Rewind() {
  if (my_b_flush_io_cache(&m_file, /*need_append_buffer_lock=*/0) == -1 ||
      reinit_io_cache(&m_file, READ_CACHE, 0, false, false)) {
    my_error(ER_TEMP_FILE_WRITE_FAILURE, MYF(0));
    return true;
  }
  return false;
}

bool HashJoinChunk::WriteRowToChunk(String *buffer, bool matched) {
  if (StoreFromTableBuffers(m_tables, buffer)) {   // ★ 按 read_set 紧凑序列化
    my_error(ER_OUTOFMEMORY, MYF(ME_FATALERROR),
             ComputeRowSizeUpperBound(m_tables));
    return true;
  }

  if (m_uses_match_flags) {
    if (my_b_write(&m_file, pointer_cast<const uchar *>(&matched),
                   sizeof(matched)) != 0) {        // ① match flag：sizeof(bool)
      my_error(ER_TEMP_FILE_WRITE_FAILURE, MYF(0));
      return true;
    }
  }

  // Write out the length of the data.
  size_t data_length = buffer->length();
  if (my_b_write(&m_file, pointer_cast<const uchar *>(&data_length),
                 sizeof(data_length)) != 0) {      // ② data_length：sizeof(size_t) = 8
    my_error(ER_TEMP_FILE_WRITE_FAILURE, MYF(0));
    return true;
  }

  // ... and then write the actual data.
  if (my_b_write(&m_file, pointer_cast<uchar *>(buffer->ptr()), data_length) !=
      0) {                                          // ③ payload：变长
    my_error(ER_TEMP_FILE_WRITE_FAILURE, MYF(0));
    return true;
  }
  m_num_rows++;
  return false;
}

bool HashJoinChunk::LoadRowFromChunk(String *buffer, bool *matched) {
  if (m_uses_match_flags) {
    if (my_b_read(&m_file, pointer_cast<uchar *>(matched), sizeof(*matched)) != 0) {
      my_error(ER_TEMP_FILE_WRITE_FAILURE, MYF(0));
      return true;
    }
  }

  size_t row_length;
  if (my_b_read(&m_file, pointer_cast<uchar *>(&row_length),
                sizeof(row_length)) != 0) { ... }

  if (buffer->reserve(row_length)) { ... }
  buffer->length(row_length);
  if (my_b_read(&m_file, pointer_cast<uchar *>(buffer->ptr()), row_length) != 0) { ... }

  hash_join_buffer::LoadBufferRowIntoTableBuffers(
      m_tables, {buffer->ptr(), buffer->length()});   // ★ 解包回各表 record[0]
  return false;
}
```

**分桶：`WriteRowsToChunks` 的核心**（`hash_join_iterator.cc:288`）：

```cpp
static bool WriteRowToChunk(
    THD *thd, Mem_root_array<ChunkPair> *chunks, bool write_to_build_chunk,
    const pack_rows::TableCollection &tables,
    const Prealloced_array<HashJoinCondition, 4> &join_conditions,
    const uint32 xxhash_seed, bool row_has_match,
    bool store_row_with_null_in_join_key, String *join_key_and_row_buffer) {
  assert(!thd->is_error());
  bool null_in_join_key = ConstructJoinKey(
      thd, join_conditions, tables.tables_bitmap(), join_key_and_row_buffer);
  if (thd->is_error()) return true;

  if (null_in_join_key && !store_row_with_null_in_join_key) {
    // NULL values will never match in a inner join or a semijoin. The optimizer
    // will often set up a NULL filter for inner joins, but not in all cases. So
    // we must handle this gracefully instead of asserting.
    return false;                       // ★ "NULL 永不匹配"②
  }

  const uint64_t join_key_hash =
      join_key_and_row_buffer->length() == 0
          ? kZeroKeyLengthHash
          : MY_XXH64(join_key_and_row_buffer->ptr(),
                     join_key_and_row_buffer->length(), xxhash_seed);
  //                 ★★ xxHash64 —— 只在这里用，哈希表内部用的是 ankerl hash

  assert((chunks->size() & (chunks->size() - 1)) == 0);
  // Since we know that the number of chunks will be a power of two, do a
  // bitwise AND instead of (join_key_hash % chunks->size()).
  const size_t chunk_index = join_key_hash & (chunks->size() - 1);
  //                 ★ 位与代替取模

  ChunkPair &chunk_pair = (*chunks)[chunk_index];
  if (write_to_build_chunk) {
    return chunk_pair.build_chunk.WriteRowToChunk(join_key_and_row_buffer,
                                                  row_has_match);
  } else {
    return chunk_pair.probe_chunk.WriteRowToChunk(join_key_and_row_buffer,
                                                  row_has_match);
  }
}
```

**chunk 数量估算**（`InitializeChunkFiles`，`hash_join_iterator.cc:370`）：

```cpp
  constexpr double kReductionFactor = 0.9;
  const double reduced_rows_in_hash_table =
      std::max<double>(1, rows_in_hash_table * kReductionFactor);

  // Avoid underflow, since the hash table may contain more rows than the
  // estimate from the planner.
  const size_t remaining_rows =
      std::max(rows_in_hash_table, estimated_rows_produced_by_join) -
      rows_in_hash_table;

  const size_t chunks_needed = std::max<size_t>(
      1, std::ceil(remaining_rows / reduced_rows_in_hash_table));
  const size_t num_chunks = std::min(max_chunk_files, chunks_needed);

  // Ensure that the number of chunks is always a power of two. This allows
  // us to do some optimizations when calculating which chunk a row should
  // be placed in.
  const size_t num_chunks_pow_2 = my_round_up_to_next_power(num_chunks);
  ...
  for (ChunkPair &chunk_pair : *chunk_pairs) {
    if (chunk_pair.build_chunk.Init(build_tables, /*uses_match_flags=*/false) ||
        chunk_pair.probe_chunk.Init(probe_tables,
                                    include_match_flag_for_probe)) {
      my_error(ER_TEMP_FILE_WRITE_FAILURE, MYF(0));
      return true;
    }
  }
```

**常量速查**（全部实测）：

```cpp
// sql/iterators/hash_join_iterator.h:545
  // Note that this is a random number, and has no special meaning. It is
  // deliberately different from the hash seed used in the hash table, so that
  // the chunk partitioning does not degenerate the hash table when a chunk
  // is loaded back into memory.
  static constexpr uint32_t kChunkPartitioningHashSeed{899339};
// :561
  static constexpr size_t kMaxChunks = 128;

// sql/iterators/hash_join_iterator.cc:58
static constexpr size_t kZeroKeyLengthHash = 2669509769;   // 避免对 nullptr 做算术（UB）

// include/my_io.h:159 / :164
constexpr const size_t IO_SIZE{4096};
constexpr const unsigned int DISK_BUFFER_SIZE{IO_SIZE * 16};   // = 65536

// sql/immutable_string.h:81 / :198
static constexpr int kMaxVarintBytes = 10;
```

> ⚠️ **`kChunkPartitioningHashSeed = 899339` 与哈希表哈希刻意不同**——注释明说"随机选的数，无特殊含义"，目的是避免重新装载 chunk 时 hash 表分布退化（`hash_join_iterator.h:545`）。

### 6.3 DS-MRR 完整握手（BKA 的核心，之前缺）

MRR 标志从哪来（`sql_optimizer.cc`）：

| 标志 | 位置 | 含义 |
|------|------|------|
| `HA_MRR_NO_NULL_ENDPOINTS` | `:3472-3474` | 端点不含 NULL |
| `HA_MRR_INDEX_ONLY` + `multi_range_read_info` | `:3663-3666` | 只走索引，不回表 |
| `USE_DEFAULT_IMPL` / `NO_ASSOCIATION` → **放弃 BKA** | `:3673-3676` | 引擎不支持则用不了 |

**握手过程**：

1. `MultiRangeRowIterator::Init`（`bka_iterator.cc:335-374`）注册 `RANGE_SEQ_IF {MrrInitCallbackThunk, MrrNextCallbackThunk, skip_record}`，调 `multi_range_read_init(n_ranges, m_mrr_flags, &m_mrr_buffer)`
2. `dsmrr_init`（`handler.cc:6550`）：若引擎返回 `USE_DEFAULT_IMPL | HA_MRR_SORTED` 则**退回默认实现**（`:6561-6564`）
3. **克隆第二个 handler `h2`** 做索引扫描（`:6600-6621`）并把 ICP 转移到它（`:6640-6649`）——为什么要第二个 handler？因为主 handler 当前正用于顺序读，MRR 需要独立的索引扫描上下文
4. `dsmrr_fill_buffer`（`:6767-6826`）：收集 `{rowid, range_info}`（`:6793-6800`，`elem_size = ref_length + is_mrr_assoc * sizeof(void*)`）后 **qsort 按 rowid 排序**，把随机 IO 转成顺序 IO

**BKA 的 MRR 缓冲区算法**（`bka_iterator.cc:198-218`）：`mrr_buffer_size = m_mrr_bytes_needed_per_row * rows`；超预算时**至少保 1 行**；有余量时 `min(2*mrr + 16384, max_mem - used)`。

## 参考

**论文**
- **Graefe《Query Evaluation Techniques for Large Databases》(1993)** —— hash join（grace / hybrid）、block nested loop 的经典分析

**官方文档**
- *MySQL 8.0 Reference Manual → Hash Join Optimization*
- *MySQL 8.0 Reference Manual → Block Nested-Loop and Batched Key Access Joins*
- *MySQL 8.0 Reference Manual → Multi-Range Read Optimization*（BKA 依赖 MRR）

**内核月报**
- **2019/11《MySQL 哈希连接实现介绍》** —— 8.0.18 哈希连接的 `std::unordered_multimap` + `xxHash64`、`HashJoinChunk` 落盘

**两侧分工**
- 优化器侧（能否用 buffer、用 BNL 还是 BKA、外连接/semijoin 约束）见 [`../07_optimize/10_plan_refinement.md`](../07_optimize/10_plan_refinement.md)
