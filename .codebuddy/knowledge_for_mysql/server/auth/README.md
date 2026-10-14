# server/auth —— 认证与授权（AuthN / AuthZ）

> 本目录剖析 MySQL 的访问控制：**认证**（"你是谁"）与**授权**（"你能做什么"）两条链，以及它们之间的交界点 `Security_context`。基于 MySQL 8.0.39 源码。
>
> **官方术语对照**：MySQL 手册不用 AuthN / AuthZ 缩写，而是把访问控制分成两"阶段"——
> **Stage 1: Connection Verification**（= 认证，[8.2.6](https://dev.mysql.com/doc/refman/8.0/en/connection-access.html)）与
> **Stage 2: Request Verification**（= 授权，[8.2.7](https://dev.mysql.com/doc/refman/8.0/en/access-control.html)）。
> 整章名为 *Access Control **and Account Management***——**账户管理与访问控制是并列的两件事**，前者（`CREATE USER`、密码策略、资源限额）本目录尚缺，见 [五、缺口](#五缺口)。

## 目录

- [一、全景图：从 TCP 连接到权限判定](#一全景图从-tcp-连接到权限判定)
- [二、三层数据源与"GRANT 何时生效"](#二三层数据源与grant-何时生效)
- [三、篇目导航](#三篇目导航)
- [四、术语对照表](#四术语对照表)
- [五、缺口](#五缺口)

---

## 一、全景图：从 TCP 连接到权限判定

```
客户端
   │  TCP connect / unix socket
   ▼
╔═══════════════════════════════════════════════════════════════════════════════╗
║ AuthN  认证（"你是谁"）                                                        ║
║ 官方：8.2.6 Access Control, Stage 1: Connection Verification                  ║
║ 详见 → authentication.md                                                      ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  ① check_and_incr_conn_count()      max_connections 粗判                      ║
║  ② ip_to_hostname()                 host cache（LRU+hash）                    ║
║     └─ m_connect ≥ max_connect_errors → ER_HOST_IS_BLOCKED                    ║
║        ★ 只累计"握手错"，密码错不算                                            ║
║  ③ acl_check_host()                 无任何账号匹配该 host → ER_HOST_NOT_PRIV   ║
║  ④ send_server_handshake_packet()   Handshake V10 (0x0A) + 20B nonce          ║
║  ⑤ parse_client_handshake_packet()  HandshakeResponse41 / SSLRequest          ║
║     └─ 先解析头部 → sslaccept() → 再读真正的 Response                          ║
║  ⑥ find_mpvio_user()                账号不存在 → decoy_user()（诱饵，防枚举）   ║
║  ⑦ do_auth_once()  ×最多 2 次       ★ 乐观猜测 → RESTART → 换插件重跑          ║
║     └─ cached_client_reply 可省掉换插件的那次往返                              ║
║  ⑧ 插件内凭据校验（可多轮 AuthMoreData 0x01）                                 ║
║     native: 双重 SHA1 ｜ caching_sha2: SHA256 5000 轮 + RSA / 快速路径         ║
║  ⑨ do_multi_factor_auth()           2/3 因子，AuthNextFactor (0x02)           ║
║  ⑩ 装配 + 后置检查                   SSL / 锁定 / 过期 / 资源 / 连接数         ║
║     proxy → 默认角色 → checkout_access_maps()                                 ║
╚═══════════════════════════════════════════════════════════════════════════════╝
   │
   ▼
╔═══════════════════════════════════════════════════════════════════════════════╗
║ ★ 交界点：Security_context                          详见 → security_context.md ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║   m_user          → USER()          （连接身份）                               ║
║   m_priv_user     → CURRENT_USER()  （权限身份；proxy 时 = 被代理账号）         ║
║   m_master_access → 全局权限位图（登录时快照）                                 ║
║   m_acl_map       → role 展开后的权限快照（Acl_cache 借出）                     ║
║   m_restrictions  → partial revokes 限制集                                     ║
╚═══════════════════════════════════════════════════════════════════════════════╝
   │
   ▼
╔═══════════════════════════════════════════════════════════════════════════════╗
║ AuthZ  授权（"你能做什么"）                                                    ║
║ 官方：8.2.7 Access Control, Stage 2: Request Verification                      ║
║ 详见 → authorization.md                                                       ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  ① precheck()              ★ 开表之前（只有库名/表名可用）                     ║
║     check_table_access()                                                      ║
║       └─ check_access()    global → 内部 schema → db                          ║
║          ★ 逐层"扣除"：want_access &= ~该层已授予，不是短路                    ║
║       └─ check_grant()     表级 + 列级预判（两遍循环 + 延迟加锁）              ║
║  ② open_tables_for_query() ★ 取 MDL（权限先于 MDL）                            ║
║  ③ check_privileges()      ★ 开表之后（列级判定需要真实列信息）                ║
║       └─ check_grant_column() / check_grant_all_columns()                     ║
║       └─ check_grant_routine()（存储过程 / 函数）                              ║
╚═══════════════════════════════════════════════════════════════════════════════╝
   │
   ▼
╔═══════════════════════════════════════════════════════════════════════════════╗
║ 委托：definer（执行期权限切换）                          详见 → definer.md     ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║  SQL SECURITY DEFINER（默认）→ change_security_context() 把本 THD 的          ║
║  security context 临时换成 definer 的，执行完恢复（sp_head.cc）；              ║
║  INVOKER 不切换，沿用调用者上下文                                             ║
║  ★ 委托过去的权限如何被判定？→ 回到上面 AuthZ 判定链，原样再跑一遍             ║
╚═══════════════════════════════════════════════════════════════════════════════╝
   │
   ▼
执行 SQL
```

**三条时间线上的关键事实**：

| 事实 | 说明 |
|---|---|
| **权限先于 MDL** | `open_table()` 内**没有任何**权限检查；权限在 `precheck()`（开表前）与 `check_privileges()`（开表后）完成 |
| **授权是两阶段** | ① 只有库名/表名，② 开表后才有真实列信息 ⇒ 列级判定只能在 ② |
| **认证最多两次插件调用** | 不是循环；多轮往返发生在**单个插件内部** |

---

## 二、三层数据源与"GRANT 何时生效"

这是理解"为什么有时要 `FLUSH PRIVILEGES`、有时要重连"的**唯一**结构图。

```
┌─ L0 磁盘（权威）──────────────────────────────────────────────────────┐
│ mysql.user / db / tables_priv / columns_priv / procs_priv             │
│ proxies_priv / global_grants / role_edges / default_roles             │
│ password_history                                                     │
│ （InnoDB 事务表；受 DD 保护但由 server 直接用 SQL 读写）               │
└──────────────┬───────────────────────────────────────────────────────┘
               │  启动时 acl_reload() + grant_reload() 全量装载
               │  ★ 直接 INSERT/UPDATE/DELETE 改这里，内存不会感知
               │    → 必须 FLUSH PRIVILEGES（reload_acl_caches）
               ▼
┌─ L2 内存 ACL 缓存（全局，跨会话）─────────────────────────────────────┐
│ acl_users / acl_dbs / acl_proxy_users        （有序数组）              │
│ column_priv_hash / proc_priv_hash / func_priv_hash                    │
│ db_cache                        ← key = ip\0user\0db                  │
│ g_granted_roles（role 图）/ g_default_roles / g_mandatory_roles        │
│ g_dynamic_privileges_map / acl_restrictions                           │
│ 失效信号：grant_version++                                             │
│ ★ GRANT/REVOKE 走 replace_*_table()：写盘的同时就地改这些容器          │
│   → 所以表/列/routine 级权限立刻生效，不需要 FLUSH                     │
└──────────────┬───────────────────────────────────────────────────────┘
               │  有 active role 时按 (uid, active_roles, version) 展开
               ▼
┌─ L3 per-会话快照 ──────────────────────────────────────────────────────┐
│ Acl_cache（LF_HASH）→ Acl_map     ★ 版本号被编进 hash key              │
│ Security_context.m_master_access  ★ 登录时快照，之后不再查 acl_users    │
│ Table_ref::grant                  ← L1，靠 version != grant_version 失效│
└───────────────────────────────────────────────────────────────────────┘
```

**由此推出"GRANT 后多久生效"（官方 [8.2.13](https://dev.mysql.com/doc/refman/8.0/en/privilege-changes.html) 与源码一致）**：

| 权限类型 | 读哪一层 | GRANT 后对**已连接会话** |
|---|---|---|
| 表 / 列 / routine 级 | L2（每次查 hash） | **立刻** |
| 库级 | L2 + db_cache（GRANT 会清 db_cache） | **立刻** |
| 动态权限、role 继承 | L3（Acl_map） | 下一次 `checkout_access_maps()`（**语句级**） |
| **静态全局权限** | L3（`m_master_access` 登录快照） | **不生效** → 需重连 / 改自己 / 重算 Acl_map |

★ 最后一行就是"为什么 `GRANT ... ON *.*` 之后老会话还是不行"的**唯一正确答案**。

---

## 三、篇目导航

| 篇 | 第一主语 | 讲什么 | 不讲 |
|---|---|---|---|
| [`authentication.md`](authentication.md) | 握手包 + 认证插件 | 握手字节布局、native/caching_sha2 算法级剖析、认证插件清单、MFA 串接 | 认证后各字段的语义 |
| [`security_context.md`](security_context.md) | `Security_context` 对象 | 认证后各字段语义，proxy 与匿名用户 | 握手过程 |
| [`mfa.md`](mfa.md) | 多因素凭据 | 2/3 因子的凭据存储 | 认证流程中的串接（在 authentication.md） |
| [`definer.md`](definer.md) | definer | 定义者 + 权限委托 + 复制传递 | 权限如何被判定 |
| [`authorization.md`](authorization.md) | 权限 | 位图与五层模型、ACL 缓存、check_access 判定链、role、动态权限 | 认证 |
| [`account_management.md`](account_management.md) | 账户那一行 | CREATE/ALTER/DROP USER 主流程、密码子系统、资源限额、账户锁定 | 权限如何被判定 |

**阅读顺序建议**：`authentication.md`（认证）→ `security_context.md`（交界）→ `authorization.md`（授权）。`mfa.md` 与 `definer.md` 作为专题按需读。

---

## 四、术语对照表

**中文三个词的混乱**（"鉴权"歧义最大，本库不用它做分类轴）：

| 概念 | 英文 | 回答什么 | 官方对应 | 本库篇目 |
|---|---|---|---|---|
| **认证** | Authentication（**AuthN**） | "你是谁" | 8.2.6 Stage 1 + 8.2.17 + 8.2.18 | `authentication.md` |
| **授权** | Authorization（**AuthZ**） | "你能做什么" | 8.2.7 Stage 2 + 8.2.2/3/10/12/13 | `authorization.md` |
| **账户管理** | Account Management | "账户怎么建/改/限" | 8.2.8 / 8.2.14~8.2.16 / 8.2.20 / 8.2.21 | [`account_management.md`](account_management.md) |
| **审计** | Accounting / Audit | "做了什么" | 8.2.23 | **缺** |

> **AuthN / AuthZ 是 IAM 行业通用缩写**（出处：[Cloudflare](https://www.cloudflare.com/learning/access-management/authn-vs-authz/)），**不是 MySQL 官方术语**。英文里 `Auth` 单独用有歧义（同属 authentication 与 authorization 两个词），故用 N / Z 区分。中文"认证/授权"是两个完全不同的词，天然无歧义，所以这两个缩写在中文资料里少见。

**四者关系**：AAA（Authentication、Authorization、Accounting）是安全领域经典模型；官方章名 *Access Control **and** Account Management* 说明**访问控制 ≠ 账户管理**——`CREATE USER` 属于账户管理，不属于授权（尽管常顺带 `GRANT`）。

### 术语盘点：认证 / 授权全谱系（含 MySQL 未涉及的）

行业里这些概念很多，逐一标注 MySQL 8.0.39 的落点，避免读外部资料时张冠李戴。**"归属"列**即上面那四分类；**"MySQL"列**以本仓库源码核实为准。

**① 凭据与密码学**

| 术语 | 一句话 | MySQL | 归属 | 落点 |
|---|---|---|---|---|
| 挑战-应答（challenge-response） | 服务端发随机数，客户端用它算应答，口令不上线 | ✅ 核心 | 认证 | `authentication.md` §为什么用挑战-应答 |
| nonce / scramble | 每次连接现生成的挑战值（20B，8+12 分段） | ✅ | 认证 | 同上，3.0 辨析表 |
| 盐（salt） | 与密码一起存盘、作为哈希/KDF 输入的随机值 | ✅ | 认证 | caching_sha2 的 20B salt |
| KDF / 密钥派生 | 用多轮哈希抬高离线爆破成本 | ✅ 自研 | 认证 | SHA256 × 5000 轮 |
| PBKDF2 / bcrypt / scrypt / Argon2 | 标准 KDF | ❌ **都没用** | — | MySQL 用自研的 SHA256 多轮，**不是** PBKDF2 |
| 对称加密（AES） | 加解密同密钥 | ✅ 但不是认证 | **不属于这四分类** | 存储加密：keyring / 表空间 TDE / binlog 加密 |
| 非对称加密（RSA） | 公私钥 | ✅ | 认证 | caching_sha2 全认证路径的公钥交换（2048 位） |
| SHA1 / SHA2 | 摘要算法 | ✅ 都用 | 认证 | native 用双重 SHA1；caching_sha2 用 SHA256 |
| pass-the-hash | 拿到哈希即可冒充，无需知道明文 | ✅ 正是 double SHA1 要防的 | 认证 | `authentication.md` |
| 时序攻击 / 侧信道 | 用响应时间差异探测信息 | ✅ 有防护 | 认证 | `decoy_user()`（防账号枚举） |

**② 传输安全**

| 术语 | 一句话 | MySQL | 归属 | 落点 |
|---|---|---|---|---|
| TLS（≈SSL） | 传输层加密 | ✅ | 认证（连接建立） | `vio_start_ssl` |
| CA 证书 / 证书链 | 验证对端证书的可信根 | ✅ | 账户管理（存）+ 认证（判） | `ssl_ca`、`REQUIRE ISSUER` |
| mTLS（双向 TLS） | 服务端也验证**客户端**证书 | ✅ 支持 | 跨两处 | `REQUIRE X509 / SUBJECT / ISSUER / CIPHER` |
| 证书固定 / pinning | 固定信任某个证书 | ❌ | — | — |

**③ 现代认证协议（★ MySQL 8.0 全部没有）**

| 术语 | 一句话 | MySQL | 说明 |
|---|---|---|---|
| OAuth 2.0 | 授权委托框架（第三方拿 token 访问资源） | ❌ **0 处** | 见下方"为什么没有" |
| JWT | 自包含签名的 token | ❌ **0 处** | 同上 |
| OIDC / OpenID Connect | OAuth 之上的身份层 | ❌ **0 处** | — |
| SAML | XML 断言式 SSO | ❌ **0 处** | — |
| SigV4 | AWS 的请求签名算法 | ❌ **0 处** | — |
| IdP（身份提供商） | 集中签发身份的服务 | ❌ | ⚠️ 源码里 `idp` 是 `Object_id *idp` 参数名的**假阳性**，别误读 |
| SSO（单点登录） | 一次登录多处通行 | ⚠️ | 社区源码无；企业版提供 Windows/SSO 相关插件，本仓库无法核实实现 |
| Kerberos | 票据式网络认证 | ⚠️ | **仅客户端插件** `libmysql/authentication_kerberos`；服务端属企业版 |
| LDAP | 目录服务认证 | ⚠️ | **仅客户端插件**；服务端属企业版 |
| PAM | 可插拔认证模块 | ⚠️ | 社区源码无（企业版） |
| RADIUS | 远程认证拨号服务 | ❌ | — |
| FIDO / WebAuthn | 硬件密钥、无密码认证 | ⚠️ | 8.0.27+ 有 **客户端** `authentication_fido`（含 `extra/libfido2`）；服务端属企业版 |
| TOTP（动态口令） | 基于时间的 6 位验证码 | ❌ | MySQL 有 **MFA 框架**（最多 3 factor），但**无内置 TOTP 插件** |
| 2FA | 双因素 | ✅ 框架有 | 8.0.27+ MFA，见 `mfa.md` |

> **★ 为什么 MySQL 没有 JWT/OAuth 这一整套**：不是没跟上，而是**模型不同**。MySQL 是**连接协议级认证**——凭据只在握手那几轮交换，之后连接长期有状态，一条连接上跑几十万条 SQL，**没有"每个请求"这个位置**去放 token。JWT/OAuth 那套是**无状态 HTTP 请求级**的。MySQL 的扩展点是**认证插件接口**（`st_mysql_auth`），而不是 token 格式。

**④ 授权模型**

| 术语 | 一句话 | MySQL | 归属 | 落点 |
|---|---|---|---|---|
| RBAC（基于角色的访问控制） | 权限授予角色，用户通过角色获得 | ✅ **就是 role** | 授权 | `authorization.md`；官方 [WL#988](https://dev.mysql.com/worklog/task/?id=988) 标题即 *Role based privilege model* |
| DAC（自主访问控制） | 对象属主可自行决定授权 | ✅ **MySQL 属此类** | 授权 | `GRANT` 语义 |
| MAC（强制访问控制） | 系统级策略强制，属主无权改 | ❌ | — | （SELinux 属 OS 层，非 MySQL 内） |
| ABAC（基于属性） | 按主体/客体/环境属性动态判定 | ❌ | — | — |
| ACL（访问控制列表） | 对象上挂一张"谁有什么权限"的表 | ✅ | 授权 | MySQL 内核里 `acl_users` 等容器就叫 ACL |
| 最小权限原则 | 只给必需的权限 | ✅ 实践建议 | 授权 | — |
| 权限提升（privilege escalation） | 借高权限主体拿到额外能力 | ✅ **正是 DEFINER 的风险** | 授权 + 委托 | `definer.md` |
| 委托（delegation） | 把权限借给别的对象/身份 | ✅ | 授权 | `definer.md`（对象）+ proxy user（身份） |
| RLS（行级安全） | 按行过滤 | ❌ **社区版没有** | — | 最小粒度是**列级**（`columns_priv`） |
| 列级权限 | 精确到列 | ✅ | 授权 | `check_grant_column()` |
| 视图安全屏障（SECURITY BARRIER） | 防止视图用函数偷看底层数据（PG 概念） | ❌ | — | MySQL 用 `view_suid` 而非 barrier |
| 动态权限 | 不在 grant 表列里、可注册扩展的权限 | ✅ 8.0 特色 | 授权 | `global_grants` 表 |
| 部分撤销（partial revoke） | 全局权限里挖掉个别库 | ✅ 8.0.16+ | 授权 | `partial_revokes` 系统变量 |

**⑤ 审计**

| 术语 | 一句话 | MySQL | 归属 |
|---|---|---|---|
| AAA | 认证 + 授权 + 审计的安全经典模型 | ✅ 概念对齐 | 四分类的总纲 |
| 审计（Accounting / Audit） | 记录"做了什么" | ⚠️ | 插件/组件体系，本目录暂无（见[缺口](#五缺口)） |
| 不可抵赖（non-repudiation） | 行为可追溯且不可否认 | ⚠️ 靠审计实现 | 审计 |

**两个常见疑问，源码已核实的答案**（2026-09 检索确认）：

- **RBAC 不是缺失**：MySQL 的 role 体系就是 RBAC——引入它的官方 WorkLog **[WL#988](https://dev.mysql.com/worklog/task/?id=988)** 标题即 *"Role based privilege model"*，落在 [`authorization.md`](authorization.md) 的角色三章。
- **OAuth / JWT / SAML / SigV4 / IdP 在 8.0.39 源码中 0 处**：MySQL 是**连接协议级认证**（凭据只在握手期交换，连接长有状态），没有"每请求验 token"的位置；扩展点是认证插件接口 `st_mysql_auth`，不是 token 格式。Kerberos / LDAP / FIDO / OCI 仅有**客户端插件**在社区源码（`libmysql/authentication_*`），服务端插件属企业版。

---

## 五、缺口

按官方 8.2 章的覆盖面盘点，本目录还剩一块空白（**全库 0 篇**）：

| 缺口 | 官方对应 | 源码主战场 | 优先级 |
|---|---|---|---|
| **审计** | 8.2.23 SQL-Based Account Activity Auditing | audit plugin / audit log 组件 | 低——属插件/组件体系，更适合放 [`../plugin/`](../plugin/) |

> ✅ **账户管理已补完**（[`account_management.md`](account_management.md)，1175 行），覆盖官方 8.2.1 / 8.2.8 / 8.2.14~8.2.16 / 8.2.20 / 8.2.21 五节。
> ✅ **证书约束（账户粒度 X509）已在其中成章**：`REQUIRE SSL/X509/SUBJECT/ISSUER/CIPHER` 的完整链路（语法 → `LEX::ssl_type` → `mysql.user` 四列 → `ACL_USER::ssl_type` → `acl_check_ssl()`），横跨账户管理（存）与认证（判）两篇。

> ✅ [`security_context.md`](security_context.md) 已补强（198 → 715 行、代码 43%）：完整内存布局、`acl_getroot()` 装配算法、`Acl_map` 借出语义、两层 `check_access`、`has_global_grant` 双路径、★ 登录路径**不走** `acl_getroot` 的纠正。
> ✅ [`mfa.md`](mfa.md) 已补强（212 → 718 行）：`I_multi_factor_auth` 类体系、`do_multi_factor_auth()` 驱动循环、★ `START_MFA` 一次性包选择器、registration 与 passwordless 的 1FA/2FA 互换、`authentication_policy` 三条硬规则、★ 社区版插件生态真相。

**本目录 7 篇已全部达到剖析深度**（合计 9961 行），官方 8.2 章的覆盖面只剩**审计**（8.2.23，属插件/组件体系，更适合放 [`../plugin/`](../plugin/)）。

> [`definer.md`](definer.md) 已补全 INVOKER 侧（207 → 436 行，与 DEFINER 对等剖析），不再属于缺口。
