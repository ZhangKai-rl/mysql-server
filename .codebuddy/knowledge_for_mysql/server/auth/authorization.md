# 授权子系统深度解析：权限模型、ACL 缓存与 `check_access` 判定链

> 基于 MySQL 8.0.39 源码，涵盖权限的位图模型与五级层次、ACL 内存缓存（`acl_users` / `column_priv_hash` / `db_cache` / `Acl_cache`）、`check_access()` 分层判定链、role 图与 BFS 展开、动态权限、部分撤销（partial_revokes）、`SHOW GRANTS` 反向拼装、缓存刷新与生效时机。
>
> **边界**：本篇讲**授权**（Authorization，"你能做什么"）——权限的内存模型、判定链、role、动态权限、部分撤销与缓存刷新。认证上下文（`m_user` / `m_priv_user` / `m_master_access` 的**来源**）见 [`security_context.md`](security_context.md)；多因素认证见 [`mfa.md`](mfa.md)；definer 与 SQL SECURITY 的委托语义见 [`definer.md`](definer.md)；开表时的元数据锁见 [`../../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)；grant 表与数据字典的关系见 [`../dd/dd.md`](../dd/dd.md)。
>
> **官方术语对照**：MySQL 手册**不用** AuthN/AuthZ 缩写，而是把访问控制分成两"阶段"。本篇对应 [8.2.7 Access Control, Stage 2: Request Verification](https://dev.mysql.com/doc/refman/8.0/en/access-control.html) + [8.2.2 Privileges Provided](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html) + [8.2.3 Grant Tables](https://dev.mysql.com/doc/refman/8.0/en/grant-tables.html) + [8.2.10 Using Roles](https://dev.mysql.com/doc/refman/8.0/en/roles.html) + [8.2.12 Partial Revokes](https://dev.mysql.com/doc/refman/8.0/en/partial-revokes.html) + [8.2.13 When Privilege Changes Take Effect](https://dev.mysql.com/doc/refman/8.0/en/privilege-changes.html)。
>
> ★ **本篇最反直觉的一条**（正文会反复回到它）：判定链**不是"命中即放行"的短路**，而是**逐层扣除**——每一层只把自己能满足的位从 `want_access` 里剔掉，剩下的交给下一层。理解这一点，才能解释"为什么 db 级有权限但表级没有时行为如此"。

## 目录

- [概述](#概述)
  - [是什么](#是什么)
  - [用途](#用途)
  - [版本演进](#版本演进)
- [理论基础](#理论基础)
  - [设计思想与权衡](#设计思想与权衡)
  - [理论溯源](#理论溯源)
  - [算法与数据结构](#算法与数据结构)
  - [他库对比与演进动机](#他库对比与演进动机)
- [核心实现](#核心实现)
  - [主链路](#主链路)
  - [一、内存模型：ACL 缓存与 grant 表](#一内存模型acl-缓存与-grant-表)
  - [二、判定链：`check_access` 分层裁决](#二判定链check_access-分层裁决)
  - [三、角色：图、展开与激活](#三角色图展开与激活)
  - [四、动态权限与部分撤销](#四动态权限与部分撤销)
  - [五、`SHOW GRANTS`：从位图反向拼装语句](#五show-grants从位图反向拼装语句)
  - [六、缓存刷新与生效时机](#六缓存刷新与生效时机)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

授权子系统把"用户拥有的权限"组织成**一组分层的内存结构**（统称 ACL 缓存），在每条 SQL 的**权限预检查阶段**通过 `check_access()` / `check_grant()` / `check_grant_column()` 逐级裁决"这个操作是否被允许"。

它的三个组成部分是：

- **磁盘上的 grant 表**（`mysql.user` / `db` / `tables_priv` / `columns_priv` / `procs_priv` / `proxies_priv` / `global_grants` / `role_edges` / `default_roles` / `password_history`）——10 张 InnoDB 表，是权限的权威来源；
- **内存中的 ACL 缓存**——启动时全量装载，之后**所有判定都只读内存**；
- **判定链**——`check_access()`（global + db + 内部 schema）→ `check_grant()`（表级 + 列级预判）→ `check_grant_column()`（列级），外加 role 展开与动态权限两条旁路。

### 用途

从一个 DBA 每天都会遇到的困惑开场：

```sql
-- 会话 A（已连接）
GRANT SELECT ON mydb.* TO 'app'@'%';   -- 另一个会话执行
-- 会话 A 立刻就能 SELECT mydb.t1 —— 不需要 FLUSH PRIVILEGES

GRANT SELECT ON *.* TO 'app'@'%';      -- 换全局级
-- 会话 A 仍然 SELECT 不了别的库！必须重连

UPDATE mysql.user SET Select_priv='Y' WHERE User='app';  -- 直接改表
-- 无论等多久都不生效，必须 FLUSH PRIVILEGES
```

三条语句三种结果，答案全部藏在内存缓存的组织方式里：**库/表/列级权限是"运行时查全局容器"，而全局权限是"登录时快照进 `Security_context`"**。本篇要把这条分界线讲清楚，并给出源码证据。

在整条 SQL 链路上的位置：授权发生在 **`prepare()` 阶段的 `precheck()`**（粗粒度，开表之前）与 **`execute()` 阶段的 `check_privileges()`**（细粒度，开表之后，因为列级判定需要真实列信息）。**`open_table()` 内部没有任何权限检查**——权限先于 MDL。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | grant 表是 **MyISAM**、**非事务**：一条 `CREATE USER` 对多个用户可能部分成功部分失败。无 role、无动态权限、无 partial_revokes。`mysql.user` 仍有 `Password` 列。ACL 缓存锁是 `Partitioned_rwlock`（按 thread_id 分片的 rwlock）。 |
| **8.0.0** | **role**（[WL#988](https://dev.mysql.com/worklog/task/?id=988)）：新增 `mysql.role_edges` / `mysql.default_roles`，引入 `CREATE ROLE` / `SET ROLE` / `SET DEFAULT ROLE` 与静态权限 `CREATE_ROLE` / `DROP_ROLE`。★ 源码里至今保留着"**不重构既有权限系统**"的痕迹：`check_access` / `check_grant` 等函数都是"有激活角色走新路径，否则 fallback 到 legacy 路径"（见 [三、角色](#三角色图展开与激活)）。 |
| **8.0** | grant 表改为 **InnoDB 事务表**（官方手册 [8.2.3 Grant Tables](https://dev.mysql.com/doc/refman/8.0/en/grant-tables.html) 明写 "In MySQL 8.0, grant tables use the InnoDB storage engine and are transactional"）。★ 该变更的 WL 编号**未公开**（WL#6963 不存在），不要引用。副作用（月报视角）：5.7 上"部分成功"的 ACL 语句在 8.0 上会**整体回滚**，从 5.7 用"搭建备库"方式升级时会出现整条语句失败。 |
| **8.0.2 起** | **动态权限**：新增 `mysql.global_grants` 表 + 运行时注册/注销机制（component service `dynamic_privilege_register`），服务器内置权限在 `dynamic_privilege_init()` 里注册。★ 精确引入小版本与 WL 编号**未核实**（官方 WorkLog 检索无果）。 |
| **8.0.14** | `mysql.user` 增加 `User_attributes` **JSON 列**——后续 partial revoke 的 `Restrictions`、双密码、失败登录锁定、MFA 全部挂在这一列上。这是"给 grant 表加可扩展字段"的第二种解法（与动态权限的"另开一张表"形成对照）。 |
| **8.0.16** | **[WL#12098 MySQL system users](https://dev.mysql.com/worklog/task/?id=12098)**：一次引入两件事——动态权限 **`SYSTEM_USER`** 与 **partial revokes**（`partial_revokes` 变量 + `Restrictions`）。配套 [WL#12364](https://dev.mysql.com/worklog/task/?id=12364)（`CONNECTION_ADMIN` 不能 KILL `SYSTEM_USER` 会话）与 [WL#12820](https://dev.mysql.com/worklog/task/?id=12820)（GRANT 语法扩展 `AS` 子句以传递 partial revoke 信息）。 |
| **8.0.18+** | ACL 缓存锁**从 `Partitioned_rwlock` 换成 MDL**（`MDL_key::ACL_CACHE`），见 [WL#14084 Lock improvements for ACL DDLs](https://dev.mysql.com/worklog/task/?id=14084)。`sql/auth/partitioned_rwlock.h` 文件保留但生产代码已无实例（只剩单测在用）。★ 精确小版本未核实。 |
| **8.0.22** | Grant Table Concurrency：为支持 grant 表上的并发 DML/DDL，原先加行锁的读改为**非锁定读**。 |
| **8.0.27** | MFA：`ACL_USER` 的凭据变成 `credentials[NUM_CREDENTIALS]`（2 组），见 [`mfa.md`](mfa.md)。 |
| 最新（8.4 / 9.x） | 授权侧**未见机制性变更**的官方说明（`partial_revokes` 仍默认 OFF、`Acl_cache` 结构未变）。★ 此项需持续跟踪 Release Notes，本篇以 8.0.39 为基线。 |

---

## 理论基础

### 设计思想与权衡

#### 1）为什么权限必须缓存成内存结构，而不是每次查 grant 表

**约束**：权限检查位于**每条 SQL 的必经路径**上（`precheck()`），且一条 SQL 可能涉及多张表、每张表多次 `check_access`。若每次判定都去读 `mysql.user`/`mysql.db`（InnoDB 索引查找 + server 层行格式化 + 权限位从 `SET` 列解包），代价与主查询同量级，不可接受。

**选定方案**：启动时 `acl_init()` → `acl_reload()` 把 10 张表全量读入内存；此后**所有判定只读内存**；账户管理语句（`GRANT`/`REVOKE`/`CREATE USER`…）**写盘的同时同步改内存**。

**代价（这条代价就是 `FLUSH PRIVILEGES` 存在的全部理由）**：内存是唯一权威，磁盘改了内存不知道。用 `INSERT`/`UPDATE`/`DELETE` 直接改 grant 表，只有走 `acl_reload()` + `grant_reload()` 才会被感知。官方手册 [8.2.13 When Privilege Changes Take Effect](https://dev.mysql.com/doc/refman/8.0/en/privilege-changes.html) 说得最直白：直接 DML 改 grant 表 "have **no effect**" 直到 flush。

源码层面的证据很硬：能改内存容器的函数只有 `acl_insert_user` / `acl_update_user` / `acl_insert_db` / `acl_update_db` / `acl_insert_proxy_user` / `grant_load` / `replace_table_table` / `grant_role` / `grant_dynamic_privilege` / `acl_load()` 等一族，**它们全部带 `assert(assert_acl_cache_write_lock(current_thd))`**，即必须在 `MDL_key::ACL_CACHE` 的 `MDL_EXCLUSIVE` 下执行。普通 DML 不拿这把锁，也拿不到。

**失效场景**：这是"用内存换速度"最典型的失效面——运维脚本直接改 `mysql.user`（尤其是老脚本、以及某些"批量改权限"的中间件）会静默失效；更糟的是改坏了结构会让实例**起不来**（见 [Misc · 坑与已知缺陷](#坑与已知缺陷) 的 [Bug #100468](https://bugs.mysql.com/bug.php?id=100468)）。

#### 2）为什么分五层（global / db / table / column / routine），以及"命中即放行"是个误解

**约束**：SQL 标准的层次化权限模型，`GRANT <priv> ON <level>` 的 `<level>` 天然是 `*.*` / `db.*` / `db.tbl` / `db.tbl(col)` / `PROCEDURE db.p`。若压平成一张"用户 × 对象 × 权限"表，管理粒度（`GRANT SELECT ON mydb.*` 一条语句覆盖整库）就没了，且 `mysql.db` 这种"库级通配"也无法表达。

★ **真正反直觉的是判定语义**。很多人的心智模型是"从粗到细，命中即放行"。源码不是这样：

```cpp
// check_access()：先把全局已满足的位剔掉
want_access &= ~sctx->master_access(db_name);
// check_grant()：进入表级判定前，再剔一次
want_access = orig_want_access;
want_access &= ~sctx->master_access(db_name);
// check_grant_column()：进入列级判定前，再剔一次
want_privilege &= ~grant->privilege;
if (!want_privilege) return false;   // Already checked
```

即：**`want_access` 是一个"还欠着多少权限"的递减计数器**，每过一层就 `&= ~该层已授予`。层与层之间是**并集（OR）累加到 `save_priv`/`grant->privilege`**，而不是"上层命中就短路返回"。

这个设计带来两个必须讲清的后果：

- **"更细粒度的限制优先"其实不存在于判定链里**——表级不能"撤销"db 级给的权限，只能**追加**。要表达"全局都有，但这个库没有"，必须靠 partial_revokes 这个**额外机制**（见 §4）。
- **`need_table_or_column_check` 这个布尔量**是唯一一处"提前放行"：当 `want_access` 全部落在 `TABLE_ACLS | PROC_ACLS | db_access` 内时，`check_access()` 就返回成功，把剩余的表/列级判定交给 `check_grant()`。所以 `check_access` 返回 `false`（通过）**不代表真的能访问表**——调用方必须接着调 `check_grant`。这正是 `check_table_access()` 存在的意义（它把两件事绑成一个整体）。

#### 3）为什么 8.0 才引入 role，以及它的代价

**约束**：直接授权给用户的管理成本是 O(用户数 × 权限数 × 对象数)。role 是**间接层**（权限 → role → 用户），把写成本降到 O(权限数 + 用户数)。这不是新思想（RBAC 1996 就有了），MySQL 拖到 8.0 才做是历史包袱 + 权限模型兼容性。

**代价（这是本节最值得记住的一条）**：role 引入后，一个用户的"有效权限"不再是 `mysql.user` 里的一行，而是**角色图上的一个可达闭包**。这个闭包计算很贵（BFS + 每层聚合 db/table/sp/dynamic/restrictions 六个 map），不能每条语句算一次。于是 8.0 引入 `Acl_cache`——一个以 **`(user@host, active_roles, global_version)`** 为 key 的 LF_HASH，缓存算好的 `Acl_map` 快照。

由此产生一条**极其重要的行为分界线**（官方手册也确认）：

| 权限类型 | 运行时读取路径 | GRANT 后多久对已连接会话生效 |
|---|---|---|
| 库级 | `acl_get()` 扫 `acl_dbs`（+ `db_cache`） | **立刻**（`replace_db_table()` 里 `clear_and_init_db_cache()`） |
| 表/列/routine 级 | `check_grant()` 查 `column_priv_hash` / `proc_priv_hash` | **立刻**（hash 就地改） |
| 动态权限、role 继承 | `Security_context` 的 `m_acl_map` 快照 | **下一次 `checkout_access_maps()`**（版本号变 → key 变 → 重新展开） |
| **静态全局权限** | `sctx->master_access()` 位图 | **不生效**，只有重连 / 改自己 / 重算 Acl_map 才刷新 |

最后一行就是概述里那个"GRANT 全局权限后必须重连"的源码原因：`m_master_access` 是登录时 `acl_getroot()` 一次性设定的位图，此后 `check_access()` 只读它，不再查 `acl_users`。

#### 4）为什么需要动态权限而不是继续给 grant 表加列

**约束**：新特性（备份、clone、keyring、加密、连接管理、资源管理…）不断引入新权限。若每种新权限都在 `mysql.user` 上加一个 `enum('N','Y')` 列，表结构随版本线性膨胀，升级必须改表（`mysql_upgrade` 的噩梦），且**第三方插件无法扩展**。

**选定方案**：动态权限用一个**运行时可增删改的字符串集合**（`Dynamic_privilege_register` = `std::unordered_set<std::string>`）承载，赋值信息另存 `mysql.global_grants`（4 列：`USER, HOST, PRIV, WITH_GRANT_OPTION`）。新增权限 = 组件在初始化时调一次 `register_privilege()`，**不动表结构**。

**代价**：
- 动态权限**只有全局级**（没有 db/table 级动态权限），因为 `global_grants` 表没有库/表列；
- **组件 UNINSTALL 时不会注销其注册的动态权限**（官方手册明说这是 "current practice, not a requirement"），而注销发生在服务器关闭时 ⇒ 已授予的动态权限在组件卸载后仍然存在且有效，**重启后才失效**；
- `GRANT ALL ON *.*` = 所有静态权限 + **当时已注册的**所有动态权限；之后才注册的动态权限**不会追溯授予**。这是"快照语义"的坑。

**一个有意思的对照**：8.0.14 给 `mysql.user` 加 `User_attributes` JSON 列，是同一问题的**另一种解法**（"用 JSON 承载可扩展属性"）。所以 8.0 同时存在两套扩展机制：动态权限走**新表 + 注册表**（面向权限），MFA/restrictions 走 **JSON 列**（面向账户属性）。

#### 5）为什么需要 partial_revokes

**约束**：全局权限无法表达"除了某个库之外都有"。`GRANT ALL ON *.*` 之后想排除 `mysql` 库，在 5.7/8.0.15 之前**没有任何办法**。

**更深的动机**（[WL#12098](https://dev.mysql.com/worklog/task/?id=12098) 原文例子）：只要某账号有 `CREATE USER, UPDATE, DELETE ON *.* WITH GRANT OPTION` + `SELECT ON mysql.*`，它就能 `ALTER USER root@localhost IDENTIFIED BY 'gibberish'`、`DROP USER root@localhost`、甚至 `DELETE FROM mysql.user WHERE user='root'`——**root 被"降权"账号反杀**。WL#12098 同时引入 `SYSTEM_USER`（保护 power user 不被 regular user 修改）与 `partial_revokes`（限制某个账号对特定库的权利）。

**代价（最大的一个坑）**：`partial_revokes` 是**全局开关**。打开后，`mysql.db` 里库名的**通配符不再通配**——`acl_get()` 从 `wild_compare()` 退化成 `strcmp()`：

```cpp
if (!acl_db->db ||
    (db &&
     (mysqld_partial_revokes()
          ? (!strcmp(db, acl_db->db))                       // ON：字面比较
          : (!wild_compare(db, strlen(db), acl_db->db,      // OFF：通配比较
                           strlen(acl_db->db), db_is_pattern))))) {
```

官方手册因此建议："在可能开启该特性的安装中，避免在权限赋权里使用未转义通配符"。也就是说，**一个只给 A 用户做 partial revoke 的操作，会改变全实例所有通配符库授权的匹配语义**——这正是"打开 partial_revokes 要让所有会话重算权限"的根因。

另外，只要还存在任何一条 restriction，`SET GLOBAL partial_revokes=OFF` 就会报 `ER_PARTIAL_REVOKES_EXIST` 被拒；启动时若表里有 restriction 而变量是 OFF，服务器会**自动打开它**并写 `ER_TURNING_ON_PARTIAL_REVOKES` 到 error log。WL#12098 的 S-9 给了三选一论证：候选 (a) OFF 时不传播限制（ON/OFF 切换会让 grantee 比 grantor 权限更多）、候选 (b) 始终传播但 OFF 时忽略（状态一致但有维护开销）、**最终选 (c) 只要还有限制就不允许关闭**——理由是语义简单、防误操作、便于将来默认开启。

#### 6）简化假设带来的失效与退化阈值（汇总）

| 简化假设 | 失效场景 | 阈值 / 触发条件 |
|---|---|---|
| 内存是权限的唯一权威 | 直接 DML 改 grant 表静默失效 | 任何 `INSERT/UPDATE/DELETE` 于 `mysql.*` |
| 全局权限登录时快照 | `GRANT ... ON *.*` 对已连接会话无效 | 已连接会话 + 无 active role |
| role 闭包按版本缓存 | 每次 `FLUSH PRIVILEGES` / `GRANT role` 后，所有带 role 的会话都要重算 BFS | 带 active role 的会话数 × GRANT 频率 |
| db 名大小写：restrictions 敏感、`mysql.db` 按 `lower_case_table_names` 折叠 | `lower_case_table_names=1` 下用非小写库名做 partial revoke 会失效（源码里专门有 `ER_AUTHCACHE_DB_ENTRY_LOWERCASED_REVOKE_WILL_FAIL` 告警） | `lower_case_table_names=1` + 混合大小写库名 |
| role 图无环 | 环的防线只在 `GRANT` 时（`check_if_granted_role_recursive()`），展开期靠 BFS color map 天然免疫、不报错 | 手工写 `mysql.role_edges` 可绕过 GRANT 期检查 |
| 动态权限注册表不记录归属 | 任何组件可注销别人的权限名；重复注册静默成功 | 多组件同名权限 |

### 理论溯源

| 理论 / 论文 | 源码落点 |
|---|---|
| **Access Matrix（Lampson 1971）/ HRU 模型**：访问控制状态可表示为"主体 × 客体"的矩阵 | MySQL 把它**按主体（auth_id）聚集**成行：`ACL_USER.access`（global）、`ACL_DB.access`（db）、`GRANT_TABLE.privs`（table）、`GRANT_COLUMN.rights`（column）。"按主体聚集"而非"按客体聚集"是 MySQL 与 PostgreSQL 的根本分野（见 [他库对比](#他库对比与演进动机)）。 |
| **Capability list / 位图权限**：主体的权限集合压成一个机器字 | `Access_bitmask`（`uint32_t`），31 个静态权限位；`Access_bitmask` 的按位与/或就是判定与合并的全部运算。 |
| **RBAC（Sandhu et al., *Role-Based Access Control Models*, IEEE Computer 1996）** | MySQL 实现的是 **RBAC1（role hierarchy）+ 部分 RBAC2**：`mysql.role_edges` 是角色继承关系（RBAC1）；`SET ROLE` / 默认角色 / `mandatory_roles` 是会话级激活与强制约束。**★ 但标准 RBAC2 的两大支柱 SSD（静态职责分离）与 DSD（动态职责分离）在 8.0.39 完全没有**，基数约束也没有——详见本节末尾「RBAC 模型族」。 |
| **BFS + color map（CLRS 第 22 章）** | `get_privilege_access_maps()` 用 `boost::breadth_first_search` + `vector_property_map<default_color_type>` 遍历 `Granted_roles_graph`；`Get_access_maps::discover_vertex()` 累加权限。color map 使**有环也不死循环**——不需要显式的深度限制或环检测。 |

#### ★ RBAC 模型族：RBAC0 / RBAC1 / RBAC2 / RBAC3 与 SSD / DSD

标准模型出自 Sandhu et al. 1996 的**四级框架**（RBAC0 是基座，往上加能力）。先把概念说清，再对 MySQL 落位：

| 层级 | 加了什么 | 一句话 |
|---|---|---|
| **RBAC0（core）** | 三个基本集合 + 两个指派 | 用户 U、角色 R、权限 P、会话 S。**UA**（用户↔角色指派）+ **PA**（角色↔权限指派）+ 会话到角色的**激活**（session roles）。用户不直接拿权限，全靠角色间接 |
| **RBAC1 = RBAC0 + 角色层次** | 角色可以继承角色 | 角色之间形成偏序（如"经理"继承"员工"）。权限沿层次**向下传递** |
| **RBAC2 = RBAC0 + 约束（constraints）** | 一组**约束规则** | 见下面 SSD / DSD / 基数约束 |
| **RBAC3 = RBAC1 + RBAC2** | 层次 + 约束 | 完整形态 |

**★ RBAC2 里的"约束"具体指什么**（这是最常被误读的部分）：

| 约束 | 全称 | 含义 | 例子 |
|---|---|---|---|
| **SoD** | Separation of Duty，**职责分离**（总称） | 把一笔业务的权力拆给不同角色，防止一人独揽 | 出纳与会计不能是同一个人 |
| **SSD** | **Static** SoD | **指派阶段**就禁止：同一个用户**不能被授予**两个互斥角色 | "出纳"与"会计"互斥 ⇒ 任何用户至多拥有其一，GRANT 时直接拒绝 |
| **DSD** | **Dynamic** SoD | 指派可以都有，但**同一个会话内不能同时激活**互斥角色 | 用户同时被授予出纳与会计，但一次登录只能激活一个；`SET ROLE` 时校验 |
| 基数约束 | Cardinality | 数量上限：一个角色最多多少成员 / 一个用户最多多少角色 | |
| 先决约束 | Prerequisite | 要拿到高级角色必须先有低级角色 | |

⇒ **SSD 管"能不能授予"，DSD 管"能不能同时激活"**；两者都是**服务器强制校验**，不是靠 DBA 自觉。

**★ MySQL 8.0.39 的准确落位：RBAC1（完整）+ 部分 RBAC2，不是 RBAC3**

| 能力 | 有 / 没有 | 源码落点 |
|---|---|---|
| RBAC0 三要素 | ✅ | UA=`mysql.role_edges`；PA=角色那行 `ACL_USER` + 各 grant 表；会话激活=`Security_context::m_active_roles` |
| **RBAC1 角色层次** | ✅ 完整 | `GRANT r2 TO r1` 建边，BFS 传递（详见第三章） |
| **SSD（互斥角色）** | ❌ **没有** | 见下「RBAC2 的真相」 |
| **DSD（会话内互斥）** | ❌ **没有** | 同上 |
| **基数约束** | ❌ **没有** | 同上 |
| 激活约束（只能激活已授予的） | ✅ | `check_if_granted_role()` → 错则 `ER_ROLE_NOT_GRANTED` |
| 委托/管理约束（ARBAC 风格，标准外） | ✅ | `WITH ADMIN OPTION` / `ROLE_ADMIN` / `SUPER` |
| 强制角色 | ✅ | `mandatory_roles` |
| 结构约束（禁止成环） | ✅ | `ER_ROLE_GRANTED_TO_ITSELF` |

⇒ 所以**职责分离在 MySQL 里只能靠 DBA 手工设计角色层次来"模拟"**（不把两个互斥角色授予同一用户），服务器**不提供任何强制机制**。这是它与 Oracle / PostgreSQL 等有 SoD 实现的系统的实质差距。
| **Split-ordered list（Shalev & Shavit, JACM 2006）** | `Acl_cache::m_cache` 是 `LF_HASH`（MySQL 的无锁哈希，实现即 split-ordered list + dummy 哨兵）。本库 [`../../infra/structure/hash.md`](../../infra/structure/hash.md) 有完整剖析。 |
| **Generation counter 做缓存失效** | 两处：`grant_version`（`GRANT_INFO.version` 的代际标记）与 `Acl_cache::m_role_graph_version`（**被编进 LF_HASH 的 key**，见 [工程实现技法](#二经典算法的实现落地不限于并发)）。 |
| **双缓冲 + 原子替换（RCU 的工程简化版）** | `acl_reload()`：new 出新数组 → 装载 → 成功后 `delete` 旧的；失败则整体回滚。整段在 `Acl_cache_lock_guard(WRITE_MODE)` 下，读侧要么看到全旧，要么看到全新。 |

### 算法与数据结构

| 结构 | 类型 | 为什么选它 | 复杂度 |
|---|---|---|---|
| 权限集合 | `Access_bitmask`（`uint32_t`） | 判定就是按位与；31 个权限正好塞进一个字 | 判定 O(1) |
| `acl_users` / `acl_dbs` / `acl_proxy_users` | `Prealloced_array<T, 10>*`，**有序** | 有序是为了"最具体的账号排在最前"（`get_sort()` 给通配符加权），遍历时首个命中即最具体；`Prealloced_array` 就地预分配 10 个元素避免小实例频繁分配 | 登录匹配 O(n)（n = 账号数），但排序保证语义正确 |
| `name_to_userlist` | `unordered_map<用户名, list<ACL_USER*>>` | 二级索引：先 O(1) 定位同名账号链表，再在链表里线性比较 host | 登录匹配 O(同名账号数) |
| `column_priv_hash` | `malloc_unordered_multimap<std::string, GRANT_TABLE>`，key = `user\0db\0table\0` | 表/列级权限的 key 天然是三元组；用 `\0` 拼接成一个 `std::string` 当 key，避免自定义 hasher | 表级判定 O(1) |
| `GRANT_TABLE::hash_columns` | `collation_unordered_multimap<std::string, GRANT_COLUMN>` | 列级按列名精确查；`collation_` 前缀表示用 `system_charset_info` 做**大小写不敏感**比较 | 列级判定 O(1) |
| `db_cache` | `malloc_unordered_map<std::string, acl_entry>`，key = `ip\0user\0db` | db 级权限在 `acl_dbs` 上是 O(n) 线性扫描，命中率低时很贵，故加一层结果缓存 | 命中 O(1)，miss O(n) |
| `Granted_roles_graph` | `boost::adjacency_list<setS, vecS, bidirectionalS, …>` | role 之间是多对多有向关系，且需要"出边（授予给谁的 role）"与"入边（谁是 role）"双向查询（`in_degree` 决定 `is_role`）；`setS` 保证重复 `GRANT` 幂等 | 展开 O(V+E) |
| `Acl_map` 缓存 | `LF_HASH` + 版本化 key + 原子引用计数 | role 闭包计算昂贵，按 `(uid, active_roles, version)` 缓存；无锁读是高频路径的硬需求 | checkout O(1) |
| `Dynamic_privilege_register` | `std::unordered_set<std::string>` | 只关心"这个名字注册过吗"，不需要 value | O(1) |
| `DB_restrictions` | `unordered_map<std::string, Access_bitmask>`（懒分配指针） | 绝大多数账号没有 restriction，懒分配避免每个 `ACL_USER` 都背一个 map | O(1) |

### 他库对比与演进动机

| 维度 | MySQL 8.0 | PostgreSQL | Oracle |
|---|---|---|---|
| **权限存放** | 按**主体**聚集：`mysql.user`/`db`/`tables_priv`/`columns_priv` 里每行是"某主体在某范围上的位图" | 按**客体**聚集：`pg_class.relacl` 是挂在对象上的 ACL 数组（`aclitem[]`），对象自己带着"谁能对我做什么" | 系统权限 + 对象权限两分，对象权限挂在对象上 |
| **后果** | `SHOW GRANTS FOR u` 一条语句就能列出该用户的全部权限（天然反向拼装）；但"谁对这个表有权限"要扫全表 | 反之：`\dp tbl` 一目了然；但"用户 u 有哪些权限"要遍历所有对象 | 介于两者之间，靠 `DBA_*_PRIVS` 视图补齐 |
| **角色** | role 是一条 `mysql.user` 记录（`is_role`），**需要显式激活**（`SET ROLE` / 默认角色 / `activate_all_roles_on_login`） | role 是集群级对象，登录即自动继承，**不需要激活** | role 需要 `SET ROLE`，与 MySQL 更接近 |
| **权限扩展** | 动态权限（运行时注册） | 无对应机制，权限集合编译时固定 | 固定 |
| **"全局除某库"** | partial_revokes（全局开关，改通配符语义） | 无内建机制，靠 `REVOKE ... FROM PUBLIC` + schema 权限组合 | 无（靠 VPD） |

★ **MySQL 为什么要"激活"这一步**（这是与上表最相关的设计决策）：`SET ROLE` 让一个会话可以**主动降权**（`SET ROLE NONE`），符合最小权限原则；代价是多了一层"授予 ≠ 生效"的心智负担，也是 `CURRENT_ROLE()` 返回 `NONE` 时最常被问"我明明 GRANT 了为什么没权限"的原因。

**社区边界澄清（务必写清）**：MySQL 社区版**没有行级安全（RLS / row-level security / Virtual Private Database）**。最小粒度就是**列级**（`mysql.columns_priv`）。Oracle 有 VPD/RLS，PostgreSQL 9.5+ 有 `CREATE POLICY`，MariaDB/Percona 也没有内建 RLS（Percona 曾有过相关讨论但未进主干）。社区里"按行过滤"只能靠**视图**（`CREATE VIEW ... WHERE tenant_id = CURRENT_USER()`）实现，且视图走 definer/invoker 语义（见 [`definer.md`](definer.md)）。

---

## 核心实现

### 主链路

一条 `SELECT` 从进来到取到数据，授权相关调用的真实顺序（`open_table()` 内**无**权限检查，这点必须纠正直觉）：

```
dispatch_command()
  └─ THD::reset_for_next_command()
       └─ sctx->checkout_access_maps()            【role 展开：借出/重建 Acl_map 快照】
  └─ mysql_execute_command()
       └─ Sql_cmd_select::execute()                [sql/sql_select.cc]
            ├─ Sql_cmd_dml::prepare()
            │    ├─ precheck()                     ★① 粗粒度权限检查（无 MDL）
            │    │    └─ check_table_access(SELECT_ACL, tables, false, UINT_MAX, false)
            │    │         ├─ check_access()       global → internal schema → db
            │    │         └─ check_grant()        表级 → 列级预判（两遍循环 + 延迟加 ACL 读锁）
            │    └─ open_tables_for_query()        ★② 开表 + 取 MDL
            │         └─ open_tables() → open_table() → open_table_get_mdl_lock()
            │              └─ thd->mdl_context.acquire_lock(&mdl_request)
            └─ Sql_cmd_dml::execute()
                 ├─ restore_cmd_properties()
                 └─ check_privileges()             ★③ 细粒度检查（开表后，因需真实列信息）
                      ├─ check_all_table_privileges()  → check_single_table_access()
                      └─ Query_block::check_column_privileges()
                           └─ Item::check_column_privileges
                                └─ check_column_grant_in_table_ref()
                                     └─ check_grant_column()
```

★ 为什么必须**两阶段**：① 阶段只有库名/表名可用（还没开表），只能做 global/db/内部 schema 与"表级位图"的判定；列级判定需要 `Field_iterator_table_ref` 遍历真实字段，只能在开表之后的 ③ 做。

**分层判定的五层队列**（`check_table_access()` 的头注释原文）：

```
1. Global privileges                      ← check_access()   （sctx->master_access）
2. OR (db privileges AND host privileges) ← check_access()   （acl_get() / sctx->db_acl()）
3. OR table privileges                    ← check_grant()    （column_priv_hash）
4. OR column privileges                   ← check_grant_column()（hash_columns）
5. OR routine privileges                  ← check_grant_routine()（proc/func_priv_hash）
```

配合"逐层扣除"的语义（[理论基础 §2](#2为什么分五层global--db--table--column--routine以及命中即放行是个误解)），整条链的形状是：

```
want_access ─[剔 global]─→ ─[剔 internal schema]─→ ─[剔 db]─→ ─[剔 table]─→ ─[剔 column]─→ 0 ?
                                                                                          ├─ 0   → 通过
                                                                                          └─ ≠0  → 拒绝（报错信息取"还欠着"的那些位）
```

---

### 一、内存模型：ACL 缓存与 grant 表

#### 权限位：一个 `uint32_t` 装下全部静态权限

权限位定义在 `sql/auth/auth_acls.h`（**不是** `sql_acl.h`——8.0 里 `sql_acl.h` 已退化成 29 行的兼容壳，只包含 `auth_common.h`）：

```cpp
/* Total Number of ACLs present in mysql.user */
#define NUM_ACLS 31
typedef uint32_t Access_bitmask;

#define SELECT_ACL ((Access_bitmask)1 << 0)
#define INSERT_ACL ((Access_bitmask)1 << 1)
#define UPDATE_ACL ((Access_bitmask)1 << 2)
#define DELETE_ACL ((Access_bitmask)1 << 3)
#define CREATE_ACL ((Access_bitmask)1 << 4)
#define DROP_ACL   ((Access_bitmask)1 << 5)
#define RELOAD_ACL ((Access_bitmask)1 << 6)
#define SHUTDOWN_ACL ((Access_bitmask)1 << 7)
#define PROCESS_ACL ((Access_bitmask)1 << 8)
#define FILE_ACL   ((Access_bitmask)1 << 9)
#define GRANT_ACL  ((Access_bitmask)1 << 10)
#define REFERENCES_ACL ((Access_bitmask)1 << 11)
#define INDEX_ACL  ((Access_bitmask)1 << 12)
#define ALTER_ACL  ((Access_bitmask)1 << 13)
#define SHOW_DB_ACL ((Access_bitmask)1 << 14)
#define SUPER_ACL  ((Access_bitmask)1 << 15)
#define CREATE_TMP_ACL ((Access_bitmask)1 << 16)
#define LOCK_TABLES_ACL ((Access_bitmask)1 << 17)
#define EXECUTE_ACL ((Access_bitmask)1 << 18)
#define REPL_SLAVE_ACL ((Access_bitmask)1 << 19)
#define REPL_CLIENT_ACL ((Access_bitmask)1 << 20)
#define CREATE_VIEW_ACL ((Access_bitmask)1 << 21)
#define SHOW_VIEW_ACL ((Access_bitmask)1 << 22)
#define CREATE_PROC_ACL ((Access_bitmask)1 << 23)
#define ALTER_PROC_ACL ((Access_bitmask)1 << 24)
#define CREATE_USER_ACL ((Access_bitmask)1 << 25)
#define EVENT_ACL ((Access_bitmask)1 << 26)
#define TRIGGER_ACL ((Access_bitmask)1 << 27)
#define CREATE_TABLESPACE_ACL ((Access_bitmask)1 << 28)
#define CREATE_ROLE_ACL ((Access_bitmask)1 << 29)
#define DROP_ROLE_ACL ((Access_bitmask)1 << 30)

#define NO_ACCESS  ((Access_bitmask)1 << NUM_ACLS)   /* == 0x80000000 */
#define ALL_ACCESS (NO_ACCESS - 1)                   /* == 0x7FFFFFFF */
```

**★ `NO_ACCESS` 是个反直觉的哨兵**：它是 `1 << 31`，比 `ALL_ACCESS` 还大。所以 `if (access & NO_ACCESS)` 恒为假——它不是一个权限位，而是"权限集合为空"的标记。`get_table_grant()` 在拿不到 ACL 缓存读锁时就 `return (NO_ACCESS)`（表示"什么都没有"，而不是"全部都有"）。

各层**可授予范围**用掩码常量表达（`_OP_ACLS` 是不含 `GRANT_ACL` 的操作权限子集）：

```cpp
#define DB_OP_ACLS   (UPDATE|SELECT|INSERT|DELETE|CREATE|DROP|REFERENCES|INDEX|ALTER| \
                      CREATE_TMP|LOCK_TABLES|EXECUTE|CREATE_VIEW|SHOW_VIEW| \
                      CREATE_PROC|ALTER_PROC|EVENT|TRIGGER)      /* 18 位 */
#define TABLE_OP_ACLS (SELECT|INSERT|UPDATE|DELETE|CREATE|DROP|REFERENCES|INDEX|ALTER| \
                       CREATE_VIEW|SHOW_VIEW|TRIGGER)            /* 12 位 */
#define PROC_OP_ACLS  (ALTER_PROC_ACL | EXECUTE_ACL)             /* 2 位 */

#define DB_ACLS         (DB_OP_ACLS    | GRANT_ACL)
#define TABLE_ACLS      (TABLE_OP_ACLS | GRANT_ACL)
#define COL_ACLS        (SELECT_ACL | INSERT_ACL | UPDATE_ACL | REFERENCES_ACL)
#define PROC_ACLS       (PROC_OP_ACLS  | GRANT_ACL)
#define SHOW_PROC_ACLS  (PROC_OP_ACLS  | CREATE_PROC_ACL)
#define GLOBAL_ACLS     ( 31 位全集 )
#define TMP_TABLE_ACLS  (SELECT|INSERT|UPDATE|DELETE|CREATE|DROP|INDEX|ALTER)
```

**★ `SUPER_ACL ∉ DB_OP_ACLS` 这一条决定了"SUPER 只能全局授予"**。判定链里对 `SUPER` 没有任何 `if` 特判，它的"只能全局"是被这一行**隐式**表达的：

```cpp
if (((want_access & ~sctx->master_access(db_name)) & ~DB_ACLS) || ...) {
  my_error(ER_ACCESS_DENIED_ERROR, ...);   // 请求的位里有不属于 DB_ACLS 的 → 全局没有就必拒
  return true;
}
```

`~DB_ACLS` 把 `SUPER / RELOAD / SHUTDOWN / PROCESS / FILE / CREATE_USER / CREATE_TABLESPACE / CREATE_ROLE / DROP_ROLE / SHOW_DB / REPL_*` 全部保留下来——只要这些位里有一个全局没给，连 db 级都不用查了。**这就是"权限作用域"的实现方式：不是查一张作用域表，而是用位掩码常量。**

位图 ↔ 权限名 的双向映射靠 `sql/auth/auth_acls.cc` 里的两个容器（下标即位序）：

```cpp
const std::vector<std::string> global_acls_vector = {
    consts::SELECT, consts::INSERT, consts::UPDATE, consts::DELETE,
    consts::CREATE, consts::DROP, consts::RELOAD, consts::SHUTDOWN,
    consts::PROCESS, consts::FILE, consts::GRANT, consts::REFERENCES,
    consts::INDEX, consts::ALTER, consts::SHOW_DATABASES, consts::SUPER,
    consts::CREATE_TEMPORARY_TABLES, consts::LOCK_TABLES, consts::EXECUTE,
    consts::REPLICATION_SLAVE, consts::REPLICATION_CLIENT, consts::CREATE_VIEW,
    consts::SHOW_VIEW, consts::CREATE_ROUTINE, consts::ALTER_ROUTINE,
    consts::CREATE_USER, consts::EVENT, consts::TRIGGER,
    consts::CREATE_TABLESPACE, consts::CREATE_ROLE, consts::DROP_ROLE};

const std::unordered_map<std::string, int> global_acls_map{...};  // 名字 → 位序
```

`SHOW GRANTS` 与 `get_privilege_desc()`（错误信息里"you need (at least one of) the XXX privilege(s)"的那串名字）都靠它。

#### 磁盘位序 ≠ 内存位序

`mysql.db` / `tables_priv` / `columns_priv` / `procs_priv` 里的 `SET(...)` 列，其**元素的排列顺序与内存位图不同**（历史遗留）。所以有一族宏做双向转换：

```cpp
#define DB_CHUNK0 (SELECT_ACL|INSERT_ACL|UPDATE_ACL|DELETE_ACL|CREATE_ACL|DROP_ACL)
#define DB_CHUNK1 (GRANT_ACL | REFERENCES_ACL | INDEX_ACL | ALTER_ACL)
...
#define fix_rights_for_db(A)   ...   /* 磁盘 → 内存 */
#define get_rights_for_db(A)   ...   /* 内存 → 磁盘 */
#define fix_rights_for_column(A)      (((A)&7) | (((A) & ~7) << 8))
#define get_rights_for_column(A)      (((A)&7) | ((A) >> 8))
```

`acl_load()` 里 `db.access = fix_rights_for_db(db.access)`、`GRANT_TABLE` 构造里 `privs = fix_rights_for_table(privs)`——**读盘必经一步位重排**。这也是"直接改 `mysql.db` 的列值极易改错"的原因之一。

#### 内存容器全景

```
                            ┌──────────────────────────────────────────┐
   磁盘（mysql 库 InnoDB）  │            内存（ACL 缓存）              │
   ─────────────────────    │                                          │
   mysql.user          ───► │ acl_users      : Prealloced_array<ACL_USER>  有序
                            │   ├ name_to_userlist : unordered_map<user, list<ACL_USER*>>
                            │   └ acl_check_hosts  : collation_unordered_map<host, ACL_USER*>
                            │     acl_wild_hosts   : Prealloced_array<ACL_HOST_AND_IP>
   mysql.db            ───► │ acl_dbs        : Prealloced_array<ACL_DB>    有序
                            │ db_cache            : unordered_map<ip\0user\0db, acl_entry>
   mysql.tables_priv   ───► │ column_priv_hash : multimap<user\0db\0table\0, GRANT_TABLE>
   mysql.columns_priv  ───► │   └ GRANT_TABLE::hash_columns : multimap<列名, GRANT_COLUMN>
   mysql.procs_priv    ───► │ proc_priv_hash / func_priv_hash : multimap<..., GRANT_NAME>
   mysql.proxies_priv  ───► │ acl_proxy_users  : Prealloced_array<ACL_PROXY_USER>
   mysql.global_grants ───► │ g_dynamic_privileges_map : multimap<Role_id, (priv, grant_opt)>
                            │ g_dynamic_privilege_register : unordered_set<string>（已注册名）
   mysql.role_edges    ───► │ g_granted_roles   : Granted_roles_graph（boost 有向图）
                            │ g_authid_to_vertex: unordered_map<authid, vertex>
   mysql.default_roles ───► │ g_default_roles   : unordered_multimap<Role_id, Role_id>
   (无对应表，来自变量)      │ g_mandatory_roles : vector<Role_id>
   mysql.user.User_attributes
       .Restrictions   ───► │ acl_restrictions  : malloc_unordered_map<authid, Restrictions>
                            │
                            │ ★ 二级缓存：Acl_cache（LF_HASH<key, Acl_map*>）
                            │   key = user@host ` version ` role1@h1 ` role2@h2 `
                            └──────────────────────────────────────────┘
```

两个 MEM_ROOT 分管生命周期：`global_acl_memory`（`ACL_USER` / `ACL_DB` 的字符串）与 `memex`（`GRANT_NAME` / `GRANT_TABLE` / `GRANT_COLUMN`）。

#### `ACL_USER`：role 也是一种 user

```cpp
class ACL_ACCESS {          // 基类：权限 + 主机 + 排序键
 public:
  ACL_HOST_AND_IP host;
  ulong sort;
  Access_bitmask access;
};

class ACL_USER : public ACL_ACCESS {
 public:
  USER_RESOURCES user_resource;         // max_questions / updates / connections 等账户限额
  char *user;
  enum SSL_type ssl_type;
  const char *ssl_cipher, *x509_issuer, *x509_subject;
  LEX_CSTRING plugin;
  bool password_expired;
  bool can_authenticate;
  MYSQL_TIME password_last_changed;
  uint password_lifetime;
  bool use_default_password_lifetime;
  bool account_locked;
  /**
   If this ACL_USER was used as a role id then this flag is true.
   During RENAME USER this variable is used for determining if it is safe
   to rename the user or not.
  */
  bool is_role;                          // ★ role 与 user 的唯一区分标志
  uint32 password_history_length;
  bool  use_default_password_history;
  uint32 password_reuse_interval;
  bool  use_default_password_reuse_interval;
  Lex_acl_attrib_udyn password_require_current;
  Acl_credential credentials[NUM_CREDENTIALS];   // 8.0.27 MFA：2 组凭据
  Password_locked_state password_locked_state;   // 失败登录锁定
  I_multi_factor_auth *m_mfa;
};
```

★ **`is_role` 不是持久化字段**（`mysql.user` 表里没有这一列），它是**从角色图推导出来的拓扑属性**：

```cpp
void static update_role_flag_of_acl_user(
    const Role_vertex_descriptor &role_vert, ACL_USER *acl_user) {
  degree_s_t count = boost::in_degree(role_vert, *g_granted_roles);
  acl_user->is_role = (count > 0) ? true : false;
}
```

即：**只要有人 GRANT 过它（入度 > 0），它就是 role**。这解释了一个常见困惑：`CREATE ROLE r` 之后再 `REVOKE` 掉最后一条边，`r` 就"退化"成普通 user（`is_role` 复位为 false），可以登录了（虽然它是 locked 的）。

`CREATE ROLE r` 的实质（`Sql_cmd_create_role::execute()` 里先设 `account_locked = true`、`password_expired`、清空密码相关 LEX 字段，再调 `mysql_create_user(..., is_role=true)`）：

> **`CREATE ROLE r` ≡ `CREATE USER 'r'@'%' ACCOUNT LOCKED PASSWORD EXPIRE`（无密码）**

`LEX_USER::init()` 在 host 未指定时填 `"%"`，所以 role 的 host 恒为 `%`。`is_role=true` 在 `mysql_create_user()` 内部的实质作用只有一处：**跳过密码 hash 的生成与校验**（`if (!is_role && (...))`）。

#### 表级与列级：`GRANT_TABLE`

```cpp
class GRANT_NAME {                       // tables_priv / procs_priv 公共基类
 public:
  ACL_HOST_AND_IP host;
  char *db;
  const char *user;
  char *tname;
  Access_bitmask privs;                  // 表级（或 routine 级）权限位图
  ulong sort;
  std::string hash_key;
};

class GRANT_TABLE : public GRANT_NAME {
 public:
  Access_bitmask cols;                   // ★ 该表所有列权限的 OR（"值不值得去查列"的快判）
  collation_unordered_multimap<std::string,
                               unique_ptr_destroy_only<GRANT_COLUMN>> hash_columns;
  bool ok() override { return privs != 0 || cols != 0; }
};

class GRANT_COLUMN {
 public:
  Access_bitmask rights;                 // 该列的权限位图
  std::string column;
};
```

★ **`cols` 不是"每一列的位图"，而是并集**——它的唯一用途是快速判否：`want_access & ~(grant_table->cols | grant->privilege)` 非 0 就说明"即使把所有列级的权限都借上也不够"，直接拒绝（源码注释写 `// impossible`）。真正的每列权限在 `hash_columns` 里按列名精确查。

三级 key 的构造（`GRANT_NAME::set_user_details()`）：

```cpp
hash_key = user;
hash_key.push_back('\0');
hash_key.append(db);
hash_key.push_back('\0');
hash_key.append(tname);
hash_key.push_back('\0');
```

★ 用 `\0` 把三元组拼成一个 `std::string` 当 hash key，是这份代码里反复出现的手法（role 图顶点 key、default_roles 都这么干）——好处是不用写自定义 hasher，坏处是 key 里可能含 `\0` 时调试很痛苦。

#### `GRANT_INFO`：挂在每个 `Table_ref` 上的一级缓存

**★ 勘误**：`GRANT_INFO grant` 是 **`Table_ref`（即 `TABLE_LIST`）** 的成员（`sql/table.h`），**不是** `TABLE`，也**不是** `TABLE_SHARE`。8.0 把它从 `TABLE_SHARE` 移到 `Table_ref`，因为同一个 share 被不同用户/不同 `Table_ref` 打开时权限不同，放在 share 上是错的。

```cpp
struct GRANT_INFO {
  GRANT_TABLE *grant_table{nullptr};   // 二级缓存里查到的结果；同时是"要不要查列级"的开关
  uint version{0};                     // ★ 一级缓存的代际标记
  Access_bitmask privilege{0};         // 到目前为止"已满足"的权限位（各层不断 |=）
  GRANT_INTERNAL_INFO m_internal;      // I_S / P_S 内部表的权限状态
};
```

`version` 存在的理由很具体：它缓存的是指向二级缓存（`column_priv_hash`）中 `GRANT_TABLE` 的**裸指针**，而 `column_priv_hash` 会被 `grant_reload()` **整体替换**。如果缓存了旧指针而 hash 被重建，就悬空了。于是用全局计数器 `grant_version` 做代际校验：

```cpp
/* reload table if someone has modified any grants */
if (grant->version != grant_version) {
  grant->grant_table = table_hash_search(sctx->host().str, sctx->ip().str, db_name,
                                         sctx->priv_user().str, table_name, false);
  grant->version = grant_version;
}
```

于是形成了**三层缓存**（这是理解"GRANT 何时生效"的关键结构）：

```
L1  Table_ref::grant        （per-语句、per-表）   失效信号：grant->version != grant_version
L2  column_priv_hash        （全局，可被整体替换）  失效信号：被 replace 就地改 / 被 grant_reload 重建
L3  Acl_cache（LF_HASH）     （per-(用户, role集, 版本) 的展开快照）  失效信号：版本编进 key
```

#### grant 表：InnoDB 系统表，但不是 DD 核心表

10 张表由 `sql/auth/sql_user_table.h` 的枚举与 `ACL_TABLE_NAMES` 数组定义：

```cpp
typedef enum ACL_TABLES {
  TABLE_USER = 0, TABLE_DB, TABLE_TABLES_PRIV, TABLE_COLUMNS_PRIV,
  TABLE_PROCS_PRIV, TABLE_PROXIES_PRIV, TABLE_ROLE_EDGES,
  TABLE_DEFAULT_ROLES, TABLE_DYNAMIC_PRIV, TABLE_PASSWORD_HISTORY, LAST_ENTRY
} ACL_TABLES;

static const char *ACL_TABLE_NAMES[MAX_ACL_TABLE_NAMES] = {
    "user", "db", "tables_priv", "columns_priv", "procs_priv",
    "proxies_priv", "role_edges", "default_roles", "global_grants", "password_history"};
```

它们建在 `scripts/mysql_system_tables.sql`（脚本开头 `set default_storage_engine=InnoDB;`，每张表显式 `engine=InnoDB ... TABLESPACE=mysql`）。启动时 `check_engine_type_for_acl_table()` 会逐张校验存储引擎，非 InnoDB 只告警不阻止启动（为了让权限表升级能继续）。

**★ 与数据字典的关系（这是最容易被写错的一处）**，三条源码证据：

1. **在 DD 里注册为 `System_tables::Types::SYSTEM`**（`sql/dd/impl/system_registry.cc`）：
   ```cpp
   dd::System_tables::Types system = dd::System_tables::Types::SYSTEM;
   register_table("user", system);
   register_table("db", system);
   register_table("tables_priv", system);
   ...
   ```
   注意最后一个参数是 `nullptr`——即这些表**没有**对应的 `dd::Object_table` 子类（对比 DD 核心表是 `register_table<Catalogs>(core)` 这种带 C++ 定义的模板形式）。`sql/dd/impl/tables/` 下 30 个头文件里没有任何 grant 表。
2. **bootstrap 时显式跳过**（`sql/dd/impl/bootstrap/bootstrapper.cc`）：`if ((*it)->property() == System_tables::Types::SYSTEM) continue;`——DD 只把核心表的元数据写进 `mysql.tables`，grant 表的 DDL 是靠执行 `mysql_system_tables.sql` 的普通 SQL 建出来的。
3. **访问矩阵放行外部 DML/DDL**（`sql/dd/impl/dictionary_impl.cc`）：
   ```cpp
   /*  SYSTEM   |  X    X     X    X  |     ← 外部 DDL 与外部 DML 都放行
   */
   return (table_type == nullptr ||
           (*table_type == System_tables::Types::DDSE_PROTECTED && !is_ddl_statement) ||
           *table_type == System_tables::Types::SYSTEM);
   ```

**结论**：grant 表是**受 DD 保护的普通 InnoDB 表**——`is_system_table_name()` 对它们返回 true（所以不能随便 DROP/ALTER，会报 `ER_NO_SYSTEM_TABLE_ACCESS_FOR_TABLE`），但 server 对它们的读写**完全走 SQL 层**（`open_and_lock_tables()` + `RowIterator` / `ha_write_row`），**不经过 `dd::Table` 的对象 API**。所以它们不是"隐藏表"（`mysql.user` 对有权限的用户是可见的），也不是"DD 管理的表"。

表结构一致性由**独立于 DD 的机制**校验：`Acl_table_intact`（`System_table_intact` 子类）+ `check_acl_tables_intact()` + `mysql_acl_table_defs[]`。官方手册的说法是：对任何修改 grant 表的操作，服务器会检查表结构是否符合预期，不符则报错（需走升级流程）。

---

### 二、判定链：`check_access` 分层裁决

#### 自由函数 `check_access()`：db 参数的三重语义

声明在 `sql/auth/auth_common.h`（**没有重载**，全仓库仅一处声明 + 一处定义）：

```cpp
bool check_access(THD *thd, Access_bitmask want_access, const char *db,
                  Access_bitmask *save_priv,
                  GRANT_INTERNAL_INFO *grant_internal_info,
                  bool dont_check_global_grants, bool no_errors);
```

★ **`db` 为什么必须是裸 `const char *`**：因为它承载三种语义，其中一种是**指针哨兵**：

```cpp
const char *any_db = "*any*";  // Special symbol for check_access
...
if (db != any_db) { ... }        // 地址比较：内部 schema 规则只对真实库名生效
if (db == any_db) { return false; }   // "允许对任意库操作"的短路
```

三种语义：① 真实库名 → 走 db 级匹配；② `nullptr`/空串 → 配合 `thd->db()`，两者都空则报 `ER_NO_DB_ERROR`；③ `any_db`（`"*any*"` 这个**全局常量对象的地址**）→ 通配哨兵。改成 `std::string` 后 `db == any_db` 会变成值比较，`"*any*"` 这个合法库名会被误判——所以必须保留指针。

`dont_check_global_grants` 名字很误导，它其实兼任**两个**语义：① 真的跳过全局权限检查；② **"这是一条 db 级 GRANT 语句"**——因为 db 级 `GRANT` 的库名可以是通配符：

```cpp
const bool db_is_pattern = ((want_access & GRANT_ACL) && dont_check_global_grants);
```

#### 完整代码与逐段解释

```cpp
bool check_access(THD *thd, Access_bitmask want_access, const char *db,
                  Access_bitmask *save_priv,
                  GRANT_INTERNAL_INFO *grant_internal_info,
                  bool dont_check_global_grants, bool no_errors) {
  Security_context *sctx = thd->security_context();
  Access_bitmask db_access;
  const std::string &db_name = db ? db : "";

  const bool db_is_pattern =
      ((want_access & GRANT_ACL) && dont_check_global_grants);
  Access_bitmask dummy;

  /* (1) save_priv 兜底：调用方不关心就用一个局部 dummy */
  if (save_priv)
    *save_priv = 0;
  else {
    save_priv = &dummy;
    dummy = 0;
  }

  THD_STAGE_INFO(thd, stage_checking_permissions);

  /* (2) 完全没库可判 → ER_NO_DB_ERROR */
  if ((!db || !db[0]) && !thd->db().str && !dont_check_global_grants) {
    if (!no_errors) my_error(ER_NO_DB_ERROR, MYF(0));
    return true;
  }

  /* (3) 内部 schema（I_S / P_S / mysql）特权规则先行 */
  if (db != nullptr) {
    if (db != any_db) {
      const ACL_internal_schema_access *access;
      access = get_cached_schema_access(grant_internal_info, db);
      if (access) {
        switch (access->check(want_access, save_priv, false)) {
          case ACL_INTERNAL_ACCESS_GRANTED:
            return false;                       // 内部规则已全部放行
          case ACL_INTERNAL_ACCESS_DENIED:
            if (!no_errors) {
              my_error(ER_DBACCESS_DENIED_ERROR, MYF(0), sctx->priv_user().str,
                       sctx->priv_host().str, db);
            }
            return true;
          case ACL_INTERNAL_ACCESS_CHECK_GRANT:
            want_access &= ~(*save_priv);       // 已放行的位剔除，剩余继续走常规路径
            break;
        }
      }
    }
  }

  /* (4) 【全局短路】m_master_access 已足够 */
  if (sctx->check_access(want_access, db_name)) {
    /*
      1. 如果没有全局 SELECT，仍要取 db 级权限，才能处理
         UPDATE t1 SET a=1 WHERE b > 0 这类需要读+写的语句；
      2. 若目标库不是当前库，db 级权限要重新算。
    */
    if (!(sctx->check_access(SELECT_ACL, db_name))) {
      if (db && (!thd->db().str || db_is_pattern || strcmp(db, thd->db().str))) {
        if (sctx->get_active_roles()->size() > 0) {
          db_access = sctx->db_acl({db, strlen(db)}, db_is_pattern);   // role 路径：查 Acl_map
        } else {
          db_access = acl_get(thd, sctx->host().str, sctx->ip().str,
                              sctx->priv_user().str, db, db_is_pattern); // 普通路径：查 acl_dbs
        }
      } else {
        db_access = sctx->current_db_access();   // 当前库：用缓存的 m_db_access
      }
      *save_priv |= sctx->master_access(db_name) | db_access;
    } else
      *save_priv |= sctx->master_access(db_name);
    return false;                                 // ★ 通过
  }

  /* (5) 【只能全局授予的权限】全局没有 → 直接拒绝（SUPER 等） */
  if (((want_access & ~sctx->master_access(db_name)) & ~DB_ACLS) ||
      (!db && dont_check_global_grants)) {  // We can never grant this
    if (!no_errors) {
      if (thd->password == 2)
        my_error(ER_ACCESS_DENIED_NO_PASSWORD_ERROR, MYF(0),
                 sctx->priv_user().str, sctx->priv_host().str);
      else
        my_error(ER_ACCESS_DENIED_ERROR, MYF(0), sctx->priv_user().str,
                 sctx->priv_host().str,
                 (thd->password ? ER_THD(thd, ER_YES) : ER_THD(thd, ER_NO)));
    }
    return true;
  }

  /* (6) any_db 哨兵：任意库放行，save_priv 保持 0 */
  if (db == any_db) {
    return false;
  }

  /* (7) 取 db 级权限（未被全局覆盖的部分） */
  if (db && (!thd->db().str || db_is_pattern || strcmp(db, thd->db().str))) {
    if (sctx->get_active_roles()->size() > 0) {
      db_access = sctx->db_acl({db, strlen(db)}, db_is_pattern);
    } else {
      db_access = acl_get(thd, sctx->host().str, sctx->ip().str,
                          sctx->priv_user().str, db, db_is_pattern);
    }
  } else
    db_access = sctx->current_db_access();

  db_access = (db_access | sctx->master_access(db_name));   // 全局 ∪ db 级
  *save_priv |= db_access;

  /* (8) 是否需要继续下探到表/列/routine 级 */
  bool need_table_or_column_check =
      (want_access & (TABLE_ACLS | PROC_ACLS | db_access)) == want_access;

  if (((db_access & want_access) == want_access) ||
      (!dont_check_global_grants && need_table_or_column_check)) {
    return false;   // ★ 通过（但仍需 check_grant 复核表/列）
  }

  /* (9) 拒绝 */
  if (!no_errors)
    my_error(ER_DBACCESS_DENIED_ERROR, MYF(0), sctx->priv_user().str,
             sctx->priv_host().str,
             (db ? db : (thd->db().str ? thd->db().str : "unknown")));
  return true;
}
```

**逐段要点**：

- **(3) 内部 schema 先行**：`get_cached_schema_access()` 返回 I_S / P_S / `mysql` 库的特殊规则对象。它的三态返回值体现了"部分放行"：`GRANTED` 全放行、`DENIED` 直接拒、`CHECK_GRANT` 表示"我放了一部分，剩下的 `want_access &= ~(*save_priv)` 你继续判"。这是"逐层扣除"语义在内部表上的体现。
- **(4) 全局短路**：注意调用的是 `sctx->check_access(want_access, db_name)`（成员函数），它**带 db 名参数**是为了 partial_revokes（见下）。短路成功后还要补 db 级权限，因为 `UPDATE t1 SET a=1 WHERE b>0` 需要 `UPDATE` **且** `SELECT`——全局可能只给了 `UPDATE`。
- **(5) `~DB_ACLS` 的隐式作用域检查**：见前文对 `SUPER` 的说明。
- **(8) `need_table_or_column_check`**：这是全函数唯一一处"提前放行但仍需后续检查"。它的语义是"现在欠着的这些位，全部落在表级/routine 级可授予的范围内，或者已经被 db 级覆盖了" ⇒ 交给 `check_grant()` 去查 `column_priv_hash`。

#### `Security_context::check_access()`：全局短路谓词 + partial_revokes 的渗透点

```cpp
bool Security_context::check_access(Access_bitmask want_access,
                                    const std::string &db_name /* = "" */,
                                    bool match_any /* = false */) {
  if ((want_access & DB_ACLS) &&
      (is_access_restricted_on_db(want_access, db_name))) {
    return false;
  }
  return (match_any ? (m_master_access & want_access)
                    : ((m_master_access & want_access) == want_access));
}
```

`match_any=false` 是 AND（全部命中），`match_any=true` 是 OR（任一命中）——`check_global_access(thd, CREATE_ROLE_ACL | CREATE_USER_ACL)` 用的就是 OR。

**partial_revokes 的一半生效点在这里**：`master_access()` 带 db 名参数，返回的是**过滤后**的全局权限：

```cpp
Access_bitmask Security_context::master_access(const std::string &db_name) const {
  return filter_access(m_master_access, db_name);
}

Access_bitmask Security_context::filter_access(const Access_bitmask access,
                                               const std::string &db_name) const {
  Access_bitmask access_mask = access;
  auto &db_restrictions = m_restrictions.db();
  if (!db_restrictions.is_empty()) {
    Access_bitmask restrictions_mask;
    if (db_restrictions.find(db_name, restrictions_mask))
      access_mask = (access_mask & restrictions_mask) ^ access;   // == access & ~restrictions_mask
  }
  return access_mask;
}
```

★ 注意 `check_access()` 全程用的是 `sctx->master_access(db_name)` 而**不是**裸 `m_master_access`——这是 restrictions 能生效的关键。另一半生效点是 `sctx->check_access()` 里的 `is_access_restricted_on_db()`：只要过滤后的值与原值不同就直接返回 false（禁止走全局短路）。

#### `check_grant()`：两遍循环 + 延迟加锁

```cpp
bool check_grant(THD *thd, Access_bitmask want_access, Table_ref *tables,
                 bool any_combination_will_do, uint number, bool no_errors) {
  Table_ref *tl;
  Table_ref *const first_not_own_table = thd->lex->first_not_own_table();
  Security_context *sctx = thd->security_context();
  const Access_bitmask orig_want_access = want_access;
  std::vector<Table_ref *> tables_to_be_processed_further;
  assert(number > 0);

  /* ============ 第一遍：不需要 ACL cache 锁的快速路径 ============ */
  for (tl = tables; tl && number-- && tl != first_not_own_table;
       tl = tl->next_global) {
    Table_ref *const t_ref =
        tl->correspondent_table ? tl->correspondent_table : tl;
    sctx = (t_ref->security_ctx != nullptr) ? t_ref->security_ctx
                                            : thd->security_context();
    const char *db_name = t_ref->get_db_name();
    const ACL_internal_table_access *access = get_cached_table_access(
        &t_ref->grant.m_internal, db_name, t_ref->get_table_name());

    /* (1) I_S / P_S 内部表规则 */
    if (access) {
      switch (access->check(orig_want_access, &t_ref->grant.privilege,
                            any_combination_will_do)) {
        case ACL_INTERNAL_ACCESS_GRANTED:
          t_ref->grant.privilege |= TMP_TABLE_ACLS;   // 跳过后续列级检查
          continue;
        case ACL_INTERNAL_ACCESS_DENIED:
          goto err;
        case ACL_INTERNAL_ACCESS_CHECK_GRANT:
          break;
      }
    }

    /* (2) 全局权限已覆盖这一张表 → 跳过 */
    want_access = orig_want_access;
    want_access &= ~sctx->master_access(db_name);
    if (!want_access) continue;  // ok

    /* (3) check_access 阶段已把表级权限写进 grant.privilege 且够用 → 跳过 */
    if (!(~t_ref->grant.privilege & want_access) || t_ref->is_internal() ||
        t_ref->schema_table)
      continue;

    /* (4) 临时表：自动拥有 TMP_TABLE_ACLS */
    if (is_temporary_table(t_ref)) {
      t_ref->grant.privilege |= TMP_TABLE_ACLS;
      continue;
    }

    /* (5) role 路径：查 Acl_map（不需要 ACL cache 读锁） */
    if (sctx->get_active_roles()->size() != 0) {
      t_ref->grant.grant_table = nullptr;
      t_ref->grant.version = grant_version;

      Grant_table_aggregate aggr = sctx->table_and_column_acls(
          {t_ref->get_db_name(), strlen(t_ref->get_db_name())},
          {t_ref->get_table_name(), strlen(t_ref->get_table_name())});

      /* ★ any_combination_will_do：只要"任意列组合上有任意权限"就过
         （SHOW COLUMNS / SHOW INDEX 用） */
      if (any_combination_will_do && (aggr.cols != 0 || aggr.table_access != 0))
        continue;

      t_ref->grant.privilege |= aggr.table_access;
      if (!(~t_ref->grant.privilege & want_access)) continue;   // 表级够了

      /* 表级不够：看"列级权限的并集 cols"能不能覆盖剩下的位 */
      if (want_access & ~(aggr.cols | t_ref->grant.privilege)) {
        want_access &= ~(aggr.cols | t_ref->grant.privilege);
        goto err;
      }
    } else {
      /* (6) 非 role 路径：推迟到第二遍统一加锁处理 */
      tables_to_be_processed_further.push_back(tl);
    }
  }

  /* ============ 第二遍：一次性拿 ACL cache 读锁 ============ */
  if (!tables_to_be_processed_further.empty()) {
    tl = nullptr;
    Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::READ_MODE);
    if (!acl_cache_lock.lock(!no_errors)) return true;

    for (Table_ref *tl_tmp : tables_to_be_processed_further) {
      tl = tl_tmp;
      Table_ref *const t_ref =
          tl->correspondent_table ? tl->correspondent_table : tl;
      sctx = (t_ref->security_ctx != nullptr) ? t_ref->security_ctx
                                              : thd->security_context();
      const char *db_name = t_ref->get_db_name();
      want_access = orig_want_access;
      want_access &= ~sctx->master_access(db_name);
      assert(want_access != 0);

      GRANT_TABLE *grant_table = table_hash_search(
          sctx->host().str, sctx->ip().str, db_name, sctx->priv_user().str,
          t_ref->get_table_name(), false);

      if (!grant_table) {
        want_access &= ~t_ref->grant.privilege;
        goto err;  // No grants
      }

      if (any_combination_will_do) continue;   // ★ 同上语义

      t_ref->grant.grant_table = grant_table;  // Remember for column test
      t_ref->grant.version = grant_version;
      t_ref->grant.privilege |= grant_table->privs;

      if (!(~t_ref->grant.privilege & want_access)) continue;

      if (want_access & ~(grant_table->cols | t_ref->grant.privilege)) {
        want_access &= ~(grant_table->cols | t_ref->grant.privilege);
        goto err;  // impossible
      }
    }
  }
  return false;

err:
  if (!no_errors) {  // Not a silent skip of table
    char command[128];
    get_privilege_desc(command, sizeof(command), want_access);
    my_error(ER_TABLEACCESS_DENIED_ERROR, MYF(0), command,
             sctx->priv_user().str, sctx->host_or_ip().str,
             tl ? tl->get_table_name() : "unknown");
  }
  return true;
}
```

**四个必须讲清的点**：

1. **多张表之间是 AND**（for 循环里任一表失败即 `goto err`），**不是 OR**。表间的 OR 是 `check_some_access()` 那种"逐位试"的循环实现的。
2. ★ **`any_combination_will_do` 是"单张表内部的列组合 OR"，不是表间 OR**。语义是"只要在这张表的任意列组合上拥有任意相关权限即可"（`SHOW COLUMNS` / `SHOW INDEX` 用）。这个名字极易误读。
3. **两遍循环 + 延迟加锁**：第一遍处理所有不需要访问 ACL hash 的分支；需要 `table_hash_search()` 的推进 `tables_to_be_processed_further`，循环结束后**一次性**加 `Acl_cache_lock_guard(READ_MODE)`。这显著缩短了 MDL 持有时间（源码注释：*"It allows us to delay locking of ACL caches."*）。
4. **`grant_table->cols` 的快速判否**：见 [内存模型](#表级与列级grant_table)。

#### 列级：`check_grant_column()` 与"不泄露列名"的设计

```cpp
bool check_grant_column(THD *thd, GRANT_INFO *grant, const char *db_name,
                        const char *table_name, const char *name, size_t length,
                        Security_context *sctx, Access_bitmask want_privilege) {
  GRANT_TABLE *grant_table;
  GRANT_COLUMN *grant_column;

  // ★ 裁剪：表级/全局/db 级已经满足的位先剔掉
  want_privilege &= ~grant->privilege;
  if (!want_privilege) return false;  // Already checked

  Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::READ_MODE);
  if (!acl_cache_lock.lock()) return true;

  /* reload table if someone has modified any grants */
  if (sctx->get_active_roles()->size() != 0) {
    /* ---- role 路径：查 Acl_map 的列权限 map ---- */
    assert(grant->grant_table == nullptr);
    Grant_table_aggregate agg = sctx->table_and_column_acls(
        {(const char *)db_name, strlen(db_name)},
        {(const char *)table_name, strlen(table_name)});
    std::string q_name(name);
    Column_map::iterator it = agg.columns.find(q_name);
    if (it != agg.columns.end()) {
      if (!(~(it->second) & want_privilege)) return false;
    }
  } else {
    /* ---- 非 role 路径：查 column_priv_hash ---- */
    if (grant->version != grant_version) {
      grant->grant_table = table_hash_search(
          sctx->host().str, sctx->ip().str, db_name, sctx->priv_user().str,
          table_name, false);
      grant->version = grant_version;
    }
    if (!(grant_table = grant->grant_table)) goto err;

    grant_column = column_hash_search(grant_table, name, length);
    if (grant_column && !(~grant_column->rights & want_privilege)) {
      return false;
    }
  }

err:
  char command[128];
  get_privilege_desc(command, sizeof(command), want_privilege);
  my_error(ER_COLUMNACCESS_DENIED_ERROR, MYF(0), command, sctx->priv_user().str,
           sctx->host_or_ip().str, name, table_name);
  return true;
}
```

返回值是**纯 bool**（`false` 通过 / `true` 拒绝），**没有"部分通过"这个语义**。

★ **一个安全设计细节**：`check_grant_all_columns()` 里有个 `using_column_privileges` 标志——如果用户对该表**没有任何**列级权限，报错时用 `ER_TABLEACCESS_DENIED_ERROR`（只说表名），**不点名列名**；只有用户确实持有某些列权限时才用 `ER_COLUMNACCESS_DENIED_ERROR` 点名。这是防止通过报错信息探测表结构。

#### 上层包装族

| 函数 | 语义 |
|---|---|
| `check_table_access(thd, requirements, tables, any_combination_of_privileges_will_do, number, no_errors)` | 对表链做**完整**检查：`check_access()`（逐表）+ `check_grant()`（整体）。是"两件事绑一起"的最小正确组合。 |
| `check_single_table_access(thd, privilege, all_tables, no_errors)` | 单表版：`check_access` + `check_grant(..., number=1, ...)` |
| `check_one_table_access(thd, privilege, all_tables)` | 在 `check_single_table_access` 基础上，额外要求子查询里的表只需 `SELECT_ACL` |
| `check_some_access(thd, want_access, table)` | **逐位试**：`for (access = 1; access < want_access; access <<= 1)`，任一位通过即通过。用于 `SHOW CREATE TABLE` 等"有任意一个权限就该看见"的场景 |
| `check_global_access(thd, want_access)` | 全局权限 + 统一报错（`ER_SPECIFIC_ACCESS_DENIED_ERROR`），内部 `match_any = true` |
| `check_grant_db(thd, db, check_table_grant)` | 判断"用户在这个库里是否**沾点边**"（有任意表/列/routine 权限），用于 `SHOW DATABASES` / `SHOW TABLES` |
| `check_grant_routine(thd, want_access, procs, is_proc, no_errors)` | routine 级（`PROC_ACLS`） |
| `has_grant_role_privilege(thd, roles)` | **不在判定链上**：它是"能不能把 role 授予别人"的管理动作授权检查（`ROLE_ADMIN` / `SUPER` / `WITH ADMIN OPTION`） |

★ `check_table_access()` **没有** `want_access` 的拆分逻辑（很多资料误称有 `want_access & ~table_acls` 的拆分）。8.0.39 里它把完整的 `requirements` 原样交给 `check_access()` 和 `check_grant()`，"按表用途算 want_access" 的责任在**调用方**：

```cpp
// sql/sql_select.cc — Sql_cmd_dml::check_all_table_privileges()
Access_bitmask want_privilege = 0;
if (tr->is_inserted()) want_privilege |= INSERT_ACL;
if (tr->is_updated())  want_privilege |= UPDATE_ACL;
if (tr->is_deleted())  want_privilege |= DELETE_ACL;
if (want_privilege == 0) want_privilege = SELECT_ACL;
```

```cpp
// sql/sql_insert.cc — Sql_cmd_insert_base::precheck()
ulong privilege = INSERT_ACL | (duplicates == DUP_REPLACE ? DELETE_ACL : 0) |
                  (update_value_list.empty() ? 0 : UPDATE_ACL);
```

#### 错误码与"还欠着什么"的诊断

| 错误码 | 抛出点 | 说明 |
|---|---|---|
| `ER_NO_DB_ERROR` | `check_access()` (2) | 既无 db 参数也无当前库 |
| `ER_DBACCESS_DENIED_ERROR` | `check_access()` (3)(9) | 库级不足 / 内部 schema 拒绝 |
| `ER_ACCESS_DENIED_ERROR` | `check_access()` (5) | 请求了只能全局授予的权限（报的是"账号"级错误，不是库级） |
| `ER_TABLEACCESS_DENIED_ERROR` | `check_grant()` err / `check_grant_all_columns()` | 表级不足 |
| `ER_COLUMNACCESS_DENIED_ERROR` | `check_grant_column()` / `check_grant_all_columns()` | 列级不足（点名列名） |
| `ER_PROCACCESS_DENIED_ERROR` | `check_grant_routine()` | routine 级不足 |
| `ER_SPECIFIC_ACCESS_DENIED_ERROR` | `check_global_access()` | "you need (at least one of) the XXX privilege(s)" |

错误信息里那串权限名由 `get_privilege_desc()` 从 **`want_access` 被扣除后剩下的位**生成——所以报错信息天然是"你还缺什么"，这正是"逐层扣除"设计的副产品优势。

#### MDL 与权限检查的先后（纠正直觉）

`sql/sql_base.cc` 里 `open_table()` / `open_tables()` / `open_tables_for_query()` **没有任何** `check_access` / `check_grant` 调用（grep 只有 5 处命中，全在 `list_open_tables()`（SHOW OPEN TABLES 的实现）与 I_S 填充逻辑里，与开表路径无关）。`open_table()` 内的锁序是：

```cpp
/* 1) 只读事务里请求写锁 → 直接报错 */
/* 2) 若需要，先拿 GRL 保护锁（MDL_key::GLOBAL, MDL_INTENTION_EXCLUSIVE） */
/* 3) ★ 拿表级 MDL */
if (open_table_get_mdl_lock(thd, ot_ctx, table_list, flags, &mdl_ticket) || ...)
```

所以是**先权限、后 MDL**：权限在 `precheck()` 阶段（开表前）完成，MDL 在 `open_tables_for_query()` 里获取。这个顺序的意义很明显——没权限就不该为它拿锁、不该为它打开表。

---

### 三、角色：图、展开与激活

#### 角色图：boost::adjacency_list

```cpp
namespace boost {
enum vertex_acl_user_t { vertex_acl_user };
BOOST_INSTALL_PROPERTY(vertex, acl_user);
}  // namespace boost

typedef boost::property<boost::vertex_acl_user_t, ACL_USER,
                        boost::property<boost::vertex_name_t, std::string>>
    Role_properties;
typedef boost::property<boost::edge_capacity_t, int> Role_edge_properties;

/** A graph of all users/roles privilege inheritance */
typedef boost::adjacency_list<boost::setS,            // OutEdges：集合，重复加边幂等
                              boost::vecS,            // Vertices：向量
                              boost::bidirectionalS,  // 有向图，双向可遍历
                              Role_properties,
                              Role_edge_properties>
    Granted_roles_graph;

typedef std::unordered_map<std::string, Role_vertex_descriptor> Role_index_map;
```

三个全局变量：`g_granted_roles`（图）、`g_authid_to_vertex`（authid → 顶点，hash 查找）、`g_default_roles`（`unordered_multimap<Role_id, Role_id>`）。

**边方向**（容易搞错，已逐行核实）：`grant_role(ACL_USER *role, const ACL_USER *user, bool with_admin_opt)` 里 `add_edge(user_vert, role_vert)`，即 **grantee → granted role**。所以 `out_edges(用户顶点)` = 授予给该用户的角色，`in_degree(某顶点) > 0` ⇒ 它是 role。`get_granted_roles()` 用的是 `boost::out_edges`：

```cpp
void get_granted_roles(Role_vertex_descriptor &v,
                       std::function<void(const Role_id &, bool)> f) {
  boost::graph_traits<Granted_roles_graph>::out_edge_iterator ei, ei_end;
  boost::tie(ei, ei_end) = boost::out_edges(v, *g_granted_roles);
  for (; ei != ei_end; ++ei) {
    ACL_USER acl_user = get(boost::vertex_acl_user_t(),
                            *g_granted_roles)[boost::target(*ei, *g_granted_roles)];
    int with_admin_opt = boost::get(boost::edge_capacity_t(), *g_granted_roles)[*ei];
    ...
    f(id, with_admin_opt != 0);
  }
}
```

`mysql.role_edges` 表的列是 `FROM_*`（**被授予的 role**）/ `TO_*`（**grantee**），`WITH_ADMIN_OPTION`。`populate_roles_caches()` 读表时正是 `acl_role = find_acl_user(from_*)`、`acl_user = find_acl_user(to_*)`，然后 `grant_role(acl_role, acl_user, ...)`——与内存图方向一致。

**user→role 与 role→role 用同一张图、同一类边表示**，不做区分——这是 RBAC1 角色层次的自然表达。`WITH ADMIN OPTION` 存在边的 `edge_capacity_t` 属性上（`int`，1 = with admin）。

#### 展开：BFS + visitor 累加

`Acl_map` 的构造就是这一次昂贵计算：

```cpp
Acl_map::Acl_map(Security_context *sctx, uint64 ver)
    : m_reference_count(0), m_version(ver), m_restrictions() {
  Acl_cache_lock_guard acl_cache_lock(current_thd, Acl_cache_lock_mode::READ_MODE);
  if (!acl_cache_lock.lock(false)) { ...; return; }
  m_global_acl = 0;
  ACL_USER *acl_user = find_acl_user(sctx->priv_host().str, sctx->priv_user().str, true);
  if (acl_user == nullptr) { ...; return; }
  List_of_granted_roles granted_roles;
  get_privilege_access_maps(
      acl_user, sctx->get_active_roles(), &m_global_acl, &m_db_acls,
      &m_db_wild_acls, &m_table_acls, &m_sp_acls, &m_func_acls, &granted_roles,
      &m_with_admin_acls, &m_dynamic_privileges, m_restrictions);
}
```

`get_privilege_access_maps()` 是展开的主函数，流程：

```
① 先把用户自己的权限铺底
   *access = acl_user->access;                      全局
   get_database_access_map(...)                     db / db_wild
   get_table_access_map(...)                        表 + 列
   get_sp_access_map(..., proc_priv_hash)           存储过程
   get_sp_access_map(..., func_priv_hash)           函数
   get_dynamic_privileges(...)                      动态权限
   restrictions = acl_restrictions->find_restrictions(acl_user);

② if (acl_user->user == nullptr) return;            ★ 匿名账户不支持角色层次

③ 收集三份集合
   all_granted_roles  = 直接授予的角色(out_edges 单层) ∪ mandatory_roles
   all_active_roles   = sctx->m_active_roles（或 SHOW GRANTS 的 USING 列表）
   granted_active_roles = 两者交集          ★ 安全校验：只有"确实被授予且已激活"的才展开

④ 对交集中每个角色顶点做 BFS（Get_access_maps visitor 累加权限）
```

★ 第 ③ 步的**交集**是安全关键：`USING` 子句或 `SET ROLE` 里写一个没被授予的角色，不会获得它的权限（显式 `USING` 还会直接报 `ER_ROLE_NOT_GRANTED`）。

visitor：

```cpp
class Get_access_maps : public boost::default_bfs_visitor {
 public:
  template <typename Vertex, typename Graph>
  void discover_vertex(Vertex u, const Graph &) const {
    ACL_USER acl_user = get(boost::vertex_acl_user_t(), *g_granted_roles)[u];
    if (acl_user.user == g_active_dummy_user) return;  // skip root node

    get_database_access_map(&acl_user, m_db_map, m_db_wild_map);

    /* Add restrictions —— role 的限制要传播给 grantee */
    {
      const Auth_id granter(&acl_user);
      Restrictions restrictions = acl_restrictions->find_restrictions(&acl_user);
      std::unique_ptr<Restrictions_aggregator> aggregator =
          Restrictions_aggregator_factory::create(
              granter, m_grantee, acl_user.access & DB_ACLS,
              *m_access & DB_ACLS, restrictions.db(), m_restrictions->db(),
              acl_user.access & DB_ACLS, m_db_map);
      if (aggregator) {
        DB_restrictions db_restrictions;
        if (aggregator->generate(db_restrictions)) return;
        m_restrictions->set_db(db_restrictions);
      }
    }

    /* Up-cast to base class to avoid gcc 7.1.1 warning:
       dereferencing type-punned pointer will break strict-aliasing rules */
    *m_access |= implicit_cast<ACL_ACCESS *>(&acl_user)->access;

    get_table_access_map(&acl_user, m_table_map);
    get_sp_access_map(&acl_user, m_sp_map, proc_priv_hash.get());
    get_sp_access_map(&acl_user, m_func_map, func_priv_hash.get());
    get_dynamic_privileges(&acl_user, m_dynamic_acl);
  }

  template <typename Edge, typename Graph>
  void examine_edge(const Edge &edge, Graph &granted_roles) {
    ACL_USER to_user = boost::get(boost::vertex_acl_user_t(),
                                  granted_roles)[boost::target(edge, granted_roles)];
    int with_admin_opt = boost::get(boost::edge_capacity_t(), granted_roles)[edge];
    if (with_admin_opt) {
      /* 记录 "WITH ADMIN" 的角色名，供 has_with_admin_acl() 用 */
      m_with_admin_acl->insert(std::string(qname.c_ptr_quick(), qname.length()));
    }
  }
};
```

**权限合并一律是 `|=`（并集）**——角色只会增加权限，永远不会减少（除了 restrictions）。

**★ 角色循环继承如何被处理（两层防线，性质完全不同）**：

- **GRANT 期：主动拒绝成环**。`mysql_grant_role()` 在加边前调 `check_if_granted_role_recursive(role, role_host, user, user_host)`：从**待授予的 role** 出发做 BFS，若能在它的后代里找到 **grantee**，则会成环 → 报 `ER_ROLE_GRANTED_TO_ITSELF`（"…The GRANT would create a loop"）。同时也会检查 `mandatory_roles` 的后代。
- **展开期：靠 BFS 的 color map 天然免疫**。`boost::vector_property_map<boost::default_color_type> v_color` 保证每个顶点只 discover 一次。即使图里有环（例如手工往 `mysql.role_edges` 插记录绕过 GRANT 期检查），也**不会死循环、没有深度限制、不报错**。

还有一个**源码级的怪癖**（值得记一笔）：源码里 `vertex_count` 的 `++` 写在 `else` 分支，而初值 0 时只走 `if` 分支 ⇒ `vertex_count` 永远是 0 ⇒ `breadth_first_visit` 分支是**死代码**，每个激活角色都会重新初始化一次 color map。功能没错（BFS 语义一致），但说明这段多角色优化没生效。

#### ★ 图的构建时机：全量重建 vs 增量改边

| 时机 | 做法 | 位置 |
|---|---|---|
| 启动 / `FLUSH PRIVILEGES`（`acl_reload()`） | 换出旧图 → `roles_init()` 建新图 → **`populate_roles_caches()`** 全表重扫 | `sql_auth_cache.cc` |
| `DROP USER` / `DROP ROLE` / `RENAME USER` | **`populate_roles_caches()`** 全量重建（rename 另加 `roles_rename_authid()`） | `sql_user.cc` |
| **`GRANT r TO u`** | **增量**：`grant_role()` 直接 `add_edge`，不重扫表 | `sql_authorization.cc` |
| **`REVOKE r FROM u`** | **增量**：`revoke_role()` → `remove_edge()` + `update_role_flag_of_acl_user()` | `sql_authorization.cc` |
| 顶点删除导致 descriptor 失效 | `rebuild_vertex_index()` 重建 `g_authid_to_vertex` | `sql_authorization.cc` |

★ 读表重建的**唯一入口是 `populate_roles_caches()`**（`role_tables.cc`）——★ **`Roles::read_role_edges_from_table()` 这个名字不存在**，别被旧资料误导。

#### ★ 边的方向：`role_edges` 列方向与图边方向**相反**

这是最容易搞混的一处：

```sql
-- scripts/mysql_system_tables.sql
CREATE TABLE role_edges (FROM_HOST, FROM_USER, TO_HOST, TO_USER, WITH_ADMIN_OPTION, ...)
--                        ↑ 角色                ↑ 被授予者(grantee)
```

而图里建边是 **`add_edge(user_vert, role_vert)`**（`grant_role()`），即 **grantee → role**：

```cpp
  acl_role = find_acl_user(from_host, from_user, true);   // FROM → 角色
  acl_user = find_acl_user(to_host,   to_user,   true);   // TO   → 被授予者
  ...
  grant_role(acl_role, acl_user, *with_admin_opt == 'Y');
  ...
  tie(edge, is_added) = add_edge(user_vert, role_vert, *g_granted_roles);
```

⇒ **表列方向 FROM→TO = 角色→被授予者；图边方向 = 被授予者→角色**。BFS 沿 **out_edges** 前进（`breadth_first_search` 默认），所以遍历方向是 **grantee → role → role's role → …**。

**传递性实例**：`GRANT B TO A; GRANT C TO B;` ⇒ 边 `A→B`、`B→C` ⇒ 激活 A 的用户得到 **A+B+C** 全部权限；反过来只激活 B **得不到** A 的权限。

> `boost::bidirectionalS` 唯一被用到的地方是 `in_degree`——用来推导 `is_role`（有入边 = 被别人授予 = 是角色，见下文）。

#### ★ 展开算法：交集规则 + 多源 BFS

```cpp
void get_privilege_access_maps(ACL_USER *acl_user,
                               const List_of_auth_id_refs *using_roles, ...) {
  /* ① 先取用户自己的权限（九份 map） */
  *access = acl_user->access;
  get_database_access_map(acl_user, db_map, db_wild_map);
  get_table_access_map(acl_user, table_map);
  get_sp_access_map(acl_user, sp_map, proc_priv_hash.get());
  get_sp_access_map(acl_user, func_map, func_priv_hash.get());
  get_dynamic_privileges(acl_user, dynamic_acl);
  restrictions = acl_restrictions->find_restrictions(acl_user);
  if (acl_user->user == nullptr) return;    // ★ 匿名账号不支持角色层次

  get_mandatory_roles(&mandatory_roles);
  boost::vector_property_map<boost::default_color_type> v_color(
      boost::num_vertices(*g_granted_roles));
  Get_access_maps vis(...);

  if (has_granted_roles || mandatory_roles.size() > 0) {
    get_granted_roles(user_vertex, [&](const Role_id &rid, bool with_admin) {
      all_granted_roles.insert(rid); granted_roles->push_back({rid, with_admin});
    });
    for (auto &rid : mandatory_roles) all_granted_roles.insert(rid);  // ★ mandatory 视为"已授予"
    for (auto &rid : *using_roles)    all_active_roles.insert(id);

    std::set_intersection(                      // ★★ 交集规则就在这里
        all_granted_roles.begin(), all_granted_roles.end(),
        all_active_roles.begin(),  all_active_roles.end(),
        std::inserter(granted_active_roles, granted_active_roles.begin()));

    int vertex_count = 0;
    for (auto &&rid : granted_active_roles) {
      ... 角色已被删 → 警告 ER_UNKNOWN_AUTHID 并跳过 ...
      if (vertex_count == 0)
        boost::breadth_first_search(*g_granted_roles, active_role_vertex,
                                    boost::color_map(v_color).visitor(vis));
      else
        boost::breadth_first_visit(*g_granted_roles, active_role_vertex,
                                   boost::color_map(v_color).visitor(vis));
      ++vertex_count;
      /* 该边若带 WITH ADMIN，收集进 with_admin_acl */ ...
    }
  }
}
```

★ 四个要点：

1. **★ 起点的公式**：`(直接授予的角色 ∪ mandatory_roles) ∩ 会话已激活的角色`——`std::set_intersection()` 这一行就是"**被授予 ∩ 已激活**"规则的落点。
2. **`all_granted_roles` 只含直接授予**（`get_granted_roles()` 只走 `out_edges`）。**传递角色不是靠加进起点实现的，而是靠 BFS 走下去**——所以传递权限天然被覆盖。
3. **★ 多源 BFS 的两个 API**：第一个起点用 `breadth_first_search`（**会初始化** color map），后续起点用 `breadth_first_visit`（**不初始化**，复用同一个 color map）⇒ 多个起点合起来每个顶点仍只展开一次，既不重复累加也不死循环。
4. **九份 map，不是六份**：`Get_access_maps::discover_vertex()` 合并的是 global / db / db_wild / table / sp / func / with_admin / dynamic / restrictions —— 全部是**按位或 `|=`**，即标准 RBAC 的"权限并集"，**无减法**（唯一例外是 partial_revokes 的 `Restrictions_aggregator`）。

> ★ 聚合函数真名是 **`get_privilege_access_maps()`**（`sql_authorization.cc`）——★ **`acl_get_access_maps()` 不存在**。

#### 展开结果如何落到 `Security_context`

```
Security_context::checkout_access_maps()
   → Acl_cache::checkout_acl_map(this, uid, m_active_roles)
        → Acl_cache::create_acl_map(version, sctx)
             → Acl_map::Acl_map(sctx, ver)
                  → get_privilege_access_maps(...)      ← BFS 展开
   → set_master_access(m_acl_map->global_acl(), m_acl_map->restrictions())
```

```cpp
void Security_context::checkout_access_maps(void) {
  if (m_acl_map != nullptr) {
    get_global_acl_cache()->return_acl_map(m_acl_map);   // 先归还旧 map
    m_acl_map = nullptr;
  }

  if (m_active_roles.size() == 0) return;                // ★ 没有激活 role 就什么都不做

  Auth_id_ref uid;
  uid.first.str = this->m_priv_user; ...
  m_acl_map = get_global_acl_cache()->checkout_acl_map(this, uid, m_active_roles);
  if (m_acl_map != nullptr) {
    set_master_access(m_acl_map->global_acl(), m_acl_map->restrictions());
  } else {
    set_master_access(0);
  }
}
```

★ **`set_master_access(m_acl_map->global_acl(), ...)` 这一行就是"role 是预先展开的"的全部答案**：展开后的全局权限**直接覆盖** `m_master_access`，判定时刻不再递归查图。这也是为什么"有 active role 的会话"在 `GRANT` 后会重算全局权限——因为版本号变了，checkout 会拿到新的 `Acl_map`。

**调用时机**：`THD::reset_for_next_command()`（**每条语句开始时**）会调 `checkout_access_maps()`。所以 role/动态权限的生效粒度是**语句级**——这也解释了源码注释里那句 "the granularity of after effects of grant/revoke or flush due to roles is **per statement**"。

**判定时刻的"两条路"**：展开完成后，判定不再走图，而是靠 `sctx->get_active_roles()->size() > 0` 这个布尔开关在两套数据源间切换：

| | 无激活 role | 有激活 role |
|---|---|---|
| 全局 | `m_master_access`（来自 `mysql.user` 单行） | 同样是 `m_master_access`（已被 Acl_map 覆盖） |
| db 级 | `acl_get()` → `acl_dbs` / `db_cache` | `sctx->db_acl()` → `Acl_map::db_acls()` |
| 表/列级 | `table_hash_search()` → `column_priv_hash` | `sctx->table_and_column_acls()` → `Acl_map::table_acls()` |
| routine 级 | `routine_hash_search()` → `proc_priv_hash` | `sctx->procedure_acl()` / `function_acl()` |

#### 激活：`SET ROLE` / 默认角色 / mandatory_roles

`SET ROLE` **不在** `mysql_execute_command` 的 switch 里，而是走 `Sql_cmd::execute` 分发（`sql/sql_admin.cc`）：

```cpp
bool Sql_cmd_set_role::execute(THD *thd) {
  switch (role_type) {
    case role_enum::ROLE_NONE:    ret = mysql_set_active_role_none(thd); break;
    case role_enum::ROLE_DEFAULT: ret = mysql_set_role_default(thd); break;
    case role_enum::ROLE_ALL:     ret = mysql_set_active_role_all(thd, except_roles); break;
    case role_enum::ROLE_NAME:    ret = mysql_set_active_role(thd, role_list); break;
  }
  if (!ret) {
    set_system_user_flag(thd, true);      // ★ 激活后要重算 SYSTEM_USER 标志
    set_connection_admin_flag(thd, true);
  }
  return ret;
}
```

四个分支共用 `Roles::Role_activation`（`sql/auth/roles.cc`）的骨架：**备份 active_roles → 清空 → 逐个 `sctx->activate_role()` → 成功则 `checkout_access_maps()` + `cache_current_db_access()` 并释放备份内存；失败则回滚**。`activate_role_none()` 是唯一不 checkout 的分支（active roles 为空时 checkout 直接 return），它改为从 `acl_users` 重新取 master access。

**★ `SET ROLE` 不需要任何权限**（只需该角色确实被授予给你或在 `mandatory_roles` 里）。判定在 `Security_context::activate_role()` → `check_if_granted_role()` → `find_if_granted_role()`（遍历 out_edges）+ mandatory_roles 兜底。失败报 `ER_ROLE_NOT_GRANTED`。`ROLE_ADMIN` / `WITH ADMIN OPTION` 只在 **`GRANT role` / `REVOKE role`** 时才要求（`has_grant_role_privilege()`）。

**三个激活开关的系统变量**（`sql/sys_vars.cc` 核实）：

| 变量 | 默认值 | 生效点 |
|---|---|---|
| `activate_all_roles_on_login` | **`false`** | 登录认证成功后：`opt_always_activate_granted_roles` 为真则 `activate_all_granted_and_mandatory_roles(acl_user, sctx)`，否则只激活 `mysql.default_roles` 里的默认角色 |
| `mandatory_roles` | **`""`（空串）** | `get_mandatory_roles()` 惰性解析 + 缓存（`opt_mandatory_roles_cache`）；需要 `ROLE_ADMIN` **外加** `SYSTEM_VARIABLES_ADMIN` 才能设置 |
| （无变量）默认角色 | — | `mysql.default_roles` 表，登录时 `acl_getroot()` 里激活 |

**`mandatory_roles` 为什么会"意外生效"**（四条机制叠加，这是运维最容易踩的坑）：

1. **它不写 `mysql.role_edges`**（变量注释明说 "The granted roles will not be visible in the mysql.role_edges table"），只在内存里生效；但 `get_privilege_access_maps()` 把它塞进 `all_granted_roles`，从而**能通过"被授予 ∩ 已激活"的交集**。用户只要 `SET ROLE ALL` / 开 `activate_all_roles_on_login` / `SET DEFAULT ROLE ALL`，权限就立刻生效。
2. **`check_if_granted_role()` 把 mandatory role 当作"已授予"** ⇒ `SET ROLE mandatory_role` 即使没 `GRANT` 也能通过。
3. **解析是惰性 + 缓存的**：`opt_mandatory_roles_cache=false` 时只在下一次 `get_mandatory_roles()` 才重新解析；**解析时角色不存在就静默丢弃并只写 warning**（`ER_UNKNOWN_AUTH_ID_IN_MANDATORY_ROLE` / `ER_ANONYMOUS_AUTH_ID_NOT_ALLOWED_IN_MANDATORY_ROLES`）。所以"先设 `mandatory_roles` 再 `CREATE ROLE`"或拼错名字，只有 error log 告警；等到某个 DDL/登录触发重新解析后才"突然"生效。
4. **`DROP` / `REVOKE` 被硬性拦截**（`ER_MANDATORY_ROLE`），且带 `SYSTEM_USER` 的 role **不能**进 `mandatory_roles`（防权限提升；启动时配置会导致服务器写 error log 并**退出**）。

#### ★ RBAC2 的真相：SSD / DSD / 基数约束，**一个都没有**

这一节专门澄清"MySQL 到底实现了 RBAC2 的哪些约束"。

**① SSD（静态职责分离 / 互斥角色）—— 没有**

负面证据（在 `sql/auth/`——RBAC 实现的全部所在目录——grep 以下关键词，**全部 0 命中**）：

```
separation of duty / separation_of_duty / conflict of interest / SoD
mutually exclusive / exclusive role / forbidden role / incompatible
conflicting role / 互斥 / 职责分离
```

实际命中的只有 5 条**完全无关**的结果（认证插件"incompatible with"、密码策略"incompatible with policy"、JSON 的 `cardinality()` 等）。全仓（含 `share/messages_to_clients.txt`）搜 `conflict.*role` / `exclusive.*role` / `too many role`：**0 命中**。

★ **比 grep 更强的结构性证据**（三条）：

1. `Granted_roles_graph` 的**边属性只有一个 `int`**（`edge_capacity_t`，就是 WITH_ADMIN_OPTION），**顶点属性只有 `ACL_USER` + name**——**根本没有存放"互斥关系"的数据槽位**。
2. `grant_role()` 全函数只有 40 行，加了边就结束，**没有任何校验**。
3. 角色相关错误码共 13 个（`ER_FAILED_ROLE_GRANT` / `ER_ROLE_NOT_GRANTED` / `ER_ROLE_GRANTED_TO_ITSELF` / `ER_MANDATORY_ROLE` …），**没有任何一个是互斥或数量上限**。

**② DSD（动态职责分离 / 会话内互斥）—— 没有**

`Security_context::activate_role()` 里**唯一的校验**是 `check_if_granted_role()`（"是否已被授予"），没有任何"与已激活角色冲突"的判定；BFS 结果一律 `|=` 合并。⇒ 一次会话可以同时激活任意多个已授予角色。

**③ 基数约束 —— 没有**

grep `cardinality`/`max_role`/`limit` 在 `sql/auth/` 无相关命中；`grant_role()` 不查成员数，`activate_role()` 不查激活个数；边容器 `boost::setS`、顶点 `vecS`、`unordered_map` **全部无界**。

**④ 那 MySQL 实际拥有的约束是什么**：

| 约束 | 实现 |
|---|---|
| **激活约束** | `check_if_granted_role()`：只能激活已（直接）授予或 mandatory 的角色，否则 `ER_ROLE_NOT_GRANTED` 并**整条语句回滚**（`clear_active_roles()` + 还原 `backup_active_list`） |
| ★ **委托/管理约束**（`WITH ADMIN OPTION`，标准 RBAC 之外的 ARBAC 风格） | `has_grant_role_privilege()` 三条任一：① `ROLE_ADMIN` 动态权限或 `SUPER`；② 对该角色被 `WITH ADMIN` 授予（沿角色图传播，由 `Get_access_maps::examine_edge()` 收集进 `with_admin_acl`）；③ 兜底查 `role_edges.WITH_ADMIN_OPTION='Y'`。失败报 `ER_SPECIFIC_ACCESS_DENIED_ERROR` "WITH ADMIN, ROLE_ADMIN, SUPER" |
| **强制角色** | `mandatory_roles`（见下） |
| **默认角色** | `mysql.default_roles` + 登录后 `get_default_roles()` 逐个 `activate_role()`；任一失败则 `clear_active_roles()` **全部回滚** |
| **结构约束：禁止成环** | `check_if_granted_role_recursive()` → `ER_ROLE_GRANTED_TO_ITSELF`。★ 这是**唯一的图结构约束**，目的是保证无环，**不是 SoD** |

> ★ 一个易忽略的坑：`check_if_granted_role()` **只走 out_edges（直接授予）**，所以**通过角色层次间接获得的角色不能单独 `SET ROLE`**（虽然它的权限会通过 BFS 自动带出来）。

#### ★ `mandatory_roles` 的完整规则（6 条，含一处既往结论更正）

| # | 规则 | 后果 |
|---|---|---|
| ① | **不能是匿名 authid**（`user=''`） | ★ **启动时 `grant_init()` 失败 → `unireg_abort` 服务器退出**；运行期 `SET GLOBAL` 被拒 |
| ② | **不能有 `SYSTEM_USER` 权限**（且**沿角色层次递归判定**） | 运行期 `SET GLOBAL` **直接拒绝**（`ER_DA_AUTH_ID_WITH_SYSTEM_USER_PRIV_IN_MANDATORY_ROLES`）；**启动期只写 WARNING 并把整个变量清空** |
| ③ | **不能给 mandatory role 授予 `SYSTEM_USER`** | `ER_CANNOT_GRANT_SYSTEM_PRIV_TO_MANDATORY_ROLE` |
| ④ | **不能被 DROP / REVOKE** | `ER_MANDATORY_ROLE`（`mysql_drop_user()` 与 `mysql_revoke_role()` 各一处检查） |
| ⑤ | **不写 `mysql.role_edges`**，也不出现在 GRANT 出来的角色列表里 | 只在内存生效；由 `get_mandatory_roles()` 塞进 `all_granted_roles`，因而**可以绕过 role_edges 直接被 `SET ROLE` 激活** |
| ⑥ | host 缺省补 `'%'`；角色必须已存在 | 不存在则 WARNING `ER_UNKNOWN_AUTH_ID_IN_MANDATORY_ROLE` 并**静默跳过** |

★★ **★ 更正一处既往结论**：此前写过的"带 `SYSTEM_USER` 的 role 进 `mandatory_roles` 会导致服务器启动退出"**在 8.0.39 不成立**。`check_authorization_id_string()` 是分支处理的：

```cpp
    if (el.first == "") {                      // ① 匿名：不看 mysqld_server_started，直接 error
      error = true;
    } else if (sctx->has_global_grant(..., consts::system_user, true).first) {
      if (mysqld_server_started) {              // ② 运行期
        my_error(ER_DA_AUTH_ID_WITH_SYSTEM_USER_PRIV_IN_MANDATORY_ROLES, ...);
        error = true;
      } else {                                  // ② 启动期：只警告 + 清空
        LogErr(WARNING_LEVEL, ER_WARN_AUTH_ID_WITH_SYSTEM_USER_PRIV_IN_MANDATORY_ROLES, ...);
        mandatory_roles.str = empty_c_string;
        mandatory_roles.length = 0;
      }
    }
```

⇒ **启动期是 WARNING + 静默清空，服务器照常启动**（mysql-test 的 `system_user_priv.test` 也是按"重启后日志里有 warning"断言的）；**真正会导致启动退出的是 ① 匿名 authid**。

（另：`SET PERSIST_ONLY` **不校验**，所以可以把非法值写进配置文件，下次启动被清空——这是个隐蔽的运维坑。）

#### ★ 会话激活是**会话级**，但重算粒度是**每语句**

- `m_active_roles` 挂在会话的 `Security_context` 上（`List_of_auth_id_refs`，元素是 strdup 深拷贝）。**一旦 `SET ROLE` 成功就持续到连接结束**，中途不被任何语句重置；仅在 `SET ROLE`（重建）、`SET ROLE NONE`（`clear_active_roles()`）、连接结束/`change_user`（`logout()`/`destroy()`）时变化。
- ★ 但**"GRANT/REVOKE/FLUSH 之后权限何时重新生效"是每语句粒度**——`Acl_cache` 的 version 在 `mysql_grant_role()` 结尾 `increase_version()`，而 `checkout_access_maps()` **每条语句前**都检查版本，变了就换一份新 `Acl_map`（重跑一遍 `get_privilege_access_maps()`）。源码注释原文：*"the granularity of after effects of grant/revoke or flush due to roles is per statement"*。

#### ★ 角色的 DDL 生命周期与两张系统表

上面三节讲的是**运行期**（图怎么建、怎么展开、怎么激活）。这一节补**写入期**——角色是怎么被造出来、连起来、删掉的。

**① `CREATE ROLE` = 造一个"被锁住的用户"**

★ 这是回答"role 为什么不能登录"的源码答案——**没有任何专门的角色存储结构，它就是一行 `account_locked = true` 且无凭据的用户**。`Sql_cmd_create_role::execute()`（`sql/sql_admin.cc`）只是把 LEX 预置好再复用 `mysql_create_user()`，注释自己都承认这是 hack：

```cpp
bool Sql_cmd_create_role::execute(THD *thd) {
  if (check_global_access(thd, CREATE_ROLE_ACL | CREATE_USER_ACL)) return true;
  /*
    Roles must be locked for authentication by default.
    The below is a hack to make mysql_create_user() behave
    correctly.
  */
  thd->lex->alter_password.account_locked = true;
  thd->lex->alter_password.update_account_locked_column = true;
  thd->lex->alter_password.expire_after_days = 0;
  thd->lex->alter_password.update_password_expired_column = true;
  ...
  while ((role = it++)) {
    role->first_factor_auth_info.uses_identified_by_clause = false;      // 不设口令
    role->first_factor_auth_info.auth = {};
    role->alter_status.account_locked = true;
    role->alter_status.update_account_locked_column = true;
    ...
  }
  return mysql_create_user(thd, *roles, if_not_exists, true) == nullptr;
}
```

⇒ 所以 `CREATE ROLE` 等价于 `CREATE USER ... ACCOUNT LOCK`，而"role 不能登录"是靠 `account_locked` 而非某个 role 专属标志；`ACL_USER::is_role` 是**从图推导的拓扑属性，不落盘**（见「`ACL_USER`：role 也是一种 user」一节）。

**② `DROP ROLE`：一段"反推权限来源"的技巧**

```cpp
bool Sql_cmd_drop_role::execute(THD *thd) {
  /*
    We want to do extra checks (if user login is disabled) when holding a
    using DROP_ROLE privilege.
    To do that we record if CREATE USER was granted.
    Then if one of DROP ROLE or CREATE USER was granted (the original
    requirement) and CREATE USER was not granted we know that it was DROP ROLE
    that caused the check to pass.

    Thus we raise the flag (drop_role) in this case.
  */
  bool on_create_user_priv = thd->security_context()->check_access(CREATE_USER_ACL, "", true);
  if (check_global_access(thd, DROP_ROLE_ACL | CREATE_USER_ACL)) return true;
  if (mysql_drop_user(thd, *roles, ignore_errors, !on_create_user_priv)) return true;
  my_ok(thd);
  return false;
}
```

★ 因为权限要求是 **`DROP_ROLE_ACL | CREATE_USER_ACL`（或关系）**，通过检查后**无法区分是凭哪个通过的**。于是先单独测一次 `CREATE_USER_ACL`，把结果取反传给 `mysql_drop_user` 当 `drop_role` 标志——**凭 DROP ROLE 通过时额外做"目标是否可登录"的检查**，防止用 `DROP ROLE` 权限去删一个真实用户。`CREATE ROLE` 侧没有这个技巧，因为它只新建不删。

**③ 两张系统表：`role_edges`（边）与 `default_roles`（默认激活）**

```cpp
enum mysql_role_edges_table_field {
  MYSQL_ROLE_EDGES_FIELD_FROM_HOST = 0,   // grantee（被授予者）的 host
  MYSQL_ROLE_EDGES_FIELD_FROM_USER,       // grantee 的 user
  MYSQL_ROLE_EDGES_FIELD_TO_HOST,         // 被授予的 role 的 host
  MYSQL_ROLE_EDGES_FIELD_TO_USER,         // 被授予的 role 的 user
  MYSQL_ROLE_EDGES_FIELD_WITH_ADMIN_OPTION,
  MYSQL_ROLE_EDGES_FIELD_COUNT
};

enum mysql_default_roles_table_field {
  MYSQL_DEFAULT_ROLES_FIELD_HOST = 0,
  MYSQL_DEFAULT_ROLES_FIELD_USER,
  MYSQL_DEFAULT_ROLES_FIELD_DEFAULT_ROLE_HOST,
  MYSQL_DEFAULT_ROLES_FIELD_DEFAULT_ROLE_USER,
  MYSQL_DEFAULT_ROLES_FIELD_COUNT
};
```

| 语句 | 写哪张表 | 备注 |
|---|---|---|
| `GRANT role TO u` | `mysql.role_edges` | 写入前先做环检测（见上节「角色循环继承如何被处理」的两层防线） |
| `REVOKE role FROM u` | `mysql.role_edges` | `mandatory_roles` 里的角色会被 `ER_MANDATORY_ROLE` 拦下 |
| `SET DEFAULT ROLE ... TO u` | `mysql.default_roles` | 只记"默认激活哪些"，不复制权限 |
| `GRANT` / `SET DEFAULT ROLE` 的实际写盘 | `modify_role_edges_in_table()`（`sql/auth/role_tables.cc`） | |

★ **两张表都只存"关系"，不存权限**。角色的权限仍在 `mysql.user` / 各 grant 表的**角色那一行**上（`is_role` 的用户行），展开时靠图把它们累加到 grantee。这一点决定了：给角色 `GRANT` 权限后，**所有被授予者的已激活会话在下一次权限检查时才看到**（`checkout_access_maps()` 重算，没有即时推送）。

**④ 观测手段**：`SELECT roles_graphml()` 可把整张角色图导出成 GraphML（源码注释原文：*"To show the role graph use `SELECT roles_graphml()`"*），另有 I_S 视图 `APPLICABLE_ROLES` / `ENABLED_ROLES` / `ADMINISTRABLE_ROLE_AUTHORIZATIONS` / `ROLE_*_GRANTS`（详见后文「I_S 视图与真视图」一节）。

---

### 四、动态权限与部分撤销

#### 动态权限：一个 `unordered_set<string>` 就是全部注册表

```cpp
// sql/auth/dynamic_privilege_table.h
typedef std::unordered_set<std::string> Dynamic_privilege_register;
```

★ **它就是一个全局 `unordered_set<string>`，不记录注册者（组件名）**。这是后面若干"坑"的根源。

三个容易混淆的 map（都定义在 `sql/auth/auth_internal.h`）：

```cpp
typedef std::map<std::string, bool> Dynamic_privileges;      // 展开后：权限名 → 是否有 GRANT OPTION
typedef std::pair<std::string, bool> Grant_privilege;
typedef std::unordered_multimap<Role_id, Grant_privilege, role_id_hash>
    User_to_dynamic_privileges_map;                          // 持久缓存：authid → (权限名, grant_opt)
```

- `g_dynamic_privilege_register`：已注册的权限名（**全局**）；
- `g_dynamic_privileges_map`：谁有什么动态权限（**持久缓存**，从 `mysql.global_grants` 装载）；
- `Acl_map::m_dynamic_privileges`：`Dynamic_privileges`，**展开 role 后**当前会话可见的集合。

**注册接口是 component service**（`sql/server_component/server_component.cc`）：

```cpp
BEGIN_SERVICE_IMPLEMENTATION(mysql_server, dynamic_privilege_register)
dynamic_privilege_services_impl::register_privilege,
    dynamic_privilege_services_impl::unregister_privilege
    END_SERVICE_IMPLEMENTATION();

BEGIN_SERVICE_IMPLEMENTATION(mysql_server, global_grants_check)
dynamic_privilege_services_impl::has_global_grant END_SERVICE_IMPLEMENTATION();
```

实现（`sql/auth/dynamic_privileges_impl.cc`）——**强制 toupper**，且**重复注册视为成功**：

```cpp
DEFINE_BOOL_METHOD(dynamic_privilege_services_impl::register_privilege,
                   (const char *privilege_str, size_t privilege_str_len)) {
  std::string priv;
  const char *c = &privilege_str[0];
  for (size_t i = 0; i < privilege_str_len; ++i, ++c)
    priv.append(1, static_cast<char>(toupper(*c)));

  Thd_creator get_thd(current_thd);
  Acl_cache_lock_guard acl_cache_lock(get_thd(), Acl_cache_lock_mode::WRITE_MODE);
  acl_cache_lock.lock();
  Dynamic_privilege_register *reg = get_dynamic_privilege_register();
  if (reg->find(priv) != reg->end()) {
    /* If the privilege ID already is registered; report success */
    return false;
  }
  return !get_dynamic_privilege_register()->insert(priv).second;
}
```

**服务器内置的 33 个动态权限**在 `dynamic_privilege_init()` 里注册（注意它是通过 plugin registry 拿 service 的，因为 bootstrap 时组件可能还没起来）：

```
ROLE_ADMIN, SYSTEM_VARIABLES_ADMIN, BINLOG_ADMIN, REPLICATION_SLAVE_ADMIN,
GROUP_REPLICATION_ADMIN, ENCRYPTION_KEY_ADMIN, CONNECTION_ADMIN, SET_USER_ID,
XA_RECOVER_ADMIN, PERSIST_RO_VARIABLES_ADMIN, BACKUP_ADMIN, CLONE_ADMIN,
RESOURCE_GROUP_ADMIN, RESOURCE_GROUP_USER, SESSION_VARIABLES_ADMIN,
BINLOG_ENCRYPTION_ADMIN, SERVICE_CONNECTION_ADMIN, APPLICATION_PASSWORD_ADMIN,
SYSTEM_USER, TABLE_ENCRYPTION_ADMIN, AUDIT_ADMIN, TELEMETRY_LOG_ADMIN,
REPLICATION_APPLIER, SHOW_ROUTINE, INNODB_REDO_LOG_ENABLE, FLUSH_OPTIMIZER_COSTS,
FLUSH_STATUS, FLUSH_USER_RESOURCES, FLUSH_TABLES, GROUP_REPLICATION_STREAM,
AUTHENTICATION_POLICY_ADMIN, PASSWORDLESS_USER_ADMIN, SENSITIVE_VARIABLES_OBSERVER
```

插件/组件注册的有 4 个：`NDB_STORED_USER`（NDB）、`INNODB_REDO_LOG_ARCHIVE`（`log0meb.cc`）、`VERSION_TOKEN_ADMIN`（version_token 插件）、`SKIP_QUERY_REWRITE`（Rewriter 插件）。

**★ 还有一条"隐含注册"路径**：`populate_dynamic_privilege_caches()` 加载 `mysql.global_grants` 时，遇到未注册的名字会**就地注册**：

```cpp
if (grant_dynamic_privilege(str_priv, str_user, str_host,
                            (*with_grant_option == 'Y' ? true : false), no_update)) {
  /*
    This privilege ID hasn't been registered yet. It can happen when a
    previously grant has been given but the plugin or component which owns
    the privilege ID isn't loaded yet.
    The policy is that any privilege ID that exist in mysql.global_grants
    is a valid privilege ID.
  */
  if (service->register_privilege(str_priv.str, str_priv.length) || ...
```

官方手册确认了同样的行为：服务器启动时会自动注册 `global_grants` 中出现的权限名，`FLUSH PRIVILEGES` 也会。**这构成一个安全边界：能写 `mysql.global_grants` 就等于能"发明"动态权限名。**

**判定**：

```cpp
std::pair<bool, bool> Security_context::has_global_grant(const char *priv,
                                                         size_t priv_len) {
  /* server started with --skip-grant-tables */
  if (!initialized || m_is_skip_grants_user) return std::make_pair(true, true);

  std::string privilege(priv, priv_len);

  if (m_acl_map == nullptr) {
    /* ---- 无 active role：直接查持久缓存，不展开 role ---- */
    THD *thd = m_thd ? m_thd : current_thd;
    Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::READ_MODE);
    if (!acl_cache_lock.lock(false)) return std::make_pair(false, false);
    Role_id key(&m_priv_user[0], m_priv_user_length, &m_priv_host[0], m_priv_host_length);
    User_to_dynamic_privileges_map::iterator it, it_end;
    std::tie(it, it_end) = get_dynamic_privileges_map()->equal_range(key);
    it = std::find(it, it_end, privilege);
    if (it != it_end) return std::make_pair(true, it->second.second);
    return std::make_pair(false, false);
  }
  /* ---- 有 active role：查展开后的 Acl_map ---- */
  Dynamic_privileges::iterator it = m_acl_map->dynamic_privileges()->find(privilege);
  if (it != m_acl_map->dynamic_privileges()->end())
    return std::make_pair(true, it->second);
  return std::make_pair(false, false);
}
```

返回 `pair<是否有权限, 是否带 GRANT OPTION>`。典型调用模式（贯穿 `sql_parse.cc`）：

```cpp
if (!sctx->check_access(SUPER_ACL) &&
    !sctx->has_global_grant(STRING_WITH_LEN("BINLOG_ADMIN")).first) { ...拒绝... }
```

`--skip-grant-tables` 下直接返回 `{true, true}`。

**语法层如何区分静态/动态权限？语法上不区分**。`sql_yacc.yy` 里 `GRANT <ident>` 统一走 `PT_role_or_dynamic_privilege`，靠 **`opt_column_list` 是否存在**分叉到 `PT_dynamic_privilege`，靠 `@` 分叉到 role（`PT_role_at_host`），只有关键字（`SELECT_SYM` 等）才一定是 `PT_static_privilege`。真正的"这个名字注册了吗"检查在语义层：`is_dynamic_privilege_registered()`。

#### partial_revokes：在全局权限上"挖洞"

```cpp
using db_revocations = std::unordered_map<std::string, Access_bitmask>;
using Db_access_map  = std::map<std::string, Access_bitmask>;

class DB_restrictions final : public Abstract_restrictions {
 public:
  void add(const std::string &db_name, const Access_bitmask revoke_privs);
  bool find(const std::string &db_name, Access_bitmask &access) const;
  bool has_more_restrictions(const DB_restrictions &, Access_bitmask) const;
  void get_as_json(Json_array &restrictions_array) const;
 private:
  db_revocations *m_restrictions = nullptr;   // ★ 懒分配：绝大多数账号没有 restriction
};

class Restrictions {
 public:
  const DB_restrictions &db() const;
  void set_db(const DB_restrictions &db_restrictions);
  bool is_empty() const;
 private:
  DB_restrictions m_db_restrictions;
};
```

**持久化在 `mysql.user.User_attributes` 的 JSON 里**：

```json
[{"Database": "mysql", "Privileges": ["INSERT","DELETE"]}, ...]
```

序列化/反序列化在 `sql/auth/acl_table_user.cc`（`read_user_attributes()` / 写盘分支），`DB_restrictions::get_as_json()` 生成。全局缓存是 `Acl_restrictions`（`malloc_unordered_map<authid, Restrictions>`，key 是 `'user'@'host'` 字符串）。

**系统变量**（`sql/sys_vars.cc`）：

```cpp
static Sys_var_bool Sys_partial_revokes(
    "partial_revokes",
    "Access of database objects can be restricted, "
    "even if user has global privileges granted.",
    GLOBAL_VAR(opt_partial_revokes), CMD_LINE(OPT_ARG),
    DEFAULT(DEFAULT_PARTIAL_REVOKES), &Plock_partial_revokes, IN_BINLOG,
    ON_CHECK(check_partial_revokes), ON_UPDATE(partial_revokes_update), nullptr,
    sys_var::PARSE_EARLY);
```

★ `DEFAULT_PARTIAL_REVOKES` 来自 **CMake**（`cmake/install_layout.cmake`），社区默认 **0（OFF）**，只有 `-DPARTIAL_REVOKES_DEFAULT=1` 时才默认 ON（Oracle 企业版构建的行为）。

★ **真实状态是原子的**，`sql/mysqld.h` 里有明确警告：

```cpp
/*
  @warning : The real value is in @ref partial_revokes. The @ref
  opt_partial_revokes is just a tool to trick the Sys_var class into
  operating on an atomic variable.
  Thus : do not use or access @ref opt_partial_revokes in your code.
         If you need the value of the flag please use the @ref partial_revokes global.
*/
extern bool opt_partial_revokes;

bool mysqld_partial_revokes() { return partial_revokes.load(std::memory_order_relaxed); }
```

这是"Sys_var 体系不支持 `std::atomic` 时的适配技巧"，值得记一笔（见 [工程实现技法](#一高级语法技巧--c-特性)）。

**判定**：见 [判定链](#security_contextcheck_access全局短路谓词--partial_revokes-的渗透点) 的 `filter_access()`。要点是 `(access & restrictions_mask) ^ access` = `access & ~restrictions_mask`，且**只有 `want_access & DB_ACLS` 的部分才受限制**（`check_access` 里的守卫）——动态权限、`SUPER` 这类全局权限**不受 partial revoke 限制**。

**限制如何传播**：合并逻辑没有 `Restrictions::merge`，而是在 `DB_restrictions_aggregator::aggregate_restrictions()`（`sql/auth/partial_revokes.cc`）。核心规则（对应 WL#12098 的 FR2-07/08/09）：

- grantor 有 restriction、grantee 无 restriction 且 grantee 全局权限已覆盖请求 ⇒ **不传播**（grantee 本来就比 grantor 权限多）；
- 双方都有 ⇒ `rm1 | rm2`（两个来源的 OR）；
- 只有 grantor 有 ⇒ 过滤掉"grantee 在 db 级或全局级已经拥有的"部分再传播；
- 只有 grantee 有 ⇒ 把"本次请求授予的权限"对应的限制**解除**。

调用点在 role 图遍历里（`Get_access_maps::discover_vertex()`）：**每个 role 顶点都重算一次并 `m_restrictions->set_db(...)` 覆盖**，所以最终值依赖 BFS 遍历顺序（见 [坑](#坑与已知缺陷)）。

**`partial_revokes=ON` 会改变全实例的库名匹配语义**：`acl_get()` / `Security_context::db_acl()` / `acl_getroot()` 三处都从 `wild_compare()` 切成 `strcmp()` / `my_strcasecmp()`。官方手册的对应说法是：启用后 schema 名中未转义的 `_` 和 `%` 会被当作字面字符。

**相关错误/警告**：

| 符号 | 说明 |
|---|---|
| `ER_PARTIAL_REVOKES_EXIST` | 还有 restriction 时试图关掉变量 → 拒绝 |
| `ER_TURNING_ON_PARTIAL_REVOKES` | 启动/FLUSH 时发现表里有 restriction 而变量 OFF → 自动打开（写 error log） |
| `ER_PARTIAL_REVOKE_AND_DB_GRANT_BOTH_EXISTS` | 同一库上"库级授权"与"partial revoke"共存 → 说明 `mysql` 库可能已损坏 |
| `ER_WARN_INVALID_DB_RESTRICTIONS` | 限制的库上没有对应全局权限 → 忽略该限制并告警 |
| `ER_AUTHCACHE_DB_ENTRY_LOWERCASED_REVOKE_WILL_FAIL` | 库名被 `lower_case_table_names` 折叠 → revoke 会失败 |

**与复制的关系**：`partial_revokes=ON` 时，binlog 里记录的 `GRANT` 是**扩展语法，包含发出语句的 current user 及其激活的角色**（对应 `AS` 子句与 WL#12820）。若副本上不存在该 user/role，applier 线程会在该 `GRANT` 处**报错停止**。源码里对应：

```cpp
      /*
        2. If all of the following conditions are met:
           - --partial_revokes is ON
           - Statement is a GRANT at global level (*.*)
           This is required because, current user may have restriction
           list and grant may propagate the same to grantee. In order
           to repliably replay this on other nodes in an HA setup, details
           of current user has to be captured.
      */
      } else if (partial_revokes && !revoke_grant && !is_proxy && !db) {
```

---

### 五、`SHOW GRANTS`：从位图反向拼装语句

#### 入口与权限要求

看**别人**的 `SHOW GRANTS` 需要 `mysql.user` 上的 **SELECT**（不是 `CREATE_USER_ACL`）：

```cpp
bool Sql_cmd_show_grants::execute_inner(THD *thd) {
  bool show_mandatory_roles = (for_user == nullptr);
  bool have_using_clause = (using_users != nullptr && using_users->elements > 0);
  ...
  } else if (strcmp(thd->security_context()->priv_user().str,
                    for_user->user.str) != 0) {
    Table_ref table("mysql", "user", nullptr, TL_READ);
    if (!is_granted_table_access(thd, SELECT_ACL, &table)) {
      my_error(ER_TABLEACCESS_DENIED_ERROR, MYF(0), "SELECT", ..., "user");
      return false;
    }
  }
  ...
}
```

#### 主函数

```cpp
bool mysql_show_grants(THD *thd, LEX_USER *lex_user,
                       const List_of_auth_id_refs &using_roles,
                       bool show_mandatory_roles, bool have_using_clause) {
  int error = 0;
  ACL_USER *acl_user = nullptr;
  char buff[1024];

  assert(initialized);
  Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::READ_MODE);
  if (!acl_cache_lock.lock()) return true;

  acl_user = find_acl_user(lex_user->host.str, lex_user->user.str, true);
  if (!acl_user) {
    my_error(ER_NONEXISTING_GRANT, MYF(0), lex_user->user.str, lex_user->host.str);
    return true;
  }

  /*
    For a SHOW GRANTS USING one needs to check if the session has access to
    the roles specified in the USING clause.
    But if there's no USING clause the list of active session roles is used
    instead. ... Thus a check if these are still granted might fail.
    So we skip the check if there's no explicit USING ...
  */
  if (have_using_clause) {
    /* 校验 using_roles 里每个角色确实被授予（或在 mandatory_roles 中），否则 ER_ROLE_NOT_GRANTED */
  }

  Item_string *field = new Item_string("", 0, &my_charset_latin1);
  field->max_length = 1024;
  strxmov(buff, "Grants for ", lex_user->user.str, "@", lex_user->host.str, NullS);
  field->item_name.set(buff);
  ...

  // aggregate over the active role and user privileges
  Db_access_map db_map, db_wild_map;
  Table_access_map table_map;
  SP_access_map sp_map, func_map;
  Grant_acl_set with_admin_acl;
  Dynamic_privileges dynamic_acl;
  List_of_granted_roles granted_roles;
  Restrictions restrictions;
  Access_bitmask access;
  table_map.set_thd(thd);
  get_privilege_access_maps(acl_user, &using_roles, &access, &db_map,
                            &db_wild_map, &table_map, &sp_map, &func_map,
                            &granted_roles, &with_admin_acl, &dynamic_acl,
                            restrictions);
  String output;
  make_global_privilege_statement(thd, access, acl_user, &output);
  Protocol *protocol = thd->get_protocol();
  protocol->start_row();
  protocol->store_string(output.ptr(), output.length(), output.charset());
  protocol->end_row();

  make_dynamic_privilege_statement(thd, acl_user, protocol, dynamic_acl);
  make_database_privilege_statement(thd, acl_user, protocol, db_map,
                                    db_wild_map, restrictions.db());
  make_table_privilege_statement(thd, acl_user, protocol, table_map);
  make_sp_privilege_statement(thd, acl_user, protocol, sp_map, 0);
  make_sp_privilege_statement(thd, acl_user, protocol, func_map, 1);
  make_proxy_privilege_statement(thd, acl_user, protocol);
  make_roles_privilege_statement(thd, acl_user, protocol, granted_roles,
                                 show_mandatory_roles);
  make_with_admin_privilege_statement(thd, acl_user, protocol, with_admin_acl,
                                      granted_roles);
  my_eof(thd);
  return error;
}
```

**★ 关键在 `get_privilege_access_maps()` 这一步**：`SHOW GRANTS` **不是**"把 GRANT 语句记下来再回放"，而是**先展开成 8 个容器，再从容器反向拼装**。这就是它天然能显示"role 展开后的有效权限"的原因，也是 `SHOW GRANTS FOR u USING r1,r2` 能工作的原因（把 `using_roles` 直接喂给展开函数）。

**输出顺序（源码顺序）**：

```
1. GRANT <global privs> ON *.* TO u@h [WITH GRANT OPTION]         make_global_privilege_statement
2. GRANT <dyn privs> ON *.* TO u@h                                make_dynamic_privilege_statement（第 1 轮：无 GRANT OPTION）
   GRANT <dyn privs> ON *.* TO u@h WITH GRANT OPTION                                              （第 2 轮）
3. GRANT ... ON `db`.* TO u@h                                     make_database_privilege_statement（精确库名）
   GRANT ... ON `db_%`.* TO u@h                                                                    （通配库名）
   REVOKE <privs> ON `db`.* FROM u@h                                                               （partial revokes，按库名排序）
4. GRANT <privs> [(c1,c2)] ON `db`.`tbl` TO u@h [WITH GRANT OPTION]  make_table_privilege_statement（表级 + 列级合并）
5. GRANT ... ON PROCEDURE `db`.`p` TO u@h                         make_sp_privilege_statement(type=0)
6. GRANT ... ON FUNCTION `db`.`f` TO u@h                          make_sp_privilege_statement(type=1)
7. GRANT PROXY ON `proxied`@`h` TO u@h [WITH GRANT OPTION]        make_proxy_privilege_statement
8. GRANT r1@%, r2@% TO u@h                                        make_roles_privilege_statement
9. GRANT r3@% TO u@h WITH ADMIN OPTION                            make_with_admin_privilege_statement
```

注意：**没有独立的 column 段**（并入表级）；**动态权限永远拼成 `ON *.*`**；**`WITH ADMIN OPTION` 单独成行且在最后**。

#### 三个值得单独拎出来的细节

**(a) 位图 → 权限名**靠 `global_acls_vector[counter]`：

```cpp
const Access_bitmask test_access = want_access & ~GRANT_ACL;
int counter = 0;
Access_bitmask j = SELECT_ACL;
for (; j <= GLOBAL_ACLS; counter++, j <<= 1) {
  if (test_access & j) {
    if (found) global->append(STRING_WITH_LEN(", "));
    found = true;
    global->append(global_acls_vector[counter].c_str(), ...);
  }
}
```

`want_access == GRANT_ACL`（只有 grant option）时输出 `USAGE`；位图全满时输出 `ALL PRIVILEGES`。

**(b) partial revoke 行的输出与排序**：只有 `mysqld_partial_revokes() && !restrictions.is_empty()` 时才输出，且因为底层是 `unordered_map`，源码**先拷到数组再 `std::sort`**：

```cpp
Mem_root_array<std::pair<std::string, Access_bitmask>> restrictions_array(thd->mem_root);
for (const auto &rl_itr : restrictions.get())
  restrictions_array.push_back({rl_itr.first, rl_itr.second});
std::sort(restrictions_array.begin(), restrictions_array.end(),
          [](const auto &p1, const auto &p2) -> bool {
            return (p1.first.compare(p2.first) <= 0);
          });
```

目的很实际：让输出**稳定可重放**（mysqldump 依赖它）。

**(c) 列级权限会在同一条语句里把权限名写两次**：

```cpp
if (agg.cols) {
  for (col_it ...) {
    if (col_it->second & j) {
      if (!found_col) {
        found_col = 1;
        /*
          If we have a duplicated table level privilege, we
          must write the access privilege name again.
        */
        if (agg.table_access & j) {
          global.append(STRING_WITH_LEN(", "));
          global.append(global_acls_vector[counter].c_str(), ...);
        }
        global->append(STRING_WITH_LEN(" ("));
      } else global->append(STRING_WITH_LEN(", "));
      append_identifier(thd, &global, col_it->first.c_str(), col_it->first.length());
    }
  }
  if (found_col) global->append(')');
}
```

所以会看到 `GRANT SELECT (c1, c2), SELECT ON db.tbl TO u` 这种形态——为了让输出被原样重放时语义等价。

**(d) mandatory roles 只在无 `FOR` 子句时列出**：

```cpp
/*
  Because the output of SHOW GRANTS is used by tools like mysqldump we
  cannot include mandatory roles if the FOR clause is used.
*/
if (show_mandatory_roles) {   // == (for_user == nullptr)
  get_mandatory_roles(&mandatory_roles);
}
```

#### `SHOW CREATE USER`：反向填 LEX 再复用 CREATE USER 的 rewriter

`mysql_show_create_user()` 不自己拼字符串，而是**把 `ACL_USER` 反向填回 `LEX`**，然后交给 `Rewriter_show_create_user::rewrite()`（继承 `Rewriter_user`，只 override 三件事：前缀 `"CREATE USER "`、从 `Show_user_params::metadata_str` 读出的 `ATTRIBUTE` 子句、以及 `print_identified_with_as_hex` 打开时把 auth string 转 hex）。

它必须先 `open_and_lock_tables()` 读 `mysql.user.user_attributes` 的 JSON（因为 ATTRIBUTE/COMMENT 只在盘上），源码注释专门交代了锁序：

```cpp
/*
   Open user table so we later can read the JSON data in the user_attribute
   field. All tables must be opened before the acl_cache_lock
*/
```

密码哈希是否隐藏取决于 `are_both_users_same` 与当前用户对 `mysql.user` 是否有 SELECT。

#### I_S 权限视图：填充表 vs 真视图（两种完全不同的数据源）

| 视图 | 形态 | 数据源 |
|---|---|---|
| `USER_PRIVILEGES` / `SCHEMA_PRIVILEGES` / `TABLE_PRIVILEGES` / `COLUMN_PRIVILEGES` | **填充表**（`sql/sql_show.cc` 的 `schema_tables[]` 注册 `fill_schema_*`，实现在 `sql/auth/sql_authorization.cc`） | **内存 ACL 缓存**（带 ACL 读锁）。`USER_PRIVILEGES` 还会遍历 `g_dynamic_privileges_map` 把动态权限也列进去 |
| `APPLICABLE_ROLES` / `ENABLED_ROLES` / `ADMINISTRABLE_ROLE_AUTHORIZATIONS` / `ROLE_*_GRANTS` | **真 I_S 视图**（`sql/dd/impl/system_views/`） | **直接读 `mysql.*` 基表**；`APPLICABLE_ROLES` 是**递归 CTE**，用 `SHA2(CONCAT(QUOTE(user),'@',QUOTE(host)))` 做路径环检测 |
| `DEFAULT_ROLES` | **不存在**（只有基表 `mysql.default_roles`） | — |

★ 这个差异有实际后果：`USER_PRIVILEGES` 反映的是"已加载进内存的权限"（受 FLUSH 影响），`APPLICABLE_ROLES` 反映的是"盘上的 `role_edges`"（不受 FLUSH 影响，但也不会多算 role 继承）。

---

### 六、缓存刷新与生效时机

#### 各语句触发哪些缓存更新（逐个核实）

| 语句 | 写盘 | 同步改内存 | 是否 bump Acl_cache 版本 |
|---|---|---|---|
| `CREATE USER` | `replace_user_table()` | `acl_insert_user()` + `rebuild_cached_acl_users_for_name()` + `rebuild_check_host()` + `clear_and_init_db_cache()` | **否**（新用户没有已存在的会话缓存） |
| `DROP USER` | 同上 | `acl_users->erase()` + 上述两个 rebuild；末尾 `populate_roles_caches()` | **是**（经 `populate_roles_caches`） |
| `RENAME USER` | 同上 | 改 `ACL_USER` 的 user/host + 上述两个 rebuild + `roles_rename_authid()`；末尾 `populate_roles_caches()` | **是** |
| `ALTER USER`（非 DEFAULT ROLE） | 同上 | `acl_update_user()` + `clear_and_init_db_cache()` | **否** |
| `SET PASSWORD` | 同上 | 同上 + `update_sctx_cache()` | **否** |
| `GRANT/REVOKE ON *.*` | `replace_user_table()` | `acl_update_user()` / `acl_insert_user()` + `clear_and_init_db_cache()` | **是**（`mysql_grant()` 末尾） |
| `GRANT/REVOKE ON db.*` | `replace_db_table()` | `acl_update_db()` / `acl_insert_db()` + `clear_and_init_db_cache()` | **是** |
| `GRANT/REVOKE ON db.t` | `replace_table_table()` | 直接改 `column_priv_hash` 里的 `GRANT_TABLE` | **是**（`mysql_table_grant()` 末尾） |
| `GRANT/REVOKE (col)` | `replace_column_table()` | 直接改 `GRANT_TABLE::hash_columns` | **是** |
| `GRANT/REVOKE ON PROCEDURE` | `replace_routine_table()` | 改 `proc_priv_hash` / `func_priv_hash` | **是**（`mysql_routine_grant()` 末尾） |
| `GRANT/REVOKE PROXY` | `replace_proxies_priv_table()` | `acl_insert_proxy_user()` / `acl_update_proxy_user()` | **是** |
| `GRANT/REVOKE role` | `modify_role_edges_in_table()` | `grant_role()` / `revoke_role()`（改图） | **是** |
| `GRANT/REVOKE 动态权限` | `modify_dynamic_privileges_in_table()` | `grant_dynamic_privilege()` | **是** |
| `SET DEFAULT ROLE` | `modify_default_roles_in_table()` | 改 `g_default_roles` | **是**（`mysql_alter_or_clear_default_roles()`） |
| `FLUSH PRIVILEGES` | — | `acl_reload()` + `grant_reload()`（全量重建） | **是** |

★ **一个反直觉的核实结果**：`sql/auth/sql_user.cc` 里**一处** `increase_version()` 都没有（全仓库 13 处 `increase_version` 调用，`sql_user.cc` 不在其中）。所以 `CREATE USER` / `ALTER USER` / `SET PASSWORD` **不会**让已连接会话重算 `Acl_map`。

#### 为什么 `GRANT` 不需要 `FLUSH PRIVILEGES`

因为 `GRANT` 走的是**先写盘、再同步改内存**的 `replace_*_table()` 族。以 `replace_user_table()` 为例：

```cpp
    clear_and_init_db_cache(); /* Clear privilege cache */
    if (old_row_exists) {
      acl_update_user(combo->user.str, combo->host.str, lex->ssl_type, ...);
    } else
      acl_insert_user(thd, combo->user.str, combo->host.str, lex->ssl_type, ...);
```

**一条反证**（很有说服力）：ACL DDL 提交失败/回滚时，`acl_end_trans_and_close_tables()` 会**反过来**用磁盘去修内存：

```cpp
  if (result || rollback_transaction) {
    /*
      Try to bring in-memory structures back in sync with on-disk data if we
      have failed to commit our changes.
    */
    reload_acl_caches(thd, true);
    close_thread_tables(thd);
  }
```

说明"磁盘 → 内存"的同步**只有** `acl_reload()/grant_reload()` 这一条路；而"内存 → 磁盘 + 内存"的同步是 ACL DDL 自己负责的。

#### `FLUSH PRIVILEGES` 的完整路径

语法层：`FLUSH PRIVILEGES` → `SQLCOM_FLUSH`，`options |= REFRESH_GRANT`。

★ **没有 `REFRESH_PRIVILEGES` 这个宏**（`include/mysql_com.h`：`#define REFRESH_GRANT 1 /**< Refresh grant tables, FLUSH PRIVILEGES */`）。也不存在 `REFRESH_ROLES` / `flush_roles()`。

```cpp
// sql/sql_reload.cc
bool handle_reload_request(THD *thd, unsigned long options, Table_ref *tables,
                           int *write_to_binlog) {
  ...
  if (options & REFRESH_GRANT) {
    THD *tmp_thd = nullptr;
    /*
      If handle_reload_request() is called from SIGHUP handler we have to
      allocate temporary THD for execution of acl_reload()/grant_reload().
    */
    if (!thd && (thd = (tmp_thd = new THD))) { ... }

    if (thd) {
      bool reload_acl_failed = reload_acl_caches(thd, false);
      bool reload_servers_failed = servers_reload(thd);
      notify_flush_event(thd);
      ...
    }
    reset_mqh(thd, (LEX_USER *)nullptr, true);
    ...
  }
```

它**顺带**做了两件与权限无关的事：`servers_reload()`（`mysql.servers`）与 `reset_mqh(nullptr, true)`（清空所有用户的资源计数）。

权限要求：`REFRESH_GRANT` 不在"可用动态权限替代"的白名单里（白名单是 `FLUSH_OPTIMIZER_COSTS` / `FLUSH_STATUS` / `FLUSH_USER_RESOURCES` / `FLUSH_TABLES`），所以它**只接受 `RELOAD_ACL`**（`is_reload_request_denied()` → `check_global_access(thd, RELOAD_ACL)`）。

`reload_acl_caches()` 的锁序很讲究：**先拿 ACL 表的 MDL（`MDL_SHARED_READ_ONLY`），再拿 ACL cache 锁**——源码注释明写 "To avoid deadlocks we should obtain table locks before obtaining acl_cache->lock mutex"。

#### `FLUSH PRIVILEGES` 的代价：双缓冲 + 全量重建

```cpp
bool acl_reload(THD *thd, bool mdl_locked) {
  ...
  Acl_cache_lock_guard acl_cache_lock(thd, Acl_cache_lock_mode::WRITE_MODE);
  ...
  auto swap_role_cache = [&]() {
    std::swap(old_granted_roles, g_granted_roles);
    std::swap(old_default_roles, g_default_roles);
    std::swap(old_authid_to_vertex, g_authid_to_vertex);
  };

  /* To avoid deadlocks we should obtain table locks before obtaining
     acl_cache->lock mutex. */
  Table_ref tables[6] = { user, db, proxies_priv, global_grants, role_edges, default_roles };
  ...
  if (open_and_lock_tables(thd, tables, flags)) { ... goto end; }
  if (!acl_cache_lock.lock()) goto end;

  old_acl_users = acl_users;  old_acl_dbs = acl_dbs;  old_acl_proxy_users = acl_proxy_users;
  old_acl_restrictions = std::move(acl_restrictions);
  swap_role_cache();
  roles_init();

  acl_users       = new Prealloced_array<ACL_USER, ACL_PREALLOC_SIZE>(key_memory_acl_mem);
  acl_dbs         = new Prealloced_array<ACL_DB, ACL_PREALLOC_SIZE>(key_memory_acl_mem);
  acl_proxy_users = new Prealloced_array<ACL_PROXY_USER, ACL_PREALLOC_SIZE>(key_memory_acl_mem);
  acl_restrictions = make_unique<Acl_restrictions>();

  old_mem = std::move(global_acl_memory);
  delete acl_wild_hosts;  acl_wild_hosts = nullptr;
  delete acl_check_hosts; acl_check_hosts = nullptr;
  old_dyn_priv_map = swap_dynamic_privileges_map(new User_to_dynamic_privileges_map());

  /*
    Revert to the old acl caches, if either loading of acl cache or role
    cache failed. We do this because roles caches maintain the shallow
    copies of the ACL_USER(s).
  */
  if ((return_val = acl_load(thd, tables)) ||
      (return_val = reload_roles_cache(thd, (tables + 4)))) {   // Error. Revert
    acl_free();
    acl_users = old_acl_users;  acl_dbs = old_acl_dbs;
    acl_proxy_users = old_acl_proxy_users;
    global_acl_memory = std::move(old_mem);
    acl_restrictions = std::move(old_acl_restrictions);
    swap_role_cache();
    delete_old_role_cache();
    init_check_host();
    delete swap_dynamic_privileges_map(old_dyn_priv_map);
    if (!old_dyn_priv_map) dynamic_privileges_init();
    if (acl_users) rebuild_cached_acl_users_for_name();
  } else {
    delete old_acl_users; delete old_acl_dbs; delete old_acl_proxy_users;
    delete old_dyn_priv_map;
    delete_old_role_cache();
    old_mem.Clear();
  }

end:
  ...
  get_global_acl_cache()->increase_version();
  return return_val;
}
```

**双缓冲 + 原子替换的证据**：全局指针直接换成 new 出来的新数组，旧的暂存到 `old_*`；成功才 `delete` 旧的，失败整体回滚。整段在 `WRITE_MODE` 的 ACL cache 锁下，读侧要么看到全旧要么看到全新。

**代价构成**：
1. 打开并锁 6 张 ACL 表（`MDL_SHARED_READ_ONLY`）；
2. 全表扫描 + 重建 `acl_users` / `acl_dbs` / `acl_proxy_users` / `acl_restrictions` / 角色图 / 动态权限 map / `acl_check_hosts` / `acl_wild_hosts` / `name_to_userlist` / `db_cache`；
3. `grant_reload()` 再扫 `tables_priv` / `columns_priv` / `procs_priv` 重建三个 hash（也是双缓冲，`old_column_priv_hash` 回滚）；
4. `increase_version()` → `flush_cache()` ⇒ **所有带 active role 的会话下一条语句都要重跑一次 BFS 展开**。

#### `Acl_cache`：版本编进 key 的 LF_HASH

```cpp
Acl_map *Acl_cache::checkout_acl_map(Security_context *sctx, Auth_id_ref &uid,
                                     List_of_auth_id_refs &active_roles) {
  uchar *key;
  unsigned key_len;
  uint64 version = m_role_graph_version.load();
  if (!create_acl_cache_hash_key(&key, &key_len, version, uid, active_roles)) {
    active_roles.clear();
    return nullptr;
  }
  LF_PINS *pins = lf_hash_get_pins(&m_cache);
  Acl_hash_entry *entry = (Acl_hash_entry *)lf_hash_search(&m_cache, pins, key, key_len);
  if (entry == nullptr || entry == MY_LF_ERRPTR) {
    lf_hash_search_unpin(pins);
    Acl_map *map = create_acl_map(version, sctx);  // deleted in cache_flusher
    Acl_hash_entry new_entry{version, key, key_len, map};
    int rc = lf_hash_insert(&m_cache, pins, &new_entry);  // shallow copy of entry
    if (rc != 0) {
      /* There was a duplicate; throw away the allocated memory */
      lf_hash_put_pins(pins); my_free(key); delete map;
      return checkout_acl_map(sctx, uid, active_roles);   // 递归重试
    }
    map->increase_reference_count();
    lf_hash_put_pins(pins);
    return map;
  }
  Acl_map *map = entry->map;
  map->increase_reference_count();
  lf_hash_search_unpin(pins);
  lf_hash_put_pins(pins);
  my_free(key);
  return map;
}

void Acl_cache::return_acl_map(Acl_map *map) { map->decrease_reference_count(); }
```

**key 的构成**（`create_acl_cache_hash_key()`）：

```
user  '@'  host  '`'  version(uint64)  [ role1user '@' role1host '`' ]  [ role2user '@' role2host '`' ] ...
```

★ **版本号被编进 key** —— 这是本实现最漂亮的一处：bump 版本**不需要**修改或删除任何旧条目，旧 key 自然不再被命中，新 checkout 用新 key 重新计算。**把"缓存失效"变成"缓存 key 变化"，从而完全避免了失效时的写操作**（LF_HASH 的读路径因此保持无锁）。

**延迟删除**（保证正在用旧 map 的线程不会 use-after-free）：

```cpp
static int cache_flusher(const uchar *ptr, void *) {
  const Acl_hash_entry *entry = reinterpret_cast<const Acl_hash_entry *>(ptr);
  if (entry != nullptr) {
    if (entry->map->reference_count() == 0 &&
        entry->map->version() < l_cache_flusher_global_version)
      return 1;
  }
  return 0;
}

void Acl_cache::flush_cache() {
  LF_PINS *pins = lf_hash_get_pins(&m_cache);
  mysql_mutex_lock(&m_cache_flush_mutex);
  l_cache_flusher_global_version = version();
  do {
    entry = static_cast<Acl_hash_entry *>(
        lf_hash_random_match(&m_cache, pins, &cache_flusher, 0, nullptr));
    if (entry && !lf_hash_delete(&m_cache, pins, entry->key, entry->key_length)) {
      my_free(entry->key);
      delete entry->map;       // Hash element is removed from cache; safe to delete
    }
    lf_hash_search_unpin(pins);
  } while (entry != nullptr);
  lf_hash_put_pins(pins);
  mysql_mutex_unlock(&m_cache_flush_mutex);
}
```

引用计数 > 0 的 `Acl_map` **不会被删**，等 `return_acl_map()` 减到 0 后由下一次 `flush_cache()` 回收。所以 `Acl_cache_items_count` 在 FLUSH 后可能仍非 0（有会话 checkout 未归还）。

#### `--skip-grant-tables`：权限系统的总开关

```cpp
bool initialized = false;
bool skip_grant_tables(void) { return !initialized; }
```

- 启动带 `--skip-grant-tables` ⇒ `acl_init(true)` 直接 return，不读任何 ACL 表 ⇒ `initialized = false`；
- `acl_getroot()` 开头：`if (!initialized) { sctx->skip_grants(); return false; }` —— **所有人拿到"跳过权限检查"的上下文**；
- 语句级门禁：`command_satisfy_acl_cache_requirement()` 用 `sql_command_flags[cmd] & CF_REQUIRE_ACL_CACHE` 把 `GRANT`/`REVOKE`/`CREATE USER`/`SET ROLE`/`SHOW GRANTS`/… 全部拦掉（`ER_OPTION_PREVENTS_STATEMENT`）；
- **`FLUSH PRIVILEGES` 不在禁名单里** ⇒ 这正是"skip-grant-tables 下重新启用权限检查"的标准做法（此时 `acl_reload()` 会真正把表装载起来，`initialized` 变 true）。
- 注意 `grant_reload()` 在 `!initialized` 时直接 return（`acl_reload()` 不检查），所以 skip-grant-tables 下的第一次 FLUSH 只装载 global/db 级，不装表/列/routine 级。

---

## ★ 本机制里的工程实现技法

### 一、高级语法技巧 / C++ 特性

#### `std::atomic` 与 Sys_var 体系的适配 hack

`partial_revokes` 的真实状态是 `std::atomic<bool>`，但 `Sys_var_bool` 只能操作普通 bool。源码的做法是保留一个"影子"变量 `opt_partial_revokes` 让 Sys_var 读写，再由 `ON_UPDATE` 回写原子变量：

```cpp
// sql/mysqld.cc
std::atomic<bool> partial_revokes;
bool opt_partial_revokes;  // Initialized through Sys_var

bool mysqld_partial_revokes() { return partial_revokes.load(std::memory_order_relaxed); }
void set_mysqld_partial_revokes(bool value) { partial_revokes.store(value, std::memory_order_relaxed); }
```

```cpp
// sql/mysqld.h —— 头注释是一份"使用契约"
/*
  @warning : The real value is in @ref partial_revokes. The @ref
  opt_partial_revokes is just a tool to trick the Sys_var class into
  operating on an atomic variable.
  Thus : do not use or access @ref opt_partial_revokes in your code.
         If you need the value of the flag please use the @ref partial_revokes global.
  @todo : opt_partial_revokes to be removed when the Sys_var classes can operate
          safely on an atomic.
*/
```

**为什么这么做**：`mysqld_partial_revokes()` 位于 `acl_get()` / `check_access()` 等极高频路径，必须是无锁读；而 Sys_var 体系是通用的、不支持 atomic。**代价**：两个变量必须靠 `ON_UPDATE` 保持同步，且"影子变量"极易被误用——所以头文件里写了 `@warning` 明令禁止。这是"通用框架不适配时的最小侵入适配"的典型。

#### boost::graph 的自定义属性 + visitor

```cpp
namespace boost {
enum vertex_acl_user_t { vertex_acl_user };
BOOST_INSTALL_PROPERTY(vertex, acl_user);
}  // namespace boost

typedef boost::property<boost::vertex_acl_user_t, ACL_USER,
                        boost::property<boost::vertex_name_t, std::string>>
    Role_properties;
```

`BOOST_INSTALL_PROPERTY` 注册一个自定义顶点属性 tag，之后就能用 `boost::get(boost::vertex_acl_user_t(), graph)[v]` 取到**整个 `ACL_USER` 对象**（值语义拷贝）。源码里有一处 TODO 抱怨这一点：

```cpp
/**
  Custom vertex properties used in Granted_roles_graph
  TODO ACL_USER contains too much information. We only need global access,
  username and hostname. If this was a POD we don't have to hold the same
  mutex as ACL_USER.
*/
```

以及一处因 strict-aliasing 警告而做的 up-cast：

```cpp
    /* Add global access */
    /*
      Up-cast to base class to avoid gcc 7.1.1 warning:
      dereferencing type-punned pointer will break strict-aliasing rules
     */
    *m_access |= implicit_cast<ACL_ACCESS *>(&acl_user)->access;
```

**为什么用 visitor 而不是手写 BFS**：`boost::breadth_first_search(g, start, boost::color_map(v_color).visitor(vis))` 一行搞定"遍历 + 环免疫 + 回调"。`Get_access_maps` 继承 `boost::default_bfs_visitor`，只 override `discover_vertex()`（累加权限）与 `examine_edge()`（收集 WITH ADMIN 角色名）。这是 **Visitor 模式**在算法库里的标准落地：算法骨架固定，行为外挂。

#### `unique_ptr_destroy_only`：所有权与析构的解耦

`column_priv_hash` 的元素类型是 `unique_ptr_destroy_only<GRANT_TABLE>`——**对象内存由 MEM_ROOT (`memex`) 拥有，`unique_ptr` 只负责析构，不负责释放**。这解决了"C 风格内存池 + C++ 对象生命周期"的经典矛盾：

```cpp
mem_check = new (*THR_MALLOC) GRANT_COLUMN(*res, fix_rights_for_column(priv));
hash_columns.emplace(mem_check->column,
                     unique_ptr_destroy_only<GRANT_COLUMN>(mem_check));
```

`new (*THR_MALLOC)` 是 MEM_ROOT 的 placement new；`unique_ptr_destroy_only` 保证容器析构/erase 时调用 `~GRANT_COLUMN()`（否则 `hash_columns` 里的 `std::string column` 会泄漏），但不调 `operator delete`（内存归 MEM_ROOT，会在 `memex.Clear()` 时整体释放）。**这是 C 与 C++ 边界处理的一个干净范例。**

#### RAII 锁守卫：`Acl_cache_lock_guard`

```cpp
enum class Acl_cache_lock_mode { READ_MODE = 1, WRITE_MODE };

class Acl_cache_lock_guard {
 public:
  Acl_cache_lock_guard(THD *thd, Acl_cache_lock_mode mode);
  ~Acl_cache_lock_guard() { unlock(); }
  bool lock(bool raise_error = true);
  void unlock();
 private:
  bool already_locked();
  THD *m_thd;
  Acl_cache_lock_mode m_mode;
  bool m_locked;
};

static const ulong ACL_CACHE_LOCK_TIMEOUT = 3600UL;
static const MDL_key ACL_CACHE_KEY(MDL_key::ACL_CACHE, "", "");

bool Acl_cache_lock_guard::lock(bool raise_error) {
  assert(!m_locked);
  if (already_locked()) return true;             // 可重入：已持有同级或更强的锁就直接成功
  mysql_mutex_assert_not_owner(&LOCK_open);      // 契约：拿 ACL 锁时不能持有 LOCK_open

  MDL_request lock_request;
  MDL_REQUEST_INIT_BY_KEY(
      &lock_request, &ACL_CACHE_KEY,
      m_mode == Acl_cache_lock_mode::READ_MODE ? MDL_SHARED : MDL_EXCLUSIVE,
      MDL_EXPLICIT);
  Acl_cache_error_handler handler;
  m_thd->push_internal_handler(&handler);
  m_locked = !m_thd->mdl_context.acquire_lock(&lock_request, ACL_CACHE_LOCK_TIMEOUT);
  m_thd->pop_internal_handler();

  if (!m_locked && raise_error)
    my_error(ER_CANNOT_LOCK_USER_MANAGEMENT_CACHES, MYF(0));
  return m_locked;
}
```

三个设计点：① RAII + 支持提前 `unlock()`（`acl_get()` 里就先 unlock 再拿写锁插 `db_cache`）；② `already_locked()` 使嵌套调用安全；③ `push_internal_handler` 抑制错误，配合 `raise_error=false` 实现"拿不到锁就静默降级为无权限"（`get_table_grant()` 返回 `NO_ACCESS`）。

### 二、经典算法的实现落地（不限于并发）

#### LF_HASH 的"版本编进 key"技巧（与教科书原型的差异）

| 维度 | 教科书 / 通用做法 | `Acl_cache` 的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 失效方式 | 遍历删除 / 惰性 TTL / 版本号字段比对 | **版本号编进 hash key**（`user@host`version`roles…`） | 失效从"写操作"变成"key 自然不同" ⇒ 读路径完全无锁；代价是同一用户在同一版本下换角色集会生成新条目（内存放大），且旧条目要等 refcount 归零才回收 |
| 删除时机 | 引用计数归零立即删 | **只在 `flush_cache()` 时删**，且只删 `refcount == 0 && version < current` | 避免"正在使用中被删"；代价是内存回收滞后，`Acl_cache_items_count` 在 FLUSH 后可能仍非 0 |
| 并发插入 | 锁 / CAS | `lf_hash_insert` 返回非 0（重复）就释放自己的、递归重试 | 用"乐观创建 + 冲突放弃"代替加锁 |

**代码证据**（见 [六、`Acl_cache`](#acl_cache版本编进-key-的-lf_hash) 的 `checkout_acl_map`）：`rc != 0` 时 `my_free(key); delete map; return checkout_acl_map(...)`——典型的乐观并发。

#### BFS + color map：用"遍历算法自带的性质"替代环检测

| 维度 | 教科书原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 角色图有环怎么办 | 显式拓扑排序 / 环检测 / 深度限制 | **两处不同策略**：GRANT 期用 BFS 主动拒绝成环（`check_if_granted_role_recursive`）；展开期**完全依赖 BFS 的 color map**，无深度限制、不报错 | 运行期假设"图无环"（防线在写侧）；代价是手工往 `mysql.role_edges` 插记录可以造出环，此时行为是"静默地每个顶点只算一次"，无告警 |
| 权限聚合 | — | `discover_vertex()` 里对 6 个 map + 全局位图 + restrictions 分别累加 | 所有合并都是 `|=`（角色只增权限），restrictions 用专门的 aggregator 处理 |

**代码证据**：`boost::vector_property_map<boost::default_color_type> v_color(boost::num_vertices(*g_granted_roles))` + `boost::breadth_first_search(..., boost::color_map(v_color).visitor(vis))`。

#### `get_sort()`：把"通配符权重"编码进一个整数

查找账号时"最具体的账号必须排在最前"——`acl_users` 是**有序数组**，排序键由 `get_sort()` 生成：

```cpp
/*
  Return a number which, if sorted 'desc', puts strings in this order:
    no wildcards
    strings containing wildcards and non-wildcard characters
    single muilt-wildcard character('%')
    empty string
*/
ulong get_sort(uint count, ...) {
  /* Should not use this function with more than 4 arguments for compare. */
  assert(count <= 4);
  while (count--) {
    char *start, *str = va_arg(args, char *);
    uint chars = 0;
    uint wild_pos = 0;
    if ((start = str)) {
      for (; *str; str++) {
        if (*str == wild_prefix && str[1]) str++;
        else if (*str == wild_many || *str == wild_one) {
          wild_pos = (uint)(str - start) + 1;
          if (!(wild_pos == 1 && *str == wild_many && *(++str) == '\0'))
            wild_pos++;
          break;
        }
        chars = 128;  // Marker that chars existed
      }
    }
    sort = (sort << 8) + (wild_pos ? min(wild_pos, 127U) : chars);
  }
  return sort;
}
```

**与教科书原型的差异**：这不是字典序，而是"**通配符出现位置越靠后权重越大、无通配符权重最大**"的自定义全序，且最多 4 个字段各占 8 位打包进一个 `ulong`。`ACL_compare` / `ACL_USER_compare` 里再叠加 "有 IP 的排前、掩码长的排前" 的规则。**收益**：O(1) 的整数比较代替逐字符的通配符分析；**代价**：排序语义藏在位打包里，可读性极差。

#### 双缓冲 + 原子替换（RCU 的工程简化版）

| 维度 | 教科书 RCU | `acl_reload()` 的落地 | 差异原因 |
|---|---|---|---|
| 读者 | 无锁、无需屏障（或仅读屏障） | 必须拿 `Acl_cache_lock_guard(READ_MODE)`（MDL） | 用 MDL 换取"可超时 + 可死锁检测 + 与表锁统一排序"（WL#14084 的动机）；代价是读路径有锁开销 |
| 写者 | 复制 → 发布 → 宽限期后回收 | new 新数组 → 装载 → 指针替换 → `delete` 旧数组（**在写锁下，无需宽限期**） | 写锁保证没有读者 ⇒ 不需要真正的 grace period；代价是写期间完全阻塞 |
| 失败回滚 | — | 保留 `old_*` 指针整体换回 + `acl_free()` | 装载失败不能留下半新半旧的状态（`acl_load` 与 `reload_roles_cache` 必须都成功） |

#### 分层缓存：三级的分工

```
L1  Table_ref::grant（per-语句、per-表）
    失效：grant->version != grant_version        ← 全局计数器，GRANT/REVOKE/FLUSH 时 ++
L2  column_priv_hash / proc_priv_hash（全局）
    失效：被 replace_*_table 就地改 / 被 grant_reload 整体替换
L3  Acl_cache（LF_HASH，per-(用户, role集, 版本)）
    失效：版本编进 key，自然不命中
```

三级缓存覆盖三种不同的"复用距离"：L1 是同一语句内多次列级判定的复用；L2 是跨会话、跨语句的表级结果复用；L3 是跨语句的 role 展开结果复用。**代价是失效信号要穿透三级**（`grant_version` 与 `m_role_graph_version` 两个独立计数器），这也是"GRANT 后哪一级立刻生效"这个问题的全部复杂度来源。

### 三、复杂体系与设计模式的代码结构

#### ACL 缓存锁：从 `Partitioned_rwlock` 到 MDL 的体系切换

`sql/auth/partitioned_rwlock.h` 还在，但**生产代码已无实例**（只有 `partitioned_rwlock-t.cc` 单测在用）。它的设计值得一提：

```cpp
/**
  *) Read lock is acquired only on one of its partitions. The specific
     partition is chosen according to thread id.
  *) Write lock is acquired on all partitions.
  ...
  OTOH acquisition of write lock becomes more expensive. So this rwlock
  is not supposed to be used in cases when number of write requests is
  significant.
*/
```

8.0 换成 MDL（`MDL_key::ACL_CACHE`）后，`sql/mdl.cc` 专门为这个 namespace 做了大量优化——它承认这是一个**被每条语句都拿的单例锁**：

```cpp
  bool is_lock_object_singleton(const MDL_key *mdl_key) const {
    return (mdl_key->mdl_namespace() == MDL_key::GLOBAL ||
            mdl_key->mdl_namespace() == MDL_key::COMMIT ||
            mdl_key->mdl_namespace() == MDL_key::ACL_CACHE ||
            mdl_key->mdl_namespace() == MDL_key::BACKUP_LOCK);
  }

  bool is_affected_by_max_write_lock_count() const {
    /*
      Disable max_write_lock_count handling for ACL_CACHE namespace to
      enable optimization that avoids find_deadlock() for it.
    */
    return key.mdl_namespace() != MDL_key::ACL_CACHE && ...;
  }
```

以及**延迟死锁检测**：

```cpp
    /*
      ... We do this for specific case when we try to acquire S lock on
      ACL_CACHE singleton. While being singleton this lock is acquired by each
      connection attempt and in some cases even by each statement, so on
      systems with many connections deadlock detection can get really
      expensive. ... If threads acquiring S lock on ACL_CACHE have to wait for
      more than 1 second, then the system is likely to be in trouble (as it
      means more than 1 second stalls for any statement doing privilege
      checks).
    */
    bool delayed_find_deadlock = false;
    if (lock->key.mdl_namespace() != MDL_key::ACL_CACHE || ...) {
      find_deadlock();
    } else if (has_locks()) {
      delayed_find_deadlock = true;
    }
```

**这是"用一个通用子系统（MDL）替换专用原语（分片 rwlock）"的完整案例**：收益是可超时、可死锁检测、与表锁统一排序（WL#14084 的直接动机是"只有 `mysql` 库 SELECT 权限的用户就能完全阻塞所有用户管理 DDL"）；代价是要为这个高频单例做专门优化（延迟死锁检测、跳过 `max_write_lock_count`）。

#### 设计模式清单

| 模式 / 体系 | 代码落点 | 结构说明 |
|---|---|---|
| **Visitor** | `Get_access_maps : public boost::default_bfs_visitor` | BFS 骨架由 boost 提供，权限累加行为外挂到 visitor 的 `discover_vertex()` / `examine_edge()` |
| **RAII 锁守卫** | `Acl_cache_lock_guard`、`Column_privilege_tracker`、`Partitioned_rwlock_read_guard`（历史） | 构造加锁、析构解锁，支持提前 `unlock()` |
| **Factory（按 SQL 命令分派）** | `Restrictions_aggregator_factory::create(thd, ...)` 按 `thd->lex->sql_command` 返回 6 种聚合器之一 | partial revoke 在 GRANT/REVOKE 的 4 种形态（global/db/revoke/revoke-all）+ role 场景下语义完全不同，用工厂把差异封进策略对象 |
| **Functor（可空操作的写盘器）** | `Update_dynamic_privilege_table`：`m_no_update=true` 时 `operator()` 直接返回 false | 装载 `global_grants` 时只改内存不写盘（`Update_dynamic_privilege_table no_update;`），GRANT 时既改内存又写盘 |
| **内部 schema/table 的策略对象** | `ACL_internal_schema_access` / `ACL_internal_table_access` + 三态 `check()` | I_S / P_S / `mysql` 库的特殊规则用策略对象外挂，避免污染主判定链；三态返回支持"部分放行" |
| **Rewriter（语句重写）** | `Rewriter_show_create_user : Rewriter_user` | `SHOW CREATE USER` 复用 `CREATE USER` 的重写器，只 override 三处 |
| **双缓冲** | `acl_reload()` / `grant_reload()` | new 新 → 装载 → 原子替换 → delete 旧的；失败整体回滚 |

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `partial_revokes` | **OFF**（`DEFAULT_PARTIAL_REVOKES`，CMake 决定，社区为 0） | Global（+ 命令行 `--partial-revokes`，`PARSE_EARLY`） | 打开后：① 允许"全局权限上挖洞"；② **库名通配符退化为字面匹配**；③ 存在任何 restriction 时**无法关闭** |
| `activate_all_roles_on_login` | **`false`** | Global | 登录时激活所有已授予 + mandatory 角色（否则只激活默认角色） |
| `mandatory_roles` | **`""`** | Global | 视作"授予给所有用户"的角色；不写 `role_edges`；设置需 `ROLE_ADMIN` + `SYSTEM_VARIABLES_ADMIN`；带 `SYSTEM_USER` 的角色不能列入 |
| `--skip-grant-tables` | 关闭 | 启动项 | 不读 grant 表、不做访问控制；`FLUSH PRIVILEGES` 可重新启用 |

| 状态变量 | 说明 |
|---|---|
| **`Acl_cache_items_count`** | `Acl_cache`（LF_HASH）里的条目数，即"已算好的 (用户 × 角色集 × 版本) 权限快照"数量。★ 这是权限子系统**唯一**对外暴露的数值型指标。长期居高 ⇒ 带 active role 的会话很多，或有会话长期 checkout 未归还 |

> 注：全仓库 grep 无 `Acl_cache_items`（无 `_count`）、无 `Acl_cache_roles_count`；`grant_version` 与 `m_role_graph_version` 都是内部变量，未注册为 status var。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 某用户有什么权限（含 role 展开） | SQL | `SHOW GRANTS FOR u@h`；`SHOW GRANTS FOR u USING r1,r2` 看"如果激活这些角色会怎样" |
| 全局/库/表/列级权限的扁平清单 | SQL | `information_schema.USER_PRIVILEGES` / `SCHEMA_PRIVILEGES` / `TABLE_PRIVILEGES` / `COLUMN_PRIVILEGES`（**读内存 ACL 缓存**） |
| 角色层次（谁继承了谁） | SQL | `information_schema.APPLICABLE_ROLES`（递归 CTE，**读 `mysql.role_edges` 基表**）、`ENABLED_ROLES`、`ADMINISTRABLE_ROLE_AUTHORIZATIONS` |
| 动态权限有谁持有 / 注册表里有什么 | SQL | `mysql.global_grants`；注册表本身无视图，只能 `SHOW GRANTS` 反推 |
| partial revoke 是否生效 | SQL | `SHOW GRANTS` 里是否出现 `REVOKE ... ON db.* FROM ...`；`SELECT User_attributes FROM mysql.user`（JSON 里的 `Restrictions`） |
| ACL 缓存的规模 / 是否泄漏 | SQL | `SHOW GLOBAL STATUS LIKE 'Acl_cache_items_count'` |
| ACL 缓存锁争用 | PFS | `SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_TYPE='ACL CACHE'`（MDL 名 `wait/lock/metadata/sql/mdl`）；相关 mutex instrument：`wait/synch/mutex/sql/LOCK_acl_cache_flush`、`LOCK_mandatory_roles`、`LOCK_partial_revokes` |
| 权限子系统是否初始化失败 / 表结构不符 | 日志 | `ER_AUTHCACHE_CANT_OPEN_AND_LOCK_PRIVILEGE_TABLES`、`ER_AUTHCACHE_USER_TABLE_DODGY`、`ER_AUTHCACHE_TABLE_PROXIES_PRIV_MISSING` |
| partial_revokes 被自动打开 | 日志 | `ER_TURNING_ON_PARTIAL_REVOKES` |
| mandatory_roles 配错 | 日志 | `ER_UNKNOWN_AUTH_ID_IN_MANDATORY_ROLE`、`ER_ANONYMOUS_AUTH_ID_NOT_ALLOWED_IN_MANDATORY_ROLES` |
| role_edges / default_roles 里有脏记录 | 日志 | `ER_AUTHCACHE_ROLE_EDGES_UNKNOWN_AUTHORIZATION_ID`、`ER_AUTHCACHE_DEFAULT_ROLES_UNKNOWN_AUTHORIZATION_ID` |
| 库名大小写导致 revoke 失效 | 日志 | `ER_AUTHCACHE_DB_ENTRY_LOWERCASED_REVOKE_WILL_FAIL` |
| 账号因 plugin 缺失/为空而无法登录 | 日志 | `ER_AUTHCACHE_USER_IGNORED_NEEDS_PLUGIN`、`ER_AUTHCACHE_PLUGIN_MISSING`、`ER_AUTHCACHE_PLUGIN_CONFIG` |
| 判定过程（每次 check_access 的入参与结果） | DBUG | `DBUG_PRINT("enter", ("db: %s want_access: %" PRIu32 " master_access: %" PRIu32, ...))` 在 `check_access()` 开头；`--debug='d,check_access'` |

---

## Misc

### 扩展点：新增一种动态权限要动哪些地方

这是一个**不需要改表结构**的扩展，也是动态权限机制的设计目标。步骤：

1. **注册**：组件在初始化序列里通过 service `dynamic_privilege_register.mysql_server` 调 `register_privilege("MY_ADMIN", ...)`。服务器内置权限则加到 `sql/auth/dynamic_privileges_impl.cc` 的 `dynamic_privilege_init()`（注意它会 `toupper`，且重复注册"成功"）。
2. **判定**：在需要校验的地方写 `sctx->has_global_grant(STRING_WITH_LEN("MY_ADMIN")).first`（要 GRANT OPTION 就取 `.second`）。若希望它可被 `SUPER` 替代，用 `!sctx->check_access(SUPER_ACL) && !sctx->has_global_grant(...).first` 的组合模式。
3. **持久化**：不用管——`GRANT MY_ADMIN ON *.* TO u` 会自动写 `mysql.global_grants`（`modify_dynamic_privileges_in_table()`），启动时由 `populate_dynamic_privilege_caches()` 装回内存。
4. **卸载**（可选）：组件 UNINSTALL 时**官方实践是不注销**（"current practice, not a requirement"）。若确实要注销，调 `unregister_privilege()`——但注意注册表不记录归属，任何组件都能注销别人的名字。
5. **文档/错误提示**：若权限名要出现在错误信息里，`global_acls_vector` **只覆盖静态权限**，动态权限的名字就是注册时的字符串（大写）。

**对比：新增一种静态权限**则要动 `auth_acls.h`（加位，注意 `NUM_ACLS` 上限 31，位图已用满）、`auth_acls.cc`（`global_acls_vector` / `global_acls_map`）、`mysql.user` 表结构（加列）——**成本远高于动态权限**，这正是动态权限存在的理由。

### 坑与已知缺陷

以下每条都能落到源码或官方材料。

1. **`unregister_privilege` 无法校验归属**。`Dynamic_privilege_register` 只是 `unordered_set<string>`，不记录注册者。任何组件都能注销别人的权限名（`rewriter_plugin.cc` / `version_token.cc` 在 deinit 时无条件 unregister）。
2. **重复注册"静默成功"**（`if (reg->find(priv) != reg->end()) return false;`）。组件 A 注册 `FOO_ADMIN`，组件 B 注册同名也"成功"，但 A 卸载时会把它删掉。
3. **`mysql.global_grants` 里的任何权限名都会被就地注册**（源码注释："The policy is that any privilege ID that exist in mysql.global_grants is a valid privilege ID"）。⇒ 能写这张表就等于能发明权限名。**官方手册也警告：不要给 regular 账号授予 `mysql` schema 的权限**（"do not grant mysql schema privileges to regular accounts"），因为那可以改密码、加权限。
4. **`GRANT ALL ON *.*` 只授予"当前已注册"的动态权限**，之后注册的不追溯。`REVOKE ALL` 则撤销所有已授予的静态 + 动态权限（不对称）。
5. **动态权限与 GRANT OPTION 是"只增不减"**（`grant_dynamic_privilege` 注释："We never remove the GRANT OPTION as a result of GRANT statement"）。
6. **partial revoke 的 db 名比较是大小写敏感的**（裸 `unordered_map<string, Access_bitmask>`，无自定义 hasher），而 `mysql.db` 的 db 名按 `lower_case_table_names` 折叠（源码里专门有 `ER_AUTHCACHE_DB_ENTRY_LOWERCASED_REVOKE_WILL_FAIL` 告警）⇒ `lower_case_table_names=1` 下用混合大小写库名做 partial revoke 可能**失效**。
7. **role 的 restrictions 传播是"覆盖式"而非"并集式"**：`Get_access_maps::discover_vertex()` 里 `m_restrictions->set_db(db_restrictions)` 每个 role 顶点重算并覆盖，最终值依赖 BFS 遍历顺序。
8. **`SET GLOBAL partial_revokes=OFF` 在有 restriction 时必被拒**（`ER_PARTIAL_REVOKES_EXIST`）；且 `sys_vars.cc` 的注释承诺"ON 时若存在 DB 通配符授权则报错"，但 `check_partial_revokes()` **只实现了 OFF 方向的检查**——注释与代码不符。
9. **`SET PASSWORD` / `ALTER USER ... IDENTIFIED BY` 并不检查 `is_role`**：源码里没有"禁止给 role 设密码"的显式拦截（`change_password()` 只是把 `is_role` 传给审计）。role 不能登录是因为 `CREATE ROLE` 设了 `account_locked=Y` + 无密码。
10. **`ER_ROLE_GRANTED_TO_ITSELF` 抛出后没有 `break`/`continue`**（`mysql_grant_role()`），紧接着仍会执行 `grant_role(...)` 与 `modify_role_edges_in_table(...)`。错误信息已抛，但循环继续往下走。
11. **`vertex_count` 逻辑缺陷**：`get_privilege_access_maps()` 里 `++vertex_count` 写在 `else` 分支，导致 `breadth_first_visit` 分支不可达，多激活角色时 color map 被重复初始化（功能正确，但优化未生效）。
12. **`is_role` 是拓扑推导值**，不是持久字段 ⇒ 手工改 `mysql.role_edges` 可以改变"谁是 role"。
13. **直接 DML 改 grant 表可能让实例起不来**：[Bug #100468](https://bugs.mysql.com/bug.php?id=100468) —— 手工往 `mysql.role_edges` / `default_roles` 插入指向不存在 authid 的记录后，`FLUSH PRIVILEGES` 报 `ER_ACCESS_DENIED`-类错误，**重启 mysqld 直接起不来**（`The privilege system failed to initialize correctly`），只能 `--skip-grant-tables` 清理。这与官方手册 "Direct modification of grant tables … is discouraged and done at your own risk" 相呼应。
14. **`GRANT` 静态全局权限对已连接会话不生效**（`m_master_access` 是登录时快照），而库/表/列级立刻生效——这两者的差异是"GRANT 后要不要重连"这个问题的唯一正确答案（官方手册 [8.2.13](https://dev.mysql.com/doc/refman/8.0/en/privilege-changes.html) 有同样的表格）。

### 社区边界澄清

| 有人以为有 | 实际 |
|---|---|
| 行级安全（RLS / VPD / row filter） | **社区版没有**。最小粒度是列级（`mysql.columns_priv`）。社区里"按行过滤"只能靠视图 + `SQL SECURITY`（见 [`definer.md`](definer.md)）。Oracle 有 VPD，PostgreSQL 9.5+ 有 `CREATE POLICY`。 |
| 表级/库级动态权限 | **没有**。动态权限只有全局级（`global_grants` 表没有库/表列）。 |
| `SHOW ROLES` 语句 | **没有**。角色列表只能查 `mysql.user`（`is_role` 是内存标志，盘上无列）或 `information_schema.APPLICABLE_ROLES` / `ENABLED_ROLES`。 |
| `information_schema.DEFAULT_ROLES` 视图 | **不存在**（只有基表 `mysql.default_roles`）。 |
| 角色不需要激活（像 PG 那样登录即继承） | **需要激活**。`GRANT r TO u` 之后 `CURRENT_ROLE()` 仍是 `NONE`。 |
| `REFRESH_PRIVILEGES` / `flush_roles()` | **不存在**。`FLUSH PRIVILEGES` 用的是 `REFRESH_GRANT`。 |
| `LOCK_grant` rwlock（PFS instrument） | **8.0.39 已不存在**（只在旧 mtr 文件里残留）。ACL 缓存由 MDL `MDL_key::ACL_CACHE` 保护。 |

### 认证 / 授权 / 鉴权：术语划分

中文技术语境里这三个词长期混用，先定下本篇的用法（依据 MySQL 官方手册 [8.2 Access Control and Account Management](https://dev.mysql.com/doc/refman/8.0/en/access-control.html)）：

| 概念 | 英文 | 回答什么问题 | MySQL 手册对应 | 源码归属 |
|---|---|---|---|---|
| **认证** | Authentication（**AuthN**） | "你是谁" | [8.2.6 Access Control, Stage 1: Connection Verification](https://dev.mysql.com/doc/refman/8.0/en/connection-access.html) + 8.2.17 Pluggable Authentication + 8.2.18 MFA | 连接握手、`acl_authenticate()`、认证插件 → 见 [`authentication.md`](authentication.md)（**本篇不讲**） |
| **授权** | Authorization（**AuthZ**） | "你能做什么" | [8.2.7 Access Control, Stage 2: Request Verification](https://dev.mysql.com/doc/refman/8.0/en/access-control.html) + 8.2.2/8.2.3/8.2.10/8.2.12/8.2.13 | **本篇**：`check_access()` 判定链、ACL 缓存、role、动态权限 |
| **账户管理** | Account Management | "账户本身怎么建/改/限" | 8.2.1/8.2.8/8.2.14~8.2.16/8.2.20/8.2.21 | `CREATE/ALTER/DROP USER`、密码策略、资源限额（`USER_RESOURCES`）、账户锁定 |
| **审计** | Accounting / Audit | "做了什么"（AAA 的第三个 A） | 8.2.23 SQL-Based Account Activity Auditing | audit plugin / audit log 组件 |

★ **"鉴权"这个词本篇不用作分类轴**。它在中文语境里至少有三种用法——① 指认证（验证 token/签名）、② 指授权（校验权限）、③ 泛指 AuthN+AuthZ 整个访问控制（云厂商文档常见）。歧义太大，写作时一律用**认证**/**授权**这两个无歧义的词，英文对应 AuthN / AuthZ。

注意官方章名是 "Access Control **and Account Management**"——访问控制（认证+授权）与**账户管理是并列的两件事**，不要混为一谈：前者是运行时的判定，后者是对 `mysql.user` 行的增删改（`CREATE USER` 属于账户管理，不属于授权，尽管它常常顺带 `GRANT`）。

### 易混淆点

| 概念 A | 概念 B | 区别 |
|---|---|---|
| role 授予（`GRANT r TO u`） | role 激活（`SET ROLE r` / 默认角色 / `activate_all_roles_on_login`） | 授予写 `mysql.role_edges`，激活只改会话的 `m_active_roles`。**只有激活的角色才参与权限展开**（`all_granted_roles ∩ all_active_roles`） |
| 静态权限 | 动态权限 | 静态是 `mysql.user` 的一列 + 一个位（`Access_bitmask`），31 个，永远存在；动态是 `mysql.global_grants` 的一行 + 注册表里的一个字符串，可注册/注销，只有全局级 |
| `GRANT ALL ON *.*` | `SUPER` | `GRANT ALL` **不包含** `SUPER` 之外的……注意：`GRANT ALL ON *.*` 包含全部静态全局权限 + 已注册的动态权限，但**不含** `GRANT OPTION`（除非显式写 `WITH GRANT OPTION`）；且 `PROXY` 需要单独 `GRANT PROXY` |
| `check_access()` 自由函数 | `Security_context::check_access()` 成员函数 | 前者做 global+db+内部 schema 三层组合（有副作用：写 `*save_priv`、可能 `my_error`）；后者只是"全局位图够不够"的纯谓词（无副作用），且是 partial_revokes 的渗透点之一 |
| `check_grant()` 的多表 AND | `any_combination_will_do` 的 OR | 前者是表间关系（全部表都要满足）；后者是**单表内部的列组合 OR**（`SHOW COLUMNS` / `SHOW INDEX` 用） |
| `GRANT_INFO::privilege` | `GRANT_INFO::grant_table` | 前者是"已满足的权限位图"（各层 `|=` 累加）；后者是二级缓存里查到的 `GRANT_TABLE*` 指针，同时充当"要不要查列级"的开关 |
| `grant_version` | `Acl_cache::m_role_graph_version` | 前者驱动 L1（`Table_ref::grant.version`）失效，在 `mysql_grant` / `acl_load` / `grant_reload` 时 `++`；后者驱动 L3（`Acl_map`）失效，被编进 LF_HASH 的 key |

### 一句话总结

授权子系统把权限组织成"**磁盘 grant 表 → 内存分层容器 → per-会话快照**"的三级缓存，判定链用 `want_access` 的**逐层扣除**（而非短路）遍历 global → 内部 schema → db → table → column 五层；role 用**有向图 + BFS 展开 + 版本化快照缓存**把间接层的成本摊掉；动态权限用**可扩展的字符串集合**避开表结构膨胀；partial_revokes 用一个**全局开关**补上"全局权限挖洞"的表达力缺口，代价是改变了全实例的通配符库名语义。

---

## 参考

**官方文档**

- *MySQL 8.0 Reference Manual → 8.2.13 [When Privilege Changes Take Effect](https://dev.mysql.com/doc/refman/8.0/en/privilege-changes.html)*（★ 最关键的一节：内存缓存的存在、何时需要/不需要 `FLUSH PRIVILEGES`、变更对已连接会话的生效时机表）
- *MySQL 8.0 Reference Manual → 8.2.3 [Grant Tables](https://dev.mysql.com/doc/refman/8.0/en/grant-tables.html)*（10 张 grant 表、InnoDB 事务化、"不建议直接 DML 修改"的官方告警、`User_attributes` JSON 列）
- *MySQL 8.0 Reference Manual → 8.2.2 [Privileges Provided by MySQL](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html)*（含 *Static Versus Dynamic Privileges* 与动态权限版本表）
- *MySQL 8.0 Reference Manual → 8.2.10 [Using Roles](https://dev.mysql.com/doc/refman/8.0/en/roles.html)*（激活语义、`SET ROLE`、mandatory role 与 `SYSTEM_USER` 的互斥）
- *MySQL 8.0 Reference Manual → 8.2.12 [Privilege Restriction Using Partial Revokes](https://dev.mysql.com/doc/refman/8.0/en/partial-revokes.html)*
- *MySQL 8.0 Reference Manual → 8.2.11 [Account Categories](https://dev.mysql.com/doc/refman/8.0/en/account-categories.html)*（`SYSTEM_USER` / power user / regular user）
- *MySQL 8.0 Reference Manual → 15.7.1.6 [GRANT Statement](https://dev.mysql.com/doc/refman/8.0/en/grant.html)*（`AS` 子句）
- *MySQL 8.0 Reference Manual → 16.3 [Transactional Storage of Dictionary Data](https://dev.mysql.com/doc/refman/8.0/en/data-dictionary-transactional-storage.html)*
- *MySQL Secure Deployment Guide → [D SQL Roles and Dynamic Privileges](https://dev.mysql.com/doc/mysql-secure-deployment-guide/8.0/en/secure-deployment-roles-dynamic-privileges.html)*

**WorkLog**

- [WL#988: Roles](https://dev.mysql.com/worklog/task/?id=988)（8.0.0 引入角色；含"不重构既有权限系统"的设计约束、BFS 展开、Acl_cache 设计）
- [WL#12098: MySQL system users](https://dev.mysql.com/worklog/task/?id=12098)（★ 一次引入 `SYSTEM_USER` 与 partial revokes；S-9 的三选一论证、S-10 通配符字面化、binlog `AS` 子句与 `Q_ACTIVE_ROLES` flag）
- [WL#12364: Kill administration for system users](https://dev.mysql.com/worklog/task/?id=12364)（`CONNECTION_ADMIN` 不能 KILL `SYSTEM_USER` 会话；`THD` 上的原子 `system_user` 标志）
- [WL#12820: Extend GRANT syntax to cover partial revokes information](https://dev.mysql.com/worklog/task/?id=12820)
- [WL#14084: Lock improvements for ACL DDLs](https://dev.mysql.com/worklog/task/?id=14084)（ACL DDL 的 MDL 锁方案；HLS 2.3 的死锁 victim 副作用）
- [WL#12217: SESSION_VARIABLES_ADMIN](https://dev.mysql.com/worklog/task/?id=12217)

**Bug 论坛**

- [Bug #100468 — inconcistent records in role_edges block mysqld to start](https://bugs.mysql.com/bug.php?id=100468)（★ 直接 DML 改 `role_edges` 导致实例起不来；S2，官方 Can't repeat 但现象极具警示价值）
- [Bug #111710 — Mysql 8 Role Issue](https://bugs.mysql.com/bug.php?id=111710)（5.7 → 8.0.32 升级后的角色问题；仅检索到摘要，细节未核实）
- [Bug #94394](https://bugs.mysql.com/bug.php?id=94394)（`mysql.user` 损坏时旧版以 skip-grant-tables 启动；8.0.16 起改为直接启动失败）

**内核月报 / 技术文章**

- [数据库内核月报 2019/06 · MySQL · 引擎特性 · 安全及权限改进相关](http://mysql.taobao.org/monthly/2019/06/01/)（基于 8.0.16；Atomic ACL Statement 的升级副作用、Partial Revoker 与 WL#12098 动机、`SYSTEM_USER` + KILL 实测。★ 函数名与行号对不上 8.0.39，只取视角）
- [The MySQL 8.0.0 Milestone Release is available](https://dev.mysql.com/blog-archive/the-mysql-8-0-0-milestone-release-is-available/)（Roles (WL#988) 的官方里程碑说明）
- [The SYSTEM_USER Dynamic Privilege（mysqlserverteam）](https://mysqlserverteam.com/the-system_user-dynamic-privilege/)
- [Partial Revokes from Database Objects（mysqlserverteam）](https://mysqlserverteam.com/partial-revokes-from-database-objects/)

**理论基础**

- Sandhu, Coyne, Feinstein, Youman. *Role-Based Access Control Models*. IEEE Computer, 1996.（RBAC0/RBAC1/RBAC2；MySQL 实现的是 RBAC1 + 部分 RBAC2）
- Lampson, B. *Protection*. ACM SIGOPS, 1971.（Access Matrix；MySQL 按主体聚集、PostgreSQL 按客体聚集的分野）
- Shalev, E. & Shavit, N. *Split-Ordered Lists: Lock-Free Extensible Hash Tables*. JACM 2006.（`LF_HASH`；本库 [`../../infra/structure/hash.md`](../../infra/structure/hash.md) 有本仓库实现剖析）
- Cormen, Leiserson, Rivest, Stein. *Introduction to Algorithms*, 第 22 章（BFS + color map）

**相关文档**

- 认证上下文（"我是谁"）：[`security_context.md`](security_context.md)
- 多因素认证：[`mfa.md`](mfa.md)
- definer 与 SQL SECURITY（权限委托）：[`definer.md`](definer.md)
- 数据字典（grant 表的归属与 `SYSTEM` 类型系统表）：[`../dd/dd.md`](../dd/dd.md)
- 开表与元数据锁（权限先于 MDL）：[`../../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)
- 无锁哈希 `LF_HASH` 与 `Acl_cache` 的底层：[`../../infra/structure/hash.md`](../../infra/structure/hash.md)
- Clone 插件（用到 `CLONE_ADMIN` 动态权限）：[`../plugin/clone.md`](../plugin/clone.md)
- 视图（definer / invoker 的权限上下文，社区"按行过滤"的唯一手段）：[`../view.md`](../view.md)
- 存储程序（SQL SECURITY）：[`../stored_program.md`](../stored_program.md)
