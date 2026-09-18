# InnoDB Undo Log 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 undo 存储体系总览（tablespace → rseg → segment → page → record 五层）、undo tablespace DDL 与页面布局、rollback segment 与 undo segment、undo page 布局、undo record 二进制格式与四类记录、history list 组织、purge 全过程剖析（线程模型/边界/rseg 轮转/单条 rec purge/truncate/限流）、DDL 与 undo（原子 DDL + DDL log）、GTID 持久化与 undo。
>
> 注：本工作区源码含中文研读批注，部分行号与官方 8.0.39 有偏移，引用时以函数名为主。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [存储体系总览（五层结构）](#存储体系总览五层结构)
  - 物理层
    - [Undo Tablespace 与 DDL](#undo-tablespace-与-ddl)
    - [Rollback Segment（rseg）与 Undo Segment](#rollback-segmentrseg与-undo-segment)
    - [Undo Page 布局](#undo-page-布局)
    - [Undo Record 格式与记录类型](#undo-record-格式与记录类型)
  - 写路径
    - [Undo 生成路径](#undo-生成路径)
    - [Undo 与事务回滚](#undo-与事务回滚)
  - 读路径
    - [Undo 与 MVCC](#undo-与-mvcc)
  - 清理与协同
    - [History List](#history-list)
    - [Purge 机制](#purge-机制)
    - [DDL 与 Undo](#ddl-与-undo)
    - [GTID 持久化与 Undo](#gtid-持久化与-undo)
    - [Undo 与 Clone / 备份](#undo-与-clone--备份)
- [Misc](#Misc)
- [关键源码位置速查](#关键源码位置速查)
- [参考](#参考)

---

## 概述

### 是什么

InnoDB 的回滚日志，记录事务对数据修改的"逆操作"信息。每条 undo log record 挂在 undo log segment 中，通过 `DB_ROLL_PTR`（7 字节 roll pointer）串联成记录的旧版本链。它是事务原子性（A，回滚）与隔离性（I，MVCC 一致性读）的载体。

### 用途

- 事务回滚：未提交事务失败时，按 undo log 逆向执行逆操作，恢复数据到事务前状态
- MVCC 一致性读：活跃 read view 通过 `DB_ROLL_PTR` 沿 undo 版本链重建历史版本，实现非锁定读
- Purge：已提交事务产生的 delete-marked 记录与过期 undo log，由 purge 线程异步清理
- 崩溃恢复：未提交事务的 undo 在恢复阶段回滚（与 redo 重放配合）
- GTID 持久化：commit 阶段将 GTID 写入 undo header，随 redo 持久化，后台线程异步刷 `mysql.gtid_executed` 表

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 回滚段固定在系统表空间；undo log 与数据页混在 ibdata |
| 5.7 | 引入独立 undo tablespace（`innodb_undo_tablespaces`），undo 可与数据分离 |
| 8.0 | 默认独立 undo tablespace（隐式 2 个）；`innodb_undo_log_truncate` 自动 truncate；原子 DDL 引入 `mysql.innodb_ddl_log` 表；rseg 目录改用每表空间独立的 RSEG_ARRAY 页（page 3） |
| 8.0.30 | undo tablespace 重新设计为固定 2 个隐式表空间，truncate 机制优化 |

---

## 理论基础

### 设计模式

- **undo/redo 分离**：undo 记录"如何撤销"，redo 记录"如何重做"。InnoDB 对页面修改同时产生 redo（前向）与 undo（逆向），符合 ARIES 的 do-undo-redo 模型
- **版本链（version chain）**：每条聚簇索引记录的 `DB_ROLL_PTR` 指向最近一次修改的 undo record，undo record 内嵌旧 `DB_ROLL_PTR`，形成链表，沿链可重建任意历史版本
- **生产者-消费者（purge）**：事务 commit 产生 undo（生产者），purge 线程异步清理（消费者），靠 history list 与 purge view 衔接
- **延迟物化（DDL log）**：DDL 的物理操作先"记账"到 `innodb_ddl_log`，提交后才 replay 执行，使不可用 undo 回滚的物理操作获得事务语义

### 相关论文

- **ARIES**（Mohan et al., 1992）：do-undo-redo 恢复模型。undo log 用于事务回滚与崩溃恢复回滚，redo log 用于前滚。InnoDB 遵循此模型但做了工程化简化

- **Gray & Reuter《Transaction Processing: Concepts and Techniques》(1993)**：多版本并发控制的经典表述。InnoDB 的"版本链 + 提交序号定序"是该书 MVCC 模型的工程实现，区别是 InnoDB 把旧版本外置到独立的 undo 空间，而非留在表内。
- **Bernstein, Hadzilacos, Goodman《Concurrency Control and Recovery in Database Systems》(1987)**：版本链与快照读的正确性论证。落点是 `row_vers_old_has_index_entry` 与 purge view 配合保证二级索引条目不被误删。

### 算法与数据结构

- **space_id_bank**：undo space_id 预分配表（127 槽 × 40 万 ID），truncate 时换 ID 而非复用
- **小顶堆（purge_queue）**：按 `trx_no` 排序聚合跨 rseg 的 purge 顺序
- **file-based list（fut0lst）**：history list 与 undo page list 都用 `flst_base_node_t` + `flst_node_t` 的物理链表
- **变长整数编码**：`mach_write_compressed`（1-5B）/ `mach_u64_write_much_compressed`（1-11B）压缩 undo_no、table_id

复杂度一览：

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| rseg 选择 | O(1) | 全局游标 round-robin，跳过非 active 空间 |
| undo slot 分配 | O(1024) | `trx_rsegf_undo_find_free` 线性扫描第一个 `FIL_NULL`；但命中 cached 链表时是 O(1) |
| history list 入链 | O(1) | `flst_add_first` 头插 |
| purge 取下一个 rseg | O(log n) | `purge_queue` 小顶堆 |
| purge 消费一个 log | O(1) | 沿 `last_page_no` 游标 + 链表 prev |
| 一致性读回溯一跳 | O(1) + 可能的磁盘 I/O | 单跳固定开销，但链长是 O(n) 次单跳 |

选择这些结构的动因：history 入链在 **commit 关键路径**上，必须 O(1)（所以头插）；而 purge 是后台，可以容忍 O(log n) 的堆操作。这是一处典型的"把成本从关键路径挪到后台"的取舍。

### 类似实现对比

四家数据库的"旧版本放哪、谁来清"各不相同，这决定了它们各自的运维痛点：

| 系统 | 旧版本存放 | 清理者 | 膨胀体现在 | 读老版本的代价 |
|------|-----------|--------|-----------|--------------|
| **InnoDB** | 独立 undo 表空间（undo log） | purge 线程 | **undo 表空间**；表本身不膨胀 | 要沿版本链回溯，可能触发随机 I/O |
| **PostgreSQL** | **留在堆表内**（dead tuple） | VACUUM | **表本身**膨胀（bloat），需要 VACUUM/ vacuum full | 旧版本就在本页或附近，无需外链，但可见性判断要查 CLOG |
| **Oracle** | 独立 undo 表空间 | undo 表空间自动管理（受 `UNDO_RETENTION` 约束） | undo 表空间；可精确控制保留时长 | 一致性读直接在 undo 里取前镜像，且支持闪回 |
| **SQL Server** | tempdb 的 version store | 后台清理（依赖最早活跃事务） | **tempdb** | 快照隔离/RCSI 下从 tempdb 读 |

关键差异：InnoDB 与 Oracle 把版本外置，**表本身干净**；PostgreSQL 把版本留在表内，**表会膨胀但读链短**；SQL Server 用全局 tempdb，代价是所有库共享一个瓶颈。

InnoDB 位置的代价是：长链查询慢（外链随机 I/O）、history 堆积会拖慢 purge 进而限流 DML。收益是主表不受历史版本影响，`OPTIMIZE TABLE` 不会像 PG 的 `VACUUM FULL` 那样成为常规运维项。

另需注意：Oracle 的 `UNDO_RETENTION` 是**主动保留**（为了一致性读和闪回），InnoDB 是**尽快清理**（purge 越快越好）——所以 InnoDB 没有"闪回查询"这类能力，但也不容易因为保留策略不当而撑爆 undo。

> 别家分支的私有特性（如 Percona 的某些 undo 优化）不在此列，本篇只讨论社区版行为。

### 历史背景

**为什么从系统表空间搬出来**：5.6 及更早，回滚段固定在 ibdata1，undo 与数据字典、插入缓冲混存。直接后果是——**一个大事务就能把 ibdata1 撑到几十上百 GB，而且永远无法回收**（ibdata 不支持收缩，只能 mysqldump 后重建实例）。这是运维上最痛的问题之一。5.7 引入独立 undo tablespace 就是为了把这块"易涨的空间"隔离出去，让它可被单独管理。

**为什么 8.0 又把 rseg 目录从 TRX_SYS 页下沉到各表空间的 RSEG_ARRAY 页**：TRX_SYS 页在系统表空间里，它的槽是 8 字节（space_id + page_no），因为它要跨表空间指向。下沉后，每个 undo 表空间有自己的目录页，槽只需 4 字节存 page_no。这个改动看着是省空间，真正的动因是**让 undo 表空间自包含**——只有自包含，才能安全地 truncate、在线增删、以及把某个表空间设为 inactive，而不影响其他部分。

**为什么 8.0 要引入自动 truncate**：独立出来只是能管理，不解决"大事务撑大后回收"的问题。8.0 的 `innodb_undo_log_truncate` 让 undo 表空间超过阈值时自动重建。这里的权衡是——**truncate 是换 space_id 重建而非原地清空**（见「设计思想与权衡」第七条），代价是 space_id 成为消耗品，收益是旧 space_id 上的并发引用可以自然消亡，不需要等待或强制驱逐。

**为什么 DDL 需要 DDL log**：8.0 之前 DROP TABLE 中途崩溃会留下"数据字典已删、.ibd 文件还在"这类残骸。原子 DDL 的动因就是消除这种中间态，而 undo 无法表达"删文件"的逆操作，于是引入"记账 + 延迟 replay"这一层。

---

## 核心实现

### 存储体系总览（五层结构）

undo 的物理组织是严格分层的，理解这个层级是理解后面所有章节的基础：

```
undo tablespace（.ibu 文件，space_id ∈ [s_min_undo_space_id, s_max_undo_space_id]）
  └─ RSEG_ARRAY 页（page 3）→ rseg 目录，每槽 4B 存 rseg header page_no
       └─ rollback segment（rseg，trx_rseg_t）
            ├─ rseg header page：history list base node + undo slot 数组（1024 槽 × 4B）
            └─ undo log segment（trx_undo_t，由 undo slot 指向）
                 ├─ segment 首页：SEG_HDR + 1..N 个 LOG_HDR + 记录区
                 └─ 后续页（undo page list）：PAGE_HDR + 记录区
                      └─ undo record（trx_undo_rec_t）
```

关键数量关系：

| 层级 | 数量上限 | 来源 |
|------|---------|------|
| undo tablespace | 127（`FSP_MAX_UNDO_TABLESPACES`） | `fsp0types.h` |
| 每 tablespace 的 rseg | 128（`TRX_SYS_N_RSEGS`） | `fsp0types.h` |
| 每 rseg 的 undo slot | 1024（`UNIV_PAGE_SIZE/16`） | `trx0rseg.h` |
| 每 rseg 理论并发事务 | 512（`TRX_RSEG_MAX_N_TRXS`） | `trx0rseg.h` |

---

### Undo Tablespace 与 DDL

### CREATE / ALTER / DROP UNDO TABLESPACE 路径

Server 层入口在 `sql/sql_tablespace.cc`：`Sql_cmd_create_undo_tablespace::execute()`先做权限校验、名字校验、MDL X 锁，然后**先建 DD 对象**（`dd::create_object<dd::Tablespace>`），再组装 `st_alter_tablespace` 下推 SE：

```
// sql/sql_tablespace.cc
st_alter_tablespace ts_info{m_undo_tablespace_name.str, nullptr,
                            CREATE_UNDO_TABLESPACE, ..., m_data_file_name.str, ...};
int ha_error = hton->alter_tablespace(hton, thd, &ts_info, tsmp.first, tsmp.second);
```

InnoDB 层统一分派在 `innobase_alter_tablespace()`，按 `ts_cmd_type` 三分：

| 命令 | 处理函数 |
|------|---------|
| CREATE_UNDO_TABLESPACE | `innodb_create_undo_tablespace` |
| ALTER_UNDO_TABLESPACE | `innodb_alter_undo_tablespace` |
| DROP_UNDO_TABLESPACE | `innodb_drop_undo_tablespace` |

CREATE 的核心步骤：

```cpp
  mutex_enter(&undo::ddl_mutex);              // 串行化所有 undo DDL
  space_id_t space_id = undo::get_next_available_space_num();  // 分配 space_id
  if (space_id == SPACE_UNKNOWN || undo::spaces->size() == FSP_MAX_UNDO_TABLESPACES)
    ib::error(ER_IB_MSG_MAX_UNDO_SPACES_REACHED, ...);         // 最多 127
  err = srv_undo_tablespace_create(alter_info->tablespace_name,
                                        alter_info->data_file_name, space_id);
  dd_write_tablespace(dd_space, space_id, flags, DD_SPACE_STATE_ACTIVE);
  undo::set_active(space_id);                 // 立刻置 ACTIVE
```

ALTER 支持 `SET ACTIVE` / `SET INACTIVE`。INACTIVE 的约束（ha_innodb.cc）：至少保留 2 个 active 空间；且 `fil_count_undo_deleted` 超 `CONCURRENT_UNDO_TRUNCATE_LIMIT`（50000）时拒绝；最后 `srv_wake_purge_thread_if_not_active()` 唤醒 purge 去 truncate。

DROP 的约束（ha_innodb.cc）：必须是非隐式（num > 2）且已 empty，否则报 `HA_ERR_TABLESPACE_IS_NOT_EMPTY`。

### undo::spaces 组织与 space_id 分配

注意：`undo::Tablespaces` / `undo::Tablespace` **不在** trx0sys.h，而在 `storage/innobase/include/trx0purge.h` / `:314-662`；全局对象 `undo::spaces` 定义在 trx0purge.cc。

`undo::Tablespace` 关键成员（trx0purge.h）：`m_id`（space_id）、`m_num`（undo 编号 1..127，即 roll pointer 里的 7-bit）、`m_implicit`、`m_space_name`、`m_file_name`、`m_rsegs`。

隐式 vs 显式靠**文件名后缀**判定（trx0purge.cc）：显式用 `.ibu` 扩展名。

```
// trx0purge.cc
/* Explicit undo tablespaces use an IBU extension. */
m_implicit = (Fil_path::has_suffix(IBU, tmp_fn) ? false : true);
```

space_id 分配区间（dict0dict.h）：

```
s_log_space_id         = 0xFFFFFFF0
s_undo_space_id_range  = 400000        // 每个 undo 编号 40 万个 ID
s_min_undo_space_id    = s_log_space_id - (127 * 400000)
s_max_undo_space_id    = s_log_space_id - 1 = 0xFFFFFFEF
```

`num2id` / `id2num`（trx0purge.h）在编号与 ID 间换算。truncate 时从同一 num 的 ID 区间取下一个（`next_space_id`，trx0purge.cc），因此一个 undo 编号可累积约 40 万个历史 space_id——这就是 `CONCURRENT_UNDO_TRUNCATE_LIMIT=50000` 限制旧 space_id 残留页数的原因。

`Rsegs` 五态状态机（trx0types.h）：`INIT / ACTIVE / INACTIVE_IMPLICIT / INACTIVE_EXPLICIT / EMPTY`。

### undo tablespace 页面布局

8.0 的 undo 表空间前几页固定（fsp0types.h）：

| 页号 | 用途 | 类型常量 |
|------|------|---------|
| 0 | FSP header（含 XDES 数组） | `FIL_PAGE_TYPE_FSP_HDR`(8) |
| 1 | insert buffer bitmap | — |
| 2 | 第一个 INODE 段信息页（懒分配） | `FIL_PAGE_INODE` |
| **3** | **RSEG_ARRAY（rseg 目录，undo 独有）** | `FIL_PAGE_TYPE_RSEG_ARRAY`(21) |
| 4+ | rseg header 页（SYS）+ undo log 页 | `FIL_PAGE_TYPE_SYS`(6) / `FIL_PAGE_UNDO_LOG` |

RSEG_ARRAY 页布局（trx0rseg.h）：

```cpp
 RSEG_ARRAY_HEADER = FSEG_PAGE_DATA;              // = FIL_PAGE_DATA = 38
 RSEG_ARRAY_VERSION = 0x52534547 + 1;             // 'RSEG' 派生，用于校验
 RSEG_ARRAY_VERSION_OFFSET = 0;                   // 4B 版本号
 RSEG_ARRAY_SIZE_OFFSET = 4;                      // 4B 当前跟踪的 rseg 数
 RSEG_ARRAY_FSEG_HEADER_OFFSET = 8;               // 10B 本页所属段 inode
 RSEG_ARRAY_PAGES_OFFSET = 8 + FSEG_HEADER_SIZE;  // ★ rseg 槽数组起点
 RSEG_ARRAY_RESERVED_BYTES = 200;                 // 页尾保留
 RSEG_ARRAY_SLOT_SIZE = 4;                        // 每槽 4B = rseg header page_no
```

槽位存放方式：`RSEG_ARRAY_PAGES_OFFSET + slot * 4`，每槽 4 字节存该 rseg 的 header page_no，空槽填 `FIL_NULL(0xFFFFFFFF)`。slot 编号 == rseg id。读写 helper 在 `trx0rseg.ic`。

对比系统表空间的 TRX_SYS 页（page 5）：它的 rseg 槽是 **8 字节**（space_id 4B + page_no 4B，`trx0sys.ic`），因为要跨表空间；而 RSEG_ARRAY 只需 4 字节 page_no（整页属同一 space）。8.0 独立 undo 表空间一律用 RSEG_ARRAY。

### 创建时的页面初始化

真正落盘在 `srv_undo_tablespace_create`（srv0start.cc / 224-297）：建文件 → `os_file_set_size(..., UNDO_INITIAL_SIZE=16MB)` → 加入 construction list → `srv_undo_tablespace_open` → **`srv_undo_tablespaces_construct()`**（srv0start.cc）补写页面：

```cpp
// srv0start.cc
if (!fsp_header_init(space_id, UNDO_INITIAL_SIZE_IN_PAGES, &mtr)) { ... }  // Step-A: page 0
/* Add the RSEG_ARRAY page. */
trx_rseg_array_create(space_id, &mtr);                                     // Step-B: page 3
```

- **Step-A**：`fsp_header_init`（fsp0fsp.cc）写 space_id/size/flags、初始化 5 个 free list、`FSP_SEG_ID=1`，并调 `fsp_fill_free_list` 建 page 1。
- **Step-B**：`trx_rseg_array_create`（trx0rseg.cc）`fseg_create` 出 page 3，写版本号、`size=0`、全部槽 `memset(0xff)`。
- **Step-C**：随后 `trx_rseg_init_rollback_segments` → `trx_rseg_create` → `trx_rseg_header_create`（trx0rseg.cc）为每个 rseg 建 header 页，并回填 RSEG_ARRAY 槽位（`trx_rsegsf_set_page_no`）。

### undo tablespace truncate

`undo::Truncate` 类（trx0purge.h）挂在 `purge_sys->undo_trunc`。触发判据 `needs_truncation()`（trx0purge.cc）：显式 INACTIVE → 直接需要；否则要求 `srv_undo_log_truncate=ON`、文件大小超 `max(srv_max_undo_tablespace_size, 初始大小)`、且旧 space_id 残留页不超阈值。

真正的页面重建在 `trx_undo_truncate_tablespace()`（trx0undo.cc）：

```
 old_space_id = marked_space->id();
 undo::unuse_space_id(old_space_id);
 new_space_id = undo::use_next_space_id(space_num);   // ★ 换同一 num 的下一个 space_id
 fil_delete_tablespace(old_space_id, BUF_REMOVE_NONE);      // Step-1 删旧
 fil_ibd_create(new_space_id, ..., n_pages);                // Step-2 建新文件
 fsp_header_init(new_space_id, n_pages, &mtr);               // Step-3 page 0
 trx_rseg_array_create(new_space_id, &mtr);                  // Step-4 page 3
 rseg->page_no = trx_rseg_header_create(new_space_id, ...);   // Step-5 重建 rseg header
 marked_space->set_space_id(new_space_id);                    // 原子切换
```

崩溃保护靠 `undo_<num>_trunc.log`（`undo::start_logging`/`done_logging`），启动时 `srv_undo_tablespace_fixup()`（srv0start.cc）据此重建。

---

### Rollback Segment（rseg）与 Undo Segment

### trx_rseg_t 内存对象

定义在 `trx0types.h`（**不在** trx0rseg.h）：

```cpp
  size_t id{};              // rseg id == 在 rseg 目录页中的 slot 下标
  space_id_t space_id{};    // rseg header page 所在 space
  page_no_t page_no{};      // rseg header page 页号
  page_no_t curr_size{};    // 当前页数
  Undo_list update_undo_list;     // 活跃 update undo
  Undo_list update_undo_cached;   // 可复用 update undo
  Undo_list insert_undo_list;
  Undo_list insert_undo_cached;
  page_no_t last_page_no{};   // ★ history list 中下一个待 purge 的 log header 页
  size_t last_offset{};       // ★ 页内偏移
  trx_id_t last_trx_no;       // ★ 该 log 的 trx_no
  bool last_del_marks{};      // ★ 该 log 是否需要 purge
  std::atomic<size_t> trx_ref_count{};  // 防止 undo 表空间被 truncate
```

**关键点**：rseg 内存对象里**没有** history list 的 base node。history list 只存在于 rseg header page 的 `TRX_RSEG_HISTORY` 偏移处；内存只保存"进度游标" `last_page_no/last_offset/last_trx_no/last_del_marks`，purge 从这里继续消费。

### rseg header page 布局

常量全在 `trx0rseg.h`（16KB 页的绝对偏移）：

| 常量 | 相对偏移 | 绝对偏移 | 含义 |
|------|---------|---------|------|
| `TRX_RSEG = FSEG_PAGE_DATA` | — | 38 | rseg header 起点 |
| `TRX_RSEG_MAX_SIZE` | 0 | 38 | 4B 允许的最大页数 |
| `TRX_RSEG_HISTORY_SIZE` | 4 | 42 | 4B history list 占用的**页数** |
| `TRX_RSEG_HISTORY` | 8 | 46 | 16B `flst_base_node_t` history list 根 |
| `TRX_RSEG_FSEG_HEADER` | 24 | 62 | 10B 本 rseg 的文件段 inode |
| `TRX_RSEG_UNDO_SLOTS` | 34 | 72 | 1024 × 4B undo slot 数组 |
| `TRX_RSEG_MAX_TRX_NO` | — | 4168 | 8B 本 rseg history 中最大 trx_no |

undo slot 的组织（`trx0rseg.ic`）：每槽 4B 存 undo segment 的首页 page_no，`FIL_NULL` 表示空闲。分配时线性扫描找第一个空闲（`trx_rsegf_undo_find_free`），占用在 `trx_undo_seg_create` 中写入（trx0undo.cc），归还时置 FIL_NULL。slot 耗尽 → `DB_TOO_MANY_CONCURRENT_TRXS`。

### undo segment（trx_undo_t）

定义在 `trx0undo.h`：

```cpp
  ulint id;           // rseg 内的 undo log slot 号
  ulint type;         // TRX_UNDO_INSERT(1) / TRX_UNDO_UPDATE(2)
  ulint state;        // ACTIVE/CACHED/TO_FREE/TO_PURGE/PREPARED...
  bool del_marks;     // 仅 update undo：是否可能做了 delete-mark（需 purge）
  trx_rseg_t *rseg;
  page_no_t hdr_page_no;   // undo log header 所在页（segment 首页）
  ulint hdr_offset;
  ulint size;              // 当前页数
  ulint empty;             // 记录栈是否为空
  page_no_t top_page_no;   // 最新 undo rec 所在页
  ulint top_offset;
  undo_no_t top_undo_no;
  UT_LIST_NODE_T(trx_undo_t) undo_list;   // 挂到 rseg 的 4 个链表之一
```

**insert vs update 的生命周期差异**（`trx_undo_set_state_at_finish`，trx0undo.cc）：

```cpp
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) < TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;      // 单页且空闲足够 → 缓存复用
  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;     // insert → 直接释放（提交后即无用）
  } else {
    state = TRX_UNDO_TO_PURGE;    // update → 交 purge 释放
```

- **insert undo**：提交/回滚后即可丢弃，**永不进 history list**。cached 时用 `trx_undo_insert_header_reuse`（trx0undo.cc）整页重置（`PAGE_FREE` 回到 `TRX_UNDO_SEG_HDR + TRX_UNDO_SEG_HDR_SIZE`）。
- **update undo**：提交后必须入 history list 供 MVCC/purge。注意 **CACHED 的 update undo 也已进 history list**（`trx0purge.cc` 无条件执行），只是 segment 被缓存复用，purge 时只摘 hdr 不 free segment。

### fseg 与 inode 管理

rseg 自身的文件段在 `trx_rseg_header_create` 中创建（trx0rseg.cc `fseg_create`）。undo segment 的文件段在 `trx_undo_seg_create`（trx0undo.cc）创建：

```cpp
  slot_no = trx_rsegf_undo_find_free(rseg_hdr, mtr);
  if (slot_no == ULINT_UNDEFINED) return DB_TOO_MANY_CONCURRENT_TRXS;
  block = fseg_create_general(space, 0, TRX_UNDO_SEG_HDR + TRX_UNDO_FSEG_HEADER, true, mtr);
  trx_undo_page_init(*undo_page, type, mtr);     // FIL_PAGE_UNDO_LOG
  trx_rsegf_set_nth_undo(rseg_hdr, slot_no, page_get_page_no(*undo_page), mtr);
```

扩展加页 `trx_undo_add_page`（trx0undo.cc）：`fseg_alloc_free_page_general` 分配新页挂到 `TRX_UNDO_PAGE_LIST` 尾，`undo->size++`、`rseg->incr_curr_size()`。释放整段 `trx_undo_seg_free`（trx0undo.cc）：`fseg_free_step` + 归还 slot（`trx_rsegf_set_nth_undo(..., FIL_NULL)`）。

### undo 分配路径

入口 `trx_undo_assign_undo()`（trx0undo.cc）：

```
  rseg->latch();
  undo = trx_undo_reuse_cached(rseg, type, trx->id, trx->xid, gtid_storage, &mtr);
  if (undo == nullptr) err = trx_undo_create(...);
  UT_LIST_ADD_FIRST(rseg->insert_undo_list / update_undo_list, undo);
  if (ddl/dict op) trx_undo_mark_as_dict_operation(undo, &mtr);
  rseg->unlatch();
```

- `trx_undo_create`（trx0undo.cc）：新分配文件段 + 占新 slot + `curr_size++` + 建 log header。
- `trx_undo_reuse_cached`（trx0undo.cc）：从 rseg 的 cached 链表摘一个（必然 `size==1`），只重置/重建 log header，`curr_size` 与 slot 不变。

rseg 选择策略（trx0trx.cc）：`get_next_redo_rseg_from_undo_spaces()`**round-robin** 遍历 `(space, rseg_id)`，跳过非 active 空间；`get_next_temp_rseg()`从 `trx_sys->tmp_rsegs` 取。**只读事务不分配 durable rseg**（`trx_start_low`）。临时表写 undo 走 `m_noredo` 且 `MTR_LOG_NO_REDO`。

### rsegs / tmp_rsegs / undo::spaces 关系

| 容器 | 位置 | 内容 | 持久化 | 用途 |
|------|------|------|--------|------|
| `trx_sys->rsegs` | trx0sys.h | TRX_SYS 页扫出的 rseg（含系统表空间 slot 0） | TRX_SYS 页 | 升级兼容/purge 扫描 |
| `trx_sys->tmp_rsegs` | trx0sys.h | 临时表空间 rseg | 无（每次启动重建） | 临时表 undo，noredo |
| `undo::spaces->m_spaces[i]->rsegs()` | trx0purge.h | 各独立 undo 表空间的 rseg | 各空间 page 3 RSEG_ARRAY | 8.0 主分配路径 |
| `trx_sys->rseg_history_len` | trx0sys.h | 全局 history 条数总和 | — | 触发 purge 唤醒 |

启动扫描：`trx_rsegs_init`（trx0rseg.cc）扫 TRX_SYS 的 128 槽 + 各 undo 表空间的 RSEG_ARRAY，调 `trx_rseg_mem_create`建内存对象并压入 purge queue。

---

### Undo Page 布局

### 页头 TRX_UNDO_PAGE_HDR

常量在 `trx0undo.h`（16KB 页绝对偏移）：

```cpp
 TRX_UNDO_PAGE_HDR = FSEG_PAGE_DATA;      // = 38
 TRX_UNDO_PAGE_TYPE  = 0;   // 2B: TRX_UNDO_INSERT(1) / TRX_UNDO_UPDATE(2)
 TRX_UNDO_PAGE_START = 2;   // 2B: 本页最新事务的记录区起点
 TRX_UNDO_PAGE_FREE  = 4;   // 2B: 第一个空闲字节偏移
 TRX_UNDO_PAGE_NODE  = 6;   // 12B: flst_node，挂到 segment 的 page list
 TRX_UNDO_PAGE_HDR_SIZE = 6 + FLST_NODE_SIZE;   // = 18
```

```
 0..37     FIL Header (38B)
 38..55    TRX_UNDO_PAGE_HDR (18B)：TYPE(38) START(40) FREE(42) NODE(44,12B)
 56..85    TRX_UNDO_SEG_HDR (30B)  —— 仅 segment 首页
 86..      TRX_UNDO_LOG_HDR (46B 起) —— 仅段首页，可有多个
 ...
 记录区    从 PAGE_FREE 向高地址增长
 末尾 8B   FIL Trailer
```

页初始化 `trx_undo_page_init`（trx0undo.cc）设 TYPE/START/FREE 并 `fil_page_set_type(FIL_PAGE_UNDO_LOG)`。

### segment 头 TRX_UNDO_SEG_HDR

```cpp
 TRX_UNDO_SEG_HDR = TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_HDR_SIZE;   // 56
 TRX_UNDO_STATE       = 0;   // 2B: ACTIVE(1)/CACHED(2)/TO_FREE(3)/TO_PURGE(4)/PREPARED(5,6,7)
 TRX_UNDO_LAST_LOG    = 2;   // 2B: 本段最后一个 undo log header 偏移，0=无
 TRX_UNDO_FSEG_HEADER = 4;   // 10B
 TRX_UNDO_PAGE_LIST   = 14;  // 16B flst_base_node_t，本 segment 的页链表
 TRX_UNDO_SEG_HDR_SIZE = 30;
```

### undo log header TRX_UNDO_LOG_HDR

```cpp
 TRX_UNDO_TRX_ID      = 0;    // 8B
 TRX_UNDO_TRX_NO      = 8;    // 8B 事务 no（进 history list 后才有意义）
 TRX_UNDO_DEL_MARKS   = 16;   // 2B 是否需要 purge
 TRX_UNDO_LOG_START   = 18;   // 2B 本 log 第一条记录偏移（purge 推进它）
 TRX_UNDO_FLAGS       = 20;   // 1B: XID(0x01)/GTID(0x02)/XA_PREPARE_GTID(0x04)
 TRX_UNDO_DICT_TRANS  = 21;   // 1B DDL 事务
 TRX_UNDO_NEXT_LOG    = 30;   // 2B 本页下一个 log header 偏移
 TRX_UNDO_PREV_LOG    = 32;   // 2B 本页上一个 log header 偏移
 TRX_UNDO_HISTORY_NODE= 34;   // 12B flst_node（进 history list 用）
 TRX_UNDO_LOG_OLD_HDR_SIZE = 34 + FLST_NODE_SIZE;   // = 46
```

扩展区：`+46` XA 区（XID 128B → 186）、`+186` GTID（64B → 251）、`+251` XA GTID（64B → 315）。`trx_undo_header_add_space_for_xid`（trx0undo.cc）按需推进 `PAGE_START/PAGE_FREE/LOG_START`。

### 一页内多个 undo log 的组织

一个 update undo 段若被缓存复用，其**首页可容纳多个 log header**（每个 ≥46B），用 `TRX_UNDO_NEXT_LOG` / `TRX_UNDO_PREV_LOG` 双向串联，`TRX_UNDO_LAST_LOG` 指向最后一个。**只有最后一个 log 拥有后续页上的记录**。

边界由 `trx_undo_page_get_start/end` 决定（`trx0undo.ic`）：

```
log N 的记录区间 = [log N 的 LOG_START, log N 的 NEXT_LOG 或 PAGE_FREE)
```

跨页时通过 `TRX_UNDO_PAGE_NODE` 走页链表。

### insert vs update section

页头 `TRX_UNDO_PAGE_TYPE` 决定该 segment 整体是 INSERT(1) 还是 UPDATE(2)，写记录时有严格断言（trx0rec.cc insert 路径要求 `== TRX_UNDO_INSERT`；:1196-1197 modify 路径要求 `== TRX_UNDO_UPDATE`）。因此**一个 undo page 只属于一类**。

---

### Undo Record 格式与记录类型

### 物理头尾：next / prev

一条 undo record 在页内连续存放，**头部 2B 存本记录结束偏移（=下一条起点），尾部 2B 存本记录起始偏移**。由 `trx_undo_page_set_next_prev_and_add`（trx0rec.cc）写出：

```cpp
  mach_write_to_2(ptr, first_free);              // 记录尾部 2B = 本记录起始偏移
  mach_write_to_2(undo_page + first_free, end_of_rec);  // 记录头部 2B = 本记录结束偏移
```

读取：下一条读本记录头部 2B；**上一条要读前一条记录尾部 2B**（`trx0undo.ic` `mach_read_from_2(rec - 2)`）。记录长度 = `next - offset`（trx0rec.ic）。

### type_cmpl 位定义

记录类型常量（trx0rec.h）：

```cpp
 TRX_UNDO_INSERT_REC    = 11;   // 聚簇索引新插入
 TRX_UNDO_UPD_EXIST_REC = 12;   // 更新未打删除标记的记录
 TRX_UNDO_UPD_DEL_REC   = 13;   // 更新已 delete-mark 的记录
 TRX_UNDO_DEL_MARK_REC  = 14;   // 删除标记
 TRX_UNDO_CMPL_INFO_MULT = 16;
 TRX_UNDO_MODIFY_BLOB    = 64;  // bit6
 TRX_UNDO_UPD_EXTERN     = 128; // bit7
```

位分解：

```
bit7(0x80) UPD_EXTERN   : 是否更新了 externally stored 字段（purge 需释放）
bit6(0x40) MODIFY_BLOB  : 8.0 起 update 类恒置位 → 后面多 1B flag
bit5..4    cmpl_info    : UPD_NODE_NO_ORD_CHANGE=1 / UPD_NODE_NO_SIZE_CHANGE=2
bit3..0    type (11..14)
```

**关键**：8.0 中所有 update 类（12/13/14）都置 `MODIFY_BLOB`（写侧 trx0rec.cc 无条件 `|=`），所以 `undo_no` 固定从 rec+4 开始；insert 类（11）从 rec+3 开始（trx0rec.ic）。

### 通用参数区

```
偏移(相对 rec)  大小        内容
+0              2B          next = 本记录结束偏移
+2              1B          type_cmpl
+3              0/1B        [update 类] 1B flag，当前恒 0x00
+3/+4           1..11B      undo_no（much_compressed）
next            1..11B      table_id（much_compressed）
...             ...         各 type 的 payload
end-2           2B          prev = 本记录起始偏移
```

变长编码：`mach_write_compressed` 1-5B（mach0data.ic）；`mach_u64_write_much_compressed` 1-11B（mach0data.ic）。列值编码为 `mach_write_compressed(flen)` + data，`UNIV_SQL_NULL(0xFFFFFFFF)` 只写长度不写数据。

### 各 type 的 payload

**INSERT(11)** — `trx_undo_page_report_insert`（trx0rec.cc）：

```
[2B next][1B type=11][undo_no][table_id]
[len1][pk1][len2][pk2] ... （n_unique 个主键列）
[可选: 2B v_cols 总长][虚拟列块 ...]
[2B prev]
```

**update 类通用头部（12/13/14 共有）** — `trx_undo_page_report_modify`（trx0rec.cc）：

```
[2B next][1B type_cmpl][1B flag=0x00][undo_no][table_id]
[1B info_bits][trx_id 5..9B][roll_ptr 5..9B]     ← 旧 DB_TRX_ID / DB_ROLL_PTR
[len][pk] × n_unique
```

之后各类型追加不同内容：

| 类型 | 更新向量段 | 排序列全量段 | 说明 |
|------|-----------|-------------|------|
| UPD_EXIST(12) | ✅ n_updated + 逐列旧值 | 仅当 `!(cmpl_info & UPD_NODE_NO_ORD_CHANGE)` | 完整更新 |
| UPD_DEL(13) | ✅ | ✅（同上条件） | 更新已删标记记录 |
| DEL_MARK(14) | ❌ n_fields=0，只回滚系统列 | ✅ | 仅打删除标记 |

"排序列全量段"（trx0rec.cc）记录**所有出现在任意索引排序中的列**的旧值，供 purge 重建二级索引条目（`trx_undo_rec_get_partial_row` 读取）。它是否存在由 `cmpl_info` 决定——这正是 `UPD_NODE_NO_ORD_CHANGE` 优化的核心：若更新未改任何有序列，就不记这段，purge 也可跳过。

外部列长度用哨兵值 `UNIV_EXTERN_STORAGE_FIELD`（= `UNIV_SQL_NULL - 16384`）编码，bit12-13 嵌入 spatial status（GIS）。LOB 部分更新另有块（trx0rec.cc）。

### roll pointer（DB_ROLL_PTR，7 字节）

编码在 `trx0undo.ic`：

```
bit:  55    54.........48  47...................16  15.........0
     +----+----------------+------------------------+-----------+
     |1b  |    7b         |        32b              |   16b     |
     |ins | rseg_id/     | page_no                 | offset    |
     |ert | undo space no|                         |           |
     +----+----------------+------------------------+-----------+
```

```cpp
  roll_ptr = (roll_ptr_t)is_insert << 55 | (roll_ptr_t)id << 48 |
             (roll_ptr_t)page_no << 16 | offset;
```

8.0 起这 7 bit 是 **undo space number**（0..127，0 表示系统表空间），由 `undo::id2num(space_id)` 换算（trx0rseg.ic）。生成点 `trx_undo_report_row_operation`（trx0rec.cc）。

---

### Undo 生成路径

### 总入口 `trx_undo_report_row_operation`

所有 DML 写 undo 都收敛到这一个函数。先澄清一个常见误解：它的 `op_type` 参数**只有两个值**——`TRX_UNDO_INSERT_OP` 和 `TRX_UNDO_MODIFY_OP`。前面讲的 record type（11/12/13/14）**不是**入口参数，而是 `trx_undo_page_report_modify` 内部根据 `update` 是否为 NULL、以及 `rec` 是否已打删除标记推导出来的。

选择 insert_undo 还是 update_undo 完全由 `op_type` 决定：

```c
switch (op_type) {
  case TRX_UNDO_INSERT_OP:
      undo = undo_ptr->insert_undo;
      if (undo == nullptr) { trx_undo_assign_undo(trx, undo_ptr, TRX_UNDO_INSERT); ... }
      break;
  default:
      ut_ad(op_type == TRX_UNDO_MODIFY_OP);
      undo = undo_ptr->update_undo;
      if (undo == nullptr) { trx_undo_assign_undo(trx, undo_ptr, TRX_UNDO_UPDATE); ... }
```

`undo_ptr` 本身也有讲究：临时表走 `trx->rsegs.m_noredo` 并把 mtr 设为 `MTR_LOG_NO_REDO`（临时表无需崩溃恢复），普通表走 `m_redo`。

**定位当前 undo 页用的是 `last_page_no`，不是 `top_page_no`。** 三个"位置"字段语义不同：`hdr_page_no/hdr_offset` 是 log header（存 XID/GTID/flags 的地方），`last_page_no` 是页链表尾（新记录写这里），`top_page_no/top_offset/top_undo_no` 是**栈**顶（回滚用时才动）。写入成功后才更新 top_*。

### 三种 DML 各写什么

| DML | 上层 | B-tree 层 | 落到的 record type |
|---|---|---|---|
| INSERT | `row_ins_clust_index_entry_low` | `btr_cur_ins_lock_and_undo` | `INSERT_REC(11)`，只记主键列 |
| UPDATE（不改有序列） | `row_upd_clust_step` → `row_upd_clust_rec` | `btr_cur_upd_lock_and_undo` | `UPD_EXIST_REC(12)` |
| UPDATE（改主键） | `row_upd_clust_rec_by_insert` | del_mark + insert | 一条 `DEL_MARK(14)` + 一条 `INSERT(11)` |
| INSERT 撞上 delete-marked 记录 | `row_ins_clust_index_entry_by_modify` | `btr_cur_*_update` | `UPD_DEL_REC(13)` |
| DELETE | `row_upd_del_mark_clust_rec` | `btr_cur_del_mark_set_clust_rec` | `DEL_MARK(14)`，`update == NULL` |

`btr_cur_ins_lock_and_undo` 里有一句关键的短路：**只有聚簇索引才写 undo**，二级索引和 ibuf 直接返回。

**INSERT 为什么只记主键列**：insert 的逆操作是"按主键把这条记录删掉"，而 insert undo 不参与 MVCC 版本链（事务提交后即可丢弃），所以唯一需要的就是能定位到这一行的主键。

**UPDATE 的 type 推导**：

```c
if (!update)                                    type_cmpl = TRX_UNDO_DEL_MARK_REC;   // 14
else if (rec_get_deleted_flag(rec, ...))        type_cmpl = TRX_UNDO_UPD_DEL_REC;    // 13
else                                            type_cmpl = TRX_UNDO_UPD_EXIST_REC;  // 12
type_cmpl |= cmpl_info * TRX_UNDO_CMPL_INFO_MULT;
*type_cmpl_ptr |= TRX_UNDO_MODIFY_BLOB;         // 8.0 update 类恒置位
```

**`cmpl_info` 从哪来**：MySQL 接口路径下 `row_upd` 现算——若本次 update 改动了任一出现在索引排序中的列（`row_upd_changes_some_index_ord_field_binary`），`cmpl_info = 0`；否则为 `UPD_NODE_NO_ORD_CHANGE`。这个值直接决定"排序列全量旧值"那一段写不写，进而决定 purge 能否整条跳过（见「Undo Record 格式」）。

**DELETE 必然写排序列**：因为 `update == NULL`，那个 `if (!update || !(cmpl_info & UPD_NODE_NO_ORD_CHANGE))` 的分支必然成立——这正是 purge 需要的（它要靠这些旧值去二级索引里定位删除）。

### `undo_no` 的分配

`trx->undo_no` 是"下一条待分配的编号"，从 0 开始，**insert_undo 与 update_undo 共享同一个计数器**。成功写入后：

```c
undo->top_undo_no = trx->undo_no;   // 本条记录的编号
trx->undo_no++;                     // 推进
```

所以 `top_undo_no == trx->undo_no - 1`。这个编号会被写进 record 正文，回滚时靠它弹栈（逆序），purge 时靠它做截断水位线。

### 页空间：不存在"部分写"

`trx_undo_left()` 返回当前页剩余空间（预留 10 字节安全边界）。写记录的每一步都会检查，空间不足就 `return 0`：

```c
if (trx_undo_left(undo_page, ptr) < 2 + 1 + 11 + 11) return 0;   // insert 开头
if (trx_undo_left(undo_page, ptr) < 50) return 0;                // modify 开头
```

`return 0` 时页头的 `TRX_UNDO_PAGE_FREE` **从未推进**，所以从页的视角这条半途而废的记录根本不存在。随后 `trx_undo_erase_page_end` 把尾部残留字节抹成 0xff，再加一页重试：

- 页上原来有记录 → 抹尾 + `trx_undo_add_page` + 重试（最多加一页）
- 页是空的都放不下 → 释放刚分配的页，返回 `DB_UNDO_RECORD_TOO_BIG`（表现为 `ER_UNDO_RECORD_TOO_BIG`）

**结论：一条 undo record 必须完整落在一页内，InnoDB 不支持跨页的部分写。**

### undo 与 redo 的关系：两个 mtr，undo 在前

这是最容易误解的一点。`trx_undo_report_row_operation` **自己开一个 mtr**（签名里没有 mtr 参数），调用方的数据页修改在**另一个 mtr** 里。时序必然是：

```
[调用方 mtr 已持有索引页 latch]
   └─ trx_undo_report_row_operation
        mtr_start(undo mtr) → X-latch undo 页 → 写 record → MLOG_UNDO_INSERT
        mtr_commit(undo mtr)                    ← undo 的 redo 先落
   ← 返回 roll_ptr
[调用方 mtr 继续]
   修改数据页（写 roll_ptr / delete-mark / 插入记录）
   mtr_commit                                   ← 数据页 redo 后落
```

**redo LSN 顺序上 undo 严格在前**，这正是"先有 undo、再有指向它的 roll_ptr"的保证——恢复时不会出现数据页有个 roll_ptr 指向尚未恢复的 undo。

`MLOG_UNDO_INSERT` 只记录 record 的 **body**（从 `old_free+2` 开始的净长度），**next/prev 两个 2 字节指针不入 redo**——重做时由 `trx_undo_parse_add_undo_rec` 依据页当前的 `first_free` 现场重建。

**为什么 undo 自身必须受 redo 保护**：undo 页也是 buffer pool 里的普通页，同样遵守 WAL。若不记 redo，崩溃后重做数据页会恢复出一条 `roll_ptr` 指向某个 (rseg, page, offset) 的记录，但那个 undo 页可能还没刷盘 → 该位置是旧数据或垃圾 → 回滚无法进行、版本链断裂、purge 无法清理。唯一例外是临时表（`MTR_LOG_NO_REDO`），因为它重启后就不存在了。

### `DB_ROLL_PTR` 的回填时机

roll_ptr 的回填发生在**数据页的 mtr** 里（不是 undo 的 mtr），由两个函数分工：

- `row_upd_rec_sys_fields`——原地改写已存在的记录（UPDATE-in-place、DELETE 打标记），同时写 `DB_TRX_ID` 和 `DB_ROLL_PTR`
- `row_upd_index_entry_sys_field`——把值填进待物化的 dtuple（INSERT、UPDATE 重建记录）

所以**插入的记录同样带 `DB_TRX_ID` + `DB_ROLL_PTR`**，只是 roll_ptr 的第 55 位（is_insert）为 1。唯一的例外是 intrinsic 表（优化器内部临时表），它不分配 roll_ptr 字段也不写 undo。

### latching 顺序

一次 DML 写 undo 涉及的 latch，级别严格降序：

```
聚簇索引页 latch(SYNC_TREE_NODE / SYNC_INDEX_TREE)
  → trx->undo_mutex（保护 undo_no 与 top_* 的原子推进）
    → rseg mutex(SYNC_RSEG_HEADER)
      → undo 页 latch(SYNC_TRX_UNDO_PAGE)
        → FSP / FSP_PAGE latch
```

有一处必须留意：**加页/删页前必须先把 mtr 提交掉**。因为加页要拿更高级别的 rseg mutex，而当时手里可能握着更低级别的 FSP / undo 页 latch——按 InnoDB 的 latch 排序规则（只能降序获取），这会形成反向等待。源码里的注释说得很直白，加页"类比 B-tree 的悲观插入，必须预留对应的树 latch，即 rseg mutex"，所以显式 `mtr_commit` + `mtr_start` 把低级别 latch 全释放，再拿 rseg mutex。

---

### Undo 与事务回滚

### 三条入口，收敛到一个函数

| 场景 | 入口 |
|---|---|
| 整事务回滚（ROLLBACK / 连接关闭 / kill / XA ROLLBACK） | `trx_rollback_for_mysql` → `trx_rollback_low` |
| 语句级回滚 | `trx_rollback_last_sql_stat_for_mysql`（等价于回滚到 `last_sql_stat_start`） |
| 命名 savepoint / 部分回滚 | `trx_rollback_to_savepoint` |

三者最终都收敛到 `trx_rollback_to_savepoint_low(trx, savept)`，`savept == nullptr` 即全回滚。

### 两层 query graph

回滚**用 query graph 执行**，这不是历史包袱，而是三个实际需要：

1. **复用 row/btr 层原语**：回滚要调 `btr_cur_optimistic_update`、`btr_cur_pessimistic_delete`、`row_search_index_entry` 等，这些 API 的签名统一需要 `que_thr_t *thr`（用于取 trx、写 `error_state`、锁等待）。没有 `thr` 根本调不动。
2. **step 化**：一条 undo record 的应用必然跨多个 mtr（先乐观后悲观，mtr 在 latch 新页前必须 commit）。`undo_node_t::state`（`UNDO_NODE_FETCH_NEXT` / `INSERT` / `MODIFY`）把"取记录 / 处理 / 清理"拆成可重入的 step，正好由 graph 循环驱动。
3. **统一错误语义**：任何非 `DB_SUCCESS` 直接 `ib::fatal`——**回滚不允许失败**，连 `DB_OUT_OF_FILE_SPACE` 也是 fatal（"Out of tablespace during rollback"），失败就自杀让崩溃恢复重来。

结构上分两层：外层节点 `QUE_NODE_ROLLBACK`（`trx_rollback_step`，负责 `trx_commit_or_rollback_prepare` 和建内层图），内层节点 `QUE_NODE_UNDO`（`row_undo_step` → `row_undo`），外层跑完再跑内层。

### 按 roll_ptr 的 insert 位分派

`row_undo` 决定走 insert 回滚还是 update 回滚，依据是 **roll_ptr 的第 55 位**：

```c
if (trx_undo_roll_ptr_is_insert(roll_ptr)) node->state = UNDO_NODE_INSERT;
else                                       node->state = UNDO_NODE_MODIFY;
```

不是看 record 里的 type 字段。这决定了从 `insert_undo` 还是 `update_undo` 的栈顶取记录。

### 逆序：两个 undo 合成一个逻辑栈

回滚必须**从最新记录往回走**——行版本链是新→旧的单向链，只有先撤最新的，才能把系统列正确还原成上一版的状态。

弹栈由 `trx_roll_pop_top_rec_of_trx_low` 完成，它会把 insert_undo 与 update_undo **按 `top_undo_no` 合成一个逻辑栈**：

```c
if      (!ins_undo || ins_undo->empty)                  undo = upd_undo;
else if (!upd_undo || upd_undo->empty)                  undo = ins_undo;
else if (upd_undo->top_undo_no > ins_undo->top_undo_no) undo = upd_undo;
else                                                    undo = ins_undo;

if (!undo || undo->empty || limit > undo->top_undo_no) return nullptr;  // 回滚到点
```

`limit` 为 0 时就是全回滚。弹栈过程中 `trx->undo_no` 会**随回滚递减**（被赋成刚弹出记录的 undo_no），崩溃恢复时正是用它打印进度百分比。

遍历靠页内的双向链：`mach_read_from_2(rec - 2)` 取上一条（更旧）记录，跨页则回退到前一页取最后一条。

### 三类回滚各做什么

| 类型 | 二级索引 | 聚簇索引 |
|---|---|---|
| **insert** | 从聚簇记录全列重建条目逐个删除 | **直接物理删除** |
| **UPD_EXIST** | 两趟：先用"新值行"删掉新条目，再用"旧值行"恢复旧条目 | 用 update vector 反向更新 |
| **DEL_MARK** | 用"新值行"建条目，清删除标记 | update vector 只有 2 个系统列，靠 `info_bits` 清标记 |
| **UPD_DEL** | 用"新值行"把条目打标记或删除 | 先恢复成 delete-marked 旧版本，再尝试物理删除 |

**insert 回滚为什么能"无检查"地直接删**：
1. fresh insert 的前像是"不存在的行"，任何 read view 都不可能需要它的旧版本；
2. 定位时已做身份校验——`row_get_rec_roll_ptr(rec) == node->roll_ptr` 且断言 `row_get_rec_trx_id(rec) == trx->id`，排除了槽位被别的事务复用的可能；
3. 不需要加锁检查（所有 btr 调用带 `BTR_NO_LOCKING_FLAG`，插入行的锁就是 trx_id 承载的隐式锁）。

唯一例外是部分回滚：`row_convert_impl_to_expl_if_needed` 在 `partial && isolation_level >= RR` 时把隐式锁转成显式锁，避免 IODKU/REPLACE 在语句回滚中途被别的事务插入而破坏可串行化。

**如何恢复 `DB_TRX_ID` / `DB_ROLL_PTR`**：`trx_undo_update_rec_get_update` 构造 update vector 时**无条件多留两个槽位**——倒数第二个放旧的 `DB_TRX_ID`，最后一个放旧的 `DB_ROLL_PTR`。而 `row_undo_mod_clust_low` 传 `BTR_KEEP_SYS_FLAG`，让 btr 层**跳过**"用当前 trx 覆盖系统列"这一步，改由 update vector 里的旧值写入。回滚本身还带 `BTR_NO_UNDO_LOG_FLAG`——**回滚不写 undo**。

### savepoint：只需要一个 undo 号

8.0 的 `trx_savept_t` 简化到只有一个字段 `least_undo_no`（5.x 时代还记 undo 段的 top_page_no/top_offset，现在不需要了——栈顶位置由 `trx_undo_t` 内存对象维护）。

部分回滚的语义就是：**撤回 `undo_no >= least_undo_no` 的记录，保留更小的**。全回滚时 `roll_limit = 0`，等价于撤到 0。

注意一个语义细节：部分回滚**不释放内存中的锁**，只有"新插入行上由 trx_id 承载的隐式锁"会随行被删除而自然释放。

### XA PREPARE：undo 置 PREPARED 态

`trx_prepare` 把 undo 段状态从 `TRX_UNDO_ACTIVE` 改为 `TRX_UNDO_PREPARED`，并把 XID 写进 log header。这一步的意义是：**prepare 后即使崩溃，重启也能恢复出 prepared 状态**。

**为什么 prepare 之后的 update undo 不会被 purge**：purge 只从 rseg 的 history list 取活，而挂进 history 的唯一时机是 commit 的序列化（`trx_purge_add_update_undo_to_history`）。prepare 阶段**不分配 `trx->no`**，undo 段仍挂在 rseg slot 上、状态是 PREPARED，既不在 history list 也不在 purge queue——purge 完全看不到它，它保护的行版本也就不会被清理。这正是"XA PREPARE 后崩溃仍能 XA COMMIT/ROLLBACK"的前提。

XA ROLLBACK 时则反向操作：把段状态写回 `TRX_UNDO_ACTIVE` 并落 redo，这样万一再崩溃，恢复阶段会把它当普通未提交事务回滚掉。

### 崩溃恢复中的回滚

1. redo 回放先把 undo 页恢复到崩溃时刻；
2. `trx_undo_lists_init` 逐 slot 扫描每个 rseg，把磁盘上的 undo 段重建为内存对象，**只有进入 `insert_undo_list` / `update_undo_list`（非 CACHED）的段才代表"未完成事务"**；
3. `trx_resurrect` 据此重建 trx：状态是 PREPARED 的记为 `TRX_STATE_PREPARED`（交给 MySQL 层决定 XA COMMIT 还是 ROLLBACK），ACTIVE 的交给 `trx_rollback_active` 走标准回滚；
4. `trx_resurrect_table_ids` 从栈顶沿链遍历整条 undo 收集 table_id，用于后续加 MDL 和 IX 表锁。

恢复中的回滚就是标准的"弹栈 + `row_undo`"，只是多了一些宽松断言（`thr_is_recv(thr)`）——因为崩溃可能发生在"聚簇记录写完了但二级索引还没写完"的中间态。

### 回滚的性能特征与一个反直觉的事实

大事务回滚慢，原因叠加：

- **逐条、单行、单线程**，每条记录一次（UPD_EXIST 是两次）B-tree 定位，比原操作"顺序追加 undo + 一次 B-tree 修改"更贵；
- 回滚**要写 redo**（回滚不写 undo，但数据页修改照常记 redo）；
- 空间只按页粒度回收，且 **header page 在回滚过程中永不释放**；
- 锁一直持有（2PL），期间 purge 无法介入这些行。

**一个反直觉的事实：回滚掉的 update undo 仍然会进 history list。** 因为回滚结束走的是标准的 commit 序列化路径，`trx_purge_add_update_undo_to_history` 是无条件调用的。也就是说**回滚会让 `rseg_history_len` 增加**，purge 需要跟进。不过记录体在回滚过程中已被 `trx_undo_truncate_end` 截掉，purge 扫到这些段时基本没行要清；如果该段没有 del_mark 记录，purge 还有快路径（见「Purge 全流程」的 `last_del_marks` 优化）直接跳过解析。

---

### Undo 与 MVCC

> **边界**：本篇只讲 **undo 侧**——版本链怎么组织、怎么沿链回溯、怎么重建出旧版本。read view 的四字段与 `changes_visible` 的可见性规则、RR/RC 下 view 的创建时机、AC-NL-RO 的 view 生命周期，见 [`mvcc.md`](mvcc.md)。两者的接口只有一句：`view->changes_visible(trx_id, name)` 返回 true 表示"这个版本够老了，可以停"。

### 版本链是怎么串起来的

聚簇索引每条记录行头有两个物理相邻的系统列：`DB_TRX_ID`（6B，最后修改该版本的事务）和 `DB_ROLL_PTR`（7B，指向产生当前版本的那次修改所对应的 undo record）。

关键在于——**每条 update undo record 内部也存了一份"旧的 `DB_TRX_ID` / 旧的 `DB_ROLL_PTR`"**。反向应用时这两个值会和普通列一起被写回，于是链就往前推进了一跳：

```
V_n（页上当前记录）: DB_TRX_ID=T_n, DB_ROLL_PTR=RP_n ──► undo_rec_n
                                                          ├─ 旧 DB_TRX_ID  = T_{n-1}
                                                          ├─ 旧 DB_ROLL_PTR= RP_{n-1}
                                                          └─ 被更新列的旧值
                              │ 反向应用到 V_n
                              ▼
V_{n-1}：DB_TRX_ID=T_{n-1}, DB_ROLL_PTR=RP_{n-1} ──► undo_rec_{n-1} ──► ...
```

所以版本链**不是** undo 页内的一条物理链表，而是靠"每次 UPDATE 时把旧系统列存进 undo"逐跳反解出来的**逻辑链**。undo 页内的 `prev/next` 偏移只是同页记录的物理串联，与版本链无关。

### `trx_undo_prev_version_build`：单跳引擎

这是整个 MVCC undo 侧的核心，**每次调用只回溯一跳**。流程：

1. 取 `roll_ptr`；若 `is_insert` 位为 1，说明是首次插入的版本，**返回 true 但 `old_vers = nullptr`**（"成功地告诉你没有更老版本了"）。
2. `trx_undo_get_undo_rec` 取 undo record——它是安全边界的守门人（见下）。
3. 解析：`trx_undo_rec_get_pars`（type/cmpl/undo_no/table_id）→ `trx_undo_update_rec_get_sys_cols`（旧系统列）→ `trx_undo_rec_skip_row_ref`（跳过主键，一致性读不需要）→ `trx_undo_update_rec_get_update`（构造 update vector）。
4. **反向应用**：把 update vector 应用到当前版本上，得到上一版本。

反向应用有两条路径，由 `row_upd_changes_field_size_or_external` 选择：

```c
if (row_upd_changes_field_size_or_external(index, offsets, update)) {
    /* 变长 / 外部存储：重建 dtuple 再转换 */
    entry = row_rec_to_index_entry(rec, index, offsets, heap);
    row_upd_index_replace_new_col_vals(entry, index, update, heap);
    *old_vers = rec_convert_dtuple_to_rec(buf, index, entry);
} else {
    /* 定长：直接复制 + 原地覆写，最快路径 */
    *old_vers = rec_copy(buf, rec, offsets);
    row_upd_rec_in_place(*old_vers, index, offsets, update, nullptr);
}
```

`row_upd_rec_in_place` 会连 `DB_TRX_ID` / `DB_ROLL_PTR` 那两个槽位一起覆写——这就是链向前推进的物理动作；`info_bits`（含 delete-mark 位）也是在这里被翻回的。

**BLOB 不 fetch**：undo 里只存 BLOB 的**前缀**（由 `trx_undo_page_fetch_ext` 写入），长度保证不小于二级索引可能的最长列前缀，因此重建二级索引条目无需解引用 BLOB 指针。唯一的例外是 delete-marked 且 BLOB 已被"disown"的情况，需要显式向 purge view 确认，否则当作 fresh insert 处理。

### 安全边界：顶端靠 latch，底端靠 purge view

这是理解"为什么一致性读可以放心沿链走"的关键。

- **顶端安全**：任何新版本只能由"修改聚簇索引记录"产生，而修改必须先 latch 该页。既然我们持有 latch，栈顶就不可能在遍历期间改变。`trx_undo_prev_version_build` 开头的 `ut_ad(mtr_memo_contains_page(...))` 就是这个契约的运行时检查。
- **底端安全**由 `trx_undo_get_undo_rec` 把关：

```c
rw_lock_s_lock(&purge_sys->latch);
missing_history = purge_sys->view.changes_visible(trx_id, name);
if (!missing_history) *undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
rw_lock_s_unlock(&purge_sys->latch);
return missing_history;    // true = 历史已被 purge 回收
```

用**当前版本的 `DB_TRX_ID`** 去问 purge view："这个版本你 purge 看得到吗？"若可见，说明没有任何活跃事务需要它，其 undo 可能已被回收 → 拒绝继续回溯。整个 fetch 都在 `purge_sys->latch` 的 s-lock 下完成，保证中途 purge 不会把页抽走。

于是链的最末端那个 `DB_ROLL_PTR` 可能指向已被回收的垃圾，但因为遍历会在到达它之前就停下，**它永远不会被解引用**。

### `trx_undo_get_undo_rec_low`：从 roll_ptr 到 undo 页

五步：`trx_undo_decode_roll_ptr` 拆出 (is_insert, rseg_id, page_no, offset) → `trx_rseg_id_to_space_id` 换算 space → `fil_space_get_page_size` → `trx_undo_page_get_s_latched`（`buf_page_get(RW_S_LATCH)`）→ `trx_undo_rec_copy` 拷进 heap 后立即 `mtr_commit`。

两点值得注意：

1. **页不在 buffer pool 时会触发同步磁盘 I/O**（没有传 `BUF_GET_IF_IN_POOL` 之类的标志）。这是一致性读回溯长链可能很慢的物理根源之一——版本链在 undo 表空间里跳跃分布，往往是随机 I/O。
2. **必须 copy 到 heap**：mtr 提交后 S latch 即释放，若不 copy，后续解析就是在无保护下读 buffer pool 页。而且 update vector 里的 `dfield` 直接指向 copy 出来的字节，整条 record 的生存期都要靠这份 copy 保障。

### 上层：`row_vers_build_for_consistent_read` 多跳循环

单跳引擎只走一跳，真正"沿链找到可见版本"的是这个循环：

```c
for (;;) {
    mem_heap_t *prev_heap = heap;
    heap = mem_heap_create(1024);
    purge_sees = trx_undo_prev_version_build(rec, mtr, version, index, *offsets, heap,
                                             &prev_version, nullptr, vrow, 0, lob_undo);
    if (prev_heap != nullptr) mem_heap_free(prev_heap);
    if (prev_version == nullptr) { *old_vers = nullptr; break; }   // 快照里不存在这行

    trx_id = row_get_rec_trx_id(prev_version, index, *offsets);
    if (view->changes_visible(trx_id, index->table->name)) {
        *old_vers = rec_copy(buf, prev_version, *offsets);          // 够了，拷进长期堆返回
        break;
    }
    version = prev_version;                                          // 不够老，继续往回
}
```

调用方（`row_search_mvcc`）保证进入时 `!view->changes_visible(trx_id)`（当前版本不可见）。`old_vers == nullptr` 表示"这行在快照里根本不存在"，直接跳过——这正是**未提交/晚于快照的 INSERT 对其他事务不可见**在 undo 侧的落地。

封装有两层：`row_sel_build_prev_vers`（服务端内部执行器）和 `row_sel_build_prev_vers_for_mysql`（handler 路径，额外传 `vrow` 与 `lob_undo`）。此外还有半一致读变体 `row_vers_build_for_semi_consistent_read`，它的判据不是 read view 而是"该版本所属事务是否已提交"。

### 链终止：insert undo 只有一段

`trx_undo_roll_ptr_is_insert` 为 true 即到链底。原因是 insert undo record **没有**"旧 `DB_TRX_ID` / 旧 `DB_ROLL_PTR`"——插入之前没有行，无所谓"旧版本"——也**没有**被更新列的 before-image，只存了主键。所以：

- insert 之后不可能再沿链向下走，它本身不携带任何能重构出更早版本的信息；
- 真正的版本链从**第一次 UPDATE** 才开始生长。

另有几种"非典型"终止：表已 rebuild/drop 导致 `table_id` 不匹配（静默当链底）、`missing_history`（返回 `DB_MISSING_HISTORY`，上层转成 `HA_ERR_TABLE_DEF_CHANGED`——宁可报错也不返回错误数据）。

### 版本链过长的代价

每一跳的固定开销不小：两次堆分配、一次 `purge_sys->latch` 的 s-lock、一次 mtr + 可能的同步 I/O、一次内存 dup、解析整条 record 并构造 update vector、重新计算 offsets、反向应用。单行建版本成本是 **O(链长)**。

更糟的是它会形成正反馈：

```
长事务/慢查询持有老 read view
  → purge view 无法推进（= 最老活跃 view 的克隆）
  → history 无法回收，版本链变长
  → 一致性读变慢（更多跳、更多随机 I/O、更多 purge latch 争用）
  → 查询更慢，read view 持有更久 → purge 更滞后
```

源码里给出了量化阈值：`BTR_CUR_FINE_HISTORY_LENGTH = 100000`，注释说明"history list 长度超过约 10 万，吞吐就会有可观测的下降"（官方实验值）。

---

### History List

### 组织与入链

history list 是**挂在 rseg header page 上**的物理链表：base node 在 `TRX_RSEG_HISTORY`（trx0rseg.h），每个 undo log header 通过内嵌的 `TRX_UNDO_HISTORY_NODE`（trx0undo.h，12B `flst_node_t`）挂上去。

由于链表存的是 **node 地址**而非 header 地址，需要反向换算（trx0purge.ic）：

```cpp
  node_addr.boffset -= TRX_UNDO_HISTORY_NODE;   // 回退 34B 到 undo log header 开头
```

入链在 `trx_purge_add_update_undo_to_history`（trx0purge.cc）：

```cpp
  trx_rsegf_set_nth_undo(rseg_header, undo->id, FIL_NULL, mtr);  // 归还 slot（非 CACHED）
  mlog_write_ulint(rseg_header + TRX_RSEG_HISTORY_SIZE, hist_size + undo->size, ...);
  flst_add_first(rseg_header + TRX_RSEG_HISTORY,        // ★ 头插
                 undo_header + TRX_UNDO_HISTORY_NODE, mtr);
  trx_sys->rseg_history_len.fetch_add(n_added_logs);
  if (trx_sys->rseg_history_len.load() > srv_n_purge_threads * srv_purge_batch_size)
    srv_wake_purge_thread_if_not_active();
  mlog_write_ull(rseg_header + TRX_RSEG_MAX_TRX_NO, trx->no, mtr);
  mlog_write_ull(undo_header + TRX_UNDO_TRX_NO, trx->no, mtr);   // ★ 写 trx_no
  mlog_write_ulint(undo_header + TRX_UNDO_DEL_MARKS, false, ...); // 无 del_mark 则标记跳过
  if (rseg->last_page_no == FIL_NULL) {   // rseg 从空变非空时初始化进度游标
    rseg->last_page_no = undo->hdr_page_no;
    rseg->last_trx_no = trx->no;
    rseg->last_del_marks = undo->del_marks;
```

**头插 + 尾消费**：`flst_add_first` 使链表从头到尾是 trx_no **递减**（新→旧），purge 从 `flst_get_last()`（尾 = 最老）开始消费。

### rseg_history_len

全局所有 rseg 的 history **log 条数**之和（trx0sys.h，`std::atomic<uint64_t>`），不是页数。维护点：

| 操作 | 位置 |
|------|------|
| 启动扫描按 `flst_get_len(TRX_RSEG_HISTORY)` 累加 | trx0rseg.cc, 405-408 |
| commit 时 `fetch_add(n_added_logs)` | trx0purge.cc |
| 摘除一个 log hdr 时 `fetch_sub(1)` | trx0purge.cc |

一个事务可能同时有 redo rseg 与 temp(noredo) rseg 的 update undo，两者必须用同一 trx_no 整体入队，所以先加 redo 时传 `update_rseg_len=false` 延迟，等 noredo 加完再一次性 `n_added_logs=2`（trx0trx.cc）。

### 有序性：per-rseg 有序，跨 rseg 靠 purge_queue 聚合

**单个 rseg 内严格按 trx_no 单调递增**（从头到尾递减）。保证来源（trx0trx.cc 注释）：

> "We have to hold the rseg mutex because update log headers have to be put to the history list in the (serialisation) order of the UNDO trx number."

**跨 rseg 无序**，聚合由 **purge_queue 小顶堆**完成，key 是 `TrxUndoRsegs::trx_no`（trx0types.h）。一个 rseg 同一时刻最多在堆里出现一次；一个事务的 redo + temp rseg 打包成一个 `TrxUndoRsegs`（最多 2 个 rseg，trx0types.h）。

insert undo **永不进 history list**（commit 后即无用）。

---

### Purge 机制

### Purge 线程模型

线程创建（`srv0start.cc srv_start_purge_threads`）：1 个 coordinator + N-1 个 worker，coordinator 自己也占一个 worker 名额。默认 `innodb_purge_threads=4`，`innodb_purge_batch_size=300`。

`srv_purge_coordinator_thread`（srv0srv.cc）主循环挂起/唤醒；`srv_do_purge`（srv0srv.cc）**自适应线程数**——history 变长则加线程，追上了则减。

`trx_purge_t` 关键字段（trx0purge.h）：

| 字段 | 含义 |
|------|------|
| `view` | purge view（clone_oldest_view） |
| `iter` / `limit` / `done` | 读指针 / 已 purge 边界 / debug 精确位置 |
| `rseg` / `page_no` / `offset` | 下一条要 purge 的 undo rec 位置 |
| `hdr_page_no` / `hdr_offset` | 该 rec 所属 log header |
| `rseg_iter` | `TrxUndoRsegsIterator`，产生 rseg |
| `purge_queue` | 按 trx_no 的小顶堆 |
| `undo_trunc` | undo 表空间 truncate 状态机 |
| `n_submitted` / `n_completed` | 投出/完成的任务数 |

`iter >= limit` 是核心不变式（trx0purge.ic）：`iter` 是"已读到哪"，`limit` 是"已 purge 到哪"，只有 `limit` 能用于 truncate。

### Purge 边界

Purge 的边界是 `purge_sys->view.low_limit_no()`，即 purge view 的 `m_low_limit_no`。检查点在 `trx_purge_fetch_next_rec`：

```
2286:2288:storage/innobase/trx/trx0purge.cc
  // XXXXXXXXX: purge停的位置
  if (purge_sys->iter.trx_no >= purge_sys->view.low_limit_no()) {
    return nullptr;
  }
```

**来源链**：`trx_purge`（trx0purge.cc）入口执行 `trx_sys->mvcc->clone_oldest_view(&purge_sys->view)`（read0read.cc），克隆 MVCC 中最老的活跃 read view。`clone_oldest_view` 从 `m_views` 链表尾部取第一个未关闭的 view（链表按 `m_low_limit_no` 降序，尾部最老）；若无活跃 view 则 `view->prepare(0)` 新建。

`m_low_limit_no = trx_get_serialisation_min_trx_no()`（read0read.cc），即 `trx_sys->serialisation_min_trx_no`——serialisation_list（已提交待 purge 的事务链表）中最小的 `trx->no`。维护在 `trx_add_to_serialisation_list` / `trx_erase_from_serialisation_list_low`（trx0trx.cc）：list 非空时取首元素 no，空时取 `next_trx_id_or_no`。

**正确性论证**（read0read.cc FACT C）：view_list 按 `low_limit_no` 降序，purge 克隆最老 view，故任何活跃 view 的 `low_limit_no` 都 >= purge view 的。`m_low_limit_no` 语义是"view 不需要 `trx_no < 该值` 事务的 undo log"，因此 `trx_no < purge_view.low_limit_no()` 的 undo log 无任何活跃 view 需要，purge 安全。

**为何用 `m_low_limit_no`（trx->no 提交序）而非 `m_low_limit_id`（trx->id 开始序）**：purge 清理已提交事务的 undo 必须按提交序判断。一个 `trx->id` 很小但很晚提交的事务，其 undo 不能被早 purge。

**为何取"最老 view"而非"当前 serialisation_min"**：长期活跃的老 view 创建时 serialisation_min 还很小，仍可能需要 [其创建时 serialisation_min, 当前 serialisation_min) 区间内事务的旧版本。

**GTID 持久化压低边界**：`clone_oldest_view` 末尾调 `view->reduce_low_limit(gtid_oldest_trxno)`（read0read.cc），若 GTID 未刷盘的最老 trx_no 更小则压低边界。

### Buffer Pool Watch 与 purge

Watch 哨兵是 purge 清理 secondary index 上 delete-marked 记录时的关键协作手段，目的是让 purge 不必为操作未读入 buffer pool 的页而阻塞 OLTP。

purge 删二级索引记录走 `BTR_DELETE_OP`，fetch 模式 `Page_fetch::IF_IN_POOL_OR_WATCH`（btr0cur.cc）——页在 BP 就直接用，不在就放 watch 哨兵占位。哨兵由 `Buf_fetch::is_on_watch`（buf0buf.cc）调 `buf_pool_watch_set`（buf0buf.cc）插入 page_hash，注释明确"only by purge thread"。

哨兵放好后 purge 不读盘，而是把删除缓冲到 change buffer：

```cpp
1124:1141:storage/innobase/btr/btr0cur.cc
        if (!row_purge_poss_sec(cursor->purge_node, index, tuple)) {
          cursor->flag = BTR_CUR_DELETE_REF;      // 还有活跃版本引用 → 不能删
        } else if (ibuf_insert(IBUF_OP_DELETE, tuple, index, page_id, page_size, cursor->thr)) {
          cursor->flag = BTR_CUR_DELETE_IBUF;      // ★ 已 buffer 到 change buffer
        } else {
          buf_pool_watch_unset(page_id);
          break;                                   // buffer 失败 → 改读盘
        }
        buf_pool_watch_unset(page_id);
```

**哨兵状态机**：`buf_pool->watch[]`（`BUF_POOL_WATCH_SIZE` = 最大 purge 线程数）。`BUF_BLOCK_POOL_WATCH`（空闲，不在 hash、无 page_id、`zip.data=null`、fix=0）↔ `BUF_BLOCK_ZIP_PAGE`（激活，在 page_hash、有 page_id、fix≥1、`zip.data` 仍 null 伪装压缩页无数据）。转换在 `buf_pool_watch_set`（POOL_WATCH→ZIP_PAGE 入 hash）与 `buf_pool_watch_remove`（buf0buf.cc，反向）。

**`buf0buf.ic` 的 `BUF_BLOCK_POOL_WATCH` 必须 `ut_error`**：`buf_page_in_file`（buf0buf.ic）判断一个 buf_page_t 是否代表真实文件页。空闲哨兵不在任何 hash/list，正常路径拿不到，流到这里是 use-after-free 类 bug；它无 frame/zip.data，当真实页操作必崩；而激活态 ZIP_PAGE 故意能通过（在 hash 会被查到），区分激活哨兵 vs 真实压缩页靠专门的 `buf_pool_watch_is_sentinel`（buf0buf.cc，检查地址在 watch[] 范围 + zip.data==null）。

### trx_purge 主循环

```cpp
// trx0purge.cc
  srv_dml_needed_delay = trx_purge_dml_delay();               // ① 算 DML 限流
  ut_a(purge_sys->n_submitted == purge_sys->n_completed);     // 上批必须已收尾
  trx_sys->mvcc->clone_oldest_view(&purge_sys->view);         // ② 克隆最老 ReadView
  n_pages_handled = trx_purge_attach_undo_recs(n_purge_threads, batch_size);  // ③ 批量读+分组
  srv_que_task_enqueue_low(thr);                              // ④ 投递给 worker
  que_run_threads(thr);                                       // ⑤ coordinator 自己也干一份
  trx_purge_wait_for_workers_to_complete();                   // ⑥ 等 worker
  node->free_lob_pages();                                     // ⑦ 批末统一释放 LOB 首页
  trx_purge_truncate();                                       // ⑧ 截断 history
```

purge 查询图（`trx_purge_graph_build`，trx0purge.cc）：一个 worker 对应一个 `que_thr_t`，每个 thr 挂一个 `purge_node_t`（`QUE_NODE_PURGE`）。

### rseg 遍历：TrxUndoRsegsIterator

`TrxUndoRsegsIterator::set_next()`（trx0purge.cc）决定下一个处理哪个 rseg：

```cpp
  if (m_iter != m_trx_undo_rsegs.end()) {
    m_purge_sys->iter.trx_no = (*m_iter)->last_trx_no;   // 同一事务的下一个 rseg（redo→temp）
  } else if (!m_purge_sys->purge_queue->empty()) {
    m_trx_undo_rsegs = purge_sys->purge_queue->top();     // 取 trx_no 最小
    // trx_no 相同的连续元素合并（redo + temp 打包）
    m_iter = m_trx_undo_rsegs.begin();
  m_purge_sys->rseg = *m_iter++;
  m_purge_sys->iter.trx_no = m_purge_sys->rseg->last_trx_no;   // 推进读指针
  m_purge_sys->hdr_offset  = m_purge_sys->rseg->last_offset;
  m_purge_sys->hdr_page_no = m_purge_sys->rseg->last_page_no;
```

rseg 只在消费完一个 undo log 后以新的 `last_trx_no` 重新 push 回堆，这保证"老的先 purge"。

### trx_purge_attach_undo_recs 与分组

```cpp
// trx0purge.cc
  purge_sys->limit = purge_sys->iter;        // 上批已 purge 完，limit 追上 iter
  while (n_pages_handled < batch_size) {
    if (trx_purge_check_limit()) purge_sys->limit = purge_sys->iter;
    rec.undo_rec = trx_purge_fetch_next_rec(&rec.modifier_trx_id, &rec.roll_ptr,
                                            &n_pages_handled, heap);
    if (rec.undo_rec == &trx_purge_ignore_rec) continue;   // 整组可跳过
    else if (rec.undo_rec == nullptr) break;
    purge_groups.add(rec);
  purge_groups.distribute_if_needed();
  purge_groups.assign(run_thrs);
```

`n_pages_handled` 累加两处：每消费完一个 log header +1、purge 指针跨到新页 +1。所以 `innodb_purge_batch_size` 单位是"处理的 undo 页数"。

`trx_purge_get_next_rec`推进 iter/page_no/offset。注意它会跳过不需要 purge 的记录：只取 `DEL_MARK_REC`、有 extern storage、或 `UPD_EXIST_REC` 且 `!(cmpl_info & UPD_NODE_NO_ORD_CHANGE)` 三类。

**分组粒度是 `table_id`**（`Purge_groups_t::add`，trx0purge.cc）：同一张表的 undo rec 一定落在同一 worker，**不同 worker 不会并发 purge 同一张表**，避免跨线程同表 B-tree 争用。`find_smallest_group()` 做负载均衡；`distribute_if_needed()` 只在 purge lag 超限时才 rebalance。

**last_del_marks 优化**（`trx_purge_read_undo_rec`）：若该 undo log 无 delete-mark，`offset=0` → 走 dummy rec 分支，整组 undo log 被跳过，不逐条解析。

### 单条 undo rec 的 purge

入口 `row_purge_step`（row0purge.cc）→ `row_purge`→ `row_purge_parse_undo_rec`→ `row_purge_record_func`。

`row_purge_parse_undo_rec` 的跳过优化：

```cpp
  if (type == TRX_UNDO_UPD_DEL_REC && !*updated_extern) return (false);  // 无需 purge
 if (type == TRX_UNDO_UPD_EXIST_REC && (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE)
      && !*updated_extern) goto close_exit;                            // 无需 purge
```

`row_purge_record_func` 三种操作：

| 类型 | 处理 | 位置 |
|------|------|------|
| `TRX_UNDO_DEL_MARK_REC` | `row_purge_del_mark` | row0purge.cc |
| `TRX_UNDO_UPD_EXIST_REC` / extern | `row_purge_upd_exist_or_extern_func` | :702 |

`row_purge_del_mark` 顺序：**先所有二级索引 → 最后聚簇索引**（聚簇记录是版本根，必须先保证二级索引条目已判可删）：

```cpp
  while (node->index != nullptr) {
      row_purge_remove_sec_if_poss(node, node->index, entry);
    node->index = node->index->next();
  return (row_purge_remove_clust_if_poss(node));
```

`row_purge_remove_clust_if_poss_low`的关键保护——**roll_ptr 变了就不能删**（记录已被后续事务修改）：

```cpp
  if (node->roll_ptr != row_get_rec_roll_ptr(rec, index, offsets)) {
    /* Someone else has modified the record later: do not remove */
    goto func_exit;
  if (mode == BTR_MODIFY_LEAF) success = btr_cur_optimistic_delete(...);
    btr_cur_pessimistic_delete(...);     // 乐观失败转悲观
```

`row_purge_remove_sec_if_poss`两条路径：乐观 leaf（带 `BTR_DELETE` 可走 ibuf）→ 失败转悲观 tree。

**`row_purge_poss_sec` 是安全闸门**：

```cpp
  can_delete =
      !row_purge_reposition_pcur(BTR_SEARCH_LEAF, node, &mtr) ||        // 聚簇记录都没了 → 可删
      !row_vers_old_has_index_entry(true, node->pcur.get_rec(), &mtr, index,
                                    entry, node->roll_ptr, node->trx_id);
```

`row_vers_old_has_index_entry`（row0vers.cc）沿 undo 版本链回溯，找是否存在"未被 delete-mark 且能构造出相同二级索引条目"的版本：

```cpp
  if (entry && dtuple_coll_eq(entry, ientry)) return true;   // 当前版本匹配 → 不能删
  trx_undo_prev_version_build(...);                          // 沿版本链回溯
  if (!prev_version) return false;                           // 链走完都没匹配 → 可删
  if (!rec_get_deleted_flag(prev_version, comp)) {
    if (entry && dtuple_coll_eq(entry, ientry)) return true; // 某历史版本匹配 → 不能删
```

判定要点：用 **`dtuple_coll_eq`（collation 比较而非二进制比较）**；只检查未 delete-mark 的版本；版本链顶端由 mtr 页 latch 锁住，底端由 purge view 保证。

### Purge 与 change buffer 的协同

purge 是**唯一**使用 `BTR_DELETE` latch mode 的调用者（btr0cur.cc `ut_a(cursor->purge_node)`）。关键协同点：

1. purge 用 `IF_IN_POOL_OR_WATCH` 而非普通 `IF_IN_POOL`：latch 顺序要求——watch 保证在 ibuf B-tree 页 latch 期间目标页不会"悄悄"被读入 BP（否则 `row_purge_poss_sec` 判定与后续 merge 会 race）。
2. **必须先 `row_purge_poss_sec()` 通过才允许 buffer**（btr0cur.cc），否则会 buffer 掉仍被引用的条目 → 数据错误。
3. `IBUF_OP_DELETE` 只有在该页**已有 ≥2 条 buffered insert/delete-mark** 时才允许 buffer（防页面被清空，ibuf0ibuf.cc）；且它走 `skip_watch` 分支，因为调用方已自行 `buf_pool_watch_set`。
4. merge 时 `ibuf_delete()`（ibuf0ibuf.cc）执行真正删除。

### History list truncate

`trx_purge_truncate`（trx0purge.cc）→ `trx_purge_truncate_history`遍历所有 rseg → `trx_purge_truncate_rseg_history`从**最老端**逐个摘除：

```cpp
  hdr_addr = trx_purge_get_log_from_hist(flst_get_last(rseg_hdr + TRX_RSEG_HISTORY, &mtr));
  undo_trx_no = mach_read_from_8(log_hdr + TRX_UNDO_TRX_NO);
  if (undo_trx_no >= limit->trx_no) {          // 到边界，停止
    trx_undo_truncate_start(rseg, hdr_addr.page, hdr_addr.boffset, limit->undo_no);  // 部分截断
    return;
  if ((mach_read_from_2(seg_hdr + TRX_UNDO_STATE) == TRX_UNDO_TO_PURGE) &&
      (mach_read_from_2(log_hdr + TRX_UNDO_NEXT_LOG) == 0)) {
    trx_purge_free_segment(rseg, hdr_addr, is_temp);   // 整段释放
    trx_purge_remove_log_hdr(rseg_hdr, log_hdr, &mtr); // 只摘 hdr（CACHED 场景）
  goto loop;                                   // 继续向"更新"方向截
```

truncate 上限保守地钳制在 purge view：`if (limit->trx_no >= view->low_limit_no()) limit->trx_no = view->low_limit_no()`。

整段释放 `trx_purge_free_segment`：先清 `TRX_UNDO_DEL_MARKS`（防崩溃后重复访问尾部），再**同 mtr** 内摘 hdr + free 段（注释解释为何必须同 mtr：否则崩溃后段可能变成不可访问的垃圾）。

undo 表空间 truncate（`trx_purge_truncate_undo_spaces`）：每批最多截 1 个，且必须等 `last_page_no == FIL_NULL && trx_ref_count == 0`（即 history 全部清完且无事务引用）。

### Purge 滞后与限流

`trx_purge_dml_delay`（trx0purge.cc）：

```cpp
  if (srv_max_purge_lag > 0 && trx_sys->rseg_history_len.load() >
                                  srv_n_purge_threads * srv_purge_batch_size) {
    ratio = float(trx_sys->rseg_history_len.load()) / srv_max_purge_lag;
    if (ratio > 1.0) delay = (ulint)((ratio - 0.9995) * 10000);   // µs
    if (delay > srv_max_purge_lag_delay) delay = srv_max_purge_lag_delay;
```

延迟落地在 `row_mysql_delay_if_needed`（row0mysql.cc），每次 DML 前 sleep。默认 `innodb_max_purge_lag=0` → 不限流。

history list length 监控：`SHOW ENGINE INNODB STATUS` 的 "History list length"（lock0lock.cc）、monitor `MONITOR_RSEG_HISTORY_LEN`（srv0mon.cc）。

---

### DDL 与 Undo

DROP TABLE 和 TRUNCATE TABLE **不记录用户数据的 undo log**——它们不像 `DELETE FROM t` 那样为每一行产生 undo record。它们保证原子性和可恢复性靠的是 DDL log 和数据字典元数据的 undo。

### 为什么不记数据 undo

DDL 是隐式提交的，DDL 语句本身是一个原子事务，用户层面不可回滚。DROP/TRUNCATE 是物理操作（删表空间文件、free btree root），不是逐行 delete，没有逐行 undo 的产生点。被删或被清空的表数据不需要 MVCC 旧版本——表都没了或被重建了，没有 read view 会访问旧数据。

### DDL log 机制（8.0 原子 DDL）

InnoDB 8.0 引入 `mysql.innodb_ddl_log` 表和 `Log_DDL` 类（log0ddl.cc），配合 `HTON_SUPPORTS_ATOMIC_DDL` 实现原子 DDL。DROP TABLE 在 `row_drop_table_for_mysql` 中写入三类 DDL log：

```
  err = log_ddl->write_free_tree_log(trx, index, true);    // 释放索引 btree（共享表空间的表）
  log_ddl->write_drop_log(trx, table_id);                   // 记录 drop（table_id，清理 dict cache）
  err = log_ddl->write_delete_space_log(trx, nullptr, space_id, filepath, true, true);  // 删 .ibd
```

DDL log 通过 `DDL_Log_Table::insert` 向 `mysql.innodb_ddl_log` 表 insert 记录。`mysql.innodb_ddl_log` 是普通 InnoDB 表，insert 产生 undo + redo。注意这个 undo 不是为了"回滚 DROP TABLE 的数据"，而是为了 DDL 事务失败时回滚对 `innodb_ddl_log` 表本身的修改——这是原子 DDL 的元数据层面保证。

除了 DDL log，DROP/TRUNCATE 还修改 8.0 的数据字典（`dd::Table` 等存储在 InnoDB 系统表 `mysql.tables` 中）。这些元数据修改同样通过普通事务执行，产生 undo + redo。

### DDL log 生命周期

DDL 执行期间（事务未提交），向 `innodb_ddl_log` 表写入物理操作记录。此时物理操作本身并不执行，只记录意图。`insert_free_tree_log` 里注释说得很明确："if committed, will be redo only"（log0ddl.cc）——free tree 操作要等 DDL 事务提交后通过 redo（DDL log）重放才执行。

DDL 事务提交后进入 post-DDL 阶段，`Log_DDL::post_ddl(thd)`（log0ddl.cc）被调用，`replay_by_thread_id` 按线程 ID 查找该 DDL 写入的所有记录，`replay`（log0ddl.cc）根据记录类型执行真正的物理操作——`replay_free_tree_log` 实际 free 索引 btree、`replay_delete_space_log` 实际删除 .ibd 文件、`replay_drop_log` 从 dict cache 移除。replay 完成后 `delete_by_ids` 删除 `innodb_ddl_log` 中已处理的记录。

崩溃恢复时，`Log_DDL::recover`（log0ddl.cc）调用 `replay_all()` 重放所有残留的 DDL log 记录。DDL 事务已提交则 DDL log 记录存在，replay 确保物理操作完成；DDL 事务未提交则 undo 回滚该事务，`innodb_ddl_log` 的 insert 被 undo 撤销，无 DDL log 可 replay，表恢复原状。

原子 DDL 的本质：物理操作通过 DDL log 延迟到提交后执行，元数据修改通过 undo 可回滚，两者结合保证 DDL 要么完全成功，要么完全回滚。

### TRUNCATE TABLE：rename + drop + create

TRUNCATE 在 InnoDB 中走 `HTON_CAN_RECREATE` 路径（ha_innodb.cc），本质是 drop + recreate。从 `innobase_truncate::truncate()`（ha_innodb.cc）看三步：先 `rename_tablespace()`把旧 .ibd 重命名为临时名，再 `innobase_basic_ddl::delete_impl`删除旧表（走 `row_drop_table_for_mysql`，写 DROP 的 DDL log），最后 `innobase_basic_ddl::create_impl`创建新空表（写 CREATE 的 DDL log）。

TRUNCATE 先 rename 旧表空间文件而非直接删，这样即使 create 失败旧文件还在（以临时名存在），有恢复余地。TRUNCATE 的"undo"= DROP 阶段的 DDL log + CREATE 阶段的 DDL log + 两阶段的 dd 元数据 undo，全部在同一个 DDL 事务内，提交后由 post_ddl 统一 replay。

---

### GTID 持久化与 Undo

InnoDB 的 GTID 持久化与 undo log 紧密耦合，有三条持久化路径：binlog 文件（Gtid_log_event）/ binlog rotate 刷表 / InnoDB undo log 异步刷表。

**InnoDB GTID 持久化**：事务 commit 阶段同时将 GTID 写入 undo log header（随 redo 持久化，保证崩溃不丢）+ 写入内存 list（后台线程异步刷 `mysql.gtid_executed` 表）。后台线程从内存 list 读取 GTID 刷表，不从 undo log 解析；undo log 只在崩溃恢复时解析。

**Clone_persist_gtid 双 buffer**：`m_gtids[2]` 奇偶切换，事务写 active list，后台线程读 flush list。

**purge 与 GTID 的协同**：purge 边界会被 GTID 持久化进度压低。`clone_oldest_view` 末尾调用 `view->reduce_low_limit(gtid_persistor.get_oldest_trx_no())`，若还有事务的 GTID 未持久化到 `mysql.gtid_executed` 表，它的 trx_no 不能被 purge 越过——因为 purge 会删 undo log，而 GTID 持久化依赖从 undo log header 读取 GTID。

补充两点：一是**为什么 GTID 可以写进 undo header**——undo log header 里有专门的 XID 区与 GTID 区（`TRX_UNDO_FLAG_GTID` / `TRX_UNDO_FLAG_XA_PREPARE_GTID` 两个标志位区分普通 GTID 与 XA prepare GTID），写入时若空间不够会由 `trx_undo_header_add_space_for_xid` 把 header 往后扩展。二是**崩溃恢复时如何补回**：启动时扫描 rseg 的 undo slot，对每个非 CACHED 的 update undo 调 `trx_undo_gtid_read_and_persist`，从 undo header 里把 GTID 读出来补刷到 `mysql.gtid_executed`。完整链路与 `Gtid_set` 的区间表示见 [`../server/replication/gtid.md`](../server/replication/gtid.md)。

---

### Undo 与 Clone / 备份

备份与 undo 的关系分两类：**逻辑备份**通过持有 read view 间接钉死 purge 边界；**物理备份**（Clone / MEB）则必须把 undo 表空间作为数据集的一部分带走。两者机制完全不同，故障表现也不同。

### 逻辑备份：一个 read view 拖住整个 purge

`mysqldump --single-transaction` 会发 `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ` + `START TRANSACTION WITH CONSISTENT SNAPSHOT`（`client/mysqldump.cc:5300-5306`），服务端由 `innobase_start_trx_and_assign_read_view` 承接，最终 `trx_assign_read_view()` 把 view 挂进 `trx_sys->mvcc`。

后果是连锁的，因为 purge 每个批次都重新克隆最老 view：

```
mysqldump 的 read view 挂着
  → purge 每批次 clone_oldest_view() 取到它        trx0purge.cc:2495 → read0read.cc:726
  → purge_sys->view.low_limit_no() 被钉死
  → trx_purge_fetch_next_rec() 直接返回 nullptr    trx0purge.cc:2286-2288
  → trx_purge_truncate_history() 上限也被夹住      trx0purge.cc:1645-1652
  → rseg_history_len 只增不减                      trx0purge.cc:399 / :438
```

三点值得强调：

1. **`innodb_max_purge_lag` 不是解法，只是背压**。它按 `ratio = history_len / max_purge_lag` 给 DML 注入 sleep（`trx0purge.cc:2396-2426` → `row0mysql.cc:140-145`），但**不会解除 read view 对 purge 的阻塞**。备份期间 history 照涨，只是 DML 被拖慢。
2. **生产构建里没有 history 过长的告警**。那段 "Purge reached the head of the history list..." 的告警包在 `#ifdef UNIV_DEBUG` 里（`trx0purge.cc:1785-1807`），非 debug 构建看不到。
3. **超过 10 万会有可观测的吞吐下降**（`BTR_CUR_FINE_HISTORY_LENGTH = 100000`，`btr0cur.cc:115-124`），此时 purge 的删除操作会强制抢占 `index->lock`。

### Clone：undo 表空间随数据一起走

Clone 枚举 donor 的所有持久化表空间时**不过滤 undo**：

```cpp
// storage/innobase/clone/clone0copy.cc:230-236
/* Iterate all tablespace files and add persistent data files. */
auto error = Fil_iterator::for_each_file(
    [&](fil_node_t *file) { return (add_node(file, false)); });
```

`Clone_Snapshot::add_node()` 只跳过 `space->is_deleted()`，没有 `fsp_is_undo_tablespace` 过滤（`clone0copy.cc:865-871`）。recipient 侧还专门记录了 undo 文件的 index（`m_undo_file_indexes`，`clone0snapshot.h:1017`），并强制把 undo 文件放到自己的 `srv_undo_dir`（`clone0apply.cc:370-375`），按 `UNDO_INITIAL_SIZE_IN_PAGES` 预建（`clone0apply.cc:1075-1078`）——所以 **donor 与 recipient 的 undo 目录不必相同**。

一致性位点**不是单一的 "start LSN"**，而是一个区间：FILE COPY 开始启动 page tracking（`clone0copy.cc:207-210`），PAGE COPY 开始启动 redo 归档（`clone0copy.cc:261-263`），REDO COPY 时 `m_redo_ctx.stop()` 确定终点（`clone0copy.cc:517-522`）。

### Clone 不阻塞 purge（一个常见误解）

在 `storage/innobase/clone/` 下没有任何 `trx_purge_stop` 或 history 保护代码——**clone 期间 purge 照常运行**。一致性不靠冻结 purge，而靠两个补偿机制：

1. FILE COPY 期间被修改的页由 page archiver 记录，PAGE COPY 阶段重传（含 purge 改过的 undo 页）；
2. PAGE COPY 起 redo 归档，REDO COPY 把这段 redo 传走，recipient 重放到终点 LSN。

clone 与 undo 真正的耦合在 **GTID** 和 **truncate** 两处。

### Clone 必须等 GTID 落表：非 InnoDB 事务没有 undo 载体

`Clone_Snapshot::init_redo_copy()` 在停止 redo 归档前会 `synchronize_binlog_gtid()`（`clone0copy.cc:504` → `:332-348`），其中 `gtid_persistor.wait_flush(true, ...)` 强制刷一次。原因是：

```cpp
// storage/innobase/clone/clone0repl.cc:366-372
int Clone_persist_gtid::write_other_gtids() {
  int err = 0;
  if (opt_bin_log) {
    err = gtid_state->save_gtids_of_last_binlog_into_table();
  }
  return (err);
}
```

**非 InnoDB 事务的 GTID 不存在于任何 undo header 里**，重启时 `trx_rseg_persist_gtid` 扫不到。若 clone 前不落表就永久丢失。此外 XA 的 GTID 可能"先于 InnoDB 持久化"进入 `gtid_executed`，所以还要 `Clone_handler::XA_Block` 挡住外部 XA（`clone0copy.cc:485`）。

这与 purge 侧形成闭环：commit 把 GTID 写进 undo header 并加入 `Clone_persist_gtid` 的双 buffer → purge view 被 `reduce_low_limit(get_oldest_trx_no())` 压住不能越过（`read0read.cc:746-747`）→ 后台线程 flush 到表后 `m_gtid_trx_no` 前进并 `srv_purge_wakeup()`（`clone0repl.cc:473-476`）→ purge 恢复。**clone 只需保证在 REDO COPY 终止前 flush 一次，就把"表"和"undo"两条 GTID 载体对齐了。**

### 为什么 undo DDL 要单独通知 Clone

`Clone_notify::Type` 里 `SPACE_UNDO_DDL` 的注释说明了原因（`clone0api.h:201-203`）：

> Special consideration is needed for UNDO as these DDLs don't use DDL log and needs special consideration during recovery.

即 **undo 相关的 DDL 不走 DDL log**（见「DDL 为何不记数据 undo」），因此崩溃恢复没有 DDL log 可 replay，必须靠 clone 侧的显式通知来协调。

`CREATE UNDO TABLESPACE` 会发 `Clone_notify(SPACE_UNDO_DDL, ...)`（`ha_innodb.cc:16089-16096`），而 **DROP UNDO TABLESPACE 不通知**——因为 drop 走 DDL log，可恢复。

`block_state_change()` 里有个精妙的处理（`clone0snapshot.cc:1390-1402`）：undo space 的普通 file create/drop 通知**不阻塞 clone**（避免 DDL 内部递归通知自锁，外层 `SPACE_UNDO_DDL` 已挡住）；而 `no_wait=true` 是专门给 **undo truncate 后台线程**用的——宁可让 truncate 失败重试，也绝不能把 purge 线程卡住。

### undo truncate 与 Clone 的两道拦截

| 拦截点 | 位置 | 逻辑 |
|--------|------|------|
| ① 进入 truncate 前 | `trx0purge.cc:1594` | `clone_check_provisioning()` 为真（本地克隆 provisioning 中）→ 放弃并打 `ER_IB_MSG_UNDO_TRUNCATE_DELAY_BY_CLONE` |
| ② 写完 trunc.log 后、真正重建前 | `trx0purge.cc:1482-1488` | `Clone_notify(SPACE_UNDO_DDL, no_wait=true)`，failed 则放弃本次 truncate |

冲突根源：`trx_undo_truncate_tablespace()` 会**删原文件并改 space_id**，而 clone 是按 file index 直接 open + read 文件的，文件删改会让 index 与磁盘错位。

### 克隆实例启动：history list 完整保留

识别克隆库靠 redo 文件头的 creator 字段：

```cpp
// storage/innobase/log/log0log.cc:1588-1594
} else if (str_starts_with(creator_name, LOG_HEADER_CREATOR_CLONE)) {
  recv_sys->is_cloned_db = true;
```

启动顺序上 `clone_init()` 先于 `fil_scan_for_tablespaces()`（`srv0start.cc:1692` vs `:1698`）。与 undo 相关的差异：

- **history list 原样带过来**，`trx_rseg_physical_initialize()` 从 rseg header 的 `TRX_RSEG_HISTORY` 重建并累加 `rseg_history_len`（`trx0rseg.cc:267-270`），purge 从链表**尾（最老）**继续（`trx0rseg.cc:275-276, 313`）
- **GTID 从 undo 恢复**：先读 TRX_SYS 的水位 `TRX_SYS_TRX_NUM_GTID`（`trx0rseg.cc:576`），再从 history list 头往回扫直到 `undo_trx_no < 水位`，逐个 `trx_undo_gtid_read_and_persist()`（`trx0rseg.cc:159-213`）
- 克隆库不做 IBUF merge（`srv0start.cc:2036`）、跳过 dblwr 恢复（`buf0dblwr.cc:3155`）、忽略 redo 里的加密信息
- 恢复完成后写 `#clone/#status_recovery` 供 clone plugin 汇报（`clone0api.cc:1441`）

### 物理备份（MEB）必须带 undo 表空间

MEB 是物理备份，`apply-log` 本质是一次 crash recovery，因此 undo 表空间是必需组成。源码侧的硬约束：

```cpp
// storage/innobase/srv/srv0start.cc:724-729
/* Open existing undo tablespaces up to the number in target_undo_tablespace.
...
If we fail to open any of these it is a fatal error. */
```

**若物理备份不含 undo 文件，且 `innodb_force_recovery < 5`，实例直接起不来。** 特别要注意 `innodb_undo_directory` 若指向 datadir 之外（常见的"undo 放独立盘"），备份工具必须额外采集该目录。

MEB 的 redo 归档起点是 **checkpoint LSN**：

```cpp
// storage/innobase/log/log0meb.cc:1687-1691
log_meb_consumer = std::make_unique<Log_user_consumer>("MEB");
log_meb_consumer->set_consumed_lsn(log_get_checkpoint_lsn(log));
log_consumer_register(log, log_meb_consumer.get());
```

注册一个 consumer 等价于把 checkpoint 之后的 redo 文件"钉住"不让回收（`log0files_governor.cc:795-806`）。从更小 LSN 起步文件可能已被重写，从更大起步则缺 redo——所以必须是 checkpoint。Clone 的归档同理（`arch0log.cc:305-310` 先强制 checkpoint 再取 `last_checkpoint_lsn`）。

**redo 归档不阻塞 purge**，它只影响 redo 文件回收，与 history list 完全正交。

### 关机与 force_recovery 对 undo 的影响

| 场景 | 行为 |
|------|------|
| `innodb_fast_shutdown=0` | 关机循环 purge 直到 0 页（`srv0srv.cc:3157`），等回滚完成 |
| `=1`（默认） | purge 直接退出，重启走 crash recovery；**history 会保留**，启动后往往观察 history 很高 |
| `=2` | 连 buffer pool 都不 flush，必然 crash recovery |
| `force_recovery >= 2` | **purge 完全停摆**（`ha_innodb.cc:4190-4193` 置 `PURGE_STATE_DISABLED`），history 只增不减 |
| `>= 3` | 不回滚恢复事务（`trx0roll.cc:718` 的 `ut_a` 断言会拦住调用） |
| `>= 5` | 完全不扫 undo（`trx0sys.cc:450-475` 不建 purge_queue），只读，是"dump 出来重建实例"的最后手段 |

另需注意：关机时会抑制 undo truncate（`trx0purge.cc:1265-1267` 的 `in_fast_shutdown` 判断）。

### 监控

- `SHOW ENGINE INNODB STATUS` 的 History list length——**唯一输出点**在 `lock0lock.cc:4722`，取自 `trx_sys->rseg_history_len`
- `information_schema.INNODB_METRICS` 的 `trx_rseg_history_len`（默认开启，`srv0mon.cc:786-790`）、`purge_dml_delay_usec`、`purge_undo_log_pages`
- PFS 里没有 history list 表；purge 年龄类指标（`innodb_purge_trx_id_age` 等）仅 debug 构建才有（`srv0srv.cc:1744-1776`）

---

## Misc

### purge 与 commit order 的关系

事务 commit 阶段分配 `trx->no`（`trx_add_to_serialisation_list`，trx0trx.cc），决定 purge 顺序与 MVCC read view 可见性。`binlog_order_commits=ON` 时 `trx->no` 分配序 == binlog 序；OFF 时各线程抢 mutex 分配顺序不保证。Clone/一致性快照依赖 `trx->no` 序 == binlog 序取一致 snapshot。

### 关键不变式速查

1. history list **头插、尾消费**：头是最新，purge 从最老往新走。
2. **per-rseg trx_no 有序**由 rseg latch 保证；**跨 rseg 顺序**由 purge_queue 小顶堆聚合。
3. **`iter >= limit`** 永远成立；只有 `limit` 能用于 truncate。
4. **一个 rseg 同时在堆里最多一个 entry**；消费完一个 log 才以新 `last_trx_no` push 回。
5. **purge 分组粒度是 table_id**：同表 rec 在同一 worker 串行。
6. **`row_purge_poss_sec` 是安全闸门**：必须在持有二级索引 leaf latch 或 buffer pool watch 时调用。
7. **purge 只处理 update undo**；insert undo 不进 history。
8. undo 表空间 truncate 每批最多 1 个，且必须等 history 清空且无事务引用。

**两个容易遇到的报错**：

- `ER_UNDO_RECORD_TOO_BIG`（内部 `DB_UNDO_RECORD_TOO_BIG`）：单条 undo record 连一个空页都放不下。常见于单行更新超大的 BLOB/TEXT，或 `innodb_page_size` 设得很小却存大字段。它是设计上的硬限制——undo record 不支持跨页部分写。
- `HA_ERR_TABLE_DEF_CHANGED`（内部 `DB_MISSING_HISTORY`）：一致性读要回溯的历史版本已被 purge 回收。正常配置下不该出现，出现通常意味着回滚段空间不足被提前覆盖，或 `innodb_force_recovery` 跳过了 undo 扫描。设计取向是**宁可报错也不返回错误数据**。

**长事务的三重放大效应**：一个长事务同时造成——purge 边界无法推进（history 堆积）、版本链变长（一致性读变慢）、undo 表空间膨胀。这三者互相加强，是 undo 相关故障最常见的根因，排查时优先看 `SHOW ENGINE INNODB STATUS` 的 History list length 与 `information_schema.innodb_trx` 里最老的事务。

---

## 关键源码位置速查

> 仅作本地检索用。本工作区源码含中文研读批注，部分行号与官方 8.0.39 有偏移，以函数名为准。

**Undo Tablespace 与 DDL**

| 位置 | 说明 |
|------|------|
| `sql/sql_tablespace.cc:1380` | `Sql_cmd_create_undo_tablespace::execute` |
| `ha_innodb.cc:16436` | `innobase_alter_tablespace` 三分派 |
| `ha_innodb.cc:16069` / `:16178` / `:16319` | create / set inactive / drop undo tablespace |
| `ha_innodb.cc:16089` | CREATE UNDO 的 `Clone_notify(SPACE_UNDO_DDL)` |
| `srv0start.cc:1070` / `:224` | `srv_undo_tablespace_create`（DDL 入口 / 建文件） |
| `srv0start.cc:913` | `srv_undo_tablespaces_construct`（写 page 0 + page 3） |
| `trx0purge.h:666` / `:314` | `undo::Tablespaces` / `undo::Tablespace` |
| `trx0purge.h:163-252` | `is_reserved` / `num2id` / `id2num` |
| `trx0purge.cc:661-762` | space_id_bank 分配 |
| `fsp0types.h:153-186` | 页号常量（`FSP_RSEG_ARRAY_PAGE_NO=3`） |
| `trx0rseg.h:231-263` | RSEG_ARRAY 页布局 |
| `trx0rseg.cc:1135` / `:60` | `trx_rseg_array_create` / `trx_rseg_header_create` |
| `trx0undo.cc:2049` | `trx_undo_truncate_tablespace`（换 space_id 重建） |

**rseg 与 undo segment**

| 位置 | 说明 |
|------|------|
| `trx0types.h:180` | `trx_rseg_t`（注意：不在 trx0rseg.h） |
| `trx0rseg.h:189-227` | rseg header 常量（N_SLOTS / HISTORY / UNDO_SLOTS） |
| `trx0undo.h:341` | `trx_undo_t` |
| `trx0undo.cc:394` / `:924` / `:1246` | seg_create / add_page / seg_free |
| `trx0undo.cc:1697` | `trx_undo_assign_undo` |
| `trx0undo.cc:1558` / `:1617` | `trx_undo_create` / `trx_undo_reuse_cached` |
| `trx0undo.cc:1808` / `:1849` | `set_state_at_finish` / `set_state_at_prepare` |
| `trx0trx.cc:1167` | `get_next_redo_rseg_from_undo_spaces`（round-robin） |
| `trx0rseg.cc:617` | `trx_rsegs_init`（启动扫描） |

**Undo Page 与 Record**

| 位置 | 说明 |
|------|------|
| `trx0undo.h:466-525` | PAGE_HDR / SEG_HDR 常量 |
| `trx0undo.h:527-567` | LOG_HDR 常量（含 `HISTORY_NODE=34`） |
| `trx0undo.cc:369` / `:499` / `:870` | page_init / header_create / insert_header_reuse |
| `trx0undo.ic:151-192` | `trx_undo_page_get_start/end` |
| `trx0rec.h:299-320` | record type 常量（11/12/13/14） |
| `trx0rec.h:330-365` | `type_cmpl_t` 位域 |
| `trx0rec.ic:37-80` | `get_type` / `get_cmpl_info` / `get_undo_no` |
| `trx0rec.cc:182` | `set_next_prev_and_add`（next/prev 写入） |
| `trx0rec.cc:485` / `:1159` | `report_insert` / `report_modify` |
| `trx0rec.cc:556` / `:1720` / `:1941` | get_pars / sys_cols / partial_row |
| `trx0undo.ic:39-93` | roll_ptr 编解码（56 位布局） |
| `trx0rec.cc:2190` | `trx_undo_report_row_operation`（生成总入口） |
| `trx0rec.cc:142` | `trx_undo_left`（页空间检查） |

**回滚 / 版本链 / History List**

| 位置 | 说明 |
|------|------|
| `trx0roll.cc` | `trx_rollback_*` 系列、`trx_roll_pop_top_rec_of_trx` |
| `row0undo.cc` | `row_undo` / `row_undo_mod` 总控 |
| `row0uins.cc:326` | `row_undo_ins`（直接物理删） |
| `trx0rec.cc:2469` | `trx_undo_prev_version_build`（单跳引擎） |
| `trx0rec.cc` | `trx_undo_get_undo_rec`（purge view 守门） |
| `row0vers.cc` | `row_vers_build_for_consistent_read` / `old_has_index_entry` |
| `trx0purge.cc:344` | `trx_purge_add_update_undo_to_history` |
| `trx0purge.ic:39` | `trx_purge_get_log_from_hist`（node→header） |
| `trx0sys.h:480` | `trx_sys->rseg_history_len` |
| `trx0types.h:603` | `purge_pq_t` 小顶堆 |

**Purge**

| 位置 | 说明 |
|------|------|
| `srv0srv.cc:3094` / `:2907` / `:2850` | coordinator / do_purge / worker |
| `trx0purge.h:986` / `:118` | `trx_purge_t` / `purge_iter_t` |
| `trx0purge.ic:52` | `trx_purge_check_limit`（iter >= limit） |
| `trx0purge.cc:2474` | `trx_purge` 主循环 |
| `trx0purge.cc:107` | `TrxUndoRsegsIterator::set_next` |
| `trx0purge.cc:2309` / `:2050` | `attach_undo_recs` / `Purge_groups_t` |
| `trx0purge.cc:2265` / `:2286` | `fetch_next_rec` / 停止条件 |
| `trx0purge.cc:1928` / `:1747` | `get_next_rec` / `rseg_get_next_history_log` |
| `trx0purge.cc:2456` / `:1636` / `:533` | truncate 三级 |
| `trx0purge.cc:446` / `:433` / `:2396` | free_segment / remove_log_hdr / dml_delay |
| `row0purge.cc:1233` / `:1172` / `:856` / `:1096` | purge_step / row_purge / parse / record_func |
| `row0purge.cc:657` / `:702` / `:275` | del_mark / upd_exist / poss_sec |
| `read0read.cc:726` / `:448` / `:128` | clone_oldest_view / prepare / FACT C |
| `trx0trx.cc:1501` / `:1527` | serialisation_list 增删 |
| `buf0buf.ic:346` | `buf_page_in_file`（POOL_WATCH → ut_error） |
| `buf0buf.cc:2926` / `:3028` / `:2891` | watch_set / watch_remove / is_sentinel |
| `btr0cur.cc:1118-1141` | purge 删 sec index + ibuf + watch |
| `ibuf0ibuf.cc:3081` | `IBUF_OP_DELETE` 的 buffer 条件 |

**DDL / GTID / Clone / 备份**

| 位置 | 说明 |
|------|------|
| `row0mysql.cc:3771` | `row_drop_table_for_mysql` |
| `row0mysql.cc:4032` / `:4115` / `:4126` | DROP 写三类 DDL log |
| `log0ddl.cc:920` / `:1576` / `:1905` / `:1942` | insert / replay / post_ddl / recover |
| `ha_innodb.cc:14558` | `innobase_truncate::truncate` |
| `sql_truncate.cc:468` | `Sql_cmd_truncate_table::truncate_base` |
| `clone0repl.h:69-385` / `:117-135` | `Clone_persist_gtid` / `get_oldest_trx_no` |
| `clone0repl.cc:462` / `:479` | `update_gtid_trx_no` / `flush_gtids` |
| `clone0copy.cc:230` / `:862` | `for_each_file` / `add_node`（不过滤 undo） |
| `clone0copy.cc:332` / `:474` | `synchronize_binlog_gtid` / `init_redo_copy` |
| `clone0api.h:182-206` | `Clone_notify::Type` 枚举 |
| `clone0snapshot.cc:1385` | `block_state_change`（undo 特殊逻辑） |
| `trx0purge.cc:1482` / `:1594` | undo truncate 的两道 clone 拦截 |
| `trx0rseg.cc:159-213` / `:249-321` | `persist_gtid` / `physical_initialize` |
| `trx0undo.cc:670-720` | `trx_undo_gtid_read_and_persist` |
| `log0log.cc:1588` | `is_cloned_db` 判定 |
| `log0meb.cc:1687` / `:2138-2146` | MEB consumer 起点=checkpoint / UDF 注册 |
| `srv0start.cc:724-729` | undo 打开失败是 fatal |
| `srv0start.cc:1692` / `:2036` / `:2146` | 克隆启动：clone_init / 禁 ibuf / 重置 creator |
| `srv0start.cc:2720-2739` | 关机等 purge |
| `srv0srv.cc:3152-3174` | coordinator 关机收尾 |
| `srv0srv.h:925-942` | `force_recovery` 级别定义 |
| `lock0lock.cc:4722` | History list length 唯一输出点 |
| `srv0mon.cc:786-790` | `trx_rseg_history_len` 指标 |

---

## 参考

**论文**
- Mohan, Haderle, Lindsay, Pirahesh, Schwarz. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. ACM TODS 1992.（undo/redo 分离与 physiological logging 的理论基础；InnoDB 中落点为 undo 页自身的修改也走 mtr 产生 redo）
- Graefe. *Volcano—An Extensible and Parallel Query Evaluation System*. 1990.（本篇未直接使用，仅作迭代器风格参照）

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Undo Tablespaces*
- *MySQL 8.0 Reference Manual → InnoDB Multi-Versioning*
- *MySQL 8.0 Reference Manual → Purging and Change Buffer*
- WorkLog: [WL#9506](https://dev.mysql.com/worklog/task/?id=9506) InnoDB: Undo tablespace improvements（8.0 独立 undo 表空间与 RSEG_ARRAY）
- WorkLog: [WL#10322](https://dev.mysql.com/worklog/task/?id=10322) InnoDB: Atomic DDL（DDL log 与 post-DDL replay）

**相关文档**
- 上游（谁写 undo、提交时如何翻转状态）见 [`trx.md`](trx.md)
- 下游（undo 如何被消费成历史版本）见 [`mvcc.md`](mvcc.md)
- undo 页与 undo 记录本身受 redo 保护，见 [`redo_log.md`](redo_log.md)
- 行记录中 `DB_ROLL_PTR` 字段的物理格式见 [`physical/record.md`](physical/record.md)；页结构通览见 [`physical/page_structure.md`](physical/page_structure.md)
- fsp / segment / inode 的通用机制见 [`physical/tablespace.md`](physical/tablespace.md)
- purge 删除索引记录的 B-tree 操作见 [`btr.md`](btr.md)
- purge 使用 buffer pool watch 的哨兵机制见 [`buffer_pool.md`](buffer_pool.md)
- DDL 整体框架与 Online/INSTANT 见 [`ddl.md`](ddl.md)
- undo header 中 GTID 的持久化路径见 [`../server/replication/gtid.md`](../server/replication/gtid.md)
