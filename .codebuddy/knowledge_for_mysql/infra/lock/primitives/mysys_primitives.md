# server 侧同步原语族（mysys 封装 + PFS 埋点）深度解析

> 基于 MySQL 8.0.39 源码，涵盖从 OS 原语（pthread/CriticalSection）到 `mysql_mutex_t`/`mysql_rwlock_t`/`mysql_prlock_t`/`mysql_cond_t` 的三层封装、PFS 埋点宏链、SAFE_MUTEX 调试版、`m_psi` 无条件内嵌的 ABI 权衡，以及 **prlock 为什么为 MDL 手搓而不用 pthread_rwlock**。
>
> **边界**：本篇讲 server 侧原语的**类型与机制**；全局实例清单见 [`../README.md`](../README.md) A2；InnoDB 自研原语（`rw_lock_t`/`os_event`/PolicyMutex 模板族）见 [`innodb_sync.md`](innodb_sync.md)；RCU 见 [`rcu.md`](rcu.md)。

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

server 侧的每把 mutex/rwlock/cond 都是**三层包装**后的产物。头注释把分层讲得最清楚：

```c
/**
  There are three "layers":
  1) native_mutex_*()
       Functions that map directly down to OS primitives.
       Windows    - CriticalSection
       Other OSes - pthread
  2) my_mutex_*()
       Functions that implement SAFE_MUTEX (default for debug),
       Otherwise native_mutex_*() is used.
  3) mysql_mutex_*()
       Functions that include Performance Schema instrumentation.
*/
```

```
L0  OS 原语      pthread_mutex_t / pthread_rwlock_t / pthread_cond_t / CRITICAL_SECTION
                  │
L1  可移植封装    native_mutex_t / my_mutex_t（SAFE_MUTEX 调试版或直通）
                  │   （类型在 components/services/bits/*_bits.h，ABI 稳定层）
L2  PFS 埋点版    mysql_mutex_t / mysql_rwlock_t / mysql_prlock_t / mysql_cond_t
                  │   （= L1 结构 + PSI_mutex* m_psi 钩子）
L3  实例          LOCK_open、THD::LOCK_thd_data、MDL_context::m_LOCK_waiting_for ...
```

### 用途

给 server 与各存储引擎提供**统一的、带可观测性的、跨平台的**原语接口。业务代码只写 `mysql_mutex_lock(&m)`，至于走 PFS 计时、SAFE_MUTEX 调试还是直通 pthread，全部由编译期维度决定，业务代码零改动。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | `mysql_mutex_*` 逐步成型；`my_atomic_*` 是原子操作封装 |
| 5.6+ | PFS 引入，L2 埋点层出现 |
| 8.0 | 类型定义迁到 `include/mysql/components/services/bits/`（组件服务 ABI 层）；`my_atomic.h` 缩水为只剩 `LF_BACKOFF` 自旋宏，**真正的原子操作直接用 `std::atomic`**；`DISABLE_ALL_PSI_THREAD` 这类总闸拆成 per-interface 的 `DISABLE_PSI_<TYPE>` 细粒度开关 |

---

## 理论基础

### 设计思想与权衡

#### 一、为什么是三层：正交的三个维度

**源码自带的官方注释**（`include/mysql/psi/mysql_mutex.h:38`）——作者明说这是"不可避免的 #ifdef 复杂度"：

```cpp
/*
  Note: there are several orthogonal dimensions here.

  Dimension 1: Instrumentation
  HAVE_PSI_MUTEX_INTERFACE is defined when the instrumentation is compiled in.
  This may happen both in debug or production builds.

  Dimension 2: Debug
  SAFE_MUTEX is defined when debug is compiled in.
  This may happen both with and without instrumentation.

  Dimension 3: Platform
  Mutexes are implemented with one of:
  - the pthread library
  - fast mutexes
  - window apis
  This is implemented by various macro definitions in my_thread.h

  This causes complexity with '#ifdef'-ery that can't be avoided.
*/
```

**★ 完整链路是 5 层，不是 3 层**（`mysql_mutex_lock()` → `pthread_mutex_lock()`）：

| 层 | 位置 | 作用 |
|---|---|---|
| ① 宏门面 | `mysql_mutex.h:170` | 注入 `__FILE__/__LINE__` |
| ② PFS inline | `mysql_mutex.h:242` | `m_psi` 埋点（双重判空） |
| ③ mysys 中间层 | `thr_mutex.h:180` | SAFE_MUTEX 二选一 |
| ④ safe_mutex（仅 debug） | `mysys/thr_mutex.cc:68` | 记录持有者/文件/行号 |
| ⑤ OS 原语 | `thr_mutex.h:90` | `pthread_mutex_lock` / `EnterCriticalSection` |

```cpp
// ① —— 两层宏，_with_src 才是真身
#define mysql_mutex_lock(M) mysql_mutex_lock_with_src(M, __FILE__, __LINE__)
#define mysql_mutex_lock_with_src(M, F, L) inline_mysql_mutex_lock(M, F, L)
```

```cpp
// ③ —— my_mutex_lock：SAFE_MUTEX 下转 safe_mutex_lock，否则直落 native
static inline int my_mutex_lock(my_mutex_t *mp
#ifdef SAFE_MUTEX
                                , const char *file, uint line
#endif
) {
#ifdef SAFE_MUTEX
  assert(mp != nullptr);
  assert(mp->m_u.m_safe_ptr != nullptr);
  return safe_mutex_lock(mp->m_u.m_safe_ptr, false, file, line);
#else
  return native_mutex_lock(&mp->m_u.m_native);
#endif
}
```

```cpp
// ⑤ —— 终点
static inline int native_mutex_lock(native_mutex_t *mutex) {
#ifdef _WIN32
  EnterCriticalSection(mutex);
  return 0;
#else
  return pthread_mutex_lock(mutex);
#endif
}
```

> ⚠️ **勘误**：**不存在 `inline_mysql_mutex_assert_owner()` / `assert_not_owner()` 内联函数**——它们只是**宏** `mysql_mutex_assert_owner(M)` / `mysql_mutex_assert_not_owner(M)`，release 下展开成 `{}`。
> ⚠️ **不存在 `include/mysql/psi/mysql_prlock.h`**——prlock 的 PFS 封装**全在 `mysql_rwlock.h` 里**，被 `#ifndef DISABLE_MYSQL_PRLOCK_H` 包裹。
> ⚠️ **不存在 `include/my_pthread.h`**——`native_mutex_*` 在 **`include/thr_mutex.h`**（不是 `mysys/`）；`native_rw_*` 在 `include/thr_rwlock.h`；`native_cond_*` 在 `include/thr_cond.h`。`mysys/` 下只有三个 `.cc`。
> ⚠️ **不存在 `MY_MUTEX_NO_DEADLOCK_DETECTION`**——8.0.39 只有 3 个 attr 宏：`MY_MUTEX_INIT_SLOW / FAST / ERRCHK`。死锁检测靠 `WITH_LOCK_ORDER`（lock_order 子系统），不再是 attr 标志。

`mysql_mutex.h` 开头的注释是理解整个设计的钥匙——这里**不是一个开关，而是三个正交维度**：

> ```c
> /*
>   Note: there are several orthogonal dimensions here.
>   Dimension 1: Instrumentation  HAVE_PSI_MUTEX_INTERFACE ...
>   Dimension 2: Debug            SAFE_MUTEX is defined when debug is compiled in.
>                                 This may happen both with and without instrumentation.
>   Dimension 3: Platform ...
> */
> ```

| 维度 | 取值 | 由谁决定 |
|---|---|---|
| 埋点 | `HAVE_PSI_MUTEX_INTERFACE` 有/无 | `WITH_PERFSCHEMA_STORAGE_ENGINE` + `DISABLE_PSI_MUTEX` 细粒度开关 |
| 调试 | `SAFE_MUTEX` 有/无 | **CMake 在 Debug 构建注入 `-DSAFE_MUTEX`**（`STRING_PREPEND(CMAKE_${LANG}_FLAGS_DEBUG "-DSAFE_MUTEX ")`） |
| 平台 | Linux/macOS/Windows | 头文件内的 `#ifdef` |

关键推论：**SAFE_MUTEX 与 PFS 不互斥，可以同时开**——官方 debug 构建就是这样：`inline_mysql_mutex_lock` 先做 PSI 计时，再调 `my_mutex_lock`，后者因 `SAFE_MUTEX` 转发到 `safe_mutex_lock`。一层包装两头受益（`src_file/line` 同时喂给两者）。

被否决的方案：如果调试和埋点做成一个维度的三种模式（如编译期三选一），就无法"边计时边查锁 bug"——注释特意强调 "orthogonal" 就是为了堵死这种理解。

#### 二、★ `m_psi` 指针无条件内嵌：ABI 换零分支成本

**结构体定义**（`include/mysql/components/services/bits/mysql_mutex_bits.h:40`）：

```cpp
struct mysql_mutex_t {
  /** The real mutex. */
  my_mutex_t m_mutex;
  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c mysql_mutex_t interface.
  */
  struct PSI_mutex *m_psi{nullptr};      // ★ 无条件存在，release 下恒为 nullptr
};
```

```cpp
/** union：native / safe 二选一 */
struct my_mutex_t {
  union u {
    native_mutex_t m_native;      // release：内嵌 pthread_mutex_t
    safe_mutex_t *m_safe_ptr;     // SAFE_MUTEX：改为指向 malloc 出来的 safe_mutex_t
  } m_u;
};
```

> ⚠️ **勘误**：**只有 `m_psi`，没有 `m_psi_internal`**（全仓 sync 结构体中无此成员）。

**★ `HAVE_PSI_MUTEX_INTERFACE` 不是"两套函数"，而是同一个 inline 函数体内的 `#ifdef`**（`mysql_mutex.h:242`）：

```cpp
static inline int inline_mysql_mutex_lock(mysql_mutex_t *that,
                                          const char *src_file, uint src_line) {
  int result;

#ifdef HAVE_PSI_MUTEX_INTERFACE
  if (that->m_psi != nullptr) {
    if (that->m_psi->m_enabled) {              // ★ 双重判空：已 instrument + 该 instrument 被启用
      /* Instrumentation start */
      PSI_mutex_locker *locker;
      PSI_mutex_locker_state state;            // ★ state 在调用方栈上，PFS 复用它做 locker，避免 malloc
      locker = PSI_MUTEX_CALL(start_mutex_wait)(
          &state, that->m_psi, PSI_MUTEX_LOCK, src_file, src_line);

      /* Instrumented code */
      result = my_mutex_lock(&that->m_mutex
#ifdef SAFE_MUTEX
                             , src_file, src_line
#endif
      );

      /* Instrumentation end */
      if (locker != nullptr) {                 // ★ PFS 判定"无需记录"时返回 NULL，直接跳过
        PSI_MUTEX_CALL(end_mutex_wait)(locker, result);
      }

      return result;
    }
  }
#endif

  /* Non instrumented code */
  result = my_mutex_lock(&that->m_mutex
#ifdef SAFE_MUTEX
                         , src_file, src_line
#endif
  );

  return result;
}
```

```cpp
static inline int inline_mysql_mutex_unlock(mysql_mutex_t *that,
                                            const char *src_file, uint src_line) {
  int result;
#ifdef HAVE_PSI_MUTEX_INTERFACE
  if (that->m_psi != nullptr) {
    PSI_MUTEX_CALL(unlock_mutex)(that->m_psi);
    //  ★ 注意：unlock 不检查 m_enabled（与 lock/trylock 不同），
    //   因为 PFS 侧 unlock_mutex 只清 m_owner，成本极低
  }
#endif
  result = my_mutex_unlock(&that->m_mutex
#ifdef SAFE_MUTEX
                           , src_file, src_line
#endif
  );
  return result;
}
```

**★ PFS 官方对开销的说明**（`storage/perfschema/pfs.cc:774`）：

```cpp
  When the performance schema is disabled at runtime, the @c m_psi pointer
  is @c NULL, so the extra overhead is one if statement.

  When the performance schema is enabled at runtime, the @c m_psi pointer
  points to the actual instrumentation added to the mutex,
  and two extra calls are present on the code path:
  - a call to @c pfs_start_mutex_wait_v1() before the operation
  - a call to @c pfs_end_mutex_wait_v1() after the operation

  Both these calls are static, and point directly to the
  performance_schema implementation.

  @section PFS_PSI_PLUGIN Instrumentation for a plugin

  Inside a plugin, @c PSI_MUTEX_CALL expands to a call using a function pointer.
  This is to decouple the plugin binary from the implementation,
  to avoid having hard coded references to symbols like
  pfs_start_mutex_wait_v1() from the plugin library (either shared or static).
```

```cpp
// include/pfs_mutex_provider.h:46 —— server 内部直接符号，插件走函数指针
#define PSI_MUTEX_CALL(M) pfs_##M##_v1
// 否则
#ifndef PSI_MUTEX_CALL
#define PSI_MUTEX_CALL(M) psi_mutex_service->M
#endif
```

**★ `pfs_start_mutex_wait_v1()` 的三条返回 NULL 快路径**（`pfs.cc:3735`）：

```cpp
  if (flag_thread_instrumentation) {
    PFS_thread *pfs_thread = my_thread_get_THR_PFS();
    if (unlikely(pfs_thread == nullptr)) return nullptr;   // ①未 instrument 的线程
    if (!pfs_thread->m_enabled) return nullptr;            // ②该线程未启用
    ...
    if (flag_events_waits_current) {
      if (unlikely(pfs_thread->m_events_waits_current >=
                   &pfs_thread->m_events_waits_stack[WAIT_STACK_SIZE])) {
        locker_lost++;                                     // ③等待栈溢出
        return nullptr;
      }
      ...
      wait->m_source_file = src_file;    // ★ __FILE__/__LINE__ 落到这里
      wait->m_source_line = src_line;
      wait->m_object_instance_addr = pfs_mutex->m_identity;
    }
  } else {
    if (pfs_mutex->m_timed) { ... }
    else {
      /* Complete shortcut. */
      /* Aggregate to EVENTS_WAITS_SUMMARY_BY_INSTANCE (counted) */
      pfs_mutex->m_mutex_stat.m_wait_stat.aggregate_counted();  // ★ 最廉价：只加计数
      return nullptr;
    }
  }
  return reinterpret_cast<PSI_mutex_locker *>(state);   // ★ locker 就是调用方栈上的 state
```

> ★ `pfs_end_mutex_wait_v1()` 只在 `rc == 0`（加锁成功）时记 `m_owner`——**trylock 失败不记 owner**。`pfs_unlock_mutex_v1()` 整个只做一件事：`m_owner = nullptr`。

**`HAVE_PSI_*_INTERFACE` 的开关源头**（`include/my_psi_config.h:38`）：

```cpp
#ifdef WITH_PERFSCHEMA_STORAGE_ENGINE
#define HAVE_PSI_INTERFACE
#endif
#ifdef HAVE_PSI_INTERFACE
#ifndef DISABLE_PSI_MUTEX
#define HAVE_PSI_MUTEX_INTERFACE
#endif
#ifndef DISABLE_PSI_RWLOCK
#define HAVE_PSI_RWLOCK_INTERFACE
#endif
#ifndef DISABLE_PSI_COND
#define HAVE_PSI_COND_INTERFACE
#endif
```

> ★ **`PSI_mutex_key == 0` 恒表示"不 instrument"**（`psi_mutex_bits.h:47`：`Using a zero key always disable the instrumentation`）。`pfs_init_mutex_v1()` 里 `find_mutex_class(key)` 找不到就返回 NULL → `m_psi == NULL` → 后续零开销。

**assert 宏的 release 空实现**（`mysql_mutex.h:108`）：

```cpp
#ifdef SAFE_MUTEX
#define mysql_mutex_assert_owner(M) \
  safe_mutex_assert_owner((M)->m_mutex.m_u.m_safe_ptr);   // ★ 穿透到 union 里的 m_safe_ptr
#else
#define mysql_mutex_assert_owner(M) \
  {}                                                      // ★ release：展开成空语句块
#endif
```

```cpp
// mysql_mutex_bits.h —— L2 类型
struct mysql_mutex_t {
  my_mutex_t m_mutex;
  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c mysql_mutex_t interface.
  */
  struct PSI_mutex *m_psi{nullptr};
};
```

注释明说：`m_psi` **不随 `#ifdef` 消失**，为了 `mysql_mutex_t` 接口的二进制兼容（插件/组件与 server 可能分开编译）。

代价与收益：

| | 内容 |
|---|---|
| 代价 | 每个原语多 8 字节（server 内有数千个 mutex 实例） |
| 收益 | 同一份二进制接口覆盖"埋点/不埋点"两种构建；热路径开销被压到 `m_psi == nullptr` + `m_psi->m_enabled` **两次分支**（实例级开关允许动态关闭单个 instrument） |
| 兜底 | `mysql_mutex_init(key=0, ...)` 即 `PSI_NOT_INSTRUMENTED` → `m_psi = nullptr` → 永远走非埋点分支，只付一次分支开销 |

#### 三、prlock 的取舍：手搓读写锁，故意让写者饿死

**底层实现全文**（`mysys/thr_rwlock.cc:36`）——★ 读者**完全不阻塞**：

```cpp
int rw_pr_init(rw_pr_lock_t *rwlock) {
  native_mutex_init(&rwlock->lock, nullptr);
  native_cond_init(&rwlock->no_active_readers);
  rwlock->active_readers = 0;
  rwlock->writers_waiting_readers = 0;
  rwlock->active_writer = false;
#ifdef SAFE_MUTEX
  rwlock->writer_thread = 0;
#endif
  return 0;
}

int rw_pr_rdlock(rw_pr_lock_t *rwlock) {
  native_mutex_lock(&rwlock->lock);
  /*
    The fact that we were able to acquire 'lock' mutex means
    that there are no active writers and we can acquire rd-lock.
    Increment active readers counter to prevent requests for
    wr-lock from succeeding and unlock mutex.
  */
  rwlock->active_readers++;          // ★★ 只要没写者，直接 +1 走人
  native_mutex_unlock(&rwlock->lock);
  return 0;
}

int rw_pr_wrlock(rw_pr_lock_t *rwlock) {
  native_mutex_lock(&rwlock->lock);

  if (rwlock->active_readers != 0) {
    /* There are active readers. We have to wait until they are gone. */
    rwlock->writers_waiting_readers++;

    while (rwlock->active_readers != 0)       // ★★ 无超时、无让步：读者不断就来不了
      native_cond_wait(&rwlock->no_active_readers, &rwlock->lock);

    rwlock->writers_waiting_readers--;
  }

  /*
    We own 'lock' mutex so there is no active writers.
    Also there are no active readers.
    This means that we can grant wr-lock.
    Not releasing 'lock' mutex until unlock will block
    both requests for rd and wr-locks.
    ...
    Thanks to the fact wr-lock/unlock in the absence of
    contention from readers is essentially mutex lock/unlock
    with a few simple checks make this rwlock implementation
    wr-lock optimized.
  */
  rwlock->active_writer = true;
#ifdef SAFE_MUTEX
  rwlock->writer_thread = my_thread_self();
#endif
  return 0;                          // ★ 注意：不释放 lock，持到 unlock
}
```

```cpp
int rw_pr_unlock(rw_pr_lock_t *rwlock) {
  if (rwlock->active_writer) {
    /* We are unlocking wr-lock. */
    rwlock->active_writer = false;
    if (rwlock->writers_waiting_readers) {
      /*
        Avoid expensive cond signal in case when there is no contention
        or if it is write-only.

        Note that from a performance point of view it would be better to
        signal on the condition variable after unlocking mutex (as it
        reduces the number of context switches).

        Unfortunately this would mean that such rwlock can't be safely
        used by the MDL subsystem, which relies on the fact that it is okay
        to destroy rwlock once it is in unlocked state.
      */
      native_cond_signal(&rwlock->no_active_readers);   // ★★ 必须在持 lock 时 signal —— MDL 的硬约束
    }
    native_mutex_unlock(&rwlock->lock);
  } else {
    /* We are unlocking rd-lock. */
    native_mutex_lock(&rwlock->lock);
    rwlock->active_readers--;
    if (rwlock->active_readers == 0 && rwlock->writers_waiting_readers) {
      native_cond_signal(&rwlock->no_active_readers);
    }
    native_mutex_unlock(&rwlock->lock);
  }
  return 0;
}
```

**★ 设计意图的官方注释**（`include/.../thr_rwlock_bits.h:64`）——两条"不寻常"性质：

```cpp
/**
  Portable implementation of special type of read-write locks.

  These locks have two properties which are unusual for rwlocks:
  1) They "prefer readers" in the sense that they do not allow
     situations in which rwlock is rd-locked and there is a
     pending rd-lock which is blocked (e.g. due to pending
     request for wr-lock).
     This is a stronger guarantee than one which is provided for
     PTHREAD_RWLOCK_PREFER_READER_NP rwlocks in Linux.
     MDL subsystem deadlock detector relies on this property for
     its correctness.
  2) They are optimized for uncontended wr-lock/unlock case.
     This is scenario in which they are most often used
     within MDL subsystem. Optimizing for it gives significant
     performance improvements in some of tests involving many
     connections.

  Another important requirement imposed on this type of rwlock
  by the MDL subsystem is that it should be OK to destroy rwlock
  object which is in unlocked state even though some threads might
  have not yet fully left unlock operation for it (of course there
  is an external guarantee that no thread will try to lock rwlock
  which is destroyed).
  Putting it another way the unlock operation should not access
  rwlock data after changing its state to unlocked.

  TODO/FIXME: We should consider alleviating this requirement as
  it blocks us from doing certain performance optimizations.
*/
struct rw_pr_lock_t {
  native_mutex_t lock;              /* 保护结构；wr-lock 期间也持有 */
  native_cond_t no_active_readers;  /* 写者等读者走光 */
  unsigned int active_readers;
  unsigned int writers_waiting_readers;
  bool active_writer;
  my_thread_t writer_thread;        /* 仅 debug */
};
```

> ★★ **关键**：性质 1 比 Linux 的 `PTHREAD_RWLOCK_PREFER_READER_NP` **更强**——**MDL 死锁检测器依赖它才正确**。所以这不是"随便手搓"，而是为 MDL 定制的正确性前提。
> ★ **"故意让写者饿死"**：`rw_pr_wrlock()` 的 `while` 没有任何让步机制，读者源源不断则写者永远进不来。MDL 场景下可接受（DDL 稀少）。

**PFS 侧：prlock 复用 rwlock 的 PSI 类型**（`mysql_rwlock.h:270`）：

```cpp
#ifndef DISABLE_MYSQL_PRLOCK_H
static inline int inline_mysql_prlock_init(PSI_rwlock_key key,
                                           mysql_prlock_t *that, ...) {
#ifdef HAVE_PSI_RWLOCK_INTERFACE
  that->m_psi = PSI_RWLOCK_CALL(init_rwlock)(key, &that->m_prlock);
#else
  that->m_psi = nullptr;
#endif
  return rw_pr_init(&that->m_prlock);
}
#endif
```

> ★ 所以在 PFS 里 **prlock 就是一个 rwlock instrument**，看不出区别。
> ★ **reader/writer 走两套独立 service 槽位**：`start_rwlock_rdwait` / `end_rwlock_rdwait` vs `start_rwlock_wrwait` / `end_rwlock_wrwait`（`pfs.cc:9347` 的 8 槽服务表）。`unlock` 不分读写，统一 `PSI_RWLOCK_UNLOCK`。
> ★ op 枚举区分 5 种：`PSI_RWLOCK_READLOCK / WRITELOCK / TRYREADLOCK / TRYWRITELOCK / UNLOCK`。`try` 变体仍走 `rdwait`/`wrwait` 通道，仅 op 不同。

`mysql_prlock_t` 的底座 `rw_pr_lock_t` **不是** pthread_rwlock 的包装，而是 mysys 用 mutex + cond 手搓的"优先读者"锁。它给出的两条硬保证（头注释原文翻译）：

> 1) **"prefer readers"**：不允许"锁已被 rd-lock，且存在一个被阻塞的 pending rd-lock（例如被 wr-lock 请求挡住）"的局面。**这是比 Linux `PTHREAD_RWLOCK_PREFER_READER_NP` 更强的保证**。MDL 子系统的死锁检测器的正确性依赖它。
> 2) **为无竞争的 wr-lock/unlock 优化**（退化为 mutex 的 lock/unlock）。

为什么 MDL 需要第一条：死锁检测遍历 wait-for 图时多线程并发 `rdlock`（读边），只有单线程改边时 `wrlock`。若普通 rwlock 在写者排队时挡住新读者，**检测器自身会被卡住**——这不是性能问题，是正确性问题。代价是**写者可能被读者流饿死**，MDL 场景写极稀少，可接受。

还有一条隐蔽的契约（`rw_pr_unlock` 注释）：**unlock 改完状态后不得再碰锁数据**——这样 MDL 允许"锁一进 unlocked 态就 destroy"。为此 signal 必须在释放内部 mutex **之前**做。

### 理论溯源

- **读者-写者问题的优先级变体**：Courtois et al. (1971) 给出读者优先/写者优先/公平三种原型。prlock 是"**强读者优先**"变体——比教科书"读者优先"多一条"读者间绝不互相阻塞（即使有写者排队）"的保证，落点是 `rw_pr_rdlock` 无条件 `active_readers++` 成功。
- **PFS 的低开销观测模型**：静态注册（key 预分配）+ 动态实例钩子（`m_psi`）+ 双重闸门（全局 `m_enabled`/实例 enabled）——思想与 eBPF 的静态探针类似：埋点代码常驻，开销由开关控制。

### 算法与数据结构

**四个 instrumented 结构体**（⚠️ 全在 `include/mysql/components/services/bits/`，不是 `include/mysql/psi/`）：

```cpp
struct mysql_mutex_t   { my_mutex_t m_mutex;      struct PSI_mutex  *m_psi{nullptr}; };
struct mysql_rwlock_t  { native_rw_lock_t m_rwlock; struct PSI_rwlock *m_psi; };
struct mysql_prlock_t  { rw_pr_lock_t m_prlock;    struct PSI_rwlock *m_psi; };  // ★ 复用 rwlock 的 PSI 类型
struct mysql_cond_t    { native_cond_t m_cond;     struct PSI_cond   *m_psi; };
```

**`native_*_t` 的平台定义**：

```cpp
// mutex
#ifdef _WIN32
typedef CRITICAL_SECTION native_mutex_t;
typedef int native_mutexattr_t;
#else
typedef pthread_mutex_t native_mutex_t;
typedef pthread_mutexattr_t native_mutexattr_t;
#endif

// rwlock —— ★ Windows 下不是裸 SRWLOCK
#ifdef _WIN32
struct native_rw_lock_t {
  SRWLOCK srwlock;             /* native reader writer lock */
  BOOL have_exclusive_srwlock; /* used for unlock */   // ★★ 必须自己记账
};
#else
typedef pthread_rwlock_t native_rw_lock_t;
#endif

// cond
#ifdef _WIN32
typedef CONDITION_VARIABLE native_cond_t;
#else
typedef pthread_cond_t native_cond_t;
#endif
```

> ★★ **Windows SRWLOCK 为什么要额外一个 bool**：`SRWLOCK` 的释放函数分 `ReleaseSRWLockShared` / `ReleaseSRWLockExclusive`，**没有一个统一的 unlock**，所以必须自己记住"当前是共享还是独占"。POSIX 的 `pthread_rwlock_unlock` 一个通吃。

```cpp
static inline int native_rw_unlock(native_rw_lock_t *rwp) {
#ifdef _WIN32
  if (rwp->have_exclusive_srwlock) {
    rwp->have_exclusive_srwlock = false;
    ReleaseSRWLockExclusive(&rwp->srwlock);
  } else
    ReleaseSRWLockShared(&rwp->srwlock);
  return 0;
#else
  return pthread_rwlock_unlock(rwp);      // ★ pthread 一个 unlock 通吃
#endif
}
```

> ★ Windows 的 `native_mutex_trylock()` 还手工禁止了递归（`CRITICAL_SECTION` 天然递归）：
> ```cpp
>   if (TryEnterCriticalSection(mutex)) {
>     /* Don't allow recursive lock */
>     if (mutex->RecursionCount > 1) { LeaveCriticalSection(mutex); return EBUSY; }
>     return 0;
>   }
> ```

**★ `my_cond_timedwait` 是绝对时间（absolute），不是相对**（这是最常见的坑）：

```cpp
static inline int native_cond_timedwait(native_cond_t *cond,
                                        native_mutex_t *mutex,
                                        const struct timespec *abstime) {
#ifdef _WIN32
  DWORD timeout = get_milliseconds(abstime);   // ★ Windows 分支才做"绝对→相对"转换
  if (!SleepConditionVariableCS(cond, mutex, timeout)) return ETIMEDOUT;
  return 0;
#else
  return pthread_cond_timedwait(cond, mutex, abstime);  // ★★ POSIX：abstime 是 CLOCK_REALTIME 绝对时刻
#endif
}
```

调用方必须自己把"现在 + 偏移"写进 `abstime`（`mysys/my_systime.cc:58`）：

```cpp
void set_timespec_nsec(struct timespec *abstime, Timeout_type nsec) {
  if (nsec == TIMEOUT_INF) { *abstime = TIMESPEC_POSINF; return; }
  unsigned long long int now = my_getsystime() + (nsec / 100);   // ★ now = 当前 + 偏移 → 绝对时刻
  ...
}
void set_timespec(struct timespec *abstime, Timeout_type sec) {
  if (sec == TIMEOUT_INF) { *abstime = TIMESPEC_POSINF; return; }
  set_timespec_nsec(abstime, sec * 1000000000ULL);
}
```

标准范例（`my_thr_init.cc:213`）：

```cpp
  struct timespec abstime;
  set_timespec(&abstime, my_thread_end_wait_time);   // ★ 先换算
  mysql_mutex_lock(&THR_LOCK_threads);
  while (THR_thread_count > 0) {
    int error = mysql_cond_timedwait(&THR_COND_threads, &THR_LOCK_threads, &abstime);
    if (is_timeout(error)) { ...; break; }
  }
```

```cpp
static inline int is_timeout(int e) {    // include/my_thread.h:57
#if ETIMEDOUT == ETIME
  return e == ETIMEDOUT;
#else
  return e == ETIMEDOUT || e == ETIME;
#endif
}
```

> ⚠️ Windows 分支因为 `SleepConditionVariableCS` 只接受相对毫秒，在 `get_milliseconds()`（`thr_cond.h:61`）里换算，**负差被 clamp 成 0**。

**`MY_MUTEX_INIT_FAST` 的平台换挡**（`include/thr_mutex.h:60`）：

```cpp
#define MY_MUTEX_INIT_SLOW NULL                 // "慢" = 默认 attr

#ifdef PTHREAD_ADAPTIVE_MUTEX_INITIALIZER_NP
extern native_mutexattr_t my_fast_mutexattr;
#define MY_MUTEX_INIT_FAST &my_fast_mutexattr   // ★ Linux/glibc：指向真 attr
#else
#define MY_MUTEX_INIT_FAST NULL                 // ★ 其它平台（含 macOS）：退化为 NULL，FAST == SLOW
#endif

#ifdef PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP
extern native_mutexattr_t my_errorcheck_mutexattr;
#define MY_MUTEX_INIT_ERRCHK &my_errorcheck_mutexattr
#else
#define MY_MUTEX_INIT_ERRCHK NULL
#endif
```

```cpp
// mysys/my_thr_init.cc:167 —— adaptive 的官方解释
#ifdef PTHREAD_ADAPTIVE_MUTEX_INITIALIZER_NP
  /*
    Set mutex type to "fast" a.k.a "adaptive"

    In this case the thread may steal the mutex from some other thread
    that is waiting for the same mutex.  This will save us some
    context switches but may cause a thread to 'starve forever' while
    waiting for the mutex (not likely if the code within the mutex is
    short).
  */
  pthread_mutexattr_init(&my_fast_mutexattr);
  pthread_mutexattr_settype(&my_fast_mutexattr, PTHREAD_MUTEX_ADAPTIVE_NP);
#endif
```

> ★ **adaptive mutex = 先自旋若干次再 futex 睡眠**，用"可能饿死"换"少上下文切换"。

**`my_atomic.h`：文件仍存在，但原子族已消亡**（`include/my_atomic.h`，全文 64 行）：

```cpp
#if defined(_MSC_VER)
#define YIELD_LOOPS 200
static inline int my_yield_processor() {
  int i;
  for (i = 0; i < YIELD_LOOPS; i++) { YieldProcessor(); }
  return 1;
}
#define LF_BACKOFF my_yield_processor()
#else
#define LF_BACKOFF (1)          // ★ 非 MSVC：空操作
#endif
```

> ⚠️ **勘误**：文件**存在**（不是"已删除"），但 `my_atomic_add32/64`、`my_atomic_loadptr`、`my_atomic_fasptr`、`my_atomic_casptr` 等**全部被删除**（8.0 起改用 `std::atomic`）。**现在只剩 `LF_BACKOFF` 一个宏**。

**`st_my_thread_var`：只有 2 个字段，且仅 debug 存在**（`mysys/my_thr_init.cc:95`）：

```cpp
#ifndef NDEBUG
struct st_my_thread_var {
  my_thread_id id;            // ★ 字段 1：mysys 线程号（报错打印 T@%u）
  struct CODE_STATE *dbug;    // ★ 字段 2：DBUG 的 per-thread 状态
};
#endif
```

```cpp
extern "C" bool my_thread_init() {
#ifndef NDEBUG
  if (mysys_thread_var()) return false;          // ★ 幂等
  if (!(tmp = (struct st_my_thread_var *)calloc(1, sizeof(*tmp)))) return true;
  mysql_mutex_lock(&THR_LOCK_threads);
  tmp->id = ++thread_id;
  ++THR_thread_count;
  mysql_mutex_unlock(&THR_LOCK_threads);
  set_mysys_thread_var(tmp);
#endif
  return false;
}
```

> ⚠️ **勘误**：`st_my_thread_var` **不含 mutex/cond/mysys_var 等一堆字段**，release（`NDEBUG`）下 `my_thread_init()` **几乎什么都不做**（整个函数体被 `#ifndef NDEBUG` 掏空）。`THR_mysys` 已是 C++11 `thread_local`，不再是 pthread_key。
> ★ `my_thread_end()` 里 `PSI_THREAD_CALL(delete_current_thread)()` **必须在释放 `tmp` 之前**——注释明说 LF_HASH 依赖它。

| 结构 | 组成 | 复杂度 |
|---|---|---|
| `native_mutex_t` | `pthread_mutex_t`（Win: `CRITICAL_SECTION`） | lock/unlock O(1)（uncontended CAS） |
| `my_mutex_t` | union：native 或 `safe_mutex_t*` | SAFE_MUTEX 下多两把内部锁 |
| `mysql_mutex_t` | `my_mutex_t + PSI_mutex*` | 非埋点路径 +2 次分支 |
| `rw_pr_lock_t` | `native_mutex_t + native_cond_t + 4 个计数字段` | rdlock O(1)；wrlock 等读者清零 |

SAFE_MUTEX 的 `safe_mutex_t`：

```c
struct safe_mutex_t {
  native_mutex_t global, mutex;
  const char *file;
  uint line, count;
  my_thread_t thread;
};
```

两把锁 + 持有者元数据：`global` 保护元数据（count/thread/file），`mutex` 是真锁。能抓四类错误且全部 `abort()`：未初始化就 lock、**同线程重复 lock（自死锁——fast mutex 下会悄悄死锁，这里直接报）**、非持有者 unlock、锁着 destroy。

### 他库对比与演进动机

| 数据库 | server 层原语封装 | 与 MySQL 的差异 |
|---|---|---|
| **PostgreSQL** | 自己的 `s_lock`（spinlock）/ `LWLock` / `lwlock` 都带内置统计与 trace（`TRACE_POSTGRESQL_LWLOCK_*`） | PG 把可观测做进**锁实现内部**；MySQL 把它做成**外挂包装层**（L2），底层可换成任何 OS 原语 |
| **Oracle** | latch/free list 自研，统计进 v$latch | 与 MySQL 的 PFS 模型类似但更早 |
| **MySQL InnoDB** | 完全自研（见 [`innodb_sync.md`](innodb_sync.md)） | 同一个仓库里两套哲学：server 包 OS 原语，InnoDB 自研——因为 InnoDB 需要 spin+wait 两级等待与 S/SX/X 三态 |

**为什么演进成现在这样**：8.0 把类型定义迁入 `components/services/bits/`，是为**组件架构**服务——组件/插件不再 `#include` server 内部头，只依赖稳定的 ABI bits 头；`m_psi` 无条件内嵌正是这次 ABI 化的一部分。

---

## 核心实现

### 主链路：一次 `mysql_mutex_lock` 的完整旅程

```
mysql_mutex_lock(M)                          [宏，注入 __FILE__/__LINE__]
  → inline_mysql_mutex_lock(that, src_file, src_line)     [static inline]
      → if (m_psi && m_psi->m_enabled):
          PSI_MUTEX_CALL(start_mutex_wait)(&state, m_psi, PSI_MUTEX_LOCK, F, L)
          result = my_mutex_lock(&that->m_mutex)          ← 真正加锁点
          PSI_MUTEX_CALL(end_mutex_wait)(locker, result)
      → else: my_mutex_lock(...) 直接走（+2 次分支的代价）
```

`my_mutex_lock` 再按维度分流：

```
my_mutex_lock(m)
  ├─ SAFE_MUTEX: safe_mutex_lock(m->m_u.m_safe_ptr, ...)   记录持有者/自死锁检测
  └─ 否则:      native_mutex_lock(m->m_u.m_native)        直通 pthread
```

关键代码：

```cpp
static inline int inline_mysql_mutex_lock(mysql_mutex_t *that,
                                          const char *src_file, uint src_line) {
  int result;
#ifdef HAVE_PSI_MUTEX_INTERFACE
  if (that->m_psi != nullptr) {
    if (that->m_psi->m_enabled) {
      PSI_mutex_locker *locker;
      PSI_mutex_locker_state state;
      locker = PSI_MUTEX_CALL(start_mutex_wait)(
          &state, that->m_psi, PSI_MUTEX_LOCK, src_file, src_line);
      result = my_mutex_lock(&that->m_mutex);
      if (locker != nullptr) {
        PSI_MUTEX_CALL(end_mutex_wait)(locker, result);
      }
      return result;
    }
  }
#endif
  /* Non instrumented code */
  result = my_mutex_lock(&that->m_mutex);
  return result;
}
```

逐段：

- `start_mutex_wait` 返回 **可空的 locker**：PFS 按采样/开关决定是否真的记录，只有非空才调 `end_mutex_wait` 结算耗时写入 `events_waits_current/history`。
- unlock 不需要 timer，只调 `unlock_mutex`（用于 owner 清理与统计）。
- `PSI_MUTEX_CALL` 有两种绑定：server 内部（`MYSQL_SERVER`/`PFS_DIRECT_CALL`）直接 include `pfs_mutex_provider.h` 直连 PFS 实现；否则走 service 句柄（组件 ABI）。

### 断言宏：release 下编译为空

```cpp
#ifdef SAFE_MUTEX
#define mysql_mutex_assert_owner(M) \
  safe_mutex_assert_owner((M)->m_mutex.m_u.m_safe_ptr);
#else
#define mysql_mutex_assert_owner(M) {}
#endif
```

**非 SAFE_MUTEX 构建下这些断言是空操作**——代码里遍布的 `mysql_mutex_assert_owner` 只在 debug 构建起作用。排查锁序/所有权问题时，必须用 debug 构建（这一条与 InnoDB 的 `UNIV_DEBUG` 同理）。

### SAFE_MUTEX：CMake 注入，不是 `#define` 开关

`SAFE_MUTEX` **不在任何头文件里定义**，而是 CMake 在 Debug 配置统一注入：

```cmake
# Add safemutex for debug configurations
FOREACH(LANG C CXX)
  STRING_PREPEND(CMAKE_${LANG}_FLAGS_DEBUG "-DENABLED_DEBUG_SYNC ")
  STRING_PREPEND(CMAKE_${LANG}_FLAGS_DEBUG "-DSAFE_MUTEX ")
ENDFOREACH()
```

`mysys/CMakeLists.txt` 印证：只有 flags 里有 `DSAFE_MUTEX` 时才编译 `thr_mutex.cc/thr_cond.cc`（inline 版够用）。

`safe_mutex_lock` 的核心（自死锁检测）：

```c
native_mutex_lock(&mp->global);
if (mp->count > 0) {
  if (try_lock) { ... return EBUSY; }
  else if (my_thread_equal(my_thread_self(), mp->thread)) {
    /* "Trying to lock mutex at %s, line %d, when the mutex was already
       locked at %s, line %d in thread T@%u" */
    abort();          /* fast mutex 下会悄悄死锁，这里直接报 */
  }
}
...
mp->thread = my_thread_self();
mp->file = file; mp->line = line;
```

### mutex 属性：ADAPTIVE_NP 的平台换挡

```c
#ifdef PTHREAD_ADAPTIVE_MUTEX_INITIALIZER_NP
extern native_mutexattr_t my_fast_mutexattr;
#define MY_MUTEX_INIT_FAST &my_fast_mutexattr
#else
#define MY_MUTEX_INIT_FAST NULL
#endif
```

初始化时的注释把权衡讲透：

```c
#ifdef PTHREAD_ADAPTIVE_MUTEX_INITIALIZER_NP
  /*
    Set mutex type to "fast" a.k.a "adaptive"
    In this case the thread may steal the mutex from some other thread
    that is waiting for the same mutex. This will save us some
    context switches but may cause a thread to 'starve forever' ...
  */
  pthread_mutexattr_settype(&my_fast_mutexattr, PTHREAD_MUTEX_ADAPTIVE_NP);
#endif
```

**同一个二进制语义在不同平台不同**：Linux（glibc）用自适应互斥（允许"偷锁"，省上下文切换、接受理论饿死）；macOS/Windows 宏未定义 → `NULL` → 默认类型。分析锁饥饿问题前必须先确认平台。

### mysql_rwlock_t：纯包装

```c
#ifdef _WIN32
struct native_rw_lock_t { SRWLOCK srwlock; BOOL have_exclusive_srwlock; };
#else
typedef pthread_rwlock_t native_rw_lock_t;   /* pthread_rwlockattr_t is not used in MySQL */
#endif
```

`inline_mysql_rwlock_rdlock` 与 mutex 版完全同构（`start_rwlock_rdwait` → `native_rw_rdlock` → `end_rwlock_rdwait`）。注释明确**不用 rwlock 属性**（即不设 prefer-writer 之类的策略，走平台默认）。

### ★ mysql_prlock_t：为 MDL 手搓的"强读者优先"锁

```c
struct rw_pr_lock_t {
  native_mutex_t lock;                  /* 保护本结构；wr-lock 期间也一直持有 */
  native_cond_t no_active_readers;
  unsigned int active_readers;
  unsigned int writers_waiting_readers;
  bool active_writer;
  my_thread_t writer_thread;            /* debug only */
};
```

```c
int rw_pr_rdlock(rw_pr_lock_t *rwlock) {
  native_mutex_lock(&rwlock->lock);
  /*
    The fact that we were able to acquire 'lock' mutex means
    that there are no active writers and we can acquire rd-lock.
  */
  rwlock->active_readers++;            /* ← 无条件成功，不等任何人 */
  native_mutex_unlock(&rwlock->lock);
  return 0;
}

int rw_pr_wrlock(rw_pr_lock_t *rwlock) {
  native_mutex_lock(&rwlock->lock);
  if (rwlock->active_readers != 0) {
    rwlock->writers_waiting_readers++;
    while (rwlock->active_readers != 0)
      native_cond_wait(&rwlock->no_active_readers, &rwlock->lock);  /* 等读者清零 */
    rwlock->writers_waiting_readers--;
  }
  /* ... Not releasing 'lock' mutex until unlock will block
     both requests for rd and wr-locks. ... wr-lock optimized. */
  rwlock->active_writer = true;
  return 0;
}
```

**"读者永不阻塞读者"的机制就藏在 `native_cond_wait` 的语义里**：写者 wrlock 时持有内部 `lock` mutex 并 `cond_wait`——`cond_wait` 会**释放 mutex**，所以后续读者的 `rdlock` 照样拿到内部 mutex，`active_readers++` 直接返回，**哪怕写者在排队**。写者醒来后重新持住 `lock` mutex 直到 unlock，此时 rd/wr 全被挡（写锁独占）。

对比普通 `pthread_rwlock`：glibc 的 `PTHREAD_RWLOCK_PREFER_READER_NP` 也偏向读者，但**不保证**"挂着读者时新读者必成功"（写者排队时可能挡住新读者）。prlock 的保证更强，且 `MDL_context::m_LOCK_waiting_for` 的注释点明了用途：

```cpp
  mysql_prlock_t m_LOCK_waiting_for;
  /**
    Tell the deadlock detector what metadata lock or table
    definition cache entry this session is waiting for. ...
    we'd very much like it to be readily available to the
    wait-for graph iterator.
  */
```

wait-for 图的边必须"随时可读"（检测器并发遍历）——这正是 prlock 的设计场景。注册上，prlock 复用 rwlock 的 PSI 通道但靠 flag 分流：

```c
/* sql/mdl.cc 的注册数组 */
static PSI_rwlock_info all_mdl_rwlocks[] = {
    {&key_MDL_lock_rwlock, "MDL_lock::rwlock", PSI_FLAG_RWLOCK_PR, 0, PSI_DOCUMENT_ME},
    {&key_MDL_context_LOCK_waiting_for, "MDL_context::LOCK_waiting_for",
     PSI_FLAG_RWLOCK_PR, 0, PSI_DOCUMENT_ME}};
```

PFS 侧按 flag 分前缀：

```c
if (info->m_flags & PSI_FLAG_RWLOCK_SX) {        /* wait/synch/sxlock/ */
} else if (info->m_flags & PSI_FLAG_RWLOCK_PR) { /* wait/synch/prlock/ */
} else {                                          /* wait/synch/rwlock/ */
}
```

### mysql_cond_t：与 mutex 的配对约定

```cpp
struct mysql_cond_t {
  native_cond_t m_cond;
  struct PSI_cond *m_psi;
};
```

`inline_mysql_cond_wait` 展示配对约定——**PFS 同时接收 cond 和 mutex 两个 `m_psi`**：

```cpp
locker = PSI_COND_CALL(start_cond_wait)(
    &state, that->m_psi, mutex->m_psi, PSI_COND_WAIT, src_file, src_line);
result = my_cond_wait(&that->m_cond, &mutex->m_mutex);
```

即 POSIX 约定（调用者必须持有该 mutex），PFS 侧借这次调用把 cond 等待与 mutex 关联记录，且等待期间线程不计为持锁者。`LOCK_xxx` ↔ `COND_xxx` 的命名配对是人工约定，key 各自注册。

### instrument 注册：一个新 mutex 的完整旅程

1. **声明 key**：`PSI_mutex_key key_LOCK_mynew;`
2. **进注册数组**（全局的在 `mysqld.cc` 的 `all_server_mutexes[]`，子系统自带自己的数组如 `mdl.cc` 的 `all_mdl_mutexes`）：

```c
static PSI_mutex_info all_server_mutexes[] = {
  { &key_LOCK_tc, "TC_LOG_MMAP::LOCK_tc", 0, 0, PSI_DOCUMENT_ME},
  { &key_LOCK_crypt, "LOCK_crypt", PSI_FLAG_SINGLETON, 0, PSI_DOCUMENT_ME},
  { &key_LOCK_thd_data, "THD::LOCK_thd_data", 0, PSI_VOLATILITY_SESSION, PSI_DOCUMENT_ME},
  ...
};
```

flags：`PSI_FLAG_SINGLETON` = 全局唯一实例；`PSI_VOLATILITY_SESSION` = 易变实例（PFS 据此做 instance 回收策略）。

3. **启动期注册**：`init_server_psi_keys()` 里 `mysql_mutex_register(category, all_server_mutexes, count)` → PFS 拼出 `wait/synch/mutex/sql/NAME` 建 `PFS_mutex_class`，把编号写回 `*info->m_key`。
4. **运行期 init**：`mysql_mutex_init(key, &m, MY_MUTEX_INIT_FAST)` → `m_psi = PSI_MUTEX_CALL(init_mutex)(key, &m)`。
5. 之后每次 `mysql_mutex_lock` 即可被 PFS 观测。

未注册 key 的 mutex（`key=0`）功能完全正常，只是 PFS 不可见。

---

## ★ 本机制里的工程实现技法

### 一、ABI 稳定层的结构设计

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| 类型定义放 `components/services/bits/` | 全部 L0/L2 类型 | 组件/插件只需 ABI bits 头，不需 server 内部头 | 类型改动受 ABI 约束，加了字段就摘不掉 |
| `m_psi` 无条件内嵌 | L2 全部结构 | 一份二进制接口覆盖埋点/非埋点两种构建 | 每实例 +8 字节 |
| 双重闸门（`m_psi != nullptr` + `m_psi->m_enabled`） | 埋点热路径 | 实例级动态开关，单 instrument 可在线关闭 | 两次分支；忘查 `m_enabled` 会白付计时开销 |
| `start_*_wait` 返回可空 locker | 全部 wait 埋点 | 按采样/开关决定是否结算 | 忘判空就 `end_*` 会崩 |

### 二、正交维度而非三选一

| 维度 | 教科书原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 调试 vs 埋点 | 通常做成编译期三选一（无/调试/埋点） | **两个独立编译维度**（`SAFE_MUTEX` × `HAVE_PSI_*`），2×2=4 种组合全部可用 | 官方 debug 构建同时具备"锁 bug 即 abort"与"全量观测"；代价是包装层多一次 if |
| 平台差异 | 通常 assert 同语义 | **ADAPTIVE_NP 只在 Linux 生效**，其他平台静默退化 | 同一份代码在不同平台公平性语义不同——分析饥饿问题先看平台 |

### 三、prlock：为单一用户定制原语

| 维度 | pthread_rwlock 原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 优先级保证 | `PREFER_READER_NP`：偏向读者但不保证新读者必成功 | **强保证**：挂着读者时新读者必成功（写者 `cond_wait` 时释放内部 mutex） | MDL 死锁检测器正确性依赖；代价是**写者可被读者流饿死**（MDL 场景写极稀少） |
| 生命周期 | unlock 后仍不能 destroy（glibc 语义） | **unlocked 即可 destroy**（unlock 改完状态后不再碰锁数据，signal 先于放 mutex） | MDL 需要销毁灵活；这是一个对实现顺序的硬约束，写错就是 use-after-free |
| 无竞争路径 | pthread_rwlock 有内部 atomic 排队状态 | wr-lock 退化为 mutex lock/unlock（注释 "wr-lock optimized"） | 贴合 MDL 高频短临界区 |

### 四、`my_atomic.h` 的消亡

8.0.39 里它只剩 `LF_BACKOFF`（Windows 下 `YieldProcessor()` 循环 200 次、其他平台 `1` 的自旋宏）——**原子操作已全面让位给 `std::atomic`**。这是"自研封装随平台能力补齐而退役"的典型例子，读旧代码时别再把 `my_atomic_*` 当现存 API。

---

## 可观测性

### instrument 前缀一览

| 前缀 | 类型 | 例 |
|---|---|---|
| `wait/synch/mutex/sql/NAME` | `mysql_mutex_t` | `wait/synch/mutex/sql/LOCK_open` |
| `wait/synch/rwlock/sql/NAME` | `mysql_rwlock_t` | `wait/synch/rwlock/sql/global_sid_lock` |
| `wait/synch/prlock/sql/NAME` | `mysql_prlock_t` | `wait/synch/prlock/sql/MDL_context::LOCK_waiting_for` |
| `wait/synch/sxlock/innodb/NAME` | InnoDB `rw_lock_t` 的 SX 用法 | `wait/synch/sxlock/innodb/dict_operation_lock` |
| `wait/synch/cond/sql/NAME` | `mysql_cond_t` | `wait/synch/cond/sql/COND_open` |

（prlock/sxlock **没有独立的 PSI 接口**，复用 rwlock 通道、靠 `PSI_FLAG_RWLOCK_PR/SX` flag 分流前缀。）

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 当前谁卡在哪把 mutex 上 | PFS | `events_waits_current`，`event_name LIKE 'wait/synch/%'`，`object_name` 即 instrument 名 |
| 哪把锁最热（累计等待最久） | PFS | `events_waits_summary_global_by_event_name` 按 `sum_timer_wait DESC` |
| PFS 对象与源码实例的对应 | PFS | `OBJECT_INSTANCE_BEGIN` = `mysql_mutex_t.m_psi` 指向的 `PFS_mutex` 地址 |
| 锁 bug（自死锁/重复解锁） | debug 构建 | SAFE_MUTEX 的 `abort()`，错误消息带加锁位置（file/line） |

一次 mutex 等待的完整链路：`inline_mysql_mutex_lock` → `pfs_start_mutex_wait_v1`（取 `PFS_thread`、记 `timer_start`）→ 阻塞在 `pthread_mutex_lock` → `pfs_end_mutex_wait_v1` 结算耗时并推进 `events_waits_current`（每线程一条，环形覆盖，满后进 `_history`）。

---

## Misc

### 面向二次开发

**扩展点**：新增一把可观测的 mutex = 声明 `PSI_mutex_key` → 加入所属模块的注册数组 → `mysql_mutex_init(key, ...)` → 全部用 `mysql_mutex_*` 包装函数。**新增一种原语类型**（如 prlock 这种）= 在 bits 头加类型、实现 mysys 版、加 `mysql/psi/mysql_xxx.h` 包装层、PFS 侧加 provider 接口与 flag 分流——成本很高，优先考虑复用 rwlock + flag。

**坑与已知缺陷**：

1. **release 构建下所有 `mysql_mutex_assert_owner` 是空操作**——锁所有权错误只能靠 SAFE_MUTEX（debug）构建暴露。
2. **ADAPTIVE_NP 允许"偷锁"**：理论上单线程可永久饿死（注释自认）。遇到"某线程拿不到锁"的诡异案例，先确认平台。
3. **prlock 写者可被读者流饿死**——设计如此（MDL 写极稀少），不要在写频繁的场景复用 `rw_pr_lock_t`。
4. **prlock 的 unlock 顺序契约**（signal 先于放内部 mutex、之后不碰锁数据）是实现细节级约束，改代码时极易破坏"unlocked 即可 destroy"。
5. `my_atomic_*` 已退役（只剩 `LF_BACKOFF`），新代码用 `std::atomic`。
6. **`DISABLE_MYSQL_THREAD_H` / `DISABLE_MYSQL_PRLOCK_H` 不是关闭埋点的总闸**，只是切断头文件依赖链的 workaround；真正开关是 `DISABLE_PSI_<TYPE>` 系列。

**社区边界澄清**：Percona 版对 PFS 有额外增强（如 thread instrumentation 的补充 instrument），社区版 8.0.39 以本篇为准。

### 易混淆概念

- **`native_mutex_*` vs `my_mutex_*` vs `mysql_mutex_*`**：分别是不带调试/带 SAFE_MUTEX/带 PFS 的三层入口。业务代码只用第三层。
- **`SAFE_MUTEX` 与 PFS 不互斥**（正交维度），但 **`mysql_mutex_assert_owner` 只在 SAFE_MUTEX 下有实现**。
- **prlock ≠ 普通 rwlock**：它是手搓的强读者优先锁，专为 MDL 死锁检测器定制；`wait/synch/prlock/` 前缀下没有一套独立的锁算法，只有一种 `rw_pr_lock_t`。
- **`key=0`（`PSI_NOT_INSTRUMENTED`）不等于"没埋点代码"**：埋点代码常驻，只是该实例 `m_psi == nullptr` 走快分支。

---

## 参考

**论文 / 经典算法**
- P. J. Courtois, F. Heymans, D. L. Parnas. *Concurrent Control with "Readers" and "Writers"*. CACM 1971.（读者-写者问题的优先级变体；prlock 是"强读者优先"落地，落点 `rw_pr_rdlock` 无条件成功）

**官方文档**
- *MySQL 8.0 Reference Manual → Performance Schema Instrument Naming Conventions*（wait/synch 层级）
- *MySQL 8.0 Reference Manual → The Performance Schema events_waits_* Tables*

**相关文档**
- InnoDB 自研原语（`rw_lock_t` S/SX/X、`os_event`、PolicyMutex 模板族、sync0arr）见 [`innodb_sync.md`](innodb_sync.md)——同一仓库里"包装 OS"与"完全自研"两种哲学的对照
- RCU 见 [`rcu.md`](rcu.md)
- 全局实例清单（A2：server 侧 mutex/rwlock/cond 全量）见 [`../README.md`](../README.md)
- prlock 最大的用户 MDL 见 [`../transactional/mdl.md`](../transactional/mdl.md)
