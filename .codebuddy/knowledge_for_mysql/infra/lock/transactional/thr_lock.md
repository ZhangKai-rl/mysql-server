# server 层表锁 THR_LOCK 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 `mysys/thr_lock.cc` 排队锁框架、`THR_LOCK`/`THR_LOCK_DATA`/`MYSQL_LOCK` 三件套、13 种 `thr_lock_type` 与优先级编码、写优先与防饥饿、并发插入的四处锁升级、以及它与 MDL、InnoDB `lock_t` 的分工。
>
> **边界**：本篇讲 **server 层表级排队锁 THR_LOCK**；MDL 框架见 [`mdl.md`](mdl.md)，InnoDB 的表锁与行锁见 [`innodb_trx_lock.md`](innodb_trx_lock.md)。三者的分工见「理论基础」与「与 InnoDB 的关系」两节。

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

THR_LOCK 是 MySQL 3.x 就有的、**按"已打开的锁对象地址"排队的表级读写锁**框架，实现在 `mysys/thr_lock.cc`。它有两级结构：

```
THR_LOCK        —— 主锁对象，挂在每个引擎的 share 上（一张表一个）
    ├── read / read_wait / write / write_wait   四个队列
    └── THR_LOCK_DATA  —— 锁实例，每个 handler 打开实例一个（"某线程对某表的一次请求"）
```

它**不是**元数据锁，也**不是** InnoDB 的行锁——很多人把 MySQL 的"表锁"理解成一件事，实际上有三套完全不同的东西（见「与 InnoDB 的关系」）。

### 用途

文件头注释把定位讲得很清楚（原文翻译）：

> 给 POSIX 线程用的读写锁。一把锁由主锁（`THR_LOCK`）与锁实例（`THR_LOCK_DATA`）组成；一个线程对同一把锁可以有任意多个锁实例。**每个线程必须通过 `thr_multi_lock()` 一次性获取它需要的全部锁，以避免死锁。** 所有锁实例最终都必须释放。

三句话点出了这个框架的全部特征：**两级结构**、**批量申请**、**无死锁检测**。

今天它实际承担的职责：

1. **非事务引擎（MyISAM / MEMORY / ARCHIVE / CSV / BLACKHOLE / FEDERATED / MERGE）真正的表级互斥**——这是它不可替代的地方；
2. **MyISAM 并发插入（concurrent insert）**——`THR_LOCK` 上那 5 个回调（`get_status` / `copy_status` / `update_status` / `restore_status` / `check_status`）全为它而生，也是全框架最复杂的部分；
3. **作为"本语句对表的访问意图"的跨层协议**——即便 InnoDB 不用它排队，`thr_lock_type` 仍被翻译成 InnoDB 的行锁意图（`LOCK_NONE`/`LOCK_S`/`LOCK_X`）与 `select_mode`（SKIP LOCKED / NOWAIT），并派生出 MDL 类型。

### 版本演进

| 版本 | 变化 |
|---|---|
| 3.x | THR_LOCK 诞生，作为 server 层唯一的表级锁框架（当时还没有 MDL） |
| 4.1 | InnoDB 引入 `innodb_table_locks`，AUTOCOMMIT=1 时 `LOCK TABLES` 不再取 InnoDB 表锁（注释说明原因：表锁刚拿就放，极易死锁） |
| 5.5 | **MDL 引入**，元数据与表定义稳定性从 THR_LOCK 剥离；`TL_READ_NO_INSERT` 的语义逐步移交 MDL（Bug#42147：曾导致高写负载下 `LOCK TABLE READ` 饿死，修复方式是"推给 MDL"而非改 THR_LOCK） |
| 8.0 | 路线明确为"把语义推给 MDL，THR_LOCK 退化为非事务引擎专用兼容层"：InnoDB `lock_count()` 返回 **0**；新增 `HA_NO_READ_LOCAL_LOCK` flag 让不支持 THR_LOCK 的引擎用 MDL SRO 表达 READ LOCAL；`Lock_descriptor` 把 `thr_lock_type` + `THR_SKIP/NOWAIT` 打包，`thr_lock_type` 变成纯粹意图描述 |

---

## 理论基础

### 设计思想与权衡

#### 一、两级结构：为什么拆分 `THR_LOCK` 与 `THR_LOCK_DATA`

一把"锁"被拆成**每表一个的共享主锁**与**每打开实例一个的锁实例**：前者承载队列与状态（所有线程竞争它），后者代表"某线程的一次请求"（挂在队列里）。这样拆的好处是：线程的锁请求可以自然地用侵入式链表串成队列，而主锁对象随引擎 share 生命周期存在，不必每次加锁都分配。

代价是**两级都要初始化且必须配对释放**——文件头注释专门强调 "All lock instances must be freed"，而 `reset_lock_data()` 的注释（Bug #18544）指出：`TABLE` 在 table cache 里被复用，`type` 残留会导致下次用错锁类型，所以失败路径必须逐个置 `TL_UNLOCK`。

#### 二、★ 用"排序 + 批量申请"代替死锁检测

这是本框架最重要的权衡。THR_LOCK **完全没有死锁检测**——`enum enum_thr_lock_result` 里定义了 `THR_LOCK_DEADLOCK`，`thr_lock_errno_to_mysql[]` 也把它映射到了 `ER_LOCK_DEADLOCK`，但 **thr_lock.cc 里没有任何一处返回这个值**（实际只返回 SUCCESS / ABORTED / WAIT_TIMEOUT）。

替代方案是经典的**资源定序（resource ordering）**：所有线程按同一个全局全序请求锁，就不可能形成循环等待。

```c
#define LOCK_CMP(A, B) \
  ((uchar *)(A->lock) - (uint)((A)->type) < (uchar *)(B->lock) - (uint)((B)->type))
```

排序键有两部分：主键是 `THR_LOCK*` 的**地址**（表级全局序），次键是 `- type`（**同一张表上 type 大的排前面，即写锁先于读锁**）。

代价（三处）：

1. **调用方必须遵守"一次申请全部锁"的约定**——文件头注释用 "must" 强调。任何绕过 `thr_multi_lock()` 单独调 `thr_lock()` 的代码都可能破坏全序。
2. **`MYSQL_LOCK::locks` 必须分配 2 倍空间**：`sort_locks()` 原地重排数组，但无法回头更新 `TABLE::lock_data_start`，所以复制一份副本到后半段、只对副本排序（源码注释原文说明了这一点）。
3. **失败即整体回滚，没有 victim 选择**：`thr_multi_lock()` 中某个锁失败时 `thr_multi_unlock()` 掉已获得的前 k 个就返回——没有 MDL 那样的"选受害者 + 回退 + 重开表 + 重试"。这也是 8.0 把 DDL/名字锁全交给 MDL 的原因。

#### 三、优先级编码进枚举值：整数比较代替矩阵查表

13 种 `thr_lock_type` 的**枚举值顺序就是优先级顺序**（从低到高）：

```
TL_WRITE_ALLOW_WRITE < TL_WRITE_CONCURRENT_INSERT < TL_WRITE_LOW_PRIORITY
  < TL_READ < TL_WRITE < TL_READ_HIGH_PRIORITY < TL_WRITE_ONLY
```

（文件头注释原文给出这个顺序，并注明"同一优先级内 FIFO"。）

于是全代码大量使用 `lock_type <= TL_READ_NO_INSERT`（读/写分界）、`>= TL_WRITE_ALLOW_WRITE`（写）、`> TL_WRITE_ALLOW_WRITE`（强写）这样的整数比较，**没有一个兼容性矩阵数组或查表函数**（`lock_type_compatible` 这个名字在 THR_LOCK 里不存在，只存在于 MDL）。

收益是判定的常数开销；代价是**枚举值顺序成为隐式契约**——插入一个新类型必须同时满足"语义位置"与"优先级位置"，且源码在 `enum thr_lock_type` 上方直接警告：新增类型必须同步 `sql_test.cc` 的 `lock_descriptions[]`（那里有 `static_assert(TL_WRITE_ONLY + 1 == array_elements(lock_descriptions))` 兜着）。

#### 四、写优先 + 防饥饿：两个方向的不对称设计

| 方向 | 机制 | 代码位置 |
|---|---|---|
| **写优先** | 新来的读锁看到 `write_wait` 里有"强写锁"时，**自己排队让位** | `thr_lock()` 读锁分支的 `else if` 条件 |
| **读的高优先级通道** | `TL_READ_HIGH_PRIORITY` 可无视 `write_wait` 直接获取 | 同上（`\|\| lock_type == TL_READ_HIGH_PRIORITY`） |
| **防写者饿死读者** | 连续放行写锁超过 `max_write_lock_count` 次后强制先放一批读锁 | `wake_up_waiters()` |

**这里有一处必须点明的失效场景**：`max_write_lock_count` 的默认值是 **`ULONG_MAX`**（`sys_vars.cc` 里 `DEFAULT(ULONG_MAX)`），即**防饿死机制默认关闭**。而且 `write_lock_count` 只在 `free_all_read_locks()` 里"读锁全部放完、没人再等读"时才清零——只要一直有读锁在等，计数就一直涨，但阈值是无穷大，等于不生效。所以"写优先"在生产配置下是**无对冲的**：持续的写负载确实可能饿死读。

#### 五、并发插入：用回调把策略下放给引擎

THR_LOCK 允许"一个 `TL_WRITE_CONCURRENT_INSERT` 与多个读锁并存"（文件头注释：*The lock algorithm allows one to have one TL_WRITE_CONCURRENT_INSERT lock at the same time as multiple read locks*），这是它唯一的"高级特性"。

能否并发插入**不由 THR_LOCK 判断**，而是回调给引擎的 `check_status()`。MyISAM 的实现：

```c
bool mi_check_status(void *param) {
  MI_INFO *info = (MI_INFO *)param;
  return (bool)!(info->s->state.dellink == HA_OFFSET_ERROR ||
                 (myisam_concurrent_insert == 2 && info->s->r_locks &&
                  info->s->w_locks == 1));
}
```

即"**表里没有删除留下的空洞**"才允许（新行才能安全追加到文件尾）。

代价是**锁升级逻辑散落在 4 个地方**（`thr_lock()` 两处 + `wake_up_waiters()` 两处），且升级目标是一个**全局变量** `thr_upgraded_concurrent_insert_lock`（默认 `TL_WRITE`，`--low-priority-updates` 时变成 `TL_WRITE_LOW_PRIORITY`）——升级行为依赖全局状态，不是纯函数。

### 理论溯源

- **读者-写者问题（readers-writers problem）**与优先级调度：经典教科书给出三种变体（读者优先、写者优先、公平）。本实现是**写者优先 + 一个默认关闭的公平化开关**，落点是 `thr_lock()` 读锁让位判断与 `wake_up_waiters()` 的 `write_lock_count > max_write_lock_count`。
- **资源定序（resource ordering / lock ordering）**死锁预防：Dijkstra 的"按全序获取资源"经典结论。落点是 `sort_locks()` 的 `LOCK_CMP` 与"必须经 `thr_multi_lock()` 入口"的约定——这是"用预防代替检测"的完整实例，与 InnoDB 行锁"用 wait-for graph 检测 + 回滚 victim"（见 `innodb_trx_lock.md`）形成鲜明对照。

### 算法与数据结构

**侵入式链表 + 二级指针 `last`**。四个队列都是 `st_lock_list { THR_LOCK_DATA *data, **last; }`——`last` 是"指向最后一个结点的 `next` 字段（空表时指向 `data` 字段）的二级指针"。这个设计让三种操作都成为 O(1) 且无分支：

| 操作 | 写法 | 为什么 O(1) |
|---|---|---|
| 尾插 | `(*lock->read.last) = data; data->prev = lock->read.last; lock->read.last = &data->next;` | 空表插入也无需分支 |
| 自删除 | `if (((*data->prev) = data->next)) data->next->prev = data->prev;` | `prev` 是二级指针，删队首也走同一句 |
| 整段搬迁 | `(*lock->read.last) = data; lock->read.last = lock->read_wait.last;` | `free_all_read_locks` 把 `read_wait` 整段接到 `read` 尾部 |

**插入排序**：`sort_locks()` 用插入排序，注释说明原因——"fast because almost always few locks"（锁数量通常很少，此时插入排序的常数优于快排）。

复杂度：加锁判定 O(1)（快路径）+ 队列长度 O(n)（冲突扫描，如 `has_old_lock` 遍历）、`thr_multi_lock` 排序 O(k²)（k = 锁数，通常 < 10）。

### 他库对比与演进动机

| 数据库 | 表级锁怎么做 | 与 MySQL 的差异及原因 |
|---|---|---|
| **PostgreSQL** | 没有独立的 server 层表锁框架。`LOCK TABLE ...` 与行锁共用同一个 lock manager（`LockAcquire` + `LOCKTAG`），统一管理、统一死锁检测 | MySQL 的 THR_LOCK 是**历史包袱**：3.x 时没有 MDL、没有统一 lock manager，表锁只能先自己实现一套；PG 从一开始就是统一 lock manager |
| **Oracle** | 表锁（TM lock）与行锁（TX lock）同属 enqueue 机制，统一队列 + 统一死锁检测 | 同左：Oracle 无"server 层排队锁 vs 引擎锁"的分裂 |
| **MySQL（今天）** | THR_LOCK 退化，MDL 接管元数据与大部分语义，InnoDB 用自己的 `lock_t` | 演进动机见下 |

**为什么演进成今天这样**：THR_LOCK 的三个致命缺陷在 MDL 出现后被放大——① 无死锁检测，只能靠超时；② 锁的是"已打开的对象地址"，表不存在时无法加锁（MDL 锁名字，不存在也能锁）；③ 生命周期只有语句级，`LOCK TABLES` 与事务的交互、DDL 的一致性都要另想办法。于是 5.5 引入 MDL 后，8.0 的路线是**把所有语义推给 MDL，让 THR_LOCK 退化成非事务引擎专用**——三条具体措施：`HA_NO_READ_LOCAL_LOCK`（不支持 THR_LOCK 的引擎用 MDL SRO 代替 READ LOCAL）、InnoDB `lock_count()==0`、`LOCK TABLE READ` 的语义搬到 MDL。

---

## 核心实现

### 主链路

```
open_and_lock_tables(thd, tables, flags)                    ← 所有 DML/DQL 的统一入口
  ├─ open_tables(...)                                       ① MDL 在这里获取
  │    └─ open_table() → open_table_get_mdl_lock()
  │         └─ thd->mdl_context.acquire_lock(mdl_request, timeout)
  └─ lock_tables(thd, tables, counter, flags)               ② THR_LOCK 在这里获取
       └─ mysql_lock_tables()
            ├─ lock_tables_check()      语义检查 + 断言"拿 THR_LOCK 前必须已持 MDL"
            ├─ get_lock_data()          → 每个 handler::store_lock() 收集 THR_LOCK_DATA
            ├─ lock_external()          → handler::ha_external_lock()（F_RDLCK/F_WRLCK）
            └─ thr_multi_lock()
                 ├─ sort_locks()        按 (THR_LOCK* 地址, -type) 全序排序
                 ├─ thr_lock() × N      快路径授予 / wait_for_lock() 阻塞等
                 └─ thr_lock_merge_status()   对齐 MyISAM 的 status_param
```

**先 MDL、后 THR_LOCK，顺序不可颠倒**（MDL 保证"表还在、定义没变"，THR_LOCK 才有对象可锁），`lock_tables_check()` 里那条断言在运行时强制这条不变式：

```cpp
assert(t->s->tmp_table ||
       thd->mdl_context.owns_equal_or_stronger_lock(
           MDL_key::TABLE, t->s->db.str, t->s->table_name.str,
           t->reginfo.lock_type >= TL_WRITE_ALLOW_WRITE ? MDL_SHARED_WRITE
                                                        : MDL_SHARED_READ));
```

还有一条容易忽略的顺序约束：**`lock_external()` 必须早于 `thr_multi_lock()`**。`sql/lock.cc` 文件头注释专门解释：

> Because of the new concurrent inserts, we must first get external locks before getting internal locks. If we do it in the other order, the status information is not up to date when called from the lock handler.

即并发插入的判定（`check_status`）依赖 `external_lock()` 里已建立的引擎侧状态（`r_locks/w_locks`、文件状态），顺序反了状态就是脏的。这也是"MyISAM 锁 + InnoDB 事务边界为何耦合在同一条路径上"的钥匙。

### 锁的数据结构

#### `THR_LOCK`：主锁对象

```c
struct THR_LOCK {
  LIST list{nullptr, nullptr, nullptr};
  mysql_mutex_t mutex;
  struct st_lock_list read_wait;
  struct st_lock_list read;
  struct st_lock_list write_wait;
  struct st_lock_list write;
  /* write_lock_count is incremented for write locks and reset on read locks */
  ulong write_lock_count{0};
  uint read_no_write_count{0};
  void (*get_status)(void *, int){nullptr}; /* When one gets a lock */
  void (*copy_status)(void *, void *){nullptr};
  void (*update_status)(void *){nullptr};  /* Before release of write */
  void (*restore_status)(void *){nullptr}; /* Before release of read */
  bool (*check_status)(void *){nullptr};
};
```

- **四个队列**是整个算法的核心：`read` / `read_wait` / `write` / `write_wait`。
- `write_lock_count`：连续授予写锁的计数，配合 `max_write_lock_count` 防读者饿死（**默认关闭**，见「设计思想与权衡」）。
- `read_no_write_count`：`read` 队列里 **`TL_READ_NO_INSERT` 类型读锁的数量**。它决定并发插入/WRITE_ALLOW_WRITE 还能否与现有读锁共存——这是兼容矩阵里那唯一的 `-` 的实现载体。
- 5 个回调全部为 MyISAM 并发插入服务；不支持并发插入的引擎全是 `nullptr`。
- `mutex`（PFS 名 `"THR_LOCK::mutex"`）保护本结构体；**等待时线程睡在自己的 `THD::COND_thr_lock` 上，但等的时候持有的是 `lock->mutex`**（`mysql_cond_timedwait(data->cond, &lock->mutex, ...)` 原子释放）。

#### `THR_LOCK_DATA`：锁实例与状态机

```c
struct THR_LOCK_DATA {
  THR_LOCK_INFO *owner{nullptr};
  THR_LOCK_DATA *next{nullptr}, **prev{nullptr};
  THR_LOCK *lock{nullptr};
  mysql_cond_t *cond{nullptr};
  thr_lock_type type{TL_IGNORE};
  void *status_param{nullptr};
  void *debug_print_param{nullptr};
  struct PSI_table *m_psi{nullptr};
};
```

`owner` 指向 **THD 的 `lock_info`**，`thr_lock_owner_equal()` 直接比较指针（即"是不是同一个 THD"）。`cond` 与 `type` 的组合构成状态机（源码注释明确了三种语义）：

| 状态 | `cond` | `type` | 含义 |
|---|---|---|---|
| 已解锁 | `nullptr` | `TL_UNLOCK` | — |
| 请求中（立即授予） | `nullptr` | 请求类型 | 进 `read`/`write` |
| 等待中 | `!= nullptr` | 请求类型 | 进 `read_wait`/`write_wait` |
| 已授予（被唤醒） | `nullptr`（授予方置空） | 请求类型 | `THR_LOCK_SUCCESS` |
| 被 abort | `nullptr`（abort 方置空） | `TL_UNLOCK` | `THR_LOCK_ABORTED` |
| 超时 | `!= nullptr` | 随后置 `TL_UNLOCK` | `THR_LOCK_WAIT_TIMEOUT` |

#### `MYSQL_LOCK`：一次加锁请求的打包

```c
struct MYSQL_LOCK {
  TABLE **table;
  uint table_count, lock_count;
  THR_LOCK_DATA **locks;
};
```

两个极易混淆的 count：**`table_count` = 表的数量**（`external_lock()` 用），**`lock_count` = `THR_LOCK_DATA` 的数量**（`thr_multi_lock()` 用；MyISAM 每表 1 个，MERGE/分区表 > 1，**InnoDB 恒为 0**）。

`locks` 数组分配 **2 倍**空间，源码注释原文：

> Allocating twice the number of pointers for lock data for use in thr_mulit_lock(). This function reorders the lock data, but cannot update the table values. So the second part of the array is copied from the first part immediately before calling thr_multi_lock().

#### `TABLE_SHARE` 里没有 THR_LOCK

grep 确认：`TABLE_SHARE` **没有任何 THR_LOCK 字段**（只 `#include "thr_lock.h"` 以拿到 `thr_lock_type`）。主锁对象在**引擎自己的 share** 里：

```
MyISAM:   MYISAM_SHARE { THR_LOCK lock; }        MI_INFO { THR_LOCK_DATA lock; }
MEMORY:   HP_SHARE     { THR_LOCK lock; }        HP_INFO { THR_LOCK_DATA lock; }
黑盒类:   ha_blackhole::lock / ha_tina::lock / ha_archive::lock / ha_federated::lock
InnoDB:   —— 根本没有 THR_LOCK_DATA 成员
```

### 锁类型体系

#### 13 种 `thr_lock_type`

| 类型 | 含义 |
|---|---|
| `TL_IGNORE` (-1) | "跟上次一样"，`store_lock()` 必须忽略它 |
| `TL_UNLOCK` (0) | 无锁 / 已解锁 |
| `TL_READ_DEFAULT` | **仅 parser 用**，`open_tables()` 时按 binlog 格式(SBR/RBR)与表类别解析成 `TL_READ` 或 `TL_READ_NO_INSERT` |
| `TL_READ` | 普通读锁（低优先级） |
| `TL_READ_WITH_SHARED_LOCKS` | `SELECT ... FOR SHARE` / `LOCK IN SHARE MODE` |
| `TL_READ_HIGH_PRIORITY` | `SELECT HIGH_PRIORITY`，优先级**高于 `TL_WRITE`** |
| `TL_READ_NO_INSERT` | 读锁但禁止并发插入。**现在只用于 `LOCK TABLES ... READ`** |
| `TL_WRITE_ALLOW_WRITE` | 写锁但允许别人读写；本质是"标记"而非排他锁 |
| `TL_WRITE_CONCURRENT_DEFAULT` | **仅 parser 用** → `thd->insert_lock_default` |
| `TL_WRITE_CONCURRENT_INSERT` | MyISAM 并发插入的写锁，无空洞时可与读锁共存 |
| `TL_WRITE_DEFAULT` | **仅 parser 用** → `thd->update_lock_default` |
| `TL_WRITE_LOW_PRIORITY` | `INSERT/UPDATE/DELETE LOW_PRIORITY`，优先级低于 `TL_READ` |
| `TL_WRITE` | 普通写锁（注释里字符串叫 "High priority write lock"） |
| `TL_WRITE_ONLY` | 最高优先级，**任何新请求直接返回 `THR_LOCK_ABORTED`**（不等待）。8.0 已无语句产生它 |

三个 "parser only" 的 DEFAULT 类型到执行期一定已被改写，`get_lock_data()` 里有 assert 强制这一点，对应的 `lock_descriptions[]` 槽位是 `nullptr`。

#### 兼容矩阵（源码里的 ASCII 图原文）

`thr_lock()` 读锁分支里有一张手绘的上三角矩阵（行 = 已持有，列 = 请求）：

```
           Request
          /-------
         H|++++  WRITE_ALLOW_WRITE
         e|+++-  WRITE_CONCURRENT_INSERT
         l ||||
         d ||||
           |||\= READ_NO_INSERT
           ||\ = READ_HIGH_PRIORITY
           |\  = READ_WITH_SHARED_LOCKS
           \   = READ

        + = Request can be satisfied.
        - = Request cannot be satisfied.
```

**只有 `TL_WRITE_CONCURRENT_INSERT` 与 `TL_READ_NO_INSERT` 冲突**，其余全部兼容（连"已有写锁"时也能给读锁——所以这张图是"有写锁时能否再给读锁"的特化矩阵，不是全矩阵）。

紧接着的注释解释了一个历史坑：

> READ_NO_INSERT and WRITE_ALLOW_WRITE should in principle be incompatible. Before this could have caused starvation of LOCK TABLE READ in InnoDB under high write load. However now READ_NO_INSERT is only used for LOCK TABLES READ and this statement is handled by the MDL subsystem. See Bug#42147.

### 加锁主链路：`thr_lock()`

#### 读锁分支

```c
if ((int)lock_type <= (int)TL_READ_NO_INSERT) {
  /* Request for READ lock */
  if (lock->write.data) {
    /* ... 上面的兼容矩阵 ... */
    if (thr_lock_owner_equal(data->owner, lock->write.data->owner) ||
        (lock->write.data->type < TL_WRITE_CONCURRENT_INSERT) ||
        ((lock->write.data->type == TL_WRITE_CONCURRENT_INSERT) &&
         ((int)lock_type <= (int)TL_READ_HIGH_PRIORITY))) { /* Already got a write lock */
      (*lock->read.last) = data;          /* Add to running FIFO */
      data->prev = lock->read.last;
      lock->read.last = &data->next;
      if (lock_type == TL_READ_NO_INSERT) lock->read_no_write_count++;
      if (lock->get_status) (*lock->get_status)(data->status_param, 0);
      locks_immediate++;
      goto end;
    }
    if (lock->write.data->type == TL_WRITE_ONLY) {
      /* We are not allowed to get a READ lock in this case */
      data->type = TL_UNLOCK;
      result = THR_LOCK_ABORTED; /* Can't wait for this one */
      goto end;
    }
  } else if (!lock->write_wait.data ||
             lock->write_wait.data->type <= TL_WRITE_LOW_PRIORITY ||
             lock_type == TL_READ_HIGH_PRIORITY ||
             has_old_lock(lock->read.data, data->owner)) /* Has old read lock */
  {                                     /* No important write-locks */
    (*lock->read.last) = data;          /* Add to running FIFO */
    ...
    locks_immediate++;
    goto end;
  }
  /*
    We're here if there is an active write lock or no write
    lock but a high priority write waiting in the write_wait queue.
    In the latter case we should yield the lock to the writer.
  */
  wait_queue = &lock->read_wait;
}
```

逐段：

1. **有活跃写锁时能否再给读锁**：同线程 / 已有写锁只是 `TL_WRITE_ALLOW_WRITE`（与一切兼容）/ 已有 `CONCURRENT_INSERT` 且请求不是 `READ_NO_INSERT`——满足则直接授予。
2. `TL_WRITE_ONLY` 存在且不是自己持有 → `THR_LOCK_ABORTED`，**不等待**。
3. **无活跃写锁的快路径**（`else if`）：没有排队写锁 / 排队写锁是"弱写"（`<= TL_WRITE_LOW_PRIORITY`，优先级都 ≤ READ）/ 请求是 `TL_READ_HIGH_PRIORITY` / **本线程已在 `read` 队列里有读锁**（`has_old_lock`，避免自己等自己）。
4. **都不满足 → 进 `read_wait` 等待**。这就是写优先的实现：**新来的读锁看到 `write_wait` 里有 `TL_WRITE` 级以上的请求时要让位**。

#### 写锁分支与四处锁升级

```c
{
  if (lock_type == TL_WRITE_CONCURRENT_INSERT && !lock->check_status)
    data->type = lock_type = thr_upgraded_concurrent_insert_lock;      // 升级点 ①

  if (lock->write.data) { /* If there is a write lock */
    if (lock->write.data->type == TL_WRITE_ONLY) {
      if (!thr_lock_owner_equal(data->owner, lock->write.data->owner)) {
        data->type = TL_UNLOCK;
        result = THR_LOCK_ABORTED;
        goto end;
      }
    }
    /* ... 大段注释：同一表的请求按 type 降序排列，故"本线程已持有写锁"可直接再给 ... */
    if ((lock_type == TL_WRITE_ALLOW_WRITE && !lock->write_wait.data &&
         lock->write.data->type == TL_WRITE_ALLOW_WRITE) ||
        has_old_lock(lock->write.data, data->owner)) {
      (*lock->write.last) = data; /* Add to running fifo */
      ...
      locks_immediate++;
      goto end;
    }
  } else {
    if (!lock->write_wait.data) { /* no scheduled write locks */
      bool concurrent_insert = false;
      if (lock_type == TL_WRITE_CONCURRENT_INSERT) {
        concurrent_insert = true;
        if ((*lock->check_status)(data->status_param)) {                // 升级点 ②
          concurrent_insert = false;
          data->type = lock_type = thr_upgraded_concurrent_insert_lock;
        }
      }
      if (!lock->read.data || (lock_type <= TL_WRITE_CONCURRENT_INSERT &&
                               ((lock_type != TL_WRITE_CONCURRENT_INSERT &&
                                 lock_type != TL_WRITE_ALLOW_WRITE) ||
                                !lock->read_no_write_count))) {
        (*lock->write.last) = data; /* Add as current write lock */
        ...
        locks_immediate++;
        goto end;
      }
    }
  }
  wait_queue = &lock->write_wait;
}
result = wait_for_lock(wait_queue, data, owner, false, lock_wait_timeout);
```

- **升级点 ①**：请求并发插入但该 `THR_LOCK` **没有 `check_status` 回调**（引擎不支持并发插入）→ 改写成 `thr_upgraded_concurrent_insert_lock`。
- **升级点 ②**：有回调但 `check_status()` 说"现在不行"（MyISAM：表里有删除留下的空洞）→ 同样改写。
- 已有写锁时能再给写锁的两种情形：新请求是 `TL_WRITE_ALLOW_WRITE` 且已有也是它且无 `write_wait`（ALLOW_WRITE 彼此兼容，可叠加）；或**本线程已持有写锁**（`has_old_lock`）——对应 `INSERT INTO t1 VALUES(f1())` 里 `f1()` 又 UPDATE t1 的场景，源码注释用这个例子说明了为什么必须支持。
- 无写锁无 `write_wait` 时的授予条件：没有活跃读锁，**或**是"弱写"且（`read_no_write_count == 0` 或不是 `CONCURRENT_INSERT`/`ALLOW_WRITE`）。`read_no_write_count` 就是在这里把关。

另外两个升级点在 `wake_up_waiters()` 里（等到真要授予时再查一次 `check_status`）。

#### `wait_for_lock()`：等待、授予、超时、abort

```c
static enum enum_thr_lock_result wait_for_lock(struct st_lock_list *wait,
                                               THR_LOCK_DATA *data,
                                               THR_LOCK_INFO *owner,
                                               bool in_wait_list,
                                               ulong lock_wait_timeout) {
  ...
  if (!in_wait_list) {
    (*wait->last) = data; /* Wait for lock */
    data->prev = wait->last;
    wait->last = &data->next;
  }
  locks_waited++;
  data->cond = owner->suspend;      /* Set up control struct to allow others to abort locks */
  enter_cond_hook(nullptr, data->cond, &data->lock->mutex,
                  &stage_waiting_for_table_level_lock, &old_stage, ...);
  set_timespec(&wait_timeout, lock_wait_timeout);
  while (!is_killed_hook(nullptr) || in_wait_list) {
    int rc = mysql_cond_timedwait(data->cond, &data->lock->mutex, &wait_timeout);
    if (data->cond == nullptr) break;                 /* 已授予（或被 abort） */
    if (is_timeout(rc)) { result = THR_LOCK_WAIT_TIMEOUT; break; }
  }
  if (data->cond || data->type == TL_UNLOCK) {
    if (data->cond) {                                  /* aborted or timed out */
      if (((*data->prev) = data->next))                /* remove from wait-list */
        data->next->prev = data->prev;
      else
        wait->last = data->prev;
      data->type = TL_UNLOCK;
      wake_up_waiters(data->lock);                     /* 别把自己挡住的队首饿死 */
    }
  } else { result = THR_LOCK_SUCCESS; ... }
  mysql_mutex_unlock(&data->lock->mutex);
  exit_cond_hook(nullptr, &old_stage, ...);
  return result;
}
```

要点：

- **`cond` 具有双重身份**：既是等待的条件变量，又是"是否被授予"的标志——授予方只做 `data->cond = nullptr; mysql_cond_signal(cond)` 两件事，等待方醒来靠 `cond == nullptr` 判断"我被授予了"。没有单独的 granted 标志位。
- **注释明确了检查顺序不能反**（"Order of checks below is important to not report about timeout if the predicate is true"）：先判 `cond == nullptr`（可能同时超时又被授予），再判超时——否则会误报超时。
- 超时/被 kill 后**自己把自己摘出队列**并调 `wake_up_waiters()`，避免自己挡住别人。
- `enter_cond_hook` 把线程 stage 设为 `stage_waiting_for_table_level_lock`——这就是 `SHOW PROCESSLIST` 里的 **`Waiting for table level lock`**。
- `before_lock_wait`/`after_lock_wait` 回调由 `thr_set_lock_wait_callback()` 注册，供线程池/连接调度感知"有线程要开始等表锁了"。

#### `wake_up_waiters()`：调度核心

这是全框架最复杂的函数。三种大情形：

**(1) 表完全空闲（无写锁、无读锁）**——优先放行 `write_wait` 队首：

```c
if (data && (data->type != TL_WRITE_LOW_PRIORITY || !lock->read_wait.data ||
             lock->read_wait.data->type < TL_READ_HIGH_PRIORITY)) {
  if (lock->write_lock_count++ > max_write_lock_count) {     /* 防饿死 */
    lock->write_lock_count = 0;
    if (lock->read_wait.data) { free_all_read_locks(lock, false); goto end; }
  }
  for (;;) {
    /* 从 write_wait 搬到 write */
    if (data->type == TL_WRITE_CONCURRENT_INSERT &&
        (*lock->check_status)(data->status_param))
      data->type = TL_WRITE;                                 /* 升级点 ③ */
    {
      mysql_cond_t *cond = data->cond;
      data->cond = nullptr;    /* Mark thread free */
      mysql_cond_signal(cond); /* Start waiting thread */
    }
    if (data->type != TL_WRITE_ALLOW_WRITE || !lock->write_wait.data ||
        lock->write_wait.data->type != TL_WRITE_ALLOW_WRITE) break;
    data = lock->write_wait.data;   /* ALLOW_WRITE 可批量放行 */
  }
  if (data->type >= TL_WRITE_LOW_PRIORITY) goto end;         /* 排他写，不再放读锁 */
}
```

- `TL_WRITE_LOW_PRIORITY` 只有在"无读锁等待"或"等的读锁不是 HIGH_PRIORITY"时才被放行 → **LOW_PRIORITY 写让位于读**。
- 防饿死检查（默认不生效，见理论基础）。
- 放行后若类型是 `TL_WRITE`(11) / `TL_WRITE_ONLY`(12) 就 `goto end`——排他写不放读锁；否则（`ALLOW_WRITE` / `CONCURRENT_INSERT`）继续放读锁。

**(2) 有活跃读锁、无写锁**——只放行"弱写"：

```c
} else if (data && (lock_type = data->type) <= TL_WRITE_CONCURRENT_INSERT &&
           ((lock_type != TL_WRITE_CONCURRENT_INSERT &&
             lock_type != TL_WRITE_ALLOW_WRITE) ||
            !lock->read_no_write_count)) {
  if (lock_type == TL_WRITE_CONCURRENT_INSERT &&
      (*lock->check_status)(data->status_param)) {
    data->type = TL_WRITE;                                   /* 升级点 ④ */
    if (lock->read_wait.data) free_all_read_locks(lock, false);
    goto end;
  }
  /* 把弱写锁放进 write 队列，然后同时放行读锁 */
  ...
  if (lock->read_wait.data)
    free_all_read_locks(lock, (lock_type == TL_WRITE_CONCURRENT_INSERT ||
                               lock_type == TL_WRITE_ALLOW_WRITE));
}
```

即文件头注释说的"一个 `TL_WRITE_CONCURRENT_INSERT` + 多个读锁"并存。

**(3) 有活跃写锁**：整块跳过——写锁不释放就不会有人被唤醒。

#### `free_all_read_locks()`：批量放行 + 一处回退

```c
static inline void free_all_read_locks(THR_LOCK *lock, bool using_concurrent_insert) {
  THR_LOCK_DATA *data = lock->read_wait.data;
  /* O(1) 整段搬迁：read_wait 接到 read 尾部 */
  (*lock->read.last) = data;
  data->prev = lock->read.last;
  lock->read.last = lock->read_wait.last;
  lock->read_wait.last = &lock->read_wait.data;       /* 清空 read_wait */
  do {
    mysql_cond_t *cond = data->cond;
    if ((int)data->type == (int)TL_READ_NO_INSERT) {
      if (using_concurrent_insert) {
        /* 不能放行：从 read 摘下、塞回 read_wait */
        if (((*data->prev) = data->next)) data->next->prev = data->prev;
        else lock->read.last = data->prev;
        *lock->read_wait.last = data;
        data->prev = lock->read_wait.last;
        lock->read_wait.last = &data->next;
        continue;
      }
      lock->read_no_write_count++;
    }
    data->cond = nullptr;
    mysql_cond_signal(cond);
  } while ((data = data->next));
  *lock->read_wait.last = nullptr;
  if (!lock->read_wait.data) lock->write_lock_count = 0;    /* 复位点 */
}
```

**"先整体搬迁再逐个回退"**是个有意思的选择：绝大部分读锁能放行，只有 `TL_READ_NO_INSERT`（且正在授予并发插入时）需要塞回队列——先乐观搬迁再回退，比为每个元素先判断再搬迁更快。

`if (!lock->read_wait.data) lock->write_lock_count = 0;` 是 `write_lock_count` 的唯一复位点。

### 批量申请与死锁规避

```c
static void sort_locks(THR_LOCK_DATA **data, uint count) {
  /* Sort locks with insertion sort (fast because almost always few locks) */
  for (pos = data + 1, end = data + count; pos < end; pos++) {
    tmp = *pos;
    if (LOCK_CMP(tmp, pos[-1])) {
      prev = pos;
      do { prev[0] = prev[-1]; } while (--prev != data && LOCK_CMP(tmp, prev[-1]));
      prev[0] = tmp;
    }
  }
}

enum enum_thr_lock_result thr_multi_lock(THR_LOCK_DATA **data, uint count,
                                         THR_LOCK_INFO *owner,
                                         ulong lock_wait_timeout) {
  if (count > 1) sort_locks(data, count);
  for (pos = data, end = data + count; pos < end; pos++) {
    enum enum_thr_lock_result result = thr_lock(*pos, owner, (*pos)->type, lock_wait_timeout);
    if (result != THR_LOCK_SUCCESS) {           /* Aborted */
      thr_multi_unlock(data, (uint)(pos - data));
      return result;
    }
  }
  thr_lock_merge_status(data, count);
  return THR_LOCK_SUCCESS;
}
```

注意 `thr_multi_unlock(data, pos - data)` 只回滚**已成功的前 k 个**，且此时数组已排序——回滚顺序与获取顺序相同。而常规解锁走的是**前半段未排序的原始副本**。

#### `thr_lock_merge_status()`：一段自嘲为 "crutch" 的补丁

```cpp
/**
  Ensure that all locks for a given table have the same status_param.

  This is a MyISAM and possibly Maria specific crutch. ... One thing MyISAM
  doesn't do is to ensure that when the same table is opened twice in a
  connection all instances share the same status_param. This is necessary,
  however: ... unless this is done, myisam_share will always get updated from
  the last unlocked instance (in mi_update_status()), and when this instance
  was not the one that was used to update data, records may be lost.
*/
```

MyISAM 把"数据文件长度/记录数"存在 `status_param` 里：加锁时 `mi_get_status()` 从全局 share 拷一份到本地，解锁时 `mi_update_status()` 写回。若同一连接把同一张表打开多次且实例间不同步，**最后解锁的那个实例（可能没参与写）会覆盖全局 share，导致记录丢失**。于是 server 层在 `thr_multi_lock()` 末尾统一对齐到"最后一个锁实例（写锁优先）"的 `status_param`——MyISAM 的实现就一行 `((MI_INFO*)to)->state = &((MI_INFO*)from)->save_state;`。

### 与 InnoDB 的关系：空壳，但不是无用的空壳

#### 铁证：`lock_count()` 返回 0

```cpp
/** Returns number of THR_LOCK locks used for one instance of InnoDB table.
 InnoDB no longer relies on THR_LOCK locks so 0 value is returned.
 Instead of THR_LOCK locks InnoDB relies on combination of metadata locks
 (e.g. for LOCK TABLES and DDL) and its own locking subsystem.
 Note that even though this method returns 0, SQL-layer still calls
 "::store_lock()", "::start_stmt()" and "::external_lock()" methods for InnoDB
 tables. */
uint ha_innobase::lock_count(void) const { return 0; }
```

基类默认是 `return 1`。而 `ha_innobase::store_lock()` 的最后一行是 `return (to);`——**没有 `*to++ = &lock`**。对比 MyISAM：

```cpp
THR_LOCK_DATA **ha_myisam::store_lock(THD *, THR_LOCK_DATA **to,
                                      enum thr_lock_type lock_type) {
  if (lock_type != TL_IGNORE && file->lock.type == TL_UNLOCK)
    file->lock.type = lock_type;
  *to++ = &file->lock;      // ← 真正把 THR_LOCK_DATA 交给 server
  return to;
}
```

于是 InnoDB 表的 `sql_lock->lock_count == 0`，`thr_multi_lock()` 拿到空数组——**排队、冲突检测、等待、优先级这些 THR_LOCK 的核心机制对 InnoDB 一行都不起作用**。

#### 但 `thr_lock_type` 仍被翻译成 InnoDB 的行锁意图

`ha_innobase::store_lock()` 的 if-else 链把 `thr_lock_type` 映射到 `m_prebuilt->select_lock_type`：

```cpp
} else if ((lock_type == TL_READ && in_lock_tables) ||
           (lock_type == TL_READ_HIGH_PRIORITY && in_lock_tables) ||
           lock_type == TL_READ_WITH_SHARED_LOCKS ||
           lock_type == TL_READ_NO_INSERT ||
           (lock_type != TL_IGNORE && sql_command != SQLCOM_SELECT)) {
  /* LOCK TABLES READ / SELECT FOR SHARE / INSERT...SELECT / 非 SELECT 一律锁定读 */
  if (sql_command == SQLCOM_CHECKSUM ||
      (trx->skip_gap_locks() && (lock_type == TL_READ || lock_type == TL_READ_NO_INSERT) &&
       (sql_command == SQLCOM_INSERT_SELECT || sql_command == SQLCOM_REPLACE_SELECT ||
        sql_command == SQLCOM_UPDATE || sql_command == SQLCOM_CREATE_TABLE))) {
    m_prebuilt->select_lock_type = LOCK_NONE;      /* 半一致性读路径 */
  } else {
    m_prebuilt->select_lock_type = LOCK_S;
  }
} else if (lock_type != TL_IGNORE) {
  /* We set possible LOCK_X value in external_lock, not yet here */
  m_prebuilt->select_lock_type = LOCK_NONE;
}

/* SKIP LOCKED / NOWAIT 也走这条通道 */
if (lock_type != TL_IGNORE) {
  switch (table->pos_in_table_list->lock_descriptor().action) {
    case THR_SKIP:   m_prebuilt->select_mode = SELECT_SKIP_LOCKED; break;
    case THR_NOWAIT: m_prebuilt->select_mode = SELECT_NOWAIT; break;
    default:         m_prebuilt->select_mode = SELECT_ORDINARY; break;
  }
}
```

这就是 8.0 里 `Lock_descriptor::action`（`THR_SKIP`/`THR_NOWAIT`）的真实用途——把 SQL 语法的 `SKIP LOCKED` / `NOWAIT` 传到 InnoDB。

而真正的表锁在 `external_lock()` 里，且条件苛刻（源码注释）：

> Starting from 4.1.9, no InnoDB table lock is taken in LOCK TABLES if AUTOCOMMIT=1. It does not make much sense to acquire an InnoDB table lock if it is released immediately at the end of LOCK TABLES, and InnoDB's table locks in that case cause VERY easily deadlocks.

条件：`sql_command == SQLCOM_LOCK_TABLES && innodb_table_locks && select_lock_type != LOCK_NONE`（+ autocommit 豁免）。

#### 三层锁的完整分工（InnoDB 表）

| 场景 | MDL | THR_LOCK | InnoDB `lock_t` |
|---|---|---|---|
| 普通 `SELECT`（autocommit） | `MDL_SHARED_READ` | **空壳**（`lock_count()==0`） | 无锁，一致性读 |
| `SELECT ... FOR SHARE` | SR | 空壳，翻译成 `LOCK_S` | 行级 S 锁 |
| `SELECT ... FOR UPDATE` / `UPDATE` / `DELETE` | `MDL_SHARED_WRITE` | 空壳，`external_lock` 里可升 `LOCK_X` | 行级 X 锁 + 表级 **IX** |
| `LOCK TABLE t READ` | SR **升级为 SRO**（因 `HA_NO_READ_LOCAL_LOCK`） | 空壳 | 表级 **S** |
| `LOCK TABLE t WRITE` | SW | 空壳 | 表级 **X** + IX |
| DDL | `MDL_EXCLUSIVE` | 无关 | 表级 X |

一句话：**MyISAM 的 `LOCK TABLES`/DML 互斥完全由 THR_LOCK 提供（它是真锁）；InnoDB 的 `LOCK TABLES` 语义 = MDL + InnoDB `lock_t` 表锁，THR_LOCK 只是保留的意图通道。**

中间形态也值得一提：BLACKHOLE / FEDERATED **有** `THR_LOCK_DATA`，但在非 LOCK TABLES 场景把写锁降级：

```cpp
if ((lock_type >= TL_WRITE_CONCURRENT_INSERT && lock_type <= TL_WRITE) &&
    !thd_in_lock_tables(thd))
  lock_type = TL_WRITE_ALLOW_WRITE;
/* INSERT INTO t1 SELECT FROM t2：把 TL_READ_NO_INSERT 降为 TL_READ，
   否则会与 TL_WRITE_ALLOW_WRITE 冲突，阻塞对 t2 的插入 */
if (lock_type == TL_READ_NO_INSERT && !thd_in_lock_tables(thd))
  lock_type = TL_READ;
```

TEMPTABLE 引擎更直接：`store_lock()` 返回 `nullptr`，连 `to` 都不推进。

### 跨系统的死锁：需要手工打断

MDL 与 THR_LOCK 是两套独立系统，它们之间会形成跨系统死锁，而 THR_LOCK 侧没有检测能力。解法是**外部打断**——`thr_abort_locks_for_thread()`：

```c
void thr_abort_locks_for_thread(THR_LOCK *lock, my_thread_id thread_id) {
  mysql_mutex_lock(&lock->mutex);
  for (data = lock->read_wait.data; data; data = data->next) {
    if (data->owner->thread_id == thread_id) {
      data->type = TL_UNLOCK;       /* Mark killed */
      mysql_cond_signal(data->cond);
      data->cond = nullptr;         /* Removed from list */
      if (((*data->prev) = data->next)) data->next->prev = data->prev;
      else lock->read_wait.last = data->prev;
    }
  }
  /* ... write_wait 同样处理 ... */
  wake_up_waiters(lock);
  mysql_mutex_unlock(&lock->mutex);
}
```

`sql/mdl.h` 的注释点明了典型场景：MDL 的 SNRW 锁升级为 X 锁时，若对方正卡在 THR_LOCK 等待上，必须用这个机制打断。上层入口是 `mysql_lock_abort_for_thread()`，由 `THD::awake()` / `close_cached_tables()` 路径调用。**这是"两套锁系统并存"的直接代价。**

---

## ★ 本机制里的工程实现技法

### 一、侵入式链表 + 二级指针：三种操作全 O(1) 无分支

`st_lock_list { THR_LOCK_DATA *data, **last; }` 与 `THR_LOCK_DATA::prev`（**二级**指针）的组合，让"尾插/自删除/整段搬迁"都变成常数时间且无分支（代码见「算法与数据结构」与 `free_all_read_locks`）。

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| `last` 二级指针 | 四个队列的尾插 | 空表插入无需分支 | 初值必须指向 `data` 字段本身（`&lock->read.data`），初始化漏了就崩 |
| `prev` 二级指针 | 队列自删除 | 删队首/中间同一句代码 | 读代码时 `(*data->prev) = data->next` 不易一眼看懂 |
| 整段链表搬迁 | `free_all_read_locks` | O(1) 把 `read_wait` 全量转入 `read` | 搬完还要逐个 signal，且 `READ_NO_INSERT` 要塞回队列（先乐观后回退） |

### 二、资源定序代替死锁检测：教科书结论的最简落地

教科书原型是"给所有资源编号，按序获取"；本实现的落地有两处简化：

| 维度 | 教科书原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 排序键 | 资源编号（需维护一张全局资源表） | **直接用 `THR_LOCK*` 的地址**（`- type` 作次键让同表写锁优先） | 不需要资源表，代价是排序键依赖内存布局（地址），不可移植到"跨进程锁" |
| 执行方式 | 每个线程自己按序申请 | 由 `thr_multi_lock()` 统一 `sort_locks()` 后申请 | 调用方**必须**走这个入口，否则全序被破坏且无运行时检测 |
| 违反后果 | 可能死锁 | 只能靠 `lock_wait_timeout` 超时暴露 | 无 victim 选择、无重试，失败即整体回滚 |

### 三、优先级编码进枚举值：用整数比较代替查表

13 种类型的枚举顺序即优先级顺序，`lock_type <= TL_READ_NO_INSERT` 这类比较贯穿全文件。

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| 枚举值即优先级 | 读写判定、强弱写判定 | O(1) 比较，无矩阵数组 | 新增类型必须同时摆对"语义位置"与"优先级位置"，插错位置会静默改变行为 |
| `static_assert` 兜底 | `lock_descriptions[]` 长度校验 | 编译期发现漏改字符串表 | 只兜住字符串表，兜不住"语义位置" |
| 全局可变升级目标 | `thr_upgraded_concurrent_insert_lock` | `--low-priority-updates` 可运行时改升级行为 | 升级不是纯函数，依赖全局变量，读代码时不易看出 |

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `max_write_lock_count` | **`ULONG_MAX`** | Global | 连续放行这么多次写锁后强制放一批读锁；**默认无穷大 = 防饿死机制关闭** |
| `low_priority_updates` | OFF | Global/Session | 影响 `thr_upgraded_concurrent_insert_lock`（并发插入锁的升级目标） |
| `lock_wait_timeout` | 31536000（1 年） | Global/Session | THR_LOCK 等待超时（`wait_for_lock` 的 `mysql_cond_timedwait`） |

| 状态变量 | 说明 |
|---|---|
| `Table_locks_immediate` | 立即获得表锁的次数（`locks_immediate`，在 `thr_lock()` 的 4 个 `goto end` 快路径累加） |
| `Table_locks_waited` | **进入等待的次数**（`locks_waited`，仅 1 处累加：进入等待队列后、真正睡眠前）——统计的是次数不是时长，也不是被阻塞的查询数 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 谁在等表锁、等了多久 | SQL | `performance_schema.processlist` / `SHOW PROCESSLIST` 的 `State: Waiting for table level lock` |
| 累计等了多少次 | SQL | `SHOW GLOBAL STATUS LIKE 'Table_locks_%'` |
| 每张表的表锁等待汇总 | PFS | `performance_schema.table_lock_waits_summary_by_table` |
| 单次等待的 instrument 记录 | PFS | `wait/lock/table/sql/handler` 事件（`thr_lock()` 的 `MYSQL_START_TABLE_LOCK_WAIT`） |
| 当前谁持有表锁（间接） | SQL | `SHOW OPEN TABLES WHERE In_use > 0` |

**⚠️ 两个关键前提，否则会看错**：

1. `Waiting for table level lock` **只可能来自 THR_LOCK**（MyISAM 等）。InnoDB 行锁等待要看 `data_locks` / `SHOW ENGINE INNODB STATUS`，MDL 等待是 `Waiting for table metadata lock`——三者 state 字符串不同，可据此区分。
2. **InnoDB 表在 `table_lock_waits_summary_by_table` 里几乎不会有记录**（因为 `lock_count()==0`，没有 `THR_LOCK_DATA`，埋点无对象）。这张 PFS 表基本是 **MyISAM 专用**。唯一例外是 `PFS_TL_READ_EXTERNAL`/`WRITE_EXTERNAL`，那来自 `ha_external_lock()`，与 thr_lock.cc 无关。

---

## Misc

### 面向二次开发

**扩展点**：新增一种 `thr_lock_type` 要改这几处——`enum thr_lock_type`（**位置即优先级，插错会静默改行为**）、`sql_test.cc` 的 `lock_descriptions[]`（有 `static_assert` 兜底）、`thr_lock()` 里所有依赖整数比较的分支、`storage/perfschema` 的 `lock_flags_to_lock_type()` 映射、以及每个引擎的 `store_lock()`。**注意 `TL_WRITE_ONLY` 在 8.0 已无任何产生者**——它是历史遗留，新增类型前先看清楚它为什么没被删。

**坑与已知缺陷**（源码证据）：

1. **无死锁检测**：`THR_LOCK_DEADLOCK` 定义了但从不返回，只能靠排序预防 + 超时兜底。
2. **防饿死默认关闭**：`max_write_lock_count` 默认 `ULONG_MAX`。
3. **只能等不能回退**：`thr_multi_lock()` 失败即整体回滚，没有 victim 选择 + 重试（对比 MDL）。
4. **`thr_lock_merge_status()` 自认是 "crutch"**：MyISAM 不做 status_param 同步会在 `mi_update_status()` 丢记录。
5. **`check_status()` 的隐式耦合**：写锁分支里 `(*lock->check_status)(...)` 的解引用安全，依赖上面那句"无回调就把类型改写掉"——两处代码相距很近但逻辑上是隐式契约。
6. **MDL ↔ THR_LOCK 跨系统死锁需手工打断**（`thr_abort_locks_for_thread`）。
7. **`LOCK TABLES` 与事务互斥**：`case SQLCOM_LOCK_TABLES` 先 `trans_commit_implicit()` 再释放 transactional MDL；`LOCK TABLES` 模式下 `lock_schema_name()` 直接报 `ER_LOCK_OR_ACTIVE_TRANSACTION`。
8. **`Table_locks_waited` 计数不准**：mysql-test 的 `lock_multi.test` 里有专门注释提到 Bug#30331。
9. **`get_lock_data()` 会多分配内存**：注释说明 MERGE 表 `store_lock()` 可能返回比 `lock_count()` 更少的锁，"Now we may allocate too much, but better safe than memory overrun"。

**社区边界澄清**：`thr_lock_merge_status()` 的注释写 "MyISAM and possibly Maria specific crutch"——MariaDB 在此基础上的改动属 MariaDB 分支特性，社区版 8.0.39 的行为以本篇为准。

### 易混淆概念

- **THR_LOCK 的"表锁" vs InnoDB 的"表锁"**：前者是 server 层排队锁（MyISAM 用），后者是 `lock_t` 的 `LOCK_TABLE`（IS/IX/S/X，InnoDB 用）。`LOCK TABLES` 在两种引擎上走的是完全不同的实现——**InnoDB 上 THR_LOCK 是空壳**。
- **`MYSQL_LOCK::table_count` vs `lock_count`**：前者是"表的数量"，后者是"THR_LOCK_DATA 的数量"，**命名很坑**。InnoDB 表 `lock_count` 恒为 0 但 `table_count` 不为 0。
- **`LOCK TABLES`（server 语义）vs `lock_t` 表锁（InnoDB）**：MySQL 语法只有一个 `LOCK TABLES`，但落到 MyISAM 是 THR_LOCK、落到 InnoDB 是 MDL + `lock_t`。
- **"写优先"不等于"写者不会饿死读者"**：防饿死开关默认关闭。

---

## 参考

**论文 / 经典算法**
- Courtois, Heymans, Parnas. *Concurrent Control with "Readers" and "Writers"*. CACM 1971.（读者-写者问题；本实现是"写者优先"变体，落点在 `thr_lock()` 读锁让位与 `wake_up_waiters()`）
- Dijkstra 的资源定序死锁预防（*Cooperating Sequential Processes*, 1965）。（落点：`sort_locks()` 的 `LOCK_CMP` + "必须经 `thr_multi_lock()` 入口"约定）

**官方文档**
- *MySQL 8.0 Reference Manual → Internal Locking Methods / Table-Level Locks*
- *MySQL 8.0 Reference Manual → `LOCK TABLES` and `UNLOCK TABLES` Statements*

**相关文档**
- MDL 框架（18 namespace、优先级矩阵、死锁检测）见 [`mdl.md`](mdl.md)——本篇多次引用它来对比"THR_LOCK 没有的能力"
- InnoDB 表锁与行锁（`lock_t`、意向锁、AUTOINC、谓词锁）见 [`innodb_trx_lock.md`](innodb_trx_lock.md)
- 全局锁二件套（FTWRL / 备份锁）见 [`global_lock.md`](global_lock.md)
- 锁的全量盘点与本目录归属判据见 [`../README.md`](../README.md)
