# Performance Schema（PFS）：MySQL 内建可观测性框架

> 本篇聚焦 **PFS 可观测性**。内存分配器矩阵见 [memory.md](memory.md)；读源码通用知识地图与共享术语/论文见 [README.md](README.md)。
> 基于 MySQL 8.0 源码（`storage/perfschema`、`include/mysql/psi` 等模块）。

## 目录

- [摘要](#摘要)
- [一、Performance Schema](#一performance-schema)
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

PFS 不只是埋点框架——**8.0 起它还是 `SHOW VARIABLES` / `SHOW STATUS` 的实际实现者**（两条 SHOW 语句在语法分析阶段被重写成 `SELECT ... FROM performance_schema.global_variables / global_status`）。这也是"描述符与值分离"（见 [`variables.md`](variables.md)）在监控层的复用：三类变量（系统/状态/用户）都被统一成 `SHOW_VAR` 中间格式，走同一条管线。

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

#### 状态变量的聚合（do_materialize_global）

`GLOBAL = global_status_var + Σ(所有登记在 Global_THD_manager 的 THD)`，靠 `PFS_connection_status_visitor`：`visit_global()` 把全局基线 `add_to_status` 进局部 totals，`visit_THD()` 对每个 THD 累加。**含后台线程**（源码留 `// TODO: filter bg threads?`）。account/user/host 维度是**平行账本**（`PFS_status_stats` 遗留账本 + `do_materialize_client`），与全局通道隔离——`PFS_account::aggregate_status` 的注释："Never aggregate to global_status_var, because of the parallel THD -> global_status_var flow"（否则双倍计数）。

#### 12 张变量表与 ACL

`global_variables`/`session_variables`/`variables_info`/`global_status`/`session_status` 用 `pfs_readonly_world_acl`（人人可读）；`variables_by_thread`/`user_variables_by_thread` 用 `pfs_readonly_acl`（**需显式授权**——能看到别人的会话变量是敏感能力）；`status_by_*`/`persisted_variables` 走常规授权。全表清单与语义见 [`variables.md` 的 H 节](variables.md#h-与-pfs--i_s-系统表的关系)。

---

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

*本文基于 MySQL 8.0 源码分析撰写。内存分配器部分见 [memory.md](memory.md)，读源码通用知识地图见 [README.md](README.md)。*
