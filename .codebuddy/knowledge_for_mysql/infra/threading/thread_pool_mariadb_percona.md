# MariaDB 与 Percona 线程池源码级剖析及三方对比

> **分析基准**
> - **MariaDB Server 12.3.3**：`/Users/morphyzhang/Documents/Code/DB/MariaDB`
>   - `sql/threadpool.h`(166) / `sql/threadpool_generic.h`(157) / `sql/threadpool_generic.cc`(1770) / `sql/threadpool_common.cc`(639)
> - **Percona Server for MySQL 8.0.43-34**：`/Users/morphyzhang/Documents/Code/DB/percona-server`
>   - `sql/threadpool.h`(86) / `sql/threadpool_unix.cc`(1620) / `sql/threadpool_common.cc`(268)
> - **对照方 MySQL Community 26.7.0**（`plugin/thread_pool/`，本地副本 `.tmp_tp/`）
>
> 行号引用格式 `file:line`。
>
> **边界**：本篇讲 **MariaDB / Percona 两家**的线程池实现与三方对比；MySQL 26.7 侧完整机制见 [`thread_pool.md`](thread_pool.md)，Per_thread 连接模型见 [`connection_threading.md`](connection_threading.md)。
>
> **本文定位**：源码级剖析 MariaDB 与 Percona 两家的线程池实现（这两家同源），并与 MySQL 26.7 做逐机制对比。所有结论均标注源码位置或文档出处，可复核。

---

## 目录

- [概述](#概述)　（是什么 / 用途 / 版本演进）
- [理论基础](#理论基础)　（设计思想与权衡 / 理论溯源 / 算法与数据结构）
- [核心实现](#核心实现)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [性能数据](#性能数据)
- [11. MariaDB / Percona 运维实践](#11-mariadb--percona-运维实践)
- [Misc](#misc)
- [参考](#参考)

> 「核心实现」内部保留编号小节以便交叉引用：

1. [血缘关系：Percona 移植自 MariaDB](#c1)
2. [MariaDB 源码剖析](#c2)
   - 2.1 数据结构 / 2.2 tp_init 与组初始化 / 2.3 连接分配与认证
   - 2.4 listener / 2.5 get_event（核心循环）/ 2.6 worker_main
   - 2.7 wait_begin / wait_end / 2.8 check_stall 与 timer thread
   - 2.9 start_io 与 change_group / 2.10 唤醒与节流 / 2.11 io_poll 多平台
3. [Percona 源码剖析](#c3)
   - 3.1 结构差异 / 3.2 双重过载判定 / 3.3 queue_get 节流
   - 3.4 get_event 的 tickets 处理 / 3.5 wait_begin 的编译期开关
   - 3.6 start_io / 3.7 handle_event 与 worker_main / 3.8 tp_init
4. [逐函数源码对照](#c4)
5. [核心机制三方对比](#c5)
6. [oversubscribe 语义辨析](#c6)
6.5 [stall 检测三家源码级辨析](#65-stall-检测三家源码级辨析)
6.6 [防饿死三家源码级辨析](#66-防饿死三家源码级辨析)
7. [死锁：经典案例与三家对策](#c7)
8. [行业演进：AliSQL 与 TXSQL](#c8)

---

## 概述

### 是什么

MariaDB 与 Percona 的线程池是**同一开源血缘的两条分支**：MariaDB 5.5 首创（内建、GPL），Percona 5.5–5.7 移植并加入优先级调度。二者与 MySQL 企业版线程池（独立起源、插件、长期闭源）是**并行演化**关系，不是派生关系。

### 用途

与 MySQL 线程池同一目标：在连接数远超 CPU 核数时，用"限制同时推进的语句数"换取吞吐不崩塌。但在**实现路径**上做了完全不同的取舍——内建而非插件、队列排空速率而非单查询计时、tickets 而非 kickup（Percona）。

本篇要回答的核心问题：**同一份设计，两个分支各自演化出了什么差异？这些差异说明了什么？**

### 版本演进

| 版本 | 变化 |
|---|---|
| MariaDB 5.5 | **首创**线程池，内建于 server，GPL 开源 |
| Percona 5.5–5.7 | **移植自 MariaDB**，新增优先级调度（`high_prio_mode` + `tickets`） |
| MariaDB 10.2 | 加入优先级调度（`thread_pool_priority`），方式与 Percona 类似但细节不同 |
| MariaDB 10.5 | 新增 `thread_pool_dedicated_listener`、`thread_pool_exact_stats`、**线程创建节流** |
| Percona 8.0.12-1 | 原生支持线程池（自 5.7 移植） |
| **Percona 8.0.14** | **破坏性变更**：`extra_port` / `extra_max_connections` 被上游 `admin_port` 取代并移除，配置保留会**启动失败** |
| Percona 8.0.43-34 | 本文分析基准 |
| MariaDB 12.3.3 | 本文分析基准（实现已重构成 `threadpool_generic.cc`） |

---

## 理论基础

### 设计思想与权衡

#### 决策一：内建而非插件（对比 MySQL）

MariaDB/Percona 选择编译进内核，MySQL 选择插件。

- **收益**：可以更紧耦合——直接调用内核函数，无需 `thread_pool_priv.h` 那层 ABI 间接，也无需把 THD 包装成不透明指针
- **代价**：**无法独立演进或跨版本搬移**。Percona 文档明说 "this implementation is **not compatible with upstream**"。而 MySQL 的插件化让它能在 2026 年把整个实现从企业版搬到社区版，**内核零改动**

> 这正是前面讨论的"ABI 隔离"价值的反面印证：MySQL 多付了一层间接的成本，换来了 15 年后的可移植性。

**源码落点**：内建的体现是 `tp_init`（3.8）直接初始化所有线程组、直接调 `io_poll_create()`，没有任何"向内核注册"的动作；对比 MySQL 1.1 必须先 `my_connection_handler_set(&thread_pool_handler_functions, &thread_pool_event_functions)` 注册两套回调才能工作。代价那句 "not compatible with upstream" 出自 Percona 文档——内建意味着**必须与内核版本同步演进**。

#### 决策二：stall 用"队列排空速率"而非"单查询计时"

| | MariaDB / Percona | MySQL 26.7 |
|---|---|---|
| 判据 | `queue_event_count == 0` 且队列非空 | `now - time_of_attach > 60ms` |
| 语义 | "这个组停止推进了" | "这个查询跑太久了" |
| 开销 | 两个计数器，O(1) | 需遍历组内所有线程，维护 `time_of_attach` |
| 精度 | 只能定位到组 | 能定位到具体线程 |

**源码落点**：见 6.5 节 `check_stall` 完整源码——`queue_event_count` 每轮被 timer 重置为 0，判定条件就是"计数仍为 0 且队列非空"。注意两家的 `io_event_count` / `queue_event_count` 两个计数器正是为这个判定服务的（MySQL 侧对应物是 `mark_and_count_stalled_threads` 遍历统计）。

**代价**：MariaDB/Percona 无法对"被判定卡住的查询"做差异化处理（MySQL 会给 stalled 线程独立名额）；且**检查周期 = stall_limit 本身**（默认 500ms），响应比 MySQL 的 1ms/10ms 自适应慢一个数量级。

#### 决策三：Percona 引入 busy 计数（active + waiting）

`too_many_busy_threads` 把"活跃的 + 等待中的"一起计数。理由写在注释里：

> A thread is busy if it is in either the active state or the waiting state (i.e. between `thd_wait_begin()` / `thd_wait_end()` calls).

等待中的线程虽不占 CPU，但**仍持有事务和锁**——只数活跃线程会低估争用。这是从"限线程数"走向"限事务数"的中间形态（MySQL 26.7 的 MTL 是终态）。

**源码落点**：见 3.2 节 `too_many_busy_threads` 源码与 3.1 的 `waiting_thread_count` 字段。MySQL 26.7 的对应物是 `trxn_threads`（6.1 三档判定第一档）——两者都在解决"活跃≠占资源"，但 Percona 数的是"等待中的线程"，MySQL 数的是"持有事务的连接"，后者更贴近争用源头。

#### 决策四：防饿死的两条路线

- **MariaDB**：`thread_pool_prio_kickup_timer`——低优等待超时后**升级**到高优。简单，但高负载时搬运速率有限
- **Percona**：`thread_pool_high_prio_tickets`——限制**每个连接能插队多少次**。更公平，防止单连接长期霸占

**代价与失效场景**：

- Percona 的 `too_many_busy_threads` 会让低优队列**被视为"空"**（`queues_are_empty`），从而**不触发 stall**——组已经很忙时不再雪上加霜，但也意味着此时低优查询可能长时间得不到处理
- MariaDB 注释承认的退化路径：**"in case there is a flood of very long queries, threadpool would slowly approach thread-per-connection behavior"**——极端长查询洪泛下线程池会退化成 thread-per-connection，这是设计上可接受的兜底

#### 决策五：为什么不做负载均衡

三家都用 `thread_id % group_count`，**都不考虑负载**。dbkernel 评价："繁忙的连接可能会恰巧被分配到相同的线程组，从而导致负载不均衡"。

不做的原因是**成本**：维护全局负载序列需要全局锁（TXSQL 原文："线程组每处理完一次任务都可能需要更新自身的状态，并在全局锁的保护下更新其在全局负载序列中的位置，如此一来对性能的影响势必较大"）。

> 只有 TXSQL 做了局部负载均衡（只与相邻 1-2 个组比较），MariaDB 只在 `thread_pool_size` 变化时做 `change_group` 迁移。

**源码落点**：`start_io`（2.9 / 3.6）里就是一句 `thread_id % group_count`——**没有任何负载查询**。对照 TXSQL 需要维护全局负载序列并在全局锁保护下更新位置（3.6 节引述），成本差异一目了然。MySQL 26.7 侧对应的是 `assign_connection_to_thread_group` 的 round-robin（同样是纯取模，不考虑负载）。

### 理论溯源

| 理论 / 经典技术 | 源码落点 |
|---|---|
| **Reactor / 事件驱动** | `io_poll_wait(thread_group->pollfd, ev, MAX_EVENTS, -1)`（MariaDB 723 / Percona 637） |
| **LIFO 唤醒保 cache** | `waiting_threads.push_front(current_thread)` —— 注释原话 "ensures LIFO wakeup order (hot caches, working inactivity timeout)" |
| **self-pipe trick** | MariaDB `wake_listener`注释明写 "Self-pipe trick is used in most cases, except IOCP" |
| **多级队列 + 优先级老化（aging）** | MariaDB `check_stall` 里的 prio kickup；Percona 的 `high_prio_queue` + tickets |
| **准入控制 / 过载节流** | Percona `too_many_busy_threads` + normal-priority queue throttling |
| **动态角色（角色切换）** | worker ↔ listener 互换（两者都是 `get_event` 里"无 listener 则自己变 listener"） |
| **Ticket 配额（公平调度）** | Percona `connection->tickets--` / 恢复 |

### 算法与数据结构

| 结构 / 算法 | 用途 | 复杂度 |
|---|---|---|
| `I_P_List` 侵入式链表 | 连接同时挂队列、线程挂 waiting_threads，零额外分配 | O(1) 插删 |
| 固定数组 `all_groups[MAX_THREAD_GROUPS]` | 组槽位（MariaDB 100000 / Percona 1024） | O(1) 索引 |
| `native_event` typedef 抽象 | epoll / kevent / event ports / IOCP 四平台统一 | — |
| **Percona `alignas(128)` + `padding[328]` + `static_assert(sizeof==512)`** | 强制结构体占 512 字节（8 cache line）**彻底防组间 false sharing** | — |
| **MariaDB `pad[CPU_LEVEL1_DCACHE_LINESIZE]`** | 同样的目的，但只填一个 cache line | — |
| 条件变量 + `woken` 标志 | 区分真实唤醒 vs 虚假唤醒/超时 | O(1) |
| 分档节流 `microsecond_throttling_interval()` | 线程数越多，创建下一个要等越久 | O(1) |

> **Percona 的 `static_assert` 比 MariaDB 的 `pad` 更硬**：直接把结构体尺寸钉死，编译期就能发现布局变化。

---

## 1.5 三家设计哲学对照（为什么各自长成这样）

> 前面 5 个决策是"某一点上谁选了什么"；本节回答更上层的问题：**三家各自的设计目标是什么，这些目标如何把它们塑造成不同的形状**。

### MariaDB 的四条设计目标（官方原文）与源码落点

MariaDB 官方文档明确列出线程池的四条设计目标，每一条都能在源码里找到落点：

| 目标（官方原文） | 含义 | 源码落点 |
|---|---|---|
| **Make the pool dynamic, so that it will grow and shrink whenever required** | 池随负载伸缩，不是固定大小 | `wake_or_create_thread`（按需创建）+ `thread_pool_idle_timeout`（空闲退休） |
| **Minimize the amount of overhead that is required to maintain the thread pool itself** | 池自身的维护开销要尽可能小 | 侵入式 `I_P_List` 队列（无额外节点分配）、`I_P_List_null_counter`（不维护计数）、无 per-connection 锁（靠 `EPOLLONESHOT`） |
| **Make the best use of underlying OS capabilities** | 充分利用操作系统原生能力 | `io_poll` 多平台封装：Linux epoll / FreeBSD+macOS kqueue / Solaris event ports / Windows IOCP（原生 `CreateThreadpool`） |
| **Limit the resources used by threads** | 限制线程占用的资源 | `thread_pool_max_threads` + **"每组至少 2 线程"防死锁的硬保证** |

**"dynamic and adaptive"是 MariaDB 最核心的自我定位**——它不追求精确控制并发数，而是追求"池能自己伸缩到合适大小"。这与 MySQL 26.7 的"精确限制事务数（MTL）"是**两种不同的治理哲学**：

```
MariaDB： 让池自己伸缩 → 用 max_threads 兜底 → 靠 stall 检测防死锁
MySQL 26.7：精确限制并发事务数 → 用 MTL + 长事务背压 → 靠 stalled 不计配额防饿死
```

前者是**自适应**（adaptive），后者是**精确控制**（controlled）。Percona 介于两者之间：继承 MariaDB 的自适应，又加了 `busy` 计数做**准入控制**。

### Percona 的三条增量设计

Percona 是 MariaDB 的移植（血缘见第 1 章），它的设计目标是**在 MariaDB 基础上补三块**：

| 增量 | 动机 | 源码落点 |
|---|---|---|
| **优先级调度** | MariaDB 早期没有细粒度优先级 | `thread_pool_high_prio_mode` + `high_prio_tickets`（票据消耗与重置） |
| **锁感知 / busy 计数** | 只算"活跃线程"会低估真实负载——**正在等锁的线程也占着连接** | `waiting_thread_count` + `too_many_busy_threads(active + waiting)` |
| **过载时的准入节流** | 池已经饱和时，不该再无脑接收低优查询 | `queue_get` 对低优队列节流 + `queues_are_empty` 把低优队列视为空 |

第三条最微妙：**Percona 在池过载时会"假装低优队列是空的"**，从而不触发 stall、不新建线程——这是主动的过载拒绝，而不是被动的排队等待。

### 三方设计哲学一句话总结

| | 设计哲学 | 并发控制手段 | 防死锁手段 | 防饿死手段 |
|---|---|---|---|---|
| **MySQL 26.7** | **精确控制**（限事务数） | MTL + 长事务背压 | stalled 不计配额（给长查询额外名额） | `prio_kickup_timer` 踢升 |
| **MariaDB** | **自适应伸缩** | `oversubscribe`（稳态收敛，官方称内部用途） | **每组至少 2 线程**硬保证 | `prio_kickup_timer` 踢升 |
| **Percona** | **自适应 + 准入控制** | `oversubscribe` + `busy` 计数双判定 | 继承 MariaDB 的 2 线程保证 | `high_prio_tickets` 票据 |

> 这张表是理解三家差异的骨架：**同样的"线程池"三个字，底下是三种不同的治理思路**。MySQL 相信"精确限制事务数能压住争用"；MariaDB 相信"池自己会伸缩到合适大小"；Percona 在 MariaDB 基础上加了"饱和时主动拒绝"。

## 核心实现

<a id="c1"></a>
## 1. 血缘关系：Percona 移植自 MariaDB

这不是推测，有双重证据。

**证据一：函数结构逐行对应**

> 下表及结论中的行号是**证据本身**——两个独立分支的同名函数落在完全相同的行号，是"移植"而非"各自实现"的铁证。这是本篇**唯一保留行号**的地方（其余位置一律按模板不写行号），请勿当作位置索引使用。

| 函数 | MariaDB (threadpool_generic.cc) | Percona (threadpool_unix.cc) |
|---|---|---|
| `wake_thread` | 1033 | 919 |
| `queue_put` | 1122 | 992 |
| `timeout_check` | — | 465 |
| `check_stall` | 612 | 546 |
| `timer_thread` | 556 | 494 |
| `listener` | 723 | 637 |
| `create_worker` | 786 | 786 |
| `wake_or_create_thread` | 856 | 856 |
| `queue_get` | — | 412 |
| `get_event` | 1169 | 1030 |
| `wait_begin` / `wait_end` | 1283 / 1310 | 1145 / 1173 |
| `start_io` | 1498 | 1390 |
| `handle_event` | — | 1421 |
| `worker_main` | 1447 | 1447 |
| `tp_init` | 1487 | 1487 |

连 `worker_main`、`tp_init`、`create_worker`、`wake_or_create_thread` 的**行号都完全一致**（1447 / 1487 / 786 / 856）。

**证据二：注释原文雷同**

`get_event` 中的休眠注释在两份代码中一字不差：

```cpp
/*
  Add current thread to the head of the waiting list  and wait.
  It is important to add thread to the head rather than tail
  as it ensures LIFO wakeup order (hot caches, working inactivity timeout)
*/
```

**证据三：官方互相承认**

- MariaDB 文档："Percona's implementation is **a port of the MariaDB's thread pool** with some added features."
- Percona 文档："this implementation is not compatible with upstream: **Built into the server**; does not minimize the number of concurrent transactions."

**而 MySQL 是独立起源**（5.5 同期，闭源插件），与这条开源主线并行演化。这解释了为什么 MySQL 的内部结构（双队列命名反置、per-thread stall 判定、冷热线程分层）与 MariaDB 差异巨大。

---

<a id="c2"></a>
## 2. MariaDB 源码剖析

### 2.1 数据结构（threadpool_generic.h）

```cpp
/* Per-thread structure for workers */
struct worker_thread_t {
  ulonglong  event_count;          /* number of request handled by this thread */
  thread_group_t* thread_group;
  worker_thread_t* next_in_list;
  worker_thread_t** prev_in_list;
  mysql_cond_t  cond;
  bool          woken;             // 用于区分真实唤醒 vs 虚假唤醒/超时
};

struct TP_connection_generic : public TP_connection {
  thread_group_t* thread_group;
  TP_connection_generic* next_in_queue;
  TP_connection_generic** prev_in_queue;
  ulonglong abs_wait_timeout;
  ulonglong enqueue_time;          // 用于 prio_kickup 判定
  TP_file_handle fd;
  bool bound_to_poll_descriptor;
  int waiting;
  bool fix_group;                  // 需要重新计算所属组的标志
};

const int NQUEUES = 2;             /* high and low priority queues */

enum class operation_origin { WORKER, LISTENER };   // 区分事件来源，用于分类统计

struct thread_group_counters_t {
  ulonglong thread_creations;
  ulonglong thread_creations_due_to_stall;    // ★ 区分 stall 引起的建线程
  ulonglong wakes;
  ulonglong wakes_due_to_stall;               // ★ 区分 stall 引起的唤醒
  ulonglong throttles;
  ulonglong stalls;
  ulonglong dequeues[2];                      // 按优先级分类
  ulonglong polls[2];                         // 按 WORKER/LISTENER 分类
};

struct thread_group_t {
  mysql_mutex_t mutex;
  connection_queue_t queues[NQUEUES];
  worker_list_t waiting_threads;
  worker_thread_t* listener;
  pthread_attr_t* pthread_attr;
  TP_file_handle  pollfd;
  int  thread_count;
  int  active_thread_count;
  int  connection_count;
  /* Stats for the deadlock detection timer routine.*/
  int io_event_count;              // listener 收到事件时 ++
  int queue_event_count;           // worker 从队列取事件时 ++
  ulonglong last_thread_creation_time;
  int  shutdown_pipe[2];
  bool shutdown;
  bool stalled;
  thread_group_counters_t counters;
  char pad[CPU_LEVEL1_DCACHE_LINESIZE];       // ★ 缓存行填充，防 false sharing
};
```

**要点**：

1. **`io_event_count` / `queue_event_count` 两个计数器是 stall 检测的全部依据**（注释直说 "Stats for the deadlock detection timer routine"）
2. **`counters` 区分了 stall 引起的建线程/唤醒** —— 这是诊断"线程池是否在跟负载对抗"的关键指标
3. **`pad[CPU_LEVEL1_DCACHE_LINESIZE]` 缓存行填充** —— 与 MySQL 的 cache line 工程同类手法
4. **`operation_origin` 枚举**（WORKER/LISTENER）让 `polls[2]`/`dequeues[2]` 能区分事件来源

### 2.2 tp_init 与组初始化

```cpp
bool tp_init() {
  /* 分配 all_groups（threadpool_max_size 个 thread_group_t） */
  pool_timer.tick_interval = threadpool_stall_limit;   // timer 周期 = stall_limit
  start_timer(&pool_timer);
}
```

注意 **`tick_interval = threadpool_stall_limit`** —— 在 MariaDB 里 `stall_limit` 既是"多久算停滞"，也是 timer 线程的检查周期，两者是同一个数（默认 500ms）。这与 MySQL 把 stall 判定（60ms）和 stall 检查周期（1ms/10ms）**解耦**形成鲜明对比。

> Percona 对应：`void tp_start_timer_thread() noexcept { pool_timer.tick_interval = threadpool_stall_limit; start_timer(&pool_timer); }`

### 2.3 连接分配与认证

```cpp
/* Assign connection to a group. */
thread_group_t *group = &all_groups[get_group_id(c->thread_id)];   // thread_id % group_count
mysql_mutex_lock(&group->mutex);
group->connection_count++;
mysql_mutex_unlock(&group->mutex);
```

**关键**：`tp_add_connection` 只是把连接**放入低优先级队列**，`active_thread_count == 0` 时唤醒/创建线程：

```cpp
static void queue_put(thread_group_t *thread_group, TP_connection_generic *connection) {
  connection->enqueue_time = threadpool_exact_stats ? microsecond_interval_timer()
                                                    : pool_timer.current_microtime;
  thread_group->queues[connection->priority].push_back(connection);
  if (thread_group->active_thread_count == 0)
    wake_or_create_thread(thread_group);                    // 1122-1133
}
```

> 阿里云源码剖析明确指出：**"连接尚未 login，认证等工作在 worker 线程里完成"** —— 这意味着认证（可能涉及 DNS 反查、权限表读盘）会占用一个 worker 线程。MySQL 26.7 用专门的 `CONNECTION_HANDLER_WORKER_THREAD` 隔离这件事，是架构上的改进。

### 2.4 listener

```cpp
for (;;) {
  if (thread_group->shutdown) break;
  native_event ev[MAX_EVENTS];
  int cnt = io_poll_wait(thread_group->pollfd, ev, MAX_EVENTS, -1);   // 阻塞等待
  TP_INCREMENT_GROUP_COUNTER(thread_group, polls[(int)operation_origin::LISTENER]);
  if (cnt <= 0) break;

  mysql_mutex_lock(&thread_group->mutex);
  thread_group->io_event_count += cnt;
  /* ... 决策：自己处理还是入队 ... */
}
```

**决策逻辑**（755-800 的完整注释）：

```
Q1 : Should listener handle an event itself, or put all events into queue?

Solution :
  Generally, listener that handles events itself is preferable. We do not
  want listener thread to change its state from waiting to running too often.
  Since listener has just woken from poll, it better uses its time slice and
  does some work. Besides, not handling events means they go to the queue,
  and often to wake another worker must wake up to handle the event. This is
  not good, as we want to avoid wakeups.

  The downside of listener that also handles queries is that we can
  potentially leave thread group for long time not picking the new network
  events... relying on timer is not always good, because it may "tick" too
  slow (large timer_interval)

  We use following strategy - if queue was not empty we suspect flood of
  network events and listener stays, Otherwise, it handles a query.

Q2: If queue is not empty, how many workers to wake?

Solution:
  We generally try to keep one thread per group active... we will wake only
  one worker, and only if there is not active threads currently, and listener
  is not going to handle a query.
```

即：**listener 默认自己处理单个事件**（利用刚唤醒的时间片和 cache），但若发现队列非空（事件洪峰），listener 专收事件不再执行。

> MySQL 的 `can_process_direct` / `process_direct_query` 是同一思想的不同实现。

### 2.5 get_event—— 核心循环

这是 MariaDB worker 线程的心脏：

```cpp
TP_connection_generic *get_event(worker_thread_t *current_thread,
                                 thread_group_t *thread_group, struct timespec *abstime) {
  mysql_mutex_lock(&thread_group->mutex);
  for (;;) {
    int err = 0;
    bool oversubscribed = too_many_threads(thread_group);       // 1182
    if (thread_group->shutdown) break;

    /* ① 先尝试从队列取 */
    if (!oversubscribed) {
      connection = queue_get(thread_group, operation_origin::WORKER);
      if (connection) break;
    }

    /* ② 组内没有 listener，则自己变成 listener */
    if (!thread_group->listener) {
      thread_group->listener = current_thread;
      thread_group->active_thread_count--;                      // listener 不算活跃
      mysql_mutex_unlock(&thread_group->mutex);
      connection = listener(current_thread, thread_group);      // 阻塞在 io_poll_wait
      mysql_mutex_lock(&thread_group->mutex);
      thread_group->active_thread_count++;
      thread_group->listener = NULL;                            // listener 回来了
      break;
    }

    /* ③ 睡前最后一次尝试：非阻塞轮询（timeout=0） */
    if (!oversubscribed && !threadpool_dedicated_listener) {
      native_event ev[MAX_EVENTS];
      int cnt = io_poll_wait(thread_group->pollfd, ev, MAX_EVENTS, 0);
      TP_INCREMENT_GROUP_COUNTER(thread_group, polls[(int)operation_origin::WORKER]);
      if (cnt > 0) {
        queue_put(thread_group, ev, cnt);
        connection = queue_get(thread_group, operation_origin::WORKER);
        break;
      }
    }

    /* ④ 真的睡 */
    current_thread->woken = false;
    /*
      Add current thread to the head of the waiting list and wait.
      It is important to add thread to the head rather than tail
      as it ensures LIFO wakeup order (hot caches, working inactivity timeout)
    */
    thread_group->waiting_threads.push_front(current_thread);   // ★ LIFO
    thread_group->active_thread_count--;
    if (abstime) err = mysql_cond_timedwait(&current_thread->cond, &thread_group->mutex, abstime);
    else         err = mysql_cond_wait(&current_thread->cond, &thread_group->mutex);
    thread_group->active_thread_count++;

    if (!current_thread->woken) {
      /* 未被 wake() 显式唤醒 → 虚假唤醒或超时，自己从链表摘除 */
      thread_group->waiting_threads.remove(current_thread);
    }
    if (err) break;
  }
  thread_group->stalled = false;                                // ★ 出队即清 stall 标志
  mysql_mutex_unlock(&thread_group->mutex);
  return connection;
}
```

**六个设计要点**：

1. **`listener` 是动态角色**：谁发现组内没有 listener，谁就临时变成 listener。这比 MySQL 的 `waiting_thread` 角色更彻底（MySQL 的 waiting thread 只是收事件，MariaDB 会真的阻塞在 `io_poll_wait` 直到有事件）
2. **listener 不计入 `active_thread_count`**（1200 行 `--`）
3. **`too_many_threads` 决定是否从队列取**：`active > 1 + oversubscribe && !stalled`。注意 **`!stalled`** —— 组被判 stall 后恒为 false，允许超额取事件（放开并行）
4. **睡前非阻塞轮询**：避免"刚要睡就有事件来"的唤醒开销
5. **LIFO 唤醒**（`push_front`）：最后睡的先醒，保 cache 热度
6. **`woken` 标志处理虚假唤醒**：未被显式唤醒的线程自己从链表摘除

### 2.6 worker_main

```cpp
static void *worker_main(void *param) {
  /* 事件循环 */
  for (;;) {
    connection_t *connection;
    struct timespec ts;
    set_timespec(&ts, threadpool_idle_timeout);      // 空闲超时
    connection = get_event(&this_thread, thread_group, &ts);
    if (!connection) break;                          // 超时 → 线程退出
    this_thread.event_count++;
    handle_event(connection);
  }
  /* 线程退出：清理 */
  mysql_cond_destroy(&this_thread.cond);
  mysql_mutex_lock(&thread_group->mutex);
  add_thread_count(thread_group, -1);
  mysql_mutex_unlock(&thread_group->mutex);
}
```

（注：以上为 Percona 版 1447-1485，MariaDB 版结构相同）

**线程退出机制**：`get_event` 带 `abstime`（= `threadpool_idle_timeout`），超时返回 NULL → worker 退出。这是线程数动态收缩的实现方式。

### 2.7 wait_begin / wait_end

```cpp
void wait_begin(thread_group_t *thread_group) {
  mysql_mutex_lock(&thread_group->mutex);
  thread_group->active_thread_count--;
  DBUG_ASSERT(thread_group->active_thread_count >= 0);
  DBUG_ASSERT(thread_group->connection_count > 0);

  if ((thread_group->active_thread_count == 0) &&
      (!is_queue_empty(thread_group) || !thread_group->listener)) {
    /*
      Group might stall while this thread waits, thus wake
      or create a worker to prevent stall.
    */
    wake_or_create_thread(thread_group);              // ★ 无条件（无编译期开关）
  }
  mysql_mutex_unlock(&thread_group->mutex);
}

void wait_end(thread_group_t *thread_group) {
  mysql_mutex_lock(&thread_group->mutex);
  thread_group->active_thread_count++;
  mysql_mutex_unlock(&thread_group->mutex);
}
```

**关键补充**（来自阿里云源码剖析，源码注释里没有但很重要）：

> **不是所有等待都要调 `wait_begin`**：mutex 这类预期很快结束的等待可以不调；**行锁或磁盘 IO 这种长等待必须调**。

这是"与 InnoDB 交互"的接缝处。

### 2.8 check_stall 与 timer thread

```cpp
static void* timer_thread(void *param) {
  set_timespec_nsec(ts, timer->tick_interval * 1000000LL);
  err = mysql_cond_timedwait(&timer->cond, &timer->mutex, &ts);
  if (err == ETIMEDOUT) {
    timer->current_microtime = microsecond_interval_timer();
    for (i = 0; i < threadpool_max_size; i++) {
      if (all_groups[i].connection_count) check_stall(&all_groups[i]);
    }
    if (timer->next_timeout_check.load(...) <= timer->current_microtime)
      server_threads.iterate(timeout_check, timer);      // 顺带处理 wait_timeout
  }
}
```

```cpp
void check_stall(thread_group_t *thread_group) {
  mysql_mutex_lock(&thread_group->mutex);

  /* ① 优先级提升：低优队列待太久 → 挪到高优 */
  for (;;) {
    c = thread_group->queues[TP_PRIORITY_LOW].front();
    if (c && pool_timer.current_microtime - c->enqueue_time >
             1000ULL * threadpool_prio_kickup_timer) {
      thread_group->queues[TP_PRIORITY_LOW].remove(c);
      thread_group->queues[TP_PRIORITY_HIGH].push_back(c);
    } else break;
  }

  /* ② listener 缺失 → 建线程（此分支优先且短路 return） */
  if (!thread_group->listener && !thread_group->io_event_count) {
    wake_or_create_thread(thread_group, true);
    mysql_mutex_unlock(&thread_group->mutex);
    return;
  }
  thread_group->io_event_count = 0;

  /* ③ 队列排空速率为 0 且队列非空 → stalled */
  if (!is_queue_empty(thread_group) && !thread_group->queue_event_count) {
    thread_group->stalled = true;
    TP_INCREMENT_GROUP_COUNTER(thread_group, stalls);
    wake_or_create_thread(thread_group, true);
  }
  thread_group->queue_event_count = 0;
  mysql_mutex_unlock(&thread_group->mutex);
}
```

**stall 判定的完整语义**（文档 + 源码一致）：
1. group 的 work queue 里还有请求等待分发
2. **自上次检查以来没有任何请求被出队执行**（`queue_event_count == 0`）

**MariaDB 注释里的重要 Q&A**：

> Q: Will this handling lead to an unbound growth of threads, if queue stalls permanently?
> A: No. If queue stalls permanently, it is an indication for many very long simultaneous queries. The maximum number of simultaneous queries is `max_connections`, further we have `threadpool_max_threads` limit... **So in case there is a flood of very long queries, threadpool would slowly approach thread-per-connection behavior.**
>
> NOTE: If long queries never wait, creation of the new threads is done by timer, so it is slower than in real thread-per-connection. However if long queries do wait and indicate that via `thd_wait_begin/end` callbacks, thread creation will be faster.

**即线程池在极端情况下会"退化"为 thread-per-connection，这是设计上可接受的兜底。**

### 2.9 start_io 与 change_group（1498 / 1468）

```cpp
int TP_connection_generic::start_io() {
  /*
    Usually, connection will stay in the same group for the entire
    connection's life. However, we do allow group_count to change at runtime...
    So we recalculate in which group the connection should be, based
    on thread_id and current group count, and migrate if necessary.
  */
  if (fix_group) {                                     // ★ 只有标志置位时才重算
    fix_group = false;
    thread_group_t *new_group = &all_groups[get_group_id(thd->thread_id)];
    if (new_group != thread_group) {
      if (change_group(this, thread_group, new_group)) return -1;
    }
  }
  if (!bound_to_poll_descriptor) {
    bound_to_poll_descriptor = true;
    return io_poll_associate_fd(thread_group->pollfd, fd, this, ...);   // 首次注册
  }
  return io_poll_start_read(thread_group->pollfd, fd, this, ...);       // 重新监听
}
```

```cpp
static int change_group(TP_connection_generic *c, thread_group_t *old_group,
                        thread_group_t *new_group) {
  mysql_mutex_lock(&old_group->mutex);
  if (c->bound_to_poll_descriptor) {
    io_poll_disassociate_fd(old_group->pollfd, c->fd);   // 从旧组 epoll 摘除
    c->bound_to_poll_descriptor = false;
  }
  c->thread_group->connection_count--;
  mysql_mutex_unlock(&old_group->mutex);

  mysql_mutex_lock(&new_group->mutex);
  c->thread_group = new_group;
  new_group->connection_count++;
  if (!new_group->thread_count) ret = create_worker(new_group, false);  // 保证新组有线程
  mysql_mutex_unlock(&new_group->mutex);
}
```

**这是 MariaDB 的"负载均衡"雏形**：只在 `thread_pool_size` 运行时改变时触发迁移，不做基于负载的主动均衡。对比 TXSQL 的主动负载均衡（见第 8 章）。

### 2.10 唤醒与节流

```cpp
static int wake_thread(thread_group_t *thread_group, bool due_to_stall) {
  worker_thread_t *thread = thread_group->waiting_threads.front();
  if (thread) {
    thread->woken = true;
    thread_group->waiting_threads.remove(thread);
    mysql_cond_signal(&thread->cond);
    TP_INCREMENT_GROUP_COUNTER(thread_group, wakes);
    if (due_to_stall) TP_INCREMENT_GROUP_COUNTER(thread_group, wakes_due_to_stall);
    return 0;
  }
  return 1;   /* no thread in waiter list => missed wakeup */
}
```

`wake_or_create_thread`逻辑：先 `wake_thread()` → 若 `active_thread_count == 0` 立即 `create_worker()` → 否则受 `time_since_last_thread_created > microsecond_throttling_interval(group)` 节流。

**节流表**（MariaDB 10.5+ 官方文档）：

| group 内线程数 | 节流间隔 |
|---|---|
| 0 ~ (1 + oversubscribe) | 0（不节流） |
| 4-7 | 50 × `THROTTLING_FACTOR` ms |
| 8-15 | 100 × `THROTTLING_FACTOR` ms |
| 16-65536 | 20 × `THROTTLING_FACTOR` ms |

`THROTTLING_FACTOR = thread_pool_stall_limit / MAX(500, thread_pool_stall_limit)`

> ⚠️ 文档表格有跳跃（第二档直接写 4-7，未覆盖 2-3）。中文文章引用的 5.5 源码是固定值 50/100/200 ms。**按目标版本源码为准**。

### 2.11 io_poll 多平台（178 / 254 / 310 / 358）

`threadpool_generic.cc` 里四套实现并列：

| 行号 | 实现 | 平台 |
|---|---|---|
| 178 | generic（select/poll） | 兜底 |
| **254** | **epoll** | Linux |
| 310 | kevent | FreeBSD / macOS |
| 358 | Windows（IOCP） | Windows |

`threadpool_generic.h` 用 typedef 统一抽象：

```cpp
#ifdef __linux__
#include <sys/epoll.h>
typedef struct epoll_event native_event;
#elif defined(HAVE_KQUEUE)
typedef struct kevent native_event;
#elif defined(__sun)
typedef port_event_t native_event;      // Solaris event ports
#elif defined(HAVE_IOCP)
typedef OVERLAPPED_ENTRY native_event;
#else
#error threadpool is not available on this platform
#endif
```

对照 MySQL 只有 `epoll.cc` + `poll.cc`（MariaDB 文档原话：Oracle 仅 Linux 用 epoll，其余 `poll()`）。

### 2.12 shutdown 的 self-pipe

```cpp
/*
   Wake listener thread (during shutdown)
   Self-pipe trick is used in most cases, except IOCP.
*/
static int wake_listener(thread_group_t *thread_group) {
  if (pipe(thread_group->shutdown_pipe)) return -1;
  if (io_poll_associate_fd(thread_group->pollfd, thread_group->shutdown_pipe[0], NULL, NULL))
    return -1;
  char c = 0;
  if (write(thread_group->shutdown_pipe[1], &c, 1) < 0) return -1;
}
```

MySQL 的 `epoll.cc` 用 socketpair 做同样的事（`tp_group_low_level_waiter_wake`）。

---

<a id="c3"></a>
## 3. Percona 源码剖析

### 3.1 结构差异（threadpool_unix.cc）

```cpp
struct connection_t {
  THD *thd;
  thread_group_t *thread_group;
  connection_t *next_in_queue;
  connection_t **prev_in_queue;
  ulonglong abs_wait_timeout;
  bool logged_in;
  bool bound_to_poll_descriptor;
  bool waiting;
  uint tickets;                      // ★ Percona 独有：高优先级票
};

struct alignas(128) thread_group_t {
  mysql_mutex_t mutex;
  connection_queue_t queue;
  connection_queue_t high_prio_queue;   // ★ 显式命名，不像 MariaDB 用数组
  worker_list_t waiting_threads;
  worker_thread_t *listener;
  pthread_attr_t *pthread_attr;
  int pollfd;
  int thread_count;
  int active_thread_count;
  int connection_count;
  int waiting_thread_count;             // ★ Percona 独有：等待中的线程数
  /* Stats for the deadlock detection timer routine.*/
  int io_event_count;
  int queue_event_count;
  ulonglong last_thread_creation_time;
  int shutdown_pipe[2];
  bool shutdown;
  bool stalled;
  char padding[328];                    // ★ 强制补齐
};

static_assert(sizeof(thread_group_t) == 512,
              "sizeof(thread_group_t) must be 512 to avoid false sharing");
```

**与 MariaDB 的关键差异**：

| | MariaDB | Percona |
|---|---|---|
| 队列表示 | `queues[NQUEUES]` 数组 | `queue` + `high_prio_queue` 显式命名 |
| **等待线程计数** | **无** | **`waiting_thread_count`** ★ |
| **tickets** | 无 | **`tickets`**（在 connection_t 里）★ |
| false sharing 防护 | `char pad[CPU_LEVEL1_DCACHE_LINESIZE]` | **`alignas(128)` + `padding[328]` + `static_assert(sizeof == 512)`** |
| counters | `thread_group_counters_t`（8 个计数器） | 无专门 counters 结构 |

**`static_assert(sizeof(thread_group_t) == 512)` 是很硬核的写法** —— 直接强制结构体占 512 字节（8 个 cache line），彻底避免组间 false sharing。

#### worker_thread_t：`woken` 标志防虚假唤醒

Percona 的每线程结构（MariaDB 的同名结构没有 `woken`）：

```cpp
/* Per-thread structure for workers */
struct worker_thread_t {
  ulonglong event_count;        /* number of request handled by this thread */
  thread_group_t *thread_group;
  worker_thread_t *next_in_list;
  worker_thread_t **prev_in_list;

  mysql_cond_t cond;
  bool woken;                   /* ★ 防虚假唤醒 */
};
```

`woken` 的用途：worker 线程睡在自己的 `cond` 上，被唤醒时**不能确定是真有任务还是虚假唤醒**（pthread_cond_wait 允许虚假返回）。Percona 用 `woken` 区分——唤醒者置 `woken=true`，被唤醒者检查它；若为 false 则继续睡。

**这与 MariaDB 的做法不同**：MariaDB 的 `worker_main` 靠"睡前再做一次非阻塞轮询 + 出队即清 stalled"来兜底，Percona 则用显式的 `woken` 标志。两者都能防虚假唤醒，但 Percona 的更直白。

`event_count` 记录该线程处理过的请求数，用于判断线程"是否活跃"（与 cache 亲和、线程退休决策相关）。

#### 队列的容器：侵入式 `I_P_List`

```cpp
typedef I_P_List<connection_t,
                 I_P_List_adapter<connection_t, &connection_t::next_in_queue,
                                  &connection_t::prev_in_queue>,
                 I_P_List_null_counter, I_P_List_fast_push_back<connection_t>>
    connection_queue_t;
```

与 MariaDB 一样用 **侵入式双向链表**（`I_P_List`），靠 adapter 把 `next_in_queue`/`prev_in_queue` 两对指针复用给多个队列（低优队列、高优队列各一个链表实例，但共用同一对指针字段——所以一个 `connection_t` 同一时刻只能在一条队列里）。

`I_P_List_fast_push_back` 是 O(1) 尾部插入，配合 `I_P_List_null_counter`（不维护计数，计数由 `connection_count` 单独维护）。

### 3.2 双重过载判定（368 / 381）

```cpp
inline bool too_many_active_threads(const thread_group_t &thread_group) noexcept {
  return (thread_group.active_thread_count >= 1 + (int)threadpool_oversubscribe &&
          !thread_group.stalled);
}

/*
  Limit the number of 'busy' threads by 1 + thread_pool_oversubscribe. A thread
  is busy if it is in either the active state or the waiting state (i.e. between
  thd_wait_begin() / thd_wait_end() calls).
*/
inline bool too_many_busy_threads(const thread_group_t &thread_group) noexcept {
  return (thread_group.active_thread_count + thread_group.waiting_thread_count >
          1 + (int)threadpool_oversubscribe);
}
```

**`too_many_busy_threads` 是 Percona 的核心创新**：把"活跃的 + 等待中的"一起计数。因为等待中的线程虽不占 CPU，但**仍持有事务和锁**，会加剧争用。

对比 MariaDB 只有 `too_many_threads`：`active > 1 + oversubscribe && !stalled`。

**语义差异**：MariaDB 用 `>`，Percona 用 `>=`（`too_many_active_threads`）—— 边界差 1。

### 3.3 queue_get：低优先级队列节流

```cpp
static connection_t *queue_get(thread_group_t *thread_group) noexcept {
  thread_group->queue_event_count++;
  connection_t *c;

  if ((c = thread_group->high_prio_queue.front())) {
    thread_group->high_prio_queue.remove(c);          // 高优队列永不被节流
  }
  /*
    Don't pick events from the low priority queue if there are too many
    active + waiting threads.
  */
  else if (!too_many_busy_threads(*thread_group) &&
           (c = thread_group->queue.front())) {
    thread_group->queue.remove(c);
  }
  return c;
}
```

**这是 Percona 防死锁的核心机制**：组繁忙时停止从低优队列取任务。

### 3.4 get_event 的 tickets 处理

Percona 在"睡前非阻塞轮询"这一步加入了 MariaDB 没有的复杂逻辑：

```cpp
if (!oversubscribed) {
  native_event nev;
  if (io_poll_wait(thread_group->pollfd, &nev, 1, 0) == 1) {
    thread_group->io_event_count++;
    connection = (connection_t *)native_event_get_userdata(&nev);

    /*
      Since we are going to perform an out-of-order event processing for the
      connection, first check whether it is eligible for high priority
      processing. We can get here even if there are queued events, so it
      must either have a high priority ticket, or there must be not too many
      busy threads (as if it was coming from a low priority queue).
    */
    if (connection_is_high_prio(*connection))
      connection->tickets--;                            // 高优：扣票
    else if (too_many_busy_threads(*thread_group)) {
      /*
        Not eligible for high priority processing. Restore tickets and put
        it into the low priority queue.
      */
      connection->tickets =
          connection->thd->variables.threadpool_high_prio_tickets;   // 恢复票
      thread_group->queue.push_back(connection);                     // 放回低优队列
      connection = nullptr;
    }

    if (connection) {
      thread_group->queue_event_count++;
      break;
    }
  }
}
```

**这段解决的问题**：worker 绕过队列直接从 epoll 拿事件，属于"插队"（out-of-order）。Percona 用 tickets 机制确保这种插队也受优先级配额约束——不是高优且组繁忙时，**把事件退回低优队列**。

MariaDB 的对应代码简单得多：直接 `queue_put` 再 `queue_get`，无 tickets 概念。

**另一个细节**：MariaDB 轮询用 `MAX_EVENTS` 个事件并全部入队，Percona 只取 **1 个**（`&nev, 1`）。

### 3.5 wait_begin 的编译期开关—— 重大发现

```cpp
static void wait_begin(thread_group_t *thread_group) noexcept {
  mysql_mutex_lock(&thread_group->mutex);
  thread_group->active_thread_count--;
  thread_group->waiting_thread_count++;                  // ★ 维护 waiting 计数

#ifdef THREADPOOL_CREATE_THREADS_ON_WAIT
  if ((thread_group->active_thread_count == 0) &&
      (!queues_are_empty(*thread_group) || !thread_group->listener)) {
    /*
      Group might stall while this thread waits, thus wake
      or create a worker to prevent stall.
    */
    wake_or_create_thread(thread_group);
  }
#endif
  mysql_mutex_unlock(&thread_group->mutex);
}
```

**与 MariaDB 的关键差异**：

| | MariaDB | Percona |
|---|---|---|
| wait 时建线程 | **无条件** | **被 `#ifdef THREADPOOL_CREATE_THREADS_ON_WAIT` 包裹** ★ |
| waiting 计数 | 无 | `waiting_thread_count++` |

如果 `THREADPOOL_CREATE_THREADS_ON_WAIT` 未在编译时定义，Percona 在引擎等待时**不会主动创建线程**，只靠 timer thread 的 stall 检测（500ms 周期）兜底 —— 这会显著劣化长等待场景的响应。

> 这是个值得在实际部署中确认的编译选项（不同发行版/构建可能有差异）。

### 3.6 start_io

```cpp
static int start_io(connection_t *connection) {
  thread_group_t *const group =
      &all_groups[connection->thd->thread_id() % group_count];

  if (group != connection->thread_group) {
    if (change_group(connection, connection->thread_group, group)) return -1;
  }

  Vio *vio = connection->thd->get_protocol_classic()->get_vio();
  int fd = mysql_socket_getfd(vio->mysql_socket);
  if (!connection->bound_to_poll_descriptor) {
    connection->bound_to_poll_descriptor = true;
    return io_poll_associate_fd(group->pollfd, fd, connection);
  }
  return io_poll_start_read(group->pollfd, fd, connection);
}
```

**与 MariaDB 的差异**：MariaDB 用 `fix_group` 标志（只在需要时重算），Percona **每次 `start_io` 都重算** `thread_id() % group_count`。

代价：每次请求后多一次取模 + 比较。收益：pool size 变化时立刻生效，无需额外标志。

### 3.7 handle_event 与 worker_main

```cpp
static void handle_event(connection_t *connection) {
  int err;
  if (!connection->logged_in) {
    err = threadpool_add_connection(connection->thd);     // 认证
    connection->logged_in = true;
  } else {
    err = threadpool_process_request(connection->thd);     // 执行
  }
  if (err) goto end;
  set_wait_timeout(connection);
  err = start_io(connection);                              // 重新监听
end:
  if (err) connection_abort(connection);
}

static void *worker_main(void *param) {
  thread_group_t *thread_group = (thread_group_t *)param;
  worker_thread_t this_thread;                             // ★ 栈上分配，非堆
  mysql_cond_init(key_worker_cond, &this_thread.cond);
  this_thread.thread_group = thread_group;
  this_thread.event_count = 0;

  for (;;) {
    connection_t *connection;
    struct timespec ts;
    set_timespec(&ts, threadpool_idle_timeout);
    connection = get_event(&this_thread, thread_group, &ts);
    if (!connection) break;                                // 超时退出
    this_thread.event_count++;
    handle_event(connection);
  }
  mysql_cond_destroy(&this_thread.cond);
  mysql_mutex_lock(&thread_group->mutex);
  add_thread_count(thread_group, -1);
  mysql_mutex_unlock(&thread_group->mutex);
}
```

**注意**：worker 线程的 `worker_thread_t` 是**栈上分配**（`worker_thread_t this_thread;`），线程退出即销毁。这解释了为什么 `waiting_threads` 链表里存的是指向各线程栈的指针——线程活着时指针有效。

### 3.8 tp_init / tp_end / tp_set_threadpool_size

```cpp
bool tp_init() {
  DBUG_ENTER("tp_init");
  threadpool_started = true;

  for (uint i = 0; i < array_elements(all_groups); i++) {
    thread_group_init(&all_groups[i], get_connection_attrib());
  }
  tp_set_threadpool_size(threadpool_size);
  if (group_count == 0) {
    /* Something went wrong */
    sql_print_error("Can't set threadpool size to %d", threadpool_size);
    DBUG_RETURN(true);
  }
#ifdef HAVE_PSI_INTERFACE
  mysql_mutex_register("threadpool", mutex_list, array_elements(mutex_list));
  mysql_cond_register("threadpool", cond_list, array_elements(cond_list));
  mysql_thread_register("threadpool", thread_list, array_elements(thread_list));
#endif
  DBUG_RETURN(false);
}
```

三个要点：

1. **`all_groups` 是预分配的静态数组**——`tp_init` 先初始化**全部**组（`array_elements(all_groups)`），但真正启用的只有 `threadpool_size` 个。这是"组数可变"的前提。
2. **`tp_set_threadpool_size()` 负责按需创建 pollfd**：

```cpp
/** Ensure that poll descriptors are created when threadpool_size changes */
void tp_set_threadpool_size(uint size) noexcept {
  if (!threadpool_started) return;

  bool success = true;
  for (uint i = 0; i < size; i++) {
    thread_group_t *group = &all_groups[i];
    mysql_mutex_lock(&group->mutex);
    if (group->pollfd == -1) {
      group->pollfd = io_poll_create();
      success = (group->pollfd >= 0);
      if (!success) {
        sql_print_error("io_poll_create() failed, errno=%d\n", errno);
      }
    }
    mysql_mutex_unlock(&all_groups[i].mutex);
    if (!success) {
      group_count = i;
      return;
    }
  }
  group_count = size;
}
```

`if (group->pollfd == -1)` 这个判断让**扩大组数时只为新增的组创建 pollfd**，已有的不动——这正是 `change_group`（组数变化时连接迁移，见 2.9 节）能实现的基础。失败时 `group_count = i` 表示"只成功启用了 i 个组"，属于降级而非崩溃。

3. **PFS 注册三类对象**（mutex / cond / thread），与 MySQL 26.7 插件的 `init_tp_psi_keys()` 是同一件事，但 Percona 是内建直接注册，MySQL 要作为插件动态注册。

关闭路径：

```cpp
void tp_end() noexcept {
  DBUG_ENTER("tp_end");
  if (!threadpool_started) DBUG_VOID_RETURN;
  if (timer_thread_started) stop_timer(&pool_timer);
  for (uint i = 0; i < array_elements(all_groups); i++) {
    thread_group_close(&all_groups[i]);
  }
  threadpool_started = false;
  DBUG_VOID_RETURN;
}
```

**先停 timer 线程，再关闭各线程组**——顺序敏感：timer 负责 stall 检测与死锁检测，它会访问各组状态；若先关组再停 timer，timer 会访问到已关闭的组。这与 MySQL 26.7 `thd_pool_end()` 的"先停 stall check 线程、再关各线程组"是**同一个顺序约定**（见 [`thread_pool.md`](thread_pool.md) 的 8.1）。

> 组结构里还有 `int shutdown_pipe[2]`——每组的关闭通知管道，用于把阻塞在 `io_poll_wait` 上的线程唤醒（与 MySQL 的 socketpair self-pipe 同工，见 [`thread_pool.md`](thread_pool.md) 的 4.1）。

### 3.9 io_poll 层：★ 边缘触发 vs 水平触发

Percona 的事件注册（Linux）：

```cpp
static int io_poll_start_read(int pollfd, int fd, void *data) noexcept {
  struct epoll_event ev;
  ev.data.u64 = 0; /* Keep valgrind happy */
  ev.data.ptr = data;
  ev.events = EPOLLIN | EPOLLET | EPOLLERR | EPOLLRDHUP | EPOLLONESHOT;
  return epoll_ctl(pollfd, EPOLL_CTL_MOD, fd, &ev);
}
```

**★ 与 MySQL 26.7 的关键差异**：

| | Percona | MySQL 26.7 |
|---|---|---|
| 触发模式 | **`EPOLLET`（边缘触发）** | 水平触发（未设 `EPOLLET`） |
| oneshot | `EPOLLONESHOT` | `EPOLLONESHOT` |
| 附加事件 | `EPOLLERR` / `EPOLLRDHUP` | 仅 `EPOLLIN` |
| 超时用法 | 注释明说"**只用 infinite 或 0**" | 见 [`thread_pool.md`](thread_pool.md) |

边缘触发意味着：**状态变化只通知一次**，读事件到来后必须把数据读完，否则剩余数据不会再次触发。这对"连接亲和连读"的设计有直接影响——Percona 需要确保一次事件把数据读干净；MySQL 用水平触发则宽容一些（没读完会再次触发）。

`epoll_wait` 的包装也值得一读：

```cpp
/*
 Wrapper around epoll_wait.
 NOTE - in case of EINTR, it restarts with original timeout. Since we use
 either infinite or 0 timeouts, this is not critical
*/
static int io_poll_wait(int pollfd, native_event *native_events, int maxevents,
                        int timeout_ms) noexcept {
  int ret;
  do {
    ret = epoll_wait(pollfd, native_events, maxevents, timeout_ms);
  } while (ret == -1 && errno == EINTR);
  return ret;
}
```

`do...while (EINTR)` 重启是标准做法；注释解释为什么重启时用原 timeout 无所谓——因为**实际只用 0（非阻塞轮询）或 -1（无限等待）**，不存在"超时被重置导致等待过久"的问题。

多平台封装（与 MariaDB 一致，都按 OS 取最优）：

| 平台 | 机制 | oneshot 对应 |
|---|---|---|
| Linux | `epoll` | `EPOLLONESHOT` |
| FreeBSD / macOS | `kqueue` | `EV_ADD \| EV_ONESHOT` |
| Solaris | event ports | — |
| Windows | IOCP | — |

> 对比 MySQL 26.7：只有 Linux 用 `epoll`，其他平台退化为 `poll()`（`epoll.cc` + `poll.cc` 两个实现并存）。这是 MariaDB/Percona 相对 Oracle 实现的一个明确优势。

---

<a id="c4"></a>
## 4. 逐函数源码对照

### get_event（最核心的对照）

| 步骤 | MariaDB | Percona |
|---|---|---|
| 过载判定 | `too_many_threads()` = `active > 1+oversub && !stalled` | `too_many_active_threads()` = `active >= 1+oversub && !stalled` |
| ① 从队列取 | `queue_get(tg, WORKER)` | `queue_get(tg)`（含 busy 节流） |
| ② 无 listener 则变 listener | ✅ 相同 | ✅ 相同 |
| ③ 睡前非阻塞轮询 | `MAX_EVENTS` 个，全部入队 | **1 个**，含 **tickets 判定**，不合规则退回低优队列 |
| ④ 休眠 | `push_front` LIFO + `woken` 标志 | **完全相同**（注释一字不差） |
| ⑤ 退出 | `stalled = false` | `stalled = false`（相同） |

### wait_begin

| | MariaDB | Percona |
|---|---|---|
| `active_thread_count--` | ✅ | ✅ |
| `waiting_thread_count++` | ❌ | ✅ |
| wait 时建线程 | **无条件** | **`#ifdef THREADPOOL_CREATE_THREADS_ON_WAIT`** |

### start_io

| | MariaDB | Percona |
|---|---|---|
| 组重算时机 | `fix_group` 标志置位时 | **每次都算** |
| fd 获取 | `c->fd` | 从 `vio->mysql_socket` 取 |

### check_stall

| | MariaDB | Percona |
|---|---|---|
| 加锁 | `mysql_mutex_lock`（阻塞） | **`mysql_mutex_trylock`**（拿不到就走） |
| prio kickup | ✅ 在 check_stall 里 | ❌（无 kickup 机制，用 tickets 代替） |
| listener 缺失分支 | ✅ 短路 return | ✅ 相同 |
| 队列排空判定 | `!is_queue_empty() && !queue_event_count` | `!queue_event_count && !queues_are_empty()` |
| `queues_are_empty` 定义 | `is_queue_empty()` | **`high_prio_queue.is_empty() && (queue.is_empty() \|\| too_many_busy_threads())`** ★ |

**`queues_are_empty` 的差异很关键**：Percona 中 `too_many_busy_threads` 会让低优队列被视为"空"，从而**不触发 stall**。这是有意设计——组已经很忙时不再判定 stall，避免雪上加霜。

### 数据结构

| | MariaDB | Percona | MySQL 26.7 |
|---|---|---|---|
| 队列 | `queues[2]` 数组 | `queue` + `high_prio_queue` | `queued_queries`(高) + `queued_trans`(低) |
| 等待计数 | ❌ | `waiting_thread_count` | `threads_for_consumer/reserve` |
| tickets | ❌ | `connection_t::tickets` | ❌（用 kickup timer） |
| false sharing | `pad[CPU_LEVEL1_DCACHE_LINESIZE]` | `alignas(128)` + `static_assert(512)` | 手工排布热冷字段 + `alignas(128)` |
| 统计 | `thread_group_counters_t`（8 项，区分 stall） | 无 | `tp_group_statistics_t`（含 10 类等待计数） |

---

<a id="c5"></a>
## 5. 核心机制三方对比

### 5.1 stall 检测：三种哲学

| | **MySQL 26.7** | **MariaDB 12.3** | **Percona 8.0** |
|---|---|---|---|
| 度量对象 | **单个查询的执行时长** | **组的队列排空速率** | 同 MariaDB |
| 判据 | `now - time_of_attach > 60ms` | `queue_event_count == 0` && 队列非空 | 同左（`queues_are_empty` 受 busy 影响） |
| 语义 | "这个查询跑太久了" | "这个组停止推进了" | 同左 |
| 检查者 | 全局 1 个 stall check 线程 | 全局 1 个 timer thread | 同左 |
| **检查周期** | **1ms/10ms 自适应**（与判定阈值解耦） | **`stall_limit` 本身**（500ms） | 同左 |
| 参数单位 | **10ms**（6 = 60ms） | **1ms**（500 = 500ms） | **1ms** |
| 后果 | **stalled 线程获额外名额**（不计入配额） | 置 `stalled` 标志 + 唤醒/建线程 | 同左 |
| 锁策略 | `mysql_mutex_lock` | `mysql_mutex_lock` | **`mysql_mutex_trylock`** |

**为什么 MySQL 更精细**：MariaDB/Percona 只知道"没人出队"，不知道是哪个查询卡住；MySQL 的 per-thread 计时能定位到具体线程并做**差异化处理**（给 stalled 线程独立名额）。

**代价**：MySQL 需维护 `time_of_attach` 并在 stall 检查时遍历所有线程。

### 5.2 并发控制的度量单位

| | MySQL 26.7 | MariaDB | Percona |
|---|---|---|---|
| 主要度量 | **并发事务数（MTL）** | 活跃线程数 | 活跃+等待线程数 |
| 长事务背压 | **✅（滞回）** | ❌ | ❌ |
| 硬上限 | 3×vCPU + 4096/组 | max_threads 65536 | max_threads 100000 |

**演进方向**：线程数 → 事务数。Percona 的 `too_many_busy_threads`（活跃+等待）是从"线程数"向"事务数"过渡的中间形态；MySQL 的 MTL 是终态。

### 5.3 优先级与防饿死

| | MySQL 26.7 | MariaDB 12.3 | Percona 8.0 |
|---|---|---|---|
| 进高优条件 | 事务活跃 / kickup=0 / 高优连接 | 事务中（`priority=auto`） | **事务中 + 持表锁 + 持 MDL + 持全局读锁 + 持 backup 锁 + USER_LEVEL_LOCK + LOCKING_SERVICE**（且 `tickets > 0`） |
| 防饿死 | **`prio_kickup_timer`**（每 10ms 搬 1 条） | **`prio_kickup_timer`** | **`high_prio_tickets`** |
| 锁感知 | ❌ | ❌ | **✅** |
| 管理连接 | **admin thread group** | — | `is_admin_connection()` → 高优 |

**两种防饿死思路**：
- **kickup timer**（MySQL/MariaDB）：低优等待超时后**升级**
- **tickets**（Percona）：限制**每个连接能插队多少次**，更公平

### 5.4 应急通道

| | MySQL 26.7 | MariaDB 12.3 | Percona 8.0 |
|---|---|---|---|
| 机制 | **admin thread group**（池内第 513 组） | **`extra_port`**（池外独立端口，one-thread-per-connection） | `admin_port`（8.0.14+，取代 extra_port） |
| 权限 | `TP_CONNECTION_ADMIN`（默认不授予任何人） | 端口本身即后门 | — |

**三种完全不同的路线**：一个在池内保留专属资源，一个在池外开后门。

> MariaDB 文档的必要性论证："**即使把 `thread_pool_max_threads` 设得很大，全局锁场景仍可能堵死整个池**"（`FLUSH TABLES WITH READ LOCK` 后客户端挂起，写连接撑满 max_threads 导致连 `UNLOCK TABLES` 都发不出）。

---

<a id="c6"></a>
## 6. oversubscribe 语义辨析（重要，普遍被误读）

**交叉验证中最值得澄清的一点。**

| 来源 | 语义 |
|---|---|
| **MariaDB 官方文档** | **"does not play any part in the creation of new worker threads"**。定义 oversubscribed 后**保持 active 的 worker 上限** |
| **MariaDB 源码** | `too_many_threads()` = `active > 1 + oversub && !stalled`，作用于 **`get_event` 是否从队列取事件** |
| **Percona 源码** | `too_many_busy_threads()` = `active + waiting > 1 + oversub`，作用于 **① `queue_get` 是否取低优队列 ② 普通队列节流** |
| **内核月报 2016/02（Percona 口径）** | "活跃+等待线程数超过该值就不再创建新线程"，用于防死锁 |
| **腾讯 TXSQL / dbkernel** | 沿用 Percona 口径 |

**结论**：不是文档错误，是**两个分支语义确实不同**：

- **MariaDB**：oversubscribe 是**稳态收敛上限**（stall 后允许超额，之后收敛回该值）
- **Percona**：oversubscribe 是**准入闸门**（active+waiting 超限就停止从低优队列取任务并节流）

做三方对比时**必须区分**，否则会得出错误结论。

---

<a id="c7"></a>
## 6.5 stall 检测三家源码级辨析

> stall（停滞）检测是线程池防止"一个长查询拖死整组"的兜底机制，但三家对"什么算 stall"的定义完全不同。本节像 6 章（oversubscribe）一样做源码级辨析。

#### 先理解：stall 到底是什么，为什么非有不可

**一句话**：stall 是线程池的"卡死检测"——发现某个正在执行的线程卡住了，从而防止它独占线程组、让同组其他连接全部干等。

**它要解决的核心矛盾**，来自线程池自身的设计：

```
线程池的基本假设：每个线程组同时只让很少的线程执行（MySQL 说最多 3 个）
   ↓ 收益：CPU cache 局部性好（这是线程池的性能来源）
   ↓ 代价：正在执行的那个线程一旦卡住，整组就停摆
              ——队列里的连接都在等"有线程空出来"，而唯一的线程卡着
```

也就是说，**线程池为了性能而限制并发，这个限制本身会制造一个新风险：一个卡住的执行体可以饿死整组**。stall 检测就是打破这个僵局的兜底开关：**判定"卡住了" → 放行其他线程**。

MySQL 官方文档把会触发这个风险的场景列为三类：

| 场景 | 说明 |
|---|---|
| **长查询** | 执行时间长（数据仓库类、未优化 SQL） |
| **binlog dump 线程** | 一种"永远在运行"的语句，本质上是长查询的极端形态 |
| **未插桩代码的阻塞** | 等行锁/表锁/sleep，但**引擎没有向线程池上报**——线程池无从得知，只能靠超时猜 |

第三类是关键：MySQL 优先靠引擎回调（`wait_begin` / `wait_end`）**精确**知道阻塞；**只有未上报的阻塞才需要 stall 超时来兜底**。所以 stall 是**兜底机制**，不是主力机制。

**三个常见误解**：

1. **stall ≠ 慢查询**。stall 是一个**调度决策**（"该不该放行别人了"），不是性能诊断（"这个查询慢不慢"）。
2. **stall 不是错误**。它是线程池正常工作的一部分——高并发下几乎必然会有查询被判 stalled。
3. **单位容易搞错**：MySQL 的 `thread_pool_stall_limit` 单位是 **10ms**（默认 6 = 60ms），MySQL/Percona 是 **ms**（默认 500）。差一个数量级还差一个单位。

**一个比喻**——把线程组想成只有一个收银台的超市：

| | MySQL 26.7 | MariaDB / Percona |
|---|---|---|
| 怎么发现堵了 | 看表：**这个顾客结账超过 60ms 还没完** | 每 500ms 看一眼：**队伍有没有往前挪** |
| 判定的对象 | 那个顾客（线程） | 整个收银台（组） |
| 发现之后 | 开新收银台，且**不把掏钱包的顾客算作"忙碌收银员"**（所以开新台不受他影响） | 开新收银台（允许突破 oversubscribe） |

MySQL 问的是"**谁**卡了"，两家问的是"**堵了没有**"——都能解决停摆，但前者因为知道是谁，才能做出"给它开额外名额"这种精细化处理。这是两种 stall 定义的根本分野。

### MariaDB：`check_stall` 是"多合一"，且用阻塞锁

```cpp
void check_stall(thread_group_t *thread_group)
{
  mysql_mutex_lock(&thread_group->mutex);          // ★ 阻塞式加锁

  /*
   Bump priority for the low priority connections that spent too much
   time in low prio queue.
  */
  TP_connection_generic *c;
  for (;;)
  {
    c= thread_group->queues[TP_PRIORITY_LOW].front();
    if (c && pool_timer.current_microtime - c->enqueue_time > 1000ULL * threadpool_prio_kickup_timer)
    {
      thread_group->queues[TP_PRIORITY_LOW].remove(c);
      thread_group->queues[TP_PRIORITY_HIGH].push_back(c);
    }
    else
      break;
  }

  /*
    Check if listener is present. If not, ... Create a new worker
    (it will make itself listener).
  */
  if (!thread_group->listener && !thread_group->io_event_count)
  {
    wake_or_create_thread(thread_group, true);
    mysql_mutex_unlock(&thread_group->mutex);
    return;
  }

  /*  Reset io event count */
  thread_group->io_event_count= 0;
  ...
```

三个要点：

1. **★ 优先级踢升内嵌在 `check_stall` 里**——每次 tick（500ms）批量把低优队列**队首**超时的连接移到高优队列（`for(;;)` 循环直到队首不再超时）。也就是说 **MariaDB 的 `prio_kickup_timer` 是靠 timer 线程驱动的**，不是入队时判定。
2. **listener 存活检查**：`!listener && !io_event_count` ⇒ 没人监听且没有 IO 事件 ⇒ 说明 listener 要么在死循环要么 `thd_wait_begin()` 被忘了调 ⇒ 建新 worker 让它当 listener。
3. **阻塞式 `mysql_mutex_lock`**——timer 线程愿意等 worker 释放组锁。

### Percona：`check_stall` 用 trylock，"别打扰"

```cpp
static void check_stall(thread_group_t *thread_group) {
  if (mysql_mutex_trylock(&thread_group->mutex) != 0) {
    /* Something happens. Don't disturb */
    return;
  }
  ...
```

**★ 拿不到锁直接放弃**——timer 不等。代价是**这一轮跳过该组的检查**（下个 tick 再试）；收益是 **timer 线程永不被 worker 阻塞**（timer 是全池单点，被阻塞会拖延所有组的检查）。

注意 Percona 的 `check_stall` 里**没有**优先级踢升——它用 tickets 机制（连接自带票据）在入队时决定优先级，不需要 timer 批量搬运。

stall 判定的完整逻辑（源码注释原话，四步）：

```
1. thread_group->queue_event_count 记录"从队列移除的事件数"，
   timer 每次运行重置为 0。
2. timer 判定 stall：该计数仍为 0 且 高/低优队列至少一个非空。
3. 判定 stall 后置 stalled 标志，唤醒空闲 worker
   （或创建新的，受节流限制）。
4. stalled 标志在事件出队时重置。
```

**这就是"队列排空速率"判定**：不看任何单个查询跑了多久，只看"这个组这 500ms 里有没有从队列取走过任务"。对照 MariaDB 同样的注释结构（两份代码几乎逐字相同，见第 1 章血缘实证）。

源码注释里还自带一个 Q&A，回答了"队列永久停滞会不会导致线程无限增长"：

> A: No. If queues stall permanently, it is an indication for many very long simultaneous queries. The maximum number of simultaneous queries is `max_connections`, further we have `threadpool_max_threads` limit.

### ★ `queues_are_empty`：Percona 会"假装低优队列是空的"

```cpp
/*
  Check if both the high and low priority queues are empty.

  NOTE: we also consider the low priority queue empty in case it has events, but
  they cannot be processed due to the too_many_busy_threads() limit.
*/
static bool queues_are_empty(const thread_group_t &tg) noexcept {
  return (tg.high_prio_queue.is_empty() &&
          (tg.queue.is_empty() || too_many_busy_threads(tg)));
}
```

**低优队列里有事件、但被 `too_many_busy_threads` 限制不能处理 ⇒ 视为"空"**。后果：`queues_are_empty` 为真 ⇒ 不触发 stall ⇒ 不新建线程。

这是**主动的过载拒绝**（见 1.5 设计哲学）：池饱和时，Percona 宁可让低优查询等着，也不为它们扩容。这与 MariaDB"stall 就唤醒/建线程"形成直接对比——**同样的 stall 检测框架，Percona 在"该不该为新负载扩容"上做了收紧**。

### timer 主循环：两家的守卫与遍历范围

```cpp
// MariaDB：遍历到 threadpool_max_size（所有可能的组，不只是当前启用的）
for (i = 0; i < threadpool_max_size; i++) {
  if (all_groups[i].connection_count)
    check_stall(&all_groups[i]);
}

// Percona：遍历到数组上限 + 同样的 connection_count 守卫
for (size_t i = 0; i < array_elements(all_groups); i++) {
  if (all_groups[i].connection_count) check_stall(&all_groups[i]);
}
```

**为什么遍历"所有可能的组"而不是 `group_count`**：因为组数可以动态变化（`tp_set_threadpool_size` / `change_group`，见 3.8），timer 必须覆盖未来可能启用的组。`connection_count` 守卫跳过空组，避免无谓加锁。

两者还都顺带做 `wait_timeout` 检查（清理超过 `wait_timeout` 的空闲连接）。

### 三家对比表

| 维度 | MySQL 26.7 | MariaDB | Percona |
|---|---|---|---|
| **stall 的定义** | **线程级**：单线程执行超过 `stall_limit`（60ms） | **组级**：队列排空速率（`queue_event_count` 为 0 且队列非空） | 同 MariaDB |
| 加锁方式 | 锁各组 `LOCK_group`（顺序遍历） | **阻塞 `lock`** | ★ **`trylock`，失败跳过** |
| 判定阈值与检查周期 | ★ **解耦**（60ms 判定；1ms/10ms 检查周期自适应） | **耦合**（tick = `stall_limit` = 500ms） | 耦合（同 MariaDB） |
| timer 数量 | 全局 1 个 stall check 线程 | 全局 1 个 timer 线程 | 全局 1 个 timer 线程 |
| stalled 之后 | ★ **stalled 线程不计入活跃配额**（长查询获得额外名额） | 置 `stalled`，允许突破 oversubscribe 唤醒/建线程 | 同 MariaDB，**但过载时低优队列被视为空 ⇒ 不扩容** |
| 优先级踢升位置 | `prio_kickup`（每组每 10ms 最多移 1 条） | ★ **内嵌在 `check_stall`**（每 tick 批量踢队首） | ❌ 无救济式踢升（准入式判定 + 票据重置，详见 6.6） |
| 额外职责 | 连接数周期报告、过期连接清理 | `wait_timeout` 检查 | `wait_timeout` 检查 |

### 三个深层洞察

**1. "线程级" vs "组级"判定的分野**

MySQL 判的是"**这个线程跑了多久**"（`time_of_attach`），MariaDB/Percona 判的是"**这组有没有在排水**"（`queue_event_count`）。

- 线程级 ⇒ 能精确指出"是谁在拖"，stalled 的线程被单独豁免（不计配额）
- 组级 ⇒ 只知道"组卡了"，不知道是谁；只能靠唤醒新线程并行处理，stalled 标志是组级的

这也是为什么 MySQL 敢做"stalled 不计配额"——它有线程级的信息做支撑。

**2. `trylock` vs `lock` 的取舍**

Percona 的 "Don't disturb" 是对的：timer 是**全池单点**，被一个忙组阻塞会拖延其他所有组的检查。MariaDB 用阻塞锁能拿到更完整的检查结果（不会漏组），但把风险换成了延迟抖动。注意 MariaDB 的 tick 是 500ms——即使偶尔被阻塞，下个 tick 还有机会；如果 MySQL 那样 1ms 粒度的检查用阻塞锁，代价就不可接受了。

**3. 阈值与周期：解耦 vs 耦合**

MariaDB/Percona 的 `stall_limit` **既是判定阈值又是检查周期**（tick_interval = stall_limit = 500ms）——调参数等于同时调两件事。MySQL 把两者拆开（60ms 判定，1ms/10ms 检查周期），所以 26.7 能做到"线程一过 60ms 就在 1ms 内被发现"，响应粒度比两家的 500ms **细两个数量级**。这是 [`thread_pool.md`](thread_pool.md) 里"stall 响应慢一个数量级"结论的机制根源。

---

## 6.6 防饿死三家源码级辨析

> 低优先级的查询怎么避免永远排不上？三家给出了三种本质不同的答案。6.5 节已顺带提到踢升位置，本节展开全部源码。

### MySQL 26.7：救济式 + 速率限制

```cpp
static void check_trans_queue_for_prio_kickups(tp_group_t *my_tp_group,
                                               bool new_10ms,
                                               Time_pt time_of_check) {
  tp_client_low_level_t *client_cntx;

  client_cntx = my_tp_group->queued_trans.front();     // ★ 只看队首
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
    my_tp_group->queued_queries.push_back(client_cntx);   // 低优 → 高优
    /* Update statistics on prio kickups */
    my_tp_group->stats.prio_kickups++;
  }
}
```

三个要点：

1. **"每组每 10ms 最多移 1 条"的落地就是 `new_10ms` 参数**——由 stall check 线程传入，标识"进入了新的 10ms 区间"。只有新区间才开始才允许踢一条，这就是速率限制的全部实现（无额外定时器）。
2. **只看队首（`front()`）**——不遍历整个低优队列。含义：低优队列近似 FIFO，队首是等得最久的；把队首提走后，下一条 10ms 再提下一个。**这个"每次一条 + 只看队首"的组合，正是官方 222 秒最坏延迟推演里"把它前面的都移过去要 200 条 × 10ms = 2 秒"的机制来源**。
3. **注释承认设 0 的语义**："Setting this time to zero effectively means we're no queueing transactions at all"——`prio_kickup_timer = 0` 等于关闭排队（所有事务直接按高优处理）。

调用点在 **stall check 线程**（每轮遍历各组时顺带调用）——所以 MySQL 的踢升也是 timer 驱动的，这一点与 MariaDB 相同。

### MariaDB：救济式 + 批量踢

6.5 节已贴源码，这里只讲与 MySQL 的对照点：

```cpp
for (;;)
{
  c = thread_group->queues[TP_PRIORITY_LOW].front();      // 同样只看队首
  if (c && pool_timer.current_microtime - c->enqueue_time
           > 1000ULL * threadpool_prio_kickup_timer)
  {
    remove from LOW; push_back to HIGH;
  }
  else break;
}
```

| | MySQL 26.7 | MariaDB |
|---|---|---|
| 驱动 | stall check 线程（timer 驱动） | timer 线程（内嵌 `check_stall`） |
| 每轮踢几条 | **1 条**（`new_10ms` 限速） | **批量**（`for(;;)` 循环踢到队首不超时为止） |
| 阈值默认 | 1000ms | 1000ms（同名参数 `prio_kickup_timer`） |
| 统计 | `stats.prio_kickups` | 无专门计数 |

**批量 vs 限速**是实质差异：MariaDB 一个 tick 能把所有超时的队首连续踢完；MySQL 刻意限速 100 条/秒——**防止低优查询瞬间集体涌入高优队列**，把高优也淹没（踢升的目的是救饿死的，不是给低优开闸）。

### Percona：准入式 + ★ 锁感知（这是三家唯一的）

Percona **没有**"等太久再提"的救济机制（无 `prio_kickup_timer`）。它在**入队的那一刻**就判定：

```cpp
/*
   Checks if a given connection is eligible to enter the high priority queue
   based on its current thread_pool_high_prio_mode value, available high
   priority tickets and transactional state and whether any locks are held.
*/
inline bool connection_is_high_prio(const connection_t &c) noexcept {
  const ulong mode = c.thd->variables.threadpool_high_prio_mode;

  return c.thd->is_admin_connection() ||  // Admin connection is high prio
         (mode == TP_HIGH_PRIO_MODE_STATEMENTS) ||
         (mode == TP_HIGH_PRIO_MODE_TRANSACTIONS && c.tickets > 0 &&
          (thd_is_transaction_active(c.thd) ||
           c.thd->variables.option_bits & OPTION_TABLE_LOCK ||
           c.thd->locked_tables_mode != LTM_NONE ||
           c.thd->mdl_context.has_locks() ||
           c.thd->global_read_lock.is_acquired() ||
           c.thd->backup_tables_lock.is_acquired() ||
           c.thd->mdl_context.has_locks(MDL_key::USER_LEVEL_LOCK) ||
           c.thd->mdl_context.has_locks(MDL_key::LOCKING_SERVICE)));
}
```

★ **这不是"事务优先"，是"持锁优先"**。`TRANSACTIONS` 模式下要同时满足"有票据"**且**"持有以下任意一种锁"：

| 条件 | 对应的锁 |
|---|---|
| `thd_is_transaction_active` | 活跃事务 |
| `OPTION_TABLE_LOCK` / `locked_tables_mode != LTM_NONE` | 表锁 / `LOCK TABLES` |
| `mdl_context.has_locks()` | 任意 MDL |
| `global_read_lock.is_acquired()` | 全局读锁（FTWRL） |
| `backup_tables_lock.is_acquired()` | 备份锁 |
| `has_locks(USER_LEVEL_LOCK)` | `GET_LOCK()` 用户锁 |
| `has_locks(LOCKING_SERVICE)` | Locking Service 组件 |

**为什么锁感知重要**：持锁连接如果被饿死，它持有的锁就迟迟不释放，**放大**成其他所有等锁连接的等待——饿死一个持锁者等于饿死一串。所以 Percona 让持锁者插队，本质是**缩短锁持有时间以减少连锁阻塞**。这与 7 章（死锁案例）里"持锁的 C 需要先被调度"是同一个问题在优先级层的解法。

**票据的完整生命周期**（6 章已贴部分源码，串起来）：

```
连接建立/进低优队列时：tickets = thd->variables.threadpool_high_prio_tickets
入队判定：connection_is_high_prio() 为真 且 tickets > 0
    → tickets--，进高优队列
判定为否 → 进低优队列，且 tickets 重置为初值（下一条语句重新有机会）
admin 连接 → 无条件高优（queue_put 里特判，"to be able to login when threadpool is exhausted"）
```

即 **Percona 的粒度是"每条语句一次机会"**：票据用完落低优，但立即重置——下一条语句（如果满足 mode+锁条件）又有票据可用。限的是"同一连接连续插队的次数"，不是"永远失去机会"。

### 三家对比表（防饿死）

| 维度 | MySQL 26.7 | MariaDB | Percona |
|---|---|---|---|
| **思路** | 救济式（等太久再提） | 救济式（等太久再提） | **准入式**（入队时就判） |
| 驱动 | stall check 线程 | timer（内嵌 check_stall） | **无 timer 参与**（入队时判定） |
| 判定依据 | 队首等待 > `prio_kickup_timer` | 队首 `enqueue_time` > 同名参数 | `mode` + `tickets` + **持锁状态** |
| 速率 | ★ 每 10ms 最多 1 条 | 每 tick **批量**踢 | 票据计数（用完即重置） |
| 判定范围 | 只看队首 | 只看队首（循环） | 每条连接独立 |
| 锁感知 | ❌ | ❌ | ★ **✅（8 种锁条件）** |
| admin 特判 | Admin 线程组（独立组） | — | 强制高优队列 |
| 统计 | `stats.prio_kickups` | — | `Threadpool_threads` 侧无专列 |

### 两个深层洞察

**1. 救济式 vs 准入式的代价互换**

救济式（MySQL/MariaDB）的代价是**最坏等待时间**——MySQL 官方 222 秒推演就是救济式的极限：低优队列越长，"把它前面的都救完"越慢。准入式（Percona）没有这个问题（该高优的在入队时就进去了），但代价是**判定必须便宜**——所以 Percona 的判定全是读连接自身状态（tickets、thd 标志位），没有队列扫描；而救济式天然要把"扫描队列"放在低频的 timer 里做。

**2. 锁感知是 Percona 对"死锁案例"的系统性回答**

7 章的死锁案例（A、B 等 C 提交，但 A、B 占满线程导致 C 无法调度）在优先级层的解法就是：**让持有锁的 C 有更高优先级**。MySQL 的解法是 `wait_begin` 释放 credit + Admin 组；MariaDB 是"每组至少 2 线程"硬保证；Percona 则是把"持锁"直接编码进优先级判定——三条路线在这里汇合到同一个问题上。

---

## 7. 死锁：经典案例与三家对策

### 7.1 内核月报给出的经典死锁场景

> A、B、C 三个事务，A、B 需要等 C 提交；**A、B 先被调度且活跃线程数达到 `thread_pool_max_threads`，C 无法被调度提交 → A、B 永久等待**。

### 7.2 三家对策

| | 对策 | 性质 |
|---|---|---|
| **MySQL 26.7** | ① wait_begin/wait_end 回调（A、B 阻塞时**主动释放 credit**，让 C 有机会进来）<br>② MTL + 长事务背压<br>③ admin thread group 兜底 | **主动式** |
| **MariaDB** | ① **每组至少 2 线程**（1 worker + 1 listener）硬保证（即使达 max_threads）<br>② `extra_port` 池外后门 | **结构式 + 后门** |
| **Percona** | ① `too_many_busy_threads` 限制 active+waiting<br>② **normal-priority queue throttling**：繁忙时只处理已开始事务的事件，让锁尽快释放、池"排空"<br>③ tickets 防单连接霸占 | **节流式** |

**Percona 的 throttle 是最直接的防死锁设计**：

> **问题**：group oversubscribed 且多数线程在等锁，而持锁事务的连接还没被捞起来。oversubscribe 限制**不计"标记 inactive 但在等待"的线程**，于是线程数持续增长直到 `max_threads`；若持锁连接在此之前没进池 → **死锁**；若进来了 → 并发跑接近 max_threads 个线程，性能极差。
>
> **对策**：group 内 worker 总数（active + waiting）达到 `oversubscribe` 时**节流普通优先级队列**：不启动新事务、不创建新线程，**只处理已开始事务的排队事件**，让锁尽快释放，使线程池"排空"（drain）。

**MariaDB 的"每组至少 2 线程"是最硬的结构保证**（源码可见：`wake_or_create_thread` 的条件含"全局 < max_threads **或** group < 2"）。

---

<a id="c8"></a>
## 8. 行业演进：AliSQL 与 TXSQL

国内云厂商在 Percona/MariaDB 基础上做了大量工程优化，这些是开源三家**没有**的。

### 8.1 阿里云 AliSQL

- **管理类 SQL 高优先级**，保证高负载时新建连接、管理、监控操作稳定执行
- **复杂查询低优先级 + 最大并发数限制**，避免复杂 SQL 耗尽资源
- **支持动态开关线程池**（`thread_pool_enabled`）
- **两层队列**（内核月报 2019/02）：
  - 第一层 = **网络请求队列**：「请求队列」（不在事务中）+「高优先级队列」（已在事务中，收到即执行）
  - 第二层 = **工作任务队列**：查询队列 / 更新队列 / 事务队列；管理操作直接执行不入队
  - 每层各自设并发度（接近 CPU 数），只要总线程数 > 四类操作并发度之和，各类操作互不干扰
  - 新增参数：`thread_pool_normal_weights`（查询/更新目标线程比例）、`thread_pool_trans_weights`（事务目标比例）
  - 并发度 = `oversubscribe × 目标比例 / 100`
- 内核月报 2016/02 列出的 4 项改进：
  1. `LOCK TABLES READ` 等显式表锁事件放入高优先级队列
  2. **binlog dump 线程不受 oversubscribe 限制**（多个 dump 线程落同一组会轻易超限）
  3. 增加 `information_schema.THREAD_GROUP_STATUS` 诊断表
  4. **本地登录用户走老的 one_thread_per_connection**，解决调度异常时无法连接

### 8.2 腾讯云 TXSQL

**核心方案与 Percona 完全一致**，额外三项优化：

**(1) 线程池动态切换**

`thread_handling_switch_mode`：`disabled` / `stable`（只有新连接迁移）/ `fast`（新连接+新请求都迁移，默认）/ `sharp`（kill 活跃连接迫其重连）

切换点选择很讲究：
- Per_thread → Thread_pool：**`do_command` 之后**（请求处理完，尚未接下一个请求之前）
- Thread_pool → Per_thread：**`threadpool_process_request` 之后、`start_io` 之前**（需先从 epoll 摘除、清除组信息，再创建 `handle_connection` 线程）

> 原生 MySQL 的 `thread_handling` 是只读参数，不允许在线修改。TXSQL 在初始化阶段就初始化**所有**连接管理方法以支持动态切换。

性能影响：`pool → per_thread` 无堆积；`per_thread → pool` 在 QPS 极高时可能有请求累积（线程池之前处于休眠）。解法：适当增大 oversubscribe + 调小 stall_limit，或短暂降流量。

**(2) 线程池负载均衡**

负载度量的三种信息策略：
- `queue_length`：两队列长度。简单，但未考虑 event 处理效率差异
- `average_wait_usecs_in_queue`：最近 n 个 event 的平均等待时间
- `group_efficiency`：周期内处理完的 event 数 /（存量 + 新增）

为什么不做全局排序：

> "为了维护全局线程组负载动态序列，线程组每处理完一次任务都可能需要更新自身的状态，并在**全局锁**的保护下更新其在全局负载序列中的位置，如此一来对性能的影响势必较大" → **改为只与左右相邻 1-2 个 group 局部比较**

触发条件：负载 > 阈值 M **且** 与相邻组负载比 > N（避免过度触发造成抖动）。手段：新连接避开高负载组 + 旧连接迁移。

**(3) 线程池断连优化（指出 Percona 的缺陷）**

> percona 线程池需要等到请求处理结束之后才会使用 epoll 重新监听用户连接的新网络事件。**如果用户连接在重新被 epoll 监听前自行退出了，此时用户连接发出的断连信号无法被 epoll 捕捉**，因此在 mysql 服务器端无法及时退出该用户连接。

后果：① 客户端已退出但服务端仍运行该连接，继续消耗 CPU/内存、甚至持有锁；② 连接数不清理，可能触达 max_connections 上限。

对策：
1. epoll 接到正常网络事件后，**立刻监听该用户连接的断连事件**
2. 所有连接退出**从同步改为异步**，先入 `quit_connection_queue`
3. epoll 接到断连事件 → `thd->killed = THD::KILL_CONNECTION` + 入队
4. **listener 每隔约 100ms 处理一次 quit 队列**

> **MySQL / MariaDB / Percona 三家都没有这个优化。** 这是国内厂商在极端场景下的实践积累，值得在做高并发短连接方案时参考。

### 8.3 监控能力对照

| | 状态变量/表 | 数量 |
|---|---|---|
| Percona / MariaDB（原生） | `Threadpool_threads`、`Threadpool_idle_threads` | 2 |
| Percona 9.7（增强） | + `requests_waiting_in_queue` / `hp_queue` / `starved_in_queue` / `average_*_wait_us` | 6 |
| AliSQL | `active/big/dml/idle/qry/total/trx/wait_threads` | 8 |
| TXSQL | `show threadpool status`（groupid、havelistener、active_thread_count、waiting_thread_count、queue_size、high_prio_queue_size、create_thread_num、wake_thread_num、oversubscribed_num、check_stall_nolistener、check_stall_stall、max_req_latency_us、events_consumed、average_wait_usecs_in_queue、connections_moved_in/out…） | **25** |
| **MySQL 26.7** | 4 张 PFS 表（含单连接/单线程粒度） | **最细** |

---

## ★ 本机制里的工程实现技法

### 一、高级语法技巧 / C++ 特性

| 技法 | 落点 |
|---|---|
| **抽象基类 + 纯虚函数** | MariaDB `TP_connection`（`init()`/`start_io()`/`wait_begin()`/`wait_end()`）与 `TP_pool` |
| **`I_P_List` + `I_P_List_adapter` 模板** | 侵入式链表，adapter 用成员指针指定用哪对 `next/prev`（MariaDB 103-109 / Percona 105-127） |
| **`enum class operation_origin`** | MariaDB 独有，类型安全地区分 WORKER/LISTENER 事件来源，供 `polls[2]`/`dequeues[2]` 分类统计 |
| **`PRAGMA_DISABLE_CHECK_STACK_FRAME`** | `get_event` 里 `native_event ev[MAX_EVENTS]` 会撑大栈帧（注释："may bloat the stack frame beyond 16 KiB"），显式关闭告警 |
| **`Atomic_counter<uint32_t>`** | MariaDB `TP_STATISTICS::num_worker_threads` |
| **`noexcept`** | Percona 大量使用（`queue_get`、`too_many_*`、`get_event` 外的多数函数） |
| **`alignas(128)` + `static_assert(sizeof == 512)`** | Percona `thread_group_t`，**编译期强制布局**防 false sharing |
| **栈上分配线程对象** | Percona `worker_main`：`worker_thread_t this_thread;` —— 线程退出即销毁，无堆管理 |
| **`std::atomic<uint64>`** | `pool_timer_t::current_microtime` / `next_timeout_check`（timer 与 worker 跨线程读） |
| **条件编译宏作行为开关** | Percona `#ifdef THREADPOOL_CREATE_THREADS_ON_WAIT`（见 3.5）—— 编译期决定是否"等待时建线程" |

### 二、经典算法的实现落地（不限于并发）

| 算法 | 落点 | 说明 |
|---|---|---|
| **LIFO 唤醒** | `waiting_threads.push_front(current_thread)` | 注释原话："ensures LIFO wakeup order (hot caches, working inactivity timeout)" —— 最后睡的先醒，栈还在 cache 里 |
| **self-pipe trick** | MariaDB `wake_listener` | 注释明写 "Self-pipe trick is used in most cases, except IOCP" |
| **睡前非阻塞轮询** | `io_poll_wait(pollfd, ev, MAX_EVENTS, 0)` | 睡前的最后一次尝试，避免"刚要睡就有事件到达"的唤醒往返 |
| **分档节流** | `microsecond_throttling_interval()` | 组内线程越多，创建下一个线程的等待间隔越长 |
| **Ticket 配额（公平调度）** | Percona `connection->tickets--` / 恢复 | 限制单连接插队次数，防高优垄断 |
| **优先级老化（aging）** | MariaDB `check_stall` 中的 prio kickup | 低优等待超时升级 |
| **动态角色切换** | `get_event`："无 listener 则自己变 listener" | 无需常驻的 listener 线程，线程数即并发数 |

### 三、复杂体系与设计模式的代码结构

**1. 策略模式（平台隔离）**

```
TP_pool (抽象)
   ├── TP_pool_generic    Unix：自实现 io_poll（epoll/kevent/ports）
   └── TP_pool_win        Windows：原生 CreateThreadpool（IOCP）
```

`threadpool.h` 用两个子类把平台差异隔离，上层通过基类指针调用。

**2. 模板方法模式（连接契约）**

`TP_connection` 定义 `init()` / `set_io_timeout()` / `start_io()` / `wait_begin()` / `wait_end()` 五个纯虚函数，`TP_connection_generic` 实现。线程池核心逻辑只依赖契约，不依赖具体连接类型。

**3. 侵入式容器（C 性能 + C++ 类型安全）**

`I_P_List` 把 `next/prev` 指针**直接嵌进业务对象**（`TP_connection_generic::next_in_queue`），零额外内存分配、O(1) 插删，同时用 adapter 模板保证类型安全。这是 MySQL 源码里"穷人版 STL"的典型代表（对照 `sql_plist.h`）。

**4. 角色模式（动态职责）**

worker 与 listener **不是两类线程，而是同一批线程的两种角色**。`get_event` 里谁发现组内没有 listener，谁就临时承担 listener 职责。这比 MySQL 的 `waiting_thread` 标记更彻底——MariaDB/Percona 的"listener"会真的阻塞在 `io_poll_wait` 直到有事件。

**5. 不透明句柄（平台抽象）**

```cpp
typedef struct epoll_event native_event;   // Linux
typedef struct kevent     native_event;    // FreeBSD/macOS
typedef port_event_t      native_event;    // Solaris
typedef OVERLAPPED_ENTRY  native_event;    // Windows IOCP
```

一个 typedef 屏蔽四套 I/O 多路复用 API 的差异，上层 `io_poll_wait` / `io_poll_associate_fd` / `io_poll_start_read` 接口统一。

---

## 可观测性

<a id="c9"></a>
### 9. 参数与监控对照

### 9.1 参数全表

| 参数 | MySQL 26.7 | MariaDB 12.3 | Percona 8.0 |
|---|---|---|---|
| 主开关 | `thread_handling`(+插件) | `thread_handling` | `thread_handling` |
| 动态开关 | ❌ | ❌ | ❌（TXSQL/AliSQL 支持） |
| 组数 | `thread_pool_size`=**16**（`DEF_POOL_SIZE=16`，1-512） | =**8**（源码 `DEFAULT(8)`，**不是** CPU 数） | =**CPU 数**（`DEFAULT(my_getncpus())`） |
| 每组/全局线程上限 | 4096/组 | `max_threads`=65536 (Win 1000) | `max_threads`=100000 |
| oversubscribe | ❌（MTL 替代） | =3，语义 A | =3，语义 B |
| stall limit | =6（**10ms 单位**） | =500（**ms**） | =500（ms） |
| 事务数限制 | **`max_transactions_limit`** | ❌ | ❌ |
| 长事务背压 | **`longrun_trx_limit`**=2000ms | ❌ | ❌ |
| 优先级模式 | 事务活跃/kickup=0/高优连接 | `priority`=auto/high/low | `high_prio_mode`=transactions/statements/none |
| 防饿死 | `prio_kickup_timer`=1000ms | `prio_kickup_timer`=1000ms | **`high_prio_tickets`**=UINT32_MAX |
| 锁感知 | ❌ | ❌ | **✅** |
| 专用 listener | `dedicated_listeners`=false | `dedicated_listener`=0 | ❌ |
| 空闲线程 | `max_unused_threads`=32 | `idle_timeout`=60s | `idle_timeout`=60s |
| 高精度统计 | — | `exact_stats`=0 | — |
| 连接迁移 | ❌ | **`change_group`**（fix_group 标志） | **`change_group`**（每次重算） |
| 应急通道 | admin thread group | `extra_port` | `admin_port`（8.0.14+） |
| 线程数下限 | ❌ | `min_threads`=1（**仅 Windows**） | `min_threads`=1 |
| `thread_pool_max_threads` 默认 | ❌（用 3×vCPU 节流替代） | 65536（Win 1000） | 100000 |

> ★ **三家的 `thread_pool_size` 默认值互不相同**（16 / 8 / CPU 核数），这是个容易被忽略的差异：
> 官方（Oracle）建议"= 物理核数"，但 **MySQL 自己默认 16、MariaDB 默认 8**，只有 **Percona 默认就是核数**。
> 也就是说在 MySQL 与 MariaDB 上，**默认配置并不等于官方推荐配置**，必须按核数手动调；Percona 开箱即符合建议。
> （本行已按源码核实：MySQL `thread_pool.h` 的 `DEF_POOL_SIZE = 16`；MariaDB `sys_vars.cc` 的 `DEFAULT(8)`；Percona `DEFAULT(my_getncpus())`。）

### 9.2 MariaDB 独有参数

| 变量 | 默认 | 说明 |
|---|---|---|
| `thread_pool_priority` | auto | 10.2.2+；Global+Connection 作用域 |
| `thread_pool_dedicated_listener` | 0 | 10.5+；专有 listener，不做 worker |
| `thread_pool_exact_stats` | 0 | 10.5+；高精度时间戳统计入队时刻（有性能代价） |
| `thread_pool_min_threads` | 1 | **仅 Windows**；防 Windows 原生池退休线程 |
| `extra_port` / `extra_max_connections` | 0 / 1 | 非动态 / 动态 |

> 启用线程池后 **`thread_cache_size` 不再生效**，`Threads_cached` 恒为 0。

### 9.3 Percona 状态变量（9.7）

- `Threadpool_threads`（总数）、`Threadpool_idle_threads`（`busy = threads − idle`）
- `Threadpool_requests_waiting_in_queue` / `_in_hp_queue`
- **`Threadpool_requests_starved_in_queue`** —— 普通队列被高优饿死的请求数，**非 0 即池饱和信号**
- `Threadpool_average_queue_wait_us` / `_hp_queue_wait_us` —— 格式 `avg/min/max/dev/cnt`，微秒

> **Percona 独有门槛**：**"If you have fewer than 20,000 connections, using the thread pool does not provide significant benefits."**

---

<a id="c10"></a>
## 10. 性能数据

### MariaDB 官方 benchmark（5.5，无优先级队列，24 核 pitbull，Sysbench 0.4）

**POINT_SELECT**（差异最悬殊）

| 并发 | 16 | 64 | 256 | 512 | 1024 | 2048 | 4096 |
|---|---|---|---|---|---|---|---|
| per_thread | 148673 | 169747 | 69036 | 42041 | 21775 | 4368 | **282** |
| threadpool | 143222 | 167270 | 164983 | 158410 | 148690 | 147107 | **143934** |

**OLTP_RW**

| 并发 | 16 | 64 | 256 | 512 | 1024 | 2048 | 4096 |
|---|---|---|---|---|---|---|---|
| per_thread | 4561 | 5332 | 2874 | 2476 | 1380 | 265 | **53** |
| threadpool | 4504 | 5694 | 5302 | 4514 | 2548 | 1186 | **484** |

**UPDATE_NOKEY**

| 并发 | 16 | 64 | 256 | 512 | 1024 | 2048 | 4096 |
|---|---|---|---|---|---|---|---|
| per_thread | 65213 | 19418 | 11155 | 8742 | 5645 | 635 | **332** |
| threadpool | 64902 | 70037 | 69930 | 69929 | 67099 | 62376 | **17766** |

**规律**：
- 低并发（≤64）时两者持平，per_thread 有时略优（无调度开销）
- **临界点约 128–256 并发**（24 核，≈ 核数的 5–10 倍），之后 per_thread 断崖下跌

### AliSQL 多队列线程池（TPCC 1000DW）

- 并发 1000 → `Threadpool_threads=179`，`Threads_connected=1001`，TpmC ≈ 8 万
- 并发 3000 → **线程数仍是 179**，`Threads_connected=3001`，TpmC 仍 ≈ 8 万，**无 TPS 损失**

（文中自述为内部实验室版本，公共云不可见）

> 腾讯 TXSQL、Percona 官方手册都**没有**公开性能数据。

---

## Misc

### 易混淆概念

| 概念 | 澄清 |
|---|---|
| **Percona 线程池不是 MariaDB 的"分支版本"** | 是**移植**：从 MariaDB 代码移植后独立演进，参数与语义已分化（最典型是 `oversubscribe`） |
| **`oversubscribe` 在两家语义不同** | MariaDB = 稳态收敛上限（**与线程创建无关**）；Percona = 准入闸门（活跃+等待超限则节流低优队列）。详见[第 6 章](#c6) |
| **`stall_limit` 单位是 ms，不是 10ms** | MariaDB / Percona 都是 **1ms**（默认 500）；**MySQL 是 10ms**（默认 6 = 60ms）。跨库迁移时极易搞错 |
| **"listener"不是常驻线程** | 是 worker 线程的**动态角色**，可与 worker 互相切换。MySQL 的 `waiting_thread` 同理（但 MySQL 另有 dedicated listener 选项） |
| **`extra_port` ≠ `admin_port` ≠ Admin 线程组** | MariaDB 用**池外独立端口**（走 one-thread-per-connection）；Percona 8.0.14+ 用 `admin_port`；MySQL 用**池内 Admin 线程组**。三种不同路线 |
| **`Threadpool_idle_threads` 包含被 IO/锁阻塞的线程** | MariaDB 文档明说：等待中的线程也算 inactive，**仅 Unix 有意义** |
| **极端长查询下线程池会退化** | MariaDB 注释承认："would slowly approach thread-per-connection behavior"——这是设计上可接受的兜底，不是 bug |
| **认证发生在 worker 线程内** | 两家都是：连接先入低优队列，由 worker 取出后完成登录认证。MySQL 26.7 用专门的 `CONNECTION_HANDLER_WORKER_THREAD` 隔离了这件事 |

### ★ 本篇核实出的"流传但不准确"的说法

1. **"oversubscribe 限制线程创建"** —— MariaDB **官方文档明确否定**："does not play any part in the creation of new worker threads"。这个说法来自 Percona 分支的语义和早期内核月报，被误当成通用定义。
2. **"Percona 与 MariaDB 实现差不多"** —— 结构骨架相似（同源），但 **tickets / busy 计数 / wait_begin 编译期开关 / 结构布局** 四处已实质分化，`get_event` 的非阻塞轮询分支逻辑完全不同。
3. **"MariaDB 用 epoll"** —— 更准确是"按 OS 各取最优"：Linux epoll、Solaris event ports、FreeBSD/macOS kevent、Windows IOCP。epoll 只是 Linux 分支。
4. **"`thread_pool_size` 在 Percona 是只读的"** —— 8.0 文档标 No、9.7 文档标 Yes，二者矛盾，需实测确认。

<a id="c11"></a>
### 11. 交叉验证发现的文档矛盾

1. **Percona `thread_pool_size` 是否动态**：8.0 文档标 **No**，9.7 文档标 **Yes**，必有一处笔误（MariaDB 侧是动态的）。

2. **Percona 8.0 对 stall 的描述错误**："the pool may **redistribute it to another group**" —— 与源码（唤醒/创建本 group 内线程）不符，**不建议引用**。

3. **MariaDB `thread_pool_size` 范围矛盾**：变量页 Range 写 `1-128`，但启动最大值是 100000。

4. **MariaDB 节流间隔数值不一致**：现行文档（带 THROTTLING_FACTOR 分档）与中文文章引用的 5.5 源码（固定 50/100/200ms）不同 —— 按目标版本源码为准。

5. **oversubscribe 语义分歧**：MariaDB 官方 vs Percona/社区口径 —— **不是错误，是两个分支语义确实不同**（见第 6 章）。

6. **Percona 8.0.14 的破坏性变更**：`extra_port` / `extra_max_connections` 被 `admin_port` 取代并移除，**配置保留会导致启动失败**，升级前必须删除。

---

## 11. MariaDB / Percona 运维实践

> 本章与 [`thread_pool.md`](thread_pool.md) 的「运维实践」对称，补 MariaDB / Percona 两家。取值与建议均标注来源（官方文档 / 源码），不是经验猜测。

#### 10.1 启用

| | MariaDB | Percona |
|---|---|---|
| 形态 | **内建**（非插件） | 内建（8.0.12-1 起原生，移植自 Percona 5.7） |
| 开关 | `thread_handling=pool-of-threads` | 同左 |
| Windows | **默认就是 `pool-of-threads`**（无需配置）；旧版 XP/2003 会静默退回 one-thread-per-connection | 同 MySQL |

#### 10.2 参数默认值与语义（含源码作用）

| 参数 | MariaDB | Percona | 动态 | 语义 / 源码作用 |
|---|---|---|---|---|
| `thread_pool_size` | 源码 `DEFAULT(8)`（官方文档称"=CPU 数"，**与源码矛盾，以源码为准**） | `DEFAULT(my_getncpus())` = CPU 核数 | ❌ 静态 | 线程组数。Percona 官方建议常用 2~64 |
| `thread_pool_oversubscribe` | 3 | 3 | ✅ | 见 10.3 |
| `thread_pool_max_threads` | 65536（Unix）/ **1000**（Win） | **100000** | ✅ | 池内 worker 总上限 |
| `thread_pool_stall_limit` | **500（毫秒）** | **500（毫秒）** | MariaDB ✅ / **Percona ❌ 需重启** | ★ 单位是**毫秒**；MySQL 是 10ms 单位（默认 6=60ms），**差一个数量级还差一个单位** |
| `thread_pool_idle_timeout` | 60 秒 | 60 秒 | ✅ | 空闲 worker 退出前等待 |
| `thread_pool_priority` | auto/high/low | —（用 high_prio_mode 替代） | ✅ | 见 10.3 |
| `thread_pool_high_prio_mode` | ❌ | transactions / statements / **none** | ✅ | 默认 `transactions` |
| `thread_pool_high_prio_tickets` | ❌ | **UINT32_MAX** | ✅ | 见 10.3 |
| `thread_pool_min_threads` | 1（**仅 Windows**） | 1 | — | Windows 防原生池退休线程 |
| `thread_pool_mode` | windows/generic，**只读**，默认 `windows` | ❌ | ❌ | 见 10.4 |

两个值得单独说的默认值：

- **`thread_pool_max_threads` 三家全不同**：MySQL **没有**这个参数（用 3×vCPU 创建节流替代）；MariaDB 65536（Windows 仅 1000）；Percona 100000。
- **Percona 的 `stall_limit` 非动态**——调优必须重启，这是运维上最容易忽略的一条。

#### 10.3 oversubscribe 与优先级：源码里到底怎么用

**`oversubscribe` 的作用点**（两家不同，源码实证）：

```cpp
// MariaDB：只用于"是否从队列取任务"（稳态收敛）
static bool too_many_threads(thread_group_t *thread_group) {
  return (thread_group->active_thread_count > 1 + (int)threadpool_oversubscribe
          && !thread_group->stalled);
}
// 用法：bool oversubscribed = too_many_threads(...); if (!oversubscribed) queue_get(...)

// Percona：两个判定，多一个 busy（活跃+等待）概念
static bool too_many_threads(...) {
  return (active_thread_count >= 1 + (int)threadpool_oversubscribe && !stalled);
}
inline bool too_many_busy_threads(const thread_group_t &thread_group) {
  return (active + waiting > 1 + (int)threadpool_oversubscribe);
}
```

| | MariaDB | Percona |
|---|---|---|
| 比较符 | `>` | `>=`（差 1） |
| 判定数 | 1 个 | **2 个**（多 `too_many_busy_threads`） |
| 作用 | 是否取队列 | 额外用于**低优队列节流**（准入闸门） |
| 官方口径 | "primarily for internal use, **not meant to be changed for most users**" | "per-group queue saturation threshold" |

> ★ MariaDB 官方明确说 oversubscribe **主要是内部用途、大多数用户不该改**——这点与 Percona 把它当可调节流阀不同。

**Percona 高优票据的源码机制**：

```cpp
// 入队判定
if (/* 满足高优条件 */) {
    c->tickets--;
    thread_group->high_prio_queue.push_back(c);
} else {
    c->tickets = c->thd->variables.threadpool_high_prio_tickets;   // ★ 重置
    thread_group->queue.push_back(c);                              // 低优
}

// queue_put：新连接
connection->tickets = connection->thd->variables.threadpool_high_prio_tickets;
// put new admin connection into high prio queue instead of normal
// to be able to login when threadpool is exhausted
if (connection->thd->is_admin_connection()) {
    thread_group->high_prio_queue.push_back(connection);           // ★ 强制高优
}
```

- 票据用完 ⇒ 进低优并**重置**为初值（下一轮重新计）
- ★ **admin 连接被强制放进高优队列**，注释写明目的："to be able to login when threadpool is exhausted"——这是 Percona 的应急通道在调度层的实现（不只是开个端口）

**MariaDB 的 `priority`**：`auto` 时**事务内的查询自动升为高优**，让运行中的事务更快结束，从而降低并行事务数、提升吞吐；也可显式设 high/low。配 `prio_kickup_timer` 防止高优连接垄断。

#### 10.4 `thread_pool_mode`（MariaDB 独有，Windows 相关）

```cpp
"thread_pool_mode",
"Chose implementation of the threadpool. Use 'windows' unless you have a "
"workload with a lot of concurrent connections and minimal contention",
READ_ONLY GLOBAL_VAR(threadpool_mode), CMD_LINE(REQUIRED_ARG),
threadpool_mode_names, DEFAULT(TP_MODE_WINDOWS)
```

- **只读**，默认 `windows`
- 语义：选择线程池实现——Windows 原生池（`CreateThreadpool`）还是 generic（Unix 式）实现
- 官方建议：默认用 `windows`，除非是"大量并发连接且争用极小"的负载
- 源码：`threadpool.h` 注释 "Thread pool implementation, windows or generic"；`threadpool_common.cc` 里按 `TP_MODE_WINDOWS` 分派

#### 10.5 监控

| | MariaDB | Percona |
|---|---|---|
| 状态变量 | `Threadpool_threads`（池内线程数）、`Threadpool_idle_threads`（空闲线程数，**仅 Unix 有意义**） | 同左 |
| 专用表 | I_S：`THREAD_POOL_GROUPS` / `THREAD_POOL_QUEUES` / `THREAD_POOL_STATS` / `THREAD_POOL_WAITS` | PFS 附加表 |
| 命令 | `SHOW STATUS LIKE 'Threadpool%'` | 同左 |

`Threadpool_idle_threads` 的口径要注意：**被阻塞（等 IO / 等锁）的线程也算 idle**，不是"没分配到工作"。

> ★ Percona 官方提醒：**动态变量的修改只对新建连接生效**，已有会话保留原设置直到重连。

#### 10.6 应急通道与"池被阻塞怎么救"

| | MariaDB | Percona |
|---|---|---|
| 机制 | `extra_port` + `extra_max_connections`（走 one-thread-per-connection） | `admin_port`（8.0.14+） |
| 配置 | `extra_port=8385` / `extra_max_connections=10` | `admin_port` |

> ⚠️ **Percona 8.0.14 的Breaking变更**：`extra_port` 与 `extra_max_connections` **已被移除**，配置文件里若仍保留，**服务器会启动失败**。升级前必须先删掉这两行。

**★ 池被完全阻塞的场景（MariaDB 官方给出）**：

```
客户端执行 FLUSH TABLES WITH READ LOCK 然后暂停
  → 其他客户端涌入写操作，连接数超过 thread_pool_max_threads
     → 整个池阻塞，连 UNLOCK TABLES 都发不出去
        → MaxScale 等监控系统也无法连入
```

解法只能靠应急端口：连进去后**调大 `thread_pool_max_threads`**，或 **kill 掉持有全局锁的连接**（它处于 sleep 状态）。

这也是为什么 MariaDB 官方强调"即使把 `max_threads` 设很大，全局锁场景仍可能阻塞整个池"——**上限设得再高也挡不住这种自锁**，必须有独立通道。

#### 10.7 已知陷阱与不适合场景

1. **`thread_cache_size` 失效**：启用线程池后不再使用，`Threads_cached` 恒为 0
2. **突发负载**：长时间空闲 + 短时极高活跃，线程创建节流反而不利；可调 `idle_timeout`（Unix）或 `min_threads`（Windows）缓解
3. **长非让出查询**（数据仓库场景）：永不等待、也不上报等待的查询会拖住整组，只能靠 stall 检测兜底
4. **"简单查询永远快"的假设失效**：池开启后，**连 `SELECT 1` 都可能排队**——这是官方明确列出的不适合场景
5. **Percona 动态变量只对新连接生效**（见 10.5）
6. **压测要分离 CPU**：官方建议 sysbench 驱动与服务器跑在不同 CPU（`taskset -c`），经验比例是 1/4 给压测、3/4 给服务器；能分 NUMA 节点更好。否则驱动线程会抢占服务器线程，测出不真实的低值
7. **I/O 多路复用差异**：MariaDB 按 OS 各取最优（Windows IOCP / Linux epoll / Solaris event ports / FreeBSD+macOS kevent），MySQL 只在 Linux 用 epoll、其余退化为 `poll()`

---

## 参考来源

**官方文档**
1. [MariaDB - Thread Pool in MariaDB](https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/buffers-caches-and-threads/thread-pool/thread-pool-in-mariadb)
2. [MariaDB - Thread Groups in the Unix Implementation](https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/buffers-caches-and-threads/thread-pool/thread-groups-in-the-unix-implementation-of-the-thread-pool)
3. [MariaDB - Thread Pool System and Status Variables](https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/buffers-caches-and-threads/thread-pool/thread-pool-system-status-variables)
4. [Percona Server for MySQL 9.7 - Thread pool](https://docs.percona.com/percona-server/9.7/threadpool.html)
5. [Percona Server for MySQL 8.0 - Thread pool](https://docs.percona.com/percona-server/8.0/threadpool.html)

**内核月报与深度博客**
6. [MySQL · 特性分析 · 线程池 · 数据库内核月报 2016/02](http://mysql.taobao.org/monthly/2016/02/09/)
7. [MySQL · 最佳实践 · MySQL 多队列线程池优化 · 内核月报 2019/02](http://mysql.taobao.org/monthly/2019/02/09/)
8. [万字长文 | 业内 MySQL 线程池主流方案详解（dbkernel / 卢文双）](http://dbkernel.com/2023/05/04/mysql-threadpool-main-solutions-details/)
9. [原创｜线程池详解 - 腾讯数据库技术](https://cloud.tencent.com/developer/article/1892326)
10. [MariaDB · 源码分析 · thread pool - 阿里云开发者社区](https://developer.aliyun.com/article/560514)
11. [MySQL 线程池总结 - 腾讯云开发者社区](https://cloud.tencent.com/developer/article/1899769)

**本地源码**
12. MariaDB 12.3.3：`/Users/morphyzhang/Documents/Code/DB/MariaDB`（`sql/threadpool_generic.h/.cc` 等）
13. Percona Server 8.0.43-34：`/Users/morphyzhang/Documents/Code/DB/percona-server`（`sql/threadpool_unix.cc` 等）
14. MySQL 26.7.0 副本：`.tmp_tp/`

---

### 相关篇目

- [`thread_pool.md`](thread_pool.md) —— **MySQL 26.7 线程池源码级剖析**（本篇的对照方；插件化两套回调、`EPOLLONESHOT`、credit/MTL、stall 给长查询独立名额、冷热线程分层）
- [`connection_threading.md`](connection_threading.md) —— Per_thread 连接模型（线程池的对立面；末节收在 `no-threads` 与插件接口，是理解"社区版只有接口无实现"的前置）
- [`pfs.md`](../observability/pfs/pfs.md) —— PFS 插桩机制（本篇"监控能力对照"的基础）
- [`../../infra/threading_architecture.md`](threading_architecture.md) —— 线程架构总览

> **姊妹篇**：[`thread_pool.md`](thread_pool.md) —— MySQL 26.7 线程池源码级剖析，本篇的对照基准。
