# Registry（服务注册表）深度解析

> 基于 MySQL 8.0.39 源码（`components/libminchassis/registry.cc`、`minimal_chassis.cc`）。Registry 是组件体系的**中枢**：唯一一个"必须先存在、又必须注册到自己里面"的元服务。本篇把它的数据组织、自举、锁与引用计数语义讲透。
>
> **边界**：本篇讲 registry **自身的实现**；服务的定义/消费契约见 [`service.md`](service.md)，组件的加载/卸载（dynamic loader）见 [`component.md`](component.md)，插件体系见 [`plugin.md`](plugin.md)，ABI 见 [`abi.md`](abi.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

Registry 是一张**运行期的"名字 → 服务实现"映射表**，对外表现为 5 个服务（`registry`、`registry_registration`、`registry_query`、`registry_metadata_enumerate`、`registry_metadata_query`），对内是 `mysql_registry_imp` 这个静态类持有的两张 map 和一把读写锁。

它是整个组件体系里**唯一不可替代的部件**：没有 registry，组件无法声明自己提供什么、也无法找到别人提供什么——dynamic loader 自己都要靠 registry 才能注册/注销服务（见 [`component.md`](component.md) 的加载链）。

### 用途

- 服务解析：`acquire("X")` 取默认实现、`acquire("X.Y")` 取指定实现；
- 服务注册/注销/切换默认（`registry_registration`）；
- 服务发现：按名迭代、查询元数据（`registry_query` / `registry_metadata_*`）；
- 支撑生命周期：引用计数决定"能否注销/卸载"（见 [`component.md` 卸载链](component.md)）。

### 版本演进

Registry 本身自 8.0 引入组件体系后**接口未变**（5 个服务的方法集在 8.0.39 与初版一致），演进主要发生在**服务数量**上（174 个服务定义）。这与插件侧"版本数字 + 结构迁移"的路线形成对比（见 [`abi.md`](abi.md) 设计思想 ⑤）。

---

## 理论基础

### 第零条：Registry 解决的是"自举悖论"

Registry 是服务，但它自己必须被注册进 registry 才能被人 acquire——**先有鸡还是先有蛋**。解法在 `minimal_chassis_init()` 里，思路是**绕过服务层、直接调实现类**完成种子注册（详见核心实现 ①）。

这个悖论及其解法揭示了一个架构事实：**registry 是唯一"不通过服务就能拿到"的服务**。所有其它能力的获取路径最终都收敛到这里，所以它必须是静态类 + 静态成员（`mysql_registry_imp::service_registry`），而不是某个组件实例的一部分。

### 名字空间设计：为什么一个 map 里放两种 key？

`service_registry` 同时存放：

| key 形式 | 含义 | 何时插入 |
|---|---|---|
| `"X.Y"` | **实现名**（服务名.实现者） | 每次 `register_service` |
| `"X"` | **服务名** → 当前默认实现 | 仅当该服务**还没有**默认实现时 |

好处：**一次 `find()` 同时支持"按实现名精确取"与"按服务名取默认"**，不需要两张表，也不需要在 acquire 时判断"传进来的是哪种名字"。这是用"空间换判断"的简化——代价是 map 条目数 ≈ 服务数 + 实现数，且注销时要额外维护 "X" 这一条（顺位接任或删除）。

### 锁粒度：读多写少，但迭代器是例外

- `acquire` / `release` / `get_reference_count`：读锁（共享）；
- `register` / `unregister` / `set_default`：写锁（独占，通过 `lock_registry_for_write()`）；
- **`registry_query::iterator_create`：读锁一直持有到 `iterator_release`**——源码注释原文："This read lock on whole registry will be held, until the iterator is released."

最后一条是设计上的**代价点**：迭代期间所有写操作（注册/注销/换默认，也就包括**组件加载与卸载**）都会被阻塞。所以遍历服务列表的代码必须尽快 `iterator_release`（`unload_do_check_dependencies` 就是 create 完立刻 release 的范例）。

### 引用计数：acquire 是"取快照"，不是"占坑"

三条精确语义（常被混淆）：

1. **每次 `acquire` 计数 +1，每次 `release` -1**——计数等于"当前有多少个未释放的句柄"；
2. **跨线程共享同一个句柄不再计数**：因为服务是无状态函数表（见 [`service.md` 并发小节](service.md)），多线程共用一个句柄是安全的，不需要 per-thread acquire；
3. **句柄是快照**：`set_default` 之后 acquire 的人拿新实现，已持有的人仍指向旧实现（不漂移）。

---

## 核心实现

### ① 自举：先有鸡还是先有蛋

```cpp
bool minimal_chassis_init(SERVICE_TYPE_NO_CONST(registry) **registry,
                          mysql_component_t *comp_ref) {
  /* 1. 先把最小底盘自己的基础服务占位符指向自己的实现 */
  mysql_service_mysql_rwlock_v1     = &imp_mysql_minimal_chassis_mysql_rwlock_v1;
  mysql_service_mysql_psi_system_v1 = &imp_mysql_minimal_chassis_mysql_psi_system_v1;

  /* 2. 创建 registry 的内部结构（空表 + 读写锁） */
  mysql_registry_imp::init();

  /* 3. 【种子注册】直接调实现类，把最小底盘提供的 11 个服务塞进 registry */
  for (int inx = 0;
       mysql_component_mysql_minimal_chassis.provides[inx].implementation != nullptr;
       ++inx) {
    if (imp_mysql_minimal_chassis_registry_registration.register_service(
            mysql_component_mysql_minimal_chassis.provides[inx].name,
            reinterpret_cast<my_h_service>(
                mysql_component_mysql_minimal_chassis.provides[inx].implementation)))
      return true;
  }

  /* 4. 之后就可以走正常服务路径了：acquire("registry") 把句柄交给调用方 */
  if (registry != nullptr) {
    my_h_service registry_handle;
    if (imp_mysql_minimal_chassis_registry.acquire("registry", &registry_handle))
      return true;
    *registry = reinterpret_cast<SERVICE_TYPE_NO_CONST(registry) *>(registry_handle);
  }
  ...
}
```

第 3 步是关键：**绕开 acquire 路径，直接调用 `imp_mysql_minimal_chassis_registry_registration`（实现类对象）**。这 11 个种子服务包括 registry 五件套、dynamic loader 四件套、`mysql_runtime_error`、以及 rwlock/psi_system。

第 4 步之后，所有组件（包括服务器本体 `mysql_server`）都通过 `acquire` 正常取用。注意 `mysql_server` 的 146 个服务也是在同一流程里注册的——**因此它们的 default 地位来自"注册得早"，不是"更重要"**（见 [`service.md` 实战章节](service.md)）。

### ② 数据结构全图

```cpp
/* registry_imp.h */
typedef std::map<const char *, mysql_service_implementation *, c_string_less>
    my_service_registry;

class mysql_registry_imp {
  typedef std::map<my_h_service, mysql_service_implementation *> my_interface_mapping;

  static my_service_registry  service_registry;    /* 名字 → 实现对象 */
  static my_interface_mapping interface_mapping;   /* 句柄 → 实现对象（反查） */
  static mysql_rwlock_t       LOCK_registry;
  ...
};

/* mysql_service_implementation.h */
class mysql_service_implementation : public my_ref_counted, public my_metadata {
  my_h_service m_interface;    /* 对外句柄，即函数指针表地址 */
  my_string    m_service;      /* 服务名 "X" */
  my_string    m_full_name;    /* 实现名 "X.Y" */
};

/* my_ref_counted.h */
class my_ref_counted {
  std::atomic<uint64> m_count;                     /* 原子计数 + 虚析构 */
  uint64 add_reference();
  bool   release_reference(uint64 *new_count = nullptr);
  uint64 get_reference_count() const;
};

/* my_metadata.h */
class my_metadata {
  std::unordered_map<std::string, std::string> data;   /* key-value 元数据 + 迭代器 */
  bool set_value(const char *name, const char *value);
  bool get_value(const char *name, const char **value);
  const_iterator create_iterator();
};
```

两个容易被忽略的实现细节：

- **key 是 `const char *` 配自定义比较器 `c_string_less`**（不是 `std::string`）：省掉每次查找的字符串构造，代价是 **key 指针的寿命必须由被指向者保证**——它指向 `mysql_service_implementation::m_full_name` 内部的字符串。这正是 `set_default` 必须先 `erase` 旧 key 再 `emplace_hint` 新 key 的原因（旧对象的字符串可能随对象析构失效）。
- **两个 map 互为反查**：`service_registry` 支持按名字取，`interface_mapping` 支持按句柄取（release、引用计数查询、`acquire_related` 都靠它反查实现对象）。

### ③ 五个服务的方法集

| 服务 | 方法 | 语义 |
|---|---|---|
| `registry` | `acquire(name, &h)` | 按服务名取默认 / 按实现名精确取，计数 +1 |
| | `acquire_related(name, h, &out)` | 以 `h` 的实现者为锚，取它的另一个服务 |
| | `release(h)` | 计数 -1；之后句柄不可再用 |
| `registry_registration` | `register_service("X.Y", impl)` | 插入实现条目；若 "X" 尚无默认则顺带设为默认 |
| | `unregister("X.Y")` | 计数非 0 → 失败；是默认 → 顺位接任 |
| | `set_default("X.Y")` | 显式切换默认实现 |
| `registry_query` | `iterator_create(pattern, &it)` | **持读锁直到 release**；pattern 为空则从头遍历 |
| | `iterator_get / next / is_valid / release` | 遍历与释放（释放时才放锁） |
| `registry_metadata_enumerate` | — | 枚举某服务的元数据 |
| `registry_metadata_query` | — | 查询元数据键值 |

### ④ acquire / release 的精确语义

```cpp
DEFINE_BOOL_METHOD(mysql_registry_imp::acquire,
                   (const char *service_name, my_h_service *out_service)) {
  minimal_chassis::rwlock_scoped_lock lock(&LOCK_registry, false, __FILE__, __LINE__);
  return mysql_registry_imp::acquire_nolock(service_name, out_service);
}
/* acquire_nolock:
   iter = service_registry.find(service_name);
   if (iter == cend()) return true;          // 名字不存在（无论是 "X" 还是 "X.Y"）
   iter->second->add_reference();            // 原子 +1
   *out_service = iter->second->interface(); // 返回函数指针表地址
*/
```

`release` 走 `interface_mapping` 反查对象后 `release_reference()`。**注意 release 之后句柄立刻失效**（`registry.h` 注释："the usage of the Service Implementation handle will lead to unpredicted results"）——因为对象可能已被注销并析构。

### ⑤ 注册、注销与默认实现（四条规则）

```cpp
/* register_service 的要点 */
service_registry.emplace(full_name, imp.get());                 /* "X.Y" */
if (尚无 "X" 条目)
  service_registry.emplace_hint(addition_result.first,
                                imp->service_name_c_str(), imp.get());   /* "X" → 首个实现 */
```

| 规则 | 行为 | 源码 |
|---|---|---|
| **首注册者为默认** | 用 `emplace`（非覆盖），后来的实现抢不走 | `register_service` |
| **注销时顺位接任** | 若注销的是默认，从 map 中找同服务名的下一个实现接任 | `unregister` |
| **可显式切换** | `set_default("X.Y")` erase + emplace_hint | `set_default` |
| **已持有句柄不漂移** | default 只在 acquire 那一刻生效 | `service_plugin_registry.h` 注释 |

### ⑥ 迭代器与锁（难点）

```cpp
DEFINE_BOOL_METHOD(mysql_registry_imp::iterator_create,
                   (const char *service_name_pattern, my_h_service_iterator *out_iterator)) {
  /* This read lock on whole registry will be held, until the iterator is released. */
  minimal_chassis::rwlock_scoped_lock lock(&LOCK_registry, false, __FILE__, __LINE__);

  my_service_registry::const_iterator r =
      (!service_name_pattern || !*service_name_pattern)
          ? service_registry.cbegin()
          : service_registry.find(service_name_pattern);
  if (r == service_registry.cend()) return true;

  *out_iterator = new my_h_service_iterator_imp{r, std::move(lock)};   /* 锁移交迭代器对象 */
  return false;
}
```

设计要点与代价：

- **锁的所有权被 `std::move` 进迭代器对象**——RAII 保证迭代结束（release）时才解锁，异常路径也安全。
- **代价**：迭代期间 registry 的**写操作全部阻塞**，也就是**组件加载/卸载被阻塞**。所以：
  - 遍历代码必须尽快 release（`unload_do_check_dependencies` 是 create→get/release 立即完成的范式）；
  - 长时间持有迭代器（例如把遍历嵌进耗时循环）会造成整机的组件装卸卡住。
- `pattern` 为空串则从头遍历整个 registry；给出服务名则从该条目开始（用于枚举某服务的多个实现）。

### ⑦ registry 与 dynamic loader 的分工

| | registry | dynamic loader |
|---|---|---|
| 管什么 | **服务**的名字、实现、引用计数、元数据 | **组件**的生命周期（加载/卸载/依赖） |
| 谁调谁 | loader 调 registry 注册/注销服务 | registry 不认识组件 |
| 卸载耦合 | loader 在写锁下检查引用计数，再注销 | 见 [`component.md` 卸载链](component.md) |

一句话：**registry 管"能力市场"的账本，loader 管"供货商"的进出**。

---

### `*_nolock` 方法与线程安全边界

registry 的实现分成两层：**对外服务方法自己加锁**，**内部 `*_nolock` 方法要求调用方已持锁**。这个分层不是风格问题，而是为了让调用方能做**跨多步的原子操作**。

| 方法 | 要求的锁 | 说明 |
|---|---|---|
| `acquire_nolock(name, &h)` | **至少读锁** | 注释原文："Assumes caller has at least a read lock" |
| `release_nolock(h)` | **至少读锁** | 同上 |
| `get_service_implementation_reference_count(h)` | **至少读锁** | 卸载判据用 |
| `get_service_implementation_by_interface(h)` | **至少读锁** | 句柄 → 对象反查 |
| `register_service_nolock(name, impl)` | 读锁（仅查表） | 实际插入由调用方在写锁下完成 |
| `unregister_nolock(name)` | **写锁** | 注释原文："Assumes caller has a write lock on the Registry" |
| `lock_registry_for_write()` | — | 供**外部**（dynamic loader）取写锁的 RAII 接口 |

**为什么要把"裸方法"暴露出来？** 看 dynamic loader 卸载时的需求（`component.md` 卸载链第 4-5 步）：它需要"**在持锁状态下**：逐个检查引用计数 → 决定是否继续 → 注销服务 → 卸载组件"——这是四步操作，若每步各自加解锁，中间就可能被别人插入一次 `acquire`，让"检查"的结果失效。

源码正是这么写的（`dynamic_loader.cc`）：

```cpp
bool mysql_dynamic_loader_imp::unload_do_lock_provided_services(...) {
  /* We do lock the whole registry, as we don't have yet any better granulation. */
  minimal_chassis::rwlock_scoped_lock lock =
      mysql_registry_imp::lock_registry_for_write();
  return mysql_dynamic_loader_imp::unload_do_check_provided_services_reference_count(...);
}
```

一把写锁贯穿"检查 + 注销"全过程 → 期间任何 `acquire` 都被阻塞 → 判据不会失效。代价是卸载期间 registry 完全不可读（这就是为什么卸载被设计成"整批、尽快"）。

**使用准则**：

1. 普通消费者只用服务层（`acquire/release`），绝不要碰 `*_nolock`；
2. 只有需要"多步原子"的框架代码（dynamic loader）才用 `lock_registry_for_write()` + `*_nolock`；
3. 持写锁期间**不要**调用任何会反过来 acquire 服务的东西——读写锁不可重入，会自死锁。

## Misc

### 常见坑

1. **迭代器长期持锁**：遍历服务期间会阻塞组件装卸，务必尽快 `iterator_release`。
2. **"装了组件就自动生效"是误解**：默认实现归**首注册者**（通常是 mysql_server 的代理实现），装了组件只是多了一个并存实现；要生效得 `set_default` 或用 `acquire_related` 锁定实现者（见 [`service.md`](service.md)）。
3. **release 后句柄即失效**：不可缓存已 release 的句柄；长期持有句柄会**阻止提供者被卸载**。
4. **key 是 `const char *`**：自定义组件名/服务名时，字符串寿命必须与实现对象同长（实现侧由 `m_full_name` 持有副本，这点 registry 已处理，但手写注册代码时要注意）。

### 一句话总结

Registry 是"**先把 11 个种子服务硬塞进自己、再让全世界按名字来找**"的自举中枢：两张 map（名字↔句柄互查）、一把读写锁、一套原子引用计数，撑起了 174 个服务的发现、替换与生命周期管理。

---

## 参考

**源码**
- `components/libminchassis/registry.cc`、`registry_imp.h`、`minimal_chassis.cc`、`mysql_service_implementation.h`、`my_ref_counted.h`、`my_metadata.h`
- `include/mysql/components/services/registry.h`（5 个服务的定义与注释）

**相关文档**
- 服务契约与消费：[`service.md`](service.md)
- 组件加载/卸载（dynamic loader 如何使用 registry）：[`component.md`](component.md)
- 插件体系与代理：[`plugin.md`](plugin.md)、[`keyring.md`](keyring.md)
- 架构总览：[`architecture.md`](architecture.md)
