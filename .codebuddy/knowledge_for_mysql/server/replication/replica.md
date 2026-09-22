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

空间判定 `exceeds_relay_log_limit` 精确到字节：`log_space_limit != 0 && log_space_limit < log_space_total + queued_size`——`log_space_total` 由"写事件加、purge 文件减"持续维护。GTID 事件之所以用 `trx_length`（整个事务长度）估算而不是 `event_len`（单个事件字节）：反压一旦在事务中间触发就无解（不能在事务中间 purge 文件），所以要在**事务开始前**用全事务长度预判。

`wait_for_relay_log_space`（rpl_replica.cc）的协议是"置标记 → 主动 rotate → 睡在 `log_space_cond` 等 SQL 侧 purge"：

```cpp
static bool wait_for_relay_log_space(Relay_log_info *rli, size_t queued_size) {
  // from now on, until the time is_receiver_waiting_for_rl_space is
  // cleared, every rotation made by coordinator and executed
  // outside of a transaction, will purge the currently rotated log
  rli->is_receiver_waiting_for_rl_space.store(true);

  // rotate now to avoid deadlock with FLUSH RELAY LOGS, which calls
  // rotate_relay_log with a default locking order ...
  // Before rotation, is_receiver_waiting_for_rl_space is already set, so
  // after exiting the rotate_relay_log, coordinator executing rotation
  // requested here will see the correct value and will purge applied
  // logs with force option
  rotate_relay_log(mi, true, true, true);

  // capture the log name to which we rotated:
  mysql_mutex_lock(rli->relay_log.get_log_lock());
  std::string receiver_log = rli->relay_log.get_log_fname();
  mysql_mutex_unlock(rli->relay_log.get_log_lock());

  ...
  mysql_mutex_lock(&rli->log_space_lock);
  thd->ENTER_COND(&rli->log_space_cond, &rli->log_space_lock, ...);
  while (exceeds_relay_log_limit(rli, queued_size) &&
         !(slave_killed = io_slave_killed(thd, mi)) &&
         rli->coordinator_log_after_purge != receiver_log) {
    mysql_cond_wait(&rli->log_space_cond, &rli->log_space_lock);
  }
  mysql_mutex_unlock(&rli->log_space_lock);
  thd->EXIT_COND(&old_stage);

  rli->is_receiver_waiting_for_rl_space.store(false);
  return slave_killed;
}
```

三个动作各有其不得已的理由，合起来是一个**双死锁避免协议**：

1. **`is_receiver_waiting_for_rl_space` 置位**：给 SQL 侧的声明——"IO 正在等空间，凡事务外的 relay log rotate 都给我 force purge 掉已回放文件"。SQL 侧 coordinator 在 rotate 时检查这个标志，决定是否用 force 选项 purge（`purge_applied_logs` 的 force 分支），否则正常 rotate 不会主动释放已回放空间，IO 可能永远等不到空间。
2. **先主动 `rotate_relay_log` 一次**：动机不是"挤出旧文件"这么简单——源码注释明说 **"rotate now to avoid deadlock with FLUSH RELAY LOGS"**。`FLUSH RELAY LOGS` 会以默认锁序调 `rotate_relay_log`，若 IO 先睡在 `log_space_cond` 上等空间、而空间又需要一次 rotate 才能释放，就与 FLUSH 命令形成锁序死锁。先自己 rotate 一次（此时 `is_receiver_waiting_for_rl_space` 已置位，coordinator 执行这次 rotate 时会 force purge），把"需要 rotate 才能释放的空间"提前释放掉，之后等待期间就不再需要 rotate。
3. **while 第三条 `coordinator_log_after_purge != receiver_log`**：`receiver_log` 是 rotate 后 IO 当前写入的文件名，`coordinator_log_after_purge` 是 SQL 侧"已 force purge 到哪个文件"的推进标记（每次 purge 后更新为当前 group 文件并广播 `log_space_cond`）。当两者相等时，SQL 已把 receiver 当前文件之前的所有文件都 purge 完了——**不可能再有 purge 释放空间**，继续睡就是无限等待。官方字段注释（rpl_rli.h）点出最极端的触发场景：**单事务 + relay log 元数据本身比 `relay_log_space_limit` 还大**时，这个事务永远装不进限额。此时必须 break 出去、允许暂时超限继续入队——这是 IO/SQL 互等死锁的解法：空间反压的最终出路不是"等到空间"，而是"等不到就放行，让 SQL 追上"。

唤醒条件对应三条：空间够了 / 被 kill / `coordinator_log_after_purge` 前进了（SQL 侧 purge 后广播 `log_space_cond`）。

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

#### purge：SQL 侧的清理（完整机制）

**触发点**：`move_to_next_log()`——SQL thread 读到文件尾、切到下一个 relay log 时，且 `!is_in_group()`（**事务外**）。事务内绝不 purge（与 rotate 同理）。

`purge_applied_logs`（rpl_applier_reader.cc）的完整流程：

```cpp
if (!relay_log_purge) return false;                    // ① 开关

Shared_backup_lock_guard backup_lock{current_thd};     // ② 备份锁
switch (backup_lock) {
  case locked: break;
  case not_locked:
    LogErr(WARNING_LEVEL, ER_LOG_CANNOT_PURGE_BINLOG_WITH_BACKUP_LOCK);
    return false;                                      //    备份中：放弃本次 purge
  case oom: m_errmsg = ER_OUT_OF_RESOURCES_MSG; return true;
}

if (m_rli->flush_info(RLI_FLUSH_IGNORE_SYNC_OPT)) {    // ③ 先把 SQL 位点落盘
  m_errmsg = "Error purging processed logs"; return true;
}

m_rli->relay_log.lock_index();
mysql_mutex_lock(&m_rli->log_space_lock);
auto current_log_space = m_rli->log_space_total.load();
m_rli->relay_log.purge_logs(
    m_rli->get_group_relay_log_name(),
    false /* include */,                               // ④ 不含当前文件
    false, false, &current_log_space, true);
m_rli->log_space_total.store(current_log_space);       // ⑤ 扣减空间
m_rli->coordinator_log_after_purge = m_rli->get_group_relay_log_name();
mysql_cond_broadcast(&m_rli->log_space_cond);          //    唤醒等空间的 IO
mysql_mutex_unlock(&m_rli->log_space_lock);

m_rli->relay_log.find_log_pos(&m_linfo,                // ⑥ 文件被删，重定位读游标
                              m_rli->get_event_relay_log_name(), false);
m_rli->relay_log.unlock_index();
```

六个设计点各有其理由：

1. **① 开关 `relay_log_purge`（默认 ON）**：关闭后 relay log **只增不删**——磁盘持续增长，且必须靠 `relay_log_recovery` 或手工清理；官方不建议关（手册明确"relay log 只能由 MySQL 自己 purge"）。
2. **② 备份锁**：purge 会删文件，而物理备份可能正在读它们。拿不到备份锁就**放弃本次 purge 并告警**（不阻塞、不报错），下次机会再试——这是"宁可晚删也不破坏备份"的取舍。
3. **③ 顺序：先 flush_info 再 purge**——这是"位点不能领先于数据"的**反向保证**：必须先把 SQL 位点（已回放到哪）持久化，才能删除对应的 relay log。若顺序反过来，删完文件后崩溃 → 位点说"还没回放完"但文件已经没了 → 事务永久丢失。
4. **④ `include = false`**：purge 掉 group 位点所在文件**之前**的所有文件，**当前正在回放的文件不能删**（SQL thread 正在读它）。
5. **⑤ 空间记账与唤醒在同一把 `log_space_lock` 下完成**：先扣 `log_space_total`、更新 `coordinator_log_after_purge`、再广播 `log_space_cond`——源码注释明说要在 signal 之前改完变量。这正是「空间反压」节 IO thread 的唤醒源。
6. **⑥ 重定位**：文件被删后 `linfo` 缓存失效，必须 `find_log_pos` 重新定位读游标。

**与空间反压的联动**：`should_purge_current_relay_log` 标志由 IO 侧的 `is_receiver_waiting_for_rl_space` 触发（见「空间反压」节）。IO 等空间时，SQL 侧切文件时会把 group 位点**推进到下一个文件**（`set_group_relay_log_name(next_file)`），使**当前文件也进入可 purge 范围**——这是反压能真正释放空间的最后一环：正常情况下当前文件不删，等空间时破例让它也能被删。

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

#### 为什么必须 crash-safe 持久化（file+pos 的根本原因）

**因为 file+pos 模式下"起点"是从库自己的责任**：主库只是照着从库给的 (文件名, 偏移) 发数据，不参与计算、也不做去重（见 `check_start_file` 分支① 与"file+pos 没有事件级跳过"）。于是从库持久化的 io 位点就是**重连时唯一的依据**，它的准确性直接决定数据正确性：

| 位点出错方向 | 后果 |
|---|---|
| **落后**（位点 < relay log 实际内容） | 重连重复拉取已收过的事务 → 重复执行 → `1062` 主键冲突 / `1032` 找不到行。**file+pos 没有幂等机制**（不像 GTID 有集合去重） |
| **超前**（位点 > 实际已落盘的事件） | 那些事件再也不会被请求 → **永久丢事务**，且不报错 |

第二个方向最危险，所以真正的约束是**"位点绝不能领先于数据"**——这决定了 `queue_event` 的写入顺序：先 `write_buffer` 把事件写进 relay log、再推进 `master_log_pos`、最后才 `flush_info`。

**对比 GTID 模式为什么没这么苛刻**：GTID 重连时发 exclude 集合由主库重新定位，位点稍落后只会"多拉一点"，而多拉的事务因在集合里被 `skip_event` 跳过（整事务粒度），天然幂等。所以 GTID 模式下 io 位点仍要维护（用于显示、`relay_log_recovery`），但**不需要精确到事件级的 crash-safe**。这也是 GTID_ONLY 模式敢把文件名从仓库里删掉的底气。

**落地三要素**：

1. **仓库必须是 TABLE**（`mysql.slave_master_info` / `slave_relay_log_info` 是 InnoDB 表）——位点的更新与业务数据在同一崩溃恢复体系内，要么一起提交要么一起回滚，不存在"数据提交了位点没提交"。8.0 已强制这一点：两个 repository 变量默认值就是 `TABLE` 且**已标记为 DEPRECATED**（不再支持 FILE）。这也是 5.7+ "crash-safe replication"的标准配置。
2. **节流与强制**：周期性同步由 `sync_source_info`（默认 10000 个事件）/ `sync_relay_log_info` 控制；关键节点用 `force` 强制落盘（见下节 `do_flush_info` 的 `force ||` 判据）。
3. **`relay_log_recovery=ON` 兜底**：启动时若 relay log 与位点信息不一致（崩溃残留），丢弃不可信的 relay log、按 SQL 位点重新从主库拉取（详见「relay log recovery」章）。

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

### Seconds_Behind_Source 的计算公式（★ 最常被误解的指标）

`SHOW REPLICA STATUS` 的 `Seconds_Behind_Source`（旧名 `Seconds_Behind_Master`，8.0 改名）本质是**「SQL 线程现在回放的事件，是主库多久之前产生的」**，不是"从库比主库慢了多少秒"的字面意思。它的计算在 `sql/rpl_replica.cc` 的 `fill_slave_rows`，源码里就有一段伪代码注释把逻辑讲得最清楚：

```
if (SQL 线程在运行) {
  if (SQL 线程已处理完所有 relay log) {      // IO 位点 == SQL 位点
    if (IO 线程在运行)  打印 0;               // 追平
    else               打印 NULL;             // 位点追平但 IO 断了 → 无法判断
  } else {
    compute Seconds_Behind_Source;            // 真落后 → 计算
  }
} else {
  打印 NULL;                                   // SQL 线程没跑
}
```

核心公式（`rpl_replica.cc` 的 `fill_slave_rows`）：

```cpp
long time_diff = ((long)(time(nullptr) - mi->rli->last_master_timestamp) -
                  mi->clock_diff_with_master);
protocol->store(
    (longlong)(mi->rli->last_master_timestamp ? max(0L, time_diff) : 0));
```

公式里两个量，各自有独立的更新时机和含义：

**① `last_master_timestamp`——SQL 线程正在回放的事件，主库何时产生的**。SQL 线程每执行一个事件，更新为「事件头的时间戳 + 该语句执行耗时」（`rpl_replica.cc` 的 `apply_event_and_update_pos`）：

```cpp
rli->last_master_timestamp =
    ev->common_header->when.tv_sec + (time_t)ev->exec_time;
```

所以它**不是主库当前时间，而是"当前正在回放的那条 binlog 事件"的落盘时间**。MTS 下它只在 coordinator 从 GAQ 取出 job 时更新（并行回放时没有"正在回放哪一条"的单一概念，取的是 GAQ 队头事务的时间戳）。

**② `clock_diff_with_master`——主从时钟差**。IO 线程每次连主库时，通过 `SELECT UNIX_TIMESTAMP()` 在主库会话里读主库时钟，再与从库本地时钟相减得到（`rpl_replica.cc` 的 `get_master_version_and_clock`）：

```cpp
mi->clock_diff_with_master =
    (long)(time(nullptr) - strtoul(master_row[0], nullptr, 10));
```

它的存在是为了**抵消主从系统时钟不同步**——SBM 要算的是"主库视角下的延迟"，所以必须把主从时钟差减掉。若读时钟失败（如权限不足），置 0 并记 warning `ER_RPL_REPLICA_SECONDS_BEHIND_SOURCE_DUBIOUS`。

**三个最容易踩的坑**（源码注释专门解释了每一个）：

1. **负数归零**：`max(0L, time_diff)`。可能出负数的原因包括——主库本身是另一台主库的从库（时间领先）、主库有人 `SET TIMESTAMP`、或时间函数秒级粒度造成的 ±1 误差（注释给了个 `0-(2-1)=-1` 的精确例子）。归零是为了不吓到用户。
2. **`last_master_timestamp == 0` 是"已追上"的哨兵**：0 对应 1970 这个"不可能"的时间戳，专门用来标记"考虑自己已经追平"。所以公式里 `last_master_timestamp ? ... : 0`——一旦是 0 直接报 0。
3. **返回 NULL 不是"追平"而是"无法判断"**：SQL 线程没运行、或位点追平但 IO 线程断了，都返回 NULL。很多人把 NULL 当 0 看，其实语义相反——NULL 意味着"这个值此刻没有意义"。

**由此得到的正确解读**：SBM 反映的是**纯 SQL 回放侧**的延迟（IO 拉取侧不直接体现在这个数里）。IO 严重落后但 SQL 追着 IO 跑时，SBM 可能显示 0（因为 SQL 处理的 relay log 都是刚拉到的、时间戳很新），而真实的"数据延迟"要看 IO 位点（`Relay_Log_Pos` vs `Read_Source_Log_Pos`）——这正是上面速查表里"两段延迟"要分开看的原因。

### 经典错误码速查

> 错误码是复制的"体检报告"——`Last_IO_Errno` / `Last_SQL_Errno` 直接告诉你链路断在哪一段。本节按 IO 侧 / SQL 侧 / GTID 侧 / MTS 侧分类，错误码名称与消息均经 8.0.39 `share/messages_to_clients.txt` 核实；5.7 旧编号在 8.0 的改名/删除情况逐条标注。

#### IO 侧：拉取与落盘失败

| 错误码 | 名称 | 触发场景 | 根因与排查 |
|--------|------|---------|-----------|
| **13114** | `ER_SERVER_SOURCE_FATAL_ERROR_READING_BINLOG` | dump 时主库报错"Got fatal error from source when reading data from binary log" | ★ 经典 1236 的 8.0 版。**同一错误的两个形态**：主库 dump 线程发 `ER_SOURCE_FATAL_ERROR_READING_BINLOG`（客户端错误码），从库 IO thread 收到后包装成 `ER_SERVER_SOURCE_FATAL_ERROR_READING_BINLOG`=13114 记入 `Last_IO_Errno`（rpl_replica.cc 的 `handle_slave_io` switch 分支）。file+pos 模式：请求的 binlog 已被 purge 或从未存在；GTID 模式：集合校验失败（★ 三道校验任一失败主库都统一发这个错误码，靠 `Last_IO_Error` 文本区分具体原因，机制见 [`gtid.md`](gtid.md)「三道校验」）。排查：主库 `SHOW BINARY LOGS` 对比 `Read_Source_Log_Pos`；处置：file+pos 重建复制或 `CHANGE REPLICATION SOURCE TO AUTO_POSITION=1` |
| `ER_REPLICA_RELAY_LOG_WRITE_FAILURE` | 同左 | `queue_event` 写 relay log 失败 | 磁盘满/只读/permission；IO thread 终止但 SQL thread 可继续消费存量 relay log |
| `ER_NETWORK_READ_EVENT_CHECKSUM_FAILURE` | 同左 | 网络传输 checksum 校验失败（`queue_event` 入口） | 网络损坏/半途断包；IO 重连后主库重发（checksum 在入队前校验，脏数据不落 relay log） |
| `ER_RELAY_LOG_INIT` | 同左 | relay log 位点初始化失败（"Failed initializing relay log position"） | 常与 index 文件损坏、位点仓库内容非法有关；见「坑与已知缺陷」的 Bug #92882 |

#### SQL 侧：回放数据不一致（★ 最高频且最危险的类别）

| 错误码 | 名称 | 触发场景 | 根因与排查 |
|--------|------|---------|-----------|
| **1032** | `ER_KEY_NOT_FOUND` | 回放 DELETE/UPDATE 时用 before-image 定位不到行 | 主从数据已分叉（从库少了行、主库修改过该行、或从库手动改过数据）。经典诱因：从库被误写、`SET SQL_LOG_BIN=0` 改主库、主库 binlog 部分丢失 |
| **1062** | `ER_DUP_ENTRY` | 回放 INSERT 时主键冲突 | 从库多了行（同 1032 的反向分叉）；诱因同上 |
| **1146** | `ER_NO_SUCH_TABLE` | 回放 DML 时表不存在 | 从库缺 DDL（DDL 被过滤/复制权限不足/手滑 `DROP` 了从库的表） |
| **1050** | `ER_TABLE_EXISTS_ERROR` | 回放 CREATE 时表已存在 | 与 1146 对称的分叉 |
| **1205** | `ER_LOCK_WAIT_TIMEOUT` | 回放 DML 行锁等待超时（`innodb_lock_wait_timeout`） | 从库本地负载（读写分离）与回放抢行锁；MTS 下 worker 间也可能互等 |
| **1213** | `ER_LOCK_DEADLOCK` | 回放 DML 死锁 | 同上，且 MTS 并行回放两个本应串行的事务时可能触发（主库依赖编码的 false negative 或本地写与回放交错） |
| `ER_REPLICA_CANT_CREATE_CONVERSION` | 同左 | 字符集转换表创建失败 | 从库缺字符集/字符集不一致 |

**1032/1062 的通用处置**：确认分叉后，`SET GLOBAL SQL_REPLICA_SKIP_COUNTER=1`（GTID 模式用 `SET GTID_NEXT` 注入空事务）跳过后用 `pt-table-checksum` / `pt-table-sync` 修复数据，或重建从库。跳过是治标，数据修复是治本。

#### GTID 侧：集合校验失败

| 错误码 | 名称 | 触发场景 | 根因 |
|--------|------|---------|------|
| `ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE` | 同左 | 从库 exclude 集合 ⊄ 主库 `executed ∪ owned` | GTID 分叉：从库执行了主库没有的事务（从库曾被提升写数据 / 主库 binlog 截断丢失）。消息明说 "source may have rolled back transactions that were already replicated to the replica" |
| `ER_SOURCE_HAS_PURGED_REQUIRED_GTIDS` | 同左 | 主库 `lost_gtids` ⊄ 从库 exclude 集合 | 主库 purge 掉了从库还没收到的事务——数据在主库已物理消失，复制无解。处置：从备份重建从库（消息原文 "provision a new replica from backup"）。**预防**：purge binlog 前确认所有从库 `Exec_Source_Log_Pos` 已过 purge 点 |

> 这两个错误码的完整判定逻辑（dump 线程的三道校验）见 [`gtid.md`](gtid.md)「GTID 复制协议：exclude 集合」。

#### MTS 侧：并行调度相关

| 错误码 | 名称 | 触发场景 | 说明 |
|--------|------|---------|------|
| `ER_REPLICA_WORKER_STOPPED_PREVIOUS_THD_ERROR` | 同左 | `replica_preserve_commit_order=ON` 时某 worker 报错后，后序 worker 连带停止 | 保序机制的代价：前序事务未提交，后序已执行完的事务也不能提交（消息原文 "the last transaction executed by this thread has not been committed"）。修复主 worker 错误后需一并重启 |
| `ER_WARN_OPEN_TEMP_TABLES_MUST_BE_ZERO` | 同左 | 需要临时表清零才能安全执行的操作 | 警告级：临时表会保持打开直到重启或被复制的 DROP 删除 |

**8.0 删除/改名的对照（5.7 经验直接迁移会踩坑）**：

| 5.7 名称/编号 | 8.0.39 实际 | 说明 |
|--------------|------------|------|
| `ER_MTS_INCONSISTENT_DATA`（1756） | **已删除**（全仓库 0 匹配） | 5.7 MTS gap 恢复的经典错误码；8.0 的 MTS 恢复重写后不再需要（见「relay log recovery」的 `UNTIL SQL_AFTER_MTS_GAPS` 路径） |
| `ER_MASTER_FATAL_ERROR_READING_BINLOG`（1236） | `ER_SOURCE_FATAL_ERROR_READING_BINLOG`（主库侧）/ `ER_SERVER_SOURCE_FATAL_ERROR_READING_BINLOG`（13114，从库侧 `Last_IO_Errno`） | 术语改名（master→source），且拆成主从两侧两个符号 |
| `ER_SLAVE_RELAY_LOG_READ_FAILURE` / `ER_SLAVE_RELAY_LOG_WRITE_FAILURE` | `OBSOLETE_` 前缀标记 | 消息文件里已标记废弃，实际使用新名 `ER_REPLICA_*` |

**排查顺序总结**：`Last_IO_Errno` ≠ 0 → 看 IO 侧（连接/拉取/purge）；`Last_SQL_Errno` ≠ 0 → 看 SQL 侧（1032/1062 = 数据分叉，1205/1213 = 锁，1146/1050 = DDL 分叉）；GTID 侧两个错误码在任何一侧都可能出现，本质是集合校验失败而非线程故障。

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
