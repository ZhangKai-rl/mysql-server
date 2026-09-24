# Server 关闭流程深度解析

> 基于 MySQL 8.0.39 源码，覆盖关闭的全链路：三条入口（`SHUTDOWN` SQL / 信号 / `mysqladmin`）→ 汇聚到信号线程 → `close_connections()`（温和两轮 kill + 强制断开 + 无限等待）→ 主线程 `clean_up()`（逆序卸载）→ InnoDB 九态关闭状态机 → `mysqld_exit()`。
>
> **边界**：本篇讲**运行期关闭与退出**；
> 启动失败走的 `unireg_abort()` 属于启动链路，见 [`02_startup.md`](02_startup.md)；
> `--initialize` 用 abort 路径走成功退出码，见 [`01_initialize.md`](01_initialize.md)；
> 崩溃恢复（关闭不干净的后果）见 [`../../innodb/recovery.md`](../../innodb/recovery.md)；
> redo 与 checkpoint 机制本身见 [`../../innodb/redo_log.md`](../../innodb/redo_log.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 的关闭是一个**三阶段过程**，且**由一个专门的信号线程驱动第一阶段**：

1. **阶段 1 `close_connections()`**——在信号线程里执行：停接受、温和 kill 所有连接、停复制、强制断 vio、无限等待所有 THD 退出。
2. **阶段 2 `clean_up()`**——回到主线程执行：逆序卸载各子系统，其中 `plugin_shutdown()` 触发 InnoDB 真正刷盘。
3. **阶段 3 `mysqld_exit()`**——清 mutex、`my_end()`、关 error log、`exit()`。

### 用途

关闭要在三个约束之间取平衡：

- **数据安全**：已提交事务必须落盘（`innodb_fast_shutdown=0/1` 时写 `flushed_lsn`，下次启动判定为干净关闭）。
- **客户端体验**：先给连接一个"优雅退出"的机会（`KILL_CONNECTION` 标记），不行再强制断 socket。
- **可托管**：退出码要能表达"别重启我"（1）/"请重启我"（16），配合 systemd / `mysqld_safe`。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | 用全局 `volatile bool abort_loop` 作为关闭标志；InnoDB 关闭是 `innobase_end` / `logs_empty_and_mark_files_at_shutdown`；error log 关闭用 `release_error_log` |
| 8.0 | ① `abort_loop` **被删除**，改为 `connection_events_loop_aborted_flag`（`std::atomic<int32>`）+ `connection_events_loop_aborted()`；② InnoDB 关闭重构为 **9 态 `srv_shutdown_t` 状态机**（`srv0shutdown.h`），并拆出 `srv_pre_dd_shutdown()`（DD 关闭前先停掉所有会访问 DD 的后台线程）；③ 引入 `handlerton::pre_dd_shutdown` 回调，使"DD 关闭"成为关闭序列中一个**显式的屏障**；④ 新增 `MYSQLD_RESTART_EXIT = 16` 支持 `RESTART` 语句；⑤ error log 关闭改为 `destroy_error_log()` |
| 8.0.19+ | `sd_notify` 的 `STOPPING=1` / `STATUS=Server shutdown complete` |
| 8.0.30 | redo 改版后，`srv_shutdown_log()` 里的 checkpoint 走 `log_make_empty_and_stop_background_threads()`；新增 `log_stop_background_threads_nowait()` 用于兜底 |
| 8.4 / 9.x | 关闭骨架不变；`innodb_fast_shutdown` 仍为 0/1/2 三档 |

---

## 理论基础

### 设计思想与权衡

#### 一、为什么三条关闭入口都要汇聚到"信号线程"

```
SHUTDOWN SQL ──┐
kill -TERM   ──┼──→ 信号线程 signal_hand() ──→ close_connections() ──→ 线程退出
mysqladmin   ──┘                                                          │
                                                                          ↓
                                                        主线程 join → clean_up() → mysqld_exit()
```

`SHUTDOWN` 语句的实现是**给自己发一个 SIGTERM**：

```cpp
void kill_mysql(void) {
  if (!mysqld_server_started) {
    mysqld_process_must_end_at_startup = true;
    return;
  }
#if defined(_WIN32)
  { if (!SetEvent(hEventShutdown)) { ... } }
#else
  if (pthread_kill(signal_thread_id.thread, SIGTERM)) { ... }
#endif
}
```

**为什么不直接在 SQL 线程里调 `close_connections()`**：

1. **关闭不可重入**。若 `SHUTDOWN` 线程直接执行关闭，第二个 `SHUTDOWN` 或同时到达的 SIGTERM 会重入。汇聚到单点后，`if (!connection_events_loop_aborted())` 这一句就完成了幂等保护。
2. **不能在业务线程里 kill 自己**。`close_connections()` 会遍历并 kill 所有 THD，包括正在执行 `SHUTDOWN` 的这个。在专用线程里做，逻辑上干净。
3. **信号线程本来就是那个"只做控制面"的线程**。它 `sigwaitinfo()` 阻塞等待，天然适合当关闭的执行者。

**代价**：SQL 线程发完信号就返回了，它**不知道**关闭是否完成。`mysqladmin shutdown` 因此要靠"等 pid 文件消失"来判断（超时默认 3600 秒）。

#### 二、为什么 `close_connections()` 要 kill 两轮

```
第一轮 Set_kill_conn    只打 KILL_CONNECTION 标记，不断 socket
   ↓  sleep(2) 宽限
第二轮 Call_close_conn  打 ER_FORCE_CLOSE_THREAD 警告 + close_connection()（断 vio）
```

**第一轮为什么不断 socket**：连接线程可能正处于一个长事务的中间。给它一个 `killed` 标记，让它自己走到检查点后**优雅回滚并给客户端发错误**——这样客户端能收到"服务器正在关闭"而不是一个突兀的 EOF。

**第二轮为什么必须断 socket**：有些线程阻塞在 `net_read()` 上等客户端发下一条命令。它们永远等不到 `killed` 检查点（因为根本没在执行语句）。唯一能唤醒它们的方式是**关掉 vio**，让 `read()` 返回 EOF。

**为什么是 `sleep(2)` 这个魔数**：源码里就是硬编码 `if (thd_manager->get_thd_count() > 0) sleep(2);`，没有配置项。这是权衡的结果——给一个固定的、足够短的宽限期，而不是引入一个可调但没人会调的参数。

**dump 线程是特例中的特例**：

```cpp
  if (set_kill_conn.get_dump_thread_count()) {
    /*
      Replication dump thread should be terminated after the clients are
      terminated. Wait for few more seconds for other sessions to end.
     */
    while (thd_manager->get_thd_count() > dump_thread_count &&
           dump_thread_kill_retries) {
      sleep(1);
      dump_thread_kill_retries--;
    }
    set_kill_conn.set_dump_thread_flag();
    thd_manager->do_for_all_thd(&set_kill_conn);
  }
```

`COM_BINLOG_DUMP` / `COM_BINLOG_DUMP_GTID` 在第一轮被**跳过并计数**，等其余会话结束后（最多等 8 秒）才在第二轮杀掉。原因：replica 依赖 dump 线程收到一个**正常的 EOF/错误**才能干净地记录位点并重连；如果 dump 线程先死，replica 会看到一个突兀的断连。

#### 三、为什么所有等待都**没有超时**

`wait_till_no_thd()` 与 `wait_till_no_connection()` 都是无条件 `mysql_cond_wait` 循环：

```cpp
void Global_THD_manager::wait_till_no_thd() {
  for (int i = 0; i < NUM_PARTITIONS; i++) {
    MUTEX_LOCK(lock, &LOCK_thd_list[i]);
    while (thd_list[i].size() > 0) {
      LogErr(INFORMATION_LEVEL, ER_WAITING_FOR_NO_THDS, i, thd_list[i].size(), get_thd_count());
      mysql_cond_wait(&COND_thd_list[i], &LOCK_thd_list[i]);
    }
  }
}
```

**设计取舍**：不做超时强制退出，是因为"强行退出"意味着丢失正在执行的事务——对数据库而言，宁可**hang 住让人来查**，也不能静默丢数据。这是"数据安全优先于可服务性"的典型体现。

**代价（这是权衡的另一半）**：关闭 hang 是 MySQL 运维里最常见的问题之一。根因几乎总是某个连接线程卡在一个不可中断的等待上（磁盘满、锁等待、InnoDB 内部 latch）。源码提供了两个日志线索（`ER_WAITING_FOR_NO_THDS` / `ER_WAITING_FOR_NO_CONNECTIONS`），但**没有任何自动解除机制**。

#### 四、为什么 InnoDB 的关闭要拆成 `pre_dd_shutdown` 和 `panic` 两段

这是 8.0 最重要的一处关闭结构变化。

问题的本质：**DD 本身存在 InnoDB 表里**。于是有一个环：

- 要关 DD（`dd::shutdown()`），必须保证**没有任何线程再访问 DD 表**；
- 但 InnoDB 有多个后台线程（dict_stats、fts_optimize、purge、ts_alter_encrypt、master）会访问 DD；
- 而要停这些后台线程，又需要 InnoDB 还在工作状态（它们要用 InnoDB 的锁、事务、latch）。

8.0 的解法是在 `handlerton` 上新增一个 `pre_dd_shutdown` 回调，把关闭序列变成：

```
clean_up()
├─ ha_pre_dd_shutdown()   → innodb_pre_dd_shutdown() → srv_pre_dd_shutdown()
│                            停掉所有"可能访问 DD"的 InnoDB 后台线程
│                            （状态推进到 SRV_SHUTDOWN_DD）
├─ dd::shutdown()         ← 此时才敢关 DD
└─ ...
   └─ plugin_shutdown()  → hton->panic() → innodb_shutdown() → srv_shutdown()
                            这时才真正刷脏、checkpoint、关文件
```

源码注释把契约写得非常清楚：

> Stopping threads that might use system transactions or DD objects. ... If your thread might touch DD objects or use system transactions it must be stopped within `SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS` phase.

**这是给插件作者的一条硬规则**：你的后台线程如果会访问 DD，就必须在 `pre_dd_shutdown` 阶段停掉，否则 DD 关闭后你会访问到已释放的对象。

#### 五、九态状态机：为什么不用"一把锁 + 一个 bool"

```
SRV_SHUTDOWN_NONE
  → SRV_SHUTDOWN_RECOVERY_ROLLBACK
  → SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS
  → SRV_SHUTDOWN_PURGE
  → SRV_SHUTDOWN_DD               ← clean_up() 在这里去关 DD
  → SRV_SHUTDOWN_CLEANUP
  → SRV_SHUTDOWN_MASTER_STOP
  → SRV_SHUTDOWN_FLUSH_PHASE
  → SRV_SHUTDOWN_LAST_PHASE
  → SRV_SHUTDOWN_EXIT_THREADS
```

推进器强制 **单调 +1**：

```cpp
static void srv_shutdown_set_state(srv_shutdown_t new_state) {
  ut_a(static_cast<int>(srv_shutdown_state.load()) + 1 == static_cast<int>(new_state));
  srv_shutdown_state.store(new_state);
}
```

**为什么用状态机而不是 bool**：每个后台线程需要回答一个问题——"我现在还能访问 DD 吗？还能产生新的 undo 吗？还能写 redo 吗？"。布只能回答"是否要退出"，状态机能回答"现在处于哪个阶段，我能做什么"。典型例子：

```cpp
static bool srv_purge_should_exit(ulint n_purged) {
  switch (srv_shutdown_state.load()) {
    case SRV_SHUTDOWN_NONE:
    case SRV_SHUTDOWN_RECOVERY_ROLLBACK:
    case SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS:
      break;                                          // 正常运作
    case SRV_SHUTDOWN_PURGE:
      return (srv_fast_shutdown != 0 || n_purged == 0);  // ★ slow shutdown 要清完
    case SRV_SHUTDOWN_EXIT_THREADS:
      return (true);
    case SRV_SHUTDOWN_LAST_PHASE:
    case SRV_SHUTDOWN_FLUSH_PHASE:
    case SRV_SHUTDOWN_MASTER_STOP:
    case SRV_SHUTDOWN_CLEANUP:
    case SRV_SHUTDOWN_DD:
      ut_error;                                       // 到这个阶段 purge 早该退了
  }
  return (false);
}
```

`ut_error` 那几行是**用断言把顺序契约固定下来**——如果 purge 线程活到了 DD 之后，说明状态机被破坏了，直接崩比静默继续好。

#### 六、`innodb_fast_shutdown` 三档：拿什么换速度

| | `=0`（slow） | `=1`（**默认**） | `=2`（crash-like） |
|---|---|---|---|
| 语义 | 完整收尾：等 recovery 回滚、purge 清空、change buffer 全 merge | 只刷脏 + 干净关 redo | 什么都不做，等价于一次 crash |
| 下次启动 | 直接可用 | 直接可用 | **必须走崩溃恢复** |

**三档的本质区别只有一件事**：`srv_shutdown_log()` 里是否执行 `fil_write_flushed_lsn(lsn)`。

```cpp
  if (srv_fast_shutdown == 2) {
    if (!srv_read_only_mode) {
      ib::info(ER_IB_MSG_1253);
      /* In this fastest shutdown we do not flush the buffer pool:
      it is essentially a 'crash' of the InnoDB server.
      Make sure that the log is all flushed to disk, so
      that we can recover all committed transactions in
      a crash recovery. We must not write the lsn stamps
      to the data files, since at a startup InnoDB deduces
      from the stamps if the previous shutdown was clean. */
      log_stop_background_threads(*log_sys);
    }
    ...
    return (log_get_lsn(*log_sys));
  }
```

即：**`flushed_lsn` 是"上次是否干净关闭"的判据**。写它 = 干净关闭；不写 = 让下次启动以为崩了。而"已提交事务不丢"是由 `log_stop_background_threads()` 保证的（redo 一定落盘），与脏页是否刷盘无关——这是 fast=2 能安全的前提。

**一条隐含的保护规则**：如果 redo logging 被禁用（`ALTER INSTANCE DISABLE INNODB REDO_LOG`）而 `fast_shutdown=2`，会被**强制降级为 1**：

```cpp
  if (mtr_t::s_logging.is_disabled() && srv_fast_shutdown == 2) {
    ib::warn(ER_IB_WRN_FAST_SHUTDOWN_REDO_DISABLED);
    srv_fast_shutdown = 1;
  }
```

因为没有 redo 就没有崩溃恢复能力，一旦不刷脏就真丢数据了。

### 理论溯源

| 思想 / 理论 | 源码落点 |
|---|---|
| **Graceful degradation（优雅降级）** | `close_connections` 的两轮 kill：先请求、后强制 |
| **Phase barrier（阶段屏障）** | `srv_shutdown_t` 九态；`pre_dd_shutdown` 是 DD 关闭前的屏障 |
| **逆序析构（reverse-order destruction）** | `clean_up()` 基本是 `init_server_components()` 的逆序 |
| **幂等关闭** | `set_server_shutting_down()` + `if (!connection_events_loop_aborted())` |
| **Clean shutdown 标志位**（WAL 系统的经典设计：用一个持久化的 lsn 标记区分"干净关闭"与"崩溃"） | `fil_write_flushed_lsn()` / `SysTablespace::read_lsn_and_check_flags()` |
| **POSIX 同步信号处理**（`pthread_sigmask` + 专用线程 `sigwaitinfo`） | `my_init_signals()` + `signal_hand()` |

### 算法与数据结构

| 结构 | 用途 | 特点 |
|---|---|---|
| `Do_THD_Impl` + `Global_THD_manager::do_for_all_thd()` | 遍历所有 THD 执行一个操作 | **访问者模式**；`Set_kill_conn` / `Call_close_conn` 是两个访问者 |
| `Global_THD_manager` 的 `thd_list[NUM_PARTITIONS]` | THD 列表**分区**存储 | 降低锁竞争；代价是 `wait_till_no_thd()` 要逐分区等（关闭路径上无所谓） |
| `Thread_to_stop[]` 表 | 声明式地描述"哪些线程在哪个阶段要停、怎么唤醒它" | 表驱动：`{名字, IB_thread 引用, 唤醒函数, 等待阶段}`；驱动循环统一处理 |
| `srv_shutdown_t` | InnoDB 关闭阶段 | `std::atomic<enum srv_shutdown_t>`，只能 +1 |

### 他库对比与演进动机

| 系统 | 关闭形态 | 与 MySQL 的差异 |
|---|---|---|
| **PostgreSQL** | `smart` / `fast` / `immediate` 三档（`pg_ctl stop -m`）。smart 等所有连接断开 | 概念与 `innodb_fast_shutdown` 三档高度对应，但 PG 的档位作用在**连接层**（是否等客户端），MySQL 的档位作用在**引擎层**（是否刷盘）。MySQL 的连接层没有档位——它总是一段固定的两轮 kill |
| **Oracle** | `SHUTDOWN NORMAL / IMMEDIATE / TRANSACTIONAL / ABORT` | 与 PG 同思路（按"允许多少用户继续"分档）。MySQL 没有 `ABORT` 这一档的 SQL 等价物（`SHUTDOWN` 语句只接受 `SHUTDOWN_DEFAULT`） |
| **Redis** | `SHUTDOWN` 可选 `NOSAVE`；先关监听再持久化 | 单线程模型，无并发线程要停，关闭链极短 |
| **MongoDB** | `db.shutdownServer()`；靠 WiredTiger checkpoint 保证 | 无"停后台线程"的分阶段状态机 |

**演进动机（为什么 8.0 要引入 `pre_dd_shutdown` + 九态机）**：5.7 时代没有事务型 DD，关闭时不需要回答"后台线程还能不能访问 DD"这个问题——顺序 roughly 是"停线程 → 刷盘 → 关文件"一把梭。8.0 把元数据搬进 InnoDB 表之后，"关 DD"成了关闭序列中一个**必须与后台线程停池顺序严格协调的事件**，于是必须把关闭过程显式阶段化。这就是九态状态机与 `pre_dd_shutdown` 回调存在的全部理由。

---

## 核心实现

### 主链路

```
[路径 A] SHUTDOWN SQL
  do_command → dispatch_command → mysql_execute_command
    → Sql_cmd_shutdown::execute()          sql/sql_admin.cc
      → shutdown(thd, SHUTDOWN_DEFAULT)    sql/sql_parse.cc
        → check_global_access(thd, SHUTDOWN_ACL)
        → my_ok(thd)                       ← 先给客户端回 OK
        → kill_mysql()                     sql/mysqld.cc
            → pthread_kill(signal_thread_id.thread, SIGTERM)

[路径 B] kill -TERM / systemd stop
    内核 → 信号线程 signal_hand()

[路径 C] mysqladmin shutdown  ==  路径 A（就是发一条 "shutdown" SQL）

              ▼ 汇聚：信号线程
signal_hand(SIGTERM / SIGQUIT / SIGUSR2)
  → LogErr(ER_SERVER_SHUTDOWN_INFO)
  → query_logger.set_handlers(LOG_FILE)
  → if (!connection_events_loop_aborted())            ← 幂等保护
      set_connection_events_loop_aborted(true)
      pthread_kill(main_thread_id, SIGALRM) × N       ← 打断主线程的 poll()
      等 socket_listener_active == false
      close_connections()                             ← 阶段 1
  → my_thread_exit()

              ▼ 主线程从 connection_event_loop() 返回
mysqld_main
  → server_operational_state = SERVER_SHUTTING_DOWN
  → sysd::notify("STOPPING=1\nSTATUS=Server shutdown in progress\n")
  → mysql_audit_notify(SERVER_SHUTDOWN, REASON_SHUTDOWN, MYSQLD_SUCCESS_EXIT)
  → terminate_compress_gtid_table_thread()            ← GTID 压缩线程
  → gtid_state->save_gtids_of_last_binlog_into_table() ← ★ 必须在 InnoDB 还活着时
  → socket_listener_active = false; broadcast
  → my_thread_join(&signal_thread_id)
  → clean_up(true)                                    ← 阶段 2
      ├─ set_server_shutting_down()                   ← 幂等
      ├─ ha_pre_dd_shutdown()  → srv_pre_dd_shutdown()  （InnoDB 停访问 DD 的线程）
      ├─ dd::shutdown()
      ├─ ha_binlog_end() → mysql_bin_log.cleanup()
      ├─ acl_free / grant_free / my_tz_free / udf_unload_udfs
      ├─ plugin_shutdown() → hton->panic() → innodb_shutdown() → srv_shutdown()
      │                                       （★ 刷脏 + checkpoint 在此时）
      ├─ gtid_server_cleanup()                ← 注释：after plugin_shutdown
      ├─ delete_pid_file() + LogErr(ER_SERVER_SHUTDOWN_COMPLETE)
      └─ component_infrastructure_deinit() → sys_var_end()
  → mysqld_exit(signal_hand_thr_exit_code)            ← 阶段 3
      → mysql_audit_finalize / Srv_session::module_deinit
      → clean_up_mutexes → my_end → destroy_error_log
      → shutdown_performance_schema
      → exit(exit_code)
```

### 一、三条入口

#### 1.1 `SHUTDOWN` 语句

语法层（`sql_yacc.yy`）只有一个终结符——**8.0 的 `SHUTDOWN` 不接受任何参数**：

```
shutdown_stmt:
          SHUTDOWN
          {
            Lex->sql_command= SQLCOM_SHUTDOWN;
            $$= NEW_PTN PT_shutdown();
          }
        ;
```

```cpp
bool shutdown(THD *thd, enum mysql_enum_shutdown_level level) {
  thd->lex->no_write_to_binlog = true;

  if (check_global_access(thd, SHUTDOWN_ACL)) goto error;

  if (level == SHUTDOWN_DEFAULT)
    level = SHUTDOWN_WAIT_ALL_BUFFERS;  // soon default will be configurable
  else if (level != SHUTDOWN_WAIT_ALL_BUFFERS) {
    my_error(ER_NOT_SUPPORTED_YET, MYF(0), "this shutdown level");
    goto error;
  }

  my_ok(thd);
  LogErr(SYSTEM_LEVEL, ER_SERVER_SHUTDOWN_INFO, thd->security_context()->user().str, ...);
  query_logger.general_log_print(thd, COM_QUERY, NullS);
  kill_mysql();
  res = true;
error:
  return res;
}
```

两点值得注意：

1. **`my_ok()` 先回 OK，再 `res = true`**。`dispatch_command()` 里有一句配套：

```cpp
      /* Need to set error to true for graceful shutdown */
      if ((thd->lex->sql_command == SQLCOM_SHUTDOWN) &&
          (thd->get_stmt_da()->is_ok()))
        error = true;
```

即：客户端收到干净的 OK，而 server 侧认为"这条连接结束了"从而关闭它。这样 `mysqladmin shutdown` 不会报奇怪的错误。

2. **`enum mysql_enum_shutdown_level` 有 6 个值，但服务端只接受一个**。`SHUTDOWN_WAIT_CONNECTIONS` / `WAIT_TRANSACTIONS` / `WAIT_UPDATES` / `WAIT_CRITICAL_BUFFERS` 全部会撞 `ER_NOT_SUPPORTED_YET`。这些枚举值是协议遗留（`mysql_com.h` 里还留着），**不是可用功能**。

权限：`SHUTDOWN_ACL = (Access_bitmask)1 << 7`（`RELOAD_ACL` 是 `1<<6`），属 `GLOBAL_ACLS`。

#### 1.2 信号

见 [`02_startup.md`](02_startup.md) 的信号章节。关闭相关的是 SIGTERM / SIGQUIT / SIGUSR2 分支：

```cpp
      case SIGUSR2:
        signal_hand_thr_exit_code = MYSQLD_RESTART_EXIT;
        [[fallthrough]];
      case SIGTERM:
      case SIGQUIT:
        if (sig_info.si_pid != getpid())
          LogErr(SYSTEM_LEVEL, ER_SERVER_SHUTDOWN_INFO, "<via user signal>", ...);
        query_logger.set_handlers((log_output_options != LOG_NONE) ? LOG_FILE : LOG_NONE);
        if (!connection_events_loop_aborted()) {
          set_connection_events_loop_aborted(true);
#ifdef HAVE_PSI_THREAD_INTERFACE
          PSI_THREAD_CALL(delete_current_thread)();
#endif
          /*
            Kill the socket listener.
            The main thread will then set socket_listener_active= false,
            and wait for us to finish all the cleanup below.
          */
          mysql_mutex_lock(&LOCK_socket_listener_active);
          while (socket_listener_active) {
            if (pthread_kill(main_thread_id, SIGALRM)) { assert(false); break; }
            mysql_cond_wait(&COND_socket_listener_active, &LOCK_socket_listener_active);
          }
          mysql_mutex_unlock(&LOCK_socket_listener_active);

          close_connections();
        }
        my_thread_end();
        my_thread_exit(nullptr);
        return nullptr;
```

**"杀 listener" 的机制**是这里最巧的一处：主线程阻塞在 `poll()` 上等连接，根本不会去检查任何标志。信号线程用 `pthread_kill(main_thread_id, SIGALRM)` 打过去，`poll()` 因 `EINTR` 返回（`empty_signal_handler` 什么都不做），主线程这才走到循环条件看到 `connection_events_loop_aborted()`，退出循环并把 `socket_listener_active` 置 false + broadcast。信号线程在 cond 上等这个信号。

```cpp
          while (socket_listener_active) {
            if (pthread_kill(main_thread_id, SIGALRM)) { assert(false); break; }
            mysql_cond_wait(&COND_socket_listener_active, &LOCK_socket_listener_active);
          }
```

注意这个 `while` 循环：如果主线程被 SIGALRM 打断后又回到 `poll()`（比如刚好有个连接到来），信号线程会再打一次。**用重试循环对抗"信号丢失"**，而不是依赖单次投递。

**SIGHUP / SIGUSR1 不关闭服务器**：

| 信号 | 行为 |
|---|---|
| SIGHUP | `handle_reload_request(REFRESH_LOG \| REFRESH_TABLES \| REFRESH_FAST \| REFRESH_GRANT \| REFRESH_THREADS \| REFRESH_HOSTS)` —— 重载权限/hosts + flush |
| SIGUSR1 | `handle_reload_request(REFRESH_ERROR_LOG \| REFRESH_GENERAL_LOG \| REFRESH_SLOW_LOG)` —— 轮转 error log + flush 日志 |

**Windows**：没有信号，走 `handle_shutdown_and_restart()`（`WaitForMultipleObjects(hEventShutdown, hEventRestart)`），命中后 `set_connection_events_loop_aborted(true); close_connections();`。

#### 1.3 `mysqladmin shutdown`

```cpp
      case ADMIN_SHUTDOWN: {
        if (mysql_query(mysql, "shutdown")) { ... return -1; }
        argc = 1; /* force SHUTDOWN to be the last command */
        if (got_pidfile) {
          /* Wait until pid file is gone */
          if (wait_pidfile(pidfile, last_modified, &pidfile_status)) return -1;
        }
```

即：发一条 `shutdown` SQL + **等 pid 文件消失**（默认超时 `SHUTDOWN_DEF_TIMEOUT = 3600`）。

已废弃的 `COM_SHUTDOWN` 协议命令在 8.0 只剩一个占位：`COM_DEPRECATED_1, /**< Deprecated, used to be COM_SHUTDOWN */`。老的 C API `mysql_shutdown()` 也改为发 SQL。

### 二、阶段 1：`close_connections()`

完整原文（注释保留，它们是设计说明的一部分）：

```cpp
static void close_connections(void) {
  (void)RUN_HOOK(server_state, before_server_shutdown, (nullptr));

  Per_thread_connection_handler::kill_blocked_pthreads();

  uint dump_thread_count = 0;
  uint dump_thread_kill_retries = 8;

  // Close listeners.
  if (mysqld_socket_acceptor != nullptr)
    mysqld_socket_acceptor->close_listener();
#ifdef _WIN32
  if (named_pipe_acceptor != NULL) named_pipe_acceptor->close_listener();
  if (shared_mem_acceptor != NULL) shared_mem_acceptor->close_listener();
#endif

  /*
    First signal all threads that it's time to die
    This will give the threads some time to gracefully abort their
    statements and inform their clients that the server is about to die.
  */
  Global_THD_manager *thd_manager = Global_THD_manager::get_instance();
  LogErr(INFORMATION_LEVEL, ER_DEPART_WITH_GRACE,
         static_cast<int>(thd_manager->get_thd_count()));

  Set_kill_conn set_kill_conn;
  thd_manager->do_for_all_thd(&set_kill_conn);
  LogErr(INFORMATION_LEVEL, ER_SHUTTING_DOWN_REPLICA_THREADS);
  end_slave();

  if (set_kill_conn.get_dump_thread_count()) {
    while (thd_manager->get_thd_count() > dump_thread_count &&
           dump_thread_kill_retries) {
      sleep(1);
      dump_thread_kill_retries--;
    }
    set_kill_conn.set_dump_thread_flag();
    thd_manager->do_for_all_thd(&set_kill_conn);
  }

  // Disable the event scheduler
  Events::stop();

  if (thd_manager->get_thd_count() > 0) sleep(2);  // Give threads time to die

  /*
    Force remaining threads to die by closing the connection to the client
    This will ensure that threads that are waiting for a command from the
    client on a blocking read call are aborted.
  */
  LogErr(INFORMATION_LEVEL, ER_DISCONNECTING_REMAINING_CLIENTS,
         static_cast<int>(thd_manager->get_thd_count()));

  Call_close_conn call_close_conn(true);
  thd_manager->do_for_all_thd(&call_close_conn);

  (void)RUN_HOOK(server_state, after_server_shutdown, (nullptr));

  /*
    All threads have now been aborted. Stop event scheduler thread
    after aborting all client connections, otherwise user may
    start/stop event scheduler after Events::deinit() deallocates
    scheduler object(static member in Events class)
  */
  Events::deinit();
  thd_manager->wait_till_no_thd();
  Connection_handler_manager::wait_till_no_connection();

  delete_slave_info_objects();
}
```

#### 2.1 两个访问者（`Do_THD_Impl`）

**没有 `kill_all_server_threads` 这个函数**（全仓库 0 命中）。真实实现是两个 functor + `do_for_all_thd()`。

`Set_kill_conn`（温和）：

```cpp
  void operator()(THD *killing_thd) override {
    if (!m_kill_dump_threads_flag) {
      // We skip slave threads & scheduler on this first loop through.
      if (killing_thd->slave_thread) return;

      if (killing_thd->get_command() == COM_BINLOG_DUMP ||
          killing_thd->get_command() == COM_BINLOG_DUMP_GTID) {
        ++m_dump_thread_count;
        return;
      }
    }
    mysql_mutex_lock(&killing_thd->LOCK_thd_data);

    if (killing_thd->kill_immunizer) {
      /*
        If killing_thd is in kill immune mode (i.e. operation on new DD tables
        is in progress) then just save state_to_set with THD::kill_immunizer
        object.
      */
      killing_thd->kill_immunizer->save_killed_state(THD::KILL_CONNECTION);
    } else {
      killing_thd->killed = THD::KILL_CONNECTION;
      MYSQL_CALLBACK(Connection_handler_manager::event_functions,
                     post_kill_notification, (killing_thd));
    }

    if (killing_thd->is_killable && killing_thd->kill_immunizer == nullptr) {
      mysql_mutex_lock(&killing_thd->LOCK_current_cond);
      if (killing_thd->current_cond.load()) {
        mysql_mutex_lock(killing_thd->current_mutex);
        mysql_cond_broadcast(killing_thd->current_cond);
        mysql_mutex_unlock(killing_thd->current_mutex);
      }
      mysql_mutex_unlock(&killing_thd->LOCK_current_cond);
    }
    mysql_mutex_unlock(&killing_thd->LOCK_thd_data);
  }
```

两条精细设计：

- **`kill_immunizer`**：处于"正在操作新 DD 表"这种不可中断临界区的线程，不会被立即 kill，而是**把 kill 状态存起来**，等退出临界区后再生效。这是对 8.0 引入 DD 之后新增的不可中断区段的直接应对。
- **broadcast 它正在等的 cond**：线程可能睡在 `current_cond` 上（比如等锁、等磁盘空间），只置 `killed` 不够，必须把它唤醒。

`Call_close_conn`（强制）：

```cpp
  void operator()(THD *closing_thd) override {
    if (closing_thd->get_protocol()->connection_alive()) {
      LEX_CSTRING main_sctx_user = closing_thd->m_main_security_ctx.user();
      LogErr(WARNING_LEVEL, ER_FORCE_CLOSE_THREAD, my_progname,
             (long)closing_thd->thread_id(),
             (main_sctx_user.length ? main_sctx_user.str : ""));
      /*
        Do not generate MYSQL_AUDIT_CONNECTION_DISCONNECT event, when closing
        thread close sessions. Each session will generate DISCONNECT event by
        itself.
      */
      close_connection(closing_thd, 0, is_server_shutdown, false);
    }
  }
```

日志：`ER_FORCE_CLOSE_THREAD` = "`%s: Forcing close of thread %ld  user: '%-.48s'.`"

#### 2.2 复制线程的收尾

```
close_connections()
  └ end_slave()
      └ terminate_slave_threads(mi, SLAVE_FORCE_ALL, rpl_stop_replica_timeout)
          ├ mi->rli->abort_slave = true
          ├ mi->rli->flush_info(Relay_log_info::RLI_FLUSH_IGNORE_SYNC_OPT)  ← 强制刷 relay-log.info
          ├ Source_IO_monitor::terminate_monitoring_process()
          └ mi->abort_slave = true; terminate_slave_thread(IO, force_io_stop?)
  └ delete_slave_info_objects()   ← 放在 wait_till_no_thd() 之后
```

`terminate_slave_threads()` 的两个细节：

- 超时 `rpl_stop_replica_timeout` 在 **SQL 与 IO 两线程之间共享**（`total_stop_wait_timeout`），源码注释明确说明这是"两个线程总共"的超时；但 `SLAVE_FORCE_ALL` 时超时不再返回错误——**关闭时不等超时也继续**。
- 磁盘满的特殊路径：IO 线程若阻塞在"等磁盘空间"，普通信号叫不醒，要走 `force_io_stop` 用 `KILL_CONNECTION` 打断 `my_write`。

`delete_slave_info_objects()` 的位置由注释固定：

> Free all resources used by slave threads at time of executing shutdown. The routine must be called after all possible users of channel_map have left.

#### 2.3 等待机制（两套计数器，都无超时）

```cpp
void Global_THD_manager::wait_till_no_thd() {
  for (int i = 0; i < NUM_PARTITIONS; i++) {
    MUTEX_LOCK(lock, &LOCK_thd_list[i]);
    while (thd_list[i].size() > 0) {
      LogErr(INFORMATION_LEVEL, ER_WAITING_FOR_NO_THDS, i, thd_list[i].size(), get_thd_count());
      mysql_cond_wait(&COND_thd_list[i], &LOCK_thd_list[i]);
    }
  }
}
```

```cpp
void Connection_handler_manager::wait_till_no_connection() {
  mysql_mutex_lock(&LOCK_connection_count);
  while (connection_count > 0) {
    LogErr(INFORMATION_LEVEL, ER_WAITING_FOR_NO_CONNECTIONS, connection_count);
    mysql_cond_wait(&COND_connection_count, &LOCK_connection_count);
  }
  mysql_mutex_unlock(&LOCK_connection_count);
}
```

为什么要两套：THD 已从全局列表摘除后，连接线程还要走一小段清理（release_resources / remove_thd / 归还线程缓存），`connection_count` 才归零。注释原文：`Connection threads might take a little while to go down after removing from global thread list.`

> **澄清**：`COND_thread_count` 与 `Global_THD_manager::inc_thread_used` 在 8.0.39 **都不存在**（5.7 遗留名，全仓库 0 命中）。真实的是上面两套。

### 三、阶段 2：`clean_up()`

#### 3.1 幂等保护

```cpp
static bool set_server_shutting_down() {
  /* Server shutting down already set. */
  if (server_shutting_down) return true;
  mysql_rwlock_wrlock(&LOCK_server_shutting_down);
  server_shutting_down = true;
  mysql_rwlock_unlock(&LOCK_server_shutting_down);
  return false;
}
```

`clean_up()` 可能被 `unireg_abort()` 与正常路径各调一次。

#### 3.2 清理顺序与依赖

| # | 调用 | 为什么放这里 |
|---|---|---|
| 1 | `ha_pre_dd_shutdown()` | **最先**。让 SE 停掉所有会访问 DD 的后台线程 |
| 2 | `dd::shutdown()` | 必须在 #1 之后、插件卸载之前 |
| 3 | `Events::deinit()` / `stop_handle_manager()` | 后台线程，依赖 THD/plugin |
| 4 | `memcached_shutdown()` | `daemon_memcached` 插件的特殊提前 deinit（它持有 plugin 引用） |
| 5 | `release_keyring_handles()` / `keyring_lockable_deinit()` | 插件 deinit 前释放引用 |
| 6 | `LogErr(ER_BINLOG_END)` + `ha_binlog_end(current_thd)` | 注释：`make sure that handlers finish up what they have that is dependent on the binlog` —— 必须在 binlog 关闭之前 |
| 7 | `injector::free_instance()` | 复制 injector，依赖 binlog 对象 |
| 8 | `mysql_bin_log.cleanup()` | 此时已无 SE 依赖 |
| 9 | `my_tz_free` / `servers_free` / `acl_free` / `grant_free` / `hostname_cache_free` | 依赖 plugin/UDF/表 |
| 10 | `udf_unload_udfs()` / `table_def_start_shutdown()` / `delegates_shutdown()` | UDF 与 observer 要在插件卸载前 |
| 11 | **`plugin_shutdown()`** | **InnoDB 真正刷盘/checkpoint 发生在这一步**（`hton->panic()`） |
| 12 | `gtid_server_cleanup()` | 注释 `// after plugin_shutdown`——GTID 的**持久化**依赖 SE（写 `mysql.gtid_executed`），所以真正的保存已在 `mysqld_main` 里做完，这里只 delete 对象 |
| 13 | `ha_end()` | 只做 `my_error_unregister` + `my_free(handler_errmsgs)` |
| 14 | `tc_log->close()` | binlog TC |
| 15 | `Recovered_xa_transactions::destroy()` / `xa::Transaction_cache::dispose()` / `table_def_free()` / `mdl_destroy()` | |
| 16 | `query_logger.cleanup()` / `end_ssl()` / `vio_end()` / `u_cleanup()` | 全局子系统 |
| 17 | **`delete_pid_file()`** + `ER_SERVER_SHUTDOWN_COMPLETE` + `sysd::notify("STATUS=Server shutdown complete")` | 对外宣告；`mysqladmin shutdown` 就是等 pid 文件消失 |
| 18 | `free_connection_acceptors()` / `Connection_handler_manager::destroy_instance()` / `Global_THD_manager::destroy_instance()` | 连接层基础设施，必须在所有 THD 销毁之后 |
| 19 | `log_error_stage_set(LOG_ERROR_STAGE_SHUTTING_DOWN)` | 源码注释：`Is this the best place for components deinit? It may be changed when new dependencies are discovered...` |
| 20 | `deinitialize_manifest_file_components()` → `component_infrastructure_deinit()` | 组件基础设施 |
| 21 | **`sys_var_end()`** | 注释：`component unregister_variable() api depends on system_variable_hash` ⇒ **必须在 #20 之后** |
| 22 | `free_status_vars()` / `finish_client_errs()` / `deinit_errmessage()` | 错误消息与状态变量最后释放 |

**几个容易忽略的顺序约束**：

- `gtid_server_cleanup()` 在 `plugin_shutdown()` **之后**：因为 GTID 对象里可能还有对 SE 的引用；而 GTID 的**落盘**必须反过来在 `plugin_shutdown()` **之前**（见 §4）。
- `sys_var_end()` 在 `component_infrastructure_deinit()` **之后**：组件的 deinit 会调 `unregister_variable()`，需要 `system_variable_hash` 还在。
- error log 是**最后**关的（在 `my_end()` 之后，由 `destroy_error_log()` 做）——因为整个关闭过程还要写日志。

#### 3.3 GTID 的落盘必须在 InnoDB 还活着时

`mysqld_main` 里，`clean_up()` **之前**：

```cpp
  terminate_compress_gtid_table_thread();
  /*
    Save set of GTIDs of the last binlog into gtid_executed table
    on server shutdown.
  */
  if (opt_bin_log)
    if (gtid_state->save_gtids_of_last_binlog_into_table())
      LogErr(WARNING_LEVEL, ER_CANT_SAVE_GTIDS);
```

顺序：`terminate_compress_gtid_table_thread()`（停压缩线程）→ `save_gtids_of_last_binlog_into_table()`（写 `mysql.gtid_executed`）→ `clean_up()`（`plugin_shutdown()` 会让 InnoDB 停下）。

`terminate_compress_gtid_table_thread()`：

```cpp
  mysql_mutex_lock(&LOCK_compress_gtid_table);
  terminate_compress_thread = true;
  mysql_cond_signal(&COND_compress_gtid_table);
  mysql_mutex_unlock(&LOCK_compress_gtid_table);

  if (compress_thread_id.thread != 0) {
    error = my_thread_join(&compress_thread_id, nullptr);
    compress_thread_id.thread = 0;
```

`gtid_server_cleanup()` 只是 delete 四个全局对象（`gtid_state` / `global_sid_map` / `global_sid_lock` / `gtid_table_persistor`）。

#### 3.4 binlog 收尾：写 Stop event，不是 rotate

```cpp
void MYSQL_BIN_LOG::cleanup() {
  if (inited) {
    inited = false;
    close(LOG_CLOSE_INDEX | LOG_CLOSE_STOP_EVENT, true, true);
    mysql_mutex_destroy(&LOCK_log);
    mysql_mutex_destroy(&LOCK_index);
    mysql_mutex_destroy(&LOCK_commit);
    ... /* 一堆 mutex/cond 销毁 */
    if (!is_relay_log) Commit_stage_manager::get_instance().deinit();
  }
  delete m_binlog_file;
  m_binlog_file = nullptr;
}
```

关闭时写的是一个 **Stop event**（不是 rotate）：

```cpp
    if ((exiting & LOG_CLOSE_STOP_EVENT) != 0) {
      /**
        TODO(WL#7546): Change the implementation to Stop_event after write() is
        moved into libbinlogevents
      */
      Stop_log_event s;
      s.common_footer->checksum_alg =
          is_relay_log ? relay_log_checksum_alg
                       : static_cast<enum_binlog_checksum_alg>(binlog_checksum_options);
```

`close()` 的 `exiting` 位掩码定义：`LOG_CLOSE_INDEX`（关 index 文件）/ `LOG_CLOSE_TO_BE_OPENED`（关完马上要开）/ `LOG_CLOSE_STOP_EVENT`（写 stop 事件）。

### 四、InnoDB 关闭（本篇最核心）

#### 4.1 函数名核实（这批名字错得最多）

| 常见误记 | 8.0.39 真实 |
|---|---|
| `innobase_shutdown` / `innobase_end` / `innobase_shutdown_for_mysql` | **都不存在**。真实是 `static int innodb_shutdown(handlerton *, ha_panic_function)`，挂在 `innobase_hton->panic` |
| `log_shutdown` | 不存在。真实是 `log_sys_close()`（= `log_sys_free()`）与 `log_make_empty_and_stop_background_threads()` / `log_stop_background_threads()` |
| `logs_empty_and_mark_files_at_shutdown` | 不存在（8.0 redo 改版后删除）。现在是 `srv_shutdown_log()` |
| `log_make_checkpoint_at` | 不存在。现在是 `log_make_latest_checkpoint(log)` |
| `buf_pool_free` | 存在但是 **static**。关闭入口是 `buf_pool_free_all()` |

注册：

```cpp
  innobase_hton->pre_dd_shutdown = innodb_pre_dd_shutdown;
  innobase_hton->panic = innodb_shutdown;
```

调用链：`plugin_shutdown()` → `plugin_deinitialize()` → `ha_finalize_handlerton()` → `hton->panic(hton, HA_PANIC_CLOSE)`。

#### 4.2 `innodb_shutdown()`

```cpp
static int innodb_shutdown(handlerton *, ha_panic_function) {
  if (innodb_inited) {
    log_pfs_delete_tables();
    innodb_inited = false;
    ut::delete_(innobase_open_tables);
    innobase_open_tables = nullptr;
    for (auto file : innobase_sys_files) ut::delete_(file);
    innobase_sys_files.clear();
    innobase_sys_files.shrink_to_fit();

    mutex_free(&master_key_id_mutex);
    srv_shutdown();
    innodb_space_shutdown();

    mysql_mutex_destroy(&innobase_share_mutex);
    ... /* 一堆 mutex/cond/event 销毁 */
    os_event_global_destroy();
  }
  innobase::component_services::deinitialize_service_handles();
  return 0;
}

static void innodb_pre_dd_shutdown(handlerton *) {
  if (innodb_inited) srv_pre_dd_shutdown();
}
```

#### 4.3 九态状态机全谱

```cpp
enum srv_shutdown_t {
  SRV_SHUTDOWN_NONE = 0,
  /** Shutdown has started. Stopping the thread responsible for rollback of
  recovered transactions. ... @remarks Note that user transactions are stopped
  earlier, when the shutdown state is still equal to SRV_SHUTDOWN_NONE (user
  transactions are closed when related connections are closed in
  close_connections()). */
  SRV_SHUTDOWN_RECOVERY_ROLLBACK,
  /** Stopping threads that might use system transactions or DD objects.
  ... List of threads being stopped within this phase:
    - dict_stats thread, - fts_optimize thread, - ts_alter_encrypt thread.
  The master thread exits its main loop and finishes its first phase
  of shutdown (in which it was allowed to touch DD objects). */
  SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS,
  /** Stopping the purge threads. Before we enter this phase, we have
  the guarantee that no new undo records could be produced. */
  SRV_SHUTDOWN_PURGE,
  /** Shutting down the DD. */
  SRV_SHUTDOWN_DD,
  /** Stopping remaining InnoDB background threads except: master, redo log
  threads, page cleaner threads, archiver threads. List: lock_wait_timeout,
  error_monitor, monitor, buf_dump, buf_resize.
  @remarks If your thread might touch DD objects or use system transactions
  it must be stopped within SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS phase. */
  SRV_SHUTDOWN_CLEANUP,
  /** Stopping the master thread. */
  SRV_SHUTDOWN_MASTER_STOP,
  /** ... the page cleaners can clean up the buffer pool and exit. The redo log
  threads write and flush the log buffer and exit after the page cleaners. */
  SRV_SHUTDOWN_FLUSH_PHASE,
  /** Last phase after ensuring that all data have been flushed to disk and
  the flushed_lsn has been updated in the header of system tablespace. ... */
  SRV_SHUTDOWN_LAST_PHASE,
  /** Exit all threads and free resources. We might reach this phase in one
  of two different ways: - after visiting all previous states (usual shutdown),
  - or during startup when we failed and we abort the startup. */
  SRV_SHUTDOWN_EXIT_THREADS
};
```

注意第一条注释里的关键信息：**用户事务在 `SRV_SHUTDOWN_NONE` 阶段就已经停了**（`close_connections()` 杀连接时顺带结束），InnoDB 的状态机从"处理 recovery 回滚"开始。

#### 4.4 `srv_pre_dd_shutdown()`（clean_up 第 1 步到达）

```cpp
void srv_pre_dd_shutdown() {
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_NONE);

  /* Warn and wait if there are still some query threads alive. ... */
  for (size_t count = 0; count < 10; ++count) {
    const auto threads_count = srv_conc_get_active_threads();
    if (threads_count == 0) break;
    ib::warn(ER_IB_MSG_1154, threads_count);
    std::this_thread::sleep_for(std::chrono::seconds(1));
  }
  /* Crash if some query threads are still alive. */
  ut_a(srv_conc_get_active_threads() == 0);

  ut_a(!srv_thread_is_active(srv_threads.m_recv_writer));

  /* Avoid fast shutdown, if redo logging is disabled. Otherwise, we won't be
  able to recover. */
  if (mtr_t::s_logging.is_disabled() && srv_fast_shutdown == 2) {
    ib::warn(ER_IB_WRN_FAST_SHUTDOWN_REDO_DISABLED);
    srv_fast_shutdown = 1;
  }

  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  gtid_persistor.stop();

  if (srv_read_only_mode) {
    /* 直接跳到 SRV_SHUTDOWN_DD */
    srv_shutdown_set_state(SRV_SHUTDOWN_RECOVERY_ROLLBACK);
    srv_shutdown_set_state(SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS);
    srv_shutdown_set_state(SRV_SHUTDOWN_PURGE);
    srv_shutdown_set_state(SRV_SHUTDOWN_DD);
    return;
  }

  srv_shutdown_set_state(SRV_SHUTDOWN_RECOVERY_ROLLBACK);
  if (srv_shutdown_waits_for_rollback_of_recovered_transactions()) {
    for (uint32_t count = 0;; ++count) {
      if (!srv_thread_is_active(srv_threads.m_trx_recovery_rollback)) break;
      const auto total_trx = trx_sys_recovered_active_trxs_count();
      if (total_trx == 0) break;
      if (count >= SHUTDOWN_SLEEP_ROUNDS) {
        ib::info(ER_IB_MSG_1249, total_trx);
        count = 0;
      }
      std::this_thread::sleep_for(std::chrono::microseconds(SHUTDOWN_SLEEP_TIME_US));
    }
  }
  ...
  srv_shutdown_set_state(SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS);
  fts_optimize_shutdown();
  dict_stats_shutdown();
  dict_stats_thread_deinit();
  ... 等 ts_alter_encrypt 退出 ...
  /* Wait until the master thread exits its main loop and notices that:
    - it should do shutdown-cleanup, - and still is allowed to access DD objects. */
  if (srv_thread_is_active(srv_threads.m_master)) {
    srv_wake_master_thread();
    os_event_wait(srv_threads.m_master_ready_for_dd_shutdown);
  }
  /* Since this point we do not expect accesses to DD coming from InnoDB. */
  ut_d(trx_sys_before_pre_dd_shutdown_validate());

  srv_shutdown_set_state(SRV_SHUTDOWN_PURGE);
  for (uint32_t count = 1; srv_purge_threads_active(); ++count) {
    srv_purge_wakeup();
    if (count % SHUTDOWN_SLEEP_ROUNDS == 0) ib::info(ER_IB_MSG_1152);
    std::this_thread::sleep_for(std::chrono::microseconds(SHUTDOWN_SLEEP_TIME_US));
  }
  ...
  srv_shutdown_set_state(SRV_SHUTDOWN_DD);
}
```

**两个硬约束**：

1. **活动线程数必须归零，否则直接 crash**：

```cpp
  for (size_t count = 0; count < 10; ++count) { ... sleep(1s) ... }
  /* Crash if some query threads are still alive. */
  ut_a(srv_conc_get_active_threads() == 0);
```

给了 10 秒宽限，之后 `ut_a` 断言失败（debug 下是断言，release 下 `ut_a` 同样 abort）。**这是关闭期间真实存在的 crash 路径**——如果某个查询线程卡死在 InnoDB 内部，10 秒后服务器会自己崩掉。

2. **等待常量**：`SHUTDOWN_SLEEP_TIME_US = 100`，`SHUTDOWN_SLEEP_ROUNDS = 60 * 1000 * 1000 / 100 = 600000` ⇒ **每 60 秒**打印一次等待消息。

#### 4.5 `srv_shutdown()`（plugin_shutdown 到达）

```cpp
void srv_shutdown() {
  /* Ensure threads below have been stopped. */
  const auto threads_stopped_before_shutdown = {
      std::cref(srv_threads.m_purge_coordinator), std::cref(srv_threads.m_ts_alter_encrypt),
      std::cref(srv_threads.m_fts_optimize), std::cref(srv_threads.m_recv_writer),
      std::cref(srv_threads.m_dict_stats)};
  for (const auto &thread : threads_stopped_before_shutdown) {
    ut_a(!srv_thread_is_active(thread));
  }
  ...
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_DD);

  /* Write dynamic metadata to DD buffer table. */
  dict_persist_to_dd_table_buffer();

  /* 0. Stop remaining background threads except page-cleaners, redo-log-threads,
        archiver threads. After this call the state is SRV_SHUTDOWN_MASTER_STOP. */
  srv_shutdown_cleanup_and_master_stop();

  dict_persist_to_dd_table_buffer();

  /* The steps 1-4 is the real InnoDB shutdown.
  All before was to stop activity which could produce new changes.
  All after is just cleaning up (freeing memory). */

  /* 1. Flush the buffer pool to disk. */
  srv_shutdown_page_cleaners();
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_FLUSH_PHASE);

  /* 2. Write the current lsn to the tablespace header(s). */
  const lsn_t shutdown_lsn = srv_shutdown_log();
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_LAST_PHASE);

  /* 3. Close all opened files. */
  ibt::close_files();
  fil_close_all_files();
  if (srv_monitor_file) fclose(srv_monitor_file);
  if (srv_misc_tmpfile) fclose(srv_misc_tmpfile);

  /* 4. Copy all log data to archive and stop archiver threads. */
  srv_shutdown_arch();

  srv_shutdown_exit_threads();
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_EXIT_THREADS);

  /* 5. Free all the resources acquired by InnoDB (mutexes, events, memory). */
  ...
  /* This must be disabled before closing the buffer pool
  and closing the data dictionary.  */
  btr_search_disable();
  ibuf_close();
  ddl_log_close();
  log_sys_close();
  recv_sys_close();
  trx_sys_close();
  lock_sys_close();
  trx_pool_close();
  dict_close();
  dict_persist_close();
  btr_search_sys_free();
  undo_spaces_deinit();
  os_aio_free();
  que_close();
  row_mysql_close();
  srv_free();
  fil_close();
  pars_close();
  pars_lexer_close();
  buf_pool_free_all();
  /* 6. Free the thread management resources. */
  clone_free();
  arch_free();
  dblwr::close();
  os_thread_close();
  sync_check_close();

  ib::info(ER_IB_MSG_1155, ulonglong{shutdown_lsn});   // "Shutdown completed; log sequence number %llu"
}
```

注释把这段分成了清晰的语义段：**"之前都是停止产生新变更的活动，1-4 是真正的关闭，之后只是清理内存"**。

#### 4.6 后台线程怎么被停：表驱动 + 反复 `os_event_set`

```cpp
struct Thread_to_stop {
  const char *m_name;              // 打印用
  const IB_thread &m_thread;
  std::function<void()> m_notify;  // 唤醒函数，可重复调用
  srv_shutdown_t m_wait_on_state;  // 在哪个阶段等它
};

static const Thread_to_stop threads_to_stop[]{
    {"lock_wait_timeout", srv_threads.m_lock_wait_timeout, lock_set_timeout_event, SRV_SHUTDOWN_CLEANUP},
    {"error_monitor", srv_threads.m_error_monitor, []() { os_event_set(srv_error_event); }, SRV_SHUTDOWN_CLEANUP},
    {"monitor", srv_threads.m_monitor, []() { os_event_set(srv_monitor_event); }, SRV_SHUTDOWN_CLEANUP},
    {"buf_dump", srv_threads.m_buf_dump, []() { os_event_set(srv_buf_dump_event); }, SRV_SHUTDOWN_CLEANUP},
    {"buf_resize", srv_threads.m_buf_resize, []() { os_event_set(srv_buf_resize_event); }, SRV_SHUTDOWN_CLEANUP},
    {"master", srv_threads.m_master, srv_wake_master_thread, SRV_SHUTDOWN_MASTER_STOP}};
```

驱动循环：

```cpp
  for (;;) {
    bool print;
    if (count >= SHUTDOWN_SLEEP_ROUNDS) { print = true; count = 0; } else { print = false; }

    size_t active_found = 0;
    for (const auto &thread_info : threads_to_stop) {
      ut_a(thread_info.m_wait_on_state <= max_wait_on_state);
      if (thread_info.m_wait_on_state == srv_shutdown_state.load() &&
          srv_thread_is_active(thread_info.m_thread)) {
        ++active_found;
        if (print) ib::info(ER_IB_MSG_1248, thread_info.m_name);   // "Waiting for %s to exit."
        thread_info.m_notify();
      }
    }

    if (active_found == 0) {
      if (srv_shutdown_state.load() == max_wait_on_state) break;
      srv_shutdown_set_state(static_cast<srv_shutdown_t>(
          static_cast<int>(srv_shutdown_state.load()) + 1));
    }
    std::this_thread::sleep_for(std::chrono::microseconds(SHUTDOWN_SLEEP_TIME_US));
    ++count;
  }
```

**模式**：轮询（100μs）+ **反复**调 `m_notify()` + 每 60s 打日志。为什么不用条件变量：每个线程的"退出条件"各不相同（有的等 event、有的检查状态、有的等任务队列空），用统一的表驱动 + 反复通知最省事，代价是 100μs 粒度的忙等（对关闭路径无所谓）。

**隐含契约**：各后台线程必须在自己的循环里检查 `srv_shutdown_state.load()`，否则 `m_notify()` 唤醒它之后它又会睡回去——这正是"反复通知"存在的原因。

#### 4.7 master 线程的三个循环

```cpp
void srv_master_thread() {
  srv_master_main_loop(slot);                                  // ① 主循环
  srv_master_pre_dd_shutdown_loop();                           // ② PRE_DD 阶段
  os_event_set(srv_threads.m_master_ready_for_dd_shutdown);    // ← 通知 srv_pre_dd_shutdown

  while (srv_shutdown_state.load() < SRV_SHUTDOWN_MASTER_STOP) {
    srv_master_wait(slot);
  }
  srv_master_shutdown_loop();                                  // ③ MASTER_STOP 阶段
}
```

③ 里做 **change buffer 全量 merge**（仅 slow shutdown）：

```cpp
static bool srv_master_do_shutdown_tasks(...) {
  /* In very fast shutdown none of the following is necessary */
  if (srv_fast_shutdown >= 1) return (false);

  /* In case of slow shutdown we do ibuf merge (unless innodb_force_recovery
  is greater or equal to SRV_FORCE_NO_IBUF_MERGE). */
  if (srv_force_recovery < SRV_FORCE_NO_IBUF_MERGE) {
    srv_main_thread_op_info = "doing insert buffer merge";
    n_bytes_merged = ibuf_merge_in_background(true);
  }
  srv_shutdown_print_master_pending(last_print_time, 0, n_bytes_merged);
  return (n_bytes_merged != 0);
}
```

② 里做 **background drop tables**（fast=2 时跳过）：

```cpp
  /* In very fast shutdown none of the following is necessary */
  if (srv_fast_shutdown == 2) return (false);
  if (srv_force_recovery < SRV_FORCE_NO_BACKGROUND) {
    srv_main_thread_op_info = "doing background drop tables";
    n_tables_to_drop = row_drop_tables_for_mysql_in_background();
  }
```

#### 4.8 purge coordinator 收尾

```cpp
  /* Ensure that all records are purged if it is not a fast shutdown.
  This covers the case where a record can be added after we exit the loop above. */
  while (srv_fast_shutdown == 0 && n_pages_purged > 0) {
    n_pages_purged = trx_purge(1, srv_purge_batch_size, false);
  }

  /* This trx_purge is called to remove any undo records (added by
  background threads) after completion of the above loop. When
  srv_fast_shutdown != 0, a large batch size can cause significant
  delay in shutdown, so reducing the batch size to magic number 20
  (which was default in 5.5) ... */
  const uint temp_batch_size = 20;
  n_pages_purged = trx_purge(1, srv_purge_batch_size <= temp_batch_size
                                    ? srv_purge_batch_size : temp_batch_size, true);
  ut_a(n_pages_purged == 0 || srv_fast_shutdown != 0);
  ut_a(srv_get_task_queue_length() == 0);
```

注意最后那个 `ut_a`：任务队列必须空。以及 `temp_batch_size = 20` 这个魔数——注释解释它是"5.5 的默认值，希望足够清掉后台线程产生的 undo"。

undo truncate 在 fast shutdown 时被跳过：

```cpp
  bool normal_operation = (srv_shutdown_state == SRV_SHUTDOWN_NONE);
  bool in_fast_shutdown = (!normal_operation && srv_fast_shutdown > 0);
  /* Save time during a fast shutdown by skipping undo truncation.
  This does not affect correctness since undo tablespaces that need
  truncation can be truncated during or after startup. */
  if (in_fast_shutdown) return (false);
```

#### 4.9 page cleaner 收尾

```cpp
  if (srv_fast_shutdown == 2 ||
      srv_shutdown_state.load() == SRV_SHUTDOWN_EXIT_THREADS) {
    /* In very fast shutdown or when innodb failed to start, we
    simulate a crash of the buffer pool. We are not required to do
    any flushing. */
    goto thread_exit;
  }

  do {
    pc_request(ULINT_MAX, LSN_MAX);
    while (pc_flush_slot() > 0) { }
    pc_wait_finished(&n_flushed_lru, &n_flushed_list);
    n_flushed = n_flushed_lru + n_flushed_list;
    if (n_flushed == 0) std::this_thread::sleep_for(std::chrono::milliseconds(100));
  } while (srv_shutdown_state.load() < SRV_SHUTDOWN_FLUSH_PHASE);
```

`pc_request(ULINT_MAX, LSN_MAX)` 就是"把整个 buffer pool 刷干净"的请求。

主线程侧等待：

```cpp
static void srv_shutdown_page_cleaners() {
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_MASTER_STOP);
  ut_a(!srv_master_thread_is_active());
  srv_shutdown_set_state(SRV_SHUTDOWN_FLUSH_PHASE);

  buf_pool_wait_for_no_pending_io();

  for (uint32_t count = 0; buf_flush_page_cleaner_is_active(); ++count) {
    if (count >= SHUTDOWN_SLEEP_ROUNDS) {
      ib::info(ER_IB_MSG_1251);    // "Waiting for page_cleaner to finish flushing of buffer pool."
      count = 0;
    }
    os_event_set(buf_flush_event);
    std::this_thread::sleep_for(std::chrono::microseconds(SHUTDOWN_SLEEP_TIME_US));
  }
}
```

#### 4.10 `srv_shutdown_log()`：checkpoint 与 `flushed_lsn`（最关键的一段）

```cpp
/** Closes redo log. If this is not fast shutdown, it forces to write a
checkpoint which should be written for logically empty redo log. ...
After checkpoint is written, the flushed_lsn is updated within header of the
system tablespace. This is lsn of the last clean shutdown. */
static lsn_t srv_shutdown_log() {
  ut_a(srv_shutdown_state.load() == SRV_SHUTDOWN_FLUSH_PHASE);
  ut_a(!buf_flush_page_cleaner_is_active());

  if (srv_fast_shutdown == 2) {
    if (!srv_read_only_mode) {
      ib::info(ER_IB_MSG_1253);
      /* ... We must not write the lsn stamps to the data files, since at a
      startup InnoDB deduces from the stamps if the previous shutdown was clean. */
      log_stop_background_threads(*log_sys);
    }
    log_background_threads_inactive_validate();
    srv_shutdown_set_state(SRV_SHUTDOWN_LAST_PHASE);
    return (log_get_lsn(*log_sys));
  }

  if (!srv_read_only_mode) {
    log_make_empty_and_stop_background_threads(*log_sys);
  }
  log_background_threads_inactive_validate();
  buf_must_be_all_freed();

  const lsn_t lsn = log_get_lsn(*log_sys);

  if (!srv_read_only_mode) {
    /* Redo log has been flushed at the log_flusher's exit. */
    fil_flush_file_spaces();
  }
  srv_shutdown_set_state(SRV_SHUTDOWN_LAST_PHASE);

  ut_a(log_is_data_lsn(lsn) || srv_force_recovery >= SRV_FORCE_NO_LOG_REDO);
  ut_a(lsn == log_sys->last_checkpoint_lsn.load() ||
       srv_force_recovery >= SRV_FORCE_NO_LOG_REDO);
  ut_a(lsn == log_get_lsn(*log_sys));

  if (!srv_read_only_mode) {
    ut_a(srv_force_recovery < SRV_FORCE_NO_LOG_REDO);
    auto err = fil_write_flushed_lsn(lsn);       // ★ 干净关闭的标记
    ut_a(err == DB_SUCCESS);
  }
  ...
  return (lsn);
}
```

`log_make_empty_and_stop_background_threads()` 里有个循环，注释解释得很清楚：

```cpp
void log_make_empty_and_stop_background_threads(log_t &log) {
  log_files_dummy_records_disable(log);
  while (log_make_latest_checkpoint(log)) {
    /* It could happen, that when writing a new checkpoint,
    DD dynamic metadata was persisted, making some pages
    dirty (with the persisted data) and writing new redo
    records to protect those modifications. In such case,
    current lsn would be higher than lsn and we would need
    another iteration to ensure, that checkpoint lsn points
    to the newest lsn. */
  }
  log_stop_background_threads(log);
}
```

即：**写 checkpoint 这个动作本身会产生新的 redo**（持久化 DD dynamic metadata），所以必须循环到"checkpoint lsn == 当前 lsn"为止。这是典型的"不动点迭代"。

`ut_a(lsn == log_sys->last_checkpoint_lsn.load())` 就是那个不动点的验证。

`log_stop_background_threads()` 逐个等六个线程退出：

```cpp
  log.should_stop_threads.store(true);
  while (log_writer_is_active())         { os_event_set(log.writer_event); sleep(10us); }
  while (log_write_notifier_is_active()) { os_event_set(log.write_notifier_event); ... }
  while (log_flusher_is_active())        { os_event_set(log.flusher_event); ... }
  while (log_flush_notifier_is_active()) { os_event_set(log.flush_notifier_event); ... }
  while (log_checkpointer_is_active())   { os_event_set(log.checkpointer_event); ... }
  while (log_files_governor_is_active()) { os_event_set(log.m_files_governor_event); ... }
```

#### 4.11 `srv_shutdown_exit_threads()`：兜底逃生阀

```cpp
void srv_shutdown_exit_threads() {
  srv_shutdown_state.store(SRV_SHUTDOWN_EXIT_THREADS);
  if (srv_start_state == SRV_START_STATE_NONE) return;

  /* All threads end up waiting for certain events. Put those events
  to the signaled state. Then the threads will exit themselves after
  os_event_wait(). */
  for (i = 0; i < SHUTDOWN_SLEEP_ROUNDS; i++) {
    /* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM HERE OR EARLIER */
    for (const auto &thread_info : threads_to_stop) {
      if (srv_thread_is_active(thread_info.m_thread)) thread_info.m_notify();
    }
    ... srv_purge_wakeup() / os_event_set(buf_flush_event) / os_aio_wake_all_threads_at_shutdown() ...
    arch_wake_threads();
    if (log_sys != nullptr) {
      /* Preserve the log threads for the 75% of the total time we are waiting
      here until all threads are stopped. ... */
      if (!buf_flush_page_cleaner_is_active() || i >= SHUTDOWN_SLEEP_ROUNDS * 0.75) {
        log_stop_background_threads_nowait(*log_sys);
      } else {
        /* Ensure log threads are working. The redo log is like a blood,
        we need it for a lot of other systems to work. Ensure the blood flows. */
        log_wake_threads(*log_sys);
      }
    }
    bool active = os_thread_any_active();
    sleep(100us);
  }
```

这是**唯一**允许直接 `store` 到最终状态的函数（绕过 +1 约束），服务两种场景：正常关闭的最后一步、以及**启动失败时 abort**（此时状态机可能根本没走到 DD）。

三条精妙之处：

1. **"75% 的红线"**：redo 线程要留到最后才停，因为"redo 像血液，很多系统要靠它工作"——先让别的线程靠 redo 退出，最后才停 redo。
2. **最多等 60 秒**（600000 × 100μs）后放弃。这是**唯一有界的等待**。
3. `/* NOTE: IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM HERE OR EARLIER */` —— 给内核开发者的直接契约。

#### 4.12 `innodb_fast_shutdown` 三档行为对照（源码级）

sysvar 定义（**默认值 = 1**）：

```cpp
static MYSQL_SYSVAR_ULONG(
    fast_shutdown, srv_fast_shutdown, PLUGIN_VAR_OPCMDARG,
    "Speeds up the shutdown process of the InnoDB storage engine. Possible"
    " values are 0, 1 (faster) or 2 (fastest - crash-like).",
    nullptr, nullptr, 1, 0, 2, 0);
```

| 行为 | `=0`（slow） | `=1`（**默认**） | `=2`（crash-like） |
|---|---|---|---|
| 等 recovery 回滚完成 | ✅（`srv_fast_shutdown == 0`） | ❌ | ❌ |
| master PRE_DD：background drop tables | ✅ | ✅ | ❌ |
| master SHUTDOWN：**change buffer 全 merge** | ✅ | ❌（`>= 1` 时 return） | ❌ |
| purge：清完所有 undo | ✅（循环到 `n_pages_purged == 0`） | ❌（只做 batch=20 的一轮） | ❌ |
| undo 表空间 truncate | ✅ | ❌（`srv_fast_shutdown > 0` 即 fast） | ❌ |
| page cleaner 刷脏 | ✅ | ✅ | ❌（`goto thread_exit`） |
| 写 checkpoint（redo 逻辑清空） | ✅ | ✅ | ❌（只 `log_stop_background_threads`） |
| **`fil_write_flushed_lsn`（干净关闭标记）** | ✅ | ✅ | **❌ 不写** |
| 打 `ER_IB_MSG_1253` 警告 | ❌ | ❌ | ✅ |

#### 4.13 内存释放顺序（step 5）

```
btr_search_disable();     ← 注释：必须在关 buffer pool 与关 DD 之前
ibuf_close();
ddl_log_close();
log_sys_close();          ← 就是 log_sys_free()
recv_sys_close();
trx_sys_close();
lock_sys_close();
trx_pool_close();
dict_close();
dict_persist_close();
btr_search_sys_free();
undo_spaces_deinit();
os_aio_free();
que_close();
row_mysql_close();
srv_free();
fil_close();
pars_close();
pars_lexer_close();
buf_pool_free_all();      ← 释放所有 buffer pool 实例
clone_free();
arch_free();
dblwr::close();           ← 打印 ER_IB_DBLWR_BYTES_INFO 统计
os_thread_close();
sync_check_close();
```

注意 **`fil_close_all_files()`（step 3）在 `dblwr::close()`（step 5）之前**——先关文件句柄再释放内存。

### 五、插件 / 组件 deinit

#### 5.1 `plugin_shutdown()`

```cpp
void plugin_shutdown() {
  if (initialized) {
    size_t count = plugin_array->size();
    mysql_mutex_lock(&LOCK_plugin);
    reap_needed = true;

    /*
      We want to shut down plugins in a reasonable order, this will
      become important when we have plugins which depend upon each other.
      Circular references cannot be reaped so they are forced afterwards.
      TODO: Have an additional step here to notify all active plugins that
      shutdown is requested to allow plugins to deinitialize in parallel.
    */
    while (reap_needed && (count = plugin_array->size())) {
      reap_plugins();
      for (i = 0; i < count; i++) {
        plugin = plugin_array->at(i);
        if (plugin->state == PLUGIN_IS_READY &&
            strcmp(plugin->name.str, "binlog") == 0 && skip_binlog) {
          skip_binlog = false;
        } else if (plugin->state == PLUGIN_IS_READY) {
          plugin->state = PLUGIN_IS_DELETED;
          reap_needed = true;
        }
      }
      if (!reap_needed) {
        unlock_variables(&global_system_variables);
        unlock_variables(&max_system_variables);
      }
    }
    ...
    /* If we have any plugins which did not die cleanly, we force shutdown */
    for (i = 0; i < count; i++) {
      plugins[i] = plugin_array->at(i);
      if (plugins[i]->state == PLUGIN_IS_DELETED)
        plugins[i]->state = PLUGIN_IS_DYING;
    }
    mysql_mutex_unlock(&LOCK_plugin);

    for (i = 0; i < count; i++)
      if (!(plugins[i]->state & (PLUGIN_IS_UNINITIALIZED | PLUGIN_IS_FREED | PLUGIN_IS_DISABLED))) {
        LogErr(WARNING_LEVEL, ER_PLUGIN_FORCING_SHUTDOWN, plugins[i]->name.str);
        plugin_deinitialize(plugins[i], false);
      }
```

算法是**迭代引用计数回收 + 兜底强杀**：

1. 把所有 `PLUGIN_IS_READY` 的插件标成 `PLUGIN_IS_DELETED`，然后 `reap_plugins()` 回收引用计数归零的；
2. **循环直到没有可 reap 的**——这自然处理了插件之间的依赖（A 依赖 B，则 A 先被 reap，B 的引用才归零）；
3. **循环引用无法被 reap**，剩下的一律强制 `plugin_deinitialize()` 并打 `ER_PLUGIN_FORCING_SHUTDOWN`。

`binlog` 插件被特殊跳过一轮（`skip_binlog`），因为它是最底层的依赖。

> 注意：`plugin_deinit` 这个名字**不存在**（单个插件的 deinit 函数是 `plugin_deinitialize()`）。

#### 5.2 组件 deinit 与 `sys_var_end` 的顺序

```cpp
  /*
    Is this the best place for components deinit? It may be changed when new
    dependencies are discovered, possibly being divided into separate points
    where all dependencies are still ok.
  */
  log_error_stage_set(LOG_ERROR_STAGE_SHUTTING_DOWN);
  ...
  deinitialize_manifest_file_components();
  component_infrastructure_deinit();
  /*
    component unregister_variable() api depends on system_variable_hash.
    component_infrastructure_deinit() interns calls the deinit function
    of components which are loaded, and the deinit functions can have
    the component system unregister_ variable()  api's, hence we need
    to call the sys_var_end() after component_infrastructure_deinit()
  */
  sys_var_end();
```

#### 5.3 semisync 插件

```cpp
static int semi_sync_master_plugin_deinit(void *p) {
  if (ack_receiver == nullptr || repl_semisync == nullptr) return 0;
  THR_RPL_SEMI_SYNC_DUMP = false;
  if (unregister_trans_observer(&trans_observer, p)) { ... return 1; }
  if (unregister_binlog_storage_observer(&storage_observer, p)) { ... }
  ...
  LogErr(INFORMATION_LEVEL, ER_SEMISYNC_UNREGISTERED_REPLICATOR);
  deinit_logging_service_for_plugin(&reg_srv, &log_bi, &log_bs);
  return 0;
}
```

由 `plugin_shutdown()` 统一卸载。注意此时 dump 线程已在 `close_connections()` 里被 kill。

### 六、阶段 3：`mysqld_exit()`

```cpp
static void mysqld_exit(int exit_code) {
  assert((exit_code >= MYSQLD_SUCCESS_EXIT && exit_code <= MYSQLD_ABORT_EXIT) ||
         exit_code == MYSQLD_RESTART_EXIT);
  mysql_audit_finalize();
  Srv_session::module_deinit();
  delete_optimizer_cost_module();
  clean_up_mutexes();
  my_end(opt_endinfo ? MY_CHECK_ERROR | MY_GIVE_INFO : 0);
  destroy_error_log();
  log_error_read_log_exit();
#ifdef WITH_PERFSCHEMA_STORAGE_ENGINE
  shutdown_performance_schema();
#endif
#ifdef WITH_LOCK_ORDER
  LO_cleanup();
#endif
#if defined(_WIN32)
  if (hEventShutdown) CloseHandle(hEventShutdown);
  close_service_status_pipe_in_mysqld();
#endif
  exit(exit_code);
}
```

**注意顺序**：PFS 的关闭在 `my_end()` **之后**。因为 PFS  instrumentation 要一直活到内存分配器被销毁为止（否则 `my_end` 里的 free 操作无法被正确追踪/解绑）。

error log 同理：`destroy_error_log()` 在 `my_end()` 之后，因为它自己还要调 `flush_error_log_messages()`。

`destroy_error_log()`：

```cpp
void destroy_error_log() {
  // We should have flushed before this...
  // ... but play it safe on release builds
  flush_error_log_messages();
  if (error_log_initialized) {
    error_log_initialized = false;
    error_log_file = nullptr;
    mysql_mutex_destroy(&LOCK_error_log);
    log_builtins_exit();
  }
}
```

#### 退出码

| 值 | 名字 | 语义 |
|---|---|---|
| 0 | `MYSQLD_SUCCESS_EXIT` | 正常关闭（`signal_hand_thr_exit_code` 初值） |
| 1 | `MYSQLD_ABORT_EXIT` | 失败，**systemd 不要自动重启** |
| 2 | `MYSQLD_FAILURE_EXIT` | daemon 化中间层失败 / fatal signal（`handle_fatal_signal` → `_exit(2)`，**完全不 clean_up**） |
| 16 | `MYSQLD_RESTART_EXIT` | SIGUSR2（`RESTART` 语句）→ 由 systemd / `mysqld_safe` 识别为"重启" |

`signal_hand_thr_exit_code` 声明：`std::atomic<int> signal_hand_thr_exit_code(MYSQLD_SUCCESS_EXIT);`

#### `unireg_abort()` 与正常关闭的关系

| | `unireg_abort()`（启动失败） | 正常关闭 |
|---|---|---|
| 调用点 | 启动各阶段失败（80+ 处） | `mysqld_main` 末尾 |
| audit reason | `..._REASON_ABORT` | `..._REASON_SHUTDOWN`（在 `mysqld_main` 里先发出） |
| 信号线程 | **主动 `pthread_kill(SIGTERM)` + join** | 信号线程已经自行退出，只 join |
| daemon 父进程通知 | `signal_parent(pipe_write_fd, 0)` | `signal_parent(pipe_write_fd, 1)`（在进入循环时已发） |
| 共享 | 都调 `clean_up()`（靠 `set_server_shutting_down()` 幂等） | 同 |

### 七、关闭期间崩溃会怎样

| 阶段 | 后果 |
|---|---|
| `close_connections()` 期间 | 事务可能未提交/未回滚 → 重启走崩溃恢复；binlog 与引擎由 XA/binlog recover 协调 |
| `srv_pre_dd_shutdown()`（state ≤ PURGE） | `fil_write_flushed_lsn` 还没执行 → 下次判定为不干净 → 崩溃恢复 |
| `SRV_SHUTDOWN_FLUSH_PHASE`（刷脏中） | 部分脏页落盘、checkpoint 未推进 → redo 重放可修复；半页写由 doublewrite 兜底 |
| `srv_shutdown_log()` 之后 | `flushed_lsn` 已写 → 下次判定干净关闭，不重放 redo |
| `plugin_shutdown()` 之后 | 所有 SE 已 flush，只剩内存释放，无数据风险 |
| fatal signal（SIGSEGV 等） | `handle_fatal_signal()` → stack trace + core + `_exit(2)`，**完全不 clean_up** |

**一个真实的 crash 路径**（源码级）：`srv_pre_dd_shutdown()` 里

```cpp
  for (size_t count = 0; count < 10; ++count) {
    const auto threads_count = srv_conc_get_active_threads();
    if (threads_count == 0) break;
    ib::warn(ER_IB_MSG_1154, threads_count);
    std::this_thread::sleep_for(std::chrono::seconds(1));
  }
  /* Crash if some query threads are still alive. */
  ut_a(srv_conc_get_active_threads() == 0);
```

如果有查询线程卡在 InnoDB 内部超过 10 秒，**服务器自己崩**。这是"宁可崩也不带着不一致继续"的另一个体现。

---

## ★ 本机制里的工程实现技法

### 一、C++ 特性

| 特性 | 用在哪 | 收益 | 代价 / 反直觉处 |
|---|---|---|---|
| **访问者模式（`Do_THD_Impl`）** | `Set_kill_conn` / `Call_close_conn` + `do_for_all_thd()` | 遍历逻辑与"对每个 THD 做什么"解耦；两个 functor 复用同一套遍历 | `do_for_all_thd()` 内部要按分区加锁，functor 里再操作 THD 需注意锁序 |
| **`std::function<void()>` 表驱动** | `Thread_to_stop::m_notify` | 把"怎么唤醒这个线程"做成可重复调用的函数对象，表驱动统一处理 | 每个条目一次 `std::function` 构造（启动期一次性，无所谓）；但 lambda 捕获要小心生命周期 |
| **`std::atomic<enum>` 单向状态机** | `srv_shutdown_state` | 后台线程无锁读"现在什么阶段"；主线程用 `srv_shutdown_set_state()` 强制 +1 | 只能单向推进；兜底路径（`srv_shutdown_exit_threads`）必须绕过约束直接 `store` |
| **`std::atomic<int>` 退出码** | `signal_hand_thr_exit_code` | 信号线程写、主线程读，无需加锁 | |
| **RAII 锁** | `MUTEX_LOCK(lock, &LOCK_thd_list[i])` | 异常/早退路径自动解锁 | |
| **`[[fallthrough]]`** | `case SIGUSR2:` → `case SIGTERM:` | 显式表达"故意穿透"，避免编译器警告 | 阅读时容易漏看 |

### 二、经典算法的落地

| 维度 | 教科书原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| **插件依赖卸载** | 拓扑排序后逆序销毁 | **迭代引用计数回收**：反复标记 `PLUGIN_IS_DELETED` + `reap_plugins()`，直到无可回收；剩下的一律强杀 | 插件依赖图可能有**环**（注释明说 "Circular references cannot be reaped"），拓扑排序处理不了，只能靠引用计数 + 兜底强杀 |
| **线程停止** | 条件变量 / join | **轮询 + 反复 `os_event_set()` + 100μs sleep**，每 60s 打日志 | 线程退出条件各不相同，统一表驱动最省事；代价是忙等，但关闭路径上无所谓。且"反复通知"天然对抗信号丢失 |
| **checkpoint 收敛** | 一次 checkpoint | **不动点迭代**：`while (log_make_latest_checkpoint(log));` | 写 checkpoint 本身会产生新 redo（DD dynamic metadata 持久化），必须迭代到 lsn 不再前进 |
| **redo 线程的保留策略** | 无对应原型 | `srv_shutdown_exit_threads()` 里保留 redo 线程到总等待时间的 75% | "redo 像血液，很多系统要靠它工作"——先让别的线程靠 redo 退出，最后才停 redo。这是把"依赖顺序"编码成时间比例的一个务实做法 |
| **关闭的幂等** | 通常靠状态检查 | 两处独立保护：`if (!connection_events_loop_aborted())`（信号线程侧）+ `set_server_shutting_down()`（`clean_up` 侧） | 因为关闭由两个不同的线程分别触发第一阶段与第二阶段 |

### 三、复杂体系与设计模式的协作结构

```
          信号线程                              主线程
             │                                    │
   close_connections()                   connection_event_loop() 返回
             │                                    │
   ┌─────────┴─────────┐                          │
   │ ① listener 关闭    │                   GTID 落盘（InnoDB 还活着）
   │ ② Set_kill_conn    │                          │
   │   （温和，跳过 dump）│                   join(signal_thread)
   │ ③ end_slave()      │                          │
   │ ④ dump 线程 kill    │                   clean_up()
   │ ⑤ sleep(2)          │                     ├─ ha_pre_dd_shutdown ──┐
   │ ⑥ Call_close_conn   │                     │                        │ 契约：
   │   （强制断 vio）     │                     ├─ dd::shutdown()  ◄─────┘ 停掉访问
   │ ⑦ wait_till_no_thd  │                     ├─ binlog cleanup         DD 的线程
   │ ⑧ wait_till_no_conn │                     ├─ plugin_shutdown        （状态机
   └─────────────────────┘                     │    └─ hton->panic       SRV_SHUTDOWN_DD）
             │                                 │        └─ srv_shutdown
        my_thread_exit()                       └─ pid 文件 / 组件 / sys_var
```

| 模式 / 体系 | 代码落点 | 结构说明 |
|---|---|---|
| **访问者** | `Do_THD_Impl` + `do_for_all_thd()` | 遍历与操作解耦 |
| **观察者（delegate）** | `RUN_HOOK(server_state, before_server_shutdown / after_server_shutdown)` | 复制 / 审计模块在关闭两端被通知 |
| **两阶段终止（Phase barrier）** | `handlerton::pre_dd_shutdown` vs `handlerton::panic` | 把"DD 关闭"变成关闭序列中的显式屏障 |
| **状态机（单向）** | `srv_shutdown_t` 九态 + `srv_shutdown_set_state()` 的 +1 断言 | 后台线程据此判断"我现在能做什么" |
| **表驱动** | `threads_to_stop[]` | 声明式描述线程停止策略 |
| **幂等关闭** | `set_server_shutting_down()` | `clean_up()` 可被两条路径各调一次 |

---

## 可观测性

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `innodb_fast_shutdown` | **1** | Global | 0=slow（purge + ibuf merge）/ 1=只刷脏 / 2=crash-like |
| `innodb_force_recovery` | 0 | Global | 影响是否跳过 ibuf merge / redo 应用 |
| `rpl_stop_replica_timeout` | 60 | Global | `terminate_slave_threads` 的超时（SQL+IO 共享） |
| `innodb_purge_batch_size` | 300 | Global | purge 收尾时降到 20 |
| `innodb_force_recovery` 相关断言 | — | — | `srv_force_recovery < SRV_FORCE_NO_IBUF_MERGE` 才做 ibuf merge |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 关闭卡在哪一步 | error log | `ER_DEPART_WITH_GRACE`（"Giving N client threads a chance to die gracefully"）→ `ER_SHUTTING_DOWN_REPLICA_THREADS` → `ER_DISCONNECTING_REMAINING_CLIENTS` → `ER_WAITING_FOR_NO_THDS` / `ER_WAITING_FOR_NO_CONNECTIONS` → `ER_BINLOG_END` → `ER_SERVER_SHUTDOWN_COMPLETE` |
| 哪个连接被强制断开 | error log | `ER_FORCE_CLOSE_THREAD`（"`%s: Forcing close of thread %ld  user: '%-.48s'`"） |
| InnoDB 停线程卡住 | error log | `ER_IB_MSG_1248` "Waiting for %s to exit."、`ER_IB_MSG_1251` "Waiting for page_cleaner to finish flushing of buffer pool."、`ER_IB_MSG_1152`（purge）——**每 60 秒一条** |
| 活动线程没归零 | error log + 崩溃 | `ER_IB_MSG_1154`（"Query counter shows N queries still inside InnoDB at shutdown"），10 秒后 `ut_a` 崩溃 |
| fast=2 的告警 | error log | `ER_IB_MSG_1253`："...At the next mysqld startup InnoDB will do a crash recovery!" |
| 关闭是否完成 | error log | `ER_SERVER_SHUTDOWN_COMPLETE`；`SHOW ENGINE INNODB STATUS` 里的 "Shutdown completed; log sequence number N" |
| 谁发的关闭 | error log | `ER_SERVER_SHUTDOWN_INFO`："<via user signal>" vs 用户名 |
| systemd 视角 | journal | `STOPPING=1` / `STATUS=Server shutdown complete` |
| doublewrite 统计 | error log | `ER_IB_DBLWR_BYTES_INFO`（`dblwr::close()` 打印） |

---

## Misc

### 扩展点：加一个关闭步骤要改哪

1. **如果你的插件有后台线程会访问 DD** → 必须实现 `handlerton::pre_dd_shutdown`（对应 InnoDB 的 `innodb_pre_dd_shutdown`），在其中停线程。否则 DD 关闭后你会访问悬空对象。
2. **如果你的插件需要在 SE 刷盘之后做点事** → 放在 `plugin_shutdown()` 里（`hton->panic`），但注意此时 DD 已关闭。
3. **如果你要在关闭时持久化状态** → 必须在 `clean_up()` 的 `plugin_shutdown()` **之前**（参考 `save_gtids_of_last_binlog_into_table()` 的位置）；`clean_up()` 里 `gtid_server_cleanup()` 那个位置只适合 delete 对象。
4. **如果你加了 InnoDB 后台线程** → 在 `threads_to_stop[]` 里加一行，并在 `srv_shutdown_exit_threads()` 里处理（注释原文："IF YOU CREATE THREADS IN INNODB, YOU MUST EXIT THEM HERE OR EARLIER"）。
5. **加一个新退出码** → 改 `sql/sql_const.h` + `mysqld_exit()` 的 assert + 外部管理者语义。

### 坑与已知缺陷

1. **关闭可以永久 hang**：`wait_till_no_thd()` / `wait_till_no_connection()` **无超时**。一个卡在不可中断等待上的连接线程就能让 mysqld 关不掉。
2. **10 秒后自毁**：`srv_pre_dd_shutdown()` 里 `ut_a(srv_conc_get_active_threads() == 0)`——活动线程 10 秒不归零就崩。这是"关闭期间服务器自己 crash"的真实源码路径。
3. **`SHUTDOWN` 语句不接受参数**：协议里的 `SHUTDOWN_WAIT_*` 枚举值全部撞 `ER_NOT_SUPPORTED_YET`。想要 PG 那种 `smart/fast/immediate` 档位，社区版没有——档位只存在于引擎层（`innodb_fast_shutdown`）。
4. **`innodb_fast_shutdown=2` 不是"快"那么简单**：它故意不写 `flushed_lsn`，让下次启动走完整崩溃恢复。若同时 `ALTER INSTANCE DISABLE INNODB REDO_LOG`，会被强制降级为 1（否则无法恢复）。
5. **`--init-file` 失败不 abort**（初始化路径）——见 [`01_initialize.md`](01_initialize.md)。
6. **`plugin_shutdown()` 的循环引用只能强杀**：`ER_PLUGIN_FORCING_SHUTDOWN` 出现即说明有插件依赖成环。
7. **`plugin_deinit` / `kill_server` / `kill_all_server_threads` / `COND_thread_count` / `release_error_log` / `logs_empty_and_mark_files_at_shutdown`** —— 这些名字在 8.0.39 **都不存在**（多为 5.7 遗留），真实替代见正文各节的核实表。
8. **PFS 在 `my_end()` 之后才关**：如果自定义代码在 `mysqld_exit()` 里插东西，要注意内存分配器已销毁。
9. **社区边界**：社区版**没有**"关闭超时后强制退出"（只有 InnoDB 层那个 60 秒的 `srv_shutdown_exit_threads`）、没有"关闭进度可查询"、没有 PG 式的连接层关闭档位。

### 易混淆命名

| 名字 | 实际是什么 |
|---|---|
| `handle_shutdown` | 服务端**不存在**。Windows 侧是 `handle_shutdown_and_restart()`；SQL 侧是 `Sql_cmd_shutdown::execute()` → `shutdown()` |
| `kill_mysql()` vs `mysqld_exit()` | 前者是"发 SIGTERM 给信号线程"（触发关闭）；后者是"真正 `exit()`"（关闭最后一环） |
| `close_connections()` vs `clean_up()` | 前者在**信号线程**里跑（停连接），后者在**主线程**里跑（卸子系统） |
| `ha_pre_dd_shutdown()` vs `ha_binlog_end()` | 前者是"DD 关闭前停 SE 线程"；后者是"binlog 关闭前让 SE 了结对 binlog 的依赖"。两个方向相反的屏障 |
| `srv_pre_dd_shutdown()` vs `srv_shutdown()` | 前者停在 `SRV_SHUTDOWN_DD`（由 `clean_up` 第 1 步触发）；后者从 `SRV_SHUTDOWN_DD` 走到 `SRV_SHUTDOWN_EXIT_THREADS`（由 `plugin_shutdown` 触发） |
| `plugin_shutdown()` vs `plugin_deinitialize()` | 前者是"卸载全部"的入口；后者是"卸载单个"。`plugin_deinit` 不存在 |
| `MYSQLD_ABORT_EXIT(1)` vs `MYSQLD_RESTART_EXIT(16)` | 1 = 别自动重启；16 = 请重启我 |
| `log_stop_background_threads` vs `log_make_empty_and_stop_background_threads` | 前者只停线程（fast=2）；后者先做不动点 checkpoint 再停（fast=0/1） |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → SHUTDOWN Statement*
- *MySQL 8.0 Reference Manual → innodb_fast_shutdown*（三档语义与"下次启动要崩溃恢复"的说明）
- *MySQL 8.0 Reference Manual → Server Shutdown Process*（连接关闭与刷新行为）
- *MySQL 8.0 Reference Manual → mysqld_safe / systemd*（退出码 16 的重启语义）
- WorkLog: [WL#7546](https://dev.mysql.com/worklog/task/?id=7546) —— binlog Stop event 实现（`sql/binlog.cc` 里 `TODO(WL#7546)` 原文引用）
- WorkLog: [WL#6391](https://dev.mysql.com/worklog/task/?id=6391) —— 事务型 DD，是"为什么需要 `pre_dd_shutdown` + 九态机"的上游动因

**Bug 论坛**
- bugs.mysql.com：搜索 "shutdown hang" / "waiting for threads to die" 可查关闭 hang 的报告（多数根因落在 `wait_till_no_thd` 与 InnoDB 后台线程停止阶段）

**内核月报 / 技术文章**
- 腾讯云数据库内核月报：InnoDB shutdown 与 `innodb_fast_shutdown` 专题（注意：月报函数名多基于 5.7，`logs_empty_and_mark_files_at_shutdown` 等在 8.0.39 已不存在，需回源码核实）

**相关文档**
- 启动流程与 `unireg_abort`：见 [`02_startup.md`](02_startup.md)
- 初始化与退出码：见 [`01_initialize.md`](01_initialize.md)
- 崩溃恢复（关闭不干净的后果）：见 [`../../innodb/recovery.md`](../../innodb/recovery.md)
- redo / checkpoint 机制：见 [`../../innodb/redo_log.md`](../../innodb/redo_log.md)
- doublewrite 收尾统计：见 [`../../innodb/dblwr.md`](../../innodb/dblwr.md)
- 插件框架：见 [`../plugin/plugin.md`](../plugin/plugin.md)
- 复制线程生命周期：见 [`../replication/replica.md`](../replication/replica.md)
