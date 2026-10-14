# 线程池（Thread Pool）深度解析

> 基于 **MySQL 26.7.0**（tag `mysql-26.7.0`）源码，涵盖 `plugin/thread_pool/` 全部核心模块：插件接入、线程组与线程角色、连接生命周期、epoll 事件采集、调度算法、credit/MTL 并发控制、stall 检测、wait 回调协同、PFS 观测。
>
> **边界**：本篇讲 **pool-of-threads 线程池**；`one-thread-per-connection` 的 Per_thread 模型见 [`connection_threading.md`](connection_threading.md)。MariaDB / Percona 的同源实现与三方对比见 [`thread_pool_mariadb_percona.md`](thread_pool_mariadb_percona.md)。
>
> ⚠️ **版本注意**：线程池在 8.0.x 中是**闭源商业插件**，社区版源码树只提供 `include/mysql/thread_pool_priv.h` 接入框架而无实现。本篇的源码行号均对应 **26.7.0**——线程池首次进入社区版的版本（WL #17296）。本地 8.0.39 树上找不到 `plugin/thread_pool/` 是正常的，不是遗漏。

## 目录

- [概述](#概述)
  - [是什么](#是什么) / [用途](#用途) / [版本演进](#版本演进)
- [理论基础](#理论基础)
  - [设计思想与权衡](#设计思想与权衡) / [理论溯源](#理论溯源) / [算法与数据结构](#算法与数据结构) / [他库对比与演进动机](#他库对比与演进动机)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [一、插件接入与生命周期](#一插件接入与生命周期)
  - [二、线程组与线程角色](#二线程组与线程角色)
  - [三、连接生命周期](#三连接生命周期)
  - [四、事件采集层（epoll）](#四事件采集层epoll)
  - [五、调度与执行](#五调度与执行)
  - [六、并发控制：credit 与 MTL](#六并发控制credit-与-mtl)
  - [七、stall 检测与引擎协同](#七stall-检测与引擎协同)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

线程池是一个**以服务器插件形式实现的可替换连接调度器**：它接管"由谁、在什么时候、用哪个 OS 线程去执行哪条连接上的语句"这一职责，把默认的"每连接独占一个线程"改为"有限线程组多路复用大量连接"。

### 用途

一句话定位：**在连接数远超 CPU 核数时，用"限制同时推进的语句数"换取吞吐不崩塌。**

官方文档列举它解决的三个问题（`thread-pool.html`）：

1. **CPU cache 失效** —— "Too many thread stacks make CPU caches almost useless in highly parallel execution workloads."
2. **上下文切换开销** —— "context switching overhead is high... The thread pool controls the number of active threads."
3. **资源争用** —— "Too many transactions executing in parallel increases resource contention. In `InnoDB`, this increases the time spent holding **central mutexes**."

**切入点**：为什么 4096 个并发连接时，MySQL 的吞吐会掉到 282 QPS（MariaDB 官方 benchmark，24 核 POINT_SELECT），而开启线程池后仍有 143934 QPS？差了 **500 倍**。本篇要回答的就是：线程池到底限制了什么，才让这条断崖被抹平。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.5 | 企业版引入线程池，**插件**形式，闭源（设计者 Mikael Ronström）。同期 MariaDB 5.5 独立开源实现（**内建**，GPL） |
| 5.6 / 5.7 | 企业版持续演进；社区版始终只有 `thread_pool_priv.h` 接入框架 |
| 8.0.x | 结构稳定。8.0.14 引入 `admin_port`；**8.0.31 引入 `thread_pool_transaction_delay`** |
| 8.4 LTS | 无机制性变化 |
| 9.7 LTS（2026-04） | 企业版→社区版下放的**前奏**：8 组能力进社区（Hypergraph、Telemetry、3 个 GR 组件、applier metrics、JSON Duality View DML），**线程池不在其中** |
| **26.7.0（2026-07-28）** | **① 线程池插件从企业版变为社区版可用（WL #17296）；② `thread_pool_max_unused_threads` 默认值由 2 改为 32（Bug #39405207）** |

> 26.7 同时是**日历版本号（CalVer）**的第一个版本：`YY.M.P` = 26 年 7 月。9.7 是最后一条使用旧顺序版本号的产品线。

---

## 理论基础

### 设计思想与权衡

#### 核心决策：限制"事务数"而不是"线程数"

⚠️ **源码基线说明**：本仓库社区版 `sql/` 下**没有**线程池源码（`sql/threadpool*` 全仓 0 命中）。企业版实现被预置在 **`.tmp_tp/`**（15 个文件，`thread_pool.cc` 约 5100 行）。下面所有行号均为 `.tmp_tp/` 内实测。

**MTL（Max Transactions Limit）全套**（`thread_pool.cc:463`）：

```cpp
/**
  Returns true if MTL has been temporarily suspended due to too many
  long running transactions.
 */
static bool is_mtl_suspended(const tp_group_t *my_tp_group) {
  return my_tp_group->time_of_last_longrunning_trxs_check > BEGINNING_OF_EPOCH;
}

/**
  Returns the duration in which a thread group must have a normal number
  of long running transactions before MTL can be re-enabled.
*/
static Misec longrun_grace_period() {
  return std::clamp(15 * ATMC_longrun_trx_limit.load(std::memory_order_relaxed),
                    Misec(5 * 1000), Misec(30 * 1000));
  //  ★★ 时间参数①：clamp(15 × longrun_trx_limit, 5000ms, 30000ms)
}

/**
  Returns the minimum time which must elapse between each check for long
  running transactions.
*/
static Misec longrun_check_interval() {
  return ATMC_longrun_trx_limit.load(std::memory_order_relaxed) / 2;
  //  ★★ 时间参数②：longrun_trx_limit / 2（默认 1000ms）
}

/**
  Returns the configured thread_pool_max_transactions_limit_per_tg for this
  thread group. That is, 0 for the admin threadgroup and the value of
  thread_pool_max_transactions_limit_per_tg otherwise.
 */
std::uint32_t configured_max_transactions_limit_per_group(const tp_group_t *tg) {
  return tg == admin_thread_group
             ? 0                                  // ★★ admin 组永不受 MTL 限制
             : thread_pool_max_transactions_limit_per_tg;
}
```

**"限事务数"的计数点**（`tp_process_event()`，`thread_pool.cc:1123`）——注意计的是**线程**上的事务，不是线程本身：

```cpp
  // Increment trxn threads count if configured
  // thread_pool_max_transactions_limit is set for the thread group.
  if (txnlim != 0) {
    my_tp_group->trxn_threads++;                  // ★ 真正被限的是"并发事务数"
  }
  auto trxn_threads_decrement_guard = create_scope_guard([&]() {
    if (txnlim != 0) { my_tp_group->trxn_threads--; }
  });
```

**MTL 挂起判定**（`check_longrunning_transactions_and_adjust_MTL()`，`thread_pool.cc:598`）：

```cpp
  auto longrunning_threshold =
      now - ATMC_longrun_trx_limit.load(std::memory_order_relaxed);

  std::uint32_t longrunning = std::count_if(      // ★ 统计"attach 超过 longrun_trx_limit 的线程数"
      my_tp_group->group_threads,
      my_tp_group->group_threads + my_tp_group->max_thread_ids_in_group,
      [&](const tp_thread_t &t) {
        return t.time_of_attach > BEGINNING_OF_EPOCH &&
               t.time_of_attach < longrunning_threshold;
      });

  if (!is_mtl_suspended(my_tp_group)) {
    if (longrunning >= txnlim) {                  // ★★ 长事务数 ≥ MTL/组 → 挂起 MTL（放开限制防死锁）
      LogErr(INFORMATION_LEVEL, ER_THREAD_POOL_MTL_DISABLE,
             my_tp_group->group_idx, longrunning);
      my_tp_group->time_of_last_longrunning_trxs_check = now;
    }
    return;
  }
```

> ★★ **为什么必须能"挂起 MTL"**：如果 MTL 被长事务占满且不放开，新连接连进来连 `BEGIN` 都发不出去，而长事务又可能在等这些新连接释放的资源 → 死锁。所以"长事务把配额占满"时**反向**放开限制，这是整套设计里最反直觉的一步。

**建线程时给 listener 留的名额**（`tp_create_worker_threads()`，`thread_pool.cc:1256`）——★ 硬编码 `+1`：

```cpp
      if (txnlim != 0) {
        mysql_mutex_lock(&cur_group->LOCK_query_threads_count);
        if (worker_type == Worker_thread_type::QUERY_WORKER_THREAD) {
          // Allow one more for the waiting thread in case all trxn
          // threads are tied to long-running transactions. If we don't do this
          // we will not detect credit increases due to MTL changes and kill
          // requests. This must matche the // condition used in
          // is_credit_available() + one extra thread.
          if (cur_group->query_threads_count <= txnlim + 1 ||     // ★★ +1 给 listener
              is_mtl_suspended(cur_group)) {
            thread_pool_query_worker_threads++;
            cur_group->query_threads_count++;
          } else {
            mysql_mutex_unlock(&cur_group->LOCK_query_threads_count);
            return 0;                              // ★ 拒绝建线程
          }
        }
        mysql_mutex_unlock(&cur_group->LOCK_query_threads_count);
      }
```

> ⚠️ **`tp_add_group()` 不存在**——group 是编译期静态数组，`thd_pool_init()` 里 `new (cur_group) tp_group_t` 就地构造（`thread_pool.cc:3279`）：
> ```cpp
>     tp_group_t *cur_group = &(tp_group_list[i]);
>     new (cur_group) tp_group_t;      // ★ placement new，无任何动态增删
>     cur_group->group_low_level_cntx = tp_group_low_level_init(cur_group);
> ```
> `thread_pool_size` 是 `PLUGIN_VAR_READONLY`，**启动后不可变**。

**`thread_pool_size` 是线程"组"数，不是线程数**（`thread_pool.h:116`）：

```cpp
// Thread pool size is the number of thread groups.          // ★ 官方注释原文
inline constexpr auto DEF_POOL_SIZE = 16;
inline constexpr auto TP_MIN_POOL_SIZE = 1;
inline constexpr int TP_MAX_POOL_SIZE = MAX_NORMAL_THREAD_GROUPS;   // = 512
```

```cpp
// thread_pool.h:59
inline constexpr int MAX_NORMAL_THREAD_GROUPS = 512;
inline constexpr int MAX_ADMIN_THREAD_GROUPS = 1;         // ★ 多出的 1 个 admin 组
// Normal thread groups + admin thread group to allow managenent operations
// even when all the normal thread groups are blocked
inline constexpr int MAX_THREAD_GROUPS =
    MAX_NORMAL_THREAD_GROUPS + MAX_ADMIN_THREAD_GROUPS;   // = 513
inline constexpr auto MAX_THREADS_PER_GROUP = 4096;
inline constexpr auto MAX_EVENTS_PER_WAIT_CALL = 32;      // ★ 一次最多取 32 个事件
```

> ⚠️ `DEF_POOL_SIZE = 16` **只是占位符**——真实默认值在 `update_config_defaults()` 里按 `clamp(my_num_vcpus()/2, 1, 16)` 计算（`thread_pool_plugin.cc:837-839`），且只在变量来源为 `COMPILED`（未被命令行/配置文件显式设置）时才覆盖。

这是 26.7 最重要的设计判断。此前所有实现（含企业版旧版、MariaDB、Percona）限制的都是**线程数**；26.7 引入 `thread_pool_max_transactions_limit`（MTL）限制**并发事务数**。

为什么？因为压垮 InnoDB 的不是线程多，而是**并发事务对共享结构（central mutex）的争用**。一个正在等行锁的线程不占 CPU，却仍持有事务——只数"活跃线程"会漏掉它。

**代价与失效场景**：

- MTL 打满时新事务会 hang，且 `thread_pool_longrun_trx_limit` 也救不回来时，**只能靠 `TP_CONNECTION_ADMIN` 特权连接**进入——而"该权限默认不授予任何人"（官方原文）
- 长事务背压带滞回（挂起阈值 `txnlim`，恢复阈值 `txnlim/2` + grace period），意味着**恢复不是即时的**，短事务在长事务降下来之前一直受限
- `max_active_query_threads` 明确标注 **"Use is dangerous and could lead to deadlocks"**

**源码落点**：这个决策的证据就是 6.1 的三档判定本身——第一档用的是 `trxn_threads`（事务数），只有它未配置时才回落到 `threads_active`（线程数）：

```cpp
std::uint32_t txnlim = configured_max_transactions_limit_per_group(my_tp_group);
if (txnlim != 0)
  res = (my_tp_group->trxn_threads <= txnlim || is_mtl_suspended(my_tp_group));
else if (thread_pool_max_active_query_threads)
  res = (my_tp_group->threads_active < thread_pool_max_active_query_threads);
else
  res = (my_tp_group->threads_active <= my_tp_group->max_active_threads);
```

注意 `trxn_threads` 与 `threads_active` 是**两个独立计数器**（见 9.4）：`threads_active` 会因 `wait_begin` 而递减（线程睡着就不算活跃），`trxn_threads` 则只看"是否持有事务"——**睡着的事务照样计数**。这正是"只数活跃线程会漏掉等锁事务"在代码里的体现。

#### 决策二：靠"引擎主动上报"而非"超时猜测"

⚠️ **勘误**：`start_io()` / `end_io()` **不存在**（`.tmp_tp/` 全目录 0 命中，那是 MariaDB/Percona 老 threadpool 的回调名）；`tp_wait_begin` / `tp_wait_end` 也**不存在**。企业版真名是 **`thd_pool_wait_begin()` / `thd_pool_wait_end()`**。

**注册点只有一行**（`thread_pool_plugin.cc:84`）：

```cpp
/*
  Pool of threads scheduler.

  This is the scheduler we add if we have a pool of threads
  implementation.

  Linux: poll, epoll
  FreeBSD/Mac OS X: poll
  Solaris: poll
  Windows: WSAPoll
*/
static THD_event_functions thread_pool_event_functions = {
    thd_pool_wait_begin, thd_pool_wait_end, thd_pool_post_kill_notification};

static Connection_handler_functions thread_pool_handler_functions = {
    0,  // Temporary value
    thd_pool_add_connection, thd_pool_end};
```

```cpp
// sql/conn_handler/connection_handler_manager.h:42 —— server 侧结构体定义
struct THD_event_functions {
  void (*thd_wait_begin)(THD *thd, int wait_type);
  void (*thd_wait_end)(THD *thd);
  void (*post_kill_notification)(THD *thd);
};
```

```cpp
// include/mysql/thread_pool_priv.h:54
struct Connection_handler_functions {
  /** The maximum number of threads this connection handler will create. */
  uint max_threads;
  add_connection_t add_connection;
  end_t end;
};
```

**★ `thd_pool_wait_begin()` 全文**（`thread_pool.cc:3800`）：

```cpp
/*
  An API call to the thread pool.  Call this when a client/socket command
    is being processed and a stall (long held mutex or disk IO) is about to
    occur.  This notifies the thread pool that an OS thread is about to sleep.

  @param thd       Connection handle. If nullptr is passed, current_thd is used.
  @param wait_type An enum value from the enum thd_wait_type (defined
                   in include/mysql/service_thd_wait.h) but passed as int to
                   preserve compatibility with exported service api.

  @note
    Called when the mysql server or storage engine is about to stall
    This will:
      - Adjust the active thread count so other threads may start
      - Possibly wake more threads to process new requests

    This may wake another OS thread
*/
void thd_pool_wait_begin(THD *thd, int wait_type) {
  ...
  client_cntx = (tp_client_low_level_t *)thd_get_scheduler_data(thd);
  if (client_cntx != nullptr) {
    my_tp_group = tp_group_low_level_get_tp_group(client_cntx->group);
    my_thread_data = client_cntx->processing_thread;
    // Can only call this from the original tp_worker_thread
    if (my_thread_self() != my_thread_data->thread_handle.thread) return;

    // thd_wait_begin() must not be called again without an intervening
    // thd_wait_end()
    assert(client_cntx->wait_type == THD_WAIT_NONE);
    if (client_cntx->wait_type > THD_WAIT_NONE) return;

    client_cntx->wait_type = static_cast<THD_wait_type>(wait_type);

    if (!my_thread_data->stalled) {
      my_thread_data->not_stalled = true;
      dec_active_threads(my_tp_group, my_thread_data);
      //  ★★★ 核心：归还 credit，让别人能起来干活
    }

    mysql_mutex_lock(&my_tp_group->LOCK_group);

    if (my_tp_group->waiting_thread == my_thread_data) {
      /*
        Since I'm stalling, I'll no longer be the waiting thread.
        It's important to create a thread to avoid a deadlock here
        since we have no check in stall check thread for a missing
        waiter thread.
      */
      my_tp_group->waiting_thread = nullptr;
      level = WAKE_OR_CREATE_ONE;      // ★ listener 自己要睡 → 必须"唤醒或新建"顶上
    }

    assert(wait_type < THD_WAIT_LAST);
    my_tp_group->stats.wait_counts[wait_type]++;   // ★ I_S *_WAITS 列的来源

    /*
      Wake another thread to see if necessary to handle waiting thread or there
      are queries ready to execute. Use a reserve thread if necessary. It is
      essential for performance to ensure a thread gets woken up here
      unconditionally to check if anything needs to be done to get things going
      again.

      The most important thing here is not really to wake the thread up. It's
      quite likely that it will go to sleep again. However even if this happens
      it will move from the reserve thread pool to the consumer pool of threads
      very likely. This is needed here to avoid that we decrease the concurrency
      level on the thread group. This is why we need to make the wake up
      unconditional here, it's more about moving the thread to the consumer
      pool more than about waking a thread up. If no thread exists it will be
      created by the stall check thread.
    */
    tp_wake_thread(my_tp_group, level);            // ★★ 无条件唤醒（重点在"迁池"不在"唤醒"）
    mysql_mutex_unlock(&my_tp_group->LOCK_group);
  }
}
```

**★ `thd_pool_wait_end()`**（`thread_pool.cc:3913`）：

```cpp
void thd_pool_wait_end(THD *thd) {
  ...
  if (client_cntx != nullptr) {
    if (my_thread_self() != my_thread_data->thread_handle.thread) return;
    // thd_wait_end() must always have been preceded by thd_wait_begin()
    assert(client_cntx->wait_type > THD_WAIT_NONE);

    if (my_thread_data->not_stalled) {
      my_thread_data->not_stalled = false;
      inc_active_threads(my_tp_group, my_thread_data);
      //  ★ 把借出去的 credit 收回来（可能瞬时超额，这是设计允许的）
    }
    client_cntx->wait_type = THD_WAIT_NONE;
  }
}
```

> ★★ **与"超时猜测"的分工**：引擎在即将阻塞（行锁、IO、binlog 组提交、fsync）前**主动**调用 `thd_wait_begin()` → 立即归还 credit + 迁池唤醒。stall check 线程**只是兜底**，专门对付那些"占着 CPU 不上报"的长查询（如大排序、大 JOIN）。两类机制互补，不是二选一。
>
> ⚠️ **begin/end 必须严格配对**：源码两处断言（`thread_pool.cc:3831` / `:3938`）——begin 后不能再 begin；end 必须有前置 begin。

**credit 计数的三个原语**（`thread_pool.cc:509`）：

```cpp
static inline void inc_active_threads(tp_group_t *tp_group,
                                      tp_thread_t *my_thread_data) {
  assert(!my_thread_data->contributes_to_threads_active);
  ++tp_group->threads_active;
  my_thread_data->contributes_to_threads_active = true;
}

static inline void dec_active_threads(tp_group_t *tp_group,
                                      tp_thread_t *my_thread_data) {
  assert(my_thread_data->contributes_to_threads_active);
  --tp_group->threads_active;
  my_thread_data->contributes_to_threads_active = false;
}

static inline std::uint32_t get_active_threads_relaxed(const tp_group_t *tp_group) {
  return tp_group->threads_active.load(std::memory_order_relaxed);
}
```

线程池要精确工作，必须知道"某个线程现在是否在等 IO / 等锁"。两种做法：

- **猜**：定时扫描，看查询跑了多久（stall 检测）——粗糙，60ms 级
- **问**：让引擎在每次阻塞前后**主动回调**（`THD_event_functions`）——精确，微秒级

MySQL 选了**两者都要**，且以回调为主：

> 官方："A statement becomes blocked if it encounters a disk I/O operation or a user level lock... **there are callbacks to the thread pool** to ensure that the thread pool can immediately start a new thread in this group."

**代价**：这套回调要求**内核与存储引擎深度协同**，这也是线程池必须做成"插件 + 内核回调"而非纯插件的原因。并且"未被插桩代码的阻塞"永远无法上报——官方明确列出三类只能靠 stall 兜底的场景（长查询、binlog dump 线程、未插桩的锁等待）

**源码落点**：回调的注册在 1.1（`THD_event_functions` 三个钩子），消费在 7 章（`thd_pool_wait_begin` / `wait_end`）。最能体现"以回调为主"的是 **`wait_begin` 里的无条件唤醒**：

```cpp
if (!my_thread_data->stalled) {
  my_thread_data->not_stalled = true;
  dec_active_threads(my_tp_group, my_thread_data);   // ★ 立刻释放 credit
}
if (my_tp_group->waiting_thread == my_thread_data) {
  my_tp_group->waiting_thread = nullptr;
  level = WAKE_OR_CREATE_ONE;                        // ★ 必须创建，否则没人收事件
}
tp_wake_thread(my_tp_group, level);
```

注意注释里那句"**唤醒的真正目的常常是把线程从冷池挪到热池**"——引擎一上报阻塞，线程池立刻补一个线程，这是"问"比"猜"快的根本原因（stall 检测要等 60ms，回调是即时的）。。

#### 决策三：CPU cache 优先于公平

**证据 ①：`tp_wake_thread()` 的 consumer / reserve 双池**（`thread_pool.cc:1391`）——这是"cache 优先"最直白的实现：

```cpp
/*
  Wake one of the worker threads that is waiting to process clients commands.
  Or create a new thread to help process client commands.

  @param my_tp_group    The thread pool group context to active a thread
  @param level          Ability to wake up one thread or create a new one

  @note
    This will:
      - Wake a consumer or reserve thread
      - Create a thread if there are no "idle" threads AND level is
        WAKE_OR_CREATE_ONE

    The purpose of consumer and reserve threads is a very simple manner of
    a queue. We don't need a full purpose queue, it's enough to keep track
    of which thread was last used. The purpose of this is to ensure that we
    always use the thread with most of its stack in the CPU cache. Given
    that the CPU cache changes very rapidly it makes no sense to keep track
    of more than the latest thread used. However to keep track of this one
    thread gives a significant boost of 5-10% in performance in experiments.
*/
static void tp_wake_thread(tp_group_t *my_tp_group, wake_level level) {
  mysql_mutex_assert_owner(&my_tp_group->LOCK_group);

  if (my_tp_group->threads_for_consumer > 0) {
    /* Get another thread from the consumer group */
    mysql_cond_signal(&my_tp_group->COND_consumer);      // ★①"热"线程：最近用过，cache 友好
  } else {
    if (level == WAKE_IF_CREATED || level == WAKE_OR_CREATE_ONE) {
      if (my_tp_group->threads_for_reserve > 0) {
        mysql_cond_signal(&my_tp_group->COND_reserve);   // ★②"冷"线程：已创建但休眠
      } else {
        if (level == WAKE_OR_CREATE_ONE) {
          if (my_tp_group->num_query_threads >
                  1 + my_tp_group->stats.connection_count ||   // ★ 线程数 ≤ 连接数 + 1
              my_tp_group->num_query_threads.load() > MAX_THREADS_PER_GROUP)
            return;
          if (my_tp_group->threads_active == 0 ||
              (tp_thread_create_control_ctx.total_query_threads <
               (3 * tp_thread_create_control_ctx.vcpu_count))) {  // ★ 3×vcpu 内不限速
            if (tp_create_worker_threads(my_tp_group, 1,
                    Worker_thread_type::QUERY_WORKER_THREAD) == 1) {
              my_tp_group->num_query_threads++;
              tp_thread_create_control_ctx.total_query_threads++;
              my_tp_group->last_thread_creation_time = my_micro_time();
            }
          } else {
            /* Rate limit thread creation. */
            ulonglong now = my_micro_time();
            if (now - my_tp_group->last_thread_creation_time >
                tp_thread_create_ctrl_interval()) {     // ★③超过 3×vcpu 后限速
              if (tp_create_worker_threads(my_tp_group, 1,
                      Worker_thread_type::QUERY_WORKER_THREAD) == 1) {
                my_tp_group->num_query_threads++;
                tp_thread_create_control_ctx.total_query_threads++;
                my_tp_group->last_thread_creation_time = my_micro_time();
              }
            }
          }
        }
      }
    }
  }
}
```

```cpp
// thread_pool.cc:1361 —— 建线程的三档限速
static inline ulonglong tp_thread_create_ctrl_interval() {
  uint64 total_threads_in_tp = tp_thread_create_control_ctx.total_query_threads;
  if (total_threads_in_tp < (3 * tp_thread_create_control_ctx.vcpu_count))
    return 0;                    // < 3×vcpu：不限速
  if (total_threads_in_tp < (5 * tp_thread_create_control_ctx.vcpu_count))
    return 50;                   // 3~5×vcpu：间隔 ≥ 50 微秒
  return 100;                    // ≥ 5×vcpu：间隔 ≥ 100 微秒
}
```

```cpp
// thread_pool.cc:173 —— 三档唤醒级别
/*
  When we wake threads we have 3 scenarios:
  WAKE_IF_CONSUMER:
    This is used when we don't want to use reserved threads and
    also don't want to start a new thread. We only want to use
    a consumer thread.
  WAKE_IF_CREATED:
    This is used when we can also accept that a reserved thread is
    started, but we don't accept a new thread to be started. We
    still prefer consumer thread if there is one.
  WAKE_OR_CREATE_ONE:
    This is used when we can also accept to start a new thread.
    This is only used from stall check thread to avoid that we create
    new threads too fast which could lead to too many threads started.
*/
enum wake_level {
  WAKE_IF_CONSUMER = 0,
  WAKE_IF_CREATED = 1,
  WAKE_OR_CREATE_ONE = 2
};
```

> ★★ **`WAKE_OR_CREATE_ONE` 只有 stall check 线程会用**（注释原文）——普通路径最多用到 `WAKE_IF_CREATED`。这是"防止建线程过快"的总闸门。

**证据 ②：热冷分层（hot/cold region）是字段布局，不是高低优先级队列**（`methods.h:556`）：

```cpp
struct tp_client_low_level_t {
#ifdef HAVE_EPOLL
  struct epoll_event ev;                 // ★ 热区：每次事件返回都要读
  my_socket epoll_fd;
#else
  /*
    The low level group object is heavily used in poll implementation
    and thus placed here in "hot" region of data structure.
  */
  tp_group_low_level_t *group;           // ★ poll 版：group 指针极频繁 → 热区
  uint index;
  uint fill_to_16_bytes;                 // ★ 显式填充到 16 字节
#endif
  /* Low level info is 16 bytes */
  my_socket fd;
  uint active_flag;
  uint ready_to_execute_flag;
  killed_state connection_killed_state;
  /* Next 16 bytes, total 32 bytes */
  tp_thread_t *processing_thread;
  THD *thd;
  Time_pt time_of_enqueueing;
  tp_client_low_level_t *next_client_query;
  tp_client_low_level_t **prev_client_query;
  /* Next 32 bytes, total 64 bytes */

  /* Uncommonly used variables */        // ★★ 冷区开始
  THD_wait_type wait_type = THD_WAIT_NONE;
  cleanup_level cleanup_state;
  tp_client_low_level_t *next_open_connection;
  tp_client_low_level_t **prev_open_connection;
#ifdef HAVE_EPOLL
  /*
    Low level group object very rarely used in epoll implementation and
    thus placed in "cold" region.
  */
  tp_group_low_level_t *group;           // ★ epoll 版：group 指针极少用 → 冷区
#endif
  ...
};
```

> ★★ **分层的判据是"访问频率"**：`poll` 实现里遍历 pollfd 每次都要 `client->group`，所以放热区；`epoll` 实现里 `ev` 才是热点，group 指针被踢到冷区。**同一个结构体，两种平台两套布局**，靠 `#ifdef HAVE_EPOLL` 切换。这就是"决策三"落到代码里的样子。

**证据 ③：`unsafe_atomic_add()` —— 用"良性竞争"换掉原子 RMW**（`thread_pool.h:286`）：

```cpp
/** Adds an increment to an atomic unsafely. That is, unlike for operator++ or
   fetch_add there is no guarantee that concurrent updates are not lost.
   Intended use-case is when the atomic is ONLY updated from one thread, but may
   be read from many threads (typically worker threads executing P_S queries).
   Without using an atomic this would be a so-called "benign" race. */
template <typename T>
void unsafe_atomic_add(std::atomic<T> *atm, T inc) {
  atm->store(atm->load(std::memory_order_relaxed) + inc,
             std::memory_order_relaxed);
}
```

**证据 ④：`alignas(128)` 隔离 stall check 的锁**（`thread_pool.cc:316`）：

```cpp
/* Ensure stall check mutexes have their own cache line */
alignas(128) static mysql_mutex_t tp_LOCK_stall_check;
alignas(128) static mysql_cond_t tp_COND_stall_check;
alignas(128) static tp_group_t tp_group_list[MAX_THREAD_GROUPS];
```

**证据 ⑤：主循环热点字段刻意挤在一起**（`methods.h:405`）：

```cpp
  /*
    These are heavily used together in the main loop, so good if they can
    fit in the same cache line. Align this to 64 byte cache line if
    possible.
  */
  std::uint32_t max_active_threads{0};
  std::uint32_t stalled_threads = 0;
  std::atomic<std::uint32_t> threads_active{0};
```

**证据 ⑥：直执路径（process_direct）也是 cache 优化**（`thread_pool.cc:3994`）：

```cpp
bool can_process_direct(tp_group_t *my_tp_group, int events_total,
                        bool allowed_direct) {
  if (events_total == 1 && !in_highly_concurrent_query_processing_mode() &&
      !thread_pool_dedicated_listeners && allowed_direct &&
      !is_query_available(my_tp_group) && is_credit_available(my_tp_group)) {
    return true;    // ★ 单事件 + 队列空 + 有 credit → 不入队，listener 自己直接执行
  }
  return false;
}
```

> ★ 省掉一次"入队 → 唤醒 → 出队 → 上下文切换"，代价是 listener 期间不再监听——所以注释里强调"只有单连接活跃时才是这种情形"。

整个代码里到处是 cache 相关的权衡，宁可让查询多排一会儿也要保住局部性：

- 单事件**直执行**（避免唤醒开销）
- 冷热线程分层（复用热栈，注释称值 **5-10%**）
- 每组最多 3 线程并行（避免 cache 污染）
- 唤醒过于激进会损失 **15%** 性能的血泪注释

**代价**：延迟不公平。低优先级查询在极端积压下可能等 222 秒（官方推演，见[优先级](#优先级与防饿死)）

**源码落点**：三条证据分散但都能定位——

| 体现 | 源码 |
|---|---|
| 冷热线程分层 | `become_consumer_thread` / `become_reserve_thread`（9.1）——分别睡在 `COND_consumer` / `COND_reserve` 上，唤醒只取热线程（`WAKE_IF_CONSUMER`，见 5 章） |
| 单事件直执行 | `can_process_direct` / `process_direct_query`（4.3）——跳过入队+唤醒，让刚干完活的线程接着干 |
| 连接亲和连读 | `tp_bind_to_current_worker_thread`（9.3）与 `tp_process_event` 的 do-while（6.3） |
| "别乱唤醒"的血泪教训 | 5 章那段注释：*"an earlier variant of this condition lost 15% in performance simply by waking threads up to frivolously"* |

这四处共同说明一件事：**26.7 愿意用延迟换局部性**——宁可让查询多排一会儿，也要让CPU cache 保持命中。222 秒的最坏延迟正是这个取舍的另一半。。

#### 决策四：锁粒度压到极致

**证据 ①：`epoll_wait()` 期间不持 `LOCK_group`**（`handle_waiting_thread()`，`thread_pool.cc:1571`）：

```cpp
  /*
    my_tp_group->waiting_thread is the "lock state" that protects
    my_tp_group->events_index_produce from having multiple threads
    accessing it. Thus we don't need to hold a pthread lock any longer.
    Unlock the group before I wait for new tp_events
  */
  mysql_mutex_unlock(&my_tp_group->LOCK_group);
  //  ★★★ 用 waiting_thread != nullptr 这个"角色标记"当锁用 —— 谁拿到谁就是 listener

  events_ready = tp_group_low_level_wait_for_events(
      my_tp_group->group_low_level_cntx, my_tp_group, my_thread_data,
      next_events, wait);

  mysql_mutex_assert_not_owner(&my_tp_group->LOCK_group);
  /* Lock the group so I can figure out what to do next */
  mysql_mutex_lock(&my_tp_group->LOCK_group);
```

> ★★ **这是整套设计里最精妙的一步**：阻塞式的 `epoll_wait()`（可能几秒）**完全在锁外**。组内互斥靠"`waiting_thread` 是否已被占用"这个原子语义保证——不需要任何 pthread 锁。注释原文称之为 "the lock state"。

**证据 ②：执行 SQL 期间也放锁**（`process_query_with_existing_lock()`，`thread_pool.cc:1631`）：

```cpp
  auto unassign_grd = create_scope_guard([&] {
    mysql_mutex_assert_owner(&my_tp_group->LOCK_group);
    unassign_thread_from_connection(my_tp_group, my_thread_data);
  });

  // Process the event
  do {
    // Unlock the group before I process an event
    mysql_mutex_unlock(&my_tp_group->LOCK_group);
    auto relock_grd =
        create_scope_guard([&] { mysql_mutex_lock(&my_tp_group->LOCK_group); });

    mysql_mutex_assert_not_owner(&my_tp_group->LOCK_group);
    if (unlikely(tp_process_event(my_tp_group, my_thread_data))) {
      assert(client_cntx == nullptr);
      return;
    }
    // The query on the connection must be re-tried as long as
    // set_connection_inactive_with_existing_lock() returns true.
  } while (
      set_connection_inactive_with_existing_lock(my_tp_group, my_thread_data));
```

> ★ `create_scope_guard` 是 MySQL 自研的 RAII（类似 `std::scope_exit`），**异常/提前 return 都能保证重新加锁与解绑**。整个线程池大量使用它替代手写 unlock。

**证据 ③：一把 `LOCK_group` 保护全组，但只有 3 把 mutex**（`methods.h:370`）：

```cpp
  /** Mutex protecting this group */
  mysql_mutex_t LOCK_group;                 // ★ 主锁：队列、连接链表、统计
  // Lock protecting connection context queue.
  mysql_mutex_t LOCK_connect;               // ★ 只保护"新连接请求队列"
  // Lock protecting query worker threads count during write.
  mysql_mutex_t LOCK_query_threads_count;   // ★ 只保护 query线程计数（PSI_NOT_INSTRUMENTED）
  /** Condition variable for start/stop of the threads */
  mysql_cond_t COND_group;
  /** Condition variable for consumer thread */
  mysql_cond_t COND_consumer;               // ★ "热"线程休眠
  /** Condition variable for reserve thread */
  mysql_cond_t COND_reserve;                // ★ "冷"线程休眠
  // Condition variable for connect handler thread.
  mysql_cond_t COND_connect;
```

> ★ **consumer 与 reserve 分开两个 cond** 是"cache 优先"的直接后果：需要线程时优先 signal `COND_consumer`，只有在 consumer 池空了才碰 `COND_reserve`。

**证据 ④：`I_P_List` 侵入式链表零分配**（`methods.h:296`）：

```cpp
/** Type of the list of open connections */
typedef I_P_List<tp_client_low_level_t, tp_client_low_level_open_conn_adapter_t>
    Open_connections_list;

/** Type of the list of active queries */
typedef I_P_List<tp_client_low_level_t, tp_client_low_level_query_adapter_t,
                 I_P_List_counter,                        // ★ 带元素计数
                 I_P_List_fast_push_back<tp_client_low_level_t>>   // ★ O(1) 尾插
    Query_list;

/**
  Helper class which specifies which members of tp_client_low_level_t are
  used for participation in the list of open connections for the group.
*/
struct tp_client_low_level_open_conn_adapter_t
    : public I_P_List_adapter<tp_client_low_level_t,
                              &tp_client_low_level_t::next_open_connection,
                              &tp_client_low_level_t::prev_open_connection> {};

struct tp_client_low_level_query_adapter_t
    : public I_P_List_adapter<tp_client_low_level_t,
                              &tp_client_low_level_t::next_client_query,
                              &tp_client_low_level_t::prev_client_query> {};
```

> ★★ **同一个连接对象用两组不同的 next/prev 指针同时挂在两个链表上**（组内连接链表 + 查询队列）。零额外内存分配、O(1) 摘除、`elements()` 直接给队列长度（PFS 的 `QUEUED_QUERIES` 列就读它）。

**证据 ⑤：`unsafe_atomic_add()` 放弃原子 RMW**（见"决策三"）——统计累加在"单写者多读者"场景下用 `load + store(relaxed)` 代替 `fetch_add`，省掉总线锁。

**证据 ⑥：thread_id 分配改成分区**（server 侧对照）——`Global_THD_manager` 用 8 分区 `LOCK_thd_list[]`，而线程池**根本不用它**：group 的连接分配走 `LOCK_connect`，且只在 push/pop 队列时短暂持有。

```cpp
// thread_pool.cc:3411 —— 新连接入队的加锁范围极小
  {
    mysql_mutex_lock(&my_tp_group->LOCK_connect);
    auto unlock_guard = create_scope_guard([&]() { mysql_mutex_unlock(&my_tp_group->LOCK_connect); });
    my_tp_group->connection_context_queue.push_back(connection_context);
    ++opt_option_tracker_usage_thread_pool_plugin;

    if (my_tp_group->num_connect_handler_thread_in_sleep > 0) {
      mysql_cond_signal(&my_tp_group->COND_connect);
      return false;
    }
  }  // End of scope holding LOCK_connect
```

**证据 ⑦：连接 round-robin 分配在 Windows 才用原子**（`thread_pool.cc:3058`）：

```cpp
static tp_group_t *assign_connection_to_thread_group() {
#ifdef _WIN32
  static std::atomic_int next_group_idx = -1;   // ★ Windows 多线程 accept，必须原子
#else
#ifndef NDEBUG
  // Verify that this function is only ever called by a single thread (the
  // thread listening to the socket).
  static auto tid = std::this_thread::get_id();
  assert(std::this_thread::get_id() == tid);
#endif /* !NDEBUG */
  // Index which incremented on each call. Since this function is only ever
  // called by one thread (see assert above), we can use an ordinary static
  // variable without synchronization.
  static int next_group_idx = -1;                // ★ POSIX：单线程 accept，普通变量即可
#endif /* _WIN32 */
  next_group_idx = (next_group_idx + 1) % tp_normal_groups;  // ★ 模"普通组数"，不含 admin 组
  return tp_group_list + next_group_idx;
}
```

> ★ admin 连接**绕过 round-robin** 直接进 admin 组（`thd_pool_add_connection()`，`thread_pool.cc:3411`）：
> ```cpp
>   tp_group_t *my_tp_group = (channel_info->is_admin_connection()
>                                  ? admin_thread_group
>                                  : assign_connection_to_thread_group());
> ```

- `waiting_thread` **角色标记替代互斥锁**保护事件索引（`epoll_wait` 期间完全不持 `LOCK_group`）
- 组内高频字段手工排进同一 cache line
- `threads_active` 等用原子而非互斥
- 单写多读场景用 `unsafe_atomic_add`（`thread_pool.h`，注释称之为 "benign race"）

**源码落点**：最典型的一处是 `handle_waiting_thread`（4.4）——**用"角色标记"替代互斥锁**：

```cpp
my_tp_group->waiting_thread = my_thread_data;      // 声明自己是 waiter
set_state(my_thread_data, Thread_state::POLLING);
/*
  my_tp_group->waiting_thread is the "lock state" that protects
  my_tp_group->events_index_produce from having multiple threads
  accessing it. Thus we don't need to hold a pthread lock any longer.
*/
mysql_mutex_unlock(&my_tp_group->LOCK_group);      // ★ 收事件期间完全不持锁

events_ready = tp_group_low_level_wait_for_events(...);
```

**"谁在当 waiter"本身就成了锁**：同一时刻只允许一个 waiter，所以它独占 `events_index_produce`，无需加锁。收益是 `epoll_wait` 期间（可能阻塞很久）不占用 `LOCK_group`——若持锁，全组所有线程都会被卡住。

其余三处的落点：高频字段手工排 cache line 见 2.1；`threads_active` 用原子见 9.4 的 `inc/dec_active_threads`；`unsafe_atomic_add` 见 `thread_pool.h`。

### 理论溯源

| 理论 / 经典技术 | 源码落点 |
|---|---|
| **线程池 / 事件驱动（Reactor）** | `tp_group_low_level_wait_for_events`（`epoll.cc`）—— 一组一个 epoll 实例 + 收事件线程 |
| **self-pipe trick**（Schwartz, 1990s） | `tp_group_low_level_waiter_wake`（`epoll.cc`）—— 往 socketpair 写 1 字节打断 `epoll_wait` |
| **`EPOLLONESHOT` 串行化** | `handle_epoll_ctl`（`epoll.cc`）—— fd 触发一次后自动摘除，天然保证单线程处理单连接 |
| **多级反馈队列 / 优先级老化（aging）** | `check_trans_queue_for_prio_kickups`（`thread_pool.cc`）—— 低优等待超时后升级 |
| **滞回控制（hysteresis）** | `check_longrunning_transactions_and_adjust_MTL`（`thread_pool.cc`）—— 挂起/恢复双阈值防抖动 |
| **LIFO 唤醒保 cache**（热线程复用） | MySQL 侧是 consumer/reserve 双条件变量（`tp_wake_thread`, 1391）；MariaDB 侧是 `push_front` |
| **Bridge + Facade（插件 ABI）** | `Plugin_connection_handler` + `Connection_handler_functions`（见[一、插件接入](#一插件接入与生命周期)） |

### 算法与数据结构

| 结构 / 算法 | 用途 | 复杂度 |
|---|---|---|
| `I_P_List`（侵入式双向链表） | 连接同时挂在 open_connections / queued_queries / queued_trans 三条链上，用 adapter 区分指针 | O(1) 插入删除 |
| 固定数组 `group_threads[4096]` | 线程槽位，stall 检查时线性遍历 | O(组数×线程数) |
| `std::chrono` 强类型时间（`Musec`/`Misec`/`Cesec`/`Time_pt`） | 避免时间单位错误——`stall_limit` 是厘秒、`prio_kickup_timer` 是毫秒，用类型系统区分 | — |
| 双队列 + 老化 | 高优优先，低优等待超时升级 | O(1) |
| epoll + `EPOLLONESHOT` | 事件采集，天然串行化单连接 | O(就绪数) |

### 他库对比与演进动机

| | MySQL 26.7 | MariaDB 12.3 | Percona 8.0 | PostgreSQL |
|---|---|---|---|---|
| 形态 | **插件** | 内建 | 内建 | 无内置线程池（靠连接池 pgbouncer）|
| 并发度量 | **事务数（MTL）** | 活跃线程数 | 活跃+等待线程数 | — |
| stall 判定 | **单查询时长** | 队列排空速率 | 队列排空速率 | — |

**演进动机**：MySQL 线程池是**独立起源**（5.5 与 MariaDB 同期），与 MariaDB 主线并行演化。这解释了为什么它内部结构（双队列命名反置、per-thread stall、冷热分层）与 MariaDB 差异巨大，却又有 `thread_pool_size`、`thread_pool_prio_kickup_timer` 等同名参数——趋同演化 + 有意对齐。

**为什么 2026 年才下放社区**：线程池长期是企业版的核心差异化卖点。26.7 下放的同时，Oracle 用 **MTL + 长事务背压**（社区版独有、企业版旧版没有）重新拉开差距——不是简单搬运，而是借社区化做了一次机制升级。

> 详细对比见 [`thread_pool_mariadb_percona.md`](thread_pool_mariadb_percona.md)。

---

## 核心实现

### 主链路

```
新连接 ──► thd_pool_add_connection                        [插件入口]
             └─ assign_connection_to_thread_group()       round-robin 选组
                  └─ connection_context_queue.push()      入"待认证"队列
                       └─ [CONNECTION_HANDLER 线程] create_thd_and_authenticate_conn()
                            ├─ create_thd() + thd_prepare_connection()  建 THD + 认证
                            ├─ (若 admin 权限) 重分配到 admin 线程组
                            └─ tp_client_low_level_arm()                注册进 epoll(ONESHOT)
                                 │
查询到达 ──► [waiting thread] tp_group_low_level_wait_for_events()  epoll_wait(≤32)
             ├─ 单事件且允许 ─► process_direct_query()   直执行，不入队
             └─ 否则 ─► queue_query()                    入 高优/低优 队列
                          │
        ┌─────────────────┘
        ▼
[QUERY 线程] high_concurrency_algorithm() / low_concurrency_algorithm()
        └─ is_query_ready_to_process() = 有排队 && is_credit_available()
             └─ process_queued_query() → pop_highest_priority_query()
                  └─ tp_process_event()                  ★ 执行主体
                       ├─ trxn_threads++（RAII guard）
                       ├─ thread_attach()                切换 PSI + thread_stack
                       ├─ do { tp_execute_command() }    可能连读多条（SSL 缓存）
                       │     └─ do_command(thd)
                       ├─ thread_detach()                恢复 PSI + 内存屏障
                       └─ tp_client_low_level_rearm()    重新武装 → ARMED
```

**并发旁路**（与主链路并行）：

```
[stall check 线程] 每 1ms/10ms 遍历所有组
   ├─ mark_and_count_stalled_threads()   判定 stalled（>60ms）
   ├─ update_max_active_threads()        max = user_request + stalled ★
   ├─ check_longrunning_..._MTL()        长事务背压
   └─ tp_wake_thread()                   需要时唤醒/创建

[引擎阻塞时] thd_wait_begin ─► thd_pool_wait_begin()
   ├─ dec_active_threads()               释放 credit
   └─ tp_wake_thread()                   无条件（含冷→热迁移意图）
```

### 一、插件接入与生命周期

#### 1.1 两套回调

线程池能做成插件而不动内核，是因为内核把两件事抽象成了回调（`thread_pool_plugin.cc`）：

```cpp
static THD_event_functions thread_pool_event_functions = {
    thd_pool_wait_begin, thd_pool_wait_end, thd_pool_post_kill_notification};

static Connection_handler_functions thread_pool_handler_functions = {
    0,  // max_threads，init 时填
    thd_pool_add_connection, thd_pool_end};
```

| 回调集合 | 职责 | 内核侧使用者 |
|---|---|---|
| `Connection_handler_functions` | 接管连接的建立（`add_connection`）与销毁（`end`） | `Plugin_connection_handler` → `Connection_handler` 子类 |
| `THD_event_functions` | 引擎阻塞前/后上报（`wait_begin`/`wait_end`）、KILL 通知 | 执行路径中的 `thd_wait_begin`/`thd_wait_end` |

注册（`thread_pool_plugin.cc`）：

```cpp
thread_pool_handler_functions.max_threads =
    static_cast<uint>(get_max_connections() + MAX_NORMAL_THREAD_GROUPS);
if (my_connection_handler_set(&thread_pool_handler_functions,
                              &thread_pool_event_functions)) {
  LogErr(ERROR_LEVEL, ER_THREAD_POOL_CON_HANDLER_INIT_FAILED);
  return TP_INIT_FAILURE;
}
```

内核侧包装（`sql/conn_handler/plugin_connection_handler.h`，8.0 树里就有）：

```cpp
class Plugin_connection_handler : public Connection_handler {
  Connection_handler_functions *m_functions;
 public:
  ~Plugin_connection_handler() override { m_functions->end(); }
 protected:
  bool add_connection(Channel_info *channel_info) override {
    return m_functions->add_connection(channel_info);
  }
  uint get_max_threads() const override { return m_functions->max_threads; }
};
```

#### 1.2 这就是 ABI 隔离的教科书案例

- 内核只看到**函数指针** + 不透明的 `Channel_info*`
- 插件通过 `include/mysql/thread_pool_priv.h`（`MYSQL_THD` 对 C 是 `void*`）反向调用内核
- **结果**：线程池能作为独立 `.so` 在企业版↔社区版之间搬移，内核零改动

> 这正是"不透明指针 + C 风格函数接口"这套 ABI 隔离模式的实际应用——参考 [`../../infra/threading_architecture.md`](threading_architecture.md) 与 plugin 系列的讨论。

#### 1.3 为什么切换需要重启

插件可 `INSTALL PLUGIN` 动态加载，但 `thread_handling` 决定内核用哪个 `Connection_handler`，是**启动期绑定**的。

> 官方："If the server loads the thread pool plugin successfully, it sets the `thread_handling` system variable to **`loaded-dynamically`**."
>
> （对比：TXSQL 实现了动态切换 `thread_handling_switch_mode`，代价是初始化阶段就把所有连接管理方法一并初始化。见对比篇。）

#### 1.4 动态权限：插件如何消费组件服务

线程池是**插件**，但它需要的"动态权限"能力属于**组件服务**。这个跨越在源码里发生得很直接：

```cpp
const char *DYN_PRIV_REG_SVCNAM = "dynamic_privilege_register.mysql_server";
using Dyn_priv_reg = const s_mysql_dynamic_privilege_register;
...
  assert(reg_srv != nullptr);
  {
    my_service<Dyn_priv_reg> const service(DYN_PRIV_REG_SVCNAM, reg_srv);
    if (!service.is_valid() ||
        service->register_privilege(&TP_CONNECTION_ADMIN.front(),
                                    TP_CONNECTION_ADMIN.length())) {
      LogErr(ERROR_LEVEL, ER_THREAD_POOL_CANNOT_REGISTER_DYNAMIC_PRIVILEGE,
             &TP_CONNECTION_ADMIN.front());
      return TP_INIT_FAILURE;
    }
  }
```

三个要点：

1. **`my_service<Dyn_priv_reg>` 是插件侧的 RAII 服务句柄**——构造时 acquire、析构时 release。插件因此能消费组件服务，这就是"插件 ↔ 组件"之间的**桥**（见 [`../../server/plugin/architecture.md`](../../server/plugin/architecture.md) 的"服务做货币"）。
2. **注册的是动态权限 `TP_CONNECTION_ADMIN`**——用于标记 admin 连接，配合 3.3 的 Admin 线程组：持有该权限的连接会被分配到 admin 组。
3. **注册失败即插件加载失败**（`return TP_INIT_FAILURE`）——动态权限是线程池的**硬依赖**，不是可选增强。

deinit 时对称注销：

```cpp
static int thread_pool_plugin_deinit(void *) {
  DBUG_TRACE;
  if (reg_srv != nullptr) {
    my_service<Dyn_priv_reg> const service(DYN_PRIV_REG_SVCNAM, reg_srv);
    if (service.is_valid()) {
#ifndef NDEBUG
      auto r =
#endif /* NDEBUG */
          service->unregister_privilege(&TP_CONNECTION_ADMIN.front(),
                                        TP_CONNECTION_ADMIN.length());
      assert(!r);
    }
    if (reg_reg) {
      thread_pool_plugin_option_usage_deinit(reg_srv, reg_reg);
    }
```

注意 `unregister_privilege` 的返回值被 `#ifndef NDEBUG` 包裹——**注销失败在 release 构建下是静默的**，只有 debug 会 assert。

同一 init 流程里还获取/注册了另外两项：

| 服务 | 用途 |
|---|---|
| `registry_registration`（`reg_reg = svc.untie()`） | 拿到"注册"能力本身，供下面的 option usage 使用 |
| `thread_pool_plugin_option_usage_init(reg_srv, reg_reg)` | 注册**系统变量使用说明**，让服务器知道线程池用了哪些变量 |

> 这三条合起来是"插件消费组件服务"的完整样本：插件拿 registry → 通过它 acquire 动态权限服务与注册服务 → 注册自己的权限与变量使用声明。
> 它与 [`thd.md`](thd.md) 讲的"插件只看到 C 函数"是**同一套体系的两个方向**：**向内**靠 ABI 隔离保稳定，**向外**靠服务化拿能力。

### 二、线程组与线程角色

#### 2.1 tp_group_t（methods.h）

```cpp
struct tp_group_t {
  tp_group_low_level_t *group_low_level_cntx;   // epoll/poll 低层上下文
  tp_thread_t *waiting_thread;                  // 当前收事件的线程（listener 角色）
  Open_connections_list open_connections;
  std::atomic<uint> num_query_threads;
  connection_context_queue_t connection_context_queue;  // 新连接待认证队列
  Query_list queued_queries;   // ★ 高优先级队列
  Query_list queued_trans;     // ★ 低优先级队列
  mysql_mutex_t LOCK_group;
  mysql_cond_t COND_group, COND_consumer, COND_reserve, COND_connect;
  tp_group_statistics_t stats;
  tp_thread_t group_threads[MAX_THREADS_PER_GROUP];     // 4096
  // ↓ 高频字段，注释要求尽量挤进同一 cache line
  std::uint32_t max_active_threads;                     // 动态
  std::uint32_t stalled_threads;
  std::atomic<std::uint32_t> threads_active;
  std::atomic<std::uint32_t> trxn_threads;              // 执行事务的线程数
};
```

**三条队列**（命名容易搞反）：

| 字段 | 实际语义 |
|---|---|
| `queued_queries` | **高优先级**队列 |
| `queued_trans` | **低优先级**队列 |
| `connection_context_queue` | 新连接（待建 THD + 认证）队列 |

判据（`queue_query`，`thread_pool.cc`）：

```cpp
if (thd_is_transaction_active(client_cntx->thd) ||
    thread_pool_prio_kickup_timer == 0 ||
    is_high_priority_connection(client_cntx->thd)) {
  my_tp_group->queued_queries.push_back(client_cntx);   // 高优
} else {
  my_tp_group->queued_trans.push_back(client_cntx);     // 低优
}
client_cntx->conn_state = Connection_state::QUEUED;
```

`is_high_prio_query_available()`返回 `!queued_queries.is_empty()`。

> ⚠️ **命名反直觉**：`queued_trans` 装的恰恰是"没在事务里"的普通查询。读源码时必须记住这一点。

#### 2.2 tp_client_low_level_t 的热冷分层（methods.h）

```cpp
struct tp_client_low_level_t {
#ifdef HAVE_EPOLL
  struct epoll_event ev;   // epoll 版：事件结构放最热位置（16 字节）
  my_socket epoll_fd;
#else
  tp_group_low_level_t *group;  // poll 版：group 是热字段，放最前
  uint index;  uint fill_to_16_bytes;
#endif
  my_socket fd;  uint active_flag;  uint ready_to_execute_flag; ...
  /* Next 16 bytes, total 32 bytes */
  tp_thread_t *processing_thread;  THD *thd;  Time_pt time_of_enqueueing;
  /* Next 32 bytes, total 64 bytes —— 以下是不常用字段 */
  ...
#ifdef HAVE_EPOLL
  tp_group_low_level_t *group;   // ★ epoll 版 group 几乎不用 → 放"冷区"
#endif
  Connection_state conn_state;
  std::atomic_uint64_t direct_events, queued_events;
  std::atomic<Musec> management_time;   // 事件到达 → 开始执行的耗时
};
```

**同一个 `group` 指针在两个实现里被放在相反位置**——epoll 版放冷区、poll 版放热区，因为两者热路径不同。

#### 2.3 四种线程角色（methods.h）

⚠️ **勘误**：
- **不存在 `TP_LISTENER` / `TP_WORKER` / `TP_TIMER` 常量**——真名是 **`enum class Worker_thread_type`**
- **不存在 `tp_connection_state` / `TP_STATE_IDLE` 等**——连接状态机真名是 **`enum class Connection_state`**
- **不存在 `struct thread_group_t` / `struct connection_t`**——真名是 `tp_group_t` / `tp_client_low_level_t`
- **不存在 `tp_metrics`**——统计结构是 `tp_group_statistics_t`

```cpp
// methods.h:64 —— 四种角色
// Role of worker thread.
enum class Worker_thread_type {
  CONNECTION_HANDLER_WORKER_THREAD,   // ★① 建连线程：从 connection_context_queue 取新连接，create_thd + 认证
  LISTENER_WORKER_THREAD,             // ★② 监听线程：阻塞在 epoll_wait()/poll()，组内同时最多 1 个
  QUERY_WORKER_THREAD,                // ★③ 查询线程：真正 do_command() 的线程
  TIMER_WORKER_THREAD                 // ★④ 定时线程：全局唯一，即 stall checker
};
```

> ★ **`CONNECTION_HANDLER_WORKER_THREAD` 是 8.0 新增的角色**——老文档里只有 listener / worker / timer 三种。它把"建 THD + 认证"从查询线程里剥离出来，避免慢认证（如 LDAP、SSL 握手）阻塞查询。
>
> ★ **TIMER 角色全局只有一个实例**（`thread_pool.cc:290`），不属于任何线程组（`GROUP_ID_GARBAGE`）：
> ```cpp
> static tp_thread_t tp_stall_check_thread = {
>     my_thread_handle{},
>     0,
>     GROUP_ID_GARBAGE,              // ★ 不属于任何 group
>     {}, false, false, nullptr, nullptr, false,
>     Worker_thread_type::TIMER_WORKER_THREAD,
>     0ULL,
>     Thread_state::SC_CHECKING,
>     false, 0, {}, 0, {}};
> ```

**线程状态机（12 态，三组分列）**（`methods.h:87`）：

```cpp
/** States for threads represented by tp_thread_t. This includes
   the stall checker thread. */
enum class Thread_state {
  // Query worker thread states

  // In main loop, not hanging in epoll wait or executing query
  // (difficult to observe)
  MANAGING,                    // ★ 主循环里做管理工作（很难观测到）

  // Hanging in epoll_wait() waiting for file descriptors to become readable
  POLLING,                     // ★ 即 listener 角色

  // Processing a single connection with a readable fd without queueing it
  PROCESSING_DIRECT,           // ★ 直执：不排队就地处理（对应 direct_events 计数）

  // Processing connection from queue. Either as the waiting thread because
  // multiple file descriptors became readable in the same epoll_wait(), or
  // as an extra worker woken up to process more queries when using
  // high_concurrency_algorithm
  PROCESSING_QUEUED,           // ★ 从队列取（对应 queued_events 计数）

  // Thread is waiting on COND_consumer as "hot" thread (first choice if
  // addtional threads are needed).
  SLEEPING_CONSUMER,           // ★ "热"线程（threads_for_consumer）

  // Thread is waiting on COND_reserve as "cold" thread.
  SLEEPING_RESERVE,            // ★ "冷"线程（threads_for_reserve）

  // Connection handler states

  // Connection handler is processing a connection request
  CH_PROCESSING,

  // Connection handler is in a timed sleep. This is used by
  // additional connection handler threads created if the
  // existing ones are busy when the connection request
  // arrives. If the timed sleep expires without another
  // connection request arriving, the extra connection
  // handler thread terminates.
  CH_SLEEPING_TIMED,           // ★ 临时扩容的建连线程，60s 超时退出

  // Connection handler is in an indefinite sleep. This is
  // used by the last connection handler remaining when there
  // are no more connection requests to process.
  CH_SLEEPING_INDEFINITE,      // ★ 最后一个常驻建连线程，不退出

  // Stall checker states

  // Stall checker is running and doing its various checks.
  SC_CHECKING,

  // Stall checker is in a short (1 ms) sleep. This happens
  // when its last run triggered an action, e.g. a thread wakeup.
  SC_SLEEPING_SHORT,           // ★ 有动作 → 只睡 1ms 快速跟进

  // Stall checker is in a long (10 ms) sleep. This happens
  // when its last run did not trigger any actions.
  SC_SLEEPING_LONG             // ★ 无动作 → 睡 10ms 省开销
};
```

**连接状态机（7 态）**（`methods.h:510`）：

```cpp
/** States for connections in the thread pool, epresented by
    tp_client_low_level_t */
enum class Connection_state {
  // Connection and its THD has been created, but its fd is
  // not being monitored for incoming queries.
  ESTABLISHED,                 // ★ THD 已建好（认证完成），fd 还没挂进 epoll

  // The connection's file descriptor has been added to the
  // set which will be monitored by the next epoll_wait() call.
  ARMED,                       // ★ fd 已 arm

  // Event (query) was detected on connection and it queued
  // for later execution
  QUEUED,                      // ★ 事件到达，按优先级进 queued_queries / queued_trans

  // A worker considered the query for execution but found that
  // configured limits prevented it (see function is_credit_available)
  WAITING_FOR_CREDIT,          // ★★ 等 credit —— P_S 里能直接看到这个状态

  // Connection is attached to a worker thread which will process
  // its event(query)
  ATTACHED,                    // ★ 已绑到 worker，正在 do_command()

  // A kill request was recieved for the connection.
  KILLED,

  // The connection had been inactive longer than the configured limit.
  EXPIRED                      // ★ 空闲超时，由 stall checker 清理
};

//  ★ 源码用宏保证枚举名与字符串永远同步
#define CONNECTION_STATE_CASE__(ev) \
  case Connection_state::ev:        \
    return #ev;
```

**配套的两个小枚举**（`methods.h:469` / `:489`）：

```cpp
enum cleanup_level {
  TP_CLEANUP_IDLE = 0,
  TP_CLEANUP_STARTED = 1,      // ★ 防止并发重复清理
  TP_CLEANUP_COMPLETED = 2
};

enum killed_state {
  TP_CONNECTION_NOT_KILLED = 0,
  TP_CONNECTION_KILLED_WAITING_FLAG = 1,   // ★ 已发起 kill，等 KILL_FLAG 送达
  TP_CONNECTION_KILLED_RECEIVED_FLAG = 2   // ★ 已收到（KILL_FLAG = 1）
};

enum class Connection_type { User, Admin_interface, Admin_privilege };
//  ★ 普通用户连接 / 走管理端口 / 有管理特权（后两者进 admin thread group）
```

**`tp_thread_t` 的关键字段**（`methods.h:184`）：

```cpp
struct tp_thread_t {
  my_thread_handle thread_handle;
  /* numeric_id is a strictly increasing number assigned to each new thread.
     It's only used for the information schema ... */
  uint numeric_id{0};
  uint thread_tp_group{0};

  // Set when thread is attached to a connection for processing a query. ...
  //
  // The thread is deemed as stalled when thread_pool_stall_limit (=6 => 60ms
  // by default) has passed since the thread was attached.
  //
  // When a thread is deemed as stalled it is no longer active (but
  // tp_client_low_level_t::active_flag will still be true) and the
  // max_active_threads can be incremented to give space for another active
  // thread. This means that only short queries will stop other queries from
  // starting, a long-running query will quickly be deemed as stalled and
  // thereafter the thread pool will more or less treat it as a separate thread
  // until the query completes.
  Time_pt time_of_attach;      // ★★ stall 判定的唯一依据

  bool not_stalled{false};     // ★ 是否需要把 credit 收回来
  bool stalled = false;        // ★ 避免重复计数
  char *stack_start{nullptr};
  tp_client_low_level_t *client_low_level_cntx{nullptr};
  bool is_connection_handler_thread{false};
  Worker_thread_type worker_type;    // ★ 四种角色
  ulonglong pfs_thread_id{0};
  Thread_state state;                // ★ 12 态
  bool contributes_to_threads_active = false;

  std::atomic_uint64_t event_count = 0;      // EVENT_COUNT
  std::atomic<Musec> acc_event_time;         // ACCUMULATED_EVENT_TIME
  std::atomic_uint64_t command_count = 0;    // EXEC_COUNT
  std::atomic<Musec> acc_command_time;       // ACCUMULATED_EXEC_TIME
};
```

> ⚠️ **`methods.h` 里没有 `class ThreadPool`、没有 `virtual`**——它 775 行全是 C 风格 free-function + 纯数据结构。真正的"接口"是 `tp_group_low_level_*`（操作 epoll/poll 实例）与 `tp_client_low_level_*`（操作连接 fd）两组自由函数（`methods.h:662-775`）。
>
> ★ **队列不是 `STAILQ` 也不是 `std::list`**，是 MySQL 自研的头文件式侵入式链表 **`I_P_List`**（`sql/sql_plist.h`）：
> ```cpp
> typedef I_P_List<tp_client_low_level_t, tp_client_low_level_query_adapter_t,
>                  I_P_List_counter,                          // ★ 带元素计数
>                  I_P_List_fast_push_back<tp_client_low_level_t>>   // ★ O(1) 尾插
>     Query_list;
> ```
> 同一个连接对象用**两组不同的 next/prev 指针**同时挂在"组内连接链表"和"查询队列"上——零额外分配、O(1) 摘除。

| 角色 | 职责 | 数量 |
|---|---|---|
| `CONNECTION_HANDLER_WORKER_THREAD` | 建 THD + 认证（**不执行查询**） | 每组 `INITIAL_CONNECT_THREADS_PER_GROUP` |
| `LISTENER_WORKER_THREAD` | 纯收事件（仅 `dedicated_listeners=ON`） | 每组 0 或 1 |
| `QUERY_WORKER_THREAD` | 执行查询 | 每组 `query_threads_per_group`（默认 2） |
| `TIMER_WORKER_THREAD` | stall 检查（**全局 1 个**） | 1 |

> 官方："When the thread pool plugin starts, it creates **one thread per group (the listener thread), plus the background thread**."（stall checker 在官方表述中是 "a background thread that regularly monitors thread group states"）

启动布局（`thd_pool_init`，3246-3360）：

```cpp
tp_normal_groups = thread_pool_size;                     // 默认 16
tp_groups = tp_normal_groups + MAX_ADMIN_THREAD_GROUPS;  // +1
admin_thread_group = tp_group_list + tp_normal_groups;   // 最后一组是 admin 组
cur_group->threads_user_request = thread_pool_query_threads_per_group;  // 2
cur_group->max_active_threads   = thread_pool_query_threads_per_group;
```

默认约 **16 组 × 2 = 32 个查询线程**。

#### 2.4 状态机

**Thread_state**（methods.h）：

```
QUERY_WORKER:  MANAGING ──► POLLING ──► PROCESSING_DIRECT / PROCESSING_QUEUED
                  ▲                            │
                  └────────────────────────────┘
               空闲时: SLEEPING_CONSUMER(热) / SLEEPING_RESERVE(冷)

CONNECTION_HANDLER: CH_PROCESSING ──► CH_SLEEPING_TIMED ──► CH_SLEEPING_INDEFINITE

TIMER(stall checker): SC_CHECKING ──► SC_SLEEPING_SHORT(1ms) / SC_SLEEPING_LONG(10ms)
```

**Connection_state**（methods.h）：

```
ESTABLISHED ──arm──► ARMED ──事件──► QUEUED ──取到──► ATTACHED ──rearm──► ARMED
                                       │
                                       └─额度不足─► WAITING_FOR_CREDIT
   另有 KILLED / EXPIRED 终态
```

### 三、连接生命周期

#### 3.1 分配（round-robin）

```cpp
static tp_group_t *assign_connection_to_thread_group() {
  next_group_idx = (next_group_idx + 1) % tp_normal_groups;
  return tp_group_list + next_group_idx;
}
```

注释明确这是"**初步分配**"：权限要建完 THD 才知道，若发现是 `TP_CONNECTION_ADMIN` 或 admin interface，会在 `create_thd_and_authenticate_conn` 里**重分配到 admin 线程组**。

> 官方："As connections are established, the thread pool assigns them to thread groups **in round-robin fashion**."（三家都不考虑负载，是 TXSQL 做负载均衡优化的动因）

#### 3.2 建 THD 与认证：PSI 双插桩（）

这是线程池区别于 Per_thread 模型的关键设计：

```cpp
/*
  Thread pool uses separate PFS instrumentations to run thread pool code and
  the user session. User session runs under the 'user job instrumentation'
  ...
  Thread pool code always runs under the 'thread pool worker instrumentation'.
*/
THR_PSI_backup = PSI_THREAD_CALL(get_thread)();              // 保存连接处理线程插桩
psi = PSI_THREAD_CALL(new_thread)(key_tp_one_connection, 0, nullptr, 0);
PSI_THREAD_CALL(set_thread)(psi);                            // 切到用户会话插桩

THD *thd = create_thd(channel_info);
thd_init(thd, (char *)&client_cntx);
PSI_THREAD_CALL(set_thread_THD(psi, thd));
thd->set_psi(psi);
prepare_rc = thd_prepare_connection(thd);                    // 认证
```

**为什么必须双插桩**：线程池里一个 OS 线程会先后服务**多个不同连接**。复用同一插桩会把不同会话的等待事件、内存分配混在一起统计，且会话结束时无法正确回收。解法是**每个连接一套独立插桩**，随连接生命周期创建/销毁。

> 官方规则："While a thread executes a statement from a user connection, Performance Schema instrumentation accounts thread activity to **the user connection**. Otherwise, ... to **the thread pool**."

#### 3.3 Admin 线程组：应急通道

`MAX_ADMIN_THREAD_GROUPS = 1`。拥有 `TP_CONNECTION_ADMIN` 权限的连接被路由到这里，即使所有普通组堵死仍能连进来执行 `KILL`。

> 官方："must be granted explicitly. **It is not granted to any user by default.**"
> 特权连接还**不受 `thread_pool_transaction_delay` 限制**。
> `tp_thread_group_stats` 返回 17 行时（16 组 + 1），**Admin 组统计在最后一行**。

> 对比：MariaDB 用 `extra_port`（池外独立端口），Percona 8.0.14+ 用 `admin_port`。三种不同路线。

### 四、事件采集层（epoll）

#### 4.1 epoll fd + socketpair 通知管道

```cpp
bool tp_group_low_level_waiter_wake(tp_group_low_level_t *group_cntx, char flag) {
  /* Write a byte to wake up the waiter thread */
  res = send(group_cntx->notify_fd, &flag, 1, 0);       // epoll.cc
  return res == 1 ? false : true;
}
```

经典的 **self-pipe trick**：KILL 查询、关闭服务器时往 socketpair 写 1 字节，`epoll_wait` 立即返回。

#### 4.2 EPOLLONESHOT：天然的 per-connection 串行化

```cpp
static bool handle_epoll_ctl(tp_client_low_level_t *client_cntx,
                             tp_group_t *my_tp_group, int op, bool arm) {
  if (op == EPOLL_CTL_MOD) {
    mysql_mutex_lock(&my_tp_group->LOCK_group);
    if (client_cntx->connection_killed_state == TP_CONNECTION_NOT_KILLED) {
      client_cntx->ev.events = EPOLLIN | EPOLLONESHOT;      // epoll.cc
      if (epoll_ctl(client_cntx->epoll_fd, EPOLL_CTL_MOD, client_cntx->fd,
                    &(client_cntx->ev)) < 0) flag = true;
    }
    mysql_mutex_unlock(&my_tp_group->LOCK_group);
  } else {
    client_cntx->ev.events = arm ? EPOLLIN | EPOLLONESHOT : 0;
    ...
  }
}
```

**`EPOLLONESHOT` 是核心**：fd 触发一次后自动从 epoll 集合移除，必须显式 `rearm` 才能再次监听。这天然保证了一个连接任意时刻只被一个工作线程处理——**不需要额外的 per-connection 锁**。

rearm 失败处理很硬核：普通连接 EBADF（socket 已关）可忽略，但**通知 socket 失败会导致无法 KILL、无法优雅关闭** → 重试 30 秒后 `abort()`。

#### 4.3 事件主循环与批处理

```cpp
int timeout = wait ? -1 : 0;          // -1 = 无限等待, 0 = 不等待
int loop    = wait ? 1000 : 1;
events_total = epoll_wait(group_cntx->fd, event_list, MAX_EVENTS_PER_WAIT_CALL, timeout);

if (can_process_direct(my_tp_group, events_total, wait)) {
  my_thread_data->client_low_level_cntx =
      pointer_cast<tp_client_low_level_t *>(event_list[0].data.ptr);
  handle_single_event(group_cntx, my_thread_data, my_tp_group);   // 直执行
} else {
  /*
    When highly concurrent query processing mode is used, quickly check if
    more events are available. Events are concurrently processed by the
    multiple query worker threads.
  */
  if (events_total == 1 && in_highly_concurrent_query_processing_mode()) {
    int new_events = epoll_wait(group_cntx->fd, event_list + 1,
                                MAX_EVENTS_PER_WAIT_CALL - 1, 0);  // 非阻塞再捞
    if (new_events > 0) events_total += new_events;
  }
  return handle_multiple_events(group_cntx, next_events, events_total, &event_list[0]);
}
```

**两处优化**：
1. **单事件直执行**：跳过"入队 → 唤醒 worker → 出队"的完整开销，针对单连接持续打查询的场景
2. **高并发模式二次捞取**：只捞到 1 个事件时立即非阻塞再捞一次，批量收上来提高多 worker 并行度

> 官方对应表述："if the statement is **the only one received, and there are no statements queued or currently executing**" → 立即执行，且 **"If immediate execution occurs, the listener thread performs it. (This means that temporarily no thread in the group is listening.)"**

#### 4.4 handle_waiting_thread：角色标记当锁用

```cpp
my_tp_group->waiting_thread = my_thread_data;              // 声明自己是 waiter
set_state(my_thread_data, Thread_state::POLLING);
/*
  my_tp_group->waiting_thread is the "lock state" that protects
  my_tp_group->events_index_produce from having multiple threads
  accessing it. Thus we don't need to hold a pthread lock any longer.
*/
mysql_mutex_unlock(&my_tp_group->LOCK_group);              // ★ 收事件期间不持锁

events_ready = tp_group_low_level_wait_for_events(...);

mysql_mutex_lock(&my_tp_group->LOCK_group);
if (my_tp_group->waiting_thread == my_thread_data)
  my_tp_group->waiting_thread = nullptr;
for (i = 0; i < events_ready; i++) queue_query(my_tp_group, next_events[i]);
```

**`waiting_thread != nullptr` 这个"角色标记"本身充当互斥语义**——所有组和连接共抢一把 `LOCK_group` 是不可接受的。

### 五、调度与执行

#### 5.1 两种调度算法

**LOW_CONCURRENCY（0）**—— 无独立 listener，同线程既收事件又执行：

```cpp
/*
  This means that in a high load situation we have one thread
  taking care of both getting events and executing queries to
  ensure we have optimal CPU cache usage.
*/
```

**HIGH_CONCURRENCY（1，默认）**：

```cpp
while (is_query_ready_to_process(my_tp_group) && (!is_tp_shutdown())) {
  if ((count_waiting_queries(my_tp_group) > 1) ||
      my_tp_group->waiting_thread == nullptr) {
    /*
      ... However we will not wake up any "cold" thread (reserve threads),
      we only want to use "hot" threads (consumer threads).
      This is to avoid polluting the CPU caches...
      At most we can have 3 threads working in parallel per thread group:
      one executing, one waiting for events, one on the consumer cond.
    */
    tp_wake_thread(my_tp_group, WAKE_IF_CONSUMER);
  }
  if (my_tp_group->waiting_thread == nullptr &&
      !is_high_prio_query_available(my_tp_group)) {
    handle_waiting_thread(my_tp_group, my_thread_data, false);
    if (count_waiting_queries(my_tp_group) == 0) continue;
  }
  process_queued_query(my_tp_group, my_thread_data);
}
```

并发上界：**每组最多 3 线程并行**，实际通常 1-2。

> ⚠️ **文档空白**：官方 Operation/Tuning 页都**没有**给出 0/1 算法的机制性描述，Tuning 只说"推荐设为 1"。源码注释标注 **Deprecated**。剖析时不要臆造官方算法细节。

#### 5.2 tp_process_event：执行主体

这是最关键的一段，值得逐段读：

```cpp
static inline bool tp_process_event(tp_group_t *my_tp_group,
                                    tp_thread_t *my_thread_data) {
  tp_client_low_level_t *client_cntx = my_thread_data->client_low_level_cntx;
  THD *thd = client_cntx->thd;
  std::uint32_t txnlim = configured_max_transactions_limit_per_group(my_tp_group);

  unsigned int errcode = 0;
  auto err_grd = create_scope_guard([&]() {
    tp_full_cleanup(thd, my_thread_data, client_cntx, my_tp_group, errcode);   // RAII 兜底
  });
  if (unlikely(thread_attach(thd, my_thread_data, my_tp_group))) return true;

  // ★ MTL 计数，同样用 scope guard 保证异常安全
  if (txnlim != 0) my_tp_group->trxn_threads++;
  auto trxn_threads_decrement_guard = create_scope_guard([&]() {
    if (txnlim != 0) my_tp_group->trxn_threads--;
  });

  unsafe_atomic_add(&client_cntx->management_time,
                    (tp_now() - client_cntx->time_of_event_arrival.load(
                                    std::memory_order_relaxed)));

  // ★ 连接亲和：socket 还有数据就继续读，不重新入队
  do {
    if (tp_execute_command(thd, my_tp_group, my_thread_data)) {
      set_tp_psi_env();
      return true;
    }
    /*
      Allow worker thread to handle next network event from the same client
      if the current transaction has not ended.
    */
    if (!thd_connection_has_data(thd)) {
      if (!thread_pool_max_transactions_limit ||
          !tp_bind_to_current_worker_thread(thd))
        break;
    }
    /*
      There is cached data already read off of the socket, so we can't wait
      safely for new events on socket, we need to process the next event
      immediately. This can happen e.g. for SSL implementations of vio.
    */
  } while (true);

  thread_detach(thd, my_thread_data, my_tp_group);
  my_thread_data->event_count.fetch_add(1, std::memory_order_relaxed);
  unsafe_atomic_add(&my_thread_data->acc_event_time, tp_now() - time_of_enter);

  /*
    This is the limit at which we can no longer use the thd object.
    As soon as we rearm the socket, another thread can pick up a
    query on the socket and start executing it.

    The system call to rearm also ensures a memory barrier for the thd object.
  */
  if (unlikely(tp_client_low_level_rearm(client_cntx, my_tp_group) != 0)) {
    if (handle_rearm_error(my_tp_group, client_cntx)) {
      errcode = ER_ERROR_ON_READ;
      return true;
    }
    err_grd.release();
    return true;
  }
  client_cntx->conn_state = Connection_state::ARMED;
  err_grd.release();
  return false;
}
```

**五个要点**：

1. **`do-while` 循环读多条命令**：只要 socket 还有数据（或 MTL 开启且事务未结束），就继续在同一线程执行，**不重新入队**。这是"连接亲和性"优化——同一事务的多个语句尽量在同一线程连续执行，最大化 cache 命中
2. **`trxn_threads` 用 RAII guard 增减**：保证任何退出路径都能正确归还配额
3. **`rearm` 是 THD 可用性的最后界限**：一旦 rearm，别的线程就能接过这个连接的查询。注释明确说 **rearm 系统调用本身提供内存屏障**
4. **SSL 场景必须连读**："cached data already read off of the socket" —— SSL 层可能已缓冲了多条命令，不能安全地回到 epoll 等待
5. **`management_time` 累计**：记录"事件到达 → 开始执行"的耗时，这是 `tp_connections` 表的核心指标

#### 5.3 thread_attach / thread_detach

```cpp
static inline bool thread_attach(THD *thd, tp_thread_t *my_thread_data, ...) {
  /*
    Set process count to zero, this will ensure that the stall check thread
    can keep track of our progress... We need to know the start of the stack
    so that we could check for stack overruns. We need to setup performance
    schema accounting to user connection and we need to increase number of
    active threads to keep our credit system working properly.
  */
  thd_set_thread_stack(thd, my_thread_data->stack_start);   // 栈溢出检测
  set_user_psi_env(thd);                                     // PSI 切到用户会话
  thd_store_globals(thd);                                    // 绑定 THD 到本线程
  thd_clear_errors(thd);
  return false;
}

static void thread_detach(THD *thd, tp_thread_t *my_thread_data, ...) {
  set_tp_psi_env();                    // 先切回线程池插桩（更新组状态）
  set_user_psi_env(thd);               // 恢复用户会话插桩
  thd_set_net_read_write(thd, 1);      // 标记"正在读"
  /*
    The last thing we do here on the THD and client context is an
    operation that requires a mutex, this mutex will also be a
    memory barrier for the THD and client context object to ensure
    that the next thread that operates on these object see our updates.
  */
  thd_set_not_killable(thd);           // ★ 借这把锁充当内存屏障
}
```

**`thread_detach` 里 `thd_set_not_killable(thd)` 的注释很妙**：明明只是设置"不可 kill"标志，却特意选了它作为最后一步，因为**它内部会取 mutex，顺带充当内存屏障**，保证下一个接管该连接的线程能看到本线程的更新。

#### 5.4 tp_execute_command 与 transaction delay

```cpp
static inline bool tp_execute_command(THD *thd, tp_group_t *my_tp_group,
                                      tp_thread_t *my_thread_data) {
  bool admingrp = (my_tp_group == admin_thread_group);

  if (!admingrp) {                                          // ★ admin 组豁免
    if (!thd_in_active_multi_stmt_transaction(thd)) {
      auto txn_delay = get_transaction_delay();
      if (txn_delay != txn_delay.zero()) {
        my_thread_data->client_low_level_cntx->wait_type = THD_WAIT_TRX_DELAY;
        std::this_thread::sleep_for(txn_delay);             // ★ 真的 sleep
        ++my_tp_group->stats.wait_counts[THD_WAIT_TRX_DELAY];
      }
    }
  }

  inc_queries_executed(my_tp_group);
  thd_set_net_read_write(thd, 0);                           // 执行期间不是"读"

  // We need to ensure the connection isn't broken both before and after
  if (unlikely(!thd_connection_alive(thd) || do_command(thd) ||
               !thd_connection_alive(thd))) {
    return true;
  }
  my_thread_data->command_count.fetch_add(1, std::memory_order_relaxed);
  unsafe_atomic_add(&my_thread_data->acc_command_time, tp_now() - time_of_enter);
  return false;
}
```

`thread_pool_transaction_delay`（8.0.31+）：worker 线程在执行新事务前 **sleep 指定毫秒**。用途是并行事务影响 online index creation / buffer pool resize 时降低争用，"The delay has a throttling effect on transactions"（官方）。

#### 5.5 优先级与防饿死

⚠️ **勘误**：
- **不存在 `queue_put()` / `queue_get()`**——入队 `queue_query()`（:1534），出队 `pop_highest_priority_query()`（:4743），选队列 `next_query_queue()`（:823）
- **不是靠 tickets 防饿死**——靠 `check_trans_queue_for_prio_kickups()`，且**每 10ms 最多提升 1 个**

**入队的优先级判据**（`thread_pool.cc:1534`）：

```cpp
/**
  Support method to store query in prio queue
*/
static inline void queue_query(tp_group_t *my_tp_group,
                               tp_client_low_level_t *client_cntx) {
  DBUG_PRINT("tp", ("Queueing query from connection id %lu",
                    client_cntx->connection_id));
  if (thd_is_transaction_active(client_cntx->thd) ||
      thread_pool_prio_kickup_timer == 0 ||
      is_high_priority_connection(client_cntx->thd)) {
    my_tp_group->queued_queries.push_back(client_cntx);
    //  ★★ 高优先：①事务已开启 ②kickup=0（等于禁用低优先队列）③会话设了 high_priority
  } else {
    my_tp_group->queued_trans.push_back(client_cntx);
    //  ★ 低优先：非事务中的普通查询
  }
  client_cntx->is_queued = true;
  client_cntx->conn_state = Connection_state::QUEUED;
}
```

> ⚠️ **命名与直觉相反**：成员变量 `queued_queries` 是**高**优先队列，`queued_trans` 是**低**优先队列（`methods.h:366-369` 原文注释 `/** List of low prio queries */ Query_list queued_queries;` ... 等等——实际是变量名与注释在源码里就有歧义，判据以 `queue_query()` 的 `if` 为准）。

**出队：严格优先 + 节流**（`thread_pool.cc:816`）：

```cpp
static inline bool is_query_available(tp_group_t *my_tp_group) {
  return !my_tp_group->queued_queries.is_empty() ||
         !my_tp_group->queued_trans.is_empty();
}

/** Returns a pointer to the query(connection) queue which has
    the next (highest priority) query(connection) */
static inline Query_list *next_query_queue(tp_group_t *my_tp_group) {
  if (!my_tp_group->queued_queries.is_empty())
    return &my_tp_group->queued_queries;   // ★ 高优先非空就永远先取它
  return &my_tp_group->queued_trans;       // ★ 只有高优先空了才碰低优先 —— 这就是节流
}

static inline tp_client_low_level_t *next_queued_query(tp_group_t *my_tp_group) {
  return next_query_queue(my_tp_group)->front();
}
```

```cpp
// thread_pool.cc:4743
static inline tp_client_low_level_t *pop_highest_priority_query(
    tp_group_t *my_tp_group) {
  auto next_queue = next_query_queue(my_tp_group);
  tp_client_low_level_t *client_cntx = next_queue->front();
  if (client_cntx != nullptr) {
    next_queue->remove(client_cntx);
    assert(client_cntx->is_queued);
    client_cntx->is_queued = false;
  }
  return client_cntx;
}
```

**★ 防饿死：prio kickup**（`thread_pool.cc:4710`）——低优先等待过久被"踢"进高优先：

```cpp
/**
  Check if any query needs to have its priority increased

  @param my_tp_group        Thread group data
  @param new_10ms           Is it a new 10ms interval
*/
static void check_trans_queue_for_prio_kickups(tp_group_t *my_tp_group,
                                               bool new_10ms,
                                               Time_pt time_of_check) {
  tp_client_low_level_t *client_cntx;

  client_cntx = my_tp_group->queued_trans.front();
  /*
    At most we will move one transaction per 10ms interval. We will only
    move when a transaction has waited longer than the set maximum wait
    time. Setting this time to zero effectively means we're no queueing
    transactions at all.
  */
  if (client_cntx && new_10ms &&
      (time_of_check -
           client_cntx->time_of_event_arrival.load(std::memory_order_relaxed) >
       Misec(thread_pool_prio_kickup_timer))) {
    assert(client_cntx->is_queued);
    my_tp_group->queued_trans.remove(client_cntx);
    my_tp_group->queued_queries.push_back(client_cntx);   // ★★ 提升
    /* Update statistics on prio kickups */
    my_tp_group->stats.prio_kickups++;                    // ★ P_S PRIO_KICKUPS 列
  }
}
```

> ★★ **三个硬约束**：① 只提升**队首**一个；② 每 **10ms** 最多一次（`new_10ms` 由 stall checker 主循环算出）；③ 阈值是 `thread_pool_prio_kickup_timer`（默认 1000ms）。**把它设成 0 等于取消低优先队列**（所有查询一律高优先）。

**会话级优先级**（`thread_pool_plugin.cc:1108`）——只有 2 档：

```cpp
bool is_high_priority_connection(THD *thd) {
  return (THDVAR(thd, high_priority_connection) != 0);
}

static int high_priority_connection_check(THD *, SYS_VAR *, void *save,
                                          struct st_mysql_value *value) {
  longlong new_val;
  if (value->val_int(value, &new_val)) return 1;
  *(longlong *)save = new_val;
  if (new_val >= 0 && new_val <= 1)   /* 2 prio levels supported */
    return 0;
  return 1;
}
```

**`is_query_ready_to_process()` —— 队列 + credit 合判**（`thread_pool.cc:1504`）：

```cpp
static inline bool is_query_ready_to_process(tp_group_t *my_tp_group) {
  tp_client_low_level_t *next = next_queued_query(my_tp_group);
  if (next == nullptr) return false;

  if (is_credit_available(my_tp_group)) return true;

  next->conn_state = Connection_state::WAITING_FOR_CREDIT;
  //  ★★ 这个状态是排查"为什么查询不动"的关键：队列里有活但 credit 不够
  return false;
}

static inline bool is_high_prio_query_available(tp_group_t *my_tp_group) {
  return !my_tp_group->queued_queries.is_empty();
}
```

> ★ `WAITING_FOR_CREDIT` 是**唯一能直接观测"credit 不足"的途径**——`TP_THREAD_STATE` 表里连接处于这个状态时，说明队列非空但被 MTL / max_active_threads 卡住了。

> 官方："There are two queues... **The first statement in a transaction goes to the low-priority queue. Any following statements for the transaction go to the high-priority queue if the transaction is ongoing**, or to the low-priority queue otherwise."

**⚠️ autocommit 陷阱**（官方明确，源码看不出来）：

> "Statements for a **nontransactional storage engine**, or a transactional engine **if `autocommit` is enabled**, are treated as **low-priority** statements... With `autocommit` enabled, **all statements have low priority**."

MySQL 默认 `autocommit=1` → **所有语句都是低优先级，优先级机制实际失效**。

**防饿死（kickup）**：

> "If a statement stays in the low-priority queue too long, the thread pool moves to the high-priority queue... **For each thread group, a maximum of one statement per 10ms (100 per second)** is moved."

默认 `prio_kickup_timer = 1000ms`。

**极端延迟推演**（官方定量例）：低优队列 200 条 + 高优 10 条、`kickup=10s`、`stall_limit=1s` → **"it takes 222 seconds before the new statement starts executing."**

### 六、并发控制：credit 与 MTL

#### 6.1 三档判定（is_credit_available）

**全文**（`thread_pool.cc:1468`）：

```cpp
/**
  Support method to check if credit is available

  @param my_tp_group        Thread group data

  @retval                   true if credit available, otherwise false
*/
static inline bool is_credit_available(tp_group_t *my_tp_group) {
  DBUG_TRACE;
  bool res;
  std::uint32_t txnlim =
      configured_max_transactions_limit_per_group(my_tp_group);
  if (txnlim != 0)
    res = (my_tp_group->trxn_threads <= txnlim || is_mtl_suspended(my_tp_group));
  else if (thread_pool_max_active_query_threads)
    res = (my_tp_group->threads_active < thread_pool_max_active_query_threads);
  else
    res = (my_tp_group->threads_active <= my_tp_group->max_active_threads);

  DBUG_LOG("tp_query_ready", X_(res));
  return res;
}
```

**三档汇总**：

| 档 | 触发条件 | 判定式 | 语义 |
|---|---|---|---|
| **① MTL** | `thread_pool_max_transactions_limit_per_tg != 0`（admin 组恒 0） | `trxn_threads <= txnlim` **或** MTL 已挂起 | 限**并发事务数** |
| **② 已废弃** | `thread_pool_max_active_query_threads != 0` | `threads_active < max_active_query_threads` | 变量描述原文：*"Deprecated. Use is dangerous and could lead to deadlocks."* |
| **③ 默认** | 以上都为 0 | `threads_active <= max_active_threads` | `max_active_threads` = `threads_user_request` + `stalled_threads` |

**credit 上限的动态来源**（`update_max_active_threads()`，`thread_pool.cc:662`）：

```cpp
/**
  Recalulates the number of max_active threads from
  tp_group_t::stalled_threads and tp_group_t::threads_user_request.
 */
static inline int update_max_active_threads(tp_group_t *my_tp_group) {
  mysql_mutex_assert_owner(&my_tp_group->LOCK_group);
  auto old_max_active_threads = my_tp_group->max_active_threads;
  auto threads_user_request = my_tp_group->threads_user_request;

  /*
    Allow user defined number of threads per group + allow stalled
    threads to also continue without being counted in group.
  */
  auto new_max_active_threads =
      threads_user_request + my_tp_group->stalled_threads;
  //  ★★ 每检测到一个 stall，上限 +1 → 允许更多并发
  my_tp_group->max_active_threads = new_max_active_threads;

  return (new_max_active_threads - old_max_active_threads);
  //  ★★ 返回值 > 0 时 stall 线程会去唤醒线程
}
```

```cpp
// methods.h:417 —— max_active_threads 的官方定义
  /**
    max_active_threads is the maximum number of active threads we are
    allowing to execute in parallel in this thread group. Whenever a
    thread is deemed as stalled this number will be incremented and
    similarly it is decremented when a stalled thread has completed
    query execution.
  */
  std::uint32_t max_active_threads{0};
```

> ★★ **credit 不是"票据"（ticket），而是"当前活跃线程数是否越过动态上限"的布尔判定**。`threads_active` 由 `inc/dec_active_threads()` 维护，由 `thd_pool_wait_begin/end()` 在阻塞前后动态加减——**阻塞的线程不占 credit**。这是与 MariaDB/Percona "tickets" 机制最本质的区别。
>
> ⚠️ **`.tmp_tp/` 全目录 grep `ticket` = 0 命中**——企业版完全没有 tickets 概念。
>
> ⚠️ **不存在 `too_many_active_threads()`**——等价判定全部收敛到 `is_credit_available()` + `update_max_active_threads()`。

**STALLED_THREAD_COUNT 列的真实算法**（`table_thread_group_state.cc:137`）——不是独立计数器：

```cpp
          grp.max_active_threads - grp.threads_user_request,
          //  ★ STALLED_THREAD_COUNT = max_active_threads - threads_user_request
```

> ★ 因为 `max_active_threads = threads_user_request + stalled_threads`，所以差值恰好等于 `stalled_threads`。这是"用两个已有量算第三量"的省内存写法。

```cpp
std::uint32_t txnlim = configured_max_transactions_limit_per_group(my_tp_group);
if (txnlim != 0)
  res = (my_tp_group->trxn_threads <= txnlim || is_mtl_suspended(my_tp_group));
else if (thread_pool_max_active_query_threads)
  res = (my_tp_group->threads_active < thread_pool_max_active_query_threads);
else
  res = (my_tp_group->threads_active <= my_tp_group->max_active_threads);
```

| 优先级 | 条件 | 度量 |
|---|---|---|
| 1 | `max_transactions_limit > 0` | **并发事务数**（MTL） |
| 2 | `max_active_query_threads > 0` | 活跃查询线程数（固定） |
| 3 | 默认 | 活跃线程数 ≤ `max_active_threads`（**动态**） |

额度不足 → `next->conn_state = Connection_state::WAITING_FOR_CREDIT`，PFS 可见。

#### 6.2 长事务背压（check_longrunning_transactions_and_adjust_MTL）

```cpp
auto longrunning_threshold = now - ATMC_longrun_trx_limit.load(...);   // 默认 2000ms
std::uint32_t longrunning = std::count_if(..., [&](const tp_thread_t &t) {
  return t.time_of_attach > BEGINNING_OF_EPOCH &&
         t.time_of_attach < longrunning_threshold;
});

if (!is_mtl_suspended(my_tp_group)) {
  if (longrunning >= txnlim) {                     // 长事务吃光配额 → 挂起
    LogErr(INFORMATION_LEVEL, ER_THREAD_POOL_MTL_DISABLE, ...);
    my_tp_group->time_of_last_longrunning_trxs_check = now;   // >EPOCH 即"挂起"标志
  }
  return;
}
if (longrunning > (txnlim / 2)) { ...; return; }   // 仍超一半 → 维持挂起（滞回）
if (my_tp_group->time_of_last_longrunning_trxs_check < gt) {   // 降半以下 + 过 grace → 恢复
  LogErr(INFORMATION_LEVEL, ER_THREAD_POOL_MTL_REENABLE, ...);
  my_tp_group->time_of_last_longrunning_trxs_check = BEGINNING_OF_EPOCH;
}
```

**背压 + 滞回**：挂起阈值 `txnlim`，恢复阈值 `txnlim/2` + grace period，防抖动。意义是让少数慢长事务不至于把短事务全部堵死。

### 七、stall 检测与引擎协同

#### 7.1 判定：基于单查询执行时长（mark_and_count_stalled_threads）

```cpp
auto stalled_threshold = now - get_stall_limit_us();      // 默认 6 厘秒 = 60ms
for (uint i = 0; i < my_tp_group->max_thread_ids_in_group; ++i) {
  tp_thread_t *cur_thread = &(my_tp_group->group_threads[i]);
  if (cur_thread->client_low_level_cntx == nullptr ||
      cur_thread->time_of_attach == BEGINNING_OF_EPOCH) continue;
  if (cur_thread->stalled) { ++threads_stalled; continue; }
  if (cur_thread->time_of_attach < stalled_threshold) {
    cur_thread->stalled = true;                            // ★ 标记
    ++stalled_queries_detected;
    ++threads_stalled;
  }
}
my_tp_group->stalled_threads = threads_stalled;
my_tp_group->stats.stalled_queries_executed += stalled_queries_detected;
```

#### 7.2 精髓：stalled 线程不计入配额（update_max_active_threads）

```cpp
auto threads_user_request = my_tp_group->threads_user_request;
/*
  Allow user defined number of threads per group + allow stalled
  threads to also continue without being counted in group.
*/
auto new_max_active_threads = threads_user_request + my_tp_group->stalled_threads;
my_tp_group->max_active_threads = new_max_active_threads;
return (new_max_active_threads - old_max_active_threads);
```

> 源码注释（methods.h）："a long-running query will quickly be deemed as stalled and thereafter the thread pool will more or less **treat it as a separate thread** until the query completes."

> 官方："The thread pool focuses on **limiting the number of concurrent short-running statements**... If the statement executes past the stall time, it is permitted to continue but **no longer prevents other statements from starting**."

**即**：只有短查询会阻止其他查询启动；长查询超过 60ms 后自动获得"独立名额"，不再挤占组内预算。

#### 7.3 stall 检查主循环

```cpp
while (!is_tp_shutdown()) {
  for (uint i = 0; (i < tp_groups) && (!is_tp_shutdown()); i++) {
    mysql_mutex_lock(&cur_group->LOCK_group);
    mark_and_count_stalled_threads(cur_group, stall_check_timpt);
    auto change_active_threads = update_max_active_threads(cur_group);
    if (do_longrun_check && cur_group != admin_thread_group)
      check_longrunning_transactions_and_adjust_MTL(cur_group, ...);
    /* ... 四情形唤醒决策 ... */
    terminate_expired_connections(cur_group, stall_check_timpt);
    mysql_mutex_unlock(&cur_group->LOCK_group);
  }
  sleep_stall_check(actions_taken, &wakeup_reason);
}
```

**唤醒决策四情形**：
- (a) waiting thread 自己 stalled → 解除角色让别人接替
- (b) `max_active_threads` 变大，或 无线程但有 query ready
- (c) credit 可用但无 waiting thread（仅 high concurrency）
- (d) MTL 提升后 WAITING_FOR_CREDIT 的查询可开跑

```cpp
/*
  It's necessary to be conservative here about when to start a new
  thread. If we start a new thread too often, or even wake a thread
  up too often, we quickly lose CPU cache benefits, an earlier
  variant of this condition lost 15% in performance simply by
  waking threads up to frivolously.
*/
```

**睡眠策略**：有动作 1ms（`SC_SLEEPING_SHORT`），无动作 10ms（`SC_SLEEPING_LONG`）。该线程还负责过期连接清理 + 周期连接数报告（默认 120s）。

#### 7.4 参数单位与官方口径

> "The value of `thread_pool_stall_limit` **represents a number of 10-millisecond intervals**, so that `600` (the maximum) represents 6 seconds."
> 官方例：99.9% 语句 100ms 内完成 → 设 **10（=100ms）**；默认 **6（60ms）** 适合简单语句。
> 范围：**min 4（40ms），max 600（6s），default 6**。

#### 7.5 wait_begin / wait_end：设计的灵魂（3800 / 3913）

```cpp
void thd_pool_wait_begin(THD *thd, int wait_type) {
  client_cntx = (tp_client_low_level_t *)thd_get_scheduler_data(thd);
  my_tp_group = tp_group_low_level_get_tp_group(client_cntx->group);
  my_thread_data = client_cntx->processing_thread;

  // 只能由持有该连接的原 worker 线程调用
  if (my_thread_self() != my_thread_data->thread_handle.thread) return;
  if (client_cntx->wait_type > THD_WAIT_NONE) return;   // 不允许嵌套

  client_cntx->wait_type = static_cast<THD_wait_type>(wait_type);

  if (!my_thread_data->stalled) {
    my_thread_data->not_stalled = true;
    dec_active_threads(my_tp_group, my_thread_data);     // ★ 释放 credit
  }

  mysql_mutex_lock(&my_tp_group->LOCK_group);
  if (my_tp_group->waiting_thread == my_thread_data) {
    /*
      Since I'm stalling, I'll no longer be the waiting thread.
      It's important to create a thread to avoid a deadlock here
      since we have no check in stall check thread for a missing
      waiter thread.
    */
    my_tp_group->waiting_thread = nullptr;
    level = WAKE_OR_CREATE_ONE;                          // ★ 必须创建
  }
  my_tp_group->stats.wait_counts[wait_type]++;           // 分类统计
  tp_wake_thread(my_tp_group, level);                    // ★ 无条件
  mysql_mutex_unlock(&my_tp_group->LOCK_group);
}
```

```cpp
/*
  The most important thing here is not really to wake the thread up. It's
  quite likely that it will go to sleep again. However even if this happens
  it will move from the reserve thread pool to the consumer pool of threads
  very likely. This is needed here to avoid that we decrease the concurrency
  level on the thread group. This is why we need to make the wake up
  unconditional here, it's more about moving the thread to the consumer pool
  more than about waking a thread up.
*/
```

**唤醒的真正目的常常是把线程从冷池挪到热池**，维持组并发能力。这是只有读源码才能体会到的设计意图。

#### 7.6 冷热线程池与创建节流

```cpp
enum wake_level { WAKE_IF_CONSUMER = 0, WAKE_IF_CREATED = 1, WAKE_OR_CREATE_ONE = 2 };

static void tp_wake_thread(tp_group_t *my_tp_group, wake_level level) {
  if (my_tp_group->threads_for_consumer > 0) {
    mysql_cond_signal(&my_tp_group->COND_consumer);        // 热线程优先
  } else {
    if (level == WAKE_IF_CREATED || level == WAKE_OR_CREATE_ONE) {
      if (my_tp_group->threads_for_reserve > 0) {
        mysql_cond_signal(&my_tp_group->COND_reserve);     // 其次冷线程
      } else if (level == WAKE_OR_CREATE_ONE) {
        if (my_tp_group->num_query_threads > 1 + my_tp_group->stats.connection_count ||
            my_tp_group->num_query_threads.load() > MAX_THREADS_PER_GROUP)
          return;                                          // 线程数已超连接数
        if (my_tp_group->threads_active == 0 ||
            tp_thread_create_control_ctx.total_query_threads < 3 * vcpu_count) {
          tp_create_worker_threads(...);                   // 全局 < 3×vCPU 立即建
        } else if ((now - last_thread_creation_time) > tp_thread_create_ctrl_interval()) {
          tp_create_worker_threads(...);                   // 否则按间隔节流
        }
      }
    }
  }
}
```

> "always use the thread with most of its stack in the CPU cache... gives a significant boost of **5-10%** in performance in experiments."

**全局硬上限 3×vCPU**（`vcpu_count` 来自 `my_num_vcpus()`，26.7 修复了 cgroup/容器感知）。

### 八、关闭与卸载路径

> 前面各节讲线程池"跑起来"，本节讲它"停下来"——这是本篇此前唯一缺失的模块。

#### 8.1 thd_pool_end：三步协作式停止

```cpp
void thd_pool_end() {
  DBUG_TRACE;
  assert(std::this_thread::get_id() == plugin_init_tid);

  // This check is necessary because thd_pool_end() is called as a cleanup
  // function also before thd_pool_init() has completed. Notably
  // my_connection_handler_reset() calls it and this function is called to
  // clean up after failures in thread_pool_plugin_init() before
  // thd_pool_init() has been called.
  if (thd_pool_init_completed == false) return;

  assert(thd_pool_init_completed);
  assert(tp_shutdown.load() == false);
  tp_shutdown.store(true);

  // Signal and wait for stall check thread to stop. Destroy its mutex and cond var.
  shutdown_stall_check_thread();

  // Wait for all threads in thread groups to exit.
  shutdown_thread_groups(tp_groups);

  // Free memory attached to each thread group, including the admin thread group.
  deinit_thread_groups(tp_groups);
}
```

| 步骤 | 动作 | 说明 |
|---|---|---|
| 1 | `tp_shutdown.store(true)` | 原子标志置位；**所有线程的主循环条件 `!is_tp_shutdown()` 立刻变假** |
| 2 | `shutdown_stall_check_thread()` | 停**全局唯一**的 stall 检测线程，销毁它的 mutex/cond |
| 3 | `shutdown_thread_groups()` + `deinit_thread_groups()` | 等各线程组线程退出，再释放内存（**含 admin 组**） |

**这是标准的协作式停止**（与 [`thread_lifecycle.md`](thread_lifecycle.md) 一致）：不 kill，而是"置标志 → 线程自己走到循环条件退出 → 等待"。

代价是**线程必须周期性回到循环条件检查点**。若某线程正阻塞在 `epoll_wait` 上，光置位 `tp_shutdown` 不会唤醒它——所以还需要 4.1 的 self-pipe 通知配合。`is_tp_shutdown()` 在源码里出现在 **20 余处**循环条件中（listener 循环、专用 listener 循环、两种调度算法主循环、stall check 循环等），构成协作式停止的"检查点网络"。

#### 8.2 三个防御性细节

1. **`assert(std::this_thread::get_id() == plugin_init_tid)`**——必须在**初始化时那个线程**里关闭。这是插件 init/deinit 的线程亲和性约束。
2. **`thd_pool_init_completed == false` 时直接返回**——`my_connection_handler_reset()` 会在 `thd_pool_init()` **尚未完成**时就调用 `thd_pool_end()`（清理 `thread_pool_plugin_init()` 的失败路径）。没有这个守卫会对未初始化的结构做清理。
3. **`assert(tp_shutdown.load() == false)`**——防重复关闭。

#### 8.3 初始化失败回滚：scope_guard

```cpp
bool thd_pool_init() {
  DBUG_TRACE;

  uint tp_groups_initialized = 0;
  auto init_guard = create_scope_guard([&] {
    tp_shutdown.store(true);
    shutdown_thread_groups(tp_groups_initialized);
    deinit_thread_groups(tp_groups_initialized);
  });
  ...
```

初始化用 `create_scope_guard` 注册回滚：只要中途 return（如 PSI key 初始化失败），guard 自动执行"置关闭标志 + 关闭已初始化的组 + 释放"。

注意它关闭的是 **`tp_groups_initialized`（已初始化的数量）**而非全部 `tp_groups`——因为失败时可能只初始化了一部分。这个计数变量是回滚正确性的关键。

> 与 [`thread_lifecycle.md`](thread_lifecycle.md) 里 `MySQL_thread` 的 `preamble`/`epilogue` 属同一类"手写 RAII"：C++ 没有直接的线程生命周期 RAII，只能靠 scope guard 或成对钩子模拟。

---

### 九、关键函数源码级剖析

> 前面八节按"功能域"组织，不少核心函数被概括而未提其名。本节补齐**函数级**剖析——这些函数是理解 26.7 行为的直接抓手。

#### 9.1 冷热线程切换：`become_consumer_thread` / `become_reserve_thread`

```cpp
static bool become_consumer_thread(tp_group_t *my_tp_group,
                                   tp_thread_t *my_thread_data) {
  /* I'll be a consumer thread */
  my_tp_group->threads_for_consumer++;
  my_tp_group->stats.become_consumer_thread++;
  set_state(my_thread_data, Thread_state::SLEEPING_CONSUMER);
  std::ignore =
      mysql_cond_wait(&my_tp_group->COND_consumer, &my_tp_group->LOCK_group);
  set_state(my_thread_data, Thread_state::MANAGING);
  my_tp_group->threads_for_consumer--;
  return false;
}

static bool become_reserve_thread(tp_group_t *my_tp_group,
                                  tp_thread_t *my_thread_data) {
  if (thread_pool_max_unused_threads != 0 &&
      ((my_tp_group->threads_for_reserve + 1) >=
       thread_pool_max_unused_threads)) {
    /*
      We already have a sufficient number of reserve threads. We'll
      quit this thread.
    */
    return true;                       // ★ 返回值 true = 线程该退出了
  }
  /* I'll become a reserve thread */
  ...
```

三个要点：

1. **`become_reserve_thread` 的返回值是"是否退出"**——返回 true 时调用方会终止该线程。这就是 `thread_pool_max_unused_threads`（26.7 默认已从 2 提到 32）的**实际作用点**：限制"冷池"里囤积多少空闲线程。
2. **consumer 醒后回到 `MANAGING` 态**再去干活，不直接执行——状态机的每一次转换都有明确语义（见 2.4）。
3. `stats.become_consumer_thread` / `become_reserve_thread` 就是 PFS 表 `tp_thread_group_stats` 里那两列的来源（见可观测性章）。

**为什么分两层**：consumer 是"刚干过活、栈还在 cache 里"的热线程，reserve 是"睡久了、cache 已凉"的冷线程。唤醒优先级不同（见 5 章的 `WAKE_IF_CONSUMER` 三级）。

#### 9.2 队列选取：高优绝对优先

```cpp
static inline Query_list *next_query_queue(tp_group_t *my_tp_group) {
  if (!my_tp_group->queued_queries.is_empty())
    return &my_tp_group->queued_queries;      // 高优
  return &my_tp_group->queued_trans;          // 低优
}

static inline tp_client_low_level_t *next_queued_query(tp_group_t *my_tp_group) {
  return next_query_queue(my_tp_group)->front();
}

static inline tp_client_low_level_t *pop_highest_priority_query(
    tp_group_t *my_tp_group) {
  auto next_queue = next_query_queue(my_tp_group);
  tp_client_low_level_t *client_cntx = next_queue->front();

  if (client_cntx != nullptr) {
    next_queue->remove(client_cntx);
    assert(client_cntx->is_queued);
    client_cntx->is_queued = false;
  }
  return client_cntx;
}
```

**"最高优先级"的语义就是：只要高优队列非空，低优队列一律不看。** 这不是加权轮转，是**严格优先**——低优查询只有在高优队列完全排空时才有可能被选中（饿死风险由此而来，防饿死机制见对比篇 6.6）。

`is_queued` 标志配合 `assert` 构成不变式：出队即清标志，防止同一连接被两条队列同时持有（侵入式链表共用 `next/prev` 指针，一条连接同时在两条队列会导致链表损坏）。

#### 9.3 ★ 连接亲和的实现：`tp_bind_to_current_worker_thread`

```cpp
/* Check if worker thread needs to bound until transaction commit time. */
static inline bool tp_bind_to_current_worker_thread(THD *thd) {
  return thd_is_transaction_active(thd);
}
```

**就这么一行**——但它决定了"连接亲和连读"的边界：**事务活跃期间，这条连接绑定在当前 worker 线程上**；事务结束才允许解绑、交给别的线程。

为什么必须绑：事务状态（InnoDB `trx_t`、MDL 持有、临时表）与执行线程有耦合，`thd_wait_begin`/`wait_end` 上报的阻塞也必须与同一个线程配对。若事务中途换线程，`wait_begin` 的计数会错乱（credit 释放对不上）。

这与 6.3 节的 `tp_process_event` do-while 连读是一体两面：**连读解决"一个命令内多包"，事务绑定解决"多命令之间"**。

#### 9.4 活跃计数与防重复标志

```cpp
static inline void inc_active_threads(tp_group_t *tp_group,
                                      tp_thread_t *my_thread_data) {
  assert(!my_thread_data->contributes_to_threads_active);
  ++tp_group->threads_active;
  my_thread_data->contributes_to_threads_active = true;
}

static inline void dec_active_threads(tp_group_t *tp_group,
                                      tp_thread_t *my_thread_data) {
  assert(my_thread_data->contributes_to_threads_active);
  --tp_group->threads_active;
  ...
```

★ **`contributes_to_threads_active` 标志 + 两端 `assert`，是"一个线程只能贡献一次计数"的硬保证。**

为什么需要它：`wait_begin`（引擎阻塞前）会 `dec`、`wait_end` 会 `inc`——同一线程在一条命令里可能多次进出等待。若没有这个标志，重复 `inc` 会让 `threads_active` 虚高，**credit 判定（6.1 的三档）全部失效**。这是 credit 机制正确性的地基。

#### 9.5 线程退出时的清理（主循环末尾）

```cpp
  if (my_thread_data->is_connection_handler_thread) {
    my_tp_group->connect_threads--;
  } else {
    my_tp_group->num_query_threads--;
    tp_thread_create_control_ctx.total_query_threads--;   // ★ 全局节流计数
  }

  my_tp_group->threads_initialized--;
  if (my_thread_data->worker_type == Worker_thread_type::QUERY_WORKER_THREAD) {
    mysql_mutex_lock(&my_tp_group->LOCK_query_threads_count);
    thread_pool_query_worker_threads--;
    my_tp_group->query_threads_count--;
    mysql_mutex_unlock(&my_tp_group->LOCK_query_threads_count);
  }

  update_thread_group_thread_exit(my_tp_group, my_thread_data);
```

**四类计数各自递减**，每一类都有独立用途：

| 计数 | 用途 |
|---|---|
| `connect_threads` | 连接处理线程数（CONNECTION_HANDLER 角色） |
| `num_query_threads` | 本组查询线程数（与 `MAX_THREADS_PER_GROUP` 比较） |
| `total_query_threads` | **全局**查询线程数（与 3×vCPU 比较，创建节流，见 5 章） |
| `query_threads_count` | 供 PFS 表 `tp_thread_group_state` 展示 |

`total_query_threads` 单独用一把 `LOCK_query_threads_count` 保护——它是**全局**的（跨组），不能复用组锁。

#### 9.6 ★ MTL 背压的两个时间参数（含硬编码边界）

```cpp
static Misec longrun_grace_period() {
  return std::clamp(15 * ATMC_longrun_trx_limit.load(std::memory_order_relaxed),
                    Misec(5 * 1000), Misec(30 * 1000));
}

/**
  Returns the minimum time which must elapse between each check for long
  running transactions.
*/
static Misec longrun_check_interval() {
  return ATMC_longrun_trx_limit.load(std::memory_order_relaxed) / 2;
}
```

**★ 这两个函数揭示了 6.2 节没写的两个事实**：

1. **grace period 不是直接配置的，而是从 `longrun_trx_limit` 推导的**：`15 × 阈值`，且被 **`std::clamp` 夹在 5~30 秒**之间。
2. **检查间隔 = 阈值的一半**（不是全量）。

按默认值 `longrun_trx_limit = 2000ms` 推导：

| 参数 | 推导 | 默认值下的结果 |
|---|---|---|
| 长事务判定阈值 | `thread_pool_longrun_trx_limit` | 2000ms |
| 检查间隔 | 阈值 / 2 | **1000ms**（每 1 秒查一次长事务） |
| grace period | clamp(15×2000, 5s, 30s) | **30 秒**（正好落在上界） |

**clamp 的上下界会"吃掉"极端配置**——这是调参时必须知道的：

| `longrun_trx_limit` | 15×阈值 | 实际 grace | 说明 |
|---|---|---|---|
| 3000ms | 45s | **30s** | 被上界截断 |
| 2000ms（默认） | 30s | 30s | 恰好在上界 |
| 1000ms | 15s | 15s | 区间内 |
| 500ms | 7.5s | 7.5s | 区间内 |
| 300ms | 4.5s | **5s** | 被下界截断 |

即：**把 `longrun_trx_limit` 调到 300ms 以下，grace 不会低于 5 秒；调到 3000ms 以上，grace 不会超过 30 秒。** 恢复 MTL 的等待时间有硬地板和硬天花板，不随参数线性变化。

这与 6.2 节"滞回（挂起阈值 `txnlim`、恢复阈值 `txnlim/2` + grace）"合起来才是完整的背压时序：

```
长事务数 ≥ txnlim          → 挂起 MTL（立即）
长事务数 > txnlim/2        → 维持挂起
长事务数 ≤ txnlim/2 且
  过了 grace（默认 30s）   → 恢复 MTL
（每 1 秒检查一次上述条件）
```

> 补充未展开的函数族：`init_tp_psi_keys`（PSI key 注册，见 1.4）与 `fill_thread_group_state_table` / `fill_thread_group_stats_table` / `fill_thread_state_table`（PFS 三表的行填充，见可观测性章）——它们是"插件如何把自己接进 PFS"的两端。其余 `get_*` / `set_*` / `remove_*` 为状态读写与链表摘除的辅助函数，机制含义已在对应章节说明。

---

#### 9.7 timer 线程：★ MySQL 有，但不叫 timer

⚠️ **勘误**：**不存在 `timer_thread()` 函数**。真名是 **`tp_stall_check_thread_main()`**（`thread_pool.cc:2845`），全实例**只有 1 个**（不是每 group 一个），PFS 里显示线程名 `tp_stall`。`Worker_thread_type::TIMER_WORKER_THREAD` 只是给它打的枚举标签。

**主循环全文**（`thread_pool.cc:2845`）：

```cpp
extern "C" void *tp_stall_check_thread_main(void *) {
  assert(stall_check_initialized);
  auto thread_end_guard = create_scope_guard([&]() {
    signal_stall_check_thread_change(Stall_checker_state::TERMINATED);
    my_thread_end();
  });

  if (my_thread_init()) return nullptr;
  signal_stall_check_thread_change(Stall_checker_state::RUNNING);

  uint wakeup_reason = 0;
  Conn_tracker conn_tracker;
  Time_pt time_of_completed_check;
  Time_pt time_of_next_longrun_check;

  /* START MAIN FOREVER LOOP */
  while (!is_tp_shutdown()) {
    uint actions_taken = 0;
    Time_pt stall_check_timpt = tp_now();
    auto time_since_completed_check = stall_check_timpt - time_of_completed_check;
    time_of_completed_check = stall_check_timpt;
    bool new_10ms = (time_since_completed_check >= Misec(10));   // ★ 硬编码 10ms：kickup 节流窗口
    ...
    auto lrci = longrun_check_interval();      // ★ MTL 参数①：longrun_trx_limit / 2
    bool do_longrun_check =
        (thread_pool_max_transactions_limit > 0 && lrci > Misec::zero() &&
         stall_check_timpt > time_of_next_longrun_check);
    Time_pt time_of_grace_threshold;
    if (do_longrun_check) {
      time_of_next_longrun_check = stall_check_timpt + lrci;
      time_of_grace_threshold = stall_check_timpt - longrun_grace_period();  // ★ MTL 参数②
    }

    /*
      We have one stall check thread for all thread groups, we loop over
      all thread groups once per time we're waken up. Including the admin
      thread group.
    */
    for (uint i = 0; (i < tp_groups) && (!is_tp_shutdown()); i++) {
      tp_group_t *cur_group = &(tp_group_list[i]);
      mysql_mutex_lock(&cur_group->LOCK_group);

      mark_and_count_stalled_threads(cur_group, stall_check_timpt);   // ①打 stall 标记
      auto change_active_threads = update_max_active_threads(cur_group); // ②抬高 credit 上限

      if (do_longrun_check && cur_group != admin_thread_group) {
        check_longrunning_transactions_and_adjust_MTL(
            cur_group, stall_check_timpt, time_of_grace_threshold);
      }

      tp_thread_t *cur_thread = cur_group->waiting_thread;
      bool wake_thread = false;
      if ((cur_thread != nullptr) &&
          (cur_thread->client_low_level_cntx != nullptr) &&
          cur_thread->stalled) {
        /*
          Deassign the waiting thread since the thread is now stalled
          and wake up a thread to ensure that someone takes up the role
          of being the waiting thread.
        */
        cur_group->waiting_thread = nullptr;   // ③listener 自己 stalled → 剥夺其身份
        wake_thread = true;
      }

      check_trans_queue_for_prio_kickups(cur_group, new_10ms, stall_check_timpt); // ④防饿死

      if (change_active_threads > 0 || (cur_group->threads_for_consumer == 0 &&
                                        cur_group->threads_for_reserve == 0 &&
                                        is_query_ready_to_process(cur_group))) {
        /*
          With the change in max_active_threads it might be credit available
          to start up new jobs and we might be missing threads to execute
          them, so wake a thread and start one up if necessary.

          It's necessary to be conservative here about when to start a new
          thread. If we start a new thread to often, or even wake a thread
          up too often, we quickly lose CPU cache benefits, an earlier
          variant of this condition lost 15% in performance simply by
          waking threads up to frivolously.
        */
        wake_thread = true;
      } else if (is_credit_available(cur_group) &&
                 cur_group->waiting_thread == nullptr &&
                 thread_pool_algorithm == HIGH_CONCURRENCY_ALGORITHM) {
        wake_thread = true;
      } else if (configured_max_transactions_limit_per_group(cur_group) > 0 &&
                 is_query_ready_to_process(cur_group)) {
        // This will catch the case when MTL has been increased ...
        wake_thread = true;
      }

      if (wake_thread) {
        cur_group->stats.wake_thread_stall_checker++;
        tp_wake_thread(cur_group, WAKE_OR_CREATE_ONE);  // ⑤★只有这里才允许建新线程
        actions_taken++;
      }
      terminate_expired_connections(cur_group, stall_check_timpt);   // ⑥清理 EXPIRED
      ...
      mysql_mutex_unlock(&cur_group->LOCK_group);
    }

    sleep_stall_check(actions_taken, &wakeup_reason);   // ⑦自适应睡眠
  }
  return nullptr;
}
```

**自适应睡眠**（`sleep_stall_check()`，`thread_pool.cc:2320`）——★ 硬编码 1ms / 10ms：

```cpp
static inline void sleep_stall_check(int actions_taken, uint *wakeup_reason) {
  assert(stall_check_initialized);

  uint sleep_nanosecs;
  struct timespec time_out;

  if (actions_taken == 0) {
    sleep_nanosecs = 10 * NANOSEC_PER_MILLI;    // ★ 空闲 10ms
    set_state(&tp_stall_check_thread, Thread_state::SC_SLEEPING_LONG);
  } else {
    sleep_nanosecs = 1 * NANOSEC_PER_MILLI;     // ★ 有动作 1ms（快速跟进）
    set_state(&tp_stall_check_thread, Thread_state::SC_SLEEPING_SHORT);
  }

  set_timespec_nsec(&time_out, sleep_nanosecs);
  mysql_mutex_lock(&tp_LOCK_stall_check);
  *wakeup_reason = mysql_cond_timedwait(&tp_COND_stall_check,
                                        &tp_LOCK_stall_check, &time_out);
  mysql_mutex_unlock(&tp_LOCK_stall_check);
  set_state(&tp_stall_check_thread, Thread_state::SC_CHECKING);
}
```

**stall 判定本体**（`mark_and_count_stalled_threads()`，`thread_pool.cc:553`）：

```cpp
static inline void mark_and_count_stalled_threads(tp_group_t *my_tp_group,
                                                  Time_pt now) {
  mysql_mutex_assert_owner(&my_tp_group->LOCK_group);
  std::uint32_t threads_stalled = 0;
  std::uint32_t stalled_queries_detected = 0;
  auto stalled_threshold = now - get_stall_limit_us();   // ★ thread_pool_stall_limit × 10ms

  for (uint i = 0; i < my_tp_group->max_thread_ids_in_group; ++i) {
    tp_thread_t *cur_thread = &(my_tp_group->group_threads[i]);

    // Ignore threads not associated with a connection
    if (cur_thread->client_low_level_cntx == nullptr ||
        cur_thread->time_of_attach == BEGINNING_OF_EPOCH)
      continue;

    if (cur_thread->stalled) { ++threads_stalled; continue; }

    if (cur_thread->time_of_attach < stalled_threshold) {
      cur_thread->stalled = true;                        // ★★ 打标
      ++stalled_queries_detected;
      ++threads_stalled;
    }
  }
  my_tp_group->stalled_threads = threads_stalled;
  my_tp_group->stats.stalled_queries_executed += stalled_queries_detected;
}
```

**stall 状态清除**（`init_process_count_time_unit()`，`thread_pool.cc:696`）：

```cpp
static inline void init_process_count_time_unit(tp_group_t *my_tp_group,
                                                tp_thread_t *my_thread_data) {
  mysql_mutex_assert_owner(&my_tp_group->LOCK_group);
  if (my_thread_data->stalled) {
    --my_tp_group->stalled_threads;
    --my_tp_group->max_active_threads;      // ★ 连带把 credit 上限降回来
    my_thread_data->stalled = false;
  }
  my_thread_data->time_of_attach = BEGINNING_OF_EPOCH;
}
```

> ★★ **为什么主动上报的阻塞不会被误判为 stalled**：`thd_pool_wait_begin()` 只做 `dec_active_threads()`（归还 credit），**不清 `time_of_attach`**。但 `thd_pool_wait_begin()` 里有 `if (!my_thread_data->stalled)` 的判断，而 stall 判定只有在 `time_of_attach < stalled_threshold` 时才置 `stalled = true`——也就是说**一旦被判 stalled，后续 wait_begin 就不再归还 credit**（因为 credit 已经通过 `max_active_threads++` 补偿过了）。两块逻辑靠 `not_stalled` / `stalled` 两个布尔位衔接。
>
> ⚠️ **不存在 `check_stall()`**——那是 MariaDB/Percona 的函数名。企业版的 stall 判定完全在这个独立线程里，且**逐线程打标**与**唤醒决策**是分开的两个函数。

**stall checker 的状态机**（`thread_pool.cc:310`）：

```cpp
static bool stall_check_initialized = false;
enum class Stall_checker_state {
  NOT_STARTED,
  RUNNING,
  TERMINATED
} stall_checker_state;
```

三家都有"全局唯一的定时器线程"，但**命名与职责不同**——这是最容易混淆的一处。

| | MySQL 26.7 | MariaDB | Percona |
|---|---|---|---|
| **PSI 注册名** | **`tp_stall_check`**（短名 `tp_stall`） | `timer_thread` | `timer_thread`（短名 `timer_th`） |
| 角色枚举 / 标志 | `Worker_thread_type::TIMER_WORKER_THREAD` | `PSI_FLAG_GLOBAL` | `PSI_FLAG_SINGLETON` |
| 线程函数 | **`tp_stall_check_thread_main`** | `timer_thread` | `timer_thread` |
| 数量 | 全局 1 个 | 全局 1 个 | 全局 1 个 |
| 检查周期 | ★ 与判定阈值**解耦**（1ms 有动作 / 10ms 无动作） | = `stall_limit`（500ms） | = `stall_limit`（500ms） |
| 额外职责 | 连接数周期报告、过期连接清理 | `wait_timeout` 检查 | `wait_timeout` 检查 |

**所以严格说：MySQL 没有"timer 线程"，它有的是"stall check 线程"**——两者是同一类东西，但 MySQL 按职责命名，两家按机制命名。在 `SHOW PROCESSLIST` / PFS `threads` 表里看到的也是不同的名字。

##### MySQL 的 `tp_stall_check_thread_main`

```cpp
extern "C" void *tp_stall_check_thread_main(void *) {
  assert(stall_check_initialized);

  auto thread_end_guard = create_scope_guard([&]() {
    signal_stall_check_thread_change(Stall_checker_state::TERMINATED);
    my_thread_end();
  });

  if (my_thread_init()) return nullptr;

  signal_stall_check_thread_change(Stall_checker_state::RUNNING);

  Conn_tracker conn_tracker;
  Time_pt time_of_completed_check;

  while (!is_tp_shutdown()) {
    Time_pt stall_check_timpt = tp_now();
    auto time_since_completed_check = stall_check_timpt - time_of_completed_check;
    time_of_completed_check = stall_check_timpt;
    bool new_10ms = (time_since_completed_check >= Misec(10));   // ★ 关键
    ...
```

三个要点：

1. **`new_10ms` 就是在这里算出来的**——"距上次完成检查是否过了 10ms"。它随后传给 `check_trans_queue_for_prio_kickups`（9.6），成为"每组每 10ms 最多踢 1 条"的时间基准。**没有额外定时器，10ms 是靠"每轮循环看时间差"实现的。**
2. **`extern "C"` + `create_scope_guard`**——作为 pthread 入口必须 C 链接；线程结束的清理（置 `TERMINATED` 状态 + `my_thread_end()`）用 scope guard 保证任何退出路径都执行。
3. **启动握手**：`signal_stall_check_thread_change(RUNNING)` 通知等待方（线程池初始化时会等这个信号，见 8 章关闭路径的对称操作）。

##### 两家的 `timer_thread`

MariaDB/Percona 的 timer 线程除了 `check_stall`（6.5 节源码）还负责 `timeout_check`——**清理超过 `wait_timeout` 的空闲连接**。MySQL 的等价职责也放在 stall check 线程里（`Conn_tracker` + 过期连接清理），只是没有单独叫 `timeout_check` 的函数。

两家的周期是 `tick_interval`（= `stall_limit`，默认 500ms），MySQL 则是"有动作睡 1ms、没动作睡 10ms"的自适应——**这就是 6.5 节"响应粒度细两个数量级"的机制根源**。

#### 9.8 timer 线程的睡眠策略、状态机与 PSI 双插桩

##### ★ 自适应睡眠：1ms / 10ms

```cpp
static inline void sleep_stall_check(int actions_taken, uint *wakeup_reason) {
  uint sleep_nanosecs;
  struct timespec time_out;

  if (actions_taken == 0) {
    sleep_nanosecs = 10 * NANOSEC_PER_MILLI;
    set_state(&tp_stall_check_thread, Thread_state::SC_SLEEPING_LONG);
  } else {
    sleep_nanosecs = 1 * NANOSEC_PER_MILLI;
    set_state(&tp_stall_check_thread, Thread_state::SC_SLEEPING_SHORT);
  }

  set_timespec_nsec(&time_out, sleep_nanosecs);

  mysql_mutex_lock(&tp_LOCK_stall_check);
  *wakeup_reason = mysql_cond_timedwait(&tp_COND_stall_check,
                                        &tp_LOCK_stall_check, &time_out);
  mysql_mutex_unlock(&tp_LOCK_stall_check);
  set_state(&tp_stall_check_thread, Thread_state::SC_CHECKING);
}
```

三个要点：

1. **自适应周期**：本轮**没做任何事**（`actions_taken == 0`）就睡 10ms；**做了事**（发现 stall、踢升、唤醒线程等）就只睡 1ms——**忙的时候轮询更密，闲的时候省 CPU**。这正是 6.5 节"响应粒度细两个数量级"的直接实现。
2. **用 `mysql_cond_timedwait` 而不是 `sleep`**：既到了点会醒，也能被别人在 `tp_COND_stall_check` 上 signal 提前唤醒。返回值存进 `wakeup_reason`，调用方可据此区分"超时醒"还是"被叫醒"。
3. **状态随睡眠切换**：`SC_SLEEPING_LONG` / `SC_SLEEPING_SHORT` / `SC_CHECKING`——PFS 里能看到这个线程处在哪种睡眠。

对照两家：MariaDB/Percona 是固定 `tick_interval`（= `stall_limit`），**没有这种忙闲自适应**。Percona 的耦合还有直接源码证据：

```cpp
void tp_start_timer_thread() noexcept {
  pool_timer.tick_interval = threadpool_stall_limit;   // ★ 周期 = 判定阈值
  start_timer(&pool_timer);
}
```

##### 状态机：`Stall_checker_state`

```cpp
enum class Stall_checker_state {
  NOT_STARTED,
  RUNNING,
  TERMINATED
} stall_checker_state;
```

配合 9.7 里的 `signal_stall_check_thread_change()` 做启动/退出握手：初始化线程会 `mysql_cond_wait` 等到状态不再是 `NOT_STARTED`（源码注释即"等待 stall check 线程启动完成"），关闭时（8.1）先 `tp_shutdown.store(true)` 再 signal 唤醒它退出，最后等 `TERMINATED`。

##### cache line 隔离：`alignas(128)`

```cpp
enum class Stall_checker_state { ... } stall_checker_state;

/* Ensure stall check mutexes have their own cache line */
alignas(128) static mysql_mutex_t tp_LOCK_stall_check;
alignas(128) static mysql_cond_t tp_COND_stall_check;
alignas(128) static tp_group_t tp_group_list[MAX_THREAD_GROUPS];
```

三处 `alignas(128)` 各占独立 cache line：stall check 的 mutex、cond、以及**全局线程组数组**。前两者是本线程高频访问的同步原语，后者被所有工作线程访问——**强制隔离避免它们互相 false sharing**。这是 9 章"锁粒度压到极致"决策在全局层面的延伸（组内是手工排字段，全局是 `alignas`）。

##### ★ PSI 双插桩的真实实现：`THR_PSI_backup`

3.2 节提到"PSI 双插桩"但没给源码，实现就在这里：

```cpp
/*
  @def THR_PSI_backup
  Backup for the thread instrumentation.
  A physical thread (as in pthread) can execute either the thread pooling code
  or a logical thread (as in THD).
  Both are instrumented, but with different PSI_thread to have separate
  accounting of waits.
  THR_PSI_backup is used to save the instrumentation of the thread pooling code
  while executing the user job.
*/
thread_local PSI_thread *THR_PSI_backup = nullptr;
```

**一个物理 OS 线程有两套 PFS 身份**：执行线程池自身代码时用一套 `PSI_thread`，执行用户查询时切换到另一套（该用户的）。`THR_PSI_backup` 就是切换时保存"线程池那套"的栈外变量——**`thread_local`，所以每个线程各存一份**。

这解释了官方那句"While a thread executes a statement from a user connection, Performance Schema instrumentation accounts thread activity to the user connection. Otherwise, ... to the thread pool"：它不是靠判断，是靠**真的换了一个 PSI_thread 对象**。

> 相关函数：`init_tp_psi_keys`（注册 PSI key，见 1.4）、`set_tp_psi_env` / `clear_user_psi_env`（两套环境的切入切出）。

#### 9.9 调度主循环：三种模式的分派与源码

> 5 章讲了调度策略，但两个算法函数本身的源码此前只被提及一次。本节补齐，并澄清一个此前没说清的点：**专用 listener 模式的优先级高于 `thread_pool_algorithm`**。

##### 分派结构

工作线程主循环里有三层**互斥**的分支：

```cpp
while (!is_tp_shutdown() && !ret_code) {
  if (my_thread_data->is_connection_handler_thread == true) {
    ret_code = tp_connect_thread_main(my_tp_group, my_thread_data);   // ① 建连/认证线程
    continue;
  }
  if (thread_pool_dedicated_listeners) {                              // ② 专用 listener
    if (my_thread_data->worker_type == Worker_thread_type::LISTENER_WORKER_THREAD)
      dedicated_listener_loop(my_tp_group, my_thread_data);
    else
      ret_code = query_worker_thread_loop(my_tp_group, my_thread_data);
    continue;
  }
  if (thread_pool_algorithm == LOW_CONCURRENCY_ALGORITHM) {           // ③ 默认路径
    ret_code = low_concurrency_algorithm(my_tp_group, my_thread_data);
  } else {
    ret_code = high_concurrency_algorithm(my_tp_group, my_thread_data);
  }
}
```

| 分支 | 条件 | 用途 |
|---|---|---|
| ① | 连接处理线程 | 专门做建 THD + 认证（26.7 独有，两家认证在 worker 内做） |
| ② | **`thread_pool_dedicated_listeners = ON`** | listener 线程只收事件、query 线程只执行 |
| ③ | 默认 | 按 `thread_pool_algorithm` 选低/高并发算法 |

★ **注意优先级**：② 的判断在 ③ **之前**——**一旦开启专用 listener，`thread_pool_algorithm` 参数就不再生效**。而该参数源码注释已标 **deprecated**（见已知限制第 8 条），所以实践中几乎总是走 ③。

##### `low_concurrency_algorithm`：一线程又收又执行

```cpp
static inline bool low_concurrency_algorithm(tp_group_t *my_tp_group,
                                             tp_thread_t *my_thread_data) {
  while ((!is_query_ready_to_process(my_tp_group)) && (!is_tp_shutdown())) {
    if ((!is_query_available(my_tp_group) ||
         configured_max_transactions_limit_per_group(my_tp_group) > 0) &&
        my_tp_group->waiting_thread == nullptr) {
      handle_waiting_thread(my_tp_group, my_thread_data, true);
      return false;
    }
    if (handle_worker_thread_sleep(my_tp_group, my_thread_data)) return true;
  }

  if (my_tp_group->waiting_thread == nullptr &&
      !is_high_prio_query_available(my_tp_group)) {
    handle_waiting_thread(my_tp_group, my_thread_data, false);   // 非阻塞收事件
  }

  while (is_query_ready_to_process(my_tp_group) && (!is_tp_shutdown())) {
    process_queued_query(my_tp_group, my_thread_data);
    if (my_tp_group->waiting_thread == nullptr &&
        !is_high_prio_query_available(my_tp_group)) {
      handle_waiting_thread(my_tp_group, my_thread_data, false);
    }
  }
  return false;
}
```

两个要点：

1. **没有独立 waiter**——注释明说 "we don't operate a separate waiting thread, we take care of getting events as part of the execution loop"。**同一个线程既收事件又执行查询**，CPU cache 局部性最好。代价是执行期间没人收事件，所以每处理完一个查询就回头非阻塞收一次（`wait=false`）。
2. **但仍必须保证 waiter 存在**：第一个 `if` 里，若"没有可用查询"或"配了 MTL"且当前无 waiter，就自己去当 waiter。注释解释了原因——

   > Without this we may miss credit increases due to changed MTL and kill requests.

   即：没人收事件就收不到新的 kill 通知，MTL 变化导致的 credit 增加也会被漏掉。**这是"低并发"模式里一个容易忽略的正确性要求。**

##### `high_concurrency_algorithm`：专职 waiter + 只唤醒热线程

```cpp
static inline bool high_concurrency_algorithm(tp_group_t *my_tp_group,
                                              tp_thread_t *my_thread_data) {
  if ((my_tp_group->waiting_thread != nullptr &&
       (!is_query_ready_to_process(my_tp_group)) && !is_tp_shutdown())) {
    return handle_worker_thread_sleep(my_tp_group, my_thread_data);
  }

  while (is_query_ready_to_process(my_tp_group) && (!is_tp_shutdown())) {
    if ((count_waiting_queries(my_tp_group) > 1) ||
        my_tp_group->waiting_thread == nullptr) {
      /*
        ... we will not wake up any "cold" thread (reserve threads),
        we only want to use "hot" threads (consumer threads).
        This is to avoid polluting the CPU caches, in medium concurrency
        it is important to ensure that there is a sufficient number of
        threads working to ensure that some of them get some progress

        At most we can have 3 threads working in parallel per thread group
        when 1 is the pool size per thread group. One thread is actively
        executing a query, one is a waiting thread and one is waiting on
        the consumer condition variable. In most cases 1-2 threads are
        executing per thread group though.
      */
      tp_wake_thread(my_tp_group, WAKE_IF_CONSUMER);
    }
    /*
      Check for high priority queries before processing the query to
      ensure that we don't lose performance at high concurrency.
    */
    if (my_tp_group->waiting_thread == nullptr &&
        !is_high_prio_query_available(my_tp_group)) {
      handle_waiting_thread(my_tp_group, my_thread_data, false);
      if (count_waiting_queries(my_tp_group) == 0) {
        continue;
      }
    }
    process_queued_query(my_tp_group, my_thread_data);
  }

  if (my_tp_group->waiting_thread == nullptr && !is_tp_shutdown()) {
    handle_waiting_thread(my_tp_group, my_thread_data, true);
  }
  return false;
}
```

与低并发对照：

| | 低并发 | 高并发（默认/推荐） |
|---|---|---|
| waiter | 无独立 waiter（执行线程兼任） | **有专职 waiting_thread** |
| 收事件时机 | 每执行完一个查询回头收一次 | waiter 常驻收，`epoll_wait` 期间不持锁 |
| 唤醒策略 | — | 只唤醒 `consumer`（热线程），不动 `reserve` |
| 并行上界 | — | 最多 3（1 执行 + 1 收事件 + 1 等 consumer），通常 1-2 |
| 适用场景 | 负载低、追求 cache 命中 | **高并发**（官方推荐） |

##### `query_worker_thread_loop`：带线程回收

```cpp
static inline bool query_worker_thread_loop(tp_group_t *my_tp_group,
                                            tp_thread_t *my_thread_data) {
  while (!is_tp_shutdown()) {
    if (is_query_ready_to_process(my_tp_group))
      process_queued_query(my_tp_group, my_thread_data);
    else {
      std::uint32_t txnlim = configured_max_transactions_limit_per_group(my_tp_group);
      if (txnlim != 0 && my_tp_group->query_threads_count > txnlim) return true;  // ★ 退出
      handle_worker_thread_sleep(my_tp_group, my_thread_data);
    }
  }
  return false;
}
```

★ 唯一的**带回收逻辑**的循环：当配置了 MTL 且本组 query 线程数已超过 `txnlim` 时，**该线程直接退出**（返回值 true 表示终止）。含义是"事务额度都不够用了，留着这些线程也没意义"——与 9.1 的 `become_reserve_thread` 返回 true 是同一套"返回值决定生死"的约定。

---

#### 9.10 ★ 影响范围：长查询/阻塞"拖死"的是整组，还是全池？

这是理解线程池故障传播的关键问题。答案是：**直接的停摆发生在本组，但会通过两条通道向全池扩散**。

##### 第一层：本组停摆（直接、确定）

先纠正一个容易说错的因果：**并不是"一个线程卡住 ⇒ 全组等待"**。

每组确实有自己的线程与队列（组内可能有不少空闲 worker 睡在 consumer/reserve 池里）。真正的原因是**线程池主动限制了每组"同时执行"的线程数**——这个限制就是 credit（6.1）：

```cpp
// 每组活跃线程配额 = 用户请求数 + 被判 stalled 的线程数
auto new_max_active_threads = threads_user_request + my_tp_group->stalled_threads;
```

`threads_user_request` 取自 `thread_pool_query_threads_per_group`（默认 2，官方设计意图甚至是"每组最多一个线程执行"以保 CPU cache 局部性）。于是：

| 情形 | 结果 |
|---|---|
| 1 个线程卡住，配额（2）还有余量 | 其他线程照常消费队列，**不会全组等待** |
| **配额内所有活跃线程都卡住** | 队列里的连接开始等待——**这才是 stall 检测要解决的直接问题** |
| 卡住的线程被判 stalled | 不计入配额（配额 = 2 + stalled 数）⇒ **放行额外线程**去消费队列 |

##### ★ 决定性环节：取任务的闸门是 `is_query_ready_to_process`

这一步是理解"为什么其他线程不来消费队列"的关键——**入队和出队是分离的**：

```cpp
/* 入队：queue_query 只管放进去，完全不检查 credit */
static inline void queue_query(tp_group_t *my_tp_group,
                               tp_client_low_level_t *client_cntx) {
  if (thd_is_transaction_active(client_cntx->thd) ||
      thread_pool_prio_kickup_timer == 0 ||
      is_high_priority_connection(client_cntx->thd)) {
    my_tp_group->queued_queries.push_back(client_cntx);
  } else {
    my_tp_group->queued_trans.push_back(client_cntx);
  }
  client_cntx->is_queued = true;
  client_cntx->conn_state = Connection_state::QUEUED;
}

/* 出队：取任务的唯一入口，队列非空只是必要条件，credit 才是充分条件 */
static inline bool is_query_ready_to_process(tp_group_t *my_tp_group) {
  tp_client_low_level_t *next = next_queued_query(my_tp_group);
  if (next == nullptr) return false;                    // 队列空
  if (is_credit_available(my_tp_group)) return true;    // 有 credit → 可处理
  next->conn_state = Connection_state::WAITING_FOR_CREDIT;   // ★ 无 credit
  return false;
}
```

而主循环的取任务循环就长在这上面：

```cpp
while (is_query_ready_to_process(my_tp_group) && (!is_tp_shutdown())) {
  if ((count_waiting_queries(my_tp_group) > 1) ||
      my_tp_group->waiting_thread == nullptr) {
    tp_wake_thread(my_tp_group, WAKE_IF_CONSUMER);      // ★ 只唤醒热线程
  }
  ...
  process_queued_query(my_tp_group, my_thread_data);
}
```

**所以"其他线程去消费队列"这件事，必须过 `is_query_ready_to_process` 这道闸门**：队列有任务只是让它去看一眼 credit，**credit 不够它就退出循环**（去当 listener 或去睡），队首被标记为 `WAITING_FOR_CREDIT` 继续排队。

这才是"拖死"的完整机制——**不是没有线程，也不是线程不愿意干活，而是 credit 这道闸门关着**。

##### 并发上界的准确说法

上面那段注释还给出了精确的上界（这是"每组最多 3 线程"的真正出处）：

> At most we can have **3 threads** working in parallel per thread group when 1 is the pool size per thread group. One thread is actively executing a query, one is a waiting thread and one is waiting on the consumer condition variable. **In most cases 1-2 threads are executing** per thread group though.

即：3 个并行 = 1 个执行 + 1 个收事件（waiting thread）+ 1 个在 consumer cond 上等；**实际执行查询的通常只有 1~2 个**。

另外 `tp_wake_thread(..., WAKE_IF_CONSUMER)` 只唤醒 **consumer（热线程）**、不唤醒 reserve（冷线程）——注释写明是为"避免污染 CPU cache"。所以即便队列积压，线程池也**优先复用刚干过活的线程**，而不是随便抓一个空闲线程。

##### 完整因果链

```
事件到达 → queue_query 入队（QUEUED）
   ↓
线程主循环 is_query_ready_to_process()
   ├─ 队列空        → 去睡 / 去当 listener
   ├─ credit 够     → process_queued_query 执行
   └─ credit 不够   → 队首标记 WAITING_FOR_CREDIT，队列继续堆积  ← "拖死"
   ↓
stall check 线程（1ms/10ms 一轮）
   → mark_and_count_stalled_threads()：把执行超 60ms 的线程标 stalled
   → update_max_active_threads()：配额 = threads_user_request + stalled
   ↓
credit 变大 → 闸门打开 → 队列被消费
```

所以准确的因果是：**"并发配额被占满"导致队列等待，而不是"一个线程卡住"**。更微妙的是，此时组内往往**有空闲线程，只是被 credit 挡着不让执行**——这正是"拖死"的含义：不是没线程可用，是线程池自己不让用。

范围严格限于本组：各组有独立的 `queued_queries` / `queued_trans` 与独立线程配额，**其他组的连接不受影响**。

##### 第二层：向全池扩散（间接、但有两条硬通道）

**通道一：线程总数上限是全局的**

```cpp
tp_thread_create_control_ctx.total_query_threads < (3 * tp_thread_create_control_ctx.vcpu_count)
```

`total_query_threads` 是**全局计数器**（9.5 已说明线程退出时递减）。也就是说：

```
组 A 有长查询 → stall → 建新线程 → total_query_threads++
组 B 有长查询 → stall → 建新线程 → total_query_threads++
...
多个组同时扩容 → 全局计数逼近 3×vCPU
   ⇒ 此后任何组想建线程都要走节流（按 ctrl_interval 限速）
      ⇒ 已经卡住的组更难恢复
```

**这是跨组传染的主通道**。两家的对应物是 `thread_pool_max_threads`（MariaDB 65536 / Percona 100000），同样是**全局**：`if (tp_stats.num_worker_threads >= threadpool_max_threads)` —— `num_worker_threads` 是全池统计。官方"池被完全阻塞"场景（见对比篇 11.6）正是这条通道走极端。

**通道二：锁是跨组的**

线程组是按连接 hash/round-robin 分的，但**锁不是按组分的**。一个卡住的查询若持有：

| 锁 | 等待者分布 |
|---|---|
| 全局读锁（FTWRL） | **所有组**的写连接 |
| MDL（表级） | 所有访问该表的连接，跨组 |
| 行锁 | 访问同一行的连接，跨组 |
| `GET_LOCK()` | 跨组 |

所以"一个卡住的查询"完全可以让**其他组**的连接进入等待——这不是线程池的组设计能隔离的，是锁的本质。对比篇 6.6 讲 Percona 的锁感知优先级，正是为缓解这一条：**让持锁者尽快跑完以缩短连锁等待**。

##### 第三层：MTL 是"组级隔离"，不是扩散

这一条容易搞反。源码注释写得很明确：

```
If the count is greater than or equal to MTL for the group so that all
available MTL for the group is consumed by long running transactions,
the MTL is suspended for the group.
```

**"MTL for the group" / "suspended for the group"**——MTL 的判定与挂起**都发生在组级别**（`configured_max_transactions_limit_per_group()` 返回本组限额，长事务计数也只扫 `my_tp_group->group_threads`）。

所以：一组被长事务打满 MTL ⇒ **只阻塞该组的新事务**，其他组不受 MTL 限制。这是**隔离**，不是传染。

> ⚠️ 注意区分：`thread_pool_max_transactions_limit` 这个名字里的"max"容易被读成"全池上限"，但生效单位（per_tg）是**每组**。配置时给的是总量语义，判定是分组的。

##### 小结

| 层面 | 范围 | 性质 |
|---|---|---|
| 组内队列等待 | 本组 | 直接停摆（stall 检测解决） |
| 线程总数配额 | **全池** | 间接扩散（3×vCPU / `max_threads` 被吃满） |
| 锁等待 | **跨组/全池** | 锁的本质，组设计无法隔离 |
| MTL 挂起 | 本组 | 组级隔离（不扩散） |

> 一句话：**stall 解决的是"组内停摆"，但线程池真正扛不住的故障，往往是"锁 + 全局线程配额"这两条跨组通道被打满**——这也是为什么官方说"即使把 `max_threads` 设得很大，全局锁场景仍可能阻塞整个池，必须有独立应急通道"（见对比篇 11.6）。

---

## ★ 本机制里的工程实现技法

### 一、高级语法技巧 / C++ 特性

| 技法 | 落点 |
|---|---|
| **`create_scope_guard` RAII** | 到处使用：`tp_process_event` 的 `err_grd` / `trxn_threads_decrement_guard`；`worker_main` 的 `thread_end_guard`；`create_thd_and_authenticate_conn` 的 `exit_auth` |
| **`std::chrono` 强类型时间** | `Musec`/`Misec`/`Cesec`/`Time_pt`（`thread_pool.h`）。`stall_limit` 是厘秒、`prio_kickup_timer` 是毫秒、MTL 是毫秒——用类型系统防止单位混用。`get_stall_limit_us()` 做显式转换 |
| **`std::atomic` + `memory_order_relaxed`** | `unsafe_atomic_add`（`thread_pool.h`）用于单写多读计数，注释称之为 "benign race"，避免 `fetch_add` 的总线锁开销 |
| **`alignas(128)`** | 全局变量隔离（`thread_pool.cc` 的 `tp_LOCK_stall_check`、`tp_COND_stall_check`、`tp_group_list`）防 false sharing |
| **`constexpr` / `inline constexpr`** | 所有常量与默认值（`thread_pool.h`） |
| **`I_P_List` 侵入式链表 + adapter** | 连接同时挂三条链（`methods.h`），靠 adapter 指定用哪对 `next/prev` 指针，零额外内存分配 |
| **`std::count_if` + lambda** | 长事务统计（`thread_pool.cc`） |
| **`std::for_each_n`** | 批量设置事件到达时间（`epoll.cc`） |
| **`enum class` + 字符串化函数** | `Thread_state`/`Connection_state`/`Worker_thread_type`/`Connection_type` 都配了 `*_str()`，便于 PFS 直接输出可读状态 |

### 二、经典算法的实现落地

| 算法 | 落点 |
|---|---|
| **self-pipe trick** | `tp_group_low_level_waiter_wake` 写 1 字节打断 `epoll_wait`（`epoll.cc`） |
| **`EPOLLONESHOT` 串行化** | 用内核机制替代 per-connection 锁（`epoll.cc`） |
| **批处理（batching）** | 高并发模式下只捞到 1 事件时非阻塞再捞一次（`epoll.cc`） |
| **LIFO 唤醒 / 热线程复用** | MySQL 侧用 consumer/reserve 双条件变量；MariaDB 侧用 `push_front` |
| **滞回控制** | MTL 挂起/恢复双阈值（`thread_pool.cc`） |
| **优先级老化（aging）** | `check_trans_queue_for_prio_kickups`，每组限流 100/秒 |
| **自适应采样周期** | stall checker 有动作睡 1ms、无动作睡 10ms |
| **连接亲和性** | `tp_process_event` 的 `do-while` 连读 |

### 三、复杂体系与设计模式的代码结构

**1. Bridge + Facade（插件 ABI 隔离）**

```
内核 Connection_handler (抽象)
        ▲
        │ 继承
Plugin_connection_handler  ──持有──►  Connection_handler_functions (函数指针表)
                                              │
                                              ▼
                                      插件实现 (thd_pool_add_connection / end)

内核 THD_event_functions (函数指针表) ──► 插件实现 (wait_begin / wait_end / post_kill)
```

内核通过两层间接（继承 + 函数指针）与插件解耦，插件通过 `thread_pool_priv.h`（`MYSQL_THD` 对 C 是 `void*`）反向调用内核。这就是 Pimpl/Facade 在二进制边界上的应用。

**2. 状态机（双套）**

`Thread_state`（12 态）+ `Connection_state`（7 态）各自独立演进，二者通过 `tp_process_event` 衔接。每个状态都有对应的字符串化函数供 PFS 输出。

**3. 角色标记替代锁**

`waiting_thread != nullptr` 充当互斥语义，使 `epoll_wait` 期间完全不持 `LOCK_group`（`thread_pool.cc`）。

**4. 对象池 + 槽位数组**

`group_threads[MAX_THREADS_PER_GROUP]` 是预分配的槽位数组，线程创建/退出时复用槽位（`update_thread_group_thread_exit`, 2160）。避免动态分配。

**5. PSI 插桩切换（Proxy 思路）**

每个连接独立插桩，attach/detach 时切换，解决"一个 OS 线程服务多连接"的统计串味问题。

---

## 可观测性

### 系统变量

⚠️ **勘误**：**13 个系统变量全部定义在 `thread_pool_plugin.cc:648-811`**，`thread_pool.cc` 里 grep `MYSQL_SYSVAR|MYSQL_THDVAR` **= 0 命中**。

```cpp
// thread_pool_plugin.cc:648
static MYSQL_SYSVAR_ULONG(
    size, thread_pool_size, PLUGIN_VAR_READONLY | PLUGIN_VAR_RQCMDARG,
    "How many thread groups we should create to handle query requests"
    " (minimum 1, maximum 512, all other values disables threadpool)",
    nullptr, nullptr,  // check, update
    // DEF_POOL_SIZE is only a placeholder. Default value for this
    // variable is set based on underlying hardware during plugin
    // initialization.
    DEF_POOL_SIZE, TP_MIN_POOL_SIZE, TP_MAX_POOL_SIZE, 1);

static MYSQL_SYSVAR_ULONG(
    algorithm, thread_pool_algorithm, PLUGIN_VAR_READONLY | PLUGIN_VAR_RQCMDARG,
    "0 = Low concurrency algorithm, 1 = High concurrency algorithm. "
    "Deprecated. Using the default value is recommended.",
    nullptr, nullptr, DEF_ALGORITHM, LOW_CONCURRENCY_ALGORITHM,
    HIGH_CONCURRENCY_ALGORITHM, 1);

/* This parameter is allowed to change in runtime */
static MYSQL_SYSVAR_ULONG(
    stall_limit, thread_pool_stall_limit, PLUGIN_VAR_RQCMDARG,
    "Number of 10ms interval passed before a query is declared as stalled",
    stall_limit_check, nullptr, DEF_STALL_LIMIT, TP_MIN_STALL_LIMIT,
    TP_MAX_STALL_LIMIT, 1);

/* This parameter is allowed to change in runtime */
static MYSQL_SYSVAR_ULONG(
    prio_kickup_timer, thread_pool_prio_kickup_timer, PLUGIN_VAR_RQCMDARG,
    "How many milliseconds before a queued transactions receives higher prio",
    prio_kickup_timer_check, nullptr, DEF_PRIO_KICKUP_TIMER,
    TP_MIN_PRIO_KICKUP_TIMER, TP_MAX_PRIO_KICKUP_TIMER, 1);

/* This parameter is allowed to change in runtime */
static MYSQL_SYSVAR_ULONG(
    max_unused_threads, thread_pool_max_unused_threads, PLUGIN_VAR_RQCMDARG,
    "Max number of unused threads per thread group, 0 means no limit",
    max_unused_threads_check, nullptr, DEF_MAX_UNUSED_THREADS,
    TP_MIN_MAX_UNUSED_THREADS, TP_MAX_MAX_UNUSED_THREADS, 1);

static MYSQL_THDVAR_ULONG(high_priority_connection, PLUGIN_VAR_RQCMDARG,
                          "Set priority of queries in this connection",
                          high_priority_connection_check, nullptr, 0UL, 0UL, 1UL, 1);
//  ★★ 唯一的会话级变量（THDVAR）

static MYSQL_SYSVAR_UINT(
    max_active_query_threads, thread_pool_max_active_query_threads,
    PLUGIN_VAR_RQCMDARG,
    "Maximum active query threads in a thread group. Deprecated. Use is "
    "dangerous and could lead to deadlocks.",
    check_max_active_query_threads, nullptr, DEF_MAX_ACTIVE_QUERY_THREADS,
    TP_MIN_MAX_ACTIVE_QUERY_THREADS, TP_MAX_MAX_ACTIVE_QUERY_THREADS, 1);
//  ★★ 变量描述里就写着 "Deprecated ... could lead to deadlocks"

static MYSQL_SYSVAR_UINT(
    max_transactions_limit, thread_pool_max_transactions_limit,
    PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_PERSIST_AS_READ_ONLY,
    "Maximum Transactions Allowed By Thread Pool for Processing",
    check_max_trxn_limits, update_max_trxn_limit, DEF_MAX_TRANSACTIONS_LIMIT,
    TP_MIN_MAX_TRANSACTIONS_LIMIT, TP_MAX_MAX_TRANSACTIONS_LIMIT, 1);

static MYSQL_SYSVAR_BOOL(dedicated_listeners, thread_pool_dedicated_listeners,
                         PLUGIN_VAR_READONLY | PLUGIN_VAR_RQCMDARG,
                         "Do not allow listener threads to execute queries. "
                         "Deprecated. Lowers performance and is not needed.",
                         nullptr, nullptr, DEF_DEDICATED_LISTENERS);

static MYSQL_SYSVAR_UINT(transaction_delay, thread_pool_transaction_delay,
                         PLUGIN_VAR_OPCMDARG,
                         "Time to sleep before starting to execute a new "
                         "transaction (in milliseconds).",
                         check_transaction_delay, update_transaction_delay,
                         DEF_TRANSACTION_DELAY, MIN_TRANSACTION_DELAY,
                         MAX_TRANSACTION_DELAY /*5 mins*/, 1);

static MYSQL_SYSVAR_UINT(query_threads_per_group,
                         thread_pool_query_threads_per_group,
                         PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_PERSIST_AS_READ_ONLY,
                         "Number of query worker threads per thread group.",
                         check_query_threads_per_group,
                         update_query_threads_per_group,
                         DEF_QUERY_THREADS_PER_GROUP,
                         MIN_QUERY_THREADS_PER_GROUP, MAX_QUERY_THREADS_PER_GROUP, 1);

static MYSQL_SYSVAR_UINT(
    connection_report_interval, sv_connection_report_interval,
    PLUGIN_VAR_OPCMDARG,
    "Time in seconds between each time the stall checker "
    "thread will emit a status message about connections managed"
    " by the thread pool. 0 means reporting is disabled.",
    /* check */ [](MYSQL_THD thd, ...) {
      if (thd_check_connection_admin_privilege(thd)) {
        my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "CONNECTION_ADMIN");
        return 1;
      }
      ...
    },
    /* update */ ..., DEF_CONNECTION_REPORT_INTERVAL,
    CONNECTION_REPORT_INTERVAL_OFF, MAX_CONNECTION_REPORT_INTERVAL, 1);

static MYSQL_SYSVAR_UINT(
    longrun_trx_limit, longrun_trx_limit_, PLUGIN_VAR_OPCMDARG,
    "Time a transaction can last until it is considered to be long-running "
    " (in milliseconds). The value is only used when "
    "thread_pool_max_transactions_limit > 0.",
    /* check */ ... [10, 86400000] ...,
    /* update */ ... ATMC_longrun_trx_limit.store(Misec(new_value), ...) ...,
    DEF_LONGRUN_TRX_LIMIT, MIN_LONGRUN_TRX_LIMIT, MAX_LONGRUN_TRX_LIMIT, 1);
```

```cpp
// thread_pool_plugin.cc:950 —— 注册表
static SYS_VAR *thread_pool_system_vars[] = {
    MYSQL_SYSVAR(size),
    MYSQL_SYSVAR(algorithm),
    MYSQL_SYSVAR(stall_limit),
    MYSQL_SYSVAR(prio_kickup_timer),
    MYSQL_SYSVAR(high_priority_connection),
    MYSQL_SYSVAR(max_unused_threads),
    MYSQL_SYSVAR(max_active_query_threads),
    MYSQL_SYSVAR(max_transactions_limit),
    MYSQL_SYSVAR(dedicated_listeners),
    MYSQL_SYSVAR(transaction_delay),
    MYSQL_SYSVAR(query_threads_per_group),
    MYSQL_SYSVAR(connection_report_interval),
    MYSQL_SYSVAR(longrun_trx_limit),
    nullptr};
```

**参数一览**：

| 变量（前缀 `thread_pool_`） | 范围 | 动态性 | 默认 | 说明 |
|---|---|---|---|---|
| `size` | 1..512 | **READONLY**（重启） | `clamp(vcpus/2,1,16)` | **线程组数**，不是线程数 |
| `algorithm` | 0..1 | READONLY | — | **已废弃** |
| `stall_limit` | — | **可动态改** | 6（=60ms） | 多少个 10ms 后判 stalled |
| `prio_kickup_timer` | — | **可动态改** | 1000ms | 低优先等待多久后升优先级；**0 = 取消低优先队列** |
| `high_priority_connection` | 0..1 | **可动态改（会话级）** | 0 | 本连接查询优先级 |
| `max_unused_threads` | 0..max | **可动态改** | — | 每组空闲线程上限，**0 = 不限** |
| `max_active_query_threads` | — | 可改 | — | **已废弃**（可能死锁） |
| `max_transactions_limit` | — | 需 CONNECTION_ADMIN | `clamp(32*size,1,512)` | 全局并发事务上限 |
| `dedicated_listeners` | bool | READONLY | false | **已废弃** |
| `transaction_delay` | 0..5min | 需 CONNECTION_ADMIN | 0 | 新事务前 sleep 毫秒 |
| `query_threads_per_group` | — | 需 CONNECTION_ADMIN | 2 | 每组 query worker 数 |
| `connection_report_interval` | 0..max | 需 CONNECTION_ADMIN | — | stall checker 打印连接报告间隔秒 |
| `longrun_trx_limit` | 10..86400000 ms | 需 CONNECTION_ADMIN | — | 事务多久算 long-running |

**状态变量**（`thread_pool_plugin.cc:966`）：

```cpp
static SHOW_VAR thread_pool_plugin_status_vars[] = {
    {"Thread_pool_query_threads",
     reinterpret_cast<char *>(&thread_pool_show_query_worker_threads),
     SHOW_FUNC, SHOW_SCOPE_GLOBAL},
    {"Deprecated_thread_pool_i_s_usage_count",
     reinterpret_cast<char *>(&deprecated_i_s_usage_count), SHOW_FUNC,
     SHOW_SCOPE_GLOBAL},
    //  ★★ 名字里就带 Deprecated：旧 I_S 三表已被 PFS 表取代
    {"option_tracker_usage:Enterprise Thread Pool",
     reinterpret_cast<char *>(&opt_option_tracker_usage_thread_pool_plugin),
     SHOW_LONGLONG, SHOW_SCOPE_GLOBAL},   // ★ 企业版"功能使用计量"埋点
    {nullptr, nullptr, SHOW_LONG, SHOW_SCOPE_GLOBAL}};
```

**★ 双表并存**（`thread_pool_plugin.cc:1112`）——一个 `mysql_declare_plugin` 声明 4 个插件：

```cpp
mysql_declare_plugin(thread_pool){
    MYSQL_DAEMON_PLUGIN,
    &thread_pool_plugin,
    "thread_pool",
    PLUGIN_AUTHOR_ORACLE,
    "Threads pool implementation to handle multiple connections for each thread",
    PLUGIN_LICENSE_PROPRIETARY,       // ★ 企业版专有许可
    thread_pool_plugin_init, thread_pool_plugin_check, thread_pool_plugin_deinit,
    0x0100,
    thread_pool_plugin_status_vars,
    thread_pool_system_vars,
    nullptr,
    PLUGIN_OPT_NO_INSTALL | PLUGIN_OPT_NO_UNINSTALL,   // ★★ 禁止运行时 INSTALL/UNINSTALL
},
    { MYSQL_INFORMATION_SCHEMA_PLUGIN, ..., "TP_THREAD_STATE", ... },
    { MYSQL_INFORMATION_SCHEMA_PLUGIN, ..., "TP_THREAD_GROUP_STATE", ... },
    { MYSQL_INFORMATION_SCHEMA_PLUGIN, ..., "TP_THREAD_GROUP_STATS", ... }
mysql_declare_plugin_end;
```

> ★ **两套观测口并存**：`mysql_declare_plugin` 里注册 3 个 **I_S** 插件（大写、`ST_FIELD_INFO`）；同时通过组件服务 `pfs_plugin_table_v1` 注册 4 张 **PFS** 表（小写 `tp_thread_state` / `tp_thread_group_state` / `tp_thread_group_stats` / `tp_connections`）。**PFS 版列数远多于 I_S 版**（如 `TP_THREAD_GROUP_STATE` 有 24 列 vs I_S 版 16 列），且 I_S 版已被标记 Deprecated。
>
> ⚠️ **`thread_handling` 需要重启的原因**（server 侧证据，`sql/sys_vars.cc:4047`）：
> ```cpp
> static Sys_var_enum Sys_thread_handling(
>     "thread_handling", "... one of one-thread-per-connection, no-threads, loaded-dynamically",
>     READ_ONLY GLOBAL_VAR(Connection_handler_manager::thread_handling),
>     CMD_LINE(REQUIRED_ARG), thread_handling_names, DEFAULT(0));
> ```
> ① 变量 `READ_ONLY`；② 真正的替换发生在插件 `init()` 调 `my_connection_handler_set()`，而插件带 `PLUGIN_OPT_NO_INSTALL|PLUGIN_OPT_NO_UNINSTALL`；③ `Connection_handler_manager::init()` 启动早期就按该变量选好 handler。
>
> ★ 线程池启用时 **Resource Group 被强制禁用**（`sql/mysqld.cc:6641`）：
> ```cpp
>   if (Connection_handler_manager::thread_handling !=
>           Connection_handler_manager::SCHEDULER_ONE_THREAD_PER_CONNECTION ||
>       plugin_is_ready(plugin_name, MYSQL_DAEMON_PLUGIN)) {
>     res_grp_mgr->disable_resource_group();
>     res_grp_mgr->set_unsupport_reason("Thread pool plugin enabled");
>   }
> ```

| 变量 | 默认 | 范围 | 动态 | 说明 |
|---|---|---|---|---|
| `thread_handling` | one-thread-per-connection | — | **否** | 插件加载成功后变 `loaded-dynamically` |
| `thread_pool_size` | **16** | 1–512 | **否** | 组数。InnoDB 为主：物理核数；MyISAM 为主：4–8 |
| `thread_pool_algorithm` | **1** | 0–1 | 只读 | 源码标 Deprecated |
| `thread_pool_stall_limit` | **6** | 4–600 | 是 | **单位 10ms** |
| `thread_pool_max_transactions_limit` | **0** | 0–1,000,000 | 是 | MTL。推荐 核数×32 |
| `thread_pool_prio_kickup_timer` | **1000** | 0–UINT32_MAX | 是 | 毫秒 |
| `thread_pool_query_threads_per_group` | **2** | 1–4096 | 是 | 官方推荐起始值 2 |
| `thread_pool_max_unused_threads` | **32** | 0–4096 | 是 | 26.7 由 2 改为 32 |
| `thread_pool_max_active_query_threads` | **0** | 0–512 | 是 | **"Use is dangerous and could lead to deadlocks"** |
| `thread_pool_dedicated_listeners` | **false** | bool | 只读 | Deprecated |
| `thread_pool_longrun_trx_limit` | **2000 ms** | — | 是 | 仅当 MTL>0 时生效 |
| `thread_pool_transaction_delay` | **0 ms** | 0–300000 | 是 | 新事务前 sleep |
| `thread_pool_high_priority_connection` | **0** | 0/1 | 会话级 | 该连接全进高优队列 |

### PFS 表（26.7 四张，随插件加载）

> ⚠️ 26.7 把监控表从 `INFORMATION_SCHEMA` 迁到 `performance_schema`；I_S 版本已 **deprecated**，未来会移除。

**`tp_thread_group_state`**（每组一行）：`CONSUMER_THREADS`、`RESERVE_THREADS`、`CONNECT_THREAD_COUNT`（每组最多 4）、`CONNECTION_COUNT`、`QUEUED_QUERIES`（高优）、`QUEUED_TRANSACTIONS`（低优）、`ACTIVE_THREAD_COUNT`、`STALLED_THREAD_COUNT`、`WAITING_THREAD_NUMBER`、`OLDEST_QUEUED`（毫秒）、`EFFECTIVE_MAX_TRANSACTIONS_LIMIT`、`THREADS_BOUND_TO_TRANSACTION`

**`tp_thread_group_stats`**（每组一行，22 列）：`QUERIES_EXECUTED`（**开始执行即计数**）、`QUERIES_QUEUED`（**不统计直执行**）、`PRIO_KICKUPS`、`STALLED_QUERIES_EXECUTED`、`BECOME_CONSUMER_THREAD`、`BECOME_RESERVE_THREAD`、`BECOME_WAITING_THREAD`、`WAKE_THREAD_STALL_CHECKER`，以及 **10 类等待计数**：`SLEEP/DISK_IO/ROW_LOCK/GLOBAL_LOCK/META_DATA_LOCK/TABLE_LOCK/USER_LOCK/BINLOG/GROUP_COMMIT/FSYNC_WAITS`

> Admin 组统计在**最后一行**（16 组时共 17 行）。

**`tp_thread_state`**（每线程一行）：`PROCESS_COUNT`（10ms 间隔序号）、`WAIT_TYPE`、`TP_THREAD_TYPE`（四类）、`TIME_OF_ATTACH`、`MARKED_STALLED`、`STATE`

**`tp_connections`**（26.7 新增，每连接一行）：`STATE`、`TYPE`（User/Admin_interface/Admin_privilege）、**`DIRECT_QUERY_EVENTS` / `QUEUED_QUERY_EVENTS`**、`MANAGEMENT_TIME`

### 观测对象 → 手段 速查

| 想看什么 | 手段 |
|---|---|
| **线程池是否在跟负载对抗** | `SELECT SUM(STALLED_QUERIES_EXECUTED)/NULLIF(SUM(QUERIES_EXECUTED),0) FROM tp_thread_group_stats;`（官方推荐，越低越好） |
| **队列积压 & 谁在堵** | `tp_thread_group_state` 的 `QUEUED_QUERIES` + `OLDEST_QUEUED` + `WAITING_THREAD_NUMBER`（长期 NULL = 没人收事件，危险） |
| **直执行 vs 入队比例** | `SELECT SUM(DIRECT_QUERY_EVENTS), SUM(QUEUED_QUERY_EVENTS) FROM tp_connections;`（26.7 独有） |
| **瓶颈是行锁 / IO / binlog？** | `tp_thread_group_stats` 的 10 类 `*_WAITS` 分布 |
| **优先级是否失效** | `PRIO_KICKUPS` 增长过快 → 调大 `prio_kickup_timer`；另检查 `autocommit` 是否开启 |
| **MTL 是否成为瓶颈** | `THREADS_BOUND_TO_TRANSACTION` vs `EFFECTIVE_MAX_TRANSACTIONS_LIMIT`；连接处于 `Waiting for Credit` |
| **插件是否加载** | `SELECT PLUGIN_NAME, PLUGIN_STATUS FROM information_schema.PLUGINS WHERE PLUGIN_NAME LIKE 'thread_pool%';` |
| **PFS instruments** | `SELECT * FROM performance_schema.setup_instruments WHERE NAME LIKE '%thread_pool%';` |

---

## 运维实践

### 启用

```sql
INSTALL PLUGIN thread_pool SONAME 'thread_pool.so';
SELECT PLUGIN_NAME, PLUGIN_STATUS FROM information_schema.PLUGINS
 WHERE PLUGIN_NAME LIKE 'thread_pool%' OR PLUGIN_NAME LIKE 'tp_%';
```

```ini
[mysqld]
plugin-load-add  = thread_pool.so
thread_handling  = pool-of-threads
thread_pool_size = 16                          # = 物理核数
thread_pool_max_transactions_limit = 512       # ≈ 核数 × 32
thread_pool_stall_limit = 6                    # 60ms
thread_pool_algorithm = 1
thread_pool_query_threads_per_group = 2
```

> `thread_handling` **非动态**，改它必须重启。先在主从的从库验证；回滚只需注释掉配置重启。

### 调优原则

> 以下取值建议均出自官方 26.7 Reference Manual「Thread Pool Tuning」，非经验猜测；"依据"一列是官方给出的理由。

#### 参数取值建议（官方）

| 参数 | 官方建议 | 依据 / 注意 |
|---|---|---|
| `thread_pool_size` | **InnoDB**：物理核数（上限 512）<br>**MyISAM**：4~8 | MyISAM 取高值有"轻微但非剧烈"的负面影响（MyISAM 的表锁使并行收益有限） |
| `thread_pool_max_transactions_limit` | 初始 = **物理核数 × 32**；上界参考 `Max_used_connections` | 官方方法：**从该值开始向下调**并观察吞吐；上界是"期望的最大并发连接数" |
| `thread_pool_query_threads_per_group` | **起始 2** | 低于 2 通常**无收益**；此值 × `thread_pool_size` ≈ 总查询线程数；值大 ⇒ 长查询占满组内所有线程的可能性降低。但 `thread_pool_size` 小 + 此值大 ⇒ 轮询开销上升 |
| `thread_pool_algorithm` | **1**（高并发） | 官方："for best performance under normal conditions" |
| `thread_pool_stall_limit` | 默认 6（60ms）适合**极简语句**；若 99.9% 语句在 100ms 内完成 ⇒ 设 **10** | 单位 10ms，范围 1~600（最大 6 秒） |

#### ★ stall_limit 的双向权衡（官方表述）

官方原话把这个参数的取舍讲得很清楚：

> Short wait values permit threads to start more quickly. **Short values are also better for avoiding deadlock situations.** Long wait values are useful for workloads that include long-running statements, to avoid starting too many new statements while the current ones execute.

- **调低** ⇒ 更多查询被判 stalled ⇒ 更多线程并行 ⇒ 快的查询更快启动，且**更不容易死锁**；代价是额外上下文切换与线程创建开销
- **调高** ⇒ stall 比例下降（官方明说 *"To decrease the likelihood of statements stalling, increase the value"*）；代价是长查询阻塞同组短查询更久

官方给的 stall 比例查询（这个数应尽量低）：

```sql
SELECT SUM(STALLED_QUERIES_EXECUTED) / SUM(QUERIES_EXECUTED)
FROM performance_schema.tp_thread_group_stats;
```

#### ★ 最坏延迟 222 秒：官方给出的推演

官方用一个具体场景说明"排队最坏能排多久"：

| 条件 | 值 |
|---|---|
| 低优队列 | 200 条 |
| 高优队列 | 10 条 |
| `thread_pool_prio_kickup_timer` | 10000（10 秒） |
| `thread_pool_stall_limit` | 100（1 秒） |

推演：新语句要等 10 秒才**有资格**被提升到高优队列；但提升前必须把它前面的都移过去，而**每组每 10ms 最多移 1 条**（100 条/秒）⇒ 再花 2 秒；到了高优队列前面还有一堆长查询，每条都要 1 秒 stall ⇒ **最坏 222 秒后才开始执行**。

> 官方的应对建议很实在：**如果应用对响应时间要求高，应该在应用层自己限流**；否则就用线程池参数设一个"最大等待时间"。——这是"线程池不能替代应用层限流"的官方立场。

#### ★ 特权连接：MTL 打满时的唯一入口

这是运维上最需要预先准备的应急手段（对应已知限制中"MTL 打满会 hang"）：

- `thread_pool_max_transactions_limit` 打满后，**新连接/新事务会 hang**，且调 `thread_pool_longrun_trx_limit` 也救不回来
- 此时唯一入口是**特权连接**：需要 `TP_CONNECTION_ADMIN` 权限（**默认不授予任何用户，必须显式 GRANT**）
- 特权连接**忽略** MTL 限制，可用来调大/取消限制，或 kill 掉长事务
- 它被分配到 **Admin 线程组**，且**不受** `thread_pool_transaction_delay` 影响

> **运维建议**：在生产启用线程池**之前**就把 `TP_CONNECTION_ADMIN` 授予 DBA 账号——等到 MTL 打满时再授权已经连不上了（除非重启）。

#### 其余要点

1. **`thread_pool_size` 与 `thread_pool_max_transactions_limit` 成对调**——前者是并行度骨架，后者是全局事务配额。只加大前者而后者卡着 → "组允许干活但被配额限流"，吞吐上不去
2. **注意 autocommit**——默认开启时**所有语句都是低优先级**（每条语句自成一个事务），优先级机制失效
3. **混合引擎的优先级倾斜**——autocommit 关闭时，InnoDB 语句优先于 MyISAM（因为 MyISAM 非事务，其语句恒为低优）
4. **`thread_pool_transaction_delay`** 可用来缓解并行事务对 index creation / online buffer pool resize 的资源争用；**Admin 组豁免**
5. **线程总数理论上可达 `max_connections + thread_pool_size`**（所有连接都在执行 + 每组额外一个监听线程）

### 适用场景

**适合**：连接数远超 CPU 核数、查询短且 CPU 密集的 OLTP（PHP 短连接、微服务扇出）。若能复现"吞吐随并发上升到峰值后回落"的曲线，线程池能抹平那个回落。

**不适合**：

| 场景 | 原因 |
|---|---|
| 突发负载 | 长时间非活跃 + 偶尔极活跃，延迟容忍度低，节流效果不理想 |
| 高并发长查询 / 数据仓库 | 工作线程堆积达上限后**完全阻止后续语句** |
| 严重锁冲突 | 等锁线程数超过总线程数会堆积（典型 `FLUSH TABLES WITH READ LOCK`） |
| 极高并发 Prepared Statement（非 Java） | Binary Protocol 增加网络来回，给 epoll 监听进程压力 |

### 已知限制

1. `thread_pool_size` 运行时不可改（只读，仅启动时设置）
2. 每组线程上限 **4096**（部分系统 4095）；线程总数理论上可达 `max_connections + thread_pool_size`
3. **每组最多 4 个连接处理线程**，空闲会过期
4. **autocommit=1 时所有语句都是低优先级**
5. **未插桩代码的阻塞不会被上报**，只能靠 stall 超时兜底；stall 设置过高有死锁风险
6. **binlog dump 线程**等"永久运行语句"必须靠 stalled 分类避免阻塞组
7. `thread_pool_max_transactions_limit` 打满时新连接/新事务会 **hang**；`thread_pool_longrun_trx_limit` 也救不回来时只能靠 `TP_CONNECTION_ADMIN` 特权连接
8. `thread_pool_max_active_query_threads`、`thread_pool_dedicated_listeners`、`thread_pool_algorithm` 源码注释均标 **Deprecated**
9. `INFORMATION_SCHEMA` 版 TP 表已废弃，未来会移除
10. PFS 归属规则：执行用户语句时计入用户连接，否则计入线程池
11. PFS 表随插件加载/卸载，**不允许 `TRUNCATE`**；行是某一时刻的**快照**

---

### 诊断手册：用四张 PFS 表判断线程池是否成为瓶颈

> 本节给的是**判据与查询方法**，不是实测阈值——具体数值必须在你的负载下压测得出。四张表都是**快照**且**不允许 `TRUNCATE`**（见已知限制），所以要看**变化量**：间隔取样两次算差值。

#### 判据一：有没有在排队（最直接的指标）

```sql
SELECT TP_GROUP_ID, QUEUED_QUERIES, QUEUED_TRANSACTIONS,
       OLDEST_QUEUED, THREAD_COUNT, ACTIVE_THREAD_COUNT
FROM performance_schema.tp_thread_group_state
ORDER BY QUEUED_QUERIES DESC;
```

- `QUEUED_QUERIES` = 高优队列长度，`QUEUED_TRANSACTIONS` = 低优队列长度（**注意表名是 `QUEUED_TRANSACTIONS`**，内部字段才叫 `queued_trans`）
- ★ **`OLDEST_QUEUED` 是最有价值的一列**：最老的那条排队查询已经等了多久。它持续非零且与你的延迟目标同量级 ⇒ 线程池已成为瓶颈
- `ACTIVE_THREAD_COUNT` 长期等于 `THREAD_COUNT` ⇒ 线程已用满

#### 判据二：排队是因为长查询，还是线程/额度不够

```sql
SELECT TP_GROUP_ID, QUERIES_EXECUTED, QUERIES_QUEUED,
       STALLED_QUERIES_EXECUTED,
       ROUND(STALLED_QUERIES_EXECUTED * 100.0 / NULLIF(QUERIES_EXECUTED, 0), 2) AS stall_pct
FROM performance_schema.tp_thread_group_stats
ORDER BY stall_pct DESC;
```

- **stall 比例高** ⇒ 大量查询超过 `thread_pool_stall_limit` 被判为长查询 ⇒ 调高 `thread_pool_size`，或先优化慢查询
- **stall 比例低但 `QUERIES_QUEUED` 高** ⇒ 连短查询都排不上 ⇒ 线程数或事务额度不足

#### 判据三：线程究竟在等什么（`*_WAITS` 系列）

`tp_thread_group_stats` 末尾那组 `*_WAITS` 列是全表最有价值的部分——它把等待**按类型分开**统计：

| 列 | 含义 | 说明 |
|---|---|---|
| `ROW_LOCK_WAITS` | 等行锁 | 高 ⇒ 锁冲突，非线程池问题 |
| `META_DATA_LOCK_WAITS` | 等 MDL | 高 ⇒ DDL 阻塞 |
| `TABLE_LOCK_WAITS` / `GLOBAL_LOCK_WAITS` | 等表锁 / 全局锁 | 后者常是 FTWRL |
| `DISK_IO_WAITS` / `FSYNC_WAITS` | 等磁盘 IO / fsync | 高 ⇒ IO 瓶颈 |
| `BINLOG_WAITS` / `GROUP_COMMIT_WAITS` | 等 binlog / 组提交 | 高 ⇒ 组提交成为串行点 |
| `USER_LOCK_WAITS` | 等 `GET_LOCK()` | 用户锁 |
| `SLEEP_WAITS` | 主动 sleep | 如 `SELECT SLEEP()` |

```sql
SELECT SUM(ROW_LOCK_WAITS) row_lock, SUM(META_DATA_LOCK_WAITS) mdl,
       SUM(DISK_IO_WAITS) io, SUM(GROUP_COMMIT_WAITS) gc, SUM(BINLOG_WAITS) binlog
FROM performance_schema.tp_thread_group_stats;
```

**判读要点**：若某一类 WAITS 占绝对多数，**瓶颈通常不在线程池本身**，而在对应子系统。这与已知限制（严重锁冲突会让等锁线程堆积）直接对应——这种情况下调线程池参数是治标。

#### 判据四：MTL 是否被长事务挂起

```sql
SELECT TP_GROUP_ID, EFFECTIVE_MAX_TRANSACTIONS_LIMIT,
       THREADS_BOUND_TO_TRANSACTION, TIME_OF_LAST_LONGRUNNING_TRXS_CHECK
FROM performance_schema.tp_thread_group_state;
```

- `EFFECTIVE_MAX_TRANSACTIONS_LIMIT` 是**当前生效**的 MTL——长事务背压时会临时失效（恢复后回来）
- 配合 error log 的 `ER_THREAD_POOL_MTL_DISABLE` / `..._MTL_REENABLE` 判断是否被挂起
- `THREADS_BOUND_TO_TRANSACTION` 接近上限且 `QUEUED_TRANSACTIONS` 持续增长 ⇒ 事务额度打满（已知限制第 7 条，会 hang）

#### 判据五：冷热线程与单线程视角

```sql
-- 冷热线程迁移与线程创建
SELECT SUM(BECOME_CONSUMER_THREAD) hot, SUM(BECOME_RESERVE_THREAD) cold,
       SUM(BECOME_WAITING_THREAD) waiter, SUM(THREADS_STARTED) started,
       SUM(PRIO_KICKUPS) kickups
FROM performance_schema.tp_thread_group_stats;

-- 哪些线程被标记为 stalled
SELECT TP_GROUP_ID, TP_THREAD_NUMBER, TP_THREAD_TYPE, WAIT_TYPE,
       MARKED_STALLED, TIME_OF_ATTACH
FROM performance_schema.tp_thread_state
WHERE MARKED_STALLED = 1;
```

`THREADS_STARTED` 持续增长 ⇒ 线程在反复创建销毁，可考虑调大 `thread_pool_max_unused_threads`（26.7 默认值已从 2 提到 32）。`PRIO_KICKUPS` 反映低优查询被踢升高优的频率。

#### 取样方法（表不允许 TRUNCATE，只能自己存快照）

```sql
CREATE TABLE tp_snap_1 AS SELECT * FROM performance_schema.tp_thread_group_stats;
-- 间隔 N 秒
CREATE TABLE tp_snap_2 AS SELECT * FROM performance_schema.tp_thread_group_stats;
SELECT a.TP_GROUP_ID,
       b.QUERIES_EXECUTED - a.QUERIES_EXECUTED           AS q_exec,
       b.QUERIES_QUEUED   - a.QUERIES_QUEUED             AS q_queued,
       b.STALLED_QUERIES_EXECUTED - a.STALLED_QUERIES_EXECUTED AS stalled,
       b.ROW_LOCK_WAITS   - a.ROW_LOCK_WAITS             AS row_lock_w
FROM tp_snap_1 a JOIN tp_snap_2 b USING (TP_GROUP_ID);
```

> **Admin 组提醒**：`TP_GROUP_ID` 最大的那一行是 **admin 线程组**（16 组时共 17 行）。统计普通组时要排除它，否则会被管理连接的数字干扰。

---

## Misc

### 核心术语表（线程池的"黑话"）

> 线程池有大量自造/借用的术语，且**同名不同义**的不少（如 `prio_kickup_timer` 在 MySQL 与 MariaDB 里作用点不同）。本表按"一句话本质 + 源码标识"列出，便于对照阅读。

#### credit（额度）——最核心的一个

**一句话：credit 就是"并发预算"——本组现在还能再放行几个查询去执行。**

| | 说明 |
|---|---|
| 字面 | 信用额度（credit） |
| 本质 | 允许**同时执行**的名额数，不是线程数 |
| 判定 | `is_credit_available()` 返回 true 才能从队列取任务执行（见 9.9） |
| 消耗 | 线程开始执行 → `inc_active_threads()`（`threads_active++`） |
| 归还 | 执行结束，或引擎上报阻塞（`wait_begin`）→ `dec_active_threads()` |
| 追加 | 线程被判 `stalled` → 不计入配额 → `max_active_threads = threads_user_request + stalled_threads` |

**为什么叫"额度"而不是"并发上限"**：因为它是**可借可还**的——

```
初始额度 = threads_user_request（默认 2）
执行一个查询  → 借走 1
引擎上报阻塞  → 归还 1        （线程睡着，不占额度）
查询结束      → 归还 1
判定 stalled  → 追加 1        （长查询获得"额外名额"）
```

这正是 6.2 节"stalled 线程不计入配额"的含义：**给长查询追加额度，让它继续跑，同时不挤占短查询的预算**。

三档判定（6.1）：

```cpp
if (txnlim != 0)              // ① MTL：按事务数
  res = (trxn_threads <= txnlim || is_mtl_suspended(...));
else if (max_active_query_threads)   // ② 固定活跃线程数
  res = (threads_active < max_active_query_threads);
else                          // ③ 默认：动态配额
  res = (threads_active <= max_active_threads);
```

#### MySQL 26.7 术语

| 术语 | 本质 | 源码标识 |
|---|---|---|
| **credit** | 并发预算（见上） | `is_credit_available` |
| **stall / stalled** | 执行超过 `stall_limit`（60ms）的线程，被判"卡住" | `mark_and_count_stalled_threads` |
| **MTL** | max transactions limit，**每组**并发事务上限 | `configured_max_transactions_limit_per_group` |
| **longrun_trx_limit** | 长事务判定阈值（默认 2000ms），驱动 MTL 背压 | `ATMC_longrun_trx_limit` |
| **consumer / reserve** | 热 / 冷线程，分别睡在 `COND_consumer` / `COND_reserve` | `become_consumer_thread` |
| **waiting_thread** | 正在收事件的线程；这个"角色"本身当锁用 | `my_tp_group->waiting_thread` |
| **listener** | 监听线程（收 epoll 事件） | `dedicated_listener_loop` |
| **kickup** | 低优等待超时后提升到高优队列 | `check_trans_queue_for_prio_kickups` |
| **admin thread group** | 第 N+1 组，特权连接专用，豁免 MTL | `admin_thread_group` |
| **TP_CONNECTION_ADMIN** | 动态权限，池打满时的唯一入口（默认不授予） | `register_privilege` |
| **THR_PSI_backup** | 一个 OS 线程的两套 PFS 身份切换（双插桩） | `thread_local PSI_thread *` |
| **thread group** | 线程组：独立队列 + 独立线程 + 独立额度 | `tp_group_t` |
| **queued_queries** | ★ **高优**队列（名字反直觉） | `tp_group_t::queued_queries` |
| **queued_trans** | ★ **低优**队列（名字反直觉） | `tp_group_t::queued_trans` |

#### MariaDB / Percona 术语（对比篇）

| 术语 | 本质 |
|---|---|
| **oversubscribe** | 超订：一组允许同时活跃的 worker 数（默认 3）；两家判定不同（6 章） |
| **tick_interval** | timer 线程周期 = `stall_limit`（耦合，500ms） |
| **timer_thread** | 全局定时器线程（MySQL 侧叫 `tp_stall_check`） |
| **io_poll** | 多平台事件封装（epoll / kqueue / event ports / IOCP） |
| **change_group** | 组数变化时迁移连接 |
| **tickets**（Percona） | 高优票据，用完进低优并重置 |
| **high_prio_mode**（Percona） | transactions / statements / none |
| **priority**（MariaDB） | auto / high / low |
| **max_threads** | 全池线程上限（MariaDB 65536 / Percona 100000） |
| **queue_event_count** | 队列出队计数，timer 每轮重置——stall 判定的依据 |

#### 最容易混淆的三组

| 混淆点 | 澄清 |
|---|---|
| **credit vs 线程数** | credit 限的是"**同时执行**"，不是"线程总数"。组里线程可以很多（最多 4096），但同时执行的受 credit 限制 |
| **stall(MySQL) vs stall(两家)** | MySQL 判"**单线程**跑太久"；两家判"**组**不排水了" |
| **`prio_kickup_timer`** | MySQL 用它限速踢升（每 10ms 一条）；MariaDB 同名参数，但踢升批量执行（无 10ms 限速） |

---

### 易混淆概念

| 概念 | 澄清 |
|---|---|
| **线程池 ≠ 连接池** | 连接池限制"建立多少条连接"（客户端/ProxySQL）；线程池限制"同时推进多少条语句"（服务端） |
| **`queued_trans` 是低优队列** | 命名反直觉：它装的是**没在事务里**的普通查询；`queued_queries` 才是高优 |
| **`stall_limit` 单位是 10ms** | MySQL/Percona 是 10ms 单位（6=60ms）；**MariaDB 是 1ms**（500=500ms）。跨库迁移时极易搞错 |
| **autocommit 会让优先级失效** | 官方明确：autocommit=1 时所有语句都是低优先级 |
| **MTL ≠ oversubscribe**（MySQL **没有** oversubscribe） | MTL 限**事务数**（MySQL 26.7）；`thread_pool_oversubscribe` 是 **MariaDB / Percona** 的参数（两家默认均为 3，范围 1~1000），且两家语义不同（见对比篇）。★ **MySQL 源码 grep `oversubscribe` 为 0 命中**（26.7 插件源码与 8.0.39 均无此符号）——MySQL 侧承担类似职责的是 `thread_pool_query_threads_per_group`（源码 `DEF_QUERY_THREADS_PER_GROUP = 2`）与 `thread_pool_max_active_query_threads`（已废弃） |
| **`thread_pool_query_threads_per_group` 默认值：官方文档自相矛盾** | 官方 Operation 章节写 "the default is a single query thread"（默认 1），但源码 `thread_pool.h` 的 `DEF_QUERY_THREADS_PER_GROUP = 2`，且 Tuning 章节建议"起始值 2"。**以源码为准 = 2**（官方文档陷阱，见规范⑩） |
| **线程池不是万能的** | 长查询/数据仓库/严重锁冲突场景下反而更差 |

### ★ 本篇核实出的"流传但不存在/已变"的说法

1. **"26.7 的 FAQ 说线程池仍属企业版"** —— A.15 FAQ 原文确实还写着 "included in MySQL Enterprise Edition"，但这是**未更新的旧文案**。以 Release Notes（WL #17296）为准：社区版可用。
2. **"`thread_pool_query_threads_per_group` 默认是 1"** —— Operation 页变量清单这么写，但 Tuning 页推荐"starting value of 2"、源码 `DEF_QUERY_THREADS_PER_GROUP = 2`。**以 2 为准**。
3. **"官方文档讲了 thread group / listener / worker / timer 四类线程"** —— 26.7 的 Elements 页**不讲**这些，只讲插件库/系统变量/PFS instruments。四类线程只在 `tp_thread_state.TP_THREAD_TYPE` 中被正式命名。
4. **"0/1 两种算法有官方机制描述"** —— 官方 Operation/Tuning 页**都没有**，只有"推荐设为 1"。源码注释标 Deprecated。
5. **"线程池进了 LTS"** —— 26.7 是 **Innovation** 版；9.7.2 / 8.4.11 维护版只含缺陷与安全修复，**线程池尚未进 LTS**。

---

## 参考

### 官方文档

1. [MySQL 26.7 Release Notes - Changes in MySQL 26.7.0](https://dev.mysql.com/doc/relnotes/mysql/26.7/en/news-26-7-0.html)
2. [MySQL 26.7 Reference Manual - 7.6.3 MySQL Thread Pool](https://dev.mysql.com/doc/refman/26.7/en/thread-pool.html)
3. [7.6.3.1 Elements](https://dev.mysql.com/doc/refman/26.7/en/thread-pool-elements.html) · [7.6.3.2 Installation](https://dev.mysql.com/doc/refman/26.7/en/thread-pool-installation.html) · [7.6.3.3 Operation](https://dev.mysql.com/doc/refman/26.7/en/thread-pool-operation.html) · [7.6.3.4 Tuning](https://dev.mysql.com/doc/refman/26.7/en/thread-pool-tuning.html)
4. [29.12.16 Performance Schema Thread Pool Tables](https://dev.mysql.com/doc/refman/26.7/en/performance-schema-thread-pool-tables.html)
5. [MySQL 26.7 FAQ: Thread Pool](https://dev.mysql.com/doc/refman/26.7/en/faqs-thread-pool.html)

### 相关篇目

- [`connection_threading.md`](connection_threading.md) —— Per_thread 连接模型（本篇的对立面）
- [`thread_pool_mariadb_percona.md`](thread_pool_mariadb_percona.md) —— MariaDB / Percona 源码级剖析与三方对比
- [`pfs.md`](../observability/pfs/pfs.md) / [`pfs_statement.md`](../observability/pfs/pfs_statement.md) —— PFS 插桩机制
- [`../../infra/threading_architecture.md`](threading_architecture.md) —— 线程架构总览
