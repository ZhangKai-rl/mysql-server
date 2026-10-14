# 服务（Service）体系深度解析

> 基于 MySQL 8.0.39 源码，涵盖组件服务（component service）的定义/实现/消费宏、registry 内部实现（注册表、引用计数、default 实现、`acquire_related`）、老版插件服务（plugin service）的 dlsym 注入机制，以及两套机制的对照与设计动机。
>
> **边界**：本篇讲服务契约与 registry 内部；组件如何声明/加载/持久化见 [`component.md`](component.md)；插件体系见 [`plugin.md`](plugin.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

服务（Service）是组件基础设施的最小契约单元：**一个命名的、无状态的、C 函数指针表形式的接口**。一个服务可以有多个实现（`服务名.实现名`，如 `udf_registration.mysql_server`），registry 负责"名字 → 实现对象"的注册、解析（acquire）、引用计数（release）与 default 选择。

MySQL 里存在**两套**服务机制，必须分清：

| | A. 组件服务（新版） | B. 插件服务（老版） |
|---|---|---|
| 头文件 | `include/mysql/components/services/*.h` | `include/mysql/service_*.h` |
| 消费者 | 组件；插件经"桥"间接使用 | 插件（validate_password、rewriter、X plugin 等） |
| 解析方式 | 运行期 registry 查找 + 引用计数 | 加载期 dlsym + **改写插件内指针变量**，一次赋值永不复位 |
| 依赖声明 | `BEGIN_COMPONENT_REQUIRES` 显式声明 | 无 |

### 规模（8.0.39 实测）

- `include/mysql/components/services/` 下 138 个头文件，共 **174 个 `BEGIN_SERVICE_DEFINITION`**；
- 最大的服务提供者是 `mysql_server` 组件：`server_component.cc` 中 155 处 `PROVIDES_SERVICE(mysql_server, ...)`（9 处已注释为 Obsolete），其中约 39 个是 PFS 插桩（`psi_*` / `pfs_plugin_*`，实现在 `storage/perfschema/pfs.cc` 以 `performance_schema` 实现名注册）、1 个 path filter、其余 100+ 由 `sql/server_component/` 的 49 个实现文件提供；
- 老版插件服务只有 17 条（`sql/sql_plugin_services.h` 的 `list_of_services[]`），差一个数量级——"服务化"已经全面碾压老机制。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.1+ | 老版插件服务逐渐累积（`my_plugin_log`、`srv_session` 等） |
| 8.0 | 组件服务体系随 WL#4102 落地；`plugin_registry_service` 桥接两套 |
| 8.0.3~8.0.25 | `psi_thread_v1..v4` 相继引入/废弃——"版本=改名"规则的首次大规模实践 |
| 8.0.22+ | `psi_thread_v4/v5/v6/v7` 多代并存注册，服务总数膨胀到 170+ |
| 8.4/9.x | 新能力一律走服务；老版服务未删除但被架构性架空 |

---

## 理论基础

### 设计思想与权衡

**第零条（第一性原理）：把"能力寻址"从调用方代码里搬进名字。**

三篇的机制在这里收束：插件把**调用点**写死在服务器代码里（`acl_authenticate` 必然去调认证插件的 vtable），组件把**依赖**写进自己的数据（requires 清单），而服务做的是第三件事——把**"谁提供这个能力"**从编译期推迟到运行期，交给一个名字去解析。

```
插件：  acl_authenticate()  ──► 直接调 auth 插件 vtable     （编译期已知调用点）
组件：  requires = { log_builtins }                        （加载期解析依赖）
服务：  acquire("keyring_writer") ──► registry 查名字       （运行期才知道谁提供）
```

一旦能力按名字寻址，六件事就都成为可能（每一件都对应 registry 的一个方法）：**可发现**（`registry_query` 迭代）、**可替换**（`set_default`）、**可组合**（requires 回填）、**可计数**（引用计数 → 卸载安全）、**可并存**（多实现，改名版本化）、**可转让**（`acquire_related` 锁定同一实现者）。

代价同样明确，而且正是这两条代价划定了服务的适用边界：

1. **运行期查找开销**：`acquire` 是一次 map 查找 + 原子加。所以**热路径不走服务**——PFS 插桩每秒百万次，服务器内部仍走全局符号直连（见 [`abi.md` 设计思想 ⑦](abi.md)）。
2. **失去编译期类型安全**：名字是字符串，类型靠 `SERVICE_TYPE(x)` 的**约定**保证。写错名字要到 `acquire` 返回 false 才发现；写错类型则完全没有编译期检查（只能靠 `my_service<T>` 的模板参数手动对齐）。

**1. 为什么是 C 函数指针表，而不是 C++ 虚类？**

`include/mysql/components/service.h` 开篇 `@page PAGE_COMPONENTS_SERVICE` 逐条给出了理由（原文节选）：

> Each Service should work only on basic C types and structures of these types to prevent problems with not fully-defined C++ objects ABI. ... The Services are not versioned - any change to interface must require Service being copied to one with different name before applying changes. ... The Services by itself do not carry any state, all methods are stateless.

对照 C++ 虚类的四宗罪：

| 约束 | 源码原文 | C++ 虚类为何做不到 |
|---|---|---|
| ABI 稳定 | 只许用 C 基础类型 | vtable 布局、name mangling、RTTI、异常传播、STL 布局都不在标准内，跨编译器/跨版本不可靠 |
| 无状态 | 方法全部无状态 | 虚类天然带 `this`，对象布局会被共享 |
| 不透明句柄 | opaque pointer（d-pointer） | 需要 `class X*`，布局暴露给消费者 |
| 版本化 | **改接口必须换名字** | 虚表中间插删方法会错位所有槽位 |

对句柄的约束更极端——`DEFINE_SERVICE_HANDLE` 的注释说：想写 C++ RAII 包装类可以，但**"绝不要用虚函数"（keep binary compatibility）**、**只能有一个成员**（d-pointer）。连包装类都不许虚函数，服务接口本身更不可能。

方法实现侧配套 `DEFINE_METHOD` 强制 `noexcept`——异常不能跨 .so 边界（不同堆/展开表）。

**2. "版本 = 改名"：用名字空间换 ABI 永远向后兼容。**

服务 URN 不含版本号，接口变更就复制一份新名字的服务（`psi_thread_v1 → v2 → ... → v7`），新旧实现**长期并存注册**（`psi_thread_v4/v5/v6/v7` 在 8.0.39 全部注册）。代价：二进制里堆多代接口（这正是 174 个服务里 PSI 占约 1/4 的原因）；收益：老消费者永远不用重编译，registry 无需任何版本协商逻辑。头文件里用注释记录各代弃用时间线（如 `psi_thread_service.h`："Version 4. Introduced in 8.0.22. Deprecated in 8.0.25..."）。

**3. 引用计数决定"能不能卸载"，acquire 语义决定"何时绑定"。**

registry 对每个服务实现维护原子引用计数（`my_ref_counted`）。组件卸载前 `unload_do_check_provided_services_reference_count` 检查：**有外部引用就拒绝卸载**。配套的注释还点明一个反直觉行为："已有引用不会切换到后来注册的新 default 实现——default 只在 acquire 那一刻生效"。即**句柄是快照**：先拿句柄、后换 default，旧句柄照旧指向旧实现。这把"动态替换"和"运行稳定"两件事干净地分开。

**4. 插件 vtable（角色）与服务的分工：`page_components_layering_plugins` 的回答。**

> Plugins do implement plugin APIs and may choose to call plugin service APIs exposed by the server. But in reality they have access to all of the server binary global symbols. ... The above makes plugins an integral part of the server codebase, more specifically of the server component. Plugins are dynamically loadable bits of the server component.

插件类型的 vtable 回答"我是什么角色"（被服务器调用的固定身份）；服务回答"我能提供什么 / 我能用什么"（可发现、可替换、可组合的能力市场）。vtable 做不到 override（服务器写死调用方），做不到插件互调，也不表达依赖。两者互补而非替代。

**5. 老版插件服务：dlsym 指针替换 + 版本塞进指针初值。**

老版服务的"版本协商"是把版本整数**当作指针初值**塞进同一个导出变量（`SERVICE_VERSION *mysql_string_service = (void**)VERSION_mysql_string;`），`plugin_dl_add()` 里 dlsym 找到符号后校验版本、再把真实函数指针结构体地址**直接写回插件内存**——之后永不复位。没有引用计数、没有依赖声明、一次赋值绑定终身。它的价值只剩：存量插件继续可用 + `plugin_registry_service` 桥（插件经它拿到 `SERVICE_TYPE(registry)*`，进而消费全部组件服务）。

### 理论溯源

- **Opaque pointer / d-pointer**（`service.h` 注释直接引用 Wikipedia Opaque pointer 条目）：`DEFINE_SERVICE_HANDLE(name)` = `typedef struct name##_imp *name`，`struct my_h_service_imp` **在源码树里根本没有定义**——布局完全藏进实现 .so。
- **Capability system（能力系统）**：acquire 句柄 + 引用计数 + 拒绝越权卸载，与操作系统 capability/句柄表思想同构。
- **组件模型**：provides/requires + registry + 多实现 default，即 OSGi 服务注册表的 C 化精简版（OSGi 的 service registry 同样有 default/multiple-implementation 语义）。

### 算法与数据结构

- `service_registry`：`std::map`（c_string_less），同时存 `"X.Y"`（实现名）与 `"X"`（服务名 → default 实现）两种 key；`interface_mapping`：`std::map<my_h_service, mysql_service_implementation*>` 反查（release 时句柄 → 对象）。
- `mysql_service_implementation : my_ref_counted, my_metadata`：`add_reference` / `release_reference` 原子计数。
- 读写并发：`LOCK_registry` 读写锁；loader 用 `LOCK_dynamic_loader` 写锁。

---

## 核心实现

### 一个服务的三视角

**定义侧**（`service.h` 宏展开）：

```c
#define BEGIN_SERVICE_DEFINITION(name) typedef struct s_mysql_##name {
#define DECLARE_METHOD(retval, name, args) retval(*name) args
#define DECLARE_BOOL_METHOD(name, args) \
  DECLARE_METHOD(mysql_service_status_t, name, args)
#define END_SERVICE_DEFINITION(name) } SERVICE_TYPE_NO_CONST(name);
```

**★ 四个宏的完整原文**（⚠️ `END_SERVICE_DEFINITION` 是**两行**：收尾同时做 typedef）：

```c
typedef int mysql_service_status_t;         /* ★ 服务"bool"返回值的真实类型：int，0=false，非 0=true */

#define SERVICE_TYPE_NO_CONST(name) mysql_service_##name##_t
#define SERVICE_TYPE(name) const SERVICE_TYPE_NO_CONST(name)   /* ★ SERVICE_TYPE(foo) == const mysql_service_foo_t */

#define BEGIN_SERVICE_DEFINITION(name) typedef struct s_mysql_##name {
#define END_SERVICE_DEFINITION(name) \
  }                                  \
  SERVICE_TYPE_NO_CONST(name);      /* ★ 结构体 tag 是 s_mysql_<name>，typedef 后才是 mysql_service_<name>_t */
#define DECLARE_METHOD(retval, name, args) retval(*name) args
#define DECLARE_BOOL_METHOD(name, args) \
  DECLARE_METHOD(mysql_service_status_t, name, args)
```

```c
#define DEFINE_SERVICE_HANDLE(name) typedef struct name##_imp *name
```

> ★★ **`my_h_service` 的定义就在这里**：`DEFINE_SERVICE_HANDLE(my_h_service)` → `typedef struct my_h_service_imp *my_h_service`——而 `struct my_h_service_imp` **在源码树里根本没有定义**（opaque pointer）。所有 `my_h_service` 之间的转换都靠 `reinterpret_cast`。

**★ `DEFINE_SERVICE_HANDLE` 的注释原文**（★ "不要用虚函数"与"只有一个成员"的出处，原话与常见转述不同）：

```
  Defines an object type that is meant for carrying handles to the
  implementation-specific objects used in the Service interface, but without
  their structure exposed, keeping it as an abstraction. This follows a Opaque
  Pointer design pattern, see more: https://en.wikipedia.org/wiki/Opaque_pointer
  If you would like to have a C++ RAII class to manage the resource with
  additional methods to call raw service calls, please create your class
  {handle_name} instead of following macro. Please make sure it does not use any
  virtual methods to keep binary compatibility, and try use only one member, the
  d-pointer to hide all implementation details and keep headers unmodified from
  the point of publishing it.
```

> ⚠️ 原文是 *"does not use any **virtual methods**"* 与 *"try use only one member, the **d-pointer**"*——不是常见转述的 "never use virtual functions"。

**★ `@page PAGE_COMPONENTS_SERVICE` 的四段散文**（⚠️ **不是"四条编号理由"**，是连续四段；`service.h:36-65`）：

> The Service is basic concept of the Components subsystem. A Service is a named, well-defined stateless interface to one functionality, expressed as a list of pointers to a specific methods. ... **Each Service should work only on basic C types and structures of these types to prevent problems with not fully-defined C++ objects ABI.** ... **The Services are not versioned - any change to interface must require Service being copied to one with different name before applying changes.** **The Services by itself do not carry any state, all methods are stateless.** This does not prevent one from having some state-carrying objects created and returned as handles to them. ...
>
> For any specific Service a Service Implementation is defined as a structure of the Service type filled with pointers to methods of specified implementation. **The name of the Service Implementation is a name of the Service and the implementation related name separated with a dot.** ... **Each Service can have several Service Implementations.**

**★ 实现侧宏全文**（`service_implementation.h`）：

```c
#define SERVICE_IMPLEMENTATION(component, service) imp_##component##_##service
#define BEGIN_SERVICE_IMPLEMENTATION(component, service) \
  SERVICE_TYPE(service) SERVICE_IMPLEMENTATION(component, service) = {
#define END_SERVICE_IMPLEMENTATION() }
#define DEFINE_METHOD(retval, name, args) retval name args noexcept
#define DEFINE_BOOL_METHOD(name, args) \
  DEFINE_METHOD(mysql_service_status_t, name, args)
```

> ★ `DEFINE_METHOD` 追加的 **`noexcept`** 是服务实现跨 ABI 边界的硬性要求——异常绝不能越过 .so。

**★ 消费侧宏全文**（`component_implementation.h`）：

```c
/* 提供侧 */
#define BEGIN_COMPONENT_PROVIDES(name) \
  static struct mysql_service_ref_t __##name##_provides[] = {
#define PROVIDES_SERVICE(component, service)                            \
  {                                                                     \
#service "." #component,                                            \
        const_cast < void *>                                            \
            ((const void *)&SERVICE_IMPLEMENTATION(component, service)) \
  }                                                                     /* ★★ 注册名是 "<service>.<component>"，service 在前 */
#define END_COMPONENT_PROVIDES() \
  { NULL, NULL }                 \
  }

/* 消费侧 */
#define REQUIRES_SERVICE_PLACEHOLDER(service) \
  SERVICE_TYPE(service) * mysql_service_##service
#define REQUIRES_SERVICE(service)                                              \
  {                                                                            \
#service,                                                                  \
        static_cast < void **>                                                 \
            (static_cast <void *>(const_cast <mysql_service_##service##_t **>( \
                &mysql_service_##service)))                                    \
  }
#define SERVICE_PLACEHOLDER(service) mysql_service_##service

/* ★★ 每个组件自动自带 registry 依赖，且它是 requires 数组的第一项 */
#define BEGIN_COMPONENT_REQUIRES(name)                                    \
  REQUIRES_SERVICE_PLACEHOLDER(registry);                                 \
  static struct mysql_service_placeholder_ref_t __##name##_requires[] = { \
      REQUIRES_SERVICE(registry),
#define END_COMPONENT_REQUIRES() \
  { NULL, NULL }                 \
  }
```

> ★★ **`BEGIN_COMPONENT_REQUIRES` 不止"开个结构体"**——它先插入 `REQUIRES_SERVICE_PLACEHOLDER(registry)`，再把 `REQUIRES_SERVICE(registry)` 作为数组**第一项**。另有一个 `BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY`（`mysql_minimal_chassis` 用它，因为"它就是 registry 本身"）。
> ⚠️ **不存在 `PROVIDES_SERVICE_METHOD` / `PROVIDES_CUSTOM_SERVICE`**（后者只在注释里被提及一次，无 `#define`）。存在的是 `REQUIRES_SERVICE_AS` / `REQUIRES_SERVICE_PLACEHOLDER_AS` / `SERVICE_PLACEHOLDER`。

**★ `DECLARE_COMPONENT` / `DECLARE_LIBRARY_COMPONENTS`**：

```c
#define DECLARE_COMPONENT(source_name, name)                        \
  mysql_component_t mysql_component_##source_name = {               \
      name, __##source_name##_provides, __##source_name##_requires, \
      __##source_name##_metadata,                                   /* ★ 结尾逗号：init/deinit 函数指针由调用方紧跟其后填写 */
#define END_DECLARE_COMPONENT() }

#define DECLARE_LIBRARY_COMPONENTS \
  mysql_component_t *library_components_list = {
#define END_DECLARE_LIBRARY_COMPONENTS              \
  }                                                 \
  ;                                                 \
  DLL_EXPORT mysql_component_t *list_components() { \
    return library_components_list;                 \
  }                                                 /* ★★ .so 的唯一导出入口函数 */
#define COMPONENT_REF(name) mysql_component_##name
#define COMPONENT_ENTRY_FUNC "list_components"
```

```c
#if defined(_MSC_VER)
#define DLL_EXPORT extern "C" __declspec(dllexport)
#else
#define DLL_EXPORT extern "C" __attribute__((visibility("default")))
#endif   /* ★ C++ 下必须 extern "C"：MSVC 对变量也做 C++ mangling */
```

以 `registry` 服务为例（`services/registry.h`）：

```c
BEGIN_SERVICE_DEFINITION(registry)
DECLARE_BOOL_METHOD(acquire, (const char *service_name, my_h_service *out_service));
DECLARE_BOOL_METHOD(acquire_related, (const char *service_name, my_h_service service,
                                      my_h_service *out_service));
DECLARE_BOOL_METHOD(release, (my_h_service service));
END_SERVICE_DEFINITION(registry)
```

展开成 `typedef struct s_mysql_registry { mysql_service_status_t (*acquire)(...); ... } mysql_service_registry_t;`。`SERVICE_TYPE(name)` = `const` 版（服务实现对象是 const 的静态函数指针表）。

**实现侧**（`service_implementation.h`）：`BEGIN_SERVICE_IMPLEMENTATION(component, service)` 生成 `const SERVICE_TYPE(service) imp_<component>_<service> = { ... }`；函数必须用 `DEFINE_METHOD` 写（追加 `noexcept`）。

**消费侧**（`component_implementation.h`）：`REQUIRES_SERVICE_PLACEHOLDER(foo)` 定义全局指针 `const mysql_service_foo_t *mysql_service_foo;`；`REQUIRES_SERVICE(foo)` 生成 `{ "foo", (void**)&mysql_service_foo }` 进 requires 数组，由 loader acquire 后回填（见 02）。组件代码里直接 `mysql_service_foo->method(...)`。

**C++ RAII**（`include/mysql/components/my_service.h` 的 `my_service<T>` 模板）：构造时 `acquire`，析构自动 `release`，重载 `->` / 隐式转换。真实用法遍地都是：

```cpp
my_service<SERVICE_TYPE(persistent_dynamic_loader)> persisted_loader(
    "persistent_dynamic_loader", srv_registry);
```

### registry 内部：注册、default、acquire_related

`components/libminchassis/registry.cc`（`mysql_registry_imp`）：

- **注册**（`register_service`）：`emplace("X.Y", imp)`；若 `"X"` 尚无 default，`emplace_hint(..., "X", imp)`——**先注册者自动成为 default**。

**★ 文件头的官方口径**（`registry.cc:37`，`@page PAGE_COMPONENTS_REGISTRY`）——"先注册者成为 default"的权威依据：

```
  @page PAGE_COMPONENTS_REGISTRY The Service Registry Service
  The Service Registry is a central part of Components subsystem. It maintains
  a list of the Service Implementations and allow acquiring them by any actor
  that has a reference to main Registry Service Implementation. For each Service
  a reference counter is maintained to make sure no one can remove any Service
  Implementation being actively used.
  ...
  For each Service a default Service Implementation is distinguished, the first
  one to be added, but it is possible to change it at will at any point.
```

**★★ `register_service_nolock()` 全文**（★ "先注册者成为 default"的全部实现，就靠 `emplace_hint` 的静默失败）：

```cpp
bool mysql_registry_imp::register_service_nolock(
    const char *service_implementation_name, my_h_service ptr) {
  try {
    std::unique_ptr<mysql_service_implementation> imp =
        std::unique_ptr<mysql_service_implementation>(
            new mysql_service_implementation(ptr, service_implementation_name));

    if (imp->interface() == nullptr) {
      return true;   /* ★ 构造失败的唯一信号：名字必须"有且仅有一个 '.' 且两部分非空" */
    }

    /* Register the implementation name. */
    std::pair<my_service_registry::iterator, bool> addition_result =
        mysql_registry_imp::service_registry.emplace(imp->name_c_str(), imp.get());
    /*  ★ 第 1 次 emplace：key = "服务名.实现名" 全限定名 */

    if (!addition_result.second) {
      return true;   /* ★ 全限定名重复 → 直接失败，不覆盖 */
    } else {
      try {
        /* Register interface in mapping */
        mysql_registry_imp::interface_mapping.emplace(imp->interface(), imp.get());
        /*  ★ 反查表：接口指针 → 实现对象，release() 靠它 */

        /* Register the Service Implementation as default for Service name in
          case none were registered before. */
        mysql_registry_imp::service_registry.emplace_hint(
            addition_result.first, imp->service_name_c_str(), imp.get());
        /*  ★★★ 第 2 次（emplace_hint）：用裸服务名再插一条。
            若该服务名已存在，std::map::emplace_hint 静默失败、不覆盖 ——
            这就是「先注册者自动成为 default」的全部实现，没有任何 if / 比较 */
      } catch (...) {
        mysql_registry_imp::service_registry.erase(addition_result.first);  /* 回滚 */
        return true;
      }
    }
    imp.release();   /* unique_ptr 放弃所有权，对象交给 registry 以裸指针持有 */
    return false;
  } catch (...) {}
  return true;
}
```

**★ `set_default()` 全文**（★ 为什么必须"先 erase 再 emplace_hint"）：

```cpp
DEFINE_BOOL_METHOD(mysql_registry_imp::set_default,
                   (const char *service_implementation_name)) {
  try {
    my_service_registry::const_iterator iter;
    minimal_chassis::rwlock_scoped_lock lock(&mysql_registry_imp::LOCK_registry,
                                             true, __FILE__, __LINE__);  /* 写锁 */

    /* register the implementation name */
    iter = mysql_registry_imp::service_registry.find(service_implementation_name);
    /*  ★ find 的是全限定实现名；传裸服务名会在这里失败 */
    if (iter == mysql_registry_imp::service_registry.cend()) return true;

    mysql_service_implementation *imp = iter->second;
    /* We have to remove and reinsert value as key, the string pointer will
      not be valid if we unregister previous default implementation. */
    /*  ★★ 原因：map 的 key 是 const char*，指向「旧 default 实现的 m_service.c_str()」。
        旧实现一旦被 unregister 就析构，这个 key 指针就悬空。 */
    iter = mysql_registry_imp::service_registry.erase(
        mysql_registry_imp::service_registry.find(imp->service_name_c_str()));
    mysql_registry_imp::service_registry.emplace_hint(
        iter, imp->service_name_c_str(), imp);
    return false;
  } catch (...) {}
  return true;
}
```

**★ `unregister_nolock()` 关键段**（★ "顺位接任"靠 map 排序的相邻性）：

```cpp
      my_service_registry::iterator imp_iter =
          mysql_registry_imp::service_registry.find(service_implementation_name);
      if (imp_iter == mysql_registry_imp::service_registry.end() ||
          imp_iter->second->get_reference_count() > 0) {
        return true;    /* ★ 引用计数 > 0 → 拒绝注销 */
      }
      imp.reset(imp_iter->second);
      mysql_registry_imp::service_registry.erase(imp_iter);
      /*  ★ 先摘掉实现名条目，这样后续扫描不会把它自己当成新 default */

    mysql_registry_imp::interface_mapping.erase(
        mysql_registry_imp::interface_mapping.find(imp->interface()));

    my_service_registry::iterator default_iter =
        mysql_registry_imp::service_registry.find(imp->service_name_c_str());
    if (default_iter == mysql_registry_imp::service_registry.end()) {
      /* A Service Implementation and no default present. The state is not
        consistent. */
      return true;
    }

    if (default_iter->second == imp.get()) {
      my_service_registry::iterator new_default_iter =
          mysql_registry_imp::service_registry.erase(default_iter);
      /*  ★ erase 返回被删元素的后继迭代器 */

      if (new_default_iter != mysql_registry_imp::service_registry.end() &&
          !strcmp(imp->service_name_c_str(),
                  new_default_iter->second->service_name_c_str())) {
        /*  ★★ map 按 c_string_less（strcmp）排序，"X" 与 "X.foo" 必然相邻；
            被删 default 的后继若服务名相同，就是同服务的下一个实现 → 顺位接任 */
        mysql_service_implementation *new_default = new_default_iter->second;
        mysql_registry_imp::service_registry.emplace_hint(
            new_default_iter, new_default->service_name_c_str(), new_default);
      }
    }
```

**★ `acquire_nolock()` / `release_nolock()` 全文**：

```cpp
bool mysql_registry_imp::acquire_nolock(const char *service_name,
                                        my_h_service *out_service) {
  try {
    if (out_service == nullptr) return true;
    my_service_registry::const_iterator iter;
    iter = mysql_registry_imp::service_registry.find(service_name);
    /*  ★ 裸服务名命中 default 条目；全限定名命中实现条目 —— 同一张 map，两种 key 共存 */
    if (iter == mysql_registry_imp::service_registry.cend()) return true;

    mysql_service_implementation *imp = iter->second;
    imp->add_reference();                 /* ★ std::atomic<uint64> m_count++ */
    *out_service = imp->interface();      /* ★★ 返回的"句柄"就是实现结构体指针 m_interface 本身 */
    return false;
  } catch (...) {}
  return true;
}

bool mysql_registry_imp::release_nolock(my_h_service service) {
  try {
    if (service == nullptr) return true;
    mysql_service_implementation *service_implementation =
        mysql_registry_imp::get_service_implementation_by_interface(service);
    if (service_implementation == nullptr) return true;
    return service_implementation->release_reference();   /* CAS 减 1；已为 0 时返回 true（失败） */
  } catch (...) {}
  return true;
}
```

**★ 引用计数基类 `my_ref_counted` 全文**（`components/libminchassis/my_ref_counted.h` + `.cc`）：

```cpp
class my_ref_counted {
 public:
  my_ref_counted();
  my_ref_counted(my_ref_counted &other);
  virtual ~my_ref_counted();            /* ★ virtual 析构：被 mysql_service_implementation 多继承地作基类 */
  uint64 add_reference();
  bool release_reference(uint64 *new_count = nullptr);
  uint64 get_reference_count() const;
 private:
  std::atomic<uint64> m_count;          /* ★ 成员真名是 m_count */
};
```

```cpp
uint64 my_ref_counted::add_reference() { return m_count++; }   /* ★ 返回自增前的旧值 */

bool my_ref_counted::release_reference(uint64 *new_count) {
  uint64 old_val = m_count;
  do {
    if (old_val == 0) return true;      /* ★ 减到 0 后再 release 会失败 —— 重复 release 的保护 */
  } while (!m_count.compare_exchange_weak(old_val, old_val - 1));   /* ★ CAS 循环，无锁 */
  if (new_count != nullptr) *new_count = old_val - 1;
  return false;
}
```

**★ 两张 map 与 `mysql_service_implementation` 类**：

```cpp
typedef std::map<const char *, mysql_service_implementation *, c_string_less>
    my_service_registry;      /* ★ key 是 const char*（裸指针！），靠 c_string_less 按内容而非地址排序 */

class mysql_registry_imp {
  typedef std::map<my_h_service, mysql_service_implementation *>
      my_interface_mapping;   /* ★ 第二张 map：接口指针 → 实现对象，按"指针地址"比大小 */

  static my_service_registry service_registry;      /* ★ 真名 service_registry（不是 m_registry） */
  static my_interface_mapping interface_mapping;
  static mysql_rwlock_t LOCK_registry;              /* ★ private static 成员 */
  ...
};
```

```cpp
class mysql_service_implementation : public my_ref_counted, public my_metadata {
 public:
  mysql_service_implementation(my_h_service interface, const char *full_name);
  const char *service_name_c_str() const;   /* 裸服务名 */
  const char *name_c_str() const;           /* ★ 完整 "<service>.<component>" 名 */
  my_h_service interface() const;           /* ★ 返回 m_interface */
 private:
  my_h_service m_interface;
  my_string m_service;
  my_string m_full_name;
};
```

```cpp
/** 构造函数里的名字合法性校验 + m_service 的切分 */
  first_dot = m_full_name.find_first_of('.');
  if (m_full_name.length() < 3 ||      /* dot and at least 1 char for each part */
      first_dot < 1 || first_dot >= m_full_name.length() - 1 ||
      first_dot != m_full_name.find_last_of('.')) {   /* ★ 有且仅有一个点 */
    m_interface = {};    /* 构造失败的标志 */
    return;
  }
  m_service = my_string(m_full_name.begin(), m_full_name.begin() + first_dot);
  /*  ★★ m_service.c_str() 就是 registry map 里 default 条目的 key 指针 ——
      这正是 set_default 必须先 erase 再 emplace 的根源 */
```

**★ `acquire_related()` 全文**（★ 真名 `strchr`，且**要求入参不含点**）：

```cpp
DEFINE_BOOL_METHOD(mysql_registry_imp::acquire_related,
                   (const char *service_name, my_h_service service,
                    my_h_service *out_service)) {
  try {
    minimal_chassis::rwlock_scoped_lock lock(&mysql_registry_imp::LOCK_registry,
                                             false, __FILE__, __LINE__);   /* 读锁 */
    mysql_service_implementation *service_implementation =
        mysql_registry_imp::get_service_implementation_by_interface(service);
    if (service_implementation == nullptr) return true;

    /* Find dot, the component name is right after the dot. */
    const char *component_part =
        strchr(service_implementation->name_c_str(), '.');
    /*  ★★ strchr 找第一个 '.'；component_part 形如 ".mysql_server"（含点） */
    if (component_part == nullptr) return true;

    /* Assure given service_name is not fully qualified. */
    if (strchr(service_name, '.') != nullptr) {
      return true;        /* ★★ 入参必须是「裸服务名」，带点直接拒绝 */
    }
    my_string service_implementation_name =
        my_string(service_name) + component_part;
    /*  ★ "keyring_writer" + ".mysql_server" = "keyring_writer.mysql_server" */

    if (mysql_registry_imp::acquire_nolock(service_implementation_name.c_str(),
                                           out_service)) {
      return true;
    }
    return false;
  } catch (...) {}
  return true;
}
```

**★ `get_service_implementation_reference_count()` 的一个真实 bug**（注释说返回 0，实际返回 -1）：

```cpp
uint64_t mysql_registry_imp::get_service_implementation_reference_count(
    my_h_service interface) {
  my_interface_mapping::const_iterator iter =
      mysql_registry_imp::interface_mapping.find(interface);
  if (iter == mysql_registry_imp::interface_mapping.cend()) {
    return -1;    /* ★★ 注释写的是 "Returns 0"，实际 return -1 → (uint64_t)0xFFFFFFFFFFFFFFFF */
  }
  return iter->second->get_reference_count();
}
```

> ★★ 这个 -1 是**有意的**：卸载链里"引用了未注册接口"必然报错，就是靠它。

**★ 卸载链的引用计数闸门**（`dynamic_loader.cc:1151`）——"引用计数必须**恰好等于**批内依赖数"的原文：

```cpp
bool mysql_dynamic_loader_imp::
    unload_do_check_provided_services_reference_count(...) {
  for (mysql_component *component : components_to_unload) {
    for (const mysql_service_ref_t *service : component->get_provided_services()) {
      uint64_t reference_count =
          mysql_registry_imp::get_service_implementation_reference_count(
              reinterpret_cast<my_h_service>(service->implementation));
      if (reference_count > 0) {
        std::map<const void *, std::vector<mysql_component *>>::const_iterator
            it = dependency_graph.find(service->implementation);
        if (it == dependency_graph.end() ||
            reference_count != it->second.size()) {   /* ★★ 核心判据 */
          mysql_error_service_printf(
              ER_COMPONENTS_UNLOAD_CANT_UNREGISTER_SERVICE, MYF(0),
              service->name, component->name_c_str());
          return true;
        }
      }
    }
  }
  return mysql_dynamic_loader_imp::unload_do_deinitialize_components(
      components_to_unload, scheme_services);
}
```

| 卸载链 `unload_do_*` | 定义在 | 作用 |
|---|---|---|
| `unload_do_list_components` | `dynamic_loader.cc:961` | 按 URN 查表，未加载/重复即报错 |
| `unload_do_topological_order` | `:1002` | 建依赖图 + DFS 后序拓扑排序 |
| `unload_do_get_scheme_services` | `:1087` | 在拿 registry 锁**之前**预取各 scheme 加载服务 |
| **`unload_do_lock_provided_services`** | `:1121` | **拿 registry 写锁**（`lock_registry_for_write()` 的唯一调用点） |
| `unload_do_check_provided_services_reference_count` | `:1151` | **引用计数闸门** |
| `unload_do_deinitialize_components` | `:1196` | 调各组件 `deinit()` |
| `unload_do_unload_dependencies` | `:1233` | `release_nolock()` 释放依赖句柄 |
| `unload_do_unregister_services` | `:1271` | `unregister_nolock()` 注销服务 |
| `unload_do_unload_components` | `:1312` | 摘除 + 调 scheme 服务 `unload()`（真正 dlclose） |
| `unload_do_commit` | `:1381` | **空实现** `return false;`（无回滚逻辑） |

> ★★ **整条 `unload_do_*` 链都在 registry 写锁之下运行**（第 4 步拿到后一直持有），所以后续各步都调 `*_nolock` 版本。这是"卸载是整批事务"的实现方式。
> ⚠️ 加载链则不同：`load()` 只加 `LOCK_dynamic_loader`，registry 锁由 `register_service()` 自己加（走**带锁版**）。
- **注销**（`unregister`）：引用计数 > 0 直接拒绝（`get_reference_count() > 0` 返回 true）；若注销的是 default，则在 map 里**顺位找一个同服务名的实现接任 default**。
- **切换 default**（`set_default`）：erase 旧 `"X"` key → `emplace_hint` 新 `"X"` → 同一实现对象（string 指针的寿命由实现对象管理，必须先 erase 再插）。启动代码用它把 `dynamic_loader_scheme_file` 的默认实现从 minimal chassis 换成 mysql_server path filter。
- **acquire**：`service_registry.find(name)`（传 `"X"` 命中 default，传 `"X.Y"` 命中具体实现）→ `add_reference()` → 返回 `imp->interface()`（即 `my_h_service` 句柄）。
- **`acquire_related`**（语义常被误解为"取另一个版本"，实为**取同一实现者的另一个服务**）：

```c
/* 由已 acquire 的句柄反查实现对象 */
mysql_service_implementation *service_implementation =
    mysql_registry_imp::get_service_implementation_by_interface(service);
/* 截出实现名 ".Y" */
const char *component_part = strchr(service_implementation->name_c_str(), '.');
/* 要求新名字不含 '.'，拼成 "<service_name>.Y" 再普通 acquire */
```

典型场景：InnoDB 已拿到 `keyring_reader_with_status`（某个实现），用 `acquire_related("keyring_writer", h_reader, ...)` 保证写入和读取**来自同一个 keyring 组件**，而不是 default 可能指向的另一个实现。

### 老版插件服务：17 条服务的注入

`sql/sql_plugin_services.h` 定义 `struct st_service_ref { name, version, service }` 与 `list_of_services[]`（17 条：`srv_session_service`、`command_service`、`thd_alloc_service`、`mysql_string_service`、`security_context_service`、`mysql_keyring_service`、`plugin_registry_service`……）。注入发生在 `plugin_dl_add()`（插件 .so 加载时）：

```c
for (i = 0; i < array_elements(list_of_services); i++) {
  if ((sym = dlsym(plugin_dl.handle, list_of_services[i].name))) {
    uint ver = (uint)(intptr_t) * (void **)sym;      /* 版本整数塞在指针初值里 */
    if ((*(void **)sym) != list_of_services[i].service &&
        (ver > list_of_services[i].version ||
         (ver >> 8) < (list_of_services[i].version >> 8)))
      ... "service '%s' interface version mismatch" ...
    *(void **)sym = list_of_services[i].service;      /* 直接改写插件内指针 */
  }
}
```

**★ 完整原文**（`sql/sql_plugin.cc:741`）：

```c
  /* link the services in */
  for (i = 0; i < array_elements(list_of_services); i++) {
    if ((sym = dlsym(plugin_dl.handle, list_of_services[i].name))) {
      uint ver = (uint)(intptr_t) * (void **)sym;   /* ★ 读出插件当前存的值：初值是版本号 0x0100 */
      if ((*(void **)sym) !=
              list_of_services[i].service &&      /* ★ "already replaced"：幂等保护 */
          (ver > list_of_services[i].version ||
           (ver >> 8) < (list_of_services[i].version >> 8))) {
        /*  ★★ 版本校验两段式：
            (a) ver > 服务端版本        → 插件要求的比服务器提供的还新 → 不兼容
            (b) (ver>>8) < (服务端>>8)  → 主版本号（高字节）小于服务器 → 不兼容
            即：主版本必须相等，从版本插件侧不能超前 */
        char buf[MYSQL_ERRMSG_SIZE];
        snprintf(buf, sizeof(buf), "service '%s' interface version mismatch",
                 list_of_services[i].name);
        mysql_mutex_unlock(&LOCK_plugin);
        mysql_rwlock_unlock(&LOCK_system_variables_hash);
        report_error(report, ER_CANT_OPEN_LIBRARY, dlpath, 0, buf);
        return nullptr;   /* ★ 失败即整体拒绝加载，不会"部分注入后继续" */
      }
      *(void **)sym = list_of_services[i].service;
      /*  ★★ 注入：把插件内那个"存着版本号的 void* 变量"就地改写成服务端 handler 的地址。
          这是整个老版服务机制的唯一一行"魔法"。 */
    }
  }
```

> ★ dlsym 找不到符号**不是错误**，只是"该插件用不了这个服务"——静默跳过。

**★ `struct st_service_ref` 与 17 条 `list_of_services[]` 全文**：

```c
struct st_service_ref {
  const char *name;    /* ★ 插件导出的服务指针符号名（dlsym 的目标） */
  uint version;        /* ★ 服务端提供的版本号 */
  void *service;       /* ★ 服务端侧真实结构体指针 */
};

static struct st_service_ref list_of_services[] = {
    {"srv_session_service", VERSION_srv_session_service, &srv_session_service_handler},
    {"command_service", VERSION_command, &command_handler},
    {"srv_session_info_service", VERSION_srv_session_info_service, &srv_session_info_handler},
    {"thd_alloc_service", VERSION_thd_alloc, &thd_alloc_handler},
    {"thd_wait_service", VERSION_thd_wait, &thd_wait_handler},
    {"my_thread_scheduler_service", VERSION_my_thread_scheduler, &my_thread_scheduler_handler},
    {"my_plugin_log_service", VERSION_my_plugin_log, &my_plugin_log_handler},
    {"mysql_string_service", VERSION_mysql_string, &mysql_string_handler},
    {"mysql_malloc_service", VERSION_mysql_malloc, &mysql_malloc_handler},
    {"mysql_password_policy_service", VERSION_mysql_password_policy, &mysql_password_policy_handler},
    {"mysql_parser_service", VERSION_parser, &parser_handler},
    {"rpl_transaction_ctx_service", VERSION_rpl_transaction_ctx_service, &rpl_transaction_ctx_handler},
    {"transaction_write_set_service", VERSION_transaction_write_set_service, &transaction_write_set_handler},
    {"security_context_service", VERSION_security_context_service, &security_context_handler},
    {"mysql_locking_service", VERSION_locking_service, &locking_service_handler},
    {"mysql_keyring_service", VERSION_mysql_keyring_service, &mysql_keyring_handler},
    {"plugin_registry_service", VERSION_plugin_registry_service, &plugin_registry_handler},
};
```

> ⚠️ **`VERSION_clone_protocol_service`（`service_versions.h:51`）没有对应的 `list_of_services` 条目**，也没有 `libservices/clone_protocol_service.c`——clone protocol 走另一套。这是最容易误报成"18 条"的地方。
> ★ 服务端 handler 结构体的**成员顺序必须与老版服务头文件里的声明顺序完全一致**——顺序错位即 ABI 崩溃（例：`mysql_string_handler` 的 9 个成员严格对应 `service_mysql_string.h` 的 9 个声明）。

**★ `SERVICE_VERSION` 是宏不是 typedef**——这是"版本塞进指针初值"能成立的技术前提：

```c
#ifdef _WIN32
#define SERVICE_VERSION __declspec(dllexport) void *
#else
#define SERVICE_VERSION void *
#endif
#define VERSION_mysql_string 0x0100
#define VERSION_security_context_service 0x0101   /* ★ 唯一 minor≠0 的 */
#define VERSION_plugin_registry_service 0x100     /* ★ 写法不同（3 位 vs 4 位），数值相同 */
```

```c
/* libservices/mysql_string_service.c —— 插件侧的占位桩，打进 mysqlservices 静态库 */
SERVICE_VERSION *mysql_string_service = (void **)VERSION_mysql_string;
/*  ★ Linux 展开：void **mysql_string_service = (void **)0x0100;
    Windows 展开：__declspec(dllexport) void **... （桩符号必须导出，dlsym 才找得到） */
```

> ★★ **注入前的危险性**：若插件没被 `plugin_dl_add` 处理过（例如被测试 harness 或非 mysqld 的可执行文件加载），`mysql_string_service` 就是 `0x0100`，任何宏展开的调用都会去解引用地址 `0x0100` → **立即段错误**。这就是为什么老版服务"只能由 mysqld 通过 `plugin_dl_add` 加载"是硬性前提。
> ⚠️ `libservices/CMakeLists.txt` 的 `MYSQLSERVICES_SOURCES` 恰好 **17 个** `.c`，与 `list_of_services[]` 一一对应。

**★ `@page page_ext_plugin_svc_anathomy` 的官方口径**（`sql/sql_plugin.cc:227`）——"永不复位"的权威证据：

```
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
  never reset back to service version.
```

> ★★ `free_plugin_mem()` 全文**没有任何一行碰 `list_of_services` 或 `dlsym`**（只 `dlclose` + `my_free`）；`plugin_dl_del()` 只减 ref_count。更根本的是——`struct st_plugin_dl` **里根本没有服务字段**（只有 `dl`/`handle`/`plugins`/`version`/`ref_count`），服务端侧不保存任何"插件服务指针地址"的反向记录，**想复位也没地方记**。

**★ `MYSQL_DYNAMIC_PLUGIN` 下三种宏包装形态并存**：

| 形态 | 示例 | 出处 |
|---|---|---|
| A. 带参宏，成员带 `_type` | `#define mysql_string_free(h) mysql_string_service->mysql_string_free_type(h)` | `service_mysql_string.h:79` |
| B. 带参宏，成员带 `_func` | `#define thd_alloc(thd, size) (thd_alloc_service->thd_alloc_func((thd), (size)))` | `service_thd_alloc.h:60` |
| C. **无参对象宏**（名字借用） | `#define my_plugin_log_message my_plugin_log_service->my_plugin_log_message` | `service_my_plugin_log.h:58` |

```c
/* service_mysql_string.h —— 完整转换链的第①步 */
#ifdef MYSQL_DYNAMIC_PLUGIN
#define mysql_string_convert_to_char_ptr(string_handle, charset_name, buffer, \
                                         buffer_size, error)                  \
  mysql_string_service->mysql_string_convert_to_char_ptr_type(                \
      string_handle, charset_name, buffer, buffer_size, error)
#else
int mysql_string_convert_to_char_ptr(mysql_string_handle, const char *, char *,
                                     unsigned int, int *);   /* 静态链接走真实函数 */
#endif
```

> ★ 为什么成员要带 `_type` / `_func` 后缀：**不带后缀的名字要留给用户可见的宏用**，两者不能同名。
> ★ `service_thd_alloc.h` 的不透明句柄手法：`class THD;` 前置声明 + `#define MYSQL_THD THD *`——插件永远拿不到 THD 布局。
> ★ `service_security_context.h` 用 `typedef char my_svc_bool;` 而非 `bool`——跨编译器 ABI 稳定考虑（bool 大小由实现决定）。
> ⚠️ `service_mysql_string.h` 的宏带 `_type`、`service_thd_alloc.h` 带 `_func`、`service_my_plugin_log.h` 与 `service_security_context.h` **不带后缀**——约定并不统一。

插件侧通过静态库 `libservices/`（17 个 `service_xxx.c`，`SERVICE_VERSION *mysql_string_service = (void**)VERSION_mysql_string;`）导出这些符号；`MYSQL_DYNAMIC_PLUGIN` 下头文件把服务调用包装成宏（`mysql_string_convert_to_char_ptr(...)` → `mysql_string_service->...`），静态编译时则声明真实符号。

### 两套机制的桥：`plugin_registry_service`

`include/mysql/service_plugin_registry.h` 注释自我定位："A bridge service allowing plugins to work with the service registry"。实现只有四行语义（`sql/server_component/plugin_registry_service.cc`）：`mysql_plugin_registry_acquire()` = `srv_registry->acquire("registry", &h)`，`mysql_plugin_registry_release()` = 对应 release。**插件与组件共享同一个 registry 实例**（全局 `srv_registry`）。于是插件可以：

```cpp
SERVICE_TYPE(registry) *registry = mysql_plugin_registry_acquire();
my_service<SERVICE_TYPE(mysql_thd_attributes)> svc("mysql_thd_attributes", registry);
```

**★ 桥的完整源码**：

```c
/* include/mysql/service_plugin_registry.h */
extern "C" struct plugin_registry_service_st {
  SERVICE_TYPE(registry) * (*mysql_plugin_registry_acquire_func)();
  /*  ★★ 返回类型是组件侧的 SERVICE_TYPE(registry)* —— 老版服务结构体里装的是组件服务类型，
      这就是"桥"的技术本质 */
  int (*mysql_plugin_registry_release_func)(SERVICE_TYPE(registry) * registry_ptr);
} * plugin_registry_service;   /* ★ 导出变量名 = list_of_services 第 17 条的 name */

#ifdef MYSQL_DYNAMIC_PLUGIN
#define mysql_plugin_registry_acquire() \
  plugin_registry_service->mysql_plugin_registry_acquire_func()
#define mysql_plugin_registry_release(r) \
  plugin_registry_service->mysql_plugin_registry_release_func(r)
#else
SERVICE_TYPE(registry) * mysql_plugin_registry_acquire();   /* 内建插件直连 */
int mysql_plugin_registry_release(SERVICE_TYPE(registry) *);
#endif
```

```cpp
/* sql/server_component/plugin_registry_service.cc —— 实现只有四行语义 */
#include "sql/mysqld.h"  // srv_registry      ★ 全局组件 registry 实例

SERVICE_TYPE(registry) * mysql_plugin_registry_acquire() {
  my_h_service registry_handle;
  if (srv_registry->acquire("registry", &registry_handle)) {
    return nullptr;
  }
  return reinterpret_cast<SERVICE_TYPE(registry) *>(registry_handle);
}

int mysql_plugin_registry_release(SERVICE_TYPE(registry) * reg) {
  using service_type_t = SERVICE_TYPE_NO_CONST(registry);
  return srv_registry->release(
      reinterpret_cast<my_h_service>(const_cast<service_type_t *>(reg)));
  /*  ★ SERVICE_TYPE 是 const 限定的，release 需要非 const，故用 NO_CONST + const_cast */
}
```

> ★★ 头文件注释里还有两条重要的**运维提示**：acquire/release 是 *"expensive operation"*（要访问 registry 全局结构），建议**长期持有引用**；但持有的代价是——"**被引用服务的提供者组件无法卸载**"，且"**持有旧 default 引用者不会切换到后来安装的新 default**"。

**★ 真实插件用桥消费组件服务**（`plugin/semisync/semisync.cc:44`）：

```cpp
bool is_sysvar_defined(const char *name) {
  auto registry_handle = mysql_plugin_registry_acquire();       /* ① 桥：拿 registry 新引用 */
  assert(registry_handle != nullptr);
  my_service<SERVICE_TYPE(component_sys_variable_register)> svc(
      "component_sys_variable_register", registry_handle);      /* ② RAII acquire 组件服务 */
  bool get_var_error =
      svc->get_variable("mysql_server", name, &value, &value_length);  /* ③ 直接调用 */
  mysql_plugin_registry_release(registry_handle);               /* ④ 释放 registry */
  return !get_var_error;
  /*  ★ svc 是栈对象，析构（release 具体服务）发生在 ④ 之前 —— 顺序正确 */
}
```

```cpp
/* plugin/version_token/version_token.cc:205 —— 显式作用域确保顺序 */
  SERVICE_TYPE(registry) *r = mysql_plugin_registry_acquire();
  bool has_admin_privilege = false;
  {
    my_service<SERVICE_TYPE(global_grants_check)> service(
        "global_grants_check.mysql_server", r);    /* ★ 全限定名 */
    if (service.is_valid()) {
      has_admin_privilege = service->has_global_grant(...);
    }
  }  // end scope                                  /* ★ service 析构（release）先发生 */
  mysql_plugin_registry_release(r);
```

> ★ `mysql_plugin_registry_acquire()` 的调用点遍布树内：`plugin/semisync`、`version_token`、`keyring_udf`、X plugin、`connection_control`、`group_replication`（6 处）、`pfs_table_plugin`、`test_services`，以及**内建的 `storage/perfschema`**（`table_keyring_keys.cc:183`、`service_pfs_notification.cc:488/516` 等）。

**★ `validate_password` 插件：两套机制在同一函数内共存**（`plugin/password_validation/validate_password.cc`）：

```cpp
#include <mysql/components/my_service.h>                 /* 组件侧 */
#include <mysql/components/services/log_builtins.h>
#include <mysql/service_mysql_string.h>                  /* 老版服务侧 */
#include <mysql/service_security_context.h>

static SERVICE_TYPE(registry) *reg_srv = nullptr;
SERVICE_TYPE(log_builtins) *log_bi = nullptr;
SERVICE_TYPE(log_builtins_string) *log_bs = nullptr;
```

```cpp
  /* 老版服务：mysql_string_*（宏展开为 mysql_string_service->..._type） */
  mysql_string_handle lower_string_handle = mysql_string_to_lowercase(password);
  length = mysql_string_convert_to_char_ptr(
      lower_string_handle, "utf8mb3", buffer, MAX_PASSWORD_LENGTH, &error);
  mysql_string_free(lower_string_handle);   /* ★ 服务端分配的字符串必须用服务的 free（跨堆！） */

  /* 老版服务：security_context_* */
  if (thd_get_security_context(thd_get_current_thd(), &ctx) || !ctx) {
    LogPluginErr(ERROR_LEVEL, ER_VALIDATE_PWD_FAILED_TO_GET_SECURITY_CTX);
    /*                        ↑ 组件 log_builtins 侧的宏 */
    return false;
  }
```

```cpp
static int validate_password_init(MYSQL_PLUGIN plugin_info) {
  push_deprecated_warn(thd_get_current_thd(), "validate password plugin",
                       "validate_password component");   /* ★ 8.0.39 官方口径：插件形式已废弃 */
  if (init_logging_service_for_plugin(&reg_srv, &log_bi, &log_bs)) return (1);
  /*  ★ 桥接入口：内部就是 mysql_plugin_registry_acquire() + acquire("log_builtins.mysql_server") */
  ...
}
static int validate_password_deinit(void *) {
  deinit_logging_service_for_plugin(&reg_srv, &log_bi, &log_bs);
  /*  ★ 严格对称：先 release log_bi/log_bs，再 release registry */
}
```

> ⚠️ validate_password 插件路径是 **`plugin/password_validation/`**（不是 `plugin/validate_password/`）；8.0.39 里插件（`plugin/password_validation/`）与组件（`components/validate_password/`）**两套并存**。
> ★ `log_builtins.h` 里也有一个"桥的接缝"：`#if defined(MYSQL_DYNAMIC_PLUGIN) #include <mysql/service_plugin_registry.h>`——动态插件要吃组件侧的 `log_builtins`，必须间接包含老版服务头。两条机制在这里**物理交汇**。

树内实例：`plugin/semisync`、`plugin/version_token`、`plugin/keyring_udf`、`storage/innobase/handler/ha_innodb.cc`、`storage/perfschema`、X plugin 等全部如此取用组件服务。

---

### 实战：多实现共存、default 归属与 `acquire_related` 的真实动机

**（1）同名服务的多个实现是真存在的——以 `keyring_*` 为例。**

8.0.39 里这族服务至少有两个实现者（均已核实）：

| 实现名（`<服务>.<实现者>`） | 来源 | 角色 |
|---|---|---|
| `keyring_writer.mysql_server`、`keyring_reader_with_status.mysql_server`、`keyring_aes.mysql_server`、`keyring_generator.mysql_server` … | `sql/server_component/server_component.cc`（mysql_server 的 146 个 provides 之一，见 685-692 行） | **服务器代理实现**——keyring **插件**不能提供服务（插件只能被调），所以由服务器代它注册这一族服务并转发到实际插件 |
| `keyring_writer.component_keyring_file`、`keyring_reader_with_status.component_keyring_file` … | `components/keyrings/keyring_file/keyring_file.cc` | keyring **组件**自己提供 |

`log_builtins` 同理：`log_builtins.mysql_server` 与 `log_builtins.component_keyring_file` 并存。

**（2）default 归属由"注册顺序"决定，不由实现质量决定。**

`register_service` 用 `emplace`（不是覆盖），**先注册者自动成为 default**；而 mysql_server 的 146 个服务在 `minimal_chassis_init()` 里一次性注册，**早于任何外部组件加载**。所以只要 mysql_server 提供了某服务，default 就永远是 `.mysql_server`；`component_keyring_file` 的 provides 只是以全名并存，抢不走 default（这正是 [`component.md`](component.md) 里"服务器环境下 keyring 组件的 `log_builtins` 只是并存"的机制来源）。

要让组件实现成为 default，必须显式 `set_default`——启动时就干了四次（`mysqld.cc` 的 `component_infrastructure_init`）：

```c
my_service<SERVICE_TYPE(registry_registration)> registrator("registry_registration", srv_registry);
// Sets default file scheme loader for MySQL server.
registrator->set_default("dynamic_loader_scheme_file.mysql_server_path_filter");
// Sets default rw_lock for MySQL server.
registrator->set_default("mysql_rwlock_v1.mysql_server");
// Sets default psi_system event service for MySQL server.
registrator->set_default("mysql_psi_system_v1.mysql_server");
// Sets default mysql_runtime_error for MySQL server.
registrator->set_default("mysql_runtime_error.mysql_server");
```

四次是同一个动作：**用 mysql_server 的实现覆盖 minimal chassis 的裸实现**（前者带 PSI 插桩、带服务器错误码与日志）。这就是 override 能力的实战——不改代码，只在启动早期换掉 default 指针。

**（3）`acquire_related` 的真实动机：多实现带来的歧义。**

有了（1）之后，名字寻址就出现歧义：`acquire("keyring_writer")` 可能拿到 mysql_server 的代理，而另一个调用点拿到的 reader 可能指向别的实现者——**"读用 A、写用 B"在密钥管理上是灾难**。InnoDB 的解法（`storage/innobase/os/os0enc.cc`，原文）：

```cpp
if (reg_srv->acquire("keyring_reader_with_status", &h_keyring_reader_service) ||
    reg_srv->acquire_related("keyring_writer", h_keyring_reader_service,
                             &h_keyring_writer_service) ||
    reg_srv->acquire_related("keyring_generator", h_keyring_reader_service,
                             &h_keyring_generator_service)) { ... }
```

先用名字拿到 reader 的实现（拿 default），再**以这个句柄为锚**去要 writer/generator：`acquire_related` 把锚的全名截出 `.mysql_server` 这类后缀，拼成 `keyring_writer.<同一实现者>` 再 acquire。所以它的语义是**"同一实现者的另一个服务"，不是"另一个版本"**（后者靠改名）。这是多实现环境下保证"一组协同服务来自同一实现"的唯一手段——也是理解这个方法最容易搞错的地方。

**（4）句柄是快照：default 替换不影响已持有的引用。**

`service_plugin_registry.h` 的注释点明：已经 acquire 的引用**不会**因后来 `set_default` 换了默认实现而漂移——default 只在 acquire 那一刻生效。于是"动态替换"与"运行稳定"干净分离：新消费者拿新实现，老消费者继续用旧实现直到 release。代价是**替换不是全局瞬时的**，运维上要预期"新旧实现并存一段时间"。

**（5）RAII 与句柄失效。**

`my_service<T>` 把 acquire/release 包成作用域（`include/mysql/components/my_service.h`）：

```cpp
~my_service() {
  if (this->is_valid()) {
    m_registry->release(m_service);        /* 析构自动 release */
  }
}
operator bool() const { return !is_valid(); }   /* 注意取反，见下 */
```

`operator bool()` 返回 `!is_valid()`（无效时为 true），是为了让 `if (service) { ... }` 读作"拿不到服务"——与 MySQL "返回 true 表示失败"的惯例一致，但极易写反。`untie()` 用于把句柄交出去（不再自动 release），对应"获取后跨作用域长期持有"的场景。

句柄失效的唯一安全规则：release 之后句柄立刻不可用（`registry.h` 注释原文："After the call to this method the usage of the Service Implementation handle will lead to unpredicted results"）。由于组件卸载要求引用计数归零，**任何长期持有句柄的组件，都会阻止其服务提供者被卸载**——这正是 [`component.md`](component.md) 卸载链里"引用计数必须**恰好等于**批内依赖数"这条判据的另一面。

### 并发：多消费者、锁粒度与句柄共享

**（1）锁粒度：热路径共享，结构变更独占。**

```c
/* acquire/release：读锁（false = read），可并发 */
DEFINE_BOOL_METHOD(mysql_registry_imp::acquire,
                   (const char *service_name, my_h_service *out_service)) {
  minimal_chassis::rwlock_scoped_lock lock(&LOCK_registry, false, __FILE__, __LINE__);
  return mysql_registry_imp::acquire_nolock(service_name, out_service);
}

/* register/unregister/set_default：写锁（true = write），独占 */
minimal_chassis::rwlock_scoped_lock mysql_registry_imp::lock_registry_for_write() {
  return minimal_chassis::rwlock_scoped_lock(&LOCK_registry, true, __FILE__, __LINE__);
}
```

分工很清楚：**读锁只保护 map 结构不被并发修改**，而真正的计数是原子的（`my_ref_counted::m_count` 是 `std::atomic<uint64>`）。所以多个组件可以同时 acquire——服务消费这条路径在设计上就不互斥。

**（2）多消费者：引用计数就是"当前有几个消费者"。**

同一实现被 N 个消费者 acquire，计数就是 N。最直观的例子是 `log_builtins`：8.0.39 里共 **7 个组件**在 requires 里声明了它（`log_filter_dragnet`、`log_sink_json`、`log_sink_syseventlog`、`log_sink_test`、`mysqlbackup`、`validate_password`、`component_keyring_file`），合计 **14 条 requires 条目**（多数同时声明 `log_builtins_string`）。每个组件加载时 loader 替它 acquire 一次并累加；卸载时递减。于是"引用计数必须恰好等于批内依赖数"这条卸载判据在日常运行中的含义就是：**只要还有任何一个组件在用，`log_builtins` 的提供者就卸不掉**（提供者是 mysql_server，本来就卸不掉，但同样的规则对 keyring 组件是真实生效的）。

**（3）服务无状态 ⇒ 一个句柄可跨线程共享（与插件 THDVAR 的鲜明对照）。**

这是"服务必须无状态"这条 ABI 约束（见理论基础 1）在**运行期**的直接收益：方法无状态、函数指针表是 const 的，于是**同一个 `my_h_service` 可以被任意多线程同时使用，不需要 per-thread acquire**。InnoDB 就这么干——加密子系统 acquire 一次 `keyring_reader_with_status`，之后所有线程的密钥读写共用这个句柄（`storage/innobase/os/os0enc.cc`）。

对照插件侧的 THDVAR：插件变量**有状态**（per-session 值），所以每个会话都要一份 `dynamic_variables_ptr` 拷贝（见 [`plugin.md`](plugin.md)）。**"无状态"换来并发共享的自由度，"有状态"逼出 per-session 拷贝的成本**——这是两代扩展机制在运行期开销上的根本差异，也是"为什么服务要规定无状态"这个功能层面的答案（ABI 层面只解释了"为什么不能共享 C++ 对象"）。

**（4）竞争窗口：`set_default` 与并发 acquire。**

`set_default` 持写锁 → 期间所有 acquire 阻塞；持有读锁期间也不可能有人改 default。所以"default 只在 acquire 那一刻生效"（见实战 4）的并发含义是：**acquire 与 set_default 天然串行，不存在"读到半个新 default"的中间态**；但已持有的旧句柄不受后续 `set_default` 影响，新旧实现会并存一段时间。

### 三个真实竞争场景（运维视角）

**场景 A：运行中执行 `INSTALL COMPONENT`，其它线程正在 acquire**

加载器注册服务要拿 registry **写锁**，而其它组件的 `acquire` 持有读锁——结果是**阻塞等待**，不是失败。所以 `INSTALL COMPONENT` 在有活跃消费者时会短暂卡住（等所有读锁释放），但不会破坏一致性。这也解释了为什么组件加载是"整批事务"：写锁期间必须一口气注册完，避免中间态被别人读到。

**场景 B：多个消费者共享 `log_builtins`**

承接上一节：8.0.39 里共 **7 个组件**（14 条 requires 条目）声明了 `log_builtins`，每个组件加载时 acquire 一次 → 引用计数累加（上限 14，含 `log_builtins_string`）。卸载其中任一个组件 → 计数相应递减，**其余组件完全不受影响**（各自持有独立句柄）。提供者是 `mysql_server`（本来就卸不掉），但同样的规则对第三方组件提供者是真实生效的。

**场景 C（最有运维价值）：keyring 双实现下，`UNINSTALL COMPONENT` 能不能成功**

```
情形 1：InnoDB 按默认 acquire（acquire("keyring_reader_with_status")）
        → 拿到的是 keyring_reader_with_status.mysql_server（代理实现）
        → 组件的实现引用计数 = 0
        → UNINSTALL COMPONENT 'file://component_keyring_file' 成功

情形 2：某消费者用 acquire_related 锁定了组件实现
        → 持有 keyring_writer.component_keyring_file，计数 = 1
        → 该消费者不在卸载批次内
        → 引用计数 ≠ 批内依赖数 → 拒绝卸载
           ER_COMPONENTS_UNLOAD_CANT_UNREGISTER_SERVICE
```

即：**"能不能卸载组件"取决于别人 acquire 的是不是它这个实现，而不取决于它有没有被安装**。这正是卸载判据"引用计数必须**恰好等于**批内依赖数"（[`component.md`](component.md) 卸载链）在运维上的直接后果——排查"组件卸不掉"时，要查的是谁持有了它的实现句柄。

（keyring 的双实现与代理层详见 [`keyring.md`](keyring.md)。）

### 服务实现的错误处理约定

写完一个服务方法，"出错了该怎么做"有三层，各走各的通道：

| 层 | 手段 | 服务对象 |
|---|---|---|
| **返回值** | `mysql_service_status_t`（`typedef int`），0=成功、非 0=失败 | 直接调用方（组件/插件） |
| **报错** | `my_error()` / `mysql_error_service_printf()` | 当前会话（用户可见的错误） |
| **日志** | `log_builtins` / `log_builtins_string` 服务 | 错误日志（运维可见） |
| **异常** | `try/catch(...)` + `mysql_components_handle_std_exception` | 内部消化，绝不越过边界（见 [`abi.md`](abi.md)） |

**返回值**：`DECLARE_BOOL_METHOD` 生成的方法返回 `mysql_service_status_t`，语义是"0 成功、非 0 失败"——与 MySQL 主流 C 函数一致（`DEFINE_BOOL_METHOD` 则要求返回 `false`/`true`，因为 `DEFINE_METHOD(...bool...)` 的返回类型是 `bool`；两者都要保证 `noexcept`）。

**报错——这里有一个漂亮的宏魔法**（`include/mysql/components/services/mysql_runtime_error_service.h:60-74`）：

```c
extern REQUIRES_SERVICE_PLACEHOLDER(mysql_runtime_error);
/*  ★ 占位符声明：组件里必须另有 REQUIRES_SERVICE(mysql_runtime_error) 才会真正填上指针 */

/* 变体 1：显式传 handle（适合未使用 REQUIRES_SERVICE 占位符的场合） */
inline void mysql_error_service_emit_printf(SERVICE_TYPE(mysql_runtime_error) *
                                                error_svc_handle,
                                            int error_id, int flags, ...) {
  va_list args;
  va_start(args, flags);
  error_svc_handle->emit(error_id, flags, args);
  va_end(args);
}

/* 变体 2：用全局占位符 —— 被 #define my_error 指向的就是它 */
inline void mysql_error_service_printf(int error_id, int flags, ...) {
  va_list args;
  va_start(args, flags);
  mysql_service_mysql_runtime_error->emit(error_id, flags, args);   /* 走 mysql_runtime_error 服务 */
  va_end(args);
}

typedef int myf; /* Type of MyFlags in my_funcs */
#define MYF(v) (myf)(v)      /* ★ 组件侧重新定义了一份 MYF，因为不能依赖 my_sys.h */

#ifndef MYSQL_SERVER
#define my_error mysql_error_service_printf        /* ← 组件里写 my_error 会被替换成服务调用 */
#endif
```

> ⚠️ 真名核查：变体 1 是 **`mysql_error_service_emit_printf`**（不是 `emit_printf`）。
> ⚠️ **`mysql_runtime_error` 的实现文件不叫 `mysql_runtime_error_service.cc`**——真名是 **`sql/server_component/mysql_runtime_error.cc`**（带 `_service` 后缀的只有头文件）。实现类真名是 **`mysql_server_runtime_error_imp::emit`**：

```cpp
DEFINE_METHOD(void, mysql_server_runtime_error_imp::emit,
              (int id, int flags, va_list args)) {
  const char *format;
  if (!(format = my_get_err_msg(id))) {
    char ebuff[32];
    (void)snprintf(ebuff, sizeof(ebuff), "Unknown error %d", id);
    my_message((unsigned int)id, ebuff, flags);
    return;
  }
  my_printv_error(id, format, flags, args);   /* ★ 最终与 my_error() 同源 */
}
```

**★ 异常处理：`mysql_components_handle_std_exception()` 全文**（`components/libminchassis/component_common.cc`）：

```cpp
void mysql_components_handle_std_exception(const char *funcname) {
  try {
    throw;                                  /* ★★ 重抛（rethrow）—— 只能出现在 catch(...) 内 */
  } catch (const std::bad_alloc &e) {
    mysql_error_service_printf(ER_STD_BAD_ALLOC_ERROR, MYF(0), e.what(), funcname);
  } catch (const std::domain_error &e) { ... ER_STD_DOMAIN_ERROR ... }
  } catch (const std::length_error &e) { ... }
  } catch (const std::invalid_argument &e) { ... }
  } catch (const std::out_of_range &e) { ... }
  } catch (const std::overflow_error &e) { ... }
  } catch (const std::range_error &e) { ... }
  } catch (const std::underflow_error &e) { ... }
  } catch (const std::logic_error &e) { ... }
  } catch (const std::runtime_error &e) { ... }
  } catch (const std::exception &e) {
    mysql_error_service_printf(ER_STD_UNKNOWN_EXCEPTION, MYF(0), e.what(), funcname);
  } catch (...) {
    mysql_error_service_printf(ER_UNKNOWN_ERROR, MYF(0));
  }
}
```

> ★★ **catch 顺序不可颠倒**：派生类在前，`logic_error`/`runtime_error` 基类在后，`std::exception` 再后，`catch(...)` 最后。顺序颠倒会导致前面的分支永不命中（`-Wexceptions` 会告警）。
> ★ 每个具体异常类型都有专属 `ER_STD_*` 错误码（11 个），兜底是 `ER_UNKNOWN_ERROR`。
> ★ 调用约定固定：`catch (...) { mysql_components_handle_std_exception(__func__); } return true;` —— **异常绝不让其越过 ABI 边界**，统一降级为"返回失败 + 已上报错误"。`dynamic_loader.cc` 里就有 7 处这样的调用。

含义：组件代码里照常写 `my_error(ER_XXX, MYF(0), ...)`，但因为在**非服务器构建**（`MYSQL_SERVER` 未定义）中 `my_error` 被宏替换成 `mysql_error_service_printf`，实际执行的是 `mysql_runtime_error` 服务——**组件没有链接 mysqld，却能用"服务器风格"的报错 API**。这正是"服务化让插件/组件不必直呼服务器符号"的具体兑现（见 [`abi.md` 设计思想 ②](abi.md)）。

反之，服务器自身的代码里 `MYSQL_SERVER` 已定义，`my_error` 就是真正的服务器函数。

**日志**：`log_builtins` 用于"记录"而非"报错"——组件用它写错误日志（`log_warning` / `log_error` 系列），而 `my_error` 是给当前语句/会话产生错误条件。两者不要混用：`log_builtins` 不会产生错误码，也无法被调用方的返回值检查捕获。

**异常**：任何 C++ 异常都必须在本 DSO 内消化（异常跨 .so = `std::terminate`，见 [`abi.md` 实验 3](abi.md)）。标准写法就是：整个方法体包 `try`，`catch (...) { mysql_components_handle_std_exception(__func__); }`，最后返回失败值。注意**不要**在 catch 之后继续用可能处于中间态的对象——先返回失败，由调用方决定是否重试。

---

## Misc

### 命名辨析（实测踩坑清单）

| 名字 | 状态 | 说明 |
|---|---|---|
| `mysql_service_<x>` | 组件服务占位符前缀 | `REQUIRES_SERVICE_PLACEHOLDER(x)` 生成 `mysql_service_##x` |
| `mysql_string_service` / `thd_alloc_service` / `plugin_registry_service` | 老版插件服务指针 | **没有** `mysql_service_` 前缀 |
| `SERVICE_NAME` / `SERVICE_METHOD` / `BEGIN_SERVICE_METHOD_DEFINITION` | **不存在** | 宏名实际是 `DECLARE_METHOD` / `DECLARE_BOOL_METHOD`；服务名由 `PROVIDES_SERVICE` 的 `#service "." #component` 字符串化生成 |
| `mysql_service_my_snprintf` / `mysql_service_thd_alloc` | **不存在** | 老版对应物是 `my_plugin_log_service` / `thd_alloc_service`；也没有 `service_my_snprintf.h` |
| `include/mysql/components/README` | **不存在** | 设计文档以 `@page` 形式嵌在 `service.h` / `dynamic_loader.cc` / `registry.cc` 源码里 |
| `components/mysql_server/` 目录 | **不存在** | mysql_server 组件声明在 `sql/server_component/server_component.cc` |

### 老版插件服务现状：架空但未死

8.0.39 中老版服务（a）无 `@deprecated` 标注，（b）仍被内置插件实际使用（`validate_password` 用 5 个服务头、`rewriter` 用 parser/rules_table、X plugin 用 6 个），（c）甚至有 4 个组件反向引用老版服务头。但架构上已降级：唯一入口服务 `plugin_registry_service` 自我定义为"通往 registry 的桥"；`mysql_keyring_service` 的实现在全树已无活跃调用（只剩注释掉的代码），对应能力由 10 个 `keyring_*` 组件服务接管；`dynamic_loader.cc` 把插件定义为 "dynamically loadable bits of the server component"。

### PFS 为什么服务化

插桩（`psi_*`）是**唯一必须被"非服务器二进制"调用的能力**——组件/插件不能链接 mysqld，只能经服务消费。`psi_bits.h` 把插桩接口显式分为 `psi_api`（编程接口）与 `psi_abi`（二进制接口）两组，服务属于 ABI 侧。而服务器内部代码为了性能仍走**全局符号直连**：`PSI_THREAD_CALL` 在服务器（`mysql/psi/mysql_thread.h`，指向 mysqld 导出的 `psi_thread_service` 全局符号）与组件（`components/services/psi_thread.h`，指向占位符）各有一套定义。

---

## 参考

**官方文档**
- WorkLog: [WL#4102 Service registry and component infrastructure](https://dev.mysql.com/worklog/task/?id=4102)
- *MySQL 8.0 Reference Manual → Extending MySQL → Components and Services / MySQL Services for Plugins*
- MySQL Server Doxygen: [Component Infrastructure Layers](https://dev.mysql.com/doc/dev/mysql-server/latest/page_components_layering.html)

**源码文档页**
- `include/mysql/components/service.h`：`@page PAGE_COMPONENTS_SERVICE`（为什么是 C 函数指针表、无状态、不版本化）
- `include/mysql/components/component_implementation.h`：`PAGE_COMPONENTS_COMPONENT` / `PAGE_COMPONENTS_IMPLEMENTATION`
- `sql/sql_plugin.cc`：`@page page_ext_plugin_svc_anathomy`（老版插件服务解剖）

**相关文档**
- 架构总览（双轨制与迁移进度）见 [`architecture.md`](architecture.md)
- ABI 契约专题（服务 const 表/noexcept/不透明句柄/"版本=改名"）见 [`abi.md`](abi.md)
- 组件加载/卸载/持久化见 [`component.md`](component.md)
- 插件体系见 [`plugin.md`](plugin.md)

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `components/libminchassis/registry.h` | **不存在**（目录下只有 `registry.cc` / `registry_imp.h` / `registry_metadata.h.inc` / `registry_metadata.cc.inc`） |
| 类叫 `service_registry` | **真名 `mysql_registry_imp`**，在 `components/libminchassis/registry_imp.h:40` |
| 成员 `m_registry`（std::map） | **真名是静态成员 `mysql_registry_imp::service_registry`**（`registry_imp.h:45`），typedef 为 `my_service_registry` |
| `components/libminchassis/service_implementation.h` | **不存在** → `mysql_service_implementation.h` |
| `mysql_service_implementation_ref` | **不存在**（全仓 0 匹配），引用计数靠继承的 `my_ref_counted` |
| `my_metadata::get_name()` / `get_urn()` | **都不在 `my_metadata`**（它只有 `set_value`/`get_value`/`create_iterator`/`release_iterator`）。`get_urn()` 在 `mysql_component`，`name_c_str()` 在 `mysql_service_implementation` |
| `my_ref_counted` 的计数成员叫 `m_ref_count` | 真名 **`m_count`**（`std::atomic<uint64>`） |
| `PROVIDES_SERVICE_METHOD` | **不存在**（0 匹配）。提供侧只有 `PROVIDES_SERVICE` |
| `PROVIDES_CUSTOM_SERVICE` | **不存在**（仅 `component_implementation.h:177` 注释提及一次，无 `#define`） |
| `BEGIN_COMPONENT_REQUIRES` 只是开个结构体 | **不止**：它先插入 `REQUIRES_SERVICE_PLACEHOLDER(registry)`，再把 `REQUIRES_SERVICE(registry)` 作为数组**第一项**。另有一个 `BEGIN_COMPONENT_REQUIRES_WITHOUT_REGISTRY` |
| `service.h` 的 `@page PAGE_COMPONENTS_SERVICE` 有"四条理由" | **没有编号列表**——是四段连续散文（`:36-65`） |
| `DEFINE_SERVICE_HANDLE` 注释写 "never use virtual functions" | 原话是 *"does not use any **virtual methods**"* + *"try use only one member, the **d-pointer**"* |
| `my_service<T>::operator bool()` 表示"句柄有效" | **反的**！`return !is_valid()` —— **true 表示无效/失败**（模仿错误码风格）。判断有效请用 `is_valid()` |
| `c_string_less` 在 libminchassis 里 | **在 `include/c_string_less.h:39`** |
| registry 服务实现表叫 `imp_mysql_registry` | **全仓 0 匹配**。真名 **`imp_mysql_minimal_chassis_registry`**，由 `BEGIN_SERVICE_IMPLEMENTATION(mysql_minimal_chassis, registry)` 宏生成 |
| `register_service()` 里直接写业务逻辑 | **8.0.39 拆两层**：外壳只加写锁，逻辑在 `register_service_nolock()`。`acquire`/`release`/`unregister` 各有 `_nolock` 版 |
| `register_service` 用 `lock_registry_for_write()` | **不是**。它直接构造 `rwlock_scoped_lock`。`lock_registry_for_write()` 全仓**只有一个调用者**：`unload_do_lock_provided_services()` |
| minchassis 下有 `rwlock.h` / `rw_lock.h` | **不存在** → `rwlock_scoped_lock.h`，类 `minimal_chassis::rwlock_scoped_lock`（禁止拷贝、只能 move） |
| `LOCK_registry` 是全局 C 变量 | 是 `mysql_registry_imp` 的 **private static 成员**（定义 `registry.cc:677`）；`LOCK_dynamic_loader` 是 `mysql_dynamic_loader_imp` 的（定义 `dynamic_loader.cc:1437`） |
| "先注册者成为 default" 由 `set_default` 实现 | **不是**。靠 `register_service_nolock` 里 `emplace_hint` 在 key 已存在时**静默失败不覆盖**（`registry.cc:261-262`）。文件头注释亦写 "the first one to be added" |
| `set_default` 只是换个指针 | **不是**：先 `erase` 旧条目再 `emplace_hint`。原因是 map 的 key 是 `const char*`，指向**旧 default 实现的 `m_service.c_str()`**，旧实现被 unregister 后该指针悬空 |
| `unregister` 失败只因为引用计数 > 0 | 不全面。还有：名字找不到、"有实现名却没有 default 条目"的状态不一致 |
| `acquire_related` 用 `strchr` 截"实现名" | 精确说：`strchr(name_c_str(), '.')` 找**第一个**点，`component_part` 是**含点的** `".component"` 子串；且要求入参 `service_name` **本身不含点**（含点直接拒绝） |
| `get_service_implementation_reference_count` 找不到返回 0 | **注释写 0，实际 `return -1`**（`registry.cc:155`）→ `(uint64_t)0xFFFF...`。这是"引用了未注册接口必失败"的兜底 |
| `interface_mapping` 是 map\<名字, 实现\> | 是 `std::map<my_h_service, mysql_service_implementation*>`（**接口指针 → 实现对象**），按指针地址比大小 |
| mysql_server 提供 146 个服务（都是 `.mysql_server`） | 146 条 `PROVIDES_SERVICE` 是对的，但其中 **39 条实现名是 `performance_schema`**、**1 条是 `mysql_server_path_filter`**，真正的 `.mysql_server` 是 **106 条** |
| mysql_server 的 146 个服务"最早注册" | 顺序是：① minimal_chassis 自己的 13 个 → ② `dynamic_loader_imp::init()` → ③ **才**注册 mysql_server 的 146 个。即"早于任何**动态加载**的组件，但晚于 minimal_chassis 自身" |
| `srv_registry` 在 `mysqld.cc` 里创建 | **不是**。定义在 mysql_server 组件里（`sql/mysqld.h:838` 仅 extern），由 `minimal_chassis_init()` 内部 `acquire("registry")` 后**赋值** |
| `component_infrastructure_init` 的四次 set_default 都是 mysql_server | 第 1 次的实现名是 **`mysql_server_path_filter`**（不是 mysql_server） |
| `sql/server_component/mysql_runtime_error_service.cc` | **不存在** → `sql/server_component/mysql_runtime_error.cc`；实现类是 `mysql_server_runtime_error_imp::emit` |
| `mysql_plugin_registry_acquire()` 是组件基础设施 API | 它是**老版插件服务** `plugin_registry_service` 的成员（`list_of_services[]` 第 17 条），由 `libservices/plugin_registry_service.c` 导出桩符号 |
| 插件卸载时服务指针会被复位回版本号 | **永不复位**。`free_plugin_mem()` 只 `dlclose`+`my_free`；`plugin_dl_del()` 只减 ref_count；`st_plugin_dl` 里根本没有服务字段——**想复位也没地方记** |
| `SERVICE_VERSION` 是 typedef / 结构体 | 是**宏**：Linux 下就是 `void *`，Windows 下是 `__declspec(dllexport) void *` |
| `list_of_services[]` 有 18 条 | **17 条**。`VERSION_clone_protocol_service` 没有对应条目也没有 `libservices/*.c` |
| 插件里 `mysql_string_convert_to_char_ptr(...)` 是真实函数调用 | 动态插件下它是**宏**，展开为 `mysql_string_service->mysql_string_convert_to_char_ptr_type(...)` |
| `#define my_error mysql_error_service_printf` 全局生效 | 只在 `#ifndef MYSQL_SERVER` 时替换（即组件代码内）。服务端内部 `my_error` 不受影响 |
| `plugin/validate_password` 是唯一形态 | 8.0.39 **两套并存**：插件在 `plugin/password_validation/`（已 `push_deprecated_warn`），组件在 `components/validate_password/` |
| `mysql_string_service` 桩在服务端定义 | 桩在 `libservices/*.c`（打进 `mysqlservices` 静态库供插件链接）；服务端真实实现是 `sql/sql_plugin_services.h` 里的 `mysql_string_handler` |
| 老版服务宏的成员后缀统一 | **不统一**：`service_mysql_string.h` 用 `_type`、`service_thd_alloc.h` 用 `_func`、`service_my_plugin_log.h`/`service_security_context.h` **不带后缀** |

**★ 三个"结构性事实"**

1. **"先注册者成为 default"的全部实现就是 `emplace_hint` 的静默失败**——`register_service_nolock` 用 `emplace("X.Y", imp)` 插全限定名，再用 `emplace_hint(..., "X", imp)` 插裸服务名。`std::map::emplace_hint` 在 key 已存在时**不插入不覆盖**，一行代码、无任何 if/比较。这解释了为什么 `mysql_minimal_chassis` 的 `registry` 天然是 default（它第一个注册），也解释了为什么 mysql_server 必须靠 4 次 `set_default` 才能抢回 `mysql_rwlock_v1` / `mysql_psi_system_v1` / `mysql_runtime_error` / `dynamic_loader_scheme_file`。

2. **`set_default` 必须先 erase 再 emplace_hint，根源在 `mysql_service_implementation` 的构造函数**——它把 `m_service`（点之前的裸服务名）切分出来，而 `m_service.c_str()` 正是 map 里 default 条目的 **key 指针**。旧 default 实现一旦被 `unregister` 就析构，这个 key 就悬空。同一个原因也决定了 `unregister` 里"顺位接任"要用新实现的 `service_name_c_str()` 重新 `emplace_hint`。

3. **装载与卸载的加锁策略完全不对称**——加载时 `load()` 只加 `LOCK_dynamic_loader`，registry 锁由 `register_service()`（带锁版）自己加；卸载时 `unload_do_lock_provided_services()` 拿 registry **写锁**后一路持有到链尾，后续各步都调 `*_nolock` 版本。这就是"组件加载/卸载是整批事务"的实现方式：写锁期间必须一口气做完，避免中间态被别人读到。
