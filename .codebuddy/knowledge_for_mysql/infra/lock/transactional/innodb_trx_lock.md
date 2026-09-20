# InnoDB 事务锁（lock_t 体系）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 `lock_t` 一统表锁/行锁的结构与 `type_mode` 位布局、record/gap/next-key/insert intention 锁语义、等待队列复用存储的挂载方式与 `os_event` 唤醒、wait-for graph 死锁检测、锁与 MVCC 版本/半一致性读的交互。
>
> **边界**：本篇讲 InnoDB 事务锁（`lock_t` 体系，行锁为主）。server 层元数据锁见 [`mdl.md`](mdl.md)；AUTOINC 锁与 R-tree 谓词锁本篇已完整展开（跨层 SQL 侧配合另见 [`../../feat/auto_increment.md`](../../../feat/auto_increment.md)、[`../../server/datatype/gis.md`](../../../server/datatype/gis.md)）；保护 `lock_sys` 内存结构的同步原语（`locksys::Latches` 分片锁）是原语、非事务锁，其盘点见 [`../README.md`](../README.md) A3 节。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- 核心实现
  - [主链路](#主链路)
  - 锁的数据结构
    - [lock_t 与 lock_sys_t](#锁的数据结构lock_t-体系)
  - 加锁流程
    - [表锁机制：意向锁与表级 S/X](#表锁机制意向锁与表级-sx)
    - [行锁四种形态的深入剖析](#行锁四种形态的深入剖析)
    - [加锁决策：从"范围关系"到锁类型](#加锁决策从范围关系到锁类型)
    - [显式锁的创建与入队](#显式锁的创建与入队)
    - [隐含锁：写操作为什么不建锁对象](#隐含锁implicit-lock写操作为什么默认不建锁对象)
    - [谓词锁（R-tree 空间索引）](#谓词锁r-tree-空间索引第四种锁语义)
  - 等待与唤醒
    - [等待队列的挂载](#等待队列的挂载复用存储--lock_wait-位)
    - [srv_slot_t 等待槽位](#srv_slot_t-等待槽位slot--thr--wait_lock-严格-111)
    - [释放入口](#释放入口提交回滚共用一条链)
    - [挂起：入睡的完整编排](#挂起入睡的完整编排)
    - [授予：lock_rec_grant_by_heap_no 逐行](#授予lock_rec_grant_by_heap_no-逐行)
    - [唤醒：最多一次的结构性保证](#唤醒最多一次的结构性保证)
  - 死锁检测
    - [后台线程主循环](#后台线程主循环1-秒轮询--事件提前唤醒)
    - [wait-for graph 的构建](#wait-for-graph-的构建)
    - [环检测算法：单出边 DFS + 着色](#环检测算法单出边-dfs--着色)
    - [victim 选择：权重最小者 + 高优先级仲裁](#victim-选择权重最小者--高优先级仲裁)
  - 锁与 MVCC / 半一致性读的交互
    - [锁与 MVCC 版本的关系](#锁与-mvcc-版本的关系正交)
    - [gap lock 与幻读](#gap-lock-与幻读)
    - [半一致性读与 gap lock 的边界](#半一致性读与-gap-lock-的边界)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- 可观测性
  - [三套观测体系的分工](#三套观测体系的分工)
  - [InnoDB Monitor 输出（锁段怎么读）](#innodb-monitor-输出锁段怎么读)
  - [PFS 锁表的实现](#pfs-锁表的实现server-定义表引擎填充行)
  - [lock 子系统计数器](#lock-子系统计数器innodb_metrics)
  - [系统变量与状态变量](#系统变量与状态变量)
  - [观测对象 → 手段 速查](#观测对象--手段-速查)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

InnoDB 的事务锁（transactional lock，区别于保护内存的同步原语）**收敛在 `lock_t` 一个结构里**——`type_mode` 按位区分表锁（`LOCK_TABLE`）与行锁（`LOCK_REC`），行锁再用标志位细分四种形态：

- **record lock**（`LOCK_REC_NOT_GAP`）：锁一条记录，防"两个事务同时改同一行"。
- **gap lock**（`LOCK_GAP`）：锁记录之前的间隙，防"间隙内 INSERT"。
- **next-key lock**（`LOCK_ORDINARY` = 0，record + gap）：上面两者的组合，RR 默认形态，防幻读。
- **insert intention lock**（`LOCK_INSERT_INTENTION`）：插入意向锁，`INSERT` 前在间隙上打标，多个插入互不阻塞、只与 gap lock 冲突。

表锁侧有 IS/IX/S/X 四种意向/常规锁，外加第五种模式 **AUTOINC**（借 `LOCK_TABLE` 承载，语句级释放，是锁机制里唯一非事务生命周期的锁）；另有 R-tree 空间索引专用的**谓词锁**（`LOCK_PREDICATE` / `LOCK_PRDT_PAGE`）——它物理上仍是 `LOCK_REC`，但语义是与 gap/next-key 并列的第四种类别。

### 用途

在事务之间协调对**数据库对象**（表、索引记录、间隙）的访问：控制并发写（record lock）与并发读一致性（gap/next-key lock 防幻读），并配套等待、唤醒、死锁检测一整套运行时机制。它不保护任何内存结构——`lock_sys` 的 hash 表、等待队列本身靠 `locksys::Latches` 分片锁保护（那是原语，见 lock 目录 README）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | `innodb_locks_unsafe_for_binlog` 控制 RC 下是否禁用 gap lock（semi-consistent read 的前身） |
| 8.0 | 移除该变量，gap lock 由隔离级别决定（`skip_gap_locks()`）；`LOCK_REC_NOT_GAP` 于 4.0.5 引入以模拟 Oracle 式 RC |
| 8.0.21 | WL#10314：`locksys::Latches` 分片锁取代单一大锁（512 表分片 + 512 页分片 + 全局闸门），锁等待从 sync array 改为 `waiting_threads` 槽位数组 + `os_event` |

---

## 理论基础

### 设计思想与权衡

**① 一个 `lock_t` 一统表锁与行锁（union + 类型位）**。表锁与行锁本是两种对象：表锁天然挂表（`dict_table_t::locks` 链表），行锁天然挂页（hash cell）。但二者共享"事务归属、等待语义、grant/wait 状态、死锁检测入口"这一整套机制，用一个结构 + union 字段（`tab_lock` / `rec_lock`）+ `LOCK_TABLE`/`LOCK_REC` 类型位区分，比两套结构省一半的队列管理、唤醒、检测代码。代价是 `lock_t` 里总有几个字段对另一种锁无意义（如行锁的 `index`、`hash` 指针）。

**② 行锁用 hash 表、表锁用链表**。行锁量大（一行一锁、一页一 bitmap），按 `(space_id, page_no)` hash 分桶、桶内单链表（`lock_t::hash` 侵入式节点），O(1) 定位到"这一页上谁持了锁"；表锁量少（每张表同一时刻只有少量意向锁），挂 `dict_table_t::locks` 双向链表即可。这是 `lock_rec_t` 里没有链表节点、`lock_table_t` 里有 `UT_LIST_NODE_T` 的原因——**数据量决定存储结构**。

**③ 等待队列不独立建，复用锁存储本身 + `LOCK_WAIT` 位**。granted 与 waiting 的 `lock_t` 在**同一个** hash cell / 链表里（granted 在头、waiting 在尾），状态差异只靠 `type_mode` 的 `LOCK_WAIT` bit 表达。被否决的方案是独立等待队列——那样每次 grant 都要跨两个结构搬运、保持一致性，成本更高。代价是"遍历一个 cell 的锁"时总要先按 bit 分类。

**④ 位图锁：一页一 bitmap，一个事务在一页上只占一个 `lock_t`**。同一个事务在同一页上锁多行，不建多个 `lock_t`，而是在锁结构尾部的 bitmap 里置多个 bit（bit 位 = 行在页内的 heap_no）。用"一页最多几百行"的空间（`n_bits` 个 bit）换"锁结构数量"的骤减——否则大事务批量 UPDATE 会产生百万级 `lock_t`。代价：① 锁的粒度只能精确到"页内第几行"，不能表达"某几列"；② bitmap 大小在建锁时定死，页上插入新行可能超界，靠 `LOCK_PAGE_BITMAP_MARGIN`（64 位预留）缓解；③ 一个事务持页上多把锁时，模式只能取最强的（不能对同一页的不同行持不同模式）。

**⑤ 死锁用检测（wait-for graph 拆环）而非预防（锁序规约）**。事务锁的"死锁"是运行时正常现象（两个事务互相等对方已持有的行），不能靠开发期约定加锁顺序消除——锁谁不是代码写的，是数据分布的动态结果。所以走"后台线程周期检测 + 回滚受害者"。这是与同步原语的根本分歧：latch 死锁是编程 bug 要预防（`latch_level_t`），事务锁死锁是业务必然要检测。代价：检测有开销（DFS 遍历等待图）、且死锁已实际发生（事务已互等一段时间）。

**权衡的另一半（失效场景与退化阈值）**：

- **gap lock 只在 RR/SERIALIZABLE 生效**，RC/RU 直接跳过（`skip_gap_locks()`）——换来插入并发，代价是幻读。
- **`innodb_deadlock_detect=OFF`** 时死锁无人拆，退化为等 `innodb_lock_wait_timeout` 超时——极端并发下超时事务堆积。
- **无锁升级（lock escalation）**：大范围 UPDATE 会持海量行锁（每个 `lock_t` 若干十字节 + bitmap），内存随锁数量线性增长——这是细粒度锁的固有代价，SQL Server/DB2 用"锁多则升级为表锁"缓解，InnoDB 选择不升级（避免升级瞬间的惊群与语义突变），代价就是锁内存不可控。
- **gap lock 依附记录**（见核心实现）⇒ 边界记录被删除并 purge 后，"锁间隙"失去锚点——这正是半一致性读与 gap lock 边界问题的根源（见核心实现最后一节）。

### 理论溯源

- **多粒度锁协议（Gray）**：IS/IX 意向锁是它的标准落地——在表（粗粒度）上放意向锁，让 DDL（X 表锁）无需检查每行就能判断"表上有没有行锁"。`lock0priv.h` 的注释直接写明"InnoDB normally acquires IS or IX locks" for tables、"S or X table locks are only acquired for LOCK TABLES"。
- **两阶段锁协议（2PL）**：锁在事务内只增不减（growing phase），commit/rollback 一次性释放（shrinking phase）——InnoDB 行锁正是事务结束才释放。
- **ARIES / ARIES-KVL**：next-key locking（key value locking）的出处——在 B-tree 上锁"键值区间"防幻读，InnoDB 把它落地为"锁记录 + 锁其前间隙"。
- **wait-for graph 环检测**：死锁检测的本质是把"谁等谁"建模成有向图找环（图论 SCC 的特例），DFS 回溯到起点即环。
- Gray & Reuter《Transaction Processing: Concepts and Techniques》：锁粒度分级、锁模式兼容矩阵、死锁检测的标准体系。

### 算法与数据结构

**锁模式**（`lock0types.h` 的 `lock_mode`）：`LOCK_IS=0` / `LOCK_IX=1` / `LOCK_S=2` / `LOCK_X=3` / `LOCK_AUTO_INC=4` / `LOCK_NONE=5`。

**兼容性矩阵**（`lock_compatibility_matrix[5][5]`，行 = 已持有，列 = 请求，true = 可兼容）：

| 已持有 \ 请求 | IS | IX | S | X | AI |
|---|---|---|---|---|---|
| **IS** | ✓ | ✓ | ✓ | ✗ | ✓ |
| **IX** | ✓ | ✓ | ✗ | ✗ | ✓ |
| **S** | ✓ | ✗ | ✓ | ✗ | ✗ |
| **X** | ✗ | ✗ | ✗ | ✗ | ✗ |
| **AI** | ✓ | ✓ | ✗ | ✗ | ✗ |

行锁只有 S/X（矩阵注释："for rows, InnoDB only acquires S or X locks"）。强度序：X > S > IX > IS，且 X > AI；AI 只与自己等价（`lock_strength_matrix`）。

**`type_mode` 位布局**（`lock_t` 的核心字段，一个 `uint32_t` 同时承载类型、模式、形态、状态）：

```
bit 14  LOCK_PRDT_PAGE      (16384)  谓词页锁（R-tree）
bit 13  LOCK_PREDICATE      (8192)   谓词锁（R-tree）
bit 11  LOCK_INSERT_INTENTION (2048) 插入意向
bit 10  LOCK_REC_NOT_GAP    (1024)   仅锁记录
bit 9   LOCK_GAP            (512)    只锁间隙
bit 8   LOCK_WAIT           (256)    等待中（未授予）
bit 5   LOCK_REC            (32)     记录锁类型
bit 4   LOCK_TABLE          (16)     表锁类型
bit 3-0 LOCK_MODE_MASK      (0xF)    模式：IS/IX/S/X/AI
```

`LOCK_ORDINARY = 0`（next-key lock = 没有任何形态位，即 record + gap 全锁）。

### 他库对比与演进动机

| 数据库 | 行锁粒度 | 幻读防护 | 死锁处理 | 锁升级 |
|---|---|---|---|---|
| **InnoDB** | 行 + 间隙（next-key） | RR 下 gap lock | wait-for graph 检测回滚 | 无 |
| **PostgreSQL** | 行（tuple 版本，锁不阻读） | REPEATABLE READ/SERIALIZABLE 用快照隔离天然防幻读 | 同 wait-for graph（`pg_locks`） | 无 |
| **Oracle** | 行（ITL + 块级事务槽） | 不防（RC 为主流，RR 语义与 MySQL 不同） | 同 wait-for graph | 无 |
| **SQL Server** | 行 → 表（自动升级） | 键范围锁（key-range lock） | wait-for graph | 有 |

演进动机：MySQL 主从复制依赖 binlog 的顺序性，RR + next-key lock 保证主库语句级串行语义在从库可重放；PG/Oracle 的 MVCC 只解决"读到一致版本"，不解决"两次范围查询结果集一致"，所以 InnoDB 才需要 gap lock。5.x 的 `innodb_locks_unsafe_for_binlog` 允许 RC 下关 gap lock（牺牲主从一致性换并发），8.0 移除它、统一由隔离级别决定——决策收敛到"隔离级别是唯一开关"。

---

## 核心实现

### 主链路

```
-- 加锁（快照读不加锁，锁定读/写才走这条）--
row_search_mvcc                         逐行扫描 + 加锁
  → sel_set_rec_lock                    决定"这条记录按范围关系加什么锁"
    → lock_clust_rec_read_check_and_lock   聚集索引
      / lock_sec_rec_read_check_and_lock   二级索引
        → lock_rec_lock                 核心入口（快路径 lock_rec_lock_fast）
          → 兼容 → 授予；冲突 → 排队（LOCK_WAIT）
            → lock_set_lock_and_trx_wait   记录 blocking_trx / wait_thr
            → lock_wait_suspend_thread     os_event 上睡眠

-- 唤醒（持有者提交/回滚释放锁时）--
lock_rec_release / lock_trx_release_locks
  → lock_rec_dequeue_from_page          把释放的锁从 hash cell 摘下
    → lock_rec_grant_by_heap_no         遍历同位置等待锁，筛选被本事务阻塞者
      → lock_grant                      授予：清 LOCK_WAIT + 唤醒线程
        → lock_reset_wait_and_release_thread_if_suspended
          → os_event_set                唤醒睡眠线程

-- 死锁检测（后台线程，与加锁并发）--
lock_wait_timeout_thread                锁等待超时/死锁检测线程
  → lock_wait_update_schedule_and_check_for_deadlocks
    → lock_wait_snapshot_waiting_threads   收集等待事务快照
    → lock_wait_build_wait_for_graph       构建 wait-for graph
    → lock_wait_find_and_handle_deadlocks  找环 + 选 victim 回滚
```

### 锁的数据结构（lock_t 体系）

`lock_t`（`lock0priv.h`）：

```cpp
struct lock_t {
  trx_t *trx;                      // 锁归属的事务
  UT_LIST_NODE_T(lock_t) trx_locks; // 事务锁链表的节点（基节点 trx->lock.trx_locks）
  dict_index_t *index;             // 行锁：所在索引
  lock_t *hash;                    // 行锁：hash cell 单链表的 next 指针
  union {
    lock_table_t tab_lock;         // 表锁：{ dict_table_t *table; UT_LIST_NODE_T(lock_t) locks; }
    lock_rec_t rec_lock;           // 行锁：{ page_id_t page_id; uint32_t n_bits; }
  };
  uint32_t type_mode;              // 类型 + 模式 + 形态 + 状态（见上文位布局）
  // PSI 字段：m_psi_internal_thread_id / m_psi_event_id（供 data_locks 表）
};
```

三个要点：

- **union**：同一 `lock_t` 要么是表锁（`tab_lock`，挂 `dict_table_t::locks` 链表）要么是行锁（`rec_lock`，挂 hash cell），由 `type_mode` 的 `LOCK_TABLE`/`LOCK_REC` 位决定读哪个分支。
- **`trx_locks` 是第二条链**：`lock_t` 同时挂在两个结构里——事务侧（`trx->lock.trx_locks`，遍历一个事务的全部锁）和锁侧（hash cell 或表链表，遍历一个对象上的全部锁）。这是"从两个视角找锁"的侵入式双链。
- **行锁 bitmap 不在结构体里**：`rec_lock.n_bits` 只记位数，bitmap 紧跟在 `lock_t` 内存之后（`RecLock::create` 按 `sizeof(lock_t) + bitmap 大小` 分配）。bit i 为 1 = 本事务锁了该页 heap_no = i 的行。

`lock_sys_t`（全局锁系统）：

```cpp
struct lock_sys_t {
  locksys::Latches latches;   // 分片锁：512 表分片 + 512 页分片 + 全局读写闸门
  hash_table_t *rec_hash;     // 行锁 hash 表（key = page_id），不含谓词锁
  hash_table_t *prdt_hash;    // 谓词锁 hash 表
  hash_table_t *prdt_page_hash; // 谓词页锁 hash 表
  Lock_mutex wait_mutex;      // 保护 waiting_threads / last_slot
  srv_slot_t *waiting_threads; // 等待线程槽位数组
  srv_slot_t *last_slot;      // 已用到的最末槽位
  bool rollback_complete;     // 恢复期回滚是否完成
  os_event_t timeout_event;   // 超时/死锁检测线程的事件
};
```

`rec_hash` 的组织（`lock0lock.h` 注释画出的结构）：

```
rec_hash[page_id]
  └─ lock_t (trx_A, granted, heap_no bit=1)
  └─ lock_t (trx_B, LOCK_WAIT, heap_no bit=1)
        └─ lock->trx = trx_B
              └─ trx_B->lock.blocking_trx = trx_A   ← "我被 A 阻塞"
              └─ trx_B->lock.wait_thr = thr_B
                    └─ thr_B->slot = srv_slot_t
```

`locksys::Latches` 分片体系（`lock0latches.h` 头注释完整描述了动机）：

```
[                        global_latch（Unique_sharded_rw_lock）                ]
                              │  S（常规） / X（stop the world）
[table shard 1] ... [table shard 512]  [page shard 1] ... [page shard 512]
      每个 shard 一把 Cacheline_padded<ib_mutex_t>，防 false sharing
```

三层组成：

**① 512 + 512 分片 mutex**。`Page_shards`：页路由用 `get_shard(page_id)`，**复用 `lock_rec_hash_value` 的 hash 函数**——注释明言"two pages which share a single lock queue fall into the same shard"，即 hash cell 与 shard 用同一个键映射，保证"锁队列在哪个 cell，哪个 shard 就护着它"（一致性靠共用 hash 函数，而非两份独立映射）。`Table_shards`：按 `table_id` hash，每表一个锁队列（`dict_table_t::locks`），几张队列归一个 shard。两套 shard 的 mutex 都做了 cache line 填充（`Cacheline_padded`），避免 512 把 mutex 相邻摆放互相踩 cache line。

**② `global_latch`（`Unique_sharded_rw_lock`）**。自身是 `Sharded_rw_lock`（内部再分片的读写锁，缓解 ARM 上 S 锁计数 cache line 争抢），外层包装用 **`thread_local` 记录本线程 S 锁落在哪个内部分片**（`m_shard_id`），`s_unlock` 时才能精确解锁同一个分片——这是"分片读写锁"做不到标准 RAII 接口的原因：解锁需要知道当初锁了哪片，只能靠 thread_local 记。也因此**该类全进程最多一个实例**（头注释明言，第二个实例会共享 thread_local 导致错乱）。X 锁路径不受分片影响（`try_x_lock`/`x_lock` 直接透传）。

**③ RAII guard 族**（`lock0guards.h`），各司其职：

- `Shard_latch_guard`：常规路径（单对象，如表锁）——先 S 锁 global，再锁单个分片 mutex。
- `Shard_latches_guard`：**需要同时操作两个对象时**（典型如 B-tree 页分裂/合并涉及两个页）。成员顺序是死锁预防的核心：

```cpp
class Shard_latches_guard {
  Global_shared_latch_guard m_global_shared_latch_guard;  // ① 先 S 锁 global
  Shard_naked_latches_guard m_shard_naked_latches_guard;  // ② 再锁两个分片
};
// Shard_naked_latches_guard 内部：
//   m_shard_mutex_1 / m_shard_mutex_2 = 按 std::less<Lock_mutex*> 排"小""大"
//   static constexpr std::less<Lock_mutex *> MUTEX_ORDER{};  ← 用指针序定全序
```

两把分片 mutex **按指针地址的全局全序**先小后大获取（`std::less<Lock_mutex*>`），所有线程遵守同一序 → 不可能互等形成环；两页恰好落同一 shard 时指针相同、天然退化为锁一次。注释还强调"初始化顺序重要：必须先 S 锁 global 再算 hash 找 shard"——反序则计算 shard 期间锁表结构可能被 X 持有着改动。
- `Global_exclusive_latch_guard`：X 锁 global，"stop the world"——用于死锁检测的快照校验、`lock_sys` 全量校验（debug 下每事务结束都校验一次，故单把大锁太慢才分片的）。
- `Global_exclusive_try_latch`：尝试 X，拿不到返回（避免特定路径阻塞）。

**演进**：8.0.21 之前是**一把大锁**（`lock_sys->mutex`）保护所有队列，WL#10314 才分片。调试模式频繁"全量校验"（X 锁 global）使单锁方案成为瓶颈，分片后常规路径只碰 1~2 个 shard。

### 加锁流程

#### 表锁机制：意向锁与表级 S/X

##### 队列结构与 count_by_mode 快路径

表锁挂在 `dict_table_t::locks`（一条双向链表），另有一个 `count_by_mode[LOCK_NUM]` 计数数组。这个数组的全部意义是**利用兼容矩阵的代数性质把 O(n) 判断变成 O(1)**：从矩阵读出 IS 只被 X 挡、IX 只被 S/X 挡，于是"要不要扫链表"退化为"两个整数是否为 0"：

```cpp
if ((mode == LOCK_IS || mode == LOCK_IX) &&
    table->count_by_mode[LOCK_S] == 0 && table->count_by_mode[LOCK_X] == 0) {
  return nullptr;   // 不可能有冲突，一次链表都不碰
}
for (lock = UT_LIST_GET_LAST(table->locks); lock != nullptr;
     lock = UT_LIST_GET_PREV(tab_lock.locks, lock)) { ... }
```

这条快路径在三处复用（冲突扫描、等待者重判、释放后授予）。**为什么值得**：OLTP 几乎永远没有表级 S/X（只有 DDL 才有），而每条 DML 都要拿一次 IX；没有快路径的话每条 DML 都要遍历表锁队列**且全程持有 table shard latch**（全局竞争点）。代价是**不对称**：一旦有 DDL 在跑就退化成 O(n²)（源码注释自己写了 `Omega(n^2)`，举例"DDL 释放 S/X 后要批量唤醒 1000 个等待者"）。

##### lock_table 的完整流程

```cpp
dberr_t lock_table(ulint flags, dict_table_t *table, lock_mode mode, que_thr_t *thr) {
  if ((flags & BTR_NO_LOCKING_FLAG) || srv_read_only_mode || table->is_temporary())
    return DB_SUCCESS;                       // ① 三处豁免
  ut_a(flags == 0);                          // ② 8.0 所有调用点都传 0，是死参数
  if (lock_table_has(trx, table, mode))      // ③ 已有更强/等价表锁 → 直接成功
    return DB_SUCCESS;
  if ((mode == LOCK_IX || mode == LOCK_X) && !trx->read_only && no rseg)
    trx_set_rw_mode(trx);                    // ④ 只读事务升级为读写事务（分配 trx id）
  locksys::Shard_latch_guard table_latch_guard{...};
  wait_for = lock_table_other_has_incompatible(trx, LOCK_WAIT, table, mode);
  if (wait_for != nullptr) err = lock_table_enqueue_waiting(mode, table, thr, wait_for);
  else                     { lock_table_create(table, mode, trx); err = DB_SUCCESS; }
}
```

要点：

- **③ 的复用检查走 `trx->lock.trx_locks` 而非 `table->locks`**，且循环条件 `lock_get_type(lock) == LOCK_TABLE` 遇第一个行锁就停——因为 `add_to_trx_locks` 刻意**表锁加到链表头、行锁加到链表尾**，所以是 O(表锁数)≈O(1)。这是精心设计的布局契约（若哪天表锁也 ADD_LAST，`lock_table_has` 会静默失效）。
- **`wait = LOCK_WAIT`**：等待中的锁也算阻塞者，这是表锁 FIFO 公平性的来源——新请求不能插队绕过已排队的人。
- **进等待的两道闸门**：`TRX_DICT_OP_*`（DDL 事务）下不允许等表锁（否则"持 dict lock 等表锁 ⊕ 持表锁等 dict lock"死锁），直接报错；`TRX_FORCE_ROLLBACK`（被 kill/判 victim）直接返回 `DB_DEADLOCK` 不入队。
- 表锁与行锁**共用同一个 wait-for 图机制**（同一 `lock_create_wait_for_edge` → `trx->lock.blocking_trx`），所以表锁↔行锁的混合死锁也能被检出。

##### 配套函数：create / other_has_incompatible / enqueue_waiting / dequeue

**`lock_table_other_has_incompatible`**（冲突扫描，含快路径）：

```cpp
static inline const lock_t *lock_table_other_has_incompatible(
    const trx_t *trx, ulint wait, const dict_table_t *table, lock_mode mode) {
  ut_ad(locksys::owns_table_shard(*table));
  /* 快路径：IS/IX 只可能被 S/X 挡，队列里没有 S/X 就不用扫 */
  if ((mode == LOCK_IS || mode == LOCK_IX) &&
      table->count_by_mode[LOCK_S] == 0 && table->count_by_mode[LOCK_X] == 0) {
    return nullptr;
  }
  for (lock = UT_LIST_GET_LAST(table->locks); lock != nullptr;
       lock = UT_LIST_GET_PREV(tab_lock.locks, lock)) {
    if (lock->trx != trx && !lock_mode_compatible(lock_get_mode(lock), mode) &&
        (wait || !lock_get_wait(lock))) {     // wait=LOCK_WAIT：等待中的也算阻塞者
      return (lock);
    }
  }
  return (nullptr);
}
```

两点：① 遍历方向是**从队尾往队头**，返回的是"最近一个"冲突者——这个返回值只用作 wait-for 出边终点与死锁报告，具体该不该等由 `lock_table_has_to_wait_in_queue` 完整判定；② `wait` 参数是 FIFO 公平性的开关（`lock_table` 传 `LOCK_WAIT`）。

**`lock_table_create`**（建锁入队，**不做兼容性检查也不做死锁检查**——纯机械入队）：

```cpp
static inline lock_t *lock_table_create(dict_table_t *table, ulint type_mode, trx_t *trx) {
  check_trx_state(trx);
  ++table->count_by_mode[type_mode & LOCK_MODE_MASK];   // 掩掉 LOCK_WAIT 等标志位

  if (type_mode == LOCK_AUTO_INC) {                     // ① AI 且直接授予：复用表级单例
    lock = table->autoinc_lock;
    ut_ad(table->autoinc_trx == nullptr);
    table->autoinc_trx = trx;
    ib_vector_push(trx->lock.autoinc_locks, &lock);
  } else if (trx->lock.table_cached < trx->lock.table_pool.size()) {
    lock = trx->lock.table_pool[trx->lock.table_cached++];   // ② 事务私有池
  } else {
    lock = static_cast<lock_t *>(mem_heap_alloc(trx->lock.lock_heap, sizeof(*lock)));  // ③ 堆
  }

  lock->type_mode = uint32_t(type_mode | LOCK_TABLE);
  lock->trx = trx;
  lock->tab_lock.table = table;

  locksys::add_to_trx_locks(lock);        // 表锁加到 trx_locks 头部
  ut_list_append(table->locks, lock);    // 追加到队尾 → 严格 FIFO

  if (type_mode & LOCK_WAIT) lock_set_lock_and_trx_wait(lock);
  MONITOR_INC(MONITOR_TABLELOCK_CREATED);
  MONITOR_INC(MONITOR_NUM_TABLELOCK);
  return (lock);
}
```

`type_mode == LOCK_AUTO_INC` 是**严格相等**——从 `lock_table_enqueue_waiting` 进来的是 `LOCK_AUTO_INC | LOCK_WAIT`，不相等，所以**等待中的 AI 锁走普通路径从 heap 分配**。这就是注释说的"we reuse the lock instance only if there is no wait involved"。

**`lock_table_enqueue_waiting`**（进等待）：

```cpp
static dberr_t lock_table_enqueue_waiting(ulint mode, dict_table_t *table,
                                          que_thr_t *thr, const lock_t *blocking_lock) {
  trx = thr_get_trx(thr);
  if (que_thr_stop(thr)) ut_error;        // ① 必须返回 false（线程本来不该停）
  switch (trx_get_dict_operation(trx)) {  // ② DDL 事务不允许等表锁
    case TRX_DICT_OP_TABLE: case TRX_DICT_OP_INDEX:
      ib::error(ER_IB_MSG_642) << "A table lock wait happens in a dictionary operation...";
      ut_d(ut_error);
  }
  if (trx->in_innodb & TRX_FORCE_ROLLBACK) return (DB_DEADLOCK);   // ③ 已被判受害者

  lock_t *lock = lock_table_create(table, mode | LOCK_WAIT, trx);
  trx->lock.que_state = TRX_QUE_LOCK_WAIT;
  trx->lock.wait_started = ...;
  auto stopped = que_thr_stop(thr);
  ut_a(stopped);                          // ④ 这次必须返回 true
  MONITOR_INC(MONITOR_TABLELOCK_WAIT);
  lock_create_wait_for_edge(lock, blocking_lock);
  return (DB_LOCK_WAIT);
}
```

`que_thr_stop` 的两次调用是理解点：进入时线程状态必须是 `QUE_FORK_ACTIVE` 且无错误 → 返回 false（否则再入队一个锁请求会泄漏）；设置 `que_state = TRX_QUE_LOCK_WAIT` 之后再调 → 命中"lock wait"分支置 `thr->state = QUE_THR_LOCK_WAIT`。**与行锁顺序相反**：行锁是 `create → lock_create_wait_for_edge → set_wait_state`（内含 que_thr_stop），表锁把建边放在 `que_thr_stop` 之后——因为 `lock_create_wait_for_edge` 断言 `wait_lock != nullptr`，必须先由 `lock_table_create` 的 LOCK_WAIT 分支设上。

**`lock_table_dequeue`**（释放并授予，含第二个快路径）：

```cpp
static void lock_table_dequeue(lock_t *in_lock) {
  ut_ad(trx_mutex_own(in_lock->trx));
  ut_ad(locksys::owns_table_shard(*in_lock->tab_lock.table));
  const auto mode = lock_get_mode(in_lock);
  const auto table = in_lock->tab_lock.table;
  lock_t *lock = UT_LIST_GET_NEXT(tab_lock.locks, in_lock);

  lock_table_remove_low(in_lock);         // 摘链 + --count_by_mode

  /* 反向快路径：释放 IS/IX 只可能影响 S/X 等待者 */
  if (!lock || ((mode == LOCK_IS || mode == LOCK_IX) &&
                table->count_by_mode[LOCK_S] == 0 &&
                table->count_by_mode[LOCK_X] == 0)) {
    return;
  }
  for (; lock != nullptr; lock = UT_LIST_GET_NEXT(tab_lock.locks, lock)) {
    lock_grant_or_update_wait_for_edge_if_waiting(lock, in_lock->trx);
  }
}
```

授予判定最终落到 `lock_table_has_to_wait_in_queue`（**扫队列中排在 wait_lock 之前的所有锁**，`if (lock == wait_lock) break;`）——这就是表锁的严格 FIFO：后来者不能插到前面等待者之前，即使它与那个 waiting 锁兼容。判定本身只有 `locksys::has_to_wait` 的两行：`lock1->trx != lock2->trx && !lock_mode_compatible(...)`，**没有 heap_no、没有 gap/II 语义、没有 bitmap**。

`lock_table_remove_low` 里有个自洽性断言 `ut_a(0 < table->count_by_mode[lock_mode])`——`count_by_mode` 必须永远非负，若 create 与 remove 不配对，debug/release 都会炸。

##### IS/IX 的获取时机

| 操作 | 表锁 | 说明 |
|---|---|---|
| 普通 SELECT（非锁定读、autocommit） | **不拿任何锁** | `select_lock_type == LOCK_NONE` → 走 ReadView，根本不调 `lock_table` |
| `SELECT ... FOR SHARE` | **IS** | `store_lock(TL_READ_WITH_SHARED_LOCKS)` → `LOCK_S` |
| `SELECT ... FOR UPDATE` | **IX** | `external_lock(F_WRLCK)` → `LOCK_X` |
| INSERT / UPDATE / DELETE | **IX** | `row_ins_step` / `row_upd_step` |
| 外键检查（父表） | **IS** | 只对父行加 S 记录锁 |
| DDL（建索引 / ALTER COMMIT） | **真 S / X** | `ddl::lock_table` |
| `LOCK TABLES t READ/WRITE` | **真 S / X** | 需 `innodb_table_locks=ON` + 非 autocommit |
| SERIALIZABLE + 显式事务的"普通" SELECT | **IS**（被升级） | 条件含 `OPTION_NOT_AUTOCOMMIT \| OPTION_BEGIN` |

**server→引擎的决策链**：`ha_innobase::store_lock`（按 `thr_lock_type` 定 LOCK_NONE/LOCK_S）→ `external_lock`（`F_WRLCK` 定 LOCK_X；SERIALIZABLE 下把 LOCK_NONE 升级为 LOCK_S；DD 表 / `no_read_locking` 强制降为 LOCK_NONE）→ 收敛到 `prebuilt->select_lock_type` 一个枚举。InnoDB 自己不判断 SQL 语义。

**为什么真正的表级 S/X 只在 `LOCK TABLES`/DDL/DISCARD 出现**：源码注释直说 "no InnoDB table lock is taken in LOCK TABLES if AUTOCOMMIT=1 ... InnoDB's table locks in that case cause VERY easily deadlocks"——真表锁会挡住所有 DML 的 IX，代价太高。这也解释了生产中"看到 TABLE LOCK 等待几乎总是 DDL / LOCK TABLES 引起"。

##### 意向锁存在的意义

行锁存在 `lock_sys->rec_hash`（按页分桶），**不挂在 `dict_table_t` 上**——想从表出发找全部行锁需遍历所有 page。有了 IX/IS，且**任何事务拿行级 X 前必先拿表级 IX**（源码多处断言强制，如 `lock_rec_lock_fast` 里 `ut_ad(lock_table_has(trx, table, LOCK_IX))`），则"表上有 X 行锁" ⟺ "表上有 IX"。于是 DDL 拿表级 X 只需扫 `table->locks` 那几个元素，**不扫任何 page**。

注意 8.0 **不存在锁升级**：一千万行 UPDATE 就是一千万个行锁 + 一个 IX（`lock_table_has_locks` 同时查 `n_rec_locks` 与 `table->locks`，正因为意向锁不能替代真实行锁计数）。

##### 表锁的授予：纯 FIFO，无 CATS

`lock_table_dequeue` 从被摘锁的后继开始沿链表逐个 `lock_grant_or_update_wait_for_edge_if_waiting`。与行锁的三条硬差异：

| 维度 | 表锁 | 行锁 |
|---|---|---|
| 授予顺序 | **纯 FIFO** | CATS 按 `schedule_weight` 降序排序 |
| 冲突判定 | 两行代码（不同事务 + 模式不兼容） | bitmap + gap/II + CATS 复杂逻辑 |
| 高优先级事务 | **不参与**（`lock_make_trx_hit_list` 对表锁直接 return，注释 "TBD: could this technique be used for table locks as well?"） | 参与 |

后果：高优先级事务被别人的**表锁**挡住时无法用常规手段标记对方为受害者，只能靠超时。另一边界：**表锁等待不计入 `Innodb_row_lock_*` 统计**（`lock0wait.cc` 只在 `thr->lock_state == QUE_THR_LOCK_ROW` 时累加）。

##### AUTOINC 锁

`LOCK_AUTO_INC`（值 4）是**第五种锁模式**，与 IS/IX/S/X 并列；它只能与 `LOCK_TABLE` 组合（`lock_table_create` 里 `type_mode | LOCK_TABLE`），所以物理上就是一张表锁，只是模式位不同——这也是 PFS 里它显示为 `LOCK_TYPE=TABLE` + `LOCK_MODE=AUTO_INC` 的原因。

**兼容矩阵的两个关键格**（`lock0priv.h`）：

| | 含义 |
|---|---|
| `AI / AI = false` | **AI 与自己不兼容** → 同一时刻一张表上只能有一个事务持有 AUTOINC 锁，其余排队等待 |
| `AI ↔ IS/IX = true`（双向） | 必须与意向锁兼容，否则持有 AI 的事务自己就插不进去（DML 必先拿 IX） |
| `AI ↔ S/X = false` | `LOCK TABLES` 的 S/X 与 AI 互斥 |

强度矩阵里 **AI 只 ≥ AI**（`lock_strength_matrix[AI][*]` 仅 `[AI][AI]=true`）——即 `lock_table_has()` 判断"本事务是否已持有不弱于 mode 的锁"时，IS/IX/S/X 都**顶替不了** AI，必须精确匹配。

矩阵上方的注释给出了存在理由：

```
Auto-increment (AI) locks are needed because of
statement-level MySQL binlog.
```

**AUTOINC 锁不是为了保护计数器本身**——计数器有独立的 `autoinc_mutex`。它保护的是「**分配权的语句级互斥**」，目的是让 SBR（statement-based replication）能重放：binlog 只记自增区间的**下界**（`handler.cc` 注释明写 "MySQL binlog only stores the min value of the autoinc interval"），slave 靠"从该值连续递增"复现；若 master 上并发语句的自增值交错，slave 就会得到不同分配结果 → 主从不一致。

###### 两条保护的界线（理解整个机制的关键）

| | `dict_table_t::autoinc_mutex` | AUTOINC 表锁 |
|---|---|---|
| 保护对象 | 计数器**数值**本身（`table->autoinc`） | 分配的**顺序/排他性** |
| 临界区 | 极短：读旧值 → 算区间 → 写新值 | 长：整个语句 |
| 三档模式 | **都拿** | mode 0/1 拿，mode 2 不拿 |
| 释放时机 | `get_auto_increment()` 末尾 | 语句结束 |

计数器语义是"**下一个要分配出去的值**"（不是已用最大值），初值来自 DD 的 `se_private_data`（`dict_table_autoinc_initialize(m_table, autoinc + 1)`——DD 存的是"上一个已用值"）。另有 `autoinc_persisted_mutex` 保护持久化计数器 `autoinc_persisted`，**独立于** `autoinc_mutex`，原因是 checkpoint 线程写 DDTableBuffer 时不能去抢 `autoinc_mutex`（注释："we can't read the 'autoinc' directly easily, because the autoinc_lock is required and there could be a deadlock"）。

###### 三档模式的选择逻辑（`ha_innobase::innobase_lock_autoinc`）

```cpp
if (table->is_intrinsic() || m_prebuilt->no_autoinc_locking) {
  lock_mode = AUTOINC_NO_LOCKING;          // ① 强制降级
}
switch (lock_mode) {
  case AUTOINC_NO_LOCKING:                 // ② mode 2
    dict_table_autoinc_lock(table);        //    仅 mutex
    break;
  case AUTOINC_NEW_STYLE_LOCKING:          // ③ mode 1
    if (sql_command == SQLCOM_INSERT || sql_command == SQLCOM_REPLACE) {
      dict_table_autoinc_lock(ib_table);
      if (ib_table->count_by_mode[LOCK_AUTO_INC]) {
        dict_table_autoinc_unlock(ib_table);   // ④ 放 mutex 再降级
      } else {
        break;                                 // 无人持锁 → 只靠 mutex
      }
    }
    [[fallthrough]];                       // ⑤ 批量插入直接降级
  case AUTOINC_OLD_STYLE_LOCKING:          // ⑥ mode 0
    error = row_lock_table_autoinc_for_mysql(m_prebuilt);
    if (error == DB_SUCCESS) dict_table_autoinc_lock(m_prebuilt->table);
    break;
}
```

| 档位 | 常量 / 官方名 | 简单 INSERT | `INSERT ... SELECT` / LOAD |
|---|---|---|---|
| 0 | `AUTOINC_OLD_STYLE_LOCKING` / traditional | 拿 AI 表锁，**语句结束**释放 | 同左 |
| 1 | `AUTOINC_NEW_STYLE_LOCKING` / consecutive | 无人持 AI 锁则只拿 mutex；否则降级 old style | 降级 old style（因 sql_command 不是 INSERT/REPLACE） |
| 2（**默认**） | `AUTOINC_NO_LOCKING` / interleaved | **完全不拿表锁**，只拿 mutex | **也不拿表锁** |

四个要点：

1. **默认 = 2**，`PLUGIN_VAR_READONLY`（只读变量，需配置文件重启）。sysvar 帮助文本里唯一的设计警告就是 `" 2 => No AUTOINC locking (unsafe for SBR)"`。三档的 official 命名（traditional/consecutive/interleaved）只在手册与 sysvar 文本里，源码里叫 OLD_STYLE / NEW_STYLE / NO_LOCKING（"TRADITIONAL" 这个词只在 `innobase_next_autoinc` 注释里出现过一次）。
2. **④ 的"放 mutex 再降级"是死锁预防**：拿 mutex 去等表锁会形成 `mutex → lock` 的顺序，与别人 `lock → mutex` 相反，必然死锁。注释原文 "Release the mutex to avoid deadlocks."。
3. **⑤ 的判据是 sql_command**，不是行数——`INSERT ... SELECT` 与 `LOAD DATA` 行数未知，为保 SBR 可重放必须独占，故直接降级。
4. **① 的两个强制降级**：`is_intrinsic()`（优化器内部临时表，连接私有）；`no_autoinc_locking`（由 SQL 层 `ha_extra(HA_EXTRA_NO_AUTOINC_LOCKING)` 设置，唯一发出者是 **数据字典写入路径** `sql/dd/impl/transaction_impl.cc`，注释动机 "It leads to increased chances of deadlocks during atomic DDL and is not really necessary for replicating data-dictionary changes"）。**这是 SQL 层用 `extra()` 反向影响 InnoDB 加锁决策的实例**。

`count_by_mode[LOCK_AUTO_INC]` 统计的是 **granted + pending**（"there can be multiple waiters"），所以 mode 1 下"看到别人有 AI 锁就降级"是**保守**的——对方只是在排队也会触发降级。这是性能取舍，非正确性问题。

###### 加锁入口与语句级释放

`row_lock_table_autoinc_for_mysql` 是拿 AI 锁的唯一入口，首行是幂等短路：

```cpp
if (trx == table->autoinc_trx) return DB_SUCCESS;   // 无锁"偷看"
```

注释论证了无锁读 `autoinc_trx` 安全：只有本线程能释放自己的锁。这个短路很重要——一条语句里每行 `write_row` 都可能走到这里。成功后 `lock_table(0, table, LOCK_AUTO_INC, thr)` 走标准表锁路径（含冲突检查、等待、死锁检测）。

**释放是语句级，不是事务级**——这是它与所有其它 InnoDB 锁最本质的区别。`lock_unlock_table_autoinc` 的 4 个调用点全在 handler 层：

| 调用点 | 时机 |
|---|---|
| `innobase_commit(..., commit_trx=false)` | **语句边界**（autocommit=off 或显式事务内的语句结束） |
| `innobase_rollback()` | 回滚**之前**（注释 "release it now before a possibly lengthy rollback"） |
| `innobase_rollback_trx()` | 纯事务回滚路径 |
| `innobase_xa_prepare()` | XA PREPARE 的"仅语句结束"分支 |

真正事务提交那一支**不调**它——`trx_commit` → `lock_trx_release_locks()` 会一次性放掉全部锁，且该函数末尾有 `ut_a(ib_vector_is_empty(trx->lock.autoinc_locks))` 断言：提交时 AI vector 必须已空。所以多语句事务里 **AI 锁每条语句末就放，其它锁留到事务提交**。

**⚠️ 一个极易写错的陷阱**：`handler::release_auto_increment()` **并不释放 InnoDB 的 AUTOINC 表锁**。InnoDB 的实现（`ha_innobase::release_auto_increment`）只清语句级计数 `trx->n_autoinc_rows`。真正的释放是**间接**的、通过 handlerton 的 commit 回调完成。

###### 单例复用与逆序释放（O(1) 的实现代价）

直接授予的 AI 锁用 `table->autoinc_lock`（表级预分配单例），不从 `trx->lock.lock_heap` 分配，原因是 `dict0mem.h` 注释原话："otherwise the lock heap would grow rapidly if we do a large insert from a select"。条件是 `type_mode == LOCK_AUTO_INC` **严格相等**——等待后授予的锁带 `LOCK_WAIT`，走 heap 分支（已授予与等待中的 AI 锁可能同时存在，单例只有一个槽位）。同时置 `table->autoinc_trx = trx`（快速判断"我是不是持有者"）、push `trx->lock.autoinc_locks`、`++count_by_mode[LOCK_AUTO_INC]`。

`trx->lock.autoinc_locks` **当栈用**：逆序释放 = 每次 pop 栈顶 O(1)；非栈顶删除（存储过程里中途 DROP 表）退化为 O(n) 扫描并把该槽位置 NULL 留**空洞**，后续 pop 需跳过（注释："With stored functions and procedures the user may drop a table within the same 'statement'"）。

释放路径还有个性能代价：`lock_unlock_table_autoinc` 因为不知道锁在哪些表上（可能跨 shard），直接上 **lock_sys 全局排他 latch**，注释承认 "Identifying and latching them in correct order would complicate this rarely-taken path"。先用 `trx->mutex` 做廉价探测（vector 是否为空），非空才上全局锁。

###### 自增值的预分配与 gap（缺号）成因

```cpp
next_value = innobase_next_autoinc(current, nb_reserved_values, increment, offset, col_max_value);
dict_table_autoinc_update_if_greater(table, next_value);   // 一次性推进 N 个
dict_table_autoinc_unlock(table);                          // mutex 到此为止
```

`nb_desired_values` 由 SQL 层算出，三种来源：① `estimation_rows_to_insert`（`ha_start_bulk_insert` 写入，但 **`INSERT ... SELECT` 与 `LOAD` 传 0 = 未知**）；② `thd->lex->bulk_insert_row_cnt`（多行 VALUES 行数）；③ **递增默认 1,2,4,...,65535**（批量插入的实际路径）。InnoDB 把它缓存进 `trx->n_autoinc_rows`（注释承认这是 hack："nb_desired_values seems to be accurate only for the first call ... and meaningless for other statements e.g, LOAD etc."），之后每行 `write_row` 成功就 `--`，用完才再向引擎要新区间。

拿到 `[first, first + N*increment)` 区间后，**SQL 层在 `Discrete_interval` 里自行发号，不再打扰引擎**——所以跨层调用每条语句只有几次，不是每行一次。

mode 0 不走这条路：置 `autoinc_last_value = 0` 强制 `write_row` 逐行推进计数器（独占表锁下安全）。**gap 的成因清单**：

1. 预分配 N 个、实际插 M < N 行（INSERT IGNORE 跳重、最后一批不足）；
2. `INSERT ... SELECT` / LOAD 的指数增长预留（1,2,4,...,65535），最后一轮必然超标——注释 "Don't go beyond a max to not reserve 'way too much'（because reservation means potentially losing unused values）"；
3. **事务/语句回滚**（自增值不回滚，见下）；
4. 显式插入更大的值触发 `set_max_autoinc` 把计数器抬高；
5. mode 2 下并发区间交错。

###### 自增值为什么不回滚

**自增值属于"表级持久化元数据"，不是事务数据**：① 它在内存 `dict_table_t::autoinc`，不属任何 trx 私有状态，回滚路径从不 touch 它；② 持久化是"**先落 redo 再插行**"（注释 "Always log the counter change first, so it won't be affected by any follow-up failure"），redo 不受事务回滚影响；③ AI 锁在回滚**之前**就被放掉。

8.0 的持久化语义（与 5.x 最大差别）：每次真正插入/更新一行都在 mtr 里写一条 `MLOG_TABLE_DYNAMIC_META` redo（`PM_TABLE_AUTO_INC`，不强制 flush），后台再写回 `mysql.innodb_dynamic_metadata`（DDTableBuffer）。崩溃恢复先回放 redo 里的 autoinc，再用 DDTableBuffer 补齐。因此 **8.0 重启后自增值不会回退到 `MAX(id)+1`**（5.x 会），也就不会重用已删除的 id。

###### 一次 INSERT 的完整跨层调用链（mode 2）

```
write_record()                                        [sql/sql_insert.cc]
  → ha_innobase::write_row()                          [handler/ha_innodb.cc]
    → handler::update_auto_increment()                [sql/handler.cc]  算 nb_desired_values
      → ha_innobase::get_auto_increment()             [虚函数下潜]
        → innobase_get_autoinc()
          → innobase_lock_autoinc()      ← 按 mode 决定"表锁 or 仅 mutex"
          → dict_table_autoinc_read()    ← 真正的值来源
          → innobase_next_autoinc()      ← 算区间上界
          → dict_table_autoinc_update_if_greater()  ← 一次推进 N 个
          → dict_table_autoinc_unlock()  ← mutex 释放（表锁仍在手）
      → Discrete_interval 登记；SBR 下记 binlog 下界
  → row_ins_step() → lock_table(0, table, LOCK_IX)    [每行]
    → row_ins_clust_index_entry_low() → dict_table_autoinc_log()  ← 写 redo
语句末：innobase_commit(commit_trx=false) → lock_unlock_table_autoinc()
        → lock_release_autoinc_locks() → lock_table_dequeue()
```

##### 表锁的边界与坑

- **AC-NL-RO 事务（autocommit 非锁定读）一颗锁都不拿**，连 IS 都没有；`lock_table_create` 首行的 `check_trx_state` 有 `ut_ad(!trx_is_autocommit_non_locking(trx))` 守卫。
- `srv_read_only_mode` / **临时表** → `lock_table` 直接返回成功不建锁（"Given limited visibility of temp-table we can avoid locking overhead"）。所以 `start_stmt` 里对临时表调 `row_lock_table` 是**看起来加锁其实没加**的迷惑代码。
- **DD 表**（数据字典表）在 SERIALIZABLE 下也强制 `LOCK_NONE`。
- `lock_table` 带 `[[nodiscard]]`，返回 `DB_LOCK_WAIT` 必须走 `run_again:` 循环重试（四个调用方同一套模板）。

#### 行锁四种形态的深入剖析

**前提**：四种形态**不是四个枚举值**，而是 `lock_mode`（S/X）之上的**三个独立标志位**——`LOCK_GAP`(512) / `LOCK_REC_NOT_GAP`(1024) / `LOCK_INSERT_INTENTION`(2048)，且 `LOCK_ORDINARY == 0`。`static_assert` 强制这三个位与 `LOCK_MODE_MASK`(0xF)、`LOCK_TYPE_MASK`(0x30) **完全不相交**，否则"模式"与"类型/精确模式"会互相污染。

`LOCK_ORDINARY == 0` 的实质是：**next-key 是缺省态/剩余态**——既无 GAP、也无 REC_NOT_GAP、也无 II 的行锁就是 next-key。所以 `lock_mode_is_next_key_lock` 的实现是 `(mode & ~LOCK_MODE_MASK) == 0` 而非 `mode & LOCK_ORDINARY`（后者恒为 0）。static_assert 的意义：若有人把 `LOCK_ORDINARY` 改成非零位，依赖它的语义会**静默反转**，钉在编译期杜绝改动。

**可观测后果**：`data_locks` 里 next-key 显示为裸的 **`X` / `S`**（无后缀），因为它没有标志位可显示。

##### record lock（`LOCK_REC_NOT_GAP`）

**加锁入口**：① 修改路径 `lock_clust_rec_modify_check_and_lock` / `lock_sec_rec_modify_check_and_lock`（配 `impl=true`，见下）；② RC 的锁定读（`skip_gap_locks()` 分支）；③ **唯一索引等值命中非 delete-mark 记录**的降级（`row_compare_row_to_range` 的 `unique_search && !rec_get_deleted_flag`——唯一性保证间隙里不可能再冒同键记录）；④ **二级索引回表时恒为 `LOCK_REC_NOT_GAP`**（用完整主键 `PAGE_CUR_LE` 精确定位，最多命中一条，源码注释 "we are searching the clust rec with a unique condition, hence..."）；⑤ 隐式锁物化。

**为什么修改路径能配 `impl=true` 省略显式锁**：隐式锁的载体是聚簇记录里的 `DB_TRX_ID`，它只能表达"这条记录当前版本被我改过"，**表达不了间隙**——所以隐式锁 ≡ `X|REC_NOT_GAP`。断言 `ut_ad(!impl || ((mode & LOCK_REC_NOT_GAP) == LOCK_REC_NOT_GAP))` 就是这条法理。`impl=true` 且页上无锁时快路径**连 `lock_t` 都不建**。

**冲突**：只与 record / next-key 冲突；**不与 gap 锁、不与 II 锁冲突**（分支 B/C）。反直觉但正确：`X,REC_NOT_GAP` 挡不住往它前面插记录。

##### gap lock（`LOCK_GAP`）

**加锁入口**：① **等值查询未命中**（`WHERE id=15` 无此行 → 在 id=20 上加 gap，护住 (10,20)）；② 降序扫描防幻读；③ 范围扫描越界但间隙相交；④ INSERT 重复键扫描的 `is_next`；⑤ **purge/页重组时的锁继承**（`lock_rec_inherit_to_gap`——被删记录上的 next-key/gap 锁继承给后继记录，**形态强制退化为 gap**；纯 record 锁不继承；WAITING 锁继承后变 GRANTED gap 锁）；⑥ next-key 拆分出的 gap 部分。

**gap 锁之间完全兼容**（分支 A）：请求带 GAP 且不带 II → **无条件 `NO_CONFLICT`**，不管对方是 gap/next-key/record/II、GRANTED 还是 WAITING。注释的哲学是 "different users can have conflicting lock types on gaps"——gap 锁唯一职责是阻止插入（只阻塞 II），不阻止任何读/写/其它 gap 锁。配合分支 B（非 II 请求遇 gap 锁放行），两个方向合起来才是完整的"**gap 锁只与 II 锁冲突**"。

**supremum 上的 gap 锁**（最易踩坑）：`RecLock::init` 与 `lock_rec_add_to_queue` 两处都对 supremum 做**归一化** `m_mode &= ~(LOCK_GAP | LOCK_REC_NOT_GAP)`——"supremum 上的锁天然就是 gap 型"，清掉标志位的好处是能**与同页其它 next-key 锁合并进同一个 `lock_t` 的 bitmap** 省内存。运行时靠 `rec_lock_check_conflict` 的 `lock_is_on_supremum` 参数识别（由 `lock_t::includes_supremum()` 算出，即 bitmap 的 supremum 位）。**副作用**：`data_locks` 里 supremum 锁显示为 **`X`（无 GAP 后缀）+ `LOCK_DATA: supremum pseudo-record`**——判断 supremum gap 锁必须看 LOCK_DATA，不能只看 LOCK_MODE。

**RC 下为何不加 gap**：四道闸门——`set_also_gap_locks=false`、`row_compare_row_to_range` 的 `skip_gap_locks` 降级、`row_sel` 里 RC 连 supremum 都跳过、`lock_rec_inherit_to_gap` 的 `!skip_gap_locks || inherit_all` 例外（以及 PREPARE 时主动释放 gap 部分）。

##### next-key lock（`LOCK_ORDINARY`）

**加锁入口**：RR 下的范围/全表锁定读，**扫描到的每条记录（含 supremum）都是 next-key**；降序游标刚打开时对后继记录；INSERT 重复扫描的等值键与 supremum。

**`lock_reuse_for_next_key_lock` 完整代码**（上节已讲思路，这里看实现）：

```cpp
static void lock_reuse_for_next_key_lock(const lock_t *held_lock, ulint mode,
                                         const buf_block_t *block, ulint heap_no,
                                         dict_index_t *index, trx_t *trx) {
  ut_ad(mode == LOCK_S || mode == LOCK_X);
  ut_ad(lock_mode_is_next_key_lock(mode));

  if (!held_lock->is_record_not_gap()) {
    ut_ad(held_lock->is_next_key_lock());   // 已是 next-key：gap 部分本来就覆盖
    return;
  }
  /* We have a Record Lock granted, so we only need a GAP Lock. We assume
  that GAP Locks do not conflict with anything. Therefore a GAP Lock
  could be granted to us right now if we've requested: */
  mode |= LOCK_GAP;
  ut_ad(nullptr == lock_rec_other_has_conflicting(mode, block, heap_no, trx).wait_for);
  if (lock_rec_has_expl(mode, block, heap_no, trx) == nullptr) {
    lock_rec_add_to_queue(LOCK_REC | mode, block, heap_no, index, trx);
  }
}
```

`ut_ad(nullptr == ...wait_for)` 这个断言就是"gap 锁不与任何东西冲突"的**代码级体现**：补一个 gap 锁永远不可能需要等待。

拆分启发式已在上节讲过，这里补两点：

- **强度语义**：`lock_rec_has_expl` 用 `p_implies_q` 判定"已持有锁是否够强"——请求 next-key 时要求既存锁**既不是 REC_NOT_GAP 也不是 GAP**，即**只有 next-key 锁能满足"请求 next-key 已持有"**（record 锁缺 gap 部分、gap 锁缺 record 部分）。这是 `LOCK_ORDINARY` 作为"两部分并集"的体现。
- **拆分的可见后果**：同一事务在同一条记录上可能出现**两把锁** `X,REC_NOT_GAP` + `X,GAP`（合起来 ≡ next-key，但物理上是两个 `lock_t`，type_mode 不同无法合并）——这是 `data_locks` 里"同一事务同一 LOCK_DATA 出现两行"的常见原因。

**RR 典型加锁范围**（`t(id PK)`，数据 10/20/30）：

| SQL | 加锁 |
|---|---|
| `WHERE id = 20 FOR UPDATE` | `X,REC_NOT_GAP`(20)（唯一等值降级） |
| `WHERE id = 15 FOR UPDATE` | `X,GAP`(20) |
| `WHERE id > 10 AND id < 30 FOR UPDATE` | `X`(20)、`X`(30)（next-key 显示裸 X） |
| `WHERE id > 100 FOR UPDATE` | `X` on supremum pseudo-record |
| 二级索引 `WHERE c = 20 FOR UPDATE` | 二级 `X`(20,20) + 回表 `X,REC_NOT_GAP`(20) |

##### insert intention lock（`LOCK_INSERT_INTENTION`）

形态固定为 `X | LOCK_GAP | LOCK_INSERT_INTENTION`（**必定带 GAP**），且只有 X 型。

**`lock_rec_insert_check_and_lock` 完整代码**：

```cpp
dberr_t lock_rec_insert_check_and_lock(ulint flags, const rec_t *rec,
                                       buf_block_t *block, dict_index_t *index,
                                       que_thr_t *thr, mtr_t *mtr, bool *inherit) {
  if (flags & BTR_NO_LOCKING_FLAG) return (DB_SUCCESS);
  dberr_t err = DB_SUCCESS;
  lock_t *lock;
  auto inherit_in = *inherit;
  trx_t *trx = thr_get_trx(thr);
  const rec_t *next_rec = page_rec_get_next_const(rec);   // ① 后继记录
  ulint heap_no = page_rec_get_heap_no(next_rec);

  {
    locksys::Shard_latch_guard guard{UT_LOCATION_HERE, block->get_page_id()};
    ut_ad(lock_table_has(trx, index->table, LOCK_IX));
    ut_ad(!dict_index_is_spatial(index));

    lock = lock_rec_get_first(lock_sys->rec_hash, block, heap_no);
    if (lock == nullptr) {
      *inherit = false;                                   // ② 无锁队列直接短路
    } else {
      *inherit = true;
      const ulint type_mode = LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION;
      const auto conflicting =
          lock_rec_other_has_conflicting(type_mode, block, heap_no, trx);
      /* LOCK_INSERT_INTENTION locks can not be allowed to bypass waiting locks,
      because they allow insertion of a record which splits the gap ... */
      ut_a(!conflicting.bypassed);                        // ③ II 永不插队
      if (conflicting.wait_for != nullptr) {
        RecLock rec_lock(thr, index, block, heap_no, type_mode);
        trx_mutex_enter(trx);
        err = rec_lock.add_to_waitq(conflicting.wait_for);
        trx_mutex_exit(trx);
      }
    }
  }
  switch (err) {
    case DB_SUCCESS_LOCKED_REC: err = DB_SUCCESS; [[fallthrough]];
    case DB_SUCCESS:
      if (!inherit_in || index->is_clustered()) break;
      page_update_max_trx_id(block, ..., trx->id, mtr);   // ④ 维护 PAGE_MAX_TRX_ID
    default: break;
  }
  return (err);
}
```

它由 `btr_cur_*_insert` 调用。三个关键点：① **加在"后继记录"上**（"我要插到这条记录前面的间隙"）；② **快路径**：后继记录上没有任何 lock_t 时连 II 锁都不建，直接插——并发 INSERT 零开销的根本原因；③ **不走 `lock_rec_lock`**（绕过拆分启发式）。

**冲突规则**（把 `X|GAP|II` 代入判定链）：与 gap 锁冲突 ✅、与 next-key 锁冲突 ✅、**不与 record 锁冲突**（分支 C，反直觉）、**不与 II 锁冲突**（分支 D）。源码注释的官方口径："a lock on a gap blocks only Insert Intention, and II is only blocked by locks on a gap. A 'lock on a gap' can be either a LOCK_GAP, or a part of LOCK_ORDINARY."

**为什么任何锁都不需要等 II 锁**：分支 B（非 II 请求遇带 GAP 位的锁放行，而 II 必带 GAP）+ 分支 D 的合力。注释点出真实动机不是"II 锁弱"，而是消除一个**假死锁**：next-key 等 II，而 II 一旦授予插入就完成、插入又死锁在那个 WAITING 的 next-key 上。

**II 不得越过 WAITING 的 gap/next-key 锁**（`ut_a(!conflicting.bypassed)`；分支 E 只对非 II 生效）——四层含义：

1. 分支 E 的插队豁免条件是 `!(type_mode & LOCK_INSERT_INTENTION)`，II 请求**永远进不去**，故 `CAN_BYPASS` 不会由 II 产生。
2. **物理后果**：若允许 II 越过 WAITING 的 next-key 锁，II 被授予后会真的插入记录，**把间隙一分为二**；插入时 `lock_rec_inherit_to_gap_if_gap_lock` 把 R 上的锁继承给新记录，**WAITING 的锁被复制一份**（继承后变 GRANTED gap 锁）。
3. 这破坏了 `trx->lock.wait_lock` 是**单指针**的不变量（死锁检测、blocking_trx 单值语义都依赖它）。
4. record/next-key 请求能插队，是因为 `has_granted_blocker` 保证"我已持有 ≥`S|REC_NOT_GAP` 的 GRANTED 锁挡着它 → 它不可能在我之前被授予"，**不涉及插入记录**，不会分裂间隙。

##### 判定原语：rec_lock_check_conflict（矩阵就是这张表的代码）

上面所有规则都出自这一个函数，贴出来对照：

```cpp
static inline Conflict rec_lock_check_conflict(const trx_t *trx, ulint type_mode,
                                               const lock_t *lock2,
                                               bool lock_is_on_supremum,
                                               Trx_locks_cache &trx_locks_cache) {
  ut_ad(lock_get_type_low(lock2) == LOCK_REC);

  if (trx == lock2->trx ||                                    // 0a 同事务
      lock_mode_compatible(LOCK_MODE_MASK & type_mode, lock_get_mode(lock2)))
    return Conflict::NO_CONFLICT;                              // 0b S↔S 直接放行

  const bool is_hp = trx_is_high_priority(trx);
  if (is_hp && lock2->is_waiting() && !trx_is_high_priority(lock2->trx))
    return Conflict::NO_CONFLICT;                              // 0c HP 插队

  if ((lock_is_on_supremum || (type_mode & LOCK_GAP)) &&
      !(type_mode & LOCK_INSERT_INTENTION))
    return Conflict::NO_CONFLICT;                              // A gap 请求与万物兼容

  if (!(type_mode & LOCK_INSERT_INTENTION) && lock_rec_get_gap(lock2))
    return Conflict::NO_CONFLICT;                              // B 非 II 请求不等 gap 锁

  if ((type_mode & LOCK_GAP) && lock_rec_get_rec_not_gap(lock2))
    return Conflict::NO_CONFLICT;                              // C gap 请求不等 record 锁

  if (lock_rec_get_insert_intention(lock2))
    return Conflict::NO_CONFLICT;                              // D 谁都不等 II 锁

  /* II 不得越过 WAITING 的 gap/next-key 锁 */
  if (!(type_mode & LOCK_INSERT_INTENTION) && lock2->is_waiting() &&
      lock2->mode() == LOCK_X && (type_mode & LOCK_MODE_MASK) == LOCK_X) {
    if (trx_locks_cache.has_granted_blocker(trx, lock2))
      return Conflict::CAN_BYPASS;                             // E 插队豁免
  }
  return Conflict::HAS_TO_WAIT;
}
```

**为什么 A 与 B 必须成对存在**：A 说"gap 锁作为**请求方**不与任何东西冲突"；B 说"gap 锁作为**既存方**只阻挡 II 请求（非 II 请求全放行）"。两个方向合起来才是完整的"gap 锁只与 II 锁冲突"。只看一个方向会得出错误结论（比如误以为 gap 锁能挡住 next-key 请求）。

**D 分支的动机不是"II 锁弱"**，注释写得很清楚：消除一个假死锁——next-key 等 II，而 II 一旦被授予、插入就完成，插入又死锁在那个 WAITING 的 next-key 上。

**E 分支只对非 II 生效**（条件里 `!(type_mode & LOCK_INSERT_INTENTION)`），所以 `CAN_BYPASS` 永远不由 II 请求产生——这正是 `lock_rec_insert_check_and_lock` 里 `ut_a(!conflicting.bypassed)` 能成立的原因。

##### 四种形态 × 冲突矩阵（源码判定版）

记号 **W** = 必须等待，**–** = 放行。行 = 请求方，列 = 队列中既存锁。前提：同一 `(page_id, heap_no)`、不同事务、S↔X 或 X↔X（S↔S 直接放行）。

| 请求 ↓ \ 既存 → | R：`X,REC_NOT_GAP` | G：`X,GAP` | N：`X`（next-key） | I：`X,GAP,INSERT_INTENTION` |
|---|---|---|---|---|
| **R** | **W** | – | **W** | – |
| **G** | – | – | – | – |
| **N** | **W** | – | **W** | – |
| **I** | – | **W** | **W** | – |

特例：N/G 请求打在 **supremum** 上 → 整行全 `–`（分支 A，永不等待）；R 请求打在 supremum 上不可能（`ut_ad` 禁止）。

换成"既存锁阻塞哪些请求"更好记：**record 锁阻塞 R/N；gap 锁只阻塞 I；next-key 锁阻塞 R/N/I（两者之并）；II 锁不阻塞任何东西。**

##### 一个禁止的组合：GAP | REC_NOT_GAP 不能共存

位层面可行（三个取位函数是独立按位与），但语义矛盾（"只锁间隙" ∧ "只锁记录" = 既锁间隙又锁记录 = 应是 next-key，而 next-key 恰是两位都为 0）。由 `lock_rec_lock` / `lock_rec_lock_fast` / `lock_rec_lock_slow` 三处 `ut_ad`（要求精确模式位等于三者之一）+ supremum 归一的 `ut_ad` 联合禁止。若真出现，它在复用判定里"比任何单一形态都弱"（永远无法复用），且会让 II 请求被误判为 `NO_CONFLICT` 而**错误地允许插入**。

#### 加锁决策：从"范围关系"到锁类型

锁类型不是随便选的，由 `row_compare_row_to_range` 的返回值驱动。它产出三个布尔（`row_to_range_relation_t`），语义都是**"保守为 true"**——`true` = 无法排除相关，`false` = 确定无关：

```cpp
struct row_to_range_relation_t {
  bool row_can_be_in_range;       // 记录可能在扫描范围内？
  bool gap_can_intersect_range;   // 记录前的间隙可能与范围相交（可能被并发插入）？
  bool row_must_be_at_end;        // 记录恰好是范围终点？（优化信息）
};
```

"保守为 true"是防幻读的根本要求：凡是拿不准的场景一律按"相关"处理，宁多锁、勿漏锁。`gap_can_intersect_range = false` 的直接判定有五个条件（满足任一即确认间隙无关）：

1. 本扫描模式不需要 gap 锁；
2. `trx->skip_gap_locks()`（RC/RU）；
3. **唯一键等值搜索找到未删除记录**——唯一性保证该记录前不可能再插入同键，锁记录本身即可（delete-marked 记录除外，它可能被 purge，仍须锁 gap）；
4. R-tree（无 gap 语义）；
5. **聚集索引 `>=` 边界恰好命中**——如 `WHERE pk >= 100` 且找到 `pk=100` 的记录，其前间隙插入任何值都不在范围内，注释里给了这个具体例子。

`row_can_be_in_range = false` 的判定刻意收窄到"最常规、可推理"的场景（注释原话 "we limit ourselves just to the cases, which are at the same common, tested, actionable and easy to reason about"）：只有聚集索引 + 正序扫描（`PAGE_CUR_GE/G` + `direction 0/NEXT`）+ 读范围时才推理——降序/混合升降索引 + 二级索引重复键会让推理爆炸。`HANDLER` 接口直接全部返回"未知"，退化到保守加锁。而半一致性读的注释在此处给出了那段著名的断言：`ut_ad(!trx->skip_gap_locks())`——"semi-consistent read 的证明依赖不用 gap 锁"（见「锁与 MVCC 交互」节的边界分析）。

两个布尔到锁类型的映射（`row_search_mvcc` 里的调用处）：

```cpp
if (row_to_range_relation.row_can_be_in_range) {
  lock_type = row_to_range_relation.gap_can_intersect_range
                  ? LOCK_ORDINARY        // 记录要锁 + 间隙可能被插入
                  : LOCK_REC_NOT_GAP;    // 记录要锁 + 间隙无关
} else {
  if (row_to_range_relation.gap_can_intersect_range) {
    lock_type = LOCK_GAP;                // 记录已越界 + 间隙仍可能被插入 → 只锁 gap
  } else {
    err = DB_RECORD_NOT_FOUND;           // 都不相关 → 完全不锁
    goto normal_return;
  }
}
```

semi-consistent read 场景（`ROW_READ_TRY_SEMI_CONSISTENT` + 非唯一搜索 + 聚集索引 + 非高优先级）会改用 `SELECT_SKIP_LOCKED`——注释明言"反正不会等，别浪费在创建 WAITING 锁上"。

`sel_set_rec_lock` 是**分发器而非决策器**（锁类型是入参）：按索引类型三分支——聚集索引 → `lock_clust_rec_read_check_and_lock`；普通二级索引 → `lock_sec_rec_read_check_and_lock`；空间索引 → 禁止 `LOCK_GAP`/`LOCK_ORDINARY`（R-tree 会分裂，"前一个间隙"语义不稳定）走 `sel_set_rtr_rec_lock`。入口还有一道资源保护：本事务显式锁链超 10000 且 buffer pool 吃紧时直接报 `DB_LOCK_TABLE_FULL`——锁对象占内存（含 bitmap），内存紧张时停止累积。

两条 check_and_lock 路径的骨架一致，三个关键点：

1. **先物化隐含锁、再拿 shard latch**——隐含锁检查（读 `DB_TRX_ID`、二级索引要回表）**不能**持有页 shard latch（回表要拿别的页 latch，先拿 shard 违反锁序），转换完成后再持 shard 做显式判断与入队。
2. 请求 X 前必须已持表的 IX、请求 S 前必须已持 IS——表级意向锁与行锁的层级契约（断言强制）。
3. 都以 `impl=false` 收口到 `lock_rec_lock`：读路径需要显式锁对象，让其他事务的冲突扫描能看到它。

二级索引的隐含锁检查是唯一实质差异：二级记录无 `DB_TRX_ID`，先用页头 `PAGE_MAX_TRX_ID` 粗筛（`can_older_trx_be_still_active`，只可能假阳性不可能假阴性），再 `row_vers_impl_x_locked` **回表**——以二级记录为入口找聚簇记录，在 undo 版本链上找"活跃事务对二级索引的修改"。找不到聚簇记录时必无隐含锁（活跃事务改二级索引必先改聚簇索引，回滚顺序先二级后聚簇，这个顺序保证推理成立）。

#### 显式锁的创建与入队

**`lock_rec_lock` 是全部行锁的公共入口**——修改路径（`lock_clust_rec_modify_check_and_lock` / `lock_sec_rec_modify_check_and_lock`）与锁定读路径（`lock_clust_rec_read_check_and_lock` / `lock_sec_rec_read_check_and_lock`）四条路都收敛到这里；**唯一的例外是 INSERT**（走 `lock_rec_insert_check_and_lock`，因为它要加的是"后继记录上的 II 锁"，不参与拆分启发式）。先贴本体：

```cpp
static dberr_t lock_rec_lock(bool impl, select_mode sel_mode, ulint mode,
                             const buf_block_t *block, ulint heap_no,
                             dict_index_t *index, que_thr_t *thr) {
  ut_ad(locksys::owns_page_shard(block->get_page_id()));
  ut_ad(!srv_read_only_mode);
  /* ① 行锁只可能是 S 或 X（行不存 IS/IX/AUTO_INC） */
  ut_ad((LOCK_MODE_MASK & mode) == LOCK_S || (LOCK_MODE_MASK & mode) == LOCK_X);
  /* ② 精确模式位只能是三者之一（next-key = 0） */
  ut_ad(mode - (LOCK_MODE_MASK & mode) == LOCK_GAP ||
        mode - (LOCK_MODE_MASK & mode) == LOCK_REC_NOT_GAP ||
        mode - (LOCK_MODE_MASK & mode) == 0);
  ut_ad(index->is_clustered() || !dict_index_is_online_ddl(index));
  /* ③ 隐含锁 = X|REC_NOT_GAP，故只有 REC_NOT_GAP 请求才允许省略显式锁 */
  ut_ad(!impl || ((mode & LOCK_REC_NOT_GAP) == LOCK_REC_NOT_GAP));
  switch (lock_rec_lock_fast(impl, mode, block, heap_no, index, thr)) {
    case LOCK_REC_SUCCESS:
      return (DB_SUCCESS);
    case LOCK_REC_SUCCESS_CREATED:
      return (DB_SUCCESS_LOCKED_REC);
    case LOCK_REC_FAIL:
      return (lock_rec_lock_slow(impl, sel_mode, mode, block, heap_no, index, thr));
    default:
      ut_error;
  }
}
```

逐条看断言，它们是理解整个行锁体系的钥匙：

- **①** 行锁的强模式只有 S/X 两种；IS/IX/AUTO_INC 是表锁专属（`lock0priv.h` 矩阵注释："for rows, InnoDB only acquires S or X locks"）。
- **②** `mode - (LOCK_MODE_MASK & mode)` 是把"强模式位"减掉、剩下精确模式位的写法，要求它等于 `LOCK_GAP` / `LOCK_REC_NOT_GAP` / `0` 三者之一——即**三者互斥**，`LOCK_ORDINARY` 就是那个 `0`。
- **③** 这条已在「隐含锁」节展开——`impl` 与"能否省略显式锁"的法理依据。

返回值三态是调用方语义的来源：`DB_SUCCESS`（本来就有足够强的锁，什么都没做）、`DB_SUCCESS_LOCKED_REC`（**新建了锁对象**——修改路径会把它转成 `DB_SUCCESS`，但 `releases_non_matching_rows` 等场景需要知道"新锁落下了"）、`DB_LOCK_WAIT`（已入队等待，上层需 `lock_wait_suspend_thread`）。

`lock_rec_lock` 的 `impl` 参数是全链路最核心的语义之一：

```cpp
/* Implicit locks are equivalent to LOCK_X|LOCK_REC_NOT_GAP, so we can omit
creation of explicit lock only if the requested mode was LOCK_REC_NOT_GAP */
ut_ad(!impl || ((mode & LOCK_REC_NOT_GAP) == LOCK_REC_NOT_GAP));
```

`impl=true` 只在**修改路径**使用（`lock_*_modify_check_and_lock`）：调用方即将改记录、会在记录头写下自己的 trx_id 天然构成隐含 X 锁，所以无冲突时省掉显式锁对象。但隐含锁等价于 `LOCK_X|LOCK_REC_NOT_GAP`（不锁 gap）——请求 `LOCK_ORDINARY` 时 gap 部分无人兜底，故断言禁止。读路径一律 `impl=false`。

**快路径 `lock_rec_lock_fast`**（完整代码，注释直说它只覆盖最常见的两种情形）：

```cpp
static inline lock_rec_req_status lock_rec_lock_fast(
    bool impl, ulint mode, const buf_block_t *block, ulint heap_no,
    dict_index_t *index, que_thr_t *thr) {
  lock_t *lock = lock_rec_get_first_on_page(lock_sys->rec_hash, block);
  trx_t *trx = thr_get_trx(thr);
  ut_ad(!trx_mutex_own(trx));
  lock_rec_req_status status = LOCK_REC_SUCCESS;

  if (lock == nullptr) {
    if (!impl) {                                  // 页上零锁：必无冲突
      RecLock rec_lock(index, block, heap_no, mode);
      trx_mutex_enter(trx);
      rec_lock.create(trx);
      trx_mutex_exit(trx);
      status = LOCK_REC_SUCCESS_CREATED;
    }
    /* impl=true 时连 lock_t 都不建 —— 靠记录上的 trx_id 兜底 */
  } else {
    trx_mutex_enter(trx);
    if (lock_rec_get_next_on_page(lock) != nullptr ||   // 页上不止一个锁对象
        lock->trx != trx ||                             // 不是本事务的
        lock->type_mode != (mode | LOCK_REC) ||         // 形态不完全一致
        lock_rec_get_n_bits(lock) <= heap_no) {         // bitmap 装不下本记录
      status = LOCK_REC_FAIL;                            // → 转慢路径
    } else if (!impl) {
      if (!lock_rec_get_nth_bit(lock, heap_no)) {        // 复用：只置一个 bit
        lock_rec_set_nth_bit(lock, heap_no);
        status = LOCK_REC_SUCCESS_CREATED;
      }
    }
    trx_mutex_exit(trx);
  }
  return (status);
}
```

它只处理两个"最常规"情形：页上零显式锁（必无冲突，直接创建）；或页上仅一个锁、且**同时**满足"属于本事务 + type_mode 完全一致 + bitmap 容纳得下 heap_no"（只置一个 bit 完事）。四个条件任一不满足就转慢路径——这就是注释说的 "there are no explicit locks on the page, or there is just one lock, owned by this transaction, and of the right type_mode"。

注意两点：① **`impl=true` 且页上无锁时连 `lock_t` 都不创建**，直接返回 SUCCESS——完全依赖记录上的 trx_id；② 全程只持页 shard latch（调用方保证），`trx->mutex` 只在动事务私有结构时才短暂获取。

**慢路径 `lock_rec_lock_slow`**（完整代码，Next-Key 拆分启发式的注释是全链最有价值的设计意图之一）：

```cpp
static dberr_t lock_rec_lock_slow(bool impl, select_mode sel_mode, ulint mode,
                                  const buf_block_t *block, ulint heap_no,
                                  dict_index_t *index, que_thr_t *thr) {
  trx_t *trx = thr_get_trx(thr);
  ut_ad(sel_mode == SELECT_ORDINARY ||
        (sel_mode != SELECT_ORDINARY && !trx_is_high_priority(trx)));

  /* A very common type of lock in InnoDB is "Next Key Lock", which is almost
  equivalent to two locks: Record Lock and GAP Lock separately.
  Thus, in case we need to wait, we check if we already own a Record Lock,
  and if we do, we only need the GAP Lock.
  We don't do the opposite thing (of checking for GAP Lock, and only requesting
  Record Lock), because if Next Key Lock has to wait, then it is because of a
  conflict with someone who locked the record, as locks on gaps are compatible
  with each other, so even if we have a GAP Lock, narrowing the requested mode
  to Record Lock will not make the conflict go away.

  In current implementation locks on supremum are treated like GAP Locks,
  in particular they never have to wait for anything ..., so there is no gain
  in using the above "lock splitting" heuristic for locks on supremum. */

  auto checked_mode =
      (heap_no != PAGE_HEAP_NO_SUPREMUM && lock_mode_is_next_key_lock(mode))
          ? mode | LOCK_REC_NOT_GAP      // ① 收缩：先查"记录部分"
          : mode;

  const auto *held_lock = lock_rec_has_expl(checked_mode, block, heap_no, trx);
  if (held_lock != nullptr) {
    if (checked_mode == mode) return (DB_SUCCESS);   // ② 已持有足够强：啥也不做
    ut_ad(!impl);                                    // ③ 能到这儿说明请求是 next-key
    lock_reuse_for_next_key_lock(held_lock, mode, block, heap_no, index, trx);
    return (DB_SUCCESS);
  }

  const auto conflicting = lock_rec_other_has_conflicting(mode, block, heap_no, trx);
  if (conflicting.wait_for != nullptr) {
    switch (sel_mode) {
      case SELECT_SKIP_LOCKED: return (DB_SKIP_LOCKED);
      case SELECT_NOWAIT:      return (DB_LOCK_NOWAIT);
      case SELECT_ORDINARY:
        RecLock rec_lock(thr, index, block, heap_no, mode);
        trx_mutex_enter(trx);
        dberr_t err = rec_lock.add_to_waitq(conflicting.wait_for);
        trx_mutex_exit(trx);
        ut_ad(err == DB_SUCCESS_LOCKED_REC || err == DB_LOCK_WAIT ||
              err == DB_DEADLOCK);
        return (err);
    }
  }
  /* In case we've used a heuristic to bypass a conflicting waiter, we prefer to
  create an explicit lock so it is easier to track the wait-for relation.*/
  if (!impl || conflicting.bypassed) {               // ④ 正常授予
    lock_rec_add_to_queue(LOCK_REC | mode, block, heap_no, index, trx);
    return (DB_SUCCESS_LOCKED_REC);
  }
  return (DB_SUCCESS);                               // ⑤ impl=true 且无冲突：不建锁
}
```

五个分支各自的含义：

- **① 收缩检查**：next-key ≈ 记录锁 + gap 锁。请求 `LOCK_ORDINARY` 时先只查"记录部分是否已持有"，有则只需补 gap。
- **②** `checked_mode == mode`（即请求本就是 REC_NOT_GAP，或打在 supremum 上）时已持有就返回，**注意返回 `DB_SUCCESS` 而非 `DB_SUCCESS_LOCKED_REC`**——没创建新锁。
- **③** 走到这说明请求是 next-key、`impl` 必为 false（隐含锁模拟不了 next-key）。
- **④** 无冲突时建锁入队。**`conflicting.bypassed` 是例外**：即使 `impl=true` 也强制显式建锁。
- **⑤** `impl=true` 且无冲突 → 不建任何锁对象。

Next-Key ≈ 记录锁 + gap 锁两个独立锁。请求 `LOCK_ORDINARY` 时先收缩检查"记录部分是否已持有"：已有记录锁则只需补 gap——而 **gap 锁互不冲突、永不等待**，补上即秒成功，把大量本要等待的请求变成无等待授予。**为何不做反向拆分**（有 gap 只请求记录锁）：注释指出 Next-Key 要等必因"锁记录的人"冲突（gap 互相兼容），持有 gap 锁缩小请求消不掉冲突。**supremum 例外**：supremum 上的锁一律按 gap 处理、永不等待，拆分无收益且省掉特殊分支。

冲突扫描 `lock_rec_other_has_conflicting` 基于 `rec_lock_check_conflict` 的三值判定（`Conflict::HAS_TO_WAIT / NO_CONFLICT / CAN_BYPASS`），NO_CONFLICT 的规则组是锁兼容性矩阵的完整实现——除"同事务、强模式兼容、高优先级插队"外，**gap 相关三连**：① 请求是 gap 型（非 II）→ 永不等待（"different users can have conflicting lock types on gaps"）；② 记录锁 vs 对方 gap → 不等待；③ gap vs 对方 REC_NOT_GAP → 不等待。以及**对方是 II 锁 → 永不等待**——注释解释这消除经典伪死锁：next-key 等 II，而 II 的插入又死锁在等待的 next-key 上。

**CAN_BYPASS** 是最精妙的一支：我方与对方都是 X 型记录锁、对方 WAITING、且我方已持有一个 granted 锁正在阻塞对方（`Trx_locks_cache::has_granted_blocker`）。此时按 FIFO 排它后面会**自我死锁**（我等一个在等我的人）；对方尚未授予，跳过它不破坏已授予锁语义。跳过之后即使 `impl=true` 也强制创建显式锁——"we prefer to create an explicit lock so it is easier to track the wait-for relation"，否则被绕过的等待者醒来授予时看不到我们。

`sel_mode` 三分支（`SELECT_ORDINARY / SELECT_SKIP_LOCKED / SELECT_NOWAIT`，注意枚举名是 `SELECT_NOWAIT`）：冲突时分别"入队等待 / 返回 `DB_SKIP_LOCKED` 跳下一行 / 返回 `DB_LOCK_NOWAIT` 报错"。高优先级事务断言只走 ORDINARY（要用插队逻辑）。

**锁对象的创建 `RecLock::create`**：bitmap 大小 `lock_size(page) = 1 + ((n_recs + LOCK_PAGE_BITMAP_MARGIN) / 8)` 字节，`n_recs = page_dir_get_n_heap(page)`，bitmap 紧随 `lock_t` 尾部。`LOCK_PAGE_BITMAP_MARGIN = 64` 给将来插入该页的记录预留 bit——否则页每插一条新行都可能迫使既有锁重建扩容（注意 `lock_rec_t` 结构注释里写的 "8" 是过时注释，以常量 64 为准）。supremum 上清掉 `LOCK_GAP|LOCK_REC_NOT_GAP` 位——"supremum 上的锁自动就是 gap 锁"，归一化后同事务的 `LOCK_S` 与 `LOCK_S|LOCK_GAP` 能复用同一个锁对象省内存。

`lock_alloc` 的内存来源：优先**事务私有锁池** `trx->lock.rec_pool`（预分配槽 + `rec_cached` 计数，免锁免全局堆碎片）；池耗尽或 bitmap 超 `REC_LOCK_SIZE` 时回退到事务堆 `trx->lock.lock_heap`（`mem_heap_alloc`，事务结束整堆释放）。锁对象是事务内高频小分配，私有池是标配。

**入队 `lock_rec_add_to_queue`**（完整代码，含锁对象复用）：

```cpp
static void lock_rec_add_to_queue(ulint type_mode, const buf_block_t *block,
                                  const ulint heap_no, dict_index_t *index,
                                  trx_t *trx, const bool we_own_trx_mutex = false) {
  /* DEBUG 不变量：非 WAIT 且非 GAP（即 S/X 的 REC_NOT_GAP 或 ORDINARY）时，
     队列中不允许存在其他事务的相反模式锁请求 */
  type_mode |= LOCK_REC;

  if (heap_no == PAGE_HEAP_NO_SUPREMUM) {          // ① supremum 归一化
    ut_ad(!(type_mode & LOCK_REC_NOT_GAP));
    type_mode &= ~(LOCK_GAP | LOCK_REC_NOT_GAP);
  }

  if (!(type_mode & LOCK_WAIT)) {                  // ② 只有 granted 才复用
    lock_t *const first_lock = lock_rec_get_first_on_page(hash, block);
    if (first_lock != nullptr) {
      bool found_waiter_before_lock = false;
      lock_t *lock = lock_rec_find_similar_on_page(type_mode, heap_no, first_lock,
                                                   trx, found_waiter_before_lock);
      if (lock != nullptr) {
        /* 只有 bitmap 为空（不属于任何队列）时才安全移动到队头 */
        ut_ad(!found_waiter_before_lock || (ULINT_UNDEFINED == lock_rec_find_set_bit(lock)));
        if (!lock_rec_get_nth_bit(lock, heap_no)) {
          lock_rec_set_nth_bit(lock, heap_no);     // 复用：只置一个 bit
          if (found_waiter_before_lock) {
            lock_rec_move_granted_to_front(lock, RecID{lock, heap_no});
          }
        }
        return;
      }
    }
  }
  RecLock rec_lock(index, block, heap_no, type_mode);   // ③ 复用失败才新建
  if (!we_own_trx_mutex) trx_mutex_enter(trx);
  rec_lock.create(trx);
  if (!we_own_trx_mutex) trx_mutex_exit(trx);
}
```

匹配复用靠 `lock_rec_find_similar_on_page`（`lock->trx == trx && lock->type_mode == type_mode && heap_no < n_bits`），它顺带输出 `found_waiter_before_lock`——用于那个边角情形：B-tree 重组可能清空某锁的 bitmap 并清 LOCK_WAIT，使"曾是 waiting 的锁"残留在 granted 区之后；复用它时若前面有 waiting 锁，必须 `lock_rec_move_granted_to_front` 摘到桶头，**只有 bitmap 为空（不属于任何队列）时才安全移动**，否则破坏队列顺序甚至让遍历死循环。

granted 请求先在本页 hash 链找"同事务 + 同 type_mode + bitmap 容纳得下"的锁（`lock_rec_find_similar_on_page`），找到只置一个 bit——这就是"一个事务一页通常只有极少 `lock_t`"的原因。一个大注释交代了边角情形：B-tree 重组可能清空某锁的 bitmap 并清 LOCK_WAIT，使"曾是 waiting 的锁"残留在 granted 区之后；复用它时若发现前面有 waiting 锁，必须 `lock_rec_move_granted_to_front` 摘到桶头——**只有 bitmap 为空（不属于任何队列）时才安全移动**，否则破坏队列顺序甚至让遍历死循环。waiting 锁则走 `RecLock::create` + `lock_add`：granted 插桶头、waiting 插桶尾，维持"granted 全在 waiting 前"的不变量。

**`lock_set_lock_and_trx_wait` 的硬断言**：

```cpp
ut_a(trx->lock.wait_lock == nullptr);   // 注意是 ut_a：release 也生效
trx->lock.wait_lock = lock;
trx->lock.wait_lock_type = lock_get_type_low(lock);
lock->type_mode |= LOCK_WAIT;
```

**一个事务同一时刻至多一个等待锁**——死锁检测与唤醒依赖"trx ↔ 唯一 wait_lock"建立唯一的 wait-for 边；等待队列中同一事务也只能有一个 WAITING 锁对象（与"II 不得越过 WAITING gap/next-key 锁"的规则呼应：插队会让 gap 分裂复制出等待锁，破坏该不变量）。写 `wait_lock` 受 trx->mutex 保护、改 `type_mode` 受 shard 保护，两边同时持有才能原子完成状态转移。

#### 隐含锁（implicit lock）：写操作为什么默认不建锁对象

**为什么需要它**：修改一条记录时若每次都建一个显式 `lock_t`，大事务会产生百万级锁对象（内存 + 遍历代价）。InnoDB 的做法是**不建锁**——记录头的 `DB_TRX_ID`（谁最后改的）本身就是一把"看不见的 X 锁"。只有当**别人来检查冲突**时，才把它"物化"成显式锁。这是锁系统的核心省内存设计。

```cpp
void lock_rec_convert_impl_to_expl(const buf_block_t *block, const rec_t *rec,
                                   dict_index_t *index, const ulint *offsets) {
  if (index->is_clustered()) {
    trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);  // 直接读 DB_TRX_ID
    trx = trx_rw_is_active(trx_id, true);                        // 确认事务仍活跃
  } else {
    trx = lock_sec_rec_some_has_impl(rec, index, offsets);       // 二级索引：需回表
  }
  if (trx != nullptr) {
    ut_ad(trx_is_referenced(trx));
    lock_rec_convert_impl_to_expl_for_trx(block, rec, index, offsets, trx, heap_no);
  }
}
```

聚簇索引路径是 O(1)：`lock_clust_rec_some_has_impl` 直接 `return row_get_rec_trx_id(...)`，再由 `trx_rw_is_active` 确认——它先做**无锁 min_id 快筛**（`trx_id < shard.active_rw_trxs.peek().min_id()` 则必然不在，由 min_id 维护临界区保证 happens-before），否则持 shard mutex 精确查找。`do_ref_count=true` 顺便加引用计数（保证物化期间 trx 对象不被释放）。

**二级索引没有 `DB_TRX_ID`，只能"回表推断"**：先用页头 `PAGE_MAX_TRX_ID` 粗筛（`can_older_trx_be_still_active`，注释明言 "can return false positives but never false negatives"），再 `row_vers_impl_x_locked` **沿聚簇记录的 undo 版本链**找"活跃事务对这条二级索引的修改"，且**刻意不读聚簇记录的当前版本**（只读历史版本）。找不到聚簇记录时必无隐含锁——因为活跃事务改二级索引必先改聚簇索引，且回滚顺序是先二级后聚簇（这个顺序保证推理成立）。

**物化本体**（`lock_rec_convert_impl_to_expl_for_trx`）做最终双重确认后才建锁：

```cpp
if (!trx_state_eq(trx, TRX_STATE_COMMITTED_IN_MEMORY) &&
    !lock_rec_has_expl(LOCK_X | LOCK_REC_NOT_GAP, block, heap_no, trx)) {
  lock_rec_add_to_queue(LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP, block, heap_no, index, trx, true);
}
```

**物化出的锁形态恒为 `LOCK_X | LOCK_REC_NOT_GAP`**，三个层层递进的理由：

1. **载体的表达能力上限**：trx_id 只能编码"某个事务改过这个版本"，**无法编码间隙**——gap 是区间概念，页内没有任何字段能表达它。所以物化只可能是 record 部分。
2. **恒为 X**：能写 trx_id 的唯一操作是修改（INSERT/UPDATE/DELETE 写 undo），修改必然排他，故无 S 型隐式锁。
3. **源码自身的一致性断言**：`lock_rec_lock` 里 `ut_ad(!impl || ((mode & LOCK_REC_NOT_GAP) == LOCK_REC_NOT_GAP))`；`lock_rec_lock_slow` 拆分分支里 `ut_ad(!impl)`（"Next Key Lock can not be emulated by implicit lock"）。

**物化即授予，不排队**：传入的 type_mode **不含 `LOCK_WAIT`**，`lock_rec_add_to_queue` 走非等待路径——优先复用同事务同页同形态的锁只置一个 bit，否则新建 granted 锁插队列头。语义上隐含锁本来就是该事务已拥有的锁，物化只是"显式化"让别人看得见，不是新请求。

**副作用**：这是**代别的 trx 建锁**（`we_own_trx_mutex=true`）。若此时该 trx 已提交则跳过；否则建一个它"不知道"的显式锁，靠引用计数归零保证安全。这也是为什么提交前必须 `while (trx_is_referenced(trx))` 自旋等待引用清零——否则清堆时别人正拿着这个 `lock_t`。

#### 谓词锁（R-tree 空间索引）：第四种锁语义

普通 B-tree 用「记录 + 间隙」表达范围，R-tree 做不到——**R-tree 上没有全序，"两条记录之间的间隙"没有定义**（一个 MBR 可同时被多个兄弟节点包含，兄弟节点 MBR 之间还可互相重叠）。于是 InnoDB 把**查询条件本身（MBR + 拓扑算子）当作锁对象**存下来：锁的不是"哪些记录"，而是"**满足这个谓词的任何记录**"。

源码里两处硬证据：

```cpp
// lock_rec_insert_check_and_lock()（普通 B-tree 插入检查）
/* Spatial index does not use GAP lock protection. It uses
"predicate lock" to protect the "range" */
ut_ad(!dict_index_is_spatial(index));      // ← 该路径对 R-tree 根本不允许被调用

// row0sel.cc：在 R-tree 上请求 LOCK_GAP / LOCK_ORDINARY 直接报错
if (type == LOCK_GAP || type == LOCK_ORDINARY) {
  ib::error(ER_IB_MSG_1026) << "Incorrectly request GAP lock on RTree";
  ut_d(ut_error);
}
```

`lock_mode_is_next_key_lock()` 的注释把谓词锁与 gap / insert-intention / record 并列为独立的锁语义类别。所以 **R-tree 上"gap 语义"完全由谓词锁承担，行锁层面只可能是 `LOCK_REC_NOT_GAP`**。

##### 数据结构：谓词不在 `lock_t` 的 union 里

`lock_t` 的 union **只有 `tab_lock` / `rec_lock` 两个分支**。谓词结构体被放在锁尾部：

```cpp
typedef struct lock_prdt {
  void *data;   /* Predicate data —— 当前唯一实现即 rtr_mbr_t */
  uint16 op;    /* Predicate operator */
} lock_prdt_t;

lock_prdt_t *lock_get_prdt_from_lock(const lock_t *lock) {
  return reinterpret_cast<lock_prdt_t *>(
      &((reinterpret_cast<byte *>(const_cast<lock_t *>(&lock[1])))[UNIV_WORD_SIZE]));
}
```

```
内存布局：[ lock_t ][ bitmap 1 字节 ][ 填充到 UNIV_WORD_SIZE ][ lock_prdt_t ]
                                                                  data → rtr_mbr_t{xmin,xmax,ymin,ymax}
```

`data` 是指针：栈上 MBR 用于"试探"（`lock_init_prdt_from_mbr(..., heap=nullptr)` 直接指向栈变量），确认要长期持有才拷进 `trx->lock.lock_heap`。`op` 是 `PAGE_CUR_CONTAIN` / `INTERSECT` / `WITHIN` / `DISJOINT` / `MBR_EQUAL` 之一。

**bitmap 只要 1 bit**：`PRDT_HEAPNO = PAGE_HEAP_NO_INFIMUM = 0`，所有谓词锁一律挂在页的 infimum 伪记录上，不管它保护的 MBR 覆盖页内哪些记录。粒度是**页**，精度由 MBR + 算子补齐。`RecLock::lock_size()` 对 `LOCK_PREDICATE` 返回 `sizeof(lock_prdt_t) + UNIV_WORD_SIZE`，对 `LOCK_PRDT_PAGE` 只要 1 字节（不挂 MBR）。

##### 三张 hash 表：键都是 `page_id`，谓词锁不跨页

```cpp
if (mode & LOCK_PREDICATE)     return lock_sys->prdt_hash;
else if (mode & LOCK_PRDT_PAGE) return lock_sys->prdt_page_hash;
else                            return lock_sys->rec_hash;
```

三条证据表明键与 `rec_hash` 完全相同（都是 `page_id`，`lock_rec_hash_value`）：三表共用同一 hash 函数；resize 时用同一个 `lock_rec_lock_hash_value` 迁移；`lock0latches.cc` 断言三表 cell 数相同且"cell 合并的两页必须落在同一 shard"。

那差别在哪？**① 队列隔离**（一页上"行锁队列"与"谓词锁队列"是两条独立链，混在一条队列里通用冲突逻辑会给出错误结论）；**② 粒度解释不同**（行锁每位 = 一条记录；谓词锁只有 bit 0 有意义，真正的"锁了什么"在尾部的 `lock_prdt_t` 里，靠 MBR 比对区分）；**③ 冲突判定函数不同**（`rec_lock_has_to_wait` vs `lock_prdt_has_to_wait`）。

**关键设计**：`lock_prdt_add_to_queue` 里 `type_mode |= LOCK_REC` —— 谓词锁的 `type_mode` 带 `LOCK_REC` 位，所以 `lock_get_type_low()` 看它就是 `LOCK_REC`（`LOCK_PREDICATE=8192`/`LOCK_PRDT_PAGE=16384` 在 bit 13/14，属 flags 不属 type）。这带来**零成本复用**：`RecLock::create`、`lock_rec_dequeue_from_page`、`lock_rec_grant_by_heap_no`、死锁检测、CATS 权重全部走通用路径，只需在 `locksys::has_to_wait` 里一行分流：

```cpp
if (lock1->type_mode & (LOCK_PREDICATE | LOCK_PRDT_PAGE)) {
  return lock_prdt_has_to_wait(lock1->trx, lock1->type_mode,
                               lock_get_prdt_from_lock(lock1), lock2);
}
```

代价是通用代码里到处要加 `ut_ad(!(mode & LOCK_PREDICATE))` 护栏（如 `lock_rec_has_expl` 直接断言拒绝谓词锁，所以 `lock0prdt.cc` 必须自带 `lock_prdt_has_lock`）。

##### 加锁入口：`lock_prdt_lock`

```cpp
hash_table_t *hash = type_mode == LOCK_PREDICATE ? lock_sys->prdt_hash
                                                 : lock_sys->prdt_page_hash;
locksys::Shard_latch_guard guard{UT_LOCATION_HERE, block->get_page_id()};
lock_t *lock = lock_rec_get_first_on_page(hash, block);

if (lock == nullptr) {                                    // 快路径 A：队列空
  RecLock rec_lock(index, block, PRDT_HEAPNO, prdt_mode);
  lock = rec_lock.create(trx);
  status = LOCK_REC_SUCCESS_CREATED;
} else if (队列里只有我这一个 && 模式完全一致 &&
           lock_prdt_consistent(已有谓词, 新谓词, 0, srs)) {   // 快路径 B：复用
  if (!lock_rec_get_nth_bit(lock, PRDT_HEAPNO)) lock_rec_set_nth_bit(lock, PRDT_HEAPNO);
} else {                                                  // 慢路径
  lock = lock_prdt_has_lock(mode, type_mode, block, prdt, trx);   // 我已有强锁？
  if (lock == nullptr) {
    wait_for = lock_prdt_other_has_conflicting(prdt_mode, block, prdt, trx);
    if (wait_for != nullptr) err = rec_lock.add_to_waitq(wait_for);
    else lock_prdt_add_to_queue(prdt_mode, block, index, trx, prdt);
  }
}
```

三点注意：① 谓词锁**只在 RR/SERIALIZABLE + 锁定读**下才下（`need_prdt_lock = set_also_gap_locks && !skip_gap_locks() && select_lock_type != LOCK_NONE`）——与 gap 锁同源，再次印证"谓词锁就是 R-tree 上的 gap 锁替代品"；② `btr_cur_search_to_nth_level` 的循环体里每层都下锁，所以**一次搜索会在路径上每个非目标层页留下一个 `LOCK_S|LOCK_PREDICATE`**；③ `gis0sea.cc` 注释原文 "Attach predicate lock if needed, **no matter whether there are matched records**" —— 即使该页一条匹配记录都没有也要下锁，这正是防幻读的要求。

**MBR 复用时单调放大**：`lock_prdt_add_to_queue` 命中已有锁后调 `lock_prdt_enlarge_prdt`，把 MBR **扩大成包络矩形**（逐边取 min/max）。省了 `lock_t` 数量，但事务读过的区域越多 MBR 越大 → 与无关插入产生**假冲突**的概率越高。

##### ★ 冲突判定的核心：只有插入意向锁会等待

`lock_prdt_has_to_wait` 的判定链（按顺序，命中即返回）：

| # | 条件 | 结果 |
|---|---|---|
| 1 | 同事务 / S-S 兼容 | 不等待 |
| 2 | 我方高优先级(CATS) 且对方在等待且非高优先级 | 不等待（bypass） |
| 3 | 新锁是 `LOCK_PRDT_PAGE` | **等待** |
| 4 | **新锁无 `LOCK_INSERT_INTENTION`** | **不等待** ⭐ |
| 5 | 已有锁有 `LOCK_INSERT_INTENTION` | **不等待** ⭐ |
| 6 | `!lock_prdt_consistent(已有谓词, 新 MBR, 算子, srs)` | 不等待（MBR 不重叠） |
| 7 | 其余 | **等待** |

第 4 条是最反直觉的设计，注释原文：

```
/* PREDICATE locks without LOCK_INSERT_INTENTION flag
do not need to wait for anything. This is because
different users can have conflicting lock types
on predicates. */
```

即 **S 谓词锁之间、S 与 X 谓词锁之间、X 与 X 谓词锁之间，只要不是插入意向，一律立即授予，永不等待**。`SELECT ... FOR UPDATE` 下的 `S|PREDICATE` 不会因为别人锁了重叠区域而阻塞——这与 B-tree 的 next-key 锁完全不同。

为什么可以？谓词锁保护的是"**不允许新对象进入我的区域**"，而插入是唯一的新对象来源；读-读、读-写、写-写之间不会凭空产生幻影。**只有 INSERT 会产生幻影，所以只有插入意向需要等待**——幻读防护被精确收窄到代价最小的那一种。

第 5 条（多个插入者互不阻塞）注释类比得非常清楚："This makes it similar to GAP lock, that allows conflicting insert intention locks"。

最终判定语义：已有谓词区 R（算子 op）与新插入 MBR 满足 `mbr_*_cmp(srs, R, MBR_insert)` → 新对象会落入查询结果 → 幻读 → 等待。MBR 比较委托给 `sql/gis/rtree_support.cc` 的 `mbr_contain_cmp` 等，内部用真正的 GIS 算子并传 `srs`（**支持地理坐标系/球面，不只是笛卡尔**）。

##### 页级谓词锁 `LOCK_PRDT_PAGE`

模式固定为 **`LOCK_S | LOCK_PRDT_PAGE`**，且**永远不等待**（`lock_place_prdt_page_lock` 直接返回 `DB_SUCCESS`，无 `add_to_waitq` 路径）——它是"声明我在用这个页"的软标记，不是互斥锁。用途由注释直说：**"Lock the page, preventing it from being shrunk"**。

两个下锁点：① R-tree 用 **SSN（Split Sequence Number）** 检测"我上次看到的路径之后发生过分裂"，一旦发现就给未搜索的兄弟页下页锁，防止它在补搜索期间被合并；② 下钻时锁住子页。

被消费的地方：页压缩前检查、purge 判断能否回收页、以及**合并时的硬断言**（`lock_update_merge_left/right` 里 `ut_ad(prdt_page_hash 上无锁)`，注释 "There should exist no page lock on the left page, otherwise, it will be blocked from merge"）。

##### 分裂与父页传播

`lock_prdt_update_split` 用 `op = PAGE_CUR_DISJOINT` 作判据（"已有谓词区域与某半页 MBR 是否相离"），三种情形：

| 与老半页 | 与新半页 | 动作 |
|---|---|---|
| 相离 | 不相离 | **move**（只在新页建） |
| 不相离 | 相离 | 不动（留在老页） |
| 都不相离 | | **duplicate**（两页都建） |

页锁无条件复制到新页；**等待中的 X 锁（插入意向）不复制**（"No need to duplicate waiting X locks"）。

`lock_prdt_update_parent` 要同时操作 left/right/parent 三页，注释给出解法："Latching their shards without deadlock is easiest using exclusive global latch"——直接上**全局排他 latch**（罕见路径，接受性能代价）。

##### 与隐含锁的关系：叶子有，谓词锁路径不管

R-tree 叶子记录上的普通 S/X 行锁**有隐含锁**（`lock_prdt_insert_check_and_lock` 成功后调 `page_update_max_trx_id`，`sel_set_rtr_rec_lock` 走 `lock_sec_rec_read_check_and_lock` → `lock_rec_convert_impl_to_expl`）。

但谓词锁路径**不做**隐含锁转换，理由写在 `lock_prdt_lock` 与 `lock_place_prdt_page_lock` 的注释里：

```
/* Another transaction cannot have an implicit lock on the record,
because when we come here, we already have modified the clustered
index record, and this would not have been possible if another active
transaction had modified this secondary index record. */
```

即"先改聚簇再改二级"的顺序保证了此时二级索引页上的相关修改要么来自自己，要么来自已提交事务。

##### 已知债务（源码明写）

1. **`/* FIXME: This needs to deal with predicate lock too */`**（`lock_move_reorganize_page`）——页面重组重建 bitmap 时**只处理 `rec_hash`，完全忽略两张 prdt hash**。谓词锁因固定在 heap_no=0（infimum 不变）"碰巧"不出错，但这是登记的未竟事项。
2. **Bug #89737**（`mysql-test/suite/innodb/t/innodb_cats.test` 注释）：CATS 模式下 `LOCK_PREDICATE` 曾"**完全不被释放**"，导致等待方饿死（FCFS 模式正常）。这是"复用通用 grant 路径"接入 CATS 时的历史代价。
3. `#ifdef PRDT_DIAG` 是**死代码**（全仓无定义，块内还引用了不存在的 `page_no` 变量——从未启用过）。

##### 可观测性：MBR 在任何地方都不可见

| 观测面 | 谓词锁长什么样 |
|---|---|
| `data_locks.LOCK_TYPE` | **`RECORD`**（不是 PREDICATE！因为 `lock_get_type_low` 只看 type 位） |
| `data_locks.LOCK_MODE` | `S,PREDICATE` / `X,PREDICATE,INSERT_INTENTION` / `S,PRDT_PAGE` |
| `data_locks.LOCK_DATA` | **恒为 `infimum pseudo-record`**（heap_no=0） |
| `SHOW ENGINE INNODB STATUS` | **看不出是谓词锁**——`lock_rec_print` 只打印 GAP/REC_NOT_GAP/INSERT_INTENTION/WAITING，**没有 PREDICATE 分支**；唯一线索是 `n bits 8` + `heap no 0` |
| MBR 坐标 / 算子 | **任何地方都不打印** |

`lock_get_mode_str()`（供 PFS）**有** PREDICATE，`lock_rec_print()`（供 SHOW ENGINE / 死锁日志）**没有** —— 两套输出不一致，排查 R-tree 锁问题时必须查 `performance_schema.data_locks`，`SHOW ENGINE INNODB STATUS` 会给不出答案。

### 等待与唤醒

#### 等待队列的挂载：复用存储 + LOCK_WAIT 位

granted 与 waiting 的锁在**同一个** hash cell（行锁）/ `dict_table_t::locks` 链表（表锁）里，逻辑布局（`lock0lock.h` 注释）：

```
Grows <---- [HEAD] [G7 -- G3 -- G2 -- G1] --|-- [W4 -- W5 -- W6] [TAIL] ---> Grows
                      granted                 |        waiting
```

granted 在头（新授予的插头部）、waiting 在尾（按到达顺序），区分全靠 `type_mode` 的 `LOCK_WAIT` bit。**没有独立的等待队列数据结构**。

#### srv_slot_t 等待槽位：slot ↔ thr ↔ wait_lock 严格 1:1:1

等待线程**不睡在锁上**，睡在全局 `waiting_threads[]` 数组的槽位上：

```
一个 srv_slot_t ↔ 一个 que_thr_t ↔ 一个 lock_t(wait_lock)
```

三层原因保证了这个 1:1:1：

- 分配时双向互指：`slot->thr = thr; thr->slot = slot;`。
- 事务侧 `trx->lock.wait_lock` 是**单指针**，`lock_set_lock_and_trx_wait` 里 `ut_a(trx->lock.wait_lock == nullptr)` 硬断言——一个事务同一时刻只能等一把锁。
- 架构上事务单线程执行：等锁时线程挂起（`TRX_QUE_LOCK_WAIT`），不可能继续请求第二把锁；被唤醒后 `wait_lock` 已清 NULL，才可能等下一把。

`srv_slot_t` 关键字段：`in_use`（槽位占用）、`suspended`（线程已挂起）、`thr`（等待线程）、`event`（`os_event_t`，睡眠/唤醒的载体）、`reservation_no`（ABA 版本号）、`wait_timeout`（该等待的到期时刻）。

#### 释放入口：提交/回滚共用一条链

提交与回滚**共用 `lock_trx_release_locks`**：`trx_commit_in_memory`（状态置 `TRX_STATE_COMMITTED_IN_MEMORY` 之后）与 `trx_rollback_finish`（回滚完直接调 `trx_commit`）殊途同归——对应 2PL 的 shrinking 阶段。释放分三步：

```cpp
void lock_trx_release_locks(trx_t *trx) {
  /* ① 等引用清零：其他线程可能在把本事务的隐含锁物化成显式锁
     （会先 trx_reference 拿引用），不清零就清堆会让人拿着悬空指针 */
  while (trx_is_referenced(trx)) { ... 随机退避自旋 ... }
  /* ② 摘除全部锁（可能被"X 等待者"打断，yield 后重试） */
  while (!locksys::try_release_all_locks(trx)) {
    std::this_thread::yield();
  }
  /* ③ 一次性清空整堆：所有 lock_t 都从 trx->lock.lock_heap 分配，
     "We don't free the locks one by one for efficiency reasons" */
  mem_heap_empty(trx->lock.lock_heap);
}
```

`try_release_all_locks` 的两难与解法：要摘的锁散布在不同 shard，本应拿全局 X latch——但源码注释实测**全 X 会让 sysbench TPS 掉 3%~11%**，于是改为全局 **S** latch + 让路逻辑：摘除过程中若发现有人正等全局 X（`is_x_blocked_by_us`），立刻放弃返回 false，由外层 `yield()` 重试，防止长事务释放锁饿死 DDL。另一个工程难点是锁序：`trx_locks` 链表遍历需要 trx->mutex，但拿 shard 前必须先放 trx->mutex（锁序规约），解法是"看链表尾锁 → 放 trx mutex → 拿 shard → 重拿 trx mutex → 校验锁仍在链表 → 摘除"的七步法（`try_relatch_trx_and_shard_and_do` 封装）。

摘除本体是"先摘下、后授予"两段：`lock_rec_dequeue_from_page` = `lock_rec_discard`（`HASH_DELETE` 从 hash cell 摘链 + 从 trx_locks 摘链 + `n_rec_locks` 原子减计数，**内存不 free**，留给事务结束整堆清空）+ `lock_rec_grant`（见下）。

#### 挂起：入睡的完整编排

`lock_wait_suspend_thread` 不只是一个 sleep，围绕睡眠有一整套编排（完整代码）：

```cpp
void lock_wait_suspend_thread(que_thr_t *thr) {
  trx = thr_get_trx(thr);
  const auto lock_wait_timeout = trx_lock_wait_timeout_get(trx);
  lock_wait_mutex_enter();
  trx_mutex_enter(trx);
  trx->error_state = DB_SUCCESS;

  if (thr->state == QUE_THR_RUNNING) {          // ① 快速路径：入睡前已被授予
    if (trx->lock.was_chosen_as_deadlock_victim) {
      trx->error_state = DB_DEADLOCK;
      trx->lock.was_chosen_as_deadlock_victim = false;
    }
    lock_wait_mutex_exit(); trx_mutex_exit(trx);
    return;
  }
  slot = lock_wait_table_reserve_slot(thr, lock_wait_timeout);   // ② 分配槽位
  if (thr->lock_state == QUE_THR_LOCK_ROW) {
    srv_stats.n_lock_wait_count.inc();          // ③ 只有行锁等待计入统计
    srv_stats.n_lock_wait_current_count.inc();
    start_time = std::chrono::steady_clock::now();
  }
  lock_wait_mutex_exit();
  trx_mutex_exit(trx);

  /* ④ 入睡前让路：释放 data dictionary 锁 + 让出 InnoDB 并发配额 */
  if (was_declared_inside_innodb) srv_conc_force_exit_innodb(trx);

  thd_wait_begin(trx->mysql_thd, lock_type == LOCK_REC ? THD_WAIT_ROW_LOCK
                                                       : THD_WAIT_TABLE_LOCK);
  os_event_wait(slot->event);                   // ===== 唯一挂起点 =====
  thd_wait_end(trx->mysql_thd);

  /* ⑤ 醒来恢复：重进 InnoDB、重拿 dd 锁、释放槽位 */
  if (was_declared_inside_innodb) srv_conc_force_enter_innodb(trx);
  lock_wait_table_release_slot(slot);

  if (thr->lock_state == QUE_THR_LOCK_ROW) {    // ⑥ 统计等待时长
    srv_stats.n_lock_wait_current_count.dec();
    srv_stats.n_lock_wait_time.add(diff);
    if (diff > lock_sys->n_lock_max_wait_time) lock_sys->n_lock_max_wait_time = diff;
    thd_set_lock_wait_time(trx->mysql_thd, diff);
  }
  if (trx->error_state == DB_DEADLOCK) return;              // ⑦ 唤醒原因分派
  if (trx->error_state == DB_LOCK_WAIT_TIMEOUT) MONITOR_INC(MONITOR_TIMEOUT);
  if (trx_is_interrupted(trx)) trx->error_state = DB_INTERRUPTED;
}
```

几个设计点：

- **① 快速路径**与唤醒侧对称：唤醒者先把 `thr->state` 置 RUNNING、清 `wait_lock`，入睡者持 trx mutex 检查一次就不睡——这让"锁已授予但线程还没睡"的竞态只需一次状态检查消解，**唤醒侧因此完全不需要拿 `lock_wait_mutex`**。
- **③ 只统计行锁**：`QUE_THR_LOCK_ROW` 才计 `Innodb_row_lock_*`——表锁等待看不到统计的根因。
- **④ 入睡前让路**是必需的：等待期间绝不能让其他线程因我们持 dd 锁/占着并发配额而无法推进（"堵住我们的线程需要进来"）。
- **⑦ 靠 `error_state` 区分唤醒原因**：`DB_SUCCESS`（锁已授予，上层重试同一条 SQL）/ `DB_DEADLOCK`（回滚整个事务）/ `DB_LOCK_WAIT_TIMEOUT`（回滚当前语句）/ `DB_INTERRUPTED`（被 kill）。



```cpp
if (thr->state == QUE_THR_RUNNING) {
  /* 快速路径：锁在入睡前已被授予 / 已被判死锁——唤醒者已把 state 置
  RUNNING、wait_lock 置 NULL，这里就不睡直接返回 */
  ...
  return;
}
slot = lock_wait_table_reserve_slot(thr, lock_wait_timeout);
...
/* 入睡前让路：释放 data dictionary 的 S/X 锁（等待期间绝不能让其他线程
   因我们持 dd 锁而无法推进）、srv_conc_force_exit_innodb 让出 InnoDB
   并发配额（堵住我们的线程需要进来） */
thd_wait_begin(trx->mysql_thd, ...);   // PFS 上报 THD_WAIT_ROW_LOCK/TABLE_LOCK
os_event_wait(slot->event);            // ==== 唯一挂起点 ====
thd_wait_end(...);
/* 醒来恢复：重进 InnoDB、重拿 dd 锁、lock_wait_table_release_slot 释放槽位 */
if (trx->error_state == DB_DEADLOCK) { ...in_rollback... return; }
```

槽位分配 `lock_wait_table_reserve_slot` 是 1:1:1 与 ABA 防伪的落点：线性扫描 `waiting_threads` 找空槽 → `slot->reservation_no = lock_wait_table_reservations++`（全局单调递增的"预约号"，一物三用：槽位复用的 ABA 版本号、睡眠期间被挂起线程数的计量、死锁环上"最后入环者"的识别）→ 双向绑定 `slot->thr`/`thr->slot` → `os_event_reset`（必须在 wait 前，防"先 set 后 wait"丢唤醒）→ 入槽后 `lock_wait_request_check_for_cycles` 踢一脚检测线程。槽满（`srv_max_n_threads` 个都在等）直接报错 abort。

**为什么是"入槽后"而不是"建边时"触发检测**——源码注释把这个顺序讲得很清楚：

```cpp
/* We call lock_wait_request_check_for_cycles() because the
node representing the `thr` only now becomes visible to the thread which
analyzes contents of lock_sys->waiting_threads. The edge itself was created
by lock_create_wait_for_edge() during RecLock::add_to_waitq() or lock_table(),
but at that moment the source of the edge was not yet in the
lock_sys->waiting_threads, so the node and the outgoing edge were not yet
visible. */
```

即：wait-for 图的**边**在 `lock_create_wait_for_edge`（建锁排队时）就建好了，但**源节点**（本事务）要等入槽后才被检测线程看见。提前触发等于让检测线程扫一份不完整的图——这就是 `lock_create_wait_for_edge` 里明写"we don't call ... here as it would be slightly premature"的原因。（该处注释原文写 "I hope this explains why we do waste time on calling ... from lock_create_wait_for_edge()"，与代码行为和相邻注释矛盾，应为漏写 not，按实际行为理解为"不在建边时调用"。）

#### 授予：lock_rec_grant_by_heap_no 逐行

这是"释放一把锁后如何把等待者唤醒"的核心，贴完整代码：

```cpp
static void lock_rec_grant_by_heap_no(lock_t *in_lock, ulint heap_no) {
  const auto hash_table = in_lock->hash_table();
  ut_ad(in_lock->is_record_lock());
  ut_ad(locksys::owns_page_shard(in_lock->rec_lock.page_id));

  using LockDescriptorEx = std::pair<trx_schedule_weight_t, lock_t *>;
  Scoped_heap heap(...);                       // 预分配：4 个 vector × 32 把锁
  RecID rec_id{in_lock, heap_no};
  Locks<lock_t *> low_priority_light{heap.get()};
  Locks<lock_t *> waiting{heap.get()};
  Locks<lock_t *> granted{heap.get()};
  Locks<LockDescriptorEx> low_priority_heavier{heap.get()};
  const auto in_trx = in_lock->trx;

  // ===== Stage 1：遍历同 (page, heap_no) 的锁队列，分组 =====
  Lock_iter::for_each(rec_id, [&](lock_t *lock) {
    if (!lock->is_waiting()) {
      ut_ad(!seen_waiting_lock);               // 不变量：granted 必在 waiting 前
      granted.push_back(lock);
      return (true);
    }
    const auto trx = lock->trx;
    if (trx->error_state == DB_DEADLOCK || trx->lock.was_chosen_as_deadlock_victim)
      return (true);                           // 死锁 victim 不参与授予
    const auto blocking_trx = trx->lock.blocking_trx.load(std::memory_order_relaxed);
    ut_ad(blocking_trx);
    if (blocking_trx != in_trx) return (true); // ★ 只考虑"被正在释放的事务"阻塞的
    if (trx_is_high_priority(trx)) { waiting.push_back(lock); return (true); }
    const auto schedule_weight = trx->lock.schedule_weight.load(std::memory_order_relaxed);
    if (schedule_weight <= 1) { low_priority_light.push_back(lock); return (true); }
    low_priority_heavier.push_back({schedule_weight, lock});
    return (true);
  }, hash_table);

  if (waiting.empty() && low_priority_light.empty() && low_priority_heavier.empty())
    return;                                    // 没人可授予

  // ===== Stage 2：组装授予顺序（公平性调度）=====
  std::stable_sort(low_priority_heavier.begin(), low_priority_heavier.end(),
                   [](const LockDescriptorEx &a, const LockDescriptorEx &b) {
                     return (a.first > b.first);     // 权重降序，平局按队列位置
                   });
  for (const auto &d : low_priority_heavier) waiting.push_back(d.second);
  waiting.insert(waiting.end(), low_priority_light.begin(), low_priority_light.end());

  const auto new_granted_index = granted.size();     // ★ 授予前的边界
  granted.reserve(granted.size() + waiting.size());

  // ===== Stage 3：逐个复核并授予 =====
  for (lock_t *wait_lock : waiting) {
    ut_ad(wait_lock->trx != in_trx);
    const lock_t *blocking_lock =
        lock_rec_has_to_wait_for_granted(wait_lock, granted, new_granted_index);
    if (blocking_lock == nullptr) {
      lock_grant(wait_lock);
      lock_rec_move_granted_to_front(wait_lock, rec_id);
      granted.push_back(wait_lock);            // 后来的候选要能看到它（FIFO 保序）
    } else {
      lock_update_wait_for_edge(wait_lock, blocking_lock);
    }
  }
}
```

三点补充：

- **`Scoped_heap` 预分配**：四个 vector 先预留"4 × 32 把锁"的空间，因为持 shard latch 期间做堆分配代价高；多数场景根本用不到。
- **`blocking_trx != in_trx` 剪枝**的价值：把"释放一个锁要扫整个队列并逐个判冲突"降为"只处理我真正挡住的少数人"。它复用的是 wait-for 图的单一出边语义。
- **`new_granted_index`** 记录"授予前的 granted 边界"，供 `lock_rec_has_to_wait_for_granted` 分两段检查——段一（旧锁）**逆序**、段二（本轮新授予的）**正序**。逆序的目的是让 `blocking_trx` 收敛到"最老的、死锁检测器还没分析过的阻塞原因"，使 wait-for 边稳定指向环上的下一个节点。

**复核函数 `lock_rec_has_to_wait_for_granted`**（Stage 3 用的那个，是模板函数）：

```cpp
template <typename Container>
static const lock_t *lock_rec_has_to_wait_for_granted(
    const typename Container::value_type &wait_lock, const Container &granted,
    const size_t new_granted_index) {
  /* We iterate over granted locks in reverse order.
  Conceptually this corresponds to chronological order.
  This way, we pick as blocking_trx the oldest reason for waiting we haven't
  yet analyzed in deadlock checker. Our hope is that eventually (perhaps after
  several such updates) we will set blocking_trx to the real cause of the
  deadlock, which is the next node on the deadlock cycle. */
  for (size_t i = new_granted_index; i--;) {          // 段一：旧 granted，逆序
    const auto granted_lock = granted[i];
    if (lock_has_to_wait(wait_lock, granted_lock)) return (granted_lock);
  }
  for (size_t i = new_granted_index; i < granted.size(); ++i) {  // 段二：新授予，正序
    const auto granted_lock = granted[i];
    ut_ad(granted_lock->trx->error_state != DB_DEADLOCK);
    ut_ad(!granted_lock->trx->lock.was_chosen_as_deadlock_victim);
    if (lock_has_to_wait(wait_lock, granted_lock)) return (granted_lock);
  }
  return (nullptr);
}
```

段一**逆序**的注释把动机说得很清楚：从最新锁走向最老锁，让 `blocking_trx` 逐步收敛到"最老的、死锁检测器还没分析过的阻塞原因"——几次更新之后出边会稳定指向死锁环上真正的下一个节点，检测器才能找到环。段二的两个 `ut_ad` 说明：本轮刚授予的锁，其事务绝不可能是死锁 victim。

释放锁时，`lock_rec_grant` 先快速探测本页有无 WAITING 锁（无则跳过，注释明言 replication applier 高频场景靠这个省掉向量分配），再对释放锁 bitmap 里每个置位 bit 逐个 heap_no 调 `lock_rec_grant_by_heap_no`。该函数三 stage：

**Stage 1 遍历分组**：`Lock_iter::for_each` 沿 hash cell 收集**同 `(page, heap_no)` 记录上的锁队列**，分四组——`granted`（已授予）、`waiting`（高优先级事务的等待锁）、`low_priority_heavier`（`schedule_weight > 1`）、`low_priority_light`。两个过滤：① 死锁 victim（`error_state == DB_DEADLOCK` 或 `was_chosen_as_deadlock_victim`）跳过——它马上被摘掉；② **`blocking_trx != 释放事务` 跳过**——只考虑被"正在释放的这个事务"阻塞的候选，这是把"释放一个锁要扫整个队列"降为"只处理我真正挡住的少数人"的关键剪枝（wait-for graph 的单一出边在此复用）。

**Stage 2 组装顺序（公平性调度）**：高优先级事务保持队列序排最前，heavier 组按 `schedule_weight` 降序 `stable_sort`（同权重按队列位置），light 组殿后。注意注释解释的细节：权重必须先 `load` 快照再排序，`std::sort` 要求比较序在执行期间一致。

**Stage 3 逐个复核授予**：`new_granted_index = granted.size()` 记录"授予前"的边界，然后对每个候选调 `lock_rec_has_to_wait_for_granted` 复核与当前 granted 集合的冲突。复核失败则 `lock_update_wait_for_edge` 更新阻塞出边（阻塞原因已从释放事务变成另一个锁）并触发增量死锁检测。复核通过则 `lock_grant` + `lock_rec_move_granted_to_front`（摘下来插回桶头，维护"granted 全在 waiting 前"的不变量）+ 并入 granted 集合（同记录上模式互斥的后续等待者会撞上它继续等，FIFO 保序）。

`lock_rec_has_to_wait_for_granted` 的两段检查是死锁检测正确性的关键：**段一（旧 granted）逆序遍历**、**段二（本轮新授予的）正序遍历**。逆序 = 从最新锁走向最老锁，注释说明目的：让 `blocking_trx` 收敛到"尚未被死锁检测器分析过的最老阻塞原因"——几次更新后出边会稳定指向死锁环上真正的下一个节点，检测器才能找到环。

**表锁侧的授予**（`lock_table_dequeue`）简单得多：从被摘锁的后继开始扫整条队列，对每个 waiting 锁问"你前面还有没有冲突"（`lock_table_has_to_wait_in_queue` 从队头扫到自身）。无排序、纯 FIFO；有意图锁快路径（队列无 S/X 时意图锁释放不可能产生授予）；AUTOINC 例外——复用 `table->autoinc_lock` 单例，授予时改 `table->autoinc_trx` 归属并压入新持有者的 `autoinc_locks` vector。

#### 唤醒：最多一次的结构性保证

`lock_grant` 核心只一行 `lock_reset_wait_and_release_thread_if_suspended(lock)`，其内部四步序：

1. 先清 `blocking_trx = nullptr`（wait-for 边正式消失；死锁快照读到 null 跳过该事务，`lock_wait_check_and_cancel` 也靠它区分"wait_lock 暂时为 null（B-tree 搬锁）"与"永久醒来"）；
2. `que_thr_end_lock_wait`：thr 状态 `QUE_THR_LOCK_WAIT → QUE_THR_RUNNING`、`que_state → TRX_QUE_RUNNING`；
3. `lock_reset_lock_and_trx_wait`：`wait_lock = nullptr`、清 `LOCK_WAIT` 位（`blocking_trx` 刻意不在这里清——B-tree 搬锁也走这里，此时 wait-for 边并未消失）；
4. `lock_wait_release_thread_if_suspended(thr)`：`os_event_set(slot->event)`。

"最多唤醒一次"是注释用四条规则证明的结构性保证：① 唤醒 trx 的唯一途径是 `os_event_set`；② `os_event_set` 的唯一调用点就是这里；③ 调用前必先 `lock_reset_lock_and_trx_wait` 且两步都在持 shard latch 的临界区内；④ 后者断言 `wait_lock == lock` 并置 NULL。⇒ 第二次唤醒尝试过不了断言——**一个睡眠至多对应一个唤醒原因**（granted / deadlock / timeout / interrupted 不可能叠加），否则两方各以为自己的原因是生效的那个。

唤醒侧还免了 `lock_wait_mutex`：`thr->slot` 三条件（`slot != nullptr && in_use && slot->thr == thr`）在 trx->mutex 下检查就够了——我们是第一个把 wait_lock 置 null 的人、唯一有权唤醒；未入睡者马上会拿 trx mutex 检查 `thr->state == QUE_THR_RUNNING` 而不睡（与挂起侧快速路径对称）。deadlock victim 在此转换：`was_chosen_as_deadlock_victim → error_state = DB_DEADLOCK`。

**另外两条唤醒路径**与授予同终点：死锁 victim（检测线程置标志后 `lock_cancel_waiting_and_release`，同样摘锁 + 唤醒）；超时/中断（`lock_wait_check_slots_for_timeouts` 每秒扫槽位，到期者置 `error_state = DB_LOCK_WAIT_TIMEOUT` 再取消；HP 事务对非 HP 阻塞者不让超时）。

**醒来后怎么办**：`que_run_threads` 状态机里 `QUE_THR_LOCK_WAIT` 分支调 `lock_wait_suspend_thread` 返回后检查 `error_state`——`DB_SUCCESS` 则 `goto loop` 重跑同一条 SQL（"锁被授予"= 从挂起点返回重试，冲突已消）；非 SUCCESS 则进入错误处理（DB_DEADLOCK 回滚整个事务、DB_LOCK_WAIT_TIMEOUT 回滚当前语句）。

### 死锁检测

#### 后台线程主循环：1 秒轮询 + 事件提前唤醒

`lock_wait_timeout_thread` 是单线程守护循环，每轮固定做三件事：

```cpp
do {
  /* ① 超时检查：每秒最多一次（锁超时参数只有秒级分辨率，
     "probably nobody cares if we wake up after T or T+0.99"） */
  if (std::chrono::seconds(1) <= current_time - last_checked_for_timeouts_at) {
    last_checked_for_timeouts_at = current_time;
    lock_wait_check_slots_for_timeouts();
  }
  /* ② 死锁检测 + 权重：每轮必做 */
  lock_wait_update_schedule_and_check_for_deadlocks();
  /* ③ 睡最多 1 秒；新等待边 lock_set_timeout_event 会提前唤醒 */
  os_event_wait_time_low(event, std::chrono::seconds{1}, sig_count);
  sig_count = os_event_reset(event);
} while (srv_shutdown_state.load() < SRV_SHUTDOWN_CLEANUP);
```

注意**不是**"按最急 wait_timeout 精确计算下一次唤醒时刻"——8.0.39 是固定 1 秒轮询 + 事件唤醒（新等待事务入槽时 `lock_wait_request_check_for_cycles → lock_set_timeout_event` 踢一脚），超时最多晚 0.99 秒发现，而超时本身只有秒级精度。超时分支 `lock_wait_check_slots_for_timeouts` 扫 `[waiting_threads, last_slot)` 区间（`last_slot` 高水位避免扫满全表），持 `lock_wait_mutex` 读槽位（槽位分配/释放都需该 mutex，无需 trx mutex）；超时或 `trx_is_interrupted`（KILL）才 `lock_wait_try_cancel`——HP 事务对非 HP 阻塞者不让超时。

#### wait-for graph 的构建

`lock_wait_update_schedule_and_check_for_deadlocks` 每轮四步：

```cpp
lock_wait_snapshot_waiting_threads(infos);        // 1. 快照
lock_wait_build_wait_for_graph(infos, outgoing);  // 2. 构图：每个等待者一条出边
lock_wait_compute_and_publish_weights_except_cycles(...);  // 3. 非环节点权重
if (innobase_deadlock_detect) {
  lock_wait_find_and_handle_deadlocks(...);       // 4. 找环 + 处理
}
```

快照 `lock_wait_snapshot_waiting_threads` 持 `lock_wait_mutex` 一次性扫槽位表，产出 `waiting_trx_info_t` 四元组：`{trx, waits_for（取自 blocking_trx 的 relaxed load）, slot, reservation_no}`；`blocking_trx == nullptr` 的槽位跳过（B-tree 搬锁或已被唤醒决定）。追求短平快——注释明言 `push_back` 已是最快做法，甚至论证了"快照其实不需要一致"（拆分多段拼接也能工作）。**注意**：8.0.39 当前**没有**"同一事务多槽位去重"的代码——注释只把它作为未来优化方向预留（若将来为降 mutex 争用分段快照，才需要在构图阶段按 trx 去重保留最新 `reservation_no`）；当前快照在持锁下原子完成，无去重必要。

构图 `lock_wait_build_wait_for_graph` 产出 `outgoing[from] = to`（等待者 → 阻塞者在 infos 中的下标；阻塞者不在快照内则 -1）。实现选 **sort + lower_bound**：按 `trx` 指针排序后对每个 `waits_for` 二分，O(n log n) 零额外分配——注释给出了三方案对比结论（`unordered_map` 桶链表分配太慢、自定义开放寻址哈希无吞吐收益，故保持朴素）。构图阶段**不做任何事务状态过滤**（不存在 `lock_wait_is_older_than` 这类函数），验证推迟到候选环校验阶段。

`innobase_deadlock_detect=OFF` 时跳过第 4 步，死锁靠 `innodb_lock_wait_timeout` 兜底。

#### 环检测算法：单出边 DFS + 着色

wait-for graph 的每个节点**恰好一条出边**（一个事务同一时刻只等一个事务），这是算法能大幅简化的前提。`lock_wait_find_and_handle_deadlocks` 的核心：

```cpp
ut::vector<uint> colors;                 // 0 = 从未访问
colors.resize(n, 0);
uint current_color = 0;
for (uint start = 0; start < n; ++start) {
  if (colors[start] != 0) continue;      // 已处理过的起点跳过
  ++current_color;
  for (int id = start; 0 <= id; id = outgoing[id]) {   // 沿出边一路走
    ut_ad(id != outgoing[id]);           // 断言：不会自等（环长≥2）
    if (colors[id] == 0) {
      colors[id] = current_color;        // 本轮 DFS 首次到达：染色继续
      continue;
    }
    if (colors[id] == current_color) {   // 撞到本轮的色 = 环！
      lock_wait_extract_cycle_ids(cycle_ids, id, outgoing);
      if (lock_wait_check_candidate_cycle(cycle_ids, infos, new_weights)) {
        MONITOR_INC(MONITOR_DEADLOCK);
      } else {
        MONITOR_INC(MONITOR_DEADLOCK_FALSE_POSITIVES);
      }
    }
    break;                               // 旧色 = 汇入已处理路径，直接停
  }
}
```

算法解读（三色变体，但只用一个数组 + 轮次色号）：

- **一轮 DFS 一个色号**（`current_color`），同轮内再次撞到同色节点 → 存在环（`lock_wait_extract_cycle_ids` 沿出边把环成员摘出来）。
- **撞到旧色 → 立即 break**：该路径汇入了更早轮次已处理的子图，那个子图若有环早被报告过——这是"单出边"带来的剪枝：每个节点最多属于一条主路径，不需要 Tarjan/三色栈回溯。
- **复杂度**：每个节点最多被走两次（一次自己轮、一次被别的轮汇入），O(n) 级；无需递归，无栈深问题。

**假阳性防御（两段式校验的加锁编排）**：快照释放 mutex 后到检测前，等待关系可能已变化——`infos[i].trx` 可能已回滚、`trx_t` 对象被复用甚至释放（直接解引用会 segfault）。`lock_wait_check_candidate_cycle` 用两道关卡、两把越来越贵的锁确认环仍真实：

```cpp
lock_wait_mutex_enter();
/* 第一关：只碰槽位、不碰 trx 对象——比对每个槽位当前 reservation_no
   与快照值；相等 ⇒ 该槽位从快照起一直被同一线程占用 ⇒ trx 指针仍有效
   （槽位释放再被他人占用时 reservation_no 必然变大，这就是 ABA 防伪） */
if (!lock_wait_trxs_are_still_in_slots(cycle_ids, infos)) {
  lock_wait_mutex_exit();
  return false;
}
/* 第二关：槽位还在 ≠ 还在等待——可能已被通知唤醒（wait_lock 已置 NULL）
   只是没来得及清理槽位。检查 wait_lock 需要全局独占 latch：
   环上事务的锁散布在不同 shard，B-tree 搬锁会临时置 NULL、HP 事务会 abort 别人 */
locksys::Global_exclusive_latch_guard guard{UT_LOCATION_HERE};
if (!lock_wait_trxs_are_still_waiting(cycle_ids, infos)) {
  lock_wait_mutex_exit();
  return false;
}
/* 此时可以释放 lock_wait_mutex：① 已验证 wait_lock 非 NULL ② 持全局独占
   latch ③ wait_lock 置 NULL 必须先拿全局 latch ④ 只有 wait_lock 置 NULL
   后事务才能结束 ⇒ 只要不放全局 latch，这些 trx 就不会被唤醒、不会结束 */
lock_wait_mutex_exit();
trx_t *chosen_victim = lock_wait_choose_victim(cycle_ids, infos);
lock_wait_handle_deadlock(chosen_victim, cycle_ids, infos, new_weights);
```

第一关只花 `lock_wait_mutex`（多数假阳性在此拦下），第二关才动用全局独占 latch——注释用四条推理链证明"过了第二关、持着全局 latch，trx 指针就绝对安全"。两道都过才动手，否则计入 `MONITOR_DEADLOCK_FALSE_POSITIVES`——这个计数器存在本身就说明"快照并发检测"会撞上假环。

#### victim 选择：权重最小者 + 高优先级仲裁

`lock_wait_choose_victim` 在环成员里挑牺牲者：

```cpp
auto sorted_trxs = lock_wait_order_for_choosing_victim(cycle_ids, infos);
for (auto *trx : sorted_trxs) {
  if (chosen_victim == nullptr) { chosen_victim = trx; continue; }

  if (trx_is_high_priority(chosen_victim) || trx_is_high_priority(trx)) {
    auto victim = trx_arbitrate(trx, chosen_victim);   // 高优先级参与 → 仲裁
    if (victim != nullptr) {
      if (victim == trx) chosen_victim = trx;
      continue;
    }
  }
  if (trx_weight_ge(chosen_victim, trx)) {
    chosen_victim = trx;                 // 新事务"更小" → 换它为牺牲者
  }
}
```

三个要点：

- **权重公式精确到字段**（`trx_weight_ge`）：先比"是否改过非事务表"（`thd_has_edited_nontrans_tables`，改过 MyISAM 的不能回滚 ⇒ 永不作 victim，两个事务此维度相同才比数值）；数值 = `TRX_WEIGHT = undo_no + UT_LIST_GET_LEN(trx->lock.trx_locks)`——`undo_no` 是事务私有 undo 记录序号（无空洞递增 ⇒ 约等于修改/插入的行数），加上持有的显式锁对象数。最小者回滚，代价最低。该函数也是 `INNODB_TRX` 视图 `trx_weight` 列的数据源。
- **环上遍历顺序有讲究**（`lock_wait_order_for_choosing_victim`）：把"最后加入等待"的事务（reservation_no 最新，即"闭合环"的那条边）与其前驱移到队尾，平局时最新等待者优先被牺牲——注释坦承这些旋转"从正确性角度是 no-op"，唯一理由是兼容旧版本 victim 选择与报告格式的确定性、让既有测试通过。工程上把"可复现性"摆在与"最优解"同等重要的位置。
- **高优先级事务仲裁**（`trx_arbitrate`）：`thd_trx_priority > 0` 的事务参与仲裁而非纯比重量——DDL 类事务（无 `mysql_thd` 或非用户事务）"should never rollback"，注释点明它们本就不该在表/行锁上等待。高优先级事务还有 `trx_kill_blocking` 主动杀阻塞者的机制。

**victim 的处理是三段接力**（`lock_wait_handle_deadlock`）：① `lock_wait_update_weights_on_cycle`——victim 回滚后环"展开"成链，从 victim 处切开，环上其余节点补算权重（注释坦承这步 "mostly for correctness"，性能影响可忽略）；② `lock_notify_about_deadlock`（`Deadlock_notifier`，见下）；③ `lock_wait_rollback_deadlock_victim`——置 `was_chosen_as_deadlock_victim` 标志 + `lock_cancel_waiting_and_release`（摘锁 + 唤醒，与授予同终点），被唤醒线程在 `lock_wait_release_thread_if_suspended` 里把标志转为 `error_state = DB_DEADLOCK`，最终由事务回滚逻辑整事务回滚（DB_DEADLOCK 回滚整个事务，超时只回滚当前语句——这正是超时路径断言不覆盖 DB_DEADLOCK 的原因）。

**死锁报告走匿名临时文件**：`Deadlock_notifier::notify` 在杀 victim **之前**生成报告（此时环上 wait_lock 尚未拆除，`lock_has_to_wait_in_queue` 才能反查每把冲突锁），写入 `lock_latest_err_file`（`os_file_create_tmpfile` 创建的匿名临时文件，每次 `rewind` 覆盖旧内容）。格式对环上每个事务三段：`*** (i) TRANSACTION:`（事务详情含 SQL 文本）+ `HOLDS THE LOCK(S):`（它持有的阻塞锁）+ `WAITING FOR THIS LOCK TO BE GRANTED:`（它的等待锁），最后 `*** WE ROLL BACK TRANSACTION (N)` 标 victim。`SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 段由 `lock_print_info_summary` 检查 `lock_deadlock_found` 标志后 `ut_copy_file` 拷出；只有 `innodb_print_all_deadlocks=ON` 才同步刷 error log。

**schedule_weight 与 victim 权重是两个不同的"权重"**：前者衡量"授予收益"（一个节点 + 所有传递地被它阻塞的节点的初始权重之和，Kahn 式拓扑累加），用于释放锁时的授予排序；初始权重 1，"等太久"者获 `WEIGHT_BOOST`（默认 = n、夹在 1e9/n 防溢出）——判据 `reservation_no + 2*n < table_reservations`，即睡着期间又有超过 2n 个线程挂起 ⇒ 至少 n 个事务插了队。发布用 relaxed store（允许短暂陈旧，只影响公平性），发布前再验一次 ABA。**消费点只有 `lock_rec_grant_by_heap_no` 一处**（外加 I_S 展示）——它不是 purge 的优先级（purge 看 ReadView），本质是唤醒顺序的公平性调度：防 convoy 中长事务被反复饿死 + 防插队饥饿。

### 锁与 MVCC / 半一致性读的交互

#### 锁与 MVCC 版本的关系（正交）

**核心结论：行锁锁的是"索引记录（rec，物理记录）"，不是 MVCC 版本。两者正交。**

- 锁（`lock_rec_t`）：控制"谁能修改这条记录"，定位靠 `(space_id, page_no, heap_no)`。
- 版本（MVCC）：记录上的 `DB_TRX_ID` + `DB_ROLL_PTR` 指向 undo 链，控制"谁能读到哪个历史值"。

事务 UPDATE 一条记录时同时发生两件事：在 rec 上加 X 锁（锁物理位置，阻止别人改），并把旧值写进 undo（生成旧版本）。所以"记录被别的锁占了"指 rec 上有不兼容锁（别人正在改/已改未提交），与版本无关。

semi-consistent read 的做法：rec 被锁加不了锁（要改得等），但 rec 的 undo 链上有"已提交的旧版本"，可沿链读出来（`row_vers_build_for_semi_consistent_read`）判断 WHERE。

#### gap lock 与幻读

**为什么需要 gap lock**：幻读（phantom read）= 同一事务内两次范围查询，第二次多出了新插入的行。RR 及以上隔离级别用 gap/next-key lock 锁住"间隙"，阻止别的记录插入该范围，从而防幻读。

**gap lock 依附于记录**：gap lock 锁的是"间隙"（记录之间的空隙），但实现上必须**依附一条记录**表达：`LOCK_GAP` 类型的锁挂在某记录上，表示锁住"这条记录之前的空隙"。所以 gap lock 总是和某条"边界记录"绑定。

**gap lock 的精确语义（只禁止插入）**：

- `LOCK_GAP`：只禁止**向间隙 INSERT 新记录**（防幻读的"多出记录"那一半），**不管**已有记录的修改/删除。
- `LOCK_REC_NOT_GAP`（record lock）：禁止修改/删除**这条记录**。
- `LOCK_ORDINARY`（next-key lock = record + gap）：两者结合，才实现"范围既不被插入、已有记录也不被改删"的完整稳定。

所以"gap lock 禁止整个 range 任意变化"是误解——gap lock 单独只禁止"插入"。

**skip_gap_locks 与隔离级别**：`trx_t::skip_gap_locks()`——READ UNCOMMITTED / READ COMMITTED 返回 true（不加 gap lock，容忍幻读）；REPEATABLE READ / SERIALIZABLE 返回 false（加 gap lock，防幻读）。

#### 半一致性读与 gap lock 的边界

`row_compare_row_to_range` 在加锁路径上判断"当前记录 vs 扫描范围"、决定加 `LOCK_ORDINARY`/`LOCK_REC_NOT_GAP`/`LOCK_GAP` 哪种锁。

`row0sel.cc` 一处注释 + 断言（`ut_ad(!trx->skip_gap_locks())`、`ut_ad(row_read_type == ROW_READ_WITH_LOCKS)`）的含义：

走到该分支说明前面的 `if (trx->skip_gap_locks() || ...)` 已 return 掉所有 RC/RU 情况，即当前必在 RR（加 gap lock）。而 semi-consistent read 只在"不加 gap lock"的 RC/RU 下启用（`allow_semi_consistent() == skip_gap_locks()`）。两者永不共存，因此无需证明"范围末尾行被锁+删除+purge，同时还在对它做半一致性读"的复杂场景。

"范围末尾的行" = 扫描范围（WHERE 划定的边界）处那条记录，它的前一个间隙需要被 gap lock 锁住；若该记录被删/purge，"给已不存在的记录锁间隙"变得微妙，这是注释不愿去证明的情形。

**为什么本质互斥（为什么只能 RC/RU）**：

semi-consistent read 的优化手段是"读到不满足 WHERE 的**最新已提交版本**就**跳过这行、不锁它**"；gap lock 的使命是"**锁住范围边界**、防止新记录插入（防幻读）"。前者要求"允许跳过、允许范围变化"，后者要求"禁止插入、边界稳定"，语义互斥——同一行不可能既"跳过不锁"又"锁住它前面的间隙"。

注意：semi-consistent 读的是"最新已提交版本"，不是"未提交的新值"（被锁的 rec 上当前值属于未提交事务 X，不能用来判 WHERE），也不是"任意旧值"——是沿 undo 链找的第一个已提交版本（`row_vers_build_for_semi_consistent_read` 用 `trx_rw_is_active` 判断）。

**为什么这个 case 与 RR/gap lock 绑定（RC 也有记录被删被 purge）**：

关键不在"记录被删被 purge"本身（RC 也有），而在"**要不要给这条被删的记录挂 gap lock 锁边界**"。

反例：`UPDATE t SET x=1 WHERE a>10` 扫到范围末尾 R（`a=100`），事务 X 删除 R 并已 purge（物理移除）。

- **RC（无 gap lock）**：只对 R 本身负责——读最新已提交版本判 WHERE，满足重读加锁/不满足跳过；R 被 purge 下次扫描自然看不到。**无需锁 R 前面的间隙**，锚点消失无害。
- **RR（有 gap lock）**：除处理 R 本身，还**必须给"R 之前的间隙"加 gap lock**（防 `a>100` 插入）。这个间隙锁必须**挂在 R 上**（gap lock 依附记录），R 被 purge 后索引里定位不到锚点，范围末尾边界锁失效 → 幻读。

所以 case 与 RR 的关联点是：**gap lock 的施加依赖"范围边界记录还存在、能当锚点"，而 semi-consistent read（读已提交版本、跳过、配合记录被删被 purge）会让锚点消失。** RC 没有"必须锁边界"这回事，故同样的"记录被删被 purge"在 RC 无害、在 RR 却让防幻读失效。

---

## ★ 本机制里的工程实现技法

### 一、位图锁：变长结构 + 预留 margin

`lock_t` 的 bitmap **不在结构体里**，分配时按 `sizeof(lock_t) + bitmap_bytes` 一次性申请，bitmap 紧贴结构体尾部（`RecLock::create`）。这是"变长尾数组"的 C 式实现——避免在结构体里放固定大 bitmap（一页最多上千行），只按实际页内行数（`n_bits`）分配。

`LOCK_PAGE_BITMAP_MARGIN = 64` 是工程缓冲：建锁时按 `当前行数 + 64` 分配，页上后续 INSERT 少量新行（heap_no 变大）时不必重建锁、重扩 bitmap——否则每次页增长都要重新分配锁对象并搬移。代价是每把行锁多 64 bit（8 字节）的固定浪费。

### 二、侵入式双链：一个对象挂两个视角

`lock_t` 同时是两条链表的节点：`trx_locks`（事务视角：遍历"这个事务持了哪些锁"）和 `hash`（锁视角：行锁挂 hash cell 单链表）/ `tab_lock.locks`（表锁挂表链表）。标准库链表是"节点包数据"，侵入式是"数据里嵌节点"——零额外分配、无 `std::list` 的节点寻址开销，代价是对象必须知道自己同时属于哪些链（删除时要摘两条链）。

### 三、锁系统的分片 + 全局闸门：两级 latching

结构详见「核心实现 → 锁的数据结构」的 `locksys::Latches` 段，此处只讲与教科书"分段锁（striped lock）"原型的差异：**多了一层全局读写闸门**，且闸门本身又是分片的 `Sharded_rw_lock`（ARM 上 S 锁计数 cache line 争抢的优化）。分片是为并发（常规路径只碰 1~2 个 shard），闸门是为正确性（死锁检测校验、全量校验需要一致视图时 X 锁"stop the world"）。两级 latching 的代价是**双重持锁顺序规约**：先 S 锁 global 再按指针序锁 shard，任何路径违反都会在 debug 断言里暴露。

### 四、用"结构性不变量"把复杂问题退化成简单问题

本篇最值得记住的一条：**每个事务同一时刻至多一个等待锁**（`trx->lock.wait_lock` 是单指针，`lock_set_lock_and_trx_wait` 用 `ut_a` 硬断言）。这一条约束同时简化了四件事：

1. **wait-for graph 每个节点恰好一条出边** → 死锁检测的 DFS 从"三色回溯"退化成"一个数组 + 轮次色号"，O(n) 无递归（见「死锁检测」节）。
2. **"谁阻塞我"可以用单值 `blocking_trx` 表示**，不必为每把等待锁单独存阻塞者。
3. **唤醒只需一个 `os_event`**（`slot ↔ thr ↔ wait_lock` 严格 1:1:1），唤醒侧甚至不需要 `lock_wait_mutex`。
4. 反过来，它也是**其他规则的约束来源**：II 锁不得越过 WAITING 的 gap/next-key 锁，正是因为插队会分裂间隙、把等待锁复制成两份，破坏这个不变量。

这是"先定死一个强不变量，再让所有设计围绕它简化"的典型手法——代价是任何想打破它的优化都要额外论证。

### 五、用"代数性质"换 O(1)：两处非对称规则

两处看起来是特判、实则是利用代数性质的优化：

- **`count_by_mode`**：从兼容矩阵读出"IS 只被 X 挡、IX 只被 S/X 挡" → 请求意向锁时，把"遍历表锁队列"换成"看两个整数是否为 0"（`count_by_mode[S] == 0 && count_by_mode[X] == 0`）。代价是有 DDL 在跑时退化成 O(n²)。
- **gap 锁的非对称冲突规则**：gap 锁作为请求方与万物兼容（分支 A）、作为既存方只挡 II 锁（分支 B）。这条非对称让两件事成立——① "补一个 gap 锁"永远秒成功，`lock_reuse_for_next_key_lock` 的拆分启发式才可能（否则 next-key 拆分就没意义）；② 大量本要等待的请求变成无等待授予。

两处的共同点：**不是加代码优化，而是从"规则本身"推出捷径**——先看清规则的代数结构，再让常见路径绕过通用逻辑。

### 六、用版本号与引用计数解决"并发期间的身份与生命周期"

三处同构的手法，都用来解决"我看到的东西还是不是刚才那个"：

| 场景 | 机制 | 解决什么 |
|---|---|---|
| 等待槽位复用 | `reservation_no` 单调递增 + 快照比对 | ABA：事务离开又重入同一槽位（死锁假阳性校验的第一道关） |
| 隐含锁物化 | `trx` 引用计数（`trx_rw_is_active(..., do_ref_count=true)` + `trx_release_reference`） | 物化期间目标 `trx_t` 被释放→悬空指针；提交前必须 `while (trx_is_referenced)` 自旋等清零 |
| 锁对象归属 | `slot->thr = thr; thr->slot = slot` 双向绑定 | 唤醒时如何确认"这个槽位还是我的" |

`reservation_no` 还一物三用（ABA 防伪、睡眠期间挂起线程数计量、识别"最后入环者"选 victim）——**一个计数器服务三个原本独立的需求**，代价是它的语义必须被所有使用方一致理解。

---

## 可观测性

### 三套观测体系的分工

MySQL 8.0 观测锁有**三套并存、互不隶属**的体系，混淆它们是"为什么我查不到锁"这类问题的根源：

| 体系 | 载体 | 数据模型 | 开销 | 开关 |
|---|---|---|---|---|
| **① InnoDB Monitor 输出** | 一整块自由文本（`SHOW ENGINE INNODB STATUS`、error log） | 查询时**现场生成**的全景快照 | 重（持全局 latch、可能读盘） | `innodb_status_output` / `innodb_status_output_locks` |
| **② monitor counter** | `information_schema.INNODB_METRICS` 的 ~300 行 | 全局聚合计数（次数/极值/均值），无 per-thread 维度 | **极轻**（埋点 = 一次位图判断 + 原子加） | `innodb_monitor_enable/disable/reset[_all]` |
| **③ PFS 锁表** | `performance_schema.data_locks` / `data_lock_waits` | 单把锁 × 单个记录的**关系行**，可 SQL 过滤/聚合 | 中（持全局 latch 批量扫描，不读盘） | 无（`performance_schema` 总开关之外）|
| **（附）server 状态变量** | `SHOW STATUS LIKE 'Innodb_row_lock%'` | 计数与耗时 | 极轻 | 无（默认有） |

三者的关系不是"三个实现"，而是**同一份底层事实的三种暴露**——例如 `Innodb_row_lock_waits` 与 `INNODB_METRICS` 的 `lock_row_lock_waits` 读的是同一个 `srv_stats.n_lock_wait_count`（见下文 `MONITOR_EXISTING`）。

### InnoDB Monitor 输出（锁段怎么读）

Monitor 是 **InnoDB 全局的可观测基础设施，不是锁专属**：三个 monitor 类后台线程、输出组织与临时文件中转机制、SHOW 的完整路径见 [`../../innodb/monitor.md`](../../../innodb/monitor.md)。这里只交代**与锁相关**的三点。

**① 锁段在输出里的位置**：`TRANSACTIONS` 段由 `srv_printf_locks_and_transactions` 单独抽出（因为它需要持 `lock_sys` 全局排他 latch），内部先嵌 `LATEST DETECTED DEADLOCK`（用 `ut_copy_file` 从 `lock_latest_err_file` 拷出），再是 `LIST OF TRANSACTIONS FOR EACH SESSION`。

**② `innodb_status_output_locks` 决定锁段详细程度**：OFF（默认）只打印每个事务的摘要行（`---TRANSACTION ...` + `TRX HAS BEEN WAITING ...`）；ON 才逐个打印它持有的每把锁，且**每个事务最多 10 把**（超出打印 `10 LOCKS PRINTED FOR THIS TRX: SUPPRESSING FURTHER PRINTS`）——大事务的锁清单必然被截断，只能靠 PFS 表拿全。

**③ 打印锁会读盘、并临时放锁**：打印锁住的记录内容需要 `lock_rec_fetch_page()` 把页读进 buffer pool，读盘时**会临时释放全局排他 latch 与 `trx_sys->mutex`**——所以这份输出跨页读的边界处不是同一时刻的一致快照（一致性比 PFS 表更松）。这也是周期 monitor 线程要用 `MUTEX_NOWAIT` 降级的原因：系统已卡住时诊断不能自己阻塞在全局 latch 上（详见 monitor 篇）。

### PFS 锁表的实现：server 定义表、引擎填充行

`data_locks` / `data_lock_waits` 是 **PFS 定义虚拟表、InnoDB 提供数据**的依赖倒置结构：表结构与 4 个 HASH 索引硬编码在 `storage/perfschema/table_data_locks.cc`（`Plugin_table`，`perpetual=false` 表示不物化），InnoDB 通过 `PSI_data_lock_service_v1` 注册一个 inspector，每次 SELECT 创建有状态 iterator 现场填行。**WHERE 通过 `accept_*()` 回调下推到引擎**——引擎生成行之前先问容器"这个 trx_id/表对象你要不要"，不要就跳过。

#### 批量可重启扫描（restartable batch scan）

`p_s.cc` 顶部的 doxygen 把三种方案与取舍写得很完整，值得原样记住：全量物化（引擎冻结时间不可控 + 内存无上界）与单行扫描（O(N²) 且无锁重启游标不可靠）都被否决，实现的是**按事务分批**：一次 `scan()` 最多报 `SCAN_RANGE = 256` 个事务的锁。

关键难题是"释放 latch 后如何续扫"——事务链表既不安 id 排序也不安地址排序，且在放锁期间持续变化。解法是引入单调不变量 `trx_immutable_id(trx)` 把事务映射到自然数空间再切区间 `[start, end)`，每批用 `Max_of_n_smallest<256>`（小顶堆，O(RANGE) 内存）找出"当前未处理的最小 256 个 id 中的最大值"作为右边界。代价：数据**只保证"块内一致"**（源码原文 `consistent by chunks`）。

遍历入口是**事务链表**（先 `rw_trx_list` 后 `mysql_trx_list`）再走到每个 trx 的 `trx_locks`，而不是遍历 `rec_hash` 的每个 cell——因为需要按事务分批、需要 `ENGINE_TRANSACTION_ID`、且能保证一个 `lock_t` 只被访问一次。代价是必须持 `Global_exclusive_latch_guard` + `trx_sys->mutex`（跨 shard、保证链表与 `wait_lock` 稳定），所以 **`SELECT * FROM performance_schema.data_locks` 是一个会短暂冻结整个 InnoDB 锁子系统的操作**，生产上应带 WHERE 利用下推，别裸扫。

#### 列怎么来的

- `LOCK_TYPE`：`"RECORD"` / `"TABLE"`。
- `LOCK_MODE`：由 `lock_get_mode_str()` 按位拼接——`mode`（去掉 `LOCK_` 前缀，因为列宽只有 varchar(32)）+ 按常量表升序追加的标志：`X` / `X,GAP` / `X,REC_NOT_GAP` / `X,GAP,INSERT_INTENTION` / `IX` / `AUTO_INC`；含未识别位则 `UNKNOWN`。注意 **`LOCK_WAIT` 被显式剔除**（等待状态由 `LOCK_STATUS` 表达），且 next-key（`LOCK_ORDINARY`）没有额外标志位，所以 RR 默认的 next-key 锁在表里就显示为 **`X`**（和 record lock 长得一样，别误判）。另一个实现细节：PFS 不负责释放字符串，所以这里用一个 `unordered_map` 做**永久缓存**（永不 free，故必须在 exclusive latch 下调用）。
- `LOCK_STATUS`：`lock == trx->lock.wait_lock ? "WAITING" : "GRANTED"`——判据非常朴素。
- `LOCK_DATA`：记录锁主键各字段的拼接（`p_s_fill_lock_data`），infimum/supremum 有专门字面量（**gap 锁锁到 supremum 在表里的样子**）。用 `buf_page_try_get` **非阻塞取页——页不在 buffer pool 就返回 NULL，绝不读盘**（"a NULL is a valid result, not a failure"），这是"信息完整度"换"快照一致性与持有时间有界"。
- `THREAD_ID`/`EVENT_ID`：建锁时刻打点的 `lock_t::m_psi_internal_thread_id/m_psi_event_id`（`PSI_THREAD_CALL(get_current_thread_event_id)`），**这是 data_locks 独有的能力**——可 JOIN `events_statements_history` 定位"是哪条 SQL 加的这把锁"。
- 一个 `lock_t` 的 bitmap 有 N 个置位 bit 就会展开成表里 N 行。

#### data_lock_waits 的边是"现场重算"，不是读 blocking_trx

这是最容易误解的一点：`data_lock_waits` 的等待边**不是** `trx->lock.blocking_trx`（那是**单值**，只服务于死锁检测与 CATS 调度）。表的做法是对每个 `que_state == TRX_QUE_LOCK_WAIT` 的事务，从它的 `wait_lock` 出发用 `lock_queue_iterator_get_prev()` 沿锁队列往前遍历，对每个前驱调 `locksys::has_to_wait()` 做真实冲突判定——所以一个等待锁可能产出**多行**（同时被多个事务阻塞），更贴近事实；代价是要 GROUP BY 才能看清"谁在阻塞我"。

#### ★ 一个必须纠正的流传说法：`lock_report_wait_for_edge_to_server` 上报的不是 PFS

`lock_create_wait_for_edge` / `lock_update_wait_for_edge` 里除了维护 `blocking_trx`，还会调 `lock_report_wait_for_edge_to_server()`。顺着它追下去是 `thd_report_lock_wait()` → 只在 **`is_mts_worker(self) && is_mts_worker(wait_for)`** 时触发 `Commit_order_manager::check_and_report_deadlock()`——它服务于**复制 MTS（多线程从库）的跨子系统死锁检测**（worker A 等 worker B 的行锁、B 又等 A 的 commit-order 授权，InnoDB 自己检测不到）。**非复制场景是完全的空操作，8.0.39 中它没有任何 PFS 写入路径。**

同理，`lock_wait_suspend_thread` 里的 `thd_wait_begin(trx->mysql_thd, THD_WAIT_ROW_LOCK)` 是 **thread pool 插件回调**（"我要去睡了，请再开/唤醒一个 worker"），不是 PFS instrument——**8.0 的 PFS 里根本没有 InnoDB 行锁等待 instrument**（`wait/lock/*` 只有 `wait/lock/metadata/sql/mdl` 是真 instrument）。所以行锁等待期间 `events_waits_current` 不会有任何行，只能看到 `events_statements_current` 里那条语句一直 RUNNING、计时器在涨。

结论：锁等待对 server 的暴露是**纯拉模型（pull）**——InnoDB 只在内存维护状态（锁队列、`wait_lock`、`blocking_trx`），任何可见性都由查询时刻的扫描产生。唯一的 push 通道是给复制 MTS 的，与 PFS 无关。这也是 `data_lock_waits` 只能给"当前这一瞬间的快照"、给不了"过去 5 分钟谁等过谁"的原因。

### lock 子系统计数器（INNODB_METRICS）

monitor counter 是 **InnoDB 全局的计数器体系**（位图开关、无锁累加、`MONITOR_EXISTING` 复用），完整机制见 [`../../innodb/monitor.md`](../../../innodb/monitor.md)。`lock` 子系统共 22 个计数器，**默认开 13 个**（含 5 个 EXISTING），与本文直接相关的：

| NAME | 默认 | 说明 / 埋点位置 |
|---|---|---|
| `lock_deadlocks` | ✅ | 真死锁次数（`lock_wait_find_and_handle_deadlocks` 校验通过） |
| `lock_deadlock_false_positives` | ✅ | **假阳性**次数（候选环被两道校验否决）——不为 0 即证明"快照并发检测撞过假环" |
| `lock_deadlock_rounds` | ✅ | wait-for graph 扫描轮数 |
| `lock_threads_waiting` | ✅ | 当前睡眠等锁线程数（`MONITOR_SET` 写现值，配 `DISPLAY_CURRENT`） |
| `lock_timeouts` | ✅ | 锁超时次数 |
| `lock_rec_release_attempts` / `lock_rec_grant_attempts` | ✅ | 释放/授予尝试次数（`lock_rec_grant` 附近） |
| `lock_schedule_refreshes` | ✅ | 权重刷新次数（**原生计数器，非 OVLD**） |
| `lock_rec_lock_waits` | ❌ | 记录锁入等待队列次数（`RecLock::add_to_waitq`）——**入队就 +1，不管等多久** |
| `lock_table_lock_waits` | ❌ | 表锁入等待队列次数 |
| `lock_rec_lock_requests` / `lock_rec_lock_created` / `lock_rec_lock_removed` / `lock_rec_locks` | ❌ | 请求/创建/移除/当前持有数 |
| `lock_row_lock_waits` / `_time` / `_time_max` / `_time_avg` / `_current_waits` | ✅ | EXISTING，映射 `Innodb_row_lock_*` |

两个易混的"等待次数"：`lock_rec_lock_waits` 是**入队次数**，而 `Innodb_row_lock_waits` 是**实际挂起次数**（`lock0wait.cc` 真正睡下去才计，且只计行锁）——超时场景两者明显不一致。9 个默认关闭项需 `SET GLOBAL innodb_monitor_enable='module_lock'` 才会计数，查到 0 是"没开"不是"没发生"。

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `transaction_isolation` | REPEATABLE-READ | Global/Session | 决定是否加 gap lock（`skip_gap_locks()`） |
| `innodb_lock_wait_timeout` | 50 | Global/Session | 锁等待超时（秒）；`innodb_deadlock_detect=OFF` 时的死锁兜底 |
| `innodb_deadlock_detect` | ON | Global | 死锁检测开关；OFF 则跳过检测、靠超时 |
| `innodb_status_output` | OFF | Global | 周期线程是否把 monitor 输出写进 error log（与 `innodb_status_output_locks` 独立） |
| `innodb_status_output_locks` | OFF | Global | 锁段是否逐个打印每把锁（每事务最多 10 把）；需配合 SHOW 或 `innodb_status_output` 才可见 |
| `innodb_status_file` | OFF | Global | 是否用 datadir 下的 `innodb_status.<pid>` 具名文件代替匿名临时文件 |
| `innodb_print_all_deadlocks` | OFF | Global | 每次死锁都打印到 error log |
| `innodb_monitor_enable/disable/reset` | NULL | Global | 计数器开关（名字/模块/`%` 通配） |

| 状态变量 | 说明 |
|--------|------|
| `Innodb_row_lock_waits` | 行锁等待次数（实际挂起） |
| `Innodb_row_lock_time` | 行锁等待总时长（毫秒） |
| `Innodb_row_lock_time_avg` / `_max` | 平均/最长等待 |
| `Innodb_row_lock_current_waits` | 当前正在等待的行锁数（只计行锁，表锁不计） |
| `Innodb_truncated_status_writes` | SHOW ENGINE STATUS 输出被 1MB 截断的次数 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 | 代价 |
|--------|------|------|------|
| 谁持锁、谁等锁（可过滤聚合） | PFS | `performance_schema.data_locks`（一行 = 一把锁的一个 heap_no bit） | 中（持全局 latch） |
| 等待关系（谁阻塞谁） | PFS | `performance_schema.data_lock_waits`；`sys.innodb_lock_waits` | 中 |
| 加这把锁的是哪条 SQL | PFS | 用 `data_locks.THREAD_ID/EVENT_ID` JOIN `events_statements_history` | 低 |
| 锁的次数与耗时趋势 | SQL | `INNODB_METRICS`（`lock_*`）+ `SHOW STATUS LIKE 'Innodb_row_lock%'` | 极低 |
| 死锁详情（最近一次） | SQL | `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 段（来自 `lock_latest_err_file`） | 重 |
| 锁全景 + 事务/undo/历史 | SQL | `SHOW ENGINE INNODB STATUS` 的 `TRANSACTIONS` 段 | 重 |
| 每次死锁都留痕 | trace | `innodb_print_all_deadlocks=ON` 进 error log | 低 |

`---TRANSACTION` 段读法：`TABLE LOCK table ... lock mode IX` = 表级意向锁；`RECORD LOCKS space id N page no N n bits N index ... lock_mode X locks rec but not gap` = 记录锁（`waiting` 后缀表示等待中）；`Record lock, heap no N PHYSICAL RECORD: ...` = 被锁记录（heap no + 字段 hex，含 `DB_TRX_ID`/`DB_ROLL_PTR`）；`TRX HAS BEEN WAITING N SEC FOR THIS LOCK TO BE GRANTED` = 已等待时长。

---

## Misc

### 易混淆概念

**锁 vs 版本（锁与 MVCC 正交）**

- "记录被锁" = rec 上有不兼容锁（要改它得等）。
- "读旧版本" = 沿 rec 的 undo 链找已提交历史值。
- 二者正交：一条记录可以同时"被锁着"（别人在改）和"有可读旧版本"（我能读）。

**幻读 vs 不可重复读**

- 不可重复读：同一行两次读到不同值（record lock 防）。
- 幻读：范围查询多出/少了行（gap/next-key lock 防）。

**gap lock 的精确语义**："gap lock 禁止整个 range 任意变化"是误解——它**只禁止 INSERT**，不禁止已有记录的改/删（那是 record lock 的职责）。

**`innodb_row_lock_current_waits` 背后的坑**：该状态变量统计的是锁等待，但"等待"的成因也可能是索引树锁（`dict_index_t::lock`，保护索引内存结构的 rwlock）而非行锁——见 [`../README.md`](../README.md) 盘点 C 节。

### 面向二次开发

**扩展点**：新增一种锁形态 = `type_mode` 占一个新 bit（需 `lock0priv.h` 的位布局与注释同步）+ 兼容性矩阵（若涉及新模式）+ `type_mode_string()` 的打印分支。新增一种锁类型（第三种大类）= `lock_t` union 加成员 + `lock_get_type_low` 判据 + 独立 hash 表（参考谓词锁的 `prdt_hash`）。

**坑与已知缺陷**：

- **gap lock 阻塞 INSERT 并发**是 RR 的固有代价；RC 下无 gap lock 但幻读。二选一，没有"既要并发又要防幻读"的免费午餐。
- **锁多内存涨**：无锁升级，大事务批量改行时 `lock_t` 数量线性增长（`trx->lock.trx_locks` 链表遍历也会变慢）。
- **`innodb_deadlock_detect=OFF` 的代价**：官方注释明言"rely on innodb_lock_wait_timeout in case of deadlock"——死锁不再即时拆环，全部事务等到超时，吞吐骤降。
- **谓词锁 vs 行锁**：谓词锁 `type_mode` 里带 `LOCK_REC` 位，`lock_get_type_low()` 看它就是 `LOCK_REC`，因此 PFS 的 `LOCK_TYPE` 显示 `RECORD` 而非 `PREDICATE`——识别它只能看 `LOCK_MODE` 的 `PREDICATE`/`PRDT_PAGE` 后缀。它挂 `prdt_hash`/`prdt_page_hash`，但**键与 `rec_hash` 相同（都是 `page_id`），谓词锁不跨页**。普通 B-tree 索引用不到（在 R-tree 上请求 `LOCK_GAP`/`LOCK_ORDINARY` 会直接报错）。GIS 侧背景见 [`../../server/datatype/gis.md`](../../../server/datatype/gis.md)。
- **AUTOINC 锁 vs 其它表锁**：唯一**语句级**释放的锁（其它都是事务级）。且它保护的不是计数器数值（`autoinc_mutex` 管这个），而是**分配的排他性**，目的是 SBR 可重放。SQL 层的配合（三档模式选择、`nb_desired_values` 计算）见 [`../../feat/auto_increment.md`](../../../feat/auto_increment.md)。

---

## 参考

**论文**
- C. Mohan et al. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. TODS 1992.（next-key locking / WAL 与锁配合）
- J. Gray, A. Reuter. *Transaction Processing: Concepts and Techniques*. Morgan Kaufmann, 1993.（多粒度锁、锁模式兼容矩阵、死锁检测体系）
- J. Gray, R. Lorie, G. Putzolu, I. Traiger. *Granularity of Locks and Degrees of Consistency in a Shared Data Base*. 1976.（意向锁 IS/IX 出处）

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Locking*
- WorkLog: [WL#10314](https://dev.mysql.com/worklog/task/?id=10314) Reducing lock_sys contention（8.0.21 分片锁与等待 slot）

**内核月报 / 技术文章**
- [《MySQL · 引擎特性 · InnoDB 事务锁系统简介》（数据库内核月报）](https://www.kancloud.cn/taobaomysql/monthly/117956)——行锁/表锁概念与实现的总览，两个死锁案例有借鉴价值
- [《MySQL · 引擎特性 · InnoDB 事务子系统介绍》（数据库内核月报）](https://www.kancloud.cn/taobaomysql/monthly/96586)——事务侧背景
- ⚠️ 月报基于 5.x/早期版本，**函数名与行号与 8.0.39 基本对不上**（如 `lock_rec_has_to_wait` 在 8.0 已拆分为 `rec_lock_check_conflict` → `locksys::has_to_wait`；`row_lock_table_for_mysql` 已改名为 `row_lock_table`）。本篇一律以本仓库源码为准，月报仅作问题视角参考。

**相关文档**
- server 层元数据锁（MDL）见 [`mdl.md`](mdl.md)——两套锁的分工见其 Misc
- 锁目录全量盘点（同步原语 vs 事务锁、lock_sys 原语清单）见 [`../README.md`](../README.md)
- 本篇已覆盖 AUTOINC 锁（三档模式、语句级释放、持久化）与 R-tree 谓词锁（MBR 谓词、三张 hash 表、插入意向独占等待）；SQL 层自增语义见 [`../../feat/auto_increment.md`](../../../feat/auto_increment.md)，空间索引结构见 [`../../server/datatype/gis.md`](../../../server/datatype/gis.md)
