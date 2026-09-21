# InnoDB change buffer（insert buffer / ibuf）深度解析

> 基于 MySQL 8.0.39 源码，核心文件 `storage/innobase/ibuf/ibuf0ibuf.cc` / `include/ibuf0ibuf.h`。涵盖：**为什么只缓存非唯一二级索引**（唯一性检查与"页不在 BP"的互斥）、**ibuf 树与 bitmap 的物理布局**、**插入路径的 14 条否决条件**、**合并路径的 8 条触发条件**、**redo 的真相**（ibuf 树走的是普通 B-tree redo）、**三级自我保护 contract**、**★ 为什么 purge/delete-mark 反而能缓存唯一索引**。

> **边界**：本篇讲 **change buffer 本身**。Buffer Pool 的读页与预读见 [`buffer_pool.md`](buffer_pool.md)；二级索引的 B-tree 操作见 [`btr.md`](btr.md)；purge 与 undo 见 [`undo_log.md`](undo_log.md)；崩溃恢复见 [`recovery.md`](recovery.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [物理结构](#物理结构)
    - [ibuf 记录格式](#ibuf-记录格式)
  - 写路径（插入）
    - [插入路径 `ibuf_insert`](#插入路径-ibuf_insert)
  - 读路径（合并）
    - [★ 合并路径](#合并路径)
    - [ibuf bitmap 的维护](#ibuf-bitmap-的维护)
  - 协同与生命周期
    - [崩溃恢复](#崩溃恢复)
    - [purge 与 ibuf](#purge-与-ibuf)
- [★ 本机制里的工程实现技法](#本机制里的工程实现技法)
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
| **"写完立刻读"是负收益** | 见 [Misc](#什么时候该关-change-buffer) |

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

---

### 插入路径 `ibuf_insert`

#### 3.1 入口条件

`btr0cur.cc`：只有 `buf_page_get_gen(..., Page_fetch::IF_IN_POOL, ...)` 返回 **nullptr**（页不在 BP）时才尝试 ibuf。这本身就是第一条否决条件。

#### 3.2 `ibuf_insert` 的决策矩阵

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

#### 3.3 ★ 不缓存的 14 条条件（完整清单）

| # | 条件 | 位置 |
|---|---|---|
| 1 | `innodb_change_buffering = none` 或 `ibuf->max_size == 0` | `ibuf0ibuf.ic` |
| 2 | 聚簇索引 | `ibuf0ibuf.ic` + `ut_a` |
| 3 | 空间索引 / 含降序列 / quiesce 中的表 / DD 表空间 | `ibuf0ibuf.ic` |
| 4 | **唯一二级索引 + INSERT** | `ibuf0ibuf.ic` |
| 5 | `innodb_force_recovery >= 4` | `ibuf0ibuf.ic` |
| 6 | op 与 change_buffering 组合不允许 | `ibuf0ibuf.cc` |
| 7 | **页已在 BP，或已设 buffer pool watch** |  |
| 8 | `entry_size >= 空页可用空间/2` |  |
| 9 | `ibuf->size >= max_size + 10` |  |
| 10 | 悲观插入前 `ibuf_add_free_page` 失败 |  |
| 11 | **页在 BP 中，或页上有显式记录锁** |  |
| 12 | INSERT 且"已缓存量 + 本条 + 目录槽 > FREE 位代表的空间" |  |
| 13 | DELETE 且 `min_n_recs < 2` 或 watch 已触发 |  |
| 14 | counter 无法计算（遇到老格式记录） |  |

> **第 11 条是"写完立刻读"负收益的根因之一**：`buf_page_peek(page_id)` 为真的页一律不缓存。

```c
  if (buf_page_peek(page_id) || lock_rec_expl_exist_on_page(page_id)) {
    ibuf_mtr_commit(&bitmap_mtr);
    goto fail_exit;
  }
```

#### 3.4 ibuf 大小限制与三级自我保护

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
- **★ 锁序伪装（本函数最精巧的并发设计）**：同一 mtr 里先 latch ibuf 树节点（锁序 `SYNC_IBUF_TREE_NODE`），又 latch 二级索引目标页（正常锁序 `SYNC_TREE_NODE`，**更高**）——latch level 数字越大层级越高、规则是"后拿的 ≤ 已持有的"（先高后低），拿 296（`SYNC_TREE_NODE`）时已持有 277（`SYNC_IBUF_TREE_NODE`）即违规。解法是把目标页**伪装成 ibuf 树节点**（`buf_block_dbg_add_level(block, SYNC_IBUF_TREE_NODE)` 把它的 debug 锁级别降为 ibuf 树级，277 ≤ 277 合法；函数机制见 [buffer_pool.md](buffer_pool.md) 的「`buf_block_dbg_add_level`」节）。**合法性来自 io_fix**：block 处于 `io_fix_read`（异步读进行中），`buf_page_get` 对 io-fixed block 一律不许 latch（其他线程会等待或放弃），伪装期间不可能真的发生乱序并发。注释原话 "This should be OK, because buffered changes are applied immediately while the block is io-fixed"——**用"I/O 固定"的互斥性换锁序合法性**。

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

原因：ibuf merge 时要读入二级索引页。如果它和普通读抢同一批 AIO 槽位，可能出现**"所有槽位都被 ibuf merge 的读占满，而 ibuf merge 又在等这些读完成"的死锁**。单独一个 `s_ibuf` 数组把这个环切断（详见 [`io.md`](io.md) / [`fil.md`](fil.md)）。

#### 4.5 ★ Bug#120698：`access_time` 不是可靠的 merge 门控

> **边界**：本节省掉 buffer pool 侧（压缩页驱逐竞态窗口、`HASH_DELETE`/`HASH_INSERT`、`access_time` 继承）的完整分析见 [`buffer_pool.md`](buffer_pool.md)「压缩页驱逐与 change buffer 竞态」。本节只写 change buffer 侧的判据问题。

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

每层都是"无回退"约束的直接推论：缓存前悲观估算（预防）、判据用权威位（正确性）、mtr 原子（崩溃安全）、丢弃兜底（无法预防的终态）。对照 [buffer_pool.md](buffer_pool.md) 的"汇聚点模式"，这是另一种体系组织方式——**契约链**：从一条不可违反的约束倒推每个环节的设计。

#### ② 并发体系：资源分域切断死锁环

普通读和 ibuf merge 读如果共用同一批 AIO 槽位：merge 发起的读占满全部槽位 → 这些读等待完成 → 完成回调才做 merge → 但 merge 又需要发新读 → **等待的读永远不完成**（机制详见「合并路径 → 为什么用 `AIO_mode::IBUF`」）。切断手段是**给 ibuf 单独一组 AIO 数组 + 独立线程**（`s_ibuf`）。

这与 [buffer_pool.md](buffer_pool.md) 的"8 把 mutex 分工"、`flush_rbt` 的"恢复期专用结构"同属一种技法：**按使用场景给共享资源划独立的分域**，而不是无脑共用。代价是内存与线程的冗余，收益是彻底消除跨场景的相互阻塞。

#### ③ 生命周期寄生：无专用 merge 线程

merge 没有自己的后台线程，寄生在 **master 线程**（`ibuf_merge_in_background`，周期性检查 ibuf 大小超限时收缩）。对照 buffer pool 的 page cleaner 专用 coordinator/worker 线程组——为什么 ibuf 不配专用线程？因为 merge 的触发点是**页被读**（读完成回调，被动触发），后台 merge 只是"ibuf 太大"时的兜底收缩，频率低、优先级低。专用线程的调度/唤醒开销不值得。这是**"按触发频率选生命周期归属"**的设计：高频主动 → 专用线程；低频被动 → 寄生。

---

## 可观测性

> 按「我想看什么」倒排的速查索引。观测点跟着机制走——各值的含义已在「核心实现」对应机制处交代，此处汇总备查。

### 系统变量

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

- Buffer Pool 读页与 `buf_page_io_complete`（merge 的调用点）：[`buffer_pool.md`](buffer_pool.md)
- B-tree 与二级索引操作：[`btr.md`](btr.md)
- purge 与 undo：[`undo_log.md`](undo_log.md)
- 崩溃恢复：[`recovery.md`](recovery.md)
- 云盘为何放大 change buffer 的收益：[`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)
