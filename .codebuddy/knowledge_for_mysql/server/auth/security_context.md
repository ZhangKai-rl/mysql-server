# Security_context 认证上下文深度解析

> 基于 MySQL 8.0.39 源码，剖析 `Security_context`——AuthN 与 AuthZ 之间的**唯一交界对象**：它的完整内存布局、每个字段由谁在何时赋值、`acl_getroot()` 的装配算法、与 ACL 缓存 / `Acl_map` 借出 / role 展开 / partial revokes 的接口，以及 `USER()` 与 `CURRENT_USER()` 为什么不同。
>
> **边界**：本篇讲认证**之后**这个对象本身（"我是谁、按谁的权限算"）。握手协议、插件协商与凭据校验（认证**过程**）见 [`authentication.md`](authentication.md)；权限**判定链**如何消费它见 [`authorization.md`](authorization.md)；definer 场景下如何临时切换它见 [`definer.md`](definer.md)；多因素凭据存储见 [`mfa.md`](mfa.md) 与 [`account_management.md`](account_management.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`Security_context`（`sql/auth/sql_security_ctx.h`）是**每条连接（THD）携带的"身份 + 权限"快照**。它把"一次认证"的结果固化成内存态，供整个会话生命周期复用——server 层所有权限检查（`check_access`、`check_global_access`、`has_global_grant`…）都读它，而不是每次去查 `mysql.user`。

它同时属于两个世界：

| 归属 | 字段 | 谁读 |
|---|---|---|
| **认证结果**（协议层身份） | `m_user` / `m_host` / `m_ip` / `m_host_or_ip` | `USER()`、审计日志 |
| **授权结果**（权限层身份） | `m_priv_user` / `m_priv_host` / `m_master_access` / `m_acl_map` / `m_restrictions` | `CURRENT_USER()`、所有权限判定 |

### 用途

回答三个实际问题：

1. **`USER()` 和 `CURRENT_USER()` 为什么不一样？** → 见[核心实现六](#六user-vs-current_user-读的字段不同)
2. **为什么 `GRANT ... ON *.*` 之后老会话不生效？** → `m_master_access` 是登录时的快照，见[核心实现三](#三checkout_access_maps与-acl_cache-的借出语义)
3. **definer 怎么"临时变成别人"？** → `change_security_context()` 只换指针，见 [`definer.md`](definer.md)

### 版本演进

| 版本 | 变化 |
|---|---|
| 早期 | 权限直接散落在 THD 字段里 |
| 5.x | 收敛成 `Security_context`，引入 proxy user |
| 8.0 | 加入 role（`m_active_roles`）、动态权限（`has_global_grant`）、**`m_acl_map`**（role 展开后的扁平权限快照）、partial revokes（`m_restrictions`） |
| 8.0.16+ | `m_restrictions`（partial revokes） |
| 8.0.27+ | `m_registration_sandbox_mode`（MFA 设备登记态） |

---

## 理论基础

### 设计思想与权衡

**核心区分是"登录身份"与"权限身份"的分离**。`m_user`（登录用户）与 `m_priv_user`（权限用户）多数情况下相同，但匿名用户和 proxy 用户会让它们不同——这正是分开存储的原因。

**为什么不能用 `m_user` 一个字段搞定**：权限判定必须基于"服务器在 ACL cache 里真正匹配到的那条账户记录"，而不是"客户端随口报的名字"。如果客户端报了一个不存在的用户名、却匹配到了匿名账户 `''@'%'`，权限必须按匿名账户算。于是需要 `m_priv_user` 单独记录"真正生效的账户名"。

**代价**：多存一份 `priv_user`/`priv_host` 增加赋值路径复杂度（易出不一致 bug），换来 proxy/匿名场景下的正确语义。

**第二个权衡：权限快照 vs 每次现查**。把全局权限位、role 展开结果、动态权限在登录时算好并缓存，代价是"GRANT 后老会话不生效"；收益是每条 SQL 的判定几乎零成本（一次位与运算）。**`m_acl_map` 是唯一被"借出"而非"拷贝"的部分**——因为它是共享的不可变快照，见下文。

### 理论溯源

"身份 vs 授权"分离是访问控制的标准模型：Authentication（确认你是谁）与 Authorization（决定你能做什么）正交。`m_user` 属于前者，`m_priv_user`/`m_master_access` 属于后者。proxy user 是"委托"的体现——一个账户把身份委托给另一个（`GRANT PROXY`）。

### 算法与数据结构

#### 完整字段（`sql/auth/sql_security_ctx.h`，private 段）

```cpp
 private:
  /** m_user - user of the client, set to NULL until the user has been read
             from the connection */
  String m_user;
  /** m_host - host of the client */
  String m_host;
  /** m_ip - client IP */
  String m_ip;
  /** m_host_or_ip - points to host if host is available, otherwise ip */
  String m_host_or_ip;
  String m_external_user;

  /** m_priv_user - The user privilege we are using. May be "" for anonymous user. */
  char m_priv_user[USERNAME_LENGTH];
  size_t m_priv_user_length;
  char m_proxy_user[USERNAME_LENGTH + HOSTNAME_LENGTH + 6];
  size_t m_proxy_user_length;
  /** The host privilege we are using */
  char m_priv_host[HOSTNAME_LENGTH + 1];
  size_t m_priv_host_length;

  /** Global privileges from mysql.user. */
  Access_bitmask m_master_access;
  /** Privileges for current db */
  Access_bitmask m_db_access;

  /**
    password expiration flag.
    This flag is set according to connecting user's context and not the
    effective user.
  */
  bool m_password_expired;
  List_of_auth_id_refs m_active_roles;
  Acl_map *m_acl_map;
  /** True if this account can't be logged into. */
  bool m_is_locked;
  /** True if the skip_grants_user is set. */
  bool m_is_skip_grants_user;

  bool m_executed_drop_policy;
  bool m_has_drop_policy;
  std::unique_ptr<std::function<void(Security_context *)>> m_drop_policy;
  Restrictions m_restrictions;
  /** sandbox（MFA 设备登记）模式：只允许 ALTER USER / SET PASSWORD */
  bool m_registration_sandbox_mode;

  /** m_thd - nullptr if this does not belong to any THD yet */
  THD *m_thd;
```

#### ★ 内存布局的三个要点

1. **5 个 `String`**（`m_user`/`m_host`/`m_ip`/`m_host_or_ip`/`m_external_user`）各自持有堆内存。赋值有两套 API：
   - `assign_xxx()` —— **深拷贝**；
   - `set_xxx_ptr()` —— **只借指针，不复制**（`acl_getroot()` 用的是这套，所以调用方必须保证缓冲区活得比 sctx 久）。
2. **3 个定长 `char[]` 内联在对象里**（`m_priv_user` / `m_priv_host` / `m_proxy_user`），写入用 `strmake` 且 `std::min(len, sizeof-1)` **静默截断**。`m_proxy_user` 多 6 字节是因为要装 `'u'@'h'` 这种带引号形式。
3. **`m_acl_map` 是借来的裸指针，不拥有**——所以 `copy_security_ctx()` 里明确置 `nullptr`（注释原文：*"acl maps are reference counted we can't copy or share them!"*）。

`m_active_roles` 的真实类型是：

```cpp
typedef std::pair<LEX_CSTRING, LEX_CSTRING> Auth_id_ref;   // (user, host)
typedef std::vector<Auth_id_ref> List_of_auth_id_refs;
```

元素里的 `LEX_CSTRING` 是 `my_strdup` 出来的裸指针，必须靠 `clear_active_roles()` 手工 `my_free`。

#### `init()` 与生命周期

```cpp
void Security_context::init() {
  m_user.set((const char *)nullptr, 0, system_charset_info);
  m_host.set("", 0, system_charset_info);
  m_ip.set("", 0, system_charset_info);
  m_host_or_ip.set(STRING_WITH_LEN("connecting host"), system_charset_info);
  ...
  m_master_access = 0;
  m_db_access = NO_ACCESS;
  m_acl_map = nullptr;
  ...
}
```

★ `m_host_or_ip` 的**初值是字符串 `"connecting host"`**——认证成功之前的占位符（所以你在错误日志里偶尔能看到它）。

`destroy()` 会先 `execute_drop_policy()`，再归还 `m_acl_map` 并 `clear_active_roles()`（否则引用计数泄漏）。

---

## 核心实现

### 一、★ 两条装配路径（原文档的一大错误）

这是本篇最容易搞错的地方：**正常的客户端登录连接，走的是 `sql_authentication.cc` 里手写的一段装配代码，不是 `acl_getroot()`**。

| 路径 | 谁用 | 处理 mandatory_roles？ | 处理 `activate_all_roles_on_login`？ |
|---|---|---|---|
| **登录路径**（`sql_authentication.cc` 的 `acl_authenticate()`） | 客户端连接、`COM_CHANGE_USER` | ✅ | ✅ |
| **`acl_getroot()`**（`sql/auth/sql_auth_cache.cc`） | definer（SP/view/trigger/event）、复制 SQL 线程、component 服务 | ❌ **只激活 default roles** | ❌ |

★ **后果**：**DEFINER 上下文、复制线程、component 建的 sctx 都拿不到 `mandatory_roles`，也拿不到"全部已授予 role"**，只有真正的客户端登录连接才有。这是官网没明说但源码确凿的行为差异。

`acl_getroot()` 全仓库只有 5 个调用点：

| 位置 | 用途 |
|---|---|
| `sql/table.cc` `Table_ref::prepare_view_security_context()` | SUID view 的 definer sctx |
| `sql/auth/sql_security_ctx.cc` `change_security_context()` | definer（SP/view/trigger/event） |
| `sql/auth/service_security_context.cc` / `sql/server_component/security_context_imp.cc` | component service |
| `sql/rpl_rli.cc` | 复制 SQL 线程 |

### 二、`acl_getroot()`：装配主函数

> ★ 注意它在 **`sql/auth/sql_auth_cache.cc`**，不在 `sql_authentication.cc`。

真实签名（6 参，无默认值）：

```cpp
bool acl_getroot(THD *thd, Security_context *sctx, const char *user,
                 const char *host, const char *ip, const char *db);
```

函数上方有一段**罕见的自我批评**注释，值得原文引用：

> *"Gets user credentials without authentication and resource limit checks. This function is used to initialized a new Security_context. **It's a terrible anti-pattern that needs to go.**"*

完整实现（逐段）：

```cpp
bool acl_getroot(THD *thd, Security_context *sctx, const char *user,
                 const char *host, const char *ip, const char *db) {
  int res = 1;
  ACL_USER *acl_user = nullptr;

  sctx->set_user_ptr(user, user ? strlen(user) : 0);      // (A1) 协议层身份
  sctx->set_host_ptr(host, host ? strlen(host) : 0);
  sctx->set_ip_ptr(ip, ip ? strlen(ip) : 0);
  sctx->set_host_or_ip_ptr();

  if (!initialized) {                                     // (B) skip-grant-tables
    sctx->skip_grants();
    return false;
  }

  sctx->set_master_access(0);                             // (A2) 清空授权层
  sctx->cache_current_db_access(0);
  sctx->assign_priv_user("", 0);
  sctx->assign_priv_host("", 0);

  Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::READ_MODE);
  if (!acl_cache_lock.lock(false)) return true;

  /* Find acl entry in user database. */
  for (ACL_USER *acl_user_tmp = acl_users->begin();       // (C) 线性扫
       acl_user_tmp != acl_users->end(); ++acl_user_tmp) {
    if ((!acl_user_tmp->user && !user[0]) ||
        (acl_user_tmp->user && strcmp(user, acl_user_tmp->user) == 0)) {
      if (acl_user_tmp->host.compare_hostname(host, ip)) {
        acl_user = acl_user_tmp;
        res = 0;
        break;
      }
    }
  }

  if (acl_user) {
    sctx->clear_active_roles();                           // (D1) 只激活 default roles
    List_of_auth_id_refs default_roles;
    Auth_id_ref authid = create_authid_from(acl_user);
    get_default_roles(authid, default_roles);
    for (auto it = default_roles.begin(); it != default_roles.end(); ++it) {
      if (sctx->activate_role(it->first, it->second)) {
        sctx->clear_active_roles();     // ★ 任一失败就整体放弃，不部分激活
        break;
      }
    }

    if (sctx->get_active_roles()->size() == 0) {          // (D2) 无 role 的老路径
      for (ACL_DB *acl_db = acl_dbs->begin(); acl_db != acl_dbs->end(); ++acl_db) {
        if (!acl_db->user || (user && user[0] && !strcmp(user, acl_db->user))) {
          if (acl_db->host.compare_hostname(host, ip)) {
            if (!acl_db->db ||
                (db && (mysqld_partial_revokes()
                            ? (!strcmp(db, acl_db->db))
                            : (!wild_compare(db, strlen(db), acl_db->db,
                                             strlen(acl_db->db), false))))) {
              sctx->cache_current_db_access(acl_db->access);
              break;
            }
          }
        }
      }
      sctx->set_master_access(acl_user->access,
                              acl_restrictions->find_restrictions(acl_user));
    }

    sctx->assign_priv_user(user, user ? strlen(user) : 0);          // (E)
    sctx->assign_priv_host(acl_user->host.get_host(),
        acl_user->host.get_host() ? strlen(acl_user->host.get_host()) : 0);

    sctx->set_password_expired(acl_user->password_expired);         // (F)
    sctx->lock_account(acl_user->account_locked);
  }

  if (acl_user && sctx->get_active_roles()->size() > 0) {           // (G) 有 role
    sctx->checkout_access_maps();
    const Access_bitmask db_acl = db ? sctx->db_acl({db, strlen(db)}) : 0;
    sctx->cache_current_db_access(db_acl);
  }
  return res;
}
```

**逐段要点**：

- **(A)** 先无条件设身份字段，再清空授权层。用的是 `set_*_ptr`（不复制）。
- **(B)** `!initialized` 即 `--skip-grant-tables`：`acl_users` 是空的，无法查 ACL，直接 `skip_grants()` 后返回成功。
- **(C)** **线性扫** `acl_users`（不是 hash）。用户名是 `strcmp`（**大小写敏感**），host 用 `compare_hostname()`（支持 hostmask + IP）。
- **(D1)** **只激活 default roles**，且任一失败整体回滚。
- **(D2)** 无 role 时才扫 `acl_dbs` 缓存 `m_db_access`，并把 `m_master_access` 设为 `acl_user->access` + 该用户 restrictions。**有 role 时不设**——交给 (G)。
- **(E)** `priv_user` 用的是**传入的 `user` 参数**（不是 `acl_user->user`，因为匿名账号后者是 NULL）；`priv_host` 用 **`acl_user->host.get_host()`**（ACL 里的 host 模式，可能是 `%`）。
- **(G)** 有 role 时 `checkout_access_maps()` 会用展开后的快照**覆写** `m_master_access` 与 `m_restrictions`。

返回值 `res` 初值 1，只有找到 `acl_user` 才置 0 —— **找不到账号返回 true（错误），但 sctx 已被清空为"无权限 + 空 priv_user/priv_host"**，由调用方（如 `change_security_context`）负责报错。

#### `skip_grants()` 的语义

```cpp
void Security_context::skip_grants(const char *user = "skip-grants user",
                                   const char *host = "skip-grants host") {
  /* privileges for the user are unknown everything is allowed */
  set_host_or_ip_ptr("", 0);
  assign_priv_user(user, strlen(user));
  assign_priv_host(host, strlen(host));
  m_master_access = ALL_ACCESS;              // ★ 全位，不是 GLOBAL_ACLS 子集
  m_is_skip_grants_user = true;

  if (m_thd && m_thd->security_context() == this) {
    m_thd->set_system_user(true);            // ★ 所以能杀掉任何连接
    m_thd->set_connection_admin(true);
  }
}
```

★ 置位 `m_is_skip_grants_user` 后，`has_global_grant()` 对它**无条件返回 `{true, true}`**（含 grant option）。且它**不设 `m_restrictions`** ⇒ partial revokes 在 skip-grants 下也失效。

### 三、`checkout_access_maps()` 与 Acl_cache 的借出语义

```cpp
/**
  Subscribes to a cache entry of aggregated ACLs.
  A Security_context can only have one subscription at a time. If another one
  is requested, the former will be returned.

  We do this subscription before execution of every statement(prepared or
  conventional) as the global acl version might have increased due to
  a grant/revoke or flush. Hence, the granularity of after effects of
  grant/revoke or flush due to roles is per statement.
*/
void Security_context::checkout_access_maps(void) {
  if (m_acl_map != nullptr) {
    get_global_acl_cache()->return_acl_map(m_acl_map);
    m_acl_map = nullptr;
  }
  if (m_active_roles.size() == 0) return;        // ★ 没 role 就不需要 map
  Auth_id_ref uid;
  uid.first.str = this->m_priv_user;  uid.first.length = this->m_priv_user_length;
  uid.second.str = this->m_priv_host; uid.second.length = this->m_priv_host_length;
  m_acl_map = get_global_acl_cache()->checkout_acl_map(this, uid, m_active_roles);
  if (m_acl_map != nullptr)
    set_master_access(m_acl_map->global_acl(), m_acl_map->restrictions());
  else
    set_master_access(0);                        // ★ OOM 时 fail-closed（静默降为无权限）
}
```

★ **为什么需要"借出"而不是拷贝**：`Acl_map` 是**共享的不可变快照**，多个 sctx 可能命中同一张。引用计数（`std::atomic<int32> m_reference_count`）保证"还有人在用就不能 delete"。归还只是 `decrease_reference_count()`，真正删除由 `Acl_cache::flush_cache()` 在版本推进时做。

**key 由三部分组成**，这决定了"什么时候会拿到新的 map"：

```
key = create_acl_cache_hash_key(version, uid, active_roles)
      ↑                     ↑        ↑
      m_role_graph_version  账号     有序的活跃 role 列表
```

所以：**改了 role 图版本（GRANT/REVOKE/FLUSH）、换了账号、或改了 active role 集合** ⇒ 下次 checkout 拿到新 map，并据此重设 `m_master_access` / `m_restrictions`。

**★ 语句级 checkout 在哪**：

```cpp
// sql/sql_parse.cc, THD::reset_for_next_command() 末尾
  /*
    In case we're processing multiple statements we need to checkout a new
    acl access map here as the global acl version might have increased due to
    a grant/revoke or flush.
  */
  thd->security_context()->checkout_access_maps();
```

即**每条语句执行前重新借一次**——这正是官方"GRANT role 在语句级生效"的落点。

**`Acl_map` 里有什么**：

```cpp
class Acl_map {
  std::atomic<int32> m_reference_count;
  uint64 m_version;
  Access_bitmask m_global_acl;
  Db_access_map m_db_acls;          // std::map<std::string, Access_bitmask>
  Db_access_map m_db_wild_acls;     // 通配库权限
  Table_access_map m_table_acls;
  SP_access_map m_sp_acls, m_func_acls;
  Grant_acl_set m_with_admin_acls;
  Dynamic_privileges m_dynamic_privileges;   // std::map<std::string,bool>（value = with grant option）
  Restrictions m_restrictions;
};
```

### 四、判定链如何消费它：两层 `check_access`

#### 成员版（纯全局位 + partial revoke 否决）

```cpp
bool Security_context::check_access(Access_bitmask want_access,
                                    const std::string &db_name = "",
                                    bool match_any = false) {
  if ((want_access & DB_ACLS) && is_access_restricted_on_db(want_access, db_name))
    return false;
  return (match_any ? (m_master_access & want_access)
                    : ((m_master_access & want_access) == want_access));
}
```

★ **它不看 `m_db_access`**，db 级权限由自由函数单独叠加。

#### 自由函数版（完整路径，`sql/auth/sql_authorization.cc`）

关键片段：

```cpp
  if (sctx->check_access(want_access, db_name)) {         // ① 全局位够 → 短路放行
    if (!(sctx->check_access(SELECT_ACL, db_name))) {
      if (db && (!thd->db().str || db_is_pattern || strcmp(db, thd->db().str))) {
        // ★ 有 role 走 Acl_map，无 role 走 acl_get()
        if (sctx->get_active_roles()->size() > 0)
          db_access = sctx->db_acl({db, strlen(db)}, db_is_pattern);
        else
          db_access = acl_get(thd, sctx->host().str, sctx->ip().str,
                              sctx->priv_user().str, db, db_is_pattern);
      } else
        db_access = sctx->current_db_access();            // 当前库 → 用缓存
      *save_priv |= sctx->master_access(db_name) | db_access;
    } else
      *save_priv |= sctx->master_access(db_name);
    return false;
  }
```

★ **两个要点**：

1. `m_db_access`（`current_db_access()`）**只在 `db == thd->db()`（当前库）时被复用**，否则现算。
2. db 级权限的来源**取决于有没有 active role**：有 role 时查已展开的 `Acl_map`，无 role 时查 `acl_get()`（走 `db_cache`）。

`check_global_access()` 是薄封装，用 `match_any = true`：

```cpp
bool check_global_access(THD *thd, Access_bitmask want_access) {
  if (thd->security_context()->check_access(
          want_access, thd->db().str ? thd->db().str : "", true))
    return false;
  get_privilege_desc(command, sizeof(command), want_access);
  my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), command);
  return true;
}
```

### 五、`has_global_grant()`：动态权限的两条路径

```cpp
std::pair<bool, bool> Security_context::has_global_grant(const char *priv,
                                                         size_t priv_len) {
  if (!initialized || m_is_skip_grants_user) return {true, true};

  std::string privilege(priv, priv_len);

  if (m_acl_map == nullptr) {           // ① 无 role：现查全局动态权限表
    THD *thd = m_thd ? m_thd : current_thd;
    if (thd == nullptr) return {false, false};
    Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::READ_MODE);
    if (!acl_cache_lock.lock(false)) return {false, false};
    Role_id key(&m_priv_user[0], m_priv_user_length,
                &m_priv_host[0], m_priv_host_length);
    auto [it, it_end] = get_dynamic_privileges_map()->equal_range(key);
    it = std::find(it, it_end, privilege);
    if (it != it_end) return {true, it->second.second};
    return {false, false};
  }
  Dynamic_privileges::iterator it =     // ② 有 role：查已展开的 Acl_map
      m_acl_map->dynamic_privileges()->find(privilege);
  if (it != m_acl_map->dynamic_privileges()->end()) return {true, it->second};
  return {false, false};
}
```

★ **语义差异**：有 active role 时读 **`m_acl_map->dynamic_privileges()`**（已含 role 展开 + 转授展开）；**没有 role 时只读"直接授予该 auth_id 的"**，不会展开非活跃 role。想展开非活跃 role 必须显式传 `cumulative=true`（`fetch_global_grant()` 会临时造一个 sctx 强制激活所有 role 再查）。

#### ★ SYSTEM_USER / CONNECTION_ADMIN 不在 sctx 里

它们的真身是 **THD** 的成员：`THD::m_is_system_user` / `THD::m_is_connection_admin`（`std::atomic<bool>`），由 `set_system_user_flag()` / `set_connection_admin_flag()` 从 sctx 探测后写入：

```cpp
void set_connection_admin_flag(THD *thd, bool check_for_main_security_ctx = false) {
  Security_context *sctx = thd->security_context();
  if (check_for_main_security_ctx == false || sctx == &thd->m_main_security_ctx)
    thd->set_connection_admin(
        sctx->has_global_grant(consts::connection_admin.c_str(), ...).first ||
        sctx->check_access(SUPER_ACL));        // ★ CONNECTION_ADMIN = 动态权限 或 SUPER
}
```

★ 注意不对称：`CONNECTION_ADMIN` = 动态权限 **或** `SUPER_ACL`；`SYSTEM_USER` **只看动态权限**。且登录末尾的注释明确要求"必须在所有 role 激活之后"调用，否则会漏掉通过 role 拿到的 `SYSTEM_USER`。

### 六、`USER()` vs `CURRENT_USER()`：读的字段不同

```cpp
// sql/item_strfunc.cc
String *Item_func_user::val_str(String *) {
  if (!m_evaluated &&
      evaluate(current_thd->m_main_security_ctx.user().str,
               current_thd->m_main_security_ctx.host_or_ip().str))
    return nullptr;
  ...
}
String *Item_func_current_user::val_str(String *) {
  if (!m_evaluated) {
    Security_context *const ctx = context->security_ctx
                                      ? context->security_ctx
                                      : current_thd->security_context();
    if (evaluate(ctx->priv_user().str, ctx->priv_host().str)) return nullptr;
  }
  ...
}
```

| 函数 | 读的字段 | 关键差异 |
|---|---|---|
| `USER()` | **`m_main_security_ctx`** 的 `user()` + `host_or_ip()` | ★ **硬编码只读 main sctx**——即使在 DEFINER 存储过程里执行，它仍是登录者 |
| `CURRENT_USER()` | `ctx->priv_user()` + `ctx->priv_host()`，`ctx` 优先取 **name resolution context 的 `security_ctx`** | ★ **在 DEFINER 对象里会跟着变成 definer** |

### 七、proxy user：委托授权

登录路径（`sql_authentication.cc`）的关键几步：

```cpp
    sctx->assign_proxy_user("", 0);                     // ① 先清空
    ...
    if (is_proxy_user) {
      snprintf(proxy_user_buf, sizeof(proxy_user_buf) - 1, "'%s'@'%s'",
               auth_user, acl_user->host.get_host() ? ... : "");
      sctx->assign_proxy_user(proxy_user_buf, strlen(proxy_user_buf));  // ② 记原始身份
      acl_user = acl_proxy_user->copy(thd->mem_root);   // ③ ★换成被代理账号的 ACL_USER
      *(mpvio.restrictions) = acl_restrictions->find_restrictions(acl_user);
    }
    sctx->set_master_access(acl_user->access, *(mpvio.restrictions));   // ④
    assign_priv_user_host(sctx, const_cast<ACL_USER *>(acl_user));      // ⑤
```

于是：

| 字段 | proxy 时 |
|---|---|
| `m_user` | 客户端握手包里的用户名（**被代理前的外部用户**） |
| `m_proxy_user` | 带引号的 `'外部用户'@'外部host'` —— ★ **只用于审计/日志/错误报告，权限检查完全不用它** |
| `m_priv_user` / `m_priv_host` | **被代理账号**（proxied identity） |
| `m_master_access` / `m_restrictions` / default roles / `m_acl_map` | **全部来自被代理账号** |

★★ **两个故意的例外**——密码过期与锁定状态取的是**连接者**而非被代理者：

```cpp
      /*
        We are copying the connected user's password expired flag to the
        security context. This allows proxy user to execute queries even if
        proxied user password expires.
      */
      sctx->set_password_expired(mpvio.acl_user->password_expired || password_time_expired);
```

（`lock_account()` 同理，用的也是 `mpvio.acl_user->account_locked`。）

### 八、partial_revokes：sctx 如何消费 `m_restrictions`

```cpp
Access_bitmask Security_context::filter_access(Access_bitmask access,
                                               const std::string &db_name) const {
  Access_bitmask access_mask = access;
  auto &db_restrictions = m_restrictions.db();
  if (!db_restrictions.is_empty()) {
    Access_bitmask restrictions_mask;
    if (db_restrictions.find(db_name, restrictions_mask))
      access_mask = (access_mask & restrictions_mask) ^ access;
  }
  return access_mask;
}
```

位运算 `(access & restrictions_mask) ^ access` 的效果是**从 `access` 中剔除那些被部分回收的位**。

- 容器真名是 **`db_revocations = std::unordered_map<std::string, Access_bitmask>`**（★ 不存在 `Restrictions::Db_map` 这个类型）；
- **大小写敏感**（`std::hash<std::string>` + `equal_to`），`find("DB1")` 与 `find("db1")` 不互通；
- 懒分配（注释原文：*"Dynamically allocating the memory everytime in constructor would be expensive because this is frequently accessed code path."*）；
- 合并发生在 **role 图 BFS 的 visitor** 里，用 `Restrictions_aggregator` 以"用户自己的 restrictions 为 grantee 基线、逐个 role 作为 grantor"聚合，产物落在 `Acl_map::m_restrictions`，再由 `checkout_access_maps()` 灌进 sctx。

### 九、字段赋值时机总表

| 字段 | 赋值者 | 时机 |
|---|---|---|
| `m_user` / `m_host` / `m_ip` | `set_*_ptr()` | 登录解析握手包时 / `acl_getroot()` |
| `m_host_or_ip` | `set_host_or_ip_ptr()` | 同上（初值 `"connecting host"`） |
| `m_priv_user` / `m_priv_host` | `assign_priv_user()` / `assign_priv_host()` | 登录路径 ⑤ / `acl_getroot()` (E) |
| `m_proxy_user` | `assign_proxy_user()` | 登录时先清空，命中 proxy 才写 |
| `m_master_access` | `set_master_access(access[, restrictions])` | `acl_getroot`（无 role）、`checkout_access_maps()`（有 role，覆写）、`SET ROLE NONE` |
| `m_db_access` | `cache_current_db_access()` | `acl_getroot`、`mysql_change_db`、登录末尾置 0 |
| `m_password_expired` | `set_password_expired()` | 登录（取连接者）、`acl_getroot`（取 acl_user） |
| `m_is_locked` | `lock_account()` | 同上 |
| `m_active_roles` | `activate_role()` / `clear_active_roles()` | default roles、`SET ROLE`、`activate_all_roles_on_login` |
| `m_acl_map` | `checkout_access_maps()` | **仅在有 active role 时非 null** |
| `m_restrictions` | 经 `set_master_access(a, r)` | 同上 |
| `m_registration_sandbox_mode` | `set_registration_sandbox_mode()` | MFA 设备登记（`CR_OK_AUTH_IN_SANDBOX_MODE`） |

---

## 相关的系统变量/状态变量

| 变量 | 默认 | 作用域 | 说明 |
|---|---|---|---|
| `check_proxy_users` | OFF | Global | 是否按 `GRANT PROXY` 映射代理身份 |
| `mysql_native_password_proxy_users` | OFF | Global | 该插件是否参与 proxy 映射 |
| `sha256_password_proxy_users` | OFF | Global | 同上 |
| `activate_all_roles_on_login` | **OFF** | Global | 登录时激活**全部**已授予 role + mandatory roles（内部变量 `opt_always_activate_granted_roles`） |
| `mandatory_roles` | 空 | Global | 强制角色；★ **只在登录路径生效**，DEFINER 上下文拿不到 |
| `partial_revokes` | OFF | Global | 开启后改变 `db_acl()` 的匹配语义（见下） |

> 默认值已从 `sys_vars.cc` 核实。proxy 映射需要 `check_proxy_users=ON` 且对应插件开启 `*_proxy_users`。

---

## 可观测性

| 想看什么 | 怎么看 |
|---|---|
| 登录身份 vs 权限身份 | `SELECT USER(), CURRENT_USER();` |
| 当前活跃 role | `SELECT CURRENT_ROLE();`（对应 `m_active_roles`） |
| 我的全局权限 | `SHOW GRANTS;`（对应 `m_master_access` + `m_acl_map`） |
| 是否有某动态权限 | `SELECT * FROM information_schema.user_privileges` / 直接 `SHOW GRANTS`（`GRANT ... ON *.*` 部分） |
| 是否被 partial revoke | `SHOW GRANTS` 里出现 `REVOKE ... ON db.*` 段 |
| 密码是否过期 | 登录后执行任意语句报 `ER_MUST_CHANGE_PASSWORD`（`ERROR 1820`） |
| definer 上下文切换 | `SELECT CURRENT_USER();` 在 DEFINER 存储过程内会变成 definer |

★ 排查口诀：**`USER()` 说"谁连进来的"，`CURRENT_USER()` 说"按谁的权限算"**。两者不一致 ⇒ 命中了匿名用户或 proxy。

---

## Misc

### 易混淆概念

- **`m_user` vs `m_priv_user`**：登录用户（客户端报的名字）vs 权限用户（ACL 匹配到的账户名）。普通连接相同；匿名/proxy 时不同。
- **`m_host` vs `m_ip` vs `m_priv_host`**：客户端主机名（可能反解）vs 客户端 IP vs **权限判定用的 host**（ACL 记录里的，可能是 `%`）。
- **`m_user` 是 `String`，`m_priv_user` 是 `char[]`**：前者可变长（客户端任意输入），后者定长（会被静默截断），反映"登录名自由 vs 权限账户来自系统表"。
- **`m_db_access` 不是"所有库的权限"**：它只是**当前库**的缓存，其它库每次现算。
- **`checkout_access_maps()` 不总是借到东西**：没有 active role 时 `m_acl_map` 恒为 `nullptr`，整个判定退化到"老路径"。

### ★ 已知坑（源码证据逐条）

1. **★ `acl_getroot()` 只激活 default roles** —— DEFINER 上下文、复制线程、component sctx **拿不到 `mandatory_roles` 和"全部已授予 role"**，只有客户端登录连接才有（见[核心实现一](#一两条装配路径原文档的一大错误)）。
2. **`acl_getroot` 的自我批评**（源码注释原文）：
   > *"It's a terrible anti-pattern that needs to go."*
3. **`copy_security_ctx()` 故意不复制三个字段**：`m_acl_map = nullptr`（引用计数不能共享）、`m_has_drop_policy = false`、**以及 `m_is_locked` / `m_is_skip_grants_user` / `m_registration_sandbox_mode` 都没复制**。`COM_CHANGE_USER` 失败回滚用的是这个拷贝 ⇒ **回滚后 sandbox 标志会丢失**。
4. **★ `COM_CHANGE_USER` 失败回滚会丢掉 active roles**：
   ```cpp
   if (auth_rc) {
     *thd->security_context() = save_security_ctx;   // operator= 先 destroy() 再 copy
   ```
   `operator=` 先 `destroy()`（归还旧 `Acl_map`），而 `save_security_ctx.m_acl_map` 是 `nullptr` ⇒ 回滚后 `m_acl_map` 变 nullptr、**active roles 也被清空**，且恢复不回来。
5. **★ `db_acl()` 在 `partial_revokes` 开关下语义分裂**：
   ```cpp
      if (mysqld_partial_revokes()
              ? (my_strcasecmp(system_charset_info, db.str, it->first.c_str()) == 0)
              : (wild_compare(db.str, db.length, it->first.c_str(), ...) == 0))
   ```
   即：**开启 `partial_revokes` 会把通配符库权限（如 `GRANT ... ON db%.*`）从通配匹配退化成大小写不敏感精确比较**。同样的三元分支在 `acl_getroot` 扫 `acl_dbs` 时也出现一次。这是官方承认的行为变更。
6. **`checkout_access_maps()` OOM 时 fail-closed**：`set_master_access(0)` 静默降为无权限；且 `Acl_cache::checkout_acl_map` 里 OOM 会 **`active_roles.clear()`**（把用户的活跃 role 列表清掉）。
7. **`assign_priv_user` / `assign_priv_host` 静默截断**（`std::min(len, sizeof-1)`），超长账号可能因此变成同一个。
8. **`set_*_ptr()` 不复制内存**：注释明说 "user_arg value is *not* copied"，传栈缓冲会悬垂。
9. **`any_sp_acl()` / `any_table_acl()` 未判空 `m_acl_map`**（`db_acl()` 内部判了但下一行没判）——目前调用点都有 role 前置条件，属潜在 nullptr 解引用。
10. **hostname 比较的 @TODO**（`sql_auth_cache.cc`）：
    > *"@TODO This function should ideally only be called during authentication and not from authorization code. You may authenticate with a hostmask, but all authentication should be against a specific security context."*
    即 `compare_hostname()` 被授权路径（`acl_getroot`）调用了，违反设计意图。
11. **`ACL_USER contains too much information`**（`sql_auth_cache.h` 的 role 图顶点 TODO）：顶点直接存整个 `ACL_USER`，导致遍历 role 图必须持 ACL_USER 的锁（粒度粗）。
12. **`change_security_context()` 已被标记待替换**：注释说 *"The Security_context_factory should be used as a replacement to this function at every opportunity."* —— 但 8.0.39 里 `Security_context_factory` **尚未落地**。

### ★ 本篇核实出的"流传但不存在"的名字

| 常见写法 | 真实 |
|---|---|
| `m_apdb`（active partial revoked db） | **不存在**（全仓 0 命中）。"当前库权限"是 `m_db_access`，"部分回收"是 `m_restrictions` |
| `m_account_locked` | **不存在**；真名 **`m_is_locked`**（访问器 `account_is_locked()` / `lock_account(bool)`） |
| `m_is_system_user` / `m_is_connection_admin`（在 sctx 里） | **不在 sctx**；是 **THD** 的成员 |
| `return_access_maps()` | 真名 **`Acl_cache::return_acl_map()`**（在 Acl_cache 上，不是 sctx 方法） |
| `Restrictions::Db_map` | **不存在**；真名 `db_revocations`（`std::unordered_map`） |
| `acl_getroot()` 在 `sql_authentication.cc` | 在 **`sql_auth_cache.cc`** |
| 登录连接走 `acl_getroot()` | **不走**（见[核心实现一](#一两条装配路径原文档的一大错误)） |

---

## 参考

**源码**

- `sql/auth/sql_security_ctx.h` / `.cc`：类定义与全部方法、`change_security_context()`
- `sql/auth/sql_auth_cache.cc`：**`acl_getroot()`**、`Acl_map`、`Acl_cache::checkout/return/flush`、`initialized`
- `sql/auth/sql_authentication.cc`：登录装配主路径、proxy、`assign_priv_user_host`、`activate_all_roles_on_login`
- `sql/auth/sql_authorization.cc`：自由 `check_access`、`check_global_access`、`get_privilege_access_maps`、`activate_all_*_roles`
- `sql/auth/partial_revokes.h/.cc`：`Restrictions` / `DB_restrictions`
- `sql/auth/roles.cc`：`SET ROLE` 四种形态
- `sql/auth/auth_common.cc`：`set_system_user_flag()` / `set_connection_admin_flag()`
- `sql/sql_class.h/.cc`：`m_main_security_ctx` / `m_security_ctx` / `m_view_ctx_list`
- `sql/item_strfunc.cc`：`USER()` / `CURRENT_USER()`

**官方文档**

- MySQL 8.0 Reference Manual → [8.2.6 Access Control, Stage 1: Connection Verification](https://dev.mysql.com/doc/refman/8.0/en/connection-access.html)
- [8.2.10 Using Roles](https://dev.mysql.com/doc/refman/8.0/en/roles.html)
- [8.2.12 Using Partial Revokes](https://dev.mysql.com/doc/refman/8.0/en/partial-revokes.html)
- [8.2.19 Proxy Users](https://dev.mysql.com/doc/refman/8.0/en/proxy-users.html)

**相关文档**

- 认证过程（握手与插件）：[`authentication.md`](authentication.md)
- 判定链如何消费它：[`authorization.md`](authorization.md)
- definer 场景的临时切换：[`definer.md`](definer.md)
- MFA 与凭据存储：[`mfa.md`](mfa.md)、[`account_management.md`](account_management.md)
