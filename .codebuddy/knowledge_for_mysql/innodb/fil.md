# InnoDB 文件与表空间层（fil）深度解析

> 基于 MySQL 8.0.39 源码。`fil` 层（Tablespace Memory Cache）是 InnoDB **所有表空间文件 I/O 的统一入口与元数据中心**：管理 `fil_space_t` / `fil_node_t`、做 `space_id` 分片、决定每次 I/O 是同步还是异步、给 `IORequest` 补上压缩/加密/打洞语义，并用一套 flag + 计数器协调扩展 / flush / DROP / TRUNCATE / RENAME 的并发。

> **边界**：本篇讲 **fil 层（表空间元数据与 I/O 分发）**。它下面的 `os_file` 原语（O_DIRECT / fsync / `os_file_io` 总收口）与 AIO 子系统见 [`io.md`](io.md)；Buffer Pool 的刷脏与读页见 [`buffer_pool.md`](buffer_pool.md)；redo 的格式见 [`redo_log.md`](redo_log.md)；崩溃恢复中 fil 的角色见 [`recovery.md`](recovery.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [三层结构与分片](#三层结构与分片)
    - [文件类型与后缀](#文件类型与后缀)
  - 主链路（读写共用）
    - [主链路 fil_io → Fil_shard::do_io](#主链路-fil_io--fil_sharddo_io)
    - [★ AIO 模式的三选一（含缺页读为何是同步）](#-aio-模式的三选一含缺页读为何是同步)
  - 写路径
    - [I/O 并发控制（flag 与计数器体系）](#io-并发控制flag-与计数器体系)
    - [文件扩展 space_extend](#文件扩展-space_extend)
    - [刷盘与 fsync 并发去重 space_flush](#刷盘与-fsync-并发去重-space_flush)
  - 生命周期
    - [文件生命周期（打开 / 关闭 / 删除）](#文件生命周期打开--关闭--删除)
- [相关的系统变量](#相关的系统变量)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

`fil` 层回答三个问题：

1. **一个 `space_id` 对应哪些物理文件？**（`fil_space_t` → `fil_node_t` 向量）
2. **一次页读写怎么变成一次文件 I/O？**（`page_no` → 文件偏移 → `os_aio`）
3. **多个操作抢同一个文件时怎么协调？**（flag + 计数器，不是锁）

它是 InnoDB 里**唯一**知道".ibd 文件在哪、多大、能不能写"的地方。上层（Buffer Pool、DDL、clone）只传 `page_id_t`，其余全部由 fil 解释。

### 用途

| 调用方 | 用什么 fil 接口 |
|--------|----------------|
| Buffer Pool 读页 | `fil_io(READ, sync, page_id, ...)` |
| Buffer Pool 刷脏 | `fil_io(WRITE, ...)` |
| 崩溃恢复 | `fil_io` + `fil_tablespace_redo_extend` |
| DDL 建/删/截断表 | `fil_create_tablespace` / `fil_delete_tablespace` / `Fil_shard::space_truncate` |
| 刷盘 | `fil_flush` / `fil_flush_file_spaces` |
| clone | `fil_clone_...` + `OS_CLONE_DATA_FILE` |
| 文件扩展 | `fil_space_extend` |

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | `fil_system` 全局单例 + 单把 mutex；`fil_space_t` / `fil_node_t` 结构定型 |
| 5.7 | 独立 undo 表空间、临时表空间（`ibt`）引入；`fil_space_t` 增加 `purpose`（`FIL_TYPE_TEMPORARY` 等） |
| 8.0 | **★ `Fil_shard` 分片**（5.7 是单 mutex，8.0 拆成 68 把）；`fil_node_t` 增加 `flush_size` / `modification_counter` / `flush_counter` 支持 `O_DIRECT_NO_FSYNC`；`space_extend` 改为 `posix_fallocate` + 可选写零；引入 `m_undo_extend` 自适应扩展 |

> **为什么演进**：8.0 的分片是为了解决高并发 DDL（建表/删表/刷表空间）时的 `fil_system` mutex 争用；`flush_size` 那套计数器是为了支持 `O_DIRECT_NO_FSYNC` 下"只在文件大小变化时 fsync"的语义。

---

## 理论基础

### 设计思想与权衡

#### 一、为什么要把 `fil_system` 拆成 68 个 shard

5.7 时代所有表空间的元数据操作抢**一把**全局 mutex。高并发建表 / 删表 / 刷脏时这把锁成为瓶颈。

8.0 的解法是**按 `space_id` 取模分片**：不同表空间的 I/O 落在不同 shard、不同 mutex，互不干扰。

> **为什么是 68 而不是 64**：64 个普通 shard + 4 个 undo 专用（`UNDO_SHARDS_START` 之后）。undo 独立是因为 **undo 表空间的 extend 会持锁较久**（要 `posix_fallocate` + 写 redo + fsync），不能让它卡住普通表空间。

**代价**：分片数固定（不是动态扩容），`space_id` 分布不均时可能偏斜；跨 shard 的操作（如 `fil_flush_file_spaces` 遍历所有未刷表空间）需要逐个加锁。

#### 二、为什么并发控制用"flag + 计数器"而不是锁

对**同一个文件**的并发操作，fil 层**不用读写锁**协调，而是靠一组标志位与计数器：

| 为什么 | 说明 |
|--------|------|
| **I/O 本身不需要锁** | 底层是 `pread`/`pwrite`，不同线程读写同一文件天然安全（OS 保证） |
| **需要协调的是"结构性操作"** | 扩展、删除、截断、重命名——这些操作期间**不允许**有并发 I/O 或关闭 |
| **计数器即屏障** | `n_pending_ios > 0` ⇒ 不能关文件；`n_pending_flushes > 0` ⇒ 不能删表空间 |

> **代价**：等待方式是**轮询 + 睡眠**（`space_extend` 里 20μs / 100ms），不是事件驱动。源码注释自嘲：*"It'd have been better to use an event driven mechanism but the entire module is peppered with polling code."*

#### 三、`page_no → 文件偏移` 的映射是"哑"的

```cpp
auto offset = (os_offset_t)page_no * page_size.physical;
offset += byte_offset;
```

**LBA 完全由 `page_no << 14` 决定**。这意味着：

- 页的物理位置由逻辑页号线性决定，**与底层设备的物理布局无关**；
- 在云盘上，这层映射之后还要再做一次 LBA→chunk 的映射——**MySQL 完全不知情**（见 [`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)）；
- 顺序性只能靠"相邻 `page_no`"来近似，这是预读与邻接刷盘能工作的前提。

#### 四、为什么 `IORequest` 的压缩/加密语义在 fil 层注入

`buf_flush_write_block_low` 只说"写这个页"，**不知道**它要不要压缩、用什么密钥加密。这些语义在 `Fil_shard::do_io` 里补上：

```cpp
fil_io_set_encryption(req_type, page_id, space);
req_type.block_size(file->block_size);
```

> **设计收益**：上层只描述"意图"，底层统一决定"怎么变换"。这让加密/压缩对 Buffer Pool 完全透明。

### 他库对比

| 数据库 | 表空间元数据管理 |
|--------|----------------|
| **MySQL / InnoDB** | `fil_system` + `Fil_shard × 68`，内存缓存 + 分片 mutex |
| **PostgreSQL** | 没有独立的"表空间缓存层"——`relfilenode` → 文件路径由 `smgr`（`md.c`）直接算，靠 OS 的 open cache |
| **Oracle** | 数据字典 + 控制文件，ASM 时代由 ASM 实例管理文件布局 |

**差异根源**：MySQL 支持"一个表空间多个文件"（`fil_space_t.files` 是向量，最后一文件可自动扩展），所以需要一层元数据；PG 一个 relation 就是一坨文件（可能分 segment），映射更简单。

---

## 核心实现

### 三层结构与分片

```
fil_system（全局单例，fil0fil.cc）
  └── Fil_shard × 68                ← 路由：space_id % 64；undo 走 UNDO_SHARDS_START 之后
        └── m_shards[i]:
              ├── mutex（每 shard 一把，LATCH_ID_FIL_SHARD）
              ├── m_spaces    : 本 shard 的 fil_space_t 集合
              ├── m_unflushed_spaces : 有未刷改动的表空间链表
              ├── m_LRU       : 打开的文件句柄 LRU（受 innodb_open_files 限制）
              └── m_modification_counter : 本 shard 的全局写序号
```

```cpp
// storage/innobase/fil/fil0fil.cc
/** Shard ID for undo tablespaces. */
constexpr size_t UNDO_SHARDS_START = 64;
/** Total number of shards. */
constexpr size_t MAX_SHARDS = 68;
```

**`fil_space_t`（一个表空间）**的关键成员：

| 字段 | 用途 |
|------|------|
| `id` / `name` | 表空间标识 |
| `files` | `std::vector<fil_node_t>`——**一个表空间可以多个文件**（最后一文件可自动扩展） |
| `size` | 表空间总页数 |
| `flags` | 页大小、压缩类型、加密等 |
| `purpose` | `FIL_TYPE_TABLESPACE` / `FIL_TYPE_TEMPORARY` / `FIL_TYPE_LOG` / `FIL_TYPE_IMPORT` |
| `stop_new_ops` | 正在被 drop/truncate，禁止新操作 |
| `n_pending_flushes` | 正在 fsync 中（**阻止 drop**） |
| `n_pending_ops` | 待处理的 change buffer merge |
| `m_undo_extend` / `m_undo_initial` | undo 自适应扩展量 |

**`fil_node_t`（一个物理文件）**的关键成员：

| 字段 | 用途 |
|------|------|
| `handle` / `name` / `is_open` | 文件句柄与状态 |
| `size` | 本文件的页数 |
| `flush_size` | **最后一次 fsync 时的文件大小**（`O_DIRECT_NO_FSYNC` 下靠它触发 fsync） |
| `n_pending_ios` | 在途 I/O 数（**阻止 close**） |
| `n_pending_flushes` | 在途 fsync 数（并发去重） |
| `is_being_extended` | 正在扩展（**扩展之间的互斥**） |
| `modification_counter` / `flush_counter` | 写序号 / 已刷到的序号 |
| `sync_event` | fsync 的同步事件（**合并 fsync** 用） |
| `block_size` | 设备扇区大小 |
| `punch_hole` | 是否支持打洞（page compression） |

---

### 文件类型与后缀

#### 类型常量：不是 enum，是 `static const ulint`

`include/os0file.h`：

| 常量 | 值 | 说明 |
|---|---|---|
| `OS_DATA_FILE` | 100 | 数据文件（.ibd / 系统 / undo / temp） |
| `OS_LOG_FILE` | 101 | redo log |
| `OS_BUFFERED_FILE` | 102 | 小文件 / 写入字节数非扇区整数倍时强制 buffered |
| `OS_CLONE_DATA_FILE` | 103 | clone 拷贝的数据文件 |
| `OS_CLONE_LOG_FILE` | 104 | clone 拷贝的 redo |
| `OS_DBLWR_FILE` | 105 | doublewrite 文件 |
| `OS_REDO_LOG_ARCHIVE_FILE` | **105** | redo archive（与 DBLWR **同值**，历史遗留） |

> **为什么不用 enum**：避免 `-fshort-enums` 带来的 ABI 问题（这类约定见 `../server/plugin/abi.md`）。

**这些类型决定 O_DIRECT 是否生效**：只有 `OS_DATA_FILE` / `OS_CLONE_DATA_FILE` / `OS_DBLWR_FILE` 会走 O_DIRECT 分支，`OS_LOG_FILE` 不会（见 [`io.md`](io.md)）。

#### 文件后缀

`enum ib_file_suffix`（`fil0fil.h`）：`.ibd` / `.cfg` / `.cfp` / `.ibt` / `.ibu` / `.dblwr` / `.bdblwr`。

---

### 主链路 `fil_io` → `Fil_shard::do_io`

#### 入口

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

#### `Fil_shard::do_io` 的尾部：所有附加语义在这里补上

`fil0fil.cc`：

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

1. **地址映射**：`offset = page_no × page_size.physical + byte_offset`。**LBA 完全由 `page_no << 14` 决定**——页的物理位置与底层设备布局无关（云盘上还要再做一次映射，MySQL 不知情）。
2. **punch hole 决策（page compression）**：写 + 非压缩表 + **非 page 0** + 文件系统支持打洞 → 打上 `PUNCH_HOLE` 与压缩算法。**排除 page 0**（页 0 是表空间头，不能被"打洞"成稀疏文件）。
3. **加密注入**：`fil_io_set_encryption` 把密钥信息塞进 `IORequest`；真正的加密在更底层的 `os_file_encrypt_page`（见 [`io.md`](io.md)）。
4. **下发**：`os_aio` 是同步/异步的分流点，模式来自 `get_AIO_mode`（见下）。

---

### ★ AIO 模式的三选一（含缺页读为何是同步）

#### `Fil_shard::get_AIO_mode`（`fil0fil.cc`）

```cpp
AIO_mode Fil_shard::get_AIO_mode(const IORequest &type, bool sync) {
  if (sync) {
    return AIO_mode::SYNC;
  } else if (type.is_ibuf) {
    return AIO_mode::IBUF;
  }
  return AIO_mode::NORMAL;
}
```

| 模式 | 常量 | 何时用 |
|------|------|--------|
| `AIO_mode::SYNC` | 24 | `sync == true`——**直接同步 `pwrite`/`pread`，不进 AIO 子系统** |
| `AIO_mode::NORMAL` | 21 | 普通异步 I/O（预读、刷脏） |
| `AIO_mode::IBUF` | 22 | **change buffer 相关的页读** |

> **`IBUF` 为什么单独存在**：ibuf merge 时要读入二级索引页。如果它和普通读抢同一批 AIO 槽位，可能出现"所有槽位都被 ibuf merge 的读占满，而 ibuf merge 又在等这些读完成"的死锁。**单独的 `s_ibuf` 数组 + 单独的线程**把这个环切断。

#### ★ 缺页读为什么是 `SYNC`（完整判定链）

这是理解 InnoDB 读路径最容易搞错的一处。完整证据链：

| 步骤 | 位置 | 证据 |
|---|---|---|
| 1 | `buf0rea.cc` | `buf_read_page_low(&err, /*sync=*/true, 0, BUF_READ_ANY_PAGE, ...)` |
| 2 | `fil0fil.cc` | `get_AIO_mode: if (sync) return AIO_mode::SYNC;` |
| 3 | `os0file.cc` | `if (aio_mode == AIO_mode::SYNC) { return os_file_read_func(...); }`——注释："*no need to use an i/o-handler thread*" |
| 4 | `os0file.cc` | `SyncFileIO::execute` → **`pread`** |
| 5 | `buf0rea.cc` | `if (sync) { /* The i/o is already completed when we arrive from fil_read */ ... }` |

**结论：单页缺页读 = 调用线程自己的 `pread`，不占 AIO 槽位，不需要 io-handler 线程。**

#### 真正走 `sync=false`（真 AIO）的三条路径

| 调用者 | sync | 说明 |
|---|---|---|
| `buf_read_ahead_random` / `_linear` | **false** | 预读；用 `DO_NOT_WAKE` 攒批，最后统一唤醒（`buf0rea.cc`） |
| `buf_read_ibuf_merge_pages` | 仅最后一页 true | `AIO_mode::IBUF` |
| `buf_read_recv_pages` | 仅最后一页 true | 崩溃恢复 |

#### 有些页会被强制降级为同步（`buf0rea.cc`）

```cpp
  if (ibuf_bitmap_page(page_id, page_size) || trx_sys_hdr_page(page_id)) {
    /* Trx sys header is so low in the latching order that we play
    safe and do not leave the i/o-completion to an asynchronous
    i/o-thread. Ibuf bitmap pages must always be read with
    synchronous i/o, to make sure they do not get involved in
    thread deadlocks. */
    sync = true;
  }
```

> **理由**：这两个页在 latching order 里位置极低。若由异步 io-handler 线程完成后处理（会去拿 ibuf 树上的 latch），可能造成死锁。

**对应的断言**（`fil0fil.cc`）：

```cpp
  ut_ad(recv_no_ibuf_operations || req_type.is_write ||
        !ibuf_bitmap_page(...) || sync);
```

#### ⭐ 并发的真正来源

N 个用户线程各缺一个**不同**的页 → N 个并发 `pread`（内核层面天然并行）。

**InnoDB 缺页读从不做页合并**——一次 `buf_read_page_low` 永远只发一个页。合并只发生在预读层：那也是**逐页循环调用** `buf_read_page_low`，只是用 `DO_NOT_WAKE` 攒批、最后统一唤醒 io 线程。

---

### I/O 并发控制（flag 与计数器体系）

对**同一个文件**的并发操作不靠锁，靠一组标志位与计数器。这套机制决定了"为什么大表 DROP/TRUNCATE 有时会卡住"。

| 场景 | 机制 |
|------|------|
| **文件扩展中**（`Fil_shard::space_extend`） | `fil_node_t::is_being_extended = true`——**扩展之间的互斥**；同时 `n_pending_ios > 0` 阻止 close/rename/delete |
| **删表**（`fil_delete_tablespace`） | 先 `fil_space_t::stop_new_ops = true`，再检查三类 pending 全部归零：change buffer merge（`n_pending_ops`）、异步 I/O（`n_pending_ios`）、文件 flush（`n_pending_flushes`） |
| **TRUNCATE** | `is_being_truncated = true` + 同样的 pending 检查 |
| **RENAME** | `stop_ios = true`，**阻止该文件上所有 I/O** |
| 异步读时发现 `stop_new_ops` | **返回错误**（但同步读与异步写仍被允许） |
| flush 时发现 `stop_new_ops` / `is_being_truncated` | **跳过**该文件的 flush（不报错） |

**能关闭一个文件的条件**（`fil_node_t::can_be_closed`，`fil0fil.cc`）：

```cpp
  if (n_pending_ios != 0)       return false;   // 等在途 I/O
  if (n_pending_flushes != 0)   return false;   // 等在途 fsync
  if (is_being_extended)        return false;   // ★ 等扩展完成
  if (is_fast_shutdown)       return true;    // fast shutdown=2 例外
```

**`is_being_extended` 为什么不能只用 `n_pending_ios`**：`n_pending_ios` 是"文件正在被读写"的**通用计数**，无法表达"**扩展操作正在进行**"这一特定状态。两个扩展线程之间需要互斥（否则会重复计算扩展量、重复 `fallocate`、把 `space->size` 加两次），这就是 `is_being_extended` 的独立职责。

> **实践意义**：`DROP TABLE` / `TRUNCATE` 在大表上有 pending I/O 时**会等待**，这就是"删表卡住"的根因（也解释了为什么阿里 RDS 与 Percona 各自做了 buffer pool 层面的优化来缩短这个窗口）。

---

### 文件扩展 `space_extend`

`Fil_shard::space_extend`（`fil0fil.cc`）。

#### 排队：为什么是轮询

```cpp
  for (;;) {
    mutex_acquire;
    space = get_space_by_id(space->id);
    if (size < space->size) { mutex_release; return true; }   // 已经够大
    file = &space->files.back;
    if (!file->is_being_extended) {
      file->is_being_extended = true;
      break;
    }
    mutex_release;
    if (!tbsp_extend_and_initialize) {
      std::this_thread::sleep_for(std::chrono::microseconds(20));
    } else {
      std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
  }
```

**为什么要释放 shard mutex 再等待**：扩展过程要做 `posix_fallocate` / 写零（毫秒级阻塞），持锁会冻结整个 shard。释放期间必须有 `is_being_extended` 表示"正在扩展"。

#### 怎么扩展

```
posix_fallocate(fd, node_start, len)     ← 预留空间、更新 FS 元数据，不写数据
      ↓ 失败（EINVAL/EINTR 等）或默认 innodb_extend_and_initialize=ON
fil_write_zeros(...)                      ← 按 1MB 一批写零
```

- 失败时 `EINVAL`（ext3 + O_DIRECT）/ `EINTR`（被信号中断）不报错，直接走写零；其它错误报 `ER_IB_MSG_319`。
- `atomic_write`（FusionIO）时跳过写零。

#### ★ 会产生 redo，但故意不 `log_write_up_to`

```cpp
      fil_op_write_space_extend(space->id, node_start, len, &mtr);   // MLOG_FILE_EXTEND
      mtr_commit(&mtr);
      /* NOTE: Though against for the Write-Ahead-Log principal,
      log_write_up_to is not needed here, because no file shrinks and
      duplicate extending is allowed. */
```

- **为什么记 redo**：`posix_fallocate` 存在原子性问题——可能"预留了空间但没更新 FS 元数据"就崩溃。恢复时只看文件大小无法知道旧大小，redo 里的 `offset` 给出了"从哪开始需要初始化"的精确起点。
- **为什么不刷 redo**：扩展是幂等的、文件只增不减，重复扩展无害——**这是有意违反 WAL**。
- **例外**：临时表空间（`FIL_TYPE_TEMPORARY`）与系统表空间不记 redo（前者每次启动重建，后者不 resize）。

#### ★ 末尾一定 `space_flush`

哪怕 `O_DIRECT_NO_FSYNC`。因为**文件大小变化必须同步 FS 元数据**，否则重启后可能读到"短文件"。

#### undo 的自适应扩展量

`fil_space_t::m_undo_extend`（`fsp_try_extend_data_file` + `adjust_undo_extend`）：

- **100 ms 内又需要扩展** → 步长**翻倍**（上限 16 × `UNDO_INITIAL_SIZE_IN_PAGES`）
- **超过 100 ms** → 步长**减半**（下限回到初始值）

目的：突发大事务期间少走扩展路径（每次扩展都要写 redo + fsync），空闲后收敛回小步长避免文件虚胖。

---

### 刷盘与 fsync 并发去重 `space_flush`

`Fil_shard::space_flush`（`fil0fil.cc`）。

#### ★ 合并 fsync

```cpp
    while (file.n_pending_flushes > 0 && !skip_flush) {
      /* We want to avoid calling os_file_flush on the file twice at the same
      time, because we do not know what bugs OS's may contain in file I/O */
      int64_t sig_count = os_event_reset(file.sync_event);
      mutex_release;
      os_event_wait_low(file.sync_event, sig_count);
      mutex_acquire;
      if (file.flush_counter >= old_mod_counter) {
        skip_flush |= true;              // ★ 别人的 fsync 已覆盖我要的，我不刷了
      }
      skip_flush |= is_fast_shutdown;
    }

    if (!skip_flush) {
      ++file.n_pending_flushes;
      mutex_release;
      os_file_flush(file.handle);        // ★ fsync 在无锁状态下做
      file.flush_size = file.size;
      mutex_acquire;
      os_event_set(file.sync_event);     // 唤醒所有等待者
      --file.n_pending_flushes;
    }
```

多个线程同时想 fsync 同一个文件时，**只有一个真做**，其余等 `sync_event`；醒来后若发现 `flush_counter >= old_mod_counter` 就**直接跳过**。这正是 `sync_event` 注释说的 *"event that groups and serializes calls to fsync"*。

#### 三个计数器（不是两个）

| 字段 | 何时更新 | 用途 |
|---|---|---|
| `m_modification_counter`（shard 级） | 每次写 I/O 完成 +1 | 全局单调"写序号" |
| `file.modification_counter` | `= ++m_modification_counter` | **本文件最后一次写**的序号 |
| `file.flush_counter` | fsync 成功后 `= old_mod_counter` | **最后一次 fsync 覆盖到**的序号 |
| `file.flush_size` | fsync 成功后 `= file.size` | 最后一次 fsync 时的**文件大小（页）** |

- 判定"要不要刷"用**快照** `old_mod_counter`（进入时取），而不是 `file.modification_counter`——因为 fsync 期间（无锁）可能有新写进来，那些不该算"我已刷"。
- **为什么需要 `flush_size`**：`O_DIRECT_NO_FSYNC` 模式下每次写后立即 `set_flushed`，`mod/flush_counter` **恒相等**，无法判断。此时唯一需要 fsync 的事件是**文件变大**，所以用 `flush_size != size` 触发。

#### `is_fast_shutdown` 跳过但仍记账

```cpp
static bool is_fast_shutdown {
  return srv_shutdown_state >= SRV_SHUTDOWN_LAST_PHASE && srv_fast_shutdown >= 2;
}
```

`innodb_fast_shutdown=2`（crash-style）且已进入最后阶段 → **不 fsync**，但仍执行 `flush_counter = old_mod_counter` 记账（一致性由崩溃恢复保证）。

#### `flush_file_spaces` 的遍历策略

```cpp
void Fil_shard::flush_file_spaces {
  Space_ids space_ids;
  mutex_acquire;
  for (auto space : m_unflushed_spaces) {
    if ((to_int(space->purpose) & FIL_TYPE_TABLESPACE) && !space->stop_new_ops) {
      space_ids.push_back(space->id);
    }
  }
  mutex_release;

  /* Flush the spaces. It will not hurt to call fil_flush on a non-existing
     space id. */
  for (auto space_id : space_ids) {
    mutex_acquire;
    space_flush(space_id);
    mutex_release;
  }
}
```

> **为什么先快照再逐个处理**：`space_flush` 内部会释放并重新获取 shard mutex（等待/执行 fsync），边遍历边释放会导致迭代器失效。快照 + 逐个的模式规避了这个问题；且 `space_flush(nullptr)` 对已删除的 space 是安全的。

---

### 文件生命周期（打开 / 关闭 / 删除）

| 操作 | 入口 | 说明 |
|------|------|------|
| 打开 | `Fil_shard::open_file` | 需要时（`prepare_file_for_io`）才打开；`is_open` 标记 |
| I/O 前准备 | `prepare_file_for_io`（`fil0fil.cc`） | `++n_pending_ios` 并把文件**从 LRU 摘出**（防止被关） |
| I/O 完成 | `complete_io` | `--n_pending_ios`；若是写则 `write_completed`（bump 计数器 + 挂入 `m_unflushed_spaces`） |
| 关闭 | `Fil_shard::close_file` | 需要 `n_pending_ios == 0` 等条件 |
| LRU 淘汰句柄 | `close_files_in_LRU` | 超过 `innodb_open_files` 时关 LRU 尾部句柄 |
| 删除 | `Fil_shard::space_delete` / `fil_delete_tablespace` | **`unlink` 同步执行，没有异步/AIO 后台清理** |
| 截断 | `Fil_shard::space_truncate` | `is_being_truncated` + pending 检查 |

**`prepare_file_for_io` 的屏障作用**（`fil0fil.cc`）：

```cpp
bool Fil_shard::prepare_file_for_io(fil_node_t *file) {
  ut_ad(mutex_owned);
  fil_space_t *space = file->space;
  if (space->is_deleted) return false;             // 已被标记删除 → 失败
  if (!file->is_open) {
    ut_a(file->n_pending_ios == 0);
    if (!open_file(file)) return false;
  }
  if (file->n_pending_ios == 0) remove_from_LRU(file);
  ++file->n_pending_ios;                             // ★ 阻止 close
  ut_ad(!ut_list_exists(m_LRU, file));
  return true;
}
```

> **`space->is_deleted` 的检查**是 DROP 与 I/O 之间的第一道屏障：`stop_new_ops` 置位后，新的 `prepare_file_for_io` 直接失败。

---

## 相关的系统变量

| 变量 | 默认 | 作用域 | 说明 |
|------|------|--------|------|
| `innodb_open_files` | -1（自适应） | Global，READONLY | 同时打开的文件句柄上限；超出时 `close_files_in_LRU` |
| `innodb_data_file_path` | `ibdata1:12M:autoextend` | Global，READONLY | 系统表空间文件与自动扩展配置 |
| `innodb_undo_tablespaces` | 2 | Global，READONLY（已废弃） | undo 表空间数量 |
| `innodb_temp_data_file_path` | `ibtmp1:12M:autoextend` | Global，READONLY | 临时表空间 |
| `innodb_extend_and_initialize` | `ON` | Global，动态 | 扩展时是否写零初始化（`tbl_status` 相关） |
| `innodb_max_undo_log_size` | 1 GiB | Global，动态 | undo truncate 的阈值（与 `m_undo_initial` 配合） |
| `innodb_flush_method` | `fsync` | Global，READONLY | 影响 `space_flush` 的 `disable_flush` 判定（`O_DIRECT_NO_FSYNC`） |
| `innodb_fast_shutdown` | `1` | Global，动态 | `=2` 时 `space_flush` 跳过 fsync |

---

## Misc

### 社区边界澄清

| 常被问到 | 8.0.39 实际情况 |
|---------|----------------|
| **DROP / TRUNCATE 的异步文件清理** | **没有**。`unlink` 同步执行，没有 AIO 后台清理 |
| **`space_extend` 的事件驱动等待** | **没有**，是轮询 + 睡眠（源码注释自嘲） |
| **`fil_node_t` 上的读写锁** | **没有**，靠 `n_pending_ios` / `n_pending_flushes` / `is_being_extended` 三个计数器协调 |
| **`OS_DBLWR_FILE` 与 `OS_REDO_LOG_ARCHIVE_FILE` 不同值** | **同值**（都是 105），历史遗留 |

### 常见误解

| 误解 | 正确 |
|------|------|
| "缺页读走异步 AIO" | **不**：`sync=true` → `AIO_mode::SYNC` → 调用线程 `pread`。异步只在预读/ibuf/recovery |
| "所有 I/O 都可能 `io_submit`" | 只有 `AIO_mode::NORMAL` / `IBUF`；`SYNC` 直接系统调用 |
| "扩展文件会等 redo 落盘" | **不**：记了 `MLOG_FILE_EXTEND` 但故意不 `log_write_up_to`（违反 WAL，因扩展幂等） |
| "多个 fsync 会重复调用" | 不会：`sync_event` + `n_pending_flushes` 会把并发 fsync **合并成一次** |
| "undo 表空间和普通表空间抢同一把锁" | 不会：undo 走 `UNDO_SHARDS_START`(64) 之后的专用 shard |

---

## 参考

> 本文结论基于 MySQL 8.0.39 源码逐行核实；涉及的函数与类型可直接在本仓库检索。

**相关文档**

- fil 之下的 `os_file` 原语、AIO 子系统与 I/O 全景：[`io.md`](io.md)
- Buffer Pool 的刷脏与读页（fil 的上游）：[`buffer_pool.md`](buffer_pool.md)
- 云盘/块存储（fil 之下的设备语义）：[`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)
- C++ ABI 相关的常量定义约定（`static const ulint` 而非 enum）：[`../server/plugin/abi.md`](../server/plugin/abi.md)
