# 01 filesort 与临时表：排序、物化的物理实现

> 本篇覆盖：`ORDER BY`/`DISTINCT`/`GROUP BY` 背后的两个物理载体——**filesort 排序算法** 与 **temptable 临时表引擎**（算法级）。

## 目录

- [先有个整体印象](#先有个整体印象)
- [一、filesort 排序算法](#一filesort-排序算法)
- [二、临时表（temptable）](#二临时表temptable)
- [三、深潜补充：sort key 字节布局与临时表三级转换](#三深潜补充sort-key-字节布局与临时表三级转换)
- [核心调用栈](#核心调用栈)

---

## 先有个整体印象

`ORDER BY` 走 filesort，`GROUP BY`/`DISTINCT`/物化走临时表。两者关系：**filesort 是把数据"排好序"，temptable 是把数据"存下来（可能带索引）再扫描"**。

```
ORDER BY ──▶ filesort：读入 sort buffer → 排序 →（放不下）写 chunk 落盘 → 归并
GROUP BY ──▶ temptable：逐行写入内存表（带索引）→（超限）转 InnoDB 磁盘表 → 再扫描
```

**默认值速记**：`sort_buffer_size`=256KB、`tmp_table_size`/`max_heap_table_size`=16MB、`temptable_max_ram`=1GiB。

---

## 一、filesort 排序算法

### 1.1 Filesort 类与主流程

`Filesort` 类在 `sql/filesort.h:52`（配置对象），真正执行入口是 `filesort()`（`sql/filesort.cc:367`）。完整流程：

```cpp
bool filesort(THD *thd, Filesort *filesort, RowIterator *source_iterator, ...) {
  ulong memory_available = thd->variables.sortbuff_size;   // sort_buffer_size

  param->init_for_filesort(filesort, ..., sortlength(...)); // 算 key 长度/addon 长度

  // 阶段一：读入所有行；放不下时写 chunk 到临时文件
  num_rows_found = read_all_rows(thd, param, ..., &chunk_file, &tempfile, ..., source_iterator);

  if (num_chunks == 0) {
    save_index(param, rows_in_chunk, fs_info, sort_result);   // 全在内存：直接排序输出
  } else {
    merge_many_buff(thd, param, merge_buf, ..., &num_chunks, &tempfile); // 多轮归并降到 ≤15
    merge_index(thd, param, merge_buf, ..., &tempfile, outfile);         // 最终归并
  }
}
```

四个关键阶段：

1. `read_all_rows()`（`filesort.cc:924`）：逐行 `Read()` 源迭代器 → `make_sortkey()` 构造 sort key + payload。
2. 放不下时 `write_keys()`（`:1089`）：对当前 chunk 排序 → 写 `tempfile` → chunk 描述符写 `chunk_file`。
3. `save_index()`（`:1588`）：全在内存时直接排序，结果留在 sort buffer（addon 模式）或单独拷贝 rowid。
4. `merge_many_buff()`（`merge_many_buff.h:50`）+ `merge_index()`（`:2067`）：多路归并。

### 1.2 sort buffer 与 sort_buffer_size

`sort_buffer_size`（`sys_vars.cc:4869`）默认 **256KB**（`DEFAULT_SORT_MEMORY = 256*1024`，`sys_vars.cc:157`）。

`Filesort_buffer`（`filesort_utils.h:81`）的关键设计：**排序的不是整条记录，而是 `m_record_pointers` 指针数组**。内存块按 50% 指数增长（`allocate_block`，`filesort_utils.cc:316`），首块 32KB，不一次性 malloc 整个 buffer（避免小结果集浪费）。

### 1.3 addon fields vs rowid：排序记录的两种载体

sort key 决定"按什么顺序排"，但排序完要返回**完整行**——这一行的数据从哪来？两种方案：

| | **addon fields（additional_fields）** | **rowid（keep_rowid）** |
|---|---|---|
| 排序记录 | sort key + **列数据拷贝** | sort key + **行位置**（`handler::ref`） |
| 读结果 | `SortBufferIterator` 解包 addon，**不回表** | `SortBufferIndirectIterator` 用 `ha_rnd_pos` **回表取整行** |
| 取舍 | 空间换时间：记录宽，但省回表随机 IO | 时间换空间：记录窄，但读结果要回表 |

**addon 存什么**：`get_addon_fields`（`filesort.cc:2258`）遍历表的所有列，凡在 `read_set_internal` 里的（查询要读取的列），就登记为 addon 字段；`make_sortkey` 写 sort key 之后，逐列调 `field->pack()` 把这些列的**值**序列化进记录。所以 addon = 结果集所需列的完整拷贝。

**`field->pack()` 的格式转换**：源是 `field->field_ptr()` 指向的 **`record[0]` 里的列存储格式**，目标是**排序记录里的 addon 紧凑字节流**：

```
record[0] 列存储格式（源）                  addon 字节流（目标）
┌──────────────────────────────┐  pack()  ┌──────────────────────────┐
│ VARCHAR(100) "abc":          │ ───────▶ │ [0x03 0x00]"a""b""c"      │
│   [长度2B]"a""b""c"(空洞)     │           │   （长度前缀+数据，无空洞）│
│ CHAR(10) "abc     ":         │           │ [0x03]"a""b""c"          │
│   "a""b""c"+7空格(定长无前缀) │           │   （去尾空+长度前缀）      │
│ BLOB "hello":                │           │ [长度4B]"hello"           │
│   [长度4B][外置指针]          │           │   （指针解引用成真实数据） │
└──────────────────────────────┘           └──────────────────────────┘
```

**三个长度概念**（`field.h`，理解 pack 的关键）：

| 名字 | 含义 | 谁用 |
|---|---|---|
| `pack_length()` | 字段在 **`record[0]`（内存行）** 的存储字节数 | `Field::pack` 默认 memcpy 长度 |
| `pack_length_in_rec()` | 字段在**引擎存储行（磁盘行）**的字节数 | InnoDB record 格式 |
| `max_packed_col_length()` | `pack()` 写进 **addon 缓冲**的**最大**字节数 | filesort 预留 addon slot（`filesort.cc:2363`） |

> filesort 预留 addon slot 用的是 `max_packed_col_length()`，它常与 `pack_length()` 不等（CHAR/BLOB 尤甚）。

**`field->pack()` 逐行解析**（addon 的核心机制）：把字段从 record 里的存储表示，序列化成排序记录里的紧凑字节流。不同字段类型实现不同：

```cpp
// 基类（field.cc:1872）：定长字段直接搬二进制
uchar *Field::pack(uchar *to, const uchar *from, size_t max_length) const {
  size_t length = std::min<size_t>(pack_length(), max_length);
  memcpy(to, from, length);          // INT=4字节、DOUBLE=8字节，原样拷
  return to + length;
}
```

```cpp
// Field_varstring（field.cc:6771）：长度前缀 + 字符串字节
uint length = length_bytes == 1 ? (uint)*from : uint2korr(from);  // 从 record 读长度
*to++ = length & 0xFF;                       // 小端长度前缀（1 或 2 字节）
if (length_bytes == 2) *to++ = (length >> 8);
memcpy(to, from + length_bytes, length);     // 实际字符串字节
return to + length;
```

```cpp
// Field_blob（field.cc:7356）：blob 长度 + 从外置存储取数据
store_blob_length(len_buf, packlength, length);
memcpy(to, len_buf, packlength);                      // 先写长度
memcpy(to + packlength, get_blob_data(from + packlength), store_length);  // 再取数据
```

> 关键：`pack` 是**紧凑序列化**，不是简单 `memcpy` 整段——VARCHAR 只存实际长度、BLOB 要追外置存储页取数据。这正是"packed addon"能省空间的根源（见下）。

**子类衍生一览**（各类型 addon 格式）：

| 类型 | 是否重写 `pack()` | addon 格式 |
|---|---|---|
| 整数（TINY/SMALL/INT/BIGINT） | 否（走基类） | 原始二进制（1/2/4/8 字节小端） |
| `Field_real`（FLOAT/DOUBLE） | 是（`field.cc:4017`） | IEEE-754 二进制（小端机器直接 memcpy） |
| `Field_new_decimal`（DECIMAL） | 否（走基类） | `bin_size` 字节压缩十进制（`my_decimal_get_binary_size`） |
| `Field_string`（CHAR） | 是（`field.cc:6398`） | 1/2 字节长度前缀 + **去尾空格**数据 |
| `Field_varstring`（VARCHAR） | 是（`field.cc:6771`） | 1/2 字节长度前缀 + 数据（record 里已有前缀，pack 跳过长度字节） |
| `Field_blob`（BLOB/TEXT） | 是（`field.cc:7356`） | 长度 + 真实外置数据（解引用指针） |
| `Field_json` / `Field_geom` | 否（blob 子类） | 同 BLOB |
| `Field_enum` | 是（`field.cc:8534`） | packlength 字节整数 |
| `Field_bit` | 是（`field.cc:8970`） | 不足一字节的 bit 塞 null 字节 + 其余 memcpy |

> 注意：**字段层面没有"不能 pack 的类型"**，每个 `Field` 都有 `pack()`。决定"整个排序用 rowid 而非 addon"的是 `decide_addon_fields` / `SortWillBeOnRowId`（见下「8.0.20 的转折」），不是某个字段不能 pack。

**rowid 是什么**：`make_sortkey` 的 else 分支（`:1539`）直接 `memcpy(to, table->file->ref, table->file->ref_length)`——拷贝的是 **`handler::ref`**，即 server 层的**行位置抽象**（`position()` 写入 / `rnd_pos()` 读出）。

**`ref` 从哪来**：filesort 在 rowid 模式下，每读到一行后**显式调 `position()`**（`filesort.cc:987`）填 `ref`，然后 `make_sortkey` 再 `memcpy` 这个填好的值——**不是扫描接口内部自动填的**。`position()` 的语义、`key_copy` 的引擎键格式、`rnd_pos` 的回表链路见 [`../../handler.md`](../../handler.md) 的「行定位」章节。

> **注意**：这里的 rowid 是 server 层 `handler::ref` 概念，不是 InnoDB 的 `DB_ROW_ID`——InnoDB 恰好把聚簇键塞进 `ref`，那是它的实现细节，server 层并不知道也不关心。

**8.0.20 的转折**：`max_length_for_sort_data` 已废弃（`system_variables.h` 标注 `Unused`）。旧行为是"行宽超过阈值就改排 rowid 省内存"，8.0.20 起 `decide_addon_fields`（`filesort.cc:162`）改为**默认 always addon**（注释原文 "Generally, prefer using addon fields if we can"）。于是 rowid 模式只剩两个**强制**场景：

- `force_sort_rowids`：UPDATE/DELETE 两阶段读——排序后要 `ha_rnd_pos` 回表改行，必须存 rowid 定位原始行（`sql_update.cc:673`、`sql_delete.cc:534`）
- `SortWillBeOnRowId`（`filesort.cc:2187`）：FTS（MATCH 直接向 handler 要当前行，需 rowid 定位）或含大 BLOB（`row_contains_blob`，BLOB 太大不值得 pack 进 addon）

**packed addon**（`sort_param.h:129`）：addon 模式下的进一步省空间。非 packed 时每个字段按 `max_length` 占满（VARCHAR(300) 存 "abcd" 也占 300 字节），packed 后按实际长度 `field->pack()`，NULL 字段不存，前面加 4 字节总长度：

```
|key a|key b|...|<4字节长度>|<null bits>|<field a><field b>...|
```

`make_sortkey`（`filesort.cc:1392`）构造这条记录：先写 sort key（含 NULL 标志、varlen 前缀、DESC 反转、JSON 的 hash），再写 addon（`field->pack()`）或 rowid（`memcpy(ref)`）。

`sortlength()`（`:2097`）算 key 总长度：STRING 用 `strnxfrmlen` 转 collation 权重长度（>10MB 视为无限），INT=8、DOUBLE=8、DECIMAL 用二进制大小。

### 1.4 归并（merge）

`merge_many_buff()`（`merge_many_buff.h:50`）用常量 `MERGEBUFF=7`、`MERGEBUFF2=15`（`sql_sort.h:40-41`），把 chunk 数降到 ≤15：

```cpp
while (num_chunks > MERGEBUFF2) {
  for (i = 0; i < num_chunks - MERGEBUFF * 3U / 2U; i += MERGEBUFF)
    merge_buffers(..., &chunk_array[i], MERGEBUFF);   // 每 7 个归并成 1 个
  ...
  std::swap(from_file, to_file);                       // 乒乓使用两个文件
}
```

`merge_buffers()`（`filesort.cc:1910`）是 k 路归并，用 `Priority_queue<Merge_chunk*>`：

```cpp
while (queue.size() > 1) {
  merge_chunk = queue.top();                    // 取当前 key 最小的 chunk
  copy_row(to_file, from_file, merge_chunk, ...); // 输出这一行
  merge_chunk->advance_current_key(row_length);  // 该 chunk 前进到下一条
  if (0 == merge_chunk->mem_count()) { read_to_buffer(...); } // 该 chunk 缓冲耗尽，重新读入
  queue.update_top();                           // key 变了，重新下沉
}
```

`merge_index()`（`:2067`）就是 `merge_buffers(include_keys=false)`——最终输出只写 payload，不写排序 key。

### 1.5 排序算法本身

**不是手写的 sortcmp**（那是 5.x）。8.0 用 C++ 标准库，在 `Filesort_buffer::sort_buffer()`（`filesort_utils.cc:131`）：

```cpp
if (param->using_varlen_keys()) {
  // 变长 key：LIMIT 很小时先 nth_element 做 partial sort
  if (prefilter_nth_element) { nth_element(it_begin, it_begin + max_output_rows - 1, it_end, comp); ... }
  sort(it_begin, it_end, comp);                       // Mem_compare_varlen_key
} else if (num_input_rows <= 100) {
  sort(it_begin, it_end, Mem_compare(key_len));       // 小数据 std::sort
} else {
  stable_sort(it_begin, it_end, Mem_compare_longkey(key_len));  // 大数据 stable_sort
}
```

- `≤100` 行用 `std::sort`（快，不稳定无所谓）
- `>100` 行用 `std::stable_sort`（保序，为后续去重）
- 变长 key 用 `Mem_compare_varlen_key` → `cmp_varlen_keys()`（`cmp_varlen_keys.h:46`）：逐字段先比 NULL 标志，变长字段读 4 字节前缀长度再 `memcmp`，最后 JSON 分组用 8 字节 hash 兜底。

### 1.6 LIMIT 优化（priority queue / nth_element）

`check_if_pq_applicable()`（`filesort.cc:1646`）决定是否用 `Bounded_queue`（top-n 堆）：

```cpp
const double PQ_slowness = 3.0;   // PQ 比 qsort 慢约 3 倍（实测）
if (param->max_rows == HA_POS_ERROR) return false;   // 无 LIMIT
if (param->m_remove_duplicates) return false;        // 去重不支持 PQ
...
if (num_rows < num_available_keys) {                 // 全部能放内存
  if (param->max_rows < num_rows / PQ_slowness) return true; // LIMIT 很小才值得
  return false;
}
```

PQ 模式下 `preallocate_records` 一次性分配 `LIMIT+1` 条空间，`read_all_rows` 里用 `Bounded_queue::push` 维护 top-k。非 PQ 路径另有 `prefilter_nth_element`（`filesort_utils.cc:147`）：`LIMIT < 总行数/2` 时先 `std::nth_element` 把前 k 小排前面，复杂度从 `O(n log n)` 降到 `O(n + k log k)`。

---

## 二、临时表（temptable）

### 2.1 目录与核心类

引擎在 `storage/temptable/`。核心类：

| 类 | 位置 | 说明 |
|----|------|------|
| `Table` | `table.h:47` | 临时表主体，持有行存储 + 索引 |
| `Handler` | `handler.h:143` | `::handler` 子类 |
| `Row` | `row.h:402` | 一行，两种状态（轻量/自有） |
| `Cell` | `cell.h:42` | 一个字段的轻量描述（不拷贝数据） |
| `Tree` / `Hash_unique` / `Hash_duplicates` | `index.h:158/186/213` | 三种索引实现 |

### 2.2 内存 vs 磁盘：两级阈值

**每表阈值** `tmp_table_size`（`sys_vars.cc:5297`，**默认 16MB**）：`TableResourceMonitor`（`table.h:130`）超限返回 `HA_ERR_RECORD_FILE_FULL`。

**全局阈值** `temptable_max_ram`（`sys_vars.cc:5360`，**默认 1GiB**）：分配策略 `Prefer_RAM_over_MMAP_policy`（`allocator.h:210`）——RAM 未超用 RAM，超了转 mmap，mmap 超 `temptable_max_mmap`（1GiB）则分配失败。

**超限转磁盘**：temptable 引擎**不实现自己的磁盘格式**，返回 `HA_ERR_RECORD_FILE_FULL`，由 SQL 层转成 **InnoDB 磁盘临时表**。转换函数 `create_ondisk_from_heap()`（`sql_tmp_table.cc:2577`）：换 `innodb_hton` handler → 逐行迁移 → 删内存 temptable。

### 2.3 行存储 Row / Cell

`Cell`（`cell.h:42`）是轻量解释器，只持有 `is_null / data_length / data*` 指针，指向 Row 内部数据区。

`Row`（`row.h:402`）两种状态：

- **轻量**：`m_ptr` 指向 MySQL `write_row()` 传入的 record buffer，不拷贝；
- **自有**：`copy_to_own_memory()`（`src/row.cc:62`）后布局为 `[buf_len][Cell 数组][用户数据]`，一次分配整行。

`Handler::create()`（`src/handler.cc:118`）先判断"是否所有列定长"：定长则 `m_rows.element_size(rec_buff_length)`（行就是连续字节），变长（VARCHAR/BLOB/JSON/GEOMETRY）则存 `Row` 对象。**这是 temptable 快的关键**——定长行零拷贝。

### 2.4 索引结构：C++ 标准容器，非 B+ 树

temptable 用 `std::set`/`unordered_set`，而非磁盘 B+ 树：

```cpp
// src/table.cc:267
switch (mysql_index.algorithm) {
  case HA_KEY_ALG_BTREE: append_new_index<Tree>(...); break;
  case HA_KEY_ALG_HASH:
    if (mysql_index.flags & HA_NOSAME) append_new_index<Hash_unique>(...);
    else append_new_index<Hash_duplicates>(...);
}
```

默认算法是 hash（`handler.cc:837`）。`Hash_unique::insert` 重复返回 `FOUND_DUPP_KEY`；`Tree::insert` 用 `lower_bound` 判重。**temptable 的 rowid 是行存储元素的指针**（`ref_length = sizeof(Storage::Element*)`）。

### 2.5 创建路径

- `create_tmp_table()`（`sql_tmp_table.cc:879`）：内部临时表（聚合/去重/物化）。
- `create_tmp_table_from_fields()`（`:1909`）：给定字段列表直接建表（UNION/derived 物化）。
- `setup_tmp_table_handler()` + `instantiate_tmp_table()`（`:2361`）：真正 `handler->create()` + `open_tmp_table()`。

### 2.6 GROUP BY / DISTINCT / ORDER BY 物化：OT 类型

`make_tmp_tables_info()`（`sql_select.cc:4354`）决定建哪种临时表。`QEP_TAB::enum_op_type`（`sql_executor.h:410`）区分：

| op_type | 语义 | 迭代器 |
|---------|------|--------|
| `OT_MATERIALIZE` | 写入临时表，不聚合，再扫描 | `MaterializeIterator` |
| `OT_AGGREGATE_THEN_MATERIALIZE` | 先流式聚合，再物化结果 | `AggregateIterator` + 物化 |
| `OT_AGGREGATE_INTO_TMP_TABLE` | 直接在临时表上分组聚合 | `TemptableAggregateIterator` |

### 2.7 物化后扫描

**`TemptableAggregateIterator::Init`（`composite_iterators.cc:1687`）** 的核心是"读源行 → 查重 → 更新聚合或插新行"：

```cpp
for (;;) {
  int read_error = m_subquery_iterator->Read();     // 读源行
  ...
  group_found = ...ha_index_read_map(...HA_READ_KEY_EXACT);  // 查 group
  if (group_found) {
    update_tmptable_sum_func(...);                  // 更新聚合函数
    int error = table()->file->ha_update_row(...);
    if (error != 0 && error != HA_ERR_RECORD_IS_THE_SAME)
      if (move_table_to_disk(error, false)) return true;  // 满 → InnoDB
  } else {
    int error = table()->file->ha_write_row(...);   // 新 group 行
    if (error != 0)
      if (move_table_to_disk(error, true)) return true;
  }
}
```

`Read()` 直接代理给 `m_table_iterator`（扫描结果临时表）。

**`MaterializeIterator::Init`（`composite_iterators.cc:824`）**：`rematerialize=false` 且已物化时只重扫；否则 `instantiate_tmp_table` 建表 → 逐个 query block `MaterializeQueryBlock` 物化 → 之后扫描。

---

## 三、深潜补充：sort key 字节布局与临时表三级转换

### 3.1 make_sortkey 的真实字节布局（`filesort.cc:1392`）

**varlen 字段**：先 `to += size_of_varlength_field`（4 字节，**跳过长度前缀**）（`:1401-1404`），末尾回填总长 `store_varlen_key_length(orig_to, to - orig_to)`（`:1466-1470`）。前缀存的是 `actual_length + VARLEN_PREFIX`——**含自身 4 字节**（`:1445`）。

**NULL 标志是三值，不是二值**（`sort_param.h:294-295`）：

| 值 | 含义 |
|----|------|
| `0x00` | 该字段为 NULL |
| `0xff` | NULL **且** DESC（`:1437-1439`） |
| `0x01` | 非 NULL |

> **关键细节：DESC 只反转 key 数据本体**（`while (actual_length--) *to = (uchar)(~*to);`，`:1450-1454`）——**不反转 null 字节、不反转 varlen 长度前缀**。这是为了保持"长度前缀仍可读"，否则归并时无法解析记录边界。

其他：JSON 追加 8 字节 hash（`int8store`，`:1460-1464`）；**null bits 顺序是"表的 null-row 位在前，字段 null 位在后"**（`:1494-1502`）。

**addon fields 的两种布局**：
- **packed**（`:1515`）：`to = field->pack(to, ...)`，**上限是剩余 buffer**；NULL 字段**零字节**（只置 null bit，`:1512-1513`）；末尾 `store_addon_length`
- **非 packed**（`:1533`）：`to += addonf.max_length` 定长推进

### 3.2 写不下的重试协议（文档之前完全没提）

`alloc_and_make_sortkey`（`filesort.cc:837-857`）：`make_sortkey` 返回 **`UINT_MAX` = 溢出**；外层循环把 `min_bytes = sort_key_buf.size() + 1` 后重试，成功后才 `commit_used_memory(rec_sz)`（`:852`）。`longest_addon_so_far` 在 `:1536` 更新，用于 `max_record_length` 估算——**sort buffer 的记录长度是"探测式"增长的，不是预先算准的**。

### 3.3 packed addon 的启用条件（`<14` 字节就不 pack）

`try_to_pack_addons`（`filesort.cc:264-282`）：`m_packable_length < 10 + 4`（即 **<14 字节**）→ `Addon_fields_status::skip_heuristic` 不 pack。packable 只对 nullable 或 STRING/VARCHAR/BLOB 类字段累加（`:2299-2304`）。

**PQ（优先队列）模式禁用 pack**（`filesort.cc:452-454` 注释）——因为 PQ 要覆写堆顶，变长记录会破坏结构。

### 3.4 sort_buffer 是二维决策（不是一维）

`filesort_utils.cc:131`：除 `num_input_rows <= 100`（`:184`）外，**每个分支内部还按 `key_len < 10`** 选 `Mem_compare` vs `Mem_compare_longkey`（`:185`/`:219`/`:232`）。

`prefilter_nth_element` 的条件（`:147-148`）：`max_output_rows < num_input_rows/2 && !m_remove_duplicates`——**去重与 nth_element 互斥**（这点容易漏）。

### 3.5 归并的两个巧思

- `merge_many_buff.h:75`：`i < num_chunks - MERGEBUFF*3/2`——保证**最后一组至少 10 个 chunk**，避免产生极小的 chunk（合并开销 > 收益）
- **只剩 1 个 chunk 时退出堆**，让它独占整个 sort buffer 顺序拷贝（`:2023-2056`）——堆归并在单 chunk 时纯属浪费

### 3.6 temptable vs MEMORY：为什么要换（源码依据）

**最直接的动因**：`use_tmp_disk_storage_engine`（`sql_tmp_table.cc:2037`）——`MEMORY do not support BLOBs`，有 `share->blob_fields` 就**直接上 InnoDB 磁盘表**（`:2058-2062`）。

本质区别：MEMORY 是**定长行**（VARCHAR 按最大长度分配，一行 `VARCHAR(1000)` 就占 1000 字节），temptable 支持**变长行**（BLOB/JSON/GEOMETRY 都能放）。所以 8.0 把默认引擎换成 temptable（`internal_tmp_mem_storage_engine` 默认 `TMP_TABLE_TEMPTABLE`，`sys_vars.cc:5352-5358`；I_S 表强制 MEMORY）。

### 3.7 三级转换的真实逻辑（含一处纠错）

> ⚠️ **纠错**：`max_heap_table_size` **只对 MEMORY 引擎生效，对 temptable 完全不生效**！

| 级别 | 机制 | 位置 |
|------|------|------|
| **每表** | `share->max_rows = (db_type()==heap_hton ? min(tmp_table_size, max_heap_table_size) : tmp_table_size) / reclength` | `sql_tmp_table.cc:2206-2215` |
| temptable 每表 | `Prefer_RAM_over_MMAP_policy_obeying_per_table_limit`（`allocator.h:266-283`）在**块粒度**判超限 → `throw Result::RECORD_FILE_FULL` | `allocator.h:273-275` |
| **全局** | `Prefer_RAM_over_MMAP_policy`（`allocator.h:218-247`）：RAM 未达 `temptable_max_ram`(1GiB) 用 RAM；超了转 MMAP；再超 `temptable_max_mmap` 就 `RECORD_FILE_FULL` | `allocator.h:237` |
| **落盘** | 两条路径：①**建表时**就转 `create_tmp_table_with_fallback`（`sql_tmp_table.cc:2264-2309`，捕获 `HA_ERR_RECORD_FILE_FULL` 换 `innodb_hton` 重建）；②**插入途中**转 `create_ondisk_from_heap`（`:2577`） | — |

> 注：`create_ondisk_from_heap` 对 `db_type() != temptable_hton` 直接报错（`:2593-2604`）——它只处理 temptable→InnoDB 的转换。

## 核心调用栈

```
【排序】SortingIterator::DoSort()                 sorting_iterator.cc:524
         └─ filesort()                            filesort.cc:367
             ├─ read_all_rows()                   :924  → make_sortkey() 构造 key
             ├─ write_keys()                      :1089 → sort_buffer() + 落盘 chunk
             ├─ save_index()                      :1588（内存够时）
             ├─ merge_many_buff()                 merge_many_buff.h:50
             └─ merge_index()                     :2067
             └─ Filesort_buffer::sort_buffer()    filesort_utils.cc:131（std::sort/stable_sort/nth_element）

【临时表】TemptableAggregateIterator::Init()       composite_iterators.cc:1687
           └─ instantiate_tmp_table()             sql_tmp_table.cc:2361
               └─ create_ondisk_from_heap()       :2577（满时转 InnoDB）
```

---


## 参考

**论文**
- **Graefe《Query Evaluation Techniques for Large Databases》(1993)** —— 外部排序（external merge sort）、sort-merge join 的经典综述

**官方文档**
- *MySQL 8.0 Reference Manual → ORDER BY Optimization*
- *MySQL 8.0 Reference Manual → Internal Temporary Table Use in MySQL*（`tmp_table_size` / `max_heap_table_size`、内存转磁盘）

**内核月报**
- **2021/06《Order By 优化逻辑代码分析》** —— filesort 的 addon / priority queue / 多轮归并（执行阶段部分）

