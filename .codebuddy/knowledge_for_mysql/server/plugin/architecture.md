# MySQL 可扩展架构总览（插件 vs 组件）

> 基于 MySQL 8.0.39 源码。本篇回答一个宏观问题：**MySQL 到底是不是从插件架构转向了组件架构？** 以及两代机制如何共存、迁移到底进行到了哪一步、为什么不是一刀切替换。
>
> **边界**：本篇是**架构层总览**（面），不重复各篇的机制细节（点）——插件机制见 [`plugin.md`](plugin.md)，组件机制见 [`component.md`](component.md)，服务见 [`service.md`](service.md)，服务注册表（组件中枢）见 [`registry.md`](registry.md)，ABI 见 [`abi.md`](abi.md)；具体插件见 [`clone.md`](clone.md)、[`mysqlx.md`](mysqlx.md)、[`keyring.md`](keyring.md)（桥接式迁移）、[`validate_password.md`](validate_password.md)（★ 分派式迁移，唯一同功能双实现并存样本）。认证插件的凭据校验与握手协议另见 [`../auth/authentication.md`](../auth/authentication.md)。

## 目录

- [概述：一句话结论](#概述一句话结论)
- [架构分层：一张图看清全貌](#架构分层一张图看清全貌)
- [两代机制的能力矩阵](#两代机制的能力矩阵)
- [迁移到底进行到哪一步（源码数据）](#迁移到底进行到哪一步源码数据)
- [为什么不是"一刀切替换"](#为什么不是一刀切替换)
- [端到端：一次密钥读取请求的完整路径](#端到端一次密钥读取请求的完整路径)
- [跨版本演进时间线](#跨版本演进时间线)
- [选型：新扩展该用插件还是组件](#选型新扩展该用插件还是组件)
- [他库对比](#他库对比)
- [判据：如何判断一个功能处于哪个阶段](#判据如何判断一个功能处于哪个阶段)
- [两套机制的初始化链（源码对照）](#两套机制的初始化链源码对照)
- [★★ 真名勘误全表](#-真名勘误全表)
- [Misc](#misc)
- [参考](#参考)

---

## 概述：一句话结论

**MySQL 并没有从插件架构"转向"组件架构——它是在插件体系之上叠加了一层组件/服务体系，形成"双轨制"：插件仍是主力载体（存储引擎、认证、审计、clone、mysqlx、GR…全都是插件），组件负责"新能力"与"少数功能的替换式迁移"，而"服务"是两代之间的通用货币。**

三个可以直接用源码验证的事实支撑这个判断：

1. **存储引擎从未组件化，也不太可能组件化**：`MYSQL_STORAGE_ENGINE_PLUGIN` 仍是唯一的引擎接入方式（InnoDB、binlog 都是插件），组件侧没有任何"引擎"抽象。
2. **认证、审计、clone、mysqlx、GR、半同步、全文解析器、I_S 表——全部只有插件形态**（8.0.39 组件目录下没有它们的对应物）。
3. **真正完成"替换式迁移"的只有极少数功能**：keyring、validate_password 是主要例子（两者都是"插件版与组件版同时编译，官方引导用组件"）。

所以更准确的描述是：

```
旧世界（插件）：服务器在固定调用点调外部代码   →  仍是主力
新世界（组件）：外部代码之间按名字互相提供能力 →  新增层
连接两代的货币：服务（service）                →  插件经桥也能消费
```

---

## 架构分层：一张图看清全貌

```
┌──────────────────────────────────────────────────────────────────┐
│ 消费者：InnoDB、binlog 加密、clone、mysqlx、PFS、第三方插件/组件     │
└───────────────────────────┬──────────────────────────────────────┘
                            │ acquire("服务名")
┌───────────────────────────▼──────────────────────────────────────┐
│ 服务市场（registry）                                              │
│   174 个服务 × 多实现；default 机制 + 引用计数 + acquire_related    │
└───────────────────────────┬──────────────────────────────────────┘
        ┌────────────────────┴────────────────────┐
        │                                          │
┌───────▼──────────┐                    ┌─────────▼────────────────┐
│ 组件（component） │                    │ 服务器代理 / 内置插件       │
│ 自供服务          │                    │ daemon_keyring_proxy 等   │
│ keyring、validate │                    │ 把插件 vtable 包成服务     │
│ _password、日志…  │                    └─────────┬────────────────┘
└───────┬──────────┘                              │ plugin_foreach
        │ dynamic loader                 ┌─────────▼────────────────┐
        │ （URN/scheme + 依赖解析）        │ 插件（plugin）             │
        │                                │ 12 种类型；引擎/认证/审计…  │
        │                                └──────────────────────────┘
┌───────▼──────────────────────────────────────────────────────────┐
│ minimal chassis：registry + dynamic loader（可脱离服务器自举）       │
└──────────────────────────────────────────────────────────────────┘
```

关键观察：

- **组件并不是"插件的上位替代"，而是"服务的另一个提供者"**。消费者只认服务，不关心提供者是组件还是（被代理的）插件。
- **`daemon_keyring_proxy` 这类代理插件是双轨制的胶水**：因为插件无法自己提供服务（详见 [`keyring.md`](keyring.md)），服务器必须出面把插件的 vtable 翻译成服务。
- **minimal chassis 在最底层**，且能脱离服务器自举（`mysql_migrate_keyring` 就靠它加载组件）。

---

## 两代机制的能力矩阵

| 维度 | 插件 | 组件 |
|---|---|---|
| 契约形态 | 类型化 vtable（"我是什么角色"） | provides/requires（"我提供/需要什么能力"） |
| 谁调谁 | 服务器 → 插件（**单向**） | 任意组件 ↔ 任意组件（经 registry） |
| 能对外提供服务吗 | **不能**（需服务器代理） | **能** |
| 依赖表达 | 无（顺序硬编码三段式） | 显式 requires + 自动拓扑排序 |
| 运行期替换实现 | 不能（slot 写死） | 能（`set_default` override） |
| 生命周期 | 引用计数 + 标记删除延迟收割 | 引用计数 + 未归零拒绝卸载 |
| 持久化 | `mysql.plugin` 表 | `mysql.component` 表 + manifest |
| 加载时序 | 三段硬编码（early/builtin/dynamic） | 依赖驱动（可批量、可循环） |
| 与服务器耦合 | 可直呼服务器全部符号（`--export-dynamic`） | 不链接 mysqld，只消费服务 |
| 典型承载 | **存储引擎、认证、审计、clone、mysqlx、GR** | keyring、validate_password、日志、GR listener、reference_cache、PFS 插桩服务 |

（每格的详细源码证据分别见 [`plugin.md`](plugin.md) 与 [`component.md`](component.md)。）

---

## 迁移到底进行到哪一步（源码数据）

| 指标 | 数值 | 说明 |
|---|---|---|
| 插件类型总数 | **12**（`MYSQL_UDF_PLUGIN` … `MYSQL_CLONE_PLUGIN`） | 集合封闭，第三方无法新增类型 |
| 组件服务定义数 | **174** 个 `BEGIN_SERVICE_DEFINITION` | 能力面已远超插件类型 |
| `mysql_server` 组件提供的服务实现 | **146** | 服务器本体就是最大的服务提供者 |
| `plugin/` 目录下的插件 | **23 个目录**（含示例与测试插件） | 引擎、认证、clone、mysqlx、GR、semisync… |
| `components/` 目录下的组件 | 14 个目录 | keyring、validate_password、日志、reference_cache、mysqlbackup、query_attributes、PFS… |
| 老版插件服务 | 17 条 `list_of_services` | 与新服务差一个数量级 |

**按"功能"看迁移阶段**（8.0.39 实测）：

| 阶段 | 功能 | 判据 |
|---|---|---|
| **只有插件** | 存储引擎、认证、审计、clone、mysqlx、GR、半同步、rewriter、全文解析器、I_S 表 | 组件目录下无对应物 |

⚠️ **一处精确化**：8.0.39 确实存在 `mysql_authentication_registration` 服务（`include/mysql/components/services/`），但它的使用者是**服务器自己的 MFA 代码**（`sql/auth/sql_mfa.cc`），**没有任何组件提供或消费它**——所以"认证仍是插件"这个判断不变，但不能说"服务化完全没触及认证领域"。
| **双轨并存（官方引导迁组件）** | **keyring**、**validate_password** | 两者都编译两份；keyring 插件 init 打 `ER_SERVER_WARN_DEPRECATED`；`mysql_secure_installation` 默认装组件 |
| **只有组件** | 日志（`log_filter_dragnet` / `log_sink_json` 等）、`reference_cache`、`query_attributes`、`mysqlbackup`、`audit_api_message_emit` | 插件目录下无对应物 |

**结论**：真正"替换式迁移"的只有中间那一类，且数量很少；**大多数存量功能仍是纯插件**。官方路线明确会在 8.4 弃用、9.x 移除的是 **keyring 插件**这一具体功能，而不是插件体系本身。

**★ 逐项的源码证据**（★ 判断"是插件还是组件"的唯一硬标准是 `mysql_declare_plugin()` vs `PROVIDES_SERVICE`）：

```cpp
/* ① keyring 插件仍在 —— plugin/keyring/keyring.cc:252 */
mysql_declare_plugin(keyring_file){
    MYSQL_KEYRING_PLUGIN, /*   type                            */
    &keyring_descriptor,  /*   descriptor                      */
    "keyring_file",       /*   name                            */
    PLUGIN_AUTHOR_ORACLE, /*   author                          */
    ... 0x0100 ...
```

```cpp
/* ② keyring 组件 —— components/keyrings/keyring_file/keyring_file.cc:271 */
BEGIN_COMPONENT_PROVIDES(component_keyring_file)
PROVIDES_SERVICE(component_keyring_file, keyring_aes),
    PROVIDES_SERVICE(component_keyring_file, keyring_generator),
    PROVIDES_SERVICE(component_keyring_file, keyring_load),
    /* ... keyring_keys_metadata_iterator / keyring_component_metadata_query /
           keyring_reader_with_status / keyring_writer / keyring_component_status ... */
    END_COMPONENT_PROVIDES();

BEGIN_COMPONENT_REQUIRES(component_keyring_file)
REQUIRES_SERVICE(registry), REQUIRES_SERVICE(log_builtins),
    REQUIRES_SERVICE(log_builtins_string), END_COMPONENT_REQUIRES();
...
DECLARE_COMPONENT(component_keyring_file, "component_keyring_file")
keyring_file::keyring_file_init,
    keyring_file::keyring_file_deinit END_DECLARE_COMPONENT();

DECLARE_LIBRARY_COMPONENTS &COMPONENT_REF(component_keyring_file)
    END_DECLARE_LIBRARY_COMPONENTS
```

> ⚠️ **同名陷阱**：插件叫 `keyring_file`，组件的 CMake target **也叫 `keyring_file`**（`components/keyrings/keyring_file/CMakeLists.txt:79`），但 `DECLARE_COMPONENT` 的名字是 `component_keyring_file`。所以 `INSTALL PLUGIN keyring_file` 与 `INSTALL COMPONENT 'file://component_keyring_file'` 是**两个不同的东西**。

```cpp
/* ③ validate_password 插件已声明废弃 —— plugin/password_validation/validate_password.cc:474 */
static int validate_password_init(MYSQL_PLUGIN plugin_info) {
  push_deprecated_warn(thd_get_current_thd(), "validate password plugin",
                       "validate_password component");   /* ★★ 官方废弃声明 */
  ...
```

```cpp
/* ④ mysqlx 仍是插件（一个 .so 里两条声明）—— plugin/x/src/xpl_plugin.cc:35 */
mysql_declare_plugin(mysqlx){
    MYSQL_AUDIT_PLUGIN,
    modules::Module_cache::get_audit_plugin_descriptor(),
    "mysqlx_cache_cleaner",
    ...
},
    {
        MYSQL_DAEMON_PLUGIN,
        &mysqlx_deamon_plugin_descriptor,
        MYSQLX_PLUGIN_NAME,           /* "mysqlx" */
        ...
    } mysql_declare_plugin_end;
```

```cpp
/* ⑤ clone 仍是插件 —— plugin/clone/src/clone_plugin.cc:651 */
mysql_declare_plugin(clone_plugin){
    MYSQL_CLONE_PLUGIN,
    &clone_descriptor,
    clone_plugin_name, /* Plugin name */
    ...
    plugin_clone_init,   /* Plugin Init */
    plugin_clone_check,  /* Plugin check uninstall */
    plugin_clone_deinit, /* Plugin Deinit */
    CLONE_PLUGIN_VERSION,
    nullptr,                /* status variables */
    clone_system_variables, /* system variables */
    ...
} mysql_declare_plugin_end;
```

**★★★ `daemon_keyring_proxy_plugin` 是双轨制的胶水**（★ 它**是插件**，但对外**提供组件服务**——这就是"插件不能提供服务"这条规则的唯一例外，因为它是服务器自己写的）：

```cpp
/* sql/daemon_proxy_keyring/daemon_proxy_keyring.cc:44 —— 官方注释原文 */
/**
  @page PAGE_COMPONENT_DAEMON_KEYRING_PROXY component_daemon_keyring_proxy

  This plugin implementation provides wrappers over keyring plugin
  implementation and exposes functionality provided by plugin through
  keyring component service APIs.      /* ★★ 明说：把插件能力包成组件服务 */

  Following keyring component services implemented:
  - keyring_generate
  - keyring_keys_metadata_iterator
  - keyring_component_status
  - keyring_reader
  - keyring_reload
  - keyring_writer
  ...
*/
```

```cpp
/* :709 —— 它注册的服务名带 .daemon_keyring_proxy 后缀 */
static const char *proxy_keyring_service_names[] = {
    "keyring_generator.daemon_keyring_proxy",
    "keyring_keys_metadata_iterator.daemon_keyring_proxy",
    "keyring_component_status.daemon_keyring_proxy",
    "keyring_reader_with_status.daemon_keyring_proxy",
    "keyring_writer.daemon_keyring_proxy"};

static int daemon_keyring_proxy_plugin_init(void *) {
  my_service<SERVICE_TYPE(registry_registration)> registrator(
      "registry_registration", srv_registry);   /* ★ 插件通过桥拿到 registry */
  for (unsigned int i = GENERATOR; i <= WRITER; ++i) {
    retval = registrator->register_service(proxy_keyring_service_names[i],
                                           proxy_keyring_service_handles[i]);
    if (retval == true) return 1;
  }
  /* In case no keyring component was loaded, following will register
     proxy keyring services as default. This would enable keyring plugin usage. */
  set_srv_keyring_implementation_as_default();
  return 0;
}

mysql_declare_plugin(daemon_keyring_proxy){
    MYSQL_DAEMON_PLUGIN,
    &daemon_keyring_proxy_plugin,
    "daemon_keyring_proxy_plugin",
    "Oracle",
    "A plugin that implements the keyring component "
    "services atop of the keyring plugin",     /* ★★ 自我描述 */
    PLUGIN_LICENSE_GPL,
    daemon_keyring_proxy_plugin_init, ...
    0,  /* Flags */
} mysql_declare_plugin_end;
```

> ★★ **它是强制内置插件**，与 binlog / mysql_native_password / caching_sha2_password 同级（`sql/sql_builtin.cc.in`）：
> ```c
> struct st_mysql_plugin *mysql_mandatory_plugins[]=
> {
>   builtin_binlog_plugin, builtin_mysql_password_plugin,
>   builtin_caching_sha2_password_plugin, builtin_daemon_keyring_proxy_plugin,
>   @mysql_mandatory_plugins@ 0
> };
> ```
> ★ `plugin_register_builtin_and_init_core_se()` 里还特判了它——**只有 `daemon_keyring_proxy` / MyISAM / InnoDB / CSV 这四个会在"核心阶段"就被初始化**，其余全部延后。

**★ `plugin/` 目录完整清单（23 项）**：`audit_null`、`auth`、`clone`、`connection_control`、`daemon_example`、`ddl_rewriter`、`fulltext`、`group_replication`、`innodb_memcached`、`keyring`、`keyring_udf`、`password_validation`、`pfs_table_plugin`、`replication_observers_example`、`rewrite_example`、`rewriter`、`semisync`、`test_plugins`、`test_service_sql_api`、`test_services`、`udf_services`、`version_token`、`x`。

**★ `components/` 目录完整清单（14 项）**：`audit_api_message_emit`、`example`、`keyrings/`、`libminchassis`、`library_mysys`、`logging`（4 个组件）、`mysqlbackup`、`pfs_component`、`pfs_example`、`query_attributes`、`reference_cache`、`test/`（28 个测试组件）、`test_server_telemetry_traces`、`validate_password`。

> ⚠️ **`libminchassis/` 与 `library_mysys/` 不是组件**，是基础设施静态库——但 `libminchassis/minimal_chassis.cc:129` 自己也 `DECLARE_COMPONENT(mysql_minimal_chassis, ...)`。
> ★ 全仓 `MYSQL_ADD_COMPONENT(` 共 **54 处**，`MYSQL_ADD_PLUGIN(` 的插件目录由 `CONFIGURE_PLUGINS()` 用 `FILE(GLOB dirs_plugin ${CMAKE_SOURCE_DIR}/plugin/*)` **自动发现**（`cmake/plugin.cmake:340`）。

---

## 为什么不是"一刀切替换"

四个约束叠加，决定了只能是渐进共存：

1. **调用点约束（技术）**：插件的价值恰恰在于"服务器在固定位置调外部代码"——认证、引擎这些**调用点是服务器内生的**，且是热路径。组件服务的每次调用都要过一次 registry 查找，对每秒百万次的引擎接口不可接受（见 [`abi.md` 设计思想 ⑦](abi.md) 的"调用形态三分层"）。
2. **类型封闭约束（技术）**：插件类型写死在头文件里（12 个），新增类型必须改服务器——这既是缺点，也让"已有类型"天然稳定，没有迁移动力。
3. **生态约束（商业）**：第三方存储引擎、认证插件、审计插件是存量资产，Oracle 不可能要求它们全部重写为组件。
4. **桥的存在（技术）**：`plugin_registry_service` 让**老插件也能消费组件服务**（见 [`service.md`](service.md)）——于是"把能力服务化"和"把插件组件化"可以解耦：能力可以先服务化（这是已经发生的大头，174 个服务），插件本身可以不着急动。

一句话：**迁移的真实形态是"能力下沉到服务层"，而不是"插件改写为组件"。** 服务的 174 个定义才是这场演进的主战场；组件只是服务的一个提供者。

---

## 端到端：一次密钥读取请求的完整路径

把前面所有概念串起来，看 InnoDB 读一个加密密钥时实际发生了什么（每一跳都对应一篇文档）：

```
① InnoDB（插件）: os0enc.cc
     acquire("keyring_reader_with_status")                      ← [service] 服务寻址
② registry:  service_registry.find("keyring_reader_with_status") ← [registry] 双 key 查默认
     命中 "keyring_reader_with_status.mysql_server"（首注册者即默认）
③ 该实现的真实来源: daemon_keyring_proxy（内置 daemon 插件）       ← [plugin] 代理层
     iterate_plugins(): plugin_foreach(MYSQL_KEYRING_PLUGIN)
       → plugin_lock（引用计数）
       → plugin_decl(plugin)->info 取 st_mysql_keyring*
       → mysql_key_fetch(...)                                    ← [abi] C 函数指针调用
       → plugin_unlock
④ 返回密钥；此后所有工作线程共用这个句柄（服务无状态，无需 per-thread acquire）

【若已迁到组件】装了 component_keyring_file 后：
   registry 里多一个 "*.component_keyring_file" 实现，但 default 仍是 mysql_server；
   要用组件实现 → 显式 set_default，或用 acquire_related 锁定同一实现者。
```

这条路径把六件事同时暴露出来：**插件不能互调**（所以 InnoDB 不能直接叫 keyring 插件）、**插件不能提供服务**（所以要代理）、**服务是寻址货币**（所以双方解耦）、**registry 决定默认**（所以"装了组件不自动生效"）、**ABI 是 C 函数指针**（所以跨编译器安全）、**引用计数保护卸载**（所以调用期间插件不会被卸）。

**★ 转发层的真实源码**（★ `mysql_server` 组件提供的 8 条 keyring 服务**不是真实现**，只是"加锁 + 转发"到真正的实现者）：

```cpp
/* sql/server_component/server_component.cc:433 —— 对外的服务表 */
BEGIN_SERVICE_IMPLEMENTATION(mysql_server, keyring_reader_with_status)
Keyring_reader_service_impl::init, Keyring_reader_service_impl::deinit,
    Keyring_reader_service_impl::fetch_length,
    Keyring_reader_service_impl::fetch END_SERVICE_IMPLEMENTATION();
```

```cpp
/* sql/server_component/mysql_server_keyring_lockable_imp.cc:462 —— 转发体 */
DEFINE_BOOL_METHOD(Keyring_reader_service_impl::init,
                   (const char *data_id, const char *auth_id,
                    my_h_keyring_reader_object *reader_object)) {
  if (check_service(internal_keyring_reader, READER_WITH_STATUS)) return true;

  rwlock_scoped_lock lock(&LOCK_keyring_component, false, __FILE__, __LINE__);
  /* ★★ server 侧的读写锁：并发控制就加在这一层 */
  my_h_keyring_reader_object_server *local_object =
      new my_h_keyring_reader_object_server{nullptr, std::move(lock)};
  if (local_object == nullptr) return true;

  bool retval =
      internal_keyring_reader->init(data_id, auth_id, &(local_object->object_));
  ...
}

DEFINE_BOOL_METHOD(Keyring_reader_service_impl::fetch,
                   (my_h_keyring_reader_object reader_object,
                    unsigned char *data_buffer, ...)) {
  if (check_service(internal_keyring_reader, READER_WITH_STATUS)) return true;
  my_h_keyring_reader_object_server *local_object =
      reinterpret_cast<my_h_keyring_reader_object_server *>(reader_object);
  if (!local_object) return true;

  return internal_keyring_reader->fetch(      /* ★★ 转发终点 */
      local_object->object_, data_buffer, data_buffer_length, data_size,
      data_type, data_type_buffer_length, data_type_size);
}
```

**★ `internal_keyring_reader` 是怎么拿到的**（★ `set_srv_keyring_implementation_as_default()`，注释原文讲清了两条路径）：

```cpp
/**
  server component provides implementation of all keyring
  related services to provide concurrency control.

  In turn it uses either one of the following as actual
  implementation(in order of priority):
  A> A keyring component loaded through manifest file
  B> Proxy keyring component over keyring plugin

  Path 1: 启动时 minimal chassis 初始化后读 manifest，若有 keyring 组件则加载并注册服务，
          此时每个 keyring 服务都有两份实现（server 的 + 组件的），
          set_srv_keyring_implementation_as_default() 会：
          1. 把 server 的 lockable 实现设为 default
          2. acquire 出组件提供的所有服务句柄

  Path 2: 若 manifest 没有或不含 keyring，则仍设 default 但句柄为空 → 功能不可用。
          随后处理 --early-plugin-load，再加载 daemon_proxy_keyring 插件，
          它注册一部分 keyring 组件服务（这些服务会用 keyring 插件）。
          此时再次调用本函数，若句柄还没设就设上 → 走 daemon_proxy_keyring。
*/
void set_srv_keyring_implementation_as_default() {
  my_service<SERVICE_TYPE(registry_registration)> registrator(
      "registry_registration", srv_registry);
  ...
        /* Reader service */
        service_name.assign(lockable_service_names[READER_WITH_STATUS]);
        service_name.append(component_part);   /* "keyring_reader_with_status" + ".component_keyring_file" */
        srv_registry->acquire(service_name.c_str(), ... reader);

        /* AES encryption service */
        service_name.assign(lockable_service_names[AES_ENCRYPTION]);
        srv_registry->acquire_related(         /* ★★ 以后都用 acquire_related：保证同一实现者 */
            service_name.c_str(),
            reinterpret_cast<my_h_service>(
                const_cast<keyring_reader_with_status_t *>(*reader)),
            ... aes_encryption);
        /* Generator / KEYS_METADATA_ITERATOR / COMPONENT_STATUS /
           COMPONENT_METADATA_QUERY / LOAD / WRITER 同理 */
```

> ★★★ **`acquire` 只用于第一个服务，其余 7 个都用 `acquire_related`** —— 因为 `acquire_related` 会把"参照服务的实现者组件名"拼到新服务名后面（见 [registry.md](registry.md) 的 `strchr(name, '.')` 逻辑）。这保证 8 个服务**来自同一个实现者**，不会混搭（否则可能 reader 用组件 A、writer 用组件 B）。

**★ 对照：插件侧的服务发现是"dlsym 原地覆写"**（★ 与组件的 registry 查表完全不是一回事）：

```cpp
/* sql/sql_plugin.cc:741 —— 加载时把插件内的版本号原地替换成服务器的函数指针表 */
  for (i = 0; i < array_elements(list_of_services); i++) {
    if ((sym = dlsym(plugin_dl.handle, list_of_services[i].name))) {
      uint ver = (uint)(intptr) * (void **)sym;
      if ((*(void **)sym) != list_of_services[i].service && /* already replaced */
          (ver > list_of_services[i].version ||
           (ver >> 8) < (list_of_services[i].version >> 8))) {
        ... report_error(report, ER_CANT_OPEN_LIBRARY, dlpath, 0,
                         "service '%s' interface version mismatch"); ...
      }
      *(void **)sym = list_of_services[i].service;   /* ★★ 单向、不可逆、全局覆盖 */
    }
  }
```

```cpp
/* components/libminchassis/registry.cc:173 —— 组件侧：字符串查表 + 引用计数 */
bool mysql_registry_imp::acquire_nolock(const char *service_name,
                                        my_h_service *out_service) {
  iter = mysql_registry_imp::service_registry.find(service_name);
  if (iter == mysql_registry_imp::service_registry.cend()) return true;
  mysql_service_implementation *imp = iter->second;
  imp->add_reference();        /* ★★ 引用计数 +1：unload 时靠它做安全检查 */
  *out_service = imp->interface();
  return false;
}
```

> ★★ **两代机制的本质差异**：插件 = **链接期符号绑定 + 加载期 dlsym 覆写**（无版本协商表、无引用计数、卸载不安全）；组件 = **运行期字符串查表 + 引用计数 + `acquire_related` 保证一致性**。

**★ 插件侧的配套原理注释**（`sql/sql_plugin.cc:249`，★ 解释为什么插件里的服务指针"初始化成版本号"）：

```cpp
  Each plugin must export pointer symbols for every plugin service that
  the server knows about.

  The plugin service pointers are initialized with the version of the plugin
  service that the plugin expects.

  When a dynamic plugin shared object is loaded by ::plugin_dl_add it will
  iterate over ::list_of_services, find the plugin symbol by name,
  check the service version stored in that symbol against the one stored into
  ::st_service_ref and then will replace the version stored in plugin's struct
  pointer with the actual pointer of the server's copy of the same structure.

  When that is filled in the plugin can use the newly set server structure
  through its local pointer to call into the service method pointers that point
  to the server implementation functions.

  Once set to the server's structure, the plugin's service pointer value is
  never reset back to service version.   /* ★★ 单向不可逆 */
```

## 跨版本演进时间线

| 版本 | 与可扩展架构相关的变化 | 依据 |
|---|---|---|
| 5.1 | 插件 API 奠基（`st_mysql_plugin`、存储引擎插件化） | 源码/官方文档 |
| 5.5 / 5.6 | 认证插件、审计接口、全文解析器、I_S 插件类型陆续加入 | 官方文档 |
| 5.7 | keyring 插件、validate_password、Group Replication 插件 | 官方文档 |
| **8.0** | **组件/服务体系落地（WL#4102）**：registry + dynamic loader + minimal chassis；`plugin_registry_service` 桥接两代 | 源码 |
| 8.0.17 | clone 插件；服务族持续扩充 | 源码 |
| 8.0.24 | keyring 组件发布，插件开始弃用引导 | 官方文档 + 源码弃用日志 |
| 8.0.33 | `psi_thread_v7`（新增 `detect_telemetry` / `abort_telemetry`——telemetry 能力进入服务层） | 源码注释 |
| 8.4 | keyring 插件**弃用** | 官方文档 |
| 9.x（含 9.7 LTS） | keyring 插件**移除**；Telemetry 落地；hypergraph 向社区版开放（默认 off） | 官方文档 / 先前研究 |
| 26.7 | 线程池转为开源（此前是商业版 `thread_pool` 插件）；Change Stream Applier；后量子密码；InnoDB 底层重构 | 先前研究（**非本仓库源码**） |

⚠️ 8.4 及以后的条目**不在本仓库（8.0.39）源码中**，依据是官方文档与此前的研究记录——读的时候请当作"未来方向"而非"当前实现"。

两条趋势值得单独指出：

1. **新能力一律落在服务层**：Telemetry（9.x）通过 `psi_thread_v7` 的新方法 + telemetry 相关服务进入——不是新插件类型。
2. **"商业版插件 → 开源/内置"是另一条迁移线**：线程池从商业版插件变成开源能力（26.7），与 keyring 的"插件→组件"是不同的路径——前者是**归属变化**，后者是**机制变化**。

---

## 两套机制的初始化链（源码对照）

**★ 插件侧：四个入口，没有"plugin_init()"**（⚠️ **`plugin_init()` 函数不存在**，`sql_plugin.cc` 里它只出现在注释中）：

| 真名 | 位置 | 职责 |
|---|---|---|
| `plugin_init_internals()` | `sql_plugin.cc:1401` | 建 MEM_ROOT、三把锁、`plugin_dl_array`、`plugin_array`、**12 个 `plugin_hash[type]` 桶** |
| `plugin_register_early_plugins()` | `:1501` | 处理 `--early-plugin-load` |
| `plugin_register_builtin_and_init_core_se()` | `:1556` | 注册内置插件，**只初始化 4 个核心**（daemon_keyring_proxy / MyISAM / InnoDB / CSV） |
| `plugin_register_dynamic_and_init_all()` | `:1779` | 读 `mysql.plugin` 表、加载动态插件、初始化其余全部 |

```cpp
/* sql/sql_plugin.cc:439 —— 三大全局容器（★ plugin_hash 是按 type 分桶的数组） */
static Prealloced_array<st_plugin_dl *, 16> *plugin_dl_array;      /* 所有 dlopen 的库 */
static Prealloced_array<st_plugin_int *, 16> *plugin_array;        /* 所有插件（有序） */
static collation_unordered_map<std::string, st_plugin_int *>
    *plugin_hash[MYSQL_MAX_PLUGIN_TYPE_NUM] = {nullptr};           /* ★ 12 个类型桶 */
```

```cpp
/* sql/sql_plugin.h:85 + sql/sql_plugin_ref.h:45 —— 两个核心结构 */
struct st_plugin_dl {
  LEX_STRING dl;                   /* 动态库名（不含路径） */
  void *handle;                    /* dlopen 句柄；内置为 nullptr */
  struct st_mysql_plugin *plugins; /* 指向 .so 内的 _mysql_plugin_declarations_[] */
  int version;                     /* .so 的 _mysql_plugin_interface_version_ */
  uint ref_count;
};

struct st_plugin_int {
  LEX_CSTRING name{nullptr, 0};
  st_mysql_plugin *plugin{nullptr};
  st_plugin_dl *plugin_dl{nullptr};    /* ★★ 内置为 nullptr —— "内置 vs 动态"的唯一判据 */
  uint state{0};
  uint ref_count{0};                   /* number of threads using the plugin */
  void *data{nullptr};                 /* plugin type specific, e.g. handlerton */
  MEM_ROOT mem_root;
  sys_var *system_vars{nullptr};
  enum_plugin_load_option load_option{PLUGIN_OFF};
};
```

```cpp
/* sql/sql_plugin.cc:1828 —— 内置插件注册（真名 register_builtin，不是 plugin_register_builtin） */
static bool register_builtin(st_mysql_plugin *plugin, st_plugin_int *tmp,
                             st_plugin_int **ptr) {
  tmp->ref_count = 0;
  tmp->plugin_dl = nullptr;            /* ★ 内置：plugin_dl 恒为 nullptr */
  if (plugin_array->push_back(tmp)) return true;
  *ptr = plugin_array->back() = new (&plugin_mem_root) st_plugin_int(std::move(*tmp));
  plugin_hash[plugin->type]->emplace(to_string((*ptr)->name), *ptr);
  return false;
}
```

> ★ `st_plugin_int::state` 是**位图但语义互斥**（注释原文）：*"though they look like a bitmap, plugin may only be in one of those eigenstates, not in a superposition of them :) It's a bitmap, because it makes it easier to test 'whether the state is one of those...'"*。七个值：`PLUGIN_IS_FREED/DELETED/UNINITIALIZED/READY/DYING/DISABLED/WAITING_FOR_UPGRADE`。

**★ 组件侧：`component_infrastructure_init()` 的两次 acquire + 四次 set_default**（`sql/mysqld.cc:2020`）：

```cpp
static bool component_infrastructure_init() {
  if (initialize_minimal_chassis(&srv_registry)) { ... return true; }
  /* ★ ① 引导最小机箱：建 registry + dynamic_loader */

  srv_registry->acquire("dynamic_loader_scheme_file.mysql_minimal_chassis", ...);
  srv_registry->acquire("dynamic_loader", ...);

  my_service<SERVICE_TYPE(registry_registration)> registrator(
      "registry_registration", srv_registry);

  // Sets default file scheme loader for MySQL server.
  registrator->set_default("dynamic_loader_scheme_file.mysql_server_path_filter");  /* ① */
  // Sets default rw_lock for MySQL server.
  registrator->set_default("mysql_rwlock_v1.mysql_server");                         /* ② */
  srv_registry->acquire("mysql_rwlock_v1.mysql_server", ...);
  mysql_service_mysql_rwlock_v1 = ...;
  // Sets default psi_system event service for MySQL server.
  registrator->set_default("mysql_psi_system_v1.mysql_server");                     /* ③ */
  srv_registry->acquire("mysql_psi_system_v1.mysql_server", ...);
  // Sets default mysql_runtime_error for MySQL server.
  registrator->set_default("mysql_runtime_error.mysql_server");                     /* ④ */
  srv_registry->acquire("mysql_runtime_error.mysql_server", ...);
  mysql_service_mysql_runtime_error = ...;
  return retval;
}
```

> ★★ **四次 `set_default` 的含义**：minimal chassis 自带一套"够用但简陋"的实现（rwlock / psi_system / runtime_error / scheme_file）。服务器起来了之后，用 `set_default` 把这四个的默认实现**换成服务器自己的版本**——因为服务器版本能写 error log、能做 PSI 插桩。**这就是"运行期替换实现"的实际用例**。

```cpp
/* sql/mysqld.cc:2095 —— 第二层：持久化组件恢复 */
static bool mysql_component_infrastructure_init() {
  Auto_THD thd;
  server_component_init();
  if (persistent_dynamic_loader_init(thd.thd)) {   /* ★ 读 mysql.component 表 */
    LogErr(ERROR_LEVEL, ER_COMPONENTS_PERSIST_LOADER_BOOTSTRAP);
    ... return true;
  }
  return trans_commit_stmt(thd.thd) || trans_commit(thd.thd);
}
```

**★ `minimal_chassis_init()` 的两段注册**（`components/libminchassis/minimal_chassis.cc:148`）：

```cpp
bool minimal_chassis_init(SERVICE_TYPE_NO_CONST(registry) * *registry,
                          mysql_component_t *comp_ref) {
  mysql_registry_imp::init();
  /* ① 注册 mysql_minimal_chassis 自己的 11+ 个服务 */
  for (int inx = 0;
       mysql_component_mysql_minimal_chassis.provides[inx].implementation != nullptr;
       ++inx) {
    if (imp_mysql_minimal_chassis_registry_registration.register_service(
            mysql_component_mysql_minimal_chassis.provides[inx].name,
            reinterpret_cast<my_h_service>(
                mysql_component_mysql_minimal_chassis.provides[inx].implementation)))
      return true;
  }
  ...
  /* ② comp_ref == &COMPONENT_REF(mysql_server)：把服务器组件的服务也注册进来 */
  if (comp_ref != nullptr) {
    for (int inx = 0; comp_ref->provides[inx].implementation != nullptr; ++inx) {
      if (imp_mysql_minimal_chassis_registry_registration.register_service(
              comp_ref->provides[inx].name,
              reinterpret_cast<my_h_service>(comp_ref->provides[inx].implementation)))
        return true;
    }
  }
  return false;
}
```

> ★ `mysql_minimal_chassis` 提供的服务（`minimal_chassis.cc:106`）：`registry`、`registry_registration`、`registry_query`、`registry_metadata_enumerate`、`registry_metadata_query`、`dynamic_loader`、`dynamic_loader_query`、`dynamic_loader_metadata_enumerate`、`dynamic_loader_metadata_query`、`dynamic_loader_scheme_file`、`mysql_runtime_error`、`mysql_rwlock_v1`、`mysql_psi_system_v1`。
> ⚠️ **`mysql_server` 在服务器内是静态链接的**，不是 `.so`——`server_component.cc:814-819` 的 `DECLARE_LIBRARY_COMPONENTS` 被 `#ifndef WITH_MYSQL_COMPONENTS_TEST_DRIVER` 包住，**只有 test driver 才生成 `component_mysql_server.so`**。

**★ `INSTALL PLUGIN` vs `INSTALL COMPONENT` 的落盘差异**：

```cpp
/* sql/sql_plugin.cc:2252 —— INSTALL PLUGIN 直接写 mysql.plugin 物理表 */
  Table_ref tables("mysql", "plugin", TL_WRITE);
  ...
  /* We do not replicate the INSTALL PLUGIN statement. Disable binlogging
     of the insert into the plugin table, so that it is not replicated in
     row based mode. */
  Disable_binlog_guard binlog_guard(thd);      /* ★★ INSTALL PLUGIN 不写 binlog */
  table->field[0]->store(name.str, name.length, system_charset_info);   /* name */
  table->field[1]->store(dl->str, dl->length, files_charset_info);      /* dl */
  error = table->file->ha_write_row(table->record[0]);
```

> ⚠️ **不存在 `sql/dd/impl/tables/plugins.cc`**——`mysql.plugin` 是 DD 之前的**遗留系统表**：字段定义在 `sql/sql_plugin.cc:490-501` 的 `mysql_plugin_table_def`，DD 侧仅在 `sql/dd/impl/system_registry.cc:225` 注册为 system 表，DDL 在 `scripts/mysql_system_tables.sql:206`。
> ★ 组件侧走的是 `mysql.component` 表 + manifest 文件（`persistent_dynamic_loader_init`）。
> ★ `UNINSTALL PLUGIN` 对内置插件直接报错 `ER_PLUGIN_DELETE_BUILTIN`（判据就是 `!plugin->plugin_dl`）。

---

## ★★ 真名勘误全表

| 常见说法 | 8.0.39 真相 |
|---|---|
| 插件初始化总入口是 `plugin_init()` | **不存在**该函数（只在注释里出现）。真入口 4 个：`plugin_init_internals()`(1401) / `plugin_register_early_plugins()`(1501) / `plugin_register_builtin_and_init_core_se()`(1556) / `plugin_register_dynamic_and_init_all()`(1779) |
| 内置注册函数是 `plugin_register_builtin()` | **不存在** → `static bool register_builtin()`（`sql_plugin.cc:1828`） |
| `struct st_mysql_plugin` 在 `sql/sql_plugin.h` | 在 **`include/mysql/plugin.h:629`**；`sql_plugin.h` 只有 `st_plugin_dl`；`st_plugin_int` 在 **`sql/sql_plugin_ref.h:45`** |
| 存在 `sql/dd/impl/tables/plugins.cc` | **不存在**。`mysql.plugin` 是遗留系统表，字段在 `sql_plugin.cc:490-501` |
| `plugin_hash` 是一个全局 hash | 是 **按类型分桶的数组** `plugin_hash[MYSQL_MAX_PLUGIN_TYPE_NUM]`（12 个桶），`MYSQL_ANY_PLUGIN` 要遍历全部 |
| keyring 已经是组件、没有插件了 | **两者并存**，且中间还有 `daemon_keyring_proxy_plugin` 这个**插件壳**（强制内置） |
| validate_password 已转成组件 | **两者并存**，插件版 init 里 `push_deprecated_warn(..., "validate password plugin", "validate_password component")` |
| mysqlx / clone 已转成组件 | **仍是插件**（`xpl_plugin.cc:35`、`clone_plugin.cc:651`） |
| `mysql_server` 是 `component_mysql_server.so` | 服务器内**静态链接**；只有 `WITH_MYSQL_COMPONENTS_TEST_DRIVER` 才生成 `.so` |
| `component_infrastructure_init()` 有四次 `set_default` | **属实**（`mysqld.cc:2058/2062/2070/2079`），但之前还有两次 `acquire`（dynamic_loader_scheme_file、dynamic_loader） |
| `PROVIDES_SERVICE` 只有 mysql_server 用 | 每个组件都有自己的 `BEGIN_COMPONENT_PROVIDES`；mysql_server 的清单含 **8 条 keyring_\*** |
| 有 `plugin_early_load()` 函数 | **不存在** → `plugin_early_load_one()`(2230) + `plugin_register_early_plugins()`(1501) |
| `MYSQL_ADD_PLUGIN` / `MYSQL_ADD_COMPONENT` 只是名字不同 | 差异极大：`component.cmake:53-58` 有**硬门禁**（带 `-DMYSQL_SERVER` 就 `FATAL_ERROR`）；`plugin.cmake` 无此约束 |
| `components/` 里全是组件 | `libminchassis/` 与 `library_mysys/` 是**基础设施静态库**（但 libminchassis 自己也 `DECLARE_COMPONENT`） |

**★ 三个"结构性事实"**

1. **"插件不能提供服务"这条规则有一个官方例外，而那个例外恰恰是双轨制的胶水**——`daemon_keyring_proxy_plugin` 是**插件**，却在 init 里通过 `plugin_registry_service` 桥拿到 registry 并 `register_service()` 注册 5 条组件服务。它被做成**强制内置**，与 binlog / mysql_native_password 同级。理解这一点才能明白为什么"装了 keyring 插件也能用组件 API"。

2. **`mysql_server` 组件提供的 8 条 keyring 服务不是真实现，而是"加锁 + 转发"**——真实来源由 `set_srv_keyring_implementation_as_default()` 在两条路径上决定：优先 manifest 里的 keyring 组件（Path 1），否则回落到 `daemon_keyring_proxy` 插件（Path 2）。且**只有第一个服务用 `acquire`，其余 7 个用 `acquire_related`**——因为后者会把实现者组件名拼进服务名，保证 8 个服务来自同一个实现者，不会混搭。

3. **两代机制的"服务发现"是物理层面不同的两套东西**——插件 = 链接期在 `.so` 里放一个初始化成**版本号**的全局指针，加载时服务器用 `dlsym` 找到它、**原地覆写**成函数指针表（单向不可逆、全局一份、无引用计数）；组件 = 运行期按 `"服务名.实现者名"` 字符串查 registry 表 + 引用计数。这解释了为什么插件的服务绑定"卸载不安全"、组件可以"运行期换实现"（`set_default`）。

> 注：本工作副本含少量非上游中文批注（如 `sql/sql_plugin.cc` 的 `// name的作用就是用于生成其他三个参数名字`），已按原文保留。

## 选型：新扩展该用插件还是组件

给一个可操作的决策树（基于前面所有约束）：

```
要写的扩展是什么形态？
│
├─ 需要被服务器在【固定调用点】调用、且是热路径？
│     （存储引擎、认证、审计、全文解析器…）
│      └─ YES → 插件（类型已存在；新增类型必须改服务器，通常不可能）
│
├─ 需要给【其它扩展或服务器】提供能力、需要被组合/替换？
│      └─ YES → 组件 + 服务（provides/requires）
│
├─ 只是想让【老插件】用上新能力，不想重写插件？
│      └─ YES → 不用改造成组件：经 plugin_registry_service 桥消费组件服务即可
│                （clone、mysqlx 就是这么做的）
│
├─ 必须早于 InnoDB 加载（如 keyring）？
│      └─ YES → 组件 + manifest（mysqld.my），不能用 INSTALL COMPONENT
│
└─ 需要脱离服务器复用（如给独立工具用）？
       └─ YES → 组件 + minimal chassis（mysql_migrate_keyring 的离线模式）
```

一句话判据：**"调用点固定"用插件，"能力需要被发现/替换/组合"用组件；两者之间的桥是服务。**

## 他库对比

| 系统 | 扩展机制 | 与 MySQL 的关键差异 |
|---|---|---|
| **PostgreSQL** | extension（SQL 对象 + C 函数）+ background worker；**无服务注册表** | PG 扩展在 SQL 层可见、扩展之间可以互相调用（函数即接口）；MySQL **插件不可互调**，所以必须补一层组件/服务来提供同等能力——这正是 WL#4102 缺陷 1 的由来 |
| **SQL Server** | CLR 程序集、扩展存储过程（托管沙箱） | 托管环境有类型安全与权限边界；MySQL 是**进程内全信任**（性能优先，无隔离，见 [`abi.md` 设计思想 ②](abi.md)） |
| **Oracle** | cartridge / external procedure | 类似思路，但同样没有"组件 + 服务注册表"这种显式能力市场 |
| **MongoDB** | 存储引擎插件（WiredTiger 可插拔）+ 无服务注册表 | 与 MySQL 的"存储引擎插件化"相似，但没有组件层 |
| **浏览器（历史）** | 进程外插件（PPAPI）已废弃 | 反面案例：隔离优先的路线被放弃，印证 MySQL"性能优先于隔离"的选择 |

对比的结论很清晰：**MySQL 的组件/服务体系，本质上是给"插件不能互调、不能提供能力"这个历史缺陷打的补丁**——而 PostgreSQL 因为扩展在 SQL 层天然可互调，从来不需要这一层。

## 判据：如何判断一个功能处于哪个阶段

给一个可操作的检查清单（全部可用 grep 验证）：

| 检查 | 阶段信号 |
|---|---|
| `plugin/<x>/` 与 `components/.../<x>/` 都存在 | **双轨并存**，官方通常在引导迁组件 |
| 插件源码里出现 `ER_SERVER_WARN_DEPRECATED` 指向某个组件名 | **已进入弃用引导**（如 keyring_file → component_keyring_file） |
| 只在 `components/` 下 | **只有组件**（新功能，或已完成迁移） |
| 只在 `plugin/` 下 | **只有插件**（多为服务器内生调用点：引擎、认证、审计） |
| 组件 provides 里出现"本应由插件提供的能力" | 说明存在**代理层**（如 keyring_* 的 mysql_server 实现） |

---

## Misc

### 最容易误解的三点

1. **"组件取代插件"**——错。组件是**新增层**，插件仍是引擎/认证/审计的唯一载体。
2. **"服务属于组件"**——不准确。服务是**两代共享的货币**：组件提供并消费，插件经 `plugin_registry_service` 桥**也能消费**（clone、mysqlx 都是这么用的）。
3. **"装了组件就自动用组件"**——错。多实现场景下 default 归**先注册者**，而 mysql_server 的 146 个服务在 `minimal_chassis_init()` 时就注册完了，早于任何外部组件；要让组件实现生效得显式 `set_default`（详见 [`service.md` 的实战章节](service.md)）。

### 一句话总结

MySQL 的可扩展架构是**"插件做角色、组件做能力、服务做货币"**的双轨制：插件因为"调用点固定 + 高频"而留下来，组件因为"可组合 + 可替换 + 可封装"而加上去，服务把两者接成一张网——所谓"迁移"，发生在**能力下沉与服务化**这一层，而不是插件被整体改写。

---

## 参考

**官方文档**
- WorkLog: [WL#4102 Service registry and component infrastructure](https://dev.mysql.com/worklog/task/?id=4102)（组件体系的立项动机与四个插件缺陷）
- *MySQL 8.0 Reference Manual → Extending MySQL*（plugins / components / services 三章并列的现状本身就是"双轨制"的佐证）

**相关文档**
- 插件机制：[`plugin.md`](plugin.md)
- 组件机制：[`component.md`](component.md)
- 服务与桥：[`service.md`](service.md)
- ABI 契约：[`abi.md`](abi.md)
- 迁移实例：[`keyring.md`](keyring.md)；组件消费实例：[`clone.md`](clone.md)、[`mysqlx.md`](mysqlx.md)
