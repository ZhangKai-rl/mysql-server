# MySQL I/O 全景深度解析（从 SQL 到系统调用）

> 基于 MySQL 8.0.39 源码。涵盖 **InnoDB 完整 I/O 栈**（内存层 → fil 表空间层 → `IORequest` 抽象 → `os_file` 原语 → AIO 子系统 → 系统调用）、**server 层文件 I/O**（binlog / slow / general / error log）、**I/O 线程模型**、**压缩与加密对 I/O 的影响**，以及 **MySQL 跑在云盘上的特殊行为**。
>
> **边界**：本篇讲 **MySQL 与文件系统/块设备之间**的 I/O。Buffer Pool 的替换算法与 LRU 见 [`buffer_pool.md`](buffer_pool.md)；redo 的**格式、写/刷路径、后台线程与 checkpoint 语义**见 [`redo_log.md`](redo_log.md)（本篇只保留 redo 在 I/O 全景中的**横切对比**）；崩溃恢复见 [`recovery.md`](recovery.md)；DDL 的临时文件与 row log 见 [`ddl.md`](ddl.md)；**表空间元数据与 I/O 分发（fil 层）**见 [`fil.md`](fil.md)；**刷脏批次的组织、读路径与预读**见 [`buffer_pool.md`](buffer_pool.md)；**doublewrite 的完整实现**见 [`dblwr.md`](dblwr.md)；**InnoDB 并行扫描**见 [`parallel_scan.md`](parallel_scan.md)；**云存储（EBS/云盘）本身**见 [`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [文件与表空间层（fil）](#文件与表空间层fil)
    - [os_file 与 I/O 方式（O_DIRECT / O_SYNC / fsync）](#os_file-与-io-方式o_direct--o_sync--fsync)
  - 异步 I/O
    - [AIO 子系统](#aio-子系统)
  - 写路径专题
    - [doublewrite](#doublewrite)
    - [redo log 的 I/O（归位）与各文件 I/O 总览](#redo-log-的-io归位与各文件-io-总览)
    - [server 层 I/O 全景](#server-层-io-全景)
    - [其他文件 I/O](#其他文件-io)
  - 线程模型
    - [I/O 线程模型](#io-线程模型)
  - 横切专题
    - [压缩、加密与 punch hole](#压缩加密与-punch-hole)
    - [读路径、预读与并行扫描（归位）](#读路径预读与并行扫描归位)
    - [刷脏批次与文件管理（归位）](#刷脏批次与文件管理归位)
    - [特殊 I/O 场景](#特殊-io-场景)
    - [★ 云盘上的 MySQL I/O](#-云盘上的-mysql-io)
- [相关的系统变量](#相关的系统变量)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 的 I/O 是**两套并行体系**：

1. **InnoDB 的 I/O 栈**——自己管理缓冲（Buffer Pool）、自己决定刷盘时机、自己实现异步 I/O（AIO）、自己补半页写防护（doublewrite）。与块设备的契约面**只有 6 个系统调用**（`open` / `pread` / `pwrite` / `io_submit`+`io_getevents` / `fsync`+`fdatasync` / `fcntl(O_DIRECT)`，外加 `fallocate` 打洞与 `unlink`）。
2. **server 层的文件 I/O**——binlog / relay log / slow log / general log / error log，走 `IO_CACHE` 这类通用带缓冲文件读写，**不用 O_DIRECT、不走 libaio**。

落到最底层，一切归结为两种打开文件的方式：

- **Buffered I/O**：读写都经过 **OS page cache**，写用 `write`（到 page cache 即返回），真正落盘要 `fsync`；
- **O_DIRECT**：绕过 OS page cache，直接对磁盘做对齐读写，省掉一次内存拷贝，但要求对齐且无系统缓存兜底。

`innodb_flush_method` 决定**数据文件**走哪条路（默认 `fsync`，即 Buffered），而 redo log、doublewrite、sort 临时文件又各有独立的控制开关。

### 用途

本篇回答四个问题：

1. 一次 `SELECT`、一次 `COMMIT` 到底触发哪些 I/O？走 `pwrite` 还是 `io_submit`？
2. MySQL 为什么要在 OS 之上再叠一层自己的缓冲与刷盘调度，而不是交给 OS？
3. 压缩、加密、punch hole 在哪一层做，会不会改变 I/O 大小？
4. 换到云盘（网络块存储）之后，这套栈的哪些隐含假设会失效？

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.5 | `innodb_flush_method` 六值定型；Linux native AIO（libaio）成为默认 |
| 5.6 | redo log buffer 与后台写线程分离；`innodb_io_capacity` / `innodb_io_capacity_max` 引入自适应刷脏 |
| 5.7 | 独立 undo 表空间、临时表空间（`ibt`）；page compression + punch hole |
| 8.0 | **redo 无锁化重写**：`log_writer` / `log_flusher` / `log_checkpointer` / `log_write_notifier` / `log_flush_notifier` 五个后台线程；redo 文件改为 `#innodb_redo` 目录下的 `#ib_redo<N>`；**doublewrite 从系统表空间迁出为独立 `#ib_<page_size>_<id>.dblwr` 文件**（可经 `innodb_doublewrite_dir` 放到别的设备）；`innodb_use_fdatasync` 引入；`innodb_dedicated_server` 可按内存自动推算并**静默改写 flush_method** |

> **为什么演进**：8.0 对 redo 大改是为了消除 `log_sys->mutex` 的全局争用（见[理论溯源](#理论溯源)）；doublewrite 独立成文件是为了让它可迁移到更快的设备，并解除对系统表空间的依赖。

### 全景分层图与 I/O 子系统总览

一次 I/O 从 SQL 到磁盘要穿过七层。理解这张图，后面每个子系统的位置就不用死记。

```
┌──────────────────────────────────────────────────────────────────────────┐
│ L0  SQL / 执行层                                                          │
│     JOIN / Iterators（★社区无并行查询）、ha_innobase、ha_innopart          │
└──────────┬───────────────────────────────────────────┬───────────────────┘
           │ (row 接口)                                 │ (binlog 接口)
           ▼                                            ▼
┌────────────────────────────────┐   ┌────────────────────────────────────┐
│ L1  InnoDB 内存层               │   │ L1' server 日志内存层               │
│  Buffer Pool   buf_page_get_gen│   │  binlog_cache_data / _mngr         │
│  ├ change buffer（延迟写）      │   │  ├ IO_CACHE_binlog_cache_storage   │
│  ├ AHI（纯内存，零 I/O）        │   │  └ 超限 spill 到临时文件           │
│  └ page_zip（COMPRESSED）      │   │  IO_CACHE（mysys/mf_iocache.cc）   │
│  Redo Log Buffer（log_t::buf） │   └────────────────┬───────────────────┘
│  dblwr 内存 batch buffer       │                    │
└────────────────┬───────────────┘                    │
                 ▼                                    ▼
┌────────────────────────────────┐   ┌────────────────────────────────────┐
│ L2  InnoDB 页 / 日志组织层      │   │ L2' server 日志文件层               │
│  buf_flush_*  （buf0flu）      │   │  MYSQL_BIN_LOG::Binlog_ofile       │
│  buf_read_*   （buf0rea）      │   │  flush_cache_to_file → my_b_flush  │
│  dblwr::write （buf0dblwr）    │   │  sync_binlog_file  → my_sync(fsync)│
│  log_writer / log_flusher      │   │  ordered_commit（flush/sync/commit)│
└────────────────┬───────────────┘   │  File_query_log（slow/general）    │
                 ▼                   └────────────────┬───────────────────┘
┌────────────────────────────────────────────┐        │
│ L3  InnoDB 表空间层（fil0fil.cc）            │        │
│   fil_space_t / fil_node_t                  │        │
│   Fil_shard × 68（按 space_id 取模分片）     │        │
│   fil_io → Fil_shard::do_io             │        │
│   get_AIO_mode → NORMAL / IBUF / SYNC     │        │
└────────────────┬───────────────────────────┘        │
                 ▼                                     │
┌────────────────────────────────────────────┐        │
│ L4  I/O 请求抽象层                           │        │
│   IORequest（READ/WRITE/DBLWR/LOG/          │        │
│              PUNCH_HOLE/DO_NOT_WAKE…）      │        │
│   pfs_os_* 包装（PFS file instrumentation） │        │
└──────┬───────────────────────┬─────────────┘        │
       ▼ (sync)                ▼ (async)              │
┌──────────────────┐  ┌───────────────────────────┐   │
│ L5a os_file_*    │  │ L5b AIO 子系统             │   │
│  read/write/     │  │  os_aio_func → AIO / Slot  │   │
│  flush/punch_hole│  │  s_reads / s_writes/s_ibuf │   │
│  set_size/       │  │  io_handler_thread × N     │   │
│  encrypt/decrypt │  │  → fil_aio_wait(segment)   │   │
└──────┬───────────┘  └───────────┬───────────────┘   │
       │                          │                    │
       ▼                          ▼                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ L6  系统调用                                                              │
│  pread / pwrite                   同步 I/O                                │
│  io_submit / io_getevents         libaio（★ 8.0.39 无 io_uring）          │
│  fsync / fdatasync                刷盘                                    │
│  fallocate(PUNCH_HOLE|KEEP_SIZE)  page compression 打洞                   │
│  fallocate(ZERO_RANGE)            os_file_set_size_fast                   │
│  unlink                           TRUNCATE / DROP（★同步，无异步清理）    │
│  fcntl(F_SETFL, O_DIRECT)         仅 DATA / CLONE_DATA / DBLWR 文件       │
│  O_SYNC                           仅 LOG 文件（flush_method=O_DSYNC）     │
└──────────────────────────────────────────────────────────────────────────┘
```

### I/O 子系统总览（按数据流分类）

| # | 子系统 | 入口 | 用什么 I/O | 是否 O_DIRECT |
|---|--------|------|-----------|--------------|
| 1 | 数据页读 | `buf_page_get_gen` → `buf_read_page` | 异步 AIO（`io_submit`） | 随 `innodb_flush_method` |
| 2 | 数据页写（刷脏） | `buf_flush_write_block_low` → `dblwr::write` | 同步写 dblwr + 异步写数据文件 | 随 flush_method |
| 3 | doublewrite | `Segment::write` / `Segment::flush` | **同步 `pwrite` + fsync** | 随 flush_method |
| 4 | redo 写 | `log_writer` → `Log_file_handle::write` | **同步 `pwrite`** | ❌ 否 |
| 5 | redo 刷 | `log_flusher` → `Log_file_handle::fsync` | `fsync`/`fdatasync` | — |
| 6 | binlog | `MYSQL_BIN_LOG::Binlog_ofile::write/sync` | **Buffered write + fsync** | ❌ 否 |
| 7 | slow / general log | `File_query_log` + `IO_CACHE` | Buffered write | ❌ 否 |
| 8 | error log | `log_sink_trad` / `log_sink_buffer` / `log_sink_perfschema` | Buffered write | ❌ 否 |
| 9 | DDL log | `Log_DDL::write_*` | **写在 `mysql.innodb_ddl_log` 表 + redo**，不是独立文件 | 随数据文件 |
| 10 | clone | `Clone_Handle::open_file` / `apply_data` | `OS_CLONE_DATA_FILE`（**也开 O_DIRECT**） | 随 flush_method |
| 11 | 归档 | `log_archiver_thread` / `page_archiver_thread` | 独立归档文件 | ❌ 否 |

> **一眼看出的规律**：**只有 InnoDB 的数据文件（含 dblwr、clone）可能走 O_DIRECT 与 libaio**；redo、binlog、各类日志一律是 **buffered + fsync**。这是理解 "为什么 redo 和 binlog 的性能瓶颈和数据页不一样" 的关键。

---

## 理论基础

> 本章先讲**为什么 MySQL 要自己管 I/O**（设计权衡），再讲**它脚下踩的 OS 机制**（三种 I/O 方式）。顺序不能反——不理解前者，后者就只是一堆 flag。

### 设计思想与权衡

#### 一、为什么数据库要绕过 OS 自己管 I/O

OS 的 page cache 已经很完善（LRU、readahead、write-back 合并），数据库为什么还要自己搞一层 Buffer Pool？三个原因是**结构性**的：

| OS 不知道的事 | 后果 | MySQL 的对策 |
|--------------|------|------------|
| **页的语义热度** | OS 的 LRU 按访问时间排序，但数据库知道"根页永远热""全表扫描的页扫完就扔" | Buffer Pool 的 **old/new 中点替换 + `old_blocks_time` 时间窗**，专门抗扫描污染（见 `buffer_pool.md`） |
| **WAL 顺序约束** | OS 完全可以先把数据页写下去、后写 redo，那崩溃就完了 | `buf_flush_write_block_low` 在写页前**强制** `log_write_up_to(..., flush_to_disk=true)` |
| **事务边界** | OS 不知道哪些页属于未提交事务，可能在事务未提交时就把它刷下去 | 靠 redo + undo 保证可回滚（**steal** 策略） |

**代价**：这套自制缓冲带来了 **double buffering**（同一页在 page cache 与 BP 各存一份），`O_DIRECT` 就是为了消除它而存在的。

> **被否决的方案**：完全交给 OS（即 PG 的做法）。PG 的 shared_buffers 官方建议只给内存的 25%，剩下靠 OS cache；MySQL 默认给 50~75% 并强烈建议 `O_DIRECT`。**MySQL 选了"自己控制"这条路**，换来可预测性，代价是复杂度（这套代码有几万行）。

#### 二、为什么 redo 用同步写、数据页用异步 AIO

| | redo | 数据页 |
|---|---|---|
| 在关键路径上？ | **是**（提交延迟 = fsync 延迟） | 否（后台刷脏） |
| I/O 模式 | 顺序追加、量小 | 随机、量大、可攒批 |
| 并发度 | 单线程顺序写即可 | 需要几百个并发请求喂饱设备 |
| 结论 | **同步 `pwrite`**（AIO 的复杂度换不来收益） | **异步 `io_submit`**（必须并发） |

这就是为什么 `os_aio_func` 里有一条 `ut_a(!type.is_log)`——**redo 永远不许走 AIO 路径**。

#### 三、为什么 doublewrite 存在，以及它什么时候可以关掉

块设备的契约**不保证多块写原子性**：写一个 16KB 页时断电，可能一半是新的、一半是旧的（torn page）。redo 无法修复这种损坏（redo 是"页内偏移 + 内容"的物理日志，基于"页的其他部分是对的"这一前提）。

于是只能**先整页写到一个地方并 fsync，确认完整后再写到真实位置**——这就是 doublewrite。

> **若设备提供 16KB 原子写，doublewrite 就可以关掉，写放大直接减半。** MySQL 留了口子（`fil_fusionio_enable_atomic_write` 针对 FusionIO、`innodb_doublewrite=OFF`），但社区版默认不敢依赖，因为通用块存储契约里没有这一条。这是"最小契约"设计的长期成本，详见 [`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)「块语义的最小契约」。

#### 四、为什么 server 层（binlog）不用 O_DIRECT / AIO

binlog 是**顺序追加、量小**，且组提交已经把 fsync 摊薄成"一批事务一次"。AIO 带来的并发收益为零，复杂度却是实打实的（槽位管理、完成回调、部分写重投）。所以整个 `sql/` + `mysys/` **没有任何 O_DIRECT 调用**（全仓库 O_DIRECT 只出现在 InnoDB 的 `os0file.cc` 与 `fil_fusionio_enable_atomic_write`）。

#### 五、这套设计的代价清单（权衡的另一半）

| 代价 | 说明 |
|------|------|
| **O_DIRECT 要求对齐** | MySQL **不做运行时对齐校验**，而是靠分配器保证（`ut::aligned_alloc`）+ redo 侧的断言。所以 O_DIRECT 用错场景（如 tmpfs）只会 `EINVAL` 后告警回退 |
| **AIO 并发度有上限** | 在途 I/O 数 = `innodb_write_io_threads × 8 × OS_AIO_N_PENDING_IOS_PER_THREAD`，默认 4 个线程**远远喂不饱现代 SSD/云盘** |
| **fsync 失败即自杀** | `EIO` → `ib::fatal`（见[核心实现二](#核心实现二os_file-与-io-方式o_direct--o_sync--fsync)），这是故意的：不能带着可能的不一致继续跑 |
| **I/O 没有超时** | `pwrite` / `fsync` / `io_getevents` 全部无限等待。本地盘坏会返回 EIO（快速崩溃），**云盘故障更常表现为 hang**（整实例挂起）。见 [★ 云盘上的 MySQL I/O](#-云盘上的-mysql-io) |

### 理论溯源

| 理论 / 论文 | 核心主张 | 源码落点 |
|------------|---------|---------|
| **ARIES（Mohan et al., 1992）+ Gray & Reuter《Transaction Processing》** | **steal / no-force** 缓冲管理 + **Write-Ahead Logging**：允许未提交事务的页落盘（steal）、允许提交时不刷页（no-force），但**必须**先写日志 | **`buf_flush_write_block_low` 里的 `log_write_up_to(..., flush_to_disk=true)` 就是 WAL 的代码化身**；`trx_flush_log_if_needed_low` 是 no-force 的提交侧 |
| **五分钟法则（Gray & Putzolu, 1987）** | 页的访问频率 × 页大小 vs 内存/磁盘价格，决定哪些页该常驻内存 | Buffer Pool 存在的经济学依据（不是"越大越好"，而是"边际收益低于内存成本时停止"） |
| **Volcano / 迭代器模型（Graefe 1990）** | 一次一行（`ha_rnd_next`） | 决定了 MySQL 的读 I/O 是**逐行触发的随机读**，而不是批量列式扫描（也是为什么预读机制必须存在） |
| **libaio vs io_uring** | io_uring 用环形队列消除系统调用开销 | MySQL 8.0.39 **仍用 libaio**（`LINUX_NATIVE_AIO`），全仓库 `io_uring` **0 匹配** |
| **Chain/Quorum replication** | 分布式副本模型 | 不在 MySQL 侧，见 `../cloud/cloud_storage.md` |

### 算法与数据结构

| 机制 | 结构 | 为什么这么设计 |
|------|------|--------------|
| **I/O 请求描述** | `class IORequest`（`include/os0file.h`）——**位标志 + 成员对象**：位标志有 `READ/WRITE/DBLWR/DATA_FILE/LOG/DO_NOT_WAKE/PUNCH_HOLE/NO_COMPRESSION/ROW_LOG/IGNORE_MISSING/DISABLE_PARTIAL_IO_WARNINGS/DISABLE_PUNCH_HOLE_OPTIMISATION` | 一个 32 位整数携带全部 I/O 语义，避免参数爆炸。**注意：没有 `COMPRESS`/`ENCRYPT` 位**——压缩/加密用成员 `m_compression` / `m_encryption` 对象表达 |
| **表空间元数据分片** | `Fil_shard` × **68**（`fil0fil.cc`），路由 = `space_id % 64`，undo 走 `UNDO_SHARDS_START` 之后的专用分片 | 把 `fil_system` 全局 mutex 拆成 68 把，降低高并发 DDL/刷表的锁争用。undo 独立分片是为了避免 undo 的 extend 卡住普通表空间 |
| **AIO 槽位** | `struct Slot`（`os0file.cc`）+ 三个数组 `s_reads` / `s_writes` / `s_ibuf`；每个 segment 一个 `io_context_t` | **`s_ibuf` 单独一个数组**是防死锁设计：change buffer 的页读如果和其他读抢同一批槽位，可能因为槽位耗尽而等不到自己发起的读完成 |
| **dblwr 批量缓冲** | 内存 batch buffer + `Segment` | 把 N 次随机 fsync **摊成 1 次**，这是刷脏吞吐的关键（云盘上尤其明显） |
| **文件描述符管理** | `Fil_shard::close_files_in_LRU` | 受 `innodb_open_files` 限制，超过时关 LRU 尾部文件句柄（表很多时必需） |

### 他库对比与演进动机

| 数据库 | 缓存策略 | O_DIRECT | 说明 |
|--------|---------|----------|------|
| **MySQL / InnoDB** | 自己管 Buffer Pool（默认 50~75% 内存） | ✅ 支持且**推荐** | 自己控制刷盘顺序与时机 |
| **PostgreSQL** | 自己管 shared_buffers（**建议仅 25%**），其余靠 OS page cache | ❌ **无 O_DIRECT 选项** | 依赖 OS 的缓存与回写；`effective_cache_size` 只是优化器提示，不是分配 |
| **Oracle** | 自己管 SGA；可选 ASM 裸设备 / 直接 NFS | 裸设备时代天然绕过 | 历史最"重"的方案：连文件系统都不要 |

**演进动机**：8.0 对 redo 的大规模重写（五个后台线程）是为了消除 `log_sys->mutex` 的全局争用——在高并发写入下，所有用户线程都要抢这把锁来推进 `lsn`，成为瓶颈。改后的模型是**用户线程只往 ring buffer 里 memcpy，由专用线程负责 write / fsync / checkpoint**，用户线程通过 `log_write_notifier` / `log_flush_notifier` 被唤醒。

---

### Linux 的三种 I/O 方式

### 1. Buffered I/O：write 到 page cache，fsync 才落盘

这是**默认**、也是最容易被误解的方式。关键点：**`write` 成功 ≠ 数据落盘**。

```
write(fd, buf, n)
  │  用户态 buffer ──拷贝──▶ 内核 page cache（标记为 dirty page）
  │  立即返回（此时数据只在内存，断电即丢）
  ▼
fsync(fd)
  │  把该文件的 dirty page 刷到磁盘，返回时数据已持久化
  ▼
  磁盘
```

- **优点**：write 快（只写内存）；OS 做**写合并**（write-back，多次小写合并成一次磁盘写）和**读预取**（readahead）；反复读能命中缓存。
- **缺点**：数据页在内核 page cache 里存了一份，如果上层（InnoDB）又有一份自己的缓存，就是 **double buffering**（双重缓存），内存浪费一倍。
- **持久化必须 fsync**：否则断电丢最近 N 秒的数据。这就是 `innodb_flush_log_at_trx_commit=1` 每次提交都要 fsync redo 的原因。

### 2. O_DIRECT：绕过 page cache

用 `fcntl(fd, F_SETFL, O_DIRECT)` 打开文件，读写**直接落盘**，不经过 page cache。

- **优点**：省一次"page cache ↔ 用户态"拷贝，消除 double buffering，内存占用更可控；
- **代价**：没有系统缓存兜底（重复读不快）；要求**地址对齐 + 长度对齐**（通常按扇区 512B/4K 对齐），InnoDB 为此准备了 `srv_page_size` 对齐的缓冲区；**tmpfs 不支持 O_DIRECT**（会 `EINVAL`）。

源码（os0file.cc）：

```cpp
#elif defined(O_DIRECT)
  if (fcntl(fd, F_SETFL, O_DIRECT) == -1) { ... }  // 失败则告警后回退到 buffered
```

### 3. O_DSYNC / O_SYNC：每次 write 同步

打开文件时带 `O_DSYNC`，每次 `write` 相当于"write + 落盘"原子完成，**不需要额外 fsync**。适合 redo log 这类"写了就要持久"的场景，省掉一次系统调用。缺点是不能利用 OS 写合并，每次写都落盘。

三者对比：

| 方式 | 经过 page cache | 持久化时机 | 系统调用 | 典型用途 |
|------|----------------|-----------|---------|---------|
| Buffered + fsync | 是 | fsync 时 | write + fsync | 数据文件（默认） |
| O_DIRECT | 否 | 每次写直接落盘 | 对齐的 pread/pwrite | 数据文件（flush_method=O_DIRECT） |
| O_DSYNC | 是（但 write 即刷） | 每次 write | 单次 write | redo log（flush_method=O_DSYNC） |

## 核心实现

### 文件与表空间层（fil）

> **★ fil 层的完整剖析见专篇 [`fil.md`](fil.md)**——三层结构与 68 分片、文件类型常量、`Fil_shard::do_io` 的压缩/加密/打洞注入、**AIO 模式三选一的完整判定链（含"缺页读为何是同步 `pread`"）**、并发控制 flag 体系、`space_extend`、`space_flush` 的 fsync 合并。
> 本节只保留 I/O 全景所需的概要。

所有 InnoDB 的表空间 I/O 都汇聚到 `fil_io` 这一个入口，再由它按 `space_id` 路由到某个 `Fil_shard`。

#### 文件类型：不是 enum，是常量

`include/os0file.h` —— 用 `static const ulint` 而非 enum（避免 `-fshort-enums` 带来的 ABI 问题，这类细节见 `../server/plugin/abi.md`）：

| 常量 | 值 | 说明 |
|---|---|---|
| `OS_DATA_FILE` | 100 | 数据文件（.ibd / 系统 / undo / temp） |
| `OS_LOG_FILE` | 101 | redo log |
| `OS_BUFFERED_FILE` | 102 | 小文件 / 写入字节数非扇区整数倍时强制 buffered |
| `OS_CLONE_DATA_FILE` | 103 | clone 拷贝的数据文件 |
| `OS_CLONE_LOG_FILE` | 104 | clone 拷贝的 redo |
| `OS_DBLWR_FILE` | 105 | doublewrite 文件 |
| `OS_REDO_LOG_ARCHIVE_FILE` | 105 | redo archive（与 DBLWR **同值**，历史遗留） |

文件后缀（`enum ib_file_suffix`，`fil0fil.h`）：`.ibd` / `.cfg` / `.cfp` / `.ibt` / `.ibu` / `.dblwr` / `.bdblwr`。

#### 三层结构与分片

```
fil_system（全局单例）
  └── Fil_shard × 68                 ← 路由：space_id % 64；undo 走 UNDO_SHARDS_START 之后的专用分片
        ├── fil_space_t（一个表空间）
        │     └── files: vector<fil_node_t>   ← 一个物理文件（handle / size / flush_size /
        │                                        n_pending_ios / n_pending_flushes / block_size）
        └── LATCH_ID_FIL_SHARD mutex（每 shard 一把）
```

**为什么要 68 个分片**：把 `fil_system` 的全局 mutex 拆成 68 把。高并发建表 / 删表 / 刷表空间时，不同 `space_id` 落在不同 shard，锁争用大幅下降。**undo 单独分片**是因为 undo 表空间的 extend 会持锁较久，不能让它卡住普通表空间。

#### 主链路：`fil_io` → `Fil_shard::do_io`

`storage/innobase/fil/fil0fil.cc`：

```cpp
dberr_t fil_io(const IORequest &type, bool sync, const page_id_t &page_id,
               const page_size_t &page_size, ulint byte_offset, ulint len,
               void *buf, void *message) {
  auto shard = fil_system->shard_by_id(page_id.space);
  ...
  auto const err = shard->do_io(type, sync, page_id, page_size, byte_offset,
                                len, buf, message);
  ...
  return err;
}
```

`Fil_shard::do_io` 的尾部（`fil0fil.cc`）——这里决定了 I/O 的所有附加语义：

```cpp
  auto offset = (os_offset_t)page_no * page_size.physical;
  offset += byte_offset;
  ...
  /* Don't compress the log, page 0 of all tablespaces, tables compressed with
   the old compression scheme and all pages from the system tablespace. */
  if (req_type.is_write && !page_size.is_compressed &&
      page_id.page_no > 0 && IORequest::is_punch_hole_supported &&
      file->punch_hole) {
    req_type.set_punch_hole;
    req_type.compression_algorithm(space->compression_type);
  } else {
    req_type.clear_compressed;
  }

  /* Set encryption information. */
  fil_io_set_encryption(req_type, page_id, space);
  req_type.block_size(file->block_size);
  ...
  /* Queue the aio request */
  err = os_aio(
      req_type, aio_mode, file->name, file->handle, buf, offset, len,
      fsp_is_system_temporary(page_id.space) ? false : srv_read_only_mode,
      file, message);
```

**逐段解释**：

1. **地址映射**：`offset = page_no × page_size.physical`——**LBA 完全由 `page_no << 14` 决定**。这意味着页的随机性来自 `page_no`，与底层设备的物理布局无关（在云盘上，这层映射之后还要再做一次 LBA→chunk 的映射，MySQL 不知情）。
2. **punch hole 决策（page compression）**：写操作且非压缩表且非 page 0 且文件系统支持打洞 → 打上 `PUNCH_HOLE` 与压缩算法。注意**排除 page 0**（页 0 是表空间头，不能被"打洞"成稀疏）。
3. **加密注入**：`fil_io_set_encryption` 把密钥信息塞进 `IORequest`，真正的加密在更底层的 `os_file_encrypt_page` 做（见[核心实现九](#核心实现九压缩加密与-punch-hole)）。
4. **下发**：`os_aio` 是同步/异步的分流点。

#### AIO 模式的三选一

`Fil_shard::get_AIO_mode`（`fil0fil.cc`）：

| 模式 | 常量 | 何时用 |
|------|------|--------|
| `AIO_mode::SYNC` | 24 | `sync == true`——直接同步 `pwrite`，不进 AIO 子系统 |
| `AIO_mode::NORMAL` | 21 | 普通异步 I/O（读页、刷脏） |
| `AIO_mode::IBUF` | 22 | **change buffer 相关的页读** |

> `IBUF` 模式为什么单独存在：ibuf merge 时需要读入二级索引页，如果它和普通读抢同一批 AIO 槽位，可能出现"所有槽位都被 ibuf merge 的读占满，而 ibuf merge 又在等这些读完成"的死锁。**单独一个 `s_ibuf` 数组 + 单独一个线程**把这个环切断。

#### I/O 并发控制：flag 与计数器体系（概要）

对**同一个文件**的并发操作不靠锁（底层 `pread`/`pwrite` 天然安全），而靠一组 flag + 计数器：`n_pending_ios`（阻止 close）、`n_pending_flushes`（阻止 drop / 合并 fsync）、`is_being_extended`（扩展之间互斥）、`stop_new_ops`（DROP/TRUNCATE 中禁止新操作）。

> **这解释了"为什么大表 DROP/TRUNCATE 会卡住"**——要等三类 pending 归零。
> **完整机制（含 `can_be_closed` 的四个条件、`is_being_extended` 为何不能只用 `n_pending_ios`）见 [`fil.md`](fil.md)「I/O 并发控制」。**

---

### os_file 与 I/O 方式（O_DIRECT / O_SYNC / fsync）

`innodb_flush_method` 决定数据文件走哪条路，redo log、doublewrite、sort 临时文件又各有独立开关。

#### 枚举值与语义

`innodb_flush_method`（ha_innodb.cc，只读变量）六个取值，映射到 `srv_unix_flush_t`（srv0srv.h）：

| 取值 | 数据文件 | redo log |
|------|---------|---------|
| `fsync`（默认） | Buffered + `fsync` | Buffered + `fsync` |
| `O_DSYNC` | Buffered + `fsync` | `O_DSYNC` 打开，**write 即同步，免 fsync** |
| `littlesync` | 写后不 fsync | 写后 fsync |
| `nosync` | 不 fsync | 不 fsync（**危险**，仅测试用） |
| `O_DIRECT` | **O_DIRECT 绕过缓存** + 仍 `fsync` | Buffered + `fsync` |
| `O_DIRECT_NO_FSYNC` | **O_DIRECT** + 不 fsync | Buffered + `fsync` |

**关键点**：`O_DIRECT` 系列**只影响数据文件**，redo log 的 fsync 不受影响（redo 独立由 `innodb_flush_log_at_trx_commit` 控制）。`O_DIRECT` 与 `O_DIRECT_NO_FSYNC` 的区别在于——数据文件用 O_DIRECT 后，`fsync` 是否还需要（因为 O_DIRECT 绕过了 page cache，很多场景 fsync 意义变小，但文件元数据/目录项可能仍需刷）。

#### 数据文件的 O_DIRECT 判定

数据文件到底用不用 O_DIRECT，源码在 os0file.cc，只对**数据类文件**调用：

```cpp
/* We disable OS caching (O_DIRECT) only on data files. */
if ((!read_only || type == OS_CLONE_DATA_FILE) && *success &&
    (type == OS_DATA_FILE || type == OS_CLONE_DATA_FILE || type == OS_DBLWR_FILE) &&
    (srv_unix_file_flush_method == SRV_UNIX_O_DIRECT ||
     srv_unix_file_flush_method == SRV_UNIX_O_DIRECT_NO_FSYNC)) {
  os_file_set_nocache(file.m_file, name, mode_str);   // fcntl O_DIRECT
}
```

即：`OS_DATA_FILE`（.ibd）、`OS_DBLWR_FILE`（doublewrite）、clone 数据文件三者，在 `O_DIRECT` 模式下才绕过 page cache。

#### 双重缓存（double buffering）问题

默认 `fsync` 模式下，InnoDB 数据页在内存里**存了两份**：

```
读：磁盘 ──▶ OS page cache ──▶ Buffer Pool（16KB 数据页）
写：Buffer Pool ──▶ OS page cache ──▶ 磁盘
```

一份在 OS page cache（内核），一份在 InnoDB Buffer Pool（用户态）。同一页占两份内存，且写放大。**`O_DIRECT` 的目的就是消除这个重叠**——数据文件绕过 page cache，只进 Buffer Pool。

这也是为什么 `innodb_dedicated_server=ON` 时（ha_innodb.cc），若用户没显式指定 flush_method，会自动设成 `O_DIRECT_NO_FSYNC`——专属服务器上内存更该留给 Buffer Pool，而非被 page cache 重复占用。

#### ★ I/O 层的总收口：`os_file_io`（压缩 → 加密 → 读写 → 部分重试）

所有**同步** I/O 最终都汇聚到一个函数：`os_file_io`（`os0file.cc`）。调用链是
`os_file_write_func` → `os_file_write_page`→ `os_file_pwrite`→ **`os_file_io`** → `SyncFileIO::execute`。

它承担四件事。前两件（压缩、加密）的代码：

```cpp
[[nodiscard]] static ssize_t os_file_io(const IORequest &in_type,
                                        os_file_t file, void *buf, ulint n,
                                        os_offset_t offset, dberr_t *err,
                                        const file::Block *e_block) {
  ulint original_n = n;
  file::Block *block{};
  IORequest type = in_type;
  ssize_t bytes_returned = 0;
  byte *encrypt_log_buf = nullptr;

  if (type.is_compressed) {
    /* We don't compress the first page of any file. */
    ut_ad(offset > 0);
    ut_ad(!type.is_log);
    if (e_block == nullptr) {
      block = os_file_compress_page(type, buf, &n);
    } else {
      ...
    }
  }

  /* We do encryption after compression, since if we do encryption
  before compression, the encrypted data will cause compression fail
  or low compression rate. */
  if ((type.is_encrypted || e_block != nullptr) && type.is_write) {
    if (!type.is_log) {
      /* We don't encrypt the first page of any file. */
      auto compressed_block = block;
      ut_ad(offset > 0);

      /* If dblwr is involved, we should not be reaching here, because we
      encrypt the page at higher layer so that the same encrypted page can be
      written to the dblwr file and the data file. During importing an
      encrypted tablespace, we reach here. */
      if (e_block == nullptr) {
        block = os_file_encrypt_page(type, buf, n);
      } else {
        block = const_cast<file::Block *>(e_block);
      }
```

**三条硬规则**（都有断言守护）：

1. **★ 顺序固定：先压缩，后加密**。注释给了理由——先加密的话，密文熵高几乎不可压缩，会导致压缩失效或压缩率极低。
2. **★ 第一页永不压缩、永不加密**（`ut_ad(offset > 0)`）。页 0 是表空间/文件头，必须明文可读（页 0 里存着 space_id、page_size 等元信息，启动时第一件事就是读它）。
3. **★ dblwr 场景的加密在更高层完成**：注释明说——为了让**同一份密文**写进 dblwr 文件和数据文件，加密提前到 `Double_write::get_encrypted_frame`，这里通过 `e_block` 参数传进来复用。只有 IMPORT 加密表空间才会走到这里现算。

第三件（实际读写）与第四件（部分 I/O 重试）：

```cpp
  for (ulint i = 0; i < NUM_RETRIES_ON_PARTIAL_IO; ++i) {
    ssize_t n_bytes = sync_file_io.execute(type);

    /* Check for a hard error. Not much we can do now. */
    if (n_bytes < 0) {
      break;

    } else if ((ulint)n_bytes + bytes_returned == n) {
      bytes_returned += n_bytes;

      if (offset > 0 && (type.is_compressed || type.is_read)) {
        *err = os_file_io_complete(type, file, reinterpret_cast<byte *>(buf),
                                   original_n, offset, n);
      } else {
        *err = DB_SUCCESS;
      }

      if (block != nullptr) {
        os_free_block(block);
      }
      ...
      return (original_n);
    }

    /* Handle partial read/write. */
    ut_ad((ulint)n_bytes + bytes_returned < n);
    bytes_returned += (ulint)n_bytes;

    if (!type.is_partial_io_warning_disabled) {
      const char *op = type.is_read ? "read" : "written";
      ib::warn(ER_IB_MSG_812)
          << n << " bytes should have been " << op << ". Only "
          << bytes_returned << " bytes " << op << ". Retrying"
          << " for the remaining bytes.";
    }

    /* Advance the offset and buffer by n_bytes */
    sync_file_io.advance(n_bytes);
  }
```

- **部分 I/O 重试 10 次**（`NUM_RETRIES_ON_PARTIAL_IO = 10`，`os0file.cc`），每次用 `sync_file_io.advance(n_bytes)` 把 offset 和 buffer 前移。**重试耗尽后 `n < 0` 跳出，最终返回错误。**
- **读/压缩页的完成回调**：`os_file_io_complete`在这里做**解密与解压**（与写路径的压缩/加密对称）。
- 每次成功都会 `os_free_block(block)` 释放临时的压缩/加密缓冲（`file::Block` 有 `MAX_BLOCKS = 128` 的缓存池）。

**写失败时的诊断信息**（`os_file_write_page`，`os0file.cc`）值得一提——它把常见的三种根因一次说清：

```cpp
  if ((ulint)n_bytes != n && !os_has_said_disk_full) {
    ib::error(ER_IB_MSG_814) << "Write to file " << name << " failed at offset "
                             << offset << ", " << n
                             << " bytes should have been written,"
                                " only " << n_bytes << " were written."
                                " Operating system error number " << errno << "."
                                " Check that your OS and file system"
                                " support files of this size."
                                " Check also that the disk is not full"
                                " or a disk quota exceeded.";
    ...
    os_has_said_disk_full = true;
  }
```

三个提示：① 文件系统不支持这么大的文件；② 磁盘满；③ 磁盘配额超限。且用 `os_has_said_disk_full` 保证**只报一次**（避免刷屏）。

#### fsync 的失败处理：EIO 即崩溃

`storage/innobase/os/os0file.cc`：

```cpp
static int os_file_fsync_posix(os_file_t file) {
  ulint failures = 0;
  ...
  for (;;) {
    ++os_n_fsyncs;

#if defined(HAVE_FDATASYNC) && defined(HAVE_DECL_FDATASYNC)
    const auto ret = srv_use_fdatasync ? fdatasync(file) : fsync(file);
#else
    const auto ret = fsync(file);
#endif

    if (ret == 0) {
      return (ret);
    }

    switch (errno) {
      case ENOLCK:
        ++failures;
        ut_a(failures < 1000);
        ...
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        break;

      case EIO:
        ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_1358)
            << "fsync returned EIO, aborting.";
        break;

      case EINTR:
        ++failures;
        ut_a(failures < 2000);
        break;

      default:
        ut_error;
        break;
    }
  }
}
```

**分支解释**：

- `innodb_use_fdatasync`（默认 **false**）决定用 `fsync` 还是 `fdatasync`。`fdatasync` 只刷数据不刷不必要的元数据（mtime/atime 等），**更快**；对 redo 这种"大小固定、只追加"的文件尤其合适。风险是文件大小变化不受保护——但 InnoDB 自己管文件大小，所以基本安全。
- `ENOLCK` / `EINTR` → 重试（有 `ut_a(failures < N)` 上限，超限直接 abort）。
- **`EIO` → `ib::fatal` 立即崩溃**（**故意设计**）：fsync 返回 EIO 意味着数据可能没落盘，继续写会导致**静默损坏**，不如崩掉让崩溃恢复兜底。

其余 I/O 错误的处理策略在 `os_file_handle_error_cond_exit`（`os0file.cc`）：

| 错误类 | 处理 | 是否重试 |
|--------|------|---------|
| `ENOSPC`（盘满） | 只报一次错，继续等人工介入 | ❌ |
| `EAGAIN` / `EINTR` / `EABORT` | 立即重试 | ✅ |
| `OS_FILE_SHARING_VIOLATION` | **睡 10 秒**后重试 | ✅ |
| 内存/资源不足（`ENOMEM`/`EMFILE`） | 睡 100ms 后重试 | ✅ |
| 其他（含 `EIO`） | **`srv_fatal_error` 崩溃** | — |

> **设计哲学**：**MySQL 对 I/O 错误的默认策略是"崩溃而不是静默继续"**（`ENOSPC` 除外）。这个选择在云盘上有个反直觉的副作用，见 [★ 云盘上的 MySQL I/O](#-云盘上的-mysql-io)。

#### O_DIRECT 的对齐：靠分配器保证，不是运行时校验

一个常见误解是"MySQL 每次 I/O 前会检查缓冲区对齐"。**实际上没有**。相关代码只有三处：

| 函数 | 作用 |
|------|------|
| `os_is_o_direct_supported`（`os0file.cc`） | 启动时建临时文件试 `open(..., O_DIRECT)` 探测支持性 |
| `os_file_set_nocache`（`os0file.cc`） | `fcntl(fd, F_SETFL, O_DIRECT)`，**失败只 warn 不 fatal**（自动回退到 buffered） |
| `os_fusionio_get_sector_size`（`os0file.cc`） | 探测真实扇区大小（`MAX_SECTOR_SIZE = 4096`） |

对齐由**分配时保证**：

- `ut::aligned_alloc(n, os_io_ptr_align)`（`os0file.cc`）
- buffer pool chunk 与 `log_t::buf` 天然页对齐（`LOG_BUFFER_ALIGNMENT`）
- redo 侧有**断言**：`Log_file_handle::prepare_io_request` 断言 `size % OS_FILE_LOG_BLOCK_SIZE == 0` 与 `offset % OS_FILE_LOG_BLOCK_SIZE == 0`

> 结论：对齐是"**设计约束 + 分配器保证 + redo 侧断言**"，不是"每次 I/O 前 if (misaligned) error"。所以 tmpfs 这类不支持 O_DIRECT 的文件系统，表现是**告警后回退**，不是崩溃。

---

### AIO 子系统

#### 同步 I/O 与异步 I/O 的分工

MySQL 有两条**完全不同**的 I/O 下发路径，分岔点在 `os_aio_func`（`os0file.cc`）：

```cpp
  if (aio_mode == AIO_mode::SYNC) {
    /* This is actually an ordinary synchronous read or write:
    no need to use an i/o-handler thread. */
    if (type.is_read) {
      return (os_file_read_func(type, name, file.m_file, buf, offset, n));
    }

    ut_ad(type.is_write);
    return (os_file_write_func(type, name, file.m_file, buf, offset, n));
  }
```

即：**`SYNC` 模式完全绕过 AIO 子系统**，直接 `pread`/`pwrite`，不占槽位、不需要 io 线程收割。模式由 `Fil_shard::get_AIO_mode`（`fil0fil.cc`）根据 `fil_io(type, sync, ...)` 的 `sync` 参数与是否 ibuf 操作决定。

**走同步的场景**：

| 场景 | 为什么是同步 |
|------|------------|
| **redo 写** | 根本不进 `os_aio`（`ut_a(!type.is_log)`），同步 `pwrite` |
| **doublewrite 文件写** | `Segment::write` → `os_file_write_retry`，**永远**同步 `pwrite` + fsync |
| **单页同步刷盘**（`BUF_FLUSH_SINGLE_PAGE` 或 `sync=true`） | 需要立即回收该页（LRU 淘汰路径），等不了异步 |
| `buf_flush_sync_all_buf_pools` | shutdown / checkpoint 前，必须确认落盘 |
| 启动/恢复期的部分 I/O | io 线程可能尚未就绪 |

**走异步的场景**：

| 场景 | 说明 |
|------|------|
| 普通缺页读（`buf_read_page`） | **对调用者是同步等待的，但下发是异步 AIO**——这样多个并发缺页能合并成一批 I/O |
| 批量刷脏（`buf_flush_do_batch`） | page cleaner 批量下发 |
| 预读（线性 / 随机） | 异步，不阻塞当前查询 |
| ibuf merge 读 | `AIO_mode::IBUF`，走独立的 `s_ibuf` 数组 |

> **为什么同步路径不能省**：同步路径不占 AIO 槽位、不需要回调上下文、调用者自己等。在"必须立刻拿到结果"或"批量提交后统一唤醒"（`IORequest::DO_NOT_WAKE`）的场景下，它比异步更简单可靠。另外注意 **异步并不等于"调用者不等"**——缺页读对调用者仍然是同步语义，异步只是下发方式。

#### 三套实现并存，8.0.39 仍无 io_uring

`os0file.cc` 的架构注释描述了一个 AIO 抽象层下的三套实现：

| 实现 | 条件 | 系统调用 |
|------|------|---------|
| **Linux native AIO**（默认） | `LINUX_NATIVE_AIO` + `srv_use_native_aio=true` | `io_submit` / `io_getevents`（libaio） |
| **Windows native AIO** | `_WIN32` | `ReadFile`/`WriteFile` + `OVERLAPPED` + `WaitForMultipleObjects` |
| **Simulated AIO** | 以上都不可用时 | 槽位入队 → `SimulatedAIOHandler` 线程 → `pwrite` |

> **8.0.39 全仓库 grep `io_uring` → 0 匹配。** 仍停留在 libaio。`SimulatedAIOHandler`（`os0file.cc`）**依然存在**，它会合并最多 `OS_AIO_MERGE_N_CONSECUTIVE = 64` 个连续 I/O。`innodb_use_native_aio` 变量也仍在（默认 true）。

#### ★ native AIO 的降级探测：tmpdir 会决定用哪套

选择逻辑在 `AIO::start`（`os0file.cc`）：

```cpp
#if defined(LINUX_NATIVE_AIO)
  /* Check if native aio is supported on this system and tmpfs */
  if (srv_use_native_aio && !is_linux_native_aio_supported) {
    ib::warn(ER_IB_MSG_829) << "Linux Native AIO disabled.";

    srv_use_native_aio = false;
  }
#endif /* LINUX_NATIVE_AIO */
```

`is_linux_native_aio_supported`（`os0file.cc`）检查**两件事**：

```cpp
bool AIO::is_linux_native_aio_supported {
  int fd;
  io_context_t io_ctx;
  const char *name;

  if (!linux_create_io_ctx(1, &io_ctx)) {
    /* The platform does not support native aio. */

    return (false);

  } else if (!srv_read_only_mode) {
    /* Now check if tmpdir supports native aio ops. */
    fd = innobase_mysql_tmpfile(nullptr);

    if (fd < 0) {
      ib::warn(ER_IB_MSG_763) << "Unable to create temp file to check"
                                 " native AIO support.";

      return (false);
    }
```

1. 能否创建 `io_context`（平台支不支持 libaio）；
2. ★ **在 `tmpdir` 里建一个临时文件，测试它支不支持 native AIO**。

> **这是个真实的运维坑**：如果 `tmpdir` 指向 **tmpfs**（不支持 O_DIRECT / libaio），探测会失败 → **整个 native AIO 被禁用** → 静默退化到模拟 AIO，性能大幅下降。唯一线索是 error log 里的 "**Linux Native AIO disabled.**"。
>
> 所以：**不要把 `tmpdir` 放在 tmpfs 上**，除非你明确知道后果。

#### 模拟 AIO（SimulatedAIOHandler）的工作原理

当 native AIO 不可用时（非 Linux/Windows，或上面探测失败），走模拟路径：

```
① os_aio_func 占一个 Slot（把请求放进去）
② 若是 "wake" 类型 → AIO::wake_simulated_handler_thread 唤醒对应 segment 的 io 线程
③ io 线程（os_aio_simulated_handler）扫描自己 segment 的槽位
④ SimulatedAIOHandler 合并最多 OS_AIO_MERGE_N_CONSECUTIVE(64) 个连续 I/O
⑤ 用 pread / pwrite 真正执行（连续的合并成一次大 I/O）
⑥ 完成后逐个触发回调
```

`OS_AIO_MERGE_N_CONSECUTIVE = 64`（`os0file.cc`）。

唤醒逻辑（`os0file.cc`）就是遍历该 segment 的槽位，发现 `is_reserved` 就 `os_event_set`：

```cpp
  ulint n = slots_per_segment;
  ulint offset = segment * n;

  /* Look through n slots after the segment * n'th slot */

  acquire;

  const Slot *slot = at(offset);

  for (ulint i = 0; i < n; ++i, ++slot) {
    if (slot->is_reserved) {
      /* Found an i/o request */

      release;

      os_event_set(os_aio_segment_wait_events[global_segment]);

      return;
    }
  }

  release;
```

**为什么模拟 AIO 需要"合并"**：它用普通 `pread`/`pwrite`，每次系统调用有固定开销；把连续地址的多个请求合并成一次大 I/O，是在没有 `io_submit` 批量能力时的补偿手段。**这也解释了为什么它比 native AIO 慢**：合并需要扫描与等待，且无法像 `io_submit` 那样把多个不连续请求一次性交给内核。

> **⚠️ 纠正一个流传较广的说法**：阿里内核月报《MySQL · 引擎特性 · InnoDB 文件系统之 IO 系统和内存管理》（2016，基于 5.7.11）称"5.7 版本里合并操作已经被禁止了，全部改成了一个个 slot 进行读写"。
> **在 8.0.39 中合并是启用状态的**——`os_aio_simulated_handler` 明确调用 `handler.merge`（`os0file.cc`），`SimulatedAIOHandler::merge` 的实现完整存在（`os0file.cc`），最多合并 `OS_AIO_MERGE_N_CONSECUTIVE`(64) 个。合并时的临时大缓冲用 `ut::aligned_alloc(len, UNIV_PAGE_SIZE)` 分配。
> 这正是"外部材料必须回源码核实"的又一例证（月报的版本前提与 8.0.39 不同）。

**槽位耗尽与 IBUF 独立数组**：模拟 AIO 下槽位耗尽是真实风险（请求要排队等空槽）。所以 ibuf 的读走**独立的 `s_ibuf` 数组 + 独立线程**——否则"ibuf merge 的读把槽位占满，导致 ibuf merge 自己等不到读完成"就死锁了（详见[核心实现一](#核心实现一文件与表空间层fil)）。

#### 槽位数与并发上限

```cpp
bool os_aio_init(ulint n_readers, ulint n_writers) {
  /* Maximum number of pending aio operations allowed per segment */
  ulint limit = 8 * OS_AIO_N_PENDING_IOS_PER_THREAD;
  ...
  return (AIO::start(limit, n_readers, n_writers));
}
```

```
在途异步 I/O 上限 = innodb_write_io_threads × 8 × OS_AIO_N_PENDING_IOS_PER_THREAD
```

每个 segment 一个 `io_context_t`（`AIO::init_linux_native_aio`），segment 数 = `n_readers + n_writers + 1(ibuf)`。

> **实践意义**：默认 `innodb_write_io_threads=4`，在途写在默认配置下只有几十个。**现代 NVMe 与云盘的队列深度远大于此**，这就是为什么高并发写入场景必须调大 `innodb_write_io_threads`（8~16），否则设备根本喂不饱。

#### 提交：`os_aio_func`

`storage/innobase/os/os0file.cc`：

```cpp
dberr_t os_aio_func(IORequest &type, AIO_mode aio_mode, const char *name,
                    pfs_os_file_t file, void *buf, os_offset_t offset, ulint n,
                    bool read_only, fil_node_t *m1, void *m2) {
  ut_a(!type.is_log);
  ...
  if (aio_mode == AIO_mode::SYNC) {
    /* This is actually an ordinary synchronous read or write:
    no need to use an i/o-handler thread. */
    if (type.is_read) {
      return (os_file_read_func(type, name, file.m_file, buf, offset, n));
    }

    ut_ad(type.is_write);
    return (os_file_write_func(type, name, file.m_file, buf, offset, n));
  }

try_again:

  auto array = AIO::select_slot_array(type, read_only, aio_mode);

  auto slot = array->reserve_slot(type, m1, m2, file, name, buf, offset, n, e_block);

  if (type.is_read) {
    if (srv_use_native_aio) {
      ++os_n_file_reads;
#ifdef LINUX_NATIVE_AIO
      if (!array->linux_dispatch(slot)) {
        goto err_exit;
      }
#endif
    } else if (type.is_wake) {
      AIO::wake_simulated_handler_thread(
          AIO::get_segment_no_from_slot(array, slot));
    }
  } else if (type.is_write) {
    if (srv_use_native_aio) {
      ++os_n_file_writes;
#ifdef LINUX_NATIVE_AIO
      if (!array->linux_dispatch(slot)) {
        goto err_exit;
      }
#endif
    } else if (type.is_wake) {
      AIO::wake_simulated_handler_thread(
          AIO::get_segment_no_from_slot(array, slot));
    }
  }
  ...
  return (DB_SUCCESS);

#if defined LINUX_NATIVE_AIO || defined WIN_ASYNC_IO
err_exit:
#endif
  array->release_with_mutex(slot);
  if (os_file_handle_error(name, type.is_read ? "aio read" : "aio write")) {
    goto try_again;
  }
  return (DB_IO_ERROR);
}
```

**三个关键点**：

1. **`ut_a(!type.is_log)`**：redo 永远不许走 AIO（见[设计权衡](#二为什么-redo-用同步写数据页用异步-aio)）。
2. **`AIO_mode::SYNC` 完全绕过 libaio**，直接同步 `pread`/`pwrite`。所以"MySQL 用了异步 I/O"这句话只对**数据文件**成立。
3. **提交失败会重试**：`os_file_handle_error` 返回 true 就 `goto try_again`；只有不可恢复的错误才返回 `DB_IO_ERROR`。

真正的提交在 `AIO::linux_dispatch`（`os0file.cc`）——注意**一次只提交 1 个 iocb**，没有批量 submit：

```cpp
  ulint io_ctx_index;
  struct iocb *iocb = &slot->control;

  io_ctx_index = (slot->pos * m_n_segments) / m_slots.size;

  int ret = io_submit(m_aio_ctx[io_ctx_index], 1, &iocb);

  if (ret != 1) {
    errno = -ret;
  }

  return (ret == 1);
```

#### 收割：`LinuxAIOHandler::collect`

`os0file.cc`：

```cpp
void LinuxAIOHandler::collect {
  ...
  for (;;) {
    struct io_event *events;
    events = m_array->io_events(m_segment * m_n_slots);
    memset(events, 0, sizeof(*events) * m_n_slots);

    struct timespec timeout;
    timeout.tv_sec = 0;
    timeout.tv_nsec = OS_AIO_REAP_TIMEOUT;

    auto ret = io_getevents(io_ctx, 1, m_n_slots, events, &timeout);

    for (int i = 0; i < ret; ++i) {
      auto iocb = reinterpret_cast<struct iocb *>(events[i].obj);
      auto slot = reinterpret_cast<Slot *>(iocb->data);
      ...
      if (events[i].res > slot->len) {
        /* failure */
        slot->n_bytes = 0;
        slot->ret = events[i].res;
      } else {
        /* success */
        slot->n_bytes = events[i].res;
        slot->ret = 0;
      }
      m_array->release;
    }

    if (srv_shutdown_state.load == SRV_SHUTDOWN_EXIT_THREADS ||
        !buf_flush_page_cleaner_is_active || ret > 0) {
      break;
    }
    ...
  }
}
```

- **`io_getevents` 用的是带 timeout 的等待**（`OS_AIO_REAP_TIMEOUT`），不是无限阻塞——这样 io 线程才能响应 shutdown。
- **部分写的处理**：`poll` 里发现 `DB_FAIL`（部分 I/O）时调 `resubmit`，把 `ptr`/`offset`/`len` 前移后重新 `io_submit`。**resubmit 失败则 `ib::fatal`**。

---

### doublewrite

> **★ 完整剖析见专篇 [`dblwr.md`](dblwr.md)**——文件布局（**无文件头**的扁平页数组、批量区 + 512 个单页 SYNC 槽位、**奇偶文件功能切分**）、批量与单页两条路径的完整源码、崩溃恢复时如何用 dblwr 修页、加密帧为什么单独存在、`O_DIRECT_NO_FSYNC` 下五处 fsync 的取舍、`innodb_doublewrite` 的 6 个取值、参数与监控（`innodb_doublewrite_batch_size` 是 no-op）、源码里的 TODO。
>
> 本节只保留 I/O 全景所需的结论。

| 项 | 结论 |
|---|---|
| **为什么必须** | 防 **torn page**：16KB 页写不原子，**redo 修不了**——redo 是"页内偏移 + 内容"，假设页的其他部分是对的 |
| **文件** | `#ib_<page_size>_<id>.dblwr`（8.0.20+ 独立成文件），**无文件头**，扁平物理页数组 |
| **实例数** | `max(4, innodb_buffer_pool_instances × 2)`，多实例并发写 |
| **两分支** | **批量**（`!sync && != SINGLE_PAGE`）：攒批 → 同步写 dblwr → **异步 AIO** 写数据文件；**同步单页**：抢 SYNC 槽位 → 写 dblwr → fsync → **同步**写数据文件 → `fil_flush` |
| **★ 单页也走 dblwr** | 用文件尾部的 `SYNC_PAGE_FLUSH_SLOTS = 512`；只有三类跳过：`innodb_doublewrite=OFF` / 临时表空间 / 只读模式 |
| **dblwr 自身的 I/O** | **永远同步** `os_file_write_retry`（不走 AIO、不过 fil 层），失败**无限重试**（每 10s，永不放弃，不崩溃） |
| **是否 fsync** | `is_fsync_required`：`O_DIRECT` 与 `O_DIRECT_NO_FSYNC` **都不 fsync** |
| **★ 最大价值** | **N 次 fsync 摊成 1 次**——整批 AIO 完成后一次 `fil_flush_file_spaces`。云盘上 fsync 最贵，这直接决定刷脏吞吐 |
| **多文件的含义** | **不是"双池轮换"**，是**奇偶 id 功能切分**：奇数 id 承载 LRU 批量段 + 全部 SYNC 槽位，偶数 id 承载 flush list 批量段 |

---

### redo log 的 I/O（归位）与各文件 I/O 总览

> **★ redo 的写/刷完整实现见 [`redo_log.md`](redo_log.md)「redo 的 I/O 路径」**——六个专用后台线程（`log_writer` / `log_flusher` / `log_checkpointer` / 两个 notifier / `log_files_governor`）、`log_writer_write_buffer` 的环形写与 512 对齐、`log_flush_low` 的 O_DSYNC 分支、**redo 与数据文件的 I/O 方式差异表**、write-ahead（`innodb_log_write_ahead_size` 与 read-on-write）、fsync vs fdatasync。
>
> 本节只保留 I/O 全景需要的**横切对比**，以及不属于 redo 的部分。

#### 各文件 I/O 路径总览

InnoDB 与 server 层里不同文件的 I/O 走不同路径，对应不同的持久性要求：

| 文件 | 缓存方式 | 持久化控制 | 关键位置 |
|------|---------|-----------|---------|
| 数据文件（.ibd） | Buffered / O_DIRECT（flush_method） | 后台刷脏 + doublewrite | `os0file.cc` 的 O_DIRECT 判定 |
| redo log | Buffered + fsync（或 O_DSYNC） | `innodb_flush_log_at_trx_commit` | `log0write.cc` |
| doublewrite | 随 flush_method（O_DIRECT 时也绕过） | 每次刷脏前先写 dblwr | `buf0dblwr.cc` |
| sort 临时文件 | Buffered / O_DIRECT（独立开关） | 无需持久（DDL 失败即弃） | `ddl0ddl.cc` |
| binlog | Buffered + fsync | `sync_binlog` | `sql/binlog.cc` |

> **★ 一眼看出的规律**：只有 **InnoDB 数据文件（含 dblwr、clone）可能走 O_DIRECT 与 libaio**；**redo 永远只有同步 `pwrite` + fsync**（不开 O_DIRECT、不走 AIO）；binlog 与各类日志是 buffered + fsync。

**redo 在 I/O 全景中的位置（三条结论）**：

1. **redo 不走 AIO**——`os_aio_func` 里有 `ut_a(!type.is_log)`。
2. **redo 不开 O_DIRECT**——`OS_LOG_FILE` 不在 O_DIRECT 分支里（只有 `OS_DATA_FILE` / `OS_CLONE_DATA_FILE` / `OS_DBLWR_FILE` 在）。
3. **redo 的写是同步的、由 `log_writer` 单线程做**——顺序追加、量小、在关键路径上，AIO 的并发收益为零。

#### sort 临时文件的独立开关

DDL 排序的临时文件（`ibXXXXXX`）**不是数据页，不进 Buffer Pool**（没有 `space_id`/`page_no`），它唯一的缓存是 OS page cache。`innodb_disable_sort_file_cache`（默认 OFF）决定它是否用 O_DIRECT 绕过（`ddl0ddl.cc`）：

- OFF：sort file 走 page cache（反复读命中快），但大排序会污染 OS cache 挤热页；
- ON：O_DIRECT 绕过（避免污染，但排序变慢）。

详见 [`ddl.md`](ddl.md)。

---

### server 层 I/O 全景

> **这一层完全独立于 InnoDB 的 I/O 栈**：不用 `IORequest`、不走 `os_file_*`、不用 AIO、不开 O_DIRECT。它用的是 `mysys` 的一套原语与 `IO_CACHE`。
> 本节按"通用原语 → 具体子系统"组织：mysys 原语 / `IO_CACHE` / 临时文件 / binlog / 各类日志 / **内部临时表与排序文件** / 导入导出 / 其他引擎。

#### mysys 原语：裸系统调用 + 错误处理

| 函数 | 文件 | 职责 |
|------|------|------|
| `my_open` / `my_close` | `mysys/my_open.cc` | 包 `open`/`close`，失败时 `MyOsError`，成功后登记 fd→文件名 |
| `my_create` | `mysys/my_create.cc` | 包 `open(O_CREAT)`，带 `my_umask` |
| `my_read` / `my_write` | `mysys/my_read.cc` / `my_write.cc` | 包 `read`/`write`；★ **ENOSPC/EDQUOT + `MY_WAIT_IF_FULL` 时进入 `wait_for_free_space`**（线程切到 `waiting_for_disk_space` 阶段） |
| `my_pread` / `my_pwrite` | `mysys/my_pread.cc` | 定位读/写（不移动文件指针） |
| `my_sync` | `mysys/my_sync.cc` | **★ `fdatasync` 优先**（编译期 `#if HAVE_FDATASYNC`），否则 `fsync` |
| `my_mmap` / `my_msync` | `mysys/my_mmap.cc` | mmap 包装；`my_msync = msync + my_sync`（注释明写：msync 只刷到 fs cache，必须再 fsync） |
| `file_info::RegisterFilename` | `mysys/my_file.cc` | 维护 fd→文件名全局向量，供报错与 PFS 用 |

相对裸 syscall 多出来的：errno→my_errno 保存、统一报错、**fd 文件名登记**、EINTR 自动重试、**磁盘满时等待/重试**。

> **★ 两个易错点**：
> 1. **`my_*` 本身不带 PFS 埋点**。PFS 埋点在 `mysql_file_*` 这一层（`include/mysql/psi/mysql_file.h`），是"调用点按需包裹"：`PSI_FILE_CALL(start_file_open_wait)` → `my_open` → `PSI_FILE_CALL(end_file_open_wait_and_bind_to_descriptor)`。
> 2. **`my_sync` 用 `fdatasync` 是编译期决定的，与 `innodb_use_fdatasync` 完全无关**——后者只在 InnoDB 的 `os0file.cc` 里有效。

#### `IO_CACHE`：server 层的通用带缓冲 I/O

`mysys/mf_iocache.cc`。它是 binlog、日志、排序文件、LOAD DATA 的共同底座。

> **★ 完整剖析见专篇 [`../server/infra/io_cache.md`](../server/infra/io_cache.md)**——本篇只给概要。专篇覆盖：双缓冲指针的语义表、函数指针"穷人版多态"与 fast-path 去虚化、`init_io_cache_ext`/`reinit_io_cache`/`my_b_flush_io_cache`/`_my_b_write` 的逐段剖析、延迟创建文件、`READ_NET` 把网络流伪装成文件、透明加解密、以及 **`Basic_ostream` → `Truncatable_ostream` → `IO_CACHE_ostream` / `Binlog_encryption_ostream` 的装饰者链与 `Binlog_ofile` 门面**。

| 方面 | 说明 |
|------|------|
| **缓存大小** | 传 0 时回落到 `my_default_record_cache_size`（运行时 = `read_buff_size`，默认 128 KB）；最小 `min_cache = IO_SIZE*2`（8 KB，`IO_SIZE = 4096`），分配失败按 3/4 逐步缩小重试 |
| **写** | `_my_b_write`：填满 buffer → `my_b_flush_io_cache`；剩余 ≥ IO_SIZE 部分按 IO_SIZE 对齐直写 |
| **读** | `_my_b_read`/ `_my_b_read_r` / `_my_b_seq_read` |
| **★ 落盘与 spill** | `my_b_flush_io_cache`里 `if (info->file == -1) real_open_cached_file(info)`——**"缓冲写满"或"显式 flush"就是 spill 到磁盘临时文件的触发点** |
| **★ 大小上限的强制** | `_my_b_write` 里 `pos_in_file + buffer_length > end_of_file` 时返回 **EFBIG**——这就是 `max_binlog_cache_size` 的实现方式（把上限写进 `IO_CACHE::end_of_file`） |
| **加密** | IO_CACHE 自带 `m_encryptor`/`m_decryptor`（`mysql_encryption_file_write`） |
| **可选 fsync** | `info->disk_sync` 标志，只有 `SELECT INTO OUTFILE` 的 `select_into_disk_sync` 会打开 |
| **`SEQ_READ_APPEND`** | 循环双缓冲仍在，但 8.0.39 的 `sql/` 层**已无调用点**（binlog 改走 `IO_CACHE_ostream`），属遗留能力 |

#### 临时文件：创建即 unlink

唯一入口 `create_temp_file`（`mysys/mf_tempfile.cc`），三条实现分支：

| 平台/路径 | 做法 |
|-----------|------|
| Windows | `create_temp_file_uuid` → Crockford Base32 编码 UUID（**去掉 MAC 段**避免信息泄漏）→ `CreateFile(CREATE_NEW)` → 关句柄 → `my_open` → 若 `UNLINK_FILE` 则 `my_delete` |
| **Linux `O_TMPFILE` + `UNLINK_FILE`** | `open(dirname_buf, O_RDWR \| O_TMPFILE \| O_CLOEXEC, ...)`，文件名写成 `...fd=<fd>`——**目录里根本看不到名字，close 即消失** |
| 通用 POSIX | `mkstemp(to)` 之后**立刻 `unlink(to)`** |

> **"创建即 unlink"的三个好处**：① 目录里不可见（不污染 tmpdir）；② 所有 fd 关闭后空间自动回收；③ **进程被 `kill -9` 也能被内核回收**（不像依赖 atexit 清理的方案）。
>
> **文件名前缀**（识别 tmpdir 里那堆文件是谁产生的）：
> - `ML` → binlog cache 临时文件（`LOG_PREFIX = "ML"`，`binlog_ostream.cc`）
> - `MY` → filesort / Unique / hash join chunk（`TEMP_PREFIX = "MY"`，`sql_base.h`）
> - `mysql_temptable.` → TempTable 的 mmap 文件
>
> 目录取 `mysql_tmpdir`（支持**多目录 round-robin**）。`my_tmpfile` 在 8.0.39 **已不存在**（全仓库 0 命中）。

#### binlog

| 组件 | 位置 | 职责 |
|------|------|------|
| `MYSQL_BIN_LOG` | `sql/binlog.h`，全局实例 `mysql_bin_log` | binlog / relay log 主对象 |
| `MYSQL_BIN_LOG::Binlog_ofile` | `sql/binlog.cc` | 物理文件句柄封装（含加密 pipeline）：`open`(394) / `write`(508) / `flush`(550) / `sync`(551) / `flush_and_sync`(552) / `truncate`(542) |
| `IO_CACHE` | `mysys/mf_iocache.cc` | server 层通用带缓冲文件 I/O：`init_io_cache`(312) / `_my_b_write`(1238) / `_my_b_read`(424) / `my_b_flush_io_cache`(1430) / `end_io_cache`(1519) |
| `Binlog_cache_storage` | `sql/binlog_ostream.h`；`IO_CACHE_binlog_cache_storage` | 每 THD 的 binlog cache，超 `binlog_cache_size` 时 spill 到临时文件 |
| `MYSQL_BIN_LOG::ordered_commit` | `sql/binlog.cc` | 组提交三阶段（flush → sync → commit） |
| `MYSQL_BIN_LOG::flush_cache_to_file` | `sql/binlog.cc` | flush 阶段：IO_CACHE → OS |
| `MYSQL_BIN_LOG::sync_binlog_file` | `sql/binlog.cc` | sync 阶段：**按 `sync_binlog` 计数决定是否 fsync** |

**系统调用结论**：

- **不使用 O_DIRECT**。全仓库 O_DIRECT 只出现在 InnoDB 的 `os0file.cc` 与 `fil_fusionio_enable_atomic_write`；`sql/` + `mysys/` 里对 binlog 没有任何 O_DIRECT 打开。
- **buffered write（`write`）+ fsync（`my_sync`）**。
- `sync_binlog=N` 的作用点在 `binlog.cc`（`sync_period && ++sync_counter >= sync_period`）与 （决定是否引入 group-commit 延迟）。`=0` 只 write 不 fsync；`=1` 每个事务组都 fsync。
- 创建新 binlog 文件后还会 `my_sync_dir_by_file` 同步**目录项**（否则机器崩溃后新 binlog 文件可能"消失"）。

#### slow log / general log

| 组件 | 位置 |
|------|------|
| `Query_logger::slow_log_write` | `sql/log.cc` |
| `Query_logger::general_log_write` | `sql/log.cc` |
| `Log_to_file_event_handler::log_slow` | `sql/log.cc`（落文件） |
| `Log_to_csv_event_handler::log_slow` | `sql/log.cc`（写 `mysql.slow_log` CSV 表） |
| `File_query_log` | `sql/log.cc`（封装 IO_CACHE + 表） |
| `log_slow_statement` | `sql/log.cc` |

#### error log（组件化流水线）

`filter → write → sink` 三段式：

| sink | 位置 | 输出目标 |
|------|------|---------|
| `log_sink_trad` | `sql/server_component/log_sink_trad.cc` | 传统文件 |
| `log_sink_buffer` | `sql/server_component/log_sink_buffer.cc` | 内存缓冲（启动早期用） |
| `log_sink_perfschema` | `sql/server_component/log_sink_perfschema.cc` | `performance_schema.error_log` 表 |

入口 `log_builtins_init`（`sql/server_component/log_builtins.cc`），刷栈 `log_builtins_error_stack_flush`。

#### ★ 内部临时表与排序文件（最容易产生"意外 I/O"的地方）

SQL 层的内部临时表是**三级降级**：

```
① TempTable / MEMORY（纯内存）
        ↓ RAM 超 temptable_max_ram
② tmpdir 里的匿名 mmap 文件（TempTable 的 MMAP_FILE）
        ↓ MMAP 也超 temptable_max_mmap → throw RECORD_FILE_FULL
③ 真正的 InnoDB 磁盘临时表
```

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `internal_tmp_mem_storage_engine` | **`TempTable`** | 内存临时表引擎（枚举 `{"MEMORY","TempTable"}`） |
| `temptable_max_ram` | **1 GiB** | RAM 上限 |
| `temptable_max_mmap` | **1 GiB** | mmap 文件上限 |
| `tmp_table_size` | **16 MB** | 内存临时表达到此大小就转磁盘 |
| `max_heap_table_size` | **16 MB** | MEMORY 引擎增长上限 |
| `big_tables` | **false** | ★ 置 ON 且无 `SQL_SMALL_RESULT` → **直接建磁盘临时表，完全跳过内存阶段** |

**运行时降级**由 `create_ondisk_from_heap`（`sql_tmp_table.cc`）完成：只有 `HA_ERR_RECORD_FILE_FULL` 且引擎是 `heap_hton`/`temptable_hton` 才转换，目标引擎在 `sql_tmp_table.cc` **硬编码为 `innodb_hton`**，逐行把内存表搬进 InnoDB 表。

> **★ 更正一条常见误解**：`internal_tmp_disk_storage_engine` 在 8.0.39 **已被移除**（`sys_vars.cc` 里没有定义，只剩 `sql_tmp_table.cc` 的注释残骸，相关测试已被 `disabled.def` 禁用）。磁盘内部临时表引擎是**写死 InnoDB** 的，不是可配置的。

**TempTable 的"落盘"其实是 mmap**（`storage/temptable/include/temptable/memutils.h`）：

```cpp
  File f = create_temp_file(file_path, mysql_tmpdir, "mysql_temptable.", mode,
                            UNLINK_FILE, MYF(MY_WME));
  if (my_fallocator(f, bytes, 0xa, MYF(MY_WME)) != 0 || ...) {...}
  void *ptr = my_mmap(nullptr, bytes, PROT_READ | PROT_WRITE, MAP_SHARED, f, 0);
  my_close(f, MYF(MY_WME));        // mmap 后立刻关 fd（文件已因 UNLINK_FILE 不可见）
```

即：**tmpdir 里的匿名 mmap 文件，零 fd、close 即回收**，与"真正降级到 InnoDB"是两回事。

**filesort 的临时文件**（`sql/filesort.cc`）：

| 函数 | 位置 | 职责 |
|------|------|------|
| `filesort` |  | 排序总控 |
| `read_all_rows` |  | 读源数据到 `Filesort_buffer`（纯内存） |
| **`write_keys`** |  | 缓冲满 → `open_cached_file(chunk_file, mysql_tmpdir, TEMP_PREFIX, DISK_BUFFER_SIZE, MY_WME)` 刷 chunk 到 tmpdir |
| `merge_many_buff` | `sql/merge_many_buff.h` | chunk > `MERGEBUFF2`(15) 时多轮归并，每轮新建临时文件并 `reinit_io_cache` 交替读写 |
| `copy_bytes` |  | 归并用 **`mysql_file_pread`** 直读 + `my_b_write` |

常量：`TEMP_PREFIX = "MY"`、`IO_SIZE = 4096`、`DISK_BUFFER_SIZE = IO_SIZE*16 = 64 KB`、`READ_RECORD_BUFFER = IO_SIZE*8`。

**同样的 `open_cached_file` 模式还用在**：`Unique`（去重，`sql/uniques.cc`）、**hash join 的 chunk**（`sql/iterators/hash_join_chunk.cc`）、多表 UPDATE 的 rowid 缓冲（`sql/sql_update.cc`）。

#### LOAD DATA / SELECT INTO OUTFILE

| 方向 | 入口 | I/O |
|------|------|-----|
| **导入** | `Sql_cmd_load_table::execute_inner`（`sql/sql_load.cc`；★ 旧名 `mysql_load` 已不存在） | `mysql_file_open(O_RDONLY)` → `READ_INFO` 用 `init_io_cache(READ_CACHE / READ_FIFO / READ_NET)`；FIFO 有专门分支；`secure_file_priv` 检查在 `is_secure_file_path` |
| **导出** | `create_file`（`sql/query_result.cc`）+ `Query_result_export` / `Query_result_dump`（★ 旧名 `select_export` / `select_dump` 已不存在） | `mysql_file_create(..., O_EXCL)` → `init_io_cache(cache, file, select_into_buffer_size, WRITE_CACHE, ...)`；**可选 fsync**：`select_into_disk_sync` 打开 `cache->disk_sync`，每次 flush 后 `mysql_file_sync` |

#### 其他存储引擎的文件 I/O

| 引擎 | 文件 | I/O 方式 |
|------|------|---------|
| **MyISAM** | `.MYD` + `.MYI` | 默认 `mysql_file_pread/pwrite`（buffered）；开 `myisam_use_mmap` 时走 **mmap**（`mi_mmap_pread/pwrite`）。★ **key cache（`mysys/mf_keycache.cc`）是独立的 I/O 层**，缓存 `.MYI` 索引块 —— 与 OS page cache 叠加，典型**双缓存** |
| **CSV** | `.CSV` + `.CSM` + `.CSN` | `O_APPEND` + `mysql_file_write`，**无 fsync** |
| **ARCHIVE** | `.ARZ` / `.ARN` / `.ARM` | `azopen`/`azread`/`azwrite`，zlib 流式压缩 + buffered |
| **BLACKHOLE / HEAP** | 无 | 无文件 I/O |

> **★ 一个反直觉的事实**：8.0 的 mysql 系统表**并非全部 InnoDB**——`mysql.general_log` 与 `mysql.slow_log` 仍是 **CSV 引擎**（`scripts/mysql_system_tables.sql:357/360`）。所以 `log_output=TABLE` 时，日志写入的是 CSV 表：无 fsync、O_APPEND 追加、且有表级锁竞争。**高负载下开 TABLE 输出是 I/O 与锁的双重负担。**

#### 其他产生文件 I/O 的 server 层组件

| 组件 | 位置 | I/O |
|------|------|-----|
| **SDI 文件** | `sql/dd/impl/sdi_file.cc` `write_sdi_file` | `mysql_file_create` + `mysql_file_write`，**无 fsync**；只在 `innodb_read_only` / `IMPORT TABLESPACE` 场景写 `.sdi` |
| **keyring** | `plugin/keyring/file_io.cc` `File_io::sync` | buffered + `my_sync`（keyring 文件 + `.backup`） |
| **tc_log（XA 2PC）** | `sql/tc_log.cc` | **mmap + `my_msync`（msync + fsync）**——server 层唯一严谨的 mmap 持久化用法 |
| **persisted variables / auto.cnf** | `sql/persisted_variable.cc` / `mysqld.cc` | buffered write + `my_sync` |
| **PFS file 埋点** | `include/mysql/psi/mysql_file.h` | 每次被 instrument 的 I/O 前后各一次 `PSI_FILE_CALL`；`file_summary_by_instance` 按文件实例聚合，**文件多时内存与哈希开销线性增长** |

#### server 层 vs InnoDB 层：一张对照表

| 维度 | server 层（`sql/`+`mysys/`+非 InnoDB 引擎） | InnoDB |
|------|-------------------------------------------|--------|
| **O_DIRECT** | **无**（全仓库仅注释与 `handler.h` 的 clone 标志位提及） | 有（`innodb_flush_method=O_DIRECT`） |
| **libaio / `io_submit`** | **无** | 有 |
| **O_SYNC** | **无** | 有（`O_DSYNC`） |
| **mmap** | **有 3 处**：`tc_log`（XA）、MyISAM `myisam_use_mmap`、TempTable | 无 |
| **fsync 语义** | `my_sync` = **`fdatasync` 优先（编译期 `#if`）**，与 `innodb_use_fdatasync` **无关** | `os_file_fsync`，受 `innodb_use_fdatasync` 控制 |
| **★ 目录项 fsync** | **无**。`my_sync_dir` / `my_sync_dir_by_file` **在 8.0.39 已不存在**，只剩 `sql/binlog.cc` 的 TODO 注释——**binlog/relay log 的 dentry 没有 fsync** | 有（建/删文件后 fsync 目录） |
| **双缓存** | **明显**：OS page cache + `IO_CACHE` + MyISAM key cache + BP 叠加 | 尽量单层（BP + O_DIRECT） |
| **异步** | 无，全部同步 write | libaio / 同步 pwrite 双路径 |
| **同步粒度** | 按 `IO_CACHE` buffer（8 KB~64 KB）或"每行一次 flush"（general log） | 按页（16 KB）+ redo 组提交 |
| **临时文件** | `create_temp_file`：**创建即 unlink**，close 自动回收 | 临时表空间（`innodb_temp_data_file_path`） |
| **崩溃恢复** | 无 redo；binlog 靠 **crash-safe index 文件**弥补 | redo + doublewrite |
| **并发控制** | 组提交（`ordered_commit`） | AIO 槽位 + 组提交 |
| **持久性变量** | `sync_binlog` / `sync_relay_log` / `select_into_disk_sync` | `innodb_flush_log_at_trx_commit` / `innodb_flush_method` |

---

### 其他文件 I/O

| 子系统 | 入口 | ★ 关键事实 |
|--------|------|-----------|
| **DDL log** | `Log_DDL::write_delete_space_log`(961) / `write_rename_space_log`(1065) / `write_drop_log`(1258) / `write_free_tree_log`(853) / `post_ddl`(1906) / `recover`(1943)，`log/log0ddl.cc` | **不是独立文件**！记录在 `mysql.innodb_ddl_log` 这张 InnoDB 表 + redo。这是"DDL 不记 undo，用 DDL log 记账 + 提交后 replay"的实现方式，详见 `undo_log.md` |
| **clone** | `Clone_Handle::open_file`(2226) / `apply_data`(1516) / `modify_and_write`(1333) / `sparse_file_write`(1266)；`Clone_Snapshot::build_file`(622) | 用 `OS_CLONE_DATA_FILE` / `OS_CLONE_LOG_FILE`，**clone 的数据文件也开 O_DIRECT** |
| **keyring（旧插件）** | `keyring::File_io::open/read/write/sync`，`plugin/keyring/file_io.cc/82/93/127` | buffered file IO |
| **keyring（组件）** | `File_writer` / `File_reader`，`components/keyrings/common/data_file/{writer,reader}.h` | JSON 格式密钥文件 |
| **audit log** | `sql/sql_audit.cc`（`mysql_audit_notify`） | ★ **社区版没有文件型 audit log 插件**，只有示例插件 `plugin/audit_null`（写 error log）。文件审计是企业版特性 |
| **buffer pool dump/load** | `buf_dump`（`buf0dump.cc`） / `buf_load` | 关机/开机读写 `ib_buffer_dump`；`buf_load` 受 `innodb_io_capacity` 限速 |
| **归档** | `log_archiver_thread`（`arch0arch.cc`）/ `page_archiver_thread`（`arch0page.cc`）；`Arch_File_Ctx::open/read/write` | redo 归档与页跟踪归档 |
| **表空间 import/export** | `row_quiesce_table_start`(933) → `row_quiesce_write_cfg`(714) → `row_quiesce_write_cfp`(850)；`row_quiesce_table_complete`(995) | `FLUSH TABLE ... FOR EXPORT` 生成 `.cfg` / `.cfp`，结束时删除 |

---

### I/O 线程模型

| 线程 | 入口 | 数量由谁决定 |
|------|------|------------|
| io handler（读） | `io_handler_thread`，`os0file.cc`（数组 `s_reads`） | `innodb_read_io_threads`（默认 4，READONLY） |
| io handler（写） | 同上（数组 `s_writes`） | `innodb_write_io_threads`（默认 4，READONLY） |
| io handler（ibuf） | 同上（数组 `s_ibuf`） | 固定 1（只读模式不启动） |
| page cleaner 协调 | `buf_flush_page_coordinator_thread`，`buf0flu.cc` | 固定 1 |
| page cleaner worker | `buf_flush_page_cleaner_thread`，`buf0flu.cc` | `innodb_page_cleaners`（会被 `innodb_buffer_pool_instances` 截断） |
| log_writer / log_flusher / log_checkpointer / log_write_notifier / log_flush_notifier / log_files_governor | 见 [`redo_log.md`](redo_log.md)「写路径：五个专用后台线程」 | 各 1（`innodb_log_writer_threads=OFF` 时由用户线程代做 write） |
| **ibuf merge** | **无专用线程** | 由 `srv_master_thread` 调 `ibuf_merge_in_background`（`srv0srv.cc/2411/2510`） |
| master | `srv_master_thread`，`srv0start.cc` | 固定 1 |
| purge coordinator / worker | `srv0start.cc/2457` | `innodb_purge_threads` |
| recovery writer | `recv_writer_thread`，`log0recv.cc` | 恢复期临时 |
| BP resize / dump | `buf_resize_thread` / `buf_dump_thread` | 各 1 |
| 并行扫描 | `row0pread.cc` | `innodb_parallel_read_threads` |

> **ibuf merge 没有自己的线程**是个容易记错的点——它寄生在 master 线程里。

---

### 压缩、加密与 punch hole

#### 三种"压缩"在不同层

| 类型 | 层次 | I/O 影响 |
|------|------|---------|
| **表压缩**（`ROW_FORMAT=COMPRESSED`，zip） | **页层**（`page0zip.cc` 的 `page_zip_compress`/`page_zip_decompress`） | 物理页可 < 16K（8K/4K），以压缩后尺寸读写；`buf_page_t::zip.data` + `buf_buddy_alloc` |
| **page compression** | **I/O 层** | 写入前 `os_file_compress_page`，写后 `os_file_punch_hole` 把多余空间还给文件系统 |
| redo 压缩 | 无 | — |

#### punch hole（page compression）

`os_file_punch_hole`（`os0file.cc`）→ `os_file_punch_hole_posix`：

```cpp
fallocate(fh, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, off, len);
```

- 触发条件在 `Fil_shard::do_io`：写 + 非压缩表 + **非 page 0** + 文件系统支持 + `file->punch_hole`。
- 支持性探测：`os_is_sparse_file_supported`（`os0file.cc`）——方法是**真的打一个洞试试**。
- 优化：压缩后尺寸与原始相同则跳过打洞（除非设了 `DISABLE_PUNCH_HOLE_OPTIMISATION`）。
- 文件扩展用 `fallocate(FALLOC_FL_ZERO_RANGE)`：`os_file_set_size_fast`（`os0file.cc`）。

#### 加密在哪一层，会不会改变 I/O 大小

| 对象 | 层 | 函数 |
|------|-----|------|
| 数据页 | **I/O 层**：写前 `os_file_encrypt_page`（`os0file.cc`）→ `Encryption::encrypt_low`（`os0enc.cc`）；读后 `os_file_io_complete`（`os0file.cc`）解密 | 见下 |
| redo | **log I/O 层**：`Encryption::encrypt_log_block`（`os0enc.cc`）/ `encrypt_log`(906)；由 `Log_file_handle::prepare_io_request` 根据 `innodb_redo_log_encrypt` 注入 | 见下 |

> **★ 关键结论：加密不改变 I/O 大小。** AES 是分组加密，密文长度 = 明文长度（对齐到块）。所以**加密不会导致额外的读写放大**，代价是 CPU 与（keyring 轮换时）额外的 `file::Block` 拷贝。

---

### 读路径、预读与并行扫描（归位）

> **读路径与预读是 Buffer Pool 的职责，并行扫描已有专篇**。本节只给归位表，避免三处重复。

| 主题 | 归位 | 关键结论 |
|---|---|---|
| **一次缺页的完整流程** | [`buffer_pool.md`](buffer_pool.md) | `fil_io(sync=true)` → **调用线程自己的 `pread`**，不走异步 AIO |
| **并发请求同一页：只发一次 I/O** | 同上 | `buf_page_init_for_read` 的"插入 + 查重"在同一个 hash_lock X 锁临界区 |
| **`buf_wait_for_read` 没有任何 os_event** | 同上 | **"能拿到 S 锁"本身就是唤醒** |
| **`buf_page_io_complete` 十步后处理 + 两层解压** | 同上 | os 层解密 / 解透明页压缩；buf 层解表压缩 / checksum / ibuf merge |
| **读失败：重试 100 次才 fatal** | 同上 | `BUF_PAGE_READ_MAX_RETRIES = 100`；`srv_force_recovery >= 1` 会吞掉坏页 |
| **change buffer（ibuf）** | [`ibuf.md`](../index/ibuf.md) | 用延迟写换随机读；**云盘上收益更大**；"写完立刻读"的负载该关 |
| **AHI（自适应哈希索引）** | [`ahi.md`](../index/ahi.md) | 纯内存、零 I/O；只在叶子页；**每组只存一条**；分片按 (space_id, index_id) 路由故单索引热点仍争用 |
| **两种预读（线性 / 随机）** | [`buffer_pool.md`](buffer_pool.md) | 另有 ibuf merge 批量读与恢复期区域预读；**SSD 上收益可变负** |
| **并行扫描 `Parallel_reader`** | [`parallel_scan.md`](parallel_scan.md) | 只服务 DDL / 分区 / 直方图；**社区版无 SQL 层并行查询** |

#### 一处必须留在 I/O 全景里的结论：真异步只读预读

**只有这三条真正走 `sync=false`（真 AIO）**（判定链见 [`fil.md`](fil.md)「AIO 模式的三选一」）：

| 调用者 | sync | 说明 |
|---|---|---|
| `buf_read_ahead_random` / `_linear` | **false** | 预读；用 `DO_NOT_WAKE` 攒批，最后统一唤醒（`buf0rea.cc`） |
| `buf_read_ibuf_merge_pages` | 仅最后一页 true | `AIO_mode::IBUF`（单独数组，防槽位耗尽死锁） |
| `buf_read_recv_pages` | 仅最后一页 true | 崩溃恢复 |

此外，**有些页会被强制降级为同步**：`ibuf_bitmap_page` 与 `trx_sys_hdr_page`——它们在 latching order 里位置太低，若交给异步 io-handler 完成后处理（要拿 ibuf 树上的 latch）可能死锁。

> **⭐ 单页缺页读从不做页合并**：一次 `buf_read_page_low` 永远只发一个页。并发来自"N 个用户线程各缺一页 → N 个并发 `pread`"（内核层面天然并行）。

---

### 刷脏批次与文件管理（归位）

> 前面几章讲"一页怎么读写"。**刷脏批次的组织属于 Buffer Pool 的职责，文件管理属于 fil 层**——两者都已独立成篇。本节只给归位表 + 一处与 AIO 直接相关的结论，避免三处重复。

| 主题 | 归位 | 关键结论 |
|---|---|---|
| **刷脏批次的 single-flight**（`buf_flush_do_batch` 返回 false = "没轮到我"） | [`buffer_pool.md`](buffer_pool.md) | 同类批次互斥；`buf_flush_end` 末尾 `dblwr::force_flush` |
| **批次内部扫描与 hazard pointer**（`flush_hp` / `lru_hp`、LRU 的 `mutex_enter_nowait`） | 同上 | 跳过一页不影响扫描；`nowait` 是为避免持 page latch 时死锁 |
| **`buf_flush_page` 的锁契约**（返回值决定 block_mutex 归谁释放） | 同上 | LIST 刷的"延迟 SX 加锁"是唯一可能阻塞点 |
| **跳过会不会漏刷（三层保证）** | 同上 | LIST 路径不看 `buf_fix_count`，不会饥饿 |
| **邻接刷盘 `buf_flush_try_neighbors`** | 同上 | 机械盘设计，**云盘/SSD 设 0** |
| **全量刷脏 `buf_flush_sync_all_buf_pools`** | 同上 | `min_n=ULINT_MAX` 不做均摊；`buf_flush_fsync` 在循环外只调一次 |
| **用户线程单页刷为什么慢（四条叠加）** | 同上 | 对应 `Innodb_buffer_pool_wait_free` |
| **文件扩展 `Fil_shard::space_extend`** | [`fil.md`](fil.md) | `is_being_extended` 互斥；记 redo 但**故意不** `log_write_up_to` |
| **fsync 并发去重 `space_flush`（合并 fsync）** | [`fil.md`](fil.md) | `sync_event` + 四计数器；`O_DIRECT_NO_FSYNC` 下只在文件变大时刷 |
| **doublewrite 的完整实现** | [`dblwr.md`](dblwr.md) | 见[核心实现四](#核心实现四doublewrite) |

#### AIO 槽位耗尽会直接阻塞刷脏

完整调用链：

```
buf_flush_write_block_low → dblwr::write → fil_io(NORMAL) → os_aio_func → AIO::reserve_slot  ← 阻塞
```

`reserve_slot` 在槽位满时 `os_event_wait(m_not_full)`，于是 page cleaner 批次卡住、`n_flush[]` 不归零、`no_flush[]` 不 set；用户线程走 `buf_flush_single_page_from_LRU` 同样可能卡在这。上层表现为 `buf_LRU_get_free_block` 迭代次数飙升，最终打 **"Difficult to find free blocks in the buffer pool"** 警告。

> **这解释了为什么 `innodb_io_capacity` 调得过高（超过磁盘真实能力）反而会让刷脏和前台查询一起抖动**：投递速度超过收割速度 → 槽位打满 → 阻塞。
>
> **槽位的两种选择策略**（native AIO 跨 segment 轮询分摊 vs 模拟 AIO 按偏移映射便于合并）见[核心实现三](#核心实现三aio-子系统)。

---

### 特殊 I/O 场景

#### `buf_flush_sync_all_buf_pools`：同步刷所有实例全部脏页

`buf0flu.cc`。调用点：

- `srv0start.cc`（shutdown / 启动后）
- `log0chkp.cc`（checkpoint 前强刷）
- `ha_innodb.cc/5536/20813/21731`（如 `innodb_buffer_pool_dump_now`、`innodb_buffer_pool_load_abort`）

最后一步是 `buf_flush_fsync`，注释很直白：

```cpp
  /* All pages have been written to disk, but we need to make fsync for files
  to which the writes have been made. */
  buf_flush_fsync;
```

#### `FLUSH TABLES` 不等于"刷脏页"

server 层 `close_cached_tables`（`sql/sql_base.cc`）的语义是**关闭表缓存 + 让存储引擎把表刷到磁盘**（`ha_flush` / `closefrm`），**不是** InnoDB 全量刷脏。带 `WITH READ LOCK` 走 `flush_tables_with_read_lock`（`sql/sql_reload.cc`）。

> 想要"把所有脏页刷下去"，看 `buf_flush_sync_all_buf_pools` 的调用点，而不是 `FLUSH TABLES`。

#### TRUNCATE / DROP 的文件删除是同步的

`os_file_delete_func`（`os0file.cc`）= `unlink`，`fil_delete_tablespace`（`fil0fil.cc`）同步调用它。

> **★ 没有异步/AIO 后台清理机制。** 大表 DROP 时 `unlink` 大文件会让文件系统忙一阵（ext4 尤甚，xfs 好得多）。唯一的"延迟"部分是 **buffer pool 中该表空间页的失效**（`buf_LRU_remove_pages` / `fil_space_t::set_deleted` + `bump_version`），那是内存清理，不是文件删除。

#### 其他

| 场景 | 入口 | 说明 |
|------|------|------|
| 表空间 export/import | `row_quiesce_table_start` / `row_quiesce_write_cfg` | 生成 `.cfg`（元数据）与 `.cfp`（加密传输密钥） |
| `ALTER TABLE ... IMPORT TABLESPACE` | `ha_innobase::discard_or_import_tablespace` + `Fil_shard::space_delete` 等 | 会删 `.cfg`/`.cfp` |
| clone | `Clone_Handle::apply` / `extend_and_flush_files` | 见[核心实现七](#核心实现七其他文件-io) |
| BP dump/load | `buf_dump` / `buf_load` | 关机导出 page id 列表，开机预读（受 io_capacity 限速） |

---

#### ★ 云盘上的 MySQL I/O

> 云存储（EBS / 云盘 / CBS / ESSD）本身的定义、卷类型与选型见 [`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)。本节只讲**云盘对这套 I/O 栈意味着什么**。

#### MySQL 对云盘完全无感知（有证据）

对 `storage/innobase` 检索 `EBS`、`cloud`、`network storage`、`SAN`、`iSCSI`、`SSD`、`NVMe`：

```
结果：0 matches
```

对整个 MySQL 源码树检索 `cloud`：唯一命中是 NDB Cluster `thrman.cpp:2132` 的一条注释（与 I/O 无关）。

**契约面只有 6 个系统调用**（`open` / `pread` / `pwrite` / `io_submit`+`io_getevents` / `fsync`+`fdatasync` / `fcntl(O_DIRECT)`），也没有任何与云存储相关的系统变量。唯一的"设备自适应"是 `innodb_dedicated_server`，但它自适应的是**内存**，不是存储。

#### WAL 屏障：刷脏页前必须先刷 redo

`storage/innobase/buf/buf0flu.cc`：

```cpp
  /* Force the log to the disk before writing the modified block */
  if (!srv_read_only_mode) {
    const lsn_t flush_to_lsn = bpage->get_newest_lsn;

    if (log_sys->flushed_to_disk_lsn.load < flush_to_lsn) {
      Wait_stats wait_stats;

      wait_stats = log_write_up_to(*log_sys, flush_to_lsn, true);

      MONITOR_INC_WAIT_STATS_EX(MONITOR_ON_LOG_, _PAGE_WRITTEN, wait_stats);
    }
  }
```

- 刷脏页**之前**必须保证覆盖该页的 redo 已落盘，否则崩溃时页是新的、redo 没有 → 物理损坏。
- **先比较 `flushed_to_disk_lsn` 再调用**是 8.0 的性能优化（源码注释明说：避免大量无谓进入 log 线程的原子计数自旋）。
- 随后 `buf_flush_init_for_writing` 把 `newest_lsn` 写进页头 `FIL_PAGE_LSN`（崩溃恢复判断"页是否比 checkpoint 新"的唯一依据）并重算 checksum。
- **`dblwr::write` 是唯一出口**，没有 bypass。

> **云盘相关性**：这个 `fsync` 是每次刷脏批次的固定成本。云盘 fsync 比本地盘贵一个数量级，所以 **redo 与数据文件应放同一块卷**（跨卷快照不一致 + fsync 打两处）。

#### 五条隐含假设（云盘上每条都更脆弱）

| # | 假设 | 源码落点 | 云盘上的风险 |
|---|------|---------|------------|
| 1 | 512B 扇区写是原子的 | `OS_FILE_LOG_BLOCK_SIZE = 512` | 云盘普遍保证；但 **16KB 页写仍不原子** → 必须 doublewrite |
| 2 | `fsync` 返回 = 已持久化 | 全部持久性保证的基础 | 依赖云盘/虚拟化层 flush 语义完整 |
| 3 | **`O_DIRECT` 的 write 返回 = 已持久化** | `Double_write::is_fsync_required` 在 O_DIRECT 下跳过 fsync | **最高风险**，见下 |
| 4 | 设备能力 = `innodb_io_capacity` | 默认 **200**，无自动探测 | 与实际云盘差 1~2 个数量级 → 刷脏跟不上 |
| 5 | I/O 要么成功要么报错 | `os_file_fsync_posix` 遇 EIO 即 fatal | 云盘更常 **hang 而非报错** → 整实例挂起，无超时保护 |

#### ★ 最危险的一条：`innodb_dedicated_server` 的静默改写

`ha_innodb.cc`：

```cpp
  if (srv_dedicated_server && sysvar_source_svc != nullptr &&
      os_is_o_direct_supported) {
    ...
      if (source == COMPILED) {
        innodb_flush_method = static_cast<ulong>(SRV_UNIX_O_DIRECT_NO_FSYNC);
      }
    ...
  }
```

开了 `innodb_dedicated_server=ON` 且未显式指定 `innodb_flush_method` 时，MySQL 会**静默改成 `O_DIRECT_NO_FSYNC`**——即 dblwr 写后**不再 fsync**（`is_fsync_required` 返回 false）。本地 NVMe 通常没问题；云盘上这等于把数据安全完全押在"设备的 O_DIRECT 写返回即持久化"上。

> **结论：云上必须显式设置 `innodb_flush_method=O_DIRECT`。**

#### 故障模式的差异（最容易被忽略）

| | 本地盘 | 云盘 |
|---|--------|------|
| 典型故障 | 返回 `EIO` | **长时间 hang** |
| MySQL 行为 | `ib::fatal` 崩溃 → HA 接管（快速、明确） | 卡在 `pwrite` / `fsync` / `io_getevents`（**这些调用都没有超时**）→ 整实例挂起，HA 心跳可能一起卡住 |

#### 云上三件套

| 参数 | 建议 | 理由 |
|------|------|------|
| `innodb_flush_method` | **显式 `O_DIRECT`** | 不被 `innodb_dedicated_server` 静默改成 NO_FSYNC |
| `innodb_io_capacity` | 按卷稳态 IOPS × 0.7~0.8 | 默认 200 是机械盘数字，会让云盘 95% 能力闲置、脏页锯齿式堆积 |
| `innodb_flush_neighbors` | **0** | 邻接刷盘是"把随机写顺序化"的**机械盘设计**，云盘/SSD 上收益为零却带来写放大与窗口扫描开销 |
| `innodb_write_io_threads` | 4 → 8~16 | 云盘队列深，默认 4 个线程发不出足够并发请求 |

> **邻接刷盘的源码与三档取值见 [`buffer_pool.md`](buffer_pool.md)「邻接刷盘」**；这里只记结论：**云盘/SSD 上必须设 0**。

---

## 相关的系统变量

### I/O 方式与缓冲

| 变量 | 默认 | 作用域 | 说明 |
|------|------|--------|------|
| `innodb_flush_method` | `fsync` | Global，**READONLY** | 数据文件的 I/O 方式；六个取值见[核心实现二](#核心实现二os_file-与-io-方式o_direct--o_sync--fsync)。**云上显式设 `O_DIRECT`** |
| `innodb_use_fdatasync` | `OFF` | Global，动态 | `fsync` → `fdatasync`，省元数据刷写 |
| `innodb_disable_sort_file_cache` | `OFF` | Global，动态 | DDL sort 临时文件是否 O_DIRECT（避免污染 OS cache） |
| `innodb_dedicated_server` | `OFF` | Global，READONLY | 按内存推算 BP / 日志文件大小；★ 未显式指定 flush_method 时**静默改为 `O_DIRECT_NO_FSYNC`** |
| `innodb_use_native_aio` | `true` | Global，READONLY | 8.0.39 仍保留 simulated AIO，但应当用 native（libaio） |
| `innodb_doublewrite` | `ON` | Global | 半页写防线。**除非设备保证 16KB 原子写，否则不要关** |
| `innodb_doublewrite_dir` | 空（数据目录） | Global，READONLY | 8.0 起 dblwr 独立成文件，可迁移到更快设备 |
| `innodb_doublewrite_pages` | = `innodb_write_io_threads` | Global，READONLY | 每个 dblwr 实例的页数 |

### 并发与能力

| 变量 | 默认 | 作用域 | 说明 |
|------|------|--------|------|
| `innodb_read_io_threads` | `4` | Global，READONLY | AIO 读 segment 数；在途读上限 = 它 × 8 × `OS_AIO_N_PENDING_IOS_PER_THREAD` |
| `innodb_write_io_threads` | `4` | Global，READONLY | 同上（写）。**直接决定在途写并发上限**，SSD/云盘建议 8~16 |
| `innodb_io_capacity` | **200** | Global，动态 | 后台刷脏的目标 IOPS。**默认 200 是机械盘数字，现代设备上必须调高** |
| `innodb_io_capacity_max` | `max(2×io_capacity, 2000)` | Global，动态 | 自适应刷脏上限 |
| `innodb_flush_neighbors` | `1` | Global，动态 | 邻接刷盘。**SSD/云盘设 0** |
| `innodb_page_cleaners` | = BP 实例数 | Global，READONLY | page cleaner worker 数 |
| `innodb_parallel_read_threads` | `4` | Global/Session，动态 | **InnoDB 内部**并行扫描线程数（只服务 DDL/分区/直方图） |
| `innodb_read_ahead_threshold` | `56` | Global，动态 | 线性预读阈值 |
| `innodb_random_read_ahead` | `OFF` | Global，动态 | 随机预读开关 |
| `innodb_open_files` | — | Global，READONLY | 文件句柄上限，超出时 `close_files_in_LRU` |

### 持久性

| 变量 | 默认 | 说明 |
|------|------|------|
| `innodb_flush_log_at_trx_commit` | `1` | `1`=每次提交 fsync redo；`2`=只 write 不 fsync（OS 崩丢数据）；`0`=完全后台（丢约 1s） |
| `sync_binlog` | `1` | binlog 的 fsync 频率（server 层） |
| `innodb_redo_log_encrypt` | `OFF` | redo 加密（在 log I/O 层做，**不改变 I/O 大小**） |

### server 层

| 变量 | 默认 | 说明 |
|------|------|------|
| `binlog_cache_size` | **32768** | 事务 binlog cache 的内存大小，也是 spill 到 tmpdir 的阈值 |
| `binlog_stmt_cache_size` | **32768** | 非事务语句的 cache |
| `max_binlog_cache_size` | 极大（≈16 EB，即"无限制"） | 写进 `IO_CACHE::end_of_file`，超限 → `EFBIG` |
| `internal_tmp_mem_storage_engine` | **`TempTable`** | 内部临时表的内存引擎（`MEMORY` / `TempTable`） |
| `tmp_table_size` / `max_heap_table_size` | **16 MB** | 内存临时表达到此大小转磁盘 |
| `temptable_max_ram` / `temptable_max_mmap` | **1 GiB** | TempTable 的 RAM / mmap 上限 |
| `big_tables` | `false` | 置 ON → 直接建磁盘（InnoDB）临时表，跳过内存阶段 |
| **`tmpdir`** | 系统临时目录 | ★ **不要指向 tmpfs**——native AIO 探测会失败，导致整个 AIO 退化到模拟实现 |
| `select_into_disk_sync` | `OFF` | `SELECT INTO OUTFILE` 是否每次 flush 后 fsync |
| `secure_file_priv` | `NULL` | 限制 `LOAD DATA` / `INTO OUTFILE` 的目录 |
| `log_output` | `FILE` | `TABLE` 时写入 `mysql.general_log`/`slow_log`（**CSV 引擎，无 fsync**） |
| `myisam_use_mmap` / `myisam_mmap_size` | — | MyISAM 是否用 mmap 代替 pread/pwrite |
| `key_buffer_size` | — | MyISAM key cache（独立于 OS page cache 的第二层缓存） |

> `internal_tmp_disk_storage_engine` **在 8.0.39 已移除**（详见[核心实现六](#内部临时表与排序文件最容易产生意外-io-的地方)）。

---

## Misc

### 诊断与观测


### 状态变量

| 指标 | 累加位置 | 说明 |
|------|---------|------|
| `Innodb_data_reads` | `os_n_file_reads`（`os0file.cc`） | 读次数 |
| `Innodb_data_writes` | `os_n_file_writes` | 写次数 |
| `Innodb_data_fsyncs` | `os_n_fsyncs`（`os0file.cc`） | **fsync 次数——云盘上最该盯的成本** |
| `Innodb_os_log_fsyncs` | `log_total_flushes` | redo fsync 次数 |
| `Innodb_data_read` / `written` | `MONITOR_*` | 字节数 |
| `Innodb_buffer_pool_wait_free` | — | 持续 >0 = 刷脏跟不上，先查 `innodb_io_capacity` |
| `Innodb_dblwr_writes` / `Innodb_dblwr_pages_written` | `srv_stats.dblwr_pages_written` | doublewrite 写入量（评估写放大） |

### `information_schema.innodb_metrics`

`srv0mon.cc` 里的完整计数器，常用项：

`os_data_reads` / `os_data_writes` / `os_data_fsyncs` / `os_log_fsyncs` / `buffer_data_reads`（真正产生物理读的次数）/ `ibuf_merges` / `innodb_ibuf_merge_usec` / `log_write_notifier_*` / `log_flush_notifier_*`。

### PFS

- instrument 名：`wait/io/file/innodb/innodb_data_file`、`innodb_log_file`、`innodb_temp_file`、`innodb_dblwr_file`、`innodb_arch_file`、`innodb_clone_file`。
- InnoDB 通过 `pfs_os_*` 内联包装接入（`include/os0file.ic/190/347`），同步与异步走**同一套** PSI，所以 `file_summary_by_instance` 同时覆盖两者。
- 按文件实例看：`performance_schema.file_summary_by_instance`。

### error log 里的 I/O 统计

`os_aio_print` / `os_aio_refresh_stats`（`os0file.cc`）每 5 秒由 `srv_monitor_thread` 触发，对比 `os_n_file_reads_old` 等算出 reads/s、writes/s、fsyncs/s 打到 error log。

`os_aio_print_pending_io`（`os0file.h`）在 shutdown hang 时打印未完成的 pending I/O——**排查"关不掉"的利器**。

---


### 选择建议

| 场景 | 建议 |
|------|------|
| 通用生产（内存充足） | 默认 `fsync` + `flush_log_at_trx_commit=1` |
| 专属服务器 / 想消除 double buffering | `innodb_flush_method=O_DIRECT` |
| **云盘 / 网络块存储** | **显式 `O_DIRECT`** + `innodb_io_capacity` 按卷规格 + `innodb_flush_neighbors=0` |
| 极致 redo 性能 | `O_DSYNC`（免 fsync 调用）+ `innodb_use_fdatasync=ON` |
| 大表 DDL 且内存紧张 | `innodb_disable_sort_file_cache=ON` |
| **tmpfs 上的文件** | **不能用 O_DIRECT**（`EINVAL`，会自动告警回退） |

> 核心权衡一句话：**Buffered 快但双重缓存 + 需 fsync；O_DIRECT 省内存但要求对齐、无系统缓存兜底**。数据文件看 `innodb_flush_method`，redo 看 `innodb_flush_log_at_trx_commit`，sort 临时文件看 `innodb_disable_sort_file_cache`，**三者独立**。

### 社区边界澄清（"MySQL 没有 X"）

| 常被问到 | 8.0.39 实际情况 |
|---------|----------------|
| **SQL 层并行查询** | **没有**。只有 InnoDB 内部 `Parallel_reader`，且只服务 DDL / 分区 / 直方图 |
| **逻辑预读（logical read-ahead）** | **没有**。`row_read_ahead_logical` 在 8.0.39 全仓库 **0 匹配**——它是 Facebook / WebScaleSQL 分支的特性（扫描聚簇索引搜集叶子页号后批量异步读），社区版只有随机预读与线性预读 |
| **io_uring** | **没有**（全仓库 0 匹配）。仍是 libaio + Windows AIO + Simulated AIO 三套 |
| **`log_closer` 线程** | **已不存在**（8.0 只剩 `closer_mutex`，由用户线程代劳） |
| **文件型 audit log 插件** | **社区版没有**，只有示例插件 `plugin/audit_null`（写 error log） |
| **DROP / TRUNCATE 的异步文件清理** | **没有**，同步 `unlink` |
| **`IORequest::COMPRESS` / `ENCRYPT` 位标志** | **不是位标志**，是成员对象 `m_compression` / `m_encryption` |
| **通用的 O_DIRECT 对齐校验函数** | **没有**，靠 `ut::aligned_alloc` + redo 侧断言保证 |
| **PG 式的 O_DIRECT 关闭选项** | MySQL **有** O_DIRECT；反倒是 PG **没有** |

### 常见误解

| 误解 | 正确 |
|------|------|
| "MySQL 全程用异步 I/O" | 只有**数据文件**走 libaio；redo、dblwr、binlog、各类日志全是同步写 |
| "redo 也走 `os_aio`" | **不**：`os_aio_func` 有 `ut_a(!type.is_log)` |
| "单页刷盘不走 doublewrite" | **走**，只是用文件尾部的 `SYNC_PAGE_FLUSH_SLOTS` |
| "`FLUSH TABLES` 会刷所有脏页" | 不会，它关表缓存 + `ha_flush`；全量刷脏是 `buf_flush_sync_all_buf_pools` |
| "加密会增大 I/O" | 不会，AES 分组加密长度不变 |
| "O_DIRECT 就不需要 fsync 了" | 取决于取值：`O_DIRECT` **仍 fsync**；只有 `O_DIRECT_NO_FSYNC` 才跳过（且 dblwr 也跟着跳过） |
| "开了 `innodb_dedicated_server` 就是调优了" | 它会**静默改写 flush_method**，云上是数据安全风险 |
| "ibuf merge 有独立后台线程" | 没有，寄生在 master 线程 |
| "异步 I/O = 调用者不用等" | 不等的是**下发方式**；缺页读对调用者仍是同步等待，只是底层用 `io_submit` 以便合并多个请求 |
| "定了 `innodb_use_native_aio=ON` 就一定会用 libaio" | 不一定：**探测会检查 tmpdir**，`tmpdir` 在 tmpfs 上会失败并**静默退化**到模拟 AIO（error log 里只有一句 "Linux Native AIO disabled."） |
| "可以配置磁盘内部临时表引擎" | `internal_tmp_disk_storage_engine` **8.0.39 已移除**，磁盘内部临时表引擎硬编码为 InnoDB |
| "binlog 会 fsync 目录项（dentry）" | 不会。`my_sync_dir` / `my_sync_dir_by_file` 已不存在，`sql/binlog.cc` 里只剩一条 TODO 注释 |
| "`log_output=TABLE` 比 `FILE` 省 I/O" | 相反：写的是 `mysql.general_log`/`slow_log` 两张 **CSV 表**（无 fsync 但有表级锁竞争），且 8.0 里它们仍是 CSV 引擎而非 InnoDB |
| "`my_sync` 受 `innodb_use_fdatasync` 控制" | 否。`my_sync` 用 `fdatasync` 是**编译期 `#if`** 决定的，与该变量无关（后者只影响 InnoDB 的 `os_file_fsync`） |

---

## 参考

**论文 / 经典理论**

- Mohan et al. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. **ACM TODS 1992**. （steal/no-force + WAL；源码落点见[理论溯源](#理论溯源)）
- Gray & Reuter. *Transaction Processing: Concepts and Techniques*. Morgan Kaufmann. （WAL 与缓冲管理的经典表述）
- Gray & Putzolu. *The 5 Minute Rule for Trading Memory for Disk Accesses*. **SIGMOD 1987**. （Buffer Pool 存在的经济学依据）
- Graefe. *Volcano — An Extensible and Parallel Query Evaluation System*. **IEEE TKDE 1994**. （迭代器模型 → 逐行随机读的根源）

**官方文档**

- *MySQL 8.0 Reference Manual → InnoDB I/O Configuration*（`innodb_flush_method` / `innodb_io_capacity` / `innodb_flush_neighbors`）
- *MySQL 8.0 Reference Manual → Optimizing InnoDB Disk I/O*
- *MySQL 8.0 Reference Manual → InnoDB Startup Options and System Variables*

**源码**（MySQL 8.0.39，本文所有函数名与行号均已核实）

- `storage/innobase/fil/fil0fil.cc` / `include/fil0fil.h` —— `fil_io`、`Fil_shard::do_io`、`get_AIO_mode`、`space_flush`、`space_delete`、`fil_aio_wait`
- `storage/innobase/os/os0file.cc` / `.h` / `.ic` —— `os_file_create_func`、`os_file_set_nocache`、`os_file_fsync_posix`、`os_file_write_retry`、`os_file_punch_hole`、`os_file_encrypt_page`、`os_aio_func`、`AIO::linux_dispatch`、`LinuxAIOHandler::collect`、`SimulatedAIOHandler`、`os_file_handle_error_cond_exit`
- `storage/innobase/include/os0file.h` —— 文件类型常量、`IORequest` 位标志、`AIO_mode`
- `storage/innobase/buf/buf0dblwr.cc` —— `dblwr::write`、`Segment::write/flush`、`is_fsync_required`、`write_complete`、`dblwr_file_open`
- `storage/innobase/buf/buf0flu.cc` —— `buf_flush_write_block_low`、`buf_flush_try_neighbors`、`set_flush_target_by_lsn`、`buf_flush_sync_all_buf_pools`、`buf_flush_fsync`
- `storage/innobase/buf/buf0rea.cc` —— `buf_read_page`、`buf_read_ahead_linear`、`buf_read_ahead_random`、`buf_read_ibuf_merge_pages`
- `storage/innobase/log/` —— `log0write.cc`（`log_writer`/`log_flusher`/`log_writer_write_buffer`/`log_flush_low`）、`log0files_io.cc`（`Log_file_handle::write/fsync/prepare_io_request`）、`log0files_governor.cc`、`log0chkp.cc`、`log0ddl.cc`、`log0recv.cc`
- `storage/innobase/srv/srv0start.cc` —— 各类后台线程的创建
- `storage/innobase/handler/ha_innodb.cc` —— 各 I/O sysvar 定义、`innodb_dedicated_server` 的 flush_method 改写
- `sql/binlog.cc`、`mysys/mf_iocache.cc`、`sql/log.cc`、`sql/server_component/log_sink_*.cc` —— server 层文件 I/O

**相关文档**

- Buffer Pool 的替换算法与脏页调度（本篇的上游）见 [`buffer_pool.md`](buffer_pool.md)
- redo 的**格式**、LSN 与 checkpoint 语义见 [`redo_log.md`](redo_log.md)
- 崩溃恢复中 I/O 的角色见 [`recovery.md`](recovery.md)
- DDL 的临时文件、row log 与并行扫描见 [`ddl.md`](ddl.md)
- **云存储（EBS / 云盘 / CBS / ESSD）本身**见 [`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)
- C++ ABI 相关的常量定义约定（`static const ulint` 而非 enum）见 [`../server/plugin/abi.md`](../server/plugin/abi.md)
