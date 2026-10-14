# 01 filesort：排序算法与 sort buffer 的物理实现

> 基于 MySQL 8.0.39 源码。本篇回答：**filesort 是怎么排序的？什么时候落盘？排序记录里到底存了什么（addon 还是 rowid）？为什么 LIMIT 有两个层次的优化？sort_buffer_size 真的是一次性分配吗？**
>
> **边界**：本篇讲 filesort 排序算法本身。排序结果用的临时表（物化载体、TempTable/MEMORY、降级链）见 [`07_temptable.md`](07_temptable.md)。

## 目录

- [概述](#概述)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
- [深潜补充：字节布局与边界行为](#深潜补充字节布局与边界行为)
- [演进、性能与坑](#演进性能与坑)
- [相关的系统变量](#相关的系统变量)
- [参考](#参考)

---

## 概述

**是什么**：filesort 是 MySQL 的排序实现。名字里的 "file" 不代表一定用文件——**如果数据能装进 sort buffer，整个过程在内存完成，一次磁盘 I/O 都没有**。

**解决什么问题**：`ORDER BY` / `GROUP BY` / `DISTINCT` 无法利用索引顺序时，需要显式排序。

**在链路的哪个位置**：优化器决定"需要排序"后产出 `AccessPath::SORT`，执行期由 `SortingIterator::Init()` 调 `filesort()` 一次排完，`Read()` 逐条吐出。符合火山模型"重活在 `Init()`"的规律。

**四个阶段**（源码文件头注释就是最好的总纲）：

> Standard external sort. We read rows into a buffer until there's no more room. At that point, we use it (using the sorting algorithms from STL), and write it to disk (thus the name "filesort"). When there are no more rows, we merge chunks recursively, seven and seven (although we can go all the way up to 15 in the final pass if it helps us do one pass less).
> **If all the rows fit in a single chunk, the data never hits disk, but remains in RAM.**

```
read_all_rows()  读入所有行（放不下就排序→写 chunk→清空→继续）
     ↓
num_chunks == 0 ?  save_index()（纯内存，直接排序输出）
                 : merge_many_buff() + merge_index()（多路归并）
```

---

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 边装边溢，而不是先预判要不要落盘

这是理解 filesort 行为的关键，**也是很多人误判的地方**：

> **MySQL 8.0 没有"先估算一下要不要落盘"的预判逻辑。** 它是 **fill-until-full（边装边溢）**：往 buffer 里塞记录，直到再也分不出空间，此刻把当前 chunk 排序、写盘、清空、继续。

代码里所有"公式"（如 `keys = memory_available / max_record_length`）只服务于两件事：**PQ 是否适用**、以及 optimizer trace 的展示。真正的溢出判定在 `Filesort_buffer::allocate_block()` 返回"装不下"。

**为什么不预判**：优化器给的 `num_rows_estimate` 本来就不可靠（尤其带 WHERE 时），而实际记录长度受 collation、NULL、变长字段影响波动很大。与其赌估算，不如**用实测的内存边界说话**——这也是唯一能保证"小结果集绝不落盘"的做法。

**可预测的边界行为**：

- 单行就超过剩余预算且当前 chunk 为空 ⇒ 直接报 `ER_OUT_OF_SORTMEMORY`("Out of sort memory")，即使 buffer 是空的
- 单行超过 32KB 而 `sort_buffer_size` 是最小值 32KB ⇒ 必然失败
- 实际记录比 `max_record_length` 短很多时（packed addon / 变长），峰值内存远小于 `sort_buffer_size`

**★ 源码证据：`read_all_rows()` 的 Doxygen 伪码**（`sql/filesort.cc:885`，★ 官方把策略写成了伪码）：

```cpp
  @note
    Basic idea:
    @verbatim
     while (get_next_sortkey())
     {
       if (using priority queue)
         push sort key into queue
       else
       {
         try to put sort key into buffer;
         if (no free space in sort buffer)
         {
           do {
             allocate new, larger buffer;
             retry putting sort key into buffer;
           } until (record fits or no space for new buffer)
           if (no space for new buffer)
           {
             sort record pointers (all buffers);
             dump sorted sequence to 'tempfile';
             dump Merge_chunk describing sequence location into 'chunk_file';
           }
         }
         if (key was packed)
           tell sort buffer the actual number of bytes used;
       }
     }
     if (buffer has some elements && dumped at least once)
       sort-dump-dump as above;
     else
       don't sort, leave sort buffer to be sorted by caller.
  @endverbatim
```

**★ 两层"重试"的实现**（★ 注意：**没有"落盘失败重试"**）：

```cpp
/* ① filesort.cc:837 —— 内层：只多要 1 字节，让 Filesort_buffer 自己扩块 */
static bool alloc_and_make_sortkey(Sort_param *param, Filesort_info *fs_info,
                                   const Mem_root_array<TABLE *> &tables,
                                   size_t *key_length, size_t *longest_addons) {
  size_t min_bytes = 1;
  for (;;) {  // Termination condition within loop.
    Bounds_checked_array<uchar> sort_key_buf =
        fs_info->get_next_record_pointer(min_bytes);
    if (sort_key_buf.array() == nullptr) return true;   /* ★ 预算耗尽 → 交给上层落盘 */
    const uint rec_sz = param->make_sortkey(sort_key_buf, tables, longest_addons);
    if (current_thd->is_error()) return true;
    if (rec_sz > sort_key_buf.size()) {
      // The record wouldn't fit. Try again, asking for a larger buffer.
      min_bytes = sort_key_buf.size() + 1;              /* ★ 只多要 1 字节即可 */
    } else {
      fs_info->commit_used_memory(rec_sz);
      *key_length = rec_sz;
      return false;
    }
  }
}
```
```cpp
/* ② filesort.cc:1006 —— 外层：flush chunk 后"再试一次"，仍失败才放弃 */
        // Out of room, so flush chunk to disk (if there's anything to flush).
        if (num_records_this_chunk > 0) {
          if (write_keys(param, fs_info, num_records_this_chunk, chunk_file,
                         tempfile)) return HA_POS_ERROR;
          num_records_this_chunk = 0;
          num_written_chunks++;
          fs_info->reset();        /* ★ reset 只保留最大块并清空指针数组 */

          // Now we should have room for a new row.
          out_of_mem_or_error = alloc_and_make_sortkey(
              param, fs_info, tables, &key_length, &longest_addon_so_far);
        }
        // If we're still out of memory after flushing to disk, give up.
        if (out_of_mem_or_error) {
          my_error(ER_OUT_OF_SORTMEMORY, ME_FATALERROR);
          LogErr(ERROR_LEVEL, ER_SERVER_OUT_OF_SORTMEMORY);
          return HA_POS_ERROR;
        }
```
> ⚠️ **`write_keys()` 自身无重试**（`my_b_write` 失败直接 `return 1`）：磁盘写失败 = 直接向上冒泡。所谓"重试协议"只有上面两处。

#### 2. sort_buffer_size 不是一次性分配的（8.0 的关键演进）

老认知是"sort_buffer_size 一次性 malloc"。8.0 改成了**块式指数增长**：

```cpp
  if (m_current_block_size == 0) {
    next_block_size = MIN_SORT_MEMORY;          // 首块 32KB
  } else {
    next_block_size = m_current_block_size + m_current_block_size / 2;   // ×1.5
  }
```

**意义**：把 `sort_buffer_size` 调大（比如 8MB）**不会立刻吃掉 8MB**，只有真的要排序 8MB 数据才会增长到那么多。小排序只付 32KB。这是"预算"语义而非"预分配"语义。

同时 `space_left` 的计算会**扣掉未来记录指针数组的开销**（保守按最大行估算），源码注释坦白：*"This means that, for smaller records, we could go above the maximum permitted total memory usage."*

**另一个重要设计：排序的不是整条记录，而是指针数组。** `Filesort_buffer` 维护 `m_record_pointers`（每条记录一个 `uchar*`），排序时只搬指针、不搬数据——这让 `std::stable_sort` 的归并变成顺序访存，cache 友好。

**★ 官方 Doxygen：为什么"预算"而不是"预分配"**（`sql/filesort_utils.h:44`，★ 这段注释把代价算得很细）：

```cpp
/**
  Buffer used for storing records to be sorted. The records are stored in
  a series of buffers that are allocated incrementally, growing 50% each
  time, similar to how a MEM_ROOT works. This allows the user to set a large
  maximum buffer size without getting huge allocations for sorting small result
  sets. ...

  In addition, Filesort_buffer stores a vector of pointers to the beginning of
  each record. It is these pointers that are actually sorted in filesort.
  If the records are small, this can add up to overhead on top of the amount of
  memory the user expected to use. ...

  Assume that we have set maximum record size to infinity, but that in
  practice, they are are about the smallest size possible (4-byte sort key plus
  4-byte rowid) and that we are on a 64-bit system. Then, the worst possible
  overhead is that we use as much space on pointers as the size of the last
  (largest) block. We can look at the two possible extremes:

    - Smallest possible sort buffer (32 kB): 32 kB overhead.
    - A huge sort buffer (x kB): If the last block is y kB, the total size will
      be y + 2/3y + (2/3)²y + ... = 3y, which means the last block is 1/3 of the
      total size. Thus, pointer overhead will be no worse than 33%.

  ... The buffer must be kept available for multiple executions of the
  same sort operation, so one can call reset() for reuse. Again similar
  to MEM_ROOT, this keeps the last (largest) block and discards the others.
*/
```
```cpp
/* filesort_utils.cc:316 —— 首块 32KB、之后 ×1.5，且指针数组也计入预算 */
bool Filesort_buffer::allocate_block(size_t num_bytes) {
  size_t next_block_size;
  if (m_current_block_size == 0) {
    // First block.
    next_block_size = MIN_SORT_MEMORY;            /* ★ 32KB，与 sort_buffer_size 最小值同源 */
  } else {
    next_block_size = m_current_block_size + m_current_block_size / 2;   /* ×1.5 */
  }
  ...
  size_t space_used = m_current_block_size + m_space_used_other_blocks;
  space_used += m_record_pointers.capacity() * sizeof(m_record_pointers[0]);  /* ★ 指针也吃预算 */
  ...
  size_t excess_bytes =
      (m_record_pointers.capacity() - m_record_pointers.size()) *
      sizeof(m_record_pointers[0]);
  if (excess_bytes >= 32768) {          /* ★ 极端情况下 shrink_to_fit 回收指针数组再试 */
    size_t old_capacity = m_record_pointers.capacity();
    m_record_pointers.shrink_to_fit();
    if (m_record_pointers.capacity() < old_capacity) return allocate_block(num_bytes);
  }
```
> ★★ 关键数字：**指针数组开销上界 33%**，小 buffer 时是 100%（32KB 数据 + 32KB 指针）。这解释了为什么"记录越小，指针开销占比越离谱"。

#### 3. sort key 为什么要编码成"可直接 memcmp"

filesort 的核心技巧：把每一列编码成**定长、大端、可直接逐字节比较**的字节序列，这样比较就是一次 `memcmp`，完全不需要知道列类型。

**DESC 怎么实现？** 不是比较时反向，而是**编码时把数据本体按位取反**：

```cpp
    if (sort_field->reverse) {
      while (actual_length--) {
        *to = (uchar)(~*to);
        to++;
      }
    } else {
      to += actual_length;
    }
```

**为什么只反转数据本体，不反转 NULL 标志和长度前缀**：

- **NULL 标志是三值元信息**：`0x00`=NULL、`0x01`=非 NULL、`0xff`=NULL 且 DESC。它必须保持可识别取值，供比较器识别 NULL 语义。注意 DESC 时是**显式赋 0xff 而非取反**——因为比较器判定"两边都是 NULL"用的是 `if (k1_nullbyte == 0 || k1_nullbyte == 0xff)`，取反得到 0xfe 会让判定失效
- **长度前缀是二进制整数**：`uint4korr()` 要读出真实长度，取反就废了；长度的方向性由比较器**显式处理**（`if (kp1_len != kp2_len) return reverse ? kp2_len < kp1_len : kp1_len < kp2_len;`）
- **数据本体取反 + memcmp 升序 = 数据降序**（对无符号字节，按位取反把全序反转）

⇒ **推论**：`ORDER BY a DESC, b ASC` 这种混合方向，MySQL 依然能用纯 `memcmp`——每列各自按方向编码。这是这套编码方案最大的价值。

**★ `make_sortkey()` 完整实现**（`sql/filesort.cc:1392`，⚠️ **不在** `filesort_utils.cc`）：

```cpp
uint Sort_param::make_sortkey(Bounds_checked_array<uchar> dst,
                              const Mem_root_array<TABLE *> &tables,
                              size_t *longest_addon_so_far) {
  uchar *to = dst.array();
  uchar *to_end = dst.array() + dst.size();
  uchar *orig_to = to;
  const st_sort_field *sort_field;
  ulonglong hash = 0;

  if (using_varlen_keys()) {
    to += size_of_varlength_field;      /* ★ 先跳过 4 字节总长前缀，末尾回填 */
    if (to >= to_end) return UINT_MAX;
  }
  for (sort_field = local_sortorder.begin(); sort_field != local_sortorder.end();
       sort_field++) {
    if (to >= to_end ||
        (!sort_field->is_varlen &&
         static_cast<size_t>(to_end - to) < sort_field->length)) {
      return UINT_MAX;                  /* ★ 装不下返回 UINT_MAX，由调用方扩容重试 */
    }
    bool maybe_null;
    std::optional<size_t> dst_length;
    if (!sort_field->is_varlen) dst_length = sort_field->length;
    uint actual_length;
    Item *item = sort_field->item;
    actual_length = make_sortkey_from_item(item, sort_field->result_type,
                                           dst_length, &tmp_buffer, to, to_end,
                                           &maybe_null, &hash);
    if (actual_length == UINT_MAX) return UINT_MAX;

    bool is_null = maybe_null && *to == 0;
    if (maybe_null) {
      assert(*to == 0 || *to == 1);
      if (sort_field->reverse && is_null) {
        *to = 0xff;                     /* ★ NULL+DESC：显式赋 0xff 而非取反 */
      }
      ++to;                             /* ★ NULL 标记独立 1 字节，不参与取反 */
    }

    // Fill out the varlen prefix if it exists.
    if (sort_field->is_varlen && !is_null) {
      int4store(to, actual_length + VARLEN_PREFIX);
      to += VARLEN_PREFIX;
    }

    // Reverse the key if needed.
    if (sort_field->reverse) {
      while (actual_length--) {
        *to = (uchar)(~*to);            /* ★ DESC 的唯一实现：按位取反 */
        to++;
      }
    } else {
      to += actual_length;
    }
  }

  if (use_hash) {
    if (to_end - to < 8) return UINT_MAX;
    int8store(to, hash);                /* ★ JSON 分组用：8 字节 hash 拼在 key 尾 */
    to += 8;
  }
  if (using_varlen_keys()) {
    Sort_param::store_varlen_key_length(orig_to, static_cast<uint>(to - orig_to));
  }
  ...
```
> ⚠️ 真名核查：**没有 `Sort_field` 类**，真名是 `st_sort_field`（POD struct，`sort_param.h:86`）。`make_sortkey` 声明在 `sort_param.h:401`，实现在 `filesort.cc:1392`。

**★ 各类型怎么编码成"可 memcmp"**（`make_sortkey_from_item()`，`filesort.cc:1224`）：

```cpp
    case INT_RESULT: {
      ...
      } else
        copy_native_longlong(to, dst_length.value(), value,
                             item->unsigned_flag);   /* ★ 整数：大端可 memcmp 的定长编码 */
      return dst_length.value();
    }
    case REAL_RESULT: {
      ...
      } else if (dst_length.value() < sizeof(double)) {
        uchar buf[sizeof(double)];
        change_double_for_sort(value, buf);   /* ★ double→可 memcmp：正数置高位 1，负数按位翻转 */
        memcpy(to, buf, dst_length.value());
      } else {
        change_double_for_sort(value, to);
      }
```
```cpp
    case STRING_RESULT: {
      if (item->data_type() == MYSQL_TYPE_JSON) {
        return make_json_sort_key(item, to, null_indicator, to_end - to, hash);
      }
      String *res = item->val_str(tmp_buffer);   /* ★ tmp_buffer 复用，省一次 alloc */
      ...
      } else {
        actual_length = cs->coll->strnxfrm(
            cs, to, dst_length.value(), item->max_char_length(),
            pointer_cast<const uchar *>(from), src_length,
            MY_STRXFRM_PAD_TO_MAXLEN);            /* ★ 定长路径：PAD 到等长保证 memcmp 可比 */
        assert(actual_length == dst_length.value());
      }
```
> ★ 关键：`tmp_buffer`（`Sort_param::tmp_buffer`）只是**复用的 `String` 暂存 `val_str()` 结果**，不是"ASCII 直接 memcpy 优化"——**filesort 路径所有字符串一律过 `strnxfrm`，没有例外**。

**★ `sortlength()`：长度从哪来**（`filesort.cc:2083`）：

```cpp
uint sortlength(THD *thd, st_sort_field *sortorder, uint s_length) {
  uint total_length = 0;
  // Heed the contract that strnxfrm() needs an even number of bytes.
  const uint max_sort_length_even = (thd->variables.max_sort_length + 1) & ~1;
                                        /* ★ 强制偶数：strnxfrm 的硬约束 */
  for (; s_length--; sortorder++) {
    ...
    if (item->type() == Item::FIELD_ITEM &&
        (down_cast<const Item_field *>(item)->field->real_type() == MYSQL_TYPE_ENUM ||
         ... == MYSQL_TYPE_SET)) {
      // Sort enum and set fields as their underlying ints.
      sortorder->result_type = INT_RESULT;   /* ★ ENUM/SET 按底层 int 排，不当字符串 */
    }
    if (sortorder->field_type == MYSQL_TYPE_JSON) sortorder->is_varlen = true;
    if (item->is_temporal()) sortorder->result_type = INT_RESULT;  /* ★ 日期时间全当整数 */
    switch (sortorder->result_type) {
      case STRING_RESULT: {
        const CHARSET_INFO *cs = item->collation.collation;
        sortorder->length = item->max_length;
        if (cs->pad_attribute == NO_PAD) sortorder->is_varlen = true;
                                        /* ★ NO_PAD（如 utf8mb4_0900_ai_ci）→ 变长 key */
        if (sortorder->length < (10 << 20)) {
          // How many bytes do we need (including sort weights) for strnxfrm()?
          sortorder->length = cs->coll->strnxfrmlen(cs, sortorder->length);
                                        /* ★★ 不是原始字节数，是 strnxfrm 后的权重字节数 */
        } else {
          sortorder->length = 0xFFFFFFFFu;   /* ★ "无限"，见 sort_param.h:88 的警告 */
        }
        is_string_type = true;
        break;
      }
      case INT_RESULT:
        sortorder->length = 8;  break;   // Size of intern longlong
      case DECIMAL_RESULT:
        sortorder->length = my_decimal_get_binary_size(
            item->max_length - (item->decimals ? 1 : 0), item->decimals);
        break;
      case REAL_RESULT:
        sortorder->length = sizeof(double);  break;
    }
    sortorder->maybe_null = item->is_nullable();
    if (!sortorder->is_varlen && is_string_type) {
      /*
        We would love to never have to care about max_sort_length anymore,
        but that would make it impossible for us to sort blobs (TEXT) with
        PAD SPACE collations, since those are not variable-length (the padding
        is serialized as part of the sort key) and thus require infinite space.
      */
      sortorder->length = std::min(sortorder->length, max_sort_length_even);
                                        /* ★★ max_sort_length 只在这里生效 */
    }
    if (sortorder->maybe_null)
      AddWithSaturate(1u, &total_length);  // Place for NULL marker
    if (sortorder->is_varlen)
      AddWithSaturate(VARLEN_PREFIX, &sortorder->length);   /* ★ 变长再预付 4 字节前缀 */
    AddWithSaturate(sortorder->length, &total_length);
  }
  return total_length;
}
```
> ⚠️ **不存在 `Field::sort_length()`**。sort key 长度一律由 `sortlength()` 在 **Item 层**用 `result_type()` / `max_length` / `strnxfrmlen()` 算，与 `Field` 类无关。`Field::make_sort_key()` 是纯虚函数（`field.h:1335`），但 **filesort 路径不经过它**——调用者只有 `rpl_write_set_handler.cc` 与 `item_strfunc.cc`。`sort_param.h:263` 的相关注释已过时。

#### 4. addon vs rowid：排序完怎么拿到完整行

排完序要返回完整行，这一行的数据从哪来？两种载体：

| | **addon fields** | **rowid** |
|---|---|---|
| 排序记录 | sort key + **列数据拷贝** | sort key + **行位置**（`handler::ref`） |
| 读结果 | `SortBufferIterator` 解包，**不回表** | `SortBufferIndirectIterator` 用 `ha_rnd_pos` **回表取整行** |
| 取舍 | 空间换时间：记录宽，省掉回表随机 IO | 时间换空间：记录窄，读结果要随机回表 |

**8.0.20 的转折**：过去由 `max_length_for_sort_data` 控制"行太宽就改排 rowid"，该参数已废弃（`system_variables.h` 标注 `Unused`）。`decide_addon_fields` 改为**默认 always addon**（注释原文："Generally, prefer using addon fields if we can"）。

**为什么默认 addon**：回表的随机 IO 比多占点内存贵得多，尤其在 SSD 普及前更是如此；而且 addon 还能进一步 pack 压缩。

于是 rowid 模式只剩三个**强制**场景（`Addon_fields_status` 枚举记录了原因）：

- `force_sort_rowids`：**UPDATE/DELETE 的两阶段读**——排序后要 `ha_rnd_pos` 回表**改行**，必须存 rowid 定位原始行
- `fulltext_searched`：**FTS**（MATCH 直接向 handler 要当前行，需 rowid 定位）
- `row_contains_blob`：**含大 BLOB**（BLOB 太大不值得 pack 进 addon）

**一个精妙细节**：rowid 模式下，`init_for_filesort()` 会把 rowid 长度**加进 sort key 长度**：

```cpp
  } else {
    fixed_res_length = sum_ref_length;
    /*
      The reference to the record is considered
      as an additional sorted field
    */
    AddWithSaturate(sum_ref_length, &m_fixed_sort_length);
  }
```

即"rowid 也算一个排序列"——这让相同 key 的行**天然按 rowid 有序**，从而得到确定性的输出顺序。代价是去重时必须把这段减掉（`sort_buffer()` 里有 `key_len -= param->sum_ref_length`），否则两行内容相同但 rowid 不同就不算重复了。

**★★ `decide_addon_fields()`：默认 always addon 的官方论述**（`filesort.cc:162`）：

```cpp
void Sort_param::decide_addon_fields(Filesort *file_sort,
                                     const Mem_root_array<TABLE *> &tables,
                                     bool force_sort_rowids) {
  if (m_addon_fields_status != Addon_fields_status::unknown_status) return;

  DBUG_EXECUTE_IF("filesort_force_sort_row_ids", {
    m_addon_fields_status = Addon_fields_status::keep_rowid;
    return;
  });

  // Generally, prefer using addon fields (ie., sorting rows instead of just
  // row IDs) if we can.
  //
  // NOTE: If the table is already in memory (e.g. the MEMORY engine; see the
  // HA_FAST_KEY_READ flag), it would normally be beneficial to sort row IDs
  // over rows to get smaller sort chunks. However, eliding the temporary table
  // entirely is even better than using row IDs, and only possible if we sort
  // rows. Furthermore, since we set up filesort at optimize time, we don't know
  // yet whether the source table would _remain_ in memory, or could be spilled
  // to disk using InnoDB. Thus, the only case that reasonably remains where
  // we'd want to use row IDs without being forced would be on user (ie.,
  // non-temporary) MEMORY tables, which doesn't seem reasonable to add
  // complexity for.
  /* 【注】★ 关键理由：只有排整行才能"省掉中间临时表"（StreamingIterator），
             这个收益大于 rowid 的省内存。 */

  for (TABLE *table : tables) {
    if (table->pos_in_table_list &&
        table->pos_in_table_list->is_fulltext_searched()) {
      // See comment in SortWillBeOnRowId().
      m_addon_fields_status = Addon_fields_status::fulltext_searched;
      return;
    }
  }

  if (force_sort_rowids) {
    m_addon_fields_status = Addon_fields_status::keep_rowid;
  } else {
    addon_fields = file_sort->get_addon_fields(
        &m_addon_fields_status, &m_addon_length, &m_packable_length);
  }
}
```

**★ `Addon_fields_status`：不 pack / 不用 addon 的 7 种理由**（`sort_param.h:49`）：

```cpp
enum class Addon_fields_status {
  unknown_status,
  using_addon_fields,
  // The remainder are reasons why we are _not_ using addon fields.
  fulltext_searched,
  keep_rowid,
  row_not_packable,
  row_contains_blob,
  skip_heuristic,
  using_priority_queue
};
```
> ★ 想看"为什么没 pack"，只能开 optimizer trace 读 `filesort_summary.unpacked_addon_fields`（`filesort.cc:600`）。

**★★ rowid 的第三个强制条件其实是 blob 的硬阈值 70000**（`SortWillBeOnRowId()`，`filesort.cc:2187`，★ 常被说成"含大 BLOB"但真值在这里）：

```cpp
bool SortWillBeOnRowId(const TABLE *table) {
  if (table->pos_in_table_list && table->pos_in_table_list->is_fulltext_searched()) {
    // MATCH() (except in “boolean mode”) doesn't use the actual value,
    // it just goes and asks the handler directly for the current row.
    // Thus, we need row IDs, so that the row is positioned correctly.
    return true;
  }
  for (Field **pfield = table->field; *pfield != nullptr; ++pfield) {
    Field *field = *pfield;
    if (!bitmap_is_set(&table->read_set_internal, field->field_index())) continue;
    /*
      Having large blobs in addon fields could be very inefficient,
      but small blobs are OK ... There are two types of small blobs:
       - Those explicitly bounded to small lengths, namely tinyblob
         (255 bytes) and blob (65535 bytes).
       - Those that are _typically_ fairly small, which includes JSON and
         geometries. ...
      That leaves only mediumblob and longblob. ...
    */
    if (field->type() == MYSQL_TYPE_BLOB &&
        field->max_packed_col_length() > 70000u) {   /* ★★ 阈值常量 70000 */
      return true;
    }
  }
  return false;
}
```
> ★★ **tinyblob(255) / blob(65535) / JSON / geometry 都允许进 addon**；只有 **mediumblob / longblob**（`max_packed_col_length() > 70000`）才强制 rowid。这比"含大 BLOB"精确得多。
> ★ 顺带：`Field_blob::max_packed_col_length()` 对 packlength=4 返回 `UINT_MAX`（`field.cc:7356`）。

**★ rowid 载体：就是 `handler::ref` 的裸字节**（`make_sortkey` 尾部 + `handler.h:4431`）：

```cpp
  } else {
    if (static_cast<size_t>(to_end - to) < sum_ref_length) return UINT_MAX;
    /* Save NULL flags and filepos last */
    for (TABLE *table : tables) {
      if (table->is_nullable()) {
        *to++ = table->has_null_row();      /* ★ 每表 1 字节 NULL 行标记 */
      }
      if (table->is_nullable() && table->has_null_row()) {
        // The contents are not used, but it's nice to have them
        // defined when writing them to disk nevertheless.
        memset(to, 0, table->file->ref_length);
      } else {
        memcpy(to, table->file->ref, table->file->ref_length);  /* ★★ 裸拷 ref */
      }
      to += table->file->ref_length;
    }
  }
```
```cpp
/* handler.h:4431 / :4506 */
 public:
  handlerton *ht; /* storage engine of this handler */
  /** Pointer to current row */
  uchar *ref;
  ...
  /** Length of ref (1-8 or the clustered key length) */
  uint ref_length;
```
> ★ `ref` 由 `handler::position()` 填、`handler::ha_rnd_pos()` 消费；分配见 `handler.cc:2824`（`mem_root->Alloc(ALIGN_SIZE(ref_length) * 2)`）。

**★★ 回表读取：`SortFileIndirectIterator::Read()`**（`sorting_iterator.cc:156`，★ 随机 IO 的现场）：

```cpp
int SortFileIndirectIterator::Read() {
  for (;;) {
    if (my_b_read(m_io_cache, m_ref_pos, m_sum_ref_length))
      return -1; /* End of file */
    uchar *ref_pos = m_ref_pos;
    bool skip = false;
    for (TABLE *table : m_tables) {
      if (m_has_null_flags && table->is_nullable()) {
        if (*ref_pos++) {
          table->set_null_row();
          ref_pos += table->file->ref_length;
          continue;
        } else {
          table->reset_null_row();
        }
      }
      int tmp = table->file->ha_rnd_pos(table->record[0], ref_pos);  /* ★★ 每行一次随机定位 */
      ref_pos += table->file->ref_length;
      /* The following is extremely unlikely to happen */
      if (tmp == HA_ERR_RECORD_DELETED ||
          (tmp == HA_ERR_KEY_NOT_FOUND && m_ignore_not_found_rows)) {
        skip = true;
        break;
      } else if (tmp != 0) {
        return HandleError(thd(), table, tmp);
      }
    }
    if (skip) continue;
    ...
```
> ★ `Init()` 里必须先 `ha_rnd_init(false)` 切到随机读模式——注释解释："The sort's source iterator could have initialized an index read, and it won't call end until it's destroyed ... as a small hack, we need to reset it here."

#### 5. packed addon：为什么"可省 <14 字节就不 pack"

addon 模式下，非 packed 时每个字段按 `max_length` 占满（`VARCHAR(300)` 存 "abcd" 也占 300 字节）。packed 后按实际长度存，NULL 字段零字节，代价是每条记录多一个 4 字节长度域。

```cpp
  // Heuristic: skip packing if potential savings are less than 10 bytes.
  const uint sz = Addon_fields::size_of_length_field;
  if (m_packable_length < (10 + sz)) {
    m_addon_fields_status = Addon_fields_status::skip_heuristic;
    return;
  }
```

**判据推导**：pack 的固定成本是 4 字节长度域；净收益阈值取 10 字节；所以只有"可节省空间上界 ≥ 10 + 4 = 14 字节"才值得。`m_packable_length` 只累加 **nullable / VARCHAR / VARSTRING / STRING / BLOB** 类字段（定长非空字段本来就没有节省空间）。

**为什么 PQ 模式禁用 pack**：PQ 的记录是**预分配的定长槽**，队列满时原地覆盖 top 指向的槽。记录一旦变长，一次覆盖就可能越界。所以走 PQ 时 `m_addon_fields_status = using_priority_queue`，明确记录"没 pack 是因为 PQ"。

**★★ 写入侧：packed vs 非 packed 的差别就在这几行**（`filesort.cc:1504`）：

```cpp
    // Actually store the addons.
    if (addon_fields->using_packed_addons()) {
      for (const Sort_addon_field &addonf : *addon_fields) {
        Field *field = addonf.field;
        if (field->table->has_null_row()) {
          assert(field->table->is_nullable());
          continue;                      /* ★ 外连接 NULL 行：整行一个字节都不写 */
        }
        if (addonf.null_bit && field->is_null()) {
          nulls[addonf.null_offset] |= addonf.null_bit;   /* ★ NULL 只置位，不写数据 */
        } else {
          to = field->pack(to, field->field_ptr(), to_end - to);  /* ★ 变长写入，pack 返回新游标 */
          if (to >= to_end) return UINT_MAX;
        }
      }
      Addon_fields::store_addon_length(p_len, to - p_len);  /* ★ 回填 4 字节长度 */
    } else {
      for (const Sort_addon_field &addonf : *addon_fields) {
        Field *field = addonf.field;
        if (static_cast<size_t>(to_end - to) < addonf.max_length) return UINT_MAX;
        if (addonf.null_bit && field->is_null()) {
          nulls[addonf.null_offset] |= addonf.null_bit;
        } else {
          uchar *ptr [[maybe_unused]] = field->pack(to, field->field_ptr(), to_end - to);
          assert(ptr <= to + addonf.max_length);
        }
        to += addonf.max_length;        /* ★★ 定长占位：NULL / 短值也白占 max_length */
      }
    }
```

**★ `m_packable_length` 只统计这几类**（`filesort.cc:2298`，★ 所以"全是定长列"的表天然走不上 pack）：

```cpp
      const enum_field_types field_type = field->type();
      if (field->is_nullable() || field_type == MYSQL_TYPE_STRING ||
          field_type == MYSQL_TYPE_VARCHAR ||
          field_type == MYSQL_TYPE_VAR_STRING ||
          field->is_flag_set(BLOB_FLAG)) {
        AddWithSaturate(field_length, &packable_length);
      }
```

**★ `Addon_fields` 类**（`sort_param.h:121`，⚠️ **不存在 `Packed_addon_fields` 类**）：

```cpp
class Addon_fields {
 public:
  /// SortFileIterator needs an extra buffer when unpacking.
  uchar *allocate_addon_buf(uint sz) { ... }
  bool using_packed_addons() const { return m_using_packed_addons; }
  /**
    How many bytes to skip to get to the actual data; first NULL flags
    (for tables and addon fields) and then the actual addons.
   */
  size_t skip_bytes() const {
    if (m_using_packed_addons) return Addon_fields::size_of_length_field;
    else return 0;
  }
  /**
    @returns Total number of bytes used for packed addon fields.
   */
  static uint read_addon_length(uchar *p) {
    return size_of_length_field + uint4korr(p);   /* ★★ uint4korr → 证实长度字段 = 4 字节 */
  }
  static void store_addon_length(uchar *p, uint sz) {
    // We actually store the length of everything *after* the length field.
    int4store(p, sz - size_of_length_field);
  }
  static const uint size_of_length_field = 4;
 private:
  Addon_fields_array m_field_descriptors;
  uchar *m_addon_buf;
  uint m_addon_buf_length;
  bool m_using_packed_addons;
  int m_first_addon_relative_offset = 0;
};
```
> ⚠️⚠️ **`sort_param.h:266-269` 的 Doxygen 注释说长度字段是 "2 bytes"，这是过时注释**——代码 `size_of_length_field = 4` 且 `read_addon_length` 用 `uint4korr`。
> ★ `Packed_addon_fields` 只是 **`template <bool Packed_addon_fields>` 的模板参数名**，不是类。

**★ `Field::pack()` 与 `Field_varstring::pack()`**（`field.cc:1872` / `:6766`）：

```cpp
uchar *Field::pack(uchar *to, const uchar *from, size_t max_length) const {
  size_t length = std::min<size_t>(pack_length(), max_length);
  memcpy(to, from, length);
  return to + length;      /* ★ 默认实现：定长裸拷，返回新游标 */
}
```
```cpp
uchar *Field_varstring::pack(uchar *to, const uchar *from,
                             size_t max_length) const {
  uint length = length_bytes == 1 ? (uint)*from : uint2korr(from);
  if (max_length < length_bytes)
    length = 0;
  else if (length > max_length - length_bytes)
    length = max_length - length_bytes;   /* ★ 截断到 max_length */
  /* Length always stored little-endian */
  if (max_length >= 1) {
    *to++ = length & 0xFF;
    if (length_bytes == 2 && max_length >= 2) *to++ = (length >> 8) & 0xFF;
  }
  memcpy(to, from + length_bytes, length);
  return to + length;                     /* ★ packed 靠返回值推进，写多少用多少 */
}
```
> ★ pack 的"省空间"与"丢数据"是同一件事——`max_length` 不够就**静默截断**。

**★ 解包侧：非 packed 的浪费在这里显形**（`sorting_iterator.cc:540`）：

```cpp
template <bool Packed_addon_fields>
inline void Filesort_info::unpack_addon_fields(
    const Mem_root_array<TABLE *> &tables, uchar *buff) {
  const uchar *nulls = buff + addon_fields->skip_bytes();
  ...
  const uchar *start_of_record = buff + addon_fields->first_addon_offset();
  for (const Sort_addon_field &addonf : *addon_fields) {
    Field *field = addonf.field;
    const bool is_null = addonf.null_bit && (addonf.null_bit & nulls[addonf.null_offset]);
    if (is_null) field->set_null();
    if (Packed_addon_fields) {
      if (!is_null && !field->table->has_null_row()) {
        field->set_notnull();
        start_of_record = field->unpack(start_of_record);   /* ★ 变长：游标由 unpack 推进 */
      }
    } else {
      if (!is_null) {
        field->set_notnull();
        field->unpack(start_of_record);
      }
      start_of_record += addonf.max_length;   /* ★★ 定长滑过，NULL 也占满 */
    }
  }
}
```

#### 6. LIMIT 优化的两个层次（容易混淆）

MySQL 对 `ORDER BY ... LIMIT k` 有**两套不同机制**，常被混为一谈：

| 机制 | 适用 | 做法 | 复杂度 |
|---|---|---|---|
| **`nth_element` 预筛** | 普通排序路径 | 先 `O(n)` 选到第 k 位，再只排前 k 个 | `O(n + k log k)` |
| **优先队列（PQ）** | 整趟排序都省掉 | 维护大小 k+1 的堆，边读边淘汰最大的 | `O(n log k)`，且**永不落盘** |

`nth_element` 的触发条件：`max_output_rows < num_input_rows / 2 && !m_remove_duplicates`（去重需要全局相邻，预筛会破坏它）。

PQ 的触发条件见后面"LIMIT 优化"一节——注意它有个反直觉的判据：**PQ 比全排慢约 3 倍**（`PQ_slowness = 3.0`），所以数据能全装进内存时，只有 `LIMIT < 行数/3` 才值得用 PQ。

### 理论溯源

- **外部排序（external merge sort）**：Graefe《Query Evaluation Techniques for Large Databases》(1993) 的经典综述。filesort 就是这个算法的标准实现：run generation（chunk 生成）+ 多路归并
- **top-k 堆选择**：PQ 用的是经典的大顶堆维护"目前最小的 k 个"
- **可 memcmp 的键编码**：源自"order-preserving encoding"思想，把类型相关的比较转成字节比较

### 算法与数据结构

| 结构 | 作用 |
|---|---|
| `Filesort` | 优化期构造的描述符（limit、sortorder、是否强制 rowid、keep_buffers） |
| `Sort_param` | 运行期长度参数：`m_fixed_sort_length`（key 最大长）、`m_addon_length`、`m_fixed_rec_length`（整条记录最大长）、`sum_ref_length`、`max_rows` |
| `Filesort_buffer` | 分块内存 + `m_record_pointers` 指针数组 |
| `Merge_chunk` | 一个 chunk 的描述符（文件位置、行数、内存区间） |
| `Bounded_queue` | LIMIT 的优先队列（k+1 槽） |

关键的三个长度关系：

```
m_fixed_sort_length = sortlength() 结果 + (rowid? sum_ref_length) + (varlen? 4) + (json? 8)
m_addon_length      = sum(max_packed_col_length) + ceil(null_fields/8) + (packed? 4)
m_fixed_rec_length  = m_fixed_sort_length + m_addon_length     ← 所有内存预算的基础
```

### 他库对比与演进动机

| | MySQL 8.0 | PostgreSQL | SQL Server |
|---|---|---|---|
| 溢出策略 | 边装边溢 + 7 路归并 | 类似（tuplesort） | 内存授予 + spill |
| 排序内存 | `sort_buffer_size`（预算式，块增长） | `work_mem`（每算子） | memory grant（预估算） |
| top-N | PQ + nth_element | top-N heapsort | Top N Sort |

**演进**：5.x 的 `filesort_compare`（手写比较器，带 sort_length/res_length 分离）→ 8.0 改用 C++ 标准库 + 三种比较器；5.x 的一次性分配 → 8.0 块式指数增长；`max_length_for_sort_data` 废弃 → addon 默认化。

---

## 核心实现

### 主链路

```
SortingIterator::Init()
  └─ DoSort()
       └─ filesort()
            ├─ init_for_filesort()          所有长度参数定型
            ├─ check_if_pq_applicable()?    → PQ 分支（预分配 LIMIT+1 槽）
            │                               → 否则 try_to_pack_addons()
            ├─ read_all_rows()             逐行 Read → make_sortkey → 塞 buffer
            │     └─ 装不下 → write_keys()（排序 + 写 chunk）→ reset → 重试
            └─ num_chunks == 0 ? save_index()          （纯内存）
                              : merge_many_buff() → merge_index()（多路归并）
```

### `filesort()` 主流程

关键片段与逐段解释：

**① rowid 模式要先准备定位**

```cpp
  if (!param->using_addon_fields()) {
    for (TABLE *table : filesort->tables) {
      if (table->pos_in_table_list == nullptr ||
          (tables_to_get_rowid_for & table->pos_in_table_list->map())) {
        table->prepare_for_position();
      }
    }
  }
```

只有 rowid 模式需要（为后面的 `ha_rnd_pos()` 做准备）。

**② 必须先 `source_iterator->Init()`**

注释说明：`table->file`（尤其 `ref_length`）在扫描初始化前可能没就绪，而 `ref_length` 决定 rowid 模式下一行多长。顺序反了会用错误长度。

**③ PQ 分支 vs 普通分支**

```cpp
  if (check_if_pq_applicable(trace, param, fs_info, num_rows_estimate,
                             memory_available)) {
    if (fs_info->preallocate_records(param->max_rows_per_buffer)) { ... }
    if (pq.init(param->max_rows, param, fs_info->get_sort_keys())) { ... }
    filesort->using_pq = true;
    param->using_pq = true;
    param->m_addon_fields_status = Addon_fields_status::using_priority_queue;
  } else {
    filesort->using_pq = false;
    param->using_pq = false;
    /*
      When sorting using priority queue, we cannot use packed addons.
      Without PQ, we can try.
    */
    param->try_to_pack_addons();
    ...
    ha_rows keys = memory_available / (param->max_record_length() + sizeof(char *));
    param->max_rows_per_buffer = min(num_rows_estimate > 0 ? num_rows_estimate : 1, keys);
    fs_info->set_max_size(memory_available, param->max_record_length());
  }
```

注意 `max_rows_per_buffer` 在非 PQ 分支**只是给 trace 看的**（源码自己注明），真正的内存上限由 `set_max_size()` 设定。

**④ 是否落盘的判定——只看 chunk 描述符文件有没有被写过**

```cpp
  if (my_b_inited(&chunk_file))
    num_chunks = my_b_tell(&chunk_file) / sizeof(Merge_chunk);
  else
    num_chunks = 0;
```

因为 `chunk_file` 只在第一次溢出时才 `open_cached_file()`（惰性建文件），所以 `num_chunks == 0` ⇔ 一次都没溢出 ⇔ 纯内存排序。

**⑤ 结果留在哪**

- addon 模式：`save_index()` 把结果**留在 filesort buffer 里**（`sorted_result_in_fsbuf = true`），零拷贝
- rowid 模式：把 payload（rowid + NULL 标志）抠出来压成紧凑数组，读的时候逐条 `ha_rnd_pos()` 回表

**⑥ optimizer trace 的输出**（诊断 filesort 的主要手段）：

```
"filesort_summary": {
  "memory_available": 262144, "key_size": 17, "row_size": 231,
  "num_initial_chunks_spilled_to_disk": 0,
  "sort_algorithm": "std::stable_sort",
  "unpacked_addon_fields": "using_priority_queue",
  "sort_mode": "<fixed_sort_key, additional_fields>"
}
```

`sort_mode` 的三段式组合正好反映本节的两个维度：`{fixed|varlen}_sort_key` × `{packed_additional_fields|additional_fields|rowid}`。

### 内存管理：`Filesort_buffer` 的块式增长

```cpp
bool Filesort_buffer::allocate_block(size_t num_bytes) {
  size_t next_block_size;
  if (m_current_block_size == 0) {
    next_block_size = MIN_SORT_MEMORY;              // 首块 32KB
  } else {
    next_block_size = m_current_block_size + m_current_block_size / 2;   // ×1.5
  }
  ...
  size_t space_used = m_current_block_size + m_space_used_other_blocks;
  space_used += m_record_pointers.capacity() * sizeof(m_record_pointers[0]);
  size_t space_left = (space_used > m_max_size_in_bytes)
                          ? 0 : m_max_size_in_bytes - space_used;
  ...
  next_block_size = min(max(next_block_size, num_bytes), space_left);
  if (next_block_size < num_bytes) {
    // 若指针数组有 ≥32KB 的浪费，先 shrink 再试一次
    size_t excess_bytes = (m_record_pointers.capacity() - m_record_pointers.size()) *
                          sizeof(m_record_pointers[0]);
    if (excess_bytes >= 32768) {
      size_t old_capacity = m_record_pointers.capacity();
      m_record_pointers.shrink_to_fit();
      if (m_record_pointers.capacity() < old_capacity) return allocate_block(num_bytes);
    }
    return true;   // We're full.
  }
  return allocate_sized_block(next_block_size);
}
```

**要点**：

- 首块 32KB，之后 ×1.5 指数增长 ⇒ **sort_buffer_size 是预算不是预分配**
- `space_left` 扣除记录指针数组的开销（保守按最大行估），所以小记录时实际用量可能略超预算
- 装不下时先尝试 `shrink_to_fit()` 回收指针数组的浪费（≥32KB 才值得），再判定"满了"
- 返回 true ⇒ `get_next_record_pointer()` 返回空 ⇒ 触发落盘

### sort key 的编码

#### 布局（`sort_param.h` 的权威注释）

```
  |<key a><key b>...    | ( <null row flag> | <rowid> | ) * num_tables    ← rowid 模式
  / m_fixed_sort_length / (  0 or 1 bytes   | ref_len / )

  |<key a><key b>...    |<null bits>|<field a><field b>...|                ← addon 模式
  / m_fixed_sort_length /        addon_length             /

  |<key a><key b>...    |<length>|<null bits>|<field a><field b>...|       ← packed addon
  / m_fixed_sort_length /             addon_length                 /
```

**核心目标**：定长 key 一律可用 `memcmp()` 逐字节比较，不需要知道类型。

#### 长度从哪来：`sortlength()`

- **ENUM / SET 按底层整数排**（注释："Sort enum and set fields as their underlying ints"）⇒ **ENUM 的排序顺序是定义顺序，不是字符串顺序**
- 时间类型按 INT 排
- 字符串长度 = `strnxfrmlen()`（collation 权重长度），**不是原始字节长度** ⇒ utf8mb4 下 `varchar(200)` 的 key 可能 800+ 字节
- **`NO_PAD` collation（如 utf8mb4_0900_ai_ci）的字符串是变长的**；`PAD SPACE` collation（如 utf8mb4_bin、*_general_ci）是定长且受 `max_sort_length` 截断 ⇒ **8.0 换默认 collation 后 filesort 行为有变化**
- 长度 > 10MB 直接饱和为 `0xFFFFFFFFu`（"无界"），后果是 PQ 不可用

#### `make_sortkey()` 的关键分支

- **NULL 标志先写 1（默认非 NULL），求值后回填 0**：因为很多 Item 只有在 `val_*()` 被调用后才设置 `null_value`
- **varlen 前缀只在非 NULL 时写**（NULL 时连 4 字节前缀都省了），存的是**含自身 4 字节**的长度
- **DESC 只反转数据本体**（前面已论证）
- **JSON 追加 8 字节 hash**，注释说明用途：`Add hash at the end of sort key to order cut values correctly. Needed for GROUPing, rather than for ORDERing.`（用于 GROUP BY 区分被截断的值，不是排序）
- **addon 部分**：先清长度域+NULL bitmap（**必须 memset，因为 buffer 块是复用的**），表级 NULL 位排在字段 NULL 位之前；packed 时 NULL 字段完全不写数据

#### 字符串一律过 `strnxfrm()`

这解释了两件事：① 排序结果与 collation 一致（`_ci` 大小写不敏感、`_ai` 忽略重音）；② **key 比原字符串长（权重膨胀）且 CPU 成本显著**。

一个细节：varlen 字符串若 `strnxfrm` 结果**刚好填满**可用空间，被视为溢出（无法区分"刚好"和"真的不够"），这是保守设计。

另一个：`change_double_for_sort()` 的注释坦白——

> NOTE: This does not sort infinities or NaN correctly.

即 **filesort 不保证 ±Inf / NaN 的正确排序**。

**★ `strnxfrm` 的两条调用路径与"刚好填满算溢出"**（`filesort.cc:1265`）：

```cpp
      size_t actual_length;
      if (is_varlen) {
        size_t max_length = to_end - to;
        if (max_length % 2 != 0) {
          // Heed the contract that strnxfrm needs an even number of bytes.
          --max_length;                 /* ★★ strnxfrm 要求偶数长度，奇数就退 1 */
        }
        actual_length = cs->coll->strnxfrm(
            cs, to, max_length, item->max_char_length(),
            pointer_cast<const uchar *>(from), src_length, 0);   /* ★ 不 pad */
        if (actual_length == max_length) {
          /*
            The sort key eithen fit perfectly, or overflowed; we can't
            distinguish between the two, so we have to count it as overflow.
          */
          return UINT_MAX;              /* ★★ 保守设计 */
        }
      } else {
        actual_length = cs->coll->strnxfrm(
            cs, to, dst_length.value(), item->max_char_length(),
            pointer_cast<const uchar *>(from), src_length,
            MY_STRXFRM_PAD_TO_MAXLEN);  /* ★ 定长：PAD 到等长保证 memcmp 可比 */
        assert(actual_length == dst_length.value());
      }
```
> ★ 代价：`val_str(tmp_buffer)` 一次物化拷贝 + `strnxfrm` 一次权重变换 + key 膨胀。`tmp_buffer` 是 **`Sort_param::tmp_buffer`**（复用的 `String`），不是优化。

**★ JSON 走独立路径**（`make_json_sort_key()`，`filesort.cc:1128`）：

```cpp
/**
  Make a sort key for the JSON value in an Item.
  ...
  @param[in,out] hash    The hash key of the JSON values in the current row.
  @returns length of the key stored
*/
NO_INLINE
static uint make_json_sort_key(Item *item, uchar *to, uchar *null_indicator,
                               size_t length, ulonglong *hash) {
  Json_wrapper wr;
  if (item->val_json(&wr)) { ... }
  ...
  size_t actual_length = wr.make_sort_key(to, length);   /* ★ JSON 必然 varlen */
  *hash = wr.make_hash_key(*hash);      /* ★★ 截断的 JSON 靠这个 hash 区分相等性 */
  return actual_length;
}
```
> ★ 为什么需要 hash：JSON key 会被截断，两个"前缀相同但整体不同"的值会误判相等——hash 用于兜底区分。
> ★ `make_sortkey()` 末尾会把 8 字节 hash 拼在 key 尾部（`int8store(to, hash)`），且 `enum_sort_algorithm` 里这个 hash 只在 `use_hash`（有 JSON key）时启用。
> ★ 配套的 `Filesort_error_handler`（`filesort.cc:794`）只为 JSON 服务：**把重复的 `ER_NOT_SUPPORTED_YET` warning 压成一条**。

### addon fields 与 `field->pack()`

**addon 存什么**：`get_addon_fields` 遍历表的所有列，凡在 `read_set_internal` 里的（查询要读取的列）就登记为 addon 字段；`make_sortkey` 之后逐列调 `field->pack()` 把值序列化进记录。

**⚠️ 一个值得注意的"过度包含"**：源码自己承认标准太宽——

> This this a too strong condition. Actually we need only the fields referred in the result set.

以及 `read_all_rows()` 的 NOTE：`SELECT a FROM t1 WHERE b ORDER BY c` 里 **c 已经在 key 里，但 addon 里仍会存一份 c 的副本**。这是 `row_size` 常常比预期大的原因之一。

**`field->pack()` 是紧凑序列化，不是 memcpy 整段**：

```
record[0] 列存储格式（源）                  addon 字节流（目标）
┌──────────────────────────────┐  pack()  ┌──────────────────────────┐
│ VARCHAR(100) "abc":          │ ───────▶ │ [0x03 0x00]"a""b""c"     │
│   [长度2B]"a""b""c"(空洞)     │           │   （长度前缀+数据，无空洞）│
│ CHAR(10) "abc     ":         │           │ [0x03]"a""b""c"          │
│   "a""b""c"+7空格(定长无前缀) │           │   （去尾空+长度前缀）      │
│ BLOB "hello":                │           │ [长度4B]"hello"           │
│   [长度4B][外置指针]          │           │   （指针解引用成真实数据） │
└──────────────────────────────┘           └──────────────────────────┘
```

**三个长度概念**（理解 pack 的关键）：

| 名字 | 含义 | 谁用 |
|---|---|---|
| `pack_length()` | 字段在 `record[0]`（内存行）的存储字节数 | `Field::pack` 默认 memcpy 长度 |
| `pack_length_in_rec()` | 字段在引擎存储行（磁盘行）的字节数 | InnoDB record 格式 |
| `max_packed_col_length()` | `pack()` 写进 addon 缓冲的**最大**字节数 | filesort 预留 addon slot |

> filesort 预留 slot 用的是 `max_packed_col_length()`，它常与 `pack_length()` 不等（CHAR/BLOB 尤甚）。

**各类型的 pack 行为**：

| 类型 | 是否重写 `pack()` | addon 格式 |
|---|---|---|
| 整数（TINY/SMALL/INT/BIGINT） | 否（走基类） | 原始二进制（1/2/4/8 字节小端） |
| `Field_real`（FLOAT/DOUBLE） | 是 | IEEE-754 二进制 |
| `Field_new_decimal` | 否 | `bin_size` 字节压缩十进制 |
| `Field_string`（CHAR） | 是 | 1/2 字节长度前缀 + **去尾空格**数据 |
| `Field_varstring`（VARCHAR） | 是 | 1/2 字节长度前缀 + 数据 |
| `Field_blob`（BLOB/TEXT） | 是 | 长度 + **真实外置数据**（解引用指针） |
| `Field_json` / `Field_geom` | 否（blob 子类） | 同 BLOB |
| `Field_enum` | 是 | packlength 字节整数 |

> 注意：**字段层面没有"不能 pack 的类型"**，每个 `Field` 都有 `pack()`。决定"整趟排序改用 rowid"的是 `decide_addon_fields` / `SortWillBeOnRowId`，不是某个字段不能 pack。

### rowid 载体与 `handler::ref`

rowid 模式下 `make_sortkey` 的 else 分支直接 `memcpy(to, table->file->ref, table->file->ref_length)`——拷的是 **`handler::ref`**，即 server 层的**行位置抽象**（`position()` 写入 / `rnd_pos()` 读出）。

**`ref` 从哪来**：每读到一行后**显式调 `position()`** 填 `ref`，`make_sortkey` 再 memcpy——不是扫描接口内部自动填的。

> **注意**：这里的 rowid 是 server 层 `handler::ref` 概念，**不是** InnoDB 的 `DB_ROW_ID`——InnoDB 恰好把聚簇键塞进 `ref`，那是它的实现细节，server 层并不知道也不关心。语义详见 [`../../handler.md`](../../handler.md) 的「行定位」章节。

**rowid 保留的三个场景**（8.0.20 后）：

1. `force_sort_rowids`：UPDATE/DELETE 两阶段读——要回表**改行**，必须定位原始行
2. FTS：MATCH 直接向 handler 要当前行
3. 含大 BLOB：`row_contains_blob`，BLOB 太大不值得 pack

### 排序算法的选择

```cpp
  const bool prefilter_nth_element =
      max_output_rows < num_input_rows / 2 && !param->m_remove_duplicates;

  if (param->using_varlen_keys()) { ... sort(...); }              // 变长 key
  else if (num_input_rows <= 100) { sort(...); }                  // 小数据
  else { stable_sort(...); }                                      // 大数据
```

完整决策表：

| 条件 | 算法 | 比较器 |
|---|---|---|
| `max_output_rows == 0` / `num_input_rows <= 1` | **不排** | — |
| 变长 key（含 JSON） | `std::sort` | `Mem_compare_varlen_key` |
| 定长 key 且 ≤ 100 行 | `std::sort`（introsort） | `key_len < 10` → `Mem_compare`，否则 `Mem_compare_longkey` |
| 定长 key 且 > 100 行 | `std::stable_sort`（归并） | 同上 |

**为什么大数据用 `std::stable_sort`？** 源码注释：

> std::stable_sort has some extra overhead in allocating the temp buffer, which takes some time. The cutover point where it starts to get faster than quicksort seems to be somewhere around 10 to 40 records. So we're a bit conservative, and stay with quicksort up to 100 records.

配套单测的注释更直接：`Stable sort seems to be faster for all test cases, on all platforms.` 原因是**被排序的是指针数组**，`stable_sort` 的归并是顺序访问 + 顺序写，cache 友好。

但注意 `Sort_param` 里的提醒：

> NOTE: Even with `FILESORT_ALG_STD_STABLE`, we do not necessarily have a stable sort if spilling to disk; this is purely a performance option.

⇒ **落盘归并时稳定性不保证**（chunk 内部稳定，跨 chunk 用堆，胜者是"最小 key"而非"最先插入"）。真正想要全局稳定，靠的是 rowid 模式下 key 尾部拼上 rowid。

**比较器的选择**：`key_len < 10` 用逐字节循环（避免函数调用，多数在前几字节分出胜负）；长 key 先手工比前 4 字节再交给 `memcmp`（可利用字长/向量化的库实现）。

**去重**：`std::unique` + `Equality_from_less`（`!(a<b || b<a)`），只在**同一已排序数组内**生效；跨 chunk 的去重由归并期的 `m_last_key_seen` 完成。

**★ `sort_buffer()` 完整实现**（`filesort_utils.cc:131`）：

```cpp
size_t Filesort_buffer::sort_buffer(Sort_param *param, size_t num_input_rows,
                                    size_t max_output_rows) {
  param->m_sort_algorithm = Sort_param::FILESORT_ALG_NONE;
  if (max_output_rows == 0) return max_output_rows;
  if (num_input_rows <= 1) return num_input_rows;
  if (param->max_compare_length() == 0) return num_input_rows;

  const auto it_begin = begin(m_record_pointers);
  auto it_end = begin(m_record_pointers) + num_input_rows;

  // Due to LIMIT, we don't need to sort all of the elements, so find out
  // which ones that we need and sort only those. This reduces the total
  // running time from O(n log n) to O(n + k log k), which is a significant
  // win for small k. nth_element() isn't guaranteed to be a stable sort,
  // though, so we can only use it if an unstable one is okay.
  const bool prefilter_nth_element =
      max_output_rows < num_input_rows / 2 && !param->m_remove_duplicates;

  if (param->using_varlen_keys()) {
    const Mem_compare_varlen_key comp(param->local_sortorder, param->use_hash);
    if (prefilter_nth_element) {
      nth_element(it_begin, it_begin + max_output_rows - 1, it_end, comp);
      it_end = it_begin + max_output_rows;
    }
    // TODO: Make more elaborate heuristics than just always picking std::sort.
    param->m_sort_algorithm = Sort_param::FILESORT_ALG_STD_SORT;
    sort(it_begin, it_end, comp);
    ...
  }

  // If we don't use addon fields, we'll have the record position appended to
  // the end of each record. This disturbs our equality comparisons, so we'll
  // have to remove it.
  size_t key_len = param->max_compare_length();
  if (!param->using_addon_fields()) {
    key_len -= param->sum_ref_length;   /* ★★ rowid 模式下把 ref 从比较长度里剔除 */
  }

  /*
    std::stable_sort has some extra overhead in allocating the temp buffer,
    which takes some time. The cutover point where it starts to get faster
    than quicksort seems to be somewhere around 10 to 40 records.
    So we're a bit conservative, and stay with quicksort up to 100 records.
  */
  if (num_input_rows <= 100) {          /* ★ 阈值常量 100 */
    if (key_len < 10) {                 /* ★ 阈值常量 10 */
      param->m_sort_algorithm = Sort_param::FILESORT_ALG_STD_SORT;
      ...
      sort(it_begin, it_end, Mem_compare(key_len));
    }
    ...
  }
  param->m_sort_algorithm = Sort_param::FILESORT_ALG_STD_STABLE;  /* ★ >100 行 */
  ...
}
```

**★ 两个比较器**（`filesort_utils.cc:52`）：

```cpp
/*
  An inline function which does memcmp().
  This one turns out to be pretty fast on all platforms, except sparc.
  See the accompanying unit tests, which measure various implementations.
 */
inline bool my_mem_compare(const uchar *s1, const uchar *s2, size_t len) {
  for (size_t i = 0; i < len; ++i) {
    if (s1[i] != s2[i]) return s1[i] < s2[i];
  }
  return false;
}

#define COMPARE(N) if (s1[N] != s2[N]) return s1[N] < s2[N]

inline bool my_mem_compare_longkey(const uchar *s1, const uchar *s2, size_t len) {
  COMPARE(0); COMPARE(1); COMPARE(2); COMPARE(3);
  return memcmp(s1 + 4, s2 + 4, len - 4) < 0;   /* ★ 先内联比 4 字节再交给 libc */
}
```

**★ "稳定性不保证"的官方坦白**（`sort_param.h:455`）：

```cpp
  // NOTE: Even with FILESORT_ALG_STD_STABLE, we do not necessarily have a
  // stable sort if spilling to disk; this is purely a performance option.
  enum enum_sort_algorithm {
    FILESORT_ALG_NONE,
    FILESORT_ALG_STD_SORT,
    FILESORT_ALG_STD_STABLE
  };
```

**★ `nth_element` 的两个阈值常量**：`max_output_rows < num_input_rows / 2`（**2**）与 `key_len < 10`（比较器选择）、`num_input_rows <= 100`（算法选择）。

### LIMIT 优化：nth_element 与优先队列

#### `nth_element` 预筛（普通排序路径）

复杂度从 `O(n log n)` 降到 `O(n + k log k)`。触发条件：`LIMIT < 行数/2` 且**不去重**。

#### 优先队列 PQ

`check_if_pq_applicable()` 的判据：

1. 必须有 LIMIT
2. **不能去重**（`duplicate removal not supported yet`）
3. `LIMIT + 2 < UINT_MAX`，且单行长度不是"无界"
4. **源集能全装进内存时**：只有 `LIMIT < 行数 / 3` 才用 PQ
5. **源集装不下时**：只要内存装得下 `LIMIT+1` 条就用 PQ（因为能**完全避免落盘**，收益远超常数）

第 4 条的根据是注释：

```cpp
  /*
    How much Priority Queue sort is slower than qsort.
    Measurements (see unit test) indicate that PQ is roughly 3 times slower.
  */
  const double PQ_slowness = 3.0;
```

**PQ 为什么慢**：每条记录都要一次 `make_sortkey` + 堆调整（O(log k)），而全排是 O(n log n) 次纯 `memcmp`。当 k 接近 n 时，`n log k` ≳ `n log n`，且 PQ 常数大 3 倍。

**堆的不变式（k+1 槽的巧思）**：队列预分配 `LIMIT + 1` 个槽，是大顶堆（`top()` = key 最大者 = 排序序里最靠后的那条）。满员后：

```cpp
    if (m_queue.size() == m_queue.capacity()) {
      const Key_type &pq_top = m_queue.top();
      const uint rec_sz = m_sort_param->make_sortkey(pq_top, element_size, opaque);
      ...
      m_queue.update_top();
```

即把新行**直接写进 top 指向的槽**（原地覆盖），然后下沉。为什么无条件覆盖仍然正确？因为多出的那 1 个槽使不变式成立："目前已见过的 k 个最小行一定都在队列里"，多出的那个是候选垃圾。最终排完再取 `min(count, max_rows)` 条。

**PQ 的代价**：不能 pack（定长槽）、不能去重、按**最坏行宽**预分配（`(k+1) × max_record_length`）——LIMIT 很大而行很宽时会吃掉大量内存甚至失败。

**★ `Bounded_queue::push()`：满员后 `update_top()` 的巧思**（`bounded_queue.h:33`）：

```cpp
  template <class Opaque>
  [[nodiscard]] bool push(const Opaque &opaque) {
    /*
      Add one extra byte to each key, so that sort-key generating functions
      won't be returning out-of-space. ...
     */
    const uint element_size = m_element_size + 1;

    if (m_queue.size() == m_queue.capacity()) {
      const Key_type &pq_top = m_queue.top();
      const uint rec_sz =
          m_sort_param->make_sortkey(pq_top, element_size, opaque);  /* ★ 原地重写堆顶那块 buffer */
      if (rec_sz == UINT_MAX) return true;
      assert(rec_sz <= m_element_size);
      m_queue.update_top();      /* ★★ 等价于 pop+push，但只做一次 sift-down，省一半比较 */
      return false;
    } else {
      const uint rec_sz = m_sort_param->make_sortkey(
          m_sort_keys[m_queue.size()], element_size, opaque);
      if (rec_sz == UINT_MAX) return true;
      return m_queue.push(m_sort_keys[m_queue.size()]);
    }
  }
```
```cpp
  bool init(ha_rows max_elements, Key_generator *sort_param, Key_type *sort_keys) {
    ...
    // We allocate space for one extra element, for replace when queue is full.
    if (m_queue.reserve(max_elements + 1)) return true;
    // We cannot have packed keys in the queue.        /* ★★ PQ 下不能 pack 的源码依据 */
    m_queue.m_compare_length = sort_param->max_compare_length();
    ...
  }
```

**★ PQ 比较器：依然是纯 memcmp 升序**（`filesort.cc:121`）：

```cpp
struct Mem_compare_queue_key {
  bool operator()(const uchar *s1, const uchar *s2) const {
    if (m_param)
      return cmp_varlen_keys(m_param->local_sortorder, m_param->use_hash, s1, s2);
    // memcmp(s1, s2, 0) is guaranteed to return zero.
    return memcmp(s1, s2, m_compare_length) < 0;   /* ★ DESC 已编码进 key，比较器无方向概念 */
  }
  size_t m_compare_length;
  Sort_param *m_param;
};
```

**★ PQ 启用分支：为什么 PQ 下不调 `try_to_pack_addons()`**（`filesort.cc:475`）：

```cpp
  } else {
    DBUG_PRINT("info", ("filesort PQ is not applicable"));
    filesort->using_pq = false;
    param->using_pq = false;

    /*
      When sorting using priority queue, we cannot use packed addons.
      Without PQ, we can try.
    */
    param->try_to_pack_addons();      /* ★★ 只在非 PQ 分支调用 */

    if (num_rows_estimate < MERGEBUFF2) num_rows_estimate = MERGEBUFF2;
                                      /* ★ MERGEBUFF2(15) 在此作"至少按 15 行估"的下限 */
    ha_rows keys =
        memory_available / (param->max_record_length() + sizeof(char *));
                                      /* ★ 记录体 + 指针，两者共享 sort_buffer_size 预算 */
    param->max_rows_per_buffer =
        min(num_rows_estimate > 0 ? num_rows_estimate : 1, keys);
```

### 外部归并

#### chunk 落盘：`write_keys()`

顺序很重要：**先排序，再写盘**。因为每个 chunk 内部必须有序；同时把 LIMIT 之外的行直接丢掉（若某行在本 chunk 内排在 k 名之后，它不可能进入全局前 k 名）。

两个 `IO_CACHE`（`tempfile` 存记录流、`chunk_file` 存 `Merge_chunk` 描述符数组）都是**惰性创建**——第一次溢出时才真正建文件，这就是"小排序 0 字节磁盘 I/O"的实现基础。

`open_cached_file()` 的语义：*"The actual file is created when the IO_CACHE buffer gets filled"* —— 同时提供 64KB 缓冲把小写聚合、以及 `reinit_io_cache()` 让同一句柄在写/读间切换（filesort 的 ping-pong 需要这个）。

#### 多路归并：`merge_many_buff()`

```cpp
while (num_chunks > MERGEBUFF2) {                       // MERGEBUFF2 = 15
  for (i = 0; i < num_chunks - MERGEBUFF * 3U / 2U; i += MERGEBUFF)   // MERGEBUFF = 7
    merge_buffers(..., &chunk_array[i], MERGEBUFF);
  ...
  std::swap(from_file, to_file);                        // 乒乓
}
```

`merge_buffers()` 是 k 路归并，用 `Priority_queue<Merge_chunk*>`：

```cpp
  while (queue.size() > 1) {
    merge_chunk = queue.top();                     // 当前 key 最小的 chunk
    copy_row(to_file, from_file, merge_chunk, ...); // 输出
    merge_chunk->advance_current_key(row_length);
    if (0 == merge_chunk->mem_count()) {           // 该 chunk 内存段读完
      if (!read_to_buffer(...)) { queue.pop(); reuse_freed_buff(...); break; }
    }
    queue.update_top();                            // key 变了，重新下沉
  }
```

**两个巧思**：

1. **`i < num_chunks - MERGEBUFF*3/2`** —— 保证最后一组至少保留 10 个 chunk，避免产生极小的 chunk（合并开销 > 收益）
2. **只剩 1 个 chunk 时退出堆**，让它独占整个 sort buffer 顺序拷贝——堆归并在单 chunk 时纯属浪费

**归并缓冲等分**：每个 chunk 分一段连续区间；某 chunk 空了之后它的内存区间会**并给相邻 chunk**（`reuse_freed_buff`，要求两块首尾相接）。

**变长记录的读**：`read_to_buffer()` 按字节数尽量读，然后**逐条扫长度域把最后那条不完整的"切掉"**。有绝境兜底：如果连第一条都装不下（缺的只是 addon 部分），则"假装读了这一条"，剩余字节在真正拷贝时**从文件直接搬到文件**（注释：*"we can still merge the row and only stream the addon fields from disk to disk"*），避免单行过大导致归并失败。

**最终归并** `merge_index()` 就是 `merge_buffers(include_keys=false)`——只写 payload，不写排序 key（之后不再需要比较）。

---

### ★ 读回侧：四个排序结果迭代器

上面几节讲的都是**写侧**（怎么排）。排完之后怎么读出来，是另一半——`SortingIterator` 在 `Init()` 里根据结果落在哪、以什么形态存在，从**四个**迭代器里挑一个，之后 `Read()` 全部代理给它。

#### 第一个维度：内存还是文件

源码总纲注释：

```
// Readers relating to reading sorted data (from filesort).
//
// Filesort will produce references to the records sorted; these
// references can be stored in memory or in a temporary file.
//
// The temporary file is normally used when the references doesn't fit into
// a properly sized memory buffer. For most small queries the references
// are stored in the memory buffer.
//
// The temporary file is also used when performing an update where a key is
// modified.
```

#### ★ 第二个维度：direct 还是 indirect（真正的分界）

⚠️ **常见误解**：以为 `Indirect` 变体的理由是"addon fields 是二级指针，要多解一层"。源码里确实有二级指针（`Filesort_buffer` 的 `m_record_pointers`，注释：*"It is these pointers that are actually sorted in filesort"*），但**内存分支的 direct 与 indirect 都走这一层**（`SortBufferIterator::Read()` 就是 `m_sort->get_sorted_record(...)`）。所以"多解一层指针"**不是** direct/indirect 的分界。

**真正的分界是：排序记录里 sort key 之后挂的是「addon 字段（整行有效载荷）」还是「rowid（行引用）」。**

硬证据一，`Filesort::using_addon_fields()` 的注释：

```cpp
  /// Whether we are using addon fields (sort entire rows) or not (sort row
  /// IDs). ...
```

硬证据二，`Addon_fields_status` 枚举把"为什么不能用 addon fields"列全了——这就是 indirect 变体的存在理由清单：

```cpp
enum class Addon_fields_status {
  unknown_status,
  using_addon_fields,

  // The remainder are reasons why we are _not_ using addon fields.
  fulltext_searched,
  keep_rowid,
  row_not_packable,
  row_contains_blob,
  skip_heuristic,
  using_priority_queue
};
```

硬证据三（最关键），`Filesort::m_force_sort_rowids`：

```cpp
  // If true, we will always sort references to rows on table (and crucially,
  // the result iterators used will always position the underlying table on
  // the original row before returning from Read()).
  bool m_force_sort_rowids;
```

**这句是四个类存在意义的核心：indirect 变体保证"从 `Read()` 返回时，底层表的游标停在那一行上"。** UPDATE / DELETE（要改原行）、`SELECT ... FOR UPDATE`、需要 rowid 做 weedout 的场景、`Unique` 的结果——都必须满足这个不变量。而 direct 变体只把字段值 unpack 到各 `Field::ptr`，**表本身没被定位**，所以那些场景不能用。

硬证据四，`get_start_of_payload()` 从二进制布局上确认两种格式：

```cpp
  /// Skips the key part, and returns address of payload.
  uchar *get_start_of_payload(uchar *p) const {
    size_t offset = using_varlen_keys() ? uint4korr(p) : max_compare_length();
    if (!using_addon_fields() && !using_varlen_keys())
      offset -= sum_ref_length;  // The reference is also part of the sort key.
    return p + offset;
  }
```

即：非 addon、非变长 key 时 **rowid 是 sort key 的一部分**（payload 起点要回退 `sum_ref_length`）。

硬证据五，`SortWillBeOnRowId()` 给的具体场景（全文检索）：

```cpp
    // MATCH() (except in "boolean mode") doesn't use the actual value,
    // it just goes and asks the handler directly for the current row.
    // Thus, we need row IDs, so that the row is positioned correctly.
```

#### 2×2 汇总

| | **direct**（addon fields，整行在排序记录里） | **indirect**（只有 rowid，要 `ha_rnd_pos` 回表） |
|---|---|---|
| **内存** | `SortBufferIterator<Packed>`：从 `Filesort_info` 的 filesort buffer 直接取，unpack 到 `Field::ptr`，**不定位表** | `SortBufferIndirectIterator`：从 `sorted_result` 取 (null flag + rowid)，`ha_rnd_pos` 回表，**定位表** |
| **文件** | `SortFileIterator<Packed>`：从 IO_CACHE 读到 addon_buf 再 unpack，**不定位表** | `SortFileIndirectIterator`：从 IO_CACHE 读 ref 再 `ha_rnd_pos`，**定位表** |

几个实现细节值得记：

- `SortFileIterator::Init()` 是**空实现**——排序结束时 `outfile` 已被 `reinit_io_cache(..., READ_CACHE, ...)` 定位到文件头，无需再 Init
- `SortBufferIndirectIterator::Init()` 里有两件读侧独有的事：先 `ha_index_or_rnd_end()`（因为源迭代器可能还开着索引读），以及对全文检索调 `EndFullTextIndexScan()`
- indirect 的多表记录布局是 `null flag + rowid` 逐表拼接（`m_sum_ref_length`）
- 直接读回时 `Read()` 会跳过被删行：`if (tmp == HA_ERR_RECORD_DELETED || ...) skip = true`

#### `SortingIterator::Init()` 如何选择

```cpp
  if (m_sort_result.io_cache && my_b_inited(m_sort_result.io_cache)) {
    // Test if ref-records was used
    if (m_fs_info.using_addon_fields()) {
      if (m_fs_info.addon_fields->using_packed_addons())
        m_result_iterator.reset(
            new (&m_result_iterator_holder.sort_file_packed_addons)
                SortFileIterator<true>(...));
      else
        m_result_iterator.reset(
            new (&m_result_iterator_holder.sort_file) SortFileIterator<false>(...));
    } else {
      m_result_iterator.reset(
          new (&m_result_iterator_holder.sort_file_indirect)
              SortFileIndirectIterator(...));
    }
    m_sort_result.io_cache = nullptr;  // The result iterator has taken ownership.
  } else {
    assert(m_sort_result.has_result_in_memory());
    ...  // 对称的 SortBufferIterator / SortBufferIndirectIterator
  }
```

三个判据：① 是否落盘（io_cache 是否初始化）② 是否 addon fields（direct vs indirect）③ 是否 packed addons（模板实参）。

两个细节：

- 文件分支末尾 `m_sort_result.io_cache = nullptr` 是**所有权转移**（由结果迭代器的析构 `close_cached_file` + `my_free`）
- 四个 `new (&m_result_iterator_holder.xxx)` 是 placement new 到 `SortingIterator` 内预置的 union 里，避免每次排序都做一次堆分配

`SortingIterator::Read()` 只是代理；`SetNullRowFlag()` 由 `SortingIterator` 自己处理，四个子迭代器的 `SetNullRowFlag` 一律 `assert(false)`。

#### 模板参数 `Packed_addon_fields` 的含义

```cpp
  @tparam Packed_addon_fields Are the addon fields packed?
     This is a compile-time constant, to avoid if (....) tests during execution.
```

做成模板参数就是为了**把逐行循环里的分支提到编译期**。区别是：packed 时记录**变长**，要先读 `size_of_length_field` 字节的长度头再读剩余；非 packed 时定长一次读完。`unpack_addon_fields<Packed>()` 里同样分叉——packed 时由 `field->unpack()` 返回值推进指针，非 packed 时固定加 `addonf.max_length`。

#### 与写侧的交接

写侧产物落在 `Sort_result`（注释写清三种归宿）：

```cpp
  /**
    If the entire result fits in memory, we skip the merge phase.
    We may leave the result in the parent Filesort_info's filesort_buffer
    (indicated by sorted_result_in_fsbuf), or we may strip away
    the sort keys, and copy the sorted result into a new buffer.
    Unique always uses the latter.
  */
```

`save_index()` 决定内存分支的产物形态：**addon 时零拷贝**（结果留在 filesort buffer，`sorted_result_in_fsbuf = true`）；**非 addon 时把 sort key 剥掉、只把 rowid 区 memcpy 进新 buffer**（这正是 `SortBufferIndirectIterator::Init()` 里 `m_cache_pos = m_sort_result->sorted_result.get()` 的那块内存）。

---

## 深潜补充：字节布局与边界行为

### write_keys 之外的重试协议

`alloc_and_make_sortkey`：`make_sortkey` 返回 **`UINT_MAX` = 溢出**；外层把 `min_bytes = sort_key_buf.size() + 1` 后重试，成功后才 `commit_used_memory(rec_sz)`。即 **sort buffer 的记录长度是"探测式"增长的，不是预先算准的**。

### packed addon 的启用条件

`try_to_pack_addons`：`m_packable_length < 10 + 4`（即 <14 字节）→ `skip_heuristic` 不 pack。packable 只对 nullable 或 STRING/VARCHAR/BLOB 类字段累加。

### sort_buffer 的二维决策

除 `num_input_rows <= 100` 外，**每个分支内部还按 `key_len < 10`** 选 `Mem_compare` vs `Mem_compare_longkey`。`prefilter_nth_element` 的条件是 `max_output_rows < num_input_rows/2 && !m_remove_duplicates`——**去重与 nth_element 互斥**。

### 归并的两个常量

- `MERGEBUFF = 7`、`MERGEBUFF2 = 15`
- 只剩 1 个 chunk 时退出堆，独占整个 buffer 顺序拷贝

**★ 常量定义与精确语义**（`sql/sql_sort.h:38`，⚠️ 两者含义不同）：

```cpp
/* Defines used by filesort and uniques */

constexpr size_t MERGEBUFF = 7;      /* ★ 每轮实际归并扇入 = 7 */
constexpr size_t MERGEBUFF2 = 15;    /* ★ 阈值：chunk 数 > 15 才继续多轮归并 */
// Number of bytes used to store varlen key's length
constexpr size_t VARLEN_PREFIX = 4;
```

**★ 多趟乒乓归并**（`merge_many_buff.h:37`）：

```cpp
  if (num_chunks <= MERGEBUFF2) return false;   /* ★ ≤15 个 chunk 直接交给 merge_index 一把梭 */

  IO_CACHE *from_file = t_file;
  IO_CACHE *to_file = &t_file2;

  while (num_chunks > MERGEBUFF2) {
    ...
    for (i = 0; i < num_chunks - MERGEBUFF * 3U / 2U; i += MERGEBUFF) {
                                     /* ★ 7*3/2 = 10：给最后一次合并预留 ≥10 个 chunk，
                                          避免最后一轮只剩 1 个（退化成纯拷贝） */
      if (merge_buffers(thd, param, from_file, to_file, sort_buffer,
                        last_chunk++,
                        Merge_chunk_array(&chunk_array[i], MERGEBUFF),
                        /*include_keys=*/true))
        goto cleanup;
    }
    if (merge_buffers(thd, param, from_file, to_file, sort_buffer, last_chunk++,
                      Merge_chunk_array(&chunk_array[i], num_chunks - i),
                      /*include_keys=*/true))
      break;
    ...
    std::swap(from_file, to_file);   /* ★★ 双缓冲乒乓：本轮输出即下轮输入 */
    num_chunks = last_chunk - chunk_array.begin();
  }
```
> ★ 归并缓冲按"放得下 `MERGEBUFF2=15` 个 key"切分（`filesort.cc:1890` 的 Doxygen）。
> ★ 归并代价模型有个**镜像实现**在 `sql/uniques.cc:471`（`get_merge_many_buffs_cost()`，注释 `Do it exactly as merge_many_buff function does`）——这是"外部归并排序"在代码里最直接的理论溯源证据。

**★ chunk 落盘 `write_keys()`**（`filesort.cc:1069`）：

```cpp
static int write_keys(Sort_param *param, Filesort_info *fs_info, uint count,
                      IO_CACHE *chunk_file, IO_CACHE *tempfile) {
  Merge_chunk merge_chunk;
  count = fs_info->sort_buffer(param, count, param->max_rows);  /* ★ 先排序，再顺序写盘 */
  ...
  merge_chunk.set_file_position(my_b_tell(tempfile));
  merge_chunk.set_rowcount(static_cast<ha_rows>(count));

  for (uint ix = 0; ix < count; ++ix) {
    uchar *record = fs_info->get_sorted_record(ix);
    size_t rec_length = param->get_record_length(record);  /* ★ packed/varlen 下每行长度不同，逐行问 */
    if (my_b_write(tempfile, record, rec_length)) return 1;
  }
  if (my_b_write(chunk_file, pointer_cast<uchar *>(&merge_chunk),
                 sizeof(merge_chunk)))
    return 1;
  return 0;
}
```
> ★ 每个 chunk **内部有序**是先排序后写盘的结果——这是归并排序能成立的前提。
> ⚠️ **`write_keys()` 自身无重试**（`my_b_write` 失败直接 `return 1`）。

**★ 内存态结果的落位**：`save_index()`（`filesort.cc:1562`）

```cpp
  if (param->using_addon_fields()) {
    sort_result->sorted_result_in_fsbuf = true;   /* ★ addon：原地留在 Filesort_buffer，零拷贝 */
    return false;
  }
  ...
  for (uint ix = 0; ix < count; ++ix) {
    uchar *record = table_sort->get_sorted_record(ix);
    uchar *start_of_payload = param->get_start_of_payload(record);
    memcpy(to, start_of_payload, res_length);   /* ★ rowid：剥掉 sort key，只拷 payload */
    to += res_length;
  }
```
> ★★ 注释原话："For addon fields, we keep the result in the filesort buffer. **This saves us a lot of memcpy calls.**"

---

## 演进、性能与坑

### 8.0 的演进

| 变化 | 说明 |
|---|---|
| **块式指数增长** | `sort_buffer_size` 从"一次性 malloc"变成"预算 + 32KB 起 ×1.5 增长"，小排序只付 32KB |
| **`max_length_for_sort_data` 废弃** | 不再按行宽在 addon/rowid 间切换，改为默认 addon |
| **改用 C++ 标准库** | 5.x 的手写 `filesort_compare` → 8.0 的 `std::sort` / `std::stable_sort` + 三种比较器 |
| **addon 默认化** | 注释："Generally, prefer using addon fields if we can" |

**★ 演进证据清单（逐个带源码落点）**：

| 演进点 | 证据 | 位置 |
|---|---|---|
| `SortingIterator` 是 8.0 全新迭代器封装 | 文件头 Copyright `2018, 2024`；"we currently have no Iterators for joins" | `sorting_iterator.h:4,46` |
| `Filesort_buffer` 改为**块式 1.5× 增长** | "allocated incrementally, growing 50% each time, similar to how a MEM_ROOT works" | `filesort_utils.h:44-80` |
| 首块固定 32KB | `next_block_size = MIN_SORT_MEMORY` | `filesort_utils.cc:321` |
| **packed addon** 引入 | `Addon_fields::m_using_packed_addons` + `try_to_pack_addons()` + `Packed_addon_fields` 模板参数 | `sort_param.h:213`；`filesort.cc:264`；`basic_row_iterators.h:150` |
| **varlen key** 引入（JSON/变长串） | `constexpr size_t VARLEN_PREFIX = 4;` + `Mem_compare_varlen_key` | `sql_sort.h:43`；`filesort_utils.cc:150` |
| `max_length_for_sort_data` **彻底失效** | `ulong max_length_for_sort_data; ///< Unused.` + `DEPRECATED_VAR("")` | `system_variables.h:246`；`sys_vars.cc:3078` |
| 结果迭代器改为 **RowIterator + union 零堆分配** | `IteratorHolder` union + placement new | `sorting_iterator.h:139-149` |

```cpp
/* system_variables.h:246 —— ★★ 这个变量已完全不参与 filesort 任何决策 */
  ulong max_length_for_sort_data;  ///< Unused.
```
```cpp
/* sys_vars.cc:3072 */
static Sys_var_ulong Sys_max_length_for_sort_data(
    "max_length_for_sort_data",
    "This variable is deprecated and will be removed in a future release.",
    HINT_UPDATEABLE SESSION_VAR(max_length_for_sort_data),
    CMD_LINE(REQUIRED_ARG), VALID_RANGE(4, 8192 * 1024L), DEFAULT(4096),
    BLOCK_SIZE(1), NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(nullptr),
    ON_UPDATE(nullptr), DEPRECATED_VAR(""));
```

### 性能特征

- **代价模型**：优化器按 `n + k×log2(k)` 建模（对应 nth_element 预筛的复杂度）
- **大结果集慢的三个来源**：落盘写 chunk、多路归并的 I/O、rowid 模式的随机回表
- **`Sort_merge_passes` 状态变量**：每次 `merge_buffers()` 调用 +1，是判断"归并了几轮"的直接指标

**★ 两条源码实证**：

```cpp
/* ① basic_row_iterators.h:177 —— rowid 回表的随机 IO */
/**
  Fetch the record IDs from a memory buffer, but the records themselves from
  the table on disk.
  ... In this case the record data is fetched from the handler using the saved
  reference using the rnd_pos handler call.
 */
```
```cpp
/* ② basic_row_iterators.h:219 —— ★★ "落盘 + addon" 未必比 "内存 + rowid" 慢 */
/**
  Fetch the records from a tempoary file.

  There used to be a comment here saying “should obviously not really happen
  other than in strange configurations”, but especially with packed addons
  and InnoDB (where fetching rows needs a primary key lookup), it's not
  necessarily suboptimal compared to e.g. SortBufferIndirectIterator.
 */
```
> ★★ 这条很有价值：**InnoDB 的 `ha_rnd_pos` 本质是一次主键查找**，所以别默认"落盘一定比内存回表慢"。

### 已知坑

| 坑 | 说明 |
|---|---|
| **collation 影响巨大** | 字符串一律过 `strnxfrm()`，utf8mb4 下 key 严重膨胀；`NO_PAD`（8.0 默认）走变长，`PAD SPACE` 走定长且受 `max_sort_length` 截断 |
| **±Inf / NaN 排序不正确** | `change_double_for_sort` 注释明说 |
| **addon 过度包含** | `SELECT a FROM t1 WHERE b ORDER BY c` 里 c 已在 key 中，addon 仍存一份 |
| **PQ 按最坏行宽预分配** | LIMIT 大 + 行宽时可能吃掉大量内存 |
| **落盘后稳定性不保证** | `std::stable_sort` 只是性能选项，真正稳定靠 rowid 模式尾部拼 rowid |
| **`sort_buffer_size` 是会话级** | 每个可能排序的连接一份（虽然是预算式增长），高并发下要小心 |
| **单行过大直接报错** | 单行超过剩余预算且 chunk 为空 ⇒ `ER_OUT_OF_SORTMEMORY` |
| **归并缓冲连一行都放不下** | `count == 0` ⇒ `ER_OUT_OF_SORTMEMORY`（`filesort.cc:1736`，带 TODO） |
| **NULL 行的 rowid 仍占 1 字节** | `// TODO(sgunders): Allow variable-length ref, so that we don't have to store the row ID for NULL rows.`（`filesort.cc:223`） |
| **TABLE 对象不析构导致内存泄漏** | `Filesort_buffer::free_sort_buffer()` 用 swap 空 vector 的 workaround（`filesort_utils.cc:415`，注释坦白"many places construct TABLE objects and never destroy them properly"） |
| **`Filesort*` 职责混乱** | `// TODO: Clean up so that Filesort / Filesort_info / Filesort_buffer / Sort_result have less confusing overlap.`（`sql_sort.h:153`） |

**★ 最硬的一条坑：read set 过大导致 addon 被动膨胀**（`filesort.cc:946`，★ ORDER BY 的列会重复计入）：

```cpp
  // NOTE(sgunders): When we sort row IDs, our read sets are a bit larger
  // than required by read_all_rows(); in particular, columns that we
  // don't sort on will still be read. (In particular, this makes us read
  // and allocate blobs, where present.) ...
  //
  // This isn't the only case of too large read sets. For a query such as
  // SELECT a FROM t1 WHERE b ORDER BY c, all three fields a,b,c will be in
  // the read set; in particular, find_order_in_list() will include any
  // columns used in ORDER BY in the read sets, as part of resolving them.
  // This is required for correct operation. However, anything that is part of
  // the read set will by extension be included as addon fields,
  // unless we sort row IDs for some reason -- even c, which is part of
  // the key. To remedy this, one would probably need a system
  // of pushing read sets through the iterator tree ...
```
> ★★ 即：**`c` 既在 sort key 里，又在 addon 里**——纯粹因为它在 read_set 中。这是"addon 过度包含"的源码根因。

**★ sort key 长度可以是"无穷大"**（`sort_param.h:88`）：

```cpp
  /// Length of sort field. Beware, can be 0xFFFFFFFFu (infinite)!
  uint length;
```

---

## 相关的系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `sort_buffer_size` | **256 KB**（`DEFAULT_SORT_MEMORY`） | SESSION | 排序内存**预算**（非预分配）；范围 `MIN_SORT_MEMORY`(32KB) ~ ULONG_MAX |
| `max_sort_length` | 1024（待核实） | SESSION | `PAD SPACE` collation 下字符串 sort key 的截断长度 |
| `max_length_for_sort_data` | **已废弃**（`Unused`） | — | 8.0.20 起不再用于在 addon/rowid 间切换 |

**★ 逐个补注册原文**（⚠️ `max_sort_length` 默认 **1024** 已核实）：

```cpp
/* sql/sys_vars.cc:4868 —— sort_buffer_size */
static Sys_var_ulong Sys_sortbuff(
    "sort_buffer_size",
    "Each thread that needs to do a sort allocates a buffer of this size",
    HINT_UPDATEABLE SESSION_VAR(sortbuff_size), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(MIN_SORT_MEMORY, ULONG_MAX), DEFAULT(DEFAULT_SORT_MEMORY),
    BLOCK_SIZE(1));
```
```cpp
/* sql/sys_vars.cc:3123 —— max_sort_length：★ 只对 PAD SPACE 的定长字符串 key 生效 */
static Sys_var_ulong Sys_max_sort_length(
    "max_sort_length",
    "The number of bytes to use when sorting long values with PAD SPACE "
    "collations (only the first max_sort_length bytes of each value are "
    "used; the rest are ignored)",
    HINT_UPDATEABLE SESSION_VAR(max_sort_length), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(4, 8192 * 1024L), DEFAULT(1024), BLOCK_SIZE(1));
```
```cpp
/* sql/sql_const.h:123 —— ★ sort_buffer_size 最小值 = 32768（常见说法"32KB"属实） */
constexpr const size_t MIN_SORT_MEMORY{32 * 1024};
```
```cpp
/* sql/sys_vars.cc:157 —— 默认 256KB */
static constexpr const unsigned long DEFAULT_SORT_MEMORY{256UL * 1024UL};
```

**★ 另三个与 filesort 相关的变量**：

| 变量 | 文件:行 | 默认值 | 注释要点 |
|---|---|---|---|
| `read_buffer_size` | `sys_vars.cc:3637` | **128 KB** | 顺序扫描缓冲（下限 `IO_SIZE*2 = 8192`） |
| `read_rnd_buffer_size` | `sys_vars.cc:3911` | **256 KB** | ★ **"When reading rows in sorted order after a sort, the rows are read through this buffer to avoid a disk seeks"** |
| `tmpdir` | `sys_vars.cc:4011` | nullptr | READ_ONLY；被 `open_cached_file(..., mysql_tmpdir, ...)` 直接消费 |
| `internal_tmp_mem_storage_engine` | `sys_vars.cc:5352` | `TMP_TABLE_TEMPTABLE` | 只有 `_mem_` 版 |

> ⚠️ **真名核查**：8.0.39 **不存在 `internal_tmp_disk_storage_engine`**（全仓仅 2 处注释提及）。
> ★ `read_rnd_buffer_size` 并不被 `SortFileIndirectIterator` 直接用（那里是裸 `ha_rnd_pos`），而是供 handler 层的 MRR / rnd_pos 缓冲。

**★ EXPLAIN 的 "Using filesort" 与真实形态是两回事**（⚠️ 重要区分）：

```cpp
/* sql/opt_explain_format.h:63 —— Extra_tag */
enum Extra_tag { ET_none, ET_USING_TEMPORARY, ET_USING_FILESORT, ... };
```
```cpp
/* sql/opt_explain.cc:1183 —— ★ "Using temporary; Using filesort" 就是这两个 push 的顺序 */
  if (need_tmp_table_arg && push_extra(ET_USING_TEMPORARY)) return true;
  if (need_sort_arg && push_extra(ET_USING_FILESORT)) return true;
```
```cpp
/* sql/filesort.cc:575 —— ★ 四象限的真实形态只写在 optimizer trace 里 */
    sort_mode.append(param->using_packed_addons()
                         ? "packed_additional_fields"
                         : param->using_addon_fields() ? "additional_fields"
                                                       : "rowid");
    ...
    filesort_summary.add("num_initial_chunks_spilled_to_disk", num_initial_chunks)
        .add("peak_memory_used", fs_info->peak_memory_used())
        .add_alnum("sort_algorithm", algo_text[param->m_sort_algorithm]);
    if (!param->using_packed_addons())
      filesort_summary.add_alnum(
          "unpacked_addon_fields",
          addon_fields_text(param->m_addon_fields_status));
    filesort_summary.add_alnum("sort_mode", sort_mode.c_ptr());
```
> ★★ `Using filesort` **只表示"计划里有 SORT 节点"**，与是否落盘、addon、packed **全部无关**。要看真实形态必须开 `optimizer_trace` 读 `filesort_summary.sort_mode`（如 `<fixed_sort_key, packed_additional_fields>`）。

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| `make_sortkey()` 在 `filesort_utils.cc` | **不在** → 实现在 `filesort.cc:1392`，声明 `sort_param.h:401` |
| 有 `Sort_field` 类 | **不存在** → 真名 `st_sort_field`（POD struct，`sort_param.h:86`） |
| 有 `Packed_addon_fields` 类 | **不存在** → 只是 `template <bool Packed_addon_fields>` 的**模板参数名** |
| addon vs rowid 由 `max_length_for_sort_data` 决定 | **已废弃** → `system_variables.h:246` 标注 `///< Unused.` + `DEPRECATED_VAR("")` |
| `SortingIterator` 在 `sql/sql_sort.h` | **不在** → `sql/iterators/sorting_iterator.h:56`；`sql_sort.h` 只有 `Merge_chunk`/`Sort_result`/`Filesort_info` |
| 四迭代器是 `Sort_file_indirect_iterator` 等蛇形名 | **全错** → 大驼峰：`SortBufferIterator`/`SortBufferIndirectIterator`/`SortFileIterator`/`SortFileIndirectIterator` |
| 四迭代器实现在 `basic_row_iterators.cc` | **一个都没有** → 全在 `sql/iterators/sorting_iterator.cc:86-398` |
| sort key 用 `Field::make_sort_key()` 生成 | **filesort 不用** → 直接调 `cs->coll->strnxfrm()`；`Field::make_sort_key` 调用者只有 `rpl_write_set_handler.cc` 与 `item_strfunc.cc` |
| 有 `Field::sort_length()` | **不存在** → 长度全由 `sortlength()` 在 **Item 层**计算 |
| packed addon 长度字段是 2 字节 | **注释过时** → 代码 `size_of_length_field = 4` 且用 `uint4korr` |
| DESC 靠"排序时反向比较" | **错** → 编码期逐字节取反 `*to = ~*to`，比较器永远是同一个升序 memcmp |
| PQ 下可调 `nth_element` / 可 pack | **都错** → PQ 走 `Bounded_queue::push`；`try_to_pack_addons()` 只在 `else` 分支调用 |
| 字符串 key 有"ASCII 直接 memcpy"优化 | **不存在** → filesort 路径所有字符串一律 `strnxfrm`；`tmp_buffer` 只是复用 `String` |
| "含大 BLOB 就用 rowid" | **不精确** → 硬阈值 `MYSQL_TYPE_BLOB && max_packed_col_length() > 70000`；tinyblob/blob/JSON/geometry 都允许 |
| `MERGEBUFF=7` 就是扇入上限 | **半对** → 7 是每轮实际扇入，`MERGEBUFF2=15` 才是"是否继续多轮"的阈值 |
| `write_keys()` 有落盘失败重试 | **不存在** → `my_b_write` 失败直接 `return 1`；"重试"只在 `alloc_and_make_sortkey` 与 flush 后再试一次 |
| 存在 `internal_tmp_disk_storage_engine` | **不存在** → 只有 `internal_tmp_mem_storage_engine` |

**状态变量**：`Sort_scan`（排序次数）、`Sort_rows`（排序行数）、`Sort_merge_passes`（归并轮数）、`Sort_range`（用索引范围排序的次数）。

---

## 参考

**论文**
- **Graefe《Query Evaluation Techniques for Large Databases》(1993)** —— 外部排序（external merge sort）、sort-merge join 的经典综述

**官方文档**
- *MySQL 8.0 Reference Manual → ORDER BY Optimization*
- *MySQL 8.0 Reference Manual → The Sort Buffer*

**内核月报**
- **2021/06《Order By 优化逻辑代码分析》** —— filesort 的 addon / priority queue / 多轮归并（执行阶段部分）

**相关文档**
- 排序结果落到的临时表（物化载体、TempTable/MEMORY、降级链）见 [`07_temptable.md`](07_temptable.md)
- `handler::ref` / `position()` / `rnd_pos()` 的行定位语义见 [`../../handler.md`](../../handler.md)
- `SortingIterator` 与火山模型的 `Init()` 契约见 [`../09_executor_iterator.md`](../09_executor_iterator.md)
