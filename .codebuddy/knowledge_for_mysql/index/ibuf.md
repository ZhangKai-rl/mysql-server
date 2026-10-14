# InnoDB change buffer（insert buffer / ibuf）深度解析

> 基于 MySQL 8.0.39 源码，核心文件 `storage/innobase/ibuf/ibuf0ibuf.cc` / `include/ibuf0ibuf.h`。涵盖：**为什么只缓存非唯一二级索引**（唯一性检查与"页不在 BP"的互斥）、**ibuf 树与 bitmap 的物理布局**、**插入路径的 14 条否决条件**、**合并路径的 8 条触发条件**、**redo 的真相**（ibuf 树走的是普通 B-tree redo）、**三级自我保护 contract**、**★ 为什么 purge/delete-mark 反而能缓存唯一索引**。

> **边界**：本篇讲 **change buffer 本身**。Buffer Pool 的读页与预读见 [`buffer_pool.md`](../innodb/buffer_pool.md)；二级索引的 B-tree 操作见 [`btr.md`](btr.md)；purge 与 undo 见 [`undo_log.md`](../innodb/undo_log.md)；崩溃恢复见 [`recovery.md`](../innodb/recovery.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [物理结构](#物理结构)
    - [ibuf 记录格式](#ibuf-记录格式)
  - 写路径（插入）
    - [插入路径 `ibuf_insert`](#插入路径-ibuf_insert)
  - 读路径（合并）
    - [★ 合并路径](#-合并路径)
    - [ibuf bitmap 的维护](#ibuf-bitmap-的维护)
  - 协同与生命周期
    - [崩溃恢复](#崩溃恢复)
    - [purge 与 ibuf](#purge-与-ibuf)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

change buffer（旧称 insert buffer，简称 ibuf）是 **系统表空间里的一棵 B-tree**，用来缓存**二级索引页不在 Buffer Pool 时**对该页的修改。等该页以后被读到 BP 时，再把缓存的修改合并（merge）进去。

源码注释的定义（`include/ibuf0ibuf.h`）：

```c
/* The purpose of the insert buffer is to reduce random disk access.
When we wish to insert a record into a non-unique secondary index and
the B-tree leaf page where the record belongs to is not in the buffer
pool, we insert the record into the insert buffer B-tree, indexed by
(space_id, page_no).  When the page is eventually read into the buffer
pool, we look up the insert buffer B-tree for any modifications to
the page, and apply these upon the completion of the read operation.
This is called the insert buffer merge. */
```

一句话：**用"顺序写 ibuf + 延迟的批量随机读"换掉"每次 DML 的随机读 + 随机写"**。

### 用途

```
不用 change buffer：  改 1 行 → 读二级索引页（随机读，云盘 0.1~3 ms）→ 改 → 后续刷脏
用   change buffer：  改 1 行 → 写 ibuf（内存 + 系统表空间，顺序）→ 立即返回
                      同一页的多次修改合并 → 页被读时一次 merge
```

**双重收益**：① 省掉随机读；② 同一页的多次修改合并成一次。

### 版本演进

| 版本 | 变化 |
|------|------|
| 4.0 | 引入 insert buffer，只缓存 INSERT |
| 4.1 | 记录格式加入 space_id + marker（此前只有 page_no） |
| 5.0.3 | COMPACT 格式支持 |
| **5.5** | **★ delete buffering**：新增 `IBUF_OP_DELETE_MARK` / `IBUF_OP_DELETE`，记录新增 4 字节 info（counter + type + flags）；改名 change buffer |
| 8.0 | 结构稳定；术语统一为 change buffer（但源码里仍全是 `ibuf`） |

---

## 理论基础

### 设计思想与权衡

#### 一、为什么只对二级索引有效

**硬性断言**：`ibuf_insert` 与 `ibuf_insert_low` 开头就是 `ut_a(!index->is_clustered)`（release 版也会 abort）。

**三个理由**：

1. **语义原因**：聚簇索引是 DML 的**必达路径**——INSERT 必须先写聚簇索引，UPDATE/DELETE 必须先定位聚簇索引记录。所以聚簇索引页**几乎总已在 BP 中**，缓存它没有收益。
2. **记录格式原因**：ibuf 用**字段数**而不是 `dict_index_t` 元数据来解析记录（见[记录格式](#ibuf-记录格式)），而聚簇索引记录含系统列（`DB_TRX_ID`/`DB_ROLL_PTR`），无法脱离真实 index 解析。源码注释（`ibuf0ibuf.cc`）：*"The insert buffer is only used for secondary indexes, whose records never contain any system columns, such as DB_TRX_ID."*
3. **主键也是唯一索引**，同样需要读页做唯一性检查（见下）。

#### 二、★★ 为什么"唯一二级索引的 INSERT"不能被缓存

源码证据在 `ibuf_should_try()`（`include/ibuf0ibuf.ic:116`）与它的唯一调用点（`btr/btr0cur.cc:1065`）：

```cpp
/** A basic partial test if an insert to the insert buffer could be possible and
 recommended. */
static inline bool ibuf_should_try(
    dict_index_t *index,     /*!< in: index where to insert */
    ulint ignore_sec_unique) /*!< in: if != 0, we should
                             ignore UNIQUE constraint on
                             a secondary index when we
                             decide */
{
  return (innodb_change_buffering != IBUF_USE_NONE && ibuf->max_size != 0 &&
          index->space != dict_sys_t::s_dict_space_id &&
          !index->is_clustered() && !dict_index_is_spatial(index) &&
          !dict_index_has_desc(index) &&
          index->table->quiesce == QUIESCE_NONE &&
          (ignore_sec_unique || !dict_index_is_unique(index)) &&
          srv_force_recovery < SRV_FORCE_NO_IBUF_MERGE);
}
```

```cpp
// btr0cur.cc:1061 —— 唯一调用点
  } else if (latch_mode <= BTR_MODIFY_LEAF) {
    rw_latch = latch_mode;

    if (btr_op != BTR_NO_OP &&
        ibuf_should_try(index, btr_op != BTR_INSERT_OP)) {
      /* Try to buffer the operation if the leaf
      page is not in the buffer pool. */

      fetch = btr_op == BTR_DELETE_OP ? Page_fetch::IF_IN_POOL_OR_WATCH
                                      : Page_fetch::IF_IN_POOL;
```

> ★★ **`ignore_sec_unique = (btr_op != BTR_INSERT_OP)`** —— 这一行就是答案：**只有 INSERT 必须非唯一**；DELETE_MARK / DELETE 即便在唯一二级索引上也允许缓冲，因为删除不需要做唯一性检查。
>
> ⚠️ 另外注意 8.0 起多了一条：**降序索引（`dict_index_has_desc`）也被禁用** change buffering。
>
> ⚠️ `btr_op == BTR_DELETE_OP`（purge 的 delete）用 `Page_fetch::IF_IN_POOL_OR_WATCH`——页不在 BP 时设置 **buffer pool watch**，之后页一读入立刻触发 merge，保证"刚被 purge 的页进入 BP 前不会残留已 purge 的旧记录"。

**★ 唯一性检查为什么必须读页**（源码佐证，见 `ibuf_insert_low()` 闸门 E）：

```cpp
  /* We check if the index page is suitable for buffered entries */
  if (buf_page_peek(page_id) || lock_rec_expl_exist_on_page(page_id)) {
    ibuf_mtr_commit(&bitmap_mtr);
    goto fail_exit;
  }
```

> ★ 唯一性约束的判定需要扫描索引页上的已有记录。既然"页不在 buffer pool"是缓冲的前提，那就无法做唯一性检查——两者在定义上互斥。DELETE_MARK/DELETE 是先定位记录再改标记，同样需要读页……**但它们之所以能缓冲，是靠 buffer pool watch + 读页后 merge 的时序保证，而不是靠"不读页"**。真正"不读页也能做"的只有 INSERT。

这是理解 change buffer 最关键、也最容易说错的一点。

**关键代码**（`btr0cur.cc`）：

```c
  } else if (latch_mode <= BTR_MODIFY_LEAF) {
    rw_latch = latch_mode;

    if (btr_op != BTR_NO_OP &&
        ibuf_should_try(index, btr_op != BTR_INSERT_OP)) {
      /* Try to buffer the operation if the leaf
      page is not in the buffer pool. */
      fetch = btr_op == BTR_DELETE_OP ? Page_fetch::IF_IN_POOL_OR_WATCH
                                      : Page_fetch::IF_IN_POOL;
    }
  }
```

> **第二个参数传的是 `btr_op != BTR_INSERT_OP`**，即 `ignore_sec_unique`：

- **`BTR_INSERT_OP`（普通 INSERT）→ `ignore_sec_unique = 0`** → `ibuf_should_try` 要求 `!dict_index_is_unique(index)` → **唯一二级索引的 INSERT 不缓存**。
- **`BTR_DELMARK_OP` / `BTR_DELETE_OP` → `ignore_sec_unique = 1`** → **唯一二级索引的 delete-mark / purge 反而可以被缓存！**
- `BTR_INSERT_IGNORE_UNIQUE_OP`（上层已保证唯一性，如 bulk load、online DDL 的 row log 应用）也能缓存。

**根本机制（为什么唯一索引必须读页）**：

插入唯一二级索引前必须做唯一性检查：

```c
// row0ins.cc
    if (!thr_get_trx(thr)->check_unique_secondary) {
      search_mode |= BTR_IGNORE_SEC_UNIQUE;
    }
```

正常 DML 路径下 `check_unique_secondary` 为真 → 必须调 `row_ins_scan_sec_index_for_duplicate` **把索引页读进 BP 并加 S 锁**。

> **★ 这就是互斥的本质**："必须读页做唯一性检查"与"页不在 BP 才缓存"这两条**不可能同时成立**。所以唯一索引的 INSERT 天然无法用 change buffer——不是策略选择，是逻辑必然。

#### 三、"merge 必须永远成功"（整个子系统的设计基石）

源码注释（`ibuf0ibuf.h`）：

```c
/* The insert buffer merge must always succeed.  To guarantee this,
the insert buffer subsystem keeps track of the free space in pages for
which it can buffer operations.  Two bits per page in the insert
buffer bitmap indicate the available space in coarse increments.  The
free bits in the insert buffer bitmap must never exceed the free space
on a page.  It is safe to decrement or reset the bits in the bitmap in
a mini-transaction that is committed before the mini-transaction that
affects the free space.  It is unsafe to increment the bits in a
separately committed mini-transaction, because in crash recovery, the
free bits could momentarily be set too high. */
```

**核心不变式**：`IBUF_BITMAP_FREE` 位必须**悲观地低估**页的空闲空间——只能减、不能单独加。

为什么：如果 merge 时页装不下，`ibuf_insert_to_index_page` 会失败甚至触发页分裂，而 merge 是在**读页的完成回调里**做的，没有"失败回退"的路径。所以必须保证"缓存的量 ≤ 页的真实空闲空间"。

#### 四、代价清单

| 代价 | 说明 |
|------|------|
| **读路径变慢** | 每次读页可能要 merge，增加 CPU 开销 |
| **可能触发页分裂** | merge 时插入记录可能导致分裂（额外 I/O） |
| **占用 BP** | ibuf 树本身在系统表空间，但其页会占 BP；上限 `innodb_change_buffer_max_size`% |
| **崩溃恢复依赖 redo** | ibuf 的修改记 redo，所以崩溃安全（但也意味着它会加重 redo 负担） |
| **"写完立刻读"是负收益** | 见 [Misc](#-什么时候该关-change-buffer) |

### 他库对比

| 数据库 | 类似机制 |
|--------|---------|
| **MySQL / InnoDB** | change buffer（**持久化在磁盘**，崩溃安全） |
| **PostgreSQL** | 无直接对应（HOT update + visibility map 解决部分问题） |
| **Oracle** | 无（延迟块清除 / 直接路径插入） |

> change buffer 是 InnoDB 的特色设计，源于"二级索引多 + 随机写多"的 OLTP 场景。

---

## 核心实现

### 物理结构

#### 1.1 ibuf 树的位置

| 常量 | 值 | 定义位置 |
|---|---|---|
| `IBUF_SPACE_ID` | **0**（系统表空间） | `include/ibuf0types.h` |
| `IBUF_HEADER_PAGE_NO` | **3** | `fsp0types.h` / `ibuf0ibuf.h` |
| `IBUF_TREE_ROOT_PAGE_NO` | **4** | `fsp0types.h` / `ibuf0ibuf.h` |
| `FSP_IBUF_BITMAP_OFFSET` | **1** | `fsp0types.h` |

**只有一棵 ibuf 树，永远在 space 0**（`ibuf0ibuf.cc`：*"there will only be one insert buffer tree, and that is in the system tablespace"*）。

> **为什么要单独的 header page(3) 而不直接用 root page(4)**：`ibuf_add_free_page` 的注释说明——避免在持有 ibuf 树 latch 时递归进入 ibuf。

#### 1.2 ibuf bitmap 页

⚠️ **勘误**：
- **不存在 `IBUF_BITMAP_PAGE` / `IBUF_BITMAP_OFFSET` / `IBUF_BITMAP_PAGE_NO` / `IBUF_BITMAP_SIZE` 常量**。bitmap 页页号由 **`FSP_IBUF_BITMAP_OFFSET = 1`**（`fsp0types.h:162`，**不在 ibuf 头文件里**）+ 换算函数得出。
- **不存在 `FSP_HDR_XDES_OFFSET`**（真名 `FSP_XDES_OFFSET = 0`，`fsp0types.h:159`）。
- **free 位占 2 bit 不是 3 bit；每页共 4 bit**。

```cpp
// ibuf0ibuf.cc:48
/** Number of bits describing a single page */
constexpr size_t IBUF_BITS_PER_PAGE = 4;
static_assert(IBUF_BITS_PER_PAGE % 2 == 0,
              "IBUF_BITS_PER_PAGE must be an even number!");

/** The start address for an insert buffer bitmap page bitmap */
constexpr uint32_t IBUF_BITMAP = PAGE_DATA;      // ★ = 94（38 FIL 头 + 36 PAGE 头 + 2×10 segment header）

// ibuf0ibuf.cc:240
/** Bits indicating the amount of free space */
constexpr uint32_t IBUF_BITMAP_FREE = 0;         // ★ bit0+bit1，值域 0..3
/** true if there are buffered changes for the page */
constexpr uint32_t IBUF_BITMAP_BUFFERED = 2;     // ★ bit2
/** true if page is a part of the ibuf tree, excluding the root page, or is in
 the free list of the ibuf */
constexpr uint32_t IBUF_BITMAP_IBUF = 3;         // ★ bit3
```

```cpp
// fsp0types.h:153
/** extent descriptor */
constexpr uint32_t FSP_XDES_OFFSET = 0;
/** insert buffer bitmap; The ibuf bitmap pages are the ones whose page number
is the number above plus a multiple of XDES_DESCRIBED_PER_PAGE */
constexpr uint32_t FSP_IBUF_BITMAP_OFFSET = 1;
constexpr uint32_t FSP_FIRST_INODE_PAGE_NO = 2;
```

**页号换算（唯一一个函数）**：

```cpp
// ibuf0ibuf.cc:681
static inline const page_id_t ibuf_bitmap_page_no_calc(
    const page_id_t &page_id, const page_size_t &page_size) {
  page_no_t bitmap_page_no;
  bitmap_page_no = FSP_IBUF_BITMAP_OFFSET +
                   (page_id.page_no() & ~(page_size.physical() - 1));
  //  ★ physical 是 2 的幂 ⇒ & ~(phys-1) = 向下取 phys 整数倍，再 +1
  return (page_id_t(page_id.space(), bitmap_page_no));
}

// ibuf0ibuf.ic:147 —— 反判
static inline bool ibuf_bitmap_page(const page_id_t &page_id,
                                    const page_size_t &page_size) {
  return ((page_id.page_no() & (page_size.physical() - 1)) ==
          FSP_IBUF_BITMAP_OFFSET);
}
```

**每页占几位 / 一个 bitmap 页管多少页**（推导，源码依据 `ibuf0ibuf.cc:597`）：

```
bit_offset  = (page_no % page_size.physical()) * IBUF_BITS_PER_PAGE + bit
byte_offset = bit_offset / 8
```

- 每页 4 bit ⇒ 一个 `physical` 大小的 bitmap 页需 `physical × 4 / 8 = physical / 2` 字节。16K 页需 8192 字节，可用 `16384 − 94 − 8 = 16282` ⇒ 装得下。
- **一个 bitmap 页管理 `page_size.physical()` 个页**：编号 `phys×k + 1` 的 bitmap 页描述区间 `[phys×k, phys×k + phys − 1]`。16K 页即 **16384 页 = 256 MiB**——`ibuf_check_bitmap_on_import()` 的官方注释坐实："`16 KiB * 16384 = 256 MiB`"（`ibuf0ibuf.cc:4448`）。
- 区间内第 0 号是 XDES 页、第 1 号是 bitmap 自身，两组 bit 无意义，所以 import 校验从 `FSP_IBUF_BITMAP_OFFSET + 1 = 2` 开始扫。

**free bits 的 4 档量化**（`ibuf0ibuf.ic:158`）：

```cpp
/** An index page must contain at least UNIV_PAGE_SIZE /
IBUF_PAGE_SIZE_PER_FREE_SPACE bytes of free space for ibuf to try to
buffer inserts to this page. */
constexpr uint32_t IBUF_PAGE_SIZE_PER_FREE_SPACE = 32;   // 每档 = page_size/32 字节

static inline ulint ibuf_index_page_calc_free_bits(ulint page_size,
                                                   ulint max_ins_size) {
  ulint n;
  n = max_ins_size / (page_size / IBUF_PAGE_SIZE_PER_FREE_SPACE);
  if (n == 3) { n = 2; }      // ★★ 3 刻意降到 2：又一处"悲观低估"
  if (n > 3) { n = 3; }
  return (n);
}

static inline ulint ibuf_index_page_calc_free_from_bits(
    const page_size_t &page_size, ulint bits) {
  if (bits == 3) {
    return (4 * page_size.physical() / IBUF_PAGE_SIZE_PER_FREE_SPACE);
  }
  return (bits * (page_size.physical() / IBUF_PAGE_SIZE_PER_FREE_SPACE));
}
```

> ★ 16K 页：每档 512 字节 ⇒ bits=0/1/2/3 反解为 0 / 512 / 1024 / **2048**（bits==3 是开放的"≥4 档"）。

**bitmap 页的初始化**（`ibuf_bitmap_page_init()`，`ibuf0ibuf.cc:542`）：

```cpp
void ibuf_bitmap_page_init(buf_block_t *block, mtr_t *mtr) {
  page_t *page = buf_block_get_frame(block);
  fil_page_set_type(page, FIL_PAGE_IBUF_BITMAP);   // ★ 页类型 = 5

  /* Write all zeros to the bitmap */
  byte_offset =
      UT_BITS_IN_BYTES(block->page.size.physical() * IBUF_BITS_PER_PAGE);
  //  ★ 16K 页 → 16384*4/8 = 8192 字节全清 0
  memset(page + IBUF_BITMAP, 0, byte_offset);

  /* The remaining area (up to the page trailer) is uninitialized. */
  //  ★ 94+8192 = 8286，到 trailer 之间留垃圾

  mlog_write_initial_log_record(page, MLOG_IBUF_BITMAP_INIT, mtr);
  //  ★★ 唯一用到 ibuf 专属 redo 类型的地方
}
```

```cpp
// fsp0fsp.cc:1530 —— 创建 bitmap 页必须"用独立 mtr"
      /* Initialize the ibuf bitmap page in a separate
      mini-transaction because it is low in the latching
      order, and we must be able to release its latch.
      Note: Insert-Buffering is disabled for tables that
      reside in the temp-tablespace. */
      if (!fsp_is_system_temporary(space->id)) {
        mtr_t ibuf_mtr;
        mtr_start(&ibuf_mtr);
        if (space->purpose == FIL_TYPE_TEMPORARY) {
          mtr_set_log_mode(&ibuf_mtr, MTR_LOG_NO_REDO);
        }
        const page_id_t page_id(space->id, i + FSP_IBUF_BITMAP_OFFSET);
        block = buf_page_create(page_id, page_size, RW_SX_LATCH, &ibuf_mtr);
        fsp_init_file_page(block, &ibuf_mtr);
        ibuf_bitmap_page_init(block, &ibuf_mtr);
        mtr_commit(&ibuf_mtr);       // ★★ 立刻提交，释放 bitmap 页 latch
      }
```

> ⚠️ **不存在 `fsp_ibuf_page_low()`**（那是 5.x 的名字）。
>
> ★ 临时表空间**完全不建** ibuf bitmap 页 ⇒ 临时表走不了 change buffer。
>
> ★ latch 层级：bitmap 页标为 `SYNC_IBUF_BITMAP`，**非常低**。所以 `buf0rea.cc` 里 bitmap 页**禁止异步 IO / 禁止 read-ahead**，`fil0fil.cc:7628` 断言它必须走 sync AIO。

**布局**：

```c
constexpr size_t IBUF_BITS_PER_PAGE = 4;      // 每页 4 bit
constexpr uint32_t IBUF_BITMAP = PAGE_DATA;   // = 94，位图起始偏移

constexpr uint32_t IBUF_BITMAP_FREE = 0;      // 2 bit：页空闲空间档位
constexpr uint32_t IBUF_BITMAP_BUFFERED = 2;  // 1 bit：该页有缓存变更
constexpr uint32_t IBUF_BITMAP_IBUF = 3;      // 1 bit：该页属于 ibuf 树或空闲链表
```

| 位 | 位数 | 含义 |
|---|---|---|
| `FREE` | **2 bit** | 页空闲空间的粗粒度档位（0–3） |
| `BUFFERED` | 1 bit | 有缓存的变更 |
| `IBUF` | 1 bit | 属于 ibuf 树（除 root）或 ibuf 空闲链表 |

**bitmap 页号计算**（`ibuf0ibuf.cc`）：

```c
  bitmap_page_no = FSP_IBUF_BITMAP_OFFSET +
                   (page_id.page_no & ~(page_size.physical - 1));
```

16 KiB 页下：页号按 16384 对齐 → **每 16384 个页（256 MiB）一张 bitmap 页**，页号 = `1 + k*16384`。每张 bitmap 页占 `16384 × 4 bit = 8192 字节`。

**FREE 位的档位映射**（16 KiB 页，`IBUF_PAGE_SIZE_PER_FREE_SPACE = 32`，故粒度 512 B）：

| FREE 值 | 代表的最大可插入空间 |
|---|---|
| 0 | < 512 B（**不允许缓存 INSERT**） |
| 1 | ≥ 512 B（1/32 页） |
| 2 | ≥ 1024 B（1/16 页；1536 B 也记作 2，因为 `n==3` 被压到 2） |
| 3 | ≥ 2048 B（1/8 页） |

```c
static inline ulint ibuf_index_page_calc_free_bits(ulint page_size,
                                                   ulint max_ins_size) {
  ulint n = max_ins_size / (page_size / IBUF_PAGE_SIZE_PER_FREE_SPACE);
  if (n == 3) { n = 2; }
  if (n > 3) { n = 3; }
  return (n);
}
```

---

### ibuf 记录格式

#### 2.1 5.5+ 格式（当前）

```
 field#0  SPACE    : 4 bytes  (space id)
 field#1  MARKER   : 1 byte   (恒为 0，用于区分 <4.1 格式)
 field#2  PAGE     : 4 bytes  (page_no)
 field#3  METADATA : 4 + 6*n  bytes
            +0  COUNTER : 2 bytes  ★ 同一 (space,page) 内的操作序号
            +2  TYPE    : 1 byte   0=INSERT / 1=DELETE_MARK / 2=DELETE
            +3  FLAGS   : 1 byte   bit0 = IBUF_REC_COMPACT
            +4         : 每用户字段 6 字节的类型信息
 field#4.. USER[]  : 二级索引 entry 的真实字段值
```

```c
constexpr uint32_t IBUF_REC_FIELD_SPACE = 0;
constexpr uint32_t IBUF_REC_FIELD_MARKER = 1;
constexpr uint32_t IBUF_REC_FIELD_PAGE = 2;
constexpr uint32_t IBUF_REC_FIELD_METADATA = 3;
constexpr uint32_t IBUF_REC_FIELD_USER = 4;

constexpr uint32_t IBUF_REC_INFO_SIZE = 4;
constexpr uint32_t IBUF_REC_OFFSET_COUNTER = 0;
constexpr uint32_t IBUF_REC_OFFSET_TYPE = 2;
constexpr uint32_t IBUF_REC_OFFSET_FLAGS = 3;
constexpr uint32_t IBUF_REC_COMPACT = 0x1;
```

操作类型（`include/ibuf0ibuf.h`）：

```c
typedef enum {
  IBUF_OP_INSERT = 0,
  IBUF_OP_DELETE_MARK = 1,
  IBUF_OP_DELETE = 2,
  IBUF_OP_COUNT = 3
} ibuf_op_t;
```

> 注释警告：**"DO NOT CHANGE THE VALUES OF THESE, THEY ARE STORED ON DISK."**

#### 2.2 ★ COUNTER 为什么存在

源码注释（`ibuf0ibuf.cc`）：

> *"2 bytes: Counter field, used to sort records within a (space id, page no) in the order they were added. This is needed so that for example the sequence of operations "INSERT x, DEL MARK x, INSERT x" is handled correctly."*

即：**保证同一页上多个操作在 merge 时按加入顺序应用**。counter = 该 `(space, page_no)` 最后一条记录的 counter + 1（`ibuf_get_entry_counter_func`）。

#### 2.3 解析与空间估算

- `ibuf_rec_get_info_func`用 `len % 6` 判断格式：0 = REDUNDANT 老格式，1 = COMPACT 老格式，4 = 5.5+ 新格式。
- `ibuf_rec_get_volume_func`：`DELETE_MARK`/`DELETE` 返回 **0**（"Delete-marking a record doesn't take any additional space"）；INSERT 返回 `rec_get_converted_size + page_dir_calc_reserved_space(1)`。

#### 2.4 ★ 那 6 字节"类型信息"是什么：`dtype_t` 的裁剪序列化

`METADATA` 里"每用户字段 6 字节类型信息"不是随手写的——它是 **`dtype_t` 的裁剪序列化**，由 `data0type.ic` 的这对函数读写（ibuf 侧：写 `ibuf0ibuf.cc:1668`，读 `:1229/1385/1437`）：

```c
/* data0type.ic:198 —— 写：把 dtype 压成 6 字节 */
static inline void dtype_new_store_for_order_and_null_size(
    byte *buf, const dtype_t *type, ulint prefix_len) {
  static_assert(DATA_NEW_ORDER_NULL_TYPE_BUF_SIZE == 6, ...);
  buf[0] = (byte)(type->mtype & 0xFFUL);
  if (type->prtype & DATA_BINARY_TYPE)  buf[0] |= 128;   // ★ bit7 = binary
  /* In versions < 4.1.2 we had: DATA_NONLATIN1 → buf[0] |= 64; */
  if (type->prtype & DATA_MULTI_VALUE)  buf[0] |= 64;    // ★ bit6 改给"多值列"了
  buf[1] = (byte)(type->prtype & 0xFFUL);
  len = prefix_len ? prefix_len : type->len;
  mach_write_to_2(buf + 2, len & 0xFFFFUL);              // 2 字节 len
  mach_write_to_2(buf + 4, dtype_get_charset_coll(type->prtype));   // 2 字节 charset-collation
}
```

```c
/* data0type.ic:269 —— 读：重建一个"部分 dtype" */
type->mtype  = buf[0] & 63;                    // 低 6 位是 mtype
type->prtype = buf[1];                         // ★ prtype 只存了低 8 位
if (buf[0] & 128) type->prtype |= DATA_BINARY_TYPE;
if (buf[0] & 64)  type->prtype |= DATA_MULTI_VALUE;
/* len 与 charset_coll 各自 2 字节（小端） */
```

**6 字节的完整布局**（`DATA_NEW_ORDER_NULL_TYPE_BUF_SIZE == 6`，有 `static_assert` 钉死）：

| 偏移 | 内容 |
|---|---|
| 0 | `mtype`（低 6 位）｜`binary`（bit7）｜`multi_value`（bit6） |
| 1 | `prtype` 的**低 8 位**（只留必要标志） |
| 2–3 | `len`（小端 2 字节；**有前缀时用 `prefix_len` 替换**） |
| 4–5 | `charset_coll`（小端 2 字节） |

★ **被裁掉的是什么**：`mbminmaxlen`（5 位）**整个没存**。**原因**：merge 时只需要两类信息——"**字母序比较**"（`mtype` / `prtype` 标志位 / `charset_coll`）与"**NULL 的存储尺寸**"（`len`，供 `dtype_get_sql_null_size`）。**`mbminmaxlen` 只服务"字符对齐/补空格"，merge 用不到**——这正是"dtype 能被压进 6 字节"的前提（对照 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md) 的 dtype 位域打包一节）。

★ `prefix_len` 参数的设计：merge 时字段可能只按**前缀**参与（`ibuf0ibuf.cc:1668` 传 `ti->prefix_len`），所以写入的 `len` 可被前缀长度替换——**读回来的是"当初写进去的那个长度语义"**。

---

### 插入路径 `ibuf_insert`

#### 3.1 入口条件

`btr0cur.cc`：只有 `buf_page_get_gen(..., Page_fetch::IF_IN_POOL, ...)` 返回 **nullptr**（页不在 BP）时才尝试 ibuf。这本身就是第一条否决条件。

#### 3.2 `ibuf_insert` 的决策矩阵

⚠️ **勘误**：`ibuf_insert()` 返回 **`bool`** 不是 `dberr_t`；签名里**没有** `mode` / `entry_size` / `no_counter`（那三个是 `ibuf_insert_low()` 的形参）。

```cpp
/** Buffer an operation in the insert/delete buffer, instead of doing it
directly to the disk page, if this is possible. Does not do it if the index
is clustered or unique.
@param[in]      op              operation type
@param[in]      entry           index entry to insert
@param[in,out]  index           index where to insert
@param[in]      page_id         page id where to insert
@param[in]      page_size       page size
@param[in,out]  thr             query thread
@return true if success */
bool ibuf_insert(ibuf_op_t op, const dtuple_t *entry, dict_index_t *index,
                 const page_id_t &page_id, const page_size_t &page_size,
                 que_thr_t *thr) {
  /* Read the settable global variable ibuf_use only once in
  this function, so that we will have a consistent view of it. */
  assert(innodb_change_buffering <= IBUF_USE_ALL);
  ibuf_use_t use = static_cast<ibuf_use_t>(innodb_change_buffering);

  ut_a(!index->is_clustered());

  auto no_counter = use <= IBUF_USE_INSERT;
  //  ★ none/insert 两种模式写"老格式"(无 counter)，一旦开启 delete 缓冲就必须写 counter

  switch (op) {
    case IBUF_OP_INSERT:
      switch (use) {
        case IBUF_USE_NONE:
        case IBUF_USE_DELETE:
        case IBUF_USE_DELETE_MARK:
          return false;                       // ★ 不匹配 → 转同步
        case IBUF_USE_INSERT:
        case IBUF_USE_INSERT_DELETE_MARK:
        case IBUF_USE_ALL:
          goto check_watch;
      }
      break;
    case IBUF_OP_DELETE_MARK:
      switch (use) {
        case IBUF_USE_NONE:
        case IBUF_USE_INSERT:
          return false;
        case IBUF_USE_DELETE_MARK:
        case IBUF_USE_DELETE:
        case IBUF_USE_INSERT_DELETE_MARK:
        case IBUF_USE_ALL:
          ut_ad(!no_counter);
          goto check_watch;
      }
      break;
    case IBUF_OP_DELETE:
      switch (use) {
        case IBUF_USE_NONE:
        case IBUF_USE_INSERT:
        case IBUF_USE_INSERT_DELETE_MARK:
          return false;
        case IBUF_USE_DELETE_MARK:
        case IBUF_USE_DELETE:
        case IBUF_USE_ALL:
          ut_ad(!no_counter);
          goto skip_watch;                    // ★★ DELETE 跳过 watch 检查
      }
      break;
    case IBUF_OP_COUNT:
      break;
  }
  ut_error;
```

**`ibuf_op_t` 只有 3 种 op**（`include/ibuf0ibuf.h:52`）：

```cpp
/* Possible operations buffered in the insert/whatever buffer. See
ibuf_insert(). DO NOT CHANGE THE VALUES OF THESE, THEY ARE STORED ON DISK. */
typedef enum {
  IBUF_OP_INSERT = 0,
  IBUF_OP_DELETE_MARK = 1,
  IBUF_OP_DELETE = 2,

  /* Number of different operation types. */
  IBUF_OP_COUNT = 3
} ibuf_op_t;
```

> ★★ **为什么没有 UPDATE**：InnoDB 的二级索引 UPDATE 在 row 层就被拆成 **DELETE_MARK 旧记录 + INSERT 新记录** 两条独立操作，不存在原子 UPDATE。purge 线程真正物理删除时才是 `IBUF_OP_DELETE`。三种 op 已覆盖二级索引的全部变更形态。
>
> ★ op 值**持久化在磁盘**（ibuf 记录 metadata 字段的 `IBUF_REC_OFFSET_TYPE` 字节），所以注释警告"DO NOT CHANGE THE VALUES"。

**三个调用点**（`btr0cur.cc:1091`）——返回 true 就跳过同步页修改：

```cpp
    switch (btr_op) {
      case BTR_INSERT_OP:
      case BTR_INSERT_IGNORE_UNIQUE_OP:
        if (ibuf_insert(IBUF_OP_INSERT, tuple, index, page_id, page_size,
                        cursor->thr)) {
          cursor->flag = BTR_CUR_INSERT_TO_IBUF;
          goto func_exit;
        }
        break;

      case BTR_DELMARK_OP:
        if (ibuf_insert(IBUF_OP_DELETE_MARK, tuple, index, page_id, page_size,
                        cursor->thr)) {
          cursor->flag = BTR_CUR_DEL_MARK_IBUF;
          goto func_exit;
        }
        break;

      case BTR_DELETE_OP:
        ut_ad(fetch == Page_fetch::IF_IN_POOL_OR_WATCH);
        if (!row_purge_poss_sec(cursor->purge_node, index, tuple)) {
          cursor->flag = BTR_CUR_DELETE_REF;      // 还不能 purge（有活跃版本引用）
        } else if (ibuf_insert(IBUF_OP_DELETE, tuple, index, page_id, page_size,
                               cursor->thr)) {
          cursor->flag = BTR_CUR_DELETE_IBUF;     // ★ purge 操作被缓冲
        }
```

**两级 `ibuf_insert_low()` 重试**：

```cpp
  err = ibuf_insert_low(BTR_MODIFY_PREV, op, no_counter, entry, entry_size,
                        index, page_id, page_size, thr);
  if (err == DB_FAIL) {
    err = ibuf_insert_low(BTR_MODIFY_TREE | BTR_LATCH_FOR_INSERT, op, no_counter,
                          entry, entry_size, index, page_id, page_size, thr);
  }

  if (err == DB_SUCCESS) {
    return true;
  } else {
    ut_a(err == DB_STRONG_FAIL || err == DB_TOO_BIG_RECORD);
    return false;                    // ★ 一律翻译成 false → 转同步路径
  }
```

> ★ 乐观 `BTR_MODIFY_PREV`（只锁 leaf + prev）→ 空间不足返回 `DB_FAIL` → 降级悲观 `BTR_MODIFY_TREE | BTR_LATCH_FOR_INSERT`（可能引起树结构变更/分配新页）。
>
> ★ `ibuf_insert_low()` 只允许返回 `DB_SUCCESS` / `DB_FAIL` / `DB_STRONG_FAIL` / `DB_TOO_BIG_RECORD`。

```c
bool ibuf_insert(ibuf_op_t op, const dtuple_t *entry, dict_index_t *index,
                 const page_id_t &page_id, const page_size_t &page_size,
                 que_thr_t *thr) {
  ut_a(!index->is_clustered);
  auto no_counter = use <= IBUF_USE_INSERT;

  switch (op) {
    case IBUF_OP_INSERT:
      switch (use) {
        case IBUF_USE_NONE:
        case IBUF_USE_DELETE:
        case IBUF_USE_DELETE_MARK:
          return false;
        case IBUF_USE_INSERT:
        case IBUF_USE_INSERT_DELETE_MARK:
        case IBUF_USE_ALL:
          goto check_watch;
      }
      break;
    case IBUF_OP_DELETE_MARK:  /* deletes / changes / purges / all */
      ...
    case IBUF_OP_DELETE:       /* purges(deletes) / all */
      ...
  }
```

| `innodb_change_buffering` | INSERT | DELETE_MARK | DELETE(purge) |
|---|---|---|---|
| `none` | ✗ | ✗ | ✗ |
| `inserts` | ✓ | ✗ | ✗ |
| `deletes` | ✗ | ✓ | ✓ |
| `changes` | ✓ | ✓ | ✗ |
| `purges` | ✗ | ✓ | ✓ |
| `all`（默认） | ✓ | ✓ | ✓ |

> **注意枚举名与 SQL 名的对应**：`IBUF_USE_DELETE_MARK` 对应 SQL 的 `deletes`，`IBUF_USE_DELETE` 对应 `purges`——**名字容易串**。

**其他要点**：

- `no_counter = use <= IBUF_USE_INSERT`：只有 `inserts` 模式用"无 counter 的老格式"（老格式不支持 delete buffering）。
- **`check_watch`**：若该页已设 buffer pool watch（purge 正在处理）或页已被读入 BP → **拒绝缓存**。
- **entry 太大不缓存**：`entry_size >= 空页可用空间/2`（约 8 KB）→ 放弃（否则 merge 时装不下）。
- **两级重试**：先 `BTR_MODIFY_PREV`（乐观），返回 `DB_FAIL` 后用 `BTR_MODIFY_TREE`（悲观，可能引 ibuf 树分裂）。

#### 3.3 ★ 不缓存的条件：五道闸，不是一张平铺清单

**闸门 0（粗筛，在 `btr_cur_search_to_nth_level` 里）**：`ibuf_should_try()` 的 9 个 AND 条件（见「为什么唯一二级索引不能被缓存」一节）。

**闸门 1（op × `innodb_change_buffering` 矩阵）**：见 3.2，不匹配直接 `return false`。

**闸门 2（buffer pool watch，仅 INSERT / DELETE_MARK）**：

```cpp
check_watch:
  /* If a thread attempts to buffer an insert on a page while a
  purge is in progress on the same page, the purge must not be
  buffered, because it could remove a record that was
  re-inserted later.  For simplicity, we block the buffering of
  all operations on a page that has a purge pending.

  We do not check this in the IBUF_OP_DELETE case, because that
  would always trigger the buffer pool watch during purge and
  thus prevent the buffering of delete operations.  We assume
  that the issuer of IBUF_OP_DELETE has called
  buf_pool_watch_set(space, page_no). */

  {
    buf_pool_t *buf_pool = buf_pool_get(page_id);
    buf_page_t *bpage = buf_page_get_also_watch(buf_pool, page_id);

    if (bpage != nullptr) {
      /* A buffer pool watch has been set or the
      page has been read into the buffer pool.
      Do not buffer the request. ... */
      return false;
    }
  }
```

**闸门 3（entry 太大）**：

```cpp
skip_watch:
  entry_size = rec_get_converted_size(index, entry);

  if (entry_size >=
      page_get_free_space_of_empty(dict_table_is_comp(index->table)) / 2) {
    return false;
  }
```

> ★ 16KB 页：条目 > 约 8KB 就不进 ibuf。

**闸门 A/B（`ibuf_insert_low()` 开头，脏读）**：

```cpp
  /* Perform dirty reads of ibuf->size and ibuf->max_size, to
  reduce ibuf_mutex contention. Given that ibuf->max_size and
  ibuf->size fit in a machine word, this should be OK; at worst
  we are doing some excessive ibuf_contract() or occasionally
  skipping an ibuf_contract(). */
  if (ibuf->max_size == 0) {
    return (DB_STRONG_FAIL);                       // ★ 闸门 A：change_buffer_max_size=0
  }

  if (ibuf->size >= ibuf->max_size + IBUF_CONTRACT_DO_NOT_INSERT) {
    /* Insert buffer is now too big, contract it but do not try
    to insert */
    ibuf_contract(true);                           // ★ 闸门 B：三级自我保护第 3 级
    return (DB_STRONG_FAIL);
  }
```

**闸门 C（free list 不足，仅悲观模式）**：

```cpp
  if (BTR_LATCH_MODE_WITHOUT_INTENTION(mode) == BTR_MODIFY_TREE) {
    for (;;) {
      mutex_enter(&ibuf_pessimistic_insert_mutex);
      mutex_enter(&ibuf_mutex);
      if (UNIV_LIKELY(ibuf_data_enough_free_for_insert())) break;
      mutex_exit(&ibuf_mutex);
      mutex_exit(&ibuf_pessimistic_insert_mutex);
      if (!ibuf_add_free_page()) {
        mem_heap_free(heap);
        return (DB_STRONG_FAIL);
      }
    }
  }
```

```cpp
// ibuf0ibuf.cc:1749 —— 判据
static inline bool ibuf_data_enough_free_for_insert(void) {
  /* We want a big margin of free pages, because a B-tree can sometimes
  grow in size also if records are deleted from it, as the node pointers
  can change, and we must make sure that we are able to delete the
  inserts buffered for pages that we read to the buffer pool, without
  any risk of running out of free space in the insert buffer. */
  return (ibuf->free_list_len >= (ibuf->size / 2) + 3 * ibuf->height);
}
```

**闸门 D（仅 `IBUF_OP_DELETE`）**：

```cpp
  if (op == IBUF_OP_DELETE &&
      (min_n_recs < 2 || buf_pool_watch_occurred(page_id))) {
    /* The page could become empty after the record is
    deleted, or the page has been read in to the buffer
    pool.  Refuse to buffer the operation. */
    ...
  fail_exit:
    err = DB_STRONG_FAIL;
    goto func_exit;
  }
```

> ★ `min_n_recs < 2`：该页缓冲的记录不足 2 条 ⇒ 删完可能空页 ⇒ 拒绝（InnoDB 不允许空 B-tree 非 root 页）。

**闸门 E（页已进入 BP / 有显式锁）**：

```cpp
  /* We check if the index page is suitable for buffered entries */
  if (buf_page_peek(page_id) || lock_rec_expl_exist_on_page(page_id)) {
    ibuf_mtr_commit(&bitmap_mtr);
    goto fail_exit;
  }
```

**闸门 F（仅 `IBUF_OP_INSERT`，空间不足 → 触发 merge 而非插入）**：

```cpp
  if (op == IBUF_OP_INSERT) {
    ulint bits = ibuf_bitmap_page_get_bits(bitmap_page, page_id, page_size,
                                           IBUF_BITMAP_FREE, &bitmap_mtr);

    if (buffered + entry_size + page_dir_calc_reserved_space(1) >
        ibuf_index_page_calc_free_from_bits(page_size, bits)) {
      /* Release the bitmap page latch early. */
      ibuf_mtr_commit(&bitmap_mtr);

      /* It may not fit */
      do_merge = true;

      ibuf_get_merge_page_nos(false, pcur.get_rec(), &mtr, space_ids, page_nos,
                              &n_stored);
      goto fail_exit;
    }
  }
```

> ★★ 闸门 F 是**唯一会"顺带把邻居页也读进来 merge"**的一档：`ibuf_get_merge_page_nos()` 收集页号，函数末尾 `buf_read_ibuf_merge_pages(false, ...)` 异步读这些页。
>
> ★ `buffered` 来自 `ibuf_get_volume_buffered()`——它扫 ibuf 树里该 (space,page_no) 的所有记录求体积和；扫到再前一页会违反 latching order 时，直接返回 `UNIV_PAGE_SIZE` 作为上界（**宁可高估占用**）。

**counter 补丁（老格式保护）**：

```cpp
  if (!no_counter) {
    /* Patch correct counter value to the entry to
    insert. This can change the insert position, which can
    result in the need to abort in some cases. */
    ulint counter = ibuf_get_entry_counter(
        page_id.space(), page_id.page_no(), pcur.get_rec(), &mtr,
        pcur.get_btr_cur()->low_match < IBUF_REC_FIELD_METADATA);
    if (counter == ULINT_UNDEFINED) {
      ibuf_mtr_commit(&bitmap_mtr);
      goto fail_exit;                    // ★ 遇到老格式记录 → 拒绝插入
    }
    dfield_t *field = dtuple_get_nth_field(ibuf_entry, IBUF_REC_FIELD_METADATA);
    mach_write_to_2((byte *)dfield_get_data(field) + IBUF_REC_OFFSET_COUNTER,
                    counter);
  }
```

**游标位置的选择**（`ibuf_insert_low` 段落 3）：

```cpp
  ibuf_mtr_start(&mtr);
  pcur.open(ibuf->index, 0, ibuf_entry, PAGE_CUR_LE, mode, &mtr,
            UT_LOCATION_HERE);
```

> ★★ 用 **counter = 0xFFFF** 构造的 `ibuf_entry` 作为搜索键，配 `PAGE_CUR_LE`（小于等于）⇒ 定位到 **(space, page_no) 分组内的最后一条记录**，从而能读出最大 counter N，插入前再改成 N+1。这就是"如何选 B-tree 游标位置"的答案。

**置 BUFFERED bit + 实际插入**：

```cpp
  /* Set the bitmap bit denoting that the insert buffer contains
  buffered entries for this index page, if the bit is not set yet */
  old_bit_value = ibuf_bitmap_page_get_bits(bitmap_page, page_id, page_size,
                                            IBUF_BITMAP_BUFFERED, &bitmap_mtr);
  if (!old_bit_value) {
    ibuf_bitmap_page_set_bits(bitmap_page, page_id, page_size,
                              IBUF_BITMAP_BUFFERED, true, &bitmap_mtr);
  }
  ibuf_mtr_commit(&bitmap_mtr);        // ★★ bitmap 用独立 mtr 先提交

  cursor = pcur.get_btr_cur();
  if (mode == BTR_MODIFY_PREV) {
    err = btr_cur_optimistic_insert(BTR_NO_LOCKING_FLAG, cursor, ...);
    if (block->page.id.page_no() == FSP_IBUF_TREE_ROOT_PAGE_NO) {
      ibuf->empty = page_is_empty(root);
    }
  } else {
    root = ibuf_tree_root_get(&mtr);   // ★ 悲观插入前先 sx-latch root
    err = btr_cur_optimistic_insert(BTR_NO_LOCKING_FLAG | BTR_NO_UNDO_LOG_FLAG,
                                    cursor, ...);
    if (err == DB_FAIL) {
      err = btr_cur_pessimistic_insert(BTR_NO_LOCKING_FLAG | BTR_NO_UNDO_LOG_FLAG,
                                       cursor, ...);
    }
    ibuf_size_update(root);
    ibuf->empty = page_is_empty(root);
  }
  ...
  if (err == DB_SUCCESS &&
      BTR_LATCH_MODE_WITHOUT_INTENTION(mode) == BTR_MODIFY_TREE) {
    ibuf_contract_after_insert(entry_size);    // ★ 只有悲观插入才可能让树长大
  }
  if (do_merge) {
    buf_read_ibuf_merge_pages(false, space_ids, page_nos, n_stored);
  }
```

> ★ ibuf 树上的插入一律带 **`BTR_NO_LOCKING_FLAG | BTR_NO_UNDO_LOG_FLAG`**（不加锁、不写 undo）——ibuf 是系统内部结构。

否决条件分布在 `ibuf_insert` → `ibuf_insert_low` 的**五个检查阶段**——越往后越接近真正写入，判定的东西也越从"能不能用"变成"这一刻行不行"：

| 阶段 | # | 条件 | 判定点 |
|---|---|---|---|
| **A 全局开关**（ibuf 整体能不能用） | 1 | `innodb_change_buffering = none` 或 `ibuf->max_size == 0` | `ibuf_insert_low`：`ibuf->max_size == 0` |
| | 5 | `innodb_force_recovery >= 4`（崩溃恢复模式不写 ibuf） | `ibuf_insert_low` |
| **B 索引类型**（这个索引能不能缓存） | 2 | 聚簇索引——ibuf 只服务二级索引（`ut_a` 断言级） | `ibuf0ibuf.ic` |
| | 3 | 空间索引 / 含降序列 / quiesce 中的表 / DD 表空间 | `ibuf0ibuf.ic` |
| | 4 | **唯一二级索引 + INSERT**（唯一性必须立即校验，见「为什么只缓存非唯一二级索引」） | `ibuf0ibuf.ic` |
| | 6 | op 与 `change_buffering` 选项组合不允许（如只开 inserts 却来一个 delete-mark） | `ibuf0ibuf.cc` |
| **C 页级现状**（这一刻这张页能不能缓存） | 7 | **页已在 BP，或已设 buffer pool watch**（purge 正在处理） | `ibuf_insert` 的 `check_watch` 分支 |
| | 11 | **页在 BP 中，或页上有显式记录锁** | `ibuf_insert_low`：`buf_page_peek(page_id) \|\| lock_rec_expl_exist_on_page(page_id)` |
| | 13 | DELETE 且 `min_n_recs < 2` 或 watch 已触发 | `ibuf_insert_low`：`min_n_recs < 2 \|\| buf_pool_watch_occurred(page_id)` |
| **D 容量与空间** | 8 | `entry_size >= 空页可用空间/2`（约 8KB）——否则 merge 时装不进目标页 | `ibuf_insert_low` |
| | 9 | `ibuf->size >= max_size + IBUF_CONTRACT_DO_NOT_INSERT`（+10 是留出的收缩余量） | `ibuf_insert_low` |
| | 10 | 悲观插入前 `ibuf_add_free_page()` 失败（ibuf 树要分裂却申请不到空闲页） | `ibuf_insert_low` |
| | 12 | INSERT 且"已缓存量 + 本条 + 目录槽 > FREE 位代表的空间"（与 `page_dir_calc_reserved_space(1)` 比较） | `ibuf_insert_low` |
| **E 格式兼容** | 14 | counter 无法计算（遇到"无 counter 的老格式"记录，见 3.2） | `ibuf_insert_low` |

★ **第 7 条与第 11 条不是重复条目，而是两个检查点**：第 7 条在 `ibuf_insert` 的 `check_watch` 分支（bitmap 检查之后、构造 entry 之前），第 11 条在 `ibuf_insert_low` 写入前的最终检查。两者之间隔了 entry 构造与 ibuf 树搜索，**期间目标页可能已被并发读入 BP**，所以必须再查一次。"页在 BP"这个条件在两处出现是**时序造成的必要重查**，不是清单凑数。

> **第 11 条是"写完立刻读"负收益的根因之一**：`buf_page_peek(page_id)` 为真的页一律不缓存——刚写过（页还热在 BP）又改，本来也不该走 ibuf（见「收益与代价」）。

```c
  if (buf_page_peek(page_id) || lock_rec_expl_exist_on_page(page_id)) {
    ibuf_mtr_commit(&bitmap_mtr);
    goto fail_exit;
  }
```

#### 3.4 ibuf 大小限制与三级自我保护

⚠️ **勘误**：
- **不存在 `IBUF_MAX_SIZE` 编译期常量**——`max_size` 是运行时按 `BP 页数 × pct / 100` 算出来的 `ibuf->max_size`（页数）
- **不存在 `IBUF_THRESHOLD` / `ibuf->threshold`**——常被误称的其实是 **`IBUF_MERGE_THRESHOLD = 4`**
- **不存在 `ibuf_contract_ext()`**——收缩就是 `ibuf_contract(bool sync)` 一个函数，靠 `sync` 区分

**三个阈值常量**（`ibuf0ibuf.cc:307`）：

```cpp
/** If the combined size of the ibuf trees exceeds ibuf->max_size by this
many pages, we start to contract it in connection to inserts there, using
non-synchronous contract */
const ulint IBUF_CONTRACT_ON_INSERT_NON_SYNC = 0;

/** If the combined size of the ibuf trees exceeds ibuf->max_size by this
many pages, we start to contract it in connection to inserts there, using
synchronous contract */
const ulint IBUF_CONTRACT_ON_INSERT_SYNC = 5;

/** If the combined size of the ibuf trees exceeds ibuf->max_size by
this many pages, we start to contract it synchronous contract, but do
not insert */
const ulint IBUF_CONTRACT_DO_NOT_INSERT = 10;
```

**第 1 / 2 级：`ibuf_contract_after_insert()`**（`ibuf0ibuf.cc:2448`）：

```cpp
static inline void ibuf_contract_after_insert(ulint entry_size) {
  /* Perform dirty reads of ibuf->size and ibuf->max_size, to
  reduce ibuf_mutex contention. ... */
  auto size = ibuf->size;
  auto max_size = ibuf->max_size;

  if (size < max_size + IBUF_CONTRACT_ON_INSERT_NON_SYNC) {
    return;                                    // ★ 第 0 档：什么都不做
  }

  auto sync = (size >= max_size + IBUF_CONTRACT_ON_INSERT_SYNC);
  //          ★ size >= max_size+5 → 第 2 级同步收缩；否则第 1 级异步收缩

  /* Contract at least entry_size many bytes */
  ulint sum_sizes = 0;
  size = 1;

  do {
    size = ibuf_contract(sync);
    sum_sizes += size;
  } while (size > 0 && sum_sizes < entry_size);
  //  ★★ 循环条件：至少收缩掉本次插入的字节量，做到"收支平衡"
}
```

**第 3 级**（`ibuf0ibuf.cc:3019`，见 3.3 闸门 B）：`size >= max_size + 10` ⇒ 同步收缩 + **本次不插入**。

**三级总结**：

| 档 | 判据 | 行为 |
|---|---|---|
| 0 | `size < max_size + 0` | 不收缩 |
| 1 | `max_size + 0 ≤ size < max_size + 5` | **异步**收缩（`sync=false`） |
| 2 | `size ≥ max_size + 5` | **同步**收缩（`sync=true`，调用线程等读完成） |
| 3 | `size ≥ max_size + 10` | 同步收缩 **且本次插入失败**（`DB_STRONG_FAIL`） |

```cpp
// ibuf0ibuf.cc:2380 —— 收缩的本质
static ulint ibuf_contract(bool sync) {
  ulint n_pages;
  return (ibuf_merge_pages(&n_pages, sync));
  //  ★★ 收缩 = 把二级索引页读进 BP → 触发 ibuf_merge_or_delete_for_page
}
```

**`max_size` 的绑定链**：

```cpp
// ibuf0ibuf.h:47 —— 默认 25%
constexpr uint32_t CHANGE_BUFFER_DEFAULT_SIZE = 25;

// ha_innodb.cc:22969 —— 范围 [0, 50]
static MYSQL_SYSVAR_UINT(
    change_buffer_max_size, srv_change_buffer_max_size, PLUGIN_VAR_RQCMDARG,
    "Maximum on-disk size of change buffer in terms of percentage"
    " of the buffer pool.",
    nullptr, innodb_change_buffer_max_size_update, CHANGE_BUFFER_DEFAULT_SIZE,
    0, 50, 0);

// ibuf0ibuf.cc:530
void ibuf_max_size_update(ulint new_val) {
  ulint new_size =
      ((buf_pool_get_curr_size() / UNIV_PAGE_SIZE) * new_val) / 100;
  mutex_enter(&ibuf_mutex);
  ibuf->max_size = new_size;      // ★★ 单位是"页数"，不是百分比
  mutex_exit(&ibuf_mutex);
}
```

> ★ 四个调用点：启动（`ha_innodb.cc:5599`）、sysvar 更新（`:20739`）、**BP resize**（`buf0buf.cc:2592`）、以及 `ibuf_init_at_db_start()` 里的默认值初始化。

**后台自适应 merge**（`ibuf_merge_in_background()`，`ibuf0ibuf.cc:2391`）：

```cpp
  if (full) {
    n_pages = PCT_IO(100);            // ★ full=true：一批 100% io_capacity
  } else {
    n_pages = PCT_IO(5);              // ★ 后台常规：5% io_capacity
    mutex_enter(&ibuf_mutex);
    /* If the ibuf->size is more than half the max_size
    then we make more aggressive contraction. */
    if (ibuf->size > ibuf->max_size / 2) {
      ulint diff = ibuf->size - ibuf->max_size / 2;
      diff = std::min(diff, ibuf->max_size);
      n_pages += PCT_IO((diff * 100) / (ibuf->max_size + 1));
      //  ★★ 自适应：超过 50% 就线性加压，最多到 PCT_IO(100)
    }
    mutex_exit(&ibuf_mutex);
  }
```

> ★ 主线程 active 循环调 `ibuf_merge_in_background(false)`，idle 循环调 `(true)`，**慢关机**（`srv_fast_shutdown=0`）也调 `(true)` 直到 ibuf 清空（`srv0srv.cc:2501`）。默认 `innodb_fast_shutdown=1` ⇒ **关机不做 ibuf merge**，留给下次崩溃恢复。
>
> ⚠️ **不存在 `ibuf_merge_full()`**（真名 `ibuf_merge_in_background(true)`）。
>
> ⚠️ **不存在 `srv_ibuf_merge_in_background` 系统变量**——只有 debug-only 的 `srv_ibuf_disable_background_merge`。

**merge 批量的三个常量**（`ibuf0ibuf.cc:295`）：

```cpp
/** The area in pages from which contract looks for page numbers for merge */
const ulint IBUF_MERGE_AREA = 8;

/** Inside the merge area, pages which have at most 1 per this number less
buffered entries compared to maximum volume that can buffered for a single
page are merged along with the page whose buffer became full */
const ulint IBUF_MERGE_THRESHOLD = 4;

/** In ibuf_contract at most this number of pages is read to memory in one
batch ... */
const ulint IBUF_MAX_N_PAGES_MERGED = IBUF_MERGE_AREA;
```

**`ibuf_merge_pages()` 用"随机位置"而不是顺序扫**（`ibuf0ibuf.cc:2227`）：

```cpp
  /* Open a cursor to a randomly chosen leaf of the tree, at a random
  position within the leaf */
  bool available;
  available = pcur.set_random_position(ibuf->index, BTR_SEARCH_LEAF, &mtr,
                                       UT_LOCATION_HERE);
  //  ★★ 随机挑叶子 + 叶内随机位置：避免热页被反复 merge、避免全表顺序扫

  if (page_is_empty(pcur.get_page())) { ... return (0); }   // ibuf 空

  sum_sizes = ibuf_get_merge_page_nos(true, pcur.get_rec(), &mtr, space_ids,
                                      page_nos, n_pages);
  ibuf_mtr_commit(&mtr);
  pcur.close();

  buf_read_ibuf_merge_pages(sync, space_ids, page_nos, *n_pages);
  //  ★★ 真正的 merge 在这里：读页 → buf_page_io_complete → ibuf_merge_or_delete_for_page
```

**"顺带 merge 邻居"的三条件**（`ibuf_get_merge_page_nos_func()`，`ibuf0ibuf.cc:2108`）：

```cpp
      if (contract ||
          (prev_page_no == first_page_no && prev_space_id == first_space_id) ||
          (volume_for_page > ((IBUF_MERGE_THRESHOLD - 1) * 4 * UNIV_PAGE_SIZE /
                              IBUF_PAGE_SIZE_PER_FREE_SPACE) /
                                 IBUF_MERGE_THRESHOLD)) {
        space_ids[*n_stored] = prev_space_id;
        page_nos[*n_stored] = prev_page_no;
        (*n_stored)++;
        sum_volumes += volume_for_page;
      }
```

> ★ 阈值算出来是 `(4-1) * 4 * 16384 / 32 / 4 = 1536` 字节，即**单页缓冲上限 2048 字节的 3/4**。这就是 `IBUF_MERGE_THRESHOLD` 的实际语义。

**初始化**：

```c
  ibuf->max_size = ((buf_pool_get_curr_size / UNIV_PAGE_SIZE) *
                    CHANGE_BUFFER_DEFAULT_SIZE) / 100;   // 默认 25%
```

**阈值常量**：

```c
const ulint IBUF_CONTRACT_ON_INSERT_NON_SYNC = 0;
const ulint IBUF_CONTRACT_ON_INSERT_SYNC = 5;
const ulint IBUF_CONTRACT_DO_NOT_INSERT = 10;
```

| 条件 | 行为 |
|---|---|
| `size < max_size + 0` | 正常插入 |
| `max_size ≤ size < +5` | 插入 + **异步** contract |
| `+5 ≤ size < +10` | 插入 + **同步** contract |
| `size ≥ max_size + 10` | **同步 contract，且本次插入失败** |

`ibuf_contract_after_insert`：

```c
  auto sync = (size >= max_size + IBUF_CONTRACT_ON_INSERT_SYNC);
  ulint sum_sizes = 0;
  size = 1;
  do {
    size = ibuf_contract(sync);
    sum_sizes += size;
  } while (size > 0 && sum_sizes < entry_size);
```

**第二种自我保护**：当发现某页的 FREE 位装不下新操作时，不是简单放弃，而是 `do_merge = true` + `buf_read_ibuf_merge_pages` —— **把附近页一起读进来触发 merge，腾出空间**。

#### 3.5 ibuf 自身的 redo（★ 常见误解）

⚠️ **勘误（最重要的一条）**：**`MLOG_IBUF`、`mlog_ibuf_open()`、`mlog_ibuf_close()` 在 8.0.39 全部不存在**（全仓 0 命中）。那是 5.x 的遗留机制——**5.x 里 `ibuf_mtr_start()` 会在 redo log 里预留一个 ibuf bitmap 槽位，8.0 已彻底删除**。

**8.0 的 `ibuf_mtr_start()` 只是打一个 boolean 标记**（`include/ibuf0ibuf.ic:46`）：

```cpp
/** Starts an insert buffer mini-transaction. */
static inline void ibuf_mtr_start(mtr_t *mtr) {
  mtr_start(mtr);
  mtr->enter_ibuf();        // ★★ 就这一句：m_inside_ibuf = true
}

/** Commits an insert buffer mini-transaction. */
static inline void ibuf_mtr_commit(mtr_t *mtr) {
  ut_ad(mtr->is_inside_ibuf());
  ut_d(mtr->exit_ibuf());
  mtr_commit(mtr);
}
```

```cpp
// mtr0mtr.h:500
  /** Note that we are inside the change buffer code. */
  void enter_ibuf() { m_impl.m_inside_ibuf = true; }
  /** Note that we have exited from the change buffer code. */
  void exit_ibuf() { m_impl.m_inside_ibuf = false; }
  /** @return true if we are inside the change buffer code */
  [[nodiscard]] bool is_inside_ibuf() const { return (m_impl.m_inside_ibuf); }
```

**唯一的 ibuf 专属 redo 类型**（`include/mtr0types.h:129`）：

```cpp
  /** initialize an ibuf bitmap page */
  MLOG_IBUF_BITMAP_INIT = 27,
```

```cpp
// ibuf0ibuf.cc:562 —— 只在新 bitmap 页初始化时写一条
  mlog_write_initial_log_record(page, MLOG_IBUF_BITMAP_INIT, mtr);
```

**ibuf 树本身的 redo** —— ibuf 树就是 space 0 里的一棵普通 B-tree（`ibuf->index`）：

```cpp
  ibuf->index =
      dict_mem_index_create("innodb_change_buffer", "CLUST_IND", IBUF_SPACE_ID,
                            DICT_CLUSTERED | DICT_IBUF, 1);
  ibuf->index->id = DICT_IBUF_ID_MIN + IBUF_SPACE_ID;
  ibuf->index->table = dict_mem_table_create("innodb_change_buffer",
                                             IBUF_SPACE_ID, 1, 0, 0, 0, 0);
  ibuf->index->n_uniq = REC_MAX_N_FIELDS;
  rw_lock_create(index_tree_rw_lock_key, &ibuf->index->lock,
                 LATCH_ID_IBUF_INDEX_TREE);
  ibuf->index->search_info = btr_search_info_create(ibuf->index->heap);
  ibuf->index->page = FSP_IBUF_TREE_ROOT_PAGE_NO;
```

> ★★ ibuf 树的 redo 与普通二级索引插入**完全相同**（`MLOG_COMP_REC_INSERT` / `MLOG_COMP_PAGE_CREATE` / `MLOG_COMP_PAGE_REORGANIZE` 等），唯一区别是插入时带 **`BTR_NO_LOCKING_FLAG | BTR_NO_UNDO_LOG_FLAG`**（不加锁、不写 undo）。

**bitmap 页的 redo** —— 用通用 `mlog_write_ulint(..., MLOG_1BYTE, ...)`：

```cpp
// ibuf0ibuf.cc:678
  mlog_write_ulint(page + IBUF_BITMAP + byte_offset, map_byte, MLOG_1BYTE, mtr);
```

```cpp
// mtr0types.h:68
  /** one byte is written */
  MLOG_1BYTE = 1,
```

```cpp
// mtr0log.cc:256 —— MLOG_1BYTE 的全部开销：2 字节 offset + 1 字节值
void mlog_write_ulint(byte *ptr, ulint val, mlog_id_t type, mtr_t *mtr) {
  switch (type) {
    case MLOG_1BYTE:  mach_write_to_1(ptr, val); break;
    case MLOG_2BYTES: mach_write_to_2(ptr, val); break;
    case MLOG_4BYTES: mach_write_to_4(ptr, val); break;
    default: ut_error;
  }
  if (mtr == nullptr) return;
  byte *log_ptr = nullptr;
  if (!mlog_open(mtr, REDO_LOG_INITIAL_INFO_SIZE + 2 + 5, log_ptr)) return;
  log_ptr = mlog_write_initial_log_record_fast(ptr, type, log_ptr, mtr);
```

**★ 与数据页 redo 的关系：两个完全独立的 mtr**：

| mtr | 内容 | 提交顺序 |
|---|---|---|
| `bitmap_mtr` | bitmap 页的 BUFFERED bit | **先**（`ibuf0ibuf.cc:3183`） |
| `mtr` | ibuf 树插入 + `page_update_max_trx_id` | **后**（`ibuf0ibuf.cc:3252`） |

> ★★ **插入路径根本不产生用户数据页的 redo**——页本来就不在 BP，所以才缓冲。用户索引页的 redo 只在 **merge 阶段**（`ibuf_merge_or_delete_for_page` → `ibuf_insert_to_index_page` / `ibuf_set_del_mark` / `ibuf_delete`）才产生。
>
> ★ `ibuf->seg_size` / `free_list_len` / `height` / `size` 都是**纯内存**状态，启动时从 root 页的 `PAGE_BTR_IBUF_FREE_LIST` 与 `fseg_n_reserved_pages` 重算，**不写 redo**：
> ```cpp
> static void ibuf_size_update(const page_t *root) {
>   ibuf->free_list_len =
>       flst_get_len(root + PAGE_HEADER + PAGE_BTR_IBUF_FREE_LIST);
>   ibuf->height = 1 + btr_page_get_level(root);
>   /* the '1 +' is the ibuf header page */
>   ibuf->size = ibuf->seg_size - (1 + ibuf->free_list_len);
> }
> ```

**崩溃安全的关键：悲观删除前先打 delete-mark**（`ibuf_delete_rec()`，`ibuf0ibuf.cc:3835`）：

```cpp
  /* We have to resort to a pessimistic delete from ibuf.
  Delete-mark the record so that it will not be applied again,
  in case the server crashes before the pessimistic delete is
  made persistent. */
  btr_cur_set_deleted_flag_for_ibuf(pcur->get_rec(), nullptr, true, mtr);
  pcur->store_position(mtr);
  ibuf_btr_pcur_commit_specify_mtr(pcur, mtr);   // ★★ 先提交！
  ...
  btr_cur_pessimistic_delete(&err, true, pcur->get_btr_cur(), 0, false, 0, 0, 0,
                             mtr, nullptr, nullptr);
```

> ★★ 这样即便在两次 mtr 之间崩溃，也只会"少 merge 一条"，绝不会"重复 apply"。
>
> ★ merge 侧的 `IBUF_OP_DELETE` 分支也是同样手法（`ibuf0ibuf.cc:4170` 前后的 `btr_cur_set_deleted_flag_for_ibuf`）。
>
> ★ 另外 `ibuf_delete()` 里有一条硬校验：**拒绝 purge 掉页上最后一条记录、或一条未被 delete-mark 的记录**，否则报 "Unable to purge a record" 并 `ut_d(ut_error)`。

**8.0.39 中只有一种 `MLOG_IBUF_*` 类型**：

```c
  /** initialize an ibuf bitmap page */
  MLOG_IBUF_BITMAP_INIT = 27,
```

**真相**：

| 对象 | redo |
|------|------|
| **ibuf 树本身的插入/删除** | **普通 B-tree redo**（`MLOG_REC_INSERT` / `MLOG_COMP_REC_DELETE` 等）——因为 ibuf 树就是系统表空间里一棵普通 B-tree |
| **bitmap 页的位修改** | `MLOG_1BYTE` |
| **bitmap 页初始化** | `MLOG_IBUF_BITMAP_INIT` |

> ibuf 的索引对象是 `ibuf_init_at_db_start` 构造的一棵名为 `CLUST_IND` 的假索引（`ibuf0ibuf.cc`），插入用的是 `btr_cur_optimistic_insert` / `btr_cur_pessimistic_insert`，**完全复用 btr 层的 redo 生成**。

`ibuf_mtr_start` / `ibuf_mtr_commit`（`ibuf0ibuf.ic`）只多设一个 `m_inside_ibuf` 标志，**不影响是否记 redo**。

---

### ★ 合并路径

#### 4.1 8 条提前返回条件

准确说是 **8 个布尔判定条件，收敛到 5 个 `return` 语句**（`ibuf0ibuf.cc:3951`）：

```cpp
void ibuf_merge_or_delete_for_page(buf_block_t *block, const page_id_t &page_id,
                                   const page_size_t *page_size,
                                   bool update_ibuf_bitmap) {
  ut_ad(block == nullptr || page_id == block->page.id);
  ut_ad(block == nullptr || block->page.is_io_fix_read());

  if (srv_force_recovery >= SRV_FORCE_NO_IBUF_MERGE ||    // ① force_recovery >= 4 禁 merge
      trx_sys_hdr_page(page_id) ||                        // ② trx sys header 页
      fsp_is_system_temporary(page_id.space())) {         // ③ 系统临时表空间
    return;                                               // ★ return #1
  }

  if (ibuf_fixed_addr_page(page_id, univ_page_size) ||    // ④ ibuf 树 root 页 / bitmap 页
      fsp_descr_page(page_id, univ_page_size)) {          // ⑤ XDES 描述页
    return;                                               // ★ return #2
  }

  if (update_ibuf_bitmap) {
    if (ibuf_fixed_addr_page(page_id, *page_size) ||      // ⑥ 同上，用真实 page_size 再判
        fsp_descr_page(page_id, *page_size)) {
      return;                                             // ★ return #3
    }

    space = fil_space_acquire_silent(page_id.space());
    if (space == nullptr) {
      /* Do not try to read the bitmap page from space;
      just delete the ibuf records for the page */
      block = nullptr;                    // ★ 表已 DROP → 退化成"只删 ibuf 记录"
      update_ibuf_bitmap = false;
    } else {
      ibuf_mtr_start(&mtr);
      bitmap_page = ibuf_bitmap_get_map_page(page_id, *page_size,
                                             UT_LOCATION_HERE, &mtr);
      bitmap_bits = ibuf_bitmap_page_get_bits(bitmap_page, page_id, *page_size,
                                              IBUF_BITMAP_BUFFERED, &mtr);
      ibuf_mtr_commit(&mtr);

      if (!bitmap_bits) {
        /* No inserts buffered for this page */
        fil_space_release(space);
        return;                                           // ★ return #4（⑦ 最重要的快速路径）
      }
    }
  } else if (block != nullptr && (ibuf_fixed_addr_page(page_id, *page_size) ||
                                  fsp_descr_page(page_id, *page_size))) {
    return;                                               // ★ return #5（⑧）
  }
```

> ★★ **条件 ⑦ 是性能关键**：bitmap 的 `IBUF_BITMAP_BUFFERED` 位为 0 ⇒ 该页**根本没有任何缓冲操作**，连 ibuf 树都不用查。这让"读一个无缓存的页"的 merge 开销降到一次 bitmap 单字节读。

**第一次实质动作：构造搜索 tuple 并定位**：

```cpp
  heap = mem_heap_create(512, UT_LOCATION_HERE);
  search_tuple =
      ibuf_search_tuple_build(page_id.space(), page_id.page_no(), heap);
  //  ★ 只含 (space, marker=0, page_no) 三字段

  if (block != nullptr) {
    /* Move the ownership of the x-latch on the page to this OS
    thread, so that we can acquire a second x-latch on it. */
    rw_lock_x_lock_move_ownership(&(block->lock));
    page_zip = buf_block_get_page_zip(block);

    if (!fil_page_index_page_check(block->frame) ||
        !page_is_leaf(block->frame)) {
      corruption_noticed = true;
      //  ★ bitmap 说有 ibuf 记录，但这页不是索引叶子页 → 判定损坏，只丢弃不 apply
      ib::error(ER_IB_MSG_624) << "Corruption in the tablespace. Bitmap"
                                  " shows insert buffer records to page " ...;
    }
  }

loop:
  ibuf_mtr_start(&mtr);

  /* Position pcur in the insert buffer at the first entry for this
  index page */
  pcur.open_on_user_rec(ibuf->index, search_tuple, PAGE_CUR_GE, BTR_MODIFY_LEAF,
                        &mtr, UT_LOCATION_HERE);

  if (block != nullptr) {
    auto success = buf_page_get_known_nowait(
        RW_X_LATCH, block, Cache_hint::KEEP_OLD, __FILE__, __LINE__, &mtr);
    ut_a(success);
    /* This is a user page (secondary index leaf page),
    but we pretend that it is a change buffer page in
    order to obey the latching order. ... */
    buf_block_dbg_add_level(block, SYNC_IBUF_TREE_NODE);
    //  ★★ 伪装成 ibuf 页以满足 latch order 检查
  }

  if (!pcur.is_on_user_rec()) {
    ut_ad(pcur.is_after_last_in_tree(&mtr));
    goto reset_bit;                    // ibuf 树里没有该页记录 → 只清 bitmap
  }
```

**主循环：逐条 apply + 删除 ibuf 记录**：

```cpp
  for (;;) {
    rec = pcur.get_rec();

    /* Check if the entry is for this index page */
    if (ibuf_rec_get_page_no(&mtr, rec) != page_id.page_no() ||
        ibuf_rec_get_space(&mtr, rec) != page_id.space()) {
      if (block != nullptr) {
        page_header_reset_last_insert(block->frame, page_zip, &mtr);
      }
      goto reset_bit;                  // ★ 游标已越过本页的所有记录
    }

    if (corruption_noticed) {
      fputs("InnoDB: Discarding record\n ", stderr);
      rec_print_old(stderr, rec);
    } else if (block != nullptr && !rec_get_deleted_flag(rec, 0)) {
      ibuf_op_t op = ibuf_rec_get_op_type(&mtr, rec);
      max_trx_id = page_get_max_trx_id(page_align(rec));
      page_update_max_trx_id(block, page_zip, max_trx_id, &mtr);

      entry = ibuf_build_entry_from_ibuf_rec(&mtr, rec, heap, &dummy_index);
      //  ★★ ibuf 记录 → 二级索引 entry + 临时 dummy index

      switch (op) {
        case IBUF_OP_INSERT:
          ibuf_insert_to_index_page(entry, block, dummy_index, &mtr);
          break;
        case IBUF_OP_DELETE_MARK:
          ibuf_set_del_mark(entry, block, dummy_index, &mtr);
          break;
        case IBUF_OP_DELETE:           // ★ purge buffer 的落地
          ibuf_delete(entry, block, dummy_index, &mtr);
          /* Because ibuf_delete() will latch an insert buffer bitmap page,
          commit mtr before latching any further pages. ... */
          btr_cur_set_deleted_flag_for_ibuf(pcur.get_rec(), nullptr, true, &mtr);
          pcur.store_position(&mtr);
          ibuf_btr_pcur_commit_specify_mtr(&pcur, &mtr);
          ibuf_mtr_start(&mtr);
          ...
          if (!ibuf_restore_pos(page_id.space(), page_id.page_no(), search_tuple,
                                BTR_MODIFY_LEAF, &pcur, &mtr)) { ... goto loop; }
          break;
        default: ut_error;
      }
      mops[op]++;                      // ★ merged 计数
      ibuf_dummy_index_free(dummy_index);
    } else {
      dops[ibuf_rec_get_op_type(&mtr, rec)]++;   // ★ discarded 计数
    }

    /* Delete the record from ibuf */
    if (ibuf_delete_rec(page_id.space(), page_id.page_no(), &pcur, search_tuple,
                        &mtr)) {
      ut_ad(mtr.has_committed());
      goto loop;                       // ★ 悲观删除提交了 mtr → 从头再来
    } else if (pcur.is_after_last_on_page()) {
      ibuf_mtr_commit(&mtr);
      pcur.close();
      goto loop;
    }
  }
```

**收尾：清 BUFFERED 位 + 重算 FREE 位 + 计数器自增**：

```cpp
reset_bit:
  if (update_ibuf_bitmap) {
    bitmap_page = ibuf_bitmap_get_map_page(page_id, *page_size,
                                           UT_LOCATION_HERE, &mtr);
    ibuf_bitmap_page_set_bits(bitmap_page, page_id, *page_size,
                              IBUF_BITMAP_BUFFERED, false, &mtr);   // ★ 清 BUFFERED

    if (block != nullptr) {
      ulint old_bits = ibuf_bitmap_page_get_bits(bitmap_page, page_id,
                                                 *page_size, IBUF_BITMAP_FREE, &mtr);
      ulint new_bits = ibuf_index_page_calc_free(block);
      if (old_bits != new_bits) {
        ibuf_bitmap_page_set_bits(bitmap_page, page_id, *page_size,
                                  IBUF_BITMAP_FREE, new_bits, &mtr);
        //  ★★ 这里是"同一个 mtr 内"重算 FREE 位，所以可以双向（上调也安全）
      }
    }
  }

  ibuf_mtr_commit(&mtr);
  pcur.close();
  mem_heap_free(heap);

  ibuf->n_merges.fetch_add(1);                    // ★ ibuf_merges
  ibuf_add_ops(ibuf->n_merged_ops, mops);         // ★ ibuf_merges_insert / _delete_mark / _delete
  ibuf_add_ops(ibuf->n_discarded_ops, dops);      // ★ ibuf_merges_discard_*
```

**三个 merge 调用点小结**：

| 调用点 | `block` | `page_size` | `update_ibuf_bitmap` | 语义 |
|---|---|---|---|---|
| `buf0buf.cc:5765`（读完成） | 非 NULL | `&bpage->size` | true | 真正 apply |
| `buf0buf.cc:3962`（解压完成） | 非 NULL | `&m_page_size` | true | 真正 apply |
| `buf0buf.cc:5126`（BP 内新建页） | **NULL** | `&page_size` | true | **只删 ibuf 记录**（页属于已 drop 的索引） |
| `buf_read_ibuf_merge_pages()`（表已删） | **NULL** | **NULL** | **false** | 只删 ibuf 记录 |

**`buf_page_io_complete()` 里触发 merge 的 8 个条件**（`buf0buf.cc:5765`）：

```cpp
    if (uncompressed &&                                    // ① 非压缩页
        !Compression::is_compressed_page(frame) &&         // ② 不是压缩页内容
        !recv_no_ibuf_operations &&                        // ③ 崩溃恢复的 redo 应用后期禁 ibuf
        fil_page_get_type(frame) == FIL_PAGE_INDEX &&      // ④ 索引页
        page_is_leaf(frame) &&                             // ⑤ 叶子页
        !fsp_is_system_temporary(bpage->id.space()) &&     // ⑥ 非系统临时表空间
        !fsp_is_undo_tablespace(bpage->id.space()) &&      // ⑦ 非 undo 表空间
        !bpage->was_stale()) {                             // ⑧ 非 stale 页
      ibuf_merge_or_delete_for_page((buf_block_t *)bpage, bpage->id,
                                    &bpage->size, true);
    }
```

**批量读 `buf_read_ibuf_merge_pages()`**（`buf0rea.cc:592`）：

```cpp
  for (ulint i = 0; i < n_stored; i++) {
    ...
    if (space == nullptr) {
      /* The tablespace was not found, remove the entries for that page */
      ibuf_merge_or_delete_for_page(nullptr, page_id, nullptr, false);
      continue;
    }
    ...
    while (buf_pool->n_pend_reads >
           buf_pool->curr_size / BUF_READ_AHEAD_PEND_LIMIT) {
      std::this_thread::sleep_for(std::chrono::milliseconds(500));   // ★ 背压
    }
    buf_read_page_low(&err, sync && (i + 1 == n_stored),   // ★★ 只对最后一个页用 sync
                      IORequest::IGNORE_MISSING, BUF_READ_ANY_PAGE, page_id,
                      page_size, true);
    if (err == DB_TABLESPACE_DELETED) {
      ibuf_merge_or_delete_for_page(nullptr, page_id, &page_size, false);
    }
  }
```

**★ `AIO_mode::IBUF` 的由来**（`fil0fil.cc:7626`）：

```cpp
  /* ibuf bitmap pages must be read in the sync AIO mode: */
  ut_ad(recv_no_ibuf_operations || req_type.is_write() ||
        !ibuf_bitmap_page(page_id, page_size) || sync);
  //  ★★ bitmap 页必须走 SYNC AIO（不能异步），否则 IO 完成回调反过来又要读 bitmap 页 → 死锁

  auto aio_mode = get_AIO_mode(req_type, sync);

  if (req_type.is_read()) {
    if (aio_mode == AIO_mode::NORMAL && !recv_no_ibuf_operations &&
        ibuf_page(page_id, page_size, UT_LOCATION_HERE, nullptr)) {
      /* Reduce probability of deadlock bugs
      in connection with ibuf: do not let the
      ibuf I/O handler sleep */
      req_type.clear_do_not_wake();
      aio_mode = AIO_mode::IBUF;
      //  ★★ ibuf 树页的读被分到独立的 slot 数组，该组 IO handler 不允许 sleep
      //     因为 ibuf merge 会在 IO 完成回调里同步读 bitmap 页，handler 睡了就自死锁
    }
```

`ibuf_merge_or_delete_for_page`（`ibuf0ibuf.cc`）：

| # | 条件 | 说明 |
|---|---|---|
| 1 | `srv_force_recovery >= SRV_FORCE_NO_IBUF_MERGE (4)` | 强制恢复模式 |
| 2 | `trx_sys_hdr_page(page_id)` | 事务系统头页 |
| 3 | `fsp_is_system_temporary(page_id.space)` | 临时表空间 |
| 4 | `ibuf_fixed_addr_page(page_id, univ_page_size)` | ibuf root / bitmap 页 |
| 5 | `fsp_descr_page(page_id, univ_page_size)` | XDES 页 |
| 6 | （若 update_ibuf_bitmap）用真实 page_size 重查 #4 | |
| 7 | （若 update_ibuf_bitmap）用真实 page_size 重查 #5 | |
| 8 | `IBUF_BITMAP_BUFFERED == 0` → 什么都不用做；`space == nullptr`（表空间已删）→ 降级为"只删 ibuf 记录" | |

#### 4.2 ★ 逐行解析：`ibuf_merge_or_delete_for_page`

按执行顺序分五阶段（前置守卫见 4.1，bitmap BUFFERED 判据见 4.5）。

**阶段 1：锁所有权移交 + 损坏防御**：

```c
if (block != nullptr) {
  /* Move the ownership of the x-latch on the page to this OS
  thread, so that we can acquire a second x-latch on it. This
  is needed for the insert operations to the index page to pass
  the debug checks. */
  rw_lock_x_lock_move_ownership(&(block->lock));
  page_zip = buf_block_get_page_zip(block);

  if (!fil_page_index_page_check(block->frame) ||
      !page_is_leaf(block->frame)) {
    corruption_noticed = true;
    ib::error(ER_IB_MSG_624) << "Corruption in the tablespace. Bitmap"
        " shows insert buffer records to page " << page_id
        << " though the page type is "
        << fil_page_get_type(block->frame)
        << ", which is not an index leaf page. ...";
  }
}
```

- **x-latch 所有权移交**：目标页的 x-latch 是**发起读的线程**拿的（异步读，latch 挂在 IO 线程上下文），rw-lock 记录"writer 是哪个线程"。merge 要对同一页做插入、debug 检查要求**本线程**持 x-latch → 先把所有权移到当前线程，才合法拿第二把 x-latch。
- **损坏防御**：bitmap 说该页有 buffered 变更，但页类型**不是** index leaf page（表被 ALTER/DROP 后页被复用等）→ 置 `corruption_noticed`，后续循环把记录**丢弃**（计入 dops）而非应用，提示 `CHECK TABLE`。这是"bitmap 与页实际状态不一致"的唯一兜底。

**阶段 2：定位 search_tuple + 锁序伪装**：

```c
search_tuple = ibuf_search_tuple_build(page_id.space(), page_id.page_no(), heap);

loop:
  ibuf_mtr_start(&mtr);
  pcur.open_on_user_rec(ibuf->index, search_tuple, PAGE_CUR_GE, BTR_MODIFY_LEAF,
                        &mtr, UT_LOCATION_HERE);

  if (block != nullptr) {
    buf_page_get_known_nowait(RW_X_LATCH, block, Cache_hint::KEEP_OLD, ..., &mtr);
    /* This is a user page (secondary index leaf page),
    but we pretend that it is a change buffer page in
    order to obey the latching order. This should be OK,
    because buffered changes are applied immediately while
    the block is io-fixed. Other threads must not try to
    latch an io-fixed block. */
    buf_block_dbg_add_level(block, SYNC_IBUF_TREE_NODE);
  }
```

- **search_tuple = (space, page_no, counter=0xFFFF)**：ibuf 记录的排序键是 `(space, page_no, counter)`，counter 取最大值让 `PAGE_CUR_GE` 定位到**该页的第一条记录**（counter 0 才是最小键的下界）。
- **★ 锁序伪装（本函数最精巧的并发设计）**：同一 mtr 里先 latch ibuf 树节点（锁序 `SYNC_IBUF_TREE_NODE`），又 latch 二级索引目标页（正常锁序 `SYNC_TREE_NODE`，**更高**）——latch level 数字越大层级越高、规则是"后拿的 ≤ 已持有的"（先高后低），拿 296（`SYNC_TREE_NODE`）时已持有 277（`SYNC_IBUF_TREE_NODE`）即违规。解法是把目标页**伪装成 ibuf 树节点**（`buf_block_dbg_add_level(block, SYNC_IBUF_TREE_NODE)` 把它的 debug 锁级别降为 ibuf 树级，277 ≤ 277 合法；函数机制见 [buffer_pool.md](../innodb/buffer_pool.md) 的「`buf_block_dbg_add_level`」节）。**合法性来自 io_fix**：block 处于 `io_fix_read`（异步读进行中），`buf_page_get` 对 io-fixed block 一律不许 latch（其他线程会等待或放弃），伪装期间不可能真的发生乱序并发。注释原话 "This should be OK, because buffered changes are applied immediately while the block is io-fixed"——**用"I/O 固定"的互斥性换锁序合法性**。

**阶段 3：主循环——逐条应用**：

```c
  for (;;) {
    rec = pcur.get_rec();

    /* 这条记录还属于本页吗？不属于 → 本页的变更已全部处理 */
    if (ibuf_rec_get_page_no(&mtr, rec) != page_id.page_no() ||
        ibuf_rec_get_space(&mtr, rec) != page_id.space()) {
      page_header_reset_last_insert(block->frame, page_zip, &mtr);
      goto reset_bit;
    }

    if (!rec_get_deleted_flag(rec, 0)) {
      dtuple_t *entry;
      trx_id_t max_trx_id;
      dict_index_t *dummy_index;
      ibuf_op_t op = ibuf_rec_get_op_type(&mtr, rec);

      /* 同步 trx_id 到目标页头 */
      max_trx_id = page_get_max_trx_id(page_align(rec));
      page_update_max_trx_id(block, page_zip, max_trx_id, &mtr);

      /* 把 ibuf 记录还原成二级索引 entry。
       ★ entry 是指向 rec 字段的指针拷贝——插入完成前不能放
       ibuf 页的 latch！（源码注释：we must keep the latch to
       the rec page until the insertion is finished） */
      entry = ibuf_build_entry_from_ibuf_rec(&mtr, rec, heap, &dummy_index);

      switch (op) {
        case IBUF_OP_INSERT:
          ibuf_insert_to_index_page(entry, block, dummy_index, &mtr);
          break;
        case IBUF_OP_DELETE_MARK:
          ibuf_set_del_mark(entry, block, dummy_index, &mtr);
          break;
        case IBUF_OP_DELETE:
          ibuf_delete(entry, block, dummy_index, &mtr);
          /* 见阶段 4 */
          goto loop;
      }
      mops[op]++;
      ibuf_dummy_index_free(dummy_index);
    } else {
      dops[ibuf_rec_get_op_type(&mtr, rec)]++;   // 已打删除标记 → 丢弃
    }

    /* 把这条记录从 ibuf 树删掉；悲观删除（mtr 已提交）→ goto loop 重来 */
    if (ibuf_delete_rec(page_id.space(), page_id.page_no(), &pcur, search_tuple,
                        &mtr)) {
      ut_ad(mtr.has_committed());
      goto loop;
    }
  }
```

- **归属检查是"按序"的**：ibuf 树里同一页的记录按 counter 升序**连续排列**，所以"当前记录不属于本页"意味着**本页的所有记录已处理完**，直接跳 reset_bit。顺带 `page_header_reset_last_insert`——merge 插入了记录，页头的 last_insert 位置失效必须重置。
- **`page_update_max_trx_id`**：ibuf 记录**缓存时**的 trx_id 要同步到目标页头 `PAGE_MAX_TRX_ID`，保证 merge 后该页的 MVCC 可见性判断与"直接插入"完全一致——merge 必须语义等价于原始操作。
- **★ entry 是 rec 字段的指针拷贝**：dummy entry 的字段直接指向 ibuf 记录里的字节，**插入完成前不能放掉 ibuf 页的 latch**，否则其他线程可能改/删这条记录。这决定了整个 merge 循环必须持续持有 ibuf 树游标。
- **三 op 落到 `ibuf_insert_to_index_page` / `ibuf_set_del_mark` / `ibuf_delete`**：都是真正的 btr 操作（乐观失败退化悲观 → **页分裂**）。merge 的原子性由外层 mtr 保证——要么全部应用、要么崩溃恢复重做。
- **删记录的悲观路径**：`ibuf_delete_rec` 乐观删除失败 → 悲观删除（内部 commit mtr）→ 返回 true → `goto loop`（游标已失效，从头再来）。

**阶段 4：DELETE 的两段式 mtr（崩溃安全核心）**：

```c
case IBUF_OP_DELETE:
  ibuf_delete(entry, block, dummy_index, &mtr);
  /* Because ibuf_delete() will latch an insert buffer bitmap
  page, commit mtr before latching any further pages.
  Store and restore the cursor position. */

  /* Mark the change buffer record processed, so that it will
  not be merged again in case the server crashes between the
  following mtr_commit() and the subsequent mtr_commit() of
  deleting the change buffer record. */
  btr_cur_set_deleted_flag_for_ibuf(pcur.get_rec(), nullptr, true, &mtr);

  pcur.store_position(&mtr);
  ibuf_btr_pcur_commit_specify_mtr(&pcur, &mtr);   // ★ 第一段 mtr 提交

  ibuf_mtr_start(&mtr);                             // ★ 第二段 mtr
  buf_page_get_known_nowait(RW_X_LATCH, block, ...);
  buf_block_dbg_add_level(block, SYNC_IBUF_TREE_NODE);

  if (!ibuf_restore_pos(page_id.space(), page_id.page_no(), search_tuple,
                        BTR_MODIFY_LEAF, &pcur, &mtr)) {
    mops[op]++;
    goto loop;                     // 恢复游标失败（悲观操作）→ 从头再来
  }
  break;
```

- **为什么必须两段 mtr**：`ibuf_delete`（真正的物理删除，可能触发页合并/分裂）内部要 latch **另一张 bitmap 页**——锁序要求"commit 当前 mtr 后才能拿别的页"。所以"应用 DELETE"和"从 ibuf 树删除记录"必须在两个 mtr 里。
- **★ 先打删除标记再提交（防重复 merge）**：两个 mtr 之间存在崩溃窗口——第一段提交后、第二段删除 ibuf 记录前崩溃，重启后这条 DELETE 记录**还在 ibuf 树里**。没有删除标记，它会被再次 merge → **同一条 DELETE 被应用两次**（幂等性破坏）。打了标记后，阶段 3 的 `!rec_get_deleted_flag` 检查会跳过它、计入 dops。
- **游标恢复**：`ibuf_restore_pos` 重新定位到该页第一条记录；失败（期间发生悲观操作导致游标失效）→ `goto loop`。

**阶段 5：reset_bit 收尾**：

```c
reset_bit:
  ibuf_bitmap_page_set_bits(bitmap_page, page_id, *page_size,
                            IBUF_BITMAP_BUFFERED, false, &mtr);   // 清 BUFFERED
  if (block != nullptr) {
    ulint new_bits = ibuf_index_page_calc_free(block);
    if (old_bits != new_bits) {
      ibuf_bitmap_page_set_bits(bitmap_page, page_id, *page_size,
                                IBUF_BITMAP_FREE, new_bits, &mtr);  // 重算 FREE
    }
  }
```

merge 后页的空闲空间变了：清 `BUFFERED` 位（不再有缓存变更），并**重算** `FREE` 档位（升档！）——这正是「工程技法 → bitmap 单向阀」讲的升档侧：**必须与 merge 同 mtr**（merge 释放了空间，FREE 升档与事实同原子），单独提交会在崩溃时造成高估。

1. **整个 merge 在 `buf_page_io_complete` 的读完成回调里做**——这就是"页被读到时合并"的实现位置。
2. **`IBUF_OP_DELETE` 特殊**：`ibuf_delete` 内部要 latch 另一个 bitmap 页，所以先 `btr_cur_set_deleted_flag_for_ibuf` 打删除标记 → 提交 mtr → 重新定位（`ibuf_restore_pos`）。
3. **可能触发页分裂**：`ibuf_insert_to_index_page` 是真正的 `btr_cur_optimistic_insert`，页满时会退化为悲观插入 → 分裂。这就是"merge 可能触发页分裂"的源码证据。
4. 最后 `reset_bit` 清空 `BUFFERED` 位并重算 `FREE` 位。

#### 4.3 其他 merge 入口

| 函数 | 位置 | 触发者 |
|------|------|--------|
| `ibuf_merge_or_delete_for_page` |  | `buf_page_io_complete`（读页完成） |
| `ibuf_merge_pages` |  | 批量合并（指定 space + page_no 列表） |
| `ibuf_merge_space` |  | 合并整个表空间 |
| `ibuf_contract` |  | 收缩一次（ibuf 太大时） |
| `ibuf_merge_in_background` |  | **由 master 线程调用**（ibuf merge **无专用线程**） |
| `buf_read_ibuf_merge_pages` | `buf0rea.cc` | 批量读页做 merge |

#### 4.4 ★ 为什么用 `AIO_mode::IBUF`

`buf_read_ibuf_merge_pages` 用 `AIO_mode::IBUF`，**单独的 AIO 数组 + 单独的线程**。

原因：ibuf merge 时要读入二级索引页。如果它和普通读抢同一批 AIO 槽位，可能出现**"所有槽位都被 ibuf merge 的读占满，而 ibuf merge 又在等这些读完成"的死锁**。单独一个 `s_ibuf` 数组把这个环切断（详见 [`io.md`](../innodb/io.md) / [`fil.md`](../innodb/fil.md)）。

#### 4.5 ★ Bug#120698：`access_time` 不是可靠的 merge 门控

> **边界**：本节省掉 buffer pool 侧（压缩页驱逐竞态窗口、`HASH_DELETE`/`HASH_INSERT`、`access_time` 继承）的完整分析见 [`buffer_pool.md`](../innodb/buffer_pool.md)「压缩页驱逐与 change buffer 竞态」。本节只写 change buffer 侧的判据问题。

**缺陷**：压缩页解压路径 `Buf_fetch::zip_page_handler`（buf0buf.cc）用 `access_time != 0` 作为"跳过 merge"的判据：

```c
if (!recv_no_ibuf_operations) {
  if (access_time != std::chrono::steady_clock::time_point{}) {
    /* 跳过 merge —— 假定"页被访问过 ⇒ 已 merge 过" */
  } else {
    ibuf_merge_or_delete_for_page(block, m_page_id, &m_page_size, true);
  }
}
```

该假定在压缩页驱逐解压帧后被打破——描述符被 `HASH_INSERT` 插回 `page_hash` 时带着**继承来的非零 `access_time`**（stale），但它已是新 incarnation，窗口期缓冲的 ibuf entry 尚未应用 ⇒ merge 被永久跳过 ⇒ 页分裂后记录落错页 ⇒ B+tree 页间顺序违反（`btr_check_sibling_boundary` 报错）。

**change buffer 侧的正确判据在函数内部**。`ibuf_merge_or_delete_for_page`（ibuf0ibuf.cc）自己读 bitmap 的 `IBUF_BITMAP_BUFFERED` 位，无缓冲变更即返回：

```c
bitmap_bits = ibuf_bitmap_page_get_bits(bitmap_page, page_id, *page_size,
                                        IBUF_BITMAP_BUFFERED, &mtr);
ibuf_mtr_commit(&mtr);
if (!bitmap_bits) {
  /* No inserts buffered for this page */
  fil_space_release(space);
  return;
}
```

因此**无条件调用它是安全的**（无 BUFFERED 时仅多读一次 bitmap 页），而 `access_time` 这个外部快捷判断既多余又危险。官方认可修复：释放压缩页时置 `access_time = 0`，且改为无条件进入本函数、由内部 BUFFERED 位裁决。

**教学价值**：这是"把判据放在离真正权威状态最远的地方"的典型反面案例——`access_time` 本职是预读启发式（首次访问时间），却被重载成"是否已 merge"的代理变量，而真正的权威状态（bitmap 的 BUFFERED 位）就躺在被调用函数内部。

---

### ibuf bitmap 的维护

#### 5.1 读写函数

⚠️ **勘误**：**不存在** `ibuf_bitmap_page_set_free_bits()` / `_set_buffered()` / `_set_ibuf()` / `_set_bits_func()` / `ibuf_set_bitmap_page_bits()`。8.0.39 全部收敛成一个 **`ibuf_bitmap_page_set_bits(page, page_id, page_size, bit, val, mtr)`**。

**★ 读：单字节读 = 原子**（`ibuf0ibuf.cc:575`）：

```cpp
static inline ulint ibuf_bitmap_page_get_bits_low(
    const page_t *page, const page_id_t &page_id, const page_size_t &page_size,
    IF_DEBUG(ulint latch_type, mtr_t *mtr, ) ulint bit) {
  ut_ad(bit < IBUF_BITS_PER_PAGE);
  static_assert(IBUF_BITS_PER_PAGE % 2 == 0,
                "IBUF_BITS_PER_PAGE must be an even number!");

  bit_offset =
      (page_id.page_no() % page_size.physical()) * IBUF_BITS_PER_PAGE + bit;

  byte_offset = bit_offset / 8;
  bit_offset = bit_offset % 8;
  //  ★★ physical 是 2 的幂、4 bit/页 ⇒ bit_offset%8 只能是 0 或 4
  //     ⇒ 一页的 4 个 bit 永不跨字节边界

  map_byte = mach_read_from_1(page + IBUF_BITMAP + byte_offset);
  //  ★★ 只做一次 1 字节读 ⇒ 不会读到"半个旧值半个新值"的撕裂状态

  value = ut_bit_get_nth(map_byte, bit_offset);

  if (bit == IBUF_BITMAP_FREE) {
    ut_ad(bit_offset + 1 < 8);      // ★ 断言再次确认：free 的两 bit 同字节
    value = value * 2 + ut_bit_get_nth(map_byte, bit_offset + 1);
  }
  return (value);
}
```

**★ 写：为什么必须 `mlog_write_ulint(..., MLOG_1BYTE, ...)`**（`ibuf0ibuf.cc:634`）：

```cpp
static void ibuf_bitmap_page_set_bits(page_t *page, const page_id_t &page_id,
                                      const page_size_t &page_size, ulint bit,
                                      ulint val, mtr_t *mtr) {
  ut_ad(mtr_memo_contains_page(mtr, page, MTR_MEMO_PAGE_X_FIX));  // ★ 写要 X latch
  ...
  map_byte = mach_read_from_1(page + IBUF_BITMAP + byte_offset);

  if (bit == IBUF_BITMAP_FREE) {
    ut_ad(bit_offset + 1 < 8);
    ut_ad(val <= 3);
    map_byte = ut_bit_set_nth(map_byte, bit_offset, val / 2);
    map_byte = ut_bit_set_nth(map_byte, bit_offset + 1, val % 2);
  } else {
    ut_ad(val <= 1);
    map_byte = ut_bit_set_nth(map_byte, bit_offset, val);
  }

  mlog_write_ulint(page + IBUF_BITMAP + byte_offset, map_byte, MLOG_1BYTE, mtr);
  //  ★★★ 为什么不能直接 mach_write_to_1？
  //  1) 这是"读-改-写"单字节，mlog_write_ulint 会先落内存再追加一条
  //     MLOG_1BYTE redo（2B offset + 1B 值）。直接写就丢 redo，
  //     crash recovery 后 bit 回到旧值 → ibuf 状态与真实页不一致。
  //  2) MLOG_1BYTE 是最紧凑的 redo 类型，日志开销仅 3 字节。
  //  3) 单字节粒度 + x-latch 使这次 RMW 对并发读者是原子的。
}
```

**三层封装**（都开**独立 mtr** 并立刻提交）：

```cpp
// ibuf0ibuf.cc:718
static inline void ibuf_set_free_bits_low(const buf_block_t *block, ulint val,
                                          mtr_t *mtr) {
  if (!page_is_leaf(buf_block_get_frame(block))) {
    return;                      // ★ 只有叶子页维护 free bits
  }
  bitmap_page = ibuf_bitmap_get_map_page(block->page.id, block->page.size,
                                         UT_LOCATION_HERE, mtr);
  ibuf_bitmap_page_set_bits(bitmap_page, block->page.id, block->page.size,
                            IBUF_BITMAP_FREE, val, mtr);
}

// ibuf0ibuf.cc:746 —— 自己开 mtr，目的是不长期持 bitmap 页 latch
void ibuf_set_free_bits_func(buf_block_t *block, ulint val) {
  if (!page_is_leaf(page)) return;
  mtr_start(&mtr);               // ★★ 独立 mtr
  ...
  switch (space->purpose) {
    case FIL_TYPE_TABLESPACE: break;
    case FIL_TYPE_TEMPORARY:
    case FIL_TYPE_IMPORT:
      mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);    // ★ 临时/import 不写 redo
  }
  ibuf_bitmap_page_set_bits(bitmap_page, block->page.id, block->page.size,
                            IBUF_BITMAP_FREE, val, &mtr);
  mtr_commit(&mtr);
}

// ibuf0ibuf.cc:817
void ibuf_reset_free_bits(buf_block_t *block) {
  ibuf_set_free_bits(block, 0, ULINT_UNDEFINED);   // ★ 直接清零
}
```

**★ free bits 何时重算 / 为什么只能"悲观低估"**：

| 场景 | 函数 | 方向 | 是否同 mtr |
|---|---|---|---|
| 往页插了记录（页变满） | `ibuf_update_free_bits_if_full()` | **只能下调** | 独立 mtr |
| 页 reorganize / 结构变更 | `ibuf_update_free_bits_low()` | 双向 | **与改页同 mtr** |
| 压缩页 | `ibuf_update_free_bits_zip()` | 无条件写 | 与改页同 mtr |
| 分裂/合并影响两页 | `ibuf_update_free_bits_for_two_pages_low()` | 双向，加 `ibuf_bitmap_mutex` | 与改页同 mtr |
| 强制归零 | `ibuf_reset_free_bits()` | 置 0 | 独立 mtr |
| bulk load | `ibuf_set_bitmap_for_bulk_load()` | 直接写 + 清 BUFFERED | 独立 mtr |

```cpp
// ibuf0ibuf.ic:246 —— ★★ 官方注释就是"不做变空时上调"的理由
/** ... NOTE: The free bits in the insert buffer bitmap must never exceed
 the free space on a page.  It is unsafe to increment the bits in a
 separately committed mini-transaction, because in crash recovery, the
 free bits could momentarily be set too high.  It is only safe to use
 this function for decrementing the free bits.  Should more free space
 become available, we must not update the free bits here, because that
 would break crash recovery. */
{
  before = ibuf_index_page_calc_free_bits(block->page.size.physical(), max_ins_size);
  if (max_ins_size >= increase) {
    after = ibuf_index_page_calc_free_bits(block->page.size.physical(),
                                           max_ins_size - increase);
    //  ★ 用"操作前 max_ins_size − 增量"推算，避免再扫一遍页
  } else {
    after = ibuf_index_page_calc_free(block);
  }

  if (after == 0) {
    /* We move the page to the front of the buffer pool LRU list:
    the purpose of this is to prevent those pages to which we
    cannot make inserts using the insert buffer from slipping
    out of the buffer pool */
    buf_page_make_young(&block->page);
    //  ★ free bits 归 0 ⇒ ibuf 不再缓冲该页 ⇒ 别让它被淘汰出去
  }

  if (before > after) {
    ibuf_set_free_bits(block, after, before);
    //  ★★★ 单向阀：只有 after < before（页变满）才写
  }
}
```

> ★★ **为什么"变空时不许上调"**：独立 mtr 里上调，崩溃恢复可能出现「bitmap 已上调 → 腾出空间的那个 mtr 还没重放」的窗口，导致 ibuf 往一个其实没空间的页 merge，直接页溢出。所以"变空"路径只允许在**同一个 mtr** 里做（`_low` / `_zip`），调用点很有限。
>
> ⚠️ **不存在 `ibuf_update_free_bits()`（无后缀）与 `ibuf_update_free_bits_if_empty()`**。

**两个 bitmap 页要同时改时用 mutex 串行化**：

```cpp
// ibuf0ibuf.cc:901
void ibuf_update_free_bits_for_two_pages_low(buf_block_t *block1,
                                             buf_block_t *block2, mtr_t *mtr) {
  /* As we have to x-latch two random bitmap pages, we have to acquire
  the bitmap mutex to prevent a deadlock with a similar operation
  performed by another OS thread. */
  mutex_enter(&ibuf_bitmap_mutex);     // ★★ 两页可能落在不同 bitmap 页上
  state = ibuf_index_page_calc_free(block1);
  ibuf_set_free_bits_low(block1, state, mtr);
  state = ibuf_index_page_calc_free(block2);
  ibuf_set_free_bits_low(block2, state, mtr);
  mutex_exit(&ibuf_bitmap_mutex);
}
```

**`IBUF_BITMAP_IBUF` 的置/清**：置位在 `ibuf_add_free_page()` 尾部（`:1839`），清位在 `ibuf_remove_free_page()` 尾部（`:1950`）；读取用 `ibuf_page_low()`（`:944`）。

**三个 mutex**（`ibuf0ibuf.cc:286`）——⚠️ **不存在 `ibuf_mutex_enter()` 封装**，全是裸调用：

```cpp
/** The mutex used to block pessimistic inserts to ibuf trees */
static ib_mutex_t ibuf_pessimistic_insert_mutex;
/** The mutex protecting the insert buffer structs */
static ib_mutex_t ibuf_mutex;
/** The mutex protecting the insert buffer bitmaps */
static ib_mutex_t ibuf_bitmap_mutex;
```

**三级页层级与防死锁的完整设计注释**（`ibuf0ibuf.cc:134-201`，这是理解 ibuf 并发的钥匙）：

```
To avoid deadlocks in the ibuf system, we divide file pages into three levels:

(1) non-ibuf pages,
(2) ibuf tree pages and the pages in the ibuf tree free list, and
(3) ibuf bitmap pages.

No OS thread is allowed to access higher level pages if it has latches to
lower level pages; even if the thread owns a B-tree latch it must not access
the B-tree non-leaf pages if it has latches on lower level pages. Read-ahead
is only allowed for level 1 and 2 pages. Dedicated i/o-handler threads handle
exclusively level 1 i/o. A dedicated i/o handler thread handles exclusively
level 2 i/o. ...
```

**import 校验**（⚠️ **不存在 `ibuf_check_bitmap_page()`**，真名 `ibuf_check_bitmap_on_import()`，`ibuf0ibuf.cc:4420`）：

```cpp
    for (i = FSP_IBUF_BITMAP_OFFSET + 1;
         i < static_cast<page_no_t>(page_size.physical()); i++) {
      const page_id_t cur_page_id(space_id, page_no + i);

      if (ibuf_bitmap_page_get_bits(bitmap_page, cur_page_id, page_size,
                                    IBUF_BITMAP_IBUF, &mtr)) {
        ... return (DB_CORRUPTION);
        //  ★ 业务表空间里绝不该有任何页被标成 IBUF —— 硬错误
      }

      if (ibuf_bitmap_page_get_bits(bitmap_page, cur_page_id, page_size,
                                    IBUF_BITMAP_BUFFERED, &mtr)) {
        ib_errf(..., "Buffered changes for space %u page %u are lost", ...);
        /* Tolerate this error, so that slightly corrupted tables can be
        imported and dumped.  Clear the bit. */
        ibuf_bitmap_page_set_bits(bitmap_page, cur_page_id, page_size,
                                  IBUF_BITMAP_BUFFERED, false, &mtr);
        //  ★ BUFFERED 残留只是警告：清掉，那些变更被丢弃
      }
    }
```

| 函数 | 位置 | 职责 |
|------|------|------|
| `ibuf_bitmap_page_get_bits` |  | 读取某页的位（FREE 是 2 bit） |
| `ibuf_bitmap_page_set_bits` |  | 写位，用 `mlog_write_ulint(..., MLOG_1BYTE, mtr)` 记 redo |
| `ibuf_bitmap_page_no_calc` |  | 计算 bitmap 页号 |
| `ibuf_index_page_calc_free` | `ibuf0ibuf.ic` | 由 block 计算 FREE 档位 |
| `ibuf_index_page_calc_free_bits` |  | 由最大可插入空间算档位 |
| `ibuf_index_page_calc_free_from_bits` |  | 反算 |

**位偏移计算**：

```c
  bit_offset = (page_id.page_no % page_size.physical) * IBUF_BITS_PER_PAGE + bit;
  byte_offset = bit_offset / 8;
  bit_offset = bit_offset % 8;
```

> 注意这里 `page_size.physical` 被当作"页数"用——设计上 `XDES_DESCRIBED_PER_PAGE == UNIV_PAGE_SIZE`。

#### 5.2 何时置/清

| 位 | 置 | 清 |
|---|---|---|
| `BUFFERED` | `ibuf_insert_low` 成功插入后（若原本未置） | merge 完成的 `reset_bit` |
| `FREE` | merge 后按实际重算；页重组/分裂时更新 | 页空间减少（插入记录）时 |
| `IBUF` | ibuf 树扩展新页时 | — |

---

### 崩溃恢复

ibuf 的修改**全部记 redo**（见 [3.5](#35-ibuf-自身的-redo-常见误解)），所以崩溃安全。

**恢复流程**：

1. redo 扫描阶段：ibuf 树的插入/删除记录被解析并重放（`recv_recover_page` 应用到 ibuf 树所在的页）。
2. `MLOG_IBUF_BITMAP_INIT` 在 `log0recv.cc` 解析，重建 bitmap 页。
3. 恢复完成后，ibuf 树与 bitmap 都回到一致状态，后续的 merge 正常进行。

> **若 ibuf 树自身损坏**：`innodb_force_recovery=4`（`SRV_FORCE_NO_IBUF_MERGE`）会**跳过 ibuf 操作**——此时不再合并，但也可能丢失缓存的修改（这也是为什么 4 级以上强制恢复**可能丢数据**，官方要求之后必须导出重建）。

---

### purge 与 ibuf

**为什么 delete/purge 也能被缓存**：

- **delete-mark**（`IBUF_OP_DELETE_MARK`）：只是给记录打删除标记，**不需要检查唯一性**，所以 `ignore_sec_unique` 可以为 1。
- **purge**（`IBUF_OP_DELETE`）：真正物理删除。`row_purge_poss_sec` 先判断是否可删，然后也可以走 `ibuf_insert`。

**关键区分**（`btr0cur.cc`）：

```c
      case BTR_DELETE_OP:
        ut_ad(fetch == Page_fetch::IF_IN_POOL_OR_WATCH);
        if (!row_purge_poss_sec(cursor->purge_node, index, tuple)) {
          cursor->flag = BTR_CUR_DELETE_REF;
        } else if (ibuf_insert(IBUF_OP_DELETE, tuple, index, page_id, page_size,
                               cursor->thr)) {
          cursor->flag = BTR_CUR_DELETE_IBUF;
        } else {
          buf_pool_watch_unset(page_id);
          break;
        }
        buf_pool_watch_unset(page_id);
        goto func_exit;
```

> 注意 `BTR_DELETE_OP` 用 `Page_fetch::IF_IN_POOL_OR_WATCH`——purge 会设置 buffer pool watch，而 `ibuf_insert` 的 `check_watch` 会发现它并**拒绝缓存**（防止 purge 与缓存竞争同一页）。

---

## ★ 本机制里的工程实现技法

> 判据见自检 ⑬。ibuf 以 C 风格为主（无复杂模板/继承体系），本章**只写算法落地与体系结构，不硬凑 C++ 段落**。重点是"这份代码与教科书/常规写法的差异"。

### 一、经典算法的实现落地

#### ① bitmap 空间管理：一条"只能悲观低估"的单向阀

教科书里的 bitmap（文件系统空闲块位图、内存分配位图）是**精确反映 + 读写对称**：空间变多少，位图就如实改多少。ibuf 的 `FREE` 位**故意违反**这个直觉——它是粗粒度档位（2 bit 只有 4 档），而且**只允许低估、不允许高估**。设计契约写在头注释里：

```c
/* The insert buffer merge must always succeed.  To guarantee this,
the insert buffer subsystem keeps track of the free space in pages for
which it can buffer operations.  Two bits per page in the insert
buffer bitmap indicate the available space in coarse increments.  The
free bits in the insert buffer bitmap must never exceed the free space
on a page.  It is safe to decrement or reset the bits in the bitmap in
a mini-transaction that is committed before the mini-transaction that
affects the free space.  It is unsafe to increment the bits in a
separately committed mini-transaction, because in crash recovery, the
free bits could momentarily be set too high. */
```

逐句拆这四条工程决策：

1. **"must never exceed"**：FREE 位是**页剩余空间的悲观上界估计**，不是精确值。缓存时按 FREE 位判断"这个页还能不能装下"，所以高估 = 缓存进去装不下 = merge 失败。
2. **"coarse increments"（粗粒度）**：2 bit 只有 4 档（<512B / ≥512B / ≥1024B / ≥2048B）。粗粒度本身就是悲观化手段——把真实空闲空间**向下取整**到档位。`ibuf_index_page_calc_free_bits` 里的 `if (n == 3) { n = 2; }`（3 档被压进 2 档）是同一个方向的刻意保守：宁可低估 1KB，不可高估 1 字节。
3. **"safe to decrement or reset ... committed before"**：**先减（缓存时）后算（真实空间变化时）**——缓存一条 INSERT 时把 FREE 位减档，这个 mtr 可以**先提交**；真正写入该页（消耗空间）的操作在其后的 mtr 里。减是"提前悲观"，方向安全。
4. **"unsafe to increment in a separately committed mtr"**：**加档必须与"实际释放空间的操作"同 mtr**。因为崩溃恢复只重放已提交 mtr——如果"加 FREE 档"单独提交、而"释放空间"的 mtr 还没提交就崩溃，恢复后 FREE 位高估了真实空间。加是"滞后乐观"，必须与原子上锁绑定。

一句话：**FREE 位是一个方向不对称的状态机——降档可以超前，升档必须滞后到与事实同原子。** 这比教科书 bitmap 多了一个"崩溃安全方向性"维度。

#### ② B-tree 的"伪索引"复用：脱离 `dict_index_t` 的 KV 存储

⚠️ **勘误**：**不存在 `IBUF_FIELD_MARKER` / `IBUF_FIELD_PAGE_NO` / `IBUF_FIELD_TYPE` / `IBUF_FIELD_COUNTER` / `IBUF_FIELD_DATA` / `IBUF_FIELDS`**（全仓 grep `IBUF_FIELD` = 0 命中）。真实常量族是 **`IBUF_REC_FIELD_*`**：

```cpp
// ibuf0ibuf.cc:251
/** in the pre-4.1 format, the page number. later, the space_id */
constexpr uint32_t IBUF_REC_FIELD_SPACE = 0;
/** starting with 4.1, a marker consisting of 1 byte that is 0 */
constexpr uint32_t IBUF_REC_FIELD_MARKER = 1;
/** starting with 4.1, the page number */
constexpr uint32_t IBUF_REC_FIELD_PAGE = 2;
/** the metadata field */
constexpr uint32_t IBUF_REC_FIELD_METADATA = 3;
/** first user field */
constexpr uint32_t IBUF_REC_FIELD_USER = 4;      // ★ 是"第一个用户字段的下标"，不是字段数

/** Combined size of info fields at the beginning of the fourth field */
constexpr uint32_t IBUF_REC_INFO_SIZE = 4;
/** Operation counter */
constexpr uint32_t IBUF_REC_OFFSET_COUNTER = 0;  // 2 字节
/** Type of operation */
constexpr uint32_t IBUF_REC_OFFSET_TYPE = 2;     // 1 字节
/** Additional flags */
constexpr uint32_t IBUF_REC_OFFSET_FLAGS = 3;    // 1 字节
/** Set in IBUF_REC_OFFSET_FLAGS if the user index is in COMPACT format or later */
constexpr uint32_t IBUF_REC_COMPACT = 0x1;
```

> ★★ **记录不是"4 字段"，是「4 个固定前缀字段 + N 个用户字段」**。counter / type / flags **不是独立字段**，被打包进第 4 字段 METADATA 的头 4 字节（`IBUF_REC_INFO_SIZE`），后面紧跟 N 份「每字段 `DATA_NEW_ORDER_NULL_TYPE_BUF_SIZE` 字节的类型描述符」。

**★ ibuf 树自身的"伪索引"**：一个内存中手工拼装的 `dict_index_t`（`ibuf_init_at_db_start()`，`ibuf0ibuf.cc:516`）：

```cpp
  ibuf->index =
      dict_mem_index_create("innodb_change_buffer", "CLUST_IND", IBUF_SPACE_ID,
                            DICT_CLUSTERED | DICT_IBUF, 1);
  ibuf->index->id = DICT_IBUF_ID_MIN + IBUF_SPACE_ID;
  ibuf->index->table = dict_mem_table_create("innodb_change_buffer",
                                             IBUF_SPACE_ID, 1, 0, 0, 0, 0);
  ibuf->index->n_uniq = REC_MAX_N_FIELDS;
  rw_lock_create(index_tree_rw_lock_key, &ibuf->index->lock,
                 LATCH_ID_IBUF_INDEX_TREE);
  ibuf->index->search_info = btr_search_info_create(ibuf->index->heap);
  ibuf->index->page = FSP_IBUF_TREE_ROOT_PAGE_NO;
  ut_d(ibuf->index->cached = true);
```

> ★★ `n_uniq = REC_MAX_N_FIELDS` 是关键：ibuf 记录字段数是变化的（4 + 用户字段数），若按真实字段数算唯一键，`cmp_dtuple_rec` 会截断比较。设成最大字段数保证**整条记录都参与比较**。
>
> ★ root 页固定在 space 0 的 `FSP_IBUF_TREE_ROOT_PAGE_NO`（page 4），header 页在 page 3。
>
> ⚠️ **不存在 `ibuf_tree` 全局 B-tree 变量，也不存在 `ibuf_index_page()` 函数**。取 root 的标准方式是 `ibuf_tree_root_get()`：
> ```cpp
> static page_t *ibuf_tree_root_get(mtr_t *mtr) {
>   ut_ad(ibuf_inside(mtr));
>   ut_ad(mutex_own(&ibuf_mutex));
>   mtr_sx_lock(dict_index_get_lock(ibuf->index), mtr, UT_LOCATION_HERE);
>   block = buf_page_get(page_id_t(IBUF_SPACE_ID, FSP_IBUF_TREE_ROOT_PAGE_NO),
>                        univ_page_size, RW_SX_LATCH, UT_LOCATION_HERE, mtr);
>   ...
> }
> ```

**★ merge 时的"临时 dummy index"——每条记录现造现销**（⚠️ **不存在全局静态 `ibuf_dummy_index`**）：

```cpp
// ibuf0ibuf.cc:1273
static dict_index_t *ibuf_dummy_index_create(ulint n, bool comp) {
  dict_table_t *table;
  dict_index_t *index;

  table = dict_mem_table_create("IBUF_DUMMY", DICT_HDR_SPACE, n, 0, 0,
                                comp ? DICT_TF_COMPACT : 0, 0);
  index = dict_mem_index_create("IBUF_DUMMY", "IBUF_DUMMY", DICT_HDR_SPACE, 0, n);
  index->table = table;
  /* avoid ut_ad(index->cached) in dict_index_get_n_unique_in_tree */
  index->cached = true;
  return (index);
}

static void ibuf_dummy_index_free(dict_index_t *index) {
  dict_table_t *table = index->table;
  dict_mem_index_free(index);
  dict_mem_table_free(table);
}
```

**ibuf 记录 → 二级索引 entry**（`ibuf_build_entry_from_ibuf_rec_func()`，`ibuf0ibuf.cc:1317`）：

```cpp
  data = rec_get_nth_field_old(nullptr, ibuf_rec, IBUF_REC_FIELD_MARKER, &len);
  ut_a(len == 1);
  ut_a(*data == 0);                     // ★ marker 必须恒 0，否则不是 4.1+ 格式

  n_fields = rec_get_n_fields_old_raw(ibuf_rec) - IBUF_REC_FIELD_USER;

  tuple = dtuple_create(heap, n_fields);

  types = rec_get_nth_field_old(nullptr, ibuf_rec, IBUF_REC_FIELD_METADATA, &len);

  ibuf_rec_get_info_func(IF_DEBUG(mtr, ) ibuf_rec, nullptr, &comp, &info_len, nullptr);

  index = ibuf_dummy_index_create(n_fields, comp);   // ★★ 每条记录现造一个

  len -= info_len;
  types += info_len;                    // ★ 跳过 counter/type/flags 4 字节

  ut_a(len == n_fields * DATA_NEW_ORDER_NULL_TYPE_BUF_SIZE);

  for (i = 0; i < n_fields; i++) {
    field = dtuple_get_nth_field(tuple, i);
    data = rec_get_nth_field_old(nullptr, ibuf_rec, i + IBUF_REC_FIELD_USER, &len);
    dfield_set_data(field, data, len);

    dtype_new_read_for_order_and_null_size(
        dfield_get_type(field), types + i * DATA_NEW_ORDER_NULL_TYPE_BUF_SIZE);
    //  ★★ 从 METADATA 里还原出该列的 dtype —— 这就是"类型自包含"的兑现

    ibuf_dummy_index_add_col(index, dfield_get_type(field), len);
  }

  /* Prevent an ut_ad() failure in page_zip_write_rec() by
  adding system columns to the dummy table ... */
  ut_d(dict_table_add_system_columns(index->table, index->table->heap));
```

**构造 ibuf 记录时前缀 4 字段用"全 0 的 fake type"**（`ibuf_entry_build()`，`ibuf0ibuf.cc:1519`）：

```cpp
  memset(&fake_type, 0, sizeof(dtype_t));
  //  ★★ 前缀 4 字段全部用全 0 的 fake type：
  //     ibuf 树的比较靠 dtuple_set_types_binary() 走"二进制比较"，
  //     不走 dtype 语义比较，所以 type 只要占位即可

  /* 1) Space Id */   mach_write_to_4(buf, space);
  /* 2) Marker byte */ mach_write_to_1(buf, 0);
  /* 3) Page number */ mach_write_to_4(buf, page_no);

  /* 4) Type info, part #1 */
  if (counter == ULINT_UNDEFINED) {
    i = dict_table_is_comp(index->table) ? 1 : 0;   // 老格式：只 1 字节 compact flag
  } else {
    i = IBUF_REC_INFO_SIZE;                          // 新格式：4 字节
  }
  switch (i) {
    case 0: /* the old format does not allow delete buffering */
      ut_ad(op == IBUF_OP_INSERT); break;
    case IBUF_REC_INFO_SIZE:
      mach_write_to_2(ti + IBUF_REC_OFFSET_COUNTER, counter);
      ti[IBUF_REC_OFFSET_TYPE] = (byte)op;
      ti[IBUF_REC_OFFSET_FLAGS] =
          dict_table_is_comp(index->table) ? IBUF_REC_COMPACT : 0;
      ti += IBUF_REC_INFO_SIZE;
      break;
  }

  /* 5+) Fields from the entry */
  for (i = 0; i < n_fields; i++) {
    dfield_copy(field, entry_field);     // ★★ 只拷指针不拷数据（注释里的 NOTE）
    dtype_new_store_for_order_and_null_size(ti, dfield_get_type(entry_field),
                                            fixed_len);
    ti += DATA_NEW_ORDER_NULL_TYPE_BUF_SIZE;
  }

  dtuple_set_types_binary(tuple, n_fields + IBUF_REC_FIELD_USER);
  //  ★★★ 关键：整条 tuple 设为"二进制比较"
  //      这样 ibuf 树完全不需要真实 dict_index_t 的字段类型
```

> ★★ **"脱离 `dict_index_t` 的 KV 存储"的全部秘密就是这两行**：① 字段类型以 `dtype_new_store_for_order_and_null_size()` 的裁剪形式**自包含在 METADATA 字段里**；② 整条 tuple 设为**二进制比较**。于是 ibuf 树可以在不知道任何真实索引元数据的情况下，完成排序、二分查找、以及 merge 时的记录重建。
>
> ⚠️ `ibuf_entry_build()` 的 NOTE 原文："the original entry must be kept because we copy pointers to its fields" —— 用户字段是**浅拷贝**，调用期间原 entry 不能释放。

**搜索 tuple 只建 3 个字段**（`ibuf_search_tuple_build()`，`ibuf0ibuf.cc:1688`）：

```cpp
  tuple = dtuple_create(heap, IBUF_REC_FIELD_METADATA);
  //  ★ 只建 SPACE / MARKER / PAGE 三字段 —— 因为 ibuf 树按 (space,marker,page) 前缀排序，
  //    用 PAGE_CUR_GE 定位即可扫出该页所有 buffered 记录
```

**merge 时定位**（`ibuf_merge_or_delete_for_page` 中）：

```cpp
  pcur.open_on_user_rec(ibuf->index, search_tuple, PAGE_CUR_GE, BTR_MODIFY_LEAF,
                        &mtr, UT_LOCATION_HERE);
```

**按 space 批量定位用的是 `(space, 0, 0)`**（`ibuf_merge_space()` / `ibuf_delete_for_discarded_space()`）：

```cpp
  /* Use page number 0 to build the search tuple so that we get the
  cursor positioned at the first entry for this space id */
  search_tuple = ibuf_search_tuple_build(space, 0, heap);
```

> ⚠️ **不存在 `ibuf_can_merge_tuple()` / `ibuf_tuple_to_data()`**。
>
> 🧹 **工作区污染提示**：`ibuf0ibuf.cc:79` 有一行 `xxxxx： 待阅读` 被插在块注释中间（第 78 行 `/* STRUCTURE OF AN INSERT BUFFER RECORD` 与第 80 行之间）；`btr0cur.cc:1057/1118/1123/1126/1131` 夹有本地中文注释。**这些都不是上游原文**，引用源码时请剔除。

InnoDB 的 B-tree 代码（插入/删除/分裂/游标）本来是为"受字典管理的索引"写的——记录解析靠 `dict_index_t` 提供字段数和类型。ibuf 树却把它当**应用层自定义格式的 KV 存储**用：键是 `(space_id, page_no, counter)`，值是"二级索引 entry + 操作类型"，**记录自带类型描述**（METADATA 字段里内嵌每用户字段 6 字节的类型信息），解析完全不依赖任何 `dict_index_t`。

代码证据——解析函数把 index 参数传 `nullptr`、用**位置号硬编码**读字段：

```c
/* ibuf_rec_get_page_no_func（ibuf0ibuf.cc）——注意 index 传 nullptr */
field = rec_get_nth_field_old(nullptr, rec, IBUF_REC_FIELD_MARKER, &len);
ut_a(len == 1);
field = rec_get_nth_field_old(nullptr, rec, IBUF_REC_FIELD_PAGE, &len);
ut_a(len == 4);
return (mach_read_from_4(field));
```

字段位置是 `constexpr` 常量（`IBUF_REC_FIELD_SPACE=0` / `MARKER=1` / `PAGE=2` / `METADATA=3` / `USER=4`），格式版本靠 MARKER 字节长度区分（`len>1` → 老格式）。这是"把通用代码复用到脱离通用假设的场景"：B-tree 的键序、分裂、游标逻辑全盘复用，只有"记录怎么解析"被替换成应用层约定。

merge 时怎么把这条"裸记录"还原成真实二级索引 entry？构造一个**用完即弃的 dummy index**：

```c
/* ibuf0ibuf.cc——dummy index 是元数据替身 */
entry = ibuf_build_entry_from_ibuf_rec_func(IF_DEBUG(mtr, ) ibuf_rec, heap, &dummy_index);
volume = rec_get_converted_size(dummy_index, entry);   // 估算 merge 后体积
ibuf_dummy_index_free(dummy_index);
```

`ibuf_build_entry_from_ibuf_rec` 从记录**内嵌的类型信息**造出 dtuple + 临时 `dict_index_t`（`ibuf_dummy_index_create`），用它的 `rec_get_converted_size` 算体积（供 volume 校验），用完即释放。**常规用法是"先有字典再有记录"，这里反过来了："记录自带字典，字典用完即弃"。**

**代价**（与教科书 B-tree 的差异）：
- 记录格式**永久锁定**——枚举注释明写 "DO NOT CHANGE THE VALUES OF THESE, THEY ARE STORED ON DISK."；
- 版本迁移靠格式探测（`len % 6` 判 REDUNDANT/COMPACT/5.5+ 三种格式）而非在线升级；
- 类型信息冗余存储（每条记录重复一遍字段类型），用空间换"解析零依赖"。

### 二、复杂体系与设计模式的代码结构

#### ① "merge 必须永远成功"的契约链：一个约束牵出四层设计

这是 ibuf 子系统的**体系级主线**——不是单个算法，而是"一个约束 → 层层落实"的设计链：

```
约束：merge 在 buf_page_io_complete 的读完成回调里做，没有失败回退路径
  ├─ 缓存侧：bitmap FREE 悲观低估（"装得下才缓存"）——技法①
  ├─ 判据侧：IBUF_BITMAP_BUFFERED 位是"有无缓存"的唯一权威（Bug#120698 教训：别用外部代理变量）
  ├─ 执行侧：整个 merge 在一个 mtr 里原子完成——要么全合并、要么崩溃恢复重做（merge 的 redo 也是 mtr 的一部分）
  └─ 兜底侧：discarded operations——页/索引已被删除时，缓存操作被**丢弃**而非失败（SHOW ENGINE 的 discarded 计数）
```

每层都是"无回退"约束的直接推论：缓存前悲观估算（预防）、判据用权威位（正确性）、mtr 原子（崩溃安全）、丢弃兜底（无法预防的终态）。对照 [buffer_pool.md](../innodb/buffer_pool.md) 的"汇聚点模式"，这是另一种体系组织方式——**契约链**：从一条不可违反的约束倒推每个环节的设计。

#### ② 并发体系：资源分域切断死锁环

普通读和 ibuf merge 读如果共用同一批 AIO 槽位：merge 发起的读占满全部槽位 → 这些读等待完成 → 完成回调才做 merge → 但 merge 又需要发新读 → **等待的读永远不完成**（机制详见「合并路径 → 为什么用 `AIO_mode::IBUF`」）。切断手段是**给 ibuf 单独一组 AIO 数组 + 独立线程**（`s_ibuf`）。

这与 [buffer_pool.md](../innodb/buffer_pool.md) 的"8 把 mutex 分工"、`flush_rbt` 的"恢复期专用结构"同属一种技法：**按使用场景给共享资源划独立的分域**，而不是无脑共用。代价是内存与线程的冗余，收益是彻底消除跨场景的相互阻塞。

#### ③ 生命周期寄生：无专用 merge 线程

merge 没有自己的后台线程，寄生在 **master 线程**（`ibuf_merge_in_background`，周期性检查 ibuf 大小超限时收缩）。对照 buffer pool 的 page cleaner 专用 coordinator/worker 线程组——为什么 ibuf 不配专用线程？因为 merge 的触发点是**页被读**（读完成回调，被动触发），后台 merge 只是"ibuf 太大"时的兜底收缩，频率低、优先级低。专用线程的调度/唤醒开销不值得。这是**"按触发频率选生命周期归属"**的设计：高频主动 → 专用线程；低频被动 → 寄生。

---

## 可观测性

> 按「我想看什么」倒排的速查索引。观测点跟着机制走——各值的含义已在「核心实现」对应机制处交代，此处汇总备查。

### 系统变量

**`innodb_change_buffering`** —— ⚠️ 是 **`MYSQL_SYSVAR_ENUM`（全局级，无 session 级）**，不是 `MYSQL_THDVAR_ENUM`：

```cpp
// ha_innodb.cc:22962
static MYSQL_SYSVAR_ENUM(change_buffering, innodb_change_buffering,
                         PLUGIN_VAR_RQCMDARG,
                         "Buffer changes to reduce random access:"
                         " OFF, ON, inserting, deleting, changing, or purging.",
                         nullptr, nullptr, IBUF_USE_ALL,
                         &innodb_change_buffering_typelib);
```

```cpp
// ha_innodb.cc:575 —— ⚠️ 数组名是 innodb_change_buffering_names，不是 ..._values
static const char *innodb_change_buffering_names[] = {
    "none",    /* IBUF_USE_NONE */
    "inserts", /* IBUF_USE_INSERT */
    "deletes", /* IBUF_USE_DELETE_MARK */
    "changes", /* IBUF_USE_INSERT_DELETE_MARK */
    "purges",  /* IBUF_USE_DELETE */
    "all",     /* IBUF_USE_ALL */
    NullS};

static TYPELIB innodb_change_buffering_typelib = {
    array_elements(innodb_change_buffering_names) - 1,
    "innodb_change_buffering_typelib", innodb_change_buffering_names, nullptr};
```

```cpp
// include/ibuf0ibuf.h:61
enum ibuf_use_t {
  IBUF_USE_NONE = 0,
  IBUF_USE_INSERT,             /* insert */
  IBUF_USE_DELETE_MARK,        /* delete */
  IBUF_USE_INSERT_DELETE_MARK, /* insert+delete */
  IBUF_USE_DELETE,             /* delete+purge */
  IBUF_USE_ALL                 /* insert+delete+purge */
};
```

> ⚠️ **不存在 `IBUF_USE_PURGE`**。值 `purges` 映射的是 `IBUF_USE_DELETE`，其注释是 `delete+purge`——**命名与注释有历史遗留的不一致**。
>
> ★★ **purge buffer（DELETE 缓冲）是真实现，不是空壳**：`IBUF_OP_DELETE = 2` 存在，`ibuf_insert()` 有完整路由（`ibuf0ibuf.cc:3330`），merge 时有 `ibuf_delete()` 落到索引页（`:4170`），metric `ibuf_merges_delete` 的描述就是 "Number of **purge** records merged by change buffering"。
>
> ★ 全局开关：`ulong innodb_change_buffering = IBUF_USE_ALL;`（`ibuf0ibuf.cc:204`）。

**`innodb_change_buffer_max_size`**：

```cpp
// ha_innodb.cc:22969 —— 默认 25，范围 [0, 50]
static MYSQL_SYSVAR_UINT(
    change_buffer_max_size, srv_change_buffer_max_size, PLUGIN_VAR_RQCMDARG,
    "Maximum on-disk size of change buffer in terms of percentage"
    " of the buffer pool.",
    nullptr, innodb_change_buffer_max_size_update, CHANGE_BUFFER_DEFAULT_SIZE,
    0, 50, 0);
```

```cpp
// ha_innodb.cc:20727 —— 唯一绑定点
static void innodb_change_buffer_max_size_update(THD *, SYS_VAR *, void *,
                                                 const void *save) {
  srv_change_buffer_max_size = (*static_cast<const uint *>(save));
  ibuf_max_size_update(srv_change_buffer_max_size);
}
```

> ★ 设为 **0** ⇒ `ibuf->max_size == 0` ⇒ `ibuf_should_try()` 直接返回 false ⇒ **彻底关闭 change buffering**。

**Debug-only 两个**（`ha_innodb.cc:22983`）：

```cpp
#if defined UNIV_DEBUG || defined UNIV_IBUF_DEBUG
static MYSQL_SYSVAR_UINT(
    change_buffering_debug, ibuf_debug, PLUGIN_VAR_RQCMDARG,
    "Debug flags for InnoDB change buffering (0=none, 2=crash at merge)",
    nullptr, nullptr, 0, 0, 2, 0);

static MYSQL_SYSVAR_BOOL(disable_background_merge,
                         srv_ibuf_disable_background_merge,
                         PLUGIN_VAR_NOCMDARG | PLUGIN_VAR_RQCMDARG,
                         "Disable change buffering merges by the master thread",
                         nullptr, nullptr, false);
#endif
```

> ⚠️ **不存在 `srv_ibuf_merge_in_background` 系统变量**。

**`innodb_force_recovery`**：`>= 4`（`SRV_FORCE_NO_IBUF_MERGE`）会禁掉 merge——两处判据：`ibuf_should_try()` 的最后一项、`ibuf_merge_or_delete_for_page()` 的提前返回条件 ①。

**`innodb_fast_shutdown`**：`>= 1`（默认 1）跳过关机 ibuf merge。

**purge 与 ibuf 的接触点**（`row0purge.cc:481`）——⚠️ 全文 grep "ibuf" 只有这一处注释：

```cpp
  /* Set the query thread, so that ibuf_insert_low() will be
  able to invoke thd_get_trx(). */
  pcur.m_btr_cur.thr = static_cast<que_thr_t *>(que_node_get_parent(node));
```

> ★★ **purge 线程本身不直接调 `ibuf_*`**。真正决定是否缓冲的是 `row_purge_del_mark_sec()` → `btr_pcur_open` → `btr_cur_search_to_nth_level()` 里的 `ibuf_should_try()`，再由 `btr_cur_optimistic_delete()` / `btr_cur_pessimistic_delete()` 调 `ibuf_insert(IBUF_OP_DELETE_MARK / DELETE, ...)`。
>
> ★ `row0upd.cc:2093` / `:2358` 有同一句注释（UPDATE 的 delete-mark + insert 路径、INSERT 路径），作用同为把 `que_thr_t` 塞进 `btr_cur_t::thr`。
>
> ★ `ibuf_merge_space()` **不是常规 merge 入口**——全仓只有 1 个调用点：`row0quiesce.cc:949`（EXPORT/IMPORT TABLESPACE 的 quiesce 循环）。常规后台 merge 走 `ibuf_merge_in_background()` → `ibuf_merge()` → `ibuf_merge_pages()`。
>
> ⚠️ **不存在 `ibuf_delete_all()`**——只有按 space 删的 `ibuf_delete_for_discarded_space(space_id_t)`（`ibuf0ibuf.cc:4282`，用于 DISCARD/IMPORT TABLESPACE，注释明说"The space will become CORRUPT when you call this function!"）。

| 变量 | 默认 | 范围 | 说明 |
|------|------|------|------|
| `innodb_change_buffering` | **`all`** | none/inserts/deletes/changes/purges/all | 缓存哪些操作类型 |
| `innodb_change_buffer_max_size` | **25** | 0–50（BP 百分比） | ibuf 树占用 BP 上限（`ibuf0ibuf.h` `CHANGE_BUFFER_DEFAULT_SIZE`） |
| `innodb_ibuf_disable_background_merge` | false | 仅 DEBUG 构建 | 禁用后台 merge（调试用） |

`max_size` 动态更新（`ibuf_max_size_update`）：

```c
void ibuf_max_size_update(ulint new_val) {
  ulint new_size =
      ((buf_pool_get_curr_size / UNIV_PAGE_SIZE) * new_val) / 100;
  mutex_enter(&ibuf_mutex);
  ibuf->max_size = new_size;
  mutex_exit(&ibuf_mutex);
}
```

### 观测对象 → 手段 速查

**★ `SHOW ENGINE INNODB STATUS` 的段头与数据源**（`srv0srv.cc:1449` / `ibuf_print()`）：

```cpp
  fputs(
      "-------------------------------------\n"
      "INSERT BUFFER AND ADAPTIVE HASH INDEX\n"
      "-------------------------------------\n",
      file);
  ibuf_print(file);          // ★ ibuf 部分
  for (ulint i = 0; i < btr_ahi_parts; ++i) { ... ha_print_info(file, part.hash_table); }
```

```cpp
// ibuf0ibuf.cc:4380
void ibuf_print(FILE *file) {
  mutex_enter(&ibuf_mutex);

  fprintf(file,
          "Ibuf: size %lu, free list len %lu,"
          " seg size %lu, %lu merges\n",
          (ulong)ibuf->size, (ulong)ibuf->free_list_len, (ulong)ibuf->seg_size,
          (ulong)ibuf->n_merges);
  //  ★★ 这就是 "Ibuf: size 1, free list len 0, seg size 2, 0 merges" 的源头

  fputs("merged operations:\n ", file);
  ibuf_print_ops(ibuf->n_merged_ops, file);       // → "insert 0, delete mark 0, delete 0"
  fputs("discarded operations:\n ", file);
  ibuf_print_ops(ibuf->n_discarded_ops, file);

  mutex_exit(&ibuf_mutex);
}

static void ibuf_print_ops(const std::atomic<ulint> *ops, FILE *file) {
  static const char *op_names[] = {"insert", "delete mark", "delete"};
  static_assert(UT_ARR_SIZE(op_names) == IBUF_OP_COUNT);
  for (ulint i = 0; i < IBUF_OP_COUNT; i++) {
    fprintf(file, "%s %lu%s", op_names[i], (ulong)ops[i].load(),
            (i < (IBUF_OP_COUNT - 1)) ? ", " : "");
  }
  putc('\n', file);
}
```

> ⚠️ **常见误解：`ibuf_count_get()` 不是 STATUS 的数据源**。它整块被 `#ifdef UNIV_IBUF_COUNT_DEBUG` 包裹（`ibuf0ibuf.cc:397-418`），**默认根本不编译**；`IBUF_COUNT_N_RECS` 与 `ibuf_count_reset()` **都不存在**。真实数据源是 `ibuf_t` 的成员。

**`ibuf_t` 结构体**（`include/ibuf0ibuf.ic:61`）：

```cpp
struct ibuf_t {
  ulint size;          /*!< current size of the ibuf index tree, in pages */
  ulint max_size;      /*!< recommended maximum size ... in pages */
  ulint seg_size;      /*!< allocated pages of the file segment containing
                       ibuf header and tree */
  bool empty;          /*!< Protected by the page latch of the root page ...
                       true if and only if the insert buffer tree is empty. */
  ulint free_list_len; /*!< length of the free list */
  ulint height;        /*!< tree height */
  dict_index_t *index; /*!< insert buffer index */

  std::atomic<ulint> n_merges;
  std::atomic<ulint> n_merged_ops[IBUF_OP_COUNT];     // 各 op 已 merge 的条数
  std::atomic<ulint> n_discarded_ops[IBUF_OP_COUNT];  // 各 op 被丢弃的条数
};
```

**INNODB_METRICS 的 8 个 ibuf 计数器**（`srv0mon.cc:1131`）：

```cpp
    {"ibuf_merges_insert", "change_buffer",
     "Number of inserted records merged by change buffering", ...,
     MONITOR_OVLD_IBUF_MERGE_INSERT},
    {"ibuf_merges_delete_mark", "change_buffer",
     "Number of deleted records merged by change buffering", ...,
     MONITOR_OVLD_IBUF_MERGE_DELETE},
    {"ibuf_merges_delete", "change_buffer",
     "Number of purge records merged by change buffering", ...,
     MONITOR_OVLD_IBUF_MERGE_PURGE},
    {"ibuf_merges_discard_insert", ... MONITOR_OVLD_IBUF_MERGE_DISCARD_INSERT},
    {"ibuf_merges_discard_delete_mark", ... MONITOR_OVLD_IBUF_MERGE_DISCARD_DELETE},
    {"ibuf_merges_discard_delete", ... MONITOR_OVLD_IBUF_MERGE_DISCARD_PURGE},
    {"ibuf_merges", "change_buffer", "Number of change buffer merges", ...},
    {"ibuf_size", "change_buffer", "Change buffer size in pages", ...},
```

```cpp
// srv0mon.cc:1908 —— 取值
    case MONITOR_OVLD_IBUF_MERGE_INSERT:  value = ibuf->n_merged_ops[IBUF_OP_INSERT]; break;
    case MONITOR_OVLD_IBUF_MERGE_DELETE:  value = ibuf->n_merged_ops[IBUF_OP_DELETE_MARK]; break;
    case MONITOR_OVLD_IBUF_MERGE_PURGE:   value = ibuf->n_merged_ops[IBUF_OP_DELETE]; break;
    ...
    case MONITOR_OVLD_IBUF_SIZE:          value = ibuf->size; break;
```

**自增点汇总**：

| metric | 自增点 |
|---|---|
| `ibuf_merges` | `ibuf0ibuf.cc:4269` `ibuf->n_merges.fetch_add(1)` |
| `ibuf_merges_insert/_delete_mark/_delete` | `:4270` `ibuf_add_ops(ibuf->n_merged_ops, mops)`（`mops[op]++` 在 `:4219`/`:4209`） |
| `ibuf_merges_discard_*` | `:4271`（`dops[op]++` 在 `:4223`）与 `:4355`（`ibuf_delete_for_discarded_space` 路径，`dops[op]++` 在 `:4332`） |
| `ibuf_size` | 只读 `ibuf->size`（由 `ibuf_size_update()` 在 `:510` 更新） |

> ⚠️ 不存在名为 `ibuf_discarded` 的 metric——真实名是 `ibuf_merges_discard_insert` / `_delete_mark` / `_delete`。

**主线程的调度点**（`srv0srv.cc`）：

| 场景 | 调用 | 批量 |
|---|---|---|
| active 循环（`:2339`） | `ibuf_merge_in_background(false)` | `PCT_IO(5)` + 自适应 |
| idle 循环（`:2407`） | `ibuf_merge_in_background(true)` | `PCT_IO(100)` |
| 慢关机（`:2501`） | `ibuf_merge_in_background(true)` 直到清空 | `PCT_IO(100)` |

```cpp
  /* In very fast shutdown none of the following is necessary */
  if (srv_fast_shutdown >= 1) {
    return (false);       // ★★ 默认 innodb_fast_shutdown=1 ⇒ 关机不做 ibuf merge，留给下次崩溃恢复
  }
  if (srv_force_recovery < SRV_FORCE_NO_IBUF_MERGE) {
    n_bytes_merged = ibuf_merge_in_background(true);
  }
```

**速查表**：

| 想看什么 | 手段 |
|---|---|
| ibuf 多大 / 多少次 merge | `SHOW ENGINE INNODB STATUS` 的 `INSERT BUFFER AND ADAPTIVE HASH INDEX` 段；或 `ibuf_size` / `ibuf_merges` |
| 各类 op 的 merge / discard 数 | `ibuf_merges_insert` / `_delete_mark` / `_delete` / `ibuf_merges_discard_*` |
| 是否开启 | `innodb_change_buffering`（默认 `all`）；`innodb_change_buffer_max_size`（默认 25，0 = 关闭） |
| ibuf 是否为空 | `ibuf_is_empty()`（`ibuf0ibuf.cc:4360`，需要 root 页 latch，内部 `ut_a(is_empty == ibuf->empty)`） |
| 慢关机是否 merge 完 | `innodb_fast_shutdown = 0`；否则留给崩溃恢复 |
| 后台 merge 是否卡住 | debug-only：`innodb_disable_background_merge`（`srv_ibuf_disable_background_merge`） |
| 崩溃恢复阶段 | `innodb_force_recovery >= 4`（`SRV_FORCE_NO_IBUF_MERGE`）会**禁掉 merge** |

| 我想看 | 手段 | 入口 |
|--------|------|------|
| ibuf 树大小 / 合并统计 | SQL | `SHOW ENGINE INNODB STATUS` 的 **INSERT BUFFER** 段 |
| 缓存 / 丢弃的操作数 | SQL | 同段的 merged / discarded operations |

`SHOW ENGINE INNODB STATUS` 的 INSERT BUFFER 段：

```text
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
```

| 字段 | 含义 |
|------|------|
| `size` | 当前 ibuf 树占用页数 |
| `seg size` | ibuf 段总页数 |
| `merges` | merge 次数 |
| `merged operations` | 实际合并的操作数（insert / delete mark / delete） |
| **★ `discarded operations`** | **丢弃的操作数**——页已被删除（如 DROP 了索引）时，缓存的操作被丢弃而非应用 |

相关计数器：`ibuf->n_merges`、`ibuf->n_merged_ops[3]`、`ibuf->n_discarded_ops[3]`。

---

## Misc

### 社区边界澄清

| 常被问到 | 8.0.39 实际情况 |
|---------|----------------|
| **多棵 ibuf 树** | **不存在**。永远只有一棵，在 space 0 |
| **`MLOG_IBUF_*` 多种类型** | **只有一种** `MLOG_IBUF_BITMAP_INIT = 27`；ibuf 树走普通 B-tree redo |
| **ibuf merge 的专用后台线程** | **不存在**，寄生在 master 线程 |
| **`ibuf_rec_get_n_bytes`** | 不存在（n_bytes 是 ibuf bitmap 的概念，不是记录的） |

### 常见误解

| 误解 | 正确 |
|------|------|
| "change buffer 也缓存主键/聚簇索引" | **不**。`ut_a(!index->is_clustered)` |
| "唯一索引完全不能缓存" | **不对**。**唯一索引的 INSERT 不能缓存，但它的 delete-mark / purge 可以** |
| "唯一索引不能缓存是因为实现没做" | **不是**。"必须读页做唯一性检查"与"页不在 BP 才缓存"**逻辑互斥** |
| "change buffer 只在 INSERT 时有用" | **不是**。delete-mark 与 purge 也缓存（`changes`/`purges` 模式） |
| "change buffer 是纯内存结构" | **不是**。它是**系统表空间里的一棵 B-tree**，持久化 |
| "关掉 change buffer 一定更快" | 不一定。随机写密集（如批量导入二级索引多的表）时它收益巨大 |
| "ibuf 不占 buffer pool" | 占。ibuf 树的页要在 BP 里；上限 `innodb_change_buffer_max_size`% |
| "merge 不会触发页分裂" | **会**。`ibuf_insert_to_index_page` 是真实的 btr 插入 |

### ★ 什么时候该关 change buffer

| 场景 | 建议 |
|------|------|
| **"写入后立刻读同一批数据"**（批量导入后立即全表扫描） | **关**（`=none`）——merge 被立刻触发，只增加开销 |
| 负载以读为主、二级索引很少改 | 可关，收益有限 |
| 二级索引很多 + 随机写入密集 | **开**（默认 `all`） |
| **云盘 / 网络块存储** | **开，且收益比本地盘更大**——随机读在云盘上 0.1~3 ms，本地 ~100 µs，省下的随机读更值钱 |
| SSD 时代"随机读不再贵" | 这个说法对本地 NVMe 部分成立，但**云盘上不成立**，这就是 change buffer 至今默认开启的原因 |

> **★ 云盘上的意义**：change buffer 省的是**随机读**，而随机读在云盘上比本地盘贵**一个数量级**。所以在云盘上它的收益**更大**。

---

## 参考

> 本文结论基于 MySQL 8.0.39 源码逐行核实；涉及的函数与类型可直接在本仓库检索。

**相关文档**

- Buffer Pool 读页与 `buf_page_io_complete`（merge 的调用点）：[`buffer_pool.md`](../innodb/buffer_pool.md)
- B-tree 与二级索引操作：[`btr.md`](btr.md)
- purge 与 undo：[`undo_log.md`](../innodb/undo_log.md)
- 崩溃恢复：[`recovery.md`](../innodb/recovery.md)
- 云盘为何放大 change buffer 的收益：[`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)
