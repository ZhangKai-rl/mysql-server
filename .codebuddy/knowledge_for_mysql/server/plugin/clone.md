# Clone 插件深度解析

> 基于 MySQL 8.0.39 源码。Clone（8.0.17 引入）是 MySQL 官方的**物理数据拷贝机制**，横跨三层——SQL 层（`CLONE` 命令 + 插件框架）、server 层协调（`Clone_handler`、备份锁、XA 阻塞）、InnoDB 侧的文件/页拷贝与 redo 追踪。它是云上 provisioning 与 HA 重建从库的基础设施。
>
> **边界**：
> - **本篇讲**：`CLONE` 命令与插件框架、本地 clone vs 远程 clone（donor / recipient）的架构、拷贝的三阶段（file copy / page copy / redo copy）与**一致性点的证明**、与备份锁/DDL 的协作、并发与限流、状态与失败处理、与复制的衔接（clone 后如何挂复制）。
> - **不讲（指向别处）**：备份锁（`BACKUP_LOCK`）本身的语义与两段式设计见 [`global_lock.md`](../../infra/lock/transactional/global_lock.md)；redo 与 LSN 机制见 [`redo_log.md`](../../innodb/redo_log.md)；数据文件路径与表空间见 [`fil.md`](../../innodb/fil.md)；复制拓扑与故障转移见 [`replication.md`](../replication/replication.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

Clone 插件是 MySQL 官方的**物理全量数据拷贝**机制——recipient 从 donor 拿到一份字节级一致的数据副本（数据文件 + 增量页 + redo + 元数据），用于快速建实例/建从库。它以**插件**形态存在（`MYSQL_CLONE_PLUGIN`，类型 11），却在内部大量消费**组件服务**（`clone_protocol`、`mysql_backup_lock`）——是"插件消费组件服务"这条通道最重的真实用例（见 [`service.md`](service.md) 与 [`component.md`](component.md) 的互操作小节）。

### 用途

替代 XtraBackup 做物理备份恢复；云上快速 provisioning；HA 场景重建从库；`CLONE LOCAL DATA DIRECTORY`（本地克隆到指定目录，用于快速造测试数据或物理备份）。

### 版本演进

| 版本/线索 | 变化 | 出处 |
|---|---|---|
| 8.0.17 | clone 插件引入，协议 V1（`CLONE_PROTOCOL_VERSION_V1 = 0x0100`） | `plugin/clone/include/clone.h:122`（引入版本号引自官方文档，协议常量经源码核实） |
| 协议 V2（`0x0101`） | "Send also SO names along with plugin name" → `COM_RES_PLUGIN_V2` | `clone.h:125`、`:183` |
| 协议 V3（`0x0102`） | "Send more configurations required by recipient" → `COM_RES_CONFIG_V3` | `clone.h:128`、`:186` |
| 后续 | 断线重连（`COM_REINIT`）、PFS `clone_status`/`clone_progress`、IO 限流（`clone_max_data_bandwidth`）、动态元数据持久化 | 源码结构推断，**具体小版本未核实** |
| 本分支门槛 | GR 使用 clone 要求成员版本 ≥ 8.0.37（`CLONE_GR_SUPPORT_VERSION = 0x080037`） | `plugin/group_replication/include/plugin_constants.h:83` |

---

## 理论基础

### 1. 为什么需要 clone，而不是"拷文件 + crash recovery"？

直接拷贝运行中的 InnoDB 数据文件，得到的是**跨时间窗口的页集合**：先拷的页属于时刻 T1，后拷的属于 T2，中间还有脏页没落盘。单纯 crash recovery 只能把 redo 前滚到某个 LSN，但**基线文件本身不是一个时刻的快照**，redo 无法修复"页来自不同时刻"的问题。

clone 的做法是给拷贝过程配两个"追赶"通道（详见核心实现的一致性证明）：

- **page tracking** 记录拷贝期间被修改的页号 → 第二阶段把这些页**重新拷一遍**；
- **redo archiving** 归档第二阶段的 redo → 第三阶段把 redo 补齐到一致 LSN。

于是 recipient 拿到的是"某个 LSN 时刻的完整状态"，恢复只需极短的前滚。

### 2. 为什么 clone 拿备份锁，而不是 FTWRL？

FTWRL（`FLUSH TABLES WITH READ LOCK`）挡的是 **DML**，而 clone 只需要在**文件集合稳定**的意义上挡 **DDL**（新建/删除/重命名表空间文件会让文件清单漂移）。备份锁的语义正好是"排他备份锁只与 DDL 冲突，不影响 DML"——所以 clone 期间业务仍可写。

实现上 clone **不调** SQL 层的 `acquire_shared_backup_lock()`，而是走组件服务 `mysql_backup_lock`（`include/mysql/components/services/backup_lock_service.h`）——因为插件不该直呼服务器函数（见 [`abi.md`](abi.md) 的封装纪律）。详见核心实现「与备份锁的协作」。

### 3. 为什么拷贝分三个阶段？

这是"**快照 + 增量追赶 + 日志补齐**"的通用模式（与存储快照、备份的增量合并同源）。三阶段各自解决一个约束：

| 阶段 | 解决什么 | 代价 |
|---|---|---|
| FILE COPY | 传输主体数据（耗时最长） | 期间页被修改 → 文件基线不纯 |
| PAGE COPY | 用 page tracking 补拷被改的页 | 仍可能在补拷期间再被改 |
| REDO COPY | 用归档 redo 把最后一段差异补齐到一致 LSN | 需要 donor 额外写归档 |

**为什么不再多几轮？** 因为 redo 是**顺序追加**的：一旦开始归档 redo，"追赶"就从"找散落的页"变成了"拷一段连续的日志"，一轮即可收敛——这是选择 redo 而不是继续追页的关键原因。

值得注意：源码里的快照类型其实有四种（`HA_CLONE_BLOCKING` / `HA_CLONE_REDO` / `HA_CLONE_HYBRID` / `HA_CLONE_PAGE`），clone 插件实际只用 **`HA_CLONE_HYBRID`**（`clone_local.cc:115`、`clone_server.cc:175`）；`HA_CLONE_PAGE` 源码标注 **Not Implemented**（`clone0desc.h:91`）。

### 4. 增量/一致性依赖什么开关？

- **page tracking**：进入 FILE COPY 时由 `m_page_ctx.start()` 启动（`clone0copy.cc:210`），**不需要额外的用户开关**——clone 自己启停。
- **redo archiving**：进入 PAGE COPY 时启动（`:263`），同样由 clone 自己管理。

⚠️ **骨架里曾假设的 `clone_force_blocking` 变量不存在**（全仓库 grep 0 命中）。真正相关的开关是 `clone_block_ddl`（是否加备份锁挡 DDL，默认 **false**）与 `clone_ddl_timeout`（默认 300 秒）。

### 5. 他库对比

云数据库（RDS / PolarDB）的快照与 clone 多在**存储层**完成（写时重定向 / 共享存储的卷快照），因此能做到秒级、与数据量无关；MySQL 社区 clone 是**文件级**拷贝，耗时随数据量线性增长。根本差异在存储架构（共享存储 vs 本地盘），见 [`cloud_storage.md`](../../cloud/cloud_storage.md)。这也是为什么"云上 provisioning"往往优先用存储快照，而本地盘部署才需要 clone。

---

## 核心实现

### 插件与命令

**插件声明**（`plugin/clone/src/clone_plugin.cc:650-671`）：

```c
mysql_declare_plugin(clone_plugin){
    MYSQL_CLONE_PLUGIN,
    &clone_descriptor,          /* struct Mysql_clone */
    clone_plugin_name,          /* "clone" */
    PLUGIN_AUTHOR_ORACLE,
    "CLONE PLUGIN",
    PLUGIN_LICENSE_GPL,
    plugin_clone_init,          /* init */
    plugin_clone_check,         /* check uninstall */
    plugin_clone_deinit,        /* deinit */
    CLONE_PLUGIN_VERSION,       /* 0x0100 */
    nullptr,                    /* status variables —— 没有状态变量 */
    clone_system_variables,     /* 14 个插件变量 */
    nullptr, 0,
} mysql_declare_plugin_end;
```

`Mysql_clone`（`include/mysql/plugin_clone.h:43-71`）是 1 个 `interface_version` + **3 个**函数指针（`clone_local` / `clone_client` / `clone_server`）——不是 4 个方法。绑定在 `clone_plugin.cc:510`。

`plugin_clone_init`（`:345`）做四件事，值得单独看，因为它就是"插件消费组件服务"的样板：① `init_logging_service_for_plugin`；② 经 registry 取 `mysql_backup_lock` 服务（`:355`，即 `mysql_plugin_registry_acquire()` → `my_service<...>`）；③ 取 `clone_protocol` 服务（`:362`）；④ `clone_handle_create()` + 注册 PFS 代理表。

**语法与 dispatch**（`sql/sql_yacc.yy:18143-18171`，两条产生式）：

| 语句 | 构造 | 权限 |
|---|---|---|
| `CLONE LOCAL DATA DIRECTORY = '...'` | `Sql_cmd_clone(LEX_CSTRING data_dir)` | **BACKUP_ADMIN** |
| `CLONE INSTANCE FROM u:p IDENTIFIED BY ... [DATA DIRECTORY='...']` | `Sql_cmd_clone(LEX_USER*, port, data_dir)` | 带 DATA DIRECTORY：**BACKUP_ADMIN**；不带（**替换本实例 datadir**）：**CLONE_ADMIN** |

权限检查在 `sql/sql_admin.cc:1988-2006`；`is_replace = (m_data_dir.str == nullptr)`。GR 运行时用户会话禁止替换 datadir（`ER_CLONE_DISALLOWED`），但 **`mysql.session` 用户豁免**——GR 内部的分布式恢复正是走这个豁免。

**recipient 调用栈**：

```
Sql_cmd_clone::execute()                       sql/sql_admin.cc:1975
 ├─ clone_plugin_lock(thd, &m_plugin)          sql/clone_handler.cc:324（仅 PLUGIN_IS_READY 可用）
 ├─ m_clone->clone_local(...)                  → Clone_handler::clone_local (clone_handler.cc:129)
 │     → Mysql_clone::clone_local              plugin/clone/src/clone_plugin.cc:445
 │        → myclone::Local::clone_exec         plugin/clone/src/clone_local.cc:87
 │           → hton_clone_begin/copy/apply     plugin/clone/src/clone_hton.cc
 │              → storage/innobase/clone/clone0api.cc
 └─ m_clone->clone_remote_client(...)          → myclone::Client::clone  plugin/clone/src/clone_client.cc:701
```

**donor 调用栈（易漏）**：donor 不走 `COM_QUERY`，而是 **`COM_CLONE`**——先正常回一个语句状态包，然后**把当前连接的协议切换成 clone 私有协议**：

```
dispatch_command() case COM_CLONE:            sql/sql_parse.cc:1846-1861
   clone_cmd = new Sql_cmd_clone(); clone_cmd->load(thd);   /* 动态加载插件 + BACKUP_ADMIN 检查 */
   ... thd->send_statement_status();                        /* 先回状态 */
   clone_cmd->execute_server(thd);                          sql_parse.cc:2457-2461
     → Sql_cmd_clone::execute_server()        sql/sql_admin.cc:2116
        → myclone::Server::clone              plugin/clone/src/clone_server.cc:62
```

`Clone_handler`（`sql/clone_handler.{h,cc}`）是 SQL 层与插件之间的薄适配层 + 全局单例：`m_plugin_handle = (Mysql_clone *)plugin_decl(plugin)->info`；`validate_dir()` 校验目标目录必须绝对路径、不存在、**且不在 datadir 之内**；`XA_Block` 在 REDO COPY 前阻塞外部 XA；`is_provisioning()` 供 GR 判断。

### 远程 clone 架构

| 角色 | 实现 | 关键对象 |
|---|---|---|
| donor | `myclone::Server` | socket 来自客户端连接 |
| recipient | `myclone::Client` + 本地 `myclone::Server`（apply 用） | `Client_Share`（跨线程共享 `m_storage_vec/m_threads/m_stat`） |
| 本地 clone | `myclone::Local` = Server(源) + Client(目的) 同进程直连 | 数据链路从 socket 换成 BUFFER/FILE（`Data_Link_Type`） |

即**本地克隆就是 donor 与 recipient 在同一进程内的直连**，复用同一套 `hton_clone_*` 接口。

**协议层**是 14 个方法的组件服务 `clone_protocol`（`include/mysql/components/services/clone_protocol_service.h:74-236`），分三组：会话/元信息（`start_statement`、`get_charsets`、`get_configs`…）、客户端（`mysql_clone_connect`、`send_command`、`get_response`、`disconnect`）、服务端（`get_command`、`send_response`、`send_error`）。

RPC 命令集（`plugin/clone/include/clone.h:139-160`）：`COM_INIT / COM_ATTACH / COM_REINIT / COM_EXECUTE / COM_ACK / COM_EXIT`。响应集：locators / data descriptor / data / plugin / config / collation / complete(99) / error(100)。**donor 会降级到 `min(client_ver, CLONE_PROTOCOL_VERSION)`**（`clone_server.cc:351-354`）。

### 三阶段流程与一致性点（核心难点）

**状态机**（`storage/innobase/include/clone0desc.h:53-114`，源码自带 plantuml）：

```
[*] → INIT → FILE COPY → PAGE COPY → REDO COPY → DONE
       │          │            │
       │       起 page tracking  起 redo 归并 停 redo（+XA 阻塞 + GTID 同步）
       └──────── 由 Clone_Snapshot::get_next_state() 决定下一跳
```

三个阶段的关键代码（均在 `storage/innobase/clone/clone0copy.cc`）：

```c
/* 进入 FILE COPY：起 page tracking，然后遍历所有文件建 chunk 列表 */
int Clone_Snapshot::init_file_copy(Snapshot_State new_state) {
  ...
  } else if (m_snapshot_type == HA_CLONE_HYBRID || m_snapshot_type == HA_CLONE_PAGE) {
    /* Start modified Page ID Archiving */
    err = m_page_ctx.start(false, nullptr);        /* ← L0：page tracking 开始 */
  }
  ...
  auto error = Fil_iterator::for_each_file([&](fil_node_t *file) { return add_node(file, false); });
}

/* 进入 PAGE COPY：先起 redo 归档，再停 page tracking（顺序不能反！） */
int Clone_Snapshot::init_page_copy(...) {
  if (m_snapshot_type == HA_CLONE_HYBRID) {
    err = init_redo_archiving();                   /* ← L1：redo 归档开始 */
  }
  ...
  err = m_page_ctx.stop(nullptr);                  /* ← L1'：page tracking 停止 */
  err = m_page_ctx.get_pages(add_page_callback, context, page_buffer, page_buffer_len);
  m_page_vector.assign(m_page_set.begin(), m_page_set.end());   /* 脏页 ID 集合 */
}

/* 进入 REDO COPY：停 redo 归档 + 与 binlog/GTID 对齐 */
int Clone_Snapshot::init_redo_copy(...) {
  Clone_handler::XA_Block xa_block_guard(thd);     /* 阻塞外部 XA */
  if (binlog_error == 0) binlog_error = synchronize_binlog_gtid(cbk);
  dict_persist_t::Enable_immediate dyn_metadata_guard(dict_persist);  /* 动态元数据立即落 DD */
  auto redo_error = m_redo_ctx.stop(m_redo_trailer, m_redo_trailer_size, m_redo_trailer_offset);  /* ← L2 */
  ...
}
```

**一致性点的证明**（把上面三个时刻串起来）：

| 时刻 | 事件 | 代码 |
|---|---|---|
| L0 | page tracking 启动，开始拷文件 | `init_file_copy` → `m_page_ctx.start()` |
| L1 | redo archiving 启动 | `init_page_copy` → `init_redo_archiving()` |
| L1' | page tracking 停止，导出脏页 ID 集合 | `m_page_ctx.stop()` + `get_pages()` |
| L2 | redo archiving 停止 | `init_redo_copy` → `m_redo_ctx.stop()` |

因为 **`L1 ≤ L1'`**（`init_redo_archiving()` 在 `m_page_ctx.stop()` **之前**调用）：

```
[L0, L1'] 的修改  ⊆  脏页集合（第二阶段重拷）
[L1, L2]  的修改  ⊆  redo 归档          且 L1 ≤ L1'
⇒  [L0, L2] 的所有修改都被覆盖，无空隙
```

recipient 侧"基线文件 + 脏页重放 + 从 `m_begin_lsn` 起的 redo 前滚"= **LSN = L2 时刻的一致状态**。

**L2 之前那三件事为什么必要**：① `XA_Block`——否则 XA 的 GTID 在 binlog 与 InnoDB 之间乱序；② `synchronize_binlog_gtid()` + `update_binlog_position()`——把 donor 的 binlog 位点**写进 InnoDB trx sys 页**，随数据一起过去（见「与复制衔接」）；③ `dict_persist_t::Enable_immediate`——动态元数据立即落 DD，使恢复不依赖 redo 里的动态元数据。

**状态迁移是 barrier 协议**（`Clone_Task_Manager::change_state`，`clone0clone.cc:1767`）：多个 worker 任务各自拷 chunk，第一个发现"本阶段取完"的任务设 `m_num_tasks_transit = m_num_tasks`；后续任务到达就 `--`；**只有 master 且只剩自己时**才真正执行 `Clone_Snapshot::change_state()`（即启停归档）。等待期间 donor 周期性 `send_keep_alive`，避免 recipient 网络超时。

**Page Archiving 的物理细节**：脏页 ID 存在 `Clone_Page_Set m_page_set`（去重，日志会打印 duplicate pages）；归档落盘路径 `#ib_archive/page_group_<lsn>/ib_page_<n>`；归档器内存是 **32 × 16 KiB = 512 KiB** 的环形块缓冲（`ARCH_PAGE_NUM_BLKS=32` × `UNIV_PAGE_SIZE_DEF`），每块可存 `(16384-32)/8 = 2044` 个页 ID。⚠️ **"脏页集合的内存上限/降级策略"在源码中未找到**（只有分配失败的 `ER_OUTOFMEMORY` 路径）——诚实标注。

**Redo Archiving**：起点 = 进 PAGE COPY 那一刻的 LSN（`Log_Arch_Client_Ctx::start` 记 `m_begin_lsn`），终点 L2（`stop` 记 `m_end_lsn`），文件路径 `#ib_archive/log_group_<lsn>/ib_log_<n>`。

### 本地 clone 的差异

`CLONE LOCAL DATA DIRECTORY` 走 `myclone::Local`，同进程内 Server→Client 直连：

- **不经过网络**（数据链路是 BUFFER/FILE），但**仍走三阶段**（同样需要 page/redo 归档，因为源就是运行中的实例）；
- **仍需备份锁**：`clone_local.cc:103` 处 `mysql_service_mysql_backup_lock->acquire(...)`，且只有 `is_master && clone_block_ddl` 时才拿；
- 权限是 **BACKUP_ADMIN**（不是 CLONE_ADMIN，因为不替换 datadir）。

### 与备份锁的协作

clone 通过组件服务拿锁，三处调用点：

```
本地：    clone_local.cc:103   （条件：is_master && clone_block_ddl）
donor：   clone_server.cc:156  （条件：block_ddl()，由 recipient 传来的标志决定）
recipient：clone_client.cc:1629（仅当替换自身 datadir，无条件拿）
```

**一个反直觉的细节**：donor **是否**加备份锁，由 **recipient 的 `clone_block_ddl`** 决定，而不是 donor 自己的。传递方式是把标志位复用进 `clone_ddl_timeout` 的最高位：

```c
/* clone_client.cc:1360 */
uint32_t timeout_value = clone_ddl_timeout;
if (!clone_block_ddl) timeout_value |= NO_BACKUP_LOCK_FLAG;   /* 1ULL << 31 */
int4store(buf_ptr, timeout_value);
/* clone_server.h:134 —— donor 侧解出 */
m_backup_lock = ((client_timeout & NO_BACKUP_LOCK_FLAG) == 0);
m_client_ddl_timeout = client_timeout & ~NO_BACKUP_LOCK_FLAG;
```

这是"不改协议版本就能加能力"的位复用技巧（与"服务改名版本化"是同一种 ABI 保守主义，见 [`abi.md`](abi.md)）。

**DDL 阻塞有互补的两套机制**：

1. **备份锁**（SQL 层，`clone_block_ddl=ON` 时）：挡住需要排他备份锁的 DDL，不影响 DML。
2. **`Clone_notify`**（InnoDB 层，**永远生效**，与备份锁无关）：DDL 在关键段构造 `Clone_notify` guard（`clone0api.h:178-206`，12 种通知类型：SPACE_CREATE/DROP/RENAME/IMPORT/ALTER_INPLACE/ALTER_ENCRYPT…）。
   - FILE COPY 阶段：DDL 的文件操作会 **block 住 clone 的状态迁移**，并用 `.#clone` / `.#clone_save` / `.#clone_ddl` 扩展名记录待处理文件；
   - PAGE COPY 阶段：**bulk 操作**（不写 redo）与**加密属性变更**必须等 PAGE COPY 结束——否则"增量页 + redo"方案会漏数据或解不开 donor 主密钥加密的 redo；
   - 协调失败则 `Clone_Sys::mark_abort(true)` → clone 报 `ER_CLONE_DDL_IN_PROGRESS`。

### 并发与限流

**一个节点同时只能跑一个 clone**（源码三处上限）：

```c
const int MAX_SNAPSHOTS = 1;              /* clone0clone.h:136 */
const int MAX_CLONES = 1;
const uint MAX_CLONES_PER_SNAPSHOT = 1;   /* clone0snapshot.cc:43 */
const int CLONE_MAX_TASKS = 128;          /* clone0desc.h:51 */
```

recipient 侧并发检查在 `Client::pfs_begin_state()`（`clone_client.cc:567`），第二个 clone 直接报 `ER_CLONE_TOO_MANY_CONCURRENT_CLONES`。注意 provisioning（替换 datadir）成功后**不**清零 `s_num_clones`，这样重启期间也不允许新 clone。

**线程与分片**：worker 数 = `clone_max_concurrency - 1`（master 也干活），默认 **`clone_max_concurrency = 16`**（`CLONE_DEF_CON`，`clone_os.h:42`）。⚠️ 源码注释写的是 "Default = 8 threads"，**与常量不符，实际默认 16**。数据按 **chunk** 切分，任务经 `Clone_Task_Manager::reserve_next_chunk()` 原子抢占。`clone_autotune_concurrency`（默认 true）开启时由 `Client::update_stat()` 动态加线程，**只增不减**。

**限流**：`clone_max_network_bandwidth`（网络，MiB/s）与 `clone_max_data_bandwidth`（IO，MiB/s）默认均为 0（不限）。限流还会反作用于线程数——`Client::limit_workers()` 保证每 worker 至少 64 MiB/s：`num_workers = min(num_workers, bandwidth / 64)`。

### 状态与失败处理

| 场景 | 行为 |
|---|---|
| 克隆到**新目录** | 目录必须不存在（`validate_dir`）；失败由 `clone_file_roll_back` / `clone_old_file_roll_back` 按扩展名回滚 |
| **替换 datadir**（provisioning） | 先 `clone_drop_user_data()`（drop 所有用户表/schema/tablespace + `RESET MASTER`），**此后无回滚**；成功必须重启 `signal_restart_server()`，失败直接 `kill_mysql()` |
| 网络中断 | `Client::clone()` 的 do-while 支持断点重连，上限 `CLONE_MAX_RESTART = 100`；重连发 `COM_REINIT`，donor 允许窗口由 `clone_donor_timeout_after_network_failure`（默认 **5 分钟**）控制 |

错误传递：`Clone_Task_Manager::set_error()` 保存 `m_saved_error`，且**网络错误可被任何非网络错误覆盖**（因为网络错误会触发重连等待，若已有真实错误就不该等）；`handle_error_other_task()` 负责把任一 worker 的错误传播到主线程；donor → recipient 通过 `COM_RES_ERROR`，再由 `Clone_handler::get_donor_error()` 从 DA 文本反解出真实 errno。

### 与复制的衔接（一个很容易猜错的点）

**clone 协议里没有"位点描述符"字段**——donor 的 binlog 位点是被**写进 InnoDB 的 trx sys 页**，随数据一起克隆过去的：

```c
/* donor：clone0copy.cc:350 update_binlog_position() */
trx_sys_read_binlog_position(&file_name[0], file_pos);     /* ① InnoDB 已记的位点 P1 */
mysql_bin_log.get_current_log(&log_info);                  /* ② binlog 当前位点 P2 */
bool written = trx_sys_write_binlog_position(&file_name[0], file_pos,
                                             &log_info.log_file_name[0], log_info.pos);
if (written) wait_for_binlog_prepared_trx();               /* ④ 写入过则再等 prepared 事务收尾 */
```

recipient 恢复后从 trx sys 页读出，追加 GTID，一并写进 `#clone/status_recovery`，最终暴露为 `performance_schema.clone_status` 的 `BINLOG_FILE` / `BINLOG_POSITION` / `GTID_EXECUTED`。

**clone 插件本身不含任何 `CHANGE REPLICATION SOURCE` 构造代码**（`plugin/clone/` 全目录 grep `gtid|binlog` 只命中 `clone_status.{cc,h}`）——挂复制是运维/上层自动化的事：

```sql
SELECT BINLOG_FILE, BINLOG_POSITION, GTID_EXECUTED FROM performance_schema.clone_status;
CHANGE REPLICATION SOURCE TO SOURCE_HOST=..., SOURCE_LOG_FILE=..., SOURCE_LOG_POS=...;
-- 或 SET @@GLOBAL.GTID_PURGED = '<GTID_EXECUTED>';
```

**Group Replication**：`Remote_clone_handler`（`plugin/group_replication/src/plugin_handlers/remote_clone_handler.cc`）决定 `DO_CLONE` / `DO_RECOVERY`（阈值突破或 donor GTID 已被 purge 时走 clone）；provisioning 期间 `Clone_handler::is_provisioning()` 为真，拒绝 `START GROUP_REPLICATION`。

---

## 可观测性

**两张 PFS 表其实是插件代理表**（不是 `storage/perfschema/` 原生表），由 clone 插件经 `pfs_plugin_table_service` 注册（`plugin/clone/src/clone_status.cc:326` / `:606`）：

| 表 | 字段要点 | 持久化 |
|---|---|---|
| `performance_schema.clone_status` | ID / PID / STATE / BEGIN_TIME / END_TIME / SOURCE / DESTINATION / ERROR_NO / ERROR_MESSAGE / **BINLOG_FILE / BINLOG_POSITION / GTID_EXECUTED** | `#clone/view_status` |
| `performance_schema.clone_progress` | 阶段、状态、已拷字节、预估速度 | `#clone/view_progress` |

阶段枚举（`clone_status.h:136`）比 InnoDB 的三阶段更细，共 8 个：`STAGE_CLEANUP / FILE_COPY / PAGE_COPY / REDO_COPY / FILE_SYNC / RESTART / RECOVERY`——多出来的三个是 recipient **apply 侧**的收尾（文件同步、重启、崩溃恢复），这说明"PFS 的阶段"与"donor 的快照阶段"不是同一套。

**系统变量（14 个，全部是插件变量，不在 `sql/sys_vars.cc`）**：

| 变量 | 默认 | 说明 |
|---|---|---|
| `clone_block_ddl` | **false** | 是否加备份锁挡 DDL（默认不挡） |
| `clone_ddl_timeout` | **300** 秒 | 等备份锁/DDL 的超时 |
| `clone_max_concurrency` | **16**（注释误写 8） | 1..128 |
| `clone_autotune_concurrency` | **true** | 动态加线程（只增不减） |
| `clone_max_network_bandwidth` | 0（不限） | MiB/s，每 worker 保证 64 MiB/s |
| `clone_max_data_bandwidth` | 0（不限） | IO 限流（C 变量名是 `clone_max_io_bandwidth`） |
| **`clone_enable_compression`** | **false** | 网络压缩（**不是** `clone_network_compression`） |
| `clone_buffer_size` | 4 MiB | 1..256 MiB |
| `clone_valid_donor_list` | NULL | donor 白名单（`check_donor_addr_format`） |
| `clone_ssl_key/cert/ca` | NULL | SSL |
| `clone_donor_timeout_after_network_failure` | **5** 分钟 | 断线后 donor 等待窗口（C 变量 `clone_restart_timeout`） |
| `clone_delay_after_data_drop` | 0 秒 | 0..3600 |

---

### clone 的两个独立压缩机制（别混淆，可叠加）

> 全景对照见 [`../feat/compression.md`](../../feat/compression.md)。

| | 网络压缩 | 文件重压缩/打洞 |
|---|---|---|
| 开关 | `clone_enable_compression`（默认 false） | 表空间透明页压缩（`m_punch_hole`）自动 |
| 层级 | 连接级，整包压缩 | 页级，仅页 payload |
| 算法/级别 | 复用经典协议压缩（zlib/zstd，`my_compress`） | `os_file_compress_page`（zlib/LZ4，`page_zip_level`） |
| 目的 | 省网络带宽 | 省 recipient 磁盘（还原稀疏文件） |

#### ① 网络压缩：复用经典协议压缩的挂接

变量（`clone_plugin.cc:69`）`bool clone_enable_compression;`，sysvar 注释 "If compression is done at network"。客户端 `Client::connect_remote`（`clone_client.cc:889-896`）：

```cpp
int Client::connect_remote(bool is_restart, bool use_aux) {
  mysql_clone_ssl_context ssl_context;
  ssl_context.m_enable_compression = clone_enable_compression;
  ssl_context.m_server_extn =
      ssl_context.m_enable_compression ? &m_conn_server_extn : nullptr;
  ssl_context.m_ssl_mode = m_share->m_ssl_mode;
```

`m_conn_server_extn` 是 `NET_SERVER` 成员（`clone_client.h:752`），构造时 `net_server_ext_init(&m_conn_server_extn)`（`clone_client.cc:468`）。真正的挂接在服务组件 `clone_protocol_service.cc:492-496`：

```cpp
  /* Enable compression. */
  if (ssl_ctx->m_enable_compression) {
    mysql_options(mysql, MYSQL_OPT_COMPRESS, nullptr);       // 握手带 CLIENT_COMPRESS
    mysql_extension_set_server_extn(mysql, ssl_ctx->m_server_extn);  // 挂 compress_ctx
  }
```

clone 未单独指定算法，用连接默认（`protocol_compression_algorithms` 协商）；协商成功后 `sql_connect.cc:780-786` 把算法装载进 `compress_ctx`，此后所有包走 `my_compress`/`my_uncompress`（实现见 [`authentication.md`](../auth/authentication.md) 压缩节）。

#### ② 文件重压缩/打洞：donor 重压、recipient 还原稀疏文件

**为什么必须重压**：表启用 TPC 时磁盘是"压缩字节 + 稀疏空洞"布局，clone 读到的是**解压后的完整逻辑页**（16K），空洞不是字节、无法经网络传输——donor 按原算法重压一遍，把压缩长度随页头（`FIL_PAGE_COMPRESSED` + `FIL_PAGE_COMPRESS_SIZE_V1`）一起发出。

**donor**（`clone0snapshot.cc:948-967`）：

```cpp
  /* Do transparent page compression if needed. */
  if (page_id.page_no() != 0 && file_meta->m_punch_hole &&
      file_meta->m_compress_type != Compression::NONE) {
    auto compressed_data = page_data + data_size;
    memset(compressed_data, 0, data_size);
    IORequest request(IORequest::WRITE);
    request.compression_algorithm(file_meta->m_compress_type);
    ulint compressed_len = 0;
    auto buf_ptr = os_file_compress_page(
        request.compression_algorithm(), file_meta->m_fsblk_size, page_data,
        data_size, compressed_data, &compressed_len);
    if (buf_ptr != page_data) {              // 真的压缩了（压不动返回原页）
      encrypted_data = page_data;
      page_data = compressed_data;
      data_len = static_cast<uint>(compressed_len);   // 只传压缩长度
    }
  }
```

**recipient** `sparse_file_write`（`clone0apply.cc:1266-1331`，节选核心循环）：

```cpp
  while (len >= page_len) {
    uint32_t comp_len;
    bool is_compressed = read_compressed_len(
        buffer, len, static_cast<uint32_t>(file_meta->m_fsblk_size), comp_len);
    auto write_len = is_compressed ? comp_len : page_len;
    bool first_page = (start_off == 0);
    if (first_page || write_len > page_len) write_len = page_len;   // 页 0 不压缩

    err = os_file_write(request, "Clone data file", file, buffer, start_off,
                        (start_off == 0) ? page_len : write_len);
    os_offset_t offset = start_off + write_len;
    os_offset_t hole_size = page_len - write_len;
    if (file_meta->m_punch_hole && hole_size > 0) {
      err = os_file_punch_hole(file.m_file, offset, hole_size);
      if (err != DB_SUCCESS) {
        file_meta->m_punch_hole = false;     // 整文件降级，不再打洞
        ib::info(ER_IB_CLONE_PUNCH_HOLE) << "Innodb Clone Apply failed to punch hole: ...";
      }
    }
    start_off += page_len; buffer += page_len; len -= page_len;
  }
```

`read_compressed_len`（:1245-1264）从 `FIL_PAGE_COMPRESSED`/`FIL_PAGE_COMPRESSED_AND_ENCRYPTED` 页头读 `FIL_PAGE_COMPRESS_SIZE_V1` 并按 donor FS block size 对齐。

**启用判定 `set_compression`**（:1009-1039）：压缩表 + `donor fsblk*2 <= srv_page_size` + recipient 支持打洞且 `recipient fsblock*2 <= srv_page_size` 才置 `m_punch_hole=true`；**加密场景下 donor/recipient FS block 大小必须一致**（:1043-1051）。

**正交叠加**：donor 先页重压缩产出"压缩页+长度"，整条字节流再被网络层压缩传输——内层页压缩、外层网络压缩。

---

## Misc

### ★ 工程实现技法

1. **barrier 式状态迁移**：多线程拷贝下，启停归档这种"全局动作"必须由最后一个到达的线程（master）执行，`m_num_tasks_transit` 就是计数器屏障。
2. **位复用扩展协议**：`NO_BACKUP_LOCK_FLAG = 1<<31` 与 timeout 复用同一个字段——不改协议版本也能加能力（与"服务改名版本化"同源的 ABI 保守主义）。
3. **一致性点用"区间覆盖"证明**：`L1 ≤ L1'` 这个顺序约束是整个算法正确性的支点，代码里只是两行调用的先后。
4. **位点随数据走**：把 binlog 位点写进 trx sys 页而不是协议字段——恢复后天然可用，无需额外握手。
5. **插件消费组件服务**：clone 不直呼服务器函数，备份锁与协议都经 registry 获取，是"新插件写法"的样板。

### 失效与退化场景

- **版本/配置不兼容**：协议版本降级、字符集/页大小/加密配置校验失败（`COM_INIT` 阶段）。
- **DDL 并发**：`Clone_notify` 无法协调时报 `ER_CLONE_DDL_IN_PROGRESS`；bulk/加密 DDL 在 PAGE COPY 阶段只能等待（最多 5 分钟保护性超时）。
- **网络限速下的超长耗时**：带宽 100 MiB/s 时 worker 数被压到 1（`limit_workers` 的 64 MiB/s 下限）。
- **provisioning 失败即灾难**：数据已 drop，只能重启（失败则 `kill_mysql()`）——**无回滚路径**。
- **并发限制**：`MAX_CLONES = 1`，第二个 clone 直接被拒；GR provisioning 期间连 `START GROUP_REPLICATION` 都被拒。

### 易混淆点

| 对比 | 结论 |
|---|---|
| clone vs mysqldump | clone 是**物理**拷贝（字节级、含 redo、快）；mysqldump 是逻辑导出（可跨版本/跨平台、慢） |
| clone vs XtraBackup | 同为物理备份；clone 是**官方内置**且支持远程 provision，XtraBackup 是外部工具、增量备份体系更成熟 |
| `clone_max_concurrency` 默认值 | **16**（源码注释写 8，是错的） |
| `clone_network_compression` | **不存在**，正确名 `clone_enable_compression` |
| `clone_max_io_bandwidth` | 作为 SQL 变量名**不存在**，SQL 名是 `clone_max_data_bandwidth` |
| `clone_force_blocking` | **不存在**（全仓库 0 命中） |

**社区边界澄清**：clone 是**社区版**功能（GPL），插件源码在 `plugin/clone/`；但 `component_keyring_encrypted_file` 等企业版组件不影响 clone 本身。clone **不**自动挂复制（需手工 `CHANGE REPLICATION SOURCE`），这是社区实现的边界而非缺失。

### 一句话总结

Clone 的本质是"**用 page tracking + redo 归档把一次跨时间的文件拷贝，收敛成一个 LSN 上的一致快照**"，再借插件框架与组件服务把它接进 MySQL 的扩展体系——三阶段是手段，一致性点（`L1 ≤ L1'`）是支点，备份锁与 `Clone_notify` 是与并发 DDL 共存的代价。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → The Clone Plugin*
- *MySQL 8.0 Reference Manual → CLONE Statement*

**内核月报 / 技术文章**
- [MySQL · 引擎特性 · clone_plugin](http://mysql.taobao.org/monthly/2019/08/05/)（kongzhi）：clone 的快照一致性三段式与远程协议源码走读——本篇的主要起点之一
- [MySQL · 引擎特性 · 初探 Clone Plugin](http://mysql.taobao.org/monthly/2019/09/02/)（weixiang）：功能与阶段概览

**相关文档**
- 备份锁与 FTWRL：[`global_lock.md`](../../infra/lock/transactional/global_lock.md)
- redo 与 LSN：[`redo_log.md`](../../innodb/redo_log.md)
- 表空间与文件层：[`fil.md`](../../innodb/fil.md)
- 复制拓扑与故障转移：[`replication.md`](../replication/replication.md)
- 云存储（快照为何能做到秒级）：[`cloud_storage.md`](../../cloud/cloud_storage.md)
- 插件框架：[`plugin.md`](plugin.md)；组件服务与桥：[`component.md`](component.md)、[`service.md`](service.md)
