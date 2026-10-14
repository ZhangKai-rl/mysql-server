# ABI（二进制接口）契约深度剖析

> 基于 MySQL 8.0.39 源码。本篇从 C++ 语言/动态链接两个层面，把插件、组件、服务三套机制共用的 ABI 工程手法逐一拆开：为什么契约面必须是 C 的、C++ 的五个 ABI 不稳定点各自怎么被挡在边界外、符号可见性与对象生命周期如何管理。
>
> **边界**：本篇是 ABI 专题的**唯一详写处**；插件加载主链路见 [`plugin.md`](plugin.md)，组件加载/持久化见 [`component.md`](component.md)，服务的定义/消费宏与 registry 见 [`service.md`](service.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [① 语言链接与符号导出](#-语言链接与符号导出mysql_plugin_export-的四分支)
  - [② 符号可见性与介入风险](#-符号可见性与介入风险)
  - [③ 异常的三道防线](#-异常的三道防线)
  - [④ dlopen](#-dlopen标志位与静态初始化)
  - [⑤ 类型规约](#-类型规约的源码证据为什么是-define-不是-enum)
  - [⑦ 服务 ABI 三件套](#-服务-abi-三件套const-表--noexcept--d-pointer)
  - [⑧ ABI 指纹门禁](#-abi-指纹门禁cmakedo_abi_checkcmake)
- [动手实证：编译器与链接器的真实行为](#动手实证编译器与链接器的真实行为)
- [★★ 真名勘误全表](#-真名勘误全表)
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

### 8. 案例：`psi_thread` 七代共存——"版本=改名"的完整账本

理论说到这里，最需要的是一个**真实的演进账本**。PSI 线程插桩服务（`include/mysql/components/services/psi_thread_service.h`）提供了教科书级的样本——源码头注释完整记录了每一代的引入/弃用/废弃时间与**具体改了什么**：

| 版本 | 引入 | 弃用 | 状态 | 相对上一代的改动 |
|---|---|---|---|---|
| v1 | 8.0.3 | 8.0.15 | **Removed** | — |
| v2 | 8.0.15 | 8.0.17 | **Removed** | — |
| v3 | 8.0.17 | 8.0.22 | **Removed** | — |
| **v4** | 8.0.22 | 8.0.25 | deprecated（**8.0.39 仍注册**） | — |
| **v5** | 8.0.25 | 8.0.28 | deprecated（**仍注册**） | `register_thread` 换成含 `m_os_name` 的 `PSI_thread_info_v5`；`spawn_thread` / `new_thread` 各多一个 sequence number 参数 |
| **v6** | 8.0.28 | 8.0.33 | deprecated（**仍注册**） | 新增 `set_thread_secondary_engine` |
| **v7** | 8.0.33 | — | **active** | 新增 `detect_telemetry` / `abort_telemetry` |

从这张表能读出四件事：

1. **触发换名的改动可以极小**：v5→v6 只是"加一个方法"，v6→v7 也只是加两个方法。放在虚类里这只是追加一个虚函数——但那会**错位所有既有槽位**，对已编译的消费者是灾难；换名则完全无副作用。
2. **改名版本化让"弃用"变成文档事件而非二进制事件**：v4/v5/v6 都已弃用多年，仍在 8.0.39 的 registry 里注册着，老组件照旧能 acquire 到。代价就是二进制里躺着多代函数表。
3. **真正被移除的（v1~v3）都是跨大版本后的清理**：说明"并存"有期限，但期限以年计。
4. **能力下沉的痕迹清晰可见**：v7 新增的 `detect_telemetry` / `abort_telemetry` 是 telemetry 能力进入 PSI 的入口——与 9.x 引入 Telemetry 的方向一致（**具体版本映射未核实**）。

对照插件侧的路线更能看出差异：插件的 `MYSQL_AUDIT_INTERFACE_VERSION` 之类是一个**数字**，靠"major 相等、minor 向后兼容 + sizeof 迁移"来兼容；组件服务则直接**换名字**。前者省名字、需要迁移逻辑；后者不需要迁移逻辑、靠名字空间换自由。两条路线服务于同一个目标，只是赌注下在不同地方（见设计思想 ⑤）。

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

**★ 完整原文**（⚠️ 上面是摘要，真身有 4 个 `#define` 且**不含 `dllimport`**）：

```c
/* include/mysql/plugin.h:42-72 —— 一字不漏 */
/*
  On Windows, exports from DLL need to be declared.
  Also, plugin needs to be declared as extern "C" because MSVC
  unlike other compilers, uses C++ mangling for variables not only
  for functions.
*/
#if defined(_MSC_VER)

#if defined(MYSQL_DYNAMIC_PLUGIN)
#ifdef __cplusplus
#define MYSQL_PLUGIN_EXPORT extern "C" __declspec(dllexport)   /* 分支 1 */
#else
#define MYSQL_PLUGIN_EXPORT __declspec(dllexport)              /* 分支 2 */
#endif
#else /* MYSQL_DYNAMIC_PLUGIN */
#ifdef __cplusplus
#define MYSQL_PLUGIN_EXPORT extern "C"                         /* 分支 3：只压 mangling，不导出 */
#else
#define MYSQL_PLUGIN_EXPORT                                    /* 分支 4a：空 */
#endif
#endif /*MYSQL_DYNAMIC_PLUGIN */

#else /*_MSC_VER */

#if defined(MYSQL_DYNAMIC_PLUGIN)
#define MYSQL_PLUGIN_EXPORT MY_ATTRIBUTE((visibility("default")))  /* 分支 4 */
#else
#define MYSQL_PLUGIN_EXPORT                                        /* 分支 4b：空 */
#endif

#endif /*_MSC_VER */
```

> ⚠️ **`dllimport` 不在这个宏里**——它在另一个宏 `MYSQL_PLUGIN_IMPORT`（`include/my_sharedlib.h:63-72`）：
> ```c
> /*
>   MYSQL_PLUGIN_IMPORT macro is used to export mysqld data
>   (i.e variables) for usage in storage engine loadable plugins.
>   Outside of Windows, it is dummy.
> */
> #if (defined(_WIN32) && defined(MYSQL_DYNAMIC_PLUGIN))
> #define MYSQL_PLUGIN_IMPORT __declspec(dllimport)
> #else
> #define MYSQL_PLUGIN_IMPORT
> #endif
> ```
> ★★ **区分关键**：`EXPORT` 是"我导出给插件用"，`IMPORT` 是"我要用 mysqld 里的**静态数据符号**"（变量而非函数），且**只在 Windows 有意义**。

**★ `visibility("default")` 的展开底座**（`include/my_compiler.h:97`）：

```c
#ifndef MY_ATTRIBUTE
#if defined(__GNUC__) || defined(__clang__)
#define MY_ATTRIBUTE(A) __attribute__(A)
#else
#define MY_ATTRIBUTE(A)
#endif
#endif
```

**★ `MYSQL_DYNAMIC_PLUGIN` 由 CMake 注入，且与 `MYSQL_SERVER` 互斥**：

```cmake
# cmake/plugin.cmake:150-151 —— 内置插件拿 MYSQL_SERVER
    SET_TARGET_PROPERTIES(${target}
      PROPERTIES COMPILE_DEFINITIONS "MYSQL_SERVER")

# cmake/plugin.cmake:200-209 —— 动态插件拿 MYSQL_DYNAMIC_PLUGIN
    IF(ARG_SERVER_AND_CLIENT)
      SET_TARGET_PROPERTIES (${target} PROPERTIES
        COMPILE_DEFINITIONS "MYSQL_DYNAMIC_CLIENT_PLUGIN;MYSQL_DYNAMIC_PLUGIN")
    ELSEIF(ARG_CLIENT_ONLY)
      SET_TARGET_PROPERTIES (${target} PROPERTIES
        COMPILE_DEFINITIONS "MYSQL_DYNAMIC_CLIENT_PLUGIN")
    ELSE()
      SET_TARGET_PROPERTIES (${target} PROPERTIES
        COMPILE_DEFINITIONS "MYSQL_DYNAMIC_PLUGIN")
    ENDIF()
    SET_TARGET_PROPERTIES (${target} PROPERTIES PREFIX "")   /* ★ 去掉 lib 前缀 */
```

> ⚠️ **不存在 `MYSQL_LEGACY_PLUGIN` / `MYSQL_DYNAMIC_COMPONENT`**（全仓 0 命中）。同类宏实际只有四个：`MYSQL_DYNAMIC_PLUGIN`、`MYSQL_DYNAMIC_CLIENT_PLUGIN`、`MYSQL_COMPONENT`、`MYSQL_ABI_CHECK`。

**★ 这个宏决定的是"导出哪三个符号"**（`include/mysql/plugin.h:144-166`）：

```c
#ifndef MYSQL_DYNAMIC_PLUGIN
#define __MYSQL_DECLARE_PLUGIN(NAME, VERSION, PSIZE, DECLS)         \
  MYSQL_PLUGIN_EXPORT int VERSION = MYSQL_PLUGIN_INTERFACE_VERSION; \
  MYSQL_PLUGIN_EXPORT int PSIZE = sizeof(struct st_mysql_plugin);   \
  MYSQL_PLUGIN_EXPORT struct st_mysql_plugin DECLS[] = {
#else
#define __MYSQL_DECLARE_PLUGIN(NAME, VERSION, PSIZE, DECLS)  \
  MYSQL_PLUGIN_EXPORT int _mysql_plugin_interface_version_ = \
      MYSQL_PLUGIN_INTERFACE_VERSION;                        \
  MYSQL_PLUGIN_EXPORT int _mysql_sizeof_struct_st_plugin_ =  \
      sizeof(struct st_mysql_plugin);                        \
  MYSQL_PLUGIN_EXPORT struct st_mysql_plugin _mysql_plugin_declarations_[] = {
#endif

// name的作用就是用于生成其他三个参数名字
#define mysql_declare_plugin(NAME)                                        \
  __MYSQL_DECLARE_PLUGIN(NAME, builtin_##NAME##_plugin_interface_version, \
                         builtin_##NAME##_sizeof_struct_st_plugin,        \
                         builtin_##NAME##_plugin)

#define mysql_declare_plugin_end                 \
  , { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 } \
  }
```

> ★★ **同一份 `mysql_declare_plugin()` 源码，靠 `MYSQL_DYNAMIC_PLUGIN` 这一个宏生成两套符号名**：内置时是 `builtin_<NAME>_plugin`（被 `sql_builtin.cc.in` 的 `mysql_mandatory_plugins[]` 数组直接引用），动态时是固定的 `_mysql_plugin_declarations_`（供 `dlsym` 查找）。这就是"builtin/dynamic 同构"的物理实现。
> ★ `mysql_declare_plugin_end` 补的那个全 0 元素就是**数组终止哨兵**（`plugin->info == nullptr` 结束遍历）。

**★ 真实插件的用法**（⚠️ `plugin/` 目录下**没有**直接书写 `MYSQL_PLUGIN_EXPORT` 的——58 个插件全部通过 `mysql_declare_plugin()` 间接使用）：

```cpp
/* plugin/daemon_example/daemon_example.cc:199 */
mysql_declare_plugin(daemon_example){
    MYSQL_DAEMON_PLUGIN,
    &daemon_example_plugin,
    "daemon_example",
    PLUGIN_AUTHOR_ORACLE,
    "Daemon example, creates a heartbeat beat file in mysql-heartbeat.log",
    PLUGIN_LICENSE_GPL,
    daemon_example_plugin_init,   /* Plugin Init */
    nullptr,                      /* Plugin Check uninstall */
    daemon_example_plugin_deinit, /* Plugin Deinit */
    0x0100 /* 1.0 */,
    nullptr, /* status variables */
    nullptr, /* system variables */
    nullptr, /* config options   */
    0,       /* flags            */
} mysql_declare_plugin_end;
```

```cpp
/* plugin/x/src/xpl_plugin.cc:35 —— 一个 .so 里声明两个插件 */
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

### ② 符号可见性与介入风险

组件的 `-fvisibility=hidden`（`component.cmake`，见上）与插件的 `VISIBILITY_HIDDEN` 选项（`plugin.cmake`）解决三个问题：

1. **减少 PLT/GOT 重定位**：隐藏符号走内部调用，不经过动态链接器；
2. **防止符号介入（interposition）**：两个 DSO 定义同名全局函数时，GNU 链接规则按"先到先得"全局共享一个——插件里恰好有个函数叫 `my_hash` 就会静默替换（或被替换）服务器版本；hidden 把内部符号全部私有化；
3. **导出面最小化**：`dlsym` 只能摸到 3 个插件符号 / 1 个 `list_components`，其他内部实现不可被外部探知或依赖。

与服务器侧 `--export-dynamic` 形成对照：服务器全量导出（插件依赖它解析符号），插件/组件最小导出（服务器只依赖固定入口符号）。**不对称是刻意的**——插件对服务器是"开放依赖"，服务器对插件是"封闭入口"。

**★★ 隐藏符号必须"显式要求"，不是默认行为**：

```cmake
# ① 插件：必须传 VISIBILITY_HIDDEN 关键字
# cmake/plugin.cmake:38 —— 关键字声明（带 TODO 说想变成默认）
    VISIBILITY_HIDDEN # Add -fvisibility=hidden on UNIX
                      # TODO(tdidriks) make this default if MODULE_ONLY

# cmake/plugin.cmake:317-323 —— 落地
  IF(BUILD_PLUGIN AND ARG_VISIBILITY_HIDDEN AND UNIX)
    TARGET_COMPILE_OPTIONS(${target} PRIVATE "-fvisibility=hidden")
  ENDIF()
```

```cmake
# ② 组件：无条件加，但 clang + UBSan 例外跳过
# cmake/component.cmake:106-118
    # To hide the component symbols in the shared object
    IF(UNIX)
      IF(MY_COMPILER_IS_CLANG AND WITH_UBSAN)
        # nothing, clang/ubsan gets confused
        UNSET(COMPONENT_COMPILE_VISIBILITY CACHE)
      ELSE()
        # Use this also for component libraries and tests.
        SET(COMPONENT_COMPILE_VISIBILITY
          "-fvisibility=hidden" CACHE INTERNAL
          "Use -fvisibility=hidden for components" FORCE)
        TARGET_COMPILE_OPTIONS(${target} PRIVATE "-fvisibility=hidden")
      ENDIF()
    ENDIF()
```

> ★★ **"clang/ubsan gets confused" 这个注释很值得注意**：消毒器（UBSan）需要在 DSO 边界外看到符号来做类型检查/虚函数表校验，隐藏符号会让它的运行时库"confused"。**又一次"检测工具改变 ABI 行为"的取舍**（与 `RTLD_NODELETE` 同类）。

**★ 组件还有一个插件没有的硬门禁：禁止 `MYSQL_SERVER`**：

```cmake
# cmake/component.cmake:53-58
  GET_PROPERTY(CWD_DEFINITIONS DIRECTORY PROPERTY COMPILE_DEFINITIONS)
  LIST(FIND CWD_DEFINITIONS "MYSQL_SERVER" FOUND_DEFINITION)
  IF(NOT FOUND_DEFINITION EQUAL -1)
    MESSAGE(FATAL_ERROR
      "component ${component} has -DMYSQL_SERVER")
  ENDIF()
```

> ★★ 查的是**目录属性**（`DIRECTORY PROPERTY COMPILE_DEFINITIONS`）不是 target 属性——语义是"**整个目录**不许带 `-DMYSQL_SERVER`"。而插件恰恰靠 `MYSQL_SERVER` 区分内置形态（`plugin.cmake:150`）。**两个宏把两套机制彻底隔开**。
> ★ 组件侧无条件 `TARGET_COMPILE_DEFINITIONS(${target} PUBLIC MYSQL_COMPONENT)`（`component.cmake:79`）。

**★ 服务器侧的反向导出**（`ENABLE_EXPORTS`）：

```cmake
# sql/CMakeLists.txt:1267
SET_TARGET_PROPERTIES(mysqld PROPERTIES ENABLE_EXPORTS TRUE)
TARGET_LINK_LIBRARIES(mysqld
  sql_main sql_gis binlog rpl rpl_source rpl_replica sql_dd mysys
  minchassis binlogevents_static ext::icu)
```

```cmake
# cmake/mysql_add_executable.cmake:74-82 —— 关键字文档（注释原文）
    ENABLE_EXPORTS     # For Linux, link with: -Wl,--export-dynamic -rdynamic
                       # This option is needed for some uses of "dlopen" or
                       # to allow obtaining backtraces from within a program.
                       # We disable it for non-Linux platforms.
                       # On WIN32 it would add /implib:.... to the linker
                       # command, which is probably not what you want
                       # (except for mysqld.lib which is used by plugins).

# cmake/mysql_add_executable.cmake:168-170 —— ★ 落地，仅 LINUX
  IF(LINUX AND ARG_ENABLE_EXPORTS)
    SET_TARGET_PROPERTIES(${target} PROPERTIES ENABLE_EXPORTS TRUE)
  ENDIF()
```

> ⚠️ **`-rdynamic` 只在 Linux 生效**，且注释明说 Win32 下会加 `/implib:`（只有 `mysqld.lib` 需要它，给插件用）。

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

**★ 完整清单 + 同类证据**（⚠️ **没有一处用 `enum`**）：

```c
/* include/mysql/plugin.h:74-102 */
#ifdef __cplusplus
class THD;
class Item;
#define MYSQL_THD THD *        /* ★ C++ 侧是真类型 */
#else
#define MYSQL_THD void *       /* ★★ C 侧退化成 void* —— 不把 C++ 类型的 ABI 泄漏给插件 */
#endif

typedef void *MYSQL_PLUGIN;

#ifndef MYSQL_ABI_CHECK
#include <mysql/services.h>
#endif

#define MYSQL_XIDDATASIZE 128

struct MYSQL_XID {
  long formatID;
  long gtrid_length;
  long bqual_length;
  char data[MYSQL_XIDDATASIZE]; /* Not \0-terminated */
};
```

```c
/* include/mysql/plugin.h:104-136 —— 全部 #define */
#define MYSQL_PLUGIN_INTERFACE_VERSION 0x010B    /* = 267；major=(v>>8)=0x01，minor=(v&0xFF)=0x0B */

#define MYSQL_UDF_PLUGIN 0                /* User-defined function        */
#define MYSQL_STORAGE_ENGINE_PLUGIN 1     /* Storage Engine               */
#define MYSQL_FTPARSER_PLUGIN 2           /* Full-text parser plugin      */
#define MYSQL_DAEMON_PLUGIN 3             /* The daemon/raw plugin type */
#define MYSQL_INFORMATION_SCHEMA_PLUGIN 4 /* The I_S plugin type */
#define MYSQL_AUDIT_PLUGIN 5              /* The Audit plugin type        */
#define MYSQL_REPLICATION_PLUGIN 6        /* The replication plugin type */
#define MYSQL_AUTHENTICATION_PLUGIN 7     /* The authentication plugin type */
#define MYSQL_VALIDATE_PASSWORD_PLUGIN 8  /* validate password plugin type */
#define MYSQL_GROUP_REPLICATION_PLUGIN 9  /* The Group Replication plugin */
#define MYSQL_KEYRING_PLUGIN 10           /* The Keyring plugin type   */
#define MYSQL_CLONE_PLUGIN 11             /* The Clone plugin type   */
#define MYSQL_MAX_PLUGIN_TYPE_NUM 12      /* The number of plugin types   */

/* We use the following strings to define licenses for plugins */
#define PLUGIN_LICENSE_PROPRIETARY 0
#define PLUGIN_LICENSE_GPL 1
#define PLUGIN_LICENSE_BSD 2

#define PLUGIN_LICENSE_PROPRIETARY_STRING "PROPRIETARY"
#define PLUGIN_LICENSE_GPL_STRING "GPL"
#define PLUGIN_LICENSE_BSD_STRING "BSD"

#define PLUGIN_AUTHOR_ORACLE "Oracle Corporation"
```

```c
/* include/mysql/plugin.h:168-187 —— flags 也是 #define（unsigned long 位） */
#define PLUGIN_OPT_NO_INSTALL 1UL   /* Not dynamically loadable */
#define PLUGIN_OPT_NO_UNINSTALL 2UL /* Not dynamically unloadable */
#define PLUGIN_OPT_ALLOW_EARLY 4UL  /* allow --early-plugin-load */
#define PLUGIN_OPT_DEFAULT_OFF 8UL  /* Turned off by default */
/*
  All "extra" plugins declared together, same mysql_declare_plugin statement,
  depends on the first "main" plugin. ... Use it when it does not make sense
  to have the extra plugins on when the main plugin is off.
 */
#define PLUGIN_OPT_DEPENDENT_EXTRA_PLUGINS 16UL
```

```c
/* include/mysql/plugin.h:653-679 —— 各类插件自己的接口版本 */
#define MYSQL_FTPARSER_INTERFACE_VERSION 0x0101
#define MYSQL_REWRITE_PRE_PARSE_INTERFACE_VERSION 0x0010
#define MYSQL_REWRITE_POST_PARSE_INTERFACE_VERSION 0x0010

/* handlertons of different MySQL releases are incompatible */
#define MYSQL_DAEMON_INTERFACE_VERSION (MYSQL_VERSION_ID << 8)
/* ★★ daemon 插件的接口版本直接绑死服务器版本号 —— 跨版本必不兼容，故意的 */

struct st_mysql_daemon {
  int interface_version;
};
```

> ★★ **`MYSQL_THD` 的双分支是最能说明问题的例子**：同一个标识符在 C++ 里是 `THD*`（服务器内部可得真类型），在 C 里是 `void*`（插件只当不透明句柄）。**一个 `#define` 同时满足"内部类型安全"与"外部 ABI 稳定"**。
> ★ 字符串类型也不用 STL，`MYSQL_LEX_STRING` 是纯 POD：
> ```c
> struct MYSQL_LEX_STRING { char *str; size_t length; };
> struct MYSQL_LEX_CSTRING { const char *str; size_t length; };
> ```

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

**★ `DEFINE_SERVICE_HANDLE` 完整定义**（★ 注释原文直接回答了"为什么用 struct 不用 class/虚函数"）：

```c
/* include/mysql/components/service.h:115-129 */
/**
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

  @param name Handle name, must be a valid C name.
*/
#define DEFINE_SERVICE_HANDLE(name) typedef struct name##_imp *name
```

> ★★ 三句话就是答案：① **不要用任何 virtual 方法**（vtable 布局是 ABI 的不稳定点）；② **只保留一个成员**（d-pointer）；③ **头部发布后不再修改**。
> ★ 真实用例：`DEFINE_SERVICE_HANDLE(my_h_service)`、`my_h_service_iterator`、`my_h_component_iterator`、`MYSQL_H`、`MYSQL_RES_H`、`MYSQL_FIELD_H`；还有 `#define MYSQL_ROW_H char **`。

**★★ "版本 = 改名"的权威出处**（`include/mysql/components/service.h:36` 的 `@page` 注释）：

```c
/**
  @page PAGE_COMPONENTS_SERVICE A Service and a Service Implementation
  ...
  Each Service should work only on basic C types and structures
  of these types to prevent problems with not fully-defined C++ objects ABI. The
  Service is a default way to operate inside the Components subsystem as a mean
  to show that one is interested only on functionality interface, not its exact
  implementation. The Services are not versioned - any change to interface must
  require Service being copied to one with different name before applying
  changes. The Services by itself do not carry any state, all methods are
  stateless. ...
*/
```

> ★★ **"The Services are not versioned - any change to interface must require Service being copied to one with different name before applying changes."** —— 这就是"版本 = 改名"的原文。

**★ 改名机制的宏实现**：

```c
/* include/mysql/components/service.h:62-93 */
#define SERVICE_TYPE_NO_CONST(name) mysql_service_##name##_t
#define SERVICE_TYPE(name) const SERVICE_TYPE_NO_CONST(name)   /* ★ const 表 */

#define BEGIN_SERVICE_DEFINITION(name) typedef struct s_mysql_##name {
/* ★★ 名字里直接带版本：psi_thread_v7 → struct s_mysql_psi_thread_v7 */

#define END_SERVICE_DEFINITION(name) \
  }                                  \
  SERVICE_TYPE_NO_CONST(name);       /* → mysql_service_psi_thread_v7_t; */

#define DECLARE_METHOD(retval, name, args) retval(*name) args
#define DECLARE_BOOL_METHOD(name, args) \
  DECLARE_METHOD(mysql_service_status_t, name, args)
```

**★★★ `psi_thread` 七代共存完整账本**（★ 这一份是"版本=改名"最完整的实证）：

```c
/* include/mysql/components/services/psi_thread_service.h:31-51
   —— v1/v2/v3 只剩"墓碑注释"，结构体已被删除 */
/*
  Version 1.
  Introduced in MySQL 8.0.3
  Deprecated in MySQL 8.0.15
  Abandoned in MySQL 8.0.17
  Status: Removed.
*/
/*
  Version 2.
  Introduced in MySQL 8.0.15
  Abandoned in MySQL 8.0.17
  Status: Removed.
*/
/*
  Version 3.
  Introduced in MySQL 8.0.17
  Abandoned in MySQL 8.0.22
  Status: Removed.
*/
/*
  Version 4.
  Introduced in MySQL 8.0.22
  Deprecated in MySQL 8.0.25
  Status: deprecated, use version 5 instead.
*/
```

| 代 | 版本注释（Introduced / Deprecated / Status） | 结构体 | 与上一代的差异（原文） |
|---|---|---|---|
| v4 | 8.0.22 / 8.0.25 / deprecated | `s_mysql_psi_thread_v4`（:60-125） | — |
| v5 | 8.0.25 / 8.0.28 / deprecated | `:139-204` | *"register_thread takes an expanded PSI_thread_info_v5 instrumentation, which includes a new m_os_name attribute. spawn_thread takes an extra sequence number parameter. new_thread takes an extra sequence number parameter"* |
| v6 | 8.0.28 / 8.0.33 / deprecated | `:215-282` | *"added set_thread_secondary_engine"* —— **插入在 `set_thread_info` 之后、`set_thread` 之前（位置也是 ABI 的一部分）** |
| **v7** | **8.0.33 / — / active** | `:292-362` | *"added detect_telemetry, abort_telemetry"* —— **追加在最末尾** |

```c
/* v7 的末尾（:357-362）—— ★★ 只追加不重排，这是 ABI 兼容的铁律 */
  notify_session_change_user_v1_t notify_session_change_user;

thread_detect_telemetry_v7_t detect_telemetry;
thread_abort_telemetry_v7_t abort_telemetry;
END_SERVICE_DEFINITION(psi_thread_v7)
```

**★ 组件默认消费的是 v6，不是最新的 v7**：

```c
/* include/mysql/components/services/psi_thread.h:30-36 */
#define REQUIRES_PSI_THREAD_SERVICE REQUIRES_SERVICE(psi_thread_v6)
#define REQUIRES_PSI_THREAD_SERVICE_PLACEHOLDER \
  REQUIRES_SERVICE_PLACEHOLDER(psi_thread_v6)

extern REQUIRES_PSI_THREAD_SERVICE_PLACEHOLDER;

#define PSI_THREAD_CALL(M) mysql_service_psi_thread_v6->M
```

> ★★ 这解释了为什么"服务改名"不会引发连锁升级：每个组件在**编译期**用 `REQUIRES_SERVICE(psi_thread_v6)` 固定自己要哪一代，registry 里同时挂着 v4/v5/v6/v7 四个实现，互不干扰。

**★ PSI 插件 ABI 侧另有运行时协商**（与组件侧"改名"并存的第二套机制）：

```c
/* include/mysql/psi/psi_thread.h:45-99 */
#define PSI_THREAD_VERSION_1 1   /* This version is obsolete. */
#define PSI_THREAD_VERSION_2 2   /* This version is obsolete. */
#define PSI_THREAD_VERSION_3 3   /* This version is obsolete. */
#define PSI_THREAD_VERSION_4 4   /* This version is supported. */
#define PSI_THREAD_VERSION_5 5   /* This version is supported. */
#define PSI_THREAD_VERSION_6 6   /* This version is supported. */
#define PSI_THREAD_VERSION_7 7   /* This version is supported. */
#define PSI_CURRENT_THREAD_VERSION 7
```

```c
/* include/mysql/psi/psi_thread.h:101-115 */
/** Entry point for the performance schema interface. */
struct PSI_thread_bootstrap {
  /**
    ABI interface finder.
    Calling this method with an interface version number returns either
    an instance of the ABI for this version, or NULL.
    @sa PSI_THREAD_VERSION_1 ... @sa PSI_CURRENT_THREAD_VERSION
  */
  void *(*get_interface)(int version);   /* ★★ "版本号 → 结构体指针"的运行时协商入口 */
};
```

> ★★ **两套版本机制并存**：组件服务用**改名**（编译期绑定，靠 `REQUIRES_SERVICE` 选代）；PSI 插件 ABI 用**运行时协商**（`get_interface(int version)` 按号拿结构体）。同一个 `psi_thread` 功能，两条 ABI 路径。

**★ 服务实现的"表本体"**（★ 顺序必须与 `BEGIN_SERVICE_DEFINITION` 的声明顺序逐一对应）：

```cpp
/* sql/server_component/mysql_mutex_service.cc:30-68 */
extern SERVICE_TYPE(mysql_mutex_v1)
    SERVICE_IMPLEMENTATION(mysql_server, mysql_mutex_v1);
/* ★ 展开：extern const mysql_service_mysql_mutex_v1_t imp_mysql_server_mysql_mutex_v1; */

SERVICE_TYPE(mysql_mutex_v1)
SERVICE_IMPLEMENTATION(mysql_server, mysql_mutex_v1) = {
    impl_mysql_mutex_register, impl_mysql_mutex_init,
    impl_mysql_mutex_destroy,  impl_mysql_mutex_lock,
    impl_mysql_mutex_trylock,  impl_mysql_mutex_unlock};
```

```c
/* include/mysql/components/service_implementation.h:46-67 —— 宏定义 */
#define SERVICE_IMPLEMENTATION(component, service) imp_##component##_##service
#define BEGIN_SERVICE_IMPLEMENTATION(component, service) \
  SERVICE_TYPE(service) SERVICE_IMPLEMENTATION(component, service) = {
#define END_SERVICE_IMPLEMENTATION() }
```

### ⑧ ABI 指纹门禁（`cmake/do_abi_check.cmake`）

流程（文件头注释自述）：`gcc -E -nostdinc -dI -DMYSQL_ABI_CHECK -I<include>` 对每个 ABI 头文件生成预处理输出 → `sed` 去 `#` 行/空行/OS 杂讯 → diff 仓库金丝雀 `include/mysql/plugin.pp` → 失败留下 `abi_check.out`，开发者**有意变更时** `mv abi_check.out include/mysql/plugin.pp` 签认新基线。

定位辨析：这是**文本指纹**检查，不是符号级检查（`abidiff`/`abi-compliance-checker` 比较符号表与布局，是另一类工具）。它的能力边界：

- **能发现**：布局改动、类型改名、宏改值、新增/删除字段——任何落到预处理文本里的差异；
- **发现不了**：语义变化（同签名不同行为）。这与运行时校验是同构的盲区——**版本号/指纹都是"人为声明"的兼容性承诺，不是从行为自动推导的**。工程上两条腿都要：门禁防"无意改动"，版本号+迁移管"有意演化的兼容"。

**★ `cmake/abi_check.cmake` 全文**（★ 门禁的准入条件与被检查的头文件清单）：

```cmake
#
# Headers which need to be checked for abi/api compatibility are in
# API_PREPROCESSOR_HEADER. plugin.h is tested implicitly via
# plugin_audit.h and plugin_ftparser.h.
#
# We use gcc specific preprocessing command and sed/diff, so it will
# only be run on Unix and only if gcc is used. On some Unixes,
# (Solaris) sed or diff might act differently from GNU, so we run only
# on systems we can trust.
# On Windows we use the Windows subsystem for Linux and gcc and sed
# installed in it, if available.
SET(RUN_ABI_CHECK 0)
IF(LINUX AND MY_COMPILER_IS_GNU)
  SET(RUN_ABI_CHECK 1)
ELSEIF(WIN32)
  FIND_PROGRAM(WSL_EXECUTABLE wsl HINTS C:/Windows/Sysnative)
  IF (WSL_EXECUTABLE)
    SET (COMPILER "gcc")
    # Check that the compiler is available under WSL.
    EXECUTE_PROCESS(
      COMMAND ${WSL_EXECUTABLE} which ${COMPILER}
      OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE COMPILER_LOCATION)
    IF (COMPILER_LOCATION STREQUAL "/usr/bin/gcc")
      SET(RUN_ABI_CHECK 1)
      MESSAGE(STATUS "Will do ABI check using ${WSL_EXECUTABLE} (gcc/sed)")
    ENDIF()
  ENDIF()
ENDIF()

SET(API_PREPROCESSOR_HEADER
  ${CMAKE_SOURCE_DIR}/include/mysql/plugin_audit.h
  ${CMAKE_SOURCE_DIR}/include/mysql/plugin_ftparser.h
  ${CMAKE_SOURCE_DIR}/include/mysql.h
  ${CMAKE_SOURCE_DIR}/include/mysql/client_plugin.h
  ${CMAKE_SOURCE_DIR}/include/mysql/plugin_auth.h
  ${CMAKE_SOURCE_DIR}/include/mysql/plugin_keyring.h
)
IF(NOT WITHOUT_SERVER)
  LIST(APPEND API_PREPROCESSOR_HEADER
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_thread_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_mutex_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_rwlock_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_cond_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_file_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_socket_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_table_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_mdl_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_idle_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_stage_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_statement_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_transaction_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_memory_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_error_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/psi/psi_abi_system_v1.h
    ${CMAKE_SOURCE_DIR}/include/mysql/services.h
  )
ENDIF()
...
IF(RUN_ABI_CHECK)
  UNSET(API_PP_FILES)
  FOREACH(file ${API_PREPROCESSOR_HEADER})
    LIST(APPEND API_PP_FILES ${file}.pp)
  ENDFOREACH()

  ADD_CUSTOM_COMMAND(
    OUTPUT ${CMAKE_BINARY_DIR}/abi_check.out
    COMMAND ${CMAKE_COMMAND}
    -DCOMPILER=${COMPILER}
    -DSOURCE_DIR=${CMAKE_SOURCE_DIR}
    -DBINARY_DIR=${CMAKE_BINARY_DIR}
    -DWSL_EXECUTABLE=${WSL_EXECUTABLE}
    "-DABI_HEADERS=${API_PREPROCESSOR_HEADER}"
    -P ${CMAKE_SOURCE_DIR}/cmake/do_abi_check.cmake
    DEPENDS ${API_PREPROCESSOR_HEADER} ${API_PP_FILES}
    VERBATIM
  )

  ADD_CUSTOM_TARGET(abi_check ALL
    DEPENDS ${CMAKE_BINARY_DIR}/abi_check.out
    )
ENDIF()
```

> ★★ **`abi_check` 挂在 `ALL` 上**——每次构建都会跑，不是可选检查。
> ★ 只在 **Linux + GCC**（或 Windows + WSL 里有 `/usr/bin/gcc`）才启用。
> ⚠️ **`plugin.h` 不在清单里**——它"隐式"通过 `plugin_audit.h` / `plugin_ftparser.h` 被 include 进来（文件头注释明说）。

**★ `do_abi_check.cmake` 的算法说明（文件头注释原文）**：

```cmake
#
# Rules for checking that the abi/api has not changed.
#
# 1) Generate preprocessor output for the files that need to
#    be tested for abi/api changes. use -nostdinc to prevent
#    generation of preprocessor output for system headers. This
#    results in messages in stderr saying that these headers
#    were not found. Redirect the stderr output to /dev/null
#    to prevent seeing these messages.
# 2) sed the output to
#    2.1) remove blank lines and lines that begin with "# "
#    2.2) When gcc -E is run on the Mac OS  and solaris sparc platforms it
#         introduces a line of output that shows up as a difference between
#         the .pp and .out files. Remove these OS specific preprocessor text
#         inserted by the preprocessor.
# 3) diff the generated file and the canons (.pp files already in
#    the repository).
# 4) delete the .out file that is generated.
#
# If the diff fails, the generated file is not removed. This will
# be useful for analysis of ABI differences (e.g. using a visual
# diff tool).
#
# A ABI change that causes a build to fail will always be accompanied
# by new canons (.out files). The .out files that are not removed will
# be replaced as the new .pp files.
#
# e.g. If include/mysql/plugin.h has an ABI change then this rule would
# leave a <build directory>/abi_check.out file.
#
# A developer with a justified API change will then do a
# mv <build directory>/abi_check.out include/mysql/plugin.pp
# to replace the old canons with the new ones.
```

**★ 实际执行段**（★ 指纹生成 + sed 清洗 + diff）：

```cmake
  EXECUTE_PROCESS(
    COMMAND ${WSL_EXECUTABLE} ${COMPILER}
      -E -nostdinc -dI -DMYSQL_ABI_CHECK -I${ABI_SOURCE_DIR}/include
      -I${ABI_BINARY_DIR}/include -I${ABI_SOURCE_DIR}/include/mysql
      -I${ABI_SOURCE_DIR}/sql -I${ABI_SOURCE_DIR}/libbinlogevents/export
      ${abi_file}
      ERROR_QUIET OUTPUT_FILE ${tmpfile})
  /* ★★ 指纹生成：gcc -E 纯预处理；-nostdinc 排除系统头；-DMYSQL_ABI_CHECK 打开 ABI 专用分支 */

  EXECUTE_PROCESS(
    COMMAND ${WSL_EXECUTABLE} sed -e "/^# /d"
                -e "/^[	]*$/d"
                -e "/^#pragma GCC set_debug_pwd/d"
                -e "/^#ident/d"
    RESULT_VARIABLE result OUTPUT_FILE ${abi_check_out} INPUT_FILE ${tmpfile})
  /* ★ sed 清洗：去掉 '#' 行、空行、set_debug_pwd、#ident（OS/编译器杂讯） */

  EXECUTE_PROCESS(
    COMMAND ${WSL_EXECUTABLE} diff -w ${abi_original}.pp ${abi_current}
    RESULT_VARIABLE result)
  /* ★ 比对金样：<头文件>.pp vs 刚生成的 abi_check.out；-w 忽略空白 */
  IF(NOT ${result} EQUAL 0)
    MESSAGE(FATAL_ERROR
      "ABI check found difference between ${file}.pp and ${abi_check_out}")
  ENDIF()
```

**★ `-DMYSQL_ABI_CHECK` 在头文件里对应的开关**：

```c
/* include/mysql/plugin.h:84-86 */
#ifndef MYSQL_ABI_CHECK
#include <mysql/services.h>
#endif
```

> ★★ 这就是"指纹稳定"的关键：正常编译时 `#include <mysql/services.h>` 会把一堆**平台相关**的内容拉进来；ABI 检查时定义 `MYSQL_ABI_CHECK` 把它排除掉，指纹才不含平台杂讯。

---

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

### 实验 7：模板与 inline 是弱符号——`my_service<T>` 为什么可以安全地用 C++

头文件（`tpl.h`）里放一个模板和一个 inline 函数，让**两个独立的 DSO** 各自用它，再看符号表：

```cpp
// tpl.h
template <typename T> T twice(T v) { return v + v; }
inline int inl(int x) { return x * 2; }
// a.cc:  int use_a() { return twice<int>(3) + inl(4); }
// b.cc:  int use_b() { return twice<int>(7); }
```

```
--- libt.so ---
0000000000003f54 (__TEXT,__text) weak external __Z5twiceIiET_S0_
0000000000003f70 (__TEXT,__text) weak external __Z3inli
--- libt2.so ---
0000000000003f78 (__TEXT,__text) weak external __Z5twiceIiET_S0_
```

三个可直接读出的结论：

1. **模板实例化是弱符号**（`weak external`，ELF 上是 `W`，COMDAT 段），不是强符号——它不会被"导出给对方用"，而是**每个用到它的 DSO 各自生成一份**（两处地址完全不同：0x3f54 vs 0x3f78）。
2. **`inline` 函数同理**（`__Z3inli` 也是 weak），这就是 ODR（One Definition Rule）在链接层的兑现：同名 weak 定义多处存在，链接器任选其一，前提是它们"等价"。
3. 正因如此，**头文件里的 C++ 模板是安全的**：`my_service<TService>`（`include/mysql/components/my_service.h`）是模板、带 RAII、用 `reinterpret_cast` 取服务表——这些 C++ 特性全都**在消费侧实例化**，没有任何对象布局跨边界传递。

把这三条和实验 1 放在一起，就得到整个 ABI 生态最精炼的一条规则：

> **C++ 的"语法"可以用在边界上（模板、RAII、重载解析），但 C++ 的"类型布局"绝不能送过边界。**
> 模板在消费侧实例化 → 两端各自的副本，互不影响；`std::string`/`std::vector` 进了签名 → 符号名里刻着标准库身份（实验 1）、布局依赖库实现版本，两端必须同源。

对应的风险也一目了然：**ODR 违规**——若两个 DSO 里的同名 inline 函数实现不同，链接器任选其一，行为不确定且无警告。MySQL 头文件里的模板/inline 都是短小、无状态、随服务器版本整体升级的，把这份风险压到了可接受范围。

### 实验小结：理论与源码的对应关系

| 实验 | 证明的理论点 | MySQL 源码落点 |
|---|---|---|
| 1 mangling | C++ 符号名嵌入实现身份（libc++ `__1` / libstdc++ `__cxx11`），API 兼容 ≠ ABI 兼容 | 契约面禁 STL；`mysql_string` 服务家族 |
| 2 visibility | 导出面可控 | `-fvisibility=hidden` + 三符号点名放行 |
| 3 noexcept 违反 | 异常越界 = `std::terminate` | `DEFINE_METHOD noexcept` + rethrow 惯用法就地消化 |
| 4 vtable | 槽位布局编译器私有 | 契约面零虚类；服务表=手工布局的函数指针表 |
| 5 RTLD_LOCAL | 符号隔离是链接器行为 | 插件/组件互相不可见符号，只共享 mysqld 导出符号 |
| 6 static init | `dlopen` 即执行全局构造 | 初始化靠显式回调，不靠静态初始化 |
| 7 模板/inline 是弱符号 | 模板在消费侧实例化，无布局跨边界 | `my_service<T>` 可安全用 C++；服务签名禁 STL |

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

**★ 服务器侧还有两个"异常转错误码"的函数**（⚠️ 与组件侧的 `mysql_components_handle_std_exception` 是**三份独立实现**）：

```cpp
/* sql/sql_exception_handler.cc:60-90 —— 服务器侧 */
void handle_std_exception(const char *funcname) {
  try {
    throw;
  } catch (const std::bad_alloc &e) {
    my_error(ER_STD_BAD_ALLOC_ERROR, MYF(0), e.what(), funcname);
  } catch (const std::domain_error &e) { ... }
  catch (const std::length_error &e) { ... }
  catch (const std::invalid_argument &e) { ... }
  catch (const std::out_of_range &e) { ... }
  catch (const std::overflow_error &e) { ... }
  catch (const std::range_error &e) { ... }
  catch (const std::underflow_error &e) { ... }
  catch (const std::logic_error &e) { ... }
  catch (const std::regex_error &e) {          /* ★★ 比组件版多一个 regex_error 分支 */
    my_error(ER_STD_REGEX_ERROR, MYF(0), e.what(), funcname);
  } catch (const std::runtime_error &e) { ... }
  catch (const std::exception &e) { ... }
  catch (...) {
    my_error(ER_UNKNOWN_ERROR, MYF(0));
  }
}
```

```cpp
/* sql/sql_exception_handler.cc:92-143 —— GIS 专项，最后兜底回落到 handle_std_exception */
void handle_gis_exception(const char *funcname) {
  try {
    throw;
  } catch (const gis::longitude_out_of_range_exception &e) { ... }
  catch (const gis::latitude_out_of_range_exception &e) { ... }
  catch (const gis::not_implemented_exception &e) { ... /* 按 SRS 类型分派 */ }
  catch (const gis::invalid_buffer_argument_exception &) { ... }
  ... /* invalid_geometry / too_large_polygon / boost::geometry::* 若干 */
  catch (const std::exception &) {
    handle_std_exception(funcname);   /* ★★ 兜底回落 */
  } catch (...) {
    my_error(ER_GIS_UNKNOWN_EXCEPTION, MYF(0), funcname);
  }
}
```

> ★ 三份的差别：组件版用 `mysql_error_service_printf`（走服务），服务器版用 `my_error`（直接）；服务器版多一个 `std::regex_error`；GIS 版处理 boost::geometry 的异常族。

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `MYSQL_PLUGIN_EXPORT` 定义在 `my_compiler.h` 或 components 目录 | **只在 `include/mysql/plugin.h:42-72`**。全仓 `include/` 下仅此一处命中 |
| 该宏有 `dllexport` / `dllimport` 四分支 | **没有 `dllimport`**。真四分支 = MSVC×(C++/C)×(有/无 `MYSQL_DYNAMIC_PLUGIN`)；`dllimport` 归 **`MYSQL_PLUGIN_IMPORT`**（`include/my_sharedlib.h:63-72`，**只在 Windows** 有意义，用于 mysqld 的**静态数据符号**） |
| `MYSQL_LEGACY_PLUGIN` / `MYSQL_DYNAMIC_COMPONENT` | **均不存在**（0 命中）。同类实际只有四个：`MYSQL_DYNAMIC_PLUGIN`、`MYSQL_DYNAMIC_CLIENT_PLUGIN`、`MYSQL_COMPONENT`、`MYSQL_ABI_CHECK` |
| 异常防线叫 `mysql_components_handle_std_exception_string()` | **不存在**。真名 `mysql_components_handle_std_exception(const char*)`（`components/libminchassis/component_common.cc:38`）；另有 `handle_std_exception()`（`sql/sql_exception_handler.cc:60`）与 `handle_gis_exception()`（同文件 `:92`）两份独立实现 |
| `dlopen` 用了 `RTLD_NOW \| RTLD_GLOBAL` | **只有 `RTLD_NOW`**（`sql/sql_plugin.cc:687`），即默认 `RTLD_LOCAL`。UDF 侧同样（`sql_udf.cc:311/705`） |
| `ENABLE_EXPORTS` 是手写 `-rdynamic` | `sql/CMakeLists.txt:1267` 是 `SET_TARGET_PROPERTIES(mysqld PROPERTIES ENABLE_EXPORTS TRUE)`；`-Wl,--export-dynamic -rdynamic` 的注释与落地在 `cmake/mysql_add_executable.cmake:76/168-170`，且**仅 LINUX** |
| 组件/插件默认就隐藏符号 | 必须**显式**：插件要传 `VISIBILITY_HIDDEN` 关键字（`plugin.cmake:38/317-323`，带 `TODO make this default if MODULE_ONLY`）；组件无条件加但 **clang+UBSan 例外跳过**（`component.cmake:106-118`，注释 "clang/ubsan gets confused"） |
| psi_thread 七代都在同一头文件里定义结构体 | **组件侧只保留 v4/v5/v6/v7** 四个 `BEGIN_SERVICE_DEFINITION`；**v1/v2/v3 只剩注释 "Status: Removed"**，结构体已删除。PSI 侧 `PSI_thread_service_v4..v7` 共存，`PSI_THREAD_VERSION_1..7` 常量全留 |
| ABI check 用 nm / abi-compliance-checker | **用 `gcc -E` 预处理 + `sed` + `diff -w`** 比对 `.pp` 金样（`do_abi_check.cmake:92-124`），且**只在 Linux+GCC**（或 Windows 走 WSL 的 `/usr/bin/gcc`） |
| 组件服务用虚函数/类 | 服务是 **const 函数指针表 struct**；句柄是**不透明指针** `typedef struct X_imp *X`；`DEFINE_SERVICE_HANDLE` 的注释明确要求「不要用虚函数、只保留一个成员（d-pointer）、发布后不再改头部」 |
| `plugin.h` 在 ABI 检查清单里 | **不在**——它"隐式"通过 `plugin_audit.h` / `plugin_ftparser.h` 被 include（文件头注释明说） |
| 服务器侧异常转换与组件侧是同一份 | **三份独立**：组件用 `mysql_error_service_printf`，服务器用 `my_error`，且服务器版多一个 `std::regex_error` 分支，GIS 版处理 boost::geometry 异常族 |

**★ 三个"结构性事实"**

1. **"内置/动态同构"靠一个宏实现**——`mysql_declare_plugin()` 在 `MYSQL_DYNAMIC_PLUGIN` 有无时展开成**两套符号名**：内置是 `builtin_<NAME>_plugin`（被 `sql_builtin.cc.in` 的数组直接引用），动态是固定的 `_mysql_plugin_declarations_`（供 `dlsym`）。所以同一份插件源码既能编成 `.so` 也能静态链进 mysqld，而服务器侧的代码完全不变。

2. **"版本 = 改名"是组件服务唯一的演进方式，且它之所以可行，是因为改名后旧代仍然存在**——`psi_thread_v4/v5/v6/v7` 四个实现同时挂在 registry 里，每个组件用 `REQUIRES_SERVICE(psi_thread_v6)` 在编译期固定自己要哪一代。对比之下，老插件 ABI 只能靠"版本号 + 结构迁移"（`plugin_dl_add` 里的 `memcpy` 搬运）。**这就是两代机制在"如何演进"上的根本分歧**：一个靠"命名空间唯一性"（改名即可，无需兼容代码），一个靠"版本号 + 迁移代码"（省了命名，但每代都要写兼容逻辑）。

3. **ABI 门禁是"文本指纹"而非"符号指纹"，它的盲区与版本号同构**——`gcc -E` + `sed` + `diff -w` 能抓到布局改动、类型改名、宏改值、字段增删（任何落到预处理文本里的差异），但**抓不到语义变化**（同签名不同行为）。这与运行时版本校验是同一种局限：**版本号/指纹都是"人为声明"的兼容性承诺，不是从行为自动推导的**。所以工程上必须两条腿：门禁防"无意改动"，版本号+迁移管"有意演化的兼容"。

> 注：`component.cmake` / `plugin.cmake` 的 CMake 片段、`do_abi_check.cmake` 的算法注释均为 Oracle 上游原文。

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
- 架构总览（双轨制与迁移进度）见 [`architecture.md`](architecture.md)
- 插件加载主链路见 [`plugin.md`](plugin.md)
- 组件加载/持久化见 [`component.md`](component.md)
- 服务定义/消费宏与 registry 见 [`service.md`](service.md)
