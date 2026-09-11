# MySQL 的 I/O：Buffered I/O 与 O_DIRECT

> 基于 MySQL 8.0.39 源码，聚焦 InnoDB 的两种核心 I/O 方式——**Buffered I/O（write + fsync）** 与 **O_DIRECT**，讲清它们的差异、各自适用场景，以及 `innodb_flush_method` 如何把它们串起来。这是理解 InnoDB 持久性、性能与"双重缓存"问题的基础。

## 目录

- [概述](#概述)
- [理论基础：Linux 的三种 I/O 方式](#理论基础linux-的三种-io-方式)
- [核心实现：innodb_flush_method](#核心实现innodb_flush_method)
- [各文件的 I/O 路径](#各文件的-io-路径)
- [系统变量速查](#系统变量速查)
- [选择建议](#选择建议)
- [参考](#参考)

---

## 概述

InnoDB 的持久性建立在"先写日志、后刷数据页"之上，但落到操作系统层，一切都归结为两种打开文件的方式：

- **Buffered I/O**：读写都经过 **OS page cache**，写用 `write()`（到 page cache 即返回），真正落盘要 `fsync()`；
- **O_DIRECT**：绕过 OS page cache，直接对磁盘做对齐读写，省掉一次内存拷贝，但要求对齐且无系统缓存兜底。

MySQL 里由 **`innodb_flush_method`** 决定数据文件走哪条路（默认 `fsync`，即 Buffered），而 redo log、doublewrite、sort 临时文件又各有独立的控制开关。这篇文章把这条主线讲透。

## 理论基础：Linux 的三种 I/O 方式

### 1. Buffered I/O：write 到 page cache，fsync 才落盘

这是**默认**、也是最容易被误解的方式。关键点：**`write()` 成功 ≠ 数据落盘**。

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

源码（os0file.cc:5285-5286）：

```cpp
#elif defined(O_DIRECT)
  if (fcntl(fd, F_SETFL, O_DIRECT) == -1) { ... }  // 失败则告警后回退到 buffered
```

### 3. O_DSYNC / O_SYNC：每次 write 同步

打开文件时带 `O_DSYNC`，每次 `write()` 相当于"write + 落盘"原子完成，**不需要额外 fsync**。适合 redo log 这类"写了就要持久"的场景，省掉一次系统调用。缺点是不能利用 OS 写合并，每次写都落盘。

三者对比：

| 方式 | 经过 page cache | 持久化时机 | 系统调用 | 典型用途 |
|------|----------------|-----------|---------|---------|
| Buffered + fsync | 是 | fsync 时 | write + fsync | 数据文件（默认） |
| O_DIRECT | 否 | 每次写直接落盘 | 对齐的 pread/pwrite | 数据文件（flush_method=O_DIRECT） |
| O_DSYNC | 是（但 write 即刷） | 每次 write | 单次 write | redo log（flush_method=O_DSYNC） |

## 核心实现：innodb_flush_method

### 枚举值与语义

`innodb_flush_method`（ha_innodb.cc:22119，只读变量）六个取值，映射到 `srv_unix_flush_t`（srv0srv.h:876-890）：

| 取值 | 数据文件 | redo log |
|------|---------|---------|
| `fsync`（默认） | Buffered + `fsync()` | Buffered + `fsync()` |
| `O_DSYNC` | Buffered + `fsync()` | `O_DSYNC` 打开，**write 即同步，免 fsync** |
| `littlesync` | 写后不 fsync | 写后 fsync |
| `nosync` | 不 fsync | 不 fsync（**危险**，仅测试用） |
| `O_DIRECT` | **O_DIRECT 绕过缓存** + 仍 `fsync()` | Buffered + `fsync()` |
| `O_DIRECT_NO_FSYNC` | **O_DIRECT** + 不 fsync | Buffered + `fsync()` |

**关键点**：`O_DIRECT` 系列**只影响数据文件**，redo log 的 fsync 不受影响（redo 独立由 `innodb_flush_log_at_trx_commit` 控制）。`O_DIRECT` 与 `O_DIRECT_NO_FSYNC` 的区别在于——数据文件用 O_DIRECT 后，`fsync()` 是否还需要（因为 O_DIRECT 绕过了 page cache，很多场景 fsync 意义变小，但文件元数据/目录项可能仍需刷）。

### 数据文件的 O_DIRECT 判定

数据文件到底用不用 O_DIRECT，源码在 os0file.cc:3207-3216，只对**数据类文件**调用：

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

### 双重缓存（double buffering）问题

默认 `fsync` 模式下，InnoDB 数据页在内存里**存了两份**：

```
读：磁盘 ──▶ OS page cache ──▶ Buffer Pool（16KB 数据页）
写：Buffer Pool ──▶ OS page cache ──▶ 磁盘
```

一份在 OS page cache（内核），一份在 InnoDB Buffer Pool（用户态）。同一页占两份内存，且写放大。**`O_DIRECT` 的目的就是消除这个重叠**——数据文件绕过 page cache，只进 Buffer Pool。

这也是为什么 `innodb_dedicated_server=ON` 时（ha_innodb.cc:4994-5013），若用户没显式指定 flush_method，会自动设成 `O_DIRECT_NO_FSYNC`——专属服务器上内存更该留给 Buffer Pool，而非被 page cache 重复占用。

## 各文件的 I/O 路径

InnoDB 里不同文件的 I/O 走不同路径，对应不同的持久性要求：

| 文件 | 缓存方式 | 持久化控制 | 源码 |
|------|---------|-----------|------|
| 数据文件（.ibd） | Buffered / O_DIRECT（flush_method） | 后台刷脏 + doublewrite | os0file.cc:3207 |
| redo log | Buffered + fsync（或 O_DSYNC） | `innodb_flush_log_at_trx_commit` | ha_innodb.cc:22112 |
| doublewrite | 随 flush_method（O_DIRECT 时也绕过） | 每次刷脏前先写 dblwr | buf0dblwr.cc |
| sort 临时文件 | Buffered / O_DIRECT（独立开关） | 无需持久（DDL 失败即弃） | ddl0ddl.cc:174 |
| binlog | Buffered + fsync | `sync_binlog` | server 层 |

### redo log 的 fsync：innodb_flush_log_at_trx_commit

redo 的刷盘频率由 `innodb_flush_log_at_trx_commit`（ha_innodb.cc:22112-22115）控制：

- **0**：每秒 write + flush 一次（断电丢最近 1s 的事务）
- **1**（默认）：**每次提交 write + fsync**（最安全，性能最差）
- **2**：每次提交 write，每秒 flush 一次（write 到 page cache，靠 OS 兜底）

redo 的 fsync 受 flush_method 影响（log0log.cc:1716-1718）：当 flush_method 为 `O_DSYNC` / `NOSYNC` 时，`s_skip_fsyncs=true`，因为 `O_DSYNC` 打开的文件 write 即同步，无需再 fsync；`log0write.cc:2431` 的 `do_flush = srv_unix_file_flush_method != SRV_UNIX_O_DSYNC` 同义。

### fsync vs fdatasync

`os_file_fsync_posix`（os0file.cc:2815-2840）封装了系统调用：

```cpp
const auto ret = srv_use_fdatasync ? fdatasync(file) : fsync(file);
```

- `fsync`：刷**数据 + 文件元数据**（大小、时间戳等）；
- `fdatasync`：只刷**数据**（和必要的元数据），省掉元数据写入，**更快**。

由 `innodb_use_fdatasync` 控制（默认 false，用 fsync）。对 redo 这类"大小固定、只追加数据"的文件，fdatasync 更合适；对数据文件（会改大小）则 fsync 更稳妥。

### sort 临时文件的独立开关

DDL 排序的临时文件（`ibXXXXXX`）**不是数据页，不进 Buffer Pool**（没有 `space_id`/`page_no`），它唯一的缓存是 OS page cache。`innodb_disable_sort_file_cache`（默认 OFF）决定它是否用 O_DIRECT 绕过（ddl0ddl.cc:174-178）：

- OFF：sort file 走 page cache（反复读命中快），但大排序会污染 OS cache 挤热页；
- ON：O_DIRECT 绕过（避免污染，但排序变慢）。

详见 ddl.md「并行 DDL」。

## 系统变量速查

| 变量 | 默认 | 作用 |
|------|------|------|
| `innodb_flush_method` | `fsync` | 数据文件 + redo 的 I/O 方式（只读） |
| `innodb_flush_log_at_trx_commit` | `1` | redo 的 write/fsync 频率（0/1/2） |
| `innodb_use_fdatasync` | `OFF` | fsync 换成 fdatasync |
| `innodb_disable_sort_file_cache` | `OFF` | sort 临时文件是否 O_DIRECT |
| `innodb_dedicated_server` | `OFF` | 开启时自动设 O_DIRECT_NO_FSYNC |
| `sync_binlog` | `1` | binlog 的 fsync 频率（server 层） |

## 选择建议

| 场景 | 建议 |
|------|------|
| 通用生产（内存充足） | 保持默认 `fsync` + `flush_log_at_trx_commit=1` |
| 专属服务器 / 内存紧张、想消除 double buffering | `innodb_flush_method=O_DIRECT`（或 O_DIRECT_NO_FSYNC，但需确认文件系统支持） |
| 极致 redo 性能 | `O_DSYNC`（免 fsync 系统调用）+ `innodb_use_fdatasync=ON` |
| 大表 DDL 且内存紧张 | `innodb_disable_sort_file_cache=ON`（避免 sort file 污染 OS cache） |
| **tmpfs 上的文件** | **不能用 O_DIRECT**（EINVAL，会自动告警回退） |

> 核心权衡一句话：**Buffered 快但双重缓存 + 需 fsync；O_DIRECT 省内存但要求对齐、无系统缓存兜底**。数据文件选谁看 `innodb_flush_method`，redo 看 `innodb_flush_log_at_trx_commit`，sort 临时文件看 `innodb_disable_sort_file_cache`，三者独立，别混。

## 参考

- MySQL 8.0.39 源码：`os/os0file.cc`（`os_file_set_nocache`、`os_file_fsync_posix`）、`srv0srv.h`（`srv_unix_flush_t`）、`ha_innodb.cc`（`innodb_flush_method`、`innodb_flush_log_at_trx_commit`）、`log/log0log.cc` / `log0write.cc`（redo fsync）
- 关联文档：`innodb/ddl.md`（sort 临时文件与 `innodb_disable_sort_file_cache`）、`innodb/buffer_pool.md`
