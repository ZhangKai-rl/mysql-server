# MDL 元数据锁深度解析

> 基于 MySQL 8.0.39 源码，涵盖 MDL 锁类型分级、双兼容性矩阵（granted/waiting）、`can_grant_lock` 排队语义、防饥饿优先级切换、wait-for graph 死锁检测、5.6 排队问题与 online/instant DDL 演进。
>
> **边界**：本篇讲 server 层 MDL 元数据锁；InnoDB 行锁/间隙锁/死锁检测见 [`../innodb/lock.md`](../innodb/lock.md)；MDL 与 InnoDB 行锁的分工见本文 Misc。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MDL（Metadata Lock，元数据锁）是 server 层的锁机制，保护**表结构/元数据**（而非数据行），防止 DDL 与 DML/查询并发时破坏一致性。它由 `SQL 层` 统一管理（`sql/mdl.cc`），独立于存储引擎的行锁。

### 用途

解决"DDL 与 DML 并发"的经典问题：一个事务正在 `SELECT` 某表时，另一个会话 `DROP TABLE` 该表，如果 DDL 立刻执行，正在扫描的行可能被物理删除导致崩溃。MDL 让 DML/查询持有共享锁、DDL 持有排他锁，二者互斥。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.5 | 引入 MDL，取代旧的 `.frm` 文件 + table cache 锁 |
| 5.6 | 已有完整的锁类型分级与排队语义；但几乎全部 DDL 全程持 X 锁，排队阻塞严重 |
| 5.7 | 引入 online DDL（`ALGORITHM=INPLACE`），DDL 只在准备/提交阶段短暂持 X 锁 |
| 8.0 | 引入 instant DDL（instant ADD COLUMN 等），只改元数据、锁更轻 |

---

## 理论基础

### 设计思想与权衡

**核心：锁类型分级 + 意向锁**。MDL 把"对表的访问意图"分成从最弱（只读元数据）到最强（改结构）的十余级（`enum_mdl_type`），每级对应一类 SQL：

| 锁类型 | 对应语句 | 强度 |
|---|---|---|
| `MDL_SHARED`（S） | 只关心元数据（存储例程、预处理） | 最弱 |
| `MDL_SHARED_HIGH_PRIO`（SH） | INFORMATION_SCHEMA 查询 | 弱（高优先级） |
| `MDL_SHARED_READ`（SR） | SELECT | 弱 |
| `MDL_SHARED_WRITE`（SW） | INSERT/UPDATE/DELETE | 中 |
| `MDL_SHARED_UPGRADABLE`（SU） | online DDL 第一阶段（可升级） | 中 |
| `MDL_SHARED_READ_ONLY`（SRO） | LOCK TABLES READ | 强 |
| `MDL_SHARED_NO_WRITE`（SNW） | 可升级到 X（online DDL 提交前） | 强 |
| `MDL_SHARED_NO_READ_WRITE`（SNRW） | LOCK TABLES WRITE | 强 |
| `MDL_EXCLUSIVE`（X） | CREATE/DROP/RENAME TABLE | 最强 |

`MDL_INTENTION_EXCLUSIVE`（IX）是**意向排他锁**，用于 scoped lock（如锁整个 schema），源自多粒度锁协议——先对 schema 加 IX，再对表加具体锁。

**核心权衡：双兼容性矩阵（granted vs waiting）**。这是理解 MDL 排队语义的关键，也是本机制最有设计含量的一点。MDL 维护**两个不同的兼容性矩阵**（`mdl.cc:469-489`）：

```469:489:sql/mdl.cc
    /** 已授予锁的兼容性矩阵 */
    bitmap_t m_granted_incompatible[MDL_TYPE_END];
    /** 等待锁的优先级矩阵（4 个，防饥饿） */
    bitmap_t m_waiting_incompatible[4][MDL_TYPE_END];
```

- `m_granted_incompatible`：正常的锁兼容性——两个已授予的锁是否互斥（SR 与 SR 兼容，SR 与 X 互斥）。
- `m_waiting_incompatible`：**等待中的锁**对"新请求能否授予"的优先级影响——一个 X 锁在等待时，即使它还没拿到锁，也会"影子阻塞"后续的 SR 请求。

**为什么需要两套矩阵**：如果只按 granted 兼容性判断，一个等待中的 X 锁（DDL）永远等不到——因为后续的 SR（SELECT）源源不断插入，X 锁不断被"插队"，造成**写锁饥饿**。`m_waiting_incompatible` 让等待中的 X 锁能挡住后续 SR，保证 X 锁最终能拿到。代价就是著名的"**MDL 排队问题**"：DDL 一排队，后续所有同表查询都被影子阻塞——这是"防写锁饥饿"与"读并发"之间的权衡，X 锁的公平性优先。

### 理论溯源

- **意向锁**（IX）：来自 Gray 的多粒度锁协议（multiple granularity locking），先用粗粒度 IX 锁表空间/schema，再细粒度锁具体表。
- **死锁检测**：wait-for graph 的深度优先搜索（DFS），见「核心实现·死锁检测」。

### 算法与数据结构

锁时长（`enum_mdl_duration`，`mdl.h:333`）：

```333:348:sql/mdl.h
enum enum_mdl_duration {
  MDL_STATEMENT = 0,   // 语句结束释放
  MDL_TRANSACTION,     // 事务结束释放（DML 的 SW/SR 用这个）
  MDL_EXPLICIT,        // 显式释放（LOCK TABLES 用）
};
```

核心对象（`mdl.h`）：`MDL_key`（锁对象标识：namespace+db+name）、`MDL_request`（一次锁请求）、`MDL_ticket`（请求的票据，代表一次持有）、`MDL_lock`（某对象的锁，挂 granted/waiting 两个 ticket 队列）、`MDL_context`（一个会话/事务持有的锁集合）。

### 他库对比与演进动机

PostgreSQL 的 DDL 也走 `pg_locks` 的 AccessExclusiveLock，但 PG 的 MVCC 允许 `ALTER TABLE` 大部分操作与查询并发（DDL 与 DML 的冲突模型更宽松，靠 catalog 版本化）。MySQL 早期（5.6）DDL 全程持 X 锁阻塞严重，5.7 的 online DDL 是向 PG 的"DDL 少持锁"靠拢——把 DDL 拆成"短暂持 X 锁 + 大部分时间持 SU/SRO 可升级锁"，让 DML 在 DDL 进行中继续执行。

---

## 核心实现

### 主链路

```
SQL 执行 → open_tables() → 为每张表构造 MDL_request
  → MDL_context::acquire_lock(request)
    → try_acquire_lock_impl     尝试快路径（unobtrusive 锁 + 无冲突）
    → 冲突 → 进入 m_waiting 队列 → find_deadlock() 检查死锁
      → timed_wait 等待
  → 唤醒后 → reschedule_waiters() 把可授予的等待锁重新分配
```

### can_grant_lock：双矩阵判断（关键环节）

`can_grant_lock`（`mdl.cc:2405`）判断一个新请求能否立即授予：

```2405:2426:sql/mdl.cc
bool MDL_lock::can_grant_lock(enum_mdl_type type_arg,
                              const MDL_context *requestor_ctx) const {
  // ① 等待中的锁对当前请求的兼容性（防饥饿的优先级）
  bitmap_t waiting_incompat_map = incompatible_waiting_types_bitmap()[type_arg];
  // ② 已授予锁对当前请求的兼容性
  bitmap_t granted_incompat_map = incompatible_granted_types_bitmap()[type_arg];

  if (!(m_waiting.bitmap() & waiting_incompat_map)) {          // 无等待锁影子阻塞
    if (!(fast_path_granted_bitmap() & granted_incompat_map)) { // 无 fast-path 冲突
      if (!(m_granted.bitmap() & granted_incompat_map))        // 无 slow-path 冲突
        can_grant = true;
```

逐段解释：

- **① 等待锁检查**：`m_waiting.bitmap() & waiting_incompat_map` 非零，说明等待队列里有与当前请求"在 waiting 语义下不兼容"的锁（典型是等待中的 X 锁），于是**拒绝授予**，当前请求也去排队。这就是"DDL 的 X 锁一排队，后续 SR 也排队"的机制根源。
- **②③ 已授予锁检查**：`fast_path_granted_bitmap()` 和 `m_granted.bitmap()` 分别检查快路径和慢路径已授予的锁是否冲突。这里用 `granted_incompat_map`（正常兼容性）。

关键对比：`granted_incompat_map` 决定"两个已授予锁能否共存"，`waiting_incompat_map` 决定"等待锁能否挡住后来的锁"。前者是锁语义，后者是**公平性/防饥饿**语义。

### 排队语义与防饥饿

`m_waiting_incompatible` 有 **4 个矩阵**（`mdl.cc:592-598`），通过 `max_write_lock_count` 阈值动态切换，防止低优先级锁（"piglet"=SW，"hog"=X 等）饿死：

```592:603:sql/mdl.cc
    uint idx = 0;
    if (m_piglet_lock_count >= max_write_lock_count) idx += 1;
    if (m_hog_lock_count >= max_write_lock_count) idx += 2;
    return idx;
```

`m_piglet_lock_count`/`m_hog_lock_count` 记录"连续授予低优先级锁的次数"。当一个等待中的 X 锁被连续授予的 SW/SR 锁"插队"超过阈值时，切换到更严格的优先级矩阵，让等待中的 X 锁能插队——避免 X 锁无限饥饿。这是"防饥饿"的又一重保障，与双矩阵的 waiting 语义配合。

### 死锁检测

MDL 的死锁检测是 wait-for graph 的 DFS（`mdl.cc:291` 的 `Deadlock_detection_visitor` + `mdl.cc:4077` 的 `find_deadlock` + `mdl.cc:3892` 的 `visit_subgraph`）：

```357:388:sql/mdl.cc
bool Deadlock_detection_visitor::enter_node(MDL_context *node) {
  m_found_deadlock = ++m_current_search_depth >= MAX_SEARCH_DEPTH;  // 深度超 32 视为死锁
  ...
}
bool Deadlock_detection_visitor::inspect_edge(MDL_context *node) {
  m_found_deadlock = node == m_start_node;   // 回到起点 = 找到环
  return m_found_deadlock;
}
```

- `visit_subgraph` 从等待锁的 context 出发，DFS 遍历"谁阻塞了谁"（wait-for graph），`inspect_edge` 检测是否回到起点（成环）。
- `MAX_SEARCH_DEPTH = 32` 限制递归深度，超过就假定死锁（防无限递归）。
- 找到死锁后按 `get_deadlock_weight()` 选代价最低的 context 作为 victim 回滚。

---

## 相关的系统变量/状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `lock_wait_timeout` | 31536000 | Session/Global | MDL/行锁等待超时（秒） |

> `max_write_lock_count` 是编译期常量（非系统变量），控制防饥饿的优先级切换阈值。

---

## Misc

### 易混淆概念

- **MDL vs InnoDB 行锁**：MDL 是 **SQL 层**的元数据锁，保护**表结构**（DDL 与 DML 互斥）；InnoDB 行锁是**引擎层**的，保护**数据行**（事务间并发）。二者独立：一个 DML 同时持 MDL 的 SW 锁（元数据层面）+ InnoDB 行锁（数据层面）。见 [`../innodb/lock.md`](../innodb/lock.md)。
- **MDL 的 X 锁 vs InnoDB 的 X 锁**：同名不同物。MDL X 是元数据排他锁（DDL），InnoDB X 是行级排他锁（行锁）。
- **"写锁"不是 MDL 术语**：MDL 里 DML 用 `MDL_SHARED_WRITE`（SW，共享写锁），DDL 用 `MDL_EXCLUSIVE`（X，排他锁）。口语里的"DDL 写锁"指的是 X 锁。
- **排队语义 vs 锁不兼容**：DDL 的 X 锁阻塞后续 SELECT，**不是因为 SR 和 X 不兼容**（它们确实不兼容），而是因为 X 在**等待队列**里通过 `waiting_incompat_map` 影子阻塞了后来的 SR——即使这些 SR 和已授予的 SR 是兼容的。这是"防饥饿"导致的额外阻塞，不是锁语义本身。

---

## 参考

**官方文档**
- MySQL 8.0 Reference Manual → Metadata Locking
- MySQL 8.0 Reference Manual → Online DDL Operations

**相关文档**
- InnoDB 行锁/间隙锁/死锁检测见 [`../innodb/lock.md`](../innodb/lock.md)
- server 层锁顺序工具见 [`vio.md`](infra/vio.md) 相邻的 lock order 主题（待补链接）
