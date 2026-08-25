# MySQL DBUG 调试框架深度解析

> 基于 MySQL 8.0.39 源码，涵盖 DBUG 宏体系、trace 输出、调试模式控制、与 InnoDB trace 的关系。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [DBUG 宏体系](#dbug-宏体系)
- [trace 输出与格式](#trace-输出与格式)
- [调试模式控制](#调试模式控制)
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

## DBUG 宏体系

`DBUG_TRACE` / `DBUG_PRINT` / `DBUG_RETURN` / `DBUG_ENTER` 等宏的展开与语义。

---

## trace 输出与格式

<!-- TODO: 待补 -->

---

## 调试模式控制

`MYSQL_DEBUG` 环境变量（`d:t:o` 语法）、`debug` 系统变量、`mysql-test` 的 dbug 输出。

---

## 核心调用栈

```
入口 → DBUG_PRINT (include/mysql/service_debug_sync.h)
  → _db_doprnt (mysys/dbug.cc)
    → Debug::print (mysys/dbug.cc)
      → 输出到 stderr/trace 文件
```

> 必填。调用链补全后再整理。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `debug` | 空 | Session | 控制 DBUG 开关与模式 |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| 无 | XXX |

---

## Misc

### InnoDB 的 trace

- `UNIV_DEBUG` / `ut_d` / `ut_a`：InnoDB 自己的调试断言编译开关，非 DBUG
- `innodb_monitor` / `INNODB_METRICS` 表：InnoDB 运行状态输出
- 注意区分：DBUG trace（函数调用追踪） vs InnoDB 断言（`ut_a`） vs InnoDB 状态输出（`SHOW ENGINE INNODB STATUS`）

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `mysys/dbug.cc` | DBUG 框架实现 |
| `include/mysql/service_debug_sync.h` | DBUG 宏定义 |
| `sql/sql_class.cc` | debug 系统变量处理 |
