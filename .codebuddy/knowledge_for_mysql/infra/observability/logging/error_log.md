# Error Log：log service / sink 三层架构

> 基于 MySQL 8.0.39。8.0 的错误日志是一次**架构重构**：从"写文件"变成"可插拔的 sink 组件体系"。本篇剖析这套三层架构（log service / sink / logger），以及一条日志从 `LogErr` 到落盘的完整路径。

## 目录

- [概述](#概述)
- [三层架构](#三层架构)
- [log_line：key/value 列表](#log_linekeyvalue-列表)
- [log_vmessage → log_line_submit](#log_vmessage--log_line_submit)
- [五个 sink](#五个-sink)
- [LogEvent：RAII 的日志事件](#logeventraii-的日志事件)
- [LogErr 宏填了什么](#logerr-宏填了什么)
- [★ `log_sink_json` 字段全集](#-log_sink_json-字段全集)
- [filter：verbosity 与 dragnet](#filterverbosity-与-dragnet)
- [★ InnoDB 侧 `ib::` 宏族](#-innodb-侧-ib-宏族)
- [★ `innodb_print_ddl_logs` 的陷阱](#-innodb_print_ddl_logs-的陷阱)
- [`error_log` 表与 5MB 环形缓冲](#error_log-表与-5mb-环形缓冲)
- [代价与轮转](#代价与轮转)
- [★ 错误日志暴涨的源码级根因](#-错误日志暴涨的源码级根因)

---

## 概述

8.0 之前，错误日志就是 `my_error` + 直接写文件。8.0 引入 **log service（组件服务）** 后，日志变成可插拔：

- **log service**（`log_builtins`）：统一接口，屏蔽"sink 怎么实现"
- **sink 组件**（`log_sink_*.cc`）：实际落盘/落表的具体实现，可动态组合
- **logger**：老的 `log.cc` 里的 `LogErr` / `sql_print_*`，是对 log service 的封装

核心变量 `log_error_services` 决定"日志流经哪些 sink"。

## 三层架构

```
业务代码
  ├─ LogErr(ERROR_LEVEL, ER_xxx, ...)         ← 宏，填充错误码/参数
  ├─ sql_print_error / sql_print_warning      ← 字符串直接输出
  └─ my_error → my_message_local → ...
         │
         ▼
logger（log.cc）：log_vmessage / log_message
  ├─ 组装 log_line（key/value）
  └─ log_line_submit
         │
         ▼
log service（log_builtins）：log_service_chistics（特征位）
  ├─ 按 log_error_services 顺序，逐个 sink 处理
  └─ 每个 sink 的 run() 消费 log_line
         │
         ▼
sink 组件（server_component/log_sink_*.cc）
  ├─ log_sink_trad        传统文件（error.log）
  ├─ log_sink_json        JSON 格式文件
  ├─ log_sink_syseventlog 系统日志（syslog / Windows event log）
  ├─ log_sink_perfschema  写 performance_schema.error_log 表
  └─ log_sink_buffer      内存环形缓冲（供 log_error_services 缓冲）
```

## log_line：key/value 列表

日志在框架内以 **`log_line`** 形式流转——一个 key/value 对的列表，而非格式化的字符串。格式化（"2024-01-01T00:00:00 0 [ERROR] [MY-xxxx] ..."）延迟到 **sink 层**才做。

这样设计的好处：**同一个 log_line 可以被不同 sink 以不同格式消费**（trad 写文本、json 写 JSON、perfschema 拆列存表），格式是 sink 的职责，不是 logger 的职责。

## log_vmessage → log_line_submit

> `sql/log.cc`。

```cpp
int log_vmessage(int log_type, va_list fili) {   // 2046
  // 遍历 varargs，按 key/type/value 填充 log_line
  ...
  return log_line_submit(&ll);                    // 2275
}
```

`log_vmessage` 负责把 varargs 解析成 `log_line`（错误码、消息、时间戳、错误级别等 key），然后交给 `log_line_submit` 提交。

`log_builtins_init()`（1874）在启动时初始化 log service，`log_builtins_exit()`（1940）在退出时清理——它们是 log service 的生命周期入口。

## 五个 sink

| sink | 组件文件 | 落盘形式 |
|---|---|---|
| **internal**（服务名 `log_sink_internal`） | `sql/server_component/`——**服务器内置实现，不是可加载组件** | 传统文本 `error.log`（`[MY-xxxx] [LEVEL] message`） |
| **json** | `log_sink_json.cc` | JSON 行 |
| **syseventlog** | `log_sink_syseventlog.cc` | 系统日志（Linux syslog / Windows EventLog） |
| **perfschema** | `log_sink_perfschema.cc` | `performance_schema.error_log` 表 |
| **buffer** | `log_sink_buffer.cc` | 内存环形缓冲 |

★ `log_sink_buffer` 特殊：它是**内存 sink**，不落盘，用于"缓冲 + 延迟处理"。`log_sink_buffer_flush(LOG_BUFFER_DISCARD_ONLY)` / `LOG_BUFFER_PROCESS_AND_DISCARD` 两个 flush 模式（1858/1862）——前者丢弃缓冲，后者处理（转发给后续 sink）后丢弃。

`log_error_services` 变量把这些 sink **串联**成流水线：例如 `log_filter_internal; log_sink_internal; log_sink_json` 表示"过滤 → 写传统日志 → 写 JSON"。

## LogEvent：RAII 的日志事件

`log.cc` 里有多个 `LogEvent()`（223/587/622/2292/2359 行附近），是 **RAII 的日志事件对象**——构造时收集上下文（线程 id、时间、错误级别），析构时提交。这让"临时构造一条日志"变成作用域内的自动行为，避免忘记 flush。

## 完整调用栈

```
LogErr(ERROR_LEVEL, ER_xxx, ...)         // 宏，展开为错误码 + 参数
  → log_error / log_message
       → log_vmessage(log_type, args)    // log.cc
            → 解析 varargs → 填充 log_line
            → log_line_submit(&ll)       // log.cc
                 → log_service 分发
                      → 按 log_error_services 顺序，逐个 sink->run()
                           → log_sink_trad / json / perfschema / ...
```

## LogErr 宏填了什么

`include/mysql/components/services/log_builtins.h`：

```c
#define LogErr(severity, ecode, ...) \
  LogEvent()                         \
      .prio(severity)                \
      .errcode(ecode)                \
      .subsys(LOG_SUBSYSTEM_TAG)     \
      .component(LOG_COMPONENT_TAG)  \
      .source_line(__LINE__)         \
      .source_file(MY_BASENAME)      \
      .function(__FUNCTION__)        \
      .lookup(ecode, ##__VA_ARGS__)
```

（组件形态下 `.component(LOG_COMPONENT_TAG)` + `.message(...)`；`LogPluginErr` 变体用 `"plugin:" LOG_COMPONENT_TAG`。`DISABLE_ERROR_LOGGING` 时退化为 `dummy_log_message()`。）

消息模板来自 `share/messages_to_error_log.txt`（386 KB，即 `MY-xxxxxx` 的注册表；`.lookup(ecode, ...)` 从这里取模板）。

## ★ `log_sink_json` 字段全集

### 输出机制

JSON sink **不认固定字段表**——它遍历 `log_line` 里的每个 item，按 `item_class` 输出：

```cpp
case LOG_LEX_STRING: ... "%s\"%s\"%s\"%.*s\""   /* JSON 转义：\ " 和 \x00..\x1f (RfC7159) */
case LOG_INTEGER:    ... "%s\"%s\"%s%lld"
case LOG_FLOAT:      ... "%s\"%s\"%s%.12lf"
```

- **字段顺序 = `log_line` 中 item 的顺序**（由 `LogErr` 宏的填充顺序决定），**源码里没有固定的字段顺序表**
- 缺 `label` 时自动补：`label_from_prio(level)`
- 三种排版：`JSON_NOSPACE` / `JSON_PAD`（**默认**）/ `JSON_MULTILINE`
- 文件扩展名 `#define LOG_EXT ".json"`

**官方示例**（源码文件头注释原文）：

```
{ "prio" : 2, "err_code" : 3581, "msg" : "Parser saw: SET @@global.log_error_filter_rules= DEFAULT",
  "err_symbol" : "ER_PARSER_TRACE", "label" : "Note" }

{ "prio" : 2, "err_code" : 3581, "subsystem" : "parser", "SQL_state" : "XX999",
  "source_file" : "sql_parse", "function" : "dispatch_command",
  "msg" : "Parser saw: SELECT \"logging as traditional MySQL error log and as JSON\"",
  "time" : "1970-01-01T00:00:00.000000Z", "thread" : 0,
  "err_symbol" : "ER_PARSER_TRACE", "label" : "Note" }
```

### 完整字段清单（well-known key 表，来自 `sql/server_component/log_builtins.cc`）

| JSON key | 存储类 | `LOG_ITEM_*` |
|---|---|---|
| `log_type` | INTEGER | `LOG_ITEM_LOG_TYPE` |
| `err_code` | INTEGER | `LOG_ITEM_SQL_ERRCODE` |
| `err_symbol` | CSTRING | `LOG_ITEM_SQL_ERRSYMBOL` |
| `SQL_state` | CSTRING | `LOG_ITEM_SQL_STATE` |
| `OS_errno` | INTEGER | `LOG_ITEM_SYS_ERRNO` |
| `OS_errmsg` | CSTRING | `LOG_ITEM_SYS_STRERROR` |
| `source_file` | CSTRING | `LOG_ITEM_SRC_FILE` |
| `source_line` | INTEGER | `LOG_ITEM_SRC_LINE` |
| `function` | CSTRING | `LOG_ITEM_SRC_FUNC` |
| `subsystem` | CSTRING | `LOG_ITEM_SRV_SUBSYS` |
| `component` | CSTRING | `LOG_ITEM_SRV_COMPONENT` |
| `user` | LEX_STRING | `LOG_ITEM_MSC_USER` |
| `host` | LEX_STRING | `LOG_ITEM_MSC_HOST` |
| `thread` | INTEGER | `LOG_ITEM_SRV_THREAD` |
| `query_id` | INTEGER | `LOG_ITEM_SQL_QUERY_ID` |
| `table` | CSTRING | `LOG_ITEM_SQL_TABLE_NAME` |
| `prio` | INTEGER | `LOG_ITEM_LOG_PRIO` |
| `label` | CSTRING | `LOG_ITEM_LOG_LABEL` |
| `verbatim` | CSTRING | `LOG_ITEM_LOG_VERBATIM` |
| `msg` | CSTRING | `LOG_ITEM_LOG_MESSAGE` |
| `msg_id` | INTEGER | `LOG_ITEM_LOG_LOOKUP` |
| `time` | CSTRING | `LOG_ITEM_LOG_TIMESTAMP` |
| `ts` | INTEGER | `LOG_ITEM_LOG_TS` |
| `buffered` | INTEGER | `LOG_ITEM_LOG_BUFFERED` |
| `and_n_more` | INTEGER | `LOG_ITEM_LOG_SUPPRESSED` |
| `stream_id`（仅 json sink） | string | 多流复用同一文件时由 sink 追加 |

⇒ **JSON 里的 key 名就是这张表的第一个字段**。

## filter：verbosity 与 dragnet

### `log_filter_internal` 的三条硬编码规则

```cpp
// failsafe: simple built-in filter may not drop "System" or "Error" messages
r->match ... LOG_ITEM_LOG_PRIO ... ->data_integer = WARNING_LEVEL;
r->cond = LOG_FILTER_COND_LT;
r->verb = LOG_FILTER_RETURN;      // prio < WARNING(2) → 直接放行（System=0, Error=1 永不丢）

// sys_var: log_error_verbosity
r->match ... LOG_ITEM_LOG_PRIO ... ->data_integer = log_error_verbosity;
r->cond = LOG_FILTER_COND_GT;
r->verb = LOG_FILTER_DROP;        // prio > log_error_verbosity → 丢弃

// example: remove all source-line log items
// these are not desirable by default, only while debugging.
```

第三条：**默认 unset `source_line`**（注释明确 "only while debugging"）⇒ 正常日志里看不到 `source_line`。

⚠️ 源码注释还强调：**`log_filter_internal` 与 `log_filter_dragnet` 同时只能用一个**。

### ★ `log_error_verbosity`（默认 2）

```cpp
static Sys_var_ulong Sys_log_error_verbosity(
    "log_error_verbosity",
    "How detailed the error log should be. "
    "1, log errors only. 2, log errors and warnings. "
    "3, log errors, warnings, and notes. ...",
    PERSIST_AS_READONLY GLOBAL_VAR(log_error_verbosity), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1, 3), DEFAULT(2), BLOCK_SIZE(1), ...);
```

级别枚举（`include/my_loglevel.h`）：

```c
enum loglevel {
  SYSTEM_LEVEL = 0, ERROR_LEVEL = 1, WARNING_LEVEL = 2,
  INFORMATION_LEVEL = 3, MORPHY_INFORMATION_LEVEL = 4   /* 本 fork 专有 */
};
```

过滤规则 `prio > verbosity → DROP` ⇒

| verbosity | 保留 |
|---|---|
| 1 | System(0) + Error(1) |
| **2（默认）** | + Warning(2)，**Note/Information(3) 全丢** |
| 3 | + Information(3) |

### dragnet 规则语法

```c
#define LOG_FILTER_LANGUAGE_NAME "dragnet"
#define LOG_FILTER_SYSVAR_NAME "log_error_filter_rules"
#define LOG_FILTER_DEFAULT_RULES \
  "IF prio>=INFORMATION THEN drop. IF EXISTS source_line THEN unset source_line."
```

- 变量名 **`log_error_filter_rules`**（属性 `PLUGIN_VAR_PERSIST_AS_READ_ONLY`）
- 语法例子：
```
IF prio>=3 THEN drop.
IF err_code==1408 THEN set label:="HELO".
IF EXISTS source_line THEN unset source_line.
```
- token 分类：条件（`LOG_FILTER_XLATE_COND`）、动作（`drop`/`unset`/`set`）、流程（`IF`/`THEN`）、连接（`or`/`and`）、前缀（`EXISTS`）、字面量（`INFORMATION`）
- 配置期会做**类型检查**（注释：检查字段是否已知、存储类是否匹配，不匹配就报错）
- 规则以 `.` 结尾；状态查看 `dragnet.Status`

## ★ InnoDB 侧 `ib::` 宏族

定义 `storage/innobase/include/ut0log.h`（`namespace ib`），实现 `storage/innobase/ut/ut0ut.cc`。

```cpp
class logger { ... std::ostringstream m_oss; int m_err; loglevel m_level; };
class info  : public logger { info()  : logger(INFORMATION_LEVEL) {} };
class warn  : public logger { warn()  : logger(WARNING_LEVEL)     {} };
class error : public logger { error() : logger(ERROR_LEVEL)       {} };
class fatal : public logger { fatal(ut::Location location) : logger(ERROR_LEVEL), ... };
```

**落盘路径（关键证据）**：

```cpp
void logger::log_event(std::string msg) {
  LogEvent()
      .type(LOG_TYPE_ERROR)
      .prio(m_level)
      .errcode(m_err)
      .subsys("InnoDB")
      .verbatim(msg.c_str());
}
logger::~logger() { log_event(m_oss.str()); }        // 消息在析构时才发出
```

⇒ **`ib::` 宏族最终就是 `LogEvent`**，完全受 `log_error_verbosity` / filter / sink 管辖；`subsystem` 恒为 `"InnoDB"`。

### ★ `fatal` 会 abort 吗？会

```cpp
fatal::~fatal() {
  log_event("[FATAL] " + m_oss.str());
  ut_dbg_assertion_failed("ib::fatal triggered", m_location.filename, m_location.line);
}
```

`ut_dbg_assertion_failed` 声明为 **`[[noreturn]]`**（`ut0dbg.h`），实现先打 `"Assertion failure: ..."`、再 `flush_error_log_messages()`、最终 abort。

（`ut_error` 宏就是 `ut_dbg_assertion_failed(0, __FILE__, __LINE__)`。）

| 宏 | `loglevel` | 默认 verbosity=2 下 |
|---|---|---|
| `ib::info` | INFORMATION(3) | ❌ **被 DROP（看不到）** |
| `ib::warn` | WARNING(2) | ✅ |
| `ib::error` | ERROR(1) | ✅ |
| `ib::fatal` | ERROR(1)，消息前缀 `[FATAL] ` | ✅，随后 abort |
| `ib::morphy_info`（fork 专有） | 4 | ❌ **永远被 DROP** |

## ★ `innodb_print_ddl_logs` 的陷阱

**存在**，是 InnoDB 插件变量（`storage/innobase/handler/ha_innodb.cc`）：

```cpp
static MYSQL_SYSVAR_BOOL(
    print_ddl_logs, srv_print_ddl_logs, PLUGIN_VAR_OPCMDARG,
    "Print all DDl logs to MySQL error log (off by default)", nullptr, nullptr, false);
```

底层：`srv0srv.cc` 的 `bool srv_print_ddl_logs = false;`（注释 `/** Print all DDL logs to mysqld stderr */`）

`log0ddl.cc` 里所有打印点（**全为 `ib::info`**）：

| 类别 | 错误码 | 输出 |
|---|---|---|
| insert | `ER_IB_MSG_647`~`652`、`1284` | `"DDL log insert : " << record` |
| delete | `ER_IB_MSG_DDL_LOG_DELETE_BY_ID_OK` | `"DDL log delete : " << id` |
| replay | `ER_IB_MSG_654` | `"DDL log replay : " << record` |
| replay | `ER_IB_MSG_655` | `"... FREE tablespace <space_id> is missing."` |
| replay | `ER_IB_MSG_656` | `"... RENAME from <old> to <new> failed"` |
| replay | `ER_IB_MSG_657` | `"... in recovery, skip RENAME TABLE"` |
| replay | `ER_IB_MSG_659` | `"... in recovery, skip REMOVE CACHE"` |
| post ddl | `ER_IB_MSG_660` / `661` | `"DDL log post ddl : begin/end for thread id : <tid>"` |

### ★★ 实操陷阱

这些点**全是 `ib::info` → INFORMATION_LEVEL(3)**，而 `log_error_verbosity` **默认 2**，过滤规则 `prio > verbosity → DROP`：

⇒ **只设 `innodb_print_ddl_logs=ON` 看不到任何输出，必须同时设 `log_error_verbosity=3`**。

```sql
SET GLOBAL log_error_verbosity = 3;          -- ★ 必须
SET GLOBAL innodb_print_ddl_logs = ON;
```

## `error_log` 表与 5MB 环形缓冲

表定义（`storage/perfschema/table_error_log.cc`）：

```cpp
    "  LOGGED TIMESTAMP(6) NOT NULL,\n"
    "  THREAD_ID BIGINT UNSIGNED,\n"
    "  PRIO ENUM ('System','Error','Warning','Note') NOT NULL,\n"
    "  ERROR_CODE VARCHAR(10),\n"
    "  SUBSYSTEM VARCHAR(7),\n"
    "  DATA TEXT NOT NULL,\n"
    "  PRIMARY KEY (LOGGED) USING HASH,\n"
    "  KEY (THREAD_ID) USING HASH, KEY (PRIO) USING HASH,\n"
    "  KEY (ERROR_CODE) USING HASH, KEY (SUBSYSTEM) USING HASH\n",
```

- **只读**（`&pfs_readonly_acl`，`write_row`/`delete_all_rows` 均为 nullptr）
- ⚠️ `SUBSYSTEM VARCHAR(7)` —— 只够 `"InnoDB"`(6) / `"Server"`(6)，**长 subsystem 会截断**
- `PRIO` 映射（`table_error_log.h`）：`SYSTEM=1 / ERROR=2 / WARNING=3 / NOTE=4`（注释：与 `my_loglevel.h` 有固定对应关系，不得改）

**后端是 5MB 环形缓冲**（`sql/server_component/log_sink_perfschema.cc`）：

```cpp
/*
  In the interest of not adding more settings to confuse the using,
  the error-log ring-buffer is of a static size for now. ...
  That lead us to expect a yield of 4-6 events per KB,
  and thus about 25,000 for a buffer of 5 MB.
*/
static const size_t ring_buffer_size = 5 * 1024 * 1024;
```

- 状态：`log_sink_pfs_buffered_bytes` / `_buffered_events` / `_expired_events`
- 满了覆盖最老的（`expired_events` 递增）
- 启动时可**回读**上次 error log 文件灌进表（`log_sink_trad_parse_log_line()` 或 json sink 的 `parse_log_line`）

⇒ **该表最多约 2.5 万条**，超出会丢最老的——不能当长期审计存储。

## 代价与轮转

### ★ 写入开销：基本没有缓冲

```cpp
void log_write_errstream(const char *buffer, size_t length) {
  if (error_log_initialized) mysql_mutex_lock(&LOCK_error_log);
  fprintf(stderr, "%.*s\n", (int)length, buffer);
  fflush(stderr);
  if (error_log_initialized) mysql_mutex_unlock(&LOCK_error_log);
}
```

代价构成：

1. **每条日志一次 `LOCK_error_log` 全局互斥**（所有线程串行化）
2. **一次 `fflush(stderr)`** —— 强制 syscall，最大的一块
3. filter 遍历规则集 + sink 里 `line_item_iter_*` 迭代 + `substitute()` 格式化（JSON sink 还要逐字符 JSON 转义）
4. 若 `log_sink_perfschema` 在管道里，还要写环形缓冲

⚠️ **唯一的"缓冲"是 `log_sink_buffer`**，但它用于**启动早期组件未就绪**时暂存（`discard_error_log_messages()` / `flush_error_log_messages()`），**不是性能缓冲**。

### 轮转：`FLUSH ERROR LOGS`

- 标志 `REFRESH_ERROR_LOG 256`（`include/mysql_com.h`）
- `sql/sql_reload.cc` → `reopen_error_log()`：
```cpp
bool reopen_error_log() {
  // call flush function in all logging services
  if ((component_failures = log_builtins_error_stack_flush()) < 0) { ... }
```

⇒ 轮转 = **让管道里每个 sink 执行自己的 flush/重开文件**。裸 `FLUSH LOGS` 会带上所有 `REFRESH_*`。

### `log_error_suppression_list`（8.0 **有**）

```cpp
static Sys_var_charptr Sys_log_error_suppression_list(
    "log_error_suppression_list",
    "Comma-separated list of error-codes. Error messages corresponding to "
    "these codes will not be included in the error log. Only events with a "
    "severity of Warning or Information can be suppressed; events with "
    "System or Error severity will always be included. ...",
    PERSIST_AS_READONLY GLOBAL_VAR(opt_log_error_suppression_list), ... DEFAULT(""), ...);
```

- 默认空；**只能抑制 Warning/Information**（System/Error 永远保留，与 failsafe 规则一致）
- 规则槽位有上限：注释——"Reserve one rule for `--log-error-verbosity` and one for our 'ERROR and SYSTEM always pass' shortcut"
- 需要管道里有 `log_filter_internal`

## ★ 错误日志暴涨的源码级根因

### ① InnoDB semaphore wait（最经典的刷屏源）

- 后台线程 `srv_error_monitor_thread()`（`srv0srv.cc`），注释："A thread which prints warnings about semaphore waits which have lasted too long."，循环体注释确认频率 **~1 Hz**
- 每次调 `sync_array_print_long_waits()`（`sync0arr.cc`），对超时 cell：
```cpp
  ib::warn(ER_IB_MSG_1160) << "A long semaphore wait:";
  sync_array_cell_print(stderr, cell);
```
- 升级致命：连续 10 次同一 waiter/sema →
```cpp
  if (fatal_cnt > 10) {
    ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_1047,
              ulonglong{srv_fatal_semaphore_wait_threshold});
  }
```
消息文本：`"Semaphore wait has lasted > %llu seconds. We intentionally crash the server because it appears to be hung."`
阈值 `srv_fatal_semaphore_wait_threshold = 600`（即 `innodb_fatal_semaphore_wait_threshold`，600 秒）

⇒ **每秒一条 Warning + 一大段 sync array dump**，几分钟就能刷出 GB 级日志。**这是"错误日志暴涨"最常见的根因。**

### ② 有内置节流的路径（`Error_log_throttle`）

`Log_throttle::LOG_THROTTLE_WINDOW_SIZE = 60000000`（**60 秒**），`Error_log_throttle` 每窗口只放行 **1 条**，超出在窗口结束打一条汇总 `"Error log throttle: %lu time(s) ..."`。

现有实例：
- `connection_handler_per_thread.cc`：`create_thd_err_log_throttle`（创建线程失败）
- `log_event.cc`：`slave_ignored_err_throttle`（`ER_SERVER_REPLICA_IGNORED_TABLE`）

⇒ **有 `Error_log_throttle` 的路径不会暴涨；没有的（如 semaphore wait）才会**。

### ③ 磁盘满 / 文件不可写

`open_error_log()` 失败 → `LogErr(ERROR_LEVEL, ER_CANT_OPEN_ERROR_LOG, ...)`。**磁盘满时写日志失败还会再产生一条错误日志**；若慢日志也开着，每条慢查询失败一次 ⇒ 同步放大错误日志量。

### ④ `log_error_verbosity=3` 本身

把 `INFORMATION_LEVEL`(3) 全量放行（包括所有 `ib::info`：DDL log、clone GTID、lock-free hash 统计等）⇒ 量级跳一个数量级。

## 可观测性

```sql
-- ① 当前管道（默认只有两个环节）
SHOW VARIABLES LIKE 'log_error_services';
-- 默认: log_filter_internal; log_sink_internal

-- ② 读 PFS 里的错误日志（5MB 环形缓冲，约 2.5 万条上限）
SELECT * FROM performance_schema.error_log ORDER BY LOGGED DESC LIMIT 50;
-- JSON sink 模式下取 msg：
SELECT RIGHT(logged,15), prio, error_code, subsystem,
       IF(LEFT(data,1)='{', JSON_EXTRACT(data,'$.msg'), data) AS msg
  FROM performance_schema.error_log;

-- ③ 改管道（PERSIST_AS_READONLY，需 SET PERSIST_ONLY 才能持久化）
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_internal; log_sink_json';
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_internal; log_sink_perfschema';

-- ④ ★ 想看 InnoDB 的 ib::info（含 DDL log）必须两个都设
SET GLOBAL log_error_verbosity = 3;
SET GLOBAL innodb_print_ddl_logs = ON;

-- ⑤ 抑制已知噪音错误码
SET GLOBAL log_error_suppression_list = 'MY-010035,MY-013360';

-- ⑥ 轮转
FLUSH ERROR LOGS;
```

### 观测对象 → 手段 速查

| 我想看 | 入口 |
|---|---|
| 当前日志管道 | `log_error_services` |
| 最近错误（SQL 可读） | `performance_schema.error_log` |
| **结构化（要 msg/err_symbol/源码位置）** | 开启 `log_sink_json` |
| InnoDB DDL log 明细 | `innodb_print_ddl_logs=ON` + **`log_error_verbosity=3`** |
| 只关心 Warning 以上 | `log_error_verbosity=2`（默认） |
| 精细过滤 | `log_filter_dragnet` + `log_error_filter_rules` |
| 抑制已知噪音 | `log_error_suppression_list` |
| 慢查询原文 | 见 [`slow_log.md`](slow_log.md) |

## 参考

- *MySQL 8.0 Reference Manual*: "The Error Log" / "Error Log Components" / "Error Log Message Format"
- 源码：
  - 服务名宏与默认管道 `sql/server_component/log_builtins_imp.h`（`LOG_BUILTINS_FILTER` / `LOG_BUILTINS_SINK` / `LOG_ERROR_SERVICES_DEFAULT`）
  - well-known key 表 `sql/server_component/log_builtins.cc`（= JSON 字段名）
  - sink：`sql/server_component/log_sink_trad.cc`（**服务名 `log_sink_internal`，函数名 `log_sink_trad`**）、`log_sink_perfschema.cc`、`log_sink_buffer.cc`；`components/logging/log_sink_json.cc`、`log_sink_syseventlog.cc`、`log_sink_test.cc`
  - filter：`sql/server_component/log_builtins_filter.cc`（三条规则 + verbosity + suppression list）；`components/logging/log_filter_dragnet.cc`
  - 宏：`include/mysql/components/services/log_builtins.h`（`LogErr`）、`include/my_loglevel.h`（loglevel 枚举）
  - 变量：`sql/sys_vars.cc`（`log_error_verbosity` / `log_error_services` / `log_error_suppression_list`）
  - InnoDB 侧：`storage/innobase/include/ut0log.h` + `ut/ut0ut.cc`（`ib::` 宏族）、`storage/innobase/log/log0ddl.cc` + `handler/ha_innodb.cc`（`innodb_print_ddl_logs`）、`storage/innobase/srv/srv0srv.cc` + `sync/sync0arr.cc`（semaphore wait 刷屏源）
  - PFS 表：`storage/perfschema/table_error_log.cc` / `.h`
  - 消息注册表：`share/messages_to_error_log.txt`（`MY-xxxxxx`）
- 相关文档：[`slow_log.md`](slow_log.md)（慢查询日志）、`general_log.md`（通用查询日志）、[`../README.md`](../README.md)（观测手段全景与决策树）
