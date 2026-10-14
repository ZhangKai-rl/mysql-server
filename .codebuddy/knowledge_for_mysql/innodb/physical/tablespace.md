# InnoDB 表空间物理结构与空间管理深度解析

> 基于 MySQL 8.0.39 源码，涵盖两部分：**物理结构**——表空间的四级结构（表空间 → 段 → 区 → 页）、三类管理结构的逐字节布局（FSP header 112B / XDES 40B / inode 192B / fseg header 10B）、**segment header → inode → fseg** 的完整解析链路、extent 与页的两级分配算法（`fseg_alloc_free_page_low` 七分支 / `fsp_alloc_free_page` / `fsp_free_page` / lease 机制 / reserve factor）；**碎片度量**——`DATA_FREE` / `FREE_EXTENTS` 的计算链路与两层碎片辨析。
>
> **边界**：本篇讲**空间管理与分配**（页之上）。页的通用布局与全页类型清单见 [`page_structure.md`](page_structure.md)；行的物理格式见 [`record.md`](record.md)；LOB 专用页见 [`lob.md`](lob.md)；页内合并（`MERGE_THRESHOLD`）属 B-tree 页机制。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [**★ 物理层级总览：record → page → extent → fseg → tablespace → file**](#物理层级总览record--page--extent--fseg--tablespace--file)
  - [Fil 层内存对象 / 表空间分类与 space_id](#内存中的表空间fil-层的四级对象)
  - [FSP_SIZE / FSP_FREE_LIMIT / FSP_FREE 三边界](#fsp_size--fsp_free_limit--fsp_free三个边界的辨析)
  - [AUTOEXTEND_SIZE](#autoextend_size8-0-的表空间增量扩展单位)
  - [FSP header / extent 与 XDES / segment 与 inode](#fsp-header112-字节的表空间头部)
  - [segment header → inode → fseg 解析链](#segment-header--inode--fseg10-字节指针的完整解析链)
  - [分配算法（一）：fseg_alloc_free_page_low 七分支](#分配算法一fseg_alloc_free_page_low-的七分支)
  - [分配算法（二）：fsp_alloc_free_page](#分配算法二fsp_alloc_free_page--碎片页的借与还)
  - [分配算法（三）：fseg_alloc_free_extent](#分配算法三fseg_alloc_free_extent--段从表空间要-extent)
  - [分配算法（四）：fseg_mark_page_used](#分配算法四fseg_mark_page_used--置位与链表迁移)
  - [段内 FREE 链填充与 reserve factor](#段内-free-链的填充与-reserve-factor-计算)
  - [表空间扩展与 fsp_fill_free_list](#表空间扩展与-fsp_fill_free_list)
  - [FSP_FLAGS 位域](#fsp_flags-位域一个-32-位承载表空间所有属性)
  - [系统表空间与 undo 表空间页布局](#系统表空间与-undo-表空间的页布局)
  - [碎片度量](#碎片度量fsp_get_available_space_in_free_extents)
  - [**★ B+ 树索引与两个段：链路的最终收口**](#b-树索引与两个段链路的最终收口)
  - [**★ 真名勘误全表**](#真名勘误全表)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

一个 `.ibd` 文件（或共享表空间）内部是一套**四级自管理结构**：

```
表空间 (tablespace / fil_space_t)
  ├── 段 segment（fseg：一个索引的叶子/非叶子各一个段）
  │     ├── 区 extent（16K 页时 64 页 = 1MB）
  │     │     └── 页 page（16K）
  │     └── 碎片页（FSEG_FRAG_ARR：≤32 个独立页）
  └── 三类管理结构：
        FSP header（页 0 的表空间头部，112B）
        XDES（extent 描述符，40B，256 个/页）
        inode（段描述符，192B，85 个/页）
```

核心洞察：**InnoDB 在"页"之上做了两层管理**——extent 管"连续页块"（减少分配碎片、利于顺序 I/O），segment 管"索引的数据生命周期"（一个 B+ 树的叶子/非叶子各自成段，删索引即整段释放）。这一层的全部元数据就是 FSP header / XDES / inode 三张结构，均以**侵入式链表（FLST）+ 位图**组织。

对外可见的碎片度量有两个入口：`information_schema.TABLES.DATA_FREE`（估算值，字节）与 `information_schema.FILES.FREE_EXTENTS`（精确值，extent 个数）。两者的数据来源是同一个底层事实——`fil_space_t` 里的 free extent 链表（`free_len`），只是加工口径不同。

### 内存中的表空间：Fil 层的四级对象

上面讲的是**磁盘布局**；内存中由 Fil 层（`fil0fil.cc`）用另一套对象表示，两者是"同一事实的两种存在"：

```
fil_system（单例）
  └── Fil_shard × 68            ← 按 space_id 分片，每片一把 mutex
        └── fil_space_t × N     ← 一个表空间（space_id + flags + 页状态）
              └── fil_node_t × N ← 一个物理文件（.ibd；多数场景 1 个）
```

分片规则（`fil0fil.cc:333`）：

```cpp
static const size_t MAX_SHARDS = 68;                            // 总分片数
static const size_t UNDO_SHARDS = 4;                            // 末尾 4 个留给 undo
static const size_t UNDO_SHARDS_START = MAX_SHARDS - UNDO_SHARDS;  // = 64

Fil_shard *shard_by_id(space_id_t space_id) {
  if (space_id >= UNDO_SHARDS_START)          // undo 表空间：独立分片
    return m_shards[UNDO_SHARDS_START + limit];
  return m_shards[space_id % UNDO_SHARDS_START];   // 普通表空间：取模
}
```

**为什么分片**：5.7 及以前整套表空间内存结构共一把大锁，8.0 改成 68 个分片各持一把 mutex，把"打开/创建/删除表空间"的锁竞争降到约 1/64。undo 表空间独占末尾 4 个分片，是因为它的创建与 truncate 频率高、且有独立后台线程，不该污染普通表空间的哈希桶。

`class Fil_shard`（`fil0fil.cc:658`）内部四个容器各司其职：

```cpp
using File_list  = UT_LIST_BASE_NODE_T(fil_node_t, LRU);              // 打开文件句柄 LRU
using Space_list = UT_LIST_BASE_NODE_T(fil_space_t, unflushed_spaces); // 有脏页待刷的 space
using Spaces     = std::unordered_map<space_id_t, fil_space_t *>;      // id → space（主路径）
using Names      = std::unordered_map<const char *, fil_space_t *, ...>; // 名 → space（DDL/open）
```

★ **注意 `fil_space_t` 与磁盘 FSP header 是两套并存的状态**：`FSP_SIZE`/`FSP_FREE_LIMIT`/`FSP_FREE` 持久化在页 0 上（mtr 保护），而 `fil_space_t::size`/`free_limit`/`free_len` 是内存缓存（latch 保护）。`fsp_fill_free_list` 里能看到两者**同步更新**：

```cpp
space->free_limit = i + FSP_EXTENT_SIZE;                                       // 内存态
mlog_write_ulint(header + FSP_FREE_LIMIT, i + FSP_EXTENT_SIZE, MLOG_4BYTES, mtr); // 磁盘态 + redo
```

启动时由 `fsp_header_init`/`fsp_load` 从页 0 读回填充内存态；这也是为什么 `FSP_FREE_LIMIT` 的推进必须记 redo——恢复后内存态要靠它重建。

**★★ `fil_node_t` 完整定义**（★ 一个 .ibd 文件在内存里的样子，`fil0fil.h:154`）：

```cpp
struct fil_node_t {
  /** tablespace containing this file */
  fil_space_t *space;         // a tablespace may correspond to multiple files.

  /** file name; protected by Fil_shard::m_mutex and log_sys->mutex. */
  char *name;

  /** whether this file is open. Note: We set the is_open flag after
  we increase the write the MLOG_FILE_OPEN record to redo log. Therefore
  we increment the in_use reference count before setting the OPEN flag. */
  bool is_open;

  /** file handle (valid if is_open) */
  pfs_os_file_t handle;

  /** size of the file in database pages (0 if not known yet);
  the possible last incomplete megabyte may be ignored if space->id == 0 */
  page_no_t size;             /* ★ 该 node 自己的页数 —— 跨 node 路由的基准 */

  /** Size of the file when last flushed, used to force the flush when file
  grows to keep the filesystem metadata synced when using O_DIRECT_NO_FSYNC */
  page_no_t flush_size;

  page_no_t init_size;        /* FIL_IBD_FILE_INITIAL_SIZE by default */
  page_no_t max_size;

  size_t n_pending_ios;       /* ★ is_open must be true if nonzero */
  size_t n_pending_flushes;

  /** Set to true when a file is being extended. */
  bool is_being_extended;     /* ★ 扩展串行化标志，见 space_extend 的 for(;;) 自旋 */

  int64_t modification_counter;   /* 自启动以来的写次数 */
  int64_t flush_counter;          /* 最近一次 flush 时的值 → is_flushed() 比较二者 */

  bool punch_hole;            /* 文件系统是否支持 PUNCH HOLE（页压缩依赖） */
  size_t block_size;          /* punch hole 的块大小，来自 stat */
  bool atomic_write;
  bool is_raw_disk;
  os_event_t sync_event;      /* 串行化 fsync 的 event */
  List_node LRU;              /* fil_system->LRU */
  size_t magic_n;             /* FIL_NODE_MAGIC_N = 89389 */
};
```

**★★ `fil_space_t` 的关键成员**（`fil0fil.h:236/347`，★ 与磁盘 FSP header 一对一镜像的那些）：

```cpp
struct fil_space_t {
  using Files = std::vector<fil_node_t, ut::allocator<fil_node_t>>;  /* ★ 是 vector 不是链表 */

  char *name{};
  space_id_t id;

  /** Files attached to this tablespace. Note: Only the system tablespace
  can have multiple files, this is a legacy issue. */
  Files files{};

  page_no_t size{};             /* ★ 物理页数 = 各 node.size 之和 */
  page_no_t size_in_header{};   /* ★ FSP_SIZE 的内存镜像（向下对齐整 MB） */
  uint64_t autoextend_size_in_bytes{};
  uint32_t free_len{};          /* ★ FSP_FREE 链表长度的内存缓存 */
  page_no_t free_limit{};       /* ★ FSP_FREE_LIMIT 的内存缓存 */
  uint32_t flags{};             /* 页大小/压缩/加密等，见 FSP_FLAGS 位域 */
  uint32_t n_reserved_extents{};/* 预留给 B-tree split 等 */
  uint32_t n_pending_flushes{};
  uint32_t n_pending_ops{};     /* ★ space_acquire/space_release 加减它 */

  rw_lock_t latch;              /* ★ 每个表空间一把 rw_lock，fsp 层 X/S latch */
  fil_type_t purpose;
  bool stop_new_ops{};          /* ★ "stop_ios" 的真身：禁新 read IO / ibuf merge / flush */
  bool prevent_file_open{};     /* ★ rename 期间禁止 open_file */

  Compression::Type compression_type;      /* ★ 页压缩属性（不是 CRC） */
  Encryption_metadata m_encryption_metadata;

  std::atomic<uint32_t> m_version{};   /* ★ truncate/delete 时 bump，让 BP 旧页变 stale */
  std::atomic_size_t m_n_ref_count{};  /* ★ 引用计数；inc_ref/dec_ref */

  fil_node_t *get_file_node(page_no_t *page_no) noexcept;   /* ★★ 跨 node 路由入口 */
};
```

> ★★ **`size` vs `size_in_header` 是两个不同的数**：前者是物理页数（fil 层），后者是 FSP_SIZE 的镜像且**向下对齐到整 MB**（`fil0fil.cc:1374`）。**两者的差就是"文件比 header 大"**——`DATA_FREE` 的计算正依赖于这个差。
> ⚠️ **`fil_space_t` 里没有 `full_crc32`**（全库 0 匹配）。8.0 的 full_crc32 是**全局**的（redo / `innodb_checksum_algorithm`），不按表空间存储。"每页属性"只有 `compression_type` 与 `flags` 的 ENCRYPTION 位。

**★★ `get_file_node()`：跨 node 路由**（★ 多文件表空间的页号换算就在这里）：

```cpp
fil_node_t *fil_space_t::get_file_node(page_no_t *page_no) noexcept {
  if (files.size() > 1) {
    ut_a(id == TRX_SYS_SPACE || purpose == FIL_TYPE_TEMPORARY);   /* ★ 只有系统/临时可多文件 */

    for (auto &f : files) {
      if (f.size > *page_no) {
        return &f;                    /* ★ 命中：*page_no 已就地改写成「node 内相对页号」 */
      }
      *page_no -= f.size;             /* ★★ 逐 node 减去前序文件页数 */
    }
  } else if (!files.empty()) {
    fil_node_t &f = files.front();
    if ((fsp_is_ibd_tablespace(id) && f.size == 0) || f.size > *page_no) {
      return &f;                      /* 单文件：页号不变 */
    }
    /* The page is outside the current bounds of the file. We should not assert
    here as we could be loading pages in buffer pool from dump file having pages
    from dropped tablespaces. ... */
  }
  return nullptr;
}
```

**★ `space_id` 从哪来**（⚠️ 不存在 `fil_space_get_next_id()`）：

```cpp
bool Fil_system::assign_new_space_id(space_id_t *space_id) {
  mutex_acquire_all();
  space_id_t id = *space_id;
  if (id < m_max_assigned_id) id = m_max_assigned_id;   /* 从「已分配最大值」起步 */
  ++id;                                                  /* ★★ 真相：纯全局自增计数器 */
  ...
  bool success = !dict_sys_t::is_reserved(id);
  if (success) *space_id = m_max_assigned_id = id;
  mutex_release_all();
  return success;
}
```

> ★ 外层 `fil_assign_new_space_id()` 只是一层壳。且注意 `fil_space_create()` 拿的是 **`mutex_acquire_all()`（所有 shard 的锁）**，不是单个分片——创建是稀有操作，简单起见全局串行。

### 表空间分类与 space_id 分配

`information_schema.INNODB_TABLESPACES`（8.0 是 DD 视图）可直接观察：

```sql
mysql> select space, name, flag, space_type from information_schema.INNODB_TABLESPACES;
+------------+------------------+-------+------------+
|     4294967294 | mysql         | 18432 | General    |
|     4294967293 | innodb_temporary|  4096 | System    |
|     4294967279 | innodb_undo_001 |     0 | Undo      |
|              2 | test_0506/t1    | 16417 | Single    |
+------------+------------------+-------+------------+
```

五类表空间：

| 类型 | 文件 | 说明 |
|---|---|---|
| 系统表空间 | `ibdata1` | space 0，含 DD 头、TRX_SYS、双写缓冲等固定页 |
| 独立表空间 | `xxx.ibd` | `innodb_file_per_table`（8.0 默认开），一张表一个文件 |
| 通用表空间 | 用户指定 | `CREATE TABLESPACE` 创建，**可容纳多张表**（与系统表空间类似、与独立表空间类似可指定路径） |
| undo 表空间 | `undo_001/002` | 8.0 默认独立，`space_type=Undo` |
| 临时表空间 | `ibtmp1` | `s_temp_space_id`，会话临时表与内部临时表 |

space_id 是 32 位，8.0 把高地址段划作特殊用途（`dict0dict.h` / `trx0sys.h`）：

```
0x0                    :  SYSTEM_TABLESPACE (ibdata1)
0x1 ~ 0xFFF9E108       :  用户表空间（普通/通用/独立）
0xFFF9E108~0xFFFFFB88  :  session temp tablespace（会话临时表）
0xFFFFFF70~0xFFFFFFEF  :  undo tablespace ID
0xFFFFFFF0             :  redo log pseudo-tablespace
0xFFFFFFF1             :  checkpoint file space
0xFFFFFFFD             :  innodb_temporary (dict_sys_t::s_temp_space_id)
0xFFFFFFFE             :  data dictionary tablespace
0xFFFFFFFF             :  invalid space
```

**为什么 space_id 从高往低分配特殊值**：它们是"哨兵常量"，用接近 `0xFFFFFFFF` 的值可以避免与用户表空间（从 1 递增）冲突，同时 `space_id >= 0xFFFFFF70` 这个判断可以一次筛出所有特殊空间（见 `fsp_is_undo_tablespace`、`fsp_is_system_tablespace` 等判定函数）。

### 用途

回答两类问题：

1. **空间管理**："新页从哪来"（`fseg_alloc_free_page` 的智能分配策略）与"页还回去后归谁"（free/frag/full 三态流转）。它决定了表空间的碎片程度、顺序扫描的物理连续性、以及 DROP INDEX 能否 O(1) 整段回收。
2. **碎片度量**："这个表浪费了多少磁盘？删数据能不能瘦身？"——判断是否需要 `OPTIMIZE TABLE` 重建的依据。

### 版本演进

| 版本 | 变化 |
|---|---|
| 远古（4.x） | 段/区结构确立：XDES + inode + fseg 三件套，碎片页数组 32 个 |
| 5.1.7 | 页类型枚举规范化（`FIL_PAGE_TYPE_*`），新增 `FIL_PAGE_TYPE_ALLOCATED` |
| 5.6 | 原子 BLOB（atomic BLOB）标志进 `FSP_FLAGS`；`FSP_FLAGS` 位域扩展（PAGE_SSIZE/ZIP_SSIZE） |
| 5.7 | 碎片度量入口：`INNODB_SYS_TABLESPACES.FREE_EXTS / TOTAL_EXTS`；`TABLES.DATA_FREE` 已存在 |
| 8.0 | **lease 机制**：新增 `XDES_FSEG_FRAG` 状态，段可租借碎片 extent 的页 2~63；DD 系统视图化：`INNODB_SYS_TABLESPACES` 被 `FILES` 视图取代 |
| 8.0.23 | 独立 undo 表空间普及：每个 undo 表空间页 3 放 `FSP_RSEG_ARRAY_PAGE_NO`（回滚段数组） |

---

## 理论基础

### 设计思想与权衡

**1. 为什么 extent 和 segment 之间还要插一层"碎片页"？**

extent = 64 页，但一张只有几行的新表只需要 2~3 页。若每次分配都整 extent，空间浪费 96%+。InnoDB 的答案是**两档分配**：

- **碎片页**（`FSEG_FRAG_ARR`，段 inode 里的 32 个 4B 槽位）：段初期从表空间的 `FSP_FREE_FRAG` 碎片 extent 里**逐页**借，每段最多 32 个碎片页；
- **整 extent**：段内碎片页用满（`FSEG_FRAG_LIMIT=32`）后，转为从 `FSP_FREE` 整 extent 分配。

这是"小对象 slab、大对象 buddy"式的经典分档策略，代价是一套段内三链表（FREE/NOT_FULL/FULL）的维护逻辑。32 这个阈值来自 `FSEG_FRAG_ARR_N_SLOTS = FSP_EXTENT_SIZE / 2`——碎片页槽位正好占半个 extent 的页数。

**2. 链表管 extent，位图管页——两级粒度的刻意分工。**

XDES 用**侵入式链表节点**（`XDES_FLST_NODE`，12B）把自己挂到某个链表上（全局 FREE/FREE_FRAG/FULL_FRAG，或段内 FREE/NOT_FULL/FULL），同时用**位图**（`XDES_BITMAP`，每页 2bit）描述 extent 内 64 个页的占用。链表解决"哪个 extent 有空页"（O(1) 取链头），位图解决"extent 内哪个页空"（O(1) 位测试）。两者缺一不可。

**3. `FSP_FREE_LIMIT` 的惰性初始化：文件是"半描述"的。**

`FSP_FREE_LIMIT` 标记"已初始化元数据的页边界"，它之上的页**根本还没加入任何 extent 链表**（代码注释：free_limit 后的 page 还没有加入 extent 中）。分配时若 FREE 链表空了，`fsp_fill_free_list` 一次性把 `FSP_FREE_ADD=4` 个 extent 从 free_limit 之上拉进链表。表空间可以预扩展（文件大但元数据小），初始化批量摊销。代价是"文件大小"与"已管理页数"是两个概念（`FSP_SIZE` vs `FSP_FREE_LIMIT`）。

**4. lease 机制（8.0）：段借碎片 extent 的"页 2~63"。**

传统碎片页来自 `FSP_FREE_FRAG` 链表里那些"半满 extent"中的零散页；8.0 新增 `XDES_FSEG_FRAG` 状态，允许段把**整个碎片 extent 租借**过来：extent 的页 0/页 1 仍是 FSP 系统页（XDES 页 + ibuf bitmap 页，`XDES_FRAG_N_USED=2`），段只使用页 2~63。相比逐页借，lease 把"一个 extent 的碎片页"打包分配，减少链表操作；归还时整个 extent 回 free_frag。

**5. 顺序分配方向 FSP_UP/DOWN：把"随机 I/O"挡在页分裂之外。**

页分裂分配新页时，`fseg_alloc_free_page` 接收方向提示：插入是升序就尽量分配**相邻的下一页**（FSP_DOWN 向低地址、FSP_UP 向高地址），让同一段的数据页物理相邻。这是 InnoDB 在没有操作系统预读提示下，用"分配器偏好"模拟顺序性的手段。

**6. 段保留因子（reserve factor）：预留空间对抗页分裂。**

段不会把 extent 全部分光：`fseg_reserve_pct`（默认 `FSEG_RESERVE_PCT_DFLT=12.50%`）要求段内空闲页占比不低于该值——保证 B+ 树分裂时有就近空间可用。`FSEG_FREE_LIST_LIMIT=40`（reserved 达 40 extent 才允许 FREE 链非空）/`FSEG_FREE_LIST_MAX_LEN=4`（一次最多填 4 个）是它的具体约束。

**7. 为什么"碎片量"是个估算值，而不是精确计数器？**

InnoDB 不维护"每张表当前有多少字节没被数据占用"的精确数字——那需要额外全局记账、每次 DML 维护。它选择**被查询时现场估算**：拿 free extent 链表长度，扣掉预留余量。代价：不是精确值（"可插入空间的上界"）；获取要拿内部 latch（`info_low()` 在 `HA_STATUS_NO_LOCK` 时跳过，Bug#38185）；语义易被误读（`DATA_FREE` 大不代表 `.ibd` 能立刻收缩——file-per-table 下文件只增不减）。

**8. 两层碎片的取舍**：B-tree 页默认填充 15/16（约 93.75%），预留 1/16 给 UPDATE 就地更新——这层"页内空隙"是**有意为之的写放大缓冲**，不在 `DATA_FREE` 统计范围；`DATA_FREE` 只统计段外整块空闲 extent。离散 vs 顺序删除的本质差异：extent 释放要求**整 extent 64 页全空**才挂回 `FSP_FREE`；离散删除留下大量"半空"页 → `DATA_FREE` 几乎不动。

### 理论溯源

- **伙伴/分档分配**：碎片页（小粒度）+ 整 extent（大粒度）的两档分配，与 slab/buddy allocator 的"按大小分类、整块切分"思想同源，只是粒度只有两档。
- **侵入式链表（FLST）**：链表节点内嵌在被管理对象（XDES、inode）里——零额外堆内存、节点与数据同页（崩溃一致性好写）。redo/undo 链表同构。
- **空间管理粒度 = extent**：与 Oracle 的 extent 概念同源。释放/分配都以 extent 为单位，free extent 用 `FSP_FREE` 链表串联，避免逐页跟踪的高开销。

### 算法与数据结构

四张结构的逐字节布局（详细解析见「核心实现」）：

| 结构 | 大小 | 关键字段 |
|---|---|---|
| `fseg_header_t` | 10B | inode 的 (space_id 4B, page_no 4B, offset 2B) |
| FSP header | 112B | space_id/size/free_limit/flags + 5 个 FLST 链表头 + seg_id |
| `xdes_t`（XDES） | 40B | segment_id 8B + FLST_NODE 12B + state 4B + bitmap 16B |
| `fseg_inode_t` | 192B | seg_id 8B + not_full_n_used 4B + 3 链表头 48B + magic 4B + 碎片页数组 128B |

### 他库对比

- **PostgreSQL**：表碎片通常看 `pgstattuple` 扩展（精确到死元组/空闲空间，但需全表扫描）；InnoDB 的 `DATA_FREE` 是 O(1) 估算，精度换速度。
- **Oracle**：`dba_free_space` / `dba_segments` 能精确列出每个表空间的空闲块；InnoDB 不做这么细的记账。

---

## 核心实现

### 主链路

**分配一个段内页**（B+ 树插入/分裂触发的最终出口）：

```
fseg_alloc_free_page(seg_header, hint, direction, mtr)
  → fseg_inode_get(seg_header)              // segment header → inode 定位
  → fseg_alloc_free_page_low                // 七分支智能分配（见下）
      ① hinted 页在本段且空闲 → 直接用
      ② hinted extent 是 FREE 且段空间不足 → 整 extent 进段 FREE 链
      ③ 方向分配：FSP_DOWN 取 extent 末页 / FSP_UP 取首页
      ④ hinted extent 属于本段且未满 → 取 hinted 页
      ⑤ used < reserved → 从 FSEG_NOT_FULL/FREE 链取一页
      ⑥ used < 32 → fsp_alloc_free_page（碎片页）
      ⑦ 否则 → fseg_alloc_free_extent 分配新 extent 取首页
  → fseg_mark_page_used                    // 置位 + FREE→NOT_FULL→FULL 链表迁移
```

**碎片度量**：

```
information_schema.TABLES.DATA_FREE
  → INTERNAL_DATA_FREE(...)
  → ha_innobase::info_low() [HA_STATUS_VARIABLE_EXTRA 时]
    → calculate_delete_length_stat()
      → fsp_get_available_space_in_free_extents()
  → stats->delete_length = avail_space * 1024

information_schema.FILES.FREE_EXTENTS
  → INTERNAL_TABLESPACE_FREE_EXTENTS(...)
  → innobase_get_tablespace_statistics()
    → stats->m_free_extents = space->free_len   // 直接是链表长度
```

> 关键命名陷阱：InnoDB 填的是 `ha_statistics::delete_length`（头文件注释即 "Free bytes"），server 层用 `set_data_free(stats.delete_length)` 映射成 `DATA_FREE`。**引擎的 `delete_length` 就是外部看到的 `DATA_FREE`**，一个东西两个名字。

---

## ★★ 物理层级总览：record → page → extent → fseg → tablespace → file

这一节把整条链路**一次性串起来**。后面各章是它的逐层展开。

### 六级层次与各自的"名字"

| 层 | 寻址单位 | 谁管理 | 元数据存储位置 | 关键换算 |
|---|---|---|---|---|
| **file** | 字节偏移 | OS / `fil_node_t` | 无（文件系统） | `offset = node内页号 × physical page size + byte_offset` |
| **tablespace** | `page_no`（逻辑页号） | `fil_space_t` + FSP header | **page 0**（FSP_HDR，112B header） | 1 个 space = 1..N 个 file；`space->size` = 各 node.size 之和 |
| **segment (fseg)** | inode entry（192B） | `fseg_inode_t` | **INODE 页**（通常 page 2） | 10B segment header → (space_id, page_no, offset) |
| **extent** | XDES entry（40B） | `xdes_t` | **XDES 页**（page 0 / 16384 / 32768 …） | 1 extent = 64 页（16K 页）；1 XDES 页管 256 个 extent |
| **page** | `page_no` | buf_pool / `page_t` | 页自身 38B FIL header + 8B trailer | `page_no % 16384 == 0` ⇒ XDES 页；`== 1` ⇒ IBUF bitmap 页 |
| **record** | 页内偏移 | `rec_t` + `offsets[]` | 记录头 5B + NULL 位图 + 变长长度 | `rec_get_offsets()` 返回**索引列数**，缺列靠 `REC_OFFS_DEFAULT` 补 |

### ★★ 一次"分配一个 B+ 树新页"的完整下沉路径

```
btr_page_alloc()                                    ← B+ 树插入/分裂触发
  │
  ├─ ① [fseg 层] fseg_alloc_free_page_general(seg_header, hint, dir, mtr)
  │     └─ fseg_inode_get(seg_header)              10B header → inode 定位
  │     └─ fseg_alloc_free_page_low()              七分支（见 §分配算法一）
  │           └─ fseg_alloc_free_extent()          段没有余量 → 要整 extent
  │                ├─ FSEG_FREE 链表有货 → 取头
  │                ├─ fsp_alloc_xdes_free_frag()   8.0「租借」路径
  │                └─ fsp_alloc_free_extent()      向表空间要
  │                     └─ ② [tablespace 层] FSP_FREE 链表空？
  │                          └─ fsp_fill_free_list()   从 FSP_FREE_LIMIT 之上初始化新 extent
  │                               └─ ③ [file 层] fsp_try_extend_data_file()
  │                                    └─ fil_space_extend()
  │                                         └─ Fil_shard::space_extend()
  │                                              └─ posix_fallocate() / fil_write_zeros()
  │
  └─ fseg_mark_page_used()                         置 XDES_FREE_BIT=0 + 链表迁移
        └─ fsp_page_create()  →  buf_page_create() → 最终 fil_io() 落盘
                                  └─ ④ [file 层] fil_io()
                                       ├─ Fil_system::shard_by_id(space_id)   space_id % 64
                                       ├─ Fil_shard::get_space_by_id()        unordered_map
                                       ├─ fil_space_t::get_file_node(&page_no) 跨 node 路由
                                       └─ offset = page_no * page_size.physical() + byte_offset
```

### ★★★ 三处最容易搞混的"定位换算"

**① 页号 → XDES entry（extent 描述符）**：

```cpp
  page_no
    ├─ xdes_calc_descriptor_page()  → XDES 页号  = page_no 向下取整到 16384 的倍数
    ├─ xdes_calc_descriptor_index() → 页内下标   = (page_no % 16384) / 64
    └─ XDES 地址 = 帧 + XDES_ARR_OFFSET(150) + XDES_SIZE(40) × 下标
```

**② 10 字节 segment header → inode**：

```cpp
  inode_addr.page    = mach_read_from_4(header + FSEG_HDR_PAGE_NO);   // +4，4B
  inode_addr.boffset = mach_read_from_2(header + FSEG_HDR_OFFSET);    // +8，2B
  ut_ad(space == mach_read_from_4(header + FSEG_HDR_SPACE));          // +0，4B（只做断言）
  inode = fut_get_ptr(space, page_size, inode_addr, RW_SX_LATCH, mtr, block);
```

**③ 逻辑页号 → 文件字节偏移**：

```cpp
  /* Fil_shard::do_io() 的五跳 */
  auto shard = fil_system->shard_by_id(page_id.space());            // 跳 1：space_id % 64
  auto space = shard->get_space_by_id(page_id.space());             // 跳 2：unordered_map
  auto page_no = page_id.page_no();
  get_file_for_io(space, &page_no, file);                           // 跳 3：page_no 就地改写成 node 内相对页号
  prepare_file_for_io(file);                                        // 跳 4：懒打开 + pending IO 计数
  auto offset = (os_offset_t)page_no * page_size.physical();        // 跳 5：★ 最终换算
  offset += byte_offset;
```

> ★★★ 第 3 跳是"多文件表空间"的关键：`fil_space_t::get_file_node()` 会**就地改写**传入的 `page_no`，逐 node 减去前序文件的页数。所以后面所有地方用的 `page_no` 都已变成"该 node 内的相对页号"。

### ★ 一个 .ibd 文件的典型固定页布局（16K 页）

```
page 0    FSP_HDR    —— FIL header(38) + FSP header(112) + XDES Arr[0..255]
page 1    IBUF_BITMAP
page 2    INODE      —— 85 个 inode entry（50 + 192*i）
page 3    SDI        —— 序列化字典信息（8.0 新增）
page 4    INDEX      —— ★ 第一个索引的根页（PAGE_BTR_SEG_LEAF@74 / TOP@84）
page 5..  数据页 / 索引页
page 16384  XDES     —— 第二个描述页（每 16384 页一个）
```

> ★★ **"每 16384 页一组"这个魔数的来源**：一个 XDES 页能描述 `16384 / 64 = 256` 个 extent，而每个 extent 64 页 → 一组正好 16384 页。所以 `page_no % 16384 == 0` 的页必然是描述页，`== 1` 的必然是 IBUF bitmap 页。这就是 `fsp_fill_free_list()` 里 `init_xdes` 判据的物理含义。
> ⚠️ **page 0 的 FIL_PAGE_TYPE 是 `FIL_PAGE_TYPE_FSP_HDR`(8)，不是 `FIL_PAGE_TYPE_XDES`(9)**——只有 page 0 之外的描述页才是 9。

### ★ 层次间的"归属"关系一图流

```
                    ┌─────────── file（.ibd）───────────┐
                    │  fil_node_t: size / handle / is_open │
                    └──────────────┬──────────────────────┘
                                   │ 1..N（一般 N==1，仅系统/临时表空间可多文件）
                    ┌──────────────▼──────────────────────┐
                    │ fil_space_t: id / size / free_len / free_limit
                    │   FSP header (page 0): FSP_SIZE / FSP_FREE_LIMIT
                    │   FSP_FREE / FSP_FREE_FRAG / FSP_FULL_FRAG 三条链
                    └──────────────┬──────────────────────┘
                                   │ 空间以 extent 为单位批发出去
        ┌──────────────────────────▼──────────────────────────┐
        │ extent (64 页) ← XDES entry (40B, 存于 XDES 页)      │
        │   state: FREE / FREE_FRAG / FULL_FRAG / FSEG / FSEG_FRAG │
        │   bitmap: 64 页 × 2 bit (FREE_BIT, CLEAN_BIT)        │
        └──────────────┬───────────────────────────────────────┘
                       │ 整块归段（XDES_FSEG）或单页借出（碎片页）
        ┌──────────────▼───────────────────────────────────────┐
        │ segment (fseg) ← inode entry (192B, 存于 INODE 页)   │
        │   FSEG_FREE / FSEG_NOT_FULL / FSEG_FULL 三条链        │
        │   FSEG_FRAG_ARR[32]  ← 段早期借的零散页              │
        └──────────────┬───────────────────────────────────────┘
                       │ 一个 B+ 树索引 = 两个段（leaf + top）
        ┌──────────────▼───────────────────────────────────────┐
        │ page（16K）← 从段里分出来                             │
        │   FIL_PAGE_TYPE: INDEX / INODE / XDES / SDI / ...     │
        └──────────────┬───────────────────────────────────────┘
                       │ 页内
        ┌──────────────▼───────────────────────────────────────┐
        │ record（rec_t）← 5B 头 + NULL 位图 + 变长长度 + 列数据 │
        │   缺的列靠 offsets[] 的 REC_OFFS_DEFAULT/DROP 虚拟补齐 │
        └──────────────────────────────────────────────────────┘
```

### FLST 侵入式链表：八大链表的统一底座

FSP header 与 inode 里的"链表"全是同一套机制（FLST，`fut0lst.h`）。先把它讲清楚，后面所有链表都是它的实例。

同样是**`typedef byte` 范式**（见上文）：

```cpp
typedef byte flst_base_node_t;                              // fut0lst.h:46
typedef byte flst_node_t;

constexpr size_t FIL_ADDR_SIZE = 6;                          // fil0types.h:130
constexpr ulint FLST_BASE_NODE_SIZE = 4 + 2 * FIL_ADDR_SIZE; // = 16B
constexpr ulint FLST_NODE_SIZE      = 2 * FIL_ADDR_SIZE;     // = 12B
```

```
flst_base_node_t（16B，"链表的头"）      flst_node_t（12B，"链表里的一环"）
 +0  ┌──────────────────┐                +0  ┌──────────────────┐
     │ len      (4B)    │  链表长度           │ prev     (6B)    │  前驱 (page_no+boffset)
 +4  ├──────────────────┤                +6  ├──────────────────┤
     │ first    (6B)    │  首节点地址         │ next     (6B)    │  后继
+10  ├──────────────────┤               +12  └──────────────────┘
     │ last     (6B)    │  尾节点地址
+16  └──────────────────┘
        每个地址 = page_no(4B) + 页内偏移 boffset(2B)
```

★ **`fil_addr_t` 是内存结构、不在物理页上**：

```cpp
struct fil_addr_t {        // fil0fil.h:1160
  page_no_t page{FIL_NULL};   // 4B，默认 FIL_NULL(0xFFFFFFFF)
  uint32_t boffset{0};        // 2B（实际只用低 16 位）
};
```

物理页上就是裸的 6 字节（4+2），`fil_addr_t` 只是把它读进内存后的**结构体视图**。这也解释了为什么 `FIL_ADDR_SIZE` 硬编码 6 而不是 `sizeof(fil_addr_t)`（后者会因对齐变成 8+）。

**三个设计后果**：

1. **链表天然跨页**：节点地址是 `(page_no, boffset)` 二元组，所以一条链表可以串起分布在**不同页**的对象——XDES 链表正是如此（256 个 XDES entry 散落在各个 XDES 页里，却同属 `FSP_FREE` 链）。
2. **`len` 字段换来 O(1) 判空/计数**：`flst_get_len(base) == 0` 就能判断"`FSP_SEG_INODES_FREE` 空了、要新分配 inode 页"（`fsp_alloc_seg_inode` 第一步就是这个判断），不必遍历。
3. **头尾双指针**支持 `flst_add_last` / `flst_add_first` 双向插入，代价是 base node 比 node 大 4B。

常用操作：`flst_init`（初始化空链）、`flst_add_last`/`flst_add_first`、`flst_remove`、`flst_insert_after`/`before`、`flst_get_len`/`first`/`last`、`flst_validate`（debug 遍历校验）。**全部带 `mtr` 参数**，因为链表指针的每一次改动都要记 redo。

#### 八大链表一览

| # | 层级 | 链表 | base node 位置 | 节点所在对象 | 节点偏移 |
|---|---|---|---|---|---|
| 1 | 表空间 | `FSP_FREE` | FSP header +62（绝对 100） | XDES entry | `XDES_FLST_NODE`(8) |
| 2 | 表空间 | `FSP_FREE_FRAG` | +78（绝对 116） | XDES entry | 同上 |
| 3 | 表空间 | `FSP_FULL_FRAG` | +94（绝对 132） | XDES entry | 同上 |
| 4 | 段 | `FSEG_FREE` | inode +12 | XDES entry | 同上 |
| 5 | 段 | `FSEG_NOT_FULL` | inode +28 | XDES entry | 同上 |
| 6 | 段 | `FSEG_FULL` | inode +44 | XDES entry | 同上 |
| 7 | 表空间 | `FSP_SEG_INODES_FREE` | FSP header +96（绝对 134） | **inode 页** | `FSEG_INODE_PAGE_NODE`(50) |
| 8 | 表空间 | `FSP_SEG_INODES_FULL` | +80（绝对 118） | **inode 页** | 同上 |

前六条的**节点都是 XDES entry**、只有 base node 不同（在表空间 FSP header 里 vs 在段 inode 里）——同一个 XDES entry 靠改挂不同 base node，就在"表空间级"和"段级"之间流转。后两条的节点是**整个 inode 页**（不是 inode entry），管理的是"哪些 inode 页还有空槽"。

#### 附：B+ 树同层页双向链表（与 FLST 完全两套机制）

除 FLST 外，页与页之间还有一条链表，靠 FIL header 里的两个字段：

```cpp
FIL_PAGE_PREV = 8;    // 4B，同层前驱页号
FIL_PAGE_NEXT = 12;   // 4B，同层后继页号
```

它把**同一 `PAGE_LEVEL` 的所有页**按页内最小键顺序串成双向链表——这是范围扫描跨页、页合并时定位邻居的物理基础。

**四类页链表对比**（这个视角最能看清它们各管什么）：

| | FLST 链表（FSP_FREE / FSEG_NOT_FULL …） | 同层页链表（PREV/NEXT） |
|---|---|---|
| 地址形式 | `(page_no 4B, boffset 2B)` = 6B | **只有 4B 页号**（节点即整页） |
| 有没有 base node | 有（16B：len + first + last） | **没有** |
| 找头/取长度 | O(1)（`flst_get_first` / `flst_get_len`） | 要从 root 沿最左路径下钻 |
| 维护方 | 通用 `flst_*` API，任何对象都能挂 | B+ 树层，页分裂/合并时改指针 |
| 成员关系 | 动态（按 extent 空闲度流转） | 固定（同索引同层的所有页） |

★ **没有 base node 是个重要后果**：想遍历某层所有页（如 `CHECK TABLE`、全索引扫描的并行分片），不能像 `flst_get_first` 那样一步拿到头，必须从 root page 沿最左子节点下钻到目标层的第一页。这也是"找层首"在代码里总是一段 `btr_page_get_father`/最左下钻循环的原因。

★ **两个例外**（容易踩坑）：

1. **页 0（FSP_HDR）的 PREV/NEXT 被复用**为 `FIL_PAGE_SRV_VERSION` / `FIL_PAGE_SPACE_VERSION`（详见 [`page_structure.md`](page_structure.md)）——它不属于任何 B+ 树层；
2. **非索引页**（FSP_HDR / XDES / INODE / IBUF_BITMAP / RSEG_ARRAY 等）不参与此链表，这两个字段要么为 `FIL_NULL`，要么被挪作他用。

所以看到 `FIL_PAGE_PREV` **不能默认是页号**——先确认页类型。

### 结构速查表

**物理结构变量一览**（全部 `typedef byte`，"形状"由偏移常量表达）：

| typedef | 大小 | 名称 | 组成 |
|---|---|---|---|
| `fseg_header_t` | 10B | segment header | space_id(4B) + page_no(4B) + offset(2B) |
| `fseg_inode_t` | 192B | inode entry | id(8B) + not_full_n_used(4B) + FREE/NOT_FULL/FULL 三个 FLST(3×16B) + magic(4B) + frag_arr(32×4B=128B) |
| `xdes_t` | 40B | XDES entry | id(8B) + flst_node(12B) + state(4B) + bitmap(16B=64页×2bit) |
| `flst_base_node_t` | 16B | 链表头 | len(4B) + first(6B) + last(6B) |
| `flst_node_t` | 12B | 链表节点 | prev(6B) + next(6B) |
| `fil_addr_t` | —（内存结构） | fsp 内地址 | page(4B) + boffset(2B)；**页上是裸 6B，非物理结构** |

**关键偏移常量**（`page0types.h`；注意 PAGE_BTR_SEG_LEAF 等 INDEX 页常量是**相对 PAGE_HEADER** 的）：

| 常量 | 相对值 | 绝对偏移 | 含义 |
|---|---|---|---|
| `PAGE_HEADER` | — | 38 | page header 起点（= FIL header 结束） |
| `PAGE_BTR_SEG_LEAF` | 36 | 74 | 叶子段 fseg header（仅 root 页） |
| `PAGE_BTR_SEG_TOP` | 36+10=46 | 84 | 非叶子段 fseg header（仅 root 页） |
| `PAGE_DATA` | 36+2×10 | 94 | 用户数据区起点 |
| `XDES_STATE` | — | — | XDES entry 内偏移（=20） |
| `FSEG_FRAG_ARR` | — | — | inode entry 内偏移（=64） |
| `RSEG_ARRAY_HEADER` | — | 48 | undo 页 3 的 rseg 数组头（= FSEG_PAGE_DATA） |

### FSP header：112 字节的表空间头部

位于页 0（`FIL_PAGE_TYPE_FSP_HDR`）的 `FSP_HEADER_OFFSET` 处，完整字段：

```cpp
/* 每个字段的页内偏移（页 0 的 FSP_HEADER_OFFSET 起） */
constexpr uint32_t FSP_SPACE_ID        = 0;    // 4B：表空间 ID
constexpr uint32_t FSP_NOT_USED        = 4;    // 4B：已废弃（旧的 flush LSN）
constexpr uint32_t FSP_SIZE            = 8;    // 4B：文件当前页数
constexpr uint32_t FSP_FREE_LIMIT      = 12;   // 4B：已初始化 extent 的边界（惰性）
constexpr uint32_t FSP_SPACE_FLAGS     = 16;   // 4B：FSP_FLAGS
constexpr uint32_t FSP_FRAG_N_USED     = 20;   // 4B：FREE_FRAG 链上已用页数
constexpr uint32_t FSP_FREE            = 24;   // FLST：空闲 extent 链表头
constexpr uint32_t FSP_FREE_FRAG       = 40;   // FLST：碎片（半空）extent 链表头
constexpr uint32_t FSP_FULL_FRAG       = 56;   // FLST：碎片满 extent 链表头
constexpr uint32_t FSP_SEG_ID          = 72;   // 8B：下一个可分配的段号
constexpr uint32_t FSP_SEG_INODES_FULL = 80;   // FLST：inode 页链表（已满）
constexpr uint32_t FSP_SEG_INODES_FREE = 96;   // FLST：inode 页链表（有空槽）

constexpr uint32_t FSP_HEADER_SIZE = 32 + 5 * FLST_BASE_NODE_SIZE;  // = 112B
```

三个全局 extent 链表是空间管理的骨架：**FREE**（整 extent 空闲）、**FREE_FRAG**（extent 里有碎片页可逐页借）、**FULL_FRAG**（碎片 extent 已借满）。`FSP_FRAG_N_USED` 追踪碎片页用量，`FSP_SEG_ID` 是段号计数器。

### extent 与 XDES：40 字节的 extent 描述符

```cpp
constexpr uint32_t XDES_ID        = 0;                  // 8B：所属 segment id
constexpr uint32_t XDES_FLST_NODE = 8;                  // 12B：链表节点（挂全局/段内链表）
constexpr uint32_t XDES_STATE     = FLST_NODE_SIZE + 8;  // 4B：状态（= 20）
constexpr uint32_t XDES_BITMAP    = FLST_NODE_SIZE + 12; // 16B：页占用位图（= 24）
// XDES_SIZE = 24 + 16 = 40B（64 页 × 2bit = 128bit = 16B）
```

位图每页 2 bit：`XDES_FREE_BIT=0`（空闲）+ `XDES_CLEAN_BIT=1`（**已废弃**，注释明说 "currently not used"，历史上标记"页上有待清理的旧版本"）。extent 状态机 6 态：

```cpp
enum xdes_state_t {
  XDES_NOT_INITED = 0,   // 未初始化
  XDES_FREE       = 1,   // 在表空间 FREE 链表（整 extent 空闲）
  XDES_FREE_FRAG  = 2,   // 在表空间 FREE_FRAG 链表（碎片 extent，可逐页借）
  XDES_FULL_FRAG  = 3,   // 在表空间 FULL_FRAG 链表（碎片已借满）
  XDES_FSEG       = 4,   // 属于某个段（整 extent 给了段）
  XDES_FSEG_FRAG  = 5    // 8.0 lease：从 FREE_FRAG 租借给段的碎片 extent
};
```

ASCII 图（一个 40B 的 XDES entry）：

```
 +0    ┌──────────────────────────────┐
       │ XDES_ID (8B)                 │  所属 segment id（0 = 不属于任何段）
 +8    ├──────────────────────────────┤
       │ XDES_FLST_NODE (12B)         │  侵入式链表节点：前驱 6B + 后继 6B
 +20   ├──────────────────────────────┤
       │ XDES_STATE (4B)              │  上面 6 态之一
 +24   ├──────────────────────────────┤
       │ XDES_BITMAP (16B)            │  64 页 × 2bit
+40    └──────────────────────────────┘
        位图内每页 2bit：[FREE(bit0)] [CLEAN(bit1，已废弃)]
```

`XDES_SIZE` 之所以用宏而非常量，是因为它依赖 `FSP_EXTENT_SIZE`（随 page size 变）：

```cpp
#define XDES_SIZE \
  (XDES_BITMAP + UT_BITS_IN_BYTES(FSP_EXTENT_SIZE * XDES_BITS_PER_PAGE))
// 16K 页：24 + 64*2/8 = 24 + 16 = 40B
// 4K 页： FSP_EXTENT_SIZE=256 → 24 + 64 = 88B
```

**XDES 的操作函数**（都围绕"descr 指针 + 偏移 + 记 redo"）：

| 函数 | 作用 |
|---|---|
| `xdes_get_descriptor(space, page_no, page_size, mtr, ...)` | 由页号算出它属于哪个 extent、XDES 在哪一页、页内偏移 |
| `xdes_calc_descriptor_page(page_size, page_no)` | 纯算术：该页的 XDES 落在哪个 XDES 页 |
| `xdes_get_descriptor_with_space_hdr(...)` | 已知 FSP header 时的加速版本（避免重复读页 0） |
| `xdes_lst_get_descriptor(space, page_size, fil_addr_t, mtr)` | 由链表节点地址反查 XDES（fil_addr = page + boffset） |
| `xdes_get_state` / `xdes_set_state` | 读写状态（置状态时同时记 redo） |
| `xdes_get_segment_id` / `xdes_set_segment_id` | 读写所属段号（后者同时置状态） |
| `xdes_mtr_get_bit(descr, XDES_FREE_BIT, offset, mtr)` | 读某页的 free 位 |
| `xdes_set_bit(descr, bit, offset, val, mtr)` | 写位（记 redo） |
| `xdes_find_bit(descr, bit, val, hint, mtr)` | 从 hint 起找第一个值为 val 的位 → 页内偏移 |
| `xdes_is_full` / `xdes_is_free` / `xdes_get_n_used` | 位图整体判定 |
| `xdes_get_offset(descr)` | XDES 对应的 extent 首页号 |
| `xdes_init(descr, mtr)` | 初始化一个 XDES（清零 + 状态） |

XDES 页本身也是一个页（`FIL_PAGE_TYPE_XDES`），页内 `XDES_ARR_OFFSET` 起是**连续数组**，每页约 256 个 XDES 描述 256 个 extent。XDES 页**每 256 个 extent 出现一次**，页 0 的 FSP_HDR 页兼任第一个 XDES 页。

### segment 与 inode：192 字节的段描述符

#### 先说"源码结构在哪"：`typedef byte` 范式

读 InnoDB 代码找"XDES 结构体""inode 结构体"会一无所获——**它们不是 C struct**：

```cpp
typedef byte rec_t;            // rem0types.h：一条物理记录
typedef byte page_t;           // page0types.h：一个页
typedef byte xdes_t;           // fsp0fsp.h：extent 描述符
typedef byte fseg_inode_t;     // fsp0fsp.h：段描述符（inode entry）
typedef byte fseg_header_t;    // fsp0types.h：段头（10B 三元组）
```

全部是**指向页内某个字节的不透明指针**，结构的"形状"完全由偏移常量表 + 宏/inline 函数表达：

```
结构 = typedef byte 指针  +  偏移常量（FSEG_ID / XDES_STATE / ...）  +  访问函数（mach_read_from_8 等）
```

为什么这样设计：这些结构**住在 buffer pool 的页帧里**，是磁盘字节块的直接映射——不能用 C struct 描述（编译器插入的对齐填充会破坏磁盘布局，且页帧地址会随 LRU 换入换出变化）。于是 InnoDB 统一采用"字节指针 + 偏移常量 + `mach_read_from_N`/`mach_write_to_N` 按端序读写"的风格。这套范式的代价是读代码时必须对着偏移常量表看，收益是磁盘/内存布局完全一致、零序列化成本。

访问方式的实际样子：

```cpp
/* 读 inode 里的段号：指针 + 偏移 + 8 字节大端读 */
seg_id = mach_read_from_8(inode + FSEG_ID);

/* 写 XDES 的段号并置状态 */
xdes_set_segment_id(descr, seg_id, XDES_FSEG, mtr);

/* 读/写都记 redo（mtr 参数）——空间元数据修改必须崩溃可恢复 */
mlog_write_ulint(inode + FSEG_MAGIC_N, FSEG_MAGIC_N_VALUE, MLOG_4BYTES, mtr);
```

#### inode entry 的字段布局

```cpp
constexpr uint32_t FSEG_ID              = 0;                     // 8B：段 ID，0=未用
constexpr uint32_t FSEG_NOT_FULL_N_USED = 8;                     // 4B：NOT_FULL 链已用页数
constexpr uint32_t FSEG_FREE            = 12;                    // FLST：段内空闲 extent
constexpr uint32_t FSEG_NOT_FULL        = 12 + FLST_BASE_NODE_SIZE;  // 28：部分使用
constexpr uint32_t FSEG_FULL            = 12 + 2 * FLST_BASE_NODE_SIZE; // 44：满 extent
constexpr uint32_t FSEG_MAGIC_N         = 12 + 3 * FLST_BASE_NODE_SIZE; // 60：魔数
constexpr uint32_t FSEG_FRAG_ARR        = 16 + 3 * FLST_BASE_NODE_SIZE; // 64：碎片页数组
// FSEG_INODE_SIZE = 16 + 48 + 32*4 = 192B
constexpr uint32_t FSEG_MAGIC_N_VALUE = 97937874;
```

ASCII 图（一个 192B 的 inode entry）：

```
 +0    ┌──────────────────────────────┐
       │ FSEG_ID (8B)                 │  段号，0 = 空闲槽
 +8    ├──────────────────────────────┤
       │ FSEG_NOT_FULL_N_USED (4B)    │  NOT_FULL 链上已用页数（O(1) 记账）
 +12   ├──────────────────────────────┤
       │ FSEG_FREE       (16B FLST)   │  段内完全空闲的 extent
 +28   ├──────────────────────────────┤
       │ FSEG_NOT_FULL   (16B FLST)   │  段内部分使用的 extent
 +44   ├──────────────────────────────┤
       │ FSEG_FULL       (16B FLST)   │  段内已用满的 extent
 +60   ├──────────────────────────────┤
       │ FSEG_MAGIC_N    (4B)         │  恒为 97937874
 +64   ├──────────────────────────────┤
       │ FSEG_FRAG_ARR   (32 × 4B)    │  32 个碎片页的页号（FIL_NULL=空槽）
+192   └──────────────────────────────┘
```

段内三个 extent 链表（FREE/NOT_FULL/FULL）与表空间三个全局链表（FREE/FREE_FRAG/FULL_FRAG）**同名不同物**：前者分类"段拥有的 extent 的占用程度"，后者分类"未归属段的 extent 的状态"。一个段从 FSP 拿到 extent 后，XDES 状态从 `XDES_FREE` 翻转为 `XDES_FSEG`，其链表节点从全局链摘下、挂进段的某个链。

#### inode 页的结构与读写函数

```
FIL header (38B) │ FSEG_INODE_PAGE_NODE (12B FLST) │ inode[0] (192B) │ ... │ inode[84] │ FIL trailer
                 ↑ FSEG_ARR_OFFSET = FSEG_PAGE_DATA + FLST_NODE_SIZE = 50
```

页头那 12B 是链表节点（`FSEG_INODE_PAGE_NODE = FSEG_PAGE_DATA`），把 inode 页挂进 `FSP_SEG_INODES_FREE`（有空槽）/ `FSP_SEG_INODES_FULL`（已用满）两个链表。每页 inode 数：

```cpp
static inline uint32_t FSP_SEG_INODES_PER_PAGE(page_size_t page_size) {
  return (page_size.physical() - FSEG_ARR_OFFSET - 10) / FSEG_INODE_SIZE;
}   // 16K 页：(16384 - 50 - 10) / 192 = 85
```

（尾部减 10 是给页尾预留，避免 inode 数组紧贴 FIL trailer。）

操作 inode 的函数都围绕"页 + 槽位"展开：

| 函数 | 作用 |
|---|---|
| `fseg_inode_try_get(header, space, page_size, mtr, &block)` | 经 fseg header 定位 inode（见下节），返回时 inode 页已 X-latch |
| `fseg_inode_get(...)` | 同上，但 `ut_a(inode)` 断言非空 |
| `fsp_seg_inode_page_find_free(page, 0, page_size, mtr)` | 在页内找空槽（FSEG_ID==0） |
| `fsp_seg_inode_page_find_used(page, page_size, mtr)` | 找已用槽 |
| `fseg_find_free_frag_page_slot(inode, mtr)` | 找 `FSEG_FRAG_ARR` 里的空槽 |
| `fseg_get_nth_frag_page_no` / `fseg_set_nth_frag_page_no` | 读写第 n 个碎片页槽（4B，记 redo） |
| `fsp_free_seg_inode(space, page_size, inode, mtr)` | 释放：FSEG_ID 置 0、magic 改写 `0xfa051ce3`、页在 FULL/FREE 链表间迁移；整页空则 `fsp_free_page` |

`fsp_free_seg_inode` 里有个细节：释放时把魔数改写成另一个值（`0xfa051ce3`）而非清零——保留"这里曾经是段"的痕迹，便于诊断内存/磁盘越界。

#### INODE 页 vs XDES 页：两种元数据存储页的定位策略

两个都是"存放管理元数据的页"，但定位方式截然不同，对比着看最能理解设计取舍：

| 特征 | XDES 页 | INODE 页 |
|---|---|---|
| 位置 | **固定**：每 256 个 extent 的第 1 页 | **按需分配，位置不固定** |
| 首个页 | 页 0（FSP_HDR） | 页 2（**仅第一个固定**，其余走链表） |
| 数量 | 可预测（≈ 表空间 extent 数 / 256） | **动态增长** |
| 定位方式 | **O(1) 算术计算** | **链表头访问**（`FSP_SEG_INODES_FREE` 首节点） |
| 后续页 | 页 16384、32768…（可预测） | 任意空闲页 |
| 页内空闲管理 | 无（槽位与 extent 一一对应，天然无空洞） | `FSEG_ID == 0` 判空 + 页级 FREE/FULL 双链 |

★ **为什么 XDES 能算、INODE 不能**：XDES 描述的是"第 n 个 extent"，n 与页号之间存在**固定的算术关系**（每 256 个 extent 一组、组首页即 XDES 页），所以 `xdes_calc_descriptor_page` 一个 `ut_2pow_round` 就能定位；而 inode 描述的是"第 n 个**段**"——段随 CREATE/DROP INDEX 动态生灭，没有任何算术规律可循，只能把"有空槽的 inode 页"串成链表按需取用。

这个差异进一步解释了两个细节：XDES 数组**不需要**空闲位图（无空洞），inode 页却需要 `FSEG_ID == 0` 判空 + 整页满时在 FREE/FULL 链表间迁移——**inode 的空闲度有两个层级**（页内槽位 + 页级链表），XDES 只有一个。

**★★ INODE 页的完整布局与分配**（`fsp0fsp.cc:2001`）：

```cpp
static bool fsp_alloc_seg_inode_page(fsp_header_t *space_header, mtr_t *mtr) {
  block = fsp_alloc_free_page(space, page_size, 0, RW_SX_LATCH, mtr, mtr);
                                              /* ★ 向 FSP 借「单个碎片页」当新 INODE 页 */
  page = buf_block_get_frame(block);
  mlog_write_ulint(page + FIL_PAGE_TYPE, FIL_PAGE_INODE, MLOG_2BYTES, mtr);
                                              /* ★ 页类型 = FIL_PAGE_INODE */

  for (page_no_t i = 0; i < FSP_SEG_INODES_PER_PAGE(page_size); i++) {
    inode = fsp_seg_inode_page_get_nth_inode(page, i, page_size, mtr);
    mlog_write_ull(inode + FSEG_ID, 0, mtr);  /* ★ 85 个 entry 的 FSEG_ID 全清 0 = 全空闲 */
  }
  flst_add_last(space_header + FSP_SEG_INODES_FREE, page + FSEG_INODE_PAGE_NODE, mtr);
                                              /* ★ 挂到 FSP_SEG_INODES_FREE 链表尾（节点在页内偏移 38） */
  return true;
}

static fseg_inode_t *fsp_alloc_seg_inode(fsp_header_t *space_header, mtr_t *mtr) {
  if (flst_get_len(space_header + FSP_SEG_INODES_FREE) == 0 &&
      !fsp_alloc_seg_inode_page(space_header, mtr)) {
    return nullptr;                           /* 没空闲 INODE 页 → 新开一页 */
  }
  /* 取 FSP_SEG_INODES_FREE 链表第一个 INODE 页 */
  const page_id_t page_id(page_get_space_id(page_align(space_header)),
                          flst_get_first(space_header + FSP_SEG_INODES_FREE, mtr).page);
  block = buf_page_get(page_id, page_size, RW_SX_LATCH, UT_LOCATION_HERE, mtr);
  page = buf_block_get_frame(block);
  n = fsp_seg_inode_page_find_free(page, 0, page_size, mtr);
  ut_a(n != FIL_NULL);
  inode = fsp_seg_inode_page_get_nth_inode(page, n, page_size, mtr);
                                              /* ★ inode 地址 = page + 50 + 192*n */

  if (FIL_NULL == fsp_seg_inode_page_find_free(page, n + 1, page_size, mtr)) {
    /* There are no other unused headers left on the page: move it to another list */
    flst_remove(space_header + FSP_SEG_INODES_FREE, page + FSEG_INODE_PAGE_NODE, mtr);
    flst_add_last(space_header + FSP_SEG_INODES_FULL, page + FSEG_INODE_PAGE_NODE, mtr);
                                              /* ★ 该页用完 → FREE 链表迁到 FULL 链表 */
  }
  return (inode);
}

/* 定位：定长数组，不是链表 */
static inline fseg_inode_t *fsp_seg_inode_page_get_nth_inode(
    page_t *page, page_no_t i, const page_size_t &page_size, mtr_t *mtr) {
  return (page + FSEG_ARR_OFFSET + FSEG_INODE_SIZE * i);   /* ★ = page + 50 + 192*i */
}
```

```cpp
/* inode entry 的字段布局（192 字节，16K 页） */
constexpr uint32_t FSEG_ID               = 0;   // 8B  段 ID，0 = 未使用
constexpr uint32_t FSEG_NOT_FULL_N_USED  = 8;   // 4B  NOT_FULL 链表中的已用页数
constexpr uint32_t FSEG_FREE             = 12;  // 16B 空闲 extent 链表
constexpr uint32_t FSEG_NOT_FULL         = 28;  // 16B 部分使用 extent 链表
constexpr uint32_t FSEG_FULL             = 44;  // 16B 满 extent 链表
constexpr uint32_t FSEG_MAGIC_N          = 60;  // 4B  魔数 97937874
constexpr uint32_t FSEG_FRAG_ARR         = 64;  // 32 × 4B = 128B，段早期借的零散页
#define FSEG_INODE_SIZE (16 + 3*FLST_BASE_NODE_SIZE + 32*4)   // = 192
constexpr uint32_t FSEG_MAGIC_N_VALUE = 97937874;

/* ★ INODE 页自身也是链表节点：偏移 38 处是 12B 的 FLST node */
constexpr uint32_t FSEG_INODE_PAGE_NODE = FSEG_PAGE_DATA;      // = 38
constexpr uint32_t FSEG_ARR_OFFSET      = FSEG_PAGE_DATA + FLST_NODE_SIZE;  // = 50

/* 一个 INODE 页能放几个 entry —— 是函数不是常量！ */
static inline uint32_t FSP_SEG_INODES_PER_PAGE(page_size_t page_size) {
  return (page_size.physical() - FSEG_ARR_OFFSET - 10) / FSEG_INODE_SIZE;
}
```

> ★★ **"85 个 inode"只对 16K 页成立**——`FSP_SEG_INODES_PER_PAGE()` 是**函数**不是常量：4K→21、8K→42、16K→**85**、32K→170、64K→341。
> ★ **INODE 页的回收是"墓碑 + 整页归还"两级**：`fsp_free_seg_inode()` 先把 `FSEG_ID` 写 0、魔数改写成 `0xfa051ce3`（墓碑值），再判断整页是否还有在用 entry——没有才 `fsp_free_page()` 把整页还给表空间。

**★★ `fseg_create_general()`：创建一个段的完整流程**（`fsp0fsp.cc:2277`）：

```cpp
  if (!has_done_reservation) {
    /* ★ 预留 2 个 extent：1 个给 inode 页，1 个给段自己 */
    if (!fsp_reserve_free_extents(&n_reserved, space_id, 2, alloc_type, mtr)) return nullptr;
  }

  inode = fsp_alloc_seg_inode(space_header, mtr);          /* ① 分配/复用一个 inode entry */
  if (inode == nullptr) goto funct_exit;

  seg_id = mach_read_from_8(space_header + FSP_SEG_ID);    /* ② 段 ID 从 FSP_SEG_ID(+72) 单调分配 */
  mlog_write_ull(space_header + FSP_SEG_ID, seg_id + 1, mtr);
  mlog_write_ull(inode + FSEG_ID, seg_id, mtr);            /* ③ 写 FSEG_ID —— 这一写即"占用"该 slot */

  fseg_inode.write_not_full_n_used(0);                     /* ④ */
  flst_init(inode + FSEG_FREE, mtr);                       /* ⑤ 三条 extent 链表全部清空 */
  flst_init(inode + FSEG_NOT_FULL, mtr);
  flst_init(inode + FSEG_FULL, mtr);
  mlog_write_ulint(inode + FSEG_MAGIC_N, FSEG_MAGIC_N_VALUE, MLOG_4BYTES, mtr);  /* ⑥ */
  for (i = 0; i < FSEG_FRAG_ARR_N_SLOTS; i++) {
    fseg_set_nth_frag_page_no(inode, i, FIL_NULL, mtr);    /* ⑦ 32 个碎片页 slot 全填 FIL_NULL */
  }

  if (page == 0) {
    /* ★ 情形 B：page==0 ⇒ 段自己先申请一页，segment header 就写在自己这一页上 */
    block = fseg_alloc_free_page_low(space, page_size, inode, 0, FSP_UP,
                                     RW_SX_LATCH, mtr, mtr ...);
    header = byte_offset + buf_block_get_frame(block);
    mlog_write_ulint(frame + FIL_PAGE_TYPE, FIL_PAGE_TYPE_SYS, MLOG_2BYTES, mtr);
  }

  mlog_write_ulint(header + FSEG_HDR_OFFSET,  page_offset(inode), MLOG_2BYTES, mtr);  /* ⑧ +8，2B */
  mlog_write_ulint(header + FSEG_HDR_PAGE_NO, page_get_page_no(page_align(inode)), MLOG_4BYTES, mtr); /* ⑨ +4，4B */
  mlog_write_ulint(header + FSEG_HDR_SPACE,   space_id, MLOG_4BYTES, mtr);            /* ⑩ +0，4B */
```

> ★★★ **`page == 0` 这个参数决定段的第一页是谁**：传 0 时段自己申请一页并把 header 写在上面——**B+ 树的根页就是这么来的**（`btr_create()` 建 TOP 段时传 `space, 0, PAGE_HEADER + PAGE_BTR_SEG_TOP`）。传非 0 时（如建 LEAF 段）把 header 写在**已存在的**根页上。
> ⚠️ 真名是 `FSEG_HDR_SPACE`（不是 `FSEG_HDR_SPACE_ID`）。

### segment header → inode → fseg：10 字节指针的完整解析链

B+ 树根页里内嵌两个 fseg header（叶子段 + 非叶子段），每个 10 字节。和 XDES/inode 一样，`fseg_header_t` 也只是 `typedef byte`（`fsp0types.h`），结构由偏移常量表达：

```cpp
constexpr uint32_t FSEG_HDR_SPACE   = 0;   // 4B：inode 所在 space_id
constexpr uint32_t FSEG_HDR_PAGE_NO = 4;   // 4B：inode 所在 page_no
constexpr uint32_t FSEG_HDR_OFFSET  = 8;   // 2B：inode 页内偏移
constexpr uint32_t FSEG_HEADER_SIZE = 10;
```

```
 fseg header (10B)
 +0    ┌──────────────────────────────┐
       │ FSEG_HDR_SPACE   (4B)        │  space_id（通常与所在页相同，冗余保存）
 +4    ├──────────────────────────────┤
       │ FSEG_HDR_PAGE_NO (4B)        │  inode 页号
 +8    ├──────────────────────────────┤
       │ FSEG_HDR_OFFSET  (2B)        │  inode 页内字节偏移
+10    └──────────────────────────────┘
```

在根页（`FIL_PAGE_INDEX`）里的位置：

```
 page header ...  +36  ┌────────────────────────┐
                       │ PAGE_BTR_SEG_LEAF 10B  │  叶子段（leaf segment）
                  +46  ├────────────────────────┤
                       │ PAGE_BTR_SEG_TOP  10B  │  非叶子段（top segment）
                  +56  ├────────────────────────┤
                       │ PAGE_DATA（用户数据区）  │  = PAGE_HEADER + 36 + 2*FSEG_HEADER_SIZE
```

即 `PAGE_BTR_SEG_LEAF = 36`、`PAGE_BTR_SEG_TOP = 36 + FSEG_HEADER_SIZE = 46`，两者**只在根页有效**（非根页的这 20 字节是用户数据区）。一个索引两个段：叶子段管所有叶子页、非叶子段管所有内节点页——这就是 `fseg header` 存在的全部理由。

**fseg header 不直接管理任何页**——它只是指向段 inode 的 (space, page, offset) 三元组。要操作段，先经 fseg header 找到 inode，再操作 inode 里的三个链表与碎片页数组。这一层间接（indirection）的意义：段 inode 可以随 inode 页被换出/移动，而根页里的 10 字节指针恒不变。

这条解析链的实际代码（`fseg_inode_try_get`）：

```cpp
static fseg_inode_t *fseg_inode_try_get(const fseg_header_t *header,
                                        space_id_t space,
                                        const page_size_t &page_size,
                                        mtr_t *mtr, buf_block_t **block) {
  fil_addr_t inode_addr;
  fseg_inode_t *inode;

  inode_addr.page = mach_read_from_4(header + FSEG_HDR_PAGE_NO);   // 读三元组
  inode_addr.boffset = mach_read_from_2(header + FSEG_HDR_OFFSET);
  ut_ad(space == mach_read_from_4(header + FSEG_HDR_SPACE));

  inode = fut_get_ptr(space, page_size, inode_addr, RW_SX_LATCH, mtr, block);
                                                  // (page, offset) → 页内指针，加锁
  if (UNIV_UNLIKELY(!mach_read_from_8(inode + FSEG_ID))) {
    inode = nullptr;                              // 段已释放（ID 归零）
  } else {
    ut_ad(mach_read_from_4(inode + FSEG_MAGIC_N) == FSEG_MAGIC_N_VALUE); // 魔数校验
  }
  return (inode);
}
```

四步：读三元组 → `fut_get_ptr` 把 (page_no, offset) 解析成 inode 页内指针（顺带对该页加 SX 锁，mtr 记账）→ `FSEG_ID` 非 0 校验段未释放 → `FSEG_MAGIC_N` 魔数校验页没写坏。**整条链路的两个校验点**（段 ID + 魔数）就是崩溃恢复/调试时最常见的两个断言失败源。

#### 段创建：fseg_create_general 逐段解析（`fsp0fsp.cc:2277`）

```cpp
buf_block_t *fseg_create_general(space_id_t space_id, page_no_t page,
                                 ulint byte_offset,
                                 bool has_done_reservation, mtr_t *mtr) {
  ut_ad(byte_offset + FSEG_HEADER_SIZE <= UNIV_PAGE_SIZE - FIL_PAGE_DATA_END);
  fil_space_t *space = fil_space_get(space_id);
  mtr_x_lock_space(space, mtr);                       // ① 全程持空间 X 锁

  if (page != 0) {                                    // ② 指定了宿主页（如 B+ 树 root）
    block = buf_page_get(page_id_t(space_id, page), page_size, RW_SX_LATCH, ..., mtr);
    header = byte_offset + buf_block_get_frame(block);
    const auto type = space_id == TRX_SYS_SPACE && page == TRX_SYS_PAGE_NO
                          ? FIL_PAGE_TYPE_TRX_SYS : FIL_PAGE_TYPE_SYS;
    buf_block_reset_page_type_on_mismatch(*block, type, *mtr);
  }

  /* This thread did not own the latch before this call: free excess pages
     from the insert buffer free list */
  if (rw_lock_get_x_lock_count(&space->latch) == 1) {  // ③ 首个持锁者顺手清 ibuf
    if (space_id == IBUF_SPACE_ID) ibuf_free_excess_pages();
  }

  if (!has_done_reservation) {                         // ④ 预留 2 个 extent
    fsp_reserve_t alloc_type = (fsp_is_undo_tablespace(space_id) ? FSP_UNDO : FSP_NORMAL);
    if (!fsp_reserve_free_extents(&n_reserved, space_id, 2, alloc_type, mtr))
      return nullptr;
  }
```

② `page != 0` 表示**段头要写在已有的页里**（B+ 树 root 页的 `PAGE_BTR_SEG_TOP`/`PAGE_BTR_SEG_LEAF`）；`page == 0` 表示段要自己新分配一页。注意对系统表空间的 TRX_SYS 页有专门的页类型分支。

③ ★ `rw_lock_get_x_lock_count == 1` 判断"本线程是不是第一个拿到这个 latch 的"——只有首个持锁者才顺手做 ibuf 清理，避免每个调用者都做。

```cpp
  space_header = fsp_get_space_header(space_id, page_size, mtr);
  inode = fsp_alloc_seg_inode(space_header, mtr);      // ⑤ 分配 inode entry
  if (inode == nullptr) goto funct_exit;

  seg_id = mach_read_from_8(space_header + FSP_SEG_ID);
  mlog_write_ull(space_header + FSP_SEG_ID, seg_id + 1, mtr);   // ⑥ 段号递增
  mlog_write_ull(inode + FSEG_ID, seg_id, mtr);

  { /* Introducing a new scope to localize this object. Otherwise, I have to
       declare this object before the goto statement above. */
    File_segment_inode fseg_inode(space_id, page_size, inode, mtr);
    fseg_inode.write_not_full_n_used(0);               // ⑦ NOT_FULL_N_USED = 0
  }

  flst_init(inode + FSEG_FREE, mtr);                   // ⑧ 三条 extent 链表初始化
  flst_init(inode + FSEG_NOT_FULL, mtr);
  flst_init(inode + FSEG_FULL, mtr);

  mlog_write_ulint(inode + FSEG_MAGIC_N, FSEG_MAGIC_N_VALUE, MLOG_4BYTES, mtr);
  for (i = 0; i < FSEG_FRAG_ARR_N_SLOTS; i++)          // ⑨ 32 个碎片页槽置 FIL_NULL
    fseg_set_nth_frag_page_no(inode, i, FIL_NULL, mtr);

  if (page == 0) {                                     // ⑩ 段自建首页
    block = fseg_alloc_free_page_low(space, page_size, inode, 0, FSP_UP,
                                     RW_SX_LATCH, mtr, mtr IF_DEBUG(, has_done_reservation));
    if (block == nullptr) {
      fsp_free_seg_inode(space_id, page_size, inode, mtr);   // 分配失败 → 回滚 inode
      goto funct_exit;
    }
    header = byte_offset + buf_block_get_frame(block);
    mlog_write_ulint(... + FIL_PAGE_TYPE, FIL_PAGE_TYPE_SYS, MLOG_2BYTES, mtr);
  }

  mlog_write_ulint(header + FSEG_HDR_OFFSET,  page_offset(inode), MLOG_2BYTES, mtr);
  mlog_write_ulint(header + FSEG_HDR_PAGE_NO, page_get_page_no(page_align(inode)),
                   MLOG_4BYTES, mtr);
  mlog_write_ulint(header + FSEG_HDR_SPACE,   space_id, MLOG_4BYTES, mtr);

funct_exit:
  if (!has_done_reservation) fil_space_release_free_extents(space_id, n_reserved);
  return block;
}
```

几个细节：

- ⑦ 那段注释很可爱——**为了让对象声明能放在 `goto` 之后**，特意加了一层作用域（C++ 不允许跨过初始化跳转到其作用域内）。
- ⑩ 若段需要自建首页但分配失败，会调 `fsp_free_seg_inode` **回滚刚分配的 inode**，不留垃圾。
- 三步辅助宏：`page_offset(inode)` 取 inode 在页内偏移、`page_align(inode)` 取所在页起始地址、`page_get_page_no()` 取页号——正好凑齐 fseg header 的 `(space, page_no, offset)` 三元组。

**释放段**（`fsp_free_seg_inode`）：`FSEG_ID` 归零、magic 改写 `0xfa051ce3`，inode 页在 FULL/FREE 链表间迁移，整页无段则 `fsp_free_page` 释放。

### 分配算法（一）：fseg_alloc_free_page_low 的七分支

段内分配页的完整决策链（"intelligent allocation strategy which tries to minimize file space fragmentation"）：

```cpp
static buf_block_t *fseg_alloc_free_page_low(
    fil_space_t *space, const page_size_t &page_size, fseg_inode_t *seg_inode,
    page_no_t hint, byte direction, rw_lock_type_t rw_latch, mtr_t *mtr,
    mtr_t *init_mtr IF_DEBUG(, bool has_done_reservation)) {
  ...
  reserved = fseg_n_reserved_pages_low(space_id, page_size, seg_inode, &used, mtr);

  descr = xdes_get_descriptor_with_space_hdr(space_header, space_id, hint, mtr);
  if (descr == nullptr) {
    hint = 0;                                        // hint 越界：重置
    descr = xdes_get_descriptor(space_id, hint, page_size, mtr);
  }

  /* 分支 1：hinted 页所在 extent 属于本段，且该页空闲 → 直接用 */
  if (xdes_in_segment(descr, seg_id, mtr) &&
      (xdes_mtr_get_bit(descr, XDES_FREE_BIT, hint % FSP_EXTENT_SIZE, mtr))) {
    ret_descr = descr;
    ret_page = hint;
    goto got_hinted_page;
  }

  /* 分支 2：hinted extent 是 FREE 态、段空间不足（reserve 触发）、且已过碎片期
     → 整 extent 进段 FREE 链，取 hinted 页 */
  if (xdes_get_state(descr, mtr) == XDES_FREE &&
      reserved - used < reserved * (fseg_reserve_pct / 100) &&
      used >= FSEG_FRAG_LIMIT) {
    ret_descr = fsp_alloc_free_extent(space_id, page_size, hint, mtr);
    ut_a(ret_descr == descr);
    xdes_set_segment_id(ret_descr, seg_id, XDES_FSEG, mtr);
    flst_add_last(seg_inode + FSEG_FREE, ret_descr + XDES_FLST_NODE, mtr);
    fseg_fill_free_list(seg_inode, space_id, page_size, hint + FSP_EXTENT_SIZE, mtr);
    goto take_hinted_page;
  }

  /* 分支 3：方向分配（弃用 hint）——reserve 触发时，FSP_DOWN 取 extent 末页、
     FSP_UP 取首页；lease 的 extent 找空闲页 */
  if (direction != FSP_NO_DIR &&
      reserved - used < reserved * (fseg_reserve_pct / 100) &&
      used >= FSEG_FRAG_LIMIT) {
    ret_descr = fseg_alloc_free_extent(seg_inode, space_id, page_size, mtr);
    if (ret_descr) {
      ret_page = xdes_get_offset(ret_descr);
      if (direction == FSP_DOWN) {
        ret_page += FSP_EXTENT_SIZE - 1;
      } else if (xdes_get_state(ret_descr, mtr) == XDES_FSEG_FRAG) {
        ret_page += xdes_find_bit(ret_descr, XDES_FREE_BIT, true, 0, mtr);  // lease
      }
    }
  }

  if (ret_page == FIL_NULL) {
    /* 分支 4：hinted extent 属于本段且未满 → 取 hinted 页 */
    if (xdes_in_segment(descr, seg_id, mtr) && (!xdes_is_full(descr, mtr))) {
      ret_descr = descr;
      ret_page = xdes_get_offset(ret_descr) +
                 xdes_find_bit(ret_descr, XDES_FREE_BIT, true, hint % FSP_EXTENT_SIZE, mtr);
    } else if (used < reserved) {
      /* 分支 5：used < reserved → 从段内 NOT_FULL 或 FREE 链取一页 */
      fil_addr_t first;
      if (flst_get_len(seg_inode + FSEG_NOT_FULL) > 0) {
        first = flst_get_first(seg_inode + FSEG_NOT_FULL, mtr);
      } else if (flst_get_len(seg_inode + FSEG_FREE) > 0) {
        first = flst_get_first(seg_inode + FSEG_FREE, mtr);
      } else {
        ut_ad(!has_done_reservation);
        return (nullptr);
      }
      ret_descr = xdes_lst_get_descriptor(space_id, page_size, first, mtr);
      ret_page = xdes_get_offset(ret_descr) +
                 xdes_find_bit(ret_descr, XDES_FREE_BIT, true, 0, mtr);
    } else if (used < FSEG_FRAG_LIMIT) {
      /* 分支 6：碎片期（used < 32）→ 表空间逐页借碎片页，写进 inode 的 frag 数组 */
      buf_block_t *block = fsp_alloc_free_page(space_id, page_size, hint,
                                               rw_latch, mtr, init_mtr);
      if (block != nullptr) {
        n = fseg_find_free_frag_page_slot(seg_inode, mtr);
        ut_a(n != ULINT_UNDEFINED);
        fseg_set_nth_frag_page_no(seg_inode, n, block->page.id.page_no(), mtr);
      }
      return (block);
    } else {
      /* 分支 7：分配新 extent 取首页（lease 则取第一个空闲页） */
      ret_descr = fseg_alloc_free_extent(seg_inode, space_id, page_size, mtr);
      if (ret_descr == nullptr) {
        ret_page = FIL_NULL;
      } else {
        const xdes_state_t state = xdes_get_state(ret_descr, mtr);
        ret_page = xdes_get_offset(ret_descr);
        if (state == XDES_FSEG_FRAG) {
          ret_page += xdes_find_bit(ret_descr, XDES_FREE_BIT, true, 0, mtr);
        }
      }
    }
  }
  ...
got_hinted_page:
  if (ret_descr != nullptr) {
    fseg_mark_page_used(space_id, page_size, seg_inode, ret_page, ret_descr, mtr);
  }
  return (fsp_page_create(page_id_t(space_id, ret_page), page_size, rw_latch,
                          mtr, init_mtr));
}
```

七个分支的优先级排序值得背下来：

1. **hint 优先**（分支 1/2/4 都在努力用 hint 页）——顺序插入时 hint 往往是"上一页 +1"，命中即物理连续；
2. **reserve factor 触发才整 extent 扩张**（分支 2/3 的条件 `reserved - used < reserved * (fseg_reserve_pct/100)`）——空闲占比跌破 12.5% 才向 FSP 要地；
3. **碎片期（used < 32）走碎片页**（分支 6），之后永远走 extent（分支 5/7）——两档分配的切换点；
4. **FSP_DOWN 取 extent 末页、FSP_UP 取首页**（分支 3）——降序插入从 extent 尾部吃起，升序从头部，避免两个方向的插入互相踩。

### 分配算法（二）：fsp_alloc_free_page —— 碎片页的借与还

分支 6 调用的 `fsp_alloc_free_page` 是**表空间层**的逐页分配（碎片页专用）：

```cpp
static buf_block_t *fsp_alloc_free_page(
    space_id_t space, const page_size_t &page_size, page_no_t hint,
    rw_lock_type_t rw_latch, mtr_t *mtr, mtr_t *init_mtr) {
  header = fsp_get_space_header(space, page_size, mtr);

  /* 优先取 hinted 页所在 extent（必须是 FREE_FRAG 态） */
  descr = xdes_get_descriptor_with_space_hdr(header, space, hint, mtr);
  if (descr && (xdes_get_state(descr, mtr) == XDES_FREE_FRAG)) {
    /* Ok, we can take this extent */
  } else {
    /* 否则取 FREE_FRAG 链表头 */
    first = flst_get_first(header + FSP_FREE_FRAG, mtr);
    if (fil_addr_is_null(first)) {
      /* FREE_FRAG 空：从 FSP_FREE 拿一个整 extent，翻成 FREE_FRAG 挂进链 */
      descr = fsp_alloc_free_extent(space, page_size, hint, mtr);
      if (descr == nullptr) return (nullptr);       // 空间耗尽
      xdes_set_state(descr, XDES_FREE_FRAG, mtr);
      flst_add_last(header + FSP_FREE_FRAG, descr + XDES_FLST_NODE, mtr);
    } else {
      descr = xdes_lst_get_descriptor(space, page_size, first, mtr);
    }
    hint = 0;
  }

  /* 在 extent 内找空闲页（从 hint 位置起找） */
  free = xdes_find_bit(descr, XDES_FREE_BIT, true, hint % FSP_EXTENT_SIZE, mtr);
  ...
  page_no = xdes_get_offset(descr) + free;          // 区首 + 区内偏移 = 真实页号

  fsp_alloc_from_free_frag(header, descr, free, mtr);   // 置位 + FRAG_N_USED++ + 满则迁移链
  return (fsp_page_create(page_id_t(space, page_no), page_size, rw_latch, mtr, init_mtr));
}
```

归还方向（`fsp_free_page`）的链表迁移是状态机的完整收尾：

```cpp
state = xdes_get_state(descr, mtr);
...
xdes_set_bit(descr, XDES_FREE_BIT, bit, true, mtr);      // 释放页

frag_n_used = mtr_read_ulint(header + FSP_FRAG_N_USED, MLOG_4BYTES, mtr);
if (state == XDES_FULL_FRAG) {
  /* 碎片 extent 满 → 释放一页后回到 FREE_FRAG 链 */
  flst_remove(header + FSP_FULL_FRAG, descr + XDES_FLST_NODE, mtr);
  xdes_set_state(descr, XDES_FREE_FRAG, mtr);
  flst_add_last(header + FSP_FREE_FRAG, descr + XDES_FLST_NODE, mtr);
  mlog_write_ulint(header + FSP_FRAG_N_USED, frag_n_used + FSP_EXTENT_SIZE - 1, MLOG_4BYTES, mtr);
} else {
  ut_a(frag_n_used > 0);
  mlog_write_ulint(header + FSP_FRAG_N_USED, frag_n_used - 1, MLOG_4BYTES, mtr);
}

if (xdes_is_free(descr, mtr)) {
  /* 整个 extent 全空 → 摘出 FREE_FRAG，还给 FSP_FREE */
  flst_remove(header + FSP_FREE_FRAG, descr + XDES_FLST_NODE, mtr);
  fsp_free_extent(page_id, page_size, mtr);
}
```

两处 `mlog_write_ulint` 说明**链表迁移和计数器变化都记 redo**——空间元数据必须崩溃一致，否则恢复后 free 空间账目错乱。

### 分配算法（三）：fseg_alloc_free_extent —— 段从表空间要 extent

```cpp
static xdes_t *fseg_alloc_free_extent(fseg_inode_t *inode, space_id_t space,
                                      const page_size_t &page_size, mtr_t *mtr) {
  if (flst_get_len(inode + FSEG_FREE) > 0) {
    /* 段内 FREE 链非空：直接取链头 */
    first = flst_get_first(inode + FSEG_FREE, mtr);
    descr = xdes_lst_get_descriptor(space, page_size, first, mtr);
  } else {
    /* 段内 FREE 链空：先尝试 lease（fsp 的 free_frag） */
    descr = fsp_alloc_xdes_free_frag(space, inode, page_size, mtr);
    if (descr != nullptr) return (descr);

    /* lease 不成：从表空间 FREE 链拿整 extent */
    descr = fsp_alloc_free_extent(space, page_size, 0, mtr);
    if (descr == nullptr) return (nullptr);

    seg_id = mach_read_from_8(inode + FSEG_ID);
    xdes_set_segment_id(descr, seg_id, XDES_FSEG, mtr);
    flst_add_last(inode + FSEG_FREE, descr + XDES_FLST_NODE, mtr);

    /* 顺带把段 FREE 链填到 4 个（连续 extent，条件允许时） */
    fseg_fill_free_list(inode, space, page_size,
                        xdes_get_offset(descr) + FSP_EXTENT_SIZE, mtr);
  }
  return (descr);
}
```

三层递进：**段内 FREE 链 → lease（FSP 的 free_frag）→ 表空间 FREE 链**。lease 的完整实现（`fsp_alloc_xdes_free_frag`）：

```cpp
static xdes_t *fsp_alloc_xdes_free_frag(space_id_t space, fseg_inode_t *inode,
                                        const page_size_t &page_size, mtr_t *mtr) {
  /* 取 FREE_FRAG 链尾的 extent（链尾通常是最"新"归还的） */
  descr = fsp_get_last_free_frag_extent(header, page_size, mtr);
  if (!descr) return (nullptr);

  /* 判据：是 XDES 页组首 extent + 页 0/1 被占（XDES + ibuf bitmap）+ 其余全空 */
  if (!xdes_is_leasable(descr, page_size, mtr)) return (nullptr);
  ut_ad(xdes_get_n_used(descr, mtr) == XDES_FRAG_N_USED);

  /* 从 FSP_FREE_FRAG 摘除，转移所有权 */
  flst_remove(header + FSP_FREE_FRAG, descr + XDES_FLST_NODE, mtr);
  n_used = mtr_read_ulint(header + FSP_FRAG_N_USED, MLOG_4BYTES, mtr);
  mlog_write_ulint(header + FSP_FRAG_N_USED, n_used - XDES_FRAG_N_USED, MLOG_4BYTES, mtr);

  seg_id = mach_read_from_8(inode + FSEG_ID);
  xdes_set_segment_id(descr, seg_id, XDES_FSEG_FRAG, mtr);   // ★ 状态翻转

  /* 挂进段的 NOT_FULL 链（页 0/1 已用，其余供段分配） */
  flst_add_last(inode + FSEG_NOT_FULL, descr + XDES_FLST_NODE, mtr);
  File_segment_inode fseg_inode(space, page_size, inode, mtr);
  n_used = fseg_inode.read_not_full_n_used();
  fseg_inode.write_not_full_n_used(static_cast<uint32_t>(n_used + XDES_FRAG_N_USED));
  return (descr);
}
```

lease 的**判据**（`xdes_is_leasable`）：①extent 是"每 256 extent 一组"的组首（`!ut_2pow_remainder(page_no, page_size.physical())`，即含有 XDES 页）；②页 0/页 1 都不空闲（被 XDES 页和 ibuf bitmap 页占着）；③其余页全空闲。此时整 extent 租给段最划算——段白得 62 页，FSP 甩掉一个"永远半空"的碎片 extent。

### 分配算法（四）：fseg_mark_page_used —— 置位与链表迁移

```cpp
static void fseg_mark_page_used(space_id_t space_id, const page_size_t &page_size,
                                fseg_inode_t *seg_inode, page_no_t page,
                                xdes_t *descr, mtr_t *mtr) {
  if (xdes_is_free(descr, mtr)) {
    /* extent 此前全空：FREE 链 → NOT_FULL 链 */
    flst_remove(seg_inode + FSEG_FREE, descr + XDES_FLST_NODE, mtr);
    flst_add_last(seg_inode + FSEG_NOT_FULL, descr + XDES_FLST_NODE, mtr);
  }

  /* 页置位：空闲 → 占用 */
  xdes_set_bit(descr, XDES_FREE_BIT, page % FSP_EXTENT_SIZE, false, mtr);

  File_segment_inode fseg_inode(space_id, page_size, seg_inode, mtr);
  not_full_n_used = fseg_inode.read_not_full_n_used();
  not_full_n_used++;
  fseg_inode.write_not_full_n_used(not_full_n_used);

  if (xdes_is_full(descr, mtr)) {
    /* extent 满了：NOT_FULL 链 → FULL 链，计数扣掉整个 extent */
    flst_remove(seg_inode + FSEG_NOT_FULL, descr + XDES_FLST_NODE, mtr);
    flst_add_last(seg_inode + FSEG_FULL, descr + XDES_FLST_NODE, mtr);
    ut_ad(not_full_n_used >= FSP_EXTENT_SIZE);
    fseg_inode.write_not_full_n_used(not_full_n_used - FSP_EXTENT_SIZE);
  }
}
```

段内三链的状态流转闭环：**FREE →(取首页) NOT_FULL →(64 页用满) FULL**。`FSEG_NOT_FULL_N_USED` 计数器在这里维护：每用一页 +1，extent 转 FULL 时一次性 -64——这就是 `fseg_n_reserved_pages_low` 里 `used = n_frags + n_total_full + n_used_not_full` 能 O(1) 算出来的记账基础。

#### 段释放：fseg_free_page_low 逐段解析（`fsp0fsp.cc:3379`）

**① 前置校验与重复释放检测**

```cpp
ut_ad(mach_read_from_4(seg_inode + FSEG_MAGIC_N) == FSEG_MAGIC_N_VALUE);   // 魔数
ut_ad(!((page_offset(seg_inode) - FSEG_ARR_OFFSET) % FSEG_INODE_SIZE));    // inode 对齐

if (ahi) btr_search_drop_page_hash_when_freed(page_id, page_size);         // AHI 清理

descr = xdes_get_descriptor(page_id.space(), page_id.page_no(), page_size, mtr);

if (xdes_mtr_get_bit(descr, XDES_FREE_BIT, page_id.page_no() % FSP_EXTENT_SIZE, mtr)) {
  fputs("InnoDB: Dump of the tablespace extent descriptor: ", stderr);
  ut_print_buf(stderr, descr, 40);                       // 打印 40 字节 XDES
  ib::error(ER_IB_MSG_421) << "InnoDB is trying to free page " << page_id
      << " though it is already marked as free in the tablespace! ...";
crash:
  ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_422) << FORCE_RECOVERY_MSG;        // ★ 故意崩溃
}
```

★ **重复释放 = 直接 `ib::fatal` 崩溃**（而不是报错返回），并先把 40 字节 XDES dump 到 stderr。理由：能走到这里说明"页已标记空闲却还要释放"，空间元数据自相矛盾，继续跑只会扩散损坏——**宁可崩，不可错**。这是 InnoDB 空间管理的一处"硬性自校验"。

**② 按 extent 状态分两条路**

```cpp
switch (state) {
  case XDES_FSEG:
  case XDES_FSEG_FRAG:
    break;                                    // 属于段 → 走下面的 extent 归还逻辑

  case XDES_FREE_FRAG:
  case XDES_FULL_FRAG:                        // ★ 碎片页路径
    for (i = 0;; i++) {
      const page_no_t page_no = fseg_get_nth_frag_page_no(seg_inode, i, mtr);
      if (page_no == page_id.page_no()) {
        fseg_set_nth_frag_page_no(seg_inode, i, FIL_NULL, mtr);   // frag_arr 槽清空
        break;
      }
    }
    fsp_free_page(page_id, page_size, mtr);   // 直接还给表空间
    return;                                    // ← 碎片页到此结束

  case XDES_FREE:
  case XDES_NOT_INITED:
    ut_error;
}
```

**碎片页**（段内 ≤32 页的部分）不走 extent 逻辑：在 `FSEG_FRAG_ARR` 里线性找到该页、置 `FIL_NULL`，然后 `fsp_free_page` 直接归还表空间。注意这个 `for(;;)` 是**无界线性查找**——注释之外值得注意：若页不在数组里会无限循环（由前面的重复释放/段 ID 校验保证不会发生）。

**③ 段 ID 一致性校验**

```cpp
descr_id = xdes_get_segment_id(descr);
seg_id = mach_read_from_8(seg_inode + FSEG_ID);
if (UNIV_UNLIKELY(descr_id != seg_id)) {
  ut_print_buf(stderr, descr, 40);            // dump XDES
  ut_print_buf(stderr, seg_inode, 40);        // dump inode
  ib::error(ER_IB_MSG_423) << "InnoDB is trying to free page " << page_id
      << ", which does not belong to segment " << descr_id
      << " but belongs to segment " << seg_id << ".";
  goto crash;                                  // 同样致命
}
```

**④ `not_full_n_used` 记账与链表迁移**

```cpp
not_full_n_used = fseg_inode.read_not_full_n_used();
if (xdes_is_full(descr, mtr)) {
  /* The fragment is full: move it to another list */
  flst_remove(seg_inode + FSEG_FULL, descr + XDES_FLST_NODE, mtr);
  flst_add_last(seg_inode + FSEG_NOT_FULL, descr + XDES_FLST_NODE, mtr);
  not_full_n_used += FSP_EXTENT_SIZE - 1;      // ★ 原满 extent 释放 1 页后剩 63 空闲
} else {
  ut_a(not_full_n_used > 0);
  not_full_n_used -= 1;
}

const page_no_t bit = page_id.page_no() % FSP_EXTENT_SIZE;
xdes_set_bit(descr, XDES_FREE_BIT, bit, true, mtr);
xdes_set_bit(descr, XDES_CLEAN_BIT, bit, true, mtr);
page_no_t n_used = xdes_get_n_used(descr, mtr);
```

★ `not_full_n_used` 是**段在 NOT_FULL 链上已用页数的运行账本**（O(1) 判断段是否"够满"以决定分配策略，见前文 `fseg_alloc_free_page_low` 的 `reserved - used < reserved * fseg_reserve_pct/100` 判据）。extent 从 FULL 降级到 NOT_FULL 时要一次性加回 `EXTENT_SIZE - 1`，不是 +1。

**⑤ ★★ lease 归还：借来的碎片 extent 怎么还**

```cpp
/* 若 extent 是 XDES_FSEG_FRAG，释放的页一定不是 page 0 / page 1
   ——那是 FSP 的元数据页（XDES 页 + ibuf bitmap），不归段 */
ut_ad(state != XDES_FSEG_FRAG || (bit != 0 && bit != 1));
ut_ad(state != XDES_FSEG_FRAG || n_used > 1);

/* 可归还租约的判据 */
ut_ad(xdes_is_leasable(descr, page_size, mtr) ==
      (state == XDES_FSEG_FRAG && n_used == XDES_FRAG_N_USED));

/* A leased fragment extent might have no more pages belonging to the segment. */
if (state == XDES_FSEG_FRAG && n_used == XDES_FRAG_N_USED) {
  n_used = 0;                                  // ★ 只剩 FSP 自己的 2 个元数据页 ⇒ 视同全空
  ut_ad(not_full_n_used >= XDES_FRAG_N_USED);
  not_full_n_used -= XDES_FRAG_N_USED;
}
```

这段把 lease 机制讲透了：段从 `FSP_FREE_FRAG` 借来的 extent，**页 0/1 属于 FSP 自己**（XDES 页 + ibuf bitmap），段只用了页 2~63。所以当段用完后，`n_used` 剩下的是 `XDES_FRAG_N_USED`（=2），**不能真的等到 0**——代码把"n_used == 2 且状态是 FSEG_FRAG"直接**视同全空**（`n_used = 0`），从而触发下面的归还。

**⑥ 全空则还给表空间（不回段内 FREE 链）**

```cpp
if (n_used == 0) {
  /* The extent has become free: free it to space */
  flst_remove(seg_inode + FSEG_NOT_FULL, descr + XDES_FLST_NODE, mtr);
  fsp_free_extent(page_id, page_size, mtr);    // ★ 还给表空间，不是段内 FREE 链
}

fseg_inode.write_not_full_n_used(not_full_n_used);
```

★ 这就是"归还方向"的代码证据（呼应 `fseg_free_page_low` 注释原文："即使 extent 空了也是移动到 fsp_free/fsp_free_frag，而不是 fseg free！"）：段释放页后，**extent 全空时直接还给表空间**（`fsp_free_extent`），**从不退回段内 `FSEG_FREE` 链**。段内 FREE 链只有"进"（`fseg_fill_free_list` 一次预存 4 个连续 extent），没有"回"口。

### 段内 FREE 链的填充与 reserve factor 计算

```cpp
static void fseg_fill_free_list(fseg_inode_t *inode, space_id_t space,
                                const page_size_t &page_size, page_no_t hint,
                                mtr_t *mtr) {
  reserved = fseg_n_reserved_pages_low(space, page_size, inode, &used, mtr);

  if (reserved < FSEG_FREE_LIST_LIMIT * FSP_EXTENT_SIZE) {
    return;                                   // 段太小（< 40 extent）：不给 FREE 链预存
  }
  if (flst_get_len(inode + FSEG_FREE) > 0) {
    return;                                   // FREE 链非空：不补
  }

  for (i = 0; i < FSEG_FREE_LIST_MAX_LEN; i++) {   // 一次最多 4 个
    descr = xdes_get_descriptor(space, hint, page_size, mtr);
    if ((descr == nullptr) || (XDES_FREE != xdes_get_state(descr, mtr))) {
      return;                                   // 拿不到连续 extent：停止
    }
    descr = fsp_alloc_free_extent(space, page_size, hint, mtr);
    seg_id = mach_read_from_8(inode + FSEG_ID);
    xdes_set_segment_id(descr, seg_id, XDES_FSEG, mtr);
    flst_add_last(inode + FSEG_FREE, descr + XDES_FLST_NODE, mtr);
    hint += FSP_EXTENT_SIZE;                    // 只拉 hint 之后的连续 extent
  }
}
```

reserve factor 的核心计算（`fseg_n_reserved_pages_low`）：

```cpp
uint32_t n_used_not_full = fseg_inode.read_not_full_n_used();
ulint n_total_not_full = FSP_EXTENT_SIZE * flst_get_len(inode + FSEG_NOT_FULL);
ulint n_total_full     = FSP_EXTENT_SIZE * flst_get_len(inode + FSEG_FULL);
ulint n_total_free     = FSP_EXTENT_SIZE * flst_get_len(inode + FSEG_FREE);
ulint n_frags          = fseg_get_n_frag_pages(inode, mtr);

*used = n_frags + n_total_full + n_used_not_full;                    // 已用页
ret   = n_frags + n_total_full + n_total_free + n_total_not_full;    // 预留页
ut_ad(*used <= ret);
```

全部 O(1)（链表长度 + 一个计数器 + 碎片页数组扫描），这正是 `fseg_alloc_free_page_low` 分支 2/3 里 reserve 判定（`reserved - used < reserved * 12.5%`）的输入。**分配热路径上零扫描**——这是碎片页数组限 32 个（扫描 O(32)）和 NOT_FULL 计数器的设计本意。

### 表空间 DDL：CREATE / ALTER(RENAME/ADD/DROP DATAFILE) / DROP

SQL 层**每种表空间语句一个 `Sql_cmd` 类**（`sql_tablespace.h/cc`），经 handlerton 回调进入 InnoDB：

| SQL | 执行类 | 引擎入口 |
|---|---|---|
| `CREATE TABLESPACE ... ADD DATAFILE` | `Sql_cmd_create_tablespace::execute`（sql_tablespace.cc:452） | `innodb_create_tablespace`（ha_innodb.cc:15583）→ `dict_build_tablespace` |
| `ALTER TABLESPACE ... RENAME TO` | `Sql_cmd_alter_tablespace_rename::execute`（:1231） | `innobase_alter_tablespace`（ha_innodb.cc:16436） |
| `ALTER TABLESPACE ... ADD/DROP DATAFILE` | `Sql_cmd_alter_tablespace_add_datafile`（:1057）/ `_drop_datafile`（:1148） | 同上 |
| `ALTER TABLESPACE ... AUTOEXTEND_SIZE = N` | `Sql_cmd_alter_tablespace::execute`（:906） | 同上 |
| `DROP TABLESPACE` | — | `innobase_drop_tablespace` |

CREATE 的调用栈：

```
dispatch_command → mysql_execute_command
  → Sql_cmd_create_tablespace::execute        (sql_tablespace.cc:452)
    → innodb_create_tablespace                (ha_innodb.cc:15583, handlerton 回调)
      → dict_build_tablespace                 (dict0crea.cc)
        → fil_ibd_create                      (建物理文件 + 写 space flags)
        → fsp_header_init                     (初始化页 0：FSP header + fsp_fill_free_list)
```

要点：

- **AUTOEXTEND_SIZE 的 ALTER**：`dict0dd.cc::dd_implicit_alter_tablespace` 更新 `dd::Tablespace::options()` 并调 `fil_set_autoextend_size` 刷 `fil_space_t` 内存态（见下文 AUTOEXTEND_SIZE 节）；
- **RENAME** 只改名字（DD + `fil_space_t::name`），文件路径的增减由 ADD/DROP DATAFILE 负责；
- **DROP**：MDL + `dict_sys_mutex`（log0ddl.cc:1689 注释）下清 `fil_space_t` → 删物理文件 → 删 DD 记录。

### 预留协议：fsp_reserve_free_extents 的三分支

所有要分配空间的操作（`fseg_create`、`btr_page_alloc`…）遵循**先预留、后分配**协议——防中途空间耗尽：分配过程会嵌套锁多个页，预留保证不会做到一半才回滚。签名（`fsp0fsp.cc:3127`）：

```cpp
bool fsp_reserve_free_extents(ulint *n_reserved, space_id_t space_id,
                              ulint n_ext, fsp_reserve_t alloc_type,
                              mtr_t *mtr, page_no_t n_pages = 2 /* 按页预留时的数量 */);
```

#### 完整源码逐行解析（`fsp0fsp.cc:3127-3266`）

```cpp
bool fsp_reserve_free_extents(ulint *n_reserved, space_id_t space_id,
                              ulint n_ext, fsp_reserve_t alloc_type, mtr_t *mtr,
                              page_no_t n_pages) {          // n_pages 默认 2
  fsp_header_t *space_header;
  ulint n_free_list_ext;      // FSP_FREE 链上的 extent 数（已初始化且空闲）
  page_no_t free_limit;       // FSP_FREE_LIMIT
  page_no_t size;             // FSP_SIZE
  ulint n_free;               // n_free_list_ext + n_free_up，总可预期空闲
  ulint n_free_up;            // 已扩文件但未初始化的 extent 数（"潜在"空闲）
  ulint reserve;              // 按 alloc_type 计算的安全余量

  *n_reserved = n_ext;                        // ① 乐观预设：按 extent 记账

  fil_space_t *space = fil_space_get(space_id);
  mtr_x_lock_space(space, mtr);               // ② 整个函数持表空间 X 锁（同空间串行）

  const page_size_t page_size(space->flags);
  buf_block_t *block = nullptr;
  space_header = fsp_get_space_header_block(space_id, page_size, mtr, &block);

try_again:                                    // ③ 扩展文件后回跳重算
  size = mach_read_from_4(space_header + FSP_SIZE);
  ut_ad(size == space->size_in_header);       // ④ 磁盘态与内存态必须一致
```

① `*n_reserved = n_ext` 是**乐观默认值**：走 extent 路径时预留量就是请求的 extent 数；后面走"按页"路径会改写成 0。

② **整个函数持表空间 X 锁**——预留是空间级的串行操作，这也是它必须"快"的原因（只碰 FSP header 页）。

③④ `try_again` 是重试入口；`ut_ad(size == space->size_in_header)` 又一次断言磁盘 FSP_SIZE 与 `fil_space_t::size_in_header` 同步（呼应前文"两套并存状态"）。

```cpp
  /* 设置了 AUTOEXTEND_SIZE 的表空间 */
  if (space->autoextend_size_in_bytes > 0) {
    page_no_t autoextend_size_pages = space->autoextend_size_in_bytes / page_size.physical();

    if (size < autoextend_size_pages) {
      goto try_to_extend;                     // ⑤ 还没到 autoextend_size → 先扩文件
    }

    if (size == autoextend_size_pages) {
      xdes_t *descr = xdes_get_descriptor_with_space_hdr(space_header, space->id, 0, mtr);
      page_no_t n_used = xdes_get_n_used(descr, mtr);
      if (n_used < autoextend_size_pages &&
          n_pages < (autoextend_size_pages - (FSP_EXTENT_SIZE / 2))) {
        *n_reserved = 0;                      // ⑥ 改为按页预留
        return fsp_reserve_free_pages(space, space_header, size, mtr, n_pages);
      }
    }
  } else if (size < FSP_EXTENT_SIZE && n_pages < FSP_EXTENT_SIZE / 2) {
    /* Use different rules for small single-table tablespaces */
    *n_reserved = 0;
    bool success = fsp_reserve_free_pages(space, space_header, size, mtr, n_pages);
    if (success) {
      buf_page_t *page = &block->page;
      buf_page_make_old(page);                // ⑦ 让 FSP header 页尽快被刷出
    }
    return success;
  }
```

⑤ 表空间还小于 `AUTOEXTEND_SIZE` → 直接跳去扩文件（扩到该值）。

⑥ 恰好等于 `AUTOEXTEND_SIZE`、且已用页数还少、且请求页数小于 `autoextend − EXTENT/2`（即还剩足够余量）→ **不必扩文件**，降级为**按页预留**（`fsp_reserve_free_pages`），`*n_reserved` 归零（因为不再是 extent 记账）。

⑦ **小表空间规则**（`size < EXTENT && n_pages < EXTENT/2`）：不足一个 extent 的表空间按页预留，避免"为几页数据扩 1MB extent"。★ 成功后调 `buf_page_make_old` 把 FSP header 页移到 LRU old 端——注释明说 "so that it gets flushed at the earliest"，让这个高频修改的关键页尽快落盘。

```cpp
  n_free_list_ext = flst_get_len(space_header + FSP_FREE);
  ut_ad(space->free_len == n_free_list_ext);  // ⑧ 又一次磁盘/内存一致性断言

  free_limit = mtr_read_ulint(space_header + FSP_FREE_LIMIT, MLOG_4BYTES, mtr);
  ut_ad(space->free_limit == free_limit);

  /* 计算 [free_limit, size) 之间还能切出多少完整 extent —— 这些是"文件已覆盖、但尚未初始化 XDES 元数据"的页 */
  if (size >= free_limit) {
    n_free_up = (size - free_limit) / FSP_EXTENT_SIZE;
  } else {
    ut_ad(alloc_type == FSP_BLOB);            // ⑨ 只有 BLOB 分配会走到这个异常分支
    n_free_up = 0;
  }

  if (n_free_up > 0) {
    n_free_up--;                                                  // ⑩ 扣掉组首 extent
    n_free_up -= n_free_up / (page_size.physical() / FSP_EXTENT_SIZE);  // ⑪ 再扣掉每组一个
  }

  n_free = n_free_list_ext + n_free_up;       // ⑫ 总可预期空闲 extent
```

⑧ 第三次一致性断言（`space->free_len` ↔ `FSP_FREE` 链长）——**这些断言本身就是"内存态是磁盘态缓存"这一设计的自证**。

⑨⑩⑪ 是本函数最"物理"的一段：`[free_limit, size)` 之间的页虽然文件里存在，但初始化后**并非全部能进 `FSP_FREE`**——每 256 个 extent 的组首 extent 含 XDES 页（页 0）+ ibuf bitmap 页（页 1），只能进 `FSP_FREE_FRAG`。所以要扣掉：

- `n_free_up--`：扣掉紧邻 `free_limit` 的第一个（它正是下一组的组首）；
- `n_free_up -= n_free_up / 256`：`page_size.physical() / FSP_EXTENT_SIZE = 16384/64 = 256`，每 256 个扣 1 个。

★ 这两行把"**XDES 分组导致组首 extent 不可用**"的物理约束直接编码进了空闲量估算——与前文 `fsp_fill_free_list` 里 `init_xdes` 分支的处理互为印证。

```cpp
  switch (alloc_type) {
    case FSP_NORMAL:
      /* We reserve 1 extent + 0.5 % of the space size to undo logs
      and 1 extent + 0.5 % to cleaning operations; NOTE: this source
      code is duplicated in the function below! */
      reserve = 2 + ((size / FSP_EXTENT_SIZE) * 2) / 200;   // ⑬ 2 extent + 1% 空间
      if (n_free <= reserve + n_ext) goto try_to_extend;
      break;
    case FSP_UNDO:
      /* We reserve 0.5 % of the space size to cleaning operations */
      reserve = 1 + ((size / FSP_EXTENT_SIZE) * 1) / 200;   // ⑭ 1 extent + 0.5%
      if (n_free <= reserve + n_ext) goto try_to_extend;
      break;
    case FSP_CLEANING:
    case FSP_BLOB:
      break;                                                 // ⑮ 不预留
    default:
      ut_error;
  }

  if (fil_space_reserve_free_extents(space_id, n_free, n_ext)) {   // ⑯ 真正的"预留"
    return true;
  }
try_to_extend:
  if (fsp_try_extend_data_file(space, space_header, mtr)) {        // ⑰ 扩物理文件
    buf_page_t *page = &block->page;
    buf_page_make_old(page);
    goto try_again;                                                // ⑱ 扩完重算
  }
  return false;
}
```

⑬⑭ ★ **`reserve` 是"应急预留"**：`FSP_NORMAL` 预留 2 个 extent + 空间大小 ×1%（undo 日志 1 extent + 0.5%，cleaning 操作 1 extent + 0.5%）。**这不是给用户请求用的，是保证系统始终有余量完成 undo 写入与清理**——否则空间耗尽时事务连回滚都写不下。`FSP_UNDO` 只给 cleaning 留 0.5%。

⑮ `FSP_CLEANING` / `FSP_BLOB` **不预留**——它们本身就是清理/临时用途，再预留会造成自锁。

⑯ ★ **`fil_space_reserve_free_extents` 只是"记账"不是分配**：它在 fil 层扣减 `space->free_len` 的可用计数。真正的空间分配发生在后续 `fseg_alloc_free_page`/`fsp_alloc_free_extent` 调用中。这就是"先预留后分配"的本质——**预留买的是"后续分配一定成功"的承诺**。

⑰⑱ 预留失败 → `fsp_try_extend_data_file` 扩物理文件（可能触发 autoextend）→ `buf_page_make_old` → `goto try_again` **回到最开始重算所有量**（因为 size/free_limit 都变了）。

#### 为什么必须先预留：协议的意义

预留不是优化，是**锁安全协议**：一次真正的分配（`fseg_alloc_free_page_low`）会依次锁 FSP header 页 → XDES 页 → inode 页 → 目标数据页，路径上任何一步发现"空间不够"都要回滚整个 mtr——代价高且持有大量锁时回滚易引发死锁。预留阶段**只锁 FSP header 页**，把"空间是否充足"的判定提前，让后续分配路径不必处理失败。

调用方用完必须 `fil_space_release_free_extents(space_id, n_reserved)` 归还——**还的是预留计数，不是空间**（空间在真正分配时才消耗）。

### 物理扩展：fsp_try_extend_data_file 与"是否填 0"

预留失败就去扩文件。这个函数（`fsp0fsp.cc:1269`）是"表空间扩容"的真正落地点，逐段解析：

```cpp
static UNIV_COLD ulint fsp_try_extend_data_file(fil_space_t *space,
                                                fsp_header_t *header, mtr_t *mtr) {
  const char *OUT_OF_SPACE_MSG =
      "ran out of space. Please add another file or use"
      " 'autoextend' for the last file in setting";
  ut_d(fsp_space_modify_check(space->id, mtr));
```

**`UNIV_COLD`**：告诉编译器这是冷路径（扩展文件极少发生），把代码放冷区、不内联。

```cpp
  /* 系统表空间 / 临时表空间：不允许 autoextend 就直接报错，且只报一次 */
  if (space->id == TRX_SYS_SPACE && !srv_sys_space.can_auto_extend_last_file()) {
    if (!srv_sys_space.get_tablespace_full_status()) {          // ① 一次性告警
      ib::error(ER_IB_MSG_415) << "Tablespace " << srv_sys_space.name() << " "
                               << OUT_OF_SPACE_MSG << " innodb_data_file_path.";
      srv_sys_space.set_tablespace_full_status(true);
    }
    return false;
  } else if (fsp_is_global_temporary(space->id) && !srv_tmp_space.can_auto_extend_last_file()) {
    ...（同构，报 ER_IB_MSG_416）
  }
```

① ★ **"只报一次"的设计**：`get/set_tablespace_full_status` 是个一次性开关。源码注释解释为什么不复位——"dealing with this error requires server restart"（要解决只能重启加文件），期间反复报同一条错误只会刷屏错误日志。

```cpp
  size = mach_read_from_4(header + FSP_SIZE);
  ut_ad(size == space->size_in_header);
  const page_size_t page_size(mach_read_from_4(header + FSP_SPACE_FLAGS));

  if (space->id == TRX_SYS_SPACE) {
    size_increase = srv_sys_space.get_increment();                    // ② innodb_autoextend_increment
  } else if (fsp_is_global_temporary(space->id)) {
    size_increase = srv_tmp_space.get_increment();
  } else {
    page_no_t autoextend_size_pages = space->autoextend_size_in_bytes / page_size.physical();
    if (autoextend_size_pages > 0) {                                  // ③ AUTOEXTEND_SIZE 路径
      ut_ad((autoextend_size_pages % fsp_get_extent_size_in_pages(page_size)) == 0);
      if ((size % autoextend_size_pages) > 0)
        size_increase = autoextend_size_pages - (size % autoextend_size_pages);  // 补足到整数倍
      else
        size_increase = autoextend_size_pages;
    } else {                                                           // ④ 传统路径
      page_no_t extent_pages = fsp_get_extent_size_in_pages(page_size);
      if (size < extent_pages) {
        if (!fsp_try_extend_data_file_with_pages(space, extent_pages - 1, header, mtr))
          return false;
        size = extent_pages;
      }
      size_increase = fsp_get_pages_to_extend_ibd(page_size, size);   // 小表空间逐个 extent 扩
    }
    if (space->m_undo_extend != 0) {                                  // ⑤ undo 表空间额外算法
      adjust_undo_extend(space);
      size_increase = std::max(size_increase, space->m_undo_extend);
    }
    DBUG_EXECUTE_IF("fsp_crash_before_space_extend", DBUG_SUICIDE(););  // ⑥ 崩溃注入点
  }
```

`size_increase` 有**五条来源**（②系统表空间的 `innodb_autoextend_increment`、③AUTOEXTEND_SIZE 对齐、④传统的 `fsp_get_pages_to_extend_ibd`、⑤undo 表空间的 `m_undo_extend` 取 max）。⑥是专门用于测试"扩展过程中崩溃"的注入点。

```cpp
  if (size_increase == 0) return false;

  if (!fil_space_extend(space, size + size_increase)) return false;   // ⑦ 真正扩文件

  /* We ignore any fragments of a full megabyte when storing the size to the space header */
  space->size_in_header =
      ut_calc_align_down(space->size, (1024 * 1024) / page_size.physical());   // ⑧ 向下对齐到整 MB
  fsp_header_size_update(header, space->size_in_header, mtr);                  // ⑨ 写回 FSP_SIZE
  return true;
}
```

★★ **⑧ 是最容易被忽略的一行**：文件实际扩到了 `size + size_increase`，但写入 `FSP_SIZE` 的是**向下对齐到整 MB** 的值（`ut_calc_align_down`），注释明说 "We ignore any fragments of a full megabyte"。

后果：**文件大小可以大于 `FSP_SIZE × page_size`**——那些零头页（不足 1MB 的部分）不纳入 `FSP_SIZE`，但在下一次扩展时会被重新计入（因为下次是从 `space->size` 而非 `FSP_SIZE` 继续算）。这是刻意设计：让 `FSP_SIZE` 保持 MB 对齐，简化后续 extent 计算。

#### 扩文件到底填不填 0？

`fil_space_extend`（fil0fil.cc:6800）→ `Fil_shard::space_extend`，内部顺序是：

```cpp
/* 1. 先写 redo —— 在真正扩展之前！ */
fil_op_write_space_extend(space->id, node_start, len, &mtr);
mtr_commit(&mtr);

/* 2. 优先 posix_fallocate（只预留空间、不写数据） */
int ret = posix_fallocate(file->handle.m_file, node_start, len);

/* 3. 失败 或 开关要求 → 显式写 0 */
if ((tbsp_extend_and_initialize && !file->atomic_write) || err == DB_IO_ERROR) {
  err = fil_write_zeros(file, phy_page_size, node_start, len);
}
```

★ **答案：默认填 0，但优先尝试不填的快路径**

- `tbsp_extend_and_initialize`（`srv0srv.cc:523`）**默认 true** —— 所以默认会显式 `fil_write_zeros`；
- 若设为 false，则依赖 `posix_fallocate` 的语义（文件系统保证读新空间返回 0）；
- `posix_fallocate` 失败（如 ext3 + O_DIRECT 不支持、或磁盘满）时**回退到写 0**。

★ **为什么要在扩展之前先记 redo**（`fil0fil.cc:6660-6676` 的注释详述）：`posix_fallocate()` 存在原子性缺陷——可能"空间已预留、但文件元数据未更新"时就崩溃。此时无法从文件大小反推旧大小，也就不知道哪段需要补 0。所以 redo 里记下 `(offset, len)`，恢复时对比物理文件大小，**只补写不足的那段**（`fil0fil.cc:10529-10544` 注释："Write out the 0's in the extended space only if the physical size of the file is less than the expected size"）。

**★★ `Fil_shard::space_extend()` 的完整关键段**（★ 落盘三步 + 内存更新）：

```cpp
bool Fil_shard::space_extend(fil_space_t *space, page_no_t size) {
  for (;;) {
    mutex_acquire();
    space = get_space_by_id(space->id);
    if (size < space->size) { mutex_release(); return true; }   /* 已够大 */
    file = &space->files.back();                                /* ★ 永远只扩「最后一个」node */
    if (!file->is_being_extended) {
      file->is_being_extended = true;                           /* ★ 标志位 + 轮询做互斥 */
      break;
    }
    mutex_release();
    /* Another thread is currently using the file. Wait for it to finish.
    It'd have been better to use an event driven mechanism but the entire
    module is peppered with polling code. */
    std::this_thread::sleep_for(std::chrono::microseconds(20));
  }

  if (!prepare_file_for_io(file)) { file->is_being_extended = false; ... return false; }

  const size_t phy_page_size = page_size.physical();   /* ★ 换算基准是 physical（压缩表用 ZIP 大小） */
  mutex_release();                                     /* ★ 拿着 n_pending_ios 就能放 mutex 做慢 IO */

  os_offset_t node_start = os_file_get_size(file->handle);      /* 以文件当前字节数为起点 */
  page_no_t node_first_page = space->size - file->size;         /* 该 node 在表空间里的起始页号 */
  page_no_t n_node_physical_pages = node_start / phy_page_size;
  page_no_t n_node_extend = size - (node_first_page + file->size);

  if (file->size + n_node_extend > n_node_physical_pages) {
    os_offset_t len = ((file->size + n_node_extend) * phy_page_size) - node_start;  /* ★ 目标字节长度 */

    /* ① 先记 redo（Linux 且非临时/系统表空间） */
    fil_op_write_space_extend(space->id, node_start, len, &mtr);
    mtr_commit(&mtr);

    /* ② 优先 posix_fallocate —— 只预留空间，不写数据 */
    int ret = posix_fallocate(file->handle.m_file, node_start, len);

    /* ③ 只有开关要求 或 fallocate 失败，才真的写 0 */
    if ((tbsp_extend_and_initialize && !file->atomic_write) || err == DB_IO_ERROR) {
      err = fil_write_zeros(file, phy_page_size, node_start, len);
    }

    os_offset_t end = os_file_get_size(file->handle);
    success = (end == node_start + len);     /* ★ 磁盘满时可能只加了一部分 */
    pages_added = (end / phy_page_size) - file->size;
  } else {
    success = true; pages_added = n_node_extend;
  }

  mutex_acquire();
  file->size += pages_added;      /* ★ node 页数增加 */
  space->size += pages_added;     /* ★★ 表空间 size 同步增加 —— 这是 size 的唯一增长点 */
  file->is_being_extended = false;
  complete_io(file, IORequestWrite);
  space_flush(space->id);         /* ★ 扩完立刻 fsync，保证文件系统元数据落盘 */
  mutex_release();
  return success;
}
```

> ★★★ **`space->size` 的唯一增长点就是这里**。所有"表空间变大"的诉求（`fsp_try_extend_data_file`、自动扩展、AUTOEXTEND_SIZE）最终都汇流到 `Fil_shard::space_extend()`。
> ★ `fil_write_zeros()` 单次最多写 1MB（`std::min(1024*1024, len)`）循环推进，且用 `ut::aligned_zalloc(n_bytes, page_size)`——**对齐是 O_DIRECT 的硬性要求**，zalloc 保证内容天然全 0。

**★ `fil_io()` → `Fil_shard::do_io()` 的偏移换算**（★ 链路的最末端）：

```cpp
dberr_t fil_io(const IORequest &type, bool sync, const page_id_t &page_id, ...) {
  auto shard = fil_system->shard_by_id(page_id.space());        /* 跳 1：space_id % 64 */
  return shard->do_io(type, sync, page_id, page_size, byte_offset, len, buf, message);
}

dberr_t Fil_shard::do_io(...) {
  mutex_acquire();
  auto space = get_space_by_id(page_id.space());                /* 跳 2：unordered_map */
  ...
  fil_node_t *file;
  auto page_no = page_id.page_no();
  get_file_for_io(space, &page_no, file);                       /* 跳 3：page_no 就地改写 */
  if (!prepare_file_for_io(file)) { ... }                       /* 跳 4：懒打开 + pending IO */
  mutex_release();

  auto offset = (os_offset_t)page_no * page_size.physical();    /* ★★ 跳 5：核心换算 */
  offset += byte_offset;                                        /* 页内偏移 */
  ...
}
```

> ★ `Fil_shard::prepare_file_for_io()`：文件没开就 `open_file()`，然后 `++file->n_pending_ios`——**这个计数是 mutex 释放后文件不会被 LRU 关掉的保证**（`close_file()` 要求 `can_be_closed()` 即 `n_pending_ios == 0 && n_pending_flushes == 0`）。

---

#### ★★ extent 层的完整机制（补充）

**★ `xdes_state_t` 是 6 个值，不是 4 个**（⚠️ 常见说法只列 4 个）：

```cpp
enum xdes_state_t {
  XDES_NOT_INITED = 0,   /* extent descriptor is not initialized */
  XDES_FREE = 1,         /* extent is in free list of space */
  XDES_FREE_FRAG = 2,    /* extent is in free fragment list of space */
  XDES_FULL_FRAG = 3,    /* extent is in full fragment list of space */
  XDES_FSEG = 4,         /* extent belongs to a segment */
  XDES_FSEG_FRAG = 5     /* ★ fragment extent leased to segment（8.0 新增「租借」） */
};
```

**★★ 状态迁移的权威来源**（`xdes_set_state()` 里的 debug 断言，这是唯一权威版本）：

```
                  xdes_init()
   XDES_NOT_INITED (0) ──────────► XDES_FREE (1)          [挂 FSP_FREE]
                                    │        │
            fsp_alloc_free_page()   │        │  fseg_alloc_free_extent()
                                    ▼        ▼
                          XDES_FREE_FRAG (2) XDES_FSEG (4)  [FSEG_FREE→NOT_FULL→FULL]
                            │      ▲   │
            满了 → FULL_FRAG │      │   │ fsp_alloc_xdes_free_frag()（8.0 租借）
                            ▼      │   ▼
                     XDES_FULL_FRAG (3)  XDES_FSEG_FRAG (5)
                            │                 │
                            └────────┬────────┘ 段用完还回 FSP
```

> ★★★ **`XDES_FSEG_FRAG`(5) 是 8.0 新增的"租借"机制**——把**整个 extent** 从 `FSP_FREE_FRAG` 借给段（段只能用 page 2~63，page 0/1 是 XDES 页和 IBUF bitmap 页），而不是像碎片页那样只借单页。判据在 `xdes_is_leasable()`：① 必须是"每组 extent 的头一个"（含 XDES 页）② page 0/1 必须已占用 ③ page 2~63 必须全空闲。

**★ XDES 页的完整布局**（16K 页）：

```
offset 0     ┌ FIL Header (38B) ────────────────────────────┐
offset 24    │ FIL_PAGE_TYPE = 8(FSP_HDR) / 9(XDES)  ← 区别 │
offset 38    ├ FSP Header (112B，只在 page 0 有意义) ───────┤
             │  +0  FSP_SPACE_ID   +8   FSP_SIZE            │
             │  +12 FSP_FREE_LIMIT +16  FSP_SPACE_FLAGS     │
             │  +20 FSP_FRAG_N_USED                         │
             │  +24 FSP_FREE      +40  FSP_FREE_FRAG        │
             │  +56 FSP_FULL_FRAG +72  FSP_SEG_ID           │
             │  +80 FSP_SEG_INODES_FULL / +96 ..._FREE      │
offset 150   ├ XDES Arr[0..255]，每项 40B，共 10240B ───────┤ ← XDES_ARR_OFFSET
             │  每项：+0 seg_id(8) +8 FLST node(12)          │
             │        +20 state(4) +24 bitmap(16)            │
offset 10390 ├ 加密元数据 Encryption::INFO_SIZE ────────────┤
16384-8      └ FIL Trailer (8B) ────────────────────────────┘
```

```cpp
/* 每个 extent 描述符 40 字节的构成 */
constexpr uint32_t XDES_ID        = 0;                     // 8B  seg_id，0 = 不属于任何段
constexpr uint32_t XDES_FLST_NODE = 8;                     // 12B FLST 节点（prev 6B + next 6B）
constexpr uint32_t XDES_STATE     = FLST_NODE_SIZE + 8;    // = 20，4B
constexpr uint32_t XDES_BITMAP    = FLST_NODE_SIZE + 12;   // = 24，页位图
constexpr uint32_t XDES_BITS_PER_PAGE = 2;                 // ★ 每页 2 bit
constexpr uint32_t XDES_FREE_BIT  = 0;
constexpr uint32_t XDES_CLEAN_BIT = 1;   /* NOTE: currently not used! */
#define XDES_SIZE (XDES_BITMAP + UT_BITS_IN_BYTES(FSP_EXTENT_SIZE * XDES_BITS_PER_PAGE))
/* 16K 页：24 + (64*2)/8 = 40 */
```

> ★★ **`XDES_CLEAN_BIT` 源码注释明写 "currently not used"**——第二个 bit 是历史遗留，实际只有 `XDES_FREE_BIT` 起作用。
> ★ `xdes_init()` 把 bitmap 全写 `0xFFFFFFFF`——**每页 2 bit 都是 1**，即"全页空闲且干净"。

**★★ extent 大小随页大小变化**（⚠️ "1MB extent" 只对 4K/8K/16K 成立）：

```cpp
#define FSP_EXTENT_SIZE static_cast<page_no_t>(          \
      ((UNIV_PAGE_SIZE <= (16384) ? (1048576 / UNIV_PAGE_SIZE)          \
      : ((UNIV_PAGE_SIZE <= (32768)) ? (2097152 / UNIV_PAGE_SIZE)       \
                                     : (4194304 / UNIV_PAGE_SIZE)))))
```

| 页大小 | FSP_EXTENT_SIZE（页） | extent 字节 | XDES_SIZE | 每 XDES 页的 extent 数 |
|---|---|---|---|---|
| 4 KiB | 256 | 1 MiB | 88 | 16 |
| 8 KiB | 128 | 1 MiB | 56 | 64 |
| **16 KiB** | **64** | **1 MiB** | **40** | **256** |
| 32 KiB | 64 | 2 MiB | 40 | 512 |
| 64 KiB | 64 | 4 MiB | 40 | 1024 |

> ⚠️ **`XDES_DESCRIBED_PER_PAGE` 在 8.0.39 已废弃**——`fsp0types.h:150` 只剩一行被注释掉的 `#define`。等价语义由 `page_size.physical()` 承担。
> ⚠️ **不存在 `xdes_clear_bit()` / `xdes_is_bit_set()`**——清位就是 `xdes_set_bit(descr, bit, offset, false, mtr)`。

**★ `fsp_fill_free_list()`：从 FSP_FREE_LIMIT 之上初始化新 extent**（★ 表空间的"批发"动作）：

```cpp
static void fsp_fill_free_list(bool init_space, fil_space_t *space,
                               fsp_header_t *header, mtr_t *mtr) {
  ...
  if (size < limit + FSP_EXTENT_SIZE * FSP_FREE_ADD) {   /* 不足 4 个 extent → 先物理扩容 */
    fsp_try_extend_data_file(space, header, mtr);
    size = space->size_in_header;
  }

  i = limit;   /* ★ 从 FSP_FREE_LIMIT 开始 */
  while ((init_space && i < 1) ||
         ((i + FSP_EXTENT_SIZE <= size) && (count < FSP_FREE_ADD))) {   /* ★ 一次最多 4 个 */

    bool init_xdes = (ut_2pow_remainder(i, page_size.physical()) == 0);
    /* ★★ i % 16384 == 0 ⇒ 这个 extent 含 XDES 描述页 + IBUF bitmap 页，只能进 FREE_FRAG */

    space->free_limit = i + FSP_EXTENT_SIZE;              /* 内存态前移 */
    mlog_write_ulint(header + FSP_FREE_LIMIT, i + FSP_EXTENT_SIZE, MLOG_4BYTES, mtr);

    if (init_xdes) {
      if (i > 0) {                                        /* page 0 已存在，跳过 */
        block = buf_page_create(page_id, page_size, RW_SX_LATCH, mtr);
        fsp_init_file_page(block, mtr);
        mlog_write_ulint(frame + FIL_PAGE_TYPE, FIL_PAGE_TYPE_XDES, MLOG_2BYTES, mtr);
        /* ★ 新描述页的 type = 9；page 0 保持 8 */
      }
      /* IBUF bitmap 页（extent 内第 2 页）在独立的 mtr 里初始化
         —— 因为它在 latching order 里层级很低，必须能提前放 latch */
      if (!fsp_is_system_temporary(space->id)) {
        mtr_start(&ibuf_mtr);
        const page_id_t page_id(space->id, i + FSP_IBUF_BITMAP_OFFSET);
        ...
        ibuf_bitmap_page_init(block, &ibuf_mtr);
        mtr_commit(&ibuf_mtr);
      }
    }

    descr = xdes_get_descriptor_with_space_hdr(header, space->id, i, mtr, init_space, &desc_block);
    xdes_init(descr, mtr);                                /* ★ seg_id=0 / XDES_FREE / bitmap 全 1 */

    if (init_xdes) {
      fsp_init_xdes_free_frag(header, descr, mtr);        /* 含 XDES 页 → FREE_FRAG，page0/1 标已用 */
    } else {
      flst_add_last(header + FSP_FREE, descr + XDES_FLST_NODE, mtr);
      count++;                                            /* ★ 只有真正进 FSP_FREE 的才计数 */
    }
    i += FSP_EXTENT_SIZE;
  }
  space->free_len += (uint32_t)count;                     /* ★ 内存态计数，供 DATA_FREE 用 */
}
```

> ★★★ **`init_xdes` 这个判据是整章最关键的细节**：`i % 16384 == 0` 的 extent 因为要"牺牲"前两页（XDES 页 + IBUF bitmap 页），所以**不能整块交给段**，只能进 `FSP_FREE_FRAG` 走碎片分配。这也解释了 `fsp_get_available_space_in_free_extents()` 里那句 `n_free_up -= n_free_up / (page_size.physical() / FSP_EXTENT_SIZE)`——**每 256 个 extent 要扣掉 1 个**。
> ★ `const ulint XDES_FRAG_N_USED = 2` 就是这两个先天被占的页。

**★ FLST：八大链表的统一底座**（★ 侵入式、可跨页）：

```cpp
/* fut0lst.h:42 */
typedef byte flst_base_node_t;
typedef byte flst_node_t;
constexpr ulint FLST_BASE_NODE_SIZE = 4 + 2 * FIL_ADDR_SIZE;   // = 16：len(4) + first(6) + last(6)
constexpr ulint FLST_NODE_SIZE      = 2 * FIL_ADDR_SIZE;       // = 12：prev(6) + next(6)

/* fil0types.h:123 —— 一个"文件地址"= 4B 页号 + 2B 页内偏移 */
constexpr size_t FIL_ADDR_PAGE = 0;
constexpr size_t FIL_ADDR_BYTE = 4;
constexpr size_t FIL_ADDR_SIZE = 6;
```

> ★★ **链表节点存的是 `fil_addr_t`（页号 + 页内偏移），不是指针**——所以链表天然可跨页、可持久化。`fut_get_ptr()` 才把它翻译成内存指针（必要时额外 fix 一页）。
> ★ 增删节点时若前驱/后继与自己在**同一页**，可直接 `page_align(node) + addr.boffset` 定位；否则要 `fut_get_ptr()` 多 fix 一页。这是 FLST 操作可能涉及 2~3 个 page latch 的原因。
> ⚠️ **不存在 `xdes_lst_add_node()` / `xdes_lst_remove_node()`**——XDES 链表操作直接用通用 `flst_add_last(header + FSP_FREE, descr + XDES_FLST_NODE, mtr)`。

### FSP_SIZE / FSP_FREE_LIMIT / FSP_FREE：三个"边界"的辨析

页 0 的 FSP header 里有三个字段名字相近、语义递进，是理解扩展行为的钥匙：

| 概念 | 字段 | 含义 | 页号 >= 该值意味着 |
|---|---|---|---|
| **物理大小** | `FSP_SIZE` | 表空间文件的实际页面数 | 超出文件范围，**页不存在** |
| **已初始化边界** | `FSP_FREE_LIMIT` | extent 元数据已初始化的边界 | 未初始化，属于"潜在空闲" |
| **Free 链表** | `FSP_FREE` | 已初始化且空闲的 extent 链表 | — |

```
 Page 0                                    FSP_FREE_LIMIT          FSP_SIZE
   │                                              │                    │
   ▼                                              ▼                    ▼
 ┌────────┬───────────────────────────────────────┬────────────────────┐
 │FSP hdr │   已初始化区域                         │   未初始化区域       │
 │        │   （在 Free/Frag/Segment 链表中）       │   （尚未纳入管理）    │
 └────────┴───────────────────────────────────────┴────────────────────┘
          │◄──── 由 extent 链表管理 ──────────────►│◄── 按需懒初始化 ────►
```

**"未初始化"的准确含义**：这些页在文件里**已经存在**（文件大小已覆盖它们），但没有对应的 XDES entry、也不在任何链表上——要先用 `fsp_fill_free_list` 初始化出 XDES 元数据、挂进 `FSP_FREE`，才能被分配。这正是 `fsp_fill_free_list` 每次推进 4 个 extent（`FSP_FREE_ADD`）而不是一次全初始化的原因：把"扩展文件"和"初始化元数据"的代价摊薄到多次分配里。

三者的推进关系：`FSP_FREE` 空 → 从 `FSP_FREE_LIMIT` 之上初始化新 extent（推进 `FSP_FREE_LIMIT`）→ 若 `FSP_FREE_LIMIT + 4×extent > FSP_SIZE`，先扩文件（推进 `FSP_SIZE`）。

### AUTOEXTEND_SIZE：8.0 的表空间增量扩展单位

```sql
CREATE TABLESPACE ts1 AUTOEXTEND_SIZE = 4M;
ALTER  TABLESPACE ts1 AUTOEXTEND_SIZE = 8M;
```

8.0 新增的表空间属性：**每次扩展文件时按这个粒度对齐增长**，避免"每次只扩一点点、反复扩展文件"带来的碎片与系统调用开销。

三个环节的流转：

1. **SQL → 内存**：`HA_CREATE_INFO::m_implicit_tablespace_autoextend_size` 承接 SQL 选项；
2. **内存 → DD**：`dd::Tablespace::options()` 以 `autoextend_size` 键持久化（`dict0dd.cc:4114`： `toptions.set(autoextend_size_str, space->autoextend_size_in_bytes)`）；
3. **DD → `fil_space_t`**：开表时从 `dd::Properties` 读回，调 `fil_set_autoextend_size()`（`fil0fil.cc:8930`）写入 `space->autoextend_size_in_bytes`。

扩展时的对齐逻辑（`fsp0fsp.cc:1322`）：

```cpp
page_no_t autoextend_size_pages = space->autoextend_size_in_bytes / page_size.physical();
if (autoextend_size_pages > 0) {
  ut_ad((autoextend_size_pages % fsp_get_extent_size_in_pages(page_size)) == 0);
  /* 当前 size 不是 autoextend_size 的整数倍时，只补足到最近的倍数 */
  if ((size % autoextend_size_pages) > 0)
    size_increase = autoextend_size_pages - (size % autoextend_size_pages);
  else
    size_increase = autoextend_size_pages;
} else { /* 未设置：按 extent 粒度扩展 */ }
```

两个约束值得注意：

- ★ **必须是 extent 页数的整数倍**（`ut_ad` 断言）——否则扩展出的页数无法被 XDES 整除管理；
- **建表空间时就用它**：`dict0crea.cc:142` 取 `tablespace->get_autoextend_size() / srv_page_size` 作为初始页数，未设置才回退 `FIL_IBD_FILE_INITIAL_SIZE`。

另：`fil_set_autoextend_size` 首行是 `ut_ad(space_id != TRX_SYS_SPACE)`——**系统表空间不支持该属性**。

### 表空间扩展与 fsp_fill_free_list

FREE 链表耗尽时，从 `FSP_FREE_LIMIT` 之上批量拉 extent（完整实现见前文「理论基础」的惰性初始化）：

```cpp
static void fsp_fill_free_list(bool init_space, fil_space_t *space,
                               fsp_header_t *header, mtr_t *mtr) {
  size = mach_read_from_4(header + FSP_SIZE);
  limit = mach_read_from_4(header + FSP_FREE_LIMIT);

  /* 若不够 4 个 extent 且可 autoextend，先扩文件 */
  if (size < limit + FSP_EXTENT_SIZE * FSP_FREE_ADD) {
    if ((!init_space && !fsp_is_system_tablespace(space->id) && ...) || ...) {
      fsp_try_extend_data_file(space, header, mtr);
      size = space->size_in_header;
    }
  }

  i = limit;
  while ((init_space && i < 1) ||
         ((i + FSP_EXTENT_SIZE <= size) && (count < FSP_FREE_ADD))) {
    /* 每 256 个 extent 为一组，组首是 XDES 页 */
    bool init_xdes = (ut_2pow_remainder(i, page_size.physical()) == 0);

    space->free_limit = i + FSP_EXTENT_SIZE;
    mlog_write_ulint(header + FSP_FREE_LIMIT, i + FSP_EXTENT_SIZE, MLOG_4BYTES, mtr);

    if (init_xdes) {
      /* 初始化新的 XDES 页 + ibuf bitmap 页（独立 mtr，latch 顺序低） */
      ...ibuf_bitmap_page_init(block, &ibuf_mtr);
    }

    descr = xdes_get_descriptor_with_space_hdr(header, space->id, i, mtr, ...);
    xdes_init(descr, mtr);

    if (init_xdes) {
      /* 组首 extent 因 XDES 页占用页 0，只能当碎片 extent */
      fsp_init_xdes_free_frag(header, descr, mtr);
    } else {
      /* 普通 extent 加入 FSP_FREE 链表 */
      flst_add_last(header + FSP_FREE, descr + XDES_FLST_NODE, mtr);
      count++;
    }
    i += FSP_EXTENT_SIZE;
  }
  space->free_len += (uint32_t)count;
}
```

要点：

- **组首 extent 与其余 extent 的差别对待**：每 256 个 extent 的第一页是 XDES 页，它自己占了 extent 的页 0，所以这个 extent 只能当"碎片 extent"进 `FSP_FREE_FRAG`（页 1 留给 ibuf bitmap，页 2~63 供碎片页租借）——这正是 `XDES_FRAG_N_USED=2` 的含义，也是 lease 的候选来源。
- **ibuf bitmap 页单独 mtr 初始化**：它在 latching 顺序里位置低，必须能单独释放锁——一个 mtr 里不能既持 FSP 页锁又持 bitmap 页锁跨过这个顺序。
- `mlog_write_ulint` 写 free_limit 的 redo：free_limit 的推进必须崩溃可恢复，否则恢复后"已初始化 extent"边界丢失。

### FSP_FLAGS 位域：一个 32 位承载表空间所有属性

- `POST_ANTELOPE`（bit0）：1 表示 COMPRESSED/DYNAMIC，REDUNDANT/COMPACT 为 0；
- `ZIP_SSIZE`（4bit）：压缩页大小（0 = 未压缩）；
- `ATOMIC_BLOBS`（bit5）：是否用原子 BLOB；
- `PAGE_SSIZE`（4bit）：页大小的对数编码（16K → 5）；
- `DATA_DIR`/`SHARED`/`TEMPORARY`/`ENCRYPTION`/`SDI`：`CREATE TABLESPACE` 的对应属性。

经典坑：**`FSP_FLAGS` 无法区分 REDUNDANT 与 COMPACT**（二者 POST_ANTELOPE 都是 0），`fsp_flags_to_dict_tf` 需要额外的 `compact` 布尔参数才能还原 table flags。

### 系统表空间与 undo 表空间的页布局

### 各表空间的固定页布局

**先看一张独立表空间的整体嵌套布局**（Group → Extent → Page 三层，16KB 页）：

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 0: Extent 0-255 (pages 0-16383)                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Extent 0 (pages 0-63):                                    │  │
│  │   Page 0: FSP_HDR + XDES array（描述本组 extent 0-255）     │  │
│  │   Page 1: IBUF_BITMAP                                     │  │
│  │   Page 2: INODE                                           │  │
│  │   Page 3-63: 普通数据页                                    │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Extent 1 (pages 64-127):    64 个普通数据页                 │  │
│  │ Extent 2 (pages 128-191):   64 个普通数据页                 │  │
│  │   ...                                                     │  │
│  │ Extent 255 (pages 16320-16383): 64 个普通数据页             │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│ Group 1: Extent 256-511 (pages 16384-32767)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Extent 256 (pages 16384-16447):                           │  │
│  │   Page 16384: XDES（描述 extent 256-511，类型同 FSP_HDR）   │  │
│  │   Page 16385: IBUF_BITMAP                                 │  │
│  │   Page 16386-16447: 普通数据页                             │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Extent 257+ : 64 个普通数据页                              │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│ Group 2, 3, ... : 每 16384 页（= 256 extent）重复一组             │
└─────────────────────────────────────────────────────────────────┘
```

三条规则（与 `fsp_fill_free_list` 的 `init_xdes` 判断一致）：

1. **extent 每 256 个（= 16384 页）为一组**；
2. **每组的第一个 extent 的 page 0 是 FSP_HDR/XDES 页**，内含 XDES array 描述本组 256 个 extent；
3. **每组前两页固定**：FSP_HDR/XDES + IBUF_BITMAP。首组（Group 0）额外多一个固定的 INODE 页（页 2）。

组首 extent 因页 0/1 被元数据占用，只能当碎片 extent（进 `FSP_FREE_FRAG`）——这是 lease 机制的由来（段可租借其页 2~63）。

**独立表空间**（file-per-table `.ibd`）只有前 3 页固定：页 0 = FSP_HDR、页 1 = IBUF_BITMAP、页 2 = 第一个 INODE 页。**页 3 通常是该表聚簇索引的根页**——`dict_build_tablespace` 的注释写得很直白：

```cpp
/* We create a new generic empty tablespace.
We initially let it be 4 pages:
- page 0 is the fsp header and an extent descriptor page,
- page 1 is an ibuf bitmap page,
- page 2 is the first inode page,
- page 3 will contain the root of the clustered index of the
first table we create here. */
```

`btr_create` 里连续调两次 `fseg_create`（**先非叶子段、再叶子段**），把两个 segment header 写进 root page——调试时能看到此时 `page_no = 3`。这也解释了为什么 `fseg_create_general` 的 `page` 参数传 3：段头要写在根页的 `PAGE_BTR_SEG_TOP`/`PAGE_BTR_SEG_LEAF` 两个槽位。

⚠️ 但**页 3 不保证是索引页**：注释用的是 "will contain"（将要存放），它只是"新建表空间的第一个表"的分配结果。若这个表空间里已存在别的表（通用表空间）、或经历过 DDL 重建，root 就会落在别处。**定位 root 页的唯一可靠途径是数据字典**（`dict_index_t::page`，源头是 DD 的 `mysql.indexes` / SYS_INDEXES 里记的 root page no），不能把"页 3"当常量硬编码。

**系统表空间（space 0）** 的固定页号（`fsp0types.h`）：页 0 = XDES/FSP_HDR、页 1 = ibuf bitmap、页 2 = 第一个 inode 页、页 3 = ibuf header、页 4 = ibuf 树根、页 5 = TRX_SYS、页 6 = 第一个回滚段、页 7 = 字典头。

**独立 undo 表空间**的页 3 是 `FSP_RSEG_ARRAY_PAGE_NO`（`fsp0types.h:185`），页类型 `FIL_PAGE_TYPE_RSEG_ARRAY = 21`。它取代了"系统表空间页 5 的 TRX_SYS 里内嵌 rseg 数组"的旧方案——8.0 undo 表空间可独立创建/截断，rseg 目录必须跟着表空间走。

完整布局（`trx0rseg.h:233` 起的偏移常量）：

```
 +0   ┌──────────────────────────────┐
      │ FIL header (38B)             │
+38   ├──────────────────────────────┤
      │ fseg header (10B)            │  本页所属段的段头
+48   ├──────────────────────────────┤ ← RSEG_ARRAY_HEADER = FSEG_PAGE_DATA
      │ RSEG_ARRAY_VERSION   (4B)    │  = 0x52534547 + 1（"RSEG"+1）
+52   ├──────────────────────────────┤
      │ RSEG_ARRAY_SIZE      (4B)    │  当前已跟踪的 rseg 数
+56   ├──────────────────────────────┤ ← +RSEG_ARRAY_FSEG_HEADER_OFFSET(8)
      │ fseg header (10B)            │  跟踪 rseg array 页自身的段
+66   ├──────────────────────────────┤ ← +RSEG_ARRAY_PAGES_OFFSET(18)
      │ slot[0]  (4B)  rseg 0 页号   │
      │ slot[1]  (4B)  rseg 1 页号   │
      │ ...                          │  最多 FSP_MAX_ROLLBACK_SEGMENTS(128)
      │ slot[127](4B)                │
      ├──────────────────────────────┤
      │ 保留区 (200B)                │  RSEG_ARRAY_RESERVED_BYTES，留作将来扩展
      ├──────────────────────────────┤
      │ FIL trailer (8B)             │
      └──────────────────────────────┘
```

★ **每个 slot 只有 4B，只存 page_no**（不是 space_id + page_no 的 8B）：

```cpp
constexpr uint32_t RSEG_ARRAY_SLOT_SIZE = 4;    // trx0rseg.h:263

inline page_no_t trx_rsegsf_get_page_no(trx_rsegsf_t *rsegs_header, ulint slot, mtr_t *mtr) {
  return (mtr_read_ulint(
      rsegs_header + RSEG_ARRAY_PAGES_OFFSET + slot * RSEG_ARRAY_SLOT_SIZE,   // 4B
      MLOG_4BYTES, mtr));
}
```

**不需要 space_id 的理由**：rseg array 页本身位于某个 undo 表空间内，它记录的 rseg 必然同属该表空间——跟 XDES/inode 链表中"不记 space_id"是同一个道理（同一表空间内无需自报家门）。初始化时 `memset(ptr, 0xff, len)` 把全部 slot 置 `FIL_NULL`（`0xFFFFFFFF`），`trx_rseg_array_create` 再逐个填充。

另一个易混点：**页里有两个 fseg header**（偏移 38 和 56）。前者是"本页属于哪个段"，后者是"跟踪 rseg array 页的段"——`trx0rseg.cc:1144` 通过 `RSEG_ARRAY_HEADER + RSEG_ARRAY_FSEG_HEADER_OFFSET` 访问后者。

### 碎片度量：fsp_get_available_space_in_free_extents

```cpp
uintmax_t fsp_get_available_space_in_free_extents(const fil_space_t *space) {
  ulint size_in_header = space->size_in_header;              // 表空间头记录的 size（页数）
  if (size_in_header < FSP_EXTENT_SIZE) return 0;            // 不足一个 extent 的小表：报 0

  // free_limit 以上的页还没初始化，按 64 页/extent 切
  ulint n_free_up = (size_in_header - space->free_limit) / FSP_EXTENT_SIZE;
  if (n_free_up > 0) {
    n_free_up--;                                             // 第一个 extent 是 XDES 描述页
    n_free_up -= n_free_up / (page_size.physical() / FSP_EXTENT_SIZE); // 每 256MB 一组再让出 XDES
  }

  // 预留：2 extent + 总 extent 数的 1%，给 undo 段和清理操作
  ulint reserve = 2 + ((size_in_header / FSP_EXTENT_SIZE) * 2) / 200;
  ulint n_free = space->free_len + n_free_up;                // free extent 链表 + 未初始化部分
  if (reserve > n_free) return 0;

  return (n_free - reserve) * FSP_EXTENT_SIZE * (page_size.physical() / 1024); // 单位 KiB
}
```

逐段解释：

1. **`size_in_header < FSP_EXTENT_SIZE` 返回 0**：表太小、连一个 extent 都不满时没有"整块空闲 extent"可言。这也是新表 `DATA_FREE` 常常是 0 的原因。
2. **`n_free_up`**：free_limit 之上的页还没被任何段用掉，理论上都能切成完整 extent。减 1 是因为第一个 extent 的首页要做 XDES 页；再按 `page_size / FSP_EXTENT_SIZE`（16KB 页 = 256）减一组，因为每 256 个 extent 构成一个"区组"，组首 extent 的首页也是 XDES 页——**与前文 `fsp_fill_free_list` 的 `init_xdes` 是同一个规则的两处体现**。
3. **`reserve`**：保守预留 `2 extent + 1%`，给 undo 段和清理操作兜底，防止上层拿到的"空闲量"误导写入。宁可少报。
4. **结果单位 KiB**：`FSP_EXTENT_SIZE`（64 页）× 每页 KiB 数，调用方再 `* 1024` 转字节。

### FREE_EXTENTS：更"纯"的视角

```cpp
// innobase_get_tablespace_statistics (ha_innodb.cc)
stats->m_free_extents = space->free_len;                                  // 直接就是链表长度
stats->m_total_extents = space->size_in_header / extent_pages;
stats->m_data_free = fsp_get_available_space_in_free_extents(space) * 1024;
```

`FREE_EXTENTS` 不加预留、不掺未初始化区，就是 `FSP_FREE` 链表长度——**它就是前文分配算法里那个链表的实时计数**。物理结构（分配/归还）是"因"，碎片度量是"果"，两者在本篇合一后可以互相印证。

### OPTIMIZE TABLE 的真实实现

`ha_innobase::optimize` 不做任何"优化"，只返回 `HA_ADMIN_TRY_ALTER`（`innodb_optimize_fulltext_only` 时只整理 FTS）。server 层拿到后改走 `mysql_alter_table()`，本质是 `ALTER TABLE t ENGINE=InnoDB`：8.0 默认 INPLACE + LOCK=NONE，DML 暂存 row log（`innodb_online_alter_log_max_size` 默认 128MB）。所以 `OPTIMIZE TABLE` 中间能写入、结尾毫秒级 MDL 阻塞。

---

## 相关的系统变量/状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_file_per_table` | ON | Global | 决定碎片是否隔离到每个 `.ibd`；ON 时重建表才能收缩单表文件，OFF 时进共享表空间 `ibdata1` |
| `innodb_page_size` | 16384 | Global | 决定一个 extent 的页数（= 1MB / page_size，16K 页时 64 页），间接影响 `DATA_FREE` 粒度与 XDES 分组阈值 |
| `innodb_online_alter_log_max_size` | 134217728 | Global | OPTIMIZE TABLE 期间 row log 容量上限，超限则 online alter 失败 |

> 段保留因子 `fseg_reserve_pct`（默认 12.50%）是**内部变量**（fsp0fsp.cc 的 `fseg_reserve_pct`），不对外暴露。

---

## Misc

### 容易误解的概念

| 概念 | 澄清 |
|---|---|
| `FSP_SIZE` vs `FSP_FREE_LIMIT` | 前者是文件实际页数，后者是"已初始化 extent 的边界"——free_limit 之上的页物理存在但未挂任何链表 |
| 段内 FREE/NOT_FULL/FULL vs 表空间 FREE/FREE_FRAG/FULL_FRAG | 前者分类"段拥有的 extent"（按占用度），后者分类"未归属段的 extent"。两套链表、两种 XDES 状态 |
| 碎片页 vs lease extent | 碎片页 = 从 `FSP_FREE_FRAG` 逐页借（inode 的 32 槽）；lease = 8.0 把整个碎片 extent 借给段（`XDES_FSEG_FRAG`），段用页 2~63 |
| `fil_space_t::size` vs `size_in_header` | 前者物理页数（各 node.size 之和），后者 FSP_SIZE 镜像且**向下对齐整 MB**；**两者的差就是"文件比 header 大"**，`DATA_FREE` 正依赖它 |
| 段释放页的方向 | `fseg_free_page_low` 注释明说：extent 空了也**还给表空间**（FSP_FREE/FREE_FRAG），**不回段内 FREE 链**——段 FREE 链只有"进"（`fseg_fill_free_list` 预存）没有"回" |
| `XDES_CLEAN_BIT` | 已废弃（"currently not used"），bitmap 每页 2bit 现在只用 `XDES_FREE_BIT` |
| `FSP_FLAGS` 不区分 REDUNDANT/COMPACT | POST_ANTELOPE 位两者都是 0，需 DD 另传 compact 布尔 |
| `ha_statistics::delete_length` | 引擎内部名字，就是外部 `DATA_FREE`（"Free bytes"），不是"删除的行数" |
| extent 大小随 page size | 16K 页 = 64 页 = 1MB；4K 页 = 256 页 = 1MB；32K 页 = 64 页 = 2MB；64K 页 = 64 页 = 4MB |
| "一个 INODE 页 85 个 inode" | 只对 16K 页成立，`FSP_SEG_INODES_PER_PAGE()` 是函数：4K→21、8K→42、16K→85、32K→170、64K→341 |

### DATA_FREE / FREE_EXTENTS / 页内碎片 三者辨析

| 对象 | 单位 | 口径 | 是否估算 |
|------|------|------|----------|
| `TABLES.DATA_FREE` | 字节 | free extent + 未初始化区，扣 2 extent + 1% 预留 | 是（估算可写空间） |
| `FILES.FREE_EXTENTS` | extent 个数 | 仅 free extent 链表长度 | 否（精确链表长度） |
| 页内空隙（fill factor） | — | B-tree 页 15/16 填充预留 | 不对外暴露 |

---

## ★★ B+ 树索引与两个段：链路的最终收口

前面所有层次（file / tablespace / extent / fseg）最终服务于一个目标——**支撑一棵 B+ 树**。这是整条链路的"用户"。

**★★ `btr_create()`：建一个索引 = 建两个段 + 建根页**（`btr0btr.cc:832`）：

```cpp
ulint btr_create(ulint type, space_id_t space, space_index_t index_id,
                 dict_index_t *index, mtr_t *mtr) {
  if (type & DICT_IBUF) {
    /* ibuf 树：segment header 不在根页，而在独立的 IBUF header 页（page 3） */
    buf_block_t *ibuf_hdr_block =
        fseg_create(space, 0, IBUF_HEADER + IBUF_TREE_SEG_HEADER, mtr);
    ut_ad(ibuf_hdr_block->page.id.page_no() == IBUF_HEADER_PAGE_NO);
    /* Allocate then the next page to the segment: it will be the tree root page */
    block = fseg_alloc_free_page(frame + IBUF_HEADER + IBUF_TREE_SEG_HEADER,
                                 IBUF_TREE_ROOT_PAGE_NO, FSP_UP, mtr);
  } else {
    /* ① 建「非叶段」：page=0 ⇒ 段自己先申请一页，这一页就是 B+ 树根页
          segment header 写在根页的 PAGE_HEADER(38) + PAGE_BTR_SEG_TOP(46) = 偏移 84 */
    block = fseg_create(space, 0, PAGE_HEADER + PAGE_BTR_SEG_TOP, mtr);
  }
  if (block == nullptr) return (FIL_NULL);

  page_no = block->page.id.page_no();      /* ★ 根页页号（非 ibuf 表通常就是 4） */
  frame = buf_block_get_frame(block);

  if (type & DICT_IBUF) {
    flst_init(frame + PAGE_HEADER + PAGE_BTR_IBUF_FREE_LIST, mtr);
    /* ★ ibuf 根页用 PAGE_BTR_SEG_LEAF 的位置(74)放 free list base node */
  } else {
    /* ② 建「叶子段」：page = 根页页号（!=0）⇒ header 写回根页的
          PAGE_HEADER(38) + PAGE_BTR_SEG_LEAF(36) = 偏移 74 */
    if (!fseg_create(space, page_no, PAGE_HEADER + PAGE_BTR_SEG_LEAF, mtr)) {
      btr_free_root(block, mtr);           /* ★ 叶子段建失败 ⇒ 回滚已建的非叶段 */
      if (!index->table->is_temporary()) btr_free_root_invalidate(block, mtr);
      return (FIL_NULL);
    }
  }
  ...
  page = page_create(block, mtr, dict_table_is_comp(index->table), page_create_type);
  btr_page_set_level(page, nullptr, 0, mtr);   /* ★ 新树只有一层：level = 0（根即叶） */
  btr_page_set_index_id(page, page_zip, index_id, mtr);
  btr_page_set_next(page, page_zip, FIL_NULL, mtr);
  btr_page_set_prev(page, page_zip, FIL_NULL, mtr);
  ...
  return (page_no);                            /* ★ 返回根页页号 ⇒ 写入 dict_index_t::page */
}
```

**★★ 根页里两个 segment header 的确切位置**（`page0types.h:90/98`，⚠️ 不在 `page0page.h`）：

```cpp
constexpr uint32_t PAGE_HEADER        = FSEG_PAGE_DATA;   // = 38，索引页头起点
constexpr uint32_t PAGE_LEVEL         = 26;               // 绝对偏移 64
constexpr uint32_t PAGE_INDEX_ID      = 28;               // 绝对偏移 66，8B
constexpr uint32_t PAGE_BTR_SEG_LEAF  = 36;               // ★ 绝对偏移 74，10B —— 叶子段
constexpr uint32_t PAGE_BTR_IBUF_FREE_LIST     = PAGE_BTR_SEG_LEAF;   // ibuf 复用此位
constexpr uint32_t PAGE_BTR_IBUF_FREE_LIST_NODE = PAGE_BTR_SEG_LEAF;
constexpr uint32_t PAGE_BTR_SEG_TOP   = 36 + FSEG_HEADER_SIZE;        // ★ 绝对偏移 84，10B —— 非叶段
```

```
B+ 树根页（16KB，偏移自页首）
  0    ┌ FIL Header (38B) ─────────────────────────────┐
  38   ├ PAGE_HEADER ──────────────────────────────────┤
  64   │   PAGE_LEVEL (2B)                             │
  66   │   PAGE_INDEX_ID (8B)                          │
  74   │   ★ PAGE_BTR_SEG_LEAF (10B) → 叶子段 inode ───┼──► FSEG_FRAG_ARR / FSEG_FREE /
  84   │   ★ PAGE_BTR_SEG_TOP  (10B) → 非叶段 inode ───┼──► FSEG_NOT_FULL / FSEG_FULL
  94   ├ 用户记录（infimum / supremum / 数据）──────────┤
  ...  └ ...                                            ┘
```

> ★★★ **根页同时是 TOP 段的第一个碎片页**——因为 `fseg_create_general()` 中 `page==0` 分支让段先给自己分配一页并把 header 写在自己页上。所以"根页"既是一棵树的入口，也是非叶段物理上的第一块地。
> ★ **为什么是 `SEG_TOP` 先建、`SEG_LEAF` 后建**：`SEG_TOP` 传 `page=0` 才能产出根页；拿到根页页号后才能传非 0 给 `SEG_LEAF`，把叶子段的 header 也写回同一页。顺序不能颠倒。
> ⚠️ **ibuf 树是唯一例外**——它的两个"段位"被复用了：根页的 `PAGE_BTR_SEG_LEAF` 位置放的是 **ibuf free list 的 base node**（16B），而每个空闲页的同一位置放的是 **free list node**（12B）。

**★ 释放一棵 B+ 树**（`btr_free_but_not_root()`，`btr0btr.cc:954`）：

```cpp
  /* 叶子段：完整释放（含根页本身，因为根页也是它的碎片页之一） */
  do {
    mtr_start(&mtr);
    finished = fseg_free_step(root + PAGE_HEADER + PAGE_BTR_SEG_LEAF, true, &mtr);
    mtr_commit(&mtr);
  } while (!finished);   /* ★ 大段分多轮 mtr 释放，避免单个 mtr 过大 */

  /* 非叶段：用 not_header 版本 —— 保住根页（根页是 TOP 段的 header 页） */
  do {
    mtr_start(&mtr);
    finished = fseg_free_step_not_header(root + PAGE_HEADER + PAGE_BTR_SEG_TOP, true, &mtr);
    mtr_commit(&mtr);
  } while (!finished);
```

> ★ `fseg_free_step()` 每次调用只释放**一整个 extent**，返回 false 表示还没完——这是"大操作拆成多个 mtr"的典型手法（避免单个 mtr 的 redo 过大、避免 fix 住太多页）。
> ★ `fseg_free_step()` 用 `fseg_inode_try_get()` 而非 `_get()`——**允许 inode 已被回收（幂等）**，命中时打 "Double free of inode" 信息并直接返回 true。

**★ 八个链表的一句话定位**（把前面所有链表对齐到层次上）：

| 层级 | 链表 | 挂在 | 元素 | 语义 |
|---|---|---|---|---|
| 表空间 | `FSP_FREE` | FSP header +24 | XDES | 完全空闲的 extent |
| 表空间 | `FSP_FREE_FRAG` | +40 | XDES | 部分使用的碎片 extent（逐页借出） |
| 表空间 | `FSP_FULL_FRAG` | +56 | XDES | 已用满的碎片 extent |
| 表空间 | `FSP_SEG_INODES_FREE` | +96 | INODE 页 | 有空闲槽位的 inode 页 |
| 表空间 | `FSP_SEG_INODES_FULL` | +80 | INODE 页 | 槽位用满的 inode 页 |
| **段** | `FSEG_FREE` | inode +12 | XDES | 段预存的空闲 extent |
| **段** | `FSEG_NOT_FULL` | inode +28 | XDES | 段内部分使用的 extent |
| **段** | `FSEG_FULL` | inode +44 | XDES | 段内已用满的 extent |

> ★ 加上 B+ 树同层页的双向链表（`PAGE_BTR_SEG_*` 之外的 `FIL_PAGE_PREV/NEXT`，**与 FLST 完全两套机制**），一共"八大 + 一"。

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `fil_space_create()` 是创建主体 | 只是**薄封装**（fil0fil.cc:3329）→ 真身 `Fil_shard::space_create()`（3247）。且必须先 `mutex_acquire_all()` 拿**所有** shard 的锁 |
| `fil_node_create()` 直接构造 | 只是封装（2448）→ 真身 `Fil_shard::create_node()`（2357），且**返回 `char*`（文件名）不是 `fil_node_t*`** |
| `fil_node_open_file()` / `fil_node_close_file()` | **均不存在** → `Fil_shard::open_file()`（2704）/ `close_file()`（3010） |
| `fil_node_prepare_for_io()` | **不存在** → `Fil_shard::prepare_file_for_io()`（7429） |
| `fil_space_get_next_id()` | **不存在** → `Fil_system::assign_new_space_id()`（3367）+ 外层 `fil_assign_new_space_id()` |
| `fsp_get_size()` / `fsp_get_free_limit()` / `fsp_get_flags()` | **三个都不存在** → `fsp_header_get_field(page, FSP_SIZE/...)` |
| `fsp_get_extent_size_from_flags()` / `fsp_get_pages_in_extent()` | **不存在** → `fsp_get_extent_size_in_pages()` / `fsp_get_pages_to_extend_ibd()` |
| `fsp_space_create()` | **不存在**（0 匹配）→ `fsp_header_init()` |
| `fsp_descr_size()` / `fsp_last_page_no_and_size()` | **均不存在** → 只有 `fsp_descr_page()` |
| `fil_space_t` 里有 `full_crc32` | **不存在**（0 匹配）。8.0 的 full_crc32 是**全局**的；空间级只有 `compression_type` 与 flags 的 ENCRYPTION 位 |
| `fil_space_t::files` 是链表 | 是 **`std::vector<fil_node_t>`**；`UT_LIST_NODE_T` 只用于 LRU 与 unflushed_spaces |
| "一个表空间可以有多个文件" | 断言只允许**系统/临时**表空间：`ut_a(space->id == TRX_SYS_SPACE \|\| purpose == FIL_TYPE_TEMPORARY \|\| files.size() == 1)` |
| `fil_space_free()` 立刻释放内存 | 两段式：`space_free()`（3185）只 detach + 摘 map；真正 free 在 `space_free_low()`（3148）且要求 `has_no_references()` |
| 扩展文件一定写 0 | **不一定**。先 `posix_fallocate()`；只有 `tbsp_extend_and_initialize && !atomic_write` 或 fallocate 失败才 `fil_write_zeros()` |
| `XDES_DESCRIBED_PER_PAGE` 是常量 | **已废弃**（只剩注释掉的 `#define`）；等价语义由 `page_size.physical()` 承担 |
| "一个 XDES 页描述 16384 个 extent" | **错**：描述 **256 个**（16K 页）。16384 是"一组 extent 的页数" |
| XDES 状态只有 4 种 | **6 种**：多 `XDES_NOT_INITED=0` 与 8.0 新增的 `XDES_FSEG_FRAG=5`（租借） |
| 有 `XDES_FSEG_FRAG_LIMIT` | **不存在** → `XDES_FRAG_N_USED=2` 与 `FSEG_FRAG_LIMIT=32` 是两个不同层级的东西 |
| `xdes_clear_bit()` / `xdes_is_bit_set()` | **均不存在** → `xdes_set_bit(descr, bit, off, false, mtr)` 即清位 |
| `xdes_lst_add_node()` / `xdes_lst_remove_node()` | **均不存在** → 直接用通用 `flst_add_last(header + FSP_FREE, descr + XDES_FLST_NODE, mtr)` |
| `fsp_alloc_free_frag_page()` | **不存在** → `fsp_alloc_free_page()` |
| XDES 页的 type 就是 `FIL_PAGE_TYPE_XDES` | **page 0 是 `FIL_PAGE_TYPE_FSP_HDR`(8)**，只有 page 0 之外的描述页才是 9 |
| extent 恒 1MB / 64 页 | 4K/8K/16K 页是 1MiB；**32K 页 = 2MiB，64K 页 = 4MiB**；压缩表按 `physical()` 换算 |
| `FSEG_HDR_SPACE_ID` | **不存在** → `FSEG_HDR_SPACE`（fsp0types.h:92） |
| `fseg_create_root_page()` | **不存在** → `fseg_create(space, page, byte_offset, mtr)` |
| `fsp_free_seg_inode_page()` | **不存在** → `fsp_free_seg_inode()`（释放 inode；整页归还内嵌在其尾部） |
| `fseg_reserve_free_extents()` | **不存在** → `fsp_reserve_free_extents()`；另有无关的全局变量 `fseg_reserve_pct`（默认 12.50） |
| `fseg_verify()` / `fsp_validate()` | **均不存在** → 只有 `File_segment_inode::verify_not_full_n_used()` 与 `btr_root_fseg_validate()`（均 UNIV_DEBUG） |
| `fseg_header_get_space_id()` 等三个 C 函数 | **均已不存在** → 裸 `mach_read_from_4/2`；`UNIV_DEBUG` 下有打印用封装类 `class fseg_header` |
| "一个 INODE 页固定 85 个 inode" | 只对 16K 页成立。`FSP_SEG_INODES_PER_PAGE()` 是**函数**：4K→21、8K→42、16K→85、32K→170、64K→341 |
| "inode 数组从偏移 38 开始" | **错**：`FSEG_ARR_OFFSET = 38 + FLST_NODE_SIZE(12) = **50**`；前 12B 是 INODE 页自身的链表节点 |
| 释放页时 extent 空了会回到 `FSEG_FREE` | **错**：`fseg_free_page_low()` 在 `n_used==0` 时调 `fsp_free_extent()` **直接还给表空间** |
| `PAGE_BTR_SEG_LEAF/TOP` 在 `page0page.h` | **错**：在 **`include/page0types.h:90/98`** |
| `space->size == FSP_SIZE` | 两个不同的数：`size` 物理页数，`size_in_header` 是 FSP_SIZE 镜像且**向下对齐整 MB** |

**★ 四个"结构性事实"**

1. **`offsets[]` 之上、物理之下，InnoDB 用"算术定位"和"链表定位"两种方式管理元数据，选择依据是"有没有算术规律"**。XDES 描述"第 n 个 extent"，n 与页号有固定关系（每 16384 页一组），所以 `xdes_calc_descriptor_page()` 一个 `ut_2pow_round` 就能 O(1) 定位，**不需要空闲位图**（槽位与 extent 一一对应，天然无空洞）。inode 描述"第 n 个段"，段随 DDL 动态生灭，无规律可循，只能把"有空槽的 inode 页"串成链表按需取用，于是**空闲度有两级**（页内 `FSEG_ID==0` 判空 + 页级 FREE/FULL 双链）。

2. **`init_xdes`（`i % 16384 == 0`）是整章最关键的判据**——它决定了"哪个 extent 不能整块交给段"。原因是要"牺牲"前两页（XDES 描述页 + IBUF bitmap 页，`XDES_FRAG_N_USED = 2`），所以这些 extent 只能进 `FSP_FREE_FRAG` 走碎片分配。这个约束一路影响到 `DATA_FREE` 的计算（`n_free_up -= n_free_up / (page_size.physical() / FSP_EXTENT_SIZE)`，**每 256 个 extent 扣掉 1 个**）。

3. **`space->size` 的唯一增长点是 `Fil_shard::space_extend()`**，且扩展永远是"最后一个 node"。落盘三步：① 先记 redo（`fil_op_write_space_extend`，因为 `posix_fallocate()` 有"空间已预留但元数据未更新"的原子性缺陷）② `posix_fallocate()` 只预留不写数据 ③ 只有开关要求或 fallocate 失败才 `fil_write_zeros()`（单次最多 1MB、按 `page_size` 对齐以满足 O_DIRECT）。**磁盘满时可能只加了一部分**，`success = (end == node_start + len)`。

4. **`btr_create()` 把整条链路串成一句话：建索引 = 建两个段 + 建根页**。顺序不能颠倒——`SEG_TOP` 传 `page=0` 才能产出根页，拿到页号后才能传非 0 给 `SEG_LEAF` 把 header 写回同一页。于是根页既是树的入口，**也是非叶段物理上的第一块地**（`fseg_create_general` 的 `page==0` 分支）。根页偏移 74/84 各 10 字节的 segment header，就是"逻辑索引"与"物理段"之间唯一的接缝。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Tablespace Management（File-Per-Table / General Tablespaces / Undo Tablespaces）*
- *MySQL 8.0 Reference Manual → [File Space Management](https://dev.mysql.com/doc/refman/8.0/en/innodb-file-space.html)*（页 / 区 / 段与空闲空间管理）
- *MySQL 8.0 Reference Manual → [Tablespace AUTOEXTEND_SIZE Configuration](https://dev.mysql.com/doc/refman/8.0/en/innodb-tablespace-autoextend-size.html)*（AUTOEXTEND_SIZE 的扩展规则）
- *MySQL 8.0 Reference Manual → INFORMATION_SCHEMA TABLES Table*（`DATA_FREE` 列说明）
- *MySQL 8.0 Reference Manual → INFORMATION_SCHEMA FILES Table*（`FREE_EXTENTS` / `TOTAL_EXTENTS` 列说明）

**社区文章 / 博客**
- [MySQL 文件存储结构 — code0xff](https://code0xff.org/post/2022/12/mysql文件存储结构/)：非常优秀
- [The basics of InnoDB space file layout — Jeremy Cole](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/)：经典入门（5.6 时代；lease 机制等 8.0 新内容以本篇源码为准）
- [InnoDB 的文件组织结构 — 利维坦](https://leviathan.vip/2019/04/18/InnoDB的文件组织结构/)
- [InnoDB tablespace 源码分析 — pagefault](https://www.pagefault.info/2019/02/17/innodb-tablespace-source-code-analysis.html)：质量很高
- [浅析 InnoDB 文件结构 — 腾讯云](https://cloud.tencent.com/developer/article/1472363)（`fseg_alloc_free_page_low` 七分支的通俗版）
- [InnoDB：Tablespace management（1）— Skywalker](https://zhuanlan.zhihu.com/p/446044092)
- 《MySQL 源码分析系列 5 —— ibd 解析》
- 《InnoDB tablespace 源码分析》

**内核月报**（归档 <http://mysql.taobao.org/monthly/>）
- 《MySQL · 引擎特性 · Innodb 表空间》（zongde）
- 《MySQL · 引擎特性 · InnoDB 文件系统之文件物理结构》（印风）——FLST / XDES / inode 框架来源；精确字节偏移以本篇源码核实为准
- 《MySQL · 引擎特性 · InnoDB 数据文件简述》（慕星）
- 《MySQL 引擎特性 · InnoDB 文件管理系统》（easfire）

**相关文档**
- 页通用布局、页内查找与全页类型清单：[`page_structure.md`](page_structure.md)
- 行记录格式与 offsets 数组：[`record.md`](record.md)
- LOB 专用页与 LOB 版本：[`lob.md`](lob.md)
- B-tree 页内合并与 `MERGE_THRESHOLD`：页内碎片机制（后续补充）
