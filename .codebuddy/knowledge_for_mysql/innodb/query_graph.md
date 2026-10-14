# InnoDB Query Graph 执行模型深度解析

> 基于 MySQL 8.0.39 源码，涵盖 fork/thr/node 三层架构、查询图构建与生命周期、row_prebuilt_t 与查询图关系、que_run_threads 调度器、锁等待挂起与精确唤醒机制、srv_sys->tasks 全局队列与 purge 调度。
>
> **边界**：本篇讲 query graph 执行模型的**通用机制**（构建 / 调度 / 挂起 / 唤醒）；回滚图（roll_node 外层 + undo graph 内层的两层结构）与事务回滚的剖析见 [`trx.md`](trx.md)「回滚机制」。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [查询图的构建](#查询图的构建)
    - [核心概念：fork / thr / node 三层架构](#核心概念fork--thr--node-三层架构)
    - [节点类型与 C 风格多态](#节点类型与-c-风格多态)
    - [row_prebuilt_t 与查询图的关系](#row_prebuilt_t-与查询图的关系)
    - [SELECT 不走查询图](#select-不走查询图)
  - 执行与调度
    - [执行调度器：que_run_threads 与 que_thr_step](#执行调度器que_run_threads-与-que_thr_step)
    - [thr 状态机](#thr-状态机)
    - [锁等待挂起机制](#锁等待挂起机制)
    - [精确唤醒：从 2PL 放锁到 os_event_set](#精确唤醒从-2pl-放锁到-os_event_set)
  - 全局队列与执行模型
    - [srv_sys->tasks 全局队列与 purge 调度](#srv_systasks-全局队列与-purge-调度)
    - [两种执行模型对比：1:1 阻塞 vs task queue 分发](#两种执行模型对比11-阻塞-vs-task-queue-分发)
- [Misc](#Misc)
- [参考](#参考)

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

## 核心实现

> 本章讲 query graph 的**构建、调度、挂起与唤醒**，共 11 个环节：查询图构建、fork/thr/node 三层架构、节点多态、prebuilt 关系、SELECT 特例、执行调度器、thr 状态机、锁等待挂起、精确唤醒、purge 队列调度、两种执行模型。回滚图（roll_node 外层 + undo graph 内层）的剖析见 [`trx.md`](trx.md)「回滚机制」。

### 查询图的构建

#### 入口：pars_complete_graph_for_exec

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

#### 四种操作的查询图构建

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
void row_prebuild_sel_graph(row_prebuilt_t *prebuilt) {
  sel_node_t *node;
  ut_ad(prebuilt && prebuilt->trx);

  // ★ 只在没建过时创建 ⇒ 同一 prebuilt 的多次查询复用同一个 graph
  if (prebuilt->sel_graph == nullptr) {
    node = sel_node_create(prebuilt->heap);

    // pars_complete_graph_for_exec 返回的是 que_thr_t，取其 parent 才是 que_fork_t
    // ⇒ 这正是"fork / thr / node 三层架构"的落地（见下文「核心概念」）
    prebuilt->sel_graph = static_cast<que_fork_t *>(que_node_get_parent(
        pars_complete_graph_for_exec(static_cast<sel_node_t *>(node),
                                     prebuilt->trx, prebuilt->heap, prebuilt)));

    // 建好即置为 ACTIVE，后续 row_search_mvcc 直接拿 thr 用
    prebuilt->sel_graph->state = QUE_FORK_ACTIVE;
  }
}
```

★ **为什么 SELECT 也要建图（而且叫 dummy）**：SELECT 本身**不走查询图执行**（见下文「SELECT 不走查询图」），建它只是为了拿到一个 `que_thr_t` 上下文——**锁等待必须挂在 thr 上**（`que_thr_stop` 挂起的是 thr，而 thr 必须属于某个 fork）。⇒ 这个 graph 是"为了能挂起等锁而存在"的壳子，所以是 dummy。

#### `pars_complete_graph_for_exec`：三层是这么拼起来的

调用里的 `pars_complete_graph_for_exec` 只有 20 行，但它**不是包装**——fork / thr / node 三层的连接就发生在这里：

```cpp
que_thr_t *pars_complete_graph_for_exec(que_node_t *node, trx_t *trx,
                                        mem_heap_t *heap,
                                        row_prebuilt_t *prebuilt) {
  que_fork_t *fork;
  que_thr_t *thr;

  // fork 类型 QUE_FORK_MYSQL_INTERFACE ⇒ 给 handler 层用的图，
  // 区别于 SQL 解析器内部那条路径建的图
  fork = que_fork_create(nullptr, nullptr, QUE_FORK_MYSQL_INTERFACE, heap);
  fork->trx = trx;

  // ★ thr 创建时就绑定了 prebuilt ⇒ 执行期 thr 能直接找回 row_prebuilt_t
  thr = que_thr_create(fork, heap, prebuilt);

  // 三层连起来（双向指针）：fork ← thr ← node
  thr->child = node;
  if (node) {
    que_node_set_parent(node, thr);
  }

  // ★ 不挂到 trx->graph ⇒ 这个 dummy 图不参与 trx 的图管理与回滚
  trx->graph = nullptr;

  return (thr);   // ★ 返回的是 thr，不是 fork
}
```

⇒ 这解释了调用方为什么必须 `que_node_get_parent(thr)` 才能拿到 fork：**函数返回的是最下层的 thr**，fork 要沿 parent 指针往上找一层。所谓"fork / thr / node 三层架构"，在代码上就是这两个指针赋值。

> 判据：它虽短，但**没有把活甩给另一个函数**（主体是实际组装），所以**不是包装函数**——这与"26 行壳 + 转调 109 行实现"那种情况不同。

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

### 核心概念：fork / thr / node 三层架构

#### que_fork_t — 语句容器

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

#### que_thr_t — 虚拟执行线程

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

#### node — 操作状态节点

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

#### 三者关系

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

### 节点类型与 C 风格多态

#### que_common_t — 多态基类

所有节点类型都以 `que_common_t` 作为第一个字段，实现 C 风格多态：

⚠️ **勘误**：`que_common_t` **不在 `que0que.h`**，定义在 **`include/que0types.h:52`**；且它是 **5 个字段**（不是常说的 4 个），**没有叫 `next` 的字段**——"下一个"就是 `brother`。

```cpp
// que0types.h:40
/* Pseudotype for all graph nodes */
typedef void que_node_t;   // ★★ que_node_t 就是 void —— 所谓"C 风格多态"全靠强转

/* Query graph root is a fork node */
typedef struct que_fork_t que_t;   // ★ que_t 与 que_fork_t 是同一类型，图的根就是 fork 节点

struct que_thr_t;

/* Common struct at the beginning of each query graph node; the name of this
substruct must be 'common' */          // ★★ 首字段 + 必须叫 common：这套强转的硬约定

struct que_common_t {
  ulint type;          /*!< query node type */
  que_node_t *parent;  /*!< back pointer to parent node, or NULL */
  que_node_t *brother; /* pointer to a possible brother node */   // ★ "next" 就是它
  dfield_t val;        /*!< evaluated value for an expression */  // ★ 连 row 节点都带
  ulint val_buf_size;
  /* buffer size for the evaluated value data,
  if the buffer has been allocated dynamically: ... */
};
```

**访问器不是宏，是 `static inline` 函数**（`include/que0que.ic:36`，由 `que0que.h:405` 引入）：

```cpp
/** Gets the type of a graph node. */
static inline ulint que_node_get_type(const que_node_t *node) {
  return (reinterpret_cast<const que_common_t *>(node)->type);   // ★★ 多态核心一行
}

/** Gets the next list node in a list of query graph nodes. */
static inline que_node_t *que_node_get_next(que_node_t *node) {
  return (((que_common_t *)node)->brother);      // ★ next == brother
}

/** Gets the parent node of a query graph node. */
static inline que_node_t *que_node_get_parent(que_node_t *node) {
  return (((que_common_t *)node)->parent);
}

static inline dfield_t *que_node_get_val(que_node_t *node) {
  return (&(((que_common_t *)node)->val));
}

static inline void que_node_set_parent(que_node_t *node, que_node_t *parent) {
  ((que_common_t *)node)->parent = parent;
}

/** Catenates a query graph node to a list of them, possible empty list. */
static inline que_node_t *que_node_list_add_last(que_node_t *node_list,
                                                 que_node_t *node) {
  que_common_t *cnode = (que_common_t *)node;
  cnode->brother = nullptr;
  if (node_list == nullptr) return (node);
  que_common_t *cnode2 = (que_common_t *)node_list;
  while (cnode2->brother != nullptr) {
    cnode2 = (que_common_t *)cnode2->brother;      // ★ 单向 brother 链，无 prev 指针
  }
  cnode2->brother = node;
  return (node_list);
}
```

> ⚠️ **不存在 `que_node_get_prev()`**（全仓 0 命中）——语义上的"前一个"是 `que_thr_t::prev_node`（**运行时**字段，不是图结构字段）。
>
> ★ **`que_fork_t` 与 `que_thr_t` 也都以 `que_common_t` 开头**，所以它们自己也是"节点"（`QUE_NODE_FORK` / `QUE_NODE_THR`），能被统一释放与遍历。

**`que_fork_t`（== `que_t`）**（`que0que.h:306`）：

```cpp
struct que_fork_t {
  que_common_t common; /*!< type: QUE_NODE_FORK */
  que_t *graph;        /*!< query graph of this node */
  ulint fork_type;     /*!< fork type */          // QUE_FORK_*（类型）
  ulint n_active_thrs; /*!< if this is the root of a graph, the number query
                       threads that have been started in que_thr_move_to_run_state
                       but for which que_thr_dec_refer_count has not yet been
                       called */
  trx_t *trx;          /*!< transaction: this is set only in the root node */
  ulint state;         /*!< state of the fork node */   // QUE_FORK_*（状态，另一组常量）
  que_thr_t *caller;   /*!< pointer to a possible calling query thread */
  UT_LIST_BASE_NODE_T(que_thr_t, thrs) thrs;
  /*------------------------------*/
  /* The fields in this section are defined only in the root node */
  sym_tab_t *sym_tab;  /*!< symbol table of the query, generated by the parser,
                      or NULL if the graph was created 'by hand' */
  pars_info_t *info;
  ulint cur_end; ulint cur_pos; bool cur_on_row;   // 仅 select graph 有意义
  sel_node_t *last_sel_node;
  UT_LIST_NODE_T(que_fork_t) graphs;
  mem_heap_t *heap;    /*!< memory heap where the fork was created */
};
```

**`que_thr_t`**（`que0que.h:243`）：

```cpp
struct que_thr_t {
  que_common_t common;   /*!< type: QUE_NODE_THR */
  ulint magic_n;         /*!< magic number to catch memory corruption */
  que_node_t *child;     /*!< graph child node */
  que_t *graph;          /*!< graph where this node belongs */   // ★ 指向根 fork
  que_thr_state_t state; /*!< state of the query thread */
  bool is_active;        /*!< true if the thread has been set to the run state
                          in que_thr_move_to_run_state, but not deactivated in
                          que_thr_dec_reference_count */
  /* The following fields are private to the OS thread executing the
  query thread, and are not protected by any mutex: */
  que_node_t *run_node;  /*!< pointer to the node where the subgraph down from
                         this node is currently executed */   // ★★ "程序计数器"
  que_node_t *prev_node; /*!< pointer to the node from which the control came */
  ulint resource;        /*!< resource usage of the query thread thus far */
  ulint lock_state;
  struct srv_slot_t *slot;   // 锁等待槽位
  UT_LIST_NODE_T(que_thr_t) thrs;      // fork->thrs 链
  UT_LIST_NODE_T(que_thr_t) trx_thrs;  // trx 等待链
  UT_LIST_NODE_T(que_thr_t) queue;     // srv_sys->tasks 队列（后台任务用）
  ulint fk_cascade_depth;
  row_prebuilt_t *prebuilt;  /*!< prebuilt structure processed by the query thread */
};

constexpr uint32_t QUE_THR_MAGIC_N = 8476583;
constexpr uint32_t QUE_THR_MAGIC_FREED = 123461526;
```

**`QUE_FORK_*` 两组常量（都是 `constexpr`，不是 enum）**（`que0que.h:350`）：

```cpp
/* Query fork (or graph) types */
constexpr uint32_t QUE_FORK_SELECT_NON_SCROLL = 1;  /* forward-only cursor */
constexpr uint32_t QUE_FORK_SELECT_SCROLL = 2;      /* scrollable cursor */
constexpr uint32_t QUE_FORK_INSERT = 3;
constexpr uint32_t QUE_FORK_UPDATE = 4;
constexpr uint32_t QUE_FORK_ROLLBACK = 5;
/* This is really the undo graph used in rollback,
no signal-sending roll_node in this graph */
constexpr uint32_t QUE_FORK_PURGE = 6;
constexpr uint32_t QUE_FORK_EXECUTE = 7;
constexpr uint32_t QUE_FORK_PROCEDURE = 8;
constexpr uint32_t QUE_FORK_PROCEDURE_CALL = 9;
constexpr uint32_t QUE_FORK_MYSQL_INTERFACE = 10;   // ★ ha_innobase 走的就是这个
constexpr uint32_t QUE_FORK_RECOVERY = 11;

/* Query fork (or graph) states */
constexpr uint32_t QUE_FORK_ACTIVE = 1;
constexpr uint32_t QUE_FORK_COMMAND_WAIT = 2;
constexpr uint32_t QUE_FORK_INVALID = 3;
constexpr uint32_t QUE_FORK_BEING_FREED = 4;
```

> ★ `fork_type` 与 `state` **共用 `ulint`**，靠不同常量组区分——这是 C 风格代码常见的隐患点。

**构造：三处绑定 graph 与 trx**（不存在 `que_fork_init()`）：

```cpp
// que0que.cc:198
que_fork_t *que_fork_create(que_t *graph, que_node_t *parent,
                            ulint fork_type, mem_heap_t *heap) {
  que_fork_t *fork = static_cast<que_fork_t *>(mem_heap_zalloc(heap, sizeof(*fork)));
  fork->heap = heap;
  fork->fork_type = fork_type;
  fork->common.parent = parent;
  fork->common.type = QUE_NODE_FORK;      // ★ 给"基类"打类型标签
  fork->state = QUE_FORK_COMMAND_WAIT;
  fork->graph = (graph != nullptr) ? graph : fork;   // ★★ 根 fork 的 graph 自指
  UT_LIST_INIT(fork->thrs);
  return (fork);
}

// que0que.cc:233
que_thr_t *que_thr_create(que_fork_t *parent, mem_heap_t *heap,
                          row_prebuilt_t *prebuilt) {
  que_thr_t *thr = static_cast<que_thr_t *>(mem_heap_zalloc(heap, sizeof(*thr)));
  thr->graph = parent->graph;             // ★ 直接继承根的 graph（→ trx）
  thr->common.parent = parent;
  thr->magic_n = QUE_THR_MAGIC_N;
  thr->common.type = QUE_NODE_THR;
  thr->state = QUE_THR_COMMAND_WAIT;
  thr->lock_state = QUE_THR_LOCK_NOLOCK;
  thr->prebuilt = prebuilt;               // ★ prebuilt 挂在 thr 上，锁模块可反查
  UT_LIST_ADD_LAST(parent->thrs, thr);
  return (thr);
}

// pars0pars.cc:1725 —— graph 与 trx 的唯一绑定点
que_thr_t *pars_complete_graph_for_exec(que_node_t *node, trx_t *trx,
                                        mem_heap_t *heap,
                                        row_prebuilt_t *prebuilt) {
  que_fork_t *fork = que_fork_create(nullptr, nullptr, QUE_FORK_MYSQL_INTERFACE, heap);
  fork->trx = trx;                        // ★★ 就这一行把 trx 装进图
  que_thr_t *thr = que_thr_create(fork, heap, prebuilt);
  thr->child = node;
  if (node) que_node_set_parent(node, thr);
  trx->graph = nullptr;                   // ★ MySQL 接口路径不用 trx->graph（那是内部 SQL 用的）
  return (thr);
}
```

#### 节点类型清单

`que_thr_step()`（`que0que.cc:923`）中的 switch 分发列出了所有节点类型：

| 节点类型 | 类型常量 | step 函数 | 说明 |
|----------|----------|-----------|------|
| UPDATE | `QUE_NODE_UPDATE` | `row_upd_step` | 更新行 |
| INSERT | `QUE_NODE_INSERT` | `row_ins_step` | 插入行 |
| SELECT | `QUE_NODE_SELECT` | `row_sel_step` | 查询行 |
| UNDO | `QUE_NODE_UNDO` | `row_undo_step` | 回滚行（回滚图的两层结构见 [`trx.md`](trx.md)「回滚与查询图」） |
| PURGE | `QUE_NODE_PURGE` | `row_purge_step` | 清理 undo |
| ASSIGNMENT | `QUE_NODE_ASSIGNMENT` | `assign_step` | 赋值 |
| IF | `QUE_NODE_IF` | `if_step` | 条件分支 |
| WHILE | `QUE_NODE_WHILE` | `while_step` | 循环 |
| FOR | `QUE_NODE_FOR` | `for_step` | 循环 |
| PROC | `QUE_NODE_PROC` | `proc_step` | 过程容器 |
| FETCH | `QUE_NODE_FETCH` | `fetch_step` | 游标取值 |
| COMMIT | `QUE_NODE_COMMIT` | `trx_commit_step` | 事务提交 |
| FUNC | `QUE_NODE_FUNC` | `proc_eval_step` | 函数求值 |

⚠️ **勘误**：
- **`QUE_NODE_*` 不是 enum**，是一组 `constexpr uint32_t`（`que0que.h:376`，带原注释 `/* TODO: Query graph node types */`）
- **不存在 `QUE_NODE_DELETE`，也不存在 `struct del_node_t`**——枚举值 **3 从未被使用**；DELETE 由 `upd_node_t::is_delete`（`row0upd.h:705`）实现，注释原文 "Update node structure **which also implements the delete operation**"
- `del_node_t` 只有 `row0types.h:42` 一行前向声明，**全仓无定义**
- **`step` 函数返回 `que_thr_t*` 不是 `void`**——只是 `que_thr_step()` 里部分调用的返回值被丢弃

**完整常量表**（`que0que.h:372`）：

```cpp
/* Flag which is ORed to control structure statement node types */
constexpr uint32_t QUE_NODE_CONTROL_STAT = 1024;   // ★★ 位标志 0x400，不是枚举值

/* TODO: Query graph node types */
constexpr uint32_t QUE_NODE_LOCK = 1;
constexpr uint32_t QUE_NODE_INSERT = 2;
constexpr uint32_t QUE_NODE_UPDATE = 4;    // ★ 从 2 跳到 4：值 3 从未被使用（DELETE 已并入 UPDATE）
constexpr uint32_t QUE_NODE_CURSOR = 5;
constexpr uint32_t QUE_NODE_SELECT = 6;
constexpr uint32_t QUE_NODE_AGGREGATE = 7;
constexpr uint32_t QUE_NODE_FORK = 8;
constexpr uint32_t QUE_NODE_THR = 9;
constexpr uint32_t QUE_NODE_UNDO = 10;
constexpr uint32_t QUE_NODE_COMMIT = 11;
constexpr uint32_t QUE_NODE_ROLLBACK = 12;
constexpr uint32_t QUE_NODE_PURGE = 13;
constexpr uint32_t QUE_NODE_SYMBOL = 14;
constexpr uint32_t QUE_NODE_RES_WORD = 15;
constexpr uint32_t QUE_NODE_FUNC = 16;
constexpr uint32_t QUE_NODE_ORDER = 17;
constexpr uint32_t QUE_NODE_PROC = 18 + QUE_NODE_CONTROL_STAT;   // 1042
constexpr uint32_t QUE_NODE_IF = 19 + QUE_NODE_CONTROL_STAT;     // 1043
constexpr uint32_t QUE_NODE_WHILE = 20 + QUE_NODE_CONTROL_STAT;  // 1044
constexpr uint32_t QUE_NODE_ASSIGNMENT = 21;                     // ★ 不带标志位
constexpr uint32_t QUE_NODE_FETCH = 22;
constexpr uint32_t QUE_NODE_OPEN = 23;
constexpr uint32_t QUE_NODE_COL_ASSIGNMENT = 24;
constexpr uint32_t QUE_NODE_FOR = 25 + QUE_NODE_CONTROL_STAT;    // 1049
constexpr uint32_t QUE_NODE_RETURN = 26;                         // ★ 不带标志位
constexpr uint32_t QUE_NODE_ELSIF = 27;
constexpr uint32_t QUE_NODE_CALL = 28;
constexpr uint32_t QUE_NODE_EXIT = 29;
```

> ★★ **带 `QUE_NODE_CONTROL_STAT` 的只有 4 个：PROC / IF / WHILE / FOR**。所以 `que_thr_step()` 的 `if (type & QUE_NODE_CONTROL_STAT)` **只覆盖这 4 类**；`ASSIGNMENT / FETCH / OPEN / RETURN / EXIT / COL_ASSIGNMENT` 是**不带标志**的普通语句节点，走 else-if 链。

**各节点结构的"首字段是 common"证据**（多态契约的落点）：

```cpp
// row0ins.h:162
struct ins_node_t {
  que_common_t common;     /*!< node type: QUE_NODE_INSERT */   // ★★ 首字段
  ulint ins_type;          // INS_VALUES / INS_SEARCHED / INS_DIRECT
  dtuple_t *row;
  dict_table_t *table;
  sel_node_t *select;      // INSERT ... SELECT 的子查询
  ...
  UT_LIST_BASE_NODE_T(dtuple_t, tuple_list) entry_list;  // ★ 每个索引一个 entry tuple
  mem_heap_t *entry_sys_heap;
  uint32_t ins_multi_val_pos;   // 锁等待时记住多值字段进度
  ulint magic_n;
};

// row0upd.h:703 —— ★ DELETE 就是它
/* Update node structure which also implements the delete operation
of a row */
struct upd_node_t {
  que_common_t common;     /*!< node type: QUE_NODE_UPDATE */
  bool is_delete;          /* true if delete, false if update */   // ★★
  bool searched_update;    // true=UPDATE ... WHERE；false=游标定位更新
  bool in_mysql_interface; // true 表示为 MySQL 接口创建
  dict_foreign_t *foreign;
  upd_node_t *cascade_node;   // 外键级联子图
  mem_heap_t *cascade_heap;
  sel_node_t *select;
  btr_pcur_t *pcur;
  upd_t *update;
  ...
  uint32_t del_multi_val_pos;
  uint32_t upd_multi_val_pos;
};

// row0sel.h:330
struct sel_node_t {
  que_common_t common;       /*!< node type: QUE_NODE_SELECT */
  enum sel_node_state state;
  que_node_t *select_list;
  sym_node_t *into_list;
  plan_t *plans;             // 内部 SQL 的执行计划（MySQL 接口下不用）
  ReadView *read_view;       // 非锁定一致性读时的视图
  ...
};

// row0undo.h:101 / row0purge.h:91 —— 同样是首字段
struct undo_node_t  { que_common_t common; enum undo_exec state; ... };
struct purge_node_t { que_common_t common; ... void init(); void deinit(); };
```

> ★ `purge_node_t` 在 8.0 已经**半 C++ 化**——带成员函数 `init()` / `deinit()`，与纯 C 风格的 `ins_node_t` 形成对比。

**释放：另一张手写的"虚函数表"**（`que_graph_free_recursive()`，`que0que.cc:442`）：

```cpp
void que_graph_free_recursive(que_node_t *node) {
  if (node == nullptr) return;
  switch (que_node_get_type(node)) {      // ★★ 基于 type 的手写"虚析构"
    case QUE_NODE_FORK:
      for (auto thr = UT_LIST_GET_FIRST(fork->thrs); thr; thr = UT_LIST_GET_NEXT(thrs, thr))
        que_graph_free_recursive(thr);
      break;
    case QUE_NODE_THR:
      ut_a(thr->magic_n == QUE_THR_MAGIC_N);
      thr->magic_n = QUE_THR_MAGIC_FREED;      // ★ 防重复释放 / 内存破坏检测
      que_graph_free_recursive(thr->child);
      break;
    case QUE_NODE_UNDO:   mem_heap_free(undo->heap); break;
    case QUE_NODE_SELECT: sel_node_free_private(sel); break;
    case QUE_NODE_INSERT:
      que_graph_free_recursive(ins->select);   // 递归释放 INSERT...SELECT 子查询
      ins->select = nullptr;
      if (ins->entry_sys_heap != nullptr) { mem_heap_free(ins->entry_sys_heap); ins->entry_sys_heap = nullptr; }
      break;
    case QUE_NODE_PURGE: purge->deinit(); break;
    case QUE_NODE_UPDATE:
      if (upd->update) upd->update->free_per_stmt_heap();
      if (upd->in_mysql_interface) { btr_pcur_t::free_for_mysql(upd->pcur); upd->in_mysql_interface = false; }
      que_graph_free_recursive(upd->cascade_node);    // ★ 外键级联子图
      if (upd->cascade_heap) { mem_heap_free(upd->cascade_heap); upd->cascade_heap = nullptr; }
      que_graph_free_recursive(upd->select);
      if (upd->heap != nullptr) { mem_heap_free(upd->heap); upd->heap = nullptr; }
      break;
    case QUE_NODE_PROC:  que_graph_free_stat_list(((proc_node_t *)node)->stat_list); break;
    case QUE_NODE_IF:
      que_graph_free_stat_list(((if_node_t *)node)->stat_list);
      que_graph_free_stat_list(((if_node_t *)node)->else_part);
      que_graph_free_stat_list(((if_node_t *)node)->elsif_list);
      break;
    case QUE_NODE_ELSIF: que_graph_free_stat_list(((elsif_node_t *)node)->stat_list); break;
    case QUE_NODE_WHILE: que_graph_free_stat_list(((while_node_t *)node)->stat_list); break;
    case QUE_NODE_FOR:   que_graph_free_stat_list(((for_node_t *)node)->stat_list); break;
    case QUE_NODE_ASSIGNMENT: case QUE_NODE_EXIT: case QUE_NODE_RETURN:
    case QUE_NODE_COMMIT: case QUE_NODE_ROLLBACK: case QUE_NODE_LOCK:
    case QUE_NODE_FUNC: case QUE_NODE_ORDER: case QUE_NODE_OPEN: case QUE_NODE_FETCH:
      /* No need to do anything */
      break;
    default: ut_error;
  }
}

/** Frees a query graph. */
void que_graph_free(que_t *graph) {
  if (graph->sym_tab) {
    /* The following call frees dynamic memory allocated for variables etc.
    during execution. Frees also explicit cursor definitions. */
    sym_tab_free_private(graph->sym_tab);   // ★ 显式游标声明在这里释放，不在 recursive 里
  }
  if (graph->info && graph->info->graph_owns_us) pars_info_free(graph->info);
  que_graph_free_recursive(graph);
  mem_heap_free(graph->heap);               // ★★ 额外释放 heap —— 只用于"图独占 heap"的一次性图
}
```

> ★★ **`que_graph_free_recursive()` 不释放 heap，`que_graph_free()` 才释放**。prebuilt 的三个图与 `prebuilt->heap` 共享，所以只能用前者；一次性图（如 `lock_table_for_trx()`）才用后者。
>
> ⚠️ **不存在通用 `que_node_create()`**——改为各类型专属创建函数：`ins_node_create` / `upd_node_create` / `sel_node_create` / `row_undo_node_create` / `row_purge_node_create` / `roll_node_create` / `trx_commit_node_create`。

#### 设计模式

这是经典的 **Interpreter Pattern**（解释器模式），具体实现为基于状态机的协程式执行：node 是数据（做什么），thr 是控制流（谁在做、下一步做什么），分离后 thr 可在不同 node 间流转，node 状态在锁等待挂起/恢复之间保持。

#### 回滚图：模型的一个特殊应用

节点清单里的 `UNDO`（`QUE_NODE_UNDO`）不是孤立存在的——它属于**回滚图**，这是 query graph 执行模型的一个特殊应用：**回滚不是普通函数调用，而是构造并解释执行两层 graph**：

```
外层 fork（QUE_FORK_MYSQL_INTERFACE）
└── roll_node_t（QUE_NODE_ROLLBACK）     ← 回滚"命令"节点，trx_rollback_step 驱动
    └── undo_thr ──> 内层 fork（QUE_FORK_ROLLBACK）
        └── undo_node（QUE_NODE_UNDO）    ← 真正逐行撤销，row_undo_step 驱动
```

两层结构、三种 fork type（`QUE_FORK_MYSQL_INTERFACE` / `QUE_FORK_ROLLBACK` / `QUE_FORK_RECOVERY`）的分工、与 DML query graph 的异同（动态构建 vs 预编译缓存），详见 [`trx.md`](trx.md)「回滚机制 → 回滚与查询图」。

这里只需记住一个与 thr 状态机的呼应：**`QUE_FORK_ROLLBACK` 类型的 fork 在 `que_thr_stop` 里走 `SUSPENDED` 分支**（见下文 thr 状态机分支 [4]）——内层 undo graph 的 thr 挂起，正是等外层 roll_node 调度它继续。

#### 控制语句与 brother 链遍历

`que_common_t` 中的 `brother` 字段构成**兄弟链**，控制语句（IF/WHILE/FOR/PROC）靠它顺序执行子语句。

#### PROC：过程容器

`proc_step()`（`eval0proc.ic:40`）：

```cpp
static inline que_thr_t *proc_step(que_thr_t *thr) {
  node = static_cast<proc_node_t *>(thr->run_node);

  if (thr->prev_node == que_node_get_parent(node)) {
    /* 从父节点下来 = 第一次进入过程：从第一条语句开始 */
    thr->run_node = node->stat_list;
  } else {
    /* 从子语句返回：next 应为 NULL（已处理完所有语句） */
    ut_ad(que_node_get_next(thr->prev_node) == nullptr);
    thr->run_node = nullptr;
  }

  /* run_node 为 NULL 说明所有语句执行完 → 回到父节点 */
  if (thr->run_node == nullptr) {
    thr->run_node = que_node_get_parent(node);
  }

  return (thr);
}
```

**`prev_node == parent` 判断语义**：这是查询图执行的核心模式。如果"上一个执行的节点"就是"当前节点的父节点"，说明控制流是**第一次从上层进入**这个节点；否则说明是**从子语句返回**。

#### WHILE：循环

`while_step()`（`eval0proc.cc:107`）：

```cpp
que_thr_t *while_step(que_thr_t *thr) {
  node = static_cast<while_node_t *>(thr->run_node);

  ut_ad((thr->prev_node == que_node_get_parent(node)) ||
        (que_node_get_next(thr->prev_node) == nullptr));

  eval_exp(node->cond);                       // 求值循环条件

  if (eval_node_get_bool_val(node->cond)) {
    thr->run_node = node->stat_list;          // 条件为真 → 进入循环体
  } else {
    thr->run_node = que_node_get_parent(node); // 条件为假 → 退出到父节点
  }

  return (thr);
}
```

WHILE 循环的实现：每次回到 while_node 时重新求值条件，为真则 `run_node = stat_list` 重新进入循环体，为假则 `run_node = parent` 跳出循环。**循环体的最后一条语句执行完后，`que_thr_step` 的控制语句分支会把 `run_node` 设回 while_node**（因为最后一条语句 `next == nullptr`，会走到 `else if (type == QUE_NODE_WHILE)` 分支再次求值条件）。

#### IF：条件分支

`if_step()`（`eval0proc.cc:40`）：

```cpp
que_thr_t *if_step(que_thr_t *thr) {
  node = static_cast<if_node_t *>(thr->run_node);

  if (thr->prev_node == que_node_get_parent(node)) {
    eval_exp(node->cond);                     // 第一次进入：求值条件

    if (eval_node_get_bool_val(node->cond)) {
      thr->run_node = node->stat_list;        // 条件为真 → then 分支
    } else if (node->else_part) {
      thr->run_node = node->else_part;        // 有 else → else 分支
    } else if (node->elsif_list) {
      elsif_node = node->elsif_list;
      for (;;) {
        eval_exp(elsif_node->cond);
        if (eval_node_get_bool_val(elsif_node->cond)) {
          thr->run_node = elsif_node->stat_list;  // elsif 条件为真
          // ...
        }
      }
    }
  }
  return (thr);
}
```

IF 同样用 `prev_node == parent` 判断是否第一次进入——只有第一次进入才求值条件，从分支返回时不再重复求值。

#### 控制语句的三段式结构

所有控制语句都遵循这个模式：

```
1. 进入判断：prev_node == parent ? 第一次进入 : 从子语句返回
2. 分支选择：根据条件/状态设置 run_node（进入子语句 / 回到父节点）
3. 兄弟推进：子语句执行完且还有兄弟 → run_node = next(brother)
```

`que_thr_step` 开头的控制语句分支就是第 3 步的实现：

```cpp
if (type & QUE_NODE_CONTROL_STAT) {
    if ((thr->prev_node != que_node_get_parent(node)) &&
        que_node_get_next(thr->prev_node)) {
      /* 从子语句返回且有下一个兄弟 → 执行兄弟 */
      thr->run_node = que_node_get_next(thr->prev_node);
    } else {
      /* 否则进入控制语句自己的 step 函数（if_step/while_step/...） */
    }
}
```

**注意**：控制语句的 step 函数（`if_step`/`while_step`/`for_step`/`proc_step`）**返回 void**，在函数内部直接修改 `thr->run_node`；而 DML 的 step 函数（`row_upd_step`/`row_ins_step`/`row_sel_step`）**返回 `que_thr_t*`**，可以返回不同的 thr（虽然实际返回原 thr 或 nullptr）。

⚠️ **勘误**：上面"返回 void"的说法**不准**——**所有 `*_step()` 都返回 `que_thr_t*`**，只是 `que_thr_step()` 里调用控制语句 step 时把返回值**丢弃**了（`if_step(thr);` 而不是 `thr = if_step(thr);`）。

⚠️ **位置勘误**：`if_step/while_step/for_step/assign_step` **不在 `que0que.cc`**，在 **`eval/eval0proc.cc`**；`proc_step/proc_eval_step` 在 `include/eval0proc.ic`；`fetch_step` 在 `row/row0sel.cc:2208`。`que0que.cc` 只是**调用方**。

**`if_step()` 全文**（`eval0proc.cc:38`）：

```cpp
que_thr_t *if_step(que_thr_t *thr) {
  if_node_t *node = static_cast<if_node_t *>(thr->run_node);
  ut_ad(que_node_get_type(node) == QUE_NODE_IF);

  if (thr->prev_node == que_node_get_parent(node)) {   // ★ 从父节点下来 = 第一次进入
    /* Evaluate the condition */
    eval_exp(node->cond);

    if (eval_node_get_bool_val(node->cond)) {
      /* The condition evaluated to true: start execution
      from the first statement in the statement list */
      thr->run_node = node->stat_list;                 // ★ 跳进 THEN 语句链首条
    } else if (node->else_part) {
      thr->run_node = node->else_part;
    } else if (node->elsif_list) {
      elsif_node_t *elsif_node = node->elsif_list;
      for (;;) {
        eval_exp(elsif_node->cond);
        if (eval_node_get_bool_val(elsif_node->cond)) {
          thr->run_node = elsif_node->stat_list;
          break;
        }
        elsif_node = static_cast<elsif_node_t *>(que_node_get_next(elsif_node));
        //                                        ★ brother 链遍历 ELSIF
        if (elsif_node == nullptr) { thr->run_node = nullptr; break; }
      }
    } else {
      thr->run_node = nullptr;
    }
  } else {                                             // ★ 从子节点返回 = IF 已执行完
    /* Move to the next statement */
    ut_ad(que_node_get_next(thr->prev_node) == nullptr);
    thr->run_node = nullptr;
  }

  if (thr->run_node == nullptr) {
    thr->run_node = que_node_get_parent(node);   // ★ 回父亲，由父亲的 CONTROL_STAT 分支继续推 brother
  }
  return (thr);
}
```

**`while_step()` 全文**（`eval0proc.cc:105`）——比 IF 简单得多：

```cpp
que_thr_t *while_step(que_thr_t *thr) {
  while_node_t *node = static_cast<while_node_t *>(thr->run_node);
  ut_ad(que_node_get_type(node) == QUE_NODE_WHILE);
  ut_ad((thr->prev_node == que_node_get_parent(node)) ||
        (que_node_get_next(thr->prev_node) == nullptr));
  //     ★ 要么刚进入，要么循环体最后一条语句返回

  /* Evaluate the condition */
  eval_exp(node->cond);

  if (eval_node_get_bool_val(node->cond)) {
    /* The condition evaluated to true: start execution
    from the first statement in the statement list */
    thr->run_node = node->stat_list;              // ★ 条件为真 → 再跑一轮循环体
  } else {
    thr->run_node = que_node_get_parent(node);    // ★ 条件为假 → 出循环，回父节点
  }
  return (thr);
}
```

> ★★ **WHILE 每次回到 `while_step` 都重新求值条件**——这就是"循环体执行完后控制权回到 WHILE 节点"的机制：`que_thr_step` 的 CONTROL_STAT 分支发现"没有下一个兄弟"，于是再调一次 `while_step`，而此时 `prev_node` 是循环体最后一条语句，条件被重新求值。

**`for_step()` 全文**（`eval0proc.cc:157`）——唯一有提前 return 的：

```cpp
que_thr_t *for_step(que_thr_t *thr) {
  for_node_t *node = static_cast<for_node_t *>(thr->run_node);
  ut_ad(que_node_get_type(node) == QUE_NODE_FOR);
  que_node_t *parent = que_node_get_parent(node);

  if (thr->prev_node != parent) {              // ★ 循环体里返回（中间态）
    /* Move to the next statement */
    thr->run_node = que_node_get_next(thr->prev_node);
    if (thr->run_node != nullptr) {
      return (thr);                            // ★★ 循环体还有下一条语句 → 直接继续
    }
    /* Increment the value of loop_var */
    loop_var_value = 1 + eval_node_get_int_val(node->loop_var);
  } else {                                     // ★ 刚从父节点进入 → 初始化循环
    /* Initialize the loop */
    eval_exp(node->loop_start_limit);
    eval_exp(node->loop_end_limit);
    loop_var_value = eval_node_get_int_val(node->loop_start_limit);
    node->loop_end_value = (int)eval_node_get_int_val(node->loop_end_limit);
  }

  /* Check if we should do another loop */
  if (loop_var_value > node->loop_end_value) {
    /* Enough loops done */
    thr->run_node = parent;                    // ★ 越界 → 退出循环
  } else {
    eval_node_set_int_val(node->loop_var, loop_var_value);
    thr->run_node = node->stat_list;           // ★ 再跑一轮
  }
  return (thr);
}
```

**`proc_step()` / `proc_eval_step()`**（`include/eval0proc.ic:38`）：

```cpp
static inline que_thr_t *proc_step(que_thr_t *thr) {
  proc_node_t *node = static_cast<proc_node_t *>(thr->run_node);
  ut_ad(que_node_get_type(node) == QUE_NODE_PROC);

  if (thr->prev_node == que_node_get_parent(node)) {
    /* Start execution from the first statement in the statement list */
    thr->run_node = node->stat_list;          // ★ 进入过程体
  } else {
    ut_ad(que_node_get_next(thr->prev_node) == nullptr);
    thr->run_node = nullptr;
  }
  if (thr->run_node == nullptr) {
    thr->run_node = que_node_get_parent(node);  // ★ 过程结束 → 回父
  }
  return (thr);
}

static inline que_thr_t *proc_eval_step(que_thr_t *thr) {
  func_node_t *node = static_cast<func_node_t *>(thr->run_node);
  ut_ad(que_node_get_type(node) == QUE_NODE_FUNC);   // ★ 注意：处理的是 FUNC 不是 PROC
  eval_exp(node);
  thr->run_node = que_node_get_parent(node);
  return (thr);
}
```

**`assign_step()`**（`eval0proc.cc:135`）——最简单的叶子语句：

```cpp
que_thr_t *assign_step(que_thr_t *thr) {
  assign_node_t *node = static_cast<assign_node_t *>(thr->run_node);
  ut_ad(que_node_get_type(node) == QUE_NODE_ASSIGNMENT);
  eval_exp(node->val);                                // 求值右值
  eval_node_copy_val(node->var->alias, node->val);    // 写到符号表变量的 alias
  thr->run_node = que_node_get_parent(node);          // ★ 叶子语句：立即回父亲
  return (thr);
}
```

**`fetch_step()`（游标取值，动态改 parent 的巧技）**（`row0sel.cc:2208`）：

```cpp
que_thr_t *fetch_step(que_thr_t *thr) {
  fetch_node_t *node = static_cast<fetch_node_t *>(thr->run_node);
  sel_node_t *sel_node = node->cursor_def;
  ut_ad(que_node_get_type(node) == QUE_NODE_FETCH);

  if (thr->prev_node != que_node_get_parent(node)) {   // ★ 从 sel_node 取完一行返回
    if (sel_node->state != SEL_NODE_NO_MORE_ROWS) {
      if (node->into_list) {
        sel_assign_into_var_values(node->into_list, sel_node);
      } else {
        bool ret = (*node->func->func)(sel_node, node->func->arg);
        if (!ret) sel_node->state = SEL_NODE_NO_MORE_ROWS;
      }
    }
    thr->run_node = que_node_get_parent(node);
    return (thr);
  }

  /* Make the fetch node the parent of the cursor definition for
  the time of the fetch, so that execution knows to return to this
  fetch node after a row has been selected or we know that there is
  no row left */
  sel_node->common.parent = node;      // ★★★ 动态改 parent！临时把 fetch 设为 sel_node 的父

  if (sel_node->state == SEL_NODE_CLOSED) {
    ib::error(ER_IB_MSG_1027) << "fetch called on a closed cursor";
    thr_get_trx(thr)->error_state = DB_ERROR;
    return (nullptr);                  // ★ 返回 NULL = 线程终止（区别于返回 thr）
  }

  thr->run_node = sel_node;            // ★ 控制权交给 select 节点去取下一行
  return (thr);
}
```

> ★★ **`fetch_step` 是唯一一个在运行时修改图结构（`common.parent`）的 step**——"取一行就回到 fetch"这个控制流不是靠 step 判断，而是靠临时改写 parent 让既有机制自然生效。这是这套 C 风格图解释器里最"脏"但最省事的一招。

**`EXIT` 的一步跳出**（`que_thr_step()` 尾部 + `que_node_get_containing_loop_node()`）：

```cpp
// que_thr_step() 尾部
  if (type == QUE_NODE_EXIT) {
    old_thr->prev_node = que_node_get_containing_loop_node(node);
    //  ★★ 把 prev_node 伪造成"来自包含它的循环"，从而一步跳出整个循环
  } else {
    old_thr->prev_node = node;         // 通用：记录"控制流来自哪个节点"
  }
```

```cpp
// que0que.cc:844
que_node_t *que_node_get_containing_loop_node(que_node_t *node) {
  for (;;) {
    node = que_node_get_parent(node);   // ★ 沿 parent 链一路向上（不是 brother 链）
    if (!node) break;
    ulint type = que_node_get_type(node);
    if ((type == QUE_NODE_FOR) || (type == QUE_NODE_WHILE)) break;   // 只认 FOR / WHILE
  }
  return (node);
}
```

> ★ `old_thr` 的存在意义：子过程调用可能**换线程**，但 `prev_node` 仍要写在**旧 thr** 上——所以函数开头先 `old_thr = thr;` 存一份。
>
> ⚠️ `QUE_NODE_LOCK` 在 8.0.39 已无实现：`case QUE_NODE_LOCK: ut_error;`

---

### row_prebuilt_t 与查询图的关系

#### row_prebuilt_t 中的 graph 字段

`row_prebuilt_t` 是 MySQL handler 层与 InnoDB 之间的预编译结构。

⚠️ **勘误：是 3 个查询图字段，不是 4 个**——**不存在 `del_graph`**（全仓 0 命中）。DELETE 复用 `upd_graph`（源码注释原文 "used in updates **or deletes**"）。

```cpp
// row0mysql.h:649 / :669
  /** Innobase SQL insert node used to perform inserts to the table */
  ins_node_t *ins_node;
  byte *ins_upd_rec_buff;     // MySQL 格式 → InnoDB 格式的转换缓冲
  multi_value_data *mv_data;  // 多值虚拟列

  upd_node_t *upd_node;       /*!< Innobase SQL update node used
                              to perform updates and deletes */      // ★ DELETE 也用它

  trx_id_t trx_id;            /*!< The table->def_trx_id when
                              ins_graph was built */                // ★ ins_graph 的"版本号"

  que_fork_t *ins_graph;      /*!< Innobase SQL query graph used
                              in inserts. Will be rebuilt on
                              trx_id or n_indexes mismatch. */      // ★★ 注释即两个复用判据

  que_fork_t *upd_graph;      /*!< Innobase SQL query graph used
                              in updates or deletes */              // ★ 永不重建

  btr_pcur_t *pcur;           /*!< persistent cursor used in selects and updates */
  btr_pcur_t *clust_pcur;     /*!< persistent cursor used in some selects and updates */

  // create in: pars_complete_graph_for_exec
  que_fork_t *sel_graph;      /*!< dummy query graph used in selects */   // ★ dummy
```

**三个图的构建函数（真实名字）**：

| 图 | 构建函数 | 位置 | 复用判据 |
|---|---|---|---|
| `ins_graph` | **`row_get_prebuilt_insert_row()`**（`static`，故头文件无声明） | `row0mysql.cc:1054` | trx_id + 索引数 |
| `upd_graph` | **`row_get_prebuilt_update_vector()`** | `row0mysql.cc:1814` | 只有"建过没有" |
| `sel_graph` | **`row_prebuild_sel_graph()`** | `row0mysql.cc:1747` | 只有"建过没有" |

> ⚠️ **不存在 `row_create_graph_for_insert()` / `row_create_graph_for_update()` / `row_create_graph_for_delete()` / `row_create_graph_for_select()` / `row_get_prebuilt_ins_graph()`** —— 全仓 `row_create_graph` 与 `prebuilt_ins_graph` **均为 0 命中**。

**内存预算：三个图都在同一个 heap 上预分配**（`row0mysql.cc:831`）——这是 `row_prebuilt_free()` 只能用 `que_graph_free_recursive()`（不释放 heap）的原因：

```cpp
#define PREBUILT_HEAP_INITIAL_SIZE                                          \
  (sizeof(*prebuilt)                         /* allocd in this function */  \
   + DTUPLE_EST_ALLOC(search_tuple_n_fields) /* search_tuple */             \
   + DTUPLE_EST_ALLOC(search_tuple_n_fields) /* m_stop_tuple */             \
   + DTUPLE_EST_ALLOC(ref_len) /* allocd in row_prebuild_sel_graph() */     \
   + sizeof(sel_node_t) + sizeof(que_fork_t) +                              \
   sizeof(que_thr_t) /* allocd in row_get_prebuilt_update_vector() */       \
   + sizeof(upd_node_t) + sizeof(upd_t) +                                   \
   sizeof(upd_field_t) * table->get_n_cols() + sizeof(que_fork_t) +         \
   sizeof(que_thr_t)    /* allocd in row_get_prebuilt_insert_row() */       \
   + sizeof(ins_node_t) /* mysql_row_len could be huge and we are not       \
                        sure if this prebuilt instance is going to be       \
                        used in inserts */                                  \
   + (mysql_row_len < 256 ? mysql_row_len : 0) +                            \
   DTUPLE_EST_ALLOC(table->get_n_cols() + dict_table_get_n_v_cols(table)) + \
   sizeof(que_fork_t) + sizeof(que_thr_t) + sizeof(*prebuilt->pcur) +       \
   sizeof(*prebuilt->clust_pcur))
```

> ★ 注释里点名了三个构建函数，**正好对应三个 graph 字段，没有第四个**。

**handle 换 THD/TRX 时三个图都要改写 trx 指针**（`row_update_prebuilt_trx()`，`row0mysql.cc:1030`）：

```cpp
void row_update_prebuilt_trx(row_prebuilt_t *prebuilt, trx_t *trx) {
  prebuilt->trx = trx;
  if (prebuilt->ins_graph) prebuilt->ins_graph->trx = trx;
  if (prebuilt->upd_graph) prebuilt->upd_graph->trx = trx;
  if (prebuilt->sel_graph) prebuilt->sel_graph->trx = trx;
}
```

**释放**（`row_prebuilt_free()`，`row0mysql.cc:954`）：

```cpp
  if (prebuilt->ins_graph) que_graph_free_recursive(prebuilt->ins_graph);
  if (prebuilt->sel_graph) que_graph_free_recursive(prebuilt->sel_graph);
  if (prebuilt->upd_graph) que_graph_free_recursive(prebuilt->upd_graph);
  ...
  mem_heap_free(prebuilt->heap);   // ★ 最后一次性释放整个 heap；三个图的内存随之消失
```

> ★ 调用点：`ha_innobase::close()`（`ha_innodb.cc:7691`）、`ha_innopart.cc:1238`、`handler0alter.cc:7906`、C API `api0api.cc:998`。

**★ dummy `sel_graph` 为什么必须存在**（`row0sel.cc:4561` 与 `row0mysql.cc:1239` 的原注释）：

```cpp
    if (prebuilt->sel_graph == nullptr) {
      /* Build a dummy select query graph */
      row_prebuild_sel_graph(prebuilt);
    }
```

```cpp
  trx->op_info = "setting table lock";
  if (prebuilt->sel_graph == nullptr) {
    /* Build a dummy select query graph */
    row_prebuild_sel_graph(prebuilt);
  }
  /* We use the select query graph as the dummy graph needed
  in the lock module call */
  thr = que_fork_get_first_thr(prebuilt->sel_graph);
  que_thr_move_to_run_state_for_mysql(thr, trx);

run_again:
  thr->run_node = thr;                     // ★★ run_node 指向 thr 自己（不是 sel_node）
  thr->prev_node = thr->common.parent;     //    ⇒ 这个 thr 永不进入 que_thr_step
  trx_start_if_not_started_xa(trx, false, UT_LOCATION_HERE);
  err = lock_table(0, prebuilt->table,
                   static_cast<enum lock_mode>(prebuilt->select_lock_type), thr);
```

四个理由（均有源码支撑）：
1. **`lock_table()` 签名要 `que_thr_t *thr`** —— 没有 thr 就无法把 lock 挂进 `trx_locks`、也无法标识 `lock_t` 的归属 trx。
2. **锁等待要用 thr 做唤醒载体** —— `lock_wait_suspend_thread(thr)`（`row0mysql.cc:735`）把 thr 挂进等待队列，唤醒后从 `run_again` 继续。
3. **`n_active_thrs` 计数自洽** —— `que_thr_move_to_run_state_for_mysql()` 做 `trx->lock.n_active_thrs++`；`que_thr_stop_for_mysql()` 里有 `ut_ad(trx->lock.n_active_thrs == 1)`，**必须**先有一个 thr。
4. **表锁路径没有可执行节点**，所以 `thr->run_node = thr` 指向自己而非 node。

> ★★ **`row_search_mvcc` 从不对 `sel_graph` 调 `que_run_threads` / `que_thr_step`**——8.0 的 `row_search_mvcc` 已是手写状态机（含 `fetch_cache`）。dummy graph 只提供 `que_thr_t` 上下文，让底层 `lock_clust_rec_read_check_and_set` / `row_ins_set_exclusive_rec_lock` 这些**仍要 `que_thr_t*` 参数**的锁函数能工作。
>
> ★ 官方自己承认这是补丁 —— `que_thr_stop_for_mysql_no_error()` 的注释原文："A patch for MySQL used to 'stop' a **dummy** query thread used in MySQL select"。
>
> ⚠️ **另一类 dummy graph 是"现造现销"的**（用完连 heap 一起 `que_graph_free()`）：`row_mysql_lock_table()`（`row0mysql.cc:3605`）、`lock_table_for_trx()`（`lock0lock.cc:3957`）——它们 `mem_heap_create(512)` + `sel_node_create()` + `pars_complete_graph_for_exec()`，用完即弃，**不挂在 prebuilt 上**。
>
> ⚠️ **`row_mysql_lock_data_dictionary()` 与 query graph 完全无关**——它压的是 `dict_operation_lock`（rw_lock）+ `dict_sys mutex`，不是 lock_sys 的表锁，不需要 thr。常与 `row_mysql_lock_table()` 混淆。

#### 各 graph 的使用场景

| graph 字段 | 使用场景 | 是否真执行 |
|-----------|---------|-----------|
| `sel_graph` | 锁系统参数载体（row_lock_table） | 否（dummy） |
| `ins_graph` | `row_insert_for_mysql_using_ins_graph()` | 是 |
| `upd_graph` | `row_update_for_mysql()` / `row_delete_for_mysql()` | 是 |

#### 缓存复用与失效

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

**真实源码**（`row0mysql.cc:1050`，函数名是 `row_get_prebuilt_insert_row`，是 `static`）：

```cpp
/** Gets pointer to a prebuilt dtuple used in insertions. If the insert graph
 has not yet been built in the prebuilt struct, then this function first
 builds it.
 @return prebuilt dtuple; the column type information is also set in it */
static dtuple_t *row_get_prebuilt_insert_row(row_prebuilt_t *prebuilt) {
  dict_table_t *table = prebuilt->table;
  ut_ad(prebuilt && table && prebuilt->trx);

  if (prebuilt->ins_node != nullptr) {
    prebuilt->ins_node->ins_multi_val_pos = 0;

    /* Check if indexes have been dropped or added and we
    may need to rebuild the row insert template. */
    if (prebuilt->trx_id == table->def_trx_id &&                   // ★ 判据一
        UT_LIST_GET_LEN(prebuilt->ins_node->entry_list) ==
            UT_LIST_GET_LEN(table->indexes)) {                     // ★ 判据二
      return (prebuilt->ins_node->row);                            // ★★ 复用：零分配
    }

    ut_ad(prebuilt->trx_id < table->def_trx_id);
    //  ★ def_trx_id 只增不减：一旦不等，必然是"表被 DDL 改过索引"

    que_graph_free_recursive(prebuilt->ins_graph);   // ★ 只释放节点，不动 heap（共享）
    prebuilt->ins_graph = nullptr;
  }

  /* Create an insert node and query graph to the prebuilt struct */
  ins_node_t *node = ins_node_create(INS_DIRECT, table, prebuilt->heap);
  prebuilt->ins_node = node;
  ...
  dtuple_t *row = dtuple_create_with_vcol(prebuilt->heap, table->get_n_cols(),
                                          dict_table_get_n_v_cols(table));
  dict_table_copy_types(row, table);
  ins_node_set_new_row(node, row);

  prebuilt->ins_graph = static_cast<que_fork_t *>(
      que_node_get_parent(pars_complete_graph_for_exec(
          node, prebuilt->trx, prebuilt->heap, prebuilt)));

  prebuilt->ins_graph->state = QUE_FORK_ACTIVE;
  prebuilt->trx_id = table->def_trx_id;      // ★★ 记下"版本号"
  return (prebuilt->ins_node->row);
}
```

**谁去 bump `def_trx_id` 从而使缓存失效**（`ddl/ddl0ddl.cc:417`）：

```cpp
/** Invalidate all row_prebuilt_t::ins_graph that are referring
to this table. That is, force row_get_prebuilt_insert_row() to
rebuild prebuilt->ins_node->entry_list). */
static void drop_secondary_indexes(trx_t *trx, dict_table_t *table) noexcept {
  ut_ad(table->def_trx_id <= trx->id);
  table->def_trx_id = trx->id;   // ★★ 推进到当前 trx->id → 所有 handle 的 ins_graph 一次性失效
```

> ★★ **为什么 `upd_graph` 没有这套校验**：`ins_node->entry_list` 是**按索引数量预先分配**的 tuple 链表，ADD/DROP INDEX 必须重建；而 `upd_node` 的 update vector 是每次 UPDATE 由 `ha_innobase::update_row` 重新填 `upd_field_t` 的，节点本身与索引结构无关（索引维护在 `btr_cur_pessimistic_update` 内部）。所以 `upd_graph` 的判据只有 `if (prebuilt->upd_node == nullptr)` 一条。

**`ins_graph` 的两个调用点**：

| 调用点 | 位置 | 目的 |
|---|---|---|
| `row_insert_for_mysql_using_ins_graph()` | `row0mysql.cc:1590` | 真正执行 INSERT |
| `row_lock_table_autoinc_for_mysql()` | `row0mysql.cc:1170` | 借 ins_graph 当 dummy 去拿 AUTO_INC 表锁 |

```cpp
// row_insert_for_mysql_using_ins_graph() 关键段
  row_get_prebuilt_insert_row(prebuilt);      // ★ 建图 / 复用判据的唯一入口
  node = prebuilt->ins_node;
  row_mysql_convert_row_to_innobase(node->row, prebuilt, mysql_rec, &temp_heap);

  savept = trx_savept_take(trx);
  thr = que_fork_get_first_thr(prebuilt->ins_graph);   // ★ 取 ins_graph 上唯一那条 thr

  if (prebuilt->sql_stat_start) {
    node->state = INS_NODE_SET_IX_LOCK;      // ★ 语句首行：先拿表级 IX 锁
    prebuilt->sql_stat_start = false;
  } else {
    node->state = INS_NODE_ALLOC_ROW_ID;     // ★ 同语句第二行起：直接分配 row id
  }

  que_thr_move_to_run_state_for_mysql(thr, trx);

run_again:
  thr->run_node = node;
  thr->prev_node = node;
  row_ins_step(thr);                         // ★★ 直接调 step，不走 que_run_threads

  err = trx->error_state;
  if (err != DB_SUCCESS) {
error_exit:
    que_thr_stop_for_mysql(thr);
    thr->lock_state = QUE_THR_LOCK_ROW;
    auto was_lock_wait = row_mysql_handle_errors(&err, trx, thr, &savept);
    thr->lock_state = QUE_THR_LOCK_NOLOCK;
    if (was_lock_wait) {
      ut_ad(node->state == INS_NODE_INSERT_ENTRIES ||
            node->state == INS_NODE_ALLOC_ROW_ID);
      goto run_again;                        // ★★ 锁等待解决后原地续跑
    }
    return (err);
  }
  que_thr_stop_for_mysql_no_error(thr, trx);
```

> ★★ **注意 `row_ins_step(thr)` 是直接调用的，没有 `que_run_threads()`**——MySQL 接口路径只"借用"查询图的数据结构（`ins_node` 状态机 + `ins_graph` 的 thr 上下文），而**不使用查询图调度器**。调度由 `run_again` 这个 C 层 goto 循环完成。这与 purge（`que_run_threads` 真的跑）形成鲜明对比。
>
> ★ `node->state` 三态：`INS_NODE_SET_IX_LOCK`(1) → `INS_NODE_ALLOC_ROW_ID`(2) → `INS_NODE_INSERT_ENTRIES`(3)。锁等待后回到 `run_again` 时 state 保持不变，所以能**从断点继续**而不是重头再来——这正是"node 是状态节点"的意义。

---

### SELECT 不走查询图

#### SELECT 的实际执行路径

SELECT **不走查询图**。`row_search_mvcc()` 和 `row_search_no_mvcc()` 是独立的搜索函数，直接操作 B+树和持久化游标：

```
MySQL SQL 层 → ha_innobase::general_fetch()     // ha_innodb.cc:10524
  → row_search_mvcc(buf, ..., m_prebuilt, ...)    // 直接搜索，不走查询图
  → row_search_no_mvcc(buf, ..., m_prebuilt, ...) // 同上
```

`sel_graph` 只是给锁系统借 thr 用的 dummy（如 `row_lock_table()`），不是执行 SELECT 的载体。

#### 为什么 SELECT 不走查询图

查询图执行模型是为**锁等待挂起/恢复**设计的。SELECT 在 InnoDB 中通常不需要锁等待（MVCC 读不加锁），只有在 `SELECT ... FOR UPDATE` / `SELECT ... LOCK IN SHARE MODE` 时才加锁。即使加锁，`row_search_mvcc()` 内部直接调用 `sel_set_rec_lock()` 处理锁等待，不需要 query graph 的挂起/恢复机制。

INSERT / UPDATE / DELETE 必须走查询图，因为它们可能遇到锁冲突（唯一性检查、gap lock 等），需要锁等待挂起/恢复。

---

### 执行调度器：que_run_threads 与 que_thr_step

#### 入口：que_run_threads

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

#### 内层：que_run_threads_low 完整循环与 next_thr 机制

`que_run_threads_low()`（`que0que.cc:1024`）是实际的执行循环，包含关键的 **next_thr 切换机制**：

```cpp
static void que_run_threads_low(que_thr_t *thr) {
  trx = thr_get_trx(thr);

  do {
    log_free_check();                        // [1] 检查 redo 空间是否充足

    next_thr = que_thr_step(thr);            // [2] 执行一步，可能返回不同 thr

    trx_mutex_enter(trx);

    if (next_thr != thr) {                   // [3] thr 变了
      ut_a(next_thr == nullptr);             //     实际上只会是 nullptr

      /* 减引用计数；若锁等待已结束，可能返回新的可运行 thr */
      que_thr_dec_refer_count(thr, &next_thr);

      if (next_thr != nullptr) {
        thr = next_thr;                      //     切换到新 thr
      }
    }

    trx_mutex_exit(trx);

  } while (next_thr != nullptr);             // [4] nullptr 即退出循环
}
```

三个关键点：

- **[1] `log_free_check()`**：每执行一步前检查 redo log 是否有足够空间，避免执行到一半因 redo 满而失败。
- **[2] `next_thr = que_thr_step(thr)`**：`que_thr_step` 返回值可能是**不同的 thr**（注释说"if, e.g., a subprocedure call is made"），但 `ut_a(next_thr == nullptr)`（第 1061 行）断言表明：实际上 `que_thr_step` 只会返回 nullptr（表示 thr 挂起/结束）或原 thr。
- **[3] `que_thr_dec_refer_count()`**：这是 thr 切换的唯一入口。它调用 `que_thr_stop(thr)` 尝试停止当前 thr，如果 `que_thr_stop` 返回 false（说明挂起/等待的原因已消失——锁已释放），就把 `trx->error_state = DB_SUCCESS` 并 `*next_thr = thr`，让循环继续执行这个 thr（**不切换**）。

正是因为 `que_thr_step` 实际只返回 nullptr 或原 thr，且 `que_thr_dec_refer_count` 只在"等待已结束"时返回原 thr，**永远不会真正切换到另一个 thr**。这是"为什么不 thr = thr->next 切换"的代码级答案——虽然框架设计了切换能力，但 MySQL 路径下每个 fork 只有 1 个活跃的 thr，没有可切换的目标。

#### que_thr_dec_refer_count 完整逻辑

`que_thr_dec_refer_count()`（`que0que.cc:712`）是唯一能减引用计数的地方（另一个是 `que_thr_stop_for_mysql`），且**只能从 `que_run_threads` 内部调用**：

```cpp
static void que_thr_dec_refer_count(que_thr_t *thr, que_thr_t **next_thr) {
  trx = thr_get_trx(thr);
  ut_a(thr->is_active);
  ut_ad(trx_mutex_own(trx));

  if (thr->state == QUE_THR_RUNNING) {
    // 尝试停止 thr；返回 false 说明停止原因已消失
    if (!que_thr_stop(thr)) {
      ut_a(next_thr != nullptr && *next_thr == nullptr);

      /* The reason for the thr suspension or wait was
      already canceled before we came here: continue
      running the thread. */

      trx->error_state = DB_SUCCESS;   // 必须在这里重置，否则没人重置
      *next_thr = thr;                 // 继续运行原 thr

      return;                          // ← 注意：这里直接 return，不减引用
    }
  }

  fork = static_cast<que_fork_t *>(thr->common.parent);

  --trx->lock.n_active_thrs;   // 事务活跃 thr 数减 1
  --fork->n_active_thrs;       // fork 活跃 thr 数减 1
  thr->is_active = false;
}
```

**注意**：如果 `que_thr_stop` 返回 false（等待已结束），函数直接 `return` **不减引用计数**，因为 thr 还要继续运行。

#### que_thr_step 的完整分发逻辑

`que_thr_step()`（`que0que.cc:925`）按节点类型分发，并维护 `prev_node`：

```cpp
static inline que_thr_t *que_thr_step(que_thr_t *thr) {
  trx = thr_get_trx(thr);
  ut_ad(thr->state == QUE_THR_RUNNING);
  ut_a(trx->error_state == DB_SUCCESS);

  thr->resource++;                 // 资源计数（用于并发控制）

  node = thr->run_node;
  type = que_node_get_type(node);
  old_thr = thr;

  if (type & QUE_NODE_CONTROL_STAT) {          // [控制语句]
    if ((thr->prev_node != que_node_get_parent(node)) &&
        que_node_get_next(thr->prev_node)) {
      // 从子语句返回且有下一个兄弟语句 → 执行下一个兄弟
      thr->run_node = que_node_get_next(thr->prev_node);
    } else if (type == QUE_NODE_IF) {
      if_step(thr);
    } else if (type == QUE_NODE_FOR) {
      for_step(thr);
    } else if (type == QUE_NODE_PROC) {
      if (thr->prev_node == que_node_get_parent(node)) {
        trx->last_sql_stat_start.least_undo_no = trx->undo_no;
      }
      proc_step(thr);
    } else if (type == QUE_NODE_WHILE) {
      while_step(thr);
    } else {
      ut_error;
    }
  } else if (type == QUE_NODE_ASSIGNMENT) {
    assign_step(thr);
  } else if (type == QUE_NODE_SELECT) {
    thr = row_sel_step(thr);
  } else if (type == QUE_NODE_INSERT) {
    thr = row_ins_step(thr);
  } else if (type == QUE_NODE_UPDATE) {
    thr = row_upd_step(thr);
  } else if (type == QUE_NODE_FETCH) {
    thr = fetch_step(thr);
  } else if (type == QUE_NODE_OPEN) {
    thr = open_step(thr);
  } else if (type == QUE_NODE_FUNC) {
    proc_eval_step(thr);
  } else if (type == QUE_NODE_THR) {
    thr = que_thr_node_step(thr);
  } else if (type == QUE_NODE_COMMIT) {
    thr = trx_commit_step(thr);
  } else if (type == QUE_NODE_UNDO) {
    thr = row_undo_step(thr);
  } else if (type == QUE_NODE_PURGE) {
    thr = row_purge_step(thr);
  } else if (type == QUE_NODE_RETURN) {
    thr = return_step(thr);
  } else if (type == QUE_NODE_EXIT) {
    thr = exit_step(thr);
  } else if (type == QUE_NODE_ROLLBACK) {
    thr = trx_rollback_step(thr);
  } else {
    ut_error;
  }

  // 更新 prev_node：EXIT 特殊处理为包含它的循环节点
  if (type == QUE_NODE_EXIT) {
    old_thr->prev_node = que_node_get_containing_loop_node(node);
  } else {
    old_thr->prev_node = node;
  }

  if (thr) {
    ut_a(thr_get_trx(thr)->error_state == DB_SUCCESS);
  }

  return (thr);
}
```

**关键**：`prev_node` 的更新很重要。控制语句（IF/WHILE/PROC）通过比较 `prev_node == parent` 来判断"控制是从父节点下来的（第一次进入）"还是"从子语句返回的（继续执行兄弟）"。

#### step 函数的固定签名

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

step 函数的返回值语义：
- 返回**原 thr**：继续执行
- 返回 **nullptr**：thr 挂起（锁等待）或完成，`que_run_threads_low` 退出循环

#### 调用链示例

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

### thr 状态机

#### 四种状态

```cpp
enum que_thr_state_t {
  QUE_THR_RUNNING,      // 正在执行
  QUE_THR_COMPLETED,    // 执行完成
  QUE_THR_LOCK_WAIT,    // 锁等待中
  QUE_THR_SUSPENDED,    // 挂起（fork 等待命令）
};
```

#### 状态转换

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

### 锁等待挂起机制

#### 状态设置的完整链路

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

#### que_thr_stop 完整分支

`que_thr_stop()`（`que0que.cc:650`）按优先级判断 thr 应该进入哪个状态，返回 `true` 表示已停止：

```cpp
bool que_thr_stop(que_thr_t *thr) {
  que_t *graph = thr->graph;
  trx_t *trx = thr_get_trx(thr);

  ut_ad(trx_mutex_own(trx));

  if (graph->state == QUE_FORK_COMMAND_WAIT) {
    /* [1] fork 在等待命令 → thr 挂起 */
    thr->state = QUE_THR_SUSPENDED;

  } else if (trx->lock.que_state == TRX_QUE_LOCK_WAIT) {
    /* [2] 事务进入锁等待 → thr 锁等待（最常用的分支） */
    trx->lock.wait_thr = thr;
    thr->state = QUE_THR_LOCK_WAIT;

  } else if (trx->error_state != DB_SUCCESS &&
             trx->error_state != DB_LOCK_WAIT) {
    /* [3] 有其他错误（非锁等待）→ thr 完成 */
    thr->state = QUE_THR_COMPLETED;

  } else if (graph->fork_type == QUE_FORK_ROLLBACK) {
    /* [4] rollback 类型的 fork → thr 挂起 */
    thr->state = QUE_THR_SUSPENDED;

  } else {
    /* [5] fork 仍活跃 → 不停止，返回 false */
    ut_ad(graph->state == QUE_FORK_ACTIVE);
    return false;      // ← 这个返回值很关键
  }

  return true;
}
```

五个分支的完整语义：

| 分支 | 条件 | 新状态 | 说明 |
|------|------|--------|------|
| [1] | `graph->state == QUE_FORK_COMMAND_WAIT` | `SUSPENDED` | fork 等待命令（如游标 fetch 间隙），thr 挂起等待下次唤醒 |
| [2] | `trx->lock.que_state == TRX_QUE_LOCK_WAIT` | `LOCK_WAIT` | **最常用的锁等待分支**，同时设置 `trx->lock.wait_thr = thr` 供后续唤醒定位 |
| [3] | 有其他错误且非 `DB_LOCK_WAIT` | `COMPLETED` | 真错误（如死锁 victim、超时），thr 结束 |
| [4] | `graph->fork_type == QUE_FORK_ROLLBACK` | `SUSPENDED` | rollback 图，thr 挂起等回滚调度（回滚图两层结构详见 [`trx.md`](trx.md)「回滚与查询图」） |
| [5] | fork 仍 `ACTIVE` 且无错误 | — | **返回 false，不停止** |

**分支 [5] 的返回值语义**：`que_thr_stop` 返回 `false` 表示"没有理由停止这个 thr，它应该继续运行"。`que_thr_dec_refer_count` 正是靠这个返回值判断"挂起/等待的原因已消失，thr 应继续执行"（见前文）。

**分支 [2] 是锁等待的核心**：注意这里设置的是 `trx->lock.wait_thr = thr`（事务记录等待的 thr），而 `trx->lock.que_state = TRX_QUE_LOCK_WAIT` 是在 `lock_table()`/`lock_rec_lock()` 检测到冲突时设置的。两者缺一不可——`que_thr_stop` 只是读取 `que_state` 来决定 thr 状态。

#### que_thr_stop vs que_thr_stop_for_mysql

| | `que_thr_stop()` | `que_thr_stop_for_mysql()` |
|---|---|---|
| 位置 | `que0que.cc:650` | `que0que.cc:770` |
| 调用者 | `que_thr_dec_refer_count()`（内部调度路径） | MySQL 接口路径（如 `row_lock_table()`） |
| 是否设置 LOCK_WAIT | **是**（分支 [2]） | **否** |
| 返回值 | `bool`（false = 不应停止） | `void` |
| 场景 | 内部 query graph 执行循环 | MySQL handler 层调用 InnoDB 后的收尾 |

`que_thr_stop_for_mysql()` 不设置 `LOCK_WAIT`。它处理 MySQL 接口路径的收尾——如果 thr 还在 RUNNING 且有错误就标记 COMPLETED，如果已经因为锁等待被 `que_thr_stop()` 设置了 `LOCK_WAIT` 就直接 return（保持原状态不变）。

**关键差异**：`que_thr_stop` 是**内部调度器**用的（在 `que_run_threads_low` 循环内被调用，通过返回值影响控制流）；`que_thr_stop_for_mysql` 是 **MySQL handler 层**用的（在 row0mysql.cc 的各 `row_xxx_for_mysql` 函数末尾调用，只做清理，不影响控制流）。降低引用计数的只有这两个函数。

#### lock_wait_suspend_thread 的实际挂起

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

#### 挂起的是哪个线程

**是 user thd 的 OS 线程在等。** 不是后台线程，也不是独立的 query graph 线程。

InnoDB 的 query graph 执行是 **1:1 模型**——一个 user connection 对应一个 OS 线程，这个线程同时就是执行 query graph 的线程。`que_thr_t` 只是逻辑状态机，底层跑在 user thd 的 OS 线程上。锁等待时 user thd 阻塞在 `os_event_wait` 上，不会去执行其他 query node。

#### 竞态检查的必要性

从 `que_thr_stop()` 设置 `QUE_THR_LOCK_WAIT` 到 `lock_wait_suspend_thread()` 被调用之间，持有锁的事务可能已经释放锁并调用了 `lock_wait_release_thread_if_suspended()`，把 thr 状态改回了 `QUE_THR_RUNNING`。

如果到 suspend 时发现状态已经变回 `RUNNING`，说明锁已释放，不需要睡了，直接返回继续执行。这避免了"锁已释放但线程还在等"的假唤醒问题。

---

### 精确唤醒：从 2PL 放锁到 os_event_set

#### 完整调用链

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

#### 授权等待者

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

#### 精确唤醒

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

#### 防二次唤醒设计

`lock0wait.cc:358-423` 注释明确了四条规则保证每个 trx 最多被唤醒一次：

1. 唯一唤醒途径是 `os_event_set`
2. 唯一 `os_event_set` 调用在 `lock_wait_release_thread_if_suspended`
3. 调用前必须先 `lock_reset_lock_and_trx_wait(lock)`
4. `lock_reset_lock_and_trx_wait` 断言 `wait_lock == lock` 且置 `wait_lock = NULL`

再次唤醒需要 `wait_lock` 非 NULL，故不可能二次唤醒。

---

#### 错误分四类恢复：`row_mysql_handle_errors` 的完整 switch

上面第 3 步 `row_mysql_handle_errors()` 看到 `DB_LOCK_WAIT`——把它完整展开，整个函数是一个 `switch (err)`，开头带 `handle_new_error:` 标签（等锁期间产生新错误会跳回来重处理）：

```cpp
bool row_mysql_handle_errors(dberr_t *new_err, trx_t *trx, que_thr_t *thr,
                             trx_savept_t *savept) {
  dberr_t err;
handle_new_error:
  err = trx->error_state;
  ut_a(err != DB_SUCCESS);
  trx->error_state = DB_SUCCESS;   // 先清零，避免污染下一轮判定
  switch (err) {
```

**第一类：锁等待超时**（是否回滚整事务可配）

```cpp
    case DB_LOCK_WAIT_TIMEOUT:
      if (row_rollback_on_timeout) {
        trx_rollback_to_savepoint(trx, nullptr);   // 回滚整个事务
        break;
      }
      [[fallthrough]];   // 否则落到第二类，只回滚最近一条语句
```

**第二类：只回滚最近一条 SQL**（16 种业务错误共用）

```cpp
    case DB_DUPLICATE_KEY:
    case DB_FOREIGN_DUPLICATE_KEY:
    case DB_TOO_BIG_RECORD:
    case DB_UNDO_RECORD_TOO_BIG:
    case DB_ROW_IS_REFERENCED:
    case DB_NO_REFERENCED_ROW:
    case DB_CANNOT_ADD_CONSTRAINT:
    case DB_TOO_MANY_CONCURRENT_TRXS:
    case DB_OUT_OF_FILE_SPACE:
    case DB_READ_ONLY:
    case DB_FTS_INVALID_DOCID:
    case DB_INTERRUPTED:
    case DB_CANT_CREATE_GEOMETRY_OBJECT:
    case DB_COMPUTE_VALUE_FAILED:
    case DB_LOCK_NOWAIT:
    case DB_BTREE_LEVEL_LIMIT_EXCEEDED:
      if (savept) {
        /* Roll back the latest, possibly incomplete insertion or update */
        trx_rollback_to_savepoint(trx, savept);
      }
      /* MySQL will roll back the latest SQL statement */
      break;
```

**第三类：挂起等锁，等到了就重试**（★ 唯一返回 `true` 的分支）

```cpp
    case DB_LOCK_WAIT:
      trx_kill_blocking(trx);                    // 必要时 kill 阻塞者
      DEBUG_SYNC_C("before_lock_wait_suspend");
      lock_wait_suspend_thread(thr);             // 真正挂起
      if (trx->error_state != DB_SUCCESS) {
        que_thr_stop_for_mysql(thr);
        goto handle_new_error;                   // ★ 等锁期间又出错 ⇒ 回开头重分类
      }
      *new_err = err;
      return (true);                             // ★ true = 等到了锁，请重试
```

**第四类：回滚整事务 / 直接 fatal**

```cpp
    case DB_DEADLOCK:
    case DB_LOCK_TABLE_FULL:
      /* Roll back the whole transaction; this resolution was added
      to version 3.23.43 */
      trx_rollback_to_savepoint(trx, nullptr);   // 死锁、锁表满 ⇒ 整事务回滚
      break;

    case DB_MUST_GET_MORE_FILE_SPACE:
      ib::fatal(...);                            // 空间彻底不够 ⇒ 无法继续
      break;
    ...
    default:
      ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_975)
          << "Unknown error code " << err << ": " << ut_strerr(err);
  }

  if (trx->error_state != DB_SUCCESS) {
    *new_err = trx->error_state;
  } else {
    *new_err = err;
  }
  trx->error_state = DB_SUCCESS;
  return (false);   // 其余全部 false：错误已处理，不需要重试
}
```

| 类别 | 错误 | 恢复动作 | 返回值 |
|---|---|---|---|
| 超时 | `DB_LOCK_WAIT_TIMEOUT` | 视 `row_rollback_on_timeout` 回滚整事务或落二类 | false |
| 语句级 | `DB_DUPLICATE_KEY` 等 16 种 | 回滚到 savepoint（最近一条语句） | false |
| **锁等待** | `DB_LOCK_WAIT` | 挂起等锁，等到了**重试** | **true** |
| 事务级 / 致命 | `DB_DEADLOCK`、`DB_LOCK_TABLE_FULL`、空间耗尽、未知码 | 回滚整事务 / `ib::fatal` | false |

⇒ ★ 只有 `DB_LOCK_WAIT` 返回 `true`，调用方（如 `row_search_mvcc`）据此判断"要不要重跑一次"——**这就是锁等待能自动重试的机制**。而 `goto handle_new_error` 让"等锁期间又超时 / 又死锁"能被重新分类处理，不必加一层调用。

### srv_sys->tasks 全局队列与 purge 调度

#### 队列定义

`srv_sys_t`（`srv0srv.cc:758`）中定义了一个全局 task queue：

```cpp
struct srv_sys_t {
  ib_mutex_t tasks_mutex;                          // 保护队列的 mutex
  UT_LIST_BASE_NODE_T(que_thr_t, queue) tasks;     // 全局 query thread 就绪队列
  srv_slot_t *sys_threads;                          // server 线程表
  // ...
};
```

#### 入队：srv_que_task_enqueue_low

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

#### 出队：srv_task_execute

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

#### purge worker 线程循环

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

#### purge coordinator 完整流程

#### 1. 构建 purge 查询图（启动时）

`trx_purge_graph_build()`（`trx0purge.cc:197`）在 InnoDB 启动时构建 purge 查询图，**每个 purge 线程一个 thr**：

```cpp
static que_t *trx_purge_graph_build(trx_t *trx, ulint n_purge_threads) {
  heap = mem_heap_create(512, UT_LOCATION_HERE);

  // 创建 purge 类型的 fork 根节点
  fork = que_fork_create(nullptr, nullptr, QUE_FORK_PURGE, heap);
  fork->trx = trx;

  for (i = 0; i < n_purge_threads; ++i) {
    thr = que_thr_create(fork, heap, nullptr);
    thr->child = row_purge_node_create(thr, heap);   // 每个 thr 挂一个 purge node
  }

  return (fork);
}
```

构建结果：
```
que_fork_t (QUE_FORK_PURGE)
├── thr[0] → purge_node_t (清理 undo rec)
├── thr[1] → purge_node_t
├── thr[2] → purge_node_t
└── ...（共 n_purge_threads 个）
```

**这是查询图中唯一真正的"多 thr 并行"场景**——一个 fork 下有多个 thr，每个 thr 有自己的 purge node。

#### 2. 初始化 purge sys

`trx_purge_sys_initialize()`（`trx0purge.cc:255`）调用 `trx_purge_graph_build` 创建 `purge_sys->query`。注意注释说明 purge 事务是"假事务"：

```cpp
/* A purge transaction is not a real transaction, we use a transaction
here only because the query threads code requires it. It is otherwise
quite unnecessary. We should get rid of it eventually. */
purge_sys->trx->id = 0;
purge_sys->query = trx_purge_graph_build(purge_sys->trx, n_purge_threads);
```

purge 本身不需要事务，但**查询图框架要求每个 graph 必须有 trx**，所以创建了一个 `id = 0` 的假事务。

#### 3. 一个 purge batch：trx_purge

`trx_purge()`（`trx0purge.cc:2471`）执行一次 purge batch：

```cpp
ulint trx_purge(ulint n_purge_threads, ulint batch_size, bool truncate) {
  /* [1] 克隆最老 read view 作为 purge 边界 */
  trx_sys->mvcc->clone_oldest_view(&purge_sys->view);

  /* [2] 读取待 purge 的 undo rec，分配到各 purge node */
  n_pages_handled = trx_purge_attach_undo_recs(n_purge_threads, batch_size);

  if (n_purge_threads > 1) {
    /* [3a] 多 worker：把前 n-1 个 thr 入全局队列（异步执行） */
    for (i = 0; i < n_purge_threads - 1; ++i) {
      thr = que_fork_scheduler_round_robin(purge_sys->query, thr);
      srv_que_task_enqueue_low(thr);       // 入 task queue，srv worker 执行
    }

    /* [3b] coordinator 自己执行最后一个 thr（同步） */
    thr = que_fork_scheduler_round_robin(purge_sys->query, thr);
    purge_sys->n_submitted += n_purge_threads - 1;

    goto run_synchronously;
  } else {
    /* [3c] 单线程：coordinator 直接执行 */
    thr = que_fork_scheduler_round_robin(purge_sys->query, nullptr);
  }

run_synchronously:
  ++purge_sys->n_submitted;
  que_run_threads(thr);                    // coordinator 执行自己的 thr
  purge_sys->n_completed.fetch_add(1);

  if (n_purge_threads > 1) {
    trx_purge_wait_for_workers_to_complete();   // 等待所有 worker 完成
  }
}
```

**核心设计**：`n_purge_threads` 个 thr 中，前 `n-1` 个入全局队列由 srv worker 异步执行，**coordinator 自己同步执行最后 1 个**（避免 coordinator 空转等待）。

#### 4. undo rec 分配：trx_purge_attach_undo_recs

`trx_purge_attach_undo_recs()`（`trx0purge.cc:2304`）把 undo rec 分配到各 purge node：

```cpp
static ulint trx_purge_attach_undo_recs(const ulint n_purge_threads,
                                        ulint batch_size) {
  /* [1] 取前 n_purge_threads 个 thr，重置其 purge node */
  for (auto thr : purge_sys->query->thrs) {
    if (n_purge_threads <= i) break;
    node = static_cast<purge_node_t *>(thr->child);
    ut_a(que_node_get_type(node) == QUE_NODE_PURGE);
    node->recs = nullptr;
    node->done = false;
    run_thrs[i++] = thr;
  }

  /* [2] 读取 batch_size 条 undo rec，按 table_id 分组 */
  Purge_groups_t purge_groups(n_purge_threads, purge_sys->heap);
  purge_groups.init();

  while (n_pages_handled < batch_size) {
    rec.undo_rec = trx_purge_fetch_next_rec(&rec.modifier_trx_id,
                                            &rec.roll_ptr, &n_pages_handled,
                                            heap);
    if (rec.undo_rec == nullptr) break;
    purge_groups.add(rec);        // 按 table_id 分组
  }

  /* [3] 负载不均时重新分配（仅 history 过长时触发） */
  purge_groups.distribute_if_needed();

  /* [4] 把分组结果赋给各 thr 的 purge node */
  purge_groups.assign(run_thrs);
}
```

`Purge_groups_t::distribute()`（`trx0purge.cc:2204`）做负载均衡：计算每组最多 `max_n = (total_rec + n_threads - 1) / n_threads` 条，超出部分移到下一组，最多两趟处理完。

**注意**：`distribute_if_needed()` 只在 `srv_max_purge_lag > 0 && rseg_history_len > srv_max_purge_lag` 时触发——即 history list 过长时才做负载均衡，正常情况保持按 table_id 分组的自然分布。

#### 5. round-robin 调度器

`que_fork_scheduler_round_robin()`（`que0que.cc:315`）按顺序选下一个 thr：

```cpp
que_thr_t *que_fork_scheduler_round_robin(que_fork_t *fork, que_thr_t *thr) {
  trx_mutex_enter(fork->trx);

  /* 若无当前 thr，取第一个；否则取下一个兄弟 thr */
  if (thr == nullptr) {
    thr = UT_LIST_GET_FIRST(fork->thrs);
  } else {
    thr = UT_LIST_GET_NEXT(thrs, thr);
  }

  if (thr) {
    fork->state = QUE_FORK_ACTIVE;
    fork->last_sel_node = nullptr;

    switch (thr->state) {
      case QUE_THR_COMMAND_WAIT:
      case QUE_THR_COMPLETED:
        ut_a(!thr->is_active);
        que_thr_init_command(thr);       // 初始化 thr 准备执行
        break;
      case QUE_THR_SUSPENDED:
      case QUE_THR_LOCK_WAIT:
      default:
        ut_error;                        // 不支持这些状态
    }
  }

  trx_mutex_exit(fork->trx);
  return (thr);
}
```

调度器只接受 `COMMAND_WAIT` / `COMPLETED` 状态的 thr，`SUSPENDED` / `LOCK_WAIT` 会 `ut_error`。这保证了每个 thr 在被调度前处于可初始化状态。

---

### 两种执行模型对比：1:1 阻塞 vs task queue 分发

#### 模型 1：User DML/DDL — 1:1 阻塞模型

```
User Connection (OS Thread)
  → ha_innobase::update_row()
  → que_run_threads(thr)              ← user thd 自己执行
    → thr 不进 srv_sys->tasks 队列
    → 锁等待 → os_event_wait          ← user thd 阻塞
    → 被唤醒后继续执行
```

普通 DML/DDL 的 thr 从创建到完成都在 user thd 上执行，不进 task queue，不切换到 srv worker。

#### 模型 2：Purge — task queue 调度模型

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

#### 对比

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

## 参考

**论文**

- Lehman, P. L., Yao, S. B. *Efficient Locking for Concurrent Operations on B-Trees*. ACM TODS, 1981. — B-link tree 与乐观锁耦合，锁等待挂起/恢复与其"遇到锁冲突则等待"思路一致
- Gray, J. N., Lorie, R. A., Putzolu, G. R., Traiger, I. L. *Granularity of Locks and Degrees of Consistency in Shared Data Banks*. IBM, 1975. — 2PL，精确唤醒发生在 shrinking phase 放锁时
- Graefe, G. *Volcano — An Extensible and Parallel Query Evaluation System*. IEEE TKDE, 1994. — 火山模型（iterator model），与 query graph 的逐行拉取相似

**官方文档**

- *MySQL 8.0 Reference Manual → InnoDB Locking and Transaction Model*

**相关文档**

- 回滚图（roll_node 外层 + undo graph 内层）与事务回滚剖析见 [`trx.md`](trx.md)「回滚机制」
- 事务状态机、提交与回滚的完整生命周期见 [`trx.md`](trx.md)
