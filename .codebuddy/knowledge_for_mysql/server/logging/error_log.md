# Error Log：log service / sink 三层架构

> 基于 MySQL 8.0.39。8.0 的错误日志是一次**架构重构**：从"写文件"变成"可插拔的 sink 组件体系"。本篇剖析这套三层架构（log service / sink / logger），以及一条日志从 `LogErr` 到落盘的完整路径。

## 目录

- [概述](#概述)
- [三层架构](#三层架构)
- [log_line：key/value 列表](#log_linekeyvalue-列表)
- [log_vmessage → log_line_submit](#log_vmessage--log_line_submit)
- [五个 sink](#五个-sink)
- [LogEvent：RAII 的日志事件](#logeventraii-的日志事件)

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

## 可观测性

```sql
-- 查看日志 sink 串联
SHOW VARIABLES LIKE 'log_error_services';

-- 读 perfschema 里的错误日志
SELECT * FROM performance_schema.error_log;

-- 修改 sink 组合
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_internal; log_sink_json';
```

## 参考

- MySQL 8.0 Reference Manual, "The Error Log" / "Error Log Components"
- 相关文档：`slow_log.md`（慢查询日志）、`general_log.md`（通用查询日志）
