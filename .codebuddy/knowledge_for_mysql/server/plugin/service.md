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

插件侧通过静态库 `libservices/`（17 个 `service_xxx.c`，`SERVICE_VERSION *mysql_string_service = (void**)VERSION_mysql_string;`）导出这些符号；`MYSQL_DYNAMIC_PLUGIN` 下头文件把服务调用包装成宏（`mysql_string_convert_to_char_ptr(...)` → `mysql_string_service->...`），静态编译时则声明真实符号。

### 两套机制的桥：`plugin_registry_service`

`include/mysql/service_plugin_registry.h` 注释自我定位："A bridge service allowing plugins to work with the service registry"。实现只有四行语义（`sql/server_component/plugin_registry_service.cc`）：`mysql_plugin_registry_acquire()` = `srv_registry->acquire("registry", &h)`，`mysql_plugin_registry_release()` = 对应 release。**插件与组件共享同一个 registry 实例**（全局 `srv_registry`）。于是插件可以：

```cpp
SERVICE_TYPE(registry) *registry = mysql_plugin_registry_acquire();
my_service<SERVICE_TYPE(mysql_thd_attributes)> svc("mysql_thd_attributes", registry);
```

树内实例：`plugin/semisync`、`plugin/version_token`、`plugin/keyring_udf`、`storage/innobase/handler/ha_innodb.cc`、`storage/perfschema`、X plugin 等全部如此取用组件服务。

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
- ABI 契约专题（服务 const 表/noexcept/不透明句柄/"版本=改名"）见 [`abi.md`](abi.md)
- 组件加载/卸载/持久化见 [`component.md`](component.md)
- 插件体系见 [`plugin.md`](plugin.md)
