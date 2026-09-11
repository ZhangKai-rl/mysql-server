# InnoDB B+ 树页（Page）物理结构深度解析

> 本篇讲 InnoDB **B+ 树页（page）的物理结构**——`rec_t` 记录（字段编码见 [`../server/handler.md`](../server/handler.md) 的「格式图谱」）所在的物理环境：页头、页目录、记录在页内如何组织、如何在页内定位一条记录。
>
> 承接：`rec_t` 的字段级编码（变长长度列表 / NULL 位图 / 记录头）在 handler.md；本篇讲这些记录**在页内怎么摆放、怎么被找到**。

## 目录

- [页的物理布局](#页的物理布局)
- [page header（56 字节）](#page-header56-字节)
- [infimum / supremum 伪记录](#infimum--supremum-伪记录)
- [记录在页内的组织](#记录在页内的组织)
- [页内记录查找：page_cur_search](#页内记录查找page_cur_search)
- [页的其他物理字段](#页的其他物理字段)
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

## 页的其他物理字段

### page type

只有三种是"索引页"（具备本文的 page header + page directory 结构）：

```cpp
FIL_PAGE_INDEX  = 17855;  // B-tree 数据/索引页
FIL_PAGE_RTREE  = 17854;  // R-tree
FIL_PAGE_SDI    = 17853;  // SDI 索引页
```

其余（`FIL_PAGE_UNDO_LOG`、`FIL_PAGE_INODE`、`FIL_PAGE_TYPE_BLOB` 等）是别的页类型，布局不同。

### page size

默认 16KB（`UNIV_PAGE_SIZE_DEF = 1<<14`），`innodb_page_size` 可配 4/8/16/32/64KB。

### extent / segment / tablespace 层级

```
tablespace (space_id)
 └── segment（叶子/非叶子 segment，由 inode 管理）
      └── extent（默认 1MB，16KB 页时 = 64 个页，连续分配单元）
           └── page（本篇所述物理结构）
```

`PAGE_BTR_SEG_LEAF/TOP`（10B）就是指向 segment inode 的指针（space_id 4B + page_no 4B + offset 2B）。

---

## 参考

**相关文档**
- 上游（`rec_t` 的字段级编码：变长长度列表 / NULL 位图 / 记录头）见 [`../server/handler.md`](../server/handler.md) 的「格式图谱」
- 行如何被读取（`row_search_mvcc` 如何用 `page_cur_search` 定位）见 [`row_search.md`](row_search.md)
- LOB 的 off-page 溢出页见 [`lob.md`](lob.md)
- 页如何被 Buffer Pool 管理（LRU / flush）见 [`buffer_pool.md`](buffer_pool.md)
