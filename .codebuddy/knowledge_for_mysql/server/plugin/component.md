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

**第零条（第一性原理）：把"依赖"从服务器代码里搬进数据里。**

组件体系的一切设计都源自一个动作：**把插件时代藏在 `mysqld_main` 里的初始化顺序（先 keyring、再 InnoDB、再其他），变成每个组件自带的数据（requires 清单）**。一旦依赖成为数据，后面的所有机制都能机械地推导出来：

| 依赖成为数据之后 | 派生出的机制 |
|---|---|
| 依赖可枚举 | 加载前先做**依赖检查**（`load_do_check_dependencies`：同批 provides 集合 + registry 两层查找） |
| 依赖可排序 | **DFS 后序拓扑排序**决定卸载顺序（含环处理） |
| 依赖可回填 | 占位符指针注入（`load_do_resolve_dependencies`） |
| 依赖可计数 | 引用计数 > 批内引用数 → **拒绝卸载** |
| 依赖可回滚 | 每阶段 scope_guard，失败整批撤销 |
| 依赖可替换 | default 实现 + `set_default` → **override** 能力 |

反过来说，插件体系"没有依赖数据"这件事也机械地决定了它的三条宿命：初始化顺序硬编码成三段、卸载只能延迟收割、插件之间互不可见（见 [`plugin.md`](plugin.md) 的设计思想）。

代价同样清晰：**批量而非增量**——加载/卸载都以"组"为单位成败，粒度比插件粗；依赖图的正确性由加载器保证，组件作者只声明、不控制顺序。

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

**★★ 完整宏链（补齐上面被截断的部分）**。`DECLARE_COMPONENT` 只有 4 行，且**故意不闭合花括号**——init/deinit 两个函数指针要由使用者手写填进去，最后由 `END_DECLARE_COMPONENT()` 补 `}`：

```c
/* include/mysql/components/component_implementation.h:152 */
#define DECLARE_COMPONENT(source_name, name)                        \
  mysql_component_t mysql_component_##source_name = {               \
      name, __##source_name##_provides, __##source_name##_requires, \
      __##source_name##_metadata,          /* ← 注意：此处不闭合 */

#define END_DECLARE_COMPONENT() }
```

```c
/* provides / requires / metadata 三段的开闭宏（:175-383） */
#define BEGIN_COMPONENT_PROVIDES(name) \
  static struct mysql_service_ref_t __##name##_provides[] = {

#define PROVIDES_SERVICE(component, service)                            \
  {                                                                     \
    #service "." #component,       /* ★ 顺序是【服务名.组件名】，不是反过来 */ \
        const_cast<void *>(                                             \
            ((const void *)&SERVICE_IMPLEMENTATION(component, service))) \
  }

#define END_COMPONENT_PROVIDES() \
  { NULL, NULL }                 \
  }                              /* ★ 哨兵收尾，遍历靠 implementation != nullptr 判结束 */

#define BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY(name) \
  static struct mysql_service_placeholder_ref_t __##name##_requires[] = {

#define REQUIRES_SERVICE_PLACEHOLDER(service) \
  SERVICE_TYPE(service) * mysql_service_##service   /* ★ 是变量声明，不是类型定义 */

#define REQUIRES_SERVICE(service)                                           \
  {                                                                        \
    #service,                                                              \
        static_cast<void **>(                                              \
            static_cast<void *>(const_cast<mysql_service_##service##_t **>( \
                &mysql_service_##service)))                                \
  }   /* ★ 三次转型把"服务指针的地址"擦成 void**，供 loader 统一回写 */

#define END_COMPONENT_REQUIRES() \
  { NULL, NULL }                 \
  }

#define BEGIN_COMPONENT_METADATA(name) \
  static struct mysql_metadata_ref_t __##name##_metadata[] = {
#define METADATA(key, value) { key, value }
#define END_COMPONENT_METADATA() \
  { NULL, NULL }                 \
  }

#define COMPONENT_REF(name) mysql_component_##name
#define COMPONENT_ENTRY_FUNC "list_components"   /* ★ dlsym 用的入口名 */
```

> ★★ **`PROVIDES_SERVICE` 里名字的顺序是 `服务名.组件名`**（如 `registry.mysql_minimal_chassis`），不是反过来。这是理解 registry 里"默认实现"那套机制的前提。
> ★ `REQUIRES_SERVICE` 的三次转型（`const mysql_service_x_t **` → `void *` → `void **`）就是为了绕过 C++ 类型系统，让 dynamic loader 能用一段通用代码回写任意服务类型的指针。

**★ 四个结构的完整定义**（`include/mysql/components/services/dynamic_loader.h:230`）：

```c
struct mysql_service_ref_t {           /* 我提供的某个 service 实现 */
  const char *name;                    /* "服务名.组件名" */
  void *implementation;                /* 指向 imp_<comp>_<svc> 方法指针结构体 */
};

struct mysql_service_placeholder_ref_t {   /* 我依赖的某个 service */
  const char *name;                    /* 只写服务名，如 "registry" */
  void **implementation;               /* ★ 框架 acquire 后回写到这个槽位 */
};

struct mysql_metadata_ref_t { const char *key; const char *value; };

struct mysql_component_t {
  const char *name;                              /* URN */
  struct mysql_service_ref_t *provides;
  struct mysql_service_placeholder_ref_t *requires_service;
  struct mysql_metadata_ref_t *metadata;
  mysql_service_status_t (*init)();
  mysql_service_status_t (*deinit)();
};
```

**★ service 定义宏的真身**（⚠️ `BEGIN_SERVICE_DEFINITION` / `DECLARE_METHOD` / `DEFINE_SERVICE_HANDLE` 在 **`include/mysql/components/service.h`**，不在 `service_implementation.h`）：

```c
/* service.h:67 —— 定义"服务的接口"（方法指针结构体） */
#define SERVICE_TYPE(name) const mysql_service_##name##_t
#define BEGIN_SERVICE_DEFINITION(name) typedef struct s_mysql_##name {
#define END_SERVICE_DEFINITION(name) } SERVICE_TYPE_NO_CONST(name);
#define DECLARE_METHOD(retval, name, args) retval(*name) args
#define DECLARE_BOOL_METHOD(name, args) \
  DECLARE_METHOD(mysql_service_status_t, name, args)
#define DEFINE_SERVICE_HANDLE(name) typedef struct name##_imp *name  /* ★ 不透明句柄 */

/* service_implementation.h —— 定义"服务的实现" */
#define SERVICE_IMPLEMENTATION(component, service) imp_##component##_##service
#define BEGIN_SERVICE_IMPLEMENTATION(component, service) \
  SERVICE_TYPE(service) SERVICE_IMPLEMENTATION(component, service) = {
#define END_SERVICE_IMPLEMENTATION() }
#define DEFINE_METHOD(retval, name, args) retval name args noexcept  /* ★ noexcept 保证不跨 ABI 抛异常 */
#define DEFINE_BOOL_METHOD(name, args) DEFINE_METHOD(mysql_service_status_t, name, args)
```

> ★ `DEFINE_METHOD` 里的 **`noexcept`** 是刻意的：服务方法跨动态库边界调用，绝不能抛异常。

---

### ★★ Registry 的真身：`mysql_registry_imp`（⚠️ 不在 `sql/server_component/`）

**⚠️ 重大勘误**：不存在 `sql/server_component/registry.{h,cc}`，也不存在 `class Registry`。真正的实现是 **`class mysql_registry_imp`**，位于 **`components/libminchassis/registry_imp.h:40`** + **`components/libminchassis/registry.cc`**。（`registry_imp.h` 的 include guard 还叫 `MYSQL_SERVER_REGISTRY_H`——是历史迁移留下的痕迹。）

它用**两张 `std::map`**，不是链表：

```cpp
/* components/libminchassis/registry_imp.h:28 */
typedef std::map<const char *, mysql_service_implementation *, c_string_less>
    my_service_registry;      /* ★ 主表：名字 → 实现对象 */

class mysql_registry_imp {
  typedef std::map<my_h_service, mysql_service_implementation *>
      my_interface_mapping;   /* ★ 反查表：接口结构体指针 → 实现对象 */

  static my_service_registry service_registry;
  static my_interface_mapping interface_mapping;
  static mysql_rwlock_t LOCK_registry;   /* ★ 整个 registry 一把 rwlock */
};
```

> ★ 比较器是 **`c_string_less`**（`include/c_string_less.h:35`，即 `strcmp(s1,s2) < 0` 按**内容**字典序比较，不是按指针）。这个"按内容排序"是下面"默认实现"机制成立的前提。

**★★★ "默认实现"的实现机制：往同一张 map 再插一条短名条目**

这是最反直觉的一处。注册 `registry.mysql_server` 时，会往 **同一张 map** 插**两条**记录——一条 key 是全名，一条 key 是服务名（`registry`）：

```cpp
/* components/libminchassis/registry.cc:249 */
bool mysql_registry_imp::register_service_nolock(
    const char *service_implementation_name, my_h_service ptr) {
  std::unique_ptr<mysql_service_implementation> imp = ...(
      new mysql_service_implementation(ptr, service_implementation_name));

  if (imp->interface() == nullptr) return true;   /* 名字不合法 → 失败 */

  /* Register the implementation name. */
  std::pair<my_service_registry::iterator, bool> addition_result =
      mysql_registry_imp::service_registry.emplace(imp->name_c_str(), imp.get());
  if (!addition_result.second) return true;       /* ★ 同名已存在 → 冲突失败 */

  /* Register interface in mapping */
  mysql_registry_imp::interface_mapping.emplace(imp->interface(), imp.get());

  /* Register the Service Implementation as default for Service name in case
    none were registered before. */
  mysql_registry_imp::service_registry.emplace_hint(
      addition_result.first, imp->service_name_c_str(), imp.get());
  /* ★★ 往同一张 map 再插一条 key = "registry" 的条目。
     若该 key 已存在，emplace_hint 静默失败 —— 于是"第一个注册的成为默认"。 */

  imp.release();
  return false;
}
```

> ★★★ 因此 **`acquire("registry")` 与 `acquire("registry.mysql_server")` 走的是同一张 map 的两次不同查找**——前者命中短名条目（默认实现），后者命中全名条目。所谓"默认实现"就是"多插一条短名条目"，**没有任何独立的 default 字段**。
> ★ `emplace_hint` 的 hint 传的是刚插入全名条目的迭代器——因为字典序下 `"registry"` 恰好排在 `"registry.xxx"` 紧前面，提示命中即 O(1)。

**★★ 注销时如何选新默认**：靠"erase 返回下一个迭代器 + 字典序相邻"

```cpp
/* registry.cc:280 */
bool mysql_registry_imp::unregister_nolock(const char *service_implementation_name) {
  {
    my_service_registry::iterator imp_iter =
        mysql_registry_imp::service_registry.find(service_implementation_name);
    if (imp_iter == service_registry.end() ||
        imp_iter->second->get_reference_count() > 0) {
      return true;      /* ★ 两条拒绝：找不到 / 引用计数 > 0（还有人在用） */
    }
    imp.reset(imp_iter->second);
    mysql_registry_imp::service_registry.erase(imp_iter);
  }
  mysql_registry_imp::interface_mapping.erase(
      interface_mapping.find(imp->interface()));

  /* Look if it is the default implementation. */
  my_service_registry::iterator default_iter =
      service_registry.find(imp->service_name_c_str());
  if (default_iter == service_registry.end()) return true;   /* 状态不一致 */

  if (default_iter->second == imp.get()) {
    my_service_registry::iterator new_default_iter =
        service_registry.erase(default_iter);   /* ★ erase 返回"被删元素的下一个" */
    if (new_default_iter != service_registry.end() &&
        !strcmp(imp->service_name_c_str(),
                new_default_iter->second->service_name_c_str())) {
      /* ★ 确认下一个条目确实属于同一个 service（前缀相同）才提升为新默认 */
      mysql_service_implementation *new_default = new_default_iter->second;
      service_registry.emplace_hint(new_default_iter,
                                    new_default->service_name_c_str(),
                                    new_default);
    }
  }
  return false;
}
```

> ★★ 因为 map 按字典序排，删除 `"registry"` 后返回的迭代器**正好指向 `"registry.<下一个组件>"`**——这就是"任意挑一个仍注册的成为默认"的实现。`c_string_less` 按内容比较是这个技巧成立的前提。

**★ `set_default()` 为什么必须"先删后插"**（★ 一处很容易写错的坑）：

```cpp
    /* We have to remove and reinsert value as key, the string pointer will
      not be valid if we unregister previous default implementation. */
    iter = service_registry.erase(service_registry.find(imp->service_name_c_str()));
    service_registry.emplace_hint(iter, imp->service_name_c_str(), imp);
```

> ★ 因为 **map 的 key 是 `imp` 对象内部的 `char*`**（`m_full_name.c_str()`），换默认实现意味着旧默认对象的 key 缓冲可能随之失效——所以不能只改 value，必须换 key。

**★ 引用计数**：`std::atomic<uint64>` + CAS，没有单独锁（`components/libminchassis/my_ref_counted.cc`）：

```cpp
class my_ref_counted {
  std::atomic<uint64> m_count;   /* ★ 就一个原子变量 */
};

uint64 my_ref_counted::add_reference() { return m_count++; }

bool my_ref_counted::release_reference(uint64 *new_count) {
  uint64 old_val = m_count;
  do {
    if (old_val == 0) return true;    /* ★ 已为 0 → 拒绝（多 release 被这里挡住） */
  } while (!m_count.compare_exchange_weak(old_val, old_val - 1));
  ...
}
```

**★ `acquire` / `release` 的两层结构**（每个对外方法都是"加锁 → 调 `_nolock`"）：

```cpp
/* registry.cc:161 —— 引用计数 +1 的落点 */
bool mysql_registry_imp::acquire_nolock(const char *service_name,
                                        my_h_service *out_service) {
  my_service_registry::const_iterator iter;
  iter = mysql_registry_imp::service_registry.find(service_name);
  if (iter == service_registry.cend()) return true;
  mysql_service_implementation *imp = iter->second;
  imp->add_reference();             /* ★★ 引用计数 +1 */
  *out_service = imp->interface();  /* ★ 返回的是【接口结构体指针】，不是 imp 对象 */
  return false;
}

/* release 只有接口指针，必须靠第二张 map 反查 */
bool mysql_registry_imp::release_nolock(my_h_service service) {
  mysql_service_implementation *service_implementation =
      get_service_implementation_by_interface(service);   /* ★ interface_mapping 反查 */
  if (service_implementation == nullptr) return true;
  return service_implementation->release_reference();
}

DEFINE_BOOL_METHOD(mysql_registry_imp::acquire, (const char *service_name,
                                                 my_h_service *out_service)) {
  minimal_chassis::rwlock_scoped_lock lock(&LOCK_registry, false, __FILE__, __LINE__);
  /* ★ 第二参 false = 读锁。acquire 全程只加读锁 —— 引用计数靠原子变量自增 */
  return mysql_registry_imp::acquire_nolock(service_name, out_service);
}
```

**★ 迭代器把读锁"移动"进句柄**（★ 这就是接口文档里那句 `@warning` 的落点）：

```cpp
struct my_h_service_iterator_imp {
  my_service_registry::const_iterator m_it;
  minimal_chassis::rwlock_scoped_lock m_lock;   /* ★ RAII 持有读锁 */
};

DEFINE_BOOL_METHOD(mysql_registry_imp::iterator_create, (...)) {
  minimal_chassis::rwlock_scoped_lock lock(&LOCK_registry, false, __FILE__, __LINE__);
  ...
  *out_iterator = new my_h_service_iterator_imp{r, std::move(lock)};
  /* ★★ 锁的所有权被 move 进迭代器，直到 iterator_release 才释放 */
}
```

> ★★ 所以 **`registry_query` 的迭代器存活期间，整个 registry 被读锁锁住**。接口注释明确警告：*"Make sure to close the iterator before looking up the name into the registry! Otherwise undefined behavior will ensure. Most probably a deadlock."*

**★ 命名规则的强制点**（`mysql_service_implementation.cc:29`）——名字必须**恰好一个点**、两边各至少 1 字符、总长 ≥ 3：

```cpp
  first_dot = m_full_name.find_first_of('.');
  if (m_full_name.length() < 3 ||      /* dot and at least 1 char for each part */
      first_dot < 1 || first_dot >= m_full_name.length() - 1 ||
      first_dot != m_full_name.find_last_of('.')) {
    /* Set interface to NULL, this should be interpreted as failure to
      construct the object instance. */
    m_interface = {};
    return;
  }
  m_service = my_string(m_full_name.begin(), m_full_name.begin() + first_dot);
```

---

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

前四个阶段的代码要点（后三个阶段已在上面贴出 resolve 的实现）：

```c
/* ② collect：把 provides 里的 "X.Y" 截断成 "X" 收集成本批可提供集合 */
const char *dot_position = strchr(service_provided->name, '.');
if (dot_position == nullptr)
  services_provided.insert(my_string(service_provided->name));
else
  services_provided.insert(my_string(service_provided->name,
                                     dot_position - service_provided->name));

/* ③ check：每个 require 先查本批集合，再查 registry；都无则失败 */
if (services_provided.find(my_string(service_required->name)) != services_provided.end())
  continue;                                   /* 本批互供 */
my_h_service_iterator service_iterator;
if (!mysql_registry_imp::iterator_create(service_required->name, &service_iterator)) {
  mysql_registry_imp::iterator_release(service_iterator);
  continue;                                   /* registry 里已有 */
}
... ER_COMPONENTS_CANT_SATISFY_DEPENDENCY ...

/* ④ register：逐个注册，失败时用 scope_guard 反向 unregister 全部已注册项 */
auto guard = create_scope_guard([&registered_services]() {
  for (const char *service_name : registered_services)
    mysql_registry_imp::unregister(service_name);
});
for (const mysql_service_ref_t *implementation_it : loaded_component->get_provided_services())
  if (mysql_registry_imp::register_service(implementation_it->name,
          reinterpret_cast<my_h_service>(implementation_it->implementation))) { ...失败... }

/* ⑥ init：逐个调 mysql_component_t::init()，失败时把已 init 的逐个 deinit */
if (loaded_component->get_data()->init != nullptr &&
    loaded_component->get_data()->init()) { ... ER_COMPONENTS_LOAD_CANT_INITIALIZE ... }
```

三个值得注意的细节：

- **② 的截断**意味着"本批互供"只看服务名（`X`），不看实现名——所以 A 提供 `X.A`、B 需要 `X`，同批加载即可满足，**加载顺序无关**（这正是"依赖声明化"带来的自由）。
- **③ 的两层查找顺序**是"本批集合 → registry"：同批优先，是为了支持循环依赖（A 依赖 B、B 依赖 A 时，两边都还没注册进 registry，只能靠本批集合互相满足）。
- **④ 与 ⑥ 的回滚**都是"反向遍历已做过的列表"（`registered_services` / `initialized_components`），所以用 scope_guard + `commit()` 表达：正常路径调 `guard.commit()` 取消回滚，异常路径自动回滚。

#### 代际链表：`urns_with_gen_list` 是什么

`load_do_commit()` 除了把组件放进 `components_list`（URN → 组件），还做了一件事：

```c
mysql_dynamic_loader_imp::urns_with_gen_list.emplace_front(components_to_load);
```

`urns_with_gen_list` 的类型是 `generation_urns_list`（`std::forward_list<std::vector<mysql_component*>>`）——**一个"代"就是一次 `load()` 调用的那一批组件**，插在链表头。它的用途有三处：

1. **卸载时定位批次**：`unload_do_unload_components` 遍历代际链表找到包含目标组件的那个 vector，从中 erase；vector 空了就把整个代节点移除——**"批"的边界因此得以保持**。
2. **关闭时逐代回退**：`mysql_dynamic_loader_imp::deinit()` 里对每一代调一次 `unload_do_topological_order()`（源码注释还提醒要先**复制 vector** 再传，否则迭代中修改会让迭代器失效）。
3. **循环依赖的物理基础**：正是一批组件共享一个代节点，才可能"同批卸载"来解决环。

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

卸载链是十阶段，比加载长得多——因为**卸载要考虑顺序、引用计数与依赖释放**，而加载只需要"按顺序建立"。逐个拆解（前五个阶段是难点）：

| 阶段 | 做什么 | 关键设计点 |
|---|---|---|
| `unload_do_list_components` | 按 URN 找出组件，先发现"未加载/重复"错误 | 批内去重，任一 URN 未加载即整体失败 |
| `unload_do_topological_order` | **DFS 后序**拓扑排序，产出卸载顺序 | 见下方详解 |
| `unload_do_get_scheme_services` | 预取各 URN 的 scheme 服务 | 因为下阶段要持 registry 写锁，**不能再 acquire**——与插件 `INSTALL PLUGIN` 预取 audit 句柄是同一个套路 |
| `unload_do_lock_provided_services` | 锁住整个 registry（`lock_registry_for_write`） | 注释自陈"we don't have yet any better granulation"——粗粒度锁，保证"检查引用计数"与"后续注销"之间不被插入新引用 |
| `unload_do_check_provided_services_reference_count` | 引用计数检查（**卸载安全的核心**） | 见下方详解 |
| `unload_do_deinitialize_components` | 调各组件 `deinit` | **失败不回滚**——见下方详解 |
| `unload_do_unload_dependencies` | release 本批组件 acquire 过的依赖服务 | 逆序释放 |
| `unload_do_unregister_services` | 从 registry 注销本批提供的服务实现 | 顺序保证"消费者先走、提供者后走" |
| `unload_do_unload_components` | 调 scheme 服务 `unload`，`dlclose` | 真正卸载 `.so` |
| `unload_do_commit` | 从 `components_list` / `urns_with_gen_list` 移除 | 提交，不可回滚 |

#### 拓扑排序：DFS 后序 + 显式依赖图

`unload_do_topological_order` 先建依赖图（`dependency_graph`：某个服务实现 → 批内依赖它的组件列表），再对每个未访问节点跑 DFS：

```c
/* A DFS run on directional graph, possibly a cyclic-one, with
   V=(list of the Components to unload) and
   E={A->B : component B depends on component A}. A DFS post-order will
   result in a topological ordered list of components. */
depth_first_search(
    component, [](mysql_component *) {},
    [&components_to_unload_ordered](mysql_component *visited_component) {
      components_to_unload_ordered.push_back(visited_component);   // 后序
    }, ...);
```

两个要点：**后序**（子节点先出 → 依赖者先卸载，被依赖者后卸载），以及 **`visited_set` 跨 DFS 调用保留状态**——这就是环的处理方式：有环时 DFS 不会无限递归，产出的顺序虽非严格拓扑，但批内组件最终全部卸载，这也是"循环依赖必须同批卸载"在算法层的兑现。

#### 引用计数检查：判据是"恰好等于批内引用数"

```c
uint64_t reference_count =
    mysql_registry_imp::get_service_implementation_reference_count(
        reinterpret_cast<my_h_service>(service->implementation));
if (reference_count > 0) {
  it = dependency_graph.find(service->implementation);
  if (it == dependency_graph.end() || reference_count != it->second.size()) {
    mysql_error_service_printf(ER_COMPONENTS_UNLOAD_CANT_UNREGISTER_SERVICE, MYF(0),
                               service->name, component->name_c_str());
    return true;                      // 拒绝卸载
  }
}
```

判据不是"计数 > 0 就拒绝"，而是**"计数必须恰好等于批内依赖它的组件数"**——多一个就说明批外还有人持有句柄。这是"市场退场规则"的精确实现：批内引用会在本次卸载中一起消失，批外引用则无从释放。

与插件的对照：同样是"有人还在用"，**组件选择阻塞卸载，插件选择标记删除后异步收割**（`DELETED` + `ref_count` 归零后 reap）——数据结构不同（多实现市场 vs 全局单例），失败语义自然不同。

> ⭐ 补充两个源码细节：① **判据其实分两档**——`==0` 直接放行（没人用，随便卸）；`>0` 时才要求"恰好等于"。② 建图处用 `dependency_graph[...]`（`operator[]`，找不到会**插入空 vector**），而这里检查用 `find()`，二者语义不对称。另 `get_service_implementation_reference_count()` 在找不到时返回 `(uint64_t)-1` 而**不是**注释写的 0——源码与注释不一致，不过因 `> 0` 判据恰好无影响。

**★★ 代际链表的两处坑**（源码实证）：

⚠️ `dynamic_loader_imp.h:44-50` 那段注释说 `my_component_registry` 是"(component, generation ID) 的 pair"——**这注释已过时**。真实类型是：

```cpp
typedef std::map<const char *, std::unique_ptr<mysql_component>, c_string_less>
    my_component_registry;   /* ★ map 里根本没有 generation ID 字段 */

using components_vector = std::vector<mysql_component *>;
using generation_urns_list = std::forward_list<components_vector>;
```

"generation" = **一次 `load()` 调用的一个批次**，不是递增 ID。增加与删除都是**内联**的——**不存在** `add_component_to_urn_list()` / `remove_component_from_urn_list()` 这两个函数：

```cpp
/* 增加（load_do_commit, dynamic_loader.cc:943） */
  mysql_dynamic_loader_imp::urns_with_gen_list.emplace_front(components_to_load);
  /* ★ 头插 → 链表从头到尾 = 从"最新批次"到"最老批次"（LIFO） */

/* 删除（unload_do_unload_components, :1343）—— 线性扫描 + 按指针值比较 */
    for (auto list = urns_with_gen_list.begin(); list != urns_with_gen_list.end(); ++list) {
      auto comp_ele = (*component_it).second.get();
      auto v_element = find_if((*list).begin(), (*list).end(),
                  [comp_ele](mysql_component *ptr) { return ptr == comp_ele; });
      if (v_element != list->end()) {
        list->erase(v_element);
        if (list->size() == 0) urns_with_gen_list.remove(*list);  /* 组空 → 摘节点 */
        break;
      }
    }
    mysql_dynamic_loader_imp::components_list.erase(component_it);
```

**★ `deinit()` 每轮都重新 `begin()`**（★ 因为下游会把节点删掉，且必须先复制 vector）：

```cpp
    if (components_list.size() > 0) {
      for (auto it = urns_with_gen_list.begin(); it != urns_with_gen_list.end();) {
        /* If we pass (*it) vector, unload_do_unload_components() function
           will iterates and modify urns_with_gen_list's vector. And both
           iterator and modify will work on same vector, which leads to
           vector iterator invalidation. So, we need to make a copy
           of the vector and pass it to below function */
        auto components_to_unload = (*it);            /* ★ 拷贝整组，避免迭代器失效 */
        if (unload_do_topological_order(components_to_unload)) {
          it->clear();
          urns_with_gen_list.remove(*it);
        }
        it = urns_with_gen_list.begin();              /* ★★ 每轮从头开始 */
      }
    }
```

**★★ 拓扑排序的 DFS 模板本体**（⚠️ 是**通用模板** `include/depth_first_search.h:34`，不在 libminchassis 下；且是**显式栈迭代**而非递归）：

```cpp
template <typename TVertex, typename TVisit_start, typename TVisit_end,
          typename TGet_neighbors, typename TVertex_less = std::less<TVertex>>
void depth_first_search(TVertex start_vertex, TVisit_start visitor_start,
                        TVisit_end visitor_end, TGet_neighbors get_neighbors,
                        std::set<TVertex> &visited_set = std::set<TVertex>{}) {
  constexpr int START_VISITING = 0;
  constexpr int CURRENT_VERTEX = 1;

  std::stack<std::tuple<bool, TVertex>> stack;
  if (!visited_set.insert(start_vertex).second) return;
  stack.emplace(true, start_vertex);

  while (!stack.empty()) {
    std::tuple<bool, TVertex> &elem = stack.top();
    if (std::get<START_VISITING>(elem)) {
      visitor_start(std::get<CURRENT_VERTEX>(elem));
      std::get<START_VISITING>(elem) = false;
      for (TVertex neighbor : get_neighbors(std::get<CURRENT_VERTEX>(elem))) {
        if (visited_set.insert(neighbor).second) stack.emplace(true, neighbor);
      }
    } else {
      visitor_end(std::get<CURRENT_VERTEX>(elem));   /* ★★ 后序：所有邻居处理完才调用 */
      stack.pop();
    }
  }
}
```

> ★ 用 `std::tuple<bool, TVertex>` 同一个元素进出栈两次：第一次置 `false` 并压入邻居，第二次（邻居都处理完）调 `visitor_end`。**这个"二进二出"是后序的经典迭代式写法**，避免了递归爆栈。

**★ 依赖图的顶点是"服务实现指针"，不是组件**（`dynamic_loader.cc:990`）：

```cpp
  std::map<const void *, std::vector<mysql_component *>> dependency_graph;
  /* ★ key = 服务实现指针(service->implementation)，value = 批内依赖它的组件 */

  for (mysql_component *component : components_to_unload)
    for (const mysql_service_ref_t *service : component->get_provided_services())
      dependency_graph.emplace(service->implementation, std::vector<mysql_component *>{});

  for (mysql_component *component : components_to_unload)
    for (const mysql_service_placeholder_ref_t *service : component->get_required_services()) {
      auto it = dependency_graph.find(*service->implementation);
      if (it != dependency_graph.end()) it->second.push_back(component);
      /* ★ 用"已解析的指针"比对，而不是名字 —— 精确知道依赖的是哪个实现 */
    }
```

**★★ "循环依赖必须同批加载"在加载侧的机制**（`load_do_check_dependencies`，★ 与卸载侧的拓扑排序是同一件事的两面）：

```cpp
      /* Try lookup in services provided by other components being loaded. */
      if (services_provided.find(my_string(service_required->name)) !=
          services_provided.end()) {
        continue;      /* ★★ 批内互相满足 → 这就是"循环依赖必须同批加载"的实现 */
      }
      /* Try to lookup in services registered in registry service */
      my_h_service_iterator service_iterator;
      if (!mysql_registry_imp::iterator_create(service_required->name,
                                               &service_iterator)) {
        mysql_registry_imp::iterator_release(service_iterator);
        continue;
      }
      /* None service matches requirement, we shall fail. */
      mysql_error_service_printf(ER_COMPONENTS_CANT_SATISFY_DEPENDENCY, MYF(0), ...);
```

> ★ `services_provided` 里只登记**点前部分**（服务名，剥离实现名），所以批内组件 A 可以依赖同批组件 B 提供的服务——这是环能成立的原因。

---

#### ★ URN → scheme 的分发：不是硬编码 `file://`

**★★ 分发核心**（`dynamic_loader.cc:1387`）——取 `://` 前的部分，拼成**服务名**去 registry acquire：

```cpp
bool mysql_dynamic_loader_imp::get_scheme_service_from_urn(
    const my_string &urn, SERVICE_TYPE(dynamic_loader_scheme) **out_scheme_service,
    scheme_service_map &scheme_services) {
  /* Find scheme prefix. */
  size_t scheme_end = urn.find("://");
  if (scheme_end == my_string::npos) {
    mysql_error_service_printf(ER_COMPONENTS_NO_SCHEME, MYF(0), urn.c_str());
    return true;                                  /* ★ 没有 "://" → ER_COMPONENTS_NO_SCHEME */
  }
  my_string scheme(urn.begin(), urn.begin() + scheme_end, urn.get_allocator());

  scheme_service_map::iterator scheme_it = scheme_services.find(scheme);
  if (scheme_it != scheme_services.end()) {
    *out_scheme_service = scheme_it->second;      /* 缓存命中 */
  } else {
    my_service<SERVICE_TYPE(dynamic_loader_scheme)> service(
        (my_string("dynamic_loader_scheme_") + scheme).c_str(),
        &imp_mysql_minimal_chassis_registry);
    /* ★★★ 服务名 = "dynamic_loader_scheme_" + scheme
       file:// → "dynamic_loader_scheme_file" */
    if (service) {
      mysql_error_service_printf(ER_COMPONENTS_NO_SCHEME_SERVICE, MYF(0),
                                 scheme.c_str(), urn.c_str());
      return true;
    }
    *out_scheme_service = service;
    scheme_services.insert(make_pair(scheme, std::move(service)));
  }
  return false;
}
```

> ★★★ 所以 **`file://` 不是硬编码**——它走的是"拼服务名 → registry acquire"。新增一个 scheme 只需提供名为 `dynamic_loader_scheme_<xxx>` 的服务即可，**不需要改 dynamic loader 一行代码**。
> ⚠️ `builtin://` 只是保留名，**无实现**（`dynamic_loader.cc:183-195` 只是一段注释）。

**★ 服务器侧还有一个 `file://` 的包装器**（★ 这是"为什么组件 .so 必须在 plugin_dir 下"的答案）：

```cpp
/* sql/server_component/dynamic_loader_path_filter.cc:56 */
DEFINE_BOOL_METHOD(mysql_dynamic_loader_scheme_file_path_filter_imp::load,
                   (const char *urn, mysql_component_t **out_data)) {
  my_string path;
  if (check_and_make_absolute_urn(urn, path)) return true;
  return (scheme_file_srv->load(path.c_str(), out_data));
  /* ★ 把 URN 规整成 plug-in dir 下的绝对路径后，转调底层 file scheme 实现 */
}
```

它注册为**同名服务**并成为**默认实现**——在 `component_infrastructure_init()` 里：

```cpp
  registrator->set_default("dynamic_loader_scheme_file.mysql_server_path_filter");
  /* ★★ 此后 dlopen 只允许加载 plugin dir 下的组件 */
```

**★★ 真正的 `dlopen` 现场**（`dynamic_loader_scheme_file.cc:84`）：

```cpp
DEFINE_BOOL_METHOD(mysql_dynamic_loader_scheme_file_imp::load,
                   (const char *urn, mysql_component_t **out_data)) {
  if (object_files_list.find(urn_string) != object_files_list.end())
    return true;                                     /* ① 重复 URN → 失败 */

  const char *file = strstr(urn, "://");
  file += 3;                                         /* ★ 跳过 "://" */
  std::string file_name = std::string(file);
  file_name += ".so";                                /* ★★ 自动补后缀：file://x → x.so */

#if defined(HAVE_ASAN) || defined(HAVE_LSAN)
  void *handle = dlopen(file_name.c_str(), RTLD_NOW | RTLD_NODELETE);
#else
  void *handle = dlopen(file_name.c_str(), RTLD_NOW);   /* ★★ 真正的 dlopen */
#endif
  ...
  list_components_func list_func = reinterpret_cast<list_components_func>(
      dlsym(handle, COMPONENT_ENTRY_FUNC));         /* ★★ dlsym("list_components") */
  if (list_func == nullptr) return true;

  /* Check if library is not already loaded, by comparing "list_components"
    function address. */
  if (library_entry_set.insert(list_func).second == false) return true;
  /* ★★ 二次去重：按入口函数地址 —— 同一 .so 的不同 URN / 符号链接也能挡住 */

  *out_data = list_func();                          /* ★ 调 list_components() 拿组件结构 */
  if (object_files_list.emplace(urn_string, handle).second == false) return true;
```

```cpp
/* 记录已加载库的容器（dynamic_loader_scheme_file_imp.h:61） */
  typedef std::map<std::string, void *> my_registry;   /* URN → dlopen handle */
  typedef mysql_component_t *(*list_components_func)();
  static my_registry object_files_list;
  static std::unordered_set<list_components_func> library_entry_set;  /* 按入口地址去重 */
```

> ★★ **两重去重**：① 按 URN 字符串；② **按 `list_components` 函数地址**——后者能挡住"同一 .so 用不同路径/符号链接重复加载"。
> ★ `unload()` 里 `object_files_list.size() == 0` 时返回 `false`（成功）——幂等放行。

#### 一个不对称：加载严格回滚，卸载的 deinit 失败不回滚

```c
if (component->get_data()->deinit()) {
  /* In case of error we don't want to try to restore consistent state.
     This is arbitrary decision, rollback of this operation is possible,
     but it's not sure if components will be able to initialize again
     properly, causing state to be inconsistent. */
  mysql_error_service_printf(ER_COMPONENTS_UNLOAD_CANT_DEINITIALIZE, MYF(0), ...);
  deinit_result = true;
}
```

注释直言这是"arbitrary decision"：**回滚（重新 init）在技术上可行，但不能保证组件能重新初始化成功**，半卸载状态比"继续往下卸完"更危险，所以选择继续。这与加载链每阶段严格 scope_guard 回滚形成刻意的不对称——**可逆的操作严格回滚，不可逆的操作宁可一路走完**。

### 组件 init/deinit 的两条硬约束（源码教程原文）

`include/mysql/components/component_implementation.h` 的 Tutorial 里有两条不起眼但会咬人的规定（原文照录）：

> -# Make sure component is loaded/initialized before using its services.
>    Atomic variable is_intialized represents the state of the component.
>    Please check the details about the variable from validate_password_imp.cc file.
> -# Using registry acquire/release services inside component's init/deint
>    functions is not supported. All the required services for the component
>    has to be specified under BEGIN_COMPONENT_REQUIRES section only.

展开讲清"为什么"：

1. **init/deinit 里不能 acquire/release**。因为依赖解析（`load_do_resolve_dependencies`）本来就发生在 **init 之前**——你在 init 里再 acquire 的服务，卸载时不会有人替你 release（`unload_do_unload_dependencies` 只释放 requires 列表里的那些），引用计数泄漏会直接导致"组件永远卸不掉"（见卸载链的引用计数判据）。所以规则是：**一切依赖必须静态声明，运行期不许临时索取**。这条限制也是"声明式契约"设计（见理论基础 ④）的直接后果。
2. **`is_intialized` 原子变量是惯例而非框架能力**。框架只保证"init 返回后才算加载完成"，但服务被谁、在什么时机调用，框架管不了；所以组件自己要维护一个原子标志（参考 `components/validate_password/validate_password_imp.cc` 的写法），防止"组件正在初始化、服务已被调用"的竞态。

这两条合起来说明一件事：**组件框架的"声明式"是有代价的**——它换来了依赖可解析、可拓扑、可回滚，代价是运行期的灵活性被严格限制（不能动态取服务），正确性责任部分落到了组件作者身上（自己维护初始化标志）。

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

**`load()` 的完整流程（代码级）——"先写表、后加载"与双回滚：**

```cpp
DEFINE_BOOL_METHOD(mysql_persistent_dynamic_loader_imp::load,
                   (void *thd_ptr, const char *urns[], int component_count)) {
  MUTEX_LOCK(lock, &component_id_by_urn_mutex);
  Disable_binlog_guard binlog_guard(thd);            /* ① INSTALL COMPONENT 不复制 */

  TABLE *component_table;
  auto guard_close_tables = create_scope_guard([&thd] {   /* ② 表级回滚 */
    trans_rollback_stmt(thd); close_mysql_tables(thd);
  });
  if (open_component_table(thd, TL_WRITE, &component_table, INSERT_ACL)) { ... }

  uint64 group_id = ++mysql_persistent_dynamic_loader_imp::s_group_id;   /* ③ 批次号 */

  std::map<my_string, uint64> copy_urn_map(component_id_by_urn);         /* ④ 内存缓存副本 */
  auto guard = create_scope_guard([&copy_urn_map]() {
    mysql_persistent_dynamic_loader_imp::component_id_by_urn = copy_urn_map;
  });

  for (int i = 0; i < component_count; ++i) {                            /* ⑤ 逐行写表 */
    restore_record(component_table, s->default_values);
    component_table->field[CT_FIELD_GROUP_ID]->store(group_id, true);
    component_table->field[CT_FIELD_COMPONENT_URN]->store(urns[i], strlen(urns[i]), system_charset);
    int res = component_table->file->ha_write_row(component_table->record[0]);
    if (res != 0) { my_error(ER_COMPONENT_MANIPULATE_ROW_FAILED, MYF(0), urns[i], res); ... }
    component_id_by_urn.emplace(urns[i], component_table->file->insert_id_for_cur_row);
  }

  if (dynamic_loader_srv->load(urns, component_count)) return true;      /* ⑥ 才真正加载 */

  guard.commit(); guard_close_tables.commit(); trans_commit_stmt(thd);    /* ⑦ 双提交 */
  return false;
}
```

四点值得记住：

1. **顺序是"先写表、后加载"**（⑥ 在 ⑤ 之后），与 `INSTALL PLUGIN` 完全同构。代价是加载失败必须回滚表行——所以准备了**两套回滚**：表行靠事务（`trans_rollback_stmt`，② 的 scope_guard），内存缓存 `component_id_by_urn` 靠拷贝-恢复（④ 的 scope_guard）。两个 scope_guard 必须**同时 commit**（⑦），否则会出现"表里有、缓存里没有"的不一致。
2. **`component_group_id` 是批次号**：一条 `INSTALL COMPONENT 'a','b'` 里的多个 URN 共享同一 group_id——这就是卸载时"按组"处理的依据（循环依赖必须同批，数据源头在这里）。
3. **不写 binlog**（① `Disable_binlog_guard`）：与 `INSTALL PLUGIN` 一致，备库不会因主库装了组件就自动装——**组件/插件的安装是本地状态**，这也解释了为什么升级后要人工确认组件。
4. **`component_id_by_urn` 内存缓存**（URN → component_id）用于卸载时快速定位行，它的正确性依赖 ④ 的副本回滚，而不是依赖事务——因为内存结构不在事务保护范围内。

### 启动引导：四条链（不是三条）

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

**★★ 纠正：其实是四条链，不是三条。** 硬编码的 `component_urns[]` 是独立的第 3 条（**不写 `mysql.component` 表**）：

| 顺序 | 行号 | 入口 | 加载对象 | 写入 `mysql.component`？ |
|---|---|---|---|---|
| ① | `mysqld.cc:7602` | `component_infrastructure_init()` → `initialize_minimal_chassis()` → `minimal_chassis_init()` | `mysql_minimal_chassis`(静态) + `mysql_server`(静态, `"mysql:core"`) | 否（静态注册） |
| ② | `mysqld.cc:7618` | `initialize_manifest_file_components()` → `Deployed_components::load()` | manifest 文件（`mysqld.my`，典型是 keyring） | 否 |
| ③ | `mysqld.cc:7901`→`6191` | `init_server_components()` → `dynamic_loader_srv->load(...)` | `component_urns[]` | **否**（绕过持久层） |
| ④ | `mysqld.cc:8090` | `mysql_component_infrastructure_init()` → `persistent_dynamic_loader_init()` | `mysql.component` 表全部 group | 是（读取来源） |

```cpp
/* mysqld.cc:2022 —— 第 ③ 条链的 URN 表目前只有一个 */
const char *component_urns[] = {"file://component_reference_cache"};
#define NUMBER_OF_COMPONENTS 1

/* mysqld.cc:6191 —— 直接调 dynamic loader，不经 persistent loader */
  if (!is_help_or_validate_option() && !opt_initialize)
    dynamic_loader_srv->load(component_urns, NUMBER_OF_COMPONENTS);
  /* ★ 注释明说：The unload of these components will be done by
       minimal_chassis_deinit(). So, no need to call unload of these components. */
```

**★★ 自举的"鸡生蛋"解法**（`minimal_chassis_init()`，`components/libminchassis/minimal_chassis.cc:148`）：

```cpp
  mysql_registry_imp::init();                       /* ① 建立 registry 内部数据结构 */

  /* Seed the registry through registering the registry implementation into it,
    as well as other main bootstrap dynamic loader service implementations. */
  for (int inx = 0;
       mysql_component_mysql_minimal_chassis.provides[inx].implementation != nullptr;
       ++inx) {
    if (imp_mysql_minimal_chassis_registry_registration.register_service(
            mysql_component_mysql_minimal_chassis.provides[inx].name,
            reinterpret_cast<my_h_service>(
                mysql_component_mysql_minimal_chassis.provides[inx].implementation)))
      return true;                                  /* ②★ 把自己灌进 registry */
  }
  if (registry != nullptr) {
    imp_mysql_minimal_chassis_registry.acquire("registry", &registry_handle);
    *registry = ...;                                /* ③ 拿到自己的句柄交给调用方 */
  }
  mysql_dynamic_loader_imp::init();                 /* ④ dynamic loader 就绪 */
  mysql_dynamic_loader_scheme_file_imp::init();

  if (comp_ref != nullptr) {                        /* ⑤ 把调用方静态链进来的组件（mysqld 传 mysql_server）注册进去 */
    for (int inx = 0; comp_ref->provides[inx].implementation != nullptr; ++inx)
      if (...register_service(comp_ref->provides[inx].name, ...)) return true;
  }
```

> ★★★ **`mysql_minimal_chassis` 用 `BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY`**——它是 registry 本身，绝不能 REQUIRE registry，否则自举死锁。且它的 init/deinit **是 `nullptr`**（`DECLARE_COMPONENT` 后直接 `nullptr, nullptr`），注释解释：*"they will not be called as this component is not a regular one"*。
> ⚠️ `initialize_minimal_chassis()` 里的 `#ifndef WITH_MYSQL_COMPONENTS_TEST_DRIVER / #else` **注释与宏名语义反着写**——CMake（`sql/server_component/CMakeLists.txt:76`）给 **mysqld 本体**定义了这个宏，所以 **`#else` 分支才是 mysqld 走的**（真的调 `minimal_chassis_init()`）。这是 8.0.39 真实存在的注释 bug。

**★ `mysql_minimal_chassis` 提供的 13 个 service**（底盘的全部家当）：

`registry`、`registry_registration`、`registry_query`、`registry_metadata_enumerate`、`registry_metadata_query`、`dynamic_loader`、`dynamic_loader_query`、`dynamic_loader_metadata_enumerate`、`dynamic_loader_metadata_query`、`dynamic_loader_scheme_file`、`mysql_runtime_error`、`mysql_rwlock_v1`、`mysql_psi_system_v1`。

**★ 关机顺序有一条硬约束**（`mysqld.cc:2714`）：

```cpp
  deinitialize_manifest_file_components();   /* 先卸 manifest 组件 */
  component_infrastructure_deinit();          /* 再卸全部 + 拆 registry */
  /*
    component unregister_variable() api depends on system_variable_hash.
    component_infrastructure_deinit() interns calls the deinit function
    of components which are loaded, and the deinit functions can have
    the component system unregister_ variable() api's, hence we need
    to call the sys_var_end() after component_infrastructure_deinit()
  */
  sys_var_end();
```

> ★ 同理 `component_infrastructure_deinit()` 内部：`server_component_deinit()` 必须在 `deinitialize_minimal_chassis()` **之后**——因为卸载组件时组件可能还在用 `mysql_thd_store::unregister_slot()` 这类 server 服务。

**★ `mysql.component` 表的建表语句真身**（⚠️ 在 **`scripts/mysql_system_tables.sql:364`**，不是 `sql/dd/`——`sql/dd/` 下**没有任何** `*component*` 文件，所以它不是 DD 表）：

```sql
SET @cmd = "CREATE TABLE IF NOT EXISTS component ( component_id int unsigned
  NOT NULL AUTO_INCREMENT, component_group_id int unsigned NOT NULL,
  component_urn text NOT NULL, PRIMARY KEY (component_id))
  engine=INNODB DEFAULT CHARSET=utf8mb3 COMMENT 'Components'
  ROW_FORMAT=DYNAMIC TABLESPACE=mysql";
SET @str = CONCAT(@cmd, " ENCRYPTION='", @is_mysql_encrypted, "'");
PREPARE stmt FROM @str; EXECUTE stmt; DROP PREPARE stmt;
```

> ★ 用 `PREPARE/EXECUTE` 拼串，是为了按实例是否加密动态追加 `ENCRYPTION='Y'/'N'`。

**★ 持久层不使用 SQL，直接走 handler 层**（★ 与 `INSTALL PLUGIN` 同构）：

```cpp
      int res = component_table->file->ha_write_row(component_table->record[0]);  /* 写 */
      int res = component_table->file->ha_index_read_idx_map(                     /* 定位 */
          component_table->record[0], 0, key, HA_WHOLE_KEY, HA_READ_KEY_EXACT);
      res = component_table->file->ha_delete_row(component_table->record[0]);     /* 删 */
```

配合 `Component_db_intact`（继承 `Table_check_intact`，只 override `report_error()` 改成**写错误日志**而非推 SQL 错误给客户端）。

**★ `INSTALL COMPONENT` 的权限**：**没有专门的 `COMPONENT_ACL`**——权限是靠**对 `mysql.component` 表的表级 ACL** 兜住的（`open_component_table()` 的 `check_one_table_access(thd, acl_to_check, &tables)`）：

| 语句 | 表级 ACL | 额外锁 | binlog |
|---|---|---|---|
| `INSTALL COMPONENT` | `INSERT_ACL` on `mysql.component` | `acquire_shared_backup_lock()` | `Disable_binlog_guard` 关闭 |
| `UNINSTALL COMPONENT` | `DELETE_ACL` on `mysql.component` | `acquire_shared_backup_lock()` | 关闭 |
| 启动恢复 | `SELECT_ACL` | — | — |

且 `sql_parse.cc:723` 给两条语句都打了 `CF_CHANGES_DATA | CF_AUTO_COMMIT_TRANS` 与 `CF_DISALLOW_IN_RO_TRANS`（**事务内不允许执行**）。

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

**★ URN 的四个"想当然"**（逐个 `DECLARE_COMPONENT` 核实）：

| 想当然 | 真实 URN | 证据 |
|---|---|---|
| `mysql_server` 组件 URN = `"mysql_server"` | **`"mysql:core"`** | `server_component.cc:807` |
| `mysql_minimal_chassis` URN = `"mysql:minimal_chassis"` | **`"mysql_minimal_chassis"`**（无 `mysql:` 前缀） | `minimal_chassis.cc:129` |
| performance_schema 组件 URN = `"mysql:performance_schema"` | **`"mysql:pfs"`** | `pfs.cc:9948` |
| `audit_api_message_emit` URN = `"mysql:audit_api_message_emit"` | **`"audit_api_message_emit"`**（连前缀都没有） | `audit_api_message_emit.cc:597` |

> ★ 所以 **URN 没有统一命名规范**：`mysql_server` 用 `mysql:core`，`mysql_minimal_chassis` 却不带前缀，`audit_api_message_emit` 连 `mysql:` 都没有。URN 只是 map 的一个 key，任何字符串都行（只受 `mysql_service_implementation` 对**服务实现名**"恰好一个点"的约束，URN 本身无约束）。

**★ 两项明确不存在**：

- **`mysql_no_login` 不是组件** —— 它是传统 auth **插件**：`plugin/auth/mysql_no_login.cc`（在 `plugin/` 而非 `components/`）。
- **`mysql_buffer_pool_dump` 不存在** —— 全仓库零命中。
- `mysql_query_attributes` 的真名是 **`query_attributes`**（URN `"mysql:query_attributes"`）；`mysql_server` 侧对应提供的是 `mysql_query_attributes_iterator` / `mysql_query_attribute_string` / `mysql_query_attribute_isnull` 三个服务。

**★ PFS 组件自己的声明**（`storage/perfschema/pfs.cc:9888`，★ 它是独立注册的，不是挂在 mysql_server 里）：

```cpp
DECLARE_COMPONENT(performance_schema, "mysql:pfs")
/* There are no initialization/deinitialization functions, they will not be
   called as this component is not a regular one. */
nullptr, nullptr END_DECLARE_COMPONENT();

bool pfs_init_services(SERVICE_TYPE(registry_registration) * reg) {
  int inx = 0;
  for (;;) {
    auto *pfs_service = reinterpret_cast<my_h_service>(
        mysql_component_performance_schema.provides[inx].implementation);
    if (pfs_service == nullptr) break;
    if (reg->register_service(
            mysql_component_performance_schema.provides[inx].name, pfs_service))
      return true;
```

> ⚠️ 一处不对称：`pfs.cc` 自己 provides 里有 **`psi_thread_v7`**，但 `mysql_server` 转手提供的 psi_thread 只到 **v6**。

### 可观测性与错误码覆盖

**★ 三个"没有"（逐条实证）**：

| 检查项 | 证据 | 结论 |
|---|---|---|
| `SHOW COMPONENTS` | `sql/sql_yacc.yy` 里 `COMPONENT_SYM` 只有两条生产规则：`:18090`(INSTALL)、`:18103`(UNINSTALL) | **不存在** |
| `I_S.COMPONENTS` | `sql/dd/impl/system_views/` 下 `*component*` → **0 个文件** | **不存在** |
| `mysql.component` 是 DD 表？ | `sql/dd/` 整棵树无 component 表定义；建表在 `scripts/mysql_system_tables.sql` | **不是** DD 表 |

> ★★ 所以**唯一可观测入口是 `SELECT * FROM mysql.component`**。此外只有两个状态计数器：`Com_install_component` / `Com_uninstall_component`（`mysqld.cc:4201`/`4488`）。

**★ `components/example/` 的定位**（★ CMake 里每个都标了 `MODULE_ONLY` + `TEST_ONLY`）：

```cmake
MYSQL_ADD_COMPONENT(example_component1
  english_greeting_service_imp.cc example_component1.cc simple_example_math_imp.cc
  MODULE_ONLY TEST_ONLY)
```

> ★ `MODULE_ONLY` = 只编译成 .so、**不进发布包**；`TEST_ONLY` = 仅供 MTR/gUnit。

**★ MTR 主用例覆盖了几乎所有错误码**（`mysql-test/t/component.test`）：

```perl
--error ER_COMPONENTS_CANT_LOAD
INSTALL COMPONENT "file://component_example_component1";   # 重复加载
--error ER_CANT_OPEN_LIBRARY
INSTALL COMPONENT "file://component_example_component4";   # 文件不存在
--error ER_COMPONENTS_CANT_SATISFY_DEPENDENCY
INSTALL COMPONENT "file://component_example_component3";   # 依赖不满足
--error ER_COMPONENTS_NO_SCHEME
INSTALL COMPONENT "file:component_example_component3";     # 缺 "://"
--error ER_COMPONENTS_NO_SCHEME_SERVICE
INSTALL COMPONENT "bad_scheme://component_example_component3";
--error ER_COMPONENTS_CANT_LOAD
INSTALL COMPONENT "file:///component_example_component3";  # 绝对路径被 path filter 挡
--error ER_COMPONENTS_UNLOAD_NOT_LOADED
UNINSTALL COMPONENT "file://component_example_component4";
```

> ★ 后三条正是 `dynamic_loader_scheme_file_path_filter` 的作用证据——`file:///...`、`file://./...`、`file://../...` 全部被拒。

**★ 组件注册系统变量/状态变量的服务真名**：

- 状态变量：服务 **`status_variable_registration`**（实现 `mysql_status_variable_registration_imp`，文件 `sql/server_component/component_status_var_service.cc`）
- 系统变量：**`component_sys_variable_register`** / **`component_sys_variable_unregister`**（实现 `mysql_component_sys_variable_imp`）

> ⚠️ 不存在名为 `component_status_var` / `component_sys_var` 的 *service*——那是实现文件名。

### 插件与组件的互操作：能力单向，数据格式双向

两代机制不是各自封闭，而是有一条明确的桥（全部经源码核实）：

- **插件 → 组件（能力通道）**：插件通过 `plugin_registry_service` 桥拿到 `SERVICE_TYPE(registry)*`，进而消费**全部**组件服务（机制见 [`service.md`](service.md)）。树内实例：`plugin/semisync`、`plugin/version_token`、`plugin/keyring_udf`、`storage/innobase/handler/ha_innodb.cc`（用 `acquire_related` 从同一 keyring 组件取 `keyring_writer`）、`storage/perfschema`、X plugin。
- **组件 → 插件（无能力通道）**：组件**没有**调用插件能力的途径——插件 API 只对服务器暴露，registry 里没有插件 vtable。
- **但数据格式是双向复用的**：`components/logging/log_filter_dragnet.cc`、`log_sink_syseventlog.cc` 与若干 test 组件直接 `#include <mysql/plugin.h>`，用的是里面的 **`SHOW_VAR`**（状态变量结构）——组件经 `status_variable_registration` 服务注册状态变量时，数据格式仍然是插件时代定义的结构体。

结论很直白：**能力只能单向流动（插件可用组件服务，组件用不了插件能力），数据格式却双向复用**。这解释了 8.0 的迁移方向——新能力只能落在组件一侧，插件只能"消费"而不能"提供"。

### 落地形态：同名能力以插件和组件两种形态并存

8.0.39 里同一个功能常常**同时编译出插件版和组件版**，两条安装路径并存。以 `validate_password` 为例（两份 CMake 均已核实）：

| 形态 | 构建声明 | 产物 | 安装方式 |
|---|---|---|---|
| 插件 | `plugin/password_validation/CMakeLists.txt` → `MYSQL_ADD_PLUGIN(validate_password validate_password.cc MODULE_ONLY)` | `validate_password.so` | `INSTALL PLUGIN validate_password SONAME 'validate_password.so'` |
| 组件 | `components/validate_password/CMakeLists.txt` → `MYSQL_ADD_COMPONENT(validate_password ...)` | `component_validate_password.so` | `INSTALL COMPONENT 'file://component_validate_password'` |

组件库名的来源是 `cmake/component.cmake`：`STRING(TOLOWER component_${component} target)` + `SET_TARGET_PROPERTIES(${target} PROPERTIES PREFIX "")` → 目标名 `component_validate_password`、无 `lib` 前缀 → 文件名 `component_validate_password.so`，这就是 `file://` URN 的解析目标（经 path filter 拼到 `plugin_dir` 下，见 `02` 章的 scheme 覆盖）。

**默认走哪条？** 官方工具已经给出答案：`client/mysql_secure_installation.cc` 里的 `install_password_validation_component()`——安装的是**组件**。这就是"插件 → 组件"迁移在运维侧的具象化：同一功能两套产物并存，新代码默认选组件。

keyring 同理（插件 `plugin/keyring/` vs 组件 `components/keyrings/keyring_file/`，URN `file://component_keyring_file`），区别是官方**建议 keyring 组件只用 manifest（`mysqld.my`）加载**，因为 `INSTALL COMPONENT` 落 `mysql.component` 表（InnoDB 表），加载时机晚于 InnoDB 初始化，keyring 会来得太迟。官方手册明确"同一时刻只能启用一个 keyring 组件或插件"——源码层面未找到强制互斥检查，这是**运维约束而非代码约束**（诚实标注）。

---

### 实践：如何写一个组件（清单）

把前面所有机制收成一份可执行的清单（每步都对应本篇某一节）：

1. **定义服务（可选）**：`BEGIN_SERVICE_DEFINITION(name)` + `DECLARE_METHOD/DECLARE_BOOL_METHOD`，放在 `include/mysql/components/services/`（公开）或组件自己的头文件（私有）。
2. **实现服务**：`BEGIN_SERVICE_IMPLEMENTATION(comp, svc)` + 用 `DEFINE_METHOD`（自带 `noexcept`）写函数；方法体包 `try/catch(...)`（错误处理约定见 [`service.md`](service.md)）。
3. **声明组件**：`BEGIN_COMPONENT_PROVIDES` / `REQUIRES` / `METADATA` + `DECLARE_COMPONENT` + `DECLARE_LIBRARY_COMPONENTS`（导出 `list_components`）。
4. **init / deinit**：`is_intialized` 原子标志自己维护；**init 里不要 acquire/release 服务**（见「组件 init/deinit 的两条硬约束」）。
5. **构建**：`MYSQL_ADD_COMPONENT(<name> src ... MODULE_ONLY LINK_LIBRARIES ...)`；产物固定为 `component_<name>.so`（`cmake/component.cmake` 的 `STRING(TOLOWER component_${component} target)` + `PREFIX ""`），并强制 `-fvisibility=hidden`（见 [`abi.md` 实验 2](abi.md)）。
6. **加载**：普通组件 `INSTALL COMPONENT 'file://component_<name>'`；必须在 InnoDB 之前就绪的（keyring）走 manifest（`mysqld.my`）。
7. **卸载**：确保没有外部引用（引用计数判据），否则报 `ER_COMPONENTS_UNLOAD_CANT_UNREGISTER_SERVICE`。
8. **测试**：见下「调试设施」。

### 扩展新的加载 scheme

加载位置与方式由 `dynamic_loader_scheme` 服务解耦，它只有两个方法（`include/mysql/components/services/dynamic_loader.h:282-296`）：

```c
BEGIN_SERVICE_DEFINITION(dynamic_loader_scheme)
DECLARE_BOOL_METHOD(load,  (const char *urn, mysql_component_t **out_data));
DECLARE_BOOL_METHOD(unload, (const char *urn));
END_SERVICE_DEFINITION(dynamic_loader_scheme)
```

URN 的 `scheme://` 前缀决定用哪个实现：`get_scheme_service_from_urn()` 解析前缀 → acquire `<scheme>.<实现名>` → 调它的 `load`。因此**新增一种加载方式 = 实现这个两方法服务并注册**（例如未来可以有 `builtin://` 加载静态链接组件，或某种远程 scheme）。

现状：8.0.39 只有 `file://` 一个 scheme，且被"覆盖"了一次——minimal chassis 提供 `dynamic_loader_scheme_file.mysql_minimal_chassis`（允许任意路径），服务器启动时 `set_default("dynamic_loader_scheme_file.mysql_server_path_filter")` 换成只允许 `plugin_dir` 的实现（见「启动引导三条链」）。这就是 override 能力最实用的一例。

### 调试与测试设施

| 设施 | 位置 | 用途 |
|---|---|---|
| **单元测试** | `unittest/gunit/components/mysql_server/`：`registry-t.cc`、`dynamic_loader-t.cc`、`minimal_chassis-t.cc`、`reference_cache-t.cc` | 直接测 registry / loader 的行为（acquire、default、卸载…） |
| **测试驱动** | `components/libminchassis/gunit_harness/`（`test_harness_suite.h`、`test_harness_component.cc.example`） | 提供"在 minimal chassis 里跑测试组件"的脚手架 |
| **示例组件** | `components/example/`（`example_component1..3`、`test_string_service*.cc`、`test_backup_lock_service.cc` 等） | 官方教学样例，**同时被 gunit 与 MTR 使用**以保证始终可运行 |
| **测试组件** | `components/test/`（约 40 个文件） | 覆盖 sysvar 服务、UDF 注册、敏感变量、状态变量等各个服务 |
| **MTR** | `mysql-test/` 下的 component 相关用例 | 端到端验证 `INSTALL/UNINSTALL COMPONENT` 与持久化 |

`component_implementation.h` 的 Tutorial 明确指向 `components/example`，说明示例组件就是官方认定的"入门模板"。

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

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `sql/server_component/registry.{h,cc}` + `class Registry` | **都不存在**。真身是 **`class mysql_registry_imp`**，在 **`components/libminchassis/registry_imp.h:40`** + `registry.cc`（`sql/server_component/` 下 96 个文件，无 registry） |
| `Registry::register_service()` / `unregister_service()` / `iterate()` | 真名 **`register_service`** / **`unregister`**（不是 `unregister_service`）/ **`iterator_create`**（不是 `iterate`）；且都拆两层：`xxx()` 加 rwlock → `xxx_nolock()` |
| registry 维护"代际链表" | **没有代际链表**。是**两张 `std::map`**：`service_registry`（名字→实现）+ `interface_mapping`（接口指针→实现），比较器 `c_string_less`（按内容 `strcmp`，非按指针） |
| `my_component_registry` = (component, generation ID) 的 pair | **注释已过时/撒谎**。真实类型 `std::map<const char*, std::unique_ptr<mysql_component>, c_string_less>`，**没有 generation ID 字段** |
| `urns_with_gen_list` 存 URN 字符串 | **不是**。是 `std::forward_list<std::vector<mysql_component*>>`——前向链表，每节点是**组件指针的 vector**。"generation" = **一次 `load()` 调用的批次** |
| `add_component_to_urn_list()` / `remove_component_from_urn_list()` | **两个都不存在**（0 命中）。增删都是内联：`emplace_front` / `find_if`+`erase`+`forward_list::remove` |
| `load_do_load_component()` | **不存在** → **`load_do_load_component_by_scheme()`** |
| 加载七阶段 / 卸载也是七阶段 | 加载 7 阶段（注释自证），**卸载是 10 阶段**（多 `get_scheme_services` / `lock_provided_services` / `check_provided_services_reference_count`） |
| 拓扑排序在 libminchassis 里、递归实现 | 是**通用模板 `depth_first_search()`**，在 **`include/depth_first_search.h:34`**；且是**显式栈迭代**（`std::tuple<bool,TVertex>` 二进二出），非递归 |
| 卸载判据"计数恰好等于批内引用数" | **部分证伪**。分两档：**`==0` 直接放行**；`>0` 时才要求等于依赖图里同批依赖数。且建图用 `operator[]`（会插空 vector），检查用 `find()`，语义不对称 |
| `get_service_implementation_reference_count()` 找不到返回 0 | 注释写 0，**代码返回 `(uint64_t)-1`**。因 `>0` 判据恰好无影响 |
| `file://` 是硬编码识别 | **不是**。取 `://` 前缀拼成服务名 `"dynamic_loader_scheme_" + scheme` 后**从 registry acquire** |
| `dynamic_loader_scheme_file` 是唯一 `file://` 实现 | 服务器侧还有 **`dynamic_loader_scheme_file_path_filter`**（`sql/server_component/dynamic_loader_path_filter.cc:56`），注册为同名服务并成为**默认**，强制路径限制在 plugin dir |
| `mysql_dynamic_loader_imp::deinit()` 只销毁锁 | **错**。先加写锁，按 generation 组**逐个走完整卸载链**，最后才 `mysql_rwlock_destroy` |
| `sql/server_component/component_loader.cc` | **不存在**。真名 `sql/server_component/persistent_dynamic_loader.cc`（587 行） |
| 持久层"写 SQL" | **不写 SQL**。直接 handler 层：`ha_write_row` / `ha_index_read_idx_map` + `ha_delete_row`，表由 `open_ltable` 打开 |
| `component_infrastructure_init()` 在 `server_component.cc` | 实际在 **`sql/mysqld.cc:2037`**（static） |
| `component_infrastructure_shutdown()` | **不存在** → **`component_infrastructure_deinit()`**（`mysqld.cc:2142`） |
| `initialize_minimal_chassis()` 在 `minimal_chassis.cc` | 在 **`sql/server_component/server_component.cc:774`** |
| `load_components_at_startup()` / `mysql_load_components()` / `component_load_from_table()` | **三个都不存在**。真名链 `mysql_component_infrastructure_init()` → `persistent_dynamic_loader_init()` → `mysql_persistent_dynamic_loader_imp::init()` |
| 启动引导"三条链" | **实为四条**：静态 chassis → manifest 文件 → 硬编码 `component_urns[]` → `mysql.component` 表。第 3 条**绕过持久层、不写表** |
| `mysql_server` 组件 URN = `"mysql_server"` | **`"mysql:core"`** |
| `mysql_minimal_chassis` URN = `"mysql:minimal_chassis"` | **`"mysql_minimal_chassis"`**（无前缀） |
| performance_schema 组件 URN = `"mysql:performance_schema"` | **`"mysql:pfs"`** |
| `audit_api_message_emit` URN 有 `mysql:` 前缀 | **没有**，就是 `"audit_api_message_emit"` |
| `mysql_no_login` 是内置组件 | 是**插件**（`plugin/auth/mysql_no_login.cc`） |
| `mysql_buffer_pool_dump` | **全仓库不存在** |
| 有 `SHOW COMPONENTS` / `I_S.COMPONENTS` | **都不存在**。只能 `SELECT * FROM mysql.component` |
| `mysql.component` 是 DD 表 | **不是**。建表在 **`scripts/mysql_system_tables.sql:364`**；`sql/dd/` 下无任何 component 文件 |
| `INSTALL COMPONENT` 有专门的 `COMPONENT_ACL` | **没有**。靠 `mysql.component` 表的 `INSERT_ACL` / `DELETE_ACL` 兜住 |
| `BEGIN_SERVICE_DEFINITION` 在 `service_implementation.h` | 在 **`include/mysql/components/service.h:67`**；`service_implementation.h` 只有 `SERVICE_IMPLEMENTATION` / `BEGIN/END_SERVICE_IMPLEMENTATION` / `DEFINE_METHOD` / `DEFINE_BOOL_METHOD` |
| `DEFINE_TYPED_METHOD` | **不存在**（0 命中） |
| `DECLARE_COMPONENT_INIT` / `SET_COMPONENT_VERSION` / `COMPONENT_VERSION` | **都不存在**（0 命中）。init/deinit 是在 `DECLARE_COMPONENT` 后**手写函数指针**再 `END_DECLARE_COMPONENT()` |
| `mysql_service_placeholder_t` | **不存在**。只有 `mysql_service_placeholder_ref_t`；`REQUIRES_SERVICE_PLACEHOLDER(svc)` 是**变量声明**宏，产出 `SERVICE_TYPE(svc) * mysql_service_##svc` |
| `include/mysql/components/component.h` | **不存在** |
| Registry 的迭代器方法叫 `iterate()` | 服务接口侧叫 **`create`/`get`/`next`/`is_valid`/`release`**；实现侧才叫 `iterator_create`/`iterator_get`/…（在 `minimal_chassis.cc:79-84` 绑定） |
| `initialize_minimal_chassis()` 的 `#ifndef`/`#else` 注释 | **注释与宏名语义反着写**。CMake 给 **mysqld 本体**定义了 `WITH_MYSQL_COMPONENTS_TEST_DRIVER`，故 **`#else` 分支才是 mysqld 走的** |

**★ 四个"结构性事实"**

1. **"默认实现"没有任何独立字段——就是往同一张 map 再插一条短名条目。** `register_service_nolock()` 里 `emplace_hint(addition_result.first, imp->service_name_c_str(), imp.get())` 往 **同一张** `service_registry` 再插一条 key=服务名的记录；若已存在则 `emplace` 静默失败，**于是"第一个注册的成为默认"**。注销时靠 `erase` 返回的"下一个迭代器"恰好是同 service 的下一个实现（因为 `c_string_less` 按内容字典序排）来提升新默认。而 `set_default()` 必须"先删后插"——因为 **map 的 key 是 imp 对象内部的 `char*`**，换默认实现会让旧 key 缓冲失效。这三处环环相扣，全部依赖"按内容排序的 map"这一个前提。

2. **`file://` 不是硬编码，而是"拼服务名 → registry acquire"。** 新增 scheme 只需提供名为 `dynamic_loader_scheme_<xxx>` 的服务，无需改 dynamic loader 一行代码。服务器侧正是用这个机制**偷换了默认实现**——注册同名服务 `dynamic_loader_scheme_file.mysql_server_path_filter` 并 `set_default`，把路径强制限制在 plugin dir 内（MTR 里 `file:///`、`file://./`、`file://../` 三条用例就是它的证据）。

3. **自举的解法是"registry 不 REQUIRE registry"。** `mysql_minimal_chassis` 用 `BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY`，init/deinit 填 `nullptr`，然后 `minimal_chassis_init()` 手工把自己的 provides 灌进刚建立的 registry——**因为此时还没有任何"框架"存在，只能自己填自己**。同理 `mysql_server` 组件也是静态注册（`COMPONENT_REF(mysql_server)`）而非 dlopen。

4. **加载严格回滚、卸载 deinit 失败不回滚，这个不对称是刻意的**——注释直言是 "arbitrary decision"：重新 init 在技术上可行，但不能保证组件能重新初始化成功，**半卸载状态比"继续往下卸完"更危险**。所以规律是：**可逆的操作严格回滚（scope_guard），不可逆的操作宁可一路走完**。卸载链里 deinit / release / unregister / unload 四步都遵循同一条。

> 注：`components/libminchassis/*.cc`、`sql/server_component/*.cc` 等文件中的部分 `/* 【注】... */` 与 `★` 注释为本工作区人工批注，非 Oracle 上游原文。

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
- 架构总览（双轨制与迁移进度）见 [`architecture.md`](architecture.md)
- 服务契约与 registry 内部实现见 [`service.md`](service.md)
- 插件体系见 [`plugin.md`](plugin.md)
- ABI 契约专题见 [`abi.md`](abi.md)
- 组件的落地样本：口令强度 [`validate_password.md`](validate_password.md)（组件如何注册系统变量、双实现分派）、密钥管理 [`keyring.md`](keyring.md)
