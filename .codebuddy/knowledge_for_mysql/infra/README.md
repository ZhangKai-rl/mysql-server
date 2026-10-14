# infra：跨层基础设施（数据结构 + 锁）

> 本目录收纳 MySQL 的**跨层基础设施**——server 与 InnoDB 两层共有/共用、且不属于任何单一功能模块的"基座"内容。当前含五大子域：
>
> - `structure/` —— **数据结构**：按**类型**一篇一主题，**不分层**。跨层的同主题实现（如两个无锁哈希、两个 MPMC 队列）在**一篇文档里写清对比**（本目录的组织原则）
> - `lock/` —— **锁与同步**：锁横跨 include/mysys/sql/innodb，是最重要的横切主题之一，整体从顶层迁入本目录（2026-09-20）
> - `threading/` —— **线程模型**（2026-09-28 建目录）：线程与锁**同构**——同样横跨 server 与 InnoDB，同样与并发控制、PFS 三向耦合，故同样按**主题**归位而非按层拆开（顶层 README「边界归属」把锁列为按层归位的例外，线程是同理的第二个）。含 THD 对象、全栈线程全景、连接线程模型、线程池（MySQL 26.7 / MariaDB / Percona 三家）。其中 **[threading_architecture.md](threading/threading_architecture.md)** ★ 要点：新增 **1.3 server 层后台线程线程视角深剖**（`compress_gtid_table` 纯事件驱动**无超时兜底**且带 THD 出生、`signal_hand` 是全栈**唯一 sigwait 线程**且 **JOINABLE** 与连接线程相反、事件调度器两级结构与 page cleaner 同构但 worker 是**临时**的）+ **1.4 ★ 等待模型横向对照表**（15 类线程 × 等待对象/定时/自旋/唤醒源；★ 三种等待原语 `os_event` / `mysql_cond_t` / `sigwait` 的分层）+ **1.5 ★ 按需/临时线程深剖**（启动期 fork-join `parallel_rseg_init`、查询期 fork-join `parallel_read`（线程数=0 时调用线程自己干不起线程）、`ts_alter_encrypt`、恢复期 `recv_writer`/`trx_recovery_rollback`）+ **1.6 ★ server 层其余常驻线程**（★ manager 线程**默认不启动**只有 `flush_time!=0` 才建、`Source_IO_monitor`、parser_service、Windows 专属 shutdown_restart 与 con_named_pipes/con_shared_mem）。★ 线程清单已从 PSI 注册表 + `os_thread_create` 全部调用点**源码级核对**，InnoDB 约 26 类 + server 约 14 类
> - `memory/` —— **内存**（2026-09-29 建目录）：内存分配器矩阵本身**跨层**（mem_heap/buf_buddy 属 InnoDB、MEM_ROOT 属 server），且全局内存架构（BP 页面、redo buffer、各 cache、PFS 观测）必须横向总览，故内存与锁/线程同构，按**主题**归位。含分配器矩阵与全局内存架构总览两篇。`server/infra/memory.md` 已迁入 `allocator.md`
>
> - `observability/` —— **观测体系**（2026-10-02 建目录）：PFS 框架在 server（`storage/perfschema/` 152 个 .cc / 4 MB）而**埋点有 132 个文件在 InnoDB**、`INNODB_*` 表由 `handler/i_s.cc` 实现 ⇒ 典型**跨层框架**，与 `i_s.md` 同构（server 给框架、引擎填内容），故归本目录而非 `feat/`（后者放 8.0 引入的具体特性）。★ 组织维度＝**按「观测手段」**（表 / 日志 / 命令 / 分析 / 系统层），**不是**按"XX 怎么观测"——复制/DDL/锁的观测留在各自主题篇。详见 [observability/README.md](observability/README.md)
>
> **与 `server/infra/` 的区别**：本目录是**跨层**基础设施之家；`server/infra/` 是 **server 层基础设施专题**（PFS/内存/编码/IO_CACHE 等，**暂不归入**——它是 server 层专属设施，与跨层定位不同）。**线程三篇已按主题从 `server/infra/` 迁入 `threading/`**（2026-09-28）：`connection_threading.md`、`thread_pool.md`、`thread_pool_mariadb_percona.md`。⚠️ **待办**：`server/infra/pfs.md`(1117) / `pfs_statement.md`(1077) 实为**跨层**内容（引擎侧 132 个文件埋点），按本判据应迁入 `observability/`（记入待办，未执行）。

## 目录树

```
infra/
├── README.md                  ← 本篇：定位 + 归属判据 + 全量清单
├── threading/                 ← 线程模型：THD 对象 + 全栈线程 + 连接线程 + 线程池
│   ├── thd.md                 THD 对象深度剖析
│   ├── internal_thd.md        内部 THD（后台线程持有）
│   ├── stack_depth.md         栈深度治理（thread_stack / 检查点地图 / JSON 容量）
│   ├── thread_lifecycle.md    线程生命周期
│   ├── threading_architecture.md  全栈线程模型
│   ├── connection_threading.md    连接线程模型（Per_thread）
│   ├── thread_pool.md         线程池（MySQL 26.7）
│   ├── thread_wait.md         线程等待登记与唤醒协议（enter_cond / KILL 广播）
│   └── thread_pool_mariadb_percona.md  MariaDB/Percona 线程池对比
├── lock/                      ← 锁与同步（详见 lock/README.md）
│   ├── primitives/            ← 同步原语
│   └── transactional/         ← 事务锁
├── memory/                    ← 内存：分配器矩阵 + 全局内存架构 + jemalloc
│   ├── allocator.md           分层内存分配器矩阵（mem_heap / MEM_ROOT / buf_buddy / jemalloc）
│   ├── memory_architecture.md  全局内存架构总览（各内存区、预算、OOM 排查）
│   └── jemalloc.md            ★生产重点：接入、RSS 四层次账本、mallctl 观测/OOM 诊断、调优、Profiling、THP
├── structure/                 ← 数据结构：按类型一篇一主题
│   ├── list.md                链表
│   ├── hash.md                哈希（三实现全盘点）
│   ├── queue.md               有界 MPMC 队列
│   ├── counter.md             分片计数器
│   ├── link_buf.md            Link_buf 无锁链缓冲
│   └── data_repr.md           行数据表示（跨层）
└── observability/             ← 观测体系：按「手段」组织（详见 observability/README.md）
    ├── i_s.md                     INFORMATION_SCHEMA（server 插件框架 + InnoDB 26 张表）
    ├── pfs/                       ← PFS 多篇，故独立子目录
    │   ├── pfs.md                 框架：instrument / consumer / PSI 接口 / 设计模式
    │   ├── pfs_statement.md       statement 事件表族、直方图、采样
    │   ├── thread_pfs.md          PSI 线程插桩
    │   └── pfs_events.md          事件表族：waits / stages / transactions / errors + 两级开关
    ├── logging/                    ← 日志类（多篇 ⇒ 子目录）：error_log / general_log / slow_log
    │   ├── error_log.md            错误日志
    │   ├── general_log.md          通用查询日志（记录内容、输出方式、性能开销）
    │   └── slow_log.md             慢查询日志
    ├── innodb_monitor.md          InnoDB monitor：`SHOW ENGINE INNODB STATUS`（已迁入）
    ├── explain_trace.md           EXPLAIN / EXPLAIN ANALYZE / optimizer trace（已迁入）
    ├── sys_schema.md              sys schema：102 视图 / 22 函数 / 26 过程 + x$ 视图族
    ├── processlist.md             会话观测四种入口：代价差异 + State 体系 + THREAD_OS_ID
    ├── show_commands.md          SHOW 族：改写机制 + INNODB STATUS 全段判读 + 死锁段逐字段
    └── status_variables.md        状态/系统变量：作用域 + counter vs gauge + SET PERSIST
    ├── processlist.md              待建：会话观测
    ├── show_commands.md / status_variables.md   待建
    └── system_tools.md             系统层工具：pstack/pt-pmp(会暂停) vs perf/eBPF(不暂停) + 代价对照
```

## 归属判据

- **数据结构本身**（链表、哈希、队列、计数器、链缓冲等"结构"）→ `structure/`，**按类型一篇一主题**：同一主题跨层实现合一篇对比写清（如 `hash.md` 含三个哈希、`queue.md` 含两个队列），单层独有也单篇（如 `link_buf.md`）。**哈希/队列/计数器这类"主题"要写全量盘点**，不是只写某个实现
- **用无锁结构实现的机制**（第一主语是功能）→ 各功能模块目录：`Bgc_ticket_manager`（binlog 组提交 ticket 机制）→ `server/replication/binlog.md`；redo 对 `Link_buf` 的使用 → `innodb/redo_log.md`（容器本身的剖析在 structure/）
- **锁原语/同步原语**（mutex / rwlock / cond / event / latch / RCU 锁模式 / seqlock）→ `lock/primitives/`
- **事务锁**（MDL / 行锁 / 表锁 / AUTOINC / 全局锁）→ `lock/transactional/`
- **线程模型**（THD 对象 / 全栈线程全景 / 连接线程 / 线程池）→ `threading/`：线程与锁同构地横跨 server 与 InnoDB，按**主题**归位（2026-09-28 建目录，三篇从 `server/infra/` 迁入）
- **内存**（分配器矩阵 / 全局内存架构）→ `memory/`：分配器与内存架构横跨 server 与 InnoDB，按**主题**归位（2026-09-29 建目录，分配器矩阵从 `server/infra/` 迁入）
- **"第一主语不是线程"的线程内容**（`replica.md` 的复制线程、`binlog.md` 的 dump 线程、FTS 后台线程、PFS 里的线程统计）→ 留在各功能模块目录，`threading/` 只做交叉链接，不复制内容

## 全量数据结构清单（与 `lock/README.md` 盘点同步）

| 结构 | 层 | 类型 | 用户/用途 | 文档 |
|---|---|---|---|---|
| 链表（`ut_list_base` / `SQL_I_List` / `base_list`+`List` / `I_List` / C `LIST`） | 跨层 | 五实现：侵入式双向偏移量 / 侵入式单向二级指针 / 非侵入式哨兵 / 侵入式双向自解链 / C 双向 | trx_sys / buf_pool / lock / item 链 / 执行计划链等 | ✅ [list.md](structure/list.md) |
| `LF_HASH` | server | 无锁哈希（split-ordered list + hazard pointer，**有论文原型**） | MDL_map / Acl_cache / PFS 对象表 | ✅ [hash.md](structure/hash.md) |
| `ut_lock_free_hash_t`（+ `ut_lock_free_cnt_t`） | innodb | 无锁哈希（开放寻址 + 链表数组，**无论文原型**） | `buf_stat_per_index`（bp 每索引页数） | ✅（同上 hash.md） |
| `hash_table_t` | innodb | **有锁**经典哈希（链地址法 + 分片 rw_lock） | lock_sys `rec_hash`/`prdt_hash`、bp `page_hash`、dict、AHI | ✅（同上 hash.md，三实现糅合一篇） |
| `Integrals_lockfree_queue` | server | 无锁队列（仅整型，虚拟索引 + MSB 占用位） | `Bgc_ticket_manager`（binlog 组提交 ticket） | ✅ [queue.md](structure/queue.md) |
| `mpmc_bq` | innodb | 无锁有界 MPMC（Vyukov 算法，引 1024cores.net） | dblwr `Segments` / FTS `Docq` | ✅（同上 queue.md，同族异路合一篇） |
| `ib_counter_t` | innodb | 分片计数器（RDTSC 散槽，读 fuzzy） | `srv0srv.h` 5 个 typedef + 全局统计 | ✅ [counter.md](structure/counter.md) |
| `Counter::Shards` | innodb | 分片计数器第二代（真原子，读精确） | bp `m_n_page_gets`、no-logging mtr、Parallel_reader | ✅（同上 counter.md） |
| `Link_buf<Position>` | innodb | 无锁环形"链缓冲"（乱序报告 + tail 沿链推进） | redo `recent_written` / `recent_closed` | ✅ [link_buf.md](structure/link_buf.md) |
| `Seq_lock<data_t>` | innodb | 序号锁（seqlock，回调式，引 HPL-2012-68） | `mt_fast_modulo_t`（hash 表快速取模） | 属锁原语，✅ [lock/primitives/seq_lock.md](lock/primitives/seq_lock.md) |
| `MyRcuLock<T>`（RCU） | server | RCU 锁模式 | `ssl_acceptor_context_data` | 属锁原语，✅ [lock/primitives/rcu.md](lock/primitives/rcu.md) |
| `dtype_t` / `dfield_t` / `dtuple_t`（+ `big_rec_t`/`upd_t`/`multi_value_data`/`row_ext_t`） | 跨层 | **行数据表示**（类型位域打包 / 字段 / 元组），对偶于 `rec_t` 字节流 | server↔InnoDB 插入检索全链路 | ✅ [data_repr.md](structure/data_repr.md) |
| `Field`（68 子类）/ `TABLE::record[2]` / `String` / `Copy_field` | server | server 行表示（类型视图 + 字节缓冲） | 执行器、handler 接口、字符集转换 | ✅（同上 data_repr.md，与引擎侧合一篇对比） |

## 两个无锁 hash 的对比入口

server 侧的 `LF_HASH` 与 innodb 侧的 `ut_lock_free_hash_t` 是**两条完全不同的路线**（split-ordered list + hazard pointer vs 开放寻址 + 链表数组 + 引用计数；有论文原型 vs 无论文原型）。三实现对比、论文不对称的诚实说明与"通用性越强、理论原型越重"的结论见 [hash.md](structure/hash.md)「总览」与「选择指南」。
