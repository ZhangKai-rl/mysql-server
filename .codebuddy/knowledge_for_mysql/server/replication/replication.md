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

#### 2. file+pos 的拓扑绑定 vs GTID 的拓扑无关——本篇最重要的理论

file+pos 模式的位点是 `(binlog 文件名, 文件内偏移)`，这是一个**拓扑绑定**的坐标：它只在"产生它的那台主库"上有意义。两台主库的 binlog 文件命名和内容完全无关——A 库的 `binlog.000003:120` 在 B 库上没有任何对应物。

所以传统 failover 的流程是"猜一个位点"：找到旧主库最后一个 binlog 事件在从库上的执行位置，换算成新主库上的等价位点。这既易错（漏算/重算事件）又依赖人工。

GTID 把坐标换成了**内容寻址**：每个事务带全局唯一 ID（`SERVER_UUID:GNO`），从库记住的是"我执行过哪些事务"（集合），而不是"我读到了哪个字节"。切换主库时，从库只需要问新主库："**给我所有不在我 exclude 集合里的事务**"——新主库用 auto position 算法（见核心实现）自己定位起点。**位点从"坐标"变成了"差集"**，拓扑无关性由此而来。

这正是 5.6 引入 GTID 的根本动机：让 failover 从"手工换算位点"变成"自动收敛"。

#### 3. log_replica_updates 默认翻转：级联从"手动开启"到"默认可用"

`log_replica_updates` 控制从库是否把**回放的事务再写进自己的 binlog**。5.7 及以前默认 OFF，8.0 翻转为默认 ON。

- OFF 时：从库不写 binlog，**无法再被下一级从库复制**（级联链断裂）。这是 5.7 默认值——因为从库写 binlog 有开销，官方默认"不写"，需要级联的人自己开。
- ON 时：从库同时是下一级的"主库"，级联默认可用。8.0 翻转的理由：读写分离/多级分发拓扑已是常态，默认 OFF 的"省开销"远小于"级联链默认可工作"的价值；且 GTID 模式（多数部署）下回放事务写 binlog 的语义明确（GTID 不重新分配，原样传播）。

它也是**环拓扑的开关**：双主/环拓扑必须双写 binlog 才能让事务流通（见拓扑形态节）。

#### 4. 一致性 = 确认机制 × 拓扑 的二维选择

复制的"一致性"不是单一开关，是两个维度的组合：

| 确认机制 | 丢失窗口 | 性能代价 | 实现 |
|---------|---------|---------|------|
| 异步（默认） | 主库提交后到从库收到前的所有事务 | 零 | 纯拉模型 |
| 半同步 after_commit | 缩小到"已收到未回放" | 主库 commit 等 ACK | 插件，见 binlog.md |
| 半同步 after_sync | 理论上零（已落 relay log） | 同上 + sync 延迟 | 插件，见 binlog.md |
| 组复制（MGR） | 零（多数派确认后提交） | Paxos 往返 | 插件 |

注意确认机制解决的是"**单次切换丢不丢**"，拓扑解决的是"**切到谁**"。二者正交：异步复制 + 精心挑选的从库切换，也可以做到 RPO 很小（但需要人工确认从库追平）；半同步 + 乱切换也会丢（切到一个还没收到事务的从库）。这是故障转移节要展开的。

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

#### 三个 COM 命令

| 命令 | 载荷 | 语义 |
|------|------|------|
| `COM_REGISTER_SLAVE` | server_id + 主机名/端口/用户 | 登录后注册身份，主库用于 `SHOW REPLICAS` 与僵尸连接管理（非必需） |
| `COM_BINLOG_DUMP`（0x12） | binlog 文件名 + 偏移 + server_id | file+pos 定位：从指定字节开始发事件 |
| `COM_BINLOG_DUMP_GTID` | 编码后的 GTID 集合 + server_id | GTID 定位：发"集合之外"的事务 |

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

#### 双主 / 环形：replicate_same_server_id 与防循环

```
A ⇄ B          （双主）        A → B → C → A   （环）
```

双写拓扑（两个节点互为主备，两边都可写）在 GTID 时代的正确性约束：

1. **`log_replica_updates=ON` 必须**——否则对方写来的事务进不了自己的 binlog，回传链断裂。
2. **`replicate_same_server_id` 控制"自己写的事务要不要回放"**：默认 OFF 时，从库发现事件的 `server_id == 自己的 server_id` 就直接跳过（`log_event` 的应用入口判定）——这是防环的基础机制：A 写的事务经 B 回传 A 时被 A 跳过。双主/环拓扑里**双方都要开** `replicate_same_server_id=ON`？**不是**——恰恰相反：**保持 OFF 才能防环**。开启它意味着"连自己 server_id 的事件也回放"，只用于特殊场景（如通过中间件过滤后回灌），且官方手册明确警告开启它可能在环拓扑里造成**无限循环**。
3. **GTID 是真正的防环手段**：即使事件回来了，A 发现该 GTID 已在 `gtid_executed` 里，直接跳过（GTID 幂等）。启动时还有硬校验：`opt_log_replica_updates && replicate_same_server_id && gtid_mode != ON` 时**拒绝启动**（`ER_RPL_INFINITY_DENIED`，mysqld.cc）——匿名事务 + 双写 + 回放自己 = 必然无限循环，直接不让你起。
4. **server_id / server_uuid 必须全局唯一**：IO thread 连接时检测到与主库相同会直接停止（rpl_replica.cc 的 `ER_REPLICA_FATAL_ERROR` 分支）。server_id 用于"跳自己"，uuid 用于 GTID 命名空间——重复则一切混乱。

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
| `Replica_open_temp_tables` | 从库当前打开的临时表数（STOP 的约束之一） |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|--------|------|------|
| 每个 CHANNEL 的连接/位点/延迟 | SQL | `SHOW REPLICA STATUS FOR CHANNEL 'ch1'`；`performance_schema.replication_connection_status` / `replication_applier_status_by_coordinator`（按 CHANNEL 分组） |
| 拓扑上有多少从库在拉 | SQL | 主库 `SHOW REPLICAS`（COM_REGISTER_SLAVE 注册过的） |
| failover 前确认追平 | SQL | 对比 `SHOW REPLICA STATUS` 的 `Retrieved_Gtid_Set` vs `Executed_Gtid_Set`（差集为空 = 已追平）；或 `SOURCE_LOG_FILE/POS` 与主库 `SHOW MASTER STATUS` 对齐 |
| 过滤生效范围 | SQL | `performance_schema.replication_applier_filters`（每个 CHANNEL 生效的过滤器） |
| 延迟复制的实际延迟 | SQL | `SHOW REPLICA STATUS` 的 `SQL_Delay`；`performance_schema.replication_applier_status_by_worker` 的 `APPLYING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP` |
| 复制相关错误 | 日志/SQL | error log 的 `ER_RPL_*` 系列；`Last_IO_Errno` / `Last_SQL_Errno`（速查表见 replica.md） |

---

## Misc

### 坑与已知缺陷

**双主自增冲突**：双主各自插入自增主键会撞号。GTID 不解决数据内容冲突。常见解法：`auto_increment_increment=2, auto_increment_offset=1/2`（两边奇偶错开），或应用层分片。注意：8.0 的 `auto_increment_offset` 在集群复制场景被部分禁用（MGR 默认接管），细节见 [`../../feat/auto_increment.md`](../../feat/auto_increment.md)。

**server_id / server_uuid 重复的静默灾难**：server_id 重复导致"跳自己"机制误杀别库事件（A 的事件被 server_id 相同的 B 跳过），uuid 重复则 GTID 命名空间撞车。克隆虚拟机做从库是最常见事故源——`server_id` 必查。

**从库被误写 → GTID 分叉**：读写分离下应用误写从库，从库的 `gtid_executed` 多出主库没有的事务 → 之后任何 failover/拓扑调整触发 `ER_REPLICA_HAS_MORE_GTIDS_THAN_SOURCE`。防：`super_read_only=ON`。

**过滤器与 GTID 空洞**：过滤掉的事务 GTID 仍记入 executed（否则差集永远对不齐）→ 从库有"标记过但没数据"的事务。取消过滤不会补放。提升这样的从库为主库，数据就是缺失的——**生产上用过滤器的从库不该作为 failover 候选**，或必须接受空洞语义。

**环拓扑匿名事务无限循环**：`replicate_same_server_id=ON + log_replica_updates=ON + GTID 非 ON` 的组合在启动时被直接拒绝（`ER_RPL_INFINITY_DENIED`）。历史版本没有这层校验，社区出现过"环拓扑把 CPU 打满"的事故。

### 容易误解的命名

| 名字 | 容易误解成 | 实际 |
|------|-----------|------|
| `log_replica_updates` | "是否记录从库的更新" | 是否把**回放的事务**写进本机 binlog（与从库自身的本地写入无关） |
| `replicate_same_server_id` | "允许多个从库同 id" | 是否回放 `server_id == 本机` 的事件（防环时保持 OFF） |
| CHANNEL | 网络通道 | 复制通道：一组独立的 (IO, SQL) 线程 + 位点 + relay log 的隔离单元 |
| auto position | 自动找位点 | 用 GTID 差集定位起点（协议仍是拉模型，从库主动发集合） |
| failover | 自动切换 | MySQL 社区版**没有内置自动 failover**——`SOURCE_CONNECTION_AUTO_FAILOVER` 只换连接不选主，选主由外部编排（Orchestrator / MGR / 云平台） |

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
