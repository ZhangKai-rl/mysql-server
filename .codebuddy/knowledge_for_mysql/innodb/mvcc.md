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

gap lock 背景：next-key lock（record + gap）只在 RR 及以上（SERIALIZABLE）使用，解决幻读；RC/RU 不加 gap lock（`skip_gap_locks()` 为 true），故 RC 容忍幻读，也因此才能半一致性读。

### 为什么必须聚集索引

MVCC 版本链（undo 链）只挂在聚集索引记录上（`DB_ROLL_PTR` 指向 undo）；二级索引记录无版本链，可见性要回表判断。`row_vers_build_for_semi_consistent_read` 断言 `ut_ad(index->is_clustered())`（`row0vers.cc:1377`），触发条件 `index == clust_index`（`row0sel.cc:5268`）——"读旧版本"只能在聚集索引记录上做。

### SELECT_SKIP_LOCKED vs DB_SKIP_LOCKED

- `SELECT_SKIP_LOCKED`：`enum select_mode` 值（`prebuilt->select_mode`），对应 `FOR UPDATE SKIP LOCKED` 语义。
- `DB_SKIP_LOCKED`：`dberr_t` 返回值，`sel_set_rec_lock`（`row0sel.cc:1138`）→ `lock_clust_rec_read_check_and_lock` 在 sel_mode==SKIP_LOCKED 且记录被锁时返回。
- `row0sel.cc:5290-5291` 区分：`prebuilt->select_mode == SELECT_SKIP_LOCKED` → `goto next_rec`（用户显式 SKIP LOCKED，真跳过）；否则是半一致性读临时传的 SELECT_SKIP_LOCKED（只借"不等待"语义，`prebuilt->select_mode` 本身未变），走构建旧版本。

### 重读同一行的机制（游标不推进）

半一致性读路径（`row0sel.cc:5288-5314`）不走 `next_rec` 标签（PHASE 5 移动游标在 `next_rec` 内），游标停在原记录，`pcur->store_position`（`row0sel.cc:5785`）存好位置。第二次 `row_search_mvcc`（direction != 0）经 `sel_restore_position_for_mysql`（`row0sel.cc:4884`）恢复位置；`row0sel.cc:4897-4906`：游标停同一行且 `row_read_type == DID` 时不 `goto next_rec`，继续处理同一行。

第一次没锁、第二次加锁靠 `row_read_type` 翻转：TRY → `use_semi_consistent=true` → SELECT_SKIP_LOCKED → DB_SKIP_LOCKED → 没锁返回 DID；DID → `use_semi_consistent=false` → 普通模式悲观等待 → 返回置回 TRY（`row0sel.cc:6075-6080`）。

### prebuilt 生命周期

`row_prebuilt_t`（`row0mysql.h:553`，注释 "save CPU time"）缓存访问一张表所需全部上下文（index/trx/search_tuple/m_stop_tuple/mysql_template/pcur/ins_graph/upd_graph/select_lock_type/select_mode/row_read_type）。创建 `row_create_prebuilt`（`row0mysql.cc:805`）于 `ha_innobase::open`（`ha_innodb.cc:7397`）；销毁 `row_prebuilt_free`（`row0mysql.cc:957`）于 `ha_innobase::close`（`ha_innodb.cc:7691`）；`magic_n`（ROW_PREBUILT_ALLOCATED/FREED）防 use-after-free。

### 缓冲 row id 的场景

`UpdateRowsIterator::Read`（`sql_update.cc:2864`）分两阶段：扫描（`DoImmediateUpdatesAndBufferRowIds`，`sql_update.cc:2418`）+ 延迟更新（`DoDelayedUpdates`，`sql_update.cc:2584`，按 row id `ha_rnd_pos` 回表）。缓冲 row id 场景：① UPDATE 改了被扫描的键（`used_key_is_modified`，`sql_update.cc:647`，边扫边改会重读/漏行）；② ORDER BY（需先排序）；③ 多表 UPDATE（非 `m_immediate_table` 用 `StoreRowId` 写临时表 `m_tmp_tables`，`sql_update.cc:2531-2569`）；④ AFTER 触发器关 batch 反例（`sql_update.cc:849-856`）。半一致性读行在 `DoImmediateUpdatesAndBufferRowIds` 开头 `was_semi_consistent_read()` 直接 `return false`（`sql_update.cc:2420`），让嵌套循环迭代器重读同一行。

---

## read view 与可见性判断

### 两种读：一致性读 vs 当前读

`row_search_mvcc` 开头（`row0sel.cc:4855-4864`）按 `select_lock_type` 分流：

- `LOCK_NONE`（普通 SELECT）：一致性读，`trx_assign_read_view`（`row0sel.cc:4860`）分配 read view，用快照判断可见性。
- `LOCK_X/LOCK_S`（UPDATE/DELETE/`SELECT FOR UPDATE`）：当前读（locking read），走 else 分支加表意向锁（`lock_table`），**不开 read view**，读最新已提交版本。

所以 UPDATE 是当前读，不依赖快照 read view。

### ReadView 的字段（快照内容）

`ReadView::prepare`（read0read.cc:449）快照 `trx_sys` 当前状态，得到四个字段：

| 字段 | 含义 |
|------|------|
| `m_creator_trx_id` | 创建者事务自己 |
| `m_up_limit_id` | 上界 = 创建时**最小**活跃事务 id（`id < 它` → 已提交老事务） |
| `m_low_limit_id` | 下界 = 创建时**下一个**要分配的 trx id（`id >= 它` → 未来事务） |
| `m_ids` | 创建时正在活跃（未提交）的事务 id 有序数组 |

### 可见性判断规则（changes_visible）

`ReadView::changes_visible`（read0types.h:171-191）四条规则：

1. `id == m_creator_trx_id` → 自己改的，可见。
2. `id < m_up_limit_id` → 快照前已提交 → 可见。
3. `id >= m_low_limit_id` → 快照后才开始 → 不可见。
4. 中间区间 → `binary_search` 查 `m_ids`：活跃→不可见，已提交→可见。

本质是快照隔离（Snapshot Isolation）：版本可见 ⟺ 其事务在快照创建时刻已提交。

### read view 的分配与复用（trx_assign_read_view + Fast Path）

`trx_assign_read_view`（trx0trx.cc:2355）：已有活跃 view 则复用，否则 `MVCC::view_open` 新建。`view_open`（read0read.cc:531）有 Fast Path 复用：read view 对象池化（`m_free` 链表），用**指针 LSB 位当 closed 标记**（`p & ~1` 还原指针），避免反复 new/delete。

### 深层机制：有序数组、双序列、竞态

**ids_t 有序数组 + 二分**：`m_ids` 是 ReadView 专用的有序数组 `ids_t`（read0types.h:59，裸指针 `m_ptr`+`m_size`+`m_reserved`），只为支撑 `changes_visible` 的 `std::binary_search`（O(log n)）。`ids_t::insert`（read0read.cc:284）：`back() < value` 时 `push_back`（trx_id 递增分配的常见 fast path），否则 `upper_bound + memmove` 二分插入（罕见，如 `copy_complete` 插创建者 id）。

**快照数据源 rw_trx_ids 维护**：`trx_sys->rw_trx_ids`（trx0sys.h:558）是全局有序的"活跃 RW 事务 id"数组。开始：`trx_sys_allocate_trx_id` + `push_back`（trx0trx.cc:1323）；提交：`trx_erase_lists` 用 `lower_bound + erase` 删除并 `view_close`（trx0trx.cc:1871-1885）。`prepare` 在持 `trx_sys->mutex` 时 `copy_trx_ids(rw_trx_ids)` 冻结快照。

**竞态 1（rw_trx_ids 删除顺序）**：`trx0trx.cc:1966-1970` 注释——必须"先擦 rw_trx_ids/rw_trx_list、再移出 serialisation list"。否则中间窗口新 read view 会把该事务判"活跃不可见"，但其 undo 已因 `trx->no` 失效被 purge → missing history。

**竞态 2（Fast Path bug#117553）**：AC-NL-RO 事务无锁复用 view（`m_closed=false` 后不持 mutex 检查 `m_low_limit_id`），与 purge 的 `clone_oldest_view` 竞态，把过时的 `m_low_limit_id/no` 拷进 `purge_sys->view` → `changes_visible` 对已 purge 事务误判"不可见" → 读已 purge 的 undo page → CRASH（read0read.cc:546-591 注释含完整推导）。

**trx_id vs trx_no 双序列**：`trx->id` 事务开始分配（写 `DB_TRX_ID`），只用于 MVCC 可见性判断；`trx->no` 提交时在 serialisation list 分配（提交序，运行期恒 `TRX_ID_MAX`），只用于 purge 顺序 + `m_low_limit_no`。read view 用 id 判可见性（changes_visible），用 no 定 purge 边界（`m_low_limit_no = trx_get_serialisation_min_trx_no()`，read0read.cc:454，即 `trx_sys->serialisation_min_trx_no`）。

**view 软/硬关闭 + m_views 链表**：`view_close`（read0read.cc:774）两条路——软关闭（own_mutex=false，AC-NL-RO）只 `m_closed=true` + 指针 LSB 置 1，留在 m_views 不回收（供 Fast Path 复用）；硬关闭（own_mutex=true，RW）`close()+REMOVE(m_views)+ADD(m_free)` 回收。m_views 新 view 在头（`UT_LIST_ADD_FIRST`，read0read.cc:624），尾是最老；`get_oldest_view`（read0read.cc:657）从尾扫跳过 closed 找最老活跃 view。

**purge 交互**：`clone_oldest_view`（read0read.cc:726）`copy_prepare(*oldest_view)` 拷贝最老 view + `copy_complete` 插回创建者 id，再 `reduce_low_limit(gtid_oldest_trxno)`（:747）用 GTID 最老 trx_no 压低边界，阻止 purge 清理未刷进 `mysql.gtid_executed` 的 undo。

### RR vs RC：read view 生命周期

- RR（`> READ_COMMITTED`）：read view 第一次一致性读创建，事务结束才 `view_close`，整个事务复用 → 可重复读。
- RC（`<= READ_COMMITTED`）：每条语句结束 `view_close`（ha_innodb.cc:19500，注释 "each consistent read set its own snapshot"），下条语句重建 → 每次读最新已提交。

判定规则相同，差异全在"何时关闭重建"这一行 `view_close`。

### read view 快照 vs 最新已提交版本

- 一致性读：`ReadView::changes_visible()`（`read0read.cc`）判断，只返回"read view 创建时刻已提交"的版本（事务快照）。
- semi-consistent read：`row_vers_build_for_semi_consistent_read` 用 `trx_rw_is_active(version_trx_id)`（`row0vers.cc:1396`）判断"版本所属事务是否仍活跃"，只要**已提交**（不活跃）就返回，不管它是否在 read view 创建之后才提交。即读"最新已提交版本"，比快照读更新。

这符合 UPDATE 当前读的语义：更新必须基于最新已提交值，而非旧快照。

### read view 与 purge

read view 的低水位 `m_low_limit_no` 决定 purge 边界：purge 只能清理"所有活跃 read view 都看不到"的旧版本。`ReadView::prepare`（`read0read.cc`）计算 `m_low_limit_no`。GTID 持久化可压低边界（`read0read.cc` `reduce_low_limit`）。

### 快照读执行流程与版本回溯

`row_search_mvcc` 一致性读分支：先 `trx_assign_read_view` 分配快照，每扫到一条聚集索引记录用 `lock_clust_rec_cons_read_sees`（lock0lock.cc:231）判断——取 rec 的 `trx_id`（`row_get_rec_trx_id`）→ `view->changes_visible`。

- 可见 → 直接返回当前 rec。
- 不可见 → `row_sel_build_prev_vers_for_mysql`（row0sel.cc:3071）→ `trx_undo_prev_version_build`（row0vers.cc）沿 `DB_ROLL_PTR` 指向的 undo 链回溯，找到第一个可见版本。

二级索引走 `lock_sec_rec_cons_read_sees`（lock0lock.cc:268），因二级索引页信息不足，判定"不确定"时回表到聚集索引再判断（row0sel.cc:3306-3314）。临时表/只读模式直接可见（lock0lock.cc:246-249）。

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
