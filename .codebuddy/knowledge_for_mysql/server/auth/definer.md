# definer 与 SQL SECURITY 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 definer 概念、`SQL SECURITY DEFINER/INVOKER` 两种权限语义、默认 definer 的生成、`binlog_invoker`/`Q_INVOKER` 的主从一致性传递。
>
> **边界**：本篇讲 definer 机制本身（定义者 + 权限委托 + 复制传递）；definer 默认值用到的 `priv_user`/`priv_host` 见 [`security_context.md`](security_context.md)；多因素认证见 [`mfa.md`](mfa.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

definer（定义者）是 MySQL 四类"可被执行的持久化对象"的元数据属性，记录"这个对象是谁创建/拥有的"，格式为 `'user'@'host'`：

| 对象类型 | 例子 |
|---|---|
| 存储过程/函数（Stored Procedure/Function） | `CREATE PROCEDURE ...` |
| 触发器（Trigger） | `CREATE TRIGGER ...` |
| 视图（View） | `CREATE VIEW ...` |
| 事件（Event） | `CREATE EVENT ...` |

配合 `SQL SECURITY` 子句，definer 决定"执行这个对象时按谁的权限做检查"。

### 用途

解决"低权限用户安全访问高权限数据"的权限委托问题：一个没有 `SELECT sensitive_data` 权限的用户，可以通过一个 `SQL SECURITY DEFINER` 的存储过程（definer 是高权限用户）安全地读取聚合结果，而不需要直接获得敏感表的权限。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.0 | 引入存储例程 + definer；`SQL SECURITY DEFINER/INVOKER` |
| 5.0.13 | 视图/触发器/事件也支持 DEFINER 子句 |
| 8.0 | 引入 `SET_USER_ID` 动态权限，替代 `SUPER` 作为"可指定任意 definer"的权限 |

---

## 理论基础

### 设计思想与权衡

**核心是"权限受控委托"**。definer 让对象的创建者可以把"自己的一部分访问能力"封装进对象，供调用者按需使用，而调用者本身不必拥有那些权限。

**`SQL SECURITY` 两种模式的取舍**（这是本机制最核心的权衡）：

- `SQL SECURITY DEFINER`（默认）：执行时**切换 security context 到 definer**，按 definer 的权限做检查。代价：definer 权限泄漏风险——任何被授权 `EXECUTE` 的人都能借用 definer 的权限。
- `SQL SECURITY INVOKER`：执行时**用调用者（invoker）**的权限。代价：每个调用者都必须自己拥有对象内访问的所有表的权限，失去了"封装"的价值。

**默认值是 DEFINER**（`SP_DEFAULT_SUID_MAPPING = SP_IS_SUID`，`sp.h:71`），因为 DEFINER 才是"委托"的语义基础；INVOKER 是给"想让调用者自负其责"的场景。

**一个隐含约束：触发器不能 INVOKER**。`sp_head.cc:2435` 断言 `suid != SP_IS_NOT_SUID`——因为触发器由 DML 隐式触发，没有"调用者"这个概念，只能用 DEFINER 语义。

**definer 必须主从一致**：因为 definer 影响权限语义，主从复制时 `CREATE/ALTER PROCEDURE/TRIGGER/VIEW/EVENT` 的 definer 必须随 binlog 传到从库，从库才能用同一个 definer 重建对象（否则权限语义不一致）。这正是 `m_binlog_invoker` 机制存在的意义。

### 理论溯源

definer 是"授权委托（delegation）"的落地——对象的定义者把自己持有的访问权委托给对象，调用者通过调用对象间接获得该访问权。这与 proxy user（见 [`security_context.md`](security_context.md)）是同一思想的两条路径：proxy 委托的是"账户身份"，definer 委托的是"对象执行时的权限"。

### 算法与数据结构

`SQL SECURITY` 的枚举，存储例程侧（`sql_lex.h:199`）：

```199:203:sql/sql_lex.h
enum enum_sp_suid_behaviour {
  SP_IS_DEFAULT_SUID = 0,
  SP_IS_NOT_SUID,   // SQL SECURITY INVOKER
  SP_IS_SUID        // SQL SECURITY DEFINER
};
```

视图侧（`table.h:2485`）：

```2485:2487:sql/table.h
#define VIEW_SUID_INVOKER 0
#define VIEW_SUID_DEFINER 1
#define VIEW_SUID_DEFAULT 2
```

两个枚举名不同但语义一致：`SP_IS_SUID == VIEW_SUID_DEFINER`，`SP_IS_NOT_SUID == VIEW_SUID_INVOKER`。它们最终都映射到 DD 的 `View::ST_DEFINER/ST_INVOKER`（`dd_routine.cc:352-357`）。

---

## 核心实现

### 默认 definer 的生成

建对象时若未显式写 `DEFINER = ...`，用当前用户。`get_default_definer`（`sql_parse.cc:6754`）：

```6754:6761:sql/sql_parse.cc
void get_default_definer(THD *thd, LEX_USER *definer) {
  const Security_context *sctx = thd->security_context();

  definer->user.str = sctx->priv_user().str;
  definer->user.length = strlen(definer->user.str);

  definer->host.str = sctx->priv_host().str;
  definer->host.length = strlen(definer->host.str);
```

关键：definer 取的是 **`priv_user`/`priv_host`（权限身份）**，不是 `user`（登录身份）——这与 [`security_context.md`](security_context.md) 里讨论的区分一致。

### definer 权限校验

指定了与当前用户不同的 definer 时，需要额外权限（`sql_view.cc:537`）：

```537:545:sql/sql_view.cc
  if (lex->definer &&
      (strcmp(lex->definer->user.str,
              thd->security_context()->priv_user().str) != 0 ||
       my_strcasecmp(system_charset_info, lex->definer->host.str,
                     thd->security_context()->priv_host().str) != 0)) {
    Security_context *sctx = thd->security_context();
    if (!(sctx->check_access(SUPER_ACL) ||
          sctx->has_global_grant(STRING_WITH_LEN("SET_USER_ID")).first)) {
      my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), "SUPER or SET_USER_ID");
```

即：想"替别人"定义对象，需要 `SUPER` 或 `SET_USER_ID` 动态权限（8.0 里 `SET_USER_ID` 取代了 `SUPER` 的这个用途）。

### 执行时的权限切换

`SQL SECURITY DEFINER` 的对象执行时，切换到 definer 的 security context（`sp_head.cc:3390-3396`）：

```3390:3396:sql/sp_head.cc
  LEX_CSTRING definer_user = {m_definer_user.str, m_definer_user.length};
  LEX_CSTRING definer_host = {m_definer_host.str, m_definer_host.length};

  if (m_chistics->suid != SP_IS_NOT_SUID &&   // SQL SECURITY DEFINER
      m_security_ctx.change_security_context(thd, definer_user, definer_host,
                                             m_db.str, save_ctx)) {
```

`change_security_context` 把当前 THD 的 security context 临时换成 definer，执行完再恢复。`SP_IS_NOT_SUID`（INVOKER）则跳过这个切换，沿用调用者上下文。

### binlog_invoker：definer 的主从传递（主链路）

```
主库执行 CREATE/ALTER PROCEDURE/TRIGGER/VIEW/EVENT（或 GRANT/REVOKE）
  → binlog_invoker() 置 m_binlog_invoker = true          （标记"要记录执行者"）
  → Query_log_event::write() 检测 need_binlog_invoker()   （log_event.cc:3489）
    → 编码 Q_INVOKER status 字段（user + host）
从库 SQL 线程读 Query_log_event
  → 解析 Q_INVOKER → set_invoker() 存 m_invoker_user/host
  → get_definer() 用 m_invoker_user/host 作为 definer/grantor
```

### 设置标志与编码 invoker

`binlog_invoker()` 是 THD 的 setter（`sql_class.h:4320`），在 `GRANT`/`REVOKE`（`sql_parse.cc:4018`、`4076`）、`CREATE/ALTER PROCEDURE`（`sql_parse.cc:4381`）等处调用。

`Query_log_event::write()` 消费标志，编码 invoker（`log_event.cc:3489-3510`）：

```3489:3510:sql/log_event.cc
  if (thd && thd->need_binlog_invoker()) {
    ...
    if (thd->slave_thread && thd->has_invoker()) {
      invoker_user = thd->get_invoker_user();
      invoker_host = thd->get_invoker_host();
    } else {
      Security_context *ctx = thd->security_context();
      LEX_CSTRING priv_user = ctx->priv_user();
      LEX_CSTRING priv_host = ctx->priv_host();
      ...
```

主库写 invoker 用 `priv_user`/`priv_host`；从库再转发（级联复制）时用 `m_invoker_user`/`m_invoker_host`（从 master 传来的值）。编码成 `Q_INVOKER`（code=11）status 字段：1 字节 user 长度 + user + 1 字节 host 长度 + host。

### Q_INVOKER 的解码与使用

从库 SQL 线程解析 `Q_INVOKER`（`libbinlogevents/src/statement_events.cpp:239`）读出 user/host，存进 `m_invoker_user`/`m_invoker_host`（`set_invoker`，`sql_class.h:4323`）。后续 `get_definer`（`sql_class.cc:2528`）检查 `slave_thread && has_invoker()`，用 master 传来的值作为 definer/grantor。

---

## 相关的系统变量/状态变量

definer 本身无专属系统变量。相关的权限控制是动态权限 `SET_USER_ID`（允许指定任意 definer，见 [`mfa.md`](mfa.md) 的相邻权限主题），以及 `check_proxy_users` 等 proxy 相关变量（见 [`security_context.md`](security_context.md)）。

---

## Misc

### 易混淆概念

- **definer vs invoker**：definer 是对象的**定义者**（创建者，元数据一部分）；invoker 是对象的**调用者**（执行时是谁发起的）。`SQL SECURITY DEFINER` 按 definer 权限执行，`SQL SECURITY INVOKER` 按 invoker 权限执行。
- **definer 的 `DEFINER=` 子句 vs `Q_INVOKER`**：前者是 SQL 文本的一部分（`CREATE DEFINER=... PROCEDURE`），mysqlbinlog 直接可见；后者是 `Query_log_event` 的 status 字段（`Q_INVOKER`，code=11），存的是账户管理语句的 grantor，mysqlbinlog 默认不展示，只给从库 SQL 线程消费。
- **`m_binlog_invoker` 的注释名 `current_user_used` 是历史遗留**：字段注释（`sql_class.h:4356`）写的是 `current_user_used`，但实际字段名是 `m_binlog_invoker`——注释没跟着改名更新。

---

## 参考

**官方文档**
- MySQL 8.0 Reference Manual → Stored Programs and Views → Access Control for Stored Programs and Views
- MySQL 8.0 Reference Manual → `CREATE PROCEDURE` / `CREATE VIEW` 的 `DEFINER` 与 `SQL SECURITY`

**相关文档**
- definer 默认值用到的 `priv_user`/`priv_host` 见 [`security_context.md`](security_context.md)
- 多因素认证见 [`mfa.md`](mfa.md)
