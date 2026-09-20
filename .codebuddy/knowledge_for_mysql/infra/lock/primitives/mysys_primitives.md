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

`mysql_prlock_t` 的底座 `rw_pr_lock_t` **不是** pthread_rwlock 的包装，而是 mysys 用 mutex + cond 手搓的"优先读者"锁。它给出的两条硬保证（头注释原文翻译）：

> 1) **"prefer readers"**：不允许"锁已被 rd-lock，且存在一个被阻塞的 pending rd-lock（例如被 wr-lock 请求挡住）"的局面。**这是比 Linux `PTHREAD_RWLOCK_PREFER_READER_NP` 更强的保证**。MDL 子系统的死锁检测器的正确性依赖它。
> 2) **为无竞争的 wr-lock/unlock 优化**（退化为 mutex 的 lock/unlock）。

为什么 MDL 需要第一条：死锁检测遍历 wait-for 图时多线程并发 `rdlock`（读边），只有单线程改边时 `wrlock`。若普通 rwlock 在写者排队时挡住新读者，**检测器自身会被卡住**——这不是性能问题，是正确性问题。代价是**写者可能被读者流饿死**，MDL 场景写极稀少，可接受。

还有一条隐蔽的契约（`rw_pr_unlock` 注释）：**unlock 改完状态后不得再碰锁数据**——这样 MDL 允许"锁一进 unlocked 态就 destroy"。为此 signal 必须在释放内部 mutex **之前**做。

### 理论溯源

- **读者-写者问题的优先级变体**：Courtois et al. (1971) 给出读者优先/写者优先/公平三种原型。prlock 是"**强读者优先**"变体——比教科书"读者优先"多一条"读者间绝不互相阻塞（即使有写者排队）"的保证，落点是 `rw_pr_rdlock` 无条件 `active_readers++` 成功。
- **PFS 的低开销观测模型**：静态注册（key 预分配）+ 动态实例钩子（`m_psi`）+ 双重闸门（全局 `m_enabled`/实例 enabled）——思想与 eBPF 的静态探针类似：埋点代码常驻，开销由开关控制。

### 算法与数据结构

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
