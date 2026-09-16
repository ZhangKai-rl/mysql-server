# 插件（Plugin）体系深度解析

> 基于 MySQL 8.0.39 源码，涵盖插件的 ABI 契约、类型系统、加载/安装/卸载全链路、引用计数与延迟收割、插件系统变量。
>
> **边界**：本篇讲插件体系（老一代扩展机制）；组件基础设施（Component）见 [`component.md`](component.md)，服务（Service，含 registry 内部实现与老版 plugin service）见 [`service.md`](service.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/命令行选项](#相关的系统变量命令行选项)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

插件是 MySQL "服务器 ↔ 外部代码"的**单向扩展契约**：一个外部 `.so`（或静态链接进 mysqld 的代码段）通过 `st_mysql_plugin` 描述符声明"我是什么类型、提供什么能力"，内核在启动或 `INSTALL PLUGIN` 时把它登记进 `plugin_array` / `plugin_hash`，此后按类型分派调用。内核侧用 `st_plugin_int` 管理每个已登记插件的生命周期（状态、引用计数、变量、所属动态库）。

### 用途

8.0 时代几乎所有可插拔能力仍然走这条老路：存储引擎（**InnoDB、MyISAM 本身也是插件**）、认证（`caching_sha2_password` 等）、审计、全文解析器、keyring、clone、Group Replication、半同步复制、`mysqlx`（X plugin）……组件体系（WL#4102）虽然已经发布，但插件是存量功能的实际载体，二者并存。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.1 | 插件 API 奠基：`st_mysql_plugin`、存储引擎插件化 |
| 5.5 | 认证插件类型（`MYSQL_AUTHENTICATION_PLUGIN`，pluggable authentication） |
| 5.5.31 / 5.6.10 | 审计接口（`MYSQL_AUDIT_PLUGIN`） |
| 5.6 | 全文解析器（FTPARSER）、DAEMON、INFORMATION_SCHEMA 插件类型 |
| 5.7 | KEYRING、VALIDATE_PASSWORD、GROUP_REPLICATION、REPLICATION（复制观察者）类型 |
| 8.0 | CLONE 插件（8.0.17）、内置 `daemon_keyring_proxy`、`mysqlx`；组件基础设施（WL#4102）随 8.0 登场 |
| 8.0.24 | keyring 组件发布，keyring 插件 init 时打印弃用警告（`ER_SERVER_WARN_DEPRECATED`，指向 `component_keyring_file`） |

> 为什么演进成组件见 [`component.md` → 理论基础](component.md#理论基础)。

---

## 理论基础

### 设计思想与权衡

**1. 类型化 vtable = "角色契约"，而非能力市场。**

插件必须先"认领" 12 种类型之一（`MYSQL_STORAGE_ENGINE_PLUGIN` … `MYSQL_CLONE_PLUGIN`），每种类型对应一个 `st_mysql_xxx` vtable。这带来两个结构性质：

- **单向**：只有服务器调插件（`plugin_foreach`、`ha_initialize_handlerton`、`plugins_dispatch`……），插件之间、插件与"服务器后来新增的子系统"之间没有通道。
- **封闭**：类型集合由头文件写死，服务器必须**预先**知道"有哪些扩展场景"。要支持一种新扩展，得先改服务器开新类型 + 新 vtable + 新分派代码——这正是 WL#4102 列出的缺陷之一（"服务器必须预知所有可能的扩展场景"）。

好处：调用点（如 `acl_authenticate`）可以拿到强类型 vtable 直接调用，无注册表查找、无 acquire/release 开销。

**2. builtin / dynamic 同构：一份 `st_mysql_plugin`，两种编译形态。**

`mysql_declare_plugin` 宏根据 `MYSQL_DYNAMIC_PLUGIN` 是否定义展开成两套符号（详见核心实现），但数据结构完全一致。理由：

- 核心插件（InnoDB、binlog、认证）**静态链接**进 mysqld：启动期不用 dlopen，失败面小、无符号解析开销；
- 可选插件打包成 `.so`：按需 `INSTALL PLUGIN`；
- 内核统一用 `st_plugin_int::plugin_dl == nullptr` 判据区分二者（builtin 判据），其余代码路径共享。

**3. builtin 不引用计数。**

`intern_plugin_lock()` 第一行就是 `if (!pi->plugin_dl) return pi;`——builtin 插件永不卸载（`UNINSTALL PLUGIN` 直接报 `ER_PLUGIN_DELETE_BUILTIN`），句柄不可能悬空，省掉全部 ref 开销。代价：builtin 插件无法在运行期移除，改动必须重启。

**4. 引用计数 + 状态位"延迟收割"：卸载安全的另一半。**

`UNINSTALL PLUGIN` **并不立即** deinit。它只把 `state` 改为 `PLUGIN_IS_DELETED`；此时 `intern_plugin_lock()` 对新请求返回 `nullptr`（新引用拿不到句柄），而已持有句柄的线程继续安全使用——真正的 deinit 推迟到 `ref_count` 归零后的下一次 `plugin_unlock`（`reap_needed` → `reap_plugins()`）。这与 RCU 的"宽限期"理念同构（虽然靠互斥锁而非无锁）：**标记删除 → 等所有引用退出 → 物理回收**。代价：被繁忙使用的插件永远卸不掉，只能报 `WARN_PLUGIN_BUSY` 警告；关闭时还配了"强制关停"兜底。

**5. debug 构建下 `plugin_ref` 是二级指针。**

release 构建 `plugin_ref = st_plugin_int *`；debug 构建 `plugin_ref = st_plugin_int **`——每个"锁"都是一块独立 `malloc` 的小内存，里面存 `st_plugin_int*` 副本（`sql/sql_plugin_ref.h` 注释直接指向 `intern_plugin_lock()` 说明原因）：valgrind/ASan 能精确追踪"哪个锁忘了释放"（泄漏）和"双重解锁"（踩 freed memory）。这是把内存调试器当引用计数审计器的经典手法。

**6. `mysql.plugin` 留在 InnoDB 系统表，不进数据字典（DD）。**

8.0 把绝大多数 `mysql.*` 元数据搬进了 DD，唯独 plugin 表仍是普通 InnoDB 表（`scripts/mysql_system_tables.sql` 里 `CREATE TABLE plugin(name, dl)`，DD 侧只是 `register_table("plugin", system)` 登记为 system 表）。权衡：插件记录必须在 **InnoDB 完全可用之前**被读取——启动链路里 `plugin_register_dynamic_and_init_all()` 要在 DD init 之后立刻读这张表加载插件，而 DD 的完整初始化（`dd::init`）本身要晚于早期插件（keyring 要供 InnoDB 读加密表空间）；且该表只有两列、结构 5.x 以来未变，迁移 DD 收益低、风险高。**例外**：I_S 插件的视图定义元数据写进了 DD（`dd::info_schema::store_dynamic_plugin_I_S_metadata()`），因为视图生成依赖 DD 基础设施。

**7. THDVAR 用 offset 布局 + 永不回收的 bookmark。**

插件 THD 变量（`PLUGIN_VAR_THDLOCAL`）存放在 `thd->variables.dynamic_variables_ptr` 这块 per-session 内存里，位置由 offset 决定。关键设计：offset 的 bookmark **永不从哈希表删除**（`register_var()` 注释明说），插件反复 install/uninstall 会**复用同一个 offset**。否则：老会话在插件卸载重装后，其 `dynamic_variables_ptr` 里的偏移会指向错的内存（布局漂移），造成越界读写。这是"会话生命周期 > 插件生命周期"这一事实逼出来的设计。

**8. 锁序约定（违反即死锁）。**

- `LOCK_plugin_install` → `LOCK_plugin`（INSTALL/UNINSTALL 串行化锁在前）；
- `LOCK_system_variables_hash` → `LOCK_plugin`（注释明说：同时持有二者时必须先拿 sysvar 锁）；
- 调 `plugin_del()` 前必须持有 `LOCK_plugin_delete`。

### 理论溯源

- **ABI 版本规则**：`MYSQL_PLUGIN_INTERFACE_VERSION` 高字节必须相等（不兼容的硬边界）、低字节向后兼容；老版本插件的 `st_mysql_plugin` 按 `_mysql_sizeof_struct_st_plugin_` 逐字段 `memcpy` 搬进新结构（缺失字段零填充）——这是"结构体版本化 + 字段级迁移"的手法，C ABI 领域经典。
- **延迟收割**：如上，与 RCU 宽限期同构。
- **状态机**：`st_plugin_int::state` 是单 bit 状态机（`FREED/DELETED/UNINITIALIZED/READY/DYING/DISABLED/WAITING_FOR_UPGRADE`），"同一时刻只有一种本征态、bitmap 便于批量判断"。

### 算法与数据结构

- `plugin_hash[12]`：按类型分桶的 `collation_unordered_map<name, st_plugin_int*>`，**大小写不敏感**，名字查找 O(1)。`MYSQL_ANY_PLUGIN`（-1）查找时遍历 12 个桶。
- `plugin_array`：`Prealloced_array` 顺序数组，遍历顺序 = 注册顺序（`plugin_foreach` 依赖它；存储引擎遍历时 binlog 强制最先）；卸载后**槽位不删除**（标 `PLUGIN_IS_FREED` 供复用），保证遍历期间的指针稳定。
- `plugin_dl_array`：库句柄层。`st_plugin_dl::ref_count` = 从该库加载的**插件数**（不是线程引用数），归零即 `free_plugin_mem()` → `dlclose()`。一个 .so 可含多个插件，共享一个句柄。

### 他库对比与演进动机

- **PostgreSQL**：扩展是"对象集合"（函数/类型/操作符注册进系统目录），加载靠 `shared_preload_libraries`，没有"类型化角色"概念；MySQL 插件是进程内函数指针层面的契约。两者都回避了跨进程 IPC，但 PG 的扩展可以互相引用（SQL 层对象可见），MySQL 插件互相看不见。
- **演进动机**：`components/libminchassis/dynamic_loader.cc` 开篇明列插件体系四个架构缺陷——(1) 插件只能"对服务器说话"，插件之间无法互调；(2) 插件可直呼服务器全部全局符号，**没有封装**；(3) 没有显式依赖清单，初始化顺序靠硬编码；(4) 插件依赖"正在运行的服务器"。组件体系（WL#4102）逐条解决，详见 `component.md`。

---

## 核心实现

### 主链路：启动加载

启动期有三个入口、三批插件，全部经 `plugin_initialize()` 激活：

```
mysqld_main()                                        sql/mysqld.cc
 ├─ plugin_register_early_plugins()                  ① --early-plugin-load
 │     ├─ plugin_init_internals()                    初始化锁/哈希/数组
 │     ├─ 遍历 opt_early_plugin_load_list → plugin_load_list(..., load_early=true)
 │     └─ plugin_init_initialize_and_reap()
 │
 ├─ plugin_register_builtin_and_init_core_se()       ② builtin（mandatory+optional）
 │     ├─ 遍历 mysql_mandatory_plugins[] 再 mysql_optional_plugins[]
 │     │    └─ test_plugin_options() → register_builtin()
 │     └─ 仅此阶段初始化 daemon_keyring_proxy / MyISAM / InnoDB / CSV
 │
 ├─ dd::init(...)                                    数据字典初始化
 │
 └─ plugin_register_dynamic_and_init_all()           ③ 动态插件
       ├─ 遍历 opt_plugin_load_list → plugin_load_list()      --plugin-load(-add)
       ├─ plugin_load()                              读 mysql.plugin 表逐行加载
       └─ plugin_init_initialize_and_reap()

init_server_components()                             （此后才轮到组件基础设施，见 02）
```

① 之所以最早：**keyring 必须在 InnoDB 之前就绪**——InnoDB 启动要读加密表空间密钥，`--early-plugin-load` 保证 keyring 插件在核心引擎初始化前可用。因此 early 插件必须声明 `PLUGIN_OPT_ALLOW_EARLY`，否则报 `ER_PLUGIN_NOT_EARLY`。②③ 之间隔了 `dd::init`，因为读 `mysql.plugin` 表需要 DD 已初始化。

`mysql_mandatory_plugins[]` / `mysql_optional_plugins[]` 是 CMake 生成的符号数组（`sql/sql_builtin.cc.in` 模板展开），而非段属性。**"mandatory"的真正含义**：位于 mandatory 数组 → 强制 `load_option = PLUGIN_FORCE` → 用户命令行无法覆盖（`construct_options()` 不生成对应开关）→ 初始化失败则 `reaped_mandatory_plugin = true` → **服务器拒绝启动**。另外 `PERFORMANCE_SCHEMA` 也被硬编码为 FORCE。

### 插件的 ABI 契约：`st_mysql_plugin`

```c
struct st_mysql_plugin {
  int type;           /* MYSQL_XXX_PLUGIN 之一 */
  void *info;         /* 指向类型专属 vtable；其第一个字段必须是 interface version */
  const char *name;
  const char *author;
  const char *descr;
  int license;        /* PLUGIN_LICENSE_PROPRIETARY/GPL/BSD，仅展示 */
  int (*init)(MYSQL_PLUGIN);
  int (*check_uninstall)(MYSQL_PLUGIN);
  int (*deinit)(MYSQL_PLUGIN);
  unsigned int version;
  SHOW_VAR *status_vars;
  SYS_VAR **system_vars;
  void *__reserved1;
  unsigned long flags;  /* PLUGIN_OPT_NO_INSTALL/NO_UNINSTALL/ALLOW_EARLY/DEFAULT_OFF/DEPENDENT_EXTRA_PLUGINS */
};
```

硬约束：`plugin->info` 指向的 vtable **第一个字段必须是 `int interface_version`**——内核用 `*(int*)plugin->info` 做类型级 ABI 校验。

`mysql_declare_plugin` 的双形态展开（builtin 无 `MYSQL_DYNAMIC_PLUGIN`，dynamic 由 cmake/plugin.cmake 定义之）：

```c
/* builtin：带前缀符号，登记进 sql_builtin.cc.in 生成的数组 */
#define __MYSQL_DECLARE_PLUGIN(NAME, VERSION, PSIZE, DECLS)         \
  MYSQL_PLUGIN_EXPORT int VERSION = MYSQL_PLUGIN_INTERFACE_VERSION; \
  MYSQL_PLUGIN_EXPORT int PSIZE = sizeof(struct st_mysql_plugin);   \
  MYSQL_PLUGIN_EXPORT struct st_mysql_plugin DECLS[] = {
/* dynamic：固定三符号，dlopen 后 dlsym 取出 */
#define __MYSQL_DECLARE_PLUGIN(...)                          \
  MYSQL_PLUGIN_EXPORT int _mysql_plugin_interface_version_ = \
      MYSQL_PLUGIN_INTERFACE_VERSION;                        \
  MYSQL_PLUGIN_EXPORT int _mysql_sizeof_struct_st_plugin_ =  \
      sizeof(struct st_mysql_plugin);                        \
  MYSQL_PLUGIN_EXPORT struct st_mysql_plugin _mysql_plugin_declarations_[] = {
```

真实例子（`sql/binlog.cc`）：binlog 是一个 `MYSQL_STORAGE_ENGINE_PLUGIN` 伪存储引擎，`info` 指向 `binlog_storage_engine`（一个 `st_mysql_storage_engine`，只有 interface_version 字段），init 是 `binlog_init`。`mysql_declare_plugin_end` 追加全 0 哨兵项，内核以 `plugin->info == nullptr` 终止遍历。

### 插件类型全集与分派表

类型不是 enum，而是一组 `#define`（`MYSQL_UDF_PLUGIN 0` … `MYSQL_CLONE_PLUGIN 11`，共 `MYSQL_MAX_PLUGIN_TYPE_NUM = 12` 个）。**没有 QUERY REWRITE 类型**（`MYSQL_QUERY_REWRITE_PLUGIN` 只存在于注释里）。

| 类型 | vtable | 内核侧包装的 init/deinit |
|---|---|---|
| STORAGE ENGINE | `st_mysql_storage_engine` | `ha_initialize_handlerton` / `ha_finalize_handlerton` |
| INFORMATION_SCHEMA | `st_mysql_information_schema` | `initialize_schema_table` / `finalize_schema_table` |
| AUDIT | `st_mysql_audit` | `initialize_audit_plugin` / `finalize_audit_plugin` |
| AUTHENTICATION | `st_mysql_auth` | 无包装，直调 `plugin->init` |
| KEYRING | `st_mysql_keyring` | 无包装 |
| CLONE | `Mysql_clone` | 无包装 |
| 其余（UDF/FTPARSER/DAEMON/REPLICATION/VALIDATE_PASSWORD/GROUP_REPLICATION） | 各 vtable | 无包装 |

分派表 `plugin_type_initialize[] / plugin_type_deinitialize[]`：只有 SE / I_S / AUDIT 三类有内核侧包装——它们需要内核先分配/注册数据结构（`handlerton`、schema table、全局 audit mask）再调插件自己的 init。

### 内核数据结构与全局状态

```c
struct st_plugin_int {
  LEX_CSTRING name;
  st_mysql_plugin *plugin;      /* 描述符（builtin 指静态数组，dynamic 指 .so 内数组） */
  st_plugin_dl *plugin_dl;      /* 所属库句柄；builtin 为 nullptr（builtin 判据！） */
  uint state;                   /* PLUGIN_IS_* 单 bit 状态机 */
  uint ref_count;               /* 使用该插件的线程数 */
  void *data;                   /* 类型专属数据：SE 是 handlerton*，AUDIT 是 st_mysql_audit* */
  MEM_ROOT mem_root;            /* 每插件独立内存池 */
  sys_var *system_vars;         /* sys_var_pluginvar 链头 */
  enum_plugin_load_option load_option;  /* OFF/ON/FORCE/FORCE_PLUS_PERMANENT */
};

struct st_plugin_dl {
  LEX_STRING dl;                /* 库文件名（不含路径，files_charset_info） */
  void *handle;                 /* dlopen 句柄 */
  struct st_mysql_plugin *plugins;
  int version;                  /* 库声明的 MYSQL_PLUGIN_INTERFACE_VERSION */
  uint ref_count;               /* 从该库加载的插件数，归零即 dlclose */
};
```

全局：`plugin_array`（顺序数组）、`plugin_dl_array`、`plugin_hash[12]`、`reap_needed`、`plugin_array_version`（每次 add/del 递增，`plugin_foreach_with_mask` 用它检测遍历期间数组被改），全部由 `LOCK_plugin` 保护。

### 加载一个 .so：`plugin_dl_add` 七道关

```
plugin_load_list / plugin_add
 └─ plugin_dl_add(dl, report, load_early)
      ├─ check_valid_path()：库名不允许含路径分隔符！必须位于 --plugin-dir 下
      │    （这是"防止加载任意 .so"的核心安全防线）
      ├─ plugin_dl_find() 已加载？→ ref_count++ 复用
      ├─ dlopen(dlpath, RTLD_NOW)            ← 立即解析符号，避免运行期缺符号
      ├─ dlsym("_mysql_plugin_interface_version_") → 版本双向校验
      ├─ 遍历 list_of_services[]：dlsym 老版 service 符号 → 校验 → 指针替换（见 03）
      ├─ dlsym("_mysql_plugin_declarations_")；
      │    版本不相等时按 _mysql_sizeof_struct_st_plugin_ 逐字段 memcpy 迁移
      ├─ PLUGIN_OPT_NO_INSTALL / ALLOW_EARLY 检查
      └─ plugin_dl_insert_or_reuse()
```

版本校验规则（`sql/sql_plugin.cc`）：`plugin_dl.version < min_plugin_interface_version`（`MYSQL_PLUGIN_INTERFACE_VERSION & ~0xFF`）或高字节大于当前 → 报 "plugin interface version mismatch"。

### 登记：`plugin_add` / `register_builtin`

`plugin_add()`：查重（`ER_UDF_EXISTS`）→ `plugin_dl_add` → 在库内声明数组里按名字找（大小写不敏感）→ **类型级 interface 校验**（`*(int*)plugin->info` 与 `min_plugin_info_interface_version[type]` / `cur_plugin_info_interface_version[type]` 双向比较，报 "API version for %s plugin is too different"）→ `test_plugin_options()` 解析 `--plugin-xxx` 选项 → `plugin_insert_or_reuse()` 入哈希。

`register_builtin()` 只做三件事：`ref_count = 0`、**`plugin_dl = nullptr`**、入数组入哈希。

`plugin_insert_or_reuse()` 的细节：**bootstrap 期间不复用空槽**（`get_server_state() != SERVER_BOOTING`）——注释解释：若早期插件（如 keyring）加载失败留下空槽，用户插件可能抢占该槽，导致之后 mandatory 插件（如 PFS）装载时数组序混乱。

### `INSTALL PLUGIN` 全链路

```
Sql_cmd_install_plugin::execute()                 sql/sql_plugin.cc
 └─ mysql_install_plugin(thd, name, dl)
      ├─ [权限] check_table_access(thd, INSERT_ACL, "mysql"."plugin")
      ├─ acquire_shared_global_read_lock + acquire_shared_backup_lock
      ├─ open_ltable("mysql"."plugin", TL_WRITE)   ← 必须在 LOCK_plugin 之前
      ├─ is_supported_system_table / System_table_intact::check 校验表结构
      ├─ mysql_audit_acquire_plugins(...)  ← 预取 audit 句柄防死锁（见 Misc）
      ├─ LOCK_plugin_install → LOCK_system_variables_hash → LOCK_plugin
      ├─ plugin_add(...) → dlopen + 登记 + sysvar 注册
      ├─ Disable_binlog_guard（INSTALL PLUGIN 不复制、禁 row-based binlog）
      ├─ table->file->ha_write_row()      <=== 持久化：往 mysql.plugin 插一行
      ├─ plugin_initialize(tmp)           <=== 调 init、注册状态变量、state=READY
      ├─ [I_S 插件] dd::info_schema::store_dynamic_plugin_I_S_metadata()
      └─ end_transaction(...)
```

持久化证据：8.0.39 里 `mysql.plugin` 仍是**真实 InnoDB 表**：

```sql
CREATE TABLE IF NOT EXISTS plugin (
  name varchar(64) DEFAULT '' NOT NULL,
  dl   varchar(128) DEFAULT '' NOT NULL,
  PRIMARY KEY (name)
) engine=InnoDB STATS_PERSISTENT=0 CHARACTER SET utf8mb3 ...
```

启动时 `plugin_load()` 用行迭代器逐行读出 `(name, dl)` 再 `plugin_add`（`REPORT_TO_LOG` 模式）。DD 侧只有 `sql/dd/impl/system_registry.cc` 里一行 `register_table("plugin", system)`——它只是"系统表"登记，不是 DD 对象。

### `UNINSTALL PLUGIN` 与延迟收割

```
Sql_cmd_uninstall_plugin::execute() → mysql_uninstall_plugin()
      ├─ [权限] check_table_access(thd, DELETE_ACL, ...)
      ├─ 三道拒绝检查：builtin → ER_PLUGIN_DELETE_BUILTIN
      │                 FORCE_PLUS_PERMANENT → ER_PLUGIN_IS_PERMANENT
      │                 PLUGIN_OPT_NO_UNINSTALL → ER_PLUGIN_NO_UNINSTALL
      ├─ state = PLUGIN_IS_DYING；【放锁】调用 plugin->check_uninstall()
      │    （它可能拿 MDL，必须在锁外执行）
      ├─ state = PLUGIN_IS_DELETED
      ├─ ref_count != 0 ? 仅警告 WARN_PLUGIN_BUSY : reap_needed = true
      ├─ reap_plugins()          ← ref_count==0 时真正 deinit + plugin_del
      ├─ key_copy → ha_index_read_idx_map → ha_delete_row()   <=== 删表行
      └─ end_transaction(...)
```

收割实现：

```c
static void reap_plugins() {
  if (!reap_needed) return;
  reap_needed = false;
  /* 第一遍：在锁内收集 ref_count==0 且 DELETED 的插件，标 DYING 防重入 */
  for (idx...) if (state == PLUGIN_IS_DELETED && !ref_count) { state = DYING; reap++ }
  mysql_mutex_unlock(&LOCK_plugin);            /* 放锁再 deinit，防死锁 */
  while ((plugin = *(--list))) plugin_deinitialize(plugin, true);
  /* 第二遍：重新拿三把锁，plugin_del()（卸 sysvar、出哈希、可能 dlclose） */
}
```

要点：deinit 在**锁外**执行——插件工作线程可能反向持有 plugin 锁；`plugin_del()` 只标 `PLUGIN_IS_FREED`，**数组槽位保留**（`plugin_array_version++`）。

### 插件的系统变量：`MYSQL_SYSVAR` → `sys_var_pluginvar` → THDVAR 布局

注册入口**不在** `plugin_initialize()` 而在 `test_plugin_options()`：遍历 `plugin->system_vars`，为每个变量 `new sys_var_pluginvar(...)` 并 `add_dynamic_system_variable_chain()`。`sys_var_pluginvar` 的关键成员 `orig_pluginvar_name`：变量名会被改成带插件前缀的**动态分配**名字（`plugin_name + "-" + var_name`，破折号转下划线、转小写），插件可反复装卸而 .so 不重载，卸载时必须把 `plugin_var->name` 恢复成硬编码原名——`restore_pluginvar_names()` 干这件事。

THD 变量（`PLUGIN_VAR_THDLOCAL`）的 per-session 存储是这套机制里最精巧的部分：

1. **bookmark**（`st_bookmark`，存 `plugin_mem_root`，**永不删除**）：key 格式 = `1 字节类型码 + "plugin_var" + '\0'`。查得到就复用 offset，查不到才分配新 offset。
2. **offset 分配**（`register_var()`）：按类型 size（bool/int/long/longlong/char*/double）**2 的幂对齐**，追加到 `global_system_variables.dynamic_variables_ptr` 尾部，总大小 64 字节向上取整（`(offset + size + 63) & ~63`），global 与 max 两份同步 realloc。
3. **会话懒拷贝**（`alloc_and_copy_thd_dynamic_variables()`）：会话首次访问某个 THDVAR 时（`intern_sys_var_ptr()` 发现 `offset > thd->variables.dynamic_variables_head`），把 `dynamic_variables_ptr` realloc 到全局大小，**只 memcpy 新增区间**；`PLUGIN_VAR_MEMALLOC` 的字符串变量还要 per-session `strdup`（挂在 `dynamic_variables_allocs` 链表）。
4. **取址**：`THDVAR(thd, name)` 展开为 `resolve(thd, offset)`——每种类型一个 `mysql_sys_var_XXX(thd, offset)` 函数，`construct_options()` 第一遍赋给 vtable 里的 resolve 指针；offset 初始为 -1，第二遍回填。

状态变量（`SHOW_VAR[]`）：`plugin_initialize()` 成功后 `add_status_vars()` 挂进全局 `all_status_vars`；`plugin_deinitialize()` 里 `remove_status_vars()` 摘除。

### 使用方之一：存储引擎

```
open_table → ha_resolve_by_name(thd, &name)
 └─ ha_resolve_by_name_raw → plugin_lock_by_name(thd, name, MYSQL_STORAGE_ENGINE_PLUGIN)
     └─ plugin_find_internal（plugin_hash[1] 查找）→ intern_plugin_lock（ref_count++）
 └─ plugin_data<handlerton*>(plugin)   ← st_plugin_int::data 即 handlerton
```

初始化：`ha_initialize_handlerton()` 分配 `handlerton`，**把 `hton` 而非 `st_plugin_int` 传给 `plugin->init`**；随后分配 `db_type`（动态区间找空位）与 `hton->slot`（`se_plugin_array` 槽位，同时记录 builtin 标记），`installed_htons[db_type] = hton`。名字解析支持历史别名（`se_names[]`）与 `DEFAULT` 伪名（`ha_default_plugin`/`ha_default_temp_plugin`）。

### 使用方之二：审计

`initialize_audit_plugin()`：校验 `class_mask` 与 `event_notify`，把插件的 mask 并入全局 `mysql_global_audit_mask`。分发靠 `plugin_foreach(current_thd, plugins_dispatch, MYSQL_AUDIT_PLUGIN, ...)`：对每个 READY 的审计插件调 `event_notify`。8.0.39 中审计**仍是插件接口、未迁移组件**（`include/mysql/components/services/` 下无 `event_tracking_*`；只有 `components/audit_api_message_emit` 这个供审计插件发消息的组件）。

### 关闭：`plugin_shutdown` 多轮收敛

`clean_up()` 里顺序：`delegates_shutdown()` → `plugin_shutdown()` → `gtid_server_cleanup()`（必须在插件关闭后）。另有 `memcached_shutdown()` 更早单独关 `daemon_memcached`（必须在 binlog 之前）。

`plugin_shutdown()` 是 **while 循环而非两遍**：每轮把上一轮标 DELETED 的收割掉，再把所有 READY 的标 DELETED——**binlog 例外，最后一轮才标**（其它引擎提交时还要写 binlog）；若某轮无进展（剩下都因 ref_count 不归零），则 `unlock_variables()` 释放 `global_system_variables` 持有的默认引擎句柄再试。仍收敛不了时进入**强制关停**：`ref_check=false` 直接 deinit，结束后统一查 ref_count 并报 `ER_PLUGIN_HAS_NONZERO_REFCOUNT_AFTER_SHUTDOWN`。最后 `free_plugin_mem()` 逐个 `dlclose`。

---

## 相关的系统变量/命令行选项

| 名字 | 默认值 | 说明 |
|------|--------|------|
| `plugin_dir` | 构建时注入（如 `/usr/lib/mysql/plugin`） | **READ_ONLY NON_PERSIST** 系统变量（`Sys_var_charptr Sys_plugin_dir`）；决定 dlopen 搜索目录 |
| `--plugin-load` | 空 | 命令行选项：清空列表后加入；语法 `name=lib;name2=lib2;lib3`（只写 lib 则加载库内全部插件） |
| `--plugin-load-add` | 空 | 追加（可多次），不覆盖 |
| `--early-plugin-load` | 空 | 早期加载列表，插件须声明 `PLUGIN_OPT_ALLOW_EARLY` |
| `--skip-innodb` 等 | — | `test_plugin_options()` 为每个插件生成 `--plugin-<name>` / `--<name>` 开关，值为 `OFF/ON/FORCE/FORCE_PLUS_PERMANENT`；FORCE 插件不生成开关（无法关闭） |

---

## Misc

### `MYSQL_UDF_PLUGIN` 是死类型

类型 0 存在但从未启用：分派表两行都是 `nullptr`，注释明写 "UDF: not implemented"。UDF 现在走 `udf_registration` / `udf_registration_aggregate` **组件服务**注册（`sql/udf_registration_imp.h` + `sql/sql_udf.cc` 实现），`CREATE FUNCTION` 持久化到 `mysql.func` 表，与插件体系无耦合。

### `[UN]INSTALL PLUGIN` 里的 audit 预取 hack

`mysql_install_plugin` 长时间持有 `LOCK_plugin`，而审计分派 `plugin_foreach` 也要拿这把锁——同线程二次加锁必死锁。解法：先 `mysql_audit_acquire_plugins(thd, MYSQL_AUDIT_GENERAL_CLASS, ...)` 把 audit 插件句柄缓存进 THD，随后的审计事件直接走缓存。

### 升级场景的"延迟插件"

`delayed_plugins[] = {"audit_log", "mysql_firewall"}`（这两个是商业版插件）：DD 升级期间保持 `PLUGIN_IS_WAITING_FOR_UPGRADE`（`SHOW PLUGINS` 显示 INACTIVE），升级完成后 `plugin_initialize_delayed_after_upgrade()` 再初始化。

### 容易混淆的名字

- `PLUGIN_FORCE`（`enum_plugin_load_option` 成员，强制加载级别）≠ `PLUGIN_OPT_*`（描述符 flags）。不存在 `PLUGIN_OPT_MANDATORY` 这个宏——mandatory 由"是否在 `mysql_mandatory_plugins[]` 数组"决定。
- 插件体系里**没有** maturity（成熟度）概念（8.0.39 全库 grep 无 `plugin_maturity`）；license 字段只用于 `SHOW PLUGINS` 展示，加载期不检查。
- `plugin_ref` 在 NDEBUG/非 NDEBUG 下类型不同（`st_plugin_int*` vs `st_plugin_int**`），宏 `plugin_ref_to_int` / `plugin_int_to_ref` 屏蔽差异。
- 老版 plugin service（`include/mysql/service_*.h` + `list_of_services[]`）与组件 service 是两套机制，辨析见 [`service.md`](service.md)。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Extending MySQL → MySQL Services for Plugins*（老版 plugin service 的使用方式）
- *MySQL 8.0 Reference Manual → Installing and Uninstalling Plugins*
- WorkLog: [WL#4102 Service registry and component infrastructure](https://dev.mysql.com/worklog/task/?id=4102)（组件体系对插件缺陷的总结与替代方案）

**内核月报 / 技术文章**
- [MySQL · 引擎特性 · clone_plugin](http://mysql.taobao.org/monthly/2019/08/05/)（kongzhi，2019-08）：clone 插件的快照一致性三段式与远程协议源码走读；插件/组件框架只是顺带引用，未深入框架本身
- [MySQL · 引擎特性 · 初探 Clone Plugin](http://mysql.taobao.org/monthly/2019/09/02/)（weixiang，2019-09）：clone 功能初探（INIT→FILE COPY→PAGE COPY→REDO COPY→Done），原理级介绍

> 说明：月报自 2014 年至今**没有**插件/组件/服务/ABI 框架本身的专题文章（经多轮检索核实），以上两篇是与插件最相关的落地功能文章。

**源码文档页**
- `sql/sql_plugin.cc` 顶部 `@page page_ext_plugin_svc_anathomy`（老版 plugin service 的解剖）
- `components/libminchassis/dynamic_loader.cc` 的 `@page page_components_layering_plugins`（插件与组件的关系定位）

**相关文档**
- ABI 契约专题（三层版本校验/旧结构迁移/符号可见性）见 [`abi.md`](abi.md)
- 组件基础设施见 [`component.md`](component.md)
- 服务（含老版 plugin service 与 registry 内部）见 [`service.md`](service.md)
