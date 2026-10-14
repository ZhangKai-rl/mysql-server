# 账户管理深度解析

> 基于 MySQL 8.0.39 源码，剖析**账户管理**（Account Management）——官方 8.2 章里与"访问控制"并列的那件事：`CREATE / ALTER / DROP / RENAME USER`、`SET PASSWORD`，以及它们背后的密码子系统（强度校验、哈希生成、密码历史、双密码、过期）、资源限额、账户锁定、`REQUIRE SSL/X509` 约束，和**内存 ACL 缓存如何与磁盘保持一致**。
>
> **边界**：本篇讲**账户本身怎么建/改/限**（改 `mysql.user` 那一行）；改完之后权限如何被判定见 [`authorization.md`](authorization.md)；登录时怎么校验这些凭据见 [`authentication.md`](authentication.md)；`Security_context` 的字段语义见 [`security_context.md`](security_context.md)；多因素认证的**流程**见 [`authentication.md`](authentication.md)，其**凭据存储**见本篇。
>
> ★ 官方章名 *Access Control **and** Account Management* 本身就说明：账户管理与访问控制是**并列的两件事**。`CREATE USER` 属于账户管理，不属于授权（尽管它常常顺带 `GRANT`）。

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

账户管理 = 对 `mysql.user`（及其关联 grant 表）中"一个账户那一行"的增删改。四类语句：

| 语句 | 入口函数 | 文件 |
|---|---|---|
| `CREATE USER` / `CREATE ROLE` | `mysql_create_user()` | `sql/auth/sql_user.cc` |
| `ALTER USER` | `mysql_alter_user()` | 同上 |
| `DROP USER` / `DROP ROLE` | `mysql_drop_user()` | 同上 |
| `RENAME USER` | `mysql_rename_user()` | 同上 |
| `SET PASSWORD` | `change_password()` | 同上 |

> ★ `GRANT` / `REVOKE` **不在** `sql_user.cc`——它们在 `sql/auth/sql_authorization.cc` 的 `mysql_grant()`（REVOKE 是同一个函数的 `revoke_grant=true` 分支；`REVOKE ALL PRIVILEGES` 走 `mysql_revoke_all()`）。**不存在** `mysql_revoke()` 这个函数。

### 用途

回答三类实际问题：

1. **为什么这些 DDL 不需要 `FLUSH PRIVILEGES`，而直接 `UPDATE mysql.user` 需要？** → 见[核心实现七](#七缓存一致性为什么-grant-不需要-flush-privileges)
2. **密码策略、双密码、密码历史到底存在哪？** → 见[核心实现四](#四密码子系统)
3. **`DROP USER` 会不会踢掉已连接的会话？** → **不会**（见[核心实现六](#六drop--rename-USER最难的两个)）

### 版本演进

| 版本 | 变化 |
|---|---|
| 8.0 | `SET_USER_ID` 动态权限取代 `SUPER` 作为"可指定任意 definer"的权限 |
| 8.0.14 | 支持 `ALTER USER ... IDENTIFIED WITH plugin BY RANDOM PASSWORD` 之外的随机密码（`generated_random_password_length`） |
| 8.0.16 | partial revokes（`Restrictions` 进 `User_attributes`） |
| 8.0.19 | 账户失败锁定 `FAILED_LOGIN_ATTEMPTS` / `PASSWORD_LOCK_TIME` |
| 8.0.21 | 引入 `mysql.user.User_attributes` **JSON 列**（此后新增属性都往里塞，不再加列） |
| 8.0.27 | MFA（2/3 factor），凭据进 `User_attributes` |
| 8.0.34 | `mysql_native_password` deprecated；`validate_password` **组件**取代插件 |

---

## 理论基础

### 设计思想与权衡

**核心矛盾**：账户属性既要**持久化**（重启后还在），又要**低延迟可读**（每条 SQL 都要判权）。

**选定方案**：**磁盘（InnoDB 表）+ 内存 ACL 缓存双写**，且双写必须发生在**同一个临界区**内。代价是：
- 每次账户 DDL 都要拿一个全局排他锁（见下），**所有账户 DDL 串行**；
- 必须在写盘**之后**同步改内存，否则就要靠 `FLUSH PRIVILEGES` 兜底。

**★ 为什么 8.0 之后新增属性都塞进 `User_attributes` JSON 列，而不是加列**：

| 方案 | 优点 | 代价 |
|---|---|---|
| 每加一个属性就 `ALTER TABLE mysql.user ADD COLUMN` | 类型安全、可索引 | 每次都要改系统表结构，升级成本极高；`mysql.user` 列数已经 50+ |
| **JSON 列塞一切**（8.0.21+ 选定） | 加属性**不改表结构** | 失去类型约束与索引；解析失败要容错；未知 key 必须保留 |

证据：8.0.21 之后新增的 `Password_locking`、`additional_password`（双密码）、`multi_factor_authentication` 全在 JSON 里；而**更早**的 `password_last_changed`、`password_lifetime`、`Password_reuse_history` 仍是独立列。**判断一个属性在哪，看它引入的年份**。

**为什么账户 DDL 强制以 statement 格式记 binlog**：`Save_and_Restore_binlog_format_state` 强制 statement，即使 `binlog_format=ROW`。因为账户 DDL 改的是系统表，行事件传不动（且从库要用 `Q_INVOKER` 机制还原 grantor，见 [`definer.md`](definer.md)）。

### 理论溯源

写盘层是**模板方法 + 建造者**：`Acl_table_user_writer::driver()` 固定 11 步流水线，每步一个 `update_xxx()`；更新内容是**位掩码**（`Pod_user_what_to_update`），而不是"重新写整行"——这使得 `ALTER USER` 只改密码时不会误伤资源限额。

### 算法与数据结构

**① 子操作位掩码**（`sql/auth/auth_common.h` + `sql/auth/user_table.h`）

```cpp
#define NONE_ATTR 0L
#define DEFAULT_AUTH_ATTR (1L << 0)   /* update defaults auth */
#define PLUGIN_ATTR       (1L << 1)   /* plugin / authentication_string */
#define SSL_ATTR          (1L << 2)   /* SUBJECT / CIPHER / ... */
#define RESOURCE_ATTR     (1L << 3)   /* MAX_QUERIES_PER_HOUR ... */
#define PASSWORD_EXPIRE_ATTR (1L << 4)
#define ACCESS_RIGHTS_ATTR    (1L << 5)
#define ACCOUNT_LOCK_ATTR     (1L << 6)
#define DIFFERENT_PLUGIN_ATTR (1L << 7)
#define USER_ATTRIBUTES       (1L << 8)   /* 二级掩码，见下 */
```

```cpp
const ulong USER_ATTRIBUTE_NONE = 0L;
const ulong USER_ATTRIBUTE_RETAIN_PASSWORD  = (1L << 0);
const ulong USER_ATTRIBUTE_DISCARD_PASSWORD = (1L << 1);
const ulong USER_ATTRIBUTE_RESTRICTIONS     = (1L << 3);   // ← 1<<2 是空位，历史遗留
const ulong USER_ATTRIBUTE_FAILED_LOGIN_ATTEMPTS = (1L << 4);
const ulong USER_ATTRIBUTE_PASSWORD_LOCK_TIME    = (1L << 5);
const ulong USER_ATTRIBUTE_UPDATE_MFA            = (1L << 6);

class Pod_user_what_to_update {
 public:
  ulong m_what;
  ulong m_user_attributes;
};
```

★ **这个二级结构就是"为什么 `ALTER USER u IDENTIFIED BY 'x'` 不会把 `REQUIRE X509` 冲掉"的答案**：只有显式指定的位被置位，对应列才被写。

**② 资源限额结构**（`sql/sql_connect.h`）

```cpp
/*
  This structure specifies the maximum amount of resources which
  can be consumed by each account. Zero value of a member means
  there is no limit.
*/
typedef struct user_resources {
  uint questions;        /* MAX_QUERIES_PER_HOUR */
  uint updates;          /* MAX_UPDATES_PER_HOUR */
  uint conn_per_hour;    /* MAX_CONNECTIONS_PER_HOUR */
  uint user_conn;        /* MAX_USER_CONNECTIONS（并发数，不是速率） */
  enum { QUERIES_PER_HOUR = 1, UPDATES_PER_HOUR = 2,
         CONNECTIONS_PER_HOUR = 4, USER_CONNECTIONS = 8 };
  uint specified_limits;   /* 位掩码：本次到底指定了哪几个 */
} USER_RESOURCES;
```

**③ `User_attribute_type` 枚举**（`sql/auth/acl_table_user.h`）——JSON 列里的全部 key

```cpp
enum class User_attribute_type {
  ADDITIONAL_PASSWORD = 0,                 // 双密码的第二密码
  RESTRICTIONS,                            // partial revokes
  PASSWORD_LOCKING,                        // FAILED_LOGIN_ATTEMPTS / PASSWORD_LOCK_TIME
  METADATA,                                // ATTRIBUTE 子句
  COMMENT,                                 // COMMENT 子句
  MULTI_FACTOR_AUTHENTICATION_DATA         // MFA 的 2/3 factor
};
```

对应 JSON key：`additional_password`、`Restrictions`、`Password_locking`、`metadata`、`comment`、`multi_factor_authentication`。

---

## 核心实现

### 主链路

```
mysql_execute_command()                            [sql/sql_parse.cc]
 ├─ 权限校验：check_access(INSERT/DELETE/UPDATE_ACL,"mysql") && check_global_access(CREATE_USER_ACL)
 │     ★ 注意是 if (A && B) break; 的短路 → 实际语义是 "A 或 B"
 ├─ open_grant_tables()                            [sql/auth/sql_user_table.cc]
 │     └─ 10 张 ACL 表取 MDL + 打开 + 校验表结构完整（check_acl_tables_intact）
 ├─ Save_and_Restore_binlog_format_state binlog_format_state(thd)   ← 强制 statement
 └─ { 临界区
      Acl_cache_lock_guard(WRITE_MODE)             ← MDL_key::ACL_CACHE，MDL_EXCLUSIVE
      ├─ check_system_user_privilege()             ← 操作带 SYSTEM_USER 的账号需对等权限
      ├─ while (逐个 LEX_USER)
      │    ├─ set_and_validate_user_attributes()   ← 校验 + 生成 hash + 置位掩码
      │    ├─ handle_grant_data()                  ← 读/改各 grant 表（重复检测也在这）
      │    └─ replace_user_table()                 ← ★ 写 mysql.user + 回写内存缓存
      ├─ log_and_commit_acl_ddl()                  ← 改写 SQL → 写 binlog → 提交
      └─ acl_notify_htons()                        ← 通知存储引擎（NDB 等）
    }
```

**10 张 ACL 表**（`ACL_TABLES` 枚举，`sql/auth/sql_user_table.h`）：

```
TABLE_USER(mysql.user)  TABLE_DB(mysql.db)  TABLE_TABLES_PRIV  TABLE_COLUMNS_PRIV
TABLE_PROCS_PRIV  TABLE_PROXIES_PRIV  TABLE_ROLE_EDGES  TABLE_DEFAULT_ROLES
TABLE_DYNAMIC_PRIV(mysql.global_grants)  TABLE_PASSWORD_HISTORY
```

#### ★ 三层锁，缺一不可

| 层 | 机制 | 作用 |
|---|---|---|
| 表级 | `open_grant_tables()` → 10 张表 MDL + **IX Backup Lock** | 防并发 DDL 与备份冲突 |
| **ACL 缓存** | `Acl_cache_lock_guard(WRITE_MODE)` | **MDL**，不是 mutex |
| 复制 | `Save_and_Restore_binlog_format_state` | 强制 statement 格式 |

★ **`Acl_cache_lock_guard` 是 MDL 锁**（8.0 把 ACL 缓存锁从 `Partitioned_rwlock` 换成 MDL 的成果）：

```cpp
// sql/auth/sql_auth_cache.cc
static const ulong ACL_CACHE_LOCK_TIMEOUT = 3600UL;
static const MDL_key ACL_CACHE_KEY(MDL_key::ACL_CACHE, "", "");

bool Acl_cache_lock_guard::lock(bool raise_error) {
  MDL_request lock_request;
  MDL_REQUEST_INIT_BY_KEY(&lock_request, &ACL_CACHE_KEY,
      m_mode == Acl_cache_lock_mode::READ_MODE ? MDL_SHARED : MDL_EXCLUSIVE,
      MDL_EXPLICIT);
  Acl_cache_error_handler handler;      // 吞掉死锁/超时/中断
  m_thd->push_internal_handler(&handler);
  m_locked = !m_thd->mdl_context.acquire_lock(&lock_request, ACL_CACHE_LOCK_TIMEOUT);
  m_thd->pop_internal_handler();
  if (!m_locked && raise_error)
    my_error(ER_CANNOT_LOCK_USER_MANAGEMENT_CACHES, MYF(0));
  return m_locked;
}
```

即：**所有账户 DDL 在一个名为 `MDL_key::ACL_CACHE` 的专用 MDL namespace 上串行化**，写模式是 `MDL_EXCLUSIVE`，超时 3600 秒。下游函数开头普遍有 `assert_acl_cache_write_lock(thd)` 断言。

### 〇、`mysql.user` 表：列全谱与 `user_attributes` JSON

账户的一切属性最终都落在这张表上。先给列清单，后面所有 DDL 都只是在改这些列。

★ **表定义不在 `sql/dd/impl/tables/`**（那里只有 DD 自己的字典表，`user_table.cc` **不存在**）。真实三处：

| 位置 | 作用 |
|---|---|
| `scripts/mysql_system_tables.sql` | 权威 `CREATE TABLE`（含类型与默认值） |
| `sql/dd/impl/system_registry.cc` | `register_table("user", system);`（同处还注册 `password_history` / `role_edges` / `default_roles`） |
| `sql/auth/auth_common.h` | 列序枚举 `enum mysql_user_table_field`（`sql_user_table.cc` 用它做完整性校验） |

**除 32 个 `*_priv` 列外的"属性列"**（`*_priv` 见 [`authorization.md`](authorization.md)）：

| 列 | 类型 / 默认 | 说明 |
|---|---|---|
| `plugin` | `char(64)` / `'caching_sha2_password'` | 认证插件 |
| `authentication_string` | `TEXT` | 凭据（hash / `$A$005$...`） |
| `password_expired` | `ENUM('N','Y')` / `'N'` | 硬过期标志 |
| `password_last_changed` | `timestamp NULL` | 过期判定的起点 |
| `password_lifetime` | `smallint NULL` | NULL = 用全局 `default_password_lifetime` |
| `account_locked` | `ENUM('N','Y')` / `'N'` | ★ role 的本质就是 `'Y'` |
| `ssl_type` | `ENUM('','ANY','X509','SPECIFIED')` | 空串 ↔ `SSL_TYPE_NONE` |
| `ssl_cipher` / `x509_issuer` / `x509_subject` | `BLOB` | `REQUIRE` 的三项约束 |
| `max_questions` / `max_updates` / `max_connections` / `max_user_connections` | `int unsigned` / `0` | 资源限额（★ 第三列名是 `max_connections` 不是 `max_connections_per_hour`） |
| `Password_reuse_history` / `Password_reuse_time` | `smallint NULL` | ★ **仍在列里**，未移入 JSON |
| `Password_require_current` | `ENUM('N','Y')` / `NULL` | 三态：NULL = 用全局 |
| **`User_attributes`** | **`JSON NULL`** | 见下 |

★ **没有 `Password` 列**（5.7 已移除）；`password_idx()` 直接 `assert(0)`，只有 5.6 兼容枚举 `User_table_old_schema::mysql_user_table_field_56` 里还留着。★ `Create_role_priv` / `Drop_role_priv` 是**真实列**，不是动态权限（`get_user_privileges()` 读回来拼成 `CREATE_ROLE_ACL` / `DROP_ROLE_ACL`）。

**密码历史另存独立表** `mysql.password_history`（`Host, User, Password_timestamp, Password`，主键含 `Password_timestamp DESC`）——`user_attributes` 里**没有** `password_history` 这个 key。

#### ★ `user_attributes` JSON 的 key 全表

★ 处理类的真名是 **`acl_table::Acl_user_attributes`**（`sql/auth/acl_table_user.h`）——不是 `User_attributes`（那是 `dd::system_views` 下的 **I_S 视图类**，同名易混）。key 字符串逐个核实：

| key | 值类型 | 含义 | 对应位标记 |
|---|---|---|---|
| `additional_password` | string | **第二口令**（`RETAIN CURRENT PASSWORD`） | `USER_ATTRIBUTE_RETAIN_PASSWORD` / `..._DISCARD_PASSWORD` |
| `Restrictions` | array | partial_revokes（部分回收） | `USER_ATTRIBUTE_RESTRICTIONS` |
| `Password_locking` | object | 登录失败锁定 | `USER_ATTRIBUTE_FAILED_LOGIN_ATTEMPTS` / `..._PASSWORD_LOCK_TIME` |
| └ `failed_login_attempts` | int | 阈值（`0..UINT_MAX`） | |
| └ `password_lock_time_days` | int | 锁定时长（`**-1 = 永久**`，范围 `-1..INT_MAX`） | |
| `metadata` | object | 应用自定义元数据 | |
| └ `comment` | string | `ALTER USER ... COMMENT` | |
| `multi_factor_authentication` | array | 2/3 因素凭据 | `USER_ATTRIBUTE_UPDATE_MFA` |

★ 两个易错点：

1. **MFA 的 key 是 `multi_factor_authentication`**（不是 `multi_factor_auth_data`）。
2. **序列化时"空值即 `remove()`"**——手工往顶层塞的自定义键会在**下一次 `ALTER USER` 时被丢弃**；只有 `metadata` 子对象用 `merge_patch()` 合并保留（所以自定义数据必须放 `ATTRIBUTE`，别放 `COMMENT` 之外的顶层）。

写入入口是 `Acl_table_user_writer::update_user_attributes()`（`acl_table_user.cc`）；`ALTER USER` 最终落盘分支的**权威顺序**在同一个文件的 `driver()`：

```cpp
  if (update_authentication_info(return_value) ||
      update_privileges(return_value) || update_ssl_properties() ||
      update_user_attributes(current_password, return_value) ||
      update_user_resources() || update_password_expiry() ||
      update_password_history() || update_password_reuse() ||
      update_password_require_current() || update_account_locking() ||
      update_user_application_user_metadata()) { ... }
```

### 一、`CREATE USER`：从语法到落盘

权限判定**不在** `mysql_create_user()` 里，而在 `sql_parse.cc`：

```cpp
    case SQLCOM_CREATE_USER: {
      if (check_access(thd, INSERT_ACL, "mysql", nullptr, nullptr, true, true) &&
          check_global_access(thd, CREATE_USER_ACL))
        break;
```

| 语句 | 所需权限（"或"关系） |
|---|---|
| `CREATE USER` | `INSERT` on `mysql` ‖ `CREATE_USER` |
| `DROP USER` | `DELETE` on `mysql` ‖ `CREATE_USER` |
| `RENAME USER` | `UPDATE` on `mysql` ‖ `CREATE_USER` |
| `ALTER USER` | `UPDATE` on `mysql` ‖ `CREATE_USER`（但**改自己密码有豁免**，见本章二） |

额外两道校验（在函数内部）：
- `check_orphaned_definers()` —— 若目标 authid 是某 view/trigger/event/routine 的 DEFINER，需 `SET_USER_ID`
- `check_system_user_privilege()` —— 若目标账号有 `SYSTEM_USER`，操作者必须也有

#### 用户名/host 校验在哪

**不在** `mysql_create_user()` 里，而在构造 `LEX_USER` 时（`sql/table.cc`）：

```cpp
LEX_USER *LEX_USER::init(LEX_USER *ret, THD *thd, LEX_STRING *user_arg,
                         LEX_STRING *host_arg) {
  ret->init();
  trim_whitespace(system_charset_info, user_arg);
  if (host_arg) trim_whitespace(system_charset_info, host_arg);
  ret->user.str = user_arg->str;
  ret->host.str = host_arg ? host_arg->str : "%";        // ← 默认 host 是 %
  ...
  if (check_string_char_length(ret->user, ER_THD(thd, ER_USERNAME),
                               USERNAME_CHAR_LENGTH, system_charset_info, false) ||
      (host_arg && check_host_name(ret->host)))
    return nullptr;
  if (host_arg) {
    my_casedn_str(system_charset_info, host_arg->str);   // ★ host 就地转小写
    ret->host.str = host_arg->str;
  }
  return ret;
}
```

`check_host_name()`（`sql/sql_parse.cc`）校验长度 ≤ `HOSTNAME_LENGTH` 且**禁止含 `@'`：

```cpp
bool check_host_name(const LEX_CSTRING &str) {
  ...
  if (check_string_byte_length(str, ER_THD(current_thd, ER_HOSTNAME), HOSTNAME_LENGTH))
    return true;
  while (name != end) {
    if (*name == '@') {
      my_printf_error(ER_UNKNOWN_ERROR,
                      "Malformed hostname (illegal symbol: '%c')", MYF(0), *name);
      return true;
    }
    name++;
  }
  return false;
}
```

#### ★ 凭据如何落盘：统一走插件的 `generate_authentication_string`

**不存在** `create_user_from_lex()` 这个函数。真实链路是 `mysql_create_user` → `set_and_validate_user_attributes()` → `replace_user_table()`。hash 生成由插件描述符 `st_mysql_auth` 的 `generate_authentication_string` 回调完成：

```cpp
  st_mysql_auth *auth = (st_mysql_auth *)plugin_decl(plugin)->info;
  ...
  if (auth->generate_authentication_string(outbuf, &buflen, inbuf, inbuflen) ||
      auth_verify_password_history(...)) {
    my_error(ER_CANNOT_USER, MYF(0), cmd, error_user.c_ptr_safe());
    return true;
  }
  ...
  /* 擦除内存中的明文密码（除非要作为结果集回传客户端） */
  if (Str->first_factor_auth_info.auth.length > 0)
    my_cleanse(const_cast<char *>(Str->first_factor_auth_info.auth.str),
               Str->first_factor_auth_info.auth.length);
  Str->first_factor_auth_info.auth = {password, buflen};   // 换成 hash
```

**native 的实现**（函数名是 `generate_native_password`，不是 `make_scrambled_password`）：

```cpp
static int generate_native_password(char *outbuf, unsigned int *buflen,
                                    const char *inbuf, unsigned int inbuflen) {
  THD *thd = current_thd;
  native_password_authentication_deprecation_warning();
  if (!thd->m_disable_password_validation)
    if (my_validate_password_policy(inbuf, inbuflen)) return 1;
  if (inbuflen == 0) { *buflen = 0; return 0; }         // 空密码
  ...
  my_make_scrambled_password_sha1(buffer, inbuf, inbuflen);
  *buflen = SCRAMBLED_PASSWORD_CHAR_LENGTH;             // 41
```

产出 `'*' + HEX(SHA1(SHA1(pwd)))`，41 字节。

**caching_sha2 的实现**：

```cpp
static int caching_sha2_password_generate(char *outbuf, unsigned int *buflen,
                                          const char *inbuf, unsigned int inbuflen) {
  if (inbuflen > sha2_password::CACHING_SHA2_PASSWORD_MAX_PASSWORD_LENGTH) return 1;   // 256
  if (!thd->m_disable_password_validation)
    if (my_validate_password_policy(inbuf, inbuflen)) return 1;
  if (inbuflen == 0) { *buflen = 0; return 0; }

  char salt[sha2_password::SALT_LENGTH + 1];            // 20 + 1
  generate_user_salt(salt, sha2_password::SALT_LENGTH + 1);
  random.assign(salt, sha2_password::SALT_LENGTH);

  if (g_caching_sha2_password->generate_sha2_multi_hash(
          source, random, &digest, g_caching_sha2_password->get_digest_rounds())) return 1;
  if (g_caching_sha2_password->serialize(serialized_string,
          g_caching_sha2_password->get_digest_type(), random, digest,
          g_caching_sha2_password->get_digest_rounds())) return 1;
```

产出格式（源码注释原文）：

```
DELIMITER[digest_type]DELIMITER[iterations]DELIMITER[salt][digest]
digest_type: A => SHA256
iterations:  5000 => 005      (iterations / 1000，3 位十六进制)
salt:        Random string. Length CRYPT_SALT_LENGTH   (20)
digest:      SHA2 digest. Length STORED_SHA256_DIGEST_LENGTH  (43)
```

即 `$A$005<20B salt><43B digest>`。这与 [`authentication.md`](authentication.md) 里剖析的认证流程是同一份数据的两面。

#### 写哪一列

`Acl_table_user_writer::update_authentication_info()`（`sql/auth/acl_table_user.cc`）一次写 4 个列：

| 列 | 值 |
|---|---|
| `plugin` | 插件名 |
| `authentication_string` | 上面的 hash（**只存主密码**） |
| `password_last_changed` | ★ **仅 built-in 插件**才写；外部插件该列为 NULL |
| `password_expired` | 给了密码就写 `'N'` |

### 二、`ALTER USER`：位掩码驱动的子操作矩阵

`mysql_alter_user()` 与 `mysql_create_user()` 结构几乎相同，差别只有两点：
1. `replace_user_table(..., can_create_user = false)` —— 所以 ALTER 不存在的用户会失败；
2. 子操作由 `what_to_alter` 位掩码驱动，分支极多。

#### 子操作全景（每一项的"语法 → 位 → 写盘函数 → 列"）

| 子操作 | 位 | 写盘函数 | 落到 `mysql.user` 的哪 |
|---|---|---|---|
| 改密码 / 改插件 | `PLUGIN_ATTR` | `update_authentication_info()` | `plugin`、`authentication_string`、`password_last_changed`、`password_expired` |
| 资源限额 | `RESOURCE_ATTR` | `update_user_resources()` | `max_questions` / `max_updates` / `max_connections` / `max_user_connections` |
| 锁定 / 解锁 | `ACCOUNT_LOCK_ATTR` | `update_account_locking()` | `account_locked` ENUM('N','Y') |
| 密码过期 | `PASSWORD_EXPIRE_ATTR` | `update_password_expiry()` | `password_expired`、`password_lifetime`（NULL=用全局默认） |
| **SSL 要求** | `SSL_ATTR` | `update_ssl_properties()` | `ssl_type`、`ssl_cipher`、`x509_issuer`、`x509_subject` |
| 失败锁定次数 | `USER_ATTRIBUTES`+子位 | `update_user_attributes()` | `User_attributes` JSON |
| MFA factor | `USER_ATTRIBUTES`+子位 | 同上 | `User_attributes` JSON |
| 双密码 | `USER_ATTRIBUTES`+子位 | 同上 | `User_attributes` JSON |
| COMMENT / ATTRIBUTE | `USER_ATTRIBUTES` | `update_user_application_user_metadata()` | `User_attributes` JSON |
| PASSWORD REQUIRE CURRENT | —（无条件） | `update_password_require_current()` | `Password_require_current` |
| PASSWORD HISTORY / REUSE | —（无条件） | `update_password_history()` / `update_password_reuse()` | `Password_reuse_history` / `Password_reuse_time` |

**11 步写盘流水线**（`Acl_table_user_writer::driver()`）：

```cpp
  if (setup_table(error, builtin_plugin)) { ... }
  if (update_authentication_info(return_value) ||     // ①
      update_privileges(return_value) ||              // ②
      update_ssl_properties() ||                      // ③
      update_user_attributes(current_password, return_value) ||  // ④ JSON
      update_user_resources() ||                      // ⑤
      update_password_expiry() ||                     // ⑥
      update_password_history() ||                    // ⑦
      update_password_reuse() ||                      // ⑧
      update_password_require_current() ||            // ⑨
      update_account_locking() ||                     // ⑩
      update_user_application_user_metadata()) {      // ⑪
    return err_return_value;
  }
  (void)finish_operation(error);      // ha_write_row 或 ha_update_row
```

#### ★ 改自己密码的豁免（判定在 `sql_parse.cc`，不在 `mysql_alter_user`）

```cpp
        if (user->first_factor_auth_info.uses_identified_by_clause &&
            !user->first_factor_auth_info.uses_identified_with_clause &&
            !thd->lex->mqh.specified_limits &&
            !user->alter_status.update_account_locked_column &&
            !user->alter_status.update_password_expired_column &&
            !user->alter_status.expire_after_days &&
            user->alter_status.use_default_password_lifetime &&
            (user->alter_status.update_password_require_current == Lex_acl_attrib_udyn::UNCHANGED) &&
            !user->alter_status.update_password_history &&
            !user->alter_status.update_password_reuse_interval &&
            !user->alter_status.update_failed_login_attempts &&
            !user->alter_status.update_password_lock_time &&
            (thd->lex->ssl_type == SSL_TYPE_NOT_SPECIFIED))
          update_password_only = true;

        is_self = ...;
        if ((update_password_only || ...) && is_self) {
          changing_own_password = update_password_only;
          continue;                      // ← 跳过权限检查
        } else if (check_permission) { ... }
```

即：**`ALTER USER u IDENTIFIED BY 'x'`（不带 `IDENTIFIED WITH`），且没带任何其他子操作，且改的是自己 → 完全不需要任何权限**。一旦带上 `IDENTIFIED WITH` / SSL / 资源 / 锁定 / 过期 任何一项，就要 `UPDATE` on `mysql` 或 `CREATE_USER`。

★ 双密码是例外：**即使改自己**，`RETAIN CURRENT PASSWORD` / `DISCARD OLD PASSWORD` 也需要 `CREATE_USER` 或 **`APPLICATION_PASSWORD_ADMIN`**。

#### `IDENTIFIED BY` vs `IDENTIFIED WITH plugin BY` vs `AS`

| 写法 | 走的分支 | 结果 |
|---|---|---|
| `IDENTIFIED BY 'pw'` | 不指定插件 → 取 `authentication_policy[0]`（新建）或现有插件（ALTER） | 用**该插件**算法生成 hash |
| `IDENTIFIED WITH p BY 'pw'` | 两者都置位 | 用**指定插件**算法生成 hash |
| **`IDENTIFIED WITH p`**（不给 BY/AS） | 只有 `uses_identified_with_clause` | **不生成 hash**；且若新插件支持过期，会**强制 `password_expired='Y'`** |
| `IDENTIFIED WITH p AS '<hash>'` | `uses_authentication_string_clause` | **校验**已有 hash（`validate_authentication_string`），不合格报 `ER_PASSWORD_FORMAT` |

★ 换插件时还会**清空该用户的密码历史**：

```cpp
      /* purge the password history if plugin is different */
      if ((what_to_alter.m_what & DIFFERENT_PLUGIN_ATTR) &&
          handle_password_history_table(thd, tables, true, user_from, nullptr,
                                        &dummy_row_existed)) { result = 1; break; }
```

### 三、★ `REQUIRE SSL / X509 / SUBJECT / ISSUER / CIPHER` 完整链路

这条链**横跨账户管理与认证**，是两者唯一的接缝，逐步拆开：

**Step 1 — 语法**（`sql/sql_yacc.yy`）

```
require_clause:
          %empty
        | REQUIRE_SYM require_list     { Lex->ssl_type = SSL_TYPE_SPECIFIED; }
        | REQUIRE_SYM SSL_SYM          { Lex->ssl_type = SSL_TYPE_ANY; }
        | REQUIRE_SYM X509_SYM         { Lex->ssl_type = SSL_TYPE_X509; }
        | REQUIRE_SYM NONE_SYM         { Lex->ssl_type = SSL_TYPE_NONE; }
        ;
require_list_element:
          SUBJECT_SYM TEXT_STRING   { ... lex->x509_subject = $2.str; }
        | ISSUER_SYM  TEXT_STRING   { ... lex->x509_issuer  = $2.str; }
        | CIPHER_SYM  TEXT_STRING   { ... lex->ssl_cipher   = $2.str; }
        ;
```

`enum SSL_type` 定义在 **`include/violite.h`**（不在 `sql_lex.h`，那里只是 `enum SSL_type ssl_type;` 并注释 "defined in violite.h"）。`require_clause` 只被 `CREATE USER` 与 `ALTER USER` 两个产生式引用 → **只有这两种语句能设**。

**Step 2 — 置位**

```cpp
  if (thd->lex->ssl_type != SSL_TYPE_NOT_SPECIFIED)
    what_to_set.m_what |= SSL_ATTR;
```

**Step 3 — 写 4 个列**（`update_ssl_properties()`）

```cpp
bool Acl_table_user_writer::update_ssl_properties() {
  if (m_what_to_update.m_what & SSL_ATTR) {
    LEX *lex = m_thd->lex;
    switch (lex->ssl_type) {
      case SSL_TYPE_ANY:
        store(ssl_type, "ANY");  cipher=""; issuer=""; subject="";  break;
      case SSL_TYPE_X509:
        store(ssl_type, "X509"); ... 同上清空 ...
        break;
      case SSL_TYPE_SPECIFIED: {
        store(ssl_type, "SPECIFIED");
        if (lex->ssl_cipher)  store(ssl_cipher,  lex->ssl_cipher);
        if (lex->x509_issuer) store(x509_issuer, lex->x509_issuer);
        if (lex->x509_subject)store(x509_subject,lex->x509_subject);
        break;
      }
      case SSL_TYPE_NONE:
        /* 四个列全写空串 */
        break;
```

列定义（`sql_user_table.cc`）：

```cpp
  {{STRING_WITH_LEN("ssl_type")},     {STRING_WITH_LEN("enum('','ANY','X509','SPECIFIED')")}, ...},
  {{STRING_WITH_LEN("ssl_cipher")},   {STRING_WITH_LEN("blob")}, ...},
  {{STRING_WITH_LEN("x509_issuer")},  {STRING_WITH_LEN("blob")}, ...},
  {{STRING_WITH_LEN("x509_subject")}, {STRING_WITH_LEN("blob")}, ...},
```

**Step 4 — 回写内存 `ACL_USER`**

`replace_user_table()` 尾部按"行是否已存在"分流：

```cpp
    if (old_row_exists)
      acl_update_user(combo->user.str, combo->host.str, lex->ssl_type,
                      lex->ssl_cipher, lex->x509_issuer, lex->x509_subject, ...);
    else
      acl_insert_user(thd, combo->user.str, combo->host.str, lex->ssl_type, ...);
```

`ACL_USER` 上的字段（`sql/auth/sql_auth_cache.h`）：

```cpp
  enum SSL_type ssl_type;
  const char *ssl_cipher, *x509_issuer, *x509_subject;
```

★ 注意默认值差异：新建时 `ssl_type = (ssl_type != SSL_TYPE_NOT_SPECIFIED ? ssl_type : SSL_TYPE_NONE)`，而表里存空串 `''` —— **`''` 与 `SSL_TYPE_NONE` 是同一个枚举值**。

**Step 5 — 认证时比对**（`acl_check_ssl()`，详见 [`authentication.md`](authentication.md)）

```cpp
    case SSL_TYPE_ANY:  return vio_type(vio) != VIO_TYPE_SSL;      // 只要是 SSL 就行
    case SSL_TYPE_X509: /* 任意合法证书 */
      return !(vio_type(vio) == VIO_TYPE_SSL &&
               SSL_get_verify_result(ssl) == X509_V_OK &&
               SSL_get_peer_certificate(ssl));
    case SSL_TYPE_SPECIFIED:
      if (vio_type(vio) != VIO_TYPE_SSL || SSL_get_verify_result(ssl) != X509_V_OK) return true;
      if (acl_user->ssl_cipher  && strcmp(acl_user->ssl_cipher,  SSL_get_cipher(ssl))) return true;
      ... X509_NAME_oneline(issuer/subject) 与存的值做 strcmp ...
```

★ **比较的是纯字符串**（`X509_NAME_oneline()` 的输出），所以 `SHOW CREATE USER` 里那一长串必须**逐字节一致**。校验用的 CA 来自 `ssl_ca`（默认就是自动生成的 `ca.pem`）。

> 客户端证书是**客户端自己带**的，服务端从不发。见 [`authentication.md`](authentication.md) 的 pem 章节。

### 四、密码子系统

#### 4.1 `validate_password`：8.0 里组件与插件**双份源码并存**

| 形态 | 目录 | 状态 |
|---|---|---|
| **组件**（推荐） | `components/validate_password/` | `INSTALL COMPONENT 'file://component_validate_password'` |
| **插件**（deprecated） | `plugin/password_validation/` | `push_deprecated_warn(..., "validate_password plugin", "validate_password component")` |

桥接层 `sql/auth/password_policy_service.cc` **组件优先、插件回退**：

```cpp
int my_validate_password_policy(const char *password, unsigned int password_len) {
  if (!srv_registry->acquire("validate_password", &h_pv_svc)) {
    ret = reinterpret_cast<SERVICE_TYPE(validate_password) *>(h_pv_svc);
    if (ret->validate((void *)current_thd, (my_h_string)&password_str)) {
      my_error(ER_NOT_VALID_PASSWORD, MYF(0));
      res = 1;
    }
    srv_registry->release(h_pv_svc);
  } else {
    /* 回退到插件 */
    plugin = my_plugin_lock_by_name(nullptr, validate_password_plugin, ...);
```

组件提供**两个服务**：`validate_password`（`validate` / `get_strength`）与 `validate_password_changed_characters`。

**变量对照**（★ 命名差异常踩坑）：

| 组件名 | 插件名 | 默认值 |
|---|---|---|
| `validate_password.length` | `validate_password_length` | **8** |
| `validate_password.number_count` | `_number_count` | **1** |
| `validate_password.mixed_case_count` | `_mixed_case_count` | **1** |
| `validate_password.special_char_count` | `_special_char_count` | **1** |
| `validate_password.policy` | `_policy` | **MEDIUM**（LOW=0/MEDIUM=1/STRONG=2） |
| `validate_password.dictionary_file` | `_dictionary_file` | NULL |
| `validate_password.check_user_name` | `_check_user_name` | **true** |
| `validate_password.changed_characters_percentage` | **（插件没有）** | **0** |

校验核心（字符分类：非 lower/upper/digit 一律算 special）：

```cpp
  if (n_chars >= validate_password_length) {
    if (!is_valid_password_by_user_name(thd, password)) return 0;
    if (policy == PASSWORD_POLICY_LOW) return 1;
    if (has_upper >= mixed_case_count && has_lower >= mixed_case_count &&
        has_special_chars >= special_char_count && has_digit >= number_count) {
      if (policy == PASSWORD_POLICY_MEDIUM || validate_dictionary_check(password))
        return 1;
    }
  }
  return 0;
```

字典匹配是**子串 + 大小写不敏感**：密码先 `tolower`，再对所有长度 ≥ 4 的子串查 `std::set`。

#### 4.2 `RANDOM PASSWORD`

```cpp
void generate_random_password(std::string *password, uint32_t length) {
  unsigned char buffer[256];
  if (length > 255) length = 255;
  RAND_bytes((unsigned char *)&buffer[0], length);      // ★ OpenSSL RAND_bytes
  password->reserve(length + 1);
  for (uint32_t i = 0; i < length; ++i)
    password->append(1, translate_byte_to_password_char(buffer[i]));
}
```

字符集 71 个可见字符（`translate_byte_to_password_char`），**刻意避开空白/引号/反斜杠**（不破坏 SQL 字面量），但**没避开易混淆字符**（`0/O`、`1/l/I` 都在）。长度由 `generated_random_password_length` 控制，默认 **20**，SESSION 级，范围 5..255。

★ **随机密码会绕过强度校验**：`thd->m_disable_password_validation = true`。官方 8.2.15 也明说：
> *"If the `validate_password` component is installed, the policy that it implements has no effect on generated passwords. (The purpose of password validation is to help humans create better passwords.)"*

另外补一条本篇前面没展开的：`PASSWORD REQUIRE CURRENT`（8.0.13+）要求改密时必须提供**当前**密码，即 `ALTER USER u IDENTIFIED BY 'new' REPLACE 'old'`。落盘列 `Password_require_current`，全局开关 `password_require_current` 默认 **OFF**。★ `REPLACE` 子句**不会写进 binlog**（避免明文密码落盘），与前面"明文密码不落 binlog"是同一套 `mysql_rewrite_acl_query()` 改写机制。

#### ★ 判定点：`validate_password_require_current()`

判定函数是 **`validate_password_require_current()`**（`sql_user.cc`，static），**唯一调用点**在 `set_and_validate_user_attributes()` 里、且带 `PLUGIN_ATTR` 前置条件——即**只有真正要改口令时才校验**：

```cpp
  if ((what_to_set.m_what & PLUGIN_ATTR) &&
      validate_password_require_current(thd, Str, acl_user, auth, new_password,
                                        new_password_length, is_privileged_user,
                                        user_exists)) { ... }
```

核心分支（`Lex_acl_attrib_udyn` 三态：`DEFAULT`(NULL) / `YES` / `NO`）：

```cpp
    } else if (!is_privileged_user) {
      /*
        If the field value is set or field value is NULL and global sys
        variable flag is ON then REPLACE clause must be specified.
      */
      if ((acl_user->password_require_current == Lex_acl_attrib_udyn::YES) ||
          (acl_user->password_require_current == Lex_acl_attrib_udyn::DEFAULT &&
           password_require_current)) {
        my_error(ER_MISSING_CURRENT_PASSWORD, MYF(0));
        return (true);
      }
    }
```

★ 三个要点：

1. **三态而非布尔**：账号列为 `NULL`（DEFAULT）时听全局开关；显式 `YES` 则不管全局；显式 `NO` 则强制不要。这让"个别账号例外"成为可能。
2. **`is_privileged_user` 豁免**：有 `CREATE USER` / `UPDATE` on `mysql` 权限的人给别人改密**不需要**旧密码（否则 DBA 无法重置）。
3. 函数里还有一处 `create_scope_guard` + `my_cleanse()`——**旧密码与新密码用完立即擦除内存**，这是明文口令在内存中的生命周期管理。

相关错误码：`ER_MISSING_CURRENT_PASSWORD`（该给没给）、`ER_CURRENT_PASSWORD_NOT_REQUIRED`（给别人改却给了 REPLACE）、`ER_INCORRECT_CURRENT_PASSWORD`（给了但不对）。

#### 4.3 密码历史 `mysql.password_history`

> ★ 不存在 `acl_table_password_history.cc` 这个文件。相关代码在 `sql_user.cc`（`auth_verify_password_history()`）+ `acl_table_user.cc` + `sql_user_table.cc`。

表结构（4 列，**复合主键带 DESC**）：

```sql
CREATE TABLE password_history (
  Host CHAR(255) ... NOT NULL,
  User CHAR(32) BINARY ... NOT NULL,
  Password_timestamp TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  Password TEXT,
  PRIMARY KEY(Host, User, Password_timestamp DESC)
) engine=InnoDB ... comment='Password history for user accounts'
```

★ `Password_timestamp DESC` 让主键索引天然"新→旧"，正好配合下面的裁剪。

**写入时机**：改密码生成了新 hash **且** `password_history` 或 `password_reuse_interval` 非 0 **且**插件是 `AUTH_FLAG_USES_INTERNAL_STORAGE`。

**裁剪算法是"边扫边删"**（源码注释即伪代码）：

```
count = 0;
FOR SELECT * FROM password_history WHERE user/host 按 TS DESC {
   if (count >= password_history && (NOW() - ts) > password_reuse_time) { delete row; continue; }
   if (CRED was produced by ::password) signal("wrong password");
   count++;
}
INSERT 新行;
```

| 变量 | 作用域 | 默认 | 落盘 |
|---|---|---|---|
| `password_history` | GLOBAL | **0**（关） | — |
| `password_reuse_interval` | GLOBAL | **0**（关） | — |
| `PASSWORD HISTORY n` | 每账户 | 用全局 | `Password_reuse_history` 列（smallint） |
| `PASSWORD REUSE INTERVAL n DAY` | 每账户 | 用全局 | `Password_reuse_time` 列（smallint） |

★ 每账户值**用 NULL 表示"用全局默认"**（`set_null()`），读回时 NULL → `use_default_password_* = true`。

#### 4.4 双密码（dual password）：★ 存在 JSON 里，不在 `authentication_string`

```sql
ALTER USER u IDENTIFIED BY 'new' RETAIN CURRENT PASSWORD;
ALTER USER u DISCARD OLD PASSWORD;
```

存储位置：`mysql.user.authentication_string` **只存主密码**；第二密码在 `User_attributes` JSON 的 `additional_password` key。内存侧是 `ACL_USER::credentials[NUM_CREDENTIALS]`：

```cpp
#define NUM_CREDENTIALS 2
#define PRIMARY_CRED 0
#define SECOND_CRED 1
  Acl_credential credentials[NUM_CREDENTIALS];
```

认证时 native 先试主、失败再试从；caching_sha2 直接循环 `MAX_PASSWORDS`。

★★ **最反直觉的一点：旧密码不会自动失效**。源码里 `USER_ATTRIBUTE_DISCARD_PASSWORD` 只在三种情况下被置位：① 换认证插件；② 显式 `DISCARD OLD PASSWORD`；③ 认证串被置空。**再次 `RETAIN` 之外的普通改密不会丢弃旧密码**。

**两条独立证据**：

1. 源码：MTR 用例 `multiple_passwords.inc` 明确验证了"改密后旧密码仍可登录，直到显式 DISCARD"。
2. ★ 官方 8.2.15 原文：
   > *"If an account has a secondary password and you change its primary password without specifying `RETAIN CURRENT PASSWORD`, the secondary password remains unchanged."*

双密码的设计目的官方也讲得很清楚——**无停机轮换凭据**：先在各主库设新主密码并保留旧密码为第二密码 → 等复制传播 → 改应用连新密码 → 全部迁完后再 `DISCARD OLD PASSWORD`。所以它**故意**不会自动失效。

#### 4.5 密码过期

| 项 | 内存字段 | 列 |
|---|---|---|
| 强制过期 | `bool password_expired` | `password_expired` ENUM('N','Y') |
| 上次改密 | `MYSQL_TIME password_last_changed` | `password_last_changed` timestamp（★ **仅 built-in 插件**才写） |
| 每账户生存期 | `uint password_lifetime` | `password_lifetime` smallint，**NULL = 用全局默认** |
| 是否用全局 | `bool use_default_password_lifetime` | —（由上面那列是否 NULL 表达） |

★ 判定函数真名是 **`check_password_lifetime()`**（`is_password_expired` / `check_password_expired` **都不存在**）：

```cpp
static bool check_password_lifetime(THD *thd, const ACL_USER *acl_user) {
  if (likely(acl_user != nullptr) && !acl_user->password_expired &&
      acl_user->password_last_changed.time_type != MYSQL_TIMESTAMP_ERROR &&
      auth_plugin_is_built_in(acl_user->plugin.str) &&        // ★ 只对内置插件
      (acl_user->use_default_password_lifetime || acl_user->password_lifetime)) {
    ...
    interval.day = acl_user->use_default_password_lifetime
                       ? default_password_lifetime : acl_user->password_lifetime;
    if (interval.day) password_time_expired = ...;
```

- `default_password_lifetime` 默认 **0**（= 永不过期），范围 0..65535
- 客户端声明 `CLIENT_CAN_HANDLE_EXPIRED_PASSWORDS` → 进 **sandbox 模式**（只能 `ALTER USER ... IDENTIFIED BY`）；否则（`disconnect_on_expired_password` 默认 **ON**）直接 `ER_MUST_CHANGE_PASSWORD_LOGIN`

**★ 沙箱的强制执行点**（`sql/sql_parse.cc`）——"只能改密码"这条规则不是认证时的检查，而是**每条语句执行前**的拦截：

```cpp
        if (unlikely(
                (thd->security_context()->password_expired() ||
                 thd->security_context()->is_in_registration_sandbox_mode()) &&
                lex->sql_command != SQLCOM_SET_PASSWORD &&
                lex->sql_command != SQLCOM_ALTER_USER)) {
          if (thd->security_context()->is_in_registration_sandbox_mode())
            my_error(ER_PLUGIN_REQUIRES_REGISTRATION, MYF(0));
          else
            my_error(ER_MUST_CHANGE_PASSWORD, MYF(0));
          error = 1;
        } else {
          error = mysql_execute_command(thd, true);
```

★ 要点：

1. 白名单**只有两条语句**：`SET PASSWORD` 与 `ALTER USER`。所以 `SET GLOBAL`、`USE db`、甚至 `SELECT 1` 都会报 `ER_MUST_CHANGE_PASSWORD`——**沙箱是"语句白名单"，不是"功能子集"**。
2. ★ **存在第二种沙箱**：`is_in_registration_sandbox_mode()`——FIDO/多因素设备**注册未完成**时进入，报错是 `ER_PLUGIN_REQUIRES_REGISTRATION`（不是 `ER_MUST_CHANGE_PASSWORD`）。它与过期沙箱共用同一段拦截代码，由 `Security_context::m_registration_sandbox_mode` 区分，`turn_off_sandbox_mode()`（`sql/auth/sql_user.cc`）负责解除。
3. 判定用的是 `Security_context::password_expired()`——**proxy 场景下该标志会从连接用户复制到 sctx**，源码注释原文：
   > *"We are copying the connected user's password expired flag to the security context. This allows proxy user to execute queries even if proxied user password expires."*

   即 **被代理用户口令过期不进沙箱**（沿用代理用户的标志），这是 proxy 与过期交互的一个反直觉点。
4. `disconnect_on_expired_password`（默认 ON、READ_ONLY）决定的是**登录那一刻**：客户端没声明能力位就直接断开，根本不进沙箱。它关掉才会落到上面的运行时拦截。

#### 4.6 MFA 的凭据存储

JSON 结构（第 2/3 factor，**按数组下标决定 factor 序号**）：

```json
{"multi_factor_authentication": [
   {"plugin":"...", "authentication_string":"...", "passwordless":0, "requires_registration":1},
   {"plugin":"caching_sha2_password", "authentication_string":"*...", ...}
]}
```

语法：`ALTER USER u ADD 2ND FACTOR IDENTIFIED WITH ...` / `MODIFY 3 FACTOR ...` / `DROP 2 FACTOR`（只接受 `2`/`3`）。

### 五、资源限额、账户锁定

#### 资源限额

```sql
ALTER USER u WITH MAX_QUERIES_PER_HOUR 100
                MAX_UPDATES_PER_HOUR 10
                MAX_CONNECTIONS_PER_HOUR 5
                MAX_USER_CONNECTIONS 2;
```

★ **列名不是 `max_connections_per_hour`，而是 `max_connections`**（历史命名，极易写错）。四个列：`max_questions` / `max_updates` / `max_connections` / `max_user_connections`，均为 `int`。

判定与计数（函数名已核实）：

| 函数 | 文件 | 时机 |
|---|---|---|
| `check_mqh()` | `sql/sql_connect.cc` | 每条命令前，判 `max_questions` / `max_updates` |
| `check_for_max_user_connections()` | `sql/sql_connect.cc` | 建连接时，判 `max_user_connections` / `max_connections_per_hour` |
| `THD::time_out_user_resource_limits()` | `sql/sql_class.cc` | 按**滚动 1 小时窗口**重置计数 |
| `reset_mqh()` | `sql/sql_connect.cc` | CREATE/ALTER USER 改了限额后重置；`FLUSH PRIVILEGES` 也调 |

★ `CREATE USER` 不带 `WITH` 时 `specified_limits == 0` → 不置 `RESOURCE_ATTR` → 表列取默认 0 → **0 = 不限**。

**计数载体：`USER_CONN`**（`sql/sql_connect.h`，`thd->get_user_connect()`）

```cpp
typedef struct user_resources {     // 限额（来自 mysql.user 四列）
  uint questions;      // MAX_QUERIES_PER_HOUR
  uint updates;        // MAX_UPDATES_PER_HOUR
  uint conn_per_hour;  // MAX_CONNECTIONS_PER_HOUR
  uint user_conn;      // MAX_USER_CONNECTIONS（并发数，不是速率）
  enum { QUERIES_PER_HOUR = 1, UPDATES_PER_HOUR = 2,
         CONNECTIONS_PER_HOUR = 4, USER_CONNECTIONS = 8 };
  uint specified_limits;
} USER_RESOURCES;

typedef struct user_conn {          // 计数（内存，hash_user_connections）
  char *user; char *host; size_t len;
  ulonglong reset_utime;            // ★ "小时"窗口起点
  uint connections;                 // 当前并发连接数
  uint conn_per_hour, updates, questions;
  USER_RESOURCES user_resources;
} USER_CONN;
```

★ **"per hour" 是滚动窗口，不是整点对齐**——`THD::time_out_user_resource_limits()`：

```cpp
  /* If more than a hour since last check, reset resource checking */
  if (check_time - m_user_connect->reset_utime >= 3600000000LL) {   // 3600 秒（微秒）
    m_user_connect->questions = 1;
    m_user_connect->updates = 0;
    m_user_connect->conn_per_hour = 0;
    m_user_connect->reset_utime = check_time;
  }
```

即：窗口从**该账号首个连接起算**（`reset_utime` 初值 = `thd->start_utime`），不是 0 点整。

**★ `max_user_connections` 的取值规则（账号列优先，全局仅兜底）**：

```cpp
  if (global_system_variables.max_user_connections &&
      !uc->user_resources.user_conn &&          // ★ 账号列非 0 时全局完全忽略
      global_system_variables.max_user_connections < (uint)uc->connections &&
      !thd->is_admin_connection())              // 管理连接豁免
      → ER_TOO_MANY_USER_CONNECTIONS
  if (uc->user_resources.user_conn && uc->user_resources.user_conn < uc->connections)
      → ER_USER_LIMIT_REACHED "max_user_connections"
```

| 情况 | 用谁 | 错误码 |
|---|---|---|
| 账号 `max_user_connections` ≠ 0 | **只用账号列**（全局被忽略） | `ER_USER_LIMIT_REACHED` |
| 账号列 = 0 | 全局 `@@global.max_user_connections`（管理连接豁免） | `ER_TOO_MANY_USER_CONNECTIONS` |
| `max_connections_per_hour` | 只有账号级，无全局 | `ER_USER_LIMIT_REACHED` |

★ **连接数超限时会 `thd->set_user_connect(nullptr)`**——注释原文：*"The thread may returned back to the pool and assigned to a user that doesn't have a limit. Ensure the user is not using resources of someone else."*（线程会回线程池复用，必须解绑计数，否则会占别人的额度。）

#### 账户锁定

```sql
ALTER USER u ACCOUNT LOCK;      ALTER USER u ACCOUNT UNLOCK;
```

- 列：`mysql.user.account_locked` ENUM('N','Y')
- 登录判定：`if (unlikely(mpvio.acl_user->account_locked)) my_error(ER_ACCOUNT_HAS_BEEN_LOCKED, ...)`
- ★ **role 的本质就是 `account_locked='Y'`** —— 见下节 DROP ROLE 的判定代码

> 运行时锁定（`FAILED_LOGIN_ATTEMPTS` / `PASSWORD_LOCK_TIME`）是另一套，见 [`authentication.md`](authentication.md) 的"防暴力破解"。注意 `ACCOUNT UNLOCK` 会**同时**重置运行时的 `password_locked_state`。

### 六、DROP / RENAME USER：最难的两个

#### DROP USER 清理了哪些表（逐表核实）

| 表 | 清表数据 | 清内存 |
|---|---|---|
| `mysql.user` | ✅ | ✅ `acl_users->erase()` |
| `mysql.db` | ✅ | ✅ `acl_dbs->erase()` |
| `mysql.tables_priv` | ✅ | ⚠️ **只清表**（注释："Don't touch the in-memory array"），靠 `grant_version` 失效 |
| `mysql.columns_priv` | ✅ | ✅ `remove_matching_grants(column_priv_hash)` |
| `mysql.procs_priv` | ✅ | ✅ `proc_priv_hash` / `func_priv_hash` |
| `mysql.proxies_priv` | ✅ | ✅ `acl_proxy_users->erase()` |
| `role_edges` + `default_roles` | ✅ `revoke_all_roles_from_user()` | ✅ 随后 `populate_roles_caches()` 重建 |
| `global_grants` | ✅ `revoke_all_dynamic_privileges()`（best effort） | ✅ |
| `password_history` | ✅ | 无内存结构 |

★ **DROP USER 不会踢掉已连接的会话**：`sql_user.cc` 全文 grep `kill` / `close_connection` / `disconnect` → **0 处**。已有会话继续工作（持有旧的用户名副本与已 checkout 的 `Acl_map`），只是新连接会失败。

**DROP ROLE 的特殊判定**：

```cpp
        if (drop) {
          // if we're dropping roles and the account is not locked (not a role) bail off
          if (on_drop_role_priv && !acl_user->account_locked) {
            my_error(ER_SPECIFIC_ACCESS_DENIED_ERROR, MYF(0), command /* CREATE_USER */);
            return -1;
          }
```

即：**只有 `DROP ROLE` 权限时，只能删"已锁定"的账户（= role）**；删普通用户会要求 `CREATE_USER`。另外 **mandatory role 不能删**（`ER_MANDATORY_ROLE`）。

#### RENAME USER 为什么最复杂

1. 输入是 **(from, to) 成对**列表，一次循环消费两个 `LEX_USER`；
2. 要**两次** `handle_grant_data()`：先确认新名不存在，再执行改名；
3. 要改**所有**含 user/host 的表：`user` / `db` / `tables_priv` / `columns_priv` / `procs_priv` / `proxies_priv`（`ha_update_row`）+ `role_edges` / `default_roles`（`roles_rename_authid`）+ `global_grants`（`rename_dynamic_grant`）+ `password_history`；
4. 内存侧同步：`ACL_USER/ACL_DB/ACL_PROXY_USER` 的 `set_user()/set_host()`，三个 grant hash 的 `rename_matching_grants()`，还要迁移 partial revoke 的 restrictions：
   ```cpp
        } else if (user_to) {
          auto restrictions = acl_restrictions->find_restrictions(acl_user);
          acl_restrictions->remove_restrictions(acl_user);
          acl_user->set_user(&global_acl_memory, user_to->user.str);
          acl_user->set_host(&global_acl_memory, user_to->host.str);
          acl_restrictions->upsert_restrictions(acl_user, restrictions);
          rebuild_cached_acl_users_for_name();
        }
   ```
5. 可能**改到自己**：必须先保存原始 `Security_context`，**用原始身份写 binlog**，再恢复：
   ```cpp
    /* Restore the original security context temporarily because binlog must
       write the original definer/invoker in the binlog in order for slave to work */
    Security_context *current_sctx = thd->security_context();
    current_sctx->restore_security_context(thd, orig_sctx.get());
    result = log_and_commit_acl_ddl(thd, transactional_tables);
    current_sctx->restore_security_context(thd, current_sctx);
   ```
6. 额外限制：**role 不能改名**（`ER_RENAME_ROLE`）；改成匿名用户名时不能已有授予的 role。

### 七、缓存一致性：为什么 GRANT 不需要 `FLUSH PRIVILEGES`

**答案就在 `replace_user_table()` 的尾部**——写盘与改内存在**同一个函数**里完成：

```cpp
int replace_user_table(...) {
  acl_table::Acl_table_user_writer user_table(...);
  return_value = user_table.driver();          // ← 写 mysql.user（磁盘）
  if (!(return_value.error || return_value.skip_cache_update)) {
    ...
    clear_and_init_db_cache();                 // 清 db_cache
    if (old_row_exists)
      acl_update_user(...);                    // ← 改内存 ACL_USER
    else
      acl_insert_user(thd, ...);               // ← 插入内存 acl_users
  }
  return return_value.error;
}
```

所以：

| 操作 | 是否同步改内存 | 需要 `FLUSH PRIVILEGES` |
|---|---|---|
| `CREATE/ALTER/DROP/RENAME USER` | ✅ `acl_insert/acl_update_user` | **不需要** |
| `GRANT` / `REVOKE` | ✅ `acl_insert/acl_update_db` + `replace_*_table` | **不需要** |
| **直接 `UPDATE mysql.user`** | ❌ 绕过所有上述函数 | **必须** |

★ **一个反直觉的核实结论**：`sql/auth/sql_user.cc` 里 `increase_version` 与 `grant_version` **都是 0 处**（已二次确认）。它们只出现在 `sql_auth_cache.cc`、`sql_authorization.cc`、`role_tables.cc`、`dynamic_privilege_table.cc`。

即：**账户 DDL 改的是 `acl_users` 数组本身（就地更新），并不 bump `Acl_cache` 的版本号**。这与授权篇的结论一致——`Acl_cache` 版本只在 role 图、动态权限等处递增。

其余收尾动作（`mysql_drop_user`）：

```cpp
    rebuild_check_host();                      // 重建 host 索引
    rebuild_cached_acl_users_for_name();       // 重建按名缓存
    result = populate_roles_caches(thd, tables + ACL_TABLES::TABLE_ROLE_EDGES);
```

以及 `mysql_create_user` / `mysql_alter_user` 会 `reset_mqh()` 重置资源计数（**`mysql_drop_user` 不会**，因为没有会话可重置）。

**binlog 改写**：`log_and_commit_acl_ddl()` 里 `mysql_rewrite_acl_query()` 把 `IDENTIFIED BY 'plain'` 改写成 `IDENTIFIED WITH <plugin> AS '<hash>'` 再写 binlog —— **明文密码不落 binlog**。

**那么 `FLUSH PRIVILEGES` 到底做了什么**：真名 `reload_acl_caches()`（`sql/auth/sql_auth_cache.cc`，由 `sql/sql_reload.cc` 的 `REFRESH_GRANT` 分支调用）。它的函数注释只承认两个调用者：

```cpp
/**
  Reload all ACL caches
  We call this in two cases:
  1. FLUSH PRIVILEGES
  2. ACL DDL rollback
*/
```

动作 = **全量重读 ACL 表**（`acl_reload()` + `grant_reload()`）+ `notify_flush_event()` + `reset_mqh(nullptr, true)`（刷每账号限额）+ `servers_reload()`。

★ 两点需要说清：

1. **它不会 bump `Acl_cache` 版本号**（角色图缓存走 `increase_version()`，见上）。它重建的是 `acl_users` / `acl_dbs` 等数组与 `db_cache`。
2. ★ **源码里并没有"FLUSH PRIVILEGES 是 no-op / 不鼓励"的注释**——8.0 的立场是"账户 DDL 已原子化，所以不需要它"，而不是"它是空操作"。它仍会**独占 MDL**（与 ACL DDL、FTWRL、LOCK TABLES 全部互斥），生产上跑一次会阻塞所有账户相关操作。

---

## 相关的系统变量/状态变量

| 变量 | 作用域 | 默认 | 说明 |
|---|---|---|---|
| `default_password_lifetime` | GLOBAL | **0** | 0 = 永不过期；范围 0..65535 |
| `password_history` | GLOBAL | **0** | 全局"不能重复最近 N 个密码" |
| `password_reuse_interval` | GLOBAL | **0** | 全局"N 天内不能重用" |
| `generated_random_password_length` | **SESSION** | **20** | `RANDOM PASSWORD` 长度，范围 5..255 |
| `disconnect_on_expired_password` | GLOBAL, **READ_ONLY** | **true** | 过期且客户端不支持时直接断开（而非 sandbox） |
| `max_user_connections` | SESSION+GLOBAL | **0** | 单账号**并发**数（不是速率） |
| `validate_password.*` | GLOBAL | 见[上文](#31--validate_password80-里组件与插件双份源码并存) | 组件变量，卸载后消失 |

**状态变量**：`validate_password.dictionary_file_last_parsed` / `validate_password.dictionary_file_words_count`（插件版是下划线命名）。

---

## 可观测性

| 想看什么 | 怎么看 |
|---|---|
| 某账户的完整属性 | `SHOW CREATE USER u`（含 `REQUIRE`、`ACCOUNT LOCK`、`PASSWORD EXPIRE`、资源限额）；`SELECT * FROM mysql.user` 看原始列 |
| JSON 属性（双密码/MFA/锁定次数） | `SELECT user_attributes FROM mysql.user WHERE user='u'` |
| 密码强度 | `SELECT VALIDATE_PASSWORD_STRENGTH('pw')` |
| 谁快过期了 | `password_last_changed` + `password_lifetime`（或 `default_password_lifetime`） |
| 密码历史 | `SELECT * FROM mysql.password_history` |
| 账户 DDL 卡住 | `Acl_cache_lock_guard` 是 MDL，查 `performance_schema.metadata_locks` 里 `OBJECT_TYPE` 对应 ACL_CACHE 的等待 |

**错误码速查**：

| 错误码 | 场景 |
|---|---|
| `ER_USER_ALREADY_EXISTS` | ★ 真名，不是 `ER_USER_EXISTS`（不存在） |
| `ER_USER_DOES_NOT_EXIST` | ★ 真名，不是 `ER_NONEXISTING_USER`（后者不存在） |
| `ER_CANNOT_USER` | "Operation %s failed for %.256s"（语句级汇总错误） |
| `ER_NOT_VALID_PASSWORD` | 不满足强度策略 |
| `ER_PASSWORD_FORMAT` | `IDENTIFIED ... AS '<hash>'` 的 hash 格式非法 |
| `ER_CREDENTIALS_CONTRADICT_TO_HISTORY` | 新密码与历史重复 |
| `ER_ACCOUNT_HAS_BEEN_LOCKED` | 账户被锁 |
| `ER_MUST_CHANGE_PASSWORD_LOGIN` | 密码过期且客户端不支持 |
| `ER_CANNOT_LOCK_USER_MANAGEMENT_CACHES` | 拿不到 ACL 缓存 MDL |
| `ER_PASSWORD_EXPIRE_ANONYMOUS_USER` | 匿名用户不能 `PASSWORD EXPIRE` |

---

## Misc

### 易混淆概念

- **账户管理 vs 授权**：`CREATE USER` 建的是"能不能连进来"，`GRANT` 给的是"连进来后能做什么"。官方章名 *Access Control **and** Account Management* 明确把二者并列。
- **`max_connections`（列名）不是"每小时连接数"**：它是 `MAX_CONNECTIONS_PER_HOUR`。四个资源列全是 `max_*` 前缀，看语义别看名字。
- **第二密码不在 `authentication_string`**：在 `User_attributes` JSON 的 `additional_password`。
- **`User_attributes` 不是万能口袋**：`password_last_changed`、`password_lifetime`、`Password_reuse_history`、`Password_reuse_time` 都是**独立列**，不在 JSON 里。判断依据：引入年份早于 8.0.21 的是列，晚的是 JSON。

### 已知坑（源码证据逐条）

1. **★ 旧密码不会自动失效**（`RETAIN CURRENT PASSWORD` 之后）：只有换插件、显式 `DISCARD OLD PASSWORD`、或认证串置空才会丢弃。普通改密后旧密码仍可登录。
2. **随机密码绕过 `validate_password`**：`thd->m_disable_password_validation = true`。且 `RAND_bytes` 的返回值未检查；256→71 字符的线性映射不是整除，**分布非严格均匀**。
3. **字典文件 > 1MB 直接放弃加载**（只打 WARNING）：STRONG policy 下等于字典保护**静默失效**。
4. **字典词不归一化**：逐行 `getline` 原样入 set（不 trim、不 lowercase），而匹配时只把密码转小写 ⇒ 字典文件必须**全小写且无行尾 `\r`**。
5. **密码被截断到 100 字节才做字典检查**（`MAX_PASSWORD_LENGTH = 100`）：超长密码尾部不参与匹配。
6. **换认证插件 = 清空密码历史**，且此时不做"是否重复使用"比较。
7. **`ALTER USER ... IDENTIFIED WITH plugin`（不给新密码）会自动把密码标记为过期**，强制下次登录改密。
8. **非 `AUTH_FLAG_USES_INTERNAL_STORAGE` 插件**：password history / RETAIN CURRENT / DISCARD OLD 全被**降级为 warning 并清空**（`ER_WARNING_*_CLAUSES_VOID`）。
9. **密码过期只对 built-in 插件生效**——源码 TODO 原文：
   > *"Only the plugins that are known to use the mysql.user table to store their passwords support password expiration atm. TODO: create a service and extend the plugin API to support password expiration for external plugins."*
10. **每账户 `PASSWORD HISTORY n` 缺范围校验**（对比 `PASSWORD EXPIRE INTERVAL n DAY` 明确做了 `> UINT_MAX16` 检查）：落盘列是 `smallint`，超大 n 会被静默截断。
11. **`validate_password.length` 会被悄悄抬高**（`readjust_validate_password_length`，打 `ER_VALIDATE_PWD_LENGTH_CHANGED` WARNING）：当它小于 `number_count + 2×mixed_case_count + special_char_count` 时。
12. **组件未初始化时校验一律失败**（`is_initialized` 标志返回 true = 不合格）；`UNINSTALL COMPONENT` 后 `validate_password.*` 变量立刻消失。
13. **`mysql.create_user` 不支持 `IF NOT EXISTS` 的部分成功语义混乱**：`if_not_exists` 时已存在的用户会进 `extra_users` 并打 NOTE，同时进 binlog —— 源码里还专门有个 WARNING 处理 `extra_users` 插入失败（"binlog possibly incomplete"）。
14. **★ `GRANT` 永不隐式建用户**（8.0 固化行为，旧的 `NO_AUTO_CREATE_USER` sql_mode 已被 WL#10774 移除，`sql/system_variables.h` 只剩占位 `0x10000000 /* was: MODE_NO_AUTO_CREATE_USER */`）。**两个触发点、两个错误码**：
    - `set_and_validate_user_attributes()`：用户不存在 + `command == SQLCOM_GRANT` → `ER_CANT_CREATE_USER_WITH_GRANT`
    - `Acl_table_user_writer::setup_table()`：行未找到且 `can_create_user == false` → 同上；若是 GRANT 语句本身则 `ER_PASSWORD_NO_MATCH`

    消息原文：*"You are not allowed to create a user with GRANT"*。
15. **★ role 不能被 `RENAME USER`**：`is_role_id()` 返回真时报 **`ER_RENAME_ROLE`**（*"Renaming of a role identifier is forbidden"*——★ 不是 `ER_CANNOT_RENAME_ROLE`，那名字不存在）。`ACL_USER::is_role` 的注释说明了它唯一的用途：
    > *"If this ACL_USER was used as a role id then this flag is true. During RENAME USER this variable is used for determining if it is safe to rename the user or not."*
16. **★ `DROP USER` 不踢连接、也不归还该账号的连接计数**：`mysql_drop_user()` 里**没有**对 `user_connect` 的清理（计数只在 `release_user_connection()` / `decrease_user_connections()` 里递减，即连接断开或鉴权失败时）。所以删掉的账号，其已有会话继续有效，且 `max_user_connections` 的槽位直到那些连接断开才释放。
17. **`user_attributes` 解析失败会跳过该账号并只打 WARNING**（`ER_WARNING_AUTHCACHE_INVALID_USER_ATTRIBUTES`）；顶层非 JSON object 则直接 `ER_INVALID_USER_ATTRIBUTE_JSON` 失败。叠加前面说的"空值 remove / 仅 `metadata` merge_patch"，**手工编辑该列极易丢数据**。
18. **★ `flush_operation` 从不 `DELETE` 用户行**：`acl_table_user.cc` 注释原文
    > *"We should NEVER delete from the user table, as a uses can still use mysqld even if he doesn't have any privileges in the user table!"*

    所以"删用户"实际走的是逐张 grant 表清理 + 最后删 `mysql.user` 行，而**不是**靠级联。
19. **`FLUSH PRIVILEGES` 在生产上不是免费的**：它会独占 MDL，与 ACL DDL、FTWRL、LOCK TABLES **全部互斥**（`acl_tables_setup_for_write_and_acquire_mdl()` 的注释列出了这份互斥清单）。账户 DDL 已经原子化，正常运维不需要它；直接 `UPDATE mysql.user` 才需要——那种做法本身也是错的。

### ★ 本篇核实出的"流传但不存在"的名字

| 常见写法 | 真实 |
|---|---|
| `mysql_revoke()` | **不存在**；REVOKE 是 `mysql_grant(..., revoke_grant=true)` |
| `create_user_from_lex()` | **不存在**；真实链路 `set_and_validate_user_attributes()` → `replace_user_table()` |
| `validate_user_plugin()` 函数 | **不存在**（只有全局变量 `validate_user_plugins` 与 static `validate_user_plugin_records()`） |
| `set_and_validate_user_password()` | **不存在**；真名 `set_and_validate_user_attributes()` |
| `is_password_expired()` / `check_password_expired()` | **不存在**；真名 `check_password_lifetime()` |
| `acl_table_password_history.cc` | **文件不存在** |
| `ER_USER_EXISTS` / `ER_NONEXISTING_USER` | 真名 `ER_USER_ALREADY_EXISTS` / `ER_USER_DOES_NOT_EXIST` |
| `generate_native_password` 之外的 `make_scrambled_password` 作为插件入口 | 插件入口是 `generate_native_password()`，内部才调 `my_make_scrambled_password_sha1()` |

---

## 参考

**源码**

- `sql/auth/sql_user.cc`（约 4000 行）：四个 DDL 入口 + `set_and_validate_user_attributes()` + `check_orphaned_definers()` + `generate_random_password()`
- `sql/auth/acl_table_user.cc/.h`：`Acl_table_user_writer::driver()` 11 步流水线、`Acl_user_attributes` JSON 读写
- `sql/auth/sql_user_table.cc`：`open_grant_tables()`、10 张表的列定义、`log_and_commit_acl_ddl()`
- `sql/auth/sql_auth_cache.cc`：`Acl_cache_lock_guard`、`acl_insert_user` / `acl_update_user`
- `sql/auth/password.cc`：`my_make_scrambled_password_sha1()`
- `sql/auth/sha2_password.cc`：`caching_sha2_password_generate()`
- `sql/auth/password_policy_service.cc`：组件/插件桥接
- `components/validate_password/` + `plugin/password_validation/`：两套并存实现

**官方文档**

- MySQL 8.0 Reference Manual → [8.2.1 Account User Names and Passwords](https://dev.mysql.com/doc/refman/8.0/en/user-names.html)
- [8.2.8 Adding Accounts, Assigning Privileges, and Dropping Accounts](https://dev.mysql.com/doc/refman/8.0/en/adding-users.html)
- [8.2.14 Assigning Account Passwords](https://dev.mysql.com/doc/refman/8.0/en/assigning-passwords.html)
- [8.2.15 Password Management](https://dev.mysql.com/doc/refman/8.0/en/password-management.html)
- [8.2.16 Server Handling of Expired Passwords](https://dev.mysql.com/doc/refman/8.0/en/expired-password-handling.html)
- [8.2.20 Account Locking](https://dev.mysql.com/doc/refman/8.0/en/account-locking.html)
- [8.2.21 Setting Account Resource Limits](https://dev.mysql.com/doc/refman/8.0/en/user-resources.html)

**相关文档**

- 权限判定：[`authorization.md`](authorization.md)
- 登录时的凭据校验与 pem 文件：[`authentication.md`](authentication.md)
- definer 与 `Q_INVOKER` 的复制传递：[`definer.md`](definer.md)
- `Security_context` 字段语义：[`security_context.md`](security_context.md)
- 多因素认证的**流程**：[`authentication.md`](authentication.md)；凭据存储见本篇
