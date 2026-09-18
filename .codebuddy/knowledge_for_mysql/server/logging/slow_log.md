# MySQL 慢查询日志深度解析

> 基于 MySQL 8.0.39 源码，涵盖慢日志判定条件、记录内容、文件/表两种输出、参数体系。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [慢日志判定条件](#慢日志判定条件)
- [记录内容](#记录内容)
- [输出方式：文件 vs 表](#输出方式文件-vs-表)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

### 用途

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | XXX |
| 5.7 | XXX |
| 8.0 | XXX |

---

## 理论基础

### 设计模式

### 相关论文

### 算法与数据结构

### 类似实现对比

### 历史背景

---

## 慢日志判定条件

`long_query_time`、`log_queries_not_using_indexes`、`log_throttle_queries_not_using_indexes`。

---

## 记录内容

<!-- TODO: 待补 -->

---

## 输出方式：文件 vs 表

`log_output` 参数（FILE / TABLE / NONE），`mysql.slow_log` 表结构。

---

## 核心调用栈

```
mysql_execute_command (sql/sql_parse.cc)
  → Sql_cmd_dml::execute 尾部
    → query_logger / log_slow_statement (sql/log.cc)
      → 判定 long_query_time 超时
        → 写慢日志文件 / 写 mysql.slow_log 表
```

> 必填。调用链补全后再整理。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `long_query_time` | 10 | Global/Session | 超过该秒数的查询记慢日志 |
| `slow_query_log` | OFF | Global | 慢日志总开关 |
| `slow_query_log_file` | hostname-slow.log | Global | 慢日志文件路径 |
| `log_queries_not_using_indexes` | OFF | Global | 未用索引的查询也记 |
| `log_throttle_queries_not_using_indexes` | 0 | Global | 未用索引日志节流 |
| `log_output` | FILE | Global | FILE/TABLE/NONE |
| `min_examined_row_limit` | 0 | Global/Session | 扫描行数下限 |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| `Slow_queries` | 慢查询次数 |

---

## Misc

<!-- TODO: 待补 -->

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/log.cc` | 慢日志写入实现 |
| `sql/mysqld.cc` | `long_query_time` 变量注册 |
| `sql/sql_class.cc` | 慢日志判定（`thd->server_status`） |
