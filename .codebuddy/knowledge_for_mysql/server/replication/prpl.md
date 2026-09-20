# MySQL Binlog 并行复制深度解析

> 基于 MySQL 8.0.39 源码，涵盖主库依赖追踪（COMMIT_ORDER / WRITESET / WRITESET_SESSION）、从库 MTS 调度、GAQ 与 LWM、Writeset Hash、以及 MTS 崩溃恢复。
>
> **边界**：本篇讲**并行复制**——主库怎么标记"哪些事务可并行"、从库怎么调度；binlog 本身（格式/组提交）见 [`binlog.md`](binlog.md)，GTID 见 [`gtid.md`](gtid.md)，复制拓扑与故障转移见 [`replication.md`](replication.md)，从库崩溃后的 redo 恢复见 [`../../innodb/recovery.md`](../../innodb/recovery.md)。


---

## 目录

- [概述](#概述)
  - 是什么
  - 用途
  - 版本演进
- [理论基础](#理论基础)
  - 设计思想与权衡
  - 理论溯源：WL#7165
  - 算法与数据结构
  - 2PL 与锁释放时机
  - 一致性模型详解
  - 他库对比与演进动机
- [核心实现](#核心实现)
  - 主链路
  - 核心概念：两个逻辑时间戳
  - 主库侧：依赖追踪
  - 从库侧：MTS 调度
  - 完整时序图
  - WL#7165 经典图解分析
  - Writeset Hash 生成机制
  - GAQ 与 Checkpoint 机制
  - MTS Crash Recovery 与静默丢数据缺陷
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
  - 高级语法技巧 / C++ 特性
  - 经典算法的实现落地
  - 复杂体系与设计模式的代码结构
- [可观测性](#可观测性)
  - 系统变量与状态变量
  - 观测对象 → 手段 速查
- [Misc](#misc)
  - 扩展点：加一种依赖追踪模式要改哪几处
  - 容易误解的命名：两个 "logical clock"
  - 常见问题 Q1~Q14
- [参考](#参考)


---

## 概述

### 是什么



MySQL 的主从复制是构建高可用架构的基石。自 5.7 版本引入 LOGICAL_CLOCK 并行复制后，从库不再是单线程串行回放，而是可以根据主库写入 binlog 的逻辑时间戳（`last_committed` 与 `sequence_number`），在多个 worker 线程上安全地并行执行不冲突的事务。本文从源码级深度出发，完整剖析主库侧三种依赖追踪模式（COMMIT_ORDER / WRITESET / WRITESET_SESSION）的计算原理，以及从库侧 `Mts_submode_logical_clock` 调度器如何利用这些时间戳实现低冲突并行回放。涵盖 Lock Interval 理论、Logical Clock 设计、GAQ（Global Assigned Queue）的 LWM 维护机制、以及 COMMIT_ORDER 过度保守性的根源分析。

**关键问题回答：**
- LOGICAL_CLOCK 并行复制**是** MySQL 5.7 引入的，8.0 中持续存在且为默认模式
- 两个逻辑时间戳 `sequence_number` 在 **flush 阶段**分配，`last_committed` 在**事务获取所有锁的瞬间**记录（通过对 `m_max_committed_transaction` 的快照）
- 并行性的保证来自主库的 **锁间隔（Lock Interval）重叠判定**：如果 T2 获取锁时 T1 尚未释放锁（锁间隔重叠），则两者不存在锁冲突，从库可以安全并行
- 即使判定为冲突，从库也**不能**让它们"并行加锁等待"，因为 binlog 不包含锁信息，乱序执行会破坏数据一致性

---

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 并行复制按**库（schema）**粒度（`--slave-parallel-type=DATABASE`），单库写压力无法并行 |
| 5.7 | 引入 **LOGICAL_CLOCK** 并行复制：主库在 binlog 写入 `last_committed` / `sequence_number` 两个逻辑时间戳，从库据此跨库并行 |
| 8.0 | 增加 **WRITESET** / **WRITESET_SESSION**（行级冲突检测），并行度不再受"是否同库"限制；`slave_parallel_type` 默认 `LOGICAL_CLOCK` |
| 8.0.21+ | 复制术语改名（slave→replica / master→source），变量名同步为 `replica_parallel_workers` 等 |

> 并行复制的**演进动机**（为什么要从"库级"走到"行级"）写在「理论基础 → 设计思想与权衡」。


---

## 理论基础

> 本章回答"为什么这么设计"。并行复制的全部设计压力来自一个矛盾：**从库要并行，但不能破坏主库的执行顺序语义**。

### 设计思想与权衡


#### 核心权衡：并行度 vs 正确性

并行复制唯一的正确性要求是**不产生主库上不存在的结果**。MySQL 的做法是：**主库在 binlog 里预先算好依赖，从库只做调度不做判断**——这样从库不需要理解 SQL 语义，也不持有锁信息。

被放弃的方案与代价（详见 Misc 的对应问答）：

| 方案 | 为什么没选 / 代价 |
|---|---|
| 从库自行加锁等待（冲突也并行执行） | binlog **不含锁信息**，乱序执行会破坏一致性（见 Q2） |
| 从库解析 SQL 自行判断冲突 | 成本过高且无法处理跨语句依赖（见 Q3、Q6） |
| 按库并行（5.6） | 单库热点完全无法并行，这是 5.7 引入 LOGICAL_CLOCK 的直接动因 |

这套设计的**简化假设**及其失效场景：COMMIT_ORDER 用"锁间隔是否重叠"近似"是否有冲突"，是一个**保守近似**——会产生假阳性（本可并行却被判冲突），在长事务、跨 binlog 边界等场景退化明显（见 Q3、Q6、Q13）。WRITESET 把粒度降到行级正是为了缓解这一退化。



### 2PL 与锁释放时机

#### InnoDB 的 Two-Phase Locking

InnoDB 使用严格的 **两阶段锁（2PL）** 协议：

- **Growing Phase（增长阶段）**：事务执行过程中逐步获取行锁/间隙锁/表锁
- **Lock Point**：所有锁获取完毕的时刻。对于 autocommit 事务，就在存储引擎 prepare 之前；对于显式事务，在最后一条语句执笔结束之时
- **Shrinking Phase（收缩阶段）**：所有锁在 commit 时统一释放

COMMIT_ORDER 依赖追踪正是利用了这个特性：**Lock Point 之后不再获取新锁，因此 `store_commit_parent` 在 Lock Point 调用是安全的——此时 m_max 的值准确反映了"所有已提交事务所修改的行集合"。**

#### 锁释放的调用链

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

#### lock_trx_release_locks 源码

（[lock0lock.cc](storage/innobase/lock/lock0lock.cc)）

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

#### trx_commit_in_memory 中锁释放与状态切换

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

#### 锁释放时机与 BGC 的关系

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



### 一致性模型详解

#### 从故事的直觉出发

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

#### 强一致性（Linearizability / Strict Consistency）

**定义**：系统表现得像**只有一个副本**。所有操作按真实时间的先后顺序，对所有观察者一致可见。你永远看到最新的值。

##### 银行柜台的类比

你去银行柜台取钱：柜员从金库拿出 100 元给你，金库余额**瞬间**更新为 0。下一个人查余额，一定看到 0。不可能出现"我刚取了钱但柜员说余额没变"这种魔幻场景。

##### 在分布式系统中的要求

```
服务器A: T1(转账50) → T3(发朋友圈)
服务器B: T1(转账50) → T2(查余额=150) → T3(发朋友圈) 
服务器C: T1(转账50) → T3(发朋友圈) → T4(刷到)

约束：
- T2 发生在 T1 真实时间之后 → 小明必须看到 150 元
- T4 发生在 T3 真实时间之后 → 小王必须刷到朋友圈
- 全局时钟：所有操作对所有节点，顺序完全一致
```

##### 实现代价

需要**分布式共识协议**（Paxos / Raft）。每次写操作必须等多数节点确认后才能返回成功：

```
主库写 T1 
  → 发到从库1 ✓
  → 发到从库2 ✓
  → 发到从库3 ✓   ← 等到多数确认
  → 返回客户端 "成功"

延迟 = 网络 RTT × 节点数  → 高延迟、低吞吐
```

##### 在 MySQL 中的对应

Group Replication 的 `group_replication_consistency=AFTER` **近似**实现了线性一致性（实际上叫"after consistency"，是因果一致性+读自己的写）。真正的强一致性在 MySQL 中没有开箱即用的方案——这是 Paxos/Raft 类系统（etcd、TiKV）的领域。

---

#### 最终一致性（Eventual Consistency）

**定义**：如果停止写入，经过足够长的时间，所有副本最终会变得一样。**但在稳定之前，不同副本可以返回不同结果。**

##### 微信群聊的类比

你发了一条消息，你的手机立刻显示。但小王在山里信号不好，他的手机要过 10 秒才收到。不同人的"最新聊天记录"可能不同，但最终大家都会看到这条消息。

##### 在分布式系统中的表现

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

##### 在 MySQL 中的对应

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

#### 因果一致性（Causal Consistency）

**定义**：如果 A 和 B 存在因果关系（A happens-before B），那么所有节点必须按 A → B 的顺序看到。如果 A 和 B 没有因果关系（并发），不同节点可以以不同顺序看到。

**这是"性价比"最高的一致性——不需要全局时钟，只需要追踪因果链。**

##### Git 版本控制的类比

```
你: commit A → push
同事: pull → 看到 A → commit B（基于 A）→ push
另一个人: 同时 commit C（改的是另一个文件）

约束：
- B 基于 A，所以 A 必须在 B 之前被看到 ✓（因果）
- C 和 A 没有依赖，可以任意顺序 ✓（并发）
- C 和 B 也没有依赖，可以任意顺序 ✓（并发）
```

##### 因果关系的精确定义：happens-before

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

##### 并发操作：没有 happens-before 关系的操作

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

##### 在 MySQL 并行复制中的对应

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

#### 三种一致性在行为上的对照

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

#### 为什么并行复制的目标是因果一致性，而非最终一致性？

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

#### LOGICAL_CLOCK 三种模式与因果一致性的检测精度

| 模式 | 能检测到的因果链 | 不能检测到的因果链 | 并行度 |
|---|---|---|---|
| COMMIT_ORDER | T2 在 T1 commit后获取锁 → 标记为因果（保守，假阳性） | ❌ 假阳性：Lock Interval 不重叠但实际无因果的也被标记 | 中 |
| WRITESET | 写集合相交 → 真因果；写集合不相交 → 证明无因果 | 同 session 因果、读写非修改行的因果（如 T1 写行1，T2 读行1 写行2） | 高 |
| WRITESET_SESSION | WRITESET + 同 session 程序顺序因果 | 最基本的也全覆盖了 | 最高精度 |

**COMMIT_ORDER 的核心局限**：它只能检测"不可能有读写因果"（Lock Interval 重叠），但不能检测"确实没有因果"（不同行的并发操作）。所以它过度保守——WRITESET 弥补了这个缺陷。

#### COMMIT_ORDER 能否保证因果一致性？

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

#### WRITESET 的假阴性：什么时候 WRITESET 会错误地抹掉因果依赖？

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

#### replica_preserve_commit_order：不检测因果，只强制顺序

MySQL 8.0 引入，**默认 ON**。核心类：`Commit_order_manager`（[rpl_replica_commit_order_manager.h](sql/rpl_replica_commit_order_manager.h)）。

##### 关键认知：它不记录因果关系，不检测冲突

很多人会误以为 `replica_preserve_commit_order` 用 MDL 锁来检测事务间的因果依赖——**完全不是这样。** 在你的 SELECT→UPDATE 例子中，T1 修改 `t1`，T2 修改 `t2`，两个不同表，根本不存在 MDL 冲突。`Commit_order_manager` 靠的是一种完全不同的机制。

##### 实际机制：纯位置的 FIFO 提交队列

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

##### Commit_order_lock_graph 的本质

它是 `MDL_wait_for_subgraph` 的子类（[rpl_replica_commit_order_manager.h](sql/rpl_replica_commit_order_manager.h)），但**它不对应任何真实数据库对象**——不锁表、不锁行、不锁元数据。它只是一个"等号牌"：

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

##### 类比：银行取号排队

```
这不是"冲突检测"（你没有和我抢同一个柜台），
而是"取号排队"（你号比我小，你先办完我才能办）。

InnoDB 锁 + MDL  = 柜台冲突检测："我们都在办同一张表？等一下吧"
Commit_order_manager = 取号机："不管你办哪张表，号小的先办"
```

**因果关系完全由"取号顺序 = binlog 顺序"隐含保证**。`Commit_order_manager` 不知道也不需要知道 T2 是否真的依赖 T1——只要号在 T1 之后，提交就必须在 T1 之后。这是一种比因果分析更保守但实现更简单的策略。

##### 状态机

（[rpl_replica_commit_order_manager.h](sql/rpl_replica_commit_order_manager.h)）

```
REGISTERED → FINISHED_APPLYING → 不是队列头 → REQUESTED_GRANT → WAITED → FINISHED
                                              ↑ 等前面 worker 先 commit ↑
                                    是队列头 → 直接提交（跳过 REQUESTED_GRANT）
```

##### 源码总览：Commit_order_manager 的真实类定义（8.0.39）

**命名核对**（8.0.39 已做过一轮 lock-free 重构，文件头 Copyright 2024）：`Commit_order_arc` **不存在**（等价物是 `Commit_order_lock_graph`）；`m_workers_map`/`m_granted`/`m_granted_workers` **不存在**；`wait_for_prior_commit()` **不存在**（真实函数是私有 `wait_on_graph(Slave_worker*)`）；`assign_ticket()` **不在** `Commit_order_manager`（同名函数属于 `Binlog_group_commit_ctx`，供 BGC 用；commit order 侧的"取号"发生在 `cs::apply::Commit_order_queue::push()`）；`get_rollbacker()` **不存在**（8.0.39 改为直接 `MDL_wait::set_status(VICTIM)`）；`signal_waiting()` **不存在**（唤醒靠 `set_status` 内部的 `mysql_cond_signal`）。

**真实成员**：

```cpp
class Commit_order_manager {
 private:
  std::atomic<bool> m_rollback_trx;               // 全通道唯一的"回滚传播位"
  cs::apply::Commit_order_queue m_workers;        // 提交顺序队列
 public:
  void register_trx(Slave_worker *worker);        // coordinator 派单时入队
  static bool wait(THD *thd);
  static void wait_and_finish(THD *thd, bool error);
  static bool get_rollback_status(THD *thd);
  static void finish_one(THD *thd);
  static bool wait_for_its_turn_before_flush_stage(THD *thd);
};
```

★ `m_workers` 不是普通 deque，是 `cs::apply::Commit_order_queue`——**双结构**：按 worker id 索引的静态 Node 数组 + 无锁 FIFO 整数队列（只存 worker id）。关键 Node：

```cpp
class Node {
  value_type m_worker_id{NO_WORKER};
  MDL_context *m_mdl_context{nullptr};        // 等待/唤醒的载体
  memory::Aligned_atomic<enum_worker_stage> m_stage{FINISHED};
  memory::Aligned_atomic<sequence_type> m_commit_sequence_nr{NO_SEQUENCE_NR}; // ★ ticket 序列号
  bool freeze_commit_sequence_nr(sequence_type expected);   // CAS 冻结，防双写
};
enum class enum_worker_stage { REGISTERED, FINISHED_APPLYING, REQUESTED_GRANT, WAITED, FINISHED };
```

**"取号"发生在 push 里**（8.0.39 版的 assign_ticket 等价物）：

```cpp
void Commit_order_queue::push(value_type index) {
  sequence_type next{Node::NO_SEQUENCE_NR};
  do { next = this->m_commit_sequence_generator->fetch_add(1); }
  while (next <= Node::SEQUENCE_NR_FROZEN);        // 保留 0/1 为哨兵值
  this->m_workers[index].m_commit_sequence_nr->store(next);
  this->m_commit_queue << index;                   // 追加到 FIFO 尾部
}
```

★ 序列号协议（头文件注释明说）：**持有序列号 N 的 worker 只能解封 N+1 的 worker**；N 正在执行解封操作（freeze 中）时，N+1 不能被分配新序列号——这正是 `finish_one` 里 freeze/unfreeze 的用途。

##### 复用 MDL 的完整机制：它到底"接入"了什么

> MDL 侧的完整死锁检测（BFS+DFS 遍历、`MAX_SEARCH_DEPTH=32`、权重 victim、`find_deadlock` 的 while 循环）见 [`../../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)「死锁检测」节——本节只讲 **`Commit_order_lock_graph` 如何作为第四个子类接进这套体系**。

**第一步：`MDL_wait_for_subgraph` 是"图边"的抽象**。MDL 死锁检测的图模型是：**节点 = `MDL_context`（每个线程一个），边 = `m_waiting_for`（"我在等谁"）**。而"在等谁"有四种可能：等一把 MDL 锁（`MDL_ticket`）、等 TABLE_SHARE 从 TDC 刷出（`Wait_for_flush`）、**等提交顺序轮到（`Commit_order_lock_graph`）**、单测 mock。接口只有两个纯虚函数：

```cpp
class MDL_wait_for_subgraph {
 public:
  /** Accept a wait-for graph visitor to inspect the node this edge leads to. */
  virtual bool accept_visitor(MDL_wait_for_graph_visitor *gvisitor) = 0;
  /* A helper used to determine which lock request should be aborted. */
  virtual uint get_deadlock_weight() const = 0;

  static const uint DEADLOCK_WEIGHT_CO = 0;
  static const uint DEADLOCK_WEIGHT_DML = 25;
  static const uint DEADLOCK_WEIGHT_ULL = 50;
  static const uint DEADLOCK_WEIGHT_DDL = 100;
};
```

`Commit_order_lock_graph` 就实现这两个：

```cpp
bool Commit_order_lock_graph::accept_visitor(MDL_wait_for_graph_visitor *visitor) {
  return this->m_mngr.visit_lock_graph(*this, *visitor);   // ★ 把遍历转发给 manager
}
uint Commit_order_lock_graph::get_deadlock_weight() const {
  return DEADLOCK_WEIGHT_CO;   // = 0，全图最低 ⇒ 死锁时永远先牺牲 worker
}
```

**第二步：`will_wait_for` 把"我在等提交顺序"注册成图里一条真边**（`sql/mdl.h` 内联实现）：

```cpp
  void will_wait_for(MDL_wait_for_subgraph *waiting_for_arg) {
    /*
      Before starting wait for any resource we need to materialize
      all "fast path" tickets belonging to this thread. Otherwise
      locks acquired which are represented by these tickets won't
      be present in wait-for graph and could cause missed deadlocks.
    */
    materialize_fast_path_locks();     // ★ 先物化 fast path ticket，否则图不完整漏检死锁

    mysql_prlock_wrlock(&m_LOCK_waiting_for);
    m_waiting_for = waiting_for_arg;    // ★ 单条边：每个 context 同时只能等一个东西
    mysql_prlock_unlock(&m_LOCK_waiting_for);
  }
  void done_waiting_for() {            // 醒来后摘边
    mysql_prlock_wrlock(&m_LOCK_waiting_for);
    m_waiting_for = nullptr;
    mysql_prlock_unlock(&m_LOCK_waiting_for);
  }
```

`m_waiting_for` 的类型是 `MDL_wait_for_subgraph *`——所以**同一根指针同时挂着"等 MDL 锁"和"等提交顺序"**，这决定了两种等待天然在同一张图里。`m_LOCK_waiting_for` 是 prlock（读多写少：死锁检测线程大量并发读别人的 `m_waiting_for`，只有本线程等/摘边时才写）。

**第三步：`visit_subgraph` 是死锁检测器进入"我在等的那个东西"的唯一入口**：

```cpp
bool MDL_context::visit_subgraph(MDL_wait_for_graph_visitor *gvisitor) {
  bool result = false;
  mysql_prlock_rdlock(&m_LOCK_waiting_for);
  if (m_waiting_for) result = m_waiting_for->accept_visitor(gvisitor);  // ★ 虚调用
  mysql_prlock_unlock(&m_LOCK_waiting_for);
  return result;
}
```

死锁检测线程顺着 `MDL_ticket` 链走到某节点时，调它的 `visit_subgraph` → 若该节点在等提交顺序，虚调用进 `Commit_order_lock_graph::accept_visitor` → `visit_lock_graph` 沿提交队列向前遍历——**在检测器眼里，"等 commit turn"和"等 MDL 锁"是同一张图里不同种类的边**。

**第四步：检测器三件套**（`Deadlock_detection_visitor`，`sql/mdl.cc`）：

```cpp
bool Deadlock_detection_visitor::inspect_edge(MDL_context *node) {
  m_found_deadlock = node == m_start_node;   // ★ 回到起点即环
  return m_found_deadlock;
}
bool Deadlock_detection_visitor::enter_node(MDL_context *node) {
  m_found_deadlock = ++m_current_search_depth >= MAX_SEARCH_DEPTH;  // 深度 32 兜底
  if (m_found_deadlock) opt_change_victim_to(node);
  return m_found_deadlock;
}
void Deadlock_detection_visitor::opt_change_victim_to(MDL_context *new_victim) {
  if (m_victim == nullptr ||
      m_victim->get_deadlock_weight() >= new_victim->get_deadlock_weight()) {
    MDL_context *tmp = m_victim;
    m_victim = new_victim;                  // ★ 权重最小者当选（>= 才换，平局保留先遇到）
    m_victim->lock_deadlock_victim();
    if (tmp) tmp->unlock_deadlock_victim();
  }
}
```

★ 这里有个微妙的耦合：`opt_change_victim_to` 调 `get_deadlock_weight()`——对 commit order 节点返回 0，于是只要环上出现 `Commit_order_lock_graph` 的节点（worker 在等 turn），它**必然**当选 victim。这就是"commit 等待永远优先被牺牲"的实现点。

**第五步：`find_deadlock` 的 while 循环**（为什么不是找一次就完）：

```cpp
void MDL_context::find_deadlock() {
  while (true) {
    Deadlock_detection_visitor dvisitor(this);   // ★ 每轮全新 visitor
    MDL_context *victim;
    if (!visit_subgraph(&dvisitor)) break;       // 无环退出
    victim = dvisitor.get_victim();
    (void)victim->m_wait.set_status(MDL_wait::VICTIM);   // ★ 点名：写等待槽 + signal
    victim->unlock_deadlock_victim();
    if (victim == this) break;
    /* 拆掉的是"别人的边"（victim 的等待），不是自己新加的边，
       可能还有别的环，必须重复搜索直到无环或自己成为 victim */
  }
}
```

★ `set_status(VICTIM)` 正是唤醒等待者的机制：victim 的 `m_wait` 是它自己在 `timed_wait` 上订阅的等待槽，`set_status` 写入状态并 `mysql_cond_signal` → victim 的 `timed_wait` 返回 `VICTIM` → 报 `ER_LOCK_DEADLOCK` → `report_commit_order_deadlock()` → worker 层 `retry_transaction` 重试。**MDL 只负责"点名"，回滚动作完全在 worker 层**——这解释了为什么 `Commit_order_manager::report_deadlock` 也可以直接 `set_status(VICTIM)` 直戳（复用同一个等待槽，不走图遍历）。

**"跨子图"的真正含义**：MySQL 里"事务等事务"有三套图——MDL 锁图（server 层）、InnoDB 行锁图（引擎层）、commit order 队列图（复制层）。前两者互不通信（mdl.md「死锁检测」节详述其后果）；而 commit order 图**通过 `MDL_wait_for_subgraph` 接入 MDL 图**，同时 InnoDB 侧又通过 `thd_report_lock_wait` 钩子把行锁等待"上报"进 commit order 检测——于是三套图两两衔接，`W2 等 W1 行锁 + W1 等 W2 提交` 这种跨层环能被查到（对应 `check_and_report_deadlock` 的 `sequence_number` 判据）。

##### wait_on_graph：队首判断与阻塞（核心）

```cpp
bool Commit_order_manager::wait_on_graph(Slave_worker *worker) {
  auto worker_thd = worker->info_thd;
  bool rollback_status{false};
  raii::Sentry<> wait_status_guard{[&]() -> void {
    worker_thd->mdl_context.m_wait.reset_status();
    this->m_workers[worker->id].m_stage = (rollback_status)
        ? ...REGISTERED   // 要回滚 -> 退回注册态等重试
        : ...WAITED;      // 正常 -> 已轮到
  }};

  this->m_workers[worker->id].m_stage = ...FINISHED_APPLYING;

  if (this->m_workers.front() != worker->id) {   // ★ 队首判断：队列头 worker id != 自己
    if (worker->found_commit_order_deadlock()) { // 已被别人标记成死锁牺牲者
      rollback_status = true;
      return true;
    }
    this->m_workers[worker->id].m_stage = ...REQUESTED_GRANT;

    Commit_order_lock_graph ticket{worker_thd->mdl_context, *this, worker->id};
    worker_thd->mdl_context.will_wait_for(&ticket);   // 把"等前任提交"注册成 wait-for 边
    worker_thd->mdl_context.find_deadlock();          // 立即做一次死锁检测
    raii::Sentry<> ticket_guard{[&]() { worker_thd->mdl_context.done_waiting_for(); }};

    struct timespec abs_timeout;
    set_timespec(&abs_timeout, LONG_TIMEOUT);          // ★ 名义超时一年 = 无超时
    auto wait_status = worker_thd->mdl_context.m_wait.timed_wait(
        worker_thd, &abs_timeout, true,                // signal_timeout=true
        &stage_worker_waiting_for_its_turn_to_commit);

    switch (wait_status) {
      case MDL_wait::GRANTED:  return false;           // 被 finish_one 放行
      case MDL_wait::TIMEOUT:  my_error(ER_LOCK_WAIT_TIMEOUT, MYF(0)); break;
      case MDL_wait::KILLED:   /* ER_QUERY_TIMEOUT / ER_QUERY_INTERRUPTED */ break;
      case MDL_wait::VICTIM:   my_error(ER_LOCK_DEADLOCK, MYF(0)); break;  // 死锁牺牲者
    }
    worker->report_commit_order_deadlock();
    rollback_status = true;
    return true;
  }
  return false;   // 自己是队首，直接进入 WAITED
}
```

★ **超时语义是"名义一年"**——保序场景不能因为等太久就放弃顺序；真正的退出路径只有 GRANTED（放行）/ VICTIM（死锁）/ KILLED。

##### finish_one：队首提交后放行下一个（核心）

```cpp
void Commit_order_manager::finish_one(Slave_worker *worker) {
  if (this->m_workers[worker->id].m_stage == ...WAITED) {
    assert(this->m_workers.front() == worker->id);   // 只有队首能放行

    auto [this_worker, this_seq_nr] = this->m_workers.pop();
    auto next_seq_nr = get_next_sequence_nr(this_seq_nr);  // N -> N+1

    auto next_worker = this->m_workers.front();
    if (next_worker != NO_WORKER &&
        (stage == FINISHED_APPLYING || stage == REQUESTED_GRANT) &&
        this->m_workers[next_worker].freeze_commit_sequence_nr(next_seq_nr)) {
      // ★ 只有序列号严格等于 N+1 的 worker 才有权被解封；
      //   freeze 防止它在本线程置 GRANTED 期间被 coordinator 重新注册拿到新号
      this->m_workers[next_worker].m_mdl_context->m_wait.set_status(
          MDL_wait::GRANTED);   // 写等待槽 + mysql_cond_signal 唤醒
      this->m_workers[next_worker].unfreeze_commit_sequence_nr(next_seq_nr);
    }
    this->m_workers[this_worker].m_stage = ...FINISHED;
  }
}
```

##### wait_and_finish：error 参数的两种退出路径（核心）

```cpp
void Commit_order_manager::wait_and_finish(THD *thd, bool error) {
  if (has_commit_order_manager(thd)) {
    Slave_worker *worker = dynamic_cast<Slave_worker *>(thd->rli_slave);
    Commit_order_manager *mngr = worker->get_commit_order_manager();

    if (error || worker->found_commit_order_deadlock()) {
      // 失败或被杀成死锁牺牲者：先判断还可不可以重试
      bool ret;
      std::tie(ret, std::ignore, std::ignore) =
          worker->check_and_report_end_of_retries(thd);
      if (ret) {                          // 不可重试 -> 才真正放弃队列位置
        mngr->wait(worker);               // 轮到自己的 turn 才能置回滚位（顺序不破坏）
        mngr->set_rollback_status();      // ★ 点亮 m_rollback_trx：后继全部回滚
        mngr->finish(worker);             // 出队 + 放行下一个
      }
    } else {
      mngr->wait(worker);                 // 正常路径：等 turn
      mngr->finish(worker);               // 提交完毕，出队放行
    }
  }
}
```

★ error 参数的语义差异：`false` = "我提交完了，让位并唤醒后继"；`true` = "我提交不了了，但**不立刻让位**——先问 `check_and_report_end_of_retries`"：可重试（死锁牺牲者）则**留在队里**等重试；不可重试（终局错误）才等 turn、置 `m_rollback_trx` 全局位、出队。之后所有后继 worker 在 `wait()` 里读到 rollback 位 → 统一让位 + `ER_REPLICA_WORKER_STOPPED_PREVIOUS_THD_ERROR`——**一条错误瀑布式传播，但队列位置一个不漏地释放**。

##### 死锁检测：check_and_report_deadlock 与跨子图遍历

入口是 InnoDB 行锁等待上报钩子 `thd_report_lock_wait`——**双方都是 mts worker** 时无条件调：

```cpp
void Commit_order_manager::check_and_report_deadlock(THD *thd_self,
                                                     THD *thd_wait_for) {
  Slave_worker *self_w = get_thd_worker(thd_self);
  Slave_worker *wait_for_w = get_thd_worker(thd_wait_for);
  Commit_order_manager *mngr = self_w->get_commit_order_manager();

  /* 同一通道 && 我等的那个 worker 序列号比我大（排在我后面） => 环 */
  if (mngr != nullptr && self_w->c_rli == wait_for_w->c_rli &&
      wait_for_w->sequence_number() > self_w->sequence_number()) {
    mngr->report_deadlock(wait_for_w);
  }
}

void Commit_order_manager::report_deadlock(Slave_worker *worker) {
  worker->report_commit_order_deadlock();
  this->m_workers[worker->id].m_mdl_context->m_wait.set_status(
      MDL_wait::VICTIM);   // ★ 直接戳对方等待槽：它醒来报 ER_LOCK_DEADLOCK
}
```

逻辑：W2 在 InnoDB 里等 W1 的行锁，而 W1 在 commit 顺序上排在 W2 **前面**（W1 若在等 W2 提交就成环）⇒ **牺牲"排在我后面"的 wait_for_w**——它拿着行锁却又等不了 W1 提交，必须回滚重试。

`Commit_order_lock_graph` 在 MDL 死锁检测里"假装"成图节点：`accept_visitor()` → `m_mngr.visit_lock_graph()`，沿提交队列**向前**迭代（只看排在自己前面的），对每个前驱 `visitor.inspect_edge(前驱 ctx)` 或递归 `前驱 ctx->visit_subgraph()` 下钻——`Deadlock_detection_visitor` 的 `inspect_edge` 里 `m_found_deadlock = (node == m_start_node)`（回到起点即环），搜索深度 ≥32 也直接判死锁（防栈溢出）。victim 按 `get_deadlock_weight()` 选**最低**者——`DEADLOCK_WEIGHT_CO = 0`（对比 DML=25 / ULL=50 / DDL=100），**commit 等待永远优先被牺牲**。victim 的回滚动作不由 MDL 执行，而是 worker 醒来看到 VICTIM → 报 `ER_LOCK_DEADLOCK` → 上层 `retry_transaction` 重试。

**开关 gate**：manager 实例只在 `opt_replica_preserve_commit_order && !is_parallel_exec() && opt_replica_parallel_workers > 1` 时 `new` 出来，否则 `commit_order_mngr = nullptr`；所有入口第一步 `has_commit_order_manager(thd)`——开关关闭或非 mts 线程时整条路径**统一退化为 no-op**。

##### 两个 worker 竞争队首的等待/唤醒时序

```
 W1(seq N, 队首)                                   W2(seq N+1)
   FINISHED_APPLYING ──► front()==自己                FINISHED_APPLYING ──► front()!=自己
   直接提交(不进等待)                                  REQUESTED_GRANT
                                                     will_wait_for(ticket_W2)    [边入图]
                                                     find_deadlock()             [查一遍环]
                                                     m_wait: WS_EMPTY
                                                     timed_wait() 阻塞            [等待槽订阅]

   finish_one(W1):
     m_workers.pop() ──► 拿到 N
     next_seq_nr = N+1
     freeze_commit_sequence_nr(N+1) ═══► 期间 coordinator 无法给 W2 再派新号
     W2.m_mdl_context->m_wait.set_status(GRANTED)
        └─ WS_EMPTY → GRANTED, mysql_cond_signal
     unfreeze_commit_sequence_nr(N+1) ═══╗
                                         ▼
                                        timed_wait 返回 GRANTED
                                        done_waiting_for()            [边摘除]
                                        stage = WAITED ──► 提交
```

| 决策 | 实现选择 | 原因 / 后果 |
|---|---|---|
| 等待载体 | 复用 `MDL_wait` 等待槽，不新建 cond | 白拿 kill 感知、超时关闭、PSI 阶段名 |
| 保序等待入 MDL 死锁图 | `Commit_order_lock_graph : MDL_wait_for_subgraph` | 行锁等 commit、commit 等行锁统一成环检测 |
| victim 权重 | `DEADLOCK_WEIGHT_CO = 0`（全图最低） | 死锁时永远牺牲 worker——事务可自动重试，代价最小 |
| 牺牲后的动作 | `m_wait.set_status(VICTIM)` + 标记 | MDL 只"点名"，回滚重试由 worker 层完成 |
| 队列实现 | 静态 Node 数组 + 无锁 FIFO + 自旋锁 | 免 mutex；`front()!=自己` 即队首判断 O(1) |
| 双唤醒竞态 | `freeze_commit_sequence_nr(N+1)` CAS 冻结 | 防前任已置 GRANTED 后，继任者又被重新注册拿新号 |
| 超时 | `LONG_TIMEOUT`（一年） | 语义"永不超时"——保序不能因超时而乱序 |
| 失败传播 | 单 `atomic<bool> m_rollback_trx` | 一个终局失败让队列全员快速让位退出 |
| 队首直通 | `front()==自己` 跳过图操作 | 热路径（队首）零 MDL 开销 |
| DDL 例外 | `wait_for_its_turn_before_flush_stage` 白名单 | 多阶段提交的 DDL 无法预判最后一次 commit |
| 与组提交合流 | `HA_IGNORE_DURABILITY` + `COMMIT_ORDER_FLUSH_STAGE` | 不写 binlog 的 worker 事务成组 flush |

#### 因果不一致的具体后果

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

#### GTID 空洞与因果一致性：两个维度的概念

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

#### 四层保障总结（修正版）

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

#### replica_preserve_commit_order 的归属：纯从库参数

**常见误解**：以为这个参数是主库设置的，主库保证提交顺序然后从库执行。

**真相**：参数定义中 `NOT_IN_BINLOG` 和 `ON_CHECK(check_slave_stopped)` 明确说明它是纯从库参数。源码定义（[sys_vars.cc](sql/sys_vars.cc)）：

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

#### 非 RBR 下 WRITESET 的数据一致性风险

**WRITESET 在非 RBR 下有真实的数据损坏风险。**

##### 回顾 WRITESET 假阴性场景

```
主库:
  T1: UPDATE t1 SET a=1 WHERE id=1          seq=1, writeset={hash(t1,id=1)}
  T2: BEGIN;
      SELECT a FROM t1 WHERE id=1           → 读到 a=1
      UPDATE t2 SET b=a*10 WHERE id=1
      COMMIT;                                seq=2, writeset={hash(t2,id=1)}

WRITESET 判定: hash(t1,id=1) ≠ hash(t2,id=1) → lc=0 → T2 "不依赖" T1（假阴性！）
```

##### RBR 下为什么安全

RBR 的 T2 binlog event 是：

```
#### UPDATE `db`.`t2`
#### WHERE @1=1
#### SET @2=10              ← 值已由主库算好，直接写入
```

从库 Worker-2 执行 T2 时只是把 `10` 写入 t2。不读 t1，不重新计算。即使 T1 还没 commit，T2 写入的值也是正确的。**因果链在 binlog 产生时已固化为最终行状态。**

##### 非 RBR（SBR）下为什么致命

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

##### 对比：各模式在不同 binlog format 下的安全性

| | RBR | SBR / MIXED(SBR 部分) |
|---|---|---|
| **COMMIT_ORDER** | ✅ 安全（假阴性=0） | ✅ 安全（假阴性=0，lc 为保守上界） |
| **WRITESET** | ✅ 安全（假阴性 + RBR = 行状态正确） | ❌ **数据损坏**（假阴性 + SBR = 读脏数据重算） |
| **WRITESET_SESSION** | ✅ 安全 | ❌ 同上 |

**关键**: COMMIT_ORDER 在 SBR 下也安全，因为 Lock Interval 理论保证了假阴性=0。T2 的 `lc=1` 意味着从库 T2 一定在 T1 之后执行（或被 preserve_commit_order 强制在 T1 后提交），读到的一定是 T1 更新后的值。

**WRITESET 的唯一安全前提是 RBR。** 这也是为什么 `binlog_format=ROW` 是 MySQL 8.0 的默认值，以及官方文档强烈建议并行复制配合 RBR 使用。

##### MySQL 有没有强制要求 WRITESET 必须配 RBR？

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

#### GTID 空洞与 preserve_commit_order：历史与演进

##### 2017 年 Taobao MySQL Monthly 文章的上下文

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

##### MySQL 8.0 的解决方案

`replica_preserve_commit_order=ON`（8.0 默认值）彻底解决了这个问题。Worker 执行完毕后不能立即提交，必须等在 FIFO 队列中排到自己才能提交——提交顺序严格等于 Coordinator 分发顺序（即 binlog 顺序）。

**所以在 MySQL 8.0 默认配置下，GTID 空洞不会因为并行复制产生。**

#### `Commit_order_queue` 内部机制：sequence_nr 的真实作用

很多人（包括之前的我）会误以为 `Commit_order_manager` 使用 binlog 的 `sequence_number` 来排序提交。**这不是真的。** 让我们从源码层面澄清两个不同的"序列号"。

##### 两个序列号，完全不同的用途

| | `Slave_worker::sequence_number()` | `Commit_order_queue::commit_sequence_nr` |
|---|---|---|
| **来源** | GAQ（binlog 的原始 sequence_number） | `m_commit_sequence_generator->fetch_add(1)` |
| **何时赋值** | Coordinator 分发事务时写入 GAQ | Worker `push()` 入队列时 |
| **含义** | 事务在主库 binlog 中的全局序号 | 提交请求在队列中的本地序号 |
| **用途** | 死锁检测方向判断 | CAS 原子协议保证只有前驱能解锁后继 |
| **与提交排序关系** | ❌ 无关 | ❌ 无关（排序靠的是队列位置） |

##### 提交排序靠什么：FIFO 队列的位置

```cpp
// commit_order_queue.cc
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

##### commit_sequence_nr 的真实用途：前驱-后继的原子解锁协议

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

##### sequence_number 的唯一用途：死锁检测中的方向判断

```cpp
// rpl_replica_commit_order_manager.cc
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

##### 一步到位：完整流程图

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

#### preserve_commit_order 的真正串行化位置：BGC Stage#0 vs ha_commit_low

**重要纠正**：之前的分析只关注了 `ha_commit_low` 中的 `wait`，遗漏了 BGC Stage#0 的关键作用。实际上根据从库是否写 binlog，等待发生在**不同位置**。

##### 两条路径总览

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

##### Stage#0 为什么存在？（[binlog.cc](sql/binlog.cc)）

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

关键判断：`ending_trans(thd, all=true)` → 恒为 `true`（[binlog.cc](sql/binlog.cc)：`return (all || ...)`）。

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

##### wait() 的状态机：第一次等，第二次 no-op

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

##### ha_commit_low 中的 wait_and_finish：释放后继

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

##### 最终时间线（log_replica_updates=ON，最常见情况）

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

##### undo 在哪写的？执行阶段，跟 commit order 无关

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

---




### 理论溯源：WL#7165

WL#7165 是 LOGICAL_CLOCK 的设计工作日志，其经典图解的完整逐事务分析见「核心实现 → WL#7165 经典图解分析」。核心思想：**用主库提交时的锁间隔重叠关系推断事务间无冲突**。

### 算法与数据结构

| 结构 | 作用 |
|---|---|
| `sequence_number` / `last_committed`（binlog） | 主库写入的两个逻辑时间戳，是从库调度的唯一输入 |
| writeset（三个 hash） | 行级冲突检测：主键/唯一键哈希集合 |
| GAQ（Global Assigned Queue） | 从库已分配事务的环形队列，LWM 在其上维护 |
| recovery bitmap（`recovery_groups`） | MTS 崩溃恢复时标记"哪些组已执行" |


---

### 他库对比与演进动机

| 系统 | 并行复制的做法 | 与 MySQL 的关键差异 |
|---|---|---|
| **MySQL（本篇）** | **主库在写 binlog 时就算好依赖**（`last_committed` / `sequence_number` 写进 Gtid 事件），从库只做调度、零判断 | 依赖判断**前置到主库** |
| **PostgreSQL** | 逻辑复制的订阅端长期是**单线程 apply**（并行 apply 能力是较晚版本才逐步引入的），发送端不在 WAL 里打"可并行"标记 | 并行决策在**订阅端**做，且长期没有主库侧的依赖标注 |
| **Oracle**（Data Guard / GoldenGate 类） | 并行 apply 多基于 **SCN / 事务依赖 / 按对象分片** 来判定可并行组 | 判定依据来自 redo/SCN 而非"提交时预先算好的时间戳" |

**MySQL 为什么要"前置到主库"**：从库没有主库的锁信息，也不理解 SQL 语义，要在从库判断冲突只能靠解析语句或加锁试错——成本极高且不安全（见 Q2/Q3）。把依赖在**主库提交时**算好并写进 binlog，从库就退化成一个纯调度器，只需比较两个整数。代价是主库侧有额外开销（writeset 的行哈希计算、`m_max_committed_transaction` 的原子维护），且依赖精度受主库能观测到的信息限制（无法覆盖引擎内部的级联写，故外键降级）。

**演进动机（对应版本演进表）**：5.6 的库级并行在单库热点下完全失效 → 5.7 LOGICAL_CLOCK 用提交顺序（锁间隔）把粒度降到事务级 → 8.0 WRITESET 再把粒度降到**行级**（哈希），以摆脱"是否同库/是否同时提交"的限制。每一步都是**依赖粒度变细 ⇒ 并行度上升**，代价是主库计算量上升。

## 核心实现

> 本章按执行流程分组：先给主链路，再按"主库标记依赖 → 从库调度 → 数据结构与恢复"展开。

### 主链路

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

#### 涉及的核心源码文件

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



### 核心概念：两个逻辑时间戳

每个事务在 binlog 中携带两个逻辑时间戳，嵌入在 `Gtid_log_event` 的 **post-header** 中：

#### sequence_number（事务序列号）

- **含义**：事务在 binlog 中的全局自增序号，唯一标识一个事务
- **分配时机**：`binlog_cache_data::flush()` → `m_dependency_tracker.step()` → `m_transaction_counter.step()` → `++state`
- **物理存储**：Gtid_log_event post-header 的最后 8 字节

```cpp
// binlog.cc
// for MTS. sequence_number::state++
trn_ctx->sequence_number = mysql_bin_log.m_dependency_tracker.step();
```

#### last_committed（commit parent）

- **含义**：当前事务必须等待的"父事务"的 `sequence_number`。从库必须等到 `last_committed` 号事务提交完成后，才能开始执行当前事务
- **记录时机**：事务获取完所有锁的那一刻，调用 `store_commit_parent()` 对当前 `m_max_committed_transaction` 做快照
- **物理存储**：Gtid_log_event post-header 的倒数第 9-16 字节

```cpp
// transaction_info.h
void store_commit_parent(int64 last_arg) { last_committed = last_arg; }
```

#### 调用时机

`store_commit_parent` 在多处被调用，对应不同的事务类型：

| 位置 | 场景 |
|---|---|
| [binlog.cc](sql/binlog.cc) | XA prepare（第一阶段提交） |
| [binlog.cc](sql/binlog.cc) | 普通事务，`flush_thread_caches` 时（锁已全部获取） |
| [binlog.cc](sql/binlog.cc) | 非事务型语句（如 DDL） |

#### 关键：绝对时间 vs 相对时间

`Logical_clock` 内部维护了 `state`（绝对时钟）和 `offset`（偏移量）：

```cpp
// rpl_trx_tracking.h
class Logical_clock {
private:
  std::atomic<int64> state;   // 全局绝对时钟
  int64 offset;               // binlog 轮转时的偏移量
```

- `Transaction_ctx::sequence_number` 和 `Transaction_ctx::last_committed` 存储的是**绝对值**（state 的值）
- 写入 binlog 时，`get_dependency()` 会减去 `offset` 转为**相对值**，使得 binlog 中的值总是从 1 开始递增
- binlog 轮转后，`offset` 更新为新 binlog 的起点，旧相对值失去意义，`last_committed` 回到 `SEQ_UNINIT`

---



##### 逻辑时间戳与依赖追踪的完整链路

> ★ **归属澄清（先分清，否则整章都会读串）**：本节讲的 `Transaction_dependency_tracker` 与 `Logical_clock` **全部是主库侧**——`m_dependency_tracker` 是 `MYSQL_BIN_LOG` 的成员（`sql/binlog.h`），它在 **binlog 写入路径**（`ordered_commit` / `write_transaction`）上分配时间戳并写入 `Gtid_log_event`。从库**不计算**依赖，只**消费**这两个时间戳（见「从库侧：MTS 调度」）。
>
> 这也是本篇最大的命名陷阱：**主库和从库各有一个 "logical clock"**，详见 Misc「容易误解的命名」。

两个时间戳：`sequence_number`（本事务在 binlog 中的序号）与 `last_committed`（commit parent，从库上"必须在我之前完成"的那条事务）。从库 `wait_for_last_committed_trx()` 阻塞直到 `clock_leq(last_committed, lwm)`，即**等待所有 seq ≤ last_committed 的事务执行完**，所以 T2 能与 T1 并行的充要条件是 `last_committed(T2) < sequence_number(T1)`。

**主链路：从事务提交到 binlog 落盘**

```
用户线程                         Group Commit（leader 代跑 flush/sync）
────────────────────────────────────────────────────────────────────────
ha_prepare_low(all)
  └ binlog_prepare(hton,thd,!all) ──► trn_ctx->store_commit_parent(
                                        get_max_committed_timestamp())   ← 快照 A
MYSQL_BIN_LOG::commit()
  └ (stmt_cache 非空) store_commit_parent(get_max_committed_timestamp()) ← 快照 B
  └ 各 cache finalize()  (event → IO_CACHE)
  └ MYSQL_BIN_LOG::ordered_commit()
      Stage#1 FLUSH  process_flush_stage_queue()
               └ assign_automatic_gtids_to_flush_group()
               └ flush_thread_caches(head) → binlog_cache_data::flush()
                    ├ trn_ctx->sequence_number = m_dependency_tracker.step()   ← 分配 sn
                    ├ if (last_committed==SEQ_UNINIT) last_committed = sn - 1
                    └ MYSQL_BIN_LOG::write_transaction()
                         ├ m_dependency_tracker.get_dependency()   (绝对 sn/lc → 相对值)
                         └ Gtid_log_event(..., last_committed, sequence_number, ...)
                              └ gtid_event.write(writer)                        ← 落盘
      Stage#2 SYNC   sync_binlog_file()
      Stage#3 COMMIT process_commit_stage_queue()
               └ m_dependency_tracker.update_max_committed(head)                ← 快照 C
               └ finish_transaction_in_engines()  (ha_commit_low 释放锁)
      ⋯ finish_commit() 里对 follower 重复一次 update_max_committed()
      ⋯ rotate(force_rotate) → new_file_impl → open_binlog
                             → m_dependency_tracker.rotate()                     ← offset 重置
```

`sequence_number` 在 **flush 阶段**由全局计数器 `step()` 分配；`last_committed` 早在 **prepare/语句末尾**就快照好了；真正写进 `Gtid_log_event` 的是 `get_dependency()` 换算后的**相对值**。

**分派入口（三模式串联，而非三选一）**

```cpp
void Transaction_dependency_tracker::get_dependency(
    THD *thd, bool parallelization_barrier, int64 &sequence_number,
    int64 &commit_parent) {
  sequence_number = commit_parent = 0;
  switch (m_opt_tracking_mode) {
    case DEPENDENCY_TRACKING_COMMIT_ORDER:
      m_commit_order.get_dependency(thd, parallelization_barrier,
                                    sequence_number, commit_parent);
      break;
    case DEPENDENCY_TRACKING_WRITESET:
      m_commit_order.get_dependency(thd, parallelization_barrier,
                                    sequence_number, commit_parent);
      m_writeset.get_dependency(thd, sequence_number, commit_parent);
      break;
    case DEPENDENCY_TRACKING_WRITESET_SESSION:
      m_commit_order.get_dependency(thd, parallelization_barrier,
                                    sequence_number, commit_parent);
      m_writeset.get_dependency(thd, sequence_number, commit_parent);
      m_writeset_session.get_dependency(thd, sequence_number, commit_parent);
      break;
  }
}
```

COMMIT_ORDER 必须先算出一个安全的（保守的）`commit_parent`，WRITESET 只能在它的基础上**往下压**（`std::min`），WRITESET_SESSION 只能再**往上抬**（`std::max`）。因此 **COMMIT_ORDER 是永不失效的正确性下限**。

**COMMIT_ORDER 模式**

```cpp
void Commit_order_trx_dependency_tracker::get_dependency(
    THD *thd, bool parallelization_barrier, int64 &sequence_number,
    int64 &commit_parent) {
  Transaction_ctx *trn_ctx = thd->get_transaction();
  assert(trn_ctx->sequence_number > m_max_committed_transaction.get_offset());
  sequence_number =
      trn_ctx->sequence_number - m_max_committed_transaction.get_offset();   // ① 绝对 → 相对

  if (trn_ctx->last_committed <= m_max_committed_transaction.get_offset())
    commit_parent = SEQ_UNINIT;                                              // ② 父事务在上一个文件
  else
    commit_parent =
        std::max(trn_ctx->last_committed, m_last_blocking_transaction) -
        m_max_committed_transaction.get_offset();                            // ③

  if (is_trx_unsafe_for_parallel_slave(thd) || parallelization_barrier)
    m_last_blocking_transaction = trn_ctx->sequence_number;                  // ④
}
```

1. ① 减 `offset` 把绝对序号变成当前 binlog 内的相对值；`assert` 保证本事务属于当前文件
2. ② `last_committed <= offset` 说明父事务属于**上一个 binlog 文件**，跨文件比较无意义 → 退化 `SEQ_UNINIT`，从库见到它立即返回、不等待，**天然获得跨文件并行度**
3. ③ 取 `max(自己的 lc, m_last_blocking_transaction)`：后者是"破坏了锁时钟假设"的最后一条事务（DDL / `ANALYZE|OPTIMIZE|REPAIR` / `CREATE|ALTER|DROP DB` / binlog 缓存丢失产生的 incident），它之后的全部事务必须挂在它后面
4. ④ 本事务若自己就是这类不安全事务，则登记为新的 barrier

**Logical_clock：绝对 state + 每文件 offset**

```cpp
inline int64 Logical_clock::step() { return ++state; }

inline int64 Logical_clock::set_if_greater(int64 new_val) {
  int64 old_val = new_val - 1;
  if (new_val <= offset) return SEQ_UNINIT;        // 轮转后迟到的提交不再推进时钟
  while (!atomic_compare_exchange_strong(&state, &old_val, new_val) &&
         old_val < new_val) { }
  return cas_rc ? new_val : old_val;
}

void Commit_order_trx_dependency_tracker::rotate() {
  m_max_committed_transaction.update_offset(m_transaction_counter.get_timestamp());
  m_transaction_counter.update_offset(m_transaction_counter.get_timestamp());
}
void Transaction_dependency_tracker::rotate() {
  m_commit_order.rotate();
  m_writeset.rotate(1);
  if (current_thd) current_thd->get_transaction()->sequence_number = 2;
}
```

- `state` 是**单调不重置**的全局绝对计数，`offset` 是"当前 binlog 文件起始时的 state 快照"；binlog 里存 `state - offset`，因为**时间戳只在当前文件内有意义**
- `rotate()` 由 `ordered_commit → rotate → new_file_impl → open_binlog` 触发，把两个时钟的 offset 抬到当时的 state，新文件第一条事务的相对 `sequence_number` 重新从 1 开始
- `set_if_greater()` 的 `new_val <= offset` 分支与 ② 的 `SEQ_UNINIT` 对称，**两者共同保证跨 binlog 文件的事务零依赖**

**`m_max_committed_transaction` 的更新时机 = 2PL 锁区间的右端点**

```cpp
void Transaction_dependency_tracker::update_max_committed(THD *thd) {
  Transaction_ctx *trn_ctx = thd->get_transaction();
  m_commit_order.update_max_committed(trn_ctx->sequence_number);
  trn_ctx->sequence_number = SEQ_UNINIT;
}
```

调用点只有两处：`process_commit_stage_queue()`（在 `finish_transaction_in_engines()` **之前**）与 `finish_commit()`。

```
T1: ──[ 锁区间：取得全部锁 ─────────────────────────────── ]──┐
        ↑快照A/B(last_committed)      ↑commit stage:         ↑engine commit 放锁
                                      max_committed ← sn
T2 与 T1 冲突：T2 必须等 T1 放锁才能取齐锁
    ⇒ T2 快照点 > T1 放锁点 > T1 commit stage
    ⇒ last_committed(T2) ≥ sn(T1) ⇒ 从库串行 ✓

T2 与 T1 不冲突且区间重叠：
T1: ──────────[ 锁区间 ────────────────────────────── ]──┐
                       ↑commit stage (max_committed←sn)  ↑放锁
T2:        ┌──[ 锁区间 ...
           ↑T2 快照（早于 T1 commit stage）
    ⇒ last_committed(T2) < sn(T1) ⇒ 从库并行 ✓
```

为什么**右端点必须在放锁之前**：若放到 `ha_commit_low()` 之后，"T1 已放锁、T2 才拿到锁"的冲突对会被判成并行，正确性崩塌。放在 commit stage（binlog 已 flush、事务注定提交、锁仍持有）则保证 `sn(T1) ∉ max_committed ⟹ T1 还没放锁 ⟹ 区间必与 T2 重叠`。**代价**是损失一点并行度。

| 决策 | 代码落点 | 为什么 | 代价 |
| --- | --- | --- | --- |
| 时钟 = 绝对 `state` + 每文件 `offset` | `Logical_clock::state/offset` | 时间戳只在当前 binlog 内可比，天然隔离跨文件依赖 | 落盘前多做一次减法 |
| 轮转后父事务退化 `SEQ_UNINIT` | `get_dependency()` | 跨文件比较无意义，直接给最大并行度 | 由 `m_last_blocking_transaction` 兜底 |
| sn 在 flush 阶段、lc 在 prepare 阶段快照 | `binlog_cache_data::flush`、`binlog_prepare` | 分别对应 2PL 锁区间的右/左端点 | 需 `LOCK_replica_trans_dep_tracker` 保护 |
| `max_committed` 在 engine commit **之前**推进 | `process_commit_stage_queue` | 保证"未进 max_committed ⟹ 仍持锁" | 牺牲少量并行度 |
| 三模式串联而非互斥 | `get_dependency()` | COMMIT_ORDER 恒为正确性下限 | 每次提交都要跑 COMMIT_ORDER |
| `m_last_blocking_transaction` 全局 barrier | `is_trx_unsafe_for_parallel_slave()` | DDL/早放 MDL 破坏锁时钟假设 | 一条 DDL 拉平后续并行度 |

### 主库侧：依赖追踪

> 本节讲**三种追踪模式各自的算法与理论**（COMMIT_ORDER 的锁间隔、WRITESET 的行哈希、WRITESET_SESSION 的会话约束、三层漏斗）。
> 与之配套的两处剖析见别章，本节不重复：**时间戳的完整产生链路**（主链路栈、`get_dependency` 分派、`Logical_clock` 的 state/offset、`update_max_committed` 与 2PL 的对应）见「核心概念：两个逻辑时间戳」；**WRITESET 的代码级实现**（`get_dependency` 完整代码、`add_pke`、降级表、history 容量）见「Writeset Hash 生成机制」。

#### 三种依赖追踪模式

由参数 `binlog_transaction_dependency_tracking` 控制。定义在 [rpl_trx_tracking.h](sql/rpl_trx_tracking.h)：

| 模式 | 枚举值 | 含义 |
|---|---|---|
| `COMMIT_ORDER` | 0 | 基于事务提交顺序和锁持有时间窗口判定冲突 |
| `WRITESET` | 1 | 在 COMMIT_ORDER 基础上，额外使用行级 hash 检测 |
| `WRITESET_SESSION` | 2 | 在 WRITESET 基础上，强制同一 session 的事务串行 |

#### COMMIT_ORDER 详解：Lock Interval 理论

这是 LOGICAL_CLOCK 并行的**理论基础**。源码注释给出了完整定义（[rpl_trx_tracking.h](sql/rpl_trx_tracking.h)）：

> The time intervals during which any transaction holds all its locks are tracked (the interval ends just before storage engine commit, when locks are released. For an autocommit transaction it begins just before storage engine prepare. For BEGIN..COMMIT transactions it begins at the end of the last statement before COMMIT). **Two transactions are marked as non-conflicting if their respective intervals overlap.** In other words, if trx1 appears before trx2 in the binlog, and trx2 had acquired all its locks before trx1 released its locks, then trx2 is marked such that the slave can schedule it in parallel with trx1.

##### Lock Interval 的定义

一个事务的 Lock Interval 是从"获取完所有行锁"到"存储引擎提交（释放锁）"之间的时间窗口：

- **autocommit 事务**：从存储引擎 prepare 开始，到存储引擎 commit（释放锁）结束
- **BEGIN...COMMIT 显式事务**：从最后一条语句执笔结束，到 COMMIT 结束

##### 判定规则图解（精确时间线）

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

##### 量化逻辑

```cpp
// Commit_order_trx_dependency_tracker 内部维护两个时钟：
// rpl_trx_tracking.h
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

##### COMMIT_ORDER 的保守性

情况B 中，T1 和 T2 的锁间隔不重叠，但这**不意味着它们一定有行级冲突**。它们可能修改的是完全不相关的行（不同表、不同 ID）。COMMIT_ORDER 保守地标记为串行，从而产生了**假阳性**（false positive）。这正是 WRITESET 模式要解决的问题。

#### WRITESET 详解：行级冲突检测

WRITESET 模式在 COMMIT_ORDER 计算出 `commit_parent` 的基础上，使用行级 hash 进一步缩小依赖范围。

**核心数据结构**（[rpl_trx_tracking.h](sql/rpl_trx_tracking.h)）：

```cpp
typedef std::map<uint64, int64> Writeset_history;
Writeset_history m_writeset_history;  // 行hash → 最后修改该行的seq_no
int64 m_writeset_history_start;       // history 覆盖的最小编号
std::atomic<ulong> m_opt_max_history_size;  // history 最大容量，默认 25000
```

**算法**（[rpl_trx_tracking.cc](sql/rpl_trx_tracking.cc)）：

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

#### WRITESET_SESSION 详解：会话级约束

在 WRITESET 基础上，额外保证**同一 session 内的事务必须串行**：

```cpp
// rpl_trx_tracking.cc
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

#### 三层漏斗模型

`Transaction_dependency_tracker::get_dependency()` 是统一入口，三种模式构成**三层漏斗**：

```cpp
// rpl_trx_tracking.cc
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

#### 写入 binlog：序列化到 Gtid_log_event

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



### 从库侧：MTS 调度

#### 两种调度模式

由 `replica_parallel_type` 控制，定义在 [rpl_mta_submode.h](sql/rpl_mta_submode.h)：

| 模式 | 实现类 | 调度依据 | 并行度 |
|---|---|---|---|
| `DATABASE` | `Mts_submode_database` | 按 database name hash 分配 worker | 较低，跨库事务退化 |
| `LOGICAL_CLOCK` | `Mts_submode_logical_clock` | 基于主库写入的 `last_committed` / `sequence_number` | 较高，理论上可接近主库并发度 |

#### LOGICAL_CLOCK 调度流程

`Mts_submode_logical_clock` 的核心方法（[rpl_mta_submode.h](sql/rpl_mta_submode.h)）：

**关键成员变量：**

```cpp
longlong last_committed;       // 当前事务的 commit parent
longlong sequence_number;      // 当前事务的序列号
std::atomic<longlong> last_lwm_timestamp;  // 当前 LWM（Low Water Mark）
ulong last_lwm_index;          // LWM 对应的 GAQ index
bool is_new_group;             // 是否需要强制开始新的事务组（等所有worker完成）
```

**步骤1：提取时间戳（schedule_next_event）**（[rpl_mta_submode.cc](sql/rpl_mta_submode.cc)）

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

**步骤2：判断是否需要 is_new_group（强制串行点）**（[rpl_mta_submode.cc](sql/rpl_mta_submode.cc)）

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

**步骤3：依赖判断与等待**（[rpl_mta_submode.cc](sql/rpl_mta_submode.cc)）

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

**`clock_leq` 的定义**（[rpl_mta_submode.h](sql/rpl_mta_submode.h)）：

```cpp
static bool clock_leq(longlong a, longlong b) {
  if (a == SEQ_UNINIT) return true;   // SEQ_UNINIT 视为最小
  else if (b == SEQ_UNINIT) return false;
  else return a <= b;
}
```

**步骤4：等待的实现（wait_for_last_committed_trx）**（[rpl_mta_submode.cc](sql/rpl_mta_submode.cc)）

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

#### LWM（Low Water Mark）维护

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

**LWM 计算**（[rpl_mta_submode.cc](sql/rpl_mta_submode.cc)）：

```cpp
longlong Mts_submode_logical_clock::get_lwm_timestamp(Relay_log_info *rli, bool need_lock) {
  last_lwm_index = rli->gaq->find_lwm(&ptr_g, ...);
  last_lwm_timestamp = ptr_g->sequence_number;
}
```

#### 依赖等待与事件分派：两个核心函数

**（1）怎么等依赖：`Mts_submode_logical_clock::wait_for_last_committed_trx`**

```cpp
bool Mts_submode_logical_clock::wait_for_last_committed_trx(
    Relay_log_info *rli, longlong last_committed_arg) {
  if (last_committed_arg == SEQ_UNINIT) return false;     // ① 无依赖标记 → 不用等

  mysql_mutex_lock(&rli->mts_gaq_LOCK);
  min_waited_timestamp.store(last_committed_arg);         // ② 登记"我在等哪一个时间戳"

  if ((!rli->info_thd->killed && !is_error) &&
      !clock_leq(last_committed_arg, get_lwm_timestamp(rli, true))) {   // ③ ★ 依赖是否已满足
    thd->ENTER_COND(&rli->logical_clock_cond, &rli->mts_gaq_LOCK,
                    &stage_worker_waiting_for_commit_parent, &old_stage);
    do {
      mysql_cond_wait(&rli->logical_clock_cond, &rli->mts_gaq_LOCK);
    } while ((!rli->info_thd->killed && !is_error) &&
             !clock_leq(last_committed_arg, estimate_lwm_timestamp())); // ④ 醒来重判
    min_waited_timestamp.store(SEQ_UNINIT);
    mysql_mutex_unlock(&rli->mts_gaq_LOCK);
    thd->EXIT_COND(&old_stage);
    rli->mts_total_wait_overlap += diff_timespec(&ts[1], &ts[0]);        // ⑤ 统计等待耗时
  } else {
    min_waited_timestamp.store(SEQ_UNINIT);
    mysql_mutex_unlock(&rli->mts_gaq_LOCK);              // ⑥ 依赖已满足 → 直接放行
  }
  return rli->info_thd->killed || is_error;
}
```

要点：

- **③ 的判据是 `last_committed <= LWM 时间戳`**：LWM 之前的组都已提交 ⇒ 依赖必然满足。这就是"**并行度由 LWM 决定**"的直接代码体现（见「GAQ 与 Checkpoint 机制」）
- 等待挂在 `logical_clock_cond` 上，**由 checkpoint 推进 LWM 后广播唤醒**——GAQ 的 checkpoint 越慢，等待越久
- `min_waited_timestamp` 供诊断使用：记录当前最小等待目标，`estimate_lwm_timestamp()` 会参考它
- ④ 用 `do...while` 而非 `if`：条件变量有虚假唤醒，且等待期间 LWM 可能被别的线程推进

**（2）怎么分派：`Log_event::get_slave_worker`**

```cpp
Slave_worker *Log_event::get_slave_worker(Relay_log_info *rli) {
  ...
  if ((is_s_event = starts_group()) || is_gtid_event(this) ||
      (!rli->curr_group_seen_begin && !rli->curr_group_seen_gtid &&
       (gaq->empty() ||
        gaq->get_job_group(rli->gaq->assigned_group_index)->worker_id !=
            MTS_WORKER_UNDEF))) {                        // ① ★ 判定"这是一个新组的开始"
    if (!rli->curr_group_seen_gtid && !rli->curr_group_seen_begin) {
      rli->mts_groups_assigned++;                        // ② 组计数 +1
      group.reset(common_header->log_pos, rli->mts_groups_assigned);
      gaq->assigned_group_index = gaq->en_queue(&group);  // ③ ★ 入 GAQ（严格按主库顺序）
      ...
      if (is_s_event || is_gtid_event(this)) {
        Slave_job_item job_item = {this, rli->get_event_start_pos(), {'\0'}};
        rli->curr_group_da.push_back(job_item);           // ④ B/Gtid 事件先进延迟数组
        if (starts_group()) { rli->curr_group_seen_begin = true; ... }
        if (is_gtid_event(this)) {
          rli->curr_group_seen_gtid = true;
          rli->started_processing(gtid_log_ev);
        }
        if (schedule_next_event(this, rli)) { ... }        // ⑤ ★ 调度（内部触发上面的等待）
      }
```

要点：

- **① "新组开始"的三种来源**：显式 BEGIN、GTID 事件、或"既没见 BEGIN 也没见 Gtid，但上一组已分派出去"（B-free 的多事件组，如 `{p1,p2,...,pk,g}`）
- **③ 入 GAQ 严格按主库顺序** ⇒ GAQ 中 `sequence_number` 单调，这是 `find_lwm` 能正确算水位的前提
- **④ B / Gtid 事件先攒在 `curr_group_da`（延迟数组）**：这两个事件本身不决定 worker 归属，等确定分给哪个 worker 后一次性投递
- **⑤ 等待发生在"确定分给谁之前"**：`schedule_next_event` 内部调 `wait_for_last_committed_trx`，依赖满足后才真正选 worker 并投递

#### Worker 分配策略

**同一事务的连续性保证**：同一个事务内的所有 event 必须交给同一个 worker 执行，并且该 worker 先执行完这个事务的所有 event 后才能接受下一个事务。

```cpp
// rpl_mta_submode.cc
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



##### 选出 worker：`get_least_occupied_worker`

```cpp
Slave_worker *Mts_submode_logical_clock::get_least_occupied_worker(
    Relay_log_info *rli, Slave_worker_array *ws, Log_event *ev) {
  if (rli->last_assigned_worker) {          // ① 组内后续事件：钉死在同一 worker
    worker = rli->last_assigned_worker;
  } else {                                  // ② 组首事件：找空闲 worker
    worker = get_free_worker(rli);          //   判据 = 该 worker 的 jobs 队列为空
    if (worker == nullptr) {                // 全都忙 → 忙等 yield
      while (!worker && !thd->killed) { sched_yield(); worker = get_free_worker(rli); }
      rli->mts_total_wait_worker_avail += diff_timespec(&ts[1], &ts[0]);
    }
    if (rli->get_commit_order_manager() != nullptr && worker != nullptr)
      rli->get_commit_order_manager()->register_trx(worker);   // preserve_commit_order 登记
  }
  return worker;
}
```

**`last_assigned_worker` 的三重作用**：(a) **组内粘性**——一个事务的所有事件必须到同一个 worker，否则事务被撕裂；(b) 组结束（`MTS_END_GROUP`）时置 `nullptr`，下一组重新找空闲者，这正是"**每个 worker 至多一个未完成组**"不变式的实现；(c) `apply_event` 里 `rli->last_assigned_worker = get_slave_worker(rli)`——"这次选中的"就是"下次粘的"。

**延迟数组的一次性投递**（`apply_event_and_update_pos`）：

```cpp
      if (rli->curr_group_da.size() > 0) {            // ★ B/Gtid 等攒下的事件一次性灌给同一 worker
        for (uint i = 0; i < rli->curr_group_da.size(); i++) {
          Slave_job_item da_item = rli->curr_group_da[i];
          append_item_to_jobs(&da_item, w, rli);
        }
        rli->curr_group_da.clear();
      }
      append_item_to_jobs(job_item, w, rli);          // 当前事件最后入队
      *ptr_ev = nullptr;                              // ★ 事件所有权交给 worker
```

`*ptr_ev = nullptr` 是**所有权转移的显式约定**：coordinator 靠"指针是否被置空"区分"事件已交给 worker"与"自己执行完了"。

##### 调度决策中枢：`schedule_next_event`

```cpp
int Mts_submode_logical_clock::schedule_next_event(Relay_log_info *rli, Log_event *ev) {
  Slave_job_group *ptr_group = rli->gaq->get_job_group(rli->gaq->assigned_group_index);
  switch (ev->get_type_code()) {
    case binary_log::GTID_LOG_EVENT:
    case binary_log::ANONYMOUS_GTID_LOG_EVENT:
      ptr_group->sequence_number = sequence_number =
          static_cast<Gtid_log_event *>(ev)->sequence_number;
      ptr_group->last_committed = last_committed =
          static_cast<Gtid_log_event *>(ev)->last_committed;
      break;
    default:
      sequence_number = last_committed = SEQ_UNINIT;   // ① 无时钟 → 强制串行
      break;
  }
  if (first_event) first_event = false;
  else {
    if (clock_leq(sequence_number, last_committed) && last_committed != SEQ_UNINIT)
      return ER_MTA_CANT_PARALLEL;                     // ② 一致性校验
    if (sequence_number > last_sequence_number + 1) gap_successor = true;  // ③ 序号有空洞
  }
  is_new_group =
      (first_event || force_new_group || sequence_number == SEQ_UNINIT ||
       last_committed == SEQ_UNINIT || gap_successor ||
       last_sequence_number == SEQ_UNINIT);            // ④ ★ 分叉点

  if (!is_new_group) {                                 // ---- 常规：只等依赖的那一个 ----
    if (wait_for_last_committed_trx(rli, last_committed)) return -1;
    delegated_jobs++;
  } else {                                             // ---- 依赖不可追踪：全同步 ----
    if (-1 == wait_for_workers_to_finish(rli)) return ER_MTA_INCONSISTENT_DATA;
    rli->gaq->lwm.sequence_number = last_lwm_timestamp = SEQ_UNINIT;
    delegated_jobs = 1; jobs_done = 0; force_new_group = false;
  }
  return 0;
}
```

- ① **只有 Gtid 事件带时间戳**，其余事件一律 `SEQ_UNINIT`，天然落进 `is_new_group`
- ② `last_committed < sequence_number` 且 sn 单调增，违反即 `ER_MTA_CANT_PARALLEL` 停复制——防止主库时钟出错导致从库乱序提交
- ③ `gap_successor`：序号有空洞（如 `replicate_same_server_id` 过滤）⇒ 依赖不可追踪 ⇒ 保守当新组
- **④ 就是"要不要全同步"的唯一分叉**：真 → `wait_for_workers_to_finish`；假 → `wait_for_last_committed_trx`

**GAQ 容量与 checkpoint 触发**在 coordinator 主循环里，不在 `schedule_next_event`：

```cpp
  bool force = rli->rli_checkpoint_seqno >= rli->checkpoint_group;   // replica_checkpoint_group
  if (force || rli->is_time_for_mta_checkpoint())                    // replica_checkpoint_period(ms)
    mta_checkpoint_routine(rli, force);
```

##### coordinator 主循环与返回值语义

```
handle_slave_sql: while (!main_loop_error && !sql_slave_killed) {
    read_next_event()
    switch (exec_relay_log_event(...)) {
      case OK:            /* 继续读下一个 */
      case UNTIL_REACHED: /* 下一轮自然退出 */
      case RETRY:         /* 单线程重试：同一事件再读一次 */
      case APPLY_ERROR / UPDATE_POS_ERROR / APPEND_JOB_ERROR: main_loop_error = true;
    } }
```

| 返回值 | 含义 |
|---|---|
| `OK` (0) | 已应用（或被 `sql_delay` 推迟） |
| `APPLY_ERROR` (1) | `apply_event()` 失败 |
| `UPDATE_POS_ERROR` (2) | apply 成功但 `update_pos()` 失败，不重试，直接停 |
| `APPEND_JOB_ERROR` (3) | `append_item_to_jobs()` 失败（worker 队列满时被 kill） |
| `RETRY` (4) | 可重试的临时错误，下轮重读同一事件（**仅单线程**） |
| `UNTIL_REACHED` (5) | 命中 `START REPLICA UNTIL` |

注意 `>= UPDATE_POS_ERROR` 把 2/3 一起判死，而 1 有 `slave_trans_retries` 兜底；但 **MTS 模式下不重试**（`is_mts_worker(thd)` 时跳过）——事务重试由 worker 自己 `retry_transaction` 从 relay log 重读整组。

##### worker 侧执行与 bitmap 更新

```cpp
int slave_worker_exec_job_group(Slave_worker *worker, Relay_log_info *rli) {
  job_item = pop_jobs_item(worker, job_item);        // 阻塞取第一个 job（cond_wait）
  while (true) {
    ptr_g = rli->gaq->get_job_group(ev->mts_group_idx);
    error = worker->slave_worker_exec_event(ev);      // ★ 执行（set_gaq_index 记住自己落在 GAQ 哪槽）
    if (error || worker->found_commit_order_deadlock())
      error = worker->retry_transaction(...);         // worker 自己重试整组
    if (ev->ends_group()) break;
    job_item = pop_jobs_item(worker, job_item);       // 取组内下一个事件
  }
  worker->slave_worker_ends_group(ev, 0);             // ★ 收尾：提交位置
}
```

```cpp
bool Slave_worker::commit_positions(Log_event *ev, Slave_job_group *ptr_g, bool force) {
  if (ptr_g->checkpoint_log_name) {                   // 收到新 checkpoint 坐标
    bitmap_copy(&group_shifted, &group_executed);     // ★ bitmap 整体左移
    bitmap_clear_all(&group_executed);
    for (uint pos = ptr_g->shifted; pos < c_rli->checkpoint_group; pos++)
      if (bitmap_is_set(&group_shifted, pos))
        bitmap_set_bit(&group_executed, pos - ptr_g->shifted);
  }
  bitmap_set_bit(&group_executed, ptr_g->checkpoint_seqno);   // ★ 标记本组已完成
  worker_checkpoint_seqno = ptr_g->checkpoint_seqno;
  return flush_info(force);                           // 落 slave_worker_info（crash-safe）
}
void Slave_worker::rollback_positions(Slave_job_group *ptr_g) {
  if (!is_transactional()) {                          // 仅非事务型引擎需回滚 bitmap
    bitmap_clear_bit(&group_executed, ptr_g->checkpoint_seqno);
    flush_info(false);
  }
}
```

`group_executed` 是长度 `checkpoint_group` 的 bitmap，第 `checkpoint_seqno` 位代表"本 worker 做完了第 N 个 checkpoint 槽"；GAQ 推进时**整体左移**（`shifted`），使 worker 表始终只保留最近一个 checkpoint 窗口——这正是第十三章讲的 MTS 恢复要把"worker 局部坐标"平移回全局槽位的根源。`ptr_g->done.store(1)` 是 checkpoint 推进的关键标志。

##### `wait_for_workers_to_finish`：何时必须"清空全场"

```cpp
  while (delegated_jobs > jobs_done && !thd->killed && !is_error) {
    if (mta_checkpoint_routine(rli, true)) return -1;    // ★ 忙轮询，非 cond_wait
  }
  rli->gaq->lwm.sequence_number = SEQ_UNINIT;
  rli->mts_group_status = Relay_log_info::MTS_NOT_IN_GROUP;
```

触发点：`is_new_group == true`；主从 ROTATE / STOP / INCIDENT / FD 等边界事件；`OVER_MAX_DBS_IN_EVENT_MTS`（分区信息过多）；隔离组（LOAD DATA 等）；relay log 读 EOF 切换；`sql_slave_killed`。

**代价**：① 并行度瞬间归零，尾延迟等于最慢 worker 的剩余时间；② 它是 `mta_checkpoint_routine(force=true)` 的**忙轮询**（源码 TODO 明确要改成 wait+signal），每次循环都 `move_queue_head` 并可能 `flush_info`，有 CPU/IO 开销；③ 等待期间 SBM 可能不前进；④ 任一 worker 非 `RUNNING` 立即返回 -1，整个复制线程中止。

| 决策 | 为什么 | 代价 |
|---|---|---|
| 每个 worker 至多一个未完成组（`jobs` 队列空才算空闲） | 队列有界、内存可控、恢复简单 | 源码注释自称 "clearly can't be considered as optimal"；无空闲 worker 时 coordinator 空转 |
| 组内粘性用 `last_assigned_worker`，不用哈希 | 事务不能被拆到两个线程 | 长事务长时间独占 worker，造成倾斜 |
| B/Gtid 先攒 `curr_group_da`，能定 worker 时一次性投递 | BEGIN/GTID 先到但还没算出落点 | coordinator 额外缓冲；统计口径变复杂 |
| 依赖可追踪只等单个事务，不可追踪则全同步 | 并行度最大化 + 算不清就保守串行 | `is_new_group` 任一子条件为真就全局串行 |
| 序号空洞一律按新组 | 依赖不可知，宁可串行也不错序 | 过滤规则会造成频繁全同步 |
| worker 固定长 bitmap + 整体左移 | worker 表大小固定，恢复时求并集 | 每次 checkpoint 移位 O(checkpoint_group)；窗口外信息丢失 |
| MTS 下 coordinator 不做 `slave_trans_retries` | 重试由 worker 自己重读整组 | coordinator 侧临时错误直接终止复制 |

#### DATABASE 模式（库级并行）

与 LOGICAL_CLOCK 并列的另一种调度子模式（`Mts_submode_database`），也是 5.6 的并行方式：

```cpp
Slave_worker *map_db_to_worker(const char *dbname, Relay_log_info *rli, ...);
```

- **分配依据**：按 **database name 哈希**分到 worker（`contains_partition_info()` 时用 `mts_end_group_sets_max_dbs` 映射），同一库的事务恒到同一 worker
- **并行度**：只跨库并行；单库热点完全无法并行——这正是 5.7 引入 LOGICAL_CLOCK 的直接动因
- **退化**：跨库事务（一个事务改多个库）需要额外协调；分区信息过多时 `OVER_MAX_DBS_IN_EVENT_MTS` 会把整组塞给 worker 0 并标记 `curr_group_isolated`，随后 `wait_for_workers_to_finish` 全同步

#### Anonymous_Gtid：不开 GTID 也能并行

`Anonymous_gtid_log_event` 的引入原因见 [`binlog.md`](binlog.md)（gtid_mode=OFF 时不生成 `Gtid_log_event`，MTS 就没有地方放依赖信息）。**对并行复制的意义**：

- 它不带 GTID（匿名），但**携带 `sequence_number` / `last_committed`**，因此 MTS 在不开 GTID 时依然能调度
- 标记 `LOG_EVENT_IGNORABLE_F`，对开 GTID 的从库可忽略
- `schedule_next_event` 里 `GTID_LOG_EVENT` 与 `ANONYMOUS_GTID_LOG_EVENT` **并列**作为"提取两个时间戳"的入口；除此之外的一切事件都拿不到时钟 ⇒ 落进 `is_new_group` 强制串行

#### START REPLICA UNTIL 与 recovery 的交互

`UNTIL_SQL_AFTER_MTS_GAPS`（`Relay_log_info::until_condition`，实现在 `Until_mts_gap`）：

- **语义**：一直复制到"所有 gap 补齐"为止，是 `RESET REPLICA` 前清理 MTS gap 的标准操作
- ★ **它会反向使用 `recovery_parallel_workers`**（`Until_mts_gap::init()`）：把持久化的"上次 MTS 的 worker 数"赋给 `opt_replica_parallel_workers`——因为补 gap 必须用**原来的 worker 数**才能正确重建位图坐标（见 13.1 的开关说明）
- **完成判定**：gap 跑完（`--mts_recovery_group_cnt == 0`）时把 `until_condition` 置为 `UNTIL_DONE`（`exec_relay_log_event` 内）；debug 构建里还会处理 `UNTIL` 在恢复完成时的状态切换

#### 多源复制（channel）与并行复制

- **并行复制是 per-channel 的**：每个 channel 有自己的 `Relay_log_info`（自己的 `gaq` / workers / `opt_replica_parallel_workers`），互不共享 GAQ 与 LWM
- worker 数可按 channel 单独指定（复制服务接口里 `channel_mts_parallel_workers` 决定取全局默认还是 channel 专属值），因此总并发 = **Σ 各 channel 的 workers**，调优时需按通道分别考虑
- 依赖追踪（`Transaction_dependency_tracker`）**是全局单实例**（挂在 `mysql_bin_log` 上），所有 channel 共享同一套时钟；这与从库侧 per-channel 的调度状态形成对比

#### MTS Worker 侧主循环（`handle_slave_worker` / `slave_worker_exec_job_group`）

> 前面讲的是 coordinator 怎么**调度**；本节讲 worker 线程**自己**怎么跑——它的启动、等待取任务、执行一组、回报完成、出错退出。
>
> ⚠️ **名称核对（8.0.39）**：`slave_worker_exec_job` **不存在**（8.0 已合并为单一的 `slave_worker_exec_job_group`）；`signal_worker` **不存在**（投递+唤醒统一在 `append_item_to_jobs` 内）；`current_event_index` **不存在**（8.0 用 `Slave_worker::gaq_index` + `set_gaq_index`/`reset_gaq_index`）；`circle` **不存在**（队列是 `Slave_jobs_queue : public circular_buffer_queue<Slave_job_item>`）；`mts_partition` **不存在**；`apply_event_and_update_pos` 存在但**只属 coordinator**（worker 侧走 `Slave_worker::slave_worker_exec_event`）。

##### `handle_slave_worker`：启动 → 主循环 → 退出清理

```cpp
static void *handle_slave_worker(void *arg) {
  Slave_worker *w = (Slave_worker *)arg;
  Relay_log_info *rli = w->c_rli;          // ★ 这是 coordinator 的 rli，不是自己的
  ...
  thd = new THD;
  mysql_mutex_lock(&w->info_thd_lock);
  w->info_thd = thd;                       // 发布 worker 的 THD
  mysql_mutex_unlock(&w->info_thd_lock);
  if (init_replica_thread(thd, SLAVE_THD_WORKER)) goto err;
  thd->rli_slave = w;                      // ★ worker THD 的 rli_slave 指向 Slave_worker 本身
  ...
  mysql_mutex_lock(&w->jobs_lock);
  w->running_status = Slave_worker::RUNNING;
  mysql_cond_signal(&w->jobs_cond);        // 回报 slave_start_single_worker 的等待
  mysql_mutex_unlock(&w->jobs_lock);
  ...
  while (!error) {
    error = slave_worker_exec_job_group(w, rli);     // ★ 主循环极简
  }
```

- **worker 的 `THD` 与 coordinator 的区别**：coordinator 的 THD 的 `rli_slave` 指向 `Relay_log_info *`；worker 的指向 `Slave_worker *`（因为 `class Slave_worker : public Relay_log_info`，可多态替代）。判据是 `thd->system_thread == SYSTEM_THREAD_SLAVE_WORKER`，大量代码靠 `assert(!is_mts_worker(...))` 把两条路径分开
- ★ **`c_rli` 的作用**：`Slave_worker` 虽是 `Relay_log_info` 子类（自己有 `group_relay_log_pos` 等），但 **GAQ、`pending_jobs`、`mts_*` 计数器、checkpoint 都属于 coordinator 唯一的 rli**。`c_rli` 就是回指 coordinator 的句柄，几乎所有跨线程同步都通过它
- **退出清理**：`cleanup_context` → 把 `jobs` 队列残留 job 全部 de_queue 并累计 `purge_cnt/purge_size` → 在 `pending_jobs_lock` 下把 `rli->pending_jobs -= purge_cnt`、`mts_pending_jobs_size -= purge_size`（**防止残留计数把 coordinator 永久卡在反压里**）→ 置 `NOT_RUNNING` 并 signal（源码注释 *"famous last goodbye"`，`slave_stop_workers` 正在这个 cond 上等）

`Slave_worker::running_status` 五态：

```cpp
enum en_running_state {
  NOT_RUNNING = 0, RUNNING = 1,
  ERROR_LEAVING = 2,   // is set by Worker
  STOP = 3,            // is set by Coordinator upon receiving STOP
  STOP_ACCEPTED = 4    // is set by worker upon completing job when STOP SLAVE is issued
};
```

##### 等待/取任务：`pop_jobs_item` 与双向复用的 `jobs_cond`

★ worker 在 **`&worker->jobs_cond`** 上等（不是 `&rli->data_cond`，后者是 sleep/data 条件，与 job 投递无关）：

```cpp
static struct slave_job_item *pop_jobs_item(Slave_worker *worker, Slave_job_item *job_item) {
  mysql_mutex_lock(&worker->jobs_lock);
  job_item->data = nullptr;
  while (!job_item->data && !thd->killed &&
         (worker->running_status == Slave_worker::RUNNING ||
          worker->running_status == Slave_worker::STOP)) {
    if (set_max_updated_index_on_stop(worker, job_item)) break;
    if (job_item->data == nullptr) {
      worker->wq_empty_waits++;                       // "hungry" 统计
      thd->ENTER_COND(&worker->jobs_cond, &worker->jobs_lock,
                      &stage_replica_waiting_event_from_coordinator, &old_stage);
      mysql_cond_wait(&worker->jobs_cond, &worker->jobs_lock);
      ...
    }
  }
  if (job_item->data) worker->curr_jobs--;
  mysql_mutex_unlock(&worker->jobs_lock);
}
```

- **退出条件**：`thd->killed`，或 `running_status` 变成 `STOP_ACCEPTED` / `ERROR_LEAVING` / `NOT_RUNNING`
- `set_max_updated_index_on_stop()` 是优雅停止的协商点：worker 把自己的 group index 贡献给 `rli->max_updated_index`、`exit_counter++`，并对"越过 max 的组"直接置 `STOP_ACCEPTED` 从而**不执行**这些组就退出。注意返回时 `job_item->data` 可能仍为 `nullptr` —— 这是 `slave_worker_exec_job_group` 里必须再次检查 `running_status` 的原因

★ **`worker->jobs_cond` 是双向复用的**：coordinator 在 `append_item_to_jobs` 里队列满时等"有空位"，worker 在这里等"有活儿"，都由 `jobs_lock` 保护。队列上限常量 `mts_slave_worker_queue_len_max = 16384`。coordinator 侧入队即 `if (worker->jobs.get_length() == 1) mysql_cond_signal(&worker->jobs_cond)`（只在从空变非空时唤醒）。

##### `slave_worker_exec_job_group` 执行骨架

```cpp
int slave_worker_exec_job_group(Slave_worker *worker, Relay_log_info *rli) {
  struct slave_job_item item = {nullptr, 0, {'\0'}};
  bool seen_gtid = false, seen_begin = false;
  if (unlikely(worker->trans_retries > 0)) worker->trans_retries = 0;

  job_item = pop_jobs_item(worker, job_item);        // ← 阻塞点 1（可能返回空 data）
  ...
  while (true) {
    if (unlikely(thd->killed || worker->running_status == Slave_worker::STOP_ACCEPTED)) {
      error = -1; goto err;
    }
    ev = job_item->data;
    ev->claim_memory_ownership(true);                // 认领 event 内存所有权
    if (is_gtid_event(ev)) seen_gtid = true;
    if (!seen_begin && ev->starts_group()) { seen_begin = true; ... }
    ptr_g = rli->gaq->get_job_group(ev->mts_group_idx);
    if (ptr_g->new_fd_event) { worker->set_rli_description_event(ptr_g->new_fd_event); ... }
    worker->set_group_source_log_start_end_pos(ev);
    error = worker->slave_worker_exec_event(ev);
    if (error || worker->found_commit_order_deadlock()) {
      worker->prepare_for_retry(*ev);
      error = worker->retry_transaction(...);
      if (error) goto err;
    }
    if (ev->ends_group() || (!seen_begin && !is_gtid_event(ev) && (...))) break;
    remove_item_from_jobs(job_item, worker, rli);
    if (ev->worker != nullptr) delete ev;
    job_item = pop_jobs_item(worker, job_item);      // ← 阻塞点 2（组内下一个事件）
  }
  worker->slave_worker_ends_group(ev, 0);
  remove_item_from_jobs(job_item, worker, rli);
  delete ev;
  return 0;
err:
  if (error) {
    Commit_stage_manager::get_instance().finish_session_ticket(thd);
    report_error_to_coordinator(worker);
    worker->slave_worker_ends_group(ev, error);
  }
  return error;
}
```

- ★ **一次调用 = 一个事务组**：内层 `while(true)` 一直 pop 到 `ev->ends_group()`（XID / DDL Query / commit）才 break ⇒ 一个 worker 同一时刻只推进一个组，天然串行
- **worker 侧事件应用入口是 `Slave_worker::slave_worker_exec_event`**（不是 `apply_event_and_update_pos`）：

```cpp
  set_future_event_relay_log_pos(ev->future_event_relay_log_pos);
  set_master_log_pos(static_cast<ulong>(ev->common_header->log_pos));
  set_gaq_index(ev->mts_group_idx);          // ★ 就是"当前 group index"
  ret = ev->do_apply_event_worker(this);
```

`set_gaq_index` 只在 `gaq_index == capacity`（哨兵值"未设置"）时才写入，`reset_gaq_index()` 复位为 `capacity`。

- **回报完成**（`slave_worker_ends_group` 核心）：

```cpp
    ptr_g = c_rli->gaq->get_job_group(gaq_index);
    Commit_order_manager::wait_and_finish(info_thd, false);   // preserve_commit_order
    if (!m_flag_positions_committed && !is_committed_ddl(ev))
      commit_positions(ev, ptr_g, true);                      // 写 slave_worker_info
    ptr_g->group_master_log_pos = group_master_log_pos;
    ptr_g->group_relay_log_pos  = group_relay_log_pos;
    ptr_g->done.store(1);                                     // ★ 置 GAQ done 位
    last_group_done_index = gaq_index;
    reset_gaq_index();
    groups_done++;
```

`done` 是 `std::atomic<int32>`，coordinator 的 `move_queue_head()` 靠它判定 gap 并推进 lwm。

- **错误传播**：先置 `running_status = ERROR_LEAVING`（防 C 再与它同步），再 `Commit_order_manager::wait_and_finish(info_thd, true)` 让后继事务回滚，最后 `c_rli->info_thd->awake(THD::KILL_QUERY)` **主动杀死 coordinator**

##### 依赖等待不在 worker 侧

★ `Mts_submode_logical_clock::wait_for_last_committed_trx()` 由 **coordinator** 在 `schedule_next_event()` 中调用，阻塞在 `&rli->logical_clock_cond`（持 `mts_gaq_LOCK`）。worker 侧只有一条 debug 断言：

```cpp
    assert(rli->gaq->entry == ev->mts_group_idx ||
           Mts_submode_logical_clock::clock_leq(last_committed, lwm_estimate));
```

含义：**"调度前必须已满足依赖"是 coordinator 的不变式**，worker 只负责执行。worker 完成后反过来帮忙唤醒等待中的 C：

```cpp
    if (min_child_waited_logical_ts != SEQ_UNINIT) {          // 有 C 正在等
      mysql_mutex_lock(&c_rli->mts_gaq_LOCK);
      if (mts_submode->min_waited_timestamp != SEQ_UNINIT) {
        longlong curr_lwm = mts_submode->get_lwm_timestamp(c_rli, true);
        if (mts_submode->clock_leq(mts_submode->min_waited_timestamp, curr_lwm))
          mysql_cond_signal(&c_rli->logical_clock_cond);      // ← 唤醒 C
      }
      mysql_mutex_unlock(&c_rli->mts_gaq_LOCK);
    }
```

##### 反压三层（全部在 `remove_item_from_jobs` 里解除）

```cpp
  mysql_mutex_lock(&worker->jobs_lock);
  worker->jobs.de_queue(job_item);
  if (worker->jobs.get_length() == worker->jobs.capacity - 1 && worker->jobs.overfill) {
    worker->jobs.overfill = false;
    mysql_cond_signal(&worker->jobs_cond);        // 解除 C 的 en_queue 等待
  }
  mysql_mutex_unlock(&worker->jobs_lock);

  mysql_mutex_lock(&rli->pending_jobs_lock);
  rli->pending_jobs--;
  rli->mts_pending_jobs_size -= ev->common_header->data_written;
  ...
  if (rli->mts_pending_jobs_size < rli->mts_pending_jobs_size_max && rli->mts_wq_oversize) {
    rli->mts_wq_oversize = false;
    mysql_cond_signal(&rli->pending_jobs_cond);   // ★ 解除 C 的 size 反压
  }
  mysql_mutex_unlock(&rli->pending_jobs_lock);
  worker->events_done++;
```

| 层 | 判据 | C 在哪等 | 谁解除 |
|---|---|---|---|
| 全局内存 | `mts_pending_jobs_size + ev_size > mts_pending_jobs_size_max` | `&rli->pending_jobs_cond` | worker 消费后 signal |
| 单队列满 | `jobs.en_queue() == error_result` | `&worker->jobs_cond`（置 `jobs.overfill=true`） | 降到 `capacity-1` 时 signal |
| 软性减速 | `jobs.len > underrun_level` 且无 underrun worker | 不等待 | C `my_sleep(min(1000, (mts_wq_excess_cnt+1) * mts_coordinator_basic_nap))` 自适应退避（`=5` 微秒） |

`mts_pending_jobs_size_max` 来自 `replica_pending_jobs_size_max`（`slave_pending_jobs_size_max` 是废弃别名）。★ `rli->mts_group_status`（`MTS_NOT_IN_GROUP`/`IN_GROUP`/`END_GROUP`/`KILLED_GROUP`）是 **coordinator 的组状态机**，worker 本身不改它。

##### 临时错误与重试

`if (error || worker->found_commit_order_deadlock())` → `prepare_for_retry()` → `retry_transaction()`：

```cpp
  if (slave_trans_retries == 0) return true;
  do {
    std::tie(ret, silent, error) = check_and_report_end_of_retries(thd);
    if (ret) return true;                       // 非临时错误 / 不可安全回滚 / 超过 slave_trans_retries
    if (!silent) { trans_retries++; ... }
    mysql_mutex_lock(&c_rli->data_lock);
    c_rli->retried_trans++;                     // 全局重试计数
    mysql_mutex_unlock(&c_rli->data_lock);
    clean_retry_context();
    worker_sleep(min<ulong>(trans_retries, MAX_SLAVE_RETRY_PAUSE));  // 退避
  } while (read_and_apply_events(start_relay_pos, ..., end_relay_pos, ...));
```

★ `worker_sleep` 也是等 `&jobs_cond`（`mysql_cond_timedwait`），所以 **STOP 能立刻打断退避**。

##### worker 的位点与持久化

worker 持有的（继承自 `Relay_log_info` + 自身扩展）是**固定长度字符数组形式**的 checkpoint 对（不是结构体，也没有 `circle` 成员）：

```cpp
  char      checkpoint_relay_log_name[FN_REFLEN];  ulonglong checkpoint_relay_log_pos;
  char      checkpoint_master_log_name[FN_REFLEN]; ulonglong checkpoint_master_log_pos;
```

真正持久化的入口是 `Slave_worker::commit_positions()`：吸收 `ptr_g` 里的 `checkpoint_log_name/pos` → 按 `ptr_g->shifted` 右移 `group_executed` 位图 → 置本组 seqno 位 → `flush_info()` 写 `mysql.slave_worker_info`。

★ **两表分工**：worker 只写自己的 `slave_worker_info`，coordinator 按 `done` 位算 lwm 写 `slave_relay_log_info`——写路径无跨线程争用，但两表可能不一致，恢复依赖 `group_executed` 位图（见「MTS Crash Recovery」章）。

##### worker 数量变更（重要更正）

★ `replica_parallel_workers` 在 8.0.39 是 **`PERSIST_AS_READONLY`**——**不支持在线调整**；`ON_UPDATE` 只发一条"值 0 已废弃"的 warning。增删 worker 只发生在 `START REPLICA` 时：

- `slave_start_workers()` → `rli->init_workers(max(n, rli->recovery_parallel_workers))` → 循环 `slave_start_single_worker()`（创建 + `mysql_thread_create(..., handle_slave_worker, w)`），主线程在 `&w->jobs_cond` 上等到 `running_status != NOT_RUNNING`
- `slave_stop_workers()`：置 `STOP` 并 signal → 等到 `NOT_RUNNING` → 强制 `mta_checkpoint_routine(rli, false)` → `deinit_workers()` → `delete rli->gaq`

**"wait for jobs to finish" 的必要性**：GAQ 是按 n 构造的，`workers` 数组与每个 worker 的 `jobs` 环形队列在 resize 时会被整体销毁，因此必须先把已投递 job 跑完（或按 `max_updated_index` 丢弃）再重建。

##### coordinator ↔ worker 时序图

```
 Coordinator (handle_slave_sql)                     Worker N (handle_slave_worker)
   |                                                       |
   | exec_relay_log_event()                               |
   |   -> get_slave_worker() 选 W                          |
   |   -> schedule_next_event()                           |
   |        wait_for_last_committed_trx()  ★依赖等待在 C 侧 |
   |        wait rli->logical_clock_cond                   |
   |   -> append_item_to_jobs(job_item, w, rli)            |
   |        (a) size 超限 -> mts_wq_oversize=true          |
   |            wait rli->pending_jobs_cond  <-------------+-- remove_item_from_jobs()
   |        (b) 队列满   -> jobs.overfill=true             |     signal pending_jobs_cond
   |            wait w->jobs_cond           <-------------+-- signal jobs_cond
   |        (c) en_queue + curr_jobs++                     |
   |            if(len==1) signal w->jobs_cond ----------> | pop_jobs_item() 醒来
   |                                                       | slave_worker_exec_job_group()
   |                                                       |   loop { pop; slave_worker_exec_event }
   |                                                       |     set_gaq_index(ev->mts_group_idx)
   |                                                       |     do_apply_event_worker()
   |                                                       |       -> commit_positions()->flush_info()
   |                                                       |          写 mysql.slave_worker_info
   |                                                       |   remove_item_from_jobs() (每 event)
   |                                                       |   slave_worker_ends_group(ev, 0)
   |                                                       |     ptr_g->done.store(1)           ★回报
   |                                                       |     if(有 C 等 lwm) signal logical_clock_cond
   | mta_checkpoint_routine() -> move_queue_head()         |
   |   扫 done 位推进 lwm -> 写 slave_relay_log_info       |
```

| 决策 | 实现 | 收益 | 代价 / 风险 |
|---|---|---|---|
| 依赖等待放哪侧 | 放 coordinator | worker 拿到 job 即可执行、调度唯一化 | 一个依赖未满足的事务**阻塞整个 C 的读-调度流水线** |
| worker 主循环粒度 | 一次调用 = 一个事务组 | 组内天然串行 | 长事务独占 worker；组内每 event 都要二次加锁 |
| cond 复用 | `worker->jobs_cond`（C 等空位 + W 等活儿） | 少一把锁/cond | 双方都用单发 signal，依赖 `while` 重查；状态判断散在两处 |
| 完成回报 | GAQ 条目上的 `atomic<int32> done` | C 无需遍历 worker 即可算 lwm | lwm 推进是**被动/周期**的，故障时位点回退粒度粗 |
| 位点持久化 | worker 写 `slave_worker_info`，C 算 lwm 写 `slave_relay_log_info` | 写路径无争用 | 两表可能不一致，恢复依赖位图 |
| 反压 | 硬限（条件等待）+ 软限（`my_sleep` 退避） | 避免 OOM 又降锁竞争 | 退避是忙等式短睡眠，超大数据量下 C 吞吐受限 |
| 临时错误 | worker 自持 `retry_transaction`，不改 `mts_group_status` | 重试不影响调度状态机 | 重试期间 worker 独占，可能拉大 GAQ gap |
| worker 数量 | 静态（READONLY），仅 START/STOP 时重建 | 无 resize 期一致性问题 | 无法弹性伸缩；扩容必须停 applier |
| 错误传播 | worker 置 `ERROR_LEAVING` 并杀 coordinator | 快速终止，避免 C 空等 | 只保留**最后**一个错误 |

### 完整时序图

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



### WL#7165 经典图解分析

WL#7165 是 MySQL 5.7 引入 LOGICAL_CLOCK 并行复制的工作日志（WorkLog）。

#### 图解

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

#### 逐个事务分析

| 事务 | P 时 m_max 快照 | C 后 m_max | lc | 依赖谁 | 原因 |
|---|---|---|---|---|---|
| Trx1 | 0 | 1 | **0** | 无 | 第一条事务 |
| Trx2 | 0（Trx1 未 C） | 2 | **0** | 无 | P(2)时 Trx1 还没 C，锁间隔重叠 → 不冲突 |
| Trx3 | 1（Trx1 已 C，Trx2 未 C） | 3 | **1** | Trx1 | P(3)时只看到 m_max=1 |
| Trx4 | 2（Trx1、Trx2 已 C） | 4 | **2** | Trx2 | P(4)时 Trx1/Trx2 都已 C |
| Trx5 | 2（Trx3 未 C） | 5 | **2** | Trx2 | P(5)时只看到 m_max=2（Trx3 未 C，重叠！） |
| Trx6 | 3（Trx3 已 C） | 6 | **3** | Trx3 | P(6)时 Trx3 已 C |
| Trx7 | 4（Trx4 已 C） | 7 | **4** | Trx4 | P(7)时 Trx4 已 C |

#### 关键观察

**1. Trx1 和 Trx2 可以并行**：lc[2]=0，lc[1]=0，都不依赖对方。

**2. Trx3 必须等 Trx1**：lc[3]=1，要等 seq=1 提交。但这可能是假阳性——Trx3 和 Trx1 可能修改的是不同行。

**3. Trx5 的 lc=2 而非 3**：这是 LOGICAL_CLOCK 设计的精妙之处。虽然 Trx3 的 seq=3 > Trx5 的 lc=2，但 Trx5 获取锁时 Trx3 还没 commit（m_max 还是 2），说明它们的锁间隔重叠 → 不冲突。Trx5 不需要等 Trx3。

**4. 波浪形推进**：`last_committed` 的值不是单调递增的，而是像波浪一样随着锁间隔的交错而上下波动——这正是 "LOGICAL_CLOCK"（逻辑时钟）名称的由来。

#### 从库执行顺序

```
Worker-1: Trx1(lc=0) → Trx3(lc=1,等Trx1) → Trx6(lc=3,等Trx3)
Worker-2: Trx2(lc=0) → Trx4(lc=2,等Trx2) → Trx5(lc=2,等Trx2) → Trx7(lc=4,等Trx4)
```

Trx1和Trx2 并行，Trx4和Trx5 都对 lc=2 满足条件后也可并行（在 Trx2 提交后同时分发）。

---



### Writeset Hash 生成机制

#### 核心函数：add_pke

Writeset 的核心是 **PKE（Primary Key Equivalent）**——主键等价值。定义在 [rpl_write_set_handler.cc](sql/rpl_write_set_handler.cc)。

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

**步骤2：Hash 计算**（[rpl_write_set_handler.cc](sql/rpl_write_set_handler.cc)）

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

**步骤3：存入 writeset vector**（[rpl_transaction_write_set_ctx.cc](sql/rpl_transaction_write_set_ctx.cc)）

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

#### 为什么 PKE 能找到更多可并行事务？

```
COMMIT_ORDER 判定：Trx3 的 lc=1，依赖 Trx1

WRITESET 再检查：
  Trx1 的 writeset = {hash("PRIMARY½db1½3t1½21½1"), hash("j½db1½3t1½22½1")}
  Trx3 的 writeset = {hash("PRIMARY½db2½3s2½299½2"), hash("k½db2½3s2½299½2")}
  
  → writeset 交集 = ∅ → last_parent = 0
  → commit_parent = min(0, 1) = 0 → Trx3 不再依赖 Trx1 → 恢复并行！ ✓
```

**关键设计**：`commit_parent = std::min(last_parent, commit_parent)`。Writeset 只会让依赖变小（发现更多并行机会），永远不会变大（不会引入假阴性）。

#### 唯一键也被纳入的原因

PKE 包含**所有**唯一索引（不仅是主键）。这是因为唯一键也能唯一标识一行——如果两个事务修改同一行的不同唯一索引，hash 对也会产生碰撞，从而被正确识别为冲突。

---



##### WRITESET 如何压低 commit_parent（完整代码）

```cpp
void Writeset_trx_dependency_tracker::get_dependency(THD *thd,
                                                     int64 &sequence_number,
                                                     int64 &commit_parent) {
  Rpl_transaction_write_set_ctx *write_set_ctx =
      thd->get_transaction()->get_transaction_write_set_ctx();
  std::vector<uint64> *writeset = write_set_ctx->get_write_set();

  bool can_use_writesets =                                   // ① 四项准入
      (writeset->size() != 0 || write_set_ctx->get_has_missing_keys() ||
       is_empty_transaction_in_binlog_cache(thd)) &&
      (global_system_variables.transaction_write_set_extraction ==
       thd->variables.transaction_write_set_extraction) &&
      !write_set_ctx->get_has_related_foreign_keys() &&
      !write_set_ctx->was_write_set_limit_reached();
  bool exceeds_capacity = false;

  if (can_use_writesets) {
    exceeds_capacity =
        m_writeset_history.size() + writeset->size() > m_opt_max_history_size;  // ② 预测式判超

    int64 last_parent = m_writeset_history_start;            // ③ 起点 = 历史最老水位
    for (auto it = writeset->begin(); it != writeset->end(); ++it) {
      auto hst = m_writeset_history.find(*it);
      if (hst != m_writeset_history.end()) {
        if (hst->second > last_parent && hst->second < sequence_number)
          last_parent = hst->second;                         // ④ 取"最晚的冲突者"
        hst->second = sequence_number;                       // ⑤ 覆盖写回：我成为最新修改者
      } else {
        if (!exceeds_capacity)
          m_writeset_history.insert(std::pair<uint64, int64>(*it, sequence_number));
      }
    }
    if (!write_set_ctx->get_has_missing_keys())
      commit_parent = std::min(last_parent, commit_parent);   // ⑥ 只下压，且无 missing keys 才行
  }
  if (exceeds_capacity || !can_use_writesets) {
    m_writeset_history_start = sequence_number;               // ⑦ 全清 + 抬高水位
    m_writeset_history.clear();
  }
}
```

- **③ `last_parent` 初值是 `m_writeset_history_start`**：保证即便没命中任何历史行，本事务也不会与"被清空之前的事务"并行——这是正确性的关键下限
- **④ 取 max 而非 min**：必须等**所有**冲突事务，最晚的那个才是瓶颈
- **⑤ 无论命中与否都覆盖写回**：只关心"最后修改者"
- **⑥ 只在没有 missing keys 时下压**：表缺主键/唯一键 ⇒ writeset 不完整 ⇒ 保持 COMMIT_ORDER 的值；且此时**不清 history**（引用该表的事务都会降级，清了无用）
- **⑦ 淘汰 = 全清 + 抬水位**，不是 LRU。注意 `exceeds_capacity` 时循环已跑完 ⇒ 本事务仍享受一次 writeset 并行度

##### 行级 PKE 生成：add_pke

调用点在 `binlog_log_row`，对 `after_record` 与 `before_record` **各调一次**（UPDATE 因此产生新旧两套 PKE）：

```cpp
std::array<const uchar *, 2> records{after_record, before_record};
for (auto rec : records) {
  if (rec != nullptr) { if (add_pke(table, thd, rec)) return HA_ERR_RBR_LOGGING_FAILED; }
}
```

`add_pke` 遍历**所有**索引、只跳过非唯一索引（`!(flags & HA_NOSAME)`），所以每个唯一索引各贡献一个 PKE。源码注释的例子：

> `CREATE TABLE db1.t1 (i INT NOT NULL PRIMARY KEY, j INT UNIQUE KEY, k INT UNIQUE KEY);`
> `INSERT INTO db1.t1 VALUES(1, 2, 3);` 产生 **3 个** PKE 串：`PRIMARY½db1½3t1½11½1`、`j½db1½3t1½21½1`、`k½db1½3t1½31½1`

**为什么是一行多 PKE**：两个事务可能在**不同行**上撞**同一个唯一键**（或删 A 行、插 B 行复用同一 unique 值），只在主键维度算依赖会漏掉这类冲突。每个 PKE 串后追加"长度"并用 `½` 分隔，是为了消除拼接歧义（`'ab'+'c'` vs `'a'+'bc'`）；`make_sort_key` 生成**可 memcmp 的归一化排序键**，保证语义相同的值得到同一 hash。

含 NULL 的键直接放弃该索引（`field->is_null(ptrdiff)` 时 break）；多值键（数组）由 `generate_mv_hash_pke` 逐元素生成。

> ★ **纠正一个常见说法**：8.0.39 **没有"三个 hash 表"**。实际只有 **1 个容器**：
> `typedef std::map<uint64, int64> Writeset_history; Writeset_history m_writeset_history;`
> ——注意是 **`std::map`（红黑树，有序）而非 `unordered_map`**，主键/唯一键/外键三类 PKE 的 hash **全部混在同一张表**里。"三个 hash"若指三类 PKE 串（PK/UK、多值元素、FK 引用）则属实，但它们只是 PKE 的分类，不是分类存储。hash 算法三选一：`MURMUR32` / `XXHASH64`（默认）/ `OFF`。

##### 降级逻辑（何时退回 COMMIT_ORDER）

| 触发条件 | 效果 |
|---|---|
| writeset 为空（DDL / 非行事件） | 降级 + 清空 history（空事务除外，可安全并行） |
| **表无主键/无唯一键**（`has_missing_keys`） | 不压低 commit_parent，**但不**清 history |
| **外键约束**（表被别的表 FK 引用） | 降级 + 清空 history（级联写在引擎内部，binlog 行事件不可见，无法追踪） |
| session 与 global hash 算法不一致 | 降级 + 清空（历史 hash 与本次不可比） |
| 事务 writeset 超过 `binlog_transaction_dependency_history_size` | 本事务 writeset 被丢弃，永不可用 |
| 历史超容（预测式） | 先用完历史算完本事务，再清 |
| `transaction_write_set_extraction=OFF` | 根本不允许设成 WRITESET |

**容量与淘汰**：上限即 `binlog_transaction_dependency_history_size`（默认 **25000**，范围 1~1000000），直接挂到 `m_opt_max_history_size`。判定是**预测式**（"加入本事务后是否会超"）。淘汰 = 全清 + 把 `m_writeset_history_start` 抬到当前 sn，等价于"新历史的第一条记录视为被一个虚拟事务持有"，保证清空后不会错误地与清空前的事务并行。**吞吐含义**：history 越大，可并行窗口越长；越小则频繁全清，并行度退化到接近 COMMIT_ORDER。

##### WRITESET_SESSION 只做一件事

```cpp
  int64 session_parent = thd->rpl_thd_ctx.dependency_tracker_ctx()
                             .get_last_session_sequence_number();
  if (session_parent != 0 && session_parent < sequence_number)
    commit_parent = std::max(commit_parent, session_parent);      // 只抬不下压
  thd->rpl_thd_ctx.dependency_tracker_ctx().set_last_session_sequence_number(
      sequence_number);
```

状态在 THD 上（`Dependency_tracker_ctx::m_last_session_sequence_number`）。**为什么需要**：纯 WRITESET 下同 session 的两个不冲突事务会被判可并行，但备库上同 session 有顺序语义（临时表、`@@session` 变量、用户变量、`GET_LOCK`），并行重放会破坏这些隐式依赖。代价是并行度低于 WRITESET。

| 决策 | 为什么 | 代价 |
|---|---|---|
| 行级 PKE hash（非表级/页级） | 表级太粗；行级 64-bit 碰撞只降并行度不丢正确性（安全方向） | hash 碰撞 = 假冲突 |
| `last_parent = max(命中行旧 sn)` | 必须等最晚的冲突者 | 取 min 会破坏正确性 |
| 无 PK 表不 hash 全列，整事务降级 | 全列 hash 对长行开销大 | 该表相关事务全部退化 |
| history 全清而非 LRU | 实现简单、无每条目维护成本 | 周期性并行度"断崖" |
| 容器用 `std::map` 而非 `unordered_map` | 有序稳定、无 rehash 抖动 | O(log n)（25000 项下差异可忽略） |
| binlog rotate 时 `m_writeset.rotate(1)` | 新文件内时间戳重新计数，必须允许重新并行 | 水位设为 1 |

### GAQ 与 Checkpoint 机制

#### GAQ 数据结构

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

#### find_lwm：查找低水位线

```cpp
// rpl_rli_pdb.cc
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

#### move_queue_head：Checkpoint 核心

```cpp
// rpl_rli_pdb.cc
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

#### mta_checkpoint_routine：定时 Checkpoint

（[rpl_replica.cc](sql/rpl_replica.cc)）

```
定时触发（replica_checkpoint_period，默认 300ms）
  → move_queue_head()     // 从 GAQ 头部弹掉已完成 group，更新 LWM
  → 更新 group_master_log_pos / group_relay_log_pos 为 LWM 的值
  → flush_info()          // 持久化到 mysql.slave_relay_log_info 表
  → 广播 data_cond        // 唤醒等待 checkpoint 完成的线程
```

#### Checkpoint 对并行度的影响

1. **释放 GAQ 空间**：`move_queue_head` 弹出已完成的 group，腾出位置给新事务。如果 GAQ 满了，Coordinator 必须等待 checkpoint 推进。
2. **推进 LWM**：LWM 推进后，之前因 `last_committed > old_lwm` 而阻塞在 `wait_for_last_committed_trx()` 的事务可以继续调度。
3. **崩溃恢复**：从库重启时，从 `mysql.slave_relay_log_info` 表读取的 `group_relay_log_pos` **就是 LWM**（它 ≡ GAQ 的 lwm，见第十三章 13.1），恢复从此处开始。但**"哪些组该跳过"不是按位置简单判断**——而是靠第十三章讲的 recovery 位图决定；且 GTID ON + AUTO_POSITION 时整条位图机制被旁路（13.4）。

---



##### GAQ 数据结构：定长环形缓冲

```cpp
template <typename Element_type>
class circular_buffer_queue {
  Prealloced_array<Element_type, 1> m_Q;
  size_t capacity;      // 容量 = replica_checkpoint_group
  size_t avail;         // 入队游标，单调递增到 2*capacity
  size_t entry;         // 队头真实下标 [0, capacity)
  std::atomic<size_t> len;
  bool in(size_t i) {   // 合法性判定
    return (avail >= capacity) ? (entry <= i || i < avail - capacity)
                               : (entry <= i && i < avail);
  }
};
class Slave_committed_queue : public circular_buffer_queue<Slave_job_group> {
  Slave_job_group lwm;                        // ★ 最后一次被弹出的组的快照
  Prealloced_array<ulonglong, 1> last_done;   // per-worker 进度
  ulong assigned_group_index;
  size_t move_queue_head(Slave_worker_array *ws);
  size_t find_lwm(Slave_job_group **, size_t);
};
```

容量 = `replica_checkpoint_group`（默认 512，上限 `MTS_MAX_BITS_IN_GROUP` = 524280），在 `slave_start_workers()` 里 `new Slave_committed_queue(rli->checkpoint_group, n)`。

```
         entry                                avail%capacity
           |                                        |
   +---+---+---+---+---+---+---+---+---+---+---+---+
   | d | d | . | d | d | d | ? | ? |   |   |   |   |   capacity = 12
   +---+---+---+---+---+---+---+---+---+---+---+---+
     ^       ^               ^
     |       |               +-- 下一个入队位置
     |       +-- gap: done==0  → move_queue_head 在此 break
     +-- 已弹出区间
```

##### move_queue_head：真正推进水位

```cpp
size_t Slave_committed_queue::move_queue_head(Slave_worker_array *ws) {
  size_t cnt = 0;
  while (!empty()) {
    ptr_g = &m_Q[entry];
    if (ptr_g->worker_id == MTS_WORKER_UNDEF || ptr_g->done.load() == 0)
      break;                                    // ① 队头是 gap → 停
    w_i = ws->at(ptr_g->worker_id);
    if (ptr_g->group_relay_log_name) {          // ② 先把动态字符串挪到栈上再释放
      strcpy(grl_name, ptr_g->group_relay_log_name);
      my_free(ptr_g->group_relay_log_name);
      ptr_g->group_relay_log_name = nullptr;
    }
    Slave_job_group g = Slave_job_group();
    (void)de_queue(&g);                         // ③ 弹出
    if (grl_name[0] != 0) strcpy(lwm.group_relay_log_name, grl_name);
    g.group_relay_log_name = lwm.group_relay_log_name;
    lwm = g;                                    // ④ ★ lwm = 最后一次弹出的组
    last_done[w_i->id] = ptr_g->total_seqno;    // ⑤ per-worker 进度
    cnt++;
  }
  return cnt;                                   // ⑥ 本次弹出几个组
}
```

- ① 弹出条件：`worker_id` 未定义（还没分派）**或** `done == 0`（已分派未提交）。`done` 由 worker 在 `slave_worker_ends_group()` 里 `store(1)`
- ★ **纠正一处过时注释**：头文件注释说"进度通过与 worker 的 `last_group_done_index` 比较来评估"——**8.0.39 实际只看 `done` 标志**，`last_group_done_index` 仅用于 worker 自身记账
- ④ `lwm` 永远是"最后一个被弹出的组"的快照；`cnt == 0` 意味着队头卡住、checkpoint 无法推进

##### find_lwm：只读探测（与 move_queue_head 分工）

```cpp
size_t Slave_committed_queue::find_lwm(Slave_job_group **arg_g, size_t start_index) {
  size_t i;
  for (i = start_index; i < avail; i++) {       // ① 从 start_index 向后扫
    ptr_g = &m_Q[i % capacity];
    if (ptr_g->done.load() == 0) break;         // ② 遇未完成即停
  }
  if (i == start_index) return capacity;        // ③ 起点就没完成 → 返回哨兵
  ptr_g = &m_Q[(i - 1) % capacity];
  *arg_g = ptr_g;
  return (i - 1) % capacity;
}
```

**分工**：`move_queue_head` 是**破坏性**的（真的出队、更新 lwm、释放内存），服务 checkpoint 的持久化语义；`find_lwm` 是**只读探测**（不修改游标），服务调度器的依赖判断。`start_index` 让调度器可以**增量续扫**（缓存 `last_lwm_index`，只在"缓存陈旧"时才从队头重扫）。

##### mta_checkpoint_routine：双轨触发

```cpp
  /* exec_relay_log_event 里的调用点 */
  bool force = rli->rli_checkpoint_seqno >= rli->checkpoint_group;   // GAQ 将满
  if (force || rli->is_time_for_mta_checkpoint())                    // 300ms 机会型
    mta_checkpoint_routine(rli, force);
```

```cpp
  do {
    cnt = rli->gaq->move_queue_head(&rli->workers);
  } while (!sql_slave_killed(...) && cnt == 0 && force &&            // ★ force 才忙轮询
           (my_sleep(rli->mts_coordinator_basic_nap), 1));
  if (cnt == 0) goto end;                                            // period 模式只试一次
  ...
  rli->set_group_master_log_pos(rli->gaq->lwm.group_master_log_pos);
  rli->set_group_relay_log_pos(rli->gaq->lwm.group_relay_log_pos);   // ★ coordinator 坐标 = lwm
  error = rli->flush_info(Relay_log_info::RLI_FLUSH_IGNORE_SYNC_OPT); // ★ 仅 cnt!=0 才落盘
  mysql_cond_broadcast(&rli->data_cond);                             // 唤醒 STOP/UNTIL 等待者
  rli->reset_notified_checkpoint(cnt, ts, true);                     // 释放 GAQ 配额 + bitmap_shifted
```

- **`force`**（GAQ 将满）：`do..while` + `my_sleep(mts_coordinator_basic_nap)` **忙轮询**直到弹出至少一个组
- **period**（`replica_checkpoint_period` 默认 300ms）：只试一次，`cnt == 0` 直接返回，绝不阻塞
- **落盘**：仅 `cnt != 0` 时 `flush_info`，且忽略 `sync_relay_log_info`。这是 MTS 下 coordinator 持久化自身坐标的**唯一时机**（worker 各自在 `commit_positions` 里 flush 自己的 bitmap）
- **`reset_notified_checkpoint`**：`rli_checkpoint_seqno -= cnt`（释放配额）、`w->bitmap_shifted += cnt`、`checkpoint_notified = false`（下次分派重新下发 CP 坐标）

##### checkpoint 与并行度

- **GAQ "满"不会真发生**：`force` 在填满前强制 CP，把"满"转化成 coordinator 在 `mta_checkpoint_routine` 里忙等（而非条件变量）
- **真正的并行度杀手是长事务**：只要队头那个 gap 迟迟不 `done`，`move_queue_head` 就返回 0，GAQ 积压到 `checkpoint_group` 后 coordinator 停止分派新组 ⇒ **并行度被打回 1**
- `replica_checkpoint_period` 只调节"多久试一次"，**不决定 GAQ 容量**；调小 → CP/SBM 更实时但 IO 上升；调大 → 崩溃后重放更多
- 另有**独立的一级背压**：worker 的 `Slave_jobs_queue`（`replica_pending_jobs_size_max`），与 GAQ 容量无关

##### 与 MTS 崩溃恢复的衔接（接第十三章）

恢复位图 = **rli 里 lwm 持久化的坐标** + **worker 表的 `group_executed` bitmap** 合成：

- **lwm 侧**：`mta_checkpoint_routine` 把 `lwm` 坐标写入 rli 并 `flush_info` ⇒ 这就是恢复的**起点** `cp`，保证"lwm 之前全体已提交"，只会落后不会超前
- **worker 侧**：worker 表里存的是"自我上次收到 CP 坐标以来执行了哪些组"的位图（`commit_positions` 里置位 + 按 `shifted` 整体左移）
- **合成**：只保留坐标 `> cp` 的 worker（`above_lwm_jobs`），从 lwm 的 relay 位置顺序读事件数组直到匹配该 worker 的 `w_last`，得到 `recovery_group_cnt`，再把 worker bitmap 尾段映射进全局 `recovery_groups`
- **GTID ON + auto_position 时整体短路**（`mts_recovery_group_cnt = 0`），因为 GTID 自动跳过已执行事务——这正是 13.4 讲的旁路

| 决策 | 收益 | 代价 |
|---|---|---|
| 定长环形缓冲，容量 = checkpoint_group | O(1) 入队出队、内存上界可预测 | 一个长事务就能堵死整个窗口 |
| `avail` 单调 + `entry` 归一化 | `len` 可无锁原子读 | 所有访问都要 `% capacity`，越界难查 |
| 每组 `atomic done` 标志 | 无需遍历 worker 判 gap | 遗留注释误导（已纠正） |
| `find_lwm`(只读) 与 `move_queue_head`(破坏性) 分离 | 调度可高频查 lwm 不干扰 CP | 两个"lwm"语义不同，需 `is_stale` 兜底 |
| 双轨触发 force/period | GAQ 永不满 + 空闲零阻塞 | force 路径是忙轮询，长尾事务时 CPU 空转 |
| 仅 `cnt != 0` 落盘 | 无进展不写盘 | 崩溃时坐标必然落后，靠 worker bitmap 补偿 |
| GTID auto_position 短路恢复 | 恢复路径大幅简化 | 非 GTID 部署仍走全量 bitmap 重建 |

### MTS Crash Recovery 与静默丢数据缺陷

> 本章基于 8.0.39 源码逐段剖析 MTS crash-recovery 的"位图 + 游标"机制，并收录一个 TXSQL 5.7 生产事故的缺陷分析（iWiki 4041668646，2026-09-17；**已上报上游 bugs.mysql.com 为 #121314，2026-09-18，状态 Open、S2**）：`clear_mts_recovery_groups()` 释放恢复位图时**未复位游标**，导致"新位图配旧游标"，applier 读错位把从未执行的事务整组**静默跳过**（不执行、不取 GTID、无任何告警）。本章同时给出该缺陷在 8.0.39 的现状核对结论。

#### 为什么需要 recovery：LWM 落后于 worker 实况

##### LWM 到底是什么（本章基石）

全章反复以 LWM 为原点，先把三个相关但**不同**的"位置"分清：

| 概念 | 存储 | 含义 |
|---|---|---|
| **GAQ 的 lwm**（`rli->gaq->lwm`） | 内存（GAQ） | GAQ 中**从头连续标记为 done 的最后一个组**；`find_lwm()` 返回它的 index（rpl_rli_pdb.cc，注释原话 "an index below which all jobs are marked as done"） |
| **coordinator 的 `group_relay_log_pos`** | 持久化在 `slave_relay_log_info` | **就是 LWM 的位置**（见下方代码）——recovery 的起点 |
| **各 worker 的 `group_relay_log_pos`** | 持久化在 `slave_worker_info` | 单个 worker 实际跑到的位置，**可以远在 LWM 前面** |

**LWM 如何推进**——`mta_checkpoint_routine` 的 "Coordinator::commit_positions" 段（rpl_replica.cc）：

```cpp
  mysql_mutex_lock(&rli->data_lock);
  /*
    "Coordinator::commit_positions"

    rli->gaq->lwm has been updated in move_queue_head() and
    to contain all but rli->group_master_log_name which
    is altered solely by Coordinator at special checkpoints.
  */
  rli->set_group_master_log_pos(rli->gaq->lwm.group_master_log_pos);
  rli->set_group_relay_log_pos(rli->gaq->lwm.group_relay_log_pos);
```

即：**coordinator 的位置 ≡ GAQ 的 lwm**。lwm 由 `move_queue_head()` 推进（从队头弹掉已完成的组，把 lwm 前进到最后一个"连续 done"的组，见第十章 10.2/10.3），再由 `flush_info()` 落盘。

**★ 关键：LWM 是"连续完成"的水位，不是"最快的 worker"跑到的位置。** 只要任意一个 worker 卡住（如锁等待），它后面那些**已被别的 worker 完成**的组就无法计入 LWM——这就是 LWM 必然落后于部分 worker 实况的原因，也是 recovery 存在的全部理由。

> 三个位置在 `mts_recovery_groups()` 里同时出场：`cp` = rli 的位置（**即 LWM**，用于和 worker 比较），`w_last` = 某 worker 的位置，`mts_event_coord_cmp(&w_last, &cp) > 0` 就是"该 worker 跑到了 LWM 前面"。

##### 为什么需要 recovery

worker 之间进度不齐，可能出现 worker-3 已完成第 9 组、worker-2 还卡在第 5 组的情形。此时若从库崩溃：

- LWM（已持久化）落后于部分 worker 的实际完成进度
- 重启后从 LWM 回放，那些"已执行但未计入 LWM"的组如果重放，会**重复执行**（非 GTID 模式下直接双写/主键冲突）

recovery 的目的：**把"已执行"的组标记出来跳过，只补"未执行"的 gap**。数据来源是 `slave_worker_info` 表里每个 worker 的 checkpoint（`checkpoint_seqno` + `group_executed` 位图，见第十章 10.4）。

**开关：`recovery_parallel_workers`**（rpl_rli.h，注释 "number of workers while recovering"）。要不要计算位图，**取决于它是否非 0**——两个入口都靠它分派：

```cpp
  /* rli_init_info，rpl_rli.cc */
  if (inited) {
    return recovery_parallel_workers ? mts_recovery_groups(this) : 0;
  }
  /* START SLAVE 路径，rpl_replica.cc、:2137 同样判它 */
```

★ **它是持久化字段，不是当前配置值**——读自 rli repository（rpl_rli.cc，仅当 `lines >= LINES_IN_RELAY_LOG_INFO_WITH_WORKERS`），由 `flush_info` 写回（:2321）。全部赋值点：

| 位置 | 动作 |
|---|---|
| rpl_rli.cc | 构造时置 0 |
| **rpl_rli.cc**（在 `mts_finalize_recovery()` 内） | 恢复完成后**复位为 `replica_parallel_workers`** |
| rpl_info_factory.cc（`reset_workers`）/ rpl_rli.cc（rli reset） | 置 0 |

两点推论：

1. 5.7 的 GTID 旁路（见 13.4）正是把它置 0 来绕开整条路径——旁路作用于**这个持久化开关**
2. **"当前是单线程"不等于"不会进 recovery"**：判据是持久化下来的 `recovery_parallel_workers`，若上次是 MTS 运行则它仍非 0。这也解释了 `START SLAVE UNTIL SQL_AFTER_MTS_GAPS` 为何能用（`Until_mts_gap::init()`，rpl_replica_until_options.cc）**反过来**把它赋给 `opt_replica_parallel_workers`，即"按上次 MTS 的 worker 数来补 gap"

MTS recovery 用**一张位图 + 一个游标**决定"哪些组跳过"。理解缺陷的前提是分清两者的坐标系。

##### 位图数据的来源（上游）：worker 的 `group_executed`

recovery 位图的每一位不是凭空算的，它来自**各 worker 自己持久化的执行位图**。worker 每提交一组（`Slave_worker::commit_positions`，rpl_rli_pdb.cc）：

```cpp
  bitmap_set_bit(&group_executed, ptr_g->checkpoint_seqno);   // ① ★ 记下"这一组我执行过"
  worker_checkpoint_seqno = ptr_g->checkpoint_seqno;
  group_relay_log_pos = ev->future_event_relay_log_pos;
  group_master_log_pos = ev->common_header->log_pos;
  ...
  return flush_info(force);                                   // ② ★ 落盘到 slave_worker_info
```

位图以 **BLOB 字段**持久化（`Slave_worker::write_info`，rpl_rli_pdb.cc）：`buffer = group_executed.bitmap`、`nbytes = no_bytes_in_map(&group_executed)`；启动时由 `read_info` 读回（:512-544）。

位图大小按阶段取不同值（rpl_rli_pdb.cc）：

```cpp
  size_t num_bits = is_gaps_collecting_phase ? MTS_MAX_BITS_IN_GROUP
                                             : c_rli->checkpoint_group;
```

`MTS_MAX_BITS_IN_GROUP = (1L << 19) - 8 = 524280`（rpl_replica.h）——正是 BLOB 上限 65535 字节 × 8 位；正常运行时用 `replica_checkpoint_group`（默认 512，合法范围上界即 `MTS_MAX_BITS_IN_GROUP`，sys_vars.cc）。

★ **worker 位图是"滑动窗口"**（rpl_rli_pdb.cc）：位图写满后整体左移 `ptr_g->shifted` 位再清零尾部，只保留最近若干组。正因为它是窗口内的**局部坐标**，recovery 计算时才需要那个 `(checkpoint_seqno + 1) - recovery_group_cnt` 平移，把局部坐标换算到"自 LWM 起"的全局槽位——13.2 里那步 shift 的必要性就在这里。

> 这正是第十章 GAQ 与 checkpoint 机制的落点：`mta_checkpoint_routine` 推进 GAQ 与 worker checkpoint，recovery 的数据源与之同源。

#### 位图（生产侧）与游标（消费侧）

##### 生产侧：`mts_recovery_groups()`（rpl_replica.cc）

入口有两道闸（rpl_replica.cc）：

```cpp
  /*
     Although mts_recovery_groups() is reentrant it returns
     early if the previous invocation raised any bit in
     recovery_groups bitmap.
  */
  if (rli->is_mts_recovery()) return false;        // ① 同一轮不重复计算（cnt != 0 即已在恢复中）

  /*
    The process of relay log recovery for the multi threaded applier
    is focused on marking transactions as already executed so they are
    skipped when the SQL thread applies them. ...
    When GTID_MODE=ON however we can use the old relay log position, even if
    stale as applied transactions will be skipped due to GTIDs auto skip
    feature.
  */
  if (global_gtid_mode.get() == Gtid_mode::ON && rli->mi &&
      rli->mi->is_auto_position()) {
    rli->mts_recovery_group_cnt = 0;
    return false;                                  // ② ★ GTID 旁路（见 13.4）：整个位图机制不启用
  }
```

第一步，筛选"有价值的 worker"（rpl_replica.cc）：worker 记录的最后执行位置 `w_last` **大于** coordinator 的 LWM 坐标 `cp` 才纳入 `above_lwm_jobs`，否则说明该 worker 干的活已全部被 LWM 覆盖，直接删掉。

第二步，**从 LWM 开始扫 relay log，逐个数组**，找到每个 worker 的 checkpoint 坐标后做平移（rpl_replica.cc）：

```cpp
    recovery_group_cnt = 0;
    not_reached_commit = true;
    ...
    offset = rli->get_group_relay_log_pos();        // ① 起点 = 当前 LWM

    while (not_reached_commit) {
      if (relaylog_file_reader.open(linfo.log_file_name, offset)) { ... }

      while (not_reached_commit &&
             (ev = relaylog_file_reader.read_event_object())) {
        if (ev->get_type_code() == binary_log::ROTATE_EVENT ||
            ev->get_type_code() == binary_log::FORMAT_DESCRIPTION_EVENT ||
            ev->get_type_code() == binary_log::PREVIOUS_GTIDS_LOG_EVENT) {
          delete ev; ev = nullptr; continue;        // ② 三种管理事件不计数
        }

        if (ev->starts_group()) {
          flag_group_seen_begin = true;
        } else if ((ev->ends_group() || !flag_group_seen_begin) &&
                   !is_gtid_event(ev)) {            // ③ ★ 组边界判定
          flag_group_seen_begin = false;
          recovery_group_cnt++;                     // ④ 数到自 LWM 起第 cnt 组

          if ((ret = mts_event_coord_cmp(&ev_coord, &w_last)) == 0) {
            for (uint i = (w->worker_checkpoint_seqno + 1) - recovery_group_cnt,
                      j = 0;
                 i <= w->worker_checkpoint_seqno; i++, j++) {   // ⑤ ★ 坐标平移
              if (bitmap_is_set(&w->group_executed, i)) {
                bitmap_test_and_set(groups, j);     //    局部位 → 全局槽位
              }
            }
            not_reached_commit = false;             // ⑥ 找到该 worker 的 checkpoint 即停
          } else
            assert(ret < 0);
        }
        delete ev; ev = nullptr;
      }
      relaylog_file_reader.close();
      offset = BIN_LOG_HEADER_SIZE;                 // ⑦ 跨文件继续找
      if (not_reached_commit && rli->relay_log.find_next_log(&linfo, true)) { ... }
    }

    rli->mts_recovery_group_cnt =
        (rli->mts_recovery_group_cnt < recovery_group_cnt
             ? recovery_group_cnt
             : rli->mts_recovery_group_cnt);        // ⑧ ★ 取所有 worker 扫出的最大组数
```

由此得到**位图坐标系**：

| 属性 | 含义 |
|---|---|
| 槽位 `j` | **从 LWM 起算、relay log 顺序上的第 j 个组**（0-based） |
| `j` 位 = 1 | 该组上一 session **已被某个 worker 执行过** → 应跳过 |
| `j` 位 = 0 | 该组是 **gap** → 必须执行 |
| 原点 | **计算那一刻的 LWM**（`get_group_relay_log_pos()`） |

⑤ 的平移把 worker 局部坐标（`checkpoint_seqno`）换算到全局坐标：代入 `i = checkpoint_seqno` 得 `j = recovery_group_cnt − 1`，即该 worker 最后执行的那组恰好落在全局最后一个槽位——**平移本身正确**。

##### 消费侧：`mts_recovery_index`（游标）

跳过决策（log_event.cc）：

```cpp
  if (rli->is_mts_recovery()) {
    bool skip = bitmap_is_set(&rli->recovery_groups, rli->mts_recovery_index) &&
                (get_mts_execution_mode(rli->mts_group_status ==
                                        Relay_log_info::MTS_IN_GROUP) ==
                 EVENT_EXEC_PARALLEL);
    if (skip) {
      return 0;                                  // ★ 整组静默跳过：不执行、不取 GTID、无告警
    } else {
      int error = do_apply_event(rli);           // gap 组由 coordinator 自己执行
      ...
    }
  }
```

事件进入 `exec_relay_log_event` 时还要先过一道（rpl_replica.cc）：

```cpp
  if (!(rli->is_mts_recovery() &&
        bitmap_is_set(&rli->recovery_groups, rli->mts_recovery_index))) {
    reason = ev->shall_skip(rli);
  }
```

即**已被位图判定为跳过的事件，连 `shall_skip()` 都不调用**——避免 `sql_slave_skip_counter` 等"跳过"语义与 recovery 的跳过决策相互干扰。

**跳过条件的第二半**：`get_mts_execution_mode(...) == EVENT_EXEC_PARALLEL` ——只有当该事件**本来会被分发给 worker**（log_event.h "Event is run by a Worker"）时，位图才有意义、才允许跳过。若返回 `EVENT_EXEC_ASYNC` / `EVENT_EXEC_SYNC`（由 Coordinator 执行的事件，如 ROTATE、FORMAT_DESCRIPTION、INCIDENT、STOP，见 `is_mts_sequential_exec()`，log_event.h），则**一律不跳过**：它们不属于任何 worker 的"组工作"，位图对它们无意义，跳过会漏掉 relay log 切换这类关键动作。

游标推进（rpl_replica.cc）：

```cpp
    if (!error && rli->is_mts_recovery() &&
        ev->get_type_code() != binary_log::ROTATE_EVENT && ...) {
      if (ev->starts_group()) {
        rli->mts_recovery_group_seen_begin = true;
      } else if ((ev->ends_group() || !rli->mts_recovery_group_seen_begin) &&
                 !is_gtid_event(ev)) {
        rli->mts_recovery_index++;               // ★ 每消费一组 +1
        if (--rli->mts_recovery_group_cnt == 0) {  // ★ 只有完整跑完才归零
          rli->mts_recovery_index = 0;
          LogErr(INFORMATION_LEVEL, ER_RPL_MTA_RECOVERY_COMPLETE, ...);
        }
      }
    }
```

注意两边的**组判定是同一条**（`(ends_group() || !seen_begin) && !is_gtid_event()`，rpl_replica.cc 与 :4727）——所以**位图本身不会算错，错的只可能是读它的游标**。

##### 不变式

> applier 正在看"自 LWM 起第 j 个组"时，`mts_recovery_index` 必须恰好等于 j。
> 即：**位图的原点与游标的起点必须是同一个 LWM。**

而 recovery 每成功恢复一组，LWM 就会前进并落盘。于是每轮 `START SLAVE`：

| | 原点/起点 | 每轮 START SLAVE |
|---|---|---|
| 位图 | 计算时的 LWM | **重新计算，原点跟着新 LWM 前移** |
| 游标 | 应为 0（从 LWM 开始读） | **不复位，保留上一轮的累加值**（见 13.3） |

**recovery 的终点：`mts_finalize_recovery()`**（rpl_rli.cc）——每轮恢复完整跑完（`--cnt == 0`）时调用（rpl_replica.cc；另在"本轮本就无 gap"时于 rpl_replica.cc 立即调用）：

```cpp
  for (Slave_worker **it = workers.begin(); !ret && it != workers.end(); ++it) {
    ret = (*it)->reset_recovery_info();          // ① 清空各 worker 的 recovery 信息
  }
  /*
    The loop is traversed in the worker index descending order due
    to specifics of the Worker table repository that does not like
    even temporary holes. Therefore stale records are deleted
    from the tail.
  */
  for (i = recovery_parallel_workers; i > workers.size() && !ret; i--) {
    Slave_worker *w = Rpl_info_factory::create_worker(repo_type, i - 1, this, ...);
    if (w) { ret = w->remove_info(); delete w; } // ② 倒序删除多余的 worker 记录
    else   { ret = true; goto err; }
  }
  recovery_parallel_workers = replica_parallel_workers;   // ③ 复位开关
```

它做了三件事：清 worker 恢复信息 → **倒序**删掉多余的 worker 表记录（注释说明 Worker 表仓库不接受临时空洞）→ 复位开关。失败时回退到 `Rpl_info_factory::reset_workers(rli)`（rpl_replica.cc）。

#### 缺陷：`clear_mts_recovery_groups()` 不重置游标

缺陷函数（rpl_rli.h）：

```cpp
  inline void clear_mts_recovery_groups() {
    if (recovery_groups_inited) {
      bitmap_free(&recovery_groups);
      mts_recovery_group_cnt = 0;
      recovery_groups_inited = false;
      // ← mts_recovery_index、mts_recovery_group_seen_begin 都不在此
    }
  }
```

`clear_mts_recovery_groups()` 的调用点共 **3 处**，且**三处都不重置游标**——缺陷对三个入口一视同仁：

| 调用点 | 场景 |
|---|---|
| rpl_replica.cc | SQL 线程退出（abort / `STOP SLAVE`） |
| rpl_replica.cc | `mts_recovery_groups()` 末尾：`if (mts_recovery_group_cnt == 0) clear_mts_recovery_groups();`（算出无 gap 则清理） |
| rpl_info_factory.cc | `reset_workers` |

`mts_recovery_index` 的**全部**赋值点（全库仅 3 处）：

| 位置 | 动作 |
|---|---|
| rpl_rli.cc | 构造函数置 0 |
| rpl_replica.cc | 每恢复一组 `++` |
| rpl_replica.cc | **仅当 recovery 完整跑完**（`--cnt == 0`）归 0 |

**触发链**：

1. R2：recovery 启动，**消费** k 组（注意：被跳过的组同样计数——游标按"消费的组"推进，不区分跳过还是应用）→ `index = k`；**LWM 同步前移并落盘**——源码保证在 rpl_replica.cc：每结束一组（无论该组是被跳过还是被应用）都会 `mts_recovery_group_seen_begin = false;` 并 `flush_info(RLI_FLUSH_IGNORE_SYNC_OPT)` 持久化位置
2. 第 k+1 组（gap）应用时撞临时错误（如 1205）→ 重试耗尽 → SQL 线程 abort（`STOP SLAVE` 也一样：无论主动停止还是错误中止，都从 `handle_slave_sql` 主函数退出）
3. 退出路径（rpl_replica.cc）`clear_mts_recovery_groups()`：位图释放、cnt 归 0，**游标残留 k**
4. R3：`START SLAVE` → `rli_init_info`（rpl_rli.cc）重新调 `mts_recovery_groups()`：位图原点 = 新 LWM（前移了 k 组）
5. applier 从新 LWM 读，第一个组（槽位 0）却被读成 `bit[k]` —— **错位**

**★ 为什么现有测试发现不了**（bug #121314 指出的关键点）：游标唯一的"自然归零"时机是 `Relay_log_info` 的**构造函数**（rpl_rli.cc）——也就是 **mysqld 进程重启**才会走的路。所以所有基于"杀进程再启动"的 crash-recovery 测试（如 `rpl_mts_logical_clock_recovery`）每一轮都拿到全新归零的游标，**永远测不到陈旧游标**。而触发链恰恰**不需要 mysqld 重启**：只要 SQL 线程停止（临时错误重试耗尽 abort，或 `STOP SLAVE`）再 `START SLAVE`，且恢复过程部分推进——这正是 HA 故障恢复循环的常规形态（事故报告提到的 `ha_rejoin_slave` 会重复 `start slave`——**该机制属于云平台/运维系统，不在社区源码内**，此处仅作触发场景说明）。

**错位为何"看起来没出错"**（事故文档的精华）：错位让各组读到**邻居的位**。后面的组本就"已执行、该跳过"，读邻居的 1 照样跳过——"侥幸正确"。**唯一的实质损失是槽位 0 的那个 gap**：它是唯一"本该执行却被读成已执行"的组。所以每次错位只丢一个事务，复制状态一切正常，只在后续依赖它的语句报 1032 时才间接暴露。

**数值推演**（事故复现的真实数据）：R2 后 `gtid_executed = 1-6:8-15`（G1=6 已补、T=7 仍缺，index=1）；R3 位图重算后 `SKIP_bits={1..8}`（槽位 0=T 未置位，位图正确），但 index 仍为 1 → applier 看 T 时读 `bit[1]`（=1）→ **T 被静默跳过**；8~14 连锁读邻居位跳过；读 INS(15) 时游标已到 9，超出本次位图的**槽位范围**（0..8；位图本身有 524280 位，不存在内存越界，只是该位未置位 = 0）→ 不跳过、正常执行 → 恢复计数归零、恢复结束。决定性日志：`SILENTLY SKIPPING GTID 7 (recovery_index=1)`。

**★ 错位是连锁的**——这解释了事故里为什么会丢**两个**事务（2454 与 2575）：一轮错位只丢 slot 0 的一个 gap，但这一轮里"成功恢复"的组又把 index 推高了，于是下一轮继续错位、再丢一个。只要 HA 反复 `START SLAVE` 且每轮都部分推进，空洞就会一个接一个累积，而 `gtid_executed` 与复制状态始终"看起来正常"。

**边界情形**：若某轮 slot 0 恰好**不是** gap（LWM 处那组本就已执行），错位读到的邻居位也多半为 1，该轮不产生损失——这正是缺陷极难被发现的原因：多数轮次无损失，只有"slot 0 恰好是 gap"时才丢数据。

#### 8.0.39 vs 5.7：GTID 旁路的位置（★ 事故配置在 8.0 是否可达）

事故实例配置是 **GTID ON + AUTO_POSITION=1**，那么 8.0.39 会不会踩同一个坑？答案是：**不会走到位图路径**。

上游 5.7 也有 GTID 旁路，但放在**调用方** `init_recovery()`（rpl_slave.cc）：

```cpp
  /* Set the recovery_parallel_workers to 0 if Auto Position is enabled. */
  bool is_gtid_with_autopos_on =
      ((get_gtid_mode(GTID_MODE_LOCK_NONE) == GTID_MODE_ON &&
        mi->is_auto_position()) ? true : false);
  if (is_gtid_with_autopos_on)
    rli->recovery_parallel_workers = 0;      // 绕过 mts_recovery_groups() 调用
```

8.0.39 把旁路**挪进函数内部**（rpl_replica.cc，见 13.2）：`gtid_mode==ON && is_auto_position()` → `mts_recovery_group_cnt = 0; return false;`。`cnt == 0` ⇒ `is_mts_recovery()` 为假 ⇒ 位图与游标整条路径都不启用，靠 **GTID auto-skip** 兜底（已执行的 GTID 自动跳过，未执行的自然会执行，与 relay log 位置无关）。

| | GTID ON + AUTO_POSITION（事故配置） | 非 GTID 复制 / GTID ON 但 auto_position=0 |
|---|---|---|
| 5.7（上游） | 旁路（调用方置 `recovery_parallel_workers=0`） | **走位图路径，缺陷可达** |
| 8.0.39 | 旁路（函数内 return） | **走位图路径，缺陷代码同构存在** |

两点结论：

1. **8.0.39 的缺陷代码与 5.7 同构**（`clear_mts_recovery_groups` 同样不重置游标），**但事故同配置（GTID ON + AUTO_POSITION）下不可达**；在非 GTID 复制（或 GTID ON 但 `auto_position=0`）下，"恢复中途成功若干组后 abort → 再 START SLAVE"的配方依然能触发错位——这是 8.0 分支需要修的残留风险
2. **疑点留给 TXSQL 确认**：事故实例 5.7.44-txsql 在 GTID ON + AUTO_POSITION=1 下仍计算了位图（复现日志有 `SKIP_bits`），而上游 5.7 的 `init_recovery()` 有上述旁路——说明 TXSQL 5.7.44 与该旁路有出入（删改或入口路径不同），需对照 TXSQL 源码核实

#### 现场特征：执行模式翻转

`is_parallel_exec()`（rpl_rli.h）：

```cpp
  inline bool is_parallel_exec() const {
    bool ret = (replica_parallel_workers > 0) && !is_mts_recovery();
    assert(!ret || !workers.empty());
    return ret;
  }
```

recovery 模式下它为假 → 事件不分发 worker、由 coordinator 自己 apply → 重试耗尽时 error log 走 **coordinator 侧**措辞（rpl_replica.cc）：

```cpp
        } else {
          thd->fatal_error();
          rli->report(ERROR_LEVEL, thd->get_stmt_da()->mysql_errno(),
                      "Replica SQL thread retried transaction %lu time(s) "
                      "in vain, giving up. Consider raising the value of "
                      "the replica_transaction_retries variable.",
                      rli->trans_retries);
        }
```

而正常 MTS 下事务由 worker 执行，重试耗尽走的是 **worker 侧**措辞（rpl_rli_pdb.cc，"worker thread retried transaction %lu time(s)"），并带 `Worker N` 前缀。所以事故中 R1/R3（worker 形态）与 R2/R4/R5（Replica SQL thread 形态）的**翻转本身就是"进入/退出 recovery 模式"的可观测指纹**。

> ⚠️ **8.0 已完成 slave→replica 改名**：日志是 "**Replica** SQL thread retried transaction"，变量名是 `replica_transaction_retries`。5.7 与旧资料里的 "Slave SQL thread ..." / `slave_transaction_retries` 在 8.0.39 已不适用——事故报告用的是 5.7 措辞，移植到 8.0 排查时要按新名检索。

#### 修复与防御

- **主修复**（8.0 同构适用）：`clear_mts_recovery_groups()` 补 `mts_recovery_index = 0;`——确保"新位图必配新游标"。bug #121314 报告者给出的最小修复也正是这一行（放在位图释放处或 `mts_recovery_groups()` 开头均可）
  - ★ **只需这一行，不必连带重置 `mts_recovery_group_seen_begin`**：后者在**每组结束时已被无条件置 false**（rpl_replica.cc），不会残留。事故文档建议"一并重置"属于稳妥起见，非必需——本报告按源码更正这一点
  - **修复安全性**：`clear_mts_recovery_groups()` 只在"退出路径"与"cnt 归零"两处调用，此刻不存在进行中的 recovery；游标归零不影响正常 MTS（recovery 模式下 `is_parallel_exec()` 本就为假，事件由 coordinator 自己 apply）
- **防御性加固**：消费点（log_event.cc）读取前校验 `mts_recovery_index < mts_recovery_group_cnt`，不一致**报错停复制**——宁可停，不静默跳
- **运维侧**：MTS 从库因临时错误 abort 后，先确认阻塞源（长事务/锁）已消除再 `START SLAVE`；HA 的 `ha_rejoin_slave`（云平台/运维机制，非社区源码）同一 GTID 连续失败时不要盲目重发 start；监控增加 `gtid_executed` 连续性（空洞）检查，比 `Seconds_Behind_Master` 更早暴露此类静默丢失

> 本章缺陷分析与复现数据源自事故报告 iWiki 4041668646（2026-09-17）与上游 **bug #121314**（2026-09-18，Open/S2，报告者称影响 5.7/8.0/9.x 全系列，官方修复尚未落地）；全部代码均已在 8.0.39 源码逐一核对，5.7 侧旁路以 mysql/mysql-server 5.7 分支源码为准。


---

## ★ 本机制里的工程实现技法

> 本章答 **how**：结合代码剖析具体实现，并写明**与教科书/论文原型的差异**。算法的选型权衡与论文溯源属「理论基础」，此处只交叉引用。

### 一、高级语法技巧 / C++ 特性

| 特性 | 用在哪 | 收益 | 代价 / 反直觉处 |
|---|---|---|---|
| **策略类派发**（抽象基类 `Mts_submode` → `Mts_submode_logical_clock` / `Mts_submode_database`） | `rli->current_mts_submode` 的虚调用 | 两种调度算法可运行时切换，coordinator 主循环不感知差异 | 最热路径（每事件）一次虚调用 |
| **模板 + 定长容器**：`circular_buffer_queue<Element_type>` + `Prealloced_array` | GAQ | 无动态分配、内存上界可预测、O(1) 入队出队 | 容量写死 = `checkpoint_group`，不能自适应 |
| **`std::atomic`**：`Slave_job_group::done`、`circular_buffer_queue::len`、`min_waited_timestamp` | 跨线程完成标志 / 无锁读长度 | coordinator 与 worker 之间无需加锁即可传递完成状态 | 语义靠"谁写谁读"的约定，`len` 用 relaxed（只作统计） |
| **模板特化的 hash**：`calc_hash<>()` 里 MURMUR32 / XXHASH64 二选一 | writeset PKE | 算法可配置（`transaction_write_set_extraction`） | 会话与全局算法不一致时必须降级并清空历史 |
| **所有权转移约定**：`*ptr_ev = nullptr` | 事件交给 worker | 显式表达"事件不再由 coordinator 释放" | 调用方必须靠指针是否被置空来分支，易误用 |

### 二、经典算法的实现落地（与原型差异）

**① 环形缓冲（GAQ）**

| 维度 | 教科书原型 | 本实现 | 差异原因 / 代价 |
|---|---|---|---|
| 游标 | `head` / `tail` 均取模 | `entry` 归一到 `[0,capacity)`，`avail` 单调增长到 `2*capacity` | `len = avail - entry` 可无锁原子读；代价是所有访问都要 `% capacity` |
| 容量 | 通常与数据量相关 | 绑定 `replica_checkpoint_group` | 并行窗口 = checkpoint 窗口，一个长事务就能堵死整窗 |
| "满"的处理 | 阻塞或扩容 | 靠 `force` checkpoint 保证永不真满 | "满"被转化成 coordinator 忙轮询 |

**② 位图滑动窗口（worker `group_executed`）**

| 维度 | 原型 | 本实现 | 差异原因 / 代价 |
|---|---|---|---|
| 淘汰 | LRU / 老化 | **整体左移**（`bitmap_copy` → 清空 → 逐位减 `shifted`）后全清 | 与 checkpoint 窗口对齐，恢复时对多 worker 位图求并集即可；代价是 O(checkpoint_group) 扫描与周期性"断崖" |
| 容量 | 按需 | 固定 `checkpoint_group`（恢复态用 `MTS_MAX_BITS_IN_GROUP` 524280） | worker 表 BLOB 大小固定 |

**③ 行哈希集合（writeset）**：原型是无冲突检测的哈希集合；本实现用 **64 位哈希**，碰撞只会把两个不冲突的事务判成冲突 ⇒ **只损失并行度，不丢正确性**（安全方向）；容器选 `std::map` 而非 `unordered_map`，换来有序稳定、无 rehash 抖动，代价是 O(log n)。

**④ 逻辑时钟（Lamport clock 家族）**：原型是单纯递增计数器；本实现引入 **每文件 offset**——`state` 绝对单调、`offset` 在 binlog 轮转时抬升，落盘存 `state - offset`。收益是**跨 binlog 文件的事务天然零依赖**（父事务在上个文件则退化为 `SEQ_UNINIT`），代价是每个事件落盘前多一次减法且需 `assert` 防越界。

### 三、复杂体系与设计模式的代码结构

**主从职责分离体系**（本篇的核心分工，用图最直观）：

```
主库（生产依赖）                         从库（消费依赖）
─────────────────                       ─────────────────
ordered_commit                          handle_slave_sql (coordinator)
  └ write_transaction                     └ exec_relay_log_event
      └ Transaction_dependency_tracker        ├ mta_checkpoint_routine (GAQ/lwm)
          ├ Commit_order   (锁间隔)            ├ get_slave_worker  ← 读 sn/lc
          ├ Writeset       (行哈希)            │   └ schedule_next_event
          └ Writeset_session (会话)            │       ├ wait_for_last_committed_trx
      → Gtid_log_event{sn, lc}  ───────────────┘       └ wait_for_workers_to_finish
                                               └ worker: slave_worker_exec_job_group
                                                   └ commit_positions → group_executed bitmap
```

**三级背压分工**（谁管什么）：

| 层次 | 约束对象 | 参数 | 阻塞时的 stage |
|---|---|---|---|
| GAQ（组数窗口） | 未 checkpoint 的组数 | `replica_checkpoint_group` | `Waiting for workers to process queue`（忙轮询） |
| worker 队列（字节窗口） | 分配给某 worker 的事件总字节 | `replica_pending_jobs_size_max` | `Waiting for Slave Worker to release partition` / `waiting for workers to process queue` |
| worker 空闲度 | 是否有 worker 的 jobs 队列为空 | `replica_parallel_workers` | `Waiting for an available worker`（`sched_yield` 忙等） |

**设计模式**：策略模式（`Mts_submode` 族）、模板方法（`get_slave_worker` 固定流程 + 子模式钩子 `schedule_next_event`）、生产者-消费者（coordinator 入队 / worker 出队 + `jobs_cond`）。

## 可观测性

> 观测点跟着机制走（核心实现里已就地交代），本章只做汇总备查。


### 系统变量与状态变量

| 参数 | 作用 | 默认值 |
|---|---|---|
| `binlog_transaction_dependency_tracking` | 主库依赖追踪模式：COMMIT_ORDER / WRITESET / WRITESET_SESSION | COMMIT_ORDER |
| `transaction_write_set_extraction` | writeset 的 hash 算法：OFF / MURMUR32 / XXHASH64 | XXHASH64 (8.0) |
| `binlog_transaction_dependency_history_size` | writeset history 最大行数 | 25000 |
| `replica_parallel_type` | 从库并行模式：DATABASE / LOGICAL_CLOCK | LOGICAL_CLOCK (8.0) |
| `replica_parallel_workers` | 从库 worker 线程数（0 = 关闭 MTS） | 4 (8.0) |
| `replica_preserve_commit_order` | 从库是否保证提交顺序与主库一致 | ON (8.0) |
| `replica_checkpoint_period` | MTS checkpoint 周期（毫秒） | 300 |

#### 兼容性约束

```cpp
// sys_vars.cc
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


### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 当前并行度 / worker 数 | SQL | `SHOW REPLICA STATUS`、系统变量 `replica_parallel_workers` |
| 调度是否卡在依赖等待 | SQL | `SHOW REPLICA STATUS` 的 `Replica_SQL_Running_State`、PFS `wait/synch/.../mts_gaq_LOCK` |
| MTS 恢复是否在跑 | SQL / DBUG | 恢复期 `is_parallel_exec()` 为假 → 日志走 "Replica SQL thread retried transaction"（见「MTS Crash Recovery」13.5）；DBUG `mta` |
| 是否有静默丢事务（GTID 空洞） | SQL | `GTID_SUBSET(master_gtid_executed, slave_gtid_executed)`；空洞检查比 `Seconds_Behind_Master` 更早发现问题（见 13.6） |


---

## Misc

> 面向二次开发：**已知缺陷**见「核心实现 → MTS Crash Recovery 与静默丢数据缺陷」（游标未复位导致静默丢事务，上游 bug #121314）；**扩展点**（加一种依赖追踪模式要改哪几处）待补。

### 扩展点：加一种依赖追踪模式要改哪几处

| # | 改动点 | 说明 |
|---|---|---|
| 1 | `enum Dependency_tracking_mode` + 系统变量 `binlog_transaction_dependency_tracking` 的取值校验 | 新增枚举值 |
| 2 | `Transaction_dependency_tracker::get_dependency()` 的 `switch` | 决定串联顺序（新 tracker 是下压还是上抬） |
| 3 | 新增 tracker 类（仿 `Writeset_session_trx_dependency_tracker`）+ 在 `Transaction_dependency_tracker` 里加成员 | 现有三个成员：`m_commit_order` / `m_writeset` / `m_writeset_session` |
| 4 | `Transaction_dependency_tracker::rotate()` 与 `tracking_mode_changed()` | binlog 轮转、运行期改模式时的清理 |
| 5 | 若需要 per-THD 状态 | 放在 `Dependency_tracker_ctx`（`rpl_context.h`），如 `m_last_session_sequence_number` |

★ **从库侧一行都不用改**——这正是"依赖前置到主库"设计的收益：从库只消费 `last_committed` / `sequence_number` 两个整数，不感知追踪模式。

### 容易误解的命名：两个 "logical clock"

本篇最易读串的一处命名。主从各有一个带 "logical clock" 名字的东西，职责完全不同：

| | **主库**：`Logical_clock` | **从库**：`Mts_submode_logical_clock` |
|---|---|---|
| 定义处 | `rpl_trx_tracking.h`（被 `Transaction_dependency_tracker` 持有） | `rpl_mta_submode.h`（继承 `Mts_submode`） |
| 归属对象 | **`MYSQL_BIN_LOG::m_dependency_tracker`**（binlog.h） | `rli->current_mts_submode` |
| 干什么 | **生产**时间戳：`step()` 分配 `sequence_number`，维护 `m_max_committed_transaction` | **消费**时间戳：读 Gtid 事件的 `last_committed`，做依赖等待与调度 |
| 关键字段 | `state`（绝对计数，永不重置）+ `offset`（当前 binlog 文件起点） | `sequence_number` / `last_committed`（当前组的）、`min_waited_timestamp`、`last_lwm_timestamp` |
| 关键方法 | `step()` / `set_if_greater()` / `update_offset()` / `rotate()` | `schedule_next_event()` / `wait_for_last_committed_trx()` / `wait_for_workers_to_finish()` |
| 触发路径 | 主库事务提交（`ordered_commit`） | 从库 applier 每读一个事件 |

**依赖追踪（dependency tracker）整体是主库侧的**：`sql/rpl_trx_tracking.cc` 里的三个 tracker（COMMIT_ORDER / WRITESET / WRITESET_SESSION）都在 binlog flush 阶段被调用，只服务于"往 Gtid_log_event 里写什么值"。从库侧对应的概念是 **MTS 调度子模式**（`Mts_submode_logical_clock` / `Mts_submode_database`）。

> 相关：binlog 写入与组提交（本篇上游）见 [`binlog.md`](binlog.md)。

### 常见问题 Q1~Q14



#### Q1: COMMIT_ORDER 模式下，为什么"锁间隔重叠"就能保证不冲突？

**核心逻辑**：如果 T2 获取所有锁时 T1 还没有释放锁，说明 T1 和 T2 锁住的是**不同的行**（如果锁的是同一行，T2 会被 T1 阻塞，不可能在这个时间点获取到锁）。既然锁住不同的行，从库上并行执行也绝不会冲突。

**形式化证明**：假设 T1 修改行 r1，T2 修改行 r2，且 T2 在 T1 commit 之前获得了 r2 的锁。如果 r1=r2，则 T2 会被 T1 的锁阻塞，不可能在此刻获取锁。因此 r1≠r2，两个事务修改不同行，可以在从库并行执行而不冲突。

#### Q2: 既然冲突也可以让从库"并行加锁等待"，为什么不这样做？

**因为 binlog 不包含锁信息。** 如果两个冲突的事务在从库并行执行：
- Worker-1 执行 T1：`UPDATE t SET a=1 WHERE id=1`
- Worker-2 执行 T2：`UPDATE t SET a=2 WHERE id=1`

它们的执行顺序无法保证。在主库，T1 持有锁 → T2 等待 → T1 提交释放锁 → T2 获取锁执行——顺序确定，最终 `a=2`。在从库如果并行执行且 T2 先于 T1 完成，最终 `a=1`——**数据不一致**。

所以 LOGICAL_CLOCK 的设计保证的是：被标记为可并行的事务，在从库上执行时**完全不需要锁等待**——它们的写集合不相交。

#### Q3: COMMIT_ORDER 的"假阳性"到底有多严重？

COMMIT_ORDER 会把**所有锁间隔不重叠的事务**都标记为"可能冲突"。但锁间隔不重叠 ≠ 行级冲突。举例：

```
T1: UPDATE t SET a=1 WHERE id=1       (锁行1)
T2: UPDATE s SET b=2 WHERE id=100     (锁表s的行100，与T1完全不相交)

如果 T1 commit → T2获取锁，lock interval 不重叠 → 被 COMMIT_ORDER 标记为串行
但实际上它们完全不冲突！
```

这就是 WRITESET 的价值：通过行级 hash 直接检测"T1 和 T2 修改了相同的行吗？"，消除这类假阳性。

#### Q4: `sequence_number` 和 binlog 中的 `last_committed` 为什么是"相对值"？

`Logical_clock` 内部用 `offset` 处理 binlog 轮转。轮转时：

```cpp
// rpl_trx_tracking.cc
void Commit_order_trx_dependency_tracker::rotate() {
  m_max_committed_transaction.update_offset(m_transaction_counter.get_timestamp());
  m_transaction_counter.update_offset(m_transaction_counter.get_timestamp());
}
```

轮转后写入 binlog 时减去 offset，使新 binlog 中的值从小的正整数开始。从库读取新 relay log 时，这些相对值独立生效。**跨 binlog 的事务依赖会丢失**（`commit_parent` 会被设为 `SEQ_UNINIT`），这意味着跨 binlog 的事务在从库可以做最大程度的并行——这是合理的，因为 binlog 轮转点本身就是天然的同步点。

#### Q5: 一张表有外键，WRITESET 为什么必须降级？

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

#### Q6: COMMIT_ORDER 是一个保守策略吗？社区有优化吗？

**是的，COMMIT_ORDER 本质上是保守的，但保守的方向不是 BGC 边界，而是"假阳性"。**

##### 保守性来自哪里？

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

##### BGC 边界不是保守性的来源

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

##### 社区优化方案

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

#### Q7: Lock Interval 和 sequence_number/last_committed 是一回事吗？为什么两个都需要？

**不是一回事。** Lock Interval 是主库运行时的**物理现象**（事务持有锁的时间窗口），而 `sequence_number` 和 `last_committed` 是对这一现象的**数值化编码**，存储在 binlog 中传给从库。

打个比方：Lock Interval 是"两辆车并排行驶的时间段"，`last_committed` 是"摄像头拍到后车时前车的车牌号"，`sequence_number` 是"当前这辆车的车牌号"。

**为什么不能只用 `last_committed`？** 因为从库还需要 `sequence_number` 做三件事：

1. **推进 LWM**：`find_lwm` 扫描 GAQ 找的是"连续已提交的 `sequence_number` 最大值"。没有 sequence_number，Coordinator 不知道各个事务在 GAQ 中的相对位置，无法判断"连续性"，LWM 永远无法推进。

2. **作为 lc 的引用目标**：`lc=1` 的语义是"等 sequence_number=1 的事务提交"。如果只有 lc 没有 sn，"等 1"等于谁就没有含义了。

3. **检测 GAQ 空洞**：`is_new_group` 的触发条件之一是 `gap_successor = (sequence_number != last_sequence_number + 1)`。如果 sn 从 1 跳到 5，说明中间有缺失，必须强制串行化以保证安全。

#### Q8: Lock Interval 为什么说"prepare 到 commit"？这是精确的吗？

这需要区分事务类型。源码注释（[rpl_trx_tracking.h](sql/rpl_trx_tracking.h)）明确区分了两种情况：

- **autocommit 事务**（单条 DML）：Lock Interval 从 "just before storage engine **prepare**" 开始，到 "just before storage engine **commit**" 结束。此时 Lock Interval ≈ prepare → commit，**近似正确**。

- **显式事务**（BEGIN...COMMIT，含多条语句）：Lock Interval 从 "**end of the last statement** before COMMIT" 开始。这是因为多条语句在执行过程中逐步获取锁，只有最后一条语句执笔结束，所有锁才算拿齐。此时的 Lock Interval ≠ prepare → commit，而是 "最后一条语句结束 → commit"。

**为什么有这种区分？** 因为 2PL 的 Lock Point 取决于锁何时拿齐。autocommit 只有一条语句，执行即拿锁，prepare 时已拿齐。显式事务可能有多条语句，锁是逐步积累的。

#### Q9: LWM 是什么的缩写？

**Low Water Mark**（低水位线）。这是计算机科学中的通用术语，描述"已处理数据与未处理数据之间的分界"。

在 MTS 中的精确定义：**GAQ 中连续已提交事务的最大 `sequence_number`**。水位线以下的事务全部已提交，可以安全回收 GAQ 空间。水位线以上的事务要么在执行中，要么在等待依赖。

#### Q10: 为什么 INSERT 一行产生三个 PKE？这难道不是表级并行吗？

**不是表级并行，是行级的冗余覆盖。** 三个 PKE 来自表上的三个唯一索引（1 个主键 + 2 个唯一键），每个唯一索引产生一个独立的行标识。

```cpp
// rpl_write_set_handler.cc
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

#### Q11: 为什么 `sequence_number` 用 `state - offset`，而不直接用全局 `state`？

**因为每个 binlog 文件是一个独立的名字空间。** 源码注释直接说明了设计意图（[rpl_trx_tracking.cc](sql/rpl_trx_tracking.cc)）：

> "Prepare sequence_number and commit_parent **relative to the current binlog**. A transaction that commits after the binlog is rotated, can have a commit parent in the previous binlog. In this case, subtracting the offset from the sequence number results in a negative number. The commit parent dependency gets lost in such case. Therefore, we log the value SEQ_UNINIT."

**轮转时发生了什么？**

```cpp
// rpl_trx_tracking.cc
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

#### Q12: writeset 的三个 hash 是行级粒度的确认

> "因为三个唯一索引所以产生了三个 hash"是正确的，**但 INSERT 只插入了一行而不是三行**。三个 hash 指向的是**同一行的三个不同唯一标识**：主键 `i=1`、唯一键 `j=2`、唯一键 `k=3`。

从代码路径来看（[rpl_write_set_handler.cc](sql/rpl_write_set_handler.cc)）：

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

#### Q13: 不同 binlog 的事务一定能并行吗？offset 是优化还是语义要求？

**不是计算优化，是语义正确性要求。**

##### 轮转时的强制同步屏障

`new_file_impl()` 在创建新 binlog 之前，会阻塞等待**所有正在提交中的事务完成**（[binlog.cc](sql/binlog.cc)）：

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

##### 为什么这个屏障保证了"不同 binlog 的事务无依赖"？

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

##### 对比：有 barrier vs 无 barrier

| | 无 barrier | 有 barrier（实际实现） |
|---|---|---|
| Binlog-2 第一条事务的依赖 | 可能依赖 Binlog-1 中尚未提交的事务 | 无依赖（所有旧事务已提交） |
| 从库实现 | 必须跨 relay log 跟踪事务状态 | 自包含，只关心当前 relay log |
| `lc = SEQ_UNINIT` | 是 hack，不安全 | 是正确语义，安全 |

**所以 offset 不是为了"减少计算量"**——没有 barrier 的话，用 offset 让 `lc` 变小反而会破坏正确性（把真的依赖错误地抹掉了）。offset 之所以安全，**正是因为 barrier 先在语义层面保证了跨文件无依赖**。offset 的作用是让 binlog 文件内的逻辑时间戳自包含、可独立解析。

##### rotate() 还做了什么

```cpp
void Transaction_dependency_tracker::rotate() {
  m_commit_order.rotate();   // offset = 当前 state
  m_writeset.rotate(1);      // 清空 writeset history
  if (current_thd)
    current_thd->get_transaction()->sequence_number = 2;
}
```

清空 writeset history 也是同理——旧 binlog 的事务已全部提交，它们的 writeset 不需要再参与冲突判断。

#### Q14: Causal Consistency（因果一致性）在并行复制中意味着什么？

##### 分布式系统中的定义

**Causal Consistency**（因果一致性）是介于强一致性（linearizability）和最终一致性（eventual consistency）之间的一致性模型：

> **If operation A happens-before operation B（A causally precedes B）, then all nodes must see A before B. Operations that are not causally related（concurrent）can be seen in any order.**

##### 在 MySQL 并行复制中的映射

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

##### LOGICAL_CLOCK 如何实现因果一致性

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

##### 与 Group Replication 的 Causal Consistency 的区别

如果在外部讨论中看到 "Causal Consistency" 指的是 MySQL Group Replication 的 `group_replication_consistency=BEFORE_ON_PRIMARY_FAILOVER`，那是另一个层面——它保证主库故障切换时，新主库不丢失旧主库上已提交但尚未复制到所有节点的"因果链末端"事务。

在 MTS LOGICAL_CLOCK 的语境中，因果一致性就是指：**从库对事务的执行顺序必须尊重主库上的 happens-before 关系**。`last_committed` 是因果依赖的保守上界——"我最晚需要等到谁，因为可能与它存在因果依赖。"

---


---

## 参考

**官方 WorkLog / 文档**

- WorkLog [WL#7165](https://dev.mysql.com/worklog/task/?id=7165)：MTS 与 LOGICAL_CLOCK 的设计（本篇第八章有其经典图解的逐事务分析）
- WorkLog [WL#6314](https://dev.mysql.com/worklog/task/?id=6314)：WRITESET 基于行的并行复制
- *MySQL 8.0 Reference Manual → Replication and Binary Logging Options and Variables*
- *MySQL 8.0 Reference Manual → Replication Implementation → Parallel Applier*
- Bug [#121314](https://bugs.mysql.com/bug.php?id=121314)：MTS recovery silently skips transactions, leaving holes in gtid_executed（本篇第十三章分析的缺陷）

**内核月报 / 技术文章**

- 内核月报《MySQL 并行复制演进》系列
- 《MySQL 8.0 WRITESET 并行复制原理》

**相关文档**

- 上游：binlog 的写入与组提交见 [`binlog.md`](binlog.md)
- 相关：GTID 与自动定位见 [`gtid.md`](gtid.md)；复制拓扑与故障转移见 [`replication.md`](replication.md)
- 相关：锁释放时机（2PL）见 [`../../lock/transactional/innodb_trx_lock.md`](../../infra/lock/transactional/innodb_trx_lock.md)
- 相关：从库崩溃后的 redo 恢复见 [`../../innodb/recovery.md`](../../innodb/recovery.md)
