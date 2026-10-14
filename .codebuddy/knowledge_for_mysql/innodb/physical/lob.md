# InnoDB LOB 深度解析

> 基于 MySQL 8.0.39 源码。LOB（Large Object）是 InnoDB 8.0 重写的大对象存储子系统，是 JSON 部分更新能落地的物理基础。本文独立于 [`server/datatype/json.md`](../../server/datatype/json.md)，那边只保留"JSON 当 BLOB 存"的结论。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [一、物理结构：三类页 + 一个 ref](#一物理结构三类页--一个-ref)
    - [二、版本机制：lob_version 与 COW](#二版本机制lob_version-与-cow)
  - 写路径
    - [三、写入路径](#三写入路径)
  - 读路径
    - [四、读路径：版本化 MVCC 读](#四读路径版本化-mvcc-读)
  - 生命周期
    - [五、purge 回收](#五purge-回收)
    - [六、回滚](#六回滚)
    - [七、崩溃恢复](#七崩溃恢复)
- [核心调用栈](#核心调用栈)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

LOB 是 InnoDB 存储**外部列（off-page BLOB/TEXT/JSON/GEOMETRY）**的子系统。8.0 之前叫 "externally stored fields"，只有简单的页链表；8.0 重写为 **first page + data page + index entry + 版本链** 的结构，目的就是支持**部分更新**：改一个大对象的其中几十字节，不必重写整个对象。

### 用途

- 支撑超过页半满阈值的大列（约 8KB）的 off-page 存储；
- **支撑 JSON 部分更新**：server 层算出 `Binary_diff{offset,length}`，LOB 按偏移量改字节；
- 支撑大对象的 MVCC 多版本读（旧版本保留 + purge 回收）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0 之前 | 旧 extern 机制：BLOB 首页 + 溢出页链表，无 index、无版本 |
| 8.0.0 | LOB 重写：引入 first page / data page / node page / index entry |
| 8.0.3 | JSON 部分更新依赖 LOB 的部分写能力 |
| 8.0.12 | WL#11328：小改动（≤100 字节）用 undo 记 Lob diff 并原地改页，避免 COW 新页（官方称最高约 3 倍 TPS） |

---

## 理论基础

### 设计模式与算法

| 模式/算法 | 体现 |
|-----------|------|
| **写时复制（COW）** | 大改动时新分配数据页 + 新 index entry，旧页保留给 MVCC（`lob0first.cc:60`、`lob0pages.cc:59`） |
| **版本链（MVCC 的 "version vault"）** | 每个 index entry 带一条 `versions` 单向链，存自己的历史版本，读时按 `lob_version` 挑（`lob0index.h:81-107`） |
| **意图日志 + 延迟执行** | first page 的释放延迟到 purge batch 末尾（`trx0purge.cc:2561`），避免多 purge 线程竞争 |
| **B+树式的链表管理** | index list / free list / versions list 都用 InnoDB 通用的 `flst`（file list）基节点 + 6 字节 `fil_addr` |
| **可重入/幂等而非原子** | LOB 回滚每轮提交一个 mtr，用 `ref.set_length(0)` 做哨兵，源码里预留了 3 个 crash 注入点 |

### 类似实现对比

| 系统 | 做法 |
|------|------|
| PostgreSQL TOAST | 大字段切片成 chunk 存 TOAST 表，更新任何一部分都要**重写整个值 + 产生新的 toast 行**，无部分更新 |
| Oracle LOB | 有 SecureFile LOB 的 chunk 级版本化，与 InnoDB 8.0 LOB 思路接近 |
| MongoDB | 文档级重写（WiredTiger 的 delta update 是较新的能力） |
| InnoDB 8.0 LOB | **字节区间级部分写 + 页级 COW + index entry 版本链** |

---

## 核心实现

### ★ 外置的决策：什么时候外置、外置哪些列、行内留多少

下面几节讲"**外置之后怎么存**"；这一节先讲"**什么时候会外置**"——决策全在 `dtuple_convert_big_rec()`（`storage/innobase/data/data0data.cc:423`）：

```cpp
big_rec_t *dtuple_convert_big_rec(dict_index_t *index, upd_t *upd, dtuple_t *entry) {
  ...
  // ★ 只有聚簇索引才外置：二级索引记录本身就短，不需要这套机制
  if (!index->is_clustered()) {
    return nullptr;
  }

  if (!dict_table_has_atomic_blobs(index->table)) {
    /* up to MySQL 5.1: store a 768-byte prefix locally */
    local_len = BTR_EXTERN_FIELD_REF_SIZE + DICT_ANTELOPE_MAX_INDEX_COL_LEN;  // 20 + 768
  } else {
    /* new-format table: do not store any BLOB prefix locally */
    local_len = BTR_EXTERN_FIELD_REF_SIZE;                                    // ★ 只留 20
  }

  size = rec_get_converted_size(index, entry);
  ...
  /* Decide which fields to shorten: the algorithm is to look for
  a variable-length field that yields the biggest savings when
  stored externally */
```

**四个要点**：

1. **★ 只有聚簇索引会外置**——二级索引直接 `return nullptr`
2. **★ `COMPACT` / `REDUNDANT`（Antelope）与 `DYNAMIC` / `COMPRESSED`（Barracuda）的关键差异就在这里**：
   - 前者行内留 **768 字节前缀** + 20B 指针（`DICT_ANTELOPE_MAX_INDEX_COL_LEN = 768`）
   - 后者**完全外置**，行内只留 20B 指针
   ⇒ 这才是 `ROW_FORMAT` 影响大字段行为的实质
3. **外置哪些列是贪心选的**——源码注释直说 *"look for a variable-length field that yields the biggest savings"*：挑**外置后节省最多的变长字段**，逐个外置直到行能塞进页
4. **触发条件是"行放不进页"**：`rec_get_converted_size()` 算出转换后大小，超过页可用空间（约半页、8KB 量级）才启动外置

⇒ ★ **所以"off-page 就是 BLOB 吗"——不完全是**。BLOB / TEXT / JSON / GEOMETRY 是典型对象，但外置的判据是"**变长且够长**"：一个足够长的 `VARCHAR` 同样会被外置；**定长列永远不会外置**。8.0 之前这套机制叫 *externally stored fields*，8.0 重写后叫 LOB。

### 一、物理结构：三类页 + 一个 ref

### 1.1 三类页

**★ 先纠正一个极易搞混的点：有 4 个同名的 `LOB_PAGE_DATA`，值各不相同**，三者不能混用：

```cpp
first_page_t::LOB_PAGE_DATA = 96;    /* lob0first.h:76 —— first page 数据区起点 */
data_page_t::LOB_PAGE_DATA  = 49;    /* lob0pages.h:39 —— data page 数据区起点 */
node_page_t::LOB_PAGE_DATA  = 39;    /* lob0impl.h:498 —— node page 数据区起点 */
ZLOB_PAGE_DATA = FIL_PAGE_DATA = 38; /* lob0lob.h:151 —— 压缩 LOB（全局常量） */
```

> ⚠️ 另一个常见误解：`LOB_HDR_PART_LEN` / `LOB_HDR_NEXT_PAGE_NO` / `LOB_HDR_SIZE`（在 `lob0lob.h:141-148`）**不是** first page 的字段——它们是**旧式（Antelope、无 LOB index）BLOB 页**的 8 字节页头。新格式 first page 用的是下面这套 `OFFSET_*`。

**FIRST PAGE —— `lob0first.h:45-79` 的完整字段布局**（16 KiB 页为例）：

```cpp
struct first_page_t : public basic_page_t {
  /** Version information. One byte. */
  static const ulint OFFSET_VERSION = FIL_PAGE_DATA;          // 38, 1B，固定写 0
  /** One byte of flag bits.  Currently only one bit (the least significant
      bit) is used, other 7 bits are available for future use.*/
  static const ulint OFFSET_FLAGS = FIL_PAGE_DATA + 1;        // 39, 1B ★ bit0=1 ⇒ 此 LOB 不再可部分更新
  /** LOB version. 4 bytes.*/
  static const uint32_t OFFSET_LOB_VERSION = OFFSET_FLAGS + 1;          // 40, 4B
  /** The latest transaction that modified this LOB. */
  static const ulint OFFSET_LAST_TRX_ID = OFFSET_LOB_VERSION + 4;       // 44, 6B
  /** The latest transaction undo_no that modified this LOB. */
  static const ulint OFFSET_LAST_UNDO_NO = OFFSET_LAST_TRX_ID + 6;      // 50, 4B
  /** Length of data stored in this page.  4 bytes. */
  static const ulint OFFSET_DATA_LEN = OFFSET_LAST_UNDO_NO + 4;         // 54, 4B
  /** The trx that created the data stored in this page. */
  static const ulint OFFSET_TRX_ID = OFFSET_DATA_LEN + 4;               // 58, 6B
  /** The offset where the list base node is located.  This is the list
      of LOB pages. */
  static const ulint OFFSET_INDEX_LIST = OFFSET_TRX_ID + 6;             // 64, 16B flst_base_node_t
  /** The offset where the list base node is located.  This is the list
      of free nodes. */
  static const ulint OFFSET_INDEX_FREE_NODES =
      OFFSET_INDEX_LIST + FLST_BASE_NODE_SIZE;                          // 80, 16B flst_base_node_t
  /** The offset where the contents of the first page begins. */
  static const ulint LOB_PAGE_DATA =
      OFFSET_INDEX_FREE_NODES + FLST_BASE_NODE_SIZE;                    // 96
  static const ulint LOB_PAGE_TRAILER_LEN = FIL_PAGE_DATA_END;          // 8
};
```

- 偏移 64 的 `index_list` = **index entry 链表**（逻辑上的 LOB 数据页序列）
- 偏移 80 的 `free_list` = **空闲 index entry 链表**（复用槽位，避免反复分配）
- `LOB_PAGE_DATA`(96) 之后**先是 10 个 index entry（600 字节），再才是 LOB 数据**：

```cpp
  /** Get the number of index entries this page can hold. */
  constexpr static ulint node_count() {
    /* Each index entry is of size 60 bytes.  We store only 10
       index entries in the first page of the LOB. ... */
    return (10);
  }
  /** Obtain the location where the data begins. */
  byte *data_begin() const {
    constexpr ulint index_array_size = node_count() * index_entry_t::SIZE;  // 10*60=600
    return (frame() + LOB_PAGE_DATA + index_array_size);                    // = frame + 696
  }
  static ulint payload() {
    return (UNIV_PAGE_SIZE - LOB_PAGE_DATA - LOB_PAGE_TRAILER_LEN);         // 16384-96-8 = 16280
  }
```

⇒ LOB 数据在 first page 里的可用空间 = `payload() - 600` = **15680 字节**。

**分配时两条 flst 链表的初始化**（10 个 entry 先全部挂到 free_list）：

```cpp
buf_block_t *first_page_t::alloc(mtr_t *alloc_mtr, bool is_bulk) {
  m_block = alloc_lob_page(m_index, alloc_mtr, hint, is_bulk);
  set_page_type();              // FIL_PAGE_TYPE_LOB_FIRST
  set_version_0();
  set_data_len(0);
  set_trx_id(0);
  flst_init(index_lst, m_mtr);
  flst_init(free_lst, m_mtr);

  byte *cur = nodes_begin();
  for (ulint i = 0; i < nc; ++i) {
    index_entry_t entry(cur, m_mtr, m_index);
    entry.init();
    flst_add_last(free_lst, cur, m_mtr);
    cur += index_entry_t::SIZE;
  }
  set_next_page_null();
  return (m_block);
}
```

**DATA PAGE —— `lob0pages.h:34-48`，页头只有 11 字节**：

```cpp
struct data_page_t : public basic_page_t {
  static const ulint OFFSET_VERSION = FIL_PAGE_DATA;            // 38, 1B
  static const ulint OFFSET_DATA_LEN = OFFSET_VERSION + 1;      // 39, 4B
  static const ulint OFFSET_TRX_ID = OFFSET_DATA_LEN + 4;       // 43, 6B
  static const ulint LOB_PAGE_DATA = OFFSET_TRX_ID + 6;         // 49  ★ 注意是 49，不是 96
};
  static ulint payload() {
    return (UNIV_PAGE_SIZE - LOB_PAGE_DATA - FIL_PAGE_DATA_END);   // 16384-49-8 = 16327
  }
```

```cpp
ulint data_page_t::write(const byte *&data, ulint &len) {
  byte *ptr = data_begin();
  ulint written = (len > payload()) ? payload() : len;
  mlog_write_string(ptr, data, written, m_mtr);
  set_data_len(written);
  data += written;
  len -= written;
  return (written);
}
```

**NODE PAGE（= index page）—— `lob0impl.h:493-510`，页头只有 1 字节**：

```cpp
/** The node page (also can be called as the index page) contains a list of
index_entry_t objects. */
struct node_page_t : public basic_page_t {
  static const ulint OFFSET_VERSION = FIL_PAGE_DATA;      // 38, 1B
  static const ulint LOB_PAGE_DATA = OFFSET_VERSION + 1;  // 39  ★ 只有 1 字节头
  ...
  void set_page_type() {
    mlog_write_ulint(frame() + FIL_PAGE_TYPE, FIL_PAGE_TYPE_LOB_INDEX, MLOG_2BYTES, m_mtr);
  }
};
ulint node_page_t::node_count() {
  return (max_space_available() / index_entry_t::SIZE);   // 16337 / 60 = 272
}
```

**index entry 只有 60 字节，落在 first page（内联 10 个）或 node page 上——实证**：

```cpp
flst_node_t *first_page_t::addr2ptr_x(const fil_addr_t &addr, mtr_t *mtr) const {
  flst_node_t *result = fut_get_ptr(space, page_size, addr, RW_X_LATCH, mtr, &block);
  const page_type_t type = block->get_page_type();
  ut_a(type == FIL_PAGE_TYPE_LOB_FIRST || type == FIL_PAGE_TYPE_LOB_INDEX);
  return result;
}
```

> ★ **别把 `node_page_t` 归到 `lob0index.h`**——它在 `lob0impl.h`。`lob0index.h` 里只有 `index_entry_mem_t` 与 `index_entry_t`。

**`index_entry_t` 的 60 字节逐字段布局**（`lob0index.h:80-107`）：

```cpp
struct index_entry_t {
  static const ulint OFFSET_PREV = 0;                                    // 0..5   6B flst 前驱
  static const ulint OFFSET_NEXT = OFFSET_PREV + FIL_ADDR_SIZE;          // 6..11  6B flst 后继
  /** Points to base node of the list of versions. The size of base node is 16 bytes. */
  static const ulint OFFSET_VERSIONS = OFFSET_NEXT + FIL_ADDR_SIZE;      // 12..27 16B ★ 版本链 base node
  /** The creator trx id. */
  static const ulint OFFSET_TRXID = OFFSET_VERSIONS + FLST_BASE_NODE_SIZE;        // 28..33 6B
  /** The modifier trx id. */
  static const ulint OFFSET_TRXID_MODIFIER = OFFSET_TRXID + 6;                    // 34..39 6B
  static const ulint OFFSET_TRX_UNDO_NO = OFFSET_TRXID_MODIFIER + 6;              // 40..43 4B
  /** The undo number of the modifier trx. */
  static const ulint OFFSET_TRX_UNDO_NO_MODIFIER = OFFSET_TRX_UNDO_NO + 4;        // 44..47 4B
  static const ulint OFFSET_PAGE_NO = OFFSET_TRX_UNDO_NO_MODIFIER + 4;            // 48..51 4B ★ 指向 first/data page
  static const ulint OFFSET_DATA_LEN = OFFSET_PAGE_NO + 4;                        // 52..55 4B（实际只写 2 字节）
  /** The LOB version number. */
  static const ulint OFFSET_LOB_VERSION = OFFSET_DATA_LEN + 4;                    // 56..59 4B ★ 该 entry 自己的版本
  /** Total length of an index node. */
  static const ulint SIZE = OFFSET_LOB_VERSION + 4;                               // = 60
};
```

> ⚠️ `set_data_len()` 写的是 **`MLOG_2BYTES`**（只占 2 字节），尽管偏移槽留了 4 字节。

**版本链的三个操作**（注意真名——**没有** `insert_version` / `get_version`）：

```cpp
  void add_version(index_entry_t &entry) const {
    flst_node_t *node = entry.get_node_ptr();
    flst_base_node_t *version_list = get_versions_ptr();
    flst_add_first(version_list, node, m_mtr);      /* 旧 entry 插到 versions 链头 */
  }

  /* The given entry becomes the old version of the current entry.
     Move the version base node from old entry to current entry. */
  void set_old_version(index_entry_t &entry) {
    flst_base_node_t *version_list = get_versions_ptr();
    ut_ad(flst_get_len(version_list) == 0);
    entry.move_version_base_node(*this);       /* ① 先搬旧 entry 原有的"更旧版本链" */
    flst_add_first(version_list, node, m_mtr); /* ② 再把旧 entry 本身挂到链头 */
  }
```

```cpp
void index_entry_t::move_version_base_node(index_entry_t &to_entry) {
  flst_base_node_t *from_node = get_versions_list();
  flst_base_node_t *to_node = to_entry.get_versions_list();
  mlog_write_string(to_node, from_node, FLST_BASE_NODE_SIZE, m_mtr);
  ut_ad(flst_get_len(from_node) == flst_get_len(to_node));
  flst_init(from_node, m_mtr);
}
```

**entry 指向的页可能是 first page 也可能是 data page**（`purge()` 里显式区分，first page 永不在此释放）：

```cpp
/** Purge the current index entry. An index entry points to either a FIRST
    page or DATA page.  That LOB page will be freed if it is DATA page.  A FIRST
    page should not be freed. */
void index_entry_t::purge(dict_index_t *index) {
  page_type_t type = fil_page_get_type(block->frame);
  if (type != FIL_PAGE_TYPE_LOB_FIRST) {
    data_page_t data_page(block, m_mtr, index);
    data_page.dealloc();
  }
  ...
}
```

| 页类型 | 常量 | 定义 | 职责 |
|--------|------|------|------|
| `FIL_PAGE_TYPE_LOB_FIRST` (24) | `fil0fil.h:1292` | `first_page_t`（`include/lob0first.h:43`） | **既是第 1 个数据页，又是控制页**：存全局 `lob_version`、index entry 链表头、空闲链表头、`last_trx_id/undo_no` |
| `FIL_PAGE_TYPE_LOB_DATA` (23) | `fil0fil.h:1289` | `data_page_t`（`include/lob0pages.h:35`） | 只装用户数据，无链表无版本 |
| `FIL_PAGE_TYPE_LOB_INDEX` (22) | `fil0fil.h:1286` | `node_page_t`（`include/lob0impl.h:495`） | index entry 的容器（first page 固定放 10 个，不够再分配 node page） |

共同基类 `basic_page_t`（`include/lob0util.h:42`）。

#### ★ 那"一个 ref"：20 字节 BLOB 引用的逐字节布局

> ⚠️ **两个常见错误**：① 以为偏移 12 处就是 4 字节长度（错——12 号字节是**标志字节**）；② 以为 `BTR_EXTERN_OFFSET` 与版本号是两个字段（错——**同一个位置**）。

```cpp
/* storage/innobase/include/lob0lob.h:102-136 */
/** Space identifier where stored. */
const ulint BTR_EXTERN_SPACE_ID = 0;      /* 字节 0..3   4B space_id */
/** page number where stored */
const ulint BTR_EXTERN_PAGE_NO = 4;       /* 字节 4..7   4B = LOB first page 页号 */
/** offset of BLOB header on that page */
const ulint BTR_EXTERN_OFFSET = 8;        /* 字节 8..11  旧式：BLOB 头在页内偏移 */
/** Version number of LOB (LOB in new format)*/
const ulint BTR_EXTERN_VERSION = BTR_EXTERN_OFFSET;  /* ★ 同位置！新格式这里存 lob_version */
/** 8 bytes containing the length of the externally stored part of the LOB.
    The 2 highest bits are reserved to the flags below. */
const ulint BTR_EXTERN_LEN = 12;          /* 字节 12..19，但 12 号字节是标志字节 */

const ulint BTR_EXTERN_OWNER_FLAG = 128UL;           /* bit7：不是 owner ⇒ 无权释放 */
const ulint BTR_EXTERN_INHERITED_FLAG = 64UL;        /* bit6：继承自旧版本 ⇒ 回滚不得释放 */
const ulint BTR_EXTERN_BEING_MODIFIED_FLAG = 32UL;   /* bit5：正在被修改 */
```

完整 20 字节：

| 字节 | 含义 |
|---|---|
| 0–3 | `space_id` |
| 4–7 | `page_no`（LOB **first page** 页号） |
| 8–11 | 旧式 = BLOB 头页内偏移；**新格式 = `lob_version`** |
| 12 | 标志字节：`owner(128)` / `inherited(64)` / `being_modified(32)` |
| 13–15 | 保留 |
| **16–19** | **真正的 32 位长度** |

代码侧的印证：

```cpp
  /** Read the length from the blob reference. */
  ulint length() const {
    return (mach_read_from_4(m_ref + BTR_EXTERN_LEN + 4));    /* ★ +4，即字节 16 */
  }
  void set_length(const ulint len, mtr_t *mtr) {
    ut_ad(len <= MAX_SIZE);
    mlog_write_ulint(m_ref + BTR_EXTERN_LEN + 4, len, MLOG_4BYTES, mtr);
  }
  bool is_owner() const {
    ulint byte_val = mach_read_from_1(m_ref + BTR_EXTERN_LEN);
    return (!(byte_val & BTR_EXTERN_OWNER_FLAG));
  }
```

**ref 固定位于该列"本地存储部分"的末尾 20 字节**：

```cpp
ulint btr_rec_get_field_ref_offs(const dict_index_t *index, const ulint *offsets,
                                 ulint n) {
  ut_a(rec_offs_nth_extern(index, offsets, n));       /* 必须是外置列 */
  field_ref_offs = rec_get_nth_field_offs(index, offsets, n, &local_len);
  ut_a(local_len >= BTR_EXTERN_FIELD_REF_SIZE);
  return (field_ref_offs + local_len - BTR_EXTERN_FIELD_REF_SIZE);   /* ★ 尾部 20 字节 */
}
```

外置时把列体"截断 + 尾部清零"变成 ref 占位（`data0data.cc`）：

```cpp
    local_prefix_len = local_len - BTR_EXTERN_FIELD_REF_SIZE;
    big_rec_field_t big_rec(longest_i,
                            dfield_get_len(dfield) - local_prefix_len,
                            (char *)dfield_get_data(dfield) + local_prefix_len);

    data = static_cast<byte *>(mem_heap_alloc(heap, local_len));
    memcpy(data, dfield_get_data(dfield), local_len);
    /* Clear the extern field reference (BLOB pointer). */
    memset(data + local_prefix_len, 0, BTR_EXTERN_FIELD_REF_SIZE);
    ...
    dfield_set_data(dfield, data, local_len);
    dfield_set_ext(dfield);
```

真正填值在插入路径（`lob0ins.cc`）：

```cpp
  blobref.set_length(field.len - m_remaining, mtr);                    /* 字节 16..19 */
  blobref.update(m_ctx->space(), m_cur_blob_page_no, FIL_PAGE_DATA, mtr); /* 0..3 / 4..7 / 8..11 */
```

> ⚠️ 对比 **ZBLOB**：`zlob0ins.cc:68` 写的是 `blobref.update(space, page_no, FIL_PAGE_NEXT, nullptr)`——**ref.offset 的语义不同**（见 1.1b）。

### 1.1b ZBLOB：LOB 页的 zlib 压缩形态（第四类存储）

> 全景地图见 [`../feat/compression.md`](../../feat/compression.md)。以上三类是**未压缩**形态；另有 zlib 压缩形态 `FIL_PAGE_TYPE_ZBLOB(11)`/`ZBLOB2(12)`/`FIL_PAGE_SDI_ZBLOB(19)`（`fil0fil.h:1248-1277`）——单条 LOB 压成一段**跨页连续的 zlib 流**。

**触发阈值 `Z_CHUNK_SIZE = 128K`**（`lob0lob.h:86-91`）：`ref_t::use_single_z_stream(lob_len)` = `length() <= 128K`。调用点 `btr_store_big_rec_extern_fields`（`lob0lob.cc:514-525`）：≤128K 走 `zInserter` 单流；更大走 `lob::z_insert` 分块格式（每 128K 一个独立流 + LOB 索引，`ZLOB_FIRST` 首帧）。该单流分支只在 `page_zip != nullptr`（COMPRESSED 行格式）或 SDI 索引时进入。

**写入/读取的完整实现**（`zInserter` 逐页 `deflate(Z_FINISH)`、`zReader` 跨页持续 inflate 的完整贴码）→ [`../../feat/compression.md`](../../feat/compression.md) LOB 页压缩节。LOB 视角只保留三点：

1. **入口分派**：`zlob0read.cc:85-92` 按首帧类型分派——`ZBLOB`/`SDI_ZBLOB` → `zReader`（单流），`ZLOB_FIRST` → 分块格式；
2. **页布局对比**：ZBLOB 页与普通 BLOB 页差异只在 `FIL_PAGE_TYPE`（11/12/19 vs 10/18）与"payload 是压缩字节流"（无页内目录/分片元数据，长度、首页码都在行内 ref 里）；
3. **读的约束**：单流必须**从头顺序解压**（`z_stream` 是 `zReader` 成员跨页持续，`Z_STREAM_END` 出现在非末页即 `ut_error`）；单流 LOB **不支持 partial update**（`lob0lob.cc:1212-1220`）。

**页布局对比**：ZBLOB 页从 `FIL_PAGE_DATA(38)` 铺满压缩字节流到页尾（剩余清零保证 redo 可复现）、**无页内目录/分片元数据**（长度、首页码都在行内 ref 里）；与普通 BLOB 页的差异只在 `FIL_PAGE_TYPE` 与"payload 是压缩流"。

**限制**：单流 LOB **不支持 partial update**（`lob0lob.cc:1212-1220`，`use_single_z_stream()` 直接禁用 `lob::z_update`）；删除/purge 按页类型分派（`lob0purge.cc:482-489` 含 ZBLOB 族）。

### 1.2 first page 头部

```cpp
// include/lob0first.h:43-79
struct first_page_t : public basic_page_t {
  static const ulint OFFSET_VERSION        = FIL_PAGE_DATA;              // 1B 格式版本
  static const ulint OFFSET_FLAGS          = FIL_PAGE_DATA + 1;          // 1B bit0 = 不可部分更新
  static const uint32_t OFFSET_LOB_VERSION = OFFSET_FLAGS + 1;           // 4B ★LOB 版本号
  static const ulint OFFSET_LAST_TRX_ID    = OFFSET_LOB_VERSION + 4;     // 6B 最后修改者 trx
  static const ulint OFFSET_LAST_UNDO_NO   = OFFSET_LAST_TRX_ID + 6;     // 4B 最后修改者 undo_no
  static const ulint OFFSET_DATA_LEN       = OFFSET_LAST_UNDO_NO + 4;    // 4B 本页数据长度
  static const ulint OFFSET_TRX_ID         = OFFSET_DATA_LEN + 4;        // 6B 创建者 trx
  static const ulint OFFSET_INDEX_LIST     = OFFSET_TRX_ID + 6;          // index entry 链表 base node
  static const ulint OFFSET_INDEX_FREE_NODES = ...;                      // 空闲 entry 链表
  static const ulint LOB_PAGE_DATA         = OFFSET_INDEX_FREE_NODES + FLST_BASE_NODE_SIZE;
```

first page 固定只放 **10 个** index entry（`node_count()`，`lob0first.h:431`），数据区在其后。

### 1.3 index_entry_t —— 逻辑偏移 → (页号, 页内偏移)

```cpp
// include/lob0index.h:81-107
struct index_entry_t {
  static const ulint OFFSET_PREV       = 0;                       // 6B
  static const ulint OFFSET_NEXT       = OFFSET_PREV + FIL_ADDR_SIZE;
  static const ulint OFFSET_VERSIONS   = OFFSET_NEXT + FIL_ADDR_SIZE;  // ★16B 旧版本链 base node
  static const ulint OFFSET_TRXID      = ...;                     // 6B 创建者
  static const ulint OFFSET_TRXID_MODIFIER = ...;                 // 6B 修改者
  static const ulint OFFSET_TRX_UNDO_NO = ...;                    // 4B
  static const ulint OFFSET_TRX_UNDO_NO_MODIFIER = ...;           // 4B
  static const ulint OFFSET_PAGE_NO    = ...;                     // ★4B 数据页号
  static const ulint OFFSET_DATA_LEN   = OFFSET_PAGE_NO + 4;      // ★2B 本 entry 覆盖字节数
  static const ulint OFFSET_LOB_VERSION = OFFSET_DATA_LEN + 4;    // ★4B LOB 版本
  static const ulint SIZE = OFFSET_LOB_VERSION + 4;               // = 60 字节
```

**映射算法**：entry 在 index list 中按 LOB 逻辑顺序排列，每个 entry 记自己覆盖的 `data_len`。查 offset 就是顺序累加跳过（`lob0update.cc:236-252` `find_offset`）：

```
第 k 个 entry 负责逻辑区间 [Σlen(0..k-1), Σlen(0..k))，页内偏移 = offset − Σlen(0..k-1)
```

即 **O(n) 顺序查找**（n = entry 数），不是 O(1)。

### 1.4 ref_t —— 行内 20 字节指针

```cpp
// include/lob0lob.h:102-115
const ulint BTR_EXTERN_SPACE_ID = 0;   // 4B
const ulint BTR_EXTERN_PAGE_NO  = 4;   // 4B
const ulint BTR_EXTERN_OFFSET   = 8;   // 4B
const ulint BTR_EXTERN_VERSION  = BTR_EXTERN_OFFSET;   // ★ 复用同一个 4 字节存 lob_version！
const ulint BTR_EXTERN_LEN      = 12;  // 8B（高 2 位 flag，低 4B+4B 长度）
```

`ref_t::SIZE = 20`（`BTR_EXTERN_FIELD_REF_SIZE`，`btr0types.h:66`）。

标志位：`BTR_EXTERN_OWNER_FLAG=128`（不置位才允许 purge 释放）、`INHERITED_FLAG=64`（rollback 时不得释放）、`BEING_MODIFIED_FLAG=32`（READ UNCOMMITTED 保护），见 `lob0lob.h:123/130/136`。

> **关键技巧**：`BTR_EXTERN_OFFSET` 与 `BTR_EXTERN_VERSION` 是同一个字段。行记录里存的是"我这个行版本对应的 LOB 版本号"，读的时候拿它去 index entry 的 versions 链里挑可见版本。

---

### 二、版本机制：lob_version 与 COW

**★ 那个"100 字节"阈值的唯一定义处**（`lob0lob.h:200-210`）：

```cpp
struct ref_t {
 private:
  /** If the LOB size is equal to or above this limit (in physical page
      size terms), then the LOB is big enough to be partially updated.  Only
      in this case LOB index needs to be built. */
  static const ulint LOB_BIG_THRESHOLD_SIZE = 2;   /* ★ 另一个阈值：≥2 个物理页才建 LOB index */

 public:
  /** If the total number of bytes modified in an LOB, in an update
      operation, is less than or equal to this threshold LOB_SMALL_CHANGE_THRESHOLD,
      then it is considered as a small change.  For small changes to LOB,
      the changes are undo logged like any other update operation. */
  static const ulint LOB_SMALL_CHANGE_THRESHOLD = 100;
};
```

> ⚠️ 源码里**没有** "WL#11328" 字样（grep `11328` 于 `storage/innobase` 命中 0）；该模块的自我描述是 `lob0undo.h` 的文件注释 *"Undo logging small changes to BLOBs."*。写文档时别把 WL 号当源码依据。

**判定的那一刻**（`lob0update.cc:88-163`）——小改动与大改动是两条完全不同的路：

```cpp
dberr_t update(InsertContext &ctx, trx_t *trx, dict_index_t *index,
               const upd_t *upd, ulint field_no, ref_t blobref) {
  mtr_t *mtr = ctx.get_mtr();                    /* ★ 复用调用方的 btr_mtr（写是原子的） */
  const Binary_diff_vector *bdiff_vector = upd->get_binary_diff_by_field_no(field_no);
  const ulint bytes_changed = upd_t::get_total_modified_bytes(*bdiff_vector);

  /* Whether the update to the LOB can be considered as a small change. */
  const bool small_change = (bytes_changed <= ref_t::LOB_SMALL_CHANGE_THRESHOLD);

  first_page_t first_page(mtr, index);
  first_page.load_x(first_page_id, page_size);
  first_page.set_last_trx_id(trx->id);
  first_page.set_last_undo_no(undo_no);
  uint32_t lob_version = 0;

  if (small_change) {
    lob_version = first_page.get_lob_version();    /* ★ 版本号不变 —— 不产生新版本 */
  } else {
    lob_version = first_page.incr_lob_version();   /* ★ 版本号 +1 */
  }

  for (Binary_diff_vector::const_iterator iter = bdiff_vector->begin();
       iter != bdiff_vector->end(); ++iter, ++count) {
    if (small_change) {
      err = replace_inline(ctx, trx, index, blobref, first_page,
                           bdiff->offset(), bdiff->length(),
                           (byte *)bdiff->new_data(uf->mysql_field));
      /* ★ 原地覆盖写页内字节：不新分配页、不动 index entry */
    } else {
      err = replace(ctx, trx, index, blobref, first_page, bdiff->offset(),
                    bdiff->length(), (byte *)bdiff->new_data(uf->mysql_field),
                    count);
      /* ★ COW：新分配数据页 + 新 index entry + 版本链 */
    }
  }

  blobref.set_offset(lob_version, mtr);            /* ★ 把版本写回记录里 ref 的字节 8..11 */
  return err;
}
```

**小改动：原地覆盖（`replace_inline`），只改字节、不换页**：

```cpp
void first_page_t::replace_inline(ulint offset, const byte *&ptr, ulint &want,
                                  mtr_t *mtr) {
  byte *old_ptr = data_begin();
  old_ptr += offset;
  ulint data_avail = get_data_len() - offset;
  ulint data_to_copy = (want > data_avail) ? data_avail : want;
  mlog_write_string(old_ptr, ptr, data_to_copy, mtr);
  ptr += data_to_copy;
  want -= data_to_copy;
}
```

**大改动：COW 三步走**（`lob0update.cc:307-380`）：

```cpp
      /* ① 分配新的 data page，把旧页内容 + 新数据拼进去（整页复制 + 中间替换） */
      new_block = first_page.replace(trx, page_offset, ptr, want, mtr);
      ...
      /* ② 从 free_list 摘一个 index entry（必要时新分配 node page） */
      flst_node_t *new_node = first_page.alloc_index_entry(false);
      index_entry_t new_entry(new_node, mtr, index);
      new_entry.set_versions_null();
      new_entry.set_trx_id(trx->id);
      new_entry.set_trx_id_modifier(trx->id);
      new_entry.set_page_no(new_page.get_page_no());      /* 指向新 data page */
      new_entry.set_data_len(new_page.get_data_len());
      new_entry.set_lob_version(lob_version);             /* 新版本号 */

      /* ③ 新 entry 插进 index list，旧 entry 摘下并"降格"为旧版本 */
      cur_entry.insert_after(base_node, new_entry);
      cur_entry.remove(base_node);
      new_entry.set_old_version(cur_entry);               /* ★ 旧 entry 挂进 versions 链 */
```

> ★ 注意细节：即使是"first page 的 replace"，新分配的也是 **`data_page_t`**（`FIL_PAGE_TYPE_LOB_DATA`）——**first page 本身永不被换掉**，被取代的只是它的 index entry。

`replace()` 的"整页复制 + 中间替换"三步：

```cpp
  /** Copy contents from old page to new page. */
  mlog_write_string(new_ptr, old_ptr, offset, mtr);          /* ① [0, offset) 旧前缀 */
  new_ptr += offset;
  old_ptr += offset;

  /** Copy the new data to new page. */
  ulint data_avail = get_data_len() - offset;
  ulint data_to_copy = (want > data_avail) ? data_avail : want;
  mlog_write_string(new_ptr, ptr, data_to_copy, mtr);        /* ② 新数据 */
  ...
  /** Copy contents from old page to new page. */
  if (want < data_avail) {
    ulint remain = data_avail - want;
    mlog_write_string(new_ptr, old_ptr, remain, mtr);        /* ③ 剩余旧后缀 */
  }
```

**小改动走 undo：把"被覆盖的那段旧字节"原样写进 undo**（`trx0rec.cc`）：

```cpp
  const bool small_change = (bytes_changed <= lob::ref_t::LOB_SMALL_CHANGE_THRESHOLD);

  if (!small_change) {
    /* This is not a small change.  So write the size of the vector as
       0 and bailout. */
    /* ★ 大改动不记 diff —— 靠版本链 + purge 回收 */
    undo_ptr += mach_write_compressed(undo_ptr, 0);
    return undo_ptr;
  }
  ...
  /* Only for small changes to the BLOB, we do regular undo logging. */
  ...
    /* Write the offset. */
    undo_ptr += mach_write_compressed(undo_ptr, bdiff.offset());
    /* Write the length. */
    undo_ptr += mach_write_compressed(undo_ptr, bdiff.length());
    ...
    /* Write the old data. */
    ut_memcpy(undo_ptr, bdiff.old_data(fld->mysql_field), bdiff.length());
    undo_ptr += bdiff.length();
```

> ★ 一次 ≤100 字节的改动**最多跨 2 个 index entry**（即最多跨 2 个数据页）——源码断言 `ut_ad(n_entry == 1 || n_entry == 2)`。这正是阈值取 100 的原因之一。

undo 里记的东西在 `row0upd.h`：

```cpp
/** The modification done to the LOB. */
struct Lob_diff {
  /** The offset within LOB where partial update happened. */
  ulint m_offset = 0;
  /** The length of the modification. */
  ulint m_length = 0;
  /** Changes to the LOB data. */
  const byte *m_old_data = nullptr;
  /** Changes to the LOB index. */
  Lob_index_diff_vec *m_idx_diffs;
  ...
};
```

> ⚠️ **没有 `LOB_DIFF` 这个常量名**（grep `LOB_DIFF|MLOG_LOB` 命中 0）——只有 `Lob_diff` / `Lob_diff_vector` / `upd_field_t::lob_diffs`。

### 2.1 只有"大改动"才递增

```cpp
// lob/lob0update.cc:129-137
  if (small_change) {
    lob_version = first_page.get_lob_version();   // 小改动：版本号不变
  } else {
    lob_version = first_page.incr_lob_version();  // 大改动：+1
  }
```

```cpp
// lob/lob0first.cc:391-399
uint32_t first_page_t::incr_lob_version() {
  const uint32_t cur = get_lob_version();
  const uint32_t val = cur + 1;
  mlog_write_ulint(frame() + OFFSET_LOB_VERSION, val, MLOG_4BYTES, m_mtr);
  return (val);
}
```

递增后写回行内 ref：`blobref.set_offset(lob_version, mtr)`（`lob0update.cc:160`）—— 就是写 `BTR_EXTERN_OFFSET/VERSION` 那 4 字节。

初始化为 1（`lob0first.h:303`；`ref.update(space_id, first_page_no, 1, mtr)` `lob0impl.cc:1056`）。

### 2.2 COW：旧页为什么不释放

```cpp
// lob/lob0first.cc:60-121 (节选)
buf_block_t *first_page_t::replace(trx_t *trx, ulint offset, const byte *&ptr,
                                   ulint &want, mtr_t *mtr) {
  data_page_t new_page(mtr, m_index);
  new_block = new_page.alloc(mtr, false);            // ★ 新分配数据页
  new_page.set_trx_id(trx->id);
  new_page.set_data_len(get_data_len());
  mlog_write_string(new_ptr, old_ptr, offset, mtr);  // 拷 [0, offset)
  mlog_write_string(new_ptr, ptr, data_to_copy, mtr);// 写入新数据
  if (want < data_avail)
    mlog_write_string(new_ptr, old_ptr, remain, mtr);// 拷剩余尾巴
  return new_block;                                   // 旧页原封不动
}
```

旧页必须保留：旧的行版本仍持有 ref（version = 旧值），它们的读请求会按 §4 走到旧 entry → 旧数据页。

### 2.3 旧 entry 如何串进 versions 链

```cpp
// lob/lob0update.cc:355-381 (节选)
    index_entry_t new_entry(new_node, mtr, index);
    new_entry.set_versions_null();
    new_entry.set_page_no(new_page.get_page_no());
    new_entry.set_lob_version(lob_version);            // ★ 新版本号
    cur_entry.set_trx_id_modifier(trx->id);            // ★ 旧 entry 打"修改者"标记
    cur_entry.set_trx_undo_no_modifier(undo_no);
    cur_entry.insert_after(base_node, new_entry);
    cur_entry.remove(base_node);                       // 旧 entry 从 index list 摘掉
    new_entry.set_old_version(cur_entry);              // ★ 旧 entry 挂进新 entry 的 versions 链
```

```cpp
// include/lob0index.h:186-193
  void set_old_version(index_entry_t &entry) {
    entry.move_version_base_node(*this);       // 旧 entry 原来的 versions 链整体移交
    flst_add_first(version_list, node, m_mtr); // 旧 entry 头插进 versions 链
  }
```

结果：`index_list` 里全是最新 entry；每个 entry 带一条 `versions` 链，链上是它自己的历史（版本递减）。多次 COW 形成 `v3 → v2 → v1` 的版本栈。

---

### 三、写入路径

```
ha_innobase::update_row → calc_row_difference (ha_innodb.cc:9808)
 → row_upd (row0umod.cc:160)
 → btr_cur_optimistic_update / btr_cur_pessimistic_update (row0upd.cc:2943)
 → lob::btr_store_big_rec_extern_fields(..., OPCODE_UPDATE) (lob0lob.cc:410)
     判定 :481/:550 —— 必须 off-page 大 LOB 且 upd->is_partially_updated(field_no)
     → lob::update (lob0update.cc:96)     成功则 do_insert=false
     → 否则 lob::insert (:577)            全量重写
```

`lob::update()`（`lob0update.cc:96-163`）核心：

```cpp
const ulint bytes_changed = upd_t::get_total_modified_bytes(*bdiff_vector);
const bool small_change = (bytes_changed <= ref_t::LOB_SMALL_CHANGE_THRESHOLD);  // 100
if (small_change) lob_version = first_page.get_lob_version();
else              lob_version = first_page.incr_lob_version();
for (iter = bdiff_vector->begin(); ...) {
  if (small_change) err = replace_inline(...);   // mlog_write_string 原地覆盖
  else              err = replace(...);          // COW 新页 + 新 entry
}
blobref.set_offset(lob_version, mtr);            // 版本号写回行内 ref
```

**小改动 / 大改动对照表**：

| | 小改动（≤100B） | 大改动（>100B） |
|---|---|---|
| 数据页 | 原地改写 `replace_inline`（`lob0update.cc:501`） | COW 新页 `replace`（`lob0update.cc:268`） |
| lob_version | 不变 | +1 |
| index entry | 原地改 modifier trx/undo_no | 新分配 + 旧 entry 入 versions 链 |
| undo 内容 | N 个 Lob diff（offset/length/old_data + 1~2 个 `lob_index_diff_t`） | **只写 `N=0`**（`trx0rec.cc:1045`） |
| undo 标志 | `TRX_UNDO_MODIFY_BLOB`(bit6) + `TRX_UNDO_UPD_EXTERN`(bit7) | 仅 `TRX_UNDO_UPD_EXTERN` |
| 旧版本来源 | undo 里的 old_data（内存打补丁） | versions 链里的历史数据页 |
| 回滚方式 | `apply_undolog()` 反向写回 | `make_old_version_current()` 扶正旧 entry |

> 注意：`TRX_UNDO_MODIFY_BLOB` 在 8.0.39 是**无条件置位**的（`trx0rec.cc:1244`），它只表示"这条 undo 支持 BLOB 部分更新格式"，不代表真有 diff。真正判据是 `uf->lob_diffs != nullptr && size() > 0`（`lob0purge.cc:73`）。

---

### 四、读路径：版本化 MVCC 读

**两阶段**：先按版本选页（大改动用），再打 undo diff 补丁（小改动用）。

### 4.1 阶段一：按 lob_version 选 entry

```cpp
// lob/lob0impl.cc:1077, 1169-1202 (节选)
  const uint32_t lob_version = ref.version();           // 来自本行版本的 ref
  const uint32_t entry_lob_version = cur_entry.get_lob_version();
  if (entry_lob_version > lob_version) {                // 最新 entry 太新 → 找旧版本
    fil_addr_t node_versions = flst_get_first(cur_entry.get_versions_list(), &mtr);
    while (!fil_addr_is_null(node_versions)) {
      const uint32_t old_lob_version = old_version.get_lob_version();
      if (old_lob_version <= lob_version) break;        // ★ 可见
      node_versions = old_version.get_next();
    }
  }
  read_from_page_no = (old_version.is_null()) ? cur_entry.get_page_no()
                                              : old_version.get_page_no();
```

**语义**：lob_version 是单调递增的快照号；读事务取"版本 ≤ 自己 ref 里记的版本"的最新那个 entry。

### 4.2 阶段二：打 undo 里的旧字节补丁

```cpp
// lob/lob0undo.cc:41-62
void undo_data_t::apply(dict_index_t *index, byte *lob_mem, size_t len,
                        size_t lob_version, page_no_t first_page_no) {
  if (first_page_no == m_page_no) {        // ★ 只有 first page 号对得上才打补丁
    byte *ptr = lob_mem + m_offset;
    memcpy(ptr, m_old_data, m_length);
  }
}
```

### 4.3 三个 undo 容器

```
undo_vers_t  (lob0undo.h:146)   —— 一次读会话的 LOB undo 总容器
   │  get_undo_sequence(field_no)                       (:170)
   └─ undo_seq_t  (lob0undo.h:86)   —— 按 field_no(一个 BLOB 列) 聚合
         │  push_back(undo_data_t&)                     (:113)
         └─ undo_data_t  (lob0undo.h:42)  —— 一次 Binary diff 的 undo 信息
               page_no_t m_page_no;   // LOB first page 号（匹配用）
               ulint     m_offset;    // LOB 内偏移
               ulint     m_length;    // 改动长度（≤100）
               byte     *m_old_data;  // 旧值拷贝
```

生命周期：挂在 `row_prebuilt_t::m_lob_undo`，每行开始 `prebuilt->lob_undo_reset()`（`row0sel.cc:4977/6100`）。

### 4.4 完整读调用链

```
row_search_mvcc
 └─ row_sel_build_prev_vers_for_mysql()        row0sel.cc:3071
     └─ row_vers_build_for_consistent_read()    row0vers.cc:1258
         └─ trx_undo_prev_version_build()       trx0rec.cc:2469
             └─ trx_undo_update_rec_get_update() trx0rec.cc:1740
                 └─ trx_undo_read_blob_update()  trx0rec.cc:904
                     └─ lob_undo->get_undo_sequence(field_no)->push_back(...)
row_sel_store_mysql_field()                     row0sel.cc:2725
 ├─ btr_rec_copy_externally_stored_field()      row0sel.cc:2784 → lob::read() lob0impl.cc:1074
 └─ lob_undo->apply(...)                        row0sel.cc:2818 → lob0undo.cc:41
```

> **MVCC 读与回滚的区别**：读路径传的 `lob_undo != nullptr`（要拷贝 old_data 到内存打补丁）；回滚路径传 `nullptr`（`row0umod.cc:1251`），只记 offset/length/指针，不拷贝（`trx0rec.cc:952`）。

---

### 五、purge 回收

### 5.1 调用链

```
trx_purge()                                     trx0purge.cc
 └─ row_purge_step()                            row0purge.cc:1233
     └─ row_purge_record_func()                 row0purge.cc:1096
         ├─ case TRX_UNDO_DEL_MARK_REC:
         │   row_purge_del_mark()               row0purge.cc:657
         │    └─ btr_cur_pessimistic_delete()    row0purge.cc:391
         │        └─ free_externally_stored_fields()  btr0cur.cc:4834 → lob0lob.cc:1098
         │            └─ lob::purge()            lob0purge.cc:414
         └─ case TRX_UNDO_UPD_EXIST_REC (extern):
             row_purge_upd_exist_or_extern_func() row0purge.cc:702
              └─ lob::purge(&ctx, ...)           row0purge.cc:828-834
```

### 5.2 判定：这个旧版本可以删了吗

```cpp
// include/lob0index.h:178-181
  bool can_be_purged(trx_id_t trxid, undo_no_t undo_no) {
    return ((trxid == get_trx_id_modifier()) &&
            (get_trx_undo_no_modifier() == undo_no));
  }
```

旧 entry 在 COW 时被打了 `trx_id_modifier = 修改者 trx` / `undo_no_modifier = 修改者 undo_no`（§2.3）。**当 purge 处理到"造成这次 COW 的那条 undo 记录"时，trxid/undo_no 恰好匹配 → 该旧版本不再被任何 ReadView 需要 → 释放。**

主循环（`lob0purge.cc:555-588`）：遍历 index list 每个 entry 的 versions 链，逐条 `can_be_purged()` → `purge_version()`。

### 5.3 真正释放

```cpp
// lob/lob0index.cc:111-128
fil_addr_t index_entry_t::purge_version(dict_index_t *index,
                                        flst_base_node_t *lst,
                                        flst_base_node_t *free_list) {
  fil_addr_t next_loc = flst_get_next_addr(m_node, m_mtr);
  flst_remove(lst, m_node, m_mtr);        // 从 versions 链摘下
  purge(index);                            // 释放它指向的数据页
  flst_add_first(free_list, m_node, m_mtr);// entry 归还空闲链
  return (next_loc);
}
```

`index_entry_t::purge()`（`lob0index.cc:83`）里有一句关键判断：`if (type != FIL_PAGE_TYPE_LOB_FIRST)` —— **first page 永远不在这里释放**。

### 5.4 整块销毁 + first page 延迟释放

```cpp
// lob/lob0purge.cc:505-517
  bool ok_to_free = (rec_type == TRX_UNDO_UPD_EXIST_REC ||
                     rec_type == TRX_UNDO_UPD_DEL_REC) &&
                    !btr_first.can_be_partially_updated() &&   // ★ 必须"不可部分更新"
                    (last_trx_id == trxid) && (last_undo_no == undo_no);
  if (rec_type == TRX_UNDO_DEL_MARK_REC || ok_to_free) {
    btr_first.make_empty();
    purge_node->add_lob_page(index, page_id);   // 只登记
```

```cpp
// trx/trx0purge.cc:2561-2573
  /* The first page of LOBs are freed at the end of a purge batch because
  multiple purge threads will access the same LOB ... */
  for (thr = UT_LIST_GET_FIRST(purge_sys->query->thrs); ...) {
    node->free_lob_pages();
  }
```

### 5.5 `mark_not_partially_updatable` 的作用

```cpp
// lob/lob0first.cc:401-413
void first_page_t::mark_cannot_be_partially_updatable(trx_t *trx) {
  uint8_t flags = get_flags();
  flags |= 0x01;                                  // bit0
  mlog_write_ulint(frame() + OFFSET_FLAGS, flags, MLOG_1BYTE, m_mtr);
  set_last_trx_id(trxid);
  set_last_undo_no(undo_no);
}
```

触发场景：
1. update 不是部分更新（整列重写）—— `lob0lob.cc:567-573`，注释写得很直白："This is to inform the purge thread that the older version LOB ... can be freed"。
2. 外部列被改回内联（ext → inline）—— `lob0lob.cc:1312`，调用点 `btr0cur.cc:4089`。

**语义**：置位 ⇒ 该 LOB 已被整列重写，再没有行版本需要它的历史 ⇒ purge 可 `make_empty()` 一次性释放；未置位 ⇒ 只能逐版本扫描回收。

---

### 六、回滚

### 6.1 小改动：`apply_undolog`

```cpp
// lob/lob0update.cc:674-716 (节选)
    const byte *ptr = lob_diff.m_old_data;         // undo 里的旧值
    while (!fil_addr_is_null(node_loc) && want > 0) {
      first_page.replace_inline(page_offset, ptr, want, mtr);   // 原地写回
      page.replace_inline(page_offset, ptr, want, mtr);
      lob_index_diff_t &tmp = lob_diff.m_idx_diffs->at(count);
      cur_entry.set_trx_id_modifier(tmp.m_modifier_trxid);       // 还原 index entry
      cur_entry.set_trx_undo_no_modifier(tmp.m_modifier_undo_no);
```

`lob_index_diff_t`（`row0upd.h:365-377`）存在的唯一目的：记录被小改动"污染"的 index entry 的原 modifier 值，供回滚还原。

### 6.2 大改动：`make_old_version_current`

```cpp
// lob/lob0purge.cc:116-123
    if (cur_entry.can_rollback(trxid, undo_no)) {
      node_loc = cur_entry.make_old_version_current(index, first);
```

```cpp
// include/lob0index.h:169-172
  bool can_rollback(trx_id_t trxid, undo_no_t undo_no) {
    return ((trxid == get_trx_id()) && (get_trx_undo_no() >= undo_no));  // 用 creator
  }
```

`make_old_version_current()`（`lob0index.cc:47-78`）：把 versions 链首节点挪回 index list、versions 基节点移交过去、再 `purge_version()` 掉当前（新）entry。

> **注意**：大改动回滚**不回退 first page 的 lob_version 计数器**（`apply_undolog` 只还原 `last_trx_id`/`last_undo_no`）。版本号单调不回退，靠 index entry 的 versions 链恢复可见性。

### 6.3 回滚不是原子的

`rollback()` 用 `local_mtr` 每轮提交（`lob0purge.cc:137/142/181`），靠 `ref.set_length(0)` 做"部分删除"哨兵，源码预留 3 个 crash 注入点（`lob0purge.cc:139/171/183`：`crash_middle_of_lob_rollback` / `crash_almost_end_of_lob_rollback` / `crash_end_of_lob_rollback`）。这是**幂等/可重入设计**。

---

### 七、崩溃恢复

### 7.1 redo：没有 LOB 专属类型

全枚举 `mlog_id_t` 在 `mtr0types.h:63-274`，**不存在 `MLOG_LOB_*`**。LOB 的一切修改都被降解为通用记录：

| LOB 动作 | redo 类型 | 恢复分发（log0recv.cc） |
|----------|-----------|------------------------|
| `replace_inline` 原地改写 | `MLOG_WRITE_STRING` (30) | `case MLOG_WRITE_STRING:` **2276** → `mlog_parse_string()` |
| lob_version 递增 | `MLOG_4BYTES` (4) | `case MLOG_4BYTES:` **1736** |
| 页类型 / data_len | `MLOG_2BYTES` (2) | `case MLOG_2BYTES:` **1805** |
| 行内 ref 里的 version | `MLOG_4BYTES`（`ref_t::set_offset` `lob0lob.h:449`）；注意它**不是**记录级 update —— off-page 列的聚簇记录更新必走 `MLOG_REC_DELETE`+`MLOG_REC_INSERT` | **1736** |

分发总入口 `recv_parse_or_apply_log_rec_body()`（`log0recv.cc:1582`，switch 在 `:1730`）。

### 7.2 原子性：整个 UPDATE 一个 mtr

`lob::update()` 用的 mtr 就是传进来的 `btr_mtr`（`lob0update.cc:100` `ctx.get_mtr()`）。所有 `Binary_diff`（可跨多个 LOB 页）的写入 + 最后的 `blobref.set_offset(lob_version, mtr)` 都在**同一个 mtr** 内。

`BtrContext::check_redolog_normal()` 会打断 mtr，但在 UPDATE 路径只调用一次（`lob0lob.cc:443`），diff 循环里不调用 —— 所以一次部分更新的所有 LOB 页修改始终落在一个 mtr。崩溃时 redo 按 mtr（以 `MLOG_MULTI_REC_END` 定界）整体 apply。

### 7.3 恢复顺序（srv0start.cc）

| 步骤 | 行号 |
|------|------|
| redo 扫描 | `srv0start.cc:1988` |
| redo apply | `:2036` |
| 重建事务 + purge 队列 | `:2186` |
| DD 事务先回滚 | `:2384-2394` |
| 普通未提交事务回滚（后台线程） | `:2534-2545` → `trx_recovery_rollback_thread` |
| purge 线程 | `:2443` |

回滚入口在 **trx0roll.cc**（不是 trx0trx.cc）：
- `trx_recovery_rollback_thread()` `trx0roll.cc:851`
- `trx_rollback_or_clean_recovered(bool all)` `trx0roll.cc:712`
- `trx_rollback_or_clean_resurrected(trx, all)` `trx0roll.cc:765`

> `trx_rollback_or_clean_all_recovered()` **不是真实符号**，只在注释里出现。

### 7.4 恢复后没有 LOB 校验

`log0recv.cc` 里 LOB 相关只有 3 处 `MLOG_ZIP_WRITE_BLOB_PTR`（压缩页 BLOB 指针）。`lob/` 下所有 `validate*` 都是 `UNIV_DEBUG`/`ut_ad` 级别，release 不生效。唯一带修复语义的是 **IMPORT TABLESPACE**（`row0import.cc:2565-2584`），不是崩溃恢复。

**一致性完全依赖 mtr 原子性 + 事务回滚/purge，而非事后校验。**

---

## 核心调用栈

**部分更新写**
```
ha_innobase::update_row → calc_row_difference (ha_innodb.cc:9808)
 → row_upd (row0umod.cc:160) → btr_cur_pessimistic_update (row0upd.cc:2943)
 → lob::btr_store_big_rec_extern_fields (lob0lob.cc:410)
     → lob::update (lob0update.cc:96)
         → small? replace_inline (:501) : replace (:268)
             → first_page_t::replace (lob0first.cc:60) / data_page_t::replace (lob0pages.cc:59)
             → alloc_index_entry + set_old_version (lob0update.cc:355-381)
     → blobref.set_offset(lob_version, mtr) (:160)
 → trx_undo_report_blob_update (trx0rec.cc:1000)
```

**MVCC 读旧版本**
```
row_sel_build_prev_vers_for_mysql (row0sel.cc:3071)
 → row_vers_build_for_consistent_read (row0vers.cc:1258)
 → trx_undo_prev_version_build (trx0rec.cc:2469)
 → trx_undo_update_rec_get_update (trx0rec.cc:1740)
 → trx_undo_read_blob_update (trx0rec.cc:904)
row_sel_store_mysql_field (row0sel.cc:2725)
 → btr_rec_copy_externally_stored_field (:2784) → lob::read (lob0impl.cc:1074)
 → lob_undo->apply (:2818) → undo_data_t::apply (lob0undo.cc:41)
```

**purge 回收**
```
row_purge_record_func (row0purge.cc:1096)
 → lob::purge (lob0purge.cc:414)
     → ok_to_free? first_page.make_empty() + add_lob_page (:505-545)
     → 否则遍历 versions 链 → can_be_purged (lob0index.h:178) → purge_version (lob0index.cc:111)
     → first page 延迟到 purge batch 末尾 (trx0purge.cc:2561)
```

**回滚**
```
row_undo_mod (row0umod.cc:1272)
 → row_undo_mod_parse_undo_rec (:1202) → trx_undo_update_rec_get_update (trx0rec.cc:1740, lob_undo=nullptr)
 → row_undo_mod_clust_low (row0umod.cc:84) → btr_cur_pessimistic_update (btr0cur.cc:3942)
 → lob::free_updated_extern_fields (lob0lob.cc:957) → lob::purge (lob0lob.cc:1009)
 → rollback (lob0purge.cc:65)
     ├─ 有 diff → rollback_from_undolog (:73) → apply_undolog (lob0update.cc:600)
     └─ 无 diff → make_old_version_current (lob0index.cc:47)
```

---

## Misc

### ★★★ 真名勘误

| 常见说法 | 8.0.39 实际 |
|---|---|
| `LOB_HDR_PART_LEN` / `LOB_HDR_NEXT_PAGE_NO` / `LOB_HDR_SIZE` 是 first page 的字段 | **不是**。它们在 `lob0lob.h:141-148`，是**旧式无索引 BLOB 页**的 8 字节页头。新格式 first page 用 `first_page_t::OFFSET_*` |
| `LOB_HDR_TRX_ID` | **生产代码不存在**，只在 `unittest/gunit/innodb/lob/lob0int.h:41` |
| `LOB_PAGE_DATA` 是一个常量 | **4 个同名不同值**：first=96 / data=49 / node=39 / ZLOB=38 |
| `LOB_PAGE_TRAILER_DATA_LEN` | **不存在**（grep 0 命中）。只有 `LOB_PAGE_TRAILER_LEN = FIL_PAGE_DATA_END = 8` |
| ref 偏移 12 处是 4 字节长度 | 12 号字节是**标志字节**（`owner 128` / `inherited 64` / `being_modified 32`），真实长度在**字节 16–19** |
| `BTR_EXTERN_OFFSET` 与版本号是两个字段 | **同一位置**，`BTR_EXTERN_VERSION == BTR_EXTERN_OFFSET == 8` |
| `node_page_t` 在 `lob0index.h` | 在 **`lob0impl.h:495`**；`lob0index.h` 只有 `index_entry_mem_t` / `index_entry_t` |
| `lob0zip.cc` | **不存在**（该目录无此文件）。`zInserter` 在 `zlob0ins.cc`，`zReader` 在 `lob0lob.cc:157-275` |
| `LOB_DIFF` / `MLOG_LOB` 常量 | **不存在**。是 `Lob_diff` 结构体 / `Lob_diff_vector` / `upd_field_t::lob_diffs` |
| `insert_version()` / `get_version()` | **不存在**。真名 `add_version()` / `set_old_version()` / `get_lob_version()` / `move_version_base_node()` / `make_old_version_current()` |
| `lob::Purge_context` | **不存在**（grep 0）。上下文类是 **`DeleteContext`** |
| 回滚实现在 `lob0undo.cc` | `lob0undo.cc` 只有 `undo_data_t::apply()`（**读路径**用，改内存不改盘）。磁盘回滚在 **`lob0purge.cc`** 的 `rollback()` / `z_rollback()` |
| `Reader` 按 `lob_version` 挑版本 | `Reader` 只服务**老式无索引 BLOB 页**，`FIL_PAGE_TYPE_BLOB`，**没有多版本概念**。版本化读在 **`lob::read()`**（`lob0impl.cc:1067`） |
| `btr_free_externally_stored_field()` | **函数在 8.0.39 不存在**，只剩 2 处陈旧注释。真身是 `free_externally_stored_fields()` → `lob::purge()` → `Deleter::destroy()` |
| `trx_rollback_or_clean_all_recovered()` | 真名 **`trx_rollback_or_clean_recovered`**（`trx0roll.cc:712`）；带 `_all_` 的只在陈旧注释里 |
| "WL#11328" | 源码里**没有**这个编号（grep `11328` 命中 0），模块的自我描述是 "small change" |
| `btr_copy_externally_stored_field` 是普通函数 | 是 **`static inline` 宏包装**（带 `UT_LOCATION_HERE`），真身是 `..._func` |

### 常量速查

| 常量 | 值 | 位置 | 含义 |
|------|-----|------|------|
| `ref_t::LOB_SMALL_CHANGE_THRESHOLD` | **100** | `lob0lob.h:210` | ≤此值为小改动（undo 记 diff + 原地改） |
| `lob::MAX_PARTIAL_UPDATE_LIMIT` | 1000 | `lob0util.h:40` | **死常量，全库无引用**（未实现"部分更新次数上限"） |
| `ref_t::LOB_BIG_THRESHOLD_SIZE` | 2 | `lob0lob.h:203` | **无引用**；`is_big()` 被硬编码 `return true` |
| `first_page_t::node_count()` | 10 | `lob0first.h:431` | first page 固定 index entry 数 |
| `index_entry_t::SIZE` | 60 | `lob0index.h:106` | |
| `BTR_EXTERN_FIELD_REF_SIZE` | 20 | `btr0types.h:66` | 行内 ref 大小 |
| `TRX_UNDO_MODIFY_BLOB` | 64 (bit6) | `trx0rec.h:317` | undo 支持 BLOB 部分更新格式（8.0 无条件置位） |
| `TRX_UNDO_UPD_EXTERN` | 128 (bit7) | `trx0rec.h:320` | undo 涉及外部列 |

### 易混淆点

| 对比 | 说明 |
|------|------|
| **`BTR_EXTERN_OFFSET` vs `BTR_EXTERN_VERSION`** | 同一个 4 字节字段的两种叫法。对普通 BLOB 它是"页内偏移"，对 LOB 它是"版本号" |
| **purge 的 `can_be_purged` vs rollback 的 `can_rollback`** | 前者比 `trx_id_modifier`（修改者），后者比 `trx_id`（创建者）+ `undo_no >=` |
| **`lob::purge` 既是 purge 也是 rollback** | 靠 `DeleteContext::m_rollback` 分派（`lob0purge.cc:500-503`） |
| **first page 谁来释放** | 只能由 `trx0purge.cc:2561` 在 purge batch 末尾统一释放，`index_entry_t::purge()` 显式跳过它 |
| **回滚不回退 lob_version** | 版本号单调不回退，靠 versions 链恢复 |
| **阈值 100 只在正向判定一次** | 回滚时不再计算字节数，只看 undo 里有没有 diff 向量 |

---

## 参考


### 社区文章 / 博客
- 阿里内核月报：*[MySQL · 源码分析 · innodb-BLOB 演进与实现（张云乾/yunqian）](http://mysql.taobao.org/monthly/2022/09/01/)* —— 对源码与理论介绍非常详细，建议多遍读
- 阿里内核月报：*[MySQL · 源码分析 · BLOB 字段 UPDATE 流程分析（zheyu）](http://mysql.taobao.org/monthly/2021/10/03/)*
- [INNODB_BLOB — whoiami](https://whoiami.github.io/INNODB_BLOB)
- [MySQL 8.0 Reference Manual → The BLOB and TEXT Types](https://dev.mysql.com/doc/refman/8.4/en/blob.html)
- 内部文档：[《InnoDB 外部存储页》](https://iwiki.woa.com/p/1674338918#第二部分-外部存储页)

### 相关主题
- 行内外置列标志（`dfield_t::ext`、20B BLOB ref）见 [`record.md`](record.md)
- 溢出页在 btr 中的交互、LOB 与 purge 见 [`../btr.md`](../../index/btr.md) 与 [`../undo_log.md`](../undo_log.md)
