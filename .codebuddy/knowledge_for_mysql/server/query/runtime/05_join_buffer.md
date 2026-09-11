# 05 join buffer：BNL / BKA / Hash Join 的缓冲机制（算法级）

> 本篇覆盖：join buffer 的三种实际载体、BNL 被 hash join 取代的真相、BKA 与 DS-MRR、hash join 的 build/probe/spill。

## 目录

- [先有个整体印象](#先有个整体印象)
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
| `HashJoinRowBuffer` | `hash_join_buffer.h:119` | hash join 的 build 端内存哈希表 |
| `BKAIterator` 的 `MEM_ROOT + m_rows` | `bka_iterator.h:202` | BKA 的外层行缓冲 |
| BKA 的 MRR rowid buffer | `bka_iterator.h:400` | 内层批量读的 rowid 排序缓冲区 |

---

## 一、关键版本事实：JOIN_CACHE 已删除

> ⚠️ `sql/sql_join_buffer.h` 只剩一个 33 行的空壳枚举：

```cpp
// sql/sql_join_buffer.h:27-31
class JOIN_CACHE {
 public:
  enum enum_join_cache_type { ALG_NONE = 0, ALG_BNL = 1, ALG_BKA = 2 };
};
```

`JOIN_CACHE_BNL`/`JOIN_CACHE_BKA` 类、`sql_join_buffer.cc`、`join_records()`/`join_matching_records()`/`get_match_flag()` **全部不存在**（5.x 时代实现，8.0 迭代器化 + hash join 引入后删除）。所以：

- 优化器仍产出 `ALG_BNL` 标记，但**执行器一律改写为 hash join**。
- 经典 BNL 的"分块循环"语义由 hash join 的 `IN_MEMORY_WITH_HASH_TABLE_REFILL` 模式承载。

---

## 二、BNL：优化器标记，执行器改写为 hash join

### 2.1 优化器侧：JT_ALL → ALG_BNL

`setup_join_buffering`（`sql_optimizer.cc:3616`）对全表扫描类访问路径打标记：

```cpp
switch (tab->type()) {
  case JT_ALL: case JT_INDEX_SCAN: case JT_RANGE: case JT_INDEX_MERGE:
    if (!bnl_on) goto no_join_cache;
    if (!join->select_count) tab->set_use_join_cache(JOIN_CACHE::ALG_BNL);
    return false;
```

### 2.2 执行器侧：OT_BNL 改写为 hash join

`QEP_TAB::OT_BNL` 的语义已写成 "Block-nested loop (rewritten to hash join)"（`sql_executor.h:423`）。

`ConnectJoins`（`sql_executor.cc:2726`）：

```cpp
const bool replace_with_hash_join =
    UseHashJoin(qep_tab) && !QueryMixesOuterBKAAndBNL(qep_tab->join());
```

`UseHashJoin` 就是 `op_type == OT_BNL`（`:2188`）。传统 EXPLAIN 里 `OT_BNL` 也直接显示成 "hash join"（`opt_explain.cc:1664`）。

### 2.3 退化两种情况

- **hash join 不允许 spill**（有 LIMIT 且无 group/order，`sql_executor.cc:2076` `allow_spill_to_disk=false`）→ 进入 `IN_MEMORY_WITH_HASH_TABLE_REFILL` 模式（即 BNL 骨架）。
- **外层 BKA + 内层 hash join 混用** → `QueryMixesOuterBKAAndBNL`（`:2210`）检测到后两者都关掉，整条查询退化成普通 nested loop（避免 matched-row 信号错乱）。

---

## 三、BKA：BKAIterator + MultiRangeRowIterator + DS-MRR

BKA 用两个迭代器协作（`bka_iterator.h:27` 注释）：`BKAIterator` 批量读外层行，`MultiRangeRowIterator` 用 MRR 批量读内层行。

### 3.1 BKAIterator：批量读外层 + 内存预算（`bka_iterator.cc:127`）

```cpp
int BKAIterator::ReadOuterRows() {
  for (;;) {
    int result = m_outer_input->Read();
    if (result == -1) { m_end_of_outer_rows = true; break; }
    StoreFromTableBuffers(m_outer_input_tables, &m_outer_row_buffer);  // 序列化外层行

    // 预算检查：本行 + 预计内层行 + match flag 不能超 join_buffer_size
    size_t total_bytes_needed = m_bytes_used + row_size +
        (m_mrr_bytes_needed_per_row + sizeof(m_rows[0])) * (m_rows.size() + 1);
    if (!m_rows.empty() && total_bytes_needed > m_max_memory_available) {
      m_has_row_from_previous_batch = true;   // 预算用尽，本行留下批
      break;
    }
    m_rows.push_back(BufferRow(...));
  }
}
```

每读一行做**内存预算**，`m_mrr_bytes_needed_per_row` 用"每个 key 预计返回的内层行数 × 单行大小"给内层 MRR 预留空间。

### 3.2 MultiRangeRowIterator：MRR 回调生成 key range（`bka_iterator.cc:381`）

```cpp
uint MultiRangeRowIterator::MrrNextCallback(KEY_MULTI_RANGE *range) {
  LoadBufferRowIntoTableBuffers(m_outer_input_tables, *m_current_pos);
  construct_lookup(thd(), table(), m_ref);        // 由外层行构造 ref 的 key
  range->range_flag = EQ_RANGE;
  range->ptr = ... m_current_pos ...;             // 关联回外层行
  range->start_key.key = m_ref->key_buff;
  range->start_key.flag = HA_READ_KEY_EXACT;      // [key, key] 单点
  range->end_key = range->start_key;
  range->end_key.flag = HA_READ_AFTER_KEY;
  return 0;
}
```

`Read()`（`:431`）从 MRR 拿内层行，用 `rec_ptr` 关联回外层行，`LoadIntoTableBuffers` 恢复外层行到 record buffer，完成 join。

### 3.3 DS-MRR：rowid 重排，磁盘顺序 sweep（`handler.cc:6766`）

`dsmrr_fill_buffer` 收集 rowid 后按 rowid **qsort 排序**：

```cpp
qsort(rowids_buf, ..., [](const void *a, const void *b) {
  return current_handler->cmp_ref(...);
});
```

`dsmrr_next` 按排序好的 rowid 用 `ha_rnd_pos` 回表。**BKA 的关键收益**：内层随机回表被重排成按 rowid 顺序的磁盘 sweep。

### 3.4 match flag 位图（替代旧 get_match_flag）

`NeedMatchFlags`（`bka_iterator.cc:60`）：OUTER/SEMI/ANTI 需要，每行 1 bit。`MarkLastRowAsRead` 标记，`RowHasBeenRead` 用于 NULL-complemented 和 semi/anti 去重。

---

## 四、hash join 的 build / probe / spill

### 4.1 三种模式（`hash_join_iterator.h:639`）

```cpp
enum class HashJoinType { IN_MEMORY, SPILL_TO_DISK, IN_MEMORY_WITH_HASH_TABLE_REFILL };
```

### 4.2 build：StoreRow 插哈希表（`hash_join_buffer.cc:212`）

```cpp
StoreRowResult HashJoinRowBuffer::StoreRow(THD *thd, bool reject_duplicate_keys) {
  // 1. 由 join 条件拼出 join key 字节串（多列等值拼成一个 key）
  for (const HashJoinCondition &hc : m_join_conditions)
    hc.join_condition()->append_join_key_for_hash_join(...);
  // NULL 等值 key 永不匹配，直接丢弃

  // 2. 插入哈希表（key → LinkedImmutableString 链表头）
  auto [it, inserted] = m_hash_map->emplace(key, LinkedImmutableString{nullptr});
  if (inserted) {
    // 新 key：按哈希表占用重新计算 MEM_ROOT 容量上限（即 join_buffer_size 预算）
    if (bytes_used >= m_max_mem_available) { m_mem_root.set_max_capacity(1); full = true; }
  } else {
    next_ptr = it->second;   // 冲突 → 头插法链
  }
  m_last_row_stored = it->second = StoreLinkedImmutableStringFromTableBuffers(next_ptr, &full);
  return full ? BUFFER_FULL : ROW_STORED;
}
```

哈希冲突用**链表头插法**（`LinkedImmutableString::next`），同 key 多行串成链。`find`（`:322`）probe 命中后沿 `next` 逐个取。

### 4.3 probe：LookupProbeRowInHashTable（`hash_join_iterator.cc:848`）

probe 行先算 key 再探测。等值 key 含 NULL 永不匹配（inner/semi 直接跳过；anti/outer 输出 NULL-complemented）。命中后 `ReadNextJoinedRowFromHashTable`（`:944`）沿链表取行并过 extra 条件（非等值条件在探测**之后**用 `val_int()` 求值）。

### 4.4 spill：ReadNextHashJoinChunk（`hash_join_iterator.cc:565`）

1. build 内存满时 `InitializeChunkFiles` 创建最多 `kMaxChunks` 对 chunk 文件（probe+build）。
2. 剩余 build 行按**另一个哈希函数**（`kChunkPartitioningHashSeed`，与哈希表哈希不同）分桶写 build chunk。
3. probe 阶段：命中内存哈希表的同时，probe 行也按同样函数写 probe chunk（内连接全写；半连接只写未命中行）。
4. probe 读完后 `LOADING_NEXT_CHUNK_PAIR`：把对应 build chunk 读进哈希表，probe chunk 回绕，做经典 hash join（同一桶的两边必在同一对 chunk 里）。

---

## 五、setup_join_buffering 决策链

`setup_join_buffering`（`sql_optimizer.cc:3451`）逐表判定，最终按访问类型分派：

| 内层访问类型 | 结论 |
|-------------|------|
| `JT_ALL/JT_INDEX_SCAN/JT_RANGE/JT_INDEX_MERGE` | `ALG_BNL`（执行时变 hash join） |
| `JT_REF/JT_EQ_REF` 等索引点查 | `ALG_BKA`（需 MRR 可用、非默认实现、有 association） |
| 其它 / 禁用条件 | `ALG_NONE`（普通 nested loop） |

禁用条件：第一张表、两开关都 off、`QS_DYNAMIC_RANGE`、`tableno > no_jbuf_after`、外层连接嵌套不一致、LOOSE_SCAN、MATERIALIZE 首个表、lateral derived 依赖、IN 子查询 cond guards 等。

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

**分桶**（`hash_join_iterator.cc:288-324`）：`MY_XXH64(key, len, kChunkPartitioningHashSeed)`，`chunk_index = hash & (size-1)`（size 恒为 2 的幂，位与代替取模）。**`kChunkPartitioningHashSeed = 899339` 与哈希表用的哈希种子刻意不同**——否则重新装载 chunk 时 hash 表会退化（`hash_join_iterator.h:545`）。

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
