# MySQL 从库复制线程深度解析

> 基于 MySQL 8.0.39 源码，涵盖从库侧复制管线：IO thread 接收管线（dump 请求报文构造、事件入 relay log、sync 节流、空间反压）、relay log 的读写生命周期（append/rotate/purge）、SQL thread 应用管线（事件分发、位点推进、STS 重试）、三组位点体系与 crash-safe 持久化、relay log recovery、START/STOP REPLICA 的线程生命周期。
>
> **边界**：本篇讲**从库侧**——事件怎么从网络进 relay log、再从 relay log 进引擎。主库侧 dump 线程（`Binlog_sender`）见 [`binlog.md`](binlog.md)「读侧 dump 线程」；SQL thread 的 MTS 调度细节（GAQ/LWM/`schedule_next_event`/一致性模型）见 [`prpl.md`](prpl.md)；GTID 集合算法与 exclude 集合见 [`gtid.md`](gtid.md)；binlog 物理格式与写入见 [`binlog.md`](binlog.md)；崩溃后引擎侧恢复见 [`../../innodb/recovery.md`](../../innodb/recovery.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路：两条线程与 relay log 的接力](#主链路两条线程与-relay-log-的接力)
  - [位点体系：三组位点与事务组边界](#位点体系三组位点与事务组边界)
  - [IO thread：接收管线](#io-thread接收管线)
  - [relay log：读写生命周期](#relay-log读写生命周期)
  - [SQL thread：应用管线](#sql-thread应用管线)
  - [位点持久化：两套仓库与 MTS checkpoint](#位点持久化两套仓库与-mts-checkpoint)
  - [relay log recovery：崩溃后的从库复位](#relay-log-recovery崩溃后的从库复位)
  - [线程生命周期：START 与 STOP](#线程生命周期start-与-stop)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

从库侧的复制引擎由**两条线程 + 一条 relay log 流**组成：IO thread（receiver）负责连接主库、请求 binlog dump、把收到的事件原样写入 relay log；SQL thread（applier，MTS 模式下是 coordinator + worker 池）负责从 relay log 读出事件、反序列化、执行进存储引擎。relay log 是两者之间的中继缓冲——**它在字节格式上与 binlog 完全同构，但语义完全不同**：不参与 2PC 裁决、不进崩溃恢复的 XID 扫描、只服务于"接收进度与回放进度解耦"。

### 用途

解决两个问题：

1. **速度解耦**：主库写入与从库回放天然异速（主库并发写、从库可能要排队）。IO thread 把主库的 binlog 流量"削峰"成磁盘上的 relay log，SQL thread 按自己的节奏消费。没有 relay log，从库必须边收边执行，网络抖动直接卡住回放。
2. **崩溃可恢复**：回放进度（SQL 侧位点）持久化后，从库重启能从"最后一个完整事务之后"继续，不重不漏——这是 crash-safe replication 的核心。

在整条链路上的位置（与 [`binlog.md`](binlog.md) 的 dump 章节对称）：

```
主库:  事务提交 → binlog 文件 ──dump──→ 网络 ──┐
                                              │ COM_BINLOG_DUMP(_GTID) 事件字节流
从库:  存储引擎 ←─apply─ SQL thread ←─ relay log ←─ IO thread ────┘
              （执行）       （消费）        （暂存）         （接收）
```

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 引入 MTS（DATABASE 库级并行）与 `relay_log_info_repository=TABLE`；位点仍以 `master.info`/`relay-log.info` FILE 仓库为主 |
| 5.7 | GTID + `MASTER_AUTO_POSITION`；crash-safe 复制三件套（TABLE 仓库 + `relay_log_recovery`）成为标准配置；`LOGICAL_CLOCK` 并行 |
| 8.0 | 术语改名（slave→replica、master→source），但线程函数名 `handle_slave_io`/`handle_slave_sql` 保留旧词根；`GTID_ONLY` 通道；`replica_compressed_protocol` 压缩 |
| 8.0.18 / 5.7.28 | ★ Bug #92882 修复：MTS + GTID + AUTO_POSITION 下 relay log 丢失导致恢复失败的回归被修（细节见「relay log recovery」与「Misc」） |
| 8.0.22+ | heartbeat v2（携带日志名，修复旧 heartbeat 的跨文件歧义）；`source_connection_auto_failover`（异步连接故障转移）；applier 侧 change streams 基建 |

> 为什么这么变，写在「理论基础 → 他库对比与演进动机」。

---

## 理论基础

### 设计思想与权衡

#### 两条线程的分工：为什么不让一个线程既收又执行

**本可以那样做**：一个线程收一个事件执行一个事件（5.6 之前的单线程复制正是这个模型）。**为什么没有**：网络读取的节奏由主库决定（可能突发几千个事件），而执行的节奏由本地引擎决定（锁等待、磁盘写入），两者叠加会让慢的一侧拖死快的一侧——单线程模型下主库一次大事务提交，从库网络缓冲打满、回放却卡在行锁上，整个链路停摆。

两条线程 + relay log 的本质是**生产者-消费者的经典解耦**：IO thread 只做"字节搬运"（收包 → 校验 checksum → 追加 relay log → 更新接收位点），SQL thread 只做"语义执行"（读事件 → 反序列化 → apply）。代价是：

- 从库延迟被拆成两段（`Relay_Log_Pos` 与 `Read_Source_Log_Pos` 的距离 = IO 落后；`Exec_Source_Log_Pos` 与 `Read_Source_Log_Pos` 的距离 = SQL 落后），故障定位必须分清是哪一段
- relay log 占一份磁盘空间（`relay_log_space_limit` 的权衡见「relay log」节）

#### relay log 为什么与 binlog 字节同构，语义却不同

relay log 复用 `MYSQL_BIN_LOG` 类（`Relay_log` 是其子类），事件**不翻译、不重编码**——IO thread 收到什么字节就写什么字节（fake Rotate 与 FD 的 checksum 适配除外）。这带来两个结果：

1. **省掉一条序列化路径**：dump 线程发送的字节流本来就是 binlog 格式，从库原样落盘即可被 applier 用同一套 `binlog_event_deserialize` 解析
2. **语义切割必须显式**：同一个类，relay log 实例与 binlog 实例的差异靠 `is_relay_log` 标志区分——它决定不参与 2PC 裁决（崩溃恢复的 XID 扫描只看 binlog）、rotate 时机不同、checksum 算法来源不同

#### 位点的 crash-safe 模型：为什么位点要卡在"事务组边界"

SQL thread 位点不是每执行一个事件就持久化一次，而是**只在事务组边界**（XID / 已提交 DDL / 事务性语句结束）才允许推进 `group_*` 位点并落盘。这是 crash-safe 的根基：恢复时从"最后一个完整事务之后"开始，半截事务（已执行了 BEGIN + 一半行事件但没执行 XID 的）会被整体重放——事务要么整体生效要么整体不生效，正好与 InnoDB 的事务语义对齐。

**代价与退化**：一个巨大的单事务（几千万行）执行到一半崩溃，重启后**整个事务从头重放**——持久化粒度是事务而非语句。这是"位点简单性"换"恢复放大"的权衡：如果按语句粒度记录位点，恢复逻辑要处理"事务中间语句已提交"的复杂状态，复杂度爆炸。

#### GTID 对位点模型的补丁

非 GTID 模式下，"从哪继续"是 file+pos 的二元组，跨文件、跨主从拓扑变化时脆弱。GTID 把位点换成"已执行事务集合"：`recover_relay_log` 在 GTID 模式下清空 Retrieved_Gtid_Set，让 IO thread 重连时用 `executed ∪ retrieved` 作为 exclude 集合（见 [`gtid.md`](gtid.md)），主库自动补发缺失事务——**位点问题降维成集合问题**。这是 Bug #92882 修复（8.0.18）的设计基础：AUTO_POSITION 下 relay log 丢失不再是灾难，因为 GTID 集合能重建缺失的事务流。

### 理论溯源

- **两线程 + 中继缓冲**是操作系统课程级的生产者-消费者模型在数据库复制中的实例化；relay log 的"中继"角色与 TCP 接收缓冲、消息队列中间件同构——核心都是"用磁盘空间换速度解耦"
- **crash-safe 位点**与 binlog 的 2PC 恢复共享同一原理：崩溃后需要的是"可判定的重放起点"。binlog 侧用 XID 与 prepared 事务的匹配判定（见 [`binlog.md`](binlog.md)「2PC 裁决」），从库侧用"事务组边界位点"判定——两者都是**把不可判定的中间态排除出持久化状态**（WL#11262 crash-safe replication 的设计主线）
- **MTS 的 gap 恢复**（`UNTIL SQL_AFTER_MTS_GAPS`）是一个"先补齐因果洞再继续"的自愈算法：把 worker 执行进度与 coordinator 位点之间的差距建模为 GAQ 上的 LWM（见 [`prpl.md`](prpl.md)），重启后强制串行回放到洞填满

### 算法与数据结构

- **位点五元组**（详见「位点体系」）：`master_log_name/pos`（IO 侧）、`group_master_log_name/pos` + `group_relay_log_name/pos`（SQL 侧 crash-safe 点）、`event_relay_log_pos`/`future_event_relay_log_pos`（读游标）。三组的推进时机与持久化策略各不相同，是理解从库全部状态的核心
- **`Transaction_boundary_parser`**：轻量状态机，吃进事件序列、输出"当前是否在事务中"。IO 侧用它保证"GTID 只在事务最后事件落盘后加入 Retrieved_Gtid_Set"（半截事务的 GTID 不算收到）与"不在事务中间 rotate relay log"；SQL 侧用它做事务边界跟踪
- **`Rpl_applier_reader`**：relay log 的读取器封装，职责是"按 `group_relay_log_pos` 打开、读事件、跨文件、读完 purge、没有事件时等待"——把 SQL thread 主循环从文件操作细节中解放出来

### 他库对比与演进动机

- **PostgreSQL 流复制**：没有 relay log 概念。PG 复制的是物理 WAL 字节流，standby 直接回放 WAL（或 `pg_receivewal` 落成 WAL 文件），字节本身就可用，不需要"中继缓冲 + 逻辑执行"两段。MySQL 的 binlog 是逻辑事件流，从库必须"反序列化 → 执行进引擎"（行事件还要用 before-image 定位行），执行成本远高于"字节落盘"，所以必须有 relay log 解耦接收与执行
- **MariaDB**：保留 relay log 模型，但增加了 `rpl_semi_sync_master` 之外的更多并行策略与优化；其并行复制演进（`slave_parallel_threads`、conservative/optimistic 模式）与 MySQL 的 LOGICAL_CLOCK 同源不同路
- **演进动机**：5.6 之前从库单线程是复制延迟的主因 → 5.6 MTS（库级）→ 5.7 LOGICAL_CLOCK（组提交级）→ 8.0 WRITESET（行级），这条线在 [`prpl.md`](prpl.md) 详述；本篇关注的是与之配套的**基建演进**：TABLE 仓库与 `relay_log_recovery`（crash-safe）、GTID（位点降维）、heartbeat v2（跨文件心跳歧义修复）——每一项都是为了把"复制链路"从"能跑"推到"崩溃也不丢不重"

---

## 核心实现

### 主链路：两条线程与 relay log 的接力

```
                        handle_slave_io                      handle_slave_sql
                        (IO thread)                          (SQL thread / coordinator)
                              │                                      │
                              │ safe_connect / register_slave        │ applier_reader.open(group_*位点)
                              ▼                                      ▼
                        request_dump ── COM_BINLOG_DUMP_GTID ──► 主库 dump 线程
                              │                                      │
                              │ read_event（收网络包）                 │ read_next_event（读 relay log）
                              ▼                                      ▼
                        queue_event ── 写 relay log ──────────►  exec_relay_log_event
                              │  （append + sync + rotate）             │  STS: apply_event
                              │  （GTID 事务末事件 → Retrieved 集）      │  MTS: 分发给 worker
                              ▼                                      ▼
                        位点: mi->master_log_pos++              位点: group_*（事务边界）
                                                                      │
                                                              flush_info（落 repository）
```

两条线程通过 **relay log 文件 + 内存条件变量**协作：IO thread 写完事件后 `update_binlog_end_pos` 并广播 `COND_binlog_updated`；SQL thread 的 `Rpl_applier_reader::wait_for_new_event` 在读到文件尾时睡在该条件变量上。SQL thread purge 掉已回放的 relay log 后广播 `log_space_cond`，唤醒可能正在等待空间的 IO thread。

### 位点体系：三组位点与事务组边界

理解从库状态的全部钥匙是三组位点，它们的**主语不同、推进时机不同、持久化策略不同**：

| 位点 | 主语 | 语义 | 推进时机 | 持久化 |
|------|------|------|---------|--------|
| `mi->master_log_name/pos` | IO thread | 已从主库收到**并写完 relay log** 的位置 | `queue_event` 每事件 `+inc_pos` | `Master_info::flush_info`（`sync_masterinfo_period` 节流） |
| `rli->group_master_log_name/pos` + `group_relay_log_name/pos` | SQL thread | 已完成回放的**事务组边界**（crash-safe 点） | STS：事务提交时；MTS：仅 GAQ checkpoint 时 | `Relay_log_info::flush_info`（`sync_relayloginfo_period` 节流） |
| `event_relay_log_pos` / `future_event_relay_log_pos` | SQL thread 读游标 | 当前正在读 / 下一个事件的位置 | `Rpl_applier_reader::read_next_event` 每次读事件时 | **不持久化**（重启后由 group_* 重建） |

关键区分（`Master_info` 与 `Relay_log_info` 成员，均在 rpl_mi.h / rpl_rli.h）：

```cpp
// Master_info —— IO thread 私有位点
char master_log_name[FN_REFLEN];
my_off_t master_log_pos;

// Relay_log_info —— SQL thread 位点
// "Event group means a group of events of a transaction. group_relay_log_name
//  and group_relay_log_pos record the place before where all event groups are
//  applied. When slave starts, it resume to apply events from
//  group_relay_log_pos. ... For MTS, group_relay_log_pos is updated by mts
//  checkpoint mechanism."
char group_relay_log_name[FN_REFLEN];
ulonglong group_relay_log_pos;
char event_relay_log_name[FN_REFLEN];
ulonglong event_relay_log_pos;
ulonglong future_event_relay_log_pos;
char group_master_log_name[FN_REFLEN];
volatile my_off_t group_master_log_pos;
```

读游标的推进方式是"影子"式的：

```cpp
// rpl_applier_reader.cc —— 每读一个事件，游标前移
m_rli->set_event_start_pos(m_relaylog_file_reader.position());
ev = m_relaylog_file_reader.read_event_object();
if (ev != nullptr) {
  m_rli->set_future_event_relay_log_pos(m_relaylog_file_reader.position());
  ev->future_event_relay_log_pos = m_rli->get_future_event_relay_log_pos();
}

// rpl_rli.h
inline void inc_event_relay_log_pos() {
  event_relay_log_pos = future_event_relay_log_pos;
}
```

**为什么读游标不持久化**：重启后 SQL thread 从 `group_relay_log_pos` 重新打开，读游标天然重建——持久化它只会引入"游标与 group 位点不一致"的新故障面。

`group_master_log_name/pos` 与 `group_relay_log_name/pos` 是**同一个事务组在两个坐标系下的投影**：前者是"这个事务在主库 binlog 的位置"（用于 `SHOW REPLICA STATUS` 的 `Exec_Source_Log_Pos` 与 `Seconds_Behind_Source` 计算），后者是"这个事务在 relay log 的位置"（用于重启后重新打开 relay log）。两者都由 SQL thread 在事务边界同步推进，MTS 下都由 GAQ checkpoint（LWM）推进——**单线程 owner 原则**（rpl_rli.h 注释：group_master_log_pos 只允许 SQL thread 写）。

### IO thread：接收管线

#### 主循环骨架

`handle_slave_io`（rpl_replica.cc）是一个"连接 → dump → 读事件 → 入队"的循环，外层套重试：

```
init  →  safe_connect → get_master_version_and_clock → get_master_uuid
      →  io_thread_init_commands（SET 会话变量）
      →  register_slave_on_master（COM_REGISTER_SLAVE，登记 server_id/host/port）
      →  ┌─ request_dump（COM_BINLOG_DUMP / COM_BINLOG_DUMP_GTID）
         │   └─ 循环: read_event（网络收包）
         │        → queue_event（写 relay log + 位点推进 + flush master info）
         └─ 网络错误/主库重启 → try_to_reconnect → 回 connected 标签
```

`try_to_reconnect` 是重试的核心（rpl_replica.cc:5273 附近）：

```cpp
static int try_to_reconnect(THD *thd, MYSQL *mysql, Master_info *mi,
                            uint *retry_count, bool suppress_warnings,
                            const Reconnect_messages &messages) {
  mi->slave_running = MYSQL_SLAVE_RUN_NOT_CONNECT;
  ...
  thd->clear_active_vio();
  end_server(mysql);
  if ((*retry_count)++) {
    if (*retry_count > mi->retry_count) return 1;  // 重试上限（SOURCE_RETRY_COUNT）
    slave_sleep(thd, mi->connect_retry, io_slave_killed, mi);  // SOURCE_CONNECT_RETRY 秒
  }
  ...
  if (safe_reconnect(thd, mysql, mi, true) || io_slave_killed(thd, mi)) return 1;
  return 0;
}
```

注意重试语义：`retry_count` 在成功收到一个事件后**归零**（handle_slave_io 主循环内），所以"重试上限"针对的是**连续失败**，不是累计失败——偶发抖动不会累积成致命错误。

#### dump 请求报文：Retrieved ∪ Executed

`request_dump`（rpl_replica.cc:4157 附近）按是否 auto_position 选择命令，GTID 模式下组装 exclude 集合：

```cpp
enum_server_command command =
    mi->is_auto_position() ? COM_BINLOG_DUMP_GTID : COM_BINLOG_DUMP;

Gtid_set gtid_executed(&sid_map);
if (command == COM_BINLOG_DUMP_GTID) {
  // ① Retrieved_Gtid_Set：已完整落盘进 relay log 的事务
  mi->rli->get_sid_lock()->wrlock();
  gtid_executed.add_gtid_set(mi->rli->get_gtid_set());
  mi->rli->get_sid_lock()->unlock();

  // ② executed_gtids：本机已执行的事务
  global_sid_lock->wrlock();
  gtid_executed.add_gtid_set(gtid_state->get_executed_gtids());
  global_sid_lock->unlock();

  rpl->file_name = nullptr;
  rpl->start_position = 4;
  rpl->flags |= MYSQL_RPL_GTID;
  rpl->gtid_set_encoded_size = gtid_executed.get_encoded_length();
  rpl->fix_gtid_set = fix_gtid_set;   // 回调：gtid_set->encode(packet)
  rpl->gtid_set_arg = (void *)&gtid_executed;
} else {
  rpl->file_name = mi->get_master_log_name();
  rpl->start_position = mi->get_master_log_pos();
}
if (mysql_binlog_open(mysql, rpl)) { ... }  // 客户端库组装报文并发出
```

**两个容易误解的点**：

1. **集合 = Retrieved ∪ Executed，不是"已发送过"的集合**。Retrieved（`rli->get_gtid_set()`）包含"已完整收到进 relay log 但可能还没回放"的事务——这些不能丢，否则重连后主库不会再发（主库认为从库已有了），但从库还没执行。Executed 是"已经执行完的"。两者并集才是"主库可以跳过"的完整集合（与 [`gtid.md`](gtid.md) 的 exclude 集合语义一致）
2. **非 GTID 模式的位点是 IO 侧的 `master_log_pos`**，不是 SQL 侧的 `group_master_log_pos`——即"已收到"而不是"已回放"。所以非 GTID 模式下 `relay_log_recovery` 显得尤为重要（见「relay log recovery」）

#### queue_event：事件入队

`queue_event`（rpl_replica.cc:7602 附近）是 IO thread 最重的函数，一段完整的状态机。骨架：

```cpp
QUEUE_EVENT_RESULT queue_event(Master_info *mi, const char *buf,
                               ulong event_len, bool do_flush_mi) {
  // ① checksum 校验（网络传输损坏在此拦截）
  if (Log_event_footer::event_checksum_test(
          const_cast<uchar *>(pointer_cast<const uchar *>(buf)), event_len,
          checksum_alg)) {
    mi->report(ERROR_LEVEL, ER_NETWORK_READ_EVENT_CHECKSUM_FAILURE, ...);
    goto err;
  }

  // ② 拿 LOCK_log：写入与 rotate 的互斥
  // "From now, and up to finishing queuing the event, no other thread is
  //  allowed to write to the relay log, or to rotate it."
  mysql_mutex_lock(log_lock);

  // ③ 事务边界 parser 喂事件（半截事务的 GTID 不算收到；不在事务中 rotate）
  std::tie(info_error, log_event_info) = extract_log_event_basic_info(
      buf, event_len, mi->get_mi_description_event());
  if (info_error || mi->transaction_parser.feed_event(log_event_info, true)) { ... }

  // ④ 按事件类型分支
  switch (event_type) {
    case STOP_EVENT:              do_flush_mi = false; goto end;  // 不写
    case ROTATE_EVENT:            process_io_rotate(mi, &rev); break;
    case FORMAT_DESCRIPTION_EVENT: 反序列化并保存 mi_description_event; break;
    case PREVIOUS_GTIDS_LOG_EVENT: 改写成指向 master 位点的 rotate; break;
    case HEARTBEAT_LOG_EVENT:     更新 master 位点，不写正文; break;
    case GTID_LOG_EVENT:          解析 gtid/timestamp，inc_pos = event_len; break;
    ...
  }

  // ⑤ server_id 过滤（本服务器事件 / IGNORE_SERVER_IDS）
  s_id = uint4korr(buf + SERVER_ID_OFFSET);
  if ((s_id == ::server_id && !replicate_same_server_id) ||
      (mi->shall_ignore_server_id(s_id) && ...)) {
    // 不写 relay log，但推进 master_log_pos（否则重启后重读这些事件）
  } else {
    // ⑥ 真正写 relay log
    if (likely(rli->relay_log.write_buffer(buf, event_len, mi) == 0)) {
      mi->set_master_log_pos(mi->get_master_log_pos() + inc_pos);
      ...
    }
  }

  // ⑦ 刷新 master info（节流在 flush_info 内）
  if (res == QUEUE_EVENT_OK && do_flush_mi)
    if (flush_master_info(mi, false, lock_count == 0, false, ...))
      res = QUEUE_EVENT_ERROR_FLUSHING_INFO;
}
```

事件类型分支揭示了几处**relay log 与 binlog 的语义切割**：

- **`PREVIOUS_GTIDS_LOG_EVENT` 不原样落盘**，改写成"指向当前 master 位点的 rotate"（`write_rotate_to_master_pos_into_relay_log`）——PREVIOUS_GTIDS 是主库 binlog 文件头的历史集合快照，对从库回放无意义（从库有自己的 executed 集合），保留只会干扰 relay log 的位点推进。改写后的事件在 relay log 里是一个"同步点"
- **`STOP_EVENT` 直接丢弃**（主库干净关机的信号，语义已由随后的连接断开表达）
- **FD event 要被保存**（`mi->set_mi_description_event`）——relay log rotate 时要把主库的 FD 重新写进新 relay log 头部（见「relay log」节），这样 relay log 才与主库格式自洽

#### 空间反压：relay_log_space_limit

IO 主循环在入队前检查空间（只在事务外等待——不能在事务中间让 SQL thread 偷走文件）：

```cpp
// handle_slave_io 主循环内
std::size_t queued_size = event_len;
if (is_any_gtid_event(...)) {
  // GTID 事件用整个事务长度估算（trx_length），避免事务中途超限
  Gtid_log_event gtid_ev(event_buf, mi->get_mi_description_event());
  queued_size = gtid_ev.get_trx_length();
}
if (rli->log_space_limit && exceeds_relay_log_limit(rli, queued_size) &&
    !mi->transaction_parser.is_inside_transaction()) {
  if (wait_for_relay_log_space(rli, queued_size)) goto err;
}
```

`wait_for_relay_log_space`（rpl_replica.cc:3103 附近）的协议是"先 rotate 挤出旧文件、再睡在 `log_space_cond` 上等 SQL 侧 purge"：

```cpp
static bool wait_for_relay_log_space(Relay_log_info *rli, size_t queued_size) {
  // 标记：此后 coordinator 在事务外 rotate 时，会 purge 掉旧文件
  rli->is_receiver_waiting_for_rl_space.store(true);
  ...
  rotate_relay_log(mi, true, true, true);   // 先主动 rotate 一次，让旧文件可被 purge
  ...
  mysql_mutex_lock(&rli->log_space_lock);
  thd->ENTER_COND(&rli->log_space_cond, &rli->log_space_lock, ...);
  while (exceeds_relay_log_limit(rli, queued_size) &&
         !(slave_killed = io_slave_killed(thd, mi)) &&
         rli->coordinator_log_after_purge != receiver_log) {
    mysql_cond_wait(&rli->log_space_cond, &rli->log_space_lock);
  }
  mysql_mutex_unlock(&rli->log_space_lock);
  ...
}
```

唤醒条件有三条：空间够了 / 被 kill / `coordinator_log_after_purge` 前进了（SQL 侧 purge 了文件并在 `log_space_lock` 下更新了该标记后广播）。第三条是防止"SQL 侧 purge 完成但空间依然不够"时的死等。

### relay log：读写生命周期

#### 写侧：write_buffer 与 after_write_to_relay_log

`MYSQL_BIN_LOG::write_buffer`（binlog.cc:7008 附近）是追加的入口：

```cpp
bool MYSQL_BIN_LOG::write_buffer(const char *buf, uint len, Master_info *mi) {
  assert(is_relay_log);
  mysql_mutex_assert_owner(&LOCK_log);
  bool error = false;
  if (m_binlog_file->write(pointer_cast<const uchar *>(buf), len) == 0) {
    bytes_written += len;
    error = after_write_to_relay_log(mi);
  } else {
    // 写失败：截断回写之前的位置，IO thread 稍后重连重发
    truncate_relaylog_file(mi, atomic_binlog_end_pos);
    error = true;
  }
}
```

写失败截断是 relay log 特有语义：binlog 写失败按 `binlog_error_action` 处理（可能 abort），relay log 写失败只需回退位点等重连——**因为 relay log 是缓存不是真相**，主库可以重发。

`after_write_to_relay_log`（binlog.cc:6853 附近）做四件事，顺序即语义：

```cpp
bool can_rotate = mi->transaction_parser.is_not_inside_transaction();  // 不在事务中

// ① flush + sync（sync_relay_log 节流在 sync_binlog_file 里）
bool error = flush_and_sync(false);

if (!error) {
  if (can_rotate) {
    // ② 事务最后一个事件已落盘 → GTID 加入 Retrieved_Gtid_Set
    mysql_mutex_lock(&mi->data_lock);
    const Gtid *last_gtid_queued = mi->get_queueing_trx_gtid();
    if (!last_gtid_queued->is_empty()) {
      mi->rli->add_logged_gtid(last_gtid_queued->sidno, last_gtid_queued->gno);
    }
    if (mi->is_queueing_trx()) mi->finished_queueing();
    mysql_mutex_unlock(&mi->data_lock);

    // ③ 按大小 / 显式请求 rotate
    if (m_binlog_file->get_real_file_size() > max_size ||
        mi->is_rotate_requested()) {
      error = new_file_without_locking(mi->get_mi_description_event());
      mi->clear_rotate_requests();
    }
  }
}

// ④ 更新 end_pos 并广播，唤醒 SQL thread
lock_binlog_end_pos();
mi->rli->ign_master_log_name_end[0] = 0;
update_binlog_end_pos(false);
harvest_bytes_written(mi->rli, true);
unlock_binlog_end_pos();
```

**② 是 Retrieved_Gtid_Set 的入集时机**：不是收到 GTID 事件就加入，而是**事务最后一个事件 flush 之后**。这保证 `request_dump` 发出去的集合里永远不含半截事务——若 IO thread 在事务中途断开，半截事务的 GTID 不在 Retrieved 集合中，重连后主库会从该事务开头重发。

**③ 的 rotate 条件**：`max_relay_log_size` 的检查在**事务外**才生效（`can_rotate`），事务内绝不 rotate——否则事务的事件跨两个 relay log 文件，recovery 时要跨文件拼事务，复杂度陡增。

#### sync 节流：sync_relay_log

relay log 的 sync 与 binlog 共用 `sync_binlog_file`，只是周期变量不同（`Relay_log` 构造时绑定 `&sync_relaylog_period`）：

```cpp
std::pair<bool, bool> MYSQL_BIN_LOG::sync_binlog_file(bool force) {
  unsigned int sync_period = get_sync_period();
  if (force || (sync_period && ++sync_counter >= sync_period)) {
    sync_counter = 0;
    ...
    m_binlog_file->sync();
  }
}
```

`sync_relay_log` 默认 10000（每 10000 个事件 fsync 一次）。与 binlog 的 `sync_binlog` 语义同构：**relay log 丢数据不是灾难**（GTID 下可重拉、非 GTID 下靠 `relay_log_recovery` 从主库补），所以默认值远大于 `sync_binlog=1` 的推荐配置。

#### rotate：relay log 的文件切换

两条 rotate 路径：

1. **被动**（收到主库的 Rotate event）：`process_io_rotate` → `rotate_relay_log`，并把 `mi->master_log_name/pos` 更新为 rotate 携带的新文件名与位点——IO 侧位点跨文件
2. **主动**（大小超限 / `flush_relay_logs` / 空间反压）：`new_file_without_locking`，写一个 Rotate event 到旧文件尾再开新文件

relay log rotate 的独特之处在 `new_file_impl`：开新文件时把**主库的 FD** 写进新文件头（`open_binlog(..., extra_description_event)` 传 `mi->get_mi_description_event()`）。这与 binlog rotate 写自己的 FD 不同——relay log 记录的是主库的格式描述，从库自己的 FD 只在 relay log 被初始创建时写。这也是 `queue_event` 要保存主库 FD 的原因。

#### 读侧：Rpl_applier_reader

`Rpl_applier_reader`（rpl_applier_reader.cc）把 relay log 的读侧封装成 SQL thread 的输入流：

- `open()`：从 `group_relay_log_pos` 打开当前 relay log，初始化 `rli_description_event`
- `read_next_event()`：读事件 + 推进 `event_start_pos`/`future_event_relay_log_pos` 读游标
- `move_to_next_log()`：读到文件尾且还有下一个文件时切换（模拟 binlog 的 Rotate）
- `wait_for_new_event()`：读到活跃文件尾时，睡在 `COND_binlog_updated` 上等 IO thread 写入
- `purge_applied_logs()`：SQL 回放推进后，purge 掉已回放完的旧 relay log 文件，更新 `log_space_total`，广播 `log_space_cond` 唤醒等待空间的 IO thread

purge 的判定是**文件粒度**（不能 purge 当前正在读的文件），且更新 `coordinator_log_after_purge` 标记——这正是「空间反压」节中 IO thread 的唤醒条件之一。

### SQL thread：应用管线

#### 主循环与事件分发

`handle_slave_sql`（rpl_replica.cc:6966 附近）初始化 worker 池后进入主循环：

```cpp
while (!main_loop_error && !sql_slave_killed(thd, rli)) {
  Log_event *ev = nullptr;
  // 读下一个事件（Rpl_applier_reader 内部处理跨文件与等待）
  mysql_mutex_lock(&rli->data_lock);
  ev = applier_reader.read_next_event();
  mysql_mutex_unlock(&rli->data_lock);

  // MTS：调度器设置事件上下文
  if (ev != nullptr && rli->is_parallel_exec() && rli->current_mts_submode != nullptr)
    rli->current_mts_submode->set_multi_threaded_applier_context(*rli, *ev);

  // 分发
  switch (exec_relay_log_event(thd, rli, &applier_reader, ev)) {
    case SLAVE_APPLY_EVENT_AND_UPDATE_POS_OK:      // 成功，读下一个
    case SLAVE_APPLY_EVENT_UNTIL_REACHED:          // UNTIL 条件达成
    case SLAVE_APPLY_EVENT_RETRY:                  // STS 临时错误，重读同一事件
      break;
    case SLAVE_APPLY_EVENT_AND_UPDATE_POS_APPLY_ERROR:
    case SLAVE_APPLY_EVENT_AND_UPDATE_POS_UPDATE_POS_ERROR:
    case SLAVE_APPLY_EVENT_AND_UPDATE_POS_APPEND_JOB_ERROR:
      main_loop_error = true;                      // 致命，退出
      break;
    default: assert(0);
  }
}
```

返回值枚举（rpl_replica.cc:289 附近）是主循环的协议语言：

```cpp
enum enum_slave_apply_event_and_update_pos_retval {
  SLAVE_APPLY_EVENT_AND_UPDATE_POS_OK = 0,            // 应用成功且位点已推进
  SLAVE_APPLY_EVENT_AND_UPDATE_POS_APPLY_ERROR = 1,   // apply 失败（致命）
  SLAVE_APPLY_EVENT_AND_UPDATE_POS_UPDATE_POS_ERROR = 2,  // 位点推进失败（致命）
  SLAVE_APPLY_EVENT_AND_UPDATE_POS_APPEND_JOB_ERROR = 3,  // MTS 入队失败（致命）
  SLAVE_APPLY_EVENT_RETRY = 4,                        // STS 可重试
  SLAVE_APPLY_EVENT_UNTIL_REACHED = 5,                // 优雅停
};
```

#### exec_relay_log_event 内的关键分支

`exec_relay_log_event`（rpl_replica.cc:4892 附近）：

1. **checkpoint 检查**：`force || rli->is_time_for_mta_checkpoint()` 时先跑 `mta_checkpoint_routine`（MTS 的 LWM 推进，见「位点持久化」）
2. **UNTIL 条件**：`is_until_satisfied_before_dispatching_event` 命中则 `abort_slave = true` 返回 `SLAVE_APPLY_EVENT_UNTIL_REACHED`
3. **半截事务处理**（GTID 重连场景）：

```cpp
if (ev->get_type_code() == binary_log::FORMAT_DESCRIPTION_EVENT &&
    ev->server_id != ::server_id && ev->common_header->log_pos != 0 &&
    rli->is_parallel_exec() && rli->curr_group_seen_gtid) {
  if (coord_handle_partial_binlogged_transaction(rli, ev)) ...
}
```

GTID 协议下 IO thread 每次重连后主库都会重发 FD（`log_pos != 0`）。若 SQL thread 此前正处理到一个半截事务（IO 断开导致事务没传完），MTS 必须给这个半截事务注入 BEGIN + ROLLBACK（`coord_handle_partial_binlogged_transaction`，`rollback_injected_by_coord` 标记）再继续——否则 worker 状态机悬空。

4. **STS 临时错误重试**（`slave_trans_retries`）：执行失败且非 MTS 时，回滚、重载位点、`applier_reader->open` 回到事务起点，`slave_sleep` 后退避，返回 `SLAVE_APPLY_EVENT_RETRY` 让主循环重读同一事件。**MTS 不做重试**（源码条件 `!is_mts_worker`）——并行模式下重试一个事务要求回滚所有与之交错的已提交事务，代价不可接受，直接报错。

#### apply_event_and_update_pos：STS 与 MTS 的分水岭

`apply_event_and_update_pos`（rpl_replica.cc:4452 附近）核心是 skip 判定与分发：

```cpp
reason = ev->shall_skip(rli);           // 跳过判定（GTID 已执行 / skip counter）
if (reason == Log_event::EVENT_SKIP_COUNT) { --rli->slave_skip_counter; skip_event = true; }

if (reason == Log_event::EVENT_SKIP_NOT) {
  if (sql_delay_event(ev, thd, rli))    // 延迟复制在此睡眠
    return SLAVE_APPLY_EVENT_AND_UPDATE_POS_OK;

  rli->set_group_source_log_start_end_pos(ev);   // STS 位点设置
  exec_res = ev->apply_event(rli);               // 真正执行

  // MTS：事件被分配给 worker（ev->worker != rli）
  if (!exec_res && (ev->worker != rli)) {
    if (ev->worker) {
      Slave_job_item item = {ev, rli->get_event_start_pos(), {'\0'}};
      ev->mts_group_idx = rli->gaq->assigned_group_index;
      ...
      if (append_item_to_jobs(job_item, w, rli))
        return SLAVE_APPLY_EVENT_AND_UPDATE_POS_APPEND_JOB_ERROR;
      ...
    }
    *ptr_ev = nullptr;  // 事件所有权移交 worker
  }
}

// STS 位点推进：XID / 已提交 DDL 不在 apply 路径 update_pos（提交时已 crash-safe 更新）
if (exec_res == 0) {
  if (*ptr_ev &&
      ((ev->get_type_code() != binary_log::XID_EVENT &&
        !is_committed_ddl(*ptr_ev)) || skip_event || ...)) {
    error = ev->update_pos(rli);
  } else {
    rli->inc_event_relay_log_pos();    // 只推进读游标
  }
}
```

**XID 不 update_pos 的深意**：XID 是事务提交，其位点推进发生在提交路径的 crash-safe 更新（`update_commit_group` 之后），不在事件回放路径——两者是两套位点语义，混用会造成"位点领先于实际提交"的假象。这与 [`binlog.md`](binlog.md) 中"提交点 = binlog fsync"是同一原则：**位点必须落后于（或等于）持久化事实**。

#### 退出清理的顺序敏感

SQL thread 退出（rpl_replica.cc:7317 之后）：先 `atomic_is_stopping = true`、停 worker 池，再持 `run_lock` + `data_lock` 关 reader、`slave_running = 0`、广播 `data_cond`（唤醒 `source_pos_wait` 等待者），最后**先 `stop_cond` 广播再 `run_lock` 解锁**：

```cpp
/* Note: the order of the broadcast and unlock calls below (first broadcast,
   then unlock) is important. Otherwise a killer_thread can execute between
   the calls and delete the mi structure leading to a crash! (see BUG#25306) */
mysql_cond_broadcast(&rli->stop_cond);
mysql_mutex_unlock(&rli->run_lock);
```

这是复制线程退出的经典竞态：STOP REPLICA 的等待者在 `stop_cond` 上，被唤醒后会检查 `slave_running==0` 然后可能释放 `mi`/`rli`。若先 unlock 后 broadcast，等待者在 broadcast 前拿到锁、看到旧状态、提前释放结构——因此**广播必须发生在锁内**（等待者醒来时锁还在，结构必然存活）。

### 位点持久化：两套仓库与 MTS checkpoint

#### flush_info 与节流

`Master_info::flush_info` 与 `Relay_log_info::flush_info` 走同一个模式：设置 `handler->set_sync_period(...)`（IO 侧用 `sync_masterinfo_period`，SQL 侧用 `sync_relayloginfo_period`，默认都是 10000），然后 `write_info` + `handler->flush_info`。节流在 handler 的 `do_flush_info` 里：

```cpp
// Rpl_info_table::do_flush_info —— 计数节流 + 不写 binlog
if (!(force || (sync_period && ++(sync_counter) >= sync_period))) return 0;
...
saved_options = thd->variables.option_bits;
thd->variables.option_bits &= ~OPTION_BINLOG;  // 位点表自己的更新不进 binlog
thd->is_operating_substatement_implicitly = true;
...
if ((res = access->find_info(field_values, table)) == NOT_FOUND_ID) {
  table->file->ha_write_row(table->record[0]);      // INSERT
} else if (res == FOUND_ID) {
  table->file->ha_update_row(table->record[1], table->record[0]);  // UPDATE
}
```

两套仓库（FILE vs TABLE）由 `opt_mi_repository_id`/`opt_rli_repository_id` 与 `Rpl_info_factory` 工厂选择，`Rpl_info` 基类定义统一接口。TABLE 仓库（`mysql.slave_master_info`/`slave_relay_log_info`）是 crash-safe 的关键——位点表是 InnoDB 表，与事务数据在同一崩溃恢复体系内（这正是"位点领先于数据"问题在 TABLE 仓库下不存在的原理，`relay_log_info_repository=TABLE` 也因此成为 5.7+ crash-safe 的标准配置）。

#### MTS 的 checkpoint：位点推进的 LWM 化

MTS 下 `group_*` 位点**不能**随任意事务提交推进（worker 乱序提交，位点推进会越过未完成的事务）。`mta_checkpoint_routine`（rpl_replica.cc:6452 附近）在 GAQ 上取 LWM 推进：

```cpp
bool mta_checkpoint_routine(Relay_log_info *rli, bool force) {
  ...
  do {
    if (!is_mts_db_partitioned(rli)) mysql_mutex_lock(&rli->mts_gaq_LOCK);
    cnt = rli->gaq->move_queue_head(&rli->workers);   // 弹出连续 done 的组
    if (!is_mts_db_partitioned(rli)) mysql_mutex_unlock(&rli->mts_gaq_LOCK);
    ...
  } while (cnt == 0 && ...);
  ...
  // LWM 即 GAQ 弹出后的队头事务组位点
  rli->set_group_master_log_pos(rli->gaq->lwm.group_master_log_pos);
  rli->set_group_relay_log_pos(rli->gaq->lwm.group_relay_log_pos);
  ...
  error = rli->flush_info(Relay_log_info::RLI_FLUSH_IGNORE_SYNC_OPT);
  mysql_cond_broadcast(&rli->data_cond);
  ...
}
```

`Slave_committed_queue`（GAQ）维护 `lwm` 成员——"队头即 LWM 之后第一个未完成组"。checkpoint 的触发：`rli_checkpoint_seqno >= checkpoint_group`（force）或 `is_time_for_mta_checkpoint()`（按 `replica_checkpoint_period` 毫秒，默认 300）。GAQ 与 LWM 的完整调度语义见 [`prpl.md`](prpl.md)。

### relay log recovery：崩溃后的从库复位

`relay_log_recovery=ON` 时的启动路径（rpl_rli.cc 的 `rli_init_info`）：

```cpp
if (is_relay_log_recovery) {
  if (init_recovery(mi)) { error = 1; goto err; }
} else if (mi->is_gtid_only_mode()) {
  // 非 recovery 的 GTID_ONLY：位点从头开始，靠 GTID 集合定位
  set_group_relay_log_name(index_entry_name.c_str());
  set_group_relay_log_pos(BIN_LOG_HEADER_SIZE);
}
```

`init_recovery`（rpl_replica.cc:1163 附近）的分层：

```cpp
int init_recovery(Master_info *mi) {
  // ① MTS gap 计算（恢复 worker 进度与 coordinator 的洞）
  error = mts_recovery_groups(rli);
  if (rli->mts_recovery_group_cnt) return error;   // 有 gap，延迟到 SQL 线程启动时补

  // ② 非 GTID 且从未启动过 SQL 线程：从 relay log 反找主库 Rotate
  if (!group_master_log_name[0] && !rli->mi->is_gtid_only_mode()) {
    if (rli->replicate_same_server_id) { error = 1; ... }
    error = find_first_relay_log_with_rotate_from_master(rli);
    if (error == 2) run_relay_log_recovery = false;  // relay log 无主库事件，跳过
    else if (error) return error;
  }
  if (run_relay_log_recovery) recover_relay_log(mi);
}
```

`recover_relay_log`（rpl_replica.cc:1095 附近）是复位核心：

```cpp
static void recover_relay_log(Master_info *mi) {
  Relay_log_info *rli = mi->rli;

  // ① IO 位点对齐到 SQL 位点：丢弃"已收到未回放"的 relay log，从主库重拉
  if (!mi->is_gtid_only_mode()) {
    mi->set_master_log_pos(std::max<ulonglong>(
        BIN_LOG_HEADER_SIZE, rli->get_group_master_log_pos()));
    mi->set_master_log_name(rli->get_group_master_log_name());
    ...
  }

  // ② 全新 relay log
  rli->set_group_relay_log_name(rli->relay_log.get_log_fname());
  rli->set_group_relay_log_pos(BIN_LOG_HEADER_SIZE);

  // ③ GTID 模式：清空 Retrieved_Gtid_Set，让半截事务被重新拉取
  if (global_gtid_mode.get() == Gtid_mode::ON &&
      !channel_map.is_group_replication_channel_name(rli->get_channel())) {
    rli->get_sid_lock()->wrlock();
    (const_cast<Gtid_set *>(rli->get_gtid_set()))->clear_set_and_sid_map();
    rli->get_sid_lock()->unlock();
  }
}
```

三步各自的意义：

- **① IO 位点对齐 SQL 位点**：崩溃时 relay log 里可能有"已收到、没回放完"的事务。旧 relay log 被整体丢弃（不可信——可能写了一半），IO 位点回退到 SQL 的 `group_master_log_pos`，主库从那里重发。**GTID_ONLY 模式跳过此步**（注释 "the receiver doesn't care about these positions"）——因为 GTID 集合能自洽定位，位点无意义
- **② 全新 relay log**：`group_relay_log_pos` 直接指向新文件头（`BIN_LOG_HEADER_SIZE` = 4，跳过 magic）。**旧 relay log 文件不保留**——它们的内容不可信（崩溃可能截断在半事件）
- **③ 清空 Retrieved_Gtid_Set**：这是 Bug #92882 修复（8.0.18/5.7.28）落地的核心。Retrieved 集合清空后，IO thread 重连时 exclude 集合只剩 `executed_gtids`——半截收到但没执行完的事务不在其中，主库会从头重发。**在 GTID 体系下，"从库已收到"这个状态是不需要保存的**，只有"已执行"才需要

#### MTS 的 gap 恢复

MTS 崩溃后 worker 的执行进度可能领先于 checkpoint 位点（部分 worker 提交了、部分没有，形成"洞"）。`fill_mts_gaps_and_recover`（rpl_replica.cc:1220 附近）的做法是**先串行补洞再正常启动**：

```cpp
static inline int fill_mts_gaps_and_recover(Master_info *mi) {
  Relay_log_info *rli = mi->rli;
  rli->is_relay_log_recovery = false;
  Until_mts_gap *until_mg = new Until_mts_gap(rli);
  rli->set_until_option(until_mg);
  rli->until_condition = Relay_log_info::UNTIL_SQL_AFTER_MTS_GAPS;
  ...
  // 启动一个"特殊任务"的 SQL 线程：单线程回放到 gap 填满
  recovery_error = start_slave_thread(
      key_thread_replica_sql, handle_slave_sql, &rli->run_lock, &rli->run_lock,
      &rli->start_cond, &rli->slave_running, &rli->slave_run_id, mi);
  ...
  mysql_cond_wait(&rli->stop_cond, &rli->run_lock);   // 等它完成
  ...
  recover_relay_log(mi);
  mi->flush_info(true);
  rli->flush_info(Relay_log_info::RLI_FLUSH_IGNORE_SYNC_OPT);
}
```

gap 的执行期由 `rli->recovery_groups` bitmap 标记哪些组需要补做（`apply_event_and_update_pos` 开头检查），`mts_recovery_group_cnt` 递减到 0 时 `mts_finalize_recovery()` 收尾。这个"把并行恢复到串行回放"的退路，正是 MTS 调度复杂性的另一面——**并行度在恢复期主动放弃，换取 gap 状态的确定性**。

#### 非 recovery 路径的 Retrieved_Gtid_Set 重建

`relay_log_recovery=OFF` 时（rpl_rli.cc 注释明说：recovery=ON 时跳过此步以省时间，因为 relay log 反正要丢弃），启动要重建 Retrieved 集合：`relay_log.init_gtid_sets(..., &mi->transaction_parser, partial_trx)` 反向扫描 relay log 找 `Previous_gtids_log_event`，并用事务边界 parser **只保留完整事务的 GTID**（半截事务的 GTID 进 `partial_trx` 监控而非集合）——与 recovery 路径"清空集合"殊途同归：两种路径都保证 Retrieved 集合里没有半截事务。

### 线程生命周期：START 与 STOP

#### START 的握手协议

`start_slave_threads`（rpl_replica.cc:2065 附近）创建线程，`start_slave_thread` 是握手核心：

```cpp
bool start_slave_thread(PSI_thread_key thread_key, my_start_routine h_func,
                        mysql_mutex_t *start_lock, mysql_mutex_t *cond_lock,
                        mysql_cond_t *start_cond,
                        std::atomic<uint> *slave_running,
                        std::atomic<ulong> *slave_run_id, Master_info *mi) {
  ...
  if (*slave_running) {  // 已在运行
    my_error(ER_REPLICA_CHANNEL_MUST_STOP, ...);
    goto err;
  }
  start_id = *slave_run_id;
  if (mysql_thread_create(thread_key, &th, &connection_attrib, h_func, (void *)mi)) {
    ...  // 创建失败
  }
  if (start_cond && cond_lock) {
    THD *thd = current_thd;
    while (start_id == *slave_run_id && thd != nullptr) {
      // 等子线程 ++slave_run_id 并广播 start_cond（或初始化失败提前广播）
      thd->ENTER_COND(start_cond, cond_lock, ...);
      if (!thd->killed) mysql_cond_wait(start_cond, cond_lock);
      mysql_mutex_unlock(cond_lock);
      thd->EXIT_COND(&saved_stage);
      mysql_mutex_lock(cond_lock);
      if (thd->killed) { my_error(thd->killed, MYF(0)); goto err; }
    }
  }
}
```

握手协议是 **run_id 变更 + 条件变量广播**：子线程在 `run_lock` 保护下先把 `slave_run_id++`、`slave_running = 1`，再广播 `start_cond`（handle_slave_io/handle_slave_sql 开头）。父线程在 `start_cond` 上等待 `start_id != *slave_run_id`。**初始化失败路径也广播**（否则父线程死等）——子线程任何 goto err 出口前都保证 `mysql_cond_broadcast(&start_cond)` 再解锁。

#### STOP 的优雅停止

`terminate_slave_threads`（rpl_replica.cc:1716 附近）按 SQL → IO 的顺序终止，两线程**共享同一个超时预算** `total_stop_wait_timeout`（`rpl_stop_replica_timeout`）：

```cpp
ulong total_stop_wait_timeout = stop_wait_timeout;   // 两线程共享预算

// SQL 先停（停掉消费侧，IO 才不再被 purge 依赖）
mi->rli->abort_slave = true;
terminate_slave_thread(mi->rli->info_thd, sql_lock, &mi->rli->stop_cond,
                       &mi->rli->slave_running, &total_stop_wait_timeout, ...);
mi->rli->flush_info(Relay_log_info::RLI_FLUSH_IGNORE_SYNC_OPT);  // 停前落一次位点

// IO 后停（停掉接收侧）
mi->abort_slave = true;
terminate_slave_thread(mi->info_thd, io_lock, &mi->stop_cond, &mi->slave_running,
                       &total_stop_wait_timeout, ...);
mi->flush_info(true);
```

`terminate_slave_thread` 的唤醒-等待循环（rpl_replica.cc:1909 附近）：

```cpp
while (*slave_running) {
  mysql_mutex_lock(&thd->LOCK_thd_data);
  int err = pthread_kill(thd->real_id, SIGALRM);   // 打断可能的阻塞 syscall
  if (force)
    thd->awake(THD::KILL_CONNECTION);
  else
    thd->awake(THD::NOT_KILLED);
  mysql_mutex_unlock(&thd->LOCK_thd_data);

  // 2 秒一个周期：重发信号直到线程退出（注释：线程可能漏掉第一个 alarm）
  struct timespec abstime;
  set_timespec(&abstime, 2);
  mysql_cond_timedwait(term_cond, term_lock, &abstime);
  if ((*stop_wait_timeout) >= 2)
    (*stop_wait_timeout) = (*stop_wait_timeout) - 2;
  else if (*slave_running) {
    ... return 1;   // 超时预算耗尽
  }
}
```

`pthread_kill(SIGALRM)` 是打断阻塞系统调用（如卡在 socket 读）的通用手段；`thd->awake(THD::NOT_KILLED)` 只做唤醒不做 kill 标记——这是**协作式停止**：线程醒来后自己检查 `abort_slave` 标志决定退出，而不是被强制终止（保证退出清理的完整性）。`sql_slave_killed` 还有一个细节：SQL thread 若正处在含非事务性变更的组中，会延迟接受 kill 直到组边界（WL#2975），避免非事务表跨语句撕裂。

---

## ★ 本机制里的工程实现技法

### 一、高级语法技巧 / C++ 特性

**Rpl_applier_reader 的"包装流"设计**：relay log 的读侧在 `Rpl_applier_reader` 与底层 `File_reader` 之间分层——reader 负责文件操作与等待，applier_reader 负责 relay log 语义（位点、跨文件、purge）。分层方式是把 `File_reader` 作为成员组合而非继承，使 applier_reader 可以控制"什么时候等待、什么时候跨文件"。

**继承体系：`Relay_log : MYSQL_BIN_LOG`**——relay log 复用 binlog 的整个类，只加少量 relay 特有成员（`relay_log_checksum_alg` 等），用 `is_relay_log` 标志在共享代码里分叉。这是"复用大对象、差异靠标志"的典型做法，代价是**每处共享代码都必须记得检查标志**——`write_buffer` 的 `assert(is_relay_log)`、rotate 时 FD 的来源、崩溃恢复跳过 relay log，都是这面硬币的痕迹。

| 特性 | 用在哪 | 为什么（收益） | 代价 / 反直觉处 |
|---|---|---|---|
| `Relay_log : MYSQL_BIN_LOG` 继承 | relay log 复用 binlog 写入逻辑 | 一条序列化/文件管理路径两用 | 每个共享函数都要惦记 `is_relay_log` 分叉 |
| `slave_sleep` 函数模板 | IO/SQL 两处可中断睡眠 | 模板参数化 killed 检查，重试退避与 STOP 响应合一 | 模板实参是函数指针，调试栈难读 |
| `RLI_current_event_raii` | SQL thread 事件上下文 | RAII 保证异常路径也恢复 rli 状态 | 事件所有权移交 worker 时生命周期跨界 |
| `std::atomic` + 条件变量握手 | `slave_running`/`slave_run_id` + `start_cond`/`stop_cond` | 无死锁的启动/停止同步 | broadcast 必须先于 unlock（BUG#25306） |
| `unique_ptr_my_free` 风格的所有权 | dump 报文缓冲区 | my_malloc 出的裸指针 RAII 化 | 与 STL allocator 契约混用要小心 |

### 二、经典算法的实现落地

**生产者-消费者的两段式**：IO thread（生产者）与 SQL thread（消费者）之间以 relay log 文件为缓冲，以 `COND_binlog_updated`（有数据）与 `log_space_cond`（有空间）两个条件变量为信号。与教科书模型的差异：**缓冲是磁盘文件而非内存队列**——因为复制允许的积压远超内存容量（`relay_log_space_limit` 默认 0 = 不限制）。这带来教科书模型没有的问题：文件管理（rotate/purge）、空间反压、以及"消费者删文件"的额外协议（`is_receiver_waiting_for_rl_space` + `coordinator_log_after_purge` 的三方握手）。

**位点的"影子写"（shadow write）**：读游标 `event_relay_log_pos`/`future_event_relay_log_pos` 的推进采用"先写未来、再提交"的方式——每读一个事件把"下一个位置"写入 `future_event_relay_log_pos`，事件处理完成后才 `inc_event_relay_log_pos()` 把游标提交过去。这与数据库事务的 undo/redo 影子页思想同构：**不完整状态永远落在"未来"字段，已提交状态才落在"当前"字段**，任何时刻崩溃都不会留下中间态。

| 维度 | 教科书/论文原型 | 本实现的落地 | 差异原因 / 代价 |
|---|---|---|---|
| 生产者-消费者缓冲 | 有界内存队列 + 信号量 | 无界磁盘文件（relay log）+ 双条件变量 | 积压可达 TB 级；代价是文件生命周期管理 |
| 检查点 | 日志截断到 LSN | GAQ 上 LWM 弹出 + `group_*` 位点 | 位点是事务组边界而非字节边界；MTS 下检查点是 LWM |
| 崩溃恢复 | 从检查点重放 | `recover_relay_log` 丢弃 relay log + GTID 集合重拉 | relay log 不可信（可能截断），GTID 让"重放"变成"重拉" |

### 三、复杂体系与设计模式的代码结构

```
Rpl_info（基类：data_lock/run_lock/sleep_lock/info_thd_lock + start/stop/data/sleep_cond）
├── Master_info  ── IO thread 的位点与连接状态（master_log_name/pos, mysql, transaction_parser）
│     └── flush_info → Rpl_info_factory → FILE(master.info) | TABLE(mysql.slave_master_info)
├── Relay_log_info ── SQL thread 的位点与调度状态（group_*, event_*, GAQ, current_mts_submode）
│     ├── relay_log : Relay_log : MYSQL_BIN_LOG   ← 复用的 binlog 类
│     └── flush_info → FILE(relay-log.info) | TABLE(mysql.slave_relay_log_info)
└── Rpl_applier_reader ── relay log 读侧封装（open/read/wait/purge）
```

锁序（rpl_info.h 注释，防死锁的硬约定）：`run_lock → data_lock → relay_log.LOCK_log → relay_log.LOCK_index`；`run_lock → sleep_lock`；`run_lock → info_thd_lock`（读 info_thd 持任一、写持两者）。空间反压的 `log_space_lock` 独立于此链，purge 路径是 `data_lock → log_space_lock`。

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `relay_log_recovery` | ON（8.0） | Global | 启动时恢复 relay log 位点 |
| `relay_log_purge` | ON | Global | SQL 线程回放后自动 purge 旧 relay log |
| `sync_relay_log` | 10000 | Global | 每 N 个事件 fsync relay log |
| `sync_master_info` | 10000 | Global | 每 N 个事件落 master info 仓库 |
| `sync_relay_log_info` | 10000 | Global（deprecated） | 每 N 个事务落 relay log info 仓库 |
| `relay_log_space_limit` | 0（不限制） | Global | relay log 总空间上限，触发 IO 反压 |
| `max_relay_log_size` | 0（=max_binlog_size） | Global | relay log 自动 rotate 阈值 |
| `slave_net_timeout` | 60 | Global | IO 线程无数据超时（触发重连探测） |
| `master_connect_retry`（`source_connect_retry`） | 60 | Global | IO 线程重连间隔（秒） |
| `master_retry_count`（`source_retry_count`） | 86400 | Global | 连续重连次数上限 |
| `slave_trans_retries` | 10 | Global | STS 事务执行失败重试次数 |
| `replica_checkpoint_period` | 300 | Global | MTS checkpoint 周期（毫秒） |
| `relay_log_info_repository` / `master_info_repository` | TABLE | Global | 位点仓库类型 |

| 状态变量 | 说明 |
|--------|------|
| `Slave_running` / `Slave_open_temp_tables` | 线程运行状态 / 临时表残留 |
| `Replica_heartbeat_period` | heartbeat 间隔 |
| `Slave_received_heartbeats` / `Slave_retried_transactions` | 心跳计数 / 重试计数 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 两段延迟（IO 落后 vs SQL 落后） | SQL | `SHOW REPLICA STATUS` 的 `Relay_Log_Pos` vs `Read_Source_Log_Pos` vs `Exec_Source_Log_Pos`、`Seconds_Behind_Source` |
| 线程卡在哪个阶段 | PFS | `performance_schema.replication_applier_status_by_coordinator` / `by_worker`（SERVICE_STATE、LAST_ERROR_NUMBER）；`PROCESSLIST` 的 IO/SQL 线程 State（`Waiting for source to send event` / `Waiting for an event from Coordinator`） |
| relay log 空间与水线 | SQL | `SHOW REPLICA STATUS` 的 `Relay_Log_Space`；`information_schema` 无 relay log 视图，看 datadir 文件 |
| 位点仓库内容 | SQL | `SELECT * FROM mysql.slave_master_info` / `mysql.slave_relay_log_info`（TABLE 仓库） |
| IO 线程重连历史 | 日志 | error log 的 `ER_RPL_REPLICA_ERROR_RETRYING`、`replica_retried_transactions` |

---

## Misc

### 坑与已知缺陷

**Bug #92882（已修复于 8.0.18 / 5.7.28）**：MTS + GTID + `MASTER_AUTO_POSITION=ON` + `relay_log_recovery=ON` 的"全正确配置"下，OS 崩溃导致未刷盘的 relay log 丢失时，5.7.23 的 relay log recovery 会因找不到 relay log 文件直接拒绝启动复制。修复思路：AUTO_POSITION 下 recovery 的"位点复位"是多余的——GTID 自动定位可以恢复任何缺失事务。8.0.39 的落地形态是 `recover_relay_log` 清空 Retrieved_Gtid_Set + `init_recovery` 的 `find_first_relay_log_with_rotate_from_master` 返回 2（无主库事件）时跳过 recovery。**教训**：relay log 是缓存不是真相，GTID 模式下任何把缓存状态当真相的恢复路径都是 bug 源。

**MTS gap 卡死（社区案例，需以源码为准）**：社区有文章（LumaDB，2026-03）描述 8.0.35 中"物理删除 relay log 文件 + 保留 index 文件"后重启，`relay_log_recovery` 把 IO 位点重置为 file+pos 后 MTS coordinator 因 worker 状态不连续卡在 `waiting for handler commit`、`Seconds_Behind_Master` 持续增长且无报错。该场景涉及"手动删除 relay log 破坏 index 一致性"的非正常操作，且其"绕过 AUTO_POSITION"的表述与 8.0.39 源码的 GTID_ONLY 分支不完全吻合——**处置建议以官方手册为准**（relay log 只能由 MySQL 自己 purge，`relay_log_purge` 不要关），若确需手工干预，走 `STOP REPLICA → RESET REPLICA → CHANGE REPLICATION SOURCE` 重建，而非重启碰运气。

**`relay_log_recovery` 的语义边界**：它解决"relay log 不可信（截断/半事件）"，不解决"relay log 物理消失"。前者靠丢 relay log + 重拉，后者是运维事故，靠 GTID 兜底（AUTO_POSITION 场景）或备份重建（非 GTID 场景）。

### 容易误解的命名

| 名称 | 常被误解为 | 实际 |
|------|-----------|------|
| `master_log_pos`（mi） | SQL 回放位置 | IO 已接收并落 relay log 的位置 |
| `group_master_log_pos`（rli） | 与 master_log_pos 同义 | SQL 已回放的事务组在主库坐标系的位置 |
| `relay_log_pos` / `event_relay_log_pos` | 持久化位点 | 仅内存读游标，重启后由 group 位点重建 |
| Retrieved_Gtid_Set | "已执行"集合 | "已完整收到进 relay log"集合（可能未执行） |
| `sync_relay_log` | 与 `sync_binlog` 同等重要 | relay log 可丢可重拉，默认 10000 足够 |
| replica 改名 | 代码全改名 | 线程函数仍是 `handle_slave_io`/`handle_slave_sql`，错误码仍是 `ER_REPLICA_*` 与内部 `slave_*` 混杂 |

### 社区边界澄清

- **MySQL 从库没有"并行 apply 单个大事务"**：MTS 的并行粒度是事务，事务内部的事件串行执行（对比 TiDB 等可以行级并发 apply）；单事务体量大的场景 MTS 无效
- **`relay_log_space_limit` 与 binlog 的空间管理无关**：binlog 由 `binlog_expire_logs_seconds` 管理，relay log 由 SQL 回放推进 purge——两者独立

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Replication → Replication Implementation → Replication Relay and Status Logs*
- *MySQL 8.0 Reference Manual → Replication → Replication and Binary Logging Options and Variables*
- WorkLog: [WL#11262](https://dev.mysql.com/worklog/task/?id=11262) — Crash-safe replication（位点仓库与 relay log recovery 的设计动机）
- WorkLog: [WL#7165](https://dev.mysql.com/worklog/task/?id=7165) — MTS logical clock（并行调度，见 [`prpl.md`](prpl.md)）

**Bug 论坛**
- [Bug #92882](https://bugs.mysql.com/bug.php?id=92882) — MTS not replication crash-safe with GTID and all the right parameters（修复于 8.0.18/5.7.28：AUTO_POSITION 下 recovery 跳过/GTID 补齐）
- [Bug #25306](https://bugs.mysql.com/bug.php?id=25306) — 复制线程退出 broadcast/unlock 顺序竞态（线程退出清理顺序的来源）

**内核月报 / 技术文章**
- [腾讯云数据库内核月报 — 浅谈 ERROR 1872 与 5.6/5.7 MTS group recovery](https://cloud.tencent.com/developer/article/1437475)
- [LumaDB — relay_log_recovery 的陷阱：MTS 多线程复制下的数据静默丢失](https://zihua.cloud/posts/relay-log-recovery-trap/)（社区案例视角，机制细节以本仓库源码为准）

**相关文档**
- 上游（主库侧 dump 与 binlog 格式）见 [`binlog.md`](binlog.md)
- SQL thread 的 MTS 调度细节见 [`prpl.md`](prpl.md)
- GTID 集合与 exclude 集合见 [`gtid.md`](gtid.md)
- 崩溃后引擎侧恢复见 [`../../innodb/recovery.md`](../../innodb/recovery.md)
