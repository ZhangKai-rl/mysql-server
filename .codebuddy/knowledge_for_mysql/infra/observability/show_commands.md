# SHOW 命令族：机制、代价与逐命令解读

> **边界**：本篇讲 **SHOW 命令族**（除 `SHOW PROCESSLIST` 外）。
> - `SHOW PROCESSLIST` 详见 [`processlist.md`](processlist.md)（四入口对比，本篇只一句带过）
> - 状态变量的体系与 counter/gauge 分类见 [`status_variables.md`](status_variables.md)
> - PFS 事件表族见 [`pfs/pfs_events.md`](pfs/pfs_events.md)
>
> **一句话**：8.0 里多数 SHOW 已不是"独立实现"，而是被**改写成对 PFS 表或 DD system view 的一次 SELECT**——理解这点才能判断代价。

## 目录

- [一、概述](#一概述)
- [二、★ 实现机制：两条改写路径](#二-实现机制两条改写路径)
- [三、权限模型](#三权限模型)
- [四、★ `SHOW ENGINE INNODB STATUS`](#四-show-engine-innodb-status)
- [五、其余 SHOW 命令逐个](#五其余-show-命令逐个)
- [六、可观测性](#六可观测性)
- [七、★ 工程实现技法](#七-工程实现技法)
- [八、Misc（易混淆点）](#八misc易混淆点)
- [参考](#参考)

---

## 一、概述

### 是什么

`SHOW` 是 MySQL 最古老、最常用的观测入口。8.0 里它们的实现方式分三类：

| 类型 | 例子 | 特点 |
|---|---|---|
| **改写成 PFS 表 SELECT** | `SHOW STATUS` / `SHOW VARIABLES` | 走 PFS 物化流程 |
| **改写成 DD system view SELECT** | `SHOW TABLES` / `SHOW COLUMNS` / `SHOW INDEX` | 走 DD + I_S |
| **引擎/专用实现（无 plan）** | `SHOW ENGINE INNODB STATUS` / `SHOW BINLOG EVENTS` | 引擎自己输出自由文本 |

### 用途

- 人工排障的第一反应入口
- 很多监控脚本的历史依赖
- ⚠️ 但**部分 SHOW 代价很高**，不适合高频采集

### 版本演进（源码级）

- 8.0 把大量 I_S 表改为 **DD（Data Dictionary）上的 system view**，SHOW 直接改写成对这些 view 的 SELECT（权威清单见第二节的 `CF_SHOW_USES_SYSTEM_VIEW`）
- `SHOW PROFILES` 已在**解析阶段** push `ER_WARN_DEPRECATED_SYNTAX`（官方替代是 PFS）
- `SHOW ENGINE INNODB MUTEX`：**语法与实现都还在**，但输出依赖 latch monitor（默认关），且 SEMAPHORES 段的 spin 计数已恒为 0（见 4.7）⇒ 实用价值很低

---

## 二、★ 实现机制：两条改写路径

### 路径 A：改写成 PFS 表

`sql/sql_show_status.cc` 的注释原文：

```c
/**
  Build a replacement query for SHOW STATUS.
  When the parser accepts the following syntax:
    SHOW GLOBAL STATUS
  the parsed tree built for this query is in fact:
    SELECT * FROM
             (SELECT VARIABLE_NAME as Variable_name, VARIABLE_VALUE as Value
              FROM performance_schema.global_status) global_status
  ...
*/
```

四个入口函数：

```c
Query_block *build_show_session_status(...)   { table_name = "session_status"; ... }
Query_block *build_show_global_status(...)    { table_name = "global_status";  ... }
Query_block *build_show_session_variables(...) { table_name = "session_variables"; ... }
Query_block *build_show_global_variables(...)  { table_name = "global_variables"; ... }
```

⚠️ **`SHOW VARIABLES` 改写成的是 `performance_schema.global_variables` / `session_variables`，不是 `variables_info`**（后者是另一回事，见 `status_variables.md`）。

建完 parse tree 后 `sql_command` 会被**改回原 SHOW 命令** ⇒ 慢日志/审计里看到的仍是 `SHOW STATUS`。

### 路径 B：改写成 DD system view

`sql/dd/info_schema/show.cc`，命名空间 `dd::info_schema`，函数都以 `// Build a substitute query for SHOW XXX.` 开头：

| SHOW | 目标 system view |
|---|---|
| `SHOW CHARACTER SET` | `CHARACTER_SETS` |
| `SHOW COLLATION` | `COLLATIONS` |
| `SHOW DATABASES` | `SCHEMATA` |
| `SHOW TABLES` / `SHOW TABLE STATUS` | `TABLES`（靠 `include_status_fields` 布尔区分） |
| `SHOW COLUMNS/FIELDS` / `DESCRIBE` | `COLUMNS` |
| `SHOW INDEX/KEYS/INDEXES` | **`SHOW_STATISTICS`**（专门给 SHOW 用的视图，不是普通 `STATISTICS`） |
| `SHOW TRIGGERS` | `TRIGGERS` |
| `SHOW EVENTS` | `build_show_events_query` |
| `SHOW PROCEDURE/FUNCTION STATUS` | `build_show_procedures_query` |

统一拼装成 `SELECT * FROM (SELECT <field> AS <Alias> ... FROM <system_view>) <name> [WHERE ... LIKE ...] [ORDER BY ...]`，最后同样把 `sql_command` 改回原 SHOW 命令。

**★ 权威清单（哪些 SHOW 已迁移到 system view）** —— `sql/sql_parse.cc` 的标记位：

```c
  sql_command_flags[SQLCOM_SHOW_CHARSETS]     |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_COLLATIONS]   |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_DATABASES]    |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_TABLES]       |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_TABLE_STATUS] |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_FIELDS]       |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_KEYS]         |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_EVENTS]       |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_TRIGGERS]     |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_STATUS_PROC]  |= CF_SHOW_USES_SYSTEM_VIEW;
  sql_command_flags[SQLCOM_SHOW_STATUS_FUNC]  |= CF_SHOW_USES_SYSTEM_VIEW;
```

**反例（不是 system view）**：`SHOW OPEN TABLES` 走老 I_S schema-table 路径：

```c
Sql_cmd *PT_show_open_tables::make_cmd(THD *thd) {
  ...
  if (prepare_schema_table(thd, lex, nullptr, SCH_OPEN_TABLES)) return nullptr;
```

（其 flags 是 `CF_STATUS_COMMAND | CF_REEXECUTION_FRAGILE | CF_HAS_RESULT_SET`，**没有** `CF_SHOW_USES_SYSTEM_VIEW`）

### 命令对象体系（`sql/sql_show.h`）

```c
class Sql_cmd_show : public Sql_cmd_select {   // 有 plan 的 SHOW
class Sql_cmd_show_noplan : public Sql_cmd_show {  // 无 plan，直接 execute_inner
class Sql_cmd_show_schema_base : ...   // SHOW TABLES/TABLE STATUS/OPEN TABLES/TRIGGERS/EVENTS...
class Sql_cmd_show_table_base  : ...   // SHOW COLUMNS 与 SHOW KEYS 的公共基类
```

---

## 三、权限模型

**没有单一统一检查点**，是两级：

**① 被改写成 plan 的 SHOW**（基类）：

```c
bool Sql_cmd_show::check_privileges(THD *thd) {
  // If SHOW command is represented by a plan, ensure user has privileges:
  if (lex->query_tables == nullptr) return false;
  return check_table_access(thd, SELECT_ACL, lex->query_tables, false, UINT_MAX, false);
}
```

⇒ 本质是"对被改写成的底层表要有 SELECT 权限"。派生类再叠加自己的检查。

**② 无 plan 的 SHOW**（各自 override）：

| 命令 | 权限（源码原文） |
|---|---|
| `SHOW ENGINE STATUS` | `check_global_access(thd, PROCESS_ACL)` |
| `SHOW ENGINE MUTEX` | `check_global_access(thd, PROCESS_ACL)` |
| `SHOW ENGINE LOGS` | `check_access(thd, FILE_ACL, ...)` |
| `SHOW BINLOG EVENTS` / `SHOW RELAYLOG EVENTS` | `REPL_SLAVE_ACL` |
| `SHOW BINLOGS` / `SHOW MASTER STATUS` / `SHOW REPLICA STATUS` | `SUPER_ACL \| REPL_CLIENT_ACL` |
| `SHOW REPLICAS` | `REPL_SLAVE_ACL` |
| `SHOW CREATE DATABASE` | 无额外检查（`return false;`） |
| `SHOW FUNCTION/PROCEDURE CODE` | 实际检查在 `sp_head::show_routine_code` 内 |
| `SHOW EVENTS` | 源码注释明确 I_S/P_S 上不强制 ACL（返回空列表而非拒绝） |

---

## 四、★ `SHOW ENGINE INNODB STATUS`

### 4.1 实现与出口

`storage/innobase/handler/ha_innodb.cc`：

```c
/** Implements the SHOW ENGINE INNODB STATUS command. Sends the output of the
InnoDB Monitor to the client. ... */
static int innodb_show_status(handlerton *hton, THD *thd, stat_print_fn *stat_print) {
  static const char truncated_msg[] = "... truncated...\n";
  const long MAX_STATUS_SIZE = 1048576;
  if (srv_read_only_mode) { return 0; }
  mutex_enter(&srv_monitor_file_mutex);
  rewind(srv_monitor_file);
  srv_printf_innodb_monitor(srv_monitor_file, false, &trx_list_start, &trx_list_end);
  ...
```

分发：

```c
static bool innobase_show_status(handlerton *hton, THD *thd, stat_print_fn *stat_print,
                                 enum ha_stat_type stat_type) {
  switch (stat_type) {
    case HA_ENGINE_STATUS:  return (innodb_show_status(hton, thd, stat_print) != 0);
    case HA_ENGINE_MUTEX:   return (innodb_show_latch_status(hton, thd, stat_print) != 0);
    case HA_ENGINE_LOGS:    /* Not handled */ break;
  }
```

Server 侧统一出口（`sql/handler.cc` 的 `ha_show_status()`）：先发三列元数据 **`Type / Name / Status`**，然后调引擎的 `show_status`。⇒ 所有 `SHOW ENGINE xxx STATUS` 都是"三列 + 引擎自由文本"。

### 4.2 段清单（源码顺序，`srv0srv.cc` 的 `srv_printf_innodb_monitor()`）

| 段 | 生成函数 | 备注 |
|---|---|---|
| `INNODB MONITOR OUTPUT` + `Per second averages calculated from the last N seconds` | 本函数 | N = 距上次打印的秒数 |
| `BACKGROUND THREAD` | `srv_print_master_thread_info` | |
| `SEMAPHORES` | `sync_print` | |
| `LATEST FOREIGN KEY ERROR` | `ut_copy_file(dict_foreign_err_file)` | **条件段**：`!srv_read_only_mode && ftell(dict_foreign_err_file) != 0` |
| **`LATEST DETECTED DEADLOCK`** | `lock_print_info_summary`（lock0lock.cc） | **条件段**：`if (lock_deadlock_found)` |
| `TRANSACTIONS` | `lock_print_info_summary` + `lock_print_info_all_transactions` | |
| `FILE I/O` | `os_aio_print` | |
| `INSERT BUFFER AND ADAPTIVE HASH INDEX` | `ibuf_print` + AHI 分区 `ha_print_info` | |
| `LOG` | `log_print` | |
| `BUFFER POOL AND MEMORY` | `Total large memory allocated` / `Dictionary memory allocated` + `buf_print_io` | |
| `ROW OPERATIONS` | 本函数 | |
| `END OF INNODB MONITOR OUTPUT` | 本函数 | |

### 4.3 ★ `LATEST DETECTED DEADLOCK` 逐字段解读

#### 段的结构

`storage/innobase/lock/lock0lock.cc` 的 `Deadlock_notifier`：

```c
/** Handles writing the information about found deadlock to the log files
and caches it for future lock_latest_err_file() calls (for example used by
SHOW ENGINE INNODB STATUS)
  @param[in] trxs_on_cycle  trxs causing deadlock, i-th waits for i+1-th
  @param[in] victim_trx     the trx from trx_on_cycle which will be rolled back */
```

主体：

```c
  for (size_t i = 0; i < n; ++i) {
    const trx_t *trx = trxs_on_cycle[i];
    const trx_t *blocked_trx = trxs_on_cycle[0 < i ? i - 1 : n - 1];
    const lock_t *blocking_lock = lock_has_to_wait_in_queue(blocked_trx->lock.wait_lock, trx);
    print_title(i, "TRANSACTION");
    print(trx, 3000);
    print_title(i, "HOLDS THE LOCK(S)");
    print(blocking_lock);
    print_title(i, "WAITING FOR THIS LOCK TO BE GRANTED");
    print(trx->lock.wait_lock);
  }
  ... buff << "*** WE ROLL BACK TRANSACTION (" << (victim_pos + 1) << ")\n";
```

标题格式：`"\n*** (" << (pos_on_cycle + 1) << ") " << title << ":\n"` ⇒ **编号 = 环上位置 + 1**。

#### `*** (1) TRANSACTION:` 逐字段

由 `trx_print_low()` 输出（`max_query_len = 3000`）：

| 输出 | 源码 | 含义 |
|---|---|---|
| `TRANSACTION <id>` | `fprintf(f, "TRANSACTION " TRX_ID_FMT, trx_get_id_for_print(trx))` | 事务 ID |
| `, ACTIVE %lu sec` / `, ACTIVE (PREPARED) %lu sec` / `, not started` / `, forced rollback` / `, COMMITTED IN MEMORY` | `switch (trx_state)` | 状态与已活跃秒数（`now - trx->start_time`） |
| `starting index read` 等 | `op_info = trx->op_info` | InnoDB 在各操作步骤设置的当前动作描述 |
| ` recovered trx` | `if (trx->is_recovered)` | 崩溃恢复中的事务 |
| `, thread declared inside InnoDB %lu` | `if (trx->declared_to_be_inside_innodb)` | 已进入 InnoDB 的 ticket 数 |
| `mysql tables in use %lu, locked %lu` | 两者都 >0 才打印 | Server 层表数 / 已锁表数 |
| `LOCK WAIT ` / `ROLLING BACK ` / `COMMITTING ` | `switch (trx->lock.que_state)` | 前缀（源码注释：对 ACTIVE 事务这是 dirty read，为性能有意不加 trx->mutex） |
| `%lu lock struct(s), heap size %lu, %lu row lock(s)` | 见下 | |
| `, holds adaptive hash latch` | `if (trx->has_search_latch)` | |
| `, undo log entries %llu` | 仅 `trx->undo_no != 0` 时打印 | 已产生的 undo 记录数 |
| 最后一段 SQL | `innobase_mysql_print_thd(f, trx->mysql_thd, 3000)` | 最长 3000 字符 |

三个数量字段的取值：

```c
  trx_mutex_enter(trx);
  ulint n_rec_locks = lock_number_of_rows_locked(&trx->lock);   // row lock(s)
  ulint n_trx_locks = UT_LIST_GET_LEN(trx->lock.trx_locks);     // lock struct(s)
  ulint heap_size   = mem_heap_get_size(trx->lock.lock_heap);   // heap size
  trx_mutex_exit(trx);
```

⇒ `lock struct(s)` = 持有的 `lock_t` 个数；`row lock(s)` = 行锁 bitmap 中置位的行数；`heap size` = 锁内存堆字节数。

#### `HOLDS THE LOCK(S)` / `WAITING FOR THIS LOCK TO BE GRANTED`

由 `lock_rec_print()`（行锁）/ `lock_table_print()`（表锁）输出。行锁格式串：

```c
  fprintf(file,
          "RECORD LOCKS space id %lu page no %lu n bits %llu "
          "index %s of table ", ...);
  ut_print_name(file, lock->trx, lock->index->table_name);
  fprintf(file, " trx id " TRX_ID_FMT, trx_get_id_for_print(lock->trx));
```

逐字段：

| 字段 | 含义 |
|---|---|
| `space id X page no Y` | 锁所在 tablespace / page |
| `n bits Z` | 该页行锁 bitmap 的位数（≈ 该页记录数上限），**不是锁定行数** |
| `index <name> of table <db>/<tbl>` | 索引名与表名 |
| `trx id` | 持有该锁结构的事务 |
| `lock mode S` / `lock_mode X` | 共享 / 排他（源码里 S/X 两处文案不一致） |
| `locks gap before rec` / `locks rec but not gap` / `insert intention` / `waiting` | gap / record-not-gap / 插入意向 / 等待中 |
| `Record lock, heap no N` | 页内物理记录号（N=0/1 为 infimum/supremum） |
| `Physical record` / `n_fields ...` | 由 `rec_print_new()` 打印记录物理内容 |

★ **关键陷阱**：`HOLDS THE LOCK(S)` 打印的**不是该事务的全部锁**，而是**环上被阻塞方真正卡住的那一个锁**——源码是 `lock_has_to_wait_in_queue(blocked_trx->lock.wait_lock, trx)`。

`lock_trx_print_locks()` 还有一条限制：每事务最多打印 10 个锁，之后输出 `10 LOCKS PRINTED FOR THIS TRX: SUPPRESSING FURTHER PRINTS`。

#### ★ `*** WE ROLL BACK TRANSACTION (2)` —— 受害者选择

8.0.39 统一在 `storage/innobase/lock/lock0wait.cc` 的 `lock_wait_choose_victim()`：

```c
  for (auto *trx : sorted_trxs) {
    if (chosen_victim == nullptr) { chosen_victim = trx; continue; }
    if (trx_is_high_priority(chosen_victim) || trx_is_high_priority(trx)) {
      auto victim = trx_arbitrate(trx, chosen_victim);
      ...
    }
    if (trx_weight_ge(chosen_victim, trx)) {
      /* The joining transaction is 'smaller',
      choose it as the victim and roll it back. */
      chosen_victim = trx;
    }
  }
```

权重定义（`include/trx0trx.h`）：

```c
/** Calculates the "weight" of a transaction. The weight of one transaction
 is estimated as the number of altered rows + the number of locked rows. */
static inline uint64_t TRX_WEIGHT(const trx_t *t) {
  return t->undo_no + UT_LIST_GET_LEN(t->lock.trx_locks);
}
```

比较（`trx0trx.cc`）：

```c
/** Compares the "weight" (or size) of two transactions. Transactions that
 have edited non-transactional tables are considered heavier than ones
 that have not. */
bool trx_weight_ge(const trx_t *a, const trx_t *b) { ... }
```

**四条结论**：

1. **优先级仲裁优先于权重**：只要任一事务是 high priority（`thd_trx_priority > 0`），先由 `trx_arbitrate()` 裁定；后台线程事务（`mysql_thd == nullptr`）视为无优先级
2. 否则按 **`TRX_WEIGHT = undo_no（已改行数） + lock 结构数（锁行数）`** 比较，**权重小的被回滚**（回滚代价小）
   - ⚠️ 不是"undo 字节量"，是 **undo 记录条数**
3. **编辑过非事务表的事务一律更"重"**，不会被选为受害者
4. 选定后：`lock_wait_handle_deadlock()` → `lock_notify_about_deadlock()`（写 LATEST 段）→ `lock_wait_rollback_deadlock_victim()`

### 4.4 只保留最近一次 / 保留全部

存储（`lock0lock.cc`）：

```c
/** We store info on the latest deadlock error to this buffer. InnoDB
Monitor will then fetch it and print */
static bool lock_deadlock_found = false;
```

每次新死锁先清空再写（`Deadlock_notifier::start_print()`）：

```c
/** rewind(3) the file used for storing the latest detected deadlock and
print a heading message to stderr if printing of all deadlocks to stderr
is enabled. */
void Deadlock_notifier::start_print() {
  rewind(lock_latest_err_file);
  ut_print_timestamp(lock_latest_err_file);
  if (srv_print_all_deadlocks) {
    ib::info(ER_IB_MSG_643) << "Transactions deadlock detected, dumping detailed information.";
  }
}
```

输出条件段：

```c
  if (lock_deadlock_found) {
    fputs("------------------------\nLATEST DETECTED DEADLOCK\n------------------------\n", file);
    if (!srv_read_only_mode) { ut_copy_file(file, lock_latest_err_file); }
  }
```

**保留全部** = `innodb_print_all_deadlocks`：

```c
static MYSQL_SYSVAR_BOOL(
    print_all_deadlocks, srv_print_all_deadlocks, PLUGIN_VAR_OPCMDARG,
    "Print all deadlocks to MySQL error log (off by default)", nullptr, nullptr, false);
```

⇒ 同一份内容**同时写 error log**，不再受"最近一次"限制。

相关：`innodb_status_output`（定期把 monitor 输出到 error log）、`innodb_status_output_locks`（含锁明细，注释 `Requires innodb_status_output=ON.`）。

### 4.5 ★ 输出长度限制与截断

⚠️ 常量在 **InnoDB 侧**，不是 `sql/` 层：

```c
  const long MAX_STATUS_SIZE = 1048576;      // 1 MiB
```

（**`SHOW_ENGINE_STATUS_SIZE` 这个符号在本仓库不存在**，sql/ 下 grep 无结果）

三种情况：

```c
  if (flen > MAX_STATUS_SIZE) { usable_len = MAX_STATUS_SIZE; srv_truncated_status_writes++; }
  else                        { usable_len = flen; }
```

- 未超长 → 原样输出
- 超长且事务列表可跳过 → 中间插 `... truncated...\n`，注释：`/* Omit the beginning of the list of active transactions. */`
- 否则 → `/* Omit the end of the output. */` 直接截到 `MAX_STATUS_SIZE - 1`

★ **截断会被计数**并暴露为状态变量：

```c
  srv_truncated_status_writes++;      // → Innodb_truncated_status_writes
```

⇒ **判断"是否曾截断"要看这个状态变量是否 >0，不能只看单次输出**。

### 4.6 ★ 代价：持什么锁

按调用顺序：

1. **`srv_monitor_file_mutex`** —— 贯穿整个 `innodb_show_status` ⇒ **同一时刻只有一个连接能跑这个命令**
2. **`srv_innodb_monitor_mutex`** —— `srv_printf_innodb_monitor` 全程持有
3. ★ **`locksys` 全局排他 latch** —— 关键：

```c
  if (nowait) {
    locksys::Global_exclusive_try_latch guard{UT_LOCATION_HERE};
    if (guard.owns_lock()) srv_printf_locks_and_transactions(...);
    else { fputs("FAIL TO OBTAIN LOCK MUTEX, SKIP LOCK INFO PRINTING\n", file); ret = false; }
  } else {
    locksys::Global_exclusive_latch_guard guard{UT_LOCATION_HERE};   // SHOW 走这条：阻塞等待
    srv_printf_locks_and_transactions(file, trx_start_pos);
  }
```

4. **`trx_sys->mutex`** —— `lock_print_info_all_transactions` 内持有
5. 若 `innodb_status_output_locks=ON` 需打印记录内容，可能触发**磁盘读**，此时会临时释放全局 latch 与 `trx_sys->mutex`：
```c
/** Reads the page containing the record protected by the given lock.
This function will temporarily release the exclusive global latch and the
trx_sys_t::mutex if the page was read from disk. */
```
6. 还有 `dict_foreign_err_mutex`、AHI 分区 S latch、buffer pool 各统计 mutex

**"代价高"的注释级证据**（`srv0srv.cc` 顶部）：

```c
/** Maximum number of times allowed to conditionally acquire
mutex before switching to blocking wait on the mutex */
#define MAX_MUTEX_NOWAIT 20
/** Check whether the number of failed nonblocking mutex
acquisition attempts exceeds maximum allowed value. If so,
srv_printf_innodb_monitor() will request mutex acquisition
with mutex_enter(), which will wait until it gets the mutex. */
#define MUTEX_NOWAIT(mutex_skipped) ((mutex_skipped) < MAX_MUTEX_NOWAIT)
```

⇒ **后台周期性 monitor 用的是 nowait 模式**（拿不到就跳过）——这本身就是"该命令代价高，连 InnoDB 自己都要避免阻塞"的证据。

另：非 debug 构建下 `TRANSACTIONS` 段不打印 `Total number of lock structs in row lock hash table`（注释：`for performance reasons, see http://bugs.mysql.com/36942`）。

### 4.7 各段关键指标怎么读

#### SEMAPHORES

```c
  fprintf(file, "OS WAIT ARRAY INFO: reservation count " ULINTPF "\n", arr->res_count);
  fprintf(file, "OS WAIT ARRAY INFO: signal count " ULINTPF "\n", sg_count);
```

每个还在等的 cell：

```c
  fprintf(file,
          "--Thread %s has waited at %s line " ULINTPF " for %" PRId64 " seconds the semaphore:\n", ...);
```

- `reservation count`：**累计值**，进入 OS wait array 的次数（需两次采样求差）
- `signal count`：累计唤醒次数
- `--Thread ... has waited at <file> line <n> for X seconds the semaphore:`：**瞬时值**，X 很大 = 严重 latch 争用

⚠️ **★ 8.0 的重要变化**（源码注释原文）：

```c
/** Prints wait info of the sync system.
Note: The instrumental counters are deprecated
      and prints all 0 for compatibility. */
static void sync_print_wait_info(FILE *file) {
  fprintf(file, "RW-shared spins 0, rounds 0, OS waits 0\n" ...);
```

⇒ **`Mutex spin waits` / `RW-shared spins` / `Spin rounds per wait` 一类全部恒为 0，无观测价值**（8.0 已移除 spin 计数插桩，仅保留文案兼容）。

#### TRANSACTIONS

```c
  fprintf(file, "Trx id counter " TRX_ID_FMT "\n", trx_sys_get_next_trx_id_or_no());
  fprintf(file, "Purge done for trx's n:o < " TRX_ID_FMT " undo n:o < " TRX_ID_FMT " state: ", ...);
  fprintf(file, "History list length " UINT64PF "\n", trx_sys->rseg_history_len.load());
```

- `Trx id counter`：下一个将分配的 trx id
- `Purge done for trx's n:o < X`：purge 已推进到的 trx no；`state: running but idle` 表示 purge 线程在等新数据
- **`History list length`：未 purge 的 undo 长度**——最重要的 undo 压力指标，越大说明 purge 落后（长事务/大查询/view 未关）
- 之后 `LIST OF TRANSACTIONS FOR EACH SESSION:`：
```c
  fprintf(file, "------- TRX HAS BEEN WAITING %" PRId64 " SEC FOR THIS LOCK TO BE GRANTED:\n", ...);
```
⚠️ **只遍历 rw_trx_list**（注释 `We iterate over the RW trx list only.`）；autocommit 非锁定只读事务被省略（注释指向 `I_S.INNODB_TRX`）

#### LOG（`log0log.cc` `log_print()`）

```
Log sequence number          <current_lsn>
Log buffer assigned up to    <max_assigned_lsn>
Log buffer completed up to   <ready_for_write_lsn>
Log written up to            <write_lsn>
Log flushed up to            <flush_lsn>
Added dirty pages up to      <dirty_pages_added_up_to_lsn>
Pages flushed up to          <available_for_checkpoint_lsn>
Last checkpoint at           <last_checkpoint_lsn>
Log minimum file id is       <n>
Log maximum file id is       <n>
```

判读：

- `Log sequence number - Log flushed up to` = **未落盘 redo**
- `Log sequence number - Last checkpoint at` = **崩溃恢复需重做的量**
- `Pages flushed up to` 落后 `Log flushed up to` ⇒ dirty page 刷盘慢

#### BUFFER POOL AND MEMORY

```c
  fprintf(file, "Total large memory allocated " ULINTPF "\n"
                "Dictionary memory allocated %zu\n", ...);
```

每实例（`buf_print_io_instance()`）：

```
Buffer pool size / Free buffers / Database pages / Old database pages / Modified db pages /
Pending reads / Pending writes: LRU , flush list , single page /
Pages made young , not young / youngs/s, non-youngs/s /
Pages read , created , written / reads/s, creates/s, writes/s /
Buffer pool hit rate N / 1000, young-making rate ... not ... /
Pages read ahead X/s, evicted without access Y/s, Random read ahead Z/s /
LRU len: , unzip_LRU len: / I/O sum[..]:cur[..], unzip sum[..]:cur[..]
```

（`Database pages` = `lru_len`，`Old database pages` = `old_lru_len`，`Modified db pages` = `flush_list_len`）

#### ROW OPERATIONS

```c
  fprintf(file, "%" PRId32 " queries inside InnoDB, %" PRId32 " queries in queue\n",
          srv_conc_get_active_threads(), srv_conc_get_waiting_threads());
  /* This is a dirty read, without holding trx_sys->mutex. */
  fprintf(file, ULINTPF " read views open inside InnoDB\n", trx_sys->mvcc->size());
```

- `queries inside InnoDB / queries in queue`：InnoDB 并发控制（`innodb_thread_concurrency`）排队情况
- `read views open inside InnoDB`：活跃 read view 数，**长期很高 = 长事务**
- ⚠️ 每秒速率的分母是"距上次打印的秒数" ⇒ **两次 SHOW 间隔不同，速率含义不同，不能与状态变量的速率混用**

---

## 五、其余 SHOW 命令逐个

### 5.1 `SHOW ENGINE INNODB MUTEX`

- 语法仍在（`sql_yacc.yy` 有 `show_engine_mutex_stmt`），**源码未见移除或废弃标记**
- 权限 `PROCESS_ACL`；实现 `innodb_show_latch_status()` = mutex + rwlock 两部分
- **输出默认基本为空**：监控由 latch monitor 控制
```c
      if (MONITOR_IS_ON(MONITOR_LATCHES)) { mutex_monitor->enable(); }
```
  ⇒ 必须先 `SET GLOBAL innodb_monitor_enable='latch'`（或 `'all'`）
- 代价：持 `rw_lock_list_mutex` 遍历全局 rwlock 链表；打开 latch monitor 后本身还有插桩开销
- **结论**：实用价值很低（spin 计数已恒为 0），建议用 PFS 的 mutex/rwlock 实例表替代

### 5.2 `SHOW STATUS` / `SHOW GLOBAL STATUS` / `SHOW SESSION STATUS`

见 [`status_variables.md`](status_variables.md)。核心：**默认 SESSION**，三者都是 PFS 表的语法糖，`SHOW STATUS` 自身不计入统计。

### 5.3 `SHOW VARIABLES` / `SHOW GLOBAL VARIABLES`

→ `performance_schema.session_variables` / `global_variables`；默认同样是 SESSION。

### 5.4 ★ `SHOW ENGINE PERFORMANCE_SCHEMA STATUS`（很有价值）

`storage/perfschema/ha_perfschema.cc`：

```c
/**
  SHOW ENGINE PERFORMANCE_SCHEMA STATUS.
  ...
    Note about naming conventions:
    - Internal buffers exposed as a table in the performance schema are named
    after the table, as in 'events_waits_current'
    - Internal buffers not exposed by a table are named with parenthesis,
    as in '(pfs_mutex_class)'.
*/
```

逐项输出 `events_waits_current.size/.count`、`events_waits_history.size/.count/.memory`、`(pfs_mutex_class).size/.count/.memory`、`(pfs_rwlock_class).*` 等，并累加 `total_memory`。

`pfs.cc` 文档注释：

```
  This auditing interface consists of:
  SHOW ENGINE PERFORMANCE_SCHEMA STATUS;
  It displays data related to the memory usage of the performance schema,
  as well as statistics about lost events, if any.
```

**判读要点**：

- 三列 `Type=performance_schema / Name=<buffer>.size|.count|.memory / Status=<数值>`
- **带括号名的缓冲区 = 没有对应 PFS 表的内部数组**
- `total_memory` = PFS 总内存占用

⇒ **排查"PFS 占了多少内存""有没有丢事件"用这个**。

### 5.5 `SHOW CREATE TABLE` / `SHOW TABLE STATUS` / `SHOW INDEX`

- `SHOW CREATE TABLE|VIEW`：`Sql_cmd_show_create_table`（`m_is_view` 区分）。执行体在**局部 LEX 上展开表/视图**：
```c
  // Prepare a local LEX object for expansion of table/view
  LEX *old_lex = thd->lex; LEX local_lex; Pushed_lex_guard lex_guard(thd, &local_lex);
  lex->only_view = m_is_view;
  // Disable constant subquery evaluation as we won't be locking tables.
  lex->context_analysis_only = CONTEXT_ANALYSIS_ONLY_VIEW;
```
  代价：需要 DD 读取与 MDL（`Sql_cmd_show_table_base` 系列有 `try_acquire_high_prio_shared_mdl_lock()`，注释说明为避免与 MDL 子系统外的死锁，冲突时选择不等待）
- `SHOW TABLE STATUS` → I_S `TABLES` system view（`include_status_fields=true`）。**动态统计可能需要开表取**：

```c
/** The class hold dynamic table statistics for a table.
  ... Overall aim of introducing this cache is to avoid making
  multiple calls to same SE API to retrieve the statistics. */
class Table_statistics { ... };
enum class enum_table_stats_type { TABLE_ROWS, TABLE_AVG_ROW_LENGTH, DATA_LENGTH,
  MAX_DATA_LENGTH, INDEX_LENGTH, ..., CHECK_TIME, INDEX_COLUMN_CARDINALITY };
```
  ⇒ 大 schema 上会慢
- `SHOW INDEX|KEYS` → I_S **`SHOW_STATISTICS`**，排序 `ORDER BY INDEX_ORDINAL_POSITION, COLUMN_ORDINAL_POSITION`

### 5.6 `SHOW MASTER STATUS` / `SHOW REPLICA STATUS`

权限：`SUPER_ACL | REPL_CLIENT_ACL`。字段名（8.0.39 已改为 source/replica 术语）：

```c
  field_list->push_back(new Item_empty_string("Relay_Source_Log_File", FN_REFLEN));
  field_list->push_back(new Item_empty_string("Replica_IO_Running", 3));
  field_list->push_back(new Item_empty_string("Replica_SQL_Running", 3));
  field_list->push_back(new Item_return_int("Seconds_Behind_Source", 10, MYSQL_TYPE_LONGLONG));
  field_list->push_back(new Item_return_int("SQL_Delay", 10, MYSQL_TYPE_LONG));
  field_list->push_back(new Item_return_int("SQL_Remaining_Delay", 8, MYSQL_TYPE_LONG));
  field_list->push_back(new Item_empty_string("Replica_SQL_Running_State", 20));
```

**延迟判定的坑**（源码可证）：

- `Seconds_Behind_Source` 是 `LONGLONG` 且**可为 NULL**；与 source 时钟不可比时会**置 0 并告警**：
```c
    LogErr(WARNING_LEVEL, ER_RPL_REPLICA_SECONDS_BEHIND_SOURCE_DUBIOUS, ...);
```
  （注释：`mi->clock_diff_with_master = 0; /* The "most sensible" value */`）
- `SQL_Delay` 是**配置的**延迟；`SQL_Remaining_Delay` 仅当处于 `stage_sql_thd_waiting_until_delay` 时才算，否则 NULL
- **`Seconds_Behind_Source` 只看"已收到的 event 落后多少"** ⇒ IO 线程断了它**不反映真实落后**
- 更可靠：配合 `performance_schema.replication_connection_status` / `replication_applier_status_by_worker`

### 5.7 `SHOW BINLOG EVENTS`

`mysql_show_binlog_events()`（`sql/binlog.cc`），权限 `REPL_SLAVE_ACL`。

代价：**打开并顺序扫描 binlog 文件**；不开 `LIMIT` 在大 binlog 上代价显著。

### 5.8 `SHOW OPEN TABLES`

- **走老 I_S schema-table 路径**（`SCH_OPEN_TABLES`），不是 system view
- flags：`CF_STATUS_COMMAND | CF_REEXECUTION_FRAGILE | CF_HAS_RESULT_SET`
  ⇒ `CF_REEXECUTION_FRAGILE` 意味着**不能被安全重执行**（与 prepared statement / 复制重放相关）
- 代价：遍历 table cache / open tables 列表

### 5.9 `SHOW WARNINGS` / `SHOW ERRORS`

`mysqld_show_warnings(THD *thd, ulong levels_to_show)`（`sql/sql_error.cc`）。

**ERRORS 就是 WARNINGS 的一个 level 过滤**：

```c
bool Sql_cmd_show_errors::execute_inner(THD *thd) {
  return mysqld_show_warnings(thd, 1UL << (uint)Sql_condition::SL_ERROR);
}
```

⚠️ 语义关键：**是 per-connection 的诊断区（Diagnostics Area）**，每次语句会清空/覆盖。

### 5.10 `SHOW PROFILE` / `SHOW PROFILES`

**8.0.39 未移除，但 `SHOW PROFILES` 已明确废弃** —— `sql_yacc.yy`：

```
show_profiles_stmt:
          SHOW PROFILES_SYM
          {
            push_warning_printf(YYTHD, Sql_condition::SL_WARNING,
                                ER_WARN_DEPRECATED_SYNTAX, ...,
                                "SHOW PROFILES", "Performance Schema");
```

其他证据：

- 实现仍在 `sql/sql_profile.cc`（`PROFILING::show_profiles()`）
- 受编译开关控制：
```c
#if defined(ENABLED_PROFILING) ...
#else my_error(ER_FEATURE_DISABLED, MYF(0), "SHOW PROFILES", "enable-profiling");
```
- 状态变量 `Com_show_profile` / `Com_show_profiles` 仍在

⚠️ **`SHOW PROFILE`（单数）这条路径未见同样的语法层废弃警告（源码无直接证据）**，但两者共用同一套已废弃的 PROFILING 设施。官方替代：Performance Schema。

### 5.11 `SHOW PROCESSLIST`

实现在 `sql/sql_show_processlist.cc`，注释原文：

```
  If the Performance Schema is configured to support SHOW PROCESSLIST,
  then execute a query on performance_schema.processlist. Otherwise,
  fall back to the legacy method.
```

详见 [`processlist.md`](processlist.md)。

---

## 六、可观测性

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| InnoDB 整体状态 | SHOW | `SHOW ENGINE INNODB STATUS`（各段判读见 4.7） |
| **死锁详情** | SHOW | `SHOW ENGINE INNODB STATUS` → `LATEST DETECTED DEADLOCK`（逐字段见 4.3） |
| **所有死锁（不只最近一次）** | 变量 + error log | `SET GLOBAL innodb_print_all_deadlocks=ON` |
| **PFS 内存占用 / 丢事件** | SHOW | `SHOW ENGINE PERFORMANCE_SCHEMA STATUS` |
| 状态变量 | SHOW / PFS | `SHOW GLOBAL STATUS` 或 `performance_schema.global_status`（见 status_variables.md） |
| 系统变量来源 | PFS 表 | `performance_schema.variables_info`（见 status_variables.md） |
| 表结构 | SHOW | `SHOW CREATE TABLE` / `SHOW INDEX` |
| 复制延迟 | SHOW | `SHOW REPLICA STATUS`（注意 `Seconds_Behind_Source` 的坑，见 5.6） |
| binlog 内容 | SHOW | `SHOW BINLOG EVENTS`（务必带 LIMIT） |
| 当前会话的警告 | SHOW | `SHOW WARNINGS`（per-connection，会被下一条语句覆盖） |
| 当前有哪些连接 | SHOW / PFS | 见 [`processlist.md`](processlist.md) |

### 典型用法

```sql
-- ① InnoDB 死锁（LATEST 段）
SHOW ENGINE INNODB STATUS\G      -- 找 LATEST DETECTED DEADLOCK 段

-- ② 打开全部死锁记录（重启后失效，需写配置）
SET GLOBAL innodb_print_all_deadlocks = ON;

-- ③ PFS 自身内存与丢事件
SHOW ENGINE PERFORMANCE_SCHEMA STATUS;

-- ④ 判断是否曾截断过 InnoDB monitor 输出
SELECT VARIABLE_VALUE FROM performance_schema.global_status
 WHERE VARIABLE_NAME = 'Innodb_truncated_status_writes';
```

---

## 七、★ 工程实现技法

### 7.1 用"改写为 SELECT"复用优化器

把 SHOW 改写成 `SELECT * FROM (SELECT ... FROM <table>) alias WHERE ...` ⇒ 直接复用查询优化器（索引下推、WHERE 过滤、排序），不必为每个 SHOW 单独实现。这是 8.0 大量 SHOW 的实现方式。

（代价：过滤发生在物化之后，所以 `LIKE` 只省输出不省物化——见 `status_variables.md`）

### 7.2 用 nowait latch 避免自伤

后台 periodic monitor 用 `MUTEX_NOWAIT` 模式（最多试 20 次，拿不到就跳过并打印 `FAIL TO OBTAIN LOCK MUTEX, SKIP LOCK INFO PRINTING`）——**监控不能把自己变成故障源**。

### 7.3 用计数器暴露"看不见的降级"

截断不只在输出里插 `... truncated...`，还累加 `srv_truncated_status_writes` 并暴露为状态变量 ⇒ 单次输出可能看不到，但计数器能发现"曾经发生过"。

### 7.4 局部 LEX 做视图/表展开

`SHOW CREATE TABLE` 在**局部 LEX**上展开，避免污染主查询的 LEX，并 `context_analysis_only = CONTEXT_ANALYSIS_ONLY_VIEW` 禁用常量子查询求值（因为不锁表）。

### 7.5 废弃用"解析期 warning"而非直接删

`SHOW PROFILES` 在 yacc 里就 push `ER_WARN_DEPRECATED_SYNTAX` ⇒ 给用户迁移窗口，同时保留兼容性。

---

## 八、Misc（易混淆点）

| 混淆 | 澄清 |
|---|---|
| SHOW 都是独立实现？ | ❌ 8.0 里多数被**改写成 SELECT**（PFS 表或 DD system view） |
| `SHOW VARIABLES` 改写的是 `variables_info`？ | ❌ 是 `performance_schema.global_variables` / `session_variables`；`variables_info` 是记录"变量来源"的另一个表 |
| `SHOW OPEN TABLES` 也走 system view？ | ❌ 走**老 I_S schema-table**路径（flags 无 `CF_SHOW_USES_SYSTEM_VIEW`） |
| `SHOW ENGINE INNODB STATUS` 能并发跑？ | ❌ `srv_monitor_file_mutex` 全程持有 ⇒ **同一时刻只有一个** |
| 死锁段是"全部死锁"？ | ❌ **只有最近一次**（除非开 `innodb_print_all_deadlocks`） |
| 判断截断看输出里的 `... truncated...`？ | ⚠️ 更可靠是看状态变量 **`Innodb_truncated_status_writes`**（单次输出可能恰好没被截断） |
| 截断上限是 `SHOW_ENGINE_STATUS_SIZE`？ | ❌ **本仓库无此符号**；是 InnoDB 的 `MAX_STATUS_SIZE = 1048576`（1 MiB） |
| `SEMAPHORES` 段的 spin 数有参考价值？ | ❌ **恒为 0**（8.0 已移除插桩，注释明确 `deprecated ... prints all 0 for compatibility`） |
| 受害者是"事务大的"被回滚？ | ❌ 是 **权重小的**被回滚（`TRX_WEIGHT = undo_no + lock 结构数`）；且**编辑过非事务表的一律更"重"不会被选**；high priority 事务有仲裁优先 |
| `TRX_WEIGHT` 用的是 undo 字节量？ | ❌ 是 **undo 记录条数** |
| `Seconds_Behind_Source` 能反映真实落后？ | ⚠️ 只看**已收到的 event**；IO 线程断了它不反映；且与 source 时钟不可比时会置 0 并告警 |
| `SHOW ERRORS` 与 `SHOW WARNINGS` 是两套实现？ | ❌ **同一实现**，ERRORS 只是 level 过滤 |
| `SHOW PROFILE` 被移除了？ | ❌ 语法与实现都在（受 `ENABLED_PROFILING` 编译开关）；`SHOW PROFILES` 有明确的废弃 warning |
| `SHOW ENGINE INNODB MUTEX` 能直接看到 mutex？ | ❌ **默认空**，需先 `SET GLOBAL innodb_monitor_enable='latch'` |

---

## 参考

- *MySQL 8.0 Reference Manual*: *SHOW Statements* / *SHOW ENGINE Syntax* / *InnoDB Standard Monitor and Lock Monitor Output* / *SHOW PROFILE*
- 源码：
  - 机制：`sql/sql_yacc.yy`、`sql/parse_tree_nodes.{h,cc}`、`sql/sql_show.h`、`sql/sql_show.cc`、`sql/sql_show_status.cc`、`sql/sql_parse.cc`（`CF_SHOW_USES_SYSTEM_VIEW` 清单）、`sql/dd/info_schema/show.cc` + `show_query_builder.cc`
  - `SHOW ENGINE INNODB STATUS`：`storage/innobase/handler/ha_innodb.cc`（`innodb_show_status` / `innobase_show_status`）、`storage/innobase/srv/srv0srv.cc`（`srv_printf_innodb_monitor` + `srv_stages` + `MAX_MUTEX_NOWAIT`）、`storage/innobase/lock/lock0lock.cc`（`Deadlock_notifier` / `lock_rec_print`）、`storage/innobase/lock/lock0wait.cc`（`lock_wait_choose_victim`）、`storage/innobase/trx/trx0trx.{h,cc}`（`TRX_WEIGHT` / `trx_weight_ge` / `trx_print_low`）、`storage/innobase/sync/sync0sync.cc`（`sync_print`）、`storage/innobase/log/log0log.cc`（`log_print`）、`storage/innobase/buf/buf0buf.cc`（`buf_print_io`）
  - 其余命令：`sql/binlog.cc`、`sql/rpl_source.cc`、`sql/rpl_replica.cc`、`sql/sql_error.cc`、`sql/sql_profile.cc`、`sql/sql_reload.cc`
  - PFS STATUS：`storage/perfschema/ha_perfschema.cc`
- [`processlist.md`](processlist.md)（`SHOW PROCESSLIST`）· [`status_variables.md`](status_variables.md)（状态变量体系 + counter/gauge）· [`pfs/pfs_events.md`](pfs/pfs_events.md)（事件表族）· [`../README.md`](../README.md)（观测手段全景与决策树）
