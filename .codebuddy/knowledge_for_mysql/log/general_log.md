# MySQL 通用查询日志深度解析

> 基于 MySQL 8.0.39 源码，涵盖 general log 记录内容、文件/表输出、性能开销与实战用途。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [记录内容](#记录内容)
- [输出方式：文件 vs 表](#输出方式文件-vs-表)
- [性能开销](#性能开销)
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

## 记录内容

所有客户端语句（含连接/断开、执行成功与否）。

---

## 输出方式：文件 vs 表

`log_output` 参数控制 FILE / TABLE / NONE，`mysql.general_log` 表结构。

---

## 性能开销

<!-- TODO: 待补 -->

---

## 核心调用栈

```
dispatch_command (sql/sql_parse.cc)
  → query_logger.general_log_print / general_log_write
    → 写 general log 文件 / mysql.general_log 表
```

> 必填。调用链补全后再整理。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `general_log` | OFF | Global | general log 总开关 |
| `general_log_file` | hostname.log | Global | 日志文件路径 |
| `log_output` | FILE | Global | FILE/TABLE/NONE |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| 无 | XXX |

---

## Misc

<!-- TODO: 待补 -->

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/log.cc` | general log 实现 |
| `sql/sql_parse.cc` | `dispatch_command` 处调用 |
| `sql/log.h` | `query_logger` 类 |
