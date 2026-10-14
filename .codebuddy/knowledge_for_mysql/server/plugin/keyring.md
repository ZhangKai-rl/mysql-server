# Keyring 深度解析（插件 → 组件迁移）

> 基于 MySQL 8.0.39 源码。Keyring 是 MySQL 的**密钥管理（master key / 表空间密钥 / binlog 与 redo 加密密钥）**机制，也是**唯一同时需要插件、组件、服务、代理层、manifest、迁移工具**才能工作的功能——因此它是观察"插件 → 组件"两代机制交替的最佳样本。
>
> **边界**：本篇讲 keyring 的双实现、服务器代理层、服务族与迁移路径；插件框架见 [`plugin.md`](plugin.md)，组件与服务见 [`component.md`](component.md) / [`service.md`](service.md)，manifest 启动见 [`component.md` 的启动引导](component.md#启动引导四条链不是三条)；**不讲**：InnoDB 表空间加密本身的算法（`../innodb/`）、binlog 加密。

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

Keyring 负责保管 MySQL 的各类密钥：InnoDB 表空间的 master key、redo/undo 加密密钥（企业版相关）、binlog 加密密钥、以及插件/组件自存的秘密。8.0.39 里它以**两种形态同时存在**：

- **插件版** `keyring_file`（`MYSQL_KEYRING_PLUGIN`，类型 10）——老机制；
- **组件版** `component_keyring_file`（URN `file://component_keyring_file`）——新机制。

两者功能相同、文件格式不同，官方正把用户从前者迁到后者。

### 用途

静态数据加密（TDE）的密钥底座：没有 keyring，加密表空间在启动时解不开 master key；`ALTER INSTANCE ROTATE INNODB MASTER KEY` 等密钥运维也依赖它。

### 版本演进

| 版本 | 变化 | 出处 |
|---|---|---|
| 5.7 | keyring 插件引入（`MYSQL_KEYRING_PLUGIN`） | 插件类型常量 |
| 8.0 | 组件体系落地后，服务器新增**代理插件** `daemon_keyring_proxy`，把 keyring 插件包装成组件服务 | `sql/daemon_proxy_keyring/daemon_proxy_keyring.cc` |
| 8.0.24 | keyring **组件**发布（`component_keyring_file` 等）；插件开始弃用引导 | 官方文档；源码 `plugin/keyring/keyring.cc` 的弃用日志 |
| 8.0.39 | 插件 init 时打印 `ER_SERVER_WARN_DEPRECATED` 指向 `component_keyring_file` | `plugin/keyring/keyring.cc:114` |
| 8.4 / 9.x | 官方路线：keyring 插件在 8.4 弃用、9.x 移除 | 官方文档（源码无版本映射，标注为官方路线） |

---

## 理论基础

### 第零条：为什么 keyring 是理解"两代机制交替"的关键样本？

因为它把前面四篇里的**所有概念都逼出来同时上场**：

```
插件（keyring_file，MYSQL_KEYRING_PLUGIN）
   ↓ 服务器代理（daemon_keyring_proxy，内置 daemon 插件）
组件服务（keyring_writer.mysql_server 等）
   ↓ registry（多实现共存，default 归先注册者）
消费者（InnoDB、binlog 加密、clone…）

组件（component_keyring_file）也提供同一批服务 → 两个实现并存
迁移工具（mysql_migrate_keyring）在离线/在线两种模式下搬运密钥
manifest（mysqld.my）解决"keyring 必须早于 InnoDB"的加载时序
```

其它功能（clone、mysqlx）只是"插件消费组件服务"，只有 keyring 是**新旧同功能、需替换、且要搬数据**。

### 1. 为什么必须有"服务器代理"这一层？

这是本篇最核心的一问。InnoDB 要用 keyring——但 **InnoDB 自己也是插件**，而 WL#4102 的缺陷 1 正是"插件之间无法互调"（见 [`component.md`](component.md) 的理论基础）。更要命的是：keyring **插件**只能"被服务器调用"，**不能对外提供组件服务**（插件没有 provides）。

于是必须有第三方把插件的 vtable 翻译成服务，这个第三方就是内置的 **`daemon_keyring_proxy`**（`sql/daemon_proxy_keyring/`）。源码开头把这个定位写得毫不含糊：

```
@page PAGE_COMPONENT_DAEMON_KEYRING_PROXY component_daemon_keyring_proxy

This plugin implementation provides wrappers over keyring plugin
implementation and exposes functionality provided by plugin through
keyring component service APIs.

Following keyring component services implemented:
 - keyring_generate / keyring_keys_metadata_iterator / keyring_component_status
 - keyring_reader / keyring_reload / keyring_writer
For rest of the services, wrappers use keyring plugin's APIs.
```

**结论**：`keyring_writer.mysql_server` 这类"由 mysql_server 提供"的实现，**真实来源是代理插件，代理插件再转发到实际安装的 keyring 插件**。三层转发只是为了绕开"插件不能提供服务"这条限制。

### 2. 为什么不能同时启用插件版和组件版？

两个原因叠加：

1. **代理只认第一个**：`daemon_proxy_keyring` 的遍历回调命中第一个 keyring 插件后就返回 error 让 `plugin_foreach` 停止（源码注释："this function should get executed only for the first plugin. This is why it always returns error"）。
2. **官方运维约束**：手册明确"同一时刻只能启用一个 keyring 组件或插件"。

⚠️ 后者是**运维约束而非代码强制**——源码里没有"检测到并存就报错"的检查（诚实标注）。

### 3. 为什么迁移必须"搬数据"而不能"卸载再装"？

插件版与组件版的**文件格式不同**（插件的 `Buffered_file_io` 是"版本串 + 序列化 buffer + EOF TAG + SHA256 摘要"，且带字节序标记；组件有自己的后端与配置格式），元数据组织也不一样。因此官方提供专用工具 `mysql_migrate_keyring` 做密钥级搬迁。

### 4. 为什么在线迁移要一个 `keyring_operations` 开关？

迁移过程中若业务同时写新密钥/删旧密钥，目标端就会缺项或多余。`keyring_operations=OFF` 专门挡住**写类操作**（`keyring_key_generate` / `keyring_key_store` / `keyring_key_remove`），**读不受影响**——既保证迁移一致性，又让业务在迁移窗口内仍能正常读加密数据（源码注释原文，见 `sql/sys_vars.cc:7191-7208`）。

### 5. 为什么 keyring 组件必须走 manifest 而不是 `INSTALL COMPONENT`？

因为 keyring 必须在 **InnoDB 初始化之前**就绪（否则加密表空间解不开 master key），而 `INSTALL COMPONENT` 的持久化落在 `mysql.component`——一张 **InnoDB 表**，加载时机必然晚于 InnoDB。manifest（`mysqld.my`）在启动最早期读取，绕开了这个循环依赖（详见 [`component.md`](component.md) 的"启动引导四条链"）。

---

## 核心实现

### 插件版 `keyring_file`

```c
/* plugin/keyring/keyring.cc:242-267 */
static struct st_mysql_keyring keyring_descriptor = {
    MYSQL_KEYRING_INTERFACE_VERSION,
    mysql_key_store, mysql_key_fetch, mysql_key_remove, mysql_key_generate,
    mysql_key_iterator_init, mysql_key_iterator_deinit, mysql_key_iterator_get_key};

mysql_declare_plugin(keyring_file){
    MYSQL_KEYRING_PLUGIN,
    &keyring_descriptor,
    "keyring_file",
    ...
    "store/fetch authentication data to/from a flat file",
    ...
} mysql_declare_plugin_end;
```

`keyring_init()` 里第一行就打弃用警告（`:114`）：

```cpp
logger->log(WARNING_LEVEL, ER_SERVER_WARN_DEPRECATED, "keyring_file plugin",
            "component_keyring_file");
```

随后：`create_keyring_dir_if_does_not_exist` → `Keys_container` + `Buffered_file_io`（`:137-149`），并声明"本 keyring 支持的文件版本列表"（`allowedFileVersionsToInit`）。

**文件格式（值得单独看）**——`plugin/keyring/buffered_file_io.cc`：

```
[ 版本字符串 ] [ 序列化后的密钥 buffer ] [ EOF TAG ] [ SHA256 摘要 ]
```

配套三个机制：

1. **`.backup` 文件**：`recreate_keyring_from_backup_if_backup_exists()`——启动时若有 `<file>.backup` 就先从备份恢复（备份损坏则删除并告警）。写密钥时先备份再写，避免写一半崩溃导致密钥全丢。
2. **按版本校验**：`Checker` 工厂按文件版本取校验器，`check_file_structure()` 逐个试，都不匹配则 `ER_KEYRING_INCORRECT_FILE`。
3. **字节序/架构转换**：`Converter::Arch` 记录文件写入端的架构，读取时若 `native_arch != file_arch` 就 `Converter::convert_data()` —— 这让 keyring 文件可以跨平台搬运。

### 组件版 `component_keyring_file`

```
components/keyrings/keyring_file/
├── keyring_file.cc/h                  DECLARE_COMPONENT(component_keyring_file, "component_keyring_file")
├── backend/                           文件后端（真正读写密钥文件）
├── config/                            读取组件自己的配置
└── service_implementation/            7 个服务实现
    keyring_encryption / generator / keys_metadata_iterator /
    load / metadata_query / reader / writer
```

provides（已核实 `keyring_file.cc:273-282`）：`keyring_aes`、`keyring_generator`、`keyring_load`、`keyring_keys_metadata_iterator`、`keyring_component_status`、`keyring_component_metadata_query`、`keyring_reader_with_status`、`keyring_writer`，外加 **`log_builtins` / `log_builtins_string`**（自带日志实现，原因见 [`component.md` 的组件盘点](component.md)：`migrate_keyring` 工具以 `minimal_chassis_init(reg, nullptr)` 自举时没有 mysql_server 的日志服务）。requires：`registry`、`log_builtins`、`log_builtins_string`。

产物名由 `cmake/component.cmake` 决定：`STRING(TOLOWER component_${component} target)` + `PREFIX ""` → **`component_keyring_file.so`** → URN `file://component_keyring_file`。

### 服务器代理：`daemon_proxy_keyring`

这是"插件被服务化"的具体机制，代码很短但含义很重：

```cpp
static bool key_plugin_cb_fn(THD *, plugin_ref plugin, void *arg) {
  auto callback = reinterpret_cast<Callback *>(arg);
  callback->invoke(
      reinterpret_cast<st_mysql_keyring *>(plugin_decl(plugin)->info));
  plugin_unlock(nullptr, plugin);
  // this function should get executed only for the first plugin. This is why
  // it always returns error. plugin_foreach will stop after first iteration.
  return true;
}

static bool iterate_plugins(std::function<bool(st_mysql_keyring *keyring)> fn) {
  Callback callback(fn);
  plugin_foreach(current_thd, key_plugin_cb_fn, MYSQL_KEYRING_PLUGIN, &callback);
  return callback.result();
}

/* 例：keyring_generator 的代理实现 */
static DEFINE_BOOL_METHOD(generate, (const char *data_id, const char *auth_id,
                                     const char *data_type, size_t data_size)) {
  bool retval = iterate_plugins([&](st_mysql_keyring *keyring) {
    return keyring->mysql_key_generate(data_id, data_type, auth_id, data_size);
  });
  return retval;
}
```

四点：

1. **每次调用都 `plugin_lock` / `plugin_unlock`**——保证调用期间插件不会被卸载（引用计数，见 [`plugin.md`](plugin.md) 的延迟收割）。
2. **`plugin_decl(plugin)->info`** 直接取类型 vtable，无哈希查找（热路径优化，与 `plugin_data` 同理）。
3. **只代理第一个插件**，这既是性能取舍，也是"不能并存"的技术根源。
4. 代理把"非服务化的插件能力"接进了服务市场，使得**消费者完全不需要知道背后是插件还是组件**——这正是迁移能平滑进行的原因。

### 服务族与"两个实现并存"

registry 里同一批 keyring 服务**至少有两个实现**：

| 实现名 | 来源 | 说明 |
|---|---|---|
| `keyring_writer.mysql_server`、`keyring_reader_with_status.mysql_server`、`keyring_aes.mysql_server` … | `server_component.cc:685-692`（实现在 `daemon_proxy_keyring`） | **代理到 keyring 插件** |
| `keyring_writer.component_keyring_file` … | `component_keyring_file` 自己提供 | 组件自供 |

`register_service` 用 `emplace`（先注册者胜），而 mysql_server 的 146 个服务在 `minimal_chassis_init()` 时一次性注册——**早于任何外部组件**。所以 default 永远是 `.mysql_server`；装了组件后它的实现只是"并存"，**不会自动抢走 default**。想让组件实现成为 default 必须显式 `set_default`（见 [`service.md` 的实战章节](service.md)）。

这带来一个必须注意的行为：InnoDB `acquire("keyring_reader_with_status")` 拿到的**默认是代理实现**（转发到插件，或若插件没装则失败），而不是组件实现——这也是为什么要靠 `acquire_related` 保证"读/写/生成来自同一实现者"（`storage/innobase/os/os0enc.cc`）：

```cpp
reg_srv->acquire("keyring_reader_with_status", &h_keyring_reader_service) ||
reg_srv->acquire_related("keyring_writer", h_keyring_reader_service, &h_keyring_writer_service) ||
reg_srv->acquire_related("keyring_generator", h_keyring_reader_service, &h_keyring_generator_service)
```

### 消费者

- **InnoDB**：`storage/innobase/os/os0enc.cc`——acquire 一次，之后**多线程共享句柄**（服务无状态 ⇒ 无需 per-thread acquire，见 [`service.md`](service.md) 的并发小节）。
- **binlog 加密**：`sql/rpl_log_encryption.cc` 也消费 keyring 服务。
- **老接口并存**：`sql/sql_plugin_services.h:301-302` 里还留着老版插件服务 `mysql_keyring_service`（`my_key_store/fetch/remove/generate`）供老插件使用。

### 迁移：两条路径

**离线（工具自举）**：`client/migrate_keyring/components.cc:47` 里 `minimal_chassis_init(&components_registry, nullptr)`——工具自己拉起一套 registry + dynamic loader，同时加载**源插件**与**目标组件**，逐 key 搬迁。这是 [`component.md`](component.md) 里"minimal chassis 可脱离服务器复用"的真实用途。

**在线（连服务器）**：工具带 `--online-migration` + 连接参数，由**服务器端** `sql/migrate_keyring.cc` 的 `Migrate_keyring` 类执行：

```
1) SET GLOBAL KEYRING_OPERATIONS=0        /* 禁写，业务仍可读 */
2) load_plugin(SOURCE)                    /* 源：keyring 插件（plugin_early_load_one） */
3) load_component() 或 load_plugin(DEST)  /* 目标：组件（file:// URN，check_valid_path 禁路径）或另一插件 */
4) fetch_and_store_keys()                 /* 源 iterator 逐 key → 目标 writer store */
5) 失败则回滚：把已写入目标的 key 逐个删除
6) SET GLOBAL KEYRING_OPERATIONS=1
```

三点设计值得记：

- **第 1 步的禁写**是在线迁移一致性的支点（只挡 generate/store/remove，读放行）。
- **第 5 步的回滚**：`m_source_keys` 记录已写入的 key，出错时逐个删除——**目标端不留半截状态**。
- **第 3 步的安全检查**：目标 URN 由工具拼 `file://` 前缀，并用 `check_valid_path()` 拒绝路径分隔符（与插件加载同一条安全规则，见 [`plugin.md`](plugin.md)）。

**收尾**：迁移完成后，去掉 `--early-plugin-load`，改用 manifest（`mysqld.my`）加载组件：

```json
{"read_local_manifest": false, "components": "file://component_keyring_file"}
```

（字段来自 `include/manifest.h` 定义的 schema：`read_local_manifest` 布尔 + `components` 字符串。）

---

## 系统变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `keyring_file_data` | `MYSQL_DEFAULT_KEYRINGFILE` | 插件版密钥文件路径（`PLUGIN_VAR_RQCMDARG \| PLUGIN_VAR_NODEFAULT`，带 check/update 回调，改值会重新 init 容器） |
| `keyring_open_mode` | true | 插件版是否只读打开（`PLUGIN_VAR_INVISIBLE`） |
| **`keyring_operations`** | **true** | 服务器侧总开关；`NON_PERSIST GLOBAL`、`NO_CMD_LINE`，受 `LOCK_keyring_operations`（`PolyLock_mutex`）保护，有 `check_keyring_access`。**关闭后 generate/store/remove 报错，读不受影响** |
| 组件配置 | — | 组件从自己的配置（`components/keyrings/keyring_file/config/`）读取，不走插件变量 |

---

## Misc

### 易混淆：四个"keyring"分别是什么

| 名字 | 本质 | 作用 |
|---|---|---|
| `keyring_file`（插件） | `MYSQL_KEYRING_PLUGIN` 插件 | 老实现，自己管文件 |
| `component_keyring_file`（组件） | component | 新实现，自己管文件 |
| `keyring_*`（服务族） | 组件服务 | 消费者（InnoDB）真正调用的接口 |
| `daemon_keyring_proxy` | 内置 **daemon 插件** | 把插件的 vtable 包装成上面那些服务 |

还有一个容易漏的：**老版插件服务** `mysql_keyring_service`（`my_key_*`）仍在 `list_of_services` 里，供老插件使用——它和组件服务族是两代接口。

### 弃用与迁移的现实影响

- 8.0.39：每次 keyring 插件 init 都会打一条弃用警告（错误日志里能看到），指向 `component_keyring_file`。
- 8.4（官方路线）：keyring 插件弃用；9.x：移除。届时**必须**完成迁移，否则加密数据不可用。
- 迁移的硬门槛：必须在**业务可停写**（`keyring_operations=0`）或**实例可离线**的窗口内做。

### 一句话总结

Keyring 是"**同一个功能同时以插件和组件存在，靠服务器内置代理插件把插件包装成服务接进服务市场，再靠专用工具搬密钥完成替换**"——它把插件体系的封闭、组件体系的开放、以及两者交替时的胶水成本，全部暴露在同一条链路上。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → The Keyring Service / Keyring Components / Keyring Plugin*
- *MySQL 8.0 Reference Manual → Keyring Component Installation*（manifest 加载、与插件不可并存）
- *MySQL 8.0 Reference Manual → Keyring Migration（mysql_migrate_keyring）*

**相关文档**
- 插件框架：[`plugin.md`](plugin.md)
- 组件与服务、多实现 default：[`component.md`](component.md)、[`service.md`](service.md)
- ABI 契约：[`abi.md`](abi.md)
