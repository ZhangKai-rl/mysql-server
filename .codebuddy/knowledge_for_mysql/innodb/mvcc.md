# MVCC / 一致性读 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 MVCC 版本可见性、semi-consistent read（半一致性读）、undo 版本链、read view。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [semi-consistent read](#semi-consistent-read)
- [read view 与可见性判断](#read-view-与可见性判断)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 实现无锁一致性读的基础：通过 undo 日志保存行的历史版本，读事务依据 read view 判断哪些版本可见，从而不加锁读到一致性快照。semi-consistent read 是 MVCC 思想在 UPDATE/DELETE 锁定读上的一个变体优化。

### 用途

- MVCC 一致性读：让普通 SELECT 不加锁也能读到事务一致性快照，读写互不阻塞。
- semi-consistent read：在 READ COMMITTED / READ UNCOMMITTED 下，UPDATE/DELETE 扫描遇到被其他事务锁住的行时不等待锁，而是返回最新已提交版本让 SQL 层判断是否满足 WHERE；不满足直接跳过，满足才回头加锁重读。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | semi-consistent read 由系统变量 `innodb_locks_unsafe_for_binlog`（内部 `srv_locks_unsafe_for_binlog`）控制 |
| 8.0 | 移除 `innodb_locks_unsafe_for_binlog`，改为由隔离级别自动决定（`allow_semi_consistent() == skip_gap_locks()`） |

---

## 理论基础

### 设计模式

semi-consistent read 是**乐观并发控制（OCC）**思想在锁定读上的"先乐观、后悲观"两阶段应用：先不加锁读已提交版本，校验通过再退化悲观加锁重读。同时是一种**惰性/延迟加锁（lazy locking）**——把加锁推迟到确认"这行真的需要更新"之后。

### 相关论文

- Bernstein & Goodman 的多版本并发控制理论；MVCC 的可见性判断基于事务快照隔离（Snapshot Isolation）。
- semi-consistent read 无独立论文，属于工程优化，正确性证明见 `row0sel.cc:4334` 注释（依赖"不加 gap lock"前提）。

### 算法与数据结构

- 版本链：每条记录通过 `DB_ROLL_PTR` 指向 undo log 中的前一个版本，`trx_undo_prev_version_build` 沿链回溯。
- 可见性判断：read view 用 `m_ids`（活跃事务集合）+ `m_low_limit_no`（提交序下界）判断版本归属事务是否活跃。
- 复杂度：版本回溯为 O(链长)，但 purge 会周期性清理不再需要的旧版本，控制链长。

### 类似实现对比

PostgreSQL 的 MVCC 把旧版本存在堆表中（无 undo 链），Oracle 与 InnoDB 类似用 undo/回滚段。semi-consistent read 是 InnoDB 特有优化，等价于"锁定读失败后降级为一致性读"。

### 历史背景

`innodb_locks_unsafe_for_binlog` 早期用于在 RC 下提升并发（牺牲 binlog 与提交顺序的一致性保证），后来 MySQL 将 RC 下的 semi-consistent read 作为标准行为固化，去掉该开关，由隔离级别统一决定。

---

## semi-consistent read

### 是什么

InnoDB 在 READ COMMITTED / READ UNCOMMITTED 隔离级别下，UPDATE/DELETE 扫描时的一种"先乐观后悲观"优化：遇到被其他事务锁住的行不等待锁，返回该行最新已提交版本；SQL 层判断不满足 WHERE 则跳过，满足则重新加锁读取并更新。

### 触发条件（严格）

`row_search_mvcc` 中四个条件同时成立才走半一致性路径（`row0sel.cc:5264-5266`）：

```
prebuilt->row_read_type == ROW_READ_TRY_SEMI_CONSISTENT
&& !unique_search
&& index == clust_index
&& !trx_is_high_priority(trx)
```

即：row_read_type 处于 TRY 状态、非唯一搜索、必须是聚集索引、非高优先级事务。

各条件的精确定义：

- `unique_search`（`row0sel.cc:4634-4643`）= `match_mode == ROW_SEL_EXACT`（等值点查）且 `dict_index_is_unique`（唯一索引/主键）且给了完整唯一键（`search_tuple` 字段数 == 唯一索引列数）。**不是"非等于"**：等值查普通二级索引、或只给唯一索引前缀，也属于 `!unique_search`。`!unique_search` 排除的是"精确唯一点查"，因为这种点查最多命中一行、语义要求精确，不走"先读旧版本再重读"。
- 高优先级事务 `trx_is_high_priority`（`trx0trx.ic:256`）= `thd_tx_priority > 0`（`sql_thd_api.cc:408`）。`tx_priority`/`thd_tx_priority` 默认均 0（`sql_class.cc:1093`），`thd_tx_priority` 只能由 `set_thd_tx_priority`（`rpl_replica.cc:353`，断言 `SYSTEM_THREAD_SLAVE_SQL`/`SLAVE_WORKER`）设置。即**只有复制从库 SQL 线程/MTS worker 是高优先级**，与用户 SQL 无关。高优先级 applier 锁冲突时可压过普通事务（`lock0lock.cc` 的 HP 分支忽略等待锁、`trx_arbitrate` 回滚低优先级阻塞者）。

### 加锁时机（两条路径）

引擎层 `row_search_mvcc` 在 SQL 层 WHERE 过滤**之前**就尝试加锁：

- 路径 A（行未锁）：`sel_set_rec_lock` 直接成功，返回 `DB_SUCCESS_LOCKED_REC` 表示"新加了锁"，RC/RU 下记 `new_rec_lock[LOCK_PCUR] = true`（`row0sel.cc:5274-5281`）。回到 SQL 层：WHERE 满足 → 保留锁更新；不满足 → `unlock_row` 释放这个新锁（`releases_non_matching_rows()` 仅 RC/RU 为 true，RR 不释放）。
- 路径 B（行已被锁）：走 `SELECT_SKIP_LOCKED`，`sel_set_rec_lock` 不等待返回 `DB_SKIP_LOCKED`，不加锁、构建旧版本、置 DID。加锁推迟到 WHERE 判定之后：不满足 → 跳过；满足 → 重读该行，此时 `row_read_type == DID`（非 TRY），`use_semi_consistent = false`，改用 `prebuilt->select_mode`（普通模式）悲观等待加锁（`row0sel.cc:4897-4906`，注释 "repeat ... as a pessimistic locking read"）。

结论：未锁行读行即加锁（WHERE 前）；已锁行才把加锁推迟到 WHERE 满足后。semi-consistent 省的是"已锁无关行"的等待与加锁。

### 核心流程

1. SQL 层扫描前 `try_semi_consistent_read(true)`，置 `row_read_type = ROW_READ_TRY_SEMI_CONSISTENT`（`ha_innodb.cc:10043`）。
2. 引擎层 `row_search_mvcc` 用 `SELECT_SKIP_LOCKED` 请求锁（`row0sel.cc:5267`）：行被锁时 `sel_set_rec_lock` 不创建 WAITING 锁、不排队，直接返回 `DB_SKIP_LOCKED`。
3. 收到 `DB_SKIP_LOCKED` 后调 `row_sel_build_committed_vers_for_mysql`（`row0sel.cc:721`）→ `row_vers_build_for_semi_consistent_read`（`row0vers.cc:1368`）沿 undo 链回溯，找第一个属于已提交事务的版本；若回溯到 `prev_version == nullptr`（行是别的事务刚 insert 的），返回 `old_vers = nullptr`，调用方 `goto next_rec` 跳过。
4. 返回 SQL 层前置 `row_read_type = ROW_READ_DID_SEMI_CONSISTENT`（`row0sel.cc:6075-6080`）。
5. SQL 层（`sql_update.cc:759-780`）：不满足 WHERE → `unlock_row()` 跳过（DID 清回 TRY，不真正解锁，因为从未加锁）；满足 WHERE 但 `was_semi_consistent_read()` 为 true → `continue` 重新读同一行并正常加锁。

### 状态机

| row_read_type | 值 | 含义 |
|---------------|----|------|
| ROW_READ_WITH_LOCKS | 0 | 正常加锁读 |
| ROW_READ_TRY_SEMI_CONSISTENT | 1 | 尝试半一致性读 |
| ROW_READ_DID_SEMI_CONSISTENT | 2 | 刚做了半一致性读（未持锁） |

### 正确性依赖

源码 `row0sel.cc:4334-4341` 注释：正确性证明依赖"不加 gap lock"，因为 gap lock 锁定范围，若范围末尾行被删除、purge，半一致性读行为复杂难证。因此严格限定在 `skip_gap_locks()` 为 true 的 RC/RU 下，与 `allow_semi_consistent() == skip_gap_locks()` 自洽。`!unique_search` 的排除是因为唯一索引精确查找命中需精确语义，不应走"先旧版本再重读"。

---

## read view 与可见性判断

> 待补。涉及 ReadView 类（`read0types.h`）、`changes_visible` / `prepare`（`read0read.cc`）、低水位 `m_low_limit_no` 与 purge 的关系、`trx_undo_prev_version_build` 的完整回溯路径、二级索引 MVCC（`row_search_mvcc` 非锁定读分支 `row0sel.cc:5346` 起）。

---

## 核心调用栈

```
-- SQL 层开启 semi-consistent read --
mysql_update (sql_update.cc)
  → UpdateRowsIterator::Init (sql_update.cc:2851)
    → ha_innobase::try_semi_consistent_read(true) (ha_innodb.cc:10043)
      row_read_type = ROW_READ_TRY_SEMI_CONSISTENT

-- 引擎层扫描 --
ha_innobase::rnd_next / index_next
  → row_search_mvcc (row0sel.cc)
    → sel_set_rec_lock(..., SELECT_SKIP_LOCKED, ...) (row0sel.cc:5267)
      返回 DB_SKIP_LOCKED（行被锁，但不排队等待）
    → row_sel_build_committed_vers_for_mysql (row0sel.cc:721)
      → row_vers_build_for_semi_consistent_read (row0vers.cc:1368)
        → trx_undo_prev_version_build (row0vers.cc，沿 undo 链回溯已提交版本)
    → did_semi_consistent_read = true (row0sel.cc:5308)
    → row_read_type = ROW_READ_DID_SEMI_CONSISTENT (row0sel.cc:6077)

-- SQL 层判断 --
UpdateRowsIterator::DoImmediateUpdatesAndBufferRowIds (sql_update.cc:2419)
  → ha_innobase::was_semi_consistent_read (ha_innodb.cc:10037)
    true → continue，重新读同一行并正常加锁
  → 不满足 WHERE → ha_innobase::unlock_row (ha_innodb.cc:10028)
    DID 清回 TRY，跳过（不真正解锁，因为从未加锁）
```

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `transaction_isolation` | REPEATABLE-READ | Global/Session | 决定是否启用 semi-consistent read（RC/RU 启用） |

> `innodb_locks_unsafe_for_binlog` 在 8.0 已移除。

### 状态变量

暂无直接对应状态变量。

---

## Misc

### semi-consistent read vs 普通 MVCC 一致性读

- 普通一致性读（`select_lock_type == LOCK_NONE`）是纯快照读，从不碰锁，依据 read view 判断可见性。
- semi-consistent read 是**锁定读在加锁失败时的降级**，仅发生在 UPDATE/DELETE 光标下，读完带着"我其实没锁"的标记（DID）回上层，上层决定跳过或重读加锁。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/include/trx0trx.h:1138` | `skip_gap_locks()`，RC/RU 返回 true |
| `storage/innobase/include/trx0trx.h:1151` | `allow_semi_consistent()` == `skip_gap_locks()` |
| `storage/innobase/handler/ha_innodb.cc:10043` | `try_semi_consistent_read()` 设置 row_read_type |
| `storage/innobase/handler/ha_innodb.cc:10037` | `was_semi_consistent_read()` 判断 DID |
| `storage/innobase/handler/ha_innodb.cc:10028` | `unlock_row()` DID 清回 TRY |
| `storage/innobase/row/row0sel.cc:5264` | semi-consistent 触发条件 + SELECT_SKIP_LOCKED |
| `storage/innobase/row/row0sel.cc:5288` | DB_SKIP_LOCKED 分支，构建已提交版本 |
| `storage/innobase/row/row0sel.cc:721` | `row_sel_build_committed_vers_for_mysql()` |
| `storage/innobase/row/row0vers.cc:1368` | `row_vers_build_for_semi_consistent_read()` |
| `storage/innobase/include/row0mysql.h:1054` | row_read_type 三个常量 |
| `sql/sql_update.cc:2851` | UpdateRowsIterator 开启 semi-consistent read |
| `sql/sql_update.cc:2419` | DoImmediateUpdatesAndBufferRowIds 判断 |
| `sql/handler.h:5648` | semi-consistent read 协议注释 |
