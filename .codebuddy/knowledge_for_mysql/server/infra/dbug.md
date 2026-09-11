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

`DBUG_TRACE` / `DBUG_PRINT` / `DBUG_RETURN` / `DBUG_ENTER` / `DBUG_DUMP` 等宏的展开与语义。

- `DBUG_DUMP(keyword, ptr, len)`（`my_dbug.h:195`）→ `_db_dump_(__LINE__, keyword, ptr, len)`：debug 模式下把 `ptr` 起始的 `len` 字节按十六进制 dump 到 trace 输出；release 版是空宏 `do{}while(0)`（`my_dbug.h:273`），零开销。
- 用法示例：`ha_innobase::rnd_pos`（`ha_innodb.cc:10870`）里 `DBUG_DUMP("key", pos, ref_length)` 用于 dump 主键值（ref）调试。

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

### 内存破坏哨兵（canary 思想）

MySQL 源码字面没有 `canary` 标识符，但用 magic 值实现"内存破坏检测哨兵"（canary 思想）：

- `THD_SENTRY_MAGIC 0xfeedd1ff` / `THD_SENTRY_GONE 0xdeadbeef`（`sql/sql_class.h:338`）：`THD::dbug_sentry`（注释 "watch out for memory corruption"，`sql_class.h:1499`）在 debug 版构造函数置 MAGIC（`sql_class.cc:759`）、析构置 GONE（`sql_class.cc:1449`）；`THD_CHECK_SENTRY` 断言校验，检测 THD 结构体被越界写坏。
- InnoDB 内存块 `magic_n`（`mem0mem.h:407`）：`MEM_BLOCK_MAGIC_N 0x445566778899AABB` / `MEM_FREED_BLOCK_MAGIC_N 0xBBAA998877665544`；`mem_block_validate`（`mem0mem.ic:110`）校验失败抛 fatal "Memory block is invalid"；释放时改写（`memory.cc:430`）以抓 double-free / use-after-free。
- 动态数组块 `DYN_BLOCK_MAGIC_N 375767`（`dyn0types.h:41`）。
- 区分：`BINLOG_MAGIC`（`\xfe\x62\x69\x6e`）、`tc_log_magic`（`tc_log.cc:311`）是"格式标识魔数"，用于识别文件类型，非"被篡改即告警"的检测哨兵，语义不同于 canary。
- 栈保护（stack canary）由编译器 `-fstack-protector` 注入，主代码不显式写。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `mysys/dbug.cc` | DBUG 框架实现 |
| `include/mysql/service_debug_sync.h` | DBUG 宏定义 |
| `sql/sql_class.cc` | debug 系统变量处理 |
