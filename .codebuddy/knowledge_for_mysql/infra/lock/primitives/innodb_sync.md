# InnoDB 同步原语族（os_event / wait array / PolicyMutex / rw_lock_t）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 InnoDB 自研同步原语的四层谱系：`os_event`（manual-reset 事件 + signal_count 防丢信号）、sync0arr 等待数组（event 已嵌入被等对象，数组只剩诊断职责）、`ib0mutex.h` 的 PolicyMutex 模板族（TTAS 自旋 + event/futex 两种慢路径）、策略层（统计/锁序校验的零成本注入）、以及 `rw_lock_t` 的 **lock_word 单字三态编码**（S/SX/X + 递归）。
>
> **边界**：本篇讲 InnoDB 原语的**类型与机制**；全局 latch 实例清单见 [`../README.md`](../README.md) A3；server 侧原语（mysys 三层封装）见 [`mysys_primitives.md`](mysys_primitives.md)；两个著名用户——B-tree 索引锁与页 latch 协议——见 [`../../innodb/btr.md`](../../../innodb/btr.md) 与 [`../../innodb/buffer_pool.md`](../../../innodb/buffer_pool.md)。

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

| 体系 | 位置 | 校验什么 | 状态 |
|---|---|---|---|
| **`LatchDebug`**（`latch_t` 基类） | `sync0debug.cc` + `sync0types.h` | 按 `latch_level_t` 金字塔（头部有整幅 ASCII 锁序图：Dict mutex → 索引树 → 叶页 → trx sys → buf pool → log）检查加锁顺序，违反打 `"latch order violation"` 后 crash | UNIV_DEBUG |
| **`ut::Stateful_latching_rules`** | `ut0stateful_latching_rules.h` | **状态机 × 持锁集合**的声明式校验：把问题建模为"状态图，边标注所需 latch 子集"。唯一用户是 `buf_page_t::io_fix`（`Buf_io_fix_latching_rules`），提供"读状态时持有的锁足以冻结状态"的断言 | UNIV_DEBUG |
| **LOCK ORDER 工具** | `sql/debug_lock_order.cc`（`WITH_LOCK_ORDER` 构建选项） | 用 PFS 名建全局有向图（`lock_order_dependencies.txt`），运行时校验加锁序列无环 → 保证无死锁；InnoDB sxlock 被建模为三态（注释 "Shared exclusive locks are recursive... SX + X counts as X"） | 独立构建选项 |

**分层归属（三套体系谁管谁）**：

- **LatchDebug** = InnoDB 专属（UNIV_DEBUG，只覆盖 InnoDB 自研原语，锁序金字塔是 InnoDB 内部知识）
- **LOCK ORDER 工具** = **server 层跨层**（`sql/debug_lock_order.cc`，覆盖 server + InnoDB **所有 PFS 锁**——输入是 PFS instrument 名，不关心锁是 mysys 还是 InnoDB 实现）
- **PFS** = 两者共享的**锁命名层**：LOCK ORDER 以 PFS 名为图节点；LatchDebug 校验的锁在 PFS 也有对应 instrument（`wait/synch/mutex|rwlock|sxlock/innodb/*`），供运行时观测竞争（见「可观测性」章）。锁序校验的演进方向就是"从 InnoDB 内部金字塔走向以 PFS 名为通用坐标"。

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

**下游（本篇调谁）**：只有 L0——`futex`（Linux 且 `MUTEX_FUTEX`）、`pthread_cond_t`（os_event）、`pthread_mutex_t`（`OSTrackMutex`）、`std::atomic`（所有锁字）；以及 PFS（`PolicyMutex` 的 `pfs_begin_lock` / `rw_lock` 的 `pfs_rw_lock_*`）与 sync0arr 的诊断接口。

**上游（谁调本篇）**——本篇是纯基础设施，所有 InnoDB 子系统的并发都在它之上：

| 用它的子系统 | 用的是哪种 | 说明 | 展开在哪 |
|---|---|---|---|
| **B-tree 索引锁** | `dict_index_t::lock`（`rw_lock_t`，SMO 用 **SX**） | 搜索持 S、修改叶持 X、结构修改持 SX——本篇 SX 语义的第一用户 | [`../../innodb/btr.md`](../../../innodb/btr.md) |
| **Buffer Pool 页 latch** | `buf_block_t::lock`（`rw_lock_t`）+ 各 `ib_bpmutex_t`（**BlockMutexPolicy**） | 页 latch 协议与 io_fix 状态机校验 | [`../../innodb/buffer_pool.md`](../../../innodb/buffer_pool.md) |
| **锁系统分片** | `locksys::Latches`（分片 mutex + `Unique_sharded_rw_lock`） | 512+512 shard，闸门本身又是分片 rwlock | [`../transactional/innodb_trx_lock.md`](../transactional/innodb_trx_lock.md) |
| **字典锁** | `dict_operation_lock`（`rw_lock_t`，**SX**） | PFS 名 `wait/synch/sxlock/innodb/dict_operation_lock` | [`../../server/dd/dd.md`](../../../server/dd/dd.md) |
| **事务系统** | `trx_sys_mutex`、`trx_sys_shard_mutex`、`serialisation_mutex` | read view 生成与事务串行化 | [`../../innodb/trx.md`](../../../innodb/trx.md) |
| **redo / log** | `log_sys` 相关 latch + os_event（写盘等待） | 8.0 的 log 系统大量用 `os_event_wait_for` 与 `std::condition_variable` | [`../../innodb/redo_log.md`](../../../innodb/redo_log.md) |
| **自适应哈希** | `btr_search_latch`（`rw_lock_t`，可分区） | 热点等值查询的加速结构 | [`../../innodb/ahi.md`](../../../innodb/ahi.md) |
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

仍在：`srv_printf_innodb_monitor` 打 `"----------\nSEMAPHORES\n----------\n"` 后调 `sync_print()` = `rw_lock_list_print_info`（DEBUG：遍历 `rw_lock_list` 打非空闲锁）+ `sync_array_print`（OS WAIT ARRAY INFO：reservation/signal count、每个 cell 的等待者详情）+ `sync_print_wait_info`。

**★ spin 统计已废弃**——后者打印固定 0：

```cpp
fprintf(file, "RW-shared spins 0, rounds 0, OS waits 0\n"
              "RW-excl spins 0, rounds 0, OS waits 0\n"
              "RW-sx spins 0, rounds 0, OS waits 0\n");
```

（注释自述 "The instrumental counters are deprecated and prints all 0 for compatibility"。）自旋/OS wait 统计让位给 PFS——**别再从 SEMAPHORES 段读 spin 数**。另外 `innodb_status_output_locks` 是事务锁系统的开关，与 SEMAPHORES 段无关。

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
- B-tree 索引锁（`index->lock` 的 SX 使用协议）见 [`../../innodb/btr.md`](../../../innodb/btr.md)
- 页 latch 与 Buffer Pool 并发协议（`buf_block_t::lock`、`Stateful_latching_rules` 的用户 io_fix）见 [`../../innodb/buffer_pool.md`](../../../innodb/buffer_pool.md)
- 全局 latch 实例清单（A3）见 [`../README.md`](../README.md)
