# InnoDB 事务（trx_t）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 trx_t 核心结构、事务状态机与生命周期、事务启动、事务提交（InnoDB 侧）、回滚机制、回滚与 kill 的关系（为何回滚不可中断）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [trx_t 核心结构](#trx_t-核心结构)
- [事务状态机与生命周期](#事务状态机与生命周期)
- [事务提交（InnoDB 侧）](#事务提交innodb-侧)
- [回滚机制](#回滚机制)
- [回滚与 kill：为何回滚不可中断](#回滚与-kill为何回滚不可中断)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

`trx_t` 是 InnoDB 中事务的内存表示。每个 MySQL 连接（`THD`）通过 `innodb_session_t`（ha_innodb.cc:2021）持有自己的 `trx_t` 对象，贯穿事务的整个生命周期——从 `TRX_STATE_NOT_STARTED` 到 `TRX_STATE_COMMITTED_IN_MEMORY`。

### 用途

1. **undo log 的组织单元**：一个事务的 undo 记录挂在它自己的 rollback segment（rseg）下，回滚时按 `undo_no` 逆序撤销。
2. **锁的所有者**：`trx->lock.trx_locks` 链表记录事务持有的所有行锁/表锁，提交或回滚时统一释放。
3. **MVCC 的载体**：`trx->id`（写进记录的 `DB_TRX_ID`）和 `trx->no`（提交序列号）决定行的可见性。
4. **2PC 参与者**：`trx->state == TRX_STATE_PREPARED` 时事务进入 XA/内部 2PC 的 prepare 态，等待协调者裁决。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.5 及更早 | `trx_t` 字段堆叠，状态管理较粗糙 |
| 5.6 | 引入 `trx->id` 与 `trx->no` 分离的清晰化；read view 管理独立 |
| 5.7 | 引入 `TRX_FORCE_ROLLBACK`（高优先级事务异步杀阻塞者）；redo 采用简化 MTR 接口 |
| 8.0 | `trx->state` 用 `std::atomic`；新增 `TRX_STATE_FORCED_ROLLBACK`；undo 表空间可截断；并行 DDL |

---

## 理论基础

### 设计模式

- **State Machine（状态机）**：`trx_state_t` 五态（NOT_STARTED / FORCED_ROLLBACK / ACTIVE / PREPARED / COMMITTED_IN_MEMORY），转换由 `trx_prepare` / `trx_commit_in_memory` / `trx_rollback` 驱动。
- **Undo Log（逆向执行）**：回滚本质是 undo log 的逆序重放，对应 ARIES 的 undo pass 思想。
- **RAII Guard（TrxInInnoDB）**：`TrxInInnoDB` 类用构造/析构管理"进入/退出 InnoDB 上下文"（`trx->in_innodb` 计数），保证异常路径也能正确退出。

### 相关论文

- **"ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging"**, Mohan et al., TODS 1992 — undo log + WAL 的经典。InnoDB 的 `trx_undo_t` / `trx_undo_set_state_at_prepare` / 崩溃恢复回滚都源自 ARIES 的 undo pass。

### 算法与数据结构

- **undo log 链表**：`trx->rsegs` 下的 `insert_undo` / `update_undo` 链表，按 `undo_no` 单调递增，回滚时逆序。
- **rw_trx_list / mysql_trx_list**：`trx_sys->rw_trx_list` 记录所有读写事务（purge 扫描对象），`rw_trx_ids` 用于 MVCC 可见性快照。
- **serialisation list**：提交时分配 `trx->no` 的单调序列号链表，决定 purge 顺序。

### 类似实现对比

| 项目 | 事务结构 | 与 InnoDB 的异同 |
|------|----------|------------------|
| PostgreSQL | `PGPROC` + XID（32 位，回绕问题） | InnoDB trx_id 64 位无回绕；PG 的 XID 回绕需 vacuum freeze |
| Oracle | 事务槽（undo segment 直接绑定） | InnoDB rseg 与 Oracle undo segment 概念接近 |
| SQL Server | 事务日志 LSN 驱动 | InnoDB 提交顺序由 trx->no 而非 LSN 决定 |

### 历史背景

InnoDB 的事务模型从诞生起就围绕 undo log + MVCC 设计，核心思想稳定数十年。8.0 的主要演进是并发控制更细粒度化（sharded lock system、乐观 latching）和 undo 表空间可管理化，事务本身的状态机几乎没变。

---

## trx_t 核心结构

`trx_t` 定义在 `include/trx0trx.h`，关键字段：

```cpp
struct trx_t {
  trx_id_t id;                    // 事务开始分配（trx_start_low），写进行的 DB_TRX_ID
  trx_id_t no;                    // 提交时才分配（TRX_ID_MAX → 序列号），决定 purge/MVCC 顺序
  std::atomic<trx_state_t> state; // 状态机（见下节）
  trx_undo_ptr_t rsegs;           // 回滚段：insert_undo / update_undo
  lock_t *lock;                   // 持有的锁链表
  ReadView *read_view;            // MVCC 一致性视图
  XID *xid;                       // 2PC/XA 的 XID
  std::atomic<uint32_t> in_innodb; // 进入 InnoDB 上下文的标志位（含 TRX_FORCE_ROLLBACK 等）
  std::atomic<std::thread::id> killed_by; // 被哪个线程标记为异步回滚
  // ...
};
```

两个最易混淆的字段：

| 字段 | 分配时机 | 用途 |
|------|----------|------|
| `trx->id` | 事务**开始**（`trx_start_low`） | 标识"行由哪个事务修改"，写进 `DB_TRX_ID` |
| `trx->no` | 事务**提交**（`trx_sys_allocate_trx_no`） | 提交序列号，决定 purge 顺序 + MVCC 可见性 |

`trx->no` 在运行期恒为 `TRX_ID_MAX`（trx0trx.cc:1405），提交时才赋值。详见 binlog.md 的"Commit 阶段与 trx_no"。

---

## 事务状态机与生命周期

`trx_state_t` 定义在 `include/trx0types.h:83`：

```cpp
enum trx_state_t {
  TRX_STATE_NOT_STARTED,       // 未开始（新建或已回滚）
  TRX_STATE_FORCED_ROLLBACK,   // 与 NOT_STARTED 相同，但语义是"上次活跃时被异步回滚"
  TRX_STATE_ACTIVE,            // 活跃（执行 DML 中）
  TRX_STATE_PREPARED,          // 2PC/XA 已 prepare
  TRX_STATE_COMMITTED_IN_MEMORY // 内存中已提交（未持久化）
};
```

状态转换：

```
NOT_STARTED ──trx_start_if_not_started──> ACTIVE
ACTIVE ──trx_prepare（2PC）──> PREPARED
ACTIVE ──trx_commit_in_memory──> COMMITTED_IN_MEMORY ──> 释放后回 NOT_STARTED
ACTIVE ──trx_rollback（异步）──> FORCED_ROLLBACK ──> NOT_STARTED
PREPARED ──commit/rollback 裁决──> COMMITTED_IN_MEMORY / NOT_STARTED
```

关键点：

- **`TRX_STATE_FORCED_ROLLBACK` 是"标记"而非独立长驻态**：高优先级事务异步回滚一个活跃事务时，先把它标记为 FORCED_ROLLBACK，回滚完成后转回 NOT_STARTED。`innobase_rollback`（ha_innodb.cc:5966）里显式处理了 `state == TRX_STATE_FORCED_ROLLBACK` 的恢复。
- **状态字段是 `std::atomic`**：`trx->state` 用 `memory_order_relaxed` 读写，多线程（如异步 killer 与被 kill 线程）访问安全。

---

## 事务提交（InnoDB 侧）

InnoDB 侧提交的完整链路（server/binlog 层的协调见 binlog.md 的"内部 2PC"）：

```
ha_commit_low (handler.cc:1887)
  └─ innobase_commit (ha_innodb.cc:5762)
      └─ innobase_commit_low (ha_innodb.cc:5695)
          └─ trx_commit_for_mysql
              └─ trx_commit_low
                  └─ trx_commit_in_memory   // 纯内存态收尾
```

`trx_commit_in_memory` 做的事：

1. undo 标记 `TRX_UNDO_PREPARED → TRX_UNDO_COMMITTED`，移入 history list（供 purge）
2. 分配 `trx->no`（`trx_sys_allocate_trx_no`），入 serialisation list + purge queue
3. 持久化 GTID 到 undo header
4. 释放全部行锁（`trx_release_impl_and_expl_locks`）
5. 从 `rw_trx_list` / `rw_trx_ids` 移除，更新 MVCC 可见性

注意：**提交阶段没有磁盘 I/O**，持久化在 prepare（redo）+ binlog fsync 已完成。

---

## 回滚机制

### 回滚的触发场景

| 场景 | 入口 | 执行线程 |
|------|------|----------|
| 显式 `ROLLBACK` / `XA ROLLBACK` | `ha_rollback_trans` → `innobase_rollback` | 用户线程（前台） |
| 语句错误回滚（savepoint） | `innobase_rollback` 的 `rollback_trx=false` 分支 | 用户线程（前台） |
| **KILL QUERY / KILL CONNECTION** | `trx_is_interrupted` 检查点发现 killed → `ha_rollback_trans` | 用户线程（前台） |
| 高优先级事务杀阻塞者 | `trx_kill_blocking` → `trx_rollback_for_mysql` | **killer 线程（异步）** |
| 崩溃恢复 | `trx_recovery_rollback_thread` → `trx_rollback_active` | **后台线程** |

### 回滚的执行路径

**完整回滚**（rollback 到事务起点）：

```
ha_rollback_trans (handler.cc:2049)
  └─ innobase_rollback (ha_innodb.cc:5921)
      └─ trx_rollback_for_mysql (trx0roll.cc:266)
          └─ trx_rollback_low
              └─ trx_rollback_to_savepoint(trx, nullptr)  // savept=nullptr = 完整回滚
                  └─ trx_rollback_to_savepoint_low (trx0roll.cc:79)
                      └─ que_run_threads → row_undo_step    // 逐行逆序撤销
```

**语句级回滚**（savepoint 回滚）：

```
innobase_rollback (rollback_trx=false 分支, ha_innodb.cc:5981)
  └─ trx_rollback_last_sql_stat_for_mysql
      └─ trx_rollback_to_savepoint(trx, &trx->last_sql_stat_start)
```

**savepoint 命令**：

```
innobase_rollback_to_savepoint (ha_innodb.cc:6017)
  └─ trx_rollback_to_savepoint_for_mysql (trx0roll.cc:432)
```

### 回滚做了什么

`trx_rollback_to_savepoint_low`（trx0roll.cc:79）构造 roll query graph（`QUE_FORK_RECOVERY`），由 `row_undo_step` 逐行逆序执行 undo：

- 逐行读取 undo 记录，逆向恢复行的旧值（插入行→删除，更新行→恢复旧值，删除行→重新插入）
- undo 页修改写入 redo（WAL，崩溃可恢复）
- 回滚完成调 `trx_rollback_finish`：释放锁、从 rw_trx_list 移除、状态转 NOT_STARTED

**回滚复杂度与数据量成正比**，大事务回滚可能比正向执行更慢（undo 是随机访问旧页 + 二次 redo）。

---

## 回滚与 kill：为何回滚不可中断

### 结论

**回滚一旦开始，不能被 kill 中断。** kill 只能在回滚开始前生效（把事务"逼进"回滚），回滚跑起来后只能等它完成。

### 源码证据一：回滚循环无 kill 检查

`trx_rollback_to_savepoint_low`（trx0roll.cc:79）的回滚循环：

```cpp
112:    que_run_threads(thr);
113:    ut_a(roll_node->undo_thr != nullptr);
114:    que_run_threads(roll_node->undo_thr);   // ← 真正逐行撤销，无 trx_is_interrupted
```

真正逐行撤销的 `row0undo.cc`，搜 `killed` / `trx_is_interrupted` 结果是 **0 处**。

### 源码证据二：kill 检查只出现在正向执行路径

`trx_is_interrupted`（ha_innodb.cc:3152）：

```cpp
3152:bool trx_is_interrupted(const trx_t *trx) {
3154:  return (trx && trx->mysql_thd && thd_killed(trx->mysql_thd));
3155:}
```

它遍布正向 DML 的检查点——`row0mysql.cc:4669`、`row0sel.cc:4979`、`btr0btr.cc:4439`、`row0pread.cc:687` 等 30+ 处，用于让查询"随时可被打断"。**但回滚路径上它一次都不出现**。

### 源码证据三：point of no return

`TrxInInnoDB::enter`（trx0trx.h）以 `disable=true` 进入（COMMIT/ROLLBACK 方法）时设置 `TRX_FORCE_ROLLBACK_DISABLE`，注释：

> "This transaction has crossed the point of no return and cannot be rolled back asynchronously now. **It must commit or rollback synchronously.**"

回滚必须完整做完，否则 undo 半撤销，崩溃恢复无法重建一致性。

### kill 的作用时机

| 时机 | kill 能否生效 | 结果 |
|------|--------------|------|
| 事务还在执行 DML | ✅ | 下一个 `trx_is_interrupted` 检查点中断 → 触发回滚 |
| 事务在等待锁 | ✅ | `lock_cancel_if_waiting_and_release` 取消等待 → 看到 killed → 回滚 |
| **事务已在回滚中** | ❌ **无效** | 只设了 killed 标志，回滚照常跑完 |

### KILL 的完整调用链

**KILL 发起侧**（执行 `KILL` 的另一个线程）：

```
SQL: KILL [QUERY | CONNECTION] <id>
  → sql_kill (sql_parse.cc:6500)
    → kill_one_thread (sql_parse.cc:6436)
      → tmp->awake(KILL_QUERY 或 KILL_CONNECTION)   // 置 killed + 唤醒
      └─ (仅 KILL CONNECTION) ha_kill_connection (handler.cc:958)
          → innobase_kill_connection (ha_innodb.cc:6228)
            → lock_cancel_if_waiting_and_release      // 仅取消锁等待，不做回滚
```

**被 kill 线程侧**（发现自己被 kill → 回滚）：

```
被 kill 线程在 trx_is_interrupted 检查点发现 killed
  → 返回 DB_INTERRUPTED → 语句报 ER_QUERY_INTERRUPTED
    → ha_rollback_trans (handler.cc:2049)
      → innobase_rollback (ha_innodb.cc:5921)
        → trx_rollback_for_mysql (trx0roll.cc:266)
          → trx_rollback_to_savepoint (trx0roll.cc:142)
            → trx_rollback_to_savepoint_low (trx0roll.cc:79)
              → que_run_threads → row_undo_step
                  ★ 此循环不检查 killed，回滚不可中断
```

### KILL QUERY vs KILL CONNECTION

| | `KILL QUERY` | `KILL CONNECTION` |
|---|---|---|
| 目标 | 只中断当前语句 | 断开连接 |
| `only_kill_query` | true | false |
| killed 值 | `ER_QUERY_INTERRUPTED` | `ER_SERVER_SHUTDOWN` |
| 取消锁等待 | 否 | 是（`innobase_kill_connection` → `lock_cancel_if_waiting_and_release`） |
| 对回滚中事务 | 无效 | 无效（连接会先回滚完才真正断开） |

生产上常见：`KILL CONNECTION` 后连接还"挂着"一段时间，`SHOW PROCESSLIST` 的 State 显示 `ROLLBACK`，那正是在做不可中断的回滚。

### 异步回滚（TRX_FORCE_ROLLBACK）

区别于 kill，这是**高优先级事务**（`trx_is_high_priority`）主动杀死阻塞自己的普通事务的机制：

```
trx_kill_blocking (trx0trx.cc:3521)
  └─ lock_make_trx_hit_list                     // 收集阻塞者（hit_list）
      └─ for each victim:
          ├─ lock_mark_trx_for_rollback        // 设 TRX_FORCE_ROLLBACK + killed_by
          ├─ 等待 victim 退出 InnoDB 上下文（trx0trx.cc:3562 循环）
          └─ trx_rollback_for_mysql(victim_trx) // killer 线程代其回滚
```

这个异步回滚同样不可中断。`trx_rollback_for_mysql`（trx0roll.cc:266）里 `TrxInInnoDB::is_async_rollback(trx)` 判断 `killed_by == 当前线程`，跳过 InnoDB 上下文追踪，因为 killer 线程已经在 InnoDB 里了。

---

## Misc

### 观察回滚进度

- `SHOW PROCESSLIST`：State 显示 `ROLLBACK` / `query end`
- `SHOW ENGINE INNODB STATUS\G`：`TRANSACTIONS` 段有 undo log entries 和回滚信息
- 回滚期间持有行锁直到结束，会继续阻塞依赖这些行的事务

### 崩溃恢复的回滚

崩溃恢复时未提交的活跃事务由后台线程回滚（`trx_recovery_rollback_thread`，trx0roll.cc:845），打印 `Rolling back trx with id ...` / `Rollback of trx with id ... completed`。大事务恢复回滚同样慢，这也是崩溃后启动慢的常见原因。

### 回滚为何比正向慢

正向执行是顺序写 undo（追加），回滚是**随机读 undo 页 + 随机更新数据页**，且每次 undo 还要写 redo。所以回滚的单位行成本通常高于正向。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `trx0types.h:83` | `trx_state_t` 五态枚举 |
| `trx0trx.h` | `trx_t` 结构定义 |
| `trx0trx.cc:1405` | `trx->no` 运行期恒为 TRX_ID_MAX |
| `trx0trx.cc:1501` | `trx_add_to_serialisation_list` — 提交时分配 trx->no |
| `trx0trx.cc:3169` | `trx_prepare_for_mysql` — prepare 入口 |
| `trx0trx.cc:2976` | `trx_prepare_low` — undo 标记 PREPARED + 写 redo |
| `trx0trx.cc:3521` | `trx_kill_blocking` — 高优先级事务杀阻塞者 |
| `trx0roll.cc:79` | `trx_rollback_to_savepoint_low` — 回滚核心（无 kill 检查） |
| `trx0roll.cc:142` | `trx_rollback_to_savepoint` — 完整/savepoint 回滚 |
| `trx0roll.cc:266` | `trx_rollback_for_mysql` — 回滚入口（区分异步 killer） |
| `trx0roll.cc:573` | `trx_rollback_active` — 崩溃恢复回滚 |
| `trx0roll.cc:845` | `trx_recovery_rollback_thread` — 后台回滚线程 |
| `ha_innodb.cc:3152` | `trx_is_interrupted` — 检查 thd->killed |
| `ha_innodb.cc:5921` | `innobase_rollback` — 回滚 handlerton 回调 |
| `ha_innodb.cc:6228` | `innobase_kill_connection` — 取消锁等待 |
| `ha_innodb.cc:5762` | `innobase_commit` — 提交 handlerton 回调 |
| `handler.cc:2049` | `ha_rollback_trans` — server 层回滚入口 |
| `sql_parse.cc:6436` | `kill_one_thread` — KILL 处理 |
| `sql_parse.cc:6500` | `sql_kill` — KILL 语句入口 |
| `row0undo.cc` | `row_undo_step` — 逐行撤销（无 kill 检查） |
