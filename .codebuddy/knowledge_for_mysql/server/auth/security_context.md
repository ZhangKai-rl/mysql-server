# Security_context 认证上下文深度解析

> 基于 MySQL 8.0.39 源码，涵盖 `Security_context` 结构、`m_user`/`m_priv_user` 等身份字段的赋值链路、匿名用户、proxy 用户、认证握手到权限落地的完整流程。
>
> **边界**：本篇讲认证上下文（"我是谁、按谁的权限算"）；认证插件的多因素细节见 [`mfa.md`](mfa.md)；VIO 通信见 [`vio.md`](../infra/vio.md)；动态权限与 `SHOW GRANTS` 见认证与授权主题（待补链接）。

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

`Security_context`（`sql/auth/sql_security_ctx.h`）是**每个连接（THD）携带的"身份与权限"快照**。它记录了这条连接"客户端声称是谁"（`m_user`）、"服务器认定按谁的权限算"（`m_priv_user`）、主机、IP、全局权限位（`m_master_access`）、角色、动态权限 map 等。

### 用途

认证成功后，server 层所有权限检查（`check_global_access`、`check_access`、`has_global_grant` 等）都读这个对象，而不是每次都查 `mysql.user` 表。它把"一次认证"的结果固化成内存态，供整个会话生命周期复用。

### 版本演进

| 版本 | 变化 |
|------|------|
| 早期 | 权限直接散落在 THD 字段里 |
| 5.x | 收敛成 `Security_context`，引入 proxy user |
| 8.0 | 加入角色（roles）、动态权限（`has_global_grant`）、`m_acl_map`（角色合并后的访问图）、多因素认证 |

---

## 理论基础

### 设计思想与权衡

**核心区分是"登录身份"与"权限身份"的分离**。`m_user`（登录用户）和 `m_priv_user`（权限用户）在大多数情况下相同，但匿名用户和 proxy 用户会让它们不同——这正是分开存储的原因。

**为什么不能用 `m_user` 一个字段搞定**：权限判定必须基于"服务器在 ACL cache 里真正匹配到的那条账户记录"，而不是"客户端随口报的名字"。如果客户端报了一个不存在的用户名、却匹配到了匿名账户 `''@'%'`，那权限必须按匿名账户算，而不是按那个不存在的名字算。于是需要 `m_priv_user` 单独记录"真正生效的账户名"。

**权衡与代价**：多存一份 `priv_user`/`priv_host` 增加了字段和赋值路径的复杂度（容易出现两处赋值不一致的 bug），但换来的是 proxy/匿名场景下的正确权限语义——这是 MySQL 权限模型灵活性的基础。

### 理论溯源

"身份 vs 授权"的分离是访问控制的标准模型：认证（Authentication，确认你是谁）与授权（Authorization，决定你能做什么）是两个正交阶段。`m_user` 属于认证结果，`m_priv_user`/`m_master_access` 属于授权结果。proxy user 是"委托授权"的体现——一个账户把自己的身份委托给另一个账户使用（`GRANT PROXY`）。

### 算法与数据结构

`Security_context` 关键字段（`sql_security_ctx.h`）：

```325:357:sql/auth/sql_security_ctx.h
  /** m_user - user of the client, set to NULL until the user has been read from the connection */
  String m_user;

  /** m_host - host of the client */
  String m_host;

  /** m_ip - client IP */
  ...
  /** m_priv_user - The user privilege we are using. May be "" for anonymous user. */
  char m_priv_user[USERNAME_LENGTH];
  size_t m_priv_user_length;

  /** m_proxy_user - proxy user */
  char m_proxy_user[USERNAME_LENGTH + HOSTNAME_LENGTH + 6];
  size_t m_proxy_user_length;

  /** The host privilege we are using */
  char m_priv_host[HOSTNAME_LENGTH + 1];
  size_t m_priv_host_length;

  /** Global privileges from mysql.user. */
  Access_bitmask m_master_access;
```

注意 `m_priv_user` 是固定大小的 `char[]`（不是 `String`），注释明确写着 "May be "" for anonymous user"。

---

## 核心实现

### 主链路（认证 → 上下文赋值）

```
客户端连接
  → conn_handler → login_connection → check_connection → acl_authenticate
    → parse_client_handshake_packet   解析握手包里的 user_name（客户端原始名）
    → find_mpvio_user                 用 user_name 去 ACL cache 匹配 → ACL_USER
    → do_auth_once                    执行认证插件校验密码
    → do_multi_factor_auth            多因素认证（若配置）
    → server_mpvio_update_thd         赋 m_user（登录用户）
    → assign_priv_user_host           赋 m_priv_user/m_priv_host（权限账户）
```

### `m_user` 赋值（登录用户）

`server_mpvio_update_thd`（`sql_authentication.cc:3614`）：

```3618:3620:sql/auth/sql_authentication.cc
  thd->security_context()->assign_user(
      mpvio->auth_info.user_name,
      (mpvio->auth_info.user_name ? strlen(mpvio->auth_info.user_name) : 0));
```

`mpvio->auth_info.user_name` 来自 `parse_client_handshake_packet` 从握手包字节流 `my_strndup` 出的原始用户名（`sql_authentication.cc:3067`）。所以 `m_user` 是**客户端 `-u` 参数说了算**的字符串。

### `m_priv_user` 赋值（权限用户）

`assign_priv_user_host`（`sql_authentication.cc:3722`）：

```3722:3725:sql/auth/sql_authentication.cc
inline void assign_priv_user_host(Security_context *sctx, ACL_USER *user) {
  sctx->assign_priv_user(user->user, user->user ? strlen(user->user) : 0);
  sctx->assign_priv_host(user->host.get_host(), user->host.get_host_len());
}
```

注意它取的是 **`ACL_USER *user`**（ACL cache 里匹配到的账户记录），不是握手包原始名。它在 `acl_authenticate` 里被调用（`sql_authentication.cc:3964`）：

```3963:3964:sql/auth/sql_authentication.cc
    if (mpvio.can_authenticate())
      assign_priv_user_host(sctx, const_cast<ACL_USER *>(acl_user));
```

而 `assign_priv_user` 里，长度为零时清空 `m_priv_user`（`sql_security_ctx.cc:1023`）——这正是匿名用户的落点：

```1023:1030:sql/auth/sql_security_ctx.cc
  if (priv_user_arg_length) {
    m_priv_user_length =
        std::min(priv_user_arg_length, sizeof(m_priv_user) - 1);
    strmake(m_priv_user, priv_user_arg, m_priv_user_length);
  } else {
    *m_priv_user = 0;       // 匿名用户 → m_priv_user 为空
    m_priv_user_length = 0;
  }
```

### 匿名用户：m_user 与 m_priv_user 分离的典型

当 `mysql.user` 表里有 `''@'%'` 匿名账户，客户端用任意用户名连接时：

```
客户端: mysql -uanyuser
服务器: ACL cache 匹配到 ''@'%'（匿名账户）
结果:   m_user      = "anyuser"（客户端输入）
        m_priv_user = ""          （匹配到的匿名账户）
```

`m_priv_user` 的注释直接点明 "May be "" for anonymous user"。

### proxy 用户：委托授权

proxy user 通过 `mysql.proxies_priv` 表配置（`scripts/mysql_system_tables.sql:585` 建表），语法是 `GRANT PROXY ON proxied_user TO proxy_user`。认证时若启用 `check_proxy_users`（`sql_authentication.cc:3925`），会把登录账户映射到被代理账户，权限按被代理账户算：

```3925:3925:sql/auth/sql_authentication.cc
    bool proxy_check = check_proxy_users && !*mpvio.auth_info.authenticated_as;
```

proxy 时 `m_user` 是登录账户，`m_priv_user` 是代理目标账户。此外 `m_proxy_user` 单独记录代理关系（`assign_proxy_user` 在 `sql_authorization.cc:7547` 赋值，格式 `'to_user'@'to_host'`）。

---

## 相关的系统变量/状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `check_proxy_users` | OFF | Global | 是否按 `GRANT PROXY` 映射代理用户身份 |
| `mysql_native_password_proxy_users` | OFF | Global | 该插件是否参与 proxy 映射 |
| `sha256_password_proxy_users` | OFF | Global | 该插件是否参与 proxy 映射 |

> 默认值已从 `sys_vars.cc` 核实。proxy 映射需要 `check_proxy_users=ON` 且对应插件开启 `*_proxy_users`。

---

## Misc

### 易混淆概念

- **`m_user` vs `m_priv_user`**：登录用户（客户端报的名字）vs 权限用户（ACL 匹配到的账户名）。普通连接相同；匿名/proxy 时不同。
- **`m_host` vs `m_ip` vs `m_priv_host`**：客户端主机名（可能被反解）vs 客户端 IP（连接对端地址）vs 权限判定用的主机（ACL 记录里的 host，可能是 `%` 或具体主机名）。
- **`m_user` 是 `String`，`m_priv_user` 是 `char[]`**：前者可变长（客户端任意输入），后者定长（USERNAME_LENGTH），反映"登录名自由 vs 权限账户来自系统表"的语义差异。

---

## 参考

**官方文档**
- MySQL 8.0 Reference Manual → Connection Access（连接访问控制、匿名用户、代理用户）
- MySQL 8.0 Reference Manual → Account Categories（系统用户/普通用户）

**相关文档**
- 认证插件的多因素细节见 [`mfa.md`](mfa.md)
- VIO 通信抽象见 [`vio.md`](../infra/vio.md)
