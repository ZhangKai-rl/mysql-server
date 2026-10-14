# MySQL 通用查询日志（general log）深度解析

> **边界**：本篇讲**通用查询日志**这一观测手段——它是什么、记什么、怎么写出去、代价多大。
> - 慢查询日志（按阈值过滤）见 [`slow_log.md`](slow_log.md)；错误日志见 [`error_log.md`](error_log.md)
> - 两者共用 `Query_logger` 与 handler 框架，本篇会指出共用处，不重复讲

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

**记录 server 收到的每一条命令**（不只是 SQL）——包括连接、断开、`COM_PING`、出错的语句等。它是 MySQL 里**最全**的一条语句级审计轨迹。

### 用途

| 场景 | 说明 |
|---|---|
| 排查"到底执行了什么" | 慢日志只记超时语句，general log 记全部 |
| 定位客户端行为 | 连上了没、发了什么、什么时候断的 |
| 短时审计 | 合规排查时短开一会儿 |

⚠️ **它不是为生产常开设计的**（见理论基础）。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.1 | 引入 `log_output` 支持 TABLE（写 CSV 表 `mysql.general_log`） |
| 5.6+ | 与 slow log 统一到 `Query_logger` + `Log_event_handler` 框架 |
| 8.0 | 支持 `log_slow_admin_statements` 等细粒度开关；表实现改为 CSV（非 InnoDB） |

---

## 理论基础

### 设计思想与权衡

#### ① 为什么默认关闭，且官方不建议生产常开

`general_log` 记的是**每一条命令**，代价是：

- **每条语句一次写**（文件 I/O 或表插入），在高 QPS 下直接成为瓶颈
- 写**表**时还要走一遍表操作（打开表、插行、锁），比写文件更贵
- 体积增长极快（长连接 + 高频应用，一天可达几十 GB）

⇒ **官方给的定位是"短时排障工具"**，不是监控手段。

#### ② 为什么"文件 vs 表"是两种不同的 handler 而不是两种格式

`log_output` 有 `FILE` / `TABLE` / `NONE` 三种，实现上是**一条 handler 链**（`general_log_handler_list`），可以**同时挂多个**：

```
Query_logger::general_log_write()
     └─> 遍历 general_log_handler_list
            ├── Log_to_file_event_handler::log_general()   → 写文件
            └── Log_to_csv_event_handler::log_general()    → 写 mysql.general_log 表
```

⇒ 这样设计的好处：`log_output = 'FILE,TABLE'` 能同时写两处，加新输出方式只需加一个 handler。

#### ③ ★ 为什么写日志表要临时关掉 binlog

源码里这一行是必看的（见「核心实现 → 三」）：

```cpp
  thd->variables.option_bits &= ~OPTION_BIN_LOG;
```

⇒ 日志表是**本地元数据**，不是业务数据。如果写 `mysql.general_log` 也记 binlog，会把日志复制给所有从库（每台机器的日志不同，没意义且放大流量），并污染 GTID。所以写之前临时清 `OPTION_BIN_LOG`，写完恢复。这与安装 sys schema 时 `SET sql_log_bin = 0` 是**同一个道理**。

#### ④ 与 slow log 的关系：共用框架，而非互相包含

| | general log | slow log |
|---|---|---|
| 记什么 | **所有命令** | 超过 `long_query_time` 的语句（+ 可选管理语句） |
| 粒度 | 命令级（`enum_server_command`） | 语句级，带详细计时/行数 |
| 默认 | OFF | OFF |
| 共用 | `Query_logger`、`Log_event_handler` 链、`LOCK_logger` | 同 |

---

## 核心实现

### 主链路

```
客户端发命令
  └─> dispatch_command()
        └─> Query_logger::general_log_print(thd, command, ...)    ← 大多数调用点
              ├─ 格式化消息（vsnprintf）
              └─> Query_logger::general_log_write(thd, command, query, len)
                    ├─ mysql_audit_general_log()          ← 先发 audit API
                    ├─ 三重判断：log_command() && opt_general_log && handler 链非空
                    ├─ make_user_name() 拼 user@host
                    ├─ mysql_rwlock_rdlock(&LOCK_logger)
                    └─ 遍历 handler 链逐个 log_general()
```

### 一、两个入口：`general_log_print()` 与 `general_log_write()`

#### `general_log_print()` —— 带格式化的入口（多数调用点用它）

贴**完整函数**，在关键行附近就近注释剖析（★ 只注释核心与难点，不逐行）：

```cpp
bool Query_logger::general_log_print(THD *thd, enum_server_command command,
                                     const char *format, ...) {
  // ★ 短路在最前面：这类命令要不要记 / 总开关 / handler 链是否非空，三者缺一即走这里
  //   ⇒ general_log=OFF 时开销只是一次布尔判断（audit 无插件时也是空转）——零开销探针思路在日志侧的落地
  if (!log_command(thd, command) || !opt_general_log ||
      !(*general_log_handler_list)) {
    /* Send a general log message to the audit API. */
    const std::string &cn = Command_names::str_global(command);
    mysql_audit_general_log(thd, cn.c_str(), cn.length());
    return false;   // audit 只在"不写日志"这条分支里发
  }

  size_t message_buff_len = 0;
  char message_buff[LOG_BUFF_MAX];   // 定长栈缓冲 ⇒ 消息超长会被截断

  /* prepare message */
  if (format) {
    va_list args;
    va_start(args, format);
    message_buff_len =
        vsnprintf(message_buff, sizeof(message_buff), format, args);
    va_end(args);

    // 截断到缓冲上限 ⇒ 超长 SQL 在 general log 里可能不完整（排查时易误判成"语句没执行完"）
    message_buff_len = std::min(message_buff_len, sizeof(message_buff) - 1);
  } else
    message_buff[0] = '\0';

  return general_log_write(thd, command, message_buff, message_buff_len);
}
```

#### `general_log_write()` —— 真正的写入

```cpp
bool Query_logger::general_log_write(THD *thd, enum_server_command command,
                                     const char *query, size_t query_length) {
  // ★ 与 general_log_print() 相反：这里无条件先发 audit，再判断写不写
  //   ⇒ 直接调本函数的路径会比走 general_log_print() 多一次 audit 调用（源码里一处易看漏的不对称）
  /* Send a general log message to the audit API. */
  const std::string &cn = Command_names::str_global(command);
  mysql_audit_general_log(thd, cn.c_str(), cn.length());

  /*
    Do we want to log this kind of command?
    Is general log enabled?
    Any active handlers?
  */
  if (!log_command(thd, command) || !opt_general_log ||
      !(*general_log_handler_list))
    return false;

  // 只拼一次 user@host、只取一次时间，再传给所有 handler
  // ⇒ 避免重复计算，也保证同一条命令写到文件和表里的 event_time 一致
  char user_host_buff[MAX_USER_HOST_SIZE + 1];
  size_t user_host_len =
      make_user_name(thd->security_context(), user_host_buff);
  ulonglong current_utime = my_micro_time();

  // 读锁：多个会话可并发写日志（真正的串行化由底层文件/表的写锁保证）
  //       LOCK_logger 只保护"handler 链本身"不被并发修改（改 log_output 时才需要写锁）
  mysql_rwlock_rdlock(&LOCK_logger);

  bool error = false;
  for (Log_event_handler **current_handler = general_log_handler_list;
       *current_handler;) {
    // error |= 用 OR 累积各 handler 的错误 ⇒ 一个 handler 写失败不会中断其它 handler
    error |=
        (*current_handler++)
            ->log_general(thd, current_utime, user_host_buff, user_host_len,
                          thd->thread_id(), cn.c_str(), cn.length(), query,
                          query_length, thd->variables.character_set_client);
  }
  mysql_rwlock_unlock(&LOCK_logger);

  return error;
}
```

### 二、文件输出 handler

```cpp
  if (!mysql_general_log.is_open()) return false;   // 文件没打开直接返回，不抛错——记日志不该影响业务

  // ★ 临时抑制错误报告：把 Silence_log_table_errors 挂到会话上，写完 pop 掉
  //   ⇒ 磁盘满、文件被删等情况下用户不会收到错误，业务语句照常执行
  //     这是"观测不能干扰被观测对象"的直接体现
  Silence_log_table_errors error_handler;
  thd->push_internal_handler(&error_handler);
  bool retval =
      mysql_general_log.write_general(event_utime, thread_id, command_type,
                                      command_type_len, sql_text, sql_text_len);
  thd->pop_internal_handler();
  return retval;
```

### 三、表输出 handler（★ 关 binlog 的那段在这）

```cpp
  TABLE *table = nullptr;
  bool result = true;
  bool need_close = false;
  bool need_rnd_end = false;
  uint field_index;
  my_timeval tv;

  // CSV 引擎在"修复表"时会调 TIME_to_timestamp()，从而置位 thd->time_zone_used
  // ⇒ 记日志污染了会话状态，必须保存还原（观测代码不能改业务会话的状态）
  /*
    CSV uses TIME_to_timestamp() internally if table needs to be repaired
    which will set thd->time_zone_used
  */
  bool save_time_zone_used = thd->time_zone_used;

  // ★ 清掉 OPTION_BIN_LOG ⇒ 写 mysql.general_log 不记 binlog（理由见理论基础 ③）
  //   末尾会恢复 save_thd_options
  ulonglong save_thd_options = thd->variables.option_bits;
  thd->variables.option_bits &= ~OPTION_BIN_LOG;

  // TL_WRITE_CONCURRENT_INSERT = 并发插入锁，允许读的同时追加写
  // ⇒ 与上面的 save_time_zone_used 是一对：观测代码把可能改动的会话状态逐一保存还原
  Table_ref table_list(MYSQL_SCHEMA_NAME.str, MYSQL_SCHEMA_NAME.length,
                       GENERAL_LOG_NAME.str, GENERAL_LOG_NAME.length,
                       GENERAL_LOG_NAME.str, TL_WRITE_CONCURRENT_INSERT);
  ...
```

### 四、调用点在哪

`general_log_print()` 在 `sql/sql_parse.cc` 的 `dispatch_command()` 及各命令分支中被调用；`log_command()` 决定某类命令要不要记（`enum_server_command` 维度，与 slow log 的阈值过滤是两套机制）。

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `general_log` | **OFF** | Global（动态） | 总开关 |
| `log_output` | **`FILE`** | Global（动态） | `FILE` / `TABLE` / `NONE`，可组合 |
| `general_log_file` | `host_name.log` | Global | 文件名（相对 datadir） |
| `log_timestamps` | `UTC` | Global/Session | 日志时间时区 |
| `log_raw` | OFF | Global | 是否不过滤密码（★ 默认会脱敏） |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 当前有没有开 | SQL | `SHOW VARIABLES LIKE 'general_log'` |
| 最近执行了什么 | SQL | `SELECT * FROM mysql.general_log ORDER BY event_time DESC LIMIT 20`（需 `log_output` 含 TABLE） |
| 日志文件在哪 | SQL | `SHOW VARIABLES LIKE 'general_log_file'` |
| 不想开它但想看当前在做什么 | SQL | `sys.session` / `SHOW PROCESSLIST`（见 [`../processlist.md`](../processlist.md)） |

---

## Misc

### 坑与已知缺陷

- **★ 默认会脱敏密码**：含密码的语句（`CREATE USER ... IDENTIFIED BY`、`SET PASSWORD`）在 general log 里被替换成 `<secret>`，除非 `log_raw=ON`。但 `log_raw=ON` 后**明文密码会落盘**——两者取舍需明确。
- **表输出用 CSV 引擎，不支持索引**：`SELECT * FROM mysql.general_log ORDER BY event_time` 是**全表扫**；表大时极慢。
- **表输出会锁**：`TL_WRITE_CONCURRENT_INSERT` 虽允许并发读，但**写与写之间仍串行**，高并发下成为热点。
- **消息超长被截断**（`LOG_BUFF_MAX`）。
- **`log_output` 变更需要 `LOCK_logger` 写锁**，与正在写日志的会话是读-写竞争。
- ★ **两个入口的 audit 调用不对称**（`general_log_print` 不写时才发，`general_log_write` 无条件发）——做审计插件时要注意。

### 易混淆概念

| 此 X | 非彼 X |
|---|---|
| general log | **不是** audit log（MySQL 社区版没有审计日志；general log 只是语句轨迹，无权限/合规语义） |
| `log_output=TABLE` 的 `mysql.general_log` | **不是** InnoDB 表（是 **CSV**），无事务、无索引 |
| `general_log` | 与 `slow_query_log` 是**两条独立的链**（各有自己的 handler list），不是同一个开关 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 5.4.3 The General Query Log*

**源码**
- `sql/log.cc`（`Query_logger::general_log_print` / `general_log_write`、`Log_to_file_event_handler::log_general`、`Log_to_csv_event_handler::log_general`）
- `sql/sql_parse.cc`（`dispatch_command` 里的调用点）

**本目录相关篇**
- [`slow_log.md`](slow_log.md)（慢查询日志；共用 `Query_logger` 框架，过滤机制不同）
- [`error_log.md`](error_log.md)（错误日志；8.0 已改成 sink/filter 组件化，**架构不同**）
- [`../processlist.md`](../processlist.md)（不想开 general log 时的替代：看当前在做什么）
