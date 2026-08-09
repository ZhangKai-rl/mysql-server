# MySQL Binlog 并行复制深度解析

## —— 从主库依赖追踪到从库 MTS 调度的源码级分析

> 基于 MySQL 8.0 源码分析（`sql/binlog.cc`, `sql/rpl_trx_tracking.cc`, `sql/rpl_mta_submode.cc`, `sql/log_event.cc` 等模块）
> 生成时间: 2026-07-21

---

## 摘要

MySQL 的主从复制是构建高可用架构的基石。自 5.7 版本引入 LOGICAL_CLOCK 并行复制后，从库不再是单线程串行回放，而是可以根据主库写入 binlog 的逻辑时间戳（`last_committed` 与 `sequence_number`），在多个 worker 线程上安全地并行执行不冲突的事务。本文从源码级深度出发，完整剖析主库侧三种依赖追踪模式（COMMIT_ORDER / WRITESET / WRITESET_SESSION）的计算原理，以及从库侧 `Mts_submode_logical_clock` 调度器如何利用这些时间戳实现低冲突并行回放。涵盖 Lock Interval 理论、Logical Clock 设计、GAQ（Global Assigned Queue）的 LWM 维护机制、以及 COMMIT_ORDER 过度保守性的根源分析。

**关键问题回答：**
- LOGICAL_CLOCK 并行复制**是** MySQL 5.7 引入的，8.0 中持续存在且为默认模式
- 两个逻辑时间戳 `sequence_number` 在 **flush 阶段**分配，`last_committed` 在**事务获取所有锁的瞬间**记录（通过对 `m_max_committed_transaction` 的快照）
- 并行性的保证来自主库的 **锁间隔（Lock Interval）重叠判定**：如果 T2 获取锁时 T1 尚未释放锁（锁间隔重叠），则两者不存在锁冲突，从库可以安全并行
- 即使判定为冲突，从库也**不能**让它们"并行加锁等待"，因为 binlog 不包含锁信息，乱序执行会破坏数据一致性

---

## 目录

1. [第一章：整体架构](#第一章整体架构)
2. [第二章：核心概念 —— 两个逻辑时间戳](#第二章核心概念--两个逻辑时间戳)
3. [第三章：主库侧 —— 依赖追踪](#第三章主库侧--依赖追踪)
   - 3.1 三种依赖追踪模式
   - 3.2 COMMIT_ORDER 详解：Lock Interval 理论
   - 3.3 WRITESET 详解：行级冲突检测
   - 3.4 WRITESET_SESSION 详解：会话级约束
   - 3.5 三层漏斗模型
   - 3.6 写入 binlog：序列化到 Gtid_log_event
4. [第四章：从库侧 —— MTS 调度](#第四章从库侧--mts-调度)
   - 4.1 两种调度模式
   - 4.2 LOGICAL_CLOCK 调度流程
   - 4.3 LWM（Low Water Mark）维护
   - 4.4 Worker 分配策略
5. [第五章：完整时序图](#第五章完整时序图)
6. [第六章：关键参数](#第六章关键参数)
7. [第七章：常见问题](#第七章常见问题)
   - Q1: COMMIT_ORDER 为什么"锁间隔重叠"就能保证不冲突？
   - Q2: 既然冲突也可以让从库"并行加锁等待"，为什么不这样做？
   - Q3: COMMIT_ORDER 的"假阳性"到底有多严重？
   - Q4: sequence_number 和 binlog 中的 last_committed 为什么是"相对值"？
   - Q5: 一张表有外键，WRITESET 为什么必须降级？
   - Q6: COMMIT_ORDER 是一个保守策略吗？社区有优化吗？
   - Q7: Lock Interval 和 sequence_number/last_committed 是一回事吗？
   - Q8: Lock Interval 为什么说"prepare 到 commit"？
   - Q9: LWM 是什么的缩写？
   - Q10: 为什么 INSERT 一行产生三个 PKE？
   - Q11: 为什么 sequence_number 用 state - offset 而不是全局 state？
   - Q12: writeset 的三个 hash 是行级粒度的确认
   - Q13: 不同 binlog 的事务一定能并行吗？offset 是优化还是语义要求？
   - Q14: Causal Consistency（因果一致性）在并行复制中意味着什么？
8. [第八章：WL#7165 经典图解详细分析](#第八章wl7165-经典图解详细分析)
   - 8.1 图解
   - 8.2 逐个事务分析
   - 8.3 关键观察
   - 8.4 从库执行顺序
9. [第九章：Writeset Hash 生成机制](#第九章writeset-hash-生成机制)
   - 9.1 核心函数：add_pke
   - 9.2 为什么 PKE 能找到更多可并行事务
   - 9.3 唯一键也被纳入的原因
10. [第十章：GAQ 与 Checkpoint 机制](#第十章gaq-与-checkpoint-机制)
   - 10.1 GAQ 数据结构
   - 10.2 find_lwm：查找低水位线
   - 10.3 move_queue_head：Checkpoint 核心
   - 10.4 mta_checkpoint_routine：定时 Checkpoint
   - 10.5 Checkpoint 对并行度的影响
11. [第十一章：理论基础 —— 2PL 与锁释放时机](#第十一章理论基础--2pl-与锁释放时机)
   - 11.1 InnoDB 的 Two-Phase Locking
   - 11.2 锁释放的调用链
   - 11.3 lock_trx_release_locks 源码
   - 11.4 trx_commit_in_memory 中锁释放与状态切换
   - 11.5 锁释放时机与 BGC 的关系
12. [第十二章：一致性模型详解](#第十二章一致性模型详解)
   - 12.1 从故事的直觉出发
   - 12.2 强一致性（Linearizability / Strict Consistency）
   - 12.3 最终一致性（Eventual Consistency）
   - 12.4 因果一致性（Causal Consistency）
   - 12.5 三种一致性在行为上的对照
   - 12.6 为什么并行复制的目标是因果一致性而非最终一致性
   - 12.7 LOGICAL_CLOCK 三种模式与因果一致性的检测精度
   - 12.8 COMMIT_ORDER 能否保证因果一致性？
   - 12.9 WRITESET 的假阴性：什么时候 WRITESET 会错误地抹掉因果依赖？
   - 12.10 replica_preserve_commit_order：不检测因果，只强制顺序
   - 12.11 因果不一致的具体后果
   - 12.12 GTID 空洞与因果一致性：两个维度的概念
   - 12.13 四层保障总结（修正版）
   - 12.14 replica_preserve_commit_order 的归属：纯从库参数
   - 12.15 非 RBR 下 WRITESET 的数据一致性风险
   - 12.16 GTID 空洞与 preserve_commit_order：历史与演进
   - 12.17 Commit_order_queue 内部机制：sequence_nr 的真实作用
   - 12.18 preserve_commit_order 的真正串行化位置：BGC Stage#0 vs ha_commit_low

---

## 第一章：整体架构

并行复制的完整数据流贯穿主库和从库：

```
┌─────────────────────────────────────┐      ┌─────────────────────────────────────┐
│              Master（主库）           │      │            Slave（从库）              │
│                                     │      │                                     │
│  binlog_transaction_dependency_     │      │  replica_parallel_type               │
│  tracking = COMMIT_ORDER | WRITESET │      │  = LOGICAL_CLOCK | DATABASE          │
│  | WRITESET_SESSION                 │      │  replica_parallel_workers = N        │
│                                     │      │                                     │
│  ┌───────────────────────────────┐  │      │  ┌─────────────────────────────────┐ │
│  │ Transaction_dependency_      │  │      │  │ Mts_submode_logical_clock        │ │
│  │ tracker                      │  │      │  │ (Coordinator 调度器)              │ │
│  │  ├─ Commit_order             │  │      │  │                                 │ │
│  │  ├─ Writeset                 │  │      │  │ schedule_next_event()            │ │
│  │  └─ Writeset_session         │  │      │  │   ↓                             │ │
│  └───────────┬───────────────────┘  │      │  │ wait_for_last_committed_trx()   │ │
│              │                       │      │  │   ↓                             │ │
│              ▼                       │      │  │ get_least_occupied_worker()     │ │
│  ┌───────────────────────────────┐  │      │  │   ↓                             │ │
│  │ write_transaction()           │  │      │  │ Worker-1  Worker-2 ... Worker-N  │ │
│  │  → Gtid_log_event             │  │      │  └─────────────────────────────────┘ │
│  │    last_committed             │  │      │                                     │
│  │    sequence_number            │  │ binlog│  → relay log → Coordinator → Worker │
│  └───────────────────────────────┘  │      │                                     │
└─────────────────────────────────────┘      └─────────────────────────────────────┘
```

### 涉及的核心源码文件

| 侧 | 文件 | 职责 |
|---|---|---|
| 主库 | [rpl_trx_tracking.h](sql/rpl_trx_tracking.h) | 依赖追踪器接口：`Logical_clock`、`Commit_order_trx_dependency_tracker`、`Writeset_trx_dependency_tracker`、`Writeset_session_trx_dependency_tracker`、`Transaction_dependency_tracker` |
| 主库 | [rpl_trx_tracking.cc](sql/rpl_trx_tracking.cc) | 依赖追踪器实现，三种模式的 `get_dependency()` 逻辑 |
| 主库 | [binlog.cc](sql/binlog.cc) | `write_transaction()` 调用 `get_dependency()`，写入 GTID event；`binlog_cache_data::flush()` 中分配 `sequence_number`；多处 `store_commit_parent()` |
| 主库 | [log_event.cc](sql/log_event.cc) | `Gtid_log_event::write_post_header_to_memory()` 将 `last_committed` 和 `sequence_number` 序列化到 binlog 的 GTID 事件中 |
| 主库 | [transaction_info.h](sql/transaction_info.h) | `Transaction_ctx`：`sequence_number`、`last_committed` 字段定义；`store_commit_parent()` 方法 |
| 从库 | [rpl_mta_submode.h](sql/rpl_mta_submode.h) | MTS 调度子模式接口定义：`Mts_submode`、`Mts_submode_database`、`Mts_submode_logical_clock` |
| 从库 | [rpl_mta_submode.cc](sql/rpl_mta_submode.cc) | `Mts_submode_database` 和 `Mts_submode_logical_clock` 的完整实现，包含调度、等待、LWM 计算等核心逻辑 |

---

## 第二章：核心概念 —— 两个逻辑时间戳

每个事务在 binlog 中携带两个逻辑时间戳，嵌入在 `Gtid_log_event` 的 **post-header** 中：

### sequence_number（事务序列号）

- **含义**：事务在 binlog 中的全局自增序号，唯一标识一个事务
- **分配时机**：`binlog_cache_data::flush()` → `m_dependency_tracker.step()` → `m_transaction_counter.step()` → `++state`
- **物理存储**：Gtid_log_event post-header 的最后 8 字节

```cpp
// binlog.cc:2419
// for MTS. sequence_number::state++
trn_ctx->sequence_number = mysql_bin_log.m_dependency_tracker.step();
```

### last_committed（commit parent）

- **含义**：当前事务必须等待的"父事务"的 `sequence_number`。从库必须等到 `last_committed` 号事务提交完成后，才能开始执行当前事务
- **记录时机**：事务获取完所有锁的那一刻，调用 `store_commit_parent()` 对当前 `m_max_committed_transaction` 做快照
- **物理存储**：Gtid_log_event post-header 的倒数第 9-16 字节

```cpp
// transaction_info.h:228
void store_commit_parent(int64 last_arg) { last_committed = last_arg; }
```

### 调用时机

`store_commit_parent` 在多处被调用，对应不同的事务类型：

| 位置 | 场景 |
|---|---|
| [binlog.cc:2567](sql/binlog.cc) | XA prepare（第一阶段提交） |
| [binlog.cc:2897](sql/binlog.cc) | 普通事务，`flush_thread_caches` 时（锁已全部获取） |
| [binlog.cc:8164](sql/binlog.cc) | 非事务型语句（如 DDL） |

### 关键：绝对时间 vs 相对时间

`Logical_clock` 内部维护了 `state`（绝对时钟）和 `offset`（偏移量）：

```cpp
// rpl_trx_tracking.h:57-76
class Logical_clock {
private:
  std::atomic<int64> state;   // 全局绝对时钟
  int64 offset;               // binlog 轮转时的偏移量
```

- `Transaction_ctx::sequence_number` 和 `Transaction_ctx::last_committed` 存储的是**绝对值**（state 的值）
- 写入 binlog 时，`get_dependency()` 会减去 `offset` 转为**相对值**，使得 binlog 中的值总是从 1 开始递增
- binlog 轮转后，`offset` 更新为新 binlog 的起点，旧相对值失去意义，`last_committed` 回到 `SEQ_UNINIT`

---

## 第三章：主库侧 —— 依赖追踪

### 3.1 三种依赖追踪模式

由参数 `binlog_transaction_dependency_tracking` 控制。定义在 [rpl_trx_tracking.h:191-215](sql/rpl_trx_tracking.h)：

| 模式 | 枚举值 | 含义 |
|---|---|---|
| `COMMIT_ORDER` | 0 | 基于事务提交顺序和锁持有时间窗口判定冲突 |
| `WRITESET` | 1 | 在 COMMIT_ORDER 基础上，额外使用行级 hash 检测 |
| `WRITESET_SESSION` | 2 | 在 WRITESET 基础上，强制同一 session 的事务串行 |

### 3.2 COMMIT_ORDER 详解：Lock Interval 理论

这是 LOGICAL_CLOCK 并行的**理论基础**。源码注释给出了完整定义（[rpl_trx_tracking.h:191-200](sql/rpl_trx_tracking.h)）：

> The time intervals during which any transaction holds all its locks are tracked (the interval ends just before storage engine commit, when locks are released. For an autocommit transaction it begins just before storage engine prepare. For BEGIN..COMMIT transactions it begins at the end of the last statement before COMMIT). **Two transactions are marked as non-conflicting if their respective intervals overlap.** In other words, if trx1 appears before trx2 in the binlog, and trx2 had acquired all its locks before trx1 released its locks, then trx2 is marked such that the slave can schedule it in parallel with trx1.

#### Lock Interval 的定义

一个事务的 Lock Interval 是从"获取完所有行锁"到"存储引擎提交（释放锁）"之间的时间窗口：

- **autocommit 事务**：从存储引擎 prepare 开始，到存储引擎 commit（释放锁）结束
- **BEGIN...COMMIT 显式事务**：从最后一条语句执笔结束，到 COMMIT 结束

#### 判定规则图解（精确时间线）

以下图例使用源码中的精确函数调用点来标注时机：

```
情况A：锁间隔重叠 → store_commit_parent 时 m_max 尚未更新 → 无冲突 → 可以并行

T1:  ──获取锁──[══════ Lock Interval ══════]──prepare──flush(step→seq=1)──sync──commit(m_max→1)
                 ↑                                                        ↑
          store_commit_parent(0)                                   update_max_committed(1)
          （此时前一个已提交的 seq 是 0）

T2:     ──获取锁──[══════ Lock Interval ══════]──prepare──flush(step→seq=2)──sync──commit
                  ↑
           store_commit_parent(?)
           │ T2 获取锁时 T1 还在 Lock Interval 里（未 commit，m_max 还是 0）
           │ → m_max = 0 → last_committed = 0
           │ → lc[2]=0 < seq[1]=1 → 不依赖 T1 → 写入 binlog: lc=0 ✓
```

```
情况B：锁间隔不重叠 → store_commit_parent 时 m_max 已更新 → 可能冲突 → 必须串行

T1:  ──获取锁──[══ Lock Interval ══]──prepare──flush(step→seq=1)──sync──commit(m_max→1)
                                                          ↑
                                                  update_max_committed(1)

T2:                                ──获取锁──[══ Lock Interval ═══]──prepare──flush(step→seq=2)
                                     ↑
                              store_commit_parent(?)
                              │ T2 获取锁时 T1 已经 commit（m_max = 1）
                              │ → 无法证明 T2 的锁 ≠ T1 的锁
                              │ → 保守标记：last_committed = 1
                              │ → lc[2]=1 ≥ seq[1]=1 → 依赖 T1 → 写入 binlog: lc=1 ✗
```

**核心公式**：从库能否并行执行 T2 和 T1，取决于：

```
lc[T2] < seq[T1]   →  T2 不依赖 T1  →  可以与 T1 并行 ✓
lc[T2] >= seq[T1]  →  T2 必须等 T1  →  不能与 T1 并行 ✗
```

**为什么不是看"何时进入 prepare"？** 因为 `store_commit_parent` 的调用发生在获取锁之后、prepare 之前（或最后一条语句结束时），prepare 本身只是记录事务状态的节点，与依赖判定无关。真正决定 last_committed 值的是**调用 `store_commit_parent` 那一刻 `m_max_committed_transaction` 的值**。

#### 量化逻辑

```cpp
// Commit_order_trx_dependency_tracker 内部维护两个时钟：
// rpl_trx_tracking.h:130-137
Logical_clock m_max_committed_transaction;  // 已提交的最大事务号
Logical_clock m_transaction_counter;         // 已分配的最大事务号
```

状态变化追踪：

| 时刻 | T1 状态 | T2 状态 | m_max_committed | m_transaction_counter |
|---|---|---|---|---|
| T1 获取锁 | Lock Interval 开始 | - | 0 | 0 |
| T1 flush | seq=step()=1 | - | 0 | 1 |
| T1 commit | 释放锁 | - | 1 (更新) | 1 |
| T2 获取锁 | - | store_commit_parent(m_max=0 或 1) | 1 或 0 | 1 |
| T2 flush | - | seq=step()=2 | ... | 2 |

**关键**：T2 的 `last_committed` 等于 T2 获取锁时 `m_max_committed` 的值。如果此时 m_max=0（T1 未提交），则 lc=0；如果此时 m_max=1（T1 已提交），则 lc=1。

#### COMMIT_ORDER 的保守性

情况B 中，T1 和 T2 的锁间隔不重叠，但这**不意味着它们一定有行级冲突**。它们可能修改的是完全不相关的行（不同表、不同 ID）。COMMIT_ORDER 保守地标记为串行，从而产生了**假阳性**（false positive）。这正是 WRITESET 模式要解决的问题。

### 3.3 WRITESET 详解：行级冲突检测

WRITESET 模式在 COMMIT_ORDER 计算出 `commit_parent` 的基础上，使用行级 hash 进一步缩小依赖范围。

**核心数据结构**（[rpl_trx_tracking.h:157-163](sql/rpl_trx_tracking.h)）：

```cpp
typedef std::map<uint64, int64> Writeset_history;
Writeset_history m_writeset_history;  // 行hash → 最后修改该行的seq_no
int64 m_writeset_history_start;       // history 覆盖的最小编号
std::atomic<ulong> m_opt_max_history_size;  // history 最大容量，默认 25000
```

**算法**（[rpl_trx_tracking.cc:213-300](sql/rpl_trx_tracking.cc)）：

```cpp
void Writeset_trx_dependency_tracker::get_dependency(THD *thd,
                                                     int64 &sequence_number,
                                                     int64 &commit_parent) {
  std::vector<uint64> *writeset = write_set_ctx->get_write_set();
  
  // Step 1: 检查是否可以使用 writeset
  bool can_use_writesets =
      // 非空事务，或有 missing keys 的，或空事务
      (writeset->size() != 0 || write_set_ctx->get_has_missing_keys() ||
       is_empty_transaction_in_binlog_cache(thd)) &&
      // hash 算法一致（session 和 global 的 transaction_write_set_extraction 相同）
      (global_system_variables.transaction_write_set_extraction ==
       thd->variables.transaction_write_set_extraction) &&
      // 无外键级联
      !write_set_ctx->get_has_related_foreign_keys() &&
      // 未超过 writeset limit
      !write_set_ctx->was_write_set_limit_reached();

  // Step 2: 遍历当前事务修改的每一行的 hash
  int64 last_parent = m_writeset_history_start;
  for (each row_hash in writeset) {
    auto hst = m_writeset_history.find(row_hash);
    if (hst != m_writeset_history.end()) {
      // 找到了冲突行！更新 last_parent
      if (hst->second > last_parent && hst->second < sequence_number)
        last_parent = hst->second;
      hst->second = sequence_number;  // 更新为该行的最新 seq_no
    } else {
      m_writeset_history.insert({row_hash, sequence_number});
    }
  }

  // Step 3: 取 min（WRITESET 判定的冲突 vs COMMIT_ORDER 判定的冲突）
  commit_parent = std::min(last_parent, commit_parent);
}
```

**关键设计**：最终取 `std::min(last_parent, commit_parent)`——writeset 只能让依赖变得**更窄**（更小），绝不会让依赖变大。这是安全的。

**降级条件**：
- 外键级联（`get_has_related_foreign_keys()`）→ 回退到 COMMIT_ORDER
- 无主键表（`get_has_missing_keys()`）→ 对涉及该表的行回退到 COMMIT_ORDER
- hash 算法不一致（session 变量和 global 变量不同）→ 回退
- writeset history 溢出（超过 `binlog_transaction_dependency_history_size`）→ 清空 history 重新开始

### 3.4 WRITESET_SESSION 详解：会话级约束

在 WRITESET 基础上，额外保证**同一 session 内的事务必须串行**：

```cpp
// rpl_trx_tracking.cc:331-339
void Writeset_session_trx_dependency_tracker::get_dependency(
    THD *thd, int64 &sequence_number, int64 &commit_parent) {
  int64 session_parent = thd->rpl_thd_ctx.dependency_tracker_ctx()
                             .get_last_session_sequence_number();
  if (session_parent != 0 && session_parent < sequence_number)
    commit_parent = std::max(commit_parent, session_parent);  // 收紧！
  thd->rpl_thd_ctx.dependency_tracker_ctx()
      .set_last_session_sequence_number(sequence_number);
}
```

注意这里用的是 `std::max`——在 writeset 的结果上收紧依赖。

### 3.5 三层漏斗模型

`Transaction_dependency_tracker::get_dependency()` 是统一入口，三种模式构成**三层漏斗**：

```cpp
// rpl_trx_tracking.cc:344-377
void Transaction_dependency_tracker::get_dependency(...) {
  switch (m_opt_tracking_mode) {
    case DEPENDENCY_TRACKING_COMMIT_ORDER:
      m_commit_order.get_dependency(...);                        // 最宽泛
      break;
    case DEPENDENCY_TRACKING_WRITESET:
      m_commit_order.get_dependency(...);                        // 宽
      m_writeset.get_dependency(...);                            // → 收窄
      break;
    case DEPENDENCY_TRACKING_WRITESET_SESSION:
      m_commit_order.get_dependency(...);                        // 宽
      m_writeset.get_dependency(...);                            // → 收窄
      m_writeset_session.get_dependency(...);                    // → 最窄
      break;
  }
}
```

```
COMMIT_ORDER     ｜ 所有锁间隔不重叠的都标记为串行（假阳性多）     ｜ 并发度：★★☆
    ↓ 收窄
WRITESET          ｜ 用行级 hash 剔除假阳性                        ｜ 并发度：★★★
    ↓ 收窄
WRITESET_SESSION  ｜ 再加 session 约束（绝对正确性优先）            ｜ 并发度：★★★↓
```

### 3.6 写入 binlog：序列化到 Gtid_log_event

最终 `last_committed` 和 `sequence_number` 被 `Gtid_log_event::write_post_header_to_memory()` 序列化到 binlog（[log_event.cc](sql/log_event.cc)）：

```cpp
uint32 Gtid_log_event::write_post_header_to_memory(uchar *buffer) {
  uchar *ptr_buffer = buffer;
  
  // ... gtid_flags, SID, GNO 的写入 ...
  
  *ptr_buffer = LOGICAL_TIMESTAMP_TYPECODE;       // 标记为逻辑时间戳类型
  ptr_buffer += LOGICAL_TIMESTAMP_TYPECODE_LENGTH;

  // 保证：要么两个都是0（空事务），要么 sequence_number > last_committed
  assert((sequence_number == 0 && last_committed == 0) ||
         (sequence_number > last_committed));

  int8store(ptr_buffer, last_committed);          // 8字节: commit parent
  int8store(ptr_buffer + 8, sequence_number);     // 8字节: 自身序列号
}
```

**GTID event 的 post-header 物理布局：**

```
[gtid_flags(1B)] [SID(16B)] [GNO(8B)] [typecode(1B)] [last_committed(8B)] [sequence_number(8B)]
                                                                  ↑                   ↑
                                                            commit parent        自身序号
```

---

## 第四章：从库侧 —— MTS 调度

### 4.1 两种调度模式

由 `replica_parallel_type` 控制，定义在 [rpl_mta_submode.h:49-53](sql/rpl_mta_submode.h)：

| 模式 | 实现类 | 调度依据 | 并行度 |
|---|---|---|---|
| `DATABASE` | `Mts_submode_database` | 按 database name hash 分配 worker | 较低，跨库事务退化 |
| `LOGICAL_CLOCK` | `Mts_submode_logical_clock` | 基于主库写入的 `last_committed` / `sequence_number` | 较高，理论上可接近主库并发度 |

### 4.2 LOGICAL_CLOCK 调度流程

`Mts_submode_logical_clock` 的核心方法（[rpl_mta_submode.h:125-212](sql/rpl_mta_submode.h)）：

**关键成员变量：**

```cpp
longlong last_committed;       // 当前事务的 commit parent
longlong sequence_number;      // 当前事务的序列号
std::atomic<longlong> last_lwm_timestamp;  // 当前 LWM（Low Water Mark）
ulong last_lwm_index;          // LWM 对应的 GAQ index
bool is_new_group;             // 是否需要强制开始新的事务组（等所有worker完成）
```

**步骤1：提取时间戳（schedule_next_event）**（[rpl_mta_submode.cc:577-652](sql/rpl_mta_submode.cc)）

```cpp
int Mts_submode_logical_clock::schedule_next_event(Relay_log_info *rli, Log_event *ev) {
  switch (ev->get_type_code()) {
    case binary_log::GTID_LOG_EVENT:
    case binary_log::ANONYMOUS_GTID_LOG_EVENT:
      // 从 GTID event 中读出主库侧写入的 last_committed 和 sequence_number
      ptr_group->sequence_number = sequence_number =
          static_cast<Gtid_log_event *>(ev)->sequence_number;
      ptr_group->last_committed = last_committed =
          static_cast<Gtid_log_event *>(ev)->last_committed;
      break;
  }
```

**步骤2：判断是否需要 is_new_group（强制串行点）**（[rpl_mta_submode.cc:668-691](sql/rpl_mta_submode.cc)）

以下情况会触发 `is_new_group = true`，这意味着 Coordinator 必须**等待所有 worker 完成当前工作**后，才能开始调度下一个事务：

```cpp
is_new_group =
    first_event ||                      // 首次调度
    force_new_group ||                  // 强制（如 DDL、表结构变更）
    sequence_number == SEQ_UNINIT ||    // 主库未写入时间戳（老版本 binlog）
    last_committed == SEQ_UNINIT ||     // 无 commit parent（binlog 中的第一条事务）
    gap_successor ||                    // 序列号出现空洞（有缺失的事务）
    last_sequence_number == SEQ_UNINIT; // 上一个事务也没有时间戳
```

**步骤3：依赖判断与等待**（[rpl_mta_submode.cc:685-721](sql/rpl_mta_submode.cc)）

```cpp
if (!is_new_group) {
  longlong lwm_estimate = estimate_lwm_timestamp();  // 获取当前 LWM
  
  if (!clock_leq(last_committed, lwm_estimate) &&
      rli->gaq->assigned_group_index != rli->gaq->entry) {
    // last_committed > LWM → 依赖的事务还没全部提交 → 必须等待！
    if (wait_for_last_committed_trx(rli, last_committed)) {
      return -1;
    }
  }
}
```

**`clock_leq` 的定义**（[rpl_mta_submode.h:182-188](sql/rpl_mta_submode.h)）：

```cpp
static bool clock_leq(longlong a, longlong b) {
  if (a == SEQ_UNINIT) return true;   // SEQ_UNINIT 视为最小
  else if (b == SEQ_UNINIT) return false;
  else return a <= b;
}
```

**步骤4：等待的实现（wait_for_last_committed_trx）**（[rpl_mta_submode.cc:515-570](sql/rpl_mta_submode.cc)）

```cpp
bool Mts_submode_logical_clock::wait_for_last_committed_trx(
    Relay_log_info *rli, longlong last_committed_arg) {
  if (last_committed_arg == SEQ_UNINIT) return false;

  mysql_mutex_lock(&rli->mts_gaq_LOCK);
  min_waited_timestamp.store(last_committed_arg);
  
  // 条件等待：直到 LWM >= last_committed_arg
  thd->ENTER_COND(&rli->logical_clock_cond, ...);
  do {
    mysql_cond_wait(&rli->logical_clock_cond, &rli->mts_gaq_LOCK);
  } while (!clock_leq(last_committed_arg, estimate_lwm_timestamp()));
  thd->EXIT_COND(...);
  
  return false;
}
```

### 4.3 LWM（Low Water Mark）维护

LWM 是 MTS 调度中**最关键的概念**。它表示"所有编号 ≤ LWM 的事务都已经在从库提交完成"。

**数据结构：GAQ（Global Assigned Queue）**

```
GAQ（全局已分配队列）：
  ┌── 已提交 ──┐  ┌── 执行中/未提交 ──┐
  [seq=1 ✓] [seq=2 ✓] [seq=3 ✓] [seq=4 ✗] [seq=5 ✗] [seq=6 ✗]
                                    ↑
                               LWM = 3（最右的连续已提交点）

  当 seq=4 提交后 → LWM 前进到 4
  当 seq=4,5 同时提交后 → LWM 前进到 5
```

**LWM 计算**（[rpl_mta_submode.cc:302-337](sql/rpl_mta_submode.cc)）：

```cpp
longlong Mts_submode_logical_clock::get_lwm_timestamp(Relay_log_info *rli, bool need_lock) {
  last_lwm_index = rli->gaq->find_lwm(&ptr_g, ...);
  last_lwm_timestamp = ptr_g->sequence_number;
}
```

### 4.4 Worker 分配策略

**同一事务的连续性保证**：同一个事务内的所有 event 必须交给同一个 worker 执行，并且该 worker 先执行完这个事务的所有 event 后才能接受下一个事务。

```cpp
// rpl_mta_submode.cc:856-900
Slave_worker *Mts_submode_logical_clock::get_least_occupied_worker(...) {
  if (rli->last_assigned_worker) {
    // 同一事务内的事件 → 继续使用同一个 worker
    worker = rli->last_assigned_worker;
  } else {
    // 新事务 → 找一个空闲的 worker
    worker = get_free_worker(rli);
    if (worker == nullptr) {
      // 所有 worker 都忙 → 忙等（sched_yield）直到有空闲
    }
  }
}
```

---

## 第五章：完整时序图

以 3 个事务、2 个 worker 为例，完整展示从主库到从库的执行过程：

```
  时间 ──────────────────────────────────────────────────────────────────►

  【Master 侧】
  
  T1: [══════ Lock Interval ════════]  prepare  ─►  flush(seq=1)  ─►  sync  ─►  commit(m_max→1)
        获取锁     ↑ 记录lc=0              写入 binlog: GTID(seq=1, lc=0)
                 store_commit_parent(0)
  
  T2:    [══════ Lock Interval ════════]  prepare  ─►  flush(seq=2)  ─►  sync  ─►  commit(m_max→2)
          获取锁 ↑ 记录lc=0                      写入 binlog: GTID(seq=2, lc=0)
               store_commit_parent(0)             ← lc=0，与 T1 不冲突！
               (T1尚未commit,m_max=0)
  
  T3:                             [══ Lock Interval ══]  prepare  ─►  flush(seq=3)  ─►  sync  ─►  commit
                                    获取锁 ↑ 记录lc=2         写入 binlog: GTID(seq=3, lc=2)
                                         store_commit_parent(2) ← lc=2，依赖T2！
                                         (T1和T2均已commit,m_max=2)

  【Slave 侧（LOGICAL_CLOCK, 2 workers）】
  
  Coordinator 读取 relay log:
  
  ① GTID(seq=1, lc=0)
     → is_new_group=true（第一条）
     → 分配给 Worker-1
  
  ② GTID(seq=2, lc=0)
     → lc=0, LWM 可能还是 0（Worker-1 尚未完成）
     → clock_leq(0, 0) = true → 无需等待！
     → 分配给 Worker-2  ← T1 和 T2 并行执行！
  
  ③ GTID(seq=3, lc=2)
     → lc=2, LWM 当前是多少？
       如果 Worker-2 未完成 → LWM=0 → clock_leq(2, 0)=false → 必须等待 Worker-2
       如果 Worker-2 已完成 → LWM≥2 → clock_leq(2, LWM)=true → 可以立即执行
     → 分配给空闲的 Worker

  结果：T1和T2 并行，T3 等待 T2 完成后再执行
```

---

## 第六章：关键参数

| 参数 | 作用 | 默认值 |
|---|---|---|
| `binlog_transaction_dependency_tracking` | 主库依赖追踪模式：COMMIT_ORDER / WRITESET / WRITESET_SESSION | COMMIT_ORDER |
| `transaction_write_set_extraction` | writeset 的 hash 算法：OFF / MURMUR32 / XXHASH64 | XXHASH64 (8.0) |
| `binlog_transaction_dependency_history_size` | writeset history 最大行数 | 25000 |
| `replica_parallel_type` | 从库并行模式：DATABASE / LOGICAL_CLOCK | LOGICAL_CLOCK (8.0) |
| `replica_parallel_workers` | 从库 worker 线程数（0 = 关闭 MTS） | 4 (8.0) |
| `replica_preserve_commit_order` | 从库是否保证提交顺序与主库一致 | ON (8.0) |
| `replica_checkpoint_period` | MTS checkpoint 周期（毫秒） | 300 |

### 兼容性约束

```cpp
// sys_vars.cc:4238-4241
// WRITESET 模式要求 transaction_write_set_extraction != OFF
static bool check_binlog_transaction_dependency_tracking(sys_var *, THD *, set_var *var) {
  if (global_system_variables.transaction_write_set_extraction == HASH_ALGORITHM_OFF &&
      var->save_result.ulonglong_value != DEPENDENCY_TRACKING_COMMIT_ORDER) {
    // 错误：WRITESET/WRITESET_SESSION 需要 transaction_write_set_extraction != OFF
    return true;
  }
  return false;
}
```

---

## 第七章：常见问题

### Q1: COMMIT_ORDER 模式下，为什么"锁间隔重叠"就能保证不冲突？

**核心逻辑**：如果 T2 获取所有锁时 T1 还没有释放锁，说明 T1 和 T2 锁住的是**不同的行**（如果锁的是同一行，T2 会被 T1 阻塞，不可能在这个时间点获取到锁）。既然锁住不同的行，从库上并行执行也绝不会冲突。

**形式化证明**：假设 T1 修改行 r1，T2 修改行 r2，且 T2 在 T1 commit 之前获得了 r2 的锁。如果 r1=r2，则 T2 会被 T1 的锁阻塞，不可能在此刻获取锁。因此 r1≠r2，两个事务修改不同行，可以在从库并行执行而不冲突。

### Q2: 既然冲突也可以让从库"并行加锁等待"，为什么不这样做？

**因为 binlog 不包含锁信息。** 如果两个冲突的事务在从库并行执行：
- Worker-1 执行 T1：`UPDATE t SET a=1 WHERE id=1`
- Worker-2 执行 T2：`UPDATE t SET a=2 WHERE id=1`

它们的执行顺序无法保证。在主库，T1 持有锁 → T2 等待 → T1 提交释放锁 → T2 获取锁执行——顺序确定，最终 `a=2`。在从库如果并行执行且 T2 先于 T1 完成，最终 `a=1`——**数据不一致**。

所以 LOGICAL_CLOCK 的设计保证的是：被标记为可并行的事务，在从库上执行时**完全不需要锁等待**——它们的写集合不相交。

### Q3: COMMIT_ORDER 的"假阳性"到底有多严重？

COMMIT_ORDER 会把**所有锁间隔不重叠的事务**都标记为"可能冲突"。但锁间隔不重叠 ≠ 行级冲突。举例：

```
T1: UPDATE t SET a=1 WHERE id=1       (锁行1)
T2: UPDATE s SET b=2 WHERE id=100     (锁表s的行100，与T1完全不相交)

如果 T1 commit → T2获取锁，lock interval 不重叠 → 被 COMMIT_ORDER 标记为串行
但实际上它们完全不冲突！
```

这就是 WRITESET 的价值：通过行级 hash 直接检测"T1 和 T2 修改了相同的行吗？"，消除这类假阳性。

### Q4: `sequence_number` 和 binlog 中的 `last_committed` 为什么是"相对值"？

`Logical_clock` 内部用 `offset` 处理 binlog 轮转。轮转时：

```cpp
// rpl_trx_tracking.cc:185-190
void Commit_order_trx_dependency_tracker::rotate() {
  m_max_committed_transaction.update_offset(m_transaction_counter.get_timestamp());
  m_transaction_counter.update_offset(m_transaction_counter.get_timestamp());
}
```

轮转后写入 binlog 时减去 offset，使新 binlog 中的值从小的正整数开始。从库读取新 relay log 时，这些相对值独立生效。**跨 binlog 的事务依赖会丢失**（`commit_parent` 会被设为 `SEQ_UNINIT`），这意味着跨 binlog 的事务在从库可以做最大程度的并行——这是合理的，因为 binlog 轮转点本身就是天然的同步点。

### Q5: 一张表有外键，WRITESET 为什么必须降级？

外键会引入**隐式的行级依赖**。`add_pke` 只捕获当前 SQL 语句显式修改的行的 hash，无法感知外键约束引发的级联操作：

```
CREATE TABLE parent (id INT PRIMARY KEY);
CREATE TABLE child (id INT PRIMARY KEY, parent_id INT,
    FOREIGN KEY (parent_id) REFERENCES parent(id) ON DELETE CASCADE);

T1: DELETE FROM parent WHERE id=1;    -- 主库上 InnoDB 级联删除 child 表中 parent_id=1 的行
    writeset = {hash(parent, id=1)}   ← 只包含 parent，不包含 child！
    
T2: INSERT INTO child VALUES(100, 1);  -- 插入一个引用 parent.id=1 的记录
    writeset = {hash(child, id=100)}
    -- 如果 T2 在从库上与 T1 并行：
    --   1) T2 先执行 → INSERT 成功（child 表没有约束检查？不对，有外键约束，parent.id=1 仍存在）
    --   2) T1 后执行 → DELETE parent.id=1 
    --       → 但 MySQL 的 binlog 里 T1 的级联删除不会删除 T2 刚插入的 child.id=100！
    --       → 因为 binlog 中记录的是主库级联删除时的"快照"行
    -- 结果：实际不是这个问题。真正的问题是 T1 主库上可能阻止 T2 插入（外键检查失败），
    -- 但从库上 T2 先于 T1 执行 → 外键检查通过 → 数据不一致
```

**更精确的降级原因**：`get_has_related_foreign_keys()` 检查的是"当前事务中涉及的表是否有外键关系"。外键导致以下 writeset 无法正确表达因果：

1. **级联更新/删除**：子表被级联修改的行不在 `writeset` 中
2. **外键插入/更新约束**：子表插入行的合法性依赖于父表行状态，但 writeset 无法编码这种依赖

因此在 [rpl_trx_tracking.cc](sql/rpl_trx_tracking.cc) 的 `can_use_writesets` 判断中，涉及外键的事务直接退回到 COMMIT_ORDER，利用 Lock Interval 保证的假阴性为零特性来确保正确性。

### Q6: COMMIT_ORDER 是一个保守策略吗？社区有优化吗？

**是的，COMMIT_ORDER 本质上是保守的，但保守的方向不是 BGC 边界，而是"假阳性"。**

#### 保守性来自哪里？

COMMIT_ORDER 面临的根本性信息不对称：它只知道"T2 获取锁时 T1 已经 commit"，但无法区分：

- **真冲突**：T2 在等 T1 的同一行锁 → 确实不能并行
- **假冲突**：T2 修改不同行，只是刚好执行得晚 → 实际上可以并行

```
假阳性场景：
────────────────────────────────────────► 时间

T1:  [═ 锁 id=1 ═]  prepare → flush(seq=1) → sync → commit(释放id=1)

T2:                      [═ 锁 id=2 ═]  prepare → flush(seq=2)
                           ↑
                           │ T2 锁 id=2，T1 锁 id=1 → 完全不冲突！
                           │ 但 T1 已 commit(m_max=1)
                           │ → store_commit_parent(1) → lc=1
                           │ → 被标记为串行 ← 误判！
```

#### BGC 边界不是保守性的来源

有一个常见的误解："BGC-2 flush 时 BGC-1 还没 commit，m_max 没更新 → BGC-2 的事务被过度乐观地标记为并行 → 这是 bug"。**这不是 bug，反而是 Lock Interval 理论正确工作的结果**：

```
BGC-1                          BGC-2
┌──────────────┐          ┌──────────────┐
│ T1 flush(1)  │          │ T3 flush(3)  │ ← T3 此时调用 get_dependency()
│ T2 flush(2)  │          │ T4 flush(4)  │    m_max 可能还是 0
└──────┬───────┘          └──────────────┘   → T3 得到较小的 lc → 更多并行
       │ sync                               
       ▼                                    
  T1 commit (m_max→1)                       这是正确的！
  T2 commit (m_max→2)                       如果 T3 能在 T1/T2 还持锁时就获取锁
       │                                    说明 T3 的锁 ≠ T1/T2 的锁 → 不冲突
       └── m_max 更新 ──────────────────→   
```

COMMIT_ORDER 的正确性保证**不依赖 BGC 边界**，它只依赖一个不变量：**`m_max_committed` 反映了所有已经 commit 的事务**。只要 T2 在 T1 之前拿到锁，T1 的 commit 时间（在哪一个 BGC）根本不重要。

#### 社区优化方案

**1. MySQL 8.0 官方优化——WRITESET（已落地）**

这就是 `binlog_transaction_dependency_tracking = WRITESET`。在 COMMIT_ORDER 基础上，用行级 hash 来消除假阳性：

```
COMMIT_ORDER 判定 lc=1（假冲突，依赖T1）
         ↓
WRITESET 再检查：T1的写集合 ∩ T2的写集合 = ∅ → lc 可以从 1 降为 0 → 恢复并行！
```

核心代码 `std::min(last_parent, commit_parent)` 保证了 WRITESET 只会让依赖变小（更宽松），永远不会变大（更严格）。这就是 COMMIT_ORDER "保守性"的官方解法。

**2. WRITESET_SESSION（更严格的正确性保证）**

在 WRITESET 基础上再加同一 session 事务必须串行的约束。适合对数据一致性要求更高的场景。

**3. MariaDB 的乐观并行复制（不同技术路线）**

MariaDB 走了一条不同的路：先让事务并行执行，提交时检查是否有冲突（通过记录每个事务修改的主键范围），如果冲突则回滚重试。这类似于 CPU 的分支预测——乐观执行，失败回退。MySQL 没有采用这个方案。

**4. 理论上的极限优化（未实现）**

如果 binlog 能携带行级锁信息（哪些行被哪些事务锁住），就可以做到精确判定——但存储开销太大，没有实际产品这样做。WRITESET 用 hash 近似替代了锁信息，是当前工程上的最优解。

**总结对比：**

| 方案 | 保守程度 | 实现复杂度 | 状态 |
|---|---|---|---|
| COMMIT_ORDER | 高（假阳性多） | 低 | MySQL 5.7+ |
| WRITESET | 中（消除大部分假阳性） | 中 | MySQL 8.0+（推荐） |
| WRITESET_SESSION | 低（最精确） | 中 | MySQL 8.0+ |
| MariaDB 乐观并行 | 最低（接近完美） | 高 | 仅 MariaDB |
| 基于行锁的精确判定 | 无 | 极高 | 无产品实现 |

### Q7: Lock Interval 和 sequence_number/last_committed 是一回事吗？为什么两个都需要？

**不是一回事。** Lock Interval 是主库运行时的**物理现象**（事务持有锁的时间窗口），而 `sequence_number` 和 `last_committed` 是对这一现象的**数值化编码**，存储在 binlog 中传给从库。

打个比方：Lock Interval 是"两辆车并排行驶的时间段"，`last_committed` 是"摄像头拍到后车时前车的车牌号"，`sequence_number` 是"当前这辆车的车牌号"。

**为什么不能只用 `last_committed`？** 因为从库还需要 `sequence_number` 做三件事：

1. **推进 LWM**：`find_lwm` 扫描 GAQ 找的是"连续已提交的 `sequence_number` 最大值"。没有 sequence_number，Coordinator 不知道各个事务在 GAQ 中的相对位置，无法判断"连续性"，LWM 永远无法推进。

2. **作为 lc 的引用目标**：`lc=1` 的语义是"等 sequence_number=1 的事务提交"。如果只有 lc 没有 sn，"等 1"等于谁就没有含义了。

3. **检测 GAQ 空洞**：`is_new_group` 的触发条件之一是 `gap_successor = (sequence_number != last_sequence_number + 1)`。如果 sn 从 1 跳到 5，说明中间有缺失，必须强制串行化以保证安全。

### Q8: Lock Interval 为什么说"prepare 到 commit"？这是精确的吗？

这需要区分事务类型。源码注释（[rpl_trx_tracking.h:191](sql/rpl_trx_tracking.h)）明确区分了两种情况：

- **autocommit 事务**（单条 DML）：Lock Interval 从 "just before storage engine **prepare**" 开始，到 "just before storage engine **commit**" 结束。此时 Lock Interval ≈ prepare → commit，**近似正确**。

- **显式事务**（BEGIN...COMMIT，含多条语句）：Lock Interval 从 "**end of the last statement** before COMMIT" 开始。这是因为多条语句在执行过程中逐步获取锁，只有最后一条语句执笔结束，所有锁才算拿齐。此时的 Lock Interval ≠ prepare → commit，而是 "最后一条语句结束 → commit"。

**为什么有这种区分？** 因为 2PL 的 Lock Point 取决于锁何时拿齐。autocommit 只有一条语句，执行即拿锁，prepare 时已拿齐。显式事务可能有多条语句，锁是逐步积累的。

### Q9: LWM 是什么的缩写？

**Low Water Mark**（低水位线）。这是计算机科学中的通用术语，描述"已处理数据与未处理数据之间的分界"。

在 MTS 中的精确定义：**GAQ 中连续已提交事务的最大 `sequence_number`**。水位线以下的事务全部已提交，可以安全回收 GAQ 空间。水位线以上的事务要么在执行中，要么在等待依赖。

### Q10: 为什么 INSERT 一行产生三个 PKE？这难道不是表级并行吗？

**不是表级并行，是行级的冗余覆盖。** 三个 PKE 来自表上的三个唯一索引（1 个主键 + 2 个唯一键），每个唯一索引产生一个独立的行标识。

```cpp
// rpl_write_set_handler.cc:850
for (uint key_number = 0; key_number < table->s->keys; key_number++) {
    if (!((table->key_info[key_number].flags & (HA_NOSAME)) == HA_NOSAME))
        continue;  // 跳过非唯一索引
    // 对每个唯一索引，构造 PKE 字符串并 hash
}
```

**为什么需要三个？** 因为优化器可能走任意一个唯一索引来定位行：

```
Trx1: UPDATE t1 SET k=99 WHERE j=2    → 优化器走索引 j
Trx2: UPDATE t1 SET j=100 WHERE k=3   → 优化器走索引 k
```

这是同一行（`i=1, j=2, k=3`）！如果只 hash 主键，两个事务刚好都回表拿到主键 i=1 → 能检测到。但如果只 hash 索引 j，Trx2 走的是索引 k，可能漏掉。

**`add_pke` 的设计理念是冗余安全**：对行的每一个唯一标识都做 hash，确保不管优化器走哪个索引，至少有一个 hash 能捕获到这一行。多算几个 hash 的成本极低（XXHASH64 很快），但漏一个冲突的代价极高（数据不一致）。

**证明它是行级而非表级**：hash 包含的是**键值**（`1`, `2`, `3`），而非仅仅表名。同一张表的不同行产生完全不同的 hash，不碰撞：

```
INSERT INTO t1 VALUES(1, 2, 3)    → hash(i=1), hash(j=2), hash(k=3)
INSERT INTO t1 VALUES(99,100,101) → hash(i=99), hash(j=100), hash(k=101)
→ 6 个 hash 全不同 → 不冲突 → 可以并行 ✓
```

库名和表名只是命名空间前缀（防止不同表的主键值"1"碰撞），核心判定依据是键值本身。

### Q11: 为什么 `sequence_number` 用 `state - offset`，而不直接用全局 `state`？

**因为每个 binlog 文件是一个独立的名字空间。** 源码注释直接说明了设计意图（[rpl_trx_tracking.cc:153-162](sql/rpl_trx_tracking.cc)）：

> "Prepare sequence_number and commit_parent **relative to the current binlog**. A transaction that commits after the binlog is rotated, can have a commit parent in the previous binlog. In this case, subtracting the offset from the sequence number results in a negative number. The commit parent dependency gets lost in such case. Therefore, we log the value SEQ_UNINIT."

**轮转时发生了什么？**

```cpp
// rpl_trx_tracking.cc:185-190
void Commit_order_trx_dependency_tracker::rotate() {
  m_max_committed_transaction.update_offset(m_transaction_counter.get_timestamp());
  m_transaction_counter.update_offset(m_transaction_counter.get_timestamp());
}
// offset 被设为当前的全局 state 值，比如 10000
```

**对比两种方案：**

```
方案A（不用 offset，直接用全局 state）：
  Binlog-1: seq 1→9999
  Binlog-2: seq 10000→19999  ← 全局值
  Binlog-3: seq 20000→29999

  问题：从库读 Binlog-2 时，lc=9998 的事务在 Binlog-1 中
       → 从库必须记住 Binlog-1 中的事务状态才能判断依赖
       → 实现复杂，且无实际好处

方案B（用 offset）：
  全局 state:  10000 10001 10002 10003 ...
                  ↓ rotate, offset=10000
  Binlog-2 中:     1     2     3  ...   (state - offset)

  第一个事务的 lc ≤ offset → SEQ_UNINIT → 无依赖 → 从库立即并行！
```

**设计哲学**：binlog 轮转是一个天然的**同步栅栏（synchronization barrier）**。轮转后旧 binlog 中所有事务要么已提交要么已被 purge。新 binlog 的事务不需要关心旧 binlog 的依赖——让 lc 变为 `SEQ_UNINIT`，释放最大并行度。如果用全局 state 不减去 offset，就失去了这个栅栏释放并行度的效果。

**轮转后的事务也得到特殊处理**。`rotate()` 还会：

```cpp
void Transaction_dependency_tracker::rotate() {
  m_commit_order.rotate();
  m_writeset.rotate(1);     // 清空 writeset history
  if (current_thd)
    current_thd->get_transaction()->sequence_number = 2;  // 新 binlog 从 2 开始
}
```

新 binlog 的第一个事务 `sequence_number=2`，意味着 `1` 被保留（类似于 "虚拟的已提交事务"），让新 binlog 的第一个真实事务在从库上可以立即找到 `commit_parent=1` 作为依赖锚点，避免因为 `lc=0` 而被迫 `is_new_group=true`。

### Q12: writeset 的三个 hash 是行级粒度的确认

> "因为三个唯一索引所以产生了三个 hash"是正确的，**但 INSERT 只插入了一行而不是三行**。三个 hash 指向的是**同一行的三个不同唯一标识**：主键 `i=1`、唯一键 `j=2`、唯一键 `k=3`。

从代码路径来看（[rpl_write_set_handler.cc:850](sql/rpl_write_set_handler.cc)）：

```cpp
for (uint key_number = 0; key_number < table->s->keys; key_number++) {
    if (!((table->key_info[key_number].flags & (HA_NOSAME)) == HA_NOSAME))
        continue;   // 只处理唯一索引
    // 为每个唯一索引构造独立的 PKE 字符串并 hash
}
```

**行级粒度的直接证明**：

```
Trx1: INSERT INTO t1 VALUES(1, 2, 3)
  writeset = {hash(i=1), hash(j=2), hash(k=3)}

Trx2: INSERT INTO t1 VALUES(99, 100, 101)   ← 同一张表！
  writeset = {hash(i=99), hash(j=100), hash(k=101)}

hash(i=1)  ≠ hash(i=99)   ← 键值不同 → hash 不同
∩ = ∅  → 不冲突 → 两个事务可以在从库并行 ✓
```

如果是表级粒度（hash 只含表名不含键值），同表的所有操作都会碰撞——但实际不会，不同行的 hash 是不同的。这确凿证明了 writeset 是**行级**检测。

---

### Q13: 不同 binlog 的事务一定能并行吗？offset 是优化还是语义要求？

**不是计算优化，是语义正确性要求。**

#### 轮转时的强制同步屏障

`new_file_impl()` 在创建新 binlog 之前，会阻塞等待**所有正在提交中的事务完成**（[binlog.cc:6640-6645](sql/binlog.cc)）：

```cpp
mysql_mutex_lock(&LOCK_xids);
/*
  binlog rotate时要保证 in-flight committing trx = 0;
 */
while (get_prep_xids() > 0) {
  mysql_cond_wait(&m_prep_xids_cond, &LOCK_xids);
}
mysql_mutex_unlock(&LOCK_xids);
```

`get_prep_xids()` 返回的是正处于 **prepare 和 commit 之间**（即已经写了 binlog 但存储引擎还没提交完）的事务数量。轮转时**必须等它们全部提交**。

#### 为什么这个屏障保证了"不同 binlog 的事务无依赖"？

```
Binlog-1                                    Binlog-2
┌─────────────────────────────┐      ┌─────────────────────────┐
│ Trx1 ✓  Trx2 ✓  ... Trx97 ✓ │      │ Trx98（新 binlog 第一条）  │
│                             │      │                         │
│ Trx97.5（in-flight）        │      │ → barrier 等了 Trx97.5    │
│   └→ wait! 被 barrier 阻塞   │      │   所有 Binlog-1 事务已提交 │
│                             │      │   m_max 稳定              │
└─────────────────────────────┘      │                         │
        ↓ barrier 完成              │ → lc 为 SEQ_UNINIT       │
        ↓ rotate(), offset = 97     │   语义正确：真的无依赖！    │
        ↓ m_writeset.clear()        └─────────────────────────┘
```

#### 对比：有 barrier vs 无 barrier

| | 无 barrier | 有 barrier（实际实现） |
|---|---|---|
| Binlog-2 第一条事务的依赖 | 可能依赖 Binlog-1 中尚未提交的事务 | 无依赖（所有旧事务已提交） |
| 从库实现 | 必须跨 relay log 跟踪事务状态 | 自包含，只关心当前 relay log |
| `lc = SEQ_UNINIT` | 是 hack，不安全 | 是正确语义，安全 |

**所以 offset 不是为了"减少计算量"**——没有 barrier 的话，用 offset 让 `lc` 变小反而会破坏正确性（把真的依赖错误地抹掉了）。offset 之所以安全，**正是因为 barrier 先在语义层面保证了跨文件无依赖**。offset 的作用是让 binlog 文件内的逻辑时间戳自包含、可独立解析。

#### rotate() 还做了什么

```cpp
void Transaction_dependency_tracker::rotate() {
  m_commit_order.rotate();   // offset = 当前 state
  m_writeset.rotate(1);      // 清空 writeset history
  if (current_thd)
    current_thd->get_transaction()->sequence_number = 2;
}
```

清空 writeset history 也是同理——旧 binlog 的事务已全部提交，它们的 writeset 不需要再参与冲突判断。

### Q14: Causal Consistency（因果一致性）在并行复制中意味着什么？

#### 分布式系统中的定义

**Causal Consistency**（因果一致性）是介于强一致性（linearizability）和最终一致性（eventual consistency）之间的一致性模型：

> **If operation A happens-before operation B（A causally precedes B）, then all nodes must see A before B. Operations that are not causally related（concurrent）can be seen in any order.**

#### 在 MySQL 并行复制中的映射

**"因果"就是事务之间的 happens-before 关系**：

```
客户端 session-1:                   客户端 session-2:
  UPDATE t SET a=1 WHERE id=1;        等待 Trx1 提交...
  COMMIT;  -- Trx1, seq=1
                                      SELECT a FROM t WHERE id=1;
                                      -- 读到 a=1（Trx1 写入的值）
                                      UPDATE t SET b=a*2 WHERE id=1;
                                      -- Trx2 的写入依赖于 Trx1 的写入
                                      COMMIT;  -- Trx2, seq=2

Trx2 读到了 Trx1 写的数据 → Trx2 causally depends on Trx1
→ 从库上 Trx2 必须在 Trx1 之后执行
```

#### LOGICAL_CLOCK 如何实现因果一致性

三种模式的因果检测精度不同：

| 模式 | 因果依赖检测方式 | 精度 |
|---|---|---|
| COMMIT_ORDER | Lock Interval 不重叠 → T2 不可能读到 T1 的写入（T1 还未 commit）→ 无因果依赖 | 保守（不是因果也标为因果 → 假阳性） |
| WRITESET | 写集合不相交 → T2 没有修改 T1 修改过的行 → 即使 Lock Interval 不重叠，也证明不存在直接因果依赖 | 较精确 |
| WRITESET_SESSION | WRITESET + 同 session 内显式串行（同一 session 内的事务天然存在因果链） | 最精确 |

**COMMIT_ORDER 的因果逻辑**：

```
T2 在 T1 的 Lock Interval 内获取完锁
  → T1 还没 commit，T1 的写入对 T2 不可见（隔离性保证）
  → T2 不可能读到 T1 的数据
  → 不存在 T1 → T2 的因果链
  → lc=0 → 从库上 T1 和 T2 可以任意顺序 → 并行 ✓

T2 在 T1 commit 之后获取完锁
  → T1 已 commit，T1 的写入对 T2 可见
  → T2 可能读到了 T1 的数据（也可能没读，但 COMMIT_ORDER 无法区分）
  → 不能排除因果链
  → lc ≥ seq(T1) → 从库上 T2 必须在 T1 之后 → 串行
```

#### 与 Group Replication 的 Causal Consistency 的区别

如果在外部讨论中看到 "Causal Consistency" 指的是 MySQL Group Replication 的 `group_replication_consistency=BEFORE_ON_PRIMARY_FAILOVER`，那是另一个层面——它保证主库故障切换时，新主库不丢失旧主库上已提交但尚未复制到所有节点的"因果链末端"事务。

在 MTS LOGICAL_CLOCK 的语境中，因果一致性就是指：**从库对事务的执行顺序必须尊重主库上的 happens-before 关系**。`last_committed` 是因果依赖的保守上界——"我最晚需要等到谁，因为可能与它存在因果依赖。"

---

## 第八章：WL#7165 经典图解详细分析

WL#7165 是 MySQL 5.7 引入 LOGICAL_CLOCK 并行复制的工作日志（WorkLog）。

### 8.1 图解

```
                              P = flush 阶段分配 sequence_number（step()）
                              C = commit 阶段更新 m_max_committed_transaction
                              竖虚线 = 每个事务记录 last_committed 时 m_max 的快照

Trx1 ------------P(seq=1)-----C(m_max→1)----------------------->
                            |
Trx2 ----------------P(seq=2)-+---C(m_max→2)-------------------->
                            |   |
Trx3 -------------------P(seq=3)-+-----C(m_max→3)-------------->
                            |   |     |
Trx4 -----------------------+-P(seq=4)-+----C(m_max→4)--------->
                            |   |     |    |
Trx5 -----------------------+---+-P(seq=5)+----+---C(m_max→5)-->
                            |   |     |    |   |
Trx6 -----------------------+---+---P(seq=6)+--+---+--C(m_max→6)>
                            |   |     |    |   |   |
Trx7 -----------------------+---+-----+----+---+-P(seq=7)+-C(m_max→7)>
                            |   |     |    |   |   |  |
```

### 8.2 逐个事务分析

| 事务 | P 时 m_max 快照 | C 后 m_max | lc | 依赖谁 | 原因 |
|---|---|---|---|---|---|
| Trx1 | 0 | 1 | **0** | 无 | 第一条事务 |
| Trx2 | 0（Trx1 未 C） | 2 | **0** | 无 | P(2)时 Trx1 还没 C，锁间隔重叠 → 不冲突 |
| Trx3 | 1（Trx1 已 C，Trx2 未 C） | 3 | **1** | Trx1 | P(3)时只看到 m_max=1 |
| Trx4 | 2（Trx1、Trx2 已 C） | 4 | **2** | Trx2 | P(4)时 Trx1/Trx2 都已 C |
| Trx5 | 2（Trx3 未 C） | 5 | **2** | Trx2 | P(5)时只看到 m_max=2（Trx3 未 C，重叠！） |
| Trx6 | 3（Trx3 已 C） | 6 | **3** | Trx3 | P(6)时 Trx3 已 C |
| Trx7 | 4（Trx4 已 C） | 7 | **4** | Trx4 | P(7)时 Trx4 已 C |

### 8.3 关键观察

**1. Trx1 和 Trx2 可以并行**：lc[2]=0，lc[1]=0，都不依赖对方。

**2. Trx3 必须等 Trx1**：lc[3]=1，要等 seq=1 提交。但这可能是假阳性——Trx3 和 Trx1 可能修改的是不同行。

**3. Trx5 的 lc=2 而非 3**：这是 LOGICAL_CLOCK 设计的精妙之处。虽然 Trx3 的 seq=3 > Trx5 的 lc=2，但 Trx5 获取锁时 Trx3 还没 commit（m_max 还是 2），说明它们的锁间隔重叠 → 不冲突。Trx5 不需要等 Trx3。

**4. 波浪形推进**：`last_committed` 的值不是单调递增的，而是像波浪一样随着锁间隔的交错而上下波动——这正是 "LOGICAL_CLOCK"（逻辑时钟）名称的由来。

### 8.4 从库执行顺序

```
Worker-1: Trx1(lc=0) → Trx3(lc=1,等Trx1) → Trx6(lc=3,等Trx3)
Worker-2: Trx2(lc=0) → Trx4(lc=2,等Trx2) → Trx5(lc=2,等Trx2) → Trx7(lc=4,等Trx4)
```

Trx1和Trx2 并行，Trx4和Trx5 都对 lc=2 满足条件后也可并行（在 Trx2 提交后同时分发）。

---

## 第九章：Writeset Hash 生成机制

### 9.1 核心函数：add_pke

Writeset 的核心是 **PKE（Primary Key Equivalent）**——主键等价值。定义在 [rpl_write_set_handler.cc:787](sql/rpl_write_set_handler.cc)。

**步骤1：构造 PKE 字符串**

```cpp
// 格式：{index_name}½{db}½{db_len}{table}½{table_len}{key_val1}½{len1}{key_val2}½{len2}...
```

源码注释给出了完整示例：

```cpp
// CREATE TABLE db1.t1 (i INT PRIMARY KEY, j INT UNIQUE KEY, k INT UNIQUE KEY);
// INSERT INTO db1.t1 VALUES(1, 2, 3);
//
// 产生三个 PKE（一个主键 + 两个唯一键）：
// i → "PRIMARY½db1½3t1½21½1"
// j → "j½db1½3t1½22½1"
// k → "k½db1½3t1½32½1"
```

**PKE 的组成部分**：
- `PRIMARY`：索引名（主键叫 PRIMARY）
- `db1`：database 名
- `3`：db 名长度
- `t1`：table 名
- `2`：table 名长度
- `1`：键值
- `1`：键值长度

**步骤2：Hash 计算**（[rpl_write_set_handler.cc:77](sql/rpl_write_set_handler.cc)）

```cpp
template <class type>
uint64 calc_hash(ulong algorithm, type T, size_t len) {
  if (algorithm == HASH_ALGORITHM_MURMUR32)
    return (murmur3_32((const uchar *)T, len, 0));
  else
    return (MY_XXH64((const uchar *)T, len, 0));
}
```

Hash 算法由参数 `transaction_write_set_extraction` 控制，默认 `XXHASH64`。

**步骤3：存入 writeset vector**（[rpl_transaction_write_set_ctx.cc:64](sql/rpl_transaction_write_set_ctx.cc)）

```cpp
bool Rpl_transaction_write_set_ctx::add_write_set(uint64 hash) {
  if (write_set.size() >= binlog_trx_dependency_history_size) {
    m_local_has_reached_write_set_limit = true;  // 超限 → 后续回退 COMMIT_ORDER
    clear_write_set();
    return false;
  }
  write_set.push_back(hash);
}
```

### 9.2 为什么 PKE 能找到更多可并行事务？

```
COMMIT_ORDER 判定：Trx3 的 lc=1，依赖 Trx1

WRITESET 再检查：
  Trx1 的 writeset = {hash("PRIMARY½db1½3t1½21½1"), hash("j½db1½3t1½22½1")}
  Trx3 的 writeset = {hash("PRIMARY½db2½3s2½299½2"), hash("k½db2½3s2½299½2")}
  
  → writeset 交集 = ∅ → last_parent = 0
  → commit_parent = min(0, 1) = 0 → Trx3 不再依赖 Trx1 → 恢复并行！ ✓
```

**关键设计**：`commit_parent = std::min(last_parent, commit_parent)`。Writeset 只会让依赖变小（发现更多并行机会），永远不会变大（不会引入假阴性）。

### 9.3 唯一键也被纳入的原因

PKE 包含**所有**唯一索引（不仅是主键）。这是因为唯一键也能唯一标识一行——如果两个事务修改同一行的不同唯一索引，hash 对也会产生碰撞，从而被正确识别为冲突。

---

## 第十章：GAQ 与 Checkpoint 机制

### 10.1 GAQ 数据结构

GAQ（Global Assigned Queue）是从库侧跟踪所有已分发事务的环形队列。定义在 [rpl_rli_pdb.cc](sql/rpl_rli_pdb.cc)。

```
GAQ（环形缓冲区，大小 = checkpoint_group × workers）：
    entry                                       avail-1
      ↓                                            ↓
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1✓│ 2✓│ 3✓│ 4✗│ 5✗│ 6 │   │   │   │   │   │   │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
            ↑                        ↑
           LWM                       entry + size
    （连续已提交的最右点）

✓ = done=1（已提交）   ✗ = done=0（执行中/未提交）
```

每个 `Slave_job_group` 包含：
- `sequence_number` / `last_committed`
- `done` 标志（worker 提交后设为 1）
- `group_master_log_pos` / `group_relay_log_pos`（用于 checkpoint 恢复）
- `worker_id`（分配给哪个 worker）

### 10.2 find_lwm：查找低水位线

```cpp
// rpl_rli_pdb.cc:1474
size_t Slave_committed_queue::find_lwm(Slave_job_group **arg_g, size_t start_index) {
  size_t i;
  for (i = start_index; i < avail; i++) {
    ptr_g = &m_Q[i % capacity];
    if (ptr_g->done.load() == 0) break;  // ★ 遇到第一个未完成的 → 停止
  }
  // LWM = 最后一个连续已完成的 group
  ptr_g = &m_Q[(i - 1) % capacity];
  *arg_g = ptr_g;
  return (i - 1) % capacity;
}
```

**关键**：LWM 是"连续已提交"的最右点，不是"已提交总数"。如果 seq=3 未提交而 seq=4 已提交，LWM 仍然是 2，因为 4 之前有空洞（3）。

### 10.3 move_queue_head：Checkpoint 核心

```cpp
// rpl_rli_pdb.cc:1375
size_t Slave_committed_queue::move_queue_head(Slave_worker_array *ws) {
  size_t cnt = 0;
  while (!empty()) {
    ptr_g = &m_Q[entry];
    // ★ 发现 gap（未分配或未完成）→ 停止
    if (ptr_g->worker_id == MTS_WORKER_UNDEF || ptr_g->done.load() == 0)
      break;

    de_queue(&g);     // 弹出
    lwm = g;          // 更新 LWM 为该已完成的 group 的信息
    cnt++;
  }
  return cnt;
}
```

### 10.4 mta_checkpoint_routine：定时 Checkpoint

（[rpl_replica.cc:6459](sql/rpl_replica.cc)）

```
定时触发（replica_checkpoint_period，默认 300ms）
  → move_queue_head()     // 从 GAQ 头部弹掉已完成 group，更新 LWM
  → 更新 group_master_log_pos / group_relay_log_pos 为 LWM 的值
  → flush_info()          // 持久化到 mysql.slave_relay_log_info 表
  → 广播 data_cond        // 唤醒等待 checkpoint 完成的线程
```

### 10.5 Checkpoint 对并行度的影响

1. **释放 GAQ 空间**：`move_queue_head` 弹出已完成的 group，腾出位置给新事务。如果 GAQ 满了，Coordinator 必须等待 checkpoint 推进。
2. **推进 LWM**：LWM 推进后，之前因 `last_committed > old_lwm` 而阻塞在 `wait_for_last_committed_trx()` 的事务可以继续调度。
3. **崩溃恢复**：从库重启时，从 `mysql.slave_relay_log_info` 表读取 LWM 的 `group_relay_log_pos`，跳过已经提交的事务，从 LWM 之后开始重放。

---

## 第十一章：理论基础 —— 2PL 与锁释放时机

### 11.1 InnoDB 的 Two-Phase Locking

InnoDB 使用严格的 **两阶段锁（2PL）** 协议：

- **Growing Phase（增长阶段）**：事务执行过程中逐步获取行锁/间隙锁/表锁
- **Lock Point**：所有锁获取完毕的时刻。对于 autocommit 事务，就在存储引擎 prepare 之前；对于显式事务，在最后一条语句执笔结束之时
- **Shrinking Phase（收缩阶段）**：所有锁在 commit 时统一释放

COMMIT_ORDER 依赖追踪正是利用了这个特性：**Lock Point 之后不再获取新锁，因此 `store_commit_parent` 在 Lock Point 调用是安全的——此时 m_max 的值准确反映了"所有已提交事务所修改的行集合"。**

### 11.2 锁释放的调用链

```
sql/binlog.cc: ordered_commit()
  → ha_commit_low()
    → innobase_commit()                    // ha_innodb.cc
      → trx_commit_for_mysql()
        → trx_commit()
          → trx_commit_low(trx, mtr)       // trx0trx.cc
            → trx_write_serialisation_history(trx, mtr)   // 写 undo log
            → mtr_commit(mtr)                              // 提交 mtr，产生 commit lsn
            → trx_commit_in_memory(trx, mtr, serialised)   // ★ 这里释放锁
              → trx_release_impl_and_expl_locks(trx, serialised)
                → lock_trx_release_locks(trx)             // ★ 最终释放所有锁
```

### 11.3 lock_trx_release_locks 源码

（[lock0lock.cc:6162](storage/innobase/lock/lock0lock.cc)）

```cpp
void lock_trx_release_locks(trx_t *trx) {
  // 等待无人引用此事务（防止并发访问）
  while (trx_is_referenced(trx)) { /* spin wait */ }

  // ★ 核心：锁释放。循环重试直到成功
  while (!locksys::try_release_all_locks(trx)) {
    std::this_thread::yield();
  }

  // 性能优化：不逐个释放，批量清空锁堆
  trx->lock.n_rec_locks.store(0);
  mem_heap_empty(trx->lock.lock_heap);
}
```

### 11.4 trx_commit_in_memory 中锁释放与状态切换

（[trx0trx.cc](storage/innobase/trx/trx0trx.cc)）

```cpp
static void trx_commit_in_memory(trx_t *trx, const mtr_t *mtr, bool serialised) {
  if (trx_is_autocommit_non_locking(trx)) {
    // 只读非锁定事务 → 无锁可释放，直接切换状态
    trx->state.store(TRX_STATE_NOT_STARTED);
  } else {
    trx_release_impl_and_expl_locks(trx, serialised);  // ★ 释放所有锁
    // 此时 trx 已经是 TRX_STATE_COMMITTED_IN_MEMORY
  }
  
  // ... 后续清理：undo cleanup, log flush, trx_init 等
}
```

### 11.5 锁释放时机与 BGC 的关系

锁释放发生在 BGC 的 **COMMIT 阶段**（`ha_commit_low` → `innobase_commit`）。这意味着：

- FLUSH 阶段：事务获取 `sequence_number`，锁**仍然持有**
- SYNC 阶段：binlog 刷盘，锁**仍然持有**
- COMMIT 阶段：存储引擎提交 → 释放锁 → 更新 `m_max_committed_transaction`

**因此在 BGC 概念图中，正确的时机是**：

```
[══ Lock Interval ════════════════════════════════]
 ↑ 获取所有锁                                       ↑ commit 释放锁
    (store_commit_parent)                            (lock_trx_release_locks)
         │                                              │
    prepare ──► flush ──► sync ──► commit(释放锁)
                (step)                        (update_max_committed)
```

| 中文 | 英文 | 说明 |
|---|---|---|
| 逻辑时钟 | Logical Clock | 递增的抽象计数器，非 wall clock |
| 序列号 | Sequence Number | 事务在 binlog 中的全局序号 |
| 提交父事务 | Commit Parent / Last Committed | 当前事务必须等待完成的最小编号 |
| 锁间隔 | Lock Interval | 事务持有所有锁的时间窗口 |
| 假阳性 | False Positive | COMMIT_ORDER 将不冲突的事务标记为冲突 |
| 假阴性 | False Negative | 将实际冲突的事务标记为不冲突（COMMIT_ORDER 保证为零） |
| 主键等价值 | PKE (Primary Key Equivalent) | 用 {库名,表名,索引名,键值} 拼接的字符串，hash 后作为 writeset 条目 |
| 两阶段锁 | 2PL (Two-Phase Locking) | 事务先获取所有锁（Growing Phase），commit 时统一释放（Shrinking Phase） |
| 锁点 | Lock Point | 事务获取完所有锁的时刻，store_commit_parent 在此刻被调用 |
| 写集合 | Write Set | 事务修改的行的 hash 集合 |
| 全局已分配队列 | GAQ (Global Assigned Queue) | 从库侧跟踪所有已分发事务的队列 |
| 低水位线 | LWM (Low Water Mark) | GAQ 中连续已提交的最大 seq_no |
| 检查点 | Checkpoint | 定时清理 GAQ 中已完成的 group，更新 LWM 并持久化 |
| 组提交 | Group Commit / Ordered Commit | 多个事务在一个 fsync 中批量提交 |
| 多线程从库 | MTS (Multi-Threaded Slave) | 从库使用多个 worker 线程并行回放 |
| WL#7165 | WorkLog #7165 | MySQL 5.7 引入 LOGICAL_CLOCK 并行复制的工作日志 |
| 因果一致性 | Causal Consistency | 有 happens-before 关系的事务必须保持顺序，无因果关系的可以乱序 |
| 线性一致性 | Linearizability | 最强一致性，所有操作按真实时间全局排序，像只有一个副本 |
| 最终一致性 | Eventual Consistency | 如果停止写入，最终所有副本一致，中间状态可能不一致 |

---

## 第十二章：一致性模型详解

### 12.1 从故事的直觉出发

设想你有一个银行账户，余额 100 元：

```
手机：转账 50 元给小明   (T1, 发生在服务器A)
电脑：查看余额            (T2, 发生在服务器B)

物理世界中，T1 在 T2 之前 → 余额应该是 50 元
但分布式系统里，服务器B可能还没收到 T1 的结果 → 显示 100 元
```

这就是"不一致"。"一致性模型"定义了：**系统在多大程度上允许多个操作看起来是"同时"的，以及在什么条件下必须看到最新结果**。

用四件事贯穿整个分析：

```
T1: 我转 50 元给小明       (写操作, 发生在服务器A)
T2: 小明查余额             (读操作, 发生在服务器B) 
T3: 我发朋友圈 "转了50元"   (写操作, 发生在服务器A)
T4: 小王刷朋友圈           (读操作, 发生在服务器C)
```

---

### 12.2 强一致性（Linearizability / Strict Consistency）

**定义**：系统表现得像**只有一个副本**。所有操作按真实时间的先后顺序，对所有观察者一致可见。你永远看到最新的值。

#### 银行柜台的类比

你去银行柜台取钱：柜员从金库拿出 100 元给你，金库余额**瞬间**更新为 0。下一个人查余额，一定看到 0。不可能出现"我刚取了钱但柜员说余额没变"这种魔幻场景。

#### 在分布式系统中的要求

```
服务器A: T1(转账50) → T3(发朋友圈)
服务器B: T1(转账50) → T2(查余额=150) → T3(发朋友圈) 
服务器C: T1(转账50) → T3(发朋友圈) → T4(刷到)

约束：
- T2 发生在 T1 真实时间之后 → 小明必须看到 150 元
- T4 发生在 T3 真实时间之后 → 小王必须刷到朋友圈
- 全局时钟：所有操作对所有节点，顺序完全一致
```

#### 实现代价

需要**分布式共识协议**（Paxos / Raft）。每次写操作必须等多数节点确认后才能返回成功：

```
主库写 T1 
  → 发到从库1 ✓
  → 发到从库2 ✓
  → 发到从库3 ✓   ← 等到多数确认
  → 返回客户端 "成功"

延迟 = 网络 RTT × 节点数  → 高延迟、低吞吐
```

#### 在 MySQL 中的对应

Group Replication 的 `group_replication_consistency=AFTER` **近似**实现了线性一致性（实际上叫"after consistency"，是因果一致性+读自己的写）。真正的强一致性在 MySQL 中没有开箱即用的方案——这是 Paxos/Raft 类系统（etcd、TiKV）的领域。

---

### 12.3 最终一致性（Eventual Consistency）

**定义**：如果停止写入，经过足够长的时间，所有副本最终会变得一样。**但在稳定之前，不同副本可以返回不同结果。**

#### 微信群聊的类比

你发了一条消息，你的手机立刻显示。但小王在山里信号不好，他的手机要过 10 秒才收到。不同人的"最新聊天记录"可能不同，但最终大家都会看到这条消息。

#### 在分布式系统中的表现

```
T1 刚执行完的瞬间：

服务器A: 我转了50，余额=50        ← 知道 T1
服务器B: 余额=100                  ← 还没收到 T1 的 binlog
服务器C: 余额=100                  ← 还没收到 T1 的 binlog

T2（小明查余额，请求打到服务器B）→ 看到 100（旧值！）
T4（小王刷朋友圈，请求打到服务器C）→ 看不到我的朋友圈

...复制完成...
所有服务器: 余额=50，朋友圈有我的帖子 ← 最终一致
```

**在最终一致性中，T2 看到旧值是**可以接受的**。只要最终会一致就行。**

#### 在 MySQL 中的对应

**传统的异步复制**就是最终一致性：

```
主库: 写入 binlog → 立刻返回客户端 "成功"
从库: IO 线程异步拉取 → SQL 线程单线程回放

延迟取决于：
- 网络带宽
- 从库负载
- binlog 量
→ 可能落后几毫秒到几分钟
```

---

### 12.4 因果一致性（Causal Consistency）

**定义**：如果 A 和 B 存在因果关系（A happens-before B），那么所有节点必须按 A → B 的顺序看到。如果 A 和 B 没有因果关系（并发），不同节点可以以不同顺序看到。

**这是"性价比"最高的一致性——不需要全局时钟，只需要追踪因果链。**

#### Git 版本控制的类比

```
你: commit A → push
同事: pull → 看到 A → commit B（基于 A）→ push
另一个人: 同时 commit C（改的是另一个文件）

约束：
- B 基于 A，所以 A 必须在 B 之前被看到 ✓（因果）
- C 和 A 没有依赖，可以任意顺序 ✓（并发）
- C 和 B 也没有依赖，可以任意顺序 ✓（并发）
```

#### 因果关系的精确定义：happens-before

**条件1：程序顺序** — 同一连接内，先执行的发生在后执行的之前

```
同一连接:
  UPDATE t SET a=1   (T1)
  UPDATE t SET a=2   (T2)
→ T1 happens-before T2
```

**条件2：读写依赖** — B 读到了 A 写入的值

```
T1: UPDATE t SET a=1 WHERE id=1         (用户A)
T2: SELECT a FROM t WHERE id=1          (用户B) 
     → 读到 a=1
     → UPDATE t SET b=a WHERE id=1       (b 依赖于 a 的值)
→ T1 happens-before T2
→ 因为 T2 的结果依赖于 T1 的结果
```

**条件3：传递性** — if A → B and B → C, then A → C

```
T1 → T2（T2 读了 T1 写的值）
T2 → T3（T3 读了 T2 写的值）
→ T1 → T3（自动成立）
```

#### 并发操作：没有 happens-before 关系的操作

```
用户A: UPDATE t SET a=1 WHERE id=1    (T1)
用户B: UPDATE t SET a=2 WHERE id=2    (T2)

T1 和 T2 修改不同行，互不知晓
→ 没有 happens-before 关系
→ 并发操作

在因果一致性下，不同节点可以以不同顺序看到它们：
节点1: T1 → T2  ✓
节点2: T2 → T1  ✓
两者都合法！
```

#### 在 MySQL 并行复制中的对应

**LOGICAL_CLOCK 并行复制的本质，就是在实现因果一致性**：

```
主库 binlog 中有 5 个事务: T1, T2, T3, T4, T5

因果分析:
  T1 ↔ T2  无因果依赖（并发，修改不同行）
  T1 → T3  有因果依赖（T3 读了 T1 写的值）
  T2 → T4  有因果依赖
  T3 → T5  有因果依赖

合法的从库回放方案:
  Worker-1: T1 → T3 → T5    ← 因果链保持 ✓
  Worker-2: T2 → T4          ← 因果链保持 ✓
  
  T1 和 T2 同时执行 ← 无因果依赖，并发
  T3 和 T4 同时执行 ← 虽然同时执行，但各自满足因果顺序
```

---

### 12.5 三种一致性在行为上的对照

用具体数据和 MySQL 场景对照：

```
场景：主库依次执行 T1(写 x=1), T2(读 x, 然后写 y=x*2), T3(写 z=3)

因果关系：T1 → T2（T2 读了 T1 的写入）
          T1 ↔ T3（并发，互不依赖）
          T2 ↔ T3（并发，互不依赖）

┌──────────┬──────────────────┬──────────────────┬──────────────────────┐
│          │ 强一致性          │ 最终一致性         │ 因果一致性            │
├──────────┼──────────────────┼──────────────────┼──────────────────────┤
│ 约束     │ T1→T2→T3 全局顺序  │ 无约束              │ T1 在 T2 之前         │
│          │ 或 T1→T3→T2       │ 任何顺序都可以      │ T3 可以在任何位置     │
│          │ （但所有节点一致）  │                    │                      │
├──────────┼──────────────────┼──────────────────┼──────────────────────┤
│ 从库合法 │ 和主库完全一样     │ T3→T2→T1 ✓         │ T1→T3→T2 ✓          │
│ 回放顺序 │                    │ T2→T1→T3 ✓         │ T3→T1→T2 ✓          │
│          │                    │ T1→T3→T2 ✓         │ T1→T2→T3 ✓          │
│          │                    │ 等等...             │                      │
├──────────┼──────────────────┼──────────────────┼──────────────────────┤
│ 从库不   │ T3→T1→T2 ✗       │ 没有不合法的情况     │ T2→T1→T3 ✗          │
│ 合法顺序 │ (和主库不一致)     │ （最终都一致）       │ (T2 在 T1 之前       │
│          │                    │                     │  违反因果)           │
├──────────┼──────────────────┼──────────────────┼──────────────────────┤
│ 并行度   │ 0（必须串行）     │ N（全部可并行）     │ 取决于因果图          │
├──────────┼──────────────────┼──────────────────┼──────────────────────┤
│ 延迟     │ 高               │ 低                  │ 中                    │
├──────────┼──────────────────┼──────────────────┼──────────────────────┤
│ MySQL    │ Group Replication │ 传统异步复制         │ 并行复制              │
│ 对应     │ (AFTER模式)       │                     │ LOGICAL_CLOCK        │
└──────────┴──────────────────┴──────────────────┴──────────────────────┘
```

---

### 12.6 为什么并行复制的目标是因果一致性，而非最终一致性？

```
最终一致性的从库回放:
  Worker-1: T3, T1, T2  (T2 看到了 x 还没被 T1 更新的值 → b 算错了！)
  Worker-2: 任意顺序

  问题：虽然最终数据是对的（所有 binlog 都回放了），但中间态的查询会看到不一致。

因果一致性的从库回放:
  Worker-1: T1 → T2       (T2 在 T1 之后，读写依赖保持)
  Worker-2: T3            (T3 和谁都没关系，自由位置)

  保证：任何时候查从库，看到的状态不会违反因果。
```

**核心区别**：最终一致性只保证"最终正确"，中间状态可以任意乱。因果一致性保证"在因果正确的前提下尽可能并行"——这是性能和安全的最佳折中。

---

### 12.7 LOGICAL_CLOCK 三种模式与因果一致性的检测精度

| 模式 | 能检测到的因果链 | 不能检测到的因果链 | 并行度 |
|---|---|---|---|
| COMMIT_ORDER | T2 在 T1 commit后获取锁 → 标记为因果（保守，假阳性） | ❌ 假阳性：Lock Interval 不重叠但实际无因果的也被标记 | 中 |
| WRITESET | 写集合相交 → 真因果；写集合不相交 → 证明无因果 | 同 session 因果、读写非修改行的因果（如 T1 写行1，T2 读行1 写行2） | 高 |
| WRITESET_SESSION | WRITESET + 同 session 程序顺序因果 | 最基本的也全覆盖了 | 最高精度 |

**COMMIT_ORDER 的核心局限**：它只能检测"不可能有读写因果"（Lock Interval 重叠），但不能检测"确实没有因果"（不同行的并发操作）。所以它过度保守——WRITESET 弥补了这个缺陷。

### 12.8 COMMIT_ORDER 能否保证因果一致性？

**能。COMMIT_ORDER 的 Lock Interval 理论保证了假阴性为零**——即不会把有因果依赖的两个事务错误标记为"可以并行"。

逻辑链条：

```
T2 的 Lock Interval 与 T1 重叠
  → T2 获取所有锁时 T1 还没 commit
  → T1 的写入对 T2 不可见（InnoDB 隔离性保证）
  → T2 不可能读到 T1 的数据
  → 不存在 T1 → T2 的"读写因果链"
  → lc=0 ✓（从库并行，正确）

T2 的 Lock Interval 与 T1 不重叠
  → T2 获取所有锁时 T1 已经 commit（锁已释放）
  → T1 的写入对 T2 可见
  → T2 *可能*读到了 T1 的数据（也可能没读，但无法证明）
  → 不能排除因果链
  → 保守标记 lc ≥ seq(T1) ✓（从库串行，假阳性但安全）
```

**COMMIT_ORDER 的因果保证源于：只走"能证明安全"的路，绝不走"无法证明安全"的路。**

### 12.9 WRITESET 的假阴性：什么时候 WRITESET 会错误地抹掉因果依赖？

这是 WRITESET 模式的理论风险。问题出在 `std::min()` 的语义——它只能缩小依赖，不能扩大依赖：

```cpp
// rpl_trx_tracking.cc
// COMMIT_ORDER 计算出 commit_parent = 1（依赖 T1）
// WRITESET 再算：
int64 last_parent = m_writeset_history_start;

commit_parent = std::min(last_parent, commit_parent);
//              ↑                      ↑
//         可能是 0（无冲突）          1（COMMIT_ORDER 的正确结果）
//              ↓
//   min(0, 1) = 0 → 依赖被错误抹掉！✗
```

**致命场景：读写因果但写不同表**

```
T1: UPDATE t1 SET a=1 WHERE id=1;           -- 写 t1.id=1
    writeset = {hash(db, t1, PRIMARY, 1)}
    seq=1

T2: BEGIN;
    SELECT a FROM t1 WHERE id=1;            -- 读 T1 刚写的值 → a=1
    UPDATE t2 SET b=a*10 WHERE id=1;        -- 基于 T1 的值计算，写 t2.id=1
    COMMIT;                                  -- seq=2
    writeset = {hash(db, t2, PRIMARY, 1)}   ← 只有 t2，没有 t1！

时间线：
T1: [══ Lock Interval ═══] commit(释放锁, seq=1, m_max→1)

T2:                  SELECT(读 T1 的数据) → UPDATE → [══ Lock Interval ═══] commit(seq=2)
                                                   ↑ Lock Point
                                                   │ T1 已 commit, m_max=1
```

**COMMIT_ORDER**：Lock Interval 不重叠 → `lc=1` → 依赖 T1 ✓ **正确**

**WRITESET**：`hash(t1,id=1)` ≠ `hash(t2,id=1)` → 写集合无交集 → `commit_parent = min(0, 1) = 0` → "T2 不依赖 T1" ✗ **假阴性！**

**为什么这个假阴性很少导致数据损坏？**

因为 **RBR 消解了执行层因果**。binlog 中 T2 是 `UPDATE t2 SET b=10 WHERE id=1`，值 `10` 已在主库算好。从库 Worker 不会重新执行 `SELECT a FROM t1` 再计算，只是把行变更"贴"到 t2 上。所以在存储层，即使 T2 先执行、T1 后执行，数据最终一致。

**但是查询从库的客户端仍可能看到因果不一致**：

```
从库上 T2 先提交，T1 后提交：

时刻 t1: Worker-2 提交 T2 → t2.b=10
         客户端查询: SELECT * FROM t1 → a 还是旧值（T1 未提交）
                     SELECT * FROM t2 → b=10（T2 已提交）
                     → "b=10 存在，但产生它的 a 还是旧值" → 因果不一致！

时刻 t2: Worker-1 提交 T1 → t1.a=1
         现在才一致
```

**这就是 `replica_preserve_commit_order` 存在的根本原因。**

### 12.10 replica_preserve_commit_order：不检测因果，只强制顺序

MySQL 8.0 引入，**默认 ON**。核心类：`Commit_order_manager`（[rpl_replica_commit_order_manager.h](sql/rpl_replica_commit_order_manager.h)）。

#### 关键认知：它不记录因果关系，不检测冲突

很多人会误以为 `replica_preserve_commit_order` 用 MDL 锁来检测事务间的因果依赖——**完全不是这样。** 在你的 SELECT→UPDATE 例子中，T1 修改 `t1`，T2 修改 `t2`，两个不同表，根本不存在 MDL 冲突。`Commit_order_manager` 靠的是一种完全不同的机制。

#### 实际机制：纯位置的 FIFO 提交队列

```
Coordinator 按 binlog 顺序分发事务：
  分发 T1(seq=1) → Worker-1 → register_trx(Worker-1)
  分发 T2(seq=2) → Worker-2 → register_trx(Worker-2)

提 交 队 列:  [Worker-1] → [Worker-2] → ...
             front ↑

Worker-2 执行完 T2，调用 wait_on_graph():

  if (this->m_workers.front() != worker->id) {   // "你是不是排第一？"
                                                  // → 不是，Worker-1 还在前面
      // 创建 Commit_order_lock_graph ticket——这不是真实锁！
      // 它是一个继承了 MDL_wait_for_subgraph 的自定义同步对象
      Commit_order_lock_graph ticket{worker_thd->mdl_context, *this, worker->id};
      worker_thd->mdl_context.will_wait_for(&ticket);
      
      // ★ 阻塞等待，直到 Worker-1 提交完毕并释放自己
      worker_thd->mdl_context.m_wait.timed_wait(...);
  }

Worker-1 执行完 T1 → commit → finish_one():
  m_workers.pop();                    // 自己离开队列
  next_worker = m_workers.front();    // → Worker-2
  // ★ 设置 Worker-2 的等待状态为 GRANTED，唤醒它
  this->m_workers[next_worker].m_mdl_context->m_wait.set_status(MDL_wait::GRANTED);

Worker-2 被唤醒 → 提交 T2
```

#### Commit_order_lock_graph 的本质

它是 `MDL_wait_for_subgraph` 的子类（[rpl_replica_commit_order_manager.h:440](sql/rpl_replica_commit_order_manager.h)），但**它不对应任何真实数据库对象**——不锁表、不锁行、不锁元数据。它只是一个"等号牌"：

```cpp
class Commit_order_lock_graph : public MDL_wait_for_subgraph {
    // 不是 {table, schema, tablespace, ...} 锁
    // 而是 "我在提交队列等你先走" 的同步令牌
    
 private:
    MDL_context &m_ctx;              // 我的 MDL 上下文
    Commit_order_manager &m_mngr;    // 提交管理器
    uint32 m_worker_id{0};           // 我的 Worker ID
};
```

#### 类比：银行取号排队

```
这不是"冲突检测"（你没有和我抢同一个柜台），
而是"取号排队"（你号比我小，你先办完我才能办）。

InnoDB 锁 + MDL  = 柜台冲突检测："我们都在办同一张表？等一下吧"
Commit_order_manager = 取号机："不管你办哪张表，号小的先办"
```

**因果关系完全由"取号顺序 = binlog 顺序"隐含保证**。`Commit_order_manager` 不知道也不需要知道 T2 是否真的依赖 T1——只要号在 T1 之后，提交就必须在 T1 之后。这是一种比因果分析更保守但实现更简单的策略。

#### 状态机

（[rpl_replica_commit_order_manager.h:61-116](sql/rpl_replica_commit_order_manager.h)）

```
REGISTERED → FINISHED_APPLYING → 不是队列头 → REQUESTED_GRANT → WAITED → RELEASE_NEXT → FINISHED
                                              ↑ 等前面 worker 先 commit ↑
                                    是队列头 → 直接提交
```

### 12.11 因果不一致的具体后果

**后果一：查询从库看到违反因果的状态**

```
主库:
  T1(seq=1): DELETE FROM orders WHERE order_id=100    // 删订单
  T2(seq=2): INSERT INTO archive VALUES(100, ...)      // 归档（依赖删除）

从库上 T2 先提交:
  SELECT * FROM orders WHERE order_id=100   → 存在（T1 还没删）
  SELECT * FROM archive WHERE order_id=100  → 存在（T2 已归档）
  → 同一订单同时存在于两个表 → 语义矛盾！
```

**后果二：级联复制放大问题**

如果该从库又是下游的主库，其 binlog 记录**实际提交顺序**而非原始顺序。下游从库继承错误顺序，因果不一致传播到整个拓扑。

**后果三："读己之写"失效**

```
应用逻辑:
  BEGIN; UPDATE t SET status='done' WHERE id=1; COMMIT;  // T1
  -- 应用认为 T1 已提交
  BEGIN; UPDATE t SET next='archived' WHERE status='done'; COMMIT;  // T2

如果读从库时 T1 还未提交 → T2 找不到 status='done' 的行 → 逻辑错误
```

### 12.12 GTID 空洞与因果一致性：两个维度的概念

**GTID 空洞是"事务缺失"，不是"事务顺序错"。**

```
主库 GTID: uuid:1, uuid:2, uuid:3, uuid:4, uuid:5
从库 executed: uuid:1, uuid:3, uuid:5   ← 缺 uuid:2 和 uuid:4

原因：多源复制交错、故障切换、手动跳过
```

在 LOGICAL_CLOCK MTS 中，Coordinator 检测到 `sequence_number` 跳跃时触发 `is_new_group = true`：

```cpp
gap_successor = (sequence_number != last_sequence_number + 1);
if (gap_successor) is_new_group = true;  // 等所有 worker 完成 → 串行
```

这不是因果检测，而是防御性序列化——Coordinator 不知道空洞中的事务去了哪里。

| | GTID 空洞 | 因果不一致 |
|---|---|---|
| 本质 | 事务缺失（有 GTID 没执行） | 事务存在但提交顺序错误 |
| 原因 | 多源复制、故障切换 | WRITESET 假阴性 + 无 preserve_commit_order |
| 从库检测 | `sequence_number != last_seq+1` | 无法在从库检测 |
| 处理 | `is_new_group = true`，等 worker 清空 | `replica_preserve_commit_order` 强制顺序 |

### 12.13 四层保障总结（修正版）

```
                        因果一致性保障

Layer 1: COMMIT_ORDER
  → Lock Interval 理论保证假阴性 = 0
  → 判定依据：T2 在 T1 commit 后拿锁 → 可能存在读写因果
  → 知道因果关系吗？✅（保守上界，假阳性多）

Layer 2: WRITESET（优化 Layer 1，但有理论风险）
  → 消除 COMMIT_ORDER 的假阳性（不写同行 → 不冲突）
  → 但可能引入假阴性（读写因果但写不同表）
  → 知道因果关系吗？⚠️（知道行级写冲突，但漏掉读写因果）

Layer 3: RBR (Row-Based Replication)
  → binlog 存行最终状态，不重放逻辑
  → 消解了"读 → 判断 → 写"的执行层因果链
  → 知道因果关系吗？不需要知道

Layer 4: replica_preserve_commit_order (8.0 默认 ON)
  → FIFO 提交队列 + MDL 基础设施作同步原语
  → 强制提交顺序 = binlog 顺序（regardless of 是否真有因果）
  → ★ 知道因果关系吗？❌ 完全不知道！
  → 实现策略：用位置（谁先排队）替代因果分析（谁依赖谁）
  → 这是更保守但实现更简单的正确性保证
```

### 12.14 replica_preserve_commit_order 的归属：纯从库参数

**常见误解**：以为这个参数是主库设置的，主库保证提交顺序然后从库执行。

**真相**：参数定义中 `NOT_IN_BINLOG` 和 `ON_CHECK(check_slave_stopped)` 明确说明它是纯从库参数。源码定义（[sys_vars.cc:4280-4287](sql/sys_vars.cc)）：

```cpp
static Sys_var_bool Sys_replica_preserve_commit_order(
    "replica_preserve_commit_order",
    "Force replication worker threads to commit in the same order as on the "
    "source. Enabled by default",
    PERSIST_AS_READONLY GLOBAL_VAR(opt_replica_preserve_commit_order),
    CMD_LINE(OPT_ARG, OPT_REPLICA_PRESERVE_COMMIT_ORDER), DEFAULT(true),
    NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(check_slave_stopped),
    ON_UPDATE(nullptr));
```

参数归属一览：

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│ 主库侧参数（写入 binlog，从库无法控制）:                             │
│   binlog_transaction_dependency_tracking                         │
│     = COMMIT_ORDER | WRITESET | WRITESET_SESSION                │
│   → 决定主库如何计算 last_committed / sequence_number             │
│                                                                  │
│ 从库侧参数（读取 binlog 后，在从库本地生效）:                          │
│   replica_parallel_type = LOGICAL_CLOCK | DATABASE               │
│   replica_parallel_workers = N                                   │
│   replica_preserve_commit_order = ON | OFF  ← 纯从库参数          │
│   → 决定从库 worker 线程如何调度、如何提交                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**"as on the source"指的是 binlog 中的事务顺序，不是由主库来强制执行。** 整个 `Commit_order_manager` 的 FIFO 队列、`Commit_order_lock_graph` 等待机制、MDL 同步原语全都在从库进程内运行。主库甚至不知道从库有这个参数。

**MDL 是从库本地 MDL，不是主库 MDL。** `Commit_order_lock_graph` 绑定的是从库 Worker 线程自己的 `MDL_context`。主库和从库是两个独立进程，不可能共享 MDL 图。这里只是**借用 MDL 的 wait/grant 基础设施做线程间同步**。

### 12.15 非 RBR 下 WRITESET 的数据一致性风险

**WRITESET 在非 RBR 下有真实的数据损坏风险。**

#### 回顾 WRITESET 假阴性场景

```
主库:
  T1: UPDATE t1 SET a=1 WHERE id=1          seq=1, writeset={hash(t1,id=1)}
  T2: BEGIN;
      SELECT a FROM t1 WHERE id=1           → 读到 a=1
      UPDATE t2 SET b=a*10 WHERE id=1
      COMMIT;                                seq=2, writeset={hash(t2,id=1)}

WRITESET 判定: hash(t1,id=1) ≠ hash(t2,id=1) → lc=0 → T2 "不依赖" T1（假阴性！）
```

#### RBR 下为什么安全

RBR 的 T2 binlog event 是：

```
### UPDATE `db`.`t2`
### WHERE @1=1
### SET @2=10              ← 值已由主库算好，直接写入
```

从库 Worker-2 执行 T2 时只是把 `10` 写入 t2。不读 t1，不重新计算。即使 T1 还没 commit，T2 写入的值也是正确的。**因果链在 binlog 产生时已固化为最终行状态。**

#### 非 RBR（SBR）下为什么致命

SBR 的 T2 binlog event 是：

```sql
UPDATE t2 SET b=a*10 WHERE id=1   ← 完整的 SQL 语句！
```

从库 Worker-2 执行 T2 时会**重新执行这条 SQL**：

```sql
-- 从库 Worker-2 重新执行：
-- 1. 读取当前 t1.a 的值
-- 2. 读取 t2 的 id=1 行
-- 3. 计算 b = a * 10
-- 4. 写入 t2

如果 T1 还没 commit → t1.a 还是旧值(NULL 或 0)
  → b = 0 * 10 = 0  
  → 错误！应该是 10！
  → 数据损坏 ✗
```

#### 对比：各模式在不同 binlog format 下的安全性

| | RBR | SBR / MIXED(SBR 部分) |
|---|---|---|
| **COMMIT_ORDER** | ✅ 安全（假阴性=0） | ✅ 安全（假阴性=0，lc 为保守上界） |
| **WRITESET** | ✅ 安全（假阴性 + RBR = 行状态正确） | ❌ **数据损坏**（假阴性 + SBR = 读脏数据重算） |
| **WRITESET_SESSION** | ✅ 安全 | ❌ 同上 |

**关键**: COMMIT_ORDER 在 SBR 下也安全，因为 Lock Interval 理论保证了假阴性=0。T2 的 `lc=1` 意味着从库 T2 一定在 T1 之后执行（或被 preserve_commit_order 强制在 T1 后提交），读到的一定是 T1 更新后的值。

**WRITESET 的唯一安全前提是 RBR。** 这也是为什么 `binlog_format=ROW` 是 MySQL 8.0 的默认值，以及官方文档强烈建议并行复制配合 RBR 使用。

#### MySQL 有没有强制要求 WRITESET 必须配 RBR？

**没有硬性强制。** `check_binlog_transaction_dependency_tracking()` 只检查了 `transaction_write_set_extraction` 不是 OFF，**完全没有检查 `binlog_format`**：

```cpp
// sys_vars.cc
static bool check_binlog_transaction_dependency_tracking(...) {
  if (transaction_write_set_extraction == HASH_ALGORITHM_OFF &&
      var->save_result.ulonglong_value != DEPENDENCY_TRACKING_COMMIT_ORDER) {
    // 只检查 writeset 提取是否开启，不检查 binlog format
    my_error(ER_WRONG_USAGE, MYF(0), ...);
    return true;
  }
  return false;
}
```

所以你可以设置 `binlog_format=STATEMENT` + `binlog_transaction_dependency_tracking=WRITESET`，MySQL 不会报错——但数据一致性由你自己负责。这是实践中的一个大坑。

### 12.16 GTID 空洞与 preserve_commit_order：历史与演进

#### 2017 年 Taobao MySQL Monthly 文章的上下文

那篇文章写作于 MySQL 5.7 时代。当时 `replica_preserve_commit_order` **还未成为默认**（MySQL 8.0 才将其设为默认 ON）。在没有这个参数的情况下，LOGICAL_CLOCK MTS 的 Worker 可以以任意顺序提交：

```
Coordinator 按 binlog 顺序分发：
  T1(seq=1, GTID=uuid:1) → Worker-1
  T2(seq=2, GTID=uuid:2) → Worker-2

Worker-1 执行 T1（大事务，10秒）
Worker-2 执行 T2（小事务，0.5秒）

Worker-2 先完成 → 立即提交 T2
  → gtid_executed = {uuid:2}   ← 空洞！T1(uuid:1) 还没提交

Worker-1 完成 → 提交 T1
  → gtid_executed = {uuid:1, uuid:2}  ← 空洞填上
```

**文章所说的"GTID 空洞"是真实的、可观测的**——在任何时刻执行 `SELECT @@gtid_executed`，你可能看到 `uuid:2` 但看不到 `uuid:1`。这是一个**临时性空洞**，最终会被填满，但它的存在时长取决于最慢的 Worker。

#### MySQL 8.0 的解决方案

`replica_preserve_commit_order=ON`（8.0 默认值）彻底解决了这个问题。Worker 执行完毕后不能立即提交，必须等在 FIFO 队列中排到自己才能提交——提交顺序严格等于 Coordinator 分发顺序（即 binlog 顺序）。

**所以在 MySQL 8.0 默认配置下，GTID 空洞不会因为并行复制产生。**

### 12.17 `Commit_order_queue` 内部机制：sequence_nr 的真实作用

很多人（包括之前的我）会误以为 `Commit_order_manager` 使用 binlog 的 `sequence_number` 来排序提交。**这不是真的。** 让我们从源码层面澄清两个不同的"序列号"。

#### 两个序列号，完全不同的用途

| | `Slave_worker::sequence_number()` | `Commit_order_queue::commit_sequence_nr` |
|---|---|---|
| **来源** | GAQ（binlog 的原始 sequence_number） | `m_commit_sequence_generator->fetch_add(1)` |
| **何时赋值** | Coordinator 分发事务时写入 GAQ | Worker `push()` 入队列时 |
| **含义** | 事务在主库 binlog 中的全局序号 | 提交请求在队列中的本地序号 |
| **用途** | 死锁检测方向判断 | CAS 原子协议保证只有前驱能解锁后继 |
| **与提交排序关系** | ❌ 无关 | ❌ 无关（排序靠的是队列位置） |

#### 提交排序靠什么：FIFO 队列的位置

```cpp
// commit_order_queue.cc:163
void Commit_order_queue::push(value_type index) {
  sequence_type next{Node::NO_SEQUENCE_NR};
  do {
    next = this->m_commit_sequence_generator->fetch_add(1);  // 单调递增
  } while (next <= Node::SEQUENCE_NR_FROZEN);
  this->m_workers[index].m_commit_sequence_nr->store(next);
  this->m_commit_queue << index;  // ← 尾插入链表
}
```

**排序是由 `m_commit_queue` 的链表顺序决定的（FIFO），不是由 `commit_sequence_nr` 的值决定的。** `commit_sequence_nr` 的单调递增只是实现上的自然结果，不是排序的依据。

#### commit_sequence_nr 的真实用途：前驱-后继的原子解锁协议

```cpp
// finish_one() 中的核心 CAS 代码
void Commit_order_manager::finish_one(Slave_worker *worker) {
  auto [this_worker, this_seq_nr] = m_workers.pop();
  auto next_seq_nr = get_next_sequence_nr(this_seq_nr);  // seq+1
  auto next_worker = m_workers.front();
  
  // ★ 只有持有 seq=N 的 worker 才能解锁 seq=N+1 的 worker
  if (next_worker != NO_WORKER &&
      m_workers[next_worker].freeze_commit_sequence_nr(next_seq_nr)) {
    // 确认后继的 sequence_nr 等于 N+1 → 我就是它的直接前驱
    m_workers[next_worker].m_mdl_context->m_wait.set_status(MDL_wait::GRANTED);
    m_workers[next_worker].unfreeze_commit_sequence_nr(next_seq_nr);
  }
}
```

这里 `commit_sequence_nr` 充当的是一个"防重入"标记：防止 Worker-N 在 unlock Worker-N+1 的过程中，Worker-N 又被分配了新事务并获得新的 sequence_nr，导致并发冲突。`freeze` / `unfreeze` 和 `reset_commit_sequence_nr`（在 `pop()` 时调用）之间用 CAS 协议协调。

**说人话：`commit_sequence_nr` 是一个"证明你是前驱"的凭证，不是"排第几"的依据。排第几看链表位置。**

#### sequence_number 的唯一用途：死锁检测中的方向判断

```cpp
// rpl_replica_commit_order_manager.cc:345
void Commit_order_manager::check_and_report_deadlock(THD *thd_self, THD *thd_wait_for) {
  Slave_worker *self_w = get_thd_worker(thd_self);
  Slave_worker *wait_for_w = get_thd_worker(thd_wait_for);
  
  if (wait_for_w->sequence_number() > self_w->sequence_number()) {
    // "等等，你的 seq 比我大，按理说你在我后面
    //  但我却在等你 commit...这不是死锁是什么？"
    mngr->report_deadlock(wait_for_w);
  }
}
```

这发生在 InnoDB 行锁等待触发的死锁检测中。Worker-1（seq 小）在等 Worker-2（seq 大）提交，但 Worker-2 又在等 Worker-1 释放 InnoDB 行锁——经典环形等待。`sequence_number()` 帮助判断环形方向。

#### 一步到位：完整流程图

```
Coordinator 按 binlog 顺序分发事务：
  T1(seq=1) → register_trx(Worker-1) → push → 链表: [W1, csn=2]
  T2(seq=2) → register_trx(Worker-2) → push → 链表: [W1, csn=2] → [W2, csn=3]

提交顺序：由链表 FIFO 顺序决定（W1 在 W2 前面）
          commit_sequence_nr 不参与排序

csn 的作用：W1 pop 后得到 csn=2
          → next_seq_nr = 3
          → freeze_commit_sequence_nr(3) on W2 → CAS 确认 W2.csn==3
          → W1 解锁 W2
          → 防止并发问题（W1 解锁 W2 时，W1 被重新分配）

sequence_number 的作用：
  deadlock_check: if (W2.seq > W1.seq && W1 is waiting for W2)
                  → 死锁方向异常 → rollback
```

### 12.18 preserve_commit_order 的真正串行化位置：BGC Stage#0 vs ha_commit_low

**重要纠正**：之前的分析只关注了 `ha_commit_low` 中的 `wait`，遗漏了 BGC Stage#0 的关键作用。实际上根据从库是否写 binlog，等待发生在**不同位置**。

#### 两条路径总览

```
                    log_replica_updates = ON（写binlog）
                    ═══════════════════════════════════
Worker执行events → trans_commit → MYSQL_BIN_LOG::ordered_commit()
                                       │
                          Stage #0: wait() ← ★ 在这里阻塞等待
                              │
                          Stage #1: FLUSH (写binlog，顺序正确)
                              │
                          Stage #2: SYNC
                              │
                          Stage #3: COMMIT → ha_commit_low
                                        wait() ← no-op（已waited）
                                        ht->commit()
                                        wait_and_finish() ← 释放下一个

                    log_replica_updates = OFF（不写binlog）
                    ═══════════════════════════════════
Worker执行events → ha_commit_low() ← 直接调用，没有BGC
                        │
                    wait() ← ★ 在这里阻塞等待
                    ht->commit()
                    wait_and_finish() ← 释放下一个
```

#### Stage#0 为什么存在？（[binlog.cc:8892-8900](sql/binlog.cc)）

```cpp
// ordered_commit() 中的 Stage #0
/*
  Stage #0: ensure slave threads commit order as they appear in the slave's
            relay log for transactions flushing to binary log.
*/
if (Commit_order_manager::wait_for_its_turn_before_flush_stage(thd) ||
    ending_trans(thd, all) ||                           // all=true → 恒为true
    Commit_order_manager::get_rollback_status(thd)) {
  if (Commit_order_manager::wait(thd)) {                // ← DML 也会走到这里
    return thd->commit_error;
  }
}
```

关键判断：`ending_trans(thd, all=true)` → 恒为 `true`（[binlog.cc:3252](sql/binlog.cc)：`return (all || ...)`）。

所以对于从库 MTS worker，**DML 也会在 Stage#0 调用 `wait()`**，不只是 DDL。

**Stage#0 存在的原因**：如果不在这里等待，可能出现以下乱序：

```
假设 T1(seq=1) 和 T2(seq=2) 分别给 Worker-1 和 Worker-2：

  无 Stage#0（错误）:
    W1: 执行T1(慢) ──────────────────→ FLUSH(T1) → SYNC → COMMIT
    W2: 执行T2(快) → FLUSH(T2) → SYNC → COMMIT
         ↑ 先写入 binlog!
    → binlog 中 T2 排在 T1 前面 → 顺序错误！

  有 Stage#0（正确）:
    W1: 执行T1(慢) ──→ wait()通过 → FLUSH(T1) → SYNC → COMMIT
    W2: 执行T2(快) → wait()阻塞 ────────────→ FLUSH(T2) → SYNC → COMMIT
                     ↑ 等 T1 的 wait() 完成才放行
    → binlog 顺序正确
```

#### wait() 的状态机：第一次等，第二次 no-op

（[rpl_replica_commit_order_manager.cc](sql/rpl_replica_commit_order_manager.cc)）

```cpp
bool Commit_order_manager::wait(Slave_worker *worker) {
  // ★ 只有 REGISTERED 状态才真正等待
  if (this->m_workers[worker->id].m_stage ==
      cs::apply::Commit_order_queue::enum_worker_stage::REGISTERED) {
    if (this->wait_on_graph(worker))       // 在 MDL graph 上等待
      return true;                         //   直到前驱释放
    // wait_on_graph 返回后，状态变为 WAITED
    return rollback_status;
  }
  return false;  // ★ 已经是 WAITED 状态 → 直接返回，no-op
}
```

**状态转换**（[rpl_replica_commit_order_manager.h](sql/rpl_replica_commit_order_manager.h)）：

```
REGISTERED → FINISHED_APPLYING → REQUESTED_GRANT → WAITED → FINISHED
                                  ↑ wait() 中               ↑ wait_and_finish()
                                    真正阻塞                  释放下一个
```

所以：
- **第一次 `wait()`**（Stage#0 或 ha_commit_low）：状态 = REGISTERED → 阻塞等待 → 变为 WAITED
- **第二次 `wait()`**（ha_commit_low 内）：状态 ≠ REGISTERED → 直接返回 false，no-op

#### ha_commit_low 中的 wait_and_finish：释放后继

```cpp
int ha_commit_low(THD *thd, bool all, bool run_after_commit) {
  // ...
  if (is_ha_commit_low_invoking_commit_order(thd, all) || ...) {
    if (Commit_order_manager::wait(thd)) {   // no-op（已在Stage#0等待过）
      error = 1;
      Commit_order_manager::wait_and_finish(thd, error);
      goto err;
    }
    is_applier_wait_enabled = true;
  }

  for (auto &ha_info : ha_list) {
    ht->commit(ht, thd, all);   // InnoDB 提交
  }

  if (is_applier_wait_enabled) {
    Commit_order_manager::wait_and_finish(thd, error);
    // ↑ 将自己从队列弹出，finish_one() 中用 MDL GRANTED 唤醒后继
  }
}
```

#### 最终时间线（log_replica_updates=ON，最常见情况）

```
Worker-1 (T1, seq=1):
  执行events(500ms) → ordered_commit() → Stage#0 wait(0μs,是第一个)
  → FLUSH → SYNC → COMMIT(ha_commit_low: wait no-op → ht->commit(1ms) → wait_and_finish 释放W2)

Worker-2 (T2, seq=2):
  执行events(200ms) → ordered_commit() → Stage#0 wait(300ms,被W1阻塞)
  → FLUSH → SYNC → COMMIT(ha_commit_low: wait no-op → ht->commit(1ms) → wait_and_finish)

总耗时 = max(500,200) + 300(等待) + 2×1 = 802ms
单线程 = 702ms

注意：这里 wait(300ms) 是"空闲等待"——W2 执行完了但必须等 W1 也执行完并通过 Stage#0。
如果 W1 和 W2 执行耗时接近，等待时间 ≈ 0。
```

#### undo 在哪写的？执行阶段，跟 commit order 无关

**这个结论不变**（之前的分析是正确的）：undo 在 apply row events 时实时写入，不在 `ht->commit()` 时写入。

```
ROW event apply:
  ① 定位行
  ② trx_undo_report_row_operation() → 写 undo
  ③ btr_cur_update_in_place() → 更新索引
  ④ MVCC: read_view 更新

ht->commit() 做的事（轻量）:
  ① trx_write_serialisation_history() → 给已有 undo 打 trx no
  ② mtr_commit() → 产生 commit_lsn
  ③ lock_trx_release_locks() → 释放锁
```

**结论**：`preserve_commit_order` 串行化的只是提交顺序（binlog 写入顺序 + InnoDB 提交顺序），不串行 DML 执行。undo/索引/MVCC 仍然是多 Worker 并行的。Stage#0 的额外等待仅当快事务提前完成时需要等慢事务——这保证了因果一致性，代价是偶尔的空闲等待。
