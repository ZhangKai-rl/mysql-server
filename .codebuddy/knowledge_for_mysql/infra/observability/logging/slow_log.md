# Slow Query Log：判定链与写入

> 基于 MySQL 8.0.39。慢查询日志的核心不是"写日志"，而是 **`log_slow_applicable()` 那条判定链**——一条语句到底算不算慢、要不要写，中间有多个独立维度在起作用。本篇把它拆开。

## 目录

- [概述](#概述)
- [log_slow_applicable：判定链逐段](#log_slow_applicable判定链逐段)
- [log_slow_do：真正写入](#log_slow_do真正写入)
- [throttle：未用索引慢查询的限流](#throttle未用索引慢查询的限流)
- [相关变量](#相关变量)
- [调用栈](#调用栈)

---

## 概述

慢查询日志在 `sql/log.cc` 的实现里，核心是两个函数：

```cpp
void log_slow_statement(THD *thd) {
  if (log_slow_applicable(thd)) log_slow_do(thd);
}
```

- `log_slow_applicable()`：**判定**这条语句要不要写（纯读状态，不写）
- `log_slow_do()`：**无条件写**当前语句（或它的重写版本）

调用点在 `sql/sql_parse.cc` 的两处：语句执行后（`log_slow_statement(thd)`）与结尾清理阶段。一条语句可能被判定两次。

★ 一个关键认知：**"慢查询"不等于"执行时间长"**。判定维度里有一条"未用索引"（`warn_no_index`），它和"时间慢"（`SERVER_QUERY_WAS_SLOW`）是**并列的或关系**——哪怕查询很快，只要没走索引、又开了 `log_queries_not_using_indexes`，也会进慢日志。

## log_slow_applicable：判定链逐段

> `sql/log.cc` 的 `log_slow_applicable()`，逐段拆解。

### ① 三个提前排除

```cpp
if (unlikely(thd->in_sub_stmt)) return false;  // 触发器/存储函数里不记
if (unlikely(thd->killed == THD::KILL_CONNECTION)) return false;
if (unlikely(thd->is_error()) &&
    (unlikely(thd->get_stmt_da()->mysql_errno() == ER_PARSE_ERROR)))
  return false;
```

| 排除 | 理由 |
|---|---|
| `in_sub_stmt` | 触发器/存储函数里的子语句不单独记（注释："Don't set time for sub stmt"） |
| `KILL_CONNECTION` | 被 kill 的连接不记 |
| `ER_PARSE_ERROR` | 解析错误不记 |

### ② 两个判定维度

```cpp
bool warn_no_index =
    ((thd->server_status &
      (SERVER_QUERY_NO_INDEX_USED | SERVER_QUERY_NO_GOOD_INDEX_USED)) &&
     opt_log_queries_not_using_indexes &&
     !(sql_command_flags[thd->lex->sql_command] & CF_STATUS_COMMAND));
bool log_this_query =
    ((thd->server_status & SERVER_QUERY_WAS_SLOW) || warn_no_index) &&
    (thd->get_examined_row_count() >= thd->variables.min_examined_row_limit);
```

**维度 A：`SERVER_QUERY_WAS_SLOW`** —— 真正的"慢"（执行时间超过 `long_query_time`），由 `set_time()` 里 `if (t - start_utime >= long_query_time) SERVER_QUERY_WAS_SLOW` 置位。

**维度 B：`warn_no_index`** —— "未用索引"告警，三个子条件**全部满足**：

1. `server_status` 里有 `NO_INDEX_USED` 或 `NO_GOOD_INDEX_USED`
2. 开了 `log_queries_not_using_indexes`
3. 不是 `CF_STATUS_COMMAND`（`SHOW` / `SET` 等状态命令不告警）

**共同门槛**：`examined_row_count >= min_examined_row_limit` —— 扫描行数太少的语句不进慢日志（`min_examined_row_limit` 默认 0）。

★ 所以完整语义是：**（慢 OR 未用索引）AND 扫描行数够多** 才记为慢查询。

### ③ long_query_count 计数（即使日志关闭也计数）

```cpp
// The docs say slow queries must be counted even when the log is off.
if (log_this_query) thd->status_var.long_query_count++;
```

文档明确要求：**即使慢日志关闭，慢查询计数（`long_query_count`）也要累加**。这个计数独立于"是否写日志"。

### ④ 开关 + throttle

```cpp
if (thd->enable_slow_log && opt_slow_log) {
  bool suppress_logging = log_throttle_qni.log(thd, warn_no_index);
  if (!suppress_logging && log_this_query) return true;
}
return false;
```

- `enable_slow_log`（会话级）+ `opt_slow_log`（全局级）都要开
- `log_throttle_qni.log(thd, warn_no_index)`：**throttle 只针对 `warn_no_index` 的慢查询**（防止"未用索引"的查询刷屏），真正慢的（`SERVER_QUERY_WAS_SLOW`）不受限流

## log_slow_do：真正写入

```cpp
void log_slow_do(THD *thd) {
  THD_STAGE_INFO(thd, stage_logging_slow_query);
  ...
}
```

```cpp
void log_slow_do(THD *thd) {
  THD_STAGE_INFO(thd, stage_logging_slow_query);
  if (thd->rewritten_query().length())
    query_logger.slow_log_write(thd, thd->rewritten_query().ptr(),
                                thd->rewritten_query().length(), false, 0, 0);
  else
    query_logger.slow_log_write(thd, thd->query().str, thd->query().length, false, 0, 0);
}
```

⚠️ **更正**：慢日志**不走**错误日志的 filter/sink 组件体系（那是 [`error_log.md`](error_log.md) 的内容）。它有**自己的两条输出路径**：

```cpp
  /* do not log slow queries from replication threads */
  if (thd->slave_thread && !opt_log_slow_replica_statements) return false;
  ...
  if (aggregate) { query_utime = exec_usec; lock_utime = lock_usec; }
  else if (thd->start_utime) { query_utime = (current_utime - thd->start_utime);
                               lock_utime = thd->get_lock_usec(); }
  else { query_utime = 0; lock_utime = 0; }
```

| `log_output` | 输出路径 |
|---|---|
| `FILE` | `File_query_log::write_slow()`（`sql/log.cc`） |
| `TABLE` | `Log_to_csv_event_handler::log_slow()`（写 `mysql.slow_log`） |

`aggregate=false, lock_usec=0, exec_usec=0` ⇒ **正常路径的时间都是 `slow_log_write` 内部现算的**，只有 throttle 汇总行才走 `aggregate=true`。

### ★ 写入点（4 处，不止 2 处）

| 调用点 | 文件 | 场景 |
|---|---|---|
| `dispatch_sql_command()` 多语句循环内 | `sql/sql_parse.cc` | 一条 packet 里有分号分隔的多条语句时，**每条**都判一次 |
| `dispatch_command()` 末尾 | `sql/sql_parse.cc` | 单语句 / COM_* 命令 |
| `sp_instr::execute()` 后 | `sql/sp_instr.cc` | **存储程序内的语句**，直接调 `log_slow_applicable` + `log_slow_do` |
| 复制 applier | `sql/log_event.cc` | `thd->update_slow_query_status(); log_slow_statement(thd);` |

### ★ 计时语义：`Query_time` 含 `Lock_time`（不扣除）

- **起点**：`THD::set_time()` 里 `start_utime = my_micro_time()`（语句开始执行时刻）
- **终点**：`slow_log_write` 里 `current_utime = my_micro_time()`，而这个调用发生在 `send_statement_status()` **之后** ⇒ **`Query_time` 含结果集发送时间**
- **`Lock_time`** 是独立累加器 `THD::inc_lock_usec()`：
  - server 层表锁：`sql/lock.cc` 里 `thd->inc_lock_usec(lock_end_usec - lock_start_usec)`
  - 引擎侧行锁：`thd_storage_lock_wait()`（`sql/sql_thd_api.cc`）也会计入 ⇒ **InnoDB 行锁等待也计入 `Lock_time`**

⇒ **`Query_time` 是墙钟总耗时，`Lock_time` 是它的子集，源码里没有任何 `Query_time - Lock_time` 的减法。**

⚠️ **时序差**：判定用的时刻在 `send_statement_status()` **之前**取，记录用的时刻在**之后**取 ⇒ 存在"判定不慢、但日志里 `Query_time` 略大于 `long_query_time`"的边界情况（反之不会）。

### 文本输出字段全集（`File_query_log::write_slow`）

| # | 输出 | 条件 | 说明 |
|---|---|---|---|
| 1 | `# Time: <ISO8601>` | 非 SHORT_LOG_FORMAT | 时间是**写日志那一刻**，不是语句开始 |
| 2 | `# User@Host: <u>  Id: <tid>` | 同上 | `user_host` 拼成 `"%s[%s] @ %s [%s]"` = `priv_user[user] @ host [ip]` |
| 3 | `# Query_time: ... Lock_time: ... Rows_sent: ... Rows_examined: ...` | 总是 | 见下 |
| 4 | `use <db>;` | 库与上一条不同 | |
| 5 | `SET timestamp=<sec>[,insert_id=N][,last_insert_id=N];` | | |
| 6 | `# administrator command: <Cmd>` | `query == nullptr`（Quit / Init DB 等非 SQL） | |
| 7 | SQL 文本 + `;` | 总是 | |

**第 3 项（关闭 `log_slow_extra` 时的基础格式）**：

```cpp
"# Query_time: %s  Lock_time: %s Rows_sent: %lu  Rows_examined: %lu\n"
```

时间由 `sprintf(..., "%.6f", ulonglong2double(x) / 1000000.0)` 生成（微秒→秒，6 位小数）。

⚠️ **两个"以为有、实际没有"的字段**（源码核实）：

| 字段 | 结论 |
|---|---|
| `Rows_affected` | ❌ **不存在**（`sql/log.cc` 全文无；只在 PFS / X Plugin） |
| `# Thread_pool_id` | ❌ **不存在**（全库零命中，社区版无线程池插件） |
| `Bytes_sent` | 只在 `log_slow_extra=ON` 时才有 |

## throttle：未用索引慢查询的限流

`Slow_log_throttle`（继承 `Log_throttle`）：

- **时间窗口**：`window_usecs`（由 `log_throttle_queries_not_using_indexes` 决定，默认 0 关闭）
- **计数**：窗口内未用索引的慢查询数量，超限则 `suppress_logging`
- **汇总**：窗口结束时打印一条汇总（`total_exec_time` / `total_lock_time`）

`Slow_log_throttle::new_window` 相比基类多了 `total_exec_time` / `total_lock_time` 两个累加器，用于汇总输出。

## ★ `log_slow_extra` 全字段（20 个）

### 机制：语句开始时全量快照

不是"多读几个计数器"，而是**在语句开始前把整个 `System_status_var` 拷一份**，写日志时逐项做差（`sql/sql_class.h`）：

```cpp
struct System_status_var status_var;              // Per thread statistic vars
struct System_status_var *copy_status_var_ptr;    // A copy of the statistic vars as
                                                  // of the start of the query
void copy_status_var(System_status_var *dst_var) { *dst_var = status_var;
                                                   copy_status_var_ptr = dst_var; }
```

采集入口（`sql/sql_parse.cc` ×2、`sql/srv_session.cc`、`sql/log_event.cc`）：

```cpp
  struct System_status_var query_start_status;    // ★ 栈上变量
  thd->clear_copy_status_var();
  if (opt_log_slow_extra) { thd->copy_status_var(&query_start_status); }
```

⇒ `log_slow_do` 必须在同一栈帧内完成，否则指针悬空。多语句 packet 每条之间调 `reset_copy_status_var()` 重置。

判定（`sql/log.cc`）：

```cpp
/*
  As a general rule, if opt_log_slow_extra is set, the caller will
  have saved state at the beginning of execution, and passed in a
  pointer to that state in THD's copy_status_var_ptr.
*/
if (!thd->copy_status_var_ptr) { /* 基础格式 */ } else { /* extra 格式 */ }
```

### 格式串（源码原文）

```cpp
"# Query_time: %s  Lock_time: %s"
" Rows_sent: %lu  Rows_examined: %lu"
" Thread_id: %lu Errno: %lu Killed: %lu"
" Bytes_received: %lu Bytes_sent: %lu"
" Read_first: %lu Read_last: %lu Read_key: %lu"
" Read_next: %lu Read_prev: %lu"
" Read_rnd: %lu Read_rnd_next: %lu"
" Sort_merge_passes: %lu Sort_range_count: %lu"
" Sort_rows: %lu Sort_scan_count: %lu"
" Created_tmp_disk_tables: %lu"
" Created_tmp_tables: %lu"
" Start: %s End: %s\n"
```

### 字段清单（20 个，逐个核实）

| 字段 | 源码表达式 | 说明 |
|---|---|---|
| `Thread_id` | `(ulong)thd->thread_id()` | 与 `# User@Host ... Id:` 同一个值 |
| `Errno` | `thd->is_error() ? ...mysql_errno() : 0` | 无错则 0 |
| `Killed` | `(ulong)thd->killed` | 数值枚举 |
| `Bytes_received` | `status_var.bytes_received - copy->bytes_received` | **差值** |
| `Bytes_sent` | `status_var.bytes_sent - copy->bytes_sent` | 差值 |
| `Read_first` | `ha_read_first_count` 差值 | handler 层计数 |
| `Read_last` | `ha_read_last_count` 差值 | |
| `Read_key` | `ha_read_key_count` 差值 | |
| `Read_next` | `ha_read_next_count` 差值 | |
| `Read_prev` | `ha_read_prev_count` 差值 | |
| `Read_rnd` | `ha_read_rnd_count` 差值 | |
| **`Read_rnd_next`** | `ha_read_rnd_next_count` 差值 | ★ **全表扫描的主要指标** |
| `Sort_merge_passes` | `filesort_merge_passes` 差值 | |
| `Sort_range_count` | `filesort_range_count` 差值 | |
| `Sort_rows` | `filesort_rows` 差值 | |
| `Sort_scan_count` | `filesort_scan_count` 差值 | |
| `Created_tmp_disk_tables` | 差值 | |
| `Created_tmp_tables` | 差值 | |
| `Start` | `make_iso8601_timestamp(...)` | `query_start_utime`（= `thd->start_time`）；为 0 则输出空串 |
| `End` | `Start + Query_time` | |

★ **所有计数器字段都是"语句内增量"，不是会话累计值**——这是它最有价值的一点，也是必须 `copy_status_var` 的原因。

### ★ 三个陷阱（源码实证）

1. **存储程序内的语句不支持 extra** —— `sql/sp_instr.cc`：
```cpp
/*
  We currently do not support --log-slow-extra for this case,
  and therefore pass in a null-pointer instead of a pointer to
  state at the beginning of execution.
*/
log_slow_do(thd);
```
（直接调 `log_slow_do` 而没先 `copy_status_var` ⇒ `copy_status_var_ptr == nullptr` ⇒ 落基础格式）
2. **对表输出无效** —— 变量文档串原文：`"Has no effect on logging to table."`
3. **非 FILE 目标下静默降级** —— `check_slow_log_extra` 只 push warning（`ER_SLOW_LOG_MODE_IGNORED_WHEN_NOT_LOGGING_TO_FILE`），变量照样设成 ON

## ★ 输出到表（`mysql.slow_log`）的差异

表结构（`scripts/mysql_system_tables.sql`）是 **CSV 引擎**（不支持索引/事务）：

```sql
CREATE TABLE IF NOT EXISTS slow_log (
  start_time TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  user_host MEDIUMTEXT NOT NULL, query_time TIME(6) NOT NULL, lock_time TIME(6) NOT NULL,
  rows_sent INTEGER NOT NULL, rows_examined INTEGER NOT NULL, db VARCHAR(512) NOT NULL,
  last_insert_id INTEGER NOT NULL, insert_id INTEGER NOT NULL,
  server_id INTEGER UNSIGNED NOT NULL, sql_text MEDIUMBLOB NOT NULL,
  thread_id BIGINT UNSIGNED NOT NULL
) engine=CSV CHARACTER SET utf8mb3 comment="Slow log";
```

| 差异 | 证据 |
|---|---|
| `log_slow_extra` 无效 | `Log_to_csv_event_handler::log_slow` 根本不读 `copy_status_var_ptr` |
| **`query_time`/`lock_time` 会被截断** | 注释原文：`"A TIME field can not hold the full longlong range; query_time or lock_time may be truncated without warning here, if greater than 839 hours (~35 days)"`；代码 `min(query_utime, TIME_MAX_VALUE_SECONDS * 1000000LL)` |
| **`rows_examined` 是 4 字节 INTEGER** | DBUG 注入注释 `rows_examined = 4294967294LL; // overflow 4-byte int` |
| 只有 12 列 | 无 `Thread_id` 之外的 extra 字段（实际上连 `Errno`/`Killed` 也没有） |
| 写入是真 INSERT | `ha_extra(HA_EXTRA_MARK_AS_LOG_TABLE)` → `ha_rnd_init` → `ha_write_row` |
| 不写 binlog | `thd->variables.option_bits &= ~OPTION_BIN_LOG;`，注释：`/* log table entries are not replicated */` |
| 表结构被改就停写 | `log_table_intact.check(&slow_query_log_table_def)` |

## 代价

**文件输出的固定开销**（`File_query_log::write_slow`）：

```cpp
  mysql_mutex_lock(&LOCK_log);
  ...  /* 7~10 次 my_b_write / my_b_printf */
  if (my_b_write(..., sql_text, sql_text_len) || my_b_write(..., ";\n", 2) ||
      flush_io_cache(&log_file)) goto err;
  mysql_mutex_unlock(&LOCK_log);
```

- **全局 `LOCK_log` 互斥** ⇒ 所有会话写慢日志**串行化**
- **每条记录都 `flush_io_cache`** ⇒ 没有批量缓冲，每条慢查询都是一次真实 flush
- 长 SQL 文本整体写入，越长越贵

**`log_slow_extra=ON` 新增的开销**：

1. **每语句（不只是慢查询）**在入口执行一次 `copy_status_var()`——整个 `System_status_var` 的**全量拷贝**，且**无条件**（只要开关 ON，每条语句都拷，不管最后写不写日志）
2. 多语句 packet 每条之间再拷一次
3. 写日志时：20 次减法 + 2 次时间戳格式化 + 一条显著更长的 `my_b_printf`

⚠️ **源码无直接证据**：`sql/log.cc` / `sql/sys_vars.cc` 中关于 `log_slow_extra` 的注释只有行为说明，**没有任何性能开销的数字**。上述"每语句全量拷贝"是从代码行为推出的结论。

## 与 PFS / sys schema 的关系

| 维度 | 慢日志（FILE） | `events_statements_summary_by_digest` |
|---|---|---|
| **SQL 文本** | **原文**（完整参数） | 只有归一化 `DIGEST_TEXT`（字面量换 `?`，受 `max_digest_length` 截断） |
| 聚合 | 每条一次，**无聚合** | 按 `SCHEMA_NAME + DIGEST` 聚合（COUNT/SUM/MIN/AVG/MAX） |
| 样本 | 只记"慢的" ⇒ **有偏** | 记**全部**，可算 p95/p99 |
| 时间 | `Query_time`（墙钟含发送）、`Lock_time` | `SUM_TIMER_WAIT`、`SUM_LOCK_TIME`、`SUM_CPU_TIME` |
| 全表扫描标记 | 需 `log_queries_not_using_indexes=ON` | `SUM_NO_INDEX_USED > 0` **默认就有** |
| 代价 | 每条一次 mutex + flush | 常驻内存计数器，无 I/O |

⇒ **互补**：慢日志回答"**哪一条**"（有原文），digest 回答"**哪一类**"（可聚合）。实战是**用 digest 找到问题模板，再用慢日志拿到真实参数**。

`sys.statements_with_runtimes_in_95th_percentile` 只是 digest 表的一个视图（筛选 `AVG_TIMER_WAIT` ≥ p95），注意它按 **avg** 选而不是按 p95 延迟选 ⇒ 高频小慢查询会入选，低频超慢查询可能落选。

## 相关变量（含默认值与定义处，均在 `sql/sys_vars.cc`）

| 变量 | 默认值 | 作用域 | 要点 |
|---|---|---|---|
| `long_query_time` | **`10`**（秒） | SESSION（double） | `ON_UPDATE(update_cached_long_query_time)` 同步微秒缓存；判定是**严格大于**（`>`），等于不算慢 |
| `min_examined_row_limit` | **`0`** | SESSION | |
| `log_queries_not_using_indexes` | **`false`** | GLOBAL | |
| `log_slow_admin_statements` | **`false`** | GLOBAL | ★ 见下 |
| `log_slow_replica_statements` | **`false`** | GLOBAL | 有 deprecated alias `log_slow_slave_statements` |
| `log_throttle_queries_not_using_indexes` | **`0`**（关闭） | GLOBAL | |
| `slow_query_log` | **`false`** | GLOBAL | `ON_UPDATE(fix_slow_log_state)` |
| `log_slow_extra` | **`false`** | GLOBAL | |
| `slow_query_log_file` | `nullptr`（默认 `hostname-slow.log`） | GLOBAL | `ON_UPDATE(fix_slow_log_file)` |

### ★ `log_slow_admin_statements` 的真实机制

它**不是** `log_slow_applicable` 里的独立判定条件，而是**在管理语句入口把 `thd->enable_slow_log` 改写为该变量的值**：

```cpp
  thd->enable_slow_log = opt_log_slow_admin_statements;
```

出现位置：`sql/sql_alter.cc`（ALTER / RENAME）、`sql/sql_admin.cc`（ANALYZE/CHECK/OPTIMIZE/REPAIR）、`sql/sql_partition_admin.cc`（EXCHANGE PARTITION）、`sql/sql_cmd_ddl_table.cc`（CREATE/DROP INDEX，注释："CREATE INDEX or DROP INDEX cause a full table rebuild and thus classify as slow administrative statements just like ALTER TABLE"）。

⇒ 因为最后一道门是 `thd->enable_slow_log && opt_slow_log`，所以**默认（false）下所有 ALTER / ANALYZE / OPTIMIZE / CHECK / REPAIR / CREATE INDEX / DROP INDEX 都不进慢日志，哪怕跑了 10 分钟**。

同理 `log_slow_replica_statements` 有两道门：applier 里 `thd->enable_slow_log = opt_log_slow_replica_statements;` + `slow_log_write` 兜底 `if (thd->slave_thread && !opt_log_slow_replica_statements) return false;`。

⚠️ **`log_slow_rate_limit` / `log_slow_rate_type` 在社区版 8.0.39 不存在**（全库零命中，是 Percona 专有）。

## 轮转：`FLUSH SLOW LOGS`

- 语法标志：`REFRESH_SLOW_LOG 8192`（`include/mysql_com.h`）
- 执行（`sql/sql_reload.cc`）：
```cpp
if ((options & REFRESH_SLOW_LOG) && opt_slow_log && (log_output_options & LOG_FILE))
  if (query_logger.reopen_log_file(QUERY_LOG_SLOW)) result = true;
```
⇒ **三个条件缺一不可**：`slow_query_log=ON` **且** `log_output` 含 FILE。

裸 `FLUSH LOGS` 会一次性带上 `REFRESH_BINARY_LOG | REFRESH_RELAY_LOG | REFRESH_SLOW_LOG | REFRESH_GENERAL_LOG | REFRESH_ENGINE_LOG | REFRESH_ERROR_LOG`。

## 调用栈

```
mysql_execute_command / 语句执行结束
  └─ log_slow_statement(thd)              sql/sql_parse.cc
       └─ log_slow_applicable(thd)        sql/log.cc
            ├─ 三个排除（in_sub_stmt / killed / parse error）
            ├─ warn_no_index 判定（server_status + 开关 + 非 status 命令）
            ├─ log_this_query = (慢 OR 未索引) AND 行数够
            ├─ long_query_count++（即使日志关闭）
            └─ enable_slow_log && opt_slow_log + throttle
       └─ log_slow_do(thd)                sql/log.cc
            └─ 组装 log 行 → logger 框架 → sink（见 error_log.md）
```

## 可观测性

```sql
-- 状态变量：慢查询计数
SHOW GLOBAL STATUS LIKE 'Slow_queries';   -- 即 long_query_count

-- 各维度开关
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';
SHOW VARIABLES LIKE 'min_examined_row_limit';
```

## 参考

- MySQL 8.0 Reference Manual, "The Slow Query Log"
- 相关文档：`error_log.md`（logger/sink 框架）、`general_log.md`（另一条日志路径）
