# X Plugin 深度解析

> 基于 MySQL 8.0.39 源码（`plugin/x/`）。X Plugin 是 MySQL **Document Store / X DevAPI 的服务端实现**：在经典协议之外另开一个协议端点（默认 33060），用 protobuf 定义的 X Protocol 承载"集合/文档"风格的 CRUD，内部把它们**翻译成 SQL** 交给服务器执行。
>
> **边界**：本篇讲 X Plugin 自身的架构（协议栈、会话模型、翻译层、认证与缓存）；插件框架见 [`plugin.md`](plugin.md)，组件服务与插件→组件的桥见 [`service.md`](service.md)，ABI 契约见 [`abi.md`](abi.md)。**不讲**：JSON 类型与部分更新的物理实现（`../datatype/json.md`）、Group Replication 本身（`../replication/`）。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [系统变量](#系统变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

X Plugin 是**插件形态的第二个协议端点**：一个 `MYSQL_DAEMON_PLUGIN`（名 `mysqlx`），启动后监听 `mysqlx_port`（默认 33060）与 `mysqlx_socket`，用 **X Protocol**（protobuf）对外提供 X DevAPI——`getCollection().add()/find()/modify()/remove()`、SQL 直执行、事务、notice/expect 等。它**不是**文档数据库：数据仍是 InnoDB 表里的 JSON 列，每个 DevAPI 调用都被翻译成 SQL。

### 用途

- 给不想写 SQL 的开发者（JS/Python 栈）一个"文档式"接口；
- MySQL Shell（`--js` / `--py` 模式）与官方 X DevAPI Connector 的服务端；
- InnoDB Cluster / MySQL Router 生态的一环（感知 GR 成员变化）。

### 版本演进

| 版本/线索 | 变化 | 出处 |
|---|---|---|
| 8.0 | X Plugin 随 8.0 引入（MySQL Document Store） | 插件目录版权年份 2016 起 |
| 8.0 系列 | 协议演进：`mysqlx_prepare.proto`（预处理语句）、`mysqlx_notice.proto`（notice 框架）、压缩能力（`mysqlx_compression_algorithms`）陆续加入 | proto 文件与 capabilities 目录，**具体小版本未核实** |
| 8.0.19+ | 与 GR 的集成（`group_membership_listener` / `group_member_status_listener` 实现） | `plugin/x/src/services/` |
| 构建开关 | `WITH_MYSQLX` 控制是否编译；`MYSQL_ADD_PLUGIN(mysqlx ... DEFAULT ...)` → 默认启用 | `plugin/x/CMakeLists.txt:85-98` |

---

## 理论基础

### 第零条（设计思想）：为什么是"第二个协议端点"，而不是改经典协议？

经典协议（3306）是**文本协议 + 固定命令集**（`COM_QUERY` 等），整个生态——客户端、驱动、中间件、代理、监控——都按这套格式解析。要在里面塞进"集合/文档"语义，以及 DevAPI 需要的会话级能力（notice 推送、expect 条件块、预处理语句、异步流水线），要么给经典协议加命令（污染兼容性），要么另起炉灶。MySQL 选了后者：

| 维度 | 独立端点的收益 | 代价 |
|---|---|---|
| 兼容性 | 3306 的行为完全不受影响 | 多一个监听端口（攻击面） |
| 演进 | 协议可以独立加消息/能力（capabilities 协商） | 两套连接管理、两套超时/限流 |
| 运维 | 可整体开关（`--skip-mysqlx`） | 两个连接池各算一份资源 |
| 实现 | 插件可以自己起线程、自己管会话 | 必须"借用"服务器的 THD（见下） |

**为什么用 protobuf**：IDL 驱动——12 个 `.proto` 直接生成多语言客户端（Connector 覆盖 C/C++/Java/Python/Node/.NET），二进制紧凑、字段号支持向后演进。代价是与 MongoDB 的 BSON 毫无互操作性（见 Misc 的协议辨析）。

### 为什么不直接调服务器的内部函数执行 SQL？

这是本篇最值得记住的一点，也是 [`service.md`](service.md) 里"桥"的最大用例。X 连接跑在**插件自己的 worker 线程**上，要执行 SQL 必须有一个"属于这个线程的 THD"，才能正常走权限检查、事务、锁、审计。而"创建 THD / 执行 SQL"是服务器内部符号——插件若直呼就是 WL#4102 批判的 wormhole（见 [`abi.md`](abi.md) 设计思想 ②）。

于是 X Plugin 走**服务化接口**：

```
worker 线程 → srv_session_init_thread(plugin)      /* 让本线程可作为会话宿主 */
            → srv_session_open(...)                 /* 得到一个内部 THD（MYSQL_SESSION） */
            → command_service_run_command(session, COM_QUERY, ...)   /* 执行 SQL */
            → Command_delegate 回调                 /* 结果集转回 X Protocol */
```

这三个调用全部来自**老版 plugin service**（`service_srv_session.h`、`service_command.h`），而插件同时还经 `plugin_registry_service` 桥消费**组件服务**（`log_builtins`、`component_sys_var`、`udf_registration`、`audit_api_connection`）。X Plugin 因此是"两代服务机制同时在用"的标本级案例。

### 三层"兼容"辨析（最容易误解的地方）

| 层 | MongoDB | X Plugin | 兼容？ |
|---|---|---|---|
| 线协议 | MongoDB Wire Protocol（OP_MSG/OP_QUERY），载荷 **BSON** | X Protocol，**protobuf**（`plugin/x/protocol/protobuf/*.proto`，12 个文件） | ✗ |
| 认证 | SCRAM-SHA-1/256 | **PLAIN / MYSQL41 / SHA256_MEMORY**（`auth_plain.h:56`、`auth_challenge_response.h:44-58`；源码无 SCRAM） | ✗ |
| 客户端 | mongosh / pymongo / mongoose | MySQL Shell / X DevAPI Connector | ✗（驱动不可互换） |
| API 写法 | `db.c.find({age:{$gt:25}})` | `db.getCollection('c').find('age > 25')` | △ 形状相似 |
| 存储执行 | WiredTiger + 自有查询引擎 | SQL + InnoDB（JSON 列） | ✗ |

**结论**：X Plugin 是"**体验兼容**"，不是"协议兼容"。真正的协议兼容（AWS DocumentDB、FerretDB 那类）的判据是"现有 MongoDB 驱动一行不改就能连上"——MySQL 做不到。

---

## 核心实现

### 一个 `.so`，两个插件

`plugin/x/src/xpl_plugin.cc:35-67` 的声明数组里有**两项**：

```c
mysql_declare_plugin(mysqlx){
    MYSQL_AUDIT_PLUGIN,                                /* ① 审计插件，声明在前！ */
    modules::Module_cache::get_audit_plugin_descriptor(),
    "mysqlx_cache_cleaner",
    PLUGIN_AUTHOR_ORACLE,
    "Cache cleaner for sha2 authentication in X plugin",
    PLUGIN_LICENSE_GPL,
    modules::Module_cache::initialize,   /* init */
    nullptr,
    modules::Module_cache::deinitialize, /* deinit */
    0x0100, nullptr, nullptr, nullptr, 0
},
{
    MYSQL_DAEMON_PLUGIN,                               /* ② 真正的 X Plugin */
    &mysqlx_deamon_plugin_descriptor,
    MYSQLX_PLUGIN_NAME,                                /* "mysqlx" */
    ...
    &modules::Module_mysqlx::initialize,
    nullptr,
    &modules::Module_mysqlx::deinitialize,
    MYSQLX_PLUGIN_VERSION,
    modules::Module_mysqlx::get_status_variables(),
    modules::Module_mysqlx::get_plugin_variables(),
    nullptr, 0
} mysql_declare_plugin_end;
```

为什么一个"文档协议服务端"要配一个**审计插件**？因为 X Plugin 为 `SHA256_MEMORY` 认证维护了一份**密码哈希缓存**，而这份缓存必须在"用户/权限发生变化"时失效——用户被删、改密码、授权变更都是**审计事件**。社区的做法正是让插件自己注册成审计插件去监听这些事件（`Module_cache`），从而清缓存。

> **同一套路在服务器自身也有一个**：`caching_sha2_password` 的密码哈希缓存由**同一声明块里的审计插件 `sha2_cache_cleaner`** 负责失效（监听 `MYSQL_AUDIT_AUTHENTICATION_CLASS`）。两处是同一个设计模式——**"插件无法主动感知权限变更，只能把自己注册成审计插件来被动失效缓存"**。X Plugin 侧还带一个额外约束：审计插件必须先初始化，否则要走 `delayed_start_tasks()` 延后监听。服务器侧的完整剖析见 [`../auth/authentication.md`](../auth/authentication.md) 的 §3.2。

**顺序是硬约束**：审计插件声明在前，因此先初始化（`plugin_array` 顺序即初始化顺序，见 [`plugin.md`](plugin.md) 的主链路）。源码注释把这个耦合写得很直白（`module_mysqlx.cc:171-194`）：

```c
// Cache cleaning plugin started before the X plugin so cache was not
// enabled yet ... Thus is almost all cases `if` below should be evaluated as true.
// The problematic case is when module_cache was disabled by the user.
// In this case we should execute delayed startup.
if (Module_cache::m_is_sha256_password_cache_enabled) {
  m_sha256_password_cache->enable();
}
...
if (!Module_cache::m_is_sha256_password_cache_enabled) {
  m_server->delayed_start_tasks();       /* 缓存不在，监听延后启动 */
}
```

### 初始化全流程

```c
int Module_mysqlx::initialize(MYSQL_PLUGIN plugin_handle) {
  xpl::plugin_handle = plugin_handle;
  xpl::init_performance_schema();

  provide_udfs();        /* 注册 3 个 UDF */
  require_services();    /* 获取所需服务（含组件服务，经 registry 桥） */
  provide_services();    /* 反向：登记自己提供的 3 个服务 */

  if (mysqld::get_initialize()) return 0;   /* --initialize 期间不监听 */

  xpl::Server_builder builder(plugin_handle);
  auto update_plugin_vars = builder.get_result_reconfigure_server_callback();
  auto sys_var_service = m_services->m_system_variable_register.get();

  xpl::Global_status_variables::initialize();
  xpl::Plugin_system_variables::initialize(sys_var_service, update_plugin_vars, ...);

  auto acceptor_task = builder.get_result_acceptor_task();
  m_sha256_password_cache = ngs::allocate_object<xpl::SHA256_password_cache>();
  m_input_queue = ngs::allocate_object<xpl::Notice_input_queue>();
  m_server = builder.get_result_server_instance({acceptor_task, m_input_queue->create_broker_task()});
  ...
  if (!m_server->prepare()) return 0;       /* 启动失败也保持插件已加载 */
}
```

三点值得注意：

1. **UDF 与服务的注册都在监听之前**——先把自己作为"能力提供者"接进服务器，再对外服务。
2. **系统变量经 `component_sys_variable_register` 服务注册**（`m_system_variable_register`），而不是只用插件的 `MYSQL_SYSVAR` 数组——这是"插件消费组件服务"的直接证据。
3. **它自己也是服务提供者**：

```c
void Module_mysqlx::provide_services() {
  xpl::Service_registrator r;
  r.register_service(SERVICE(mysql_server, mysqlx_maintenance));       /* reset_global_status_variables */
  r.register_service(SERVICE(mysqlx, group_membership_listener));       /* GR → X */
  r.register_service(SERVICE(mysqlx, group_member_status_listener));
}
void Module_mysqlx::provide_udfs() {
  m_udf_register->insert({UDF(mysqlx_error),
                          UDF(mysqlx_generate_document_id),
                          UDF(mysqlx_get_prepared_statement_id)});
}
```

`mysqlx_maintenance` 这个服务还是**由 X Plugin 自己定义**的（`plugin/x/src/services/mysqlx_maintenance.h:30` 的 `BEGIN_SERVICE_DEFINITION`）——说明组件服务这套机制对插件也是开放的，不只是"消费"。

### 协议栈与消息流

12 个 proto 文件（`plugin/x/protocol/protobuf/`）：`mysqlx`（基础/错误/OK）、`mysqlx_connection`（能力协商/关闭）、`mysqlx_session`（认证/重置/关闭）、`mysqlx_sql`（StmtExecute）、`mysqlx_crud`（Insert/Find/Update/Delete）、`mysqlx_expr`（表达式 AST）、`mysqlx_datatypes`（标量/Any）、`mysqlx_resultset`（列元数据/行/取数完成）、`mysqlx_cursor`、`mysqlx_prepare`（预处理）、`mysqlx_expect`（条件块）、`mysqlx_notice`（警告/会话状态变更/group_replication 状态等）。生成物在 `plugin/x/generated/protobuf/` 与 `protobuf_lite/` 两套（`USE_MYSQLX_FULL_PROTO` 切换）。

一条连接的典型消息流：

```
1) 连接建立        →  Server::on_accept() 取 Vio，建 Client
2) 能力协商        →  Mysqlx::Connection::CapabilitiesGet / CapabilitiesSet
                     （client.cc:223 get_capabilities / :234 set_capabilities）
3) SASL 认证       →  Mysqlx::Session::AuthenticateStart（session.cc:147）
                     PLAIN（明文口令）/ MYSQL41（nonce+scramble）/ SHA256_MEMORY（走缓存）
4) 命令执行        →  Mysqlx::Sql::StmtExecute（stmt_command_handler.cc:42）
                     Mysqlx::Crud::Find / Insert / Update / Delete（crud_cmd_handler.cc:222）
5) 结果/通知       →  mysqlx_resultset 的列元数据+行；mysqlx_notice 的 warning / 会话变更 / GR 状态
```

### 会话与线程模型（难点）

```
监听/accept      Server + Socket_acceptors_task（独立 acceptor 线程）
   │
   ▼
Client（一个连接） ── 交给 Scheduler（worker 线程池）
   │
   ▼
Session ── 绑定一个 MYSQL_SESSION（内部 THD）
```

关键代码（`plugin/x/src/server/session_scheduler.cc:48-68`）——**每个 worker 线程必须先向服务器"登记"自己**：

```c
if (srv_session_init_thread(m_plugin_ptr) != 0) {         /* 本线程可作为会话宿主 */
  log_error(ER_XPLUGIN_SRV_SESSION_INIT_THREAD_FAILED);
  return false;
}
...
srv_session_deinit_thread();
ssl_wrapper_thread_cleanup();
```

会话的生命周期与"附着/分离"（`sql_data_context.cc`）：

```c
m_mysql_session = srv_session_open(&Sql_data_context::default_completion_handler, this);  /* :101 */
...
srv_session_detach(m_mysql_session);      /* :146 —— 线程不再持有该会话 */
srv_session_close(m_mysql_session);       /* :151 */
...
srv_session_attach(m_mysql_session, &previous_thd);   /* :582 —— 重新附着到当前线程 */
```

**attach/detach 为什么必要**：worker 线程是复用的，一次命令结束后线程要交还会话所有权（detach），下次被调度到另一个线程时再 attach。这正是 `srv_session` 服务要解决的核心问题——**让一个不固定的线程安全地使用服务器会话**。

执行 SQL 走 `command` 服务（`sql_data_context.cc:303-306`、`:637-639`、kill 场景 `:198-200`）：

```c
command_service_run_command(m_mysql_session, COM_QUERY, &data,
                            mysqld::get_default_charset(),
                            deleg.callbacks(), deleg.representation(), &deleg);
```

结果经 `Command_delegate`（`ngs/command_delegate.h`）的回调转成 X Protocol 的 resultset 消息——**插件不碰服务器的结果集结构，只接回调**，这是服务化带来的解耦。

与经典连接的差异：X 连接**不占用 3306 的连接槽位**，但 `mysqlx_max_connections` 的注释明说"Actual number of connections is also affected by the general max_connections"——因为最终都要落到 THD。权限、事务、锁、审计则与普通连接完全一致（毕竟用的是同一个 THD）。

### DevAPI → SQL 的翻译层（核心）

**集合的 DDL**（`admin_cmd_collection_handler.cc:101-113`，原文拼装）：

```sql
CREATE TABLE `people` (
  doc JSON,
  _id VARBINARY(32) GENERATED ALWAYS AS
      (JSON_UNQUOTE(JSON_EXTRACT(doc, '$._id'))) STORED PRIMARY KEY,
  _json_schema JSON GENERATED ALWAYS AS ('<schema>'),
  CONSTRAINT `...` CHECK (JSON_SCHEMA_VALID(_json_schema, doc)) [NOT] ENFORCED
) CHARSET utf8mb4 ENGINE=InnoDB;
```

**文档字段索引** = 生成列 + 二级索引（`index_field.cc:121-128` + `:199`）：

```sql
ALTER TABLE people ADD COLUMN ageIdx INT
  GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(doc, '$.age')))
```

**表达式翻译**（`expr_generator.cc`）：X Protocol 传来的 `Mysqlx::Expr::Expr`（标识符 + 文档路径 / 运算符 / 函数调用）被递归翻译成 SQL 片段，`$.age` 这类文档路径统一落成 `JSON_UNQUOTE(JSON_EXTRACT(doc, '$.age'))`（`:358-364` 的 `generate_unquote_param` 负责在比较运算时套 `JSON_UNQUOTE`，否则留 `JSON_EXTRACT` 以利用索引）。

**CRUD**：`insert_statement_builder.cc` / `update_statement_builder.cc` / `find_statement_builder.h` / delete 各自拼 SQL；`upsert`（相同 `_id` 再 add）落成 `INSERT ... ON DUPLICATE KEY UPDATE doc = ...`（`insert_statement_builder.cc:135-141`）。

**文档 `_id` 生成**：`mysqlx_generate_document_id` UDF + `@@mysqlx_document_id_unique_prefix`（`document_id_aggregator.cc:43-47`）——多实例部署靠 prefix 区分，避免 `_id` 冲突。

**会话级能力**：`mysqlx_expect`（expect 条件块，如"期望某行受影响"，失败则整块失败）、`mysqlx_prepare`（预处理语句）、`mysqlx_notice`（warning / 会话状态 / GR 状态推送）——这些是经典协议没有、X Protocol 独有的能力，也是"为什么要另起协议"的直接答案。

### 认证与安全

三种 SASL 机制（`auth_plain.h:56` 返回 `"PLAIN"`；`auth_challenge_response.h:44-58` 定义 `Sasl_mysql41_auth` / `Sasl_sha256_memory_auth`，注释写明 MYSQL41 用 SHA1、SHA256_MEMORY 用 SHA256 的 challenge-response）。`SHA256_MEMORY` 依赖**密码哈希缓存**（`m_sha256_password_cache`），缓存的失效就落在前面那个 `mysqlx_cache_cleaner` 审计插件上——**这就是两个插件必须同生共死的真实原因**。

SSL 相关变量（`mysqlx_ssl_key/ca/capath/cert/cipher/crl/crlpath`）都是 **READONLY**（`system_variables.cc:161-191`），改了要重启。`mysqlx_port` 同样是 READONLY（`:117`）。

---

## 系统变量

（全部是**插件变量**，不在 `sql/sys_vars.cc`；默认值来自 `plugin/x/src/variables/system_variables_defaults.h`）

| 变量 | 默认 | 说明 |
|---|---|---|
| `mysqlx_port` | **33060**（`MYSQLX_TCP_PORT`，`plugin/x/variables.cmake:36`） | **READONLY**，改端口要重启 |
| `mysqlx_socket` | `MYSQLX_UNIX_ADDR` | Unix socket |
| `mysqlx_bind_address` | `*`（所有地址） | |
| `mysqlx_max_connections` | **100** | 注释明说仍受全局 `max_connections` 影响 |
| `mysqlx_min_worker_threads` | **2**（1..100） | |
| `mysqlx_idle_worker_thread_timeout` | **60** 秒（0..3600） | 空闲 worker 回收 |
| `mysqlx_max_allowed_packet` | **64 MB**（512B..1GB） | |
| `mysqlx_connect_timeout` | 见默认值常量 | 建连超时 |
| `mysqlx_port_open_timeout` | **0** | 等端口可用 |
| `mysqlx_wait_timeout` / `read_timeout` / `write_timeout` / `interactive_timeout` | — | 部分为 **THDVAR**（会话级） |
| `mysqlx_enable_hello_notice` | **true** | 连接时推送 hello notice |
| `mysqlx_document_id_unique_prefix` | — | 多实例 `_id` 前缀 |
| `mysqlx_compression_algorithms` / `mysqlx_deflate*_*`/`mysqlx_lz4*_*`/`mysqlx_zstd*_*` | — | 压缩与压缩级别（有范围校验） |
| `mysqlx_ssl_*` | NULL | **READONLY** |

状态变量以 `Mysqlx_` 前缀注册（`Module_mysqlx::get_status_variables()`），覆盖会话/连接/工作线程/收发字节/CRUD 计数/expect/notice 等维度，并可通过 `mysqlx_maintenance` 服务的 `reset_global_status_variables` 重置。

---

## Misc

### 能力边界（源码里的"不支持"）

| 限制 | 出处 |
|---|---|
| `Mysqlx::Expr::Expr::VARIABLE` 尚不支持 | `expr_generator.cc:60`（"is not supported yet"） |
| unique spatial / unique fulltext 索引不支持 | `admin_cmd_index.cc:188`、`:191` |
| 预处理语句的部分参数绑定类型不支持 | `prepare_param_handler.cc:145`（`ER_X_PREPARED_EXECUTE_ARGUMENT_NOT_SUPPORTED`） |
| 认证方式不被支持时 | `ER_NOT_SUPPORTED_AUTH_MODE`（`session.cc:160`） |
| 部分 capability 不可运行时设置 | "CapabilitiesSet not supported for the %s capability"（`capabilities/handler*.h`） |

### 协议辨析（务必记住）

- **X Plugin 不是 MongoDB 兼容层**：线协议是 protobuf 而非 BSON，认证是 PLAIN/MYSQL41/SHA256_MEMORY 而非 SCRAM，MongoDB 驱动连不上。它只是把**开发体验**做成文档风格。
- 判据：真正的协议兼容 = 现有 MongoDB 驱动不改代码即可连接（DocumentDB / FerretDB 那类）。

### 与经典连接的对照

| | 3306 经典连接 | 33060 X 连接 |
|---|---|---|
| 协议 | 文本协议 + 固定命令集 | protobuf（X Protocol） |
| 会话宿主 | 服务器自有连接线程 | 插件 worker 线程 + `srv_session` |
| 连接限额 | `max_connections` | `mysqlx_max_connections`（仍受全局 `max_connections` 影响） |
| 权限/事务/锁/审计 | — | **完全一致**（共用 THD） |
| 独有 | — | 集合/文档 API、notice、expect、预处理块 |

### 一句话总结

X Plugin 是"**在 MySQL 上另开一个 protobuf 协议端点，把文档式 API 翻译成 SQL，再用 `srv_session` + `command` 服务借来一个 THD 执行**"——它借的是 MongoDB 的**体验**，不是它的协议，更不是它的存储。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Using MySQL as a Document Store*
- *MySQL 8.0 Reference Manual → X Plugin*（变量与状态变量清单）

**相关文档**
- 插件框架与双插件声明：[`plugin.md`](plugin.md)
- 服务、registry 与插件→组件桥：[`service.md`](service.md)
- ABI 契约（为何走服务而非直呼内部符号）：[`abi.md`](abi.md)
- 组件基础设施：[`component.md`](component.md)
