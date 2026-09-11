# MySQL 并行复制（MTS）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 MTS 架构、逻辑时钟（sequence_number/last_committed）、两个 Logical_clock 不可合并的原理、三种依赖跟踪模式、lock interval 理论、commit-parent-based vs lock-based、false positive 安全性、从库调度（GAQ/LWM/schedule_next_event）、writeset 生成。

## 目录

- [MTS 架构与演进](#mts-架构与演进)
- [逻辑时钟：sequence_number 与 last_committed](#逻辑时钟sequence_number-与-last_committed)
- [为什么需要两个 Logical_clock](#为什么需要两个-logical_clock)
- [为什么不做全局 sequence_number](#为什么不做全局-sequence_number)
- [三种依赖跟踪模式](#三种依赖跟踪模式)
- [Lock Interval 理论](#lock-interval-理论)
- [Commit-Parent-Based vs Lock-Based](#commit-parent-based-vs-lock-based)
- [False Positive 与安全性](#false-positive-与安全性)
- [从库调度机制](#从库调度机制)
- [Writeset 生成](#writeset-生成)
- [生产实践要点](#生产实践要点)
- [生产案例：无主键表引发的同步延迟](#生产案例无主键表引发的同步延迟)
- [关键源码位置速查](#关键源码位置速查)

---

## MTS 架构与演进

### Coordinator + Worker 模型

MySQL 并行复制（MTS，Multi-Threaded Slave）采用**单 Coordinator（协调者）+ 多 Worker（工作者）**模型。Coordinator 就是原来的 SQL 线程，负责从 relay log 读取事件、做依赖分析、把事务分发给各个 Worker；Worker 线程并行执行被分发的事务。

`slave_parallel_workers` 参数控制 Worker 数量（0 = 退化为单线程 STS）。`slave_parallel_type` 参数选择并行策略：

| 类型 | 值 | 说明 |
|------|---|------|
| `DATABASE` | `MTS_PARALLEL_TYPE_DB_NAME` = 0 | 5.6 库级并行，`Mts_submode_database` |
| `LOGICAL_CLOCK` | `MTS_PARALLEL_TYPE_LOGICAL_CLOCK` = 1 | 5.7+ 逻辑时钟并行，`Mts_submode_logical_clock` |

### 演进脉络

**5.6 库级并行（DATABASE）**：按事务涉及的 database 名做哈希分区，同一库的事务串行，不同库可并行。缺陷是真实业务往往集中在少数库上，并行度极低；跨库事务无法并行。

**5.7 组提交并行（LOGICAL_CLOCK + COMMIT_ORDER）**：核心洞察是 group commit 同一组的事务必然锁区间不重叠（被行锁阻塞的事务无法进入 commit 流程，不会同组）。5.7 把组信息编码成 `sequence_number` / `last_committed` 写进 Gtid_log_event，从库据此构建 DAG 调度。并行度受限于主库 group commit 的合并程度。

**8.0 WRITESET 系列**：即使主库没有分到同一组，只要两个事务修改的行集合不相交，从库就可以并行。WRITESET 用行主键哈希的历史记录主动压低 `last_committed`，在低并发主库上也能获得高从库并行度。

### 两个参数两层关系

```
slave_parallel_type（从库侧，选 submode）
├── DATABASE（库级，独立体系）
└── LOGICAL_CLOCK（逻辑时钟）
    └── binlog_transaction_dependency_tracking（主库侧，生成 last_committed 算法）
        ├── COMMIT_ORDER（锁区间）
        ├── WRITESET（锁区间 + 行哈希压低）
        └── WRITESET_SESSION（锁区间 + 行哈希压低 + session 约束）
```

第一层是"从库怎么调度"，第二层是"主库怎么编码依赖信息"。两者独立，但只有 LOGICAL_CLOCK 模式下第二层才生效。

---

## 逻辑时钟：sequence_number 与 last_committed

### 写入位置

`sequence_number` 和 `last_committed`（源码中也叫 `commit_parent`）写入 **`Gtid_log_event` 事件头**（log_event.cc:13217）：

```cpp
int8store(ptr_buffer, last_committed);          // 偏移 0
int8store(ptr_buffer + 8, sequence_number);     // 偏移 8
```

它们与 GTID 是两套独立的东西——GTID 标识"是哪个事务"，这两个值描述"本事务与哪些事务可并行回放"。

### 两个 Logical_clock

`Commit_order_trx_dependency_tracker`（rpl_trx_tracking.h:88）内有两个 `Logical_clock` 成员（:112-117）：

```cpp
Logical_clock m_max_committed_transaction;  // 已提交事务的最大序号
Logical_clock m_transaction_counter;        // 已进入 flush 事务的最大序号（prepared）
```

`Logical_clock`（:45）是 `std::atomic<int64> state` + `int64 offset`：

- `step()`（rpl_trx_tracking.cc:65）：`++state`，单调递增，**串行调用**（flush stage leader 串行 flush 保证安全）
- `set_if_greater(new_val)`（:83）：CAS 把 state 推到 `max(state, new_val)`，**并发安全**
- `get_timestamp()`：读当前 state（绝对值）

两个时钟的语义区别是核心：`m_transaction_counter` 在事务**进入 flush 阶段**时 step 递增（代表"开始被处理"），`m_max_committed_transaction` 在事务**完成 commit 之后**用 `set_if_greater` 更新（代表"已完成、已放锁"）。前者总是 >= 后者。

### sequence_number 的生成

在 flush stage，leader 串行处理每个事务的 binlog cache，`binlog_cache_data::flush`（binlog.cc:2419）给当前事务分配 sequence_number：

```cpp
trn_ctx->sequence_number = mysql_bin_log.m_dependency_tracker.step();
```

`step()` 让 `m_transaction_counter.state` 自增 1 返回新值。sequence_number = 事务进入 flush 的全局递增序号，严格反映 binlog 写入顺序。

### last_committed 的生成

事务进入 commit 流程**之前**（binlog.cc:2567 / 2897 / 8164），通过 `store_commit_parent`（transaction_info.h:228）记录当前已提交水位：

```cpp
thd->get_transaction()->store_commit_parent(
    mysql_bin_log.m_dependency_tracker.get_max_committed_timestamp());
```

这一刻的 `m_max_committed_transaction` 代表"已经提交完成（已释放锁）的事务的最大序号"。

### state 与 offset

| 概念 | 存储位置 | 绝对/相对 | 何时赋值 | 作用 |
|------|---------|----------|---------|------|
| `m_transaction_counter.state` | 全局单例 | 绝对 | flush 时 `step()` | 已进 flush 事务的全局计数 |
| `m_max_committed_transaction.state` | 全局单例 | 绝对 | commit 时 `set_if_greater` | 已提交事务的全局最大序号 |
| `offset` | 全局单例 | 绝对 | rotate 时赋值为当时 state | 本 binlog 起点偏移 |
| `trx_ctx->sequence_number` | 每事务 | 绝对 | flush 时 `= step()` 返回值 | 本事务的全局 flush 序号 |
| `trx_ctx->last_committed` | 每事务 | 绝对 | commit 前 `= store_commit_parent(max_committed)` | 本事务开始 commit 时的已提交水位 |
| binlog 中的 `sequence_number` | Gtid_log_event | **相对** | `= trx_ctx->sequence_number - offset` | 从库用的相对序号 |
| binlog 中的 `last_committed` | Gtid_log_event | **相对** | `= max(trx_ctx->last_committed, m_last_blocking_transaction) - offset` | 从库用的相对 commit_parent |

**内存里全是绝对值，binlog 里全是相对值。** `m_last_blocking_transaction`（rpl_trx_tracking.h:125）也是绝对值。

### trx_ctx（Transaction_ctx）

每事务上下文对象，挂在 THD 上（`thd->get_transaction()`）。封装事务全状态：`xid_state` / `last_committed` / `sequence_number` / `no_2pc` / `rw_ha_count` / `m_flags` / `savepoints` / `binlog_cache_mngr` / `write_set_ctx`。跨模块跨函数链（`ha_commit_trans` → `tc_log` → `ordered_commit` → `process_commit_stage_queue` → `finish_transaction_in_engines`）共享事务状态，标准 context 模式。

`sequence_number` / `last_committed` 是每事务属性存在 `trx_ctx` 里（从全局时钟取快照），全局时钟（`m_transaction_counter` / `m_max_committed`）是所有事务共享。

### commit stage 更新水位

事务真正提交后，`process_commit_stage_queue`（binlog.cc:8537）调 `update_max_committed`（rpl_trx_tracking.cc:382）：

```cpp
m_commit_order.update_max_committed(trn_ctx->sequence_number);  // set_if_greater
trn_ctx->sequence_number = SEQ_UNINIT;   // 用完清空
```

用 `set_if_greater` 推高 `m_max_committed_transaction`，然后把这个事务的 sequence_number 重置为 `SEQ_UNINIT`。在 `LOCK_replica_trans_dep_tracker` 保护下。

### offset 与 binlog rotate

rotate 时（`Commit_order_trx_dependency_tracker::rotate`，cc:185）两个 offset 都更新为 `m_transaction_counter.get_timestamp()`。写 event 时减 offset 转相对值，让每个 binlog 文件内 sequence_number 从 1 开始。

**跨 binlog 文件比较无意义**——rotate 是天然并行边界。若 `last_committed <= offset` 就记 `SEQ_UNINIT`（cc:169-170），从库把它当 `is_new_group` 处理。

### 屏障事务

`m_last_blocking_transaction`（rpl_trx_tracking.h:125）处理 `parallelization_barrier`（如某些 DDL）或 `is_trx_unsafe_for_parallel_slave` 判定不安全的事务（ANALYZE/REPAIR/OPTIMIZE/CREATE_DB 等，cc:119）。这类事务的 sequence_number 被记下，后续事务的 commit_parent 不会低于它——强制后续事务依赖屏障事务串行。

---

## 为什么需要两个 Logical_clock

### 不可合并的两条硬约束

**约束一**：sequence_number 必须在 flush 时写入 binlog，而 max_committed 只能在 commit 时更新。BGC 流水线的硬时序是：flush（发号 → 写 binlog）→ sync → commit（更新 max_committed）。发号和标记完成的时机不可调换。

**约束二**：`store_commit_parent` 必须读"已提交水位"（`m_max_committed_transaction`）而非"已 flush 水位"（`m_transaction_counter`）。因为事务进入 flush 后还持锁走 sync/commit，此时若用"已 flush 水平"作为 last_committed，会包含"已 flush 但未放锁"的事务，导致误判。

### m_transaction_counter 不可替代

`m_transaction_counter` 定义序号空间（坐标轴），`m_max_committed_transaction` 是这个空间里的完成进度线。两者共用同一序号空间但推进时机不同：flush（发号）vs commit（标记完成）。少一个都会导致并行判定错误。

### rotate 时 offset 基准

rotate 时用 `m_transaction_counter` 的值做 offset（而非 `m_max_committed_transaction`），因为 rotate 发生在 commit stage 末尾，此时可能有事务已经 flush 但还没 commit。用领先的 `m_transaction_counter` 确保"已 flush 未 commit"事务的绝对 sequence_number >= offset，减出来是正数。

---

## 为什么不做全局 sequence_number

问题在从库而非主库（主库的 state 本来就是全局的）：

1. **从库可从任意位点开始复制**（`CHANGE REPLICATION SOURCE TO SOURCE_LOG_POS=xxx`），用全局值需知道起始 offset
2. **rotate 是天然依赖切断点**：相对值时新文件首事务 `last_committed <= offset` 记 `SEQ_UNINIT`，从库当 `is_new_group` 处理
3. **GAQ 重启清空 LWM 重置**：相对值每文件自包含，重启后从当前文件重新开始即可
4. **空间效率 + 调试直观**

---

## 三种依赖跟踪模式

`binlog_transaction_dependency_tracking` 参数（rpl_trx_tracking.h:191）控制生成 last_committed 的算法。`Transaction_dependency_tracker::get_dependency`（cc:331）按模式分发，**三种是叠加递进而非互斥**：

### COMMIT_ORDER（0，默认）

仅用锁区间重叠判断。`Commit_order_trx_dependency_tracker::get_dependency`（cc:149）直接把 `trx_ctx->last_committed` 减 offset 作为 commit_parent。保守但正确，无需额外开销。

### WRITESET（1）

在 COMMIT_ORDER 算出的 commit_parent 基础上，**尝试把它降得更低**。`Writeset_trx_dependency_tracker::get_dependency`（cc:213）用事务修改的行哈希查 `m_writeset_history`（`std::map<uint64, int64>`，:168，记录每个行最后被哪个 sequence_number 改过）。若本事务改的行都没被近期事务碰过，就把 commit_parent 压到更低，让从库能把它与更多事务并行。

降级条件（任一成立回退 COMMIT_ORDER 并清空 history，cc:232-247）：
- 空 writeset（DDL / 无主键）
- 外键级联（`has_related_foreign_keys`）
- write_set_limit 超限
- hash 算法 session/global 不一致

### WRITESET_SESSION（2）

在 WRITESET 基础上加**同 session 强制串行**。`Writeset_session_trx_dependency_tracker::get_dependency`（cc:315）把 commit_parent 与 `session_parent`（本 session 上一事务的 sequence_number）取 max。

叠加关系（cc:341-351）：WRITESET 先跑 COMMIT_ORDER 再跑 WRITESET；WRITESET_SESSION 在两者之上再加 session 约束。

---

## Lock Interval 理论

### Lock Interval 定义

WL#7165 定义 lock interval = [L, C]：

- **L（lock interval 开始）**：事务获取**最后一把锁**的时刻（= `store_commit_parent` 点，进入 prepare 前）
- **C（lock interval 结束）**：事务释放**第一把锁**的时刻（= 引擎 commit 放锁点）

### 并行判定

两个事务 T1（先）、T2（后）：

- **lock interval 重叠（T2.L < T1.C）→ 可并行**：T2 在 T1 释放锁之前就拿到了自己的所有锁。T2 不被 T1 阻塞 = 修改的行不冲突 = 可并行。
- **lock interval 不重叠（T2.L >= T1.C）→ 串行**：T2 在 T1 释放锁之后才拿到所有锁。可能有锁冲突（T2 被 T1 阻塞过），保守串行。

### 对应到 MySQL 实现

- lock interval 重叠 → T2 记录 last_committed 时 T1 还没 commit → `m_max_committed` 还没更新到 T1.sn → `T2.lc < T1.sn` → 从库 `T2.lc <= LWM` → 可并行
- lock interval 不重叠 → T2 记录 last_committed 时 T1 已 commit → `m_max_committed` 已更新到 T1.sn → `T2.lc >= T1.sn` → 从库要等 T1 完成

### Group commit 同组无锁冲突

flush 队列是自然过滤器——被行锁阻塞的事务无法执行完用户 SQL 进入 commit 流程，不会同时出现在 flush 队列。具体是 InnoDB 行锁（record / gap / next-key lock）+ MDL 表锁，核心是行锁。

### lock-based 允许跨组并行

lock-based 的 last_committed 取的是"已结束 lock interval 事务的最大 sequence_number"（不一定等于前一组最后一个事务的 sequence_number）。因此即使事务在不同组中，只要存在 lock interval 重叠，从库也可能并行回放。

### last_committed 的精确定义

> last_committed 取自前一组中，与本组事务不存在 lock interval 重叠的最后一个事务的 sequence_number。

逐词解读：

- **"前一组"**：在当前事务 T2 之前的事务（binlog 顺序中先于 T2）
- **"不存在 lock interval 重叠"**：这些事务的 C 点（释放锁）在 T2 的 L 点（获取所有锁）之前——lock interval 已结束
- **"最后一个"**：sequence_number 最大的那个

合起来：**T2 的 last_committed = T2 的 L 点时刻，所有 C 点已过（lock interval 已结束、已释放锁）的事务中，sequence_number 最大的那个。**

这正是 `store_commit_parent(m_max_committed_transaction)` 的语义——`m_max_committed_transaction` 在事务 commit（释放锁、lock interval 结束）时通过 `set_if_greater` 更新到该事务的 sequence_number，T2 在 L 点读取它的当前值。

### t6 例子：为什么 last_committed 不是前一个事务的 sequence_number

```
t1, lc=0,  sn=3
t2, lc=3,  sn=4
t3, lc=3,  sn=5
t4, lc=3,  sn=6
t5, lc=3,  sn=7
t6, lc=6,  sn=8   ← 不是7！
t7, lc=6,  sn=9
t8, lc=9,  sn=10
```

**t6 的 last_committed=6（= t4 的 sn），不是 7（= t5 的 sn）**。

因为 t6 开始 prepare（L 点）时：
- t4 已经 commit（lock interval 已结束）→ `m_max_committed` >= 6
- t5 还没 commit（lock interval 未结束，还在持锁）→ `m_max_committed` 还没更新到 7

t6 取的是"lock interval 已结束的最远事务"= t4，跳过了"还没 commit"的 t5。

从库调度：t6/t7 的 lc=6，t4 完成后（LWM>=6）可并行分发——**不需要等 t5（sn=7）完成**。这就是 lock-based 跨组并行的体现。commit-parent-based 会取 t5 的 sn=7，导致 t6 必须等 t5。

### 图示解读

**Commit-Parent-Based（按 P 点分组）**：

```
Trx1: ━━P━━━━C━━━━━━━━━━━━━━━━
Trx2: ━━P━━━━━━━━C━━━━━━━━━━━━     ← 与Trx1同组(P点同时),可并行
Trx3: ━━━━━━━━━━P━━━━C━━━━━━━━
Trx4: ━━━━━━━━━━P━━━━━━━━C━━━━     ← 与Trx3同组,可并行
Trx5: ━━━━━━━━━━P━━━━━━━━━━━C━━     ← 与Trx3/4同组,可并行
Trx6: ━━━━━━━━━━━━━━━━━━━━━━P━━C   ← 新组,等前组完成
```

commit-parent-based 按 **P 点**先后分组，同组可并行、跨组串行。即使 Trx4/5 的 lock interval 与 Trx6 重叠（实际无锁冲突），也强制 Trx6 等待。

**Lock-Based — lock interval 重叠（可并行）**：

```
Tx0: ━━━P━━━━━━━━━━━C━━━━━
Tx1: ━━━━━━━P━━━━━━━━━━━━━━━━━━
              ↑
         Tx1.P < Tx0.C (重叠)
```

Tx1 在 Tx0 释放锁之前就拿到了自己的所有锁 → 同时持锁 → 无锁冲突 → 可并行。

**Lock-Based — lock interval 不重叠（串行）**：

```
Tx0: ━━━P━━━━━━━━━━━C━━━━━━━━━━
Tx1: ━━━━━━━━━━━━━━━━━━━P━━━━━━━━━
                        ↑
                   Tx1.P >= Tx0.C (不重叠)
```

Tx1 在 Tx0 释放锁之后才拿到锁 → 可能有锁冲突 → 串行（可能 false positive 但安全）。

### 三个概念不能混

| 概念 | 层面 | 本质 | 何时产生 | 是否写入 binlog |
|------|------|------|---------|----------------|
| **lock interval** | 物理事实 | 事务实际持有所有锁的时间区间 [L, C] | 引擎运行时自然产生 | 否，从库看不到 |
| **sequence_number** | 逻辑编号 | 事务进入 flush 的全局递增序号 | flush stage 时 `step()` 分配 | 是，写入 Gtid_log_event |
| **last_committed** | 依赖标记 | L 点时刻已 commit 事务的最大 sequence_number | prepare 前 `store_commit_parent` 记录 | 是，写入 Gtid_log_event |

因果关系：

```
lock interval（物理事实）
  ├─ L 点（获取所有锁）→ 触发 store_commit_parent → 产生 last_committed（依赖标记）
  └─ C 点（释放锁）  → 触发 update_max_committed → 更新 m_max_committed 到该事务的 sequence_number（编号）
```

从库拿到 last_committed 和 sequence_number 两个数值，**反推** lock interval 是否重叠——它看不到真正的 lock interval。这就是为什么需要两个时钟把 lock interval 信息"编码"进这两个数值。

### m_max_committed 代表 end of lock interval

`m_max_committed_transaction` 在语义上代表"end of lock interval"——即已释放锁的事务中最大的 sequence_number。

实现细节（binlog.cc:8537-8552）：`update_max_committed` 在 `finish_transaction_in_engines`（释放锁）**之前**一行调用：

```cpp
for (THD *head = first; head; head = head->next_to_commit) {
    m_dependency_tracker.update_max_committed(head);        // ① 更新 m_max_committed
    ...
    ::finish_transaction_in_engines(head, all, false);      // ② 引擎 commit，释放锁
}
```

虽然 update 在 finish 之前，但 BGC 的时序隔离保证不影响正确性：

- **同组事务**：`store_commit_parent` 在 flush stage 之前调用，此时整个 group 还没进 commit stage，m_max_committed 还是组前的值
- **跨组事务**：`store_commit_parent` 在前组 commit stage 完全完成后才调用，m_max_committed 已更新且锁已释放

没有任何事务的 `store_commit_parent` 能在"m_max_committed 已更新但锁未释放"这个窗口内被调用，所以 m_max_committed 完全可以代表 end of lock interval。

---

## Commit-Parent-Based vs Lock-Based

### 核心区别

两种方案的核心区别在于 `store_commit_parent` 读哪个时钟：

| | Commit-Parent-Based（假设/历史方案） | Lock-Based（MySQL 实现） |
|---|---|---|
| store_commit_parent 读 | `m_transaction_counter`（已 flush 水位 = 处理进度） | `m_max_committed_transaction`（已 commit 水位 = 锁释放进度） |
| 记录的信息 | 处理进度（谁在我之前开始被处理） | 锁释放进度（谁在我之前放锁） |
| 需要几个时钟 | 1 个 | 2 个 |
| 源码中是否存在 | 不存在（WL#7165 早期方案，5.7 早期实现后改进） | 是（COMMIT_ORDER 模式） |

### 两种都安全

T2 能在 T1 未 commit 时调 `store_commit_parent`，本身就证明了 T2 没有被 T1 的锁阻塞——否则 T2 根本执行不完用户 SQL、拿不到所有锁、进不了 commit 流程。所以即使 last_committed 包含了"已 flush 但未 commit"的 T1，判定可并行也是正确的。

### 区别在并行度

group commit 有"先批量 flush 再批量 commit"的特性。同组事务的 flush 是串行的（`m_transaction_counter` 递增），但 commit 是批量延迟的（整组 flush+sync 完才统一 commit，`m_max_committed` 一次性跳到组尾）：

- **prepared 方案**：把同组内已 flush 但还没 commit 的同伴事务误当作依赖 → 同组事务互相串行化 → 损失并行度
- **lock-based**：同组事务读到的 `m_max_committed` 都是组前的值 → last_committed 相同 → 从库可全部并行

`binlog.cc:2567` 的 `store_commit_parent(get_max_committed_timestamp())` 读的是 `m_max_committed_transaction` 而非 `m_transaction_counter`，这正是 lock-based 的关键。

---

## False Positive 与安全性

### 概念

- **False positive（假冲突）**：实际无冲突但被判为有冲突 → 串行化。系统判定的冲突集合是真正冲突集合的**超集**——所有真冲突都被覆盖（不遗漏），额外多包了一些其实不冲突的。
- **False negative**：实际有冲突但被判为可并行 → 从库并行执行导致数据不一致。

### 安全性分析

False positive 只影响性能（降低并行度），不影响正确性——可以接受。False negative 会导致数据不一致——不可接受。

MTS 设计铁律：**宁可 false positive，绝不 false negative**。

### 不重叠的两种情况

lock interval 不重叠有真冲突和假冲突两种：

- **真冲突**：T2 被 T1 行锁阻塞，确实需串行
- **假冲突**：T2 与 T1 无锁冲突但恰在 T1 放锁后才拿锁，实际可安全并行

COMMIT_ORDER 不区分两者，一律串行——这就是"悲观"的含义。

### 三种模式的 false positive 程度

| 模式 | False positive | 说明 |
|------|---------------|------|
| COMMIT_ORDER | 最多 | 不区分真/假冲突，一律串行 |
| WRITESET | 最少 | 行哈希确认不交集才判可并行，捞出假冲突；任何不确定回退 CO |
| WRITESET_SESSION | 介于两者 | 在 WRITESET 基础上加回同 session 的 false positive |

所有模式都保证**零 false negative**——WRITESET 只在确认无冲突时才判定可并行，任何不确定性都回退 COMMIT_ORDER。

---

## 从库调度机制

### GAQ（Group Assigned Queue）

核心数据结构 `Slave_committed_queue`（rpl_rli_pdb.h:351），`Slave_job_group` 的环形缓冲。每个 entry（:110）记录一个被调度的事务：

```cpp
struct Slave_job_group {
    longlong last_committed;          // 从 Gtid_log_event 读出
    longlong sequence_number;         // 从 Gtid_log_event 读出
    std::atomic<int32> done;          // Worker 完成后置位，Coordinator 读
    ulong worker_id;                  // 被分配给的 worker
    Slave_worker *worker;
    // ... checkpoint 相关字段
};
```

Coordinator 读到 Gtid_log_event 时在 GAQ 分配 entry 存这两个值。Worker 执行完后置 `done` 标志。GAQ 同时用于 checkpoint 和 LWM 计算。

### schedule_next_event：调度核心

`Mts_submode_logical_clock::schedule_next_event`（rpl_mta_submode.cc:574）是 coordinator 处理每个事件的核心。

**提取时间戳**（:591-598）：从 Gtid_log_event 读 last_committed / sequence_number。

**一致性检查**（:614-627）：sequence_number 必须 > last_committed 且 > 上一个 sequence_number（单调递增），否则报 `ER_MTA_CANT_PARALLEL`。

**gap 检测**（:635）：若 `sequence_number > last_sequence_number + 1`，标记 `gap_successor`，要等之前所有调度完成才执行。

**is_new_group 判定**（:651-680）：满足任一条件开启新组（串行等待所有 worker 完成）：
- `first_event`（submode 切换后首个事件）
- `force_new_group`
- `sequence_number == SEQ_UNINIT`（事件无时间戳）
- `last_committed == SEQ_UNINIT`（老主库无 commit_parent）
- `gap_successor`
- `last_sequence_number == SEQ_UNINIT`

### LWM 与并行判定

**核心逻辑在 `!is_new_group` 分支**（:685-716）：

```cpp
if (!is_new_group) {
    longlong lwm_estimate = estimate_lwm_timestamp();
    if (!clock_leq(last_committed, lwm_estimate) &&   // last_committed > LWM?
        rli->gaq->assigned_group_index != rli->gaq->entry) {
        if (wait_for_last_committed_trx(rli, last_committed))  // 阻塞等待
            return -1;
    }
    delegated_jobs++;
}
```

**从库用 LWM 整体判定，不是逐对比较 last_committed 和 sequence_number**：

- `last_committed <= LWM` → 本事务依赖的所有事务都已完成 → 可立即并行分发给空闲 worker
- `last_committed > LWM` → 还有依赖事务没跑完 → `wait_for_last_committed_trx` 阻塞等待

`last_committed` 相同的一批事务，当 LWM 推进到该值时，可全部同时分发并行执行。

### clock_leq 比较函数

```cpp
static bool clock_leq(longlong a, longlong b) {
    if (a == SEQ_UNINIT) return true;      // SEQ_UNINIT 视为最小
    else if (b == SEQ_UNINIT) return false;
    else return a <= b;
}
```

### get_lwm_timestamp：计算低水位

`get_lwm_timestamp`（cc:433）扫描 GAQ 找已完成的最低 sequence_number：`find_lwm` 从 GAQ 某索引开始找第一个 `done` 标志未置位的项的前一项——即连续已完成事务中 sequence_number 最小的那个。

### wait_for_last_committed_trx：阻塞等待

当依赖未满足时（cc:515），coordinator 在 `logical_clock_cond` 条件变量上 `cond_wait`：

```cpp
min_waited_timestamp.store(last_committed_arg);
if (!clock_leq(last_committed_arg, get_lwm_timestamp(rli, true))) {
    thd->ENTER_COND(&rli->logical_clock_cond, &rli->mts_gaq_LOCK, ...);
    do {
        mysql_cond_wait(&rli->logical_clock_cond, &rli->mts_gaq_LOCK);
    } while (!clock_leq(last_committed_arg, estimate_lwm_timestamp()));
}
```

Worker 完成事务后推进 LWM 并 `signal` 条件变量，唤醒等待的 coordinator。

### Worker 执行

`Slave_worker::slave_worker_exec_event`（rpl_rli_pdb.cc:1660）设置 thd 上下文后执行事件。同一事务内的所有事件由同一个 worker 串行执行，保证事务内一致性。不同事务的 event 可能分给不同 worker 并行。

### Commit Order Manager

`replica_preserve_commit_order = ON` 时，`Commit_order_manager`（rpl_replica_commit_order_manager.h:198）保证 worker commit 顺序与源一致——执行可并行但落盘有序。

实现用 `Commit_order_queue` + MDL 基础设施：worker 完成事务执行后，若还没轮到自己提交，在 MDL 上等待前序 worker 提交完毕；同时能检测 worker 之间的 commit 顺序死锁。仅从库生效。

---

## Writeset 生成

### Rpl_transaction_write_set_ctx

`Rpl_transaction_write_set_ctx`（rpl_transaction_write_set_ctx.cc）维护每个事务的 writeset。`add_write_set`（:64）在 InnoDB handler 层每修改一行时被调用，计算该行主键的哈希（基于 `transaction_write_set_extraction` 参数指定的算法，如 XXHASH64）加入 `write_set` 向量。

### 双用途

1. **WRITESET 依赖跟踪**：`Writeset_trx_dependency_tracker` 查 `m_writeset_history` 压低 commit_parent
2. **MGR 冲突检测认证**：写入 `Transaction_context_log_event`（log_event.cc:13871）随事务广播给组成员做 certification

### 容量限制

`add_write_set` 有容量限制（:68-76）：当 write_set 大小超过 `binlog_transaction_dependency_history_size` 时，标记 `m_local_has_reached_write_set_limit`，该事务后续在 WRITESET 模式会回退 COMMIT_ORDER。

---

## 生产实践要点

- **slave_parallel_workers=1 比 0 差约 20%**：设为 1 时 SQL thread 变为 coordinator 但只有 1 个 worker，多了一次转发开销
- **级联复制并行度衰减**：LOGICAL_CLOCK 可能使离 master 越远的 slave 并行性越差
- **slave_preserve_commit_order 版本要求**：5.7.18 无法保证提交顺序一致，5.7.19 才修复，生产环境须 >= 5.7.19
- **主库负载低时退化**：组提交效率不高时每组可能只有 1 个事务，从库开启并行复制性能反而比单线程差

---

## 生产案例：无主键表引发的同步延迟

> 案例来源：腾讯云开发者社区（cloud.tencent.com/developer/article/1688866）

### 问题现象

ROW 模式 binlog 下，无主键大表执行批量 UPDATE/DELETE，灾备实例、备库、只读实例均出现巨大同步延迟。binlog 落后 size 可能不大，但主从延迟时间不为 0 且呈稳定上升趋势。

### 根因：N 个 Row Event × 每行定位成本

**核心是 ROW 模式把一条 UPDATE 拆解成 N 个 Row Event**。一条批量 SQL 影响 N 行，ROW 模式就生成 N 个独立的行事件，从库必须逐条回放。这个 N 的数量是固定的、结构性的，与有没有主键无关。

```
主库: UPDATE t SET col = x          ← 一条 SQL，一次执行
       ↓ (ROW 模式 binlog)
binlog: Row Event 1 (BI=旧值1, AI=新值1)
        Row Event 2 (BI=旧值2, AI=新值2)
        ...
        Row Event N (BI=旧值N, AI=新值N)    ← N 行 = N 个事件
```

**无主键是放大器**，把每个事件的处理成本从 O(log N) 放大到 O(N)：

| | 有主键 | 无主键 |
|---|---|---|
| 事件数量 | N 个（固定） | N 个（固定） |
| 每行定位 | `ha_index_read_map` O(log N) | `ha_rnd_init` + `ha_rnd_next` O(N) |
| 总成本 | O(N log N) | O(N²) |
| N=10000 示例 | ~17 万次操作 | ~1 亿次操作 |

**N 个事件是放大基数，无主键是放大倍数。** 如果只有"N 个事件"没有"无主键"，每个事件 O(log N) 也很快；如果只有"无主键"没有"大量事件"，一条 UPDATE 只影响 1 行，1 次全表扫描也能忍。两者叠加才爆炸。

主库执行 UPDATE 时，即使全表扫描也是**一次 SQL 执行**——optimizer 的扫描迭代器一次性扫完所有行，匹配的行直接修改，行定位由迭代器自然完成。不存在"逐条回放 N 个事件"的问题。从库不执行 SQL，而是回放 N 个独立的 Row Event，每个事件都要独立完成"定位 + 修改"，没有主库的迭代器上下文可以复用。

STATEMENT 模式 binlog 下从库直接重放 SQL，和主库一样慢但不会更慢。ROW 模式逐行回放 + 无主键逐行全表扫描定位，才产生 N² 放大效应。

### 无主键表对复制的三重影响

**1. 行定位慢（本案例直接原因）**

从库回放 Row Event 时通过 before-image 定位行。定位方式由 `slave_rows_search_algorithms` 控制（8.0.26 默认 `INDEX_SCAN,HASH_SCAN`）：

- 有主键：主键索引直接定位（`INDEX_SCAN`），O(log N)
- 无主键有 HASH_SCAN：通过主键哈希缓存行位置，仍需主键/唯一索引
- 无主键无索引：全表扫描（`TABLE_SCAN`），O(N) 每行

**2. binlog_row_image 无法 MINIMAL**

MINIMAL 模式的 before-image 只记录主键列，无主键就无法唯一标识行。无主键表只能 FULL 或 NOBLOB，binlog 体积更大。这印证了 `mark_columns_per_binlog_row_image` 中"无主键则 read_set 全部置位"的代码逻辑。

**3. WRITESET 依赖跟踪降级**

writeset 基于主键哈希计算，无主键表没有 writeset（`writeset->size() == 0`），WRITESET 模式**回退到 COMMIT_ORDER**（rpl_trx_tracking.cc:232-247 的降级条件），从库并行度下降。

### 解决方案

给表添加主键或唯一索引（可用自增列），然后重建受影响的从库实例：

```sql
-- 检查无主键表
SELECT table_schema, table_name, TABLE_ROWS
  FROM information_schema.tables
 WHERE (table_schema, table_name) NOT IN
       (SELECT DISTINCT table_schema, table_name
          FROM information_schema.columns
         WHERE COLUMN_KEY = 'PRI')
   AND table_schema NOT IN ('sys','mysql','information_schema','performance_schema')
   AND table_type = 'BASE TABLE';

-- 添加主键
ALTER TABLE tmp1 ADD COLUMN id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY FIRST;
```

### 核心结论

无主键表在复制的全链路上都是性能杀手：binlog 体积大（无法 MINIMAL）、从库行定位慢（全表扫描 N 次）、MTS 并行度低（WRITESET 回退）。主键不仅是查询优化手段，更是复制体系的基础设施。

---

## 关键源码位置速查

| 函数/变量 | 文件:行号 | 作用 |
|-----------|----------|------|
| `Logical_clock` | rpl_trx_tracking.h:45 | 逻辑时钟类（state + offset） |
| `Logical_clock::step()` | rpl_trx_tracking.cc:65 | ++state 分配序号 |
| `Logical_clock::set_if_greater()` | rpl_trx_tracking.cc:83 | CAS 推高已提交水位 |
| `Commit_order_trx_dependency_tracker` | rpl_trx_tracking.h:88 | COMMIT_ORDER 模式 |
| `m_max_committed_transaction` | rpl_trx_tracking.h:114 | 已提交时钟 |
| `m_transaction_counter` | rpl_trx_tracking.h:117 | 已 flush 时钟 |
| `m_last_blocking_transaction` | rpl_trx_tracking.h:125 | 屏障事务序号 |
| `Commit_order_trx_dependency_tracker::get_dependency` | rpl_trx_tracking.cc:149 | COMMIT_ORDER 生成依赖 |
| `Writeset_trx_dependency_tracker::get_dependency` | rpl_trx_tracking.cc:213 | WRITESET 压低 commit_parent |
| `Writeset_session_trx_dependency_tracker::get_dependency` | rpl_trx_tracking.cc:315 | WRITESET_SESSION 加 session 约束 |
| `Transaction_dependency_tracker::get_dependency` | rpl_trx_tracking.cc:331 | 按模式分发 |
| `update_max_committed` | rpl_trx_tracking.cc:382 | commit 时推高 max_committed |
| `rotate()` | rpl_trx_tracking.cc:185 | binlog rotate 更新 offset |
| `store_commit_parent()` | transaction_info.h:228 | 记录 last_committed |
| `binlog_cache_data::flush` | binlog.cc:2396 | flush 时分配 sequence_number |
| `write_transaction` | binlog.cc:1665 | 生成依赖写入 Gtid_log_event |
| `store_commit_parent` 调用 | binlog.cc:2567 | 读 m_max_committed 赋值 last_committed |
| `process_commit_stage_queue` | binlog.cc:8513 | commit stage 有序引擎提交 |
| `update_max_committed` 调用 | binlog.cc:8537 | commit 后推高水位 |
| `Mts_submode_logical_clock` | rpl_mta_submode.h:129 | 逻辑时钟 submode |
| `schedule_next_event` | rpl_mta_submode.cc:574 | 从库调度核心 |
| `get_lwm_timestamp` | rpl_mta_submode.cc:433 | 计算 LWM |
| `wait_for_last_committed_trx` | rpl_mta_submode.cc:515 | 阻塞等待依赖完成 |
| `clock_leq` | rpl_mta_submode.h:198 | 逻辑时钟比较 |
| `Slave_job_group` | rpl_rli_pdb.h:110 | GAQ entry 结构 |
| `Slave_committed_queue` | rpl_rli_pdb.h:351 | GAQ 环形缓冲 |
| `slave_worker_exec_event` | rpl_rli_pdb.cc:1660 | Worker 执行事件 |
| `Commit_order_manager` | rpl_replica_commit_order_manager.h:198 | 提交顺序管理器 |
| `Rpl_transaction_write_set_ctx::add_write_set` | rpl_transaction_write_set_ctx.cc:64 | 添加行哈希到 writeset |
| `Writeset_history` | rpl_trx_tracking.h:168 | 行哈希 → sequence_number 映射 |
