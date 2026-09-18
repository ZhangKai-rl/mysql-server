# InnoDB 事务锁（lock_t 体系）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 `lock_t` 一统表锁/行锁的结构与 `type_mode` 位布局、record/gap/next-key/insert intention 锁语义、等待队列复用存储的挂载方式与 `os_event` 唤醒、wait-for graph 死锁检测、锁与 MVCC 版本/半一致性读的交互。
>
> **边界**：本篇讲 InnoDB 事务锁（`lock_t` 体系，行锁为主）。server 层元数据锁见 [`mdl.md`](mdl.md)；AUTOINC 锁详情见 [`../../feat/auto_increment.md`](../../feat/auto_increment.md)；保护 `lock_sys` 内存结构的同步原语（`locksys::Latches` 分片锁）是原语、非事务锁，其盘点见 [`../README.md`](../README.md) A3 节。

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

InnoDB 的事务锁（transactional lock，区别于保护内存的同步原语）**收敛在 `lock_t` 一个结构里**——`type_mode` 按位区分表锁（`LOCK_TABLE`）与行锁（`LOCK_REC`），行锁再用标志位细分四种形态：

- **record lock**（`LOCK_REC_NOT_GAP`）：锁一条记录，防"两个事务同时改同一行"。
- **gap lock**（`LOCK_GAP`）：锁记录之前的间隙，防"间隙内 INSERT"。
- **next-key lock**（`LOCK_ORDINARY` = 0，record + gap）：上面两者的组合，RR 默认形态，防幻读。
- **insert intention lock**（`LOCK_INSERT_INTENTION`）：插入意向锁，`INSERT` 前在间隙上打标，多个插入互不阻塞、只与 gap lock 冲突。

表锁侧有 IS/IX/S/X 四种意向/常规锁（AUTOINC 锁也借 `LOCK_TABLE` 承载，见 auto_increment 篇），另有 R-tree 空间索引专用的谓词锁（`LOCK_PREDICATE` / `LOCK_PRDT_PAGE`）。

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

#### 表级意向锁

普通 DML 在表上只放 **IS/IX**（行锁的"电梯"）：DDL 要拿表 X 锁时，查表的意向锁即可知道"有没有行锁存在"，不必扫所有行。**表级 S/X 只由 `LOCK TABLES` 获取**（`lock0priv.h` 矩阵注释明写）。

`lock_table`（表锁统一入口）的流程：拿 `Shard_latch_guard`（表分片锁）→ `lock_table_other_has_incompatible` 检查表锁队列里有没有不兼容请求（`wait=LOCK_WAIT` 表示连等待中的也算）→ 冲突则进等待，兼容则 `lock_table_create` 建锁入队。DML 侧有专门的快路径直接无条件建 IX（`lock_table_create(table, LOCK_IX, trx)`）——写操作必拿 IX，若本事务已有 IX 则复用。

**AUTOINC 锁**（`LOCK_AUTO_INC`）也借表锁承载：兼容性矩阵里 AI 只与 IS/IX 兼容、连 AI 自己都不兼容（同一时刻只能一个事务拿 AUTOINC），强度上 X 覆盖 AI 而 AI 只等于自己。三档模式（传统/连续/交错）是跨层特性，见 [`../../feat/auto_increment.md`](../../feat/auto_increment.md)。

**谓词锁**（`LOCK_PREDICATE`/`LOCK_PRDT_PAGE`）是第四种大类：R-tree 空间索引专用（`lock0prdt.cc`），挂在独立的 `prdt_hash`/`prdt_page_hash` 两张 hash 表上，谓词锁统一挂在页 infimum 记录（`PRDT_HEAPNO = PAGE_HEAP_NO_INFIMUM`）——因为"空间范围"没有唯一锚点记录，只能用页级伪记录承载。普通 B-tree 用不到，细节见 [`../../server/datatype/gis.md`](../../server/datatype/gis.md)。

#### 行锁的四种形态

- **record lock**（`LOCK_REC_NOT_GAP`）：只锁记录本身。RC 下的"锁定读"用它；4.0.5 引入以模拟 Oracle 式 RC。
- **gap lock**（`LOCK_GAP`）：锁**记录之前的间隙**，只禁止向间隙 INSERT。`supremum` 伪记录上的 gap lock = 锁"页内最后一条记录之后的间隙"（即页尾到下一页之间的空隙）。
- **next-key lock**（`LOCK_ORDINARY=0`）：record + gap。RR 默认形态——防"改这条"也防"往它前面插"。
- **insert intention lock**（`LOCK_INSERT_INTENTION`）：INSERT 时在目标间隙上打的"我要插这"标记。多个 insert intention 互相兼容（大家都插，不冲突），只与 gap/next-key lock 冲突——所以 gap lock 能挡住 INSERT，INSERT 之间却不互相阻塞。

`lock_mode_is_next_key_lock` 里的 `static_assert(LOCK_ORDINARY == 0)` 点明了设计：next-key 是"零标志位"的默认形态，加形态位都是"减锁"（`LOCK_GAP` 去掉 record 部分、`LOCK_REC_NOT_GAP` 去掉 gap 部分）。

#### 隐含锁（implicit lock）

**修改一条记录时，默认不建显式 `lock_t`**——记录本身带 `DB_TRX_ID`（谁最后改的），这本身就是一把隐式的 `LOCK_X | LOCK_REC_NOT_GAP`：

- 注释明言（`lock0priv.h`）："An implicit x-lock does not affect the gap, it only locks the index record from read or update"。
- 只有当**别人来检查冲突**时，才把隐含锁"物化"成显式 `lock_t`（`lock_rec_convert_impl_to_expl`）——检查聚簇索引记录的 `DB_TRX_ID` 对应事务是否还活跃（`row_vers_impl_x_locked`），活跃则视为持有 X 锁。
- 二级索引记录无 `DB_TRX_ID`，先用页头 `PAGE_MAX_TRX_ID` 粗筛（"can return false positives but never false negatives"），再回聚簇索引确认。
- 这就是"INSERT 一条未提交的行，别人 UPDATE 它要等"的实现——没有锁对象，只有行上未提交的事务 ID。

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

#### 挂起与唤醒

**挂起**（`lock_wait_suspend_thread`）：从 `waiting_threads` 找到空槽 → 绑定 thr → `os_event_reset(slot->event)` → `thd_wait_begin`（PFS 上报 THD_WAIT_ROW_LOCK/TABLE_LOCK）→ `os_event_wait(slot->event)` 睡眠。

**唤醒**（释放锁触发）：`lock_rec_grant_by_heap_no` 在释放锁的 hash cell 上遍历**同位置（同 heap_no）的等待锁**，只筛选 `blocking_trx == 释放事务` 的候选；再逐个用 `lock_rec_has_to_wait_for_granted` 复核"与当前 granted 集合是否仍冲突"，不冲突则 `lock_grant(wait_lock)`：

```cpp
static void lock_grant(lock_t *lock) {
  trx_mutex_enter(lock->trx);      // lock->trx 是等待事务（被唤醒方）
  lock_reset_wait_and_release_thread_if_suspended(lock);
  //   ↑ trx->lock.wait_lock = NULL；找到 wait_thr → thr->slot
  //     → os_event_set(thr->slot->event)  ← 最终唤醒
  trx_mutex_exit(lock->trx);
}
```

**"最多被唤醒一次"的保证**（`lock0wait.cc` 注释写明）：唤醒的唯一途径是 `os_event_set`，而它只由 `lock_wait_release_thread_if_suspended` 调用，其前置条件是 `lock_reset_wait_and_trx_wait` 已把 `wait_lock` 置 NULL——第二次唤醒尝试会发现 `wait_lock == NULL` 而直接返回，不会重复 signal。

**另外两条唤醒路径**：死锁 victim（检测线程置 `was_chosen_as_deadlock_victim` 后走同一唤醒路径，被唤醒线程见该标志设 `error_state = DB_DEADLOCK`）；超时（`lock_wait_timeout_thread` 周期扫描槽位，到期者 `lock_cancel_waiting_and_release` 取消等待）。

### 死锁检测

#### wait-for graph 的构建与环检测

后台线程 `lock_wait_timeout_thread` 周期性（`lock_set_timeout_event` 控制）执行 `lock_wait_update_schedule_and_check_for_deadlocks`：

```cpp
lock_wait_snapshot_waiting_threads(infos);        // 1. 快照：所有等待槽位 → waiting_trx_info_t 数组
lock_wait_build_wait_for_graph(infos, outgoing);  // 2. 图：infos[i].waits_for → outgoing[i]（等待对象）
lock_wait_compute_and_publish_weights_except_cycles(...);  // 3. 权重：非环上的事务计算调度权重
if (innobase_deadlock_detect) {
  lock_wait_find_and_handle_deadlocks(...);       // 4. 找环：DFS 环检测 + 选 victim 回滚
}
```

- 快照用 `reservation_no` 版本号去重（同一事务的多个等待槽位只保留最新，见 `lock0wait.cc` 注释）。
- `innobase_deadlock_detect=OFF` 时跳过检测，死锁靠 `innodb_lock_wait_timeout` 兜底。

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

**假阳性防御（两重校验）**：快照后到检测前，等待关系可能已变化（锁被授予、超时、事务重入槽位）。`lock_wait_check_candidate_cycle` 用两道关卡确认环仍真实存在：`lock_wait_trxs_are_still_in_slots`（比对每个槽位当前 `reservation_no` 与快照值，防 ABA——事务离开又重入同一槽位）和 `lock_wait_trxs_are_still_waiting`（X 锁 global 后逐个查 `trx->lock.wait_lock` 仍非 NULL）。两道都过才真正动手，否则计入 `MONITOR_DEADLOCK_FALSE_POSITIVES`——这个计数器存在本身就说明"快照并发检测"会撞上假环。

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

- **权重 = 锁数量 + undo 量**（`trx_weight_ge`，`trx0trx.cc`）：注释明言"transactions that have edited non-transactional tables are considered heavier"——已改非事务表的不能回滚（回滚不了 MyISAM 的改动），权重最高。最小者回滚，代价最低，且回滚它的锁最可能解开整个环。
- **环上遍历顺序有讲究**（`lock_wait_order_for_choosing_victim`）：把"最后加入等待"的事务（reservation_no 最新，即"闭合环"的那条边）与其前驱移到队尾，平局时最新等待者优先被牺牲——注释说明这是为了让历史测试行为稳定（闭环者通常是刚进来、工作最少、最该回滚的）。
- **高优先级事务仲裁**（`trx_arbitrate`）：`thd_trx_priority > 0` 的事务参与仲裁而非纯比重量——DDL 类事务（无 `mysql_thd` 或非用户事务）"should never rollback"，注释点明它们本就不该在表/行锁上等待。高优先级事务还有 `trx_kill_blocking` 主动杀阻塞者的机制。

选出 victim 后置 `was_chosen_as_deadlock_victim` 标志，走与 grant 相同的唤醒路径（`lock_wait_release_thread_if_suspended` 里检测该标志 → `error_state = DB_DEADLOCK`）；死锁详情由 `Deadlock_notifier` 写日志并缓存，供 `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 段读取。

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

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `transaction_isolation` | REPEATABLE-READ | Global/Session | 决定是否加 gap lock（`skip_gap_locks()`） |
| `innodb_lock_wait_timeout` | 50 | Global/Session | 锁等待超时（秒）；`innodb_deadlock_detect=OFF` 时的死锁兜底 |
| `innodb_deadlock_detect` | ON | Global | 死锁检测开关；OFF 则跳过检测、靠超时 |
| `innodb_status_output_locks` | OFF | Global | `SHOW ENGINE INNODB STATUS` 是否输出完整锁列表（需 `innodb_status_output=ON`） |
| `innodb_print_all_deadlocks` | OFF | Global | 每次死锁都打印到 error log |

| 状态变量 | 说明 |
|--------|------|
| `Innodb_row_lock_waits` | 行锁等待次数 |
| `Innodb_row_lock_time` | 行锁等待总时长 |
| `Innodb_row_lock_current_waits` | 当前正在等待的行锁数 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 谁持锁、谁等锁（实时全量） | SQL | `performance_schema.data_locks`（一行一把，含 LOCK_TYPE/LOCK_MODE/LOCK_STATUS/LOCK_DATA） |
| 等待关系（谁阻塞谁） | SQL | `performance_schema.data_lock_waits`；`sys.innodb_lock_waits`（直接给等待方/阻塞方/等待 SQL） |
| 锁等待的文本快照 | SQL | `SHOW ENGINE INNODB STATUS` 的 `---TRANSACTION` 段（先 `SET GLOBAL innodb_status_output_locks=ON` 才显示 IX 锁和完整记录锁） |
| 事务级等待状态 | SQL | `information_schema.innodb_trx`（trx_query/trx_started/trx_state） |
| 死锁历史 | trace | `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 段；`innodb_print_all_deadlocks=ON` 进 error log |

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
- **谓词锁**：R-tree 空间索引专用（`lock0prdt.cc`），普通 B-tree 索引用不到；其 hash 表独立于 `rec_hash`。细节见 [`../../server/datatype/gis.md`](../../server/datatype/gis.md)。
- **AUTOINC 锁**：自增计数器的特殊锁，三档模式（传统/连续/交错），是跨层特性，见 [`../../feat/auto_increment.md`](../../feat/auto_increment.md)。

---

## 参考

**论文**
- C. Mohan et al. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. TODS 1992.（next-key locking / WAL 与锁配合）
- J. Gray, A. Reuter. *Transaction Processing: Concepts and Techniques*. Morgan Kaufmann, 1993.（多粒度锁、锁模式兼容矩阵、死锁检测体系）
- J. Gray, R. Lorie, G. Putzolu, I. Traiger. *Granularity of Locks and Degrees of Consistency in a Shared Data Base*. 1976.（意向锁 IS/IX 出处）

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Locking*
- WorkLog: [WL#10314](https://dev.mysql.com/worklog/task/?id=10314) Reducing lock_sys contention（8.0.21 分片锁与等待 slot）

**相关文档**
- server 层元数据锁（MDL）见 [`mdl.md`](mdl.md)——两套锁的分工见其 Misc
- 锁目录全量盘点（同步原语 vs 事务锁、lock_sys 原语清单）见 [`../README.md`](../README.md)
- AUTOINC 锁见 [`../../feat/auto_increment.md`](../../feat/auto_increment.md)；R-tree 谓词锁见 [`../../server/datatype/gis.md`](../../server/datatype/gis.md)
