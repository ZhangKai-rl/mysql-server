# MySQL 从库提交保序深度剖析：replica_preserve_commit_order

> 一句话：MTS 让 worker **并行执行、乱序完成**；`replica_preserve_commit_order`（8.0 默认 ON）只把"提交"这一环强制拉回 relay log 顺序。实现上**不分析因果**，只做"取号排队"：先派发的事务先提交。

> 阅读前提：MTS 调度（`last_committed`/`sequence_number`/GAQ/依赖追踪）见 [`prpl.md`](prpl.md)；主库 binlog 组提交与 `binlog_order_commits` 见 [`binlog.md`](binlog.md)；MDL 死锁检测基础见 [`../../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)。

## 目录

- [概述](#概述)
  - 它解决什么问题
  - 它不解决什么问题（边界）
- [理论基础](#理论基础)
  - 问题的精确定义：乱序提交如何破坏因果一致性
  - 为什么用"位置"替代"因果"
  - "提交序"的精确定义：三段接力
- [核心实现](#核心实现)
  - 架构总览：Commit_order_manager 与纯位置 FIFO 队列
  - 生命周期与开关
  - 两条提交路径：等待发生在哪
  - 等待与放行：wait_on_graph / finish_one / wait_and_finish
  - 同步原语：复用 MDL 的 wait-for 图
  - 设计决策一览
- [演进与历史](#演进与历史)
- [缺陷档案](#缺陷档案)
- [可观测性](#可观测性)
  - 参数矩阵
  - 性能特征
  - 生产实践要点
  - 故障指纹
- [Misc](#misc)
  - 容易误读的点
  - 与其他机制的关系
- [参考](#参考)

---

## 概述

### 它解决什么问题

MTS 并行回放的核心假设是：`last_committed` 相同的事务在主库上无锁冲突，从库可以任意调度。但"无锁冲突"不等于"无因果依赖"——WRITESET 依赖追踪存在**假阴性**（读写因果但写不同表：T1 写 `t1`，T2 读 `t1` 后写 `t2`，writeset 不交集 → `lc` 相同 → 从库并行）。RBR 消解了**存储层**的执行因果（行值是主库算好的，乱序回放数据终态一致），但**客户端读到的中间态**仍会违反因果：

```
主库:  T1: UPDATE t1 SET a=1   (先)
       T2: 读 t1 后 UPDATE t2  (后，依赖 T1 的值)

从库 MTS 并行回放，T2 先提交:

时刻 t1: Worker-2 提交 T2 → t2.b=10
         客户端 SELECT: t1.a 还是旧值, t2.b=10
         → "b=10 存在，但产生它的 a 还是旧值" → 因果不一致！
时刻 t2: Worker-1 提交 T1 → 现在才一致
```

`replica_preserve_commit_order` 就是消除这个窗口：worker 可以乱序执行，但**提交必须按 relay log 顺序**——T2 执行得再快，也要等 T1 提交完才轮到它。

### 它不解决什么问题（边界）

| 边界 | 说明 |
|------|------|
| 只保**提交序**，不保执行序 | undo 写入、行定位、MVCC 更新全部在 apply 阶段并行完成，串行化的只是"写 binlog + 引擎 commit"这最后一环（见「性能特征」） |
| **不检测因果** | 它不知道 T2 是否真的依赖 T1——只要 T2 的号排在 T1 后面，提交就必须在 T1 后面。比因果分析保守，但实现简单且绝对正确（见「为什么用位置替代因果」） |
| 纯从库参数 | 主库不知道、不参与；主库侧对应物是 `binlog_order_commits`，但两者互不依赖（见「三段接力」） |
| 不管 GTID 空洞 | 空洞是"事务缺失"（没执行），保序是"顺序对"（执行的顺序）——两个正交维度（见「理论基础」末节） |
| 不管行级锁冲突 | worker 之间真实的行锁竞争照常发生（锁等待/死锁/重试），保序只管提交这一环；两者的交互是跨层死锁（见「缺陷档案」） |

核心类：`Commit_order_manager`（`sql/rpl_replica_commit_order_manager.{h,cc}`）。

---

## 理论基础

### 问题的精确定义：乱序提交如何破坏因果一致性

主库上事务的因果链（T2 读到了 T1 写的值）由提交顺序隐含保证：T1 先提交，T2 才能读到并提交。从库并行回放时，若 T2 先提交，会产生三类后果：

**后果一：查询从库看到违反因果的状态**

```
主库:
  T1(seq=1): DELETE FROM orders WHERE order_id=100    // 删订单
  T2(seq=2): INSERT INTO archive VALUES(100, ...)      // 归档（依赖删除）

从库 T2 先提交:
  SELECT orders WHERE order_id=100   → 存在（T1 还没删）
  SELECT archive WHERE order_id=100  → 存在（T2 已归档）
  → 同一订单同时存在于两个表 → 语义矛盾！
```

**后果二：级联复制放大问题**。若该从库又是下游的主库，其 binlog 记录的是**实际提交顺序**而非原始顺序——下游从库继承错误顺序，因果不一致传播到整个拓扑。级联链每深一级，窗口期拉长一截。

**后果三："读己之写"失效**。应用在主库上先 `UPDATE status='done'` 提交，再 `UPDATE next='archived' WHERE status='done'`——从库上若第二个事务先提交、第一个还没提交，读到从库的客户端看不到 `status='done'` 的行，业务逻辑错误。

**与 GTID 空洞的区分**（两个容易混淆的维度）：

| | GTID 空洞 | 因果不一致（本篇主题） |
|---|---|---|
| 本质 | 事务**缺失**（有 GTID 没执行） | 事务存在但提交**顺序错** |
| 原因 | 多源复制交错、故障切换、手动跳过 | WRITESET 假阴性 + 无保序 |
| 从库检测 | `sequence_number != last_seq+1` → `is_new_group=true` 串行化 | 无法检测（顺序"错"在从库看来没有异常） |
| 处置 | 等 worker 清空再继续 | `replica_preserve_commit_order` 强制顺序 |

### 为什么用"位置"替代"因果"

正确解法是"从库重算因果"：让从库知道 T2 依赖 T1，只对真依赖做排序。这条路走不通，原因是**信息在 binlog 里已经丢了**：

- 主库侧依赖追踪（COMMIT_ORDER/WRITESET）的目的是**提高并行度**，它输出的 `last_committed` 是"保守上界"——WRITESET 存在假阴性（读写因果但写不同表），从库拿到 `lc=0` 无法得知"其实有依赖"；
- 从库没有主库的锁获取序列，无法事后重建 Lock Interval 重叠关系；
- 任何"检测因果"的方案都必须回答假阴性的代价——漏一个就回到因果不一致。

所以 MySQL 的选择是**放弃因果分析，用位置做全序**：

```
这不是"冲突检测"（你没有和我抢同一个柜台），
而是"取号排队"（你号比我小，你先办完我才能办）。

InnoDB 锁 + MDL  = 柜台冲突检测："我们都在办同一张表？等一下吧"
Commit_order_manager = 取号机："不管你办哪张表，号小的先办"
```

因果关系由"取号顺序 = binlog 顺序"**隐含保证**。`Commit_order_manager` 不知道也不需要知道 T2 是否真的依赖 T1——只要号在 T1 之后，提交就必须在 T1 之后。保守（把无因果的事务也串行化了提交）但正确（漏不了）。

在 MTS 因果一致性保障的四层体系里，它是最后一层兜底：

```
Layer 1: COMMIT_ORDER   → Lock Interval 理论，假阴性 = 0，假阳性多
Layer 2: WRITESET       → 消除假阳性，但引入假阴性（读写因果漏判）
Layer 3: RBR            → 行值是终态，消解存储层执行因果
Layer 4: replica_preserve_commit_order → 强制提交序 = binlog 序，
         ★ 完全不分析因果，用位置替代 → 兜住 Layer 2 的假阴性
```

### "提交序"的精确定义：三段接力

"保序"保的是哪个序？它是**三段接力**：

```
① 主库 flush 序（= binlog 中事务出现顺序 = sequence_number 顺序）
     ↑ 由 BGC flush stage 的 Leader-Follower 队列天然保证（写文件字节流必须串行，没得选）
② 主库引擎提交序
     ↑ 由 binlog_order_commits（默认 ON）保证 = ①；OFF 时引擎各自提交、顺序不保证
③ 从库引擎提交序
     ↑ 由 replica_preserve_commit_order（默认 ON）的 Commit_order_manager 保证 = ①
```

三个关键结论：

1. **复制协议传输的"序"是 ①（flush 序），不是 ②（主库引擎提交序）**——relay log 的字节顺序就是主库写 binlog 的顺序，与主库引擎何时提交无关。所以**主库 `binlog_order_commits=OFF` 时从库保序完全不受影响**：主库引擎提交可以乱，但 binlog 序永远严格有序，从库照 relay log 保序即可。两个参数互不依赖。
2. **主库不需要为复制配任何"保序"参数**——从库要的参照物（binlog 序）在 flush stage 天然就有。`binlog_order_commits` 保证 ②=① 的目的是别的（GTID 连续区间、Clone/一致性快照、`trx->no` 分配序），不是复制协议的要求（详析见 [binlog.md](binlog.md)「binlog_order_commits 参数」）。
3. **两侧参数同名对照**：主库 `binlog_order_commits` 管"引擎提交序是否等于 binlog 序"（commit stage 串行 vs 并行），从库 `replica_preserve_commit_order` 管"从库引擎提交序是否等于 relay log 序"（FIFO 队列保序）——**同一件事在主从两端的镜像实现**：主库侧靠 BGC 队列天然有序 + 可选串行化，从库侧靠 `Commit_order_manager` 显式排队。

---

## 核心实现

### 架构总览：Commit_order_manager 与纯位置 FIFO 队列

8.0.39 的真实类定义（2024 年已完成一轮 lock-free 重构，文件头 Copyright 2024）：

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

`m_workers` 不是普通 deque，是 `cs::apply::Commit_order_queue`——**双结构**：按 worker id 索引的静态 Node 数组 + 无锁 FIFO 整数队列（只存 worker id）。关键 Node：

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

**两个序列号，必须分清**（最常见的误读是"保序靠 binlog 的 sequence_number 排序"——完全不是）：

| | `Slave_worker::sequence_number()` | `Commit_order_queue::commit_sequence_nr` |
|---|---|---|
| 来源 | GAQ（binlog 的原始 sequence_number） | `m_commit_sequence_generator->fetch_add(1)` |
| 何时赋值 | Coordinator 分发事务时写入 GAQ | Worker `push()` 入队列时 |
| 含义 | 事务在主库 binlog 中的全局序号 | 提交请求在队列中的本地序号 |
| 用途 | 死锁检测的方向判断 | CAS 原子协议：只有前驱能解锁后继 |
| 与提交排序关系 | ❌ 无关 | ❌ 无关（排序靠队列位置） |

**提交排序靠 FIFO 链表位置**。`push()` 尾插：

```cpp
void Commit_order_queue::push(value_type index) {
  sequence_type next{Node::NO_SEQUENCE_NR};
  do { next = this->m_commit_sequence_generator->fetch_add(1); }
  while (next <= Node::SEQUENCE_NR_FROZEN);        // 保留 0/1 为哨兵值
  this->m_workers[index].m_commit_sequence_nr->store(next);
  this->m_commit_queue << index;                   // 追加到 FIFO 尾部
}
```

`commit_sequence_nr` 的单调递增只是实现上的自然结果，不是排序依据。它的真实用途是**"证明你是前驱"的凭证**：`finish_one` 里 pop 出 N 后，只允许持有 N+1 的节点被解封（见「wait_on_graph / finish_one」）。

**状态机**：

```
REGISTERED → FINISHED_APPLYING → 不是队列头 → REQUESTED_GRANT → WAITED → FINISHED
                                  ↑ 等前面 worker 先 commit ↑
                        是队列头 → 直接提交（跳过 REQUESTED_GRANT）
```

### 生命周期与开关

**开关 gate**：manager 实例只在 `opt_replica_preserve_commit_order && !is_parallel_exec() && opt_replica_parallel_workers > 1` 时 `new` 出来，否则 `commit_order_mngr = nullptr`；所有入口第一步 `has_commit_order_manager(thd)`——开关关闭或非 MTS 线程时整条路径**统一退化为 no-op**。单线程回放（workers=0）本身天然保序，不需要 manager。

**入队**：coordinator 派单时 `register_trx(worker)`（`handle_slave_worker` 路径）——注意**派单即入队**，不是执行完才入队。队列顺序 = 分发顺序 = relay log 顺序。

**出队**：worker 提交完成后经 `wait_and_finish` 出队放行后继（见下）。

### 两条提交路径：等待发生在哪

根据从库是否写 binlog，保序等待发生在**不同位置**：

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

**Stage#0 为什么必须在 flush 之前等**（`sql/binlog.cc` 的 `ordered_commit`）：

```cpp
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

`ending_trans(thd, all=true)` 恒为 `true`，所以从库 MTS worker 的 **DML 也都会在 Stage#0 调 `wait()`**，不只 DDL。若不在 flush 前等：

```
无 Stage#0（错误）:
  W1: 执行T1(慢) ──────────────────→ FLUSH(T1) → SYNC → COMMIT
  W2: 执行T2(快) → FLUSH(T2) → SYNC → COMMIT
       ↑ 先写入 binlog!  → 从库自己的 binlog 里 T2 排在 T1 前 → 顺序错误（级联复制污染下游）

有 Stage#0（正确）:
  W1: 执行T1(慢) ──→ wait()通过 → FLUSH(T1) → ...
  W2: 执行T2(快) → wait()阻塞 ────────────→ FLUSH(T2) → ...
```

**`wait()` 的状态机：第一次真等，第二次 no-op**：

```cpp
bool Commit_order_manager::wait(Slave_worker *worker) {
  // ★ 只有 REGISTERED 状态才真正等待
  if (this->m_workers[worker->id].m_stage ==
      cs::apply::Commit_order_queue::enum_worker_stage::REGISTERED) {
    if (this->wait_on_graph(worker))       // 在 MDL graph 上等待
      return true;                         //   直到前驱释放
    return rollback_status;                // wait_on_graph 返回后状态已变 WAITED
  }
  return false;  // ★ 已经是 WAITED 状态 → 直接返回，no-op
}
```

- 第一次 `wait()`（Stage#0 或 ha_commit_low 入口）：状态 = REGISTERED → 真正阻塞 → 醒来变 WAITED；
- 第二次 `wait()`（写 binlog 路径中 ha_commit_low 内）：状态 ≠ REGISTERED → no-op。两次调用由同一状态位去重，两条路径共用一套状态机。

### 等待与放行：wait_on_graph / finish_one / wait_and_finish

**wait_on_graph：队首判断与阻塞**

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

**finish_one：队首提交后放行下一个**

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

序列号协议（头文件注释明说）：**持有序列号 N 的 worker 只能解封 N+1 的 worker**；N 正在执行解封操作（freeze 中）时，N+1 不能被分配新序列号——这正是 freeze/unfreeze 的用途：防"前任已置 GRANTED 后，继任者又被 coordinator 重新注册拿到新号"的双唤醒竞态。

**wait_and_finish：error 参数的两种退出路径**

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

### 同步原语：复用 MDL 的 wait-for 图

`Commit_order_manager` 不新建任何同步原语，而是让"等提交顺序"成为 **MDL wait-for 图的第四种子图**（`MDL_wait_for_subgraph` 的第三个生产子类，另两个是 `MDL_ticket` 和 `Wait_for_flush`）：

```cpp
class MDL_wait_for_subgraph {
 public:
  virtual bool accept_visitor(MDL_wait_for_graph_visitor *gvisitor) = 0;
  virtual uint get_deadlock_weight() const = 0;

  static const uint DEADLOCK_WEIGHT_CO = 0;    // commit order
  static const uint DEADLOCK_WEIGHT_DML = 25;
  static const uint DEADLOCK_WEIGHT_ULL = 50;
  static const uint DEADLOCK_WEIGHT_DDL = 100;
};

class Commit_order_lock_graph : public MDL_wait_for_subgraph {
  // 不是 {table, schema, tablespace, ...} 锁
  // 而是 "我在提交队列等你先走" 的同步令牌
 private:
  MDL_context &m_ctx;              // 我的 MDL 上下文
  Commit_order_manager &m_mngr;    // 提交管理器
  uint32 m_worker_id{0};           // 我的 Worker ID
};
```

它不对应任何真实数据库对象——不锁表、不锁行、不锁元数据，只是一个"等号牌"。接入 MDL 图的完整链路（五步）：

1. **`will_wait_for` 把"我在等提交顺序"注册成图里一条真边**（`sql/mdl.h` 内联）：先 `materialize_fast_path_locks()`（不物化 fast path ticket 则图不完整、漏检死锁），再写 `m_waiting_for = waiting_for_arg`。`m_waiting_for` 是 `MDL_wait_for_subgraph *`——同一根指针同时挂着"等 MDL 锁"和"等提交顺序"，两种等待天然在同一张图里。`m_LOCK_waiting_for` 是 prlock（读多写少：死锁检测线程大量并发读别人的边，只有本线程等/摘边时才写）。
2. **`visit_subgraph` 是检测器进入"我在等的那个东西"的唯一入口**：虚调用 `m_waiting_for->accept_visitor(gvisitor)` → `Commit_order_lock_graph::accept_visitor` → `m_mngr.visit_lock_graph()` 沿提交队列**向前**迭代（只看排在自己前面的），对每个前驱 `inspect_edge` 或递归下钻。
3. **检测器三件套**（`Deadlock_detection_visitor`，`sql/mdl.cc`）：`inspect_edge` 里 `m_found_deadlock = (node == m_start_node)`（回到起点即环）；`enter_node` 里搜索深度 ≥ `MAX_SEARCH_DEPTH=32` 直接判死锁（防栈溢出）；`opt_change_victim_to` 选**权重最小**者（`>=` 才换，平局保留先遇到）。
4. **victim 权重**：`get_deadlock_weight()` 对 commit order 节点返回 **0**（全图最低）⇒ 只要环上出现 worker 在等 turn，它**必然**当选 victim——"commit 等待永远优先被牺牲"，因为事务可自动重试，代价最小。
5. **`find_deadlock` 的 while 循环**：victim 被 `set_status(VICTIM)` 点名后（写等待槽 + signal），victim 的 `timed_wait` 返回 VICTIM → 报 `ER_LOCK_DEADLOCK` → worker 层 `retry_transaction` 重试。**MDL 只负责"点名"，回滚动作完全在 worker 层**。拆掉的是别人的边，可能还有别的环，必须循环搜索直到无环或自己成为 victim。

**跨三张图的死锁**：MySQL 里"事务等事务"有三套图——MDL 锁图（server 层）、InnoDB 行锁图（引擎层）、commit order 队列图（复制层）。前两者互不通信；而 commit order 图**通过 `MDL_wait_for_subgraph` 接入 MDL 图**，同时 InnoDB 侧通过 `thd_report_lock_wait` 钩子把行锁等待"上报"进 commit order 检测：

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
```

逻辑：W2 在 InnoDB 里等 W1 的行锁，而 W1 在 commit 顺序上排在 W2 **前面**（W1 若在等 W2 提交就成环）⇒ **牺牲"排在我后面"的 wait_for_w**——它拿着行锁却又等不了 W1 提交，必须回滚重试。`report_deadlock` 直接戳对方等待槽 `set_status(VICTIM)`（复用同一个等待槽，不走图遍历）。

### 设计决策一览

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

---

## 演进与历史

| 阶段 | 变化 | 背景 |
|------|------|------|
| 5.7 早期 | 参数已存在，默认 OFF | MTS 无保序时 worker 任意顺序提交 |
| 5.7.19 | 修复并发提交乱序 bug | 5.7.18 无法保证提交顺序一致，生产用保序至少 5.7.19+ |
| 8.0 | **默认翻转为 ON** | 与 RBR（默认）+ WRITESET 组合，兜住假阴性 |
| 8.0.23 / 5.7.33 | commit-order 等待从纯 mutex+cond 改为接入 MDL wait-for 图 | 修复 Bug#87796（跨层死锁不可检测 + giving up 信号丢失） |
| 8.0.28 | 序号生成器改用 64 位 + 正确回绕 | 修复 Bug#103636（int 溢出） |
| 8.0.34+（2024 重构） | lock-free 重构：`cs::apply::Commit_order_queue`（Node 数组 + 无锁 FIFO + freeze CAS） | 本库 8.0.39 基线形态 |

**2017 年 Taobao MySQL Monthly 文章的时代背景**：那篇文章（讨论 MTS 下 GTID 空洞）写作于 5.7 时代，`replica_preserve_commit_order` 尚未成为默认。没有保序时：

```
Coordinator 按 binlog 顺序分发：
  T1(seq=1, GTID=uuid:1) → Worker-1
  T2(seq=2, GTID=uuid:2) → Worker-2

Worker-1 执行 T1（大事务，10秒）
Worker-2 执行 T2（小事务，0.5秒）→ 先完成 → 立即提交 T2
  → gtid_executed = {uuid:2}   ← 空洞！T1(uuid:1) 还没提交
```

这是**临时性空洞**（最终会被最慢的 worker 填上），但它带来两个问题：① 任意时刻 `SELECT @@gtid_executed` 可能看到不连续集合（监控/备份/故障转移取位点的困扰）；② `is_new_group` 检测（`sequence_number != last_seq+1`）会把"空洞的填坑者"误判为新组而串行化。8.0 默认 ON 后，**提交顺序严格等于分发顺序，GTID 空洞不会因并行复制产生**。

---

## 缺陷档案

### Bug#87796：giving up 后信号丢失，三方互等挂死 12 小时

> [Bug #87796](https://bugs.mysql.com/bug.php?id=87796)（S1 Critical，2017-09 报告于 5.7.19；确认修复 **8.0.23** / **5.7.33**；相关重复：#89247、#95249——stop replica 永久阻塞、#99440）。下方事故时间线来自一例 8.0.22 生产事故，与 bug 机制完全吻合。

事故因果链（七环）：

1. **前兆**：并行回放冲突计数跳变 5~8 倍——WRITESET 把不同行的并发事务标记为可并行，从库回放冲突开始积压。
2. **主库死锁 dump 证实写入模式**：并发 `REPLACE INTO` 同一唯一索引，X-gap 锁 ⇄ insert-intention 互等（gap 锁挡插入意向）。这些"不同行"的事务 writeset 不交集 → `lc` 相同 → 从库 MTS 并行回放。
3. **从库跨层死锁**：前序 worker 持行锁后停在 "Waiting for preceding transaction to commit"（commit-order 等待，**不进 InnoDB 锁等待图**）；后序 worker 要的行锁被前序持有（进 InnoDB 锁图）。InnoDB 死锁检测只见行锁边、不见 commit-order 边 → **环不可见** → 只能等 `innodb_lock_wait_timeout` 超时。
4. **超时重试循环**：后序 worker 每轮报 1205 → `retry_transaction` 重试同一事务，8 轮无进展。
5. **重试耗尽**：`slave_transaction_retries`（默认 10）耗尽 → "in vain, giving up" → fatal_error → worker 退出流程。
6. **★ 致命点（本 bug 核心）**：8.0.22 的 giving up 路径**直接退出，没有向 SPCO 队列的后序 worker 发信号**——队列位置被跳过，后序 worker 永远等不到自己的 turn。
7. **三方互等卡死**：剩余 worker 卡 "Waiting for preceding transaction to commit"（等一个永不 commit 的前序）；Coordinator 卡 "Waiting for workers to exit"（等 worker 退出，`stop_wait_timeout` 以天计）；`Replica_SQL_Running` 表面仍 = Yes（停止流程未走完）、SBM 增长至 43500s——零日志、零自愈可能。

**8.0.22 与 8.0.39 的两处机制差异**（即 bug 的成因与修复面）：

| | 8.0.22（受影响） | 8.0.39（修复后） |
|---|---|---|
| commit-order 等待的实现 | mutex + cond 等待（`wait_for_its_turn`），**不在任何死锁图里** → 跨层环不可检测，只能靠超时 | `MDL_wait_for_subgraph`（`wait_on_graph`）接入 MDL 图，且 InnoDB 行锁等待经 `thd_report_lock_wait` 上报 → **跨层环可查**（`check_and_report_deadlock` 主动选 victim） |
| giving up 的路径 | 直接退出，不通知后继 → **信号丢失** | `wait_and_finish(error=true)`：先 `check_and_report_end_of_retries`；不可重试才**等 turn**、置 `m_rollback_trx`、出队 → 后继统一让位 + `ER_REPLICA_WORKER_STOPPED_PREVIOUS_THD_ERROR`（**"wait for its turn to call the rollback function"，队列位置一个不漏释放**） |

**三方互等为什么"零超时"**：Coordinator 卡 "Waiting for workers to exit" 是 `terminate_slave_thread` 在等 worker 退出，其超时上限是 `rpl_stop_replica_timeout`——**默认 `LONG_TIMEOUT`（365 天）**，且超时后也只是"返回警告"而非强制终止。所以在默认配置下，三方互等会挂到天荒地老，没有任何自愈出口。`KILL` worker 线程同样无效（worker 卡在行锁等待/cond 等待里，KILL 只置标志位，等不到被唤醒的时机）。

### Bug#103636：commit 序号 int 溢出，后继 worker 永不唤醒（"MTS 定时炸弹"）

> [Bug #103636](https://bugs.mysql.com/bug.php?id=103636)（S3，2021-05 报告于 8.0.24，确认 8.0.26 也受影响；**修复于 8.0.28**，changelog："The commit order sequence ticket generator now wraps around correctly"）。

**根因**（报告者 gdb 还原）：旧实现的 `finish_one()` 里 `auto this_seq_nr{0}` 被编译器推断为 **4 字节 int**。高负载（`WRITESET` + 64 workers + `replica_preserve_commit_order=1`）连续跑约 24 小时，提交序号超过 2^31（21.4 亿）后**溢出为负数**：

```
int 溢出为负
  → next_seq_nr = this_seq_nr + 1 仍是负
  → freeze_commit_sequence_nr(next_seq_nr) 把负数隐式转成 unsigned 巨大值
  → 冻结失败、后继 worker 永远不被解封
  → 从库 applier 永久挂起
```

**与 8.0.39 实现的对应**：本库剖析的 `cs::apply::Commit_order_queue` 正是 8.0.28 修复后的形态——序号类型换成专用的 `sequence_type`（64 位）+ 正确的回绕逻辑 + `Node::NO_SEQUENCE_NR` 哨兵。老版本代码里的裸 `auto`/`int` 序号已经消失。**教训**：序号生成器是高并发系统的计数器，类型宽度要按"24h × 每秒提交数"估算——`WRITESET` + 64 worker 一天就能跑 20 亿。

### 版本结论

| 版本段 | 缺陷 | 症状 | 修复 |
|--------|------|------|------|
| 5.7.19 ~ 8.0.22 | #87796：giving up 后不通知后继（信号丢失） | 三方互等挂死、`SQL_Running` 表面 Yes | 8.0.23 / 5.7.33 |
| 8.0.24 ~ 8.0.27 | #103636：commit 序号 int 溢出 | 后继 worker 永不唤醒、applier 挂起 | 8.0.28 |
| 全 5.7/8.0 | #89247、#95249、#99440（#87796 的相关重复）：`stop replica` 永久阻塞、worker 随机卡住 | stop/kill 均无效，只能 kill -9 | 随 #87796 |

★ **`replica_preserve_commit_order=ON` + 高负载并行回放的部署，8.0.28 以下都不可靠**（8.0.23~8.0.27 修了 #87796 但还有 #103636）。生产建议 8.0.28+（8.0.39 本库基线还叠加了 2024 年的 lock-free 重构）。

---

## 可观测性

### 参数矩阵

| 参数 | 作用 | 默认 |
|------|------|------|
| `replica_preserve_commit_order` | 从库是否保证提交顺序与主库一致 | ON (8.0) |
| `replica_parallel_workers` | worker 数（≤1 时 manager 不存在，保序天然成立） | 4 |
| `binlog_order_commits`（主库） | 主库引擎提交序是否等于 binlog 序（与从库保序互不依赖） | ON |

参数定义要点（`sql/sys_vars.cc`）：`NOT_IN_BINLOG`（不写进 binlog，纯从库参数）、`ON_CHECK(check_slave_stopped)`（须在复制停止时修改）、`PERSIST_AS_READONLY`。

### 性能特征

保序串行化的只是**提交**（binlog 写入 + InnoDB commit），不串行 DML 执行：

```
ROW event apply（并行）:
  ① 定位行
  ② trx_undo_report_row_operation() → 写 undo
  ③ btr_cur_update_in_place() → 更新索引
  ④ MVCC: read_view 更新

ht->commit() 做的事（轻量，串行）:
  ① trx_write_serialisation_history() → 给已有 undo 打 trx no
  ② mtr_commit() → 产生 commit_lsn
  ③ lock_trx_release_locks() → 释放锁
```

**undo 在 apply 阶段实时写入**，与 commit order 无关——保序的代价只有"快事务提前完成时空等慢事务"的空闲等待。量化（`log_replica_updates=ON`）：

```
Worker-1 (T1, seq=1):
  执行events(500ms) → ordered_commit() → Stage#0 wait(0μs,是第一个)
  → FLUSH → SYNC → COMMIT(ht->commit(1ms) → wait_and_finish 释放W2)

Worker-2 (T2, seq=2):
  执行events(200ms) → ordered_commit() → Stage#0 wait(300ms,被W1阻塞)
  → FLUSH → SYNC → COMMIT

总耗时 = max(500,200) + 300(等待) + 2×1 = 802ms
单线程 = 702ms
```

`wait(300ms)` 是**空闲等待**——W2 执行完了但必须等 W1 也执行完并通过 Stage#0。若 W1/W2 执行耗时接近，等待 ≈ 0；执行越不均衡（大小事务混杂），保序的串行化损失越接近"整组按最慢事务对齐"。

### 生产实践要点

- **版本底线 5.7.19+ / 8.0.28+**：5.7.18 并发提交乱序 bug、5.7.19 才修复；8.0 还要叠加 #87796（8.0.23）与 #103636（8.0.28）的修复面，高负载 MTS 生产建议 8.0.28+。
- **`replica_parallel_workers=1` 比 0 还慢约 20%**：设 1 时 SQL thread 变成 coordinator，但只有 1 个 worker——多一次"coordinator 派发 → worker 执行"的转发与队列开销（此时 manager 其实不存在，但 MTS 骨架开销仍在）。要么 0，要么 ≥2。
- **WRITESET + 高并发写同一唯一索引是温床**：writeset 不交集 ≠ 从库回放无锁竞争，配保序 ON 时尤其注意（#87796 的触发模式）。
- **主库低负载时 MTS 退化**：组提交合并程度低时每组可能只有 1 个事务，从库并行度 = 主库并发度；保序的空闲等待会进一步放大"慢事务拖累"效应。

### 故障指纹

| 症状 | 指向 |
|------|------|
| `stop replica` 长期不返回 + SBM 持续增长 + **零错误日志** | #87796 类三方互等（老版本），只能 kill -9 + 升级 |
| 并行回放冲突计数跳变 5~8 倍 | #87796 前兆（WRITESET 温床模式） |
| worker 卡 "Waiting for preceding transaction to commit" 且前序已退出 | giving up 信号丢失 |
| 24h 级高负载后 applier 突然挂起、无任何报错 | #103636 序号溢出（8.0.24~8.0.27） |
| 错误日志 `ER_REPLICA_WORKER_STOPPED_PREVIOUS_THD_ERROR` | 正常错误传播：某 worker 终局失败，后继连带停止——**先修主 worker 错误** |

---

## Misc

### 容易误读的点

- **"as on the source"指的是 binlog 中的事务顺序**，不是由主库来强制执行。整个 `Commit_order_manager` 的 FIFO 队列、`Commit_order_lock_graph` 等待机制、MDL 同步原语全都在从库进程内运行，主库甚至不知道从库有这个参数。
- **MDL 是从库本地 MDL，不是主库 MDL**。`Commit_order_lock_graph` 绑定的是从库 worker 线程自己的 `MDL_context`。主从是两个独立进程，不可能共享 MDL 图——这里只是**借用 MDL 的 wait/grant 基础设施做线程间同步**。
- **保序不靠 binlog 的 sequence_number 排序**，靠 FIFO 队列位置；`commit_sequence_nr` 是"证明你是前驱"的凭证，不是"排第几"的依据（见「架构总览」）。
- **保序不检测因果**。SELECT→UPDATE 例子里 T1 改 `t1`、T2 改 `t2`，没有 MDL 冲突——保序照样让 T2 等 T1，因为它只用"取号顺序"说话。
- **8.0.39 的真实命名核对**（网络文章常引用旧符号，均已不存在）：`Commit_order_arc`（等价物是 `Commit_order_lock_graph`）、`m_workers_map`/`m_granted`/`m_granted_workers`（等价物是 `cs::apply::Commit_order_queue`）、`wait_for_prior_commit()`（真实函数是私有 `wait_on_graph(Slave_worker*)`）、`assign_ticket()`（commit order 侧的"取号"在 `Commit_order_queue::push()`；同名函数属于 `Binlog_group_commit_ctx`，供 BGC 用）、`get_rollbacker()`（8.0.39 改为直接 `MDL_wait::set_status(VICTIM)`）、`signal_waiting()`（唤醒靠 `set_status` 内部的 `mysql_cond_signal`）。

### 与其他机制的关系

- **与 MTS 调度正交**（[`prpl.md`](prpl.md)）：`last_committed`/`sequence_number`/GAQ 决定"谁和谁可以并行执行"，保序决定"执行完的提交以什么顺序落盘"。GAQ/LWM 的推进、relay log recovery 的组标记都不依赖保序——保序关闭时 MTS 照常工作，只是提交乱序。
- **与 GTID 的关系**：保序消除"并行复制造成的临时 GTID 空洞"，但管不了"事务缺失"类空洞（多源交错、跳过事务）。
- **与主库 BGC 的关系**（[`binlog.md`](binlog.md)）：三段接力（见「理论基础」），主库 `binlog_order_commits` 与从库本参数互不依赖。

---

## 参考

**源码**

- `sql/rpl_replica_commit_order_manager.{h,cc}`——Commit_order_manager / Commit_order_lock_graph / cs::apply::Commit_order_queue
- `sql/mdl.{h,cc}`——MDL_wait_for_subgraph、will_wait_for、Deadlock_detection_visitor、find_deadlock
- `sql/binlog.cc`——ordered_commit Stage#0（`wait_for_its_turn_before_flush_stage`）
- `sql/handler.cc`——ha_commit_low 的 wait / wait_and_finish
- `sql/rpl_replica.cc` / `sql/rpl_rli_pdb.cc`——register_trx 调用点（派单）、thd_report_lock_wait 上报

**Bug / 文章**

- [Bug #87796](https://bugs.mysql.com/bug.php?id=87796)：giving up 信号丢失 + 跨层死锁不可检测（修复 8.0.23 / 5.7.33）
- [Bug #103636](https://bugs.mysql.com/bug.php?id=103636)：commit 序号 int 溢出（修复 8.0.28）
- 2017 Taobao MySQL Monthly（MTS 与 GTID 空洞）：5.7 时代无保序默认的上下文（见「演进与历史」）

**相关文档**

- [`prpl.md`](prpl.md)——并行复制（MTS）总览：依赖追踪、GAQ、recovery
- [`binlog.md`](binlog.md)——主库 binlog 组提交、`binlog_order_commits`
- [`replica.md`](replica.md)——从库 IO/SQL 线程与事件分发
- [`../../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)——MDL 死锁检测
