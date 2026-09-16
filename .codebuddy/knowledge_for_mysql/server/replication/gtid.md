# MySQL GTID 机制深度解析

> 基于 MySQL 8.0.39 源码，涵盖 SID/SIDNO/GNO 定义、Gtid_set 结构、GNO 分配算法、GTID 生命周期完整流程、三条持久化路径、Clone 双 buffer 机制。

## 目录

- [核心概念：SID / SIDNO / GNO](#核心概念sid--sidno--gno)
- [Gtid_set 结构](#gtid_set-结构)
- [GTID 相关系统变量：读写语义](#gtid-相关系统变量读写语义)
  - [gtid_mode](#gtid_mode)
  - [enforce_gtid_consistency](#enforce_gtid_consistency)
  - [gtid_next](#gtid_next)
  - [gtid_purged](#gtid_purged)
  - [gtid_executed / gtid_owned](#gtid_executed--gtid_owned)
  - [binlog_gtid_simple_recovery / session_track_gtids / gtid_executed_compression_period](#binlog_gtid_simple_recovery--session_track_gtids--gtid_executed_compression_period)
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

## GTID 相关系统变量：读写语义

> 变量描述符框架（`sys_var` 类树、SET 三趟、来源追踪）见 [`../infra/variables.md`](../infra/variables.md)。本节只讲这几个变量的**业务语义**：谁在何时能改、check/update 钩子做什么、背后维护什么状态。变量名（如 `fix_gtid_mode`）均已 grep 核实——8.0.39 中不存在的会明确标注。

类树总览：

```
sys_var
├── Sys_var_typelib → Sys_var_enum → Sys_var_gtid_mode            gtid_mode（GLOBAL）
├── Sys_var_multi_enum → Sys_var_enforce_gtid_consistency         enforce_gtid_consistency（GLOBAL）
├── Sys_var_gtid_next                                             gtid_next（SESSION_ONLY，直接继承 sys_var）
├── Sys_var_gtid_purged                                           gtid_purged（GLOBAL，非 READ_ONLY！）
└── Sys_var_charptr_func（构造即 READ_ONLY NON_PERSIST）
    ├── Sys_var_gtid_executed                                     gtid_executed（GLOBAL 只读）
    └── Sys_var_gtid_owned                                        gtid_owned（SESSION 只读）
```

### gtid_mode

**只有 4 个取值，不是 8 个**（`Gtid_mode::value_type`）：`OFF=0 / OFF_PERMISSIVE=1 / ON_PERMISSIVE=2 / ON=3`，默认 OFF。名字数组是 `Gtid_mode::names[]`（`rpl_gtid_mode.cc`）——**不存在** `gtid_mode_typelib`/`gtid_mode_names` 符号。

运行时修改的钩子是 `Sys_var_gtid_mode::global_update`（**不存在** `fix_gtid_mode`/`update_gtid_mode`，那是 5.7 的历史名字）。它的核心是**四把锁 + 系列约束检查 + 落地**：

```cpp
// 锁序：Gtid_mode::lock（trywrlock，抢不到直接报错不阻塞）
//      → channel_map.wrlock → mysql_bin_log.get_log_lock → global_sid_lock->wrlock
if (mysqld_server_started && abs((int)new_gtid_mode - (int)old_gtid_mode) > 1) {
  my_error(ER_GTID_MODE_CAN_ONLY_CHANGE_ONE_STEP_AT_A_TIME, MYF(0));  // 一次只能走一步
}
...
if (new_gtid_mode == Gtid_mode::ON && get_gtid_consistency_mode() != GTID_CONSISTENCY_MODE_ON) {
  my_error(ER_CANT_SET_GTID_MODE, MYF(0), "ON", "ENFORCE_GTID_CONSISTENCY is not ON");
}
...
// 落地
global_var(ulong) = new_gtid_mode;      // 写背板 Gtid_mode::sysvar_mode（供 SHOW/持久化）
global_gtid_mode.set(new_gtid_mode);    // 写原子值（全服务器其他代码读这个）
LogErr(SYSTEM_LEVEL, ER_CHANGED_GTID_MODE, ...);
mysql_bin_log.rotate(true, &dont_care); // 强制轮转 binlog，让新 Previous_gtids 反映新状态
```

约束清单：① **一次一步**（OFF→ON 必须经 OFF_PERMISSIVE→ON_PERMISSIVE，启动期 `mysqld_server_started==false` 免检，所以配置文件可直接设）；② ON 前要求 `enforce_gtid_consistency=ON`（双向联动，另一方向见下节）；③ ON 前要求无进行中的匿名事务（`get_anonymous_ownership_count()==0`）与无 AUTOMATIC 的 GTID-violating 事务；④ 设 OFF 前要求 `owned_gtids` 为空、无 AUTO_POSITION 通道、无 `WAIT_FOR_EXECUTED_GTID_SET` 等待者；⑤ 从 ON 往下改时无 `ASSIGN_GTIDS_TO_ANONYMOUS_TRANSACTIONS=LOCAL/UUID`、`GTID_ONLY`、`source_connection_auto_failover` 通道；⑥ GR 运行中禁止改非 ON。

启动路径不走钩子：`gtid_server_init()` 直接 `global_gtid_mode.set((value_type)Gtid_mode::sysvar_mode)`。

### enforce_gtid_consistency

值域 `OFF/ON/WARN`（别名 `FALSE/TRUE`），**8.0 默认 ON**。修改钩子 `Sys_var_enforce_gtid_consistency::global_update`（**不存在** `check_enforce_gtid_consistency`/`assert_enforce_gtid_consistency`，也**不扫描 binlog**）：

```cpp
global_sid_lock->wrlock();
// 与 gtid_mode 联动：gtid_mode==ON 时禁改非 ON
if (new_mode != GTID_CONSISTENCY_MODE_ON && gtid_mode == Gtid_mode::ON) {
  my_error(ER_GTID_MODE_ON_REQUIRES_ENFORCE_GTID_CONSISTENCY_ON, MYF(0)); goto err;
}
// 有进行中的 GTID-violating 事务（automatic + anonymous 两个计数）时：
//   目标是 ON → 报错 ER_CANT_ENFORCE_GTID_CONSISTENCY_WITH_ONGOING_GTID_VIOLATING_TX
//   OFF→WARN → 只警告
global_var(ulong) = new_mode;   // 写 _gtid_consistency_mode
LogErr(INFORMATION_LEVEL, ER_CHANGED_ENFORCE_GTID_CONSISTENCY, ...);
```

注意落地**没有** binlog rotate（与 gtid_mode 不同）。变量本身只是阈值状态——语句执行时由 `binlog.cc` 的检查点读 `get_gtid_consistency_mode()` 决定报错或警告（消费端）。

### gtid_next

SESSION-only、`NO_CMD_LINE`、默认 `"AUTOMATIC"`。三种用户可见输入形态（内部 `enum_gtid_type` 另有 `UNDEFINED/NOT_YET_DETERMINED/PRE_GENERATE` 三个内部态）：

| 输入 | 内部 | set_gtid_next 做什么 |
|---|---|---|
| `AUTOMATIC` | `AUTOMATIC_GTID=0` | 仅 `set_automatic()`，不获取任何所有权；提交时按 gtid_mode 决定生成 GTID 或匿名 |
| `ANONYMOUS` | `ANONYMOUS_GTID` | 要求 `gtid_mode != ON`；置 `owned_gtid.sidno = OWNED_SIDNO_ANONYMOUS(-2)` + `acquire_anonymous_ownership()` |
| `UUID:N` | `ASSIGNED_GTID` | 要求 `gtid_mode != OFF`；已执行则直接接受（语句稍后被跳过）；被占则 `wait_for_gtid` 阻塞；否则 `gtid_state->acquire_ownership()` 写 Owned_gtids + `thd->owned_gtid` |

`check_gtid_next`（真实存在）三重校验：存储函数/触发器内禁止（`ER_VARIABLE_NOT_SETTABLE_IN_SF_OR_TRIGGER`）、**多语句事务进行中禁止**（`ER_VARIABLE_NOT_SETTABLE_IN_TRANSACTION`，XA PREPARED 例外）、权限 `SESSION_VARIABLES_ADMIN`/`SYSTEM_VARIABLES_ADMIN`/`SUPER`/`REPLICATION_APPLIER`。

**关键语义：提交后 gtid_next 不是重置回 AUTOMATIC，而是进入 `UNDEFINED_GTID`**（`update_gtids_impl_own_gtid` 对 ASSIGNED 类型 `set_undefined()`），下一条语句被 `gtid_pre_statement_checks` 报 `ER_GTID_NEXT_TYPE_UNDEFINED_GTID` 强制"一个显式 GTID 只用于一个事务"；回滚/连接关闭的兜底才 `set_automatic()`（`sql_base.cc`，保证 DROP TEMPORARY TABLE 能生成自己的 GTID）。

### gtid_purged

**不是 READ_ONLY**——注册 flag 是 `NON_PERSIST GLOBAL_VAR(gtid_purged)`，无 READ_ONLY 位。所以运行时 `SET @@GLOBAL.gtid_purged` 能通过 resolve 的只读检查（权限仍要 SUPER/SYSTEM_VARIABLES_ADMIN），"只读型语义"由业务检查表达：

```cpp
// Gtid_state::add_lost_gtids —— 三个集合约束
if (!starts_with_plus) {
  if (!lost_gtids->is_subset(gtid_set))
    my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, "the new value must be a superset of the old value");
  gtid_set->remove_gtid_set(lost_gtids);       // 先剥掉已 purged 再查交集
}
if (executed_gtids.is_intersection_nonempty(gtid_set))
  my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, "must not overlap with @@GLOBAL.GTID_EXECUTED");
if (owned_gtids.is_intersection_nonempty(gtid_set))
  my_error(ER_CANT_SET_GTID_PURGED_DUE_SETS_CONSTRAINTS, "must not overlap with @@GLOBAL.GTID_OWNED");
// 落地：写 mysql.gtid_executed 表 → gtids_only_in_table/lost_gtids/executed_gtids 三集合都加
//      → broadcast_sidnos 唤醒 wait_for_gtid 等待者
```

即"binlog 必须未开/executed 为空"的直觉说法，源码里是**三集合交集检查**：新增部分与 executed（去掉 lost）不相交、与 owned 不相交、且新值必须是旧值超集（只增不减）。`check_gtid_purged` 另拒 GR 运行中、拒 `SET DEFAULT`（该变量无默认值，`ER_NO_DEFAULT`）。启动初始化**不走**这个 sys_var——`mysql_bin_log.init_gtid_sets()` 直接算好集合改 `Gtid_state::lost_gtids`（mysqldump 的 `--set-gtid-purged` 走运行时 SET 路径）。

### gtid_executed / gtid_owned

真 READ_ONLY（`Sys_var_charptr_func` 构造带 `READ_ONLY NON_PERSIST`），SET 在 resolve 阶段直接被拒。读源：global 读 `Gtid_state::executed_gtids`/`owned_gtids`（持 `global_sid_lock` wrlock 后 to_string）；session 读 `thd->owned_gtid`（`sidno==0` 显示空、`-2` 显示 "ANONYMOUS"、`-1` 读 `thd->owned_gtid_set`）。

**澄清：`gtid_current_pos` 在 MySQL 8.0.39 中不存在**（全仓库 grep 0 匹配）——它是 MariaDB 的变量，别当 MySQL 特性。

### binlog_gtid_simple_recovery / session_track_gtids / gtid_executed_compression_period

- **`binlog_gtid_simple_recovery` 默认是 true（ON），不是 false**；READ_ONLY（只能命令行/配置文件）。控制 `MYSQL_BIN_LOG::init_gtid_sets` 两处提前终止：反向扫描若最新 binlog 无任何 GTID 事件（`NO_GTIDS`）直接断定 executed/purged 为空；正向扫描只读第一个 binlog 的 `Previous_gtids_log_event` 就确定 purged。代价（注释明说）：旧 5.7.5 前 binlog + 混用 gtid_mode 的场景可能算出错误集合且不会自愈。
- **`session_track_gtids`**（8.0.26+）：`OFF/OWN_GTID/ALL_GTIDS` 三值，SESSION 作用域，默认 OFF。`on_update` 钩子调 `Session_gtids_tracker::update` 注册/注销 `Session_consistency_gtids_ctx` 监听；OWN_GTID 收集本会话刚提交的 `thd->owned_gtid`，ALL_GTIDS 提交后快照整个 `executed_gtids`；经 OK 包的 `SESSION_TRACK_GTIDS` 类型实体上报客户端。
- **`gtid_executed_compression_period`**：默认 0（8.0.23 起，之前 1000），控制 mysql.gtid_executed 表压缩线程 `compress_gtid_table` 的触发周期。**8.0.39 实测：该变量除注册/定义外没有任何代码读取**（`m_atomic_count` 无递增点），注释所称"按 period 计数触发"的路径已退化——压缩实际只由 binlog rotate 路径触发。变量仍在纯为向后兼容。

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
