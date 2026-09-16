# 组件（Component）基础设施深度解析

> 基于 MySQL 8.0.39 源码，涵盖组件声明宏与 ABI、minimal chassis、registry/dynamic loader 分工、`INSTALL/UNINSTALL COMPONENT` 与 `mysql.component` 持久化、启动引导（manifest）与内置组件盘点。
>
> **边界**：本篇讲组件这个"代码容器"及其加载/卸载/持久化；服务契约本身（定义/实现/消费宏、registry 内部数据结构、default 实现）见 [`service.md`](service.md)；插件体系见 [`plugin.md`](plugin.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量](#相关的系统变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

组件（Component）是 8.0 引入的**第二代扩展容器**（WL#4102）：一个动态库（或服务器二进制内的一段代码）承载一个 `mysql_component_t`，显式声明"我提供哪些服务实现（provides）"与"我需要哪些服务（requires）"，由 dynamic loader 按 URN 加载、由 registry 完成服务的注册与解析。组件之间**只通过服务互相调用**，不链接 mysqld、不直呼服务器符号。

### 用途

解决插件体系的四个架构缺陷（见理论基础），并已在 8.0.39 承载真实功能：keyring 组件（`component_keyring_file`）、`validate_password` 组件、PFS 插桩服务（`psi_*` / `pfs_plugin_*`）、日志服务（`log_builtins`）、UDF 注册服务（`udf_registration`）等。服务器本体也以组件形态存在——`mysql_server` 组件（URN `mysql:core`）一次性提供 146 个服务实现。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0 | 组件基础设施随 8.0 落地（WL#4102：M1 注册表+动态加载器、M2 内置组件、M3 `INSTALL/UNINSTALL COMPONENT`、M4 持久化） |
| 8.0.17 | `clone_protocol` 服务与 `audit_api_message_emit` 组件 |
| 8.0.23 | `query_attributes` 组件、`component_keyring_encrypted_file`（企业版） |
| 8.0.24 | **keyring 组件**（`component_keyring_file`）发布；keyring 插件开始弃用 |
| 8.0.26 | `reference_cache` 组件、`pfs_component` |
| 8.0.28+ | `mysqlbackup` 组件、telemetry 测试组件等持续扩充 |
| 8.4 / 9.x | keyring 插件在 8.4 弃用、9.x 移除——"插件 → 组件"迁移完成关键一步 |

---

## 理论基础

### 设计思想与权衡

**1. 动机：插件体系的四个架构缺陷。**

`components/libminchassis/dynamic_loader.cc` 开篇明写（原文照录）：

> 1. Plugins can only "talk" to the server and not with other plugins
> 2. Plugins have access to the server symbols and can call them directly, i.e. no encapsulation
> 3. There's no explicit set of dependencies of a plugin, thus it's hard to initialize them properly
> 4. Plugins require a running server to operate.

组件体系逐一回应：任意组件间的服务互调（registry 是"能力市场"而非单向契约）；禁止链接 mysqld、只消费服务（封装）；`BEGIN_COMPONENT_REQUIRES` 显式依赖清单 + 加载器做依赖解析（可批量、可循环）；minimal chassis（registry + dynamic loader）理论上**可嵌入任意二进制**。

**2. 服务市场与"覆盖（override）"：组件的真正新能力。**

组件通过**重新实现某个服务**即可覆盖/补充别的组件的功能。最直接的实例就在启动代码里：minimal chassis 自带 `dynamic_loader_scheme_file` 实现（允许从任意路径加载 .so），而服务器启动时 `registrator->set_default("dynamic_loader_scheme_file.mysql_server_path_filter")`——**用 mysql_server 组件的实现替换默认实现**，新实现强制 `file://` URN 解析到 `plugin_dir` 目录内（`check_and_make_absolute_urn()`），把"加载任意路径"收紧为"只能加载插件目录"。插件 vtable 是服务器写死的角色，做不到这种运行期替换。

**3. 显式依赖 + 批量加载：初始化顺序不再靠硬编码。**

插件时代"keyring 必须在 InnoDB 之前"这类顺序是写死在 `mysqld_main` 里的（`plugin_register_early_plugins` → builtin → dynamic）。组件把依赖写进 requires 列表，loader 在批量加载时做"集合内查找 + registry 查找"两层解析；**循环依赖**通过"同批加载"支持（unload 注释明写：`The dependencies may be circular, in such case it's necessary to specify all Components on cycle to unload in one batch`）。代价：加载/卸载都是"全批成败"（任一步失败 scope_guard 回滚），不适合做细粒度增量控制。

**4. 组件不链接 mysqld：ABI 边界的代价与收益。**

`component_implementation.h` 明说"强烈不鼓励组件链接 mysqld"。收益：组件只依赖头文件里稳定的 C ABI（函数指针表），服务器内部符号变化不破坏组件；组件可被非服务器宿主（如 `migrate_keyring` 客户端工具）复用——后者自己 `minimal_chassis_init()` 拉起一套 registry + dynamic loader 加载 keyring 组件。代价：所有能力（日志、分配内存、注册变量、访问 THD）都必须**先服务化**才能被组件使用——这正是 `mysql_string`、`log_builtins`、`component_sys_variable_*` 等服务存在的原因。

**5. URN/scheme：加载位置与方式解耦。**

组件名是 URN（如 `file://component_keyring_file`），scheme 部分（`file://`）决定由哪个 `dynamic_loader_scheme` 服务实现加载。scheme 本身可扩展、可覆盖（见上）。`file://` 映射到 `plugin_dir` 目录（`plugin_dir/<urn 文件名>.so`），**服务器没有独立的 `component_dir`**——刻意不引入第二个可调目录，插件与组件的 .so 共居一库，运维面最小。

**6. manifest：启动清单与持久化的分工。**

`INSTALL COMPONENT` 持久化到 `mysql.component` 表，启动时按表加载。但 keyring 这类**必须在 InnoDB 初始化之前**加载的组件等不及读表（表本身是 InnoDB 表）。于是引入 manifest 文件 `<datadir>/mysqld.my`（JSON）：`initialize_manifest_file_components()` 在启动早期读它加载组件。官方文档明确建议 keyring 组件**只用 manifest 加载、不用 `INSTALL COMPONENT`**——后者落 `mysql.component` 表，加载时机太晚。这是"两级持久化"：manifest 管"开机即要"，表管"运行期安装"。

### 理论溯源

- **服务定位器 + 依赖注入**：requires 列表 + loader 回填占位符，本质是编译期静态 DI（占位符指针注入），与 Spring/Guice 的运行时 DI 同构，区别是 MySQL 用宏把"注入点"显式化。
- **事务式加载**：load/unload 都是"阶段链 + scope_guard 回滚"，类比软件事务内存（STM）的 commit/rollback 结构，保证任一步失败系统状态不半残。
- **拓扑排序卸载**：`unload_do_topological_order` 按依赖逆序卸载，标准拓扑排序应用（允许同批循环）。
- **capability/security**：URN + path filter 是"白名单目录"式的沙箱思路——与插件体系 `check_valid_path()`（禁止路径分隔符）一脉相承。

### 算法与数据结构

- `mysql_component_t`（provides/requires/metadata/init/deinit）+ `list_components()` 导出函数（`COMPONENT_ENTRY_FUNC`）。
- dynamic loader 内部：`components_list`（URN → `mysql_component*` 的 map）、`urns_with_gen_list`（代际链表，unload 时定位依赖）。
- registry 内部（详见 03）：`service_registry`（名字 → 实现对象的 map）+ `interface_mapping`（句柄 → 实现对象）+ 原子引用计数。

---

## 核心实现

### 声明一个组件：宏链

```c
/* include/mysql/components/component_implementation.h */
#define DECLARE_COMPONENT(source_name, name)                    \
  mysql_component_t mysql_component_##source_name = {           \
      name, __##source_name##_provides, __##source_name##_requires, \
      __##source_name##_metadata,

#define DECLARE_LIBRARY_COMPONENTS \
  mysql_component_t *library_components_list = {
#define END_DECLARE_LIBRARY_COMPONENTS    \
  } ;                                     \
  DLL_EXPORT mysql_component_t *list_components() { \
    return library_components_list;        \
  }

#define BEGIN_COMPONENT_REQUIRES(name)                          \
  REQUIRES_SERVICE_PLACEHOLDER(registry);                       \
  static struct mysql_service_placeholder_ref_t __##name##_requires[] = { \
      REQUIRES_SERVICE(registry),
```

要点：

1. **`registry` 是每个组件的默认第 0 个依赖**——`BEGIN_COMPONENT_REQUIRES` 自动插入（不想依赖 registry 的底层组件用 `BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY`，如 minimal chassis 自己）。
2. `mysql_component_t`（定义在 `include/mysql/components/services/dynamic_loader.h`）：`name`（URN 名）、`provides`（`mysql_service_ref_t` 数组）、`requires_service`（`mysql_service_placeholder_ref_t` 数组：名字 + 回填指针）、`metadata`（key-value）、`init/deinit`。
3. `DECLARE_LIBRARY_COMPONENTS` 生成导出函数 `list_components()`，dynamic loader 用它拿到库内组件指针——当前实现**一个库只支持一个组件**。

### 主链路：`INSTALL COMPONENT`

```
dispatch_command → mysql_execute_command
 └─ Sql_cmd_install_component::execute()                  sql/sql_component.cc
      ├─ my_service<persistent_dynamic_loader>("persistent_dynamic_loader", srv_registry)
      ├─ acquire_shared_backup_lock
      ├─ 解析 INSTALL COMPONENT ... SET [GLOBAL|PERSIST] var 子句
      ├─ persisted_loader->load(thd, urns, count)
      │    └─ mysql_persistent_dynamic_loader_imp::load()  sql/server_component/persistent_dynamic_loader.cc
      │         ├─ open_component_table(TL_WRITE, INSERT_ACL)   ← 权限：对 mysql.component 的 INSERT
      │         ├─ dynamic_loader_srv->load(urns, count)        ← 真正加载
      │         └─ 往 mysql.component 插行（component_urn 等）→ 提交事务
      └─ end_transaction(...)
```

### dynamic loader 的七阶段加载链

`mysql_dynamic_loader_imp::load()` 持 `LOCK_dynamic_loader` 写锁，执行阶段链（每阶段负责调用下一阶段 + 失败时回滚自己已做的变更，scope_guard 机制）：

```
load_do_load_component_by_scheme      按 URN 的 scheme 找 scheme 服务，dlopen 出 mysql_component_t
load_do_collect_services_provided     收集本批组件的全部服务名（含同批互供）
load_do_check_dependencies            每个 require：先查"本批提供集合"，再查 registry；都无 → ER_COMPONENTS_CANT_SATISFY_DEPENDENCY
load_do_register_services             把 provides 逐个 register_service（含 "服务名.组件名" 与首个默认项）
load_do_resolve_dependencies          逐个 acquire 依赖并把句柄写入占位符指针（回填）
load_do_initialize_components         调 mysql_component_t::init()
load_do_commit                        移入 components_list 主集合，标记不可回滚
```

依赖回填的关键代码：

```c
bool mysql_dynamic_loader_imp::load_do_resolve_dependencies(...) {
  for (mysql_service_placeholder_ref_t *implementation_it :
       loaded_component->get_required_services()) {
    if (mysql_registry_imp::acquire(
            implementation_it->name,
            reinterpret_cast<my_h_service *>(implementation_it->implementation)))
      { ... ER_COMPONENTS_CANT_ACQUIRE_SERVICE_IMPLEMENTATION ... }
  }
}
```

`REQUIRES_SERVICE(foo)` 在编译期把 `{ "foo", (void**)&mysql_service_foo }` 填进 requires 数组——loader 在这里 acquire 后**直接写组件内的全局指针** `mysql_service_foo`。这就是占位符注入的全部机制。

卸载链更长（10 阶段）：`unload_do_list_components → unload_do_topological_order → unload_do_get_scheme_services → unload_do_lock_provided_services → unload_do_check_provided_services_reference_count → unload_do_deinitialize_components → unload_do_unload_dependencies → unload_do_unregister_services → unload_do_unload_components → unload_do_commit`。

其中**引用计数检查**（`unload_do_check_provided_services_reference_count`）是卸载安全的核心：若组件提供的服务实现引用计数 > 0，且引用者不是"同批卸载组"内的组件，则报 `ER_COMPONENTS_UNLOAD_CANT_UNREGISTER_SERVICE` **拒绝卸载**——"有人还拿着服务句柄，组件就卸不掉"，与插件的"延迟收割"形成对照：组件选择**阻塞卸载**，插件选择**标记删除后异步收割**。

### 持久化：`mysql.component` 表

```sql
CREATE TABLE IF NOT EXISTS component (
  component_id       int unsigned NOT NULL AUTO_INCREMENT,
  component_group_id int unsigned NOT NULL,
  component_urn     text NOT NULL,
  PRIMARY KEY (component_id)
) engine=INNODB DEFAULT CHARSET=utf8mb3 COMMENT 'Components' ...
```

与 `mysql.plugin` 一样是**普通 InnoDB 系统表**（不是 DD 对象）。`persistent_dynamic_loader`（`sql/server_component/persistent_dynamic_loader.cc`）是这张表的守护者：

- 启动时 `persistent_dynamic_loader_init()` 校验表结构（`Component_db_intact`）并读表缓存；
- `INSTALL` 时插行（INSERT_ACL 权限检查）、`UNINSTALL` 时删行（DELETE_ACL）——帮助文本原文确认："INSTALL COMPONENT requires the INSERT privilege for the mysql.component system table because it adds a row to that table"；
- `INSTALL COMPONENT ... SET PERSIST` 的变量持久化也由它牵头（配合 `Set_variables_helper` 写 `mysqld-auto.cnf`）。

### 启动引导：三条链

```
mysqld_main
 ├─ init_server_components()
 │    ├─ component_infrastructure_init()      ← 自举（见下）
 │    └─ initialize_manifest_file_components() ← 读 <datadir>/mysqld.my，加载 manifest 组件
 │
 └─ （DD 完全就绪后）
      mysql_component_infrastructure_init()    ← 读 mysql.component 表，加载持久化组件
```

自举 `component_infrastructure_init()` 的完整顺序（这是理解"分层"的关键）：

1. `minimal_chassis_init(&srv_registry, &COMPONENT_REF(mysql_server))`：初始化 registry + dynamic loader（两者来自 `components/libminchassis/`），注册 `mysql_minimal_chassis` 组件，再把 **`mysql_server` 组件**（`DECLARE_COMPONENT(mysql_server, "mysql:core")`，146 个服务实现）注册进去——这一步等于把"整个服务器"作为组件挂进注册表；
2. `srv_registry->acquire("dynamic_loader_scheme_file.mysql_minimal_chassis", ...)`：拿底盘自带的 file scheme；
3. `registrator->set_default("dynamic_loader_scheme_file.mysql_server_path_filter")`：**覆盖默认 file scheme** 为服务器的 path filter 实现（`file://` → `plugin_dir/xxx.so`）；
4. 同理覆盖 `mysql_rwlock_v1`、`mysql_psi_system_v1`、`mysql_runtime_error` 的默认实现（从 minimal chassis 实现换成 mysql_server 实现——带 PSI 插桩、带 server 错误码）。

### 内置组件盘点（8.0.39 源码树，逐个核实）

| 组件（`DECLARE_COMPONENT` 名） | 位置 | 提供的服务（provides） | 依赖的服务（requires，摘） |
|---|---|---|---|
| `mysql_server`（URN `mysql:core`） | `sql/server_component/server_component.cc` | **146 个实现**：`log_builtins*`、`udf_registration*`、`mysql_string_*`(15)、`component_sys_variable_*`、`table_access_*`、`mysql_command_*`、`keyring_*`（iterator/lockable）、`mysql_thd_*`、`psi_*`(39) 等 | 仅 registry（空 requires） |
| `mysql_minimal_chassis` | `components/libminchassis/minimal_chassis.cc` | `registry`、`registry_registration`、`registry_query`、`registry_metadata_*`、`dynamic_loader`、`dynamic_loader_query`、`dynamic_loader_metadata_*`、`dynamic_loader_scheme_file`、`mysql_runtime_error`、`mysql_rwlock_v1`、`mysql_psi_system_v1` | 无（`BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY`，自举） |
| `component_keyring_file` | `components/keyrings/keyring_file/keyring_file.cc` | `keyring_aes`、`keyring_generator`、`keyring_load`、`keyring_keys_metadata_iterator`、`keyring_component_status`、`keyring_component_metadata_query`、`keyring_reader_with_status`、`keyring_writer`、`log_builtins`、`log_builtins_string` | `registry`、`log_builtins`、`log_builtins_string` |
| `validate_password`（URN `mysql:validate_password`） | `components/validate_password/validate_password_imp.cc` | `validate_password`、`validate_password_changed_characters` | `log_builtins`、`log_builtins_string`、`mysql_string_*`(7)、`component_sys_variable_register/unregister`、`status_variable_registration`、`mysql_thd_security_context` |
| `reference_caching`（URN `mysql:reference_caching`） | `components/reference_cache/component.cc` | `reference_caching_channel`、`reference_caching_cache`、`reference_caching_channel_ignore_list` | — |
| `query_attributes`（URN `mysql:query_attributes`） | `components/query_attributes/query_attributes.cc` | UDF（`query_attribute` 等） | `psi_memory`、`udf_registration`、`mysql_query_attributes_iterator`、`mysql_query_attribute_string/isnull`、`mysql_string_converter/factory`、`mysql_udf_metadata` |
| `mysqlbackup`（URN `mysql:mysqlbackup`） | `components/mysqlbackup/mysqlbackup.cc` | 备份接口 | — |
| `audit_api_message_emit` | `components/audit_api_message_emit/audit_api_message_emit.cc` | 审计消息发送能力（供审计插件/组件用） | — |
| `log_filter_dragnet` / `log_sink_json` / `log_sink_syseventlog` / `log_sink_test` | `components/logging/` | `log_filter` / `log_service` 家族实现 | — |
| `pfs_example_component_population` | `components/pfs_component/pfs_example_component_population.cc` | PFS 插件表示例（**测试组件**） | — |
| `mysql:example_component1..3`、`mysql:test_*`（大量） | `components/example/`、`components/test/` | 教学/测试服务 | — |
| `test_server_telemetry_traces` | `components/test_server_telemetry_traces/` | telemetry 测试 | — |

补充事实（均经核实）：

- **不存在 `components/mysql_server/` 目录**——mysql_server 组件声明在 `sql/server_component/server_component.cc`；registry/dynamic loader 实现在 `components/libminchassis/`。
- `components/library_mysys/` **不含任何组件**（是给组件复用的静态工具库）。
- 真正的 PFS 插桩服务（`psi_*`/`pfs_plugin_*` 约 40 个）的实现在 `storage/perfschema/pfs.cc`，注册时实现名是 `performance_schema`（挂在 `mysql_server` 组件的 provides 里由 `minimal_chassis_init` 一次性注册）；`components/pfs_component/` 目录反而只有测试组件。
- `component_keyring_file` 自己提供一份 `log_builtins` / `log_builtins_string` 实现——真实原因是 keyring 组件还要被**独立工具**复用：`client/migrate_keyring/` 里 `minimal_chassis_init(&components_registry, nullptr)`（**不带 mysql_server 组件**），那种环境下没有任何人提供日志服务，keyring 组件必须自带一份才能工作。服务器环境下它只是以 `log_builtins.component_keyring_file` 名字与 `log_builtins.mysql_server` 并存，default 仍由"先注册者"（mysql_server）把持。

---

## 相关的系统变量

**服务器没有 `component_dir` 系统变量**。`file://` URN 的解析目标是 `plugin_dir` 目录（`dynamic_loader_path_filter.cc` 里 `my_string path = opt_plugin_dir;` 拼接 URN 文件名），与插件共用目录。`component_dir` 这个词只出现在独立工具（`client/migrate_keyring/`、`components/test/keyring_encryption_test/`）的选项里，不是服务器变量。

---

## Misc

### 组件 vs 插件（源码事实对照）

| 维度 | 组件 | 插件 |
|---|---|---|
| 单元 | `mysql_component_t`：一个库一个组件，多服务实现 | `st_mysql_plugin[]`：一个库可多个插件 |
| 契约 | provides/requires 双向显式 | 单向类型 vtable |
| 依赖 | 显式清单，loader 解析（支持循环） | 无声明，顺序靠启动代码硬编码 |
| 与服务器关系 | 不链接 mysqld，只消费服务 | 可直呼服务器全局符号 |
| 注册 | 运行期 register_service + 引用计数 + default 机制 | 编译期类型表 + 名字哈希 |
| 卸载 | 引用未归零 → **拒绝卸载** | 标记删除 → ref 归零后**异步收割** |
| 持久化 | `mysql.component` + manifest（`mysqld.my`） | `mysql.plugin` |
| 标识 | URN（`file://`、`builtin://`…，scheme 可扩展） | 名字 + `--plugin-dir` |

### keyring：迁移时间线（源码证据）

`plugin/keyring/keyring.cc` 的 `keyring_init()` 里明写：

```cpp
logger->log(WARNING_LEVEL, ER_SERVER_WARN_DEPRECATED, "keyring_file plugin",
            "component_keyring_file");
```

即 keyring_file 插件每次启动 init 都打弃用警告并指向组件。官方路线图：8.0.24 引入 keyring 组件 → 8.4 弃用 keyring 插件 → 9.x 移除。8.0.39 处于"并存 + 引导迁移"阶段。

### 社区源码里没有 audit_log 插件

8.0.39 社区树中 `plugin/` 下只有 `audit_null`（审计测试插件），**没有** `audit_log_filter`（它是商业版插件，源码不在社区树）；`delayed_plugins[] = {"audit_log", "mysql_firewall"}` 里的名字即商业插件。审计接口本体（`st_mysql_audit`）在 8.0.39 仍是插件类型、未迁移组件服务。

---

## 参考

**官方文档**
- WorkLog: [WL#4102 Service registry and component infrastructure](https://dev.mysql.com/worklog/task/?id=4102)（设计动机、分层、M1~M4 里程碑）
- *MySQL 8.0 Reference Manual → Extending MySQL → The Component Service / Components*（`INSTALL COMPONENT`、manifest 用法）
- *MySQL 8.0 Reference Manual → Keyring Component Installation*（keyring 组件与 manifest 加载）
- MySQL Server Doxygen: [Component Infrastructure Layers](https://dev.mysql.com/doc/dev/mysql-server/latest/page_components_layering.html)

**源码文档页**
- `components/libminchassis/dynamic_loader.cc`：`@page PAGE_COMPONENTS`、`PAGE_COMPONENTS_CONCEPTS`、`PAGE_COMPONENTS_DYNAMIC_LOADER`、`page_components_layering`
- `components/libminchassis/registry.cc`：`@page PAGE_COMPONENTS_REGISTRY`

**相关文档**
- 服务契约与 registry 内部实现见 [`service.md`](service.md)
- 插件体系见 [`plugin.md`](plugin.md)
- ABI 契约专题见 [`abi.md`](abi.md)
