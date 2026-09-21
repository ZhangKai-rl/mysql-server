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

无条件写入当前语句（或它的重写版本，若存在）。写入内容由 `log_event_time` / `log_event_type` 等组装，走 logger 框架（见 `error_log.md` 的 sink 体系）。

## throttle：未用索引慢查询的限流

`Slow_log_throttle`（继承 `Log_throttle`）：

- **时间窗口**：`window_usecs`（由 `log_throttle_queries_not_using_indexes` 决定，默认 0 关闭）
- **计数**：窗口内未用索引的慢查询数量，超限则 `suppress_logging`
- **汇总**：窗口结束时打印一条汇总（`total_exec_time` / `total_lock_time`）

`Slow_log_throttle::new_window` 相比基类多了 `total_exec_time` / `total_lock_time` 两个累加器，用于汇总输出。

## 相关变量

| 变量 | 作用 |
|---|---|
| `long_query_time` | 判定"慢"的阈值（秒） |
| `min_examined_row_limit` | 扫描行数门槛 |
| `log_queries_not_using_indexes` | 是否记"未用索引"的查询 |
| `log_throttle_queries_not_using_indexes` | 未用索引慢查询的限流窗口（0 关闭） |
| `log_slow_extra` | 慢日志附加字段 |
| `slow_query_log` / `log_output` | 开关与输出目标 |

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
