# InnoDB B+ 树页（Page）物理结构深度解析

> **边界**：本篇讲**页**（容器）的组织与页内查找；**记录本身**的物理格式（record header、变长长度、NULL 位图）与 offsets 数组见 [`record.md`](record.md)。

> 本篇讲 InnoDB **页（page）的物理结构**——所有页的通用布局（FIL header/page header）、B+ 树索引页的记录组织与页内查找，以及**全部 30 种页类型的清单与常见页的逐字节 layout**。
>
> 承接：`rec_t` 的字段级编码（变长长度列表 / NULL 位图 / 记录头）见 [`record.md`](record.md)；本篇讲这些记录**在页内怎么摆放、怎么被找到**，以及各类管理页/系统页的内部结构。

## 目录

- [页的物理布局](#页的物理布局)
- [page header（56 字节）](#page-header56-字节)
- [infimum / supremum 伪记录](#infimum--supremum-伪记录)
- [记录在页内的组织](#记录在页内的组织)
- [页内记录查找：page_cur_search](#页内记录查找page_cur_search)
- [页的写操作与维护](#页的写操作与维护)
  - [页内插入：page_cur_insert_rec_low](#页内插入page_cur_insert_rec_low)
  - [page directory 维护：slot 分裂与合并](#page-directory-维护slot-分裂与合并)
  - [页内删除：page_cur_delete_rec](#页内删除page_cur_delete_rec)
  - [页重组：btr_page_reorganize_low](#页重组btr_page_reorganize_low)
  - [页分裂概览](#页分裂概览)
- [页类型与全页类型清单](#页类型与全页类型清单)
  - [完整清单（30 种）](#完整清单30-种)
  - [常见页的逐字节 layout](#常见页的逐字节-layout)
  - [四分类与布局归属](#四分类与布局归属)
- [page size](#page-size)
- [extent / segment / tablespace 层级（交叉引用）](#extent--segment--tablespace-层级交叉引用)
- [参考](#参考)

---

## 页的物理布局

### 完整布局图（默认 16KB / COMPACT 格式）

```
页内偏移                        内容                           大小
──────────────────────────────────────────────────────────────────
     0  FIL_PAGE_SPACE_OR_CHKSUM  (新 checksum)               4B
     4  FIL_PAGE_OFFSET            (page_no)                  4B
     8  FIL_PAGE_PREV                                          4B
    12  FIL_PAGE_NEXT                                          4B
    16  FIL_PAGE_LSN                                            8B
    24  FIL_PAGE_TYPE                                           2B
    26  FIL_PAGE_FILE_FLUSH_LSN                                 8B
    34  FIL_PAGE_SPACE_ID                                       4B
       ├──────────── FIL header 结束（38B）──────────────────────
    38  PAGE_N_DIR_SLOTS / PAGE_HEAP_TOP / ... （14 个字段）     56B
       ├──────────── page header 结束（38..93）─────────────────
    94  [infimum extra 5B]
    99  infimum 数据 8B "infimum\0"  ← rec_t 起点
   107  [supremum extra 5B]
   112  supremum 数据 8B "supremum"  ← rec_t 起点
   120  user record 0（heap_no=2，按插入顺序）...
       ...            free space（向高地址增长）...
       ... page directory（2B/slot，向低地址增长）...
       ├──────────── FIL trailer（页尾倒数 8B）─────────────────
 16376  旧 checksum（4B）+ FIL_PAGE_LSN 低 4 字节（4B）
 16383  页尾（UNIV_PAGE_SIZE=16384）
```

三个关键约定：

1. **多字节整数都是大端**（`mach_write_to_2/4/8` / `mach_read_from_2/4/8`）。
2. **records 向高地址增长**（heap），**page directory 向低地址增长**（slot 编号增大 → 地址降低），两者相向挤压中间的 free space。
3. `rec_t` 指针指向字段数据起点（origin），记录头/长度列表在**低地址方向**（见 handler.md 格式图谱）。

### FIL header（38 字节）

| 偏移 | 字段 | 大小 | 含义 |
|---|---|---|---|
| 0 | `FIL_PAGE_SPACE_OR_CHKSUM` | 4B | 新 checksum（4.0.14 前这里是 space_id=0） |
| 4 | `FIL_PAGE_OFFSET` | 4B | 页号 `page_no` |
| 8 | `FIL_PAGE_PREV` | 4B | 同层链表**前驱**页号（`FIL_NULL=0xffffffff` 无） |
| 12 | `FIL_PAGE_NEXT` | 4B | 同层链表**后继**页号 |
| 16 | `FIL_PAGE_LSN` | 8B | 最近修改本页的 redo LSN |
| 24 | `FIL_PAGE_TYPE` | 2B | 页类型（`FIL_PAGE_INDEX` 等） |
| 26 | `FIL_PAGE_FILE_FLUSH_LSN` | 8B | 仅 system tablespace 页 0 用；压缩页复用 |
| 34 | `FIL_PAGE_SPACE_ID` | 4B | 所属 tablespace |

`FIL_PAGE_PREV/NEXT` 把同一 `PAGE_LEVEL` 的页串成**双向链表**（按每页最小键排序）——这是 B+ 树叶子层有序、范围扫描跨页的物理基础。

#### 例外：页 0 的 PREV/NEXT 被复用成版本号

★ **表空间的第一个页（页 0，FSP_HDR）里，`FIL_PAGE_PREV/NEXT` 这 8 字节不存页号，而是存两个版本号**（`fil0types.h:56`）：

```cpp
constexpr uint32_t FIL_PAGE_SRV_VERSION  = 8;    // 顶替 FIL_PAGE_PREV
constexpr uint32_t FIL_PAGE_SPACE_VERSION = 12;  // 顶替 FIL_PAGE_NEXT
```

| 偏移 | 页 0（FSP_HDR）的解释 | 其他页的解释 |
|---|---|---|
| 8 | `FIL_PAGE_SRV_VERSION`（server 版本） | `FIL_PAGE_PREV` |
| 12 | `FIL_PAGE_SPACE_VERSION`（space 版本） | `FIL_PAGE_NEXT` |

原因是**页 0 不属于任何 B+ 树层**，没有同层前后页，`PREV/NEXT` 对它毫无意义——于是借用来记录"这个表空间是由哪个版本的 server 创建的、格式版本是多少"。升级场景下 `dict0upgrade.cc:1314` 会改写它们做格式迁移，`fsp0fsp.ic:420` 的 `fsp_header_get_srv_version`/`get_space_version` 负责读取。

这是"**同一偏移在不同页类型下有不同含义**"的典型例子，读代码时看到 `FIL_PAGE_PREV` 不能直接默认是页号——要先确认页类型。源码里 `dict0dict.h:1110` 有一张现成的 Page 0 全字段布局图（含 FSP header 逐字段偏移），可对照。

页 0 的完整布局（FIL header + FSP header 前几项）：

```
[0, 38)   FIL Header (38B)
  [0,4)     FIL_PAGE_SPACE_OR_CHKSUM   checksum
  [4,8)     FIL_PAGE_OFFSET            = 0
  [8,12)    FIL_PAGE_SRV_VERSION       ← 复用
  [12,16)   FIL_PAGE_SPACE_VERSION     ← 复用
  [16,24)   FIL_PAGE_LSN
  [24,26)   FIL_PAGE_TYPE              = FIL_PAGE_TYPE_FSP_HDR
  [26,34)   FIL_PAGE_FILE_FLUSH_LSN
  [34,38)   FIL_PAGE_SPACE_ID
[38,150)  FSP Header (112B = 32 + 5×16)
  [38,42)   FSP_SPACE_ID
  [42,46)   FSP_NOT_USED（历史遗留）
  [46,50)   FSP_SIZE
  [50,54)   FSP_FREE_LIMIT
  [54,58)   FSP_SPACE_FLAGS
  [58,62)   FSP_FRAG_N_USED
  [62,78)   FSP_FREE          ← 16B base node
  [78,94)   FSP_FREE_FRAG
  [94,110)  FSP_FULL_FRAG
  [110,118) FSP_SEG_ID        ← 8B
  [118,134) FSP_SEG_INODES_FULL
  [134,150) FSP_SEG_INODES_FREE
```

### FIL trailer（8 字节，页尾）

```
页尾倒数 8B： [旧 checksum 4B][FIL_PAGE_LSN 低 4 字节 4B]
```

**为什么 trailer 里还有个 checksum**（历史遗留）：早期 InnoDB 的 checksum 就在页尾这 8 字节里（前 4B checksum + 后 4B 与 `FIL_PAGE_LSN` 低 4B 一致，用于崩溃恢复校验）。后来 space_id 移到偏移 34，偏移 0 腾出来放**新的更强 checksum**（CRC32），旧 checksum 保留兼容。所以现在一页有**两个** checksum：header 偏移 0 的"新"、trailer 里的"旧"。

---

## page header（56 字节）

`PAGE_HEADER` 不是 C struct，而是**偏移常量表**，通过 `page_header_get_field/set_field` 按 2 字节字段读写。

| 字段 | 偏移 | 大小 | 含义 |
|---|---|---|---|
| `PAGE_N_DIR_SLOTS` | 0 | 2B | page directory 的 slot 数 |
| `PAGE_HEAP_TOP` | 2 | 2B | heap 顶部（空闲区下边界），新记录从这里分配 |
| `PAGE_N_HEAP` | 4 | 2B | heap 记录总数（含 infimum/supremum）；**bit15=1 表示 COMPACT 格式** |
| `PAGE_FREE` | 6 | 2B | 已删记录 free list 头（0=无） |
| `PAGE_GARBAGE` | 8 | 2B | 已删未回收字节数 |
| `PAGE_LAST_INSERT` | 10 | 2B | 最近插入记录偏移（插入优化用） |
| `PAGE_DIRECTION` | 12 | 2B | 最近插入方向（LEFT/RIGHT/SAME_REC/SAME_PAGE/NO_DIRECTION） |
| `PAGE_N_DIRECTION` | 14 | 2B | 同方向连续插入次数 |
| `PAGE_N_RECS` | 16 | 2B | 用户记录数（**不含** infimum/supremum） |
| `PAGE_MAX_TRX_ID` | 18 | 8B | 修改过本页的最大 trx_id（二级索引） |
| `PAGE_LEVEL` | 26 | 2B | B+ 树层级，**0=叶子**，>0=内节点 |
| `PAGE_INDEX_ID` | 28 | 8B | 本页归属的索引 id |
| `PAGE_BTR_SEG_LEAF` | 36 | 10B | 叶子 segment inode 定位（仅根页） |
| `PAGE_BTR_SEG_TOP` | 46 | 10B | 非叶子 segment inode 定位（仅根页） |

**最关键的三个**：

- `PAGE_LEVEL`——叶子=0。`page_is_leaf()` 是所有遍历/插入/分裂的第一道分支。
- `PAGE_INDEX_ID`——页与 `dict_index_t` 的绑定。
- `PAGE_HEAP_TOP`——空闲区起点，`page_mem_alloc_heap` 从这里切空间。

---

## infimum / supremum 伪记录

infimum（下界）和 supremum（上界）是两条**固定伪记录**，永远占据 `PAGE_DATA` 之后第一、第二个位置，把用户记录夹在中间。

COMPACT 页初始化数据（`page0page.cc:296`）：

```
94   [infimum extra: n_owned=1, heap_no=0, status=INFIMUM, next=13]
99   ← rec_t 起点 "infimum\0"  (PAGE_NEW_INFIMUM)
107  [supremum extra: n_owned=1, heap_no=1, status=SUPREMUM, next=0]
112  ← rec_t 起点 "supremum"    (PAGE_NEW_SUPREMUM)
120  之后才是 user records
```

注意 `next=13`：supremum 起点 112 − infimum 起点 99 = 13，正是记录头里的 next-record 相对偏移。

status 是记录头第 3 字节的低 3 位（`rem/rec.h:152`）：

```cpp
REC_STATUS_ORDINARY = 0;  // 000 常规记录
REC_STATUS_NODE_PTR = 1;  // 001 内节点 node pointer
REC_STATUS_INFIMUM  = 2;  // 010
REC_STATUS_SUPREMUM = 3;  // 011
```

**为什么需要它们**：让"查第一条/最后一条"无需空页特殊分支——从 `infimum->next` 开始、`supremum->next==0` 天然是链表终点哨兵。page directory 的第一/最后一个 slot 也分别锚定它们。

---

## 记录在页内的组织

三种结构，三个不同的"序"：

### heap（插入序）

记录按**插入顺序**从 `PAGE_HEAP_TOP` 向高地址连续分配，每条记录一个 `heap_no`（infimum=0、supremum=1、用户从 2 起）。heap_no 是**物理插入序**，与键序无关。

### 单向链表（键序）

每条记录头前 2 字节是 **next-record 相对偏移**。从 infimum 沿 next 走到 supremum，得到的是**按索引键升序**的逻辑顺序。

> **物理顺序 ≠ 逻辑顺序**：物理 = 插入序（heap_no），逻辑 = 键序（next 链）。随机插入时二者完全不同。

### page directory（稀疏索引）

槽数组位于页尾倒数 8B 之上，每槽 2B，存**相对页首的偏移**（指向一组记录的第一条）。槽数 = `PAGE_N_DIR_SLOTS`。

**稀疏性**（`PAGE_DIR_SLOT_MAX_N_OWNED=8` / `MIN_N_OWNED=4`）：每个 slot 的 `n_owned` 正常在 [4,8] 之间。超过 8 就分裂 slot，少于 4 就合并。

> **为什么稀疏**：空间换查找时间。全量索引（每条一个 slot）需每记录 2B 且仍要线性找；稀疏后 `n` 条记录只需约 `n/4` 个槽，查找时 slot 二分 O(log n/4) + 组内最多 8 条顺序比较，综合最优。

---

## 页内记录查找：page_cur_search

`page_cur_search_with_match`（`page0cur.cc:334`），两段式查找：

```
输入：tuple（待查键）+ mode（L/LE/G/GE）

[1] slot 二分：low=0, up=PAGE_N_DIR_SLOTS-1
    while (up - low > 1):
       mid = (low+up)/2
       mid_rec = page_dir_slot_get_rec(slot[mid])   ← slot 指向组首记录
       cmp = tuple.compare(mid_rec)
       cmp>0 ? low=mid : up=mid

[2] 组内顺序：while (next(low_rec) != up_rec):
       mid_rec = next(low_rec)
       cmp = tuple.compare(mid_rec)
       cmp>0 ? low_rec=mid_rec : up_rec=mid_rec

[3] 定位：page_cur_position(up_rec 或 low_rec, cursor)
```

要点：

- 二分比较对象是 **slot 指向的组首记录**，不是中间记录本身——稀疏索引二分的标准做法。
- `cur_matched_fields = min(low_matched, up_matched)`：已知左右界共同匹配的字段数可跳过，从该字段继续比（`page0cur.cc:499`）——减少重复比较。
- 组内 `page_rec_get_next_const` 读记录头 next 偏移走到下一条。

---

## 页的写操作与维护

> 查找只用 page directory 二分；但插入/删除/分裂会**改 directory、改链表、改位图**，这些是页作为容器的真正难点。本节补全页的五类写操作。

### 页内插入：page_cur_insert_rec_low

```cpp
rec_t *page_cur_insert_rec_low(
    rec_t *current_rec,       // 插到它之后
    dict_index_t *index,
    const rec_t *rec,         // 待插入的物理记录
    ulint *offsets, mtr_t *mtr) {
  byte *insert_buf;
  ulint rec_size;
  page_t *page = page_align(current_rec);
  rec_t *free_rec;
  rec_t *insert_rec;
  ulint heap_no;

  /* 1. 物理记录在页内的总大小 */
  rec_size = rec_offs_size(offsets);

  /* 2. 先尝试复用 free list 里的空洞 */
  free_rec = page_header_get_ptr(page, PAGE_FREE);
  if (UNIV_LIKELY_NULL(free_rec)) {
    /* 从 free list 头取一个已删记录 */
    ulint foffsets_[REC_OFFS_NORMAL_SIZE];
    ulint *foffsets = foffsets_;
    mem_heap_t *heap = nullptr;
    rec_offs_init(foffsets_);
    foffsets = rec_get_offsets(free_rec, index, foffsets, ULINT_UNDEFINED,
                               UT_LOCATION_HERE, &heap);
    if (rec_offs_size(foffsets) < rec_size) {
      if (heap) mem_heap_free(heap);
      goto use_heap;                              // 空洞太小 → 走 heap
    }
    /* free_rec 指向 data 区起点，extra 区在其低地址侧 */
    insert_buf = free_rec - rec_offs_extra_size(foffsets);
    if (page_is_comp(page)) {
      heap_no = rec_get_heap_no_new(free_rec);    // 复用原 heap_no
      page_mem_alloc_free(page, nullptr, rec_get_next_ptr(free_rec, true), rec_size);
    } else {
      heap_no = rec_get_heap_no_old(free_rec);
      page_mem_alloc_free(page, nullptr, rec_get_next_ptr(free_rec, false), rec_size);
    }
    if (heap) mem_heap_free(heap);
  } else {
  use_heap:
    free_rec = nullptr;
    /* 从 heap 顶部（PAGE_HEAP_TOP）切一块，heap_no 递增 */
    insert_buf = page_mem_alloc_heap(page, nullptr, rec_size, &heap_no);
    if (insert_buf == nullptr) return (nullptr);  // 页满 → 上层触发分裂
  }

  /* 3. 拷贝记录内容 */
  insert_rec = rec_copy(insert_buf, rec, offsets);

  /* 4. 链入单向链表：insert_rec 插到 current_rec 和 next_rec 之间 */
  {
    rec_t *next_rec = page_rec_get_next(current_rec);
    page_rec_set_next(insert_rec, next_rec);
    page_rec_set_next(current_rec, insert_rec);
  }
  page_header_set_field(page, nullptr, PAGE_N_RECS, 1 + page_get_n_recs(page));

  /* 5. 设 n_owned=0 + heap_no */
  if (page_is_comp(page)) {
    rec_set_n_owned_new(insert_rec, nullptr, 0);
    rec_set_heap_no_new(insert_rec, heap_no);
  } else {
    rec_set_n_owned_old(insert_rec, 0);
    rec_set_heap_no_old(insert_rec, heap_no);
  }

  /* 6. 更新 PAGE_LAST_INSERT / PAGE_DIRECTION / PAGE_N_DIRECTION（顺序插入检测） */
  last_insert = page_header_get_ptr(page, PAGE_LAST_INSERT);
  if (!dict_index_is_spatial(index)) {
    if (last_insert == nullptr) {
      page_header_set_field(page, nullptr, PAGE_DIRECTION, PAGE_NO_DIRECTION);
      page_header_set_field(page, nullptr, PAGE_N_DIRECTION, 0);
    } else if ((last_insert == current_rec) &&
               (page_header_get_field(page, PAGE_DIRECTION) != PAGE_LEFT)) {
      /* 连续插在同一位置 → 向右增长 */
      page_header_set_field(page, nullptr, PAGE_DIRECTION, PAGE_RIGHT);
      page_header_set_field(page, nullptr, PAGE_N_DIRECTION,
                            page_header_get_field(page, PAGE_N_DIRECTION) + 1);
    } else if ((page_rec_get_next(insert_rec) == last_insert) &&
               (page_header_get_field(page, PAGE_DIRECTION) != PAGE_RIGHT)) {
      /* 连续插在上一位置之前 → 向左增长 */
      page_header_set_field(page, nullptr, PAGE_DIRECTION, PAGE_LEFT);
      page_header_set_field(page, nullptr, PAGE_N_DIRECTION,
                            page_header_get_field(page, PAGE_N_DIRECTION) + 1);
    } else {
      page_header_set_field(page, nullptr, PAGE_DIRECTION, PAGE_NO_DIRECTION);
      page_header_set_field(page, nullptr, PAGE_N_DIRECTION, 0);
    }
  }
  page_header_set_ptr(page, nullptr, PAGE_LAST_INSERT, insert_rec);

  /* 7. 更新 owner 记录的 n_owned（directory 计数） */
  {
    rec_t *owner_rec = page_rec_find_owner_rec(insert_rec);
    ulint n_owned = page_is_comp(page) ? rec_get_n_owned_new(owner_rec)
                                       : rec_get_n_owned_old(owner_rec);
    /* n_owned 超过 MAX(8) 就分裂 slot（见下节） */
    ...
  }
}
```

三个要点：

1. **free list 复用优先于 heap 分配**：删除的记录不进 `PAGE_GARBAGE` 之后被立即物理移除，而是挂进 `PAGE_FREE` 链表（`page_mem_free` 维护），插入时先看链头空洞够不够大。这避免了频繁的页重组——**小记录删除+插入直接在旧空洞上覆盖**，`heap_no` 也复用。
2. **`PAGE_LAST_INSERT` + `PAGE_DIRECTION` + `PAGE_N_DIRECTION` 是顺序插入检测器**：连续向同一方向插入时 `PAGE_N_DIRECTION` 递增，为上层（B-tree 层）判断"这是顺序加载还是随机插入"提供信号（影响页分裂策略、预读）。
3. 插入只改链表和计数，**不动 page directory 结构**（只是 owner 记录 n_owned +1）——只有当 n_owned 超过上限才触发 `page_dir_split_slot`。

### page directory 维护：slot 分裂与合并

directory 的稀疏性约束是 `n_owned ∈ [MIN=4, MAX=8]`（`PAGE_DIR_SLOT_MIN_N_OWNED` / `MAX_N_OWNED`），插入使某 slot 的 n_owned 到 9 时分裂、删除使其到 3 时合并：

```cpp
void page_dir_split_slot(page_t *page, page_zip_des_t *page_zip, ulint slot_no) {
  rec_t *rec;
  page_dir_slot_t *new_slot, *prev_slot, *slot;
  ulint n_owned;

  slot = page_dir_get_nth_slot(page, slot_no);
  n_owned = page_dir_slot_get_n_owned(slot);
  ut_ad(n_owned == PAGE_DIR_SLOT_MAX_N_OWNED + 1);   // 已达 9

  /* 1. 从 slot 拥有的记录里找"近似中间"那条 */
  prev_slot = page_dir_get_nth_slot(page, slot_no - 1);
  rec = (rec_t *)page_dir_slot_get_rec(prev_slot);
  for (ulint i = 0; i < n_owned / 2; i++) {
    rec = page_rec_get_next(rec);
  }

  /* 2. 在 slot 之下插入一个新 slot */
  page_dir_add_slot(page, page_zip, slot_no - 1);

  /* 3. 新 slot 接管前半，旧 slot 接管后半 */
  new_slot = page_dir_get_nth_slot(page, slot_no);
  slot = page_dir_get_nth_slot(page, slot_no + 1);
  page_dir_slot_set_rec(new_slot, rec);
  page_dir_slot_set_n_owned(new_slot, page_zip, n_owned / 2);          // 4
  page_dir_slot_set_n_owned(slot, page_zip, n_owned - (n_owned / 2));  // 5
}

void page_dir_balance_slot(page_t *page, page_zip_des_t *page_zip, ulint slot_no) {
  slot = page_dir_get_nth_slot(page, slot_no);
  if (slot_no == page_dir_get_n_slots(page) - 1) return;   // 最后一槽无法向上平衡

  up_slot = page_dir_get_nth_slot(page, slot_no + 1);
  n_owned = page_dir_slot_get_n_owned(slot);       // == MIN-1 == 3
  up_n_owned = page_dir_slot_get_n_owned(up_slot);

  if (up_n_owned > PAGE_DIR_SLOT_MIN_N_OWNED) {
    /* 上邻居有多余记录：借一条过来 */
    old_rec = (rec_t *)page_dir_slot_get_rec(slot);
    new_rec = page_is_comp(page) ? rec_get_next_ptr(old_rec, true)
                                 : rec_get_next_ptr(old_rec, false);
    rec_set_n_owned_new(old_rec, page_zip, 0);        // 老组首让位
    rec_set_n_owned_new(new_rec, page_zip, n_owned + 1);
    page_dir_slot_set_rec(slot, new_rec);             // slot 改指新组首
    page_dir_slot_set_n_owned(up_slot, page_zip, up_n_owned - 1);
  } else {
    /* 上邻居也只有 MIN 个：合并两个 slot（删除本 slot） */
    page_dir_delete_slot(page, page_zip, slot_no);
  }
}
```

`page_dir_split_slot`/`balance_slot` 是一对守恒操作：分裂把一个 9 记录的组切成 4+5，平衡把 3 记录的组向上邻居借 1 或直接合并——**保证 n_owned 永远落在 [4,8]**，从而页内查找的 slot 二分 + 组内线性扫描保持 O(log n) + 常数。

### 页内删除：page_cur_delete_rec

```cpp
void page_cur_delete_rec(page_cur_t *cursor, const dict_index_t *index,
                         const ulint *offsets, mtr_t *mtr) {
  page_t *page = page_cur_get_page(cursor);
  current_rec = cursor->rec;

  /* 只剩一条用户记录：整页清空（page_create_empty），而非删除 */
  if (page_get_n_recs(page) == 1 && !recv_recovery_is_on()) {
    page_cur_move_to_next(cursor);
    page_create_empty(page_cur_get_block(cursor),
                      const_cast<dict_index_t *>(index), mtr);
    return;
  }

  cur_slot_no = page_dir_find_owner_slot(current_rec);
  cur_dir_slot = page_dir_get_nth_slot(page, cur_slot_no);
  cur_n_owned = page_dir_slot_get_n_owned(cur_dir_slot);

  /* 0. 写 redo 日志（MLOG_REC_DELETE 之类） */
  if (mtr != nullptr) page_cur_delete_rec_write_log(current_rec, index, mtr);

  /* 1. 重置插入信息 + 递增 modify clock（乐观搜索失效） */
  page_header_set_ptr(page, page_zip, PAGE_LAST_INSERT, nullptr);
  if (mtr != nullptr) buf_block_modify_clock_inc(page_cur_get_block(cursor));

  /* 2. 沿链表找 prev_rec（从上一 slot 的组首顺着走） */
  prev_slot = page_dir_get_nth_slot(page, cur_slot_no - 1);
  rec = (rec_t *)page_dir_slot_get_rec(prev_slot);
  while (current_rec != rec) {
    prev_rec = rec;
    rec = page_rec_get_next(rec);
  }
  page_cur_move_to_next(cursor);
  next_rec = cursor->rec;

  /* 3. 摘链表 */
  page_rec_set_next(prev_rec, next_rec);

  /* 4. 若删的是 slot 组首，slot 改指 prev_rec */
  if (current_rec == page_dir_slot_get_rec(cur_dir_slot)) {
    page_dir_slot_set_rec(cur_dir_slot, prev_rec);
  }

  /* 5. owner slot 的 n_owned 减 1 */
  page_dir_slot_set_n_owned(cur_dir_slot, page_zip, cur_n_owned - 1);

  /* 6. 记录挂进 free list，PAGE_GARBAGE 累加（page_mem_free 内部） */
  page_mem_free(page, page_zip, current_rec, index, offsets);

  /* 7. n_owned 跌破下限 → 平衡 slot */
  if (cur_n_owned <= PAGE_DIR_SLOT_MIN_N_OWNED) {
    page_dir_balance_slot(page, page_zip, cur_slot_no);
  }
}
```

几个反直觉点：

1. **删除是"软删除"**：记录不立刻从物理上消失，而是挂进 `PAGE_FREE` 链表、`PAGE_GARBAGE` 累计字节数——真正回收发生在**页重组**或**复用**（下次插入正好塞进这个空洞）。这也是 `page_cur_delete_rec` 里 `buf_block_modify_clock_inc` 的原因：删除让乐观读缓存失效。
2. **单记录清空是特判**：删掉最后一条用户记录时直接 `page_create_empty` 整页重建（redo 记 `MLOG_PAGE_CREATE`），而不是走删除流程——避免"空页但还带着一堆管理结构"的中间态。
3. 删 slot 组首时 slot 改指 `prev_rec`（前一条记录），而非 next——因为 slot 必须指向"它拥有的那组的第一条"，prev_rec 是组内最靠前的一条。

### 页重组：btr_page_reorganize_low

`PAGE_GARBAGE` 累积到插入放不下（`page_get_max_insert_size` 不足）时触发重组，把有效记录紧凑到页首、清空洞：

```cpp
bool btr_page_reorganize_low(bool recovery, ulint z_level, page_cur_t *cursor,
                             dict_index_t *index, mtr_t *mtr) {
  page_t *page = buf_block_get_frame(block);
  page_zip_des_t *page_zip = buf_block_get_page_zip(block);
  buf_block_t *temp_block;
  page_t *temp_page;
  bool success = false;
  ulint pos;

  /* 关闭 redo：重组用"重建+拷贝"表达，最后整体记一次日志 */
  mtr_log_t log_mode = mtr_set_log_mode(mtr, MTR_LOG_NONE);

  /* 1. 分配临时块，整页拷贝 */
  temp_block = buf_block_alloc(buf_pool);
  temp_page = temp_block->frame;
  buf_frame_copy(temp_page, page);
  if (!recovery) btr_search_drop_page_hash_index(block);   // 失效 AHI

  /* 2. 记住游标在记录列表中的位置 */
  pos = page_rec_get_n_recs_before(page_cur_get_rec(cursor));

  /* 3. 重建页：保留段头/next 指针等全局数据，但清空记录区 */
  page_create(block, mtr, dict_table_is_comp(index->table), fil_page_get_type(page));

  /* 4. 从临时页把记录顺序拷回（先不拷锁位，第 7 步用 heap_no 映射重搬） */
  page_copy_rec_list_end_no_locks(block, temp_block,
                                  page_get_infimum_rec(temp_page), index, mtr);

  /* 5. 二级索引叶页拷回 max_trx_id（MVCC 用） */
  if (dict_index_is_sec_or_ibuf(index) && page_is_leaf(page) && ...) {
    page_set_max_trx_id(block, nullptr, page_get_max_trx_id(temp_page), mtr);
  }

  /* 6. 压缩页重新压缩；失败则恢复旧页 */
  if (page_zip && !page_zip_compress(page_zip, page, index, z_level, mtr)) {
    /* 把临时页内容拷回，放弃重组 */
    ...
  }
}
```

#### 为什么它是 `btr_` 函数：一个跨页/跨层的操作

这个函数定义在 **`btr0btr.cc:1141`**（不是 `page0page.cc`），因为它虽然"只动一页"，却要同时惊动三个非页内的子系统——**这正是它归属 btr 层的理由**：

| 触碰的东西 | 代码 | 属于哪层 |
|---|---|---|
| AHI（自适应哈希索引） | `btr_search_drop_page_hash_index(block)` | **btr 层**：AHI 里所有指向本页记录的哈希项全部失效 |
| 记录锁位图 | `lock_move_reorganize_page(block, temp_block)` | **锁系统**：按 `heap_no` 把锁从旧布局映射到新布局 |
| 二级索引叶页 max_trx_id | `page_get_max_trx_id` → `page_set_max_trx_id` | **MVCC**：二级索引页上的这个字段必须保留（聚簇索引页无此字段） |

对比之下，**页分裂**（`btr_page_split_and_insert`）才是真正"改 B+ 树"的操作：三页参与（分裂页、新右兄弟、父页）、父页插 node pointer、可能级联向上分裂。页重组**不改任何跨页结构**——记录集合、记录顺序、页在 B+ 树里的位置、父节点里的 node pointer 全部不变。所以它归 btr 文件只是"因为要顺手清理 btr 层的缓存（AHI）"，本质是页内物理重排。

要点：

- **重组的本质是"重建 + 顺序拷贝"**：不是原地搬移记录（那要处理大量 next 指针/位图重排），而是 `page_create` 出干净页、再按链表序把记录一条条拷回来（`page_copy_rec_list_end_no_locks`）。代价是临时页的一次整页拷贝（+ `buf_block_alloc`/`buf_block_free`）。
- **redo 先关后开，最终只记一条**：拷贝过程中间态不记日志（`MTR_LOG_NONE`）；成功路径在 `func_exit` 处补记**一条** `MLOG_PAGE_REORGANIZE`（压缩页是 `MLOG_ZIP_PAGE_REORGANIZE`，额外记 1 字节 `z_level`）——崩溃恢复时由 `btr_parse_page_reorganize`（也在 `btr0btr.cc`）重做整页重建。这是 InnoDB 惯用的"复杂修改用一个 mtr 表达为原子日志"。
- ★ **锁位是要重映射的**（不是"不搬"）：`page_copy_rec_list_end_no_locks` 名字里的 `no_locks` 只是说"拷贝记录时先不拷锁位"，紧接着 `lock_move_reorganize_page(block, temp_block)` 会把锁按 **`heap_no` 映射**从旧页搬到新页——因为记录锁挂在 `heap_no` 上，而重组后每条记录的 heap_no 可能变。调用条件是 `!recovery && !dict_table_is_locking_disabled()`（崩溃恢复和 intrinsic 临时表跳过）。
- **强不变量断言**：重组后必须 `data_size1 == data_size2 && max_ins_size1 == max_ins_size2`，否则报 `ER_IB_MSG_30/31` 并 `ut_error`——**重组被定义为"逻辑内容完全不变、仅压缩碎片"的纯物理操作**，这个断言就是在守护该定义。
- **游标位置靠"序号"恢复**：重组前记 `pos = page_rec_get_n_recs_before(cursor->rec)`（这是第几条记录），重组后 `cursor->rec = page_rec_get_nth(page, pos)`——因为记录链表顺序不变，按序号即可还原游标。
- **压缩页可能失败**：`page_zip_compress` 失败则把临时页内容 `memcpy` 回去、放弃重组、返回 false（`MONITOR_INDEX_REORG_ATTEMPTS`/`SUCCESSFUL` 两个计数器分别记尝试与成功）。
- **AHI 只在非 recovery 时失效**：`if (!recovery) btr_search_drop_page_hash_index(block)`——崩溃恢复期间 AHI 本来就是空的。

### 页分裂概览

页内插不下（`page_mem_alloc_heap` 返回 null 或 `page_get_max_insert_size` 不足）时，插入会向上层（B-tree 层）报"页满"，由 `btr_page_split_and_insert` 触发**页分裂**：三页参与（分裂页、新右兄弟页、父页），记录按中点一分为二，父页插一条 node pointer。分裂是跨页操作、涉及 B-tree 平衡，属于 **B-tree 层**（`btr0btr.cc`），不是"单页"的内部操作，故本篇只给概览——细节见 B-tree 相关主题（当前知识库尚未单独成篇）。

---

## 页类型与全页类型清单

### 完整清单（30 种）

页类型写在 FIL header 的 `FIL_PAGE_TYPE`（2B），是崩溃恢复与工具（`ibd2sdi` 等）识别页用途的唯一入口。完整清单来自 `fil0fil.h`：

| 值 | 常量 | 说明 |
|---|---|---|
| 17855 | `FIL_PAGE_INDEX` | B-tree 节点页（具备 page header + directory） |
| 17854 | `FIL_PAGE_RTREE` | R-tree 节点页 |
| 17853 | `FIL_PAGE_SDI` | 表空间 SDI（序列化字典信息）索引页 |
| 0 | `FIL_PAGE_TYPE_ALLOCATED` | 新分配、尚未初始化 |
| 1 | `FIL_PAGE_TYPE_UNUSED` | 未用 |
| 2 | `FIL_PAGE_UNDO_LOG` | undo 日志页 |
| 3 | `FIL_PAGE_INODE` | 段 inode 页（inode 数组） |
| 4 | `FIL_PAGE_IBUF_FREE_LIST` | change buffer 空闲链表页 |
| 5 | `FIL_PAGE_IBUF_BITMAP` | change buffer 位图页 |
| 6 | `FIL_PAGE_TYPE_SYS` | 系统页 |
| 7 | `FIL_PAGE_TYPE_TRX_SYS` | 事务系统头页 |
| 8 | `FIL_PAGE_TYPE_FSP_HDR` | 表空间头页（页 0，FSP header 112B） |
| 9 | `FIL_PAGE_TYPE_XDES` | extent 描述符页（XDES 40B 数组） |
| 10 | `FIL_PAGE_TYPE_BLOB` | 非压缩 BLOB 页 |
| 11 | `FIL_PAGE_TYPE_ZBLOB` | 压缩 BLOB 首页 |
| 12 | `FIL_PAGE_TYPE_ZBLOB2` | 压缩 BLOB 后续页 |
| 13 | `FIL_PAGE_TYPE_UNKNOWN` | 旧表空间的脏类型（flush 时替换） |
| 14 | `FIL_PAGE_COMPRESSED` | 透明页压缩 |
| 15 | `FIL_PAGE_ENCRYPTED` | 加密页 |
| 16 | `FIL_PAGE_COMPRESSED_AND_ENCRYPTED` | 压缩且加密 |
| 17 | `FIL_PAGE_ENCRYPTED_RTREE` | 加密 R-tree 页 |
| 18 | `FIL_PAGE_SDI_BLOB` | 非压缩 SDI BLOB 页 |
| 19 | `FIL_PAGE_SDI_ZBLOB` | 压缩 SDI BLOB 页 |
| 20 | `FIL_PAGE_TYPE_LEGACY_DBLWR` | 旧版 doublewrite 页 |
| 21 | `FIL_PAGE_TYPE_RSEG_ARRAY` | 回滚段数组页（undo 表空间页 3） |
| 22 | `FIL_PAGE_TYPE_LOB_INDEX` | 非压缩 LOB 索引页 |
| 23 | `FIL_PAGE_TYPE_LOB_DATA` | 非压缩 LOB 数据页 |
| 24 | `FIL_PAGE_TYPE_LOB_FIRST` | 非压缩 LOB 首页 |
| 25 | `FIL_PAGE_TYPE_ZLOB_FIRST` | 压缩 LOB 首页 |
| 26 | `FIL_PAGE_TYPE_ZLOB_DATA` | 压缩 LOB 数据页 |
| 27 | `FIL_PAGE_TYPE_ZLOB_INDEX` | 压缩 LOB 索引页（`z_index_entry_t` 数组） |
| 28 | `FIL_PAGE_TYPE_ZLOB_FRAG` | 压缩 LOB 碎片页 |
| 29 | `FIL_PAGE_TYPE_ZLOB_FRAG_ENTRY` | 压缩 LOB 碎片索引页 |

### 常见页的逐字节 layout

下面给出生产环境最常见页类型的内部布局。所有页开头都是 38B 的 FIL header（见「页的物理布局」），页尾是 8B 的 FIL trailer，中间为各自的 content 区。

---

#### ① INDEX 页（`FIL_PAGE_INDEX` = 17855）—— B+ 树数据/索引页

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │  space_id / offset / prev / next / LSN / type / space_id
 38 ├───────────────────────────────────────┤
    │ page header (56B)                     │  见「page header（56 字节）」表
 94 ├───────────────────────────────────────┤
    │ infimum 伪记录 (13B: 5B 头 + 8B 体)    │  heap_no=0，status=INFIMUM
107 │ supremum 伪记录 (13B: 5B 头 + 8B 体)   │  heap_no=1，status=SUPREMUM
120 ├───────────────────────────────────────┤
    │ ↓ user records（向高地址增长）          │  每条 = 记录头(5B) + extra(变长长度+NULL位图) + 数据
    │                                       │
    │   空闲空间                             │  PAGE_HEAP_TOP 到 directory 之间
    │                                       │
    │ page directory（slot 数组，向低地址长） │  每 slot 2B，存组内首记录的页内偏移
    └───────────────────────────────────────┘
    │ FIL trailer (8B)                      │  4B checksum + 4B LSN 低 4 字节
    └───────────────────────────────────────┘
  16K
```

要点：记录区与 slot 区**对向增长**，中间是空闲区；`PAGE_HEAP_TOP` 是记录区的上界。非叶页的 user record 是 node pointer（键值 + 4B 子页号），status=`REC_STATUS_NODE_PTR`。

---

#### ② FSP_HDR 页（页 0，`FIL_PAGE_TYPE_FSP_HDR` = 8）—— 表空间头

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │  其中 FIL_PAGE_TYPE=8
 38 ├───────────────────────────────────────┤
    │ FSP header (112B)                     │  space_id / size / free_limit / flags /
    │                                       │  frag_n_used / FSP_FREE / FSP_FREE_FRAG /
    │                                       │  FSP_FULL_FRAG / seg_id / seg_inodes_full /
    │                                       │  seg_inodes_free（详见 tablespace.md）
150 ├───────────────────────────────────────┤  = 38 + FSP_HEADER_SIZE
    │ XDES 数组（256 个 × 40B ≈ 10KB）        │  描述本页之后 256 个 extent
    │  [0] seg_id 8B | flst_node 12B |      │
    │      state 4B | bitmap 16B            │
    │  [1..255] 同构                         │
    ├───────────────────────────────────────┤
    │ 未使用（页尾剩余空间）                   │
    │ FIL trailer (8B)                      │
    └───────────────────────────────────────┘
```

要点：**FSP_HDR 页兼具 XDES 页功能**（`XDES_ARR_OFFSET` 紧跟 FSP header），只额外多出 112B 的表空间头。它自己也是 extent 0 的第一页，所以 extent 0 的页 0/1 被 XDES + ibuf bitmap 占用。

---

#### ③ XDES 页（`FIL_PAGE_TYPE_XDES` = 9）—— extent 描述符页

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │
 38 ├───────────────────────────────────────┤
    │ FSP header 预留（112B，本页不用）       │  空间保留，内容与 FSP_HDR 无关
150 ├───────────────────────────────────────┤  XDES_ARR_OFFSET
    │ XDES 数组（256 × 40B）                 │  描述接下来的 256 个 extent
    │ 每个 XDES：                            │
    │   +0   XDES_ID        8B  segment id  │
    │   +8   XDES_FLST_NODE 12B 链表节点     │  挂 FSP_FREE/FREE_FRAG/FULL_FRAG 或段内链表
    │   +20  XDES_STATE     4B  状态         │  FREE/FREE_FRAG/FULL_FRAG/FSEG/FSEG_FRAG
    │   +24  XDES_BITMAP    16B 页占用位图   │  64 页 × 2bit（FREE_BIT + 已废弃的 CLEAN_BIT）
    ├───────────────────────────────────────┤
    │ FIL trailer (8B)                      │
    └───────────────────────────────────────┘
```

要点：每 256 个 extent（16KB 页 = 256MB）出现一次 XDES 页；页内 XDES 数组是**连续定长数组**，`xdes_calc_descriptor_page` 用除法算出某页的 XDES 在哪一页。

---

#### ④ INODE 页（`FIL_PAGE_INODE` = 3）—— 段描述符页

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │
 38 ├───────────────────────────────────────┤  FSEG_PAGE_DATA
    │ FSEG_INODE_PAGE_NODE (12B)            │  链表节点：挂 FSP_SEG_INODES_FREE / FULL
50  ├───────────────────────────────────────┤  FSEG_ARR_OFFSET
    │ inode 数组（85 × 192B ≈ 16KB）          │
    │ 每个 inode：                           │
    │   +0   FSEG_ID              8B  段 id │  0 = 空闲槽
    │   +8   FSEG_NOT_FULL_N_USED 4B        │  NOT_FULL 链上已用页数
    │   +12  FSEG_FREE            16B 链表头│
    │   +28  FSEG_NOT_FULL        16B 链表头│
    │   +44  FSEG_FULL            16B 链表头│
    │   +60  FSEG_MAGIC_N         4B        │  恒为 97937874
    │   +64  FSEG_FRAG_ARR        128B      │  32 个 4B 槽位（碎片页页号）
    ├───────────────────────────────────────┤
    │ FIL trailer (8B)                      │
    └───────────────────────────────────────┘
```

85 = `(page_size - FSEG_ARR_OFFSET - 10) / FSEG_INODE_SIZE`（16KB 页）。inode 页本身也由表空间的 `FSP_SEG_INODES_FULL/FREE` 两个链表管理（通过页头的 12B 节点）。

---

#### ⑤ IBUF_BITMAP 页（`FIL_PAGE_IBUF_BITMAP` = 5）—— change buffer 位图页

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │
 38 ├───────────────────────────────────────┤  IBUF_BITMAP = PAGE_DATA
    │ 位图区（每页 4 bit）                    │  覆盖接下来连续的 N 个数据页
    │  bit0-1  IBUF_BITMAP_FREE      空闲度  │  0=0B, 1=512B, 2=1024B, 3=2048B
    │  bit2    IBUF_BITMAP_BUFFERED  有缓存  │  该页有 change buffer 缓存的变更
    │  bit3    IBUF_BITMAP_IBUF      是 ibuf │  页属于 ibuf 树
    ├───────────────────────────────────────┤
    │ FIL trailer (8B)                      │
    └───────────────────────────────────────┘
```

`IBUF_BITS_PER_PAGE = 4`，所以 16KB 页能覆盖约 16384×8/4 = 32768 个数据页（512MB）——这也是**每 256MB（XDES 组）要放一个 bitmap 页**的原因（`FSP_IBUF_BITMAP_OFFSET = 1` 即每个 XDES 组的第 2 页）。

---

#### ⑥ COMPRESSED 页（`FIL_PAGE_COMPRESSED` = 14）—— 透明页压缩

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │
    │  偏移 26 起 8B 复用为控制信息：          │
    │   +26 FIL_PAGE_VERSION        1B      │  控制信息版本
    │   +27 FIL_PAGE_ALGORITHM_V1   1B      │  压缩算法（zlib / lz4）
    │   +28 FIL_PAGE_ORIGINAL_TYPE_V1 2B    │  ★ 原始页类型（压缩前）
    │   +30 FIL_PAGE_ORIGINAL_SIZE_V1 2B    │  原始大小
    │   +32 FIL_PAGE_COMPRESS_SIZE_V1 2B    │  压缩后大小
 38 ├───────────────────────────────────────┤
    │ 压缩后的页数据                          │
    ├───────────────────────────────────────┤
    │ 空洞（原始大小 − 压缩大小，文件系统打洞） │  sparse file hole
    └───────────────────────────────────────┘
```

关键点：FIL header 里 `FIL_PAGE_FILE_FLUSH_LSN`（8B）在压缩页上被**复用**为控制信息——因为压缩页不使用 flush LSN。原始页类型存在 `FIL_PAGE_ORIGINAL_TYPE_V1`，解压后才能知道它原本是 INDEX 还是别的。

---

#### ⑦ RSEG_ARRAY 页（`FIL_PAGE_TYPE_RSEG_ARRAY` = 21）—— undo 表空间回滚段目录

```
  0 ┌───────────────────────────────────────┐
    │ FIL header (38B)                      │  位于 undo 表空间的页 3
 38 ├───────────────────────────────────────┤  RSEG_ARRAY_HEADER = FSEG_PAGE_DATA
    │ +0  RSEG_ARRAY_VERSION        4B      │  0x52534547+1（'RSEG'+1，用于校验）
    │ +4  RSEG_ARRAY_SIZE           4B      │  当前跟踪的 rseg 个数
    │ +8  RSEG_ARRAY_FSEG_HEADER    10B     │  本 rseg array 页所属段的 fseg header
 56 ├───────────────────────────────────────┤  RSEG_ARRAY_PAGES_OFFSET
    │ rseg 槽数组（每槽 4B = rseg header 页号）│  最多 (page_size - 38 - 18 - 200 - 8)/4 个
    ├───────────────────────────────────────┤
    │ 保留区 200B（RSEG_ARRAY_RESERVED_BYTES）│
    │ FIL trailer (8B)                      │
    └───────────────────────────────────────┘
```

8.0 每个独立 undo 表空间页 3 放这个目录，最多 128 个 rseg（`TRX_SYS_N_RSEGS = 128`，因为 rseg id 是 1 字节有符号）。

---

#### ⑧ TRX_SYS 页（`FIL_PAGE_TYPE_TRX_SYS` = 7）—— 系统表空间事务头

```
  0 ┌───────────────────────────────────────┐  系统表空间页 5
 38 ├───────────────────────────────────────┤
    │ +0  TRX_SYS_TRX_ID_STORE   8B         │  ★ 全局 trx id 计数器（持久化于此）
    │ +8  TRX_SYS_FSEG_HEADER    10B        │  事务系统段的 fseg header
 56 ├───────────────────────────────────────┤  TRX_SYS_RSEGS = 8 + FSEG_HEADER_SIZE
    │ rseg 槽数组（128 × 8B）                │  每槽 = rseg 的 (space_id, page_no)
    ├───────────────────────────────────────┤
    │ TRX_SYS_MYSQL_LOG_INFO (页尾 -1000)    │  binlog 文件名/位置 + magic
    │ TRX_SYS_DBLWR (doublewrite 信息)       │
    │ FIL trailer (8B)                      │
    └───────────────────────────────────────┘
```

`TRX_SYS_TRX_ID_STORE` 是 trx id 的持久化位置：崩溃恢复后从这里继续分配，且每次持久化留有 `TRX_SYS_TRX_ID_WRITE_MARGIN = 256` 的余量（避免每 256 个事务就写一次页）。

---

#### ⑨ BLOB / LOB 页

- **老式 `FIL_PAGE_TYPE_BLOB`(10)**：FIL header + 8B 页头（4B 本页数据长度 + 4B 下一页号）+ 数据，链式串联（`BLOB_HDR_SIZE = 8`）。
- **8.0 LOB（`LOB_FIRST`/`LOB_DATA`/`LOB_INDEX` = 24/23/22）**：重构后的三页体系，first page 带 512B 头部（lob_version、flags、trx 信息）+ 10 个 index entry 槽位，data page 纯数据，index page 存 index entry 数组。**完整布局见 [`lob.md`](lob.md)**。
- **压缩 LOB（`ZLOB_FIRST`/`ZLOB_DATA`/`ZLOB_INDEX`/`ZLOB_FRAG`/`ZLOB_FRAG_ENTRY` = 25-29）**：压缩 + 碎片页形态，同样见 [`lob.md`](lob.md)。

#### ⑩ UNDO_LOG 页（`FIL_PAGE_UNDO_LOG` = 2）

FIL header + 186B undo page header（类型/事务 id/段信息/日志起始偏移等）+ undo record 区（向高地址增长）+ 页尾。**完整布局见 [`../undo_log.md`](../undo_log.md)**。

#### ⑪ SDI 页（`FIL_PAGE_SDI` = 17853）

**结构上就是 INDEX 页**（`fil_page_type_is_index()` 包含它）——存序列化字典信息（SDI = Serialized Dictionary Information），数据是压缩的 JSON。SDI 记录的溢出走 `FIL_PAGE_SDI_BLOB`(18) / `FIL_PAGE_SDI_ZBLOB`(19)。每张表在独立表空间里都有两个 SDI 记录（表 + 列/索引元数据），供 `ibd2sdi` 与"表导入/导出"使用。

#### ⑫ ENCRYPTED 页（15 / 16 / 17）

与对应原始页类型布局相同，只是数据在**写入磁盘时加密、读入内存时解密**（页内明文）。`FIL_PAGE_ENCRYPTED`(15) 对应普通页、`FIL_PAGE_COMPRESSED_AND_ENCRYPTED`(16) 对应压缩页、`FIL_PAGE_ENCRYPTED_RTREE`(17) 对应 R-tree 页——**加密不改布局，只改内容**，这是它区别于 COMPRESSED（改了 FIL header 语义）的关键。

---

### 四分类与布局归属

| 类别 | 页类型 | 内容布局归属 |
|---|---|---|
| **索引页**（有 page header + directory） | `FIL_PAGE_INDEX`、`FIL_PAGE_RTREE`、`FIL_PAGE_SDI` | 本篇（layout ①） |
| **表空间管理页** | `FSP_HDR`、`XDES`、`INODE`、`IBUF_BITMAP`、`IBUF_FREE_LIST` | 本篇 layout ②③④⑤，语义见 [`tablespace.md`](tablespace.md) |
| **系统/事务页** | `TRX_SYS`、`SYS`、`RSEG_ARRAY`、`UNDO_LOG` | 本篇 layout ⑦⑧⑩，undo 语义见 [`../undo_log.md`](../undo_log.md)、[`../trx.md`](../trx.md) |
| **LOB/压缩/加密页** | `BLOB`、`ZBLOB`、`ZBLOB2`、`LOB_*`、`ZLOB_*`、`COMPRESSED`、`ENCRYPTED`、`SDI_BLOB` | 本篇 layout ⑥⑨⑫，LOB 语义见 [`lob.md`](lob.md) |

> 索引页用 magic number（17853~17855），其余用连续小整数；`FIL_PAGE_TYPE_UNKNOWN`(13) 是旧表空间脏类型被 flush 时的替换值。所有页共享 38B 的 FIL header。

### 三种索引页的 page header

`PAGE_INDEX`/`PAGE_RTREE`/`PAGE_SDI` 三种索引页共享 page header（56B），这是 `fil_page_type_is_index()` 判据的来源。其余页类型只复用 38B 的 FIL header，其 content 区是各自的结构（XDES 数组、inode 数组、undo record、LOB 索引……），不出现 page directory。

### page size

默认 16KB（`UNIV_PAGE_SIZE_DEF = 1<<14`），`innodb_page_size` 可配 4/8/16/32/64KB。

### extent / segment / tablespace 层级（交叉引用）

页之上是 extent（区）、segment（段）、tablespace（表空间）三级空间管理结构：

```
tablespace (space_id)
 └── segment（叶子/非叶子 segment，由 inode 管理）
      └── extent（16KB 页时 = 64 页 = 1MB，连续分配单元）
           └── page（本篇所述物理结构）
```

三者的**逐字节布局**（FSP header 112B / XDES 40B / inode 192B / fseg header 10B）、segment header → inode → fseg 解析链、分配算法（`fseg_alloc_free_page_low` 七分支 / `fsp_alloc_free_page`）、8.0 lease 机制见 [`tablespace.md`](tablespace.md)。`PAGE_BTR_SEG_LEAF/TOP`（10B）就是指向段 inode 的 fseg header（space_id 4B + page_no 4B + offset 2B）。

---

## 大页（large page）支持：页的物理内存从哪来

前面讲的是页的**逻辑格式**；页在内存里放在哪、用什么粒度分配，是另一回事——这就是 large page 优化的领域。

### 机制

MySQL 8.0 支持 `--large-pages`（对应 `opt_large_pages`）：开启后 **buffer pool 用大页（Linux 上是 HugePages）分配**，减少 TLB miss——buffer pool 动辄几十上百 GB、按 4KB 分页时 TLB 压力极大，这是这个优化的全部动机。

开关（仅 Linux 编译了 `HAVE_LINUX_LARGE_PAGES` 时生效）：

```cpp
/* ha_innodb.cc:4924 */
#ifdef HAVE_LINUX_LARGE_PAGES
  if ((os_use_large_pages = opt_large_pages)) {
    os_large_page_size = opt_large_page_size;
  }
#endif
```

分配走统一封装（`buf0buf.cc:885`，buffer pool chunk）：

```cpp
chunk->mem = static_cast<uint8_t *>(ut::malloc_large_page_withkey(
    ut::make_psi_memory_key(mem_key_buf_buf_pool), mem_size,
    ut::fallback_to_normal_page_t{}));      // ★ 失败回退普通页
```

Linux 实现就是 `mmap(MAP_HUGETLB)`（`detail/ut/large_page_alloc-linux.h:52`）：

```cpp
int mmap_flags = MAP_PRIVATE | MAP_ANON;
#ifndef __FreeBSD__
  mmap_flags |= MAP_HUGETLB;
#endif
void *ptr = mmap(nullptr, n_bytes, PROT_READ | PROT_WRITE, mmap_flags, -1, 0);
```

大页大小从 `/proc/meminfo` 的 `Hugepagesize:` 探测（`large_page_size()`）。

### 四个容易忽略的细节

1. **★ 分配失败会静默回退**：`fallback_to_normal_page_t{}` 表示大页不够时改用普通页，只是打个 warning（`ER_IB_MSG_856`）——所以"配了 large-pages"不代表真的用上了，要查 `os_total_large_mem_allocated` 或 `/proc/meminfo` 的 HugePages 用量确认。
2. **大页大小 ≠ InnoDB 页大小**：`os_large_page_size`（通常 2MB）与 `UNIV_PAGE_SIZE`（16KB）不同源。buffer pool chunk 的可用块数因此可能与请求值差 1（`buf0buf.cc:1025`：大页比 UNIV_PAGE_SIZE 小时可能少分一块，大时可能多分）。
3. **释放时长度要按大页对齐**：`large_page_aligned_free` 会把 size 向上取到 `large_page_default_size` 的倍数再 `munmap`（否则失败，报 `ER_IB_MSG_858`）。
4. **core dump 与大内存**：buffer pool 分配后会 `madvise(MADV_DONTDUMP)`（:865），让 core 文件不包含这些大块内存——否则一个 core 就可能几十 GB。

### 启用前提（不是改个参数就行）

- OS 需预分配 HugePages（`vm.nr_hugepages`）；
- 进程需有 `memlock` 权限（`ulimit -l` / `CAP_IPC_LOCK`），否则 mmap 失败回退；
- 使用大页后这部分内存**不可 swap**。

> 补充：除了 buffer pool，row log 缓冲区（`row0log.cc:260`）等大块内存也走 `ut::malloc_large_page_withkey`。

## 参考

**社区文章 / 博客**
- [InnoDB 文件存储结构 — code0xff](https://code0xff.org/post/2022/12/mysql文件存储结构/)
- 阿里内核月报：*[MySQL 8.0 · InnoDB 页面管理](http://mysql.taobao.org/monthly/2022/12/03/)*（独立表空间 extent 分组与固定页规则的出处）
- [InnoDB 文件组织结构 — 利维坦](https://leviathan.vip/2019/04/18/InnoDB的文件组织结构/)
- [InnoDB 存储结构详解 — 51CTO](https://www.51cto.com/article/777873.html)
- [page directory 分组规则解析](https://blog.csdn.net/qq_32099833/article/details/123150701)（infimum 1 条 / supremum 1-8 / 其余 4-8 的通俗表述）

**相关文档**
- 记录的字段级编码（变长长度列表 / NULL 位图 / 记录头 / offsets 数组）见 [`record.md`](record.md)
- 行如何被读取（`row_search_mvcc` 如何用 `page_cur_search` 定位）见 [`row_search.md`](../row_search.md)
- 表空间/extent/segment 空间管理（FSP header/XDES/inode/fseg）见 [`tablespace.md`](tablespace.md)
- LOB 的 off-page 溢出页见 [`lob.md`](lob.md)
- 页如何被 Buffer Pool 管理（LRU / flush）见 [`buffer_pool.md`](../buffer_pool.md)
