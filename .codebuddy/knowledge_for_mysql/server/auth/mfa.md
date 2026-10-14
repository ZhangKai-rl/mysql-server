# 多因素认证（MFA）深度解析

> 基于 MySQL 8.0.39 源码，剖析 MFA 子系统：`I_multi_factor_auth` 类体系、`LEX_MFA` 解析层、`do_multi_factor_auth()` 驱动循环、协议侧 `START_MFA` 机制、registration（设备登记）与 sandbox 模式、`authentication_policy` 策略校验、JSON 持久化，以及**社区版究竟能做什么**。
>
> **边界**：本篇讲 MFA 机制本身（因素链的数据与流程）。**认证握手与插件协商的通用部分**见 [`authentication.md`](authentication.md)；凭据在 `User_attributes` JSON 里的存储也见 [`account_management.md`](account_management.md)；`Security_context` 的 sandbox 标志见 [`security_context.md`](security_context.md)。

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

多因素认证（MFA）：一个账户最多绑定 **3 个认证因素**（`MAX_AUTH_FACTORS = 3`，`include/mysql_com.h`），登录时必须**依次通过所有已配置的因素**才能建立连接。

★ 关键结构性事实：**第 1 因子不进 MFA 体系**。它仍然是 `mysql.user.plugin` + `mysql.user.authentication_string`；MFA 体系只管**第 2/3 因子**。证据是枚举里根本没有 `FIRST_FACTOR`：

```cpp
enum class nthfactor { NONE = 1, SECOND_FACTOR, THIRD_FACTOR };
```

### 用途

解决"单密码泄露即完全失守"。即使 1FA（密码）泄露，缺 2FA（如 FIDO2/WebAuthn 硬件设备）仍无法登录——"something you know" + "something you have" 的叠加。

### 版本演进

| 版本 | 变化 |
|---|---|
| 8.0.27 | 引入 MFA 框架（WL#11634）、`ADD/MODIFY/DROP nth FACTOR` 语法、`MULTI_FACTOR_AUTHENTICATION` 能力位、AuthNextFactor 包 |
| 8.0.27 | `authentication_policy` 系统变量取代 `default_authentication_plugin` |
| 8.0.27 | `authentication_fido`（FIDO2/WebAuthn）与 registration / passwordless 流程 |

> 社区源码里 `authentication_fido` 只作为**注释中的示例字符串**出现，服务端实现不在社区树内（见[核心实现八](#八插件生态社区版究竟能做什么)）。

---

## 理论基础

### 设计思想与权衡

**核心决策：用"因素列表"而非"单插件多凭据"**。MFA 不是让一个认证插件存多个密码，而是给账户挂一串**互相独立的认证插件**，每个插件管自己那个因素。这样每个因素可以选完全不同的机制，彼此正交。

**为什么上限是 3**：3 是覆盖 knowledge + possession + inherence 三类所需的最小充分数，也是避免因素过多导致登录体验崩塌的工程取舍。

**"所有因素必须全通过"的代价**——两个失效场景，都显式体现在代码里：

1. 配了附加因素的账户，**旧客户端（不支持 `MULTI_FACTOR_AUTHENTICATION`）会被直接拒绝**（`do_multi_factor_auth` 开头就检查，返回 `CR_AUTH_USER_CREDENTIALS`）——这是"全通过"语义在协议层的兜底。
2. 需要登记的因素（FIDO 设备未登记）在登记完成前，允许以 **sandbox 模式**连接——这是"部分因素未通过仍可连接"的唯一例外，且该连接只能做登记操作。

### 六层架构（理解 MFA 的地图）

| 层 | 位置 | 职责 |
|---|---|---|
| 语法层 | `sql/sql_yacc.yy` | `ADD/MODIFY/DROP N FACTOR`、`INITIATE/FINISH REGISTRATION`、`UNREGISTER`、`INITIAL AUTHENTICATION BY` |
| 解析层 | `sql/table.h` 的 `struct LEX_MFA` | 一条 DDL 的"意图" |
| 语义/校验层 | `sql_mfa.{h,cc}` + `sql_user.cc` | `is_alter_allowed` / `alter_mfa` / 策略校验 |
| 持久化层 | `acl_table_user.{h,cc}` + `serialize/deserialize` | `User_attributes` 的 `multi_factor_authentication` JSON 数组 |
| 内存缓存层 | `ACL_USER::m_mfa`（挂在 `global_acl_memory`） | 认证期读取 |
| 认证驱动层 | `sql_authentication.cc` 的 `do_multi_factor_auth()` + `MPVIO_EXT::START_MFA` + `send_auth_next_factor_packet()` | 状态机与协议包 |

### 理论溯源

多因素是通用安全模型（NIST 的 authentication factor 分类：knowledge / possession / inherence）。MySQL 落成"因素枚举 + 认证插件解耦"：`nthfactor` 是因素的抽象编号，`I_multi_factor_auth` 是"一个可认证因素"的抽象，认证插件是实现。

### 他库对比

PostgreSQL 的认证是单因素的（`pg_hba.conf` 按 host/db/user 选一种方法），没有账户级多因素链；多因素要靠应用层或 LDAP/PAM 外挂。MySQL 把它做成**账户属性**（存在 `mysql.user.User_attributes` JSON 里）。

---

## 核心实现

### 主链路

```
【DDL】CREATE/ALTER USER ... ADD 2 FACTOR ...
  sql_yacc.yy → LEX_MFA（nth_factor=2, add_factor=true）
  → check_for_authentication_policy()            策略校验（可降级为 warning）
  → is_alter_allowed() → alter_mfa()             合法性与合并旧因子
  → init/finish_registration()                   登记（如需要）
  → Multi_factor_auth_list::serialize() → mysql.user.User_attributes(JSON)

【登录】
  Handshake V10 → HandshakeResponse41
  → do_auth_once()                               1FA（可能 RESTART 换插件）
  → do_multi_factor_auth()                       ★ 2FA/3FA 循环
      for 每个 factor:
        置 current_auth_factor、清空 cached_client_reply
        status = START_MFA → 插件首包走 AuthNextFactor(0x02)
        插件内部多轮走 AuthMoreData(0x01)
  → OK / ERR
```

### 一、类体系

#### `I_multi_factor_auth` 接口

```cpp
class I_multi_factor_auth {
 public:
  virtual ~I_multi_factor_auth() = default;
  virtual bool is_alter_allowed(THD *, LEX_USER *) { return false; }
  virtual void alter_mfa(I_multi_factor_auth *) {}
  virtual bool validate_plugins_in_auth_chain(THD *thd) = 0;
  virtual bool validate_against_authentication_policy(THD *) { return false; }
  virtual bool update_user_attributes() = 0;
  virtual void add_factor(I_multi_factor_auth *m [[maybe_unused]]) {}
  virtual bool serialize(Json_array &mfa_arr) = 0;
  virtual bool deserialize(uint f, Json_dom *mfa_dom) = 0;
  virtual bool init_registration(THD *, uint) = 0;
  virtual bool finish_registration(THD *, LEX_USER *, uint) = 0;
  virtual bool is_passwordless() = 0;
  virtual void get_info_for_query_rewrite(THD *, LEX_USER *) = 0;
  virtual void get_generated_passwords(Userhostpassword_list &gp, const char *u,
                                       const char *h) = 0;
  virtual void get_server_challenge(std::vector<std::string> &sc) = 0;

  Multi_factor_auth_list *get_multi_factor_auth_list() {
    return down_cast<Multi_factor_auth_list *>(this);
  }
  Multi_factor_auth_info *get_multi_factor_auth_info() {
    return down_cast<Multi_factor_auth_info *>(this);
  }
};
```

**10 个纯虚方法**；4 个有默认实现。★ 注意 `is_alter_allowed` 的语义反直觉：**返回 `false` = 允许**，`true` = 报错。

★ 设计上的坑：`get_multi_factor_auth_list()` / `get_multi_factor_auth_info()` 用 `down_cast`（等价 `static_cast`，非 debug 下无检查）在 List 与 Info 之间互转。整个 MFA 代码大量依赖"这个指针到底是 list 还是 info"的隐式约定。

#### `Multi_factor_auth_list`（容器）与 `Multi_factor_auth_info`（单因素）

```cpp
template <typename T>
using my_vector = std::vector<T, Mem_root_allocator<T>>;

class Multi_factor_auth_list : public I_multi_factor_auth {
  my_vector<I_multi_factor_auth *> m_factor;   // 元素实际全是 Multi_factor_auth_info
  void sort_mfa();                             // 保证 2FA 在前、3FA 在后
};

class Multi_factor_auth_info : public I_multi_factor_auth {
  MEM_ROOT *m_mem_root;
  LEX_MFA *m_multi_factor_auth;
  acl_table::Pod_user_what_to_update m_update;
};
```

- List 的几乎所有方法都是"for 循环转发给每个 factor"；`is_passwordless()` 是"任一 factor passwordless 即为 true"（`v |= ...`）。
- ★ `sort_mfa()` **硬断言只有 2 个元素**：`assert(m_factor.size() == 2);`
- ★ `Multi_factor_auth_info::operator=` **不拷** `nth_factor`，也不拷 add/drop/modify 动作标志——调用方必须先保证同一个 factor 序号才赋值。

#### 内存归属

| 对象 | 归属 |
|---|---|
| ACL 缓存里的 List / Info | `global_acl_memory`（`new (&global_acl_memory) Multi_factor_auth_list(...)`） |
| Info 内部的 `LEX_MFA` 与字符串 | 该 Info 持有的 `m_mem_root` |
| 解析期临时对象 | 局部 `MEM_ROOT mr(PSI_NOT_INSTRUMENTED, 256)`，用完 `mr.Clear()` |
| `mpvio->auth_info.multi_factor_auth_info[]` | `mpvio->mem_root`（= `thd->mem_root`） |

★ `ACL_USER::m_mfa` 是**裸指针且没有在析构里释放**——完全依赖 `global_acl_memory` 的整体生命周期。`set_mfa(root, m)` 是深拷贝，`set_mfa(nullptr, m)` 是浅拷贝（只赋指针）。

### 二、解析层：`LEX_MFA`

★ 它定义在 **`sql/table.h`**，不是 `sql_lex.h`（容易找错）。

```cpp
struct LEX_MFA {
  LEX_CSTRING plugin;
  LEX_CSTRING auth;
  LEX_CSTRING generated_password;
  LEX_CSTRING challenge_response;
  uint nth_factor;
  bool uses_identified_by_clause;
  bool uses_authentication_string_clause;
  bool uses_identified_with_clause;
  bool has_password_generator;
  bool passwordless;          /* CREATE USER .. INITIAL AUTHENTICATION BY */
  bool add_factor;            /* ALTER USER .. ADD nth FACTOR */
  bool modify_factor;
  bool drop_factor;
  bool requires_registration; /* 持久化状态 + 语法动作 */
  bool unregister;            /* ALTER USER .. nth FACTOR UNREGISTER */
  bool init_registration;     /* ALTER USER .. INITIATE REGISTRATION */
  bool finish_registration;   /* ALTER USER .. FINISH REGISTRATION */
  LEX_MFA() { reset(); }
  void reset() { ... nth_factor = 1; ... }
  void copy(LEX_MFA *m, MEM_ROOT *alloc);
};
```

`LEX_USER` 侧的配套字段：

```cpp
  /* restrict MFA methods to atmost 3 authentication plugins */
  LEX_MFA first_factor_auth_info;
  List<LEX_MFA> mfa_list;
  bool with_initial_auth;
```

| 标志位 | SQL 子句 | 语义 |
|---|---|---|
| `uses_identified_by_clause` | `IDENTIFIED BY 'pwd'` | 给明文密码 |
| `uses_authentication_string_clause` | `IDENTIFIED ... AS 'hash'` | 给已哈希串 |
| `uses_identified_with_clause` | `IDENTIFIED WITH plugin` | 指定插件 |
| `has_password_generator` | `BY RANDOM PASSWORD` | 服务端生成 |

可组合，如 `IDENTIFIED WITH caching_sha2_password BY RANDOM PASSWORD` 同时置两个。

### 三、`do_multi_factor_auth()`：驱动循环

```cpp
static int do_multi_factor_auth(THD *thd, MPVIO_EXT *mpvio) {
  int res = CR_OK;
  /* user is not configured with Multi factor authentication */
  if (!mpvio->acl_user->m_mfa) return res;
  /* If an old client connects ... return error. */
  if (!mpvio->protocol->has_client_capability(MULTI_FACTOR_AUTHENTICATION))
    return CR_AUTH_USER_CREDENTIALS;

  Multi_factor_auth_list *auth_factor =
      mpvio->acl_user->m_mfa->get_multi_factor_auth_list();
  for (auto m_it : auth_factor->get_mfa_list()) {
    Multi_factor_auth_info *af = m_it->get_multi_factor_auth_info();
    if (af->get_factor() == nthfactor::SECOND_FACTOR)
      mpvio->auth_info.current_auth_factor = 1;
    else if (af->get_factor() == nthfactor::THIRD_FACTOR)
      mpvio->auth_info.current_auth_factor = 2;
    /* reset cached_client_reply for 2nd and 3rd factors */
    mpvio->cached_client_reply.pkt = nullptr;

    plugin_ref plugin = my_plugin_lock_by_name(thd, af->plugin_name(),
                                               MYSQL_AUTHENTICATION_PLUGIN);
    if (plugin) {
      mpvio->plugin = plugin;
      mpvio->auth_info.auth_string =
          mpvio->auth_info
              .multi_factor_auth_info[mpvio->auth_info.current_auth_factor]
              .auth_string;
      mpvio->status = MPVIO_EXT::START_MFA;
      st_mysql_auth *auth = (st_mysql_auth *)plugin_decl(plugin)->info;
      res = auth->authenticate_user(mpvio, &mpvio->auth_info);
      if (res == CR_OK_AUTH_IN_SANDBOX_MODE) {
        if (af->get_requires_registration())
          thd->security_context()->set_registration_sandbox_mode(true);
        assert(af->get_requires_registration());
        plugin_unlock(thd, plugin);
        return CR_OK;
      }
      plugin_unlock(thd, plugin);
      if (res != CR_OK) { mpvio->status = MPVIO_EXT::FAILURE; break; }
      else mpvio->status = MPVIO_EXT::SUCCESS;
    } else {
      Host_errors errors;
      errors.m_no_auth_plugin = 1;
      inc_host_errors(mpvio->ip, &errors);
      my_error(ER_PLUGIN_IS_NOT_LOADED, MYF(0), af->get_plugin_str());
      res = CR_ERROR; break;
    }
  }
  return res;
}
```

#### ★ 它其实不是显式状态机

"第 N 个 factor 认证完再认证第 N+1 个"是靠**顺序 for 循环 + 插件同步阻塞调用**保证的：每轮循环 = 一整个 factor 的握手（插件内部反复 `write_packet` / `read_packet` 直到自己 return）。真正的状态机在 `MPVIO_EXT::status` + `server_mpvio_read/write_packet` 里。

#### `current_auth_factor` 是什么

- 取值 **0=1FA、1=2FA、2=3FA**（0-based，与 SQL 里 `n FACTOR` 的 1/2/3 差 1）
- 它是 `multi_factor_auth_info[3]` 数组的下标，元素类型：

```cpp
struct auth_factor_desc {
  const char *auth_string;
  unsigned long auth_string_length;
  unsigned int is_registration_required;
};
```

★ 一个真实的代码瑕疵：`server_mpvio_initialize()` 里

```cpp
  memset(mpvio->auth_info.multi_factor_auth_info, 0, sizeof(auth_factor_desc));
```

**只清零了 1 个元素**（少了 `* MAX_AUTH_FACTORS`）。

#### ★ 为什么每轮都要清空 `cached_client_reply.pkt`

`server_mpvio_read_packet()` 里有这段：

```cpp
  } else if (mpvio->cached_client_reply.pkt) {
    assert(mpvio->status == MPVIO_EXT::RESTART);
    /* If the data cached from the last server_mpvio_read_packet and a client
       has used the correct plugin, then we can return the cached data
       straight away and avoid one round trip. */
```

这个缓存原本是给 1FA 的 **plugin restart 省一轮 RTT** 用的（握手响应里带的密码被缓存）。如果进 2FA 时不清空：

- 2FA 插件第一次 `read_packet()` 会**错误地拿到 1FA 的密码包**（另一个插件的载荷）；
- 且 `write_packet` 也会条件性清空它，两边语义会打架。

所以每轮强制清空，保证**每个 factor 都从网络上真正读一包**。

### 四、协议侧：`START_MFA` 是"一次性包类型选择器"

```cpp
  /** when plugin returns a failure this tells us what really happened */
  enum { SUCCESS, FAILURE, RESTART, START_MFA } status;
```

`START_MFA` 只在两处被读：

**① `server_mpvio_write_packet()`** —— 该 factor 的**第一个**写包走 `AuthNextFactor(0x02)`，随后**立刻把 status 降级为 `FAILURE`**，于是该 factor 内后续所有写包走 `wrap_plguin_data_into_proper_command()`（命令字 `1` = AuthMoreData）：

```cpp
  else if (mpvio->status == MPVIO_EXT::START_MFA) {
    res = send_auth_next_factor_packet(mpvio, packet, packet_len);
    /* reset the status to avoid sending AuthNextFactor again for the same factor */
    mpvio->status = MPVIO_EXT::FAILURE;
  } else
    res = wrap_plguin_data_into_proper_command(protocol->get_net(), packet, packet_len);
```

★ **这就是"一个 factor 只有开头一个 0x02，其余全是 0x01"的实现机制**。注意这里的 `FAILURE` **不是"失败"**，而是"无特殊状态"的默认兜底分支——命名相当误导。

**② `server_mpvio_read_packet()`** —— 该 factor 的第一个读包会先触发一次 `write_packet(nullptr, 0)`（发出携带客户端侧插件名的 AuthNextFactor），再读客户端回包。

包格式（`send_auth_next_factor_packet()`，`0x02` + NUL 结尾插件名 + 剩余数据）：

```
int<1>   0x02
string[NUL]  plugin name（客户端侧插件）
string[EOF]  plugin provided data
```

能力位与常量：

```c
#define MULTI_FACTOR_AUTHENTICATION (1UL << 28)   /* 注意没有 CLIENT_ 前缀 */
#define AUTH_NEXT_FACTOR_PACKETTYPE 2
```

### 五、registration（设备登记）机制

#### 四个标志的区别

| 标志 | 性质 | 含义 |
|---|---|---|
| `requires_registration` | **持久化状态**（写进 JSON）+ 语法动作 | "这个 factor 还没完成设备登记" |
| `init_registration` | 语法动作 | `N FACTOR INITIATE REGISTRATION` —— 服务端生成随机挑战 |
| `finish_registration` | 语法动作 | `N FACTOR FINISH REGISTRATION SET CHALLENGE_RESPONSE AS '...'` |
| `unregister` | 语法动作 | 清空凭据、重新置 `requires_registration=true` |

```cpp
bool Multi_factor_auth_info::update_user_attributes() {
  m_update.m_what |= USER_ATTRIBUTES;
  m_update.m_user_attributes = acl_table::USER_ATTRIBUTE_NONE;
  if (m_multi_factor_auth->unregister) {
    set_auth_str(nullptr, 0);
    set_requires_registration(true);
  }
  return false;
}
```

#### `init_registration()`：挑战从哪来

★ **不是**通过插件 API，而是通过**组件服务** `mysql_authentication_registration.<plugin_name>`（定义在 `include/mysql/components/services/mysql_authentication_registration.h`，三个方法 `get_challenge_length` / `init` / `finish`）：

```cpp
  std::string service_name("mysql_authentication_registration.");
  service_name.append(get_plugin_str());
  ...
  if (!srv_registry->acquire(service_name.c_str(), &h_reg_svc)) {
    mysql_auth_reg_service->get_challenge_length(&plugin_buf_len);
    plugin_buf = new unsigned char[plugin_buf_len];
    mysql_auth_reg_service->init(&plugin_buf, plugin_buf_len);
  }
```

buffer 格式（源码 doxygen 原文）：

```
[salt length][random salt][relying party ID length][relying party ID][user name length][user name]
```

服务器把**用户名 + host** 追加到挑战后面，再调插件的 `generate_authentication_string()`（通常 base64）存进 `auth_string`。挑战通过 `get_server_challenge()` 以结果集（列名 `Server_challenge`）返回客户端。

★ 幂等保护：`if (get_auth_str_len()) return false;`（已 INITIATE 过就不再生成）。

#### `finish_registration()` 与 ★ passwordless 的"1FA/2FA 互换"

```cpp
  auth->generate_authentication_string(outbuf, &outbuflen, (char*)challenge_response, challenge_response_len);
  set_finish_registration(false);
  if (is_passwordless()) {
    /* 把 fido 凭据搬到第一因子 */
    lex_string_strmake(thd->mem_root, &user_name->first_factor_auth_info.plugin, get_plugin_str(), ...);
    lex_string_strmake(thd->mem_root, &user_name->first_factor_auth_info.auth, (const char *)outbuf, outbuflen);
  } else {
    set_auth_str((const char *)outbuf, outbuflen);
  }
  set_requires_registration(false);
```

★ **passwordless 的完整语义**（源码注释原文）：

```
      Before registration:
      user: u1
      plugin: caching_sha2_password
      user_attributes: {"multi_factor_authentication": [{"plugin":
          "authentication_fido", "passwordless": 1,
          "authentication_string": "", "requires_registration": 1}]}

      After registration:
      user: u1
      plugin: authentication_fido
      authentication_string: <fido credentials>
      user_attributes: NULL
```

即：**先用初始密码当 1FA 登录 → 完成登记 → FIDO 凭据变成新的 1FA，MFA JSON 清空**。语法层甚至把 `CREATE USER u IDENTIFIED WITH authentication_fido INITIAL AUTHENTICATION BY 'pass'` 在解析时就**互换**了（fido 塞进 `mfa_list` 当 2FA 且 `passwordless=true`，初始密码当 1FA）。

#### sandbox 模式

- **打开**：插件返回 `CR_OK_AUTH_IN_SANDBOX_MODE`（`-3`）时置位，**并立即 `return CR_OK` 短路掉后续所有 factor**：
  ```cpp
      if (res == CR_OK_AUTH_IN_SANDBOX_MODE) {
        if (af->get_requires_registration())
          thd->security_context()->set_registration_sandbox_mode(true);
        assert(af->get_requires_registration());
        return CR_OK;
      }
  ```
  ★ `assert` 在 release 下不生效——插件返回 `-3` 而 factor 并不 `requires_registration` 时，**不会进 sandbox 却拿到了 `CR_OK`**，等于绕过后续因子。
- **限制**：只允许 `ALTER USER` / `SET PASSWORD`，其它语句报 `ER_PLUGIN_REQUIRES_REGISTRATION`
- **`init_connect` 被忽略**（写 WARNING `ER_CONN_INIT_CONNECT_IGNORED_MFA`）
- **关闭**：`turn_off_sandbox_mode()` OR 所有 factor 的 `requires_registration`，只要还有一个没登记完就继续 sandbox

#### 语法与限制

```yacc
factor: NUM FACTOR_SYM
        { if ("2") $$ = 2; else if ("3") $$ = 3; else ER_WRONG_VALUE("nth factor"); }
```

- **只能给自己登记**（`ER_INVALID_USER_FOR_REGISTRATION`）
- **`INITIATE REGISTRATION` 不写 binlog**（`write_to_binlog = false`）
- `challenge_response` **硬上限 256 字节**

### 六、`authentication_policy`：策略校验

#### 三条硬规则

```cpp
    /* plugin name in first place holder cannot be empty */
    if (!it->length()) { if (list.size() == 1 || it == list.begin()) return true; }
    if (!it->length()) continue;              // 空 = 该因子可选
    if (!it->compare("*")) continue;          // * = 任意
    /* 2FA/3FA 不允许内部存储密码的插件 */
    if (it != list.begin() &&
        auth->authentication_flags & AUTH_FLAG_USES_INTERNAL_STORAGE) goto error;
    /* 1FA 不允许需要登记的插件 */
    if (it == list.begin() &&
        auth->authentication_flags & AUTH_FLAG_REQUIRES_REGISTRATION) goto error;
```

默认值 `"*,,"` → 解析后 3 个元素 `["*", "", ""]`（1FA 任意，2FA/3FA 可选）。

★ **规则 2 的严重后果**：`AUTH_FLAG_USES_INTERNAL_STORAGE` 正是 `caching_sha2_password` / `mysql_native_password` / `sha256_password` 的标志 ⇒ **社区版这 3 个"真"密码插件都不能做 2FA/3FA**。

★ **规则 3** 解释了为什么 FIDO 不能直接当 1FA（必须靠 passwordless 的"先放 2FA、登记完再换位"绕开）。

#### ★ 两套校验函数，行为不一致

| | 函数 | 位置 | 大小写 | 调用时机 |
|---|---|---|---|---|
| A | `Multi_factor_auth_list::validate_against_authentication_policy()` | `sql_mfa.cc` | **`std::string::compare`（敏感）** | **只在 `SQLCOM_ALTER_USER`** |
| B | `check_for_authentication_policy()` | `sql_user.cc` | `my_strcasecmp`（**不敏感**） | CREATE + ALTER（更早） |

B 比 A 复杂得多，因为要处理 DROP 导致的**因子位移**：

```cpp
        if (nth_factor == 2) {
          if (mfa_list && mfa_list->get_mfa_list_size() > 1) {
            /*
              dropping 2nd factor is like replacing 2nd factor method with 3rd
              factor if 3rd factor exists.
            */
            third_factor_plugin_name = ...;
            if (compare_plugin_with_policy(nth_factor, third_factor_plugin_name, ...)) goto error;
```

对应的实现在 `alter_mfa()`：

```cpp
      if (drop_2nd_factor) {
        lhs->m_factor.clear();
        Multi_factor_auth_info *t = tf->get_multi_factor_auth_info();
        t->set_factor(nthfactor::SECOND_FACTOR);    // ★ 原 3FA 变成新 2FA
        lhs->add_factor(tf);
      }
```

★ **持有 `AUTHENTICATION_POLICY_ADMIN` 时，违反策略只降级为 warning，语句照样成功**（两个函数的 `error:` 分支都是这个模式）。

### 七、持久化：JSON 与"序号由下标决定"

Key 与字段常量：

```cpp
const std::string json_multi_factor_authentication("multi_factor_authentication");

namespace mfa_consts {
constexpr const char requires_registration[] = "requires_registration";
constexpr const char passwordless[] = "passwordless";
constexpr const char auth_string[] = "authentication_string";
constexpr const char auth_plugin[] = "plugin";
}
```

JSON 形如：

```json
{"multi_factor_authentication": [
   {"plugin":"...", "authentication_string":"...", "passwordless":0, "requires_registration":1},
   {"plugin":"...", "authentication_string":"...", "passwordless":0, "requires_registration":0}
]}
```

★ **factor 序号完全由数组下标决定**（没有 `nth_factor` 字段）：

```cpp
bool Multi_factor_auth_info::deserialize(uint nth_factor, Json_dom *mfa_dom) {
  (nth_factor ? set_factor(nthfactor::THIRD_FACTOR)
              : set_factor(nthfactor::SECOND_FACTOR));
```

即 `i == 0 → 2FA`，`i != 0 → 3FA`。★ 副作用：**手工塞第 4 个元素也会被当作 3FA**，且不报错。

**每个 factor 存在哪**：

| 因子 | 位置 |
|---|---|
| 1FA | `mysql.user.plugin` + `authentication_string`（**不在** MFA JSON 里） |
| 2FA / 3FA | `User_attributes` → `multi_factor_authentication[i].plugin` / `.authentication_string` |
| passwordless 登记后 | 凭据搬到 1FA 的两列，MFA JSON 清空 |

**防篡改**：`validate_row()` 会针对手工 DML 改坏 `User_attributes` 的情况打 WARNING 并作废该行（`ER_MFA_USER_ATTRIBUTES_CORRUPT`）——典型场景是 2FA 用了内部存储插件，或 `requires_registration` 与插件能力矛盾。

### 八、★ 插件生态：社区版究竟能做什么

`include/mysql/plugin_auth.h` 的三个标志：

```c
#define AUTH_FLAG_PRIVILEGED_USER_FOR_PASSWORD_CHANGE (1L << 0)
#define AUTH_FLAG_USES_INTERNAL_STORAGE (1L << 1)
#define AUTH_FLAG_REQUIRES_REGISTRATION (1L << 2)
```

社区源码里可用于 MFA 的服务端插件（`plugin/auth/` 下只有 `auth_socket.cc`、`dialog.cc`、`mysql_no_login.cc`、`qa_auth_*.cc`、`test_plugin.cc`，加上 `sql/auth/` 里的三个内置）：

| 插件 | flags |
|---|---|
| `caching_sha2_password` / `mysql_native_password` / `sha256_password` | `USES_INTERNAL_STORAGE` |
| `auth_socket` / `mysql_no_login` / `dialog` / `test_plugin` / `qa_auth_*` | `PRIVILEGED_USER_FOR_PASSWORD_CHANGE` |

★★ **两条硬结论**：

1. **社区版没有插件能真正做 2FA/3FA 的"密码因子"** —— 因为策略禁止 `USES_INTERNAL_STORAGE` 用于 2FA/3FA。能用的只剩 `auth_socket`、`mysql_no_login`、`dialog`、`test_plugin` 这类非密码插件。
2. **社区版没有任何插件设置 `AUTH_FLAG_REQUIRES_REGISTRATION`** —— grep 该宏命中的全是**消费方**（`sql_mfa.cc`、`sql_user.cc`、`mysqld.cc` 等），**没有生产者**。

**服务端 `authentication_fido` 在社区源码里不存在**：

- 只有客户端插件：`libmysql/authentication_fido/`（`fido_client_plugin.cc`、`fido_assertion`、`fido_registration`），依赖 `extra/libfido2/libfido2-1.13.0/`，构建开关 `cmake/fido2.cmake`
- 客户端登记流程：`client/common/user_registration.cc`
- `plugin/` 与 `components/` 下**没有**服务端实现，也没有实现 `mysql_authentication_registration` 服务的组件

社区源码只提供了：**接口定义 + 服务契约 + 服务调用方 + 客户端插件 + sandbox 协议约定**。服务端 FIDO 插件与 registration 组件是**企业版的闭源部分**。

---

## 相关的系统变量/状态变量

| 变量 | 默认 | 作用域 | 说明 |
|---|---|---|---|
| `authentication_policy` | **`"*,,"`** | Global | 最多 3 个逗号分隔插件名；`*` = 任意，空 = 该因子可选 |
| `generated_random_password_length` | `20` | **Session** | `BY RANDOM PASSWORD` 长度（5..255） |
| `default_authentication_plugin` | `caching_sha2_password` | Global | 已废弃，被 `authentication_policy` 取代 |

解析规则（`parse_authentication_policy()`）：逗号数 ≥ 3 非法；**一旦出现空位，后面必须全空**（`'caching_sha2_password,,authentication_fido'` 这类非法）。

相关动态权限：`AUTHENTICATION_POLICY_ADMIN`（违反策略降为 warning）、`PASSWORDLESS_USER_ADMIN`（passwordless 用户的 MODIFY）。

---

## 可观测性

| 想看什么 | 怎么看 |
|---|---|
| 某账户配了几个 factor | `SELECT user_attributes FROM mysql.user WHERE user='u'`（看 `multi_factor_authentication` 数组长度） |
| 当前策略 | `SELECT @@authentication_policy;` |
| 是否处于 sandbox | 登录后执行非 `ALTER USER`/`SET PASSWORD` 语句 → 报 `ER_PLUGIN_REQUIRES_REGISTRATION`（`ERROR 3973`） |
| 服务端挑战 | `ALTER USER USER() 2 FACTOR INITIATE REGISTRATION` 返回的 `Server_challenge` 结果集 |
| MFA 配置损坏 | 错误日志 `ER_MFA_USER_ATTRIBUTES_CORRUPT` / `ER_MFA_PLUGIN_NOT_LOADED` |
| `init_connect` 被跳过 | 错误日志 `ER_CONN_INIT_CONNECT_IGNORED_MFA` |

★ 排查口诀：**2FA/3FA 认证失败先看三件事** —— ① 客户端是否支持 `MULTI_FACTOR_AUTHENTICATION`；② 该 factor 的插件是否 `USES_INTERNAL_STORAGE`（策略禁止）；③ 插件能否加载（`ER_PLUGIN_IS_NOT_LOADED`）。

---

## Misc

### 易混淆概念

- **`requires_registration` 既是状态也是动作标志**：它会被写进 JSON（状态），同时 `INITIATE`/`FINISH REGISTRATION` 语法也会把它设 true。
- **`nthfactor` 没有 FIRST**：1FA 不进 MFA 体系，用 `mysql.user.plugin`。
- **`current_auth_factor` 是 0-based**：0/1/2 对应 SQL 里的 1/2/3 factor。
- **`START_MFA` 用完即降级为 `FAILURE`**：这里的 `FAILURE` 不是失败，是"无特殊状态"。
- **`ALTER USER ... DROP 2 FACTOR` 会让原 3FA 变成新 2FA**（`set_factor(SECOND_FACTOR)`），所以策略校验要按"3FA 顶上来"重新判。
- **passwordless 用户 ADD/DROP 被禁**：只有 `MODIFY` 且持 `PASSWORDLESS_USER_ADMIN` 才允许，且插件必须仍是同一个。

### 已知坑（源码证据逐条）

1. ★ **`server_mpvio_initialize()` 的 memset 少乘 `MAX_AUTH_FACTORS`** —— 只清零了 `multi_factor_auth_info[0]`。
2. ★ **`Multi_factor_auth_info::serialize()` 首行是死代码**：
   ```cpp
   if (m_update.m_user_attributes & acl_table::USER_ATTRIBUTE_NONE) return false;
   ```
   `USER_ATTRIBUTE_NONE == 0L`，`x & 0` 恒为 0 ⇒ 永不 early-return。
3. ★ **两套策略校验的大小写处理不一致**（`std::string::compare` vs `my_strcasecmp`）。
4. ★ **`validate_against_authentication_policy()` 只在 `SQLCOM_ALTER_USER` 下被调用**。
5. **`LOCK_authentication_policy` 保护范围过窄**：只保护"取 vector 引用"这一瞬间，之后长时间无锁读。
6. **`down_cast` 而非 `dynamic_cast`**：类型错误无法在运行期发现。
7. **`Multi_factor_auth_list::sort_mfa()` 硬断言 2 元素**；`deserialize()` 的 `m_factor[nth_factor]` 无越界检查。
8. **`Multi_factor_auth_info::operator=` 不拷 `nth_factor` 与动作标志**。
9. ★ **`init/finish_registration()` 里 `srv_registry->acquire()` 失败无错误处理** —— 服务不存在时静默用空挑战继续。
10. ★ **sandbox 的 `assert(af->get_requires_registration())` 在 release 下不生效** —— 插件返回 `-3` 即可绕过后续所有 factor 并拿到 `CR_OK`。
11. **JSON key 大小写**：实现写 `multi_factor_authentication`，但 `sql_mfa.cc` 的注释里写成 `Multi_factor_authentication`。
12. **`ACL_USER::m_mfa` 是裸指针无析构释放**，`set_mfa(nullptr, m)` 是浅拷贝。
13. **`INITIATE REGISTRATION` 不写 binlog**，主从环境需注意。

### 错误码清单

| 错误码 | 文案要点 |
|---|---|
| `ER_MFA_METHOD_EXISTS` | 该 factor 已存在，先 DROP 或 MODIFY |
| `ER_MFA_METHOD_NOT_EXISTS` | 该 factor 不存在，先 ADD |
| `ER_MFA_METHODS_INVALID_ORDER` | 应先加低序号 factor |
| `ER_MFA_METHODS_IDENTICAL` | 两个 factor 不能相同 |
| `ER_AUTHENTICATION_POLICY_MISMATCH` | 与 `@@authentication_policy` 不符 |
| `ER_INVALID_MFA_PLUGIN_SPECIFIED` | 插件不能用于该 factor |
| `ER_IDENTIFIED_BY_UNSUPPORTED` | 需要登记的插件不支持 `IDENTIFIED BY` |
| `ER_INVALID_PLUGIN_FOR_REGISTRATION` | 该插件不支持登记 |
| `ER_PLUGIN_REQUIRES_REGISTRATION` | sandbox 期只能 ALTER USER / SET PASSWORD |
| `ER_PLUGIN_REGISTRATION_DONE` | 不能重复登记 |
| `ER_INVALID_USER_FOR_REGISTRATION` | 只能给自己登记 |
| `ER_USER_REGISTRATION_FAILED` | 登记校验失败 |
| `ER_INVALID_MFA_OPERATIONS_FOR_PASSWORDLESS_USER` | passwordless 用户不允许该操作 |
| `ER_MFA_USER_ATTRIBUTES_CORRUPT`（日志） | `User_attributes` 里的 MFA 数据损坏 |
| `ER_MFA_PLUGIN_NOT_LOADED`（日志） | 插件未加载，忽略该用户 |
| `ER_CONN_INIT_CONNECT_IGNORED_MFA`（日志） | sandbox 期 `init_connect` 被忽略 |

### ★ 本篇核实出的"不存在"与纠正

| 常见说法 | 真实 |
|---|---|
| `nthfactor::FIRST_FACTOR` | **不存在**；只有 `NONE/SECOND_FACTOR/THIRD_FACTOR` |
| `LEX_MFA` 在 `sql_lex.h` | 在 **`sql/table.h`** |
| 服务端 `authentication_fido` 在源码里 | **不在**（只有客户端 `libmysql/authentication_fido/`） |
| 社区版可用密码插件做 2FA | **不行**（策略禁 `USES_INTERNAL_STORAGE`） |
| `CLIENT_MULTI_FACTOR_AUTHENTICATION` | 真名 **`MULTI_FACTOR_AUTHENTICATION`**（无 `CLIENT_` 前缀） |
| registration 由插件 API 完成 | 由**组件服务** `mysql_authentication_registration.<plugin>` 完成 |

---

## 参考

**源码**

- `sql/auth/sql_mfa.h` / `.cc`：接口、List、Info、registration、策略校验
- `sql/table.h`：`struct LEX_MFA`、`LEX_USER::mfa_list`
- `sql/auth/sql_authentication.cc`：`do_multi_factor_auth()`、`send_auth_next_factor_packet()`、`server_mpvio_read/write_packet`、MFA/Registration 的 doxygen 时序图
- `sql/auth/sql_user.cc`：`check_for_authentication_policy()`、`turn_off_sandbox_mode()`
- `sql/auth/acl_table_user.cc`：JSON 序列化、`deserialize_multi_factor()`
- `sql/mysqld.cc`：`parse_authentication_policy()` / `validate_authentication_policy()`
- `include/mysql/plugin_auth.h`：`AUTH_FLAG_*`、`auth_factor_desc`
- `include/mysql/plugin_auth_common.h`：`CR_OK_AUTH_IN_SANDBOX_MODE`（`-3`）
- `include/mysql/components/services/mysql_authentication_registration.h`
- `libmysql/authentication_fido/`、`client/common/user_registration.cc`：客户端侧

**官方文档**

- MySQL 8.0 Reference Manual → [8.2.18 Multifactor Authentication](https://dev.mysql.com/doc/refman/8.0/en/multifactor-authentication.html)
- WorkLog: [WL#11634 — Multi-factor authentication](https://dev.mysql.com/worklog/task/?id=11634)

**相关文档**

- 认证流程与协议包：[`authentication.md`](authentication.md)
- `User_attributes` JSON 与凭据存储：[`account_management.md`](account_management.md)
- sandbox 标志在 sctx 上的位置：[`security_context.md`](security_context.md)
