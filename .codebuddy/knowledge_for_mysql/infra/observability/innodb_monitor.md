# InnoDB Monitor 与 monitor counter 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 InnoDB Monitor 输出体系（`SHOW ENGINE INNODB STATUS` 全景文本、三个 monitor 类后台线程、临时文件中转机制）、monitor counter 体系（`information_schema.INNODB_METRICS` 的背后：ID 即下标的三位一体结构、位图开关、无锁累加、`MONITOR_EXISTING` 复用），以及与 server 层 status variable / PFS 的分工。
>
> **边界**：本篇讲 **InnoDB 的可观测基础设施**（它是全局的，不专属任何子系统——锁文档引用它，不是它引用锁）。server 层 status variable 体系（`System_status_var`、`SHOW STATUS` → PFS 重写）见 [`../server/infra/variables.md`](../../server/infra/variables.md) G 节；PFS instrument 体系见 [`../server/infra/pfs.md`](pfs/pfs.md)；DBUG 见 [`../server/infra/dbug.md`](../../server/infra/dbug.md)。**`I_S.INNODB_*` 表族**（本篇只覆盖其中的 `INNODB_METRICS` 一张）见 [`../feat/i_s.md`](i_s.md)（跨层篇：server 的 I_S 插件框架 + InnoDB 26 张表的实现，含"三种暴露路径"对比表与选型决策树）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- 核心实现
  - [主链路](#主链路)
  - [Monitor 线程体系](#monitor-线程体系)
  - [monitor counter 体系](#monitor-counter-体系)
  - [与 server 层体系的关系](#与-server-层体系的关系)
  - [与 PFS 的能力重叠与分工](#与-pfs-的能力重叠与分工两套测量模型)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 语境里的 **"monitor" 是 InnoDB 的专有叫法**，有两件东西：

1. **InnoDB Monitor**：一整块**自由文本**的全景诊断输出（`SHOW ENGINE INNODB STATUS` / error log / `innodb_status.<pid>` 文件），由后台线程周期性生成或按需现场生成。
2. **monitor counter**：InnoDB 内部的**计数器体系**（约 300 个），对外暴露为 `information_schema.INNODB_METRICS`，可用 `innodb_monitor_enable/disable/reset` 按名字/模块/通配符开关。

**它不是事务锁专属的**——覆盖 metadata、lock、buffer、buffer_page_io、os、transaction、purge、undo、log、compression、index、adaptive_hash、file_system、change_buffer、dml、ddl 等二十来个子系统。（锁文档 [`../infra/lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md) 只引用其中 lock 相关的部分。）

### 用途

在系统已经出问题（hang、死锁、内存泄漏、I/O 停滞）时，提供**现场取证**能力：Monitor 给"此刻全系统长什么样"的全景，counter 给"各类事件发生了多少次/多久"的量化趋势。设计上的第一优先级是**不能让诊断本身把系统搞挂**（见 `MUTEX_NOWAIT` 降级）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | 只有 `SHOW ENGINE INNODB STATUS` 文本 + server 层 status variable；`srv_monitor_interval` 可调（8.0 已移除，改硬编码 15s） |
| 5.5/5.6 | **monitor counter 体系引入**（`srv0mon.h`，2009 年创建）+ `INNODB_METRICS` 表，把老 status variable 以 `MONITOR_EXISTING` 方式纳入同一套开关/reset 语义 |
| 8.0 | `data_locks`/`data_lock_waits` 走 PFS 虚拟表（server 定义表 + 引擎填充）；`SHOW STATUS` 重写为 PFS 查询；`srv_lock_timeout_thread` 包装函数移除，线程体直接是 `lock_wait_timeout_thread` | **未检索到编号** |

> ★ **WL 检索结论（如实注明）**：`srv/srv0mon.cc`、`srv/srv0srv.cc`、`include/srv0mon.h` 全树 grep `WL#` / `worklog` **零命中**；Worklog 库中亦未检索到与"InnoDB monitor counter / INNODB_METRICS"对应的独立 WL。判断：monitor counter 体系创建于 **2009 年（`srv0mon.h:32` 的 `Created 12/15/2009`）**，早于 MySQL Worklog 的系统化使用，**没有对应编号**——不是漏查。`INNODB_METRICS` 表也未见独立 WL，`i_s.cc:1692` 只有一条"改列要改 `i_s_innodb_plugin_version_postfix`"的约定注释。
>
> ★ 相关但不同主题：8.0 的 `data_locks` PFS 虚拟表属于 PFS 体系（见 [`pfs/pfs.md`](pfs/pfs.md)），其 WL 不在本篇追。

**★★ 版本演进的源码证据（8 条）**

**① 体系创建于 5.5 时代**（`srv0mon.h:32`）：

```cpp
/** @file include/srv0mon.h
 Server monitor counter related defines
 Created 12/15/2009     Jimmy Yang     /* ★ 2009-12-15，5.5 开发期 */
```
（实现文件 `srv0mon.cc:32`：`Created 12/9/2009 Jimmy Yang`）

**② 8.0 新增的模块——counter 体系仍在扩张，不是冻结状态**（`srv0mon.h:508`）：

```cpp
  /* CPU usage information */
  MONITOR_MODULE_CPU,
  MONITOR_CPU_UTIME_ABS, MONITOR_CPU_STIME_ABS,
  MONITOR_CPU_UTIME_PCT, MONITOR_CPU_STIME_PCT, MONITOR_CPU_N,

  MONITOR_MODULE_PAGE_TRACK,        /* ★ 8.0.17+ clone / page tracking */
  MONITOR_PAGE_TRACK_RESETS,
  MONITOR_PAGE_TRACK_PARTIAL_BLOCK_WRITES,
  MONITOR_PAGE_TRACK_FULL_BLOCK_WRITES,
  MONITOR_PAGE_TRACK_CHECKPOINT_PARTIAL_FLUSH_REQUEST,

  MONITOR_MODULE_DBLWR,             /* ★ 8.0.20+ 并行 doublewrite */
  MONITOR_DBLWR_ASYNC_REQUESTS, MONITOR_DBLWR_SYNC_REQUESTS,
  MONITOR_DBLWR_FLUSH_REQUESTS, MONITOR_DBLWR_FLUSH_WAIT_EVENTS,
```
> ★ `MONITOR_MODULE_PAGE_TRACK` 与 `MONITOR_MODULE_DBLWR` **都是 5.7 没有的 8.0 新模块**。共 **22 个 `MONITOR_MODULE_*`**。

**③ 枚举尾部的"哨兵复用"**（`srv0mon.h:536`，★ 匹配结果复用同一整数空间）：

```cpp
  MONITOR_ALL_COUNTER,     /* 用于 turn on/off/reset 全部 */
  NUM_MONITOR              /* This must be the last member */
};

#define MONITOR_WILDCARD_MATCH (NUM_MONITOR + 1)
#define MONITOR_NO_MATCH (NUM_MONITOR + 2)
```
> ★ `0..NUM_MONITOR-1` 有效；`NUM_MONITOR+1` = 通配；`NUM_MONITOR+2` = 未命中。

**④ 8.0 线程重命名的活化石**（`srv0start.cc:184` vs `:2272`）：PFS key 仍叫 `srv_lock_timeout_thread_key`，但创建的函数已是 `lock_wait_timeout_thread`。

**⑤ 9 个 deprecated counter**（见 `MONITOR_EXISTING` 节，值恒 0）。

**⑥ 过时的内部注释**（`srv0mon.cc:94`，★ 文档滞后于重命名 13 年）：

```cpp
/** ...
Please note: If you add a monitor here, please add its corresponding
monitor_id to "enum monitor_id_value" structure in srv0mon.h file. */
```
> ⚠️ `enum monitor_id_value` 这个符号**在 8.0.39 已不存在**（真名 `enum monitor_id_t`，`srv0mon.h:132`）。

**⑦ `INNODB_METRICS` 表结构的版本化约定**（`i_s.cc:1692`，★ 源码里**没有**"引入于 X.Y.Z"的版本标记）：

```cpp
/* Fields of the dynamic table INFORMATION_SCHEMA.innodb_metrics
Every time any column gets changed, added or removed, please remember
to change i_s_innodb_plugin_version_postfix accordingly, so that
the change can be propagated to server */
```

**⑧ `MONITOR_INC` 在 hotbackup 编译下被彻底抹掉**（`srv0mon.h:952`，★ 零成本的极端形式）：

```cpp
#else /* !UNIV_HOTBACKUP */
#define MONITOR_INC(x) ((void)0)     /* ★ 编译期空实现 */
#define MONITOR_DEC(x) ((void)0)
#endif /* !UNIV_HOTBACKUP */
```

---

## 理论基础

### 设计思想与权衡

**① 三套体系并存而非统一**。MySQL 8.0 观测同一个"锁等待"事实有三条路：Monitor 文本、monitor counter、PFS `data_locks` 表。被否决的方案是"统一到一个体系"——代价是历史兼容断裂（5.x 脚本依赖 SHOW ENGINE 文本）与开销不可控（PFS 的 per-event 精度在"只需要一个计数"的场景是浪费）。代价是**三套开关、三种语义、同名指标口径不一**（如 `lock_rec_lock_waits` 计"入队次数"、`Innodb_row_lock_waits` 计"实际挂起次数"，超时场景两者不一致）。

**② 埋点复用业务锁，不为监控引入新锁**（monitor counter 的核心决策）。头注释原话："we do not provide any synchronization for these monitor operations due to performance consideration. Most counters can be placed under existing mutex protections in respective code module." 即把监控正确性寄生在 InnoDB 已有的锁协议上。

**③ 分级精度："计数要准，极值可以不准"**。`fetch_add` 保证计数不丢，max/min 用非原子比较写并留出注释承认竞态——计数用于阈值与趋势（必须可靠），极值是辅助参考（偶发偏小可接受）。

**④ 成本从写路径挪到读路径**。`MONITOR_EXISTING` 计数器（几十个老 status variable）**平时零埋点开销**，只在开关与查询时刻做差分算术；`MONITOR_INC_NOCHECK` 把 N 次开关判定降为 1 次（模块级判断 + 多路免检自增）。

**⑤ 诊断不能变成新的 hang**（Monitor 输出的核心约束）。见下文 `MUTEX_NOWAIT` 与 LOCK_DATA 的"尽力而为"。

**失效场景与退化**：monitor counter 无 per-thread/per-table 维度、无 event 语义（答不了"哪次等待最久"，只能给聚合）；无锁路径会丢计数；`MONITOR_ON/OFF` 非原子（并发 `SET GLOBAL` 与埋点有理论竞态）。

### 理论溯源

- **观测者效应（observer effect）**：监控自身污染被观测对象。server 层 `SHOW STATUS` 的应对是"语句前快照 + 增量转移"（见 variables.md），InnoDB 侧的应对是 `MUTEX_NOWAIT` 与"绝不因打印而阻塞"。
- **data-oriented design**：`monitor_id_t` 枚举值直接当数组下标，热数据（值）与冷数据（名字/描述）彻底分离——与 server 层 `System_status_var` 连续内存块 + `offset` 定位同构。
- **依赖倒置（DIP）**：PFS 锁表是"server 定义表结构、引擎实现 iterator 填充"，引擎不认识 `TABLE`/`Field`。

### 他库对比

| 数据库 | 全景诊断 | 计数器体系 | 关系型暴露 |
|---|---|---|---|
| **MySQL/InnoDB** | `SHOW ENGINE INNODB STATUS`（文本） | monitor counter（`INNODB_METRICS`）+ status var | PFS 虚拟表（可 SQL） |
| **PostgreSQL** | 无等价物（靠 `pg_stat_*` 视图） | `pg_stat_*` 视图 + 计数器 | 全是视图，天然可 SQL |
| **Oracle** | AWR/ASH report（文本 + 表） | `v$sysstat` / `v$sesstat` | 全是视图 |

MySQL 是"文本 + 计数器 + PFS 表"三套并存；PG 从一开始就是视图（可 SQL 但缺"一整块全景快照"）；Oracle 的 AWR 是"定期快照到表"。

---

## 核心实现

### 主链路

```
【Monitor 输出】
srv_monitor_thread（15s 硬编码周期，或 srv_innodb_needs_monitoring 引用计数触发）
  → srv_printf_innodb_monitor(file, nowait, &trx_start, &trx_end)
      ├ BACKGROUND THREAD ← srv_print_master_thread_info
      ├ SEMAPHORES        ← sync_print
      ├ LATEST FOREIGN KEY ERROR（条件）← ut_copy_file(dict_foreign_err_file)
      ├ TRANSACTIONS      ← srv_printf_locks_and_transactions
      │    ├ LATEST DETECTED DEADLOCK（条件）← ut_copy_file(lock_latest_err_file)
      │    └ LIST OF TRANSACTIONS ← lock_print_info_all_transactions
      ├ FILE I/O ← os_aio_print
      ├ INSERT BUFFER AND ADAPTIVE HASH INDEX ← ibuf_print + ha_print_info
      ├ LOG ← log_print
      ├ BUFFER POOL AND MEMORY ← buf_print_io
      └ ROW OPERATIONS
  → 写 stderr（innodb_status_output）或 srv_monitor_file（中转区）

SHOW ENGINE INNODB STATUS 走另一条：
handlerton::show_status → innodb_show_status()
  → nowait=false 现场生成到 srv_monitor_file → 按 trx 起止偏移智能截断(1MB) → 读回发给客户端

【monitor counter】
埋点 MONITOR_INC(id) → monitor_inc_value → MONITOR_IS_ON 位图判断 → relaxed load+add+store（+max）
查询：SELECT ... FROM information_schema.INNODB_METRICS
  → i_s_metrics_fill → 逐计数器读 monitor_value_t（EXISTING 的先做差分）→ 填 17 列
```

### Monitor 线程体系

#### 三个 monitor 类线程

InnoDB 有二十多个后台线程，与观测/死锁直接相关的是这三个（都在 `srv0start` 创建，只读模式不启）：

| PSI 线程名 | 线程体 | 周期 | 职责 |
|---|---|---|---|
| `ib_srv_lock_to` | `lock_wait_timeout_thread` | **1 秒** | 锁超时扫描 + wait-for 图分析 + CATS 权重刷新 + 死锁检测 |
| `ib_srv_err_mon` | `srv_error_monitor_thread` | **1 秒** | LSN 单调性、LRU 统计、**latch 层死锁环检测**、长信号量等待告警与 fatal crash |
| `ib_srv_mon` | `srv_monitor_thread` | **15 秒**（硬编码） | 周期性把 Monitor 全文写到 stderr / status 文件 |

两个易踩的坑：

1. **`srv_lock_timeout_thread` 这个函数不存在**——它只是 PFS key 与线程名（`ib_srv_lock_to`）的历史命名残留，真正的线程体就是 `lock_wait_timeout_thread()`。
2. **monitor 周期不可调**——8.0.39 里 `srv_monitor_interval` 已不存在，周期是 `const auto sleep_interval = std::chrono::seconds{15}`。

**★ 线程创建点**（`srv0start.cc:2263`，★ 只读模式下一个都不起）：

```cpp
  if (!srv_read_only_mode) {
    /* Create the thread which watches the timeouts for lock waits */
    srv_threads.m_lock_wait_timeout = os_thread_create(
        srv_lock_timeout_thread_key, 0, lock_wait_timeout_thread);   /* ★ PFS key 仍是旧名 */
    srv_threads.m_lock_wait_timeout.start();

    /* Create the thread which warns of long semaphore waits */
    srv_threads.m_error_monitor = os_thread_create(srv_error_monitor_thread_key,
                                                   0, srv_error_monitor_thread);
    srv_threads.m_error_monitor.start();

    /* Create the thread which prints InnoDB monitor info */
    srv_threads.m_monitor =
        os_thread_create(srv_monitor_thread_key, 0, srv_monitor_thread);
    srv_threads.m_monitor.start();
  }
```
> ★ master thread 在更晚处单独创建（`srv0start.cc:2506`），且它是**唯一占用 `srv_slot_t` 的**（`srv_reserve_slot(SRV_MASTER)`）。

**★ 线程枚举里根本没有 monitor 类线程**（`srv0srv.h:962`）：

```cpp
enum srv_thread_type {
  SRV_NONE,
  SRV_WORKER,  /* threads serving parallelized queries ... */
  SRV_PURGE,   /* Purge coordinator thread */
  SRV_MASTER   /* the master thread, (whose type number must be biggest) */
};
```
> ★★ 只有 4 个，且只用于 **slot 分配**。monitor / error-monitor / lock-wait-timeout 三个线程**不占 slot、不在该枚举里**，靠 PFS key + `srv_threads` 成员管理。所以"InnoDB 后台线程都在 `srv_thread_type` 里"是错的。

**★ monitor thread 完整实现**（`srv0srv.cc:1781`）：

```cpp
void srv_monitor_thread() {
  uint16_t mutex_skipped{0};
  bool last_srv_print_monitor = srv_print_innodb_monitor;
  ut_ad(!srv_read_only_mode);

  srv_monitor_stats_refreshed_at = std::chrono::steady_clock::now();
  const auto sleep_interval = std::chrono::seconds{15};     /* ★★ 15 秒，硬编码 */
  while (0 != os_event_wait_time(srv_monitor_event, sleep_interval)) {
    if (srv_print_innodb_monitor || 0 < srv_innodb_needs_monitoring.load()) {
      if (!last_srv_print_monitor) { mutex_skipped = 0; last_srv_print_monitor = true; }
      if (!srv_printf_innodb_monitor(stderr, MUTEX_NOWAIT(mutex_skipped),
                                     nullptr, nullptr)) {   /* ① 打到 stderr（error log） */
        mutex_skipped++;
      } else { mutex_skipped = 0; }
    } else { last_srv_print_monitor = false; }

    if (!srv_read_only_mode && srv_innodb_status) {
      mutex_enter(&srv_monitor_file_mutex);
      rewind(srv_monitor_file);                            /* ★ 临时文件复位 */
      if (!srv_printf_innodb_monitor(srv_monitor_file, MUTEX_NOWAIT(mutex_skipped),
                                     nullptr, nullptr)) {  /* ② 打到临时文件 */
        mutex_skipped++;
      } else { mutex_skipped = 0; }
      os_file_set_eof(srv_monitor_file);
      mutex_exit(&srv_monitor_file_mutex);
    }

    if (srv_monitor_stats_refreshed_at + std::chrono::minutes{1} <
        std::chrono::steady_clock::now() + sleep_interval) {
      /* We refresh InnoDB Monitor values so that averages are printed from at
      most 60 last seconds and at least 15 seconds */
      srv_refresh_innodb_monitor_stats();                  /* ③ 每 60 秒刷一次平均值基线 */
    }
  }
}
```
> ★★ 一个线程**同时写两个地方**：`stderr`（进 error log）+ 全局临时文件 `srv_monitor_file`（供 `SHOW ENGINE INNODB STATUS` 回读）。这就是"临时文件中转"机制的源头。

**★ error monitor thread**（`srv0srv.cc:1840`，★ 五件事的循环）：

```cpp
  new_lsn = log_get_lsn(*log_sys);
  if (new_lsn < old_lsn) {                            /* ① LSN 回退检测 */
    ib::error(ER_IB_MSG_1046, ulonglong{old_lsn}, ulonglong{new_lsn});
    ut_d(ut_error);
  }
  old_lsn = new_lsn;

  /* Update the statistics collected for deciding LRU eviction policy.
  NOTE: While this doesn't relate to error monitoring, it's here for historical
  reasons, as it depends on being called ~1Hz. */
  buf_LRU_stat_update();                              /* ② 借 1Hz 心跳顺手更新 LRU 统计 */

  sync_arr_wake_threads_if_sema_free();
  sync_array_detect_deadlock();                       /* ③ latch 层死锁环检测 */

  if (sync_array_print_long_waits(&waiter, &sema) && sema == old_sema &&
      waiter == old_waiter) {
    fatal_cnt++;
    if (fatal_cnt > 10) {                             /* ④ 连续 10 次同一 sema → crash */
      ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_1047,
                ulonglong{srv_fatal_semaphore_wait_threshold});
    }
  } else { fatal_cnt = 0; old_waiter = waiter; old_sema = sema; }

  fflush(stderr);
  sig_count = os_event_reset(srv_error_event);
  os_event_wait_time_low(srv_error_event, std::chrono::seconds{1}, sig_count);  /* ⑤ 固定 1 秒 */
```
> ★ ② 那条注释很有意思：**LRU 统计挂在 error monitor 上纯属历史原因**（因为它需要 ~1Hz 的调用频率）。

**★ lock wait timeout thread**（`lock0wait.cc:1435`，★ 注意它在 `lock0wait.cc` 不是 `lock0lock.cc`）：

```cpp
void lock_wait_timeout_thread() {
  os_event_t event = lock_sys->timeout_event;   /* ★ 等待的是 lock_sys->timeout_event */
  auto last_checked_for_timeouts_at = std::chrono::steady_clock::now();
  do {
    auto current_time = std::chrono::steady_clock::now();
    if (std::chrono::seconds(1) <= current_time - last_checked_for_timeouts_at) {
      last_checked_for_timeouts_at = current_time;
      lock_wait_check_slots_for_timeouts();            /* ① 每秒扫 slot 判锁等待超时 */
    }
    lock_wait_update_schedule_and_check_for_deadlocks();
    /* ② wait-for 图 + 死锁检测 + schedule weight —— 8.0 CATS 调度的大脑 */
    os_event_wait_time_low(event, std::chrono::seconds{1}, sig_count);
    sig_count = os_event_reset(event);
  } while (srv_shutdown_state.load() < SRV_SHUTDOWN_CLEANUP);
}
```
> ★ 它对 monitor counter 只有一个直接贡献：`MONITOR_INC(MONITOR_SCHEDULE_REFRESHES)`（`lock0wait.cc:1376`）。

**★ master thread**（`srv0srv.cc:2748`，★ 没被删但主循环已被掏空）：

```cpp
static void srv_master_sleep(void) {
  srv_main_thread_op_info = "sleeping";
  std::this_thread::sleep_for(std::chrono::seconds(1));   /* ★ 硬编码 1 秒 */
  srv_main_thread_op_info = "";
}
```
```cpp
  while (srv_shutdown_state.load() < SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS) {
    srv_master_sleep();
    MONITOR_INC(MONITOR_MASTER_THREAD_SLEEP);   /* ★ master thread 自己也是 counter 源 */
    srv_main_thread_op_info = "checking free log space";
    log_free_check();
    if (srv_check_activity(old_activity_count)) {
      old_activity_count = srv_get_activity_count();
      srv_master_do_active_tasks();
    } else {
      srv_master_do_idle_tasks();               /* ++srv_main_idle_loops; MONITOR_INC(MONITOR_MASTER_IDLE_LOOPS) */
    }
    fil_purge();
  }
```
> ★ 刷脏 / checkpoint 已全部挪走（page cleaner / log writer），master thread 只剩"1 秒心跳 + 空闲/活跃任务 + purge 已删表空间页"。

`srv_error_monitor_thread` 除了周期巡检还做两件关键事——它是"系统已经 hang 住时最后的诊断手段"：

- **`sync_array_detect_deadlock()`**：**latch/mutex 层**的死锁环检测（DFS + 奇偶三色标记），与事务行锁死锁是两回事——latch 死锁无法自愈，检测到直接 `ib::fatal` 自杀。
- **长信号量等待的三级响应**：4 分钟（硬编码 `constexpr std::chrono::minutes timeout{4}`）→ warning + 打印 cell；超过 `srv_fatal_semaphore_wait_threshold`（**600 秒**，生产恒为 600，仅 debug 构建可调）→ 计 fatal；**同一个 waiter 等同一个 sema 连续观测到 11 次**才真正崩溃（约再等 10 秒，给慢 I/O 留一线生机）。触发时会 `srv_innodb_needs_monitoring++` **强开 30 秒 Monitor 输出**并 `lock_set_timeout_event()` 踢一脚死锁线程。

#### `MUTEX_NOWAIT`：诊断不能变成新的 hang

```cpp
/** Maximum number of times allowed to conditionally acquire
mutex before switching to blocking wait on the mutex */
#define MAX_MUTEX_NOWAIT 20
#define MUTEX_NOWAIT(mutex_skipped) ((mutex_skipped) < MAX_MUTEX_NOWAIT)
```

Monitor 默认以 **try-latch** 方式抓 `lock_sys` 全局 latch，抓不到就跳过锁段打印；**连续失败 20 次后才允许阻塞等待**。原因写在注释里：Monitor 常常是在系统已卡住时触发的（如 `sync_array_print_long_waits`），如果 Monitor 自己阻塞在全局 latch 上，诊断信息就永远打不出来。

#### 输出组织：单体拼装 + 临时文件中转

`srv_printf_innodb_monitor()` **一个函数硬编码了全部段落顺序**（没有注册表，加一个段就得改这个函数，见主链路图）。

**"中转临时文件"是这套设计的三个实例**：`srv_monitor_file`（Monitor 全文）、`lock_latest_err_file`（死锁报告）、`dict_foreign_err_file`（外键错误）都用 `os_file_create_tmpfile()` + `rewind()` + `ut_copy_file()` 这套组合。为什么用 `FILE*` 而不是内存 buffer？因为输出长度**高度不可预测**（事务数 × 每事务锁数 × 每锁记录内容），而 `FILE*` 天然提供可增长缓冲与 `ftell/fseek` 随机访问——SHOW 命令靠 `ftell` 记录事务列表起止偏移，超长时才能"从中间挖掉事务列表、保留更有价值的尾部"。

`srv_monitor_file` 有两种形态（启动时二选一）：默认 `innodb_status_file=OFF` → `os_file_create_tmpfile()` 匿名临时文件（纯粹当 SHOW 的中转缓冲区）；`=ON` → datadir 下具名的 `innodb_status.<pid>`，既当中转区又被周期线程覆写，供外部持续观察。

#### SHOW ENGINE INNODB STATUS 的读取路径

`handlerton::show_status` → `innodb_show_status()`：现写临时文件 → 按 `trx_list_start/end` 智能截断（上限 `MAX_STATUS_SIZE = 1048576`）→ 读回内存发客户端。三处细节：

- **SHOW 用 `nowait=false`**（阻塞路径）：用户显式查询时宁可等全局 latch，也要保证锁段完整——与周期线程的降级相反。
- **SHOW 不需要 `innodb_status_output=ON`**：它绕过那个开关（周期输出才受它控制）。
- 截断策略：若事务列表不太长，保留输出尾部（ROW OPERATIONS / BUFFER POOL 等总览更有价值），从中间挖掉事务列表开头并插入 `"... truncated...\n"`。

#### 触发方式

1. **周期**：15s，条件是 `srv_print_innodb_monitor`（`innodb_status_output=ON`）或 `srv_innodb_needs_monitoring > 0`。
2. **显式 SQL**：`SHOW ENGINE INNODB STATUS`。
3. **InnoDB 内部自举**：`srv_innodb_needs_monitoring` 是引用计数——长信号量等待（强开 30s）、buffer pool 被锁堆/AHI 占用超 67%、找不到空闲 block（`n_iterations > 20`）三处 `++`。
4. **`lock_set_timeout_event()`**：不直接触发输出，而是唤醒 `lock_wait_timeout_thread` 立刻做一轮 wait-for 分析（在长等待场景与 `++` 配对出现，效果上是"让下一轮输出的锁段是最新的"）。
5. **不存在** KILL 触发（`srv_printf_innodb_monitor` 全仓库只有 3 个调用点）；错误场景也不 dump monitor，信号量 fatal 与 latch 环死锁都是直接 abort。

### monitor counter 体系

#### 结构：ID 即下标 + 位图开关

```cpp
/** Two monitor structures are defined in this file. One is
"monitor_value_t" which contains dynamic counter values for each
counter. The other is "monitor_info_t", which contains static information...
In addition, an enum datatype "monitor_id_t" is also defined, it identifies
each monitor with an internally used symbol, whose integer value indexes
into above two structure */
```

三位一体靠一个整数下标绑定：`monitor_id_t` = `innodb_counter_value[id]`（热的动态值）= `innodb_counter_info[id]`（冷的静态元信息：名字/子系统/描述/类型标志）。两个数组必须同步维护，靠编译期 `static_assert(UT_ARR_SIZE(innodb_counter_info) == NUM_MONITOR)` 与运行期 `ut_a(count == monitor_info->monitor_id)` 双重守卫。`NUM_MONITOR` 是枚举最后一个成员，自动成为数组维度。

`monitor_value_t` 里只有 `mon_value` 是 `std::atomic`，其余（max/min/基线/时间戳）都是普通 `int64_t`：

| 字段 | 支撑的 I_S 列 |
|---|---|
| `mon_value`（本轮增量） | `COUNT_RESET` |
| `mon_value + mon_value_reset`（+历史基线） | `COUNT` |
| `mon_max/min_value`（本轮极值） | `MAX/MIN_COUNT_RESET` |
| `mon_max/min_value_start`（惰性推进） | `MAX/MIN_COUNT` |
| `mon_start_time` / `mon_stop_time` / `mon_reset_time` | `TIME_ENABLED` / `TIME_DISABLED` / `TIME_RESET` |

未初始化的极值是哨兵 `MIN_RESERVED = INT64_MAX` / `MAX_RESERVED = ~INT64_MAX`（名字指"用在哪个字段上"而非大小），I_S 里填 **NULL 而不是 INT64_MIN/MAX**。

**★ 两个结构的完整定义**（`srv0mon.h:70` / `:543`）：

```cpp
/** Structure containing the actual values of a monitor counter. */
struct monitor_value_t {
  std::chrono::system_clock::time_point mon_start_time; /* Start time of monitoring */
  std::chrono::system_clock::time_point mon_stop_time;  /* Stop time */
  std::chrono::system_clock::time_point mon_reset_time; /* Time counter was reset */
  std::atomic<mon_type_t> mon_value;        /* ★★ 唯一 atomic 的成员 */
  mon_type_t mon_max_value;
  mon_type_t mon_min_value;
  mon_type_t mon_value_reset;               /* value at last reset */
  mon_type_t mon_max_value_start;           /* Max value since start */
  mon_type_t mon_min_value_start;
  mon_type_t mon_start_value;               /* Value at the start time */
  mon_type_t mon_last_value;                /* Last set of values */
  monitor_running_t mon_status;
};
```
```cpp
struct monitor_info_t {
  const char *monitor_name;        /* Monitor name */
  const char *monitor_module;      /* Sub Module the monitor belongs to */
  const char *monitor_desc;        /* Brief desc of monitor counter */
  monitor_type_t monitor_type;     /* Type of Monitor Info（位掩码） */
  monitor_id_t monitor_related_id; /* Monitor ID of counter that related to this monitor */
  monitor_id_t monitor_id;         /* ★★ Monitor ID as defined in enum monitor_id_t */
};
```
> ★★ 第 6 个字段 `monitor_id` 是**"ID 即下标"的自校验冗余**：数组下标 `count` 必须等于 `monitor_info->monitor_id`（`i_s.cc:1831` 用 `ut_a` 逐个断言）。

**★ 位图开关的三个宏**（`srv0mon.h:583`，⚠️ 真名不是 `MONITOR_SET_ON`）：

```cpp
#define NUM_BITS_ULINT (sizeof(ulint) * CHAR_BIT)
extern ulint monitor_set_tbl[(NUM_MONITOR + NUM_BITS_ULINT - 1) / NUM_BITS_ULINT];

#define MONITOR_ON(monitor)                     \
  (monitor_set_tbl[monitor / NUM_BITS_ULINT] |= \
   ((ulint)1 << (monitor % NUM_BITS_ULINT)))
#define MONITOR_OFF(monitor)                    \
  (monitor_set_tbl[monitor / NUM_BITS_ULINT] &= \
   ~((ulint)1 << (monitor % NUM_BITS_ULINT)))
/** Check whether the requested monitor is turned on/off */
#define MONITOR_IS_ON(monitor)                 \
  (monitor_set_tbl[monitor / NUM_BITS_ULINT] & \
   ((ulint)1 << (monitor % NUM_BITS_ULINT)))
```

**★ 编译期守卫只有一处**（`srv0mon.cc:1468`）：

```cpp
  static_assert(UT_ARR_SIZE(innodb_counter_info) == NUM_MONITOR);
```
> ★ 静态数组元素个数必须 == 枚举尾哨兵 `NUM_MONITOR`。**枚举与数组漏同步 → 编译直接失败**。全文件仅此一处 `static_assert`。

**★ 类型标志位**（`srv0mon.h:89`，★ 8.0 仍到 256，没有新增位）：

```cpp
enum monitor_type_t {
  MONITOR_NONE = 0,
  MONITOR_MODULE = 1,        /* 这是模块标记行，不是计数器 */
  MONITOR_EXISTING = 2,      /* 数据来自已有 status variable */
  MONITOR_NO_AVERAGE = 4,    /* 不算均值 */
  MONITOR_DISPLAY_CURRENT = 8, /* 显示当前值而非增量（存量型：页数、bytes） */
  MONITOR_GROUP_MODULE = 16, /* 只能按模块开关，不能单开 */
  MONITOR_DEFAULT_ON = 32,   /* 启动时默认开 */
  MONITOR_SET_OWNER = 64,
  MONITOR_SET_MEMBER = 128,
  MONITOR_HIDDEN = 256       /* 不在 metrics 表里显示 */
};
```

开关是**位图** `monitor_set_tbl`（每计数器 1 bit，全部落在 1~2 个 cache line）：`MONITOR_IS_ON` 成本 = 一次数组索引 + AND。四个系统变量 `innodb_monitor_enable/disable/reset/reset_all` 支持三种写法：

| 写法 | 路径 | 语义 |
|---|---|---|
| `'lock_deadlocks'` | 精确匹配（大小写不敏感） | 单开一项 |
| `'module_lock'` | 命中模块标记行 → `srv_mon_set_module_control` | 展开该模块区间所有成员 |
| `'lock%'` | 含 `%` → 通配符逐个单开 | 注意：已开启项会报错中断（与模块路径"跳过并打 info"行为不同） |
| `'all'` | `MONITOR_ALL_COUNTER` | 全表 |

**默认开启规则**：绝大多数 `MONITOR_EXISTING` 计数器默认开；纯 InnoDB 内部计数器默认关（`srv_mon_default_on()` 按标志位逐个置位，不展开模块——故"默认集合"与"模块集合"是正交的两套划分）。

#### 计数与同步：无锁累加

```cpp
inline void monitor_inc_value_nocheck(monitor_id_t monitor, mon_type_t value,
                                      bool set_max = true) {
  /* We use std::memory_order_relaxed load() and store() as two separate steps,
  instead of single atomic fetch_add operation, because we want to leave it
  non-atomic as it was before changing mon_value to std::atomic.*/
  const auto new_value = MONITOR_VALUE(monitor).load(std::memory_order_relaxed) + value;
  MONITOR_VALUE(monitor).store(new_value, std::memory_order_relaxed);
  if (set_max) monitor_set_max_value(monitor, new_value);
}
inline void monitor_inc_value(monitor_id_t monitor, mon_type_t value) {
  MONITOR_CHECK_DEFINED(value);
  if (MONITOR_IS_ON(monitor)) monitor_inc_value_nocheck(monitor, value);
}
```

- `std::atomic` 在这里**不是为了原子 RMW，只为消除 C++ 数据竞争的 UB**——注释明说"故意用 relaxed load+store 两步而不是 fetch_add，以保持它原来的非原子语义"。所以无锁路径仍会丢计数（与改 atomic 之前一致，被明确接受）。
- 真正需要原子累加时用 `MONITOR_ATOMIC_INC`（`fetch_add`，x86 上一条 `lock xadd`）。注释把规则写死：**已有互斥保护 → `MONITOR_INC`；没有 → `MONITOR_ATOMIC_INC`**。
- max/min 是非原子"读-比较-写"，注释承认 `This is not 100% accurate because of the inherent race, we ignore it due to performance`。
- `MONITOR_INC_NOCHECK` 用于"模块级一次判断 + 多路免检自增"（全仓唯一用例是 buffer page I/O 的 39 选 1 计数器）。

#### `MONITOR_EXISTING`：把老 status variable 搭进同一张表

一批计数器（枚举名前缀 `MONITOR_OVLD_*`，OVLD = overloaded）**没有埋点**，只在开关/查询时刻读老的全局量做差分：

```cpp
/* 当前值 = 全局量 - reset基线 - 开启时快照 + 上次关闭时保存值 */
#define MONITOR_SET_DIFF(monitor, value)                                       \
  MONITOR_SET_UPD_MAX_ONLY(monitor, ((value)-MONITOR_VALUE_RESET(monitor) -    \
                                     MONITOR_FIELD(monitor, mon_start_value) + \
                                     MONITOR_FIELD(monitor, mon_last_value)))
```

例如 `Innodb_row_lock_waits` 与 `INNODB_METRICS` 的 `lock_row_lock_waits` 读的是**同一个** `srv_stats.n_lock_wait_count`——是同一份数据的两种暴露，不是两套实现。开销从写路径挪到了读路径。

**★★ 桥函数 `srv_mon_process_existing_counter()`**（`srv0mon.cc:1597`，★ 源码注释自陈这是"模拟"）：

```cpp
/** This function consolidates some existing server counters used
 by "system status variables". These existing system variables do not have
 mechanism to start/stop and reset the counters, so we simulate these
 controls by remembering the corresponding counter values when the
 corresponding monitors are turned on/off/reset, and do appropriate
 mathematics to deduct the actual value. */
void srv_mon_process_existing_counter(monitor_id_t monitor_id,
                                      mon_option_t set_option) {
  ...
  switch (monitor_id) {
    case MONITOR_OVLD_BUF_POOL_READS:
      value = srv_stats.buf_pool_reads;              /* ★ 源数据在 srv_stats，不在 innodb_counter_value */
      break;
    case MONITOR_OVLD_BUF_POOL_READ_REQUESTS:
      buf_get_total_stat(&stat);
      value = Counter::total(stat.m_n_page_gets);    /* ★ 有的还要现算 */
      break;
    ...
  }
```

四种语义（`srv0mon.cc:1996`）：

```cpp
    case MONITOR_TURN_ON:
      MONITOR_SAVE_START(monitor_id, value);         /* 记开启时基线 */
      return;
    case MONITOR_TURN_OFF:
      if (MONITOR_IS_ON(monitor_id)) {
        srv_mon_process_existing_counter(monitor_id, MONITOR_GET_VALUE);
        MONITOR_SAVE_LAST(monitor_id);               /* 记关闭时值 */
      }
      return;
    case MONITOR_GET_VALUE:
      if (MONITOR_IS_ON(monitor_id)) {
        if (monitor_info->monitor_type & MONITOR_DISPLAY_CURRENT) {
          MONITOR_SET(monitor_id, value);            /* ★ 存量型（页数/bytes）：直接覆盖 */
        } else {
          MONITOR_SET_DIFF(monitor_id, value);       /* ★ 累加型（reads/writes）：算差值 */
        }
      }
      return;
    case MONITOR_RESET_VALUE:
      if (!MONITOR_IS_ON(monitor_id)) MONITOR_LAST_VALUE(monitor_id) = 0;
      return;
```

> ★★★ 关键澄清：**`MONITOR_EXISTING` 的 counter 与老 status variable 不共享存储**。老 status var 存在 `export_vars` / `srv_stats`；monitor counter 存在 `innodb_counter_value[]`。两者靠上面这套"记账"在 **turn on / turn off / reset / get 时刻**从源头取值做差值来**模拟**共享——所以 `MONITOR_EXISTING` 的 counter 有 start/stop/reset 语义，而它背后的老 status variable 本身没有。

**★ 静态表里的写法范式**（`srv0mon.cc:262`）：

```cpp
    {"buffer_pool_pages_total", "buffer",
     "Total buffer pool size in pages (innodb_buffer_pool_pages_total)",
     static_cast<monitor_type_t>(MONITOR_EXISTING | MONITOR_DISPLAY_CURRENT |
                                 MONITOR_DEFAULT_ON),
     MONITOR_DEFAULT_START, MONITOR_OVLD_BUF_POOL_PAGE_TOTAL},
```
> ★ 六元组：`{name, module, desc（括号里带老变量名）, type 位或, related_id, id}`。带 `DISPLAY_CURRENT` → 存量语义（页数），不算差值。

**★ 惰性取值**：只有查 `INNODB_METRICS` 时才去源变量算差值（`i_s.cc:1838`）：

```cpp
    if (monitor_info->monitor_type & MONITOR_EXISTING && MONITOR_IS_ON(count)) {
      srv_mon_process_existing_counter((monitor_id_t)count, MONITOR_GET_VALUE);
    }
```

**★ 9 个已废弃但保留的计数器**（`srv0mon.cc:1237`，★ 值恒 0）：

```cpp
    {"innodb_rwlock_s_spin_waits", "server", "Deprecated counter, always 0",
     static_cast<monitor_type_t>(MONITOR_EXISTING | MONITOR_DEFAULT_ON),
     MONITOR_DEFAULT_START, MONITOR_OVLD_RWLOCK_S_SPIN_WAITS},
```
```cpp
/* srv0mon.cc:1800 —— 运行期实现 */
    case MONITOR_OVLD_RWLOCK_S_SPIN_WAITS:
    ...
      /* Deprecated counter. remained for compatibility. */
      value = 0;
      break;
```
> ★ 名字保留只为兼容，**值永远 0**。查 `INNODB_METRICS` 看到 rwlock spin 系列全是 0 就是这个原因。

#### INNODB_METRICS 表

server 侧 `i_s.cc` 定义 17 列，`i_s_metrics_fill()` 逐计数器填充（需 `PROCESS` 权限，不满足时静默返回空结果）。要点：

- 行数 = `NUM_MONITOR` − 模块标记行 − 隐藏行（`latch` 计数器是 `MONITOR_HIDDEN`，因为它会拉起重量级的 `MutexMonitor` 采集）。
- `COUNT` 与 `COUNT_RESET` 的双字段设计让一次 `innodb_monitor_reset` 既给出"最近窗口增量"又不丢"自启动总量"。
- `AVG_COUNT` 三种口径：SET_OWNER 用 `总量/次数计数器`；普通累计型用 `COUNT/TIME_ELAPSED`（每秒速率）；`DISPLAY_CURRENT` 或 `NO_AVERAGE` 为 NULL。
- `TYPE` 列：`value`（DISPLAY_CURRENT）/ `status_counter`（EXISTING）/ `set_owner` / `set_member` / `counter`。

### 与 server 层体系的关系

**server 层没有叫 "monitor" 的东西**，对等物是三套已在别处详写的机制：

| 层 | 机制 | 归属文档 |
|---|---|---|
| server | 状态变量（`System_status_var` 连续计数器块、`add_to_status` 线性聚合、`SHOW STATUS` 重写为 PFS 查询） | [`../server/infra/variables.md`](../../server/infra/variables.md) G 节 |
| server | PFS instrument / consumer 体系 | [`../server/infra/pfs.md`](pfs/pfs.md) |
| InnoDB | **Monitor 输出 + monitor counter**（本篇） | 本篇 |
| 跨层 | PFS 锁表（`data_locks`）：server 定义 `Plugin_table` + 4 个 HASH 索引，InnoDB 注册 inspector 填充，WHERE 通过 `accept_*()` 下推 | 引擎侧实现见 [`../infra/lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md) |

**InnoDB 怎么接进 server 的 status 体系**：`innodb_export_status()`（`srv_export_innodb_status()` → `export_vars` 全局结构）在每次 `SHOW STATUS` 时被调用，把 `srv_stats.*` 搬进 `export_vars`，后者通过 `SHOW_VAR` 注册进 server 的 `all_status_vars` 数组。这就是"InnoDB 状态变量"的来源——**值在 InnoDB，注册与展示在 server**。

**★ 零拷贝的注册方式**（`ha_innodb.cc:1126`，★ 每个 `SHOW_VAR` 条目直接指向 `export_vars` 成员的裸指针）：

```cpp
static SHOW_VAR innodb_status_variables[] = {
    {"buffer_pool_pages_data",
     (char *)&export_vars.innodb_buffer_pool_pages_data, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},
    {"buffer_pool_bytes_data",
     (char *)&export_vars.innodb_buffer_pool_bytes_data, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},
    {"buffer_pool_pages_dirty",
     (char *)&export_vars.innodb_buffer_pool_pages_dirty, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},
    ...
};
```
> ★ `SHOW STATUS` 读的就是 `export_vars` 本身，无拷贝。

```cpp
/* ha_innodb.cc:21526 —— 每次 SHOW STATUS 前"现算"一次 */
static int show_innodb_vars(THD *, SHOW_VAR *var, char *) {
  innodb_export_status();
  var->type = SHOW_ARRAY;
  var->value = (char *)&innodb_status_variables;
  var->scope = SHOW_SCOPE_GLOBAL;
  return (0);
}
```
> ⚠️ 真名：结构 `struct export_var_t`（`srv0srv.h:1126`）、全局实例 `export_vars`（`srv0srv.h:782`）、填充函数 `srv_export_innodb_status()`（`srv0srv.cc:1577`）。**不存在 `innodb_export_vars_at_start`**。

**★ `export_vars` 的填充入口**（`srv0srv.cc:1576`，★ 持 `srv_innodb_monitor_mutex`）：

```cpp
void srv_export_innodb_status(void) {
  buf_get_total_stat(&stat);
  buf_get_total_list_len(&LRU_len, &free_len, &flush_list_len);
  buf_get_total_list_size_in_bytes(&buf_pools_list_size);
  mutex_enter(&srv_innodb_monitor_mutex);      /* ★ 与 Monitor 输出共用同一把锁 */
  export_vars.innodb_data_pending_reads = os_n_pending_reads;
  export_vars.innodb_data_pending_writes = os_n_pending_writes;
  export_vars.innodb_data_pending_fsyncs =
      log_pending_flushes() + fil_n_pending_tablespace_flushes.load();
  ...
  mutex_exit(&srv_innodb_monitor_mutex);
}
```
> ★ 共用 `srv_innodb_monitor_mutex` 意味着：**`SHOW ENGINE INNODB STATUS` 与 `SHOW STATUS` 会互相串行化**。

**一个必须纠正的流传说法**：`lock_report_wait_for_edge_to_server()` 名字里有 "to_server"，但它**不是上报给 PFS**——它落到 `thd_report_lock_wait()`，只在 `is_mts_worker()` 双方都为真时触发 `Commit_order_manager::check_and_report_deadlock()`（复制 MTS 的跨子系统死锁检测）。非复制场景是完全的空操作。同理 `thd_wait_begin(THD_WAIT_ROW_LOCK)` 是 **thread pool 插件回调**，不是 PFS instrument——8.0 的 PFS 里没有 InnoDB 行锁等待 instrument（`wait/lock/*` 只有 `wait/lock/metadata/sql/mdl` 是真 instrument）。

---

### 与 PFS 的能力重叠与分工（两套测量模型）

能力重叠是真实的，但重叠的是**观测对象**，不是**测量模型**：

| | InnoDB monitor counter | Performance Schema |
|---|---|---|
| **模型** | **计数器模型**：`MONITOR_INC(x)` 单点无状态 | **事件模型**：`begin/end` 成对 + 计时 + 多维聚合 |
| 归属 | InnoDB 私有（`srv0mon.*`） | server 层通用框架（`storage/perfschema`），跨引擎 |
| 维度 | **无维度**：一个 int64（+ min/max/reset 附属值） | thread / account / user / host / object instance / global |
| 计时 | 基本没有 | 每个 wait 都有 `timer_start/timer_end` |
| 埋点位置 | **能贴到算法内部分支** | 只能在 API 边界 |
| 默认 | 部分 `MONITOR_DEFAULT_ON` | mutex/rwlock/cond **默认关**，file **默认开** |

**决策点：为什么 monitor 能覆盖 PFS 结构性做不到的一批信息**（因为它们埋在算法判定分支里，而 PFS 只能在 API 边界插桩）：

- `lock_deadlock_false_positives`——"启发式判定这是个假环"的 else 分支计数，PFS 没有"死锁检测假阳性"这个事件。
- `lock_schedule_refreshes`——CATS 调度权重重算次数，InnoDB 内部概念。
- `buffer_page_io` 的 **37 种页类型**分类（17 读 + 17 写 + 3 on-log）——PFS 只有 `wait/io/file/innodb/innodb_data_file` 一个数，分不清读的是 BLOB 页还是 XDES 页。
- `MONITOR_*_NO_WAITS / WAITS / WAIT_LOOPS` 三连（`MONITOR_INC_WAIT_STATS`）——log/flush 子系统的"自旋了没有 / 自旋几轮"，PFS 只有一个总等待时间。
- **mutex 自旋次数**——`SHOW ENGINE INNODB MUTEX` 的 `spins=`，PFS **完全看不到自旋**（它的 begin/end 在 mutex 外边界，对内部自旋循环一无所知）。

反过来 **PFS 能覆盖 monitor 做不到的**：per-thread/per-account 归属、延迟分布（min/avg/max）、关系型明细（`data_locks` 可 SQL 过滤与 JOIN）、按具体文件/具体 mutex 实例细分、跨引擎统一视图。

**耦合点只有三处，且都是手工的**（源码核实：`srv0mon.*` 里 grep `PSI_/pfs_` **0 命中**——monitor 与 PFS 之间**没有任何自动注册机制**）：

1. `lock_t::m_psi_internal_thread_id / m_psi_event_id`：建锁时（记录锁与表锁各一处）调 `PSI_THREAD_CALL(get_current_thread_event_id)` 打点。这是**纯 PFS 字段、与 monitor 无关**，但它让 `data_locks` 能 JOIN `events_statements_*` 定位"哪条 SQL 加的这把锁"——代价是每个 `lock_t` 大 16 字节 + 建锁路径一次间接调用。
2. `MONITOR_LATCHES` 开启时 `mutex_monitor->enable()`：**monitor 体系里唯一牵动重量级采集的地方**（遍历 `latch_meta` 全部 ~125 个 latch 类，把每个已注册实例的采集开关打开）。它默认关（`MONITOR_HIDDEN` 且非 `DEFAULT_ON`）。
3. 两边是**手工维护的两份清单**：InnoDB 手写 `all_innodb_mutexes[]` / `all_innodb_files[]` 等数组再 `mysql_xxx_register("innodb", ...)`；debug 构建还有一道断言防止新增 PSI key 忘了注册——这恰恰证明没有自动同步。

#### 选型决策树

```
要回答的问题
├─ "系统整体趋势如何？"（常开、零成本）
│     → monitor：INNODB_METRICS
├─ "内部算法哪里不对劲？"
│     ├─ 死锁假阳性 / 调度重算 / log 自旋轮数 / 页类型分布 / flush&LRU 细分 → monitor（唯一数据源）
│     └─ mutex 自旋次数 → monitor：SET GLOBAL innodb_monitor_enable='latch' + SHOW ENGINE INNODB MUTEX
├─ "是谁？哪条 SQL？哪个账号？"          → PFS（events_* / threads / data_lock_waits JOIN）
├─ "慢在哪？耗时多少？"                  → PFS（TIMER_WAIT 系列）
├─ "哪个文件 / 哪个 mutex 实例？"        → PFS（file_summary_by_instance / mutex_instances）
├─ "现在谁阻塞谁？"                      → PFS data_locks + data_lock_waits（低频 + 带 WHERE）
│     （高频探测改用 monitor 的 lock_threads_waiting / Innodb_row_lock_current_waits）
└─ "刚才那个死锁怎么回事？"              → Monitor 文本（SHOW ENGINE INNODB STATUS / innodb_print_all_deadlocks）
```

**开销量级**（选型的关键判据）：

| 操作 | 成本 |
|---|---|
| `MONITOR_INC`（默认开的那批） | 1 次位图 AND + relaxed load/store ≈ **3~5 条指令**，无原子 RMW |
| `MONITOR_ATOMIC_INC` | `lock xadd`，只在无锁保护处用 |
| PFS mutex instrument 关闭时 | 两次分支判断 ≈ 1~2 条指令 |
| PFS mutex instrument **开启** | 2 次间接调用 + **2 次时钟读取** + ≥4~6 次聚合写（比 monitor 高 1~2 个数量级） |
| `SELECT * FROM data_locks` | **持 `locksys` 全局排他 latch + `trx_sys->mutex`**，按 256 事务一批反复扫；1 万事务约 40 批 |

所以：latch 类 instrument 除非排查争用否则别开（默认就是关的）；`data_locks` 别做高频轮询（会短暂冻结整个锁子系统）；"有没有锁等待"的高频探测用计数器。

> 提醒：MTR 环境 `default_mysqld.cnf` 里有 `loose-performance-schema-instrument='%=ON'`，所以 MTR 结果里 `wait/synch/mutex/innodb/*` 显示 YES **不代表生产默认值**（生产默认 mutex/rwlock 是 NO，只有 file 类是 YES）。

## ★ 本机制里的工程实现技法

### 一、"ID 即下标"的三位一体与编译期守卫

`monitor_id_t` → 两个数组同下标，用 `static_assert` + `ut_a` 把"两个数组必须同步维护"这个最容易出错的约束变成硬失败。代价是加计数器必须同时改两处；收益是零间接寻址、零哈希查找，热路径只碰一个 `int64`。

### 二、位图开关 + 无锁累加：用"极简"换掉"通用"

对比 PFS 的 instrument 注册 + consumer 过滤体系，monitor counter 用 1 bit 表示开关、一次 AND 判定。埋点是编译期决定的，开关只是运行期的一个 bit——**关着的计数器仍有一次 AND 的开销（不可避免），但没有任何函数调用、指针追逐或 branch table**。

### 三、复用业务互斥锁而非为监控加锁

这是整套体系最核心的性能决策。它把监控正确性寄生在 InnoDB 已有的锁协议上（lock/buf/log/trx 各自临界区内的埋点天然安全）。`std::atomic` 的引入也只是为了消除 UB——**原子化不等同于语义变更**，这是对既有正确性的保守保护。

### 四、把成本从写路径挪到读路径

`MONITOR_EXISTING` 的惰性差分（几十个老 status var 零埋点开销）、`MONITOR_INC_NOCHECK` 的模块级单次判定、以及 `MEB/hotbackup` 构建下 `#define MONITOR_INC(x) ((void)0)` 的编译期整体关闭——三种手法同构。

---

## 可观测性

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `innodb_status_output` | OFF | 周期线程是否把 Monitor 输出写进 error log（15s 一次，慎开） |
| `innodb_status_output_locks` | OFF | 锁段是否逐个打印每把锁（每事务最多 10 把）；与 `innodb_status_output` 在源码中完全独立 |
| `innodb_status_file` | OFF | 是否用 datadir 下具名 `innodb_status.<pid>` 代替匿名临时文件 |
| `innodb_monitor_enable` / `_disable` / `_reset` / `_reset_all` | NULL | 计数器开关（名字 / 模块名 / `%` 通配） |
| `innodb_print_all_deadlocks` | OFF | 死锁报告同步刷 error log（否则只进 `lock_latest_err_file`） |
| `innodb_semaphore_wait_timeout_debug` | 600 | **仅 debug 构建**；生产的 `srv_fatal_semaphore_wait_threshold` 恒为 600 |

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 全系统全景快照 | SQL | `SHOW ENGINE INNODB STATUS`（现场生成，1MB 上限） |
| 某类事件的次数/极值/均值 | SQL | `information_schema.INNODB_METRICS`（按 `SUBSYSTEM` 过滤，如 `lock`） |
| 周期性的历史留痕 | trace | `innodb_status_output=ON` → error log；`innodb_status_file=ON` → 文件 |
| 死锁留痕 | trace | `innodb_print_all_deadlocks=ON` |

---

## Misc

### 易混淆的命名与口径

- **`module_lock` vs `lock`**：前者是计数器名（模块标记行，不出现在表里），后者是 `SUBSYSTEM` 列名。
- **`lock_rec_lock_waits` vs `Innodb_row_lock_waits`**：前者 = 入队次数（每次进等待队列就 +1），后者 = 实际挂起次数（真睡下去才计）。超时场景两者明显不一致。
- **`lock_deadlock_false_positives` 不为 0 是正常的**：它记录"候选环被两道校验否决"，恰恰证明快照式并发检测撞到过假环。
- **`monitor` 不是 server 层概念**：`SHOW STATUS` 是 server 层的，`SHOW ENGINE INNODB STATUS` 是 InnoDB 的，两者通过 `export_vars` + `SHOW_VAR` 衔接。

### 面向二次开发

**扩展点**：加一个计数器 = `monitor_id_t` 末尾前插入 + `innodb_counter_info[]` 同下标处插入一行（名字/子系统/描述/类型标志）+ 埋点 `MONITOR_INC`。加一个 Monitor 段落 = 改 `srv_printf_innodb_monitor()`（没有注册表，必须改单体函数）。

**坑与已知缺陷**：

- 模块标记行 `module_cpu` / `module_page_track` / `module_dblwr` 的 `monitor_type` 写的是 `MONITOR_NONE` 而非 `MONITOR_MODULE`——它们**不被当作模块边界**（无法用模块名批量开关），且会作为普通 counter 行出现在表里。
- `MONITOR_NO_AVERAGE` 标志在 8.0.39 **没有任何计数器使用**（预留能力）。
- 已开启的计数器再 `enable` 会被拒绝（模块路径打 info、通配符路径报错中断），避免误触发隐式 reset。
- 生产环境 `srv_fatal_semaphore_wait_threshold` 恒为 600 不可调——长等待崩溃的阈值改不了（除非 debug 构建）。

**社区边界澄清**：`SHOW ENGINE INNODB STATUS` 的文本格式、monitor counter 的名字、`INNODB_METRICS` 都是社区版能力；Percona/MariaDB 有各自的额外计数器与 `SHOW ENGINE INNODB STATUS` 扩展段（如 Percona 的 `INNODB_CHANGED_PAGES`），社区版没有。

### ★★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| InnoDB 有三个 monitor 线程 | 实际是 **4 个**：master / monitor / error-monitor / **lock_wait_timeout**（第 4 个在 5.7 叫 `srv_lock_timeout_thread`） |
| `srv_lock_timeout_thread` 还在 | **不存在**（全库无定义）。真名 `lock_wait_timeout_thread`，定义在 **`lock0wait.cc:1439`**（不是 `lock0lock.cc`） |
| `buf_flush_page_cleaner` 是 monitor 类线程 | **不是**。它是 page cleaner，与 monitor 输出无关 |
| `enum srv_thread_type` 列出所有线程 | **错**。只有 `SRV_NONE/WORKER/PURGE/MASTER` **4 个**，只用于 slot 分配；三个 monitor 类线程**不在枚举里、不占 slot** |
| `srv_printf_innodb_monitor` 里有 `current_monitor_counter > 15` 节流 | **不存在**（符号全库 0 命中）。节流是外层：monitor 线程每 15 秒醒一次 |
| `SRV_MONITOR_INTERVAL` 常量 | **不存在**。间隔全是内联字面量：monitor 15s、error-monitor 1s、lock-wait-timeout 1s、master 1s |
| `mysql_show_engine_status()` | **不存在**。链路是 `Sql_cmd_show_engine_status::execute_inner`（`sql_show.cc:478`）→ `ha_show_status`（`handler.cc:7677`）→ `innodb_show_status`（`ha_innodb.cc:18940`） |
| 位图开关宏叫 `MONITOR_SET_ON` | **不存在**。真名 `MONITOR_ON` / `MONITOR_OFF` / `MONITOR_IS_ON`（`srv0mon.h:583`） |
| `monitor_id_t` 是 `enum monitor_id_value` | **错**。`srv0mon.h:132` 就是 `enum monitor_id_t`；`monitor_id_value` 全库不存在（`srv0mon.cc:95` 注释里那个是**过期注释**） |
| `i_s_innodb_metrics_fill_table` 是填充函数 | 填充真名 `i_s_metrics_fill_table`（`i_s.cc:2074`）→ `i_s_metrics_fill`（`i_s.cc:1813`）；`i_s_innodb_metrics` 是**插件结构体**名（`i_s.cc:2106`） |
| `innodb_export_vars_at_start` | **不存在**。真名：`struct export_var_t`（`srv0srv.h:1126`）、实例 `export_vars`（`srv0srv.h:782`）、填充 `srv_export_innodb_status()`（`srv0srv.cc:1577`） |
| `MONITOR_EXISTING` 与老 status var 共享存储 | **错，不共享**。老的在 `export_vars`/`srv_stats`，counter 在 `innodb_counter_value[]`；靠 `srv_mon_process_existing_counter()` 在 turn on/off/reset/get 时刻**做差值模拟**共享 |
| 8.0 把 master thread 删了 | **没删**（`srv0srv.cc:2749`）。但主循环已被掏空：只剩 1s sleep + `log_free_check()` + 活跃/空闲任务 + `fil_purge()` |
| `MONITOR_INC` 是原子的 | **错**。`monitor_inc_value` 是 relaxed load + store 两步走，**非原子**；只有 `MONITOR_ATOMIC_INC/DEC` 走 `fetch_add` |

**★ 四个结构性事实**

1. **"三个 monitor 线程"这个 5.7 的说法在 8.0 需要修正为四个，且它们的身份不在同一个枚举里**。`enum srv_thread_type` 只有 4 个成员且只用于 slot 分配——master thread 是唯一占 slot 的 monitor 类线程，另外三个靠 PFS key + `srv_threads` 成员管理。只读模式下一个都不起。

2. **"临时文件中转"不是绕路，是精确控制 1 MiB 上限的必要手段**。`srv_printf_innodb_monitor()` 只会往 `FILE*` 写，而协议层要内存 buffer；先写文件、量长度、再回读，才能做到"挖心"式截断——**保留事务列表尾部**（最新事务最有价值）而砍掉头部。

3. **`MONITOR_EXISTING` 不是共享存储，是"记账模拟"**。老 status variable 没有 start/stop/reset 语义，InnoDB 通过在开/关/重置/查询四个时刻从源头取值、套 `MONITOR_SET_DIFF` 公式（`当前值 - reset基线 - 开启快照 + 上次关闭值`）来**模拟**出这些语义。这解释了为什么 `INNODB_METRICS` 里 `lock_row_lock_waits` 和 `SHOW STATUS` 的 `Innodb_row_lock_waits` 数值一致——**源头相同，存储两份**。

4. **无锁累加的代价是官方明确接受的**。`monitor_inc_value` 用 relaxed load+store 两步而非 `fetch_add`，注释直说 *"we want to leave it non-atomic as it was before changing mon_value to std::atomic"*；max/min 的 race 也明说 *"not 100% accurate ... we ignore it due to performance"*。而**关着的 counter 连一次 store 都不做**（`MONITOR_IS_ON` 先行判定）——这才是"零成本"的真正来源。

> 注：部分 `/* 【注】... */` 与 `★` 注释为本工作区人工批注，非 Oracle 上游原文。

---

### ★ monitor counter 的真实形态：一半是计数器，一半是"取值器"

这是理解 `INNODB_METRICS` 的关键——它**不是**一套统一实现的计数器，而是两套东西混在一起。

#### ① 元数据：`monitor_info_t`

```cpp
struct monitor_info_t {
  const char *monitor_name;        /*!< Monitor name */
  const char *monitor_module;      /*!< Sub Module the monitor belongs to */
  const char *monitor_desc;        /*!< Brief desc of monitor counter */
  monitor_type_t monitor_type;     /*!< Type of Monitor Info */
  monitor_id_t monitor_related_id; /*!< Monitor ID of counter that related to
                                  this monitor. This is set when the monitor
                                  belongs to a "monitor set" */
  monitor_id_t monitor_id;         /*!< Monitor ID as defined in enum monitor_id_t */
};
```

⇒ 每个 counter 是一张**静态表**里的一行（名字、所属模块、描述、类型、ID）。

#### ② 值与开关：宏 + bitmap

```cpp
#define MONITOR_FIELD(monitor, field) (innodb_counter_value[monitor].field)
#define MONITOR_VALUE(monitor) MONITOR_FIELD(monitor, mon_value)
#define MONITOR_MAX_VALUE(monitor) MONITOR_FIELD(monitor, mon_max_value)
```

```cpp
/** This "monitor_set_tbl" is a bitmap records whether a particular monitor
counter has been turned on or off */
extern ulint
    monitor_set_tbl[(NUM_MONITOR + NUM_BITS_ULINT - 1) / NUM_BITS_ULINT];
```

⇒ **值**存在全局数组 `innodb_counter_value[]` 里；**开关**用一个**位图** `monitor_set_tbl`（一个 ulint 的每一位代表一个 counter 的开/关）。这是为了**省内存**：几百个 counter 的开关只占几十字节。

★ 这也是为什么 `MONITOR_IS_ON()` 检查是位运算，以及为什么 **`INNODB_METRICS` 里默认只有一部分 counter 是 enabled**——位图初值决定的。

#### ③ ★ 真正的分歧点：`MONITOR_EXISTING`

`srv_mon_process_existing_counter()` 的实现暴露了两种 counter 的差别：

```cpp
void srv_mon_process_existing_counter(
    monitor_id_t monitor_id, /*!< in: the monitor's ID as in monitor_counter_id */
    mon_option_t set_option) /*!< in: Turn on/off reset the counter */
{
  mon_type_t value;
  monitor_info_t *monitor_info;
  bool update_min = false;
  buf_pool_stat_t stat;
  buf_pools_list_size_t buf_pools_list_size;
  ulint LRU_len;
  ulint free_len;
  ulint flush_list_len;

  monitor_info = srv_mon_get_info(monitor_id);

  ut_a(monitor_info->monitor_type & MONITOR_EXISTING);
  ut_a(monitor_id < NUM_MONITOR);

  /* Get the value from corresponding global variable */
  switch (monitor_id) {
    /* export_vars.innodb_buffer_pool_reads. Num Reads from
    disk (page not in buffer) */
    case MONITOR_OVLD_BUF_POOL_READS:
      value = srv_stats.buf_pool_reads;
      break;

    /* innodb_buffer_pool_read_requests, the number of logical
    read requests */
    case MONITOR_OVLD_BUF_POOL_READ_REQUESTS:
      buf_get_total_stat(&stat);
      value = Counter::total(stat.m_n_page_gets);
      break;
    ...
```

**★ 关键结论**：

- **`ut_a(monitor_info->monitor_type & MONITOR_EXISTING)`** —— 这个函数**只服务** `MONITOR_EXISTING` 类型的 counter。
- 这类 counter **不是"被累加的计数器"**，而是**别名**：每次要取值时，通过这个**大 switch** 从各处**现捞**：
  - 有的直接读全局变量（`value = srv_stats.buf_pool_reads;`）
  - 有的要先聚合（`buf_get_total_stat(&stat); value = Counter::total(stat.m_n_page_gets);` —— 遍历所有 buffer pool 实例求和）
- 另一类（非 `MONITOR_EXISTING`）才是代码里用 `MONITOR_INC()` **真正累加**的计数器，值就在 `innodb_counter_value[]` 里。

⇒ **实践含义**：
1. `INNODB_METRICS` 里**同一列**（`COUNT`）背后可能是两种完全不同的取数路径
2. 需要**跨实例聚合**的 counter（buffer pool 相关）每次查询都要遍历所有实例，**代价更高**
3. 所以 `innodb_monitor_enable` 打开一个 counter 的代价，取决于它是哪一类

#### ④ 开关：`mon_option_t`

```cpp
enum mon_option_t {
  MONITOR_TURN_ON = 1,     /*!< Turn on the counter */
  MONITOR_TURN_OFF,        /*!< Turn off the counter */
  MONITOR_RESET_VALUE,     /*!< Reset current values */
  MONITOR_RESET_ALL_VALUE, /*!< Reset all values */
  MONITOR_GET_VALUE        /*!< Option for srv_mon_process_existing_counter() function */
};
```

⇒ `MONITOR_RESET_ALL_VALUE` 就是 `SET GLOBAL innodb_monitor_reset_all = ...` 背后的语义（重置所有值，含 max/min）；`MONITOR_RESET_VALUE` 只重置当前值（保留 max/min）。**这两个的区别在排障时很有用**：想看峰值就得只 reset 当前值而保留 max。

#### ⑤ `SHOW ENGINE INNODB STATUS` 的输出

入口在 `ha_innodb.cc` 的 `innodb_show_status()`，核心是：

```cpp
  srv_printf_innodb_monitor(srv_monitor_file, false, &trx_list_start, ...);
```

⇒ 它是一个**一次性把全部段落（SEMAPHORES / TRANSACTIONS / BUFFER POOL AND MEMORY …）打印成一段大文本**的函数。**没有"只打印某一段"的能力**——这就是它的代价所在：你只想看死锁，它也得把整个状态算一遍。

**★ 输出链路图**（★ 自检⑪ 的"分层数据流"）：

```
                        ┌──────────────────────────────────┐
                        │  srv_monitor_thread（每 15 秒）  │
                        └───────────────┬──────────────────┘
                                        │ 同一个函数写两处
                        ┌───────────────┴───────────────┐
                        ▼                               ▼
            ① stderr（进 error log）        ② srv_monitor_file（全局临时文件）
               srv_printf_innodb_monitor      rewind() → 写 → os_file_set_eof()
                                                      │
                                                      │ 回读
                                                      ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ innodb_show_status()（SHOW ENGINE INNODB STATUS 触发）        │
   │   mutex_enter(&srv_monitor_file_mutex)                       │
   │   rewind(srv_monitor_file) → srv_printf_innodb_monitor(...)   │
   │   ftell() 量长度 → 若 > 1 MiB 做"挖心"截断                    │
   │   my_malloc → fread 回读 → stat_print() 发给客户端            │
   └──────────────────────────────────────────────────────────────┘
```

```
  调用链（三层）：
  Sql_cmd_show_engine_status::execute_inner   (sql_show.cc)     需 PROCESS_ACL
    → ha_show_status                          (handler.cc)
      → hton->show_status = innodb_show_status (ha_innodb.cc)
        → srv_printf_innodb_monitor            (srv0srv.cc)     持 srv_innodb_monitor_mutex
           分区各自的责任函数：
             BACKGROUND THREAD → srv_print_master_thread_info
             SEMAPHORES        → sync_print
             TRANSACTIONS      → lock_print_info_summary + lock_print_info_all_transactions
             FILE I/O          → os_aio_print
             INSERT BUFFER/AHI → ibuf_print + ha_print_info
             LOG               → log_print
             BUFFER POOL       → buf_print_io
             ROW OPERATIONS    → 直接 fprintf 读 srv_stats
```

> ★★ **为什么要走"临时文件中转"**：`srv_printf_innodb_monitor()` 只会往 `FILE*` 写（`fprintf` 一族），而协议层要的是内存 buffer。**先写文件、`ftell` 量长度、再 `rewind` 回读**——才能在"不知道长度"的前提下精确控制 1 MiB 上限，且能做"挖心"式截断（保留事务列表**尾部**，因为最新事务最有价值）。
> ★ 注意 `srv_innodb_monitor_mutex` 是**全局串行化**的：`SHOW ENGINE INNODB STATUS` 与 `SHOW STATUS`（`srv_export_innodb_status` 也持它）会互相阻塞。

⚠️ 而且它持有 InnoDB 内部锁（`srv_printf_innodb_monitor` 期间），**高频轮询会让业务受影响**，官方也不建议当监控数据源用。

**★★ 完整的三层调用链**（⚠️ 不存在 `mysql_show_engine_status` 这个函数）：

```cpp
/* sql/sql_show.cc:474 —— server 层 */
bool Sql_cmd_show_engine_status::check_privileges(THD *thd) {
  return check_global_access(thd, PROCESS_ACL);     /* ★ 需要 PROCESS 权限 */
}
bool Sql_cmd_show_engine_status::execute_inner(THD *thd) {
  return ha_show_status(thd, lex->create_info->db_type, HA_ENGINE_STATUS);
}
```
```cpp
/* sql/handler.cc:7677 —— 通用执行器，回调到引擎 */
      result = db_type->show_status &&            /* InnoDB = innodb_show_status */
                       db_type->show_status(db_type, thd, stat_print, stat)
                   ? true : false;
```

**★★ `innodb_show_status()`：临时文件中转 + 1 MiB 截断**（`ha_innodb.cc:18934`）：

```cpp
static int innodb_show_status(handlerton *hton, THD *thd,
                              stat_print_fn *stat_print) {
  static const char truncated_msg[] = "... truncated...\n";
  const long MAX_STATUS_SIZE = 1048576;                 /* ★ 硬上限 1 MiB */
  ...
  mutex_enter(&srv_monitor_file_mutex);
  rewind(srv_monitor_file);                             /* ① 复用全局临时 FILE*，先复位 */

  srv_printf_innodb_monitor(srv_monitor_file, false, &trx_list_start,
                            &trx_list_end);             /* ② nowait=false → 阻塞等全局 latch */

  os_file_set_eof(srv_monitor_file);
  if ((flen = ftell(srv_monitor_file)) < 0) flen = 0;

  if (flen > MAX_STATUS_SIZE) {
    usable_len = MAX_STATUS_SIZE;
    srv_truncated_status_writes++;                      /* ③ 截断计数（本身也是个指标） */
  } else { usable_len = flen; }

  str = (char *)my_malloc(PSI_INSTRUMENT_ME, usable_len + 1, MYF(0));
  rewind(srv_monitor_file);                             /* ④ 回读 */
  ...
  } else if (trx_list_end < (ulint)flen && trx_list_start < trx_list_end &&
             trx_list_start + (flen - trx_list_end) <
                 MAX_STATUS_SIZE - sizeof truncated_msg - 1) {
    /* Omit the beginning of the list of active transactions. */   /* ★★ "挖心"式截断 */
    ssize_t len = fread(str, 1, trx_list_start, srv_monitor_file);
    memcpy(str + len, truncated_msg, sizeof truncated_msg - 1);
    ...
  }
  mutex_exit(&srv_monitor_file_mutex);
  ret_val = stat_print(thd, innobase_hton_name, ..., str, (uint)flen);  /* ⑤ 发给客户端 */
```

> ★★ **为什么走临时文件**？因为 `srv_printf_innodb_monitor()` 只往 `FILE*` 写（`fprintf` 一族），而协议层要的是内存 buffer。所以先写文件、量出长度、再回读——**"先落盘再回读"是为了在不知道长度的情况下精确控制 1 MiB 上限**，且能"挖心"保留事务列表的尾部（最新事务通常最有价值）。

**★ 分区 → 责任函数对照**（★ 每个段落由谁打印）：

| 输出分区 | 责任函数 | 定义位置 |
|---|---|---|
| BACKGROUND THREAD | `srv_print_master_thread_info` | `srv/srv0srv.cc:871` |
| SEMAPHORES | `sync_print` | `sync/sync0sync.cc:204` |
| LATEST FOREIGN KEY ERROR | `ut_copy_file(file, dict_foreign_err_file)` | 另一个临时文件 |
| TRANSACTIONS（锁摘要） | `lock_print_info_summary` | `lock/lock0lock.cc:4660` |
| TRANSACTIONS（事务列表） | `lock_print_info_all_transactions` | `lock/lock0lock.cc:4991` |
| FILE I/O | `os_aio_print` | `os/os0file.cc:7605` |
| INSERT BUFFER AND ADAPTIVE HASH INDEX（ibuf） | `ibuf_print` | `ibuf/ibuf0ibuf.cc:4381` |
| ……（AHI） | `ha_print_info` | `ha/ha0ha.cc:326` |
| LOG | `log_print` | `log/log0log.cc:1132` |
| BUFFER POOL AND MEMORY | `buf_print_io` | `buf/buf0buf.cc:6669` |
| ROW OPERATIONS | 直接 `fprintf` 读 `srv_stats.*` | 无独立函数 |

**★ "Per second averages calculated from the last N seconds" 的 N 从哪来**（`srv0srv.cc:1353`）：

```cpp
  const auto time_elapsed = std::chrono::duration_cast<std::chrono::seconds>(
                                current_time - srv_monitor_stats_refreshed_at)
                                .count() + 0.001;   /* ★ +0.001 防除零 */
  srv_monitor_stats_refreshed_at = current_time;
  ...
  fprintf(file, "Per second averages calculated from the last %lu seconds\n",
          (ulong)time_elapsed);
```
> ★★ **没有"15 秒节流"**——那个数是"距上次刷新基线过了多少秒"。基线由 monitor 线程每 60 秒 `srv_refresh_innodb_monitor_stats()` 刷新一次。
> ⚠️ `current_monitor_counter > 15` 这个说法**不存在**（符号 `current_monitor_counter` 全库 0 命中）。

### 与 `INNODB_METRICS` 的关系

| | `SHOW ENGINE INNODB STATUS` | `INNODB_METRICS` 表 |
|---|---|---|
| 形态 | 一段大文本 | 一张表（可 `SELECT`、可过滤） |
| 取数 | 一次性全算 | 逐 counter 取（`MONITOR_VALUE()` 或 `srv_mon_process_existing_counter()`） |
| 粒度 | 固定段落 | 每个 counter 一行，可单独开关 |
| 代价 | 高（全量 + 持锁） | 低（只算你要的那几个） |

⇒ **能用 `INNODB_METRICS` 就别轮询 `SHOW ENGINE INNODB STATUS`**；后者只在需要看**死锁详情 / 事务列表 / 信号量**这些没有对应 counter 的信息时才用。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB INFORMATION_SCHEMA Metrics Table*
- *MySQL 8.0 Reference Manual → SHOW ENGINE INNODB STATUS*

**内核月报 / 技术文章**
- MySQL 官方 blog：InnoDB Metrics（monitor counter 的设计说明）

**相关文档**
- server 层状态变量体系见 [`../server/infra/variables.md`](../../server/infra/variables.md) G 节
- PFS instrument 体系见 [`../server/infra/pfs.md`](pfs/pfs.md)
- PFS 锁表（`data_locks`）的引擎侧实现见 [`../infra/lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)
- 死锁检测用到的 monitor 计数（真死锁/假阳性）见 [`../infra/lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)
