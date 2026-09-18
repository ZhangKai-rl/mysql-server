# MySQL 错误日志深度解析

> 基于 MySQL 8.0.39 源码，涵盖 8.0 组件化架构（log_error_services）、日志过滤与输出、错误日志格式与定位。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [8.0 组件化架构](#80-组件化架构)
- [日志过滤与输出](#日志过滤与输出)
- [错误日志格式](#错误日志格式)
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
| 5.6 | 单文件 `hostname.err` |
| 5.7 | `log_error` 变量可指定文件/表 |
| 8.0 | 组件化：`log_error_services`（filter + sink 链） |

---

## 理论基础

### 设计模式

### 相关论文

### 算法与数据结构

### 类似实现对比

### 历史背景

---

## 8.0 组件化架构

`log_error_services` 系统变量，`log_filter_*` + `log_sink_*` 插件链，`mysql_error_log` 动态加载。

---

## 日志过滤与输出

<!-- TODO: 待补 -->

---

## 错误日志格式

`[ERROR]` / `[Warning]` / `[Note]` 级别，`log_error_verbosity` 控制。

---

## 核心调用栈

```
log_error (sql/log.cc)
  → error_log_print / log_error_services 分发
    → Log_filter 处理
      → Log_sink 输出（文件/JSON/JSON 表）
```

> 必填。调用链补全后再整理。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `log_error` | stderr | Global | 错误日志目标 |
| `log_error_services` | log_filter_internal; log_sink_internal | Global | 过滤+输出链 |
| `log_error_verbosity` | 2 | Global | 日志级别(1-3) |

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
| `sql/log.cc` | 错误日志实现 |
| `sql/log.h` | `error_log_service` 接口 |
| `components/` | log_sink_* / log_filter_* 组件 |
