# 多因素认证（MFA）深度解析

> 基于 MySQL 8.0.39 源码，涵盖 MFA 因素模型、三层数据结构（`LEX_MFA`/`I_multi_factor_auth`/`auth_factor_desc`）、认证握手循环、`authentication_policy` 校验、`user_attributes` JSON 持久化。
>
> **边界**：本篇讲 MFA 认证机制本身；账户限额 `mqh`/`USER_RESOURCES` 见 [`security_context.md`](security_context.md)（认证与授权主题）；VIO 通信抽象见 [`vio.md`](../infra/vio.md)；动态权限与 `SHOW GRANTS` 见认证与授权主题（待补链接）。

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

多因素认证（Multi-Factor Authentication，MFA）：一个账户最多绑定 **3 个认证因素**（`MAX_AUTH_FACTORS = 3`，`include/mysql_com.h:87`），登录时必须**依次通过所有启用的因素**才能建立连接。第一因素是传统主认证，第二、三因素是可选附加认证。

### 用途

解决"单密码认证一旦泄露就完全失守"的问题。即使第一因素（密码）泄露，缺少第二因素（如 FIDO2/WebAuthn 硬件设备）仍无法登录——这是"something you know"（密码）+"something you have"（设备）的叠加。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0 | 引入 MFA 框架（WL#11634），`MAX_AUTH_FACTORS=3`，`ADD/MODIFY/DROP nth FACTOR` 语法 |
| 8.0.27 | `authentication_fido` 演进为 `authentication_webauthn`（FIDO2），支持无密码 + 注册（registration）流程 |

> 因素模型与注册机制自 8.0 引入后基本稳定；插件的具体演进（FIDO→WebAuthn）见官方文档。

---

## 理论基础

### 设计思想与权衡

**核心决策：用"因素列表"而非"单插件多凭据"**。MFA 不是让一个认证插件存多个密码，而是给账户挂一串**互相独立的认证插件**，每个插件管自己那个因素的验证。这样每个因素可以选完全不同的机制（密码、FIDO 设备、LDAP 等），彼此正交。

**三层数据结构解耦了三个生命周期阶段**（这是理解 MFA 的关键）：

1. **解析期 `LEX_MFA`**（`table.h:2557`）：一条 `CREATE/ALTER USER` 里一个因素子句（`IDENTIFIED BY`、`ADD 2 FACTOR ...`）解析成的 AST 节点，记录插件名、认证串、`nth_factor`、`add/modify/drop_factor` 等标志位。
2. **运行期 `I_multi_factor_auth`**（`sql_mfa.h:45`）：账户在 ACL cache 里的因素链抽象接口，具体是 `Multi_factor_auth_list`（因素列表）包 `Multi_factor_auth_info`（单因素）。
3. **握手期 `auth_factor_desc[MAX_AUTH_FACTORS]`**（`sql_authentication.cc:3609`）：认证进行时，把因素从 ACL 缓存拷到握手上下文 `mpvio` 的数组里。

**"所有因素必须全通过"的语义代价**：这是安全性与兼容性的权衡。它带来的失效场景有二，都显式体现在代码里：

- 配置了附加因素的账户，**旧客户端（不支持 `MULTI_FACTOR_AUTHENTICATION` 能力）会被直接拒绝**——`do_multi_factor_auth` 一开始就检查能力（`sql_authentication.cc:3513-3514`），不支持就返回 `CR_AUTH_USER_CREDENTIALS`。
- 需要注册的因素（FIDO/WebAuthn）在注册完成前，允许以**沙盒模式（sandbox mode）**连接——这是"部分因素未通过但允许连接"的唯一例外，服务端限定该连接只能做注册操作。

### 理论溯源

多因素是通用安全模型（NIST 的 authentication factor 分类：knowledge/possession/inherence）。MySQL 把它落成"因素枚举 + 认证插件解耦"：`nthfactor` 枚举（`sql_mfa.h:35`）是因素的抽象编号，`I_multi_factor_auth` 接口是"一个可认证的因素"的抽象，认证插件是实现。这套"接口 + 插件"解耦与 VIO 的 vtable 多态（见 [`vio.md`](../infra/vio.md)）是同一思想——面向对象接口在 C/C++ 代码里的两种落地。

### 算法与数据结构

`nthfactor` 枚举（`sql_mfa.h:35`）：

```35:35:sql/auth/sql_mfa.h
enum class nthfactor { NONE = 1, SECOND_FACTOR, THIRD_FACTOR };
```

注意 `NONE = 1`（不是 0），`SECOND_FACTOR=2`、`THIRD_FACTOR=3`，正好对齐"第 n 因素"的编号。

`I_multi_factor_auth` 接口（`sql_mfa.h:45`）是因素链的抽象，关键方法：`validate_plugins_in_auth_chain`（校验插件链）、`validate_against_authentication_policy`（对照 `authentication_policy` 校验）、`serialize/deserialize`（与 `user_attributes` JSON 互转）、`init_registration/finish_registration`（注册流程）。

`Multi_factor_auth_list`（`sql_mfa.h:111`）内部是 `my_vector<I_multi_factor_auth *> m_factor`，`sort_mfa()` 保证层级始终是"2FA 后跟 3FA"。`Multi_factor_auth_info`（`sql_mfa.h:149`）代表单个因素，持有 `LEX_MFA *m_multi_factor_auth`。

### 他库对比与演进动机

PostgreSQL 的认证是单因素的（`pg_hba.conf` 按 host/db/user 选一种认证方法），没有账户级多因素链的概念；多因素要靠应用层或 LDAP/PAM 外挂。MySQL 的 MFA 把它做成**账户属性**（存在 `mysql.user.user_attributes` JSON 里），且上限 3 因素——3 是"覆盖 knowledge+possession+inherence 三类"所需的最小充分数，也是避免因素过多导致登录体验崩塌的工程取舍。

---

## 核心实现

### 主链路

```
CREATE/ALTER USER ... ADD 2 FACTOR ...
  → sql_yacc.yy 解析 → LEX_MFA（nth_factor=2, add_factor=true）
  → mysql_alter_user → 校验 authentication_policy
  → Multi_factor_auth_list::serialize → 写 mysql.user.user_attributes(JSON)

登录时
  客户端连接 → 1FA 认证（caching_sha2_password 等插件）
  → do_multi_factor_auth()         遍历 m_mfa 列表
    → 对每个因素：my_plugin_lock_by_name + authenticate_user()
      → 任一因素非 CR_OK 则失败
```

### 因素认证循环（关键环节）

`do_multi_factor_auth`（`sql_authentication.cc:3504`）是 2FA/3FA 的执行核心：

```3504:3574:sql/auth/sql_authentication.cc
static int do_multi_factor_auth(THD *thd, MPVIO_EXT *mpvio) {
  int res = CR_OK;
  /* user is not configured with Multi factor authentication */
  if (!mpvio->acl_user->m_mfa) return res;      // ① 无附加因素 → 直接成功
  /* If an old client connects ... return error. */
  if (!mpvio->protocol->has_client_capability(MULTI_FACTOR_AUTHENTICATION))
    return CR_AUTH_USER_CREDENTIALS;             // ② 旧客户端不支持 → 拒绝

  Multi_factor_auth_list *auth_factor = mpvio->acl_user->m_mfa->get_multi_factor_auth_list();
  for (auto m_it : auth_factor->get_mfa_list()) {
    Multi_factor_auth_info *af = m_it->get_multi_factor_auth_info();
    if (af->get_factor() == nthfactor::SECOND_FACTOR)
      mpvio->auth_info.current_auth_factor = 1;
    else if (af->get_factor() == nthfactor::THIRD_FACTOR)
      mpvio->auth_info.current_auth_factor = 2;  // ③ 定位到对应因素的 auth_string
    ...
    plugin_ref plugin = my_plugin_lock_by_name(thd, af->plugin_name(),
                                               MYSQL_AUTHENTICATION_PLUGIN);
    if (plugin) {
      mpvio->auth_info.auth_string = mpvio->auth_info.multi_factor_auth_info[...].auth_string;
      st_mysql_auth *auth = (st_mysql_auth *)plugin_decl(plugin)->info;
      res = auth->authenticate_user(mpvio, &mpvio->auth_info);  // ④ 每个因素独立验证
      if (res == CR_OK_AUTH_IN_SANDBOX_MODE) {   // ⑤ 沙盒模式：需注册但允许连接
        if (af->get_requires_registration())
          thd->security_context()->set_registration_sandbox_mode(true);
        return CR_OK;
      }
      plugin_unlock(thd, plugin);
      if (res != CR_OK) { mpvio->status = MPVIO_EXT::FAILURE; break; }  // ⑥ 任一失败即断
    }
  }
  return res;
}
```

逐段解释：

- **① `m_mfa == nullptr` 直接返回成功**：这是"账户只配了 1FA"的快速路径——没有附加因素，跳过整个循环。`m_mfa` 为空的时机见本文 Misc。
- **② 能力检查**：`MULTI_FACTOR_AUTHENTICATION` 是客户端能力位。老客户端不懂多因素协议，无法逐个提供 2FA/3FA 凭据，所以直接拒绝——这是"所有因素必须全通过"语义在协议层的兜底。
- **③ 因素定位**：`current_auth_factor` 从 0 开始（1FA 占 0），2FA 映射为 1、3FA 映射为 2，用于从 `multi_factor_auth_info[]` 数组里取对应该因素的 `auth_string`。
- **④ 独立验证**：每个因素调用**它自己的认证插件**的 `authenticate_user`。因素间彼此独立——2FA 用 FIDO 设备、3FA 用另一个插件，互不干扰。
- **⑤ 沙盒模式**：若因素声明 `requires_registration`（如 WebAuthn 设备尚未注册），`authenticate_user` 返回 `CR_OK_AUTH_IN_SANDBOX_MODE`，服务端放行但把连接置为沙盒模式，仅允许完成注册。这是"部分因素未通过仍可连接"的唯一例外。
- **⑥ 失败即断**：`res != CR_OK` 就 `break` 返回错误，整个登录失败——不存在"1 个因素过了就放行"。

### 解析期语法与标志位

`ALTER USER` 的因素子句（`sql_yacc.yy:17030` 附近）：

```
user ADD 2 FACTOR identification          → nth_factor=2, add_factor=true
user MODIFY 2 FACTOR identification       → nth_factor=2, modify_factor=true
user DROP 2 FACTOR                        → nth_factor=2, drop_factor=true
```

`factor` 规则只允许 2 和 3（`sql_yacc.yy:17110`），保证因素编号合法。`CREATE USER` 的 2FA/3FA 用 `AND identification` 连续写（`sql_yacc.yy:16818-16829`）：

```
CREATE USER 'u'@'h' IDENTIFIED BY 'p1' AND IDENTIFIED WITH authentication_webauthn AS '...'
```

`LEX_MFA` 的认证标志位（`table.h:2569-2589`）区分了三种"提供密码/插件"的写法：

| 标志位 | SQL 子句 | 语义 |
|---|---|---|
| `uses_identified_by_clause` | `IDENTIFIED BY 'pwd'` | 提供明文密码 |
| `uses_authentication_string_clause` | `IDENTIFIED ... AS 'hash'` | 提供已哈希串 |
| `uses_identified_with_clause` | `IDENTIFIED WITH plugin` | 指定认证插件 |
| `has_password_generator` | `BY RANDOM PASSWORD` | 随机生成密码 |

三者可组合，例如 `IDENTIFIED WITH caching_sha2_password BY RANDOM PASSWORD` 同时置 `uses_identified_with_clause` 和 `has_password_generator`。

### 持久化：user_attributes JSON

因素链通过 `Multi_factor_auth_list::serialize/deserialize` 存进 `mysql.user` 的 `user_attributes` JSON 列，随账户持久化。`ACL_USER::m_mfa`（`sql_auth_cache.cc`）从该列反序列化，缓存到内存供认证时使用。这就是"账户级多因素"的存储基础——因素不是全局配置，而是每个账户自己的属性。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `authentication_policy` | `"*,,"` | Global | 逗号分隔最多 3 个插件名，约束 1FA/2FA/3FA 各允许哪个插件；`*` 表示任意，空表示该因素可选 |
| `generated_random_password_length` | `20` | Session | `BY RANDOM PASSWORD` 生成密码的长度（范围 5-255） |
| `default_authentication_plugin` | `caching_sha2_password` | Global | 1FA 默认插件（已废弃，被 `authentication_policy` 取代） |

> 默认值已从 `sys_vars.cc` 核实：`authentication_policy` DEFAULT `"*,,"`（第 7669 行）、`generated_random_password_length` DEFAULT(20) VALID_RANGE(5,255)（第 7531 行）。

---

## Misc

### 易混淆概念

- **`IDENTIFIED BY` vs `IDENTIFIED WITH` vs `BY RANDOM PASSWORD`**：三者是"认证方式声明"的不同维度。`IDENTIFIED BY` 给密码、`IDENTIFIED WITH` 给插件、`BY RANDOM PASSWORD` 让服务端生成。它们对应的 `LEX_MFA` 标志位各自独立，可组合。
- **`m_mfa == nullptr` 的时机**：账户只配了 1FA、从未执行 `ADD 2/3 FACTOR`。`ACL_USER` 构造时 `m_mfa = nullptr`（`sql_auth_cache.cc:368`）；`CREATE USER ... IDENTIFIED BY` 不带 `AND ...` 时也不创建因素链。认证时 `do_multi_factor_auth` 据此跳过循环。
- **`mqh`（USER_RESOURCES）与 MFA 无关**：`mqh` 是账户资源限额（每小时查询/更新/连接数），见 [`security_context.md`](security_context.md)，与认证因素是两个独立维度。

---

## 参考

**官方文档**
- MySQL 8.0 Reference Manual → Multi-Factor Authentication
- WorkLog: [WL#11634 — Multi-factor authentication](https://dev.mysql.com/worklog/task/?id=11634)

**相关文档**
- 账户限额 `mqh`/`USER_RESOURCES`、`m_user`/`m_priv_user` 见 [`security_context.md`](security_context.md)
- VIO 通信抽象见 [`vio.md`](../infra/vio.md)
- 动态权限与 `SHOW GRANTS` 见认证与授权主题（待补链接）
