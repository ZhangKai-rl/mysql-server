# Performance Schema（PFS）：MySQL 内建可观测性框架

> 本篇聚焦 **PFS 可观测性框架**。内存主题三篇分工：[allocator.md](../../memory/allocator.md)（分配器矩阵 + **memory 表族的埋点与聚合机制**：`PFS_metadata` 前缀、`UT_NEW_THIS_FILE_PSI_KEY`、`pfs_memory_alloc_vc` 三道闸、`high_alloc` 峰值语义）、[memory_architecture.md](../../memory/memory_architecture.md)（各内存区总账 + OOM 排查）、[jemalloc.md](../../memory/jemalloc.md)（★生产重点篇——PFS 覆盖不到时用 jemalloc 账本与 prof）；读源码通用知识地图与共享术语/论文见 [README.md](../../../server/infra/README.md)。
> 基于 MySQL 8.0 源码（`storage/perfschema`、`include/mysql/psi` 等模块）。

## 目录

- [摘要](#摘要)
- [一、Performance Schema](#一performance-schema)
  - [1.10 ★ PFS 的表不是固定的：静态内置 + 插件动态注册](#110--pfs-的表不是固定的静态内置--插件动态注册)（★ 查不到 `clone_progress` 先查插件装没装）
- [二、PFS 模块涉及的设计模式](#二pfs-模块涉及的设计模式)
- [三、插件系统涉及的设计模式](#三插件系统涉及的设计模式)
- [四、核心术语中英文对照（可观测性）](#四核心术语中英文对照可观测性)
- [五、学术论文索引（可观测性）](#五学术论文索引可观测性)
- [六、设计哲学（PFS 部分）](#六设计哲学pfs-部分)
- [附录：推荐阅读文件清单（PFS 部分）](#附录推荐阅读文件清单pfs-部分)

---

## 摘要

Performance Schema 是 MySQL 内建的可观测性与性能诊断框架：在所有关键路径上**埋点（instrumentation）**，运行时以 **event（事件）** 为单位记录每一次被 instrument 操作的起止时间、线程 ID、源码位置等元数据，并提供 SQL 接口供用户查询聚合统计（summary）和事件明细（history）。其设计落地了 AOP（编译期宏编织）、DTrace 零开销探针、Event-based Tracing（全量事件追踪）等理论：不启用时埋点被条件编译完全剔除（zero-cost when disabled），启用后以插件形式贯穿全引擎。

---
## 一、Performance Schema —— MySQL 内建可观测性框架

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

### 1.6.1 主链路：一次加锁的端到端 instrumentation 流程

上面的分层图是**静态结构**，主链路是它**动态走一遍**——从业务代码调 `mysql_mutex_lock` 到 event 落进 ring buffer 的完整脉络：

```
业务代码（开发者零 PFS 感知）
  mysql_mutex_lock(&dict_operation_lock)
    │ ① 第一层宏展开：注入源码位置（__FILE__ / __LINE__）
    ▼
  mysql_mutex_lock_with_src(&m, "btr0sea.cc", 320)
    │ ② 第二层宏展开：进入 inline 函数
    ▼
  inline_mysql_mutex_lock(&m, file, line)
    │ ③ 运行时逐层短路：m_psi != nullptr? m_enabled?
    ▼
  PSI_MUTEX_CALL(start_mutex_wait)          ← 编织点 A
    │  = psi_mutex_service->start_mutex_wait（vtable 间接 / PFS_DIRECT_CALL 直调）
    ▼
  pfs_start_mutex_wait_v1(&state, mutex, LOCK, file, line)
    │ ④ 记录 timer_start 到 locker state
    ▼
  my_mutex_lock(&m_mutex)                   ← 真正的加锁（业务本体）
    │
    ▼
  PSI_MUTEX_CALL(end_mutex_wait)            ← 编织点 B
    │  = pfs_end_mutex_wait_v1(locker, rc)
    ▼
  ⑤ 计算耗时 → 写入 events_waits_current（ring buffer）
     → 按 thread / event_name 聚合到 summary 表
```

**这一条链路的三个关键分叉点**：

- **③ 短路判断**：`m_psi == nullptr`（未注册）或 `m_enabled == false`（类别关闭）时直接跳过 ④⑤，开销约 2 条 `cmp` + 分支预测命中
- **① ② 宏展开 vs ③ 运行时**：宏在**编译期**决定"埋点代码是否存在"，运行时只判断"是否激活"——这就是"编译期编织 + 运行时采集"的分工
- **⑤ 落点**：start 记开始时间，end 算差值，同一个 event 从 `current` 滚入 `history` 再到 `summary` 聚合

> 对照 1.4.2 的宏展开：那里讲的是**每一层宏怎么展开**（静态），这里讲的是**展开后的代码运行时怎么流转**（动态）。两者一静一动，合成完整认知。

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

### 1.9 PFS 的变量与状态表实现（SHOW VARIABLES/STATUS 的幕后）

PFS 不只是埋点框架——**8.0 起它还是 `SHOW VARIABLES` / `SHOW STATUS` 的实际实现者**（两条 SHOW 语句在语法分析阶段被重写成 `SELECT ... FROM performance_schema.global_variables / global_status`）。这也是"描述符与值分离"（见 [`variables.md`](../../../server/infra/variables.md)）在监控层的复用：三类变量（系统/状态/用户）都被统一成 `SHOW_VAR` 中间格式，走同一条管线。

#### 三阶段管线

`storage/perfschema/pfs_variable.h` 的 OVERVIEW 注释明说：系统变量与状态变量在 server 里实现不同，但在 PFS 里处理步骤相同——**INITIALIZE（构建排序好的 SHOW_VAR 清单）→ MATERIALIZE（按类型求值、递归展开、跨线程聚合、转字符串）→ OUTPUT（SHOW 或表查询迭代缓存）**。

两个平行的 cache 类（模板基类 `PFS_variable_cache<T>`）：

```
PFS_system_variable_cache（系统变量）
  ├─ INITIALIZE: System_variable_tracker::enumerate_sys_vars(sort, scope, strict, output)
  │              —— 持 LOCK_system_variables_hash 遍历 static/dynamic 双 hash，
  │                 输出 Prealloced_array<System_variable_tracker, 200>（预分配 200 槽）
  └─ MATERIALIZE: 每个 tracker 经 access_system_variable() 找到 sys_var
                  → 包成 {name, (char*)sysvar, SHOW_SYS} 的 SHOW_VAR
                  → System_variable 对象求值转文本

PFS_status_variable_cache（状态变量）
  ├─ INITIALIZE: init_show_var_array + expand_show_var_array（展开 Com_ / Innodb 等 SHOW_ARRAY）
  └─ MATERIALIZE: manifest() 递归（SHOW_FUNC 回调可链式、SHOW_ARRAY 前缀拼接）
                  → Status_variable 对象（get_one_variable 加 status_var 基址解引用）
```

物化后的行对象：`System_variable`/`Status_variable` 各持 `m_value_str[SHOW_VAR_FUNC_BUFF_SIZE+1]`（1025 字节）——这就是 `VARIABLE_VALUE VARCHAR(1024)` 的由来。

#### 优化器/执行器如何驱动 PFS 表

**全表扫描**（`SELECT * FROM pfs.global_variables`，无谓词 → AccessPath 是 `TABLE_SCAN`）：

```
TableScanIterator::Init                     sql/iterators/basic_row_iterators.cc
  → table()->file->ha_rnd_init(true)        ← handler 包装
    → ha_perfschema::rnd_init               ha_perfschema.cc:1684
        ├─ m_table == nullptr 时 m_open_table() = table_global_variables::create() new 表游标
        └─ m_table->rnd_init(scan)
            └─ m_sysvar_cache.materialize_global()   ← ★ 全量物化发生在 Init
TableScanIterator::Read（逐行）
  → ha_rnd_next → ha_perfschema::rnd_next   ha_perfschema.cc:1714
      ├─ m_table->rnd_next()                ← cache 下标游标 + make_row（含审计事件）
      └─ m_table->read_row(...)             ← read_row_values 把 System_variable 翻译成 Field 值
扫描结束
  → ha_rnd_end → ha_perfschema::rnd_end     ha_perfschema.cc:1706
      └─ delete m_table                     ← 销毁游标与 m_sysvar_cache
```

**点查**（`WHERE VARIABLE_NAME='x'`）：优化器生成 `REF`/`EQ_REF` 或 `INDEX_RANGE_SCAN`（取决于成本判定），但无论哪条路径，落到的 handler 序列都一样——**PFS 的"索引"是线性过滤，没有 hash 查找**：

```
REF 路径:   RefIterator::Init → ha_index_init → ha_perfschema::index_init
              → table_global_variables::index_init
                  └─ m_sysvar_cache.materialize_global()          ← ★ 点查也全量物化！
                     + PFS_NEW(PFS_index_global_variables) 建索引对象
            RefIterator::Read → ha_index_read_map → ha_perfschema::index_read
              → PFS_engine_table::index_read
                  └─ m_index->read_key() 解析 key → reset_position() → index_next()
                      → 线性扫描 m_cache，m_opened_index->match() 逐行比对变量名字符串
INDEX_RANGE_SCAN 路径: IndexRangeScanIterator → ha_multi_range_read_next（handler 默认 MRR）
              → read_range_first（ha_index_read_map → index_read）
              → read_range_next（ha_index_next_same → index_next_same → 退化为 index_next）
```

关键证据——`PFS_engine_table::index_read` 解析 key 后直接从头线性扫：

```cpp
int PFS_engine_table::index_read(KEY *key_infos, uint index, const uchar *key,
                                 uint key_len, enum ha_rkey_function find_flag) {
  if (m_index == nullptr) return HA_ERR_END_OF_FILE;
  KEY *key_info = key_infos + index;
  m_index->set_key_info(key_info);
  m_index->read_key(key, key_len, find_flag);   // 只把 key 存进 PFS_engine_index
  reset_position();
  return index_next();                           // 从头线性扫描 + match 过滤
}
int PFS_engine_table::index_next_same(const uchar *, uint) { return index_next(); }
```

所以 `PRIMARY KEY (VARIABLE_NAME) USING HASH` 的 `USING HASH` **只是 DDL 声明**（`get_default_index_algorithm()` 返回 `HA_KEY_ALG_HASH`），运行时没有任何 hash 表。

**`ha_perfschema` 的 handler 特性声明**（决定优化器行为的几个关键 flags）：

```cpp
ulonglong table_flags() const override {
  /*
    About HA_FAST_KEY_READ:
    The storage engine ::rnd_pos() method is fast to locate records by key,
    so HA_FAST_KEY_READ is technically true, but the record content can be
    overwritten between ::rnd_next() and ::rnd_pos(), because all the P_S
    data is volatile.  The HA_FAST_KEY_READ flag is not advertised, to force
    the optimizer to cache records instead, to provide more consistent records.
  */
  return HA_NO_TRANSACTIONS | HA_NO_AUTO_INCREMENT |
         HA_PRIMARY_KEY_REQUIRED_FOR_DELETE | HA_NULL_IN_KEY | HA_NULL_PART_KEY;
}
```

- **故意不声明 `HA_FAST_KEY_READ`**：注释原话——PFS 数据易变，`rnd_next` 与 `rnd_pos` 之间内容可能被改写；不声明该 flag 迫使优化器**缓存整行**（`set_record_buffer`）而不是"拿 rowid 再回表"，保证"WHERE THREAD_ID=n"的过滤结果自洽。
- `index_flags()` 返回 `HA_KEY_SCAN_NOT_ROR`：禁止 Rowid Ordered Retrieval。
- `HA_NO_TRANSACTIONS`：无事务语义，锁模型最简。

#### cache 的本质与生命周期

**有 cache，但它的生命周期是"单条语句的一次表扫描"，不是跨语句的全局缓存。** 这是最容易误解的一点——同一连接连续两次查询，第二次照样从零物化，**cache 不跨语句复用**。

**为什么这么设计：cache 的目的不是"跨查询加速"，而是"单次扫描内的快照 + 免锁迭代"。** 三个理由：

1. **一致性快照语义**。变量随时可能被其他线程 `SET`、被插件装卸改集合。物化在锁内（`LOCK_plugin_delete` + 求值锁）一次性把"当前的变量集合 + 当前的值"定格成文本快照，之后 `rnd_next` 迭代读的是定格结果——**保证同一次扫描内行与行自洽**（不会扫到一半集合变了、值变了）。如果没有 cache、逐行求值，就要么每行持锁（锁持有时间放大 N 倍），要么不持锁（行间不一致）。
2. **锁的持有范围被限制在一个物化周期内**。`LOCK_plugin_delete`、`LOCK_global_system_variables` 等都只在物化循环内持有，出循环即释放；后续迭代无锁。若 cache 跨语句存活，要么跨语句持锁（灾难），要么引入失效协议（谁改了变量通知谁的缓存——复杂且易错）。
3. **PFS 的语义是"查询即当前状态"**。它的价值就在于反映实时状态；跨语句缓存必然返回陈旧值，直接违背产品语义。这与其他"为读性能而生"的 cache（table_definition_cache 等）目的根本不同。

代价就是：高频轮询 PFS 变量表 = 每次全量"枚举 hash + 排序 + N 次求值 + 格式化"，且 `get_row_count`（表打开时统计行数）还要持锁读 hash——高频 `SHOW VARIABLES` 的锁竞争是真实的性能风险点。

**PFS 表也不在磁盘上**——它是纯内存"引擎表"（PERFORMANCE_SCHEMA 引擎：无数据文件、无 page、无 buffer pool）。PFS 引擎的一切都在内存：表定义（`Plugin_table`）在代码静态区、表 share 在 PFS 的 share 注册表、表游标对象每次扫描 `new` 在堆上、cache 就住在游标对象里。

**两个 hash 不是 PFS 的**——`static/dynamic_system_variable_hash` 是 **SQL 层 `sys_var` 体系**的全局结构（`sql/set_var.cc`），值存储 `global_system_variables`/`THD::variables` 也归 SQL 层。PFS 只是"借用"它们来物化。精确的数据流时序：

```
【rnd_init —— 物化阶段，只发生一次，访问 hash 和值存储就在此刻】
  enumerate_sys_vars 遍历两个 hash      ← 这里碰 hash
    → sys_var* 列表（tracker 数组）
  → 每个 sys_var: value_ptr(OPT_GLOBAL/SESSION)
    → 读 global_system_variables / THD::variables   ← 这里碰值存储
    → 格式化文本
  → 存进 m_cache（PFS 表游标对象的文本快照数组）

【rnd_next —— 迭代阶段，每行一次，只读 cache】
  m_cache[i] → make_row → 输出
  —— 不碰 hash、不碰 global_system_variables、不再求值
```

#### 代码证据：物化只在 `rnd_init` 发生一次

`storage/perfschema/table_global_variables.cc:114-134`：

```cpp
int table_global_variables::rnd_init(bool /* scan */) {
  /*
    Build a list of system variables from the global system variable hash.
    Filter by scope.
  */
  m_sysvar_cache.materialize_global();

  return 0;
}

int table_global_variables::rnd_next() {
  for (m_pos.set_at(&m_next_pos); m_pos.m_index < m_sysvar_cache.size();
       m_pos.next()) {
    const System_variable *system_var = m_sysvar_cache.get(m_pos.m_index);
    if (system_var != nullptr) {
      m_next_pos.set_after(&m_pos);
      return make_row(system_var);
    }
  }
  return HA_ERR_END_OF_FILE;
}
```

/* 【注】★ `rnd_next` 里**只有三个纯读操作**：`size()` / `get(index)` / `make_row()`，
   完全没有再次物化、没有再碰 hash。这就是"cache 生命周期 = 一次表扫描"的代码证据。 */

还有一层**幂等守卫**（`storage/perfschema/pfs_variable.h:517-527`）：

```cpp
template <class Var_type>
int PFS_variable_cache<Var_type>::materialize_global() {
  if (is_materialized()) {
    return 0;
  }

  return do_materialize_global();
}
```

#### cache 就是游标对象的一个成员

`storage/perfschema/table_global_variables.h:104-120`：

```cpp
 private:
  /** Table share lock. */
  static THR_LOCK m_table_lock;
  /** Table definition. */
  static Plugin_table m_table_def;

  /** Current THD variables. */
  PFS_system_variable_cache m_sysvar_cache;
  /** Current row. */
  row_global_variables m_row;
  /** Current position. */
  pos_t m_pos;
  /** Next position. */
  pos_t m_next_pos;

  PFS_index_global_variables *m_opened_index;
};
```

而游标对象是**每次扫描 `new`、扫描结束 `delete`**（`storage/perfschema/table_global_variables.cc:89-91`、`:103-107`）：

```cpp
PFS_engine_table *table_global_variables::create(PFS_engine_table_share *) {
  return new table_global_variables();
}
```

```cpp
table_global_variables::table_global_variables()
    : PFS_engine_table(&m_share, &m_pos),
      m_sysvar_cache(false),
      m_pos(0),
      m_next_pos(0) {}
```

`storage/perfschema/ha_perfschema.cc:1684-1714`、`:1613-1626`：

```cpp
int ha_perfschema::rnd_init(bool scan) {
  int result;
  DBUG_TRACE;

  assert(m_table_share);
  assert(m_table_share->m_open_table != nullptr);

  stats.records = 0;
  if (m_table == nullptr) {
    m_table = m_table_share->m_open_table(m_table_share);
  } else {
    m_table->reset_position();
  }

  if (m_table != nullptr) {
    m_table->rnd_init(scan);
  }

  result = m_table ? 0 : HA_ERR_OUT_OF_MEM;
  return result;
}

int ha_perfschema::rnd_end() {
  DBUG_TRACE;
  assert(m_table);
  delete m_table;
  m_table = nullptr;
  return 0;
}
```

```cpp
int ha_perfschema::close() {
  DBUG_TRACE;

  /* Only for table added by plugin/components */
  if (!IS_NATIVE_TABLE(m_table_share)) {
    dec_ref_count(m_table_share);
  }

  m_table_share = nullptr;
  delete m_table;
  m_table = nullptr;

  return 0;
}
```

/* 【注】★ `m_open_table` 是 share 上的一枚**函数指针**（`pfs_open_table_t`），
   内置表指向 `table_xxx::create`（裸 `new`），插件表指向 `table_plugin_table::create`。
   所以"cache 随游标一起生、一起死"是**对象生命周期**层面的事实，不是靠某个清理函数。 */

#### 物化的三个动作：枚举 → 取值 → 转文本

**枚举**（`sql/set_var.cc:1101-1138`）——注意它产出的是 `System_variable_tracker`，**不是** SHOW_VAR：

```cpp
/**
  Constructs an array of system variables for display to the user.
  ...
  @note Requires r/o lock on LOCK_system_variables_hash.
*/
bool System_variable_tracker::enumerate_sys_vars(
    bool sort, enum enum_var_type query_scope, bool strict,
    System_variable_tracker::Array *output) {
  assert(output != nullptr);
  assert(query_scope == OPT_SESSION || query_scope == OPT_GLOBAL);
  int count = get_system_variable_count();

  /* Resize array if necessary. */
  if (output->reserve(count + 1)) return true;

  if (enumerate_sys_vars_in_hash(static_system_variable_hash, query_scope,
                                 strict, output) ||
      enumerate_sys_vars_in_hash(dynamic_system_variable_hash, query_scope,
                                 strict, output))
    return true;  // OOM

  if (sort)
    std::sort(
        output->begin(), output->end(),
        [](const System_variable_tracker &a, const System_variable_tracker &b) {
          return my_strcasecmp(system_charset_info, a.get_var_name(),
                               b.get_var_name()) < 0;
        });

  return false;
}
```

/* 【注】① 两个 hash **分别遍历**后**统一排序**——这解释了你看到的"变量按字母序输出"；
   ② 预分配 `SYSTEM_VARIABLE_PREALLOC = 200`（`set_var.h:855`），超出才 reserve。 */

**取值**（`sql/set_var.cc:429-443`）：

```cpp
const uchar *sys_var::value_ptr(THD *running_thd, THD *target_thd,
                                enum_var_type type,
                                std::string_view keycache_name) {
  if (type == OPT_GLOBAL || type == OPT_PERSIST || scope() == GLOBAL) {
    mysql_mutex_assert_owner(&LOCK_global_system_variables);
    AutoRLock lock(guard);
    return global_value_ptr(running_thd, keycache_name);
  } else
    return session_value_ptr(running_thd, target_thd, keycache_name);
}
```

/* 【注】`mysql_mutex_assert_owner(&LOCK_global_system_variables)` 这条断言就是
   "取全局值必须先持那把锁"的硬契约。 */

**转文本**（`storage/perfschema/pfs_variable.cc:686-735`，节选）：

```cpp
  /* Block remote target thread from updating this system variable. */
  if (target_thd != current_thread) {
    mysql_mutex_lock(&target_thd->LOCK_thd_sysvar);
  }
  /* Block system variable additions or deletions. */
  mysql_mutex_lock(&LOCK_global_system_variables);

  auto *system_var = (sys_var *)show_var->value;
  assert(system_var != nullptr);
  m_charset = system_var->charset(target_thd);
  m_type = system_var->show_type();
  m_scope = system_var->scope();

  /* Get the value of the system variable. */
  const char *value;
  value = get_one_variable_ext(current_thread, target_thd, show_var,
                               query_scope, show_var_type, nullptr, &m_charset,
                               m_value_str, &m_value_length);
  ...
  m_initialized = true;
}
```

/* 【注】看别的线程的会话变量时，还要额外锁目标线程的 `LOCK_thd_sysvar`——
   这是"跨线程看变量"的代价，也是高频查 `variables_by_thread` 特别慢的原因之一。 */

**所以不是"rnd_next 进 hash 再索引到 global"**——那是 `rnd_init` 干的事，且只干一次。`rnd_next` 读的是物化时定格的**文本快照**（这就是快照一致性的来源：迭代期间值被别的线程 SET 也影响不到本次扫描）。

PFS 表更像一个**每次查询现场计算的视图**：`rnd_init` 时从 SQL 层的内存变量存储读值、格式化、定格进 cache；`rnd_next` 只是把定格结果搬给 SQL 层。值**从不在 PFS 引擎上长期存在**——PFS 只有游标级的快照。

cache 是**表打开对象**的成员：每次 `SELECT ... FROM pfs.global_variables` 打开表时，`table_global_variables::create()` **new 一个新的 `table_global_variables` 对象**，其成员 `m_sysvar_cache`（`PFS_system_variable_cache` 实例）是全新的。所以：

- **每条语句都从零重新物化**：重新枚举双 hash、重新求值几百个变量的值、重新格式化文本。没有"上次查询的结果缓存"。
- **短路只在同一次扫描内生效**：入口包装函数先查 `is_materialized()`：

```cpp
template <class Var_type>
int PFS_variable_cache<Var_type>::materialize_global() {
  if (is_materialized()) {
    return 0;                    // 同一次扫描内二次调用（如 index_init 又调一次）直接返回
  }
  return do_materialize_global();
}
```

  这个短路服务于：① `rnd_init` 之后 `index_init` 再调不重复物化；② `status_by_thread` 的 `rnd_next` 对每个线程循环物化时，`is_materialized(pfs_thread)` 防止同一线程重复物化（`rnd_pos` 点查后又 `rnd_next` 的场景）。

cache 是**两层结构**：

| 层 | 成员 | 内容 | 标志 |
|---|---|---|---|
| 清单层 | `m_sys_var_tracker_array`（`Prealloced_array<System_variable_tracker, 200>`） | INITIALIZE 阶段枚举出来的变量描述符（tracker）数组，预分配 200 槽 | `m_initialized` |
| 值层 | `m_cache`（`Prealloced_array<System_variable, 200>`） | MATERIALIZE 阶段物化出的**文本行**（名字 + 已格式化的字符串值 + charset + source） | `m_materialized` |

求值（`sys_var::value_ptr` + 类型格式化）**在物化时一次性完成**，`rnd_next`/`rnd_pos` 只是从 `m_cache` 按索引取行——所以表扫描期间不会二次求值。

**`external_init=true` 的表（`variables_by_thread`）有微妙的生命周期差异**：`rnd_init` 时只建清单层（`initialize_session()`，`m_initialized=true` 后**不再重建**，跨语句的表对象里清单可以复用）；值层 `m_cache` 在 `rnd_next` 对每个线程物化时**先 `clear()` 再重建**（`do_materialize_session(PFS_thread*)` 开头）。`global_variables`/`session_variables`（`external_init=false`）则两层都是每条语句新建。

**"SHOW VARIABLES 与直查都走 cache 吗"**——是，**都走同一个 `PFS_system_variable_cache`、同一条物化管线**（SHOW 被重写成 SELECT 后，内层 PFS 表查询与直查在 `ha_perfschema::rnd_init` 之后完全一致）。唯一区别在**外层**：SHOW 重写的查询是 `SELECT * FROM (内层 SELECT) 派生表`，SQL 层还会把派生表**物化到内部临时表**（这就是官方文档说 "Each invocation of the SHOW STATUS statement ... increments the global Created_tmp_tables value" 的根源——SHOW 路径 = PFS cache + 临时表物化**两层**；直查只有 PFS cache 一层）。

#### cache 的级别：游标级（rnd_init → rnd_end）

精确的层级是**"表扫描游标"级，比单条语句还短命**。挂在 handler 实例的 `m_table` 成员上：

```
ha_perfschema handler（每次 SQL 打开表 new 一个，属于语句的 TABLE 对象）
  └── m_table（PFS_engine_table 实例）
        ├─ ha_perfschema::rnd_init():   m_table == nullptr 时 m_open_table() new
        │                               → rnd_init(scan) 里 materialize_*
        └─ ha_perfschema::rnd_end():    delete m_table; m_table = nullptr
```

`ha_perfschema::rnd_init()` 在 `m_table == nullptr` 时 `m_table = m_table_share->m_open_table(...)`（即 `table_global_variables::create()` new 一个），`rnd_end()` 里 `delete m_table`。**一条语句内如果执行器对同一表做多趟 rnd 扫描，每趟 `rnd_init`/`rnd_end` 都会 new/delete 一个全新对象和全新 cache。**

对照 MySQL 其他 cache 的级别：

| cache | 级别 | 跨连接共享？ |
|---|---|---|
| `table_definition_cache`（表定义） | 全局级 | 是 |
| `table_open_cache` | 全局池 + THD 级 | 部分 |
| query cache（8.0 已移除） | 全局级 | 是 |
| **PFS 变量 cache** | **游标级（rnd_init→rnd_end）** | 否 |

**连带推论：点查也全量物化。** `table_global_variables::index_init`（`WHERE VARIABLE_NAME='x'` 走索引）同样调 `materialize_global()` 物化**全部**变量，只是 `index_next` 里逐个 `match()` 过滤出匹配行——不存在"只求值那一个变量"的优化。唯一的例外是 `variables_by_thread` 的 `rnd_pos`（`materialize_session(pfs_thread, index)` 单变量物化）。

#### 物化家族全景：6 个入口函数 × 表

系统变量与状态变量的物化**语义完全不同**——状态变量是"跨线程求和"，系统变量是"选择读哪份存储"（GLOBAL 值 vs 某个 THD 的会话值）。这一差异决定了两个 cache 类的入口函数形态：

| 表 | 入口函数 | 读谁的存储 |
|---|---|---|
| `global_variables` | `PFS_system_variable_cache::do_materialize_global()` | `global_system_variables`（用 `m_current_thd` 求值） |
| `session_variables` | `PFS_system_variable_cache::do_materialize_all(THD*)` | 本会话 `current_thd->variables` |
| `variables_by_thread` | `do_materialize_session(PFS_thread*)` / `(PFS_thread*, uint index)` | 任意线程（`get_THD` 校验后） |
| `global_status` | `PFS_status_variable_cache::do_materialize_global()` | 全局基线 + **Σ 全部 THD** |
| `session_status` | `PFS_status_variable_cache::do_materialize_all(THD*)` | 本会话 `status_var`（快照优先） |
| `status_by_thread/user/host/account` | `do_materialize_session(PFS_thread*)` / `do_materialize_client(PFS_client*)` | 单线程求和 / client 维度聚合 |

#### INITIALIZE 细节：系统变量的清单构建

系统变量版的 `init_show_var_array` 与状态变量版（遍历 `all_status_vars` 数组）完全不同——它从**双 hash** 枚举：

```cpp
bool PFS_system_variable_cache::init_show_var_array(enum_var_type scope, bool strict) {
  assert(!m_initialized);
  m_query_scope = scope;

  mysql_rwlock_rdlock(&LOCK_system_variables_hash);      // 防插件装卸改 dynamic hash

  /* Record the system variable hash version to detect subsequent changes. */
  m_version = get_dynamic_system_variable_hash_version();

  /* Build the SHOW_VAR array from the system variable hash. */
  System_variable_tracker::enumerate_sys_vars(true, m_query_scope, strict,
                                              &m_sys_var_tracker_array);

  mysql_rwlock_unlock(&LOCK_system_variables_hash);

  /* Increase cache size if necessary. */
  m_cache.reserve(m_sys_var_tracker_array.size());

  m_initialized = true;
  return true;
}
```

要点：① `enumerate_sys_vars(sort=true, ...)` —— 清单**排序**（状态变量的 `all_status_vars` 是注册期排序，系统变量是每次物化时排序，因为 dynamic hash 是乱序的）；② `m_version = get_dynamic_system_variable_hash_version()` —— 快照版本号，用于检测"查询期间插件被装卸"（与状态变量侧 `get_status_vars_version()` 对应）；③ 输出是 `Prealloced_array<System_variable_tracker, 200>`（预分配 200 槽）。

`external_init=true` 的表（`variables_by_thread`）不走这个函数，而是在 `rnd_init()` 先调 `do_initialize_session()`（`LOCK_plugin_delete` 内建一次 `OPT_SESSION, strict=true` 数组），后续对每个线程**复用**该数组——避免 N 个线程 N 次重建 hash 枚举。

#### 系统变量版的 scope 过滤（`match_scope`）

注意它与状态变量版的 `match_scope` 语义不同——系统变量按 `sys_var::scope()` 三分支：

```cpp
bool PFS_system_variable_cache::match_scope(int scope) {
  switch (scope) {
    case sys_var::GLOBAL:       return m_query_scope == OPT_GLOBAL;
    case sys_var::SESSION:      return (m_query_scope == OPT_GLOBAL || m_query_scope == OPT_SESSION);
    case sys_var::ONLY_SESSION: return m_query_scope == OPT_SESSION;
    default:                    return false;
  }
}
```

含义：`SHOW GLOBAL VARIABLES` 显示 GLOBAL-only + **所有有 GLOBAL 副本的 SESSION 变量**（`SESSION` 分支对 OPT_GLOBAL 也返回 true），但不显示 ONLY_SESSION 变量（如 `gtid_next`、`sql_log_bin`——它们没有全局副本）；`SHOW SESSION VARIABLES` 显示 SESSION + ONLY_SESSION，不显示 GLOBAL-only。

#### 两个入口的实现差异（do_materialize_global vs do_materialize_all）

系统变量版 `do_materialize_global()`（`global_variables` 表）——**没有 get_THD**，用当前线程求值即可：

```cpp
int PFS_system_variable_cache::do_materialize_global() {
  mysql_mutex_lock(&LOCK_plugin_delete);
  m_materialized = false;
  if (!m_external_init) {
    init_show_var_array(OPT_GLOBAL, true);          // strict=true
  }
  for (const System_variable_tracker &i : m_sys_var_tracker_array) {
    auto f = [this](const System_variable_tracker &, sys_var *sysvar) -> void {
      if (match_scope(sysvar->scope())) {
        const SHOW_VAR show_var{sysvar->name.str, pointer_cast<char *>(sysvar),
                                SHOW_SYS, SHOW_SCOPE_UNDEF};
        const System_variable system_var(m_current_thd, &show_var, m_query_scope);
        m_cache.push_back(system_var);
      }
    };
    (void)i.access_system_variable(m_current_thd, f, Suppress_not_found_error::YES);
  }
  m_materialized = true;
  mysql_mutex_unlock(&LOCK_plugin_delete);
  return 0;
}
```

`do_materialize_all(THD *unsafe_thd)`（`session_variables` 表）——三个与上面不同的关键点：

```cpp
int PFS_system_variable_cache::do_materialize_all(THD *unsafe_thd) {
  m_unsafe_thd = unsafe_thd;
  ...
  if (!m_external_init) {
    init_show_var_array(OPT_SESSION, false);        // ① strict=false（非严格）
  }
  THD_ptr thd_ptr = get_THD(unsafe_thd);            // ② 校验本会话 THD（unsafe_thd 就是 current_thd）
  m_safe_thd = thd_ptr.get();
  if (m_safe_thd != nullptr) {
    for (const System_variable_tracker &i : m_sys_var_tracker_array) {
      auto f = [this](const System_variable_tracker &, sys_var *sysvar) {
        SHOW_VAR show_var;
        show_var.name = sysvar->name.str;
        show_var.value = (char *)sysvar;
        show_var.type = SHOW_SYS;
        show_var.scope = SHOW_SCOPE_UNDEF;
        /* Resolve value, convert to text, add to cache. */
        const System_variable system_var(m_safe_thd, &show_var, m_query_scope);  // ③ 无 match_scope
        m_cache.push_back(system_var);
      };
      (void)i.access_system_variable(m_current_thd, f, Suppress_not_found_error::YES);
    }
    ...
```

解释三点差异：① **`strict=false`** —— 非严格模式下 `enumerate_sys_vars` 的 scope 过滤放宽，GLOBAL-only 变量（如 `gtid_mode` 的全局值）也会出现在 session 视图里（与 `SHOW SESSION STATUS` 出现 `Uptime` 同理——"本会话能看到的都给你"）；② 求值循环里**没有 `match_scope` 二次过滤**（过滤已在 INITIALIZE 阶段完成，这点与 `do_materialize_global` 的"边枚举边过滤"不同）；③ `System_variable(m_safe_thd, ...)` 用 `m_safe_thd` 求值——读的是**该 THD 的 `variables`**，而 `do_materialize_global` 用 `m_current_thd` 读全局值（`sys_var::value_ptr` 里 `type==OPT_GLOBAL` 走 `global_value_ptr`）。

**配套的内存管理**：物化可能产生大量文本值（几百个变量的字符串），`do_materialize_session(PFS_thread*)` 在 `m_use_mem_root` 时调 `set_mem_root()`——把 `THR_MALLOC`（`thd->mem_root` 的宏）临时切换到专用 `m_mem_sysvar`（`SYSVAR_MEMROOT_BLOCK_SIZE` 块），物化完 `clear_mem_root()` 一次性释放并恢复，**避免耗尽被观察线程的 THD mem_root**（你正在读的那个线程的内存池不属于你）。

#### `System_variable` 类全貌与 variables_info 的复用

`System_variable` 是物化行的载体（`pfs_variable.h:169-203`），16 个 public 成员分三组：

| 组 | 成员 | 用途 |
|---|---|---|
| 值 | `m_name/m_name_length`、`m_value_str[1025]/m_value_length`、`m_type`、`m_scope`、`m_charset` | 变量名 + 物化后的文本值 + 类型/作用域/字符集 |
| 元数据 | `m_source`、`m_path_str/m_path_length`、`m_min_value_str`、`m_max_value_str`、`m_set_time`、`m_set_user_str`、`m_set_host_str` | variables_info 用：来源/路径/范围/设置时间与者 |
| 状态 | `m_initialized`（私有） | `is_null()` = `!m_initialized` |

**两个 `init` 重载对应两类表**：

- **求值版**（三参，`global_variables`/`session_variables` 用）：`get_one_variable_ext` 求值 → 文本进 `m_value_str`，元数据成员闲置。
- **元数据版**（两参，`variables_info` 用）：**不求值**（`m_value_str` 置空），只拷 `sys_var` 的 source/path/min/max/timestamp/user/host；对"只读 persisted 变量"（作为命令行选项处理，`sys_var` 本身不带 who/when），从 `Persisted_variables_cache` 的 `m_persisted_static_variables`/`m_persisted_static_parse_early_variables` 两张 map 里按名字补齐。`table_variables_info` 用的是派生类 `PFS_system_variable_info_cache`（只覆盖 `do_materialize_all`，物化时调元数据版构造），且**表定义无 PK、无 index_init**（`WHERE VARIABLE_NAME=...` 只能全表过滤）。

#### 线程间读：如何安全地读"别的线程"的会话变量

`variables_by_thread`/`session_variables` 需要读**任意 THD**（含后台线程）的会话变量，这是整个机制最需要小心的部分。`do_materialize_session` 的三重防护：

```cpp
int PFS_system_variable_cache::do_materialize_session(PFS_thread *pfs_thread) {
  /* Block plugins from unloading. */
  MUTEX_LOCK(plugin_delete_lock_guard, &LOCK_plugin_delete);   // ① 防插件卸载致 hash 变化

  /* Get and lock a validated THD from the thread manager. */
  THD_ptr thd_ptr = get_THD(pfs_thread);                       // ② 从 Global_THD_manager 校验指针仍存活
  m_safe_thd = thd_ptr.get();                                  //    （THD_ptr 引用计数防销毁）
  if (m_safe_thd != nullptr) {
    for (const System_variable_tracker &i : m_sys_var_tracker_array) {
      auto f = [this](const System_variable_tracker &, sys_var *sysvar) {
        SHOW_VAR show_var;
        show_var.name = sysvar->name.str;
        show_var.value = (char *)sysvar;   // SHOW_SYS：value 直接指向 sys_var 对象
        show_var.type = SHOW_SYS;
        if (match_scope(sysvar->scope())) {
          const System_variable system_var(m_safe_thd, &show_var, m_query_scope);  // ③ 求值时拿 target_thd 的锁
          m_cache.push_back(system_var);
        }
      };
      (void)i.access_system_variable(m_current_thd, f, Suppress_not_found_error::YES);
    }
  }
}
```

要点：① `LOCK_plugin_delete` 防插件装卸（静态变量免锁直取，动态变量持 `LOCK_system_variables_hash` 读锁——由 `access_system_variable` 按 Lifetime 分派）；② `get_THD` 经 thread manager 校验"该 PFS_thread 对应的 THD 还活着"，拿到引用计数保护的 `THD_ptr`——线程可能在物化中途结束，这是 8.0 修过的悬垂指针坑；③ 真正求值（`System_variable` 构造 → `sys_var::value_ptr(running_thd, target_thd, ...)`）时对 target_thd 拿 `LOCK_thd_sysvar`，区分 running_thd（执行查询的线程）与 target_thd（被读的线程）。

注意系统变量与状态变量在这点的差异：状态变量走 `manifest()` 直接用 `get_one_variable`，无 SHOW_SYS 类型；系统变量物化时才构造 `SHOW_SYS` 的 SHOW_VAR（value 是 `sys_var*`），复用同一条求值链。

还有第三个重载 `do_materialize_session(PFS_thread*, uint index)`：**按索引只物化单个变量**——`variables_by_thread` 的 `rnd_pos()`（点查 `WHERE THREAD_ID=x AND VARIABLE_NAME='y'` 走索引定位）用它避免物化整个线程的全部变量，`index` 是 `m_sys_var_tracker_array` 中的槽位。

#### `get_THD` / `THD_ptr`：安全拿到"别的线程"的 THD

`get_THD` 的完整机制——把裸指针交给 thread manager 校验：

```cpp
THD_ptr PFS_variable_cache<Var_type>::get_THD(THD *unsafe_thd) {
  if (unsafe_thd == nullptr) return THD_ptr{nullptr};   // 已断连的直接短路
  m_thd_finder.set_unsafe_thd(unsafe_thd);
  return Global_THD_manager::get_instance()->find_thd(&m_thd_finder);
}

THD_ptr Global_THD_manager::find_thd(Find_THD_Impl *func) {
  Find_THD find_thd(func);
  for (int i = 0; i < NUM_PARTITIONS; i++) {            // 9 个分区
    MUTEX_LOCK(lock, &LOCK_thd_list[i]);
    auto it = std::find_if(thd_list[i].begin(), thd_list[i].end(), find_thd);
    if (it != thd_list[i].end()) {
      THD_ptr thd_ptr(*it);                             // 构造即拿 LOCK_thd_data
      if (!thd_ptr->is_being_disposed()) return thd_ptr;  // 正在退出视为未找到
      break;
    }
  }
  return THD_ptr{nullptr};
}
```

三层保障：① `find_thd` 在**分区锁**内 `find_if` 线性比对（`Find_THD_variable::operator()` 只是裸指针相等比较 `thd == m_unsafe_thd`）；② 命中后**在仍持有分区锁时**构造 `THD_ptr`——其构造函数立即获取 `THD::LOCK_thd_data`（注释："ensures that THD::LOCK_thd_data mutex is acquired at instantiation"），阻止 THD 数据被并发销毁；③ `is_being_disposed()` 检查——正在退出的 THD 视为未找到。返回的 `THD_ptr` 靠移动语义传递（拷贝构造被 delete），析构时释放锁。

成员分工：`m_unsafe_thd`（调用者传入的裸指针，**只做身份比较**，绝不解引用）vs `m_safe_thd`（`THD_ptr::get()` 校验后的指针，物化循环只解引用它）。

#### 异常与边界路径

- **物化途中 THD 消失**：`get_THD` 返回空 → 所有 `do_materialize_*` 保持 `m_materialized=false`、返回 1 → 表驱动侧 `rnd_next` 的循环条件（`is_materialized()` 或 cache 为空）不满足 → **对外表现为空结果集**，不崩溃、不返回残行。
- **NULL 行**：PFS 的"NULL" = `m_initialized == false`（`init` 早退时），`make_row` 对 null 行返回 `HA_ERR_RECORD_DELETED`，`rnd_next` 的 for 循环跳过该行继续——null 行被"跳过"而非输出 SQL NULL。`variables_info` 的 `SET_TIME/SET_USER/SET_HOST` 为 0/空时才用 `f->set_null()` 表达真 SQL NULL。
- **超长值**：`m_value_length = std::min(m_value_length, 1024)` 静默截断（与 `VARIABLE_VALUE VARCHAR(1024)` 对齐），**无截断告警**（grep 确认无相关逻辑）。

#### 状态变量的聚合（do_materialize_global）

`GLOBAL = global_status_var + Σ(所有登记在 Global_THD_manager 的 THD)`，靠 `PFS_connection_status_visitor`：`visit_global()` 把全局基线 `add_to_status` 进局部 totals，`visit_THD()` 对每个 THD 累加。**含后台线程**（源码留 `// TODO: filter bg threads?`）。account/user/host 维度是**平行账本**（`PFS_status_stats` 遗留账本 + `do_materialize_client`），与全局通道隔离——`PFS_account::aggregate_status` 的注释："Never aggregate to global_status_var, because of the parallel THD -> global_status_var flow"（否则双倍计数）。

#### 性能特征：一次查询的真实工作量

一次 `SELECT * FROM pfs.global_variables` 的开销公式：

```
枚举 static+dynamic 双 hash 全部条目        O(N)，N ≈ 600~700（get_system_variable_count）
  + std::sort（my_strcasecmp 大小写不敏感）  O(N log N) —— 每次物化都排序
  + N 次求值：access_system_variable
              → System_variable 构造 → init
              → get_one_variable_ext(SHOW_SYS 特判)
              → sys_var::value_ptr（每个变量各自的求值函数）
              → 类型格式化转文本
  + N 次 m_cache.push_back（Prealloced_array 200 起扩容）
```

- `rnd_init` 与 `index_init` **都触发**这一整套流程（点查不省）；`rnd_end`/`index_end` 后 cache 随游标销毁——每次全新扫描重复全部工作。

#### 锁竞争：`LOCK_plugin_delete` 是真实瓶颈

**定义**（`sql/sql_plugin.cc:433-437`）：

```cpp
/*
  A mutex LOCK_plugin_delete must be acquired before calling plugin_del
  function.
*/
mysql_mutex_t LOCK_plugin_delete;
```

/* 【注】定义在 `sql_plugin.cc`（不是 `mutex_lock.h`），声明在 `sql/sql_plugin.h:48`；
   PFS 侧另有一处 extern：`storage/perfschema/pfs_variable.h:160`。 */

**为什么需要它——官方的锁优先级表**（`storage/perfschema/pfs_variable.h:118-131`）：

```cpp
  LOCK PRIORITIES
  ---------------
  System Variables
    LOCK_plugin_delete              (block plugin delete)
     LOCK_system_variables_hash
     LOCK_thd_data                  (block THD delete)
     LOCK_thd_sysvar                (block system variable updates,
  alloc_and_copy_thd_dynamic_variables)
       LOCK_global_system_variables (very briefly held)

  Status Variables
    LOCK_status
      LOCK_thd_data                 (block THD delete)
*/
```

/* 【注】★ `LOCK_plugin_delete` 在这张表里是**最外层**——它保护的是
   "变量集合本身不变"（插件装卸会增删 dyn hash 里的条目），
   而 `LOCK_system_variables_hash` 保护的是"hash 结构不变"。两者是包含关系。 */

物化路径上的加锁与那句"为什么"（`storage/perfschema/pfs_variable.cc:153-191`）：

```cpp
/**
  Build a GLOBAL system variable cache.
*/
int PFS_system_variable_cache::do_materialize_global() {
  /* Block plugins from unloading. */
  mysql_mutex_lock(&LOCK_plugin_delete);

  m_materialized = false;

  /*
     Build array of SHOW_VARs from system variable hash. Do this within
     LOCK_plugin_delete to ensure that the hash table remains unchanged
     during materialization.
   */
  if (!m_external_init) {
    init_show_var_array(OPT_GLOBAL, true);
  }

  /* Resolve the value for each SHOW_VAR in the array, add to cache. */
  for (const System_variable_tracker &i : m_sys_var_tracker_array) {
    auto f = [this](const System_variable_tracker &, sys_var *sysvar) -> void {
      /* Match the system variable scope to the target scope. */
      if (match_scope(sysvar->scope())) {
        const SHOW_VAR show_var{sysvar->name.str, pointer_cast<char *>(sysvar),
                                SHOW_SYS, SHOW_SCOPE_UNDEF};
        /* Resolve value, convert to text, add to cache. */
        const System_variable system_var(m_current_thd, &show_var,
                                         m_query_scope);
        m_cache.push_back(system_var);
      }
    };
    (void)i.access_system_variable(m_current_thd, f,
                                   Suppress_not_found_error::YES);
  }

  m_materialized = true;
  mysql_mutex_unlock(&LOCK_plugin_delete);
  return 0;
}
```

/* 【注】★ 锁的范围是**整个物化循环**——不是"枚举时锁一下"，而是
   "枚举 + N 次取值 + N 次格式化"全程持有。N ≈ 变量总数（几百），
   所以这一段的持锁时间是 O(N)，这就是"高频查 PFS 变量表会撞锁"的机制。 */

连 `get_row_count()` 也要持它（`storage/perfschema/table_global_variables.cc:93-101`）：

```cpp
ha_rows table_global_variables::get_row_count() {
  mysql_mutex_lock(&LOCK_plugin_delete);
  mysql_mutex_assert_not_owner(&LOCK_plugin);
  mysql_rwlock_rdlock(&LOCK_system_variables_hash);
  const ha_rows system_var_count = get_system_variable_count();
  mysql_rwlock_unlock(&LOCK_system_variables_hash);
  mysql_mutex_unlock(&LOCK_plugin_delete);
  return system_var_count;
}
```

而它返回的**只是上界**（`sql/set_var.cc:1013-1020`）：

```cpp
/*
  Number of records in the system_variable_hash.
  Requires r/o lock on LOCK_system_variables_hash.
*/
ulong get_system_variable_count(void) {
  return static_system_variable_hash->size() +
         dynamic_system_variable_hash->size();
}
```

/* 【注】两个 hash 的**全部条目数**，而实际输出行还会被 scope 过滤
   （`match_scope`）、敏感变量过滤（`is_sensitive()`，非 `SENSITIVE_VARIABLES_OBSERVER`
   就跳过）、不可见变量过滤（`not_visible()`）。所以必然是**上界**。 */

还有一条"版本漂移就告警"的兜底（`storage/perfschema/pfs_variable.cc:1564-1573`）：

```cpp
/**
  Warning issued if the version of the system variable hash table changes
  during a query. This can happen when a plugin is loaded or unloaded.
*/
void system_variable_warning() {
  THD *thd = current_thd;
  assert(thd != nullptr);
  push_warning(thd, ER_WARN_TOO_FEW_RECORDS);
  push_warning(thd, ER_SYSVAR_CHANGE_DURING_QUERY);
}
```

/* 【注】★ 这说明 `LOCK_plugin_delete` 也**没能做到绝对封闭**：插件变量是**延迟解析**的
   （tracker 只存名字，`access_system_variable` 时才去查），仍可能中途发现 hash 变了，
   于是用 `m_version` 前后比对 + 抛 warning 来兜。 */

配套的"插件不能被卸载"断言（`sql/sql_plugin.cc:1150-1155`）：

```cpp
static void plugin_del(st_plugin_int *plugin) {
  DBUG_TRACE;
  mysql_mutex_assert_owner(&LOCK_plugin);
  mysql_mutex_assert_owner(&LOCK_plugin_delete);
  /* Free allocated strings before deleting the plugin. */
  delete_dynamic_system_variable_chain(plugin->system_vars);
  restore_pluginvar_names(plugin->system_vars);
```

三把锁的**类型与持有范围**决定了谁会成为瓶颈：

| 锁 | 类型 | 持有范围 | 并发性 |
|---|---|---|---|
| `LOCK_plugin_delete` | **`mysql_mutex_t`（排他）** | 物化**全程**（`do_materialize_*` 开头到结尾，含枚举+排序+N 次求值） | ❌ **完全串行** |
| `LOCK_system_variables_hash` | `mysql_rwlock_t` | 仅"枚举双 hash"期间 | ✅ 读锁可并发 |
| `LOCK_global_system_variables` | mutex | 仅**单个变量**求值期间（N 次短持） | ❌ 串行但持时极短 |

关键事实（源码核实）：`LOCK_plugin_delete` 声明为 `extern mysql_mutex_t LOCK_plugin_delete`，注释原话"A mutex LOCK_plugin_delete must be acquired before calling plugin_del function."——它是**全局排他 mutex**，不是读写锁。

**所以并发查询 PFS 变量表会全部串行化在这一把锁上。** 竞争强度 = `查询频率 × 单次物化时长`：

- **锁获取频率 = 查询频率**：cache 不跨语句，每次查询都要物化一次 → 每次都要拿这把锁；
- **单次持有时长 = 完整物化时间**：遍历双 hash + 排序 + N 次求值 + 格式化，随变量数（含插件变量）增长——插件装得越多，持锁越久；
- **放大效应**：`SHOW VARIABLES` 还多一层派生表物化（临时表），status 全局表还要再遍历所有在线 THD；高并发下等待队列在 `LOCK_plugin_delete` 上堆积。

这就是"大量线程并发 `SHOW VARIABLES` 导致吞吐骤降"这类线上事故的机理：不是 CPU 打满，而是**线程排队在 `LOCK_plugin_delete` 上**。

实践含义：① 高频轮询 PFS 变量表（监控采集）要控制频率或在应用层缓存结果；② 直查 `SELECT` 比 `SHOW` 少一层临时表物化（但 PFS 表访问部分的锁竞争一样）；③ 装很多插件会拉长持锁时间，间接放大竞争。

> 这是设计上的**粗粒度锁换简单性与正确性**：用一把全局 mutex 保证"物化期间插件集合不变"，代价是可伸缩性。PFS 选择了正确性优先——这也和"cache 不跨语句复用"（见上）是同一个设计取向。
- status 全局表还要**再加一次持锁遍历所有在线 THD** 聚合（`visit_global(..., with_THDs=true)` → `do_for_all_thd` 逐线程 `add_to_status`），开销随活跃连接数增长。
- EXPLAIN 的 `rows` 估计来自 `get_row_count()`（= 变量总数，随插件加载变化）；`type` 列推断为：无谓词 `ALL`、点查 `range`/`ref`（取决于优化器成本判定，最终都落到线性过滤的 index 序列）。

#### 12 张变量表与 ACL

`global_variables`/`session_variables`/`variables_info`/`global_status`/`session_status` 用 `pfs_readonly_world_acl`（人人可读）；`variables_by_thread`/`user_variables_by_thread` 用 `pfs_readonly_acl`（**需显式授权**——能看到别人的会话变量是敏感能力）；`status_by_*`/`persisted_variables` 走常规授权。全表清单与语义见 [`variables.md` 的 H 节](../../../server/infra/variables.md#h-与-pfs--i_s-系统表的关系)。

---

### 1.10 ★ PFS 的表**不是固定的**：静态内置 + 插件动态注册

> ⚠️ 常见误解：把 `performance_schema` 的表数当成固定值（如"112 张"）。
> 实际上它是 **静态内置（随 server 启动就在）** + **插件/组件动态注册（装了才有）** 两部分之和。
> 这个区分直接决定一件事：**查不到某张表时，先确认插件装没装**——不是"表存在但没数据"，而是**压根没有这张表**。

#### 机制：`pfs_plugin_table_v1` 组件服务

PFS 把"注册一张表"的能力**作为组件服务暴露出去**，插件/组件调用它即可向 `performance_schema` 下新增表。

- 服务定义：`include/mysql/components/services/pfs_plugin_table_service.h`
- 表服务：`BEGIN_SERVICE_DEFINITION(pfs_plugin_table_v1)`
- 配套的列类型服务族（每张插件表要声明列的类型）：
  `pfs_plugin_column_{tiny,small,medium,integer,bigint,decimal,float,double}_v1`、`pfs_plugin_column_string_v2`

⇒ 这也解释了 README 1.1 里说的"**第七类：可扩展观测出口**"——企业版与第三方监控新增的观测能力，有一部分就是走这条路长出来的。

#### 8.0.39 里实际在用的插件表

| 插件 | 贡献的表 | 源码位置 |
|---|---|---|
| **clone** | `clone_status`、`clone_progress` | `plugin/clone/src/clone_status.cc`（表名赋值处：`m_table_name = "clone_status"` / `"clone_progress"`） |
| **Group Replication** | `replication_group_member_actions`<br>`replication_group_configuration_version`<br>`replication_group_communication_information` | `plugin/group_replication/src/perfschema/table_*.cc` |
| **NDB** | 若干 | `storage/ndb/plugin/ndb_pfs_init.cc` |

⚠️ 这些表**在 `storage/perfschema/` 下 grep 不到**——它们不在 PFS 引擎的代码里，而在各自插件的代码里。这是"按目录穷举会漏"的典型场景。

#### 静态 vs 动态：怎么区分

| | 静态内置表 | 插件动态表 |
|---|---|---|
| 定义位置 | `storage/perfschema/table_*.cc` | 各自插件目录（如 `plugin/clone/`、`plugin/group_replication/`） |
| 何时可见 | server 启动即有 | **插件/组件加载后才有** |
| 卸载后 | 仍在 | **消失** |
| 例子 | `events_waits_*`、`memory_summary_*`、`threads` | `clone_status`、`replication_group_*` |

#### 枚举自己实例上的 PFS 表（别照抄文档的表数）

```sql
-- 当前实例的 PFS 表总数
SELECT COUNT(*) AS pfs_table_count
  FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = 'performance_schema';

-- 列名出来看（想找"有没有 clone / replication_group 相关"）
SELECT TABLE_NAME
  FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = 'performance_schema'
   AND (TABLE_NAME LIKE '%clone%' OR TABLE_NAME LIKE '%replication_group%')
 ORDER BY TABLE_NAME;

-- 已加载的组件/插件（判断插件装没装）
SELECT * FROM mysql.component;
SHOW PLUGINS;
```

⇒ **排障顺序**：文档说有 `clone_progress` 但你查不到 → 先 `SELECT * FROM mysql.component` 确认 clone 插件是否加载，而不是怀疑权限或版本。

---
### 静态注册表：`all_shares[]`

`storage/perfschema/pfs_engine_table.cc:482-606`：

```cpp
static PFS_engine_table_share *all_shares[] = {
    &table_cond_instances::m_share,
    &table_error_log::m_share,
    &table_events_waits_current::m_share,
    ... /* 省略：约 100 行同类 &table_XXX::m_share 条目 */
    &table_keyring_component_status::m_share,
    nullptr};
```

遍历入口（`pfs_engine_table.cc:628-655`）：

```cpp
PFS_dynamic_table_shares pfs_external_table_shares;

/** Get all the core performance schema tables. */
void PFS_engine_table_share::get_all_tables(List<const Plugin_table> *tables) {
  PFS_engine_table_share **current;

  for (current = &all_shares[0]; (*current) != nullptr; current++) {
    tables->push_back((*current)->m_table_def);
  }
}

/** Initialize all the table share locks. */
void PFS_engine_table_share::init_all_locks() {
  PFS_engine_table_share **current;

  for (current = &all_shares[0]; (*current) != nullptr; current++) {
    thr_lock_init((*current)->m_thr_lock_ptr);
  }
}
```

DD 建表入口消费它（`ha_perfschema.cc:1379-1390`）：

```cpp
  PFS_engine_table_share::get_all_tables(tables);
  return false;
}
```

一个内置 share 的完整静态初始化（`table_sync_instances.cc:478-508`）——把"表定义 + 函数指针 + 锁"绑成一条：

```cpp
THR_LOCK table_cond_instances::m_table_lock;

Plugin_table table_cond_instances::m_table_def(
    /* Schema name */
    "performance_schema",
    /* Name */
    "cond_instances",
    /* Definition */
    "  NAME VARCHAR(128) not null,\n"
    "  OBJECT_INSTANCE_BEGIN BIGINT unsigned not null,\n"
    "  PRIMARY KEY (OBJECT_INSTANCE_BEGIN) USING HASH,\n"
    "  KEY (NAME) USING HASH\n",
    /* Options */
    " ENGINE=PERFORMANCE_SCHEMA",
    /* Tablespace */
    nullptr);

PFS_engine_table_share table_cond_instances::m_share = {
    &pfs_readonly_acl,
    table_cond_instances::create,
    nullptr, /* write_row */
    nullptr, /* delete_all_rows */
    table_cond_instances::get_row_count,
    sizeof(PFS_simple_index),
    &m_table_lock,
    &m_table_def,
    false, /* perpetual */
    PFS_engine_table_proxy(),
    {0},
    false /* m_in_purgatory */
};
```

### share 本体：一张函数指针表

`storage/perfschema/pfs_engine_table.h:353-393`：

```cpp
/**

  A PERFORMANCE_SCHEMA table share.
  This data is shared by all the table handles opened on the same table.
*/
struct PFS_engine_table_share {
  static void get_all_tables(List<const Plugin_table> *tables);
  static void init_all_locks();
  static void delete_all_locks();

  /** Get the row count. */
  ha_rows get_row_count() const;
  /** Write a row. */
  int write_row(PFS_engine_table *pfs_table, TABLE *table, unsigned char *buf,
                Field **fields) const;
  /** Table Access Control List. */
  const ACL_internal_table_access *m_acl;
  /** Open table function. */
  pfs_open_table_t m_open_table;
  /** Write row function. */
  pfs_write_row_t m_write_row;
  /** Delete all rows function. */
  pfs_delete_all_rows_t m_delete_all_rows;
  /** Get rows count function. */
  pfs_get_row_count_t m_get_row_count;
  /** Length of the @c m_pos position structure. */
  uint m_ref_length;
  /** The lock, stored on behalf of the SQL layer. */
  THR_LOCK *m_thr_lock_ptr;
  /** Table definition. */
  const Plugin_table *m_table_def;
  /** Table is available even if the Performance Schema is disabled. */
  bool m_perpetual;

  /* Interface to be implemented by plugin who adds its own table in PFS. */
  PFS_engine_table_proxy m_st_table;
  /* Number of table objects using this share currently. */
  std::atomic<int> m_ref_count;
  /* is marked to be deleted? */
  bool m_in_purgatory;
};
```

/* 【注】★ 结构里**没有** `m_name` 成员——表名从 `m_table_def->get_name()` 取。
   `m_in_purgatory` 是"软删除标记"（见下），配合 `m_ref_count` 实现安全卸载。 */

查找：`PFS_engine_table::find_engine_table_share()`（`pfs_engine_table.cc:701-725`）——**先查静态、再查动态**：

```cpp
/**
  Find a table share by name.
  @param name             The table name
  @return table share
*/
PFS_engine_table_share *PFS_engine_table::find_engine_table_share(
    const char *name) {
  DBUG_TRACE;
  PFS_engine_table_share *result;

  /* First try to find in native performance schema table shares */
  PFS_engine_table_share **current;

  for (current = &all_shares[0]; (*current) != nullptr; current++) {
    if (compare_table_names(name, (*current)->m_table_def->get_name()) == 0) {
      return *current;
    }
  }

  /* Now try to find in non-native performance schema tables shares */
  result = pfs_external_table_shares.find_share(name, false);

  // FIXME : here we return an object that could be destroyed, unsafe.
  return result;
}
```

/* 【注】末尾那句 FIXME 是上游自己的诚实标注：外部 share 可能在返回后被销毁。
   这正是 `m_ref_count` / `m_in_purgatory` 存在的理由（见下节注销流程）。 */

### 动态注册：不是类，是一个**组件服务**

★ `pfs_plugin_table` **不是类**（不存在 `class pfs_plugin_table`），8.0.39 的形态是组件服务 **`pfs_plugin_table_v1`**。服务定义（`include/mysql/components/services/pfs_plugin_table_service.h:485-497`）：

```cpp
/**
  Definition of pfs_plugin_table_v1 service and its methods.
*/
BEGIN_SERVICE_DEFINITION(pfs_plugin_table_v1)
DECLARE_METHOD(int, add_tables,
               (PFS_engine_table_share_proxy * *st_share,
                unsigned int share_count));

DECLARE_METHOD(int, delete_tables,
               (PFS_engine_table_share_proxy * *st_share,
                unsigned int share_count));
DECLARE_METHOD(unsigned int, get_parts_found, (PSI_key_reader * reader));
END_SERVICE_DEFINITION(pfs_plugin_table_v1)
```

插件传入的"share 代理"（`pfs_plugin_table_service.h:459-483`）：

```cpp
struct PFS_engine_table_share_proxy {
 public:
  /* Callback functions list of APIs */
  PFS_engine_table_proxy m_proxy_engine_table;

  /* Name of the table to be added */
  const char *m_table_name;
  /* Length of the table name */
  unsigned int m_table_name_length;

  /* Table Columns definition */
  const char *m_table_definition;
  unsigned int m_ref_length;

  /* Access allowed on the table */
  enum Access_control m_acl;

  delete_all_rows_t delete_all_rows;

  get_row_count_t get_row_count;
};
```

把代理翻译成真正的 share（`storage/perfschema/pfs_plugin_table.cc:267-343`，节选）：

```cpp
  /* Set acl */
  switch (proxy_share->m_acl) {
    case READONLY:
      share->m_acl = &pfs_readonly_acl;
      break;
    case TRUNCATABLE:
      share->m_acl = &pfs_truncatable_acl;
      break;
    case UPDATABLE:
      share->m_acl = &pfs_updatable_acl;
      break;
    case EDITABLE:
      share->m_acl = &pfs_editable_acl;
      break;
    default: /* Unknown ACL */
      share->m_acl = &pfs_unknown_acl;
      break;
  }

  /* Open table pointer to open a table with share provided */
  share->m_open_table = table_plugin_table::create;

  share->m_write_row = write_row;
  share->m_delete_all_rows = proxy_share->delete_all_rows;
  share->m_get_row_count = proxy_share->get_row_count;

  share->m_ref_length = proxy_share->m_ref_length;

  share->m_thr_lock_ptr = new THR_LOCK();

  share->m_table_def =
      new Plugin_table("performance_schema", proxy_share->m_table_name,
                       proxy_share->m_table_definition,
                       "ENGINE = 'PERFORMANCE_SCHEMA'", nullptr);
```

/* 【注】★ 插件表的 `m_open_table` 一律是 `table_plugin_table::create`——
   真正的行读写回调在 `m_st_table`（`PFS_engine_table_proxy`）里，由插件实现。
   这是 PFS 与插件之间的**稳定 ABI 边界**。 */

### 注销用"purgatory"两阶段

`storage/perfschema/pfs_plugin_table.cc:459-527`（节选）：

```cpp
  /* ============== CRITICAL SECTION 1 (begin) ================= */
  pfs_external_table_shares.lock_share_list();

  /* Check if any of the table, in the list, doesn't exist. */
  for (uint i = 0; i < share_list_count; i++) {
    ...
    PFS_engine_table_share *temp_share =
        pfs_external_table_shares.find_share(temp_st_share->m_table_name, true);

    if (temp_share != nullptr) {
      /* If share found in global list,
       *  - Move it to purgatory.
       *  - Add it to the list of shares to be removed.
       */
      temp_share->m_in_purgatory = true;
      share_list.push_back(temp_share);
    }
  }

  pfs_external_table_shares.unlock_share_list();
  /* ============== CRITICAL SECTION 1 ( end ) ================= */

  /* At this point, all 'to be removed' shares are moved to purgatory. So no
   * new thread would be able to search them in global shares list, therefore
   * no new query could be run on these tables.
   *
   * Now traverse share_list and drop tables using DD API.
   */
  for (auto *share : share_list) {
    if (drop_native_table_for_pfs(PERFORMANCE_SCHEMA_str.str,
                                  share->m_table_def->get_name())) {
      return 1;
    }
  }
  ...
  for (auto *share : share_list) {
    pfs_external_table_shares.remove_share(share);
    destroy_table_share(share);
  }
```

以及 `find_share` 里那句"跳过 purgatory"的判定（`pfs_engine_table.cc:871-884`）：

```cpp
  for (auto *it : shares_vector) {
    if ((compare_table_names(table_name, it->m_table_def->get_name()) == 0) &&
        (!it->m_in_purgatory || is_dead_too)) {
      return it;
    }
  }
  return nullptr;
```

/* 【注】★ 两阶段的意义：先把 share 标成 purgatory（**立刻对新查询不可见**），
   再用 DD API 落表删除，最后才真正 `delete`。这样"正在跑的查询"不会因为
   share 被提前释放而崩——`m_ref_count` 配合 `inc_ref_count/dec_ref_count`
   （`ha_perfschema.cc:966`、`:995`）就是这个用途。 */

动态 share 容器（`pfs_engine_table.h:395-426`）：

```cpp
class PFS_dynamic_table_shares {
 public:
  ...
  void lock_share_list() { mysql_mutex_lock(&LOCK_pfs_share_list); }

  void unlock_share_list() { mysql_mutex_unlock(&LOCK_pfs_share_list); }

  void add_share(PFS_engine_table_share *share) {
    mysql_mutex_assert_owner(&LOCK_pfs_share_list);
    shares_vector.push_back(share);
  }

  PFS_engine_table_share *find_share(const char *table_name, bool is_dead_too);

  void remove_share(PFS_engine_table_share *share);

 private:
  std::vector<PFS_engine_table_share *> shares_vector;
  mysql_mutex_t LOCK_pfs_share_list;
};

/* List of table shares added by plugin/component */
extern PFS_dynamic_table_shares pfs_external_table_shares;
```

★ **真名对照**：`PFS_engine_table_share::get_table_share()` / `find_table_share()` / `get_share_list()` / `pfs_table_registry` / `insert_table_share()` / `delete_table_share()` / `ha_perfschema::discover_table_structure` **全部不存在**（全仓 0 命中）。真实入口是 `PFS_engine_table::find_engine_table_share()`、`PFS_engine_table_share::get_all_tables()`、`all_shares[]` + `pfs_external_table_shares`、`pfs_add_tables_v1()` / `pfs_delete_tables_v1()`、`create_native_table_for_pfs()`（`sql/pfs_priv_util.h:29`）。

## 二、PFS 模块涉及的设计模式

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

### Strategy 的载体：每类插桩一个"服务指针 + 函数指针表"

★ `psi_service_v1` / `psi_service_v2` 与全局 `PSI_server` **在 8.0.39 都不存在**（全仓 0 命中）。真实形态是**按插桩类别拆分**：每类一个 `PSI_xxx_service_t *` 指针，指向一张函数指针表。以 mutex 为例（`include/mysql/psi/psi_mutex.h:45-84`）：

```cpp
/** Entry point for the performance schema interface. */
struct PSI_mutex_bootstrap {
  /**
    ABI interface finder.
    Calling this method with an interface version number returns either
    an instance of the ABI for this version, or NULL.
    @sa PSI_MUTEX_VERSION_1
    @sa PSI_MUTEX_VERSION_2
    @sa PSI_CURRENT_MUTEX_VERSION
  */
  void *(*get_interface)(int version);
};

#ifdef HAVE_PSI_MUTEX_INTERFACE

/**
  Performance Schema Mutex Interface, version 1.
  @since PSI_MUTEX_VERSION_1
*/
struct PSI_mutex_service_v1 {
  /** @sa register_mutex_v1_t. */
  register_mutex_v1_t register_mutex;
  /** @sa init_mutex_v1_t. */
  // @usage PSI_MUTEX_CALL(init_mutex)(params)
  init_mutex_v1_t init_mutex;
  /** @sa destroy_mutex_v1_t. */
  destroy_mutex_v1_t destroy_mutex;
  /** @sa start_mutex_wait_v1_t. */
  start_mutex_wait_v1_t start_mutex_wait;
  /** @sa end_mutex_wait_v1_t. */
  end_mutex_wait_v1_t end_mutex_wait;
  /** @sa unlock_mutex_v1_t. */
  unlock_mutex_v1_t unlock_mutex;
};

typedef struct PSI_mutex_service_v1 PSI_mutex_service_t;

extern MYSQL_PLUGIN_IMPORT PSI_mutex_service_t *psi_mutex_service;

#endif /* HAVE_PSI_MUTEX_INTERFACE */
```

/* 【注】★ 这样的服务指针共 **17 个**（`psi_mutex_service` / `psi_rwlock_service` /
   `psi_cond_service` / `psi_thread_service` / `psi_file_service` / `psi_statement_service` /
   `psi_memory_service` / `psi_mdl_service` / …）。所谓"PFS 的接口"不是一张大表，
   而是 **17 张小表**——这是 8.0 把 PSI 拆细后的形态。 */

### 空实现（Strategy 的"不插桩"那一档）：`mysys/psi_noop.cc`

★ 真名不是 `PSI_server_stub`（不存在），而是 `mysys/psi_noop.cc` 里的空服务表。文件头把 18 个 `HAVE_PSI_*_INTERFACE` 全部强制打开（`psi_noop.cc:29-54`）：

```cpp
/**
  @file mysys/psi_noop.cc
  Always provide the noop performance interface, for plugins.
*/

#define HAVE_PSI_MUTEX_INTERFACE
#define HAVE_PSI_RWLOCK_INTERFACE
#define HAVE_PSI_COND_INTERFACE
#define HAVE_PSI_FILE_INTERFACE
#define HAVE_PSI_THREAD_INTERFACE
...
```

典型空函数（`psi_noop.cc:92-113`）：

```cpp
static void register_thread_noop(const char *, PSI_thread_info *, int) {
  return;
}

static int spawn_thread_noop(PSI_thread_key, unsigned int,
                             my_thread_handle *thread,
                             const my_thread_attr_t *attr,
                             my_start_routine start_routine, void *arg) {
  return my_thread_create(thread, attr, start_routine, arg);
}

static PSI_thread *new_thread_noop(PSI_thread_key, unsigned int, const void *,
                                   ulonglong) {
  return nullptr;
}

static void set_thread_id_noop(PSI_thread *, ulonglong) { return; }
```

以及"可替换的全局指针"（`psi_noop.cc:268-281`）：

```cpp
static void end_mutex_wait_noop(PSI_mutex_locker *, int) { return; }

static void unlock_mutex_noop(PSI_mutex *) { return; }

static PSI_mutex_service_t psi_mutex_noop = {
    register_mutex_noop,   init_mutex_noop,     destroy_mutex_noop,
    start_mutex_wait_noop, end_mutex_wait_noop, unlock_mutex_noop};

struct PSI_mutex_bootstrap *psi_mutex_hook = nullptr;
PSI_mutex_service_t *psi_mutex_service = &psi_mutex_noop;

void set_psi_mutex_service(void *psi) {
  psi_mutex_service = (PSI_mutex_service_t *)psi;
}
```

/* 【注】★ 默认 `psi_mutex_service` 就指向空表——"没有 PFS 时不是 if 判断，
   而是**所有调用都落到一个空函数**"。这就是"零开销"的第一层。 */

### "零开销"的第二层：server 内直接静态调用，跳过函数指针

`include/pfs_mutex_provider.h:37-63`：

```cpp
#ifdef HAVE_PSI_MUTEX_INTERFACE
#if defined(MYSQL_SERVER) || defined(PFS_DIRECT_CALL)
#ifndef MYSQL_DYNAMIC_PLUGIN
#ifndef WITH_LOCK_ORDER

#include "my_inttypes.h"
#include "my_macros.h"
#include "mysql/psi/psi_mutex.h"

#define PSI_MUTEX_CALL(M) pfs_##M##_v1

void pfs_register_mutex_v1(const char *category, PSI_mutex_info_v1 *info,
                           int count);

PSI_mutex *pfs_init_mutex_v1(PSI_mutex_key key, const void *identity);

void pfs_destroy_mutex_v1(PSI_mutex *mutex);

PSI_mutex_locker *pfs_start_mutex_wait_v1(PSI_mutex_locker_state *state,
                                          PSI_mutex *mutex,
                                          PSI_mutex_operation op,
                                          const char *src_file, uint src_line);

void pfs_unlock_mutex_v1(PSI_mutex *mutex);

void pfs_end_mutex_wait_v1(PSI_mutex_locker *locker, int rc);
```

三条宏分支（`include/mysql/psi/mysql_mutex.h:67-74`）：

```cpp
#if defined(MYSQL_SERVER) || defined(PFS_DIRECT_CALL)
/* PSI_MUTEX_CALL() as direct call. */
#include "pfs_mutex_provider.h"  // IWYU pragma: keep
#endif

#ifndef PSI_MUTEX_CALL
#define PSI_MUTEX_CALL(M) psi_mutex_service->M
#endif
```

组件侧是第三条（`include/mysql/components/services/psi_mutex.h:36`）：

```cpp
#define PSI_MUTEX_CALL(M) mysql_service_psi_mutex_v1->M
```

★ **三档并存**：① server 内部 → **直接静态调用** `pfs_xxx_v1`（可内联、无间接跳转）；
② 组件 → 走组件服务指针；③ 插件 → 走 `psi_mutex_service` 函数指针（为的是**不硬编码符号**，
让插件二进制与实现解耦）。

官方对"零开销"的完整解释（`storage/perfschema/pfs.cc:797-820`）：

```
  When the performance schema is disabled at runtime, the @c m_psi pointer
  is @c NULL, so the extra overhead is one if statement.

  When the performance schema is enabled at runtime, the @c m_psi pointer
  points to the actual instrumentation added to the mutex,
  and two extra calls are present on the code path:
  - a call to @c pfs_start_mutex_wait_v1() before the operation
  - a call to @c pfs_end_mutex_wait_v1() after the operation

  Both these calls are static, and point directly to the
  performance_schema implementation.

  @section PFS_PSI_PLUGIN Instrumentation for a plugin

  Inside a plugin, @c PSI_MUTEX_CALL expands to a call using a function pointer.
  This is to decouple the plugin binary from the implementation,
  to avoid having hard coded references to symbols like
  pfs_start_mutex_wait_v1()
  from the plugin library (either shared or static).
```

### Bridge 的落点：版本分发，而不是"转换层"

★ `pfs_psi_v1_v2` / `psi_v1_v2` **不存在**（没有转换层文件）。真实的"版本桥接"是
`get_interface(version)` 分发（`storage/perfschema/pfs.cc:9666-9684`）：

```cpp
static void *get_mutex_interface(int version) {
  switch (version) {
    case PSI_MUTEX_VERSION_1:
      return &pfs_mutex_service_v1;
    default:
      return nullptr;
  }
}

static void *get_rwlock_interface(int version) {
  switch (version) {
    case PSI_RWLOCK_VERSION_1:
      return nullptr;
    case PSI_RWLOCK_VERSION_2:
      return &pfs_rwlock_service_v2;
    default:
      return nullptr;
  }
}
```

/* 【注】★ 注意 `get_rwlock_interface` 对 **v1 返回 nullptr** ——rwlock 的 v1 接口
   在 8.0.39 **已经不可用**（只有 mutex 还保留 v1）。这就是"版本化 ABI"能真正
   **废弃旧接口**的机制：不提供实现 = 让调用方自己失败。 */

v1 / v2 的实际签名差别（rwlock 是本树唯一保留双版本的 sync 插桩，
`include/mysql/components/services/bits/psi_rwlock_bits.h:279-291`）：

```cpp
/**
  Record a rwlock instrumentation unlock event.
  @param rwlock the rwlock instrumentation
*/
typedef void (*unlock_rwlock_v1_t)(struct PSI_rwlock *rwlock);

/**
  Record a rwlock instrumentation unlock event.
  @param rwlock the rwlock instrumentation
  @param op the unlock operation performed
*/
typedef void (*unlock_rwlock_v2_t)(struct PSI_rwlock *rwlock,
                                   enum PSI_rwlock_operation op);
```

同一份实现同时挂到"插件表"和"组件服务表"两处（`storage/perfschema/pfs.cc:9336-9352`）：

```cpp
PSI_mutex_service_v1 pfs_mutex_service_v1 = {
    /* Old interface, for plugins. */
    pfs_register_mutex_v1,   pfs_init_mutex_v1,     pfs_destroy_mutex_v1,
    pfs_start_mutex_wait_v1, pfs_end_mutex_wait_v1, pfs_unlock_mutex_v1};

SERVICE_TYPE(psi_mutex_v1)
SERVICE_IMPLEMENTATION(performance_schema, psi_mutex_v1) = {
    /* New interface, for components. */
    pfs_register_mutex_v1,   pfs_init_mutex_v1,     pfs_destroy_mutex_v1,
    pfs_start_mutex_wait_v1, pfs_end_mutex_wait_v1, pfs_unlock_mutex_v1};

PSI_rwlock_service_v2 pfs_rwlock_service_v2 = {
    /* Old interface, for plugins. */
    pfs_register_rwlock_v2,   pfs_init_rwlock_v2,
    pfs_destroy_rwlock_v2,    pfs_start_rwlock_rdwait_v2,
    pfs_end_rwlock_rdwait_v2, pfs_start_rwlock_wrwait_v2,
    pfs_end_rwlock_wrwait_v2, pfs_unlock_rwlock_v2};
```

/* 【注】★ 两张表指向**同一组函数**——插件 ABI 与组件服务在 PFS 侧是**同一实现的两个入口**，
   这是 8.0 从 plugin 迁移到 component 过程中的双轨设计，也是 Bridge 模式最实在的体现。 */

### 编译期摘除插桩：`PFS_SKIP_BUFFER_MUTEX_RWLOCK`

定义**默认开启**（`storage/innobase/include/sync0sync.h:73-81`）：

```cpp
#if defined UNIV_PFS_MUTEX || defined UNIV_PFS_RWLOCK

/* By default, buffer mutexes and rwlocks will be excluded from
instrumentation due to their large number of instances. */
#define PFS_SKIP_BUFFER_MUTEX_RWLOCK

/* By default, event->mutex will also be excluded from instrumentation */
#define PFS_SKIP_EVENT_MUTEX
#endif
```

三处消费点：

```cpp
#ifdef UNIV_PFS_MUTEX
/* Key to register autoinc_mutex with performance schema */
mysql_pfs_key_t autoinc_mutex_key;
mysql_pfs_key_t autoinc_persisted_mutex_key;
#ifndef PFS_SKIP_BUFFER_MUTEX_RWLOCK
mysql_pfs_key_t buffer_block_mutex_key;
#endif /* !PFS_SKIP_BUFFER_MUTEX_RWLOCK */
#endif
```

```cpp
#ifdef PFS_SKIP_BUFFER_MUTEX_RWLOCK
  LATCH_ADD_MUTEX(BUF_BLOCK_MUTEX, SYNC_BUF_BLOCK, PFS_NOT_INSTRUMENTED);
#else
  LATCH_ADD_MUTEX(BUF_BLOCK_MUTEX, SYNC_BUF_BLOCK, buffer_block_mutex_key);
#endif /* PFS_SKIP_BUFFER_MUTEX_RWLOCK */
```

```cpp
#ifndef PFS_SKIP_BUFFER_MUTEX_RWLOCK
    PSI_MUTEX_KEY(buffer_block_mutex, 0, 0, PSI_DOCUMENT_ME),
#endif /* !PFS_SKIP_BUFFER_MUTEX_RWLOCK */
```

/* 【注】★ 理由写在注释里："due to their **large number of instances**"——
   缓冲池有几十万个 block，每个都插桩会把 PFS 的实例数撑爆。这是"零开销"之外的
   另一条路：**不是让探针变便宜，而是干脆不装**。代价是 `buf_block` 的锁等待
   在 PFS 里**看不到**（表现为 `PFS_NOT_INSTRUMENTED`）。 */

## 三、插件系统涉及的设计模式

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

## 四、核心术语中英文对照（可观测性）

### 可观测性（Observability）

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

## 五、学术论文/理论工作索引（可观测性）

### 可观测性（Observability & Instrumentation）

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

## 六、设计哲学（PFS 部分）

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
| **Strategy（策略）** | GoF 1994 行为型模式 | PSI vtable：编译时/运行时切换不同埋点策略 |
| **Template Method（模板方法）** | GoF 1994 行为型模式 | PFS start→do→end 三部曲：宏定义了埋点骨架，业务代码填充步骤 2（好莱坞原则） |
| **Facade（外观）** | GoF 1994 结构型模式 | `mysql_mutex_t` 统一门面：隐藏平台锁 + PFS 插桩 + SAFE_MUTEX 三个子系统 |
| **Bridge（桥接）** | GoF 1994 结构型模式 | PSI 抽象接口 ↔ PFS 具体实现：抽象与实现独立编译，多版本 ABI 共存 |
| **Factory Method（工厂方法）** | GoF 1994 创建型模式 | `PSI_bootstrap::get_interface(version)`：根据版本号创建对应 ABI 实现 |

---

## 附录：推荐阅读文件清单（PFS 部分）

| 目的 | 文件路径 | 大小 |
|------|---------|------|
| PFS 引擎核心 | `storage/perfschema/pfs.cc` | 315KB |
| PFS event 基类定义 | `storage/perfschema/pfs_events.h` | 74 行 |
| PFS wait event 定义 | `storage/perfschema/pfs_events_waits.h` | 165 行 |
| PSI mutex 编织模板（宏 + inline 完整逻辑） | `include/mysql/psi/mysql_mutex.h` | 356 行 |
| PSI memory 编织模板 | `include/mysql/psi/mysql_memory.h` | — |
| ut::malloc 埋点 + PSI key 注册 | `storage/innobase/include/ut0new.h` | ~2200 行 |
| ut::malloc 实现 | `storage/innobase/ut/ut0new.cc` | — |
| PFS 插件声明 | `storage/perfschema/ha_perfschema.cc:1564` | — |

---

*本文基于 MySQL 8.0 源码分析撰写。内存分配器部分见 [allocator.md](../../memory/allocator.md)，读源码通用知识地图见 [README.md](../../../server/infra/README.md)。*
