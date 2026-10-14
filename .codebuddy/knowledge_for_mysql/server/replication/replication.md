# MySQL 复制拓扑与故障转移深度解析

> 基于 MySQL 8.0.39 源码，涵盖复制系统层：复制协议与握手、两种位点定位（file+pos / GTID auto position）、拓扑形态（直连/级联/多源/双主环）、复制过滤器、延迟复制、GTID_ONLY 模式、故障转移流程与运维操作。
>
> **边界**：本篇讲复制的**系统层**——主库、从库、拓扑、定位如何组成一个整体，以及 failover 怎么做。单点机制交叉引用：主库 dump 线程（`Binlog_sender`）与半同步见 [`binlog.md`](binlog.md)；从库 IO/SQL 线程、relay log、位点持久化、relay log recovery 见 [`replica.md`](replica.md)；并行复制（MTS）见 [`prpl.md`](prpl.md)；GTID 集合算法与 exclude 集合协议见 [`gtid.md`](gtid.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路：一次复制建立的完整流程](#主链路一次复制建立的完整流程)
  - [复制协议与握手](#复制协议与握手)
  - [位点定位：file+pos vs GTID auto position](#位点定位filepos-vs-gtid-auto-position)
  - [拓扑形态](#拓扑形态)
  - [复制过滤器](#复制过滤器)
  - [延迟复制](#延迟复制)
  - [GTID_ONLY 模式](#gtid_only-模式)
  - [故障转移](#故障转移)
  - [运维操作](#运维操作)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 复制（replication）是"一台服务器（source/主库）把已提交事务的事件流，通过网络传给另一台或多台服务器（replica/从库）重放"的机制。它是**异步事件流**：从库是主动的"拉取者"，主库是被动的"提供者"。

单点视角的四篇文档已经分别讲了主库写（binlog）、从库接（replica）、并行回放（prpl）、事务标识（gtid）。本篇是**系统层视角**，回答四个单点文档各自讲不了的问题：

1. **协议**——从库和主库之间到底说什么（握手、请求、心跳、事件流）
2. **定位**——从库"从哪里开始复制"这个问题怎么解决（file+pos 与 GTID 自动定位）
3. **拓扑**——多台服务器怎么组织（级联、多源、双主环）
4. **切换**——主库挂了之后，怎么让系统继续运转（failover）

**★ 源码对复制的权威定义**（`sql/rpl_replica.h:81`，★ 一段话把两个线程讲清楚）：

```cpp
/*****************************************************************************

  MySQL Replication

  Replication is implemented via two types of threads:

    I/O Thread - One of these threads is started for each master server.
                 They maintain a connection to their master server, read log
                 events from the master as they arrive, and queues them into
                 a single, shared relay log file.  A Master_info represents
                 each of these threads.

    SQL Thread - One of these threads is started and reads from the relay log
                 file, executing each event. A Relay_log_info represents this
                 thread.

  Buffering in the relay log file makes it unnecessary to reread events from
  a master server across a slave restart.  It also decouples the slave from
  the master where long-running updates and event logging are concerned--ie
  it can continue to log new events while a slow query executes on the slave.

*****************************************************************************/
```

> ★★ **relay log 的两个作用**（常被只说成"中转站"）：① **重启后不必重读**（跨重启）；② **解耦**——从库上跑慢查询时主库照样能继续发新事件。这是"异步"能成立的关键缓冲。
> ★ 两个线程各有各的状态对象：**`Master_info` = IO 线程**（每主一个），**`Relay_log_info` = SQL 线程**。多源复制下"每 channel 一个 `Master_info`"就是从这里来的。

**★ 两个状态对象的官方职责**（`sql/rpl_mi.h:55` / `sql/rpl_rli.h:132`）：

```cpp
/*****************************************************************************
  Replication IO Thread

  Master_info contains:
    - information about how to connect to a master
    - current master log name
    - current master log offset
    - misc control variables
  ...
  To write out the contents of master.info to disk a call to flush_info()
  is required. Currently, it is needed every time we read and queue data
  from the master.
```
```cpp
/*******************************************************************************
Replication SQL Thread

Relay_log_info contains:
  - the current relay log
  - the current relay log offset
  - master log name
  - master log sequence corresponding to the last update
  - misc information specific to the SQL thread
  ...
To correctly recovery from failures, one should combine transactional system
tables along with the --relay-log-recovery.
*******************************************************************************/
class Relay_log_info : public Rpl_info {
```
> ★ 最后一句是官方给出的**可靠性配方**：**事务型信息库 + `relay_log_recovery`**。
> ★ `Master_info` 的职责里"connection info + 位点 + 控制变量"三件事，正好对应 `CHANGE REPLICATION SOURCE` 的三类参数。

**★ 几个硬编码常量**（`sql/rpl_replica.h:61`）：

```cpp
#define SLAVE_MAX_HEARTBEAT_PERIOD 4294967     /* ★ 心跳周期上限（ULONG_MAX/1000 换算成秒） */
#define REPLICA_NET_TIMEOUT 60                 /* ★ replica_net_timeout 默认值 */
#define MAX_SLAVE_ERROR 14000
#define MTS_WORKER_UNDEF ((ulong)-1)
#define MTS_MAX_WORKERS 1024                   /* ★★ replica_parallel_workers 硬上限 1024 */
#define MAX_SLAVE_RETRY_PAUSE 5
#define MTS_MAX_BITS_IN_GROUP ((1L << 19) - 8) /* 524280 —— 65535 字节 BLOB × 8 bit */
```

### 用途

- **高可用**：主库故障时切换到从库（RTO 分钟级）
- **读写分离**：主写从读，分摊负载
- **备份**：从库上做物理备份，不干扰主库
- **数据分发**：把数据分发到多个数据中心/异构系统（通过级联 + 过滤 + 延迟）

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | **GTID 引入**（全局事务标识，failover 拓扑无关化的基础）；多线程从库（MTS）首版按库并行 |
| 5.7 | **多源复制**（CHANNEL 概念落地）；MTS 改为逻辑时钟并行；半同步增强 |
| 8.0.22-23 | **术语重命名**：`CHANGE MASTER TO` → `CHANGE REPLICATION SOURCE TO`、`SHOW SLAVE STATUS` → `SHOW REPLICA STATUS`（旧语法保留但弃用）；`log_slave_updates` → `log_replica_updates` 且**默认值从 0 翻转为 1** |
| 8.0.27 | **GTID_ONLY 模式**：位点仓库不再存 binlog 文件名，强制 auto position；`SOURCE_CONNECTION_AUTO_FAILOVER`（组复制场景的连接级自动切换） |
| 8.4 LTS / 9.x | 8.4 作为 LTS 延续 8.0 全部复制语义；9.x 的复制变化跟踪 Innovation Release Notes（本篇事实以 8.0.39 源码为准） |

> 演进动机（为什么 GTID 在 5.6 引入、为什么术语重命名、为什么 `log_replica_updates` 默认翻转）见「理论基础 → 他库对比与演进动机」。

**★ 源码里的版本演进证据**（12 条，全部 `grep` 实证）：

| # | 事实 | 文件:行 |
|---|---|---|
| 1 | `binlog_format` **8.0 默认 ROW**（5.7 是 STATEMENT），且已弃用 | `sys_vars.cc:1428` `DEFAULT(BINLOG_FORMAT_ROW)` + `DEPRECATED_VAR("")` |
| 2 | `master_info_repository` 只剩 TABLE | `sys_vars.cc:1713` |
| 3 | `relay_log_info_repository` 只剩 TABLE | `sys_vars.cc:1723` |
| 4 | **`log_replica_updates` 默认 1**（8.0 翻转） | `sys_vars.cc:6153` |
| 5 | 升级兼容开关 `log_bin_use_v1_row_events` 已弃用 | `sys_vars.cc:2576` |
| 6 | `expire_logs_days` → `binlog_expire_logs_seconds` | `sys_vars.cc:2230` `DEPRECATED_VAR(...)` |
| 7 | 8.0.26 术语开关，取值硬编码 `BEFORE_8_0_26` | `sys_vars.cc:7676` |
| 8 | 8.0 新增 `binlog_row_metadata` | `sys_vars.cc:1491` |
| 9 | 8.0.20 事务压缩 `binlog_transaction_compression` | `sys_vars.cc:1512` |
| 10 | `binlog_row_value_options`（PARTIAL_JSON） | `sys_vars.cc:7164` |
| 11 | 8.0.22 异步连接故障转移 | `sql_yacc.yy:3231` |
| 12 | 8.0.27 `GTID_ONLY` | `sql_yacc.yy:3248` |

**★ 5.6 时代的历史痕迹仍在源码里**（`sql/rpl_msr.h:135`）：

```cpp
    In fact it also acts like the LOCK_active_mi of MySQL 5.6, preventing two
    replication administrative commands to run in parallel.
```

**★ 协议层为 5.6 留的后门**（`sql/rpl_binlog_sender.cc` / `rpl_replica.cc:4162`）：

> *"in versions up to 5.6 RC, the master would check the lowest bit… Therefore, for backward compatibility, if we ever start to use the flags, we should leave the three [bits]"*

> ★ 这就是 dump 请求里 `m_flag` 低 3 位**至今保留不用**的原因。

**★ 三个变量定义原文**：

```cpp
/* sql/sys_vars.cc:1428 —— 8.0 默认 ROW，且变量本身已弃用 */
static Sys_var_enum Sys_binlog_format(
    "binlog_format", "... This option is deprecated and will be removed in a future version.",
    SESSION_VAR(binlog_format), CMD_LINE(REQUIRED_ARG, OPT_BINLOG_FORMAT),
    binlog_format_names, DEFAULT(BINLOG_FORMAT_ROW), NO_MUTEX_GUARD,
    NOT_IN_BINLOG, ON_CHECK(binlog_format_check),
    ON_UPDATE(fix_binlog_format_after_update), DEPRECATED_VAR(""));
```
```cpp
/* sql/sys_vars.cc:1713 / :1723 —— 信息库只剩 TABLE */
ulong opt_mi_repository_id = INFO_REPOSITORY_TABLE;
static Sys_var_enum Sys_mi_repository(
    "master_info_repository",
    "The repository format for the replication connection configuration.",
    GLOBAL_VAR(opt_mi_repository_id),
    CMD_LINE(REQUIRED_ARG, OPT_MASTER_INFO_REPOSITORY), repository_names,
    DEFAULT(INFO_REPOSITORY_TABLE), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(master_info_repository_check), ON_UPDATE(nullptr),
    DEPRECATED_VAR(""));
```
```cpp
/* sql/sys_vars.cc:2576 —— 升级兼容开关 */
static Sys_var_bool Sys_log_bin_use_v1_row_events(
    "log_bin_use_v1_row_events",
    "If equal to 1 then version 1 row events are written to a row based "
    "binary log.  If equal to 0, then the latest version of events are "
    "written.  This option is useful during some upgrades.",
    NON_PERSIST GLOBAL_VAR(log_bin_use_v1_row_events),
    CMD_LINE(OPT_ARG, OPT_LOG_BIN_USE_V1_ROW_EVENTS), DEFAULT(false),
    NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(check_log_bin_use_v1_row_events),
    ON_UPDATE(nullptr), DEPRECATED_VAR(""));
```

---

## 理论基础

### 设计思想与权衡

#### 1. 拉模型：从库是客户端

复制协议的方向与直觉相反：**不是主库"推送"给从库，而是从库"拉取"**。从库的 IO thread 像一个普通客户端，向主库发 `COM_BINLOG_DUMP`，主库的 dump 线程像响应查询一样持续返回事件流。

为什么选拉模型：

- **主库零状态**：主库不保存"哪些从库连着我、它们各自到哪了"（注册表只有防僵尸线程的轻量信息，见协议节）。主库挂了，从库重连别的主库即可——**这是 failover 能工作的前提**。如果是推模型（如 PostgreSQL 9.0 之前），主库需要维护每个从库的发送游标，主库一挂全部重建。
- **位点由从库掌握**：从库记住自己的进度（relay log + 位点仓库），重启后自己决定从哪继续。推模型则进度在主库侧，从库崩溃恢复要靠主库配合。
- **异构能力**：从库只是"会解析 binlog 协议的客户端"，任何实现者（包括 `mysqlbinlog --stop-never`、中间件）都能接入。

代价是**单向异步**：主库不知道从库收到没有。可靠性交给上层机制补（半同步、GTID 校验），这是「一致性模型」要讨论的核心权衡。

**★★★ 拉模型的源码铁证**（`sql/rpl_replica.cc:8495`，从库 IO 线程的连接循环）：

```cpp
    if (reconnect) {
      connected = !mysql_reconnect(mysql);
    } else {
      mysql_extension_set_server_extn(mysql, &mi->server_extn);
      connected = mysql_real_connect(mysql, tmp_host, user, password, nullptr,
                                     tmp_port, nullptr, client_flag);
    }
```

> ★★ 从库调用的是**标准 libmysqlclient 客户端 API** `mysql_real_connect()`，主动向主库 IP:PORT 发起 TCP 连接。**主库侧没有任何"推"的代码路径**。

```cpp
/* sql/rpl_replica.cc:8468 —— 从库在连接属性里自报身份，P_S 可见 */
  mysql_options4(mysql, MYSQL_OPT_CONNECT_ATTR_ADD, "program_name", "mysqld");
  mysql_options4(mysql, MYSQL_OPT_CONNECT_ATTR_ADD, "_client_role",
                 "binary_log_listener");
  mysql_options4(mysql, MYSQL_OPT_CONNECT_ATTR_ADD,
                 "_client_replication_channel_name", mi->get_channel());
```

**★ "主库零状态"的源码依据**（`sql/rpl_source.cc:196`）——注册信息只进**内存 hash**：

```cpp
  mysql_mutex_lock(&LOCK_replica_list);
  unregister_replica(thd, false, false /*need_lock_slave_list=false*/);
  res = !slave_list.emplace(si->server_id, std::move(si)).second;
  mysql_mutex_unlock(&LOCK_replica_list);
```

> ★ `slave_list`（键 = server_id）只服务于 `SHOW REPLICAS`。**主库不持久化从库信息**——且注册是**可选的**（失败只告警），dump 命令自带的 server_id 才是权威来源。

**★ 重连责任完全在从库**（`sql/rpl_replica.cc:8518`，★ 注释直接点出与 failover 的关系）：

```cpp
    /*
      By default we try forever. The reason is that failure will trigger
      master election, so if the user did not set mi->retry_count we
      do not want to have election triggered on the first failure to
      connect
    */
    if (++err_count == mi->retry_count) { ... }
    slave_sleep(thd, mi->connect_retry, ...);
```

> ⚠️ **真名核查**：8.0.39 里连接函数**仍叫 `connect_to_master()`**（`rpl_replica.h:643` / `rpl_replica.cc:8343`），**不存在 `connect_to_source()`**。且 `sql/rpl_slave.cc` **不存在**——主文件是 `sql/rpl_replica.cc`（410 KB）。

#### 2. file+pos 的拓扑绑定 vs GTID 的拓扑无关——本篇最重要的理论

file+pos 模式的位点是 `(binlog 文件名, 文件内偏移)`，这是一个**拓扑绑定**的坐标：它只在"产生它的那台主库"上有意义。两台主库的 binlog 文件命名和内容完全无关——A 库的 `binlog.000003:120` 在 B 库上没有任何对应物。

所以传统 failover 的流程是"猜一个位点"：找到旧主库最后一个 binlog 事件在从库上的执行位置，换算成新主库上的等价位点。这既易错（漏算/重算事件）又依赖人工。

GTID 把坐标换成了**内容寻址**：每个事务带全局唯一 ID（`SERVER_UUID:GNO`），从库记住的是"我执行过哪些事务"（集合），而不是"我读到了哪个字节"。切换主库时，从库只需要问新主库："**给我所有不在我 exclude 集合里的事务**"——新主库用 auto position 算法（见核心实现）自己定位起点。**位点从"坐标"变成了"差集"**，拓扑无关性由此而来。

这正是 5.6 引入 GTID 的根本动机：让 failover 从"手工换算位点"变成"自动收敛"。

**★★★ 分水岭只有 4 行代码**（`Binlog_sender::check_start_file()`，`sql/rpl_binlog_sender.cc:857`）：

```cpp
  if (m_start_file[0] != '\0') {
    mysql_bin_log.make_log_name(index_entry_name, m_start_file);   /* ★ file+pos：必须用从库回传的文件名 */
    name_ptr = index_entry_name;
  } else if (m_using_gtid_protocol) {
    ...
    if (mysql_bin_log.find_first_log_not_in_gtid_set(
            index_entry_name, m_exclude_gtid, &first_gtid, errmsg)) {   /* ★★ GTID：由集合反算文件名 */
      set_fatal_error(errmsg.c_str());
      return 1;
    }
```

> ★★★ **这就是全部差别**：file+pos 依赖从库**记住并回传主库文件名**；GTID 模式下从库根本不传文件名，主库调 `find_first_log_not_in_gtid_set()` **由 exclude 集合反算出该从哪个文件开始**。

**★ 从库侧的反证：GTID 分支不传文件名**（`request_dump()`，`sql/rpl_replica.cc:4189`）：

```cpp
  if (command == COM_BINLOG_DUMP_GTID) {
    ...
    rpl->file_name = nullptr;        /* ★★ No need to set rpl.file_name_length */
    rpl->start_position = 4;         /* 只给 binlog 魔数头后的最小合法值 */
    rpl->flags |= MYSQL_RPL_GTID;
    rpl->gtid_set_encoded_size = gtid_executed.get_encoded_length();
    rpl->fix_gtid_set = fix_gtid_set;
    rpl->gtid_set_arg = (void *)&gtid_executed;
  } else {
    rpl->file_name_length = 0;
    rpl->file_name = mi->get_master_log_name();   /* ★ file+pos：必须记住主库文件名 */
    rpl->start_position = mi->get_master_log_pos();
  }
```

**★ 而"exclude 集合"是什么**（`sql/rpl_replica.cc:4193`，两步 union）：

```cpp
    if (gtid_executed.add_gtid_set(mi->rli->get_gtid_set()) != RETURN_STATUS_OK) ...
    /* ① 本通道已取回的中继日志 GTID */
    if (gtid_executed.add_gtid_set(gtid_state->get_executed_gtids()) != RETURN_STATUS_OK) ...
    /* ② 本实例已执行的 GTID（全局） */
```

> ★★ 第二步 union **全局** `executed_gtids`，正是**多源复制 / 级联复制下能把整条链上所有 UUID 都带上**的关键。

**★ 官方对 GTID 模式的一句话定义**（`sql/rpl_source.h:118`）：

```cpp
  @note This function will start reading at the given (filename, offset), or
  from the oldest log if filename[0]==0.  It will send all events from that
  position; but if gtid_set!=NULL, it will skip all events in that set.
```

> ★ "**skip all events in that set**" —— 从库不告诉主库"从哪开始"，而是说"**哪些我已经有了，你跳过**"。

**★★ 模式判定点不在 `init()`，在构造函数**（`sql/rpl_binlog_sender.cc:242`）：

```cpp
      m_exclude_gtid(exclude_gtids),
      m_using_gtid_protocol(exclude_gtids != nullptr),      /* ★★ 唯一判定 */
      m_check_previous_gtid_event(exclude_gtids != nullptr),
      m_gtid_clear_fd_created_flag(exclude_gtids == nullptr),
```

> ★ 上游 `com_binlog_dump()` 传 `nullptr`、`com_binlog_dump_gtid()` 传 `&slave_gtid_executed`。**没有任何配置变量参与**——模式完全由"从库发了哪条命令"决定。
> ⚠️ `init()` 里对 GTID 只做一件事：校验主库 `gtid_mode == ON`（`:284`），否则直接 fatal（这就是"从库开了 AUTO_POSITION 但主库 GTID_MODE 不是 ON 时 dump 线程起不来"的出处）。

**★ GTID 模式独有的两道安全检查**（file+pos 完全不存在）：

```cpp
    if (!m_exclude_gtid->is_subset_for_sid(&gtid_executed_and_owned,
                                           gtid_state->get_server_sidno(),
                                           subset_sidno)) {
      set_fatal_error(ER_THD(m_thd, ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE));
      return 1;                                   /* ① 从库不能比主库拥有更多同 UUID 的 GTID */
    }
    if (!gtid_state->get_lost_gtids()->is_subset(m_exclude_gtid)) {
      mysql_bin_log.report_missing_purged_gtids(m_exclude_gtid, errmsg);
      set_fatal_error(errmsg.c_str());
      return 1;                                   /* ② 从库必须已包含主库已 purge 的所有 GTID */
    }
```

> ★ 注意 `m_exclude_gtid` 的语义是 **exclude（排除集）**，不是 "start point"——这个命名本身就说清了机制。⚠️ 不存在 `m_gtid_set` 这个成员名。

#### 3. log_replica_updates 默认翻转：级联从"手动开启"到"默认可用"

`log_replica_updates` 控制从库是否把**回放的事务再写进自己的 binlog**。5.7 及以前默认 OFF，8.0 翻转为默认 ON。

- OFF 时：从库不写 binlog，**无法再被下一级从库复制**（级联链断裂）。这是 5.7 默认值——因为从库写 binlog 有开销，官方默认"不写"，需要级联的人自己开。
- ON 时：从库同时是下一级的"主库"，级联默认可用。8.0 翻转的理由：读写分离/多级分发拓扑已是常态，默认 OFF 的"省开销"远小于"级联链默认可工作"的价值；且 GTID 模式（多数部署）下回放事务写 binlog 的语义明确（GTID 不重新分配，原样传播）。

它也是**环拓扑的开关**：双主/环拓扑必须双写 binlog 才能让事务流通（见拓扑形态节）。

**★ 变量定义**（`sql/sys_vars.cc:6153`，★ `DEFAULT(1)` + `READ_ONLY`）：

```cpp
static Sys_var_bool Sys_log_replica_updates(
    "log_replica_updates",
    "If enabled, the replication applier threads will write to this server's "
    "binary log.",
    READ_ONLY GLOBAL_VAR(opt_log_replica_updates),
    CMD_LINE(OPT_ARG, OPT_LOG_REPLICA_UPDATES), DEFAULT(1));

static Sys_var_deprecated_alias Sys_log_slave_updates("log_slave_updates",
                                                      Sys_log_replica_updates);
```
> ⚠️ 真名核查：全局变量是 **`opt_log_replica_updates`**（不存在 `opt_log_slave_updates`）；`log_slave_updates` 只是 `Sys_var_deprecated_alias`。

**★ 一个容易踩的行为：没开 binlog 时它会隐式回退为 OFF**（`sql/mysqld.cc:4940`）：

```cpp
  if (!opt_bin_log) {
    /*
      The log-replica-updates should be disabled if binary log is disabled
      and --log-replica-updates option is not set explicitly on command
      line or configuration file.
    */
    if (!log_replica_updates_supplied) opt_log_replica_updates = false;
  }
```
> ★ 所以 `SHOW VARIABLES LIKE 'log_replica_updates'` 在 `log_bin=OFF` 的机器上会显示 **OFF**，即使你没显式设过。

**★★★ 关闭时级联链为什么会"断"（`sql/rpl_gtid_state.cc:885`，★ 这是最硬的一条证据）**：

```cpp
    if (thd->slave_thread && opt_bin_log && !opt_log_replica_updates) {
      lost_gtids._add_gtid(thd->owned_gtid);
      gtids_only_in_table._add_gtid(thd->owned_gtid);
    }
```

> ★★★ applier 回放的事务在 `log_replica_updates=OFF` 时，其 GTID **不进 `gtid_executed`，而是进 `lost_gtids`**——意思是"这个事务我执行过，但**无法向别人证明**"。下一级从库来复制时拿不到这些 GTID，级联链实质断裂。

**★ 它影响的代码路径（4 处）**：

```cpp
/* ① sql/binlog.cc:1833 —— 提交时决定要不要写 binlog */
    if (!opt_bin_log || (thd->slave_thread && !opt_log_replica_updates)) {
      /*
        If the binary log is disabled for this thread (either by
        log_bin=0 or sql_log_bin=0 or by log_replica_updates=0 for a
```
```cpp
/* ② sql/sql_class.cc:2981 —— 统一判定入口 */
bool THD::is_current_stmt_binlog_log_replica_updates_disabled() const {
  return ((!opt_bin_log || (slave_thread && !opt_log_replica_updates)) ||
          !mysql_bin_log.is_open());
}
```
```cpp
/* ③ sql/rpl_replica.cc:3993 —— applier 线程的 OPTION_BIN_LOG */
  ulonglong options = thd->variables.option_bits | OPTION_BIG_SELECTS;
  if (opt_log_replica_updates)
    options |= OPTION_BIN_LOG;
  else
    options &= ~OPTION_BIN_LOG;
```
> ★ 第二处说明它等价于"**给 applier 线程关掉 `sql_log_bin`**"——这解释了为什么关掉后从库的 binlog 会缺事务。

#### 4. 一致性 = 确认机制 × 拓扑 的二维选择

复制的"一致性"不是单一开关，是两个维度的组合：

| 确认机制 | 丢失窗口 | 性能代价 | 实现 |
|---------|---------|---------|------|
| 异步（默认） | 主库提交后到从库收到前的所有事务 | 零 | 纯拉模型 |
| 半同步 after_commit | 缩小到"已收到未回放" | 主库 commit 等 ACK | 插件，见 binlog.md |
| 半同步 after_sync | 理论上零（已落 relay log） | 同上 + sync 延迟 | 插件，见 binlog.md |
| 组复制（MGR） | 零（多数派确认后提交） | Paxos 往返 | 插件 |

注意确认机制解决的是"**单次切换丢不丢**"，拓扑解决的是"**切到谁**"。二者正交：异步复制 + 精心挑选的从库切换，也可以做到 RPO 很小（但需要人工确认从库追平）；半同步 + 乱切换也会丢（切到一个还没收到事务的从库）。这是故障转移节要展开的。

**★★ "确认点在什么时候"的源码答案：同一个等待函数挂在两个不同的事务阶段钩子上**（`plugin/semisync/semisync_source_plugin.cc:95`）：

```cpp
static int repl_semi_report_binlog_sync(Binlog_storage_param *,
                                        const char *log_file,
                                        my_off_t log_pos) {
  if (rpl_semi_sync_source_wait_point == WAIT_AFTER_SYNC)
    return repl_semisync->commitTrx(log_file, log_pos);      /* ★ after_sync 钩子 */
  return 0;
}

static int repl_semi_report_commit(Trans_param *param) {
  bool is_real_trans = param->flags & TRANS_IS_REAL_TRANS;
  if (rpl_semi_sync_source_wait_point == WAIT_AFTER_COMMIT && is_real_trans &&
      param->log_pos) {
    return repl_semisync->commitTrx(param->log_file, param->log_pos);  /* ★ after_commit 钩子 */
  }
  return 0;
}
```

**★ 官方对两个等待点的权威语义**（`semisync_source_plugin.cc:302`，★ 默认值是 `WAIT_AFTER_SYNC`）：

> - **AFTER_SYNC**（默认）：*"waits for the replies just after it has synced the binary log file ... but before it has committed in the engine on the source side. Therefore, **it guarantees that no other sessions on the source can see the effects of the transaction before the replica has received it**"* → **无损**
> - **AFTER_COMMIT**：*"waits for the replies ... just after the source has committed the transaction in the engine, and before it sends an ACK packet to the client session. **Other sessions may see the effects of the transaction before it has been replicated**"*

**★★ 等待判据是按 binlog 位点，不是"已执行"**（`semisync_source.cc:687`）：

```cpp
    while (is_on()) {
      if (reply_file_name_inited_) {
        int cmp = ActiveTranx::compare(reply_file_name_, reply_file_pos_,
                                       trx_wait_binlog_name, trx_wait_binlog_pos);
        if (cmp >= 0) break;    /* ★ 从库已 ack 的 binlog 位点 >= 本事务位点 → 放行 */
      }
```

> ★★ ack 的是"**收到 binlog**"，不是"已应用"。这是半同步只能保证"不丢 binlog"、不保证"从库已回放"的源码依据。
> ★ 等待阶段名 `"Waiting for semi-sync ACK from replica"` 出自 `THD_ENTER_COND(..., &stage_waiting_for_semi_sync_ack_from_replica, ...)`（`semisync_source.cc:660`）——就是 `SHOW PROCESSLIST` 里那一行。

**★ N 副本确认（拓扑那一维）**（`semisync_source.h:808`）：

```cpp
  void handleAck(int server_id, const char *log_file_name, my_off_t log_file_pos) {
    lock();
    if (rpl_semi_sync_source_wait_for_replica_count == 1)
      reportReplyBinlog(log_file_name, log_file_pos);       /* 任一从库 ack 即可 */
    else {
      const AckInfo *ackinfo = nullptr;
      ackinfo = ack_container_.insert(server_id, log_file_name, log_file_pos);
      if (ackinfo != nullptr)
        reportReplyBinlog(ackinfo->binlog_name, ackinfo->binlog_pos);  /* 等第 N 个不同 server_id */
    }
    unlock();
  }
```

> ★ `ack_container_` **按 server_id 去重**——同一个从库重复 ack 不计数。

**★ 半同步不改复制协议**（`semisync_source_plugin.cc:185`）——它挂在 `Binlog_transmit` 的三个 observer 钩子上偷塞 2 字节头：

```cpp
static int repl_semi_before_send_event(Binlog_transmit_param *param,
                                       unsigned char *packet, unsigned long,
                                       const char *log_file, my_off_t log_pos) {
  if (!is_semi_sync_dump()) return 0;
  return repl_semisync->updateSyncHeader(packet, log_file, log_pos, param->server_id);
}
```

> ★★ 这解释了**为什么半同步对 GTID / file+pos 两种模式透明**——它与定位机制完全解耦。这正是"一致性 = 确认机制 × 拓扑"是**二维而非一维**的源码依据。
> ⚠️ 半同步源码在 **`plugin/semisync/`**，不在 `sql/rpl_semi_sync*`。变量名真名 `rpl_semi_sync_source_wait_point`（不存在 `rpl_semi_sync_master_wait_point`），但**类名仍叫 `ReplSemiSyncMaster`**、方法仍叫 `readSlaveReply` / `commitTrx`。

#### 5. 过滤器为什么只在从库侧

复制过滤（`replicate_do_db` 等）在**从库应用时**过滤，而不是主库发送时过滤。原因：主库对"哪个从库要什么"零状态（拉模型推论）——主库不知道也不应该知道。代价是**带宽浪费**（不需要的事件也传了过来），换来主库无状态和过滤策略可随时在从库侧调整。中间件（如级联节点）可以在主库侧做"真过滤"，但那是另一层。

### 理论溯源

**auto position 算法的本质**是"在有序文件序列上做集合包含定位"：每个 binlog 文件头部有 `Previous_gtids_log_event`（该文件之前所有事务的 GTID 并集），这些集合随文件顺序**单调增长**。要找"从库还没执行过的最老事务"，只需**从最新文件逆序扫描**，找第一个 `Previous_gtids ⊆ 从库 exclude 集合` 的文件——它之前的事务从库全有，它开始的事务从库需要。单调性保证第一次命中即可停止，无需全量扫描。这是典型的"利用前缀和单调性做定位"思路（与 LSN 物理定位同构：`Previous_gtids` 扮演了"文件头水位"的角色）。

**事件流协议**继承自 MySQL 客户端/服务器协议框架：dump 是一个"永不结束的查询"，`BINLOG_DUMP` 命令复用普通命令通道（`COM_*` 命令族），返回的是按 `net` 协议封装的 event 流而非结果集。协议文档就在 `rpl_source.cc` 头部的 doxygen 注释里（`@page page_protocol_com_binlog_dump` 等）。

### 他库对比与演进动机

| 系统 | 复制单位 | 位点 | failover 方式 | 关键差异 |
|------|---------|------|--------------|---------|
| MySQL（GTID） | 逻辑事务 | GTID 集合差集 | 自动收敛（auto position） | 逻辑复制，从库可异构（不同版本/引擎） |
| MySQL（file+pos） | 逻辑事务 | (文件, 偏移) | 手工换算 | 拓扑绑定，GTID 前时代 |
| PostgreSQL 流复制 | 物理 WAL 流（LSN） | LSN + replication slot | 新主库 LSN 对齐 | 物理复制：从库必须同版本同平台；slot 让**主库**记住从库进度（推模型的"位点归主库"变体，slot 丢失=主库无法安全回收 WAL） |
| Oracle Data Guard | redo（物理/逻辑两模式） | SCN | Broker 自动切换 | 商业一体化方案，切换编排内置 |
| MongoDB | oplog（逻辑） | 时间戳/任期 | 选举（Raft 协议层） | 复制与选举合并在协议层，MySQL 的 MGR 与之对齐 |

演进动机（对应版本演进表）：5.6 引入 GTID 是为了去掉 file+pos 的拓扑绑定（failover 自动化）；8.0 术语重命名（master/slave → source/replica）是官方在 2020 年的包容性语言工程，同时把 GTID_ONLY 需要的概念空间铺平；`log_replica_updates` 默认翻转是"级联拓扑成为常态"的默认值跟进。

---

## 核心实现

### 主链路：一次复制建立的完整流程

```
① 准备数据
   └─ 从库加载主库的物理/逻辑备份，使其处于主库某个历史时点

② 配置源信息
   └─ CHANGE REPLICATION SOURCE TO SOURCE_HOST=... SOURCE_AUTO_POSITION=1
      （参数写入 mysql.slave_master_info 位点仓库）

③ 启动复制
   └─ START REPLICA
      ├─ IO thread: 连主库 → 注册(COM_REGISTER_SLAVE) → 请求 dump
      │             → 收事件流 → 写 relay log → 推进 Retrieved 位点
      └─ SQL thread: 读 relay log → 应用事件 → 推进 Executed 位点
      （线程内部细节见 replica.md）

④ 稳态
   └─ 主库每提交一个事务，事件经 binlog → 网络 → relay log → 引擎 三级接力
```

三个线程的分工与文档映射：

| 线程 | 跑在哪 | 干什么 | 详解 |
|------|--------|--------|------|
| dump 线程（`Binlog_sender`） | 主库，每个从库一条 | 按请求读 binlog，封装事件流下发 | binlog.md「读侧 dump 线程」 |
| IO thread（`handle_slave_io`） | 从库 | 连主库、收事件、写 relay log | replica.md「IO thread 接收管线」 |
| SQL thread / coordinator + workers | 从库 | 读 relay log、回放、推进位点 | replica.md「SQL thread 应用管线」+ prpl.md |

### 复制协议与握手

> 分工：本篇讲**交互契约**——每一步双方交换什么；从库侧各步骤的代码实现见 [`replica.md`](replica.md)「IO thread 接收管线」，主库侧发送细节见 [`binlog.md`](binlog.md)「读侧 dump 线程」。

#### 完整握手时序

从库 IO thread 连上主库后，在发出第一个 dump 请求之前要走完一串准备动作（顺序即 `handle_slave_io` 主循环的前几步）：

```
从库 IO thread                                     主库
────────────────────────────────────────────────────────────
① connect_to_master
   TCP + 认证 + SSL(可选) + 设 net timeout
   ─────────────────────────────────────────────►
                                    （认证、版本握手由连接层完成）
② get_master_version_and_clock
   · 读 mysql->server_version  ← 版本<5 或不识别则拒绝
   · 建本地 FD event（relay log 格式基线）
   · SELECT @@GLOBAL.binlog_checksum  ─────────►  返回 checksum 算法
     （存入 mi->checksum_alg_before_fd，用于 FD 到达前的"预热校验"）
   · SELECT UNIX_TIMESTAMP()  ─────────────────►  返回主库时钟
     （算 clock_diff，供 Seconds_Behind_Source 使用）
③ get_master_uuid
   SELECT @@GLOBAL.SERVER_UUID  ───────────────►  返回主库 UUID
   ★ 与本机 UUID 相同 → 报错停止复制
④ io_thread_init_commands
   SET @slave_uuid/@replica_uuid = '<从库 UUID>'  ──►  主库记住本连接属于谁
⑤ register_slave_on_master（COM_REGISTER_SLAVE）
   上报 server_id + report_host/user/password/port ──►  主库 slave_list
                                                      （供 SHOW REPLICAS）
⑥ request_dump（COM_BINLOG_DUMP_GTID 或 COM_BINLOG_DUMP）
   ─────────────────────────────────────────────►
                                    主库：kill_zombie_dump_threads
                                          （按 ④ 的 UUID + ⑤ 的 server_id）
                                          三道校验 → 定位起点
   ◄───────────────────────────────  fake Rotate + FD + 事件流 + 心跳
⑦ read_event / queue_event 循环（收包 → 写 relay log → 推进位点）
```

#### 双方各需要对方什么信息

**从库需要从主库获取**（② ③）：

| 信息 | 获取方式 | 用途 |
|------|---------|------|
| 版本 | 连接层 `server_version` | 兼容性检查（<5 直接拒绝） |
| `binlog_checksum` 算法 | `SELECT @@GLOBAL.binlog_checksum` | FD event 到达前的校验"预热"；之后改用 FD 里带的算法 |
| 主库时钟 | `SELECT UNIX_TIMESTAMP()` | 算 `clock_diff`，`Seconds_Behind_Source` 依赖它（拿不到也不致命） |
| **主库 `server_uuid`** | `SELECT @@GLOBAL.SERVER_UUID` | ① 与本机 UUID 相同则停止复制（GTID 命名空间必须不同）；② 记入 `SHOW REPLICA STATUS` 的 `Source_UUID` |
| FD event / 事件格式 | 主库主动下发 | relay log 的格式基线 |

★ 注意：**GTID 模式下从库不需要知道主库的任何 binlog 文件名或位点**——这正是 auto position 的意义（主库自己算起点）。只有 file+pos 模式才需要维护 `Source_Log_File/Pos`。

**主库需要从库提供**（④ ⑤ ⑥）：

| 信息 | 提供方式 | 用途 |
|------|---------|------|
| 从库 `server_uuid` | `SET @slave_uuid/@replica_uuid`（用户变量，非 COM 命令） | **僵尸连接管理**：同 UUID 的旧 dump 线程被踢除（`kill_zombie_dump_threads` 用 `get_replica_uuid`） |
| 从库 `server_id` | COM_REGISTER_SLAVE / dump 报文 | 同上（server_id 维度）；也用于从库端"跳过自己发出的事件"（见 `replicate_same_server_id`） |
| exclude 集合（GTID 模式） | COM_BINLOG_DUMP_GTID 报文 | 决定跳过哪些、从哪里开始（见 gtid.md） |
| 文件名 + 偏移（file+pos） | COM_BINLOG_DUMP 报文 | 直接作起点 |
| report_host/user/password/port | COM_REGISTER_SLAVE | 仅用于 `SHOW REPLICAS` 展示（不参与协议逻辑） |

**系统层结论**：整个握手里主库真正"记住"的从库状态只有 ④ ⑤ 的注册信息（且只用于防僵尸和展示）——**没有任何与复制进度有关的状态**。这是拉模型的核心收益：主库挂掉后从库连到新主库，新主库不需要任何"旧主库留下的关于这个从库的信息"，仅凭从库自己带来的 exclude 集合就能继续。

**★★ `MYSQL_RPL` 结构体：从库请求的完整载体**（`include/mysql.h:358`，⚠️ **不在 `mysql_com.h`**）：

```cpp
/**
  Flag to indicate that COM_BINLOG_DUMP_GTID should
  be used rather than COM_BINLOG_DUMP in the @sa mysql_binlog_open().
*/
#define MYSQL_RPL_GTID (1 << 16)
/**
  Skip HEARBEAT events in the @sa mysql_binlog_fetch().
*/
#define MYSQL_RPL_SKIP_HEARTBEAT (1 << 17)

/**
 Flag to indicate that the heartbeat_event being generated
 is using the class Heartbeat_event_v2
*/
#define USE_HEARTBEAT_EVENT_V2 (1 << 1)   /* ★ 不是 MYSQL_RPL_* 家族，是主库侧用的 */

typedef struct MYSQL_RPL {
  size_t file_name_length; /** Length of the 'file_name' or 0     */
  const char *file_name;   /** Filename of the binary log to read */
  uint64_t start_position; /** Position in the binary log to      */
                           /*  start reading from                 */
  unsigned int server_id;  /** Server ID to use when identifying  */
                           /*  with the master                    */
  unsigned int flags;      /** Flags, e.g. MYSQL_RPL_GTID         */

  /** Size of gtid set data              */
  size_t gtid_set_encoded_size;
  /** Callback function which is called  */
  /*  from @sa mysql_binlog_open() to    */
  /*  fill command packet gtid set       */
  void (*fix_gtid_set)(struct MYSQL_RPL *rpl, unsigned char *packet_gtid_set);
  void *gtid_set_arg; /** GTID set data or an argument for   */
                      /*  fix_gtid_set() callback function   */

  unsigned long size;          /** Size of the packet returned by     */
  const unsigned char *buffer; /** Pointer to returned data           */
} MYSQL_RPL;
```

> ⚠️ **勘误**：`MYSQL_RPL_*` 标志**只有 2 个**（`MYSQL_RPL_GTID` / `MYSQL_RPL_SKIP_HEARTBEAT`），没有常见资料里说的 `NON_BLOCK` 等。且 `flags` 是 `unsigned int`——GTID 标志在 **bit 16**，不是低 8 位。
> ★ `fix_gtid_set` 是**回调**而非直接拷贝——避免在 `request_dump()` 里先序列化一整份 Gtid_set，写包时才现算。

**★ 主库侧记录的"从库请求"全量成员**（`sql/rpl_binlog_sender.h:82`）：

```cpp
  THD *m_thd;
  String &m_packet;

  /* Requested start binlog file and position */
  const char *m_start_file;
  my_off_t m_start_pos;

  /*
    For COM_BINLOG_DUMP_GTID, It may include a GTID set. All events in the set
    should not be sent to the client.
  */
  Gtid_set *m_exclude_gtid;
  bool m_using_gtid_protocol;
  bool m_check_previous_gtid_event;
  bool m_gtid_clear_fd_created_flag;

  LOG_INFO m_linfo;                             /* The binlog file it is reading */

  binary_log::enum_binlog_checksum_alg m_event_checksum_alg;
  binary_log::enum_binlog_checksum_alg m_slave_checksum_alg;   /* ★ 从库上报的校验和能力 */
  std::chrono::nanoseconds m_heartbeat_period;                  /* ★ 由从库 heartbeat_period 换算 */
  std::chrono::nanoseconds m_last_event_sent_ts;
  /*
    For mysqlbinlog(server_id is 0), it will stop immediately without waiting
    if it already reads all events.
  */
  bool m_wait_new_events;
  ...
  uint32 m_flag;          /* ★ 从库传来的 binlog_flags；低 3 位为 5.6 历史兼容保留 */
  bool m_observe_transmission;
  bool m_transmit_started;
  binary_log::Log_event_type m_prev_event_type;
```

**★ 主库 → 从库的事件流**（`sql/rpl_binlog_sender.cc`，★ 关键在 heartbeat）：

```cpp
int Binlog_sender::send_binlog(File_reader &reader, my_off_t start_pos) {
  if (unlikely(send_format_description_event(reader, start_pos))) return 1;
```
```cpp
int Binlog_sender::send_heartbeat_event(my_off_t log_pos) {
  uint32 hb_version_flag = m_flag & USE_HEARTBEAT_EVENT_V2;
  DBUG_EXECUTE_IF("use_old_heartbeat_version", { hb_version_flag = 0; });
  return (hb_version_flag ? send_heartbeat_event_v2(log_pos)
                          : send_heartbeat_event_v1(log_pos));
}
```
> ★★ heartbeat 的一个非直觉用途：**GTID 事件被跳过时用 heartbeat 替代回送位点**（`rpl_binlog_sender.cc` 有 4 处，行 632/651/677/825）。这是"被过滤掉的 GTID 也要推进位点"的实现——否则从库会以为卡住了。

**★ server_id 在事件头里的位置**（`libbinlogevents/include/binlog_event.h:410`）：

```cpp
#define EVENT_TYPE_OFFSET 4
#define SERVER_ID_OFFSET 5      /* ★★ 4 字节，防循环判定就 uint4korr(buf + SERVER_ID_OFFSET) */
#define EVENT_LEN_OFFSET 9
#define LOG_POS_OFFSET 13
...
#define LOG_EVENT_HEADER_LEN 19U /* the fixed header length */
```

**★ 从库自报身份的 4 个变量**（`sql/sys_vars.cc:5818`，★ `report_host` 默认 NULL = **不注册**）：

```cpp
static Sys_var_charptr Sys_repl_report_host(
    "report_host",
    "Hostname or IP that this replica will report to the source while "
    "initiating the replication connection. Will appear in the output of "
    "SHOW REPLICAS. Leave this unset if you do not want the replica to "
    "register itself with the source. Note that it is not sufficient for "
    "the source to simply read the IP of the replica off the socket once the "
    "replica connects: in the presence of NAT other routing features, that IP "
    "may not be valid for connecting to the replica from the source or other "
    "hosts.",
    READ_ONLY GLOBAL_VAR(report_host), CMD_LINE(REQUIRED_ARG), IN_FS_CHARSET,
    DEFAULT(nullptr));
```
> ★ 注释解释了为什么不能自动推断 IP：**NAT / 路由场景下 socket 对端 IP 不可用**。
> ★ 这 4 个变量全是 `READ_ONLY` —— 改了必须重启。

#### file+pos 模式的握手一样吗

**前五步完全相同**——源码里 `get_master_version_and_clock` → `get_master_uuid` → `io_thread_init_commands` → `register_slave_on_master` 是**无条件顺序调用**的，与定位模式无关（版本检查、UUID 唯一性校验、`@slave_uuid` 上报、注册这些在 file+pos 下一样要做）。事件流、心跳、`queue_event` 写 relay log 也完全一样。

**差异全部集中在第 ⑥ 步 `request_dump` 之后**：

| 维度 | file+pos（`COM_BINLOG_DUMP`） | GTID（`COM_BINLOG_DUMP_GTID`） |
|------|------------------------------|-------------------------------|
| 报文 | 文件名 + `pos(4)` | 空文件名 + `pos(8)`=4 + exclude 集合 |
| 起点谁定 | **从库指定**，主库直接用（`check_start_file` 分支①） | **主库算**（`find_first_log_not_in_gtid_set`） |
| 主库校验 | 只校验文件存在 + `pos` 合法（≥4、≤文件长度） | 三道集合校验（子集/purge/定位） |
| 事件级跳过 | **无**——`m_exclude_gtid == nullptr`，从 pos 起全发 | **有**——`skip_event` 逐事务 `contains_gtid` 过滤 |
| 从库要维护 | io 位点（文件名 + 偏移） | 只需 `gtid_executed` |
| 重连粒度 | 从持久化的 io 位点续；**若断在事务中间**，靠 `Transaction_boundary_parser` 丢弃半截事务 | 靠集合语义，天然不含半截事务（GTID 只在事务完整 flush 后入 retrieved） |

> 一句话：**握手骨架共用，差异只在"怎么表达起点"** ——file+pos 传物理坐标、主库照做；GTID 传集合、主库推导。这也解释了为什么 file+pos 必须做 io 位点的 crash-safe 持久化而 GTID 不必。

**★★★ 两种报文的字节级对照**（`sql-common/client.cc:7196`，★ 同一函数 `mysql_binlog_open()` 的两个分支）：

```cpp
  if (rpl->flags & MYSQL_RPL_GTID) {
    command = COM_BINLOG_DUMP_GTID;
    ...
    int2store(ptr, rpl->flags);                                   // flags(2)
    ptr += ::BINLOG_FLAGS_INFO_SIZE;
    int4store(ptr, rpl->server_id);                               // server_id(4)
    ptr += ::BINLOG_SERVER_ID_INFO_SIZE;
    int4store(ptr, static_cast<uint32>(rpl->file_name_length));   // name_len(4)
    ptr += ::BINLOG_NAME_SIZE_INFO_SIZE;
    memcpy(ptr, rpl->file_name, rpl->file_name_length);           // name(n)
    ptr += rpl->file_name_length;
    int8store(ptr, rpl->start_position);                          // pos(8)  ★★
    ptr += ::BINLOG_POS_INFO_SIZE;
    int4store(ptr, static_cast<uint32>(rpl->gtid_set_encoded_size));  // data_len(4)
    ptr += ::BINLOG_DATA_SIZE_INFO_SIZE;
    if (rpl->fix_gtid_set) rpl->fix_gtid_set(rpl, ptr);
    else memcpy(ptr, rpl->gtid_set_arg, rpl->gtid_set_encoded_size);  // gtid_set(n)
    ptr += rpl->gtid_set_encoded_size;
  } else {
    command = COM_BINLOG_DUMP;
    ...
    /*
      COM_BINLOG_DUMP accepts only 4 bytes for the position, so
      we are forced to cast to uint32.                             ★★
    */
    int4store(ptr, (uint32)rpl->start_position);                  // pos(4)  ★★
    ptr += ::BINLOG_POS_OLD_INFO_SIZE;
    int2store(ptr, rpl->flags);                                   // flags(2)
    ptr += ::BINLOG_FLAGS_INFO_SIZE;
    int4store(ptr, rpl->server_id);                               // server_id(4)
    ptr += ::BINLOG_SERVER_ID_INFO_SIZE;
    memcpy(ptr, rpl->file_name, rpl->file_name_length);           // name(n)  ★ 无长度前缀
    ptr += rpl->file_name_length;
  }
```

> ★★ **注意字段顺序都不一样**（旧协议 pos 在最前），而且：
> - **GTID 分支每个字段都 length-prefixed**；**file+pos 分支的文件名没有长度前缀**，靠"包尾"推断——这是旧协议最脆弱的地方。
> - **pos 宽度不同**：`COM_BINLOG_DUMP` 只有 **4 字节**（`BINLOG_POS_OLD_INFO_SIZE`），`COM_BINLOG_DUMP_GTID` 是 **8 字节**。

**★ 源码自己承认的旧协议缺陷**（`sql/rpl_source.cc:957`）：

```cpp
  /*
    4 bytes is too little, but changing the protocol would break
    compatibility.  This has been fixed in the new protocol. @see
    com_binlog_dump_gtid().
  */
  READ_INT(pos, 4);
  READ_INT(flags, 2);
  READ_INT(thd->server_id, 4);
  ...
  mysql_binlog_send(thd, thd->mem_strdup(packet + 10), (my_off_t)pos, nullptr, flags);
  /*                                    ★ packet+10 = 文件名起点（4+2+4）        ★ nullptr = file+pos 模式的标志 */
```

**★ GTID 集合的线上格式**（`sql/rpl_gtid_set.cc:1237`，`Gtid_set::encode()`）：

```
n_sids(8) [ UUID(16) n_intervals(8) [start(8) end(8)]* ]*
```

> ★ 区间是**半开区间 `[start, end)`**；`sid_map->get_sorted_sidno()` **按 UUID 字典序排序**输出 —— 保证同一集合的编码唯一（canonical form），这是 GTID set 能逐字节比较 / 去重 / 做集合运算的前提。
> ★ 主库侧解码是 `Gtid_set::add_gtid_encoding()`（`rpl_gtid_set.cc:1285`），⚠️ **不存在 `read_gtids_from_packet()`**。它把从库发来的任意 UUID 动态注册进**本连接私有的 sid_map**——`sidno` 是纯本地整数编号，**与拓扑位置完全无关**。这是"拓扑无关"的第二条硬证据。

**★ 两条命令的唯一汇合点**（`sql/rpl_source.cc:1054`，★ 只有 5 行）：

```cpp
void mysql_binlog_send(THD *thd, char *log_ident, my_off_t pos,
                       Gtid_set *slave_gtid_executed, uint32 flags) {
  Binlog_sender sender(thd, log_ident, pos, slave_gtid_executed, flags);
  sender.run();
}
```

> ⚠️ `mysql_binlog_send()` **不在 `sql/binlog.cc`**（grep 0 命中），真身在 `sql/rpl_source.cc:1054`。

#### 三个 COM 命令

| 命令 | 载荷 | 语义 |
|------|------|------|
| `COM_REGISTER_SLAVE` | server_id + 主机名/端口/用户 | 登录后注册身份，主库用于 `SHOW REPLICAS` 与僵尸连接管理（非必需） |
| `COM_BINLOG_DUMP`（0x12） | binlog 文件名 + 偏移 + server_id | file+pos 定位：从指定字节开始发事件 |
| `COM_BINLOG_DUMP_GTID` | 编码后的 GTID 集合 + server_id | GTID 定位：发"集合之外"的事务 |

**★ 枚举定义**（`include/my_command.h:36`，★ 数值位置就是全部协议兼容性）：

```cpp
enum enum_server_command {
  ...
  COM_BINLOG_DUMP,        /* 0x12 —— file+pos */
  COM_TABLE_DUMP,
  COM_CONNECT_OUT,
  COM_REGISTER_SLAVE,     /* 0x15 —— 注册（可选） */
  ...
  COM_DAEMON,
  COM_BINLOG_DUMP_GTID,   /* 0x1E —— GTID */
  ...
  /* Must be last */
  COM_END
};
```

> ★ 注释明确要求"**Add new commands to the end of this list, otherwise old servers won't be able to handle them as 'unsupported'**"——这就是"从库版本比主库新"时只能靠 flags 扩展、不能靠新命令号的原因。
> ★★ **`COM_REGISTER_SLAVE` 在 8.0.39 仍然存在且仍在线上发送**（`rpl_replica.cc:3303`）。唯一"改名"的是**用户可见的报错文本**写成 `"COM_REGISTER_REPLICA"`（`rpl_replica.cc:3312`）。

**★ 分派点**（`sql/sql_parse.cc:2270`）：

```cpp
    case COM_BINLOG_DUMP_GTID:
      error = com_binlog_dump_gtid(thd, (char *)...get_raw_packet(), ...);
      break;
    case COM_BINLOG_DUMP:
      error = com_binlog_dump(thd, (char *)...get_raw_packet(), ...);
      break;
```

> ★ 两个 case 紧邻、结构完全对称。**整个 dump 请求是"一条命令、一个长连接、无响应包"**（`my_eof` 只在结束时发）——这就是拉模型在协议层的形态。
> ⚠️ 命令码叫 SLAVE，但处理函数已改名 `register_replica()`（`sql_parse.cc:1831` → 定义在 `rpl_source.cc:142`）——"协议层不动、源码层改名"是 8.0 术语改造的统一手法。

**★ 从库 → 主库的注册字段**（`sql/rpl_replica.cc:3285`）：

```cpp
  int4store(pos, server_id);                                     pos += 4;
  pos = net_store_data(pos, (uchar *)report_host, report_host_len);
  pos = net_store_data(pos, (uchar *)report_user, report_user_len);
  pos = net_store_data(pos, (uchar *)report_password, report_password_len);
  int2store(pos, (uint16)report_port);                           pos += 2;
  /*
    Fake rpl_recovery_rank, which was removed in BUG#13963,
    so that this server can register itself on old servers, see BUG#49259.
  */
  int4store(pos, /* rpl_recovery_rank */ 0);                     pos += 4;
  /* The master will fill in master_id */
  int4store(pos, 0);                                             pos += 4;
```

> ★ 那个已删除字段仍要占 4 字节——**协议向后兼容的经典实证**。主库侧对应地 `p += 4` 跳过（`rpl_source.cc:188`），并回填 `si->master_id = server_id`。

请求的构造（`request_dump` 按 `is_auto_position` 选命令、GTID 模式发 retrieved ∪ executed 并集作为 exclude 集合）见 [`replica.md`](replica.md)「IO thread 接收管线」；主库侧三件事的代码细节见 [`binlog.md`](binlog.md)「读侧 dump 线程」——僵尸连接踢除（`kill_zombie_dump_threads`）、心跳（HEARTBEAT_EVENT v2，空闲超时后主动发，从库据此刷新 `Read_Source_Log_Pos` 探测存活）、事件流主循环。

**系统层视角只需记住两点**：其一，拉模型下主库对从库仅有的状态就是"注册信息 + 防僵尸"，这使 failover 切换主库零负担；其二，心跳解决了拉模型"收不到事件无法区分主库安静 vs 主库挂了"的固有问题。

### 位点定位：file+pos vs GTID auto position

#### file+pos：显式坐标

从库记 `(master_log_name, master_log_pos)`，重连时原样请求。**拓扑绑定**（理论基础 2 节）：坐标只在原主库有效。failover 时必须换算——官方手册的传统流程是：先 `STOP REPLICA` 对齐旧主库末位点，再在新主库上找对应位置（`SOURCE_LOG_FILE`/`SOURCE_LOG_POS`）。任何换算偏差都导致漏事务或重复事务。

#### GTID auto position：差集定位

主库侧核心算法 `find_first_log_not_in_gtid_set`——**逆序扫描 binlog index，只读每个文件的 `Previous_gtids_log_event`，找第一个 `Previous_gtids ⊆ exclude 集合` 的文件**（该文件之前的事务从库全有，从它开始 dump）。算法代码、逐事件 `skip_event` 过滤与三道集合校验的完整剖析见 [`gtid.md`](gtid.md)「发送起点的选择」与「GTID 复制协议：exclude 集合」；本篇只讲它对 failover 的意义。

```cpp
bool MYSQL_BIN_LOG::find_first_log_not_in_gtid_set(char *binlog_file_name,
                                                   const Gtid_set *gtid_set, ...) {
  // filename_list: binlog index 里的文件名列表（有序）
  list<string>::reverse_iterator rit;
  Gtid_set binlog_previous_gtid_set{gtid_set->get_sid_map()};
  ...
  // 从最新文件逆序扫描，只读每个文件的 Previous_gtids_log_event
  rit = filename_list.rbegin();
  while (rit != filename_list.rend()) {
    binlog_previous_gtid_set.clear();
    const char *filename = rit->c_str();
    switch (read_gtids_from_binlog(filename, nullptr, &binlog_previous_gtid_set, ...)) {
      case GOT_GTIDS:
      case GOT_PREVIOUS_GTIDS:
        if (binlog_previous_gtid_set.is_subset(gtid_set)) {
          // 该文件之前的全部事务从库都已执行 → 从这里开始发
          strcpy(binlog_file_name, filename);
          goto end;
        }
      case TRUNCATED:
        break;
    }
    rit++;
  }
```

逐段解释：

1. **输入** `gtid_set` 是从库发来的 exclude 集合（"我有的"）；`filename_list` 是主库 binlog index 的全部文件名。
2. **逆序扫描**：从最新文件往回。每读一个文件的 `Previous_gtids_log_event`——它是"该文件之前所有事务"的并集快照，随文件顺序单调增长（理论溯源节）。
3. **判据** `is_subset(gtid_set)`：该文件之前的所有 GTID 都在从库 exclude 集合里，即"从库已执行过它之前的一切"。
4. **第一个命中即停**：单调性保证命中点之后（更旧）的文件也满足，但我们要的是**最老的那个满足文件**的下一份内容——逆序扫描第一个命中的文件，正是"从库还没执行的最老事务"所在文件。dump 从这里开始，把该文件里不在集合中的事务全部发出（事件级过滤：每个 `Gtid_log_event` 的 GTID 再逐条检查是否在 exclude 集合内）。
5. **错误分支**：扫完全部文件都不命中 → `ER_SOURCE_HAS_PURGED_REQUIRED_GTIDS`（主库 purge 掉了从库需要的事务，复制无解，见 gtid.md exclude 集合协议）；遇到无 GTID 的老 binlog → `ER_SOURCE_FATAL_ERROR_READING_BINLOG`（GTID 模式不能与无 GTID 文件混用）。

**为什么不需要记住起点**：从库重连时重新发 exclude 集合，主库每次重新定位。位点仓库里只有"我执行过什么"，没有"我从哪读的"——这正是 GTID_ONLY 模式把文件名从仓库里删掉的前提（见 GTID_ONLY 节）。

#### 两种模式的取舍

| | file+pos | GTID auto position |
|---|---|---|
| 位点语义 | 物理坐标 | 事务集合差集 |
| failover | 手工换算位点 | 自动收敛 |
| 位点仓库 | 文件名+偏移+GTID | 仅 GTID（GTID_ONLY 时） |
| 前提 | 无 | `gtid_mode=ON`（`SOURCE_AUTO_POSITION=1` 在 gtid_mode=OFF 时被拒，见 CHANGE REPLICATION SOURCE 校验） |
| 依赖 | 文件名在目标主库上存在 | exclude 集合合法（两个错误码约束） |

### 拓扑形态

#### 单主多从（基础形态）

```
            ┌─→ replica A
source ─────┼─→ replica B
            └─→ replica C
```

最简形态。三个从库互不相关，各自拉取。这是 failover 节讨论的前提拓扑。

#### 级联复制（chained）：log_replica_updates

```
source ──→ replica A ──→ replica B
            （中间节点）
```

A 同时是从库（对 source）和主库（对 B）。**前提是 A 开启 `log_replica_updates`**（8.0 默认 ON）——A 把回放的事务写进自己的 binlog，B 才能从 A 拉。GTID 模式下回放事务的 GTID **原样传播不重分配**，所以 B 的 `gtid_executed` 与 A 一致，failover 时 B 可以直接切到 source 而不需要任何 GTID 换算。

级联的价值：**把主库的 dump 负担分出去**。几十个从库直连主库时，主库要为每个从库开一条 dump 线程 + 逐份读 binlog；改为"一个中间节点 + 下面挂几十个"，主库只服务一个下游。代价是延迟叠加（每一级多一跳）。

#### 多源复制（multi-source）：CHANNEL

```
source A ─┐
          ├─→ replica X（两个 CHANNEL 各自独立复制）
source B ─┘
```

`CHANNEL` 是复制通道的隔离单元。每个 CHANNEL 有自己完整的 `Master_info` + `Relay_log_info`、自己的 IO/SQL 线程对、自己的 relay log 文件（`relay-log-<channel>.NNNNNN`）、自己的位点仓库行。语义上等价于"两套独立的复制跑在同一台服务器上"，通过 `FOR CHANNEL 'ch1'` 子句区分。

CHANNEL 的隔离意味着：A 的复制可以停（`STOP REPLICA FOR CHANNEL 'ch1'`）而 B 的照跑；A 卡住不影响 B。`replica_parallel_workers` 等参数可按 CHANNEL 覆盖（`CHANGE REPLICATION SOURCE ... FOR CHANNEL`）。多源的典型用途：数据聚合（两个业务库汇入一个分析库）+ 级联分发。

**★ `Multisource_info`（真名，不是 `channel_map` 类）**（`sql/rpl_msr.h:113`）：

```cpp
class Multisource_info {
 private:
  /* Maximum number of channels per slave */
  static const unsigned int MAX_CHANNELS = 256;      /* ★★ 硬上限 */
  replication_channel_map rep_channel_map;           /* ★ 按 channel type 分组的两层 map */
  ...
 public:
  static const char *group_replication_channel_names[];
```
```cpp
/* sql/rpl_msr.h:617 */
extern Multisource_info channel_map;                 /* ★ 全局单例（类内 assert 保证只实例化一次） */
```

**★★ 默认 channel 的名字是空串 `""`，不是 `"default"`**（`sql/rpl_msr.cc:37`）：

```cpp
const char *Multisource_info::default_channel = "";
const char *Multisource_info::group_replication_channel_names[] = {
    "group_replication_applier", "group_replication_recovery"};
```

> ★ 所以 `SHOW REPLICA STATUS` 的 `Channel_Name` 列对默认 channel **显示空串**。

**★ channel 名的两条规则**（`sql/sql_lex.cc:4981`）：

```cpp
bool LEX::set_channel_name(LEX_CSTRING name) {
  if (name.str == nullptr) {
    mi.channel = "";            /* ★ 未写 FOR CHANNEL → 默认 channel */
    mi.for_channel = false;
  } else {
    /*
      Channel names are case insensitive. This means, even the results
      displayed to the user are converted to lower cases.
    */
    char *buf = thd->strmake(name.str, name.length);
    my_casedn_str(system_charset_info, buf);   /* ★★ 立即转小写 */
    mi.channel = buf;
    mi.for_channel = true;
  }
```

> ★ 长度上限 `CHANNEL_NAME_LENGTH = NAME_LEN = 64`（`rpl_info.h:41`）；数量上限 256。

#### 双主 / 环形：replicate_same_server_id 与防循环

```
A ⇄ B          （双主）        A → B → C → A   （环）
```

双写拓扑（两个节点互为主备，两边都可写）在 GTID 时代的正确性约束：

1. **`log_replica_updates=ON` 必须**——否则对方写来的事务进不了自己的 binlog，回传链断裂。
2. **`replicate_same_server_id` 控制"自己写的事务要不要回放"**：默认 OFF 时，从库发现事件的 `server_id == 自己的 server_id` 就直接跳过（`log_event` 的应用入口判定）——这是防环的基础机制：A 写的事务经 B 回传 A 时被 A 跳过。双主/环拓扑里**双方都要开** `replicate_same_server_id=ON`？**不是**——恰恰相反：**保持 OFF 才能防环**。开启它意味着"连自己 server_id 的事件也回放"，只用于特殊场景（如通过中间件过滤后回灌），且官方手册明确警告开启它可能在环拓扑里造成**无限循环**。
3. **GTID 是真正的防环手段**：即使事件回来了，A 发现该 GTID 已在 `gtid_executed` 里，直接跳过（GTID 幂等）。启动时还有硬校验：`opt_log_replica_updates && replicate_same_server_id && gtid_mode != ON` 时**拒绝启动**（`ER_RPL_INFINITY_DENIED`，mysqld.cc）——匿名事务 + 双写 + 回放自己 = 必然无限循环，直接不让你起。
4. **server_id / server_uuid 必须全局唯一**：IO thread 连接时检测到与主库相同会直接停止（rpl_replica.cc 的 `ER_REPLICA_FATAL_ERROR` 分支）。server_id 用于"跳自己"，uuid 用于 GTID 命名空间——重复则一切混乱。

**★★★ 防循环的判定代码（两道防线）**

① IO 线程侧——不写进 relay log（`sql/rpl_replica.cc:8109`，在 `queue_event()` 里）：

```cpp
  s_id = uint4korr(buf + SERVER_ID_OFFSET);
  s_id &= opt_server_id_mask;      /* ★ 只在比较时掩掉 server_id_bits 无关位 */

  if ((s_id == ::server_id && !mi->rli->replicate_same_server_id) ||
      (mi->ignore_server_ids->dynamic_ids.size() > 0 &&
       mi->shall_ignore_server_id(s_id) && ...)) {
    /* Do not write it to the relay log. */
```

② SQL 线程侧——再拦一次（`sql/log_event.cc:1082`，在 `Log_event::do_shall_skip()` 里）：

```cpp
  if ((server_id == ::server_id && !rpl->replicate_same_server_id) ||
      (rli->slave_skip_counter == 1 && rli->is_in_group()))
    return EVENT_SKIP_IGNORE;
```

> ★★ 两道都要，源码注释明说原因：*"we must also keep this test in the SQL thread, in case somebody upgrades a 4.0 slave which has a not-filtered relay log"*——relay log 里可能已经躺着没被过滤的事件。
> ⚠️ **不存在 `is_event_from_self()` 函数**——是内联表达式，共 3 处（上面 2 处 + `Rotate_log_event::do_update_pos()` 的取反形式 `log_event.cc:5694`）。

**★ 为什么默认 OFF 能切断环**：

1. A 写事务进 A 的 binlog，事件 header 里 `server_id = A`
2. 经 B 回放、B 写进**自己的 binlog**，但 **event 的 server_id 字段保留原始值 = A**（源码注释：*"we still put the full unmasked server_id into the Relay log"*）
3. 该事件回到 A 时，A 的 IO 线程比较 `s_id == ::server_id` → 相等 → 默认 OFF → **不写入 A 的 relay log**

**★ 连接阶段的两道硬校验**（`rpl_replica.cc:2572` / `:2806`）：

```cpp
    if (!strcmp(::server_uuid, master_row[0]) && !mi->rli->replicate_same_server_id) {
      errmsg = "The replica I/O thread stops because source and replica have equal "
               "MySQL server UUIDs; these UUIDs must be different for replication to work.";
      ret = 1;                                    /* ★ UUID 相同 → 致命错误 */
    }
```
```cpp
    if ((::server_id == (mi->master_id = strtoul(master_row[0], nullptr, 10))) &&
        !mi->rli->replicate_same_server_id) {
      errmsg = "The replica I/O thread stops because source and replica have equal "
               "MySQL server ids; ...";           /* ★ server_id 相同 → 致命错误 */
    }
```

**★★ 第一道防线的完整上下文**（`sql/rpl_replica.cc:8100`，★ 命中后仍要推进位点的细节）：

```cpp
  s_id = uint4korr(buf + SERVER_ID_OFFSET);

  /*
    If server_id_bits option is set we need to mask out irrelevant bits
    when checking server_id, but we still put the full unmasked server_id
    into the Relay log so that it can be accessed when applying the event
  */
  s_id &= opt_server_id_mask;

  if ((s_id == ::server_id && !mi->rli->replicate_same_server_id) ||
      /*
        the following conjunction deals with IGNORE_SERVER_IDS, if set
        If the master is on the ignore list, execution of
        format description log events and rotate events is necessary.
      */
      (mi->ignore_server_ids->dynamic_ids.size() > 0 &&
       mi->shall_ignore_server_id(s_id) &&
       (s_id != mi->master_id ||
        (event_type != binary_log::FORMAT_DESCRIPTION_EVENT &&
         event_type != binary_log::ROTATE_EVENT)))) {
    /*
      Do not write it to the relay log.
      a) We still want to increment mi->get_master_log_pos(), so that we won't
      re-read this event from the master if the slave IO thread is now
      stopped/restarted (more efficient if the events we are ignoring are big
      LOAD DATA INFILE).
      b) We want to record that we are skipping events, for the information of
      the slave SQL thread, otherwise that thread may let
      rli->group_relay_log_pos stay too small if the last binlog's event is
      ignored.
      But events which were generated by this slave and which do not exist in
      the master's binlog (i.e. Format_desc, Rotate & Stop) should not increment
      mi->get_master_log_pos().
      ...
    */
    if (!(s_id == ::server_id && !mi->rli->replicate_same_server_id) ||
        (event_type != binary_log::FORMAT_DESCRIPTION_EVENT &&
         event_type != binary_log::ROTATE_EVENT &&
         event_type != binary_log::STOP_EVENT)) {
      rli->relay_log.lock_binlog_end_pos();
      mi->set_master_log_pos(mi->get_master_log_pos() + inc_pos);
      memcpy(rli->ign_master_log_name_end, mi->get_master_log_name(), FN_REFLEN);
      rli->ign_master_log_pos_end = mi->get_master_log_pos();
      rli->relay_log.update_binlog_end_pos(false);
      rli->relay_log.unlock_binlog_end_pos();
    }
```

> ★★ 三个易忽略的细节：
> 1. **跳过也要推进 `master_log_pos`**——否则重启后会把整个大事件重读一遍（注释举例 LOAD DATA INFILE）。
> 2. **FD / Rotate / Stop 三类"自己产生"的事件不推进位点**（它们不存在于主库 binlog）——但 `IGNORE_SERVER_IDS` 过滤掉的**远端**事件**要推进**。
> 3. `ign_master_log_name_end` / `ign_master_log_pos_end` 是给 SQL 线程看的"我跳过了一段"标记。

**★ `IGNORE_SERVER_IDS` 用二分查找**（`sql/rpl_mi.cc:308`，★ 要求 dynamic_ids 有序）：

```cpp
bool Master_info::shall_ignore_server_id(ulong s_id) {
  return std::binary_search(ignore_server_ids->dynamic_ids.begin(),
                            ignore_server_ids->dynamic_ids.end(), s_id);
}
```

**★ 第二道防线是"取反形式"**（`sql/log_event.cc:5694`，★ 与第一道互补）：

```cpp
  if ((server_id != ::server_id || rli->replicate_same_server_id) &&
      !is_relay_log_event() && !in_group) {
```
> ★ 即"**不是**我产生的，**或**我允许复制自己产生的"才更新 group 位点——与 `queue_event` 的 `(s_id == ::server_id && !replicate_same_server_id)` 正好是 De Morgan 关系。

**★ `server_id_bits` 掩码**（★ 第三处机制，`sql/mysqld.cc:6259`）：

```cpp
static Sys_var_uint Sys_server_id_bits(
    "server_id_bits", "Set number of significant bits in server-id",
    GLOBAL_VAR(opt_server_id_bits), CMD_LINE(REQUIRED_ARG), VALID_RANGE(0, 32),
    DEFAULT(32), BLOCK_SIZE(1));
```
```cpp
  opt_server_id_mask =
      (opt_server_id_bits == 32) ? ~ulong(0) : (1 << opt_server_id_bits) - 1;
  if (server_id != (server_id & opt_server_id_mask)) { ... }
```
```cpp
/* sql/log_event.cc:1013 —— event 头里同时保存两个值 */
  server_id = common_header->unmasked_server_id & opt_server_id_mask;
```
> ★ 默认 32 位即**不裁剪**。设为 N 时只比较低 N 位——用于大拓扑里"server_id 高位可忽略"的场景。注意 **relay log 里写的是 `unmasked_server_id`**（完整值）。

**★ 启动时的组合校验**（`sql/mysqld.cc:6473`）：

```cpp
  if (opt_log_replica_updates && replicate_same_server_id) {
    if (opt_bin_log && global_gtid_mode.get() != Gtid_mode::ON) {
      LogErr(ERROR_LEVEL, ER_RPL_INFINITY_DENIED);
      unireg_abort(MYSQLD_ABORT_EXIT);            /* ★★ 拒绝启动 */
    } else
      LogErr(WARNING_LEVEL, ER_RPL_INFINITY_IGNORED);
  }
```

> ⚠️ **`replicate_same_server_id` 不能 `SET GLOBAL`**——它只是 `my_getopt` 命令行选项（`sql/mysqld.cc:9170`），`sql/sys_vars.cc` 里搜它作为变量名 **0 命中**。名称**未改名**。

双主写冲突（两边同时改同一行）**复制层解决不了**：两边各自成功，复制时以先到者为准，后到者 1032/1062 报错。所以生产上双主通常配"应用层只写一边"或"按规则分片写两边"，复制只是把数据合并。

#### Group Replication（一句话边界）

MGR 是"成员等权 + Paxos 定序 + 认证冲突检测"的同步复制插件，与本节异步拓扑是不同体系；其 XCom（Paxos）与 ticket 屏障的简述见 binlog.md「MGR 与 XCom」，本篇不展开。

### 复制过滤器

#### 参数全集与语义

过滤器定义在 `rpl_filter`（每个 `Rpl_filter` 实例挂在 `rli` 上），参数分三类：

| 参数 | 粒度 | 语义 |
|------|------|------|
| `replicate_do_db` / `replicate_ignore_db` | 库名（精确） | 只应用/不应用指定库的事件 |
| `replicate_do_table` / `replicate_ignore_table` | 库.表（精确） | 同上，表级 |
| `replicate_wild_do_table` / `replicate_wild_ignore_table` | 库.表（通配符） | `%`/`_` 模式匹配 |
| `replicate_rewrite_db` | 库对映射 | **改写**而不是过滤：从库上 `A→B` 映射（如把 `prod` 库应用到 `prod_test` 库） |

#### 过滤发生的位置与代价

过滤在**从库 SQL thread 应用时**判定（理论基础 5 节）：事件已经传过来、写进 relay log，只是不执行。过滤掉的事务其 GTID **依然记入 `gtid_executed`**（否则 exclude 集合会永远请求它、且与主库集合不一致导致分叉检测误报）。这带来一个隐蔽语义：**被过滤的 GTID 是"空洞"**——从库执行过（标记了）但数据没有。后续若取消过滤，这些事务不会重放；若从库被提升为主库，它的 binlog 里也没有这些事务——这正是 GTID 空洞概念（gtid.md 的 `gtids_only_in_table`）在拓扑层的来源之一。

#### 经典坑：db 级过滤器与 STATEMENT 模式

`replicate_do_db` 的判定依据是**当前默认数据库**（`USE` 的库），不是语句实际涉及的库。STATEMENT 模式下 `USE a; UPDATE b.t SET ...` 会按库 `a` 判定而不是 `b`——历史上造成大量"过滤失效/误过滤"事故。ROW 模式按事件携带的库名判定，语义清晰。官方手册明确建议：**db 级过滤只对 ROW 模式可靠**；能用表级就用表级。

### 延迟复制

`SOURCE_DELAY`（或 `CHANGE REPLICATION SOURCE TO SOURCE_DELAY=N`）让从库把每个事务**推迟 N 秒再应用**。典型用途：防误操作（主库误 `DROP` 后，延迟从库还有 N 秒窗口抢救数据）。

实现核心 `sql_delay_event`（rpl_replica.cc，SQL thread 应用每个事件前调用）：

```cpp
static int sql_delay_event(Log_event *ev, THD *thd, Relay_log_info *rli) {
  time_t sql_delay = rli->get_sql_delay();
  ...
  if (sql_delay) {
    int type = ev->get_type_code();
    time_t sql_delay_end = 0;

    if (rli->commit_timestamps_status == Relay_log_info::COMMIT_TS_UNKNOWN &&
        (type == binary_log::GTID_LOG_EVENT ||
         type == binary_log::ANONYMOUS_GTID_LOG_EVENT)) {
      if (static_cast<Gtid_log_event *>(ev)->has_commit_timestamps && ...) {
        rli->commit_timestamps_status = Relay_log_info::COMMIT_TS_FOUND;
      } else {
        rli->commit_timestamps_status = Relay_log_info::COMMIT_TS_NOT_FOUND;
      }
    }

    if (rli->commit_timestamps_status == Relay_log_info::COMMIT_TS_FOUND) {
      if (type == binary_log::GTID_LOG_EVENT || ...) {
        // 主库提交时刻(微秒) + 延迟秒数 = 本事务允许执行的时刻
        sql_delay_end = ceil((static_cast<Gtid_log_event *>(ev)
                                  ->immediate_commit_timestamp) /
                             1000000.00) +
                        sql_delay;
      }
    } else {
      // 主库不支持 commit timestamps：退化为"收到后睡 N 秒"
      ...
    }
```

两个关键设计：

1. **延迟基准是主库的提交时刻，不是从库的接收时刻**。`immediate_commit_timestamp`（Gtid_log_event 携带，微秒）——若以接收时刻为基准，网络抖动与 relay log 积压会让延迟失真。以提交时刻为基准，"应用时刻 = 主库提交 + N 秒"严格成立。
2. **`commit_timestamps_status` 状态机向后兼容**：`COMMIT_TS_UNKNOWN → COMMIT_TS_FOUND / COMMIT_TS_NOT_FOUND`。第一个 GTID 事件决定主库是否支持时间戳（老主库没有此字段），不支持则退化为"每事件睡 N 秒"的近似语义。这个状态机**每 CHANNEL 独立**（挂在 `rli` 上）。

### GTID_ONLY 模式

`CHANGE REPLICATION SOURCE TO GTID_ONLY=1`（8.0.27+）。约束（源码校验链）：

- **强制 `SOURCE_AUTO_POSITION=1`**——GTID_ONLY 下不允许 file+pos（位点仓库根本没有文件名可存）。
- 位点仓库（`mysql.slave_master_info` / `mysql.slave_relay_log_info`）**不再持久化 binlog 文件名与偏移**，只存 GTID 集合。
- `SOURCE_AUTO_POSITION=0` 与 GTID_ONLY 组合被拒（校验代码明确报错）。

动机：file+pos 字段在 auto position 下本来就是冗余的（从不用它定位），却承担了"仓库损坏/不一致"的风险面（replica.md 的 recovery 大量代码在维护这些字段的一致性）。GTID_ONLY 把复制元数据**收敛到纯 GTID**，使位点仓库、relay log recovery、exclude 集合三者只围绕 GTID 一种真相——failover 场景的元数据复杂度显著下降。这是复制体系"GTID 一统"的最后一步。

**★ 本质是 `flush_info()` 直接 return 0**（`sql/rpl_rli.cc:1927`，★ 最硬的一条证据）：

```cpp
int Relay_log_info::flush_info(const int flush_flags) {
  if (!inited) return 0;

  if (!(flush_flags & RLI_FLUSH_IGNORE_GTID_ONLY) && mi->is_gtid_only_mode()) {
    return 0;                    /* ★★★ applier 位点根本不落盘 */
  }
```

receiver 侧同理（`sql/rpl_replica.cc:8246`）：

```cpp
    if (flush_master_info(mi, false /*force*/, ..., mi->is_gtid_only_mode()))
      res = QUEUE_EVENT_ERROR_FLUSHING_INFO;
    if (mi->is_gtid_only_mode()) mi->update_flushed_relay_log_info();
```

**★ recovery 时不再需要位点**（`sql/rpl_replica.cc:1095`）：

```cpp
  // If GTID ONLY is enable the receiver doesn't care about these positions
  if (!mi->is_gtid_only_mode()) {
    mi->set_master_log_pos(...);
    mi->set_master_log_name(...);
    ...
  } else {
```

> ★ 空位点（`group_master_log_name[0] == 0`）在 GTID_ONLY 下是**正常状态**，不再需要 `find_first_relay_log_with_rotate_from_master()` 去猜。

**★★ 三点真名核查**：

1. ⚠️ **GTID_ONLY 不是全局系统变量**，是 **per-channel** 的 `CHANGE REPLICATION SOURCE TO GTID_ONLY=` 选项（语法 `sql_yacc.yy:3248`，**只接受 0/1**），状态存在 `Master_info::m_gtid_only_mode`（`rpl_mi.h:806`）。
2. **Group Replication 的 channel 强制开启**（`rpl_replica.cc:10423`）：`mi->set_auto_position(true); mi->rli->set_require_row_format(true); mi->set_gtid_only_mode(true);`
3. **反向约束**：任一 channel 开着 GTID_ONLY 时，`gtid_mode` 不能调离 `ON`（`sys_vars.cc:4627`，报 `ER_CANT_SET_GTID_MODE` 并提示先设 `GTID_ONLY=0`）。

**★ 开关本身没有任何校验**（`sql/rpl_mi.cc:810`，★ 就是个 bool）：

```cpp
void Master_info::set_gtid_only_mode(bool gtid_only_mode) {
  m_gtid_only_mode = gtid_only_mode;
}

bool Master_info::is_gtid_only_mode() const { return m_gtid_only_mode; }
```
> ★ 校验全在 `CHANGE REPLICATION SOURCE` 与 `gtid_mode` 的 `ON_CHECK` 里（见下）。

**★ 全仓只有 2 处 `set_gtid_only_mode()`**（★ 第二处是隐藏的自动打开路径）：

```cpp
/* sql/rpl_replica.cc:9897 —— 用户显式设置 */
  if (lex_mi->m_gtid_only != LEX_MASTER_INFO::LEX_MI_UNCHANGED) {
    mi->set_gtid_only_mode(
        (lex_mi->m_gtid_only == LEX_MASTER_INFO::LEX_MI_ENABLE));
  }
```
```cpp
/* sql/rpl_replica.cc:10423 —— ★★ 组复制通道被强制打开 */
  if (channel_map.is_group_replication_channel_name(lex_mi->channel)) {
    mi->set_auto_position(true);
    mi->rli->set_require_row_format(true);
    mi->set_gtid_only_mode(true);
  }
```

**★★ recovery 时到底跳过了哪一步**（`sql/rpl_replica.cc:1082`，★ 含完整的 else 分支）：

```cpp
static void recover_relay_log(Master_info *mi) {
  Relay_log_info *rli = mi->rli;

  // If GTID ONLY is enable the receiver doesn't care about these positions
  if (!mi->is_gtid_only_mode()) {
    // Set Receiver Thread's positions as per the recovered Applier Thread.
    mi->set_master_log_pos(std::max<ulonglong>(
        BIN_LOG_HEADER_SIZE, rli->get_group_master_log_pos()));
    mi->set_master_log_name(rli->get_group_master_log_name());

    LogErr(WARNING_LEVEL, ER_RPL_RECOVERY_FILE_SOURCE_POS_INFO, ...);
  } else {
    LogErr(WARNING_LEVEL, ER_RPL_RELAY_LOG_RECOVERY_GTID_ONLY);
  }

  // Start with a fresh relay log.
  rli->set_group_relay_log_name(rli->relay_log.get_log_fname());
  rli->set_group_relay_log_pos(BIN_LOG_HEADER_SIZE);
  /*
    Clear the retrieved GTID set so that events that are written partially
    will be fetched again.
  */
  if (global_gtid_mode.get() == Gtid_mode::ON &&
      !channel_map.is_group_replication_channel_name(rli->get_channel())) {
    (const_cast<Gtid_set *>(rli->get_gtid_set()))->clear_set_and_sid_map();
  }
}
```

> ★★ 精确结论：GTID_ONLY=1 时**跳过"把 receiver 位置回退到 applier 位置"这一步**（因为不再用文件位点），**只打一条警告**。后面"新开 relay log + 清空 Retrieved GTID set"两步**两者都做**。
> ★ 另一处（`rpl_replica.cc:1184`）：GTID_ONLY=1 时**不再调 `find_first_relay_log_with_rotate_from_master()`** 去从 relay log 里"找第一个 rotate 事件"推断位点。

**★ 语法：只接受 0/1**（`sql_yacc.yy:3248`）：

```cpp
        | GTID_ONLY_SYM EQ real_ulong_num
          {
            switch($3) {
            case 0: Lex->mi.m_gtid_only = LEX_MASTER_INFO::LEX_MI_DISABLE; break;
            case 1: Lex->mi.m_gtid_only = LEX_MASTER_INFO::LEX_MI_ENABLE;  break;
            default:
                YYTHD->syntax_error_at(@3,
                  "You have an error in your CHANGE REPLICATION SOURCE syntax; GTID_ONLY only accepts values 0 or 1");
                MYSQL_YYABORT;
            }
          }
```
> ★ GTID_ONLY 是 8.0 新增语法，**没有 `MASTER_*` 旧写法**，所以也不需要 `push_deprecated_warn`。

**★ 相关的 6 个错误码**（`share/messages_to_clients.txt`）：

| 错误码 | 行号 | 文本要点 |
|---|---|---|
| `ER_CHANGE_REPLICATION_SOURCE_NO_OPTIONS_FOR_GTID_ONLY` | 9663 | "You need GTID_MODE = ON, SOURCE_AUTO_POSITION = 1 and REQUIRE_ROW_FORMAT = 1." |
| `ER_CHANGE_REP_SOURCE_CANT_DISABLE_REQ_ROW_FORMAT_WITH_GTID_ONLY` | 9666 | GTID_ONLY=1 时不能关 REQUIRE_ROW_FORMAT |
| `ER_CHANGE_REP_SOURCE_CANT_DISABLE_AUTO_POSITION_WITH_GTID_ONLY` | 9669 | GTID_ONLY=1 时不能关 SOURCE_AUTO_POSITION |
| `ER_CHANGE_REP_SOURCE_CANT_DISABLE_GTID_ONLY_WITHOUT_POSITIONS` | 9672 | 关闭时必须提供 SOURCE_LOG_FILE/POS |
| `ER_CANT_USE_GTID_ONLY_WITH_GTID_MODE_NOT_ON` | 9681 | 启动期校验 |
| `ER_WARN_C_DISABLE_GTID_ONLY_WITH_SOURCE_AUTO_POS_INVALID_POS` | 9684 | 关闭后位点可能过期 |

> ★ 三条互锁关系：`GTID_ONLY=1` ⇒ 强制 `AUTO_POSITION=1` + `REQUIRE_ROW_FORMAT=1` + `GTID_MODE=ON`。

### 故障转移

#### failover 的本质：三个子问题的组合

1. **选谁当新主库**（最追平的从库；半同步下是已确认的从库）
2. **其余从库切到新主库**（重新配置 SOURCE_HOST + 位点）
3. **旧主库恢复后的处置**（降级为从库，追新主库的差集）

GTID 让 2 和 3 从"手工换算位点"变成"自动收敛"：

```
旧主库 S（挂了）          新主库 B（提升）
   ▲                          │
   │                          │ 事务 t1..t5（S 崩溃前，B 已收到并执行）
   └── replica C ─────────────┘ 事务 t6（S 崩溃后，客户端写到 B）
```

- **C 切换到 B**：`CHANGE REPLICATION SOURCE TO SOURCE_HOST='B', SOURCE_AUTO_POSITION=1`。C 的 exclude 集合含 t1..t5，B 用 auto position 算法定位到 t6 开始发——**C 自动补齐差集，不多不少**。
- **S 恢复后降级**：同样指向 B，S 的 exclude 集合含 t1..t5，B 把 t6 发给 S，S 追平后成为普通从库。
- **分叉检测**：若 B 执行了 S 没有的事务（如 C 曾被误提升写过数据，或 B 的 binlog 被截断），dump 握手时集合校验失败——`ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE`（从库比主库多）或 `ER_SOURCE_HAS_PURGED_REQUIRED_GTIDS`（主库 purge 了从库需要的），语义与判定见 gtid.md exclude 集合协议与 replica.md 错误码速查。**failover 的第一步永远是确认候选新主库的 `gtid_executed` 是旧主库的超集**。

#### 异步 vs 半同步的 failover 差异

| | 异步 | 半同步（after_sync） |
|---|---|---|
| 切到任意从库 | 可能丢"未传过去"的事务 | 若该从库不是 ACK 接收者，同样可能丢 |
| 正确姿势 | 选**追平**的从库（`Read_Source_Log_Pos` 对齐） | 半同步保证**至少一个**从库收到；配合 MGR 的 auto failover 才有"自动选对" |
| RPO | 视主库提交速度与切换速度，秒~分钟级 | 零（after_sync 语义下） |
| 检测与切换 | 手工/外部工具 | `SOURCE_CONNECTION_AUTO_FAILOVER`（组复制内自动换连接，8.0.27+） |

关键认知：**半同步解决"丢数据"，不解决"选对主库"**。它保证至少有从库拿到了事务，但"哪一个拿到了"需要外部机制记录。这是官方 failover 文档（Replication 手册 19.4.8 Switching Sources During Failover）反复强调的：failover 的**数据安全**来自"确认从库追平 + GTID 差集收敛"，半同步只是把"追平"这件事变得可证明。

**★★★ 源码实证：`SOURCE_CONNECTION_AUTO_FAILOVER` 不是全局变量**（`sql/rpl_mi.h:506`）：

```cpp
  /**
    Checks if Asynchronous Replication Connection Failover feature is enabled.
    ...
  */
  bool is_source_connection_auto_failover() {
    return m_source_connection_auto_failover;
  }
  void set_source_connection_auto_failover() { m_source_connection_auto_failover = true; }
  void unset_source_connection_auto_failover() { m_source_connection_auto_failover = false; }

  /**
    Return current position in the Failover source list ...
    Based on this position next failover source is selected.
  */
  uint get_failover_list_position() { return m_failover_list_position; }
  void reset_failover_list_position() { m_failover_list_position = 0; }
  void increment_failover_list_position() { m_failover_list_position++; }

  /**
    Checks if network error has occurred.
    ...
  */
  bool is_network_error() { return m_network_error; }
  void set_network_error() { m_network_error = true; }
```

> ⚠️ **勘误**：不存在全局系统变量 `source_connection_auto_failover`——它是 **per-channel** 的 `Master_info::m_source_connection_auto_failover`（`rpl_mi.h:626`），由 `CHANGE REPLICATION SOURCE TO SOURCE_CONNECTION_AUTO_FAILOVER=` 设置（`sql_yacc.yy:3231`）。

**★★★ "换源"的实现就是一次 `goto`**（`sql/rpl_replica.cc:5787`，★ 这是 failover 最核心的 15 行）：

```cpp
      /*
        Get the sender to connect to.
        If there is a STOP REPLICA ongoing for any channel, that is, a
        channel_map lock cannot be acquired by this channel IO thread,
        then this channel IO thread does skip the next sender selection.
      */
      Async_conn_failover_manager::DoAutoConnFailoverError update_source_error =
          Async_conn_failover_manager::DoAutoConnFailoverError::retriable_error;
      if (!channel_map.tryrdlock()) {
        update_source_error =
            Async_conn_failover_manager::do_auto_conn_failover(mi, false);
        channel_map.unlock();
      }
      ...
      if (Async_conn_failover_manager::DoAutoConnFailoverError::
              no_sources_error != update_source_error) {
        /* Wait before reconnect to avoid resources starvation. */
        my_sleep(1000000);                        /* ★ 等 1 秒，避免资源饥饿 */

        if (!check_io_slave_killed(thd, mi, "...")) {
          if (mysql) {
            LogErr(SYSTEM_LEVEL, ER_RPL_ASYNC_NEXT_FAILOVER_CHANNEL_SELECTED,
                   mi->retry_count, old_user.c_str(), old_host.c_str(),
                   old_port, mi->get_for_channel_str(), mi->get_user(),
                   mi->host, mi->port);
            thd->clear_active_vio();
            mysql_close(mysql);
            mi->mysql = nullptr;
            mysql = nullptr;
          }
          goto connect_init;      /* ★★ 跳回连接起点，用新 host/port 重连 */
        }
      }
```

> ★★ **`do_auto_conn_failover()` 做的事就是改写 `mi->host` / `mi->port` / `mi->user`，然后 `goto connect_init` 重来一遍**。整个"failover"在 MySQL 社区版里就是这个动作——**它只换连接，不做选主、不做数据补齐**（后者靠 GTID auto position 自动完成）。

**★ 主库多数派（quorum）校验**（`rpl_replica.cc:5459`，★ 防"连到孤立的旧主库"）：

```cpp
      quorum_status =
          Async_conn_failover_manager::get_source_quorum_status(mysql, mi);
      switch (quorum_status) {
        case ...::fatal_error:
        case ...::no_quorum_error:
          ret = 1;                              /* ★ 无多数派 → 不认这个源 */
          break;
        case ...::transient_network_error:
          ret = 2;
          break;
        case ...::no_error: default: break;
      }
```

**★ 候选源列表存在系统表里**（`scripts/mysql_system_tables.sql:467`，★ 不是内存结构）：

```sql
CREATE TABLE IF NOT EXISTS replication_asynchronous_connection_failover (
    Channel_name CHAR(64) ... NOT NULL COMMENT 'The replication channel name...',
    Host CHAR(255) CHARACTER SET ASCII NOT NULL,
    Port INTEGER UNSIGNED NOT NULL,
    Network_namespace CHAR(64) COMMENT '... If its value is empty, connections use the default (global) namespace.',
    Weight TINYINT UNSIGNED NOT NULL COMMENT 'The order in which the replica shall try to switch the connection over to when there are failures. Weight can be set to a number between 1 and 100, where 100 is the highest weight and 1 the lowest.',
    Managed_name CHAR(64) ... NOT NULL DEFAULT '' COMMENT 'The name of the group which this server belongs to.',
    PRIMARY KEY(Channel_name, Host, Port, Network_namespace, Managed_name), ...)
```
> ★ 还有配套的 `replication_asynchronous_connection_failover_managed`（管 MGR 组，`Configuration` 是 JSON：`{"Primary_weight": 80, "Secondary_weight": 60}`）。

**★★ 开着 failover 就不许关 GTID**（`sql/sys_vars.cc:4612`）：

```cpp
  /*
    Cannot set OFF when source_connection_auto_failover is enabled for any
    channel.
  */
  if (new_gtid_mode != Gtid_mode::ON) {
    for (mi_map::iterator it = channel_map.begin(); it != channel_map.end(); it++) {
      Master_info *mi = it->second;
      if (mi != nullptr && mi->is_source_connection_auto_failover()) {
        my_error(ER_DISABLE_GTID_MODE_REQUIRES_ASYNC_RECONNECT_OFF, MYF(0),
                 Gtid_mode::to_string(new_gtid_mode));
        goto err;
      }
    }
```

**★ 重连策略：默认无限重试**（`rpl_replica.cc:8500`，★ 注释解释了为什么）：

```cpp
    /*
      By default we try forever. The reason is that failure will trigger
      master election, so if the user did not set mi->retry_count we
      do not want to have election triggered on the first failure to
      connect
    */
    if (++err_count == mi->retry_count) {
      if (is_network_error(last_errno) && is_io_thread) mi->set_network_error();
      replica_was_killed = true;
      break;
    }
    slave_sleep(thd, mi->connect_retry, ...);
```
```cpp
/* rpl_replica.cc:5245 —— 运行期重连的退避 */
static int try_to_reconnect(...) {
  ...
  if ((*retry_count)++) {
    if (*retry_count > mi->retry_count) return 1;  // Don't retry forever
    slave_sleep(thd, mi->connect_retry, io_slave_killed, mi);
  }
```
> ★★ 两处配合：默认（`retry_count=0`）**永远重试**，因为"放弃重试"会被外部编排解读为"该选主了"——这是把 failover 触发权**显式留给外部**的设计。

**★ failover 后的定位约束**（`rpl_replica.cc:3067` / `:8023`）：

```cpp
    if (mi->is_auto_position() && master_gtid_mode != Gtid_mode::ON) {
      mi->report(ERROR_LEVEL, ER_REPLICA_FATAL_ERROR,
                 "The replication receiver thread cannot start in "
                 "AUTO_POSITION mode: the source has GTID_MODE = %.192s "
                 "instead of ON.", Gtid_mode::to_string(master_gtid_mode));
      return 1;
    }
```
> ★ **`SOURCE_AUTO_POSITION=1` 是唯一"自动收敛"的定位方式**，它要求新主库 `GTID_MODE=ON` 且不接受匿名事务（`ER_CANT_REPLICATE_ANONYMOUS_WITH_AUTO_POSITION`）。这就是"GTID 是 failover 前提"的源码根。

#### 传统 file+pos failover（无 GTID 时）

官方手册流程：所有从库 `STOP REPLICA IO_THREAD` 对齐到旧主库末位点（`SHOW REPLICA STATUS` 的 `Read_Source_Log_Pos` 与主库最后 binlog 位点对齐）→ 选出追平者 → 其余从库 `CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE=<新主库对应文件>, SOURCE_LOG_POS=<换算偏移>`。换算偏移的可靠性依赖"所有从库执行的是同一份 binlog 内容"（手册原文：all the servers in the group are typically executing the same events from the same binary log file），稍有偏差即漏/重事务。**这正是 GTID 想要消灭的流程**。

### 运维操作

#### CHANGE REPLICATION SOURCE：参数全集

`LEX_MASTER_INFO`（sql_lex.h）承载的全部选项：`SOURCE_HOST/SOURCE_PORT/SOURCE_USER/SOURCE_PASSWORD`（连接）、`SOURCE_LOG_FILE/SOURCE_LOG_POS`（file+pos 位点）、`SOURCE_AUTO_POSITION`（GTID 自动定位）、`SOURCE_DELAY`（延迟复制）、`SOURCE_CONNECT_RETRY/SOURCE_RETRY_COUNT`（重连间隔与次数）、`SOURCE_HEARTBEAT_PERIOD`（心跳）、`SOURCE_BIND`（本地网卡）、`SOURCE_SSL_*`（TLS 全套）、`SOURCE_COMPRESSION_ALGORITHMS`（传输压缩）、`SOURCE_IGNORE_SERVER_IDS`、`REPLICATE_DO/IGNORE_*`（过滤，按 CHANNEL）、`GTID_ONLY`、`SOURCE_CONNECTION_AUTO_FAILOVER`、`PRIVILEGE_CHECKS_USER`、`REQUIRE_ROW_FORMAT`、`ASSIGN_GTIDS_TO_ANONYMOUS_TRANSACTIONS` 等。

几个关键校验（源码）：

- `SOURCE_AUTO_POSITION=1` 且 `gtid_mode=OFF` → 拒绝；
- `GTID_ONLY=1` 必须与 `SOURCE_AUTO_POSITION=1` 同开，且 `SOURCE_AUTO_POSITION=0` 时要同时给合法 `SOURCE_LOG_FILE/SOURCE_LOG_POS`；
- `SOURCE_CONNECTION_AUTO_FAILOVER=1` 要求位点仓库是 TABLE 且存有连接凭证（组复制管理通道的前提）。

#### START / STOP REPLICA

线程生命周期（`RUNNING → STOP_ACCEPTED → STOPPED` 状态机）见 replica.md「线程生命周期」。STOP 的语义是"请求停止"：SQL thread 在事务边界停（`UNTIL` 条件满足后也自动停）；`STOP REPLICA IO_THREAD` 只停拉取，SQL 继续消化存量 relay log。

#### RESET REPLICA 的语义分级

`RESET REPLICA`：删除 relay log 文件 + 重置 relay log 位点仓库（IO/SQL 位点清空），**不动**源信息（SOURCE_HOST 等保留）。
`RESET REPLICA ALL`：连源信息、过滤、凭证一起清——服务器彻底"忘记自己是从库"。失败重配的常见起点。

#### SHOW REPLICA STATUS 的关键字段解读

`Relay_Source_Log_File/Pos`（IO 位点）、`Exec_Source_Log_File/Pos`（SQL 位点）、`Seconds_Behind_Source`（SQL 落后）、`Retrieved_Gtid_Set`（收到过的）、`Executed_Gtid_Set`（应用过的）——**两者差集 = 已收到未应用的事务**，failover 追平确认就看这个差集是否为空。完整字段与观测入口见 replica.md「可观测性」。

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `log_replica_updates` | ON | Global（只读） | 从库回放事务是否写本机 binlog（级联/双主前提） |
| `replicate_same_server_id` | OFF | 启动 | 是否回放 server_id 相同的事件（环拓扑防循环依赖它保持 OFF） |
| `skip_replica_start` | OFF | 启动 | 启动时是否自动拉起复制线程 |
| `replica_net_timeout` | 60 | Global | IO thread 收不到数据的超时（断链重连） |
| `replica_type_conversions` | 空 | Global | 从库列类型隐式转换集合 |

| 状态变量 | 说明 |
|--------|------|
| `Com_change_replication_source` | CHANGE REPLICATION SOURCE 语句计数 |

**★★★ 补：`SHOW REPLICA STATUS` 是 60 列，不是常见的"约 57 列"**（`sql/rpl_replica.cc:3341`，★ 列定义函数精确 push 60 次）：

```cpp
static void show_slave_status_metadata(mem_root_deque<Item *> *field_list,
                                       int io_gtid_set_size,
                                       int sql_gtid_set_size) {
  field_list->push_back(new Item_empty_string("Replica_IO_State", 14));
  field_list->push_back(new Item_empty_string("Source_Host", HOSTNAME_LENGTH + 1));
  field_list->push_back(new Item_empty_string("Source_User", USERNAME_LENGTH + 1));
  field_list->push_back(new Item_return_int("Source_Port", 7, MYSQL_TYPE_LONG));
  field_list->push_back(new Item_return_int("Connect_Retry", 10, MYSQL_TYPE_LONG));
  field_list->push_back(new Item_empty_string("Source_Log_File", FN_REFLEN));
  field_list->push_back(new Item_return_int("Read_Source_Log_Pos", 10, MYSQL_TYPE_LONGLONG));
  ...
  field_list->push_back(new Item_return_int("Exec_Source_Log_Pos", 10, MYSQL_TYPE_LONGLONG));
  field_list->push_back(new Item_return_int("Relay_Log_Space", 10, MYSQL_TYPE_LONGLONG));
  field_list->push_back(new Item_empty_string("Until_Condition", 6));
  ...
  field_list->push_back(new Item_return_int("Seconds_Behind_Source", 10, MYSQL_TYPE_LONGLONG));
  ...
  field_list->push_back(new Item_empty_string("Channel_Name", CHANNEL_NAME_LENGTH));
  field_list->push_back(new Item_empty_string("Source_TLS_Version", FN_REFLEN));
  field_list->push_back(new Item_empty_string("Source_public_key_path", FN_REFLEN));
  field_list->push_back(new Item_return_int("Get_Source_public_key", sizeof(ulong), MYSQL_TYPE_LONG));
  field_list->push_back(new Item_empty_string("Network_Namespace", NAME_LEN + 1));
}
```

> ★★ **60 列全部用新术语**（Source/Replica）。老名（`Master_Host` 等）**不在源码里**，而是发送元数据前由 `rename_fields_use_old_replica_source_terms()` **动态替换**回来（见「容易误解的命名」）。

**★ 填充函数的三个细节**（`rpl_replica.cc:3452`，⚠️ 函数名**未改名**，且**恒返回 false**）：

```cpp
  mysql_mutex_lock(&mi->info_thd_lock);
  protocol->store(mi->info_thd ? mi->info_thd->proc_info_session(thd) : "",
                  &my_charset_bin);            /* ① Replica_IO_State */
  mysql_mutex_unlock(&mi->info_thd_lock);
```
> ★ `Replica_IO_State` / `Replica_SQL_Running_State` 都取自**线程自身的当前 stage 名**（`proc_info_session()`），不是独立字段。

```cpp
  protocol->store(mi->rli->get_group_relay_log_name() +
                      dirname_length(mi->rli->get_group_relay_log_name()),
                  &my_charset_bin);            /* ② Relay_Log_File：剥掉目录前缀 */
```

```cpp
  // SQL_Remaining_Delay —— 只在线程恰好处于 delay 等待态时才算
  if (slave_sql_running_state == stage_sql_thd_waiting_until_delay.m_name) {
    time_t t = time(nullptr), sql_delay_end = mi->rli->get_sql_delay_end();
    protocol->store((uint32)(t < sql_delay_end ? sql_delay_end - t : 0));
  } else
    protocol->store_null();
```

**★ 完整 60 列 → 字段来源对照**：

| # | 列 | 来源 | 行 |
|---|---|---|---|
| 1 | Replica_IO_State | `mi->info_thd->proc_info_session(thd)` | 3471 |
| 2-5 | Source_Host / User / Port / Connect_Retry | `mi->host` / `get_user()` / `port` / `connect_retry` | 3486-3489 |
| 6-7 | Source_Log_File / Read_Source_Log_Pos | `mi->get_master_log_name_info()` / `_pos_info()` | 3490-3491 |
| 8-9 | Relay_Log_File / Pos | `rli->get_group_relay_log_name()`+dirname_length / `_pos()` | 3492-3495 |
| 10 | Relay_Source_Log_File | `rli->get_group_master_log_name_info()` | 3496 |
| 11-12 | Replica_IO_Running / SQL_Running | `mi->slave_running` 三态 / `rli->slave_running` | 3498-3503 |
| 13-18 | Replicate_Do/Ignore_DB/Table、Wild_* | `rpl_filter->get_*()` | 3510-3521 |
| 19-21 | Last_Errno / Last_Error / Skip_Counter | `rli->last_error().*` / `rli->slave_skip_counter` | 3524-3526 |
| 22 | **Exec_Source_Log_Pos** | `rli->get_group_master_log_pos_info()` | 3527 |
| 23 | **Relay_Log_Space** | `rli->log_space_total`（`std::atomic<ulonglong>`） | 3528 |
| 24-26 | Until_Condition / Log_File / Log_Pos | `switch(rli->until_condition)` 8 分支 | 3532-3565 |
| 27-32 | Source_SSL_* 六列 | `mi->ssl` / `ssl_ca` / `ssl_capath` / `ssl_cert` / `ssl_cipher` / `ssl_key` | 3567-3572 |
| 33 | Seconds_Behind_Source | 内联（见下节） | 3592-3634 |
| 35-38 | Last_IO_* / Last_SQL_* | `mi->last_error()` / `rli->last_error()` | 3638-3644 |
| 39 | Replicate_Ignore_Server_Ids | 遍历 `mi->ignore_server_ids->dynamic_ids` | 3646-3666 |
| 40-42 | Source_Server_Id / UUID / Info_File | `mi->master_id` / `master_uuid` / `get_description_info()` | 3668-3671 |
| 43-45 | SQL_Delay / Remaining_Delay / SQL_Running_State | 见上 | 3673-3681 |
| 46-49 | Source_Retry_Count / Bind / 两个 Timestamp | `mi->retry_count` / `bind_addr` / `last_error().timestamp` | 3683-3689 |
| 50-51 | Source_SSL_Crl / Crlpath | `mi->ssl_crl` / `ssl_crlpath` | 3691-3693 |
| 52-53 | Retrieved / Executed_Gtid_Set | `io_gtid_set_buffer` / `sql_gtid_set_buffer` | 3695-3697 |
| 54 | Auto_Position | `mi->is_auto_position() ? 1 : 0` | 3699 |
| 55-60 | Replicate_Rewrite_DB / Channel_Name / TLS_Version / public_key_path / Get_... / Network_Namespace | 见源码 | 3701-3712 |

> ★★ `Until_Condition` 是**唯一按会话语法变字符串**的列（`rpl_replica.cc:3536`）：老语法显示 `"Master"`，新语法显示 `"Source"`——但**枚举名 `UNTIL_MASTER_POS` 未改名**。
> ⚠️ 源码行内注释仍是老名（`// Master_Server_id`、`// Slave_SQL_Running_State`）——**只有 push_back 的列名改了，注释没同步改**。这是"命名不一致"的现场。

**★ 补：其余复制相关状态变量**（⚠️ 下面这些是 `SHOW STATUS`，不是系统变量）：

```cpp
/* sql/mysqld.cc:4452 —— ⚠️ 不存在 Com_start_replica！真名是 Replica_start */
    {"replica_start",
     (char *)offsetof(System_status_var, com_stat[(uint)SQLCOM_SLAVE_START]),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    {"slave_start",   /* 同一个槽，数值恒等 */
     (char *)offsetof(System_status_var, com_stat[(uint)SQLCOM_SLAVE_START]),
     SHOW_LONG_STATUS, SHOW_SCOPE_ALL},
    {"replica_stop",  ... SQLCOM_SLAVE_STOP ...},
    {"slave_stop",    ... SQLCOM_SLAVE_STOP ...},
```
```cpp
/* sql/mysqld.cc:4297 —— SHOW 系列同样是"新名+旧名共用一个槽" */
    {"show_replicas",      ... SQLCOM_SHOW_SLAVE_HOSTS ...},
    {"show_slave_hosts",   ... SQLCOM_SHOW_SLAVE_HOSTS ...},
    {"show_replica_status", ... SQLCOM_SHOW_SLAVE_STAT ...},
    {"show_slave_status",   ... SQLCOM_SHOW_SLAVE_STAT ...},
```
> ⚠️ **两组勘误**：① 不存在 `Com_start_replica` / `Com_stop_replica`，真名 **`Replica_start` / `Replica_stop`**（无 `Com_` 前缀）；② 不存在 `Com_show_replica_status`，真名 **`Show_replica_status`**。
> ★ 新旧名**共用同一个 `com_stat[]` 槽**，所以 `Com_change_master` 与 `Com_change_replication_source` 数值恒等——不是两个独立计数器。

**★ 半同步状态变量**（`plugin/semisync/semisync_source_plugin.cc:432`，★ 变量名是**宏拼接**的）：

```cpp
#define STATUS_VAR_PREFIX "Rpl_semi_sync_source_"     /* 装旧库时变回 Rpl_semi_sync_master_ */

DEF_SHOW_FUNC(status, SHOW_BOOL)
DEF_SHOW_FUNC(clients, SHOW_LONG)
DEF_SHOW_FUNC(wait_sessions, SHOW_LONG)
...
static SHOW_VAR semi_sync_master_status_vars[] = {    /* ⚠️ 数组名仍叫 master，未改名 */
    {STATUS_VAR_PREFIX "status", (char *)&SHOW_FNAME(status), SHOW_FUNC, SHOW_SCOPE_GLOBAL},
    {STATUS_VAR_PREFIX "clients", ...},
    {STATUS_VAR_PREFIX "yes_tx", (char *)&rpl_semi_sync_source_yes_transactions, SHOW_LONG, ...},
    {STATUS_VAR_PREFIX "no_tx",  (char *)&rpl_semi_sync_source_no_transactions, SHOW_LONG, ...},
    {STATUS_VAR_PREFIX "wait_sessions", ...},
    {STATUS_VAR_PREFIX "no_times", ...},
    {STATUS_VAR_PREFIX "timefunc_failures", ...},
    {STATUS_VAR_PREFIX "wait_pos_backtraverse", ...},
    {STATUS_VAR_PREFIX "tx_wait_time", ...}, {STATUS_VAR_PREFIX "tx_waits", ...},
    {STATUS_VAR_PREFIX "tx_avg_wait_time", ...},
    {STATUS_VAR_PREFIX "net_wait_time", ...}, {STATUS_VAR_PREFIX "net_waits", ...},
    {STATUS_VAR_PREFIX "net_avg_wait_time", ...},
};
```

| 状态变量 | 含义 |
|---|---|
| `Rpl_semi_sync_source_status` | 半同步当前是否生效 |
| `Rpl_semi_sync_source_clients` | 半同步 replica 连接数 |
| **`Rpl_semi_sync_source_yes_tx`** / **`no_tx`** | **成功走半同步 / 降级为异步的事务数**（★ 判断半同步实际效果的两个关键数） |
| `Rpl_semi_sync_source_wait_sessions` | 正在等 ack 的会话数 |
| `Rpl_semi_sync_source_no_times` | 半同步被关闭的次数 |
| `Rpl_semi_sync_source_wait_pos_backtraverse` | wait 位置回退次数 |
| `Rpl_semi_sync_source_tx_waits` / `tx_avg_wait_time` | 事务等待 ack 的次数 / 平均时长 |

> ★ **replica 侧只有 1 个状态变量**：`Rpl_semi_sync_replica_status`（`semisync_replica_plugin.cc:43`）。
> ⚠️ 这些是**状态变量**不是系统变量；若加载的是旧插件库 `rpl_semi_sync_master`，前缀变回 `Rpl_semi_sync_master_`。
| `Replica_open_temp_tables` | 从库当前打开的临时表数（STOP 的约束之一） |

**★★ 补齐：复制相关的其余关键变量**（★ 前表未列的，逐个核实过行号）

| 变量 | 文件:行 | 默认 | 备注 |
|---|---|---|---|
| `server_id` | `sys_vars.cc:4071` | **1** | `PERSIST_AS_READONLY`，范围 0..UINT_MAX32 |
| `server_uuid` | `sys_vars.cc:4080` | 自动生成 | **READ_ONLY / NON_PERSIST / NO_CMD_LINE** |
| `gtid_mode` | `sys_vars.cc:6790` | — | OFF / OFF_PERMISSIVE / ON_PERMISSIVE / ON |
| `enforce_gtid_consistency` | `sys_vars.cc:6550` | **ON** | |
| `sync_binlog` | `sys_vars.cc:6404` | **1** | |
| `binlog_format` | `sys_vars.cc:1428` | **ROW** | 注释明说 *"deprecated and will be removed"* |
| `log_replica_updates` | `sys_vars.cc:6153` | **1** | READ_ONLY |
| `replica_parallel_workers` | `sys_vars.cc:6443` | **4** | 范围 0..MTS_MAX_WORKERS |
| `replica_preserve_commit_order` | `sys_vars.cc:4291` | **true** | |
| `replica_transaction_retries` | `sys_vars.cc:6432` | **10** | |
| `replica_net_timeout` | `sys_vars.cc:6301` | 60 | 范围 1..LONG_TIMEOUT |
| `relay_log_recovery` | `sys_vars.cc:6217` | **false** | READ_ONLY；名字本就没有 slave/master，未改 |
| `master_info_repository` | `sys_vars.cc:1714` | TABLE | ⚠️ **8.0 仍存在**，但 `DEPRECATED_VAR("")` |
| `relay_log_info_repository` | `sys_vars.cc:1724` | TABLE | 同上，同样 deprecated |
| `terminology_use_previous` | `sys_vars.cc:7679` | NONE | 自己也是 deprecated |

**★★★ 一个必须纠正的认知：复制过滤器不是系统变量**

`sql/sys_vars.cc` 里搜 `replicate_do_db` / `replicate_ignore_db` / `replicate_wild_do_table` 等**全部 0 命中**。这 8 个都是 **my_getopt 命令行选项**：

```cpp
/* sql/mysqld.cc:9128 起 */
    {"replicate-do-db", 0, "Make replication applier threads apply changes "
     "only to the specified database...", ...},
    {"replicate-same-server-id", 0, "In replication, if set to 1, do not skip "
     "events having our server id. Default value is 0 (to break infinite loops "
     "in circular replication). Can't be set to 1 if --log-replica-updates is used.",
     &replicate_same_server_id, &replicate_same_server_id, nullptr, GET_BOOL,
     NO_ARG, 0, 0, 0, nullptr, 0, nullptr},
```

> ★★ 所以 `replicate_do_db` 等**不能用 `SET GLOBAL`**，只能通过配置文件/命令行设置并重启。同理 `replicate_same_server_id`。

**★ `SHOW REPLICA STATUS` 关键列的来源**（⚠️ 填充函数**仍叫 `show_slave_status*`**，`:3341` / `:3452`）：

| 列 | 来源 |
|---|---|
| `Replica_IO_Running` | `mi->slave_running == MYSQL_SLAVE_RUN_CONNECT ? "Yes" : (…RUN_NOT_CONNECT ? "Connecting" : "No")` |
| `Last_IO_Errno` / `Last_IO_Error` | `mi->last_error().number` / `.message` |
| `Retrieved_Gtid_Set` / `Executed_Gtid_Set` | `io_gtid_set_buffer` / `sql_gtid_set_buffer` |
| `Auto_Position` | `mi->is_auto_position() ? 1 : 0` |
| `Channel_Name` | `mi->get_channel()`（默认 channel 是**空串**） |

**★★ `Seconds_Behind_Source` 怎么算的**（`sql/rpl_replica.cc:3574`，★ 没有独立函数，是内联逻辑）：

```cpp
    if ((mi->get_master_log_pos() == mi->rli->get_group_master_log_pos()) &&
        (!strcmp(mi->get_master_log_name(), mi->rli->get_group_master_log_name()))) {
      if (mi->slave_running == MYSQL_SLAVE_RUN_CONNECT) protocol->store(0LL);
      else protocol->store_null();                    /* ★ IO 线程没连 → NULL */
    } else {
      long time_diff = ((long)(time(nullptr) - mi->rli->last_master_timestamp) -
                        mi->clock_diff_with_master);
      protocol->store((longlong)(mi->rli->last_master_timestamp ? max(0L, time_diff) : 0));
    }
  } else protocol->store_null();                      /* ★ SQL 线程没跑 → NULL */
```

> ★ 公式：`(now - last_master_timestamp) - clock_diff_with_master`，再 `max(0, ·)`。
> ★ `last_master_timestamp == 0` 是"已追上"的**哨兵值**，直接输出 0。
> ★ `last_master_timestamp` = 事件的 `when`（源上语句开始时间）**+ `exec_time`**（源上执行耗时）= **事务在源上的提交时刻**，不是 relay log 落地时刻。MTS 下只在 GAQ 出队时更新，所以**并行复制的 Seconds_Behind 会跳变**。
> ★ 源码自己解释了为什么可能为 -1（时钟粒度）并因此加了 `max(0, ·)`。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 每个 CHANNEL 的连接/位点/延迟 | SQL | `SHOW REPLICA STATUS FOR CHANNEL 'ch1'`；`performance_schema.replication_connection_status` / `replication_applier_status_by_coordinator`（按 CHANNEL 分组） |
| 拓扑上有多少从库在拉 | SQL | 主库 `SHOW REPLICAS`（COM_REGISTER_SLAVE 注册过的） |
| failover 前确认追平 | SQL | 对比 `SHOW REPLICA STATUS` 的 `Retrieved_Gtid_Set` vs `Executed_Gtid_Set`（差集为空 = 已追平）；或 `SOURCE_LOG_FILE/POS` 与主库 `SHOW MASTER STATUS` 对齐 |
| 过滤生效范围 | SQL | `performance_schema.replication_applier_filters`（每个 CHANNEL 生效的过滤器） |
| 延迟复制的实际延迟 | SQL | `SHOW REPLICA STATUS` 的 `SQL_Delay`；`performance_schema.replication_applier_status_by_worker` 的 `APPLYING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP` |
| 复制相关错误 | 日志/SQL | error log 的 `ER_RPL_*` 系列；`Last_IO_Errno` / `Last_SQL_Errno`（速查表见 replica.md） |

**★ `SHOW REPLICAS` 的真身**（`sql/rpl_source.cc:222`，⚠️ 函数已改名但 SQLCOM 没改）：

```cpp
bool show_replicas(THD *thd) {
  field_list.push_back(new Item_return_int("Server_Id", 10, MYSQL_TYPE_LONG));
  field_list.push_back(new Item_empty_string("Host", HOSTNAME_LENGTH));
  if (opt_show_replica_auth_info) {
    field_list.push_back(new Item_empty_string("User", USERNAME_CHAR_LENGTH));
    field_list.push_back(new Item_empty_string("Password", 20));
  }
  field_list.push_back(new Item_return_int("Port", 7, MYSQL_TYPE_LONG));
  field_list.push_back(new Item_return_int("Source_Id", 10, MYSQL_TYPE_LONG));
  field_list.push_back(new Item_empty_string("Replica_UUID", UUID_LENGTH));

  // TODO: once the old syntax is removed, remove this as well.
  if (thd->lex->is_replication_deprecated_syntax_used())
    rename_fields_use_old_replica_source_terms(thd, field_list);
  ...
  mysql_mutex_lock(&LOCK_replica_list);
  for (const auto &key_and_value : slave_list) {
    REPLICA_INFO *si = key_and_value.second.get();
    protocol->store((uint32)si->server_id);
    protocol->store(si->host, &my_charset_bin);
    ...
    protocol->store((uint32)si->master_id);      /* ★ 字段名仍是 master_id */
```
> ★ 数据来自全局 hash **`slave_list`**（锁 `LOCK_replica_list`），由 dump 线程在 replica 注册时填充。
> ⚠️ 三点真名：① 函数真名 **`show_replicas()`**（旧的 `show_slave_hosts` 已不存在）；② `sql_command` 仍是 **`SQLCOM_SHOW_SLAVE_HOSTS`**；③ `User`/`Password` 两列**只有 `--show-replica-auth-info` 时才输出**。

**★ P_S 复制表与 `SHOW REPLICA STATUS` 同源**（`storage/perfschema/table_replication_connection_status.cc:279`）：

```cpp
int table_replication_connection_status::make_row(Master_info *mi) {
  m_row.channel_name_length = mi->get_channel() ? strlen(mi->get_channel()) : 0;
  ...
  if (mi->slave_running == MYSQL_SLAVE_RUN_CONNECT) {
    m_row.service_state = PS_RPL_CONNECT_SERVICE_STATE_YES;
  } else if (mi->slave_running == MYSQL_SLAVE_RUN_NOT_CONNECT) {
    m_row.service_state = PS_RPL_CONNECT_SERVICE_STATE_CONNECTING;
  } else {
    m_row.service_state = PS_RPL_CONNECT_SERVICE_STATE_NO;
  }
  ...
  m_row.count_received_heartbeats = mi->received_heartbeats;
  m_row.last_heartbeat_timestamp = (ulonglong)mi->last_heartbeat;
  {
    const Gtid_set *io_gtid_set = mi->rli->get_gtid_set();      /* ★ = Retrieved_Gtid_Set */
    m_row.received_transaction_set_length =
        io_gtid_set->to_string(&m_row.received_transaction_set);
  }
  m_row.last_error_number = (unsigned int)mi->last_error().number;
  ...
```

| P_S 列 | 来源 |
|---|---|
| `CHANNEL_NAME` | `mi->get_channel()` |
| `SOURCE_UUID` | `mi->master_uuid` |
| `THREAD_ID` | `PSI_THREAD_CALL(get_thread_internal_id)(mi->info_thd)` |
| `SERVICE_STATE` | `mi->slave_running` → ON/CONNECTING/OFF |
| `COUNT_RECEIVED_HEARTBEATS` / `LAST_HEARTBEAT_TIMESTAMP` | `mi->received_heartbeats` / `mi->last_heartbeat` |
| `RECEIVED_TRANSACTION_SET` | `mi->rli->get_gtid_set()`（同 Retrieved_Gtid_Set） |
| `LAST_ERROR_*` | `mi->last_error().number/message/skr` |
| `LAST_QUEUED_*` / `QUEUEING_*` | `Trx_monitoring_info::copy_to_ps_table()` |

**★ P_S 复制表文件清单**（`storage/perfschema/`）：

| 文件 | 表 |
|---|---|
| `table_replication_connection_configuration.cc` | `replication_connection_configuration` |
| `table_replication_connection_status.cc` | `replication_connection_status` |
| `table_replication_applier_configuration.cc` / `_status.cc` | `replication_applier_configuration` / `_status` |
| `table_replication_applier_status_by_coordinator.cc` / `_by_worker.cc` | MTS 协调者 / worker 级 |
| `table_replication_applier_filters.cc` / `_global_filters.cc` | per-channel / 全局过滤器 |
| `table_replication_asynchronous_connection_failover.cc` | failover 候选源列表 |
| `table_replication_group_members.cc` / `_member_stats.cc` | 组复制成员 |

> ⚠️ **`SHOW BINLOG EVENTS` / `SHOW RELAYLOG EVENTS` 没有改术语**——`Sql_cmd_show_binlog_events`（`sql_show.h:236`）、`mysql_show_binlog_events()`（`binlog.cc:3604`）、`mysql_show_relaylog_events()`（`rpl_rli.cc:1451`）。BINARY LOG STATUS 的改名只影响 `SHOW MASTER STATUS` 那一族。

---

## Misc

### 坑与已知缺陷

**双主自增冲突**：双主各自插入自增主键会撞号。GTID 不解决数据内容冲突。常见解法：`auto_increment_increment=2, auto_increment_offset=1/2`（两边奇偶错开），或应用层分片。注意：8.0 的 `auto_increment_offset` 在集群复制场景被部分禁用（MGR 默认接管），细节见 [`../../feat/auto_increment.md`](../../feat/auto_increment.md)。

**server_id / server_uuid 重复的静默灾难**：server_id 重复导致"跳自己"机制误杀别库事件（A 的事件被 server_id 相同的 B 跳过），uuid 重复则 GTID 命名空间撞车。克隆虚拟机做从库是最常见事故源——`server_id` 必查。

**从库被误写 → GTID 分叉**：读写分离下应用误写从库，从库的 `gtid_executed` 多出主库没有的事务 → 之后任何 failover/拓扑调整触发 `ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE`。防：`super_read_only=ON`。

**过滤器与 GTID 空洞**：过滤掉的事务 GTID 仍记入 executed（否则差集永远对不齐）→ 从库有"标记过但没数据"的事务。取消过滤不会补放。提升这样的从库为主库，数据就是缺失的——**生产上用过滤器的从库不该作为 failover 候选**，或必须接受空洞语义。

**环拓扑匿名事务无限循环**：`replicate_same_server_id=ON + log_replica_updates=ON + GTID 非 ON` 的组合在启动时被直接拒绝（`ER_RPL_INFINITY_DENIED`）。历史版本没有这层校验，社区出现过"环拓扑把 CPU 打满"的事故。

**★★ 源码证实的三个额外坑**：

**① 半同步会静默降级为异步**（`plugin/semisync/semisync_source.cc:702`，★ 源码注释自陈）：

```cpp
      /*
        ... Semi sync was not enabled when transaction entered into ordered_commit
        process. ... In this case trx_wait_binlog_name and trx_wait_binlog_pos are
        set but the 'Entry' object is not present. Hence dump thread will not wait
        for reply from slave ... In such case the committing transaction should
        not wait for an ack from slave and it should be considered as an async
        transaction.
      */
      if (!entry) { is_semi_sync_trans = false; goto l_end; }
```

> ★ 半同步在事务进入 `ordered_commit` **之后**才开启时，该事务**直接降级为异步**。所以半同步不是"绝对保证"。

**② `START REPLICA UNTIL` 与过滤器联动会留下"洞"**（`sql/rpl_replica_until_options.cc:122`）：

```cpp
    /*
      Master's events will be ignored in the cases that
      - It is in the ignore server id list.
      - Its server id is same to slave's sever id and replicate_same_server_id
        is not set.
      It will cause a hole in the replicated events.
      So we also need to check the position just before dispatching an event.
    */
    return check_position(m_current_log_name,
                          m_current_log_pos - ev->common_header->data_written);
```

> ★ 被防循环 / `IGNORE_SERVER_IDS` 过滤掉的事件在复制流里留下"洞"，所以 UNTIL 必须在派发前用 `log_pos - data_written` **补判一次**。这是"延迟复制 + 过滤"组合的经典坑。

**③ 旧协议 4 字节位点截断**（`sql-common/client.cc`）：

```cpp
    /*
      COM_BINLOG_DUMP accepts only 4 bytes for the position, so
      we are forced to cast to uint32.
    */
    int4store(ptr, (uint32)rpl->start_position);
```

> ★ 这也是为什么超过 4 GB 的 binlog 在旧协议下寻址能力受限。GTID 模式（`COM_BINLOG_DUMP_GTID`）用 8 字节修好了。

### ★★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| 从库侧主文件是 `sql/rpl_slave.cc` | **不存在**。是 `sql/rpl_replica.cc`（410 KB）；`rpl_slave*` 只在 `mysql-test/` |
| 连接函数改名 `connect_to_source()` | **不存在**。仍是 `connect_to_master()`（`rpl_replica.h:643`） |
| `mysql_binlog_send()` 在 `sql/binlog.cc` | **不在**（0 命中）。真身 `sql/rpl_source.cc:1054` |
| `COM_REGISTER_SLAVE` 已被删除/改名 | **仍存在且在线上发送**（`my_command.h:74`、`rpl_replica.cc:3303`）。只有**显示名**改成 `COM_REGISTER_REPLICA` |
| `Binlog_sender::read_gtids_from_packet()` | **不存在**。解码在 `com_binlog_dump_gtid()` 里调 `Gtid_set::add_gtid_encoding()` |
| `Binlog_sender::m_gtid_set` | **不存在**。真名 `m_exclude_gtid`（语义是**排除集**） |
| `init()` 判定 GTID 还是 file+pos | 判定在**构造函数**：`m_using_gtid_protocol(exclude_gtids != nullptr)`（`rpl_binlog_sender.cc:242`） |
| 半同步源码在 `sql/rpl_semi_sync*` | **不在**。在 **`plugin/semisync/`** |
| `rpl_semi_sync_master_wait_point` | **不存在**。真名 `rpl_semi_sync_source_wait_point`（但**类名仍是 `ReplSemiSyncMaster`**、方法仍叫 `readSlaveReply`） |
| 默认 channel 名叫 `"default"` | **错**。是**空串 `""`**（`rpl_msr.cc:37`） |
| `Master_info` 改名 `Source_info` | **错**。`rpl_mi.h:87` 仍是 `Master_info`；`LEX_MASTER_INFO`、`connect_to_master`、`show_slave_status*` 全未改 |
| `master_info_repository` 已移除 | **错**。仍存在（`sys_vars.cc:1714`），只是 `DEPRECATED_VAR("")` |
| `slave_parallel_workers` 是主名 | **错**。`replica_parallel_workers` 是主名（**默认 4**），旧名是 `Sys_var_deprecated_alias` |
| `START SLAVE` / `SHOW SLAVE STATUS` 已删 | **错**。仍可用，只发 deprecation warning |
| GTID_ONLY 是全局系统变量 | **错**。是 **per-channel** 的 `CHANGE ... GTID_ONLY=` 选项，存在 `Master_info::m_gtid_only_mode` |
| `replicate_same_server_id` 可 `SET GLOBAL` | **错**。只是 `my_getopt` 命令行选项（`mysqld.cc:9170`），`sys_vars.cc` 里 0 命中 |
| `replicate_do_db` 等是系统变量 | **错**。全是**命令行选项**，不能 `SET GLOBAL` |
| 存在 `is_event_from_self()` | **不存在**。是内联表达式 `s_id == ::server_id && !replicate_same_server_id`（3 处） |
| `Seconds_Behind_Source` 有专门计算函数 | **没有**。是 `show_slave_status_send_data()` 里的内联逻辑（`rpl_replica.cc:3574`） |
| `sql/rpl_msr.h` 不存在 | **存在**（确认真名） |
| 两种 dump 命令的 pos 一样宽 | **不一样**：旧 4 字节、新 8 字节 |
| `request_dump()` 有第三种分支 | **没有**。只有 GTID / file+pos 二选一 |
| 改名后源码里没有 `slave` 了 | **错**。C++ 核心层几乎全保留（见「容易误解的命名」） |
| `SHOW REPLICA STATUS` 约 57 列 | **60 列**（`show_slave_status_metadata` 精确 push 60 次，`rpl_replica.cc:3344-3431`） |
| `show_slave_status_send_data` 返回 bool 表错误 | **恒返回 false**（错误经上层 `goto err` 传播）；且是 `static`，仅本文件可见 |
| 存在 `Com_start_replica` / `Com_stop_replica` | **不存在** → 真名 **`Replica_start` / `Replica_stop`**（无 `Com_` 前缀），与 `Slave_start`/`Slave_stop` 共用同一 `com_stat[]` 槽 |
| 存在 `Com_show_replica_status` | **不存在** → 真名 **`Show_replica_status`**，与 `Show_slave_status` 共槽 |
| 存在全局 `source_connection_auto_failover` 变量 | **不存在** → per-channel 的 `Master_info::m_source_connection_auto_failover`（`rpl_mi.h:626`） |
| 存在 `opt_log_slave_updates` | **不存在** → 真名 **`opt_log_replica_updates`**（`mysqld.cc:1234`） |
| `MYSQL_RPL_*` 有一堆标志位 | **只有 2 个**：`MYSQL_RPL_GTID`(1<<16) / `MYSQL_RPL_SKIP_HEARTBEAT`(1<<17)，在 **`include/mysql.h:358`**（不在 `mysql_com.h`） |
| `terminology_use_previous` 控制所有输出 | **只控制 1 个命令名**（`COM_REGISTER_SLAVE`）+ P_S 的 mutex/线程名；**列名回退由 `is_replication_deprecated_syntax_used()` 按单条语句判定** |
| `Sys_var_deprecated_alias` 自己发 warning | **不发**。它只设 `option.comment` + 构造参数 `deprecated=true`，告警在基类 `Sys_var_alias` 路径 |
| `IGNORE_SERVER_IDS` 用线性查找 | 用 **`std::binary_search`**（`rpl_mi.cc:322`，故 `dynamic_ids` 必须有序） |
| `SHOW SLAVE HOSTS` 已删除 | **仍在**，走同一 `SQLCOM_SHOW_SLAVE_HOSTS`；函数真名 `show_replicas()` |
| `log_replica_updates=ON` 就一定写 binlog | `log_bin=OFF` 且未显式指定时会**隐式回退为 OFF**（`mysqld.cc:4940`） |
| 从库重连默认有限次 | **默认无限重试**（`rpl_replica.cc:8517`，注释：放弃重试会触发选主，故把触发权留给外部） |

### 容易误解的命名

| 名字 | 容易误解成 | 实际 |
|------|-----------|------|
| `log_replica_updates` | "是否记录从库的更新" | 是否把**回放的事务**写进本机 binlog（与从库自身的本地写入无关） |
| `replicate_same_server_id` | "允许多个从库同 id" | 是否回放 `server_id == 本机` 的事件（防环时保持 OFF） |
| CHANNEL | 网络通道 | 复制通道：一组独立的 (IO, SQL) 线程 + 位点 + relay log 的隔离单元 |
| auto position | 自动找位点 | 用 GTID 差集定位起点（协议仍是拉模型，从库主动发集合） |
| failover | 自动切换 | MySQL 社区版**没有内置自动 failover**——`SOURCE_CONNECTION_AUTO_FAILOVER` 只换连接不选主，选主由外部编排（Orchestrator / MGR / 云平台） |

**★★★ `Sys_var_deprecated_alias` 的官方设计说明**（`sql/sys_vars.h:474`，★ 这段注释把"怎么重命名变量"讲透了）：

```cpp
/**
  A deprecated alias for a variable.

  This tool allows us to rename system variables without breaking
  backward compatibility.

  Procedure for a developer to create a new name for a variable in
  version X and remove the old name in version X+1:

  - In version X:
    - Change the string passed to the Sys_var constructor for the
      variable the new new name.  All existing code for this should
      remain as it is.
    - Create a Sys_var_deprecated_alias taking the old name as the
      first argument and the Sys_var object having the new name as the
      second argument.
  - In version X+1:
    - Remove the Sys_var_deprecated_alias.

  This has the following effects in version X:

  - Both variables coexist. They are both visible in
    performance_schema tables and accessible in SET statements and
    SELECT @@variable statements. Both variables always have the same
    values.
  - A SET statement using either the old name or the new name changes
    the value of both variables.
  - A SET statement using the old name generates a deprecation
    warning.
  - The procedure that loads persisted variables from file accepts
    both names. ...
*/
class Sys_var_deprecated_alias : public Sys_var_alias {
 private:
  std::string m_comment;
 public:
  Sys_var_deprecated_alias(const char *name_arg, sys_var &base_var)
      : Sys_var_alias{name_arg, base_var, base_var.name.str, &base_var, true} {
    m_comment = std::string("This option is deprecated. Use ") +
                base_var.get_option()->name + " instead.";
    option.comment = m_comment.c_str();
  }
};
```

> ★★ 三个关键机制：① **第 5 个构造参数 `true`（deprecated）** 让 SET 老名自动告警；② `option.comment` 只负责在 `SHOW VARIABLES` / P_S 里显示 *"This option is deprecated. Use xxx instead."*；③ **两个变量共享同一存储**（`Sys_var_alias`），所以永远同值。
> ⚠️ 类自己**不发** warning，告警在基类路径。

**★ 三个具体注册实例**（★ 模式完全一致：先新后旧，一行 alias）：

```cpp
/* sql/sys_vars.cc:4291 */
static Sys_var_bool Sys_replica_preserve_commit_order(
    "replica_preserve_commit_order",
    "Force replication worker threads to commit in the same order as on the "
    "source. Enabled by default",
    PERSIST_AS_READONLY GLOBAL_VAR(opt_replica_preserve_commit_order),
    CMD_LINE(OPT_ARG, OPT_REPLICA_PRESERVE_COMMIT_ORDER), DEFAULT(true),
    NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(check_slave_stopped),   /* ★ ON_CHECK 仍叫 slave */
    ON_UPDATE(nullptr));

static Sys_var_deprecated_alias Sys_slave_preserve_commit_order(
    "slave_preserve_commit_order", Sys_replica_preserve_commit_order);
```
```cpp
/* sql/sys_vars.cc:6443 —— ★ 默认值确认为 4 */
static Sys_var_ulong Sys_replica_parallel_workers(
    "replica_parallel_workers",
    "Number of worker threads for executing events in parallel ",
    PERSIST_AS_READONLY GLOBAL_VAR(opt_mts_replica_parallel_workers),
    CMD_LINE(REQUIRED_ARG, OPT_REPLICA_PARALLEL_WORKERS),
    VALID_RANGE(0, MTS_MAX_WORKERS), DEFAULT(4), BLOCK_SIZE(1), NO_MUTEX_GUARD,
    NOT_IN_BINLOG, ON_CHECK(nullptr),
    ON_UPDATE(replica_parallel_workers_update));

static Sys_var_deprecated_alias Sys_slave_parallel_workers(
    "slave_parallel_workers", Sys_replica_parallel_workers);
```
```cpp
/* sql/sys_vars.cc:6301 —— 默认 REPLICA_NET_TIMEOUT（宏 = 60） */
static Sys_var_uint Sys_replica_net_timeout(
    "replica_net_timeout",
    "Number of seconds to wait for more data "
    "from a replication connection before aborting the read.",
    GLOBAL_VAR(replica_net_timeout), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1, LONG_TIMEOUT), DEFAULT(REPLICA_NET_TIMEOUT), BLOCK_SIZE(1),
    &PLock_replica_net_timeout, NOT_IN_BINLOG, ON_CHECK(nullptr),
    ON_UPDATE(fix_replica_net_timeout));

static Sys_var_deprecated_alias Sys_slave_net_timeout("slave_net_timeout",
                                                      Sys_replica_net_timeout);
```

**★★ 全部 25 组 `slave_*` → `replica_*` 别名**（`sql/sys_vars.cc`，逐个核实行号）：

| 行号 | 旧 → 新 | 行号 | 旧 → 新 |
|---|---|---|---|
| 2339 | `init_slave` → `init_replica` | 6348 | `slave_skip_errors` → `replica_skip_errors` |
| 2509 | `rpl_stop_slave_timeout` → `rpl_stop_replica_timeout` | 6386 | `slave_checkpoint_period` → `replica_checkpoint_period` |
| 2777 | `log_slow_slave_statements` → `log_slow_replica_statements` | 6401 | `slave_checkpoint_group` → `replica_checkpoint_group` |
| 2934 | `slave_max_allowed_packet` → `replica_max_allowed_packet` | 6420 | `sync_master_info` → **`sync_source_info`** |
| 4116 | `slave_compressed_protocol` → `replica_compressed_protocol` | 6440 | `slave_transaction_retries` → `replica_transaction_retries` |
| 4130 | `slave_exec_mode` → `replica_exec_mode` | 6452 | `slave_parallel_workers` → `replica_parallel_workers` |
| 4152 | `slave_type_conversions` → `replica_type_conversions` | 6464 | `slave_pending_jobs_size_max` → `replica_pending_jobs_size_max` |
| 4164 | `slave_sql_verify_checksum` → `replica_sql_verify_checksum` | 6638 | `pseudo_slave_mode` → `pseudo_replica_mode` |
| 4236 | `slave_parallel_type` → `replica_parallel_type` | 7673 | `skip_slave_start` → **`skip_replica_start`** |
| 4300 | `slave_preserve_commit_order` → `replica_preserve_commit_order` | 6160 | `log_slave_updates` → `log_replica_updates` |
| 4858 | `master_verify_checksum` → **`source_verify_checksum`** | 6245 | `slave_allow_batching` → `replica_allow_batching` |
| 6310 | `slave_net_timeout` → `replica_net_timeout` | 6256 | `slave_load_tmpdir` → `replica_load_tmpdir` |
| 6337 | `sql_slave_skip_counter` → **`sql_replica_skip_counter`** | | |

> ★ 注意其中 4 组**不是 slave→replica 而是 master→source**：`sync_master_info`→`sync_source_info`、`master_verify_checksum`→`source_verify_checksum`。以及 2 组是 `skip_slave_start`→`skip_replica_start`、`sql_slave_skip_counter`→`sql_replica_skip_counter`（前缀不同）。

**★★★ 老列名是怎么"变回来"的**（`sql/rpl_utility.cc:1386`，★ 完整映射表，实际 **32 项**不是 30）：

```cpp
// TODO: once the old syntax is removed, remove this as well.
static const std::unordered_map<std::string, std::string> deprecated_field_map{
    {"Replica_IO_State", "Slave_IO_State"},
    {"Source_Host", "Master_Host"},
    {"Source_User", "Master_User"},
    {"Source_Port", "Master_Port"},
    {"Source_Log_File", "Master_Log_File"},
    {"Read_Source_Log_Pos", "Read_Master_Log_Pos"},
    {"Relay_Source_Log_File", "Relay_Master_Log_File"},
    {"Replica_IO_Running", "Slave_IO_Running"},
    {"Replica_SQL_Running", "Slave_SQL_Running"},
    {"Exec_Source_Log_Pos", "Exec_Master_Log_Pos"},
    {"Source_SSL_Allowed", "Master_SSL_Allowed"},
    {"Source_SSL_CA_File", "Master_SSL_CA_File"},
    {"Source_SSL_CA_Path", "Master_SSL_CA_Path"},
    {"Source_SSL_Cert", "Master_SSL_Cert"},
    {"Source_SSL_Cipher", "Master_SSL_Cipher"},
    {"Source_SSL_Key", "Master_SSL_Key"},
    {"Seconds_Behind_Source", "Seconds_Behind_Master"},
    {"Source_SSL_Verify_Server_Cert", "Master_SSL_Verify_Server_Cert"},
    {"Source_Server_Id", "Master_Server_Id"},
    {"Source_UUID", "Master_UUID"},
    {"Source_Info_File", "Master_Info_File"},
    {"Replica_SQL_Running_State", "Slave_SQL_Running_State"},
    {"Source_Retry_Count", "Master_Retry_Count"},
    {"Source_Bind", "Master_Bind"},
    {"Source_SSL_Crl", "Master_SSL_Crl"},
    {"Source_SSL_Crlpath", "Master_SSL_Crlpath"},
    {"Source_TLS_Version", "Master_TLS_Version"},
    {"Source_public_key_path", "Master_public_key_path"},
    {"Get_Source_public_key", "Get_master_public_key"},
    {"Server_Id", "Server_id"},              /* ★ 下面 3 项是给 SHOW REPLICAS 用的 */
    {"Source_Id", "Master_id"},
    {"Replica_UUID", "Slave_UUID"}};

void rename_fields_use_old_replica_source_terms(
    THD *thd, mem_root_deque<Item *> &field_list) {
  for (auto &item : field_list) {
    std::string name{item->full_name()};
    auto itr = deprecated_field_map.find(name);
    if (itr != deprecated_field_map.end()) name = itr->second;
    item->rename(thd->mem_strdup(name.c_str()));
  }
}
```

**★ 触发条件不是全局开关，是"本条语句用了哪种语法"**（`rpl_replica.cc:3814`）：

```cpp
  show_slave_status_metadata(&field_list, max_io_gtid_set_size, sql_gtid_set_size);

  // TODO: once the old syntax is removed, remove this as well.
  if (thd->lex->is_replication_deprecated_syntax_used())
    rename_fields_use_old_replica_source_terms(thd, field_list);
```

> ★★ **这就是"同一台 8.0.39 上，`SHOW SLAVE STATUS` 显示 Master_Host、`SHOW REPLICA STATUS` 显示 Source_Host"的源码解释**——判定发生在**发送元数据之前**，按单条语句走。
> ⚠️ 所以 `terminology_use_previous` 并**不**控制这个（它只控制 1 个命令名 + P_S 里的 mutex/线程名），这是个常见误解。

**★ 语法层 `push_deprecated_warn` 的三个实例**（`sql_yacc.yy`，★ 模式完全一致）：

```cpp
/* :2581 —— 语句级 */
change_replication_source:
          MASTER_SYM
          {
            push_deprecated_warn(YYTHD, "CHANGE MASTER",
                                        "CHANGE REPLICATION SOURCE");
          }
        | REPLICATION SOURCE_SYM
        ;
```
```cpp
/* :2840 —— 选项级 */
change_replication_source_host:
          MASTER_HOST_SYM
          {
            push_deprecated_warn(YYTHD, "MASTER_HOST", "SOURCE_HOST");
          }
        | SOURCE_HOST_SYM
        ;
```
```cpp
/* :2849 */
change_replication_source_bind:
          MASTER_BIND_SYM
          {
            push_deprecated_warn(YYTHD, "MASTER_BIND", "SOURCE_BIND");
          }
        | SOURCE_BIND_SYM
        ;
```
> ★ 老 token 的 action 里发告警，**新 token 无 action**——所以 `sql_command` 仍是 `SQLCOM_CHANGE_MASTER`（`:2596`）。

**★ PSI 线程名已全改**（`sql/rpl_replica.cc:413`，★ 但数组名没有）：

```cpp
static PSI_thread_info all_slave_threads[] = {    /* ⚠️ 数组名仍是 all_slave_threads */
    {&key_thread_replica_io, "replica_io", "rpl_rca_io", PSI_FLAG_THREAD_SYSTEM, ...},
    {&key_thread_replica_sql, "replica_sql", "rpl_rca_sql", PSI_FLAG_THREAD_SYSTEM, ...},
    {&key_thread_replica_worker, "replica_worker", "rpl_rca_wkr", PSI_FLAG_THREAD_SYSTEM, ...},
    {&key_thread_replica_monitor_io, "replica_monitor", "rpl_rca_mon",
     PSI_FLAG_SINGLETON | PSI_FLAG_THREAD_SYSTEM, ...}};
```
> ★ `SHOW PROCESSLIST` / P_S 里看到的是 `replica_io` / `replica_sql` / `replica_worker` / `replica_monitor`。

**★★★ 8.0 术语改名的实测边界**（MASTER→SOURCE / SLAVE→REPLICA）——"改名"是**分层的、不彻底的**

**已改名**：

| 层 | 证据 |
|---|---|
| SQL 语句 | `CHANGE REPLICATION SOURCE TO` / `START REPLICA` / `SHOW REPLICA STATUS` / `SHOW REPLICAS` / `RESET REPLICA` |
| 28 个 SQL 选项 | `MASTER_HOST`→`SOURCE_HOST`（`sql_yacc.yy:2843`）、`MASTER_AUTO_POSITION`→`SOURCE_AUTO_POSITION`（`:2833`）… 逐个 `push_deprecated_warn` |
| 系统变量 | 20 个 `slave_*`→`replica_*`，**机制是 `Sys_var_deprecated_alias`**（`sys_vars.h:552`）：老名不是被删，而是注册成指向新变量的别名，SET 老名**自动产生 deprecation warning** |
| 源文件名 | `rpl_slave.cc`→**`rpl_replica.cc`**、`rpl_slave.h`→**`rpl_replica.h`** |
| 状态列名 | `Master_Host`→`Source_Host`、`Slave_IO_Running`→`Replica_IO_Running`、`Seconds_Behind_Master`→`Seconds_Behind_Source`（共 30 项映射表在 `sql/rpl_utility.cc:1390`） |
| 半同步变量 | `rpl_semi_sync_master`→`rpl_semi_sync_source`（编译期宏 `USE_OLD_SEMI_SYNC_TERMINOLOGY` 可切回） |

**★★ 没有改名（C++ 核心层几乎全保留旧名）**：

| 符号 | 位置 |
|---|---|
| `class Master_info` | `sql/rpl_mi.h:87` —— ⚠️ **不存在 `Source_info`** |
| `class Relay_log_info` | `sql/rpl_rli.h` |
| `struct LEX_MASTER_INFO` | `sql/sql_lex.h:350`（注释自嘲 *"would better be renamed to st_lex_replication_info"*） |
| `connect_to_master()` | `rpl_replica.h:643` —— ⚠️ **不存在 `connect_to_source()`** |
| `show_slave_status*()` | `rpl_replica.cc:3341/3452/3732` —— ⚠️ **不存在 `show_replica_status`** |
| `COM_REGISTER_SLAVE` 命令码 | `include/my_command.h:74`（只改显示名） |
| `SQLCOM_CHANGE_MASTER` | `sql_yacc.yy:2596` |
| 表 `mysql.slave_master_info` / `slave_relay_log_info` / `slave_worker_info` | `scripts/mysql_system_tables.sql` |

> ★★ **读 8.0.39 源码时不能以为 `slave` 全没了**——同一个函数里常常新旧术语混用（如 `rpl_replica.cc` 里既有 `replica_was_killed` 也有 `slave_sleep` / `io_slave_killed`）。

**★ 版本时间点**（`sql/sys_vars.cc:7676`，★ "8.0.26" 被硬编码进枚举值）：

```cpp
static const char *terminology_use_previous_names[] = {"NONE", "BEFORE_8_0_26", nullptr};
```

> ★ 改名发生在 **8.0.26**；8.0.25 及以前用老术语。`terminology_use_previous=BEFORE_8_0_26` 可让输出回退到老名——**但它自己也标了 `DEPRECATED_VAR("")`**。
> ★ 全仓**只有 1 个**命令码受它影响：`COM_REGISTER_SLAVE`（`sql_parse.h:145`，注释原话 *"Currently, there is only one such command"*）。

**★ 旧语法在 8.0.39 仍可用**，只发 warning（`START SLAVE` `:9317` / `STOP SLAVE` `:9299` / `SHOW SLAVE STATUS` `:13869` / `SHOW SLAVE HOSTS` `:13685` / `RESET SLAVE` `:14251` / `CHANGE MASTER TO` `:2586`）。

### 社区边界澄清

- **社区版没有内置自动 failover 编排**：选主、切换、恢复是外部工具（Orchestrator、MHA 历史项目）或 MGR（组复制提供成员管理与多数派选举，但那是另一套同步复制体系）。
- **`SOURCE_CONNECTION_AUTO_FAILOVER` 不是主库自动切换**：它只在**组复制**场景下让从库自动把连接切到新 PRIMARY，选主是 MGR 的 Paxos 做的。
- 别家分支的 failover 方案（如云厂商的"秒级切换"）基于自己的 Raft/仲裁层，不属于社区复制语义。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Chapter 19 Replication*（复制原理、拓扑、failover）
- *MySQL 8.0 Reference Manual → 19.4.8 Switching Sources During Failover*（failover 官方流程）
- *MySQL 8.0 Reference Manual → CHANGE REPLICATION SOURCE TO Statement*（参数全集与校验语义）
- WorkLog: [WL#7165](https://dev.mysql.com/worklog/task/?id=7165)（MTS 逻辑时钟，见 prpl.md）

**Bug 论坛**
- 环拓扑无限循环防护：`ER_RPL_INFINITY_DENIED`（mysqld.cc 启动校验）

**内核月报 / 技术文章**
- 腾讯云开发者社区《MySQL 主从延迟分析》系列（无主键表同步延迟案例；核心结论——ROW 模式 N 个行事件 × 无主键 O(N) 全表扫描定位 = O(N²) 放大、且 WRITESET 因无 writeset 回退 COMMIT_ORDER——见 prpl.md 与 replica.md 的观测入口）

**相关文档**
- 主库发送侧（dump 线程、半同步）见 [`binlog.md`](binlog.md)
- 从库接收/应用侧（IO/SQL 线程、relay log、位点持久化、recovery）见 [`replica.md`](replica.md)
- 并行回放（MTS）见 [`prpl.md`](prpl.md)
- GTID 集合算法与 exclude 集合协议见 [`gtid.md`](gtid.md)
