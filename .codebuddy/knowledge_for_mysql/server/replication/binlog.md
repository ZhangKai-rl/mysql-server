# MySQL Binlog 机制深度解析

> 基于 MySQL 8.0.39 源码，涵盖 Ordered Commit 三阶段流水线、BGC Ticket 系统、内部 2PC、binlog_order_commits 参数、commit 阶段与 trx->no、Anonymous_Gtid、Event 格式与 mysqlbinlog 解读、Row Image 记录过程。
>
> **边界**：本篇讲 binlog 侧机制（BGC 流水线、2PC 协调、事件格式）；2PC 的引擎侧执行（trx prepare 逐行剖析、undo 状态、外部 XA）与事务生命周期见 [`../../innodb/trx.md`](../../innodb/trx.md)。

## 目录

- [Ordered Commit 三阶段流水线](#ordered-commit-三阶段流水线)
  - [提交流程全景图](#提交流程全景图)
- [BGC Ticket 系统](#bgc-ticket-系统)
- [内部 2PC（两阶段提交）](#内部-2pc两阶段提交)
- [binlog_order_commits 参数](#binlog_order_commits-参数)
- [Commit 阶段与 trx_no](#commit-阶段与-trx_no)
- [Anonymous_Gtid](#anonymous_gtid)
- [Event 格式与 mysqlbinlog 解读](#event-格式与-mysqlbinlog-解读)
- [Row Image 记录过程](#row-image-记录过程)
- [参考](#参考)

---

## Ordered Commit 三阶段流水线

### 提交流程全景图

下图是事务从发起 COMMIT 到完成的完整流程（对应 `ha_commit_trans` → `ordered_commit`），按准备/FLUSH/SYNC/COMMIT 四段着色，并标注崩溃恢复分界：

```mermaid
flowchart TD
    Start([用户线程发起 COMMIT]) --> P1

    subgraph S0[" 准备阶段（进入 ordered_commit 前） "]
        P1["① 获取 MDL COMMIT 锁（防 FTWRL）<br/>handler.cc:1751"]
        P2["② 引擎 prepare：InnoDB undo 标记 PREPARED + 写 redo<br/>（HA_IGNORE_DURABILITY，暂不 fsync）<br/>trx_prepare_low · trx0trx.cc:2976"]
        P3["③ binlog cache finalize，追加 Xid_log_event<br/>binlog.cc:8243"]
        P1 --> P2 --> P3
    end

    subgraph S1[" FLUSH 阶段（Leader 持 LOCK_log） "]
        F1["④ 加入 flush 队列（同组事务排队）"]
        F2["⑤ Leader 获取 LOCK_log"]
        F3["⑥ fetch 整个 flush 队列 = group 快照<br/>fetch_and_process_flush_stage_queue · binlog.cc:8412"]
        F4["⑦ ha_flush_logs：InnoDB redo 批量 fsync<br/>innobase_flush_logs → log_buffer_flush_to_disk"]
        F5["⑧ 分配 sequence_number（MTS 逻辑时钟）<br/>binlog_cache_data::flush · binlog.cc:2419"]
        F6["⑨ binlog cache 顺序 write → binlog 文件（仅到 OS cache）<br/>process_flush_stage_queue · binlog.cc:8456"]
        F1 --> F2 --> F3 --> F4 --> F5 --> F6
    end

    subgraph S2[" SYNC 阶段（Leader 持 LOCK_sync） "]
        Y1["⑩ 加入 sync 队列，释放 LOCK_log"]
        Y2["⑪ Leader 获取 LOCK_sync，fetch sync 队列"]
        Y3["⑫ sync_binlog_file → fsync(binlog)<br/>★ 提交点（崩溃恢复分界线）<br/>binlog.cc:8659"]
        Y1 --> Y2 --> Y3
    end

    subgraph S3[" COMMIT 阶段（Leader 持 LOCK_commit） "]
        C1["⑬ 加入 commit 队列，释放 LOCK_sync"]
        C2["⑭ Leader 获取 LOCK_commit"]
        C3["⑮ InnoDB 引擎层 commit：undo COMMITTED、分配 trx->no、<br/>释放行锁、更新 MVCC<br/>trx_commit_in_memory"]
        C4["⑯ Leader 有序批量写 gtid_executed<br/>update_commit_group · binlog.cc:8562"]
        C5["⑰ 释放 LOCK_commit"]
        C1 --> C2 --> C3 --> C4 --> C5
    end

    P3 --> F1
    F6 --> Y1
    Y3 --> C1

    C5 --> POST["⑱ after_commit hook（半同步 ACK 时机）<br/>回复客户端 OK"]
    POST --> Done([提交完成])

    Y3 -.->|"此前崩溃 → 恢复 ROLLBACK"| RB["Xid_log_event 未落盘"]
    Y3 -.->|"此后崩溃 → 恢复 COMMIT"| CM["Xid_log_event 已落盘"]

    classDef prep fill:#d6eaf8,stroke:#2e86c1,color:#1b4f72
    classDef flush fill:#fadbd8,stroke:#c0392b,color:#7b241c
    classDef sync fill:#e8daef,stroke:#8e44ad,color:#4a235a
    classDef commit fill:#fcf3cf,stroke:#b7950b,color:#7d6608
    classDef point fill:#f5b7b1,stroke:#c0392b,stroke-width:3px,color:#7b241c
    class P1,P2,P3 prep
    class F1,F2,F3,F4,F5,F6 flush
    class Y1,Y2 sync
    class Y3 point
    class C1,C2,C3,C4,C5 commit
```

> 关于右下"写盘层次"：⑨ 的 `write` 只把 binlog cache 写入 **OS page cache**（sync_binlog=1 时此处仍可能因崩溃丢失）；⑫ 的 `fsync` 才把数据刷到 **binlog 物理文件**，这才是持久化与崩溃恢复的分界。InnoDB 侧同理——⑦ 的 redo fsync 保证 prepare 记录落盘，但引擎自身的 commit（undo COMMITTED / trx->no）在 ⑮ 才发生，且不再强制 fsync（`trx->flush_log_later`）。

事务提交进入 `MYSQL_BIN_LOG::ordered_commit`（binlog.cc:8853）后，经历三个阶段的流水线。Leader 线程代表整个 group 穿越所有 stage，Follower 线程从进入 flush stage 起沉睡，直到 `signal_done` 才被唤醒。

### Flush Stage（持 LOCK_log）

Leader 串行处理 group 内每个事务的 binlog cache，将其写入 binlog 文件。这一阶段是提交的真正瓶颈所在——`process_flush_stage_queue`（binlog.cc:8456）串行 flush 每个事务的 binlog cache。

关键操作包括：调用 `ha_flush_logs(true)` 将整组事务的 InnoDB redo prepare 记录批量 fsync（redo 持久化），然后将 binlog cache（含 `Xid_log_event`）写入 binlog 文件。

崩溃注入点 `crash_commit_before_log`（binlog.cc:8867）位于写 binlog 之前。

### Sync Stage（持 LOCK_sync）

对 binlog 文件执行 `fsync`。`sync_binlog_file`（binlog.cc:8659）只是普通的 fsync 调用，没有 group 概念——group 的合并在 stage queue 层面完成，sync 阶段只是把整组事务的落盘合并成一次 fsync。

**这里是提交点（Point of No Return）**。崩溃注入点 `crash_commit_after_log`（binlog.cc:8964）位于 fsync 成功之后。在此之后崩溃，事务视为已提交。

### Commit Stage（持 LOCK_commit）

Leader 按 binlog 顺序逐个调用引擎层 commit（`finish_transaction_in_engines`），释放行锁、更新 MVCC 可见性。详见后文 [Commit 阶段与 trx_no](#commit-阶段与-trx_no)。

### 分组控制参数

| 参数 | 作用 |
|------|------|
| `binlog_group_commit_sync_delay` | 事务延迟提交的微秒数（最大 1 秒），增加组内事务数量，减少 sync 次数 |
| `binlog_group_commit_sync_no_delay_count` | 组内事务数达到此值时立即提交，无需等待 delay |

### Group 的形成

Group 的形成完全由 `LOCK_log` 和 stage queue 控制，与 ticket 无关。Leader 持有 `LOCK_log` 期间，新到达的事务进入 flush stage 队列成为 Follower。Leader 释放 `LOCK_log` 后，下一个事务成为新 group 的 Leader。

---

## BGC Ticket 系统

### Ticket 的基本机制

`Bgc_ticket_manager`（bgc_ticket_manager.h）维护两个单调递增的 ticket 值：`front_ticket`（当前正在放行的号）和 `back_ticket`（下次分配的号）。

三个核心操作用 ticket 原子变量的最高位（MSB）当锁位做 CAS 来串行化：

- `assign_session_to_ticket`：给事务分配当前 back_ticket（幂等，rpl_context.cc:239，若 `m_session_ticket.is_set` 则跳过）
- `push_new_ticket`：递增 back_ticket，开启新的一轮
- `pop_front_ticket`：推进 front_ticket 到下一个号

线程进入 flush stage 队列前，先调 `wait_for_ticket_turn`（rpl_commit_stage_manager.cc:214），**只有当自己的 ticket == front_ticket 时才被放行进队列**。

### 一个 group 内 ticket 一定相同

因为 `append_to` 先 `wait_for_ticket_turn`（只放行 ticket==front）再进队列，而 group 本质是 leader 用 `fetch_queue` 对队列做的一次快照。能同时待在队列里的成员都满足 ticket==front，所以 group 内所有成员的 ticket 必然等于同一个 front 值。

反过来不成立：ticket 相同不代表是同一个 group。在普通 BGC 中所有事务 ticket 都是 1，但系统仍按 `LOCK_log` + stage queue 切出多个不同的 group。

### 普通BGC中 ticket 不激活

`push_new_ticket` 只在 GR（Group Replication）视图变更和 debug 测试注入中调用。普通 BGC 中 `back_ticket = front_ticket = 1` 永不变化，`wait_for_ticket_turn` 直接短路通过（零开销存在）。Group 形成完全由 `LOCK_log` + stage queue 控制。

### Ticket 的目的

为 Group Replication（MGR）的**认证线程（certification）**与**binlog 提交流水线**之间提供可插入的有序屏障（barrier synchronization）。MGR 下事务先经过 Paxos（XCom）广播 + 认证才能提交，认证是异步进行的。视图变更（成员加入/退出）时，必须保证屏障之前的所有事务都提交完，之后的才开始。Ticket 的"发号-叫号"机制精确地实现了这个屏障。

### 工业界的类似实现

Ticket lock 算法最早由 Mellor-Crummey 和 Scott 在 1991 年 TOCS 论文中系统化提出。工业界最著名的实现是 Linux 内核的 ticket spinlock（2.6.25 引入），后来在 4.16 前后 ARM64 等架构转向 qspinlock（基于 MCS lock 的变体）。

需要强调的是，MySQL 的 `Bgc_ticket_manager` **不是自旋锁**，而是基于条件变量 `cond_timedwait` 阻塞的高层批次调度器。它只借了"发号-叫号、可按序放行、可插入屏障"的思想，用途和实现都与内核 ticket spinlock 完全不同。

### 关于代码中的 "fackbook" 注释

`bgc_ticket_manager.h:217-218` 的文字是用户手写学习笔记（引用了 CSDN 博客，且 "fackbook" 是 "facebook" 的笔误），**不是 MySQL 官方注释**。官方源码未声称 `Bgc_ticket_manager` 源自 Facebook 的 ticket lock，该关联是笔记作者的联想，无官方依据。

### MGR 与 XCom（简述）

MGR 完全开源（GPLv2），代码位于 `plugin/group_replication/`。上层 `src/` 负责认证、成员管理、恢复、流控；下层 `libmysqlgcs/` 封装组通信，其内部核心是 **XCom**——MySQL 自研的 Multi-Paxos 实现，主体在 `libmysqlgcs/src/bindings/xcom/xcom/xcom_base.cc`（307KB），负责在成员间对事务顺序达成一致（total order broadcast）。Ticket 屏障正是服务于 `src/` 认证线程与 binlog 提交之间的同步。

---

## 内部 2PC（两阶段提交）

### 为什么需要 2PC

一个事务的持久化数据分散在两个独立的日志系统：InnoDB redo/undo log（保证引擎内部原子性和持久性）和 binlog（保证复制和 PITR）。两者必须原子地一起提交，否则会导致主从不一致。

### 协调者与参与者

`tc_log` 是全局抽象基类指针，运行时指向具体实现：

- 开启 binlog 时，`tc_log = &mysql_bin_LOG`，binlog 文件充当事务协调日志
- 关闭 binlog 时，`tc_log` 指向 `TC_LOG_DUMMY`，退化为引擎自己 1PC

存储引擎若实现了 `handlerton::prepare` 回调，就是 2PC 参与者。事务注册参与者时，若引擎没有 prepare 方法，直接标记为 `no_2pc`（handler.cc:1333）。

### 2PC 的完整调用链

入口是 `ha_commit_trans`（handler.cc:1615）：

```
第一阶段 PREPARE（handler.cc:1772）:
  仅当 rw_ha_count > 1 且非 no_2pc
  tc_log->prepare(thd, all)
    → ha_prepare_low → 各引擎 prepare
    → InnoDB: 写 prepare 记录到 undo header

第二阶段 COMMIT（handler.cc:1788）:
  tc_log->commit(thd, all)
    → ordered_commit 三阶段流水线
    → flush: ha_flush_logs 持久化 redo prepare + 写 binlog(含 Xid_log_event)
    → sync:  fsync(binlog) ← 提交点
    → commit: ha_commit_low → InnoDB 引擎层真正 commit
```

### binlog 侧的核心环节：finalize cache 与结束事件

> 2PC 的引擎侧剖析（prepare 五层逐行、undo 状态、`HA_IGNORE_DURABILITY` 的协同、触发条件 `rw_ha_count > 1`、外部 XA 的 detach / by_xid）见 [`../../innodb/trx.md`](../../innodb/trx.md)「事务与 binlog：2PC」。本篇只讲 binlog 自己的环节。

`MYSQL_BIN_LOG::commit` 在进 `ordered_commit` 之前，先 finalize binlog cache：

| 动作 | 位置 | 说明 |
|------|------|------|
| 记录 MTS last_committed | `store_commit_parent` | 取当前 `max_committed_timestamp` 作为 commit parent |
| 追加结束事件 | binlog.cc:8243 | 普通 2PC → `Xid_log_event`；XA → `XA_prepare_log_event`；1PC → `Query_log_event("COMMIT")` |
| finalize cache | `cache_mngr->trx_cache.finalize` | 将结束事件写入 trx cache，cache 冻结不再追加 |
| before_commit hook | binlog.cc:8278 | 插件钩子（如半同步复制） |

### 提交点与崩溃恢复

提交点是 sync stage 里 **binlog fsync 成功那一刻**。携带 XID 的 `Xid_log_event` 一旦落盘，事务就算提交。

崩溃恢复入口 `binlog::Binlog_recovery::recover`（binlog.cc:7916）：

1. 扫描最后一个 binlog 文件，收集所有 `Xid_log_event` 中的 XID → 组成 commit_list
2. 让每个存储引擎 recover → InnoDB 扫描 undo，返回所有处于 PREPARED 状态的 XID 列表
3. 对每个 PREPARED 的 XID 做仲裁：在 commit_list 中 → COMMIT；不在 → ROLLBACK

| 崩溃时机 | InnoDB 状态 | binlog 状态 | 恢复决定 |
|---------|------------|-------------|---------|
| `crash_commit_after_prepare` | prepared | 无 XID | rollback |
| `crash_commit_before_log` | prepared | 无 XID | rollback |
| `crash_commit_after_log` | prepared | 有 XID（已 fsync） | commit |
| `crash_commit_after` | committed | 有 XID | 已提交，无需处理 |

### 内部 2PC 与外部 XA

MySQL 的"2PC"有两副面孔，共用同一套 prepare/commit 引擎接口：

- **内部 2PC**：对用户透明，普通 `COMMIT` 内核自动完成，目的是 binlog/redo 一致性。
- **外部 XA**：用户显式 `XA START` / `XA PREPARE` / `XA COMMIT`，跨多个 MySQL 实例，额外写 `XA_prepare_log_event`。引擎侧细节（detach、by_xid 接口、XA 返回码、崩溃恢复状态机）见 [`../../innodb/trx.md`](../../innodb/trx.md)「外部 XA：与内部 2PC 的差异」。

---

## binlog_order_commits 参数

### 参数定义

全局变量，默认 `true`（sys_vars.cc:1741）。控制 BGC 第三阶段 commit stage 里引擎层 commit 调用的顺序——具体是 `ha_commit_low`（`finish_transaction_in_engines`）这一步。

**它不影响 flush 和 sync 两个阶段**，binlog 写入顺序和 fsync 永远是有序的。它只控制"最后让 InnoDB 真正 commit"这一步是有序还是各自为政。

### ON（默认）

Leader 把整个 group 拉进 commit stage，按事务写入 binlog 的同一顺序，逐个调用引擎 commit。GTID 也由 leader 有序批量写入 `gtid_executed`（binlog.cc:9075 注释：这样 `gtid_executed` 始终是一个连续区间，避免产生临时空洞、避免频繁加锁 `Gtid_set`）。

### OFF

不进 commit stage，各线程回到 `finish_commit` 里各自调用 `finish_transaction_in_engines` 自己做引擎 commit——顺序不保证。`finish_commit` 里 8728 的注释明确点破这个语义。

### Clone 强制有序

即使 `opt_binlog_order_commits` 为 false，只要 `Clone_handler::need_commit_order()` 为真，仍强制走有序分支（binlog.cc:9054）。Clone、一致性备份依赖"引擎 commit 顺序 == binlog 顺序"。

### 为什么只有 commit 阶段有开关

三阶段本质不同：

- **flush 没得选**：写 binlog 文件字节流就是复制顺序，必须串行，一旦乱序就是复制正确性灾难。
- **sync 没得选**：对整个 binlog 文件做一次 fsync，BGC 的核心收益就是把整组落盘合并成一次 fsync。让多个线程各自 fsync 反而破坏了 group commit 的意义。
- **commit 有得选**：引擎层 commit（改自己事务状态、释放自己的行锁、写自己的 commit 标记）各事务相互独立，天然可并行。正因可并行，才出现"保持有序（leader 串行）还是放开并行（各线程自己干）"的权衡。

默认选有序，因为有序保证 GTID 连续性、支持 Clone/备份一致性、且整组一次加锁比每线程各加锁开销更小。关闭仅在极高并发下省 stage 切换 / `LOCK_commit` 争用，榨微弱吞吐，代价大不推荐。

### 开启 binlog 后都走 BGC

`binlog_order_commits` 和"是否走 BGC"是两件不同的事。只要开启 binlog，所有事务提交都进入 `ordered_commit` 的三阶段流水线，这就是 BGC 机制本身。BGC 不是可开关的，它是 binlog 提交的唯一路径。`binlog_order_commits` 只决定 BGC 内部 commit stage 是否保持有序。

---

## Commit 阶段与 trx_no

### commit order 的核心载体：trx->no

`binlog_order_commits` 控制的"引擎 commit 顺序"，本质就是内存态的提交顺序，核心体现是 `trx->no`（serialisation number）的分配顺序。

> `trx->no` 的分配细节（`trx_add_to_serialisation_list` / `trx_serialisation_number_get` 的 purge queue 优化）、`trx->id` vs `trx->no` 的对比、commit 阶段的内存收尾（undo COMMITTED / 放锁 / MVCC / GTID 落盘）见 [`../../innodb/trx.md`](../../innodb/trx.md)「事务提交」。commit 阶段可并发的理由见下文「为什么只有 commit 阶段有开关」。

`trx->no` 在 `finish_transaction_in_engines` 里、serialisation mutex 保护下按调用先后单调递增分配。谁先被调用 commit，谁的 `trx->no` 就更小。

`binlog_order_commits` ON → `trx->no` 分配顺序 == binlog 写入顺序；OFF → 顺序不保证。Clone、一致性备份、`START TRANSACTION WITH CONSISTENT SNAPSHOT` 依赖"引擎提交序 == binlog 序"来取一致快照。

---

## Anonymous_Gtid

### 与普通 Gtid 的区别

`Gtid_log_event`（type=33）和 `Anonymous_gtid_log_event`（type=34）共用同一个 C++ 类 `Gtid_log_event`，区别在于 `spec.type`：

| | Gtid_log_event | Anonymous_gtid_log_event |
|---|---|---|
| type code | 33 | 34 |
| 携带 SID:GNO | 有（全局唯一事务标识） | 无（`spec.set_anonymous()`，sid/gno 清空） |
| sequence_number/last_committed | 有 | 有（完全相同） |
| IGNORABLE 标志 | 无 | 有（`LOG_EVENT_IGNORABLE_F`） |
| gtid_next 类型 | `ASSIGNED_GTID` | `ANONYMOUS_GTID` |
| 生成条件 | gtid_mode = ON | gtid_mode = OFF |

区分逻辑在 `Gtid_log_event` 构造函数（log_event.cc:12996-13016）：根据 `gtid_next.type` 决定 event 类型和标志。

### 为什么引入 Anonymous_Gtid

5.7 引入逻辑时钟 MTS 后，需要每个事务前有一个统一的事务起始事件来携带 `sequence_number` / `last_committed`。在 gtid_mode=ON 时，`Gtid_log_event` 天然承担这个角色。但 gtid_mode=OFF 时不生成 Gtid_log_event，MTS 就没有地方放依赖信息。

解决方案是引入 `Anonymous_gtid_log_event`——即使 gtid_mode=OFF，也为每个事务写一个 Anonymous_Gtid 事件。它不带 GTID（匿名），但携带 sequence_number / last_committed，让 MTS 在不开 GTID 的情况下也能运行。同时标记 `LOG_EVENT_IGNORABLE_F`，表示对开 GTID 的从库来说这个事件可忽略。

log_event.cc:2935 的注释直接点明了这个必要性。

### 多重作用

1. **MTS 事务边界标记**：coordinator 据此识别事务边界、提取 sequence_number/last_committed 做依赖调度
2. **匿名 GTID 所有权管理**：从库收到时设置 `gtid_next=ANONYMOUS` 并获取匿名所有权（sql_class.h:3565）
3. **兼容 5.6 master**：5.6 master（gtid_mode=off）不生成任何 GTID 事件，从库在执行 Query_log_event 时通过 `gtid_reacquire_ownership_if_anonymous` 设置匿名所有权（rpl_rli_pdb.h:749）
4. **GTID 模式平滑升级**：OFF → OFF_PERMISSIVE → ON_PERMISSIVE → ON 的过渡期，Anonymous_Gtid 与 Gtid 混合存在，MTS 始终能工作

---

## Event 格式与 mysqlbinlog 解读

### Event 边界

mysqlbinlog 输出中，每个 event 由三部分组成：

```
# at <offset>                    ← event 起始：字节偏移
#<时间> server id <id>  end_log_pos <pos> CRC32 <crc>  <EventType> <详情>  ← event 头
<event 内容>                     ← event 体（SQL/SET 语句等）
```

`# at` 就是 event 的分隔符。遇到下一个 `# at` 意味着上一个 event 结束。两个 `# at` 之间的所有内容属于同一个 event。

### 关键字段

| 字段 | 含义 |
|------|------|
| `# at 157` | event 在 binlog 文件中的起始字节偏移 |
| `260703 15:17:04` | event 的时间戳 |
| `server id 1` | 生成该 event 的服务器 id |
| `end_log_pos 236` | event 结束位置 = 下一个 event 的起始位置（236-157=79 字节是 event 长度） |
| `CRC32 0x...` | 校验和 |
| `GTID` / `Query` / `Start` | event 类型 |
| `last_committed=0 sequence_number=1` | MTS 逻辑时钟依赖信息（仅 GTID event 有） |

### mysqlbinlog 显示名与内部类型对照

| mysqlbinlog 显示 | 内部 event 类型 | type code |
|---|---|---|
| `Start:` | `FORMAT_DESCRIPTION_EVENT` | 15 |
| `Previous-GTIDs` | `PREVIOUS_GTIDS_LOG_EVENT` | 35 |
| `GTID` | `GTID_LOG_EVENT` | 33 |
| `Anonymous_GTID` | `ANONYMOUS_GTID_LOG_EVENT` | 34 |
| `Query` | `QUERY_EVENT` | 2 |
| `Table_map` | `TABLE_MAP_EVENT` | 19 |
| `Update_rows` / `Write_rows` / `Delete_rows` | `ROWS_EVENT` | 23/24/25 |
| `Xid` | `XID_EVENT` | 16 |
| `Rotate` | `ROTATE_EVENT` | 4 |

### 常用参数

```bash
# 解码行事件，显示每个字段的具体值（调试最有用）
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog_file

# 显示十六进制 dump（调试 event 头部）
mysqlbinlog --hexdump binlog_file
```

### 一个事务的 event 组成

每个事务在 binlog 中由以下 event 序列组成：

```
Gtid_log_event (或 Anonymous_gtid_log_event)   ← 携带 GTID + sequence_number/last_committed
Query_log_event ("BEGIN")                       ← 事务开始（可选，autocommit DDL 可能没有）
... 行事件或语句事件 ...
Xid_log_event 或 Query_log_event ("COMMIT")    ← 事务结束
```

---

## Row Image 记录过程

> 基于 MySQL 8.0.39 源码，ROW format binlog 中 DML 操作如何转化为行镜像（before-image/after-image）写入 binlog 的完整流程。

### DML 触发入口

所有 DML 操作在存储引擎执行成功后，通过 handler 层的 `ha_write_row` / `ha_update_row` / `ha_delete_row` 调用 `binlog_log_row()` 进入 binlog 记录流程：

| DML 类型 | 前镜像 (BI) | 后镜像 (AI) | 入口 |
|----------|-------------|-------------|------|
| INSERT | 无 | 新插入行 | `ha_write_row` → `binlog_log_row(table, nullptr, buf, ...)` |
| UPDATE | 修改前行 | 修改后行 | `ha_update_row` → `binlog_log_row(table, old_data, new_data, ...)` |
| DELETE | 被删除行 | 无 | `ha_delete_row` → `binlog_log_row(table, buf, nullptr, ...)` |

### 核心调度：binlog_log_row()

`binlog_log_row()`（handler.cc:7860）完成以下工作：

1. 检查当前表是否需要行格式 binlog（`check_table_binlog_row_based`）
2. 如果启用了 `transaction_write_set_extraction`，计算主键等价（PKE）用于写集合提取
3. 若是语句的第一行，先写入 `Table_map_event`（`write_locked_table_maps`）
4. 调用对应的 `Log_func` 函数指针（`binlog_write_row` / `binlog_update_row` / `binlog_delete_row`）

### 列筛选：binlog_row_image 参数

在打包前，根据 `binlog_row_image` 参数决定哪些列参与记录：

| 取值 | read_set（前镜像） | write_set（后镜像） |
|------|-------------------|-------------------|
| **FULL**（默认） | 所有列 | 所有列 |
| **NOBLOB** | PK 列 + 非 BLOB 列 | 非 BLOB 列 |
| **MINIMAL** | 仅主键列 | 仅被修改的列 |

`mark_columns_per_binlog_row_image()`（table.cc:5682）在语句准备阶段设置 bitmap；`binlog_prepare_row_images()`（binlog.cc:11249）在实际打包前进一步调整 read_set。若无主键，read_set 全部置位（无法用主键唯一标识行）。

### 行数据打包：pack_row()

`pack_row()`（rpl_record.cc:232）将内部记录格式转换为 binlog 传输格式：

```
+-----------+----------+----------+     +----------+
| null_bits | column_1 | column_2 | ... | column_N |
+-----------+----------+----------+     +----------+
```

`null_bits` 占 `ceil(N/8)` 字节，N 为 image 中包含的列数。对于 `PARTIAL_UPDATE_ROWS_EVENT`（JSON 部分更新优化），布局还包含 `value_options` 和 `partial_bits`。

### 行镜像类型

```cpp
enum class enum_row_image_type { WRITE_AI, UPDATE_BI, UPDATE_AI, DELETE_BI };
```

- `WRITE_AI`：INSERT 后镜像，用 `table->write_set` 打包
- `UPDATE_BI`：UPDATE 前镜像，用 `table->read_set` 打包
- `UPDATE_AI`：UPDATE 后镜像，用 `table->write_set` 打包，支持 JSON 部分更新
- `DELETE_BI`：DELETE 前镜像，用 `table->read_set` 打包

### UPDATE 完整调用链（最复杂的情况）

```
1. handler::ha_update_row(old_data, new_data)
   └─ 2. binlog_log_row(table, old_data, new_data, Update_rows_log_event::binlog_row_logging_function)
       ├─ 3. check_table_binlog_row_based()
       ├─ 4. add_pke() — 写集合提取
       ├─ 5. write_locked_table_maps() — 写入 Table_map_event
       └─ 6. THD::binlog_update_row(table, is_trans, before_record, after_record, extra_row_info)
           ├─ 7. binlog_prepare_row_images(thd, table) — 调整 read_set
           ├─ 8. pack_row(table, table->read_set, before_row, before_record, UPDATE_BI)
           ├─ 9. pack_row(table, table->write_set, after_row, after_record, UPDATE_AI, value_options)
           ├─ 10. binlog_prepare_pending_rows_event<Update_rows_log_event>() — 获取/创建事件
           ├─ 11. ev->add_row_data(before_row, before_size)
           ├─ 12. ev->add_row_data(after_row, after_size)
           └─ 13. 恢复原 read_set/write_set
```

### Rows_event 二进制格式

```
Post-Header:
  table_id     (6 bytes)
  flags        (2 bytes)

Body:
  width                    (packed integer) — 表的列数
  cols                     (bitfield) — 前镜像列位图
  cols_ai                  (bitfield, UPDATE only) — 后镜像列位图
  extra_row_info           (optional)
  row data: 序列化的行数据
    对于每行: null_bits + column_values
    UPDATE 每行包含 BI 和 AI 两部分
```

---

## 参考

**官方文档**

- *MySQL 8.0 Reference Manual → Binary Log*
- *MySQL 8.0 Reference Manual → XA Transactions*

**相关文档**

- 2PC 引擎侧执行（prepare 五层逐行、undo 状态、外部 XA、崩溃恢复三幕）见 [`../../innodb/trx.md`](../../innodb/trx.md)
- GTID 的三条持久化路径见 [`gtid.md`](gtid.md)
- MTS 并行复制与 last_committed / sequence_number 见 [`replication.md`](replication.md)

