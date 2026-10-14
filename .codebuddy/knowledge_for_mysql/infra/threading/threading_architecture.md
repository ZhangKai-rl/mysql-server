# 全栈线程模型深度解析

> 基于 MySQL 8.0.39 源码，剖析**跨层**的线程模型：server 层连接线程 + InnoDB 后台线程 + 复制/事件/mysys 线程的全景清单，以及**线程 × 并发控制 × PFS** 的三向耦合。
>
> **边界**：本篇讲**全栈线程的"谁、多少、怎么管、怎么观测"**以及它与锁的耦合关系。
> - server 层连接线程的**细节**（accept / 线程缓存三件套 / 无超时等待）见 [`../server/infra/connection_threading.md`](connection_threading.md)
> - 各 latch / rw_lock 的**实现细节**见 [`../lock/`](../lock/)
> - 事务与行锁见 [`../innodb/trx.md`](../../innodb/trx.md)、[`lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)
> - PFS 的**事件/表结构**见 [`../server/infra/pfs.md`](../observability/pfs/pfs.md)
>
> ★ 为什么需要这篇：线程不是某一层的东西。`max_connections` 在 server 层，`innodb_thread_concurrency` 在 InnoDB 层，它们之间是一张**漏斗**；而一条 SQL 卡住时"挂在哪里"由它撞上的是哪一层锁决定——MDL、行锁、latch 三种等待机制**互不相同**。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量](#相关的系统变量)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 的线程体系分**两层、两类**：

| | 层 | 类别 | 数量特征 |
|---|---|---|---|
| **用户线程**（foreground） | server 层 | one-thread-per-connection | 动态，上限 `max_connections` |
| **后台线程**（background） | InnoDB + server + 复制 + 其他 | 常驻 / 按需 | 约 30 种，多数固定 1 个 |

★ **一条重要的事实**：用户线程与后台线程**不在同一个数量级管理**。用户线程由 `Connection_handler_manager` 计数与限流；InnoDB 后台线程登记在一个全局 `Srv_threads` 结构里，靠 shutdown 状态机按顺序停。

### 用途

回答三个"跨度很大"的实际问题：

1. **`max_connections` 和 `innodb_thread_concurrency` 到底谁管谁？** → 见[核心实现四](#四线程--并发控制漏斗与三种等待挂靠)
2. **一条 SQL 卡住时，线程到底"挂"在哪？** → 三种机制完全不同，见[核心实现四](#四线程--并发控制漏斗与三种等待挂靠)
3. **后台线程在 `performance_schema.threads` 里叫什么？** → 见[核心实现六](#六pfs如何观测这整堆线程)

### 版本演进

| 版本 | 变化 |
|---|---|
| 8.0 | InnoDB 线程创建从 pthread 迁移到 **`std::thread`**（`IB_thread`/`Detached_thread`） |
| 8.0 | redo 日志线程重构：新增 `log_writer`/`log_flusher`/`log_write_notifier`/`log_flush_notifier`/`log_checkpointer`/`log_files_governor` |
| 8.0.17+ | `lock_sys` 分片（缓解 latch 争用——见"漏斗"的根因分析） |

---

## 理论基础

### ★ 核心洞察：线程数与并发度是两级不同的闸门

这是理解整个线程模型的关键，也是很多人混淆的地方：

```
闸门 0：max_connections        ← "能有多少条连接"（线程维度，server 层）
闸门 1：innodb_thread_concurrency ← "能有多少线程同时在 InnoDB 里"（并发维度，InnoDB 层）
闸门 2：latch / rw_lock        ← 内存结构的互斥
闸门 3：行锁 / 表锁            ← 事务语义的互斥
闸门 4：物理资源               ← buffer pool / redo / IO
```

**闸门 0 与闸门 1 的区别**：

| | `max_connections` | `innodb_thread_concurrency` |
|---|---|---|
| 限制对象 | 连接数 / THD 数 | 同时"在 InnoDB 内核里"的线程数（`srv_conc.n_active`） |
| 阻塞方式 | **连接被拒绝**（`ER_CON_COUNT_ERROR`） | 已连上的线程在 InnoDB 门口 **`sleep_for` 轮询**（连接仍活着、THD 仍占用） |
| 默认 | 151 | **0 = 不限制（闸门关闭）** |

★ **默认配置下的后果**：闸门 1 关闭 ⇒ 151 个连接可以全部涌进 InnoDB ⇒ **压力全部落到闸门 2（latch）**。这就是 8.0 在连接数上去之后 `dict_sys` / `lock_sys` / `trx_sys` latch 争用 + CPU sys 飙高的根因，也是 8.0.17+ 引入 `lock_sys` 分片、AHI 分区、`trx_sys` shard 这些补丁要解决的问题。

### ★ 等待的"挂靠点"决定了一切

一条 SQL 阻塞时，线程"挂"在哪个数据结构上，完全取决于它撞上的是哪一层锁。**三种机制互不相同**（详见核心实现四）：

| 层 | 挂靠结构 | 由谁唤醒 | 有无超时 |
|---|---|---|---|
| MDL | `MDL_wait::m_COND_wait_status`（每 THD 一个 mutex+cond） | 持锁者 `set_status()` | 有（`lock_wait_timeout`，默认 **1 年**） |
| 行锁 / 表锁 | `lock_sys->waiting_threads[]` 的 `srv_slot_t` | `lock_wait_timeout_thread` / 放锁者 | 有（`innodb_lock_wait_timeout`，50s） |
| latch / rw_lock | `sync_array_t` 的 `sync_cell_t` | `mutex_exit()` + `srv_error_monitor_thread` 兜底 | **无** |

★ **latch 等待没有超时**——这是最危险的一层。release 版本里 latch 死锁不会自己解开，只会：等满 4 分钟开始往 stderr 打警告，连续 10 次同一 sema 后 `ib::fatal` **crash**。

### 理论溯源

两级闸门是"资源分层限流"的经典模式：外层限"接入量"（connection），内层限"并发执行量"（concurrency）。好处是内层可以在不拒绝连接的前提下平滑压力；代价是等待线程仍占内存与 THD，所以**闸门 1 的等待是有成本的**（不像闸门 0 直接拒绝）。

自旋-休眠混合是"乐观等待"的经典模式：先自旋（赌锁很快释放），失败后升级为真睡（避免烧 CPU）。InnoDB 把这条链路做到了三层，见[核心实现五](#五syncarray-与自旋-休眠模型)。

---

### 全景分层图：哪族线程、谁生的、谁杀的

```
┌─ server 层 ─────────────────────────────────────────────────────────────┐
│  main 线程（bootstrap）                                                  │
│    ├─ 常驻：signal_hand / timer_notifier / compress_gtid_table  ...      │
│    ├─ connection 线程族：one_thread / per_thread（默认）                 │
│    │     └─ per_thread：mysql_thread_create() → handle_connection()      │
│    │            └─ for(;;) { new THD → store_globals() → 服务           │
│    │                         → delete THD → block_until_new_connection() │
│    │                                        ↑ 命中 thread_cache 则复用   │
│    └─ 按需/临时：bootstrap / init_file / 插件线程 / clone / ...          │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │ 边界：绝大多数 InnoDB 代码跑在用户线程里
┌─ InnoDB 层 ──────────────┴───────────────────────────────────────────────┐
│  后台线程（8.0 起 std::thread + detach，见「二」）                        │
│    ├─ master（8.0 已很清闲）/ purge{coordinator,worker} / page_cleaner   │
│    ├─ io_read / io_write（AIO handler）/ ibuf / log_writer ...           │
│    └─ 守护三兄弟：lock_wait_timeout / error_monitor / monitor            │
│  共享结构：Srv_threads（全局句柄表）、sync_array、srv_conc 排队数组      │
└──────────────────────────────────────────────────────────────────────────┘

并发度是三级闸门（不是一级）：
   连接数 max_connections ──► 线程缓存 thread_cache_size ──► InnoDB 门票
   （闸门 0，ER_CON_COUNT_ERROR）（闸门 0.5，命中则复用）    （闸门 1，ticket）
                                                                 │
                                                    InnoDB 内部还有闸门 2/3
                                                    （行锁等待 / 后台并行度）

退出顺序（shutdown 九阶段）反过来走：先断用户线程 → 再停后台 → 最后 DD/log
```

★ 这张图要记住两点：① **用户线程是主力**——绝大多数 InnoDB 代码运行在连接线程里，后台线程只是辅助；② **线程数 ≠ 并发度**——`max_connections` 放行、线程缓存复用、`innodb_thread_concurrency` 才真正限制"同时在 InnoDB 里的线程数"，三者是串联的三级闸门。

## 核心实现

### 一、全栈线程清单

#### 1.1 InnoDB 后台线程（`Srv_threads` 全局结构）

```cpp
// storage/innobase/include/srv0srv.h
/** Structure which keeps shared future objects for InnoDB background
threads. One should use these objects to check if threads exited. */
struct Srv_threads {
  IB_thread m_monitor;               // Monitor thread (prints info)
  IB_thread m_error_monitor;         // Error monitor thread
  IB_thread m_log_files_governor;    // Redo files governor thread
  IB_thread m_log_checkpointer;      // Redo checkpointer thread
  IB_thread m_log_writer;            // Redo writer thread
  IB_thread m_log_flusher;           // Redo flusher thread
  IB_thread m_log_write_notifier;    // Redo write notifier thread
  IB_thread m_log_flush_notifier;    // Redo flush notifier thread
  IB_thread m_backup_log_archiver;
  IB_thread m_buf_dump;              // Buffer pool dump thread
  IB_thread m_buf_resize;            // Buffer pool resize thread
  IB_thread m_dict_stats;            // Dict stats background thread
  IB_thread m_lock_wait_timeout;     // Thread detecting lock wait timeouts
  IB_thread m_master;                // The master thread
  IB_thread m_ts_alter_encrypt;
  IB_thread m_trx_recovery_rollback; // Thread doing rollbacks during recovery
  IB_thread m_recv_writer;           // Thread writing recovered pages during recovery
  IB_thread m_purge_coordinator;     // Purge coordinator (also being a worker)
  size_t    m_purge_workers_n;
  IB_thread *m_purge_workers;        // [0] == m_purge_coordinator
  IB_thread m_page_cleaner_coordinator;
  size_t    m_page_cleaner_workers_n;
  IB_thread *m_page_cleaner_workers; // [0] == coordinator
  IB_thread m_log_archiver;
  IB_thread m_page_archiver;
  IB_thread m_fts_optimize;          // FTS index optimization
  IB_thread m_gtid_persister;
  os_event_t m_master_ready_for_dd_shutdown;
};
extern Srv_threads srv_threads;
```

**完整线程表**（逐个核实）：

| 线程函数 | 数量 / 控制变量（默认） | 职责 |
|---|---|---|
| `srv_master_thread` | 1 | 8.0 仍存在，但**已不做 purge/刷脏**；剩下 dict cache LRU 淘汰、`fil_purge()`、`log_free_check()` |
| `srv_purge_coordinator_thread` | 1（也是 worker[0]） | purge 调度 |
| `srv_worker_thread` | `innodb_purge_threads - 1`（**默认 4** ⇒ 1 coord + 3 worker） | purge 执行 |
| `buf_flush_page_coordinator_thread` | 1（也是 worker[0]） | 脏页刷盘调度 |
| `buf_flush_page_cleaner_thread` | `innodb_page_cleaners - 1`（**默认 4**） | 脏页刷盘执行 |
| `log_writer` | 1 | log buffer → redo 文件，推进 `write_lsn` |
| `log_flusher` | 1 | fsync，推进 `flushed_to_disk_lsn` |
| `log_write_notifier` | 1 | 批量唤醒"等 write"的用户线程 |
| `log_flush_notifier` | 1 | 批量唤醒"等 flush"的用户线程 |
| `log_checkpointer` | 1 | 周期性写 checkpoint |
| `log_files_governor` | 1 | redo 文件创建/回收/扩容 |
| `lock_wait_timeout_thread` | 1 | ★ **同时负责超时检查与死锁检测** |
| `srv_monitor_thread` | 1 | 每 15 秒打印 InnoDB Monitor |
| `srv_error_monitor_thread` | 1 | ★ **sync_array 兜底唤醒 + latch 死锁检测 + 长等待致命判定**（1Hz） |
| `buf_dump_thread` / `buf_resize_thread` | 各 1 | buffer pool dump/load、在线 resize |
| `dict_stats_thread` | 1 | 后台统计重算 |
| `fts_optimize_thread` | 1（固定） | 全文索引优化（注释："For now we only support one optimize thread."） |
| `trx_recovery_rollback_thread` | 0 或 1 | 仅崩溃恢复且需回滚时 |
| `recv_writer_thread` | 0 或 1 | 仅恢复期间 |
| `log_archiver_thread` / `page_archiver_thread` | 各 1 | Clone / 备份归档 |
| `io_handler_thread` | **1(ibuf) + `innodb_read_io_threads`(4) + `innodb_write_io_threads`(4) = 9** | AIO 收割 |
| `backup_log_archiver` | 0 或 1（仅 MEB 场景） | ★ `log0meb.cc`，MySQL Enterprise Backup 的 redo 归档；**非 MEB 场景根本不启动** |
| `parallel_read_thread` | **查询期按需**，`innodb_parallel_read_threads`（默认 4） | ★ `row0pread.cc` `Parallel_reader::worker`；★ 线程数为 0 时**调用线程自己干、不起线程** |
| `parallel_rseg_init_thread` | **启动期临时**，`srv_rseg_init_threads` | ★ `trx0rseg.cc` fork-join：并行初始化 rollback segment，`start()` 后立刻 `join()` |
| `fsp_init_resume_alter_encrypt_tablespace` | 0 或 1（仅当上次有未完成加/解密） | `srv0start.cc`；★ 主线程要 `cond_wait` 等它拿到所有 shared MDL 才继续 |
| `clone_gtid_thread` | 1（`clone0repl.cc`） | 即 `Clone_persist_gtid`，GTID 持久化（详见 [`../server/replication/gtid.md`](../../server/replication/gtid.md)） |

★ **这张清单里"没有"的三类线程**（容易被旧资料误导）：

- **`log_closer` 线程已被移除**——全仓 grep `log_closer_thread` / `m_log_closer` **零命中**，只剩 `log_closer_mutex` / `closer_event` / `SYNC_LOG_CLOSER`。
- **没有独立的死锁检测线程**——它在 `lock_wait_timeout_thread` 里（`lock_wait_update_schedule_and_check_for_deadlocks()`），受 `innodb_deadlock_detect`（默认 ON）控制。
- **也没有独立的 ibuf merge 线程**——只有 ibuf 专用的 AIO handler 线程；ibuf 合并在读盘路径由使用者线程/purge 触发。

#### ★ `Srv_threads`：后台线程句柄的唯一登记处

`storage/innobase/include/srv0srv.h:163-268`——**每个后台线程一个 `IB_thread` 成员**：

```c
/** Structure which keeps shared future objects for InnoDB background
threads. One should use these objects to check if threads exited. */
struct Srv_threads {
  /** Monitor thread (prints info). */
  IB_thread m_monitor;

  /** Error monitor thread. */
  IB_thread m_error_monitor;

  /** Redo files governor thread. */
  IB_thread m_log_files_governor;

  /** Redo checkpointer thread. */
  IB_thread m_log_checkpointer;

  /** Redo writer thread. */
  IB_thread m_log_writer;

  /** Redo flusher thread. */
  IB_thread m_log_flusher;

  /** Redo write notifier thread. */
  IB_thread m_log_write_notifier;

  /** Redo flush notifier thread. */
  IB_thread m_log_flush_notifier;

  /** Redo log archiver (used by backup). */
  IB_thread m_backup_log_archiver;

  /** Buffer pool dump thread. */
  IB_thread m_buf_dump;

  /** Buffer pool resize thread. */
  IB_thread m_buf_resize;

  /** Dict stats background thread. */
  IB_thread m_dict_stats;

  /** Thread detecting lock wait timeouts. */
  IB_thread m_lock_wait_timeout;

  /** The master thread. */
  IB_thread m_master;

  /** The ts_alter_encrypt thread. */
  IB_thread m_ts_alter_encrypt;

  /** Thread doing rollbacks during recovery. */
  IB_thread m_trx_recovery_rollback;

  /** Thread writing recovered pages during recovery. */
  IB_thread m_recv_writer;

  /** Purge coordinator (also being a worker) */
  IB_thread m_purge_coordinator;

  /** Number of purge workers and size of array below. */
  size_t m_purge_workers_n;

  /** Purge workers. Note that the m_purge_workers[0] is the same shared
  state as m_purge_coordinator. */
  IB_thread *m_purge_workers;

  /** Page cleaner coordinator (also being a worker). */
  IB_thread m_page_cleaner_coordinator;

  /** Number of page cleaner workers and size of array below. */
  size_t m_page_cleaner_workers_n;

  /** Page cleaner workers. Note that m_page_cleaner_workers[0] is the
  same shared state as m_page_cleaner_coordinator. */
  IB_thread *m_page_cleaner_workers;

  /** Archiver's log archiver (used by Clone). */
  IB_thread m_log_archiver;

  /** Archiver's page archiver (used by Clone). */
  IB_thread m_page_archiver;

  /** Thread for optimization for FTS index. */
  IB_thread m_fts_optimize;

  /** Thread for GTID persistence */
  IB_thread m_gtid_persister;
  ...
  /** When the master thread notices that shutdown has started (by noticing
  srv_shutdown_state >= SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS), it exits
  its main loop. Then the master thread proceeds with actions related to tasks:
    - which it has been responsible for,
    - and which might depend on DD objects.
  After finishing them, the master thread sets this event.
  @remarks We use this event to wait in srv_pre_dd_shutdown before we enter
  next phase (SRV_SHUTDOWN_PURGE) in which master thread is not allowed to
  use system transactions or touch DD objects. */
  os_event_t m_master_ready_for_dd_shutdown;
};
```

/* 【注】★ 两处注释是重点：① `m_purge_workers[0]` 与 `m_purge_coordinator`
   **共享同一份状态**（coordinator 自己也当一个 worker 用，见 `srv0start.cc` 的赋值）；
   ② `m_master_ready_for_dd_shutdown` 是 master 线程与 shutdown 流程之间的**反向握手**——
   master 要在"还允许碰 DD"的窗口里做完收尾，然后才放行下一阶段。 */

全局单例（`srv0srv.cc:103` / `srv0srv.h:291-292`）：

```c
Srv_threads srv_threads;
```

```c
/** Structure with state of srv background threads. */
extern Srv_threads srv_threads;
```

句柄类型 `IB_thread`（`storage/innobase/include/os0thread.h:46-66`）——**退出原语就在这里**：

```c
class IB_thread {
 public:
  enum class State { INVALID, NOT_STARTED, ALLOWED_TO_START, STARTED, STOPPED };

  State state() const {
    return (m_state == nullptr ? State::INVALID : m_state->load());
  }

  void start();
  void wait(State state_to_wait_for = State::STOPPED);
  void join();

 private:
  std::shared_future<void> m_shared_future;
  std::shared_ptr<std::atomic<State>> m_state;
  ...
};
```

/* 【注】★ `std::shared_future<void>` + 一个 `atomic<State>` ——这就是"线程退出"在 8.0 的
   表达方式：线程函数体跑完 ⇒ future 就绪 ⇒ `wait()`/`join()` 返回。比 5.7 的
   "全局 bool + 手工 cond" 干净得多，也让 shutdown 可以**逐个按名字**等（见「三」的表驱动退出）。 */

#### ★ master 线程为什么在 8.0 变得如此清闲

★ **4.0 时代的"每秒 / 每 10 秒 / background"三段式已经不存在**。8.0 只剩"**1 秒 sleep 轮询 + active/idle 两分支**"：

```cpp
static void srv_master_sleep(void) {
  std::this_thread::sleep_for(std::chrono::seconds(1));   // ★ 1 秒，轮询而非事件驱动
}

static void srv_master_main_loop(srv_slot_t *slot) {
  while (srv_shutdown_state.load() < SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS) {
    srv_master_sleep();
    log_free_check();                          // 检查 redo 空间
    if (srv_check_activity(old_activity_count)) srv_master_do_active_tasks();
    else                                       srv_master_do_idle_tasks();
    fil_purge();                               // Purge any deleted tablespace pages
  }
}
```

`srv_master_do_active_tasks()` 实际只剩六件事：后台 drop table、`ibuf_merge_in_background(false)`、`log_buffer_sync_in_background()`、`srv_update_cpu_usage()`、**唤醒 purge**（`srv_wake_purge_thread_if_not_active()`，自己不 purge）、dict cache LRU 淘汰。

★ 唯一的"间隔任务"有个奇怪的质数间隔——`SRV_MASTER_DICT_LRU_INTERVAL{47}`（47 秒，dict cache LRU 检查）。

**被谁接管的源码证据**：

| 原 master 职责 | 接管者 | 证据 |
|---|---|---|
| **周期性 checkpoint** | `log_checkpointer` 线程 | `srv0start.cc` 注释原文：*"Before 8.0, it was master thread that was doing periodical checkpoints (every 7s). Since 8.0, it is the log checkpointer thread..."* + `log_sys->periodical_checkpoints_enabled = true;` |
| **刷脏** | page cleaner 线程族 | master 主循环里**无任何 `buf_flush_*` 调用**；反证：io 线程的退出条件本身是 `buf_flush_page_cleaner_is_active()`——后台 IO 由 page cleaner 驱动 |
| **purge** | purge coordinator + workers | master 对 purge 的唯一动作是"叫醒别人"，自己不调 `trx_purge()` |

★ **唤醒机制易错**：`srv_master_wakeup_event` **不存在**。master 用 slot 机制里的 `srv_sys->sys_threads[SRV_MASTER_SLOT].event`（master 固定 slot 0），经 `srv_release_threads(SRV_MASTER,1)` / `srv_active_wake_master_thread_low()` 置位。但**主循环阶段它根本不靠事件唤醒**（是 1 秒轮询）；`srv_master_wait()` 只在 `force_recovery >= SRV_FORCE_NO_BACKGROUND` 和 pre-DD shutdown 两个阶段使用。

（purge 线程族的深剖见 [`../innodb/undo_log.md`](../../innodb/undo_log.md)「Purge 线程模型」；io handler 三种等待机制见 [`../innodb/io.md`](../../innodb/io.md)「I/O 线程模型」。）

线程入口（`storage/innobase/srv/srv0srv.cc:2748-2781`）——注意它**不止一个循环**：

```c
/** The master thread controlling the server. */
void srv_master_thread() {
  DBUG_TRACE;

  srv_slot_t *slot;

  THD *thd = create_internal_thd();

  ut_ad(!srv_read_only_mode);

  srv_main_thread_process_no = os_proc_get_number();
  srv_main_thread_id = std::this_thread::get_id();

  slot = srv_reserve_slot(SRV_MASTER);
  ut_a(slot == srv_sys->sys_threads);

  srv_master_main_loop(slot);

  srv_master_pre_dd_shutdown_loop();

  os_event_set(srv_threads.m_master_ready_for_dd_shutdown);

  /* This is just for test scenarios. */
  srv_thread_delay_cleanup_if_needed(true);

  while (srv_shutdown_state.load() < SRV_SHUTDOWN_MASTER_STOP) {
    srv_master_wait(slot);
  }

  srv_master_shutdown_loop();

  srv_main_thread_op_info = "exiting";
  destroy_internal_thd(thd);
}
```

主循环（`srv0srv.cc:2677-2718`）：

```c
/** Executes the main loop of the master thread.
@param[in]   slot     slot reserved as SRV_MASTER */
static void srv_master_main_loop(srv_slot_t *slot) {
  if (srv_force_recovery >= SRV_FORCE_NO_BACKGROUND) {
    /* When innodb_force_recovery is at least SRV_FORCE_NO_BACKGROUND,
    we avoid performing active/idle master's tasks. However, we still
    need to ensure that:
      srv_shutdown_state >= SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS,
    after we exited srv_master_main_loop(). Keep waiting until that
    is satisfied and then exit. */
    while (srv_shutdown_state.load() <
           SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS) {
      srv_master_wait(slot);
    }
    return;
  }

  ulint old_activity_count = srv_get_activity_count();

  while (srv_shutdown_state.load() <
         SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS) {
    srv_master_sleep();

    MONITOR_INC(MONITOR_MASTER_THREAD_SLEEP);

    /* Just in case - if there is not much free space in redo,
    try to avoid asking for troubles because of extra work
    performed in such background thread. */
    srv_main_thread_op_info = "checking free log space";
    log_free_check();

    if (srv_check_activity(old_activity_count)) {
      old_activity_count = srv_get_activity_count();
      srv_master_do_active_tasks();
    } else {
      srv_master_do_idle_tasks();
    }

    /* Purge any deleted tablespace pages. */
    fil_purge();
  }
}
```

节拍与 active/idle 判据（`srv0srv.cc:2657-2661`、`:1994-1998`）：

```c
static void srv_master_sleep(void) {
  srv_main_thread_op_info = "sleeping";
  std::this_thread::sleep_for(std::chrono::seconds(1));
  srv_main_thread_op_info = "";
}
```

```c
bool srv_check_activity(ulint old_activity_count) /*!< in: old activity count */
{
  return (srv_sys == nullptr ? false
                             : srv_sys->activity_count != old_activity_count);
}
```

/* 【注】★ "活跃"的判据极简：**`srv_sys->activity_count` 这一秒内有没有变**。
   它不是一个"当前有多少查询"的计数器，而是一个**累计事件数**——变了就有活动。 */

两个分支的函数头注释（`srv0srv.cc:2304-2312`、`:2377-2383`）：

```c
/** Perform the tasks that the master thread is supposed to do when the
 server is active. There are two types of tasks. The first category is
 of such tasks which are performed at each invocation of this function.
 We assume that this function is called roughly every second when the
 server is active. The second category is of such tasks which are
 performed at some interval e.g.: purge, dict_LRU cleanup etc. */
static void srv_master_do_active_tasks(void) {
```

```c
/** Perform the tasks that the master thread is supposed to do whenever the
 server is idle. We do check for the server state during this function
 and if the server has entered the shutdown phase we may return from
 the function without completing the required tasks.
 Note that the server can move to active state when we are executing this
 function but we don't check for that as we are suppose to perform more
 or less same tasks when server is active. */
static void srv_master_do_idle_tasks(void) {
```

★ **一处需要说准**：说"只剩 1 秒 sleep + active/idle"是对的，但**不是"完全没有间隔任务"**——active 分支里保留了一个 **47 秒**间隔的 dict cache 淘汰：

```c
  if (cur_time - last_dict_lru_check > SRV_MASTER_DICT_LRU_INTERVAL) {
    last_dict_lru_check = cur_time;
    srv_main_thread_op_info = "enforcing dict cache limit";
    ...
    ulint n_evicted = srv_master_evict_from_table_cache(50);
```

```c
constexpr std::chrono::seconds SRV_MASTER_DICT_LRU_INTERVAL{47};
```

/* 【注】active 分支淘汰上限 50、idle 分支 100（idle 时可以更激进），
   这个 50/100 的差别就是"清闲"与"忙"时策略不同的体现。 */

#### 1.2 server 层与其他

| 线程 | 来源 | PFS NAME |
|---|---|---|
| 连接线程 | `handle_connection` | `thread/sql/one_connection` |
| 主接纳线程 | `connection_event_loop()` | `thread/sql/main` |
| signal handler | — | `thread/sql/signal_handler` |
| bootstrap | — | `thread/sql/bootstrap` |
| manager | — | `thread/sql/manager` |
| GTID 压缩 | — | `thread/sql/compress_gtid_table` |
| 复制 | `rpl_replica.cc` | `thread/sql/replica_io` / `replica_sql` / `replica_worker` |
| 事件调度 | `events.cc` | `thread/sql/event_scheduler` / `event_worker` |
| timer | mysys | `thread/mysys/thread_timer_notifier` |
| X Plugin | `plugin/x/` | `KEY_thread_x_acceptor` / `x_worker` |
| manager | `sql/sql_manager.cc` | `thread/sql/manager` ★ **仅当 `flush_time != 0` 才启动**——默认不启动 |
| parser_service | `sql/parser_service.cc` | `thread/sql/parser_service`（组件线程） |
| Source_IO_monitor | `sql/rpl_io_monitor.cc` | ★ 复制：监控 source 连接，按需起停 |
| shutdown_restart | `sql/mysqld.cc` | ★ **Windows 专属**：`handle_shutdown_and_restart`，靠命名 Event（`hEventShutdown` / `hEventRestart`）等外部关闭/重启信号 |
| admin 连接接纳 | `sql/mysqld.cc` | `thread/sql/admin_interface`（`admin_port`；PSI 标志是 `PSI_FLAG_USER` 而非 SINGLETON） |

> ★ **Windows 专属**：`con_named_pipes` / `con_shared_mem` 两个连接接纳线程（命名管道 / 共享内存），Unix 上没有。这是 `thread_handling` 在 Windows 上另有取值的原因之一。

> 上面两张表合计 **InnoDB 约 26 类 + server 约 14 类**。其中**常驻**的只是一部分——另有一批是**按需/临时**线程，见 1.5。

> 本表是**清单级**。两个最重的后台线程族的**线程视角深剖**（创建、主循环睡眠/唤醒、事件接力、背压闭环）分别在：
> - **page cleaner 线程族**（1 协调者 + N-1 worker、slot=实例、1s 循环、`buf_flush_event` 三类唤醒源、LSN 背压闭环的 100µs 忙等落点）→ [`../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)「线程视角」
> - **log 线程族**（6 线程、`log_t` 成员事件、write/flush 分槽事件数组、notifier 跨 slot 接力、两级背压、`innodb_flush_log_at_trx_commit` 三分支）→ [`../innodb/redo_log.md`](../../innodb/redo_log.md)「线程视角」
>
> **server 层常驻后台线程**没有单独成篇（各自体量撑不起一篇，且分属不同功能域），其线程视角深剖就在下面的 **1.3**。

#### 1.2 的三个典型入口（源码）

`signal_hand`（`sql/mysqld.cc:3626-3681`）——**启动顺序有个握手**：

```c
/** This thread handles SIGTERM, SIGQUIT, SIGHUP, SIGUSR1 and SIGUSR2 signals.
 */
/* ARGSUSED */
extern "C" void *signal_hand(void *arg [[maybe_unused]]) {
  my_thread_init();

  sigset_t set;
  (void)sigemptyset(&set);
  (void)sigaddset(&set, SIGTERM);
  (void)sigaddset(&set, SIGQUIT);
  (void)sigaddset(&set, SIGHUP);
  (void)sigaddset(&set, SIGUSR1);
  (void)sigaddset(&set, SIGUSR2);

  /*
    Signal to start_signal_handler that we are ready.
    This works by waiting for start_signal_handler to free mutex,
    after which we signal it that we are ready.
  */
  mysql_mutex_lock(&LOCK_start_signal_handler);
  mysql_cond_broadcast(&COND_start_signal_handler);
  mysql_mutex_unlock(&LOCK_start_signal_handler);

  /*
    Wait until that all server components have been successfully initialized.
    This step is mandatory since signal processing can be done safely only when
    all server components have been initialized.
  */
  server_components_init_wait();
  for (;;) {
    int sig = 0;
    ...
```

创建侧也在等这个握手（`sql/mysqld.cc:3613-3621`）：

```c
  mysql_mutex_lock(&LOCK_start_signal_handler);
  if ((error = mysql_thread_create(key_thread_signal_hand, &signal_thread_id,
                                   &thr_attr, signal_hand, nullptr))) {
    LogErr(ERROR_LEVEL, ER_CANT_CREATE_INTERRUPT_THREAD, error, errno);
    flush_error_log_messages();
    exit(MYSQLD_ABORT_EXIT);
  }
  mysql_cond_wait(&COND_start_signal_handler, &LOCK_start_signal_handler);
  mysql_mutex_unlock(&LOCK_start_signal_handler);
```

/* 【注】★ 为什么要握手：**信号处理必须等所有组件初始化完**才能安全执行
   （否则一个 SIGTERM 可能落在半初始化的状态上）。所以线程先起来、先 `sigaddset` 屏蔽，
   再等 `server_components_init_wait()`。 */

`compress_gtid_table`（`sql/rpl_gtid_persist.cc:732-796`）——典型的"事件等待 + 虚假唤醒保护"：

```c
/**
  The main function of the compression thread.
  - compress the gtid_executed table when get a compression signal.

  @param  p_thd    Thread requesting to compress the table

  @return
      @retval 0    OK. always, the compression thread will swallow any error
                       for going to wait for next compression signal until
                       it is terminated.
*/
extern "C" {
static void *compress_gtid_table(void *p_thd) {
  THD *thd = (THD *)p_thd;
  mysql_thread_set_psi_id(thd->thread_id());
  my_thread_init();
  {
    DBUG_TRACE;

    init_thd(&thd);
    /*
      Gtid table compression thread should ignore 'read-only' and
      'super_read_only' options so that it can update 'mysql.gtid_executed'
      replication repository tables.
    */
    thd->set_skip_readonly_check();

    // Compress the table at server startup
    should_compress = true;

    for (;;) {
      mysql_mutex_lock(&LOCK_compress_gtid_table);
      if (terminate_compress_thread) break;
      THD_ENTER_COND(thd, &COND_compress_gtid_table, &LOCK_compress_gtid_table,
                     &stage_suspending, nullptr);
      /* Add the check to handle spurious wakeups from system. */
      while (!(should_compress || terminate_compress_thread))
        mysql_cond_wait(&COND_compress_gtid_table, &LOCK_compress_gtid_table);
      should_compress = false;
      if (terminate_compress_thread) break;
      mysql_mutex_unlock(&LOCK_compress_gtid_table);
      THD_EXIT_COND(thd, nullptr);

      THD_STAGE_INFO(thd, stage_compressing_gtid_table);
      /* Compressing the gtid_executed table. */
      if (gtid_state->compress(thd)) {
        LogErr(WARNING_LEVEL, ER_FAILED_TO_COMPRESS_GTID_EXECUTED_TABLE);
        /* Clear the error for going to wait for next compression signal. */
        thd->clear_error();
        ...
      }
    }
    ...
```

★ **两个值得记的点**：① `set_skip_readonly_check()`——这个线程**故意绕过 `read_only`/`super_read_only`**，否则只读实例上 GTID 就压不了；② 等待用 `while (!(should_compress || terminate))` 包住 `mysql_cond_wait`，注释明说"处理系统的虚假唤醒"。

`timer_notifier` 线程由 `my_timer_initialize()` 创建、平台相关（`mysys/my_init.cc:572-576` 注册 PSI 名字）：

```c
  #ifdef HAVE_PSI_THREAD_INTERFACE
  static PSI_thread_info all_mysys_thread[] = {
      {&key_thread_timer_notifier, "thread_timer_notifier", "timer_notifier",
       PSI_FLAG_SINGLETON, 0, PSI_DOCUMENT_ME}};
  #endif /* HAVE_PSI_THREAD_INTERFACE */
```

/* 【注】`start_timer_notifier` / `stop_timer_notifier` 这两个符号**不存在**（全仓 0 命中）；
   macOS/BSD 上的实现是 `mysys/kqueue_timers.cc` 的 `timer_notify_thread_func`（`kevent` 循环），
   Linux 上走 `posix_timers.cc`（`SIGEV_THREAD`，由 libc 起通知回调，**不是常驻线程**）。 */

#### 1.3 server 层后台线程的线程视角（深剖）

上表是清单级。这一节补 server 层**常驻后台线程**的执行体视角——谁创建、睡在哪、被谁叫醒、怎么退出。

**① GTID 压缩线程 `compress_gtid_table`（`sql/rpl_gtid_persist.cc`）**

创建（`create_compress_gtid_table_thread`）：

```cpp
if (!(thd = new THD)) { LogErr(ERROR_LEVEL, ER_..._OOM); return; }
thd->set_new_thread_id();
pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);
if (... mysql_thread_create(key_thread_compress_gtid_table,
                            &compress_thread_id, &attr,
                            compress_gtid_table, (void *)thd)) {
  LogErr(ERROR_LEVEL, ER_FAILED_TO_CREATE_GTID_TABLE_COMPRESSION_THREAD, error);
  delete thd;                       /* 创建失败要回收自己 new 的 THD */
}
```

★ 线程参数是 **THD 对象本身**（不是 `arg=null` 再在线程里建）。这一点与连接线程不同——它是 server 层少数"带着 THD 出生"的后台线程。

线程体主循环：

```cpp
thd->set_skip_readonly_check();     /* 只读实例也要能写 mysql.gtid_executed */
should_compress = true;             /* 启动时先压一次 */

for (;;) {
  mysql_mutex_lock(&LOCK_compress_gtid_table);
  if (terminate_compress_thread) break;
  THD_ENTER_COND(thd, &COND_compress_gtid_table, &LOCK_compress_gtid_table,
                 &stage_suspending, nullptr);
  /* Add the check to handle spurious wakeups from system. */
  while (!(should_compress || terminate_compress_thread))
    mysql_cond_wait(&COND_compress_gtid_table, &LOCK_compress_gtid_table);
  should_compress = false;
  if (terminate_compress_thread) break;
  mysql_mutex_unlock(&LOCK_compress_gtid_table);
  THD_EXIT_COND(thd, nullptr);

  THD_STAGE_INFO(thd, stage_compressing_gtid_table);
  if (gtid_state->compress(thd)) {
    LogErr(WARNING_LEVEL, ER_FAILED_TO_COMPRESS_GTID_EXECUTED_TABLE);
    thd->clear_error();             /* ★ 失败只清错误，不退出线程 */
  }
}
```

线程视角的四个要点：

- **纯事件驱动，无周期唤醒** —— 与 page cleaner（1s 循环）、`log_checkpointer`（`innodb_log_checkpoint_every` 定时）不同，它**没有 timeout**，只在被 signal 时压缩
- **`while` 而非 `if`** —— 注释明写 "Add the check to handle spurious wakeups from system"，防虚假唤醒
- **压缩失败不退线程** —— 只 WARNING + `clear_error()` 继续等下一个信号，容错设计
- **`THD_ENTER_COND`/`THD_EXIT_COND`** —— 让 PFS 正确上报 `suspending` / `compressing_gtid_table` 两个 stage（这是 server 层后台线程与 InnoDB 后台线程的关键差别：前者要借 THD 向 PFS 汇报）
- `set_skip_readonly_check()` 是它能写 `mysql.gtid_executed` 的关键——否则在 `read_only`/`super_read_only` 实例上会失败

退出（`terminate_compress_gtid_table_thread`）：

```cpp
mysql_mutex_lock(&LOCK_compress_gtid_table);
terminate_compress_thread = true;
mysql_cond_signal(&COND_compress_gtid_table);      /* 唤醒 */
mysql_mutex_unlock(&LOCK_compress_gtid_table);
if (compress_thread_id.thread != 0) {
  my_thread_join(&compress_thread_id, nullptr);     /* ★ JOINABLE，必须 join */
}
```

> ★ **别和 InnoDB 侧的 GTID 持久化线程搞混**：`Clone_persist_gtid::periodic_write()`（`srv_threads.m_gtid_persister`，见 [`../server/replication/gtid.md`](../../server/replication/gtid.md)「后台线程」）是"**周期 + 事件**"混合（`os_event_wait_time(m_event, s_time_threshold)` 带超时），负责把 GTID **落盘**；本线程是 server 层的，只负责**压缩表**（合并行），纯事件驱动。两者是 GTID 持久化的两条不同路径。

**② signal handler 线程 `signal_hand`（`sql/mysqld.cc`）**

这是全栈里**唯一一个"等待对象是信号而非条件变量"**的线程。

启动前先阻塞信号（`my_init_signals()`）：

```cpp
sigaddset(&mysqld_signal_mask, SIGQUIT);  ... SIGHUP, SIGTERM, SIGTSTP, SIGUSR1, SIGUSR2;
if (!(test_flags & TEST_SIGINT)) sigaddset(&mysqld_signal_mask, SIGINT);
pthread_sigmask(SIG_SETMASK, &mysqld_signal_mask, nullptr);
```

`start_signal_handler()` 的三个关键动作：

```cpp
pthread_attr_setdetachstate(&thr_attr, MY_THREAD_CREATE_JOINABLE);   /* ★ 与连接线程相反 */
pthread_attr_setstacksize(&thr_attr, my_thread_stack_size + guardize);
main_thread_id = my_thread_self();          /* 让 SIGTERM 能中断 socket listener */
mysql_thread_create(key_thread_signal_hand, &signal_thread_id, &thr_attr, signal_hand, nullptr);
mysql_cond_wait(&COND_start_signal_handler, &LOCK_start_signal_handler);  /* ★ 等它就绪 */
```

线程视角要点：

- **JOINABLE 而非 DETACHED** —— 与连接线程（`DETACHED`，复用靠条件变量）相反；shutdown 时要 join
- 主线程 `mysql_cond_wait(&COND_start_signal_handler, ...)` **等线程就绪才继续启动**，避免信号在线程起来前到达
- 采用 POSIX 推荐的**同步信号处理**模式：主线程 block 信号 → 专用线程 `sigwait()`。好处是把异步信号转成同步处理，绕开"信号处理函数里只能调 async-signal-safe 函数"的限制

> shutdown 时三条关闭入口为什么都汇聚到这个线程，见 [`../server/lifecycle/03_shutdown.md`](../../server/lifecycle/03_shutdown.md)「为什么三条关闭入口都要汇聚到信号线程」。

**③ 事件调度器 `Event_scheduler` + `Event_worker_thread`（`sql/event_scheduler.cc`）**

两级结构：调度线程常驻，worker **按需创建、执行完即退**（不是常驻线程池）。

- `Event_scheduler::run`—— 调度者，`mysql_cond_timedwait(&COND_state, &LOCK_scheduler_state, abstime)`
  - ★ **带 `abstime`**：要按"下一个事件的到期时间"醒来。这与 GTID 压缩线程的无限等待形成对照
- `Event_worker_thread::run`—— 每个到期事件起一个 worker 执行

★ 结构上与 page cleaner 的"协调者 + N worker"**同构**，但本质不同：page cleaner 的 worker 是**常驻**的（数量固定 = `innodb_page_cleaners`），事件调度器的 worker 是**临时**的（随事件生灭）。

**④ timer notifier（mysys）**

PFS NAME `thread/mysys/thread_timer_notifier`，服务于 mysys 层的 `my_timer` 超时通知——注意它与线程池的 `TIMER_WORKER_THREAD`（stall checker）**不是同一个东西**，后者见 [`../server/infra/thread_pool.md`](thread_pool.md)。

#### 1.4 ★ 等待模型横向对照：所有后台线程"睡在哪、被谁叫醒"

| 线程 | 等待对象 | 定时唤醒 | 自旋 | 唤醒源 |
|---|---|---|---|---|
| page cleaner 协调者 | `buf_flush_event`（InnoDB `os_event`） | **1s** | — | 三类唤醒源（详见 `buffer_pool.md`） |
| `log_writer` | `log.writer_event` | 10ms | **250000µs**（`INNODB_LOG_WRITER_SPIN_DELAY_DEFAULT`） | 用户线程写 log buffer 后 set |
| `log_flusher` | `log.flusher_event` | 各自 timeout | `srv_log_flusher_spin_delay` | writer 推进 `write_lsn` |
| `log_checkpointer` | `log.checkpointer_event` | ★ `innodb_log_checkpoint_every`（1000ms）；忙时拉长到 `×7` | — | 超时 / 事件 |
| `log_write_notifier` | `log.write_notifier_event` | — | — | writer 推进后 set（★ **批量唤醒**等 write 的用户线程） |
| `log_flush_notifier` | `log.flush_notifier_event` | — | — | flusher 推进后 set |
| purge coordinator | `purge_event`（`rseg_history_len` 变化等） | 有 | — | DML 产生 undo / 超时 |
| **`compress_gtid_table`** | `COND_compress_gtid_table`（server `mysql_cond_t`） | **无** ★ | — | 显式 signal（有压缩请求时） |
| **GTID 持久化（InnoDB）** | `m_event`（`os_event`） | **有** `s_time_threshold` | — | 超时 + 显式 |
| **`signal_hand`** | **`sigwait`（信号）** ★ | — | — | 外部信号（SIGTERM/HUP/QUIT/USR1/USR2） |
| **`Event_scheduler`** | `COND_state` | **有**（下一事件到期时间） | — | 队列变化 / 超时 |
| `lock_wait_timeout` | 各自 | 1s | — | 超时 |
| `srv_error_monitor` | 各自 | **1Hz** | — | 超时（兜底唤醒 sync_array 等待者） |
| `srv_monitor` | 各自 | **15s** | — | 超时 |
| 线程池 `TIMER_WORKER_THREAD` | `tp_COND_stall_check` | **1ms（有动作）/ 10ms（无动作）自适应** | — | 超时 |

**★ 三种等待原语的分层**（跨层看很清晰）：

| 层次 | 原语 | 特点 |
|---|---|---|
| InnoDB | `os_event` / `os_event_wait_time` | 自带超时，不依赖 server THD |
| server | `mysql_cond_t` + mutex | 需配 `THD_ENTER_COND`/`THD_EXIT_COND` 才能向 PFS 上报等待 |
| 信号 | `pthread_sigmask` + `sigwait` | 专用线程，同步化处理 |

**★ 关键结论**：`compress_gtid_table` 是**唯一"纯事件驱动、无超时兜底"的常驻后台线程**——其余全部有周期性唤醒。这意味着一旦 signal 丢失，它会一直睡到 shutdown；而周期性线程（page cleaner / checkpointer / error_monitor）即使事件丢失也能靠超时自我恢复。这是两种截然不同的可靠性取舍。

### 等待原语的三处落点（上表的源码依据）

**① purge coordinator：`os_event_wait_low` + 10 ms 超时**（`storage/innobase/srv/srv0srv.cc:2984-3024`）：

```c
/** Suspend the purge coordinator thread. */
static void srv_purge_coordinator_suspend(
    srv_slot_t *slot,       /*!< in/out: Purge coordinator
                            thread slot */
    ulint rseg_history_len) /*!< in: history list length
                            before last purge */
{
  ut_ad(!srv_read_only_mode);
  ut_a(slot->type == SRV_PURGE);

  bool stop = false;

  /** Maximum wait time on the purge event. */
  constexpr std::chrono::milliseconds SRV_PURGE_MAX_TIMEOUT{10};

  int64_t sig_count = srv_suspend_thread(slot);

  do {
    ulint ret;

    rw_lock_x_lock(&purge_sys->latch, UT_LOCATION_HERE);

    purge_sys->running = false;

    rw_lock_x_unlock(&purge_sys->latch);

    /* We don't wait right away on the the non-timed wait because
    we want to signal the thread that wants to suspend purge. */

    if (stop) {
      os_event_wait_low(slot->event, sig_count);
      ret = 0;
    } else if (rseg_history_len <= trx_sys->rseg_history_len.load()) {
      ret =
          os_event_wait_time_low(slot->event, SRV_PURGE_MAX_TIMEOUT, sig_count);
    } else {
      /* We don't want to waste time waiting, if the
      history list increased by the time we got here,
      unless purge has been stopped. */
      ret = 0;
    }
    ...
```

/* 【注】★ 三种睡法：被要求停 → **无限等**；history list 没变长 → 等 10 ms；
   history list 变长了 → **根本不睡**（`ret = 0` 立刻再干一轮）。这是"有事就干、没事小睡"的节流。 */

`os_event_wait_low` 的核心循环与"丢信号"的经典说明（`storage/innobase/os/os0event.cc:358-373` + `include/os0event.h:80-104`）：

```c
void os_event::wait_low(int64_t reset_sig_count) UNIV_NOTHROW {
  mutex.enter();

  if (!reset_sig_count) {
    reset_sig_count = signal_count;
  }

  while (!m_set && signal_count == reset_sig_count) {
    wait();

    /* Spurious wakeups may occur: we have to check if the
    event really has been signaled after we came here to wait. */
  }

  mutex.exit();
}
```

```c
/**
Waits for an event object until it is in the signaled state.

Typically, if the event has been signalled after the os_event_reset()
we'll return immediately because event->is_set == true.
There are, however, situations (e.g.: sync_array code) where we may
lose this information. For example:

thread A calls os_event_reset()
thread B calls os_event_set()   [event->is_set == true]
thread C calls os_event_reset() [event->is_set == false]
thread A calls os_event_wait()  [infinite wait!]
thread C calls os_event_wait()  [infinite wait!]

Where such a scenario is possible, to avoid infinite wait, the
value returned by os_event_reset() should be passed in as
reset_sig_count. */
```

/* 【注】★ 这段注释就是"防丢信号"的权威表述：**`signal_count` 单调递增**，
   把 `os_event_reset()` 的返回值传回 `wait_low()` 即可避免"reset 之后才来的 signal 被丢掉"。 */

**② lock_wait 线程：1 秒节拍**（`storage/innobase/lock/lock0wait.cc:1435-1471`）：

```c
/** A thread which wakes up threads whose lock wait may have lasted too long,
analyzes wait-for-graph changes, checks for deadlocks and resolves them, and
updates schedule weights. */
void lock_wait_timeout_thread() {
  int64_t sig_count = 0;
  os_event_t event = lock_sys->timeout_event;

  ut_ad(!srv_read_only_mode);

  /** The last time we've checked for timeouts. */
  auto last_checked_for_timeouts_at = std::chrono::steady_clock::now();
  do {
    auto current_time = std::chrono::steady_clock::now(); /* Calling this more
    often than once a second isn't needed, as lock timeouts are specified with
    one second resolution, so probably nobody cares if we wake up after T or
    T+0.99, when T itself can't be precise. */
    if (std::chrono::seconds(1) <=
        current_time - last_checked_for_timeouts_at) {
      last_checked_for_timeouts_at = current_time;
      lock_wait_check_slots_for_timeouts();
    }

    lock_wait_update_schedule_and_check_for_deadlocks();

    /* When someone is waiting for a lock, we wake up every second (at worst)
    and check if a timeout has passed for a lock wait */
    os_event_wait_time_low(event, std::chrono::seconds{1}, sig_count);
    sig_count = os_event_reset(event);

  } while (srv_shutdown_state.load() < SRV_SHUTDOWN_CLEANUP);
}
```

/* 【注】"1 秒"不是硬定时器，而是 **1 秒超时 + signal 唤醒**的混合：没人等锁时也每秒被唤醒一次，
   有锁等待/死锁检测需求时由 `lock_set_timeout_event` 立刻唤醒。 */

**③ page_cleaner：无限等 + 事件驱动**（`storage/innobase/buf/buf0flu.cc:3606-3621`、`:3027-3041`）：

```c
  /** Worker thread of page_cleaner. */
  static void buf_flush_page_cleaner_thread() {
#ifdef UNIV_LINUX
    /* linux might be able to set different setting for each thread
    worth to try to set high priority for page cleaner threads */
    if (buf_flush_page_cleaner_set_priority(buf_flush_page_cleaner_priority)) {
      ib::info(ER_IB_MSG_129)
          << "page_cleaner worker priority: " << buf_flush_page_cleaner_priority;
    }
#endif /* UNIV_LINUX */

    for (;;) {
      os_event_wait(page_cleaner->is_requested);
```

```c
  static bool pc_wait_finished(ulint *n_flushed_lru, ulint *n_flushed_list) {
    bool all_succeeded = true;

    *n_flushed_lru = 0;
    *n_flushed_list = 0;

    os_event_wait(page_cleaner->is_finished);
```

/* 【注】★ 与 purge 不同：page cleaner 用 **`os_event_wait()`（无限阻塞）**，
   完全由 `is_requested` / `is_finished` 事件驱动，没有轮询。
   另注：`os_event_wait_for`（自旋 + 指数退避）**不是** page cleaner 用的，
   它的实际使用点在 redo log 侧（`log0write.cc:872`、`:932`、`:1258`）。 */

#### 1.5 ★ 按需 / 临时线程的线程视角

上面两张表里有一批线程**不是常驻的**——只在特定场景出现，用完即退。这类线程最容易漏数，但"什么时候突然多出一批线程"恰恰是排查线程数暴涨的关键。

**① 启动期 fork-join：`parallel_rseg_init_thread`（`trx0rseg.cc`）**

```cpp
for (uint32_t i = 0; i < srv_rseg_init_threads; i++) {
  auto thread = os_thread_create(parallel_rseg_init_thread_key, 0,
                                 trx_rseg_init_thread, (void *)purge_queue, gtid_trx_no);
  threads.emplace_back(thread);
  thread.start();
}
for (auto t : threads) t.join();          /* ★ 起完立刻 join */
ib::info(ER_IB_MSG_PAR_RSEG_INIT_COMPLETE_MSG);
trx_rsegs_init_end();
```

- 教科书式 **fork-join**：起 N 个线程并行初始化 rollback segment，主线程立刻 `join()` 等全部完成
- 触发条件：`if (purge_sys->rsegs_queue.empty()) return;` —— 没有待 purge 的 rseg 就根本不起
- ★ 与 GTID 持久化有耦合：先 `gtid_persistor.set_oldest_trx_no_recovery(gtid_trx_no)`，再并行初始化
- ★ **生命周期极短**：启动阶段结束即销毁，**不会出现在运行期的线程列表里**

**② 查询期 fork-join：`parallel_read_thread`（`row0pread.cc`）**

```cpp
auto ptr = ut::new_withkey<Thread_ctx>(UT_NEW_THIS_FILE_PSI_KEY, i);
m_thread_ctxs.emplace_back(ptr);
m_parallel_read_threads.emplace_back(
    os_thread_create(parallel_read_thread_key, i + 1,
                     &Parallel_reader::worker, this, m_thread_ctxs[i]));
m_parallel_read_threads.back().start();
```

★ 两个容易忽略的设计：

- **线程数为 0 时完全不起线程**：
```cpp
if (m_n_threads == 0) {
  /* Set event to indicate to ::worker() that no threads will be spawned. */
  os_event_set(m_event);
  worker(m_thread_ctxs[0]);      /* ★ 由调用线程自己执行 */
  return;
}
```
即 `innodb_parallel_read_threads` 为 0（或单线程场景）时，**用户线程自己**跑 `worker()`，零线程创建开销。
- **错误传播用全局错误状态**而非逐个通知：`set_error_state(err)` —— 注释原话 "Set the global error state to tell the worker threads to exit"。

> 这个线程族是"**查询结束后线程就没了**"的典型——所以 `SHOW ENGINE INNODB STATUS` 或 `performance_schema.threads` 里偶发出现的 `ib_pread` 线程不必惊慌。

**③ 上次崩溃遗留的收尾：`fsp_init_resume_alter_encrypt_tablespace`**

只在 `!ts_encrypt_ddl_records.empty()` 时启动（即上次有未完成的加/解密）：

```cpp
srv_threads.m_ts_alter_encrypt = os_thread_create(
    srv_ts_alter_encrypt_thread_key, 0, fsp_init_resume_alter_encrypt_tablespace);
mysql_mutex_lock(&resume_encryption_cond_m);
srv_threads.m_ts_alter_encrypt.start();
/* Wait till shared MDL is taken by background thread for all tablespaces,
   for which (un)encryption is to be rolled forward. */
mysql_cond_wait(&resume_encryption_cond, &resume_encryption_cond_m);
mysql_mutex_unlock(&resume_encryption_cond_m);
```

★ **主线程要 cond_wait 等它拿到所有 shared MDL 才继续启动**——防止后续正常 DDL 与"恢复中的加解密"撞锁。这也是"创建后等就绪"模式的第三次出现。

**④ 恢复期两个**：`recv_writer_thread`（`log0recv.cc`，写恢复页）、`trx_recovery_rollback`（回滚未提交事务）——仅崩溃恢复时出现，恢复完即退。

#### 1.6 ★ server 层其余常驻线程的线程视角

**① manager 线程 `handle_manager`（`sql/sql_manager.cc`）**

★ **默认不启动** —— 这是全栈唯一一个"由系统变量决定是否存在"的常驻线程：

```cpp
void start_handle_manager() {
  abort_manager = false;
  if (flush_time && flush_time != ~(ulong)0L) {        /* ★ flush_time 非 0 才建 */
    if (mysql_thread_create(key_thread_handle_manager, &hThread,
                            &connection_attrib, handle_manager, nullptr))
      LogErr(WARNING_LEVEL, ER_CANT_CREATE_HANDLE_MGR_THREAD, error);
  }
}
```

主循环**双分支**（有无 `flush_time` 决定用哪种等待）：

```cpp
for (;;) {
  mysql_mutex_lock(&LOCK_manager);
  if (flush_time) {
    if (reset_flush_time) { set_timespec(&abstime, flush_time); reset_flush_time = false; }
    while ((!error || error == EINTR) && !abort_manager)
      error = mysql_cond_timedwait(&COND_manager, &LOCK_manager, &abstime);
  } else {
    while ((!error || error == EINTR) && !abort_manager)
      error = mysql_cond_wait(&COND_manager, &LOCK_manager);
  }
  mysql_mutex_unlock(&LOCK_manager);
  if (abort_manager) break;
  if (is_timeout(error)) {
    tdc_flush_unused_tables();      /* ★ 超时后刷掉未使用的 table cache */
    error = 0;
    reset_flush_time = true;
  }
}
```

职责单一：周期性 `tdc_flush_unused_tables()`。★ 由于默认 `flush_time = 0`，**绝大多数生产实例根本没有这个线程**——这正是它容易被误认为"存在"的原因。

**② `Source_IO_monitor`（`sql/rpl_io_monitor.cc`）**

复制侧监控 source 连接的线程：

```cpp
if (mysql_thread_create(thread_key, &m_th, &connection_attrib,
                        launch_handler_thread, (void *)this)) { ... }
m_monitor_thd_state.set_created();
while (m_monitor_thd_state.is_alive_not_running()) {
  DBUG_PRINT("sleep", ("Waiting for the Monitoring process thread to start"));
  mysql_cond_wait(&m_run_cond, &m_run_lock);       /* ★ 创建后等就绪 */
}
```

- `source_monitor_handler()` 里 `new THD` 并 `thd->thread_stack = (char *)&thd`（与 event_scheduler 的 THD 自持模式一致）
- 退出判定三条件：`m_abort_monitor || connection_events_loop_aborted() || thd->killed`

**③ `parser_service` 与 `shutdown_restart`**

- `parser_service`（`sql/parser_service.cc`）：组件化 parser 服务的线程
- `shutdown_restart`（`sql/mysqld.cc`）：★ **仅 Windows**——`handle_shutdown_and_restart()`，靠 `CreateEvent` 出来的命名事件 `hEventShutdown` / `hEventRestart` 等待外部（如服务管理器）发来的关闭/重启信号。Unix 上没有这个线程，Unix 的关闭走 `signal_hand`（这也解释了为什么 Windows 与 Unix 的关闭入口设计不同）

### 二、InnoDB 线程如何创建：8.0 已用 `std::thread`

```cpp
// storage/innobase/include/os0thread.h
class IB_thread {
 public:
  enum class State { INVALID, NOT_STARTED, ALLOWED_TO_START, STARTED, STOPPED };
  State state() const { return (m_state == nullptr ? State::INVALID : m_state->load()); }
  void start();
  void wait(State state_to_wait_for = State::STOPPED);
  void join();
 private:
  std::shared_future<void> m_shared_future;
  std::shared_ptr<std::atomic<State>> m_state;
};
```

★ `os_thread_create` 是**宏**，且默认 **not started**，必须显式 `.start()`：

```cpp
template <typename F, typename... Args>
IB_thread create_detached_thread(mysql_pfs_key_t pfs_key,
                                 PSI_thread_seqnum pfs_seqnum, F &&f, Args &&... args) {
  Detached_thread detached_thread{pfs_key, pfs_seqnum};
  auto thread = detached_thread.thread();
  std::thread t(std::move(detached_thread), f, args...);
  t.detach();
  ut_a(thread.state() == IB_thread::State::NOT_STARTED);
  return thread;
}

#ifdef UNIV_PFS_THREAD
#define os_thread_create(...) create_detached_thread(__VA_ARGS__)
#endif
```

★ **为什么要"创建与放行分离"**：避免线程在 `IB_thread` 句柄赋值到 `srv_threads` 之前就跑起来——否则 shutdown 可能看不到它。等待 `start()` 用的是**忙等** `UT_RELAX_CPU()`，不是条件变量：

```cpp
  void operator()(F &&f, Args &&... args) {
    while (m_thread.state() == IB_thread::State::NOT_STARTED) { UT_RELAX_CPU(); }
    ut_a(m_thread.state() == IB_thread::State::ALLOWED_TO_START);
    preamble();
    m_thread.set_state(IB_thread::State::STARTED);
    std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    epilogue();
    m_thread.set_state(IB_thread::State::STOPPED);
  }
```

`preamble()` / `epilogue()` 负责 PFS 登记与线程计数：

```cpp
  void preamble() {
    MySQL_thread::preamble();
    auto old = os_thread_count.fetch_add(1, std::memory_order_relaxed);
    ut_a(old <= static_cast<int>(srv_max_n_threads) - 1);
  }
```

★ `srv_max_n_threads` 是**硬编码 `100 * 1024`**（不是系统变量），同时决定 sync array 与行锁等待表的大小。

### 三、★ shutdown 九阶段：为什么是这个顺序

```cpp
// storage/innobase/include/srv0shutdown.h
enum srv_shutdown_t {
  SRV_SHUTDOWN_NONE = 0,
  SRV_SHUTDOWN_RECOVERY_ROLLBACK,        // 1
  SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS,  // 2
  SRV_SHUTDOWN_PURGE,                    // 3
  SRV_SHUTDOWN_DD,                       // 4
  SRV_SHUTDOWN_CLEANUP,                  // 5
  SRV_SHUTDOWN_MASTER_STOP,              // 6
  SRV_SHUTDOWN_FLUSH_PHASE,              // 7
  SRV_SHUTDOWN_LAST_PHASE,               // 8
  SRV_SHUTDOWN_EXIT_THREADS              // 9
};
```

状态只能**逐级 +1**（`srv_shutdown_set_state` 有 `ut_a(old + 1 == new)`）。停止清单由一张表驱动：

```cpp
struct Thread_to_stop {
  const char *m_name;                 // 日志里打印
  const IB_thread &m_thread;
  std::function<void()> m_notify;     // 可重复调用去唤醒它
  srv_shutdown_t m_wait_on_state;     // 在哪个阶段等它
};

static const Thread_to_stop threads_to_stop[]{
    {"lock_wait_timeout", srv_threads.m_lock_wait_timeout, lock_set_timeout_event, SRV_SHUTDOWN_CLEANUP},
    {"error_monitor",  srv_threads.m_error_monitor,  [](){ os_event_set(srv_error_event); },   SRV_SHUTDOWN_CLEANUP},
    {"monitor",        srv_threads.m_monitor,        [](){ os_event_set(srv_monitor_event); }, SRV_SHUTDOWN_CLEANUP},
    {"buf_dump",       srv_threads.m_buf_dump,       [](){ os_event_set(srv_buf_dump_event); },SRV_SHUTDOWN_CLEANUP},
    {"buf_resize",     srv_threads.m_buf_resize,     [](){ os_event_set(srv_buf_resize_event);},SRV_SHUTDOWN_CLEANUP},
    {"master",         srv_threads.m_master,         srv_wake_master_thread, SRV_SHUTDOWN_MASTER_STOP}};
```

**为什么是这个顺序**（源码注释给出的理由，逐条）：

1. **用户事务最先死**——server 层 `close_connections()` 在 `SRV_SHUTDOWN_NONE` 阶段就已完成（InnoDB 会检查 `srv_conc_get_active_threads() == 0`，否则 **crash**）。
2. `trx_recovery_rollback` 先停——它会**产生**新的回滚工作和事务对象。
3. `dict_stats` / `fts_optimize` / `ts_alter_encrypt` 会**访问 DD 对象或用系统事务** ⇒ 必须在 `PRE_DD_AND_SYSTEM_TRANSACTIONS` 停。
4. master thread 退出主循环，之后不再允许碰 DD。
5. ★ **purge 必须在"保证不会再产生新 undo"之后才停** ⇒ `SRV_SHUTDOWN_PURGE`。
6. DD 关闭 ⇒ `SRV_SHUTDOWN_DD`。
7. 剩余纯后台线程（lock_wait_timeout / monitor / error_monitor / buf_dump / buf_resize）⇒ `CLEANUP`。
8. master thread 最后停（还负责收尾）⇒ `MASTER_STOP`。
9. ★ **page cleaner 必须在 redo 线程之前停**——page cleaner 刷盘会产生 redo，需要 log_writer/flusher 干活 ⇒ `FLUSH_PHASE`。
10. redo 线程写 checkpoint、更新 header lsn ⇒ `LAST_PHASE`。

源码里还有一句**强制性的官方警告**：

```cpp
    /* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM
    HERE OR EARLIER */
```

以及 redo 线程的"保命"启发式（注释原文：*"The redo log is like a blood, we need it for a lot of other systems to work. Ensure the blood flows."*）：

```cpp
      if (!buf_flush_page_cleaner_is_active() || i >= SHUTDOWN_SLEEP_ROUNDS * 0.75) {
        log_stop_background_threads_nowait(*log_sys);
      } else {
        log_wake_threads(*log_sys);   // 保留 75% 时间，先让 log 线程帮别的线程收尾
      }
```

**其他顺序约束**：purge worker 必须在 coordinator **之后**退出（否则 coordinator 会永远等 `trx_purge_wait_for_workers_to_complete()`）。

### ★ 后台线程的退出是表驱动的（`threads_to_stop[]`）

`storage/innobase/srv/srv0start.cc:1324-1456`——**每个后台线程登记"名字 / 句柄 / 怎么唤醒它 / 在哪个阶段等它退出"**：

```c
struct Thread_to_stop {
  /** Name of the thread, printed to the error log if we waited too
  long (after 60 seconds and then every 60 seconds). */
  const char *m_name;

  /** Future which allows to check if given task is completed. */
  const IB_thread &m_thread;

  /** Function which can be called any number of times to wake
  the possibly waiting thread, so it could exit. */
  std::function<void()> m_notify;

  /** Shutdown state in which we are waiting until thread is exited
  (earlier we keep notifying but we don't require it to exit before
  we may switch to the next state). */
  srv_shutdown_t m_wait_on_state;
};

static const Thread_to_stop threads_to_stop[]{
    {"lock_wait_timeout", srv_threads.m_lock_wait_timeout,
     lock_set_timeout_event, SRV_SHUTDOWN_CLEANUP},

    {"error_monitor", srv_threads.m_error_monitor,
     []() { os_event_set(srv_error_event); }, SRV_SHUTDOWN_CLEANUP},

    {"monitor", srv_threads.m_monitor,
     []() { os_event_set(srv_monitor_event); }, SRV_SHUTDOWN_CLEANUP},

    {"buf_dump", srv_threads.m_buf_dump,
     []() { os_event_set(srv_buf_dump_event); }, SRV_SHUTDOWN_CLEANUP},

    {"buf_resize", srv_threads.m_buf_resize,
     []() { os_event_set(srv_buf_resize_event); }, SRV_SHUTDOWN_CLEANUP},

    {"master", srv_threads.m_master, srv_wake_master_thread,
     SRV_SHUTDOWN_MASTER_STOP}};

void srv_shutdown_exit_threads() {
  srv_shutdown_state.store(SRV_SHUTDOWN_EXIT_THREADS);

  if (srv_start_state == SRV_START_STATE_NONE) {
    return;
  }

  uint32_t i;

  /* All threads end up waiting for certain events. Put those events
  to the signaled state. Then the threads will exit themselves after
  os_event_wait(). */
  for (i = 0; i < SHUTDOWN_SLEEP_ROUNDS; i++) {
    /* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM
    HERE OR EARLIER */

    /* These threads normally finish when reaching SRV_SHUTDOWN_CLEANUP or
    SRV_SHUTDOWN_MASTER_STOP state, which we might have jumped over. */
    for (const auto &thread_info : threads_to_stop) {
      if (srv_thread_is_active(thread_info.m_thread)) {
        thread_info.m_notify();
      }
    }

    if (!srv_read_only_mode) {
      if (srv_start_state_is_set(SRV_START_STATE_PURGE)) {
        /* Wakeup purge threads. */
        srv_purge_wakeup();
      }
    }

    if (srv_start_state_is_set(SRV_START_STATE_IO)) {
      /* Exit the i/o threads */
      if (!srv_read_only_mode) {
        if (recv_sys->flush_start != nullptr) {
          os_event_set(recv_sys->flush_start);
        }
        if (recv_sys->flush_end != nullptr) {
          os_event_set(recv_sys->flush_end);
        }
      }

      os_event_set(buf_flush_event);

      if (!buf_flush_page_cleaner_is_active() && os_aio_all_slots_free()) {
        os_aio_wake_all_threads_at_shutdown();
      }
    }

    if (srv_thread_is_active(srv_threads.m_dict_stats)) {
      os_event_set(dict_stats_event);
    }

    /* Try to stop archiver threads. */
    arch_wake_threads();

    if (log_sys != nullptr) {
      /* Preserve the log threads for the 75% of the total
      time we are waiting here until all threads are stopped.
      This is because log threads are normally shut down at
      the very end and we might need their help to stop other
      threads. */
      if (!buf_flush_page_cleaner_is_active() ||
          i >= SHUTDOWN_SLEEP_ROUNDS * 0.75) {
        log_stop_background_threads_nowait(*log_sys);

      } else {
        /* Ensure log threads are working. The redo log is
        like a blood, we need it for a lot of other systems
        to work. Ensure the blood flows. */
        log_wake_threads(*log_sys);
      }
    }

    bool active = os_thread_any_active();

    std::this_thread::sleep_for(
        std::chrono::microseconds(SHUTDOWN_SLEEP_TIME_US));

    if (!active) {
      break;
    }
  }

  if (i == SHUTDOWN_SLEEP_ROUNDS) {
    ib::warn(ER_IB_MSG_1103, os_thread_count.load());

#ifdef UNIV_DEBUG
    os_aio_print_pending_io(stderr);
    ut_d(ut_error);
#endif /* UNIV_DEBUG */
  } else {
    /* Reset the start state. */
    srv_start_state = SRV_START_STATE_NONE;
  }
}
```

/* 【注】★ 三处设计值得记：
   ① **退出不是"发一个全局信号"，而是"每个线程一个 notify 函数"**（表驱动）——所以新加线程必须登记，
      这就是那句 `/* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM HERE OR EARLIER */`
      的真实语境：它在该循环的第一行，提醒你补 `threads_to_stop[]`。
   ② **redo log 线程被特意留到 75% 的时间之后才停**——注释说得很形象：
      "The redo log is like a blood, we need it for a lot of other systems to work."
      别的线程收尾时还要写 redo，所以 log 线程必须最后才停。
   ③ 超时后 **debug 构建直接 `ut_error` 崩溃**，release 只是 warn（`ER_IB_MSG_1103`）。 */

真正的阻塞式等待在 `srv_pre_dd_shutdown()`（`srv0start.cc:2683-2715`）——恢复回滚线程用 `wait()`，master 线程用事件握手：

```c
  if (srv_thread_is_active(srv_threads.m_trx_recovery_rollback)) {
    /* We should wait until rollback after recovery end to avoid
    adding more for purge and to avoid touching transaction objects
    since this point. */
    srv_threads.m_trx_recovery_rollback.wait();
  }
  ...
  /* Wait until the master thread exits its main loop and notices that:
    - it should do shutdown-cleanup,
    - and still is allowed to access DD objects. */
  if (srv_thread_is_active(srv_threads.m_master)) {
    srv_wake_master_thread();
    os_event_wait(srv_threads.m_master_ready_for_dd_shutdown);
  }

  /* Since this point we do not expect accesses to DD coming from InnoDB. */
  ut_d(trx_sys_before_pre_dd_shutdown_validate());
```

### 四、★ 线程 × 并发控制：漏斗与三种等待挂靠

#### 4.1 漏斗图

```
客户端连接请求
      │
┌─────▼──────────────────────────────┐
│ 闸门 0  max_connections（默认 151）  │  ← server：拒绝新连接
│ check_and_incr_conn_count()         │
└─────┬──────────────────────────────┘
┌─────▼──────────────────────────────┐
│ 闸门 0.5  thread_cache_size（默认 9）│  ← 复用 pthread，不复用 THD
└─────┬──────────────────────────────┘
┌─────▼──────────────────────────────┐
│ 闸门 1  innodb_thread_concurrency   │  ← InnoDB：默认 0 = 关闭！
│ （默认 0，tickets 默认 5000）        │    打开时 FIFO + sleep_for 轮询
└─────┬──────────────────────────────┘
┌─────▼──────────────────────────────┐
│ 闸门 2  latch / rw_lock             │  ← sync_array（102400 cell）
└─────┬──────────────────────────────┘
┌─────▼──────────────────────────────┐
│ 闸门 3  行锁 / 表锁（lock_sys）      │  ← srv_slot（102400 slot）
└─────┬──────────────────────────────┘
┌─────▼──────────────────────────────┐
│ 闸门 4  物理资源（buf pool / redo / IO）│
└────────────────────────────────────┘
```

#### 4.2 闸门 1 的实现：ticket + 自适应 sleep（**不是条件变量**）

```cpp
struct srv_conc_t {
  char pad[ut::INNODB_CACHE_LINE_SIZE];      // cacheline 对齐，防 false sharing
  std::atomic<int32_t> n_active;             // 已在 InnoDB 里的事务数
  std::atomic<int32_t> n_waiting;            // 在门口排队的 OS 线程数
};

static dberr_t srv_conc_enter_innodb_with_atomics(trx_t *trx) {
  for (;;) {
    if (srv_thread_concurrency == 0) return DB_SUCCESS;        // ★ 闸门关闭

    if (srv_conc.n_active.load(...) < (int32_t)srv_thread_concurrency) {
      const auto n_active = srv_conc.n_active.fetch_add(1, ...) + 1;
      if (n_active <= (int32_t)srv_thread_concurrency) {
        srv_enter_innodb_with_tickets(trx);   // ★ 发 5000 张票
        ...
        return DB_SUCCESS;
      }
      srv_conc.n_active.fetch_sub(1, ...);    // 超卖了，退票
    }
    thd_wait_begin(trx->mysql_thd, THD_WAIT_USER_LOCK);
    std::this_thread::sleep_for(std::chrono::microseconds(srv_thread_sleep_delay));
    if (srv_adaptive_max_sleep_delay > 0 && n_sleeps > 1) ++srv_thread_sleep_delay;  // 自适应退避
    ...
  }
}
```

★★ **为什么"等锁线程不计数"**（源码注释原文）：

> *"The following controls how many threads we let inside InnoDB concurrently: **threads waiting for locks are not counted into the number because otherwise we could get a deadlock.**"*

对应的实现在 `lock_wait_suspend_thread()`——等锁前**先把票还掉**，让持锁者能进来放锁：

```cpp
  auto was_declared_inside_innodb = trx->declared_to_be_inside_innodb;
  if (was_declared_inside_innodb) {
    /* We must declare this OS thread to exit InnoDB, since a
    possible other thread holding a lock which this thread waits
    for must be allowed to enter, sooner or later */
    srv_conc_force_exit_innodb(trx);        // ★ 防"票被等待者占满 → 自锁"
  }
  os_event_wait(slot->event);
  if (was_declared_inside_innodb) {
    srv_conc_force_enter_innodb(trx);       // 回来只给 1 张票
  }
```

#### 4.3 ★ 三种等待挂靠结构对比

| 层 | 挂靠结构 | 数量 | 超时 | 唤醒者 | 源码位置 |
|---|---|---|---|---|---|
| **MDL** | `MDL_wait::m_COND_wait_status`（mutex + cond） | 每 THD 1 个 | `lock_wait_timeout`（**默认 31536000 秒 = 1 年**） | 持锁者 `MDL_wait::set_status()` / kill / 超时 | `sql/mdl.cc` |
| **InnoDB 行锁/表锁** | `lock_sys->waiting_threads[]` 的 `srv_slot_t` | **`srv_max_n_threads` = 102400** | `innodb_lock_wait_timeout`（**50s**） | `lock_wait_timeout_thread` / 放锁者 / 死锁 victim | `lock/lock0wait.cc` |
| **InnoDB latch/rw_lock** | `sync_array_t` 的 `sync_cell_t` | **102400（切分 N 个数组）** | **无**（只有 `innodb_fatal_semaphore_wait_threshold` 600s，且是 **debug-only 变量**） | `mutex_exit()` → `os_event_set()`；`srv_error_monitor_thread` 每秒兜底 | `sync/sync0arr.cc` |

★ **一个容易踩的坑**：`innodb_thread_concurrency` 设太大有 redo 安全隐患，源码会直接打 warning：

```cpp
  if (!log_sys->concurrency_margin_is_safe.load()) {
    push_warning_printf(thd, ..., "Current innodb_thread_concurrency"
                        " is too big for safety of redo log files."
                        " Consider decreasing it or increasing innodb_redo_log_capacity.");
  }
```

#### 4.4 ★ 闸门 2/3 的内部：InnoDB 把"中心锁"拆成了什么

漏斗图里的闸门 2（latch）与闸门 3（行锁）在 8.0 里**都不是单一全局锁**，而是各自分片。这是"多核扩展"的实际承担者，也是理解为什么 `innodb_thread_concurrency` 默认关掉仍有意义的前提。

| 结构 | 8.0.39 的真实并发控制 | 证据 |
|---|---|---|
| **lock_sys（行锁/表锁队列）** | ★ **512 个 page shard + 512 个 table shard**（`lock0latches.h` 的 `SHARDS_COUNT = 512`），各自一个 `rw_lock` | `lock_sys->latches.page_shards.get_mutex(page_id)`（`lock0guards.cc`） |
| **buffer pool** | **按 instance 分片**：每个实例各有 `LRU_list_mutex` / `flush_list_mutex` / `free_list_mutex` / `zip_free_mutex` | `buf0buf.h`（每个字段注释都写明受哪个 mutex 保护） |
| **trx_sys** | `trx_sys_t::mutex` + `serialisation_mutex`（提交序）分工 | `trx0sys.h`（按成员分组注明保护者） |
| **redo（log_t）** | ★ **无锁化**：`sn` 用最高位做锁位的原子变量 + `Link_buf` 环形缓冲，`recent_written` / `recent_closed` | 见 [`../innodb/redo_log.md`](../../innodb/redo_log.md)「8.0 无锁化并发模型」 |

★ **`lock0latches.h` 的文件头注释是整个 InnoDB 并发设计里最值得读的一段**，它把"分片后如何避免死锁"写成了 5 条规则，其中第 5 条是关键：

> *"5. latch mutexes for the two shards in the order of their addresses"*

即：一次操作要同时拿两个 shard 时，**按地址顺序加锁**——这是教科书式的"有序分配防死锁"，也是为什么它能省掉全局 `lock_sys->mutex`（5.7 时代的热点）。配套的守卫类是 `lock0guards.cc` 里的 `locksys::Shard_latches_guard`（单 shard）与 `...::Page_shards_latches_guard`（双 shard）。

⇒ **这也解释了官方为什么不建议开 `innodb_thread_concurrency`**：真正的中心瓶颈（全局 `lock_sys` mutex、全局 buf pool mutex）在 8.0 已被分片拆掉，靠"限制进入 InnoDB 的线程数"来缓解争用，已经是在解决一个**不存在的问题**——反而会引入 ticket 轮转与 sleep 轮询的开销。

#### 4.5 官方默认值 vs 源码默认值（★ 附 8.4 的变化）

| 变量 | 官方 8.0 文档 | 源码核实 | 8.4 的变化（官方 nutshell 表） |
|---|---|---|---|
| `innodb_thread_concurrency` | 默认 **0**（= 不限）；多数负载无需限制 | 一致 | 未变 |
| `innodb_concurrency_tickets` | 默认 **5000** | 一致 | 未变 |
| `innodb_thread_sleep_delay` | 文档页存在，**默认值未逐字核实** | — | 未变 |
| `innodb_adaptive_max_sleep_delay` | 默认 **150000** µs（0..1000000） | — | 未变 |
| `innodb_commit_concurrency` | 默认 **0**；未列入 8.0/8.4 的废止清单 | 源码注释 "The initial default value is 0" | 未变 |
| `innodb_read/write_io_threads` | 均默认 **4**（1..64），"each background thread can handle up to 256 pending I/O requests" | 一致（每 segment `8 × 32 = 256`） | ★ 8.4 改为 **逻辑 CPU/2，最小 4** |
| `innodb_use_native_aio` | 仅 Linux 生效，**默认 ON**，需 libaio | 一致（READONLY） | 未变 |
| `innodb_purge_threads` | 默认 **4**（最大 32），"purge system automatically adjusts" | 一致（`MAX_PURGE_THREADS = 32`，自适应 ±1） | ★ 8.4 改为 **CPU≤16 则 1，否则 4** |
| `innodb_purge_batch_size` | 默认 **300** | 一致 | 未变 |
| `innodb_max_purge_lag` | 默认 **0**（无延迟）；8.0.14 起公式 `(lag/max - 0.9995) × 10000` | 一致 | 未变 |
| `innodb_page_cleaners` | 默认 **4**；★ 官方明说超过 buffer pool instances 时**自动设为该值** | 一致（`ha_innodb.cc` 截断） | ★ 8.4 改为 **= `innodb_buffer_pool_instances`** |
| `innodb_buffer_pool_instances` | 默认 **8**（`buffer_pool_size < 1GB` 时 1） | — | ★ 8.4 改为按 chunk 数与 CPU 计算 |
| `innodb_parallel_read_threads` | 8.0.14 引入，默认 **4** | — | ★ 8.4 改为 **逻辑 CPU/8，最小 4** |

★ **这张表最有价值的一条**：8.4 把**所有后台线程数的默认值从固定 4 改成按 CPU 核数推导**（io/4→CPU/2、purge→CPU 自适应、page cleaners→=bp instances、parallel read→CPU/8）。也就是说 8.0 的"4"在大规格机器上**长期是欠配的**——运维上值得主动调，而不是等默认值。

### 五、SyncArray 与自旋-休眠模型

#### 5.1 ★ SyncArray 的历史使命已作废（源码注释原文）

`storage/innobase/include/sync0arr_impl.h` 文件头有一段"活化石"级注释，这是理解它的关键：

> *"...Why we chose to implement a wait array? First, to make mutexes fast, we had to code our own implementation of them... Then we had the choice of assigning a unique OS event for each mutex, which would be simpler, or using a global wait array. **In some operating systems, the global wait array solution is more efficient... because we can do with a very small number of OS events, say 200. In NT 3.51, allocating events seems to be a quadratic algorithm**, because 10 000 events are created fast, but 100 000 events takes a couple of minutes to create.*
>
> ***As of 5.0.30 the above mentioned design is changed.** Since now OS can handle millions of wait events efficiently, we no longer have this concept of each cell of wait array having one event. Instead, now **the event that a thread wants to wait on is embedded in the wait object (mutex or rw_lock)**. We still keep the global wait array **for the sake of diagnostics and also to avoid infinite wait** — the error_monitor thread scans the global wait array to signal any waiting threads who have missed the signal."*

★★ 即：**8.0 里 SyncArray 只剩两个作用**——① 诊断（`SHOW ENGINE INNODB STATUS` 的 SEMAPHORES 段）；② 防丢信号兜底。

结构是两级（**不是 C++ 类**，是 `struct`）：

```cpp
struct sync_cell_t {
  sync_object_t latch;      // union{ rw_lock_t* / WaitMutex* / BlockWaitMutex* }，NULL = 空闲
  ulint request_type;
  const char *file;  ulint line;
  std::thread::id thread_id;
  bool waiting;
  int64_t signal_count;     // os_event_reset 时抓的值，防丢信号
  std::chrono::steady_clock::time_point reservation_time;
  uint64_t last_scan;       // DFS 找环用
};

struct sync_array_t {
  ulint n_reserved, n_cells;
  sync_cell_t *cells;
  SysMutex mutex;           // ★ 必须是 OS mutex，否则无限递归
  uint64_t last_scan;
};
extern sync_array_t **sync_wait_array;   // 二维：srv_sync_array_size 个数组
```

★ **总 cell 数恒等于 `srv_max_n_threads`（102400），只是切分**：

```cpp
void sync_array_init(ulint n_threads) {
  sync_array_size = srv_sync_array_size;
  ulint n_slots = 1 + (n_threads - 1) / sync_array_size;   // 总数守恒
  for (ulint i = 0; i < sync_array_size; ++i)
    sync_wait_array[i] = ut::new_withkey<sync_array_t>(..., n_slots);
}
```

`innodb_sync_array_size` 默认 **1**（1 个数组 × 102400 cell，一把 `SysMutex`），范围 1–1024，**只读**。调大能降低数组级锁争用。

#### 5.2 三层混合等待策略

```
第 1 层：ut_delay(随机 0..innodb_spin_wait_delay)  → CPU PAUSE，最多 innodb_sync_spin_loops 轮
第 2 层：std::this_thread::yield()                 → 让出 CPU 但仍 runnable
第 3 层：sync array cell + os_event_wait_low       → 真睡（pthread_cond_wait），可精确唤醒
         ↑ 进 cell 后还会再 try_lock 4 次
```

```cpp
  bool is_free(uint32_t max_spins, uint32_t max_delay, uint32_t &n_spins) const {
    do {
      if (!is_locked()) return true;
      ut_delay(ut::random_from_interval_fast(0, max_delay));   // ★ 随机化，防惊群
      ++n_spins;
    } while (n_spins < max_spins);
    return false;
  }
```

`ut_delay()` 用 `UT_RELAX_CPU()`（PAUSE 指令）而非空循环，并带 `spin_wait_pause_multiplier`（默认 50）——注释解释了这是"100 MHz Pentium + Visual C++"时代标定 `delay` 微秒的遗产。

★ **为什么随机化**：所有等待者用同一固定延时会在锁释放瞬间产生"惊群 + 总线风暴"，随机打散重试。

#### 5.3 ★ 防丢信号：一段需要形式化证明的不变式

`ut0mutex.ic` 的 `TTASEventMutex::wait()` 开头有约 120 行 **C++14 内存模型的形式化证明**（Theorem 1：*如果线程 X 决定调 `os_event_wait()`，且每个调 `enter()` 的线程最终都调 `exit()`，那么 X 最终会被唤醒*）。

原因是 `os_event` 存在 ABA 式丢信号问题（源码注释原文）：

```
thread A calls os_event_reset()
thread B calls os_event_set()   [event->is_set == true]
thread C calls os_event_reset() [event->is_set == false]
thread A calls os_event_wait()  [infinite wait!]
```

解法是把 `os_event_reset()` 返回的 `signal_count` 传给 `os_event_wait_low()`——这就是为什么 `sync_cell_t` 要有 `signal_count` 字段。

即便如此，**唤醒算法仍不完美**，所以需要 `srv_error_monitor_thread` 每秒兜底扫一遍：

```cpp
/** If the wakeup algorithm does not work perfectly at semaphore releases,
 this function will do the waking... Note that there's a race condition
 between this thread and mutex_exit ... so sometimes this finds threads
 to wake up even when nothing has gone wrong. */
static void sync_array_wake_threads_if_sema_free_low(sync_array_t *arr) { ... }
```

#### 5.4 InnoDB mutex 的 owner 就是 `std::thread::id`

```cpp
struct TTASEventMutex {
  bool try_lock() {
    auto expected = std::thread::id{};
    return m_owner.compare_exchange_strong(expected, std::this_thread::get_id());
  }
  bool is_owned() const { return peek_owner() == std::this_thread::get_id(); }
  os_event_t event() { return m_event; }      // sync0arr 用它排队
 private:
  std::atomic<std::thread::id> m_owner{std::thread::id{}};
  os_event_t m_event{};
  std::atomic_bool m_waiters{false};
};
```

rw_lock 用 `lock_word` + `writer_thread` + `recursive` 三字段，**X 锁靠"先减 `X_LOCK_DECR` 占住 next-writer 位"来防写饥饿**（后续 S 请求发现 `lock_word` 已被减而失败，不再插队）。

### 六、★ PFS 如何观测这整堆线程

#### 6.1 `performance_schema.threads` 的 NAME 从哪来

`NAME` = `PFS_thread_class::m_name`，由注册时的 `PSI_thread_info` 决定，前缀来自 category（`"sql"` / `"innodb"` / `"mysys"`）。

**InnoDB 完整清单（`thread/innodb/*`，33 个）**：

```
log_archiver_thread          (ib_log_arch)      page_archiver_thread        (ib_page_arch)
buf_dump_thread              (ib_buf_dump)      buf_resize_thread           (ib_buf_resize)
dict_stats_thread            (ib_dict_stats)    io_ibuf_thread              (ib_io_ibuf)
io_read_thread               (ib_io_rd)         io_write_thread             (ib_io_wr)
log_files_governor_thread    (ib_log_files_g)   log_writer_thread           (ib_log_writer)
log_checkpointer_thread      (ib_log_checkpt)   log_flusher_thread          (ib_log_flush)
log_write_notifier_thread    (ib_log_wr_notif)  log_flush_notifier_thread   (ib_log_fl_notif)
recv_writer_thread           (ib_recv_write)    srv_error_monitor_thread    (ib_srv_err_mon)
srv_lock_timeout_thread      (ib_srv_lock_to)   srv_master_thread           (ib_src_main)
srv_monitor_thread           (ib_srv_mon)       srv_purge_thread            (ib_srv_purge)
srv_worker_thread            (ib_srv_wkr)       trx_recovery_rollback_thread(ib_tx_recov)
page_flush_thread            (ib_pg_flush)      page_flush_coordinator_thread(ib_pg_flush_co)
fts_optimize_thread          (ib_fts_opt)       fts_parallel_merge_thread   (ib_fts_merge)
fts_parallel_tokenization_thread (ib_fts_token) srv_ts_alter_encrypt_thread (ib_ts_encrypt)
parallel_read_thread         (ib_par_rd)        parallel_rseg_init_thread   (ib_par_rseg)
clone_ddl_thread             (ib_clone_ddl)     clone_gtid_thread           (ib_clone_gtid)
ddl_thread                   (ib_ddl)           meb::redo_log_archive_consumer_thread (ib_meb_rl)
```

（括号内是 OS 线程名，会通过 `my_thread_self_setname()` 写进 pthread name。）

★ **`thread/sql/one_connection` 的注册带 `PSI_FLAG_NO_SEQNUM`** —— 正是因为**线程缓存复用**会让 seqnum 语义错乱：

```cpp
  { &key_thread_one_connection, "one_connection", "connection",
      PSI_FLAG_USER | PSI_FLAG_NO_SEQNUM, 0, PSI_DOCUMENT_ME},
```

#### 6.2 ★ 线程复用时 PFS 要显式重建

`handle_connection` 的复用循环里：

```cpp
    if (pthread_reused) {
      /* Reusing existing pthread: Create new instrumentation for the new THD job,
         and attach it to this running pthread. */
      PSI_thread *psi = PSI_THREAD_CALL(new_thread)(key_thread_one_connection, 0, thd, thd->thread_id());
      PSI_THREAD_CALL(set_thread_os_id)(psi);
      PSI_THREAD_CALL(set_thread)(psi);
    }
    ...
    delete thd;
    PSI_THREAD_CALL(delete_current_thread)();     // PFS 实例销毁
```

★ 这两个 PFS 调用的真实符号名是 **`pfs_new_thread_vc`** / **`pfs_delete_current_thread_vc`**（带 `_vc` 后缀，8.0 的 "victim cancellation" 版本）；`pfs_delete_thread` 这个无后缀名在 8.0.39 **不存在**（全仓 0 命中）。

**pthread 复用、THD 不复用**（`init_new_thd` 每连接一次）⇒ `THD*` 永远不能跨连接缓存。这也是 `threads` 表的 `THREAD_ID`（PFS 内部 id）与 `PROCESSLIST_ID` 都会持续增长的原因。

新建线程的路径是另一条（`mysql_thread_create` → `PSI_THREAD_CALL(spawn_thread)` → 真正的入口被换成 `pfs_spawn_thread`，先建 instrumentation 再调用户的 `start_routine`）。

#### 6.3 ★ PFS 自己没有线程

grep 全仓 `storage/perfschema`，**只有一处 `my_thread_create`**（就是 `pfs_spawn_thread` 包装器本身）。PFS 完全**寄生**在被观测线程上：由当前线程自己执行 `aggregate_thread()` 汇总、自己执行 `lf_pinbox` 内存回收（这就是每个 `PFS_thread` 要带 **9 个 `LF_PINS`** 的原因）。

#### 6.4 容量与静默降级

| 变量 | 默认 | 说明 |
|---|---|---|
| `performance_schema_max_thread_classes` | **100** | 能注册多少种线程 instrument（实际需求约 54） |
| `performance_schema_max_thread_instances` | **-1（autoscale）** | 容器硬上限 **256×256 = 65536** |

★ **实例耗尽时静默降级**：`create_thread()` 返回 `nullptr` 不报错——线程照常创建、照常跑，只是**没有 instrumentation**：`threads` 表看不到它，它触发的所有 `events_waits_*` 都不采集，它访问任何 PFS hash 都失败。只能靠 `Performance_schema_thread_instances_lost` 状态变量发现。

#### 6.5 鸡生蛋问题

PFS 必须在任何线程创建之前"假装可用"：`pre_initialize_performance_schema()` 里 `THR_PFS = nullptr`，而此时 `psi_thread_service` 指向 **noop 实现**（`mysys/psi_noop.cc`），所以早期线程（signal handler、bootstrap、main）的 `PSI_THREAD_CALL(...)` 是安全的空操作。**main 线程是唯一"事后补 instrumentation"的特殊线程**（`PSI_THREAD_CALL(new_thread)(key_thread_main, ...)`）。

---

### 七、★ server ↔ InnoDB 的交互边界（两族线程如何咬合）

前面各章分别讲了两族线程。这一章讲**它们之间**：谁跑 InnoDB 代码、两个世界的句柄怎么互查、后台线程为什么要伪造一个 THD、并发闸门开在哪、关机时谁等谁。

#### 7.1 谁在跑 InnoDB 代码？——绝大多数是用户线程

★ **InnoDB 不是一个"服务端进程"，它是被 server 直接调用的库**。绝大部分 InnoDB 代码跑在**连接线程**上；只有固定几件事（刷脏、purge、checkpoint、统计、恢复）在自己线程里。

server 侧的调用面有两层：

| 层 | 机制 | 例子 |
|---|---|---|
| **handlerton（每引擎一份函数指针表）** | `innodb_init()`（★ 插件 init 真名，不是 `innobase_init`）里逐个赋值 | `commit=innobase_commit`、`rollback`、`prepare`、`close_connection`、`kill_connection`（★ 不是 `kill_query`）、`pre_dd_shutdown`、`panic`、`flush_logs`、DD/SDI/Clone 一整套 |
| **handler 虚方法（每表一个实例）** | `innobase_hton->create = innobase_create_handler` 造出 `ha_innobase`，再走虚函数 | `write_row` / `update_row` / `index_read` / `general_fetch` / `open` / `store_lock` |

**证据 1**：`ha_innobase::write_row()` 里没有任何线程切换或队列投递——**同步**调进 `row_insert_for_mysql()`。

**证据 2**：`TrxInInnoDB` 的注释把这条边界写死了（`trx0trx.h`）：

```cpp
/** Track if a transaction is executing inside InnoDB code. It acts
like a gate between the Server and InnoDB.  */
class TrxInInnoDB { ... };
```

⇒ 所以"InnoDB 线程数"这个说法要小心：**真正执行 SQL 的从来是连接线程**，后台线程只做维护工作。

#### 7.2 THD ↔ trx_t：两个世界如何互查

★ **真名是 `thd_to_trx(THD*)`，不是 `thd_get_trx()`**（后者全仓无定义），且返回的是 **指针的引用 `trx_t *&`**，所以调用方能直接改写 THD 上挂的 trx：

```cpp
// storage/innobase/handler/ha_innodb.cc
[[nodiscard]] innodb_session_t *&thd_to_innodb_session(THD *thd) {
  innodb_session_t *&innodb_session =
      *(innodb_session_t **)thd_ha_data(thd, innodb_hton_ptr);
  if (innodb_session != nullptr) return innodb_session;
  innodb_session = ut::new_withkey<innodb_session_t>(UT_NEW_THIS_FILE_PSI_KEY);
  return innodb_session;
}
[[nodiscard]] trx_t *&thd_to_trx(THD *thd) {
  innodb_session_t *&innodb_session = thd_to_innodb_session(thd);
  return innodb_session->m_trx;
}
```

绑定链：`THD --thd_ha_data(thd, innodb_hton_ptr)--> void* 槽 --> innodb_session_t --> m_trx`。

**trx 何时创建**：不是笼统的"第一条 SQL"，而是**任何入口第一次调 `check_trx_exists(thd)` 时惰性创建**——最典型的第一站是 `ha_innobase::store_lock()`（server 在 `get_lock_data()` 阶段就调）。释放点在 `innobase_close_connection()`。

★ **反向指针 `trx->mysql_thd`**（真实字段名）的四个用途：

| 用途 | 代码 |
|---|---|
| 判断事务是否被 KILL | `trx_is_interrupted()` = `trx->mysql_thd && thd_killed(trx->mysql_thd)` |
| 把等待边上报给 server（PFS `data_lock_waits` 的源头） | `thd_report_lock_wait(waiting->trx->mysql_thd, blocking->trx->mysql_thd, ...)` |
| ★ **死锁 victim 选择**：后台线程的 trx **不能**当受害者 | `if (... \|\| trx->mysql_thd == nullptr \|\| ...) return true;` —— 注释原文 *"we don't rollback ... transactions owned by background threads"*，因为它没有 THD 可被通知 |
| `SHOW ENGINE INNODB STATUS` 打印事务的 SQL | `innobase_mysql_print_thd(f, trx->mysql_thd, max_query_len)` |

#### 7.3 ★ 后台线程为什么要伪造一个 THD

`create_internal_thd()`（★ 真正在 **`sql/sql_thd_internal_api.cc`**，不在 `sql_class.cc`）：

```cpp
THD *create_internal_thd() {
  THD *thd = new THD(false);
  thd->system_thread = SYSTEM_THREAD_BACKGROUND;
  thd->security_context()->skip_grants();          // 跳过权限检查
  thd->thread_stack = reinterpret_cast<char *>(&thd);
  thd->store_globals();                             // ★ 设置 current_thd / THR_MALLOC
  ...
  PSI_THREAD_CALL(set_thread_THD)(psi, thd);        // ★ 让 PFS 看到这个 THD
```

**哪些 InnoDB 后台线程调了它**（逐个核实）：master、purge coordinator、purge **worker**、`dict_stats`、fts_optimize、fts 并行分词、`buf_flush_page_coordinator`（**只协调者**）、`trx_recovery_rollback`、clone 新线程、ts_alter_encrypt；以及 **log_writer / log_checkpointer / log_files_governor 只在 debug 下**（`ut_d(...)`）。

**哪些没调**：`buf_dump`、`buf_resize`、page cleaner **worker**、log_flusher 与两个 notifier、io 线程、monitor / error_monitor / lock_wait_timeout。

**★ 为什么要 THD——三条硬证据**：

1. **要执行内部 SQL、访问 DD**：`dict_stats_thread()` 把自己的 internal THD 一路传给 `dd_table_open_on_id(table_id, thd, &mdl, ...)` 去开 DD 表，再写 `mysql.innodb_table_stats`。这类"内部 SQL"由 InnoDB 自带的解析器执行（`dict_stats_exec_sql()` → `que_eval_sql("PROCEDURE TABLE_STATS_SAVE () IS ...")`），**不走 server parser，但必须有 THD** 才能拿 MDL、开 DD 表。
2. **`store_globals()` 建立线程局部 `current_thd` / `THR_MALLOC`**：否则 `thd_to_trx()`、错误上报、`THDVAR()` 全不能用。`ddl0fts.cc` 里甚至显式断言 `ut_ad(current_thd == thd)`。反例是 `dict0dd.cc` 的兜底：*"Safeguard in release mode if background thread doesn't have THD"* → `ut_d(ut_error); ut_o(return true);`
3. **PFS**：`set_thread_THD(psi, thd)`，注释原文 *"so that system variables and status variables are visible for the background thread"*。

★ 附带一个易错点：**后台线程不会被加进 `Global_THD_manager`**——`thd_init()` 显式跳过 `SYSTEM_THREAD_BACKGROUND`（注释：*"P_S interprets all threads with thread_id != 0 as foreground threads"*）。所以它们**不出现在 `SHOW PROCESSLIST`**，只在 `performance_schema.threads` 里以 `thread/innodb/*` 出现。

#### 7.4 并发闸门恰好开在边界上

`innobase_srv_conc_enter_innodb()` / `_exit_innodb()` 是 **`static inline`，位于 `ha_innodb.cc`**——闸门就在 server 调进 InnoDB 的那一行。调用点成对出现在 `write_row` / `update_row` / `delete_row` / **`index_read`** / `general_fetch`（★ `rnd_next` 没有，它走 `index_read`/`general_fetch`）。

ticket 语义：一次进入领 `innodb_concurrency_tickets`（默认 **5000**）张票，同条 SQL 内后续 `enter` 只扣票；票用完才真去竞争 `srv_conc.n_active`。锁等待时 `srv_conc_force_exit_innodb()` 让位（避免死锁），等完 `srv_conc_force_enter_innodb()` 无条件重入。

★ **默认 `innodb_thread_concurrency = 0` 时三层全部短路**：① handler 层 `if (srv_thread_concurrency) {...}` 整块不执行；② `declared_to_be_inside_innodb` 恒 false，`exit` 根本不调底层；③ `srv_conc_enter_innodb_with_atomics()` 循环首行就 `if (srv_thread_concurrency == 0) return DB_SUCCESS;`。**零开销**。

> `skip_concurrency_ticket()`：intrinsic 临时表与 **DD 表**豁免——注释原文 *"When InnoDB uses DD APIs, it leaves InnoDB and re-enters InnoDB again. The reads, updates as part of DDLs should be exempt"*。

#### 7.5 shutdown 的反向握手：`m_master_ready_for_dd_shutdown`

★ 这是两族线程**唯一一处"InnoDB 通知 server"的显式握手**，方向是反的：不是 server 命令 master 停，而是 **master 告诉 server "我不再碰 DD 了，你可以关 DD 了"**。

```cpp
// server 侧：storage/innobase/srv/srv0start.cc  srv_pre_dd_shutdown()
  if (srv_thread_is_active(srv_threads.m_master)) {
    srv_wake_master_thread();
    os_event_wait(srv_threads.m_master_ready_for_dd_shutdown);   // ★ WAIT
  }
  ut_d(trx_sys_before_pre_dd_shutdown_validate());
  srv_shutdown_set_state(SRV_SHUTDOWN_PURGE);

// InnoDB 侧：srv_master_thread()
  srv_master_main_loop(slot);
  srv_master_pre_dd_shutdown_loop();
  os_event_set(srv_threads.m_master_ready_for_dd_shutdown);      // ★ SET
```

**为什么必须这样**：master 退出主循环前还有一批**依赖 DD / 系统事务**的收尾（`srv_master_do_pre_dd_shutdown_tasks()` → `row_drop_tables_for_mysql_in_background()` 后台删表）。server 侧紧接着就要 `dd::shutdown()`，一旦 DD 关掉 master 再用就崩。于是必须让 master 在"还允许碰 DD"的窗口里干完再举手。

调用链（server 侧已核实）：`mysqld.cc` 的 `clean_up()` → `ha_pre_dd_shutdown()` → `hton->pre_dd_shutdown` → `innodb_pre_dd_shutdown()` → `srv_pre_dd_shutdown()`；真正的全线收尾在**更晚**的 `hton->panic = innodb_shutdown` → `srv_shutdown()`。

★ **九阶段枚举在 `include/srv0shutdown.h`**（不是 `srv0srv.h`），每个状态都注明"这个阶段停哪些线程、为什么"。收尾靠一张表 `threads_to_stop[]`（lock_wait_timeout / error_monitor / monitor / buf_dump / buf_resize / master），逐个 `notify()` + 轮询等待，超 60 秒打日志；源码注释原文：

```cpp
      /* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM
      HERE OR EARLIER */
```

★ `srv_thread_delay_cleanup_if_needed()` 是**纯测试钩子**（`DBUG_EXECUTE_IF`，release 为空函数），用来人工拉长"发现要关机 → 真正清理"的窗口以验证等待逻辑。★ purge coordinator 传 **`false`**——因为它自己就在 `srv_pre_dd_shutdown()` 的等待列表里，若去等那个信号会**与 pre_dd_shutdown 互相等待而死等**（源码注释原文说明了这点）。

#### 7.6 线程标识与观测：两族的注册方式不同

| | server 线程 | InnoDB 后台线程 |
|---|---|---|
| PSI key 类型 | `PSI_thread_key key_thread_one_connection`（`mysqld.cc`） | **`mysql_pfs_key_t`**（如 `srv_master_thread_key`、`log_writer_thread_key`、`io_read_thread_key`…） |
| PFS 名 | `thread/sql/one_connection` | `thread/innodb/*`（如 `thread/innodb/srv_master_thread`） |
| 创建 | `mysql_thread_create(key, &id, &attrib, handle_connection, ...)` | ★ `os_thread_create` 是**宏** → `create_detached_thread()` → `std::thread` + `detach()` |
| 注册到 PFS | server 侧 | ★ InnoDB 自己调 `PSI_THREAD_CALL(new_thread)`（`os0thread-create.h` 的 `MySQL_thread::preamble()`，同时 `my_thread_init()`） |
| 是否进 `Global_THD_manager` | ✅ | ❌（见 7.3） |

★ **为什么 `IB_thread` 的 create 与 start 要分离**——源码没有字面为 "create and start separate" 的注释，但等价的两段注释解释了：新线程一进来就自旋等 `ALLOWED_TO_START`，`start()` 才放行。原因是：**必须先把 `IB_thread` 句柄赋到 `srv_threads.m_xxx` 这种 shutdown 能看见的地方，再 `start()`**；否则线程可能在句柄尚未发布时就跑完，关机时漏停它。因此 `thread_is_active()` 把 `ALLOWED_TO_START` 也算作"活跃"（注释原文解释了这个竞态）。

#### 7.7 ★ 本篇的"线程归属原则"：哪些线程不在这里深剖

清单表里的 26 种线程，**本篇只负责"全景 + 通用机制"**（创建方式、shutdown 顺序、并发闸门、等待挂靠、PFS 观测）。**专职任务线程一律归位到各自模块的文档**，因为它们的循环与业务语义强耦合，拆出来讲会两头不完整：

| 线程 | 归属文档 | 本篇保留什么 |
|---|---|---|
| `log_writer` / `flusher` / 两个 notifier / `checkpointer` / `files_governor` | [`../innodb/redo_log.md`](../../innodb/redo_log.md) | 清单 + 事件接力链在 [`../innodb/redo_log.md`](../../innodb/redo_log.md) |
| `buf_flush_page_coordinator` / worker | [`../innodb/buffer_pool.md`](../../innodb/buffer_pool.md) | 同上 |
| `buf_dump` / `buf_resize` | [`../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)「按需线程」 | 同上 |
| purge coordinator / worker | [`../innodb/undo_log.md`](../../innodb/undo_log.md)「Purge 线程模型」 | 同上 |
| io handler（ibuf / read / write）、master | [`../innodb/io.md`](../../innodb/io.md)「I/O 线程模型」 | 同上 |
| **`dict_stats_thread`** | [`../server/dd/statistics.md`](../../server/dd/statistics.md)「后台自动重算 → 线程视角」 | ★ 本篇只记"它存在"，循环/阈值/两个上下文都在那边 |
| **`fts_optimize_thread`** | [`../feat/fts.md`](../../feat/fts.md) | 同上 |
| **并行读线程**（`row0pread.cc`） | [`../innodb/parallel_scan.md`](../../innodb/parallel_scan.md) | 同上 |
| archiver（log / page） | [`../innodb/redo_log.md`](../../innodb/redo_log.md)「redo log archiving」 | 同上 |
| 复制线程（io / sql / worker） | [`../server/replication/replica.md`](../../server/replication/replica.md) | 同上 |
| 连接线程 | [`../server/infra/connection_threading.md`](connection_threading.md) | 同上 |
| `lock_wait_timeout` / `error_monitor` / `monitor` | **本篇第八章**（无更合适的模块宿主，属"跨模块守护线程"） | 深剖 |

（判定标准很简单：**有明确模块宿主的专职线程 → 模块文档；跨模块的守护/兜底线程 → 本篇**。）

#### 7.8 本节的纠名清单

| 流传 | 真实 |
|---|---|
| `thd_get_trx(thd)` | **`thd_to_trx(thd)`**（返回 `trx_t *&`） |
| `THR_THD` pthread key | **8.0.39 已不存在**——`current_thd` 是 C++11 `thread_local`（`sql/current_thd.cc`）；残留的只有两处 `#include` 注释 |
| `innobase_init` / `innobase_end` | 插件 init/deinit 真名 **`innodb_init` / `innodb_deinit`**；真正的收尾在 `hton->panic = innodb_shutdown` |
| `pfs_spawn_thread` / `pfs_register_thread` 覆盖 InnoDB | **不成立**——server 侧无调用，InnoDB 自己 `PSI_THREAD_CALL(new_thread)` |
| 九阶段枚举在 `srv0srv.h` | 在 **`include/srv0shutdown.h`** |
| `srv_thread_delay_cleanup_if_needed` 是关机逻辑 | 是 **`DBUG_EXECUTE_IF` 测试钩子**，release 为空 |

> server 侧连接线程的完整生命周期见 [`../server/infra/connection_threading.md`](connection_threading.md)；InnoDB 各后台线程的循环细节见 [purge](../../innodb/undo_log.md)、[io](../../innodb/io.md)、[page cleaner](../../innodb/buffer_pool.md)、[log 线程](../../innodb/redo_log.md)。

### 八、★ 三个守护线程：lock_wait / error_monitor / monitor

清单表里这三个常被一笔带过，但它们承担了**超时、死锁、防睡死**三件关键事。逐个剖。

> ★ **先澄清 5.1 节容易引起的误读**：SyncArray"只剩诊断 + 防丢信号"**不等于它不在等待路径上**——每次阻塞式 mutex / rw_lock 等待**仍会 `sync_array_get_and_reserve_cell()` 占一个 cell**，只是**睡在锁内嵌的 event 上**（不再用一个 cell 独占一个 OS event）。所以 8.0 里它是"仍在线的兜底"，不是死代码。

#### 8.1 `lock_wait_timeout_thread`：超时 + 死锁检测是同一个线程

★ **没有独立的死锁检测线程**——它就在这个线程里（`lock0wait.cc`）：

```cpp
void lock_wait_timeout_thread() {
  os_event_t event = lock_sys->timeout_event;
  auto last_checked_for_timeouts_at = std::chrono::steady_clock::now();
  do {
    auto current_time = std::chrono::steady_clock::now();
    if (std::chrono::seconds(1) <= current_time - last_checked_for_timeouts_at) {
      last_checked_for_timeouts_at = current_time;
      lock_wait_check_slots_for_timeouts();          // ① 超时检查（≤1Hz 节流）
    }
    lock_wait_update_schedule_and_check_for_deadlocks();  // ② 权重 + 死锁检测
    os_event_wait_time_low(event, std::chrono::seconds{1}, sig_count);
    sig_count = os_event_reset(event);
  } while (srv_shutdown_state.load() < SRV_SHUTDOWN_CLEANUP);
}
```

| 项 | 事实 |
|---|---|
| 事件 | ★ `lock_sys->timeout_event`（**`lock_wait_timeout_event` 这个名字不存在**） |
| 周期 | **1 秒**，硬编码字面量；但**不是纯轮询**——新等待者入槽时会 `lock_set_timeout_event()` 提前踢醒它 |
| ★ 节流差异 | **超时检查**被 `last_checked_for_timeouts_at` 限制为 ≤1Hz（注释原文：超时精度只有 1 秒，"nobody cares if we wake up after T or T+0.99"）；**死锁检测每次醒来都做**（★ `check_deadlock_frequency` 这类节流机制 8.0 已不存在） |

**一次醒来做的两件事**：

```cpp
static void lock_wait_update_schedule_and_check_for_deadlocks() {
  auto table_reservations = lock_wait_snapshot_waiting_threads(infos);
  lock_wait_build_wait_for_graph(infos, outgoing);
  lock_wait_compute_and_publish_weights_except_cycles(...);   // CATS 权重（总做）
  if (innobase_deadlock_detect) {
    lock_wait_find_and_handle_deadlocks(infos, outgoing, new_weights);
  }
}
```

★ `innodb_deadlock_detect` 默认 **ON**。关掉后**只跳过死锁检测**，等待图快照与 CATS 权重照算——死锁只能等 `innodb_lock_wait_timeout` 超时解开。★ **切换开关时要"踢一脚"**（注释原文解释了为什么要）：

```cpp
  /* In case deadlock detection was disabled for a long time it could happen
  that all clients have deadlocked with each other and thus they stopped
  changing the wait-for graph ... So if we now change from OFF to ON we
  need to "kick-start" the process. */
  lock_wait_request_check_for_cycles();
```

**等待槽位**：用户线程等锁时 `lock_wait_table_reserve_slot()` 在 `lock_sys->waiting_threads[]` 里线性找空槽，然后 `os_event_wait(slot->event)` 睡在**自己的槽位事件**上。槽位结构是 **`srv_slot_t`**（★ `lock_wait_t` 不存在），数组大小 = **`srv_max_n_threads` = 102400**（`ha_innodb.cc` 里 `srv_max_n_threads = 100 * 1024;`）。耗尽时直接 `ib::error` + `ut_error`。

**★ 超时判定的两个易错点**：

1. 判定用的是 **槽位的 `slot->suspend_time`**，**不是** `trx->lock.wait_started`——后者只用于 `SHOW ENGINE INNODB STATUS` 打印和 `information_schema.innodb_trx.trx_wait_started`。
2. ★ **超时只回滚当前语句，不是整事务**：`innodb_rollback_on_timeout` 默认 **OFF** 且是 **READONLY 启动变量**。代码原文：

```cpp
    case DB_LOCK_WAIT_TIMEOUT:
      if (row_rollback_on_timeout) { trx_rollback_to_savepoint(trx, nullptr); break; }
      [[fallthrough]];          // 否则只回滚到上一个语句 savepoint
```

（对照：**死锁受害者是整事务回滚**——源码明确说 `DB_DEADLOCK` "rolls back whole transaction, not just the current query"。）

`innodb_lock_wait_timeout` 默认 **50** 秒（`MYSQL_THDVAR_ULONG`，会话级；>100000000 视为禁用）。

#### 8.2 `srv_error_monitor_thread`：1Hz 兜底 + 长等待致命判定

`srv0srv.cc`，事件 **`srv_error_event`**，睡眠 **1 秒**（同样是硬编码字面量），`goto loop` 写法，退出条件 `< SRV_SHUTDOWN_CLEANUP`。

**它一次醒来干五件事**（逐个核实）：

| # | 动作 | 说明 |
|---|---|---|
| ① | LSN 回退检测 | `new_lsn < old_lsn` → `ib::error`（历史遗留的 bug 追踪） |
| ② | **`buf_LRU_stat_update()`** | ★ **"搭便车"的任务**——注释原文承认 *"While this doesn't relate to error monitoring, it's here for historical reasons, as it depends on being called ~1Hz"*（LRU 统计的历史窗口 50 个 interval 正是靠它推进） |
| ③ | **`sync_arr_wake_threads_if_sema_free()`** | ★ **防睡死**：源码注释原文 *"If the wakeup algorithm does not work perfectly at semaphore releases, this function will do the waking ... Note that there's a race condition ... so sometimes this finds threads to wake up even when nothing has gone wrong."* |
| ④ | **`sync_array_detect_deadlock()`** | ★ 检测到 latch 死锁 → **`ib::fatal` 直接崩溃**（不返回）。release 下精度受限（注释承认 "there might be false negatives"） |
| ⑤ | 长等待判定 | 见下 |

**★ 长等待的三级阈值**（`sync_array_print_long_waits_low()`）：

| 级别 | 阈值 | 行为 |
|---|---|---|
| 告警 | **4 分钟**（`constexpr std::chrono::minutes timeout{4}`；valgrind 下 40 分钟） | `ib::warn` "A long semaphore wait:" + `sync_array_cell_print()`（即你看到的 *"--Thread X has waited at ... for N seconds the semaphore:"*） |
| 致命 | **`srv_fatal_semaphore_wait_threshold` = 600 秒**（**只在 debug 下**暴露为 `innodb_semaphore_wait_timeout_debug`，release 是不可改的常量） | `ib::fatal` —— *"Semaphore wait has lasted > %llu seconds. We intentionally crash the server because it appears to be hung."* |
| ★ 触发条件 | **必须连续 11 次（>10）1Hz 轮询都看到同一个 (waiter, sema) 对** | `fatal_cnt++`，换人就清零 ⇒ 实际约需 **11 秒的持续确认**，不是一到 600 秒就崩 |

★ **一个容易被忽略的副作用**：一旦触发 4 分钟告警，它会 **自己阻塞式睡 30 秒**去吐诊断（`srv_innodb_needs_monitoring++` + `lock_set_timeout_event()` + `sleep_for(30s)`），期间 1Hz 心跳暂停，并顺带"叫醒" monitor 线程与 lock_wait 线程一起输出。这解释了线上 hang 时 error log 里突然出现连续 30 秒的 InnoDB Monitor 输出。

#### 8.3 `srv_monitor_thread`：15 秒周期，与 `SHOW ENGINE INNODB STATUS` 无关

```cpp
  const auto sleep_interval = std::chrono::seconds{15};   // ★ 15 秒，局部变量
  while (0 != os_event_wait_time(srv_monitor_event, sleep_interval)) { ... }
```

★ 循环写法与前两个不同：`os_event_wait_time` **超时返回非 0、被唤醒返回 0** ⇒ 超时才干活，被 `srv_monitor_event` 唤醒就**退出**（用于 shutdown）。

- `innodb_status_output`（→`srv_print_innodb_monitor`）与 `innodb_status_output_locks` **默认都是 OFF**；后者**必须先开前者**才生效。
- ★ 还有个"InnoDB 自己给自己下命令"的计数器 `srv_innodb_needs_monitoring`：即使两个变量都 OFF，>0 也会让 monitor 线程临时打印（触发者包括上面的 30 秒诊断）。
- 打印时用 **`MUTEX_NOWAIT(mutex_skipped)`**——连续拿不到锁超过阈值才退化为阻塞，避免监控本身把系统卡死。

★ **`SHOW ENGINE INNODB STATUS` 不是复用这个线程的输出**：它由**用户会话线程自己同步**调 `innodb_show_status()` → `srv_printf_innodb_monitor(srv_monitor_file, false, ...)`（第二个参数 `nowait` 传 **false**，即允许阻塞等锁）生成，与 15 秒周期完全无关；两者只是**共用同一个函数与同一个临时文件**（`srv_monitor_file_mutex` 互斥）。

#### 8.4 三个守护线程横向对照

| | `lock_wait_timeout` | `srv_error_monitor` | `srv_monitor` |
|---|---|---|---|
| 文件 | `lock/lock0wait.cc` | `srv/srv0srv.cc` | `srv/srv0srv.cc` |
| 事件 | `lock_sys->timeout_event` | `srv_error_event` | `srv_monitor_event` |
| 周期 | 1s（事件驱动 + 1s 心跳） | 1s | **15s** |
| 唤醒者 | 新等待者入槽 / 长等待诊断 | 基本只靠超时 | 基本只靠超时 |
| 停止手段 | `threads_to_stop[]` → `lock_set_timeout_event` | `os_event_set(srv_error_event)` | `os_event_set(srv_monitor_event)` |
| 退出状态 | `< SRV_SHUTDOWN_CLEANUP` | 同左 | wait 返回 0 |

**纠名**：`lock_wait_timeout_event`、`srv_lock_timeout_event`、`lock_wait_request_slots()`、`lock_wait_t`、`check_deadlock_frequency` —— **全部不存在**；`storage/innobase/include/lock0wait.h` 这个文件**也不存在**（接口声明散在 `lock0lock.h` / `lock0priv.h`）。

## 相关的系统变量

### 线程/并发相关

| 变量 | 默认 | 只读 | 说明 |
|---|---|---|---|
| `max_connections` | 151 | 否 | 闸门 0 |
| `thread_cache_size` | 运行时算：`8 + max_connections/100`（=9），cap 100 | 否 | 闸门 0.5 |
| **`innodb_thread_concurrency`** | **0（关闭）**，范围 0–1000 | 否 | 闸门 1 |
| `innodb_concurrency_tickets` | **5000** | 否 | 一次进入发多少张票 |
| `innodb_adaptive_max_sleep_delay` | 150000 µs | 否 | 闸门 1 退避上限 |
| `innodb_thread_sleep_delay` | 10000 µs | 否 | 闸门 1 初始 sleep |
| **`innodb_sync_array_size`** | **1**，范围 1–1024 | **是** | sync array 切分份数 |
| `innodb_purge_threads` | **4**（含 coord），max 32 | **是** | |
| `innodb_page_cleaners` | **4**，max 64，被 `innodb_buffer_pool_instances` 截断 | **是** | |
| `innodb_read_io_threads` / `innodb_write_io_threads` | 各 **4** | **是** | |
| **`innodb_sync_spin_loops`** | **30** | 否 | 自旋轮数 |
| **`innodb_spin_wait_delay`** | **6** | 否 | 每轮最大延时 |
| `innodb_spin_wait_pause_multiplier` | **50** | 否 | 每单位延时几个 PAUSE |
| `innodb_deadlock_detect` | **ON** | 否 | 关闭后靠 `innodb_lock_wait_timeout` |
| `innodb_lock_wait_timeout` | **50** 秒 | 否（会话级） | 行锁超时 |
| `lock_wait_timeout` | **31536000** 秒（1 年） | 否（会话级） | MDL / 全局读锁超时 |

### PFS 相关

| 变量 | 默认 | 只读 |
|---|---|---|
| `performance_schema_max_thread_classes` | 100 | 是 |
| `performance_schema_max_thread_instances` | -1（autoscale），上限 **1048576** | 是 |

### 注册定义原文（server 层）

`max_connections`（`sql/sys_vars.cc:2979-2985`）：

```c
static Sys_var_ulong Sys_max_connections(
    "max_connections", "The number of simultaneous clients allowed",
    GLOBAL_VAR(max_connections), CMD_LINE(REQUIRED_ARG), VALID_RANGE(1, 100000),
    DEFAULT(MAX_CONNECTIONS_DEFAULT), BLOCK_SIZE(1), NO_MUTEX_GUARD,
    NOT_IN_BINLOG, ON_CHECK(nullptr), ON_UPDATE(nullptr), nullptr,
    /* max_connections is used as a sizing hint by the performance schema. */
    sys_var::PARSE_EARLY);
```

默认值常量（`sql/sys_vars.h:104-109`）：

```c
/**
  Maximum number of connections default value.
  151 is larger than Apache's default max children,
  to avoid "too many connections" error in a common setup.
*/
constexpr const unsigned long MAX_CONNECTIONS_DEFAULT{151};
```

/* 【注】`sys_var::PARSE_EARLY` 的意义：PFS 要按 `max_connections` 做 autosize，
   所以它必须在**启动早期**就能读到，不能等 PFS 初始化。 */

`thread_cache_size`（`sql/sys_vars.cc:5161-5166`）——注意注册处 `DEFAULT(0)`，**真实默认值在运行时算**：

```c
static Sys_var_ulong Sys_thread_cache_size(
    "thread_cache_size", "How many threads we should keep in a cache for reuse",
    GLOBAL_VAR(Per_thread_connection_handler::max_blocked_pthreads),
    CMD_LINE(REQUIRED_ARG, OPT_THREAD_CACHE_SIZE), VALID_RANGE(0, 16384),
    DEFAULT(0), BLOCK_SIZE(1), NO_MUTEX_GUARD, NOT_IN_BINLOG, nullptr,
    ON_UPDATE(modify_thread_cache_size));
```

推导的**真实位置**在 `init_common_variables()`（`sql/mysqld.cc:5052-5070`）：

```c
  longlong default_value;
  sys_var *var;
  /* Calculate and update default value for thread_cache_size. */
  if ((default_value = 8 + max_connections / 100) > 100) default_value = 100;
  var = find_static_system_variable("thread_cache_size");
  var->update_default(default_value);

  /* Calculate and update default value for host_cache_size. */
  if ((default_value = 128 + max_connections) > 628 &&
      (default_value = 628 + ((max_connections - 500) / 20)) > 2000)
    default_value = 2000;
  var = find_static_system_variable("host_cache_size");
  var->update_default(default_value);

  /* Fix thread_cache_size. */
  if (!thread_cache_size_specified &&
      (Per_thread_connection_handler::max_blocked_pthreads =
           8 + max_connections / 100) > 100)
    Per_thread_connection_handler::max_blocked_pthreads = 100;
```

/* 【注】★ 做了**两件**事：`update_default()` 改 `SHOW` 出来的默认值；
   后一段改运行时实际值（只在用户没显式指定时，标志 `thread_cache_size_specified`
   在 `sql/mysqld.cc:10882-10884` 由 `OPT_THREAD_CACHE_SIZE` 置位）。 */

`lock_wait_timeout`（`sql/sys_vars.cc:2421-2425`）：

```c
static Sys_var_ulong Sys_lock_wait_timeout(
    "lock_wait_timeout",
    "Timeout in seconds to wait for a lock before returning an error.",
    HINT_UPDATEABLE SESSION_VAR(lock_wait_timeout), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1, LONG_TIMEOUT), DEFAULT(LONG_TIMEOUT), BLOCK_SIZE(1));
```

```c
constexpr const unsigned long LONG_TIMEOUT{3600 * 24 * 365};
```

/* 【注】`LONG_TIMEOUT` = 31536000 秒 = 1 年，即"一年"这个默认值是常量算出来的不是魔法数。 */

### 注册定义原文（InnoDB 并发闸门，均在 `storage/innobase/handler/ha_innodb.cc`）

> 前置知识：InnoDB 插件变量**注册时不写 `innodb_` 前缀**，前缀由 server 在 `register_var()` 里拼（`sql/sql_plugin.cc:2827-2830`）：

```c
  varname = ((char *)my_alloca(length));
  strxmov(varname + 1, plugin, "_", name, NullS);
  for (p = varname + 1; *p; p++)
    if (*p == '-') *p = '_';
```

**闸门 1：进入 InnoDB 的门票**

```c
static MYSQL_SYSVAR_ULONG(thread_concurrency, srv_thread_concurrency,
                          PLUGIN_VAR_RQCMDARG,
                          "Helps in performance tuning in heavily concurrent "
                          "environments. Sets the maximum number of threads "
                          "allowed inside InnoDB. Value 0 will disable the "
                          "thread throttling.",
                          nullptr, innodb_thread_concurrency_update, 0, 0, 1000,
                          0);
```

```c
static MYSQL_SYSVAR_ULONG(concurrency_tickets, srv_n_free_tickets_to_enter,
                          PLUGIN_VAR_RQCMDARG,
                          "Number of times a thread is allowed to enter InnoDB "
                          "within the same SQL query after it has once got the "
                          "ticket",
                          nullptr, nullptr, 5000L, 1L, UINT32_MAX, 0);
```

排队退避两个：

```c
static MYSQL_SYSVAR_ULONG(
    adaptive_max_sleep_delay, srv_adaptive_max_sleep_delay, PLUGIN_VAR_RQCMDARG,
    "The upper limit of the sleep delay in usec. Value of 0 disables it.",
    nullptr, nullptr, 150000, /* Default setting */
    0,                        /* Minimum value */
    1000000, 0);              /* Maximum value */

static MYSQL_SYSVAR_ULONG(
    thread_sleep_delay, srv_thread_sleep_delay, PLUGIN_VAR_RQCMDARG,
    "Time of innodb thread sleeping before joining InnoDB queue (usec)."
    " Value 0 disable a sleep",
    nullptr, nullptr, 10000L, 0L, 1000000L, 0);
```

**自旋三件套**：

```c
static MYSQL_SYSVAR_ULONG(
    sync_spin_loops, srv_n_spin_wait_rounds, PLUGIN_VAR_RQCMDARG,
    "Count of spin-loop rounds in InnoDB mutexes (30 by default)", nullptr,
    nullptr, 30L, 0L, UINT32_MAX, 0);

static MYSQL_SYSVAR_ULONG(
    spin_wait_delay, srv_spin_wait_delay, PLUGIN_VAR_OPCMDARG,
    "Maximum delay between polling for a spin lock (6 by default)", nullptr,
    nullptr, 6L, 0L, 1000, 0);

static MYSQL_SYSVAR_ULONG(spin_wait_pause_multiplier,
                          ut::spin_wait_pause_multiplier, PLUGIN_VAR_RQCMDARG,
                          "Controls how many times in a row to use a PAUSE "
                          "instruction to achieve one unit of delay in a spin "
                          "lock (see @@innodb_spin_wait_delay), defaults to 50",
                          nullptr, nullptr, 50, 0, 100, 0);
```

**闸门 2：锁等待与死锁**

```c
static MYSQL_SYSVAR_BOOL(
    deadlock_detect, innobase_deadlock_detect, PLUGIN_VAR_NOCMDARG,
    "Enable/disable InnoDB deadlock detector (default ON)."
    " if set to OFF, deadlock detection is skipped,"
    " and we rely on innodb_lock_wait_timeout in case of deadlock.",
    nullptr, innobase_deadlock_detect_update, true);
```

```c
static MYSQL_THDVAR_ULONG(lock_wait_timeout, PLUGIN_VAR_RQCMDARG,
                          "Timeout in seconds an InnoDB transaction may wait "
                          "for a lock before being rolled back. Values above "
                          "100000000 disable the timeout.",
                          nullptr, nullptr, 50, 1, 1024 * 1024 * 1024, 0);
```

/* 【注】★ `innodb_lock_wait_timeout` 是 **THDVAR**（不是 SYSVAR）——这就是它能
   `SET SESSION` 的原因；而 `innodb_thread_concurrency` 等是 SYSVAR，只能 `SET GLOBAL`。 */

**闸门 3：后台线程并行度（三个都是只读，`PLUGIN_VAR_READONLY`）**

```c
static MYSQL_SYSVAR_ULONG(sync_array_size, srv_sync_array_size,
                          PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,
                          "Size of the mutex/lock wait array.", nullptr,
                          nullptr, 1, /* Default setting */
                          1,          /* Minimum value */
                          1024, 0);   /* Maximum value */

static MYSQL_SYSVAR_ULONG(purge_threads, srv_n_purge_threads,
                          PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,
                          "Purge threads can be from 1 to 32. Default is 4.",
                          nullptr, nullptr, 4,   /* Default setting */
                          1,                     /* Minimum value */
                          MAX_PURGE_THREADS, 0); /* Maximum value */

static MYSQL_SYSVAR_ULONG(
    page_cleaners, srv_n_page_cleaners,
    PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,
    "Page cleaner threads can be from 1 to 64. Default is 4.", nullptr, nullptr,
    4, 1, 64, 0);

static MYSQL_SYSVAR_ULONG(read_io_threads, srv_n_read_io_threads,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
                          "Number of background read I/O threads in InnoDB.",
                          nullptr, nullptr, 4, 1, 64, 0);

static MYSQL_SYSVAR_ULONG(write_io_threads, srv_n_write_io_threads,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY,
                          "Number of background write I/O threads in InnoDB.",
                          nullptr, nullptr, 4, 1, 64, 0);
```

★ **`innodb_page_cleaners` 被 `innodb_buffer_pool_instances` 截断的代码位置**（`ha_innodb.cc:5084-5088`，在 `innodb_init_params()` 内）：

```c
  if (srv_n_page_cleaners > srv_buf_pool_instances) {
    /* limit of page_cleaner parallelizability
    is number of buffer pool instances. */
    srv_n_page_cleaners = srv_buf_pool_instances;
  }
```

/* 【注】"只读"的判据就是 flags 里的 **`PLUGIN_VAR_READONLY`**——因为这些数组/线程池
   在启动时按此值一次性分配并常驻（如 sync wait array），运行中改不了。 */

### PFS 观测容量（都是 `READ_ONLY`）

```c
static Sys_var_ulong Sys_pfs_max_thread_classes(
    "performance_schema_max_thread_classes",
    "Maximum number of thread instruments.",
    READ_ONLY GLOBAL_VAR(pfs_param.m_thread_class_sizing),
    CMD_LINE(REQUIRED_ARG), VALID_RANGE(0, 1024), DEFAULT(PFS_MAX_THREAD_CLASS),
    BLOCK_SIZE(1), PFS_TRAILING_PROPERTIES);

static Sys_var_long Sys_pfs_max_thread_instances(
    "performance_schema_max_thread_instances",
    "Maximum number of instrumented threads."
    " Use 0 to disable, -1 for automated scaling.",
    READ_ONLY GLOBAL_VAR(pfs_param.m_thread_sizing), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(-1, 1024 * 1024), DEFAULT(PFS_AUTOSCALE_VALUE), BLOCK_SIZE(1),
    PFS_TRAILING_PROPERTIES);
```

```c
#define PFS_AUTOSCALE_VALUE (-1)
#define PFS_AUTOSIZE_VALUE (-1)
```

/* 【注】★ `-1 = autoscale` 是这两个常量表达的；`VALID_RANGE(-1, 1024*1024)` ⇒
   上限是 **1048576**，`storage/perfschema/**` 全量 grep `65536` 无命中。 */

### `srv_max_n_threads`：不是系统变量的硬上限

```c
/** We are prepared for a situation that we have this many threads waiting for
a semaphore inside InnoDB. innodb_init_params() sets the value. */
uint32_t srv_max_n_threads = 0;
```

赋值处（`ha_innodb.cc:5027-5032`）：

```c
  /* Set the maximum number of threads which can wait for a semaphore
  inside InnoDB: this is the 'sync wait array' size, as well as the
  maximum number of threads that can wait in the 'srv_conc array' for
  their time to enter InnoDB. */

  srv_max_n_threads = 100 * 1024;
```

/* 【注】★ 注释明说它**同时**约束两样东西：sync wait array 的大小 与 srv_conc 排队数组的上限。
   消费点：`lock0lock.cc:305 / :398`、`lock0wait.cc:148`（`for (uint32_t i = srv_max_n_threads; i--; ++slot)`）。 */

---

## 可观测性

### ★ 一次"连接打满 / 线程卡死"的排查路径

**Step 0 — 确认是不是真打满**

```sql
SHOW GLOBAL STATUS LIKE 'Threads_%';        -- connected / running / created / cached
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;                            -- 看 State / Time / Command
```

对应源码计数器：`Global_THD_manager::atomic_num_thread_running`（Threads_running）、`atomic_thread_created`、`atomic_global_thd_count`。下面把这条排查路径上每个数字与每一步的**源码落点**逐条钉住。

#### 计数器的声明与四个 `Threads_*` 的口径

`sql/mysqld_thd_manager.h:347-381`：

```cpp
  // Declared static as it is referenced in handle_fatal_signal()
  static std::atomic<uint> atomic_global_thd_count;

  // Number of THD list partitions.
  static const int NUM_PARTITIONS = 8;
```

```cpp
  // Array of current THDs. Protected by LOCK_thd_list.
  typedef Prealloced_array<THD *, 60> THD_array;
  THD_array thd_list[NUM_PARTITIONS];
  ...
  // Count of active threads which are running queries in the system.
  std::atomic<int> atomic_num_thread_running;

  // Cumulative number of threads created by mysqld daemon.
  std::atomic<ulonglong> atomic_thread_created;
```

/* 【注】★ `NUM_PARTITIONS = 8` —— THD 链表和它的锁都是** 8 个分区**（`LOCK_thd_list[8]`），
   不是单个全局锁。这是 `Threads_*` 能在高并发下被频繁读写的前提。 */

`Threads_*` 在 `status_vars[]` 里的条目（`sql/mysqld.cc:10027-10035`）：

```cpp
    {"Threads_cached",
     (char *)&Per_thread_connection_handler::blocked_pthread_count,
     SHOW_LONG_NOFLUSH, SHOW_SCOPE_GLOBAL},
    {"Threads_connected", (char *)&Connection_handler_manager::connection_count,
     SHOW_INT, SHOW_SCOPE_GLOBAL},
    {"Threads_created", (char *)&show_num_thread_created, SHOW_FUNC,
     SHOW_SCOPE_GLOBAL},
    {"Threads_running", (char *)&show_num_thread_running, SHOW_FUNC,
     SHOW_SCOPE_GLOBAL},
```

只有 running / created 走回调读原子量（`sql/mysqld.cc:9379-9395`）：

```cpp
static int show_num_thread_running(THD *, SHOW_VAR *var, char *buff) {
  var->type = SHOW_LONGLONG;
  var->value = buff;
  long long *value = reinterpret_cast<long long *>(buff);
  *value = static_cast<long long>(
      Global_THD_manager::get_instance()->get_num_thread_running());
  return 0;
}

static int show_num_thread_created(THD *, SHOW_VAR *var, char *buff) {
  var->type = SHOW_LONG;
  var->value = buff;
  long *value = reinterpret_cast<long *>(buff);
  *value = static_cast<long>(
      Global_THD_manager::get_instance()->get_num_thread_created());
  return 0;
}
```

★ **四个数字的口径要分清**：

| 状态量 | 读的是 | 语义 |
|---|---|---|
| `Threads_cached` | `Per_thread_connection_handler::blocked_pthread_count` | 缓存里**阻塞等待的 OS 线程数**（不是 THD 数） |
| `Threads_connected` | `Connection_handler_manager::connection_count` | 已建连数 |
| `Threads_created` | `atomic_thread_created`（`SHOW_FUNC`） | 累计**创建 OS 线程**次数（`inc_thread_created()` 唯一业务调用点在 `connection_handler_per_thread.cc:436`） |
| `Threads_running` | `atomic_num_thread_running`（`SHOW_FUNC`） | 正在跑命令的线程数（`sql_parse.cc:1769` 加、`:2505` 减） |

/* 【注】★ `Threads_running` 高 ≠ 连接数高——它在 `do_command()` 入口加、命令结束减，
   所以用来区分"连接堆积"与"查询堆积"。事件调度器线程也维护这个计数（`event_scheduler.cc:175 / :195`）。 */

#### Step 2：`Threads_created` 飞涨 = 线程缓存没命中

缓存命中的判据只有一行（`sql/conn_handler/connection_handler_per_thread.cc:384-399`）：

```cpp
bool Per_thread_connection_handler::check_idle_thread_and_enqueue_connection(
    Channel_info *channel_info) {
  bool res = true;

  mysql_mutex_lock(&LOCK_thread_cache);
  if (Per_thread_connection_handler::blocked_pthread_count > wake_pthread) {
    DBUG_PRINT("info", ("waiting_channel_info_list->push %p", channel_info));
    waiting_channel_info_list->push_back(channel_info);
    wake_pthread++;
    mysql_cond_signal(&COND_thread_cache);
    res = false;
  }
  mysql_mutex_unlock(&LOCK_thread_cache);

  return res;
}
```

未命中才真的建线程（`connection_handler_per_thread.cc:401-437`）：

```cpp
bool Per_thread_connection_handler::add_connection(Channel_info *channel_info) {
  int error = 0;
  my_thread_handle id;

  DBUG_TRACE;

  // Simulate thread creation for test case before we check thread cache
  DBUG_EXECUTE_IF("fail_thread_create", error = 1; goto handle_error;);

  if (!check_idle_thread_and_enqueue_connection(channel_info)) return false;

  /*
    There are no idle threads available to take up the new
    connection. Create a new thread to handle the connection
  */
  channel_info->set_prior_thr_create_utime();
  error =
      mysql_thread_create(key_thread_one_connection, &id, &connection_attrib,
                          handle_connection, (void *)channel_info);
  ...
  Global_THD_manager::get_instance()->inc_thread_created();
  DBUG_PRINT("info", ("Thread created"));
  return false;
}
```

空闲线程侧"怎么睡、被谁叫醒"（`connection_handler_per_thread.cc:367-405`）：

```cpp
Channel_info *Per_thread_connection_handler::block_until_new_connection() {
  Channel_info *new_conn = nullptr;
  mysql_mutex_lock(&LOCK_thread_cache);
  if (blocked_pthread_count < max_blocked_pthreads && !shrink_cache) {
    /* Don't kill the pthread, just block it for reuse */
    DBUG_PRINT("info", ("Blocking pthread for reuse"));

    /*
      mysys_var is bound to the physical thread,
      so make sure mysys_var->dbug is reset to a clean state
      before picking another session in the thread cache.
    */
    DBUG_POP();
    assert(!_db_is_pushed_());

    blocked_pthread_count++;
    while (!connection_events_loop_aborted() && !wake_pthread && !shrink_cache)
      mysql_cond_wait(&COND_thread_cache, &LOCK_thread_cache);
    blocked_pthread_count--;

    if (shrink_cache && blocked_pthread_count <= max_blocked_pthreads) {
      mysql_cond_signal(&COND_flush_thread_cache);
    }

    if (wake_pthread) {
      wake_pthread--;
      if (!waiting_channel_info_list->empty()) {
        new_conn = waiting_channel_info_list->front();
        waiting_channel_info_list->pop_front();
      } else {
        assert(0);  // We should not get here.
      }
    }
  }
  mysql_mutex_unlock(&LOCK_thread_cache);
  return new_conn;
}
```

/* 【注】① 缓存不是"THD 链表"，而是 **`waiting_channel_info_list`（待处理的连接）+ `wake_pthread`（已被唤醒计数）**；
   ② `DBUG_POP()` 那一段是复用路径特有的陷阱——`mysys_var` 绑定在物理线程上，
   复用前必须把上一个会话残留的 dbug 状态清掉；
   ③ `wake_pthread` 是"正在被唤醒的线程数"，所以命中判据是
   `blocked_pthread_count > wake_pthread`（不是 `> 0`）——突发连接会把 wake 打满。 */

#### Step 3：`SHOW PROCESSLIST` 为什么不会长时间阻塞新连接

`sql/sql_show.cc:2961-3004` 的入口与遍历：

```cpp
  if (!thd->killed) {
    thread_infos.reserve(Global_THD_manager::get_instance()->get_thd_count());
    List_process_list list_process_list(user, &thread_infos, thd,
                                        max_query_length);
    Global_THD_manager::get_instance()->do_for_all_thd_copy(&list_process_list);
  }

  // Return list sorted by thread_id.
  std::sort(thread_infos.begin(), thread_infos.end(), thread_info_compare());
```

关键在 `do_for_all_thd_copy()`（`sql/mysqld_thd_manager.cc:290-319`）：

```cpp
void Global_THD_manager::do_for_all_thd_copy(Do_THD_Impl *func) {
  Do_THD doit(func);

  for (int i = 0; i < NUM_PARTITIONS; i++) {
    MUTEX_LOCK(lock_remove, &LOCK_thd_remove[i]);
    mysql_mutex_lock(&LOCK_thd_list[i]);

    /* Take copy of global_thread_list. */
    THD_array thd_list_copy(thd_list[i]);

    /*
      Allow inserts to global_thread_list. Newly added thd
      will not be accounted for when executing func.
    */
    mysql_mutex_unlock(&LOCK_thd_list[i]);

    /* Execute func for all existing threads. */
    std::for_each(thd_list_copy.begin(), thd_list_copy.end(), doit);

    DEBUG_SYNC_C("inside_do_for_all_thd_copy");
  }
}

void Global_THD_manager::do_for_all_thd(Do_THD_Impl *func) {
  Do_THD doit(func);
  for (int i = 0; i < NUM_PARTITIONS; i++) {
    MUTEX_LOCK(lock, &LOCK_thd_list[i]);
    std::for_each(thd_list[i].begin(), thd_list[i].end(), doit);
  }
}
```

/* 【注】★ `_copy` 版本：**拷快照 → 立刻放 `LOCK_thd_list`**，只保留 `LOCK_thd_remove`
   防止 THD 被析构；所以 PROCESSLIST 期间新连接仍能注册。
   而 `do_for_all_thd`（非 copy）是**全程持分区锁**遍历——`KILL` 走的是 `find_thd` 那条分区锁路径。 */

#### Step 4：为什么线程复用必须重绑 TLS

`sql/sql_class.cc:1681-1729`：

```cpp
/*
  Remember the location of thread info, the structure needed for
  (*THR_MALLOC)->Alloc() and the structure for the net buffer
*/

void THD::store_globals() {
  /*
    Assert that thread_stack is initialized: it's necessary to be able
    to track stack overrun.
  */
  assert(thread_stack);

  current_thd = this;
  THR_MALLOC = &mem_root;
  /*
    is_killable is concurrently readable by a killer thread.
    It is protected by LOCK_thd_data, it is not needed to lock while the
    value is changing from false not true. If the kill thread reads
    true we need to ensure that the thread doesn't proceed to assign
    another thread to the same TLS reference.
  */
  is_killable = true;
  ...
  real_id = my_thread_self();
}

/*
  Remove the thread specific info (THD and mem_root pointer) stored during
  store_global call for this thread.
*/
void THD::restore_globals() {
  // Remove reference to specific OS thread.
  real_id = 0;
  ...
  /* Undocking the thread specific data. */
  current_thd = nullptr;
  THR_MALLOC = nullptr;
}
```

/* 【注】`current_thd` / `THR_MALLOC` / `real_id` 都是 **TLS**，绑在物理线程上。
   `handle_connection` 的 for 循环每次都 `init_new_thd()` 造**新 THD**，若不重新
   `store_globals()`，TLS 会指向已被 `delete` 的旧 THD ⇒ `current_thd` 悬空。 */

#### 兜底：`Too many connections` 是从哪抛的

限流判定（`sql/conn_handler/connection_handler_manager.cc:104-129`）：

```cpp
bool Connection_handler_manager::check_and_incr_conn_count(
    bool is_admin_connection) {
  bool connection_accepted = true;
  mysql_mutex_lock(&LOCK_connection_count);
  /*
    Here we allow max_connections + 1 clients to connect
    (by checking before we increment by 1).

    The last connection is reserved for SUPER users. This is
    checked later during authentication where valid_connection_count()
    is called for non-SUPER users only.
  */
  if (connection_count > max_connections && !is_admin_connection) {
    connection_accepted = false;
    m_connection_errors_max_connection++;
  } else {
    ++connection_count;

    if (connection_count > max_used_connections) {
      max_used_connections = connection_count;
      max_used_connections_time = time(nullptr);
    }
  }
  mysql_mutex_unlock(&LOCK_connection_count);
  return connection_accepted;
}
```

抛出点（`connection_handler_manager.cc:254-267`）：

```cpp
void Connection_handler_manager::process_new_connection(
    Channel_info *channel_info) {
  if (connection_events_loop_aborted() ||
      !check_and_incr_conn_count(channel_info->is_admin_connection())) {
    channel_info->send_error_and_close_channel(ER_CON_COUNT_ERROR, 0, true);
    delete channel_info;
    return;
  }

  if (m_connection_handler->add_connection(channel_info)) {
    inc_aborted_connects();
    delete channel_info;
  }
}
```

/* 【注】★ 报 "Too many connections"（ER_CON_COUNT_ERROR, SQLSTATE 08004）说明
   **根本没进 `add_connection`**——既不消耗线程缓存，也不会 `inc_thread_created()`。
   ★ 另一个常被忽略的细节：注释明写 `max_connections + 1`——**最后一个连接位留给 SUPER**，
   所以普通用户先失败，SUPER 仍能连进来排查。 */

**Step 1 — `Threads_connected == max_connections`：接不进去**

- `Command='Sleep'` 且 Time 巨大 → 应用没关连接 / 连接池泄漏
- `Threads_created` 猛涨 + `thread_cache_size` 太小 → 调大它

**Step 2 — `Threads_running` 高但 `Threads_connected` 不高：卡在锁上**（最有价值的一步）

```sql
SELECT t.processlist_id, t.processlist_state,
       w.event_name, w.object_schema, w.object_name, w.source,
       w.timer_wait/1e9 AS wait_ms
FROM performance_schema.events_waits_current w
JOIN performance_schema.threads t USING (thread_id)
WHERE w.event_name NOT LIKE 'idle%' AND w.event_name NOT LIKE 'wait/io/socket%'
ORDER BY wait_ms DESC;
```

★ **按 `event_name` 前缀判定卡在哪一层**：

| `event_name` 前缀 | 卡在哪 | 挂靠结构 |
|---|---|---|
| `wait/synch/mutex/innodb/...` | InnoDB latch | `sync_cell_t` |
| `wait/synch/rwlock/innodb/...` | InnoDB rw_lock | `sync_cell_t` |
| `wait/synch/mutex/sql/...` | server mutex（`LOCK_open`、`LOCK_thd_list`…） | mysql_mutex |
| `wait/lock/metadata/sql/mdl` | **MDL** | `MDL_wait` |
| `wait/lock/table/sql/handler` | 引擎层表锁 | handler |
| `wait/io/file/...` | IO | AIO |
| `idle` | 空闲（等网络包） | `do_command` |

**Step 3 — 分层定位**

```sql
SELECT * FROM performance_schema.metadata_locks WHERE owner_thread_id IN (...);  -- MDL 谁等谁
SELECT * FROM performance_schema.data_locks;                                     -- InnoDB 行锁
SELECT * FROM performance_schema.data_lock_waits;                                -- 谁等谁
```

**Step 4 — ★ latch 层 PFS 看不到，只能靠 InnoDB 自己**

```sql
SHOW ENGINE INNODB STATUS;   -- SEMAPHORES 段
```

由 `sync_array_print()` / `sync_array_print_long_waits()` 输出（等 latch 超过 **4 分钟**开始往 stderr 打警告）。

---

## Misc

### 易混淆概念

- **`max_connections` vs `innodb_thread_concurrency`**：前者拒绝连接（线程维度），后者让已连上的线程 sleep（并发维度）。**后者默认关闭**。
- **`innodb_thread_sleep_delay` 跟 latch 自旋无关**：它是"进 InnoDB 排队"的 sleep；latch 自旋用的是 `innodb_spin_wait_delay` + `innodb_sync_spin_loops`。
- **等锁线程不占并发名额**：否则会自死锁（`srv_conc_force_exit_innodb` 就是干这个的）。
- **latch 等待没有超时**：只有 debug-only 的 `innodb_fatal_semaphore_wait_threshold`（600s）会 crash。
- **`thd_wait_begin()` 不是 PFS 埋点**：在 one-thread-per-connection 下它是**空操作**（只有线程池插件才实现它）。很多人误以为它等价于 PFS 的行锁等待事件——不是。

### ★ 已知坑（源码证据逐条）

1. **★ latch 死锁在 release 下不会自解**：只会 4 分钟警告 + 连续 10 次同一 sema 后 `ib::fatal` crash。
2. ★ **唤醒算法不完美**：需要 `srv_error_monitor_thread` 每秒兜底扫 sync array 补唤醒（注释承认"race condition"）。
3. ★ **`os_event` 的 ABA 丢信号问题**：靠 `signal_count` 解决，正确性需要 120 行内存模型证明。
4. ★ **`sync_array` 全部数组满 → `ut_a(*cell != nullptr)` 直接 abort**。
5. ★ **`innodb_thread_concurrency` 过大有 redo 安全隐患**（源码打 warning）。
6. **★ PFS 线程实例耗尽静默降级**，只能靠 `Performance_schema_thread_instances_lost` 发现。
7. **★ `srv_pre_dd_shutdown` 里用户线程未清零会直接 crash**（`ut_a(srv_conc_get_active_threads() == 0)`）。
8. ★ **新增 InnoDB 线程必须自己登记退出路径**（`/* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM HERE OR EARLIER */`）。
9. **`sync_array_t::mutex` 必须是 OS mutex**，否则会无限递归进 sync array。
10. **`my_thread_self_setname` 的副作用**：每次线程复用都会改 OS 线程名（变成 PFS 的 `os_name`，不是 `mysqld`）；且**绝不能改 main 线程名**（否则 `ps -C mysqld` 失效）。
11. **purge worker 必须在 coordinator 之后退出**，否则 coordinator 永远等 `trx_purge_wait_for_workers_to_complete()`。
12. **`innodb_page_cleaners` 会被 `innodb_buffer_pool_instances` 截断**。
13. **死锁检测 OFF→ON 需要"kick-start"**：因为 OFF 期间等待图不再变化，检测器会观察不到动作，必须手动 `lock_wait_request_check_for_cycles()` 踢一下。
14. **latch order 检查只在 debug 构建生效**；且 `_NODE`/`_LEAF` 级别有已知盲区（B 树高度变化会让 leaf/node 漂移）。
15. **`PFS_SKIP_BUFFER_MUTEX_RWLOCK`**：缓冲池的 block mutex/rwlock 是热点中的热点，可在编译时从 PFS 插桩里摘掉。

## 参考

**源码**

- `sql/conn_handler/connection_handler_per_thread.cc`：用户线程与线程缓存
- `sql/mysqld_thd_manager.{h,cc}`：`Global_THD_manager`（8 分区锁、thread_id 复用池）
- `sql/sql_class.{h,cc}`：`store_globals()` / `restore_globals()` / `release_resources()`
- `sql/mdl.cc`：`MDL_wait::timed_wait`
- `storage/innobase/include/srv0srv.h`：`Srv_threads`
- `storage/innobase/srv/srv0start.cc`：线程创建、`srv_pre_dd_shutdown()`、`srv_shutdown()`
- `storage/innobase/include/srv0shutdown.h`：九阶段状态机（含每阶段"为什么"的注释）
- `storage/innobase/include/os0thread.h` / `os0thread-create.h`：`IB_thread`、创建与放行分离
- `storage/innobase/srv/srv0conc.cc`：`srv_conc_t`、ticket 与自适应 sleep
- `storage/innobase/include/sync0arr_impl.h`：SyncArray（含文件头的"活化石"注释）
- `storage/innobase/include/ut0mutex.ic`：Theorem 1 的形式化证明
- `storage/innobase/lock/lock0wait.cc`：`lock_wait_timeout_thread`、`lock_wait_suspend_thread`
- `storage/innobase/srv/srv0srv.cc`：`srv_error_monitor_thread`（1Hz 兜底）
- `storage/innobase/handler/ha_innodb.cc`：`all_innodb_threads[]`、`all_innodb_mutexes[]`、`all_innodb_rwlocks[]`
- `storage/perfschema/pfs.cc` / `pfs_instr.cc`：`pfs_spawn_thread`、`create_thread`、`destroy_thread`

**官方文档**（★ 官方手册**没有**一个"InnoDB 后台线程清单"页——17.11.1 InnoDB Disk I/O 只讲异步 I/O 与预读，线程逐个的规范只有源码；以下是分散在各配置页里的权威出处）

| 主题 | 页面 | URL |
|---|---|---|
| **并发度控制（最核心）** | 17.8.4 Configuring Thread Concurrency for InnoDB | https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-thread_concurrency.html |
| 后台 IO 线程 | 17.8.5 Configuring the Number of Background InnoDB I/O Threads | https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-multiple_io_threads.html |
| Linux native AIO | 17.8.6 Using Asynchronous I/O on Linux | https://dev.mysql.com/doc/refman/8.0/en/innodb-linux-native-aio.html |
| Purge 配置 | 17.8.9 Purge Configuration | https://dev.mysql.com/doc/refman/8.0/en/innodb-purge-configuration.html |
| 刷脏 / page cleaner | 17.8.3.5 Configuring Buffer Pool Flushing | https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool-flushing.html |
| InnoDB Disk I/O | 17.11.1 InnoDB Disk I/O | https://dev.mysql.com/doc/refman/8.0/en/innodb-disk-io.html |
| InnoDB 变量总表 | 17.14 InnoDB Startup Options and System Variables | https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html |
| **线程池（企业版）** | 7.6.3 MySQL Enterprise Thread Pool | https://dev.mysql.com/doc/refman/8.0/en/thread-pool.html |
| 关闭流程（官方 6 步） | 7.1.19 The Server Shutdown Process | https://dev.mysql.com/doc/refman/8.0/en/server-shutdown.html |
| PFS threads 表 | 29.12.21.8 The threads Table | https://dev.mysql.com/doc/refman/8.0/en/performance-schema-threads-table.html |
| 线程预过滤 | 29.4.6 Pre-Filtering by Thread | https://dev.mysql.com/doc/refman/8.0/en/performance-schema-thread-filtering.html |
| 线程信息排查 | 10.14 Examining Server Thread (Process) Information | https://dev.mysql.com/doc/refman/8.0/en/thread-information.html |
| InnoDB Monitor | 17.17.2 Enabling InnoDB Monitors | https://dev.mysql.com/doc/refman/8.0/en/innodb-enabling-monitors.html |
| 8.0 新增/弃用/移除清单 | 1.4 Server and Status Variables Added/Deprecated/Removed | https://dev.mysql.com/doc/refman/8.0/en/added-deprecated-removed.html |
| 8.4 相对 8.0 的变化 | 1.4 What Is New in MySQL 8.4 since MySQL 8.0 | https://dev.mysql.com/doc/refman/8.4/en/mysql-nutshell.html |

★ **两条官方原话值得直接引用**（它们给出了源码里没有的"立场"）：

1. 关于并发度（17.8.4）：
   > *"On modern operating systems and servers with multi-core processors, where context switching is efficient, most workloads run well without any limit on the number of concurrent threads."*

   文档**没有**说 `innodb_thread_concurrency` 已弃用，措辞是"多数负载无需限制"——配合 4.4 节"中心锁已被分片"的事实，可以把它理解成：**8.0 已不再需要用它来缓解争用**。

2. 关于默认线程模型（7.6.3 线程池页）：
   > *"The default thread-handling model in MySQL Server executes statements using one thread per client connection."*

   官方同时点出了该模型的三个代价：**thread stack 冲击 CPU cache、上下文切换开销、InnoDB 中心 mutex 争用**——这正是企业版线程池要解决的问题，也是社区版没有线程池（只有 `plugin_connection_handler.h` 接口）的原因。

**相关文档**

- server 层连接线程细节：[`../server/infra/connection_threading.md`](connection_threading.md)
- 锁原语：[`../lock/primitives/`](../lock/primitives/)
- InnoDB 事务锁：[`lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)
- PFS 事件与表：[`../server/infra/pfs.md`](../observability/pfs/pfs.md)
- 启动/关闭时序：[`../server/lifecycle/02_startup.md`](../../server/lifecycle/02_startup.md)、[`03_shutdown.md`](../../server/lifecycle/03_shutdown.md)
