# InnoDB Query Graph 执行模型深度解析

> 基于 MySQL 8.0.39 源码，涵盖 fork/thr/node 三层架构、查询图构建与生命周期、row_prebuilt_t 与查询图关系、que_run_threads 调度器、锁等待挂起与精确唤醒机制、srv_sys->tasks 全局队列与 purge 调度。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [查询图的构建](#查询图的构建)
- [核心概念：fork / thr / node 三层架构](#核心概念fork--thr--node-三层架构)
- [节点类型与 C 风格多态](#节点类型与-c-风格多态)
- [row_prebuilt_t 与查询图的关系](#row_prebuilt_t-与查询图的关系)
- [SELECT 不走查询图](#select-不走查询图)
- [执行调度器：que_run_threads 与 que_thr_step](#执行调度器que_run_threads-与-que_thr_step)
- [thr 状态机](#thr-状态机)
- [锁等待挂起机制](#锁等待挂起机制)
- [精确唤醒：从 2PL 放锁到 os_event_set](#精确唤醒从-2pl-放锁到-os_event_set)
- [srv_sys->tasks 全局队列与 purge 调度](#srv_systasks-全局队列与-purge-调度)
- [两种执行模型对比：1:1 阻塞 vs task queue 分发](#两种执行模型对比11-阻塞-vs-task-queue-分发)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

InnoDB 内部实现了一套微型解释器（Interpreter Pattern），将 DML 操作（INSERT / UPDATE / DELETE）和内部任务编译成有向图（Query Graph），由虚拟执行线程（que_thr_t）在图上逐节点游走执行。

### 用途

InnoDB 不直接调用函数完成 DML 操作，而是将操作编译成查询图后统一调度执行。这提供了三个能力：

1. **锁等待挂起/恢复**：行操作遇到锁冲突时挂起当前 thr，锁释放后精确唤醒继续执行，thr 的状态（游标位置等）跨挂起/恢复保持。
2. **多行处理循环**：一条 DML 匹配多行时，thr 在 sel_node 和 upd_node 之间来回切换，每次处理一行。
3. **purge 并行分发**：purge coordinator 将多个 purge thr 入全局队列，多个 srv worker 并行取执行。

### 版本演进

| 版本 | 变化 |
|------|------|
| 1996-1998 | Heikki Tuuri 独立开发 InnoDB，实现完整 SQL 解析器（pars0pars）+ 查询图执行器 + 优化器。InnoDB 可以直接执行 SQL 字符串（CREATE TABLE 于 1998/1/27 首次执行成功） |
| 1999-2000 | InnoDB 接入 MySQL 成为存储引擎。MySQL 负责 SQL 解析和优化，InnoDB 只负责存储。**解析器和优化器大部分弃用**，但查询图执行框架保留 |
| 5.x | 查询图执行模型稳定，fork/thr/node 三层架构定型。pars0pars 仅剩 `pars_complete_graph_for_exec` 一个活跃函数，其余解析功能不再使用 |
| 8.0 | 引入 `try_relatch_trx_and_shard_and_do` 乐观 latching 优化（避免全局 X latch），SRV_WORKER 仍只服务 purge |

---

## 理论基础

### 设计模式

本机制使用了多个经典设计模式：

- **Interpreter Pattern（解释器模式）**：SQL 操作被编译成查询图（语法树），由解释器（que_run_threads）逐节点遍历执行。每个节点类型有对应的 step 函数，类似于 AST 解释器中的 eval 方法。
- **State Machine（状态机）**：que_thr_t 有四种状态（RUNNING / LOCK_WAIT / COMPLETED / SUSPENDED），状态转换由 que_thr_stop / que_thr_end_lock_wait 等函数驱动。锁等待挂起/恢复是状态机的典型应用。
- **Data-Controller Separation（数据-控制流分离）**：node 是数据（做什么、做到哪了），thr 是控制流（谁在做、下一步做什么）。分离后 thr 可在不同 node 间流转，node 状态跨挂起/恢复保持。
- **C-Style Polymorphism（C 风格多态）**：que_common_t 作为第一个字段嵌入所有节点类型，通过 type 字段 + static_cast 实现多态分发，类似 C++ vtable 但无虚函数开销。
- **Coroutine-like Execution（类协程执行）**：thr 在 node 间流转（sel_node → upd_node → sel_node）类似协程的 yield/resume，但底层是 1:1 阻塞模型而非 M:N 调度。

### 相关论文

- **"Efficient Locking for Concurrent Operations on B-Trees"**, Lehman & Yao, 1981 — 提出了 B-link tree 和乐观锁耦合，InnoDB 的 B+树遍历借鉴了这一思想。查询图中的锁等待挂起/恢复机制与 B-tree 乐观锁耦合的"遇到锁冲突则等待"思路一致。
- **"Granularity of Locks and Degrees of Consistency in Shared Data Banks"**, Gray et al., 1975 — 2PL（两阶段锁）的经典论文。InnoDB 的 `trx_release_impl_and_expl_locks` 是 2PL shrinking phase 的实现，精确唤醒发生在放锁阶段。
- **"The Volcano Optimizer Generator"**, Graefe, 1993 — 提出了 iterator model（火山模型），每个 operator 有 open/next/close 接口。InnoDB 的 query graph 与火山模型有相似之处（都是逐行拉取），但 InnoDB 用 goto + run_node 指针替代了函数调用栈，更适合锁等待挂起。

### 算法与数据结构

- **有向图（DAG）**：查询图本质是一棵有向树（fork → thr → node 链表/树），节点间通过 `next` / `child` 指针连接。遍历复杂度 O(n)，n 为节点数。
- **Wait-for Graph（等待图）**：死锁检测使用等待图，节点是事务，边是"等待谁释放锁"。`lock_grant_or_update_wait_for_edge` 维护等待图边，InnoDB 通过 DFS 检测环来判断死锁。
- **全局 task queue（链表）**：`srv_sys->tasks` 是一个侵入式双向链表（UT_LIST），入队 O(1)，出队 O(1)，mutex 保护。
- **slot 数组**：`lock_sys->waiting_threads` 是预分配的 slot 数组，大小 `srv_max_n_threads`。分配 slot 是线性扫描找空闲位，复杂度 O(n)，但通过 `last_slot` 指针优化了常见情况。
- **乐观锁耦合 latching**：`try_relatch_trx_and_shard_and_do` 使用 `trx_locks_version` 做乐观校验——先放 trx->mutex 加 shard latch，重新加 trx->mutex 后检查 version 是否变化，未变则继续，变了则重试。避免了全局 X latch 的串行化开销。

### 类似实现对比

| 项目 | 机制名称 | 与 InnoDB 的异同 |
|------|----------|------------------|
| PostgreSQL | **Executor Tree**（PlanState 树） | 相同点：都是解释器模式，每个 PlanState 节点有 ExecProcNode 回调；不同点：PG 用函数调用栈（pull model），InnoDB 用 goto + 指针（push model）；PG 的锁等待在 LockAcquire 中阻塞，无显式 thr 概念 |
| Oracle | **SQL Execution Engine** | 相同点：有 query graph 概念；不同点：Oracle 使用 row source 模型，锁等待通过 enqueue 实现，有专门的 LCK0 后台进程做死锁检测，而 InnoDB 在线检测 |
| SQL Server | **QDS (Query Data Store)** + **SQLOS** | 相同点：SQLOS 有 scheduler/worker 概念类似 thr；不同点：SQLOS 是真正的 M:N 调度（worker 可迁移 scheduler），InnoDB 是 1:1 阻塞模型 |
| SQLite | **VDBE (Virtual Database Engine)** | 相同点：都是将 SQL 编译成指令序列由解释器执行；不同点：SQLite 用线性指令流（opcode 列表），InnoDB 用图结构；SQLite 无锁等待挂起/恢复机制 |
| ClickHouse | **Pipeline Executor** | 相同点：将查询编译成 pipeline 由解释器执行；不同点：ClickHouse 是 push model + M:N 调度，InnoDB 是 push model + 1:1 阻塞 |

### 历史背景

InnoDB 最初由 Heikki Tuuri 在 1995 年独立开发，查询图执行模型从 InnoDB 诞生之初就存在。设计动机：

1. **为什么用解释器而非直接函数调用**：InnoDB 需要在 SQL 执行过程中支持锁等待挂起/恢复。如果用函数调用栈，挂起时需要保存整个调用栈（成本高）；查询图 + thr 状态机只需保存 run_node 指针和 node 中的游标位置，恢复后从 run_node 继续即可。

2. **为什么不用 M:N 调度**：1995 年 Linux 还没有 NPTL（2003 年才引入），pthread 实现昂贵且不稳定。1:1 阻塞模型最简单可靠——OS 调度器已经足够高效，不需要用户态调度层。这个决策延续至今，8.0 仍然如此。

3. **接入 MySQL 后解析器弃用**：InnoDB 最初有完整的 SQL 解析器（pars0pars）和优化器（pars0opt），可以直接执行 SQL 字符串。1999 年接入 MySQL 后，MySQL 负责 SQL 解析和优化，InnoDB 只负责存储。**解析器和优化器大部分弃用**，但查询图执行框架保留。当前 pars0pars 中仅 `pars_complete_graph_for_exec` 一个函数是活跃的——MySQL 调用 InnoDB handler API 时，InnoDB 内部仍需要构建查询图来执行 DML 操作。

4. **SRV_WORKER 注释的历史遗留**：`srv0srv.h:966` 注释说 SRV_WORKER 服务"parallelized queries and queries released from lock wait"，暗示曾计划让锁等待恢复后的 thr 迁移到 srv worker 执行（类似 M:N）。但这个设计从未实现，当前 SRV_WORKER 只服务 purge。注释保留是因为没人去改。

5. **8.0 的乐观 latching 优化**：早期版本放锁时需要全局 X latch（`lock_sys->mutex`），8.0 引入 `try_relatch_trx_and_shard_and_do` 乐观锁耦合，放 trx->mutex → 加 shard latch → 重新加 trx->mutex → 验证 version，避免了全局 X latch 的串行化瓶颈。这是 query graph 执行模型在 8.0 的主要演进。

---

## 查询图的构建

### 入口：pars_complete_graph_for_exec

所有查询图的构建都通过 `pars_complete_graph_for_exec()`（`pars0pars.cc:1734`）完成：

```cpp
que_thr_t *pars_complete_graph_for_exec(que_node_t *node, trx_t *trx,
                                        mem_heap_t *heap,
                                        row_prebuilt_t *prebuilt) {
  que_fork_t *fork = que_fork_create(nullptr, nullptr,
                                     QUE_FORK_MYSQL_INTERFACE, heap);
  fork->trx = trx;

  que_thr_t *thr = que_thr_create(fork, heap, prebuilt);

  thr->child = node;              // 把操作节点挂到 thr 下

  if (node) {
    que_node_set_parent(node, thr);
  }

  trx->graph = nullptr;
  return (thr);
}
```

这个函数做三件事：创建 fork（语句容器）→ 创建 thr（执行线程）→ 把 node 挂到 thr 下。返回 thr 供后续 `que_run_threads(thr)` 执行。

### 四种操作的查询图构建

InnoDB 为每种 SQL 操作预编译了对应的查询图：

#### INSERT 查询图

`row_get_prebuilt_insert_row()`（`row0mysql.cc:1060`）：

```cpp
if (prebuilt->ins_node != nullptr) {
    // 缓存命中：直接复用已有 graph
    if (prebuilt->trx_id == table->def_trx_id &&
        index_count_matches) {
      return prebuilt->ins_node->row;   // 复用，不重建
    }
    // 表结构变了（DDL 了），释放旧 graph 重建
    que_graph_free_recursive(prebuilt->ins_graph);
    prebuilt->ins_graph = nullptr;
}

// 创建新 ins_node + query graph
node = ins_node_create(INS_DIRECT, table, prebuilt->heap);
prebuilt->ins_node = node;
// ...
prebuilt->ins_graph = que_node_get_parent(
    pars_complete_graph_for_exec(node, prebuilt->trx,
                                 prebuilt->heap, prebuilt));
prebuilt->ins_graph->state = QUE_FORK_ACTIVE;
prebuilt->trx_id = table->def_trx_id;  // 记录构建时的 trx_id
```

**缓存复用**：如果 `ins_node != nullptr` 且 `trx_id` 未变且索引数未变，直接复用已有 graph，不重建。这是重要的性能优化——同一张表的多次 INSERT 不需要每次都构建新 graph。

#### SELECT 查询图（dummy）

`row_prebuild_sel_graph()`（`row0mysql.cc:1747`）：

```cpp
if (prebuilt->sel_graph == nullptr) {
    node = sel_node_create(prebuilt->heap);
    prebuilt->sel_graph = que_node_get_parent(
        pars_complete_graph_for_exec(node, prebuilt->trx,
                                     prebuilt->heap, prebuilt));
    prebuilt->sel_graph->state = QUE_FORK_ACTIVE;
}
```

sel_graph 是 **dummy graph**——不会真正执行 SELECT，只是借它的 thr 作为锁系统的参数载体。

#### UPDATE 查询图

`row_create_update_node_for_mysql()`（`row0mysql.cc:1768`）：

```cpp
node = upd_node_create(prebuilt->heap);
// ... 填充 table、pcur、update 向量等
prebuilt->upd_graph = que_node_get_parent(
    pars_complete_graph_for_exec(node, prebuilt->trx,
                                 prebuilt->heap, prebuilt));
prebuilt->upd_graph->state = QUE_FORK_ACTIVE;
```

#### DELETE 查询图

DELETE 复用 UPDATE 的 upd_graph——InnoDB 中 DELETE 是 UPDATE 的特殊形式（标记删除）。

---

## 核心概念：fork / thr / node 三层架构

### que_fork_t — 语句容器

一条 SQL 语句对应一个 fork（`que_fork_t` = `que_t`），定义在 `include/que0que.h:295`：

```cpp
struct que_fork_t {
  que_common_t common;      // 多态基类，type = QUE_NODE_FORK
  trx_t *trx;               // 所属事务
  ulint n_active_thrs;      // 活跃 thr 数
  UT_LIST_BASE_NODE_T(que_thr_t, thrs) thrs;  // thr 列表
  sym_tab_t *sym_tab;       // 符号表
  mem_heap_t *heap;         // 内存池
};
```

fork 包含这条语句属于哪个事务、有几个 thr 在跑。大多数情况下只有 1 个 thr，但 purge 等场景可以有多个。

### que_thr_t — 虚拟执行线程

定义在 `include/que0que.h:225`：

```cpp
struct que_thr_t {
  que_common_t common;       // type = QUE_NODE_THR
  que_node_t *child;         // thr 的子节点（第一个要执行的 node）
  que_t *graph;              // 所属 fork
  que_thr_state_t state;     // RUNNING / LOCK_WAIT / COMPLETED / SUSPENDED
  que_node_t *run_node;      // 下一个要执行的节点
  que_node_t *prev_node;     // 上一个执行完的节点
  row_prebuilt_t *prebuilt;  // 行操作预编译结构
  struct srv_slot_t *slot;   // 锁等待 slot
};
```

thr 是**控制流**的载体——记录"下一步执行哪个 node"（run_node）、"谁在执行"（通过 thr_get_trx 获取事务）、"当前状态"。thr 不是 OS 线程，是逻辑状态机，必须依附于一个 OS 线程才能执行。

### node — 操作状态节点

每种操作对应一种 node，记录"做什么、做到哪了"。例如 `upd_node_t`（`include/row0upd.h`）：

```cpp
struct upd_node_t {
  que_common_t common;      // type = QUE_NODE_UPDATE
  dict_table_t *table;      // 目标表
  btr_pcur_t *pcur;         // 持久化游标（定位到的记录）
  upd_t *update;            // 更新向量（哪些列改成什么值）
};
```

node 跨多次调用保持状态。例如 UPDATE 多行时，每次调用处理一行，node 中的 pcur 记录当前游标位置。

### 三者关系

```
que_fork_t "语句"
├── trx
├── thrs (thr 列表)
│   └── que_thr_t "执行线程"
│       ├── run_node → upd_node_t   "正在做 UPDATE"
│       ├── prev_node → sel_node_t  "刚做完 SELECT"
│       └── child → proc_node_t / upd_node_t / ...
```

thr 在不同 node 之间流转执行。一个 thr 可以执行多个不同的 node（如 sel_node 找到一行后切到 upd_node 更新，再回到 sel_node 找下一行）。

---

## 节点类型与 C 风格多态

### que_common_t — 多态基类

所有节点类型都以 `que_common_t` 作为第一个字段，实现 C 风格多态：

```cpp
struct que_common_t {
  ulint type;       // 节点类型（QUE_NODE_UPDATE / QUE_NODE_INSERT / ...）
  // ...
};
```

通过 `que_node_get_type(node)` 读取 type 字段判断节点类型，然后 `static_cast` 到具体子类型。

### 节点类型清单

`que_thr_step()`（`que0que.cc:923`）中的 switch 分发列出了所有节点类型：

| 节点类型 | 类型常量 | step 函数 | 说明 |
|----------|----------|-----------|------|
| UPDATE | `QUE_NODE_UPDATE` | `row_upd_step` | 更新行 |
| INSERT | `QUE_NODE_INSERT` | `row_ins_step` | 插入行 |
| SELECT | `QUE_NODE_SELECT` | `row_sel_step` | 查询行 |
| UNDO | `QUE_NODE_UNDO` | `row_undo_step` | 回滚行 |
| PURGE | `QUE_NODE_PURGE` | `row_purge_step` | 清理 undo |
| ASSIGNMENT | `QUE_NODE_ASSIGNMENT` | `assign_step` | 赋值 |
| IF | `QUE_NODE_IF` | `if_step` | 条件分支 |
| WHILE | `QUE_NODE_WHILE` | `while_step` | 循环 |
| FOR | `QUE_NODE_FOR` | `for_step` | 循环 |
| PROC | `QUE_NODE_PROC` | `proc_step` | 过程容器 |
| FETCH | `QUE_NODE_FETCH` | `fetch_step` | 游标取值 |
| COMMIT | `QUE_NODE_COMMIT` | `trx_commit_step` | 事务提交 |
| FUNC | `QUE_NODE_FUNC` | `proc_eval_step` | 函数求值 |

### 设计模式

这是经典的 **Interpreter Pattern**（解释器模式），具体实现为基于状态机的协程式执行：node 是数据（做什么），thr 是控制流（谁在做、下一步做什么），分离后 thr 可在不同 node 间流转，node 状态在锁等待挂起/恢复之间保持。

---

## row_prebuilt_t 与查询图的关系

### row_prebuilt_t 中的 graph 字段

`row_prebuilt_t` 是 MySQL handler 层与 InnoDB 之间的预编译结构，包含 4 个查询图字段（`include/row0mysql.h:670-683`）：

```cpp
struct row_prebuilt_t {
  // ...
  trx_id_t trx_id;          // 构建 ins_graph 时记录的 table->def_trx_id
  que_fork_t *ins_graph;    // INSERT 查询图
  que_fork_t *upd_graph;    // UPDATE/DELETE 查询图
  btr_pcur_t *pcur;         // SELECT 持久化游标
  btr_pcur_t *clust_pcur;   // 聚簇索引持久化游标
  que_fork_t *sel_graph;    // SELECT dummy 查询图
  dtuple_t *search_tuple;   // 搜索条件
  dtuple_t *m_stop_tuple;   // 范围查询结束条件
  // ...
};
```

### 各 graph 的使用场景

| graph 字段 | 使用场景 | 是否真执行 |
|-----------|---------|-----------|
| `sel_graph` | 锁系统参数载体（row_lock_table） | 否（dummy） |
| `ins_graph` | `row_insert_for_mysql_using_ins_graph()` | 是 |
| `upd_graph` | `row_update_for_mysql()` / `row_delete_for_mysql()` | 是 |

### 缓存复用与失效

INSERT graph 有缓存复用机制（`row0mysql.cc:1062-1079`）：

```cpp
if (prebuilt->ins_node != nullptr) {
    // 检查是否可复用
    if (prebuilt->trx_id == table->def_trx_id && index_count_matches) {
      return prebuilt->ins_node->row;   // 复用
    }
    // 表结构变了，释放旧 graph 重建
    que_graph_free_recursive(prebuilt->ins_graph);
    prebuilt->ins_graph = nullptr;
}
```

**trx_id 校验**：`prebuilt->trx_id` 记录构建 ins_graph 时的 `table->def_trx_id`。如果表经历了 DDL（如 ALTER TABLE），`def_trx_id` 会变化，导致旧 graph 失效并重建。这保证了 graph 与当前表结构的一致性。

**索引数校验**：检查 `ins_node->entry_list` 中的索引数与 `table->indexes` 是否一致，不一致说明索引被增删，需要重建。

---

## SELECT 不走查询图

### SELECT 的实际执行路径

SELECT **不走查询图**。`row_search_mvcc()` 和 `row_search_no_mvcc()` 是独立的搜索函数，直接操作 B+树和持久化游标：

```
MySQL SQL 层 → ha_innobase::general_fetch()     // ha_innodb.cc:10524
  → row_search_mvcc(buf, ..., m_prebuilt, ...)    // 直接搜索，不走查询图
  → row_search_no_mvcc(buf, ..., m_prebuilt, ...) // 同上
```

`sel_graph` 只是给锁系统借 thr 用的 dummy（如 `row_lock_table()`），不是执行 SELECT 的载体。

### 为什么 SELECT 不走查询图

查询图执行模型是为**锁等待挂起/恢复**设计的。SELECT 在 InnoDB 中通常不需要锁等待（MVCC 读不加锁），只有在 `SELECT ... FOR UPDATE` / `SELECT ... LOCK IN SHARE MODE` 时才加锁。即使加锁，`row_search_mvcc()` 内部直接调用 `sel_set_rec_lock()` 处理锁等待，不需要 query graph 的挂起/恢复机制。

INSERT / UPDATE / DELETE 必须走查询图，因为它们可能遇到锁冲突（唯一性检查、gap lock 等），需要锁等待挂起/恢复。

---

## 执行调度器：que_run_threads 与 que_thr_step

### 入口：que_run_threads

`que_run_threads()`（`que0que.cc:1082`）是执行入口，包含一个 `goto loop` 循环：

```cpp
void que_run_threads(que_thr_t *thr) {
loop:
  que_run_threads_low(thr);          // 执行一轮

  switch (thr->state) {
    case QUE_THR_RUNNING:
      goto loop;                      // 锁等待已结束，继续执行
    case QUE_THR_LOCK_WAIT:
      lock_wait_suspend_thread(thr);  // 挂起 OS 线程
      // ... 被唤醒后
      goto loop;                      // 继续执行
    case QUE_THR_COMPLETED:
      break;                          // 完成
  }
}
```

### 内层：que_run_threads_low → que_thr_step

`que_run_threads_low()` 循环调用 `que_thr_step()`（`que0que.cc:923`），每次执行一个 node：

```cpp
static inline que_thr_t *que_thr_step(que_thr_t *thr) {
  node = thr->run_node;                    // 取当前节点
  type = que_node_get_type(node);          // 判断类型

  if (type == QUE_NODE_UPDATE) {
    thr = row_upd_step(thr);               // 执行 UPDATE
  } else if (type == QUE_NODE_INSERT) {
    thr = row_ins_step(thr);               // 执行 INSERT
  } else if (type == QUE_NODE_SELECT) {
    thr = row_sel_step(thr);               // 执行 SELECT
  }
  // ...
}
```

### step 函数的固定签名

每种节点的 step 函数只接收 `que_thr_t *thr`，内部从 `thr->run_node` 取出具体 node：

```cpp
que_thr_t *row_upd_step(que_thr_t *thr) {
  node = static_cast<upd_node_t *>(thr->run_node);
  trx = thr_get_trx(thr);
  err = row_upd_clust_step(node, thr);   // node 和 thr 分开传给底层
  // ...
}
```

底层函数（如 `row_upd_clust_step`）接收 `node` 和 `thr` 两个参数：node 提供"做什么"（表、游标、更新向量），thr 提供"谁在做"（事务、控制流）。

### 调用链示例

以 `UPDATE t1 SET x=1 WHERE id=5` 为例：

```
MySQL SQL 层 → ha_innobase::update_row()
  → que_run_threads(thr)                         // que0que.cc:1082
    → que_run_threads_low(thr)
      → que_thr_step(thr)                         // que0que.cc:923
        → row_sel_step(thr)  (找满足条件的行)
          → thr->run_node = upd_node  (切到 UPDATE)
        → row_upd_step(thr)                       // row0upd.cc:3300
          → row_upd_clust_step(node, thr)          // row0upd.cc:3044
            → trx = thr_get_trx(thr)
            → pcur = node->pcur
            → 更新聚簇索引记录
```

---

## thr 状态机

### 四种状态

```cpp
enum que_thr_state_t {
  QUE_THR_RUNNING,      // 正在执行
  QUE_THR_COMPLETED,    // 执行完成
  QUE_THR_LOCK_WAIT,    // 锁等待中
  QUE_THR_SUSPENDED,    // 挂起（fork 等待命令）
};
```

### 状态转换

```
                    que_fork_start_command()
                         │
                         ▼
              ┌───── QUE_THR_RUNNING ─────┐
              │         │                 │
              │    que_thr_stop()    que_thr_end_lock_wait()
              │         │                 │
              │         ▼                 │
              │  QUE_THR_LOCK_WAIT ───────┘
              │         │
              │    lock_wait_suspend_thread()
              │         │
              │         ▼
              │    [os_event_wait 阻塞]
              │         │
              │    被唤醒
              │         │
              └─────────┘
                         │
              que_thr_stop_for_mysql()
                         │
                         ▼
              QUE_THR_COMPLETED
```

---

## 锁等待挂起机制

### 状态设置的完整链路

当行操作遇到锁冲突时，状态设置经过以下步骤：

```
1. lock_table() / lock_rec_lock() 冲突
   → trx->lock.que_state = TRX_QUE_LOCK_WAIT
   → 返回 DB_LOCK_WAIT

2. 回到调用者（如 row_lock_table()）
   → que_thr_stop_for_mysql(thr)
     → 此时 thr->state 还是 RUNNING
     → 看到 error_state==DB_LOCK_WAIT，直接 return（不改状态）

3. row_mysql_handle_errors() 看到 DB_LOCK_WAIT
   → que_thr_stop(thr)                          // que0que.cc:650
     → 检查 trx->lock.que_state == TRX_QUE_LOCK_WAIT
     → trx->lock.wait_thr = thr
     → thr->state = QUE_THR_LOCK_WAIT           // ← 这里才设置

4. 回到 que_run_threads() 的 switch
   → case QUE_THR_LOCK_WAIT:
     → lock_wait_suspend_thread(thr)            // que0que.cc:1100
```

**关键**：`QUE_THR_LOCK_WAIT` 状态在 `que_thr_stop()` 中设置（`que0que.cc:682`），不是在 `que_run_threads` 的 switch 中设置，也不是在 `que_thr_stop_for_mysql` 中设置。

### que_thr_stop vs que_thr_stop_for_mysql

| | `que_thr_stop()` | `que_thr_stop_for_mysql()` |
|---|---|---|
| 位置 | `que0que.cc:650` | `que0que.cc:770` |
| 调用者 | `que_thr_dec_refer_count()`（内部调度路径） | MySQL 接口路径（如 `row_lock_table()`） |
| 是否设置 LOCK_WAIT | **是** | **否** |
| 场景 | 内部 query graph 执行循环 | MySQL handler 层调用 InnoDB 后的收尾 |

`que_thr_stop_for_mysql()` 不设置 `LOCK_WAIT`。它处理 MySQL 接口路径的收尾——如果 thr 还在 RUNNING 且有错误就标记 COMPLETED，如果已经因为锁等待被 `que_thr_stop()` 设置了 `LOCK_WAIT` 就直接 return（保持原状态不变）。

### lock_wait_suspend_thread 的实际挂起

`lock_wait_suspend_thread()`（`lock0wait.cc:200`）执行真正的 OS 线程阻塞：

```cpp
void lock_wait_suspend_thread(que_thr_t *thr) {
  trx = thr_get_trx(thr);

  // [1] 竞态检查：如果 thr 已被唤醒则不睡
  if (thr->state == QUE_THR_RUNNING) {
    // 锁已释放或被选为 deadlock victim
    return;
  }

  // [2] 分配 wait slot
  slot = lock_wait_table_reserve_slot(thr, lock_wait_timeout);

  // [3] 释放持有的锁和 mutex
  if (was_declared_inside_innodb) {
    srv_conc_force_exit_innodb(trx);    // 退出 InnoDB 并发控制
  }

  // [4] 真正挂起 OS 线程
  os_event_wait(slot->event);          // ← lock0wait.cc:299

  // [5] 被唤醒后的善后
  lock_wait_table_release_slot(slot);
  srv_conc_force_enter_innodb(trx);    // 重新进入 InnoDB
}
```

`os_event_wait(slot->event)` 底层是 `pthread_cond_wait` 或 `futex`，OS 线程在此被调度出去。

### 挂起的是哪个线程

**是 user thd 的 OS 线程在等。** 不是后台线程，也不是独立的 query graph 线程。

InnoDB 的 query graph 执行是 **1:1 模型**——一个 user connection 对应一个 OS 线程，这个线程同时就是执行 query graph 的线程。`que_thr_t` 只是逻辑状态机，底层跑在 user thd 的 OS 线程上。锁等待时 user thd 阻塞在 `os_event_wait` 上，不会去执行其他 query node。

### 竞态检查的必要性

从 `que_thr_stop()` 设置 `QUE_THR_LOCK_WAIT` 到 `lock_wait_suspend_thread()` 被调用之间，持有锁的事务可能已经释放锁并调用了 `lock_wait_release_thread_if_suspended()`，把 thr 状态改回了 `QUE_THR_RUNNING`。

如果到 suspend 时发现状态已经变回 `RUNNING`，说明锁已释放，不需要睡了，直接返回继续执行。这避免了"锁已释放但线程还在等"的假唤醒问题。

---

## 精确唤醒：从 2PL 放锁到 os_event_set

### 完整调用链

事务提交时 2PL shrinking phase 释放所有锁，唤醒等待者：

```
trx_commit_in_memory()                              // trx0trx.cc
  └─ trx_release_impl_and_expl_locks(trx, ...)       // trx0trx.cc:2045
      └─ lock_trx_release_locks(trx)                  // lock0lock.cc:6173
          │
          │  // 等待 trx 不被引用
          │
          └─ while (!locksys::try_release_all_locks(trx))  // lock0lock.cc:6227
              │   yield() 重试直到成功
              │
              └─ try_release_all_locks(trx)           // lock0lock.cc:4304
                  │  // 持有 shared global latch
                  │  // 遍历 trx->lock.trx_locks 链表
                  │
                  └─ for each lock:
                      try_relatch_trx_and_shard_and_do(lock, ...)
                      │  // 乐观 latching：放 trx->mutex → 加 shard latch
                      │  // → 重新加 trx->mutex → 验证 trx_locks_version 未变
                      │
                      ├─ LOCK_REC:
                      │   lock_rec_dequeue_from_page(lock)   // lock0lock.cc:2436
                      │     ├─ lock_rec_discard(lock)          // 从 hash 删除
                      │     └─ lock_rec_grant(lock)            // 遍历等待队列授权
                      │
                      └─ LOCK_TABLE:
                          lock_table_dequeue(lock)
```

### 授权等待者

`lock_rec_grant()`（`lock0lock.cc:2392`）遍历锁队列中等待的 lock，对每个调用 `lock_grant_or_update_wait_for_edge_if_waiting()`（`lock0lock.cc:2375`）：

```cpp
static void lock_grant_or_update_wait_for_edge_if_waiting(
    lock_t *lock, const trx_t *releasing_trx) {
  if (lock->is_waiting() && lock->trx->lock.blocking_trx == releasing_trx) {
    lock_grant_or_update_wait_for_edge(lock);
  }
}
```

如果等待者的 `blocking_trx` 就是正在释放锁的事务，则授权（或更新 wait-for-graph 边）。

### 精确唤醒

授权后调用 `lock_reset_wait_and_release_thread_if_suspended()`（`lock0wait.cc:423`）：

```cpp
void lock_reset_wait_and_release_thread_if_suspended(lock_t *lock) {
  // [1] 清除阻塞者
  lock->trx->lock.blocking_trx.store(nullptr);

  // [2] thr 状态改回 RUNNING
  que_thr_t *thr = que_thr_end_lock_wait(lock->trx);  // que0que.cc:266
  //   → que_thr_move_to_run_state(thr)
  //   → thr->state = QUE_THR_RUNNING
  //   → trx->lock.que_state = TRX_QUE_RUNNING

  // [3] 清除 wait_lock（防二次唤醒）
  lock_reset_lock_and_trx_wait(lock);
  //   → trx->lock.wait_lock = nullptr

  // [4] 精确唤醒 OS 线程
  if (thr != nullptr) {
    lock_wait_release_thread_if_suspended(thr);      // lock0wait.cc:358
  }
}
```

`lock_wait_release_thread_if_suspended()`（`lock0wait.cc:358`）：

```cpp
static void lock_wait_release_thread_if_suspended(que_thr_t *thr) {
  auto trx = thr_get_trx(thr);

  // 检查 thr->slot 是否已分配（线程是否已睡着）
  if (thr->slot != nullptr && thr->slot->in_use && thr->slot->thr == thr) {
    // 线程已经在 os_event_wait 中睡着
    os_event_set(thr->slot->event);   // ← 精确唤醒！
  }
  // 如果 thr->slot 未分配：线程还没 os_event_wait
  // thr->state 已是 RUNNING，线程在 lock_wait_suspend_thread 中检查后
  // 会发现状态是 RUNNING 而直接返回，不睡
}
```

### 防二次唤醒设计

`lock0wait.cc:358-423` 注释明确了四条规则保证每个 trx 最多被唤醒一次：

1. 唯一唤醒途径是 `os_event_set`
2. 唯一 `os_event_set` 调用在 `lock_wait_release_thread_if_suspended`
3. 调用前必须先 `lock_reset_lock_and_trx_wait(lock)`
4. `lock_reset_lock_and_trx_wait` 断言 `wait_lock == lock` 且置 `wait_lock = NULL`

再次唤醒需要 `wait_lock` 非 NULL，故不可能二次唤醒。

---

## srv_sys->tasks 全局队列与 purge 调度

### 队列定义

`srv_sys_t`（`srv0srv.cc:758`）中定义了一个全局 task queue：

```cpp
struct srv_sys_t {
  ib_mutex_t tasks_mutex;                          // 保护队列的 mutex
  UT_LIST_BASE_NODE_T(que_thr_t, queue) tasks;     // 全局 query thread 就绪队列
  srv_slot_t *sys_threads;                          // server 线程表
  // ...
};
```

### 入队：srv_que_task_enqueue_low

`srv_que_task_enqueue_low()`（`srv0srv.cc:3205`）是唯一的入队函数：

```cpp
void srv_que_task_enqueue_low(que_thr_t *thr) {
  mutex_enter(&srv_sys->tasks_mutex);
  UT_LIST_ADD_LAST(srv_sys->tasks, thr);       // thr 入队尾
  mutex_exit(&srv_sys->tasks_mutex);
  srv_release_threads(SRV_WORKER, 1);           // 唤醒一个 worker
}
```

唯一调用者在 `trx0purge.cc:2515`——purge coordinator 提交 purge thr。

### 出队：srv_task_execute

`srv_task_execute()`（`srv0srv.cc:2813`）从队列取 thr 执行：

```cpp
static bool srv_task_execute(void) {
  mutex_enter(&srv_sys->tasks_mutex);
  if (UT_LIST_GET_LEN(srv_sys->tasks) > 0) {
    thr = UT_LIST_GET_FIRST(srv_sys->tasks);
    ut_a(que_node_get_type(thr->child) == QUE_NODE_PURGE);  // 只允许 purge
    UT_LIST_REMOVE(srv_sys->tasks, thr);
  }
  mutex_exit(&srv_sys->tasks_mutex);

  if (thr != nullptr) {
    que_run_threads(thr);    // srv worker 执行 purge thr
  }
  return thr != nullptr;
}
```

### purge worker 线程循环

`srv_worker_thread()`（`srv0srv.cc:2846`）：

```cpp
void srv_worker_thread() {
  slot = srv_reserve_slot(SRV_WORKER);

  do {
    srv_suspend_thread(slot);
    os_event_wait(slot->event);        // 平时睡眠

    if (srv_task_execute()) {           // 从队列取 thr 执行
      srv_wake_purge_thread_if_not_active();
    }
  } while (purge_sys->state != PURGE_STATE_EXIT);
}
```

---

## 两种执行模型对比：1:1 阻塞 vs task queue 分发

### 模型 1：User DML/DDL — 1:1 阻塞模型

```
User Connection (OS Thread)
  → ha_innobase::update_row()
  → que_run_threads(thr)              ← user thd 自己执行
    → thr 不进 srv_sys->tasks 队列
    → 锁等待 → os_event_wait          ← user thd 阻塞
    → 被唤醒后继续执行
```

普通 DML/DDL 的 thr 从创建到完成都在 user thd 上执行，不进 task queue，不切换到 srv worker。

### 模型 2：Purge — task queue 调度模型

```
purge coordinator (srv_purge thread)
  → 创建 purge thr
  → srv_que_task_enqueue_low(thr)       ← thr 入全局 task queue
  → srv_release_threads(SRV_WORKER, 1)  ← 唤醒 worker

purge worker (srv_worker thread)
  → os_event_wait(slot->event)          ← 平时睡眠
  → srv_task_execute()                   ← 从队列取 thr
    → ut_a(QUE_NODE_PURGE)              ← 断言只允许 purge
    → que_run_threads(thr)              ← srv worker 执行
```

### 对比

| | User DML/DDL | Purge |
|---|---|---|
| 执行线程 | user thd OS 线程 | srv worker thread |
| 进 task queue | 否 | 是 |
| 锁等待时 | user thd 阻塞 | srv worker 阻塞 |
| 切换到其他 thr | 否 | 否（即使 purge worker 遇锁等待也是 os_event_wait 阻塞） |
| 调度模型 | 1:1 阻塞 | task queue 分发 |

**两者都是 1:1 阻塞模型**，没有 M:N 调度。task queue 只是 purge 的分发机制（coordinator → worker），不是阻塞时切换机制。

---

## Misc

### SRV_WORKER 注释误导

`srv0srv.h:966-968` 注释说 SRV_WORKER 服务"parallelized queries and queries released from lock wait"，暗示可以执行普通 user query。但 `srv_task_execute()` 的硬断言 `ut_a(que_node_get_type(thr->child) == QUE_NODE_PURGE)`（srv0srv.cc:2828）限制了只执行 purge。`srv_worker_thread()` 循环绑定 `purge_sys->state`，`srv_release_threads(SRV_WORKER,...)` 的 4 处调用全在 purge 路径。注释描述的是早期设计意图，当前代码 SRV_WORKER 只服务 purge。

### row_lock_table 需要 dummy sel_graph 的原因

`row_lock_table()`（`row0mysql.cc:1239`）上表锁前检查 `prebuilt->sel_graph != nullptr`：

```cpp
if (prebuilt->sel_graph == nullptr) {
  row_prebuild_sel_graph(prebuilt);   // 建 dummy select query graph
}
thr = que_fork_get_first_thr(prebuilt->sel_graph);
```

原因：锁系统 `lock_table()` 的第 4 个参数需要 `que_thr_t`（获取 trx + 挂起/恢复上下文）。DDL 路径（如 `ha_innobase::lock_table()`）可能没有执行过 SELECT，query graph 尚未创建，故建一个 dummy select query graph 充当 thr 载体。这里不会真正执行 SELECT，只是借 thr。

### lock_wait_table_print 的用途

`lock_wait_table_print()`（`lock0wait.cc:51`）打印 `lock_sys->waiting_threads` 数组所有 slot 的状态。仅在 slot 数组耗尽时调用（`lock0wait.cc:185-195`）：

```cpp
slot = lock_wait_table_reserve_slot(thr, lock_wait_timeout);
if (slot == nullptr) {
    ib::error(...) << "Cannot continue operation...";
    lock_wait_table_print();     // ← 打印诊断信息
    ut_error;                    // ← 然后 abort
}
```

`srv_max_n_threads` 个 slot 全被占满（并发等待锁的事务数超过上限）时触发。正常流程不会走到。

### lock_wait_table_release_slot 善后

`lock_wait_table_release_slot()`（`lock0wait.cc:72`）在 `os_event_wait` 返回后被调用（`lock0wait.cc:322`）。此时线程已被唤醒（或超时），这一行是善后清理——把 slot 还回去让其他等待者复用，并调整 `last_slot` 指针。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `pars0pars.cc:1734` | `pars_complete_graph_for_exec()` — 查询图构建入口 |
| `row0mysql.cc:1060` | `row_get_prebuilt_insert_row()` — INSERT graph 缓存复用 |
| `row0mysql.cc:1747` | `row_prebuild_sel_graph()` — SELECT dummy graph 构建 |
| `row0mysql.cc:1768` | `row_create_update_node_for_mysql()` — UPDATE graph 构建 |
| `row0mysql.cc:1239` | `row_lock_table()` — 需要 dummy sel_graph |
| `ha_innodb.cc:10524` | `general_fetch()` — SELECT 实际执行路径（不走查询图） |
| `que0que.cc:1082` | `que_run_threads()` — 执行入口，goto loop 循环 |
| `que0que.cc:923` | `que_thr_step()` — 调度器，按 node 类型分发 |
| `que0que.cc:650` | `que_thr_stop()` — 设置 `QUE_THR_LOCK_WAIT` |
| `que0que.cc:770` | `que_thr_stop_for_mysql()` — MySQL 接口路径 stop，不设 LOCK_WAIT |
| `que0que.cc:266` | `que_thr_end_lock_wait()` — thr 状态从 LOCK_WAIT 改回 RUNNING |
| `que0que.h:225` | `que_thr_t` 结构定义 |
| `que0que.h:295` | `que_fork_t` 结构定义 |
| `row0upd.cc:3300` | `row_upd_step()` — UPDATE step 函数 |
| `row0upd.cc:3044` | `row_upd_clust_step(node, thr)` — 接收 node + thr 两个参数 |
| `lock0wait.cc:200` | `lock_wait_suspend_thread()` — 竞态检查 + os_event_wait |
| `lock0wait.cc:299` | `os_event_wait(slot->event)` — 真正的 OS 线程阻塞 |
| `lock0wait.cc:322` | `lock_wait_table_release_slot()` — 唤醒后善后 |
| `lock0wait.cc:358` | `lock_wait_release_thread_if_suspended()` — os_event_set 精确唤醒 |
| `lock0wait.cc:423` | `lock_reset_wait_and_release_thread_if_suspended()` — 唤醒入口 |
| `lock0wait.cc:51` | `lock_wait_table_print()` — slot 耗尽时诊断打印 |
| `lock0lock.cc:6173` | `lock_trx_release_locks()` — 2PL 放锁入口 |
| `lock0lock.cc:4304` | `try_release_all_locks()` — 遍历 trx_locks 释放 |
| `lock0lock.cc:2436` | `lock_rec_dequeue_from_page()` — 释放行锁并授权等待者 |
| `lock0lock.cc:2375` | `lock_grant_or_update_wait_for_edge_if_waiting()` — 检查等待者是否可授权 |
| `srv0srv.cc:758` | `srv_sys_t` — 含 tasks 全局队列定义 |
| `srv0srv.cc:2813` | `srv_task_execute()` — 从队列取 thr，断言 QUE_NODE_PURGE |
| `srv0srv.cc:2846` | `srv_worker_thread()` — purge worker 循环 |
| `srv0srv.cc:3205` | `srv_que_task_enqueue_low()` — thr 入队 |
| `srv0srv.h:963` | `srv_thread_type` 枚举（SRV_WORKER / SRV_PURGE / SRV_MASTER） |
| `trx0trx.cc:2045` | `trx_release_impl_and_expl_locks()` — 2PL shrinking phase |
