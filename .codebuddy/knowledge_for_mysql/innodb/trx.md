# InnoDB 事务（trx_t）深度解析

> 基于 MySQL 8.0.39 源码，涵盖事务在 server/InnoDB 间的传递、事务分配与启动、状态机、事务提交（undo 序列化 / trx->no / 放锁 / 可见性）、事务与 binlog 的 2PC、事务与 BGC 协同、回滚机制。
>
> **边界**：本篇讲 InnoDB 侧事务对象 `trx_t` 的生命周期与提交/回滚；binlog 侧的 `ordered_commit` 三阶段流水线见 [`../server/replication/binlog.md`](../server/replication/binlog.md)，GTID 持久化见 [`../server/replication/gtid.md`](../server/replication/gtid.md)。
>
> 交叉：ReadView 与可见性判定见 [`mvcc.md`](mvcc.md)；undo 存储与 purge 执行见 [`undo_log.md`](undo_log.md)；事务与**索引**的交叉地带（回表、delete-mark 两阶段、唯一性加锁、purge 判定接口）见 [`../index/transaction.md`](../index/transaction.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [主链路](#主链路)
    - [事务在 server/InnoDB 的传递](#事务在-serverinnodb-的传递)
    - [事务的分配与启动](#事务的分配与启动)
    - [事务状态机](#事务状态机)
  - 生命周期（启动 → 提交/回滚 → 崩溃恢复）
    - [read view 的生命周期](#read-view-的生命周期)
    - [事务提交](#事务提交)
    - [回滚机制](#回滚机制)
    - [崩溃恢复的提交与回滚](#崩溃恢复的提交与回滚)
  - 与 binlog/BGC 协同
    - [事务与 binlog：2PC](#事务与-binlog2pc)
    - [事务与 BGC 协同](#事务与-bgc-协同)
    - [2PC 与 BGC 中的锁](#2pc-与-bgc-中的锁)
  - 并发设计
    - [整体并发设计](#整体并发设计)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

`trx_t` 是 InnoDB 中事务的内存表示。每个 MySQL 连接（`THD`）通过 `innodb_session_t` 持有自己的 `trx_t`，贯穿从 `TRX_STATE_NOT_STARTED` 到 `TRX_STATE_COMMITTED_IN_MEMORY` 的完整生命周期。

一个反直觉的切入点：**事务"开始"这个动作本身是惰性的**。`BEGIN` 语句执行完，InnoDB 侧其实什么都没发生——`trx_t` 可能还没分配，`trx->id` 一定是 0。真正的启动发生在第一条 DML 触碰 InnoDB 时。这个惰性设计是理解很多现象（如"只读事务不产生 trx_id"）的基础。

### 用途

1. **undo log 的组织单元**：事务的 undo 记录挂在自己的 rollback segment（rseg）下，回滚时按 `undo_no` 逆序撤销。
2. **锁的所有者**：`trx->lock.trx_locks` 链表记录持有的所有锁，提交/回滚时统一释放。
3. **MVCC 的载体**：`trx->id`（写进记录的 `DB_TRX_ID`）和 `trx->no`（提交序列号）共同决定行的可见性。
4. **2PC 参与者**：`state == TRX_STATE_PREPARED` 时事务进入 prepare 态，等待 binlog 协调者裁决。

在整条链路中的位置：server 层 `THD` 持有事务上下文 → handler 层 `ha_commit_trans` 驱动 → InnoDB 侧 `trx_t` 执行真正的持久化与内存态收尾。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.5 及更早 | `trx_t` 字段堆叠，全局 `kernel_mutex` 保护，提交严重串行化 |
| 5.6 | 拆分 `kernel_mutex`；引入独立 `trx_sys->mutex`；redo / undo 线程独立 |
| 5.7 | 引入 `TRX_FORCE_ROLLBACK`（高优先级事务异步杀阻塞者）；AC-NL-RO 优化；`trx->no` 语义清晰化 |
| 8.0 | `trx->state` / `trx->no` 改用 `std::atomic`；新增 `TRX_STATE_FORCED_ROLLBACK`；`Trx_shard` 分片（`trx_sys->get_shard_by_trx_id`）替代全局链表；undo 表空间可截断 |

---

## 理论基础

### 设计思想与权衡

#### ① 惰性启动：为什么 `BEGIN` 不启动事务

InnoDB 的事务启动被推迟到第一条真正需要它的 DML。带来的收益：

- **只读事务零成本**：`SELECT` 不启动事务，不分配 `trx->id`，不进任何链表。
- **AC-NL-RO 快路径**：autocommit 非加锁只读事务连 `trx->id` 都不分配（`trx->id = 0`），直接走 `trx_commit_in_memory` 的简化分支。

代价：**"事务是否活跃"不能只看 `BEGIN`**，要看 `trx->state`。这给状态判断带来额外的分支复杂度，`trx_start_low` 里那一堆 `read_only` / `auto_commit` / `will_lock` 的组合判断就是代价的体现。

#### ② `trx->id` 与 `trx->no` 分离：两个正交的"序"

⚠️ **勘误**：`trx->no` 在 `trx0trx.h` 里**只有 7 行注释**（`:752-758`），不是常说的一段长注释。真正解释"no 用于 purge 序"的在 `trx_sys_t::next_trx_id_or_no`（`trx0sys.h:489`）与 `trx0sys.cc:485`。

```cpp
// trx0trx.h:735-758
  bool abort; /*!< if this flag is set then
              this transaction must abort when
              it can */

  trx_id_t id; /*!< transaction id */

  // xxxx: 只有事务产生undate_undo时才有trx no，用于purge操作！！！！
  // trx_no 主要用于 purge 系统判断旧版本数据何时可以被清理
  trx_id_t no; /*!< transaction serialization number:
               max trx id shortly before the
               transaction is moved to
               COMMITTED_IN_MEMORY state.
               Protected by trx_sys_t::mutex
               when trx->in_rw_trx_list. Initially
               set to TRX_ID_MAX. */
```

> ★★ `id` 与 `no` **类型相同**（都是 `trx_id_t` = `ib_id_t` = `uint64_t`），但语义正交：
> - `id` —— 事务**开始时**分配的"谁写的"，写进行头 `DB_TRX_ID`，供 **MVCC 可见性**判断；
> - `no` —— 事务**提交时**分配的"何时序列化"，只用于 **purge 排序**与 `ReadView` 上界。
>
> 初值 `TRX_ID_MAX`（= `UINT64_MAX`）表示"尚未序列化"。未提交事务的 `no` 一律是最大值，因为 purge 必须跳过它们。
> 每次 `trx_start_low()` 都会把 `no` 复位为 `TRX_ID_MAX`（`trx0trx.cc:1405`），因为它是"提交时才赋"的字段。

**一个计数器，两种用途**（`trx0sys.ic:229`）——⚠️ **不存在 `trx_sys_get_new_trx_id()`**，真名 `trx_sys_allocate_trx_id()`：

```cpp
inline trx_id_t trx_sys_allocate_trx_id_or_no() {
  ut_ad(trx_sys_mutex_own() || trx_sys_serialisation_mutex_own());

  trx_id_t trx_id = trx_sys->next_trx_id_or_no.fetch_add(1);

  if (trx_id % trx_sys_get_trx_id_write_margin() == 0) {
    /* Reserve the next range of trx_id values. This thread has acquired
    either the trx_sys_mutex or the trx_sys_serialisation_mutex. ... */
    trx_sys_write_max_trx_id();
  }
  return trx_id;
}

inline trx_id_t trx_sys_allocate_trx_id() {
  ut_ad(trx_sys_mutex_own());                    // ★ 发 id 持 trx_sys_mutex
  return trx_sys_allocate_trx_id_or_no();
}

inline trx_id_t trx_sys_allocate_trx_no() {
  ut_ad(trx_sys_serialisation_mutex_own());      // ★ 发 no 持 serialisation_mutex
  return trx_sys_allocate_trx_id_or_no();
}

/** Reads trx->no up to which all transactions have been serialised. */
static inline trx_id_t trx_get_serialisation_min_trx_no(void) {
  return (trx_sys->serialisation_min_trx_no.load());
}

inline trx_id_t trx_sys_get_next_trx_id_or_no() {
  return trx_sys->next_trx_id_or_no.load();      // ★ 纯读，不分配
}
```

> ★★ **两条发号流可真并发**：`next_trx_id_or_no` 是唯一的 64 位发号器，但 id 分配与 no 分配由**两个不同的 mutex** 串行化，互不阻塞。这正是启动时初始化要 `+2 * MARGIN` 的原因。

**落盘节流 `TRX_SYS_TRX_ID_WRITE_MARGIN = 256`**（`trx0sys.h:602`）：

```cpp
/** When a trx id which is zero modulo this number (which must be a power of
two) is assigned, the field TRX_SYS_TRX_ID_STORE on the transaction system
page is updated */
constexpr trx_id_t TRX_SYS_TRX_ID_WRITE_MARGIN = 256;
```

```cpp
// trx0sys.cc:485 —— 启动时：为什么是 +2*MARGIN
  /* VERY important: after the database is started, next_trx_id_or_no value
  needs to be set to a higher value than the maximum of values that have ever
  been used for either trx->id or trx->no. ...

  Note, that the factor 2 in 2 * TRX_SYS_TRX_ID_WRITE_MARGIN is required,
  because the next_trx_id_or_no might be increased concurrently in two
  threads:
    - one that has acquired trx_sys_mutex,
    - and one that has acquired the trx_sys_serialisation_mutex.
  If you decreased the factor 2, the test innodb.max_trx_id should fail. */

  trx_sys->next_trx_id_or_no.store(max_trx_id +
                                   2 * trx_sys_get_trx_id_write_margin());
  trx_sys->serialisation_min_trx_no.store(trx_sys->next_trx_id_or_no.load());
```

**`trx->no` 的真正分配点**（`trx0trx.cc:1496`，在 `serialisation_mutex` 下）：

```cpp
static inline bool trx_add_to_serialisation_list(trx_t *trx) {
  trx_sys_serialisation_mutex_enter();

  trx->no = trx_sys_allocate_trx_no();      // ★★ trx->no 就在这里诞生
  ut_d(trx_sys->rw_max_trx_no = trx->no);

  if (trx->read_only) {                     // ★ 只读事务也拿 no，但不入链表
    trx_sys_serialisation_mutex_exit();
    return false;
  }

  UT_LIST_ADD_LAST(trx_sys->serialisation_list, trx);

  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) == 1) {
    trx_sys->serialisation_min_trx_no.store(trx->no);   // ★ ReadView 的 m_low_limit_no 来源
  }
  trx_sys_serialisation_mutex_exit();
  return true;
}
```

```cpp
// trx0trx.cc:1542 —— no 与 purge 的衔接点
  /* If the rollack segment is not empty then the new trx_t::no can't be less
  than any trx_t::no already in the rollback segment. */
  if ((redo_rseg != nullptr && redo_rseg->last_page_no == FIL_NULL) ||
      (temp_rseg != nullptr && temp_rseg->last_page_no == FIL_NULL)) {
    TrxUndoRsegs elem;
    ... elem.insert(redo_rseg); ...
    mutex_enter(&purge_sys->pq_mutex);
    added_trx_no = trx_add_to_serialisation_list(trx);
    elem.set_trx_no(trx->no);
    purge_sys->purge_queue->push(std::move(elem));   // ★★ 压进 purge 优先队列
    mutex_exit(&purge_sys->pq_mutex);
  } else {
    added_trx_no = trx_add_to_serialisation_list(trx);
  }
```

> ★ `trx_serialisation_number_get()` 必须在**持有 rseg mutex** 时调用——这保证"no 的分配顺序 == undo 头挂进 history list 的顺序"，purge 优先队列才能正确排序。

这是 InnoDB 事务模型最核心的设计决策：

| | `trx->id` | `trx->no` |
|---|---|---|
| 分配时机 | 事务**开始**（`trx_sys_allocate_trx_id`） | 事务**提交**（`trx_sys_allocate_trx_no`） |
| 运行期值 | 立即可用，写进记录的 `DB_TRX_ID` | 提交前恒为 `TRX_ID_MAX` |
| 语义 | **谁改的**（修改者标识） | **第几个提交的**（提交序列号） |
| 决定 | 行的修改者、隐式锁转换 | purge 顺序、MVCC read view 可见性 |
| 是否有 0 值 | 是（只读/AC-NL-RO 事务 `id = 0`） | 否（但只读事务不进 serialisation_list） |

**为什么必须分离**？因为 MVCC 需要两个不同的判据：

- 判断"这行是不是我改的" → 用 `trx->id`（与记录上的 `DB_TRX_ID` 比较）
- 判断"这个事务的修改我能不能看到" → 用 `trx->no`（与 read view 的 `m_low_limit_no` 比较）

如果只有一个号，两个语义会打架。举例：一个长事务 T1（`id=100`）在 T2（`id=200`）之后提交，若用 `id` 判断可见性，T2 的 read view 会错误地看到 T1 的修改。

源码里这个设计有明确注释（`trx_start_low` 中）：

```cpp
/* The initial value for trx->no: TRX_ID_MAX is used in
read_view_open_now */
trx->no = TRX_ID_MAX;
```

`TRX_ID_MAX` 是一个哨兵值——运行期 `trx->no` 取最大值，任何 read view 的 `m_low_limit_no` 都小于它，因此未完成提交的事务其修改对任何新 read view 都不可见。

**代价**：只有产生了 update undo 的事务才会分配 `trx->no`（见 `trx_write_serialisation_history` 中 `update_undo != nullptr` 的判断）。纯 INSERT 事务没有 `trx->no`，这正是 INSERT 可以"快速提交"、且 insert undo 提交后立即可以 purge 的原因。

#### ③ AC-NL-RO：为了性能放弃可回滚性

AC-NL-RO（AutoCommit Non-Locking Read-Only）事务是一类特殊优化：`trx->id = 0`、`read_only = true`、不进 `rw_trx_list`。

`trx_commit_in_memory` 里有一组断言把这个"放弃"写得很直白：

```cpp
if (trx_is_autocommit_non_locking(trx)) {
  ut_ad(trx->id == 0);
  ut_ad(trx->read_only);
  ut_a(!trx->is_recovered);
  ut_ad(trx->rsegs.m_redo.rseg == nullptr);
  ut_ad(!trx->in_rw_trx_list);
  ...
  /* AC-NL-RO transactions can't be rolled back asynchronously. */
  ut_ad(!trx->abort);
  ut_ad(!(trx->in_innodb & TRX_FORCE_ROLLBACK));

  trx->state.store(TRX_STATE_NOT_STARTED, std::memory_order_relaxed);
```

**权衡**：省掉了 `trx->id` 分配、`rw_trx_list` 链表维护、放锁遍历的全部开销（这是 OLTP 只读场景的巨大收益）。代价是**这类事务不能被异步回滚**——因为它根本不在 `rw_trx_list` 里，高优先级事务的 hit_list 扫描找不到它。这也是为什么 `TrxInInnoDB::enter` 里对 AC-NL-RO 事务要特殊处理。

#### ④ 提交不是"一个动作"而是"两个世界的状态迁移"

InnoDB 提交的精妙在于区分了两个"世界"：

- **file-based world（文件世界）**：undo 段状态、redo LSN。由 `trx_write_serialisation_history` + `mtr_commit` 完成，标志是拿到 commit LSN。
- **memory world（内存世界）**：`trx_locks`、`rw_trx_ids`、read view、`trx->state`。由 `trx_commit_in_memory` 完成，标志是状态变 `TRX_STATE_COMMITTED_IN_MEMORY`。

源码注释（`trx_commit_low`）直接点破了这一点：

```cpp
/* The following call commits the mini-transaction, making the
whole transaction committed in the file-based world, at this
log sequence number. The transaction becomes 'durable' when
we write the log to disk, but in the logical sense the commit
in the file-based data structures (undo logs etc.) happens
here. */
mtr_commit(mtr);
```

**隐含假设与失效场景**：提交在"文件世界"完成（mtr_commit）之后、"内存世界"完成（状态变 COMMITTED_IN_MEMORY）之前，存在一个窗口。这个窗口里事务在磁盘上已提交但内存里还是 ACTIVE。`trx_release_impl_and_expl_locks` 用 `Trx_shard` 的 mutex 同时保护状态迁移和 `active_rw_trxs` 移除来保证这个窗口的原子性——这正是注释里大段解释"如何检查一个 trx 是否还持有隐式锁"的原因。

### 理论溯源

- **"ARIES: A Transaction Recovery Method..."**, Mohan et al., TODS 1992 — WAL + undo 的经典。InnoDB 的 `trx_write_serialisation_history`（把 undo 段状态改为 committed、按 `trx->no` 挂入 history list）就是 ARIES commit 处理的实现；`trx_rollback_to_savepoint_low` 的逐行 undo 对应 ARIES 的 undo pass。
- **"Granularity of Locks and Degrees of Consistency in Shared Data Banks"**, Gray et al., 1975 — 2PL。`trx_release_impl_and_expl_locks` 是 shrinking phase 的实现：状态先变 `TRX_STATE_COMMITTED_IN_MEMORY`，再 `lock_trx_release_locks` 释放所有锁。
- **"Notes on Data Base Operating Systems"**, Gray, 1978 — 两阶段提交（2PC）。MySQL 内部 2PC 以 binlog 为协调者，正是该协议的应用，见下文"事务与 binlog：2PC"。

### 算法与数据结构

| 结构 | 作用 | 保护锁 |
|------|------|--------|
| `trx_sys->mysql_trx_list` | 所有 MySQL 事务对象链表（用于 `SHOW ENGINE INNODB STATUS` 遍历） | `trx_sys->mutex` |
| `trx_sys->rw_trx_list` | 读写事务链表（purge 扫描对象） | `trx_sys->mutex` |
| `trx_sys->rw_trx_ids` | 活跃事务 id 集合（MVCC 快照） | `trx_sys->mutex` |
| `Trx_shard::active_rw_trxs` | **8.0 分片**的活跃读写事务集合，替代全局遍历 | shard mutex |
| `trx_sys->serialisation_list` | 已分配 `trx->no` 但未完成提交的链表 | serialisation mutex |
| `purge_sys->purge_queue` | 按 `trx->no` 排序的优先级队列，purge 的驱动源 | `purge_sys->pq_mutex` |

**8.0 的关键演进是分片**：5.7 及以前，`rw_trx_list` / `rw_trx_ids` 是全局结构，所有事务的注册/注销都在一把 `trx_sys->mutex` 上竞争。8.0 引入 `Trx_shard`（`trx_sys->get_shard_by_trx_id(trx->id)`），按 trx_id 哈希分片，状态迁移与移除在分片锁内完成，把全局竞争打散。

### 他库对比与演进动机

| 项目 | 事务标识 | 与 InnoDB 的差异及原因 |
|------|----------|------------------------|
| PostgreSQL | XID（32 位），事务 id 即提交序 | PG 的 XID 同时承担"修改者"和"提交序"两个角色，因此有 **XID 回绕**问题，需要 vacuum freeze 兜底。InnoDB 用 64 位 `trx->id`（永不回绕）+ 独立的 `trx->no`，从设计上消除了回绕问题 |
| Oracle | undo segment 绑定事务槽，SCN 作为全局序 | Oracle 用系统级 SCN 做提交序，所有事务共享一个序；InnoDB 的 `trx->no` 只在有 update undo 的事务间分配，粒度更细但语义更窄（纯 INSERT 没有 no） |
| SQL Server | LSN 驱动，事务日志统一管理 | SQL Server 的提交序直接由日志 LSN 承载；InnoDB 分离了 LSN（物理）与 `trx->no`（逻辑），因为 MVCC 需要的是逻辑提交序而非物理位置 |

**为什么演进成今天这样**：核心驱动力是**消除全局竞争**。从 5.5 的 `kernel_mutex` 一把大锁，到 5.6 拆分，到 8.0 的 `Trx_shard` 分片 + `std::atomic` 状态，每一步都是把串行点打散。AC-NL-RO 优化则是另一个方向——**让不该有开销的路径彻底没有开销**。

---

## 核心实现

### 主链路

一个事务从第一条 DML 到提交收尾的完整生命周期（★ 标记关键节点）：

**阶段一：启动（惰性，第一条 DML 才触发）**

```
ha_innobase::write_row / update_row / delete_row     handler 接口
  → check_trx_exists(thd)                            THD → trx_t，无则分配
      → trx_allocate_for_mysql                       挂入 mysql_trx_list
      → innobase_trx_init                            同步 FK / unique 开关
  → trx_start_if_not_started_xa(trx, true)
      → trx_start_low                       ★分配 trx->id、rseg，状态 → ACTIVE
  → (RR 且首次一致性读) trx_assign_read_view
      → MVCC::view_open                     ★创建 / 复用 read view
```

**阶段二：执行 DML**

```
row_ins_step / row_upd_step / row_del_step
  → 写 undo（段状态 TRX_UNDO_ACTIVE）+ 写 redo（MTR）
  → 加锁（隐式锁或显式锁，挂 trx->lock.trx_locks）
```

**阶段三：提交（server 层 2PC + BGC）**

```
ha_commit_trans                                      server 层 2PC 入口
  → tc_log->prepare                                  阶段一（仅 rw_ha_count > 1）
      → ha_prepare_low
          → innobase_xa_prepare
              → trx_prepare_for_mysql → trx_prepare → trx_prepare_low
                  ★undo 段 ACTIVE → PREPARED + redo
                   （HA_IGNORE_DURABILITY：此处不 fsync）
  → tc_log->commit                                   阶段二
      → MYSQL_BIN_LOG::commit
          → ordered_commit                           BGC 三阶段（详见 binlog.md）
              ├─ flush: ha_flush_logs → innobase_flush_logs
              │            → log_buffer_flush_to_disk  ★redo 批量 fsync
              ├─ sync:  fsync(binlog)                 ★提交点
              └─ commit: ha_commit_low → innobase_commit
                  → innobase_commit_low
                      → trx_commit_for_mysql
                          → trx_commit_low
                              ├─ trx_write_serialisation_history  ★文件世界提交
                              │   ├─ insert_undo → TO_FREE / CACHED
                              │   └─ update_undo → 分配 trx->no、入 purge
                              │       queue、写 GTID → TO_PURGE
                              └─ trx_commit_in_memory             ★内存世界提交
                                  └─ trx_release_impl_and_expl_locks
                                      ├─ 摘 rw_trx_ids / rw_trx_list
                                      ├─ state → COMMITTED_IN_MEMORY
                                      │   （Trx_shard latch 内原子完成）
                                      ├─ GTID 落盘 + 摘 serialisation_list
                                      └─ lock_trx_release_locks    ★2PL 放锁
```

**阶段四：收尾**

```
trx->state → NOT_STARTED（对象留给下次复用）
read view: view_close(own_mutex=true) → 归还 m_free 池
```

分支：回滚走 `ha_rollback_trans` → `innobase_rollback` → `trx_rollback_for_mysql`（见"回滚机制"）。

---

### 事务在 server/InnoDB 的传递

#### 传递链条

server 层的 `THD` 与 InnoDB 的 `trx_t` 通过 handler 的 `ha_data` 槽位关联：

```
THD
  └─ ha_data[innodb_hton->slot]        server 层每个引擎一个槽位
      └─ innodb_session_t*             InnoDB 私有会话结构
          └─ m_trx  →  trx_t           InnoDB 事务对象
```

这个链条由三个小函数串起来，它们短小但关键：

```cpp
[[nodiscard]] innodb_session_t *&thd_to_innodb_session(THD *thd) {
  // 取出 InnoDB 引擎在 THD 中的私有槽位并强转
  // ⇒ 这就是 server↔引擎的边界：server 只知道 void*，由 InnoDB 自己解释
  innodb_session_t *&innodb_session =
      *(innodb_session_t **)thd_ha_data(thd, innodb_hton_ptr);

  // ★ 返回值是指针的引用（*&）⇒ 调用方可直接给它赋值（thd_to_trx(thd) = new_trx）
  //   下面 check_trx_exists 的 trx_t *&trx = thd_to_trx(thd); trx = ... 就依赖这个特性
  if (innodb_session != nullptr) {
    return (innodb_session);
  }

  // 惰性创建 ⇒ 从未碰过 InnoDB 的连接，连 innodb_session_t 都不会分配
  innodb_session = ut::new_withkey<innodb_session_t>(UT_NEW_THIS_FILE_PSI_KEY);
  return (innodb_session);
}
```

```cpp
[[nodiscard]] trx_t *&thd_to_trx(THD *thd) {
  innodb_session_t *&innodb_session = thd_to_innodb_session(thd);
  ut_ad(innodb_session != nullptr);

  // 同样返回引用，直接暴露 m_trx 成员
  // ⇒ 全代码库 20 余处 thd_to_trx 调用，是获取事务的标准方式
  return (innodb_session->m_trx);
}
```

#### check_trx_exists：每个 DML 的必经之门

```cpp
trx_t *check_trx_exists(THD *thd) {
  trx_t *&trx = thd_to_trx(thd);   // 取引用；为空说明这是该连接第一次碰 InnoDB

  ut_ad(EQ_CURRENT_THD(thd));

  if (trx == nullptr) {
    trx = innobase_trx_allocate(thd);

    /* User trx can be forced to rollback,
    so we unset the disable flag. */
    // ★ 新分配的 trx 默认带 TRX_FORCE_ROLLBACK_DISABLE（在 trx_allocate_for_background 中设置）
    //   这里把它清掉——因为用户事务必须允许被强制回滚（高优先级事务的异步 kill 依赖它）
    //   ⇒ "新对象默认不可异步回滚" 与 "用户事务必须可回滚" 这对矛盾的解法
    ut_ad(trx->in_innodb & TRX_FORCE_ROLLBACK_DISABLE);
    trx->in_innodb &= TRX_FORCE_ROLLBACK_MASK;
  } else {
    ut_a(trx->magic_n == TRX_MAGIC_N);   // 校验 magic number 防内存踩踏

    innobase_trx_init(thd, trx);         // 重新同步 THD 上的会话级设置
  }

  return (trx);
}
```

```cpp
static void innobase_trx_init(THD *thd, trx_t *trx) {
  trx->check_foreigns = !thd_test_options(thd, OPTION_NO_FOREIGN_KEY_CHECKS);
  trx->check_unique_secondary =
      !thd_test_options(thd, OPTION_RELAXED_UNIQUE_CHECKS);
}
```

每次 DML 都重新同步外键/唯一性检查开关——因为用户可能在语句之间改了 `SET foreign_key_checks=0`。

#### 模块闭环

- **上游**：server 层 `ha_innobase::write_row` / `update_row` 等 handler 接口 → `check_trx_exists` → `trx_start_if_not_started_xa`。
- **下游**：`trx_start_low` 分配 id 与 rseg → 执行期用 `trx->id` 写 `DB_TRX_ID` → 提交/回滚收尾。
- **边界**：`THD` 与 `trx_t` 的绑定是"每连接一个"，**XA 事务 detach 时会被替换**（`innodb_replace_trx_in_thd`），这是唯一的例外。

#### 语句级与事务级双 scope（Transaction_ctx）

传递链条之外，server 层事务上下文还有一个关键设计：**每个 THD 的事务上下文是"语句级 + 事务级"两份**（`Transaction_ctx::m_scope_info[2]`）：

```cpp
class Transaction_ctx {
 public:
  enum enum_trx_scope { STMT = 0, SESSION };

  struct THD_TRANS {
    bool m_no_2pc;
    int m_rw_ha_count;
    Ha_trx_info *m_ha_list;              // 本 scope 已注册的引擎链表
    unsigned int m_unsafe_rollback_flags;
  };

 private:
  THD_TRANS m_scope_info[2];             // [STMT] + [SESSION]
```

- `m_scope_info[STMT]`：**当前语句**涉及的引擎与状态，语句结束即清空；
- `m_scope_info[SESSION]`：**整个用户事务**涉及的引擎与状态。

**驱动入口 `ha_commit_trans(thd, all)`**：`all=true`（显式 COMMIT / DDL 隐式提交）走 SESSION scope；`all=false`（每条语句结束）走 STMT scope。

**关键判定 `is_real_trans`**：

```cpp
bool is_real_trans = all || !trn_ctx->is_active(Transaction_ctx::SESSION);
```

- 多语句事务内（SESSION 活跃）的语句收尾：`is_real_trans = false`——只释放语句资源，**不真提交**；
- autocommit 语句（SESSION 空）：`is_real_trans = true`——语句结束 = 事务结束，**真提交**。

同一条 `ha_commit_trans(thd, false)` 代码路径，靠这一个布尔值区分"语句收尾"和"autocommit 真提交"。InnoDB 侧对应的判定是 `innobase_commit` 的 `will_commit = commit_trx || !(OPTION_NOT_AUTOCOMMIT|OPTION_BEGIN)`。

**为什么必须双 scope**（三个理由）：

1. **语句错误恢复**：`BEGIN; UPDATE A; UPDATE B（失败）`——B 失败时只调 `ha_rollback_trans(thd, all=false)`。单 scope 要么整事务回滚（毁掉 A），要么不回滚（留下 B 的半截修改）。双 scope 下 InnoDB 用 `last_sql_stat_start` 只回退本语句的 undo 区间（见「SAVEPOINT 与语句级回滚」）。
2. **autocommit 的"语句即事务"**：autocommit=1 时每条语句结束都要真提交，但提交路径只有一条（语句结束回调）。`is_real_trans` 让同一条路径在两种语义间切换。
3. **不可安全回滚操作的传播**：STMT 级的 `m_unsafe_rollback_flags`（改了非事务表 / 建删临时表）在语句末 `merge_unsafe_rollback_flags()` 并入 SESSION 级，保证"任一语句改了非事务表 ⇒ 整个事务都不能安全回滚"的告警语义。

> ★ 版本纠正：`handlerton::start_stmt` 回调在 8.0 已不存在。InnoDB 感知语句边界的职能由 `ha_innobase::external_lock()` 的 `TrxInInnoDB::begin_stmt/end_stmt` 承担（LOCK TABLES 场景由 `ha_innobase::start_stmt()` 补位）。

---

### 事务的分配与启动

#### 分配：trx_allocate_for_mysql

```cpp
trx_t *trx_allocate_for_mysql(void) {
  trx_t *trx;

  trx = trx_allocate_for_background();   // 后台/用户事务共用分配逻辑

  // 在 trx_sys->mutex 保护下挂到 mysql_trx_list 链表头 ⇒ 保证遍历者看到一致视图
  // ⇒ 该链表让 SHOW ENGINE INNODB STATUS / lock_print_info_all_transactions 能遍历所有事务
  //   注意它与 rw_trx_list 不同：前者包含所有（含只读）事务，后者只有读写事务
  trx_sys_mutex_enter();

  ut_d(trx->in_mysql_trx_list = true);
  UT_LIST_ADD_FIRST(trx_sys->mysql_trx_list, trx);

  trx_sys_mutex_exit();

  return (trx);
}
```

#### 启动：trx_start_low

这是事务真正"活过来"的函数。按段剖析，注释就近标在代码行旁：

**第一段：前置断言与版本递增**

```cpp
static void trx_start_low(trx_t *trx, bool read_write) {
  ut_ad(!trx->in_rollback);
  ut_ad(!trx->is_recovered);
  ut_ad(trx->start_line != 0);
  ...
  ut_ad(trx_state_eq(trx, TRX_STATE_NOT_STARTED));
  ut_ad(UT_LIST_GET_LEN(trx->lock.trx_locks) == 0);
  ...
  // ★ 异步回滚机制用它判断"事务是否已经结束"（version 变化说明走完一轮生命周期）
  ++trx->version;
```

**第二段：判定 auto_commit 与 read_only**

```cpp
  // auto_commit 与 read_only 均从 THD 反查 ⇒ server 语义向引擎的传递
  trx->auto_commit = (trx->api_trx && trx->api_auto_commit) ||
                     thd_trx_is_auto_commit(trx->mysql_thd);

  trx->read_only = (trx->api_trx && !trx->read_write) ||
                   (!trx->internal && thd_trx_is_read_only(trx->mysql_thd)) ||
                   srv_read_only_mode;

  if (!trx->auto_commit) {
    ++trx->will_lock;                  // 非 autocommit 事务预期会加锁
  } else if (trx->will_lock == 0) {
    // ★ AC-NL-RO 的判定入口：autocommit 且不需要加锁 → 强制 read_only
    //   一条 autocommit 的只读 SELECT 就是在这里被标记为只读的
    trx->read_only = true;
  }
  trx->persists_gtid = false;
```

**第三段：trx->no 的哨兵值**

```cpp
  if (trx->mysql_thd != nullptr) {
    trx->start_time.store(thd_start_time(trx->mysql_thd),
                          std::memory_order_relaxed);
    ...
  }

  /* The initial value for trx->no: TRX_ID_MAX is used in
  read_view_open_now */
  // ★ MVCC 的基石：运行期取最大值 ⇒ 任何 read view 都认为它"还没提交"
  //   这也是 SHOW ENGINE INNODB STATUS 里未提交事务的 trx->no 显示为一堆 F 的原因
  trx->no = TRX_ID_MAX;
```

**第四段：读写事务——分配 rseg、trx_id、入链表**

```cpp
  if (!trx->read_only &&
      (trx->mysql_thd == nullptr || read_write || trx->ddl_operation)) {
    // 分配持久化回滚段（m_redo.rseg）；临时表的 rseg（m_noredo.rseg）是另一条路径，
    // 只在真正改临时表时才分配——又一处惰性
    trx_assign_rseg_durable(trx);

    trx_sys_mutex_enter();

    trx->id = trx_sys_allocate_trx_id();      // ★ 全局单调递增；这是 trx->id 唯一的分配点

    trx_sys->rw_trx_ids.push_back(trx->id);   // 加入活跃事务 id 集合，供 read view 创建时快照

    trx_add_to_rw_trx_list(trx);              // 加入读写事务链表（purge 扫描用）

    // 状态迁移在 trx_sys->mutex 内完成 ⇒ lock_print_info_all_transactions() 需要看到一致视图
    trx->state.store(TRX_STATE_ACTIVE, std::memory_order_relaxed);

    trx_sys_mutex_exit();

    // 8.0 分片注册：把 trx 加入对应 Trx_shard 的 active_rw_trxs
    // ⇒ 在 mutex 外调用，因为分片有自己的锁
    trx_sys_rw_trx_add(trx);

  } else {
```

**第五段：只读事务——AC-NL-RO 的三种分支**

```cpp
    trx->id = 0;   // 只读事务默认不分配 trx id

    if (!trx_is_autocommit_non_locking(trx)) {
      /* If this is a read-only transaction that is writing
      to a temporary table then it needs a transaction id
      to write to the temporary table. */

      // ★ 最容易忽略的分支：只读事务写临时表也要分配 trx->id
      //   ⇒ 因为临时表的修改也要在 undo 里记录、也要走 MVCC
      if (read_write) {
        trx_sys_mutex_enter();

        trx->state.store(TRX_STATE_ACTIVE, std::memory_order_relaxed);

        trx->id = trx_sys_allocate_trx_id();

        trx_sys->rw_trx_ids.push_back(trx->id);

        trx_sys_mutex_exit();

        trx_sys_rw_trx_add(trx);

      } else {
        trx->state.store(TRX_STATE_ACTIVE, std::memory_order_relaxed);   // 显式只读事务
      }
    } else {
      ut_ad(!read_write);
      // ★ 纯粹的 AC-NL-RO：trx->id = 0、不入任何链表、提交时走简化路径
      trx->state.store(TRX_STATE_ACTIVE, std::memory_order_relaxed);
    }
```

**三个分支对比**（区分的关键是"是否要写临时表"）：

| 分支 | 条件 | `trx->id` | 是否进 rw_trx_list |
|------|------|-----------|-------------------|
| 显式只读事务（如 `START TRANSACTION READ ONLY`） | `!trx_is_autocommit_non_locking` 且 `!read_write` | 0 | 否 |
| 只读但写临时表 | `!AC-NL-RO` 且 `read_write` | **分配** | 是 |
| AC-NL-RO | `trx_is_autocommit_non_locking` | 0 | 否 |

---

### 事务状态机

```cpp
enum trx_state_t {
  TRX_STATE_NOT_STARTED,        // 未开始（新建或已完成）
  TRX_STATE_FORCED_ROLLBACK,    // 同 NOT_STARTED，但语义是"上次活跃时被异步回滚"
  TRX_STATE_ACTIVE,             // 活跃（执行 DML 中）
  TRX_STATE_PREPARED,           // 2PC/XA 已 prepare
  TRX_STATE_COMMITTED_IN_MEMORY // 内存中已提交
};
```

状态转换：

```
NOT_STARTED ──trx_start_low──> ACTIVE
ACTIVE ──trx_prepare（2PC）──> PREPARED
PREPARED ──commit 裁决──> COMMITTED_IN_MEMORY
ACTIVE ──trx_commit_in_memory──> COMMITTED_IN_MEMORY
COMMITTED_IN_MEMORY ──清理──> NOT_STARTED
ACTIVE ──异步回滚──> FORCED_ROLLBACK ──> NOT_STARTED
```

#### 变换时机总表

| 转换 | 触发函数 | 执行线程 | 保护的锁 | 同步的 undo 段状态 |
|------|---------|---------|---------|-------------------|
| NOT_STARTED → ACTIVE | `trx_start_low` / `trx_start_if_not_started_xa` | 用户线程 | `trx_sys->mutex` | —（此时还没写 undo） |
| ACTIVE 期间写 undo | DML 执行（`row_ins_step` / `row_upd_step`） | 用户线程 | `trx->undo_mutex` | 段状态 = `TRX_UNDO_ACTIVE` |
| ACTIVE → PREPARED | `trx_prepare`（2PC 阶段一） | 用户线程 | `trx_sys->mutex` + rseg latch | `TRX_UNDO_ACTIVE` → `TRX_UNDO_PREPARED` |
| PREPARED → PREPARED_IN_TC | `trx_undo_set_prepared_in_tc`（8.0.29+） | 用户线程 | rseg latch | → `TRX_UNDO_PREPARED_IN_TC` |
| ACTIVE/PREPARED → COMMITTED_IN_MEMORY | `trx_release_impl_and_expl_locks` | 用户线程（BGC commit stage） | `Trx_shard` latch + `trx->mutex` | insert → `TO_FREE`/`CACHED`；update → `TO_PURGE` |
| ACTIVE → FORCED_ROLLBACK | `trx_rollback_for_mysql` | **killer 线程**（异步） | `trx->mutex` | 逐行 undo 后段被清理 |
| 任意 → NOT_STARTED | `trx_commit_in_memory` / 回滚收尾 | 用户或 killer | `trx->mutex` | 段已归还或缓存 |

关键点：

- **`TRX_STATE_FORCED_ROLLBACK` 是标记而非长驻态**：与 `NOT_STARTED` 行为完全相同，只是语义上标记"上次是被别人强制回滚的"。`innobase_rollback` 里显式处理它并转回 `NOT_STARTED`。
- **`COMMITTED_IN_MEMORY` 是"对外可见"的那一刻**：状态一变，其他事务的 read view 就能看到这个事务的修改了。它**不保证持久性**（持久性由 redo + binlog 决定）。
- **状态字段是 `std::atomic`**：`trx->state` 用 `memory_order_relaxed` 读写。多线程（异步 killer、purge、监控线程）并发访问安全。

#### 与 undo 段状态的协同

`trx->state` 是**内存**的；`trx_undo_t::state` 是**持久化**的（写入 undo segment header 的 `TRX_UNDO_STATE`，`MLOG_2BYTES`）——**崩溃恢复判定 trx 命运的唯一依据是后者**。

```cpp
// trx0undo.h:309
/* States of an undo log segment */
/** contains an undo log of an active  transaction */
constexpr uint32_t TRX_UNDO_ACTIVE = 1;
/** cached for quick reuse */
constexpr uint32_t TRX_UNDO_CACHED = 2;
/** insert undo segment can be freed */
constexpr uint32_t TRX_UNDO_TO_FREE = 3;
/** update undo segment will not be reused: it can be freed in purge when all
 undo data in it is removed */
constexpr uint32_t TRX_UNDO_TO_PURGE = 4;
/** contains an undo log of an prepared transaction for a server version older
 * than 8.0.29 */
constexpr uint32_t TRX_UNDO_PREPARED_80028 = 5;
/** contains an undo log of an prepared transaction */
constexpr uint32_t TRX_UNDO_PREPARED = 6;
/* contains an undo log of a prepared transaction that has been processed by the
 * transaction coordinator */
constexpr uint32_t TRX_UNDO_PREPARED_IN_TC = 7;
```

**两者协同关系**：

| `trx->state` | `undo->state` | 说明 |
|---|---|---|
| ACTIVE | `TRX_UNDO_ACTIVE`(1) | 正常读写中 |
| PREPARED | `TRX_UNDO_PREPARED`(6) / `_80028`(5) / `_IN_TC`(7) | XA PREPARE，XID 写入 undo header |
| COMMITTED_IN_MEMORY | `CACHED`(2) / `TO_FREE`(3) / `TO_PURGE`(4) | 由 `trx_undo_set_state_at_finish()` 三选一 |
| 崩溃恢复 | `PREPARED*`→PREPARED；`ACTIVE`→ACTIVE；其余→COMMITTED_IN_MEMORY | `trx_resurrect_*()` |

**提交时的状态迁移**（`trx_undo_set_state_at_finish()`，`trx0undo.cc:1806`）：

```cpp
page_t *trx_undo_set_state_at_finish(trx_undo_t *undo, mtr_t *mtr) {
  undo_page = trx_undo_page_get(page_id_t(undo->space, undo->hdr_page_no),
                                undo->page_size, mtr);
  seg_hdr = undo_page + TRX_UNDO_SEG_HDR;
  page_hdr = undo_page + TRX_UNDO_PAGE_HDR;

  // undo page list的复用
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
                             TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;      // ★① 只占 1 页 且 已用 < 3/4 页 → 挂回 rseg cache 复用
  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;     // ★② insert undo：commit 后数据已无用，段可直接释放
  } else {
    state = TRX_UNDO_TO_PURGE;    // ★③ update undo：必须等 purge 清完 history list
  }

  undo->state = state;
  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, state, MLOG_2BYTES, mtr);  // ★★ 持久化
  return (undo_page);
}
```

**XA PREPARE / ROLLBACK 的状态迁移**（`trx_undo_set_state_at_prepare()`，`trx0undo.cc:1843`）：

```cpp
  if (rollback) {
    ut_ad(undo->is_prepared());
    /* Write GTID information if there. */
    trx_undo_gtid_write(trx, undo_header, undo, mtr, !rollback);

    mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, TRX_UNDO_ACTIVE, MLOG_2BYTES, mtr);
    //  ★★ XA ROLLBACK：PREPARED → ACTIVE
    //     目的就是"如果此刻宕机，恢复时按 ACTIVE 走回滚"
    return (undo_page);
  }

  ut_ad(undo->state == TRX_UNDO_ACTIVE);
  undo->set_prepared(trx->xid);   // ★ 内存态 → TRX_UNDO_PREPARED(6)

  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, undo->state, MLOG_2BYTES, mtr);
  mlog_write_ulint(undo_header + TRX_UNDO_FLAGS, undo->flag, MLOG_1BYTE, mtr);
  trx_undo_write_xid(undo_header, &undo->xid, mtr);   // ★★ XID 落盘
```

> ★★ **XA ROLLBACK 先把 undo 改回 ACTIVE 再回滚**，这是一个"用持久化状态为回滚上保险"的技巧：万一回滚中途宕机，重启后 undo 仍是 ACTIVE，恢复流程会继续回滚。
>
> ★ `TRX_UNDO_PREPARED_IN_TC`(7)（`trx_undo_set_prepared_in_tc()`，`trx0undo.cc:1892`）表示 **TC（binlog）已接管**，此后可走"一阶段提交"——这是 8.0.29+ 的 XA 优化。

**InnoDB 侧 `trx->state` 全部 23 个赋值点**里，与 undo 相关的关键几处：

```cpp
// trx0trx.cc:1949 —— trx_release_impl_and_expl_locks() 内
  state_transition = [&](trx_state_t new_state) { ... trx->state.store(new_state, ...); };
  ... → COMMITTED_IN_MEMORY   // ★ 隐式锁释放的那一刻

// trx0trx.cc:2179 —— trx_commit_in_memory() 尾部，★ 唯一产生 FORCED_ROLLBACK 的地方
  if (trx->abort) {
    trx->abort = false;
    trx->state.store(TRX_STATE_FORCED_ROLLBACK, std::memory_order_relaxed);
  } else {
    trx->state.store(TRX_STATE_NOT_STARTED, std::memory_order_relaxed);
  }

// trx0trx.cc:3071 —— trx_prepare()（受 trx_sys->mutex 保护）
  trx->state.store(TRX_STATE_PREPARED, ...);
```

> ⚠️ 所有这些点都用 `std::memory_order_relaxed`——正确性靠 `trx->mutex` / `trx_sys->mutex` 与"只有属主线程能改"的约定保证，**不靠原子序**。

`trx->state` 是**内存态**，`undo segment state` 是**文件态**（持久化在 undo 页头，随 redo 落盘）。两者必须同步推进——内存态决定并发行为，文件态决定崩溃恢复行为。

undo 段状态（`trx0undo.h`）：

| 常量 | 值 | 含义 |
|------|-----|------|
| `TRX_UNDO_ACTIVE` | 1 | 活跃事务的 undo |
| `TRX_UNDO_CACHED` | 2 | 缓存待快速复用 |
| `TRX_UNDO_TO_FREE` | 3 | insert undo，可立即释放 |
| `TRX_UNDO_TO_PURGE` | 4 | update undo，purge 清理后释放 |
| `TRX_UNDO_PREPARED_80028` | 5 | 8.0.29 之前格式，仅为升级兼容 |
| `TRX_UNDO_PREPARED` | 6 | 已 prepare |
| `TRX_UNDO_PREPARED_IN_TC` | 7 | 已被事务协调器（TC）处理过 |

**协同的核心原则：文件态先行，内存态后到。**

- **prepare 时**：`trx_prepare_low` 先把 undo 段状态改成 `TRX_UNDO_PREPARED` 并通过 MTR 写 redo，**然后** `trx_prepare` 才把 `trx->state` 改成 `TRX_STATE_PREPARED`。**顺序不能反**——若内存先变 PREPARED 而文件还是 ACTIVE，此时崩溃，内存丢失、文件是 ACTIVE，恢复时会当作未提交事务回滚，与 binlog 的裁决矛盾。
- **commit 时**：`trx_write_serialisation_history` 先把 undo 段改好并 `mtr_commit` 落 redo，**然后** `trx_release_impl_and_expl_locks` 才把 `trx->state` 改成 `TRX_STATE_COMMITTED_IN_MEMORY`。

这正是 `trx_commit_low` 注释的意思——"the transaction becomes 'durable' when we write the log to disk, but in the logical sense the commit in the file-based data structures (undo logs etc.) happens here"。

**为什么 insert 与 update 的终态不同**：insert undo 记录的是"插入的行"，事务提交后没有任何 read view 需要看到"这行不存在"，所以可以立即释放（`TO_FREE`/`CACHED`）。update undo 记录的是"被修改的旧版本"，可能还有老的 read view 需要回溯，必须挂进 history list 等 purge（`TO_PURGE`）。这也是前面说的"纯 INSERT 事务不分配 `trx->no`"的根因——它不进 purge 队列。

**`PREPARED` 与 `PREPARED_IN_TC` 的区别**（8.0.29 引入）：后者表示该 prepared 事务的 XID 已被事务协调器（binlog）写入。崩溃恢复时靠这个标记判断"binlog 是否已经写过这个事务"，避免重复写或漏写。旧的 `PREPARED_80028`(5) 只为兼容升级路径保留。

#### XA 状态机：XID_STATE 的另一套状态

⚠️ **勘误**：**不存在 `enum enum_xa_state_flags`**（`sql/xa.h` 与全仓 0 命中）。`XID_STATE` 里承担"标志位"职责的是 `m_is_detached` 与 `m_is_binlogged` 两个独立 bool。

```cpp
// sql/xa.h:295
class XID_STATE {
 public:
  enum xa_states {
    XA_NOTR = 0,          // ★ 无 XA 事务（普通事务也停在这里）
    XA_ACTIVE,            // ★ XA START 后、XA END 前
    XA_IDLE,              // ★ XA END 后（已 detach，但未 PREPARE）
    XA_PREPARED,          // ★ XA PREPARE 后
    XA_ROLLBACK_ONLY      // ★ RM 单边回滚过，只能 XA ROLLBACK
  };

 private:
  static const char *xa_state_names[];

  XID m_xid;
  /**
    This mutex used for eliminating a possibility to run two
    XA COMMIT/XA ROLLBACK statements concurrently against the same xid value.
    m_xa_lock is used on handling XA COMMIT/XA ROLLBACK and acquired only for
    external XA branches. */
  std::mutex m_xa_lock;

  /// Used by external XA only
  xa_states xa_state;
  bool m_is_detached = false;
  uint rm_error;              // RM 报告给 TM 的错误
  bool m_is_binlogged;        // ★ XA-prepare 的 binlog 二次记录标记
  ...
  void start_normal_xa(const XID *xid) {
    assert(m_xid.is_null());
    xa_state = XA_ACTIVE;
    m_xid.set(xid);
    m_is_detached = false;
    rm_error = 0;
  }

  void start_detached_xa(const XID *xid, bool binlogged_arg = false) {
    xa_state = XA_PREPARED;   // ★★ 恢复出来的 XA 直接落 PREPARED，跳过 ACTIVE/IDLE
    m_xid.set(xid);
    m_is_detached = true;
    rm_error = 0;
    m_is_binlogged = binlogged_arg;
  }
};

// sql/xa.cc:86
const char *XID_STATE::xa_state_names[] = {"NON-EXISTING", "ACTIVE", "IDLE",
                                           "PREPARED", "ROLLBACK ONLY"};
```

**★ 两套状态完全独立，不可混为一谈**：

| Server `xa_state` | InnoDB `trx->state` | InnoDB `undo->state` |
|---|---|---|
| `XA_NOTR` | `NOT_STARTED` / `ACTIVE` | 无 / `ACTIVE` |
| `XA_ACTIVE` | `ACTIVE` | `ACTIVE` |
| `XA_IDLE` | **`ACTIVE`**（不是 PREPARED！） | `ACTIVE` |
| `XA_PREPARED` | `PREPARED` | `PREPARED`(6) 或 `PREPARED_IN_TC`(7) |
| `XA_ROLLBACK_ONLY` | `PREPARED` 或已 `NOT_STARTED` | `PREPARED` → `ACTIVE` |

> ★★ **`XA END` 只改 Server 状态（→`XA_IDLE`），完全不触碰 InnoDB**。只有 `XA PREPARE` 才通过 `trx_prepare()`（`trx0trx.cc:3071`）把 `trx->state` 推到 `PREPARED`。所以"XA END 之后 InnoDB 还以为是 ACTIVE"是正常状态，不是 bug。

**`trx_disconnect_prepared()`：XA 断开但保留 trx 对象**（`trx0trx.cc:666`）：

```cpp
inline void trx_disconnect_from_mysql(trx_t *trx, bool prepared) {
  trx_sys_mutex_enter();
  UT_LIST_REMOVE(trx_sys->mysql_trx_list, trx);

  if (trx->read_view != nullptr) {
    trx_sys->mvcc->view_close(trx->read_view, true);
  }

  if (prepared) {
    ut_ad(trx_state_eq(trx, TRX_STATE_PREPARED));
    /* During disconnection there is a short period when the server layer
    already believes this XID is detached from this connection, and thus another
    connection may try to XA COMMIT/ROLLBACK it, yet InnoDB is still processing
    the trx object ... We use trx->mysql_thd field, under protection of
    trx_sys->mutex to synchronize with trx_get_trx_by_xid() ... */
    trx->mysql_thd = nullptr;    // ★★ 断开连接但 trx 对象留在 rw_trx_list
  }
  trx_sys_mutex_exit();
}
```

> ★ 这就是"一个会话 PREPARE 后断开，另一个会话可以用同一个 XID 接管提交"的实现基础。

> 注意：InnoDB 的 `trx_state_t` 之外，**server 层还有一套 XA 协议状态机** `XID_STATE::xa_states`（xa.h:297）。两套并存、各管各的，容易混淆。

```cpp
enum xa_states {
  XA_NOTR = 0,        // 没有 XA 事务
  XA_ACTIVE,          // XA START 之后、XA END 之前
  XA_IDLE,            // XA END 之后、XA PREPARE/COMMIT 之前
  XA_PREPARED,        // 已 prepare
  XA_ROLLBACK_ONLY    // 出错时标记只回滚
};
```

**转换由 XA 命令驱动**（对应 `sql/xa/sql_xa_*.cc` 的 `set_state` 调用）：

```
XA_NOTR ──XA START──> ACTIVE ──XA END──> IDLE ──XA PREPARE──> PREPARED
   ▲                    │                   │                     │
   │                    │ XA ROLLBACK       │ XA COMMIT ONE PHASE │ XA COMMIT / ROLLBACK
   │                    └───────────────────┴─────────────────────┘
   └─────────────────────────────────────────────────────────────
IDLE ──XA START (RESUME)──> ACTIVE          （同一事务继续执行）
```

**与 InnoDB `trx_state_t` 的两层映射**——关键洞察是**两层不同步**：

| 时刻 | `XID_STATE` | `trx->state` | 说明 |
|------|-----------|-------------|------|
| XA START 后 | ACTIVE | NOT_STARTED | 引擎惰性，还没碰数据 |
| 执行 DML | ACTIVE | ACTIVE | 两层都活跃 |
| XA END 后 | IDLE | ACTIVE（不变） | **引擎事务仍活着** |
| XA PREPARE 后 | PREPARED | PREPARED | ★ 唯一两层一致的态 |
| XA COMMIT 后 | NOTR | NOT_STARTED | 都结束 |

XA_ACTIVE 期间引擎可能还没启动事务（惰性启动）；XA_IDLE 期间引擎事务仍是 ACTIVE（还没 prepare）。**只有 PREPARED 状态两层语义完全对齐**——这正是 detach 以 PREPARED 为界的深层原因。

还有一个微妙的中间态（`ha_commit_trans` 里）：

```cpp
if (!error && all && xid_state->has_state(XID_STATE::XA_IDLE)) {
  ...
  xid_state->set_state(XID_STATE::XA_PREPARED);   // ONE PHASE 的中间态
}
```

`XA COMMIT ONE PHASE` 时先**短暂置 PREPARED**（让 binlog 写 `XA_prepare_log_event`），commit 完成后立刻回 NOTR——XA_IDLE 直通 NOTR 的路径中间借道 PREPARED。

---

### read view 的生命周期

> **边界**：本篇只讲 read view **何时创建、何时销毁、如何复用**（站在事务生命周期视角）。可见性判断算法、undo 版本链回溯、semi-consistent read 见 [`mvcc.md`](mvcc.md)。

#### 开启入口：trx_assign_read_view

```cpp
ReadView *trx_assign_read_view(trx_t *trx) {
  ut_ad(trx_can_be_handled_by_current_thread_or_is_hp_victim(trx));
  ut_ad(trx->state.load(std::memory_order_relaxed) == TRX_STATE_ACTIVE);  // view 只能挂在活跃事务上

  if (srv_read_only_mode) {
    ut_ad(trx->read_view == nullptr);
    return (nullptr);                    // 只读模式不需要 MVCC 快照

  } else if (!MVCC::is_view_active(trx->read_view)) {
    // ★ 已有关联 view 就不重复创建——这是复用的第一层
    //   ⇒ 同一事务内多条语句（RR 下）共用一个 view
    trx_sys->mvcc->view_open(trx->read_view, trx);
  }

  return (trx->read_view);
}
```

**何时被调用**取决于隔离级别：

- **REPEATABLE READ**：事务内第一条一致性读时创建，之后整个事务复用。`innobase_start_trx_and_assign_read_view` 对 RR 显式调用它。
- **READ COMMITTED**：**每条语句**重新创建，语句结束即关闭——这就是 RC 能看到其他事务新提交修改的原因。

#### view_open：两条路径与指针 LSB 技巧

⚠️ **勘误**：
- **`ReadView` 在 `include/read0types.h:56`**，不在 `read0read.h`（后者是 `MVCC`）；**不存在 `storage/innobase/mvcc/` 目录**
- **不存在 `ReadView::view_open()`**（真名 `MVCC::view_open()`）、**不存在 `complete()` / `copy()` / `m_cloned`**
- **不存在 `ReadView::complete()`**——被 `copy_prepare()` + `copy_complete()` 取代
- **不存在 `m_free_views`**（真名 `m_free`）

**★ `MVCC::view_open()` 全文**（`read0read.cc:527`）——LSB 技巧第一处：

```cpp
void MVCC::view_open(ReadView *&view, trx_t *trx) {
  ut_ad(!srv_read_only_mode);

  /** If no new RW transaction has been started since the last view
  was created then reuse the the existing view. */
  if (view != nullptr) {
    // 利用指针最低位（LSB）作为标志位（closed 标记）
    uintptr_t p = reinterpret_cast<uintptr_t>(view);
    view = reinterpret_cast<ReadView *>(p & ~1);
    //  ★★★ LSB 技巧：ReadView 对象按 8/16 字节对齐 ⇒ 指针 LSB 恒为 0，可借用当标志位
    //       LSB=1 表示"这个 view 已关闭但对象还在"，trx->read_view 是这种 dual representation

    ut_ad(view->m_closed);

    /* NOTE: This can be optimised further, for now we only
    reuse the view if there are no active RW transactions.

    There is an inherent race here between purge and this
    thread. Purge will skip views that are marked as closed.
    Therefore we must set the low limit id after we reset the
    closed status after the check. ... */

    if (trx_is_autocommit_non_locking(trx) && view->empty()) {
      // 注意这里没持有 trx_sys_t::mutex     ★★ 快路径完全无锁
      view->m_closed = false;

      if (view->m_low_limit_id == trx_sys_get_next_trx_id_or_no()) {
        return;      // ★★★ 快路径命中：自上次关闭以来没新 RW 事务 ⇒ 直接复用，不碰 trx_sys mutex
      } else {
        view->m_closed = true;    // 有新事务了 → 回滚，落到下面的慢路径
      }
    }
  }

  // 慢路径：SCN 优化热点 —— trx_sys_mutex
  trx_sys_mutex_enter();

  if (view != nullptr) {
    UT_LIST_REMOVE(m_views, view);      // 复用旧 view
  } else {
    view = get_view();                  // 从 m_free 取或 new
  }

  if (view != nullptr) {
    view->prepare(trx->id);
    UT_LIST_ADD_FIRST(m_views, view);   // ★ 头插 ⇒ m_views 按 m_low_limit_no 递减（新在前）
    ut_ad(!view->is_closed());
    ut_ad(validate());
  }
  trx_sys_mutex_exit();
}
```

**`ReadView::prepare()`**（`read0read.cc:444`）——`m_ids` 的唯一填充入口：

```cpp
void ReadView::prepare(trx_id_t id) {
  ut_ad(trx_sys_mutex_own());           // ★ 必须在 trx_sys mutex 下

  m_creator_trx_id = id;

  m_low_limit_no = trx_get_serialisation_min_trx_no();
  //  ★ purge 边界 = 当前已串行化最小 trx_no

  m_low_limit_id = trx_sys_get_next_trx_id_or_no();
  //  ★ 高水位 = 下一个将分配的 trx_id（"未来事务"）

  if (!trx_sys->rw_trx_ids.empty()) {
    copy_trx_ids(trx_sys->rw_trx_ids);
  } else {
    m_ids.clear();
  }

  /* The first active transaction has the smallest id. */
  m_up_limit_id = !m_ids.empty() ? m_ids.front() : m_low_limit_id;

  m_closed = false;
}
```

```cpp
// read0read.cc:352 —— copy_trx_ids 用两段 memmove 剔除 creator 自己
void ReadView::copy_trx_ids(const trx_ids_t &trx_ids) {
  ulint size = trx_ids.size();
  if (m_creator_trx_id > 0) { ut_ad(size > 0); --size; }   // ★ 剔除自己
  ...
    trx_ids_t::const_iterator it =
        std::lower_bound(trx_ids.begin(), trx_ids.end(), m_creator_trx_id);
    ::memmove(p, &trx_ids[0], i * sizeof(...));            // 段1：creator 之前
    ::memmove(p + i, &trx_ids[i + 1], n);                  // 段2：creator 之后，前移一格
  ...
  m_up_limit_id = m_ids.front();   // ★ rw_trx_ids 有序 ⇒ front() 即最小活跃 id
}
```

**`view_close` 的三条路径**：

```cpp
// read0read.cc:769 —— 慢/快两条
void MVCC::view_close(ReadView *&view, bool own_mutex) {
  uintptr_t p = reinterpret_cast<uintptr_t>(view);

  if (!own_mutex) {
    /* Sanitise the pointer first. */
    ReadView *ptr = reinterpret_cast<ReadView *>(p & ~1);
    ptr->m_closed = true;                    // ★ 路径 A（AC-NL-RO 快路径）：无锁，只置标记
    view = reinterpret_cast<ReadView *>(p | 0x1);
    //                                  ★★★ LSB 技巧第二处：打 closed 标记
    //      view 仍留在 m_views 链表里等待下次复用，不释放内存
  } else {
    view = reinterpret_cast<ReadView *>(p & ~1);
    view->close();                           // ★ 路径 B（RW trx）：m_creator_trx_id = TRX_ID_MAX
    UT_LIST_REMOVE(m_views, view);
    UT_LIST_ADD_LAST(m_free, view);          // ★★ 真正回收回空闲链表
    view = nullptr;
  }
}
```

```cpp
// read0read.cc:499 —— 第三条：view_release
void MVCC::view_release(ReadView *&view) {
  ut_ad(trx_sys_mutex_own());
  uintptr_t p = reinterpret_cast<uintptr_t>(view);
  ut_a(p & 0x1);                             // ★★ 断言：只回收"已关闭但仍挂链表"的 view
  view = reinterpret_cast<ReadView *>(p & ~1);
  ut_ad(view->m_closed);
  ut_ad(view->m_creator_trx_id == 0);        // ★ 非 RW 事务的 view
  UT_LIST_REMOVE(m_views, view);
  UT_LIST_ADD_LAST(m_free, view);
  view = nullptr;
}
```

**LSB 技巧的"读端"**（`read0read.h:89` / `trx0trx.ic:238`）：

```cpp
  /** @return true if the view is active and valid */
  static bool is_view_active(ReadView *view) {
    ut_a(view != reinterpret_cast<ReadView *>(0x1));
    return (view != nullptr && !(intptr_t(view) & 0x1));   // ★★ LSB=1 ⇒ 已关闭
  }

static inline ReadView *trx_get_read_view(trx_t *trx) {
  return (!MVCC::is_view_active(trx->read_view) ? nullptr : trx->read_view);
}
```

```cpp
// trx0trx.cc:2351
ReadView *trx_assign_read_view(trx_t *trx) {
  if (srv_read_only_mode) {
    ut_ad(trx->read_view == nullptr);
    return (nullptr);
  } else if (!MVCC::is_view_active(trx->read_view)) {
    trx_sys->mvcc->view_open(trx->read_view, trx);   // ★ 同一事务内后续一致读走 else，复用同一个 view
  }
  return (trx->read_view);
}
```

> ⚠️ **`m_closed` 不是 atomic**，就是裸 `bool`（`read0types.h:323`）。快路径 `view->m_closed = false`（`:597`）**完全无锁**——源码注释明确承认与 purge 线程存在 data race，后果是 purge view 可能偏保守（`m_low_limit_no` 偏小，purge 少清理一点），**不会出错**。
>
> ★ LSB 标记本身无 race：它写在 `trx->read_view` 里，而该字段只被拥有该 trx 的线程读写。
>
> ★ purge view 的来源（`clone_oldest_view()`，`read0read.cc:720`）：`copy_prepare()`（**锁内**，含 `m_ids` memmove）→ `copy_complete()`（**锁外**，`m_ids.insert` 可能 realloc）。`copy_complete()` 还会把 `m_creator_trx_id` 重新 `insert` 回 `m_ids` 并置 0——因为 purge view 是"旁观者"，oldest view 的 creator 对它同样不可见。

```cpp
void MVCC::view_open(ReadView *&view, trx_t *trx) {
  /** If no new RW transaction has been started since the last view
  was created then reuse the the existing view. */
  if (view != nullptr) {
    uintptr_t p = reinterpret_cast<uintptr_t>(view);

    view = reinterpret_cast<ReadView *>(p & ~1);

    ut_ad(view->m_closed);

    if (trx_is_autocommit_non_locking(trx) && view->empty()) {
      view->m_closed = false;

      if (view->m_low_limit_id == trx_sys_get_next_trx_id_or_no()) {
        return;
      } else {
        view->m_closed = true;
      }
    }
  }

  trx_sys_mutex_enter();
  ...
  view->prepare(trx->id);
  UT_LIST_ADD_FIRST(m_views, view);
  trx_sys_mutex_exit();
}
```

**第一段——指针 LSB 双关表示（pointer tagging）**：

`p & ~1` 把指针最低位清零得到真实地址；`view_close` 里对应地用 `p | 0x1` 置位。**指针最低位被借用来标记"这个 view 是否已关闭"**。

为什么可行？`ReadView` 按 cacheline 对齐分配，地址最低位恒为 0，可安全借用。这是典型的指针标记技巧：把"关闭状态"和"指针"塞进同一个字，读取时无锁；代价是每次解引用前必须 `& ~1` 清理。

**第二段——AC-NL-RO 的 fast path**：

只对 AC-NL-RO 事务生效，且要求 `view->empty()`（没有活跃事务 id 要记录）。流程：

1. **无锁**地把 `m_closed = false`（"复活"这个 view）。
2. 检查 `m_low_limit_id == trx_sys_get_next_trx_id_or_no()`——即"上次创建 view 以来有没有新事务 id 被分配"。
3. 相等（没有新事务）→ **直接复用，全程不持锁，直接 return**。
4. 不等 → 说明期间有新事务产生，旧 view 过时，`m_closed` 改回 true，退回 slow path（持 `trx_sys->mutex`）。

**收益**：高频的 autocommit 只读查询完全不碰 `trx_sys->mutex`。这是 OLTP 只读场景的关键优化。

**代价——与 purge 线程的固有竞态**：源码注释详细记录了这个 race 及它引发的 bug#117553：

| 时刻 | AC-NL-RO 线程 | purge 线程 |
|------|--------------|-----------|
| T1 | 无锁设 `m_closed = false`（此时 `m_low_limit_id` 仍是旧值） | |
| T4-T7 | | 持 `trx_sys->mutex`，`get_oldest_view()` 看到 `m_closed == false` 而选中该 view，`copy_prepare()` 拷走**过时的** `m_low_limit_id` / `m_low_limit_no` |
| T8 | 发现 id 不匹配，回滚设 `m_closed = true` | |
| T9 | | 用这个过时 view 作为 `purge_sys->view` |

**后果**：purge 的 `m_low_limit_no` 回退变小 → 本该清理的 undo 被保留；更糟时 `changes_visible` 对已 purge 的事务误判为"不可见"（missing_history），代码尝试读取已被 purge 的 undo page → **CRASH**。

注释承认这是 "an inherent race"，并要求"必须在设置 closed 状态之后再设 low limit id"。设计者知情，把风险控制在"最多有些 undo 没被及时清理"——但 bug#117553 表明边界情况仍会出问题。

#### 销毁：view_close 的两条路径

```cpp
void MVCC::view_close(ReadView *&view, bool own_mutex) {
  uintptr_t p = reinterpret_cast<uintptr_t>(view);

  /* Note: The assumption here is that AC-NL-RO transactions will
  call this function with own_mutex == false. */
  if (!own_mutex) {
    ReadView *ptr = reinterpret_cast<ReadView *>(p & ~1);

    ptr->m_closed = true;

    view = reinterpret_cast<ReadView *>(p | 0x1);
  } else {
    view = reinterpret_cast<ReadView *>(p & ~1);

    view->close();

    UT_LIST_REMOVE(m_views, view);
    UT_LIST_ADD_LAST(m_free, view);

    view = nullptr;
  }
}
```

| | `own_mutex == false`（AC-NL-RO） | `own_mutex == true`（常规） |
|---|---|---|
| 动作 | 仅置 `m_closed = true` + 指针 LSB 置 1 | `close()` + 从 `m_views` 摘除 + 加入 `m_free` 池 + `view = nullptr` |
| 是否持 `trx_sys->mutex` | 否（由调用方持有） | 是 |
| 对象去向 | **留在 `trx->read_view` 供下次复用** | 归还 `m_free` 池 |

**关键差异**：AC-NL-RO 的 view **不归还、不摘链**，只是"标记为关闭"并留在事务对象上——这正是 fast path 能复用的前提。代价是这些 view 长期挂在 `m_views` 链表里（closed 状态，`get_oldest_view()` 会跳过）。

**调用时机**：`trx_commit_in_memory` 里 AC-NL-RO 分支与只读分支都调 `trx_sys->mvcc->view_close(trx->read_view, false)`。

#### 三种隔离场景下的生命周期

```
RR 事务:
  trx_start → [首条一致性读] view_open → ... 整个事务复用同一个 view ...
            → [提交/回滚] view_close(own_mutex=true) → 归还 m_free

RC 事务:
  trx_start → [每条语句] view_open → 语句结束 view_close → [下条语句] view_open → ...
            （每条语句一个全新 view，故能看到语句开始前已提交的最新数据）

AC-NL-RO:
  trx_start → view_open(fast path: 无锁复用上次 view) → 查询
            → view_close(own_mutex=false: 仅标记关闭，对象保留待复用)
```

---

### 事务提交

提交分两层：`trx_commit_low`（文件世界）与 `trx_commit_in_memory`（内存世界）。

**入口 `innobase_commit` 的准备动作**：进入两层之前，引擎层先做两件与 binlog 协同的准备（`ha_commit_low` → `innobase_commit`）：

```cpp
/* The following call reads the binary log position of
the transaction being committed. ... */
thd_binlog_pos(thd, &trx->mysql_log_file_name, &pos);

trx->mysql_log_offset = static_cast<uint64_t>(pos);

/* Don't do write + flush right now. For group commit
to work we want to do the flush later. */
trx->flush_log_later = true;
```

- `thd_binlog_pos`：读取当前事务的 **binlog 文件名与位置**，存进 `trx->mysql_log_file_name` / `mysql_log_offset`——供 `mysqlbackup` 使用（它需要"InnoDB 提交位置 ↔ binlog 位置"的映射）。
- `trx->flush_log_later = true`：**redo 的 commit 记录也延迟刷**——与 prepare 的 `HA_IGNORE_DURABILITY` 呼应，把刷盘留到 `trx_commit_complete_for_mysql`。

#### 第一层：trx_commit_low

```cpp
void trx_commit_low(trx_t *trx, mtr_t *mtr) {
  assert_trx_nonlocking_or_in_list(trx);
  ut_ad(!trx_state_eq(trx, TRX_STATE_COMMITTED_IN_MEMORY));
  ut_ad(!mtr || mtr->is_active());
  /* undo_no is non-zero if we're doing the final commit. */
  if (trx->fts_trx != nullptr && trx->undo_no != 0 &&
      trx->lock.que_state != TRX_QUE_ROLLING_BACK) {
    error = fts_commit(trx);
    ...
  }

  bool serialised;

  // ★ mtr != nullptr 表示事务确实修改过数据（trx_commit 中按 trx_is_rseg_updated 决定是否创建 mtr）
  //   ⇒ 没改过数据的事务 mtr == nullptr，跳过整个序列化阶段——空事务快速提交的路径
  if (mtr != nullptr) {
    mtr->set_sync();
    serialised = trx_write_serialisation_history(trx, mtr);

    mtr_commit(mtr);   // ★ "文件世界提交"的分界点

  } else {
    serialised = false;
  }

  // serialised 表示"是否分配了 trx->no 并进入 serialisation_list"
  // ⇒ 只有产生 update undo 的事务才为 true；该布尔值一路传到 trx_release_impl_and_expl_locks，
  //   决定要不要处理 GTID 和 serialisation_list
  trx_commit_in_memory(trx, mtr, serialised);
}
```

#### 第二层：trx_write_serialisation_history（文件世界的核心）

```cpp
static bool trx_write_serialisation_history(trx_t *trx, mtr_t *mtr) {
  /* Change the undo log segment states from TRX_UNDO_ACTIVE to some
  other state: these modifications to the file data structure define
  the transaction as committed in the file based domain, at the
  serialization point of the log sequence number lsn obtained below. */

  bool own_redo_rseg_mutex = false;
  bool own_temp_rseg_mutex = false;

  if (trx->rsegs.m_redo.rseg != nullptr && trx_is_redo_rseg_updated(trx)) {
    trx->rsegs.m_redo.rseg->latch();
    own_redo_rseg_mutex = true;
  }

  mtr_t temp_mtr;

  if (trx->rsegs.m_noredo.rseg != nullptr && trx_is_temp_rseg_updated(trx)) {
    trx->rsegs.m_noredo.rseg->latch();
    own_temp_rseg_mutex = true;
    mtr_start(&temp_mtr);
    temp_mtr.set_log_mode(MTR_LOG_NO_REDO);
  }
```

**第一段——加 rseg latch**：

注释说明了必须持 latch 的原因：**update undo 的 history list 必须按 `trx->no` 顺序串联**，purge 的内存数据结构依赖这个顺序。这是一处关键的保序约束——如果并发提交打乱了顺序，purge 的 `purge_queue` 就会失效。

临时表的 rseg 用独立的 `temp_mtr` 且设 `MTR_LOG_NO_REDO`——临时表不需要 redo（重启即丢）。

```cpp
  /* If transaction involves insert then truncate undo logs. */
  if (trx->rsegs.m_redo.insert_undo != nullptr) {
    trx_undo_set_state_at_finish(trx->rsegs.m_redo.insert_undo, mtr);
  }

  if (trx->rsegs.m_noredo.insert_undo != nullptr) {
    trx_undo_set_state_at_finish(trx->rsegs.m_noredo.insert_undo, &temp_mtr);
  }
```

**第二段——处理 insert undo**：

`trx_undo_set_state_at_finish` 把 insert undo 段状态从 `TRX_UNDO_ACTIVE` 改为 `TRX_UNDO_TO_FREE` 或 `TRX_UNDO_CACHED`。**insert undo 提交后即可丢弃**（没有事务需要看到"不存在"的行版本），所以这里直接截断/缓存，不需要进 history list，也不需要 `trx->no`。

```cpp
  bool serialised = false;

  /* If transaction involves update then add rollback segments
  to purge queue. */
  if (trx->rsegs.m_redo.update_undo != nullptr ||
      trx->rsegs.m_noredo.update_undo != nullptr) {

    trx_undo_ptr_t *redo_rseg_undo_ptr =
        trx->rsegs.m_redo.update_undo != nullptr ? &trx->rsegs.m_redo : nullptr;

    trx_undo_ptr_t *temp_rseg_undo_ptr =
        trx->rsegs.m_noredo.update_undo != nullptr ? &trx->rsegs.m_noredo
                                                   : nullptr;

    /* Will set trx->no and will add rseg to purge queue. */
    serialised = trx_serialisation_number_get(trx, redo_rseg_undo_ptr,
                                              temp_rseg_undo_ptr);
```

**第三段——update undo：分配 trx->no 并入 purge queue**：

只有存在 update undo 才走这里。这印证了前面的结论：**纯 INSERT 事务没有 `trx->no`**。

```cpp
    if (trx->rsegs.m_redo.update_undo != nullptr) {
      page_t *undo_hdr_page;

      undo_hdr_page =
          trx_undo_set_state_at_finish(trx->rsegs.m_redo.update_undo, mtr);

      /* Delay update of rseg_history_len if we plan to add
      non-redo update_undo too. This is to avoid immediate
      invocation of purge as we need to club these 2 segments
      with same trx-no as single unit. */
      bool update_rseg_len = !(trx->rsegs.m_noredo.update_undo != nullptr);

      /* Set flag if GTID information need to persist. */
      auto undo_ptr = &trx->rsegs.m_redo;
      trx_undo_gtid_set(trx, undo_ptr->update_undo, false);

      trx_undo_update_cleanup(trx, undo_ptr, undo_hdr_page, update_rseg_len,
                              (update_rseg_len ? 1 : 0), mtr);
    }
```

**第四段——update undo 收尾**：

1. `trx_undo_set_state_at_finish` 把 update undo 状态改为 committed 相关状态，并挂入 history list。
2. `update_rseg_len` 的延迟技巧：如果同时有临时表的 update undo，先不更新 `rseg_history_len`（避免立刻触发 purge），等两个段都用**同一个 `trx->no`** 挂好再一起更新——保证 purge 把它们当作一个整体。
3. `trx_undo_gtid_set(trx, update_undo, false)`：**把 GTID 写进 undo header**。这是 InnoDB 侧 GTID 持久化的落点（配合 redo 持久化），第三个参数 `false` 表示这是提交路径（prepare 路径传 `true`）。

```cpp
  if (Clone_handler::need_commit_order()) {
    trx_sys_update_mysql_binlog_offset(trx, mtr);
  }

  return (serialised);
}
```

**第五段——记录 binlog 位置**：

只有 `Clone_handler::need_commit_order()` 为真（Clone/备份需要一致性）时，才把当前 binlog 文件名与偏移写进 trx sys header。这是为 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 和 Clone 服务的——它们需要"InnoDB 提交位置 ↔ binlog 位置"的映射。

#### trx->no 的分配：trx_serialisation_number_get

```cpp
static bool trx_serialisation_number_get(
    trx_t *trx,
    trx_undo_ptr_t *redo_rseg_undo_ptr,
    trx_undo_ptr_t *temp_rseg_undo_ptr) {
  bool added_trx_no;
  trx_rseg_t *redo_rseg = nullptr;
  trx_rseg_t *temp_rseg = nullptr;
  ...
  /* If the rollack segment is not empty then the
  new trx_t::no can't be less than any trx_t::no
  already in the rollback segment. User threads only
  produce events when a rollback segment is empty. */
  // ★ 关键优化：last_page_no == FIL_NULL 表示这个 rseg 当前是空的
  //   ⇒ 只有 rseg 从空变为非空时才需要往 purge_queue push 新元素
  if ((redo_rseg != nullptr && redo_rseg->last_page_no == FIL_NULL) ||
      (temp_rseg != nullptr && temp_rseg->last_page_no == FIL_NULL)) {
    TrxUndoRsegs elem;
    ...
    mutex_enter(&purge_sys->pq_mutex);

    added_trx_no = trx_add_to_serialisation_list(trx);

    elem.set_trx_no(trx->no);

    purge_sys->purge_queue->push(std::move(elem));

    mutex_exit(&purge_sys->pq_mutex);

  } else {
    added_trx_no = trx_add_to_serialisation_list(trx);   // 非空 rseg：只分配 no，不 push
  }

  return (added_trx_no);
}
```

**★ 为什么可以这样优化**：同一 rseg 上后续事务的 undo 会串到已有 history list 尾部，`trx->no` 一定比已有的大（见上方源码注释）。purge 只需知道每个 rseg 的**起始位置**，后续按链表顺序扫即可。⇒ 把 `purge_queue` 的 push 频率从"每事务一次"降到"每 rseg 从空变非空一次"，大幅减少 `pq_mutex` 竞争。

`trx_add_to_serialisation_list` 本身：

```cpp
static inline bool trx_add_to_serialisation_list(trx_t *trx) {
  trx_sys_serialisation_mutex_enter();

  trx->no = trx_sys_allocate_trx_no();

  ut_d(trx_sys->rw_max_trx_no = trx->no);

  if (trx->read_only) {
    trx_sys_serialisation_mutex_exit();
    return false;
  }

  UT_LIST_ADD_LAST(trx_sys->serialisation_list, trx);

  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) == 1) {
    trx_sys->serialisation_min_trx_no.store(trx->no);
  }

  trx_sys_serialisation_mutex_exit();
  return true;
}
```

在 serialisation mutex 下单调分配 `trx->no`，加入 `serialisation_list`。返回 false 表示只读事务（不进链表）。

#### 第三层：trx_commit_in_memory（内存世界）

```cpp
static void trx_commit_in_memory(
    trx_t *trx, const mtr_t *mtr, bool serialised) {

  trx->must_flush_log_later = false;
  trx->ddl_must_flush = false;

  if (trx_is_autocommit_non_locking(trx)) {
    ...
    ut_a(UT_LIST_GET_LEN(trx->lock.trx_locks) == 0);
    ...
    if (trx->read_view != nullptr) {
      trx_sys->mvcc->view_close(trx->read_view, false);
    }

    MONITOR_INC(MONITOR_TRX_NL_RO_COMMIT);

    /* AC-NL-RO transactions can't be rolled back asynchronously. */
    ut_ad(!trx->abort);
    ut_ad(!(trx->in_innodb & TRX_FORCE_ROLLBACK));

    trx->state.store(TRX_STATE_NOT_STARTED, std::memory_order_relaxed);

  } else {
    // 2PL shrinking
    trx_release_impl_and_expl_locks(trx, serialised);
    ...
  }
```

**AC-NL-RO 快路径**：关闭 read view，状态直接回 `NOT_STARTED`（连 `COMMITTED_IN_MEMORY` 都不经过），全程不碰锁链表、不碰 `rw_trx_list`。这就是前面说的"让不该有开销的路径彻底没有开销"。

**常规路径**：走 `trx_release_impl_and_expl_locks`（2PL shrinking phase）。

```cpp
static void trx_release_impl_and_expl_locks(trx_t *trx, bool serialised) {
  check_trx_state(trx);
  ut_ad(trx_state_eq(trx, TRX_STATE_ACTIVE) ||
        trx_state_eq(trx, TRX_STATE_PREPARED));

  bool trx_sys_latch_is_needed =
      (trx->id > 0) || trx_state_eq(trx, TRX_STATE_PREPARED);

  /* Check and get GTID to be persisted. Do it outside mutex. It must be done
  before trx->state is changed to TRX_STATE_COMMITTED_IN_MEMORY, ... */
  Gtid_desc gtid_desc{};
  if (serialised) {
    auto &gtid_persistor = clone_sys->get_gtid_persistor();
    gtid_persistor.get_gtid_info(trx, gtid_desc);
  }

  if (trx_sys_latch_is_needed) {
    trx_sys_mutex_enter();
  }

  if (trx->id > 0) {
    /* For consistent snapshot, we need to remove current
    transaction from running transaction id list for mvcc
    before doing commit and releasing locks. */
    trx_erase_lists(trx);
  }

  if (trx_state_eq(trx, TRX_STATE_PREPARED)) {
    ut_a(trx_sys->n_prepared_trx > 0);
    --trx_sys->n_prepared_trx;
  }

  if (trx_sys_latch_is_needed) {
    trx_sys_mutex_exit();
  }
```

**第一段——获取 GTID 与摘链表**：

- GTID 信息在 **mutex 外**获取（注释说是为了避免在锁内做复杂操作），且必须在状态变 `COMMITTED_IN_MEMORY` **之前**——因为 `get_gtid_info` 内部会检查 `trx->state`。
- `trx_erase_lists(trx)`：从 `rw_trx_ids` / `rw_trx_list` 摘除。注释强调"必须在放锁之前"——否则一个 read view 可能仍然认为该事务的修改不可见，但相关 undo 却已经被 purge 了。

```cpp
  auto state_transition = [&]() {
    trx_mutex_enter(trx);
    /* Please consider this particular point in time as the moment the trx's
    implicit locks become released.
    This change is protected by both Trx_shard's mutex and trx->mutex. ... */
    trx->state.store(TRX_STATE_COMMITTED_IN_MEMORY, std::memory_order_relaxed);
    trx_mutex_exit(trx);
  };

  if (trx->id > 0) {
    trx_sys->get_shard_by_trx_id(trx->id).active_rw_trxs.latch_and_execute(
        [&](Trx_by_id_with_min &trx_by_id_with_min) {
          state_transition();
          trx_by_id_with_min.erase(trx->id);
        },
        UT_LOCATION_HERE);
  } else {
    state_transition();
  }
```

**第二段——状态迁移（最微妙的部分）**：

状态变 `COMMITTED_IN_MEMORY` **与**从 `Trx_shard::active_rw_trxs` 移除，这两个动作必须在**同一个 shard latch 内**原子完成。

**为什么**？注释给出了完整解释：这一刻是"隐式锁被释放"的时刻。别的线程判断"某个 trx 是否还持有隐式锁"有两条路径：

1. 只知道 trx_id → 拿 `Trx_shard` mutex，检查是否还在 `active_rw_trxs`（`lock_rec_convert_impl_to_expl` 用这条）
2. 持有 trx 指针 → 拿 `trx->mutex`，检查 `state == COMMITTED_IN_MEMORY`（`lock_rec_convert_impl_to_expl_for_trx` 用这条）

因为状态迁移同时受 `trx->mutex` 和 shard mutex 保护，两条路径看到的视图是一致的——不会出现"已从 shard 移除但状态还没变"的中间态。这是 8.0 分片化后为保证正确性引入的精妙设计。

```cpp
  /* It is important to remove the transaction from the serialisation list
  after it is erased from the rw_trx_ids / rw_trx_list (not before!).
  Otherwise a read-view could be created, which could still pretend that
  changes of this transaction are invisible, but related undo records could
  become purged (because trx->no would no longer protect them). */

  if (serialised) {
    trx_sys_serialisation_mutex_enter();

    /* Add GTID to be persisted to disk table. It must be done ...
    1.After the transaction is marked committed in undo. Otherwise
      GTID might get committed before the transaction commit on disk.
    2.Before it is removed from serialization list. Otherwise the transaction
      undo could get purged before persisting GTID on disk table. */
    if (gtid_desc.m_is_set) {
      auto &gtid_persistor = clone_sys->get_gtid_persistor();
      gtid_persistor.add(gtid_desc);
    }

    trx_erase_from_serialisation_list_low(trx);

    trx_sys_serialisation_mutex_exit();
  }

  lock_trx_release_locks(trx);
}
```

**第三段——GTID 落盘与最终放锁**：

GTID 的持久化时机被两条约束夹住，注释编号列出：

1. **必须在 undo 标记 committed 之后**——否则 GTID 可能比事务本身先持久化（崩溃后 GTID 说提交了但数据没有）。
2. **必须在移除 serialisation_list 之前**——否则 `trx->no` 不再保护 undo，undo 可能被 purge 掉，GTID 来不及落盘。

这个"两难之间"的精确位置，是 GTID 与事务一致性保证的关键。详见 `../server/replication/gtid.md`。

最后 `lock_trx_release_locks(trx)` 释放事务持有的全部锁——2PL 的 shrinking phase 在此完成。

---

### 事务与 binlog：2PC

> **边界**：本节是**内部 XA**（binlog 当协调者）。由外部 TM 裁决的**外部 XA** 已独立成篇，见 [`../server/xa.md`](../server/xa.md)（含 `XA_prepare_log_event`、`Xa_state_list` 六态、`xa_detach_on_prepare`）。

#### 为什么需要：两个独立的日志系统

一个事务的持久化数据分散在两处：InnoDB 的 redo/undo（引擎内部原子性）与 binlog（复制与 PITR）。两者必须原子地一起提交，否则主从不一致。

这是经典的 **2PC（Gray 1978）** 应用：**binlog 充当协调者（coordinator），InnoDB 是参与者（participant）**。

#### 协调者的选择

`tc_log` 是全局抽象基类指针：

- 开启 binlog → `tc_log = &mysql_bin_log`，binlog 文件充当事务协调日志
- 关闭 binlog → `tc_log = &tc_log_dummy`，退化为引擎自己 1PC

#### 触发条件：rw_ha_count > 1

```cpp
if (!trn_ctx->no_2pc(trx_scope) && (trn_ctx->rw_ha_count(trx_scope) > 1))
  error = tc_log->prepare(thd, all);
```

`rw_ha_count` 统计读写参与者个数。开 binlog 且改了 InnoDB 数据时，参与者是 {binlog, InnoDB} 两个 → `rw_ha_count = 2` → 触发完整 2PC。关 binlog 只有 InnoDB 一个 → `= 1` → **跳过 prepare 走 1PC 快速路径**。

#### 演进：从串行化到 BGC 协同

2PC 在 MySQL 中的实现经历过一次**根本性重构**。不理解它，就无法理解为啥会有 `HA_IGNORE_DURABILITY` 这种反直觉设计。

| 阶段 | 机制 | 问题 |
|------|------|------|
| 5.6 及更早 | `prepare_commit_mutex` 把 prepare→commit 全程串行化 | **彻底破坏 group commit**，所有提交排队 |
| 5.7.6+ | `HA_IGNORE_DURABILITY` 让 prepare 的 redo 延迟到 flush stage 批量 fsync，由 BGC 三阶段保序 | 保序代价从"串行执行"降为"流水线" |
| 8.0.29+ | 引入 `TRX_UNDO_PREPARED_IN_TC`，区分"引擎已 prepare"与"TC(binlog) 已处理" | 修复旧版崩溃恢复的边界情况 |

**第一阶段为何失败**：早期为保证 binlog 与 InnoDB 提交顺序一致，用一把 `prepare_commit_mutex` 锁住 prepare 到 commit 的全过程。顺序确实保证了，但 **group commit 完全失效**——每个事务都要完整走一遍串行路径，高并发下是灾难。

**第二阶段的解法**：`trx_commit_in_memory` 里那段注释正是这个时代的遗留物：

```cpp
/* If we are calling trx_commit() under prepare_commit_mutex, we
will delay possible log write and flush to a separate function
trx_commit_complete_for_mysql(), which is only called when the
thread has released the mutex. This is to make the
group commit algorithm to work. */
```

新方案的核心洞察是：**顺序不需要靠"串行执行"保证，可以靠"流水线阶段"保证**。

- prepare 阶段写 redo 但**不 fsync**（`HA_IGNORE_DURABILITY`）；
- 整组事务的 redo 在 flush stage 由 leader 一次 fsync；
- binlog 写入顺序由 `LOCK_log` 保证，与 redo fsync 顺序一致；
- 于是"binlog 顺序 == InnoDB 提交顺序"这个 2PC 所需的保证，通过流水线天然成立。

`prepare_commit_mutex` 在 8.0.39 已彻底移除（仅注释残留），但 `must_flush_log_later` / `trx_commit_complete_for_mysql()` 这套"延后刷日志"机制保留了下来，由 `trx->flush_log_later` 在 BGC 场景下承担同样职责。

**第三阶段为何引入 `PREPARED_IN_TC`**：旧格式下崩溃恢复只能看到"这个事务 prepare 了"，**无法判断 binlog 是否已经写过它**。8.0.29 把状态拆成两级后，恢复逻辑才能区分：

- `TRX_UNDO_PREPARED`：引擎已 prepare，TC 还没处理 → 需要查 binlog 裁决；
- `TRX_UNDO_PREPARED_IN_TC`：TC 已处理过 → 走确定路径。

`trx0undo.h` 的注释直接标明了来源——`TRX_UNDO_PREPARED_80028`(5) 是 "for a server version older than 8.0.29"，保留只为升级兼容。

代码上体现为两个近乎相同的函数：`trx_prepare_low`（写 `PREPARED`）与 `trx_set_prepared_in_tc_low`（写 `PREPARED_IN_TC`），后者供 `innobase_set_prepared_in_tc` / `innobase_set_prepared_in_tc_by_xid` 调用。

**"标记 prepared in TC" 到底做了什么**——往下拆一层会发现它**不只是改状态**，还要顺带把 GTID 持久化：

```cpp
// trx0trx.cc:3714
dberr_t trx_set_prepared_in_tc_for_mysql(trx_t *trx) {
  ut_a(trx->state.load(std::memory_order_relaxed) == TRX_STATE_PREPARED);   // ① 前置：必须已 PREPARED

  /* For GTID persistence we need update undo segment. */
  auto db_err = trx_undo_gtid_add_update_undo(trx, true, false);            // ② ★ 为 GTID 写一条 undo
  if (db_err != DB_SUCCESS) return (db_err);

  trx->op_info = "marking transaction as prepared in TC";
  trx_set_prepared_in_tc(trx);                                              // ③ 真正标记
  trx->op_info = "";
  return (DB_SUCCESS);
}

// trx0trx.cc:3145
static void trx_set_prepared_in_tc(trx_t *trx) {
  ut_a(trx->state.load(std::memory_order_relaxed) == TRX_STATE_PREPARED);

  lsn_t lsn = trx_set_prepared_in_tc_low(trx, &trx->rsegs.m_redo);          // 写 undo 段状态 → PREPARED_IN_TC

  /* Check and get GTID to be persisted. Do it outside trx_sys mutex. */
  auto &gtid_persistor = clone_sys->get_gtid_persistor();
  Gtid_desc gtid_desc;
  gtid_persistor.get_gtid_info(trx, gtid_desc);                             // ★ 锁外取 GTID

  /* Add GTID to be persisted to disk table, if needed. */
  if (gtid_desc.m_is_set) {
    /* The gtid_persistor.add() might release and re-acquire the mutex. */
    trx_sys_serialisation_mutex_enter();                                    // ★ 才加串行化锁
    ...
```

三条都是看调用名看不出来的：

1. ★ **XA PREPARE 会额外写一条 undo**——`trx_undo_gtid_add_update_undo` 的目的写在注释里：*"For GTID persistence we need update undo segment"*。GTID 要跟着这条 prepared 事务**落到 undo 段里**，这样崩溃恢复时才能知道这个悬而未决的 prepared 事务对应哪个 GTID（否则恢复后无法与 binlog/从库对齐）。
2. ★ **锁顺序是刻意安排的**：先写 undo 段状态（持 rseg latch），然后**在 `trx_sys` mutex 之外**取 GTID（注释明说 *"Do it outside trx_sys mutex"*），最后才加 `trx_sys_serialisation` mutex 去持久化。把"查 GTID 信息"这种相对重的操作挪到锁外，是为了缩短全局锁持有时间。
3. ★ **`gtid_persistor.add()` 可能释放并重获取 mutex**——源码注释专门警告这一点（`might release and re-acquire the mutex`），所以调用点不能假设"进了临界区就一直持有"，前后的状态都要重新确认。

⇒ 这也解释了 `TRX_UNDO_PREPARED_IN_TC` 这个段状态为什么存在：它比 `PREPARED` 多记录了一层"TC 已经处理过、GTID 已持久化"的信息，恢复时据此走确定路径（见上文 1212 行）。

#### 阶段一：prepare 逐行剖析

调用链：`MYSQL_BIN_LOG::prepare` → `ha_prepare_low` → `innobase_xa_prepare` → `trx_prepare_for_mysql` → `trx_prepare` → `trx_prepare_low`。逐个看：

**① `MYSQL_BIN_LOG::prepare`——协调者的开场**

```cpp
int MYSQL_BIN_LOG::prepare(THD *thd, bool all) {
  assert(opt_bin_log);

  /* Set HA_IGNORE_DURABILITY to not flush the prepared record of the
  transaction to the log of storage engine (for example, InnoDB
  redo log) during the prepare phase. So that we can flush prepared
  records of transactions to the log of storage engine in a group
  right before flushing them to binary log during binlog group
  commit flush stage. */
  thd->durability_property = HA_IGNORE_DURABILITY;

  int error = ha_prepare_low(thd, all);

  // Invoke `commit` if we're dealing with `XA PREPARE` in order to use BGC
  // to write the event to file.
  if (!error && all && is_xa_prepare(thd)) return this->commit(thd, true);

  return error;
}
```

- 设 `HA_IGNORE_DURABILITY`：告诉引擎"prepare 的 redo 别 fsync"，这是演进第二阶段的核心（见上文）。
- **XA PREPARE 特殊分支**：`is_xa_prepare(thd)` 为真时，prepare 完成后**立刻调用 `commit(thd, true)`**。这不是真的提交——`MYSQL_BIN_LOG::commit` 里 `skip_commit = is_loggable_xa_prepare(thd)`，作用是**借 BGC 流水线把 `XA_prepare_log_event` 写进 binlog 文件**，不做引擎 commit。这是外部 XA 要求"prepare 也要持久记录"的体现。

**② `ha_prepare_low`——遍历参与者**

```cpp
int ha_prepare_low(THD *thd, bool all) {
  int error = 0;
  auto ha_list = thd->get_transaction()->ha_trx_info(trx_scope);

  if (ha_list) {
    for (auto const &ha_info : ha_list) {
      if (!ha_info.is_trx_read_write() &&   // 只读事务跳过 2PC
          !thd_holds_xa_transaction(thd))   // 但 XA 事务不能跳过
        continue;

      int err = ht->prepare(ht, thd, all);
      ...
    }
    DBUG_EXECUTE_IF("crash_commit_after_prepare", DBUG_SUICIDE(););
  }
  return error;
}
```

- **只读事务跳过**：`!is_trx_read_write()` 且非 XA → `continue`。只读事务没有 redo/undo 要持久化，不参与 prepare。
- **XA 是例外**：`thd_holds_xa_transaction(thd)` 为真时**即使只读也要走 prepare**——因为外部 XA 的 prepare 状态必须对协调者可见、可恢复。
- 崩溃注入点 `crash_commit_after_prepare` 就在循环结束后。

**③ `innobase_xa_prepare`——InnoDB 的 prepare 回调**

```cpp
static int innobase_xa_prepare(handlerton *hton, THD *thd, bool prepare_trx) {
  trx_t *trx = check_trx_exists(thd);

  // ★ 把 server 层的 XID 写进 trx->xid——这是崩溃恢复的钥匙
  //   ⇒ 恢复时靠 XID 在 binlog 里查找这个事务
  thd_get_xid(thd, (MYSQL_XID *)trx->xid);

  // 注意这里没有传 disable=true（对比 innobase_commit / innobase_rollback 都传 true）
  // ⇒ prepare 后事务还可能被回滚，不能设 TRX_FORCE_ROLLBACK_DISABLE（那会禁止异步回滚）
  TrxInInnoDB trx_in_innodb(trx);

  if (trx_in_innodb.is_aborted() || ...) {
    innobase_rollback(hton, thd, prepare_trx);
    return (convert_error_code_to_mysql(DB_FORCED_ABORT, 0, thd));   // abort → 转回滚
  }

  if (!trx_is_registered_for_2pc(trx) && trx_is_started(trx)) {
    log_errlog(ERROR_LEVEL, ER_INNODB_UNREGISTERED_TRX_ACTIVE);
  }

  // ★ 分支：显式要求 prepare 整个事务，或 autocommit 模式 → 真正 trx_prepare_for_mysql
  if (prepare_trx ||
      (!thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN))) {
    ut_ad(trx_is_registered_for_2pc(trx));

    dberr_t err = trx_prepare_for_mysql(trx);

    if (err == DB_FORCED_ABORT) {
      innobase_rollback(hton, thd, prepare_trx);                     // prepare 失败 = 事务失败
      return (convert_error_code_to_mysql(DB_FORCED_ABORT, 0, thd));
    }
  } else {
    // 语句结束但事务还在继续 ⇒ 只释放 autoinc 锁 + trx_mark_sql_stat_end
    //   （后者记录语句边界，供语句级回滚）
    lock_unlock_table_autoinc(trx);
    trx_mark_sql_stat_end(trx);
  }
  ...
}
```

**④ `trx_prepare_low`——文件态落地**

```cpp
static lsn_t trx_prepare_low(trx_t *trx, trx_undo_ptr_t *undo_ptr,
                             bool noredo_logging) {
  if (undo_ptr->insert_undo != nullptr || undo_ptr->update_undo != nullptr) {
    mtr_t mtr;
    mtr_start_sync(&mtr);

    // 临时表 rseg（m_noredo）走 MTR_LOG_NO_REDO——临时表不需要 redo
    if (noredo_logging) {
      mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);
    }

    /* Change the undo log segment states from TRX_UNDO_ACTIVE to
    TRX_UNDO_PREPARED: these modifications to the file data structure define
    the transaction as prepared in the file-based world, at the serialization
    point of lsn. */

    // 持 rseg latch 改 undo 段头状态：TRX_UNDO_ACTIVE → TRX_UNDO_PREPARED
    // ⇒ 必须串行化，否则同 rseg 并发写段头会互相破坏
    rseg->latch();

    if (undo_ptr->insert_undo != nullptr) {
      trx_undo_set_state_at_prepare(trx, undo_ptr->insert_undo, false, &mtr);
    }

    if (undo_ptr->update_undo != nullptr) {
      // ★ insert 与 update 都改状态，但只有 update undo 在非 noredo 时才写 GTID
      //   ⇒ GTID 在 prepare 阶段就已写进 update undo header
      //     （第三个参数 true 表示 prepare 场景，对应提交时的 false）
      if (!noredo_logging) {
        trx_undo_gtid_set(trx, undo_ptr->update_undo, true);
      }
      trx_undo_set_state_at_prepare(trx, undo_ptr->update_undo, false, &mtr);
    }

    rseg->unlatch();

    /* This mtr commit makes the transaction prepared in file-based world. */
    // ★ 语义：这个 mtr_commit 让事务在"文件世界"成为 prepared——呼应"文件态先行"
    //   返回的 commit_lsn 最终传给 trx_flush_logs(trx, lsn)
    //   （但因 HA_IGNORE_DURABILITY 不 fsync）
    mtr_commit(&mtr);

    if (!noredo_logging) {
      return mtr.commit_lsn();
    }
  }
  return 0;
}
```

**⑤ `trx_prepare` 收尾——内存态跟进**

```cpp
static void trx_prepare(trx_t *trx) {
  lsn_t lsn = 0;

  if (trx->rsegs.m_redo.rseg != nullptr && trx_is_redo_rseg_updated(trx)) {
    lsn = trx_prepare_low(trx, &trx->rsegs.m_redo, false);
  }
  if (trx->rsegs.m_noredo.rseg != nullptr && trx_is_temp_rseg_updated(trx)) {
    trx_prepare_low(trx, &trx->rsegs.m_noredo, true);
  }

  trx_sys_mutex_enter();
  trx->state.store(TRX_STATE_PREPARED, std::memory_order_relaxed);
  trx_sys->n_prepared_trx++;
  trx_sys_mutex_exit();

  if (trx->releases_gap_locks_at_prepare()) {
    trx->skip_lock_inheritance = true;
    lock_trx_release_read_locks(trx, true);
  }

  if (lsn > 0) {
    trx_flush_logs(trx, lsn);
  }
}
```

- 先 redo rseg 再 noredo rseg，然后才 `state → TRX_STATE_PREPARED` + `n_prepared_trx++`（`trx_sys->mutex` 保护）。**文件态先行、内存态后到**在此落地。
- **RC 及以下隔离级别会在 prepare 时释放 GAP 锁**（`releases_gap_locks_at_prepare()`）——这是 RC 下锁更少的原因之一。
- `trx_flush_logs(trx, lsn)` 因 `HA_IGNORE_DURABILITY` **不真正 fsync**，真正的 fsync 推迟到 BGC 的 flush stage。

```cpp
static void trx_prepare(trx_t *trx) {
  lsn_t lsn = 0;
  ut_a(!trx->is_recovered);

  if (trx->rsegs.m_redo.rseg != nullptr && trx_is_redo_rseg_updated(trx)) {
    lsn = trx_prepare_low(trx, &trx->rsegs.m_redo, false);
  }

  if (trx->rsegs.m_noredo.rseg != nullptr && trx_is_temp_rseg_updated(trx)) {
    trx_prepare_low(trx, &trx->rsegs.m_noredo, true);
  }

  trx_sys_mutex_enter();
  trx->state.store(TRX_STATE_PREPARED, std::memory_order_relaxed);
  trx_sys->n_prepared_trx++;
  trx_sys_mutex_exit();
  ...
  if (lsn > 0) {
    trx_flush_logs(trx, lsn);
  }
}
```

`trx_prepare_low` 的核心：把 undo 段状态从 `TRX_UNDO_ACTIVE` 改为 `TRX_UNDO_PREPARED`，通过 MTR 写 redo，拿到 `lsn`。

**关键优化——`HA_IGNORE_DURABILITY`**：`MYSQL_BIN_LOG::prepare` 在进入引擎前设置 `thd->durability_property = HA_IGNORE_DURABILITY`，使得最后的 `trx_flush_logs(trx, lsn)` **不真正 fsync**。原因：把整个 group 的 redo prepare 记录攒起来，到 flush stage 由 `ha_flush_logs` 一次性批量 fsync。这是 redo group commit 与 binlog group commit 的协同点。

#### 阶段二：commit

`tc_log->commit` → `MYSQL_BIN_LOG::commit` → `ordered_commit` 三阶段（详见 `../server/replication/binlog.md`）。

**提交点是 sync stage 的 binlog fsync 成功那一刻**，不是引擎 commit。携带 XID 的 `Xid_log_event` 一旦落盘，事务就算提交。

**Server 层在 commit 阶段的收尾**（`ha_commit_trans` 内）：

| 动作 | 说明 |
|------|------|
| 驱动 commit | `tc_log->commit(thd, all)` |
| 失败则 rollback | commit 出错走 `ha_rollback_trans` 回滚整个事务 |
| 释放 MDL COMMIT 锁 | commit 完成后释放 `MDL_INTENTION_EXCLUSIVE`（阻止 FTWRL 的那把） |
| 清理事务上下文 | `trn_ctx->cleanup()` 重置事务 scope |
| GTID 收尾 | `gtid_state->update_on_commit/rollback` 按提交结果推进 GTID 状态 |

#### 崩溃恢复的裁决

崩溃恢复时，InnoDB 扫描 undo 找出所有 `TRX_STATE_PREPARED` 的事务，把它们交给 binlog 恢复逻辑裁决：XID 出现在 binlog 的 commit_list 中 → COMMIT；否则 → ROLLBACK。

| 崩溃时机 | InnoDB 状态 | binlog 状态 | 裁决 |
|---------|------------|-------------|------|
| prepare 之后、写 binlog 之前 | PREPARED | 无 XID | ROLLBACK |
| 写 binlog 之后、fsync 之前 | PREPARED | XID 在 OS cache（可能丢） | ROLLBACK |
| binlog fsync 之后 | PREPARED | XID 已落盘 | COMMIT |
| 引擎 commit 之后 | COMMITTED | XID 已落盘 | 无需处理 |

#### 外部 XA：与内部 2PC 的差异

上面讲的是**内部 2PC**（普通事务开 binlog，对用户透明）。MySQL 另支持标准 **外部 XA**（`XA START` / `XA END` / `XA PREPARE` / `XA COMMIT`），由应用层显式控制 prepare 与 commit。两者共用同一套 `ht->prepare` / `ht->commit` 引擎接口，但有三处关键差异。

**差异一：prepare 要写 binlog 事件**

内部 2PC 的 prepare 只在 redo 与内存里做标记，**不写 binlog**。外部 XA 的 `XA PREPARE` 必须把 `XA_prepare_log_event` 写进 binlog——prepare 状态需要可恢复、需要复制到从库。

这正是 `MYSQL_BIN_LOG::prepare` 尾部的用途：

```cpp
// Invoke `commit` if we're dealing with `XA PREPARE` in order to use BGC
// to write the event to file.
if (!error && all && is_xa_prepare(thd)) return this->commit(thd, true);
```

借 `commit` 走 BGC 流水线写事件，但 `skip_commit` 为真，引擎层不提交。

**差异二：detach / attach 的三层机制**

外部 XA 的 prepared 事务可以在**另一个连接**（甚至崩溃重启后）被 COMMIT/ROLLBACK。为此，prepared 事务必须能与原连接**解绑**（detach），并能在新连接**重新绑定**（attach）。

**为什么需要 detach——根本原因是生命周期跨越连接边界**：普通事务的 prepare→commit 在 `ha_commit_trans` 内连续执行（两次调用之间没有"等待外部决定"的窗口），事务从生到死都在同一 THD 上，提交完即释放，不存在"保留 prepared 等别人"的状态。外部 XA 则不然——prepare 与 commit 之间可以隔**任意长时间、任意连接、甚至一次重启**：

1. **跨连接裁决**：`XA COMMIT xid` 按 xid 指定事务，可在任意连接执行。而 THD 只有一个 trx 槽位（`thd_to_trx`），不 detach 原连接就无法开始新事务。
2. **连接复用**：`xa_detach_on_prepare=ON` 时 PREPARE 后立即释放连接（连接池 / 应用层长事务管理的刚需）。这个参数的存在本身就说明 detach 是 XA 释放连接的机制。
3. **跨重启裁决**：detach 后事务留在引擎 2PC 注册表（prepared 状态 + XID）+ Server 层 Transaction_cache（XID_STATE）。崩溃恢复时 resurrect，由 binlog 裁决（见「崩溃恢复的提交与回滚」）。

这在三层各有一个对应物：

| 层 | 机制 | 动作 |
|----|------|------|
| Server 层 `XID_STATE` | `m_is_detached` 标志（xa.h） | 协调器按此分流 commit/rollback 路径 |
| Server 层 `Transaction_cache` | 按 xid 存 detached 事务上下文 | `find_trn_for_recover_and_check_its_state` 查找 |
| InnoDB 层 `trx_t` | `trx->mysql_thd = nullptr` | 切断 trx 与连接的关联 |

**InnoDB 层：`innodb_replace_trx_in_thd` 的三种模式**

```cpp
static void innodb_replace_trx_in_thd(THD *thd, void *new_trx_arg,
                                      void **ptr_trx_arg) {
  trx_t *&trx = thd_to_trx(thd);

  if (ptr_trx_arg) {
    *ptr_trx_arg = trx;                      // 模式一：备份
  } else if (trx != nullptr) {
    if (trx->state.load(...) == TRX_STATE_NOT_STARTED) {
      trx_free_for_mysql(trx);               // 无状态，直接释放
    } else {
      ut_ad(trx_state_eq(trx, TRX_STATE_PREPARED));
      /* The choice not to disconnect transaction when
      trx_is_redo_rseg_updated(trx) is false is copied from
      the existing detach logic for prepared XA
      transactions in innobase_close_connection().
      If transaction did not modify anything, rolling it back doesn't modify
      the db, but lets us deregister and free the trx object to conserve
      resources, while still allowing XA COMMIT it in the future, as it
      succeeds on missing xids. */
      if (trx_is_redo_rseg_updated(trx)) {
        trx_disconnect_prepared(trx);        // 模式二：断开但保留 prepared
      } else {
        trx_rollback_for_mysql(trx);         // 空事务：回滚 + 释放
        trx_deregister_from_2pc(trx);
        trx_free_for_mysql(trx);
      }
    }
  }
  trx = static_cast<trx_t *>(new_trx_arg);   // 模式三：装回
}
```

**三种模式**（代码内已就近标注，此处只补模式层面的语义）：

- **模式一（备份，`ptr_trx_arg != nullptr`）**：把当前 trx 存进备份缓冲（server 层的 `ha_ptr_backup`），函数末尾 THD 的 trx 被置为 `new_trx_arg`（nullptr）。这是 `XA START` 时的"临时摘下"——把连接之前的普通事务保存起来让位给 XA 事务，XA 结束后再装回。
- **模式二（断开，两个参数都 null）**：`xa_detach_on_prepare` 时真正 detach，三种情况见代码内注释。★ 值得单独强调的：PREPARED 但**没改过数据** → 回滚 + deregister + free——空事务回滚不修改 db，且未来 `XA COMMIT` 对 missing xid 也返回成功，不必保留对象，省资源。
- **模式三（装回）**：`reattach_native_trx` 调用，把备份的 trx 重新挂回 THD。

**断开的竞态同步：`trx_disconnect_from_mysql`**

```cpp
inline void trx_disconnect_from_mysql(trx_t *trx, bool prepared) {
  trx_sys_mutex_enter();
  UT_LIST_REMOVE(trx_sys->mysql_trx_list, trx);
  if (trx->read_view != nullptr) {
    trx_sys->mvcc->view_close(trx->read_view, true);
  }
  if (prepared) {
    ut_ad(trx_state_eq(trx, TRX_STATE_PREPARED));
    /* During disconnection there is a short period when the server layer
    already believes this XID is detached from this connection, and thus another
    connection may try to XA COMMIT/ROLLBACK it, yet InnoDB is still processing
    the trx object - another client can be executing innobase_commit_by_xid() in
    parallel to our trx_disconnect_from_mysql(). We use trx->mysql_thd field,
    under protection of trx_sys->mutex to synchronize with trx_get_trx_by_xid()
    and convey if this trx is still attached to this thd or not. */
    trx->mysql_thd = nullptr;
  }
  trx_sys_mutex_exit();
}
```

★ **这里有一个注释点破的竞态窗口**：断开期间，server 层已认为 XID detached（另一个连接可以开始 `XA COMMIT`），但 InnoDB 还在处理 trx 对象——另一个客户端可能正并行执行 `innobase_commit_by_xid()`。同步手段是 `trx->mysql_thd` 字段 + `trx_sys->mutex`：`trx_get_trx_by_xid()` 通过检查该字段判断"这个 trx 是否还 attached 到某个连接"。

**Server 层：协调器按 `is_detached()` 分流**

```cpp
bool trx_coordinator::commit_in_engines(THD *thd, bool all, ...) {
  if (thd->get_transaction()->xid_state()->is_detached())
    return trx_coordinator::commit_detached_by_xid(thd, run_after_commit);
  else
    return ha_commit_low(thd, all, run_after_commit);
}
```

detached 事务 → `ht->commit_by_xid`（按 xid 提交，不依赖 trx 挂在 THD）；正常事务 → `ha_commit_low`。`commit_detached_by_xid` 用 RAII 守卫在结束时 `xs->reset()` 清掉 detached 标记。

**XID_STATE 的 detached 初始化**（xa.h）：

```cpp
void start_detached_xa(const XID *xid, bool binlogged_arg = false) {
  xa_state = XA_PREPARED;
  m_xid.set(xid);
  m_is_detached = true;      // ★ detached 标记
  rm_error = 0;
  m_is_binlogged = binlogged_arg;
}
```

处理 detached 事务的 `XA COMMIT/ROLLBACK` 时，XID_STATE 以 `XA_PREPARED + detached` 初始化，协调器据此走 by_xid 路径。

**完整生命周期**：

```
XA START
  → detach_native_trx                       把普通事务备份到 ha_ptr_backup，THD trx 置空
DML ... / XA END
XA PREPARE
  → process_xa_prepare                      引擎 prepare（undo → PREPARED）
  → xa_detach_on_prepare=ON → detach_xa_transaction → disconnect_native_trx
      → innodb_replace_trx_in_thd(thd, nullptr, nullptr)
          → trx_disconnect_prepared → trx->mysql_thd = nullptr
      → Transaction_cache 登记（XID_STATE::start_detached_xa）
连接继续（THD 无 trx 关联，可开新事务）

另一个连接: XA COMMIT xid
  → find_and_initialize_xa_context          从 Transaction_cache 找 detached 上下文
  → acquire_locks（m_xa_lock 防并发 + 双重检查）
  → commit_in_engines → is_detached → commit_detached_by_xid → ht->commit_by_xid
      → innobase_commit_by_xid（无 THD，按 xid 提交）
  → Transaction_cache::remove
```

`xa_detach_on_prepare` 参数：OFF（默认）→ PREPARE 后事务仍 attached（旧行为）；ON → PREPARE 后立即 detach，连接释放。这也是 `innobase_xa_prepare` 里 `TrxInInnoDB` **不传 `disable=true`** 的原因之一——detached 事务不能禁止异步回滚。

**差异三：按 xid 操作的 by_xid 接口**

`XA RECOVER` 与崩溃后的 `XA COMMIT/ROLLBACK` 需要按 xid 查找事务，而非按 THD。InnoDB 提供 by_xid 入口：

```cpp
static xa_status_code innobase_rollback_by_xid(handlerton *hton, XID *xid) {
  trx_t *trx = trx_get_trx_by_xid(xid);

  if (trx != nullptr) {
    int ret;
    {
      TrxInInnoDB trx_in_innodb(trx);
      ret = innobase_rollback_trx(trx);
    }

    trx_deregister_from_2pc(trx);
    trx_free_for_background(trx);

    return (ret != 0 ? XAER_RMERR : XA_OK);
  } else {
    return (XAER_NOTA);
  }
}
```

返回码遵循 **X/Open XA 规范**：`XA_OK` 成功、`XAER_RMERR` 资源管理器错误、`XAER_NOTA` xid 不存在。

对应的两个 set_prepared 入口：

```cpp
static int innobase_set_prepared_in_tc(handlerton *hton, THD *thd) {
  trx_t *trx = check_trx_exists(thd);
  thd_get_xid(thd, (MYSQL_XID *)trx->xid);
  ...
  dberr_t err = trx_set_prepared_in_tc_for_mysql(trx);
  ...
}

static xa_status_code innobase_set_prepared_in_tc_by_xid(handlerton *hton,
                                                         XID *xid) {
  trx_t *trx = trx_get_trx_by_xid(xid);

  if (trx != nullptr) {
    /* Side effect of retrieving the transaction is XID being set to null */
    *trx->xid = *xid;

    if (trx_is_prepared_in_tc(trx)) {
      return XA_OK;
    }
    ...
    dberr_t err = trx_set_prepared_in_tc_for_mysql(trx);
    return (err != DB_SUCCESS ? XAER_RMERR : XA_OK);
  } else {
    return (XAER_NOTA);
  }
}
```

两处易忽略的细节：

1. **`trx_get_trx_by_xid` 会把 `trx->xid` 置空**（注释明说 "Side effect of retrieving the transaction is XID being set to null"），所以 by_xid 路径必须显式 `*trx->xid = *xid` 补回。
2. **`trx_is_prepared_in_tc(trx)` 提前返回 `XA_OK`**——操作**幂等**，重复 `XA COMMIT` 同一 xid 不会出错。

#### 2PC 的失败处理

| 失败点 | 处理 |
|--------|------|
| prepare 失败（`ht->prepare` 返回错误） | `ha_commit_trans` 立即 `ha_rollback_trans` 回滚整个事务 |
| prepare 成功、commit 前崩溃 | 崩溃恢复按 XID 裁决（见上表） |
| binlog 写成功但引擎 commit 失败 | 返回 `RESULT_INCONSISTENT`——**已记 binlog 但未在引擎提交**，这是 2PC 的最坏情况，需人工介入 |
| GTID 与事务状态不一致 | 由 `gtid_executed` 与 binlog 的交叉校验兜底 |

---

### 事务与 BGC 协同

BGC（binlog group commit）的三阶段流水线细节见 `../server/replication/binlog.md`，这里只讲 **InnoDB 在每个阶段做了什么**。

#### flush stage：redo 批量落盘

```
ha_flush_logs(true)
  → innobase_flush_logs(hton, binlog_group_flush=true)
      → log_buffer_flush_to_disk(srv_flush_log_at_trx_commit == 1)
```

把 prepare 阶段因 `HA_IGNORE_DURABILITY` 而延迟的 redo **整组一次 fsync**。

关键参数交互：`innobase_flush_logs` 里有个提前返回——

```cpp
if (binlog_group_flush && srv_flush_log_at_trx_commit == 0) {
  /* innodb_flush_log_at_trx_commit=0
  (write and sync once per second).
  Do not flush the redo log during binlog group commit. */
  return false;
}
```

`innodb_flush_log_at_trx_commit = 0` 时，BGC 期间**不刷 redo**（每秒刷一次）。这意味着此时 prepare 的 redo 可能没落盘——崩溃会丢失事务，这是该参数语义的一部分。

#### sync stage：提交点

InnoDB 不参与。binlog fsync 成功即提交点。

#### commit stage：引擎提交

```
process_commit_stage_queue
  → ha_commit_low
      → innobase_commit
          → innobase_commit_low
              → trx_commit_for_mysql
                  → trx_commit_low → trx_commit_in_memory
```

**这里发生的是纯内存态收尾**——持久化（redo prepare + binlog fsync）在前面已完成，此时只是改 undo 状态、分配 `trx->no`、放锁、更新可见性。**没有需要保序的磁盘 I/O**，这正是 `binlog_order_commits=OFF` 时 commit stage 可以并发的原因。

#### trx->no 顺序 = 引擎提交顺序

`trx->no` 在 `trx_add_to_serialisation_list` 里、serialisation mutex 保护下按调用先后单调递增分配。**谁先被 commit stage 调用，谁的 `trx->no` 就更小**。

- `binlog_order_commits = ON`（默认）：leader 按 binlog 顺序串行调用 → `trx->no` 分配序 == binlog 写入序。
- `= OFF`：各线程自行调用 → 顺序不保证。

Clone、一致性备份、`START TRANSACTION WITH CONSISTENT SNAPSHOT` 依赖"`trx->no` 序 == binlog 序"来取一致快照，所以 `Clone_handler::need_commit_order()` 即使参数为 false 也强制有序。

---

### 2PC 与 BGC 中的锁

#### prepare 阶段

| 锁 | 保护对象 | 持有范围 |
|-----|---------|---------|
| rseg latch | 同一回滚段的 prepare 串行化（改 undo 段头状态字段） | `trx_prepare_low` 内 |
| `trx_sys->mutex` | `trx->state` 迁移、`n_prepared_trx` 计数 | `trx_prepare` 内状态切换 |
| MTR / redo latch | undo 页修改的 redo 写入 | `mtr_commit` 期间 |

`trx_prepare_low` 必须持 rseg latch——它要修改 undo 段头部的状态字段，同一 rseg 上的并发 prepare 必须串行。

#### 这把锁为何消失

8.0.39 中 `prepare_commit_mutex` 已不存在，只在 `trx_commit_in_memory` 的注释里留有历史痕迹：

```cpp
/* If we are calling trx_commit() under prepare_commit_mutex, we
will delay possible log write and flush to a separate function
trx_commit_complete_for_mysql(), which is only called when the
thread has released the mutex. This is to make the
group commit algorithm to work. */
```

它曾把 prepare→commit 全程串行化以保证 binlog 与 InnoDB 的顺序一致，**代价是彻底破坏 group commit**。5.7.6 起改由 `HA_IGNORE_DURABILITY` + BGC 三阶段保序，这把锁随之移除。三阶段演进的完整脉络见[「演进：从串行化到 BGC 协同」](#演进从串行化到-bgc-协同)。

**从锁视角看它的遗留物**：`trx_commit_complete_for_mysql()` 这个"延后刷日志"的函数正是为那把锁设计的——持锁期间不能刷盘（会让并发事务卡在等锁上刷盘），必须等释放锁后再刷。锁虽已移除，机制本身保留，BGC 场景下由 `trx->flush_log_later` 承担同样职责。这个"延后"设计是理解 prepare 与 flush stage 为何分离的关键。

#### commit 阶段

| 步骤 | 锁 | 保护对象 |
|------|-----|---------|
| `trx_write_serialisation_history` | rseg latch | history list 按 `trx->no` 顺序串联 |
| `trx_serialisation_number_get` | `purge_sys->pq_mutex` | purge 优先级队列 |
| `trx_add_to_serialisation_list` | `serialisation_mutex` | 分配 `trx->no`、操作 `serialisation_list` |
| `trx_release_impl_and_expl_locks` | `trx_sys->mutex` | 从 `rw_trx_ids` / `rw_trx_list` 摘除 |
| 同上 | `Trx_shard` latch + `trx->mutex` | 状态迁移 + 从 `active_rw_trxs` 移除（**必须原子**） |
| 同上 | `serialisation_mutex` | GTID 落盘 + 从 `serialisation_list` 移除 |
| `lock_trx_release_locks` | lock_sys latch | 释放全部行锁、表锁 |

**锁层次（避免死锁）**：

```
rseg latch
  → purge_sys->pq_mutex
    → trx_sys->serialisation_mutex
      → Trx_shard latch
        → trx->mutex
          → lock_sys latch（放锁，最后进行）
```

放锁（`lock_trx_release_locks`）在所有 trx 系统锁释放**之后**才做——避免持锁系统锁去等锁系统 latch 造成死锁。

#### 2PC 的锁代价

内部 2PC 让每个事务的提交多了一次 rseg latch 获取、一次 `trx_sys->mutex` 获取（prepare 时）。这是"binlog 与 redo 一致性"的代价。关 binlog 走 1PC 时这些开销全部省掉——这也是关 binlog 能提升写入吞吐的原因之一。

---

### 整体并发设计

#### trx_sys_t 的 cacheline 分区

`trx_sys_t`（trx0sys.h）用 `ut::INNODB_CACHE_LINE_SIZE` padding 把成员按"由哪把锁保护"分成若干区，注释直接标注：

```cpp
struct trx_sys_t {
  /* Members protected by neither trx_sys_t::mutex nor serialisation_mutex. */
  char pad0[ut::INNODB_CACHE_LINE_SIZE];
  MVCC *mvcc;
  Rsegs rsegs;
  Rsegs tmp_rsegs;
  std::atomic<uint64_t> rseg_history_len;

  /* Members protected by either trx_sys_t::mutex or serialisation_mutex. */
  char pad1[ut::INNODB_CACHE_LINE_SIZE];
  std::atomic<trx_id_t> next_trx_id_or_no;

  /* Members protected by serialisation_mutex. */
  char pad2[ut::INNODB_CACHE_LINE_SIZE];
  TrxSysMutex serialisation_mutex;
  UT_LIST_BASE_NODE_T(trx_t, no_list) serialisation_list;
  std::atomic<trx_id_t> serialisation_min_trx_no;

  /* Members protected by the trx_sys_t::mutex. */
  char pad4[ut::INNODB_CACHE_LINE_SIZE];
  TrxSysMutex mutex;
  UT_LIST_BASE_NODE_T(trx_t, trx_list) rw_trx_list;
  UT_LIST_BASE_NODE_T(trx_t, trx_list) mysql_trx_list;
  trx_ids_t rw_trx_ids;
  ...
  Trx_shard shards[TRX_SHARDS_N];
};
```

**设计思想**：**false sharing 消除 + 锁分区（lock striping）**。

- 每个热区之间插入一整条 cacheline 的 padding，保证不同锁保护的变量不会落在同一 cacheline，避免无谓的 cacheline ping-pong。
- 注释按"由哪个 mutex 保护"给成员分组，读者一眼能看出访问某个字段需要拿哪把锁——这是把并发契约写进代码结构的做法。

#### next_trx_id_or_no：一个计数器，两种用途

```cpp
/** The smallest number not yet assigned as a transaction id
or transaction number. ... When it is used for assignment of the trx->id,
it is synchronized by the trx_sys_t::mutex. When it is used
for assignment of the trx->no, it is synchronized by the
trx_sys_t::serialisation_mutex. Note: it might be in parallel
used for both trx->id and trx->no assignments (for different
trx_t objects). */
std::atomic<trx_id_t> next_trx_id_or_no;
```

**精妙之处**：`trx->id` 与 `trx->no` 共用同一个单调计数器，但由**两把不同的锁**分别保护：

- 分配 `trx->id` → `trx_sys->mutex`
- 分配 `trx->no` → `serialisation_mutex`

两者生命周期完全不重叠（id 在事务开始、no 在事务提交），因此可以安全共用而无需额外协调。声明为 `atomic` 是为了 AC-NL-RO 的 fast path 能**无锁读取**它做比较——正是前面 `view_open` 里的 `trx_sys_get_next_trx_id_or_no()`。

**收益**：省一个全局计数器；id 与 no 共享同一个单调递增空间，便于跨类型比较。

#### 256 分片：为什么是 trx_id % 256

```cpp
constexpr size_t TRX_SHARDS_N = 256;

inline size_t trx_get_shard_no(trx_id_t trx_id) {
  ut_ad(trx_id != 0);
  return trx_id % TRX_SHARDS_N;
}
```

5.7 中活跃读写事务存在全局 `rw_trx_list` / `rw_trx_ids`，所有注册/注销在一把 `trx_sys->mutex` 上竞争。8.0 改为 256 个 `Trx_shard`：

```cpp
struct Trx_shard {
  ut::Cacheline_padded<ut::Guarded<Trx_by_id_with_min, LATCH_ID_TRX_SYS_SHARD>>
      active_rw_trxs;
};
```

**为什么用取模而不是哈希**？因为要利用 trx_id **单调递增**的特性：

- `trx_id % 256` 分片后，**同一 shard 内的 trx_id 恰好相差 256 的倍数**（`shard_no, shard_no+256, shard_no+512, ...`）。
- 这让 `Trx_by_id_with_min` 能高效维护 `m_min_id`（该 shard 内最小活跃 trx_id）——内部 `unordered_map` 的哈希函数正是 `key / TRX_SHARDS_N`，与分片规则呼应。
- 判断"某个 trx_id 是否还可能活跃"时先比 `m_min_id`：若 `trx_id < m_min_id` 则必然已结束，**O(1) 短路返回，无需查 map**。

**收益**：全局锁竞争降到 1/256；单调性让最常见的"是否已结束"查询 O(1) 短路。

#### 并发访问总览与演进主线

| 操作 | 主要竞争点 | 8.0 的改进 |
|------|-----------|-----------|
| 事务启动（分配 id、注册） | `trx_sys->mutex` | `trx_sys_rw_trx_add` 在 mutex 外做分片注册 |
| 事务提交（分配 no、入 purge 队列） | `serialisation_mutex` + `pq_mutex` | rseg 非空时不 push purge_queue，降低 pq 竞争 |
| read view 创建 | `trx_sys->mutex` | AC-NL-RO fast path 完全无锁 |
| 状态迁移 + 隐式锁释放 | `Trx_shard` latch（256 个） | 替代全局 `trx_sys->mutex` |
| 放锁 | lock_sys | 8.0 sharded lock sys + 乐观 latching（`try_relatch_trx_and_shard_and_do`） |

**演进主线**：从 5.5 的 `kernel_mutex` 一把大锁 → 5.6 拆分独立 mutex → 8.0 的分片（Trx_shard 256、lock_sys shards）+ 无锁快路径（AC-NL-RO view 复用）+ 原子变量（`next_trx_id_or_no`、`trx->state`、`rseg_history_len`）。每一步都在把串行点**打散**（分片）或**彻底消除**（无锁路径）。

---

### 回滚机制

#### 触发场景

| 场景 | 入口 | 执行线程 |
|------|------|----------|
| 显式 `ROLLBACK` / `XA ROLLBACK` | `ha_rollback_trans` → `innobase_rollback` | 用户线程（前台） |
| 语句错误回滚 | `innobase_rollback` 的 `rollback_trx=false` 分支 | 用户线程 |
| KILL QUERY / CONNECTION | 检查点发现 killed → `ha_rollback_trans` | 用户线程 |
| 高优先级事务杀阻塞者 | `trx_kill_blocking` → `trx_rollback_for_mysql` | killer 线程（异步） |
| 崩溃恢复 | `trx_recovery_rollback_thread` → `trx_rollback_active` | 后台线程 |

**崩溃恢复这条路径与正常回滚最大的不同**，藏在 `trx_recovery_rollback`（trx0roll.cc:784）——它在真正回滚之前，先做了一整套"加锁 + 复活"：

```cpp
void trx_recovery_rollback(THD *thd) {
  std::vector<MDL_ticket *> shared_mdl_list;
  ut_ad(!srv_read_only_mode);

  // Take MDL locks
  /* During this stage the server is not open for external connections, and
   * there will not be any concurrent threads requesting MDL, hence we don't
   * have risk of hitting a deadlock.
   * TODO: We can improvize code by following some order while taking MDL locks. */
  for (const auto &element : to_rollback_trx_tables) {
    trx_id_t trx_id = element.first;
    table_id_t table_id = element.second;

    /* Passing false as we only intend to validate if the transaction is
     * already committed/rolled back during other stages of recovery. */
    auto trx = trx_rw_is_active(trx_id, false);
    if (trx == nullptr) { ut_ad(false); continue; }      // 恢复期不该已结束

    auto table = dd_table_open_on_id(table_id, nullptr, nullptr, false, true);
    if (table == nullptr) continue;                      // 表已被删 → 跳过

    table->get_table_name(schema_name, table_name);
    MDL_ticket *mdl_ticket;
    if (dd_mdl_acquire(thd, &mdl_ticket, schema_name.data(), table_name.data())) {
      ut_error;                                          // ★ 恢复期拿 MDL 失败 = 致命
    }
    shared_mdl_list.push_back(mdl_ticket);

    lock_table_ix_resurrect(table, trx);                 // ★ 复活表级 IX 锁
    ib::info(ER_IB_RESURRECT_ACQUIRE_TABLE_LOCK, ulong(table->id), table->name.m_name);
    dd_table_close(table, nullptr, nullptr, false);
  }

  /* Let the startup thread proceed now */
  os_event_set(recovery_lock_taken);                     // ★ 放行启动线程

  trx_rollback_or_clean_recovered(true);                 // 真正回滚

  for (auto mdl_ticket : shared_mdl_list) dd_release_mdl(mdl_ticket);
}
```

四点决定了"为什么恢复期回滚要这么绕"：

1. ★ **崩溃前持有的表锁会丢失，必须"复活"**：`lock_table_ix_resurrect(table, trx)` 重新为这些待回滚事务加上**表级 IX 锁**。没有这一步，恢复期间的回滚操作与后续进入的 DML/DDL 就少了互斥依据。日志里对应 `ER_IB_RESURRECT_ACQUIRE_TABLE_LOCK`（"resurrect" 一词即由此而来）。
2. ★ **必须先拿 MDL 才放行启动**：`os_event_set(recovery_lock_taken)` 是一个**同步点**——所有表的 MDL 拿到之前，启动线程一直等在这里。否则可能出现"表还没加锁就被外部访问"。
3. ★ **此阶段不会死锁**：源码注释给了理由——*"the server is not open for external connections, and there will not be any concurrent threads requesting MDL"*。所以这里可以放心地无序批量加锁（注释还留了 TODO：未来可以按固定顺序加锁以进一步改进）。反过来说，**这条"无死锁"的前提一旦不成立（比如以后允许恢复期接受连接），这段代码就要重写**。
4. **`dd_mdl_acquire` 失败是 `ut_error`（直接崩溃）而非跳过**：恢复期拿不到 MDL 意味着无法保证互斥，继续跑会破坏数据，所以选择立即失败；而表已被删（`dd_table_open_on_id` 返回 nullptr）则只是 `continue` 跳过——那张表上已没有什么可回滚的。

⇒ 于是完整顺序是：**复活表锁 + 拿 MDL → 放行启动 → 回滚 → 释放 MDL**。这也是为什么启动日志里会先看到一批 "resurrect" 信息，然后才继续启动流程。

#### 执行路径

```
ha_rollback_trans
  → innobase_rollback
      → trx_rollback_for_mysql
          → trx_rollback_low
              → trx_rollback_to_savepoint(trx, nullptr)    nullptr = 完整回滚
                  → trx_rollback_to_savepoint_low
                      → que_run_threads → row_undo_step     逐行逆序撤销
```

`trx_rollback_to_savepoint_low` 构造两层 query graph，由 `row_undo_step` 逐行逆序执行 undo：插入行→删除，更新行→恢复旧值，删除行→重新插入。两层结构见下文「回滚与查询图」。

#### 回滚的逐行剖析

`trx_rollback_low` 是回滚真正的分发入口，按状态四种处理：

```cpp
static dberr_t trx_rollback_low(trx_t *trx) {
  /* We are reading trx->state without mutex protection here,
  because the rollback should either be invoked for:
    - a running active MySQL transaction associated
      with the current thread,
    - or a recovered prepared transaction,
    - or a transaction which is a victim being killed by HP transaction
      run by the current thread, in which case it is guaranteed that
      thread owning the transaction, which is being killed, is not
      inside InnoDB (thanks to TRX_FORCE_ROLLBACK and TrxInInnoDB::wait()). */

  switch (trx->state.load(std::memory_order_relaxed)) {
    case TRX_STATE_FORCED_ROLLBACK:
    case TRX_STATE_NOT_STARTED:
      trx->will_lock = 0;              // 回滚已经做过了，只清 will_lock
      return (DB_SUCCESS);

    case TRX_STATE_ACTIVE:
      /* Check an validate that undo is available for GTID. */
      // 确保 GTID 有 undo 可写（回滚时要把 GTID 的回滚信息也记进 undo）
      trx_undo_gtid_add_update_undo(trx, false, true);
      return (trx_rollback_for_mysql_low(trx));

    case TRX_STATE_PREPARED:
      trx_undo_gtid_add_update_undo(trx, false, true);
      if (trx->rsegs.m_redo.rseg != nullptr && trx_is_redo_rseg_updated(trx)) {
        /* Change the undo log state back from
        TRX_UNDO_PREPARED to TRX_UNDO_ACTIVE
        so that if the system gets killed,
        recovery will perform the rollback. */
        mtr_t mtr;
        mtr.start();
        trx->rsegs.m_redo.rseg->latch();

        // ★ WAL 协议第 1 步：把 undo 段状态从 TRX_UNDO_PREPARED 改回 TRX_UNDO_ACTIVE
        //   （第三个参数 true 表示"从 PREPARED 变回 ACTIVE"，对比 prepare 时的 false）
        if (undo_ptr->insert_undo != nullptr) {
          trx_undo_set_state_at_prepare(trx, undo_ptr->insert_undo, true, &mtr);
        }
        if (undo_ptr->update_undo != nullptr) {
          trx_undo_gtid_set(trx, undo_ptr->update_undo, false);
          trx_undo_set_state_at_prepare(trx, undo_ptr->update_undo, true, &mtr);
        }
        trx->rsegs.m_redo.rseg->unlatch();

        /* Persist the XA ROLLBACK, so that crash
        recovery will replay the rollback in case
        the redo log gets applied past this point. */
        // ★ WAL 协议第 2 步：立即持久化这个状态变更，然后才执行真正的回滚
        mtr.commit();
      }
      // ★ WAL 协议第 3 步：真正的回滚
      return (trx_rollback_for_mysql_low(trx));

    case TRX_STATE_COMMITTED_IN_MEMORY:
      check_trx_state(trx);            // 走到这里是逻辑错误（已提交不该再回滚），断言兜底
      break;
  }

  ut_error;
}
```

**★ 为什么要先改状态再回滚**（PREPARED 分支的 WAL 协议）：回滚可能很长，中途崩溃——若状态仍是 PREPARED，恢复时会被当"等待裁决的 prepared"搁置（不处理）；若已改回 ACTIVE 并落盘，恢复时看到 ACTIVE 就**继续回滚**。⇒ 回滚动作本身被做成 WAL 可重放的。

`trx_rollback_step` 是回滚 query graph 的 step 函数，驱动一个两态状态机：

```cpp
que_thr_t *trx_rollback_step(que_thr_t *thr) {
  roll_node_t *node = static_cast<roll_node_t *>(thr->run_node);

  if (thr->prev_node == que_node_get_parent(node)) {
    node->state = ROLL_NODE_SEND;    // 重新进入则重置状态
  }

  // ★ 两态状态机：SEND 做初始化并启动 undo，WAIT 表示已跑完
  if (node->state == ROLL_NODE_SEND) {
    trx_t *trx = thr_get_trx(thr);

    trx_mutex_enter(trx);
    node->state = ROLL_NODE_WAIT;

    // savepoint 回滚的边界是 savepoint 记录的 undo_no；完整回滚是 0（回到事务起点）
    roll_limit = node->partial ? node->savept.least_undo_no : 0;

    trx_commit_or_rollback_prepare(trx);   // 处理"NOT_STARTED 需先 start"等边界

    // 设 trx->roll_limit、in_rollback = true，然后 trx_roll_graph_build 构建回滚
    // query graph、que_state = TRX_QUE_ROLLING_BACK，返回首个 thr 开始执行
    node->undo_thr = trx_rollback_start(trx, roll_limit, node->partial);

    trx_mutex_exit(trx);
  } else {
    ut_ad(node->state == ROLL_NODE_WAIT);
    thr->run_node = que_node_get_parent(node);   // undo_thr 执行完，回到父节点，回滚结束
  }

  return (thr);
}
```

★ **最反直觉的一行——回滚的收尾是"提交"**：

```cpp
static void trx_rollback_finish(trx_t *trx) {
  trx_commit(trx);

  trx->mod_tables.clear();

  trx->lock.que_state = TRX_QUE_RUNNING;
}
```

回滚完成后的清理动作竟然是 `trx_commit(trx)`。**为什么**？因为回滚本身也产生了 redo 与 undo（把数据行改回旧值、把 undo 段状态改掉），这些修改需要一个"提交"来**清理 undo 段资源、归还 rseg、推进事务状态**。这里的 `trx_commit` 提交的不是业务数据，而是"回滚动作本身"。

#### 回滚与查询图：两层 graph 的关联

回滚不是普通函数调用，而是**构造并解释执行两层 query graph**——这正是 InnoDB query graph 执行模型（详见 [`query_graph.md`](query_graph.md)）在回滚上的应用。

```
外层 fork（QUE_FORK_MYSQL_INTERFACE，前台回滚）
└── roll_node_t（QUE_NODE_ROLLBACK）          ← 回滚"命令"节点
    └── undo_thr ──> 内层 fork（QUE_FORK_ROLLBACK）
        └── undo_node（QUE_NODE_UNDO）         ← 真正逐行撤销的执行体
            └── row_undo_step                  ← undo 节点的 step 函数
```

**外层 roll_node**：`roll_node_t`（trx0roll.h:156）是一个 `QUE_NODE_ROLLBACK` 节点，由 `trx_rollback_step` 驱动，状态机 `ROLL_NODE_SEND → ROLL_NODE_WAIT`。职责是"准备 + 启动 + 收尾"，真正的撤销委托给内层。它的 `undo_thr` 字段指向内层 graph。

**内层 undo graph**：`trx_roll_graph_build`（trx0roll.cc:1110）创建，fork type 是 `QUE_FORK_ROLLBACK`：

```cpp
static que_t *trx_roll_graph_build(trx_t *trx, bool partial_rollback) {
  fork = que_fork_create(nullptr, nullptr, QUE_FORK_ROLLBACK, heap);
  fork->trx = trx;

  thr = que_thr_create(fork, heap, nullptr);

  thr->child = row_undo_node_create(trx, thr, heap, partial_rollback);

  return (fork);
}
```

`row_undo_node_create`（row0undo.cc:129）创建 `QUE_NODE_UNDO` 节点挂到 thr 下，这就是逐行撤销的执行体。它的一步初始化里已经定好了状态机初态：

```cpp
  undo->common.type = QUE_NODE_UNDO;
  undo->common.parent = parent;
  undo->state = UNDO_NODE_FETCH_NEXT;      // ★ 初态：先去取下一条 undo 记录
  undo->trx = trx;
  undo->partial = partial_rollback;        // ★ 部分回滚标记（ROLLBACK TO SAVEPOINT 复用同一张图）
  undo->pcur.init();                       // 持久游标（撤销 update 时要定位聚簇记录）
  undo->heap = mem_heap_create(256, UT_LOCATION_HERE);
```

★ **往下拆一步才是回滚的真正骨架**——`row_undo_step`（row0undo.cc:255 起）用一个三态状态机驱动整条撤销链：

```cpp
  if (node->state == UNDO_NODE_FETCH_NEXT) {
    node->undo_rec = trx_roll_pop_top_rec_of_trx(trx, trx->roll_limit,
                                                 &roll_ptr, node->heap);   // ① 弹栈顶

    if (!node->undo_rec) {
      /* Rollback completed for this query thread */
      thr->run_node = que_node_get_parent(node);   // 回到父节点 = 结束
      /* Mark any partial rollback completed, so that if the transaction
      object is committed and reused later, the roll_limit will remain
      at 0. trx->roll_limit will be nonzero during a partial rollback only. */
      trx->roll_limit = 0;
      ut_d(trx->in_rollback = false);
      return (DB_SUCCESS);
    }

    node->roll_ptr = roll_ptr;
    node->undo_no = trx_undo_rec_get_undo_no(node->undo_rec);

    if (trx_undo_roll_ptr_is_insert(roll_ptr)) {  // ② 按 undo 种类分派
      node->state = UNDO_NODE_INSERT;
    } else {
      node->state = UNDO_NODE_MODIFY;
    }
  }

  if (node->state == UNDO_NODE_INSERT) {
    err = row_undo_ins(node, thr);
    node->state = UNDO_NODE_FETCH_NEXT;            // ③ 立刻回到取下一条
  } else {
    ut_ad(node->state == UNDO_NODE_MODIFY);
    err = row_undo_mod(node, thr);                 // ④ ★ 不立刻回
  }
```

状态只有三个（`row0undo.h`）：`UNDO_NODE_FETCH_NEXT` → `UNDO_NODE_INSERT` / `UNDO_NODE_MODIFY`。四个点决定了对错：

1. **LIFO**：`trx_roll_pop_top_rec_of_trx` 弹的是**栈顶**——即最后做的修改最先撤销。这是事务回滚"逆序"语义的直接实现。
2. ★ **`roll_limit` 是部分回滚的边界**：它作为参数传给 pop，只弹到该 undo_no 为止；`ROLLBACK TO SAVEPOINT` 与全回滚**共用同一张图、同一个状态机**，区别只在 `roll_limit`（0 = 全回滚）和 `node->partial`。弹完后 `trx->roll_limit = 0` 必须复位——注释说得很清楚：为的是 trx 对象提交后被**复用**时 roll_limit 保持 0。
3. **`is_insert` 决定撤销方式**：insert 产生的 undo 只需把行删掉（`row_undo_ins`），update/delete 的 undo 要恢复旧版本（`row_undo_mod`），两者的代价完全不同（后者还要处理二级索引、BLOB、可能的 rebuild）。
4. ★ **状态回环是不对称的**：`UNDO_NODE_INSERT` 撤销完**立即**回到 `FETCH_NEXT`；而 `UNDO_NODE_MODIFY` **不立即回**——因为 `row_undo_mod` 可能一次做不完（要逐个二级索引地撤销，中途要让出），状态机允许它停在 MODIFY 继续。这就是"一次 rollback 可能跨多次 step 调用"的原因。

另外这段的注释还交代了一条并发前提：回滚期间 trx 至少持有各被修改表的 `LOCK_IX`（可能还有 MDL），因此**并发 DROP/ALTER 不可能**——那需要同时拿到 `LOCK_X` 与 `MDL_EXCLUSIVE`。

**fork type 的区分**（que0que.h 的注释是关键证据）：

```cpp
constexpr uint32_t QUE_FORK_ROLLBACK = 5;
/* This is really the undo graph used in rollback,
no signal-sending roll_node in this graph */
```

注释直接说明：`QUE_FORK_ROLLBACK` 是"回滚用的 undo graph，**这个 graph 里没有发信号的 roll_node**"——roll_node 在外层。

三种 fork type 参与回滚，对应三种场景：

| fork type | 值 | 场景 | 谁创建 |
|-----------|-----|------|--------|
| `QUE_FORK_MYSQL_INTERFACE` | 10 | 前台回滚的外层 roll_node | `pars_complete_graph_for_exec` |
| `QUE_FORK_ROLLBACK` | 5 | 内层 undo graph（所有回滚共用） | `trx_roll_graph_build` |
| `QUE_FORK_RECOVERY` | 11 | 崩溃恢复回滚的外层 | `trx_rollback_active` |

**与 DML query graph 的异同**：

| | DML（INSERT/UPDATE/DELETE） | 回滚 |
|---|---|---|
| fork / thr / node 三层架构 | 是 | 是 |
| 由 `que_run_threads` 解释执行 | 是 | 是 |
| node 有 step 函数 | 是（`row_ins_step` / `row_upd_step` / `row_sel_step`） | 是（`trx_rollback_step` / `row_undo_step`） |
| node 类型 | ins_node / upd_node / sel_node | roll_node + undo_node |
| graph 构建时机 | **预编译缓存**（`prebuilt->ins_graph` / `upd_graph`） | **动态构建**（每次回滚现建现删） |
| 锁等待挂起 | 可能（DML 加锁冲突） | 回滚不加锁，不挂起 |

**关键差异**：DML 的 graph 缓存在 `prebuilt` 里复用（避免每次重建），而回滚的 graph 是**每次现建现删**——回滚频率低，且 roll_node 的状态（`savept`、`roll_limit`）每次都不同，没有复用价值。

**设计收益**：正向操作（DML）和逆向操作（回滚）被统一在"编译成 graph → 解释执行"的模型下，只是 node 类型和 fork type 不同。回滚不需要单独写一套执行引擎，复用 `que_run_threads` 即可。这是 query graph 模型超越"普通函数调用"的价值所在——它能承载任意有状态、可挂起/恢复的操作。

#### SAVEPOINT 与语句级回滚

⚠️ **勘误**：**不存在 `trx_rollback_last()`**（真名 `trx_rollback_last_sql_stat_for_mysql()`）、**不存在 `trx_rollback()`**（是 `trx_rollback_for_mysql()` → `trx_rollback_low()` → `trx_rollback_for_mysql_low()` → `trx_rollback_to_savepoint_low()`）。

**savepoint 只有一个字段**（`trx0roll.cc:563`）：

```cpp
trx_savept_t trx_savept_take(trx_t *trx) {
  trx_savept_t savept;
  savept.least_undo_no = trx->undo_no;   // ★★ 就这一个 undo_no
  return (savept);
}
```

**语句级回滚**（`trx0roll.cc:282`）：

```cpp
dberr_t trx_rollback_last_sql_stat_for_mysql(trx_t *trx) {
  switch (trx->state.load(std::memory_order_relaxed)) {
    case TRX_STATE_FORCED_ROLLBACK:
    case TRX_STATE_NOT_STARTED:
      return (DB_SUCCESS);              // ★ 没启动就没什么可回滚

    case TRX_STATE_ACTIVE:
      trx->op_info = "rollback of SQL statement";
      err = trx_rollback_to_savepoint(trx, &trx->last_sql_stat_start);
      //                          ★★★ 语句级 = 回滚到"上一条语句起点"这个隐式 savepoint
      if (trx->fts_trx != nullptr) fts_savepoint_rollback_last_stmt(trx);
      trx_mark_sql_stat_end(trx);
      return (err);

    case TRX_STATE_PREPARED:
    case TRX_STATE_COMMITTED_IN_MEMORY:
      /* The statement rollback is only allowed on an ACTIVE transaction */
      break;
  }
  ut_error;
}
```

**`last_sql_stat_start` 在哪被刷新**（`trx_mark_sql_stat_end()`，`trx0trx.cc:2551`）——⚠️ **不存在 `trx_release_locks_at_end_of_statement()`**：

```cpp
void trx_mark_sql_stat_end(trx_t *trx) {
  lock_on_statement_end(trx);
  //  ★ 语句结束唯一与锁相关的动作（见下一行），★ 不释放任何锁

  switch (trx->state.load(std::memory_order_relaxed)) {
    case TRX_STATE_NOT_STARTED:
    case TRX_STATE_FORCED_ROLLBACK:
      trx->undo_no = 0;
      trx->undo_rseg_space = 0;
      [[fallthrough]];
    case TRX_STATE_ACTIVE:
      trx->last_sql_stat_start.least_undo_no = trx->undo_no;
      //  ★★★ 这就是下一次语句级回滚的 savepoint
      return;
  }
}
```

```cpp
// lock0lock.cc:2549 —— 就一行！
void lock_on_statement_end(trx_t *trx) { trx->lock.inherit_all.store(false); }
//   ★ 只是"停止 GAP 锁继承"，真正的锁释放在事务提交/回滚时
```

**回滚的两段式执行**（`trx_rollback_to_savepoint_low()`，`trx0roll.cc:78`）：

```cpp
  roll_node = roll_node_create(heap);          // ★ 构建 QUE_NODE_ROLLBACK 节点

  if (savept != nullptr) {
    roll_node->partial = true;                 // ★ 部分回滚：带 savepoint
    roll_node->savept = *savept;
  }

  if (trx_is_rseg_updated(trx)) {              // ★ 没有 undo 就一行都不用回
    thr = pars_complete_graph_for_exec(roll_node, trx, heap, nullptr);
    ut_a(thr == que_fork_start_command(
                    static_cast<que_fork_t *>(que_node_get_parent(thr))));

    que_run_threads(thr);                      // ★ 第 1 段：建 undo graph（trx_rollback_step）
    ut_a(roll_node->undo_thr != nullptr);
    que_run_threads(roll_node->undo_thr);      // ★★ 第 2 段：跑真正的 row_undo graph
    que_graph_free(static_cast<que_t *>(roll_node->undo_thr->common.parent));
  }

  if (savept == nullptr) {
    trx_rollback_finish(trx);                  // ★ 全量回滚才 trx_commit() 收尾（释放锁、改 state）
  } else {
    trx->lock.que_state = TRX_QUE_RUNNING;     // ★ 部分回滚：不提交、不释放锁
  }
```

**回滚前必须先解除锁等待**（`trx_commit_or_rollback_prepare()`，`trx0trx.cc:2372`）：

```cpp
      /* If the trx is in a lock wait state, moves the waiting
      query thread to the suspended state */
      if (trx->lock.que_state == TRX_QUE_LOCK_WAIT) {
        ut_a(trx->lock.wait_thr != nullptr);
        trx->lock.wait_thr->state = QUE_THR_SUSPENDED;   // ★ 挂起卡在锁等待的 query thread
        trx->lock.wait_thr = nullptr;
        trx->lock.que_state = TRX_QUE_RUNNING;           // ★★ 否则回滚无法推进
      }
```

**回滚的"双栈合并"**（`trx_roll_pop_top_rec_of_trx_low()`，`trx0roll.cc:905`）：

```cpp
  ins_undo = undo_ptr->insert_undo;
  upd_undo = undo_ptr->update_undo;

  if (!ins_undo || ins_undo->empty) {
    undo = upd_undo;
  } else if (!upd_undo || upd_undo->empty) {
    undo = ins_undo;
  } else if (upd_undo->top_undo_no > ins_undo->top_undo_no) {
    undo = upd_undo;      // ★★ insert/update 两个 undo 合成"一个按 undo_no 排序的栈"
  } else {                //    谁 top_undo_no 大先弹谁
    undo = ins_undo;
  }

  if (!undo || undo->empty || limit > undo->top_undo_no) {
    //  ★★ 触到 roll_limit（部分回滚的下界）→ 停止
    trx_roll_try_truncate(trx, undo_ptr);
    return (nullptr);
  }
```

> ★ 边回滚边 truncate：`if (trx->pages_undone >= TRX_ROLL_TRUNC_THRESHOLD) trx_roll_try_truncate(...)`,`TRX_ROLL_TRUNC_THRESHOLD = 1`——每跨页就试一次，防止 undo 空间暴涨。
>
> ★ 崩溃恢复时的进度打印（`Progress in percents: ...`）就在这个函数里，条件是 `trx == trx_roll_crash_recv_trx && trx_roll_max_undo_no > 1000`。

savepoint 是"回滚"的轻量版——把事务回退到中间某点，而非起点。它的核心数据结构极简，`trx_savept_t` 只记一个数：

```cpp
struct trx_savept_t {
  undo_no_t least_undo_no; /*!< least undo number to undo */
};
```

**为什么一个 undo_no 就够**：`trx->undo_no` 是事务内单调递增的 undo 记录编号。事务的两条 undo 链（insert_undo / update_undo）按 `top_undo_no` 合成一个逻辑栈；savepoint 之后的记录编号必然大于 savepoint 时刻的 `trx->undo_no`。所以记录一个编号就足以标记回滚边界——回滚时弹出所有 `top_undo_no > least_undo_no` 的记录。

**server 层与引擎层各存各的游标**：

| 层 | 存什么 | 作用 |
|----|--------|------|
| server 层 `SAVEPOINT` 结构 | 名字、引擎列表、`mdl_savepoint`、binlog cache 字节位置 | 名字查找、MDL 回滚、binlog 截断 |
| InnoDB `trx_named_savept_t` | 名字、`savept`（undo_no）、`mysql_binlog_cache_pos` | undo 回滚边界 |

**为什么两个游标缺一不可**：回滚要同时恢复两套独立状态机，它们之间没有任何换算关系——undo_no 管"数据页修改的撤销边界"（InnoDB 内部编号），binlog cache pos 管"复制日志的字节流截断边界"（server 层字节流）。undo_no 与字节偏移没有映射，必须各自独立记录、分别恢复，否则数据侧回滚了而 binlog 里还留着这些语句，从库重放出不一致数据。

**InnoDB 的一个巧妙实现**：`innobase_savepoint` 不用 server 层给的 `SAVEPOINT` 结构存数据（注释 "TODO: use provided savepoint data area"），而是把 **savepoint 指针本身**用 `longlong2str((ulint)savepoint, name, 36)` 转成 36 进制字符串当名字，之后回滚/释放时按这个名字查——同一进程内指针唯一，免去额外映射表。

**roll_limit 的语义**（`trx_rollback_step`）：

```cpp
roll_limit = node->partial ? node->savept.least_undo_no : 0;
```

- `partial = true`（回滚到 savepoint）→ `roll_limit = least_undo_no`，只回滚编号更大的记录；
- `partial = false`（完整回滚）→ `roll_limit = 0`，回滚到事务起点。

边界判定在 `trx_roll_pop_top_rec_of_trx_low`：两个 undo 栈（insert/update）按 `top_undo_no` 大者优先弹出，栈顶编号 <= limit 时停止。

**语句级隐式 savepoint**：每条语句的边界就是一个隐式 savepoint。`trx_mark_sql_stat_end` 在语句结束时把 `trx->last_sql_stat_start.least_undo_no = trx->undo_no`（推进边界）；语句失败时 `trx_rollback_last_sql_stat_for_mysql` 以 `last_sql_stat_start` 为 savept 调 `trx_rollback_to_savepoint`，只撤销本语句的修改，然后重新锚定边界。**语句级回滚完全复用 partial rollback 机制**，没有第二条实现路径。

**binlog cache 的处理**（`binlog_savepoint_rollback`）：纯事务表 → `trx_cache.restore_savepoint(pos)` 直接截断 cache 到 savepoint 位置；动过非事务表 → 不能截断，补写一条 "ROLLBACK TO name" 语句让从库重放。

> ★ 8.0.39 现状：`trx_named_savept_t::mysql_binlog_cache_pos` 是历史接口残留——InnoDB 侧 `trx_savepoint_for_mysql` 传的是 0，binlog cache 位置实际由 binlog 引擎存在 server 层 `SAVEPOINT` 的私有区里，截断由 `binlog_savepoint_rollback` 自己完成。

#### 回滚不可被 kill 中断

**结论：回滚一旦开始，不能被 kill 中断。** kill 只能在回滚开始前生效（把事务"逼进"回滚）。

三条源码证据：

1. **回滚循环无 kill 检查**：`trx_rollback_to_savepoint_low` 的 `que_run_threads` 循环里没有 `trx_is_interrupted`；真正逐行撤销的 `row0undo.cc` 中搜 `killed` 结果为 0 处。
2. **kill 检查只出现在正向执行路径**：`trx_is_interrupted` 本质是 `thd_killed(trx->mysql_thd)`，遍布 DML 检查点（如 `row_search_mvcc`、`btr_cur_...`），但回滚路径一次都不出现。
3. **point of no return**：`TrxInInnoDB::enter(disable=true)` 设置 `TRX_FORCE_ROLLBACK_DISABLE`，注释："This transaction has crossed the point of no return and cannot be rolled back asynchronously now. It must commit or rollback synchronously."

**设计原因**：回滚必须完整做完，否则 undo 处于半撤销状态，崩溃恢复无法重建一致性。

#### KILL 的作用时机

**KILL 的全部"本地"效果只是置一个标志位**（`THD::awake()`，`sql/sql_class.cc:1489`）：

```cpp
  if (this->m_server_idle && state_to_set == KILL_QUERY) { /* nothing */
  } else {
    killed = state_to_set;      // ★★ 就这一行
  }
  ...
  /* Interrupt target waiting inside a storage engine. */
  if (state_to_set != THD::NOT_KILLED) ha_kill_connection(this);
  //                                    ★★ 唯一进入存储引擎的钩子
  ...
  if (is_killable) {
    mysql_mutex_lock(&LOCK_current_cond);
    if (current_cond.load() && current_mutex.load()) {
      mysql_mutex_lock(current_mutex);
      mysql_cond_broadcast(current_cond);   // ★ 唤醒正在 cond_wait 的 victim
      mysql_mutex_unlock(current_mutex);
    }
    mysql_mutex_unlock(&LOCK_current_cond);
  }
```

**InnoDB 端只做一件事**（`ha_innodb.cc:6227`）：

```cpp
/** Cancel any pending lock request associated with the current THD. */
static void innobase_kill_connection(handlerton *hton, THD *thd) {
  trx_t *trx = thd_to_trx(thd);
  if (trx != nullptr) {
    /* Cancel a pending lock request if there are any */
    lock_cancel_if_waiting_and_release({trx});   // ★★ 只取消正在等待的锁请求
  }
}
```

> ★★ **为什么 KILL 只在锁等待时可中断**：`THD::awake()` 不打断任何执行流。若事务正在跑 row 操作（不在锁等待），没有任何强制抢占机制——只能等它走到下一个 `thd->killed` 检查点。
>
> ⚠️ 还有一条"免疫"路径：`if (kill_immunizer && kill_immunizer->is_active())` —— **DD 操作期间 KILL 被攒着**，等退出免疫区再补发。

**另一条路：高优先级事务（CATs）代跑回滚**（`trx_kill_blocking()`，`trx0trx.cc:3522`）：

```cpp
void trx_kill_blocking(trx_t *trx) {
  if (!trx_is_high_priority(trx)) return;   // ★ 只有 CATs 高优先级事务才有"杀别人"的权利

  hit_list_t hit_list;
  lock_make_trx_hit_list(trx, hit_list);    // ★ 按锁获取顺序 old→new 造受害者名单
  ...
  for (hit_list_t::reverse_iterator it = hit_list.rbegin(); it != end; ++it) {
    trx_t *victim_trx = it->m_trx;

    if (lock_cancel_if_waiting_and_release(*it)) continue;   // ① 正在锁等待 → 直接取消

    /* Check that the transaction isn't active inside InnoDB code. We have to
    wait while it is executing in the InnoDB context. */
    while ((victim_trx->in_innodb & TRX_FORCE_ROLLBACK_MASK) > 0 &&
           victim_trx->version == version) {
      //  ★★ 自旋等 victim 退出 InnoDB 临界区（in_innodb==0）
      //     退避：20us → 1ms → 100ms
      std::this_thread::sleep_for(std::chrono::microseconds(sleep_time));
    }
    ...
    ut_ad(victim_trx->in_innodb & TRX_FORCE_ROLLBACK);       // victim 已打异步回滚标记
    ut_ad(victim_trx->killed_by == std::this_thread::get_id());
    ...
    trx_rollback_for_mysql(victim_trx);
    //  ★★★ killer 线程替 victim 跑完整回滚
    victim_trx->killed_by.store(std::thread::id{});          // 清 killer 标记
    victim_trx->in_innodb &= TRX_FORCE_ROLLBACK_MASK;        // 清 bit31 / bit29
  }
}
```

> ★★ 这是"KILL 能立刻生效"的唯一场景：**不是被打断，而是被另一个线程代跑完回滚**。等 victim 回到自己的执行流时，会发现事务已经是 `TRX_STATE_FORCED_ROLLBACK`。

| 时机 | 能否生效 | 结果 |
|------|---------|------|
| 执行 DML 中 | ✅ | 下一个 `trx_is_interrupted` 检查点中断 → 触发回滚 |
| 等待锁中 | ✅ | `lock_cancel_if_waiting_and_release` 取消等待 → 看到 killed → 回滚 |
| **回滚中** | ❌ | 只设 killed 标志，回滚照常跑完 |

`innobase_kill_connection` 的实现印证了这点——它只做一件事：

```cpp
static void innobase_kill_connection(handlerton *hton, THD *thd) {
  trx_t *trx = thd_to_trx(thd);

  if (trx != nullptr) {
    /* Cancel a pending lock request if there are any */
    lock_cancel_if_waiting_and_release({trx});
  }
}
```

只取消锁等待，**不做回滚、也不能打断回滚**。

生产现象：`KILL CONNECTION` 后连接还"挂着"一段时间，`SHOW PROCESSLIST` 的 State 显示 `ROLLBACK`，那正是在做不可中断的回滚。

#### 异步回滚（TRX_FORCE_ROLLBACK）

与 KILL 是两套独立机制。高优先级事务（`trx_is_high_priority`）主动杀死阻塞自己的普通事务：

```
trx_kill_blocking
  → lock_make_trx_hit_list                     收集阻塞者
      → for each victim:
          ├─ lock_mark_trx_for_rollback        设 TRX_FORCE_ROLLBACK + killed_by
          ├─ 等待 victim 退出 InnoDB 上下文
          └─ trx_rollback_for_mysql(victim_trx)  killer 线程代其回滚
```

`trx_rollback_for_mysql` 里 `TrxInInnoDB::is_async_rollback(trx)` 判断 `killed_by == 当前线程`，跳过 InnoDB 上下文追踪。

#### 澄清：没有"异步提交"，只有"异步回滚"

**两个标志位**（`trx0types.h:60`）：

```cpp
/** If this flag is set then the transaction cannot be rolled back
asynchronously. */
static const uint32_t TRX_FORCE_ROLLBACK_DISABLE = 1 << 29;   // ★ bit29：已进入 COMMIT/ROLLBACK，禁止再被异步回滚

/** Mark the transaction for forced rollback */
static const uint32_t TRX_FORCE_ROLLBACK = 1U << 31;          // ★ bit31：已被标记强制(异步)回滚

/** For masking out the above flags */
static const uint32_t TRX_FORCE_ROLLBACK_MASK = 0x1FFFFFFF;   // ★ 清掉 bit29/bit31，只留 in_innodb 计数
```

**★ "没有异步提交"的实现处**（`TrxInInnoDB::enter()`，`trx0trx.h:1495`）：

```cpp
  static void enter(trx_t *trx, bool disable) {
    /* If it hasn't already been marked for async rollback.
    and it will be committed/rolled back. */
    if (disable) {
      trx_mutex_enter(trx);
      if (!is_forced_rollback(trx) && is_started(trx) &&
          !trx_is_autocommit_non_locking(trx)) {
        ut_ad(trx->killed_by == std::thread::id{});
        /* This transaction has crossed the point of
        no return and cannot be rolled back
        asynchronously now. It must commit or rollback
        synchronously. */
        trx->in_innodb |= TRX_FORCE_ROLLBACK_DISABLE;
        //  ★★★ 进入 COMMIT/ROLLBACK 就钉死 —— 这就是"没有异步提交"的全部实现
      }
      trx_mutex_exit(trx);
    }
    ++trx->in_depth;
    if (trx->in_depth > 1) return;
    trx_mutex_enter(trx);
    wait(trx);                 // ★ 若别人正在对我做异步回滚，这里阻塞等它做完
    ++trx->in_innodb;
    trx_mutex_exit(trx);
  }
```

> ★ `trx_rollback_for_mysql()`（`trx0roll.cc:159`）正是这样调的：
> ```cpp
>   if (TrxInInnoDB::is_async_rollback(trx)) {
>     return (trx_rollback_low(trx));          // ★ 本线程就是 killer → 免计数
>   } else {
>     TrxInInnoDB trx_in_innodb(trx, true);    // ★ disable=true → 打 DISABLE 位
>     return (trx_rollback_low(trx));
>   }
> ```

**唯一产生 `FORCED_ROLLBACK` 的地方**（`trx_commit_in_memory()` 尾部，`trx0trx.cc:2169`）：

```cpp
  /* Because we can rollback transactions asynchronously, we change
  the state at the last step. trx_t::abort cannot change once commit
  or rollback has started because we will have released the locks by
  the time we get here. */
  if (trx->abort) {
    trx->abort = false;
    trx->state.store(TRX_STATE_FORCED_ROLLBACK, std::memory_order_relaxed);  // ★★ trx0trx.cc:2179
  } else {
    trx->state.store(TRX_STATE_NOT_STARTED, std::memory_order_relaxed);
  }
  ...
  trx_init(trx);      // ★ 这里 ++version 并把 id 归零、no 复位 TRX_ID_MAX
```

> ★ `FORCED_ROLLBACK` 与 `NOT_STARTED` **在 `trx_start_if_not_started_xa_low()` 里一视同仁**（`trx0trx.cc:3391`）：两者都直接 `trx_start_low()`。所以"被异步回滚过的事务对象"可以立刻被复用。
>
> ⚠️ `trx_init()` 里还有一句：`trx->in_innodb |= TRX_FORCE_ROLLBACK_DISABLE;`（背景/池化初始状态），随后 `check_trx_exists()` 对用户事务会 `trx->in_innodb &= TRX_FORCE_ROLLBACK_MASK;` 清掉它——**背景事务不能被异步回滚，用户事务可以**。

**`version` 字段的作用**（`trx0trx.h:1132`）：

```cpp
  /** Version of this instance. It is incremented each time the
  instance is re-used in trx_start_low(). It is used to track
  whether a transaction has been restarted since it was tagged
  for asynchronous rollback. */
  std::atomic_uint64_t version;
```

> ★★ `trx_kill_blocking()` 里 `ut_ad(version <= victim_trx->version)` + 自旋条件 `victim_trx->version == version`：如果 victim 在等待期间自己完成并重新开始了新事务（version 变了），killer 就放弃 —— 避免杀错"新一代"事务。

**handler 层抹平**（`ha_innodb.cc:5975`）：

```cpp
    error = trx_rollback_for_mysql(trx);
    if (trx->state.load(std::memory_order_relaxed) == TRX_STATE_FORCED_ROLLBACK) {
      ib::info(ER_IB_MSG_548) << "Forced rollback : " << thd_security_context(...);
      trx->state.store(TRX_STATE_NOT_STARTED, std::memory_order_relaxed);
    }
    trx_deregister_from_2pc(trx);
```

一个容易混淆的点：**InnoDB 没有真正的"异步提交"**——所有提交都是同步的，用户线程必须等 `ordered_commit` 三阶段跑完。但有几个"异步"表象：

| 概念 | 是否真异步 | 本质 |
|------|-----------|------|
| group commit follower | 表象异步 | follower 进 flush stage 后沉睡，leader 代其完成三阶段；但 follower 仍要等 `signal_done` 才拿到结果 |
| **异步回滚（TRX_FORCE_ROLLBACK）** | **真异步** | killer 线程在 victim 毫不知情下代其回滚 |
| 崩溃恢复回滚 | 真异步 | 后台线程回滚未提交事务 |
| `trx_commit_complete_for_mysql` | 非异步 | 只是"延后刷 redo"，提交本身已同步完成 |

**为什么提交必须同步、回滚可以异步**：提交是"对外宣布结果"，客户端必须等到确定答案（成功或失败）；回滚是"内部清理"，谁来做、何时做完不影响外部正确性（只要 undo 最终干净）。这个不对称性决定了**提交没有异步版，回滚有异步版**。

`trx_commit_complete_for_mysql`（trx0trx.cc:2534）常被误当作"异步提交"，实际是"延后刷 redo"：

```cpp
void trx_commit_complete_for_mysql(trx_t *trx) {
  if (trx->id != 0 || !trx->must_flush_log_later ||
      (thd_requested_durability(trx->mysql_thd) == HA_IGNORE_DURABILITY &&
       !trx->ddl_must_flush)) {
    return;
  }

  trx_flush_log_if_needed(trx->commit_lsn, trx);

  trx->must_flush_log_later = false;
  trx->ddl_must_flush = false;
}
```

它只在 `must_flush_log_later` 为真且非 `HA_IGNORE_DURABILITY` 时才真正刷 redo。这是 group commit 的收尾——提交在内存里已完成，redo 的刷盘延迟到这里。**仍是同步提交的延后收尾，不是异步提交**。

---

### 崩溃恢复的提交与回滚

崩溃恢复处理三类事务，分三幕进行。

#### 前置：事务对象的重建（trx_resurrect）

⚠️ **勘误**：**不存在 `trx_resurrect_table_locks()`**（真名 `trx_resurrect_locks(bool all)`，`trx0trx.cc:806`）；另有 `trx_resurrect_table_ids()`（`:728`）。

**`trx_resurrect_insert()` 的状态判定**（`trx0trx.cc:862`）——★ 崩溃恢复判定 trx 命运的唯一依据是 `undo->state`：

```cpp
  trx = trx_allocate_for_background();
  rseg->trx_ref_count++;
  trx->rsegs.m_redo.rseg = rseg;
  *trx->xid = undo->xid;
  trx->id = undo->trx_id;        // ★ id 从 undo 段头里回填
  trx_sys_rw_trx_add(trx);
  trx->rsegs.m_redo.insert_undo = undo;
  trx->is_recovered = true;      // ★★ 复活标记

  /* This is single-threaded startup code, we do not need the
  protection of trx->mutex or trx_sys->mutex here. */

  if (undo->state != TRX_UNDO_ACTIVE) {
    /* Prepared transactions are left in the prepared state
    waiting for a commit or abort decision from MySQL */
    if (undo->is_prepared()) {
      ib::info(ER_IB_MSG_1204) << "Transaction " << trx_get_id_for_print(trx)
                               << " was in the XA prepared state.";
      if (srv_force_recovery == 0) {
        trx->state.store(TRX_STATE_PREPARED, std::memory_order_relaxed);   // ★:896
        ++trx_sys->n_prepared_trx;
      } else {
        ib::info(ER_IB_MSG_1205) << "Since innodb_force_recovery"
                                    " > 0, we will force a rollback.";
        trx->state.store(TRX_STATE_ACTIVE, std::memory_order_relaxed);     // ★:902 force_recovery>0 时 prepared 也强制回滚
      }
    } else {
      trx->state.store(TRX_STATE_COMMITTED_IN_MEMORY, ...);                // ★:905 CACHED/TO_FREE/TO_PURGE ⇒ 已提交
    }
    trx->no = trx->id;      // ★ 已提交事务给个 dummy no（purge 不关心）
  } else {
    trx->state.store(TRX_STATE_ACTIVE, std::memory_order_relaxed);         // ★:918
    trx->no = TRX_ID_MAX;   // ★ 未提交事务的 no 一律最大值
  }

  /* trx_start_low() is not called with resurrect, so need to initialize
  start time here.*/
  if (trx->state... == TRX_STATE_ACTIVE || ... == TRX_STATE_PREPARED) {
    trx->start_time.store(std::chrono::system_clock::from_time_t(time(nullptr)), ...);
  }

  if (!undo->empty) {
    trx->undo_no = undo->top_undo_no + 1;      // ★ 回滚起点从 undo 栈顶恢复
    trx->undo_rseg_space = undo->rseg->space_id;
  }
```

> ★★ **注意 `trx_start_low()` 不参与复活**——所以 `id` 从 `undo->trx_id` 回填（不新分配），`start_time` 要手工设，`no` 由状态决定。这是"重建"而非"启动"。

**`trx_resurrect()`：同一个 trx 可能既有 insert undo 又有 update undo**（`trx0trx.cc:1047`）：

```cpp
  /* Resurrect transactions that were doing inserts. */
  for (auto undo : rseg->insert_undo_list) {
    auto trx = trx_resurrect_insert(undo, rseg);
    trx_resurrect_table_ids(trx, &trx->rsegs.m_redo, undo);
  }

  /* Ressurrect transactions that were doing updates. */
  for (auto undo : rseg->update_undo_list) {
    /* Check the active_rw_trxs.by_id first. */
    trx_t *trx = trx_sys->latch_and_execute_with_active_trx(
        undo->trx_id, [](trx_t *trx) { return trx; }, UT_LOCATION_HERE);
    //  ★★ 先按 trx_id 在 256 分片哈希表里找已建对象复用
    if (trx == nullptr) {
      trx = trx_allocate_for_background();
    }
    trx_resurrect_update(trx, undo, rseg);
    trx_resurrect_table_ids(trx, &trx->rsegs.m_redo, undo);
  }
```

**`trx_resurrect_table_ids()`：逆序遍历 undo 收集涉及的表**（`trx0trx.cc:724`，用于给复活事务补 IX 表锁）：

```cpp
  do {
    trx_undo_rec_get_pars(undo_rec, &type, &cmpl_info, &updated_extern,
                          &undo_no, &table_id, type_cmpl);
    if (tables.insert(table_id).second == true) {   // ★ 去重
      ib::info(ER_IB_RESURRECT_IDENTIFY_TABLE_TO_LOCK, ulong(table_id));
    }
    undo_rec = trx_undo_get_prev_rec(undo_rec, undo->hdr_page_no,
                                     undo->hdr_offset, false, &mtr);
    //  ★★ 逆序遍历整条 undo 链
    ...  // 每 30s 打一次进度（DBUG "resurrect_logs" 时 1s）
  } while (undo_rec);
```

**`trx_lists_init_at_db_start()`**（`trx0trx.cc:1110`）——最后一步：

```cpp
  for (auto rseg : trx_sys->rsegs) trx_resurrect(rseg);
  undo::spaces->s_lock();
  for (auto undo_space : undo::spaces->m_spaces) {
    for (auto rseg : *undo_space->rsegs()) trx_resurrect(rseg);
  }
  undo::spaces->s_unlock();

  /* 从 256 分片里捞出所有 trx，按 id 排序后重建 rw_trx_ids / rw_trx_list */
  ut::vector<trx_t *> trxs;
  for (auto &shard : trx_sys->shards) {
    shard.active_rw_trxs.latch_and_execute([&](const Trx_by_id_with_min &m) {
      for (const auto &t : m.by_id()) trxs.emplace_back(t.second);
    }, UT_LOCATION_HERE);
  }
  std::sort(trxs.begin(), trxs.end(), [](trx_t *a, trx_t *b) { return a->id < b->id; });

  for (trx_t *trx : trxs) {
    if (trx->state... == TRX_STATE_ACTIVE || ... == TRX_STATE_PREPARED) {
      trx_sys->rw_trx_ids.push_back(trx->id);   // ★ 只有 ACTIVE/PREPARED 才进 rw_trx_ids
    }
    trx_add_to_rw_trx_list(trx);
  }
```

> ★ 必须先排序再 `push_back`——因为 `rw_trx_ids` 是**有序 vector**，`ReadView::copy_trx_ids()` 依赖 `front()` 是最小活跃 id，`trx_erase_lists()` 用 `std::lower_bound` 定位。

**恢复后的回滚跑在独立后台线程**（⚠️ **不存在 `trx_rollback_or_clean_all_recovered()`**，真名 `trx_rollback_or_clean_recovered(bool all)`，`trx0roll.cc:708`）：

```cpp
static bool trx_rollback_or_clean_resurrected(trx_t *trx, bool all) {
  switch (state) {
    case TRX_STATE_COMMITTED_IN_MEMORY:
      trx_sys_mutex_exit();
      ib::info(ER_IB_MSG_1188) << "Cleaning up trx with id " << ...;
      trx_cleanup_at_db_startup(trx);
      trx_free_resurrected(trx);
      return true;
    case TRX_STATE_ACTIVE:
      if (all || trx->ddl_operation) {
        trx_sys_mutex_exit();
        trx_rollback_active(trx);          // ★ 真正回滚
        trx_free_for_background(trx);
        return true;
      }
      return false;
    case TRX_STATE_PREPARED:
      return false;                        // ★★ PREPARED 一律不动，交给 MySQL 层 XA COMMIT/ROLLBACK
  }
}
```

```cpp
void trx_recovery_rollback(THD *thd) {
  /* Take MDL locks ... 期间不对外服务，无死锁风险 */
  for (const auto &element : to_rollback_trx_tables) { ... dd_mdl_acquire(...); lock_table_ix_resurrect(table, trx); }
  os_event_set(recovery_lock_taken);       // ★ 放行启动线程
  trx_rollback_or_clean_recovered(true);
  ...
}

void trx_recovery_rollback_thread() {
  THD *thd = create_internal_thd();
  trx_recovery_rollback(thd);
  destroy_internal_thd(thd);
}
```

> ★★ **`trx_rollback_active()` 里用的 fork 类型是 `QUE_FORK_RECOVERY`**（`trx0roll.cc:589`），与 MySQL 接口的 `QUE_FORK_MYSQL_INTERFACE`、回滚图的 `QUE_FORK_ROLLBACK` 区分开。

三幕之前有个前置步骤常被忽略：**崩溃后内存里的 `trx_t` 对象全部丢失，恢复的第一步是把它们从 undo 段重建出来**。入口 `trx_lists_init_at_db_start`（由 `trx_sys_init_at_db_start` 调用）：

1. 遍历所有 rseg（系统表空间 + 各 undo 表空间），对每个 rseg 的 `insert_undo_list` / `update_undo_list` 调 `trx_resurrect_insert` / `trx_resurrect_update`；
2. 从 256 个 shard 的 `active_rw_trxs` 收集全部重建事务，按 trx id 排序，加入 `rw_trx_list`，ACTIVE/PREPARED 的 id 进 `rw_trx_ids`。

**状态映射**——undo 段头的 `TRX_UNDO_STATE` 编码了崩溃瞬间事务所处阶段，重建时据此定态：

| undo 段状态 | 重建后 trx->state | 后续处理 |
|-------------|------------------|----------|
| `TRX_UNDO_ACTIVE` | ACTIVE | 第二幕后回滚 |
| prepared（xid 有效） | PREPARED（`n_prepared_trx++`） | **保留不动**，等第三幕 binlog 裁决 |
| 其他（已提交） | COMMITTED_IN_MEMORY | `trx_cleanup_at_db_startup` 清理 |

`trx_resurrect_insert` 的核心：

```cpp
trx = trx_allocate_for_background();     // 不是 trx_allocate_for_mysql!
trx->id = undo->trx_id;                  // 沿用崩溃前的 trx id
trx_sys_rw_trx_add(trx);                 // 只进 shard 的 active_rw_trxs
trx->is_recovered = true;                // 关键标记
if (undo->state != TRX_UNDO_ACTIVE) {
  if (undo->is_prepared()) { trx->state = TRX_STATE_PREPARED; ++n_prepared_trx; }
  else { trx->state = TRX_STATE_COMMITTED_IN_MEMORY; }
  trx->no = trx->id;                     // dummy 值，purge 以磁盘 undo 头为准
} else {
  trx->state = TRX_STATE_ACTIVE;         // 需要回滚
  trx->no = TRX_ID_MAX;                  // 运行中事务 no 恒为 TRX_ID_MAX
}
```

**resurrect 事务与正常事务的关键差异**（这些差异必须存在）：

| 维度 | 正常事务 | resurrect 事务 |
|------|----------|---------------|
| 分配函数 | `trx_allocate_for_mysql` | `trx_allocate_for_background` |
| `mysql_trx_list` | 加入 | **永不加入** |
| `rw_trx_list` | 仅 rw 且未提交时 | **恒在列** |
| `is_recovered` | false | **true** |
| `mysql_thd` | 关联真实 THD | null（`sess = trx_dummy_sess`） |
| 释放路径 | `trx_free_for_mysql` | `trx_free_resurrected` |

原因：

1. **不进 mysql_trx_list**：该链表语义是"MySQL 层创建、可能尚未 start 的事务"，供 INNODB_TRX 等以 THD 为锚点的接口遍历。崩溃瞬间旧连接已不存在、无 THD，硬塞进去会导致按 THD 解引用悬空；而回滚线程和 purge 需要看到它们，所以只进 rw_trx_list。
2. **is_recovered 是差异化处理的判别位**：`trx_cleanup_at_db_startup`、`trx_free_prepared_or_active_recovered` 都 `ut_a(trx->is_recovered)`，防止正常流程误入恢复专用清理路径。
3. **PREPARED 必须原样保留**：XA 协议要求 prepare 之后由用户/MySQL 决定 commit 还是 rollback，InnoDB 不能擅自处理。

**next_trx_id_or_no 的恢复**（trx0sys.cc）：读 TRX_SYS 页持久化的 max_trx_id，`next_trx_id_or_no = max_trx_id + 2 * TRX_SYS_TRX_ID_WRITE_MARGIN`——**factor 2 是因为 id 和 no 可能被两个线程并发自增**，必须留双倍余量防重启后 id 复用。

**trx_commit_or_rollback_prepare 的作用**（trx0trx.cc:2372）：commit 和 rollback 共用的前置状态机——

- NOT_STARTED / FORCED_ROLLBACK → 先 `trx_start_low(trx, true)` 完成 start（否则后续例程缺 undo 段等未初始化字段）；
- `TRX_QUE_LOCK_WAIT` → 把等待线程置 SUSPENDED、清 wait_thr（**解除锁等待**，否则 commit/rollback 会死锁在锁上）；
- COMMITTED_IN_MEMORY → `ut_error`（已提交事务不应再进提交/回滚流程）。

#### 第一幕：DD 事务同步回滚（单线程启动阶段）

`srv_dict_recover_on_restart`（srv0start.cc:2384）在服务器启动、尚未开放连接时执行：

```cpp
void srv_dict_recover_on_restart() {
  /* Resurrect locks for dictionary transactions */
  trx_resurrect_locks(false);

  /* Roll back any recovered data dictionary transactions, so
  that the data dictionary tables will be free of any locks. */
  if (srv_force_recovery < SRV_FORCE_NO_TRX_UNDO && trx_sys_need_rollback()) {
    trx_rollback_or_clean_recovered(false);   // all=false：只回滚 DD 事务
  }
  ...
}
```

`all=false` 表示**只回滚数据字典事务**（`trx->ddl_operation` 为真的）。为什么必须**同步**做？注释给出理由：后续 `dd_table_open_on_id_low` 构造表对象时**读的是未提交数据**，必须先回滚干净 DD 事务，否则会读到不完整的 DDL 结果。DD 表必须无锁、无脏数据才能继续加载表。

#### 第二幕：后台线程异步回滚（非 PREPARED 事务）

`trx_recovery_rollback_thread`（trx0roll.cc:851）创建一个内部 THD，跑 `trx_recovery_rollback`：

```cpp
void trx_recovery_rollback_thread() {
  THD *thd = create_internal_thd();
  trx_recovery_rollback(thd);
  destroy_internal_thd(thd);
}
```

`trx_recovery_rollback` 拿完 MDL 后调 `trx_rollback_or_clean_recovered(true)`，遍历 `rw_trx_list`，对每个 recovered 事务按状态裁决（`trx_rollback_or_clean_resurrected`）：

| 恢复时看到的状态 | 处理 |
|-----------------|------|
| `TRX_STATE_COMMITTED_IN_MEMORY` | `trx_cleanup_at_db_startup`——已提交但没清理，清 insert undo |
| `TRX_STATE_ACTIVE` | `trx_rollback_active`——未提交，逐行回滚 |
| `TRX_STATE_PREPARED` | **跳过**——留给 binlog 裁决（第三幕） |

`trx_rollback_active`（trx0roll.cc:573）是后台回滚的核心：

```cpp
static void trx_rollback_active(trx_t *trx) {
  ...
  fork = que_fork_create(nullptr, nullptr, QUE_FORK_RECOVERY, heap);
  fork->trx = trx;
  thr = que_thr_create(fork, heap, nullptr);
  roll_node = roll_node_create(heap);
  thr->child = roll_node;
  ...
  trx->graph = fork;

  ut_a(thr == que_fork_start_command(fork));

  trx_sys_mutex_enter();
  trx_roll_crash_recv_trx = trx;
  trx_roll_max_undo_no = trx->undo_no;      // 记录要回滚多少
  trx_roll_progress_printed_pct = 0;
  rows_to_undo = trx_roll_max_undo_no;
  trx_sys_mutex_exit();

  ib::info(ER_IB_MSG_1186) << "Rolling back trx with id " << trx_id << ", "
                           << rows_to_undo << unit << " rows to undo";

  que_run_threads(thr);                       // 逐行撤销
  ut_a(roll_node->undo_thr != nullptr);
  que_run_threads(roll_node->undo_thr);

  trx_rollback_finish(thr_get_trx(roll_node->undo_thr));
  ...
  ib::info(ER_IB_MSG_1187) << "Rollback of trx with id " << trx_id
                           << " completed";
  ...
  trx_roll_crash_recv_trx = nullptr;
}
```

与前台回滚的区别：用 `QUE_FORK_RECOVERY` fork（而非 `QUE_FORK_MYSQL_INTERFACE`），并通过全局变量 `trx_roll_crash_recv_trx` / `trx_roll_max_undo_no` 暴露进度——`SHOW ENGINE INNODB STATUS` 的 `TRX ... ROLLING BACK` 段就是读这两个变量。日志里成对出现的 `Rolling back trx ... rows to undo` 与 `completed` 正是崩溃后启动慢时最醒目的两条。

#### 第三幕：prepared 事务的 binlog 裁决（server 层）

prepared 事务在第二幕被**刻意跳过**，因为它的命运取决于 binlog。裁决在 server 层（`sql/xa/recovery.cc`）：

```cpp
void recover_one_internal_trx(xarecover_st const &info, handlerton &ht,
                              XA_recover_txn const &xa_trx, my_xid xid,
                              ::recovery_statistics &stats) {
  // info.commit_list = binlog 恢复扫描出的、所有已写 Xid_log_event 的 XID 集合
  // ★ 最终裁决：XID 在 commit_list 中 → 提交；不在 → 回滚 ⇒ binlog 说了算
  if (info.commit_list ? info.commit_list->count(xid) != 0
                       // tc_heuristic_recover 是 --tc-heuristic-recover=COMMIT 的兜底——
                       // binlog 损坏/缺失、无法确定时，管理员手动指定"全部按提交处理"
                       : tc_heuristic_recover == TC_HEURISTIC_RECOVER_COMMIT) {
    exec_status = ht.commit_by_xid(&ht, const_cast<XID *>(&xa_trx.id));
    ...
  } else {
    exec_status = ht.rollback_by_xid(&ht, const_cast<XID *>(&xa_trx.id));
    ...
  }
}
```

外部 XA 用更复杂的状态机（`recover_one_external_trx`）：

| binlog 扫描到的 XA 状态 | 处理 |
|------------------------|------|
| `COMMITTED` / `COMMITTED_WITH_ONEPHASE` | `commit_by_xid` |
| `NOT_FOUND` / `PREPARED_IN_SE` / `ROLLEDBACK` | `rollback_by_xid` |
| `PREPARED_IN_TC` | `set_prepared_in_tc_by_xid`（保持 prepared，登记到 Recovered_xa_transactions，等用户 `XA RECOVER` 后手动裁决） |

第三幕的 `PREPARED_IN_TC` 正是前面演进章节讲的 8.0.29 新状态——外部 XA 的 prepared 事务若 binlog 也确认 prepared 了，就保持 prepared 状态等用户显式 `XA COMMIT/ROLLBACK`。

#### commit_by_xid / rollback_by_xid：无 THD 的引擎内部路径

裁决后的执行**不经过 server 层的 BGC 流水线**——此时事务没有 THD 上下文（恢复场景），直接走引擎内部：

```cpp
static xa_status_code innobase_commit_by_xid(handlerton *hton, XID *xid) {
  trx_t *trx = trx_get_trx_by_xid(xid);   // 按 XID 查 prepared 事务（与 by_xid 系列共用查找路径）

  if (trx != nullptr) {
    {
      TrxInInnoDB trx_in_innodb(trx);

      // ★ 与正常提交同一套引擎内部提交（undo 标记 COMMITTED、分配 trx->no、放锁），
      //   但不经过 BGC 三阶段——binlog 早在崩溃前就写好了 Xid_log_event，恢复只需补齐引擎侧
      innobase_commit_low(trx);
    }
    /* use cases are: disconnected xa, slave xa, recovery */
    trx_deregister_from_2pc(trx);         // 从 2PC 注册表摘除
    trx_free_for_background(trx);         // 恢复出的事务对象走 background 释放路径

    return (XA_OK);
  } else {
    return (XAER_NOTA);
  }
}
```

回滚版对称，走 `innobase_rollback_trx`（引擎内部回滚），同样 deregister + free，失败返回 `XAER_RMERR`：

```cpp
static xa_status_code innobase_rollback_by_xid(handlerton *hton, XID *xid) {
  trx_t *trx = trx_get_trx_by_xid(xid);

  if (trx != nullptr) {
    int ret;
    {
      TrxInInnoDB trx_in_innodb(trx);
      ret = innobase_rollback_trx(trx);
    }

    trx_deregister_from_2pc(trx);
    trx_free_for_background(trx);

    return (ret != 0 ? XAER_RMERR : XA_OK);
  } else {
    return (XAER_NOTA);
  }
}
```

**要点**：恢复场景的 commit/rollback 都走"纯引擎路径"——没有 THD、没有 MDL、没有 BGC，因为：

1. binlog 侧早已确定（XID 在不在 `commit_list`）；
2. 引擎侧只需把内存/文件状态补齐；
3. 此时的"提交"**不产生新的 binlog 事件**（`mysql_thd == nullptr`，无从记 binlog 位置）。

#### 三幕的完整时序

```
服务器启动
  └─ srv_dict_recover_on_restart                    第一幕（单线程）
      ├─ trx_resurrect_locks(false)
      └─ trx_rollback_or_clean_recovered(false)      回滚 DD 事务
  └─ 后台线程 trx_recovery_rollback_thread          第二幕（异步）
      └─ trx_recovery_rollback → trx_rollback_or_clean_recovered(true)
          回滚所有非 PREPARED 事务（COMMITTED_IN_MEMORY→cleanup / ACTIVE→rollback）
  └─ binlog 恢复                                    第三幕（server 层）
      └─ recover_one_internal_trx / recover_one_external_trx
          对 PREPARED 事务按 XID 裁决：commit_list 有 → commit；无 → rollback
```

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_flush_log_at_trx_commit` | 1 | Global | 控制 redo 刷盘策略。=1 每次提交 fsync（BGC 中在 flush stage 批量做）；=0 每秒一次，**BGC 期间完全不刷 redo**；=2 只 write 不 fsync |
| `innodb_lock_wait_timeout` | 50 | Global/Session | 锁等待超时秒数。超时后回滚**当前语句**（不是整个事务），除非 `innodb_rollback_on_timeout=ON` |
| `innodb_rollback_on_timeout` | OFF | Global | OFF=超时只回滚当前语句；ON=超时回滚整个事务 |
| `innodb_rollback_segments` | 128 | Global | 回滚段数量，决定并发读写事务上限（每个事务占一个 rseg slot） |
| `innodb_max_undo_log_size` | 1GB | Global | undo 表空间超过此值触发截断 |
| `innodb_purge_threads` | 4 | Global | purge 线程数，`trx->no` 是 purge 的排序依据 |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| `Innodb_trx_rseg_history_len` | history list 长度（未 purge 的 undo 量），对应 `rseg_history_len` |
| `Innodb_ibuf_...` | 与事务无直接关系，略 |

---

## Misc

### 事务的诊断接口：INNODB_TRX 与 SHOW ENGINE INNODB STATUS

**两个链表的并集**：观察接口需要同时遍历 `rw_trx_list`（MVCC 核心，含系统/恢复事务）和 `mysql_trx_list`（会话视角，含未开始/只读事务），再按 `trx->id` / `read_only` 去重：

- INNODB_TRX（`trx0i_s.cc`）：先扫 rw_trx_list，再扫 mysql_trx_list 时用 `trx->id != 0 && !trx->read_only` 过滤已处理的 RW 事务；
- SHOW ENGINE INNODB STATUS（`lock_print_info_all_transactions`）：mysql_trx_list 只取 NOT_STARTED（输出 `---`），rw_trx_list 取其余。

**两个易忽略的细节**：

1. **trx_id 列不是裸 `trx->id`**，而是 `trx_get_id_for_print`：`trx->id == 0`（只读事务）时用"trx 对象地址 | 2^48"构造伪 ID（第 49 位置 1）：

```cpp
return (trx->id != 0
            ? trx->id
            : trx_id_t{reinterpret_cast<uintptr_t>(trx)} | (max_trx_id + 1));
```

动机：INNODB_TRX、P_S data_locks、SHOW STATUS 各接口必须打印**同一个值**才能 JOIN；只读事务的伪 ID 可以复用，仅用于打印、无 MVCC 语义。

2. **INNODB_TRX 的 trx_state 列取的是 `trx->lock.que_state`**（RUNNING / LOCK WAIT / ROLLING BACK / COMMITTING），**不是** `trx->state`（状态机五态）。

**trx_rw_is_active 的查询优化**（trx0sys.ic）：无锁快路径 + 持锁慢路径：

```cpp
static inline trx_t *trx_rw_is_active(trx_id_t trx_id, bool do_ref_count) {
  auto &shard = trx_sys->get_shard_by_trx_id(trx_id);
  if (trx_id < shard.active_rw_trxs.peek().min_id()) {   // 无锁：atomic min_id
    return nullptr;
  }
  return trx_sys->latch_and_execute_with_active_trx(...); // 持 shard mutex 慢路径
}
```

`min_id()` 无锁短路是安全的：insert/erase 在同一 shard mutex 临界区内同时维护 map 与 min_id，若读到的 min_id > trx_id，该 trx 必然已不在 map 中。★ 注意 8.0.39 已无 `trx_rw_is_active_low`（老版本函数，现重构为快/慢双路径）。



### 易混淆概念对比

| 概念 | 含义 | 常见误解 |
|------|------|----------|
| `trx->id` | 事务开始分配，写进 `DB_TRX_ID`，标识"谁改的" | 误以为是提交序。只读事务 `id = 0` |
| `trx->no` | 提交时分配，决定 purge 序与 MVCC 可见性 | 误以为所有事务都有。纯 INSERT 事务不分配 `trx->no` |
| `TRX_STATE_PREPARED` | 2PC 已 prepare，等待协调者裁决 | 误以为是"已提交"。它可能最终被 ROLLBACK |
| `TRX_STATE_COMMITTED_IN_MEMORY` | 内存中已提交，**对外可见** | 误以为已持久化。持久性由 redo + binlog 决定 |
| `TRX_STATE_FORCED_ROLLBACK` | 与 `NOT_STARTED` 同义，只是标记"上次被异步回滚" | 误以为是独立的长驻状态 |
| AC-NL-RO | autocommit 非加锁只读事务 | 不知道这类事务 `id=0` 且不能被异步回滚 |

### 提交点 ≠ 引擎 commit

最容易搞错的一点：**事务的提交点是 binlog fsync 成功那一刻（sync stage），不是 InnoDB 的 `trx_commit_in_memory`（commit stage）**。

顺序是：prepare（redo prepare）→ flush（redo fsync + 写 binlog）→ **sync（binlog fsync = 提交点）**→ commit（引擎内存收尾）。

在 sync 之后崩溃，事务已提交（binlog 有 XID），InnoDB 侧的 prepare 状态会在恢复时被裁决为 COMMIT。

### 为什么大事务回滚比正向执行慢

正向执行是顺序追加 undo；回滚是**随机读 undo 页 + 随机更新数据页**，且每次 undo 还要写 redo（WAL）。单位行成本通常高于正向。

---

## 参考

**论文**

- Mohan, C., Haderle, D., Lindsay, B., Pirahesh, H., Schwarz, P. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. ACM TODS, 1992. — undo log + WAL 的经典，`trx_write_serialisation_history` 与回滚的 undo pass 直接源自此
- Gray, J. N., Lorie, R. A., Putzolu, G. R., Traiger, I. L. *Granularity of Locks and Degrees of Consistency in Shared Data Banks*. IBM, 1975. — 2PL，`trx_release_impl_and_expl_locks` 是 shrinking phase
- Gray, J. N. *Notes on Data Base Operating Systems*. Operating Systems: An Advanced Course, 1978. — 两阶段提交协议，MySQL 内部 2PC 的理论基础

**官方文档**

- *MySQL 8.0 Reference Manual → InnoDB Transaction Model and Locking*
- *MySQL 8.0 Reference Manual → XA Transactions*
- *MySQL 8.0 Reference Manual → InnoDB Startup Options and System Variables*（`innodb_purge_threads`、`innodb_rollback_segments` 等变量语义）

**相关文档**

- 上游（server 层 2PC 与 BGC 流水线）见 [`../server/replication/binlog.md`](../server/replication/binlog.md)
- 下游（GTID 持久化时机与三条路径）见 [`../server/replication/gtid.md`](../server/replication/gtid.md)
- 相关（查询图执行模型，回滚时 `que_run_threads` 的驱动机制）见 [`query_graph.md`](query_graph.md)
