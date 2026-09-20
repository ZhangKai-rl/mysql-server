# MDL 元数据锁深度解析

> 基于 MySQL 8.0.39 源码，涵盖 MDL 的 18 个 namespace 与两套策略（scoped/object）、`MDL_lock`/`MDL_ticket`/`MDL_context`/`MDL_request` 四件套、**fast path 与 obtrusive/unobtrusive 的延迟精算**、双兼容矩阵 + 4 张优先级矩阵、`can_grant_lock` 三级短路、`reschedule_waiters` 公平调度、wait-for graph 死锁检测（BFS+DFS、权重 victim）、锁升级（SU→SNW→X）与 online/instant DDL、用户级锁 GET_LOCK。
>
> **边界**：本篇讲 server 层 MDL；InnoDB 行锁/间隙锁/死锁检测见 [`innodb_trx_lock.md`](innodb_trx_lock.md)；全局锁二件套（FTWRL / 备份锁）见 [`global_lock.md`](global_lock.md)——它们是 MDL 的 GLOBAL/COMMIT/BACKUP_LOCK namespace 用法；server 层表锁 THR_LOCK 见 [`thr_lock.md`](thr_lock.md)。

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

MDL 是 server 层保护**表结构/元数据**的锁，与引擎的行锁完全独立。它由 `sql/mdl.cc` 统一管理，一个 DML 会同时持有：MDL 的 SW（元数据层，server）+ InnoDB 行锁（数据层，引擎）。

一处容易误解的实现细节：**`MDL_lock` 类不在 `mdl.h` 里，而是定义在 `mdl.cc` 内部**（头文件只有前向声明）。源码注释把它类比为 "an MDL subsystem's version of TABLE_SHARE"——对外只暴露 `MDL_request`/`MDL_ticket`/`MDL_context` 三个句柄，锁对象本身是子系统私有的。

### 用途

解决"DDL 与 DML 并发"：一个会话正在 `SELECT` 某表时，另一个会话 `DROP TABLE`——若 DDL 立刻执行，正在扫描的行可能被物理删除。MDL 让 DML/查询持共享锁、DDL 持排他锁，二者互斥。

**5.5.3 引入 MDL 时具体要解决的是两个问题**（内核月报《跟踪 Metadata lock》的梳理，设计背景见 WL#3726/WL#4284）：

1. **破坏 RR 隔离**：repeatable read 下多次 `SELECT` 的执行过程中，其他会话的 DDL 会让前后两次结果不一致——而隔离级别本该保证这一点。
2. **binlog 顺序错乱**（Bug #989）：对表的 DML 过程中穿插 DDL，导致 binlog 里 event 的顺序在备库重放的结果与主库不一致。

关键动作是 **5.5.3 起把 MDL 持有周期从语句升级为事务**（`MDL_TRANSACTION` 要等事务结束才释放）——解决了上述两问，但也埋下了"autocommit=off 时长事务阻塞 DDL"的经典运维问题（见「可观测性」的案例）。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.5.3 | 引入 MDL（设计见 WL#3726 / WL#4284），取代旧的 `.frm` 文件 + table cache 锁；**持有周期从语句升级为事务** |
| 5.6.8 | MDL map 按 key 哈希**分区**——解决全局锁竞争（Bug #66473），引入 `metadata_locks_hash_instances`（默认 8，每分区独立 mutex + hash） |
| 5.7 | **online DDL**（`ALGORITHM=INPLACE`）：只在准备/提交阶段短暂持 X；**fast path 无锁化**（WL#7304）——`m_fast_path_state` 位图 + 原子 CAS，把高频 DML 加锁的成本从 mutex 降到一条指令；5.7.6 起 PFS 新增 `metadata_locks` 表；**GET_LOCK 改为基于 MDL 的 `USER_LEVEL_LOCK` namespace** |
| 8.0 | **instant DDL** 全程只持 SU；`MDL_scoped_lock`/`MDL_object_lock` 子类删除，改为**策略对象**；新增 BACKUP_LOCK、SRID、ACL_CACHE、COLUMN_STATISTICS、RESOURCE_GROUPS、FOREIGN_KEY、CHECK_CONSTRAINT 等 namespace |
| 8.0 | **分区设计被 `LF_HASH` 无锁哈希取代**（`metadata_locks_hash_instances` 移除）——`MDL_map` 直接持 `LF_HASH`，配合 fast path 足够强后不再需要多分区拆热点 |
| 8.0 | 服务端 LOCK ORDER 工具（`sql/debug_lock_order.cc`，`WITH_LOCK_ORDER` 构建选项） |

---

## 理论基础

### 设计思想与权衡

#### 一、锁类型分级 + 意向锁：把"访问意图"编码成强度阶梯

MDL 把意图分成十余级（`enum_mdl_type`），每级对应一类 SQL：

| 类型 | 对应语句 | unobtrusive? |
|---|---|---|
| `MDL_INTENTION_EXCLUSIVE`(IX) | scoped 层的意向（Schema/GLOBAL 等） | ✅（scoped 唯一） |
| `MDL_SHARED`(S) | 只关心元数据（存储例程、预处理） | ✅ |
| `MDL_SHARED_HIGH_PRIO`(SH) | `SHOW`/I_S 查询 | ✅ |
| `MDL_SHARED_READ`(SR) | `SELECT` | ✅ |
| `MDL_SHARED_WRITE`(SW) | `INSERT/UPDATE/DELETE` | ✅ |
| `MDL_SHARED_WRITE_LOW_PRIO`(SWLP) | `... LOW_PRIORITY` | ✅ |
| `MDL_SHARED_UPGRADABLE`(SU) | online DDL 第一阶段（可升级） | ❌ |
| `MDL_SHARED_READ_ONLY`(SRO) | `LOCK TABLES ... READ` | ❌ |
| `MDL_SHARED_NO_WRITE`(SNW) | 可升级到 X（online DDL 主阶段禁写） | ❌ |
| `MDL_SHARED_NO_READ_WRITE`(SNRW) | `LOCK TABLES ... WRITE` | ❌ |
| `MDL_EXCLUSIVE`(X) | `CREATE/DROP/RENAME TABLE` | ❌ |

IX 源自 Gray 的多粒度锁协议（先用粗粒度 IX 锁 schema/GLOBAL，再细粒度锁表）。

#### 二、★ 双兼容矩阵：granted 回答"能不能给"，waiting 回答"该不该轮到我"

这是 MDL 最有设计含量的一点。MDL 维护**两组**矩阵：

```cpp
struct MDL_lock_strategy {
  /** 已授予锁的兼容性：两个已授予的锁能否共存 */
  bitmap_t m_granted_incompatible[MDL_TYPE_END];
  /** 等待锁的优先级：4 张，防饥饿 */
  bitmap_t m_waiting_incompatible[4][MDL_TYPE_END];
  ...
};
```

**为什么需要两套**：只按 granted 判断的话，一个等待中的 X（DDL）永远拿不到——后续 SR（SELECT）源源不断插队，造成**写锁饥饿**。`m_waiting_incompatible` 让等待中的 X 能"影子阻塞"后续请求。

代价就是著名的**MDL 排队问题**：DDL 一排队，后续所有同表查询都被阻塞（默认矩阵 #0 里 `S/SR/SW/...` 行的 X 位都是 1）。这是"防写锁饥饿"对"读并发"的牺牲。

#### 三、★ fast path：用"计数"换"链表"，用"CAS"换"mutex"

DML 的 SR/SW 是最高频操作。若每次都加 `m_rwlock` 写锁 + 遍历 granted 链表，所有 DML 会在全局锁上串行化。MDL 的解法（源码注释原文翻译）：

> A) **"unobtrusive" 类型**：① 集合内任意两种类型互相兼容（包括自身）；② 是 DML 常见操作。我们的目标是优化这类锁的获取与释放——避开 `m_waiting`/`m_granted` bitmap 与链表的复杂检查，**用整数计数器的检查 + 增减替代**。我们称之为 "fast path"。
> B) **"obtrusive" 类型**：① 与某些类型（含自身）不兼容；② 不是 DML 常见操作。必须走 "slow path"（操作链表）。而且**只要存在 obtrusive 的已授予/等待锁，连 unobtrusive 锁也必须走 slow path**。

这是一种**延迟精算（lazy materialization）**：把"谁持有"的精确信息从每次 DML 推迟到少数真正需要的时刻（等待前、请求 obtrusive 锁时、有 open HANDLER 时）再补齐。

代价（四条，都很实在）：

1. **丢失"谁持有"**：`MDL_lock::get_lock_owner()` 注释明写 *"It won't return context if it has used 'fast' path"*。
2. **死锁检测看不见**：故 `will_wait_for()` 开头强制 `materialize_fast_path_locks()`。
3. **强度合并**：S/SH 归为 S，SW/SWLP 归为 SW（授予后不再区分）。
4. **存在 obtrusive 锁时整体退化**，且退化是单向的（已物化的 ticket 不回 fast path）。

#### 四、防饥饿：4 张矩阵动态切换，但默认等于关闭

`m_waiting_incompatible` 有 **4 张**，由两个计数器与阈值 `max_write_lock_count` 选择：

```cpp
uint idx = 0;
if (m_piglet_lock_count >= max_write_lock_count) idx += 1;
if (m_hog_lock_count     >= max_write_lock_count) idx += 2;
return idx;
```

- **hog** = `SNW|SNRW|X`（`MDL_OBJECT_HOG_LOCK_TYPES`），**piglet** = `SW`；
- 计数逻辑：授予 hog 时若还有非 hog 在等 → `m_hog_lock_count++`；授予 SW 时若有 pending SRO → `m_piglet_lock_count++`；
- 一旦切换矩阵，必须立刻 `reschedule_waiters()` 重扫等待队列——注释说明原因："Switch of priority matrice might have unblocked some lower-prio locks... otherwise we might get deadlocks"。

**⚠️ 事实澄清**：`max_write_lock_count` **是全局系统变量，不是编译期常量**（旧资料常写错）。变量本体定义在 `mysys/thr_lock.cc`（`ulong max_write_lock_count = ~(ulong)0L`），注册在 `sql/sys_vars.cc`：

```cpp
static Sys_var_ulong Sys_max_write_lock_count(
    "max_write_lock_count",
    "After this many write locks, allow some read locks to run in between",
    GLOBAL_VAR(max_write_lock_count), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1, ULONG_MAX), DEFAULT(ULONG_MAX), BLOCK_SIZE(1));
```

默认值 **`ULONG_MAX`** ⇒ **防饥饿机制默认等于不触发**。而且它**被 MDL 与 THR_LOCK 两套锁系统共用**（THR_LOCK 侧的计数器是 `THR_LOCK::write_lock_count`）——调小它会同时改变引擎层表锁的读写调度，是个全局性行为改变。

设计取舍很明确：**吞吐优先，饥饿是罕见且可接受的**；要严格公平就自己调小。

#### 五、失效场景与退化阈值

1. **DDL 排队阻塞整表**：默认矩阵下 pending X 会阻塞后续所有 S/SW（唯一豁免是 SH，见下）。
2. **`lock_wait_timeout` 默认 1 年**（`DEFAULT(LONG_TIMEOUT)`，`LONG_TIMEOUT = 3600*24*365`）——对 MDL 意味着"等 DDL 默认几乎永不超时"。长事务 + 一个 pending X 就能让整张表无限期挂住；生产普遍建议调到 3~10 秒。内核自己打开系统表时还会用 `MYSQL_LOCK_IGNORE_TIMEOUT` 绕开用户超时。
3. **SH 的滥用风险**：`MDL_SHARED_HIGH_PRIO` 行在默认矩阵里全 0（完全无视 pending 队列），所以 I_S/SHOW 不会被排队的 DDL 堵住。但源码注释警告：SH 持有者**不得再申请任何表级/行级锁**，否则可能死锁并饿死 X。
4. **死锁检测误报**：搜索深度达 32 就**无条件判定死锁**（不是放弃检测）。
5. **跨层环检测不到**（见「死锁检测」）。

### 理论溯源

- **多粒度封锁（multiple granularity locking）**：Gray, Lorie, Putzolu, Traiger (1976)。落点：IX 意向锁 + scoped（GLOBAL/SCHEMA）与 per-object（TABLE）两级。
- **wait-for graph 死锁检测**：Gray & Reuter《Transaction Processing》的经典体系。落点：`MDL_wait_for_subgraph` 抽象 + `Deadlock_detection_visitor` + `find_deadlock`。与 InnoDB 的 DFS 环检测（见 `innodb_trx_lock.md`）是同一理论在两层的分别落地。
- **延迟计算 / lazy materialization**（fast path 的思想原型）：与编译器的 lazy codegen、数据库的 lazy evaluation 同源——把精度成本从高频路径转移到低频路径。

### 算法与数据结构

| 结构 | 组成 | 复杂度 |
|---|---|---|
| `MDL_key` | 固定 387 字节缓冲：`<ns:1B><db>\0<object>\0[<column>\0]` | 比较 O(len) |
| `MDL_lock` | 两个 `Ticket_list`（侵入式链表 + 11 位 bitmap）+ `atomic<longlong> m_fast_path_state` | fast path O(1) CAS；slow path O(n) 扫描 |
| `MDL_ticket_store` | 3 条 duration 链表 + **超 256 张 ticket 才懒建哈希索引** | 查找 O(n)→O(1) |
| 全局 `MDL_map` | `LF_HASH`（无锁哈希 + hazard pointer，靠 `m_pins`） | find_or_insert O(1) |

关键优化：**每个队列带一个 11 位类型 bitmap**（`MDL_BIT(A) = 1U << A`），于是"有无冲突"从"遍历链表"退化成"一次位与"。

### 他库对比与演进动机

| 数据库 | 元数据/DDL 锁 | 与 MySQL 的差异 |
|---|---|---|
| **PostgreSQL** | DDL 走 `AccessExclusiveLock`（也在统一的 lock manager 里，与行锁共享死锁检测） | PG 的 MVCC + catalog 版本化让 DDL 与查询冲突模型更宽松；MySQL 5.6 的"DDL 全程持 X"阻塞严重，5.7 online DDL 正是向"DDL 少持锁"靠拢 |
| **Oracle** | library cache pin/mutex + DDL 的 exclusive enqueue | 与 MDL 同构（元数据单独一套），但 Oracle 无"fast path 计数"这类优化（其 mutex 本身已极快） |
| **MySQL InnoDB** | 完全自研（见 [`innodb_trx_lock.md`](innodb_trx_lock.md)），且与 MDL **互不通信** | 分层代价：跨层死锁检测不到（见「死锁检测」末尾） |

**★ 为什么 Oracle 不需要 MDL**（内核月报《跟踪 Metadata lock》的分析，值得全文记住）：Oracle 是**堆表**——`ALTER` 只改数据字典，数据记录原地不动（加 default 值也是对原记录 update），完全可以用 SCN 构建一致性读版本，因此 serializable 级别下"select 多次结果一致"与"数据字典变化立即可见"可以同时成立。MySQL 是 **IOT 表**（InnoDB 聚簇表）——alter 要进行表重建，无法对重建中的表构建 read view。加上 Oracle 的 redo 是物理日志（写入顺序与提交顺序无关），不存在 binlog 这种逻辑日志的顺序问题。

所以两条路（月报原文观点）：要么 DDL 只改数据字典 + 行记录原地变更（在线加字段，即 8.0 instant DDL 的方向），要么用物理 redo 避免 binlog——这两条对当时的架构都是大改，MDL 是那个时代最便宜的答案。**这也是 instant DDL 与 MDL 关系的深层注脚：instant 越成熟，MDL 的强锁需求越少。**

**为什么演进成今天这样**：5.5 引入 MDL 是因为旧方案（`.frm` + table cache 锁）无法表达"DDL 与长事务的一致性"；5.7/8.0 的演进主线是**减少 DDL 持锁时间**（online → instant），而 fast path 是另一条主线——**减少 DML 的加锁成本**。两条线共同把 MDL 从"正确性机制"打磨成"性能可接受的机制"。

---

## 核心实现

### 主链路

```
SQL 执行 → open_tables() → 每张表构造 MDL_request（Table_ref::mdl_request）
  → MDL_context::acquire_lock(request, timeout)
      ├─ ① find_ticket()：本 context 已有 ≥ 强度的锁 → 复用（零成本返回）
      ├─ ② 请求 obtrusive 类型（或本线程有 open HANDLER）→ materialize_fast_path_locks()
      ├─ ③ X 锁 + 特定 namespace → 先征求 SE 同意（notify_hton_pre_acquire_exclusive）
      ├─ ④ try_acquire_lock_impl()
      │     ├─ fast path：CAS m_fast_path_state（无 mutex、无链表）→ 成功即返回
      │     └─ slow path：wrlock m_rwlock → can_grant_lock() → granted / 交给外
      └─ ⑤ 未授予 → 入 m_waiting 队列 → will_wait_for()（物化）→ find_deadlock()
            → m_wait.timed_wait(...)
  → 被唤醒（GRANTED/VICTIM/TIMEOUT/KILLED）→ done_waiting_for()
```

释放侧：

```
语句末 → MDL_context::release_statement_locks()         释放 STATEMENT 链
事务末 → release_transactional_locks()                  释放 STATEMENT + TRANSACTION
ROLLBACK TO SAVEPOINT → rollback_to_savepoint(svp)      只释放 svp 之后新加的
显式   → release_lock(ticket) / release_all_locks_for_name() / release_locks(visitor)
```

### 锁的数据结构

#### 四件套与全局 map

```
   THD  ──implements──►  MDL_context_owner（enter_cond/is_killed/notify_hton...）
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ MDL_context（每连接一个）                                     │
│   m_ticket_store : MDL_ticket_store                          │
│      m_durations[0] MDL_STATEMENT   ─ Ticket_p_list ─┐       │
│      m_durations[1] MDL_TRANSACTION ─ Ticket_p_list ─┤       │
│      m_durations[2] MDL_EXPLICIT    ─ Ticket_p_list ─┤       │
│      （>256 张 ticket 时懒建哈希索引加速 find）        │       │
│   m_wait : MDL_wait（mutex+cond+status）              │       │
│   m_waiting_for : MDL_wait_for_subgraph*  ← 死锁图边  │       │
│   m_pins : LF_PINS（LF_HASH hazard pointer）          │       │
└──────────────────────────────────────────────────────┼───────┘
                              next_in_context          │
                              prev_in_context          ▼
                       ┌──────────────────────────────────────┐
                       │ MDL_ticket（每把已授予的锁一个）        │
                       │   m_type / m_ctx / m_lock             │
                       │   m_is_fast_path / m_hton_notified    │
                       │   next_in_lock / prev_in_lock ──┐     │
                       └────────────────────────────────┼─────┘
                                    ▲                   │
                    request.ticket ─┘                   │
       ┌──────────────────────────┐                     │
       │ MDL_request（外部分配：   │                     │
       │  Table_ref 成员 / MEM_ROOT│                     │
       │  key : MDL_key           │                     │
       │  type / duration         │                     │
       └──────────────────────────┘                     ▼
    ┌────────────────────────────────────────────────────────────────┐
    │ MDL_lock（每个 key 一个，全局单例 mdl_locks : LF_HASH）          │
    │   key : MDL_key（内嵌 by value）                                │
    │   m_rwlock  ← 必须是 reader-preferring 的 mysql_prlock_t        │
    │   m_strategy ─► &m_scoped_lock_strategy | &m_object_lock_strategy│
    │   m_granted : Ticket_list { m_list + m_bitmap } ◄───────────────┤
    │   m_waiting : Ticket_list { m_list + m_bitmap } ◄───────────────┘
    │   m_fast_path_state : atomic<longlong>
    │   m_obtrusive_locks_granted_waiting_count
    │   m_hog_lock_count / m_piglet_lock_count
    └────────────────────────────────────────────────────────────────┘
```

`request → ticket` 的转变：`MDL_request` 是 SQL 层分配的"申请单"，`MDL_ticket` 是 MDL 子系统内部 new 出来的"凭证"；成功后 `mdl_request->ticket = ticket`，此后 request 只是句柄。`MDL_request` 与 `MDL_ticket` 分成两个类是因为**生命周期不同**（源码注释原文）：请求的分配由 MDL 子系统外部控制，而 ticket 的分配由 MDL 内部控制。

**为什么 `m_rwlock` 必须是 reader-preferring**：源码给了完整反例——若偏好写者，两个并发的死锁检测线程 A、B 各自 read-lock 了 obj1/obj2，此时新来的 C/D 想 write-lock 被阻塞，A、B 再尝试 read-lock 对方的对象时会被 pending writer 挡住 ⇒ **死锁检测器自身死锁**。

#### MDL_map：8.0 的无锁哈希 + 随机淘汰（取代 5.6 的分区）

全局只有一个 `MDL_map mdl_locks`，8.0 里它直接持一个 **`LF_HASH`**：

```cpp
/** LF_HASH with all locks in the server. */
LF_HASH m_locks;
MDL_lock *m_global_lock;      /* GLOBAL namespace 预分配 */
MDL_lock *m_commit_lock;      /* COMMIT namespace 预分配 */
MDL_lock *m_acl_cache_lock;   /* ACL_CACHE 预分配 */
MDL_lock *m_backup_lock;      /* BACKUP_LOCK 预分配 */
```

初始化用 `lf_hash_init2` 接入三个适配器：

```cpp
lf_hash_init2(&m_locks, sizeof(MDL_lock), LF_HASH_UNIQUE, 0, 0, mdl_locks_key,
              &my_charset_bin, &murmur3_adapter,   /* murmur3 哈希 */
              &mdl_lock_cons, &mdl_lock_dtor, &mdl_lock_reinit);  /* placement new 构造/析构/复用 */
```

**版本演进故事**：5.6.8 之前所有锁对象在一个全局哈希表里、一把 mutex 保护，成为多核热点（Bug #66473）——当时的解法是**按 key 哈希分区**（`metadata_locks_hash_instances`，默认 8，每分区独立 mutex + hash）。8.0 改走**无锁哈希 + 惰性回收**：`LF_HASH` 本身无锁，查找/插入靠 `MDL_context::m_pins`（hazard pointer）；锁对象不立即 free，先打 `IS_DESTROYED` 标志（即 `m_fast_path_state` 的 bit 62），等 fast path 计数归零（引用计数语义）才回收。

淘汰策略也很有意思——不删热点对象，**随机游标淘汰冷对象**：

```cpp
/* 注释原文大意：
   *) unused/total 比例足够高时，随机 dive 很可能很快找到未用对象；
   *) 我们的 PRNG 保证最终会遍历 LF_HASH 的所有 bucket，不会死循环；
   *) 因为是随机选择被淘汰对象，高频使用的对象自然留下，冷对象被逐出。 */
MDL_lock *lock = static_cast<MDL_lock *>(lf_hash_random_match(
    &m_locks, pins, &mdl_lock_match_unused, ctx->get_random(), nullptr));
```

这是"随机淘汰近似 LRU"的教科书技巧，`remove_random_unused` 只在 `unused/total` 比例超阈值时触发（`m_unused_lock_objects` 计数 + 阈值检查）。

#### `MDL_lock` 关键字段

```cpp
class MDL_lock {
  MDL_key key;                        // 内嵌：1:1，注释称"MDL 子系统的 TABLE_SHARE"
  mysql_prlock_t m_rwlock;            // 必须 reader-preferring
  const MDL_lock_strategy *m_strategy;      // ← 取代了旧的 MDL_scoped_lock/MDL_object_lock 子类
  Ticket_list m_granted;              // 已授予（链表 + 11 位 bitmap）
  Ticket_list m_waiting;              // 等待中
  std::atomic<fast_path_state_t> m_fast_path_state;
  uint m_obtrusive_locks_granted_waiting_count;   // ★ 真名（旧资料常写成 m_obtrusive_locks_granted）
  ulong m_hog_lock_count;             // 连续授予 hog(SNW/SNRW/X) 的次数
  ulong m_piglet_lock_count;          // 连续授予 piglet(SW) 的次数
  uint m_current_waiting_incompatible_idx;   // 当前用哪张 waiting 矩阵
};
```

**⚠️ 勘误**：`m_is_scoped`、`m_obtrusive_locks_granted`、`MDL_scoped_lock`、`MDL_object_lock`、`MDL_context::m_tickets` **在 8.0.39 都不存在**。分别是 `m_strategy` 指针、`m_obtrusive_locks_granted_waiting_count`、两个策略实例、`m_ticket_store`。看到旧写法（月报常见）请以此为准。

#### 18 个 namespace 与两套策略

```cpp
enum enum_mdl_namespace {
  GLOBAL=0, BACKUP_LOCK, TABLESPACE, SCHEMA, TABLE, FUNCTION, PROCEDURE,
  TRIGGER, EVENT, COMMIT, USER_LEVEL_LOCK, LOCKING_SERVICE, SRID,
  ACL_CACHE, COLUMN_STATISTICS, RESOURCE_GROUPS, FOREIGN_KEY,
  CHECK_CONSTRAINT, NAMESPACE_END   // = 18
};
```

| # | namespace | 用途 | 策略 | 粒度 |
|---|---|---|---|---|
| 0 | `GLOBAL` | 全局读锁（FTWRL） | **scoped** | 实例级（singleton） |
| 1 | `BACKUP_LOCK` | 备份锁 | **scoped** | 实例级（singleton） |
| 2 | `TABLESPACE` | 表空间 | **scoped** | — |
| 3 | `SCHEMA` | 数据库 | **scoped** | 库级 |
| 4 | `TABLE` | 表和视图 | object | 表级 |
| 5–8 | `FUNCTION`/`PROCEDURE`/`TRIGGER`/`EVENT` | 存储程序、触发器、事件 | object | 对象级 |
| 9 | `COMMIT` | 让 FTWRL 能阻塞提交 | **scoped** | 实例级（singleton） |
| 10 | `USER_LEVEL_LOCK` | 用户级锁 GET_LOCK | object | 字符串名 |
| 11 | `LOCKING_SERVICE` | 命名插件 RW-lock service | object | 字符串名 |
| 12 | `SRID` | 空间参考系 | object | 对象级 |
| 13 | `ACL_CACHE` | ACL 缓存 | object | 实例级（singleton） |
| 14 | `COLUMN_STATISTICS` | 列统计/直方图 | object | **列级**（key 带第 4 段列名） |
| 15–17 | `RESOURCE_GROUPS`/`FOREIGN_KEY`/`CHECK_CONSTRAINT` | 资源组、外键名、检查约束名 | **scoped** | 对象级 |

分派代码（`get_strategy()` 完整）：

```cpp
inline static const MDL_lock_strategy *get_strategy(const MDL_key &key) {
  switch (key.mdl_namespace()) {
    case MDL_key::GLOBAL:
    case MDL_key::TABLESPACE:
    case MDL_key::SCHEMA:
    case MDL_key::COMMIT:
    case MDL_key::BACKUP_LOCK:
    case MDL_key::RESOURCE_GROUPS:
    case MDL_key::FOREIGN_KEY:
    case MDL_key::CHECK_CONSTRAINT:
      return &m_scoped_lock_strategy;
    default:
      return &m_object_lock_strategy;
  }
}
```

**8 个** namespace 归 scoped，其余归 object。（**注**：该处注释仍写"4 个 namespace"，滞后于代码——这是新增 namespace 时必须记得更新的地方。）

两套策略的差异（数据表化，非继承）：

| | scoped | object |
|---|---|---|
| 合法类型 | 仅 `IX`/`S`/`X` | 除 IX 外全部 |
| unobtrusive | `IX` | `S`/`SH`/`SR`/`SW`/`SWLP` |
| obtrusive | `S`/`X` | `SU`/`SRO`/`SNW`/`SNRW`/`X` |
| 受 `max_write_lock_count` 影响 | 否（注释：scoped 下只有 IX 会被 X/S 饿死，"practically a very rare case"） | 是（ACL_CACHE 除外） |
| 是否通知 SE | 否 | 是（仅 X 锁，7 个 namespace） |
| singleton 预分配 | GLOBAL/COMMIT/BACKUP_LOCK 是 | ACL_CACHE 是 |

#### `MDL_lock_strategy`：4 个函数指针钩子（策略差异的真正载体）

完整结构（兼容矩阵、优先级矩阵、fast path 增量之外，还有 4 个行为钩子）：

```cpp
struct MDL_lock_strategy {
  bitmap_t m_granted_incompatible[MDL_TYPE_END];       /* 兼容矩阵 */
  bitmap_t m_waiting_incompatible[4][MDL_TYPE_END];    /* 4 张优先级矩阵 */
  fast_path_state_t m_unobtrusive_lock_increment[MDL_TYPE_END];  /* fast path 增量 */
  bool m_is_affected_by_max_write_lock_count;
#ifndef NDEBUG
  bool legal_type[MDL_TYPE_END];   /* debug：断言某策略只接受合法类型（scoped 不接受 SR 等） */
#endif
  bool (*m_needs_notification)(const MDL_ticket *ticket);
  void (*m_notify_conflicting_locks)(MDL_context *ctx, MDL_lock *lock);
  bitmap_t (*m_fast_path_granted_bitmap)(const MDL_lock &lock);
  bool (*m_needs_connection_check)(const MDL_lock *lock);
};
```

两套策略的函数指针取值差异：

| 钩子 | scoped | object | 语义 |
|---|---|---|---|
| `m_needs_notification` | `nullptr` | **仅 X 锁返回 true** | X 请求到来时通知其他持锁者释放（配合 `acquire_lock` 的 1 秒切片等待循环反复通知） |
| `m_notify_conflicting_locks` | `nullptr` | `object_lock_notify_conflicting_locks` | 唤醒"持 S 锁且卡在 THR_LOCK 上"的线程（`notify_shared_lock`）——源码注释自述这是为 **HANDLER / READ LOCAL / MERGE** 的历史包袱服务，"Once we will get rid of the support for READ LOCAL and MERGE clauses this code can be removed" |
| `m_fast_path_granted_bitmap` | 低位计数非零即 IX | 解码三段计数为 S/SR/SW | fast path 位图的解码方式 |
| `m_needs_connection_check` | `nullptr` | **仅 3 个 namespace 返回 true**：`USER_LEVEL_LOCK` / `LOCKING_SERVICE` / `ACL_CACHE` | 等待期间**每秒检查连接是否还在**——客户端断开就放弃等待。ULL 尤其必要：`GET_LOCK` 的等待者可能已断线，不该无限等 |

这张表回答了一个实际问题："**为什么 GET_LOCK 的等待者断线后能立即退出，而等表锁的不能**"——前者有连接检查钩子，后者没有。

**几个 namespace 的特殊行为**（对应上述钩子）：

- `USER_LEVEL_LOCK` / `LOCKING_SERVICE` / `ACL_CACHE`：连接检查 + ACL_CACHE 还**跳过/延迟死锁检测**（它的 S 锁"每连接甚至每语句"都拿，做死锁检测太贵）；
- X 锁 + 7 个 namespace（TABLESPACE/SCHEMA/TABLE/FUNCTION/PROCEDURE/TRIGGER/EVENT）：`needs_hton_notification`——加 X 前**先征求引擎同意**（`victimized` 回退）；
- `GLOBAL` / `COMMIT` / `BACKUP_LOCK` / `ACL_CACHE`：singleton 预分配、永不回收，LF_HASH 查找不 pin。

#### 防饥饿的执行链：计数、切换与豁免

`MDL_lock_strategy` 的 `m_is_affected_by_max_write_lock_count` 只是"要不要防饥饿"的开关，真正执行在 `MDL_lock` 的三个方法里。完整代码：

```cpp
/* ① 计数：授予 hog / piglet 时检查是否该切换矩阵 */
bool count_piglets_and_hogs(enum_mdl_type type) {
  if ((MDL_BIT(type) & MDL_OBJECT_HOG_LOCK_TYPES) != 0) {
    if (m_waiting.bitmap() & ~MDL_OBJECT_HOG_LOCK_TYPES) {   /* 还有低优先级在等 */
      m_hog_lock_count++;
      if (switch_incompatible_waiting_types_bitmap_if_needed()) return true;
    }
  } else if (type == MDL_SHARED_WRITE) {
    if (m_waiting.bitmap() & MDL_BIT(MDL_SHARED_READ_ONLY)) { /* 有 pending SRO */
      m_piglet_lock_count++;
      if (switch_incompatible_waiting_types_bitmap_if_needed()) return true;
    }
  }
  return false;
}

/* ② 切换：按两个计数与阈值算出新 idx，变了就换 */
bool switch_incompatible_waiting_types_bitmap_if_needed() {
  uint new_idx = get_incompatible_waiting_types_bitmap_idx();
  if (m_current_waiting_incompatible_idx == new_idx) return false;
  m_current_waiting_incompatible_idx = new_idx;
  return true;
}

/* ③ 豁免：ACL_CACHE 不做防饥饿——为了省掉 find_deadlock() */
bool is_affected_by_max_write_lock_count() const {
  /*
    Disable max_write_lock_count handling for ACL_CACHE namespace to
    enable optimization that avoids find_deadlock() for it.
  */
  return key.mdl_namespace() != MDL_key::ACL_CACHE &&
         m_strategy->m_is_affected_by_max_write_lock_count;
}
```

三段拼起来就是完整闭环：**"授予时计数（①）→ 计数超阈值切矩阵（②）→ 切矩阵后立即 `reschedule_waiters()` 重扫（在调用点）→ 等待队列清空后复位计数与 idx（在 `reschedule_waiters` 末尾）"**。① 只计"有更低优先级在等"时的授予——没有竞争就没有饥饿，不需要计数。③ 的注释揭示了豁免的真实动机：不是 ACL_CACHE 不会饿死，而是**它高频到连 `find_deadlock()` 都想省**。

**`legal_type` 的用途**（NDEBUG 下编译掉的 debug 字段）：scoped 策略 `{true,true,false,...,false,true}`（只接受 IX/S/X）、object 策略除 IX 外全 true。它在 `try_acquire_lock_impl` 里被 `assert` 检查，防止"对 GLOBAL 请求 SR"这类错误在 release 下静默跑错——策略是编译期契约的运行时断言。

### ★ fast path 与 slow path

#### obtrusive/unobtrusive 的划分

判定用"fast path 增量是否为 0"表达：

```cpp
fast_path_state_t get_unobtrusive_lock_increment(enum_mdl_type type) const {
  return m_strategy->m_unobtrusive_lock_increment[type];
}
bool is_obtrusive_lock(enum_mdl_type type) const {
  return get_unobtrusive_lock_increment(type) == 0;
}
```

两套策略的增量数组（下标 = `enum_mdl_type`）：

```cpp
// scoped：只有 IX 是 unobtrusive
{1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}
// object：S/SH 占 bits 0..19，SR 占 20..39，SW/SWLP 占 40..59
{0, 1, 1, 1ULL << 20, 1ULL << 40, 1ULL << 40, 0, 0, 0, 0, 0}
```

#### `m_fast_path_state` 的位布局

```
bit 62        IS_DESTROYED    锁对象已被销毁（LF_HASH 回收中）
bit 61        HAS_OBTRUSIVE   存在 granted/pending/"正要检查" 的 obtrusive 锁
bit 60        HAS_SLOW_PATH   已有 ticket 通过 slow path 授予
bits 59..0    打包的 unobtrusive 锁计数
                scoped: 全部给 IX
                object: [0..19]=S/SH  [20..39]=SR  [40..59]=SW/SWLP
```

每个类型占 20 位，注释给出理由："Overflow is not an issue as we are unlikely to support more than 2^20 - 1 concurrent connections in foreseeable future."

不变式 **[INV1]**（源码原文）：

> When this flag (`HAS_OBTRUSIVE`) is set all changes to `m_fast_path_state` member has to be done under protection of `m_rwlock` lock.

#### fast path 授予（无锁 CAS）

```cpp
MDL_lock::fast_path_state_t old_state = lock->m_fast_path_state;
bool first_use;
do {
  if (old_state & MDL_lock::IS_DESTROYED) { ... goto retry; }
  if (old_state & MDL_lock::HAS_OBTRUSIVE) goto slow_path;
  first_use = (old_state == 0);
} while (!lock->fast_path_state_cas(&old_state,
                                    old_state + unobtrusive_lock_increment));

ticket->m_lock = lock;
ticket->m_is_fast_path = true;
m_ticket_store.push_front(mdl_request->duration, ticket);
mdl_request->ticket = ticket;
mysql_mdl_set_status(ticket->m_psi, MDL_ticket::GRANTED);
return false;
```

三个要点：

1. **ticket 仍然创建**并挂进 `m_ticket_store`（不建 ticket 就无法释放/物化/参与死锁图）；只是**不进 `MDL_lock::m_granted`**，只在计数器里加一。
2. 注释解释了为何先"普通读"再 CAS 是安全的："correctness of value returned by it will be anyway validated by atomic compare-and-swap which happens later. In theory, this algorithm will work correctly (but not very efficiently) if the read will return random values."
3. `m_fast_path_state` 的低位同时充当**引用计数**——非零表示对象在用，LF_HASH 不能回收它。

#### 强制走 slow path 的两个条件

```cpp
/*
  If this "obtrusive" type we have to take "slow path".
  If this context has open HANDLERs we have to take "slow path"
  as well for MDL_object_lock::notify_conflicting_locks() to work properly.
*/
force_slow = !unobtrusive_lock_increment || m_needs_thr_lock_abort;

/* If "obtrusive" lock is requested we need to "materialize" all fast
   path tickets, so MDL_lock::can_grant_lock() can safely assume
   that all granted "fast path" locks belong to different context. */
if (!unobtrusive_lock_increment) materialize_fast_path_locks();
```

#### 物化（fast → slow，单向）

```cpp
void MDL_context::materialize_fast_path_locks() {
  for (int i = 0; i < MDL_DURATION_END; i++) {
    MDL_ticket_store::List_iterator it = m_ticket_store.list_iterator(i);
    MDL_ticket *matf = m_ticket_store.materialized_front(i);
    for (MDL_ticket *ticket = it++; ticket != matf; ticket = it++) {
      if (ticket->m_is_fast_path) {
        MDL_lock *lock = ticket->m_lock;
        auto increment = lock->get_unobtrusive_lock_increment(ticket->get_type());
        ticket->m_is_fast_path = false;
        mysql_prlock_wrlock(&lock->m_rwlock);
        lock->m_granted.add_ticket(ticket);
        /* 原子减计数 —— 必须在 m_rwlock 保护下，以与加链表原子、并满足 [INV1] */
        auto old_state = lock->m_fast_path_state;
        while (!lock->fast_path_state_cas(&old_state,
                   ((old_state - increment) | MDL_lock::HAS_SLOW_PATH))) {}
        mysql_prlock_unlock(&lock->m_rwlock);
      }
    }
  }
  m_ticket_store.set_materialized();
}
```

三个触发时机：① 本 context 请求 obtrusive 锁；② `will_wait_for()`（开始等待任何资源前）；③ `set_needs_thr_lock_abort(true)`（有 open HANDLER）。`m_mat_front` 是"物化水位线"，只扫描上次之后新增的 ticket。

#### 释放侧也要看 obtrusive

```cpp
if (ticket->m_is_fast_path) {
  auto old_state = lock->m_fast_path_state;
  do {
    if (old_state & MDL_lock::HAS_OBTRUSIVE) {
      mysql_prlock_wrlock(&lock->m_rwlock);
      last_use = (lock->fast_path_state_add(-increment) == increment);
      if (lock->m_obtrusive_locks_granted_waiting_count)
        lock->reschedule_waiters();          // ★ 否则 obtrusive 等待者永远等下去
      mysql_prlock_unlock(&lock->m_rwlock);
      goto end_fast_path;
    }
    last_use = (old_state == increment);
  } while (!lock->fast_path_state_cas(&old_state, old_state - increment));
} else {
  lock->remove_ticket(this, m_pins, &MDL_lock::m_granted, ticket);
}
```

### 加锁流程

#### ① 复用已有 ticket（零成本）

```cpp
if ((ticket = find_ticket(mdl_request, &found_duration))) {
  mdl_request->ticket = ticket;
  if ((found_duration != mdl_request->duration ||
       mdl_request->duration == MDL_EXPLICIT) && clone_ticket(mdl_request)) {
    mdl_request->ticket = nullptr;
    return true;
  }
  return false;      // 复用成功
}
```

`find_ticket` 在 `m_ticket_store` 里找"同 key 且强度 ≥"的 ticket。**duration 不同时要 `clone_ticket()`** —— 否则 HANDLER CLOSE 会释放掉事务锁、或 COMMIT 误释放 HANDLER 锁。

`m_ticket_store` 的查找优化：少于 256 张 ticket 时线性扫三条链表，`find_in_lists` 按 `(req.duration + i) % 3` 轮转；超过阈值才懒建哈希索引。

#### ② 征求存储引擎同意（X 锁专用）

```cpp
if (mdl_request->type == MDL_EXCLUSIVE &&
    MDL_lock::needs_hton_notification(key->mdl_namespace())) {
  mysql_mdl_set_status(ticket->m_psi, MDL_ticket::PRE_ACQUIRE_NOTIFY);
  bool victimized;
  if (m_owner->notify_hton_pre_acquire_exclusive(key, &victimized)) {
    MDL_ticket::destroy(ticket);
    my_error(victimized ? ER_LOCK_DEADLOCK : ER_LOCK_REFUSED_BY_ENGINE, MYF(0));
    return true;
  }
  ticket->m_hton_notified = true;
}
```

覆盖 7 个 namespace（TABLESPACE/SCHEMA/TABLE/FUNCTION/PROCEDURE/TRIGGER/EVENT）。`victimized=true` 表示引擎说"为了解开跨层死锁，请放弃这次请求"。

#### ④ slow path 与 `can_grant_lock` 三级短路

```cpp
bool MDL_lock::can_grant_lock(enum_mdl_type type_arg,
                              const MDL_context *requestor_ctx) const {
  bool can_grant = false;
  bitmap_t waiting_incompat_map = incompatible_waiting_types_bitmap()[type_arg];
  bitmap_t granted_incompat_map = incompatible_granted_types_bitmap()[type_arg];

  if (!(m_waiting.bitmap() & waiting_incompat_map)) {           // ① 等待队列
    if (!(fast_path_granted_bitmap() & granted_incompat_map)) { // ② fast path 计数
      if (!(m_granted.bitmap() & granted_incompat_map))         // ③ slow path bitmap
        can_grant = true;
      else {
        Ticket_iterator it(m_granted);
        MDL_ticket *ticket;
        while ((ticket = it++)) {
          if (ticket->get_ctx() != requestor_ctx &&
              ticket->is_incompatible_when_granted(type_arg)) break;
        }
        if (ticket == nullptr) /* Incompatible locks are our own. */
          can_grant = true;    // ★ 自己的锁不算冲突（锁升级/重复加锁靠这条）
      }
    }
    /* else: 与 fast path 锁冲突 ⇒ 必是 obtrusive 请求，且冲突锁不是自己的 */
  }
  return can_grant;
}
```

三级短路：等待队列 → fast path 位图 → slow path 位图 → 最后线性扫描确认"冲突锁不是自己的"。

#### 兼容矩阵（源码里的 ASCII 表，原文）

**scoped granted**：

```
                 | Type of active   |
         Request |   scoped lock    |
          type   | IS(*)  IX   S  X |
        ---------+------------------+
        IS       |  +      +   +  + |
        IX       |  +      +   -  - |
        S        |  +      -   +  - |
        X        |  +      -   -  - |
```

**object granted**（关键格解读见下）：

```
         Request  |  Granted requests for lock            |
          type    | S  SH  SR  SW  SWLP  SU  SRO  SNW  SNRW  X  |
        ----------+---------------------------------------------+
        S         | +   +   +   +    +    +   +    +    +    -  |
        SH        | +   +   +   +    +    +   +    +    +    -  |
        SR        | +   +   +   +    +    +   +    +    -    -  |
        SW        | +   +   +   +    +    +   -    -    -    -  |
        SWLP      | +   +   +   +    +    +   -    -    -    -  |
        SU        | +   +   +   +    +    -   +    -    -    -  |
        SRO       | +   +   +   -    -    +   +    +    -    -  |
        SNW       | +   +   +   -    -    -   +    -    -    -  |
        SNRW      | +   +   -   -    -    -   -    -    -    -  |
        X         | -   -   -   -    -    -   -    -    -    -  |
```

几个值得记住的格子：

- **`SU` 与 `SU` 自身冲突**——保证一张表同时只有一个 upgrader（`upgrade_shared_lock` 注释："There can be only one upgrader for a lock or we will have deadlock"）。
- **`SW` 与 `SRO`/`SNW` 冲突**、**`SRO` 与 `SW` 冲突**——`LOCK TABLES READ` 与在线 DDL 准备阶段都禁止写。
- **`SNRW` 只允许 `S`/`SH` 并存**。
- **`X` 与一切不共存**。

**object waiting 默认矩阵（#0）完整 ASCII**（这是"优先级"矩阵：行 = 新请求，列 = 已在等待队列的锁）：

```
         Request  |         Pending requests for lock          |
          type    | S  SH  SR  SW  SWLP  SU  SRO  SNW  SNRW  X |
        ----------+--------------------------------------------+
        S         | +   +   +   +    +    +   +    +     +   - |
        SH        | +   +   +   +    +    +   +    +     +   + |
        SR        | +   +   +   +    +    +   +    +     -   - |
        SW        | +   +   +   +    +    +   +    -     -   - |
        SWLP      | +   +   +   +    +    +   -    -     -   - |
        SU        | +   +   +   +    +    +   +    +     +   - |
        SRO       | +   +   +   -    +    +   +    +     -   - |
        SNW       | +   +   +   +    +    +   +    +     +   - |
        SNRW      | +   +   +   +    +    +   +    +     +   - |
        X         | +   +   +   +    +    +   +    +     +   + |
```

**#0 里最值得记住的 4 行**（`+` = 兼容 = "虽有 pending 也能插队"）：

- **`S` 行的 X 位是 `-`**：有 pending X 就排队——**MDL 排队问题的根源**（DDL 一排队，后续 DML/读全被影子阻塞）。
- **`SH` 行全 `+`**：`SH` 完全无视 pending 队列（高优先级通道，I_S/SHOW 不被排队的 DDL 堵住）。
- **`X` 行全 `+`**：X 请求从不因 pending 而等待（最高优先级）。
- **`SRO` 行的 SW 位是 `-`**：pending SW 时新 SRO 可插队（`LOCK TABLES READ` 优先于等待中的 DML）。

**★ 4 张 waiting 矩阵的语义差异**（`m_waiting_incompatible[4]`，源码位图注释原文）：

| # | 触发条件 | 相对 #0 的变化 | 目的 |
|---|---|---|---|
| 0 | 正常 | — | 默认：X 最高、SH 无视 pending、S 让位于 pending X |
| 1 | `m_piglet_lock_count >= max_write_lock_count`（SW 连续授予超阈值） | SW/SWLP 行对 SRO 更严格 → **SW 让位于 pending SRO** | 防 SW 流饿死 `LOCK TABLES READ` |
| 2 | `m_hog_lock_count >= max_write_lock_count`（hog 连续授予超阈值） | **所有非 hog 类型优先于 hog** | 防 DDL 流饿死 DML/读 |
| 3 | 两者都超阈值 | SRO 优先于 SW/SWLP；非 hog（除 SW/SWLP）优先于 hog | 综合两者；且 hog 相对 SW/SWLP 的优先级与 #0 相同（不变量 [INV3]） |

不变式 **[INV2]**（源码原文，reschedule 的正确性依赖它）：

> for all priority matrices, if A is the set of incompatible waiting requests for a given request and B is the set of incompatible granted requests for the same request, then **A will always be a subset of B**. This means that moving a lock from waiting to granted state doesn't unblock additional requests.

#### `reschedule_waiters()`：公平调度

```cpp
/**
  @note Together with MDL_lock::add_ticket() this method implements
        fair scheduling among requests with the same priority.
        It tries to grant lock from the head of waiters list, while
        add_ticket() adds new requests to the back of this list.
*/
void MDL_lock::reschedule_waiters() {
  MDL_lock::Ticket_iterator it(m_waiting);
  MDL_ticket *ticket;
  while ((ticket = it++)) {
    if (can_grant_lock(ticket->get_type(), ticket->get_ctx())) {
      if (!ticket->get_ctx()->m_wait.set_status(MDL_wait::GRANTED)) {
        m_waiting.remove_ticket(ticket);
        m_granted.add_ticket(ticket);
        if (is_affected_by_max_write_lock_count()) {
          if (count_piglets_and_hogs(ticket->get_type())) { it.rewind(); continue; }
        }
      }
      /* 若 wait slot 非空（被 kill/超时），保留在队列里继续找下一个 */
    }
  }
  if (is_affected_by_max_write_lock_count()) {
    /* 复位 hog/piglet 计数与矩阵 idx（按 waiting bitmap 是否清空）。
       注意 idx==3 时不能直接降到 2——否则并发 SW + SRO 流会永久饿死 hog，
       必须等 SW/SWLP 也从等待队列消失才整体归零。 */
    if (m_current_waiting_incompatible_idx == 3) {
      if ((m_waiting.bitmap() &
           ~(MDL_OBJECT_HOG_LOCK_TYPES | MDL_BIT(MDL_SHARED_WRITE) |
             MDL_BIT(MDL_SHARED_WRITE_LOW_PRIO))) == 0) {
        m_piglet_lock_count = 0;
        m_hog_lock_count = 0;
        m_current_waiting_incompatible_idx = 0;
      }
    } else {
      if ((m_waiting.bitmap() & ~MDL_OBJECT_HOG_LOCK_TYPES) == 0) {
        m_hog_lock_count = 0;
        m_current_waiting_incompatible_idx &= ~2;
      }
      if ((m_waiting.bitmap() & MDL_BIT(MDL_SHARED_READ_ONLY)) == 0) {
        m_piglet_lock_count = 0;
        m_current_waiting_incompatible_idx &= ~1;
      }
    }
  }
}
```

**为什么唤醒后要"重新扫一遍"而不是"谁释放唤醒谁"**：释放一个锁后可能有**多个**等待者同时变得可授予，也可能因优先级矩阵切换解锁一批——释放者无法枚举，于是统一在 `m_rwlock` 写锁下从头扫一遍。同优先级 FIFO：`add_ticket` 加队尾、`reschedule_waiters` 从队头授予。

末尾的复位逻辑有个反直觉点：`idx == 3` 时**不能**直接降到 2，注释说明原因是"否则并发 SW + SRO 流会永久饿死 hog"。

### 死锁检测

#### wait-for 边的抽象

```cpp
/**
  Abstract class representing an edge in the waiters graph
  to be traversed by deadlock detection algorithm.
*/
class MDL_wait_for_subgraph {
 public:
  virtual bool accept_visitor(MDL_wait_for_graph_visitor *gvisitor) = 0;
  static const uint DEADLOCK_WEIGHT_CO  = 0;
  static const uint DEADLOCK_WEIGHT_DML = 25;
  static const uint DEADLOCK_WEIGHT_ULL = 50;
  static const uint DEADLOCK_WEIGHT_DDL = 100;
  virtual uint get_deadlock_weight() const = 0;
};
```

实现子类共 4 个：`MDL_ticket`（等 MDL）、`Wait_for_flush`（等 TABLE_SHARE 从 TDC 刷出）、`Commit_order_lock_graph`（从库 worker 等提交顺序）、单测的 `Mock_MDL_wait_for_subgraph`。

**"我正在等谁"存在 `MDL_context::m_waiting_for` 里**，由每次开始等待前写入：

```cpp
void will_wait_for(MDL_wait_for_subgraph *waiting_for_arg) {
  /*
    Before starting wait for any resource we need to materialize
    all "fast path" tickets belonging to this thread. Otherwise
    locks acquired which are represented by these tickets won't
    be present in wait-for graph and could cause missed deadlocks.
  */
  materialize_fast_path_locks();
  mysql_prlock_wrlock(&m_LOCK_waiting_for);
  m_waiting_for = waiting_for_arg;
  mysql_prlock_unlock(&m_LOCK_waiting_for);
}
```

#### 图遍历：先 BFS 探边，再 DFS 递归

```cpp
bool MDL_lock::visit_subgraph(MDL_ticket *waiting_ticket,
                              MDL_wait_for_graph_visitor *gvisitor) {
  MDL_context *src_ctx = waiting_ticket->get_ctx();
  mysql_prlock_rdlock(&m_rwlock);
  /* 假死锁防护：granted 已更新但 m_waiting_for 未清零的窗口 */
  if (src_ctx->m_wait.get_status() != MDL_wait::WS_EMPTY) { result = false; goto end; }
  if (gvisitor->enter_node(src_ctx)) goto end;

  /* We do a breadth-first search first -- that is, inspect all
     edges of the current node, and only then follow up to the next
     node. */
  while ((ticket = granted_it++))
    if (ticket->get_ctx() != src_ctx &&
        ticket->is_incompatible_when_granted(waiting_ticket->get_type()) &&
        gvisitor->inspect_edge(ticket->get_ctx())) goto end_leave_node;
  while ((ticket = waiting_it++))
    if (ticket->get_ctx() != src_ctx &&
        ticket->is_incompatible_when_waiting(waiting_ticket->get_type()) &&
        gvisitor->inspect_edge(ticket->get_ctx())) goto end_leave_node;

  /* Recurse and inspect all adjacent nodes. */
  granted_it.rewind();  /* ... 同样条件，改调 ticket->get_ctx()->visit_subgraph(gvisitor) ... */
  waiting_it.rewind();  /* ... */
  result = false;
  ...
}
```

判边复用同一套矩阵：`is_incompatible_when_granted()` 查 granted 矩阵，`is_incompatible_when_waiting()` 查当前 waiting 优先级矩阵。

#### visitor 与深度上限

```cpp
bool Deadlock_detection_visitor::enter_node(MDL_context *node) {
  m_found_deadlock = ++m_current_search_depth >= MAX_SEARCH_DEPTH;
  if (m_found_deadlock) { assert(!m_victim); opt_change_victim_to(node); }
  return m_found_deadlock;
}
bool Deadlock_detection_visitor::inspect_edge(MDL_context *node) {
  m_found_deadlock = node == m_start_node;      // 回到起点 = 环
  return m_found_deadlock;
}
```

**`MAX_SEARCH_DEPTH = 32` 的语义是"达到即判定死锁"**（不是放弃检测）：注释说动机是防递归爆栈，并自承 "TODO: Find out what is the optimal value for this parameter."——即长链（>32）会**误报**死锁并牺牲一个无辜节点。

victim 选择：

```cpp
void Deadlock_detection_visitor::opt_change_victim_to(MDL_context *new_victim) {
  if (m_victim == nullptr ||
      m_victim->get_deadlock_weight() >= new_victim->get_deadlock_weight()) {
    /* Swap victims, unlock the old one. */
    MDL_context *tmp = m_victim;
    m_victim = new_victim;
    m_victim->lock_deadlock_victim();
    if (tmp) tmp->unlock_deadlock_victim();
  }
}
```

`find_deadlock()` 是 **while 循环**：因为拆掉的是"别人的边"而非自己新加的边，可能还有别的环，必须重复搜索直到无环或自己成为 victim。

#### victim 权重：静态常量而非回滚代价

```cpp
uint MDL_ticket::get_deadlock_weight() const {
  /*
    Waits for user-level locks have lower weight than waits for locks
    typically acquired by DDL, so we don't abort DDL in case of deadlock
    involving user-level locks and DDL. Deadlock errors are not normally
    expected from DDL by users.
    ...
  */
  if (m_lock->key.mdl_namespace() == MDL_key::USER_LEVEL_LOCK)
    return DEADLOCK_WEIGHT_ULL;
  /*
    Locks higher or equal to MDL_SHARED_UPGRADABLE:
    *) Are typically acquired for DDL and LOCK TABLES statements.
    *) Are often acquired in a way which doesn't allow simple release of
       locks and restart of lock acquisition process in case of deadlock
       (e.g. through lock_table_names() call).
    ...
    TODO/FIXME: The below condition needs to be updated. The fact that a
                lock from GLOBAL namespace is requested no longer means
                that this is a DDL statement. There is a bug report about this.
  */
  if (m_lock->key.mdl_namespace() == MDL_key::GLOBAL ||
      m_type >= MDL_SHARED_UPGRADABLE)
    return DEADLOCK_WEIGHT_DDL;
  return DEADLOCK_WEIGHT_DML;
}
```

**DDL(100) > ULL(50) > DML(25) > CO(0)，选最小者**（`>=` 才换，平局保留先遇到的）。理由写在注释里：DDL/LOCK TABLES 的锁常通过 `lock_table_names()` 批量获取，无法简单释放重来；而 DML 在 autocommit=1 下可以 backoff 重试，对用户隐藏死锁错误。

那个 backoff 机制是 DML **自愿当 victim**：

```cpp
thd->mdl_context.set_force_dml_deadlock_weight(ot_ctx->can_back_off());
bool result = thd->mdl_context.acquire_lock(mdl_request, ot_ctx->get_timeout());
thd->mdl_context.set_force_dml_deadlock_weight(false);
```

配合 `MDL_deadlock_handler` 把 `ER_LOCK_DEADLOCK` 转成 `OT_BACKOFF_AND_RETRY`。

#### 与 InnoDB 死锁检测的对比

| 维度 | MDL（server 层） | InnoDB（引擎层） |
|---|---|---|
| 触发时机 | **主动：每次开始等待前**（加边时） | 后台线程周期性 + 事件触发 |
| 图节点 | `MDL_context`（连接） | `trx_t`（事务） |
| 搜索 | 先 BFS 探边再 DFS；深度 **32 即判死锁** | DFS 找环，找到才判 |
| victim 依据 | **静态权重常量**（DDL/ULL/DML/CO） | 动态权重（锁数 + undo 量） |
| 唤醒 | `MDL_wait::set_status(VICTIM)` | 置 victim 标志后回滚 |
| 环解除 | while 循环重复搜（拆的是别人的边） | 通常一次回滚一个事务 |
| **覆盖** | 仅 MDL + 显式接入的 `Wait_for_flush` / 提交顺序 | 仅 InnoDB 锁 |
| **跨层环** | **检测不到** | **检测不到** |

**跨层环是两层的共同盲区**：等 MDL → 持行锁 → 等行锁 的环，MDL 看不到 InnoDB 的等待（无 `MDL_wait_for_subgraph` 子类），InnoDB 看不到 MDL。唯一旁路是 `notify_hton_pre_acquire_exclusive` 的"许可 + victimized 回退"，以及 `object_lock_notify_conflicting_locks()`（HANDLER + THR_LOCK 的历史包袱场景）。兜底全靠 `lock_wait_timeout` 与 `innodb_lock_wait_timeout`。

### 扩展用法

#### 用户级锁 GET_LOCK（5.7 起基于 MDL）

实现在 `sql/item_func.cc`，**直接复用 `MDL_context::acquire_lock()`**：

```cpp
MDL_REQUEST_INIT(&ull_request, MDL_key::USER_LEVEL_LOCK, "", name,
                 MDL_EXCLUSIVE, MDL_EXPLICIT);
```

即：映射到 **X 类型 + EXPLICIT duration**，db 为空串、name 为锁名。注释解释了为何这样能天然支持递归：

> For locks with EXPLICIT duration, MDL returns a new ticket every time a lock is granted. **This allows to implement recursive locks without extra allocation or additional data structures**...

`thd->ull_hash`（`User_level_lock {ticket, refs}`）只是性能补丁——避免 `find_ticket()` 在高 ticket 数时线性查找变慢；同名锁靠 `refs++`，`RELEASE_LOCK` 里 `--refs == 0` 才真正释放。

其他细节：锁名**小写化**（"to ensure that user-level lock names are treated in case-insensitive fashion even though MDL subsystem ... does binary comparison of keys"）；超时 0 立即失败、>INT_MAX32 截为 INT_MAX32（"robust, infinite wait"）；错误处理器把 `ER_LOCK_DEADLOCK` 转成 `ER_USER_LOCK_DEADLOCK`（ULL 死锁不该触发事务隐式回滚）；等待期间的**连接断开**按 KILLED 处理而非超时（与 5.7 前不同）。

#### 备份锁在 MDL 层

`BACKUP_LOCK` 属 **scoped 策略 + singleton**（与 GLOBAL/COMMIT/ACL_CACHE 一样预分配不回收）。类型只用两种：

```cpp
bool acquire_exclusive_backup_lock(...) { return acquire_mdl_for_backup(thd, MDL_SHARED, ...); }
bool acquire_shared_backup_lock(...)    { return acquire_mdl_for_backup(thd, MDL_INTENTION_EXCLUSIVE, ...); }
```

一句话原理：**利用 scoped 矩阵里"S 与 IX 互斥、IX 优先级低于 S"这一既有性质**，让稀有的备份操作用 S 免于被频繁 DDL(IX) 饿死。语义与 API 名的反转详见 [`global_lock.md`](global_lock.md)。

#### DDL 的锁升级（SU → SNW/SNRW → X）

`mysql_inplace_alter_table()` 里的分支：

| 场景 | 动作 |
|---|---|
| 需要"准备阶段独占" | `upgrade_shared_lock(ticket, MDL_EXCLUSIVE, ...)` → 准备完再 `downgrade_lock` |
| 需要"主阶段禁写" | `upgrade_shared_lock(ticket, MDL_SHARED_NO_WRITE, ...)` |
| 提交前切换表定义 | `wait_while_table_is_used()` + 升 X |
| **instant DDL** | **什么都不做**——全程只持有最初 open table 的 SU，不升级、不驱逐 TABLE 实例、不 TDC remove |

`upgrade_shared_lock` 完整代码（语义 = **重新走一遍 `acquire_lock`**（会排队、会做死锁检测），再把新 ticket 合并回老 ticket）：

```cpp
bool MDL_context::upgrade_shared_lock(MDL_ticket *mdl_ticket,
                                      enum_mdl_type new_type,
                                      Timeout_type lock_wait_timeout) {
  MDL_request mdl_new_lock_request;
  MDL_savepoint mdl_svp = mdl_savepoint();
  bool is_new_ticket;
  MDL_lock *lock;

  DEBUG_SYNC(get_thd(), "mdl_upgrade_lock");

  /* Do nothing if already upgraded. Used when we FLUSH TABLE under
     LOCK TABLES and a table is listed twice in LOCK TABLES list. */
  if (mdl_ticket->has_stronger_or_equal_type(new_type)) return false;

  MDL_REQUEST_INIT_BY_KEY(&mdl_new_lock_request, &mdl_ticket->m_lock->key,
                          new_type, MDL_TRANSACTION);
  if (acquire_lock(&mdl_new_lock_request, lock_wait_timeout)) return true;

  is_new_ticket = !has_lock(mdl_svp, mdl_new_lock_request.ticket);
  lock = mdl_ticket->m_lock;

  /* Code below assumes that we were upgrading to "obtrusive" type of lock. */
  assert(lock->is_obtrusive_lock(new_type));

  /* Merge the acquired and the original lock. */
  mysql_prlock_wrlock(&lock->m_rwlock);
  if (is_new_ticket) {
    lock->m_granted.remove_ticket(mdl_new_lock_request.ticket);
    /* We should not clear HAS_OBTRUSIVE flag in this case as we will
       get "obtrusive" lock as result in any case. */
    --lock->m_obtrusive_locks_granted_waiting_count;
  }
  /*
    Set the new type of lock in the ticket. To update state of
    MDL_lock object correctly we need to temporarily exclude ticket
    from the granted queue or "fast path" counter and then include
    lock back into granted queue.
  */
  if (mdl_ticket->m_is_fast_path) {
    /* 减 fast path 计数需在 m_rwlock 下 —— 与 granted 链表变更原子、并满足 [INV1] */
    lock->fast_path_state_add(
        -lock->get_unobtrusive_lock_increment(mdl_ticket->m_type));
    mdl_ticket->m_is_fast_path = false;
  } else {
    lock->m_granted.remove_ticket(mdl_ticket);
    if (lock->is_obtrusive_lock(mdl_ticket->m_type))
      --lock->m_obtrusive_locks_granted_waiting_count;
  }

  mdl_ticket->m_type = new_type;
  lock->m_granted.add_ticket(mdl_ticket);
  /* Since we always upgrade to "obtrusive" type of lock we need to
     increment ... HAS_OBTRUSIVE flag has been already set by acquire_lock(). */
  assert(lock->m_fast_path_state & MDL_lock::HAS_OBTRUSIVE);
  ++lock->m_obtrusive_locks_granted_waiting_count;
  mysql_prlock_unlock(&lock->m_rwlock);

  assert(is_new_ticket || !mdl_new_lock_request.ticket->m_hton_notified);
  mdl_ticket->m_hton_notified = mdl_new_lock_request.ticket->m_hton_notified;

  if (is_new_ticket) {
    m_ticket_store.remove(MDL_TRANSACTION, mdl_new_lock_request.ticket);
    MDL_ticket::destroy(mdl_new_lock_request.ticket);
  }
  return false;
}
```

三个值得注意的细节：

1. **升级的实质是"先拿一把新锁"**——所以升级会排队、会触发死锁检测、会被 `lock_wait_timeout` 打断（失败时原锁保持原状态，注释明说）。这也解释了 DDL 为什么在升级 SNRW→X 时可能长时间等待。
2. **`is_new_ticket` 分支**：若 `acquire_lock` 复用了本 context 已有的 ticket（`has_lock(svp, ...)` 判定），就不销毁；否则销毁多余的。
3. **`m_obtrusive_locks_granted_waiting_count` 先减后加**——从 fast path 计数或 granted 链表摘下旧锁（如果是 obtrusive 就 --），改类型，再挂回（新类型必是 obtrusive 就 ++）。`assert(HAS_OBTRUSIVE)` 表明全程该标志从未清掉。

对比 `downgrade_lock()` 是纯本地改类型（不重新排队、不重新检测）。

#### LOCK TABLES 走哪些 MDL（语法层就定好）

```cpp
if (lock_type >= TL_WRITE_ALLOW_WRITE)          /* ... WRITE / LOW_PRIORITY WRITE */
  mdl_lock_type = MDL_SHARED_NO_READ_WRITE;      /* SNRW */
else if (lock_type == TL_READ)                   /* ... READ LOCAL */
  mdl_lock_type = MDL_SHARED_READ;               /* SR */
else                                             /* ... READ */
  mdl_lock_type = MDL_SHARED_READ_ONLY;          /* SRO */
```

duration 为 `MDL_EXPLICIT`（跨事务，COMMIT 不释放）。

### 释放与 duration

```cpp
enum enum_mdl_duration {
  MDL_STATEMENT = 0,   // 语句结束释放
  MDL_TRANSACTION,     // 事务结束释放（DML 的 SW/SR）
  MDL_EXPLICIT,        // 显式释放：HANDLER、LOCK TABLES、GET_LOCK、GRL
};
```

三条链表 + 逆时间序的设计让 savepoint 变得极简——**savepoint 就是两个队头指针**：

```cpp
MDL_savepoint mdl_savepoint() {
  return MDL_savepoint(m_ticket_store.front(MDL_STATEMENT),
                       m_ticket_store.front(MDL_TRANSACTION));
}
void rollback_to_savepoint(const MDL_savepoint &mdl_savepoint) {
  release_locks_stored_before(MDL_STATEMENT, mdl_savepoint.m_stmt_ticket);
  release_locks_stored_before(MDL_TRANSACTION, mdl_savepoint.m_trans_ticket);
}
```

（因为"语句/事务锁总是 prepend 到链头"，所以从队头一路 pop 到哨兵即可。**显式锁不参与**——ROLLBACK TO SAVEPOINT 后 HANDLER/LOCK TABLES/GET_LOCK 仍保留。）

`release_lock()` 的两个顺序细节：① **先从 `m_ticket_store` 摘 ticket 再释放锁**（哈希索引要读 `MDL_lock::key`，而 lock 可能随后被销毁）；② X 锁的 key 要**拷贝到栈上**，因为 `MDL_lock` 对象可能在释放过程中被回收。

---

## ★ 本机制里的工程实现技法

### 一、策略对象取代子类继承

| 维度 | 教科书原型（多态） | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| scoped vs object 的差异 | 继承两个子类 `MDL_scoped_lock`/`MDL_object_lock`，虚函数分派 | **一个 `MDL_lock` + 两个 `static const MDL_lock_strategy` 实例**（矩阵/增量/4 个函数指针数据表化） | `MDL_lock` 可由 `LF_ALLOCATOR` 复用、放进 `LF_HASH` 无锁管理（虚表指针会破坏 reinit 语义）；所有锁对象大小一致；分派只一次指针间接。代价：`get_strategy()` 里写死 8 个 namespace 的 switch，新增 namespace 必须记得归类（且该处注释已滞后于代码） |
| `MDL_lock` 的可见性 | 通常放头文件 | **藏在 `mdl.cc`**（头文件只有前向声明） | 类比 TABLE_SHARE：对外只给三个句柄，内部可自由重构。代价：无法外部继承/测试（单测靠 `friend` 开洞） |

### 二、fast path = 延迟精算（lazy materialization）

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| 一个 `atomic<longlong>` 打包"状态 + 多类型计数" | `m_fast_path_state` | 高频 DML 的加锁退化成**一条 CAS**，避开全局 `m_rwlock` | 位布局要背；低位同时充当引用计数；S/SH 与 SW/SWLP 授予后不再区分 |
| "正在检查能否授予"也计入 obtrusive | `m_obtrusive_locks_granted_waiting_count` 在 `can_grant_lock()` **之前** `++` | 保证检查期间别人的 fast path CAS 必看到 `HAS_OBTRUSIVE` 并退回 slow path（[INV1] 的存在理由） | 计数语义不只是"已授予 + 已等待"，读代码容易误判 |
| 单向物化 | `materialize_fast_path_locks()` | 把精度成本从每次 DML 转移到少数 DDL | 已物化不回退；一旦表上出现过 DDL，后续全走 slow path |
| 物化水位线 `m_mat_front` | 同上 | 只扫描上次之后新增的 ticket | 与 `set_materialized()` 配对，漏调用会重复扫 |

### 三、ticket 复用与 clone

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| `find_ticket` 按"同 key 且强度 ≥"复用 | `try_acquire_lock_impl` 首步 | 同一语句对同一张表的多次请求零成本 | 强度比较要正确（`has_stronger_or_equal_type`） |
| duration 不同时 `clone_ticket()` | 同上 | 避免 HANDLER CLOSE 释放事务锁 / COMMIT 释放 HANDLER 锁 | 多一个 ticket 对象（PFS 里也会多一行） |
| 链表 >256 才建哈希 | `MDL_ticket_store` | 绝大多数连接 ticket 很少，避免哈希开销 | 阈值是经验值 |

### 四、wait-for 边的抽象与死锁检测

| 技法 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| `MDL_wait_for_subgraph` 抽象边 | 死锁检测 | 同一套 BFS+DFS 可跨资源类型工作（MDL/表刷新/从库提交顺序） | 每种新等待需自己 `will_wait_for()` + `find_deadlock()` |
| 先 BFS 探边再 DFS 递归 | `MDL_lock::visit_subgraph` | 注释称"在含环的负载下更高效" | 代码要遍历两遍队列 |
| 深度 32 **即判死锁** | `MAX_SEARCH_DEPTH` | 防递归爆栈 | 长链会**误报**（作者自承最优值未知） |
| 静态权重 victim | `get_deadlock_weight` | DDL 不可轻易回滚重来，故给它最高权重 | GLOBAL namespace 一律判 DDL 权重，源码自认是 bug（有 bug report） |
| `while` 重复检测 | `find_deadlock` | 拆的是别人的边，可能有多个环 | 最坏情况多轮搜索 |

---

## 可观测性

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `lock_wait_timeout` | **31536000（1 年）** | Session/Global | MDL/行锁等待超时（秒）。对 MDL 意味着"等 DDL 默认几乎永不超时"，生产建议调到 3~10 秒 |
| `max_write_lock_count` | **`ULONG_MAX`** | **GLOBAL only** | 连续授予这么多次 hog/piglet 后切换优先级矩阵。**默认等于关闭防饥饿**。⚠️ **同时被 MDL 与 THR_LOCK 使用**——调小会改变两套锁的调度 |
| `performance_schema_max_metadata_locks` | — | Global | PFS 的 `metadata_locks` 表容量，满了会丢行 |

### 状态变量

| 状态变量 | 说明 |
|---|---|
| `Performance_schema_metadata_lock_lost` | PFS MDL 容器**丢行**计数（`performance_schema_max_metadata_locks` 满之后新增锁没被记录的次数） |

（不存在 `Metadata_locks_cache` 之类的 MDL 状态变量——旧资料里的这类名字是错的。）

### `performance_schema.metadata_locks`

表定义在 `storage/perfschema/table_md_locks.cc`（**不是 `table_mdl.cc`**），列：

| 列 | 来源 |
|---|---|
| `OBJECT_TYPE` / `OBJECT_SCHEMA` / `OBJECT_NAME` / `COLUMN_NAME` | `MDL_key` 的四段 |
| `LOCK_TYPE` | `enum_mdl_type` 名 |
| `LOCK_DURATION` | `enum_mdl_duration` 名 |
| `LOCK_STATUS` | `PENDING` / `GRANTED` / `PRE_ACQUIRE_NOTIFY` / `POST_RELEASE_NOTIFY` |
| `SOURCE` | `MDL_REQUEST_INIT` 宏塞入的 `__FILE__, __LINE__` |
| `OWNER_THREAD_ID` / `OWNER_EVENT_ID` | 持有者线程 |

数据由 MDL 侧在 ticket 创建时通过 `mysql_mdl_create()` 写入，状态变化用 `mysql_mdl_set_status()`，销毁用 `mysql_mdl_destroy()`。**fast path 锁同样会登记**。需 `setup_instruments` 里打开 `wait/lock/metadata/sql/mdl`。

### 18 个等待状态字符串（`SHOW PROCESSLIST`）

来自 `MDL_key::m_namespace_to_wait_state_name[]`，**下标即 namespace 枚举序**——这是按 `PROCESSLIST_STATE` 反查"卡在哪类对象上"的最快方式：

| namespace | state |
|---|---|
| `GLOBAL` | `Waiting for global read lock` |
| `BACKUP_LOCK` | `Waiting for backup lock` |
| `TABLESPACE` / `SCHEMA` / `TABLE` | `Waiting for tablespace/schema/table metadata lock` |
| `FUNCTION`/`PROCEDURE`/`TRIGGER`/`EVENT` | `Waiting for stored function/procedure/trigger/event metadata lock` |
| `COMMIT` | `Waiting for commit lock` |
| `USER_LEVEL_LOCK` | **`User lock`**（故意保留旧状态名） |
| `LOCKING_SERVICE` / `SRID` / `ACL_CACHE` / `COLUMN_STATISTICS` / `RESOURCE_GROUPS` / `FOREIGN_KEY` / `CHECK_CONSTRAINT` | `Waiting for locking service / spatial reference system / acl cache / column statistics / resource groups / foreign key / check constraint lock` |

### ★ 三个实战案例（内核月报）

**案例 1：经典阻塞链——长查询 → DDL → 全表瘫痪**（《MySQL 锁问题最佳实践》）

```
Query | 6 | User sleep                          | select id, sleep(50) from t      ← 根因：长查询持 SR
Query | 4 | Waiting for table metadata lock     | alter table t add column ...     ← 卡在等 SR 释放
Query | 2 | Waiting for table metadata lock     | select * from t where id=1        ← 被 pending X 影子阻塞
Query | 1 | Waiting for table metadata lock     | update t set id=2 where id=1     ← 同上
```

诊断思路：**找最先持有或等待 MDL 的线程**（长时间查询/未提交事务），而不是盯着被阻塞的 DDL。恢复：kill 根因线程。这正是「设计思想与权衡」里"pending X 影子阻塞后续所有请求"的现场表现。

**案例 2：PFS 完整诊断**（《跟踪 Metadata lock》，5.7.6+）

```sql
UPDATE performance_schema.setup_consumers SET ENABLED='YES' WHERE NAME='global_instrumentation';
UPDATE performance_schema.setup_instruments SET ENABLED='YES' WHERE NAME='wait/lock/metadata/sql/mdl';
-- session1: set autocommit=0; select * from t limit 1;   （持 SR，TRANSACTION duration）
-- session2: truncate table t;                            （卡住）
SELECT OBJECT_TYPE, OBJECT_SCHEMA, OBJECT_NAME, LOCK_TYPE, LOCK_DURATION, LOCK_STATUS
FROM performance_schema.metadata_locks;
```

此时能看到经典的 5 行：session1 的 `TABLE/SR/TRANSACTION/GRANTED`、session2 的 `GLOBAL/IX/STATEMENT/GRANTED` + `SCHEMA/IX/TRANSACTION/GRANTED` + **`TABLE/X/TRANSACTION/PENDING`**。注意 **`LOCK_STATUS=PENDING` 且 `LOCK_TYPE` 更强的那行就是被阻塞者**，往上找同 `OBJECT_NAME` 的 `GRANTED` 行即持有者。

**案例 3：物理备份死锁——MDL 检测不到的环**（《物理备份死锁分析》）

FTWRL 的备份线程持有 `COMMIT` 锁（MDL），SQL 线程等 `COMMIT` 锁；而备份线程取位点（`SHOW SLAVE STATUS`）时需要 `rli->data_lock`（普通 mutex），该 mutex 被 SQL 线程提交时持有——**环成立但 MDL 死锁检测无效**（mutex 不在 MDL 系统里）。根因是官方 bug（SQL 线程提交时不必持有 `rli->data_lock`，已修复）。这印证了「死锁检测」末尾的结论：**跨系统环只能靠超时/排查发现**。另注意 <5.6.21 的库 kill SQL 线程会跳过事务（bug），≥5.6.21 才安全。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 谁持有/等待哪把 MDL | PFS | `performance_schema.metadata_locks`（`LOCK_STATUS` 区分 PENDING/GRANTED） |
| 谁在等、等多久 | SQL | `performance_schema.processlist` 的 `PROCESSLIST_STATE`（18 种 state 见上） |
| DDL 卡在谁身上 | SQL | `metadata_locks` 按 `OBJECT_NAME` 过滤，找 `GRANTED` 的持有者线程 |
| MDL 死锁 | SQL + 日志 | 报 `ER_LOCK_DEADLOCK`；victim 按静态权重选，DDL 几乎不会被选 |
| 内部锁对象缓存 | DBUG | `mdl_locks_unused_locks_low_water`（仅单测暴露） |

---

## Misc

### 面向二次开发

**扩展点**：新增一个 namespace 要改——`enum_mdl_namespace`、`m_namespace_to_wait_state_name[]`（源码注释明确警告"make sure to update"）、`get_strategy()` 的 switch（决定 scoped 还是 object）、若需 SE 通知则 `needs_hton_notification()` 的列表。**新增一种锁类型**要改——`enum_mdl_type`、两套策略的三个数组（granted 矩阵 ×4、waiting 矩阵 ×4×N、fast path 增量）、以及新增后必须验证 [INV2] 不变量。

**坑与已知缺陷**（源码证据）：

1. **MDL 排队问题有源码依据**：默认矩阵 #0 里 `S/SR/SW/...` 行的 X 位为 1，pending X 会阻塞后续所有请求。唯一豁免是 SH。
2. **`lock_wait_timeout` 默认 1 年**——MDL 等待几乎不超时，长事务 + 一个 pending X 就能挂住整张表。
3. **防饥饿默认关闭**（`max_write_lock_count = ULONG_MAX`），且该变量被 MDL 与 THR_LOCK 共用。
4. **`MAX_SEARCH_DEPTH=32` 会误报死锁**（深度达到即判定）。
5. **`get_deadlock_weight()` 的 TODO/FIXME**：GLOBAL namespace 一律判为 DDL 权重，源码自认 "no longer means that this is a DDL statement. There is a bug report about this."
6. **跨层环检测不到**：等 MDL → 持 InnoDB 行锁 → 等行锁 的环无人能解，只能靠超时。
7. **ACL_CACHE 的 S 锁跳过死锁检测**（性能特例，或延迟 1 秒）——注释说明这是"每连接甚至每语句都拿"的权衡。
8. **fast path 锁不入死锁图**：靠"`will_wait_for()` 前必须物化"这条不变量兜底，改代码破坏它会造成漏检。
9. **`upgrade_shared_lock` 的"只能一个 upgrader"**：由 SU/SNW/SNRW 互不兼容保证；CREATE TABLE 场景下 S 与 S 兼容，只能靠 backoff 重试兜底（注释承认）。

**社区边界澄清**：Percona 的 `lock_order` 相关增强、MariaDB 的 MDL 改进均不在社区版 8.0.39 中；`MDL_scoped_lock`/`MDL_object_lock` 子类、`Waiting_for_thr_lock`、`notify_deadlock()`、`table_mdl.cc` 都是**本版本不存在**的符号（月报/旧资料常见）。

### ★ 排查案例：卡住≠MDL

《Rename table 死锁分析》（内核月报）是个经典反例：`ALTER TABLE ... RENAME` 卡住，错误日志刷 `too many files stay open` / `problems renaming`，pstack 停在 **`fil_rename_tablespace`（InnoDB 文件层）** 而非 MDL 等待点。真实死锁是 InnoDB `fil_system` 的 IO 同步缺陷——rename 设 `stop_ios` 后等 `n_pending` 归零，master 线程提交剩余 IO 时被 `stop_ios` 挡住，又没唤醒 IO helper 线程，三方互等。修复是 rename 在 sleep 前主动 `os_aio_simulated_wake_handler_threads()`。

**教训**：rename/DDL 卡住时先看 pstack 停在哪个函数——停在 MDL 等待点才是 MDL 问题，停在 `fil_rename_tablespace` 是文件层问题。**持有 MDL 不代表死锁环里有 MDL**。

### 易混淆概念

- **MDL 的 X vs InnoDB 的 X**：同名不同物（元数据排他锁 vs 行级排他锁）。一个 DML 同时持 MDL 的 SW + InnoDB 行锁。
- **"写锁"不是 MDL 术语**：DML 用 `MDL_SHARED_WRITE`（"共享写"），DDL 用 `MDL_EXCLUSIVE`。
- **排队语义 vs 锁不兼容**：DDL 的 X 阻塞后续 SELECT，**不是因为 SR 与 X 不兼容**（它们确实不兼容，但那是 granted 矩阵），而是因为 X 在**等待队列**里通过 waiting 矩阵影子阻塞了后来的 SR。
- **`max_write_lock_count` 是系统变量**（不是编译期常量），且被 MDL 与 THR_LOCK 共用。
- **fast path 锁仍会建 ticket**，只是不进 granted 链表——"fast path 不建 ticket"是错的。
- **`S` 与 `X` 的"非对称"来自 waiting 矩阵**，不是 granted 矩阵（granted 矩阵里 S 只与 X 冲突）。

---

## 参考

**论文 / 经典算法**
- J. Gray, R. Lorie, G. Putzolu, I. Traiger. *Granularity of Locks and Degrees of Consistency in a Shared Data Base*. 1976.（多粒度封锁与意向锁；落点：IX 意向锁 + scoped/object 两级）
- J. Gray, A. Reuter. *Transaction Processing: Concepts and Techniques*. 1993.（wait-for graph 死锁检测与 victim 选择；落点：`MDL_wait_for_subgraph` + `Deadlock_detection_visitor`）

**官方文档**
- *MySQL 8.0 Reference Manual → Metadata Locking*
- *MySQL 8.0 Reference Manual → Online DDL Operations*
- *MySQL 8.0 Reference Manual → Locking Functions*（`GET_LOCK` 系列）
- WorkLog: [WL#3726](https://dev.mysql.com/worklog/task/?id=3726) DDL locking for all metadata objects（MDL 的设计）
- WorkLog: [WL#4284](https://dev.mysql.com/worklog/task/?id=4284) Transactional DDL locking（持有周期升级为事务）
- WorkLog: [WL#7304](https://dev.mysql.com/worklog/task/?id=7304) Improve MDL scalability by using lock-free hash（fast path 无锁化）

**内核月报 / 技术文章**
- [《MySQL · 特性分析 · MDL 实现分析》（数据库内核月报，2015/11）](https://www.kancloud.cn/taobaomysql/monthly/81377)——数据结构全览与加锁/死锁检测总览
- [《MySQL · 特性分析 · 跟踪 Metadata lock》（数据库内核月报，2015/10）](https://www.kancloud.cn/taobaomysql/monthly/81108)——5.5.3 持有周期语句→事务的两个问题、Oracle 对比、PFS 诊断案例
- [《MySQL · 答疑解惑 · 物理备份死锁分析》（数据库内核月报，2016/01）](https://www.kancloud.cn/taobaomysql/monthly/117960)——FTWRL 的 COMMIT 锁与 `rli->data_lock` 的跨系统环
- [《MySQL · BUG分析 · Rename table 死锁分析》（数据库内核月报，2016/03）](https://www.kancloud.cn/taobaomysql/monthly/140086)——与 MDL 无关的排查反例
- ⚠️ 以上月报基于 5.5/5.6/5.7：`MDL_scoped_lock`/`MDL_object_lock` 子类、`m_tickets` 成员、`metadata_locks_hash_instances` 分区在 8.0.39 均已不存在或重构。**本篇一律以本仓库源码为准**，月报仅作问题视角与历史脉络参考（模板 ⑩）。

**相关文档**
- InnoDB 行锁/间隙锁/死锁检测（含两层死锁检测的对比）见 [`innodb_trx_lock.md`](innodb_trx_lock.md)
- 全局锁二件套（FTWRL 的 GLOBAL+COMMIT S、备份锁的 BACKUP_LOCK namespace 与命名反转）见 [`global_lock.md`](global_lock.md)
- server 层表锁 THR_LOCK（与 MDL 的分工、`max_write_lock_count` 的另一半用途）见 [`thr_lock.md`](thr_lock.md)
- 锁全景、18 namespace 盘点与归属判据见 [`../README.md`](../README.md)
