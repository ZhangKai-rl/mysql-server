# MySQL 8.0.39 源码知识库

> 基于 MySQL 8.0.39 源码的深度解析文档。重点讲设计思想、理论溯源与核心实现；结论均经源码核实。

## 从哪开始

**想理解一条 SQL 怎么执行** → [`server/query/README.md`](server/query/README.md)（主链全景 + 6 层中间表示图）

**想查某个具体机制** → 用下面的目录索引

**★ 要新增或修改一篇文档** → 动笔前必读：① 本文「如何新增文档」（放哪、拆不拆、编号）② 本文「准入标准」③ [`_template.md`](_template.md)（单篇骨架 + 动笔自检）④ 先扫目录索引 + 读相关篇「边界」行，避免重复。

---

## 一条 SQL 的完整旅程（跨目录）

```
客户端
  │
  ▼
┌──────────────────────────────────────────────────┐
│ server/query/    协议 → 解析 → 优化 → 迭代器执行  │  ← 主链，52 篇
└──────────────────┬───────────────────────────────┘
                   │ ha_rnd_next / ha_index_next
┌──────────────────▼───────────────────────────────┐
│ server/handler.md   handler / handlerton 分界面   │
└──────────────────┬───────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────┐
│ innodb/          buffer pool / MVCC / 锁 / 日志   │
└──────────────────┬───────────────────────────────┘
                   │ 提交
┌──────────────────▼───────────────────────────────┐
│ server/replication/   binlog 组提交 → 复制         │
└──────────────────────────────────────────────────┘
```

---

## 目录索引

> **顶层分类轴**：`server/`、`innodb/`、`feat/` 是**源码知识**（按代码位置 / 跨层特性）；`infra/`、`index/` 是**横切主题**（基础设施 = 数据结构+锁；索引 = 横跨 SQL 语法/优化器/引擎存储）；`cloud/`、`papers/` 是**外部知识**（部署环境 / 论文剖析），以源码库之外的材料为事实来源。
>
> 条目风格：**文件名 + 一句话定位**。机制细节在正文里，不进索引（见「收尾检查清单」第 1 条）。

### server/ —— Server 层

**SQL 处理主链**：见 **[query/README.md](server/query/README.md)**（45 篇 = 主链 13 篇（含 [`08_access_path.md`](server/query/08_access_path.md)）+ [`07_optimize/`](server/query/07_optimize/) 优化器 24 篇 + [`runtime/`](server/query/runtime/) 执行期专题 8 篇；主链：协议分发 → 解析 → contextualize → prepare → 优化 → AccessPath → 迭代器执行 → DML）

| 文件 / 目录 | 内容 |
|---|---|
| [table.md](server/table.md) | 表：DD → `TABLE_SHARE` → `TABLE` → `Table_ref` 四层表示、表缓存与通用操作 |
| [handler.md](server/handler.md) | server ↔ 存储引擎分界面：handler/handlerton 分层 |
| [lifecycle/](server/lifecycle/) | 服务器生命周期 3 篇：initialize / startup / shutdown |
| [auth/](server/auth/) | 认证与授权：authentication / authorization / account_management（导航见 [`auth/README.md`](server/auth/README.md)） |
| [replication/](server/replication/) | 复制：binlog 格式与组提交、GTID、MTS 并行复制、从库线程与 relay log、半同步 |
| [xa.md](server/xa.md) | 外部 XA 事务：X/Open DTP、状态机、恢复态与 GTID 分配 |
| [stored_program.md](server/stored_program.md) | 存储程序（占位篇）：存储过程/函数/触发器/事件 |
| [view.md](server/view.md) | 视图（占位篇）：dd::View、可更新判定、WITH CHECK OPTION |
| [datatype/](server/datatype/) | 数据类型：json、gis |
| [dd/](server/dd/) | 数据字典（两套并存）：8.0 权威 DD 与引擎侧 `dict_sys` |
| [infra/](server/infra/) | 通用机制：dbug、pfs、vio、variables、encoding、charset、io_cache、mtr、build、reading_guide |
| [plugin/](server/plugin/) | 插件/组件/服务框架（入口 [`architecture.md`](server/plugin/architecture.md)）+ 具体插件：clone、mysqlx、keyring、validate_password |
| ~~[logging/](server/logging/)~~ | ⚠️ 已整体迁入 [`infra/observability/logging/`](infra/observability/logging/)（general_log / error_log / slow_log 均为观测手段） |

### [innodb/](innodb/) —— 存储引擎

> **物理结构**（表空间/页/行/LOB）集中在 [`innodb/physical/`](innodb/physical/) 子目录。

| 文件 | 内容 |
|------|------|
| [buffer_pool.md](innodb/buffer_pool.md) | Buffer Pool：页状态机、读路径、脏页刷盘、LRU 替换、预读 |
| [dblwr.md](innodb/dblwr.md) | doublewrite：文件布局、写流程、崩溃恢复与 torn page |
| [mvcc.md](innodb/mvcc.md) | MVCC：ReadView、undo 版本链、半一致性读、purge 闭环 |
| [row_search.md](innodb/row_search.md) | 取行主链 `row_search_mvcc` 与游标推进（承接 handler.md） |
| [row_ops.md](innodb/row_ops.md) | **行操作层状态机**：que 驱动协议、INSERT/UPDATE 状态序列、乐观→悲观 mode 升级、锁等待断点续传、undo 先于改页与回滚逆状态机 |
| [physical/tablespace.md](innodb/physical/tablespace.md) | 表空间物理结构与空间管理 |
| [physical/page_structure.md](innodb/physical/page_structure.md) | 页物理结构与全页类型清单 |
| [physical/record.md](innodb/physical/record.md) | 行记录格式与 offsets 协议 |
| [physical/lob.md](innodb/physical/lob.md) | 大对象存储 |
| [redo_log.md](innodb/redo_log.md) | redo：格式、LSN、mtr、I/O 路径、组提交、checkpoint |
| [recovery.md](innodb/recovery.md) | 崩溃恢复：redo 前滚 / undo 回滚两阶段 |
| [undo_log.md](innodb/undo_log.md) | undo 表空间与 purge |
| [query_graph.md](innodb/query_graph.md) | InnoDB 内部执行模型（fork/thr/node） |
| [parallel_scan.md](innodb/parallel_scan.md) | InnoDB 内部并行扫描（`Parallel_reader`） |
| [fil.md](innodb/fil.md) | 表空间与文件层（fil） |
| [io.md](innodb/io.md) | MySQL I/O 全景（从 SQL 到系统调用，详写已归位各专篇） |
| [trx.md](innodb/trx.md) | InnoDB 事务（`trx_t`） |

### [feat/](feat/) —— 跨层主题（server ↔ InnoDB）深度剖析

> 同时需要 server 与引擎两边源码才能讲清的主题归这里，**一篇写完两层**。判据见「第一步」与「边界归属」；篇内骨架见 [`feat/README.md`](feat/README.md)。

| 文件 | 内容 |
|---|---|
| [partitioning.md](feat/partitioning.md) | 分区表（全链路单篇） |
| [fts.md](feat/fts.md) | 全文检索（全链路单篇） |
| [auto_increment.md](feat/auto_increment.md) | 自增列（全链路单篇） |
| [generated_columns.md](feat/generated_columns.md) | 生成列（全链路单篇） |
| [ddl/](feat/ddl/) | **DDL 主题目录（9 篇）**：[ddl.md](feat/ddl/ddl.md) 主篇＝两层公共骨架（三种算法选择策略 + MDL 三段式 + DROP/TRUNCATE/RENAME + 行为速查 + 观测）；[create_table.md](feat/ddl/create_table.md) ＝ CREATE TABLE 全链路（server 编排 + 引擎侧建 .ibd / `btr_create` 建根页 / 统计置空）；[alter_table.md](feat/ddl/alter_table.md) ＝ ALTER TABLE 编排主线（★ 含 `mysql_alter_table` 剩余四段全剖析：外键 MDL 大集合 / 新表定义落地 / `mysql_inplace_alter_table` 18 步全流程 / 收尾与失败清理；三种算法各自分篇）；[atomic_ddl.md](feat/ddl/atomic_ddl.md) ＝ ★★ 原子 DDL（server 编排 + 引擎 DDL log + 崩溃恢复闭环，所有 DDL 共用的横切保障）；[copy_ddl.md](feat/ddl/copy_ddl.md)、[inplace_ddl.md](feat/ddl/inplace_ddl.md)、[instant_ddl.md](feat/ddl/instant_ddl.md) 三种算法各一篇（**每篇均写完两层**）；[ddl_engine.md](feat/ddl/ddl_engine.md) ＝ 8.0 新版 DDL 引擎与并行构建（并行不另立篇）；[online_ddl_tools.md](feat/ddl/online_ddl_tools.md) ＝ gh-ost / pt-osc 在线变更工具（外部知识） |
| [sdi.md](feat/sdi.md) | **SDI 序列化字典信息**：紧凑 JSON + 伪装内存表；★ server 只写不读，读靠 `IMPORT`/`ibd2sdi` |
| [compression.md](feat/compression.md) | **压缩全景地图**：9 机制 × 3 算法库总表、算法接入清单、Percona/MariaDB 特性不存在清单、常见误解；深度落各归属文档 |
| [check_table.md](feat/check_table.md) | **CHECK TABLE 全链路**：ADMIN 语句家族与四档语义、`ha_innobase::check` 三段流程、行级扫描与重复键检测、Msg_type 聚合、REPAIR 边界（InnoDB 无 repair） |

### [infra/](infra/) —— 跨层基础设施（数据结构 + 锁 + 线程 + 内存）

> 横跨 `include/`、`mysys/`、`sql/`、`innodb/` 的基础设施之家，五大子域：**数据结构**（`structure/`，按类型一篇一主题、跨层同主题一篇对比）、**锁与同步**（`lock/`，同步原语 `primitives/` + 事务锁 `transactional/`）、**线程模型**（`threading/`）、**内存**（`memory/`）、**观测体系**（`observability/`，按「观测手段」组织）。导航见 [`infra/README.md`](infra/README.md)、[`infra/lock/README.md`](infra/lock/README.md) 与 [`infra/observability/README.md`](infra/observability/README.md)。

**数据结构（structure/）**

| 文件 | 类别 | 内容 |
|------|------|------|
| [list.md](infra/structure/list.md) | 链表 | 链表五实现全盘点（侵入式 / 非侵入式 / C LIST） |
| [hash.md](infra/structure/hash.md) | 哈希 | 哈希三实现光谱：无锁 LF_HASH、无锁开放寻址、有锁分片 |
| [queue.md](infra/structure/queue.md) | 队列 | 有界 MPMC 队列两种裁剪（server vs InnoDB） |
| [counter.md](infra/structure/counter.md) | 计数器 | 分片计数器两代（`ib_counter_t` vs `Counter::Shards`） |
| [link_buf.md](infra/structure/link_buf.md) | 链缓冲 | `Link_buf` 无锁环形链缓冲 |
| [data_repr.md](infra/structure/data_repr.md) | 行数据表示 | 一行数据在 server → handler → InnoDB 各层的表示链路 |

**锁与同步（lock/）**

| 文件 | 锁类别 | 内容 |
|------|--------|------|
| [mysys_primitives.md](infra/lock/primitives/mysys_primitives.md) | 同步原语 | server 侧原语族：三层封装、PFS 埋点、SAFE_MUTEX、prlock |
| [innodb_sync.md](infra/lock/primitives/innodb_sync.md) | 同步原语 | InnoDB 自研原语族：os_event、PolicyMutex、`rw_lock_t` 三态、锁序校验 |
| [rcu.md](infra/lock/primitives/rcu.md) | 同步原语 | RCU：`MyRcuLock<T>` 逐行剖析与 SSL context 场景 |
| [seq_lock.md](infra/lock/primitives/seq_lock.md) | 同步原语 | `Seq_lock` 回调式 seqlock |
| [global_lock.md](infra/lock/transactional/global_lock.md) | 事务锁 | 全局锁二件套：FTWRL 与备份锁 |
| [mdl.md](infra/lock/transactional/mdl.md) | 事务锁 | MDL 元数据锁：锁类型、双兼容矩阵、死锁检测 |
| [innodb_trx_lock.md](infra/lock/transactional/innodb_trx_lock.md) | 事务锁 | InnoDB 事务锁：表锁/行锁/谓词锁/AUTOINC、死锁检测 |
| [thr_lock.md](infra/lock/transactional/thr_lock.md) | 事务锁 | server 层表锁 THR_LOCK：排序代替死锁检测 |

**线程模型（threading/）**

| 文件 | 内容 |
|------|------|
| [thd.md](infra/threading/thd.md) | THD 对象深度剖析（连接/线程/会话三合一载体） |
| [internal_thd.md](infra/threading/internal_thd.md) | 内部 THD（后台线程持有的降级会话） |
| [stack_depth.md](infra/threading/stack_depth.md) | 栈深度治理：`thread_stack`、软件栈检查、检查点地图、栈 vs MEM_ROOT |
| [thread_lifecycle.md](infra/threading/thread_lifecycle.md) | 线程生命周期：创建·注册·停止·回收 |
| [threading_architecture.md](infra/threading/threading_architecture.md) | 全栈线程模型：server 连接线程 + InnoDB 后台线程全景 |
| [connection_threading.md](infra/threading/connection_threading.md) | 连接线程模型（Per_thread）：线程缓存与接纳循环 |
| [thread_pool.md](infra/threading/thread_pool.md) | 线程池（MySQL 26.7 源码级） |
| [thread_wait.md](infra/threading/thread_wait.md) | 线程等待登记与唤醒协议：`current_cond`/`current_mutex` 双字段、漏杀窗口、PFS stage 上报 |
| [thread_pool_mariadb_percona.md](infra/threading/thread_pool_mariadb_percona.md) | MariaDB/Percona 线程池源码级对比 |

**观测体系（observability/）** —— 按「观测手段」组织（不是按"XX 怎么观测"；后者留在各主题篇）。地图与选型决策树见 [`infra/observability/README.md`](infra/observability/README.md)

| 文件 | 内容 |
|------|------|
| [i_s.md](infra/observability/i_s.md) | **INFORMATION_SCHEMA 虚拟表**：server `ST_SCHEMA_TABLE` 插件框架 + InnoDB 26 张表；I_S / PFS / SHOW 三分天下 |
| [pfs/](infra/observability/pfs/) | **Performance Schema（多篇，故独立子目录）**：[`pfs.md`](infra/observability/pfs/pfs.md) 框架（PSI 接口、零开销探针、设计模式）· [`pfs_statement.md`](infra/observability/pfs/pfs_statement.md) 语句统计 · [`pfs_events.md`](infra/observability/pfs/pfs_events.md) 事件表族（waits/stages/transactions/**errors** + 两级开关）· [`thread_pfs.md`](infra/observability/pfs/thread_pfs.md) 线程插桩 |
| [slow_log.md](infra/observability/logging/slow_log.md) | **慢查询日志**：判定链 + ★ `log_slow_extra` **20 字段全集** + `log_slow_admin_statements` 真实机制 + 与 digest 对比 |
| [error_log.md](infra/observability/logging/error_log.md) | **错误日志**：组件架构（filter/sink）+ ★ `log_sink_json` 字段全集 + `ib::` 宏族 + ★ `innodb_print_ddl_logs` 需配 `verbosity=3` 的陷阱 + 暴涨根因 |
| [innodb_monitor.md](infra/observability/innodb_monitor.md) | InnoDB 可观测基础设施：Monitor 体系、`SHOW ENGINE INNODB STATUS` |
| [explain_trace.md](infra/observability/explain_trace.md) | 调试与可观测性：EXPLAIN（TRADITIONAL/TREE/JSON/ANALYZE）、optimizer trace |
| [sys_schema.md](infra/observability/sys_schema.md) | **sys schema**：102 视图 / 22 函数 / 26 过程；★ `x$` 数值视图族与"格式化不可排序"的取舍；安装与权限模型 |
| [processlist.md](infra/observability/processlist.md) | **会话观测四种入口**：`SHOW PROCESSLIST` / `I_S.PROCESSLIST`(已废弃) / `PFS.threads`(推荐) / `sys.session`；★ 锁代价、`THREAD_OS_ID` 联动 OS 层、State 体系 |
| [show_commands.md](infra/observability/show_commands.md) | **SHOW 命令族**：★ 多数改写成 SELECT；`SHOW ENGINE INNODB STATUS` 全段判读 + **`LATEST DETECTED DEADLOCK` 逐字段** + 受害者选择算法 |
| [status_variables.md](infra/observability/status_variables.md) | **状态 / 系统变量**：三作用域（默认 SESSION）、PFS 六表、代价；★ **counter vs gauge 与采样套路**；`SET PERSIST` / `variables_info` |
| [system_tools.md](infra/observability/system_tools.md) | **系统层 / 外部工具**：★ `THREAD_OS_ID` 桥梁；`pstack`/`pt-pmp` **会暂停进程** vs `perf`/eBPF 不暂停；代价对照与决策树 |

### [index/](index/) —— 索引（横切主题）

> 以"索引"为第一主语的篇进本目录；索引只是一部分的特性篇留在原处（全文检索 feat/fts、空间 R-tree server/datatype/gis、生成列 feat/generated_columns）。**地图与阅读路径见 [`index/README.md`](index/README.md)**。

| 文件 | 内容 |
|------|------|
| [btr.md](index/btr.md) | B-tree 结构与操作（btr 模块全套，含压缩页 page_zip 双份结构专节） |
| [rtree.md](index/rtree.md) | R-tree 索引结构全套 |
| [inverted.md](index/inverted.md) | 倒排索引存储与编码 |
| [metadata.md](index/metadata.md) | 索引元数据对象模型（dd::Index ↔ dict_index_t ↔ SDI） |
| [physical_storage.md](index/physical_storage.md) | 索引物理存储与持久化 |
| [transaction.md](index/transaction.md) | 索引 × MVCC/purge/唯一性 |
| [construction.md](index/construction.md) | KEY 类与索引构造 |
| [record_format.md](index/record_format.md) | 索引记录存储格式 |
| [partitioning.md](index/partitioning.md) | 索引 × 分区 |
| [secondary.md](index/secondary.md) | 二级索引整合专篇 |
| [types.md](index/types.md) | 索引类型总览（全谱系） |
| [access.md](index/access.md) | 回表与三类访问优化（覆盖索引/ICP/MRR） |
| [constraint.md](index/constraint.md) | 索引承载的约束（唯一性/外键） |
| [stats.md](index/stats.md) | 索引统计 |
| [operations.md](index/operations.md) | 索引的生老病死与观测 |
| [ahi.md](index/ahi.md) | 自适应哈希索引 |
| [ibuf.md](index/ibuf.md) | change buffer（insert buffer） |

### [cloud/](cloud/) —— 云环境（外部知识）

| 文件 | 内容 |
|---|---|
| [cloud_storage.md](cloud/cloud_storage.md) | 云存储（EBS/云盘）：设计权衡、attach/detach ≠ mount/umount、快照语义 |
| [cloud_db.md](cloud/cloud_db.md) | 云数据库架构：数据面/支撑环境分层、存算分离、HA 切换 |
| [cloud_networking.md](cloud/cloud_networking.md) | 云上网络：VPC、L3-L4-L7、网关体系、安全组、PrivateLink、K8s 网络 |

### [papers/](papers/) —— 外部论文剖析

> 源码文档以**本仓库源码**为事实来源；论文剖析以**论文**为事实来源，并在「对 MySQL 与云数据库的启示」一节与源码文档互指。

| 文件 | 内容 |
|---|---|
| [btrlog.md](papers/btrlog.md) | BtrLog（VLDB 2026）云上 WAL 日志服务：单写者假设、SSD 日志节点 Quorum、微秒级工程 |

---

## 如何新增文档

> 单篇写法（章节骨架、必填项）见 [`_template.md`](_template.md)。本节讲**文档之间**的组织规则。

### 第一步：决定放哪

> **前置（必做）**：先扫本文「目录索引」+ 读相关篇文首「边界」行，确认没被覆盖再动手——最常见的返工是新篇讲的内容已有宿主篇讲透。

```
新主题
  │
  ├─ ★ 是"跨层端到端特性"吗？（同时改 SQL 层与引擎层：分区、全文索引、GIS…）
  │     是 → feat/，一篇讲完整条链路；引擎差异写进篇内小节，不为每个引擎单独开篇
  │
  ├─ 是"一条 SQL 必经的一步"吗？
  │     是 → server/query/ 下加数字编号（如 09_dml.md）
  │
  ├─ 是"主链某一步的横向展开"吗？（该步内部还有大量独立机制）
  │     是 → 进该步的子目录（07_optimize/ 或 runtime/）
  │
  ├─ 是"观察/调试主链的工具"吗？（EXPLAIN、trace）
  │     是 → server/query/ 下，但在 README 归入「横切内容」，不占主链步骤
  │
  ├─ 是 server 与存储引擎的接口本身吗？
  │     是 → server/handler.md（唯一的边界文档，留在 server/ 根）
  │
  ├─ 是存储引擎内部实现吗？
  │     是 → innodb/（但"索引"是第一主语的存储结构——btr/AHI/change buffer——进 index/）
  │
  ├─ 是锁/同步机制吗？（横跨 include/mysys/sql/innodb 的主题）
  │     是 → infra/lock/（同步原语进 primitives/，事务锁进 transactional/）
  │
  ├─ 是"索引"主题吗？（横跨 SQL 语法/优化器/引擎存储，以索引为第一主语）
  │     是 → index/（索引只是一部分的特性篇留在 feat|datatype，由 index/README 导航）
  │
  ├─ 是外部论文的深度剖析吗？（事实来源是论文，不是本仓库源码）
  │     是 → papers/，一篇论文一个文件；文首「边界」行与相关源码文档互指
  │
  └─ 其他 server 层主题
        ├─ 数据类型（JSON/GIS/时间类型…）  → server/datatype/
        ├─ binlog 与复制                    → server/replication/
        ├─ 被全局复用的通用机制             → server/infra/
        ├─ 插件/组件/服务（可扩展框架）     → server/plugin/
        └─ 服务器日志（error/general/slow） → infra/observability/logging/
```

> ★ 主题的天然归属优先于"它是插件/组件"这个事实：`plugin/` 收录具体插件是**兜底**；功能主体天然属于别的目录就留在那详写，`plugin/` 只写框架视角并互指（例：认证算法在 [`server/auth/authentication.md`](server/auth/authentication.md)，不在 `plugin/`）。

### 第二步：判断该独立成篇，还是并入现有篇

**该独立成篇**（满足任一）：有自己的一整套数据结构（如 range 优化的 `SEL_TREE`/`SEL_ARG`）；是另一条并行的主链（如 DML）；是通用子系统被多个上层复用（如 LOB）。

**该并入现有篇**（满足任一）：拆开会让主链断头；内容是同一件事的两半；篇幅撑不起一篇且无独立数据结构。

**★ 已决定独立成篇后，原篇内容是"迁走"还是"两处都留"？** 按新篇与宿主篇的关系定：

| 关系 | 判据 | 处理 | 实例 |
|---|---|---|---|
| **另一个子系统** | 第一主语不同（独立的数据结构与生命周期） | **归位**：完整实现迁到新篇，宿主篇只留边界说明 + 归位表 + 本篇视角的结论 | `dblwr`/`刷脏`/`读路径`/`AHI`/`change buffer` 从 `io.md` 拆出 |
| **另一个视角** | 同一机制换个角度讲 | **并存**：宿主篇保留完整版，新篇写自己的视角并互指 | `recovery.md` 里的 redo（`redo_log.md` 完整版不动） |

✗ 两种错法：把"另一个视角"当"另一个子系统"搬空宿主篇；两处都留完整副本（必然漂移）。

### 第三步：够不够建子目录

- 同一阶段 ≥ 3~4 篇 → 建子目录（如逻辑优化 4 篇 → `logical/`）
- 只有 1 篇 → 留上级目录，不建单文件目录
- 建了子目录后，上级 README 必须按阶段分组展示

### 边界归属：第一主语原则

跨层主题按**"第一主语是谁"**归位，而不是按"涉及谁"；一份内容只在一处详写，其他篇交叉引用：

| 内容 | 第一主语 | 归属 |
|---|---|---|
| 迭代器怎么调 handler 取行 | SQL 层执行器 | `query/09_executor_iterator.md` |
| `position`/`ref`/`rnd_pos` 的接口语义 | 存储引擎接口 | `server/handler.md` |
| `row_search_mvcc`、游标推进 | InnoDB | `innodb/row_search.md` |
| DML 的 buffer row id 两阶段读 | SQL 层 DML | `query/09_dml.md` |

**例外：跨层端到端特性进 `feat/`**，判据（满足任一）：① 同时改动多个层 ② 可能被多个引擎实现（一篇 + 引擎差异小节，避免 N 个引擎 N 篇）③ 读者需要一次读完闭环。不属于此类的按层归位；**锁与同步**横跨四层，按主题进 `lock/`。

> `feat/` 篇的骨架另有要求（两种形态），见 [`feat/README.md`](feat/README.md)「篇内骨架」——这是全库唯一的目录级骨架规范，其余一律以 [`_template.md`](_template.md) 为准。

### 编号规则

- 主链用数字前缀表示**时间顺序**（`01_` → `09_`）
- 子目录内**编号全局连续、不重置**（`07_optimize/logical/02_` 一直排到 `physical/09_`）——跨篇引用"见 03 篇"永远有效；用**路径**区分命名空间，不靠移出去平级避免冲突
- 横切内容（`runtime/`、EXPLAIN）不参与主链编号语义，README 里单独分组

### 命名

全库英文文件名、小写、下划线；名字要能看出**内容归属**（如 `04_logical_join.md` / `06_join_order.md` 一眼分清逻辑与物理优化）。

### 收尾检查清单

1. **改上级 README 的文件索引**。条目风格：**文件名 + 一句话定位**——机制细节（函数名/★ 堆砌/算法清单）进索引就是抄一遍且必然漂移
2. **改被影响篇的交叉引用**（拆分/合并时原篇指向会失效）。★ **链接写法约定**：**同目录互指只写文件名**（链接目标形如 `(ddl.md)`，不加 `./`、不写目录路径）——这样目录整体搬迁时同目录内的链接零改动；跨目录引用必须写相对路径（Markdown 限制，无解），因此**跨目录引用尽量收敛到各 README 的导航**，压掉目录变动时的改动面
3. **跑链接校验**——相对路径层级最易错：
   ```bash
   cd .codebuddy/knowledge_for_mysql
   for f in $(find . -name "*.md"); do d=$(dirname "$f"); \
     for l in $(grep -o "](\([^)#]*\.md\)[^)]*)" "$f" | sed 's/](//;s/)$//;s/#.*//'); do \
       [ -e "$d/$l" ] || echo "BROKEN: $f -> $l"; done; done
   ```
4. **写好本篇「参考」章节**（四类资料源都要查，见 [`_template.md`](_template.md) 自检 ⑰ 与参考章说明）

---

## 准入标准

> 上一章讲**放哪**，本章讲**够不够格进**。完整规则在 [`_template.md`](_template.md) 自检 ⑤~⑰，此处只留骨架。

**内核月报式深度剖析**，一篇合格文档回答三问：

| # | 问题 | 落在哪一章 |
|---|---|---|
| 1 | **怎么设计的？为什么？**（权衡、被否决方案、论文溯源、他库对比） | 「理论基础」——价值核心 |
| 2 | **重点难点是什么？**（反直觉处、隐含假设、复杂状态机） | 「核心实现」——**边贴代码边剖析**（写法见下方专节） |
| 3 | **怎么演进到今天的？** | 「版本演进」记变化 + **能查到的标 `WL#xxxx`**（见下方专节），「理论基础」记为什么 |

三条硬门槛：有理论基础 / 讲透实现 / 模块闭环。篇幅与可测指标基线见 `_template.md` ⑥⑮。基线版本 8.0.39，之后版本（8.4 / 9.x）也要跟踪。

★ **不写「勘误 / 纠错」章**——写错就地改对、不留痕迹；"某符号不存在"直述事实，不立纠错表（见 `_template.md` ⑫；存量不回头清理）。

### ★ 源码剖析的统一写法：在原代码里就近注释

**贴完整原代码，在关键行附近直接用注释剖析**。**禁止**"整段贴完再在末尾集中写『**逐段解释**：①…②…』"。

**为什么**：解释要**紧贴对应代码行**。末尾集中解释会让读者看代码时还没拿到解释、要来回翻，且解释脱离了对应行的上下文，退化成"对一段话的复述"而不是"对这段代码的说明"。

**四条要点**

1. **注释紧贴对应行**——行上方或行尾，让读者看代码时注释就在旁边
2. **只注释核心与难点**——**不必逐行注释**；把"为什么必须这样"讲清楚即可
3. **注释讲"为什么"**——做了什么 + 为什么这么写 / 边界怎么处理；**不复述**代码字面意思
4. **★ 保持代码完整**——**不要**为了插解释把函数切碎成多个代码块（打断阅读流、也不便复制）。⚠️ "完整"指**函数不分家**；无关行仍按既有规范**骨架化**（砍变量声明 / Doxygen 注释 / 无关分支，用 `...` 压缩），二者不冲突；代码占比基线仍按 `_template.md` ⑮ 执行
5. **★ 贴代码之前，先用一句话交代"这个函数整体在做什么"**——说清它的输入 / 输出 / 在调用链里的位置，再贴代码。⇒ 读者先有整体框架，看代码时才知道每行服务于什么；**直接从函数签名进代码，容易"每行都看得懂、但不知道整体在干什么"**。这也是自查"有没有停在包装函数上"的前提——先把被调者的职责讲清，才判断得出要不要继续往下挖
5. **★★ 不要停在包装函数**——若一个函数的主体是"准备参数 + 转调另一个函数"（通常只有十几到几十行），**它不是剖析对象**，必须**顺着调用链往下挖到真正有逻辑的那一层**。⚠️ 判据：写完后通篇都是"它调用了 X"、却没有"X 内部怎么做" ⇒ 就是停在包装层了。实例：`row_sel_build_committed_vers_for_mysql` 只有 26 行且主体是转调，真正逻辑在 `row_vers_build_for_semi_consistent_read`（109 行）——只剖前者等于没剖
6. **★ 小标题不要用纯函数名**——标题写**结论 / 语义**（读者扫目录就知道这节讲透了什么），函数名在正文首句交代即可；**同一件事的多个函数（入口 + 实现）放在同一标题下**，不要每个函数一节。⚠️ 反例：`#### row_vers_build_for_semi_consistent_read`（只有函数名，看不出讲什么）；正例：`#### 半一致读要的是「最新已提交版本」，不是「视图可见版本」`

**正反对照**

❌ 整段贴完 30 行，再在末尾写：

> **逐段解释**：- 短路在最前面… - `LOG_BUFF_MAX` 定长栈缓冲… - audit 的位置…

✅ 完整代码 + 关键行就近注释：

```cpp
  // ★ 短路在最前面：这类命令要不要记 / 总开关 / handler 链是否非空，三者缺一即走这里
  //   ⇒ general_log=OFF 时开销只是一次布尔判断——零开销探针思路
  if (!log_command(thd, command) || !opt_general_log ||
      !(*general_log_handler_list))
    return false;

  char message_buff[LOG_BUFF_MAX];   // 定长栈缓冲 ⇒ 消息超长会被截断
```

完整范例见 [`infra/observability/logging/general_log.md`](infra/observability/logging/general_log.md)「核心实现」。

> ⚠️ **存量说明**：既有篇目里仍有"整段代码 + 末尾『逐段解释』"的旧写法（2026-10-02 盘点约 27 篇）。**存量不批量整改**——本规范只对**新增 / 重写的篇目**生效；看到旧写法**不需要**回头改写（改了会造成同一篇内新旧混用）。

### ★ 版本演进要标 WorkLog 编号

「版本演进」里**能对应 MySQL Worklog 的变化，要标 `[WL#xxxx](https://dev.mysql.com/worklog/task/?id=xxxx)`**——这是演进最权威的一手设计依据，比 Release Notes 更深一层（含 Scope / HLS / 兼容性影响）。

**形式二选一**：写进"变化"列（多数篇这样做，如 `引入 MDL（设计见 WL#3726 / WL#4284）…`）；或独立成 `WL` 列（演进多且重要时更清晰）。

**查不到编号的如实注明**（如"早于 MySQL Worklog 体系，无编号"），**不要留空**——留空会让人以为漏了。

### 提交前自检（内容九条；组织四条见上一章）

1. **「理论基础」写了吗**——设计思想、权衡、理论溯源、演进动机
2. **关键代码贴了吗**——只给结论等于没写透
3. **模块闭环吗**——上下游、边界都交代了吗
4. **难点讲透了吗**——能解释"为什么不得不这样"
5. **跑过可测指标吗**——代码占比、符号覆盖 grep、链接校验；被问"X 剖析了吗"先实测再答
6. **归属对吗**——专职机制归模块文档，全景篇只留「清单 + 通用机制」
7. **★ 是「边贴边剖析」吗**——在**完整原代码的关键行附近就近注释**；**不是**整段贴完末尾集中「逐段解释」；核心与难点注释清楚即可，**不必逐行**（见上方专节）
8. **★★ 挖到实质了吗 + 标题写对了吗**——① 有没有停在包装函数上？逐个检查本篇被剖的函数：主体是不是"准备参数 + 转调"？若是**继续往下挖被调者**，直到那一层有真正的逻辑；② 小标题是不是纯函数名？若是改成**结论 / 语义**，且同一件事的多个函数（入口 + 实现）要归到**同一标题下**（判据见要点 5、6）
9. **★ 版本演进标 `WL#` 了吗**——能对应 MySQL Worklog 的标 `[WL#xxxx](...)`（见上方专节）；**查不到的注明原因**，不留空

> 单篇模板见 [`_template.md`](_template.md)；参考资料在**每篇末尾的「参考」章节**，不全局维护索引。
