# observability：观测体系（跨层主题）

> MySQL 的可观测性**横跨两层**，这是它归本目录的根本原因：
>
> | 侧 | 提供什么 |
> |---|---|
> | **server** | PFS 框架本体（`storage/perfschema/`，**152 个 .cc / 4.0 MB**）、I_S 框架、sys schema（`scripts/sys_schema/`）、慢查询日志、错误日志、SHOW 命令、EXPLAIN、状态变量 |
> | **InnoDB** | **132 个文件里都有 PFS 埋点**（`srv0srv.cc` 单文件就 12 处 `PSI_stage_info` 注册）、`INNODB_TRX` / `INNODB_METRICS` / `INNODB_BUFFER_PAGE`（`handler/i_s.cc` 实现）、`SHOW ENGINE INNODB STATUS`（`ha_innodb.cc`）、InnoDB monitor counters |
>
> ⇒ 按"**跨层框架 → 顶层 `infra/`**"的判据归此（与 [`../feat/i_s.md`](i_s.md) 同构：server 给框架、引擎填内容）。
> **不归 `feat/`**——`feat/` 放 8.0 引入的**具体特性**（SDI / FTS / DDL），而观测是贯穿所有特性的**基础设施**。

## ★ 组织维度：按「观测手段」，不是按「被观测的主题」

> **判据**：看一篇文档写的是"**某个手段本身**"（PFS 怎么工作、sys 有哪些视图）还是"**某个子系统怎么用手段**"（复制怎么观测、DDL 怎么观测）——**前者归本目录，后者留在主题篇**。

本目录放的是 **MySQL 有哪些观测手段、每个手段本身怎么实现 / 怎么用**。

**"XX 子系统怎么观测"一律留在对应主题篇**，不在此新建：

| 内容 | 归属 |
|---|---|
| DDL 观测 | [`../../feat/ddl/ddl.md`](../../feat/ddl/ddl.md) 15（已有：进度 / 阻塞 / 残留三个维度） |
| 复制观测 | [`../../server/replication/`](../../server/replication/) 各篇自己讲 |
| 锁等待 / 死锁的**诊断套路** | [`../lock/`](../lock/)；本目录只讲承载它们的**手段**（`data_locks`→I_S 篇、`metadata_locks`→PFS 篇、`SHOW ENGINE INNODB STATUS`→show_commands 篇） |
| 各子系统自己的 metrics | 对应主题篇 |

这就是"一份内容只在一处详写"：**手段篇讲手段，主题篇讲自己怎么用这个手段**（类比 `page_structure.md` 讲页布局、SDI 篇讲 SDI 专属部分）。

---

## 一、手段全景：七类

| 类型 | 手段 | 特点 |
|---|---|---|
| **① 表（结构化数据）** | `performance_schema` / `sys` / `information_schema`（含 `INNODB_*`） | 结构化、可 JOIN、可聚合、机器友好；★ **有开关代价**（instrument/consumer 默认只开一部分） |
| **② 日志** | 错误日志 / 慢查询日志 / 通用日志 / binlog（`mysqlbinlog`） | 事后、可按时间回溯；有 I/O 与体积代价；慢日志可**全量**记录 |
| **③ 命令与状态** | SHOW 族（`SHOW ENGINE INNODB STATUS` 等）/ `SHOW STATUS\|VARIABLES` / `mysqladmin` | 即时快照、人类可读、难编程消费；**看变化**要靠两次采样求差 |
| **④ 单语句分析** | `EXPLAIN` / `EXPLAIN ANALYZE` / optimizer trace | 单语句深度、能看到优化器决策；执行期开销大，**不宜长期开** |
| **⑤ 进程与栈** | `pstack` / `pt-pmp` / `gdb` / `perf` / 火焰图 / eBPF | 能看到 **MySQL 自己看不到之处**（内核、CPU、锁的持有者）；★ **部分手段会暂停进程**（见"代价"） |
| **⑥ 系统资源与平台** | `iostat` / `vmstat` / `strace` / `lsof` / `tcpdump` / PMM（Prometheus + Grafana） | 看资源瓶颈与长期趋势；PMM 适合常驻监控与告警 |
| **⑦ 可扩展出口**（★ 易漏） | PFS 插件表服务 / 日志 sink-filter 组件 / audit API / Telemetry service | **不是"现成手段"而是"手段的生长点"**：企业版与第三方工具正是靠它新增手段。见 1.1 |

> ⚠️ ①的表**不是固定 112 张**：静态内置 112 张 + **插件动态注册**的若干张（见 1.1）。

#### ② 日志：三种怎么选（定位差异很大，别开错）

| | 通用日志 `general_log` | 错误日志 `error_log` | 慢查询日志 `slow_log` |
|---|---|---|---|
| **记什么** | **所有**语句 | 启动 / 关闭 / 错误 / 警告 + 关键事件 | 超过阈值的语句 |
| **默认** | **关** | **开** | **关** |
| **主要用途** | 审计、排查"到底执行了什么" | 崩溃 / 异常 / 恢复回溯 | 性能分析（哪类语句慢） |
| **代价** | ⚠️ **最大**（每条语句都写，高并发下可拖垮实例） | 小（只在事件时写） | 中；⚠️ `long_query_time=0` 等同全量 |
| **输出** | 文件 / 表（`mysql.general_log`） | 文件（可配 JSON sink） | 文件 / 表（`mysql.slow_log`） |

⇒ **生产常开：错误日志 + 慢日志**；通用日志只在短时排障时开。
⚠️ 日志是**事后**手段——查"正在发生什么"不要开 `general_log`，用 `sys.session` 或 `SHOW PROCESSLIST`。


### ⑤ 进程与栈：三类工具的分工（★ 常被混淆）

| 工具 | 机制 | 输出 | 典型用途 |
|---|---|---|---|
| **`pstack`** | `gdb` 的封装，**attach 到进程**打印所有线程栈 | 一次全量栈快照 | 看"**现在卡在哪个函数**"（如 rename 停在 `fil_rename_tablespace` 而非 MDL 等待点） |
| **`pt-pmp`** | Percona Toolkit 的 Poor Man's Profiler：循环 `pstack` **多次采样后聚合** | 按栈出现次数排序的聚合结果 | 找"**热点等待**"。★ 排除法：**持有者的栈只出现 1 次，等锁者大量重复**——据此从 core/栈里定位锁持有者 |
| **`perf` / 火焰图 / eBPF** | 内核/硬件采样（perf）或内核态程序（eBPF） | CPU 采样、火焰图、自定义追踪 | 看 CPU 热点、off-CPU、内核态开销；**几乎不暂停进程** |

> 已有材料（本目录 `system_tools.md` 应展开并回指）：
> - [`../lock/primitives/innodb_sync.md`](../lock/primitives/innodb_sync.md)「core dump 排查锁持有者：gdb + pt-pmp 排除法」——**持有者堆栈只出现 1 次，等锁者大量重复**
> - [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md)「Rename table 死锁分析」——★ 教训：**先看 pstack 停在哪个函数**，停在 MDL 等待点才是 MDL 问题，停在 `fil_rename_tablespace` 是文件层问题

---
### 1.1 ★ 第七类：可扩展观测出口（这一层解释了"为什么静态枚举会漏"）

前六类都是"MySQL 现成给了什么"，而 8.0 还另有一层**扩展点**——企业版多出来的手段、第三方监控的采集，都是从这里长出来的：

| 扩展点 | 机制 | 长出来的手段 |
|---|---|---|
| **PFS 插件表服务**<br>`pfs_plugin_table_service.h` | 插件/组件可向 `performance_schema` 注册表 | ★ `clone_status` / `clone_progress`（`plugin/clone/src/clone_status.cc`）<br>★ Group Replication 的 `replication_group_member_actions`、`replication_group_configuration_version`、`communication_information` 等（`plugin/group_replication/src/perfschema/`）<br>NDB 的若干张（`storage/ndb/plugin/ndb_pfs_init.cc`） |
| **日志 sink / filter 组件**<br>`sql/server_component/` | 错误日志走组件化 pipeline（filter → sink），可插拔 | `performance_schema.error_log` 表就是 **perfschema sink**（`log_sink_pfs_event`）的产物；也可接 JSON sink |
| **audit API**<br>`sql/sql_audit.cc` | 插件订阅 server 事件 | 社区版唯一官方出口是 **`audit_api_message_emit` 组件**（让应用往审计流写消息）；⚠️ **审计日志（audit log）是企业版专有**，社区版源码树里只有测试用的 `audit_null` |
| **Telemetry service**<br>`mysql_server_telemetry_traces_service_imp.cc` | 向组件广播语句生命周期回调（OpenTelemetry 风格 span） | ★ **MySQL 内置了 OpenTelemetry**（导出 traces / metrics / logs），但**开放范围随版本变化**，见下方 1.3 |

⇒ **判据**：若某个"手段"在社区版源码里找不到实现但官方文档有名（audit log、telemetry），先查它是不是走这层扩展点 + 企业版专有。

### 1.2 ★ 手段的生命周期：已废弃 / 已移除 / 企业版专有

⚠️ 5.7 的资料会误导。这张表记录"曾经有、现在不该用"的手段：

| 手段 | 状态 | 替代 |
|---|---|---|
| `INNODB_LOCKS` / `INNODB_LOCK_WAITS`（I_S） | **8.0 已移除** | `performance_schema.data_locks` / `data_lock_waits` |
| `SHOW PROFILE` / `SHOW PROFILES` / `I_S.PROFILING` | **已废弃**（源码 `sql_profile.cc` 直接 `push_deprecated_warn`，官方指定替代 = **Performance Schema**） | `events_stages_*` / `events_statements_*` |
| `SHOW ENGINE LOGS` | 语法存在，但 **InnoDB 未实现**（`ha_innodb.cc` 里 `/* Not handled */`）；server 只用于 5.7→8.0 升级 | 无（不是运维手段） |
| `ha_alter_table()`（5.x 的 server↔引擎 ALTER 接缝） | 8.0 全仓 0 命中 | inplace 四件套 |
| **审计日志 audit log** | **企业版专有**，社区版无实现 | 通用日志（短时）/ audit API / `audit_api_message_emit` |
| **Telemetry** | **8.0 / 8.4 企业版专有**；★ **9.7 起下放到社区版**（详见 1.3） | 8.0.39 无 exporter；9.7 起为 `component_telemetry` |

⇒ **与其并列的两个"仍可用但常被忽略"的变体**：
- **`SHOW ENGINE MUTEX`** —— InnoDB **实现了**（`ha_innodb.cc` → `innodb_show_latch_status`），打印 latch/mutex 争用统计（自旋次数、OS wait），与 `SHOW ENGINE INNODB STATUS` 的 SEMAPHORES 段**互补**，是真手段
- **`SHOW ENGINE PERFORMANCE_SCHEMA STATUS`** —— PFS **自观测**（自身内存占用、丢事件计数）

### 1.3 ★ Telemetry（OpenTelemetry）：8.0 / 8.4 企业版 → **9.7 起下放到社区版**

MySQL 确实**内置了 OpenTelemetry**（导出 traces / metrics / logs），但开放范围随版本变化——这一点很容易被旧资料误导（三份官方页面逐一 fetch 对比得出）：

| 版本 | 官方手册位置 | 官方原文表述 |
|---|---|---|
| **8.0** | 32.8 MySQL Telemetry（父章节 = **MySQL Enterprise Edition**） | "The MySQL telemetry component is based on the OpenTelemetry (OTel) project…" |
| **8.4** | Chapter 35 Telemetry | ⚠️ 明写：*"OpenTelemetry support is a component included in **MySQL Enterprise Edition**, a commercial product."* |
| **9.7** | Chapter 35 Telemetry | ★ 该句**已删除**，改为*"OpenTelemetry support is included in MySQL as a component."*；安装即 `INSTALL COMPONENT 'file://component_telemetry'`，**全页无企业版 / 商业许可字样** |

**9.7 下放后的完整形态**：

```sql
INSTALL COMPONENT 'file://component_telemetry';
-- 确认：SELECT * FROM mysql.component;  看到 file://component_telemetry
```

装完新增一批 `telemetry.*` 系统变量（三类信号各自的 OTLP 出口）：

| 变量族 | 说明 |
|---|---|
| `telemetry.trace_enabled` / `metrics_enabled` / `log_enabled` | 三类信号开关（默认 ON） |
| `telemetry.otel_exporter_otlp_{traces,metrics,logs}_endpoint` | 默认 `http://localhost:4318/v1/{traces,metrics,logs}` |
| 同前缀的 `_protocol`（默认 `http/protobuf`）/ `_compression` / `_timeout` / `_{client_,}certificates` / `_max_tls` / `_min_tls` | OTLP 导出参数 |
| `telemetry.otel_bsp_*` / `_blrp_*` | BatchSpanProcessor / BatchLogRecordProcessor 的队列与批大小 |
| `telemetry.query_text_enabled`、`telemetry.otel_log_level`、`telemetry.otel_resource_attributes` | 是否带 SQL 文本、组件自身日志级别、资源属性 |

客户端侧另有 `--telemetry_client`（或配置 `[mysql] telemetry-client=ON`）。

⇒ **判据意义**：这是官方把可观测性往**标准协议 OTLP** 收敛的信号——下放后社区版可直接把 traces / metrics / logs 推给 OTel Collector，不必再依赖 PFS 表轮询或 PMM 的私有采集器。

> ⚠️ **版本基线声明**：本知识库源码基线是 **8.0.39**，该版本**没有** `component_telemetry`——源码树里只有 **service 实现**（`storage/perfschema/mysql_server_telemetry_traces_service_imp.cc`）和**测试组件**（`components/test_server_telemetry_traces/`），社区版可用的状态变量只有 `Telemetry_traces_supported`、`threads.telemetry_active` 列。
> 上面 9.7 的细节**来自官方文档，无法用 8.0.39 源码验证**；若日后升级基线需重新核对。

---

## 二、★ 四个系统库的边界与选型

观测最常犯的错是"不知道该查哪个库"。四者定位完全不同：

| 库 | 本质 | 数据来源 | 持久性 | 你要查它当且仅当 |
|---|---|---|---|---|
| **`performance_schema`** | 专用引擎的**内存表**（表定义里写死 `" ENGINE=PERFORMANCE_SCHEMA"`） | server + 引擎的 **instrument 埋点** | **重启即清空**，不落盘 | 要**细粒度历史事件**（一次等待、一个阶段、一条语句的耗时） |
| **`sys`** | 纯 SQL 封装（视图 + 函数 + 存储过程，源码在 `scripts/sys_schema/`） | 基于 **PFS + I_S** 加工 | 不存数据（每次查询现算） | 要**人读的结果**（已按延迟/次数排好序的聚合） |
| **`information_schema`** | 元数据 + 部分运行时状态 | server 元数据 + **引擎提供的 `INNODB_*` 表** | 不存（`INNODB_*` 读引擎内存/文件） | 要**表/列/索引/表空间元数据**，或 InnoDB 内部状态（`INNODB_TRX` 等） |
| **`mysql`** | 8.0 起是 **InnoDB 表**（`mysql.ibd`） | 数据字典 + SE 私有数据（`se_private_data`） | **持久化** | 要查 DD 表本身、`innodb_table_stats`，或排查 `innodb_ddl_log`（⚠️ **隐藏表，不可 `SELECT`） |

### 选型三问

```
1. 要「机器消费 / 稳定 schema / 细粒度」→ performance_schema
2. 要「人读 / 已聚合 / 少写 SQL」      → sys
3. 要「元数据」或「INNODB_* 运行时」    → information_schema
4. 要「持久化统计」或「DD 表本身」      → mysql
```

**同一问题的两种查法**（典型对照）：

| 需求 | sys（人读，推荐） | PFS / I_S（机器消费） |
|---|---|---|
| 谁在等锁 | `sys.innodb_lock_waits` | `performance_schema.data_lock_waits` |
| 最慢的语句 | `sys.statements_with_runtimes_in_95th_percentile` | `events_statements_summary_by_digest` |
| 当前会话在干嘛 | `sys.session` / `sys.processlist` | `performance_schema.threads` + `events_statements_current` |
| 等什么最久 | `sys.wait_classes_global_by_latency` | `events_waits_summary_global_by_event_name` |
| 表/索引 IO 热点 | `sys.schema_table_statistics` | `table_io_waits_summary_by_table` |

---

## 三、★ 选型决策树：按故障现象定位

> ★ **入口是"现象"不是"表"**——运维遇到的是"实例 hang 了""内存涨了""锁超时了"，不是"我要查某张表"。
> 下表按**故障现象**分派手段；每个分支末端的篇目给出具体 SQL。
> ⚠️ **①②⑪ 三类要优先看错误日志**——MySQL 会把自己检测到的内部异常（semaphore wait、崩溃、恢复）主动写进去，这是唯一能回溯的手段。

```
┌─ ① hang / 无响应（实例假死、语句全卡住）
│    ├─ SQL 层还连得上 → sys.session 看 State 分布 / sys.wait_classes_global_by_latency
│    ├─ SQL 层看不出 → ★ pstack / pt-pmp / perf（看 MySQL 自己看不到的地方）
│    └─ ★ 错误日志出现 semaphore wait → InnoDB 内部已卡（长时间等待会自行打断言并可能 crash）
│
├─ ② 内存激增 / OOM
│    ├─ 按事件名 → performance_schema.memory_summary_global_by_event_name
│    │              / sys.memory_global_by_current_bytes
│    ├─ 按线程 → sys.memory_by_thread_by_current_bytes
│    ├─ PFS 自身占多少 → SHOW ENGINE PERFORMANCE_SCHEMA STATUS
│    └─ ★ 注意：OOM 前往往伴随 buffer pool 之外的分配（详见 3.1）
│
├─ ③ 锁等待 / 锁超时 / 卡住
│    ├─ 行锁 → sys.innodb_lock_waits / PFS data_lock_waits
│    ├─ 元数据锁 → PFS metadata_locks（LOCK_STATUS='PENDING'）
│    └─ 死锁 → SHOW ENGINE INNODB STATUS 的 LATEST DETECTED DEADLOCK
│
├─ ④ CPU 飙高
│    ├─ 哪个语句 → sys.statements_with_runtimes_in_95th_percentile
│    └─ MySQL 内部热点 → perf / 火焰图（★ 不暂停进程，可长跑）
│
├─ ⑤ IO 打满 / IO 慢
│    ├─ 按线程 → sys.io_by_thread_by_latency
│    ├─ 按文件 → sys.io_global_by_file_by_bytes
│    └─ 系统层 → iostat / eBPF
│
├─ ⑥ 连接数打满（Too many connections）
│    ├─ 当前连接 → sys.session / performance_schema.threads
│    └─ ★ 已经连不进去 → admin_address / admin_port 管理接口（独立端口，不受 max_connections 限制）
│
├─ ⑦ 复制延迟 / 主从不一致
│    → 现象归 [`../../server/replication/`](../../server/replication/)；
│      手段层：SHOW REPLICA STATUS、PFS replication_* 表
│
├─ ⑧ 慢查询突增 / 执行计划劣化
│    ├─ 哪类慢 → 慢日志 / sys.statements_with_runtimes_in_95th_percentile
│    └─ 单条为什么慢 → EXPLAIN → EXPLAIN ANALYZE → optimizer trace   [explain_trace.md]
│
├─ ⑨ 磁盘空间暴涨
│    ├─ binlog → SHOW BINARY LOGS
│    ├─ undo / 表空间 → INNODB_TABLESPACES / INNODB_SESSION_TEMP_TABLESPACES
│    ├─ 临时文件 → 错误日志 + lsof（★ online DDL 的 sort file 会落盘，量可很大）
│    └─ 表膨胀 → INNODB_TABLESTATS
│
├─ ⑩ 长事务 / undo 不回收 / purge 跟不上
│    ├─ 谁跑最久 → INNODB_TRX（trx_started / trx_rows_modified）
│    └─ history list length → SHOW ENGINE INNODB STATUS 的 TRANSACTIONS 段
│
├─ ⑪ 崩溃 / 重启 / 要回溯
│    ├─ 恢复过程 → 错误日志（★ 唯一完整记录，含 crash recovery 与 DDL log replay）
│    └─ 崩溃现场 → core dump + gdb / pt-pmp 排除法
│
├─ ⑫ 表缓存 / 打开表数超限
│    → SHOW STATUS LIKE 'Open%' / Opened_tables；PFS table_io_waits_summary_by_table
│
├─ ⑬ 临时表落盘暴增
│    → SHOW STATUS LIKE 'Created_tmp%'；慢日志（log_slow_extra）的 Created_tmp_disk_tables
│
├─ ★ 某个**对象**（表 / 事务 / 索引 / 表空间）现在什么状态？
│    → I_S 的 `INNODB_*` 系列（详见 3.3）
│
└─ 长期趋势 / 告警
     → 状态变量两次采样求差 / PMM                              [status_variables.md]
```

**按"查询意图"的另一视角**（同一批手段，换入口）：

| 意图 | 手段 |
|---|---|
| 现在正在发生什么（谁在跑、什么 State） | `sys.session` / `sys.processlist` / `SHOW PROCESSLIST` |
| 到底在等什么（等待事件聚合） | `sys.wait_classes_global_by_latency` / PFS `events_waits_*` |
| 哪类操作最耗时（聚合视角） | `sys.statements_with_runtimes_in_95th_percentile` / 慢日志 |
| MySQL 内部在干什么（SQL 层看不见） | 瞬时 `pstack`；采样 `pt-pmp` / `perf` / 火焰图；长期 eBPF |

### 3.1 ★ 内存观测：PFS 的 memory 类（常被忽略的第四类 instrument）

"内存激增"的正解不是 `SHOW STATUS`，而是 **PFS 的 memory instrument**——它能把内存按**事件名（谁分配的）**拆开，这是状态变量做不到的。

**5 张汇总表**（`performance_schema`）：

| 表 | 维度 |
|---|---|
| `memory_summary_global_by_event_name` | 全局 × 事件名 |
| `memory_summary_by_thread_by_event_name` | 线程 × 事件名 |
| `memory_summary_by_account_by_event_name` | 账号 × 事件名 |
| `memory_summary_by_user_by_event_name` | 用户 × 事件名 |
| `memory_summary_by_host_by_event_name` | 主机 × 事件名 |

**sys 侧**（`x$` 版为原始字节数，非 `x$` 版带单位格式化）：

```sql
-- 全局：谁占了多少内存
SELECT event_name, current_alloc
  FROM sys.memory_global_by_current_bytes
 LIMIT 10;

-- 按线程：定位到具体连接
SELECT th.thread_id, th.processlist_user, th.processlist_command,
       m.current_alloc
  FROM sys.memory_by_thread_by_current_bytes m
  JOIN performance_schema.threads th USING (thread_id)
 ORDER BY m.current_alloc DESC LIMIT 10;
```

**★ 默认开关（易踩坑）**：

- `register_memory_class()` 走 `init_instr_class()`，其中 **`m_enabled = true`** ——对比 mutex / rwlock / cond 三类显式写了 `entry->m_enabled = false; /* disabled by default */`
  ⇒ **memory instrument 默认开启**（前提是 `performance_schema=ON`）
- ⚠️ 但**可见性还受 `performance_schema_max_memory_class` 上限约束**（超过上限后新注册的内存类不再计入）
- ★ **PFS 自身**的内存占用另有一处出口：`SHOW ENGINE PERFORMANCE_SCHEMA STATUS`（`performance_schema` 是 MySQL 里少数会"自己观测自己"的子系统）

> ⚠️ 与②分支的呼应：**OOM 时往往是 buffer pool 之外的分配暴涨**（连接缓冲、临时表、Prepared statement、PFS 自身）。`memory_summary_*` 是唯一能按事件名定位到"谁分配"的手段。

### 3.2 PFS 里不属于 `events_*` 的杂项表（易漏的观测点）

这类表没有统一的事件模型，容易被"按事件表族"的思路漏掉，但**其中几张是特定故障的唯一解**：

| 表 | 观测什么 | 什么时候需要 | 常用度 |
|---|---|---|---|
| **`log_status`** | redo / binlog / GTID 的**一致性位点** | 备份工具取一致性坐标（`LOCK INSTANCE FOR BACKUP` 配套） | ★★★ |
| **`prepared_statements_instances`** | 每个 prepare 语句实例、执行次数 | 排查 **prep stmt 泄漏**（`max_prepared_stmt_count` 打满、应用忘了 close） | ★★★ |
| **`events_statements_summary_by_program`** | 按**存储程序**（procedure/function/trigger/event）聚合 | 存储程序密集型业务定位热点 routine | ★★★ |
| **`session_connect_attrs`** | 客户端连接属性（`_client_name` / `_pid` / `_os`…） | 溯源"这个连接是哪个应用/驱动/进程发来的" | ★★★ |
| `objects_summary_global_by_type` | 按对象类型的操作汇总 | 各类对象的 DDL / 操作频次 | ★★☆ |
| `socket_summary_by_event_name` / `by_instance` | 网络层 socket 读写字节/次数 | 网络侧瓶颈定位 | ★★☆ |
| `user_variables_by_thread` | 会话 `@var` 的值 | ★ **可跨会话查看别人的用户变量**（排查"某个会话的变量被谁设了"） | ★☆☆ |
| `binary_log_transaction_compression_stats` | binlog 事务压缩率 / 节省字节 | 评估压缩效果 | ★☆☆ |
| `tls_channel_status` | 各连接通道的 TLS 版本 / 密码套件 / 会话复用 | 合规审计 | ★☆☆ |
| `keyring_component_status` | keyring 组件状态 | 组件运维 | ★☆☆ |

### 3.3 ★ 点查对象状态：I_S 的 `INNODB_*` 系列

> ★ **已有专篇**：[`i_s.md`](i_s.md)（925 行）已完整覆盖 I_S 的**机制与 26 张表全貌**（表族分类、数据来源、`ST_FIELD_INFO` 声明 → 注册 → `fill_table()` 填充、权限、版本机制）。
> **本节不做重复**，只给**观测（运维）视角**的两件事：① 决策树这一支的入口 ② `i_s.md` 未覆盖的**补遗**（第二个来源、instant DDL 查询、8.0 已移除的表）。

决策树里"某个对象现在什么状态"这一支，查的就是 `INNODB_*`。它**不是一张表而是一族表**，而且**有两个来源**（这点常被忽略）：

| 来源 | 数量 | 位置 | 提供什么 |
|---|---|---|---|
| **引擎插件**（InnoDB） | **26** | `storage/innobase/handler/i_s.cc` | 引擎**运行时**状态（事务、缓冲池、压缩、全文、计数器…） |
| **server 层 DD system view** | **5** | `sql/dd/impl/system_views/` | DD 里**持久化**的元数据：`INNODB_DATAFILES`、`INNODB_FIELDS`、`INNODB_FOREIGN`、`INNODB_FOREIGN_COLS`、`INNODB_TABLESPACES_BRIEF` |

按用途分组（引擎侧 26 张）：

| 用途 | 表 |
|---|---|
| **事务** | `INNODB_TRX` |
| **表 / 列 / 索引的内部属性** | `INNODB_TABLES`、`INNODB_COLUMNS`、`INNODB_INDEXES`、`INNODB_VIRTUAL` |
| **表空间 / 临时表空间** | `INNODB_TABLESPACES`、`INNODB_SESSION_TEMP_TABLESPACES`、`INNODB_TEMP_TABLE_INFO` |
| **缓冲池** | `INNODB_BUFFER_PAGE`、`INNODB_BUFFER_PAGE_LRU`、`INNODB_BUFFER_POOL_STATS`、`INNODB_CACHED_INDEXES` |
| **压缩** | `INNODB_CMP`、`INNODB_CMP_RESET`、`INNODB_CMP_PER_INDEX`、`INNODB_CMP_PER_INDEX_RESET`、`INNODB_CMPMEM`、`INNODB_CMPMEM_RESET` |
| **全文索引** | `INNODB_FT_CONFIG`、`INNODB_FT_INDEX_TABLE`、`INNODB_FT_INDEX_CACHE`、`INNODB_FT_DELETED`、`INNODB_FT_BEING_DELETED`、`INNODB_FT_DEFAULT_STOPWORD` |
| **计数器 / 统计** | `INNODB_METRICS`、`INNODB_TABLESTATS` |

**典型查询**（这一类问题的答案都在这族表里）：

```sql
-- ① 长事务（谁跑了多久、锁了多少行、在跑什么）
SELECT trx_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS running_sec,
       trx_state, trx_rows_locked, trx_rows_modified, trx_query
  FROM information_schema.INNODB_TRX
 ORDER BY trx_started LIMIT 10;

-- ② ★ 哪些表做过 instant DDL？（INSTANT_COLS = 第一次 instant ADD COLUMN 前的列数）
SELECT NAME, INSTANT_COLS, TOTAL_ROW_VERSIONS
  FROM information_schema.INNODB_TABLES
 WHERE INSTANT_COLS > 0;
-- 源码：is_upgraded_instant() ? get_instant_cols() : 0
-- ⇒ INSTANT_COLS = 0 表示没做过；> 0 表示做过
--   TOTAL_ROW_VERSIONS = 当前行版本号

-- ③ instant 加的列的默认值存在哪
SELECT NAME, HAS_DEFAULT, DEFAULT_VALUE
  FROM information_schema.INNODB_COLUMNS
 WHERE HAS_DEFAULT = 1;

-- ④ 表 / 索引的 InnoDB 内部 id 与 root page
SELECT t.NAME, t.TABLE_ID, t.SPACE, t.ROW_FORMAT, i.NAME AS idx, i.INDEX_ID, i.PAGE_NO
  FROM information_schema.INNODB_TABLES t
  JOIN information_schema.INNODB_INDEXES i ON i.TABLE_ID = t.TABLE_ID
 WHERE t.NAME LIKE 'db/%';
```

> ⚠️ 注意 `INNODB_LOCKS` / `INNODB_LOCK_WAITS` 在 **8.0 已移除**，锁信息改由 `performance_schema.data_locks` / `data_lock_waits` 提供（这也是"四系统库边界"里最容易踩的坑之一：5.7 的资料还在查 `INNODB_LOCKS`）。

---

## 四、★ 代价与注意事项（"看不到"和"不敢开"的根本原因）

### 为什么"看不到"——先查开关，不是先查有没有这个表

| 现象 | 真正原因 |
|---|---|
| 看不到 DDL 进度 | ★ **consumer 默认 OFF**（`performance_schema_consumer_events_stages_current` 默认 `false` 且 `READ_ONLY`，只能启动时指定），而 `stage/innodb/alter table%` 的 instrument **默认是开**的（带 `PSI_FLAG_STAGE_PROGRESS` ⇒ `m_enabled = true`） |
| PFS 里查不到某语句，但 `SHOW PROCESSLIST` 有 | 该 stage instrument 的 flags 为 0 ⇒ 默认 `ENABLED=NO`；但 `pfs_start_stage_v1()` 里 `/* Always update column threads.processlist_state. */` ⇒ **State 的更新不依赖 instrument 开关** |
| 表存在但查不到行 | 对应的 instrument / consumer 未启用 |

⇒ **排查"看不到"类问题的第一顺位是 `setup_instruments` / `setup_consumers` 的开关状态**，不是表结构。（DDL 篇已验证：真凶是 consumer 默认 OFF）

### 各手段的代价

| 手段 | 代价 | 注意 |
|---|---|---|
| PFS | 内存（每个 instrument 有 buffer）+ 每次事件的 CPU 采样开销 | 默认只开一部分，全开会显著增加开销 |
| 慢查询日志 | I/O + 高并发下的可观测开销 | ⚠️ `long_query_time=0` 会记录所有语句，可能拖垮实例 |
| `SHOW ENGINE INNODB STATUS` | 需持有 InnoDB 内部锁 | 高频轮询（秒级）有代价，不宜做监控数据源 |
| **`pstack` / `gdb`** | ★ **attach 会暂停目标进程** | 生产环境慎用；单次影响通常很短，但不可高频 |
| **`pt-pmp`** | 循环 `pstack` 多次采样（默认 30 次量级）⇒ **多次暂停累加** | 需评估累计 STW 时间；适合短时诊断而非常驻 |
| `perf` | 采样开销低（约 1%~5%） | 可较长时间运行 |
| eBPF | 最低（内核态聚合） | 需内核支持与权限 |
| optimizer trace | 执行期开销大 | 只针对单条语句临时开启 |

---

## 五、目录树与状态

| 篇 | 内容 | 行数 | 状态 |
|---|---|---|---|
| `README.md` | 本篇：定位 + 手段全景 + 四系统库边界 + 决策树 | 270 | ✅ |
| [`i_s.md`](i_s.md) | **INFORMATION_SCHEMA**：server 插件框架 + InnoDB 26 张表 + 权限 + 三分天下 + **「六、运维视角」** + ★ **1.6 `fill_table()` 完整实现**（每次查询现场填充 / ★ 需 `PROCESS_ACL` 否则静默 0 行 / ★ 先拷 cache 再格式化以缩短持锁时间 / ★ 截断打 `ib::warn` / ★ `Item *` 未使用 ⇒ WHERE 不下推） | 1039 | ✅ |
| [`pfs/`](pfs/) | ★ **Performance Schema（多篇 ⇒ 独立子目录）**：框架不是一张表而是一整套埋点体系，单篇装不下 | — | ✅ 已建 |
| ├ [`pfs/pfs.md`](pfs/pfs.md) | PFS **框架**：PSI 接口、零开销探针、instrument / consumer、设计模式；★ **1.10 PFS 表不是固定的**（静态内置 + 插件动态注册） | 1176 | ✅ |
| ├ [`pfs/pfs_statement.md`](pfs/pfs_statement.md) | statement 事件深剖：events_statements 表族、直方图、采样 | 1077 | ✅ |
| ├ [`pfs/thread_pfs.md`](pfs/thread_pfs.md) | PSI 线程插桩：注册表、包装器、所有权纪律 | 537 | ✅ |
| └ [`pfs/pfs_events.md`](pfs/pfs_events.md) | **事件表族**：waits / stages / transactions / **errors** + ★ **memory**（字段三组语义 / 默认开）+ ★ 非 `events_*` 杂项表 10 张 + 两级开关 + 聚合模型 + 环形缓冲 | 1268 | ✅ 已完成（★ 补上全库 **0 命中** 的 `events_errors` 与 `memory_summary`） |
| [`logging/`](logging/) | ★ **日志类（多篇 ⇒ 独立子目录）**：三种日志定位/开关/代价各不相同 | 393 | ✅ 已建 |
| ├ [`logging/error_log.md`](logging/error_log.md) | **错误日志**：组件架构（filter/sink）、★ `log_sink_json` 字段全集、`log_error_verbosity`(默认 2)、dragnet、`ib::` 宏族、★ `innodb_print_ddl_logs` 陷阱、`error_log` 表 + 5MB 环形缓冲、★ 暴涨根因（semaphore wait 1Hz） | 538 | ✅ |
| ├ [`logging/general_log.md`](logging/general_log.md) | 通用查询日志：★ 写入链路（`general_log_print` / `general_log_write` 的 **audit 不对称**）、★ 表输出临时关 binlog、文件 handler 的 `Silence_log_table_errors`、CSV 表无索引 | 330 | ✅ |
| └ [`logging/slow_log.md`](logging/slow_log.md) | **慢查询日志**：判定链、★ `log_slow_extra` **20 字段全集**、★ `log_slow_admin_statements` 真实机制、`Query_time` 含 `Lock_time`、表输出差异、代价、与 digest 对比 | 428 | ✅ |
| [`innodb_monitor.md`](innodb_monitor.md) | InnoDB monitor：`SHOW ENGINE INNODB STATUS`、★ monitor counter 的**两类形态**（真计数器 vs `MONITOR_EXISTING` 取值器大 switch）、`monitor_set_tbl` 位图开关、与 `INNODB_METRICS` 的分工 | 542 | ✅ |
| [`explain_trace.md`](explain_trace.md) | EXPLAIN（TRADITIONAL/TREE/JSON/ANALYZE）+ optimizer trace | 442 | ✅ |
| [`sys_schema.md`](sys_schema.md) | **sys schema**：102 视图 / 22 函数 / 26 过程；★ 9 个核心视图**完整 SQL 剖析**（`processlist` 五表 JOIN 的 NULL 语义开关 / `innodb_lock_waits` 四表十字关联 / ★ 95 百分位的两层纯 SQL 技巧）；★ 6 个函数完整定义（`format_bytes` 的 **C++ 1024 进制阈值段**、`format_pico_time`、`format_statement` 的"留头尾砍中间"、`format_path` 七目录脱敏、`sys_get_config` 三段式配置）；★ `sys_config` 配置机制 + 打包脚本 + 安装顺序约束；★ 6.5 **consumer 父子依赖**（`ps_is_consumer_enabled` 完整 CASE）；★ 6.8 纯 SQL ASCII 直方图；★ 6.10 `diagnostics()` 两次快照求差 | 1800 | ✅ |
| [`telemetry.md`](telemetry.md) | ★★ **MySQL Telemetry / OpenTelemetry**：★ 架构（**组件是消费者，PFS 才是服务提供者**）；★ 服务 3 方法 + 回调 6 个 + `flags` 双向（强制追踪）；★ 触发（query attribute `trace`，**名字是组件 sysvar 默认值**）；★ **以 9.7 源码为单一主线**（8.0/8.4 差异收在「版本演进」与第八节）。核心实现 8 个功能域，完整剖析：`tm_stmt_start`（★顶层不建 span）/ **`tm_stmt_notify_qa`**（★建 span 的真正位置，Span Link）/ `tm_stmt_end` / `Session_data` 的一生 / `tm_set_parent_from_qa` + `QueryAttributeTextMapCarrier` / `setup_otel_otlp_http_exporter` / `MySQLMetric::observe` / `wait_for_sessions`。理论基础含 5 条设计权衡（为何三层、为何 qa、为何 Link、为何不采样、为何 metrics_enabled 只读） | 1047 | ✅ 已完成 |
| ~~`information_schema.md`~~ | ❌ **已取消**——I_S 机制与 26 张表全貌已由 [`i_s.md`](i_s.md) 完整覆盖，不重复建篇；运维用法补在 `i_s.md` 内新增章节 | — | — |
| [`processlist.md`](processlist.md) | ★ 会话观测：四种入口的差异与代价；★ legacy SHOW 的锁代价；`THREAD_OS_ID` 联动 OS 层；**后台线程只有 threads 能看到**；State 体系（SQL 约 100 项 + InnoDB 12 项） | 937 | ✅ 已完成 |
| [`show_commands.md`](show_commands.md) | **SHOW 命令族**：★ 多数被改写成 SELECT（PFS 表 / DD system view）；`SHOW ENGINE INNODB STATUS` 全段判读 + **`LATEST DETECTED DEADLOCK` 逐字段** + 受害者选择算法 + 截断与锁代价 | 847 | ✅ 已完成 |
| [`status_variables.md`](status_variables.md) | **状态变量 / 系统变量**：三作用域（默认 SESSION）、汇总时机、PFS 六表、代价；★ **counter vs gauge 分类与采样套路**；`SET PERSIST` / `variables_info` / `variables_by_thread` | 689 | ✅ 已完成 |
| [`system_tools.md`](system_tools.md) | **系统层 / 外部工具**：★ `THREAD_OS_ID` 桥梁；`pstack`/`pt-pmp`/`gdb`（★ **会暂停进程**）vs `perf`/火焰图/eBPF（不暂停）；★ 代价对照表；决策树；★ **各工具实战命令与输出**（`pstack`=gdb 批处理 / ★ `pt-pmp` 采样聚合的"1 次 vs 大量重复"排除法 / `perf -F 99 -g` / eBPF uprobe / ★ 经 `THREAD_OS_ID` 的端到端下钻链路） | 549 | ✅ 已完成 |

**留原地不迁**：`server/infra/diagnostics.md`（777 行）——讲的是**诊断区 / 条件体系 / SIGNAL**，是**错误处理机制**不是运维观测，本目录只在相关处指向它。

---


## 参考

- *MySQL 8.0 Reference Manual → Performance Schema* / *sys Schema* / *INFORMATION_SCHEMA Tables* / *Optimizing with EXPLAIN*
- 源码：`storage/perfschema/`（PFS 框架，表定义里 `" ENGINE=PERFORMANCE_SCHEMA"`）、`scripts/sys_schema/`（sys 的视图/函数/存储过程）、`storage/innobase/handler/i_s.cc`（`INNODB_*` 表）、`sql/sql_show.cc`（SHOW 命令）
- [`../../feat/i_s.md`](i_s.md)（I_S 机制与 26 张表全貌；本目录不另建 I_S 篇，运维用法补在它的「六、运维视角」章）
- [`../lock/`](../lock/)（锁机制与诊断套路；含 `pt-pmp` 排除法与 pstack 定位 rename 死锁的案例）
- [`../../feat/ddl/ddl.md`](../../feat/ddl/ddl.md) 15（DDL 观测：进度 / 阻塞 / 残留——本目录决策树的一个具体应用）
