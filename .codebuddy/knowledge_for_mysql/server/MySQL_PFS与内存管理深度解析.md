# MySQL 内存管理与可观测性框架深度解析

## —— Performance Schema 与 InnoDB 分配器的理论渊源与工程实现

> 基于 MySQL 8.0 源码分析（`storage/innobase`, `storage/perfschema`, `mysys`, `include/mysql/psi` 等模块）
> 生成时间: 2026-07-18

---

## 摘要

MySQL 作为世界上使用最广泛的开源关系型数据库，其内部有两套基础设施支撑着生产环境的稳定运行：**Performance Schema (PFS)** 提供全量事件级（event-based tracing）可观测性，**分层内存分配器矩阵**（mem_heap、MEM_ROOT、buf_buddy、jemalloc）管理从临时查询结构到 Buffer Pool 页面的所有内存。本文从源码级深度（基于 MySQL 8.0）出发，追溯每项设计背后的学术理论与工程思想，涵盖 AOP（面向切面编程）、DTrace（零开销探针）、Event-based Tracing（全量事件追踪）、Buddy Allocator（伙伴分配器）、Arena/Bump Allocator（区域/指针碰撞分配器）、Segregated-fit Allocator（分级适配分配器）等理论在 MySQL 中的具体落地，并提供完备的学术论文引用和中英文术语对照。

---

## 第一章：Performance Schema —— MySQL 内建可观测性框架

### 1.1 概述

Performance Schema 是 MySQL 内建的可观测性与性能诊断框架。它通过在所有关键路径上**埋点（instrumentation）**，运行时以 **event（事件）** 为单位记录每一次被 instrument 操作的起止时间、线程 ID、源码位置等元数据，并提供 SQL 接口供用户查询聚合统计（summary）和事件明细（history）。

**核心特征：**
- **事件驱动（Event-based Tracing）**：每次被 instrument 的操作都记录为一个 event，不做采样（sampling）
- **零开销关闭**：不启用时，编译器通过条件编译完全剔除埋点代码（DTrace "zero-cost probes when disabled" 设计原则在用户态的实现）
- **三级漏斗存储**：`events_xxx_current → events_xxx_history → events_xxx_history_long`，基于 ring buffer，容量受控
- **多维度聚合**：按 `event_name`、`thread`、`user`、`host`、`account` 等维度做 summary 表

### 1.2 PFS 的插件架构

PFS 是作为 MySQL **内置存储引擎插件（built-in storage engine plugin）** 实现的。从 `ha_perfschema.cc:1564` 可以确认：

```cpp
mysql_declare_plugin(perfschema)
{
  MYSQL_STORAGE_ENGINE_PLUGIN,
  &pfs_storage_engine,
  "PERFORMANCE_SCHEMA",       // 引擎名
  PLUGIN_AUTHOR_ORACLE,
  "Performance Schema",
  PLUGIN_LICENSE_GPL,
  pfs_init_func,              // 初始化函数
  nullptr,                    // check_uninstall = nullptr → 不允许卸载
  pfs_done_func,              // 清理函数
  0x0001,
  ...
}
```

关键点：`check_uninstall` 为 `nullptr`，意味着 PFS 编译进 mysqld 后无法在运行时 `UNINSTALL PLUGIN`，只能通过启动参数 `performance_schema=OFF` 禁用。这是合理的——作为一个贯穿全引擎的 instrumentation 框架，允许动态卸载会导致所有埋点的函数指针悬空。

**插件化带来的工程优势：**

- PSI 接口（`psi_mutex.h`、`psi_memory.h` 等）通过函数指针表（vtable 模式）解耦。PFS 未编译或关闭时，`PSI_MUTEX_CALL(start_mutex_wait)` 等宏直接走空实现
- 插件接口支持 ABI 版本控制（如 `PSI_MUTEX_VERSION_1`），允许 PFS 独立演进而不破坏上层调用

```cpp
// include/mysql/psi/psi_mutex.h 中定义的接口函数指针表
typedef struct PSI_mutex_locker *(*start_mutex_wait_v1_t)(
    struct PSI_mutex_locker_state_v1 *state,
    struct PSI_mutex *mutex,
    enum PSI_mutex_operation op,
    const char *src_file, uint src_line);

typedef void (*end_mutex_wait_v1_t)(
    struct PSI_mutex_locker *locker, int rc);
```

#### 1.2.1 插件架构（Plugin Architecture）理论基础

PFS 的插件化设计不是凭空产生的，其背后是软件工程中 **插件架构（Plugin Architecture / Microkernel Pattern）** 的经典设计理论：

| 理论脉络 | 英文 | 会议/出处 | 年份 | 核心贡献 |
|---------|------|----------|------|---------|
| 模块化分解原则 | Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" | **CACM** | 1972 | **信息隐藏（Information Hiding）**原则：模块通过稳定接口暴露功能，内部实现可独立演化。PFS 的 PSI 接口正是这一原则的体现——上层只看到 `start_mutex_wait()` 的函数签名，不关心内部如何记录 |
| Microkernel / 微内核模式 | Buschmann et al., *"Pattern-Oriented Software Architecture (POSA)"* Vol.1 | Wiley | 1996 | 正式命名 **Microkernel** 模式：核心系统提供最小化功能，扩展以插件形式动态加载。MySQL Server 就是"核心"，PFS 就是"插件" |
| 基于组件的软件工程 | CBSE (Component-Based Software Engineering) | 工业惯例 | 1990s- | 通过明确接口契约（contract）和生命周期管理（init/deinit）实现松耦合 |

**PFS 与通用插件架构的差异：**

通用插件架构允许运行时动态加载和卸载（如 MySQL 的 `INSTALL PLUGIN` / `UNINSTALL PLUGIN`），但 PFS 作为**强制内置插件（Mandatory Built-in Plugin）**：
- `check_uninstall = nullptr`：禁止运行时卸载
- 只能通过启动参数 `performance_schema=OFF` 在进程层面整体禁用
- 原因：贯穿全引擎的 instrumentation 如果被卸载，所有埋点的函数指针将悬空，导致崩溃

**PFS 默认加载吗？是的。** 在标准编译中 PFS 作为内置存储引擎编译进 mysqld 二进制，启动时 `pfs_init_func()` 自动执行，默认可用。用户不需要执行 `INSTALL PLUGIN`。唯一的控制手段是启动参数 `performance_schema=ON|OFF`（默认 ON）。

### 1.3 理论基础之一：Event-based Tracing（全量事件追踪）

PFS 选择的是 **每次事件都记录** 的 tracing 路线，而非周期性中断的 sampling 路线。两种路线的对比深刻地影响了 PFS 的能力边界和开销模型。

| 维度 | Event-based Tracing（PFS） | Sampling-based Profiling（perf） |
|------|--------------------------|--------------------------------|
| **工作方式** | 每发生一次就显式记录 | 定时中断，读取当前调用栈 |
| **精确度** | 精确，不漏事件 | 统计近似，可能遗漏短事件 |
| **因果关系** | 可追溯 A→B→C 等待链 | 只能看到统计分布 |
| **异常捕获** | 能抓到罕见的慢事件 | 采样可能恰好跳过 |
| **开销模型** | 高频事件开销大（但 PFS 有限流） | 开销可控且稳定 |
| **根因分析能力** | 强（能看到"谁在等谁"） | 弱（只能看到"什么热"） |

**理论来源：**

Anderson 等人在 **SOSP 1997** 发表的 *"Continuous Profiling: Where Have All the Cycles Gone?"* 论证了持续可观测性对生产系统性能诊断的必要性——系统不应该只在出问题时才去"打开调试开关"，而应该默认就具备观测能力。DTrace 论文（Cantrill, Shapiro, Leventhal, USENIX ATC 2004）将其落实为生产级的全量追踪实现，PFS 是其在关系数据库领域的直接继承。

### 1.4 理论基础之二：AOP（面向切面编程）—— 编译期宏编织

**AOP（Aspect-Oriented Programming，面向切面编程）** 由 Kiczales 等人在 Xerox PARC 提出，发表于 **ECOOP 1997**。其核心思想是：将横跨多个模块的 **"横切关注点（cross-cutting concerns）"** 从业务代码中分离出来，集中定义，然后通过 **编织（weaving）** 机制在编译期或加载期注入回各个模块。

#### 1.4.1 AOP 三要素在 PFS 中的对应

| AOP 概念 | 英文 | 含义 | PFS 中的对应 |
|---------|------|------|-------------|
| **连接点** | Join Point | 可插入增强逻辑的位置 | 所有 `mysql_mutex_lock(M)` 调用点 |
| **通知/增强** | Advice | 在连接点插入的具体逻辑 | `start_mutex_wait()` + `end_mutex_wait()` |
| **编织/织入** | Weaving | 把 Advice 织入 Join Point 的过程 | `#define → inline_mysql_mutex_lock` 宏展开 |

#### 1.4.2 PFS 的编织过程（完整展开）

从 `mysql_mutex_lock(&dict_operation_lock)` 这一行业务代码开始：

```
【第 0 步】业务代码（开发者写的，完全无 PFS 代码）
    mysql_mutex_lock(&dict_operation_lock);

【第 1 步】第一层宏展开（编译期自动注入源码位置）
    → mysql_mutex_lock_with_src(&dict_operation_lock, __FILE__, __LINE__)
    // __FILE__ = "storage/innobase/btr/btr0sea.cc"
    // __LINE__ = 320

【第 2 步】第二层宏展开（跳转到 inline 函数内）
    → inline_mysql_mutex_lock(&dict_operation_lock, "btr0sea.cc", 320)

【第 3 步】inline 函数内逐层条件分叉
    #ifdef HAVE_PSI_MUTEX_INTERFACE          ← 编译期：PFS 编进去了吗？
        if (that->m_psi != nullptr)           ← 实例级：这个 mutex 注册了吗？
            if (that->m_psi->m_enabled)       ← 类别级：这个 class 开启了吗？

                // ✅ 编织点 A（Advice: start）
                locker = PSI_MUTEX_CALL(start_mutex_wait)(
                    &state, that->m_psi,
                    PSI_MUTEX_LOCK,         // 操作类型
                    "btr0sea.cc", 320);     // 源码定位

                // 真正的业务逻辑：等待锁
                result = my_mutex_lock(&that->m_mutex);

                // ✅ 编织点 B（Advice: end）
                if (locker != nullptr)
                    PSI_MUTEX_CALL(end_mutex_wait)(locker, result);
                return result;
    #else
        // ❌ 整块 Advice 在预处理阶段被完全删除
        result = my_mutex_lock(&that->m_mutex);
    #endif
```

#### 1.4.3 编织方式对比

| 方式 | 时机 | 代表技术 | 运行时开销 | PFS 选择 |
|------|------|---------|----------|---------|
| **编译期编织** | 编译时 | AspectJ (ajc), C 宏 | **零** | ✅ PFS 用这种 |
| **加载期编织** | 类加载时 | AspectJ LTW | 启动时一次性 | ❌ |
| **运行时编织** | 运行时 | Spring AOP (JDK 动态代理/CGLIB) | 每次调用有间接开销 | ❌ |

PFS 必须选择编译期编织是因为：mutex lock 每秒可能发生数百万次，运行时代理的额外开销完全不可接受。

#### 1.4.4 总结：从 AOP 到可观测性的完整链路

至此，我们可以给 PFS 一个精确的一句话描述：

> **PFS 是基于 AOP（面向切面编程）思想，通过编译期宏编织（Compile-time Macro Weaving）实现埋点/插桩（Instrumentation）来进行运行时可观测性（Runtime Observability）的动态追踪框架。**

这个描述揭示了四层递进关系：

```
AOP（面向切面编程）                  ← 理论思想
  └─ 编译期宏编织（Weaving）         ← 代码注入手段
      └─ 埋点/插桩（Instrumentation） ← 观测载体
          └─ 可观测性（Observability） ← 最终目标
```

**中文检索建议：**

如果需要检索相关中文资料，推荐以下关键词：

| 检索词 | 预期命中内容 | 与 PFS 的关联度 |
|--------|-------------|----------------|
| `面向切面编程 插桩` | AOP + instrumentation 的理论文献 | ★★★★ 直接相关 |
| `AOP 埋点 监控` | 工程实践，多为 Java 生态 | ★★★ 思想一致，实现不同 |
| `编译期织入 性能监控` | 与 PFS 最接近的方案 | ★★★★★ 高度相关 |
| `非侵入式 插桩 AOP` | 强调业务代码零修改 | ★★★★ 核心价值一致 |
| `compile-time weaving instrument C` | 英文检索，找 C/C++ 类似方案 | ★★★★★ 最精确 |

**需要注意的检索陷阱：** 中文互联网中"AOP + 插桩/埋点"的讨论主要集中在 **Java 生态**（AspectJ、Spring AOP），这些是运行时/加载期编织，和 PFS 的**编译期 C 宏编织**在实现机制上有本质区别。PFS 的这种做法在 C/C++ 系统软件领域相对独特——它用最原始的 C 预处理器实现了最先进的 AOP 思想。

### 1.5 理论基础之三：零开销探针（Zero-cost Probes）

来自 DTrace 论文（USENIX ATC 2004）的核心设计原则：**不启用时，探针必须不能拖慢生产系统**。

DTrace 在内核态的实现方式：每个探针点预先编译为几条 **NOP 指令**（No Operation，空操作，x86 上为 `0x90`），运行时如需激活，则动态将这些 NOP 在线改写为 `CALL probe_handler`。

PFS 在用户态通过三层防护实现等效效果：

| 层级 | 机制 | 关闭时的实际开销 |
|------|------|-----------------|
| **编译期** | `#ifdef HAVE_PSI_MUTEX_INTERFACE` 条件编译 | 代码在预处理阶段完全不存在 → **绝对零开销** |
| **实例级** | `if (m_psi != nullptr)` 空指针检查 | 1 条 `cmp` 指令 + CPU 分支预测命中 → ~1 cycle |
| **类别级** | `if (m_psi->m_enabled)` 使能标志 | 再 1 条 `cmp` + 分支预测命中 → ~2 cycles |

由于 `m_enabled` 和 `m_psi` 在运行时是**稳定状态**（不会频繁切换），CPU 分支预测器几乎 100% 命中。理论依据：Smith (ISCA 1981) 和 Yeh & Patt (ISCA 1991) 对分支预测的形式化分析。

### 1.6 PFS 的架构分层

```
┌────────────────────────────────────────────────────┐
│  展 现 层                                           │
│  SELECT ... FROM performance_schema.xxx             │
│  sys schema 便捷视图                                │
├────────────────────────────────────────────────────┤
│  引 擎 层 (storage/perfschema/)                    │
│  ├─ Ring Buffer: events_xxx_current                │
│  ├─ History Table: events_xxx_history              │
│  └─ Summary Table: events_xxx_summary_*            │
├───────────────── PSI ABI（插件接口边界）────────────┤
│  插 桩 层 (include/mysql/psi/)                     │
│  ├─ 编译期宏编织: inline_mysql_mutex_lock()         │
│  ├─ 内存分配: ut::malloc_withkey()                  │
│  └─ Statement 生命周期: PSI_statement 接口          │
├────────────────────────────────────────────────────┤
│  业 务 层                                           │
│  btr0cur.cc / row0purge.cc / lock0lock.cc ...       │
│  只调用 mysql_mutex_lock()，无 PFS 代码             │
└────────────────────────────────────────────────────┘
```

### 1.7 关键概念辨析：Event、Trace、Metrics、Profile

这些词经常被混用。它们在可观测性领域有精确的定义层次：

```
                    ┌─ Event（事件）──── 单次操作的完整记录
                    │  例：thread#42 在 btr0sea.cc:320
                    │       等待 dict_operation_lock 耗时 23.7μs
                    │
数据粒度 ──────────┼─ Trace（追踪）──── 有因果关系的 Event 链路
                    │  例：A 持锁 → B 等待 A → C 等待 B
                    │       (通过 m_nesting_event_id 关联）
                    │
                    └─ Metric（指标）─── 聚合后的统计量
                       例：COUNT=8,500,000, SUM=8.3s,
                            AVG=0.98μs, MAX=120ms

                    ┌─ Profile（性能画像）─── 按维度聚合的统计视图
                    │  等同于 PFS 的 *_summary_* 表
                    │  Profiling ≠ Sampling！
                    │  PFS 的 profile 用 tracing 数据聚合而成
                    │
分析视图 ──────────┼─ Flame Graph（火焰图）─── 采样结果的层次化可视化
                    │  PFS 不生成火焰图（那是 perf 的工作）
                    │
                    └─ Summary Table ── PFS 对 profile 的具体实现
```

**为什么要区分 Profiling 和 Sampling？**

业界大量 profiling 工具走 sampling 路线（如 `perf`、`gperftools`、Java Flight Recorder），导致 **"profiling ≈ sampling"** 成了一种思维惯性。但在可观测性理论里，**profiling 是一种输出（聚合统计画像），sampling 是一种采集手段（如何获取数据）**。PFS 用 tracing 手段同样可以得到 profile。

### 1.8 Instrumentation / Probing 是静态分析吗？—— 一个关键澄清

这是一个容易混淆的核心概念。简短回答：**PFS 的插桩/埋点/探针不是静态分析，而是动态分析（Dynamic Analysis）时代的运行时观测手段**。

#### 1.8.1 静态分析 vs 动态分析

| 概念 | 英文 | 工作方式 | 代表工具 | MySQL 中对应 |
|------|------|---------|---------|-------------|
| **静态分析** | Static Analysis | 不运行程序，分析源码/二进制。通过抽象解释、数据流分析、符号执行等发现缺陷 | clang-tidy、Coverity、cppcheck、Rust borrow checker | MySQL 编译时的 `-Werror` 检查、运行时无关 |
| **动态分析** | Dynamic Analysis | 程序运行时监测实际行为。通过代码注入、断点、采样等手段收集运行时信息 | Valgrind、ASan、gdb、Dtrace、Perf、PFS | ✅ PFS **整体属于这一类** |

#### 1.8.2 编译期编织 ≠ 静态分析

PFS 的宏展开、AOP 编织发生在编译期，这是"静态的"——但这只是**代码注入的方式（How to inject）**，不是**信息采集的时机（When to observe）**。决策树如下：

```
┌───────────────────────────────────────────────────┐
│  编译期编织（Compile-time Weaving）                 │
│  #define mysql_mutex_lock(M)                       │
│    → inline_mysql_mutex_lock(M, __FILE__, __LINE__) │
│                                                    │
│  【这是"静态"的——预处理阶段完成，决定注入什么代码】   │
└───────────────────┬───────────────────────────────┘
                    │ 注入进去的实际代码
                    ▼
┌───────────────────────────────────────────────────┐
│  运行时插桩（Runtime Instrumentation）              │
│  start_mutex_wait(&state, mutex, ...)   ← 运行时执行│
│  my_mutex_lock(&m_mutex)               ← 真操作     │
│  end_mutex_wait(locker, result)        ← 运行时执行 │
│                                                    │
│  【这是"动态"的——运行中采集真实的耗时、线程、锁等待】 │
└───────────────────────────────────────────────────┘
```

#### 1.8.3 学术分类中的准确定位

| 技术层次 | 英文术语 | 学术定义 | PFS 的归属 |
|---------|---------|---------|-----------|
| **代码注入方式** | Weaving Strategy | 如何把观测代码放入目标代码 | **编译期宏编织（Compile-time Macro Weaving）**：C 预处理 |
| **观测时机** | Observation Timing | 何时采集信息 | **运行时（Runtime）**：程序执行中实时采集 |
| **信息采集方式** | Collection Mode | 全部记录还是采样 | **Event-based Tracing（全量事件追踪）**：每次发生都记录 |
| **整体归类** | Classification | 在学术体系中的位置 | **Dynamic Instrumentation（动态插桩）** / **Dynamic Tracing（动态追踪）**：DTrace 家族 |

#### 1.8.4 动态插桩（Dynamic Instrumentation）的学术渊源

| 论文/系统 | 作者 | 会议 | 年份 | 贡献 |
|----------|------|------|------|------|
| *DTrace* | Cantrill et al. (Sun) | **USENIX ATC** | 2004 | **生产级动态追踪**：零开销探针、D 语言、安全保证，开创了"production system instrumentation"范式 |
| *Pin* | Luk et al. (Intel/CMU) | **PLDI** | 2005 | **通用动态二进制插桩**：在指令级插入分析代码，理论证明任意程序可被 instrumentation |
| *Dyninst* | Hollingsworth et al. (U. Maryland) | **PLDI / SC** | 1996- | 运行时动态插入/移除 instrumentation 代码，无需重新编译 |
| *Valgrind* | Nethercote & Seward | **PLDI** | 2007 | 二进制级别的 heavyweight 动态分析框架（PFS 不用，但同属动态分析大类） |

PFS 与 Pin/Dyninst 的区别在于：PFS 的 instrumentation 代码在**编译时**就已经确定并编入二进制（只是通过宏控制是否激活），而 Pin/Dyninst 可以在运行时动态插入全新的分析代码。这使 PFS 的开启/关闭开销更低（仅分支预测级别），但灵活性不如 Pin。

---

## 第二章：InnoDB 内存分配器矩阵

### 2.1 全景架构

InnoDB 不是用一个通用分配器管理所有内存，而是根据生命周期、大小和底层来源，构建了一个 **分层协作的分配器矩阵**：

```
┌──────────────────────────────────────────────────────┐
│  事务生命周期 / 临时计算                               │
│  ┌──────────────────────────────────────────────┐    │
│  │  mem_heap (InnoDB 引擎内部)                    │    │
│  │  类型: Bump/Arena Allocator                   │    │
│  │  来源: 小块→ut::malloc, 大块→Buffer Pool 页面  │    │
│  │  特色: MEM_HEAP_BTR_SEARCH 死锁规避            │    │
│  │        MEM_NO_MANS_LAND Debug 越界检测         │    │
│  │        Scoped_heap RAII 封装                  │    │
│  └──────────────────────────────────────────────┘    │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │  MEM_ROOT (Server 层)                          │    │
│  │  类型: Arena Allocator                        │    │
│  │  来源: my_malloc                              │    │
│  │  特色: 指数增长 block size                     │    │
│  │        Mem_root_allocator STL 适配            │    │
│  └──────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────┤
│  Buffer Pool 内部分配                                 │
│  ┌──────────────────────────────────────────────┐    │
│  │  buf_buddy                                    │    │
│  │  类型: Binary Buddy Allocator                 │    │
│  │  来源: Buffer Pool 预分配页面（BUF_BLOCK_MEMORY）│    │
│  │  理论: Knowlton 1965 (CACM), Knuth 1968       │    │
│  │  场景: 压缩页 frame 分配                       │    │
│  └──────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────┤
│  OS 堆分配（最终所有上层都落到这里）                   │
│  ┌──────────────────────────────────────────────┐    │
│  │  ut::malloc_withkey (PFS 埋点包装)             │    │
│  │  类型: Instrumented Wrapper（无新分配算法）    │    │
│  │  作用: 每次分配向 PFS 报告 key + size           │    │
│  └──────────────────────┬───────────────────────┘    │
│  ┌──────────────────────┴───────────────────────┐    │
│  │  jemalloc / glibc malloc (OS 分配器)          │    │
│  │  类型: Segregated-fit Allocator               │    │
│  │  思想: size class + thread cache + arena      │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### 2.2 mem_heap —— InnoDB 的 Bump Allocator（指针碰撞分配器）

#### 2.2.1 核心思想与数据结构

mem_heap 是 InnoDB 内部使用最广泛的临时内存分配器。本质是 **Bump Allocator（指针碰撞分配器）**——维护一个 `free` 偏移量指针，分配时直接向前推进，O(1) 完成。

```cpp
// mem_heap_alloc 的核心逻辑（mem0mem.ic:162-165）
buf = (byte *)block + free + MEM_NO_MANS_LAND;           // 计算返回地址
mem_block_set_free(block, free + MEM_SPACE_NEEDED(n));   // 推进 free 指针
```

数据结构：heap 是一个 `mem_block_t` 的非空线性链表：

```
mem_heap_t (heap)
  │
  ├── block 1（也是 heap 的根节点）
  │     ├── mem_block_info_t 头部（含 magic_n, type, len, free, start 等）
  │     ├── [0xCE×16] alloc_1 用户数据 [0xDF×16] + (对齐填充)
  │     ├── [0xCE×16] alloc_2 用户数据 [0xDF×16] + (对齐填充)
  │     ├── [0xCE×16] alloc_3 用户数据 [0xDF×16] + (对齐填充)
  │     └── ...剩余空闲空间...
  │
  └── block 2（不够用了，追加的）
        └── 同上结构
```

**理论来源：**Bump Allocator 是最古老、最基本的分配器策略，无特定论文——它从 1960s 编译器实现中自然产生，在 Knuth 的 *The Art of Computer Programming* Vol.1 (1968) 中有形式化描述。它的设计哲学是**以放弃单块释放的能力，换取极致的分配速度和零碎片**。

#### 2.2.2 三种 Heap Type —— 核心设计

mem_heap 最独特的设计是三种类型决定了**内存来源**和**分配失败行为**的分叉：

```cpp
// mem0mem.h
constexpr uint32_t MEM_HEAP_DYNAMIC    = 0;   // 走 ut::malloc
constexpr uint32_t MEM_HEAP_BUFFER     = 1;   // 大块走 Buffer Pool
constexpr uint32_t MEM_HEAP_BTR_SEARCH = 2;   // 标志位：用于 AHI 死锁规避

// 组合体：
constexpr uint32_t MEM_HEAP_FOR_BTR_SEARCH = MEM_HEAP_BTR_SEARCH | MEM_HEAP_BUFFER; // = 3
constexpr uint32_t MEM_HEAP_FOR_PAGE_HASH  = MEM_HEAP_DYNAMIC;                       // = 0
constexpr uint32_t MEM_HEAP_FOR_RECV_SYS   = MEM_HEAP_BUFFER;                        // = 1
```

**分支逻辑在 `mem_heap_create_block`（memory.cc:248-350）中：**

```cpp
mem_block_t *mem_heap_create_block(mem_heap_t *heap, ulint n, ulint type) {
  len = MEM_BLOCK_HEADER_SIZE + MEM_SPACE_NEEDED(n);

  // 关键分支：
  if (type == MEM_HEAP_DYNAMIC || len < UNIV_PAGE_SIZE / 2) {
    // 路径 A：走 ut::malloc
    // 条件1：DYNAMIC 类型无论多大都走 malloc
    // 条件2：即使 BUFFER 类型，小于半页(~8KB)也不值得浪费 BP 页面
    block = ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, len);
  } else {
    // 路径 B：从 Buffer Pool 分配一个完整页面
    len = UNIV_PAGE_SIZE;

    if ((type & MEM_HEAP_BTR_SEARCH) && heap) {
      // 子路径 B1：AHI 场景——从预留的 free_block 取
      buf_block = heap->free_block_ptr->load();
      if (!buf_block) return nullptr;  // 优雅失败！
      heap->free_block_ptr->store(nullptr);
    } else {
      // 子路径 B2：正常从 BP free list 取
      buf_block = buf_block_alloc(nullptr);
    }

    block = (mem_block_t *)buf_block->frame;
  }
}
```

**完整分配策略表：**

| Type | 值 | 小块 (<8KB) | 大块 (≥8KB) | 失败时 | 典型场景 |
|------|-----|------------|------------|--------|---------|
| `MEM_HEAP_DYNAMIC` | 0 | `ut::malloc` | `ut::malloc` | fatal 崩溃 | 通用临时 heap、DDL |
| `MEM_HEAP_BUFFER` | 1 | `ut::malloc` | BP 页面 (`buf_block_alloc`) | fatal 崩溃 | recovery、lock heap |
| `MEM_HEAP_FOR_BTR_SEARCH` | 3 | `ut::malloc` | 从 `free_block_ptr` 原子取 | **返回 NULL** | AHI hash table |

**为什么小块即使 type 是 BUFFER 也走 malloc？** 不值得为几 KB 的临时分配浪费一整个 16KB Buffer Pool 页面——那可是一个数据页缓存的代价。

#### 2.2.3 MEM_HEAP_BTR_SEARCH —— 死锁规避的精妙设计

这是 InnoDB 中极具工程智慧的设计。AHI（Adaptive Hash Index，自适应哈希索引）在插入新条目时需要持有 AHI latch，而扩展 heap 又可能需要从 Buffer Pool 分配页面。如果 BP 满了需要 LRU 驱逐，驱逐调用链是：

```
Thread A 持有 AHI X-latch
  → mem_heap_alloc 需要更多空间
  → mem_heap_create_block
  → buf_block_alloc  ← 从 BP 分配
  → 但 BP 满了！
  → buf_LRU_get_free_block
  → buf_LRU_scan_and_free_block
  → buf_LRU_free_page
  → btr_search_drop_page_hash_index  ← 需要 AHI latch！
  → 自己等自己 → 死锁 💀
```

**解法：预留在先（InnoDB 三步法）**

1. **预留在先（获取 AHI latch 之前）**：调用 `btr_search_check_free_space_in_heap()`，检查 `free_block_for_heap` 原子变量是否为空，为空就从 BP 分配一个页面存进去。此时还没持有 AHI latch，不会有死锁。
2. **用时不分配（持有 latch 后）**：`mem_heap_create_block` 发现 `type & MEM_HEAP_BTR_SEARCH` 且 `heap` 不为空时，不从 BP 新分配，而是从 `heap->free_block_ptr` 原子取走预留页面。如果取不到，**返回 NULL 而非崩溃**，调用方优雅处理。
3. **用后补充**：释放 AHI latch 后，下次操作前会再次调用 `btr_search_check_free_space_in_heap` 重新补充。

```cpp
// btr0sea.cc:155-184
static inline void btr_search_check_free_space_in_heap(dict_index_t *index) {
  auto &free_block_for_heap = btr_get_search_part(index).free_block_for_heap;
  if (free_block_for_heap.load() == nullptr) {
    const auto block = buf_block_alloc(nullptr);  // 还没拿 latch，安全
    buf_block_t *expected = nullptr;
    if (!free_block_for_heap.compare_exchange_strong(expected, block)) {
      buf_block_free(block);  // 并发 race：别人已经设了，归还这个
    }
  }
}
```

这个设计确保了：AHI latch 持有者永远不会为了获取 latch 而去拿自己已经持有的 latch——经典的锁顺序死锁规避。

#### 2.2.4 Debug 机制：No Man's Land（金丝雀字节 / 哨兵字节）

仅在 Debug 模式（`UNIV_DEBUG`）下启用。每次 `mem_heap_alloc` 在返回给用户的数据前后各放置 16 字节的哨兵（canary bytes）：

```
┌──────────────┬─────────────────┬──────────────┬──────────┐
│  0xCE × 16   │  用户数据 (N 字节)  │  0xDF × 16   │ (对齐填充) │
│  前方哨兵区   │                  │  后方哨兵区   │          │
└──────────────┴─────────────────┴──────────────┴──────────┘
```

```cpp
// mem0mem.h:99-108
#ifdef UNIV_DEBUG
  constexpr int MEM_NO_MANS_LAND = 16;
#else
  constexpr int MEM_NO_MANS_LAND = 0;    // Release 下完全消除
#endif

const byte MEM_NO_MANS_LAND_BEFORE_BYTE = 0xCE;  // 前方填充字节
const byte MEM_NO_MANS_LAND_AFTER_BYTE  = 0xDF;  // 后方填充字节
```

释放时 `validate_no_mans_land()` 逐字节校验：

```cpp
// mem0mem.ic:189-198
static inline void validate_no_mans_land(
    byte *no_mans_land_begin, byte mem_no_mans_land_byte) {
  for (byte *it = no_mans_land_begin;
       it < no_mans_land_begin + MEM_NO_MANS_LAND; ++it) {
    ut_a(*it == mem_no_mans_land_byte);  // 被篡改了？直接 assert
  }
}
```

**这是经典的 canary byte / guard byte / fence 机制**，是发现 buffer overflow 和 underflow 的第一道防线。

**业界类似实现：**

| 实现 | 来源 | 值 | 说明 |
|------|------|-----|------|
| **mem_heap** | InnoDB | `0xCE` / `0xDF` × 16 | 前后哨兵，Debug-only |
| **Windows Debug Heap** | Microsoft CRT | `0xFD` | 分配块前后 guard |
| **Red zone** | AddressSanitizer | 不可访问页 | 运行时检测越界 |
| **Electric Fence** | Bruce Perens | 保护页 | 经典的 malloc 调试库 |

**为什么选 0xCE 和 0xDF？** 这两个值没有特殊数学含义。选择原则：非零（避免漏检零值写入）、前后不同（便于区分方向）、在 hex dump 中连续出现时非常显眼——正常数据不可能连续 16 字节完全相同。

#### 2.2.5 Scoped_heap —— RAII 封装

`Scoped_heap` 将 C 风格的 `mem_heap_t` 封装为 RAII 对象，底层用 `std::unique_ptr` + 自定义 deleter 实现自动释放：

```cpp
// mem0mem.h:440-507
struct Scoped_heap {
  struct mem_heap_free_functor {
    void operator()(mem_heap_t *heap) { mem_heap_free(heap); }
  };
  using Ptr = std::unique_ptr<mem_heap_t, mem_heap_free_functor>;
  Ptr m_ptr{};

  // 禁止拷贝和移动（所有权唯一）
  Scoped_heap(Scoped_heap &&) = delete;
  Scoped_heap(const Scoped_heap &) = delete;

  // 构造即创建 MEM_HEAP_DYNAMIC 类型的 heap
  Scoped_heap(size_t n, ut::Location location) noexcept
      : m_ptr(mem_heap_create(n, location, MEM_HEAP_DYNAMIC)) {}
};
```

使用者：`ddl0impl-builder.h`、`ddl0impl-cursor.h`、`lock0lock.cc`、`btr0cur.cc` 等 C++ 代码中需要临时 heap 的地方。

### 2.2.6 跨分配器内存复用能力分析

一个常见的误解是"bump allocator 完全没有内存复用"。准确地说：bump allocator **放弃了任意位置的单块复用能力**，但保留了**粗粒度的快照级复用**。

#### 各分配器复用能力总览

| 分配器 | 单块级复用 | block级复用 | 复用机制 | 设计哲学 |
|--------|-----------|------------|---------|---------|
| **buf_buddy** | ✅ 完整 | ✅ 完整 | free list + buddy 合并（`zip_free[]`） | 长生命周期、按需归还 |
| **jemalloc** | ✅ 完整 | ✅ 完整 | thread cache (tcache) + arena → bin → slab | 通用 malloc，高性能 |
| **mem_heap** | ❌（仅栈顶 `free_top`） | ✅（`mem_heap_empty`） | bump allocator，无 free list | 短生命周期临时分配，用完整体释放 |
| **MEM_ROOT** | ❌ | ✅（`ClearForReuse`） | 同 bump，但 block 可保留复用 | 一次查询的临时内存 |

#### mem_heap 的三种复用粒度

```
粒度1：mem_heap_free_top —— 栈顶单块回退
  适用场景：迭代构建最终结果
  例：row_merge_buf_add() 先分配缓冲区，失败时 free_top 回退
  → free 指针回退，下次分配 "覆盖" 之前的数据（栈式复用）

粒度2：mem_heap_empty —— 整块重置
  适用场景：多次执行同一操作
  例：lock_rec_other_has_expl_req() 在遍历事务时重置临时 heap
  → free 指针回到 start，复用整个 block 的空间

粒度3：mem_heap_free —— 完全销毁
  适用场景：操作结束，不再需要
  → 整个 heap 归还，没有任何复用
```

#### 为什么 bump allocator 不做细粒度复用？

| 如果要做 | 需要付出的代价 | 为什么 bump 不做 |
|---------|-------------|-----------------|
| 记录每个空闲位置 | free list（链表管理开销） | 违背了"两条指令完成分配"的极简目标 |
| 合并相邻空闲块 | 合并逻辑（CPU 开销） | 对 microsecond 级的临时操作不可接受 |
| 按大小匹配合适空洞 | size class / best-fit 搜索 | bump 只需要追 free 指针，不需要搜索 |
| 线程安全 | 锁 / atomic | 一般 heap 由单线程持用，省去同步开销 |

**核心洞察：** bump allocator 不是"做不到"细粒度复用，而是**刻意不做**。它用"放弃复用"换来了极致的分配速度和零碎片——这对于 InnoDB 内部频繁的、短生命周期的临时分配（一次 B-tree 操作、一次 undo 构建）是正确权衡。补偿性手段是粗粒度的 `free_top` / `empty`，允许在一个操作的生命周期内重复利用同一块内存。

### 2.3 MEM_ROOT —— Server 层的 Arena Allocator

#### 2.3.1 核心思想

MEM_ROOT 是 MySQL Server 层（非 InnoDB）的 arena 分配器。思想与 mem_heap 同源（bump allocate + 批量释放），但设计上有显著差异以适应 Server 层的不同需求。

#### 2.3.2 与 mem_heap 的详细对比

| 维度 | mem_heap | MEM_ROOT |
|------|---------|----------|
| **所属层次** | InnoDB 引擎 | MySQL Server 层 |
| **底层分配接口** | `ut::malloc_withkey` + Buffer Pool | `my_malloc` |
| **Buffer Pool 集成** | ✅ 大块直接从 BP 拿页面，统一管理 | ❌ 无关（Server 层不碰 BP） |
| **Block 增长策略** | 固定大小（初始 size 决定） | **指数增长**：`m_block_size += m_block_size/2`（每次 +50%），保证 O(1) 次 malloc 调用 |
| **增长哲学** | "使用者知道自己要多少"——InnoDB 内部场景高度可预测（一次 B-tree 搜索、一次 undo 构建），block size 在创建 heap 时就确定 | "自适应用户的任意需求"——Server 层面对无法预知的 SQL 复杂度（简单的 `SELECT 1` 到 20 表 JOIN），block size 随分配次数自动扩大 |
| **单块释放能力** | ✅ 支持 `mem_heap_free_top`（LIFO 栈式释放）。因为 block 内分配按地址线性递增，可精确回退 `free` 指针 | ❌ 完全不支持任何单块释放。`Clear()` 或 `ClearForReuse()` 只能全部清空 |
| **清空复用** | `mem_heap_empty`：回退 free 指针，可释放首 block 之外的 block | `Clear()`：释放所有 block；`ClearForReuse()`：保留首 block，返回 OS 的仅有一个 |
| **容量限制** | 无 | `m_max_capacity`，超限可触发 `EE_CAPACITY_EXCEEDED` |
| **多 block 维护策略** | block 链表通过 `prev` 单向链接，追加 block 时直接 `new_block->prev = m_current_block`，结构简单（因为 block 大小固定，不需要复杂的插入位置计算） | 大块分配时插入倒数第二位置（`new_block->prev = m_current_block->prev; m_current_block->prev = new_block`），将大块与当前活跃块隔离，避免后续小分配浪费空间 |
| **Debug 机制** | `MEM_NO_MANS_LAND`（自定义哨兵字节） | 依赖 Valgrind/ASan（`MEM_ROOT_SINGLE_CHUNKS` 模式：每分配一个独立 chunk） |
| **C++ 集成** | `Scoped_heap`（RAII 封装） | `Mem_root_allocator<T>`（STL allocator，支持 `std::vector` 等容器） |
| **AHI 死锁规避** | ✅ `MEM_HEAP_BTR_SEARCH` + `free_block_ptr` | ❌ 无此概念 |
| **PFS 埋点** | 通过 `ut::malloc_withkey` → `PSI_memory_key` | 通过 `my_malloc` + `PSI_memory_key` |
| **代码位置** | `storage/innobase/mem/` + `include/mem0mem.*` | `mysys/my_alloc.cc` + `include/my_alloc.h` |

#### 2.3.3 MEM_ROOT 的指数增长与独立大块分配

```cpp
// my_alloc.cc:101-103
// 每次成功分配后，下次 block size 增大 50%
m_block_size += m_block_size / 2;

// AllocSlow 中：大请求不干扰小请求
if (length >= m_block_size || MEM_ROOT_SINGLE_CHUNKS) {
    // 单独分配一个 block，插入倒数第二位置
    new_block->prev = m_current_block->prev;
    m_current_block->prev = new_block;
}
```

这个设计保证：偶尔来一个巨大的分配（如 2MB）不会导致后续所有小分配都从 2MB 块中分配（那会浪费大量空间），而是隔离到独立 block。

### 2.4 buf_buddy —— Binary Buddy Allocator（二叉伙伴分配器）

#### 2.4.1 理论基础

二叉伙伴系统由 K. C. Knowlton 于 1965 年在 **CACM（Communications of the ACM）** 发表 *"A Fast Storage Allocator"* 中首次提出。Knuth 在 *The Art of Computer Programming* Vol.1 (1968) 中做了形式化完善。

#### 2.4.2 核心算法

内存按 2 的幂次划分为不同级别。分配时找最小满足块，不够就向上一级分裂；释放时检查伙伴是否也空闲，如果是则合并：

```
伙伴地址公式:  buddy_addr = addr XOR (1 << order)

例: order=3 (块大小 2³=8 单位)
    addr=0x00 → buddy=0x08
    addr=0x08 → buddy=0x00
```

#### 2.4.3 MySQL 中的实现

```cpp
// buf_pool_t 中的 zip_free 数组
UT_LIST_BASE_NODE_T(buf_buddy_free_t, list) zip_free[BUF_BUDDY_SIZES_MAX];

// 分配接口
static inline byte *buf_buddy_alloc(buf_pool_t *buf_pool, ulint size) {
    return buf_buddy_alloc_low(buf_pool, buf_buddy_get_slot(size));
}
```

通过 `BUF_BUDDY_STAMP_FREE` 魔数（写在 page 的 `space_id` 字段位置复用）判断伙伴是否空闲，避免引入额外元数据。

#### 2.4.4 使用场景

**仅用于压缩页（compressed page）frame 的分配**。压缩表的一个页面压缩后可能只有 1KB/2KB/4KB/8KB，不需要完整 16KB Buffer Pool 页面。buddy allocator 从 BP 中切割出合适大小的块，避免大量空间浪费。

### 2.5 jemalloc —— 底层 OS 分配器

#### 2.5.1 理论渊源

| 论文/工作 | 会议 | 年份 | 贡献 |
|----------|------|------|------|
| Hoard | **ASPLOS** | 2000 | per-thread heap + global heap，消除多线程 false sharing 和 memory blowup |
| jemalloc | **BSDCan** | 2006 | 产品级可扩展 malloc，多 arena 设计 |
| Slab Allocator | **USENIX Summer TC** | 1994 | 按对象大小预分配缓存（size class 思想源头）|

#### 2.5.2 size class 是什么？

size class（大小分级）是 segregated-fit 分配器的核心机制，也是 jemalloc 超越简单 buddy allocator 的关键设计。

**问题：** 假设程序连续分配 3、7、15 字节的对象，然后全部释放。下次再分配 5 字节，从哪里拿？

```
bump allocator 的回答：  从头开始推指针（无复用）
buddy allocator 的回答：  向上取 2^n（3→4，7→8，15→16），释放后合并回大块，
                          下次分配 5→8，重新从大块切割
size class 的回答：       3、5、7 都归入 "8 字节" 这个 size class，
                          释放的 3、7 字节块直接放入 8 字节 bin 的空闲链表，
                          下次分配 5 字节，直接从这个链表取——零切割、零搜索
```

**一句话定义：** size class 是将内存请求按**预设的分档**向上取整（round-up），同一档内的所有分配/释放共享一个 free list。释放的块不需要分裂或合并，直接作为同档内存复用。

| 概念对比 | 工作机制 | 碎片 | 复用效率 |
|---------|---------|------|---------|
| **bump allocator** | 无分类，线性推指针 | 零外部碎片 | 无单块复用 |
| **buddy allocator** | 按 2^n 分裂/合并 | 内部碎片（7→8 浪费 1 字节） | 需合并后才能复用为大块 |
| **size class (jemalloc)** | 按预定义档位向上取整，每档独立 free list | 内部碎片可控（档位设计均衡） | 释放后立即可被同档分配复用 |

**size class 与 buddy 的 2^n 有什么区别？**

buf_buddy 也按 2 的幂次分 size，但它服务于**分裂/合并**——buddy 的意义在于"两个相邻的 4K 空闲块可以合并成 8K"。size class 服务于**对象缓存**——意义在于"所有 5~8 字节的分配都用同一个 bin，释放后可以被下一个 5~8 字节请求直接拿走，不需要任何操作"。jemalloc 通常有几十个精细的 size class（不是简单的 2^n，而是 8, 16, 32, 48, 64, 80, 96...），这样内部碎片被控制在 25% 以内。

**理论来源：** Bonwick (Sun) 在 USENIX Summer TC 1994 发表的 *"The Slab Allocator: An Object-Caching Kernel Memory Allocator"* 首次系统化地提出了这个思想——为每种对象大小预分配 slab，同类对象复用同一 slab 内的槽位。jemalloc 将此思想从内核态带到用户态通用 malloc。

#### 2.5.3 jemalloc 的核心设计

```
请求 size
    │
    ▼
根据 size class 映射到 bin
    │
    ├── 小对象 (≤ 特定阈值)
    │   └── thread cache (tcache) → 无锁分配
    │
    ├── 中等对象
    │   └── arena → bin → slab → 有锁但竞争低
    │
    └── 大对象
        └── mmap → 直接映射，free 时 munmap
```

MySQL 编译时可选链接 jemalloc 或 glibc malloc。jemalloc 的优势：多核扩展性好、碎片更少、支持精确的内存用量统计（对 PFS 汇总有利）。

### 2.6 各分配器的理论渊源汇总

| 分配器 | 算法类型 | 核心论文 | 会议/出处 | 年份 |
|--------|---------|---------|----------|------|
| **mem_heap** | Bump/Arena | 无特定论文（1960s 编译器实践） | Knuth TAOCP Vol.1 | 1968 |
| **MEM_ROOT** | Arena（指数增长） | 无特定论文 | — | — |
| **buf_buddy** | Buddy Allocator | Knowlton, "A Fast Storage Allocator" | **CACM** | 1965 |
| **jemalloc** | Segregated-fit | Evans, "A Scalable Concurrent malloc(3)" | **BSDCan** | 2006 |
| **jemalloc 思想源泉** | Multi-thread scalable | Berger et al., "Hoard" | **ASPLOS** | 2000 |
| **size class 思想源泉** | Object-caching | Bonwick, "The Slab Allocator" | **USENIX Summer TC** | 1994 |

---

## 第三章：可观测性与内存管理的交汇

### 3.1 ut::malloc_withkey —— 两条线的唯一交汇点

`ut::malloc_withkey` 是 PFS 可观测性框架与 InnoDB 内存管理系统之间的桥梁。它**不实现任何新的分配算法**，而是对底层 `malloc`（jemalloc 或 glibc）做 PFS 埋点包装：

```cpp
// ut0new.h:619-626
inline void *malloc_withkey(PSI_memory_key_t key, std::size_t size) noexcept {
  using impl = detail::select_malloc_impl_t<WITH_PFS_MEMORY, false>;
  using malloc_impl = detail::Alloc_<impl>;
  return malloc_impl::alloc<false>(size, key());
}
```

每次分配时向 PFS 报告：哪个 `key`（模块标识）、分配了多少字节。这使得运维者可以通过 SQL 查询精确知道每个 InnoDB 子模块的内存占用。

### 3.2 全链路观测

```
mem_heap_create_block
  ├── 小块（<8KB）→ ut::malloc_withkey(KEY_mem_heap, size) → jemalloc → PFS 统计
  └── 大块（≥8KB, BUFFER 类型）→ buf_block_alloc → BP free list → PFS 统计

MEM_ROOT::AllocBlock
  └── my_malloc(PSI_key, size) → jemalloc → PFS 统计

buf_buddy_alloc
  └── zip_free[] 空闲链表 → buddy 内部管理 → buddy 自身统计
```

在 PFS 中查询效果：

```sql
SELECT event_name, current_alloc, high_alloc
FROM memory_summary_global_by_event_name
WHERE event_name LIKE '%innodb%'
ORDER BY current_alloc DESC;
-- memory/innodb/buf_buf_pool         8.5 GB
-- memory/innodb/mem_heap             1.2 GB
-- memory/innodb/btr_search           800 MB
-- memory/innodb/row0merge            500 MB
```

---

## 第四章：MySQL 源码中的设计模式

> **不懂设计模式，读起源码如同盲人摸象**——每一行代码都认识，但不知道为什么这么写、每一层包装是干什么的。本章梳理 MySQL（尤其 PFS 和内存管理模块）中实际使用的设计模式，结合具体代码说明其动机、结构和理论来源。掌握这些模式后，你会发现那些看似复杂的宏展开、函数指针跳转、多层封装，背后都是经典的、可预测的设计套路。

### 4.0 阅读 MySQL 源码需要补什么？—— 完整知识地图

这一节的目标不是"列出所有可能需要的东西"，而是**从源码出发，告诉你某某行代码为什么读不懂、你要补什么才能读懂**。按遇到问题的频率和阻塞程度排列。

---

#### 4.0.1 总览：你必须掌握的七个知识域

```
                              ┌─────────────────────┐
                              │   7. 工程工具链       │
                              │   CMake, gdb, git    │
                              │   ctags, perf, ASAN  │
                              ├─────────────────────┤
                              │   6. 计算机体系结构    │
                              │   cache line, NUMA   │
                              │   TLB, branch pred   │
                              ├─────────────────────┤
                              │   5. 并发与内存模型    │
                              │   futex, spinlock    │
                              │   atomic, memory_order│
                              ├─────────────────────┤
                              │   4. 操作系统基础      │
                              │   mmap, huge page    │
                              │   virtual memory     │
               ┌──────────────┴─────────────────────┴──────────────┐
               │                    3. 设计模式                    │
               │    Strategy / Template Method / Observer / Facade │
               │    Bridge / Adapter / Decorator / Factory / RAII │
               ├──────────────────────────────────────────────────┤
               │              2. 语言：C 预处理器 + C++ 惯用法     │
               │    宏展开 ## # 条件编译  /  template RAII atomic │
               ├──────────────────────────────────────────────────┤
               │              1. 领域理论基础                      │
               │    Event Tracing, AOP, DTrace, Buddy, Slab,      │
               │    Bump/Arena, Segregated-fit, Size Class        │
               └──────────────────────────────────────────────────┘
```

**读源码不是先学完所有知识再开始**，而是**遇到看不懂的地方 → 定位是哪个知识域 → 去补 → 再回来看**。下面按这个逻辑展开。

---

#### 4.0.2 第一域：领域理论基础（最优先，决定了你"为什么读"）

**如果不知道 buf_buddy 是 buddy allocator，你会把它当成普通的 linked-list 分配器来读，然后就觉得"这段代码好奇怪"然后放弃。**

| 读到哪里 | 会遇到什么困惑 | 需要补充的知识 | 补什么资料 |
|---------|--------------|--------------|----------|
| `mem0mem.h:86-91` 看到三种 type | "DYNAMIC 和 BUFFER 到底有什么区别？为什么还有 BTR_SEARCH 这个奇怪的组合值？" | Bump/Arena allocator 的设计取舍：从堆分配 vs 从预分配池分配，不可失败 vs 可返回 NULL | 本文第二章；Knuth TAOCP Vol.1 §2.5 |
| `buf0buddy.cc:buf_buddy_alloc_low` | "为什么按 `1 << i` 分裂？释放时怎么找到伙伴？" | Buddy allocator 的分裂/合并算法，伙伴地址公式 `buddy = addr XOR (1 << order)` | Knowlton 1965 (CACM)；本文 2.4 节 |
| `my_alloc.cc:103` 看到 `m_block_size += m_block_size / 2` | "为什么每次增加 50%？固定大小不好吗？" | Arena allocator 的指数增长策略 vs 固定块策略的适用场景 | mem_heap 作为对比：InnoDB 场景可预测 vs Server 层不可预测 |
| `memory.cc:278` 看到 `type == MEM_HEAP_DYNAMIC \|\| len < UNIV_PAGE_SIZE / 2` | "为什么 BUFFER 类型的小块也走 malloc？" | Buffer Pool 页面的机会成本：16KB 的 BP 页面换几 KB 临时数据是赔本买卖 | Buffer Pool 管理基础知识 |
| `jemalloc` 的设计 | "size class 和 buddy 的 2^n 级别有什么区别？" | Segregated-fit：对象按大小分类，每类独立 free list，释放后可**立即**被同大小请求复用 | Bonwick 1994 Slab Allocator 论文；本文 2.5.2 节 |
| PFS `events_waits_summary_*` 表 | "event tracing 和 sampling 哪个好？为什么 PFS 选前者？" | Event-based Tracing vs Sampling 的理论基础：精确度、因果关系、开销模型 | Anderson SOSP 1997；DTrace USENIX ATC 2004 |
| `mysql_mutex.h` 中嵌套的 `#define` | "为什么 lock 一个 mutex 要通过这么多层宏？" | AOP 的 weaving 思想：横切关注点分离 + 编译期零开销注入 | Kiczales ECOOP 1997；本文 1.4 节 |

---

#### 4.0.3 第二域：C 预处理器 + C++ 惯用法（决定了你"看不看得懂代码字面意思"）

**这是最大的阅读障碍来源**。MySQL 大量依赖 C 预处理器宏生成代码，不掌握预处理器就读不懂任何 PFS 代码。

**C 预处理器——必学清单：**

| 你必须理解的 | 为什么必须 | MySQL 中的典型例子 | 检验方法 |
|-------------|----------|-------------------|---------|
| **宏展开顺序**：参数先展开，然后替换 | 不懂就看不懂多层嵌套宏 | `#define mysql_mutex_lock(M) mysql_mutex_lock_with_src(M, __FILE__, __LINE__)` 继续展开到 `inline_mysql_mutex_lock(M, "file.cc", 320)` | 用 `gcc -E` 展开一个简单的 `mysql_mutex_lock(&foo)` 看输出 |
| **`#` 和 `##` 操作符** | PFS 中 `__FILE__`、自动拼接标识符都靠它们 | `#define PSI_MUTEX_CALL(M) psi_mutex_service->M` 中的函数名拼接 | 写一个简单宏测试 `#`(stringify) 和 `##`(token paste) |
| **条件编译 `#ifdef`** | PFS 的零开销关闭完全依赖它 | `#ifdef HAVE_PSI_MUTEX_INTERFACE` → 决定整段 instrumentation 代码是否存在 | 理解 `-DHAVE_PSI_MUTEX_INTERFACE` 编译选项如何改变生成的二进制 |
| **可变参数宏 `__VA_ARGS__`** | `ib::fatal()` 等日志宏使用 | `#define ut_a(EXPR) (void)((EXPR) || (ut_a_print(#EXPR, ...), 0))` | 找到 `ut_a` 的定义，手动展开 |
| **X-Macro 技巧** | PFS 大量使用 "用数据驱动代码生成" 的模式 | PFS 的 instrument class 注册表：一个宏定义数据表，另一个宏定义遍历逻辑 | 搜索 `PFS_engine_table_share` 相关宏 |

**C++ 必须掌握的惯用法——按在 MySQL 源码中出现频率排序：**

| 频率 | 你必须懂的 C++ 特性 | 不懂的后果 | MySQL 中哪里出现 | 推荐阅读 |
|------|-------------------|----------|----------------|---------|
| ⭐⭐⭐⭐⭐ | **RAII + `unique_ptr` + 自定义 deleter** | 看不懂 `Scoped_heap`、`Mem_root_allocator` 为什么"没有手动 free" | `mem0mem.h:440-507` Scoped_heap；`my_alloc.h` Mem_root_allocator | Stroustrup C++ 4th Ed. §5.2, §34.3 |
| ⭐⭐⭐⭐⭐ | **`std::atomic` + `memory_order`** | 看不懂 `free_block_ptr` 的 `compare_exchange_strong` | `mem0mem.h` free_block_for_heap；`rpl_handler.cc` Delegate spin lock | C++ Concurrency in Action §5 |
| ⭐⭐⭐⭐ | **模板特化 + SFINAE** | 看不懂 `ut::malloc_withkey` 为什么用 `select_malloc_impl_t` | `ut0new.h:619` 模板选择：`WITH_PFS_MEMORY == true` 时走 PFS 包装 | cppreference: SFINAE |
| ⭐⭐⭐⭐ | **`constexpr` / `static_assert`** | 看不懂编译期常量判断（如 `MEM_NO_MANS_LAND` 在 Debug/Release 差异） | `mem0mem.h:99-108` `constexpr int MEM_NO_MANS_LAND` | cppreference: constexpr |
| ⭐⭐⭐ | **`placement new`** | 看不懂 Buffer Pool page 如何在已有内存上构造对象 | `buf0buf.cc` 中的 page 初始化 | Stroustrup §11.2.4 |
| ⭐⭐⭐ | **`explicit` / `noexcept` / `[[maybe_unused]]`** | 不是阻塞项，但不认识会减慢阅读 | 遍布全项目 | cppreference 逐个查 |
| ⭐⭐ | **variadic template / `std::forward`** | `ut::new_<T>(args...)` 的完美转发 | `ut0new.h` 中的 `new_` 函数模板 | cppreference: perfect forwarding |

---

#### 4.0.4 第三域：设计模式（决定了你"看不看得懂代码组织"）

详见本章 4.1~4.5 节。最重要的三个：**Strategy**（PSI vtable）、**Template Method**（start/do/end）、**Observer**（Delegate）。

---

#### 4.0.5 第四域：操作系统基础（决定了你"看得懂底层行为"）

| 你必须理解的 OS 概念 | 为什么必须 | MySQL 中的典型例子 | 推荐资料 |
|---------------------|----------|-------------------|---------|
| **虚拟内存：`mmap` / `munmap` / `mprotect`** | 不懂就看不懂 Buffer Pool 为什么用 `mmap` 而不是 `malloc`，以及 `large_page_alloc` | `large_page_alloc-linux.h:mmap(MAP_HUGETLB)` | APUE §14.8; Linux man mmap |
| **Huge Page（大页）**：2MB/1GB 页 vs 4KB 页 | 不懂就看不懂为什么 BP 要单独走 huge page 路径 | `large_page_alloc` 强制 2MB 对齐 | Linux kernel doc: hugetlbpage |
| **TLB（Translation Lookaside Buffer）** | 不懂 huge page 就不知道为什么"减少 TLB miss" | BP 使用大页后 TLB 条目减少 512 倍 | CSAPP §9.6.2 |
| **Cache Line（缓存行）**：64 字节 | 不懂就看不懂 `ut::cacheline_aligned`、`PFS_cacheline_atomic_uint32` | PFS event 结构体中 `alignas(64)` 消除 false sharing | CSAPP §6.4 |
| **False Sharing（伪共享）** | 不懂就看不懂为什么结构体里插 `char padding[60]` | PFS 的 `PFS_cacheline_atomic_uint32` 保证每个计数器独占一个 cache line | 搜索 "false sharing cache line padding" |
| **Futex vs Pthread Mutex** | 不懂就看不懂 `mysql_mutex_t` 的底层实现选择 | `include/my_mutex.h` 中的 `native_mutex_t` 在不同平台的实现 | Linux man futex(7); pthread_mutex_lock(3) |

---

#### 4.0.6 第五域：并发与内存模型（决定了你"能理解多线程正确性"）

| 你必须理解的 | 为什么必须 | MySQL 中的典型例子 | 推荐资料 |
|-------------|----------|-------------------|---------|
| **mutex vs rwlock 的语义差异** | PFS 单独为两类 lock 设计了不同的 event 类型 | `PFS_events_waits` 中的 `WAIT_CLASS_MUTEX` vs `WAIT_CLASS_RWLOCK` | 任何并发编程教材 |
| **spinlock 与 blocking lock 的选择** | 不懂就看不懂 `replication_optimize_for_static_plugin_config` 为什么切换锁类型 | `Delegate` 中根据配置选择 `mysql_rwlock_t` 或 `Shared_spin_lock` | `rpl_handler.h:175-226` |
| **死锁的四个必要条件 + 锁顺序** | 不懂就看不懂 `MEM_HEAP_BTR_SEARCH` 的设计 | 持有 AHI latch 时不能从 BP 分配页面（防止递归锁）→ 预留在先 | 本文 2.2.3 节；Coffman 1971 "deadlock conditions" |
| **CAS（Compare-And-Swap）语义** | 不懂就看不懂 `compare_exchange_strong` | `free_block_for_heap.compare_exchange_strong(expected, block)` | cppreference: atomic::compare_exchange |
| **C++11 Memory Order（内存序）** | 不懂就看不懂 PFS `m_enabled` 为什么有时用 `relaxed` 有时用 `acquire/release` | PFS consumer flags：`m_enabled.load(std::memory_order_relaxed)` | C++ Concurrency in Action §5.3 |
| **Lock-free 编程基础** | `free_block_ptr` 的原子操作设计 | `heap->free_block_ptr->load()` + `store(nullptr)` 的无锁模式 | Herlihy & Shavit, "The Art of Multiprocessor Programming" |

---

#### 4.0.7 第六域：计算机体系结构（决定了你"理解为什么这么设计"）

| 你必须理解的 | 为什么必须 | MySQL 中的典型例子 | 推荐资料 |
|-------------|----------|-------------------|---------|
| **Branch Predictor（分支预测器）** | PFS 零开销关闭的核心机制 | `if (m_psi->m_enabled)` 在关闭时分支预测 100% 命中 | CSAPP §4.5; Smith ISCA 1981 |
| **Cache Hierarchy（L1/L2/L3/主存）** | 解释为什么 cache line padding 有效 | PFS 聚合表中的 `PFS_cacheline_atomic_uint32` | CSAPP §6 |
| **Pipeline（指令流水线）** | 解释为什么分支预测错误代价大 | PFS 开启时 branch miss 的开销来源 | CSAPP §4.4 |
| **NUMA（非一致性内存访问）** | 解释 Buffer Pool 多实例设置 | `innodb_buffer_pool_instances`：每个 NUMA 节点一个 BP 实例 | CSAPP §9.5 |

---

#### 4.0.8 第七域：工程工具链（决定了你"怎么高效读"）

**不掌握这些工具，读源码效率除以 10。**

| 工具 | 用途 | 必须掌握的命令/操作 | MySQL 源码场景 |
|------|------|-------------------|--------------|
| **grep / ripgrep** | 搜索符号定义和引用 | `rg "mem_heap_create\(" --type cpp` | 找一个函数的所有调用点 |
| **ctags / cscope** | 跳转到符号定义 | `ctags -R storage/innobase/` 后 vim 中 `Ctrl-]` 跳转 | 从 `mem_heap_alloc` 跳转到其定义 |
| **gcc -E** | 查看宏展开结果 | `gcc -E -Iinclude file.c` | **理解 PFS 编织最关键的命令**：展开 `mysql_mutex_lock(&foo)` 看三层宏 |
| **gdb** | 运行时调试 | `b mem_heap_create_block`, `p type`, `p len`, `bt` | 在三种 type 分支设断点，观察不同调用路径 |
| **perf** | 性能采样 | `perf record -g`, `perf report` | 了解热点在哪，决定先读哪段代码 |
| **git blame / log** | 追溯代码历史 | `git blame mem0mem.h \| grep "BTR_SEARCH"` | 找到 `MEM_HEAP_BTR_SEARCH` 是谁在什么 commit 中引入的、commit message 说了什么 |
| **ASAN / Valgrind** | 运行时内存错误检测 | `-DWITH_ASAN=ON` cmake 选项 | Debug 模式下 `MEM_NO_MANS_LAND` + ASAN 双重检测越界 |
| **CMake 选项** | 理解编译配置 | `cmake -LH \| grep -i psi` | 看看哪些 PFS 接口被编译进去了 |
| **clang-format** | 理解代码风格 | `.clang-format` 文件 | MySQL 使用 Allman 风格（大括号独立一行），知道这点就不会觉得格式奇怪 |

**一个具体的效率技巧：用 gcc -E 展开宏的完整示例**

```bash
# 创建最小测试文件
cat > /tmp/test_mutex.c << 'EOF'
#include "mysql/psi/mysql_mutex.h"
void test() {
    mysql_mutex_t m;
    mysql_mutex_lock(&m);
}
EOF

# 展开宏（仅预处理，不编译）
gcc -E -I include -I include/mysql -I build/include \
    /tmp/test_mutex.c 2>/dev/null | grep -A 30 "mysql_mutex_lock"

# 你会看到 mysql_mutex_lock(&m) 展开为 inline_mysql_mutex_lock(&m, "/tmp/test.c", 3)
# inline_mysql_mutex_lock 又包含 start_mutex_wait → my_mutex_lock → end_mutex_wait
```

**这就是理解 PFS 编织的终极技巧。** 任何嵌套宏，gcc -E 一下全清楚。

---

#### 4.0.9 推荐学习路线（分阶段执行，不要一次性补完）

**阶段一：能跑通一个调用链（第 1 周）**
```
1. gcc -E 展开 mysql_mutex_lock → 理解三层宏
2. gdb 在 mem_heap_create_block 设断点 → 看三种 type 分别走哪个分支
3. ctags 生成索引 → 能在 vim/vscode 里跳转定义
```
**检验标准**：能从 `mem_heap_alloc(heap, 100)` 一路追踪到 `ut::malloc_withkey` 或 `buf_block_alloc`。

**阶段二：理解设计层面（第 2-3 周）**
```
4. 学 Strategy 模式 → 去读 PSI vtable
5. 学 Template Method → 去读 start/end 三部曲
6. 学 RAII → 去读 Scoped_heap
7. 学 Observer → 去读 Delegate+FOREACH_OBSERVER
```
**检验标准**：看到任意一个 PFS 埋点，能说出用了哪几个设计模式。

**阶段三：理解底层原理（第 4-5 周）**
```
8. 学虚拟内存/mmap → 去读 large_page_alloc
9. 学 CAS/memory_order → 去读 free_block_ptr 的原子操作
10. 学分支预测/cache line → 理解 m_enabled 为什么零开销
```
**检验标准**：能解释 PFS 关闭时 `if (m_psi->m_enabled)` 为什么只需 ~1 cycle。

**阶段四：使用工具验证理解（持续）**
```
11. git blame 追溯关键设计引入的 commit
12. perf 采样看真正的热点路径
13. gdb 在 event 记录点设断点，手工验证 ring buffer 写入
```
**检验标准**：能写一个简单的 SQL 查询 PFS 表，并解释背后发生了什么。

---

#### 4.0.10 "读不懂"的自诊断流程

当你遇到一段看不懂的代码时，按以下顺序排查：

```
看不懂代码
    │
    ├─ 是语法层面（不认识某个 C/C++ 关键字/运算符）？
    │   → 查 cppreference.com
    │
    ├─ 是宏展开后看不懂（一堆嵌套的 #define）？
    │   → gcc -E 展开看实际代码
    │
    ├─ 是不知道这段代码"为什么存在"（看懂了语法但不理解设计）？
    │   → 查本章 4.1-4.4 节，看是哪种设计模式
    │
    ├─ 是不知道底层 OS/CPU 行为？
    │   → 查 4.0.5-4.0.7 节，补对应 OS/并发/体系结构知识
    │
    ├─ 是不知道业务动机（为什么需要这个功能）？
    │   → git log + git blame 看 commit message
    │   → 本文第一~第三章找对应理论解释
    │
    └─ 都不是？
        → 很可能是一段历史遗留代码或为了兼容性的 workaround
        → git blame 看是谁写的、什么时间写的、commit message 怎么说的
```

---

#### 4.0.11 必读参考资料清单（按需取用）

| 类别 | 书籍/资料 | 关联章节 | 优先级 |
|------|---------|---------|-------|
| 设计模式入门 | 《Head First Design Patterns》(Freeman, 2004) | 4.1-4.4 | ⭐⭐⭐⭐⭐ 最推荐入门 |
| 设计模式参考 | 《Design Patterns》(GoF, 1994) | 4.1-4.4 | ⭐⭐⭐ 参考手册 |
| C 预处理器 | GCC Manual §3 (Macros) | 4.0.3 | ⭐⭐⭐⭐⭐ 必读 |
| C++ 核心 | 《The C++ Programming Language》4th Ed. (Stroustrup) §5 RAII, §34 unique_ptr, §28 template | 4.0.3 | ⭐⭐⭐⭐ |
| C++ 并发 | 《C++ Concurrency in Action》(Williams, 2nd Ed.) §5 内存模型, §7 无锁 | 4.0.6 | ⭐⭐⭐⭐ |
| 体系+OS | 《Computer Systems: A Programmer's Perspective》(CSAPP) §4 处理器, §6 存储器, §9 虚拟内存 | 4.0.5, 4.0.7 | ⭐⭐⭐⭐⭐ 一本覆盖 50% |
| OS 深入 | 《Advanced Programming in the UNIX Environment》(APUE) §14 mmap | 4.0.5 | ⭐⭐⭐ |
| 分配器理论 | Knuth TAOCP Vol.1 §2.5; Knowlton CACM 1965; Bonwick USENIX 1994 | 第二章 | ⭐⭐⭐ |
| MySQL 内幕 | 《MySQL 技术内幕: InnoDB 存储引擎》(姜承尧, 第2版) | 全局 | ⭐⭐⭐⭐ 中文首选 |
| MySQL 源码 | 官方 Developer Guide: storage/perfschema/README | 第一、三章 | ⭐⭐⭐⭐ |

---

### 4.1 PFS 模块涉及的设计模式

#### 4.1.1 Strategy（策略模式）—— PSI vtable 函数指针表

**这是 MySQL PFS 中最核心的设计模式。** 如果只学一个模式来理解 PFS，就学这个。

**定义（GoF）：** 定义一系列算法，把它们一个个封装起来，并且使它们可互相替换。策略模式使得算法可独立于使用它的客户而变化。

**中文关键词检索：** "策略模式 C++ 函数指针 vtable" / "Strategy Pattern C vtable" / "策略模式 设计模式"

**MySQL 中的实现：** PFS 通过 `PSI_mutex_service_v1` 结构体（本质是一个手动实现的 vtable）实现策略模式：

```cpp
// include/mysql/psi/psi_mutex.h:66-79
struct PSI_mutex_service_v1 {
  register_mutex_v1_t register_mutex;       // 注册策略
  init_mutex_v1_t init_mutex;               // 初始化策略
  destroy_mutex_v1_t destroy_mutex;         // 销毁策略
  start_mutex_wait_v1_t start_mutex_wait;   // 开始等待策略
  end_mutex_wait_v1_t end_mutex_wait;       // 结束等待策略
  unlock_mutex_v1_t unlock_mutex;           // 解锁策略
};
```

**三种策略切换（本质就是 Strategy 模式的核心：运行时可替换算法）：**

| 编译配置 | 策略 | `start_mutex_wait` 指向 | 效果 |
|---------|------|------------------------|------|
| `-DDISABLE_PSI_MUTEX` | 空策略（Null Object Pattern） | 不存在 | 编译期完全剔除，零开销 |
| 编译 PFS 但运行时关闭 | 空策略 | `nullptr`，通过 `if (m_psi != NULL)` 短路 | 一次分支预测，近乎零开销 |
| 编译 PFS 且运行时开启 | 完整记录策略 | `pfs_start_mutex_wait_v1` | 记录 time_start, source_file 到 ring buffer |

```cpp
// 策略切换的本质：同一个宏，编译出不同代码路径
#ifdef HAVE_PSI_MUTEX_INTERFACE
  // 策略A：通过 psi_mutex_service -> vtable 间接调用
  locker = psi_mutex_service->start_mutex_wait(&state, that->m_psi, ...);
#else
  // 策略B：完全不存在（空策略）
#endif
```

**理论来源：** Gamma, Helm, Johnson, Vlissides (GoF), *"Design Patterns: Elements of Reusable Object-Oriented Software"*, Addison-Wesley, 1994. 策略模式是 GoF 23 种中最常用的行为型模式之一。在 C 语言中，策略模式通过**函数指针表（vtable）**实现，这与 C++ 虚函数表在底层是同构的。

#### 4.1.2 Template Method（模板方法模式）—— start/do/end 三部曲

**定义（GoF）：** 定义一个操作中的算法骨架，而将一些步骤延迟到子类中。模板方法使得子类可以不改变算法结构即可重定义算法的某些特定步骤。

**MySQL 中的实现：** 所有 PFS instrumentation 点都遵循固定的三步模板。注意这里不是 OOP 的虚函数继承，而是**通过 C 预处理器宏展开实现的编译期模板方法**：

```
(a) start_xxx_wait()    ← 模板步骤1：记录开始时间
(b) 执行真正的操作       ← 模板步骤2：业务逻辑（可变部分，即 GoF 中的"原语操作 primitive operation"）
(c) end_xxx_wait()      ← 模板步骤3：记录结束时间，写入 ring buffer
```

```cpp
// 模板方法的 C 语言实现：宏定义了骨架，调用者填充步骤2
// 以 mutex lock 为例——这个模式在 mutex/rwlock/cond/socket/file 中完全一致：
locker = PSI_MUTEX_CALL(start_mutex_wait)(&state, that->m_psi, ...);  // 步骤1
result = my_mutex_lock(&that->m_mutex);                                 // 步骤2 ← 可变部分
if (locker != nullptr)
  PSI_MUTEX_CALL(end_mutex_wait)(locker, result);                      // 步骤3
```

**为什么 C 语言能用宏实现模板方法？** 因为宏展开发生在编译期（C 预处理器），展开后的代码骨架完全确定——步骤1→步骤2→步骤3的顺序由宏保证，而步骤2的具体内容由调用者提供。这与 GoF 模板方法的精神一致："Don't call us, we'll call you"（好莱坞原则）。

**中文关键词检索：** "模板方法模式" / "Template Method Pattern" / "好莱坞原则"

#### 4.1.3 Facade（外观模式）—— mysql_mutex_t 的统一门面

**定义（GoF）：** 为子系统中的一组接口提供一个一致的界面，Facade 模式定义了一个高层接口，使得这一子系统更加容易使用。

**MySQL 中的实现：** `mysql_mutex_t` 对外暴露统一的 `mysql_mutex_lock()` / `mysql_mutex_unlock()` 接口，内部隐藏了三个子系统：

```cpp
// include/mysql/components/services/bits/mysql_mutex_bits.h:47-55
struct mysql_mutex_t {
  my_mutex_t m_mutex;                // 子系统1：平台互斥锁（pthread/Windows）
  struct PSI_mutex *m_psi{nullptr};  // 子系统2：PFS 插桩钩子
};
// 隐藏的子系统3：SAFE_MUTEX debug 模式下的死锁检测（编译时决定）
```

用户只需调用 `mysql_mutex_lock(&foo)`，无需关心：
- 底层是 `pthread_mutex_lock` 还是 `EnterCriticalSection`
- PFS 是否开启、如何记录 event
- Debug 模式下是否在做死锁检测

**这是阅读源码时第一个要理解的门面：** 当你看到 `mysql_mutex_lock`，要知道它不是一个简单的锁操作，而是一个**门面**——背后至少组合了 2~3 个子系统的行为。

**中文关键词检索：** "外观模式" / "Facade Pattern" / "门面模式"

#### 4.1.4 Bridge（桥接模式）—— PSI ABI 抽象与实现分离

**定义（GoF）：** 将抽象部分与它的实现部分分离，使它们都可以独立地变化。

**MySQL 中的实现：** PSI 接口定义（`psi_mutex.h` 中的函数指针类型）与 PFS 实现（`pfs.cc` 中的 `pfs_start_mutex_wait_v1`）完全分离。同一个接口可以有多个实现：

```cpp
// 桥的一侧：抽象接口（psi_mutex.h）——只定义"做什么"
typedef struct PSI_mutex_locker *(*start_mutex_wait_v1_t)(...);

// 桥的另一侧：具体实现（pfs.cc）——定义"怎么做"
PSI_mutex_locker *pfs_start_mutex_wait_v1(...) { /* 记录到 ring buffer */ }

// 桥的组装：在 PFS 初始化时完成绑定
psi_mutex_service->start_mutex_wait = pfs_start_mutex_wait_v1;
```

**Bridge 与 Strategy 的区别（经常混淆）：**
| 维度 | Bridge（桥接） | Strategy（策略） |
|------|--------------|-----------------|
| 意图 | 分离抽象与实现，让两者独立演化 | 封装可互换的算法族 |
| 侧重点 | 结构层面——"接口和实现分开编译" | 行为层面——"运行时切换算法" |
| MySQL 例子 | PSI ABI 版本1/2 共存 | m_psi->start_mutex_wait 指向不同函数 |

**中文关键词检索：** "桥接模式" / "Bridge Pattern" / "handle-body idiom" / "Pimpl idiom"

---

### 4.2 内存管理模块涉及的设计模式

#### 4.2.1 RAII（资源获取即初始化）—— Scoped_heap

**这不是 GoF 23 种模式之一，但属于 C++ 最重要的惯用法（idiom），优先级甚至高于 GoF 模式。**

```cpp
// mem0mem.h:440-507
struct Scoped_heap {
  struct mem_heap_free_functor {
    void operator()(mem_heap_t *heap) { mem_heap_free(heap); }
  };
  using Ptr = std::unique_ptr<mem_heap_t, mem_heap_free_functor>;
  Ptr m_ptr{};

  // 构造 = 获取资源
  Scoped_heap(size_t n, ut::Location location) noexcept
      : m_ptr(mem_heap_create(n, location, MEM_HEAP_DYNAMIC)) {}

  // 析构 = 自动释放资源（无需手动调用 mem_heap_free）
  ~Scoped_heap() = default;
};
```

**为什么需要 RAII？** C 风格的 `mem_heap_t` 需要手动调用 `mem_heap_free()`，在以下场景极易泄漏：
- 提前 `return` 的多个退出路径
- C++ 异常抛出（`ut_a` 断言失败会抛异常）
- 多层嵌套调用中某一层失败需要回滚

`Scoped_heap` 保证无论函数如何退出（正常 return、异常抛出、断言失败），heap 都会被释放。**在 InnoDB 的 C++ 代码中，这是内存安全的基石。**

**理论来源：** Bjarne Stroustrup, *"The C++ Programming Language"*, 第 4 版, 2013. RAII 是 C++ 最重要的资源管理思想，比 GoF 模式更基础、更常用。

**中文关键词检索：** "RAII C++" / "资源获取即初始化" / "scope-based resource management"

#### 4.2.2 Adapter（适配器模式）—— Mem_root_allocator 适配 STL

**定义（GoF）：** 将一个类的接口转换成客户希望的另外一个接口。Adapter 模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

**MySQL 中的实现：** `Mem_root_allocator<T>` 将 MEM_ROOT（arena allocator，有自己的 `AllocBlock()` / `Alloc()` 接口）适配为 STL 标准的 `std::allocator<T>` 接口：

```cpp
// 适配前：MEM_ROOT 无法直接用于 std::vector
// 适配后：
std::vector<Item *, Mem_root_allocator<Item *>> 
    items(Mem_root_allocator<Item *>(thd->mem_root));
// 这个 vector 的所有内存都从 MEM_ROOT 分配，而非 new/delete
```

**适配了什么？** STL 容器期望 `allocate()` / `deallocate()` 接口，MEM_ROOT 提供的是 `Alloc()` / `Clear()`，`Mem_root_allocator` 充当了中间的翻译层。注意 MEM_ROOT 不支持单块 `deallocate`，所以适配器在这个方法上做了"空操作"处理——这是适配器模式中常见的"退化适配"。

**中文关键词检索：** "适配器模式" / "Adapter Pattern" / "STL allocator 自定义"

#### 4.2.3 Strategy（策略模式）—— mem_heap 三种 type 的行为分支

**mysql 中的实现：** mem_heap 通过 `type` 参数选择不同的内存来源策略——这是策略模式在 C 语言中不用 vtable 的简化实现：

```cpp
constexpr uint32_t MEM_HEAP_DYNAMIC    = 0;  // 策略A：走 ut::malloc
constexpr uint32_t MEM_HEAP_BUFFER     = 1;  // 策略B：大块走 Buffer Pool
constexpr uint32_t MEM_HEAP_BTR_SEARCH = 2;  // 策略C：死锁安全模式（标志位）

// 在 mem_heap_create_block 中根据 type 选择分配路径：
if (type == MEM_HEAP_DYNAMIC || len < UNIV_PAGE_SIZE / 2) {
  // 策略A/B 的小块路径：ut::malloc_withkey → jemalloc/glibc
}
if (type == MEM_HEAP_BTR_SEARCH) {
  // 策略C：从预留 free_block_ptr 原子取，失败返回 NULL 而非崩溃
}
```

虽然用 `if/else` 而非虚函数实现，但这本质上是策略模式——**同一接口（mem_heap_alloc），不同策略（DYNAMIC/BUFFER/BTR_SEARCH），编译时通过 type 常量选择**。为什么不用 vtable？因为 type 在 heap 创建时就确定且不变，用常量比函数指针少一次间接寻址。

#### 4.2.4 Decorator（装饰器模式）—— ut::malloc_withkey

**定义（GoF）：** 动态地给一个对象添加一些额外的职责。就增加功能来说，Decorator 模式比生成子类更灵活。

**MySQL 中的实现：** `ut::malloc_withkey` 是对底层 `malloc`（jemalloc 或 glibc）的装饰——在不改变 `malloc` 接口的前提下，增加了 PFS 统计功能。这不是面向对象的装饰器，而是**函数级装饰（function-level decoration）：**

```
调用者 → ut::malloc_withkey(size, key) 
           ├── 下层 malloc(size)      ← 原始功能完全不变（被装饰者）
           └── PFS 报告 (key, size)   ← 装饰层增加的功能
```

在 OOP 中，装饰器通常通过组合 + 委托实现；在 C 中，通过包装函数实现。效果相同：**不修改底层代码，不改变接口，透明地增加功能。**

---

### 4.3 插件系统涉及的设计模式

#### 4.3.1 Microkernel（微内核模式）—— MySQL Plugin 体系

**这不是 GoF 23 之一，属于架构模式（Architectural Pattern），优先级在大型系统阅读中极高。**

MySQL Server 本身就是微内核 + 插件体系：
- **核心（Microkernel）：** SQL 解析、优化器、执行器、连接管理、权限系统
- **插件（Plugins）：** 存储引擎（InnoDB/MyISAM/Temptable）、PFS、全文搜索、复制、认证、半同步等

```cpp
// 插件的标准声明（mysql_declare_plugin 宏）
mysql_declare_plugin(perfschema)
{
  MYSQL_STORAGE_ENGINE_PLUGIN,
  &pfs_storage_engine,
  "PERFORMANCE_SCHEMA",
  PLUGIN_AUTHOR_ORACLE,
  "Performance Schema",
  PLUGIN_LICENSE_GPL,
  pfs_init_func,              // 插件初始化（构造函数）
  nullptr,                    // check_uninstall = nullptr → Mandatory Plugin
  pfs_done_func,              // 插件清理（析构函数）
  // ...
}
```

**PFS 是"强制内置插件（Mandatory Built-in Plugin）"**，与其他可选插件的关键区别：
- `check_uninstall = nullptr`：禁止运行时卸载
- 只能通过启动参数 `performance_schema=OFF` 禁用

**理论来源：** Buschmann, Meunier, Rohnert, Sommerlad, Stal, *"Pattern-Oriented Software Architecture (POSA)"* Vol.1, Wiley, 1996. 正式命名 Microkernel 模式。

**中文关键词检索：** "微内核模式" / "Microkernel Pattern" / "插件架构 plugin architecture"

#### 4.3.2 Observer（观察者模式）—— Replication Delegate

**这是 MySQL 复制系统中最重要的行为模式。**

```cpp
// rpl_handler.h: Delegate 是被观察者（Subject），维护 observer 列表
class Delegate {
  Observer_info_list observer_info_list;  // 观察者列表
  int add_observer(void *observer, st_plugin_int *plugin);     // 注册
  int remove_observer(void *observer);                          // 注销

  // 核心：遍历所有 observer 调用回调
  // FOREACH_OBSERVER 宏展开为标准的 Observer 通知循环
};

// 具体观察者实现——每个 observer 填充自己的回调函数指针表：
Binlog_relay_IO_observer binlog_IO_observer = {
  sizeof(Binlog_relay_IO_observer),
  group_replication_thread_start,    // 回调1
  group_replication_thread_stop,     // 回调2
  group_replication_applier_start,   // 回调3
  // ... 更多回调
};
```

**经典 Observer 五要素在此的体现：**
1. **Subject（主题）**: `Delegate` 类
2. **Observer（观察者接口）**: `Binlog_relay_IO_observer` 等函数指针结构体
3. **ConcreteObserver（具体观察者）**: `group_replication_*` 函数实现
4. **attach/detach**: `add_observer` / `remove_observer`
5. **notify**: `FOREACH_OBSERVER` 宏展开的遍历调用

**中文关键词检索：** "观察者模式" / "Observer Pattern" / "发布订阅模式" / "事件监听"

#### 4.3.3 Factory Method（工厂方法模式）—— PSI ABI 版本工厂

```cpp
struct PSI_mutex_bootstrap {
  /**
    ABI interface finder——工厂方法
    Calling this method with an interface version number returns either
    an instance of the ABI for this version, or NULL.
  */
  void *(*get_interface)(int version);  // 工厂方法签名
};

// 调用：get_interface(PSI_MUTEX_VERSION_1) → 返回 PSI_mutex_service_v1*
//       get_interface(PSI_MUTEX_VERSION_2) → 返回 PSI_mutex_service_v2*
```

**为什么需要工厂？** PFS 需要同时支持多个 ABI 版本（v1, v2, ...），让老插件和新插件共存。工厂方法封装了"根据版本号创建对应实现"的决策逻辑，调用者不需要知道内部有多少个版本、每个版本对应的结构体是什么。

---

### 4.4 设计模式与 MySQL 源码完整对照表

| 设计模式 | 中文 | GoF 分类 | MySQL 中的位置 | 一句话说明 |
|---------|------|---------|---------------|-----------|
| **Strategy** | 策略 | 行为型 | PSI vtable、mem_heap type 分支 | 同一接口，多种算法可替换 |
| **Template Method** | 模板方法 | 行为型 | `start_x → do_x → end_x` 宏展开 | 固定骨架，可变步骤 |
| **Observer** | 观察者 | 行为型 | `Delegate` + `xxxx_observer` 回调 | 解耦事件源与处理器 |
| **Factory Method** | 工厂方法 | 创建型 | `PSI_bootstrap::get_interface()` | 根据版本号创建对应 ABI |
| **Facade** | 外观 | 结构型 | `mysql_mutex_t` 统一接口 | 隐藏多子系统的复杂性 |
| **Bridge** | 桥接 | 结构型 | PSI 抽象 ↔ PFS 具体实现 | 抽象与实现独立演化 |
| **Adapter** | 适配器 | 结构型 | `Mem_root_allocator<T>` | Arena 分配器 → STL 接口 |
| **Decorator** | 装饰器 | 结构型 | `ut::malloc_withkey` | malloc + PFS 统计 |
| **Microkernel** | 微内核 | 架构¹ | MySQL Plugin 体系 | 核心最小化，功能插件化 |
| **RAII** | RAII | C++ 惯用法² | `Scoped_heap` | 自动管理资源生命周期 |

> ¹ Microkernel 不属于 GoF 23，出自 POSA（Pattern-Oriented Software Architecture）系列。  
> ² RAII 不属于 GoF 23，是 C++ 最重要的资源管理惯用法（idiom），重要性不亚于任何 GoF 模式。

### 4.5 推荐学习资源与路线

| 顺序 | 资源 | 说明 |
|------|------|------|
| 1 | 《Head First Design Patterns》(Freeman, 2004) | **最推荐的入门书**——图文并茂，用 Java 示例但思想跨语言。先看 Strategy、Observer、Template Method、Factory、Facade、Adapter、Bridge 这几章 |
| 2 | 中文博客：搜索 "C 语言实现设计模式" | Strategy 在 C 中用函数指针实现、Observer 在 C 中用回调函数实现——MySQL 正是这样做的 |
| 3 | GoF《Design Patterns》(1994) | 经典但抽象，适合作为参考手册而非通读 |
| 4 | POSA Vol.1 (Buschmann, 1996) | 架构模式：Microkernel 的权威定义 |
| 5 | 《API Design for C++》(Reddy, 2011) | Pimpl、Handle-Body、Factory 在 C++ 中的工业实践 |

**针对 MySQL 源码的阅读建议：**
- 先学 Strategy + Template Method → 立刻去读 `mysql_mutex.h`，你会发现所有埋点宏都是这两个模式的组合
- 再学 RAII → 去读 `Scoped_heap`，理解为什么 C++ 代码中看不到 `mem_heap_free` 调用
- 然后学 Observer → 去读 `rpl_handler.cc` 的 `FOREACH_OBSERVER` 宏
- 最后学 Microkernel → 理解 MySQL 为什么是"插件海洋"

---

## 第五章：核心术语中英文对照

### 5.1 可观测性（Observability）

| 英文 | 中文标准译法 | 说明 |
|------|-------------|------|
| **Performance Schema (PFS)** | 性能模式 / 性能架构 | MySQL 内建框架 |
| **PSI** (Performance Schema Instrumentation) | 性能模式埋点/插桩接口 | PFS 的 instrumentation API 命名空间 |
| **Instrumentation** | 插桩 / 埋点 | 在代码中插入观测代码的手段 |
| **Probing** | 探针 | 与 instrumentation 近义，更强调"探测"语义 |
| **Event** | 事件 | 单次操作的完整记录，有起止时间 |
| **Trace** | 追踪 / 链路 | 有因果关系的 event 序列 |
| **Metric** | 指标 | 聚合统计量（COUNT, SUM, AVG, MIN, MAX） |
| **Profile** | 性能画像 / 性能分析 | 按维度聚合的统计视图，非采样 |
| **Sampling** | 采样 | 按固定频率中断读取状态，PFS 不用 |
| **Weaving** | 编织 / 织入 | AOP 中把 Advice 注入 Join Point 的过程 |
| **AOP** (Aspect-Oriented Programming) | 面向切面编程 | Kiczales, ECOOP 1997 |
| **DTrace** | 动态追踪 | Sun Microsystems, USENIX ATC 2004 |
| **Ring Buffer** | 环形缓冲区 | PFS events 存储结构 |
| **NOP** (No Operation) | 空操作指令 | x86: 0x90，DTrace 零开销探针的底层机制 |
| **Join Point** | 连接点 / 切入点 | AOP 中可插入 Advice 的位置 |
| **Advice** | 通知 / 增强 | AOP 中在 Join Point 插入的逻辑 |
| **Cross-cutting Concern** | 横切关注点 | 跨越多个模块的关注点（如日志、统计） |
| **Static Analysis** | 静态分析 | 不运行程序，通过分析源码/二进制发现缺陷（如 clang-tidy、Coverity） |
| **Dynamic Analysis** | 动态分析 | 程序运行时监测实际行为（如 Valgrind、ASan、PFS） |
| **Dynamic Instrumentation** | 动态插桩 | 运行时在程序中插入观测代码。PFS 属于此类（编织在编译期，采集在运行时） |
| **Dynamic Tracing** | 动态追踪 | DTrace 开创的范式，PFS 在关系数据库领域的实现 |
| **Compile-time Weaving** | 编译期编织 | AOP 中在编译时注入 Aspect 代码，C 预处理器宏是典型手段 |
| **Zero-cost Probe** | 零开销探针 | DTrace 的设计原则：不启用时探针是 NOP 指令，零开销 |
| **Microkernel Pattern** | 微内核模式 | POSA Vol.1 (1996) 正式命名：核心 + 插件扩展。MySQL Server = 核心，PFS = 插件 |
| **Information Hiding** | 信息隐藏 | Parnas (CACM 1972)：模块通过稳定接口暴露功能，内部实现可独立演化 |
| **Plugin Architecture** | 插件架构 | Microkernel Pattern 的具体形式，PFS 通过 PSI ABI 函数指针表实现 |

### 5.2 内存管理（Memory Management）

| 英文 | 中文标准译法 | 说明 |
|------|-------------|------|
| **Bump Allocator** | 指针碰撞分配器 / 线性分配器 | 推 `free` 指针，O(1) |
| **Arena Allocator** | 区域分配器 / 竞技场分配器 | 从一大块中逐次切分 |
| **Buddy Allocator** | 伙伴分配器 | 2^n 分裂/合并 |
| **Slab Allocator** | Slab 分配器 / 板层分配器 | 同大小对象复用槽位 |
| **Segregated-fit Allocator** | 分级适配分配器 | 按 size class 分 bin |
| **Free-list Allocator** | 空闲链表分配器 | 维护空闲块链表 |
| **Stack Allocator** | 栈式分配器 | LIFO，只能释放最后分配的块 |
| **Canary Byte / Guard Byte** | 金丝雀字节 / 哨兵字节 / 守卫字节 | 检测越界写入的魔数 |
| **Red Zone** | 红区 | ASan/Valgrind 的边界检测 |
| **External Fragmentation** | 外部碎片 | malloc 长期运行产生 |
| **Size Class** | 大小分级 / 大小类 | 将请求向上取整到预设档位，同档共享 free list，释放立即可复用。**存在于 jemalloc 层**（buf_buddy 的 2^n 服务于分裂/合并，不是对象缓存意义上的 size class） |
| **Huge Page** | 大页 / 巨页 | 2MB/1GB 页，减少 TLB miss |
| **jemalloc** | — | 不翻译，直接用英文 |
| **Buffer Pool (BP)** | 缓冲池 | InnoDB 核心缓存，也是 mem_heap 大块的来源 |

### 5.3 设计模式（Design Patterns）

| 英文 | 中文标准译法 | GoF 分类 | 说明 |
|------|-------------|---------|------|
| **Design Pattern** | 设计模式 | — | 对软件设计中反复出现的问题的通用可复用解决方案。GoF 1994 定义了 23 种经典模式 |
| **Strategy Pattern** | 策略模式 | 行为型 | 定义一系列算法并使其可互换。MySQL 中 PSI vtable / mem_heap type 分支是其 C 语言版本 |
| **Template Method** | 模板方法模式 | 行为型 | 定义算法骨架，延迟步骤到子类。PFS 的 start→do→end 三部曲是宏展开的编译期模板方法 |
| **Observer / Publish-Subscribe** | 观察者模式 / 发布-订阅模式 | 行为型 | 定义一对多依赖，当对象状态变化时通知所有依赖者。MySQL 复制 Delegate 是其实现 |
| **Factory Method** | 工厂方法模式 | 创建型 | 定义创建对象的接口，让子类决定实例化哪个类。PSI ABI `get_interface(version)` 是其实现 |
| **Facade** | 外观模式 / 门面模式 | 结构型 | 为子系统提供统一高层接口。`mysql_mutex_t` 隐藏了平台锁 + PFS + debug 三个子系统 |
| **Bridge** | 桥接模式 | 结构型 | 分离抽象与实现，让两者独立变化。PSI 接口定义与 PFS 实现的分层 |
| **Adapter** | 适配器模式 | 结构型 | 将一个接口转换为另一个接口。`Mem_root_allocator<T>` 适配 arena 分配器为 STL allocator |
| **Decorator** | 装饰器模式 | 结构型 | 动态地为对象添加额外职责。`ut::malloc_withkey` 对 malloc 透明增加 PFS 统计 |
| **RAII** (Resource Acquisition Is Initialization) | 资源获取即初始化 | C++ 惯用法¹ | 将资源生命周期绑定到对象作用域。`Scoped_heap` 通过 `unique_ptr` + 自定义 deleter 实现 |
| **vtable** (Virtual Function Table) | 虚函数表 | 底层机制 | C++ 虚函数 / C 语言策略模式的底层实现机制。函数指针数组 |
| **Deleter / Custom Deleter** | 自定义删除器 | C++ 惯用法 | `std::unique_ptr<T, Deleter>` 的第二个模板参数，用于非标准的资源释放方式 |
| **Inheritance** (白盒复用) | 继承（白盒复用） | 面向对象基础 | 子类继承父类的接口和实现。需了解父类内部细节 |
| **Composition / Delegation** (黑盒复用) | 组合 / 委托（黑盒复用） | 面向对象基础 | 对象持有另一个对象的引用并委托调用。只需知道接口，不需知道内部实现。GoF 提倡"优先使用组合而非继承" |

> ¹ RAII 不属于 GoF 23，由 Bjarne Stroustrup 在《The C++ Programming Language》中提出，是 C++ 独有的、最重要的资源管理惯用法。

#### 设计模式中文检索关键词

| 你想了解 | 用这些关键词搜索 |
|---------|---------------|
| 设计模式入门 | "Head First 设计模式" / "设计模式 通俗易懂" |
| C 语言如何实现设计模式 | "C 语言 设计模式" / "函数指针 策略模式" / "C语言 观察者模式" |
| MySQL 中的设计模式 | "MySQL 源码 设计模式" / "mysql plugin 设计模式" |
| 策略模式详解 | "策略模式 C++" / "Strategy Pattern vtable" |
| C++ RAII | "RAII C++" / "资源获取即初始化" / "unique_ptr 自定义删除器" |
| GoF 23 种 | "GoF 23种设计模式" / "Design Patterns Gamma" |

---

## 第六章：全部学术论文/理论工作索引

### 6.1 可观测性（Observability & Instrumentation）

| 论文标题 | 作者 | 会议/期刊 | 年份 | MySQL 关联 |
|---------|------|----------|------|-----------|
| *On the Criteria To Be Used in Decomposing Systems into Modules* | D. L. Parnas | **CACM** | 1972 | **信息隐藏（Information Hiding）**原则：PFS PSI 接口的设计哲学源头。模块应通过稳定接口暴露功能，内部实现可独立演化 |
| *Pattern-Oriented Software Architecture (POSA) Vol.1* | Buschmann, Meunier, Rohnert, Sommerlad, Stal | Wiley | 1996 | **Microkernel（微内核）模式**正式命名：核心系统 + 插件扩展。MySQL Server 为微核心，PFS 为插件 |
| *Aspect-Oriented Programming* | Kiczales et al. (Xerox PARC) | **ECOOP** | 1997 | PFS 的编译期宏编织设计哲学 |
| *Continuous Profiling: Where Have All the Cycles Gone?* | Anderson et al. | **SOSP** | 1997 | event-based tracing vs sampling 的理论基础 |
| *Dyninst: Efficient and Language-Independent Mobile Programs* | Hollingsworth et al. (U. Maryland) | **PLDI / SC** | 1996- | 运行时动态插桩的奠基 |
| *DTrace: Dynamic Instrumentation of Production Systems* | Cantrill, Shapiro, Leventhal (Sun) | **USENIX ATC** | 2004 | **PFS 的直接灵感**：零开销探针、安全动态激活。奠定了 Dynamic Tracing 范式的理论基础 |
| *Pin: Building Customized Program Analysis Tools with Dynamic Instrumentation* | Luk et al. (Intel/CMU) | **PLDI** | 2005 | 通用动态二进制插桩理论：任意程序可在指令级被 instrument |
| *Valgrind: A Framework for Heavyweight Dynamic Binary Instrumentation* | Nethercote & Seward | **PLDI** | 2007 | 二进制级 heavyweight 动态分析框架（PFS 不用 Valgrind，但同属 Dynamic Instrumentation 大类） |
| *A Study of Branch Prediction Strategies* | Smith | **ISCA** | 1981 | 分支预测理论基础，解释 PFS `m_enabled` 的零开销 |
| *Alternative Implementations of Two-Level Adaptive Branch Prediction* | Yeh & Patt | **ISCA** | 1991 | 现代分支预测器，PFS 关闭时稳定分支预测 100% 命中的理论支撑 |

### 6.2 内存分配器（Memory Allocators）

| 论文标题 | 作者 | 会议/期刊 | 年份 | MySQL 关联 |
|---------|------|----------|------|-----------|
| *A Fast Storage Allocator* | K. C. Knowlton | **CACM** | 1965 | **buf_buddy 的直接理论来源**：二叉伙伴系统 |
| *The Art of Computer Programming, Vol.1* | Donald Knuth | Addison-Wesley | 1968 | buddy system 的形式化分析、bump allocator 描述 |
| *The Slab Allocator: An Object-Caching Kernel Memory Allocator* | Jeff Bonwick (Sun) | **USENIX Summer TC** | 1994 | jemalloc size class 的思想先祖 |
| *Hoard: A Scalable Memory Allocator for Multithreaded Applications* | Berger et al. | **ASPLOS** | 2000 | per-thread heap 消除竞争，jemalloc 的设计源头 |
| *A Scalable Concurrent malloc(3) Implementation for FreeBSD* | Jason Evans | **BSDCan** | 2006 | **jemalloc 论文**，MySQL 底层可选链接 |
| *TCMalloc* | Google | 工业项目 | 2005 | 与 jemalloc 同期竞品，类似 thread-caching |

### 6.3 软件工程与设计模式（Software Engineering & Design Patterns）

| 论文/著作 | 作者 | 会议/出版社 | 年份 | MySQL 关联 |
|---------|------|----------|------|-----------|
| *Design Patterns: Elements of Reusable Object-Oriented Software* (GoF) | Gamma, Helm, Johnson, Vlissides | Addison-Wesley | 1994 | **设计模式圣经**：Strategy、Observer、Template Method、Factory、Facade、Adapter、Bridge、Decorator——MySQL PFS 和内存管理模块使用了其中至少 8 种。Strategy（PSI vtable）、Observer（Delegate）、Template Method（start/do/end）是最核心的三种 |
| *Pattern-Oriented Software Architecture (POSA) Vol.1* | Buschmann, Meunier, Rohnert, Sommerlad, Stal | Wiley | 1996 | **Microkernel（微内核）模式**正式命名：核心系统 + 插件扩展。MySQL Server 为微核心，存储引擎/PFS 为插件 |
| *The C++ Programming Language, 4th Ed.* | Bjarne Stroustrup | Addison-Wesley | 2013 | **RAII 惯用法**的权威出处：资源获取即初始化，C++ 最重要的资源管理思想。Scoped_heap 的 `unique_ptr<T, custom_deleter>` 是其应用 |
| *Refactoring: Improving the Design of Existing Code* | Martin Fowler | Addison-Wesley | 1999 | 重构与设计模式的关系：从"代码坏味道"到设计模式的演进路径 |
| *Head First Design Patterns* | Freeman, Freeman | O'Reilly | 2004 | **最佳入门书**：用 Java 示例图文并茂讲解 GoF 模式，适合首次接触设计模式的读者 |

### 6.4 计算机科学主要领域与顶级会议

#### 系统（Systems）——PFS 与内存管理的核心领域

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **SOSP** | Symposium on Operating Systems Principles | 操作系统最高殿堂，两年一届 | Anderson 1997 持续 profiling |
| **OSDI** | Operating Systems Design and Implementation | 与 SOSP 同级，交替举办 | 系统设计与实现 |
| **USENIX ATC** | USENIX Annual Technical Conference | 通用系统顶会 | **DTrace 2004** 零开销探针 |
| **EuroSys** | European Conference on Computer Systems | 欧洲系统顶会 | jemalloc 2011 |
| **ASPLOS** | Architectural Support for Programming Languages and Operating Systems | 体系+语言+OS交叉 | **Hoard 2000** 多线程 malloc |

#### 编程语言（Programming Languages）——AOP 的学术根基

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **ECOOP** | European Conference on Object-Oriented Programming | 面向对象 | **AOP 1997** Kiczales et al. |
| **PLDI** | Programming Language Design and Implementation | PL 实现顶会 | **Pin 2005** 动态二进制插桩 |
| **OOPSLA** | Object-Oriented Programming, Systems, Languages & Applications | 面向对象系统 | AspectJ 相关论文 |

#### 数据库（Databases）——MySQL 本身的归属

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **SIGMOD** | ACM SIGMOD Conference | 数据库第一会 | 查询引擎、存储引擎论文 |
| **VLDB** | Very Large Data Bases | 与 SIGMOD 同级 | InnoDB、B+tree、AHI 论文 |
| **ICDE** | IEEE International Conference on Data Engineering | 数据库三大会之一 | 数据工程 |

#### 体系结构（Architecture）

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **ISCA** | International Symposium on Computer Architecture | 体系结构第一会 | Smith 1981 / Yeh & Patt 1991 分支预测 |

#### 软件工程（Software Engineering）

| 会议 | 全称 | 侧重 | 本文关联 |
|------|------|------|---------|
| **ICSE** | International Conference on Software Engineering | 软工第一会 | 可观测性框架设计、profiling 方法学 |
| **FSE/ESEC** | Foundations of Software Engineering | 与 ICSE 同级 | 软件诊断、性能分析 |

#### 经典期刊

| 期刊 | 全称 | 本文关联 |
|------|------|---------|
| **CACM** | Communications of the ACM | Knowlton 1965 buddy system、Parnas 1972 信息隐藏 |
| **TAOCP** | Knuth, *The Art of Computer Programming* | Vol.1 (1968) buddy/bump allocator 形式化分析 |

#### 工业会议

| 会议 | 本文关联 |
|------|---------|
| **BSDCan** | jemalloc 2006 |

#### 领域交叉关系图

```
         Systems（SOSP/OSDI/ATC/EuroSys）← DTrace, jemalloc
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
PL(ECOOP/PLDI)  Databases(SIGMOD/VLDB)  Architecture(ISCA)
AOP 1997        InnoDB/B+tree/AHI        分支预测理论
    │                    │
    └────────┬───────────┘
             │ Hoard (ASPLOS 2000)
             ▼
    Software Engineering (ICSE/FSE)
    Observability, Profiling
```

---

> **说明**：以上仅列出与我们讨论相关的领域。不涉及的领域包括：网络（SIGCOMM）、安全（CCS/S&P/USENIX Sec/NDSS）、理论（STOC/FOCS）、AI/ML（NeurIPS/ICML/ICLR）、CV（CVPR/ICCV）、NLP（ACL/EMNLP）、HCI（CHI/UIST）、分布式（PODC/DISC）、存储（FAST/MSST）。

---

## 第七章：总结：设计哲学全景

| 设计原则 | 思想/理论来源 | MySQL 中的实现 |
|---------|-------------|---------------|
| **信息隐藏** | Parnas, "Decomposing Systems into Modules" (CACM 1972) | PFS PSI 接口：上层通过稳定函数指针表调用，内部独立演化 |
| **插件化 / 微内核** | Microkernel Pattern (POSA Vol.1, 1996) | MySQL Server 为核心，PFS 为 mandatory built-in plugin |
| **编译期编织** | AOP (ECOOP 1997) | `#define → inline_mysql_mutex_lock`：关注点分离，业务代码零感知 |
| **运行时动态追踪** | Dynamic Instrumentation / DTrace (USENIX ATC 2004) | `start_mutex_wait()` / `end_mutex_wait()`：运行时采集真实事件 |
| **全量事件追踪** | Event-based Tracing (SOSP 1997) | PFS：每次 instrumented 操作记录 start/end event |
| **零开销关闭** | Zero-cost Probes (USENIX ATC 2004) | `#ifdef` + `inline` + 分支预测三层防护 |
| **受控容量** | Ring Buffer（经典数据结构） | PFS：current → history → history_long |
| **多维度聚合** | OLAP 聚合（经典） | PFS：summary_by_event_name / thread / user / host |
| **静态 vs 动态分析区分** | Dynamic Instrumentation 理论 | PFS 编织在编译期，但采集在运行时，属于 Dynamic Analysis |
| **极速分配** | Bump Allocator（Knuth 1968） | mem_heap：推 `free` 指针，O(1) |
| **BP 集成** | 自定义设计 | mem_heap `MEM_HEAP_BUFFER` 类型 |
| **死锁规避** | 锁顺序理论 | `MEM_HEAP_BTR_SEARCH` + 预留 `free_block_ptr` |
| **越界检测** | Canary/Guard Bytes（经典调试技术） | `MEM_NO_MANS_LAND`（0xCE/0xDF） |
| **伙伴分裂合并** | Buddy System (Knowlton, CACM 1965) | buf_buddy：压缩页 frame |
| **多核可扩展** | Hoard (ASPLOS 2000) / jemalloc (BSDCan 2006) | 底层 jemalloc |
| **指数增长 vs 固定大小** | 分配器自适应策略 | MEM_ROOT：`m_block_size *= 1.5`（自适应用户需求）；mem_heap：固定 block（场景可预测） |
| **栈式释放 vs 全量清空** | 生命周期管理策略 | mem_heap：`mem_heap_free_top`（LIFO）；MEM_ROOT：`Clear()` / `ClearForReuse()`（all-or-nothing） |
| **细粒度复用 vs 粗粒度复用** | 分配器设计取舍 | buf_buddy/jemalloc：free list 支持任意位置归还再分配；mem_heap/MEM_ROOT：放弃单块复用，换 O(1) 分配速度 + 零碎片，仅在 block/快照级复用 |
| **RAII 安全** | C++ 惯用法 | Scoped_heap / Mem_root_allocator |
| **Strategy（策略）** | GoF 1994 行为型模式 | PSI vtable：编译时/运行时切换不同埋点策略；mem_heap type：DYNAMIC/BUFFER/BTR_SEARCH 三种分配策略 |
| **Template Method（模板方法）** | GoF 1994 行为型模式 | PFS start→do→end 三部曲：宏定义了埋点骨架，业务代码填充步骤2（好莱坞原则） |
| **Observer（观察者）** | GoF 1994 行为型模式 | Replication Delegate：`add_observer()`/`remove_observer()`/`FOREACH_OBSERVER` 通知循环 |
| **Facade（外观）** | GoF 1994 结构型模式 | `mysql_mutex_t` 统一门面：隐藏平台锁 + PFS 插桩 + SAFE_MUTEX 三个子系统 |
| **Bridge（桥接）** | GoF 1994 结构型模式 | PSI 抽象接口 ↔ PFS 具体实现：抽象与实现独立编译，多版本 ABI 共存 |
| **Adapter（适配器）** | GoF 1994 结构型模式 | `Mem_root_allocator<T>`：将 arena 分配器适配为 STL allocator 接口 |
| **Decorator（装饰器）** | GoF 1994 结构型模式 | `ut::malloc_withkey`：对 malloc 透明增加 PFS 统计功能（函数级装饰） |
| **Factory Method（工厂方法）** | GoF 1994 创建型模式 | `PSI_bootstrap::get_interface(version)`：根据版本号创建对应 ABI 实现 |
| **Microkernel（微内核）** | POSA 1996 架构模式 | MySQL Server 核心 + 插件海洋：PFS、InnoDB、复制等均以统一插件接口集成 |
| **大页 TLB 优化** | OS 内存管理 | `large_page_alloc`：BP 底层用 `mmap(MAP_HUGETLB)` |

---

## 附录：推荐阅读文件清单

| 目的 | 文件路径 | 大小 |
|------|---------|------|
| mem_heap 数据结构 | `storage/innobase/include/mem0mem.h` | 526 行 |
| mem_heap 内联实现 | `storage/innobase/include/mem0mem.ic` | 530 行 |
| mem_heap block 分配（三种 type 分支） | `storage/innobase/mem/memory.cc` | 485 行 |
| MEM_ROOT 实现 | `mysys/my_alloc.cc` | 304 行 |
| MEM_ROOT 头文件 | `include/my_alloc.h` | 420 行 |
| buf_buddy 实现 | `storage/innobase/buf/buf0buddy.cc` | ~400 行 |
| buf_buddy 内联 | `storage/innobase/include/buf0buddy.ic` | ~91 行 |
| AHI 死锁规避（free_block 预留机制） | `storage/innobase/btr/btr0sea.cc` | — |
| PFS 引擎核心 | `storage/perfschema/pfs.cc` | 315KB |
| PFS event 基类定义 | `storage/perfschema/pfs_events.h` | 74 行 |
| PFS wait event 定义 | `storage/perfschema/pfs_events_waits.h` | 165 行 |
| PSI mutex 编织模板（宏 + inline 完整逻辑） | `include/mysql/psi/mysql_mutex.h` | 356 行 |
| PSI memory 编织模板 | `include/mysql/psi/mysql_memory.h` | — |
| ut::malloc 埋点 + PSI key 注册 | `storage/innobase/include/ut0new.h` | ~2200 行 |
| ut::malloc 实现 | `storage/innobase/ut/ut0new.cc` | — |
| Scoped_heap RAII 封装 | `storage/innobase/include/mem0mem.h:440-507` | — |
| PFS 插件声明 | `storage/perfschema/ha_perfschema.cc:1564` | — |
| temptable allocator（分层分配器） | `storage/temptable/include/temptable/allocator.h` | — |
| large_page_alloc（大页分配器） | `storage/innobase/include/detail/ut/page_alloc.h` | — |
| large_page_alloc (Linux 实现) | `storage/innobase/include/detail/ut/large_page_alloc-linux.h` | — |

---

*本文基于 MySQL 8.0 源码（`storage/innobase`、`storage/perfschema`、`mysys`、`include/mysql/psi` 等模块）分析撰写，力求在数据结构、分配算法、学术理论、工程实现四个层面提供完备的参考。全部代码引用来自项目中实际存在的文件，所有论文引用均可通过会议名称和年份检索验证。*
