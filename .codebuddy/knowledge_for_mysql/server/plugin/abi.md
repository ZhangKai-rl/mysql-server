# ABI（二进制接口）契约深度剖析

> 基于 MySQL 8.0.39 源码。本篇从 C++ 语言/动态链接两个层面，把插件、组件、服务三套机制共用的 ABI 工程手法逐一拆开：为什么契约面必须是 C 的、C++ 的五个 ABI 不稳定点各自怎么被挡在边界外、符号可见性与对象生命周期如何管理。
>
> **边界**：本篇是 ABI 专题的**唯一详写处**；插件加载主链路见 [`plugin.md`](plugin.md)，组件加载/持久化见 [`component.md`](component.md)，服务的定义/消费宏与 registry 见 [`service.md`](service.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [动手实证：编译器与链接器的真实行为](#动手实证编译器与链接器的真实行为)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 全部扩展机制——插件、组件、服务——本质上是同一件事：**用 `dlopen` 把外部编译产物拉进服务器进程**。插件是 `.so` 里的一批 `st_mysql_plugin` 描述符，组件是 `.so` 里的 `mysql_component_t` + 一堆函数指针表。一旦代码跨了动态库边界，"布局"和"符号"就从编译器的事变成了**契约**：服务器按自己的 `st_mysql_plugin` 布局去读插件内存，插件按自己的 `st_mysql_audit` 布局接服务器调用，任何一方悄悄改一个字段，轻则数据错乱、重则崩溃。ABI 设计要回答四个问题：**边界上放什么、不放什么（设计哲学）**；两端版本对不上时怎么安全发现、怎么尽量兼容、怎么把兼容变成可验证的工程纪律。

### 为什么这题考 C++ 功力

一个关键事实：**插件的契约面是 C 的，实现面却是 C++ 的**。InnoDB、X plugin、全部内置组件都是用 C++ 写的——但它们暴露给服务器的那一侧（`st_mysql_plugin`、`SERVICE_TYPE(x)`）必须退化成"C 风格"的结构体与函数指针。原因见下一章：**C++ 标准只定义 API（语义），不定义 ABI（布局/链接/异常）**，而 C 的结构体布局与函数调用约定在给定平台上是稳定的。于是"哪些 C++ 特性可以出现在边界上、哪些绝对不行"就成了这套体系的工程核心，这正是它考 C++ 功力的地方。

### 三套契约

| ABI | 载体 | 版本协商 |
|---|---|---|
| 插件库级 | `st_mysql_plugin` 布局 + `_mysql_plugin_interface_version_` 导出符号 | 双字节整数：major 必须相等、minor 向后兼容；旧结构按导出 `sizeof` 逐字段 memcpy 迁移 |
| 插件类型级 | `plugin->info` 指向的类型 vtable，**首字段必须是 `int interface_version`** | 同样的双字节规则；SE/DAEMON/I_S 三类直接绑定服务器版本（`MYSQL_VERSION_ID << 8`） |
| 组件服务级 | `SERVICE_TYPE(name)` 生成的 const 函数指针表 | **无版本号**——"改接口必须换名字"（`psi_thread_v1..v7` 多代并存） |

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.1 | 插件 ABI 诞生：`st_mysql_plugin` + 接口版本 + `_mysql_sizeof_struct_st_plugin_` 迁移 |
| 5.x | 老版 plugin service 的"版本整数塞进指针初值"手法 |
| 8.0 | 组件服务 ABI：const 表 + `noexcept` + 不透明句柄 + "版本=改名"；`.pp` 指纹门禁（`cmake/do_abi_check.cmake`） |
| 8.0.22+ | `psi_thread_v4/v5/v6/v7` 并存，改名版本化的首次大规模实践 |

---

## 理论基础

### 1. 设计思想：跨边界是头号工程问题

ABI 体系的一切选择，可以从一条第一性原理推出来：**跨边界的每一寸接口面积，都是两端独立演化的敌人**。MySQL 所有"反 C++ 直觉"的做法——契约面禁 STL、句柄不透明、导出面只有三个符号、插件类型用 `#define`——本质上是同一个动作：**把跨边界的东西压到最小，把自由留给边界之内**。下面的八条设计决策，都是这条原理在不同维度上的展开。

**① 极简契约观：接口面积与演化自由度成反比。**

边界上每暴露一个 C++ 类型、虚类、STL 容器，就是把两端的编译环境（编译器版本、标准库版本、编译选项）捆绑一次：`std::string` 进签名 = 双方必须同一份 libc++/libstdc++（实验 1 的符号名是实证）；虚类 = 双方虚表布局必须一致（实验 4）；异常 = 双方异常表机制必须兼容（实验 3）。因此契约被压到最小充分集：

- 布局面：一个 `st_mysql_plugin`（14 个 C 字段）+ vtable 首字段约定；
- 符号面：插件导出 3 个固定名符号、组件导出 1 个 `list_components`；
- 类型面：C 基础类型 + 不透明句柄；
- 能力面：服务是"无状态函数表"，粒度小到"改名即换契约"。

代价是每次跨边界多一层转换（`std::string` ↔ `my_h_string` ↔ 实现内部）——`mysql_string` 服务家族存在的全部理由；收益是两端可以独立换编译器、换库、换版本而互不感知。**接口面积与演化自由度成反比**，这是整个 ABI 设计的第一条权衡曲线。

**② 信任模型：进程内 = 全信任，被否决的隔离路线。**

一个往往被"怎么做"淹没的"为什么"：插件/组件跑在**服务器进程内、同一地址空间**，还能直呼服务器全部导出符号（`--export-dynamic` 是刻意的）。MySQL 在 ABI 层选择了**全信任模型**——坏插件写坏内存 = 整个服务器崩，零内存保护。

为什么否决"进程外插件 + IPC"的隔离路线？以存储引擎为例：`ha_innobase` 的每行读取都是进程内虚调用，一旦隔离，每次 `index_next` 都是跨进程往返，性能不可接受（与 ⑦ 的"热路径不走抽象"是同一个理由）。妥协后的防线只有三道：加载路径白名单（`check_valid_path` 禁路径分隔符、path filter 锁死 `plugin_dir`）、安装权限（INSERT_ACL）、"只装可信插件"的运维纪律——**信任问题被交给 DBA，而不是交给链接器**。

生态对照：PostgreSQL 扩展、nginx/Apache 模块、Node 原生模块都是同样的进程内信任模型——性能优先于隔离是扩展机制的普遍默认；反面是 Chrome 的进程外插件路线（PPAPI，已整体废弃）与浏览器扩展的受限 API 沙箱。MySQL 站在"数据库需要极致性能"一侧。

**③ 三条锁：布局、存在、行为——接口与实现分离的 ABI 版。**

接口与实现分离（d-pointer/Pimpl）在 ABI 层面展开成三把锁，各锁各的：

| 锁 | 锁什么 | 手段 | 可靠性 |
|---|---|---|---|
| 布局锁 | 跨边界的字节排布 | 版本号 + sizeof 导出 + 尾部追加迁移 | 代码可强制 |
| 存在锁 | 哪些符号能被对方摸到 | 导出面收窄（3 符号 / `list_components`）+ 符号名稳定 | 代码可强制 |
| 行为锁 | 调用语义 | **文档承诺**——唯一靠不住代码的锁 | 靠人 |

行为锁的脆弱性（代码管不住语义）正是 `.pp` 门禁只能卡结构、卡不住语义的根源：**ABI 设计能工程化的只有布局与存在，行为只能由版本号承载"人为承诺"**。把三把锁分清，"为什么版本号防不住行为变化"这类困惑自然消解。

**④ 声明式契约："声明即真相"，先看契约、后初始化。**

插件用静态数组（`_mysql_plugin_declarations_[]`）、组件用静态 `mysql_component_t`（provides/requires 是编译期数组）声明自己——**没有任何运行时注册 API**。这是刻意的声明式设计：加载器在**调用 init 之前**就能完整读到契约，于是做到三件运行时注册做不到的事：(1) 先校验后初始化——版本不符的库一行代码都不会执行；(2) 依赖解析先于初始化——requires 回填在 init 之前完成，init 里用到的 `mysql_service_xxx` 指针此刻已就位；(3) 失败可成批回滚——init 失败前已知全部依赖，可以整体撤销。

对照命令式路线（JNI `RegisterNatives`、Python C API 的运行期注册）：灵活，但"契约何时完整可用"没有答案，初始化顺序脆弱。MySQL 选声明式，因为它的场景是"服务器主导的批量加载"，不是"宿主与插件逐步协商"。

**⑤ 兼容策略的三条路线，MySQL 为何各选各的。**

"两端版本不一致时怎么办"有至少三条成熟路线，插件与服务恰好各走一条，第三条被整体否决：

| 路线 | 代表 | 机制 | 代价 |
|---|---|---|---|
| 结构性兼容 | glibc `.symver`、内核 syscall 表 | 尾部追加 + 按 sizeof 迁移 | 结构演化锁死在"只增不删" |
| 能力协商 | COM `QueryInterface` | 运行时按 GUID 逐接口询问 | 每个对象带协商代码、接口爆炸 |
| 名字版本化 | REST v1/v2 | 换名 = 换契约，新旧并存 | 二进制膨胀、弃用是文档问题 |

插件库级走**结构性兼容**（`st_mysql_plugin` 尾部追加 + `__reserved1` 预留坑位）——类型 slot 全局唯一，只能单线演化；组件服务走**名字版本化**（`psi_thread_v1..v7`）——registry 天然多代并存；**能力协商被否决**：服务粒度已经小到"一张无状态函数表"，`QueryInterface` 式逐接口询问的收益（省几个函数指针）远小于代价（每次调用前协商、接口身份从名字变 GUID）。

**⑥ 失败语义的三态：由数据结构逼出的哲学。**

同一个"有人在用"的事实，插件、组件、加载器给出三种失败语义——不是随意，是被各自的数据结构逼出来的：

- **插件卸载 → 延迟收割**：`st_plugin_int` 是全局单例（名字在哈希里唯一），"标记 DELETED + ref 归零后 reap"是保证任意线程句柄不悬空的唯一办法；
- **组件卸载 → 拒绝**：服务是多实现市场，引用计数 > 0 直接报错——"没人用才能退场"是市场里干净的规则；
- **组件加载 → 事务式回滚**：依赖不满足 / init 失败就整批撤销——声明式契约让"全知"成为可能。

共同底色：**ABI 体系的失败语义跟随数据结构形态，不跟随用户直觉**。这也解释了为什么不能要求插件像组件一样"拒绝卸载"——单例没有"换个实现"的退路。

**⑦ 调用形态三分层：耦合度与调用频率成正比。**

同一类"跨边界调用"，MySQL 存在三种形态并存，是刻意的分层：

```
服务器内部（PSI 热路径）    psi_thread_service 全局符号直连    最快、最强耦合
插件（引擎/审计热路径）    st_mysql_audit 类型 vtable         中间态
组件（低频生命周期）       registry acquire 服务             最慢、最解耦
```

"热路径不走抽象"与"冷路径不共享符号"同时成立：PSI 每秒百万次，走 registry 的 map 查找不可接受；组件的日志/UDF 注册是低频操作，值得用 acquire 换解耦。**ABI 边界画在哪，由调用频率决定**——这是对"为什么不全盘服务化"最直接的回答。

**⑧ 演进后门：每个机制留了什么、没留什么。**

设计者给每条演进轴都留了后门，也划了死线：

- 布局轴：尾部追加是唯一合法方向，`__reserved1` 是预挖坑位（历史上想留作依赖检查）；
- 类型轴：`MYSQL_MAX_PLUGIN_TYPE_NUM = 12` 封顶，**第三方无法注册新类型**——因为类型的分派点（`acl_authenticate`、`ha_resolve_by_name`…）是服务器内生的调用代码，新类型必须改服务器。这是"角色（类型）封闭、能力（服务）开放"的根本分界；
- 服务轴：改名即版本化，弃用靠头文件注释声明（`psi_thread_service.h` 的 "Deprecated in 8.0.25"）。

死线同样明确：`st_mysql_plugin` 中间插删字段、类型 slot 复用、服务同名改语义——三者从未发生，因为它们分别击穿布局锁、存在锁、行为锁。

### 2. C++ 没有 ABI 标准，只有事实标准

ISO C++ 标准规范的是**语言语义**（API 层），对对象布局、名字修饰、虚表结构、异常展开机制**只字未提**——它们全部是实现细节。现实中有两个互不兼容的"事实标准"：

- **Itanium C++ ABI**：GCC/Clang 在 Linux 上的约定（名字修饰规则、虚表布局、RTTI、异常表格式），以规范文档形式存在但**不属于 C++ 标准**；
- **MSVC ABI**：微软自成一派，且**不同 MSVC 大版本之间 ABI 也曾断裂**。

直接推论：**跨动态库边界传递任何"编译器相关的 C++ 构造"（虚类、STL 容器、异常、带非平凡构造的类）都是未定义行为**。同一头文件换编译器、换编译选项（甚至换 `-fshort-enums` 一个开关）就可能产生不同布局。

MySQL 的对策是分层：

- **契约面全面 C 化**：服务只许用 C 基础类型（`service.h` 原文：`Each Service should work only on basic C types ... to prevent problems with not fully-defined C++ objects ABI`）、句柄一律是不透明指针、异常不许越过边界；
- **实现面随意 C++**：边界内（每个 `.so` 内部）随便用 `std::`、虚函数、异常——只要在自己的 `try/catch` 里消化干净。

这解释了为什么 `mysql_string` 是一整个服务家族（15 个服务）：组件不能把 `std::string` 当参数传，只能传 `my_h_string` 句柄。

### 3. 五个 ABI 不稳定点，MySQL 逐一对应

| # | C++ 机制的不稳定点 | MySQL 的对应源码 |
|---|---|---|
| ① | **名字修饰（name mangling）**：函数/变量按类型编码进符号名；**MSVC 对变量也做 mangling**（GCC 只对函数） | `MYSQL_PLUGIN_EXPORT` 在 MSVC + `MYSQL_DYNAMIC_PLUGIN` 下展开为 `extern "C" __declspec(dllexport)`（`plugin.h` 注释原文："unlike other compilers, uses C++ mangling for variables not only for functions"）；组件的 `list_components()` 同样靠 `DLL_EXPORT extern "C"` 保证 dlsym 可见 |
| ② | **虚表布局**：vtable 槽位、typeinfo 指针偏移、虚基类调整，全无标准 | 契约面零虚类；`DEFINE_SERVICE_HANDLE` 注释要求连 C++ RAII 包装类都**不得有虚函数**（"make sure it does not use any virtual methods to keep binary compatibility"） |
| ③ | **RTTI**：typeinfo 名字、`dynamic_cast` 跨 DSO 比较不可靠 | 契约面不依赖 RTTI；插件类型是 `#define` 常量而非 enum/typeid，类型级版本靠 `*(int*)plugin->info` **手工读首字段**（"手工 RTTI"） |
| ④ | **异常**：`.eh_frame`/LSDA/personality routine 因编译器而异，异常越过 DSO 边界 = `std::terminate` | `DEFINE_METHOD` 强制 `noexcept`；实现函数统一 `try/catch` + `mysql_components_handle_std_exception` 就地消化（见核心实现③） |
| ⑤ | **非平凡类型**：ctor/dtor 代码生成在双方都要一致，布局依赖库实现版本 | 契约面只许 trivially copyable 结构；`st_mysql_plugin` 的 memcpy 迁移以"POD + 尾部零填充"为前提（见核心实现⑥） |

### 4. C ABI 的"稳定"是有条件的——C 自己的坑

C ABI 稳定是相对 C++ 而言，它自己也有实现定义的地方。MySQL 的 ABI 头文件是刻意躲开这些坑写出来的：

| C 的坑 | 出问题的方式 | MySQL 的规避（源码证据） |
|---|---|---|
| **枚举底层类型** | C++11 前枚举大小 implementation-defined（`-fshort-enums` 下可以是 1 字节） | 插件类型**不是 enum**，是 `#define MYSQL_AUDIT_PLUGIN 5` 这一串 int 常量（`plugin.h:113-125`）；`st_mysql_plugin.type` 字段是 **`int`** |
| **位域布局** | 位域的存储单元、方向全 implementation-defined | 全库 ABI 头文件（`st_mysql_plugin`、`st_mysql_audit`、`SERVICE_TYPE` 表）**零位域** |
| **`bool` 大小** | C89 无 bool；`_Bool`/`bool` 的实现差异 | 插件 API 的布尔语义用 `int`/`mysql_service_status_t`（`typedef int`）表达，函数指针返回 `int` |
| **`long` 宽度** | LP64（Linux 8 字节）vs LLP64（Windows 4 字节） | `my_inttypes` 归一；`MYSQL_ABI_CHECK` 宏在 50+ 头文件里屏蔽平台相关 typedef，保证 `.pp` 指纹跨平台可比 |
| **结构体 padding** | 对齐规则平台相关 | ABI 结构体按 8 字节对齐的自然顺序排字段，不玩 `#pragma pack`；`st_mysql_plugin` 的 `__reserved1` 预留尾部坑位（尾部追加是唯一安全演化方向） |

### 5. 动态链接层面：符号从哪来、谁看得见谁

**服务器侧：全部符号导出。** `cmake/mysql_add_executable.cmake` 给 mysqld 加 `ENABLE_EXPORTS`，注释直白：

```cmake
ENABLE_EXPORTS     # For Linux, link with: -Wl,--export-dynamic -rdynamic
                   # This option is needed for some uses of "dlopen" ...
```

`--export-dynamic` 把 mysqld 的**全部全局符号**放进动态符号表——这是"插件能直呼服务器符号"（WL#4102 批判的 wormhole 缺陷）的技术基础：Linux 上插件不用链接 mysqld 就能解析到它的任何全局函数。注意组件的"不链接 mysqld"是**纪律**而非链接器强制——符号照样可见，封装靠约定。

**插件/组件侧：默认隐藏。** `cmake/component.cmake` 给组件强制加编译选项，`cmake/plugin.cmake` 提供 `VISIBILITY_HIDDEN` 选项：

```cmake
# component.cmake
SET(COMPONENT_COMPILE_VISIBILITY
    "-fvisibility=hidden" CACHE INTERNAL
    "Use -fvisibility=hidden for components" FORCE)
TARGET_COMPILE_OPTIONS(${target} PRIVATE "-fvisibility=hidden")
# plugin.cmake
IF(BUILD_PLUGIN AND ARG_VISIBILITY_HIDDEN AND UNIX)
  TARGET_COMPILE_OPTIONS(${target} PRIVATE "-fvisibility=hidden")
```

`-fvisibility=hidden` 下只有显式标 `__attribute__((visibility("default")))` 的符号能进动态符号表。插件只导出三个固定名字的符号（`_mysql_plugin_interface_version_`、`_mysql_sizeof_struct_st_plugin_`、`_mysql_plugin_declarations_`），组件只导出 `list_components`——**导出面最小化**，同时避免内部符号与服务器符号互相 interpose（GNU 符号介入规则下，同名全局符号按链接/加载顺序先到先得，隐藏后就没了这个风险）。

### 6. 对象生命周期跨 DSO

ABI 不只在"布局"，也在"生命周期"：

- **分配/释放必须同源**：A DSO malloc、B DSO free 只在共享同一 CRT/分配器时安全。这就是为什么跨边界传字符串要用 `mysql_string_factory` 这类**工厂服务**（谁创建谁销毁都走服务调用），而不是裸 `char*` 约定；插件自己的内存走服务器管理的 `st_plugin_int::mem_root`。
- **静态初始化顺序灾难（SIOF）跨 DSO 更隐蔽**：`dlopen` 的那一刻会执行 `.so` 的 `.init_array`（C++ 全局对象的构造函数）。服务器**从不依赖**插件/组件的静态初始化——它们的初始化是显式回调（插件 `init` 函数、组件 `mysql_component_t::init`）；反过来说，插件里任何全局对象的构造失败都会让 `dlopen` 直接失败，报 `ER_CANT_OPEN_LIBRARY`。组件文档的 Tutorial 甚至明说"用原子变量 `is_intialized` 表示组件状态，加载/初始化完成前别用它的服务"（`component_implementation.h` 教程原文）。
- **卸载与析构**：`dlclose` 触发 `.fini_array`；组件 loader 在 ASAN/LSAN 构建下给 `dlopen` 加 `RTLD_NODELETE`（见核心实现④），让 `dlclose` 不真正卸载——LeakSanitizer 需要符号还在才能给出调用栈。这是"调试器/消毒器改变 ABI 行为"的活例子。

### 7. "版本数字 + 迁移" vs "版本=改名"：根因在命名空间的唯一性

两条路线的取舍不是口味问题，而是数据结构决定的（细节见 [`service.md`](service.md)）：

- 插件类型的 **slot 全局唯一**（`MYSQL_AUDIT_PLUGIN == 5`，同一服务器同一时刻只有一代 `st_mysql_audit` 布局生效）→ 必须"先协商一致再进同一 slot" → 数字版本 + memcpy 迁移；
- 服务实现**天然多代共存**（registry 里 `psi_thread_v4` 与 `psi_thread_v7` 同时注册）→ 改名即可，零迁移逻辑，代价是二进制膨胀（174 个服务里 PSI 家族约占 1/4）。

### 理论溯源

- **Itanium C++ ABI**（GCC/Clang 事实标准）：名字修饰、虚表、RTTI、异常表的约定来源。
- **System V AMD64 ABI**：LP64 数据模型、结构体对齐/传参约定（C 结构体布局稳定的真正依据）。
- **Pimpl / d-pointer**（Sutter, *GotW #100*）：`DEFINE_SERVICE_HANDLE` 注释直接引用 Wikipedia Opaque pointer 条目。
- **符号可见性与介入（interposition）**：Drepper《How To Write Shared Libraries》——`-fvisibility=hidden` 与 `--export-dynamic` 的经典论述。
- **Semantic Versioning**：双字节 major/minor 规则的心智来源。

---

## 核心实现

### ① 语言链接与符号导出：`MYSQL_PLUGIN_EXPORT` 的四分支

```c
/* include/mysql/plugin.h */
#if defined(_MSC_VER)
#  if defined(MYSQL_DYNAMIC_PLUGIN)
#    ifdef __cplusplus
#      define MYSQL_PLUGIN_EXPORT extern "C" __declspec(dllexport)
#    else
#      define MYSQL_PLUGIN_EXPORT __declspec(dllexport)
#    endif
#  else ...
#else /*_MSC_VER */
#  if defined(MYSQL_DYNAMIC_PLUGIN)
#    define MYSQL_PLUGIN_EXPORT MY_ATTRIBUTE((visibility("default")))
#  else
#    define MYSQL_PLUGIN_EXPORT
#  endif
#endif
```

逐条拆：

- **MSVC 分支必须 `extern "C"`**：GCC 只对函数做 C++ 修饰，变量符号名不受影响；MSVC 把变量也按 C++ 规则修饰，不加 `extern "C"` 则 dlsym 按 C 名字找不到 `_mysql_plugin_declarations_`。注释原话："unlike other compilers, uses C++ mangling for variables not only for functions"。
- **`__declspec(dllexport)` vs `visibility("default")`**：两平台各自的导出声明语法，语义相同——把符号放进动态符号表。
- **builtin 形态两个分支都为空**：静态链接进 mysqld 时根本不需要导出——符号在同一个链接单元里，直接按名字引用（`sql_builtin.cc.in` 生成的 `builtin_<name>_plugin[]` 数组）。这再次印证"builtin/dynamic 同构"：**同一份 ABI，两种到达方式**。

### ② 符号可见性与介入风险

组件的 `-fvisibility=hidden`（`component.cmake`，见上）与插件的 `VISIBILITY_HIDDEN` 选项（`plugin.cmake`）解决三个问题：

1. **减少 PLT/GOT 重定位**：隐藏符号走内部调用，不经过动态链接器；
2. **防止符号介入（interposition）**：两个 DSO 定义同名全局函数时，GNU 链接规则按"先到先得"全局共享一个——插件里恰好有个函数叫 `my_hash` 就会静默替换（或被替换）服务器版本；hidden 把内部符号全部私有化；
3. **导出面最小化**：`dlsym` 只能摸到 3 个插件符号 / 1 个 `list_components`，其他内部实现不可被外部探知或依赖。

与服务器侧 `--export-dynamic` 形成对照：服务器全量导出（插件依赖它解析符号），插件/组件最小导出（服务器只依赖固定入口符号）。**不对称是刻意的**——插件对服务器是"开放依赖"，服务器对插件是"封闭入口"。

### ③ 异常的三道防线

**第一道：`noexcept` 契约。** `service_implementation.h`：

```c
#define DEFINE_METHOD(retval, name, args) retval name args noexcept
```

服务实现函数必须是 `noexcept`——函数指针表本身声明了不抛异常的承诺，调用方（服务器）**不会**为这些调用生成异常处理代码，异常越过边界即 `std::terminate`。

**第二道：实现内部 try/catch 兜底。** 每个服务方法的骨架（例 `sql/server_component/mysql_current_thread_reader_imp.cc`）：

```cpp
DEFINE_BOOL_METHOD(mysql_component_mysql_current_thread_reader_imp::get,
                   (MYSQL_THD * thd)) {
  try {
    if (thd) *thd = static_cast<MYSQL_THD>(current_thd);
  } catch (...) {
    mysql_components_handle_std_exception(__func__);
  }
  return false;
}
```

**第三道：`mysql_components_handle_std_exception` 的重抛惯用法**（`components/libminchassis/component_common.cc`，贴全）：

```cpp
void mysql_components_handle_std_exception(const char *funcname) {
  try {
    throw;                                  /* ← 裸 throw：重抛当前异常 */
  } catch (const std::bad_alloc &e) {
    mysql_error_service_printf(ER_STD_BAD_ALLOC_ERROR, MYF(0), e.what(), funcname);
  } catch (const std::domain_error &e) {
    mysql_error_service_printf(ER_STD_DOMAIN_ERROR, MYF(0), e.what(), funcname);
  }
  ... /* length/invalid_argument/out_of_range/overflow/range/underflow ... */
  } catch (const std::logic_error &e) {
    mysql_error_service_printf(ER_STD_LOGIC_ERROR, MYF(0), e.what(), funcname);
  } catch (const std::runtime_error &e) {
    mysql_error_service_printf(ER_STD_RUNTIME_ERROR, MYF(0), e.what(), funcname);
  } catch (const std::exception &e) {
    mysql_error_service_printf(ER_STD_UNKNOWN_EXCEPTION, MYF(0), e.what(), funcname);
  } catch (...) {
    mysql_error_service_printf(ER_UNKNOWN_ERROR, MYF(0));
  }
}
```

两个 C++ 细节：

- **`throw;` 重抛惯用法**：只能在 catch 块内使用的裸 `throw` 语句，把"当前正在传播的异常"重新抛出——于是**类型匹配发生在异常诞生地（本 DSO 的 catch）**，而不是跨边界靠 RTTI 匹配（跨 DSO 的 typeinfo 比较恰恰是 RTTI 不可靠的那一类场景）；
- **按异常层级从具体到笼统 catch**：`std::bad_alloc` → 各 `logic_error/runtime_error` 子类 → 两个基类 → `catch(...)`。每个异常类型映射一个专用错误码（`ER_STD_BAD_ALLOC_ERROR` 等），把 C++ 异常体系**翻译成** MySQL 错误码体系，异常的类型信息最终以日志文本落盘——信息没有丢，但传播止步于边界。

### ④ dlopen：标志位与静态初始化

组件侧 `components/libminchassis/dynamic_loader_scheme_file.cc`：

```cpp
#if defined(HAVE_ASAN) || defined(HAVE_LSAN)
    // Do not unload the shared object during dlclose().
    // LeakSanitizer needs this in order to provide call stacks,
    // and to match entries in lsan.supp.
    void *handle = dlopen(file_name.c_str(), RTLD_NOW | RTLD_NODELETE);
#else
    void *handle = dlopen(file_name.c_str(), RTLD_NOW);
#endif
```

- **`RTLD_NOW`**：加载期解析全部未决符号。与 `RTLD_LAZY`（首次调用才解析）相比，缺符号会在 `dlopen` 当场报错（`ER_CANT_OPEN_LIBRARY`），而不是运行到一半崩——插件侧 `plugin_dl_add` 同样用 `RTLD_NOW`，两代机制一致。
- **`RTLD_NODELETE`（仅 ASAN/LSAN）**：让 `dlclose` 不真正卸载——消毒器需要符号表还在才能给出泄漏调用栈。**检测工具会改变 ABI 行为**（卸载语义被停用），这是把"可诊断性"置于"语义纯粹性"之上的典型工程取舍。
- `dlopen` 本身就执行 `.init_array`：组件/插件的 C++ 全局对象在此刻构造。随后 loader 还会做**去重**（`object_files_list` 按 URN 查 + `library_entry_set` 按 `list_components` 函数地址查），防止同一 `.so` 被二次加载导致全局对象构造两遍。

### ⑤ 类型规约的源码证据：为什么是 `#define` 不是 `enum`

```c
/* include/mysql/plugin.h */
#define MYSQL_UDF_PLUGIN 0
#define MYSQL_STORAGE_ENGINE_PLUGIN 1
...
#define MYSQL_CLONE_PLUGIN 11
#define MYSQL_MAX_PLUGIN_TYPE_NUM 12
...
#define PLUGIN_LICENSE_PROPRIETARY 0
#define PLUGIN_LICENSE_GPL 1
#define PLUGIN_LICENSE_BSD 2
```

`st_mysql_plugin.type`、`license` 都是 `int` 字段。这是**为 ABI 而写的代码**：enum 的底层类型在 C++11 前是 implementation-defined（`-fshort-enums` 可以让它变成 1 字节），`#define` 则是铁打的 int 常量——服务器按 `int` 读插件按 `int` 写，永不错位。同理，类型级校验读的是 `*(int *)plugin->info`，vtable 首字段被约定成 `int` 而非任何枚举。全库搜不到"插件类型 enum"——这个"没有"本身就是设计。

### ⑥ trivially copyable 与旧结构迁移

`plugin_dl_add` 的旧版本兼容路径（全文见 [`plugin.md`](plugin.md) 的核心实现）：

```c
if (plugin_dl.version != MYSQL_PLUGIN_INTERFACE_VERSION) {
  sizeof_st_plugin = *(int *)dlsym(handle, "_mysql_sizeof_struct_st_plugin_");
  ...
  cur = my_malloc((i + 1) * sizeof(st_mysql_plugin), MYF(MY_ZEROFILL | MY_WME));
  for (i = 0; (old = (st_mysql_plugin *)(ptr + i * sizeof_st_plugin))->info; i++)
    memcpy(cur + i, old, min<size_t>(sizeof(cur[i]), sizeof_st_plugin));
}
```

C++ 层面的合法性依据：`st_mysql_plugin` 是 **trivially copyable**（无虚函数、无用户定义构造/析构、成员全平凡）——`memcpy` 字节搬运语义正确，且注释明说零填充"matches C standard behaviour for struct initializers that have less values than the struct definition"。**反事实推演**：若往 `st_mysql_plugin` 里塞一个 `std::string name`，(a) 结构不再 trivially copyable，memcpy 复制出的 `name` 与原件共享堆指针、析构双重释放；(b) 布局依赖 libstdc++ 版本，`sizeof` 导出失效；(c) 跨 DSO 构造/析构 undefined behavior。一个字段的"现代化"会同时击穿这套迁移机制的三层假设——这正是 ABI 头文件宁可写 `const char *name` + 手动管理也不碰 STL 的原因。

### ⑦ 服务 ABI 三件套（const 表 / noexcept / d-pointer）

三件装置各对应一个理论点（宏定义见 [`service.md`](service.md)）：

1. **`const`**（`SERVICE_TYPE(x) = const mysql_service_x_t`）：服务实现是静态初始化的**只读**函数指针表，落在 `.rodata` 段——运行时不可篡改（防恶意组件改自己的接口表骗过引用方），且在 fork/多实例场景下只读页天然共享。
2. **`noexcept`**：见 ③ 第一道防线。
3. **`DEFINE_SERVICE_HANDLE(name) = typedef struct name##_imp *name`**：`struct my_h_service_imp` 在源码树**不存在**（grep 0 命中，有意为之）——实现对象（`mysql_service_implementation`）的布局完全锁在 `libminchassis` 内部，消费者连它多大都不知道，只能通过 registry API 使用句柄。d-pointer 的极端形态：连指针所指类型的**前向声明**都只活在 `typedef` 里。

### ⑧ ABI 指纹门禁（`cmake/do_abi_check.cmake`）

流程（文件头注释自述）：`gcc -E -nostdinc -dI -DMYSQL_ABI_CHECK -I<include>` 对每个 ABI 头文件生成预处理输出 → `sed` 去 `#` 行/空行/OS 杂讯 → diff 仓库金丝雀 `include/mysql/plugin.pp` → 失败留下 `abi_check.out`，开发者**有意变更时** `mv abi_check.out include/mysql/plugin.pp` 签认新基线。

定位辨析：这是**文本指纹**检查，不是符号级检查（`abidiff`/`abi-compliance-checker` 比较符号表与布局，是另一类工具）。它的能力边界：

- **能发现**：布局改动、类型改名、宏改值、新增/删除字段——任何落到预处理文本里的差异；
- **发现不了**：语义变化（同签名不同行为）。这与运行时校验是同构的盲区——**版本号/指纹都是"人为声明"的兼容性承诺，不是从行为自动推导的**。工程上两条腿都要：门禁防"无意改动"，版本号+迁移管"有意演化的兼容"。

---

## 动手实证：编译器与链接器的真实行为

> 本章全部实验在 macOS（Apple clang / libc++ / Mach-O）上真实运行，输出原文保留。选 macOS 而非 Linux 是故意的——**平台差异本身就是 ABI 的一部分**（`nm -D` 是 ELF 专用命令，Mach-O 用 `nm -g`；Mach-O 符号带 `_` 前缀；macOS 链接器默认拒绝 dylib 中的未定义符号，Linux 则放行）。

### 实验 1：名字修饰（mangling）——符号名里藏着标准库身份

源文件（模拟插件的内部函数）：

```cpp
namespace mysql_plugin {
struct st_mysql_plugin { int type; const char *name; unsigned int version; };
int plugin_init(void*) { return 0; }
void plugin_api(st_mysql_plugin&, std::string s) { std::printf("%s\n", s.c_str()); }
}
```

`nm` 输出（节选）：

```
T __ZN12mysql_plugin10plugin_apiERNS_15st_mysql_pluginENSt3__112basic_stringIcNS2_11char_traitsIcEENS2_9allocatorIcEEEE
T __ZN12mysql_plugin11plugin_initEPv
```

解读（Itanium 修饰规则逐段拆第一个符号）：

- `_Z`：C++ 修饰前缀；
- `N12mysql_plugin...E`：嵌套名字空间编码（`N` 开始、`E` 结束，`12mysql_plugin` = 长度 12 + 名字）；
- `R`/`P`：引用/指针修饰符；
- `St3__112basic_string...`：**`std::__1::basic_string`——libc++ 的内联名字空间 `__1` 被编进了符号名**。同一个 API 文本，若用 libstdc++ 编译则是 `St7__cxx1112basic_string...`（`__cxx11` ABI 标签），符号名完全不同。

**这意味着什么**：`plugin_api` 的调用双方只要用了不同的标准库（或同一标准库的不同 ABI 版本，如 GCC 5 的 `__cxx11` 迁移），链接器看到的就根本不是同一个函数——**"API 兼容"在 C++ 里不蕴含"ABI 兼容"**。这就是为什么 MySQL 的契约面禁止 `std::string` 出现在服务签名里，而 `mysql_string` 要单独成为一整个服务家族。

### 实验 2：可见性——`-fvisibility=hidden` 前后的导出面

```cpp
extern "C" __attribute__((visibility("default"))) int exported_fn() { return 1; }
int hidden_fn() { return 2; }
```

编译：`g++ -fvisibility=hidden -shared -fPIC vis.cc`。导出符号表（Mach-O 用 `nm -g`）：

```
T _exported_fn          ← 唯一导出
```

`hidden_fn` 完全不在导出表里。对照 MySQL：插件导出的固定三符号（`_mysql_plugin_interface_version_` 等）、组件导出的 `list_components`，就是在这条规则下"点名放行"的——`-fvisibility=hidden`（`component.cmake` 强制、`plugin.cmake` 可选）把导出面收窄到这三个名字，其余全部私有。若不禁，插件/组件的内部符号（上面实验里的 `hidden_fn` 级别）会与服务器符号互相介入（interposition）。

### 实验 3：`noexcept` 被违反时发生什么

```cpp
// libthrower.so
extern "C" void do_throw() noexcept { throw std::runtime_error("boom"); }
// main：dlopen 后通过 dlsym 拿到 do_throw 直接调用，无 try/catch
```

编译警告（这条警告本身就是 `noexcept` 契约的注脚）：

```
warning: 'do_throw' has a non-throwing exception specification but can still throw [-Wexceptions]
```

运行结果：

```
calling noexcept fn...
libc++abi: terminating due to uncaught exception of type std::runtime_error: boom
exit=134                                          ← SIGABRT
```

异常从 `.so` 逃出、调用方没有处理代码 → `std::terminate` → SIGABRT，进程直接死。对照 MySQL 的三道防线：`DEFINE_METHOD` 强制 `noexcept` 把"不许抛"写进函数类型（C++17 起异常规范是函数类型的一部分），实现内部 `try/catch(...)` + `mysql_components_handle_std_exception` 把一切异常就地翻译成 `ER_STD_*` 错误日志——**异常可以发生在任何 DSO 里，但绝不允许穿越边界**。

### 实验 4：vtable 布局——跨 DSO 共享虚表的物理根据

```cpp
struct Base { virtual int f() { return 1; } virtual ~Base() {} };
struct Derived : Base { int f() override { return 2; } };
```

打印对象内存里的 vptr 及 vtable 槽位（Itanium ABI 布局）：

```
vptr of object = 0x1000c8030
vtable[-1] (typeinfo)      = 0x1000c8058   ← RTTI 信息挂在 vtable 前面一格
vtable[0]  (offset-to-top) = 0x0            ← 虚基类指针调整量
vtable[1]  (typeinfo ptr)  = 0x1000c7d0c
vtable[2]  (Derived::f)    = 0x1000c7dac
vtable[3]  (dtor)          = ...
```

三个事实：① 虚调用 = `vptr[2]()` 这样的**槽位间接跳转**，槽位顺序是编译器排的、不在标准里；② 多重继承/虚继承时槽位间还夹着 **thunk**（调整 this 指针的桩函数）；③ typeinfo 藏在 vtable[-1]。三者任一在双方编译器间不一致，虚调用就是跳进随机地址。这就是"契约面零虚类、RAII 包装类也禁止虚函数"的物理根据——MySQL 的服务表本质上是一张**手工布局的、槽位自明的**函数指针表，把"虚表"这一层从编译器手里收归自己。

### 实验 5：`RTLD_LOCAL`——符号隔离的链接层机制

`liba.so` 定义 `foo()`，`libb.so` 引用 `foo()`（macOS 上需 `-Wl,-undefined,dynamic_lookup` 才能编出带未决符号的 dylib）。先 `RTLD_LOCAL` 加载 a，再加载 b：

```
a=0x910f4510 b=0x0
err=dlopen(./libb.so, 0x0006): symbol not found in flat namespace '_foo'
```

`RTLD_LOCAL` 下 a 的符号**不进全局查找表**，b 解析不到 `foo`，`dlopen` 直接失败。换成 `RTLD_GLOBAL` 加载 a 再加载 b：

```
a=0x929e0510 b=0x929e09e0 err=(null)
call_foo() = 42
```

对照 MySQL：插件 `plugin_dl_add` 和组件 `dynamic_loader_scheme_file` 的 `dlopen` **都不带 `RTLD_GLOBAL`**——所以插件之间、组件之间**互相看不见对方的符号**，它们唯一共享的符号来源是主程序（mysqld 的 `--export-dynamic`）。"插件只能对服务器说话"这句话在链接层是真实存在的物理隔离，不是描述性比喻。

### 实验 6：`dlopen` 的那一刻，全局对象已经构造

```cpp
// libinit.so
struct S { S() { std::printf("static ctor runs at dlopen\n"); } };
S s;                                  // 静态存储期全局对象
extern "C" void entry() {}
```

运行：

```
before dlopen
static ctor runs at dlopen          ← .init_array 在 dlopen 内执行
after dlopen h=0x912c0510
```

`dlopen` 返回前，`.so` 里所有静态存储期对象的构造函数已经跑完（等价于执行了 `.init_array` 段）。两个推论落到 MySQL 源码上：

- **服务器从不依赖插件的静态初始化**——插件的初始化是显式回调（`plugin->init` / `mysql_component_t::init`），静态初始化顺序灾难（SIOF）被整体规避；但插件里全局对象**构造失败**会让 `dlopen` 当场失败，报 `ER_CANT_OPEN_LIBRARY`；
- **卸载顺序对应**：`dlclose` 触发析构；ASAN/LSAN 构建下组件 loader 加 `RTLD_NODELETE`，让"卸载"不再真正发生（LeakSanitizer 需要符号还在才能报调用栈）——检测工具会改变 ABI 层的行为，这是工程取舍而非 bug。

### 实验小结：理论与源码的对应关系

| 实验 | 证明的理论点 | MySQL 源码落点 |
|---|---|---|
| 1 mangling | C++ 符号名嵌入实现身份（libc++ `__1` / libstdc++ `__cxx11`），API 兼容 ≠ ABI 兼容 | 契约面禁 STL；`mysql_string` 服务家族 |
| 2 visibility | 导出面可控 | `-fvisibility=hidden` + 三符号点名放行 |
| 3 noexcept 违反 | 异常越界 = `std::terminate` | `DEFINE_METHOD noexcept` + rethrow 惯用法就地消化 |
| 4 vtable | 槽位布局编译器私有 | 契约面零虚类；服务表=手工布局的函数指针表 |
| 5 RTLD_LOCAL | 符号隔离是链接器行为 | 插件/组件互相不可见符号，只共享 mysqld 导出符号 |
| 6 static init | `dlopen` 即执行全局构造 | 初始化靠显式回调，不靠静态初始化 |

---

## Misc

### 反事实推演：如果 X 会怎样崩

| 如果 | 击穿的机制 | 后果 |
|---|---|---|
| `st_mysql_plugin` 用 `std::string` 字段 | trivially copyable 假设 | memcpy 迁移双重释放；sizeof 导出失效；跨编译器布局漂移 |
| 服务接口用 C++ 虚类 | Itanium/MSVC 虚表差异 | vtable 槽位错位；mangling 对不上；RTTI 跨 DSO 比较 UB |
| 插件类型用 `enum` 传参 | 枚举底层类型 implementation-defined | `-fshort-enums` 下服务器按 int 读 1 字节枚举 → 越界/错位 |
| 服务方法不加 `noexcept` | 异常跨 DSO | 调用方无处理代码 → `std::terminate`；`personality routine` 不一致时栈直接损坏 |
| 组件导出全部符号 | interposition | 同名内部符号劫持服务器符号，静默行为改变 |
| 服务句柄暴露实现类定义 | d-pointer 封装 | 消费者依赖实现布局，libminchassis 无法演进内部结构 |
| 组件依赖 `.so` 的 static 初始化 | SIOF 跨 DSO | 服务器初始化顺序失控；构造函数失败路径不可控 |

### 两套机制的 ABI 盲区

版本数字与改名都管不住"**同签名不同行为**"——一个插件声明 `MYSQL_AUDIT_INTERFACE_VERSION` 不变但 `event_notify` 语义变了，服务器无法察觉。这是所有基于"自我声明"的 ABI 体系的共同盲区，MySQL 的缓解手段只有 `.pp` 门禁卡结构、Code Review 卡语义。

### 易混淆点

- **三个"版本"不是一回事**：`MYSQL_PLUGIN_INTERFACE_VERSION`（`st_mysql_plugin` 布局版本，0x010B）≠ vtable 首字段的 interface version（`st_mysql_audit` 的 0x0401）≠ 插件自己的 `version` 字段（`SHOW PLUGINS` 的 `PLUGIN_VERSION`，纯展示）。
- **两个不等号方向相反**：插件库级——库版本必须 **≥** `min_plugin_interface_version`（太老拒绝）；老版服务——插件声明版本必须 **≤** 服务器提供版本（要的比给的新就拒绝）。别记反。
- **`MYSQL_VERSION_ID << 8`** 被 SE/DAEMON/I_S 三类 vtable 用作版本号：这些接口的 ABI 承诺就是"跟版编译"，第三方引擎必须随服务器版本出包。

---

## 参考

**标准与规范**
- *Itanium C++ ABI*（GCC/Clang 事实标准：名字修饰、虚表、RTTI、异常表），https://itanium-cxx-abi.github.io/cxx-abi/
- *System V Application Binary Interface, AMD64 Architecture Processor Supplement*（LP64、对齐、传参）

**书籍 / 文章**
- Herb Sutter. *GotW #100: Compilation Firewalls*（Pimpl / d-pointer 惯用法）
- Ulrich Drepper. *How To Write Shared Libraries*（符号可见性、interposition、`--export-dynamic`）
- *Microsoft Component Object Model (COM) Specification*（`QueryInterface` 能力协商路线的原型，见「设计思想 ⑤」）
- *glibc Symbol Versioning*（`.symver` 结构性兼容的代表，见「设计思想 ⑤」）：https://sourceware.org/glibc/wiki/SymbolVersioning

**官方文档**
- *MySQL 8.0 Reference Manual → Extending MySQL*（插件与服务 ABI 约定）
- WorkLog: [WL#4102 Service registry and component infrastructure](https://dev.mysql.com/worklog/task/?id=4102)

**相关文档**
- 插件加载主链路见 [`plugin.md`](plugin.md)
- 组件加载/持久化见 [`component.md`](component.md)
- 服务定义/消费宏与 registry 见 [`service.md`](service.md)
