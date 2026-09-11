# Buffer Pool 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 页面控制块状态机、LRU/flush list、刷脏（page cleaner）、doublewrite。

## 目录

- [概述](#概述)
- [slot-based 结构：frame 与控制块](#slot-based-结构frame-与控制块)
- [理论基础](#理论基础)
- [buf_page_t 状态机与 buf_page_in_file](#buf_page_t-状态机与-buf_page_in_file)
- [脏页刷新（Flush）](#脏页刷新flush)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

InnoDB 的缓冲池，缓存表空间数据页（16KB 为主），控制块为 `buf_page_t` / `buf_block_t`。

### 用途

减少磁盘 I/O：读缓存命中、写合并（脏页延迟刷盘 + redo WAL）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 多 buffer pool instance |
| 5.7 | 在线 resize（chunk） |
| 8.0 | flush 无锁化改造（8.0.19 起 flush_list_mutex 替代 buf_pool mutex 扫 flush_list）；page cleaner 协调者/worker 模型 |

---

## slot-based 结构：frame 与控制块

InnoDB buffer pool 是经典的 slot-based（frame-based）buffer manager，术语源自 Gray & Reuter《Transaction Processing: Concepts and Techniques》（buf0buf.cc:136 注释明确引用）。

核心思想：预分配一大块内存，切成 N 个固定大小槽位（frame，每个 = `UNIV_PAGE_SIZE`），每个槽位配一个控制块记录该槽位当前装哪一页、脏否、被谁 pin。

三要素与 InnoDB 对应：

| 通用概念 | InnoDB 实现 |
|---------|------------|
| 槽位 frame | `buf_block_t::frame`（`byte*`，buf0buf.h:1710） |
| 控制块 | `buf_block_t`（内含 `buf_page_t page` + `BPageLock lock` + `frame`） |
| 页表 page table | `buf_pool_t::page_hash`（按 `(space_id, page_no)` 索引，buf0buf.h:2354） |
| 空闲链表 | `free_list` |
| 替换策略 | LRU list（old/new 两段、midpoint insertion） |

内存布局（`buf_chunk_init` buf0buf.cc:993）：一次分配一整块 chunk，头部放 `buf_block_t` 控制块数组（`chunk->blocks`），后面紧跟 frame 数组；循环 `buf_block_init(buf_pool, block, frame)`（:748，`block->frame = frame`）后 `block++`、`frame += UNIV_PAGE_SIZE`，两者一一对应连续排列。

`buf_block_t` 第一个成员必须是 `buf_page_t page`（buf0buf.h:1700），让 `page_hash` 能统一指向 `buf_page_t`（裸压缩页）或 `buf_block_t`（有 frame 的页）。

为什么叫 slot-based：槽位是固定页大小、按需动态装载任意页（页号与槽位非固定绑定，靠 page_hash 解耦），对比"固定映射式"（页号直接映射内存位置，无页表）和"对象/变长式"（malloc 级粒度）。CMU 15-445 BusTub 是最简教学实现（`pages[]` + `page_table_` + `replacer_` + `free_list_`）。

InnoDB 在经典骨架上扩展：chunk 分区（在线 resize）、多实例（`innodb_buffer_pool_instances`，每实例独立 chunk/LRU/page_hash）、压缩页突破固定槽位（`ZIP_PAGE/ZIP_DIRTY` 无 frame，压缩数据用 buddy allocator 动态切，`zip_hash`/`zip_free`）、控制块与 frame 分离但同块紧邻（不污染页数据）。

---

## 理论基础

### 设计模式

- 状态机模式：`buf_page_t::state` 是控制块单一事实来源，`buf_page_in_file()` 把 8 态折叠为"有/无磁盘身份"两个互斥类别。
- Design by Contract：全库 60+ 处 `ut_a`/`ut_ad(buf_page_in_file(...))` 做不变量断言，release 版 `ut_ad` 编译掉零成本。
- 汇聚点（统一出口）：三条刷脏路径全部汇聚到 `buf_flush_write_block_low()`，观测/埋点只需一处。
- 生产者-消费者：page cleaner coordinator 计算刷脏量并派发 slot，worker 线程消费。

### 算法与数据结构

- `buf_page_t page` 是 `buf_block_t` 第一个成员（buf0buf.h:1700），FILE_PAGE 态 bpage 可直接 cast 为 block（`buf_page_get_block` buf0buf.ic:591）。

### 历史背景

（待补充）

---

## buf_page_t 状态机与 buf_page_in_file

### 语义

`buf_page_in_file(bpage)`（buf0buf.ic:346，声明 buf0buf.h:743 "Determines if a block is mapped to a tablespace"）：控制块当前是否映射到表空间页，即有合法 `(space_id, page_no)` 磁盘身份、内容对应数据文件一页。是 `state` 字段的规范解释者（buf0buf.h:1532 `@see buf_page_in_file`）。

### 8 个状态的两类划分（buf0buf.h:129-152）

| in_file = true | in_file = false |
|---|---|
| `BUF_BLOCK_FILE_PAGE` 普通文件页（含解压帧） | `BUF_BLOCK_NOT_USED` free list |
| `BUF_BLOCK_ZIP_PAGE` 仅压缩页、干净（zip_clean） | `BUF_BLOCK_READY_FOR_USE` 刚从 free list 拿到未装页 |
| `BUF_BLOCK_ZIP_DIRTY` 仅压缩页、脏（flush_list） | `BUF_BLOCK_MEMORY` 纯内存对象（释放过渡态，buf0lru.cc:2322） |
| | `BUF_BLOCK_REMOVE_HASH` 正在释放、需先摘 AHI 的过渡态（buf0lru.cc:2265） |

- `BUF_BLOCK_POOL_WATCH`：watch[] 哨兵，`buf_page_in_file` 中直接 `ut_error`，出现即 bug。
- 只有 in_file 态，`id`、`oldest_modification`/`newest_modification`、flush_list/zip_clean 的 `list` 节点才有意义。
- 唯一被容忍的例外：`REMOVE_HASH` 瞬态可进 `buf_flush_ready_for_flush_common`（buf0flu.cc:722-728），debug 版要求此时不持 `LRU_list_mutex`；BUF_FLUSH_LIST 类型下 REMOVE_HASH 不可刷（buf0flu.cc:756）。

### 生命周期转换

```
NOT_USED ──buf_LRU_get_free_block──> READY_FOR_USE
  ──buf_page_init_for_read/buf_page_create──> FILE_PAGE
FILE_PAGE ──buf_LRU_free_page──> (有 AHI 先 REMOVE_HASH) ──> MEMORY ──> NOT_USED
```

- 反向断言：从 free list 取块必须 `!buf_page_in_file`（buf0lru.cc:1218）。
- ZIP_PAGE/ZIP_DIRTY 是 ROW_FORMAT=COMPRESSED 的"仅压缩副本"形态。
- LRU/flush_list 遍历中发现非 in_file 直接 `ib::fatal`（buf0flu.cc:691）。

### flush 主调用链（page cleaner）

```
buf_flush_page_cleaner_thread   (buf0flu.cc:3606, worker 入口；协调者 buf_flush_page_coordinator_thread:3183)
└─ pc_flush_slot                (buf0flu.cc:2934)
   └─ buf_flush_do_batch(BUF_FLUSH_LIST) → buf_flush_batch (:1951)
      └─ buf_do_flush_list_batch        (:1881)
         └─ buf_flush_page_and_try_neighbors (:1626)
            └─ buf_flush_try_neighbors  (:1482)
               └─ buf_flush_page        (:1291)
                  └─ buf_flush_write_block_low (:1174 → dblwr/写盘)
```

每一层都有 `ut_ad/ut_a(buf_page_in_file(bpage))`。

---

## 刷脏前的 WAL：redo 先落盘

`buf_flush_write_block_low`（buf0flu.cc:1174）在写数据页前，先同步等 redo fsync（:1205-1221）：

```c
const lsn_t flush_to_lsn = bpage->get_newest_lsn();  // = newest_modification
if (log_sys->flushed_to_disk_lsn.load() < flush_to_lsn) {
    wait_stats = log_write_up_to(*log_sys, flush_to_lsn, true);  // true=同步等待 fsync
}
```

- **WAL/ARIES 规则**：数据页新版本写盘前，对应 redo 必须先落盘，否则崩溃恢复时数据文件有新版本但 redo 没落盘，无法重放/回滚。
- **用 newest_modification 而非 oldest**：要保证该页**所有** redo 修改（首次变脏到最近修改）都落盘，水位必须推到 newest（最大 LSN）。`oldest_modification` 是首次变脏 LSN（进 flush_list 依据），`newest_modification` 是最近修改 LSN。
- 优化：先查 `flushed_to_disk_lsn < flush_to_lsn`，redo 已 flush 到更新位置就不调 `log_write_up_to`（避免无谓等待 + 污染 log 线程调用计数器，注释 :1209-1212）。
- `log_write_up_to(..., true)` 的 true = wait，同步等待 fsync 完成。

---

## 压缩页（ROW_FORMAT=COMPRESSED）在 buffer pool 的三态与刷盘

同一个 COMPRESSED 页在 buffer pool 里有三种存在形态：

| 状态 | zip.data | block->frame | 脏? | 在哪个 list |
|---|---|---|---|---|
| `BUF_BLOCK_ZIP_PAGE` | 压缩副本 | 无（解压帧已淘汰） | 干净 | zip_clean |
| `BUF_BLOCK_ZIP_DIRTY` | 压缩副本 | 无 | 脏 | flush_list |
| `BUF_BLOCK_FILE_PAGE` | 压缩副本 + 解压帧 | 有 | 脏/干净 | LRU (+flush_list 若脏) |

- `ZIP_PAGE`/`ZIP_DIRTY` 的 bpage 是裸 `buf_page_t`（不是 `buf_block_t`，无 frame 字段）。
- `FILE_PAGE` 形态即"压缩页解压后的版本"：block 同时持有压缩副本 `zip.data` 和解压帧 `frame`（双份，省重复解压）。

### 刷盘分支（buf_flush_write_block_low :1223-1257）

- **BUF_BLOCK_ZIP_DIRTY**（:1232）：直接对 `bpage->zip.data` 写 `FIL_PAGE_LSN` = newest_modification + `verify_zip_checksum`（只校验不重算）。不调 `buf_flush_init_for_writing`，因为压缩数据在修改时已由 `page_zip_*` 增量维护好，无需重压缩/重算 checksum。
- **BUF_BLOCK_FILE_PAGE**（:1243）：`frame = zip.data`（优先压缩版省 I/O），无压缩副本才用 `block->frame`；调 `buf_flush_init_for_writing`（:998）按页类型更新 checksum——压缩页走 `buf_flush_update_zip_checksum`（:1044）只更新 `zip.data` 的 checksum 就 return，**不重新压缩**。

### 压缩数据维护机制

压缩页的 `zip.data` 在页被修改时由 `page_zip_write_rec` / `page_zip_write_header` 等函数**增量更新**（不是每次全页重压缩），刷盘时 `zip.data` 已是最新成品。

---

## 脏页刷新（Flush）

所有脏页写盘汇聚到 `buf_flush_page()` → `buf_flush_write_block_low()`（buf0flu.cc:1174，统一汇聚点）→ `dblwr::write()`（doublewrite）→ `fil_io` → AIO 提交；IO 完成由 io handler 线程回调 `buf_page_io_complete()`（buf0buf.cc:5608）。汇聚点设计使观测/埋点只需一处。

刷脏类型 `buf_flush_t`（buf0types.h:68）：

| 值 | 类型 | 发起者 | IO |
|---|---|---|---|
| 0 | `BUF_FLUSH_LRU` | page cleaner / `buf_LRU_get_free_block` | async |
| 1 | `BUF_FLUSH_LIST` | page cleaner（主路径） | async |
| 2 | `BUF_FLUSH_SINGLE_PAGE` | 用户线程（free page 不足兜底） | sync |

`sync=true` 只允许出现在 SINGLE_PAGE（断言 buf0flu.cc:1292）。写页前先保证 newest_lsn 之前的 redo 已落盘（WAL，见上文"刷脏前的 WAL"章节）。

### 路径1：flush list 批刷（async, type=1，主路径）

```
buf_flush_page_coordinator_thread (buf0flu.cc:3183)   ← 算刷脏量、派发 slot、唤醒 worker
└─ buf_flush_page_cleaner_thread (buf0flu.cc:3606)    ← worker 线程（coordinator 兼任 worker 0）
   └─ pc_flush_slot (:2934)
      └─ buf_flush_do_batch (:2074) → buf_flush_batch (:1951)
         └─ buf_do_flush_list_batch (:1881)
            └─ buf_flush_page_and_try_neighbors (:1626)  ← 邻居页合并（innodb_flush_neighbors）
               └─ buf_flush_try_neighbors (:1482)
                  └─ buf_flush_page (:1277)  ← 置 IO_FIX(BUF_IO_WRITE)、SX lock、flush_state_mutex 统计
                     └─ buf_flush_write_block_low (:1174)  ← ★ 汇聚点（WAL 约束 + checksum + dblwr）
```

### 路径2：LRU 批刷（async, type=0）

coordinator 周期性 + `buf_LRU_get_free_block` 触发 → `buf_flush_LRU_list_batch`（buf0flu.cc:1762）→ 后半段同路径1。

### 路径3：用户线程单页同步刷（sync, type=2）

```
用户线程: buf_LRU_get_free_block 找不到 free page
└─ buf_flush_single_page_from_LRU (buf0flu.cc:2159)
   └─ buf_flush_page(..., BUF_FLUSH_SINGLE_PAGE, sync=true)  ← 持 LRU_list_mutex 进入
```

频繁发生 = free page 供给不上（page cleaner 跟不上 / redo 产生过快 / buffer pool 太小），是**性能悬崖信号**，对应状态计数 `Innodb_buffer_pool_wait_free`。

### DBUG 观测与调试

- DBUG 库（源自 Fred Fish dbug 包）条件打印：`DBUG_PRINT(keyword, args)`（my_dbug.h:181）经 `_db_keyword_` 匹配才输出；**仅 debug build 有效**（WITH_DEBUG=1），release build 宏为空（my_dbug.h:255）零开销。
- 开关：
  - 启动：`mysqld --debug='d,ib_buf:t:i:F:N:o,/tmp/mysqld.trace'`（`d`=keyword 列表，`t`=函数进出 trace，`i`=线程号，`F/N`=源文件/行号，`o`=输出文件）
  - 运行期（debug build）：`SET GLOBAL debug='+d,ib_buf';` / `'-d,ib_buf';`，不指定 `o` 时输出到 error log
- 输出行格式 `keyword: message`，如 `T@6 : ib_buf: flush async 1 page 8:331`

`ib_buf` keyword 打印点（页面全生命周期）：

| 位置 | 输出 | 含义 |
|------|------|------|
| buf0buf.cc:5067 | `create page S:P` | 页创建 |
| buf0rea.cc:104+ | read-ahead 系列 | 预读 |
| buf0flu.cc:1537 | `flush S:low..high` | 邻居页合并范围 |
| **buf0flu.cc:1183** | `flush <sync\|async> <type> page S:P` | **每页写盘前（汇聚点）** |
| buf0flu.cc:1980/1992 | `flush N completed` | batch 汇总 |
| buf0lru.cc:873 | `evict page S:P state N` | 驱逐 |
| buf0lru.cc:1801 | `free page S:P` | 释放 |

调试要点：

- `flush sync 2` 高频 → 单页同步刷，free page 不足（性能悬崖）
- 同一 `S:P` 高频 flush → 热点页 / checkpoint 压力
- type=1 应占绝对多数；trace 配 `t` 看函数嵌套确认走哪条路径
- release build 替代：`innodb_metrics` 的 `buffer_flush_*`/`buffer_LRU_batch_*`、`SHOW ENGINE INNODB STATUS` BUFFER POOL 段、gdb 断点、bpftrace

---

## io_fix 状态机与 latch 协议校验（buf0buf.cc:5475-5503）

### 两个正交状态字段

`buf_page_t` 有两个独立状态维度（勿混淆）：
- `state`（buf0buf.h:1533，buf_page_state 8 态）：页身份/生命周期（是不是文件页、压缩与否、在哪个 list）
- `io_fix`（buf0types.h:99，buf_io_fix 4 态）：I/O 进行中状态

io_fix 四态：`BUF_IO_NONE`(无I/O)/`BUF_IO_READ`(异步读中)/`BUF_IO_WRITE`(异步写中)/`BUF_IO_PIN`(钉住防重定位，buf_page_set_sticky buf0buf.ic:480)。

### 声明式 latch 协议校验器

`ut::Stateful_latching_rules<buf_io_fix, 3>`（ut0stateful_latching_rules.h:114）把 io_fix 的状态转换图 + 每条转换要求的 latch 集合声明式写出，debug 版自动校验（设计思想：FSM + 形式化 latch 协议验证，8.0 buffer pool 无锁化重构一环）。

转换图（buf0buf.cc:5496-5501）：
```
NONE ──{0,2}──> READ        发起读
READ ──{0,2}──> NONE        读完成
NONE ──{0,1,2}──> WRITE     发起写
WRITE ──{0,1,2}──> NONE     写完成
NONE ──{0}──> PIN           置 sticky
PIN  ──{0}──> NONE          取消 sticky
```

latch 编号（get_owned_latches buf0buf.cc:5519-5527）：
- **0** = block mutex（buf_page_get_mutex，页级字段）
- **1** = flush_state_mutex（buf0buf.h:2204，保护刷脏批次状态 init_flush/n_flush/no_flush，**非** flush_list 链表）
- **2** = io_responsibility 责任令牌（当前线程是否负责该页 I/O，供 io 完成线程免锁检查）

**写比读多 latch 1**：发起/完成写会动 n_flush 计数与 no_flush 事件，由 flush_state_mutex 保护；读不碰刷脏计数。**PIN 只要 latch 0**：只钉住防重定位，不碰 I/O 也不碰刷脏。

### 校验两类操作

- `on_transition_to`（:5558）：改 io_fix 时校验当前线程持有该转换要求的 latch。
- `assert_latches_let_distinguish`（:5547）：读 io_fix 判断"是 A 还是 B"时，校验持有 latch 足以阻止 io_fix 从 A/B 集合内变到集合外（否则读到的值瞬间失效）。如 `is_io_fix_write()`（:5567）调用它。

---

## Misc

（待补充：LRU 中点插入策略、doublewrite、AHI、buddy allocator）

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/include/buf0buf.ic:346` | `buf_page_in_file` 定义 |
| `storage/innobase/include/buf0buf.h:129` | `buf_page_state` 8 态枚举 |
| `storage/innobase/buf/buf0flu.cc:691` | LRU 遍历非 in_file → fatal |
| `storage/innobase/buf/buf0flu.cc:722` | 刷脏判断容忍 REMOVE_HASH 瞬态 |
| `storage/innobase/buf/buf0lru.cc:2265` | 释放页先置 REMOVE_HASH |
| `storage/innobase/buf/buf0lru.cc:2322` | `buf_LRU_block_free_hashed_page` 置 MEMORY |
| `storage/innobase/buf/buf0flu.cc:2934` | `pc_flush_slot` worker 刷脏入口 |
| `storage/innobase/buf/buf0flu.cc:1205` | 刷脏前 WAL：log_write_up_to 同步等 redo fsync |
| `storage/innobase/buf/buf0flu.cc:1232` | BUF_BLOCK_ZIP_DIRTY 刷盘分支（写 LSN+校验 zip checksum） |
| `storage/innobase/buf/buf0flu.cc:1243` | BUF_BLOCK_FILE_PAGE 刷盘分支（buf_flush_init_for_writing） |
| `storage/innobase/buf/buf0flu.cc:998` | `buf_flush_init_for_writing` 定义 |
| `storage/innobase/include/buf0buf.h:1293` | `get_newest_lsn()` = newest_modification |
| `storage/innobase/buf/buf0flu.cc:1174` | `buf_flush_write_block_low`，刷脏统一汇聚点 |
| `storage/innobase/buf/buf0flu.cc:1183` | `ib_buf` DBUG_PRINT：每页写盘前打印 |
| `storage/innobase/buf/buf0flu.cc:1277` | `buf_flush_page`：置 IO_FIX、SX lock、batch 统计 |
| `storage/innobase/buf/buf0flu.cc:1626` | `buf_flush_page_and_try_neighbors`：邻居页合并 |
| `storage/innobase/buf/buf0flu.cc:1762` | `buf_flush_LRU_list_batch`：LRU 批刷 |
| `storage/innobase/buf/buf0flu.cc:2074` | `buf_flush_do_batch`：flush list 批刷 |
| `storage/innobase/buf/buf0flu.cc:2159` | `buf_flush_single_page_from_LRU`：单页同步刷（性能悬崖信号） |
| `storage/innobase/buf/buf0flu.cc:3183` | `buf_flush_page_coordinator_thread`：coordinator |
| `storage/innobase/buf/buf0buf.cc:5608` | `buf_page_io_complete`：IO 完成回调 |
| `storage/innobase/include/buf0types.h:68` | `buf_flush_t`：LRU=0/LIST=1/SINGLE_PAGE=2 |
| `storage/innobase/include/buf0types.h:99` | `buf_io_fix` 4 态枚举（NONE/READ/WRITE/PIN） |
| `storage/innobase/buf/buf0buf.cc:5475` | io_fix 状态机 + latch 协议校验器（Stateful_latching_rules） |
| `storage/innobase/buf/buf0buf.cc:5519` | `get_owned_latches`：latch 0=block mutex / 1=flush_state_mutex / 2=io_responsibility |
| `storage/innobase/buf/buf0lru.cc:1872` | 压缩页解压帧被淘汰→ZIP_DIRTY/ZIP_PAGE 形态转换 |
| `storage/innobase/include/ut0stateful_latching_rules.h:114` | `Stateful_latching_rules` 模板 |
| `include/my_dbug.h:181` | `DBUG_PRINT` 宏定义（release 版为空 :255） |
