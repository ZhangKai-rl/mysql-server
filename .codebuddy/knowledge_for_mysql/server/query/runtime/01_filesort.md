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

---

## 演进、性能与坑

### 8.0 的演进

| 变化 | 说明 |
|---|---|
| **块式指数增长** | `sort_buffer_size` 从"一次性 malloc"变成"预算 + 32KB 起 ×1.5 增长"，小排序只付 32KB |
| **`max_length_for_sort_data` 废弃** | 不再按行宽在 addon/rowid 间切换，改为默认 addon |
| **改用 C++ 标准库** | 5.x 的手写 `filesort_compare` → 8.0 的 `std::sort` / `std::stable_sort` + 三种比较器 |
| **addon 默认化** | 注释："Generally, prefer using addon fields if we can" |

### 性能特征

- **代价模型**：优化器按 `n + k×log2(k)` 建模（对应 nth_element 预筛的复杂度）
- **大结果集慢的三个来源**：落盘写 chunk、多路归并的 I/O、rowid 模式的随机回表
- **`Sort_merge_passes` 状态变量**：每次 `merge_buffers()` 调用 +1，是判断"归并了几轮"的直接指标

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

---

## 相关的系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `sort_buffer_size` | **256 KB**（`DEFAULT_SORT_MEMORY`） | SESSION | 排序内存**预算**（非预分配）；范围 `MIN_SORT_MEMORY`(32KB) ~ ULONG_MAX |
| `max_sort_length` | 1024（待核实） | SESSION | `PAD SPACE` collation 下字符串 sort key 的截断长度 |
| `max_length_for_sort_data` | **已废弃**（`Unused`） | — | 8.0.20 起不再用于在 addon/rowid 间切换 |

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
