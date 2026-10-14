# InnoDB 同步原语族（os_event / wait array / PolicyMutex / rw_lock_t）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 InnoDB 自研同步原语的四层谱系：`os_event`（manual-reset 事件 + signal_count 防丢信号）、sync0arr 等待数组（event 已嵌入被等对象，数组只剩诊断职责）、`ib0mutex.h` 的 PolicyMutex 模板族（TTAS 自旋 + event/futex 两种慢路径）、策略层（统计/锁序校验的零成本注入）、以及 `rw_lock_t` 的 **lock_word 单字三态编码**（S/SX/X + 递归）。
>
> **边界**：本篇讲 InnoDB 原语的**类型与机制**；全局 latch 实例清单见 [`../README.md`](../README.md) A3；server 侧原语（mysys 三层封装）见 [`mysys_primitives.md`](mysys_primitives.md)；两个著名用户——B-tree 索引锁与页 latch 协议——见 [`../../index/btr.md`](../../../index/btr.md) 与 [`../../innodb/buffer_pool.md`](../../../innodb/buffer_pool.md)。

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

InnoDB 在仓库里**自带一整套与 server 侧 mysys 平行的同步原语**：

```
L0  OS 原语 / 标准库
      futex          —— TTASFutexMutex 直接裸 syscall(SYS_futex, ...)，无封装层
      pthread         —— pthread_cond_t（os_event 的唯一底层）、pthread_mutex_t（OSTrackMutex）
                          （Windows 用 CONDITION_VARIABLE / CRITICAL_SECTION）
      std::atomic     —— 锁字（m_lock_word / m_owner / lock_word）全部用它
      std::condition_variable / std::thread —— os_event 与等待线程侧
      ⚠️ sem_t 在 8.0.39 的 InnoDB 里已完全不用（旧版本遗留）
                  │
L1  底层等待     os_event（manual-reset 事件，包 pthread_cond + signal_count）
                  sync0arr（等待数组：诊断 + 防丢信号兜底 + debug 死锁检测）
                  │
L2  实现模板      TTASEventMutex / TTASFutexMutex（TTAS 自旋 + 两种慢路径）      ib0mutex.h
                  rw_lock_t（S / SX / X 三态读写锁 + 递归）                     sync0rw.h
                  │
L3  策略层        NoPolicy / GenericPolicy / BlockMutexPolicy / MutexDebug      sync0policy.h
                  │   （统计与锁序校验经模板参数零成本注入）
L4  实例          dict_operation_lock、index->lock、buf_pool 各 mutex ...        （A3 清单）
                  │
【上层用户】B-tree 索引锁、buf 页 latch、lock_sys 分片锁、log/redo、trx_sys、dict ...（见「模块闭环」）
```

> **勘误**：有些资料（包括本目录 README 的旧版谱系树）把 InnoDB 的 L1 写成 "`FutexWait` / `os_event` / `sync0arr`"，并把 `sem_t` 列为 L0 成员。**8.0.39 里 `FutexWait` 与 `ut0futex.h` 都不存在**，futex 由 `TTASFutexMutex` 直接裸调 `syscall`；`sem_t` 也已从 InnoDB 消失。看到这种说法请以此处为准。

### 用途

一句话：**server 侧把 pthread 包一层；InnoDB 把 pthread 藏在最底下，往上自己造**。原因（`sync0arr_impl.h` 头注释的历史决策 + 设计决策点汇总）：

1. pthread_mutex 的慢路径语义不透明，无法回答"**谁持锁、谁在哪自旋、等多久了**"——InnoDB 需要完整诊断能力（wait array、long-wait 报警、死锁检测）；
2. 临界区通常极短（几十条指令），需要 **spin → yield → 睡眠** 的两级等待，且自旋参数可调（`srv_n_spin_wait_rounds`/`srv_spin_wait_delay`）；
3. B-tree/字典操作需要 **SX（shared-exclusive）中间态**——pthread_rwlock 给不了。

### 版本演进

| 版本 | 变化 |
|---|---|
| 4.x–5.0 | 每把 mutex 配一个 OS event；5.0.30 起改为 **event 嵌入被等对象**，wait array 降级为"诊断 + 防丢信号"（头注释原文记录了这次重设计，见「sync0arr」节） |
| 5.6 | `rw_lock_t` 引入 SX 三态（`X_LOCK_HALF_DECR` 编码） |
| 5.7+ | mutex 模板化（`ib0mutex.h`），`MUTEX_FUTEX`/`MUTEX_EVENT`/`MUTEX_SYS` 三种后端按平台选择 |
| 8.0 | `sync_wait_array` 变为多实例数组；debug 校验体系重组为 `LatchDebug`；新增 `ut::Stateful_latching_rules` 声明式状态机校验；SEMAPHORES 段的自旋统计**废弃**（打印固定 0，让位 PFS） |

---

## 理论基础

### 设计思想与权衡

#### 一、两级等待：自旋多久才值得睡

**自旋本体 `ut_delay()`**（⚠️ 真身在 `ut/ut0ut.cc:99`，**不存在 `ut0delay.h`**）：

```cpp
ulint ut_delay(ulint delay) {
  ulint i, j;
  /* We don't expect overflow here, as ut::spin_wait_pause_multiplier is limited
  to 100, and values of delay are not larger than @@innodb_spin_wait_delay
  which is limited by 1 000. ... */
  const ulint iterations = delay * ut::spin_wait_pause_multiplier;
  //                       ★★ 实际 PAUSE 条数 = delay × 倍数，不是 delay × 常数

  UT_LOW_PRIORITY_CPU();                       // POWER 上发 "or 1,1,1" 降优先级
  j = 0;
  for (i = 0; i < iterations; i++) {
    j += i;                                    // ★ 无意义算术，唯一目的是阻止编译器消除空循环
    UT_RELAX_CPU();                            // ★ 每轮发射一条 PAUSE / ISB
  }
  UT_RESUME_PRIORITY_CPU();
  return (j);                                  // ★ 返回值是 dummy（j = Σi），调用方一律丢弃
}
```

```cpp
namespace ut {
/** The current value of @@innodb_spin_wait_pause_multiplier. Determines
how many PAUSE instructions to emit for each requested unit of delay
when calling `ut_delay(delay)`. The default value of 50 causes `delay*50` PAUSES
which was equivalent to `delay` microseconds on 100 MHz Pentium + Visual C++.
Useful on processors which have "non-standard" duration of a single PAUSE
instruction - one can compensate for longer PAUSES by setting the
spin_wait_pause_multiplier to a smaller value on such machine */
extern ulong spin_wait_pause_multiplier;       // 默认 50
}
```

**`UT_RELAX_CPU` 全平台分支**（`ut0ut.h:81`）——★ **aarch64 是 `isb` 不是 `yield`**：

```cpp
#if defined(HAVE_PAUSE_INSTRUCTION)
#define UT_RELAX_CPU() __asm__ __volatile__("pause")        // x86
#elif defined(HAVE_FAKE_PAUSE_INSTRUCTION)
#define UT_RELAX_CPU() __asm__ __volatile__("rep; nop")     // 老 x86
#elif defined _WIN32
#define UT_RELAX_CPU() YieldProcessor()
#elif defined(__aarch64__)
/* A "yield" instruction in aarch64 is essentially a nop, and does not cause
enough delay to help backoff. "isb" is a barrier that, especially inside a
loop, creates a small delay without consuming ALU resources.
Experiments shown that adding the isb instruction improves stability and reduces
result jitter. Adding more delay to the UT_RELAX_CPU than a single isb reduces
performance. */
#define UT_RELAX_CPU() __asm__ __volatile__("isb" ::: "memory")   // ★ ARM64
#else
#define UT_RELAX_CPU() __asm__ __volatile__("" ::: "memory")      // 通用兜底
#endif

#if defined(HAVE_HMT_PRIORITY_INSTRUCTION)
#define UT_LOW_PRIORITY_CPU() __asm__ __volatile__("or 1,1,1")    // ★ POWER
#define UT_RESUME_PRIORITY_CPU() __asm__ __volatile__("or 2,2,2")
#else
#define UT_LOW_PRIORITY_CPU() ((void)0)
#define UT_RESUME_PRIORITY_CPU() ((void)0)
#endif
```

> ⚠️ **勘误**：**没有 `__builtin_ia32_pause`**；**没有 POWER 的 `UT_RELAX_CPU` 分支**（POWER 走的是另一对宏 `UT_LOW/RESUME_PRIORITY_CPU`）。

**⚠️ `UT_WAIT_FOR` 宏不存在**——真名是三个模板函数：

| 函数 | 位置 | 语义 |
|---|---|---|
| `ut::wait_for(cond, max_wait)` | `ut0ut.ic:131` | 纯 sleep，固定 **2ms** 粒度 |
| `ut::wait_for(spins_limit, sleep, condition)` | `ut0ut.ic:153` | 自旋→sleep，**每 4 次翻倍**，上限 100ms |
| `os_event_wait_for(event, spins_limit, timeout, condition)` | `os0event.ic:58` | ★ **自旋 → event 的分界** |

```cpp
template <typename Condition>
inline static Wait_stats os_event_wait_for(os_event_t &event,
                                           uint64_t spins_limit,
                                           std::chrono::microseconds timeout,
                                           Condition condition = {}) {
  uint32_t next_level = 4;                       // Windows 上是 64
  uint32_t waits = 0;
  constexpr auto MIN_TIMEOUT = std::chrono::microseconds{1};
  constexpr auto MAX_TIMEOUT = std::chrono::microseconds{100 * 1000};

  while (true) {
    const bool wait = spins_limit == 0;          // ★ 递减到 0 就转入事件等待
    const int64_t sig_count = !wait ? 0 : os_event_reset(event);
    //                               ★★ 先 reset 抓计数，再检查条件 → 防丢唤醒

    if (condition(wait)) return (Wait_stats{waits});

    if (!wait) {
      --spins_limit;
      UT_RELAX_CPU();                            // ★ 自旋段只有 PAUSE，不调 ut_delay
    } else {
      ++waits;
      if (timeout < MIN_TIMEOUT) {
        /* ... we measured how long does it take to wake up on timeout,
        depending on timeout:
             1us ->   57us,   10us ->   66us,   20us ->   76us,
            50us ->  106us,  100us ->  156us, 1000us -> 1100us.
        (Oracle Linux 4.14.28) */
        timeout = MIN_TIMEOUT;                   // ★ 实测：1us 超时实际要 57us
      }
      if (waits == next_level) {
        timeout = std::min(timeout * 2, MAX_TIMEOUT);   // ★ 指数退避，上限 100ms
        next_level += 4;
      }
      os_event_wait_time_low(event, timeout, sig_count);   // ★ 传第 77 行抓到的计数
    }
  }
}
```

**spin 相关的三个系统变量**（⚠️ `srv_spin_wait_delay` 单位是 **PAUSE 轮数基数**，不是微秒）：

```cpp
// srv/srv0srv.cc:589
ulong srv_n_spin_wait_rounds = 30;      // ★ 默认 30 轮
ulong srv_spin_wait_delay = 6;          // ★ 默认 6（不是微秒）
bool srv_priority_boost = true;
// ut/ut0ut.cc:64
namespace ut { ulong spin_wait_pause_multiplier = 50; }   // ★ 默认 50
```

```cpp
// ha_innodb.cc:22789
static MYSQL_SYSVAR_ULONG(sync_spin_loops, srv_n_spin_wait_rounds,
                          PLUGIN_VAR_RQCMDARG,
                          "Count of spin-loop rounds in InnoDB mutexes (30 by default)",
                          nullptr, nullptr, 30L, 0L, UINT32_MAX, 0);
static MYSQL_SYSVAR_ULONG(spin_wait_delay, srv_spin_wait_delay,
                          PLUGIN_VAR_OPCMDARG,
                          "Maximum delay between polling for a spin lock (6 by default)",
                          nullptr, nullptr, 6L, 0L, 1000, 0);
static MYSQL_SYSVAR_ULONG(spin_wait_pause_multiplier, ut::spin_wait_pause_multiplier,
                          PLUGIN_VAR_RQCMDARG,
                          "Controls how many times in a row to use a PAUSE "
                          "instruction to achieve one unit of delay in a spin "
                          "lock (see @@innodb_spin_wait_delay), defaults to 50",
                          nullptr, nullptr, 50, 0, 100, 0);
```

**典型调用点**（`sync0rw.cc:286`）——★ **随机化退避**防止同频共振：

```cpp
  os_rmb;                                                   // 读屏障
  while (i < srv_n_spin_wait_rounds && lock->lock_word <= 0) {
    if (srv_spin_wait_delay) {
      ut_delay(ut::random_from_interval_fast(0, srv_spin_wait_delay));
      //       ★★ 每轮随机 0..delay，不是固定值
    }
    i++;
  }
  if (i >= srv_n_spin_wait_rounds) {
    std::this_thread::yield();                              // ★ 自旋耗尽 → 让出 CPU
  }
```

> ★ **两级之间的"第三级"是 `std::this_thread::yield()`**，然后才进 sync array / futex 真睡。所以严格说是**三级**：自旋(PAUSE) → yield → OS 等待。

临界区通常几十纳秒到微秒，而"睡眠 + 唤醒"的往返约 57–110µs（源码注释在 `os_event_wait_for` 处给出了实测表：自旋 1µs 唤醒延迟 57µs、1000µs 唤醒延迟 1100µs）。所以所有 InnoDB 原语的加锁循环都是同一骨架：

```
try_lock 快路径（一次 CAS）
  → TTAS 自旋 srv_n_spin_wait_rounds 轮（每轮 ut_delay 随机退避）
    → std::this_thread::yield()
      → reserve cell（登记进 wait array）→ set waiter flag → **复查一次锁**
        → os_event 睡眠
```

代价：自旋烧 CPU——负载高时"大家都拿不到锁、都在烧 CPU"会造成吞吐崩塌。缓解手段全是可调参数（`innodb_spin_wait_delay`、`innodb_sync_spin_loops`），且睡眠前的"复查一次"是防丢唤醒的关键（见下）。

#### 二、★ 防丢唤醒：为什么处处"先登记、再复查"

**源码里的原始场景注释**（`os0event.h:83` / `os0event.cc:129`）——作者把丢唤醒的时序画出来了：

```cpp
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

**机制本体：`reset()` 返回 signal_count，`wait_low()` 比对它**：

```cpp
  int64_t reset() UNIV_NOTHROW {
    mutex.enter();
    if (m_set) {
      m_set = false;                    // ★ manual-reset：必须显式清位
    }
    int64_t ret = signal_count;         // ★ 在锁内读"此刻"的计数
    mutex.exit();
    return (ret);                       // ★ 交给调用者当 reset_sig_count
  }
```

```cpp
void os_event::wait_low(int64_t reset_sig_count) UNIV_NOTHROW {
  mutex.enter();
  if (!reset_sig_count) {
    reset_sig_count = signal_count;     // ★ 传 0 = "我不关心中间是否有人 set 过"
  }
  while (!m_set && signal_count == reset_sig_count) {
    //             ★★ 双条件：既没置位、计数又没变，才真阻塞
    wait();
    /* Spurious wakeups may occur: we have to check if the
    event really has been signaled after we came here to wait. */
  }
  mutex.exit();
}
```

> ★★ **`signal_count` 从 1 起跳**（构造里 `signal_count = 1`），因为 **0 被保留为"未指定"哨兵值**（`wait_low` 里 `if (!reset_sig_count)` 判据）。

**`set()` 是幂等的 + 广播的**（⚠️ 不是 signal 一个）：

```cpp
  void set() UNIV_NOTHROW {
    mutex.enter();
    if (!m_set) {            // ★ 已 set 过就不再 broadcast，避免无谓唤醒风暴
      broadcast();
    }
    mutex.exit();
  }

  void broadcast() UNIV_NOTHROW {
    m_set = true;            // ①先置位（后续 wait 直接返回）
    ++signal_count;          // ②再递增（让持有旧 count 的 waiter 也退出）
    pthread_cond_broadcast(&cond_var);   // ③★ 广播，不是 signal
  }
```

> ⚠️ **勘误**：`os_event` **有私有 `signal()`**（`pthread_cond_signal`）但**整份 `os0event.cc` 无人调用——死代码**。所以 `os_event_set()` 唤醒的是**全部**等待者。
> ⚠️ **`m_set` / `signal_count` 不是 `std::atomic`**，是原生 `bool` / `int64_t`，全靠 `EventMutex`（=`OSMutex`，裸 pthread mutex）串行化。

**★ 教科书级用法**（`buf/buf0dblwr.cc:528` / `:643`）——reset/fence/复查/set 四步：

```cpp
    auto sig_count = os_event_reset(m_event);          // ① 先 reset 拿 signal_count
    std::atomic_thread_fence(std::memory_order_acquire);  // ② acquire fence
    if (m_batch_running.load(std::memory_order_acquire)) {  // ③ 复查条件
      mutex_exit(&m_mutex);                            //    已 true → 不睡
      ...
  void batch_completed() noexcept {
    m_batch_running.store(false, std::memory_order_release);  // ① 先发布状态
    std::atomic_thread_fence(std::memory_order_release);      // ② release fence
    os_event_set(m_event);                                    // ③ 再 set
  }                                                    // ★★ 顺序不可颠倒，否则丢唤醒
```

**wait array 里也用同一招**（`sync0arr.cc:240` / `:341`）：

```cpp
  os_event_t event = sync_cell_get_event(cell);
  cell->signal_count = os_event_reset(event);   // ★ reserve_cell 时就抓好
  ...
  os_event_wait_low(sync_cell_get_event(cell), cell->signal_count);
```

**★ `TTASEventMutex::wait()` 顶部有整整 100 行的 C++14 内存模型证明**（`ut0mutex.ic:47`，"Theorem 1"）——它论证"只要每个 `enter()` 的线程最终都 `exit()`，进入 `os_event_wait()` 的线程必然被唤醒"。推论出的**两条操作顺序硬约束**：

```cpp
  /* The memory order of the array reservation and
  the change in the waiters field is important: when
  we suspend a thread, we first reserve the cell and
  then set waiters field to 1. When threads are released
  in mutex_exit, the waiters field is first set to zero
  and then the event is set to the signaled state. */
  set_waiters();            // 等待侧：① reserve cell → ② set_waiters → ③ try_lock → ④ wait

void TTASEventMutex<Policy>::signal() UNIV_NOTHROW {
  clear_waiters();          // 释放侧：① clear_waiters → ② os_event_set
  /* The memory order of resetting the waiters field and
  signaling the object is important. See Theorem 1 above. */
  os_event_set(m_event);
  sync_array_object_signalled();
}
```

> ★★ **`rw_lock_t` 的两个 event 各有一套顺序协议**（`sync0rw.cc` 头注释，逐字抄录）：
> - `event`：**(1) `os_event_reset()` 记录计数 → (2) `waiters=1` → (3) 复查 `lock_word <= 0`**。(1) 必须在 (2) 前以保证不丢信号；(2) 必须在 (3) 前以保证信号会被发出。
> - `wait_ex_event`：**(1) `lock_word -= X_LOCK_DECR` → (2) `os_event_reset()` → (3) 复查 `lock_word < 0`**。(1) 必须最先，既阻止新读者进入，又通知 unlocker 必须发信号。

**兜底：`sync_arr_wake_threads_if_sema_free()`**（`sync0arr.cc:780`，由 `srv_error_monitor_thread` 约每 1 秒调）——注释承认这是给"漏发信号"打补丁：

> ```cpp
> /** If the wakeup algorithm does not work perfectly at semaphore releases,
>  this function will do the waking (see the comment in mutex_exit). This
>  function should be called about every 1 second in the server.
>
>  Note that there's a race condition between this thread and mutex_exit
>  changing the lock_word and calling signal_object, so sometimes this finds
>  threads to wake up even when nothing has gone wrong. */
> ```

InnoDB 的慢路径有一个统一协议，`os_event` 的注释给出了它要防的反例：

> ```
> thread A calls os_event_reset()
> thread B calls os_event_set()   [event->m_set == true]
> thread C calls os_event_reset() [event->m_set == false]
> thread A calls os_event_wait()  [infinite wait!]
> ```

pthread_cond 没有"记住已通知"的状态，reset 与 wait 之间发生的 set 会丢。InnoDB 的解法是 **sticky event + signal_count**：`reset()` 返回当时的 `signal_count`，`wait_low(sig_count)` 只在 `signal_count` 未变时才真正睡眠——reset 与 wait 之间发生的 set 不会丢。

各原语的落点：mutex 的 `wait()` 里"**先 `set_waiters()` 再最后试 spin 次、失败才睡**"；`rw_lock_s_lock_spin` 里"**先 `rw_lock_set_waiter_flag(lock)` 再 `rw_lock_s_lock_low` 复查**"。顺序反了就是丢唤醒 bug。`ut0mutex.ic` 的 `wait()` 上方甚至有一段 **~120 行的 Theorem 1 注释**，证明"reserve cell → set_waiters → try 失败 → 睡"在 seq-cst 内存序下不会无限睡眠——给并发协议写正确性证明，这在工程代码里极为罕见。

#### 三、★ SX 为什么必须存在

B-tree SMO（页分裂/合并）与字典 DDL 需要"**读者不受阻、修改者串行**"的中间态。纯 S/X 两态下，修改者要么饿死（读者源源不断）、要么阻塞全部读。SX 的兼容矩阵：

```
    S  SX  X
 S  +  +   -
 SX +  -   -
 X  -  -   -
```

SX 阻止其他 SX 与一切 X，但放行 S——"树结构修改串行化、并发读照跑"。落点两大用户：`dict_index_t::lock`（B-tree 索引锁，SMO 用 SX）与 `dict_operation_lock`（字典锁）。

#### 四、为什么 lock_word 用"负数区间"编码

一个 `atomic<int32>` 承载 S 计数 + SX + X + 递归（见「rw_lock_t」节的完整编码表）。收益：快路径一次 CAS；状态转换天然原子。代价：编码不可读（负数区间含义要背）、debug 只能靠 `magic_n` + `rw_lock_validate` 兜底。这是"信息压进一个原子字"的空间换时间设计，代价是**读代码时必须先背编码表**。

#### 五、失效场景

- **自旋风暴**：`srv_n_spin_wait_rounds` 过大 + 热点锁 → CPU 100% 但吞吐反降。观测：`SHOW ENGINE INNODB STATUS` 的 SEMAPHORES 段 + PFS 的 spin 计数。
- **长信号量等待崩溃**：等待超过 `srv_fatal_semaphore_wait_threshold`（默认 600 秒）触发 fatal——这是"latch 泄漏/死锁"暴露为崩溃的机制，不是 bug 本身。
- **TTASEventMutex 的睡眠链路长**：sync array 往返 + 每实例 os_event（内部又有 mutex+cond）——futex 构建下这条路径整个被内核接管。

### 理论溯源

- **TTAS（test-and-test-and-set）自旋锁**：教科书并发原语。本实现的落地是 `is_free()` 循环（读锁字 + 随机退避 `ut_delay`），futex 版把"测试"留给内核。
- **Linux futex**：`TTASFutexMutex` 是 futex 三态协议（0/1/2）的标准应用——0=UNLOCKED、1=LOCKED、2=LOCKED_WITH_WAITERS，与 Drew Eckhardt/futex(2) man page 的经典三态设计一致。
- **读者-写者锁的写者优先变体**：X 请求者**先减 lock_word 占坑**（成为 next-writer，负值区间阻止新 S）再等读者清零——防写者饿死的标准手法，落点 `rw_lock_x_lock_low` 的 `decr(X_LOCK_DECR, threshold=X_LOCK_HALF_DECR)`。

### 算法与数据结构

**★ 唯一的原子状态字：`lock_word`**（`sync0rw.h:106`）——⚠️ `X_LOCK_DECR` 是 **0x20000000** 不是 0x10000000：

```cpp
/* We decrement lock_word by X_LOCK_DECR for each x_lock. It is also the
start value for the lock_word, meaning that it limits the maximum number
of concurrent read locks before the rw_lock breaks. */
/* We decrement lock_word by X_LOCK_HALF_DECR for sx_lock. */
constexpr int32_t X_LOCK_DECR = 0x20000000;       // ★ = 536870912
constexpr int32_t X_LOCK_HALF_DECR = 0x10000000;  // ★ = 268435456
```

**权威编码表**（`sync0rw.cc:54` 头注释原文，逐字）：

```cpp
lock_word == X_LOCK_DECR:       Unlocked.
X_LOCK_HALF_DECR < lock_word < X_LOCK_DECR:
                                S locked, no waiting writers.
                                (X_LOCK_DECR - lock_word) is the number of S locks.
lock_word == X_LOCK_HALF_DECR:  SX locked, no waiting writers.
0 < lock_word < X_LOCK_HALF_DECR:
                                SX locked AND S locked, no waiting writers.
                                (X_LOCK_HALF_DECR - lock_word) is the number of S locks.
lock_word == 0:                 X locked, no waiting writers.
-X_LOCK_HALF_DECR < lock_word < 0:
                                S locked, with a waiting writer.
                                (-lock_word) is the number of S locks.
lock_word == -X_LOCK_HALF_DECR: X locked and SX locked, no waiting writers.
-X_LOCK_DECR < lock_word < -X_LOCK_HALF_DECR:
                                S locked, with a waiting writer which has SX lock.
                                -(lock_word + X_LOCK_HALF_DECR) is the number of S locks.
lock_word == -X_LOCK_DECR:      X locked with recursive X lock (2 X locks).
-(X_LOCK_DECR + X_LOCK_HALF_DECR) < lock_word < -X_LOCK_DECR:
                                X locked. The number of the X locks is:
                                2 - (lock_word + X_LOCK_DECR)
lock_word == -(X_LOCK_DECR + X_LOCK_HALF_DECR):
                                X locked with recursive X lock (2 X locks) and SX locked.
lock_word < -(X_LOCK_DECR + X_LOCK_HALF_DECR):
                                X locked and SX locked.
                                The number of the X locks is:
                                2 - (lock_word + X_LOCK_DECR + X_LOCK_HALF_DECR)

 LOCK COMPATIBILITY MATRIX
    S SX  X
 S  +  +  -
 SX +  -  -
 X  -  -  -
```

> ★★ **一个字编码 4 类信息**：S 锁数量、是否有 SX、X 锁重数、是否有等待中的 writer。**没有 `writer_count` / `reader_count` 字段**——全靠 `rw_lock_get_x_lock_count()` / `rw_lock_get_reader_count()` 从 `lock_word` 反推。

```cpp
static inline ulint rw_lock_get_x_lock_count(const rw_lock_t *lock) {
  lint lock_copy = lock->lock_word;
  if (lock_copy == 0 || lock_copy == -X_LOCK_HALF_DECR) {
    return (1);                                        // "1 x-lock" 或 "1 x-lock + sx"
  } else if (lock_copy > -X_LOCK_DECR) {
    return (0);                                        // s-locks / sx / x-waiter
  } else if (lock_copy > -(X_LOCK_DECR + X_LOCK_HALF_DECR)) {
    return (2 - (lock_copy + X_LOCK_DECR));
  } else {
    return (2 - (lock_copy + X_LOCK_DECR + X_LOCK_HALF_DECR));
  }
}
```

**`struct rw_lock_t` 完整定义**（`sync0rw.h:346`）：

```cpp
struct rw_lock_t
#ifdef UNIV_DEBUG
    : public latch_t          // ★ debug 下继承 latch_t，才有 get_level() / m_id 供 LatchDebug
#endif
{
  std::atomic<int32_t> lock_word;            // ★ 唯一的权威状态
  std::atomic<bool> waiters;                 // 1: 有等待者
  std::atomic<bool> recursive;               // ★ 也是"writer_thread 有效"的哨兵
  volatile ulint sx_recursive;               // ⚠️ 非 atomic，只能由持锁者自己改
  std::atomic<std::thread::id> writer_thread;
  Atomic_xor_of_thread_id reader_thread;     // ★ XOR 聚合，只能反推唯一读者
  os_event_t event;                          // 普通等待者（S/X/SX 竞争）
  os_event_t wait_ex_event;                  // ★ 已占位的 next-writer 等 S 退出
  ut::Location clocation;
  const char *last_s_file_name;
  const char *last_x_file_name;
  bool is_block_lock;
  uint16_t last_s_line;
  uint16_t last_x_line;
  uint32_t count_os_wait;                    // ★★ 8.0.39 唯一的 per-lock 等待计数器
  UT_LIST_NODE_T(rw_lock_t) list;
#ifdef UNIV_PFS_RWLOCK
  struct PSI_rwlock *pfs_psi;
#endif
  bool is_x_blocked_by_s() {
    const auto snapshot = lock_word.load();
    return snapshot < 0 && -X_LOCK_DECR < snapshot && snapshot != -X_LOCK_HALF_DECR;
  }
};
```

**初始化**（`sync0rw.cc:191`）：

```cpp
  lock->lock_word = X_LOCK_DECR;                            // ★ 初值 = 0x20000000
  lock->waiters = false;
  lock->recursive.store(false, std::memory_order_relaxed);  // ★ 默认非递归
  lock->sx_recursive = 0;
  lock->writer_thread = std::thread::id{};                  // "不可能的线程 id"
  lock->event = os_event_create();
  lock->wait_ex_event = os_event_create();                  // ★ 两个 event
  UT_LIST_ADD_FIRST(rw_lock_list, lock);                    // 挂进全局链表
```

**★ recursive-x 的完整处理**（`sync0rw.cc:447`）——这就是"经典坑"的全部：

```cpp
static inline bool rw_lock_x_lock_low(rw_lock_t *lock, ulint pass,
                                      const char *file_name, ulint line) {
  if (rw_lock_lock_word_decr(lock, X_LOCK_DECR, X_LOCK_HALF_DECR)) {
    ut_a(!lock->recursive.load(std::memory_order_relaxed));
    rw_lock_set_writer_id_and_recursion_flag(lock, !pass);
    //  ★ pass==0 → recursive=true（允许本线程递归）；pass!=0（AIO 移交）→ false
    rw_lock_x_lock_wait(lock, pass, 0, file_name, line);   // 等 S 退到 0
  } else {
    if (!pass && lock->recursive.load(std::memory_order_acquire) &&
        lock->writer_thread.load(std::memory_order_relaxed) ==
            std::this_thread::get_id()) {
      if (rw_lock_lock_word_decr(lock, X_LOCK_DECR, 0)) {
        /* 有 SX 但无 X：SX 持有者升级为 X，等 S 退到 -X_LOCK_HALF_DECR */
        rw_lock_x_lock_wait(lock, pass, -X_LOCK_HALF_DECR, file_name, line);
      } else {
        /* 已有 ≥2 个 X */
        if (lock->lock_word == 0 || lock->lock_word == -X_LOCK_HALF_DECR) {
          lock->lock_word -= X_LOCK_DECR;   // ★ 第 1→2 个 X：整块减
        } else {
          ut_ad(lock->lock_word <= -X_LOCK_DECR);
          --lock->lock_word;                // ★ 第 3/4/... 个 X：只减 1（本线程独占，无需 CAS）
        }
      }
    } else {
      return false;   // ★★ 同线程持 S 再请求 X 会走到这里 → 死锁风险
    }
  }
  ...
}
```

> ⚠️ **`rw_lock_x_lock_wait()` 是 IF_DEBUG 包装**（`sync0rw.cc:440`）——release 下 `pass` 参数被 `[[maybe_unused]]` 丢掉，真身是 `rw_lock_x_lock_wait_func()`（`:373`）。
> ⚠️ **`rw_lock_s_lock()` 禁止递归 S**（`sync0rw.ic:271` 的 `ut_ad(!rw_lock_own(lock, RW_LOCK_S))`）——注释解释：S 锁不记 thread id，若允许递归 S，可能与请求 X 的线程死锁。**X/SX 可递归，S 不可递归**。

**SX 的第一层只减 HALF**（`sync0rw.cc:520`）：

```cpp
      if (lock->sx_recursive++ == 0) {
        /* This thread is making first SX-lock request and it must be holding
        at least one X-lock here because: ... */
        ut_ad((lock->lock_word == 0) ||
              ((lock->lock_word <= -X_LOCK_DECR) &&
               (lock->lock_word > -(X_LOCK_DECR + X_LOCK_HALF_DECR))));
        lock->lock_word -= X_LOCK_HALF_DECR;
      }
```

| 结构 | 核心字段 | 复杂度 |
|---|---|---|
| `os_event` | `m_set`（sticky）、`signal_count`、mutex、cond | set/wait O(1)（各一把内部锁） |
| `sync_cell_t` | latch 联合体、request_type、signal_count、`last_scan`（死锁检测三色标记） | reserve O(1)（free-slot 链） |
| `TTASEventMutex` | `m_owner`（atomic thread::id）、`m_event`、`m_waiters` | 快路径 1 CAS；慢路径 sync array 往返 |
| `TTASFutexMutex` | `m_lock_word`（三态 uint32） | 快路径 1 CAS；慢路径 1 futex syscall |
| `rw_lock_t` | `lock_word`（atomic int32）、`waiters`、`writer_thread`、`event`/`wait_ex_event` | S 锁快路径 1 CAS |

### 他库对比与演进动机

| 数据库 | 引擎层原语 | 与 InnoDB 的差异 |
|---|---|---|
| **PostgreSQL** | `s_lock`（spinlock）+ `LWLock`（轻量锁自带等待队列） | PG 的 LWLock 同样自研同样带诊断，但**没有 SX 三态**——PG 的 buffer 锁定用"共享/独占"两态 + pin 计数组合出中间语义 |
| **Oracle** | latch（自研，spin+sleep 可调）+ enqueue | latch 与 InnoDB mutex 同构（spin 参数可调、有 v$latch 观测）；enqueue 对应事务锁不是本篇范围 |
| **MySQL server 侧** | mysys 三层包装（见 [`mysys_primitives.md`](mysys_primitives.md)） | **同一仓库两种哲学**：server 包装 OS，InnoDB 完全自研 |

**为什么 8.0 不换回 pthread**：诊断能力（sync array、long-wait 报警、LatchDebug 锁序检查）与 SX 三态是硬需求；futex 构建已把慢路径交给内核，快路径与 pthread 相当——自研的成本只剩模板代码维护。

---

## 核心实现

### L0：地基层（futex / pthread / std::atomic）

**⚠️ `ut0atomic.h` 不存在**——真名 `include/os0atomic.h`，全文 83 行，8.0 里**只剩两个屏障宏**（真正的原子操作已全面改用 `std::atomic`）：

```cpp
#ifdef HAVE_IB_GCC_ATOMIC_THREAD_FENCE
#define HAVE_MEMORY_BARRIER
#define os_rmb __atomic_thread_fence(__ATOMIC_ACQUIRE)   // ★ 读屏障 = acquire fence
#define os_wmb __atomic_thread_fence(__ATOMIC_RELEASE)   // ★ 写屏障 = release fence
#elif defined(HAVE_IB_GCC_SYNC_SYNCHRONISE)
#define os_rmb __sync_synchronize()                      // 退化为全屏障
#define os_wmb __sync_synchronize()
#elif defined(HAVE_IB_MACHINE_BARRIER_SOLARIS)
#define os_rmb __machine_r_barrier()
#define os_wmb __machine_w_barrier()
#elif defined(HAVE_WINDOWS_MM_FENCE) && defined(_WIN64)
#define os_rmb _mm_lfence()
#define os_wmb _mm_sfence()
#else
#define os_rmb                                           // ★ 完全退化为 no-op
#define os_wmb
#define IB_MEMORY_BARRIER_STARTUP_MSG "Memory barrier is not used"
#endif
```

> ★ 探测逻辑在 `innodb.cmake:173-190`。`os_rmb` 在加锁路径大量出现（如 `sync0rw.cc:288` 的 `os_rmb;`）。

**⚠️ `atomic_signal_fence` 在 InnoDB 里 0 命中**——8.0.39 完全不用它。

**`std::atomic` 的 8 处典型用法**：

| 字段 | 位置 | 类型 | 内存序 |
|---|---|---|---|
| `TTASEventMutex::m_owner` | `ib0mutex.h:577` | `std::atomic<std::thread::id>` | **全默认 seq_cst**；`static_assert(is_always_lock_free)` |
| `TTASEventMutex::m_waiters` | `ib0mutex.h:587` | `std::atomic_bool` | 全默认 seq_cst |
| `TTASFutexMutex::m_lock_word` | `ib0mutex.h:369` | `alignas(4) std::atomic<mutex_state_t>` | 全默认 seq_cst + `alignas(4)` 满足 futex 对齐 |
| `os_thread_count` | `os0thread-create.h:51` | `std::atomic_int` | `fetch_add(relaxed)` + 外层 **release fence** |
| `Link_buf::m_tail` | `ut0link_buf.h:357` | — | `load(acquire)`（必须先看到尾指针推进） |
| `copyable_atomic_t` | `buf0buf.h:1120` | 继承 `std::atomic<T>` | 为 memcpy 式容器定制，拷贝用 relaxed |
| `srv_innodb_needs_monitoring` | `srv0srv.h:722` | `std::atomic_uint32_t` | 跨模块原子开关 |
| `rw_lock_t::lock_word` | `sync0rw.h` | `std::atomic<int32_t>` | 见上文 |

> ★★ **结论：PolicyMutex 一律依赖 `std::atomic` 默认的 `memory_order_seq_cst`**，**不写任何显式内存序参数**。这与 `ut0mutex.ic` 里 Theorem 1 证明依赖的"全部 seq_cst"前提完全一致。

**显式 fence 在 InnoDB 的全部使用点**（12 处）：

| 位置 | 种类 | 用途 |
|---|---|---|
| `mach/mach0data.cc:106,122,138,154,173,188,204` | `__atomic_thread_fence(ACQUIRE)` | **GCC bug 77673 workaround**（`DEPLOY_FENCE`，仅 GCC≥5 非 clang） |
| `log/log0log.cc:809` | release | "Do not reorder writes above, below this line" |
| `log/log0buf.cc:1097 / 1154` | release | 发布 `sn` / dirty-page 事件前 |
| `log/log0buf.cc:1328` | acquire | 读到新 lsn 后读数据前 |
| `buf/buf0dblwr.cc:532 / 645` | acquire / release | ★ os_event 教科书用法（见"防丢唤醒"） |
| `ut0seq_lock.h:72 / 94` | release / acquire | Seq_lock 两端配对 |
| `os0thread-create.h:210 / 221` | release | 线程 preamble / epilogue |
| `ib0mutex.h:250 / 347` | acquire | futex `exit()` 开头 / `ttas()` 开头 |

**futex 三态 lock_word 与系统调用**（`ib0mutex.h:173`）——Linux 默认**不是 futex**：

```cpp
  enum class mutex_state_t : futex_word_t {
    UNLOCKED = 0,          // 空闲
    LOCKED = 1,            // 有人持有，无人等待
    LOCKED_WITH_WAITERS = 2  // ★ 有人持有且有等待者 → 释放时必须 syscall
  };

  bool set_waiters() UNIV_NOTHROW {
    return m_lock_word.exchange(mutex_state_t::LOCKED_WITH_WAITERS) ==
           mutex_state_t::UNLOCKED;   // ★ 返回 true = "抢到了"，不用睡
  }

  uint32_t wait() UNIV_NOTHROW {
    uint32_t n_waits = 0;
    do {
      ++n_waits;
      syscall(SYS_futex, &m_lock_word, FUTEX_WAIT_PRIVATE,
              mutex_state_t::LOCKED_WITH_WAITERS, 0, 0, 0);
      //  Since we are retrying the operation the return value doesn't matter.
    } while (!set_waiters());   // ★ 醒来立刻再抢一次
    return (n_waits);
  }

  void signal() UNIV_NOTHROW {
    syscall(SYS_futex, &m_lock_word, FUTEX_WAKE_PRIVATE, 1, 0, 0, 0);  // 只唤醒 1 个
  }
```

```cpp
  void exit() UNIV_NOTHROW {
    std::atomic_thread_fence(std::memory_order_acquire);
    if (state() == mutex_state_t::LOCKED_WITH_WAITERS) {
      m_lock_word = mutex_state_t::UNLOCKED;
    } else if (unlock() == mutex_state_t::LOCKED) {
      /* No threads waiting, no need to signal a wakeup. */
      return;   // ★★ 无等待者 → 省掉一次系统调用
    }
    signal();
  }
```

> ⚠️ **`FUTEX_WAIT_PRIVATE` / `FUTEX_WAKE_PRIVATE`** —— 带 `_PRIVATE` 表示不跨进程。

**★ 默认实现是 `MUTEX_SYS`，不是 futex**（`innodb.cmake:253`）：

```cmake
IF(MUTEXTYPE MATCHES "event")
  ADD_DEFINITIONS(-DMUTEX_EVENT)
ELSEIF(MUTEXTYPE MATCHES "futex" AND DEFINED HAVE_IB_LINUX_FUTEX)
  ADD_DEFINITIONS(-DMUTEX_FUTEX)
ELSE()
   ADD_DEFINITIONS(-DMUTEX_SYS)     # ★★ 默认走这里
ENDIF()
```

```cpp
// ut0mutex.h:88
#ifdef MUTEX_FUTEX
typedef FutexMutex ib_mutex_t;
typedef BlockFutexMutex ib_bpmutex_t;
#define MUTEX_TYPE "Uses futexes"
#elif defined(MUTEX_SYS)
typedef SysMutex ib_mutex_t;                 // ★ 默认 = OSTrackMutex = 裸 pthread
typedef BlockSysMutex ib_bpmutex_t;
#define MUTEX_TYPE "Uses system mutexes"
#elif defined(MUTEX_EVENT)
typedef SyncArrayMutex ib_mutex_t;
typedef BlockSyncArrayMutex ib_bpmutex_t;
#define MUTEX_TYPE "Uses event mutexes"
#else
#error "ib_mutex_t type is unknown"
#endif
```

> ⚠️ **勘误**：`EventMutex` **不是 `TTASEventMutex` 的封装**（`ut0mutex.h:74`：`typedef OSMutex EventMutex;`）——它是**裸 pthread mutex**。`TTASEventMutex` 的封装真名是 **`SyncArrayMutex`**。
> ⚠️ **不存在 `BufPoolMutex` / `LockMutex` / `DictMutex` / `BlockUTMutex`**。真名：`ib_bpmutex_t`、`Lock_mutex`（`lock0latches.h:107`）、`DictSysMutex`（`dict0types.h:319`）、`BlockMutexPolicy`（是策略不是 mutex）。
> ⚠️ **`ttas_lock` 符号不存在**，只有成员函数 `ttas()`（futex 版）；且**全部用 `compare_exchange_strong`，没有 weak**。
> ⚠️ **不存在 `mutex_enter()` / `mutex_exit()` 函数**——它们是**宏** → `mutex_enter_inline<Mutex>`（`ut0mutex.h:114`）。`sync0sync.cc` 全文仅 292 行，只有 PSI key + 4 个工具函数。
> ⚠️ **不存在 `UT_MUTEX_HOOK` / `m_loc_file` / `m_loc_line`**——对接点是 `UT_LOCATION_HERE` → `ut::Location{__FILE__, __LINE__}`（`ut0core.h:48`），**不分 debug/release 永远传**，代价仅是两个立即数（且 `OSTrackMutex` 路径标了 `[[maybe_unused]]` 会被优化掉）。

**PolicyMutex 的三段式 hook**（★ 解锁顺序**反直觉**）：

```cpp
  void exit() UNIV_NOTHROW {
    pfs_exit();                 // ①
    policy().release(m_impl);   // ② 先 release hook
    m_impl.exit();              // ③ 再真正解锁
  }

  void enter(uint32_t n_spins, uint32_t n_delay, const char *name, uint32_t line) {
    PSI_mutex_locker *locker = pfs_begin_lock(&state, name, line);
    policy().enter(m_impl, name, line);      // ① 加前 hook
    m_impl.enter(n_spins, n_delay, name, line);  // ② 真正加锁
    policy().locked(m_impl, name, line);     // ③ 加后 hook
    pfs_end(locker, 0);
  }
```

```cpp
/* Do nothing —— 零成本策略 */
template <typename Mutex>
struct NoPolicy {
  void init(const Mutex &, latch_id_t, const char *, uint32_t) UNIV_NOTHROW {}
  void destroy() UNIV_NOTHROW {}
  void enter(const Mutex &, const char *, ulint line) UNIV_NOTHROW {}
  void add(uint32_t, uint32_t) UNIV_NOTHROW {}
  void locked(const Mutex &, const char *, ulint) UNIV_NOTHROW {}
  void release(const Mutex &) UNIV_NOTHROW {}
  std::string to_string() const { return (""); }
  latch_id_t get_id() const;
};
```

> ★★ **两条策略的差别**：`GenericPolicy` 用 `Counter::Count m_count`（**每实例独立计数**）；`BlockMutexPolicy` 用 `Counter::Count *m_count`（**聚合计数**，buf page mutex 太多，逐个计开销大）。
> ★ **`OSTrackMutex` 不记录 owner 线程**——只记 3 个 debug bool（`m_freed` / `m_locked` / `m_destroy_at_exit`），所以 `is_owned()` 返回 `m_locked && m_policy.is_owned()`。系统 mutex 查不到 owner。

这一层常被忽略，但它决定了上面所有原语的性能上限与平台差异。

**futex —— 裸 syscall，无封装层**：

```cpp
/* ib0mutex.h —— TTASFutexMutex */
uint32_t wait() UNIV_NOTHROW {
  uint32_t n_waits = 0;
  /* Use FUTEX_WAIT_PRIVATE because our mutexes are
  not shared between processes. */
  do {
    ++n_waits;
    syscall(SYS_futex, &m_lock_word, FUTEX_WAIT_PRIVATE,
            mutex_state_t::LOCKED_WITH_WAITERS, 0, 0, 0);
  } while (!set_waiters());
  return n_waits;
}

/** Wakeup a waiting thread */
void signal() UNIV_NOTHROW {
  syscall(SYS_futex, &m_lock_word, FUTEX_WAKE_PRIVATE, 1, 0, 0, 0);
}
```

三个要点：

1. **`FUTEX_WAIT_PRIVATE` / `FUTEX_WAKE_PRIVATE`**：注释明说原因——InnoDB 的 mutex 不跨进程共享，用 PRIVATE 变体可跳过内核里的跨进程 hash 查找。
2. **没有 `FutexWait` 之类的封装**：`syscall` 直接写在 `wait()`/`signal()` 里。对比 `TTASEventMutex` 走"sync array + 每实例 os_event（内部又是 mutex+cond）"，futex 版把等待队列整个交给内核——这是两者性能差距的主要来源，也是"为什么 futex 更快"的答案。
3. **启用条件**：`innodb.cmake` 按 `MUTEXTYPE` 决定宏——检测到 futex 可用 → `-DMUTEX_FUTEX`；显式 event → `-DMUTEX_EVENT`；否则 `-DMUTEX_SYS`。Windows 构建落在 `MUTEX_SYS`（走 `OSTrackMutex` 包 pthread/CRITICAL_SECTION）。

**pthread —— 只剩两个落点**：`pthread_cond_t`（`os_event` 的底层，Windows 换 `CONDITION_VARIABLE`）与 `pthread_mutex_t`（`OSTrackMutex` 的 `OSMutex`）。**注意 os_event 不是 pthread_cond 的简单 wrapper**，而是在其上加了 manual-reset + signal_count 语义（见下节）。

**`std::atomic` —— 所有锁字的载体**：`TTASFutexMutex::m_lock_word`（`futex_word_t`/uint32）、`TTASEventMutex::m_owner`（`std::thread::id`）、`rw_lock_t::lock_word`（`atomic<int32>`）、`m_waiters`。8.0 已全面用它，不再有自研的原子封装。

**`sem_t`：已从 InnoDB 消失**（grep 全库无引用）——旧版本用过 POSIX 信号量，8.0 的等待统一走 os_event / futex。看到把 `sem_t` 列为 InnoDB L0 成员的资料，通常是抄的旧版本或月报。

### 主链路：两级等待的统一骨架

以 `TTASEventMutex::enter()` 为例（所有原语同构）：

```
mutex_enter(M)
  → PolicyMutex::enter(max_spins=srv_n_spin_wait_rounds, max_delay=srv_spin_wait_delay)
      → try_lock()                     快路径：一次 CAS
      → spin_and_try_lock()
          → is_free() 循环              TTAS 自旋 + 随机退避
          → yield()
          → wait(filename, line, 4)    进 sync array（见下）
              → sync_array_get_and_reserve_cell(this, ...)   登记
              → set_waiters()                                  先登记等待者
              → 再 try_lock() spin 次                          ★ 复查防丢唤醒
              → sync_array_wait_event()                        真正睡眠
```

### os_event：manual-reset 事件 + signal_count

```cpp
/** InnoDB condition variable. */
struct os_event {
  bool m_set;           /*!< this is true when the event is in the
                        signaled state, i.e., a thread does not stop
                        if it tries to wait for this event */
  int64_t signal_count; /*!< this is incremented each time the event
                        becomes signaled */
  EventMutex mutex;     /*!< this mutex protects the next fields */
  os_cond_t cond_var;   /*!< condition variable is used in waiting for the event */
};
```

**8.0 的 os_event 底层就是 pthread_cond**（Windows 用 `CONDITION_VARIABLE`）——源码里**没有**"为什么不用 pthread_cond"的注释，因为它就在用；自研的部分是"manual-reset 事件 + 信号计数"语义层。

`set()` 是 broadcast 且 sticky：

```cpp
void set() UNIV_NOTHROW {
  mutex.enter();
  if (!m_set) {
    broadcast();      /* m_set=true; ++signal_count; pthread_cond_broadcast() */
  }
  mutex.exit();
}
```

`wait_low` 的防丢信号（注释原文见「理论基础·二」）：

```cpp
void os_event::wait_low(int64_t reset_sig_count) UNIV_NOTHROW {
  mutex.enter();
  if (!reset_sig_count) reset_sig_count = signal_count;
  while (!m_set && signal_count == reset_sig_count) {
    wait();   /* pthread_cond_wait；虚假唤醒靠 while 重查 */
  }
  mutex.exit();
}
```

配套的 `os_event_wait_for<Condition>()`（`os0event.ic`）是"**自旋 + 事件**"通用模板：spin 用 `UT_RELAX_CPU()`，超限转事件等待，超时按倍增（1µs → … → 上限 100ms）——log 系统大量使用它。

### sync0arr：等待数组的职责收缩

**8.0 结构**：`sync_wait_array` 是多实例数组（`srv_sync_array_size` 个，默认 1），每个实例是一片 `sync_cell_t`：

```cpp
struct sync_cell_t {
  sync_object_t latch;      /* union: rw_lock_t* / WaitMutex* / BlockWaitMutex* */
  ulint request_type;       /* SYNC_MUTEX / SYNC_BUF_BLOCK / RW_LOCK_S / X / X_WAIT / SX */
  const char *file;  ulint line;
  std::thread::id thread_id;
  bool waiting;
  int64_t signal_count;     /* reset event 时抓取，wait 时传回防丢信号 */
  std::chrono::steady_clock::time_point reservation_time;   /* 长等待告警用 */
  uint64_t last_scan;       /* 死锁检测 DFS 的"三色"标记 */
};
```

**历史决策**（`sync0arr_impl.h` 头注释，原文翻译）：

> 为什么要实现 wait array？首先，为了让 mutex 快，我们必须自己写 mutex 实现……然后面临选择：给每把 mutex 分配一个独占的 OS event……还是用一个全局 wait 数组。在 NT 3.51 上分配 event 似乎是二次方复杂度……
> **5.0.30 起设计改变**：线程要等的事件**嵌入在被等对象里**（mutex 或 rw_lock 自己带 event）。我们仍保留全局 wait 数组，为了诊断、也为了避免无限等待——error_monitor 线程扫描它，给漏掉信号的等待者补发。

所以今天 wait array 的职责只剩两个：**① 诊断**（谁在等什么、等了多久——SEMAPHORES 段与 long-wait 报警的数据源）+ **② 防丢信号兜底**（`sync_arr_wake_threads_if_sema_free()` 每秒扫描，锁已空闲却还在睡的就强制 `os_event_set`）+ **③ 死锁检测**（debug 下 `sync_array_wait_event` 先做 DFS 找环，cell 的 `last_scan` 是三色标记）。

**signal 方向**：解锁方**直接 `os_event_set(被等对象自己的 event)`**；`sync_cell_get_event` 决定睡哪个 event：

```cpp
static os_event_t sync_cell_get_event(sync_cell_t *cell) {
  if (type == SYNC_MUTEX)          return (cell->latch.mutex->event());
  else if (type == SYNC_BUF_BLOCK) return (cell->latch.bpmutex->event());
  else if (type == RW_LOCK_X_WAIT) return (cell->latch.lock->wait_ex_event);
  else  /* RW_LOCK_S 和 RW_LOCK_X 等同一个 event */
                                   return (cell->latch.lock->event);
}
```

长等待告警：`sync_array_print_long_waits()`——等待超 4 分钟打 `"A long semaphore wait:"`，超过 fatal 阈值触发崩溃（"semaphore wait has lasted > 600 seconds" 的来源）。

### PolicyMutex 模板族（ib0mutex.h）

#### TTASEventMutex

```cpp
template <template <typename> class Policy = NoPolicy>
struct TTASEventMutex {
  bool try_lock() UNIV_NOTHROW {
    auto expected = std::thread::id{};
    return m_owner.compare_exchange_strong(expected, std::this_thread::get_id());
  }
  void exit() UNIV_NOTHROW {
    m_owner.store(std::thread::id{});
    if (m_waiters.load()) signal();     /* 有等待者才 os_event_set */
  }
  void enter(uint32_t max_spins, uint32_t max_delay, const char *filename,
             uint32_t line) UNIV_NOTHROW {
    if (!try_lock()) spin_and_try_lock(max_spins, max_delay, filename, line);
  }
 private:
  std::atomic<std::thread::id> m_owner{std::thread::id{}};  /* 锁字 = owner 线程 id */
  os_event_t m_event{};        /* 供 sync0arr 等待队列使用 */
  MutexPolicy m_policy;
  std::atomic_bool m_waiters{false};   /* "可能有线程在 sync array 等我" */
};
```

锁字直接是 **owner 的 thread id**（可诊断"谁持锁"）。`spin_and_try_lock` 的骨架（含那个著名的启发式常数）：

```cpp
void spin_and_try_lock(...) {
  for (;;) {
    if (is_free(max_spins, max_delay, n_spins)) { if (try_lock()) break; else continue; }
    else max_spins = n_spins + step;
    ++n_waits; std::this_thread::yield();
    /* The 4 below is a heuristic that has existed for a very long time now. */
    if (wait(filename, line, 4)) { n_spins += 4; break; }   /* 进 sync array */
  }
  m_policy.add(n_spins, n_waits);
}
```

`wait()` 的防丢唤醒三步（`ut0mutex.ic`，上方有 Theorem 1 证明注释）：

```cpp
sync_arr = sync_array_get_and_reserve_cell(this, ..., {filename, line}, &cell);
set_waiters();                     /* 先登记等待者 */
for (uint32_t i = 0; i < spin; ++i) {     /* 最后再试 spin 次 */
  if (try_lock()) { sync_array_free_cell(sync_arr, cell); return true; }
}
sync_array_wait_event(sync_arr, cell);    /* 真正睡眠 */
return false;
```

#### TTASFutexMutex（Linux 默认）

```cpp
enum class mutex_state_t : futex_word_t {
  UNLOCKED = 0, LOCKED = 1, LOCKED_WITH_WAITERS = 2
};
void enter(...) {
  uint32_t n_spins; lock_word_t lock = ttas(max_spins, max_delay, n_spins);  /* 先 TTAS 自旋 */
  const uint32_t n_waits = (lock == LOCKED_WITH_WAITERS ||
                            (lock == LOCKED && !set_waiters())) ? wait() : 0;
  m_policy.add(n_spins, n_waits);
}
uint32_t wait() UNIV_NOTHROW {
  do { ++n_waits;
    syscall(SYS_futex, &m_lock_word, FUTEX_WAIT_PRIVATE,
            mutex_state_t::LOCKED_WITH_WAITERS, 0, 0, 0);
  } while (!set_waiters());     /* set_waiters 返回 true = 中途锁被放掉、已归我 */
  return n_waits;
}
void exit() UNIV_NOTHROW {
  std::atomic_thread_fence(std::memory_order_acquire);
  if (state() == LOCKED_WITH_WAITERS) m_lock_word = UNLOCKED;
  else if (unlock() == LOCKED) return;   /* 无人等待，免 syscall */
  signal();                              /* FUTEX_WAKE_PRIVATE, 1 */
}
```

**"两次比较"在这里**：trylock 后检查旧状态——若已是 `LOCKED_WITH_WAITERS`（有人在等我放锁）或 `set_waiters()` 失败（CAS 时发现别人已设等待者），才 `futex_wait`；`set_waiters()` 成功意味着**中途锁被放掉、我已拿到**，跳过睡眠。futex 三态（0/1/2）正是为此设计。

**为什么比 TTASEventMutex 快**：锁字三态 uint32（无 thread::id 比较）、无每实例 os_event（省内部 mutex+cond 两级开销）、慢路径一次 syscall 且**只在有等待者时才 syscall**（`exit()` 里 `unlock() == LOCKED` 即无人等、直接返回）。

#### 类型选择与 `ib_mutex_t`

```cpp
#define UT_MUTEX_TYPE(M, P, T) typedef PolicyMutex<M<P>> T;
UT_MUTEX_TYPE(OSTrackMutex, GenericPolicy, SysMutex)
UT_MUTEX_TYPE(OSTrackMutex, BlockMutexPolicy, BlockSysMutex)
UT_MUTEX_TYPE(TTASEventMutex, GenericPolicy, SyncArrayMutex)
UT_MUTEX_TYPE(TTASEventMutex, BlockMutexPolicy, BlockSyncArrayMutex)
#ifdef MUTEX_FUTEX
typedef FutexMutex ib_mutex_t;             typedef BlockFutexMutex ib_bpmutex_t;
#elif defined(MUTEX_SYS)
typedef SysMutex ib_mutex_t;               typedef BlockSysMutex ib_bpmutex_t;
#elif defined(MUTEX_EVENT)
typedef SyncArrayMutex ib_mutex_t;         typedef BlockSyncArrayMutex ib_bpmutex_t;
#endif
```

`MUTEXTYPE` 构建选项决定宏：`futex`（检测到可用）→ `event` → `SYS`。`OSTrackMutex` 是包 `OSMutex`（pthread）的调试跟踪壳。最外层 `PolicyMutex<MutexImpl>` 负责 PFS 埋点（`pfs_begin_lock/pfs_end`）并按 `policy().enter/locked/release` 驱动策略回调；`mutex_enter(M)` 统一传入自旋参数。

### 策略层（sync0policy.h）：横切关注的注入点

```cpp
/* Do nothing */
template <typename Mutex> struct NoPolicy {
  void init(...) {}  void destroy() {}  void enter(...) {}  void add(...) {}
  void locked(...) {} void release(...) {} std::string to_string() const { return ""; }
};

/** Collect the metrics per mutex instance, no aggregation. */
template <typename Mutex>
struct GenericPolicy
#ifdef UNIV_DEBUG
    : public MutexDebug<Mutex>
#endif
{
  void add(uint32_t n_spins, uint32_t n_waits) {
    if (!m_count.m_enabled) return;
    m_count.m_spins += n_spins; m_count.m_waits += n_waits; ++m_count.m_calls;
  }
  void init(...) { ...
    meta.get_counter()->single_register(&m_count);   /* 每实例独立计数 */
  }
};

/** Track aggregate metrics policy, used by the page mutex. There are just
too many of them to count individually. */
template <typename Mutex>
class BlockMutexPolicy
#ifdef UNIV_DEBUG
    : public MutexDebug<Mutex>
#endif
{
  void init(...) { m_count = meta.get_counter()->sum_register(); }  /* 指针，聚合 */
};
```

- **组合而非 CRTP**：实现模板持有 `MutexPolicy m_policy;` 成员——实现只管"怎么锁"，策略决定"锁前后做什么"，release 版（`NoPolicy`，全空 inline 函数）零开销。
- **为什么 buf block mutex 用 `BlockMutexPolicy`**：注释原文 "too many of them to count individually"——buffer pool 数十万个页 mutex，逐实例注册计数器内存与遍历开销不可接受，改为所有实例累加到 `latch_meta_t` 的一个共享计数。
- **`MutexDebug`（UNIV_DEBUG）**：`enter/locked/release` 钩子接 `LatchDebug::check_order()` 锁序检查（`sync0policy.cc` 实现）。**`latch_order_check` 这个名字不存在**，对应物是 `LatchDebug::check_order()`。

### rw_lock_t：lock_word 单字三态编码

#### 编码表（sync0rw.cc 头注释，权威）

```cpp
constexpr int32_t X_LOCK_DECR = 0x20000000;       /* X 锁一次减这个量 */
constexpr int32_t X_LOCK_HALF_DECR = 0x10000000;  /* SX 锁减这个量 */
```

```
lock_word == X_LOCK_DECR:                          无锁
X_LOCK_HALF_DECR < lock_word < X_LOCK_DECR:        S 锁 n 个（n = X_LOCK_DECR - lock_word）
lock_word == X_LOCK_HALF_DECR:                     1 个 SX 锁
0 < lock_word < X_LOCK_HALF_DECR:                  SX + n 个 S
lock_word == 0:                                    1 个 X 锁
-X_LOCK_HALF_DECR < lock_word < 0:                 有等待写者 + n 个 S
lock_word == -X_LOCK_HALF_DECR:                    X + SX（同线程递归）
lock_word == -X_LOCK_DECR:                         2 个 X（递归）
< -(X_LOCK_DECR+X_LOCK_HALF_DECR):                 X(递归) + SX

LOCK COMPATIBILITY MATRIX
    S SX  X
 S  +  +  -
 SX +  -  -
 X  -  -  -
```

**S = 原子减 1；X = 减 `X_LOCK_DECR`；SX = 减 `X_LOCK_HALF_DECR`**。X 请求者先减占坑（成为 next-writer，负值区间阻止新 S），再等已有 S 清零——写者优先防饿死。递归规则：`recursive && writer_thread == self` 时 X/SX 可重入（首次减 `X_LOCK_DECR`，之后每次减 1）。

#### 关键字段与两个 event

```cpp
/* 速览（UNIV_DEBUG 下另继承 latch_t，有 magic_n=22643、debug_list） */
lock_word      /* atomic int32，编码见上 */
waiters        /* atomic bool："可能有线程在 sync array 等我"，唤醒协议的标志位 */
recursive / sx_recursive   /* X/SX 递归开关与计数 */
writer_thread  /* 只由 X/SX 持有者写 */
reader_thread  /* ★ 全体读者线程 id 的 XOR——恰一个读者时可还原 */
event          /* S/X 等待者共用 */
wait_ex_event  /* next-writer（X_WAIT）专用 */
count_os_wait  /* 进入 OS 等待的次数（可观测） */
```

`waiters` 的操作顺序注释（防丢唤醒）：**"Must be 1 when a writer starts waiting ... May only be reset to 0 immediately before a wake-up signal is sent"**——即"先设 flag → 复查锁 → 失败才睡；唤醒方先放锁 → reset flag → signal"。

`reader_thread` 的 XOR 技巧：所有读者 id 异或存一个字，正常情况下不可还原；但"恰一个读者"时 XOR 结果就是那个 id（`recover_if_single()`）——死锁检测和诊断常需要"这个读者是谁"。

#### S 锁快慢路径

```cpp
/* sync0rw.ic —— 快路径 */
bool rw_lock_s_lock_low(rw_lock_t *lock, ...) {
  if (!rw_lock_lock_word_decr(lock, 1, 0)) return false;   /* CAS 减1，阈值0：>0 才减 */
  lock->last_s_file_name = location.filename; lock->last_s_line = location.line;
  lock->reader_thread.xor_thing(std::this_thread::get_id());
  return true;
}

/* 慢路径 sync0rw.cc */
void rw_lock_s_lock_spin(rw_lock_t *lock, ulint pass, ut::Location location) {
lock_loop:
  os_rmb;
  while (i < srv_n_spin_wait_rounds && lock->lock_word <= 0) {   /* 自旋等写者走 */
    if (srv_spin_wait_delay) ut_delay(ut::random_from_interval_fast(0, srv_spin_wait_delay));
    i++;
  }
  if (i >= srv_n_spin_wait_rounds) std::this_thread::yield();
  if (rw_lock_s_lock_low(lock, pass, location)) { return; }
  else {
    if (i < srv_n_spin_wait_rounds) goto lock_loop;
    ++count_os_wait;
    sync_arr = sync_array_get_and_reserve_cell(lock, RW_LOCK_S, location, &cell);
    rw_lock_set_waiter_flag(lock);          /* 先设 waiters 再复查，防丢唤醒 */
    if (rw_lock_s_lock_low(lock, pass, location)) { sync_array_free_cell(...); return; }
    sync_array_wait_event(sync_arr, cell);  /* 睡眠 */
    i = 0; goto lock_loop;
  }
}
```

X 锁同构但多一步"占坑后等读者清零"：`rw_lock_x_lock_low` 做的是 `decr(X_LOCK_DECR, threshold=X_LOCK_HALF_DECR)`（**只有无 SX 无 X 时才能占坑**），占坑后 `rw_lock_x_lock_wait` 等剩余 S 归零；X_WAIT（next-writer 等读者）睡独立的 `wait_ex_event`。

解锁：S 解锁 `fetch_add(1)`，回到 0（或 `-X_LOCK_HALF_DECR`）时 `os_event_set(wait_ex_event)` 叫醒 next-writer；X 最后一次解锁时 `+= X_LOCK_DECR`，若 `waiters` 则 reset flag + `os_event_set(event)` 叫醒全体。

#### SX 的两大用户

```cpp
/* dict_index_t::lock 的注释 */
/** read-write lock protecting the upper levels of the index tree */
rw_lock_t lock;    /* B-tree 索引树锁，SMO 用 SX 模式 */

/* 字典锁 */
/** the data dictionary rw-latch protecting dict_sys */
extern rw_lock_t *dict_operation_lock;
```

btr 层注释佐证："an sx-latch on the tree"（悲观修改树结构时对树加 SX，允许并发读叶页但阻止其他结构修改）。PFS 名 `wait/synch/sxlock/innodb/dict_operation_lock`。

### latch 校验：三套体系并存（8.0.39 现状）

⚠️ **勘误：`lock_order` 不是 InnoDB 的子系统**——`storage/innobase` 下 grep `lock_order` / `debug_lock_order` **0 命中**。它在 **server 层** `sql/debug_lock_order.{h,cc}`，基于 PSI 的 "authorised arc" 有向图做 **MDL / THD lock** 的死锁检测。InnoDB **只有两套**（`LatchDebug` + 它的薄封装 `sync_check_*`）。

| | `LatchDebug` | `sync_check_*` | `lock_order`（server 层） |
|---|---|---|---|
| 位置 | `sync/sync0debug.cc:115` | `sync0debug.cc:1019-1086` | `sql/debug_lock_order.{h,cc}` |
| 生效 | `UNIV_DEBUG` + `srv_sync_debug` | 同上（内部判 `instance() != nullptr`） | `--debug-lock-order`，PSI 钩子驱动 |
| 粒度 | 每线程 `std::vector<Latched>` | 纯转发，无规则 | 全局有向图 `LO_graph` |
| 规则来源 | `check_order()` 的 per-level `switch` | 无 | 运行时注册的 arc 表 |
| InnoDB 用吗 | 是 | 是 | **否** |

**`sync_check_*` 全是薄封装**（`sync0debug.cc:1019`）：

```cpp
void sync_check_lock_validate(const latch_t *latch) {
  if (LatchDebug::instance() != nullptr) {
    LatchDebug::instance()->lock_validate(latch, latch->get_level());
  }
}
void sync_check_lock_granted(const latch_t *latch) { ... lock_granted(latch, latch->get_level()); }
void sync_check_unlock(const latch_t *latch)      { ... unlock(latch); }
void sync_check_relock(const latch_t *latch)      { ... relock(latch); }
void sync_check_lock(const latch_t *latch, latch_level_t level) {
  if (LatchDebug::instance() != nullptr) {
    ut_ad(latch->get_level() == SYNC_LEVEL_VARYING);
    ut_ad(latch->get_id() == LATCH_ID_BUF_BLOCK_LOCK);   // ★ 只有 buf_block 能用
    LatchDebug::instance()->lock_validate(latch, level);
    LatchDebug::instance()->lock_granted(latch, level);
  }
}
```

> ⚠️ **勘误**：`sync_check_lock()` **不是宏**，是 `UNIV_DEBUG` 下的**真实函数**。release 下 `sync0debug.h` **根本不声明它**，调用点被 `ut_d()` / `IF_DEBUG()` 编译掉。只有 `UNIV_LIBRARY`（embedded）分支才有 `#define sync_check_lock(A, B) ((void)0)`（`sync0rw.h:84`）。

**★ 完整链路**：`PolicyMutex::enter()` → `GenericPolicy::enter()` → `MutexDebug::enter()` → `sync_check_lock_validate()` → `LatchDebug::lock_validate()`。

**⚠️ 不是"全局严格递减"，而是 per-level switch**（`sync0debug.cc:704`）——三类规则：

```cpp
  switch (level) {
    case SYNC_NO_ORDER_CHECK:
    case SYNC_EXTERN_STORAGE:
    case SYNC_TREE_NODE_FROM_HASH:
      /* Do no order checking */
      break;                                     // ① 完全跳过

    case SYNC_LOG_SN: case SYNC_DICT: case SYNC_TABLE: /* ... 约 55 个 ... */
      /* This is the most typical case, in which we expect requested<held. */
      assert_requested_is_lower_than_held(level, latches);   // ② 严格递减
      break;

    case SYNC_FIL_SHARD: case SYNC_DBLWR: case SYNC_BUF_BLOCK:
    case SYNC_FSP: case SYNC_SEARCH_SYS: /* ... 15 个 ... */
      /* We can have multiple latches of this type therefore we
      can only check whether the requested<=held condition holds. */
      assert_requested_is_lower_or_equal_to_held(level, latches);  // ③ 允许同 level 多实例
      break;

    case SYNC_FSP_PAGE:
      ut_a(find(latches, SYNC_FSP) != nullptr);  // ④ 必须先持有另一把
      break;

    case SYNC_TRX:
      /* Either the thread must own the lock_sys global latch, or
      it is allowed to own only ONE trx_t::mutex. */
      if (find_lower_or_equal(latches, level) != nullptr) {
        assert_requested_is_lower_or_equal_to_held(level, latches);
        ut_a(find(latches, SYNC_LOCK_SYS_GLOBAL) != nullptr);
      }
      break;
    ...
    case SYNC_INDEX_TREE:
      assert_all_held_are_above(latches, level, SYNC_TREE_NODE - 1);  // ⑤ 自定义下界
      break;
  }
```

```cpp
  void assert_requested_is_lower_than_held(latch_level_t requested_level,
                                           const Latches *latches) const {
    assert_all_held_are_above(latches, requested_level, requested_level);
  }
  void assert_requested_is_lower_or_equal_to_held(latch_level_t requested_level,
                                                  const Latches *latches) const {
    assert_all_held_are_above(latches, requested_level, requested_level - 1);
    //                                                              ★ 下界 -1 = "允许相等"
  }

void LatchDebug::assert_all_held_are_above(const Latches *latches,
                                           latch_level_t requested_level,
                                           ulint forbidden_level) const {
  const Latched *latched = find_lower_or_equal(latches, latch_level_t(forbidden_level));
  if (latched != nullptr) {
    crash(latches, latched, requested_level);   // ★★ 违规 = 直接 abort
  }
}
```

**兜底检查只看"最后一把"**（`sync0debug.cc:227`）：

```cpp
  void lock_validate(const latch_t *latch, latch_level_t level) UNIV_NOTHROW {
    /* Ignore diagnostic latches, starting with '.' */
    if (*latch->get_name() != '.' && latch->get_level() != SYNC_LEVEL_VARYING) {
      Latches *latches = check_order(latch, level);
      if (!(latches->empty() || level == SYNC_LEVEL_VARYING ||
            level == SYNC_NO_ORDER_CHECK ||
            latches->back().get_level() == SYNC_NO_ORDER_CHECK ||
            latches->back().m_latch->get_level() == SYNC_LEVEL_VARYING ||
            latches->back().get_level() >= level)) {
        ib::error(ER_IB_LOCK_VALIDATE_LATCH_ORDER_VIOLATION)
            << "LatchDebug::lock_validate() latch order violation. level="
            << level << ...;
        ut_error;                              // ★ 直接崩
      }
    }
  }
```

> ★ **数据结构**：每线程一个 `std::vector<Latched>`，按**获取顺序** `push_back`；因为检查通过才允许 push，所以"按获取顺序" ≡ "按 level 降序"。`unlock()` 时**从后往前找**（LIFO，应对同 latch 多层递归）。
> ★ 违规信息用 `print_latches()` 打印整个栈——这是排查 latch order 违规的主要输出。

**level 是静态绑定到 `latch_id_t` 的**（⚠️ **不存在 `rw_lock_set_level()`**）：

```cpp
  latch_level_t get_level() const UNIV_NOTHROW {
    ut_a(m_id != LATCH_ID_NONE);
    return (sync_latch_get_level(m_id));   // ★ 查 LatchMeta 表，运行时不可改
  }
```

> ★ **唯一的运行时 level 是 buffer page latch**（`SYNC_LEVEL_VARYING`），通过 `buf_block_dbg_add_level()` → `sync_check_lock(&block->lock, level)` 传入。

**`latch_level_t` 全集**（`sync0types.h:199`，约 100 个值）——顺序即层级，从低到高：`SYNC_UNKNOWN=0, SYNC_MUTEX=1, RW_LOCK_SX, RW_LOCK_X_WAIT, RW_LOCK_S, RW_LOCK_X, RW_LOCK_NOT_LOCKED, SYNC_LOCK_FREE_HASH, SYNC_MONITOR_MUTEX, SYNC_ANY_LATCH, SYNC_FIL_SHARD, ... SYNC_DICT_OPERATION, SYNC_AHI_ENABLED, SYNC_TRX_I_S_LAST_READ, SYNC_TRX_I_S_RWLOCK, SYNC_RECV_WRITER, SYNC_LEVEL_VARYING, SYNC_NO_ORDER_CHECK, SYNC_LEVEL_MAX = SYNC_NO_ORDER_CHECK`。
> `LatchDebug` 构造里 `ut_ad(m_levels.size() == SYNC_LEVEL_MAX + 1)` 保证枚举与名字表一一对应。

| 体系 | 位置 | 校验什么 | 状态 |
|---|---|---|---|
| **`LatchDebug`**（`latch_t` 基类） | `sync0debug.cc` + `sync0types.h` | 按 `latch_level_t` 金字塔（头部有整幅 ASCII 锁序图：Dict mutex → 索引树 → 叶页 → trx sys → buf pool → log）检查加锁顺序，违反打 `"latch order violation"` 后 crash | UNIV_DEBUG |
| **`ut::Stateful_latching_rules`** | `ut0stateful_latching_rules.h` | **状态机 × 持锁集合**的声明式校验：把问题建模为"状态图，边标注所需 latch 子集"。唯一用户是 `buf_page_t::io_fix`（`Buf_io_fix_latching_rules`），提供"读状态时持有的锁足以冻结状态"的断言 | UNIV_DEBUG |
| **LOCK ORDER 工具** | `sql/debug_lock_order.cc`（`WITH_LOCK_ORDER` 构建选项） | 用 PFS 名建全局有向图（`lock_order_dependencies.txt`），运行时校验加锁序列无环 → 保证无死锁；InnoDB sxlock 被建模为三态（注释 "Shared exclusive locks are recursive... SX + X counts as X"） | 独立构建选项 |

**分层归属（三套体系谁管谁）**：

- **LatchDebug** = InnoDB 专属（UNIV_DEBUG，只覆盖 InnoDB 自研原语，锁序金字塔是 InnoDB 内部知识）
- **LOCK ORDER 工具** = **server 层跨层**（`sql/debug_lock_order.cc`，覆盖 server + InnoDB **所有 PFS 锁**——输入是 PFS instrument 名，不关心锁是 mysys 还是 InnoDB 实现）
- **PFS** = 两者共享的**锁命名层**：LOCK ORDER 以 PFS 名为图节点；LatchDebug 校验的锁在 PFS 也有对应 instrument（`wait/synch/mutex|rwlock|sxlock/innodb/*`），供运行时观测竞争（见「可观测性」章）。锁序校验的演进方向就是"从 InnoDB 内部金字塔走向以 PFS 名为通用坐标"。

### 桥接函数：`buf_block_dbg_add_level`——页级锁怎么进入 LatchDebug

LatchDebug 校验的是 `latch_t`，但 InnoDB 绝大多数加锁对象是**缓冲页的 rw-lock**（`buf_block_t::lock`）。把页锁接入锁序体系的就是这个看着不起眼的函数。第一层只是个空壳（`buf0buf.ic`）：

```cpp
// UNIV_DEBUG 下：
static inline void buf_block_dbg_add_level(
    buf_block_t *block,  /*!< buffer page where we have acquired latch */
    latch_level_t level) /*!< latching order level */
{
  sync_check_lock(&block->lock, level);
}
#else                                    /* UNIV_DEBUG */
#define buf_block_dbg_add_level(block, level) /* nothing */
#endif
```

**release 构建下它是空宏**——锁序检查零成本，生产环境连函数调用都没有。这是"策略层零成本注入"（见概述）在页锁上的具体落地。

第二层 `sync_check_lock`（`sync0debug.cc`）才有信息量：

```cpp
void sync_check_lock(const latch_t *latch, latch_level_t level) {
  if (LatchDebug::instance() != nullptr) {
    ut_ad(latch->get_level() == SYNC_LEVEL_VARYING);      // ★ 页锁的 level 是"可变的"
    ut_ad(latch->get_id() == LATCH_ID_BUF_BLOCK_LOCK);    // ★ 只允许缓冲页锁走这条路
    LatchDebug::instance()->lock_validate(latch, level);  // 校验顺序
    LatchDebug::instance()->lock_granted(latch, level);   // 记入本线程持锁栈
  }
}
```

★ 两条断言把设计说清楚了：**"level 可变"这个机制就是专为缓冲页锁准备的**（只有 `LATCH_ID_BUF_BLOCK_LOCK` 能进）。原因同一份文件里写着：

```
  /* NOTE that there is a problem with _NODE and _LEAF levels: if the
  B-tree height changes, then a leaf can change to an internal node
  or the other way around. We do not know at present if this can cause
  unnecessary assertion failures below. */
```

即 **B-tree 高度变化会让叶子页变成内节点（或反之）**，同一个页在不同时刻的"角色层级"会变，无法静态指定——这才是 `SYNC_LEVEL_VARYING` 存在的动机。

第三层是真正的判定，也在这里暴露一个反直觉的结果：

```cpp
  void lock_validate(const latch_t *latch, latch_level_t level) UNIV_NOTHROW {
    /* Ignore diagnostic latches, starting with '.' */
    if (*latch->get_name() != '.' && latch->get_level() != SYNC_LEVEL_VARYING) {
      ut_ad(level != SYNC_LEVEL_VARYING);
      Latches *latches = check_order(latch, level);

      if (!(latches->empty() || level == SYNC_LEVEL_VARYING ||
            level == SYNC_NO_ORDER_CHECK ||
            latches->back().get_level() == SYNC_NO_ORDER_CHECK ||
            latches->back().m_latch->get_level() == SYNC_LEVEL_VARYING ||
            latches->back().get_level() >= level)) {
        ib::error(ER_IB_LOCK_VALIDATE_LATCH_ORDER_VIOLATION)
            << "LatchDebug::lock_validate() latch order violation. ...";
        ut_error;                                   // ★ 违规即崩溃
      }
    }
  }

  void lock_granted(const latch_t *latch, latch_level_t level) UNIV_NOTHROW {
    if (*latch->get_name() != '.' && latch->get_level() != SYNC_LEVEL_VARYING) {
      latches->push_back(Latched(latch, level));     // 入栈
    }
  }
```

★ **注意进入条件**：`latch->get_level() != SYNC_LEVEL_VARYING`。而 `sync_check_lock` 传入的正是 `SYNC_LEVEL_VARYING` 的页锁 ⇒ **两个函数的主体都被跳过**。也就是说，**"页锁参与金字塔校验"是错的**——它既不校验也不入栈。判断依据是代码条件本身，不是函数名。

真正的顺序校验发生在**固定 level 的锁**（mutex / 非页 rw-lock）上，规则是 `latches->back().get_level() >= level`——**栈顶（最近持有的）层级必须 ≥ 新请求的层级**，即只能"从高层往低层"加锁。五个豁免条件：

| 豁免 | 含义 |
|---|---|
| `latches->empty()` | 线程当前没持任何锁 |
| `level == SYNC_NO_ORDER_CHECK` | 新锁**显式声明**不参与顺序检查 |
| 栈顶 `== SYNC_NO_ORDER_CHECK` | 上一个锁是豁免的 → 后续不再比对 |
| 栈顶锁是 `SYNC_LEVEL_VARYING` | 上一个是页锁 → 不比对 |
| `level == SYNC_LEVEL_VARYING` | 新锁本身可变 → 不比对 |

另外两类被跳过：名字以 **`.` 开头的是"诊断锁"**（不参与）；以及 `check_order` 里单独列出的豁免层级——`SYNC_NO_ORDER_CHECK`、`SYNC_EXTERN_STORAGE`（LOB 外存）、`SYNC_TREE_NODE_FROM_HASH`（AHI 哈希命中取页）都明确 `/* Do no order checking */`。

**level 由调用方按语义指定**，同一个 `buf_block_t` 在不同场景下可以是不同 level——因为"锁序"约束的是**角色**而非对象：

| level | 含义 | 典型调用点 |
|---|---|---|
| `SYNC_TREE_NODE` | 索引树节点页（内部/叶子） | `btr_free_root_check`、`btr_page_get_father_block` 等 B-tree 路径 |
| `SYNC_EXTERN_STORAGE` | 外部存储（LOB/BLOB）页 | `lob0lob.cc`、`lob0ins.cc`、`lob0del.cc` |
| `SYNC_TRX_UNDO_PAGE` | undo 页 | `trx0undo.ic` |
| `SYNC_NO_ORDER_CHECK` | **显式豁免**：本锁不参与顺序检查 | `fut0fut.ic`、以及"从 BP 拿页但只做短暂操作"的路径 |

⇒ 所以在 B-tree / DDL 代码里看到 `buf_block_dbg_add_level(block, SYNC_TREE_NODE)`，它表达的是**角色语义**（"这个页此刻按树节点对待"），而不是"请校验它"——release 下整行消失，debug 下也因 VARYING 而跳过。**它是给读代码的人看的角色标注**，这解释了为什么它遍布 B-tree / DDL 路径却几乎从不触发 violation。

### LatchDebug 的锁序金字塔：机制详解

**官方锁序图**（sync0types.h 头部注释，完整 ASCII 金字塔，从"必须先拿"到"最后拿"，此处精选）：

```
Dictionary mutex              字典锁 dict_sys（最高）
  ↓
Dictionary header
  ↓
Secondary index tree latch    index->lock（树锁，SMO 用 SX）
  ↓
Secondary index non-leaf / leaf
  ↓
Clustered index tree latch / non-leaf / leaf
  ↓
  ...（undo / rseg / purge / lock_sys 分片 / trx_sys 等十余层）
  ↓
Buffer pool mutexes
  ↓
Log mutex
  ↓
Any other latch
  ↓
Memory pool mutex             内存池锁（最低）
```

**方向规则**：`latch_level_t` 枚举（sync0types.h）**值越大 = 层级越高 = 必须先拿**（证据：`SYNC_POOL` 排在 `SYNC_DICT` 前、值更小，而图中 Dictionary mutex 在 Buffer pool mutexes 之上）。拿锁时校验"**后拿的 level ≤ 已持有的 level**"（先高后低），违反报 `"latch order violation"` 后 crash。

**注册与校验入口**（sync0debug.cc）：`sync_check_lock_validate`（拿锁时校验）→ `sync_check_lock_granted`（把锁挂进线程的持锁链）→ `sync_check_unlock`（摘除）。策略层在 `enter` 时自动调前两个、`release` 时调最后一个——业务代码透明，只需在 `mutex_create` 时指定 level。

**★ `sync_check_lock` 只为"可变层级锁"存在**：

```cpp
void sync_check_lock(const latch_t *latch, latch_level_t level) {
  ut_ad(latch->get_level() == SYNC_LEVEL_VARYING);      // 固定层级锁不适用
  ut_ad(latch->get_id() == LATCH_ID_BUF_BLOCK_LOCK);    // 全库唯一：buf block 页锁
  LatchDebug::instance()->lock_validate(latch, level);
  LatchDebug::instance()->lock_granted(latch, level);
}
```

`buf_block_t::lock` 是**全库唯一**可以"每次声明层级"的锁——它的固定层级是 `SYNC_LEVEL_VARYING`，必须每次拿锁后用 `buf_block_dbg_add_level(block, level)` 声明这次扮演的角色（同一页帧在不同场景是 `SYNC_TREE_NODE`/`SYNC_IBUF_TREE_NODE`/`SYNC_FSP_PAGE`/`SYNC_TRX_UNDO_PAGE`…，机制剖析见 [`buffer_pool.md`](../../../innodb/buffer_pool.md) 的「`buf_block_dbg_add_level`」节）。其他锁层级固定，注册时直接读 `get_level()`，无此问题。

**check_order 的三层 case 模式**（sync0debug.cc 的 `lock_validate` 按 level switch）：

```cpp
switch (level) {
  case SYNC_NO_ORDER_CHECK:
  case SYNC_EXTERN_STORAGE:
  case SYNC_TREE_NODE_FROM_HASH:
    /* Do no order checking */            // ① 免检组：恢复期/AHI 命中等语境
    break;

  case SYNC_DICT: ... case SYNC_POOL: ... case SYNC_LOG_*: ...
    // ② 典型组："requested < held"
    assert_requested_is_lower_than_held(level, latches);
    break;

  case SYNC_TREE_NODE:
    if (find(latches, SYNC_INDEX_TREE) == nullptr &&
        find(latches, SYNC_DICT_OPERATION) == nullptr)
      assert_requested_is_lower_or_equal_to_held(level, latches);  // ③ 豁免组
    break;

  case SYNC_TREE_NODE_NEW:
    ut_a(find(latches, SYNC_FSP_PAGE) != nullptr);   // 新页分配必须已在 fsp 保护下
    break;

  case SYNC_IBUF_TREE_NODE:
    if (find(latches, SYNC_IBUF_INDEX_TREE) == nullptr)
      assert_requested_is_lower_or_equal_to_held(level, latches);
    break;
  ...
}
```

**豁免分支的精髓**（case SYNC_TREE_NODE）：规则不是死的"必须按序"，而是"**要么按序，要么持有足够高的保护锁**"——已持 `index->lock`（SYNC_INDEX_TREE）或 `dict_operation_lock`（SYNC_DICT_OPERATION）时，拿树节点免查（树结构已被更高层级锁保护，不存在乱序并发）。**高层级锁的持有是低级乱序的豁免凭证**——这是锁序金字塔与"锁保护伞"思想的结合点。同理 `SYNC_TREE_NODE_NEW` 强制要求已持 `SYNC_FSP_PAGE`（新页分配必在 fsp latch 保护下）。

**为什么值得单独讲**：这套 case 规则是**手工维护**的——枚举注释明说 "If you modify these, you have to also update LatchDebug internals in sync0debug.cc"。每加一种锁层级，人肉在 check_order 加 case、且要保证与金字塔图注释同步。这正是三套体系演进要解决的痛点（见下节「三套锁序校验的演进」）。

**勘误**：`latch_t` **不是**"buf 页 latch 原型"——它是 UNIV_DEBUG 下所有可校验 latch 的调试基类（持 `latch_id_t`、虚 `to_string()`、`get_level()` 查金字塔层级），`rw_lock_t` 在 DEBUG 下继承它。`Stateful_latching_rules` 也**不是**通用机制，目前是 io_fix 状态机的专用校验器。三套体系的演进方向：从"人工维护 level 枚举"走向"声明式/数据驱动校验"。

旧的 `latch_add_to_history` **不存在**（已重构），现行 API 是 `sync_check_lock_validate / sync_check_lock_granted / sync_check_unlock`，由策略层（mutex）与 `rw_lock_add_debug_info`（rwlock）驱动。

### 表空间锁：fil 三级锁体系（fil_space_t::latch 详解）

fil 是 InnoDB "所有表空间文件 I/O 的统一入口与元数据中心"，用**三级锁**协调并发。本节从同步原语视角讲锁本身（保护对象 / 锁序 / 为什么分层）；功能视角的 `Fil_shard` 分片与 `do_io` 链路见 [`fil.md`](../../../innodb/fil.md)。

**两条主链路先看全景**：

```
读路径（高频，只碰 shard mutex，不碰 space latch）：
  buf 页读取 → fil_io → Fil_shard::do_io
    mutex_acquire()                          ← 持 shard mutex 查 space
    ├─ get_space_by_id(space_id)             ← 查 m_spaces hash
    ├─ stop_new_ops 检查（删除中 → DB_TABLESPACE_DELETED）
    ├─ get_file_for_io / prepare_file_for_io
    └─ mutex_release() → 发起 I/O（不持锁）

DDL truncate（低频，三步持锁）：
  space_prepare_for_truncate
    ├─ Step1: stop_new_ops=true → 20ms 轮询等 pending ops/IO 归零（shard mutex 反复拿/放）
  mutex_acquire()
    ├─ Step2: bump_version()                ← 旧页惰性失效
    ├─ Step3: os_file_truncate + size 更新
    └─ stop_new_ops = false
  mutex_release()

扩展（fsp 页分配链，space latch X 的唯一入口）：
  mtr_x_lock_space(space)        [X space latch, SYNC_FSP]
    → fsp_try_extend_data_file_with_pages
      → Fil_shard::space_extend → mutex_acquire()   [SYNC_FIL_SHARD]
```

#### 三级锁的真实结构（先纠正一个流传说法）

```cpp
// fil0fil.cc Fil_system —— 全局没有独立 mutex，靠"锁全部 shard"实现
void mutex_acquire_all() const { for (auto shard : m_shards) shard->mutex_acquire(); }
void mutex_release_all() const { for (auto shard : m_shards) shard->mutex_release(); }

// Fil_shard —— 68 把 shard mutex
Fil_shard::Fil_shard(size_t shard_id) : m_id(shard_id), ... {
  mutex_create(LATCH_ID_FIL_SHARD, &m_mutex);   // SYNC_FIL_SHARD；PFS key = fil_system_mutex_key
}

// fil_space_t —— 每个表空间一把 rw_lock_t
/** Latch protecting the file space storage allocation */
rw_lock_t latch;                                // LATCH_ID_FIL_SPACE；SYNC_FSP；PFS key = fil_space_latch_key
```

**latch 的完整生命周期**（`Fil_shard::space_create` 里的创建段，完整代码）：

```cpp
space = static_cast<fil_space_t *>(
    ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sizeof(*space)));
space->initialize();
...
space->magic_n = FIL_SPACE_MAGIC_N;
...
rw_lock_create(fil_space_latch_key, &space->latch, LATCH_ID_FIL_SPACE);

#ifndef UNIV_HOTBACKUP
if (space->purpose == FIL_TYPE_TEMPORARY) {
  ut_d(space->latch.set_temp_fsp());   // 临时表空间的锁序特殊规则（m_temp_fsp）
}
#endif
space_add(space);                      // 挂进 shard 的 m_spaces hash
```

销毁在 `space_free_low`（`rw_lock_free(&space->latch)`），**前置引用计数校验**：临时/undo space 必须 `has_no_references()`（所有 buffer pool 页引用清零）才 free，否则复用 space_id 会撞残留页。

**⚠️ 没有独立的 `fil_system_mutex`**——PFS 里那个 `fil_system_mutex_key` 是**分片 mutex 共用的 instrument 名**（历史命名），不是真的有一把全局 mutex。全局操作（建表空间、分配新 space_id）用 `mutex_acquire_all()` 顺序锁全部 68 个 shard——低频，代价可接受。

`fil_space_t::latch` 的创建在 `Fil_shard::space_create`（`zalloc + initialize` 后、`space_add` 挂 hash 前），销毁在 `space_free_low`（`rw_lock_free(&space->latch)`）。临时表空间额外 `latch.set_temp_fsp()`（`m_temp_fsp`，intrinsic 临时表的锁序特殊规则）。

#### 各自保护什么

| 锁 | 保护对象 |
|---|---|
| `Fil_shard::m_mutex`（68 把） | `m_spaces`（space 集合）、`m_names`（名字映射）、`m_deleted_spaces`、`m_LRU`（文件句柄 LRU）、`m_unflushed_spaces`、`m_modification_counter`；以及 `fil_space_t` 的 `n_pending_ops`/`size`/`files` 等**元数据** |
| `fil_space_t::latch`（每 space 一把） | **fsp 存储分配元数据**：free list、extent descriptor、header 页——通过 mtr 在页分配时 X 持锁 |
| （全局）`mutex_acquire_all` | `m_max_assigned_id`、`m_shards` 构造期、启动扫描的 `m_dirs`/`m_old_paths` |

**纠偏**：`size` 字段**没有**"由 latch 保护"的注释，实际在 shard mutex 下读写（`fil_space_get_size` 持 shard mutex 读）；`compression_type`/`purpose`/`files` 也是 shard mutex 保护的元数据。`space->latch` 只覆盖**存储分配**这一件低频写。

#### 锁序：不是"全局→局部"，是 space → shard 递减

`latch_level_t` 里 `SYNC_FSP`（space latch）的数值**大于** `SYNC_FIL_SHARD`（shard mutex），而 InnoDB 要求**严格递减**获取（`sync0debug.cc` 注释原文 "strictly descending sequence cannot have a loop"）。所以嵌套链是：

```cpp
// fsp0fsp.cc fsp_try_extend_data_file_with_pages（调用方已 mtr_x_lock_space 持 X）
bool success = fil_space_extend(space, page_no + 1);
//   → Fil_shard::space_extend → mutex_acquire()   ← 先 space latch(X, SYNC_FSP) 再 shard mutex(SYNC_FIL_SHARD)
```

**先拿 space latch、后拿 shard mutex**——与直觉的"先全局后局部"相反。全库**没有**"先持 shard mutex 再嵌套拿 space latch"的反向路径（`fil_set_autoextend_size` 是"先 `space_acquire` 计数 → 释放 shard mutex → 再 X 锁 latch"的**串行两段**，不是嵌套）。

#### 关键函数逐段剖析

##### `Fil_shard::do_io`：读路径的锁检查段

```cpp
dberr_t Fil_shard::do_io(const IORequest &type, bool sync, ...) {
  ...
  /* Reserve the mutex and make sure that we can open at
  least one file while holding it, if the file is not already open */
  mutex_acquire();

  auto space = get_space_by_id(page_id.space());

  if (space == nullptr ||
      (req_type.is_read() && !sync && space->stop_new_ops)) {
    mutex_release();
    /* A read request can happen because the reader thread has gone through
    the ::stop_new_ops check in buf_page_init_for_read() before the flag was
    set and has not yet incremented ::n_pending when we checked it above. */
    return DB_TABLESPACE_DELETED;
  }
  ...
  fil_node_t *file;
  auto err = get_file_for_io(space, &page_no, file);
  ...
  if (!prepare_file_for_io(file)) { ... }
  mutex_release();
  ...
}
```

逐段：① `mutex_acquire` 一次覆盖"查 space → 定位文件 → 打开文件"整段元数据操作（注释：至少保证能打开一个文件再放锁）；② 删除中（`stop_new_ops`）的**异步**读直接拒绝——但**同步**读（`sync==true`，如恢复期）豁免；③ 这是注释明说的竞态防线：读线程可能已在 `buf_page_init_for_read` 穿过 `stop_new_ops` 检查，所以 `do_io` 里再查一次。

##### `Fil_shard::space_extend`：扩展的完整持锁协议

```cpp
bool Fil_shard::space_extend(fil_space_t *space, page_no_t size) {
  ut_ad(!srv_read_only_mode || fsp_is_system_temporary(space->id));
  fil_node_t *file;
  bool success = true;

  for (;;) {
    mutex_acquire();
    space = get_space_by_id(space->id);

    if (size < space->size) {          /* ① 别人已扩到位 */
      mutex_release();
      return true;
    }
    file = &space->files.back();

    if (!file->is_being_extended) {    /* ② 抢到扩展权 */
      /* Mark this file as undergoing extension. This flag is used to
      synchronize threads to execute space extension in order. */
      file->is_being_extended = true;
      break;
    }

    /* ③ 别的线程在扩：释放 mutex 轮询等（注释承认"整个模块到处是轮询，
       本该用 event 机制"） */
    mutex_release();
    if (!tbsp_extend_and_initialize) std::this_thread::sleep_for(20us);
    else std::this_thread::sleep_for(100ms);
  }

  if (!prepare_file_for_io(file)) {    /* .ibd 缺失 */
    ut_a(file->is_being_extended);
    file->is_being_extended = false;
    mutex_release();
    return false;
  }
  ut_a(file->is_open);
  ...
  /* At this point it is safe to release the shard mutex. No other thread can
  rename, delete or close the file because we have set the file->in_use flag. */
  mutex_release();

  /* 真正的扩文件在锁外做：posix_fallocate + 可选写 redo（临时表空间/系统表空间
     不写 redo——注释：临时表空间启动时重建、系统表空间不 resize） */
  ...os_file_set_size(...)...
}
```

逐段：① 进入即查"是否已被别人扩到位"（size 已在 shard mutex 下更新）；② `is_being_extended` 是**扩展权的互斥旗标**（注意它不是锁，靠 shard mutex 保护检查-置位的原子性）；③ 竞争者释放锁轮询——持锁者只锁"查状态 + 置旗标 + 准备文件"这一段，**真正的 `posix_fallocate`/写 redo 在锁外做**（`file->in_use` 保活，注释原文 "No other thread can rename, delete or close the file"）。

##### `Fil_shard::space_truncate`：三步持锁

```cpp
bool Fil_shard::space_truncate(space_id_t space_id, page_no_t size_in_pages) {
  fil_space_t *space{};

  /* Step-1: Prepare tablespace for truncate. This involves stopping all the
  new operations + IO on that tablespace. Any future attempts to flush will be
  ignored and pages discarded. */
  if (space_prepare_for_truncate(space_id, space) != DB_SUCCESS) {
    return false;
  }

  mutex_acquire();

  /* Step-2: Mark the tablespace pages in the buffer pool as stale by bumping
  the version number of the space. Those stale pages will be ignored and freed
  lazily later. This includes AHI, for which entries will be removed on
  buf_page_free_stale*() -> buf_LRU_free_page -> btr_search_drop_page_hash_index() */
  space->bump_version();

  /* Step-3: Truncate the tablespace and accordingly update the fil_space_t
  handler that is used to access this tablespace. */
  ut_a(space->files.size() == 1);
  auto &file = space->files.front();
  if (!file.is_open) {
    if (!open_file(&file)) { mutex_release(); return false; }
  }
  space->size = file.size = size_in_pages;
  bool success = os_file_truncate(file.name, file.handle, 0);
  if (success) {
    os_offset_t size = size_in_pages * UNIV_PAGE_SIZE;
    success = os_file_set_size(file.name, file.handle, 0, size, true);
    if (success) space->stop_new_ops = false;
  }
  mutex_release();
  return success;
}
```

三步都在注释里自带说明：Step-1（锁外）先把 `stop_new_ops` 置位并等所有在途操作归零；Step-2（锁内）`bump_version` 让 buffer pool 旧页**惰性失效**（含 AHI 条目，失效链一路到 `btr_search_drop_page_hash_index`）；Step-3（锁内）真截断 + 更新 `size` + 放行新操作。注意 `bump_version` 的前置 `ut_a(stop_new_ops); ut_a(!m_deleted);`——只能在已 stop_new_ops 且未删除时 bump。

##### `Fil_shard::wait_for_pending_operations`：两段 20ms 轮询

```cpp
dberr_t Fil_shard::wait_for_pending_operations(space_id_t space_id,
                                               fil_space_t *&space,
                                               char **path) const {
  mutex_acquire();
  fil_space_t *sp = get_space_by_id(space_id);
  if (sp != nullptr) sp->stop_new_ops = true;   /* ① 关闸：新操作不许进 */
  mutex_release();

  /* ② Check for pending operations. */
  ulint count = 0;
  do {
    mutex_acquire();
    sp = get_space_by_id(space_id);
    count = space_check_pending_operations(sp, count);
    mutex_release();
    if (count > 0) std::this_thread::sleep_for(std::chrono::milliseconds(20));
  } while (count > 0);

  /* ③ Check for pending IO. */
  *path = nullptr;
  do {
    mutex_acquire();
    sp = get_space_by_id(space_id);
    if (sp == nullptr) { mutex_release(); return DB_TABLESPACE_NOT_FOUND; }
    ut_a(sp->files.size() == 1);
    const fil_node_t &file = sp->files.front();
    count = check_pending_io(sp, file, count);
    if (count == 0) *path = mem_strdup(file.name);
    mutex_release();
    ...
  } while (count > 0);
  ...
}
```

逐段：① 关闸后**必须反复拿/放 mutex 轮询**，因为 pending IO 的完成需要 IO 线程推进，持锁死等会死锁；② 先等"操作"（ibuf merge/读页等）归零，③ 再等"IO"（`n_pending_ios`/`n_pending_flushes`/`is_being_extended`）归零。两段分开等是因为两者的归零路径不同。

#### ★ 三个反直觉结论（纠正流传说法）

1. **`fil_space_t::latch` 8.0 几乎只以 X 模式持有**——全库没有 `rw_lock_s_lock(&space->latch)` 调用。所谓"读多写少用 S 锁"**不成立**：真正承担"读多写少"的是 shard mutex（短临界区元数据读），space latch 只覆盖低频存储分配写。它保留 rwlock 形态的原因是历史（经典 "File space management latch" 一直就是 rwlock）+ LatchDebug 集成（`rw_lock_create(..., LATCH_ID_FIL_SPACE)` 进入 `SYNC_FSP` 层级校验 + `set_temp_fsp()` 钩子）。
2. **没有独立全局 mutex**（见上），全局操作 = `mutex_acquire_all` 锁全部 shard。
3. **锁序是 space → shard**，不是 shard → space（层级数值 `SYNC_FSP > SYNC_FIL_SHARD`）。

#### 引用计数保活：`n_pending_ops` 代替长期持锁

DDL 删除与后台 IO/读页并发时，不靠长期持 shard mutex 保活对象，而是引用计数：

```cpp
fil_space_t *Fil_system::space_acquire(space_id_t space_id) {
  auto shard = shard_by_id(space_id);
  shard->mutex_acquire();
  fil_space_t *space = shard->get_space_by_id(space_id);
  if (space && !shard->space_acquire(space)) space = nullptr;  // ++n_pending_ops
  shard->mutex_release();
  return space;   // 释放 mutex 后指针仍有效：n_pending_ops>0 阻止 DDL free
}
```

`space_release` 在无锁下 `--n_pending_ops`；删除方置 `stop_new_ops=true` 后 `wait_for_pending_operations`（20ms 轮询）等计数归零，才 `set_deleted` → `bump_version()`（旧页惰性失效）。`bump_version` 的前置 `ut_a(stop_new_ops); ut_a(!m_deleted);`——**只能在已 stop_new_ops 且未删除时 bump**。

#### 与 LatchDebug 金字塔

`SYNC_FSP` 在 `sync0types.h` 头部 ASCII 锁序图里对应 "**File space management latch**"（注释："If a mini-transaction must allocate several file pages, it can do that, because it keeps the x-latch to the file space management in its memo"）。`SYNC_FSP_PAGE`（页帧临时层级）是它的下一级，校验强制**必须先持 `SYNC_FSP`**：

```cpp
case SYNC_FSP_PAGE:
  ut_a(find(latches, SYNC_FSP) != nullptr);   // 新页分配必在 fsp latch 保护下
  break;
```

区分：`SYNC_FSP` = `fil_space_t::latch` 的层级；`SYNC_FSP_PAGE` = 页帧锁（`buf_block_t::lock`）在表空间页分配场景下的**临时调试层级**（`buf_block_dbg_add_level(block, SYNC_FSP_PAGE)`）。

#### 已知坑

1. **`stop_new_ops` 挡不住已穿过检查的读和任意时刻的写**（`space_delete` 注释原文）：读线程可能在 `buf_page_init_for_read` 穿过 `stop_new_ops` 检查后才置位，写请求根本不查这个 flag——所以删除前必须 `buf_LRU_flush_or_remove_pages` + 等 `n_pending_ios`/`n_pending_flushes`/`is_being_extended` 全归零。
2. **`is_being_extended` 串行化扩展**：`space_extend` 循环里持 shard mutex 查 `is_being_extended`，被占则释放 mutex + sleep 重试（"used to synchronize threads to execute space extension in order"）。
3. **`space_free_low` 前引用计数校验**：临时/undo space 必须 `has_no_references()`（所有 buf 页引用清零）才 free，否则复用 space_id 撞残留页。

### 模块闭环：下游靠什么、上游谁在用

**`os_event` 完整结构体**（`os/os0event.cc:63`）——⚠️ 真定义在 **.cc** 里，`os0event.h` 只有前向声明：

```cpp
struct os_event {
  void destroy() UNIV_NOTHROW {
    pthread_cond_destroy(&cond_var);
    mutex.destroy();
    ut_ad(n_objects_alive.fetch_sub(1) != 0);
  }

  void set() UNIV_NOTHROW {
    mutex.enter();
    if (!m_set) { broadcast(); }        // ★ 幂等
    mutex.exit();
  }

  bool try_set() UNIV_NOTHROW {
    if (mutex.try_lock()) {             // ★ 拿不到锁直接放弃，不阻塞（signal handler 用）
      if (!m_set) { broadcast(); }
      mutex.exit();
      return (true);
    }
    return (false);
  }

  int64_t reset() UNIV_NOTHROW {
    mutex.enter();
    if (m_set) { m_set = false; }
    int64_t ret = signal_count;
    mutex.exit();
    return (ret);                       // ★ 调用者当 reset_sig_count
  }

  bool is_set() const UNIV_NOTHROW { return (m_set); }   // ★ 无锁读，仅诊断

 private:
  void init() UNIV_NOTHROW {
    mutex.init();
    pthread_cond_init(&cond_var, &cond_attr);   // ★ cond_attr 全局共享，含 CLOCK_MONOTONIC
    ut_d(n_objects_alive.fetch_add(1));
  }

  void wait() UNIV_NOTHROW {
    pthread_cond_wait(&cond_var, mutex);        // ★ 就是 pthread_cond_wait
  }

  bool timed_wait(const timespec *abstime);     // ★ 非 Windows：绝对时间

 private:
  bool m_set;                 // ★ 非 atomic，靠 EventMutex 保护
  int64_t signal_count;       // ★ 非 atomic
  EventMutex mutex;           // = OSMutex（裸 pthread）
  os_cond_t cond_var;         // pthread_cond_t / CONDITION_VARIABLE

  static pthread_condattr_t cond_attr;
  static bool cond_attr_has_monotonic_clock;
  static bool global_initialized;
#ifdef UNIV_DEBUG
  static std::atomic_size_t n_objects_alive;
#endif
 protected:
  os_event(const os_event &);              // 禁拷贝
  os_event &operator=(const os_event &);
};
```

```cpp
os_event::os_event() UNIV_NOTHROW {
  ut_a(global_initialized);
  init();
  m_set = false;
  /* We return this value in os_event_reset(), ... The value of zero is
  reserved in os_event_wait_low() for the case when the caller does not want
  to pass any signal_count value. To distinguish between the two cases we
  initialize signal_count to 1 here. */
  signal_count = 1;                        // ★★ 从 1 起跳，0 是"未指定"哨兵
}

os_event_t os_event_create() {
  os_event_t ret = (ut::new_withkey<os_event>(UT_NEW_THIS_FILE_PSI_KEY));
#if defined(LINUX_SUSE)
  /* On SuSE Linux we get spurious EBUSY from pthread_mutex_destroy() ... */
  os_event_reset(ret);                     // ★ SuSE workaround：创建后立刻加解锁一次
#endif
  return ret;
}
```

> ⚠️ **签名是 `os_event_t os_event_create()`（无参）**——老版本的 `os_event_create(name)` 已不存在。
> ⚠️ **`os_event` 没有 cond 类 PFS 埋点**。全文件唯一 PSI 是 `UT_NEW_THIS_FILE_PSI_KEY`，只做 **PFS 内存记账**。对比之下 `PolicyMutex` 有完整 PFS（`init_mutex` / `start_mutex_wait` / `end_mutex_wait` / `unlock_mutex` / `destroy_mutex` + `sync0sync.cc` 里约 **97 个 `mysql_pfs_key_t`**）。**所以"mutex 有 PFS、event 没有"是 8.0 的事实**。

**`wait_time_low()`**（`os0event.cc:450`）——超时返回值只有两种：

```cpp
ulint os_event::wait_time_low(std::chrono::microseconds timeout,
                              int64_t reset_sig_count) UNIV_NOTHROW {
  bool timed_out = false;
#ifndef _WIN32
  struct timespec abstime;
  if (timeout != std::chrono::microseconds::max()) {
    abstime = os_event::get_wait_timelimit(timeout);
  } else {
    abstime.tv_nsec = 999999999;
    abstime.tv_sec = std::numeric_limits<time_t>::max();   // ★ "伪无限"
  }
#endif
  mutex.enter();
  if (!reset_sig_count) { reset_sig_count = signal_count; }

  do {
    if (m_set || signal_count != reset_sig_count) {
      break;                       // ★★ 与 wait_low 相同的双重判据
    }
    timed_out = timed_wait(&abstime);
  } while (!timed_out);
  mutex.exit();

  return (timed_out ? OS_SYNC_TIME_EXCEEDED : 0);   // ★ 0=成功，1=超时
}
```

```cpp
bool os_event::timed_wait(const timespec *abstime) {
  int ret = pthread_cond_timedwait(&cond_var, mutex, abstime);
  switch (ret) {
    case 0:
    case ETIMEDOUT:
    case EINTR:      /* We play it safe by checking for EINTR even though
                     according to the POSIX documentation it can't return EINTR. */
      break;
    default:
      ib::error(ER_IB_MSG_742) << "pthread_cond_timedwait() returned: " << ret
                               << ": abstime={" << abstime->tv_sec << ","
                               << abstime->tv_nsec << "}";
      ut_error;
  }
  return (ret == ETIMEDOUT);       // ★ 只有 ETIMEDOUT 算超时；EINTR 视为"被唤醒"，外层循环重判
}
```

**等待数组的完整生命周期**（`sync0arr.cc`）：

```cpp
sync_cell_t *sync_array_reserve_cell(sync_array_t *arr, void *object,
                                     ulint type, ut::Location location) {
  sync_cell_t *cell;
  sync_array_enter(arr);

  if (arr->first_free_slot != ULINT_UNDEFINED) {
    cell = sync_array_get_nth_cell(arr, arr->first_free_slot);
    arr->first_free_slot = cell->line;      // ★ 空闲链表：next 指针复用 cell->line 字段
  } else if (arr->next_free_slot < arr->n_cells) {
    cell = sync_array_get_nth_cell(arr, arr->next_free_slot);
    ++arr->next_free_slot;
  } else {
    sync_array_exit(arr);
    return (nullptr);                       // ★ 本数组满了 → 换一个数组实例
  }
  ++arr->res_count;
  ++arr->n_reserved;
  cell->request_type = type;
  ...                                       // 按 type 填 latch.mutex / latch.bpmutex / latch.lock
  cell->file = location.filename;
  cell->line = location.line;
  sync_array_exit(arr);
  cell->thread_id = std::this_thread::get_id();
  cell->reservation_time = std::chrono::steady_clock::now();

  os_event_t event = sync_cell_get_event(cell);
  cell->signal_count = os_event_reset(event);   // ★★ 先 reset 再等待
  return (cell);
}
```

```cpp
void sync_array_wait_event(sync_array_t *arr, sync_cell_t *&cell) {
  sync_array_enter(arr);
  cell->waiting = true;
#ifdef UNIV_DEBUG
  rw_lock_debug_mutex_enter();
  ut_a(arr->last_scan % 2 == 0);
  ++arr->last_scan;
  sync_array_detect_deadlock(arr, cell, 0);    // ★ debug 下入队前先做 DFS 环检测
  ++arr->last_scan;
  ut_a(arr->last_scan % 2 == 0);
  rw_lock_debug_mutex_exit();
#endif
  sync_array_exit(arr);

  os_event_wait_low(sync_cell_get_event(cell), cell->signal_count);

  sync_array_free_cell(arr, cell);             // ★★ 返回后自动释放
  cell = nullptr;
}
```

```cpp
void sync_array_free_cell(sync_array_t *arr, sync_cell_t *&cell) {
  sync_array_enter(arr);
  cell->waiting = false;
  cell->signal_count = 0;
  cell->latch.mutex = nullptr;
  cell->line = arr->first_free_slot;
  arr->first_free_slot = cell - arr->cells;    // ★ 头插法回收
  arr->n_reserved--;
  if (arr->next_free_slot > arr->n_cells / 2 && arr->n_reserved == 0) {
    arr->next_free_slot = 0;                  // ★ 全空且水位过半 → 整体收缩
    arr->first_free_slot = ULINT_UNDEFINED;
  }
  sync_array_exit(arr);
  cell = nullptr;
}
```

**多数组时的随机分派**（`sync0arr.ic:44`）：

```cpp
static inline sync_array_t *sync_array_get() {
  if (sync_array_size <= 1) {
    return (sync_wait_array[0]);            // ★ 默认只有 1 个
  }
  return (sync_wait_array[default_indexer_t<>::get_rnd_index() % sync_array_size]);
}
```

```cpp
// sync_array_init() —— 总槽位 = 线程数，按数组数均分
  ulint n_slots = 1 + (n_threads - 1) / sync_array_size;
  for (ulint i = 0; i < sync_array_size; ++i) {
    sync_wait_array[i] = ut::new_withkey<sync_array_t>(UT_NEW_THIS_FILE_PSI_KEY, n_slots);
  }
```

**★ 依赖方向图**：

```
        ┌─ PolicyMutex (ib_mutex_t) ─┬─ OSTrackMutex   → pthread_mutex
        │                            ├─ TTASFutexMutex → syscall(SYS_futex)
        │                            └─ TTASEventMutex ─┐
        └─ rw_lock_t ───────────────────────────────────┤
                                                        ↓
                              sync0arr (wait array) ← os_event ← pthread_cond
                                        ↓
                              srv_error_monitor_thread（每秒兜底唤醒）

        locksys::Latches ── global_latch (Sharded_rw_lock, 64 片 rw_lock_t)
                        └─ page_shards[512] / table_shards[512]（各一把 ib_mutex_t）
```

**下游（本篇调谁）**：只有 L0——`futex`（Linux 且 `MUTEX_FUTEX`）、`pthread_cond_t`（os_event）、`pthread_mutex_t`（`OSTrackMutex`）、`std::atomic`（所有锁字）；以及 PFS（`PolicyMutex` 的 `pfs_begin_lock` / `rw_lock` 的 `pfs_rw_lock_*`）与 sync0arr 的诊断接口。

**上游（谁调本篇）**——本篇是纯基础设施，所有 InnoDB 子系统的并发都在它之上：

| 用它的子系统 | 用的是哪种 | 说明 | 展开在哪 |
|---|---|---|---|
| **B-tree 索引锁** | `dict_index_t::lock`（`rw_lock_t`，SMO 用 **SX**） | 搜索持 S、修改叶持 X、结构修改持 SX——本篇 SX 语义的第一用户 | [`../../index/btr.md`](../../../index/btr.md) |
| **Buffer Pool 页 latch** | `buf_block_t::lock`（`rw_lock_t`）+ 各 `ib_bpmutex_t`（**BlockMutexPolicy**） | 页 latch 协议与 io_fix 状态机校验 | [`../../innodb/buffer_pool.md`](../../../innodb/buffer_pool.md) |
| **锁系统分片** | `locksys::Latches`（分片 mutex + `Unique_sharded_rw_lock`） | 512+512 shard，闸门本身又是分片 rwlock | [`../transactional/innodb_trx_lock.md`](../transactional/innodb_trx_lock.md) |
| **字典锁** | `dict_operation_lock`（`rw_lock_t`，**SX**） | PFS 名 `wait/synch/sxlock/innodb/dict_operation_lock` | [`../../server/dd/dd.md`](../../../server/dd/dd.md) |
| **事务系统** | `trx_sys_mutex`、`trx_sys_shard_mutex`、`serialisation_mutex` | read view 生成与事务串行化 | [`../../innodb/trx.md`](../../../innodb/trx.md) |
| **redo / log** | `log_sys` 相关 latch + os_event（写盘等待） | 8.0 的 log 系统大量用 `os_event_wait_for` 与 `std::condition_variable` | [`../../innodb/redo_log.md`](../../../innodb/redo_log.md) |
| **自适应哈希** | `btr_search_latch`（`rw_lock_t`，可分区） | 热点等值查询的加速结构 | [`../../index/ahi.md`](../../../index/ahi.md) |
| **插入缓冲 / 文件空间** | `ibuf_mutex`、`ibuf_bitmap_mutex` + fil 三级锁（`Fil_shard::m_mutex`×68、`fil_space_t::latch` rw_lock_t，**无独立全局 mutex**） | 二级索引延迟写；表空间管理三级锁（保护对象/锁序/引用计数保活见上文「表空间锁」节） | 本节上文 |

这条"L0 → 本篇 → 各子系统"的链才是完整的：**本篇不解决任何业务问题，它只是让上面每一层都能回答"谁持锁、等了多久、顺序对不对"**。

---

## ★ 本机制里的工程实现技法

### 一、lock_word 单字三态：信息压进一个原子字

| 维度 | 教科书原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 状态表示 | 读计数 + 写标志 + 等待队列等多个字段 | **一个 `atomic<int32>` 的正负区间编码**（S=减1、SX=减 HALF_DECR、X=减 DECR） | 快路径一次 CAS、转换天然原子；代价是编码要背、debug 靠 `rw_lock_validate` 兜底 |
| 写者优先 | 额外的 writer-waiting 队列 | **X 先减占坑**（负值区间自然阻止新 S） | 无需单独队列；代价是 X 请求会立即吓退新读者 |
| 递归 | 通常单独支持 | 同一个字：X 递归每次再减 1（首次减 `X_LOCK_DECR`） | 递归次数受 `X_LOCK_DECR` 上限约束 |

### 二、策略模板：横切关注的零成本注入

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| `template <template <typename> class Policy = NoPolicy>` | `TTASEventMutex`/`TTASFutexMutex` | 统计、debug 挂钩经模板参数静默注入，release（NoPolicy 空实现）零开销 | 组合而非 CRTP——策略是**成员**不是基类，读代码易漏 |
| `GenericPolicy`（逐实例）vs `BlockMutexPolicy`（聚合） | 一般 mutex vs buf block mutex | "too many of them to count individually"——聚合策略是数量级的妥协 | 两种策略的计数语义不同（单实例 vs 求和），对比数据时要分清 |
| `MutexDebug` 以条件基类混入 | UNIV_DEBUG 下策略类 | debug 钩子与统计策略解耦 | `#ifdef` 基类导致 debug/release 的对象布局不同 |

### 三、三套锁序校验的演进

```
LatchDebug（人工 level 枚举，金字塔）
  → ut::Stateful_latching_rules（声明式：状态图 × 边标注 latch 子集）
  → LOCK ORDER 工具（数据驱动：PFS 名 + 依赖清单文件，图无环判定）
```

方向是**把"锁序正确性"从人工纪律变成可声明的规格**。`Stateful_latching_rules` 的头注释把问题建模为"状态图，边标注所需 latch 子集"，还提供 `assert_latches_let_distinguish`（证明"读状态时持有的锁足以冻结状态"）——这是把并发正确性论证写进代码的又一例（与 Theorem 1 注释同源）。

### 四、reader_thread 的 XOR 压缩

| 维度 | 原型 | 落地 | 代价 |
|---|---|---|---|
| 记录读者 | 每读者一个槽位或位图 | **全体读者线程 id 的 XOR**（一个字） | 正常不可还原；仅"恰一个读者"时可恢复（`recover_if_single()`）——够死锁检测用 |

---

## 可观测性

### `SHOW ENGINE INNODB STATUS` 的 SEMAPHORES 段

⚠️ **勘误（重要）**：`RW-shared spins / RW-excl spins / RW-sx spins` 三行**在 8.0.39 里恒为 0**——计数器**已被删除**。全仓 grep `spin_wait_count` / `spin_round_count` / `os_wait_count` **0 命中**。

**硬编码打印 0 的现场**（`sync0sync.cc:191`）：

```cpp
/**
Prints wait info of the sync system.
Note: The instrumental counters are deprecated
      and prints all 0 for compatibility.
@param file - where to print */
static void sync_print_wait_info(FILE *file) {
  fprintf(file,
          "RW-shared spins 0, rounds 0, OS waits 0\n"
          "RW-excl spins 0, rounds 0, OS waits 0\n"
          "RW-sx spins 0, rounds 0, OS waits 0\n");

  fprintf(file,
          "Spin rounds per wait: 0.00 RW-shared,"
          " 0.00 RW-excl, 0.00 RW-sx\n");
}
```

> ★★ **实战结论**：8.0 里 SEMAPHORES 段**唯一有价值的信息**是 ① "OS WAIT ARRAY INFO: reservation count / signal count"（来自 `sync_array_print_info()`）与 ② 长等待告警（`sync_array_print_long_waits()`）。**spin 统计已死**。
> ★ 唯一残留的 per-lock 等待计数是 `rw_lock_t::count_os_wait`（`sync0rw.cc` 里 `lock->count_os_wait += count_os_wait;`），`MutexMonitor::reset()` 会清零。

**wait array 的打印**（`sync0arr.cc:976`）：

```cpp
void sync_array_print(FILE *file) {
  sync_array_print_info(file, sync_wait_array, sync_array_size);
}
```
> ⚠️ **不存在 `sync_array_output()`**。存在的是 `sync_array_print()`（:976）、`sync_array_print_info()`（:933）、`sync_array_print_info_low()`（:910）、`sync_array_print_long_waits()`（:858）。

**⚠️ `innodb_sync_array_size` 是 READONLY**（`ha_innodb.cc:22083`）——运行期不可改，默认 **1**：

```cpp
static MYSQL_SYSVAR_ULONG(sync_array_size, srv_sync_array_size,
                          PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,
                          "Size of the mutex/lock wait array.", nullptr,
                          nullptr, 1, 1, 1024, 0);
```

**★ wait array 职责收缩的官方原文**（`sync0arr_impl.h:57`）——这是 5.0.30 的历史性改动：

```cpp
  As of 5.0.30 the above mentioned design is changed. Since now OS can handle
  millions of wait events efficiently, we no longer have this concept of each
  cell of wait array having one event.  Instead, now the event that a thread
  wants to wait on is embedded in the wait object (mutex or rw_lock). We still
  keep the global wait array for the sake of diagnostics and also to avoid
  infinite wait The error_monitor thread scans the global wait array to signal
  any waiting threads who have missed the signal.
```

> ★★ **所以 wait array 现在只剩两个用途**：①**诊断**（`SHOW ENGINE INNODB STATUS` 的等待列表）；②**兜底防丢唤醒**（`sync_arr_wake_threads_if_sema_free()` 每秒扫描）。
> ⚠️ **cell 不自带 event**——`sync_cell_get_event()` 从被等的 latch 里取（`sync0arr.cc:160`）：mutex → `m_event`；rw_lock → `event` 或 `wait_ex_event`（X_WAIT 用后者）。

```cpp
struct sync_cell_t {
  sync_object_t latch;       /* union: rw_lock_t* / WaitMutex* / BlockWaitMutex* */
  ulint request_type = 0;
  const char *file = nullptr;
  ulint line = 0;                      // ★ 空闲链表复用它存 next
  std::thread::id thread_id{};
  bool waiting = false;
  int64_t signal_count = 0;            // ★ os_event_reset() 的返回值
  std::chrono::steady_clock::time_point reservation_time{};
  uint64_t last_scan{0};               // ★ DFS 环检测标记（奇数=在栈上）
};
```

> ⚠️ **不存在 `sync_array_create()` / `sync_array_cell_get()`**——真名 `sync_array_t` 构造函数（:114）+ `sync_array_get_nth_cell()`（:147）。
> ★ `sync_array_t::mutex` 是 **`SysMutex`（裸 OSMutex）**——注释明说 "to prevent infinite recursion in implementation, we fall back to an OS mutex"。

**★ locksys 分片：`SHARDS_COUNT = 512`**（`lock0latches.h:166`）——⚠️ 不是 32/64：

```cpp
  static constexpr size_t SHARDS_COUNT = 512;   // ★ page shard 与 table shard 各 512，共 1024 把 mutex

class Page_shards {
  Padded_mutex mutexes[SHARDS_COUNT];           // ★ ut::Cacheline_padded<ib_mutex_t>
 public:
  static size_t get_shard(const page_id_t &page_id);
};
class Table_shards {
  Padded_mutex mutexes[SHARDS_COUNT];
 public:
  static size_t get_shard(const table_id_t table_id);
};
```

```cpp
size_t Latches::Page_shards::get_shard(const page_id_t &page_id) {
  /* We need a property that if two pages are mapped to the same bucket of the
  hash table, and thus their lock queues are merged, then these two lock queues
  are protected by the same shard. This is why to compute the shard we use the
  cell_id as the input and not the original lock_rec_hash_value's result. */
  return hash_calc_cell_id(lock_rec_hash_value(page_id), lock_sys->rec_hash) % SHARDS_COUNT;
  //     ★★ 先算 hash cell 再 %512 —— 保证"同一 cell ⇒ 同一 shard"
}

size_t Latches::Table_shards::get_shard(const table_id_t table_id) {
  return table_id % SHARDS_COUNT;      // ★ table 直接按 id 取模，无额外 hash
}
```

> ⚠️ **`include/lock0shard.h` 不存在**（全仓 0 命中）。`Shard_latches_guard` / `Global_exclusive_latch_guard` 等 RAII guard 都在 **`include/lock0guards.h`**。
> ★ **global_latch 本身也是分片的**（`Sharded_rw_lock`，内部 64 个 `rw_lock_t`）——因为 ARM 上 `rw_lock_t` 的 s-latch 计数是 read-update-CAS 循环，多线程抢同一 cache line。`Unique_sharded_rw_lock` 用 `thread_local size_t m_shard_id` 记住本线程 s-latch 了哪一片。
> ★ 分片 mutex 的 latch level：page = `SYNC_LOCK_SYS_SHARDED`，table = `SYNC_LOCK_SYS_TABLE`。
> ⚠️ **不存在 `Latches::print()`**——只有 4 个 UNIV_DEBUG 断言辅助：`owns_exclusive_global_latch()` / `owns_shared_global_latch()` / `owns_page_shard()` / `owns_table_shard()`。

仍在：`srv_printf_innodb_monitor` 打 `"----------\nSEMAPHORES\n----------\n"` 后调 `sync_print()` = `rw_lock_list_print_info`（DEBUG：遍历 `rw_lock_list` 打非空闲锁）+ `sync_array_print`（OS WAIT ARRAY INFO：reservation/signal count、每个 cell 的等待者详情）+ `sync_print_wait_info`。

**★ spin 统计已废弃**——后者打印固定 0：

```cpp
fprintf(file, "RW-shared spins 0, rounds 0, OS waits 0\n"
              "RW-excl spins 0, rounds 0, OS waits 0\n"
              "RW-sx spins 0, rounds 0, OS waits 0\n");
```

（注释自述 "The instrumental counters are deprecated and prints all 0 for compatibility"。）自旋/OS wait 统计让位给 PFS——**别再从 SEMAPHORES 段读 spin 数**。另外 `innodb_status_output_locks` 是事务锁系统的开关，与 SEMAPHORES 段无关。

### core dump 排查锁持有者：gdb + pt-pmp 排除法

latch 没有死锁检测，一旦 hang 住（semaphore wait 超时 crash 或实例无响应），core dump 是唯一现场。四类锁的 gdb 排查法（淘宝内核月报 2020-03 的完整方法论）：

| 锁 | gdb 看什么 | 关键点 |
|---|---|---|
| server `pthread_mutex_t` | `p LOCK_status` → `__data.__owner` | `__owner` = 持有者线程的 **LWP 号**，对照 `info threads` 定位持有者 |
| server `pthread_rwlock_t` | `__nr_readers` / `__nr_writers_queued` / `__writer` | **写锁优先**：`__nr_writers_queued != 0` 时新读请求也阻塞（防写饿死）；读锁持有者可能多个，需排除法 |
| InnoDB `rw_lock_t` | `lock_word` + `writer_thread` + `last_*_file_name` | ① `lock_word` 对照编码表（见「rw_lock_t」节）判断当前是 S/SX/X/等待态；② `writer_thread` 是持有者线程 id（十六进制，可直接在堆栈里搜）；③ ★ **`last_s_file_name:last_s_line` / `last_x_file_name:last_x_line`**——最近一次加 S/X 锁的**源码位置**（写锁未真正持有时指向上一个持有者），是"谁最后碰过这把锁"的直接线索 |
| InnoDB `PolicyMutex` | `m_lock_word` | 三态：0 空闲 / 1 持有 / 2 有等待者。**⚠️ `m_waiters=0 不代表无等待线程**——线程都在自旋（未进等待队列）时 waiters 仍为 0；自旋时长由 `innodb_sync_spin_loops` / `innodb_spin_wait_delay` 控制 |

**pt-pmp 排除法**（核心思想：**持有者堆栈只出现 1 次，等锁者大量重复**）：

1. 导出全部线程堆栈 → `pt-pmp` 聚合
2. 排除频次高于 `__nr_readers` 的高频堆栈（大概率是等锁者）
3. 排除阻塞在获取该锁的线程
4. 排除带 `poll()`/`epoll_wait` 的堆栈
5. 排除带 `pthread_cond_wait`/`__lll_lock_wait` 的堆栈（持有锁期间 sleep 概率极低）
6. 剩下的"只出现 1 次"的堆栈即持有者

**参数联动**：若所有拿锁线程都在自旋（`waiters=0` 且大量 CPU 空转），减小 `innodb_sync_spin_loops` 或 `innodb_spin_wait_delay` 可让竞争显形——这是"排查不出来时的最后一招"。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 谁在等哪个 latch、等多久 | SQL | `SHOW ENGINE INNODB STATUS` 的 SEMAPHORES 段（`--Thread ... has waited at ... for N seconds`） |
| mutex 自旋/等待统计 | PFS | `wait/synch/mutex/innodb/*` 的 events_waits 汇总 |
| SX 锁（树锁/字典锁）竞争 | PFS | `wait/synch/sxlock/innodb/*` |
| 长信号量等待告警 | 日志 | error_monitor：`A long semaphore wait`（>4 分钟）→ 超过 fatal 阈值崩溃 |
| 锁序违反 | debug 构建 | UNIV_DEBUG 的 `latch order violation`（LatchDebug）/ `WITH_LOCK_ORDER` 构建的 LOCK ORDER 工具 |
| 读者是谁（诊断） | debug 构建 | `reader_thread` XOR 在恰一个读者时还原；`rw_lock_debug_print` 打 "Locked: thread ... S-LOCK/X-LOCK/SX-LOCK" |

---

## Misc

### 面向二次开发

**扩展点**：新增一把 InnoDB mutex = 选 `ib_mutex_t`（futex 构建）+ `mutex_create(key, ...)` 注册 PFS key。新增一种策略 = 实现一个 `Policy` 模板类（`init/add/locked/release`）并替换 typedef。**给 `rw_lock_t` 加新状态 = 改 `X_LOCK_DECR` 编码体系**——牵一发动全身（兼容矩阵、`rw_lock_validate`、PFS 三态建模），基本不可行，SX 是 5.6 加的最后一次。

**坑与已知缺陷**：

1. **waiters 协议顺序敏感**："先设 flag → 复查 → 睡"与"先放锁 → reset flag → signal"两处顺序反了都是丢唤醒 bug——Theorem 1 注释证明的正是这类协议，改代码前先读它。
2. **`wait(filename, line, 4)` 的 4 是几十年 heuristic**（注释自述），不是推导值。
3. **SEMAPHORES 段的 spin 数字恒为 0**（已废弃），别基于它调 `innodb_spin_wait_*`。
4. **`srv_fatal_semaphore_wait_threshold` 崩溃是症状不是病根**：出现即说明有 latch 泄漏或死锁，去查LatchDebug/LOCK ORDER。
5. **futex 构建与 event 构建的观测面不同**：futex 版没有 sync array cell（等待在内核 futex 队列里），SEMAPHORES 段的 OS WAIT ARRAY INFO 在该构建下信息变少。
6. **`reader_thread` XOR 是 debug 手段不是安全机制**：多读者时 XOR 值无意义。
7. `latch_add_to_history` 已不存在（旧文档/月报常见），现行是 `LatchDebug` 的 `sync_check_*` API。

**社区边界澄清**：`MUTEX_FUTEX` 只在检测到 Linux futex 时启用；Windows 构建走 `MUTEX_SYS`（pthread/critical section 包装）。Percona 曾引入额外的 mutex 统计（如 `innodb_mutex_..`），社区版没有。

### 易混淆概念

**① `os_event` 是 "manual-reset"，不是 auto-reset**

```cpp
  /** Creates an event semaphore, i.e., a semaphore which may just have two
  states: signaled and nonsignaled. The created event is manual reset: it
  must be reset explicitly by calling os_event_reset(). */
```

> `set()` 后 `m_set` 一直为 true，**直到显式 `reset()`**。所以"事件"更接近一个"电平"而不是"脉冲"。

**② `os_event_set()` 唤醒全部，不是唤醒一个**

```cpp
  void broadcast() UNIV_NOTHROW {
    m_set = true;
    ++signal_count;
    pthread_cond_broadcast(&cond_var);   // ★ 广播
  }
```
> ⚠️ 私有 `signal()`（`pthread_cond_signal`）**全文件无调用点——死代码**。对比：futex 版 `signal()` 用 `FUTEX_WAKE_PRIVATE, 1`（**只唤醒 1 个**）。两者语义不同。

**③ `os_event_wait_time_low()` 的超时语义：Linux 是"总时长"，Windows 是"每轮重新计时"**

```cpp
#ifndef _WIN32
    abstime = os_event::get_wait_timelimit(timeout);   // ★ 算一次绝对时刻
#else
    time_in_ms = ...timeout...;                        // ★ 每次都是相对毫秒
#endif
  do {
    if (m_set || signal_count != reset_sig_count) break;
    timed_out = timed_wait(&abstime);                  // ★ Linux：abstime 每轮是同一个值
    //                                                    Windows：每轮重新计时
  } while (!timed_out);
```

```cpp
  struct timespec os_event::get_wait_timelimit(std::chrono::microseconds timeout) {
    /* We could get rid of this function if we switched to std::condition_variable
    from the pthread_cond_. The std::condition_variable::wait_for relies on the
    steady_clock internally and accepts timeout (not time increased by the
    timeout). */
    ...
      if (clock_gettime(CLOCK_MONOTONIC, &tp) == -1) { ... }
      //  ★★ 单调时钟，避免系统时钟回拨导致永久阻塞
```

> ★ 时钟源在 `os_event_global_init()` 里设置：`pthread_condattr_setclock(&cond_attr, CLOCK_MONOTONIC)`（**macOS 无此支持**，会 warn 后退化为 `gettimeofday`）。

**④ `srv_slot_t`（锁等待）与 `sync_cell_t`（latch 等待）是两套完全不同的机制**

| | `srv_slot_t` | `sync_cell_t` |
|---|---|---|
| 位置 | `lock/lock0wait.cc` | `sync/sync0arr.cc` |
| 等的是 | **事务锁**（行锁/表锁） | **latch**（mutex / rw_lock） |
| 唤醒者 | `lock_wait_release_thread_if_suspended()` | `sync_arr_wake_threads_if_sema_free()`（每秒兜底） |
| 死锁检测 | `lock_wait_find_and_handle_deadlocks()` | debug 下 `sync_array_detect_deadlock()` |
| 超时 | `innodb_lock_wait_timeout` | **无超时**（latch 等待不设超时） |

> ★★ **这是最关键的区分**：`Innodb_row_lock_waits` 涨 = 事务锁等待；SEMAPHORES 段有长等待 = latch 争用。二者排查手段完全不同。

**⑤ `EventMutex` ≠ `TTASEventMutex`**

```cpp
typedef OSMutex EventMutex;                                        // 裸 pthread mutex
UT_MUTEX_TYPE(TTASEventMutex, GenericPolicy, SyncArrayMutex)        // 这才是 TTASEventMutex
```

**⑥ `rw_lock_t` 的 S 锁不可递归，X/SX 可递归**

```cpp
  ut_ad(!rw_lock_own(lock, RW_LOCK_S));   /* see NOTE above */   // sync0rw.ic:271
  ut_ad(!rw_lock_own(lock, RW_LOCK_X));
```

**⑦ `lock_word` 初值 `X_LOCK_DECR` 同时是"最大并发 S 锁数"**

> `X_LOCK_DECR = 0x20000000 = 536870912`。`X_LOCK_DECR - lock_word` = S 锁数量，所以理论上限 2^29 个读者。

**⑧ `PolicyMutex::exit()` 的 hook 在解锁之前**

```cpp
    policy().release(m_impl);   // ① hook
    m_impl.exit();              // ② 才真正解锁
```
> 反直觉但必须如此——`release` 里要 `sync_check_unlock()`，那时锁还没放，读 `m_context` 才安全。

**⑨ `ib_mutex_t` 默认是 `SysMutex`（裸 pthread），不是 futex**

> 需 `cmake -DMUTEXTYPE=futex` 且 `HAVE_IB_LINUX_FUTEX` 探测通过。

**⑩ `innodb_sync_array_size` 是 READONLY，默认 1**

> 不是"越大越好、可在线调"。启动时定死。

- **`os_event` ≠ server 的 `mysql_cond_t`**：前者是 manual-reset 事件（sticky + signal_count），后者是标准条件变量。InnoDB 内部等待统一走 os_event。
- **wait array 不是等待队列**：5.0.30 起 event 已嵌入被等对象，wait array 只剩诊断、防丢信号兜底、debug 死锁检测三个职责。
- **`SX`（rw_lock_t 的三态之一）vs `SX latch`（旧文档对 index->lock 的泛称）**：前者是精确的锁模式（兼容 S 不兼容 SX/X），后者在老资料里常泛指"结构修改锁"。
- **`latch_t` 是 debug 基类**，不是"buf 页 latch"；buf 页的 latch 是 `rw_lock_t`（`buf_block_t::lock`）。
- **`ib_mutex_t` 与 `SysMutex`/`SyncArrayMutex`/`FutexMutex`**：前者是按构建选定的别名，后三是候选实现。

---

## 参考

**论文 / 经典算法**
- D. Dreier. *Futexes Are Tricky*（Ulrich Drepper, *Futexes Are Tricky*, 2011）。（`TTASFutexMutex` 的三态协议与"两次检查"防丢唤醒正是该文 §"Optimizations" 的标准落地）
- TTAS 自旋锁与指数退避：A. Agarwal & M. Cherian, *Adaptive Backoff Synchronization Techniques*, ISCA 1989.（`ut_delay` 随机退避的出处思想）
- 读者-写者锁写者优先变体：见 [`mysys_primitives.md`](mysys_primitives.md) 参考章（同一问题的引擎侧解法）

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Startup Options*（`innodb_spin_wait_delay` / `innodb_sync_spin_loops` / `innodb_sync_array_size`）
- *MySQL 8.0 Reference Manual → Performance Schema InnoDB Instruments*（`wait/synch/mutex/innodb` 等前缀）

**相关文档**
- server 侧 mysys 三层封装与 prlock 见 [`mysys_primitives.md`](mysys_primitives.md)
- B-tree 索引锁（`index->lock` 的 SX 使用协议）见 [`../../index/btr.md`](../../../index/btr.md)
- 页 latch 与 Buffer Pool 并发协议（`buf_block_t::lock`、`Stateful_latching_rules` 的用户 io_fix）见 [`../../innodb/buffer_pool.md`](../../../innodb/buffer_pool.md)
- 全局 latch 实例清单（A3）见 [`../README.md`](../README.md)
