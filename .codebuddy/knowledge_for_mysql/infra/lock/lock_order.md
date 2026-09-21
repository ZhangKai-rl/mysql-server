# LOCK ORDER 工具（server 层跨层锁序图校验）深度解析

> 基于 MySQL 8.0.39 源码：`sql/debug_lock_order.cc`（~2800 行）+ `sql/debug_lock_order.h` + `sql/debug_lo_parser.yy` / `debug_lo_scanner.ll`（依赖文件解析）。`WITH_LOCK_ORDER=ON` 构建选项启用。
>
> **边界**：本篇讲 **LOCK ORDER 工具**（server 层跨层、以 PFS 名为坐标的有向图判环）。InnoDB 专属的 LatchDebug（latch level 金字塔）见 [`primitives/innodb_sync.md`](primitives/innodb_sync.md)「LatchDebug 的锁序金字塔」节；全局锁类型学盘点见 [`README.md`](README.md)。

## 概述

### 是什么

LOCK ORDER 是 MySQL 官方的**运行时死锁（锁序）检测工具**：在 `WITH_LOCK_ORDER` 构建下，把 server + InnoDB 的**每一次加锁/解锁**经 PFS 钩子拦截，以 **PFS instrument 名为节点**建一张**全局有向图**（"持锁 A 时拿锁 B"记一条边 A→B），运行时用 **Tarjan 强连通分量（SCC）** 判环——**图里有环 = 存在死锁可能**。

### 解决什么问题

LatchDebug 只能校验 InnoDB 自己的锁（它不认识 mysys 的 `mysql_mutex_t`、server 的 MDL 等）。LOCK ORDER 用 **PFS instrument 名**做通用坐标，把跨层（server↔engine）的加锁关系画进**同一张图**——这是它存在的根本理由。死锁本质是"锁依赖图里有环"（图论），判环就是死锁检测。

### 与 LatchDebug 的本质区别

| | LatchDebug | LOCK ORDER 工具 |
|---|---|---|
| 思路 | **预防式**：声明每把锁的层级，运行时查"后拿的 ≤ 已持有" | **检测式**：记录真实加锁序，事后跑图判环 |
| 坐标 | `latch_level_t` 手工金字塔 | PFS instrument 名（自动、全局唯一） |
| 覆盖 | 仅 InnoDB 自研原语 | server + InnoDB 全部 PFS 锁 |
| 判据 | 违反层级序 → crash | SCC（size ≥ 2）→ 环 → 报错 |
| 开销 | UNIV_DEBUG 下每次加锁比较 | 每次加锁建图（仅 WITH_LOCK_ORDER 构建） |

## 核心实现

### 1. PSI 钩子：透明拦截所有锁操作

LOCK ORDER 不碰业务代码，而是**替换 PFS 的实现**。`LO_init`（debug_lock_order.cc）接收 `PSI_mutex_bootstrap**` / `PSI_rwlock_bootstrap**` / `PSI_cond_bootstrap**` 等参数，把 PFS 的 bootstrap 换成 LO 自己的类：

```cpp
int LO_init(LO_global_param *param, PSI_thread_bootstrap **thread_bootstrap,
            PSI_mutex_bootstrap **mutex_bootstrap,
            PSI_rwlock_bootstrap **rwlock_bootstrap, ...) {
  global_graph = new LO_graph();
  // 输出 lock_order-(timestamp)-(pid).log
  safe_snprintf(filename, ..., "%s/lock_order-%ju-%d.log",
                param->m_out_dir, (uintmax_t)now, getpid());
  ...
}
void LO_activate() { check_activated = true; }   // 启动后激活检查
```

LO 的锁类**继承 PFS 的接口**：`LO_mutex : public PSI_mutex`、`LO_rwlock`（经 `LO_rwlock_proxy` 包一层）、`LO_cond : public PSI_cond`。PFS 本来就是所有锁操作的必经之路（instrument 埋点），所以替换 bootstrap 后，**全库的加锁/解锁自动流入 LO 的钩子**，业务代码零改动。

### 2. 图模型：节点、弧、邻接矩阵

```cpp
class LO_node { /* 图节点：一把锁的一个状态（state node）或一个操作（operation node）*/ };
class LO_arc  { /* 有向边 from → to */ };

class LO_graph {
  LO_node_list m_nodes;
  LO_arc_list m_arcs;
  LO_arc *m_arc_matrix[LO_MAX_NODE_NUMBER][LO_MAX_NODE_NUMBER];  // 邻接矩阵
};
```

- **节点**分两类：**state node**（`mutex/sql/LOCK_open` 处于"已持有"态）和 **operation node**（`rwlock/sql/x` 的"read"/"write"操作，用于表达同一把锁在不同操作下的不同依赖）。
- **弧**：一条 `from → to`，含义"持 from 时再拿 to"。
- 三种 rwlock 类分别建模：`LO_rwlock_class_pr`（prlock）/ `LO_rwlock_class_rw`（rwlock）/ `LO_rwlock_class_sx`（InnoDB sxlock，**三态** S/SX/X 建模，注释 "Shared exclusive locks are recursive... SX + X counts as X"）。

### 3. 建弧：`check_common` 在持锁时加边

每次"已持 A 锁，再拿 B 锁"，`check_mutex`/`check_rwlock`/`check_cond`/`check_file` 汇聚到 `check_common`：

```cpp
void LO_graph::check_common(LO_thread *thread, ..., const LO_node *from_node,
                            ..., const LO_node *to_node, ...) {
  if (to_node->is_sink())     return;   // 汇节点（不再参与判环）
  if (to_node->is_ignored())  return;   // 依赖文件里标记 IGNORED 的节点
  // ... 建弧 from_node → to_node
}
```

关键点：**只记"跨锁"的边**（同一把锁的递归重入不建环），`IGNORED`/`sink` 节点在建图时就被过滤。

### 4. 判环：Tarjan SCC + girth

```cpp
void LO_graph::scc_util(const SCC_visitor *v, int *discovery_time, int *scc_count,
                        LO_node *n, std::stack<LO_node *> *st);   // Tarjan 主递归
int  LO_graph::compute_scc(const SCC_visitor *v);
void LO_graph::compute_scc_girth(...);   // 算环的 girth（最短环）/ circumference
```

**SCC（强连通分量）size ≥ 2 = 环 = 死锁可能**。`girth`（最短环长）帮定位最该先修的环。源码注释原话："a SCC of size greater than one is a cluster of nodes where any node is reachable... Hence, SCC should not exist, and the code must be re-factored to avoid them."

### 5. 依赖文件语法（debug_lo_parser.yy）

依赖文件（`lock_order_dependencies.txt` 及其二，由 `lo_param.m_dependencies_1/2` 指定）用 `ARC` / `BIND` / `NODE` 三种声明，手工描述"允许的依赖"与"节点属性"：

```
ARC   FROM "mutex/sql/A" STATE "X" TO "mutex/sql/B" FLAGS LOOP
BIND  "cond/sql/x" TO "mutex/sql/y" FLAGS UNFAIR
NODE  "mutex/sql/ignored_lock" FLAGS IGNORED
```

- **ARC**：声明一条授权弧（允许的依赖）。`STATE` 给 rwlock/sxlock 标状态（`R`/`W`、`S`/`SX`/`X`）；`OP`/`RECURSIVE` 表达"以某操作递归拿"；`FLAGS` 见下表。
- **BIND**：cond 变量绑定到它依赖的 mutex（`LO_FLAG_BIND`），`UNFAIR` 标非公平调度。
- **NODE**：给单节点标属性，如 `IGNORED`（从判环中剔除）。

flags 位（`LO_FLAG_*` 宏）：

| flag | 含义 |
|---|---|
| `LOOP` | 已知环——从 revised graph 剔除后分析剩余 |
| `IGNORED` | 节点从图里整体忽略（缩小大 SCC 的手段） |
| `TRACE`/`DEBUG` | 建弧时打 trace/调试信息 |
| `UNFAIR` | cond 非公平调度（BIND 专用） |
| `BIND` | cond↔mutex 绑定（BIND 语法自动置） |
| `MICRO` | 宏展开生成的微弧 |

**命名简化**：PFS 全名 → LO 短名（去掉 `wait/synch/` 或 `wait/io/` 前缀）：

| PFS instrument | LO 节点名 |
|---|---|
| `wait/synch/mutex/sql/LOCK_open` | `mutex/sql/LOCK_open` |
| `wait/synch/rwlock/sql/x` | `rwlock/sql/x` |
| `wait/synch/sxlock/innodb/dict_operation_lock` | `sxlock/innodb/dict_operation_lock` |
| `wait/io/file/sql/relaylog` | `file/sql/relaylog` |

### 6. 工作流：从"发现环"到"修掉环"

源码注释（§LO_PROCESS）给出官方工作流：

```
1. full graph 跑 SCC → 报告里列出每个环（SCC）及其内部弧
2. 人工分析每条弧：哪条是设计允许的（标记 LOOP），哪条是代码缺陷
3. 标记 IGNORED / LOOP → 重跑 → revised graph 的 SCC（缩小后的环）
4. 迭代直到 revised graph 无环 → 针对剩余 LOOP 弧提 bug → 改代码
5. 终极目标：graph 无 SCC、无 LOOP 弧、无 IGNORED 节点、测试套件通过
```

报告分六段（lock_order.txt）：`DEPENDENCY GRAPH` / `SCC ANALYSIS (full graph)` / `IGNORED NODES` / `LOOP ARC` / `SCC ANALYSIS (revised graph)` / `UNRESOLVED ARCS`（依赖文件里对不上真实节点的弧）。

## 可观测性

| 手段 | 入口 |
|---|---|
| 运行日志 | `lock_order-(timestamp)-(pid).log`（构建目录下） |
| 图报告 | `lock_order.txt`——`SET GLOBAL lock_order_print_txt=ON` + `mysqladmin debug`（COM_DEBUG）触发 dump |
| 一键跑 | `MTR_LOCK_ORDER=1 ./mtr lock_order.cycle`（辅助测试，侧产物即图 dump） |
| 环检测 | 报告里 `Found SCC number N of size M` + `Dumping arcs for SCC N` |

> 建议先加载动态插件/组件再 dump，让插件里的锁名也进图比对。

## Misc

### 面向二次开发 / 坑

- **只有 `WITH_LOCK_ORDER` 构建才生效**，是独立构建选项（非 UNIV_DEBUG，也非默认 release）。
- **加一把新锁 = 依赖文件加一条 ARC**：否则"未授权弧"会被当成环报出来（或进 UNRESOLVED ARCS）。
- **SCC ≠ 一定死锁**：是"可能死锁"（潜在环），需人工判断哪条弧是设计允许的（LOOP）还是真 bug——这是工具不能自动下结论的部分。
- **InnoDB sxlock 三态建模是近似**：SX 递归 + SX+X 算 X，是官方对 SX 语义的图形化抽象，边缘 case 以注释为准。

### 社区边界澄清

- LOCK ORDER 是 **MySQL 社区版自带**（8.0 起），不是 Oracle 商业版或第三方特性。
- 它检测的是**同步原语**（mutex/rwlock/cond/file）的锁序，**不覆盖事务锁**（InnoDB 行锁死锁由 `lock0lock` 的 wait-for graph 检测；MDL 由 `MDL_context` 死锁检测）。

## 参考

- 锁序分层归属（LatchDebug vs LOCK ORDER vs PFS）：[`README.md`](README.md) 2.1 节
- InnoDB 锁序金字塔机制详解：[`primitives/innodb_sync.md`](primitives/innodb_sync.md)
- 源码：`sql/debug_lock_order.cc`（含完整的 Doxygen 头注释，即 §LO_* 各节）、`sql/debug_lo_parser.yy`（依赖文件语法）
