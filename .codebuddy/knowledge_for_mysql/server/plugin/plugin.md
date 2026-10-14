# 插件（Plugin）体系深度解析

> 基于 MySQL 8.0.39 源码，涵盖插件的 ABI 契约、类型系统、加载/安装/卸载全链路、引用计数与延迟收割、插件系统变量。
>
> **边界**：本篇讲插件体系（老一代扩展机制）；组件基础设施（Component）见 [`component.md`](component.md)，服务（Service，含 registry 内部实现与老版 plugin service）见 [`service.md`](service.md)。
>
> ★ **认证插件不在这里单独成篇**：`caching_sha2_password` / `mysql_native_password` 的算法与握手协议属认证主题，详写于 [`../auth/authentication.md`](../auth/authentication.md)；本篇只在[「认证插件：从框架视角看这个元老类型」](#认证插件从框架视角看这个元老类型)讲它们的**框架事实**。

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

**第零条（第一性原理）：插件解决的是"服务器在固定位置需要一段可替换的代码"。**

一句话概括设计原点：**调用点驱动，而非能力驱动**。服务器写代码时就知道"打开表要解析存储引擎、认证要调认证插件、写 binlog 前要通知审计"——这些**调用点是服务器内生的、固定的**，插件要做的只是"把自己填进这个坑位"。因此契约形态必然是"类型化角色"（我是什么 → 我能被谁调），而不是"能力注册"（我提供什么 → 谁都可以来查）。

三条替代路线，插件体系为什么没走：

| 路线 | 为什么插件体系没选 | 后来怎样 |
|---|---|---|
| 通用服务注册（能力市场） | 5.1 时代服务器尚未服务化；调用点若走注册表查找会有额外开销 | 8.0 由组件体系补上（WL#4102），见 [`component.md`](component.md) |
| 进程外插件 + IPC | 引擎接口每行调用，跨进程不可接受 | 从未采用，见 [`abi.md` 设计思想 ②](abi.md) |
| 依赖声明 + 自动排序 | 插件不声明依赖，服务器无从排序 | 于是顺序只能硬编码成三段（early / builtin / dynamic），见「主链路：启动加载」 |

代价与收益是同一枚硬币：

- **代价**：类型集合封闭（12 个，第三方无法新增）、初始化顺序硬编码、插件之间互不可见（`RTLD_LOCAL`，见 [`abi.md` 实验 5](abi.md)）、支持新扩展场景必须改服务器；
- **收益**：调用点拿到强类型 vtable 直接调用——**零查找、零 acquire/release、零注册表**；部署极简（一个 `.so` + `mysql.plugin` 一行）。

这也解释了 8.0 的整体格局：**插件没有"错"，它只是为"调用点固定且高频"的场景优化到了极致**；"调用点不固定、需要互相组合"的场景只能由组件体系解。两代机制并存不是历史包袱，而是**两种调用形态各得其所**（见 [`abi.md` 设计思想 ⑦](abi.md) 的调用形态三分层）。

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
| AUTHENTICATION（类型号 `7`） | `st_mysql_auth` | 无包装，直调 `plugin->init`；vtable 七成员与 `CR_*` 返回码全表见 [`../auth/authentication.md`](../auth/authentication.md) §7.1 |
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

### 插件状态机：七态与转换（难点）

`st_plugin_int::state` 是**单 bit** 状态机，七个本征态（`sql/sql_plugin.h`）：`UNINITIALIZED`(4)、`DISABLED`(32)、`READY`(8)、`DYING`(16)、`DELETED`(2)、`FREED`(1)、`WAITING_FOR_UPGRADE`(64)。用 bitmap 而非枚举，源码注释说得很清楚——方便"是否属于某几个状态之一"的判断（`intern_plugin_lock` 就靠 `state & (READY|UNINITIALIZED)` 一行判断能否加锁）。

```
              plugin_add / register_builtin
                        │
        test_plugin_options 失败 ──► DISABLED   ← 终态：被 --skip-xxx 关闭，只能重启恢复
                        │成功
                        ▼
     UNINITIALIZED ──(DD 升级期)──► WAITING_FOR_UPGRADE ──(升级完成)──► 回到 UNINITIALIZED
                        │ plugin_initialize() 成功
                        ▼
                      READY
                        │ UNINSTALL / shutdown
                        ▼
          DYING ──(check_uninstall 失败)──► 回到 READY，报 ER_PLUGIN_CANNOT_BE_UNINSTALLED
            │成功
            ▼
         DELETED ──(ref_count 归零后的某次 unlock 触发 reap)──► DYING ──► FREED
                                                                          │
                                                              槽位可被 plugin_insert_or_reuse 复用
```

关键转换的源码落点（已逐条核实）：

- `plugin_add()`：成功置 `UNINITIALIZED`，`test_plugin_options()` 失败置 `DISABLED`，中途失败置 `FREED`；
- `plugin_initialize()`：断言进入时是 `UNINITIALIZED`，init 成功后置 `READY`；
- `mysql_uninstall_plugin()`：`READY → DYING`（**放锁后**才调 `check_uninstall`，因为它可能拿 MDL）→ 失败回 `READY` → 成功置 `DELETED`；
- `reap_plugins()`：`DELETED && ref_count == 0` → 置 `DYING`（防并发重复收割）→ 锁外 deinit → `plugin_del()` 置 `FREED`；
- `plugin_shutdown()`：`READY → DELETED`（binlog 最后一轮才标），残留者强制阶段改 `DYING` 后强行 deinit。

**为什么"删除 + 收割"必须拆成 DELETED → DYING → FREED 三步？** 三个状态各管一件事，缺一不可：

1. `DELETED` 是**对外承诺**——从此 `intern_plugin_lock()` 对新请求返回 `nullptr`，插件不再接受新引用（这是"卸载安全"的一半）；
2. `DYING` 是**内部互斥**——收割是由"最后一个 `plugin_unlock`"触发的，任意线程都可能执行，必须有状态防止两个线程收割同一插件；
3. `FREED` 是**资源状态**——槽位可复用，但 `plugin_array` **不缩容**（保证遍历中的指针稳定，`plugin_foreach` 依赖这一点）。

把"不再接受新引用"与"物理回收"解耦，是延迟收割能成立的前提——这一点与组件体系的"引用未归零就拒绝卸载"恰好是同一问题的两种解法（见 [`component.md`](component.md) 的卸载链）。

### `plugin_foreach` 的 mask 语义（反直觉点）

`plugin_foreach_with_mask()` 的 mask 语义与直觉相反——源码第一行就把它取反了：

```c
state_mask = ~state_mask;  // do it only once
...
plugins[idx] = !(plugin->state & state_mask) ? plugin : nullptr;
```

因为 `state` 同一时刻只有一个 bit，两次取反后等价于"**只选出 state 恰好等于 mask 的那些插件**"：

| 调用 | 传入 mask | 实际选出 |
|---|---|---|
| `plugin_foreach(...)`（宏展开为 `PLUGIN_IS_READY`） | READY | 只有 READY |
| `fill_plugins()` 传 `~PLUGIN_IS_FREED` | FREED | **除 FREED 外全部**（含 DISABLED、DELETING） |

`plugin_foreach()` 默认只看"就绪"的插件；`SHOW PLUGINS` 要看"除已释放槽位外的一切"（所以传取反值）。搞反会得到两类 bug：插件明明存在却遍历不到；把正在删除的插件当可用。

另外两个易忽略的细节：

- **存储引擎类型下 binlog 强制最先调用**：注释说明"GTID 需要先由 binlog 生成"——加载器先找到 binlog 的槽位单独调一次，再遍历其余；
- **回调期间不持锁**，靠 `plugin_array_version` 检测数组被改动：版本变了就在锁内把"状态已不匹配的槽位"剔除。这是"快照 + 版本校验"而非全程持锁的折中——回调可能很慢，甚至会反向拿 `LOCK_plugin`（这正是 INSTALL PLUGIN 要预取 audit 句柄的原因，见 Misc）。

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

### `plugin_initialize()`：为什么 init 在锁外执行

登记（`plugin_add` / `register_builtin`）只是把插件放进哈希表、建好系统变量；真正的初始化由 `plugin_initialize()` 完成。这段代码短，但每一行的取舍都很硬：

```c
mysql_mutex_assert_owner(&LOCK_plugin);
uint state = plugin->state;
assert(state == PLUGIN_IS_UNINITIALIZED);

mysql_mutex_unlock(&LOCK_plugin);                        /* ① 放锁 */
mysql_rwlock_unlock(&LOCK_system_variables_hash);        /* ② 也放 sysvar 锁 */

if (plugin_type_initialize[plugin->plugin->type]) {      /* ③ 类型包装优先 */
  if ((*plugin_type_initialize[plugin->plugin->type])(plugin)) { ... goto err; }
  /* FIXME: Need better solution to transfer the callback function
     array to memcached */
  if (strcmp(plugin->name.str, "InnoDB") == 0) {
    innodb_callback_data = ((handlerton *)plugin->data)->data;   /* ④ 隐藏耦合 */
  }
} else if (plugin->plugin->init) {
  if (strcmp(plugin->name.str, "daemon_memcached") == 0) {
    plugin->data = innodb_callback_data;                  /* ④ 接过来 */
  }
  if (plugin->plugin->init(plugin)) { ... goto err; }     /* ⑤ 直调插件 init */
}
state = PLUGIN_IS_READY;                                  /* ⑥ 先标 READY */

if (plugin->plugin->status_vars) {
  if (add_status_vars(plugin->plugin->status_vars)) goto err;   /* ⑦ 状态变量 */
}

/* set the plugin attribute of plugin's sys vars so they are pointing
   to the active plugin */
if (plugin->system_vars) {                                /* ⑧ sysvar 回填 */
  sys_var_pluginvar *var = plugin->system_vars->cast_pluginvar();
  for (;;) { var->plugin = plugin; if (!var->next) break; var = var->next->cast_pluginvar(); }
}
ret = 0;
err:
mysql_rwlock_wrlock(&LOCK_system_variables_hash);
mysql_mutex_lock(&LOCK_plugin);
plugin->state = state;                                    /* ⑨ 重新加锁写回 */
```

逐条解释其中的"为什么"：

**①② init 必须在锁外执行。** 插件的 init 可能很长（InnoDB 启动要恢复 redo、初始化 buffer pool），更关键的是它可能**反向获取这两把锁**——例如插件自己调 `plugin_lock_by_name()`（拿 `LOCK_plugin`）或注册系统变量（拿 `LOCK_system_variables_hash`）。若持锁调用就是自死锁。代价：init 期间别的线程能观察到 `UNINITIALIZED` 状态（所以 `intern_plugin_lock` 允许锁 `UNINITIALIZED` 状态的插件，见状态机一节）。

**③ 类型包装优先。** 只有 SE / I_S / AUDIT 三类有内核包装函数（要先分配 `handlerton`、注册 schema table、并入全局 audit mask），其余类型直接调 `plugin->init`。

**⑥→⑦ 顺序**：先置 `READY` 再注册状态变量——因为 `add_status_vars` 可能触发 `SHOW STATUS`，而调用方期望插件已就绪；失败则 `goto err`，此时 `state` 仍是 `READY`（局部变量），会被写回，上层看到的是"初始化失败但状态是 READY"，由调用者决定后续（启动阶段 mandatory 插件失败则拒绝启动）。

**⑧ sysvar 回填的意义**：系统变量在更早的 `test_plugin_options()` 阶段就已注册（那时插件还没 init），此时才把 `sys_var_pluginvar::plugin` 指向这个活跃实例——`SHOW VARIABLES`、`SET`、以及卸载时的 `plugin_del` 都靠这个指针定位"这个变量属于哪个插件"。

**④ 一个活标本：`innodb_callback_data` 的名字特判。** 这是全篇最能说明"插件只能对服务器说话"代价的例子：`daemon_memcached` 插件需要 InnoDB 的回调函数数组才能直接访问 InnoDB 行，但**插件之间没有通道**（没有服务、没有 registry），于是只能由服务器在 `plugin_initialize()` 里硬编码两个 `strcmp` 特判——先记住 InnoDB 的 `handlerton::data`，等 memcached 插件初始化时再塞给它。源码自己贴了 `FIXME: Need better solution`。这类"按名字特判"的胶水，正是 WL#4102 缺陷 1（插件只能与服务器对话）在真实代码里的样子；组件体系出现后，同类需求直接走服务，不再需要这种耦合。

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

THD 变量（`PLUGIN_VAR_THDLOCAL`）的 per-session 存储是这套机制里最精巧的部分——**插件变量的个数与类型在编译期未知，而每个会话都要有自己的一份拷贝**，且插件可以在运行期装卸。MySQL 的解法是把所有 THDVAR 打包进一块 per-session 的"动态变量区"，用 **offset + 版本号**管理。

**第一步：bookmark 分配 offset（`register_var()` 核心片段）**

```c
if (!(result = find_bookmark(nullptr, varname + 1, flags))) {
  result = (st_bookmark *)plugin_mem_root.Alloc(sizeof(st_bookmark) + length - 1);
  varname[0] = flags & PLUGIN_VAR_TYPEMASK;      /* key 首字节 = 类型码 */
  memcpy(result->key, varname, length);
  result->offset = -1;

  assert(size && !(size & (size - 1)));          /* size 必须是 2 的幂 */

  offset = global_system_variables.dynamic_variables_size;
  offset = (offset + size - 1) & ~(size - 1);    /* 按自身 size 对齐 */
  result->offset = (int)offset;

  new_size = (offset + size + 63) & ~63;         /* 总量 64 字节向上取整 */

  if (new_size > global_variables_dynamic_size) {
    global_system_variables.dynamic_variables_ptr = (char *)my_realloc(..., new_size, ...);
    max_system_variables.dynamic_variables_ptr    = (char *)my_realloc(..., new_size, ...);
    /* 只把新增区间清零——字符串变量必须保证初值为 NULL */
    memset(global_system_variables.dynamic_variables_ptr + global_variables_dynamic_size,
           0, new_size - global_variables_dynamic_size);
    memset(max_system_variables.dynamic_variables_ptr + global_variables_dynamic_size,
           0, new_size - global_variables_dynamic_size);
    global_variables_dynamic_size = new_size;
  }

  global_system_variables.dynamic_variables_head = offset;
  global_system_variables.dynamic_variables_size = offset + size;
  global_system_variables.dynamic_variables_version++;   /* 版本号 +1 */
  result->version = global_system_variables.dynamic_variables_version;
}
```

逐段拆解其中的设计决策：

- **bookmark 永不删除**（`st_bookmark` 分配在 `plugin_mem_root`，注释明说 "this structure is never removed from the hash"）：offset 一旦分配就永久保留给"插件名 + 变量名"这个 key。理由在本文开头的设计权衡里——**会话的生命周期可能长于插件**：若卸载重装时重新分配 offset，老会话的 `dynamic_variables_ptr` 里那个位置的含义就变了，直接越界读写。
- **两次对齐**：先按变量自身 size 做 2 的幂对齐（保证 `int`/`double`/指针落在自然边界），再把总大小按 64 字节向上取整（减少 realloc 次数、对齐 cache line）。
- **只清零新增区间**，而不是整块：因为现有变量的值必须保留（可能已有会话在用）。
- **版本递增**（`dynamic_variables_version`）：会话侧用它判断"我这块内存落后了几个时代"，决定要不要补拷贝。

**第二步：会话懒拷贝（`alloc_and_copy_thd_dynamic_variables()`）**

```c
thd->variables.dynamic_variables_ptr = (char *)my_realloc(
    key_memory_THD_variables, thd->variables.dynamic_variables_ptr,
    global_variables_dynamic_size, MYF(MY_WME | MY_FAE | MY_ALLOW_ZERO_PTR));

/* 只拷新增部分：从 thd 已有的 size 到 global 的 size */
memcpy(thd->variables.dynamic_variables_ptr + thd->variables.dynamic_variables_size,
       global_system_variables.dynamic_variables_ptr + thd->variables.dynamic_variables_size,
       global_system_variables.dynamic_variables_size - thd->variables.dynamic_variables_size);

/* 新增的 string+MEMALLOC 变量要 per-session strdup 一份 */
for (const auto &key_and_value : *malloced_string_type_sysvars_bookmark_hash) {
  ...
  thdvar = (char **)(thd->variables.dynamic_variables_ptr + varoff);
  *thdvar = nullptr;
  plugin_var_memalloc_session_update(thd, nullptr, thdvar, *sysvar);
}
thd->variables.dynamic_variables_version = global_system_variables.dynamic_variables_version;
```

**"懒"体现在触发时机**：`intern_sys_var_ptr(thd, offset)` 发现 `offset > thd->variables.dynamic_variables_head` 才触发拷贝（且 `current_thd != thd` 时不触发，避免跨线程改别人的内存——直接返回全局值）。绝大多数会话从不访问插件变量，就永远不用付这份内存。

**第三步：取址（为什么 `THDVAR()` 是个宏 + 函数指针）**

`THDVAR(thd, name)` 展开为 `MYSQL_SYSVAR_NAME(name).resolve(thd, offset)`——每种类型一个 `mysql_sys_var_XXX(thd, offset)` 函数，`construct_options()` 第一遍遍历时赋给 vtable 的 `resolve` 指针（offset 初值是 -1，第二遍回填）。用函数指针而非直接算地址，是为了让"取址"这一步能顺带完成**懒拷贝**——`resolve` 内部调的就是 `intern_sys_var_ptr()`。

一个容易忽略的对称设计：`PLUGIN_VAR_MEMALLOC` 的字符串变量在会话里要 `strdup`，这些堆块挂在 `System_variables::dynamic_variables_allocs` 链表上，会话结束时由 `plugin_var_memalloc_free()` 统一释放——**per-session 内存的生命周期由 THD 管理，而 offset 布局由全局 bookmark 管理**，两套生命周期解耦才能支撑插件反复装卸。

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

`initialize_audit_plugin()`：校验 `class_mask` 与 `event_notify`，把插件的 mask 并入全局 `mysql_global_audit_mask`（卸载时 `finalize_audit_plugin` 再用 `plugin_foreach` **重算**一遍全局 mask——不是做减法，因为多个插件的 mask 可能重叠）。

真正的分发函数很短，但每个字节都有用意（`sql/sql_audit.cc`）：

```c
static int plugins_dispatch(THD *thd, plugin_ref plugin, void *arg) {
  const struct st_mysql_event_generic *event_generic =
      (const struct st_mysql_event_generic *)arg;
  unsigned long subclass = static_cast<unsigned long>(
      *static_cast<const int *>(event_generic->event));
  st_mysql_audit *data = plugin_data<st_mysql_audit *>(plugin);

  /* Check to see if the plugin is interested in this event */
  if (check_audit_mask(data->class_mask[event_generic->event_class], subclass))
    return 0;

  /* Actually notify the plugin */
  return data->event_notify(thd, event_generic->event_class,
                            event_generic->event);
}
```

三点设计：

- **两级过滤**：全局 `mysql_global_audit_mask` 先挡掉"没有任何插件关心"的事件（省掉一次遍历），再在每个插件上用 `class_mask[event_class]` + subclass 位图精筛——审计事件在热路径上（每条语句、每次连接），两级过滤是必要的性能设计。
- **`plugin_data<st_mysql_audit *>(plugin)`**：从 `plugin_ref` 直接取 `st_plugin_int::data`（初始化时 `plugin->data = plugin->plugin->info`），**无哈希查找**——又一处"热路径不走查找"。
- **遍历用 `plugin_foreach`**：只遍历 READY 插件，且遍历期间不持 `LOCK_plugin`（见 `plugin_foreach` 小节）——所以审计回调里可以做慢操作，但这也是 INSTALL PLUGIN 必须预取 audit 句柄的原因。

8.0.39 中审计**仍是插件接口、未迁移组件**（`include/mysql/components/services/` 下无 `event_tracking_*`；只有 `components/audit_api_message_emit` 这个供审计插件发消息的组件）。

### `SHOW PLUGINS` / `I_S.PLUGINS`：状态如何变成字符串

两者共用 `fill_plugins()` → `plugin_foreach_with_mask(~PLUGIN_IS_FREED)` → `show_plugins()`。后者把内部状态机映射成用户可见的 `PLUGIN_STATUS`（`sql/sql_show.cc`，原文）：

```c
switch (plugin_state(plugin)) {
  /* case PLUGIN_IS_FREED: does not happen */
  case PLUGIN_IS_DELETED:                                  → "DELETED"
  case PLUGIN_IS_UNINITIALIZED:
  case PLUGIN_IS_WAITING_FOR_UPGRADE:                      → "INACTIVE"
  case PLUGIN_IS_READY:                                    → "ACTIVE"
  case PLUGIN_IS_DYING:                                    → "DELETING"
  case PLUGIN_IS_DISABLED:                                 → "DISABLED"
}
table->field[3]->store(plugin_type_names[plug->type].str, ...);              /* 类型名 */
table->field[4]->store(make_version_string(version_buf, sizeof(version_buf),
                                           *(uint *)plug->info), ...);        /* 类型 ABI 版本 */
```

两个细节：

- **`FREED` 永远不会出现**——源码注释直说 "does not happen"：`FREED` 的槽位已被 `~PLUGIN_IS_FREED` 这个 mask 排除在遍历之外（见 `plugin_foreach` 小节的 mask 语义）。
- **`WAITING_FOR_UPGRADE` 与 `UNINITIALIZED` 都显示 `INACTIVE`**——所以升级期间"延迟插件"看起来就是没初始化，运维侧看不出区别，这正是"升级场景延迟插件"那条 Misc 的由来。
- 第 5 列 `PLUGIN_TYPE_VERSION` 打印的是 `*(uint *)plug->info`，即**类型 vtable 首字段**——又一次印证"首字段即版本号"这条 ABI 约定（详见 [`abi.md`](abi.md)）。

### 关闭：`plugin_shutdown` 多轮收敛

`clean_up()` 里顺序：`delegates_shutdown()` → `plugin_shutdown()` → `gtid_server_cleanup()`（必须在插件关闭后）。另有 `memcached_shutdown()` 更早单独关 `daemon_memcached`（必须在 binlog 之前）。

`plugin_shutdown()` 是 **while 循环而非两遍**：每轮把上一轮标 DELETED 的收割掉，再把所有 READY 的标 DELETED——**binlog 例外，最后一轮才标**（其它引擎提交时还要写 binlog）；若某轮无进展（剩下都因 ref_count 不归零），则 `unlock_variables()` 释放 `global_system_variables` 持有的默认引擎句柄再试。仍收敛不了时进入**强制关停**：`ref_check=false` 直接 deinit，结束后统一查 ref_count 并报 `ER_PLUGIN_HAS_NONZERO_REFCOUNT_AFTER_SHUTDOWN`。最后 `free_plugin_mem()` 逐个 `dlclose`。

---

### 8.0.39 的 builtin 插件全清单

`sql/sql_builtin.cc.in` 是模板，cmake 生成 `sql/sql_builtin.cc`（本分支的生成物见 `build_vscode_8039/sql/sql_builtin.cc`）。它产出两个数组，**数组身份决定命运**：

```c
struct st_mysql_plugin *mysql_mandatory_plugins[] = {
  builtin_binlog_plugin, builtin_mysql_password_plugin,
  builtin_caching_sha2_password_plugin, builtin_daemon_keyring_proxy_plugin,
  builtin_csv_plugin, builtin_heap_plugin, builtin_innobase_plugin,
  builtin_myisam_plugin, builtin_myisammrg_plugin, builtin_perfschema_plugin,
  builtin_temptable_plugin, 0};

struct st_mysql_plugin *mysql_optional_plugins[] = {
  builtin_archive_plugin, builtin_blackhole_plugin, builtin_federated_plugin,
  builtin_ndbcluster_plugin, builtin_ngram_parser_plugin, builtin_mysqlx_plugin, 0};
```

| 分组 | 插件 | 说明 |
|---|---|---|
| **mandatory** | `binlog` | 伪存储引擎（代表 binlog 参与事务） |
| | `mysql_native_password` **+** `sha256_password`（同属一个 `mysql_declare_plugin(mysql_password)` 块，在数组里表现为**单个指针** `builtin_mysql_password_plugin`）、`caching_sha2_password` | 认证插件——注意是 **3 个插件、2 个数组条目**；契约与 `CR_*` 返回码见 [`../auth/authentication.md`](../auth/authentication.md) §7.1 |
| | `daemon_keyring_proxy` | 把 keyring 插件包装成组件服务的代理（见 [`keyring.md`](keyring.md)） |
| | `csv`、`heap`、`innobase`、`myisam`、`myisammrg`、`temptable` | 存储引擎，其中 MyISAM/InnoDB/CSV 在 `plugin_register_builtin_and_init_core_se` 阶段就被提前初始化 |
| | `perfschema` | PFS（源码里还额外把它硬编码成 `PLUGIN_FORCE`） |
| **optional** | `archive`、`blackhole`、`federated`、`ndbcluster` | 可选存储引擎 |
| | `ngram_parser` | 全文解析器插件（中日文分词） |
| | **`mysqlx`** | X Plugin——注意它是 **builtin**，不是 `.so`（见 [`mysqlx.md`](mysqlx.md)） |

三个要点：

- **mandatory ≠ "不能被关"**，而是"**初始化失败就拒绝启动**"（`reaped_mandatory_plugin` → 服务器退出）；它们同时也**不生成 `--plugin-<name>` 开关**（`construct_options()` 对 `PLUGIN_FORCE` 不生成），所以用户关不掉。
- **optional 可以被关**：`--skip-archive`、`--skip-mysqlx` 这类开关由 `test_plugin_options()` 为每个插件生成；关掉的插件状态是 `PLUGIN_IS_DISABLED`（状态机里的终态，只能重启恢复）。
- **builtin 插件一律不参与引用计数**（`plugin_dl == nullptr`），也**不能 `UNINSTALL PLUGIN`**（报 `ER_PLUGIN_DELETE_BUILTIN`）——这是 builtin 与 dynamic 最关键的行为差异。

### 认证插件：从框架视角看这个元老类型

认证插件的算法、握手协议与凭据校验**不在本篇**——那是认证主题，见 [`../auth/authentication.md`](../auth/authentication.md)（§3 逐个插件深剖、§7.1 契约）。本节只讲**从插件框架角度才能看清的 5 个事实**，它们是框架的通用特性，用认证插件作例子最直观。

**① 类型号 7：`MYSQL_AUTHENTICATION_PLUGIN` 是 MySQL 第一个"非引擎"插件类型。**

5.5 的可插拔认证（pluggable authentication）是插件体系从"只给存储引擎用"走向通用的起点（见 [概述·版本演进](#概述)）。在 12 个类型里它排第 7，夹在 `MYSQL_REPLICATION_PLUGIN`(6) 与 `MYSQL_VALIDATE_PASSWORD_PLUGIN`(8) 之间。

**② 一个 `mysql_declare_plugin` 块可以含**多个**描述符——不是一块一插件。**

这是最容易想错的地方。宏展开后就是一个 `st_mysql_plugin[]` 数组，长度不限：

```c
/* sql/auth/sql_authentication.cc */
mysql_declare_plugin(mysql_password){
    { MYSQL_AUTHENTICATION_PLUGIN, &mysql_native_password_handler, "mysql_native_password", ... },
    { MYSQL_AUTHENTICATION_PLUGIN, &sha256_password_handler,       "sha256_password",       ... },
    ...
} mysql_declare_plugin_end;
```

后果直接落到 builtin 数组上：上面这**一个块**在 `sql_builtin.cc` 里只对应**一个指针** `builtin_mysql_password_plugin`，却注册了 **2 个插件**。所以"3 个内置认证插件"与"强制数组里 2 个认证条目"是同一件事的两种数法（见上表）。

**③ 一个 decl 块里可以混装**不同类型**的插件。**

```c
/* sql/auth/sha2_password.cc —— 同一个块：一个认证插件 + 一个审计插件 */
mysql_declare_plugin(caching_sha2_password){
    { MYSQL_AUTHENTICATION_PLUGIN, &caching_sha2_auth_handler, "caching_sha2_password", ... },
    { MYSQL_AUDIT_PLUGIN,          &sha2_cache_cleaner,        "sha2_cache_cleaner",    ... },
} mysql_declare_plugin_end;
```

`sha2_cache_cleaner` 的唯一职责是**监听认证类审计事件去失效密码缓存**（见 [`mysqlx.md`](mysqlx.md) 里 X Plugin 的 `mysqlx_cache_cleaner` 同款套路）。两个插件被**捆在同一个库、同一个块**里，因为：

- 它们必须**同生共死**——只有认证插件在工作时，缓存才需要被清理；
- 而插件之间**没有依赖声明机制**（那是组件 `requires` 的能力），只能靠"塞进同一个块 + 声明顺序"来保证初始化次序。

★ 这是"**插件模型的表达能力不足以描述插件间关系**"的实证——组件用 `REQUIRES_SERVICE` 一行说清的事，插件只能用"物理打包在一起"来近似。

**④ 认证插件是唯一在"每条连接热路径"上走引用计数的类型。**

```c
static int do_auth_once(THD *thd, const LEX_CSTRING &auth_plugin_name, MPVIO_EXT *mpvio) {
  bool unlock_plugin = false;
  plugin_ref plugin =
      g_cached_authentication_plugins->get_cached_plugin_ref(&auth_plugin_name);   /* ① 先查缓存 */

  if (!plugin) {                                                                  /* ② 未命中才 lock */
    if ((plugin = my_plugin_lock_by_name(thd, auth_plugin_name,
                                         MYSQL_AUTHENTICATION_PLUGIN)))
      unlock_plugin = true;                                                       /* ③ 记住"这次是我锁的" */
  }
  ...
  res = auth->authenticate_user(mpvio, &mpvio->auth_info);
  if (unlock_plugin) plugin_unlock(thd, plugin);                                  /* ④ 只还自己锁的 */
}
```

三条框架语义浓缩在这 15 行里：

- **`unlock_plugin` 这个 bool 就是"借用不归还"原则的代码化**：内置认证插件的 `plugin_ref` 由 `Cached_authentication_plugins` 在 `acl_init()` 里**启动时一次性锁住、整个进程生命周期持有**（`sql_authentication.cc:1039`），`do_auth_once()` 只是**借用**——此时若调 `plugin_unlock` 会破坏计数（详见认证篇对这张缓存表的剖析）；
- **非内置插件（如 `auth_socket`）每次登录都要真 `lock` + `unlock`**，即每条连接两次 `LOCK_plugin` 竞争。这正是 `Cached_authentication_plugins` 存在的理由——**热路径上能不碰全局锁就不碰**（与 [`abi.md`](abi.md) 设计思想 ⑦「调用形态三分层」是同一条原则）；
- 认证插件因此成为**引用计数机制压力最大**的使用方：一次连接风暴就是每秒数万次 `plugin_lock`。

**⑤ `st_plugin_int::data`：认证插件**不用**它，SE 全靠它。**

分派表里认证类型"无包装，直调 `plugin->init`"（见[插件类型全集](#插件类型全集与分派表)），对应地它的 `data` 字段**不会被内核填写**：

| 类型 | `st_plugin_int::data` | 为什么 |
|---|---|---|
| STORAGE ENGINE | `handlerton *` | **有状态**：每个引擎一个 handlerton，内核要先分配再交给插件填 |
| AUDIT | `st_mysql_audit *` | 内核要拿着它算全局 audit mask |
| AUTHENTICATION | **不填** | **vtable 无状态**：`st_mysql_auth` 是纯函数表，插件自己的状态放在自己的全局变量里（如 `SHA256_password_cache`、`g_caching_sha2_rsa_keys`） |

★ 这条差异解释了"为什么 SE 必须有内核包装而认证插件不需要"：**有没有内核需要代管的实例状态**，是包装与否的真正判据，不是类型的重要性。

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
- 架构总览（双轨制与迁移进度）见 [`architecture.md`](architecture.md)
- 认证插件的完整剖析（`caching_sha2_password` / `mysql_native_password` / `sha256_password` / `auth_socket` …）见 [`../auth/authentication.md`](../auth/authentication.md)：本篇只讲插件框架，凭据校验与握手协议在那边
- ABI 契约专题（三层版本校验/旧结构迁移/符号可见性）见 [`abi.md`](abi.md)
- 组件基础设施见 [`component.md`](component.md)
- 服务（含老版 plugin service 与 registry 内部）见 [`service.md`](service.md)
- 具体插件：物理克隆 [`clone.md`](clone.md)、X Plugin [`mysqlx.md`](mysqlx.md)、密钥管理 [`keyring.md`](keyring.md)、口令强度 [`validate_password.md`](validate_password.md)（★ 插件与组件双实现、运行时择优分派）
