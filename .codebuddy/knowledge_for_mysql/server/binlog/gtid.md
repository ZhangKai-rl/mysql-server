# MySQL GTID 机制深度解析

> 基于 MySQL 8.0.39 源码，涵盖 SID/SIDNO/GNO 定义、Gtid_set 结构、GNO 分配算法、GTID 生命周期完整流程、三条持久化路径、Clone 双 buffer 机制。

## 目录

- [核心概念：SID / SIDNO / GNO](#核心概念sid--sidno--gno)
- [Gtid_set 结构](#gtid_set-结构)
- [GNO 分配算法](#gno-分配算法)
- [GTID 生命周期](#gtid-生命周期)
  - [服务器启动与初始化](#服务器启动与初始化)
  - [事务执行前——GTID 一致性检查](#事务执行前gtid-一致性检查)
  - [Flush Stage——GTID 分配与写入 binlog](#flush-stagegtid-分配与写入-binlog)
  - [Sync Stage——fsync 持久化](#sync-stagefsync-持久化)
  - [Commit Stage——GTID 外部化](#commit-stagegtid-外部化)
  - [InnoDB GTID 持久化（异步刷表）](#innodb-gtid-持久化异步刷表)
  - [Binlog Rotate 时的持久化](#binlog-rotate-时的持久化)
  - [崩溃恢复](#崩溃恢复)
  - [Rollback 时的 GTID 处理](#rollback-时的-gtid-处理)
- [GTID 持久化：三条路径（概览）](#gtid-持久化三条路径概览)
- [Clone_persist_gtid 双 buffer](#clone_persist_gtid-双-buffer)
- [关键源码位置速查](#关键源码位置速查)

---

## 核心概念：SID / SIDNO / GNO

### SID（Source ID）

128 位的 UUID，标识一个 MySQL 实例（准确说是标识一个 GTID 来源）。每个 server 在启动时生成或读取 `auto.cnf` 中的 UUID 作为自己的 SID。**SID 不是 server_id**（server_id 是 32 位整数，用于 binlog event 头和复制区分，与 GTID 体系无关）。

### SIDNO（Source ID Number）

SID 在本实例内的整数索引。`Sid_map` 维护一个 `SID` 到 `SIDNO` 的映射表，每个 SID 在本地被编号为 1, 2, 3...。SIDNO 是**本地编号**，非全局唯一——不同实例的 Sid_map 可能给同一个 SID 分配不同的 SIDNO。

### GNO（Global Transaction Number）

int64 序列号，与 SID 组合形成完整的 GTID（`SID:GNO`，如 `8e75c5ea-2403-11f0-b1bb-34210bd0f65d:17`）。

**GNO 与 `trx_id` / `trx_no` 完全无关**。`trx_id` 是 InnoDB 内部事务标识（写进行记录的 `DB_TRX_ID`），`trx_no` 是提交时分配的序列化号（决定 purge 和 MVCC），GNO 是复制层面的全局事务标识。三者属于不同层面。

### GTID 的表示

一个 GTID 由 `(SIDNO, GNO)` 二元组唯一确定。在 binlog 中以 `Gtid_log_event` 携带，在内存中以 `Gtid` 结构体表示。

---

## Gtid_set 结构

`Gtid_set` 是 GTID 集合的内存表示，采用**二维结构**：

```
Gtid_set
├── sidno=1 → [Interval(1-100), Interval(105-200), ...]   ← 该 SID 下已执行的 GNO 区间
├── sidno=2 → [Interval(1-50), ...]
├── sidno=3 → [Interval(1-1000), ...]
└── ...
```

- 按 SIDNO 索引，每个 SIDNO 下是一组 `Interval`（GNO 区间）
- `Prealloced_array<Interval *, 8>` 按 sidno 索引，8 是预分配数不是上限
- 区间用 `(start, end)` 表示连续的 GNO 范围，避免为每个 GNO 单独存储
- `executed_gtids`、`owned_gtids`、`purged_gtids` 都用 `Gtid_set` 表示

### gtids_only_in_table

`gtids_only_in_table` 表示只在 `mysql.gtid_executed` 表中、不在 binlog 中的 GTID。这种情况出现在从库开启 binlog 但关闭 `log_replica_updates` 时——事务通过引擎提交刷入 GTID 表，但不写 binlog，所以 binlog 中没有对应的 `Gtid_log_event`。

---

## GNO 分配算法

GNO 的自动分配由 `get_automatic_gno`（rpl_gtid_state.cc:413）完成。算法在 `executed_gtids`（已执行 GTID 集合）的间隙和 `owned_gtids`（已被占有但未提交的 GTID 集合）中寻找空闲的 GNO：

1. 从 `next_free_gno` 开始扫描
2. 跳过 `executed_gtids` 中已有的 GNO（已执行的不能重复分配）
3. 跳过 `owned_gtids` 中已有的 GNO（已被其他事务占有但未提交的不能分配）
4. 找到第一个空闲的 GNO，分配给当前事务

分配前事务需要获取 GNO 的 ownership（通过 `gtid_state->generate_automatic_gtid`），分配后事务持有该 GTID 的所有权直到 commit 或 rollback。

---

## GTID 生命周期

### 服务器启动与初始化

```
Gtid_state::init()
  → sid_map->add_sid(server_uuid)       // 注册本server UUID，得到server_sidno
  → next_free_gno = 1
  → read_gtid_executed_from_table()     // 从mysql.gtid_executed表加载executed_gtids
```

同时 InnoDB 从 undo log 恢复未刷表的 GTID：

```
trx_rseg_persist_gtid()
  → 遍历 rollback segment history list
  → 对 trx_no >= gtid_trx_no 的 undo log:
    → trx_undo_gtid_read_and_persist()  // 从undo header提取GTID
    → Clone_persist_gtid::add()         // 加入内存list
  → Clone_persist_gtid::flush_gtids()   // 恢复阶段首次刷表
  → m_thread_active = true              // 后台线程就绪
```

### 事务执行前——GTID 一致性检查

`gtid_pre_statement_checks` 检查：

- `gtid_next.type == AUTOMATIC` 时，当前语句是否违反 GTID 一致性
- `gtid_next.type == ASSIGNED` 时，GTID 是否已执行（executed_gtids）或被拥有（owned_gtids）

### Flush Stage——GTID 分配与写入 binlog

在 `process_flush_stage_queue`（binlog.cc:8471）中，leader 遍历 group 中的每个 THD：

```
assign_automatic_gtids_to_flush_group(first_seen)
  → 对每个 THD:
    → generate_automatic_gtid(head, ...)
      → sid_lock->rdlock()              // 全局读锁
      → lock_sidno(server_sidno)        // per-sidno锁
      → get_automatic_gno(sidno)        // 找空闲GNO
        → 从 next_free_gno 开始
        → 在 executed_gtids 区间间隙中找
        → 检查 owned_gtids 是否已被占用
      → acquire_ownership(thd, gtid)
        → owned_gtids.add_gtid_owner(gtid, thread_id)
        → thd->owned_gtid = {sidno, gno}
      → next_free_gno = gno + 1
```

随后 `Gtid_log_event` 写入 binlog：

```
flush_thread_caches(head)
  → binlog_cache_mngr::flush
    → binlog_cache_data::flush
      → Transaction_dependency_tracker::step()     // 分配 sequence_number
      → Binlog_event_writer 构造
      → MYSQL_BIN_LOG::write_transaction
        → Transaction_dependency_tracker::get_dependency()  // 确定last_committed
        → Gtid_log_event 构造
          → spec.set(thd->owned_gtid)    // GTID = (sidno, gno)
          → 携带 last_committed, sequence_number, transaction_length
        → MYSQL_BIN_LOG::write_cache     // 写入binlog文件
```

### Sync Stage——fsync 持久化

```
sync_binlog_file(false)
  → m_binlog_file->sync()   // fsync(binlog_fd)
  → GTID随binlog文件持久化到磁盘
```

### Commit Stage——GTID 外部化

```
process_commit_stage_queue(thd, commit_queue)
  → ha_commit_low(head)                    // 引擎层提交
  → gtid_state->update_commit_group(first)
    → global_sid_lock->rdlock()
    → update_gtids_impl_lock_sidnos(first)
    → 对每个 THD:
      → update_gtids_impl_own_gtid(thd, is_commit=true)
        → owned_gtids.remove_gtid(thd->owned_gtid)   // 释放所有权
        → executed_gtids._add_gtid(thd->owned_gtid)  // 加入已执行集合
        → thd->clear_owned_gtids()                   // 清理THD
        → gtid_next.set_undefined()                  // 重置

  → signal_done(final_queue)               // 唤醒所有follower
```

### InnoDB GTID 持久化（异步刷表）

```
事务InnoDB commit时:
  → Clone_persist_gtid::get_gtid_info(trx, gtid_desc)
    → 从 thd->owned_gtid 提取GTID字符串
  → Clone_persist_gtid::add(gtid_desc)
    → 加入 active_list (内存)

后台线程 periodic_write():
  → sleep(1秒)
  → flush_gtids()
    → switch_active_list()           // 切换双buffer
    → write_to_table()
      → 构造 Gtid_set
      → gtid_table_persistor->save()  // 写mysql.gtid_executed表
    → update_gtid_trx_no(oldest_trx_no)
      → trx_sys_persist_gtid_num()   // 持久化到系统表空间
    → check_compress() → compress()  // 定期压缩表
```

### Binlog Rotate 时的持久化

```
save_gtids_of_last_binlog_into_table()
  → logged_gtids_last_binlog = executed_gtids
                                - previous_gtids_logged
                                - gtids_only_in_table
  → previous_gtids_logged += logged_gtids_last_binlog
  → save(logged_gtids_last_binlog)   // 写gtid_executed表

新binlog文件开头:
  → Previous_gtids_log_event(executed_gtids)
    → 编码executed_gtids为二进制写入
```

### 崩溃恢复

```
服务器重启:
  1. read_gtid_executed_from_table()      // 从表加载executed_gtids
  2. mysql_bin_log.init_gtid_sets()       // 扫描binlog补充
     → 读最后一个binlog的Previous_gtids_log_event
     → 扫描后续Gtid_log_event
     → gtids_in_binlog_not_in_table = binlog中的 - 表中的
     → save(gtids_in_binlog_not_in_table)  // 补写表
  3. InnoDB: trx_rseg_persist_gtid()      // 从undo log恢复
     → 扫描 trx_no >= gtid_trx_no 的undo log
     → 提取GTID加入内存list
  4. Clone_persist_gtid::flush_gtids()    // 刷入表
  5. 服务器开放连接
```

### Rollback 时的 GTID 处理

```
update_gtids_impl_own_gtid(thd, is_commit=false)
  → owned_gtids.remove_gtid(thd->owned_gtid)  // 释放所有权
  → 不加入executed_gtids                       // 回滚不记录
  → if (sidno == server_sidno && next_free_gno > gno)
      next_free_gno = gno                       // 回退GNO，填补空洞
  → thd->clear_owned_gtids()
```

Rollback 后该 GNO 可被后续事务重新分配，避免 GNO 空洞永久浪费。

---

## GTID 持久化：三条路径（概览）

GTID 的持久化有三条路径，各自适用于不同场景：

### 路径一：binlog 文件（Gtid_log_event）

主库正常提交时，每个事务在 binlog 中写一个 `Gtid_log_event`，携带完整的 `SID:GNO`。从库回放时读取该 event 获知 GTID。这是最直接的持久化方式。

### 路径二：binlog rotate 刷表

当 binlog 文件发生轮转时，MySQL 将已执行的 GTID 批量写入 `mysql.gtid_executed` 表。这是一种**批量补刷**机制，避免每个事务都写表带来的性能开销。详见上文 [Binlog Rotate 时的持久化](#binlog-rotate-时的持久化)。

### 路径三：InnoDB undo log 异步刷表

对于不开 binlog 或 `log_replica_updates=OFF` 的实例（如从库），没有 `Gtid_log_event` 可依赖，GTID 通过 InnoDB undo log 持久化。详见上文 [InnoDB GTID 持久化（异步刷表）](#innodb-gtid-持久化异步刷表)。

---

## Clone_persist_gtid 双 buffer

Clone 插件在物理克隆时需要持久化 GTID 状态，使用 `Clone_persist_gtid` 的双 buffer 机制：

```
m_gtids[0]  ← active list（事务写入）
m_gtids[1]  ← flush list（后台线程读取刷表）
            ↑ 奇偶切换
```

- 事务提交时将 GTID 写入 **active list**
- 后台线程从 **flush list** 读取并刷入 `mysql.gtid_executed` 表
- 刷表完成后，两个 buffer 角色切换（奇偶翻转），原来的 active 变为 flush，原来的 flush 变为 active
- 切换在临界区内完成，保证不丢 GTID

这种双 buffer 设计避免了读写竞争——事务写和后台刷表操作不同的 buffer，只在切换瞬间需要同步。

---

## 关键源码位置速查

| 函数/变量 | 文件:行号 | 作用 |
|-----------|----------|------|
| `Gtid_state::init()` | rpl_gtid_state.cc | 启动时初始化 GTID 状态 |
| `read_gtid_executed_from_table()` | rpl_gtid_state.cc | 从 mysql.gtid_executed 表加载 |
| `get_automatic_gno` | rpl_gtid_state.cc:413 | 自动分配 GNO |
| `generate_automatic_gtid` | rpl_gtid_state.cc | 获取 GTID ownership |
| `assign_automatic_gtids_to_flush_group` | binlog.cc:8471 | flush stage 批量分配 GTID |
| `acquire_ownership` | rpl_gtid_state.cc | 获取 GTID 所有权 |
| `gtid_state->update_commit_group` | rpl_gtid_state.cc | commit 时批量更新 GTID |
| `update_gtids_impl_own_gtid` | rpl_gtid_state.cc | 单个事务 GTID 提交/回滚处理 |
| `Gtid_set` | rpl_gtid.h | GTID 集合二维结构 |
| `Sid_map` | rpl_gtid.h | SID 到 SIDNO 映射 |
| `Gtid_log_event` | log_event.cc:12983 | binlog 中携带 GTID 的事件 |
| `save_gtids_of_last_binlog_into_table` | binlog.cc | binlog rotate 刷表 |
| `mysql_bin_log.init_gtid_sets` | binlog.cc | 崩溃恢复时扫描 binlog |
| `trx_rseg_persist_gtid` | trx0rseg.cc | InnoDB 从 undo log 恢复 GTID |
| `trx_undo_gtid_read_and_persist` | trx0undo.cc | 从 undo header 提取 GTID |
| `Clone_persist_gtid` | clone_plugin.cc | Clone 双 buffer GTID 持久化 |
| `Clone_persist_gtid::flush_gtids` | clone_plugin.cc | 后台线程刷表 |
| `Clone_persist_gtid::periodic_write` | clone_plugin.cc | 后台线程入口 |
