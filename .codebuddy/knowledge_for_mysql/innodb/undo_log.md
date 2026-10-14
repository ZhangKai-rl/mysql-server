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
- [Misc](#misc)
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

★ **不存在** `undo::Tablespace::truncate()`。真正 truncate 一个 undo 表空间的是 `trx_undo_truncate_tablespace()`（`trx0undo.cc:2049`），而**调度者**是 `trx_purge_truncate_undo_spaces()`（trx0purge.cc:1703）——**由 purge 线程驱动，没有独立后台线程**。

**选择谁被 truncate**（`trx_purge_mark_undo_for_truncate`，trx0purge.cc:1253）：

```cpp
static bool trx_purge_mark_undo_for_truncate(size_t truncate_count) {
  /* We always have at least 2 undo spaces, even though one of them may be
  inactive. */
  ut_a(undo::spaces->size() >= FSP_IMPLICIT_UNDO_TABLESPACES);

  bool normal_operation = (srv_shutdown_state == SRV_SHUTDOWN_NONE);
  bool in_fast_shutdown = (!normal_operation && srv_fast_shutdown > 0);
  if (in_fast_shutdown) return (false);   // ★ 快速关机时跳过（可在启动时再 truncate）

  auto undo_trunc = &purge_sys->undo_trunc;
  if (undo_trunc->is_marked()) return (true);   // ★ 已经有标记中的

  undo::spaces->s_lock();

  /* Look for any undo space that is inactive explicitly. */
  auto undo_ts = undo::spaces->find_first_inactive_explicit(&num_active);
  if (undo_ts != nullptr) {
    undo_trunc->mark(undo_ts);       // ★ 优先级 1：用户 ALTER ... SET INACTIVE 的
    undo::spaces->s_unlock();
    return (true);
  }

  undo::spaces->s_unlock();

  /* There may be some reasons not to truncate implicitly.
  If truncate is disabled, do not truncate. */
  if (!srv_undo_log_truncate) return (false);   // ★ innodb_undo_log_truncate=OFF

  if (normal_operation) {
    if (num_active == 1) return (false);        // ★ 只剩一个 active 就不做
    if (truncate_count > 0) return (false);     // ★ 一次 purge 批次最多隐式 truncate 一个
    /* Wait at least one second between searches. */
    if (undo_trunc->check_timer() < PURGE_CHECK_UNDO_TRUNCATE_DELAY_IN_MS) {
      return (false);                           // ★ 限频
    }
    undo_trunc->reset_timer();
  }

  /* Find an undo tablespace that is too big and needs to be truncated. */
  undo::spaces->s_lock();

  /* Avoid bias selection and so start the scan immediately after the last
  space selected for truncate. */
  space_id_t space_num = undo_trunc->get_scan_space_num();
  space_id_t first_space_num_scanned = space_num;

  do {
    auto undo_space = undo::spaces->find(space_num);
    if (undo_space->needs_truncation()) {       // ★ 超过 innodb_max_undo_log_size
      undo_trunc->increment_scan();
      undo_trunc->mark(undo_space);
      break;
    }
    space_num = undo_trunc->increment_scan();   // ★ round-robin，避免总选同一个
  } while (space_num != first_space_num_scanned);

  undo::spaces->s_unlock();
  if (!undo_trunc->is_marked()) return (false);
  return (true);
}
```

```cpp
/** Select an undo tablespace to truncate, make sure it is empty of undo logs,
then finally truncate it. */
static void trx_purge_truncate_undo_spaces() {
  if (srv_is_upgrade_mode) return;      // ★ 升级模式不做

  auto &undo_trunc = purge_sys->undo_trunc;

  for (size_t i = 0; i < undo::spaces->size(); ++i) {
    if (!trx_purge_mark_undo_for_truncate(i)) break;      // ① 选一个并标记
    ut_a(undo_trunc.is_marked());
    if (!trx_purge_check_if_marked_undo_is_empty()) break; // ② 等它被 purge 清空
    ut_a(undo_trunc.is_marked_space_empty());
    if (!trx_purge_truncate_marked_undo()) break;          // ③ 真正 truncate
  }
}
```

**`ALTER ... SET INACTIVE` 的语义**（`innodb_alter_undo_tablespace_inactive`，ha_innodb.cc:16178）：

```cpp
  if (undo_space->is_active()) {
    /* There must be at least 2 active undo tablespaces besides the one we
    are trying to make inactive. ... */
    ulint other_active_spaces = 0;
    for (auto undo_ts : undo::spaces->m_spaces) {
      if (undo_ts != undo_space) {
        if (undo_ts->is_active()) {
          other_active_spaces++;
        } else if (undo_ts->is_inactive_implicit() &&
                   purge_sys->undo_trunc.get_marked_space_num() ==
                       undo_ts->num()) {
          other_active_spaces++;
        }
      }
    }
    if (other_active_spaces < 2) {
      my_printf_error(ER_DISALLOWED_OPERATION,
                      "Cannot set %s inactive since there would be"
                      " less than 2 undo tablespaces left active.", ...);
      return (HA_ERR_NOT_ALLOWED_COMMAND);
    }
  }

  /* Make sure the DD shows inactive if it is active. */
  dd_tablespace_set_state(dd_space, DD_SPACE_STATE_INACTIVE);

  /* Apply this to new transactions. */
  undo_space->set_inactive_explicit();   // ★★ 只影响【新事务】，老 undo 仍需 purge 清空

  srv_wake_purge_thread_if_not_active(); // ★★ 唤醒 purge 去清空它并 truncate
```

> ★★ **"SET INACTIVE 不会立即 truncate"**——这是最常见的误解。它只是：①改 DD 状态；②让新事务不再往里分配 rseg；③唤醒 purge。真正的 truncate 要等该表空间里的**所有老 undo 都被 purge 完**（`is_empty()`），这可能要几分钟甚至更久（长事务会拖住）。

`DROP UNDO TABLESPACE` 的三个前置条件（`innodb_drop_undo_tablespace`，ha_innodb.cc:16319）：

```cpp
  if (undo_space->num() <= FSP_IMPLICIT_UNDO_TABLESPACES ||
      !undo_space->is_empty()) {
    // ① 非内置前两个（num > 2）
    // ② 已 SET INACTIVE（is_active()==false 且非 inactive_implicit）
    // ③ 已被 purge 清空（state == empty）
    ...
    return HA_ERR_TABLESPACE_IS_NOT_EMPTY;
  }
```

**所有 undo DDL 共用一把 `undo::ddl_mutex`**（防止 DDL 的 x_lock 插队饿死 purge 的短 s_lock）：

```cpp
  /* Serialize all undo tablespace DDLs */
  mutex_enter(&undo::ddl_mutex);
  ...
  space_id_t space_id = undo::get_next_available_space_num();
  if (space_id == SPACE_UNKNOWN ||
      undo::spaces->size() == FSP_MAX_UNDO_TABLESPACES) {
    // ★ FSP_MAX_UNDO_TABLESPACES = 127（对应 roll_ptr 里那 7 bit）
    error = HA_ERR_TABLESPACE_EXISTS;
    ...
  }
```

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

### `trx_undo_t` 内存对象的生命周期（建 / 复用 / 回收）

创建（`trx0undo.cc:1463-1513`）：

```cpp
/** Creates and initializes an undo log memory object.
@param[in]   rseg     rollback segment memory object
@param[in]   id       slot index within rseg
@param[in]   type     type of the log: TRX_UNDO_INSERT or TRX_UNDO_UPDATE
...
@return own: the undo log memory object */
static trx_undo_t *trx_undo_mem_create(trx_rseg_t *rseg, ulint id, ulint type,
                                       trx_id_t trx_id, const XID *xid,
                                       page_no_t page_no, ulint offset) {
  trx_undo_t *undo;

  ut_a(id < TRX_RSEG_N_SLOTS);

  undo = static_cast<trx_undo_t *>(
      ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sizeof(*undo)));

  if (undo == nullptr) {
    return (nullptr);
  }

  undo->id = id;
  undo->type = type;
  undo->state = TRX_UNDO_ACTIVE;
  undo->del_marks = false;
  undo->trx_id = trx_id;
  undo->xid = *xid;

  undo->dict_operation = false;
  undo->flag = 0;
  undo->m_gtid_storage = trx_undo_t::Gtid_storage::NONE;

  undo->rseg = rseg;

  undo->space = rseg->space_id;
  undo->page_size.copy_from(rseg->page_size);
  undo->hdr_page_no = page_no;
  undo->hdr_offset = offset;
  undo->last_page_no = page_no;
  undo->size = 1;

  undo->empty = true;
  undo->top_page_no = page_no;
  undo->guess_block = nullptr;

  return (undo);
}
```

复用（cached undo 摘下来重置，`trx0undo.cc:1513-1536`）：

```cpp
/** Initializes a cached undo log object for new use. */
static void trx_undo_mem_init_for_reuse(
    trx_undo_t *undo, trx_id_t trx_id, const XID *xid, ulint offset)
{
  ut_ad(mutex_own(&((undo->rseg)->mutex)));

  ut_a(undo->id < TRX_RSEG_N_SLOTS);

  undo->state = TRX_UNDO_ACTIVE;
  undo->del_marks = false;
  undo->trx_id = trx_id;
  undo->xid = *xid;

  undo->dict_operation = false;
  undo->flag = 0;
  undo->m_gtid_storage = trx_undo_t::Gtid_storage::NONE;

  undo->hdr_offset = offset;
  undo->empty = true;
}
```

回收判定（`trx0undo.cc:1806-1841`）：

```cpp
/** Sets the state of the undo log segment at a transaction finish.
 @return undo log segment header page, x-latched */
page_t *trx_undo_set_state_at_finish(
    trx_undo_t *undo, mtr_t *mtr)
{
  ...
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
                             TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;

  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;
  } else {
    state = TRX_UNDO_TO_PURGE;
  }

  undo->state = state;

  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, state, MLOG_2BYTES, mtr);

  return (undo_page);
}
```

阈值（`trx0undo.h:493`）：

```cpp
#define TRX_UNDO_PAGE_REUSE_LIMIT (3 * UNIV_PAGE_SIZE / 4)
```

/* 【注】"只占 1 页且已用空间 < 3/4 页" ⇒ `TRX_UNDO_CACHED`（进 cached list 复用）；
   否则 insert → `TRX_UNDO_TO_FREE`（直接释放）、update → `TRX_UNDO_TO_PURGE`（交给 purge）。
   **绝大多数小事务走 cached**，这也是 rseg 里 `insert_undo_cached` / `update_undo_cached`
   两个 list 存在的意义。 */

两种 cleanup 的二选一（`trx0undo.cc:1923-2018`）：

```cpp
void trx_undo_update_cleanup(trx_t *trx, trx_undo_ptr_t *undo_ptr,
                             page_t *undo_page, bool update_rseg_history_len,
                             ulint n_added_logs, mtr_t *mtr) {
  ...
  trx_purge_add_update_undo_to_history(
      trx, undo_ptr, undo_page, update_rseg_history_len, n_added_logs, mtr);

  UT_LIST_REMOVE(rseg->update_undo_list, undo);

  undo_ptr->update_undo = nullptr;

  if (undo->state == TRX_UNDO_CACHED) {
    UT_LIST_ADD_FIRST(rseg->update_undo_cached, undo);

    MONITOR_INC(MONITOR_NUM_UNDO_SLOT_CACHED);
  } else {
    ut_ad(undo->state == TRX_UNDO_TO_PURGE);

    trx_undo_mem_free(undo);
  }
}
```

```cpp
/** Frees an insert undo log after a transaction commit or rollback.
Knowledge of inserts is not needed after a commit or rollback, therefore
the data can be discarded.
@param[in,out]  undo_ptr        undo log to clean up
@param[in]      noredo          whether the undo tablespace is redo logged */
void trx_undo_insert_cleanup(trx_undo_ptr_t *undo_ptr, bool noredo) {
  ...
  if (undo->state == TRX_UNDO_CACHED) {
    UT_LIST_ADD_FIRST(rseg->insert_undo_cached, undo);

    MONITOR_INC(MONITOR_NUM_UNDO_SLOT_CACHED);
  } else {
    ut_ad(undo->state == TRX_UNDO_TO_FREE);

    /* Delete first the undo log segment in the file */

    rseg->unlatch();

    DEBUG_SYNC_C("innodb_commit_wait_for_truncate");
    trx_undo_seg_free(undo, noredo);

    rseg->latch();

    trx_undo_mem_free(undo);
  }
  ...
}
```

★ **归还路径**：insert undo 提交后**通常进 cached list 复用而不是释放**——只有 `TRX_UNDO_TO_FREE` 才走 `trx_undo_seg_free()` + `trx_undo_mem_free()`。update undo 同理不是"只进 history list"：`trx_undo_update_cleanup()` 先 `add_update_undo_to_history()`，**内存对象在提交时就被释放**（cached 则回 list），留在磁盘上的 undo segment 由 purge 后续回收。另注：insert undo **从来不入 history list**；update undo 即使 `del_marks == false`（纯 update 且不改索引列）也**无条件** `add_update_undo_to_history`（`trx0undo.cc:1945`）。

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

**真实定义**（`storage/innobase/include/trx0undo.h:466`）——★ ①它们**不在** `trx0types.h`；②8.0 起是 **`constexpr uint32_t`**，不是 `#define`：

```cpp
/** The offset of the undo log page header on pages of the undo log */
constexpr uint32_t TRX_UNDO_PAGE_HDR = FSEG_PAGE_DATA;      // = FIL_PAGE_DATA = 38
/*-------------------------------------------------------------*/
/** Transaction undo log page header offsets */
/** @{ */
/** TRX_UNDO_INSERT or TRX_UNDO_UPDATE */
constexpr uint32_t TRX_UNDO_PAGE_TYPE = 0;
/** Byte offset where the undo log records for the LATEST transaction start on
 this page (remember that in an update undo log, the first page can contain
 several undo logs) */
constexpr uint32_t TRX_UNDO_PAGE_START = 2;
/** On each page of the undo log this field contains the byte offset of the
 first free byte on the page */
constexpr uint32_t TRX_UNDO_PAGE_FREE = 4;
/** The file list node in the chain of undo log pages */
constexpr uint32_t TRX_UNDO_PAGE_NODE = 6;
/*-------------------------------------------------------------*/
/** Size of the transaction undo log page header, in bytes */
constexpr uint32_t TRX_UNDO_PAGE_HDR_SIZE = 6 + FLST_NODE_SIZE;   // = 18
/** @} */
```

> ★ `TRX_UNDO_PAGE_FREE` 是整台状态机的**写游标**：`trx_undo_page_report_insert()` 从它取写入起点，`trx_undo_page_set_next_prev_and_add()` 推进它。所谓"回退 2 字节"就是减它。

页初始化（`trx0undo.cc:368`）：

```cpp
/** Initializes the fields in an undo log segment page. */
static void trx_undo_page_init(page_t *undo_page, ulint type, mtr_t *mtr) {
  trx_upagef_t *page_hdr;

  page_hdr = undo_page + TRX_UNDO_PAGE_HDR;

  mach_write_to_2(page_hdr + TRX_UNDO_PAGE_TYPE, type);
  // ★ TRX_UNDO_INSERT(1) 或 TRX_UNDO_UPDATE(2)

  mach_write_to_2(page_hdr + TRX_UNDO_PAGE_START,
                  TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_HDR_SIZE);
  mach_write_to_2(page_hdr + TRX_UNDO_PAGE_FREE,
                  TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_HDR_SIZE);
  // ★ 非首页从这里开始写 rec（首页要越过段头，见 trx_undo_seg_create）

  fil_page_set_type(undo_page, FIL_PAGE_UNDO_LOG);   // ★ FIL 页类型
  trx_undo_page_init_log(undo_page, type, mtr);      // MLOG_UNDO_INIT
}
```

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

★ **不存在** `TRX_UNDO_LOG_HDR` / `TRX_UNDO_LOG_HDR_SIZE` 这两个符号（全仓仅 `trx0undo.cc:503` 一处**过期注释**里出现过这个名字）。真实的是四个"尺寸档位"——因为 header 尾部会按需追加 XID / GTID：

```cpp
/** The undo log header. There can be several undo log headers on the first
page of an update undo log segment. */
/** @{ */
/** Transaction id */
constexpr uint32_t TRX_UNDO_TRX_ID = 0;
/** Transaction number of the transaction; defined only if the log is in a
 history list */
constexpr uint32_t TRX_UNDO_TRX_NO = 8;
/** Defined only in an update undo log: true if the transaction may have done
 delete markings of records, and thus purge is necessary */
constexpr uint32_t TRX_UNDO_DEL_MARKS = 16;
/** Offset of the first undo log record of this log on the header page; purge
 may remove undo log record from the log start, and therefore this is not
 necessarily the same as this log header end offset */
constexpr uint32_t TRX_UNDO_LOG_START = 18;
/** Transaction UNDO flags in one byte. This is backward compatible as earlier
 we were storing either 1 or 0 for TRX_UNDO_XID_EXISTS. */
constexpr uint32_t TRX_UNDO_FLAGS = 20;
/** true if undo log header includes X/Open XA transaction identification XID */
constexpr uint32_t TRX_UNDO_FLAG_XID = 0x01;
/** true if undo log header includes GTID information from replication */
constexpr uint32_t TRX_UNDO_FLAG_GTID = 0x02;
/** true if undo log header includes GTID information for XA PREPARE */
constexpr uint32_t TRX_UNDO_FLAG_XA_PREPARE_GTID = 0x04;
/** true if the transaction is a table create, index create, or drop
 transaction: in recovery the transaction cannot be rolled back in the usual
 way ... */
constexpr uint32_t TRX_UNDO_DICT_TRANS = 21;
/** Id of the table if the preceding field is true. Note: deprecated */
constexpr uint32_t TRX_UNDO_TABLE_ID = 22;
/** Offset of the next undo log header on this page, 0 if none */
constexpr uint32_t TRX_UNDO_NEXT_LOG = 30;
/** Offset of the previous undo log header on this page, 0 if none */
constexpr uint32_t TRX_UNDO_PREV_LOG = 32;
/** If the log is put to the history list, the file list node is here */
constexpr uint32_t TRX_UNDO_HISTORY_NODE = 34;
/*-------------------------------------------------------------*/
/** Size of the undo log header without XID information */
constexpr uint32_t TRX_UNDO_LOG_OLD_HDR_SIZE = 34 + FLST_NODE_SIZE;   // = 46

/** X/Open XA Transaction Identification (XID) */
constexpr uint32_t TRX_UNDO_XA_FORMAT = TRX_UNDO_LOG_OLD_HDR_SIZE;
constexpr uint32_t TRX_UNDO_XA_TRID_LEN = TRX_UNDO_XA_FORMAT + 4;
constexpr uint32_t TRX_UNDO_XA_BQUAL_LEN = TRX_UNDO_XA_TRID_LEN + 4;
constexpr uint32_t TRX_UNDO_XA_XID = TRX_UNDO_XA_BQUAL_LEN + 4;
constexpr uint32_t TRX_UNDO_LOG_XA_HDR_SIZE = TRX_UNDO_XA_XID + XIDDATASIZE;  // = 186

/* GTID is generated by replication when binlog and GTID mode is on. ... */
constexpr uint32_t TRX_UNDO_LOG_GTID_VERSION = TRX_UNDO_LOG_XA_HDR_SIZE;
constexpr uint32_t TRX_UNDO_LOG_GTID = TRX_UNDO_LOG_XA_HDR_SIZE + 1;
constexpr uint32_t TRX_UNDO_LOG_GTID_LEN = 64;
constexpr uint32_t TRX_UNDO_LOG_GTID_HDR_SIZE =
    TRX_UNDO_LOG_GTID + TRX_UNDO_LOG_GTID_LEN;                        // = 251
constexpr uint32_t TRX_UNDO_LOG_GTID_XA = TRX_UNDO_LOG_GTID_HDR_SIZE;
/** Total size with XA GTID information. For external XA transaction we need
to store both prepare and commit GTID. */
constexpr uint32_t TRX_UNDO_LOG_GTID_XA_HDR_SIZE =
    TRX_UNDO_LOG_GTID_HDR_SIZE + TRX_UNDO_LOG_GTID_LEN;               // = 315
```

> ★ **`TRX_UNDO_XID_EXISTS` 已删除**——8.0 改成 1 字节 `TRX_UNDO_FLAGS` + 三个位标志。源码注释明说：*"This is backward compatible as earlier we were storing either 1 or 0 for TRX_UNDO_XID_EXISTS"*。
>
> ★ header 是**先建 46 字节的 old header，再按需"撑开"**（`trx_undo_header_add_space_for_xid()`，`trx0undo.cc:777`）：
> ```cpp
>   ulint new_limit = TRX_UNDO_LOG_XA_HDR_SIZE;
>   switch (gtid_storage) {
>     case trx_undo_t::Gtid_storage::COMMIT:
>       new_limit = TRX_UNDO_LOG_GTID_HDR_SIZE; break;
>     case trx_undo_t::Gtid_storage::PREPARE_AND_COMMIT:
>       new_limit = TRX_UNDO_LOG_GTID_XA_HDR_SIZE; break;
>     case trx_undo_t::Gtid_storage::NONE:
>       new_limit = TRX_UNDO_LOG_XA_HDR_SIZE; break;
>   }
>   ulint new_free = free + (new_limit - TRX_UNDO_LOG_OLD_HDR_SIZE);
>   mlog_write_ulint(page_hdr + TRX_UNDO_PAGE_START, new_free, MLOG_2BYTES, mtr);
>   mlog_write_ulint(page_hdr + TRX_UNDO_PAGE_FREE,  new_free, MLOG_2BYTES, mtr);
>   mlog_write_ulint(log_hdr + TRX_UNDO_LOG_START,   new_free, MLOG_2BYTES, mtr);
> ```
> 源码还有一段吐槽：*"TODO: do not append the XID to the log header if XA is not needed by the user. The XID wastes about 150 bytes of space in every undo log."*

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

★ 类型值不是 1/2/3/4，而是 **11/12/13/14**（低 4 位；旧版 1..4 已废弃）。完整定义（`storage/innobase/include/trx0rec.h:295`）：

```cpp
/* Types of an undo log record: these have to be smaller than 16, as the
compilation info multiplied by 16 is ORed to this value in an undo log record */

/** fresh insert into clustered index */
constexpr uint32_t TRX_UNDO_INSERT_REC = 11;
/** update of a non-delete-marked  record */
constexpr uint32_t TRX_UNDO_UPD_EXIST_REC = 12;
/** update of a delete marked record to a not delete marked record; also the
fields of the record can change */
constexpr uint32_t TRX_UNDO_UPD_DEL_REC = 13;
/* delete marking of a record; fields do not change */
constexpr uint32_t TRX_UNDO_DEL_MARK_REC = 14;
/** compilation info is multiplied by this and ORed to the type above */
constexpr uint32_t TRX_UNDO_CMPL_INFO_MULT = 16;
/** If this bit is set in type_cmpl, then the undo log record has support for
 partial update of BLOBs. Also to make the undo log format extensible,
 introducing a new flag next to the type_cmpl flag. */
constexpr uint32_t TRX_UNDO_MODIFY_BLOB = 64;
/* This bit can be ORed to type_cmpl to denote that we updated external storage
 fields: used by purge to free the external storage */
constexpr uint32_t TRX_UNDO_UPD_EXTERN = 128;

/** Operation type flags used in trx_undo_report_row_operation */
constexpr uint32_t TRX_UNDO_INSERT_OP = 1;
constexpr uint32_t TRX_UNDO_MODIFY_OP = 2;

/** The type and compilation info flag in the undo record for update.
For easier understanding let the 8 bits be numbered as 7, 6, 5, 4, 3, 2, 1, 0. */
struct type_cmpl_t {
  type_cmpl_t() : m_flag(0) {}

  const byte *read(const byte *ptr) {
    m_flag = mach_read_from_1(ptr);
    return (ptr + 1);
  }
  ulint type_info() { return (m_flag & 0x0F); }            // ★ bit 0-3
  ulint cmpl_info() { return ((m_flag >> 4) & 0x03); }     // ★ bit 4-5
  /** Is an LOB updated by this update operation. */
  bool is_lob_updated() { return (m_flag & TRX_UNDO_UPD_EXTERN); }  // bit 7
  /** Does the undo log record contains information about LOB partial
  update vector. */
  bool is_lob_undo() const { return (m_flag & TRX_UNDO_MODIFY_BLOB); } // bit 6

 private:
  uint8_t m_flag;
};
```

> ★ `type_cmpl_t` 是 8.0 新增的封装，旧版直接做位运算。位布局：**bit0–3 = type**、**bit4–5 = cmpl_info**、**bit6 = MODIFY_BLOB**（8.0 **无条件置位**，标志"支持 LOB 部分更新"的新格式）、**bit7 = UPD_EXTERN**（有外存列，purge 需释放）。

配套的读函数（`trx0undo.ic` / `trx0rec.ic`）：

```cpp
// storage/innobase/include/trx0rec.ic:37
static inline ulint trx_undo_rec_get_type(const trx_undo_rec_t *undo_rec) {
  return (mach_read_from_1(undo_rec + 2) & (TRX_UNDO_CMPL_INFO_MULT - 1));
}
static inline ulint trx_undo_rec_get_cmpl_info(const trx_undo_rec_t *undo_rec) {
  return (mach_read_from_1(undo_rec + 2) / TRX_UNDO_CMPL_INFO_MULT);
}
static inline bool trx_undo_rec_get_extern_storage(const trx_undo_rec_t *undo_rec) {
  if (mach_read_from_1(undo_rec + 2) & TRX_UNDO_UPD_EXTERN) return true;
  return false;
}
/** Reads the undo log record number. */
static inline undo_no_t trx_undo_rec_get_undo_no(const trx_undo_rec_t *undo_rec) {
  const byte *ptr = undo_rec + 2;
  uint8_t type_cmpl = mach_read_from_1(ptr);
  const bool blob_undo = type_cmpl & TRX_UNDO_MODIFY_BLOB;
  if (blob_undo) {
    /* The next record offset takes 2 bytes + 1 byte for type_cmpl flag
       + 1 byte for the new flag.  Total 4 bytes. */
    ptr = undo_rec + 4;
  } else {
    ptr = undo_rec + 3;
  }
  return (mach_u64_read_much_compressed(ptr));
}
```

> ★ 注意 `undo_no` 有两种偏移（+3 或 +4），取决于 **bit6 有没有置位**——这正是 8.0 加那 1 字节保留 flag 带来的兼容性代价。

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

### 多值索引（multi-value）的 undo 表示

多值列（JSON 数组上的 `CAST(... AS ... ARRAY)` 索引）的 undo 由 `Multi_value_logger` 负责序列化——它在 **`data0data.h/.cc`**，不在 `trx0rec.cc`（按文件名找容易漏）。

`storage/innobase/include/data0data.h:523-602`：

```cpp
/** Class to log the multi-value data and read it from the log */
class Multi_value_logger {
 public:
  /** Constructor
  @param[in]    mv_data         multi-value data structure to log
  @param[in]    field_len       multi-value data field length */
  Multi_value_logger(const multi_value_data *mv_data, uint32_t field_len)
      : m_mv_data(mv_data), m_field_len(field_len) {}

  /** Get the log length for the multi-value data
  @param[in]    precise true if precise length is needed, false if rough
                estimation is OK
  @return the total log length for the multi-value data */
  uint32_t get_log_len(bool precise) const;

  /** Log the multi-value data to specified memory
  @param[in,out]        ptr     the memory to write
  @return next to the end of the multi-value data log */
  byte *log(byte **ptr);

  /** Read the log length for the multi-value data log starting from ptr
  @param[in]    ptr     log starting from here
  @return the length of this log */
  static uint32_t read_log_len(const byte *ptr);

  /** Read the multi-value data from the ptr
  @param[in]            ptr     log starting from here
  @param[in,out]        field   multi-value data field to store the array
  @param[in,out]        heap    memory heap
  @return next to the end of the multi-value data log */
  static const byte *read(const byte *ptr, dfield_t *field, mem_heap_t *heap);
  ...
  /** Determine if the log starting from ptr is for multi-value data
  @return true if it is for multi-value data, otherwise false */
  static bool is_multi_value_log(const byte *ptr) {
    return (*ptr == s_multi_value_virtual_col_length_marker);
  }

 private:
  ...
  /** Multi-value virtual column length marker. With this length marker,
  a multi-value virtual column undo log can be identified. Meanwhile, this
  marker should/will not conflict with any normal compressed written length
  leading byte */
  static constexpr uint8_t s_multi_value_virtual_col_length_marker = 0xFF;

  /** Multi-value virtual column length, which indicates that there is
  no value on the multi-value index. It's mapped to UNIV_NO_INDEX_VALUE */
  static constexpr uint16_t s_multi_value_no_index_value = 0x0;

  /** Multi-value virtual column length, which indicates that the field
  is NULL. It's mapped to UNIV_SQL_NULL. Since any not NULL and not no value
  multi-value data must be longer than 1 byte, so this is safe for this
  special meaning */
  static constexpr uint16_t s_multi_value_null = 0x1;
```

/* 【注】`0xFF` 这个 marker 的意义：**多值日志的首字节不会与"正常的压缩长度首字节"冲突**，
   所以 `is_multi_value_log()` 只看 1 个字节就能分派。 */

写（`data0data.cc:894-968`，`get_log_len` + `log`）：

```cpp
uint32_t Multi_value_logger::get_log_len(bool precise) const {
  /* Alwayas a multi-value data marker at the beginning */
  uint32_t total_len = 1;

  if (m_field_len == UNIV_SQL_NULL || m_field_len == UNIV_NO_INDEX_VALUE) {
    total_len += s_log_length_for_null_or_empty;
    return (total_len);
  }

  ut_ad(m_field_len == UNIV_MULTI_VALUE_ARRAY_MARKER);

  /* Keep two bytes for the total length of the log */
  total_len += 2;

  /* Remember the length of the multi-value array.
  Will write the data_len[i] in compressed format which at most
  costs 5 bytes */
  total_len += precise ? mach_get_compressed_size(m_mv_data->num_v)
                       : s_max_compressed_mv_key_length_size;

  /* Remember each data length and value */
  for (uint32_t i = 0; i < m_mv_data->num_v; ++i) {
    ut_ad(m_mv_data->data_len[i] != UNIV_SQL_NULL);
    total_len += m_mv_data->data_len[i];
    total_len += precise ? mach_get_compressed_size(m_mv_data->data_len[i])
                         : s_max_compressed_mv_key_length_size;
  }

  /* Remember the bitset of the multi-value data if exists*/
  if (m_mv_data->bitset != nullptr) {
    total_len += UT_BITS_IN_BYTES(m_mv_data->num_v);
  }

  return (total_len);
}

byte *Multi_value_logger::log(byte **ptr) {
  mach_write_to_1(*ptr, s_multi_value_virtual_col_length_marker);
  *ptr += 1;
  ...
  byte *old_ptr = *ptr;

  /* Store data in the same sequence as described in get_log_len() */
  *ptr += 2;
  *ptr += mach_write_compressed(*ptr, m_mv_data->num_v);
  for (uint32_t i = 0; i < m_mv_data->num_v; ++i) {
    ut_ad(m_mv_data->data_len[i] != UNIV_SQL_NULL);
    *ptr += mach_write_compressed(*ptr, m_mv_data->data_len[i]);
    ut_memcpy(*ptr, m_mv_data->datap[i], m_mv_data->data_len[i]);
    *ptr += m_mv_data->data_len[i];
  }

  if (m_mv_data->bitset != nullptr) {
    /* Always just write out the bitset of enough size for all data,
    rather than the size of bitset. */
    uint32_t bitset_len = UT_BITS_IN_BYTES(m_mv_data->num_v);
    ut_memcpy(*ptr, m_mv_data->bitset->bitset(), bitset_len);
    *ptr += bitset_len;
  }

  mach_write_to_2(old_ptr, *ptr - old_ptr);

  return (*ptr);
}
```

/* 【注】★ 布局是 `0xFF` marker → 2B 总长 → 压缩的 num_v → 每个元素（压缩长度 + 值）→ 可选 bitset。
   注意 `log()` 一开始跳过 2 字节、最后才回填总长——**长度前缀是事后回填的**。 */

读（`data0data.cc:970-1027`）：

```cpp
uint32_t Multi_value_logger::read_log_len(const byte *ptr) {
  ut_ad(is_multi_value_log(ptr));

  uint32_t total_len = mach_read_from_2(ptr + 1);

  if (total_len == s_multi_value_null ||
      total_len == s_multi_value_no_index_value) {
    return (1 + s_log_length_for_null_or_empty);
  }

  return (1 + total_len);
}

const byte *Multi_value_logger::read(const byte *ptr, dfield_t *field,
                                     mem_heap_t *heap) {
  ut_ad(is_multi_value_log(ptr));

  ++ptr;

  uint32_t total_len = mach_read_from_2(ptr);
  const byte *old_ptr = ptr;

  ptr += s_log_length_for_null_or_empty;
  ...
  uint32_t num = mach_read_next_compressed(&ptr);
  field->data = mem_heap_alloc(heap, sizeof(multi_value_data));
  field->len = UNIV_MULTI_VALUE_ARRAY_MARKER;

  multi_value_data *multi_val = static_cast<multi_value_data *>(field->data);

  multi_val->num_v = num;
  multi_val->alloc(num, false, heap);

  for (uint32_t i = 0; i < num; ++i) {
    uint32_t len = mach_read_next_compressed(&ptr);
    multi_val->datap[i] = ptr;
    multi_val->data_len[i] = len;

    ptr += len;
  }

  if (ptr < old_ptr + total_len) {
    multi_val->alloc_bitset(heap);
    multi_val->bitset->copy(ptr, UT_BITS_IN_BYTES(num));
    ptr += UT_BITS_IN_BYTES(num);
  }

  ut_ad(ptr == old_ptr + total_len);
  return (ptr);
}
```

/* 【注】★ bitset **没有独立的存在标志位**——读回时靠"读完所有 value 后 `ptr` 是否还没到
   `old_ptr + total_len`"来推断有没有 bitset。这是"用位置推断存在性"的紧凑编码。 */

与 undo 记录的接口（`trx0rec.cc`）：

```cpp
/** Decide if the following undo log record is a multi-value virtual column
@param[in]      undo_rec        undo log record
@return true if this is a multi-value virtual column log, otherwise false */
bool trx_undo_rec_is_multi_value(const byte *undo_rec) {
  return (Multi_value_logger::is_multi_value_log(undo_rec));
}
```

```cpp
/** Store the multi-value column information for undo log
@param[in,out]  undo_page       undo page to store the information
@param[in]      vfield          multi-value field information
@param[in,out]  ptr             pointer where to store the information
@return true if stored successfully, false if space is not enough */
static bool trx_undo_store_multi_value(page_t *undo_page,
                                       const dfield_t *vfield, byte **ptr) {
  ut_ad(vfield != nullptr);

  Multi_value_logger mv_logger(
      static_cast<multi_value_data *>(dfield_get_data(vfield)),
      dfield_get_len(vfield));
  uint32_t log_len = mv_logger.get_log_len(false);

  if (trx_undo_left(undo_page, *ptr) < log_len) {
    return (false);
  }

  mv_logger.log(ptr);

  return (true);
}
```

```cpp
/** Read from an undo log record of a multi-value virtual column.
@param[in]      ptr     pointer to remaining part of the undo record
@param[in,out]  field   stored field, nullptr if the col is no longer
                        indexed or existing, in the latter case,
                        this function will only skip the log
@param[in,out]  heap    memory heap
@return remaining part of undo log record after reading these values */
const byte *trx_undo_rec_get_multi_value(const byte *ptr, dfield_t *field,
                                         mem_heap_t *heap) {
  if (field == nullptr) {
    return (ptr + Multi_value_logger::read_log_len(ptr));
  }

  return (Multi_value_logger::read(ptr, field, heap));
}
```

/* 【注】`field == nullptr` 的分支是**加索引/删索引后的历史兼容**：列已不再被索引时，
   读 undo 时只"跳过"这段日志（`read_log_len` 算长度），不解析内容。 */

bitset 的语义（谁是"未变化"）——`ha_innodb.cc:9232-9304`：

```cpp
static void innobase_get_multi_value_and_diff(
    const TABLE *mysql_table, ulint field_idx, dfield_t *old_field,
    dfield_t *new_field, uint old_value, ulint comp, mem_heap_t *heap) {
  ...
  uint old_counter = 0;
  uint new_counter = 0;
  ...
  old_mv->alloc_bitset(heap);
  new_mv->alloc_bitset(heap);

  while (old_counter < old_v.element_count() &&
         new_counter < new_v.element_count()) {
    ...
    int res = old_v.element(old_counter).eq(new_v.element(new_counter));
    if (res == 0) {
      old_mv->bitset->set(old_counter++, false);
      new_mv->bitset->set(new_counter++, false);
    } else if (res < 0) {
      ++old_counter;
    } else {
      ++new_counter;
    }
  }
}
```

/* 【注】两个数组都按升序归并，**相等的元素被置 0**（未选中）。消费端
   `Multi_value_entry_builder_normal::skip()` 跳过 `!bitset->test(pos)` 的元素。 */

★ **多值 undo 记的是完整数组**：`log()` 对 `i = 0..num_v-1` **逐个**写长度 + 值，UPDATE 时新旧两个完整数组都写（`trx0rec.cc:1437` 写 `old_v_val`、`:1472` 写 `new_val`）。bitset 是**追加在完整数组之后的掩码**，语义是"**跳过未变化元素**"（相等置 0），不是"变化元素清单"；且它是可选的——值为 NULL / `UNIV_NO_INDEX_VALUE` 时根本不写。

### roll pointer（DB_ROLL_PTR，7 字节）

**构造**（`storage/innobase/include/trx0undo.ic:39`）：

```cpp
/** Build a roll pointer.
@param[in]      is_insert       true if insert undo log
@param[in]      space_id        space ID of the rollback segment
@param[in]      page_no         page number
@param[in]      offset          offset of the undo entry within page
@return roll pointer */
inline roll_ptr_t trx_undo_build_roll_ptr(bool is_insert, space_id_t space_id,
                                          page_no_t page_no, ulint offset) {
  roll_ptr_t roll_ptr;
  ulint id;

  ut_ad(offset < 65536);

  id = (fsp_is_undo_tablespace(space_id) ? undo::id2num(space_id) : 0);

  roll_ptr = (roll_ptr_t)is_insert << 55 | (roll_ptr_t)id << 48 |
             (roll_ptr_t)page_no << 16 | offset;
  return (roll_ptr);
}
```

```
 bit 55      : 1 = insert undo, 0 = update undo
 bit 48..54  : 7 bit —— 名义上是 rseg id，实际写的是 undo 表空间编号
               undo::id2num(space_id)；非独立 undo 表空间（system/temp）恒为 0
 bit 16..47  : 32 bit page_no
 bit 0..15   : 16 bit offset（故 ut_ad(offset < 65536)）
```

> ★ 很多资料说"roll_ptr 里是 rseg id"。8.0 写进去的其实是 **undo 表空间编号**，且只占 **7 bit**（`0x7F`）。这也解释了一个运维现象：独立 undo 表空间**最多 128 个**（`FSP_MAX_UNDO_TABLESPACES` = 127）。

**解构**（`trx0undo.ic:59`）：

```cpp
inline void trx_undo_decode_roll_ptr(roll_ptr_t roll_ptr, bool *is_insert,
                                     ulint *rseg_id, page_no_t *page_no,
                                     ulint *offset) {
  ut_ad(roll_ptr < (1ULL << 56));
  *offset = (ulint)roll_ptr & 0xFFFF;
  roll_ptr >>= 16;
  *page_no = (ulint)roll_ptr & 0xFFFFFFFF;
  roll_ptr >>= 32;
  *rseg_id = (ulint)roll_ptr & 0x7F;
  roll_ptr >>= 7;
  *is_insert = roll_ptr; /* true==1 */
}

static inline bool trx_undo_roll_ptr_is_insert(roll_ptr_t roll_ptr) {
  ut_ad(roll_ptr < (1ULL << 56));
  return roll_ptr >> 55;
}

/** Returns true if the record is of the insert type. */
static inline bool trx_undo_trx_id_is_insert(const byte *trx_id) {
  return (static_cast<bool>(trx_id[DATA_TRX_ID_LEN] >> 7));
}

static inline void trx_write_roll_ptr(byte *ptr, roll_ptr_t roll_ptr) {
  mach_write_to_7(ptr, roll_ptr);
}
static inline roll_ptr_t trx_read_roll_ptr(const byte *ptr) {
  return (mach_read_from_7(ptr));
}
```

> ★ `trx_undo_trx_id_is_insert()` 不解 roll_ptr，而是直接看 `DB_TRX_ID` 后面**第一个字节的 bit7**——因为 roll_ptr 是大端 7 字节存放，bit55 正落在那一字节的最高位。这个"偷看一个字节"的写法在回滚路径上被大量使用。
>
> ⚠️ **不存在** `trx_undo_roll_ptr_get_insert()` 这个符号（8.0 用 `trx_undo_roll_ptr_is_insert()`）。

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

### 页内遍历原语：next / prev 与跨页

四个页内原语（`storage/innobase/include/trx0undo.ic:194-284`）：

```cpp
/** Returns the previous undo record on the page in the specified log, or
 NULL if none exists.
 @return pointer to record, NULL if none */
static inline trx_undo_rec_t *trx_undo_page_get_prev_rec(
    trx_undo_rec_t *rec, /*!< in: undo log record */
    page_no_t page_no,   /*!< in: undo log header page number */
    ulint offset)        /*!< in: undo log header offset on page */
{
  page_t *undo_page;
  ulint start;

  undo_page = (page_t *)ut_align_down(rec, UNIV_PAGE_SIZE);

  start = trx_undo_page_get_start(undo_page, page_no, offset);

  if (start + undo_page == rec) {
    return (nullptr);
  }

  return (undo_page + mach_read_from_2(rec - 2));
}

/** Returns the next undo log record on the page in the specified log, or
 NULL if none exists.
 @return pointer to record, NULL if none */
static inline trx_undo_rec_t *trx_undo_page_get_next_rec(
    trx_undo_rec_t *rec, page_no_t page_no, ulint offset)
{
  page_t *undo_page;
  ulint end;
  ulint next;

  undo_page = (page_t *)ut_align_down(rec, UNIV_PAGE_SIZE);

  end = trx_undo_page_get_end(undo_page, page_no, offset);

  next = mach_read_from_2(rec);

  if (next == end) {
    return (nullptr);
  }

  return (undo_page + next);
}

/** Returns the last undo record on the page in the specified undo log, or
 NULL if none exists.
 @return pointer to record, NULL if none */
static inline trx_undo_rec_t *trx_undo_page_get_last_rec(
    page_t *undo_page, page_no_t page_no, ulint offset)
{
  ulint start;
  ulint end;

  start = trx_undo_page_get_start(undo_page, page_no, offset);
  end = trx_undo_page_get_end(undo_page, page_no, offset);

  if (start == end) {
    return (nullptr);
  }

  return (undo_page + mach_read_from_2(undo_page + end - 2));
}

/** Returns the first undo record on the page in the specified undo log, or
 NULL if none exists.
 @return pointer to record, NULL if none */
static inline trx_undo_rec_t *trx_undo_page_get_first_rec(
    page_t *undo_page, page_no_t page_no, ulint offset)
{
  ulint start;
  ulint end;

  start = trx_undo_page_get_start(undo_page, page_no, offset);
  end = trx_undo_page_get_end(undo_page, page_no, offset);

  if (start == end) {
    return (nullptr);
  }

  return (undo_page + start);
}
```

链是怎么建起来的（`trx0rec.cc:178-222`）：

```cpp
/** Set the next and previous pointers in the undo page for the undo record
 that was written to ptr. Update the first free value by the number of bytes
 written for this undo record.
 @return offset of the inserted entry on the page if succeeded, 0 if fail */
static ulint trx_undo_page_set_next_prev_and_add(
    page_t *undo_page, byte *ptr, mtr_t *mtr)
{
  ...
  ptr_to_first_free = undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE;

  first_free = mach_read_from_2(ptr_to_first_free);

  /* Write offset of the previous undo log record */
  mach_write_to_2(ptr, first_free);
  ptr += 2;

  end_of_rec = ptr - undo_page;

  /* Write offset of the next undo log record */
  mach_write_to_2(undo_page + first_free, end_of_rec);

  /* Update the offset to first free undo record */
  mach_write_to_2(ptr_to_first_free, end_of_rec);

  /* Write this log entry to the UNDO log */
  trx_undof_page_add_undo_rec_log(undo_page, first_free, end_of_rec, mtr);

  return (first_free);
}
```

/* 【注】★ 不是"每条记录自带 next/prev 两个字段"的经典双向链表，而是**首尾各 2 字节的偏移对**：
   - 记录**头 2 字节** = 自己的结束偏移（＝下一条的起始偏移）→ 被 `page_get_next_rec` 读成 next；
   - 记录**尾 2 字节** = 自己的**起始**偏移 → 被**下一条**以 `rec - 2` 读成 prev。
   所以 prev 指针物理上落在**前一条记录的尾部**。 */

跨页（走 undo page 的文件链表 `TRX_UNDO_PAGE_NODE`，不是记录链）：

```cpp
trx_undo_rec_t *trx_undo_get_prev_rec(
    trx_undo_rec_t *rec, page_no_t page_no, ulint offset,
    bool shared, mtr_t *mtr)
{
  trx_undo_rec_t *prev_rec;

  prev_rec = trx_undo_page_get_prev_rec(rec, page_no, offset);

  if (prev_rec) {
    return (prev_rec);
  }

  /* We have to go to the previous undo log page to look for the
  previous record */

  return (
      trx_undo_get_prev_rec_from_prev_page(rec, page_no, offset, shared, mtr));
}
```

/* 【注】`trx_undo_get_prev_rec_from_prev_page()` 内部走
   `flst_get_prev_addr(undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_NODE, mtr)`，
   落到上一页后取 `trx_undo_page_get_last_rec()`——**页间靠 FLST 链表，页内靠偏移对**。 */

谁往哪个方向走：

| 用途 | 入口 | 方向 | 证据 |
|---|---|---|---|
| **purge** | `trx_undo_get_first_rec()` 起步 + `trx_undo_page_get_next_rec()` / `trx_undo_get_next_rec()` | **最老 → 最新** | `trx0purge.cc:1875`、`:1985`、`:1990` |
| **回滚** | `trx_undo_get_prev_rec()` | **最新 → 最老** | `trx0roll.cc:895-897`、`trx0trx.cc:787` |
| truncate_end（回滚释放空间） | `page_get_last_rec()` 起步 + `page_get_prev_rec()` 回扫 | 最新 → 最老 | `trx0undo.cc:1087`、`:1105` |

purge 侧的"跳过式"前进（`trx0purge.cc:1982-2004`）：

```cpp
    /* Try first to find the next record which requires a purge operation from
    the same page of the same undo log */

    next_rec = trx_undo_page_get_next_rec(rec2, purge_sys->hdr_page_no,
                                          purge_sys->hdr_offset);

    if (next_rec == nullptr) {
      rec2 = trx_undo_get_next_rec(rec2, purge_sys->hdr_page_no,
                                   purge_sys->hdr_offset, &mtr);
      break;
    }
```

★ **三个边界细节**：
1. **没有 NULL/0 哨兵**：prev 边界是 `start + undo_page == rec`，next 边界是 `next == end`（`end` 来自 `TRX_UNDO_PAGE_FREE`）。
2. **next 方向有额外中断条件**：若当前页就是 undo log header 页且该 log header 的 `TRX_UNDO_NEXT_LOG != 0`，直接 `return nullptr`，**不允许跨到下一个 log**（`trx0undo.cc:231-238`）；prev 方向没有这个检查。
3. **purge 不是逐条处理**：循环只在遇到 `TRX_UNDO_DEL_MARK_REC` 或满足 `cmpl_info` 条件时才 break，其余直接跳过。

### Undo 生成路径

### 总入口 `trx_undo_report_row_operation`

★ 它**不在** `trx0undo.cc`，而在 **`storage/innobase/trx/trx0rec.cc:2135`**（声明在 `include/trx0rec.h:159`）。完整源码：

```cpp
/** Writes information to an undo log about an insert, update, or a delete
 marking of a clustered index record. This information is used in a rollback of
 the transaction and in consistent reads that must look to the history of this
 transaction.
 @return DB_SUCCESS or error code */
dberr_t trx_undo_report_row_operation(
    ulint flags, ulint op_type, que_thr_t *thr, dict_index_t *index,
    const dtuple_t *clust_entry, const upd_t *update, ulint cmpl_info,
    const rec_t *rec, const ulint *offsets, roll_ptr_t *roll_ptr) {
  trx_t *trx;
  trx_undo_t *undo;
  page_no_t page_no;
  buf_block_t *undo_block;
  trx_undo_ptr_t *undo_ptr;
  mtr_t mtr;
  dberr_t err = DB_SUCCESS;

  ut_a(index->is_clustered());
  ut_ad(!rec || rec_offs_validate(rec, index, offsets));

  if (flags & BTR_NO_UNDO_LOG_FLAG) {
    *roll_ptr = 0;
    return (DB_SUCCESS);
  }

  trx = thr_get_trx(thr);

  bool is_temp_table = index->table->is_temporary();

  if (!is_temp_table) {
    trx->mod_tables.insert(index->table);
  }

  /* If this is a temp-table then we assign temporary rseg. */
  if (is_temp_table && trx->rsegs.m_noredo.rseg == nullptr) {
    trx_assign_rseg_temp(trx);
  }

  mtr_start(&mtr);

  if (is_temp_table) {
    /* If object is temporary, disable REDO logging that
    is done to track changes done to UNDO logs. ... */
    undo_ptr = &trx->rsegs.m_noredo;
    mtr.set_log_mode(MTR_LOG_NO_REDO);      // ★ 临时表的 undo 不写 redo
  } else {
    undo_ptr = &trx->rsegs.m_redo;
  }

  mutex_enter(&trx->undo_mutex);            // ★ 整条 undo 写入受 trx->undo_mutex 保护

  switch (op_type) {
    case TRX_UNDO_INSERT_OP:
      undo = undo_ptr->insert_undo;
      if (undo == nullptr) {
        err = trx_undo_assign_undo(trx, undo_ptr, TRX_UNDO_INSERT);
        undo = undo_ptr->insert_undo;
        if (undo == nullptr) goto err_exit;
      }
      break;
    default:
      ut_ad(op_type == TRX_UNDO_MODIFY_OP);
      undo = undo_ptr->update_undo;
      if (undo == nullptr) {
        err = trx_undo_assign_undo(trx, undo_ptr, TRX_UNDO_UPDATE);
        undo = undo_ptr->update_undo;
        if (undo == nullptr) goto err_exit;
      }
      break;
  }

  page_no = undo->last_page_no;
  undo_block = buf_page_get_gen(page_id_t(undo->space, page_no),
                                undo->page_size, RW_X_LATCH, undo->guess_block,
                                Page_fetch::NORMAL, UT_LOCATION_HERE, &mtr);
  buf_block_dbg_add_level(undo_block, SYNC_TRX_UNDO_PAGE);

  do {
    page_t *undo_page;
    ulint offset;

    undo_page = buf_block_get_frame(undo_block);

    switch (op_type) {
      case TRX_UNDO_INSERT_OP:
        offset = trx_undo_page_report_insert(undo_page, trx, index, clust_entry,
                                             &mtr);
        break;
      default:
        offset =
            trx_undo_page_report_modify(undo_page, trx, index, rec, offsets,
                                        update, cmpl_info, clust_entry, &mtr);
    }

    if (UNIV_UNLIKELY(offset == 0)) {
      /* The record did not fit on the page. ... */
      if (!trx_undo_erase_page_end(undo_page, &mtr)) {
        /* The record did not fit on an empty undo page. ... */

        /* When we remove a page from an undo log, this is analogous to a
        pessimistic insert in a B-tree, and we must reserve the counterpart
        of the tree latch, which is the rseg mutex. We must commit the
        mini-transaction first, because it may be holding lower-level
        latches, such as SYNC_FSP and SYNC_FSP_PAGE. */
        mtr_commit(&mtr);
        mtr_start(&mtr);                    // ★★ 必须先提交 mtr 再加 rseg latch（锁序）

        if (index->table->is_temporary()) mtr.set_log_mode(MTR_LOG_NO_REDO);

        undo_ptr->rseg->latch();
        trx_undo_free_last_page(trx, undo, &mtr);
        undo_ptr->rseg->unlatch();

        err = DB_UNDO_RECORD_TOO_BIG;
        goto err_exit;
      }

      mtr_commit(&mtr);
    } else {
      /* Success */
      undo->guess_block = undo_block;
      mtr_commit(&mtr);

      undo->empty = false;
      undo->top_page_no = page_no;
      undo->top_offset = offset;
      undo->top_undo_no = trx->undo_no;

      trx->undo_no++;                       // ★ undo_no 事务内单调递增
      trx->undo_rseg_space = undo_ptr->rseg->space_id;

      mutex_exit(&trx->undo_mutex);

      *roll_ptr =
          trx_undo_build_roll_ptr(op_type == TRX_UNDO_INSERT_OP,
                                  undo_ptr->rseg->space_id, page_no, offset);
      return (DB_SUCCESS);
    }

    ut_ad(page_no == undo->last_page_no);
    ut_ad(++loop_count < 2);                // ★ 最多扩一次页

    mtr_start(&mtr);
    if (index->table->is_temporary()) mtr.set_log_mode(MTR_LOG_NO_REDO);

    /* When we add a page to an undo log, this is analogous to a pessimistic
    insert in a B-tree, and we must reserve the counterpart of the tree latch,
    which is the rseg mutex. */
    undo_ptr->rseg->latch();
    undo_block = trx_undo_add_page(trx, undo, undo_ptr, &mtr);
    undo_ptr->rseg->unlatch();

    page_no = undo->last_page_no;
  } while (undo_block != nullptr);

  ib_errf(trx->mysql_thd, IB_LOG_LEVEL_ERROR, ER_INNODB_UNDO_LOG_FULL, ...);
  err = DB_OUT_OF_FILE_SPACE;

err_exit:
  mutex_exit(&trx->undo_mutex);
  mtr_commit(&mtr);
  return (err);
}
```

> ★ 常见的"末尾调用 `trx_undo_page_set_next_prev_and_add()`"说法**不成立**。该函数只调 `report_insert` / `report_modify`；`set_next_prev_and_add` 由 `report_insert` **内部**调用，modify 路径则在**自己函数体末尾内联**实现 prev/next + 推进 FREE。

#### `trx_undo_page_report_insert()`（trx0rec.cc:483）

```cpp
static ulint trx_undo_page_report_insert(page_t *undo_page, trx_t *trx,
                                         dict_index_t *index,
                                         const dtuple_t *clust_entry,
                                         mtr_t *mtr) {
  ulint first_free;
  byte *ptr;
  ulint i;

  ut_ad(index->is_clustered());
  ut_ad(mach_read_from_2(undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_TYPE) ==
        TRX_UNDO_INSERT);

  first_free =
      mach_read_from_2(undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE);
  ptr = undo_page + first_free;

  if (trx_undo_left(undo_page, ptr) < 2 + 1 + 11 + 11) {
    return (0);      // ★ 2=next 指针, 1=type, 11+11=undo_no/table_id 变长上限
  }

  /* Reserve 2 bytes for the pointer to the next undo log record */
  ptr += 2;

  *ptr++ = TRX_UNDO_INSERT_REC;                                  // 1B type = 11
  ptr += mach_u64_write_much_compressed(ptr, trx->undo_no);      // 变长 ≤11B
  ptr += mach_u64_write_much_compressed(ptr, index->table->id);  // 变长 ≤11B

  /* Store then the fields required to uniquely determine the record
  to be inserted in the clustered index */
  for (i = 0; i < dict_index_get_n_unique(index); i++) {
    const dfield_t *field = dtuple_get_nth_field(clust_entry, i);
    ulint flen = dfield_get_len(field);

    if (trx_undo_left(undo_page, ptr) < 5) return (0);

    ptr += mach_write_compressed(ptr, flen);

    if (flen != UNIV_SQL_NULL && flen != 0) {
      if (trx_undo_left(undo_page, ptr) < flen) return (0);
      ut_memcpy(ptr, dfield_get_data(field), flen);
      ptr += flen;
    }
  }

  if (index->table->n_v_cols) {
    if (!trx_undo_report_insert_virtual(undo_page, index->table, clust_entry,
                                        &ptr)) return (0);
  }

  return (trx_undo_page_set_next_prev_and_add(undo_page, ptr, mtr));
}
```

> ★ insert undo **只存主键各列**（够定位那条被插入的行即可）——所以回滚 INSERT 就是"按主键把行删掉"。

#### `trx_undo_page_report_modify()`（trx0rec.cc:1154，节选前半）

```cpp
  if (trx_undo_left(undo_page, ptr) < 50) return 0;   // ★ 通用参数区预算 50B

  ptr += 2;                                           // 预留 next 指针

  if (!update) {
    ut_ad(!rec_get_deleted_flag(rec, dict_table_is_comp(table)));
    type_cmpl = TRX_UNDO_DEL_MARK_REC;                // 14
  } else if (rec_get_deleted_flag(rec, dict_table_is_comp(table))) {
    type_cmpl = TRX_UNDO_UPD_DEL_REC;                 // 13
    ignore_prefix = true;
  } else {
    type_cmpl = TRX_UNDO_UPD_EXIST_REC;               // 12
  }

  type_cmpl |= cmpl_info * TRX_UNDO_CMPL_INFO_MULT;   // cmpl_info 放 bit4-5
  type_cmpl_ptr = ptr;

  *ptr++ = (byte)type_cmpl;
  *type_cmpl_ptr |= TRX_UNDO_MODIFY_BLOB;             // ★ 8.0 无条件置 bit6
  *ptr++ = 0x00;                                      // ★ 8.0 新增 1B 保留 flag

  ptr += mach_u64_write_much_compressed(ptr, trx->undo_no);
  ptr += mach_u64_write_much_compressed(ptr, table->id);

  *ptr++ = (byte)rec_get_info_bits(rec, dict_table_is_comp(table));  // 1B

  /* Store the values of the system columns */
  field = rec_get_nth_field(nullptr, rec, offsets,
                            index->get_sys_col_pos(DATA_TRX_ID), &flen);
  trx_id = trx_read_trx_id(field);
  if (ignore_prefix) ignore_prefix = (trx_id != trx->id);
  ptr += mach_u64_write_compressed(ptr, trx_id);      // ★ 旧 DB_TRX_ID

  field = rec_get_nth_field(nullptr, rec, offsets,
                            index->get_sys_col_pos(DATA_ROLL_PTR), &flen);
  ptr += mach_u64_write_compressed(ptr, trx_read_roll_ptr(field)); // ★ 旧 DB_ROLL_PTR

  /* 主键各列 (len, data) */
  for (i = 0; i < dict_index_get_n_unique(index); i++) { ... }

  /* 被更新列的 (col_no, len, old_val) */
  if (update) {
    ulint n_updated = upd_get_n_fields(update);
    ptr += mach_write_compressed(ptr, n_updated);
    for (i = 0; i < upd_get_n_fields(update); i++) { ... }
  }
```

**尾部自己内联 prev/next**（trx0rec.cc:1704-1713）：

```cpp
  mach_write_to_2(ptr, first_free);                   // 末尾 2B = prev
  ptr += 2;
  mach_write_to_2(undo_page + first_free, ptr - undo_page);          // 回填 next
  mach_write_to_2(undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE,
                  ptr - undo_page);                   // 推进 FREE
  trx_undof_page_add_undo_rec_log(undo_page, first_free, ptr - undo_page, mtr);
  return first_free;                                  // ★ 返回 offset → 拼 roll_ptr
```

#### `trx_undo_page_set_next_prev_and_add()`（trx0rec.cc:178）

```cpp
static ulint trx_undo_page_set_next_prev_and_add(page_t *undo_page, byte *ptr,
                                                 mtr_t *mtr) {
  ulint first_free;
  ulint end_of_rec;
  byte *ptr_to_first_free;

  if (UNIV_UNLIKELY(trx_undo_left(undo_page, ptr) < 2)) return (0);

  ptr_to_first_free = undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE;
  first_free = mach_read_from_2(ptr_to_first_free);   // ★ 写入前的 FREE = 上一条 rec 的 offset

  /* Write offset of the previous undo log record */
  mach_write_to_2(ptr, first_free);
  ptr += 2;
  end_of_rec = ptr - undo_page;

  /* Write offset of the next undo log record */
  mach_write_to_2(undo_page + first_free, end_of_rec);

  /* Update the offset to first free undo record */
  mach_write_to_2(ptr_to_first_free, end_of_rec);

  trx_undof_page_add_undo_rec_log(undo_page, first_free, end_of_rec, mtr);

  return (first_free);
}
```

> ★ redo 只记 `(len, 从 old_free+2 开始的 len 字节)`，**不含首尾各 2 字节的 prev/next 指针**（所以 `len = new_free - old_free - 4`）：
> ```cpp
>   len = new_free - old_free - 4;
>   mach_write_to_2(log_ptr, len);
>   log_ptr += 2;
>   memcpy(log_ptr, undo_page + old_free + 2, len);   // MLOG_UNDO_INSERT
> ```

所有 DML 写 undo 都收敛到这一个函数。参数逐一剖析（`trx0rec.cc` 签名注释）：

| 参数 | 语义 |
|---|---|
| `flags` | `BTR_NO_UNDO_LOG_FLAG` 置位则**什么都不做**直接返回（`roll_ptr` 输出 0）——临时表/内部索引用 |
| `op_type` | **仅两值**：`TRX_UNDO_INSERT_OP` / `TRX_UNDO_MODIFY_OP`（record type 11/12/13/14 **不是**入口参数，见下） |
| `thr` | query thread |
| `index` | **聚簇索引**（undo 只记聚簇） |
| `clust_entry` | insert 时待插记录；update/delete-mark 时 NULL |
| `update` | update 时 update vector；否则 NULL |
| `cmpl_info` | 二级索引更新编译信息（`UPD_NODE_NO_ORD_CHANGE`）——决定排序列旧值写不写 |
| `rec` | update/delete-mark 时聚簇索引里的旧记录；insert 时 NULL |
| `offsets` | `rec_get_offsets(rec)` 的解析结果 |
| `roll_ptr` | out：undo 记录的 rollback pointer；`BTR_NO_UNDO_LOG` 时为 0 |

**insert 场景的传参对照**（`btr_cur_ins_lock_and_undo` 里的调用）——`rec=nullptr`、`offsets=nullptr`、`update=nullptr`、`clust_entry=entry`：insert 的逆操作是"按主键删掉"，undo 只需从 `clust_entry` 提取主键列，**不需要读旧记录**；update 才需要 `rec + offsets` 读旧值、`update` 提供新值。三参成对（`clust_entry`↔`update`↔`rec`）互斥，恰好对应三种 DML。

先澄清一个常见误解：`op_type` 参数**只有两个值**——`TRX_UNDO_INSERT_OP` 和 `TRX_UNDO_MODIFY_OP`。前面讲的 record type（11/12/13/14）**不是**入口参数，而是 `trx_undo_page_report_modify` 内部根据 `update` 是否为 NULL、以及 `rec` 是否已打删除标记推导出来的。

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

★ **不存在 `trx_rollback()`**（全仓 `grep "\btrx_rollback\s*("` 0 命中）。真实链路：

```
trx_rollback_for_mysql()                  trx0roll.cc:269   完整回滚
trx_rollback_last_sql_stat_for_mysql()    trx0roll.cc:284   单语句回滚
trx_rollback_to_savepoint_for_mysql()     trx0roll.cc:432   具名 savepoint 回滚
        └─→ trx_rollback_to_savepoint(trx, savept)          trx0roll.cc:142
                └─→ trx_rollback_to_savepoint_low(trx, savept)  trx0roll.cc:79  ★ static，唯一实现
```

区别**只在 `savept` 参数**：`nullptr` = 完整回滚；`&trx->last_sql_stat_start` = 单语句；`&savep->savept` = 具名 savepoint。

```cpp
/** Rollback a transaction used in MySQL. */
static void trx_rollback_to_savepoint_low(trx_t *trx, trx_savept_t *savept) {
  que_thr_t *thr;
  mem_heap_t *heap;
  roll_node_t *roll_node;

  heap = mem_heap_create(512, UT_LOCATION_HERE);
  roll_node = roll_node_create(heap);

  if (savept != nullptr) {
    roll_node->partial = true;
    roll_node->savept = *savept;
    check_trx_state(trx);
  } else {
    assert_trx_nonlocking_or_in_list(trx);
  }

  trx->error_state = DB_SUCCESS;

  if (trx_is_rseg_updated(trx)) {
    thr = pars_complete_graph_for_exec(roll_node, trx, heap, nullptr);
    // ★ 外层 graph：QUE_FORK_MYSQL_INTERFACE
    ut_a(thr == que_fork_start_command(
                    static_cast<que_fork_t *>(que_node_get_parent(thr))));

    que_run_threads(thr);                 // ① 跑外层 → trx_rollback_step()
    ut_a(roll_node->undo_thr != nullptr);
    que_run_threads(roll_node->undo_thr); // ② 跑内层 QUE_FORK_ROLLBACK graph
    // ★★ 两条 que_run_threads 就是"两层 query graph"的落点

    que_graph_free(static_cast<que_t *>(roll_node->undo_thr->common.parent));
  }

  if (savept == nullptr) {
    trx_rollback_finish(trx);             // ★ 只有完整回滚才 trx_commit()
    MONITOR_INC(MONITOR_TRX_ROLLBACK);
  } else {
    trx->lock.que_state = TRX_QUE_RUNNING;
    MONITOR_INC(MONITOR_TRX_ROLLBACK_SAVEPOINT);
  }
  ...
}
```

两层 graph 的**接缝**在 `trx_rollback_step()`（:1191）：

```cpp
que_thr_t *trx_rollback_step(que_thr_t *thr) {
  roll_node_t *node = static_cast<roll_node_t *>(thr->run_node);
  ...
  if (node->state == ROLL_NODE_SEND) {
    trx_t *trx = thr_get_trx(thr);
    ib_id_t roll_limit;

    trx_mutex_enter(trx);
    node->state = ROLL_NODE_WAIT;
    ut_a(node->undo_thr == nullptr);

    roll_limit = node->partial ? node->savept.least_undo_no : 0;
    // ★ savepoint 的 least_undo_no 在这里变成 trx->roll_limit

    trx_commit_or_rollback_prepare(trx);

    node->undo_thr = trx_rollback_start(trx, roll_limit, node->partial);
    // ★ 内层 graph（QUE_FORK_ROLLBACK）的 thr 挂到 roll_node->undo_thr
    trx_mutex_exit(trx);
  } else {
    thr->run_node = que_node_get_parent(node);
  }
  return (thr);
}

static que_t *trx_roll_graph_build(trx_t *trx, bool partial_rollback) {
  mem_heap_t *heap = mem_heap_create(512, UT_LOCATION_HERE);
  que_fork_t *fork = que_fork_create(nullptr, nullptr, QUE_FORK_ROLLBACK, heap);
  //                                                    ^^^^^^^^^^^^^^^^^^
  // ★ 内层 fork_type 与外层的 QUE_FORK_MYSQL_INTERFACE / QUE_FORK_RECOVERY 区分
  fork->trx = trx;
  que_thr_t *thr = que_thr_create(fork, heap, nullptr);
  thr->child = row_undo_node_create(trx, thr, heap, partial_rollback);
  //           ^^^^^^^^^^^^^^^^^^^ 内层节点是 undo_node_t（QUE_NODE_UNDO）
  return (fork);
}
```

#### 逆序：两个 undo 合成一个逻辑栈（`trx_roll_pop_top_rec_of_trx_low`，trx0roll.cc:920）

```cpp
  if (trx->pages_undone >= TRX_ROLL_TRUNC_THRESHOLD) {   // = 1
    rseg->latch();
    trx_roll_try_truncate(trx, undo_ptr);
    rseg->unlatch();
  }

  ins_undo = undo_ptr->insert_undo;
  upd_undo = undo_ptr->update_undo;

  // ★★★ 核心：谁 top_undo_no 大就先弹谁
  if (!ins_undo || ins_undo->empty) {
    undo = upd_undo;
  } else if (!upd_undo || upd_undo->empty) {
    undo = ins_undo;
  } else if (upd_undo->top_undo_no > ins_undo->top_undo_no) {
    undo = upd_undo;
  } else {
    undo = ins_undo;
  }

  if (!undo || undo->empty || limit > undo->top_undo_no) {
    // ★ limit 就是 savepoint 的 least_undo_no：栈顶 undo_no < limit ⇒ 部分回滚到位
    rseg->latch();
    trx_roll_try_truncate(trx, undo_ptr);
    rseg->unlatch();
    mutex_exit(&trx->undo_mutex);
    return (nullptr);
  }

  auto is_insert = (undo == ins_undo);

  *roll_ptr = trx_undo_build_roll_ptr(is_insert, undo->rseg->space_id,
                                      undo->top_page_no, undo->top_offset);
  // ★ roll_ptr 的 insert 位就是在这里按"栈顶来自哪个 undo"打上的；
  //   下游 row_undo() 用 trx_undo_roll_ptr_is_insert() 反查并分派
  ...
  undo_page = trx_roll_pop_top_rec(trx, undo, &mtr, &undo_offset);
```

外层 wrapper（:1019）说明 **redo rseg 先弹完才弹 noredo（临时表）rseg**：

```cpp
trx_undo_rec_t *trx_roll_pop_top_rec_of_trx(trx_t *trx, undo_no_t limit,
                                            roll_ptr_t *roll_ptr,
                                            mem_heap_t *heap) {
  trx_undo_rec_t *undo_rec = nullptr;

  if (trx_is_redo_rseg_updated(trx)) {
    undo_rec = trx_roll_pop_top_rec_of_trx_low(trx, &trx->rsegs.m_redo, limit,
                                               roll_ptr, heap);
  }
  if (undo_rec == nullptr && trx_is_temp_rseg_updated(trx)) {
    undo_rec = trx_roll_pop_top_rec_of_trx_low(trx, &trx->rsegs.m_noredo, limit,
                                               roll_ptr, heap);
  }
  return (undo_rec);
}
```

#### 按 roll_ptr 的 insert 位分派（`row_undo()`，row0undo.cc:248）

```cpp
    node->undo_rec = trx_roll_pop_top_rec_of_trx(trx, trx->roll_limit,
                                                 &roll_ptr, node->heap);
    if (!node->undo_rec) {
      /* Rollback completed for this query thread */
      trx->roll_limit = 0;
      return (DB_SUCCESS);
    }

    node->roll_ptr = roll_ptr;
    node->undo_no = trx_undo_rec_get_undo_no(node->undo_rec);

    if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
      node->state = UNDO_NODE_INSERT;    // → row_undo_ins()
    } else {
      node->state = UNDO_NODE_MODIFY;    // → row_undo_mod()
    }
    ...
  if (node->state == UNDO_NODE_INSERT) {
    err = row_undo_ins(node, thr);       // row0uins.cc:466
    node->state = UNDO_NODE_FETCH_NEXT;
  } else {
    err = row_undo_mod(node, thr);       // row0umod.cc:1272
  }
```

UPDATE/DELETE 的三分支在 `row_undo_mod()` 里：

```cpp
  switch (node->rec_type) {
    case TRX_UNDO_UPD_EXIST_REC:
      err = row_undo_mod_upd_exist_sec(node, thr);   // UPDATE（改了有序字段）
      break;
    case TRX_UNDO_DEL_MARK_REC:
      err = row_undo_mod_del_mark_sec(node, thr);    // DELETE：去掉 delete-mark
      break;
    case TRX_UNDO_UPD_DEL_REC:
      err = row_undo_mod_upd_del_sec(node, thr);     // 更新一条已被删标记的记录
      break;
  }
  if (err == DB_SUCCESS) {
    err = row_undo_mod_clust(node, thr);             // ★ 最后才动聚簇索引
  }
```

> ★ 常见说法里的 `row_undo_upd_del_mark_or_remove_sec()` **不存在**，真名是 `row_undo_mod_del_mark_or_remove_sec()`（row0umod.cc:550）。
>
> ★ **回滚期间任何错误都是 `ib::fatal`（直接崩库）**——`row_undo_step()`：`if (err != DB_SUCCESS) { ... ib::fatal(...) "Error (...) in rollback." }`。因为回滚不能失败。

#### savepoint 只需要一个 undo 号

```cpp
trx_savept_t trx_savept_take(trx_t *trx) {
  trx_savept_t savept;
  savept.least_undo_no = trx->undo_no;   // ★ 就这一个字段
  return (savept);
}
```

> 没有任何物理标记。回滚到 savepoint = 只回滚 `undo_no ≥ least_undo_no` 的记录，靠上面的 `limit > undo->top_undo_no` 判停。

#### XA PREPARE / ROLLBACK 的 undo 状态机（`trx_undo_set_state_at_prepare`，trx0undo.cc:1849）

⚠️ 这个函数是**双向**的：`rollback=false` = XA PREPARE，`rollback=true` = XA ROLLBACK。

```cpp
  if (rollback) {
    ut_ad(undo->is_prepared());
    trx_undo_gtid_write(trx, undo_header, undo, mtr, !rollback);
    mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, TRX_UNDO_ACTIVE, MLOG_2BYTES, mtr);
    // ★ PREPARED → ACTIVE：保证崩溃恢复继续回滚
    return (undo_page);
  }

  ut_ad(undo->state == TRX_UNDO_ACTIVE);
  undo->set_prepared(trx->xid);                 // ★ 置 PREPARED 态 + 记 XID
  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, undo->state, MLOG_2BYTES, mtr);
  mlog_write_ulint(undo_header + TRX_UNDO_FLAGS, undo->flag, MLOG_1BYTE, mtr);
  trx_undo_write_xid(undo_header, &undo->xid, mtr);   // ★ XID 落盘
```

commit 时的状态机（`trx_undo_set_state_at_finish`，trx0undo.cc:1808）：

```cpp
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
                             TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;        // ① 只占 1 页且剩余够 → 进 cached list 复用
  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;       // ② insert undo：commit 后无人需要 → 待释放
  } else {
    state = TRX_UNDO_TO_PURGE;      // ③ update undo：待 purge → 进 history list
  }
  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, state, MLOG_2BYTES, mtr);
```

> ★ `TRX_UNDO_PAGE_REUSE_LIMIT` = `3 * UNIV_PAGE_SIZE / 4`（16K 页 → 12288 字节）。这就是"能被 cache 的 undo 段必须只有 1 页"的量化判据。

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

### XA：undo segment 的 PREPARED 状态机

7 个状态（`storage/innobase/include/trx0undo.h:309-333`）：

```cpp
/** Types of an undo log segment */
/** contains undo entries for inserts */
constexpr uint32_t TRX_UNDO_INSERT = 1;
/** contains undo entries for updates and delete markings: in short, modifys
 (the name 'UPDATE' is a historical relic) */
constexpr uint32_t TRX_UNDO_UPDATE = 2;

/* States of an undo log segment */
/** contains an undo log of an active  transaction */
constexpr uint32_t TRX_UNDO_ACTIVE = 1;
/** cached for quick reuse */
constexpr uint32_t TRX_UNDO_CACHED = 2;
/** insert undo segment can be freed */
constexpr uint32_t TRX_UNDO_TO_FREE = 3;
/** update undo segment will not be reused: it can be freed in purge when all
 undo data in it is removed */
constexpr uint32_t TRX_UNDO_TO_PURGE = 4;
/** contains an undo log of an prepared transaction for a server version older
 * than 8.0.29 */
constexpr uint32_t TRX_UNDO_PREPARED_80028 = 5;
/** contains an undo log of an prepared transaction */
constexpr uint32_t TRX_UNDO_PREPARED = 6;
/* contains an undo log of a prepared transaction that has been processed by the
 * transaction coordinator */
constexpr uint32_t TRX_UNDO_PREPARED_IN_TC = 7;
```

物理落点在 **segment header 偏移 0**（不是 log header）：

```cpp
/*-------------------------------------------------------------*/
/** TRX_UNDO_ACTIVE, ... */
constexpr uint32_t TRX_UNDO_STATE = 0;
/** Offset of the last undo log header on the segment header page, 0 if none */
constexpr uint32_t TRX_UNDO_LAST_LOG = 2;
```

状态迁移（`trx0undo.ic:286-302`）：

```cpp
/* Set undo segment to prepared state and set XID
@param[in]   in_xid  transaction XID. */
inline void trx_undo_t::set_prepared(const XID *in_xid) {
  state = TRX_UNDO_PREPARED;
  xid = *in_xid;
  flag |= TRX_UNDO_FLAG_XID;
}

inline void trx_undo_t::set_prepared_in_tc() {
  ut_ad(!xid.is_null());
  state = TRX_UNDO_PREPARED_IN_TC;
}

inline bool trx_undo_t::is_prepared() const {
  return state == TRX_UNDO_PREPARED_80028 || state == TRX_UNDO_PREPARED ||
         state == TRX_UNDO_PREPARED_IN_TC;
}
```

`trx_undo_set_prepared_in_tc()`（`trx0undo.cc:1892-1921`）——**外部 XA 的 prepare GTID 就在这里落盘**：

```cpp
page_t *trx_undo_set_prepared_in_tc(trx_t *trx, trx_undo_t *undo, mtr_t *mtr) {
  ...
  offset = mach_read_from_2(seg_hdr + TRX_UNDO_LAST_LOG);
  undo_header = undo_page + offset;

  ut_ad(undo->state == TRX_UNDO_PREPARED_80028 ||
        undo->state == TRX_UNDO_PREPARED);

  /* Write GTID information if there. */
  trx_undo_gtid_write(trx, undo_header, undo, mtr, true);

  undo->set_prepared_in_tc();

  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, undo->state, MLOG_2BYTES, mtr);

  return (undo_page);
}
```

唯一路径（`trx0trx.cc:3096-3150`）：

```cpp
static lsn_t trx_set_prepared_in_tc_low(trx_t *trx, trx_undo_ptr_t *undo_ptr) {
  if (undo_ptr->insert_undo != nullptr || undo_ptr->update_undo != nullptr) {
    mtr_t mtr;
    trx_rseg_t *rseg = undo_ptr->rseg;

    mtr_start_sync(&mtr);

    /* Change the undo log segment states from TRX_UNDO_ACTIVE to
    TRX_UNDO_PREPARED: these modifications to the file data
    structure define the transaction as prepared in the file-based
    world, at the serialization point of lsn. */

    rseg->latch();

    if (undo_ptr->insert_undo != nullptr) {
      /* It is not necessary to obtain trx->undo_mutex here
      because only a single OS thread is allowed to do the
      transaction prepare for this transaction. */
      trx_undo_set_prepared_in_tc(trx, undo_ptr->insert_undo, &mtr);
    }

    if (undo_ptr->update_undo != nullptr) {
      trx_undo_gtid_set(trx, undo_ptr->update_undo, true);
      trx_undo_set_prepared_in_tc(trx, undo_ptr->update_undo, &mtr);
    }

    rseg->unlatch();

    /*--------------*/
    /* This mtr commit makes the transaction prepared in
    file-based world. */
    mtr_commit(&mtr);
```

/* 【注】① `mtr_start_sync()`（同步 redo）+ `mtr_commit()` 就是注释里说的
   "the serialization point of lsn"——**prepare 的原子性由这个 mtr 保证**；
   ② insert undo 与 update undo **都要**置 `PREPARED_IN_TC`，但 GTID 只写在 update undo 上；
   ③ `TRX_UNDO_PREPARED_80028` 是 8.0.29 之前的历史状态，仍被 `is_prepared()` 兼容。 */

反向迁移（XA ROLLBACK）：`trx_undo_set_state_at_prepare(..., rollback=true)` 把状态写回 `TRX_UNDO_ACTIVE`（`trx0undo.cc:1868-1877`），并顺带写一次 commit 槽的 GTID（`trx_undo_gtid_write(..., !rollback)`）。

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

★ 它**没有** `switch(UPD_EXIST / UPD_DEL / DEL_MARK)` 三分支。只有两个分支：

```cpp
bool trx_undo_prev_version_build(const rec_t *index_rec, mtr_t *index_mtr,
                                 const rec_t *rec, const dict_index_t *index,
                                 ulint *offsets, mem_heap_t *heap,
                                 rec_t **old_vers, mem_heap_t *v_heap,
                                 const dtuple_t **vrow, ulint v_status,
                                 lob::undo_vers_t *lob_undo) {
  ...
  roll_ptr = row_get_rec_roll_ptr(rec, index, offsets);   // ★ 版本链指针

  *old_vers = nullptr;

  if (trx_undo_roll_ptr_is_insert(roll_ptr)) {
    /* The record rec is the first inserted version */
    return true;        // ★ 分支①：insert undo —— 没有更早版本，*old_vers 保持 NULL
  }

  rec_trx_id = row_get_rec_trx_id(rec, index, offsets);
  bool is_temp = index->table->is_temporary();

  if (trx_undo_get_undo_rec(roll_ptr, rec_trx_id, heap, is_temp,
                            index->table->name, &undo_rec)) {
    if (v_status & TRX_UNDO_PREV_IN_PURGE) {
      undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
    } else {
      /* The undo record may already have been purged ... */
      return false;     // ★ 返回 false = 历史丢失 → 上层 DB_MISSING_HISTORY
    }
  }

  ptr = trx_undo_rec_get_pars(undo_rec, &type, &cmpl_info, &dummy_extern,
                              &undo_no, &table_id, type_cmpl);

  if (table_id != index->table->id) {
    /* The table should have been rebuilt, but purge has not yet removed
    the undo log records for the now-dropped old table (table_id). */
    return true;
  }

  ptr = trx_undo_update_rec_get_sys_cols(ptr, &trx_id, &roll_ptr, &info_bits);
  // ★ 读出"上一版"的 trx_id / roll_ptr / info_bits

  ptr = trx_undo_rec_skip_row_ref(ptr, index);
  // ★ 行引用（主键）对 MVCC 无用（我们已经站在 rec 上），直接跳过

  ptr = trx_undo_update_rec_get_update(ptr, index, type, trx_id, roll_ptr,
                                       info_bits, heap, &update, lob_undo,
                                       type_cmpl);
  // ★ 分支② 的真正入口：三种 update 类型的差异在函数内部用 n_fields 表达
```

**三种 update 类型的差异被下沉到 `trx_undo_update_rec_get_update()`**（trx0rec.cc:1740）：

```cpp
  if (type != TRX_UNDO_DEL_MARK_REC) {
    n_fields = mach_read_next_compressed(&ptr);
  } else {
    n_fields = 0;      // ★★ DELETE(DEL_MARK) 没有列值要回退，只回写两个系统列
  }

  update = upd_create(n_fields + 2, heap);   // ★ 尾部 +2 给 DB_TRX_ID / DB_ROLL_PTR
  update->table = index->table;
  update->info_bits = info_bits;

  /* Store first trx id and roll ptr to update vector */
  upd_field = upd_get_nth_field(update, n_fields);
  buf = static_cast<byte *>(mem_heap_alloc(heap, DATA_TRX_ID_LEN));
  trx_write_trx_id(buf, trx_id);
  upd_field_set_field_no(upd_field, index->get_sys_col_pos(DATA_TRX_ID), index);
  dfield_set_data(&(upd_field->new_val), buf, DATA_TRX_ID_LEN);

  upd_field = upd_get_nth_field(update, n_fields + 1);
  buf = static_cast<byte *>(mem_heap_alloc(heap, DATA_ROLL_PTR_LEN));
  trx_write_roll_ptr(buf, roll_ptr);
  upd_field_set_field_no(upd_field, index->get_sys_col_pos(DATA_ROLL_PTR), index);
  dfield_set_data(&(upd_field->new_val), buf, DATA_ROLL_PTR_LEN);
  ...
```

**`*old_vers` 到底是什么**——不是"undo 里保存的旧记录"，而是**在 heap 上新物化的一条 rec**：

```cpp
  if (row_upd_changes_field_size_or_external(index, offsets, update)) {
    /* 长度变了 / 有外部列：rec → dtuple → 逐列替换 → 重新物化 */
    entry = row_rec_to_index_entry(rec, index, offsets, heap);
    row_upd_index_replace_new_col_vals(entry, index, update, heap);
    buf = static_cast<byte *>(
        mem_heap_alloc(heap, rec_get_converted_size(index, entry)));
    *old_vers = rec_convert_dtuple_to_rec(buf, index, entry);
  } else {
    /* 定长：直接 memcpy 当前记录，再原地覆写被改列 + 系统列 */
    buf = static_cast<byte *>(mem_heap_alloc(heap, rec_offs_size(offsets)));
    *old_vers = rec_copy(buf, rec, offsets);
    rec_offs_make_valid(*old_vers, index, offsets);
    row_upd_rec_in_place(*old_vers, index, offsets, update, nullptr);
  }
```

> ★★ 新物化的 `*old_vers` 自带**新的 roll_ptr**（指向更早的一条 undo），所以下一次调用可以再往回跳一跳——**版本链就是这么串起来的**。

#### 安全边界：从 roll_ptr 到 undo 页（`trx_undo_get_undo_rec`，trx0rec.cc:2440）

```cpp
[[nodiscard]] static bool trx_undo_get_undo_rec(roll_ptr_t roll_ptr,
                                                trx_id_t trx_id,
                                                mem_heap_t *heap, bool is_temp,
                                                const table_name_t &name,
                                                trx_undo_rec_t **undo_rec) {
  bool missing_history;

  rw_lock_s_lock(&purge_sys->latch, UT_LOCATION_HERE);

  missing_history = purge_sys->view.changes_visible(trx_id, name);
  // ★★ purge_sys->view = 全局最老活视图 = purge 边界。
  //    连它都可见 ⇒ 这段历史随时会被 purge 清掉 ⇒ 不能依赖

  if (!missing_history) {
    *undo_rec = trx_undo_get_undo_rec_low(roll_ptr, heap, is_temp);
  }

  rw_lock_s_unlock(&purge_sys->latch);

  return (missing_history);
}
```

```cpp
[[nodiscard]] static trx_undo_rec_t *trx_undo_get_undo_rec_low(
    roll_ptr_t roll_ptr, mem_heap_t *heap, bool is_temp) {
  trx_undo_rec_t *undo_rec;
  ulint rseg_id;
  space_id_t space_id;
  page_no_t page_no;
  ulint offset;
  const page_t *undo_page;
  bool is_insert;
  mtr_t mtr;

  trx_undo_decode_roll_ptr(roll_ptr, &is_insert, &rseg_id, &page_no, &offset);
  space_id = trx_rseg_id_to_space_id(rseg_id, is_temp);
  // ★ 临时表的 undo 落在 noredo（临时）undo 表空间
  ...
  undo_page = trx_undo_page_get_s_latched(page_id_t(space_id, page_no),
                                          page_size, &mtr);
  undo_rec = trx_undo_rec_copy(undo_page, static_cast<uint32_t>(offset), heap);
  mtr_commit(&mtr);
  return (undo_rec);
}
```

> ★ **两个函数都是 `static`**（不是公开 API），且带 `[[nodiscard]]`。

#### 上层多跳循环（`row_vers_build_for_consistent_read`，row0vers.cc:1258）

```cpp
  for (;;) {                          // ★ 每循环一次 = 沿版本链往回跳一跳
    mem_heap_t *prev_heap = heap;
    heap = mem_heap_create(1024, UT_LOCATION_HERE);
    if (vrow) *vrow = nullptr;

    bool purge_sees =
        trx_undo_prev_version_build(rec, mtr, version, index, *offsets, heap,
                                    &prev_version, nullptr, vrow, 0, lob_undo);

    err = (purge_sees) ? DB_SUCCESS : DB_MISSING_HISTORY;
    // ★ false → DB_MISSING_HISTORY（"snapshot too old" 类语义）

    if (prev_heap != nullptr) mem_heap_free(prev_heap);
    // ★ 中间版本的 heap 每跳完就释放，只有最终结果拷进 in_heap

    if (prev_version == nullptr) {
      *old_vers = nullptr;    // 它是"本事务首次插入"的版本
      break;
    }

    *offsets = rec_get_offsets(prev_version, index, *offsets, ULINT_UNDEFINED,
                               UT_LOCATION_HERE, offset_heap);
    trx_id = row_get_rec_trx_id(prev_version, index, *offsets);

    if (view->changes_visible(trx_id, index->table->name)) {
      /* The view already sees this version */
      buf = static_cast<byte *>(mem_heap_alloc(in_heap, rec_offs_size(*offsets)));
      *old_vers = rec_copy(buf, prev_version, *offsets);
      rec_offs_make_valid(*old_vers, index, *offsets);
      ...
      break;
    }

    version = prev_version;   // ★ 继续往前跳
  }
```

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

★ **不存在** `trx_undo_add_to_history()` / `trx_purge_add_undo_to_history()`（全仓 0 命中）。真名是 **`trx_purge_add_update_undo_to_history()`**（`trx0purge.cc:344`）：

```cpp
/** Adds the update undo log as the first log in the history list. Removes the
 update undo log segment from the rseg slot if it is too big for reuse. */
void trx_purge_add_update_undo_to_history(
    trx_t *trx, trx_undo_ptr_t *undo_ptr, page_t *undo_page,
    bool update_rseg_history_len, ulint n_added_logs, mtr_t *mtr) {
  trx_undo_t *undo;
  trx_rseg_t *rseg;
  trx_rsegf_t *rseg_header;
  trx_ulogf_t *undo_header;

  undo = undo_ptr->update_undo;
  rseg = undo->rseg;

  rseg_header = trx_rsegf_get(undo->rseg->space_id, undo->rseg->page_no,
                              undo->rseg->page_size, mtr);
  undo_header = undo_page + undo->hdr_offset;

  if (undo->state != TRX_UNDO_CACHED) {
    ulint hist_size;

    /* The undo log segment will not be reused */
    trx_rsegf_set_nth_undo(rseg_header, undo->id, FIL_NULL, mtr);
    // ★ 把这个 undo 从 rseg 的 slot 摘掉（不再被复用）
    MONITOR_DEC(MONITOR_NUM_UNDO_SLOT_USED);

    hist_size =
        mtr_read_ulint(rseg_header + TRX_RSEG_HISTORY_SIZE, MLOG_4BYTES, mtr);
    mlog_write_ulint(rseg_header + TRX_RSEG_HISTORY_SIZE,
                     hist_size + undo->size, MLOG_4BYTES, mtr);
    // ★ history list 的"总页数" += 本 undo 的页数（undo 表空间 truncate 的判据）
  }

  /* Add the log as the first in the history list */
  flst_add_first(rseg_header + TRX_RSEG_HISTORY,
                 undo_header + TRX_UNDO_HISTORY_NODE, mtr);
  // ★★ 头插：最新的 undo log 排在最前，purge 从尾部（最老）取

  if (update_rseg_history_len) {
    trx_sys->rseg_history_len.fetch_add(n_added_logs);
    if (trx_sys->rseg_history_len.load() >
        srv_n_purge_threads * srv_purge_batch_size) {
      srv_wake_purge_thread_if_not_active();
      // ★ 超过"线程数 × 批量"才唤醒，避免频繁唤醒
    }
  }

  /* Update maximum transaction number for this rollback segment. */
  mlog_write_ull(rseg_header + TRX_RSEG_MAX_TRX_NO, trx->no, mtr);
  /* Write the trx number to the undo log header */
  mlog_write_ull(undo_header + TRX_UNDO_TRX_NO, trx->no, mtr);
  // ★ trx->no 是"提交序号"，purge 按它排序清理

  if (!undo->del_marks) {
    mlog_write_ulint(undo_header + TRX_UNDO_DEL_MARKS, false, MLOG_2BYTES, mtr);
  }

  trx_undo_gtid_write(trx, undo_header, undo, mtr, false);   // ★ GTID 落 undo

  if (rseg->last_page_no == FIL_NULL) {
    rseg->last_page_no = undo->hdr_page_no;
    rseg->last_offset = undo->hdr_offset;
    rseg->last_trx_no = trx->no;
    rseg->last_del_marks = undo->del_marks;
  }
}
```

出链对照（`trx_purge_remove_log_hdr`，trx0purge.cc:433）：

```cpp
static void trx_purge_remove_log_hdr(trx_rsegf_t *rseg_hdr,
                                     trx_ulogf_t *log_hdr, mtr_t *mtr) {
  flst_remove(rseg_hdr + TRX_RSEG_HISTORY, log_hdr + TRX_UNDO_HISTORY_NODE, mtr);
  trx_sys->rseg_history_len.fetch_sub(1);
}
```

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

★ **不存在** `purge_coordinator_thread()` / `purge_worker_thread()`。真名是 **`srv_purge_coordinator_thread()`**（`srv0srv.cc:3094`）与 **`srv_worker_thread()`**（`srv0srv.cc:2850`），且**都在 `srv0srv.cc`，不在 `trx0purge.cc`**。

```cpp
/** Purge coordinator thread that schedules the purge tasks. */
void srv_purge_coordinator_thread() {
  ...
  slot = srv_reserve_slot(SRV_PURGE);
  ulint rseg_history_len = trx_sys->rseg_history_len;

  do {
    if (srv_shutdown_state.load() < SRV_SHUTDOWN_PURGE &&
        (purge_sys->state == PURGE_STATE_STOP || n_total_purged == 0)) {
      srv_purge_coordinator_suspend(slot, rseg_history_len);
      // ★ 唯一的"睡眠点"
    }
    if (srv_purge_should_exit(n_total_purged)) break;

    n_total_purged = 0;
    rseg_history_len = srv_do_purge(&n_total_purged);   // ★ 真正干活
  } while (!srv_purge_should_exit(n_total_purged));
  ...
  /* Ensure that all records are purged if it is not a fast shutdown. */
  while (srv_fast_shutdown == 0 && n_pages_purged > 0) {
    n_pages_purged = trx_purge(1, srv_purge_batch_size, false);  // ★ 关机前单线程扫尾
  }
  ...
  /* Ensure that all the worker threads quit. */
  if (srv_n_purge_threads > 1) {
    srv_release_threads(SRV_WORKER, srv_n_purge_threads - 1);
    // ★ worker 必须在 coordinator 之后退出，否则 coordinator 会在
    //   trx_purge_wait_for_workers_to_complete() 里死等
  }
}
```

**worker 是纯粹的无限等待**（没有 10ms 轮询）：

```cpp
void srv_worker_thread() {
  ...
  do {
    srv_suspend_thread(slot);
    os_event_wait(slot->event);      // ★ 完全由 coordinator 入队后 release_threads 唤醒

    if (srv_task_execute()) {        // 取一个 thr 并 que_run_threads(thr)
      srv_wake_purge_thread_if_not_active();
    }
  } while (purge_sys->state != PURGE_STATE_EXIT);
  // ★ 退出条件只看 coordinator 是否已置 PURGE_STATE_EXIT
}
```

> ★ `srv_start_purge_threads()`（srv0start.cc:2443）里 `srv_threads.m_purge_workers[0] = srv_threads.m_purge_coordinator;`——**coordinator 自己也算一个 worker**，因为它同步跑最后一个 thr（见 `trx_purge()`）。

#### ★ 10ms 轮询，空闲才升级为无限等待（`srv_purge_coordinator_suspend`，srv0srv.cc:2984）

```cpp
static void srv_purge_coordinator_suspend(srv_slot_t *slot,
                                          ulint rseg_history_len) {
  bool stop = false;

  /** Maximum wait time on the purge event. */
  constexpr std::chrono::milliseconds SRV_PURGE_MAX_TIMEOUT{10};   // ★★ 10ms

  int64_t sig_count = srv_suspend_thread(slot);

  do {
    ulint ret;

    rw_lock_x_lock(&purge_sys->latch, UT_LOCATION_HERE);
    purge_sys->running = false;      // ★ 先置 running=false，让 trx_purge_stop() 能看见
    rw_lock_x_unlock(&purge_sys->latch);

    if (stop) {
      os_event_wait_low(slot->event, sig_count);        // ★ 第二档：无限等待
      ret = 0;
    } else if (rseg_history_len <= trx_sys->rseg_history_len.load()) {
      ret =
          os_event_wait_time_low(slot->event, SRV_PURGE_MAX_TIMEOUT, sig_count);
      // ★ 第一档：最多等 10ms
    } else {
      /* We don't want to waste time waiting, if the history list increased
      by the time we got here ... */
      ret = 0;                       // ★ history 还在变长 → 根本不睡，立刻回去 purge
    }
    ...
    stop = (srv_shutdown_state.load() < SRV_SHUTDOWN_PURGE &&
            purge_sys->state == PURGE_STATE_STOP);      // ★ 被显式 STOP → 下一轮进无限等待档
    ...
    if (ret == OS_SYNC_TIME_EXCEEDED) {
      /* No new records added since wait started then simply wait for new
      records. The magic number 5000 is an approximation for the case where
      we have cached UNDO log records which prevent truncate of the UNDO
      segments. */
      if (rseg_history_len == trx_sys->rseg_history_len &&
          trx_sys->rseg_history_len < 5000) {
        stop = true;    // ★★ "完全空闲"判据：超时了 且 history 长度没变 且 < 5000
      }
    }
  } while (stop);
  ...
}
```

> ★★ 三档总结：**忙**（history 变长）→ 不睡；**一般** → 10ms 超时等；**完全空闲**（超时 + history 没变 + <5000）→ 无限等。所以"10ms 轮询"只在中等负载下成立，真空闲时不做无用功。

#### ★ 自适应线程数：看 history 的"变化方向"（`srv_do_purge`，srv0srv.cc:2904）

⚠️ **不存在** `srv_update_purge_thread_count()`——逻辑**内联**在这里，状态就是三个 `static` 局部变量：

```cpp
static ulint srv_do_purge(ulint *n_total_purged) {
  ulint n_pages_purged;

  static ulint count = 0;
  static ulint n_use_threads = 0;          // ★ 自适应线程数
  static uint64_t rseg_history_len = 0;
  ulint old_activity_count = srv_get_activity_count();
  bool need_explicit_truncate = false;

  const auto n_threads = srv_threads.m_purge_workers_n;

  if (n_use_threads == 0) n_use_threads = n_threads;

  do {
    if (trx_sys->rseg_history_len.load() > rseg_history_len ||
        (srv_max_purge_lag > 0 && rseg_history_len > srv_max_purge_lag)) {
      /* History length is now longer than what it was when we took the last
      snapshot. Use more threads. */
      if (n_use_threads < n_threads) {
        ++n_use_threads;                   // ★ 变长了 → 加一个线程
      }
    } else if (srv_check_activity(old_activity_count) && n_use_threads > 1) {
      /* History length same or smaller since last snapshot, use fewer
      threads. */
      --n_use_threads;                     // ★ 没变长 且 期间有 DML 活动 → 追得上，减一个
      old_activity_count = srv_get_activity_count();
    }

    /* Take a snapshot of the history list before purge. */
    if ((rseg_history_len = trx_sys->rseg_history_len.load()) == 0) break;

    bool do_truncate = need_explicit_truncate ||
                       srv_shutdown_state.load() == SRV_SHUTDOWN_PURGE ||
                       (++count % srv_purge_rseg_truncate_frequency) == 0;
    // ★ innodb_purge_rseg_truncate_frequency 控制 rseg history truncate 频率

    n_pages_purged = trx_purge(n_use_threads, srv_purge_batch_size, do_truncate);
    *n_total_purged += n_pages_purged;

    need_explicit_truncate = (n_pages_purged == 0);
    if (need_explicit_truncate) {
      undo::spaces->s_lock();
      need_explicit_truncate =
          (undo::spaces->find_first_inactive_explicit(nullptr) != nullptr);
      // ★ 这一批啥也没清 → 可能只是有 SET INACTIVE 的表空间等着被 truncate
      undo::spaces->s_unlock();
    }
  } while (purge_sys->state == PURGE_STATE_RUN &&
           (n_pages_purged > 0 || need_explicit_truncate) &&
           !srv_purge_should_exit(n_pages_purged));

  return rseg_history_len;
}
```

> ⚠️ 也不存在 `srv_purge()`；真正驱动一次批量的是这个 `static srv_do_purge()`。

#### ★ purge 滞后限流：延迟加在 DML 上，不在 purge 上

```cpp
// trx0purge.cc:2394
static ulint trx_purge_dml_delay(void) {
  ulint delay = 0; /* in microseconds; default: no delay */

  if (srv_max_purge_lag > 0 && trx_sys->rseg_history_len.load() >
                                   srv_n_purge_threads * srv_purge_batch_size) {
    float ratio;
    ratio = float(trx_sys->rseg_history_len.load()) / srv_max_purge_lag;

    if (ratio > 1.0) {
      /* If the history list length exceeds the srv_max_purge_lag, the data
      manipulation statements are delayed by at least 5 microseconds. */
      delay = (ulint)((ratio - 0.9995) * 10000);   // ★ 超出 0.05% 起步
    }
    if (delay > srv_max_purge_lag_delay) {
      delay = srv_max_purge_lag_delay;             // ★ innodb_max_purge_lag_delay 封顶
    }
    MONITOR_SET(MONITOR_DML_PURGE_DELAY, delay);
  }
  return (delay);
}

// trx_purge() 第一件事：
  srv_dml_needed_delay = trx_purge_dml_delay();

// row0mysql.cc:139 —— 落地点，直接 sleep 在用户 DML 线程上
static void row_mysql_delay_if_needed(void) {
  if (srv_dml_needed_delay) {
    std::this_thread::sleep_for(
        std::chrono::microseconds(srv_dml_needed_delay));
  }
}
```

> ⚠️ **不存在** `srv_delay_dml`；真名是 `srv_dml_needed_delay`（`srv0srv.cc:129`）。
>
> ★ 设计意图很明确：**purge 线程永远全速跑**，滞后时惩罚的是**产生新 undo 的 DML 线程**，形成负反馈。

线程创建（`srv0start.cc srv_start_purge_threads`）：1 个 coordinator + N-1 个 worker，★ **`m_purge_workers[0]` 就是 coordinator 本身**（`Srv_threads` 注释原文："the m_purge_workers[0] is the same shared state as m_purge_coordinator"）：

```cpp
void srv_start_purge_threads() {
  if (srv_start_state_is_set(SRV_START_STATE_PURGE)) return;   // 升级时可能已提前启动
  srv_threads.m_purge_coordinator = os_thread_create(srv_purge_thread_key, 0, srv_purge_coordinator_thread);
  srv_threads.m_purge_workers[0] = srv_threads.m_purge_coordinator;   // ★ 0 号就是它
  /* We've already created the purge coordinator thread above. */
  for (size_t i = 1; i < srv_threads.m_purge_workers_n; ++i)
    srv_threads.m_purge_workers[i] = os_thread_create(srv_worker_thread_key, i, srv_worker_thread);
  ...
}
```

`innodb_purge_threads` 默认 **4**、**READONLY**（范围 1..32，`MAX_PURGE_THREADS = 32`）；`innodb_purge_batch_size` 默认 **300**（范围 1..5000；★ C 初值 20 被 sysvar 覆盖，注释称 20 是"5.5 的默认值"）。

#### ★ coordinator 主循环：10ms 轮询，空闲才升级为无限等待

```cpp
void srv_purge_coordinator_thread() {
  slot = srv_reserve_slot(SRV_PURGE);              // sys_threads[SRV_PURGE_SLOT] == slot 1
  ulint rseg_history_len = trx_sys->rseg_history_len;
  do {
    if (srv_shutdown_state.load() < SRV_SHUTDOWN_PURGE &&
        (purge_sys->state == PURGE_STATE_STOP || n_total_purged == 0)) {
      srv_purge_coordinator_suspend(slot, rseg_history_len);   // ★ 睡眠点
    }
    if (srv_purge_should_exit(n_total_purged)) break;
    n_total_purged = 0;
    rseg_history_len = srv_do_purge(&n_total_purged);
  } while (!srv_purge_should_exit(n_total_purged));
```

★ 睡眠的真实常量是 **`SRV_PURGE_MAX_TIMEOUT{10}`（10 毫秒）**——不是"一直睡到被唤醒"。`srv_purge_coordinator_suspend()` 的三态：

```cpp
  constexpr std::chrono::milliseconds SRV_PURGE_MAX_TIMEOUT{10};
  ...
  if (stop) {
    os_event_wait_low(slot->event, sig_count);                      // ① 无限期等（停止 / 彻底清闲）
  } else if (rseg_history_len <= trx_sys->rseg_history_len.load()) {
    ret = os_event_wait_time_low(slot->event, SRV_PURGE_MAX_TIMEOUT, sig_count);  // ② 10ms 超时轮询
  } else {
    ret = 0;                                                        // ③ history 在变长 → 不睡，立刻再干
  }
  ...
  if (ret == OS_SYNC_TIME_EXCEEDED) {
    if (rseg_history_len == trx_sys->rseg_history_len &&
        trx_sys->rseg_history_len < 5000) {                         // ★ magic number 5000
      stop = true;                                                  // 彻底清闲 → 升级为 ①
    }
  }
```

⇒ **purge 空闲时的真实行为是"10ms 一次轮询"，只有连续超时且 history 没变化且 < 5000 条时才转入无限期等待**。这就是"purge 看着闲却仍有一点周期性开销"的来源。

★ **等待的事件不是 `srv_purge_event`（不存在）**，而是 slot 机制的 `slot->event`（`srv_reserve_slot(SRV_PURGE)` 返回，`srv_release_threads(SRV_PURGE,1)` → `os_event_set` 唤醒）。`purge_sys->event` 是另一回事：它是 **STOP 回执信号**，只在 `trx_purge_stop()` 时由 coordinator 置位通知等待者。

#### ★ 自适应线程数：看 history 的"变化方向"

```cpp
static ulint srv_do_purge(ulint *n_total_purged) {
  static ulint n_use_threads = 0;                  // 自适应后的实际并发度（static 保持）
  const auto n_threads = srv_threads.m_purge_workers_n;   // 上限 = innodb_purge_threads
  do {
    if (trx_sys->rseg_history_len.load() > rseg_history_len ||
        (srv_max_purge_lag > 0 && rseg_history_len > srv_max_purge_lag)) {
      if (n_use_threads < n_threads) ++n_use_threads;     // ★ 追不上 → 加一个
    } else if (srv_check_activity(old_activity_count) && n_use_threads > 1) {
      --n_use_threads;                                    // ★ 追上了 → 减一个
    }
    ...
    n_pages_purged = trx_purge(n_use_threads, srv_purge_batch_size, do_truncate);
```

判定依据**不是** `innodb_purge_batch_size`，而是 **`rseg_history_len` 相对上一批快照的方向**：变长就加线程，持平/变短就减；步长 ±1，每轮最多调一次。purge lag 限流生效时也算"追不上"。

#### worker 主循环与任务队列

```cpp
void srv_worker_thread() {
  slot = srv_reserve_slot(SRV_WORKER);            // 从 sys_threads[2] 起找空槽
  do {
    srv_suspend_thread(slot);
    os_event_wait(slot->event);                   // ★ 无限期等待（无超时、无睡眠常量）
    if (srv_task_execute()) srv_wake_purge_thread_if_not_active();
  } while (purge_sys->state != PURGE_STATE_EXIT);
```

★ **`SRV_WORKER_SLEEP` 不存在**——worker 就是死等事件。任务队列是 `srv_sys->tasks`（链表）+ `srv_sys->tasks_mutex`（mutex，不是条件变量），入队 `srv_que_task_enqueue_low()` 末尾 `srv_release_threads(SRV_WORKER, 1)` 唤醒一个。

★ **coordinator 自己也跑一份**（`trx_purge()`）：投递 `n_purge_threads - 1` 个任务给 worker，剩下 1 个自己 `que_run_threads()` 同步执行，再 `trx_purge_wait_for_workers_to_complete()` 等齐（**前 10 次 `yield` 自旋，之后每 20µs 睡一次**）。

#### 唤醒源（谁叫醒 purge）

| 来源 | 位置 | 条件 |
|---|---|---|
| **事务提交写 history list**（主路径） | `trx0purge.cc` 的 `trx_purge_add_update_undo_to_history()` 末尾 | ★ 阈值 = `innodb_purge_threads × innodb_purge_batch_size`（默认 4×300 = **1200**） |
| master 线程 1 秒巡检 | `srv_master_do_active_tasks()` / `_idle_tasks()` | `rseg_history_len > 0` |
| worker 反馈 | `srv_worker_thread()` | 取到任务就再叫一次 coordinator |
| 统计重算 / undo 表空间 INACTIVE | `dict0stats.cc` / `ha_innodb.cc` | "wakes the last purge batch for exact recalculation" |

#### ★ purge 滞后限流：延迟加在 DML 上，不在 purge 上（参数与观测）

★ 真实链路是 `trx_purge_dml_delay()` → 全局 `srv_dml_needed_delay` → DML 路径 `row_mysql_delay_if_needed()` 睡眠（**`srv_purge_lag_delay` 这个名字不存在**）：

```cpp
static ulint trx_purge_dml_delay(void) {
  if (srv_max_purge_lag > 0 && trx_sys->rseg_history_len.load() >
                                   srv_n_purge_threads * srv_purge_batch_size) {
    ratio = float(trx_sys->rseg_history_len.load()) / srv_max_purge_lag;
    if (ratio > 1.0) delay = (ulint)((ratio - 0.9995) * 10000);   // 微秒
    if (delay > srv_max_purge_lag_delay) delay = srv_max_purge_lag_delay;
  }
  return delay;
}
```

⇒ 只有 `ratio > 1.0`（history **超过** `innodb_max_purge_lag`）才产生延迟，`(ratio - 0.9995) × 10000` 微秒，再被 `innodb_max_purge_lag_delay` 截断。两者默认都是 **0**（不限）。**限流惩罚的是写者，不是 purge**——这是"purge 慢 → DML 变慢"的传导路径。

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
/** This function runs a purge batch.
 @return number of undo log pages handled in the batch */
ulint trx_purge(ulint n_purge_threads, ulint batch_size, bool truncate) {
  que_thr_t *thr = nullptr;
  ulint n_pages_handled;

  srv_dml_needed_delay = trx_purge_dml_delay();   // ★ 每批开头重算一次 DML 延迟

  ut_a(purge_sys->n_submitted == purge_sys->n_completed);

  rw_lock_x_lock(&purge_sys->latch, UT_LOCATION_HERE);

  purge_sys->view_active = false;
  trx_sys->mvcc->clone_oldest_view(&purge_sys->view);
  // ★★ purge 的低水位 = 最老活跃 ReadView
  purge_sys->view_active = true;

  rw_lock_x_unlock(&purge_sys->latch);

  /* Fetch the UNDO recs that need to be purged. */
  n_pages_handled = trx_purge_attach_undo_recs(n_purge_threads, batch_size);

  /* Do we do an asynchronous purge or not ? */
  if (n_purge_threads > 1) {
    /* Submit the tasks to the work queue. */
    for (ulint i = 0; i < n_purge_threads - 1; ++i) {
      thr = que_fork_scheduler_round_robin(purge_sys->query, thr);
      ut_a(thr != nullptr);
      srv_que_task_enqueue_low(thr);       // ★ 前 n-1 个入队 → 唤醒 worker
    }

    thr = que_fork_scheduler_round_robin(purge_sys->query, thr);
    ut_a(thr != nullptr);
    purge_sys->n_submitted += n_purge_threads - 1;

    goto run_synchronously;                // ★★ coordinator 自己也要跑一个 thr

  } else {
    thr = que_fork_scheduler_round_robin(purge_sys->query, nullptr);
    ut_ad(thr);

  run_synchronously:
    ++purge_sys->n_submitted;
    que_run_threads(thr);
    purge_sys->n_completed.fetch_add(1);

    if (n_purge_threads > 1) {
      trx_purge_wait_for_workers_to_complete();   // ★ 等所有 worker 干完
    }
  }

  ut_a(purge_sys->n_submitted == purge_sys->n_completed);

  /* The first page of LOBs are freed at the end of a purge batch because
  multiple purge threads will access the same LOB ... */
  for (thr = UT_LIST_GET_FIRST(purge_sys->query->thrs); thr != nullptr;
       thr = UT_LIST_GET_NEXT(thrs, thr)) {
    purge_node_t *node = static_cast<purge_node_t *>(thr->child);
    node->free_lob_pages();                // ★ LOB 首页统一在批次末尾释放
  }

  if (truncate || srv_upgrade_old_undo_found) {
    trx_purge_truncate();                  // ★ history truncate + undo 表空间 truncate
  }

  return (n_pages_handled);
}
```

#### 取记录并分组（`trx_purge_attach_undo_recs`，trx0purge.cc:2305）

```cpp
static ulint trx_purge_attach_undo_recs(const ulint n_purge_threads,
                                        ulint batch_size) {
  ulint n_pages_handled = 0;

  purge_sys->limit = purge_sys->iter;   // ★ 新批次：limit 追平 iter（上一批已 purge 完）

  ...
  Purge_groups_t purge_groups(n_purge_threads, heap);
  purge_groups.init();

  while (n_pages_handled < batch_size) {
    if (trx_purge_check_limit()) {
      purge_sys->limit = purge_sys->iter;  // ★ 每读一条就把 limit 推进到 iter
    }

    purge_node_t::rec_t rec;
    rec.undo_rec = trx_purge_fetch_next_rec(&rec.modifier_trx_id, &rec.roll_ptr,
                                            &n_pages_handled, heap);

    if (rec.undo_rec == &trx_purge_ignore_rec) {
      continue;      // ★ 整组 undo log 可跳过（无 del_mark / 无 extern / 无 ord change）
    } else if (rec.undo_rec == nullptr) {
      break;         // ★ 到 purge view 边界或没日志了
    }

    purge_groups.add(rec);   // ★ 按 table_id 哈希分组
  }

  purge_groups.distribute_if_needed();
  purge_groups.assign(run_thrs);
  return (n_pages_handled);
}
```

**分组规则**（`Purge_groups_t`，trx0purge.cc:2120）：

```cpp
  void add(purge_node_t::rec_t &rec) {
    const table_id_t id = trx_undo_rec_get_table_id(rec.undo_rec);
    std::size_t grpid;

    GroupBy::iterator lb = m_grpid_umap.find(id);
    if (lb != m_grpid_umap.end()) {
      grpid = lb->second;               // ★★ 同一张表的所有 rec 必然进同一个 group
    } else {
      grpid = find_smallest_group();    // 新表分给当前最短的组
      m_grpid_umap.insert(std::make_pair(id, grpid));
    }
    m_groups[grpid]->push_back(rec);
    m_total_rec++;
  }
```

> ★★ 这就是为什么**同一张表不会被多个 purge 线程并发 purge**——分组键是 `table_id`，一个 group 绑定一个 thr。安全性由此保证，不需要额外的表锁。

⚠️ **均衡是特例不是常态**：

```cpp
void Purge_groups_t::distribute_if_needed() {
  const uint64_t rseg_history_len = trx_sys->rseg_history_len.load();

  /* If the history list length is greater than maximum allowed purge lag,
  then distribute the workload across all purge threads. */
  if (srv_max_purge_lag > 0 && rseg_history_len > srv_max_purge_lag) {
    distribute();    // ★ 只有 purge 已滞后（history > innodb_max_purge_lag）才负载均衡
  }
}
```

#### 边界判断（`trx_purge_fetch_next_rec`，trx0purge.cc:2261）

```cpp
  if (!purge_sys->next_stored) {
    trx_purge_choose_next_log();
    if (!purge_sys->next_stored) return nullptr;
  }

  if (purge_sys->iter.trx_no >= purge_sys->view.low_limit_no()) {
    return nullptr;      // ★★ 核心边界：undo 所属 trx_no 已达最老活跃 ReadView 的 low_limit_no
  }

  *roll_ptr = trx_undo_build_roll_ptr(false, purge_sys->rseg->space_id,
                                      purge_sys->page_no, purge_sys->offset);
  // ★ 注意 is_insert 恒为 false：purge 只处理 update-undo

  *modifier_trx_id = purge_sys->iter.modifier_trx_id;
  return (trx_purge_get_next_rec(n_pages_handled, heap));
```

`trx_purge_get_next_rec()` 内部只挑**需要 purge** 的三类 rec：

```cpp
    type = trx_undo_rec_get_type(rec2);
    if (type == TRX_UNDO_DEL_MARK_REC) break;
    cmpl_info = trx_undo_rec_get_cmpl_info(rec2);
    if (trx_undo_rec_get_extern_storage(rec2)) break;
    if ((type == TRX_UNDO_UPD_EXIST_REC) &&
        !(cmpl_info & UPD_NODE_NO_ORD_CHANGE)) break;
```

#### rseg 遍历：`TrxUndoRsegsIterator::set_next()`（trx0purge.cc:96）

```cpp
const page_size_t TrxUndoRsegsIterator::set_next() {
  mutex_enter(&m_purge_sys->pq_mutex);

  if (m_iter != m_trx_undo_rsegs.end()) {
    /* We are still processing rollback segment from the same transaction ... */
    m_purge_sys->iter.trx_no = (*m_iter)->last_trx_no;
    // ★ 同一事务的多个 rseg（最多 2）顺序处理 —— per-rseg 有序

  } else if (!m_purge_sys->purge_queue->empty()) {
    m_trx_undo_rsegs = NullElement;

    while (!m_purge_sys->purge_queue->empty()) {
      if (m_trx_undo_rsegs.get_trx_no() == UINT64_UNDEFINED) {
        m_trx_undo_rsegs = purge_sys->purge_queue->top();
      } else if (purge_sys->purge_queue->top().get_trx_no() ==
                 m_trx_undo_rsegs.get_trx_no()) {
        m_trx_undo_rsegs.insert(purge_sys->purge_queue->top());
        // ★ 同 trx_no 的多个元素合并成一个 TrxUndoRsegs
      } else {
        break;
      }
      m_purge_sys->purge_queue->pop();
    }
    m_iter = m_trx_undo_rsegs.begin();
  } else {
    /* Queue is empty, reset iterator. */
    ...
    m_purge_sys->rseg = nullptr;
    return (univ_page_size);
  }

  m_purge_sys->rseg = *m_iter++;
  ...
  m_purge_sys->iter.trx_no = m_purge_sys->rseg->last_trx_no;
  m_purge_sys->hdr_offset = m_purge_sys->rseg->last_offset;
  m_purge_sys->hdr_page_no = m_purge_sys->rseg->last_page_no;
  ...
}
```

**跨 rseg 靠 `purge_queue` 聚合**——它不是类，是个 typedef（`trx0types.h:603`）：

```cpp
  /** Compare two TrxUndoRsegs based on trx_no.
  @return true if elem1 > elem2 else false.*/
  bool operator()(const TrxUndoRsegs &lhs, const TrxUndoRsegs &rhs) {
    return (lhs.m_trx_no > rhs.m_trx_no);
    // ★ std::priority_queue 默认大顶堆；比较器取反 → 按 trx_no 的【小顶堆】
  }
  ...
  /** Rollback segments of a transaction, scheduled for purge. */
  Rsegs_array<2> m_rsegs;      // ★ 一个事务最多 2 个 rseg（redo + noredo）
};

typedef std::priority_queue<
    TrxUndoRsegs, std::vector<TrxUndoRsegs, ut::allocator<TrxUndoRsegs>>,
    TrxUndoRsegs>
    purge_pq_t;      // ★ 这就是常被叫 "Purge_queue" 的东西
```

```cpp
// trx0purge.ic:50 —— 不变式：limit <= iter，且 undo_no 只在同 rseg 内可比
static inline bool trx_purge_check_limit(void) {
  return (purge_sys->iter.trx_no > purge_sys->limit.trx_no ||
          (purge_sys->iter.trx_no == purge_sys->limit.trx_no &&
           ((purge_sys->iter.undo_no >= purge_sys->limit.undo_no) ||
            (purge_sys->iter.undo_rseg_space !=
             purge_sys->limit.undo_rseg_space))));
}
```

> ⚠️ **不存在** `Purge_sys` 类——全局对象是 `trx_purge_t *purge_sys`（`trx0purge.cc:76`，结构体在 `trx0purge.h:985`）。
>
> ⚠️ **不存在** `trx_purge_get_low_limit()`——边界就是 `purge_sys->view.low_limit_no()`。
>
> ★ purge_sys 里那两个迭代器的关系是**生产者/消费者模型**：`iter` = 已读到哪，`limit` = 已 purge 完成到哪。不变式 **limit ≤ iter**，`trx_purge_check_limit()` 就是断言它。

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

**取任务**（`row_purge_step`，row0purge.cc:1229）：

```cpp
que_thr_t *row_purge_step(que_thr_t *thr) {
  purge_node_t *node = static_cast<purge_node_t *>(thr->run_node);
  ...
  if (node->recs != nullptr && !node->recs->empty()) {
    purge_node_t::rec_t rec;

    rec = node->recs->front();
    node->recs->pop_front();         // ★ 从本 group 队列头取一条

    node->roll_ptr = rec.roll_ptr;
    node->modifier_trx_id = rec.modifier_trx_id;

    row_purge(node, rec.undo_rec, thr);

    if (node->recs->empty()) {
      row_purge_end(thr);            // ★ 本组清空 → done=true，thr 回 parent
    } else {
      thr->run_node = node;          // ★ 还有 → 继续跑同一个 node
    }
  } else {
    row_purge_end(thr);
  }
  return (thr);
}
```

```cpp
static void row_purge(purge_node_t *node, trx_undo_rec_t *undo_rec,
                      que_thr_t *thr) {
  bool updated_extern;
  ...
  while (row_purge_parse_undo_rec(node, undo_rec, &updated_extern, thd, thr)) {
    bool purged = row_purge_record(node, undo_rec, thr, updated_extern, thd);

    if (purged || srv_shutdown_state.load() >= SRV_SHUTDOWN_PURGE) return;

    /* Retry the purge in a second if failure occurs. */
    std::this_thread::sleep_for(std::chrono::seconds(1));   // ★ 1s 后重试
  }
}
```

**只有两条路**（`row_purge_record_func`，row0purge.cc:1096）——⚠️ **不存在 `row_purge_insert()`**：

```cpp
[[nodiscard]] static bool row_purge_record_func(
    purge_node_t *node, trx_undo_rec_t *undo_rec,
    IF_DEBUG(const que_thr_t *thr, ) bool updated_extern, THD *thd) {
  dict_index_t *clust_index;
  bool purged = true;

  clust_index = node->table->first_index();
  node->index = clust_index->next();
  ut_ad(!trx_undo_roll_ptr_is_insert(node->roll_ptr));
  // ★★ purge 只处理 update-undo；insert-undo 根本不进 history list

  switch (node->rec_type) {
    case TRX_UNDO_DEL_MARK_REC:
      purged = row_purge_del_mark(node);       // ① DELETE：真正把行删掉
      if (!purged) break;
      MONITOR_INC(MONITOR_N_DEL_ROW_PURGE);
      break;
    default:
      if (!updated_extern) {
        break;                                  // ★ 非 del_mark 且没改外部存储 → 不需处理
      }
      [[fallthrough]];
    case TRX_UNDO_UPD_EXIST_REC:
      row_purge_upd_exist_or_extern_func(IF_DEBUG(thr, ) node, undo_rec);
      //                                        ② 改了二级索引有序字段 / 有外存列
      MONITOR_INC(MONITOR_N_UPD_EXIST_EXTERN);
      break;
  }
  ...
}
```

**DELETE 的 purge**（`row_purge_del_mark`，row0purge.cc:653）：

```cpp
[[nodiscard]] static bool row_purge_del_mark(purge_node_t *node) {
  mem_heap_t *heap = mem_heap_create(1024, UT_LOCATION_HERE);

  while (node->index != nullptr) {
    dict_table_skip_corrupt_index(node->index);
    row_purge_skip_uncommitted_virtual_index(node->index);
    if (!node->index) break;

    if (node->index->type != DICT_FTS) {
      if (node->index->is_multi_value()) {
        row_purge_remove_multi_sec_if_poss(node, heap, false);
      } else {
        dtuple_t *entry = row_build_index_entry_low(
            node->row, nullptr, node->index, heap, ROW_BUILD_FOR_PURGE);
        row_purge_remove_sec_if_poss(node, node->index, entry);
        // ★ 用 undo 里的"所有有序列旧值"重建二级索引 entry，再删掉它
      }
      mem_heap_empty(heap);
    }
    node->index = node->index->next();
  }

  mem_heap_free(heap);

  return (row_purge_remove_clust_if_poss(node));
  // ★★ 先删完所有二级索引项，最后才物理删除聚集索引记录
}
```

> ★ 这解释了为什么 `TRX_UNDO_DEL_MARK_REC` 的 undo 里要记**所有出现在任意索引中的列的旧值**（见 `trx_undo_page_report_modify` 后半段）——purge 要靠它们重建二级索引 entry 才能删掉对应的索引项。

**UPDATE 的 purge**（`row_purge_upd_exist_or_extern_func`，row0purge.cc:697）——两种情形**直接跳过二级索引**：

```cpp
  /* 1. 将一个 del_mark 的 rec 更新为非 del_mark（旧索引条目由 DEL_MARK_REC 那条 undo 清理）
   * 2. update 没改变任何二级索引的记录 */
  if (node->rec_type == TRX_UNDO_UPD_DEL_REC ||
      (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE)) {
    goto skip_secondaries;
  }

  while (node->index != nullptr) {
    ...
    if (row_upd_changes_ord_field_binary(node->index, node->update, thr,
                                         nullptr, nullptr, ...)) {
      dtuple_t *entry = row_build_index_entry_low(
          node->row, nullptr, node->index, heap, ROW_BUILD_FOR_PURGE);
      row_purge_remove_sec_if_poss(node, node->index, entry);
      mem_heap_empty(heap);
    }
    node->index = node->index->next();
  }
  // ★ upd_exist 不同于 del_mark：不需要 purge 聚簇索引记录

skip_secondaries:
  /* Free possible externally stored fields */
  for (ulint i = 0; i < upd_get_n_fields(node->update); i++) {
    const upd_field_t *ufield = upd_get_nth_field(node->update, i);
    if (dfield_is_ext(&ufield->new_val)) {
      ...
      lob::purge(&ctx, index, node->modifier_trx_id,
                 trx_undo_rec_get_undo_no(undo_rec), node->rec_type, ufield, node);
      // ★ 真正释放被替换掉的旧 LOB
    }
  }
```

#### History list truncate（`trx_purge_truncate_rseg_history`，trx0purge.cc:530）

```cpp
loop:
  if (hdr_addr.page == FIL_NULL) { rseg->unlatch(); mtr_commit(&mtr); return; }

  undo_page = trx_undo_page_get(page_id_t(rseg->space_id, hdr_addr.page),
                                rseg->page_size, &mtr);
  log_hdr = undo_page + hdr_addr.boffset;
  undo_trx_no = mach_read_from_8(log_hdr + TRX_UNDO_TRX_NO);

  if (undo_trx_no >= limit->trx_no) {
    /* limit space_id should match the rollback segment space id to avoid
    freeing if the page belongs to a different rollback segment for the
    same trx_no. */
    if (undo_trx_no == limit->trx_no &&
        rseg->space_id == limit->undo_rseg_space) {
      trx_undo_truncate_start(rseg, hdr_addr.page, hdr_addr.boffset,
                              limit->undo_no);      // ★ 只能部分截断
    }
    rseg->unlatch(); mtr_commit(&mtr); return;
  }

  prev_hdr_addr = trx_purge_get_log_from_hist(
      flst_get_prev_addr(log_hdr + TRX_UNDO_HISTORY_NODE, &mtr));

  seg_hdr = undo_page + TRX_UNDO_SEG_HDR;

  if ((mach_read_from_2(seg_hdr + TRX_UNDO_STATE) == TRX_UNDO_TO_PURGE) &&
      (mach_read_from_2(log_hdr + TRX_UNDO_NEXT_LOG) == 0)) {
    /* We can free the whole log segment */
    rseg->unlatch(); mtr_commit(&mtr);
    trx_purge_free_segment(rseg, hdr_addr, is_temp);   // ★ 整段释放
  } else {
    /* Remove the log hdr from the rseg history. */
    trx_purge_remove_log_hdr(rseg_hdr, log_hdr, &mtr);
    rseg->unlatch(); mtr_commit(&mtr);
  }

  mtr_start(&mtr); ...
  hdr_addr = prev_hdr_addr;
  goto loop;
```

> ★ 从 history list **尾部**（最老）开始往前删，遇到第一个 `trx_no >= limit` 就停——这就是"purge 指针"的推进方式。

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

#### undo log header 里的 GTID 区（`storage/innobase/include/trx0undo.h:594-618`）

```cpp
/* GTID is generated by replication when binlog and GTID mode is on. We
persist GTID with undo record till it is written to gtid_exeuted table.
GTID information is present when TRX_UNDO_FLAG_GTID set. It follows XID
information */

/** GTID version offset */
constexpr uint32_t TRX_UNDO_LOG_GTID_VERSION = TRX_UNDO_LOG_XA_HDR_SIZE;

/** GTID offset */
constexpr uint32_t TRX_UNDO_LOG_GTID = TRX_UNDO_LOG_XA_HDR_SIZE + 1;

/** Total length of GTID */
constexpr uint32_t TRX_UNDO_LOG_GTID_LEN = 64;

/** Total size with GTID information. */
constexpr uint32_t TRX_UNDO_LOG_GTID_HDR_SIZE =
    TRX_UNDO_LOG_GTID + TRX_UNDO_LOG_GTID_LEN;

/** GTID offset for XA Prepare. */
constexpr uint32_t TRX_UNDO_LOG_GTID_XA = TRX_UNDO_LOG_GTID_HDR_SIZE;

/** Total size with XA GTID information. For external XA transaction we need
to store both prepare and commit GTID. */
constexpr uint32_t TRX_UNDO_LOG_GTID_XA_HDR_SIZE =
    TRX_UNDO_LOG_GTID_HDR_SIZE + TRX_UNDO_LOG_GTID_LEN;
```

/* 【注】`gtid_exeuted` 是上游拼写错误，照抄。两个 GTID 槽位**共用同一个 1 字节 version**
   （`TRX_UNDO_LOG_GTID_VERSION`），所以外部 XA 占用的是 **128 字节体 + 1 字节共享 version**。 */

标志位（`trx0undo.h:543-551`）：

```cpp
/** Transaction UNDO flags in one byte. This is backward compatible as earlier
 we were storing either 1 or 0 for TRX_UNDO_XID_EXISTS. */
constexpr uint32_t TRX_UNDO_FLAGS = 20;
/** true if undo log header includes X/Open XA transaction identification XID */
constexpr uint32_t TRX_UNDO_FLAG_XID = 0x01;
/** true if undo log header includes GTID information from replication */
constexpr uint32_t TRX_UNDO_FLAG_GTID = 0x02;
/** true if undo log header includes GTID information for XA PREPARE */
constexpr uint32_t TRX_UNDO_FLAG_XA_PREPARE_GTID = 0x04;
```

内存态枚举 `Gtid_storage`（`trx0undo.h:341-401`）：

```cpp
struct trx_undo_t {
  /** Undo log may could be allocated to store transaction GTIDs. */
  enum class Gtid_storage {
    /* No storage is allocated for GTID. */
    NONE,
    /* Storage is allocated for commit GTID. */
    COMMIT,
    /* Storage is allocated for both prepare and commit GTID. For external
    XA transaction, we have GTID fr both prepare and commit. */
    PREPARE_AND_COMMIT
  };

  /** Check if space for GTID is allocated in undo.
  @param[in]    is_prepare      if XA prepare GTID
  @return true iff space for GTID is allocated. */
  bool gtid_allocated(bool is_prepare) const;

  /** Get offset and flag for GTID stored in undo.
  @param[in]    is_prepare      if XA prepare GTID
  @return GTID flag and offset in a tuple. */
  std::tuple<int, size_t> gtid_get_details(bool is_prepare) const;
```

/* 【注】`GTID fr both` 也是上游笔误（应为 for）。 */

#### 真正落字节：`trx_undo_gtid_write()`（`storage/innobase/trx/trx0undo.cc:722-758`）

```cpp
void trx_undo_gtid_write(trx_t *trx, trx_ulogf_t *undo_header, trx_undo_t *undo,
                         mtr_t *mtr, bool is_xa_prepare) {
  int gtid_flag;
  size_t gtid_offset;

  std::tie(gtid_flag, gtid_offset) = undo->gtid_get_details(is_xa_prepare);

  if ((undo->flag & gtid_flag) == 0) {
    return;
  }

  /* Reset GTID flag */
  undo->flag &= ~gtid_flag;

  /* We must have allocated for GTID but add a safe check. */
  if (!undo->gtid_allocated(is_xa_prepare)) {
    ut_d(ut_error);
    ut_o(return );
  }

  Gtid_desc gtid_desc;
  auto &gtid_persistor = clone_sys->get_gtid_persistor();

  gtid_persistor.get_gtid_info(trx, gtid_desc);

  if (gtid_desc.m_is_set) {
    /* Persist GTID version */
    mlog_write_ulint(undo_header + TRX_UNDO_LOG_GTID_VERSION,
                     gtid_desc.m_version, MLOG_1BYTE, mtr);
    /* Persist fixed length GTID */
    static_assert(TRX_UNDO_LOG_GTID_LEN == GTID_INFO_SIZE);
    mlog_write_string(undo_header + gtid_offset, &gtid_desc.m_info[0],
                      TRX_UNDO_LOG_GTID_LEN, mtr);
    undo->flag |= gtid_flag;
  }
  mlog_write_ulint(undo_header + TRX_UNDO_FLAGS, undo->flag, MLOG_1BYTE, mtr);
}
```

/* 【注】★ `mlog_write_string` ⇒ GTID **随 redo 持久化**；`static_assert` 把
   undo 侧的 64 字节与 clone 侧的 `GTID_INFO_SIZE` 钉死为同一值。 */

#### flag 与字节是分离的：`trx_undo_gtid_set()` 只置标志（`trx0undo.cc:648-668`）

```cpp
void trx_undo_gtid_set(trx_t *trx, trx_undo_t *undo, bool is_xa_prepare) {
  int gtid_flag;
  /* For XA prepare we store the GTID separately. */
  std::tie(gtid_flag, std::ignore) = undo->gtid_get_details(is_xa_prepare);

  /* Reset GTID flag */
  undo->flag &= ~gtid_flag;

  if (!trx->persists_gtid) {
    return;
  }

  /* Verify that we have allocated for GTID */
  if (!undo->gtid_allocated(is_xa_prepare)) {
    ib::error(ER_IB_CLONE_GTID_PERSIST)
        << "Could not persist GTID as space for GTID is not allocated.";
    ut_d(ut_error);
    ut_o(return );
  }
  undo->flag |= gtid_flag;
}
```

/* 【注】`trx_undo_gtid_set()` 只动 `undo->flag`，**不写字节**；字节必须等
   `trx_undo_gtid_write()` 执行（它还会先把 flag 清掉再按需回置）。 */

`trx_undo_gtid_write()` 全部调用点（只有 3 处）：

| file:line | 场景 | `is_xa_prepare` |
|---|---|---|
| `storage/innobase/trx/trx0purge.cc:419` | commit（`trx_purge_add_update_undo_to_history` 内，紧跟 `TRX_UNDO_TRX_NO` 之后） | `false` → 写 commit 槽 |
| `storage/innobase/trx/trx0undo.cc:1872` | XA ROLLBACK（`trx_undo_set_state_at_prepare`，`rollback=true` ⇒ `!rollback=false`） | `false` → 写 commit 槽 |
| `storage/innobase/trx/trx0undo.cc:1914` | XA prepared-in-TC（`trx_undo_set_prepared_in_tc`） | `true` → 写 prepare 槽 |

#### 空间是怎么预留的（`trx0undo.cc:777-821`）

```cpp
static void trx_undo_header_add_space_for_xid(
    page_t *undo_page, trx_ulogf_t *log_hdr, mtr_t *mtr,
    trx_undo_t::Gtid_storage gtid_storage) {
  trx_upagef_t *page_hdr = undo_page + TRX_UNDO_PAGE_HDR;

  ulint free = mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE);

  /* free is now the end offset of the old style undo log header */
  ut_a(free == (ulint)(log_hdr - undo_page) + TRX_UNDO_LOG_OLD_HDR_SIZE);

  ulint new_limit = TRX_UNDO_LOG_XA_HDR_SIZE;

  switch (gtid_storage) {
    case trx_undo_t::Gtid_storage::COMMIT:
      new_limit = TRX_UNDO_LOG_GTID_HDR_SIZE;
      break;

    case trx_undo_t::Gtid_storage::PREPARE_AND_COMMIT:
      new_limit = TRX_UNDO_LOG_GTID_XA_HDR_SIZE;
      break;

    case trx_undo_t::Gtid_storage::NONE:
      new_limit = TRX_UNDO_LOG_XA_HDR_SIZE;
      break;

    default:
      ut_d(ut_error);
  }

  ulint new_free = free + (new_limit - TRX_UNDO_LOG_OLD_HDR_SIZE);
  ...
  mlog_write_ulint(page_hdr + TRX_UNDO_PAGE_START, new_free, MLOG_2BYTES, mtr);

  mlog_write_ulint(page_hdr + TRX_UNDO_PAGE_FREE, new_free, MLOG_2BYTES, mtr);

  mlog_write_ulint(log_hdr + TRX_UNDO_LOG_START, new_free, MLOG_2BYTES, mtr);
}
```

/* 【注】★ "XID/GTID 是 append 上去的，不是 header 的固定成员"——
   建 header 时基础长度仍是 `TRX_UNDO_LOG_OLD_HDR_SIZE`（`trx0undo.cc:524`），
   之后按 `Gtid_storage` 把 `PAGE_FREE` 往后推。上游注释（trx0undo.h:576-582）还吐槽：
   "The XID wastes about 150 bytes of space in every undo log... quite a large overhead." */

**Clone_persist_gtid 双 buffer**：`m_gtids[2]` 奇偶切换，事务写 active list，后台线程读 flush list。`storage/innobase/include/clone0repl.h:341-350`：

```cpp
  /** Two lists of GTID. One of them is active where running transactions
  add their GTIDs. Other list is used to persist them to table from time
  to time. */
  Gitd_info_list m_gtids[2];

  /** Number of the current GTID list. Increased when list is switched */
  std::atomic<uint64_t> m_active_number;
```

/* 【注】类名/成员名里的 `Gitd` 是**上游拼写错误**（应为 Gtid），逐字引用时照抄。
   取 list 用 `get_list(number)`：`int list_index = (list_number & 1);`（clone0repl.h:322-325）。 */

配套阈值（`clone0repl.h:325-339`）：

```cpp
  /** Time threshold to trigger persisting GTID. Insert GTID once per 1k
  transactions or every 100 millisecond. */
  static constexpr std::chrono::milliseconds s_time_threshold{100};

  /** Threshold for the count for compressing GTID. */
  const static uint32_t s_compression_threshold = 50;

  /** Number of transaction/GTID threshold for writing to disk table. */
  const static int s_gtid_threshold = 1024;

  /** Maximum Number of transaction/GTID to hold. Transaction commits
  must wait beyond this point. Not expected to happen as GTIDs are
  compressed and written together. */
  const static int s_max_gtid_threshold = 1024 * 1024;
```

/* 【注】★ 最后一条注释说明"异步刷表"**不是无条件的**：累计超过 1M 个 GTID 时
   **提交路径会同步等待**（clone0repl.cc:55-61 `wait_flush(false, false, nullptr)`）。 */

**purge 与 GTID 的协同**：purge 边界会被 GTID 持久化进度压低。`clone_oldest_view` 末尾调用 `view->reduce_low_limit(gtid_persistor.get_oldest_trx_no())`，若还有事务的 GTID 未持久化到 `mysql.gtid_executed` 表，它的 trx_no 不能被 purge 越过。**为什么**：运行期落表走内存双 buffer（与 undo 无关），但**崩溃恢复时** GTID 的唯一找回途径是从 undo header 扫出来（见下）——若 purge 把"GTID 尚未落表"的 undo 先删了，那个 GTID 就永久消失（表、binlog、undo 三处都没有），从库 GTID 集合出现空洞 ⇒ 重放/主从不一致。**完整的根因链与后果、水位落盘到 TRX_SYS 页（`TRX_SYS_TRX_NUM_GTID`）的崩溃安全性，见 [`../server/replication/gtid.md`](../server/replication/gtid.md)「purge 的相互牵制」节**。

代码落点在 `MVCC::clone_oldest_view()`（`storage/innobase/read/read0read.cc:720-748`）：

```cpp
void MVCC::clone_oldest_view(ReadView *view) {
  trx_sys_mutex_enter();

  ReadView *oldest_view = get_oldest_view();

  if (oldest_view == nullptr) {
    view->prepare(0);

    trx_sys_mutex_exit();

  } else {
    view->copy_prepare(*oldest_view);

    trx_sys_mutex_exit();

    view->copy_complete();
  }
  /* Update view to block purging transaction till GTID is persisted. */
  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  auto gtid_oldest_trxno = gtid_persistor.get_oldest_trx_no();
  view->reduce_low_limit(gtid_oldest_trxno);
}
```

`reduce_low_limit()`（`storage/innobase/include/read0types.h:219-228`）：

```cpp
  /** Check and reduce low limit number for read view. Used to
  block purge till GTID is persisted on disk table.
  @param[in]    trx_no  transaction number to check with */
  void reduce_low_limit(trx_id_t trx_no) {
    if (trx_no < m_low_limit_no) {
      /* Save low limit number set for Read View for MVCC. */
      ut_d(m_view_low_limit_no = m_low_limit_no);
      m_low_limit_no = trx_no;
    }
  }
```

以及水位的来源与推进（`clone0repl.h:114-135` / `clone0repl.cc:462-477`）：

```cpp
  /** Get oldest transaction number for which GTID is not persisted to table.
  Transactions committed after this point should not be purged.
  @return oldest transaction number. */
  trx_id_t get_oldest_trx_no() {
    trx_id_t ret_no = m_gtid_trx_no.load();
    /* Should never be zero. It can be set to max only before
    GTID persister is active and no GTID is persisted. */
    ut_ad(ret_no > 0 || srv_force_recovery >= SRV_FORCE_NO_UNDO_LOG_SCAN);
    if (ret_no == TRX_ID_MAX) {
      ut_ad(!is_thread_active());
      ut_ad(m_num_gtid_mem.load() == 0);
    } else if (m_num_gtid_mem.load() == 0) {
      /* For all transactions that are committed before this function is called
      have their GTID flushed if flush is not in progress. "flush not in
      progress" is sufficient but not necessary condition here. This is mainly
      for cases when there is no GTID and purge doesn't need to wait. */
      if (!m_flush_in_progress.load()) {
        ret_no = TRX_ID_MAX;
      }
    }
    return (ret_no);
  }
```

```cpp
void Clone_persist_gtid::update_gtid_trx_no(trx_id_t new_gtid_trx_no) {
  auto trx_no = m_gtid_trx_no.load();
  /* Noting to do if number hasn't increased. */
  if (trx_no != TRX_ID_MAX && trx_no >= new_gtid_trx_no) {
    ut_ad(trx_no == new_gtid_trx_no);
    return;
  }
  /* Update in memory variable. */
  m_gtid_trx_no.store(new_gtid_trx_no);

  /* Persist to disk. This would be useful during recovery. */
  trx_sys_persist_gtid_num(new_gtid_trx_no);

  /* Wake up purge thread. */
  srv_purge_wakeup();
}
```

/* 【注】① 水位落盘在 trx sys 页的 `TRX_SYS_TRX_NUM_GTID`（`trx0sys.cc:140-149`，`mlog_write_ull`）；
   ② 水位推进后**主动唤醒 purge**（`srv_purge_wakeup()`）——GTID 落表是 purge 解阻塞的唯一信号；
   ③ `trx_purge_run()` 也会反过来先催一次 flush（`trx0purge.cc:2663-2667`：
   `/* Flush any GTIDs to disk so that purge can proceed immediately. */`）；
   ④ **purge 本身从不读 GTID**（`trx0purge.cc` 里没有任何 `TRX_UNDO_LOG_GTID*` 读取点）——
   它只是被这个水位挡住。 */

⚠️ **真名**：读函数叫 `trx_undo_gtid_read_and_persist()`，**不存在** `trx_undo_read_gtid()`。

补充两点：一是**为什么 GTID 可以写进 undo header**——undo log header 里有专门的 XID 区与 GTID 区（`TRX_UNDO_FLAG_GTID` / `TRX_UNDO_FLAG_XA_PREPARE_GTID` 两个标志位区分普通 GTID 与 XA prepare GTID），写入时若空间不够会由 `trx_undo_header_add_space_for_xid` 把 header 往后扩展。二是**崩溃恢复时如何补回**：启动时扫描 rseg 的 undo slot，对每个非 CACHED 的 update undo 调 `trx_undo_gtid_read_and_persist`，从 undo header 里把 GTID 读出来补刷到 `mysql.gtid_executed`。完整链路与 `Gtid_set` 的区间表示见 [`../server/replication/gtid.md`](../server/replication/gtid.md)。

#### 崩溃恢复怎么把 GTID 从 undo 里捞回来

读函数（`trx0undo.cc:670-720`）：

```cpp
void trx_undo_gtid_read_and_persist(trx_ulogf_t *undo_header) {
  /* Check if undo log has GTID. */
  auto flag = mach_read_ulint(undo_header + TRX_UNDO_FLAGS, MLOG_1BYTE);

  /* Extract and add GTID information of the transaction to the persister. */
  Gtid_desc gtid_desc;

  /* Get GTID persister */
  auto &gtid_persistor = clone_sys->get_gtid_persistor();

  /* Extract and add XA prepare GTID, if there and if and only if the
     transaction is in PREPARED_IN_TC state, otherwise, there is no assurance
     that the transaction will be kept in prepared state. */
  if ((flag & TRX_UNDO_FLAG_XA_PREPARE_GTID) != 0) {
    /* Get GTID format version. */
    gtid_desc.m_version = static_cast<uint32_t>(
        mach_read_from_1(undo_header + TRX_UNDO_LOG_GTID_VERSION));

    /* Get GTID information string. */
    memcpy(&gtid_desc.m_info[0], undo_header + TRX_UNDO_LOG_GTID_XA,
           TRX_UNDO_LOG_GTID_LEN);
    /* Mark GTID valid. */
    gtid_desc.m_is_set = true;

    /* No concurrency is involved during recovery but satisfy
    the interface requirement. */
    trx_sys_serialisation_mutex_enter();
    gtid_persistor.add(gtid_desc);
    trx_sys_serialisation_mutex_exit();
  }

  if ((flag & TRX_UNDO_FLAG_GTID) == 0) {
    return;
  }
  ... /* commit 槽：读 TRX_UNDO_LOG_GTID 偏移，同样 add 进 persister */
}
```

读点 A —— rseg 内存初始化，扫 update_undo_list（`trx0undo.cc:1345-1405`）：

```cpp
  undo->m_gtid_storage = trx_undo_t::Gtid_storage::NONE;

  if ((flag & TRX_UNDO_FLAG_XA_PREPARE_GTID) != 0) {
    /* Prepare GTID implies space is allocated for both
    prepare and commit GTID. */
    undo->m_gtid_storage = trx_undo_t::Gtid_storage::PREPARE_AND_COMMIT;
  } else if ((flag & TRX_UNDO_FLAG_GTID) != 0) {
    /* Space is allocated for only commit GTID. */
    undo->m_gtid_storage = trx_undo_t::Gtid_storage::COMMIT;
  }
  ...
    ut_ad(type == TRX_UNDO_UPDATE);
    if (state != TRX_UNDO_CACHED) {
      UT_LIST_ADD_LAST(rseg->update_undo_list, undo);
      /* For XA prepared transaction and XA rolled back transaction, we
      could have GTID to be persisted. */
      if (undo->is_prepared() || state == TRX_UNDO_ACTIVE) {
        trx_undo_gtid_read_and_persist(undo_header);
      }
```

读点 B —— 沿 history list 回溯补 GTID（`trx0rseg.cc:159-213`，`trx_rseg_persist_gtid`）：

```cpp
  /* The mini-transactions used in this function should not do any
  modification/write operation. We read the undo header and send GTIDs
  to the GTID persistor. There is no impact if the server crashes
  anytime during the operation. */
  ...
  auto rseg_max_trx_no = mach_read_from_8(rseg_header + TRX_RSEG_MAX_TRX_NO);

  /* Check if GTID for transactions in this rollback segment are persisted. */
  if (rseg_max_trx_no < gtid_trx_no) {
    mtr_commit(&mtr);
    return;
  }

  /* Head of transaction history list in rollback segment. */
  auto node = rseg_header + TRX_RSEG_HISTORY;

  fil_addr_t node_addr = flst_get_first(node, &mtr);
  ...
  while (node_addr.page != FIL_NULL) {
    ...
    auto undo_trx_no = mach_read_from_8(undo_log + TRX_UNDO_TRX_NO);

    /* Check and exit if the transaction GTID is already persisted. We
    don't need to check any more as history list is ordered by trx_no. */
    if (undo_trx_no < gtid_trx_no) {
      mtr_commit(&mtr);
      break;
    }
    trx_undo_gtid_read_and_persist(undo_log);

    /* Move to next node. */
    node_addr = flst_get_next_addr(node, &mtr);
    mtr_commit(&mtr);
  }
```

/* 【注】`gtid_trx_no` 起点来自 trx sys 页的 `TRX_SYS_TRX_NUM_GTID`（`trx0rseg.cc:626-652`），
   即"上次已经落表的水位"；history list 按 `trx_no` 有序，所以遇到更小的就可以 `break`。 */

#### 写入时机与字节布局的两个细节

1. **不止 commit 时写**：XA prepared-in-TC 阶段写 prepared 专用槽（`trx0undo.cc:1914`），XA ROLLBACK 写 commit 槽（`trx0undo.cc:1872`）。而普通 XA PREPARE（`trx_undo_set_state_at_prepare`，`rollback=false`）**只把 flag 写进 header**，GTID 字节要等到 prepared-in-TC 那一刻才落盘。
2. **两段 GTID 共用同一个 version 字节**：commit 槽与 prepare 槽各 64 字节，但 `TRX_UNDO_LOG_GTID_VERSION` 只有一份，合计 **128 + 1**；且只有 `Gtid_storage::PREPARE_AND_COMMIT`（外部 XA）才预留第二个 64 字节，`COMMIT` 只预留一段。

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
- 横向（purge 在**二级索引**上的判定接口、删除顺序约束、多值索引分支）见 [`../index/transaction.md`](../index/transaction.md)——本篇讲 purge 的**执行过程**，那篇讲 purge 在索引侧的**判定与后果**
- undo 页与 undo 记录本身受 redo 保护，见 [`redo_log.md`](redo_log.md)
- 行记录中 `DB_ROLL_PTR` 字段的物理格式见 [`physical/record.md`](physical/record.md)；页结构通览见 [`physical/page_structure.md`](physical/page_structure.md)
- fsp / segment / inode 的通用机制见 [`physical/tablespace.md`](physical/tablespace.md)
- purge 删除索引记录的 B-tree 操作见 [`btr.md`](../index/btr.md)
- purge 使用 buffer pool watch 的哨兵机制见 [`buffer_pool.md`](buffer_pool.md)
- DDL 整体框架与 Online/INSTANT 见 [`ddl.md`](../feat/ddl/ddl.md)
- undo header 中 GTID 的持久化路径见 [`../server/replication/gtid.md`](../server/replication/gtid.md)
