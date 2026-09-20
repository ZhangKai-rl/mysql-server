# InnoDB Monitor 与 monitor counter 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 InnoDB Monitor 输出体系（`SHOW ENGINE INNODB STATUS` 全景文本、三个 monitor 类后台线程、临时文件中转机制）、monitor counter 体系（`information_schema.INNODB_METRICS` 的背后：ID 即下标的三位一体结构、位图开关、无锁累加、`MONITOR_EXISTING` 复用），以及与 server 层 status variable / PFS 的分工。
>
> **边界**：本篇讲 **InnoDB 的可观测基础设施**（它是全局的，不专属任何子系统——锁文档引用它，不是它引用锁）。server 层 status variable 体系（`System_status_var`、`SHOW STATUS` → PFS 重写）见 [`../server/infra/variables.md`](../server/infra/variables.md) G 节；PFS instrument 体系见 [`../server/infra/pfs.md`](../server/infra/pfs.md)；DBUG 见 [`../server/infra/dbug.md`](../server/infra/dbug.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- 核心实现
  - [主链路](#主链路)
  - [Monitor 线程体系](#monitor-线程体系)
  - [monitor counter 体系](#monitor-counter-体系)
  - [与 server 层体系的关系](#与-server-层体系的关系)
  - [与 PFS 的能力重叠与分工](#与-pfs-的能力重叠与分工两套测量模型)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 语境里的 **"monitor" 是 InnoDB 的专有叫法**，有两件东西：

1. **InnoDB Monitor**：一整块**自由文本**的全景诊断输出（`SHOW ENGINE INNODB STATUS` / error log / `innodb_status.<pid>` 文件），由后台线程周期性生成或按需现场生成。
2. **monitor counter**：InnoDB 内部的**计数器体系**（约 300 个），对外暴露为 `information_schema.INNODB_METRICS`，可用 `innodb_monitor_enable/disable/reset` 按名字/模块/通配符开关。

**它不是事务锁专属的**——覆盖 metadata、lock、buffer、buffer_page_io、os、transaction、purge、undo、log、compression、index、adaptive_hash、file_system、change_buffer、dml、ddl 等二十来个子系统。（锁文档 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md) 只引用其中 lock 相关的部分。）

### 用途

在系统已经出问题（hang、死锁、内存泄漏、I/O 停滞）时，提供**现场取证**能力：Monitor 给"此刻全系统长什么样"的全景，counter 给"各类事件发生了多少次/多久"的量化趋势。设计上的第一优先级是**不能让诊断本身把系统搞挂**（见 `MUTEX_NOWAIT` 降级）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.x | 只有 `SHOW ENGINE INNODB STATUS` 文本 + server 层 status variable；`srv_monitor_interval` 可调（8.0 已移除，改硬编码 15s） |
| 5.5/5.6 | **monitor counter 体系引入**（`srv0mon.h`，2009 年创建）+ `INNODB_METRICS` 表，把老 status variable 以 `MONITOR_EXISTING` 方式纳入同一套开关/reset 语义 |
| 8.0 | `data_locks`/`data_lock_waits` 走 PFS 虚拟表（server 定义表 + 引擎填充）；`SHOW STATUS` 重写为 PFS 查询；`srv_lock_timeout_thread` 包装函数移除，线程体直接是 `lock_wait_timeout_thread` |

---

## 理论基础

### 设计思想与权衡

**① 三套体系并存而非统一**。MySQL 8.0 观测同一个"锁等待"事实有三条路：Monitor 文本、monitor counter、PFS `data_locks` 表。被否决的方案是"统一到一个体系"——代价是历史兼容断裂（5.x 脚本依赖 SHOW ENGINE 文本）与开销不可控（PFS 的 per-event 精度在"只需要一个计数"的场景是浪费）。代价是**三套开关、三种语义、同名指标口径不一**（如 `lock_rec_lock_waits` 计"入队次数"、`Innodb_row_lock_waits` 计"实际挂起次数"，超时场景两者不一致）。

**② 埋点复用业务锁，不为监控引入新锁**（monitor counter 的核心决策）。头注释原话："we do not provide any synchronization for these monitor operations due to performance consideration. Most counters can be placed under existing mutex protections in respective code module." 即把监控正确性寄生在 InnoDB 已有的锁协议上。

**③ 分级精度："计数要准，极值可以不准"**。`fetch_add` 保证计数不丢，max/min 用非原子比较写并留出注释承认竞态——计数用于阈值与趋势（必须可靠），极值是辅助参考（偶发偏小可接受）。

**④ 成本从写路径挪到读路径**。`MONITOR_EXISTING` 计数器（几十个老 status variable）**平时零埋点开销**，只在开关与查询时刻做差分算术；`MONITOR_INC_NOCHECK` 把 N 次开关判定降为 1 次（模块级判断 + 多路免检自增）。

**⑤ 诊断不能变成新的 hang**（Monitor 输出的核心约束）。见下文 `MUTEX_NOWAIT` 与 LOCK_DATA 的"尽力而为"。

**失效场景与退化**：monitor counter 无 per-thread/per-table 维度、无 event 语义（答不了"哪次等待最久"，只能给聚合）；无锁路径会丢计数；`MONITOR_ON/OFF` 非原子（并发 `SET GLOBAL` 与埋点有理论竞态）。

### 理论溯源

- **观测者效应（observer effect）**：监控自身污染被观测对象。server 层 `SHOW STATUS` 的应对是"语句前快照 + 增量转移"（见 variables.md），InnoDB 侧的应对是 `MUTEX_NOWAIT` 与"绝不因打印而阻塞"。
- **data-oriented design**：`monitor_id_t` 枚举值直接当数组下标，热数据（值）与冷数据（名字/描述）彻底分离——与 server 层 `System_status_var` 连续内存块 + `offset` 定位同构。
- **依赖倒置（DIP）**：PFS 锁表是"server 定义表结构、引擎实现 iterator 填充"，引擎不认识 `TABLE`/`Field`。

### 他库对比

| 数据库 | 全景诊断 | 计数器体系 | 关系型暴露 |
|---|---|---|---|
| **MySQL/InnoDB** | `SHOW ENGINE INNODB STATUS`（文本） | monitor counter（`INNODB_METRICS`）+ status var | PFS 虚拟表（可 SQL） |
| **PostgreSQL** | 无等价物（靠 `pg_stat_*` 视图） | `pg_stat_*` 视图 + 计数器 | 全是视图，天然可 SQL |
| **Oracle** | AWR/ASH report（文本 + 表） | `v$sysstat` / `v$sesstat` | 全是视图 |

MySQL 是"文本 + 计数器 + PFS 表"三套并存；PG 从一开始就是视图（可 SQL 但缺"一整块全景快照"）；Oracle 的 AWR 是"定期快照到表"。

---

## 核心实现

### 主链路

```
【Monitor 输出】
srv_monitor_thread（15s 硬编码周期，或 srv_innodb_needs_monitoring 引用计数触发）
  → srv_printf_innodb_monitor(file, nowait, &trx_start, &trx_end)
      ├ BACKGROUND THREAD ← srv_print_master_thread_info
      ├ SEMAPHORES        ← sync_print
      ├ LATEST FOREIGN KEY ERROR（条件）← ut_copy_file(dict_foreign_err_file)
      ├ TRANSACTIONS      ← srv_printf_locks_and_transactions
      │    ├ LATEST DETECTED DEADLOCK（条件）← ut_copy_file(lock_latest_err_file)
      │    └ LIST OF TRANSACTIONS ← lock_print_info_all_transactions
      ├ FILE I/O ← os_aio_print
      ├ INSERT BUFFER AND ADAPTIVE HASH INDEX ← ibuf_print + ha_print_info
      ├ LOG ← log_print
      ├ BUFFER POOL AND MEMORY ← buf_print_io
      └ ROW OPERATIONS
  → 写 stderr（innodb_status_output）或 srv_monitor_file（中转区）

SHOW ENGINE INNODB STATUS 走另一条：
handlerton::show_status → innodb_show_status()
  → nowait=false 现场生成到 srv_monitor_file → 按 trx 起止偏移智能截断(1MB) → 读回发给客户端

【monitor counter】
埋点 MONITOR_INC(id) → monitor_inc_value → MONITOR_IS_ON 位图判断 → relaxed load+add+store（+max）
查询：SELECT ... FROM information_schema.INNODB_METRICS
  → i_s_metrics_fill → 逐计数器读 monitor_value_t（EXISTING 的先做差分）→ 填 17 列
```

### Monitor 线程体系

#### 三个 monitor 类线程

InnoDB 有二十多个后台线程，与观测/死锁直接相关的是这三个（都在 `srv0start` 创建，只读模式不启）：

| PSI 线程名 | 线程体 | 周期 | 职责 |
|---|---|---|---|
| `ib_srv_lock_to` | `lock_wait_timeout_thread` | **1 秒** | 锁超时扫描 + wait-for 图分析 + CATS 权重刷新 + 死锁检测 |
| `ib_srv_err_mon` | `srv_error_monitor_thread` | **1 秒** | LSN 单调性、LRU 统计、**latch 层死锁环检测**、长信号量等待告警与 fatal crash |
| `ib_srv_mon` | `srv_monitor_thread` | **15 秒**（硬编码） | 周期性把 Monitor 全文写到 stderr / status 文件 |

两个易踩的坑：

1. **`srv_lock_timeout_thread` 这个函数不存在**——它只是 PFS key 与线程名（`ib_srv_lock_to`）的历史命名残留，真正的线程体就是 `lock_wait_timeout_thread()`。
2. **monitor 周期不可调**——8.0.39 里 `srv_monitor_interval` 已不存在，周期是 `const auto sleep_interval = std::chrono::seconds{15}`。

`srv_error_monitor_thread` 除了周期巡检还做两件关键事——它是"系统已经 hang 住时最后的诊断手段"：

- **`sync_array_detect_deadlock()`**：**latch/mutex 层**的死锁环检测（DFS + 奇偶三色标记），与事务行锁死锁是两回事——latch 死锁无法自愈，检测到直接 `ib::fatal` 自杀。
- **长信号量等待的三级响应**：4 分钟（硬编码 `constexpr std::chrono::minutes timeout{4}`）→ warning + 打印 cell；超过 `srv_fatal_semaphore_wait_threshold`（**600 秒**，生产恒为 600，仅 debug 构建可调）→ 计 fatal；**同一个 waiter 等同一个 sema 连续观测到 11 次**才真正崩溃（约再等 10 秒，给慢 I/O 留一线生机）。触发时会 `srv_innodb_needs_monitoring++` **强开 30 秒 Monitor 输出**并 `lock_set_timeout_event()` 踢一脚死锁线程。

#### `MUTEX_NOWAIT`：诊断不能变成新的 hang

```cpp
/** Maximum number of times allowed to conditionally acquire
mutex before switching to blocking wait on the mutex */
#define MAX_MUTEX_NOWAIT 20
#define MUTEX_NOWAIT(mutex_skipped) ((mutex_skipped) < MAX_MUTEX_NOWAIT)
```

Monitor 默认以 **try-latch** 方式抓 `lock_sys` 全局 latch，抓不到就跳过锁段打印；**连续失败 20 次后才允许阻塞等待**。原因写在注释里：Monitor 常常是在系统已卡住时触发的（如 `sync_array_print_long_waits`），如果 Monitor 自己阻塞在全局 latch 上，诊断信息就永远打不出来。

#### 输出组织：单体拼装 + 临时文件中转

`srv_printf_innodb_monitor()` **一个函数硬编码了全部段落顺序**（没有注册表，加一个段就得改这个函数，见主链路图）。

**"中转临时文件"是这套设计的三个实例**：`srv_monitor_file`（Monitor 全文）、`lock_latest_err_file`（死锁报告）、`dict_foreign_err_file`（外键错误）都用 `os_file_create_tmpfile()` + `rewind()` + `ut_copy_file()` 这套组合。为什么用 `FILE*` 而不是内存 buffer？因为输出长度**高度不可预测**（事务数 × 每事务锁数 × 每锁记录内容），而 `FILE*` 天然提供可增长缓冲与 `ftell/fseek` 随机访问——SHOW 命令靠 `ftell` 记录事务列表起止偏移，超长时才能"从中间挖掉事务列表、保留更有价值的尾部"。

`srv_monitor_file` 有两种形态（启动时二选一）：默认 `innodb_status_file=OFF` → `os_file_create_tmpfile()` 匿名临时文件（纯粹当 SHOW 的中转缓冲区）；`=ON` → datadir 下具名的 `innodb_status.<pid>`，既当中转区又被周期线程覆写，供外部持续观察。

#### SHOW ENGINE INNODB STATUS 的读取路径

`handlerton::show_status` → `innodb_show_status()`：现写临时文件 → 按 `trx_list_start/end` 智能截断（上限 `MAX_STATUS_SIZE = 1048576`）→ 读回内存发客户端。三处细节：

- **SHOW 用 `nowait=false`**（阻塞路径）：用户显式查询时宁可等全局 latch，也要保证锁段完整——与周期线程的降级相反。
- **SHOW 不需要 `innodb_status_output=ON`**：它绕过那个开关（周期输出才受它控制）。
- 截断策略：若事务列表不太长，保留输出尾部（ROW OPERATIONS / BUFFER POOL 等总览更有价值），从中间挖掉事务列表开头并插入 `"... truncated...\n"`。

#### 触发方式

1. **周期**：15s，条件是 `srv_print_innodb_monitor`（`innodb_status_output=ON`）或 `srv_innodb_needs_monitoring > 0`。
2. **显式 SQL**：`SHOW ENGINE INNODB STATUS`。
3. **InnoDB 内部自举**：`srv_innodb_needs_monitoring` 是引用计数——长信号量等待（强开 30s）、buffer pool 被锁堆/AHI 占用超 67%、找不到空闲 block（`n_iterations > 20`）三处 `++`。
4. **`lock_set_timeout_event()`**：不直接触发输出，而是唤醒 `lock_wait_timeout_thread` 立刻做一轮 wait-for 分析（在长等待场景与 `++` 配对出现，效果上是"让下一轮输出的锁段是最新的"）。
5. **不存在** KILL 触发（`srv_printf_innodb_monitor` 全仓库只有 3 个调用点）；错误场景也不 dump monitor，信号量 fatal 与 latch 环死锁都是直接 abort。

### monitor counter 体系

#### 结构：ID 即下标 + 位图开关

```cpp
/** Two monitor structures are defined in this file. One is
"monitor_value_t" which contains dynamic counter values for each
counter. The other is "monitor_info_t", which contains static information...
In addition, an enum datatype "monitor_id_t" is also defined, it identifies
each monitor with an internally used symbol, whose integer value indexes
into above two structure */
```

三位一体靠一个整数下标绑定：`monitor_id_t` = `innodb_counter_value[id]`（热的动态值）= `innodb_counter_info[id]`（冷的静态元信息：名字/子系统/描述/类型标志）。两个数组必须同步维护，靠编译期 `static_assert(UT_ARR_SIZE(innodb_counter_info) == NUM_MONITOR)` 与运行期 `ut_a(count == monitor_info->monitor_id)` 双重守卫。`NUM_MONITOR` 是枚举最后一个成员，自动成为数组维度。

`monitor_value_t` 里只有 `mon_value` 是 `std::atomic`，其余（max/min/基线/时间戳）都是普通 `int64_t`：

| 字段 | 支撑的 I_S 列 |
|---|---|
| `mon_value`（本轮增量） | `COUNT_RESET` |
| `mon_value + mon_value_reset`（+历史基线） | `COUNT` |
| `mon_max/min_value`（本轮极值） | `MAX/MIN_COUNT_RESET` |
| `mon_max/min_value_start`（惰性推进） | `MAX/MIN_COUNT` |
| `mon_start_time` / `mon_stop_time` / `mon_reset_time` | `TIME_ENABLED` / `TIME_DISABLED` / `TIME_RESET` |

未初始化的极值是哨兵 `MIN_RESERVED = INT64_MAX` / `MAX_RESERVED = ~INT64_MAX`（名字指"用在哪个字段上"而非大小），I_S 里填 **NULL 而不是 INT64_MIN/MAX**。

开关是**位图** `monitor_set_tbl`（每计数器 1 bit，全部落在 1~2 个 cache line）：`MONITOR_IS_ON` 成本 = 一次数组索引 + AND。四个系统变量 `innodb_monitor_enable/disable/reset/reset_all` 支持三种写法：

| 写法 | 路径 | 语义 |
|---|---|---|
| `'lock_deadlocks'` | 精确匹配（大小写不敏感） | 单开一项 |
| `'module_lock'` | 命中模块标记行 → `srv_mon_set_module_control` | 展开该模块区间所有成员 |
| `'lock%'` | 含 `%` → 通配符逐个单开 | 注意：已开启项会报错中断（与模块路径"跳过并打 info"行为不同） |
| `'all'` | `MONITOR_ALL_COUNTER` | 全表 |

**默认开启规则**：绝大多数 `MONITOR_EXISTING` 计数器默认开；纯 InnoDB 内部计数器默认关（`srv_mon_default_on()` 按标志位逐个置位，不展开模块——故"默认集合"与"模块集合"是正交的两套划分）。

#### 计数与同步：无锁累加

```cpp
inline void monitor_inc_value_nocheck(monitor_id_t monitor, mon_type_t value,
                                      bool set_max = true) {
  /* We use std::memory_order_relaxed load() and store() as two separate steps,
  instead of single atomic fetch_add operation, because we want to leave it
  non-atomic as it was before changing mon_value to std::atomic.*/
  const auto new_value = MONITOR_VALUE(monitor).load(std::memory_order_relaxed) + value;
  MONITOR_VALUE(monitor).store(new_value, std::memory_order_relaxed);
  if (set_max) monitor_set_max_value(monitor, new_value);
}
inline void monitor_inc_value(monitor_id_t monitor, mon_type_t value) {
  MONITOR_CHECK_DEFINED(value);
  if (MONITOR_IS_ON(monitor)) monitor_inc_value_nocheck(monitor, value);
}
```

- `std::atomic` 在这里**不是为了原子 RMW，只为消除 C++ 数据竞争的 UB**——注释明说"故意用 relaxed load+store 两步而不是 fetch_add，以保持它原来的非原子语义"。所以无锁路径仍会丢计数（与改 atomic 之前一致，被明确接受）。
- 真正需要原子累加时用 `MONITOR_ATOMIC_INC`（`fetch_add`，x86 上一条 `lock xadd`）。注释把规则写死：**已有互斥保护 → `MONITOR_INC`；没有 → `MONITOR_ATOMIC_INC`**。
- max/min 是非原子"读-比较-写"，注释承认 `This is not 100% accurate because of the inherent race, we ignore it due to performance`。
- `MONITOR_INC_NOCHECK` 用于"模块级一次判断 + 多路免检自增"（全仓唯一用例是 buffer page I/O 的 39 选 1 计数器）。

#### `MONITOR_EXISTING`：把老 status variable 搭进同一张表

一批计数器（枚举名前缀 `MONITOR_OVLD_*`，OVLD = overloaded）**没有埋点**，只在开关/查询时刻读老的全局量做差分：

```cpp
/* 当前值 = 全局量 - reset基线 - 开启时快照 + 上次关闭时保存值 */
#define MONITOR_SET_DIFF(monitor, value)                                       \
  MONITOR_SET_UPD_MAX_ONLY(monitor, ((value)-MONITOR_VALUE_RESET(monitor) -    \
                                     MONITOR_FIELD(monitor, mon_start_value) + \
                                     MONITOR_FIELD(monitor, mon_last_value)))
```

例如 `Innodb_row_lock_waits` 与 `INNODB_METRICS` 的 `lock_row_lock_waits` 读的是**同一个** `srv_stats.n_lock_wait_count`——是同一份数据的两种暴露，不是两套实现。开销从写路径挪到了读路径。

#### INNODB_METRICS 表

server 侧 `i_s.cc` 定义 17 列，`i_s_metrics_fill()` 逐计数器填充（需 `PROCESS` 权限，不满足时静默返回空结果）。要点：

- 行数 = `NUM_MONITOR` − 模块标记行 − 隐藏行（`latch` 计数器是 `MONITOR_HIDDEN`，因为它会拉起重量级的 `MutexMonitor` 采集）。
- `COUNT` 与 `COUNT_RESET` 的双字段设计让一次 `innodb_monitor_reset` 既给出"最近窗口增量"又不丢"自启动总量"。
- `AVG_COUNT` 三种口径：SET_OWNER 用 `总量/次数计数器`；普通累计型用 `COUNT/TIME_ELAPSED`（每秒速率）；`DISPLAY_CURRENT` 或 `NO_AVERAGE` 为 NULL。
- `TYPE` 列：`value`（DISPLAY_CURRENT）/ `status_counter`（EXISTING）/ `set_owner` / `set_member` / `counter`。

### 与 server 层体系的关系

**server 层没有叫 "monitor" 的东西**，对等物是三套已在别处详写的机制：

| 层 | 机制 | 归属文档 |
|---|---|---|
| server | 状态变量（`System_status_var` 连续计数器块、`add_to_status` 线性聚合、`SHOW STATUS` 重写为 PFS 查询） | [`../server/infra/variables.md`](../server/infra/variables.md) G 节 |
| server | PFS instrument / consumer 体系 | [`../server/infra/pfs.md`](../server/infra/pfs.md) |
| InnoDB | **Monitor 输出 + monitor counter**（本篇） | 本篇 |
| 跨层 | PFS 锁表（`data_locks`）：server 定义 `Plugin_table` + 4 个 HASH 索引，InnoDB 注册 inspector 填充，WHERE 通过 `accept_*()` 下推 | 引擎侧实现见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md) |

**InnoDB 怎么接进 server 的 status 体系**：`innodb_export_status()`（`srv_export_innodb_status()` → `export_vars` 全局结构）在每次 `SHOW STATUS` 时被调用，把 `srv_stats.*` 搬进 `export_vars`，后者通过 `SHOW_VAR` 注册进 server 的 `all_status_vars` 数组。这就是"InnoDB 状态变量"的来源——**值在 InnoDB，注册与展示在 server**。

**一个必须纠正的流传说法**：`lock_report_wait_for_edge_to_server()` 名字里有 "to_server"，但它**不是上报给 PFS**——它落到 `thd_report_lock_wait()`，只在 `is_mts_worker()` 双方都为真时触发 `Commit_order_manager::check_and_report_deadlock()`（复制 MTS 的跨子系统死锁检测）。非复制场景是完全的空操作。同理 `thd_wait_begin(THD_WAIT_ROW_LOCK)` 是 **thread pool 插件回调**，不是 PFS instrument——8.0 的 PFS 里没有 InnoDB 行锁等待 instrument（`wait/lock/*` 只有 `wait/lock/metadata/sql/mdl` 是真 instrument）。

---

### 与 PFS 的能力重叠与分工（两套测量模型）

能力重叠是真实的，但重叠的是**观测对象**，不是**测量模型**：

| | InnoDB monitor counter | Performance Schema |
|---|---|---|
| **模型** | **计数器模型**：`MONITOR_INC(x)` 单点无状态 | **事件模型**：`begin/end` 成对 + 计时 + 多维聚合 |
| 归属 | InnoDB 私有（`srv0mon.*`） | server 层通用框架（`storage/perfschema`），跨引擎 |
| 维度 | **无维度**：一个 int64（+ min/max/reset 附属值） | thread / account / user / host / object instance / global |
| 计时 | 基本没有 | 每个 wait 都有 `timer_start/timer_end` |
| 埋点位置 | **能贴到算法内部分支** | 只能在 API 边界 |
| 默认 | 部分 `MONITOR_DEFAULT_ON` | mutex/rwlock/cond **默认关**，file **默认开** |

**决策点：为什么 monitor 能覆盖 PFS 结构性做不到的一批信息**（因为它们埋在算法判定分支里，而 PFS 只能在 API 边界插桩）：

- `lock_deadlock_false_positives`——"启发式判定这是个假环"的 else 分支计数，PFS 没有"死锁检测假阳性"这个事件。
- `lock_schedule_refreshes`——CATS 调度权重重算次数，InnoDB 内部概念。
- `buffer_page_io` 的 **37 种页类型**分类（17 读 + 17 写 + 3 on-log）——PFS 只有 `wait/io/file/innodb/innodb_data_file` 一个数，分不清读的是 BLOB 页还是 XDES 页。
- `MONITOR_*_NO_WAITS / WAITS / WAIT_LOOPS` 三连（`MONITOR_INC_WAIT_STATS`）——log/flush 子系统的"自旋了没有 / 自旋几轮"，PFS 只有一个总等待时间。
- **mutex 自旋次数**——`SHOW ENGINE INNODB MUTEX` 的 `spins=`，PFS **完全看不到自旋**（它的 begin/end 在 mutex 外边界，对内部自旋循环一无所知）。

反过来 **PFS 能覆盖 monitor 做不到的**：per-thread/per-account 归属、延迟分布（min/avg/max）、关系型明细（`data_locks` 可 SQL 过滤与 JOIN）、按具体文件/具体 mutex 实例细分、跨引擎统一视图。

**耦合点只有三处，且都是手工的**（源码核实：`srv0mon.*` 里 grep `PSI_/pfs_` **0 命中**——monitor 与 PFS 之间**没有任何自动注册机制**）：

1. `lock_t::m_psi_internal_thread_id / m_psi_event_id`：建锁时（记录锁与表锁各一处）调 `PSI_THREAD_CALL(get_current_thread_event_id)` 打点。这是**纯 PFS 字段、与 monitor 无关**，但它让 `data_locks` 能 JOIN `events_statements_*` 定位"哪条 SQL 加的这把锁"——代价是每个 `lock_t` 大 16 字节 + 建锁路径一次间接调用。
2. `MONITOR_LATCHES` 开启时 `mutex_monitor->enable()`：**monitor 体系里唯一牵动重量级采集的地方**（遍历 `latch_meta` 全部 ~125 个 latch 类，把每个已注册实例的采集开关打开）。它默认关（`MONITOR_HIDDEN` 且非 `DEFAULT_ON`）。
3. 两边是**手工维护的两份清单**：InnoDB 手写 `all_innodb_mutexes[]` / `all_innodb_files[]` 等数组再 `mysql_xxx_register("innodb", ...)`；debug 构建还有一道断言防止新增 PSI key 忘了注册——这恰恰证明没有自动同步。

#### 选型决策树

```
要回答的问题
├─ "系统整体趋势如何？"（常开、零成本）
│     → monitor：INNODB_METRICS
├─ "内部算法哪里不对劲？"
│     ├─ 死锁假阳性 / 调度重算 / log 自旋轮数 / 页类型分布 / flush&LRU 细分 → monitor（唯一数据源）
│     └─ mutex 自旋次数 → monitor：SET GLOBAL innodb_monitor_enable='latch' + SHOW ENGINE INNODB MUTEX
├─ "是谁？哪条 SQL？哪个账号？"          → PFS（events_* / threads / data_lock_waits JOIN）
├─ "慢在哪？耗时多少？"                  → PFS（TIMER_WAIT 系列）
├─ "哪个文件 / 哪个 mutex 实例？"        → PFS（file_summary_by_instance / mutex_instances）
├─ "现在谁阻塞谁？"                      → PFS data_locks + data_lock_waits（低频 + 带 WHERE）
│     （高频探测改用 monitor 的 lock_threads_waiting / Innodb_row_lock_current_waits）
└─ "刚才那个死锁怎么回事？"              → Monitor 文本（SHOW ENGINE INNODB STATUS / innodb_print_all_deadlocks）
```

**开销量级**（选型的关键判据）：

| 操作 | 成本 |
|---|---|
| `MONITOR_INC`（默认开的那批） | 1 次位图 AND + relaxed load/store ≈ **3~5 条指令**，无原子 RMW |
| `MONITOR_ATOMIC_INC` | `lock xadd`，只在无锁保护处用 |
| PFS mutex instrument 关闭时 | 两次分支判断 ≈ 1~2 条指令 |
| PFS mutex instrument **开启** | 2 次间接调用 + **2 次时钟读取** + ≥4~6 次聚合写（比 monitor 高 1~2 个数量级） |
| `SELECT * FROM data_locks` | **持 `locksys` 全局排他 latch + `trx_sys->mutex`**，按 256 事务一批反复扫；1 万事务约 40 批 |

所以：latch 类 instrument 除非排查争用否则别开（默认就是关的）；`data_locks` 别做高频轮询（会短暂冻结整个锁子系统）；"有没有锁等待"的高频探测用计数器。

> 提醒：MTR 环境 `default_mysqld.cnf` 里有 `loose-performance-schema-instrument='%=ON'`，所以 MTR 结果里 `wait/synch/mutex/innodb/*` 显示 YES **不代表生产默认值**（生产默认 mutex/rwlock 是 NO，只有 file 类是 YES）。

## ★ 本机制里的工程实现技法

### 一、"ID 即下标"的三位一体与编译期守卫

`monitor_id_t` → 两个数组同下标，用 `static_assert` + `ut_a` 把"两个数组必须同步维护"这个最容易出错的约束变成硬失败。代价是加计数器必须同时改两处；收益是零间接寻址、零哈希查找，热路径只碰一个 `int64`。

### 二、位图开关 + 无锁累加：用"极简"换掉"通用"

对比 PFS 的 instrument 注册 + consumer 过滤体系，monitor counter 用 1 bit 表示开关、一次 AND 判定。埋点是编译期决定的，开关只是运行期的一个 bit——**关着的计数器仍有一次 AND 的开销（不可避免），但没有任何函数调用、指针追逐或 branch table**。

### 三、复用业务互斥锁而非为监控加锁

这是整套体系最核心的性能决策。它把监控正确性寄生在 InnoDB 已有的锁协议上（lock/buf/log/trx 各自临界区内的埋点天然安全）。`std::atomic` 的引入也只是为了消除 UB——**原子化不等同于语义变更**，这是对既有正确性的保守保护。

### 四、把成本从写路径挪到读路径

`MONITOR_EXISTING` 的惰性差分（几十个老 status var 零埋点开销）、`MONITOR_INC_NOCHECK` 的模块级单次判定、以及 `MEB/hotbackup` 构建下 `#define MONITOR_INC(x) ((void)0)` 的编译期整体关闭——三种手法同构。

---

## 可观测性

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `innodb_status_output` | OFF | 周期线程是否把 Monitor 输出写进 error log（15s 一次，慎开） |
| `innodb_status_output_locks` | OFF | 锁段是否逐个打印每把锁（每事务最多 10 把）；与 `innodb_status_output` 在源码中完全独立 |
| `innodb_status_file` | OFF | 是否用 datadir 下具名 `innodb_status.<pid>` 代替匿名临时文件 |
| `innodb_monitor_enable` / `_disable` / `_reset` / `_reset_all` | NULL | 计数器开关（名字 / 模块名 / `%` 通配） |
| `innodb_print_all_deadlocks` | OFF | 死锁报告同步刷 error log（否则只进 `lock_latest_err_file`） |
| `innodb_semaphore_wait_timeout_debug` | 600 | **仅 debug 构建**；生产的 `srv_fatal_semaphore_wait_threshold` 恒为 600 |

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 全系统全景快照 | SQL | `SHOW ENGINE INNODB STATUS`（现场生成，1MB 上限） |
| 某类事件的次数/极值/均值 | SQL | `information_schema.INNODB_METRICS`（按 `SUBSYSTEM` 过滤，如 `lock`） |
| 周期性的历史留痕 | trace | `innodb_status_output=ON` → error log；`innodb_status_file=ON` → 文件 |
| 死锁留痕 | trace | `innodb_print_all_deadlocks=ON` |

---

## Misc

### 易混淆的命名与口径

- **`module_lock` vs `lock`**：前者是计数器名（模块标记行，不出现在表里），后者是 `SUBSYSTEM` 列名。
- **`lock_rec_lock_waits` vs `Innodb_row_lock_waits`**：前者 = 入队次数（每次进等待队列就 +1），后者 = 实际挂起次数（真睡下去才计）。超时场景两者明显不一致。
- **`lock_deadlock_false_positives` 不为 0 是正常的**：它记录"候选环被两道校验否决"，恰恰证明快照式并发检测撞到过假环。
- **`monitor` 不是 server 层概念**：`SHOW STATUS` 是 server 层的，`SHOW ENGINE INNODB STATUS` 是 InnoDB 的，两者通过 `export_vars` + `SHOW_VAR` 衔接。

### 面向二次开发

**扩展点**：加一个计数器 = `monitor_id_t` 末尾前插入 + `innodb_counter_info[]` 同下标处插入一行（名字/子系统/描述/类型标志）+ 埋点 `MONITOR_INC`。加一个 Monitor 段落 = 改 `srv_printf_innodb_monitor()`（没有注册表，必须改单体函数）。

**坑与已知缺陷**：

- 模块标记行 `module_cpu` / `module_page_track` / `module_dblwr` 的 `monitor_type` 写的是 `MONITOR_NONE` 而非 `MONITOR_MODULE`——它们**不被当作模块边界**（无法用模块名批量开关），且会作为普通 counter 行出现在表里。
- `MONITOR_NO_AVERAGE` 标志在 8.0.39 **没有任何计数器使用**（预留能力）。
- 已开启的计数器再 `enable` 会被拒绝（模块路径打 info、通配符路径报错中断），避免误触发隐式 reset。
- 生产环境 `srv_fatal_semaphore_wait_threshold` 恒为 600 不可调——长等待崩溃的阈值改不了（除非 debug 构建）。

**社区边界澄清**：`SHOW ENGINE INNODB STATUS` 的文本格式、monitor counter 的名字、`INNODB_METRICS` 都是社区版能力；Percona/MariaDB 有各自的额外计数器与 `SHOW ENGINE INNODB STATUS` 扩展段（如 Percona 的 `INNODB_CHANGED_PAGES`），社区版没有。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB INFORMATION_SCHEMA Metrics Table*
- *MySQL 8.0 Reference Manual → SHOW ENGINE INNODB STATUS*

**内核月报 / 技术文章**
- MySQL 官方 blog：InnoDB Metrics（monitor counter 的设计说明）

**相关文档**
- server 层状态变量体系见 [`../server/infra/variables.md`](../server/infra/variables.md) G 节
- PFS instrument 体系见 [`../server/infra/pfs.md`](../server/infra/pfs.md)
- PFS 锁表（`data_locks`）的引擎侧实现见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- 死锁检测用到的 monitor 计数（真死锁/假阳性）见 [`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
