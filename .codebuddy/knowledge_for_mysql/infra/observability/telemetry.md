# MySQL Telemetry（OpenTelemetry 集成）深度解析

> 基于 **MySQL 9.7.0 社区版**源码（`components/telemetry/`，即 `component_telemetry`），对照 8.0.39 工作区源码说明演进。
>
> **边界**：本篇讲 telemetry **机制本身**。PFS 框架见 [`pfs/pfs.md`](pfs/pfs.md)；状态变量（metrics 的数据源）见 [`status_variables.md`](status_variables.md)；错误日志 sink pipeline 见 [`logging/error_log.md`](logging/error_log.md)。
>
> 不熟悉 OpenTelemetry 的读者，**建议先看「概述 → 术语 30 秒」**，否则后面会卡在 span / traceparent 这些概念上。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [可观测性](#可观测性)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

让 MySQL 把内部的观测数据（语句执行、计数器、错误日志）**主动推送**到 OpenTelemetry 生态，从而和应用自身的调用链、指标、日志放在一起看。

### 用途

**没有它的时候**，MySQL 的观测数据被困在实例内部，只能"拉"：

```
应用 ──SQL──> MySQL ──> 内部表（PFS / sys / 慢日志 / 错误日志）
                              └─ 要靠人主动去查
```

问题：应用有自己的 APM 调用链，MySQL 有自己的 PFS 表，**两边对不上号**。一条慢查询，你没法直接跳到"是哪个用户请求触发的"。

**有了它之后**：

```
应用 ──SQL（带 traceparent）──> MySQL ──OTLP/HTTP──> Collector ──> Jaeger / Grafana
       └──────── 自己的 span ────────┘                    │
                    ↑ 共享同一个 trace_id ─────────────────┘
                      可以拼成一条完整链路
```

⇒ 观测数据从"拉"变成"**推**"，且能跟应用链路**拼在一起**。

### ★ 与 OpenTelemetry 的关系

**OpenTelemetry（OTel）不是 MySQL 的东西**——它是 CNCF 的厂商中立可观测性标准（由 OpenTracing 与 OpenCensus 合并而来），定义了：三类信号的数据模型、统一的传输协议（OTLP）、上下文传播格式（W3C Trace Context）、以及各语言的 SDK。

**MySQL 在其中的位置是「被观测方 / 数据源」**：

```
┌─ OpenTelemetry 生态（标准 + 实现，与 MySQL 无关）──────────────┐
│  规范：数据模型 / OTLP / W3C Trace Context / 语义约定          │
│  SDK：opentelemetry-cpp 1.23.0（MySQL 内嵌的第三方库）         │
└────────────────────────┬──────────────────────────────────────┘
                         │ MySQL 只是"用"它
┌────────────────────────▼──────────────────────────────────────┐
│  MySQL 负责的部分：                                            │
│   · 采集（PFS 埋点）                                           │
│   · 按 OTel 数据模型造 span / metric / log record              │
│   · 按 OTLP 发出去                                             │
└────────────────────────┬──────────────────────────────────────┘
                         │ OTLP/HTTP
┌────────────────────────▼──────────────────────────────────────┐
│  MySQL **不负责**：Collector、存储、展示、告警                  │
│  （Jaeger / Prometheus / Loki / Grafana …由你自己搭）          │
└───────────────────────────────────────────────────────────────┘
```

**具体采用了 OTel 的哪几部分**：

| OTel 规范 | MySQL 的落点 | 采用程度 |
|---|---|---|
| 数据模型（span / metric / log record） | 直接用 SDK 类型 | ✅ 完全采用 |
| **W3C Trace Context**（traceparent） | `tm_propagation.cc` | ✅ 采用，但**只入向**（`Set()` 是 `assert(false)`） |
| **OTLP** 传输协议 | `setup_otel_otlp_http_exporter` | ⚠️ **只支持 HTTP**（json / protobuf），**gRPC 没有** |
| OTel SDK | `opentelemetry-cpp 1.23.0` 内嵌于 `extra/` | ✅ 直接用 |
| 语义约定（server span 用 `kServer`） | `tm_stmt_notify_qa` | ✅ 采用 |

**明确没有采用 / 没接线的部分**（这些是重要的边界）：

| OTel 能力 | MySQL 现状 |
|---|---|
| gRPC exporter | ❌ 代码中不存在（测试环境 4317 标 "not used"） |
| 采样配置（`OTEL_TRACES_SAMPLER`） | ❌ 全量，无变量、不支持环境变量 |
| 上下文 Inject（往下传 traceparent） | ❌ 只 Extract |
| `OTEL_*` 环境变量配置 | ❌ 全部走 MySQL 系统变量，C++ 直接读全局变量 |
| `compression` | ⚠️ 有变量但**未接线** |

⇒ **一句话**：MySQL 是 OTel 生态里的一个**数据源**（而且是相当标准的一个），它把本来只写在自己表里的观测数据，翻译成 OTel 的标准格式发出去。**但它不是 OTel 的实现者，也不管数据发出去之后的事。**

### 与 PFS 的关系（顺带说清）

| | PFS | Telemetry |
|---|---|---|
| 数据存在哪 | **MySQL 内部表** | 发到**外部** |
| 消费方式 | 拉（你查） | 推（MySQL 发） |
| 是否 OTel 标准 | 否，MySQL 自有体系 | 是 |
| 关系 | **同一份采集，两条出路**（详见「核心实现」） |

⇒ Telemetry **寄生在 PFS 的 statement locker 上**，两者互补并行，**不是替代**。

### 术语 30 秒（不熟悉 OTel 必读）

| 术语 | 是什么 | 本文对应 |
|---|---|---|
| **span** | 一段有起止的工作单元 | 一条 SQL、一个会话、一次启停 |
| **trace** | 由 span 组成的树 | 一条 SQL 及其子语句（存储过程内的语句是子 span） |
| **trace_id / span_id** | span 的身份（W3C 标准，128/64 位） | 由 OTel SDK 生成 |
| **traceparent** | 上下游传递 trace 上下文的载体 | 应用通过 **query attribute** 传给 MySQL |
| **OTLP** | OTel 的传输协议 | MySQL 用 **HTTP** 版（不是 gRPC） |
| **Collector** | 接收 OTLP 的中转站 | 你自己部署，MySQL 不知道它存在 |
| **BSP / BLRP** | 批量处理器（攒批再发） | `otel_bsp_*` / `otel_blrp_*` 变量 |

### 版本演进

| 版本 | 变化 |
|---|---|
| 8.0.33 | 引入 `mysql_server_telemetry_traces_v1` 服务（PFS 侧埋点与回调框架） |
| 8.0 / 8.4 | **只有埋点框架，没有导出组件**。社区版唯一消费端是测试组件（写文件日志）；真正的 OTLP exporter 属 **MySQL Enterprise Edition** |
| **9.7** | ★ **`components/telemetry/`（= `component_telemetry`）进入社区版**，官方手册删掉企业版声明；补齐 metrics、logs 两个服务与完整 OTLP 导出 |
| 9.7 之后 | 26.7（新版本命名）延续该形态 |

> **8.0 → 9.7 的本质差别不在 MySQL 侧埋点，而在于有没有把 OTel SDK 接进来**（详见「理论基础 → 设计思想与权衡」）。

---

## 理论基础

### 设计思想与权衡

#### ① 为什么是"PFS 埋点 + 组件造 span + SDK 导出"三层，而不是 MySQL 自己实现一套

MySQL 只做了前两层，**第三层直接站在 opentelemetry-cpp 上**（1.23.0，源码内嵌于 `extra/`）：

```
① PFS 埋点层     在语句各时点"喊一嗓子"，不知道什么是 span
② 组件层          造 span、维护父子、翻译数据，不知道怎么发出去
③ OTel SDK 层     生成 ID、攒批、发 OTLP，完全不知道 MySQL 存在
```

**权衡**：
- **放弃**：自己实现 trace id 生成、W3C 解析、OTLP 编码、批量发送、重试、TLS
- **代价**：引入一个重量级第三方依赖（protobuf ≥3.12.4、curl ≥7.28.0），构建时间显著变长；且不满足依赖时组件**静默跳过**（`OPENTELEMETRY_CPP_AVAILABLE=FALSE` → CMake `RETURN()`），某些精简构建里可能根本没有 `component_telemetry.so`
- **收益**：W3C 标准合规、生态兼容（任何支持 OTLP 的后端都能接）

⇒ 这解释了 8.0 为何只有"插座"：**8.0.39 的组件想生成 trace_id 都生成不了**（没有 SDK）。

#### ② 为什么用 query attribute 传 traceparent，而不是新增 SQL 语法或协议字段

源码无直接说明，但从设计可反推（标注为推断）：

| 方案 | 为什么不选 / 为什么选 |
|---|---|
| 新增 SQL 语法（如 `SELECT /*+ trace */`） | 会进 digest、污染 prepared statement 缓存 |
| 新增协议字段 | 要改客户端/服务端协议，破坏兼容性 |
| **query attribute** ✅ | 走已有的 `bind_parameter_values`（协议已支持）；**不进 digest**（实测 `SELECT 4` 的 digest 仍是 `SELECT ?`）；天然 per-statement 粒度；可携带任意业务上下文 |

⇒ 而且 qa 只在**顶层语句**有意义（源码断言 `assert(!with_query_attributes || m_stmt_stack.size() == 1)`），这与 OTel 的 "root span 由入口决定" 语义天然对齐。

#### ③ 为什么顶层语句用 Span **Link** 而不是 parent 关联会话 span

这是全文最微妙的设计点。顶层语句同时有两个"可能的父亲"：

- **外部 traceparent**（应用传来的，跨进程链路）
- **session span**（本连接的会话 span）

代码选择：**parent 位让给外部 traceparent**，session span 降级为 **Link**（弱关联）。

**权衡**：跨进程链路续接（回答"这条 SQL 属于哪个用户请求"）比"属于哪个连接"（回答"这个连接总共做了什么"）更有价值。Link 保留了后者的可追溯性但不破坏前者的树形结构。

#### ④ 为什么不做采样

`TracerProviderFactory::Create` 不传 sampler ⇒ 默认 `ParentBased(AlwaysOn)` **全量**。源码里 `Sampler` / `sample_rate` **0 命中**。

**这是明确的简化取舍**：采样策略交给 **Collector 侧**（Collector 支持 tail sampling，且能在看到完整 trace 后再决定）。MySQL 侧只给一个 ON/OFF 二值开关。

⚠️ **代价落到具体后果**：`trace_enabled=ON` 时**每条语句都产生 span**——生产全量开必须评估，且降采样只能在 Collector 做。

#### ⑤ 为什么 `metrics_enabled` 是只读的

官方注释：

> *"Installing / uninstalling the meter providers, with the associated metric readers and expporters, is an heavy operation… Note that individual meters can be enabled / disabled dynamically instead, by using: UPDATE performance_schema.setup_meters SET ENABLED = …"*

⇒ 整子系统开关太重（要建/拆 Provider + Reader + Exporter），所以只提供**细粒度**的动态开关（单个 meter）。

### 理论溯源

| 理论 / 标准 | 源码落点 |
|---|---|
| **W3C Trace Context**（trace_id/span_id/traceparent 的编码与传播） | `tm_propagation.cc` 的 `QueryAttributeTextMapCarrier` + `Propagator::Extract` |
| **OpenTelemetry 语义约定**（server span 用 `SpanKind::kServer`） | `tm_stmt_notify_qa` 里顶层 `options.kind = SpanKind::kServer` |
| **TextMapCarrier 模式**（把任意键值载体适配成 propagator 的输入） | `QueryAttributeTextMapCarrier::Get()` |
| **BatchSpanProcessor**（攒批异步导出，避免阻塞业务线程） | `setup_otel_batch_processor()` |

### 算法与数据结构

| 结构 | 用途 | 为什么选它 |
|---|---|---|
| `std::vector<shared_ptr<Span>> m_stmt_stack` | span 栈，表达语句嵌套 | 直接对应调用栈的 push/pop，父子关系天然成立 |
| `m_depth`（计数器） | **退化为一致性校验** | 9.7 起父子由 `SpanContext` 表达，depth 只用于断言栈深对不对 |
| `PFS_cacheline_atomic_ptr<telemetry_t*> g_telemetry` | 全局单槽 | 缓存行对齐避免伪共享；单槽源于"同时只允许一个组件" |
| 三个 MeterProvider | 三档采集频率 | OTel 的 `PeriodicExportingMetricReader` **一个 reader 只有一个固定 interval**，要不同节奏只能建多个 Provider |

### 他库对比与演进动机

| 数据库 | 做法 | 与 MySQL 的差异 |
|---|---|---|
| **PostgreSQL** | 无内置 OTel；靠扩展（`pg_stat_statements` + 外部 exporter）或 contrib 生态 | PG 走"生态扩展"，MySQL 9.7 走"内核内置组件" |
| **Oracle** | 企业版内置，且不开放标准协议导出 | MySQL 选了**厂商中立**的 OTLP |
| **MariaDB** | 无对应能力 | — |
| **MySQL 自身 8.0/8.4** | 只有埋点框架，导出属企业版 | 9.7 才下放社区版 |

⇒ **演进动机**：可观测性生态逐渐收敛到 OpenTelemetry 标准，数据库作为链路的一环必须能"说 OTLP"。

---

## 核心实现

### 主链路

一条 SQL 从进入到 span 发出的完整路径：

```
客户端发 SQL（可带 query attribute traceparent）
  │
  ├─ PFS: pfs_get_thread_statement_locker_vc
  │        └─> tm_stmt_start()              ★ 顶层：不建 span，只记"开始了"
  │
  ├─ PFS: pfs_notify_statement_query_attributes_vc
  │        └─> tm_stmt_notify_qa()          ★ 建 span 的真正位置
  │              ├─ tm_set_parent_from_qa()   从 qa 提取外部 traceparent
  │              ├─ options.parent = 外部 SpanContext（若有）
  │              ├─ session span → Span Link
  │              └─ StartSpan("stmt")         ← trace_id/span_id 由 SDK 生成
  │
  ├─ （若语句内有存储过程/触发器）
  │     └─> tm_stmt_start()（嵌套分支）
  │           options.parent = m_stmt_stack.back()->GetContext()
  │           └─ StartSpan("stmt", kInternal)   ← 真父子
  │
  ├─ PFS: pfs_end_statement_vc
  │        └─> tm_stmt_end()
  │              ├─ set_span_stmt_data()   填 ~40 个 mysql.* attribute
  │              ├─ SetStatus(Ok / Error)
  │              └─ otel_span->End()
  │
  └─ OTel SDK: BSP 攒批 → OTLP/HTTP → Collector
```

> ★ 关键：**span 不是在"语句开始"建的，而是在"拿到 query attributes"时建的**。因为要等 traceparent 才知道父亲是谁。

### 一、会话与语句生命周期

#### 1.0 `Session_data` 的一生：session span 在哪创建

会话级状态挂在 THD 的一个 slot 上（`mysql_thd_store`，slot 名 `telemetry`）。创建与销毁：

```cpp
telemetry_session_t *tm_session_create() {
  MYSQL_THD thd = nullptr;

  const int rc = current_thd_srv->get(&thd);

  if (rc != 0) {
    return nullptr;
  }

  assert(thd != nullptr);

  // ★ 幂等：已存在就复用（get 非 null 时跳过 create）
  //   ⇒ 对应组件"重装"场景——同一 THD 可能经历多次 session 创建
  Session_data *data = Session_data::get(thd);
  if (data == nullptr) {
    data = Session_data::create(thd, sv_trace_enabled);   // 创建时快照一次 m_trace

    if (data != nullptr) {
      Session_data::set(thd, data);
      // 维护全局计数 g_session_count——这是卸载等待的真值来源（见第七节）
      inc_session_count();
#ifdef DEBUG_SESSION_COUNT
      debug_inc_session_count("tm_session_create", thd, nullptr);
#endif /* DEBUG_SESSION_COUNT */
    } else {
      return nullptr;
    }
  }

  assert(data->m_depth == 0);
  assert(!data->m_used_in_telemetry);
  data->m_used_in_telemetry = true;
  // 复用路径也刷新一次；此外每条顶层语句在 tm_stmt_start 里还会再采样（见 1.1）
  // ⇒ 会话级与语句级各有一个 m_trace 副本
  data->m_trace = sv_trace_enabled;
  auto *session = reinterpret_cast<telemetry_session_t *>(data);
  return session;
}

void tm_session_destroy(telemetry_session_t *session) {
  auto *data = reinterpret_cast<Session_data *>(session);
  assert(data != nullptr);
  assert(data->m_depth == 0);
  THD *thd = data->m_thd;

  assert(thd != nullptr);
  assert(data->m_used_in_telemetry);
  data->m_used_in_telemetry = false;
  // ★ 先断开 slot 再收尾 ⇒ 防止收尾期间这个 session 又被取到
  Session_data::set(thd, nullptr);
  data->close();               // ← session span 真正 End() 的地方
  Session_data::destroy(data);
  dec_session_count();
#ifdef DEBUG_SESSION_COUNT
  debug_dec_session_count("tm_session_destroy", thd, nullptr);
#endif /* DEBUG_SESSION_COUNT */
}
```

> ★ 注意：**真正的 session span 创建不在上面两个回调里**，而在 PFS 通知回调 `tm_session_connect()`——它建名为 `"session"` 的 span，kind `kInternal`，并打上线程与连接属性。

#### 1.1 `tm_stmt_start()` —— 顶层不建 span

```cpp
telemetry_locker_t *tm_stmt_start(telemetry_session_t *session,
                                  uint64_t *flags) {
  auto *data = reinterpret_cast<Session_data *>(session);
  assert(data != nullptr);

  if (data->m_depth == 0) {
    /* Top level statement. */
    data->m_depth++;
    /* Delay span creation on query attributes. */
    data->m_query_attributes_seen = false;
    /* Re capture the trace flag at every top level statement */
    data->m_trace = sv_trace_enabled;
  } else {
    if (!data->m_query_attributes_seen) {
      /*
        Under normal conditions (no degraded instrumentation),
        the SQL layer should notify attributes properly.
        Please fix the root cause if this fails.
      */
      assert(false);
      /*
        Under stress or with incomplete instrumentation,
        ignore this (malformed) statement entirely.
      */
      return nullptr;
    }

    /* Nested statement. */
    data->m_depth++;

    if (data->m_trace && (data->m_session_tracer != nullptr)) {
      if (!data->m_stmt_stack.empty()) {
        opentelemetry::trace::StartSpanOptions options;
        options.kind = opentelemetry::trace::SpanKind::kInternal;
        options.parent = data->m_stmt_stack.back()->GetContext();
        opentelemetry::nostd::shared_ptr<opentelemetry::trace::Span> otel_span;

        otel_span = data->m_session_tracer->StartSpan(k_stmt, options);
        data->m_stmt_stack.push_back(std::move(otel_span));
      }
    }
  }

  auto *locker = reinterpret_cast<telemetry_locker_t *>(data);
  *flags = TRACE_STATEMENTS;
  return locker;
}
```

**逐段解释**：

- **顶层分支（`m_depth == 0`）只做三件事**：递增 depth、把 `m_query_attributes_seen` 置 false、**重新采样 `sv_trace_enabled`**。
  ★ 不建 span——因为 traceparent 还没到。
  ★ "每条顶层语句重新采样开关"意味着：**改 `trace_enabled` 不需要重连，下一条语句就生效**。
- **守卫分支**：嵌套语句开始前若 `m_query_attributes_seen` 仍为 false，说明 SQL 层没有正常通知 qa（降级埋点）。源码先 `assert(false)`（debug 下暴露），release 下 **直接返回 nullptr 丢弃整条语句**——宁可不 trace，也不产出结构错误的 span。
- **嵌套分支**：`options.parent = m_stmt_stack.back()->GetContext()` —— 这是**真父子关系**，用栈顶 span 的 `SpanContext` 作 parent，kind 为 `kInternal`。
- **返回值的障眼法**：返回的 `telemetry_locker_t*` 其实是 `Session_data*` 强转来的（`reinterpret_cast`）。PFS 侧只当它是 opaque 句柄。
- **★ `*flags = TRACE_STATEMENTS` 无条件设置**：这是"强制追踪"——告诉 PFS"这条语句我要完整数据"，**即使 `setup_instruments` 里 `statement/%` 全关**，PFS 的语句指标/digest/文本/线程内存采集也会被拉起来。

#### 1.2 `tm_stmt_notify_qa()` —— ★ 建 span 的真正位置（全篇最核心）

```cpp
telemetry_locker_t *tm_stmt_notify_qa(telemetry_locker_t *locker,
                                      bool with_query_attributes,
                                      uint64_t * /* flags */) {
  auto *data = reinterpret_cast<Session_data *>(locker);
  assert(data != nullptr);

  if (data->m_depth > 1) {
    /* Ignore spurious calls for nested statements */
    return locker;
  }

  /* Only top level statement may have query attributes. */
  assert(data->m_depth == 1);

  /* Only one notification expected. */
  assert(!data->m_query_attributes_seen);
  data->m_query_attributes_seen = true;

  if (data->m_trace && (data->m_session_tracer != nullptr)) {
    opentelemetry::trace::StartSpanOptions options;
    opentelemetry::nostd::shared_ptr<opentelemetry::trace::Span> otel_span;

    if (data->m_stmt_stack.empty()) {
      options.kind = opentelemetry::trace::SpanKind::kServer;

      if (with_query_attributes) {
        tm_set_parent_from_qa(options, data);
      }

      if (data->m_session_span != nullptr) {
        const opentelemetry::trace::SpanContext &session_context =
            data->m_session_span->GetContext();

        static const std::map<std::string, std::string> link_map;

        const std::pair<opentelemetry::trace::SpanContext,
                        std::map<std::string, std::string>>
            link{session_context, link_map};

        const std::vector<std::pair<opentelemetry::trace::SpanContext,
                                    std::map<std::string, std::string>>>
            links = {link};

        static const std::map<std::string, std::string> no_attrs = {};

        otel_span =
            data->m_session_tracer->StartSpan(k_stmt, no_attrs, links, options);
      } else {
        otel_span = data->m_session_tracer->StartSpan(k_stmt, options);
      }
    } else {
      options.kind = opentelemetry::trace::SpanKind::kInternal;
      options.parent = data->m_stmt_stack.back()->GetContext();

      otel_span = data->m_session_tracer->StartSpan(k_stmt, options);
    }

    data->m_stmt_stack.push_back(std::move(otel_span));
  }

  return locker;
}
```

**逐段解释**：

- **前三道断言确立契约**：`m_depth > 1` 直接返回（嵌套语句不该收到 qa 通知）；`m_depth == 1`；`m_query_attributes_seen` 只能从 false 翻到 true 一次。**这三条断言正是「为什么 span 必须在这里建」的形式化表达**。
- **★ `options.kind = SpanKind::kServer`（顶层）**：符合 OTel 语义约定——服务端接收请求。8.0.39 完全没有这个概念。
- **★ `tm_set_parent_from_qa()` 只在 `with_query_attributes` 为真时调用**：没有 qa 就没有外部 parent，span 成为新 trace 的根。
- **★ Span Link 分支**（`m_session_span != nullptr`）：
  - `link{session_context, link_map}` —— 把会话 span 的 `SpanContext` 作为 **Link** 而非 parent
  - 调用的是 `StartSpan(k_stmt, no_attrs, links, options)` 这个**带 links 参数的重载**
  - ⇒ 顶层 span 因此**同时具备**：parent = 外部 traceparent（若有），Link = 本会话 span
  - **为什么必须这样**：`options.parent` 已被外部 traceparent 占用，一个 span 只能有一个 parent，所以会话 span 只能当 Link。这正是前文「设计思想与权衡 ③」的代码落点。
- **兜底分支**（`m_stmt_stack` 非空）：视为嵌套，kind 改 `kInternal` 并用栈顶作 parent。
- **注意 `m_trace && m_session_tracer != nullptr` 双重判断**：`m_trace` 是本语句的开关快照，`m_session_tracer` 是**构造时快照的 tracer 指针**——后者防止 UNINSTALL 时 `g_tracer` 被置空导致竞态。

#### 1.3 `tm_stmt_end()` —— 填数据并结束

```cpp
void tm_stmt_end(telemetry_locker_t *locker, telemetry_stmt_data_t *stmt_data) {
  auto *data = reinterpret_cast<Session_data *>(locker);
  assert(data != nullptr);

  assert(data->m_depth > 0);

  /*
    For the top level statement,
    a call to tm_stmt_notify_qa() is mandatory.
  */
  assert(data->m_depth != 1 || data->m_query_attributes_seen);

  if (data->m_stmt_stack.size() >= data->m_depth) {
    const opentelemetry::nostd::shared_ptr<opentelemetry::trace::Span>
        otel_span = data->m_stmt_stack.back();

    data->m_stmt_stack.pop_back();

    set_span_stmt_data(otel_span, stmt_data);

    otel_span->End();
  }

  data->m_depth--;
}
```

**逐段解释**：

- **`m_stmt_stack.size() >= m_depth` 是不变量检查**：栈深必须≥当前语句深度，否则说明建 span 与结束 span 不配对（例如语句被丢弃过），此时**不 pop、不 End**，只递减 depth。这是防止栈错乱的防御式写法。
- **`set_span_stmt_data()`** 把 PFS 给的 `telemetry_stmt_data_t` 翻译成 ~40 个 `mysql.*` attribute（见下节）。
- **顺序很重要**：先 pop 再 `End()`——`End()` 会触发 SDK 记录结束时间并交给 BSP，栈必须先清理。
- **断言 `m_depth != 1 || m_query_attributes_seen`**：顶层语句必须走过 notify_qa，与 1.1 的守卫呼应。

#### 1.4 span 上到底带什么

`set_span_stmt_data()` 写入的 attribute 与 `events_statements_*` 的列**一一对应**：

| 类别 | attribute |
|---|---|
| 标识 | `mysql.event_name`、`mysql.current_schema`、`mysql.digest_text` |
| SQL | `mysql.sql_text`（⚠️ 仅当 `query_text_enabled`，**默认 ON**） |
| 计时 | `mysql.lock_time`、`mysql.cpu_time` |
| 行数 | `mysql.rows_sent` / `rows_examined` / `rows_affected` |
| 错误 | `mysql.sql_errno` / `sqlstate` / `message_text` / `error_count` / `warning_count` |
| 临时表/排序 | `mysql.created_tmp_tables` / `created_tmp_disk_tables` / `sort_*` |
| 索引质量 | `mysql.no_index_used` / `no_good_index_used` / `select_*` |
| 内存 | `mysql.max_controlled_memory` / `max_total_memory` |

会话级另有 `mysql.processlist_id` / `thread_id` / `user` / `host`，以及**连接属性** `mysql.session_attr.<名>`。

错误判定：`if (stmt_data->m_error_count == 0) SetStatus(kOk); else SetStatus(kError);`

⚠️ OTel 属性不支持 uint64，源码做了截断防护：

```cpp
int64_t big_attribute(unsigned long long value) {
  // Do not pass size_t to SetAttribute(), make EXPLICIT type conversion.
  if (value > std::numeric_limits<int64_t>::max()) return std::numeric_limits<int64_t>::max();
  return static_cast<int64_t>(value);
}
```

### 二、跨进程链路续接：traceparent 怎么进来

#### 2.1 提取入口

```cpp
void tm_set_parent_from_qa(opentelemetry::trace::StartSpanOptions &options,
                           Session_data *data) {
  if (data->m_thd != nullptr) {
    const QueryAttributeTextMapCarrier carrier(data->m_thd);
    QueryAttributeTextMapPropagator propagator;
    opentelemetry::context::Context context;

    auto new_context = propagator.Extract(carrier, context);
    options.parent = opentelemetry::trace::GetSpan(new_context)->GetContext();
  }
}
```

**解释**：这是标准的 OTel 用法——**Carrier**（数据源）+ **Propagator**（解析器）。MySQL 只是把"query attributes"适配成 Carrier，解析逻辑完全交给 SDK（因此 W3C 格式 `00-<32hex>-<16hex>-<2hex>` 的解析是标准行为）。

#### 2.2 Carrier 的实现：只进不出

```cpp
QueryAttributeTextMapCarrier::QueryAttributeTextMapCarrier(MYSQL_THD thd) : m_thd(thd) {
  /* trace_parent + trace_state */
  m_seen = new std::vector<my_h_string>(2);      // ← 正好 2 个 key
}

opentelemetry::nostd::string_view QueryAttributeTextMapCarrier::Get(
    opentelemetry::nostd::string_view key) const noexcept {
  int rc = qa_iter_srv->create(m_thd, null_terminated_key.data(), &iterator);
  if (rc != 0) { /* Not found. */ return value; }
  (void)qa_iter_srv->get_type(iterator, &attr_type);
  if (attr_type != MYSQL_TYPE_STRING) goto done;
  rc = qa_string_srv->get(iterator, &attr_value);
  rc = string_get_data_srv->get_data(attr_value, &str_data, &str_length, &charset);
  m_seen->push_back(attr_value);   // 生命周期托管，因为只能返回 string_view
  value = opentelemetry::nostd::string_view(str_data, str_length);
  ...
}
```

```cpp
void QueryAttributeTextMapCarrier::Set(...)  noexcept { assert(false); }
bool QueryAttributeTextMapCarrier::Keys(...) noexcept { assert(false); return false; }
```

**逐段解释**：

- **`Get()` 只能返回 `string_view`（非拥有）**，但 qa 值由 MySQL 分配 ⇒ 用 `m_seen` 数组**托管生命周期**，保证 view 在 propagator 使用期间有效。这是 C++ 里典型的"视图 + 外部持有"权衡。
- **`attr_type != MYSQL_TYPE_STRING` 则跳过**：traceparent 必须是字符串类型。
- **★ `Set()` / `Keys()` 直接 `assert(false)`**：MySQL **只支持 Extract（入向），不支持 Inject（出向）**。语义上 MySQL 是链路终点（它不会再去调用别的带 trace 的服务），所以不该注入。

> ⚠️ 精确 key 字符串定义在未获取的 `tm_propagation.h` 中；从 `/* trace_parent + trace_state */` 与 `m_seen(2)` **推断**为标准的 `traceparent` / `tracestate`。

### 三、导出管线：从 span 到 Collector

#### 3.1 exporter 装配

```cpp
setup_otel_otlp_http_exporter(bool json) {
  opentelemetry::exporter::otlp::OtlpHttpExporterOptions options;
  opentelemetry::exporter::otlp::OtlpHttpExporterRuntimeOptions runtime_options;

  if (sv_otel_exporter_otlp_traces_endpoint == nullptr) {
    return nullptr;
  }

  if (strlen(sv_otel_exporter_otlp_traces_endpoint) == 0) {
    return nullptr;
  }

  options.url = sv_otel_exporter_otlp_traces_endpoint;

  if (sv_otel_exporter_otlp_traces_certificates != nullptr) {
    if (strlen(sv_otel_exporter_otlp_traces_certificates) != 0) {
      options.ssl_ca_cert_path = sv_otel_exporter_otlp_traces_certificates;
    }
  }
  ... /* client_key / client_certificates / cipher / cipher_suite 同构 */

  options.ssl_min_tls =
      tls_option_string(sv_otel_exporter_otlp_traces_min_tls, OTLP_TLS_12);
  options.ssl_max_tls =
      tls_option_string(sv_otel_exporter_otlp_traces_max_tls, OTLP_TLS_DEFAULT);

  if (json) {
    options.content_type =
        opentelemetry::exporter::otlp::HttpRequestContentType::kJson;
    options.json_bytes_mapping =
        opentelemetry::exporter::otlp::JsonBytesMappingKind::kHexId;
  } else {
    options.content_type = /* kBinary */;
  }
  ...
}
```

**逐段解释**：

- **★ 前两个 `strlen()==0 → return nullptr`** 是理解"默认行为"的关键：**endpoint 默认为空 ⇒ 根本没有 exporter ⇒ 装上组件也什么都不发**。
- 每个 TLS 选项都是 **双重判空**（指针非空 + 长度非零）——component sysvar 未设置时可能是 nullptr。
- **★ `json_bytes_mapping = kHexId`**：JSON 模式下 trace/span id 用十六进制字符串（而非 base64），符合 OTel 的 JSON 编码约定。
- **只有 HTTP**：协议枚举只有 `http/protobuf` 与 `http/json`，**gRPC 在代码中完全不存在**（CMakeLists 也没有 `opentelemetry_exporter_otlp_grpc`）。官方测试环境里 4317 端口标着 "not used"。

#### 3.2 攒批参数生效点

```cpp
options.max_queue_size          = sv_otel_bsp_max_queue_size;          // 默认 2048
options.schedule_delay_millis   = std::chrono::milliseconds(sv_otel_bsp_schedule_delay);  // 5000
options.max_export_batch_size   = sv_otel_bsp_max_export_batch_size;   // 512
runtime_options.thread_instrumentation =
    new MySQLThreadInstrumentation(g_otel_bsp_thread_key, g_traces_network_namespace);
```

⇒ 队列满或超时（5 秒）触发导出；**导出线程是独立的 OTel 线程**，已注册进 PFS。

#### 3.3 ★ 遥测线程接入 PFS

```cpp
void MySQLThreadInstrumentation::OnStart() {
  PSI_thread *psi = thread_srv->new_thread(m_thread_key, 0, this, 0);
  thread_srv->set_thread_os_id(psi);
  thread_srv->set_thread(psi);
#ifdef HAVE_SETNS
  if (m_network_namespace_fd != NO_FD) set_network_namespace(m_network_namespace_fd);
#endif
  TLS_state = new MySQLThreadInstrumentationState();
}
void BeforeWait() { state->m_idle_psi = idle_srv->start_idle_wait(&state->m_idle_state, ...); }
void AfterWait()  { idle_srv->end_idle_wait(state->m_idle_psi); }
void OnEnd()      { thread_srv->delete_current_thread(); }
```

**三个作用**：① OTel 线程在 `performance_schema.threads` 中可见；② 导出线程进入指定 Linux netns（**遥测流量与业务流量隔离**）；③ 空闲等待计入 PFS idle 事件。

⇒ 六个线程：`otel_bsp` / `otel_pmr` / `otel_blrp` / `otlp_traces` / `otlp_metrics` / `otlp_logs`。**遥测自身的开销也是可观测的**。

### 四、Metrics：三档 Provider 与适配器

#### 4.1 为什么是三个频率

```cpp
frequencies.push_back(sv_metrics_reader_frequency_1);                 // 默认 10s
if (sv_metrics_reader_frequency_2 != 0) frequencies.push_back(...);   // 默认 60s
if (sv_metrics_reader_frequency_3 != 0) frequencies.push_back(...);   // 默认 0（禁用）
/* Accept frequency_1, frequency_2 and frequency_3 in any order, with duplicates. */
std::sort(...); std::unique(...);

for (ulong const f : frequencies) {
  options.export_interval_millis = std::chrono::milliseconds(f * 1000);
  /* Timeout = 80 % of export, capped to 5 minutes. */
  options.export_timeout_millis  = std::chrono::milliseconds(std::min(f * 800, 300000UL));
  provider = setup_otel_meter_provider(resource, options, runtime_options);
  g_all_meter_providers.add(f, std::move(provider));
}
```

**根本原因**：OTel 的 `PeriodicExportingMetricReader` **一个 reader 只有一个固定 interval**。想要不同指标不同节奏，只能建多个 Provider。

路由：

```cpp
for (auto &e : m_providers) { if (frequency <= e.m_frequency) return e.m_provider.get(); }
return m_providers.back().m_provider.get();   // providers 已按频率升序
```

#### 4.2 适配器：MySQL 回调 → OTel ObservableInstrument

```cpp
/*
  WARNING: As soon as the callback function is added to the otel instrument,
  otel can start collection and invoke the MySQLMetric::callback.
  For this reason, the callback not added in MySQLMetric constructor,
  but added explicitly after with MySQLMetric::addCallback().
*/
m_otel_instrument->AddCallback(MySQLMetric::callback, this);
```

**解释**：这是一个**真实的竞态**——`AddCallback` 之后 SDK 随时可能开始采集并回调进来。若放在构造函数里，对象尚未构造完成就会被使用。所以显式延后到构造完成之后。

类型翻译（counter/gauge 的判定**在服务端**，组件只翻译）：

```cpp
case METRIC_INTEGER:
  switch (metric_type) {
    case ASYNC_COUNTER:        meter->createInt64ObservableCounter(...);       break;
    case ASYNC_UPDOWN_COUNTER: meter->createInt64ObservableUpDownCounter(...); break;
    case ASYNC_GAUGE_COUNTER:  meter->createInt64ObservableGauge(...);         break;
```

**采集发生时**，SDK 回调 `MySQLMetric::observe()`，它再去调 MySQL 侧的回调拿当前值：

```cpp
void MySQLMetric::observe(opentelemetry::metrics::ObserverResult result) {
  mutex_srv->lock(&m_lock, __FILE__, __LINE__);

  if (m_collecting) {
    if (opentelemetry::nostd::holds_alternative<
            opentelemetry::nostd::shared_ptr<
                opentelemetry::metrics::ObserverResultT<int64_t>>>(result)) {
      auto access = opentelemetry::nostd::get<opentelemetry::nostd::shared_ptr<
          opentelemetry::metrics::ObserverResultT<int64_t>>>(result);
      opentelemetry::metrics::ObserverResultT<int64_t> *access_ptr = access.get();

      m_metric_cb(m_metric_cb_context, &my_delivery_callback, access_ptr);
    } else if (opentelemetry::nostd::holds_alternative<
                   opentelemetry::nostd::shared_ptr<
                       opentelemetry::metrics::ObserverResultT<double>>>(
                   result)) {
      /* ... double 分支，同构 ... */
      m_metric_cb(m_metric_cb_context, &my_delivery_callback, access_ptr);
    } else {
      assert(false);
    }
  }

  mutex_srv->unlock(&m_lock, __FILE__, __LINE__);
}
```

**逐段解释**：

- **`ObserverResult` 是 `variant`**（int64 / double 两种）⇒ 用 `holds_alternative` + `get` 分发，拿到裸指针后交给 MySQL 侧回调填充。这是 OTel C++ 典型的"类型擦除 + 访问者"手法。
- **`m_metric_cb` 是服务端注册的回调**，`my_delivery_callback` 是一组函数指针（int64/double × 0/1/N 个 attribute 共 6 个）。⇒ **值不是组件算的，是每次采集时向服务端"问"来的**（这就是 Observable/Pull 语义）。
- **`m_collecting` 标志 + mutex**：卸载/停采时置 false，保证 observe 不会再回调进已释放的服务端上下文。
- **`assert(false)` 兜底**：variant 只可能是这两种类型。

#### 4.3 meter 动态增删改

```cpp
telemetry_metrics_srv->meter_iterator_create(&it_meter);
do {
  meter_get_enabled(it_meter, meter_enabled);
  meter_get_name(it_meter, &meter_name);
  meter_get_frequency(it_meter, meter_frequency);
  if (meter_enabled) setup_otel_meter(raw_meter_name, meter_frequency);
} while (!meter_iterator_advance(it_meter));
```

组件**不硬编码任何 meter 名**，全部遍历服务端。`setup_meters` 表变更后有通知回调实时增删。

🔵【官方】13 个 meter：`mysql.stats` / `.com` / `.connection` / `perf_schema` / `.handler` / `.myisam` / `.ssl` / `inno` / `inno.buffer_pool` / `inno.data` / `x` / `x.stmt` / `mle`。**每条都有 Source Variable 列，直接对应 `SHOW STATUS` 的状态变量**。

⇒ ★ **metrics 本质是 `SHOW STATUS` 的 OTLP 化**。选它还是继续拉状态变量，是**采集架构（推 vs 拉）的选择，不是数据源的选择**。

### 五、Logs：与 trace 自动关联

```cpp
if (sv_log_enabled) {
  auto otel_severity = convert_severity(severity);   // ERROR/WARN/INFO/DEBUG
  auto record = logger->CreateLogRecord();
  record->SetTimestamp(std::chrono::system_clock::from_time_t(timestamp));
  record->SetSeverity(otel_severity);
  record->SetBody(opentelemetry::common::AttributeValue(message));
  record->SetAttribute(k_event_name, logger_name);
  for (...) switch (attr_array[i].type) {  /* 8 种类型 */ }
  set_log_record_trace(record.get());      // ★ 关键
  logger->EmitLogRecord(std::move(record));
}
```

```cpp
opentelemetry::trace::SpanContext const parent = data->m_stmt_stack.back()->GetContext();
record->SetTraceId(parent.trace_id());
record->SetSpanId(parent.span_id());
record->SetTraceFlags(parent.trace_flags());
```

⇒ **在 Loki 里看到一条 MySQL 错误日志，可以直接跳到它对应的那条 SQL 的 span**。这是三类信号真正打通的地方。

⚠️ **与错误日志 sink pipeline 完全独立**（源码中无 `log_sink_*`）：

```
错误日志 ──┬─> log_sink_json / log_sink_trad ─> 本地文件   （log_error_services 控制）
           └─> mysql_server_telemetry_logs ──> OTLP       （telemetry.log_enabled 控制）
```

### 六、组件生命周期：可挂起的 8 级状态机

```
BOOT → INSTALL → DETECT_RESOURCE → DECODE_SECRET → CONFIGURE → READY
                                                             ↘ FAILED（noop 降级）
```

**为什么这么复杂**：它依赖两个**可选**组件（resource provider / secret provider），而它们可能**晚于自己加载**。所以它自己实现了一个通知服务：

```cpp
BEGIN_SERVICE_IMPLEMENTATION(component_telemetry,
                             dynamic_loader_services_loaded_notification)
  services_loaded_notification
END_SERVICE_IMPLEMENTATION();
```

⇒ 任何一个新组件被加载时都会回调它，从中断处继续。

**★ noop 降级**（官方注释解释"鸡生蛋"）：

```cpp
/*
 * There is a chicken-and-eggs problem with components and system variables:
 * - to define a telemetry.xxx system variable, the telemetry component must be loaded,
 * - to load a component, the content of PERSIST_ONLY system variables must be correct,
 * - to modify a broken system variable in PERSIST_ONLY, the component must be loaded.
 * ... we still allow INSTALL COMPONENT telemetry to proceed, and set the component in noop mode.
 */
```

⇒ 密钥读不到时 **`INSTALL COMPONENT` 依然成功**，只是空转，好让你能改 PERSIST_ONLY 配置。

### 七、卸载：先打断自己，再等

```cpp
emit_control_span(false, false, false, k_reason_shutdown);
g_shutting_down.store(true);
unregister_telemetry_logger();
unregister_telemetry_callback();
abort_current_session();      // ← 主动打断正在被追踪的会话
wait_for_sessions();          // ← 等 g_session_count == 0
...
g_tracer_provider->ForceFlush(10ms); g_tracer_provider->Shutdown(10ms);
g_logger_provider->ForceFlush(10ms); g_logger_provider->Shutdown(10ms);
```

★ **为什么要 `abort_current_session()`**：`UNINSTALL COMPONENT` 这条语句**自身也在 telemetry 里**——不先打断，就会自己等自己。

`wait_for_sessions()` 的实现细节（源码注释本身就是一份设计文档）：

```cpp
  /*
    From now on, count disconnected sessions.
    Note that arming g_shutting_down is racy,
    so we may miss some signal on g_session_notify_cond.
    As a result, always use a timedwait.
    The source of truth is g_session_count,
    not the number of signal received (or not).
    Beside, the timedwait is used to check for THD::killed as well.
  */
  assert(g_shutting_down.load());

  if (progress != nullptr) {
    progress->m_work_completed = 0;
    progress->m_work_estimated = g_session_count.load();
  }

  while (g_session_count.load() > 0) {
    /*
      TODO: Check for THD::killed flag and abort UNINSTALL.
      Currently, UNINSTALL waits for every instrumented sessions,
      and is not responsive to KILL while waiting.
      ...
      Consider making: UNINSTALL killable ... INSTALL/UNINSTALL should be restartable.
    */
    wait_until = std::chrono::system_clock::now();
    wait_until += delay;                     // 500ms
    to_timespec(wait_until, deadline);
    log_info("%s: waiting for %d sessions, attempt %d", component_name,
             g_session_count.load(), attempts);
    mutex_srv->lock(&g_session_notify_mutex, __FILE__, __LINE__);
    cond_srv->timedwait(&g_session_notify_cond, ...);
    ...
  }
```

**三个要点**：
1. **用 `timedwait` 而非 `wait`**——因为 `g_shutting_down` 的置位是竞态的，可能丢信号；**真值来源是 `g_session_count` 不是信号数**
2. **等待进度通过 PSI stage 上报**（`m_work_estimated` / `m_work_completed`）⇒ 卸载卡住时能在 PFS 里看到进度
3. **源码 TODO 承认**：UNINSTALL 期间**不可被 KILL**，未来要改进

### 八、一句话总览：两代差别在哪

| | 8.0.39（社区版可得） | 9.7 |
|---|---|---|
| 埋点（①） | ✅ PFS 提供 service + 单槽 + 6 回调 | ✅ 同（签名一致） |
| 造 span（②） | 只有测试组件，写文件日志 | ✅ `component_telemetry` |
| 导出（③） | ❌ 无 | ✅ OTel SDK 1.23.0 |
| trace_id / W3C | ❌ 无；父子只有自数的 depth | ✅ SDK 生成 + Extract |
| metrics / logs | ❌ 无 | ✅ 两个新服务 |

⇒ **8.0.39 是"预留了插座"，9.7 是"把电器买回来接上"。**

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `telemetry.trace_enabled` | **OFF** | Global（可动态改） | trace 开关，**每条顶层语句重新采样** |
| `telemetry.metrics_enabled` | **OFF** | Global（**只读**） | metrics 总开关；改它要重装组件 |
| `telemetry.log_enabled` | **OFF** | Global（可动态改） | 错误日志外发 |
| `telemetry.query_text_enabled` | **ON** | Global（可动态改） | ⚠️ SQL 文本是否进 span |
| `telemetry.otel_exporter_otlp_*_endpoint` | **""（空）** | Global（只读） | ★ 空 ⇒ 没有 exporter |
| `telemetry.metrics_reader_frequency_1/2/3` | **10 / 60 / 0** | Global（只读） | 三档采集频率 |
| `telemetry.otel_bsp_{schedule_delay,max_queue_size,max_export_batch_size}` | 5000 / 2048 / 512 | Global（只读） | span 攒批 |
| `telemetry.otel_blrp_*` | 同上 | Global（只读） | log 攒批 |
| `telemetry.otel_log_level` | info | Global（可动态改） | OTel **SDK 自身**的调试日志级别 |
| `telemetry.run_level` | — | 状态变量 | **组件当前档位**（BOOT…READY/FAILED） |
| `telemetry.live_sessions` | — | 状态变量 | 当前 telemetry session 数 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 组件活没活、卡在哪一步 | SQL | `SELECT … WHERE variable_name='telemetry.run_level'` |
| 有多少会话被接管 | SQL | `telemetry.live_sessions` / `performance_schema.threads` 的 `TELEMETRY_ACTIVE` |
| 遥测自身开销 | PFS | `performance_schema.threads` 里 `otel_bsp` / `otel_pmr` / `otel_blrp` / `otlp_*` 六个线程 |
| 卸载卡住时的进度 | PFS | stage 事件（`m_work_estimated` / `m_work_completed`） |
| OTel SDK 内部在报什么 | SQL | 错误日志（`telemetry.otel_log_level` 调高） |
| 单个 metric 开关 | SQL | `UPDATE performance_schema.setup_meters SET ENABLED=…` |

---

## ★ 本机制里的工程实现技法

### 一、C++ 特性与所有权

| 特性 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| **opaque 句柄强转** | `telemetry_locker_t*` 实为 `Session_data*` | PFS 侧不需要知道组件内部结构，保持 ABI 边界 | 全靠 `reinterpret_cast`，类型安全完全靠约定 |
| **`string_view` + 外部持有** | `QueryAttributeTextMapCarrier::Get` | 避免拷贝 | 必须用 `m_seen` 托管生命周期，否则 view 悬空 |
| **tracer 指针快照** | `Session_data` 构造时保存 `g_tracer` | 防 UNINSTALL 时全局指针被置空的竞态 | 组件卸载后 session 仍持有旧 tracer（靠 abort 兜底） |
| **延后 `AddCallback`** | `MySQLMetric` | 避免对象未构造完就被 SDK 回调 | 构造函数与初始化分离，调用方必须记得调 `addCallback()` |

### 二、经典模式落地

| 模式 | 代码落点 | 说明 |
|---|---|---|
| **TextMapCarrier** | `tm_propagation.cc` | 把任意键值源适配给 OTel propagator；只实现 `Get`，`Set`/`Keys` 断言失败 ⇒ 单向 |
| **观察者（Observable Instrument）** | `tm_mysql_metric.cc` | 拉取式指标：SDK 采集时回调 MySQL 拿当前值，而非 MySQL 主动 push |
| **状态机 + 挂起恢复** | `tm_component.cc` 的 `g_run_level` | 依赖组件可后加载；靠 `dynamic_loader_services_loaded_notification` 续跑 |
| **降级为 noop** | `telemetry_init_noop_configure()` | 解决"装组件需要变量正确、改变量需要组件已装"的循环依赖 |

### 三、体系分工图

```
服务提供者（PFS / mysql-server）
   ├── mysql_server_telemetry_traces_v1   ← 语句/会话回调
   ├── mysql_server_telemetry_metrics_v1  ← meter/metric 迭代器
   ├── mysql_server_telemetry_logs        ← 错误日志回调
   └── pfs_notification_v3                ← 连接/断开
                │
                │  register_*
                ▼
component_telemetry（消费者）
   ├── tm_notification.cc   ← 6 个回调 + PFS 通知 + meter 生命周期 + log 回调
   ├── tm_propagation.cc    ← W3C 提取
   ├── tm_otel_*.cc         ← 三个 exporter
   └── tm_setup_otel.cc     ← 装配
                │
                ▼
   opentelemetry-cpp 1.23.0（内嵌于 extra/）
```

---

## Misc

### 扩展点

| 想扩展 | 改哪 |
|---|---|
| 新增一个 span attribute | `tm_notification.cc` 的 `set_span_stmt_data()` + `telemetry_stmt_data_t`（服务端） |
| 支持新的传播格式（如 B3） | `tm_propagation.cc` 换 Propagator 实现 |
| 新增 exporter 协议（如 gRPC） | `tm_setup_otel.cc` 的 `setup_otel_*_exporter()` + CMake 链接对应 target |
| 新增动态可改的变量 | 在 `tm_system_variables.cc` 去掉 `PLUGIN_VAR_READONLY` 并提供 update 函数 |

### 坑与已知缺陷

- **★ `compression` 有变量、无效果**：三个 `*_compression` 已注册，但 `tm_setup_otel.cc` 从未赋值给 `options.compression`
- **★ 采样全量且不可配**：`TracerProviderFactory::Create` 不传 sampler，无百分比/尾部采样，不支持 `OTEL_TRACES_SAMPLER`。生产全量开须评估，降采样只能在 Collector 做
- **★ UNINSTALL 期间不可被 KILL**：源码 TODO 承认（*"...is not responsive to KILL while waiting"*）
- **`service.instance.id` 用 `host:pid`**：源码 TODO 承认"重启后不稳定"，本应用 `host:port`
- **同一时刻只能有一个 telemetry 组件**（全局单槽，装第二个顶掉第一个）
- **装了组件不等于在发数据**：默认三开关 OFF + endpoint 空
- **`query_text_enabled` 默认 ON** ⇒ 开 trace 时 SQL 文本默认外发
- **只有 OTLP HTTP，没有 gRPC**
- **依赖不满足时静默跳过**：`OPENTELEMETRY_CPP_AVAILABLE=FALSE` → CMake `RETURN()`，某些精简构建里没有 `component_telemetry.so`

### 易混淆概念

| 此 X | 非彼 X |
|---|---|
| `threads.telemetry_active = YES` | **不代表**这条连接正在被 trace，只代表它已感知到组件存在 |
| `Telemetry_traces_supported = ON` | **不代表**有组件在跑，它只是"这个二进制编译时带不带 telemetry"的编译期常量 |
| telemetry 的 logs 通路 | **不是** `log_sink_*` 组件，是独立的 `mysql_server_telemetry_logs` 服务；两者并行且开关独立 |
| 9.7 的 metrics | **不是**新数据源，是 `SHOW STATUS` 变量的 OTLP 化 |
| 8.0.39 的 `trace` query attribute | 9.7 **已废弃**；qa 现在只用于传 traceparent |

---

## 参考

**官方文档**（已 fetch 核实）
- [MySQL 9.7 Reference Manual → 35 Telemetry](https://dev.mysql.com/doc/refman/9.7/en/telemetry.html) —— ★ 已无企业版声明
- [35.1 Installing OpenTelemetry Support](https://dev.mysql.com/doc/refman/9.7/en/telemetry-trace-install.html)
- [35.3 OpenTelemetry Trace](https://dev.mysql.com/doc/refman/9.7/en/telemetry-trace.html) / [35.3.2 Trace Format](https://dev.mysql.com/doc/refman/9.7/en/telemetry-trace-format.html)
- [35.4.3 Server Metrics](https://dev.mysql.com/doc/refman/9.7/en/telemetry-metrics-meter-metrics.html) —— 13 个 meter 与 Source Variable 对照
- [35.5 OpenTelemetry Logging](https://dev.mysql.com/doc/refman/9.7/en/telemetry-logging.html)
- [MySQL 8.4 → 35 Telemetry](https://dev.mysql.com/doc/refman/8.4/en/telemetry.html) —— ⚠️ 仍写 *"included in MySQL Enterprise Edition"*（版本差异证据）
- [MySQL 8.0 → 32.8 MySQL Telemetry](https://dev.mysql.com/doc/refman/8.0/en/mysql-enterprise-telemetry.html)

**标准与规范**
- W3C Trace Context（trace_id / traceparent 的编码与传播）—— 落点：`tm_propagation.cc`
- OpenTelemetry 语义约定（server span 用 `SpanKind::kServer`）—— 落点：`tm_stmt_notify_qa`

**本目录相关篇**
- [`pfs/pfs.md`](pfs/pfs.md)（PFS 框架；telemetry 寄生于它的 statement locker，两者**互补并行**）
- [`pfs/pfs_events.md`](pfs/pfs_events.md)（events_* 表族；与 telemetry **同源并行**的两条出路）
- [`status_variables.md`](status_variables.md)（★ metrics 的 Source Variable 都来自这里）
- [`logging/error_log.md`](logging/error_log.md)（错误日志 sink pipeline；telemetry logs 是**独立**通路）
- [`README.md`](README.md) 1.1（可扩展观测出口）/ 1.3（版本演进）
