# MySQL 8.0.39 源码知识库

> 基于 MySQL 8.0.39 源码的深度解析文档。重点讲设计思想、理论溯源与核心实现；结论均经源码核实。

## 从哪开始

**想理解一条 SQL 怎么执行** → [`server/query/README.md`](server/query/README.md)（主链全景 + 6 层中间表示图）

**想查某个具体机制** → 用下面的目录索引

**★ 要新增或修改一篇文档** → **动笔前必读**（它们是本库的准入规范，不读会返工）：
① 本文[「如何新增文档」](#如何新增文档)——放哪、拆不拆、归位 vs 并存、编号、收尾清单
② 本文[「准入标准」](#准入标准)——三条硬门槛、难点怎么讲、六类事实错误、不该进什么
③ [`_template.md`](_template.md)——单篇骨架 + 13 条动笔自检

---

## 一条 SQL 的完整旅程（跨目录）

```
客户端
  │
  ▼
┌──────────────────────────────────────────────────┐
│ server/query/    协议 → 解析 → 优化 → 迭代器执行  │  ← 主链，35 篇
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

### server/ —— Server 层

**SQL 处理主链**：见 **[query/README.md](server/query/README.md)**（35 篇 = 主链 12 篇 + [`07_optimize/`](server/query/07_optimize/) 优化器 15 篇 + [`runtime/`](server/query/runtime/) 执行期专题 8 篇；主链：协议分发 → 解析 → contextualize → prepare → 优化 → AccessPath → 迭代器执行 → DML）

| 文件 / 目录 | 内容 |
|---|---|
| [table.md](server/table.md) | **表**：四层表示（DD → `TABLE_SHARE` → `TABLE` → `Table_ref`）、三个缓存分工（TDC / `table_open_cache` 16 分片 / InnoDB dict）、表的八种通用操作（open / lock / CREATE / ALTER / RENAME / TRUNCATE / DROP / FLUSH）与各自入口函数、分区表 SHARE↔`dict_table_t` 一对多 |
| [handler.md](server/handler.md) | server ↔ 存储引擎分界面：handler/handlerton 分层、prebuilt、行定位 |
| [mdl.md](server/mdl.md) | MDL 元数据锁：表结构保护、双兼容性矩阵排队语义、wait-for graph 死锁检测 |
| [auth/](server/auth/) | 认证与授权：security_context.md（认证上下文）、mfa.md（多因素认证）、definer.md（definer 与 SQL SECURITY） |
| [replication/](server/replication/) | 复制：binlog.md（格式/组提交/2PC）、gtid.md、replication.md（主从）、prpl.md（并行复制） |
| [datatype/](server/datatype/) | 数据类型：json.md（JSON 二进制/部分更新/索引）、gis.md（空间/R-tree） |
| [dd/](server/dd/) | **数据字典**（两套并存）：dd.md（8.0 权威 DD：DD 表体系、`dd::Table` 对象模型、DD↔文本往返、DD cache、SDI、`innodb_dynamic_metadata`）、innodb_dict.md（引擎侧 `dict_sys`：DICT_HDR 页、`SYS_*` 内部表、`dict_table_t`/`dict_index_t`、dict cache、从 DD 加载） |
| [infra/](server/infra/) | 通用机制：list.md（侵入式链表）、dbug.md（DBUG 框架）、pfs.md（PFS 可观测性）、memory.md（内存分配器矩阵）、vio.md（VIO 通信抽象）、variables.md（变量体系）、**io_cache.md（`IO_CACHE` 与 server 层 I/O 继承体系：数据结构双缓冲指针、函数指针"穷人版多态"、延迟建文件、`READ_NET` 把网络伪装成文件、透明加解密、`Basic_ostream`→`Truncatable_ostream`→`IO_CACHE_ostream`/`Binlog_encryption_ostream` 装饰者链与 `Binlog_ofile` 门面）**、**reading_guide.md（★ 源码阅读知识地图：七个知识域、C 预处理器与 C++ 惯用法清单、设计模式与 MySQL 源码完整对照表、分阶段学习路线、"读不懂"的自诊断流程）** |
| [plugin/](server/plugin/) | 可扩展框架：plugin.md（插件体系：ABI/类型/加载安装/引用计数延迟收割）、component.md（组件基础设施：minimal chassis/dynamic loader/mysql.component/manifest/内置组件清单）、service.md（服务：C ABI 契约/registry/default/acquire_related/老版 plugin service）、abi.md（**横切专题**：C++ ABI 五个不稳定点/符号可见性/异常防线/旧结构迁移/.pp 指纹门禁）；具体插件分析（如 clone_plugin.md）后续进此目录 |

### [innodb/](innodb/) —— 存储引擎

> **物理结构**（表空间空间管理/页/行/LOB 四篇）集中在 [`innodb/physical/`](innodb/physical/) 子目录：`tablespace`（表空间/段/区/碎片度量）、`page_structure`（页 + 全页类型清单）、`record`（行）、`lob`（大对象）。

| 文件 | 内容 |
|------|------|
| [buffer_pool.md](innodb/buffer_pool.md) | **Buffer Pool**：slot-based 骨架与 chunk 布局、`buf_page_t`/`buf_block_t` 状态机与 `buf_page_in_file`、io_fix 声明式 latch 协议校验（`Stateful_latching_rules`）、**★ 完整读路径**（缺页读全流程 = 调用线程 `pread()` 非异步 AIO、并发同页只发一次 I/O 的 hash_lock 临界区、★ `buf_wait_for_read` **无 os_event**——"能拿到 S 锁"即唤醒、`buf_page_io_complete` 十步后处理、**os 层与 buf 层两层解压不重复**、读失败重试 100 次才 fatal）、**change buffer**（延迟写换随机读，**云盘收益更大**、"写完立刻读"该关）、AHI、线性/随机预读与四种预读入口、LRU 中点替换（old/new + `old_blocks_time` 时间窗抗扫描污染）、**★ 完整脏页刷盘**：三路径汇聚 `buf_flush_write_block_low`、刷脏批次 single-flight（`buf_flush_do_batch` 返回 false = 没轮到我）、hazard pointer 扫描与 LRU 的 `mutex_enter_nowait`、`buf_flush_page` 锁契约（返回值决定 mutex 归属）、跳过不漏刷的三层保证、邻接刷盘 `buf_flush_try_neighbors`、全量刷脏 `buf_flush_sync_all_buf_pools`、用户线程单页刷为什么慢、AIO 槽位耗尽阻塞刷脏；自适应刷脏（LSN age 因子 sqrt 非线性）、压缩页三态、page cleaner coordinator/worker、Buffer Pool Watch 哨兵 |
| [dblwr.md](innodb/dblwr.md) | **doublewrite 完整实现**：★ 为什么必须有（torn page 与 **redo 的能力边界**）与何时能关（FusionIO 原子写互斥）、类结构（`Double_write`/`Segment`/`Batch_segment`/`Reduced_double_write`）、**文件布局（无文件头的扁平页数组、批量区 + 512 个 SYNC 槽位、★ 奇偶文件功能切分不是轮换）**、批量写完整流程（`enqueue`→`flush_to_disk`→`write_dblwr_pages`→`write_data_pages`→`write_complete`）、同步单页七步、`write_to_datafile` 的 IORequest 标志、**崩溃恢复时序与 torn page 三 case 判定**、★ 加密帧为什么单独存在（绕过 fil 层故须提前压缩+加密）、`O_DIRECT_NO_FSYNC` 下五处 fsync 的取舍、`force_flush` 四个调用点、参数（`innodb_doublewrite` 6 值、`batch_size` 是 no-op）、监控、源码 TODO |
| [ahi.md](innodb/ahi.md) | **自适应哈希索引（AHI）**：★ 是什么（B-tree 之上`rec_t*` 内存指针的**可丢弃**缓存）与三不保证（不检查页边界 / 可能碰撞 / **惰性修补**）；三层结构与 `btr_ahi_parts=8` 分片、`hash_table_t` 内部零锁；**★ 哈希键的前缀长度自适应算法**（用 low/up 的 match/bytes 反推最小区分长度）；查询路径 8 道门禁与 `btr_search_guess_on_hash` 逐步（nowait S 锁、先 fix 页后放 AHI 锁、三重校验）；维护路径（build/drop/insert/delete/move/惰性修补）与 **nowait 哲学**；★ **锁按 (space_id, index_id) 路由 ⇒ 同一索引所有页同一把 latch，单索引热点无解**；关闭 AHI 等 `ref_count` 归零 **600 秒后 crash**；命中率低时该关 |
| [ibuf.md](innodb/ibuf.md) | **change buffer（insert buffer）**：★ 为什么只缓存**非唯一二级索引**的 INSERT（唯一性检查与「页不在 BP 才缓存」**逻辑互斥**），而**唯一索引的 delete-mark / purge 反而可缓存**；物理布局（ibuf 树在 space 0，header page 3 / root page 4；bitmap 每 16384 页一张，4 bit/页 = FREE 2bit + BUFFERED + IBUF）；记录格式（counter 保证时序）；**插入的 14 条否决条件**、合并的 8 条触发条件、三级自我保护 contract（0/5/10）；★ redo 真相（ibuf 树走**普通 B-tree redo**，唯一的 `MLOG_IBUF_*` 是 BITMAP_INIT）；**merge 可能触发页分裂**、`AIO_mode::IBUF` 防死锁；**云盘上收益更大** |
| [mvcc.md](innodb/mvcc.md) | read view、可见性判断、半一致性读 |
| [row_search.md](innodb/row_search.md) | `row_search_mvcc`、游标推进（承接 handler.md） |
| [btr.md](innodb/btr.md) | **索引 B-tree 结构与操作（btr 模块全套）**：游标体系与 `btr_cur_search_to_nth_level` 搜索（latch coupling + 8.0 SMO 锁预测裁剪）、乐观/悲观插入与页分裂（顺序插入感知 / 根页抬高）、删除与合并（merge_threshold / lift 降高）、更新三条路径、持久游标恢复协议、自适应哈希索引 AHI（8.0.30 分片）、`Btree_load` 排序批量构建 |
| [tablespace.md](innodb/physical/tablespace.md) | **表空间物理结构与空间管理**：FSP header 112B / XDES 40B / inode 192B / fseg header 10B 逐字节布局、segment header → inode → fseg 解析链、`fseg_alloc_free_page_low` 七分支 / `fsp_alloc_free_page` / `fsp_free_page` / `fseg_mark_page_used` / lease 机制 / reserve factor、`FSP_FLAGS` 位域、系统表空间页布局、碎片度量（`DATA_FREE` / `FREE_EXTENTS`） |
| [page_structure.md](innodb/physical/page_structure.md) | **页物理结构与全页类型清单**：FIL header/page header、infimum/supremum、heap + 单向链表 + page directory 稀疏索引、`page_cur_search` 页内二分、30 种页类型清单、**12 类常见页的逐字节 layout**（INDEX / FSP_HDR / XDES / INODE / IBUF_BITMAP / COMPRESSED / RSEG_ARRAY / TRX_SYS / BLOB-LOB / UNDO / SDI / ENCRYPTED） |
| [record.md](innodb/physical/record.md) | **行记录格式与 offsets 数组**：record header 5/6 字节与 info bits、变长长度编码（1/2 字节 + 外置 0x40 位）、NULL 位图、`rec_get_offsets` 全链、offsets 前缀和 + 高 4 位标志协议、REDUNDANT 1/2 字节目录、instant V1/V2 行版本状态机、写方向 `rec_convert_dtuple_to_rec`、固定 offsets 缓存 |
| [lob.md](innodb/physical/lob.md) | 大对象存储（JSON 部分更新的物理基础） |
| [lock.md](innodb/lock.md) | 行锁、间隙锁、死锁检测 |
| [redo_log.md](innodb/redo_log.md) | **redo**：格式（文件→block→record 三层）、LSN/sn 序号、mtr 与 log mode、8.0 无锁化写路径、**★ redo 的 I/O 路径（五个专用后台线程 `log_writer`/`log_flusher`/`log_checkpointer`/两个 notifier/`log_files_governor`；`log_writer_write_buffer` 环形写与 512 对齐；`log_flush_low` 的 O_DSYNC 分支；redo 与数据文件的 I/O 方式差异表 —— redo 不走 AIO、不开 O_DIRECT、只有同步 `pwrite`+fsync；write-ahead 与 read-on-write；fsync vs fdatasync）**、落盘时机三档与组提交、后台线程暂停与恢复、checkpoint、文件管理与 resize |
| [recovery.md](innodb/recovery.md) | **崩溃恢复**：两阶段模型（redo 前滚 / undo 回滚）、扫描与解析状态机、`recv_sys_t` 恢复上下文、hash 聚合与按页应用、文件级 redo、clone/MEB 分支 |
| [undo_log.md](innodb/undo_log.md) | undo 表空间、purge |
| [query_graph.md](innodb/query_graph.md) | InnoDB 内部执行模型（fork/thr/node） |
| [ddl.md](innodb/ddl.md) | online DDL |
| [parallel_scan.md](innodb/parallel_scan.md) | InnoDB 内部并行扫描（`Parallel_reader`）：核心数据结构、**B+ 树按子树切分算法**、线程队列与同步、三个实际消费者、触发条件；SQL 层并行查询怎么实现（PolarDB 的物理计划 clone + Exchange 算子 + DOP）、**标准 MySQL 为什么不做**；补充「从 I/O 视角看 Parallel_reader」（函数清单、四个使用范围、★ 社区版无 SQL 层并行查询，`innodb_parallel_read_threads` 是 InnoDB 私有 sysvar 不是 optimizer hint） |
| [fil.md](innodb/fil.md) | **表空间与文件层（fil）**：三层结构与 `Fil_shard`×68 分片（`space_id%64` + undo 专用）、文件类型常量（`OS_DATA_FILE`/`OS_LOG_FILE`/`OS_DBLWR_FILE`…）、`fil_io`→`Fil_shard::do_io` 主链路（LBA 映射 / punch hole / 加密注入）、★ **AIO 模式三选一含"缺页读为何是同步 `pread`"的完整判定链**、并发控制 flag 体系（`n_pending_ios`/`n_pending_flushes`/`is_being_extended`/`stop_new_ops`）、`space_extend`（fallocate+写零，★ 记 redo 但故意不 `log_write_up_to`）、`space_flush` 的 **fsync 合并**与四计数器（`modification_counter`/`flush_counter`/`flush_size`） |
| [io.md](innodb/io.md) | **MySQL I/O 全景（从 SQL 到系统调用）**：七层分层图与 11 类 I/O 子系统总览；设计权衡（为什么数据库必须绕过 OS 自己管 I/O、redo 同步写 vs 数据页异步 AIO、doublewrite 为何存在）；核心实现 11 章——fil 表空间层（`Fil_shard` 68 分片 / `IORequest` 位标志 / `get_AIO_mode` 三模式）、`os_file` 与 O_DIRECT·O_SYNC·fsync（**EIO 即 `ib::fatal`**）、AIO 子系统（**同步/异步的分工**与全场景清单、libaio + Simulated + Windows 三套实现、**无 io_uring**、★ native AIO 启用前会探测 tmpdir，失败则静默退化）、**server 层 I/O 全景**（mysys 原语与 PFS 埋点层 / `IO_CACHE` 与 spill 机制 / **创建即 unlink 的临时文件** / binlog·relay log / slow·general·error log / **内部临时表三级降级与 filesort 临时文件** / 导入导出 / MyISAM·CSV·ARCHIVE，★ 全程无 O_DIRECT、无 libaio、目录项无 fsync）、I/O 线程模型、压缩加密与 punch hole（**不改变 I/O 大小**）、特殊场景（`FLUSH TABLES` ≠ 刷脏、DROP 同步 unlink）；★ 云盘上的 MySQL I/O 与五条隐含假设。★ **读路径/预读/change buffer 已归位 [`buffer_pool.md`](innodb/buffer_pool.md)、并行扫描归位 [`parallel_scan.md`](innodb/parallel_scan.md)、刷脏同前、dblwr 归位 [`dblwr.md`](innodb/dblwr.md)、**redo 的写/刷实现归位 [`redo_log.md`](innodb/redo_log.md)**——本篇保留各自的归位表与"真异步只读预读"等 I/O 视角结论 |
| [trx.md](innodb/trx.md) | InnoDB 事务（`trx_t`） |

### [feat/](feat/) —— 跨层端到端特性

> 这些主题**同时改动多个层**（语法 → 优化器 → 引擎存储），按"特性"归位，不按层拆开。判据见下面「第一步：决定放哪」与「边界归属」两节；**篇内怎么组织（按层分节 vs 生命周期）见 [`feat/README.md`](feat/README.md)**。

| 文件 | 内容 |
|---|---|
| [partitioning.md](feat/partitioning.md) | **分区表**（全链路单篇）：`partition_info` 元数据模型、DD 持久化与"DD→文本→重解析"往返、分区裁剪（假索引复用 range 优化器）、`Partition_helper` 的 DML 路由、`ha_innopart` 的分区上下文切换与 per-partition `dict_table_t`、TRUNCATE/EXCHANGE/ADD-DROP PARTITION、分区统计与能力边界 |
| [fts.md](feat/fts.md) | **全文检索**（全链路单篇）：`MATCH...AGAINST` 语法与 `Item_func_match` 生命周期、优化器 `FT_KEYPART`/`JT_FT` 与 hints 下推、`FullTextSearchIterator`、filesort 的 FTS 特例、11 张辅助表与 ilist/VLC 编码、`FTS_DOC_ID`、`fts_cache` 与 sync/后台 optimize、墓碑删除与 OPTIMIZE 重写、布尔 AST 三遍遍历、`tf × idf²` 打分、崩溃恢复、InnoDB 与 MyISAM 差异 |
| [auto_increment.md](feat/auto_increment.md) | **自增列**（全链路单篇）：三档 AUTOINC 锁模式与 `innobase_lock_autoinc` 逐分支（★ 默认 2 = NO_LOCKING 且为只读变量；mode1 检测到他人持表锁时"先放 mutex 再降级"规避死锁）、handler 区间分配与 `nb_desired_values` 的可靠性边界、★ 计数器持久化（`dict_table_autoinc_log` 写 redo + DDTableBuffer，8.0 重启不回退）、★ 纠正"重启必 SELECT MAX"（仅 IMPORT 无 cfg/表空时兜底）、DDL 保留计数器、空洞三来源与达到列上限行为 |
| [generated_columns.md](feat/generated_columns.md) | **生成列**（全链路单篇）：VIRTUAL/STORED 语义、`Value_generator` 元数据与打开表时 `PARSE_GCOL_EXPR` 重解析、`vfield` 单遍求值器（依赖位图 + 拓扑序假设）、读写两条求值链路与覆盖索引短路、虚拟列二级索引（索引页物化 vcol 值 + `innobase_get_computed_value` 回调 + purge 无 TABLE 开表求值）、undo 中的 vcol 旧值与 v_idx、binlog 不对称镜像与备库重算、instant ADD/DROP 虚拟列、隐藏生成列家族（功能索引/多值索引/GIPK） |

### [log/](log/) —— 服务器日志

[error_log.md](log/error_log.md) · [general_log.md](log/general_log.md) · [slow_log.md](log/slow_log.md)

### [cloud/](cloud/) —— 云环境

| 文件 | 内容 |
|---|---|
| [cloud_storage.md](cloud/cloud_storage.md) | **云存储（EBS / 云盘 / 分布式块存储）**：EBS vs 云盘的专名/通名之辨（★ Aurora/PolarDB 存储**不是**云盘）、网络块存储的定位与设计权衡（被否决的三方案、三副本为什么是 3、超售与 burst credit 的失效场景、块语义最小契约如何逼数据库自造 doublewrite、**控制权让渡**）、**attach/detach ≠ mount/umount** 的分层与 HA 换机链路、各家卷类型官方规格（gp3/io2/ESSD/增强型·极速型 SSD）、快照的 lazy load 冷启动与"快照≠数据库一致性备份"、RDS 三种存储形态（本地盘/云盘/存算分离）、★ MySQL 对云盘无感知的 grep 证据与五条隐含假设。（MySQL 侧 I/O 栈源码见 `innodb/io.md`） |
| [cloud_db.md](cloud/cloud_db.md) | 云数据库架构：数据面/支撑环境分层、存算分离、网络体系、HA 切换、透明切换 L0-L6 |
| [cloud_networking.md](cloud/cloud_networking.md) | 云上网络：VPC/子网、EIP/NAT、VIP/RS、L3-L4-L7、网关体系、安全组、PrivateLink、K8s 网络、VXLAN |

### [papers/](papers/) —— 外部论文剖析

> **与源码文档的区别**：源码文档以**本仓库 8.0.39 源码**为事实来源；论文剖析以**论文**为事实来源，并在「对 MySQL 与云数据库的启示」一节与源码文档对接。两者通过文首「边界」行双向互指。

| 文件 | 内容 |
|---|---|
| [btrlog.md](papers/btrlog.md) | **BtrLog（VLDB 2026）云上 WAL 日志服务**：三重困局（EBS 慢且贵 / 对象存储延迟高按次贵 / 专有后端不可复用）、**单写者假设如何省掉整个排序层**（Paxos 4 跳 vs Corfu 6 跳 vs Scalog 4 跳 vs BtrLog 1 RTT）、SSD 日志节点 Quorum + 对象存储异步段归档的分层设计、容错协议（wtoken fencing / 日志尾保守推断 / epoch 去重 / 段快照刷盘）、面向微秒级的工程（自研 io_uring 运行时、UDP、对称网络 + `reply_to`、LSN 窗口）、全量评估数据（70 µs vs EBS 318 µs、$0.00125 vs $0.0036 每百万追加、跨三云 2.6~6.4× 差距）；★ 附「**对象存储为什么成本低、延迟高**」的成本六因与延迟七因分析；含「批判性审视」与「对 MySQL 的启示」 |

---

## 如何新增文档

> 单篇的写法（章节骨架、必填项）见 [`_template.md`](_template.md)。本节讲**文档之间**的组织规则：放哪、怎么编号、边界怎么划。

### 第一步：决定放哪

```
新主题
  │
  ├─ ★ 是"跨层端到端特性"吗？（同时改 SQL 层与引擎层：分区、全文索引、GIS…）
  │     是 → feat/，一篇讲完整条链路；引擎差异写进篇内小节，
  │          不为每个引擎单独开篇（详见「边界归属」）
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
  │     是 → innodb/
  │
  ├─ 是外部论文的深度剖析吗？（事实来源是论文，不是本仓库源码）
  │     是 → papers/，一篇论文一个文件；文首「边界」行必须与相关源码文档互指
  │
  └─ 其他 server 层主题
        ├─ 数据类型（JSON/GIS/时间类型…）  → server/datatype/
        ├─ binlog 与复制                    → server/replication/
        ├─ 被全局复用的通用机制             → server/infra/
        ├─ 插件/组件/服务（可扩展框架）     → server/plugin/
        └─ 服务器日志                       → log/
```

### 第二步：判断该独立成篇，还是并入现有篇

这是最容易做错的一步。两个方向都要考虑：

**该独立成篇** —— 满足任一条：

- 有**自己的一整套数据结构**，讲不透就会拖累宿主篇
  （例：range 优化有 `SEL_TREE`/`SEL_ARG` 区间森林 + 六种访问方法，从"访问方法选择"里独立出来成 `08_range_optimizer.md`）
- 是**另一条并行的主链**，而非现有链的分支
  （例：DML 与 SELECT 共用前段但执行完全不同，独立为 `09_dml.md`，没有塞进 `runtime/`）
- 是**通用子系统**，被多个上层复用
  （例：LOB 是 InnoDB 通用机制，从 `json.md` 拆出为 `innodb/physical/lob.md`）

**该并入现有篇** —— 满足任一条：

- 拆开会让**主链断头**，读者走到边界就掉出目录
  （例：协议层收包原为 `server/net.md`，并入 `query/01`——否则主链第一步"SQL 文本从哪来"在目录外）
- 内容是**同一件事的两半**，分开必有一半无处安放
  （例：收包与回包是协议层的两个方向，独立成文会让回包没地方写）
- 篇幅撑不起一篇，且没有独立的数据结构

**★ 已确定独立成篇后，还要决定：原篇那份内容是"迁走"还是"两处都留"？**

判据是新篇与宿主篇的**关系**——这两种在本库里都发生过，处理方向**相反**：

| 关系 | 判据 | 处理 | 实例 |
|---|---|---|---|
| **另一个子系统** | 第一主语不同（新篇有自己独立的数据结构与生命周期） | **归位**：完整实现迁到新篇；宿主篇只留**概要 / 归位表 + 属于它自己视角的结论**，两边互指 | `dblwr` / `刷脏` / `读路径` / `AHI` / `change buffer` / `redo I/O` 从 `io.md` 拆出 |
| **另一个视角** | 同一机制换个角度讲（恢复视角看 redo、优化器视角看索引） | **并存**：宿主篇**保留完整版**，新篇写自己的视角并互指，**不要**搬走原篇内容 | `recovery.md` 里的 redo（`redo_log.md` 的完整版保留不动） |

✗ 两种常见错法（都返工过）：

- 把"另一个视角"当成"另一个子系统"，**搬空宿主篇** → 读者在原位置找不到内容
- 把"另一个子系统"既在新篇写、又在宿主篇留完整副本 → 两处必然漂移

> 归位时宿主篇**必须留的**是：① 一句边界说明 ② 一张归位表（写明每条去哪篇找）③ 属于本篇视角的那几条结论（不是复制正文）。

### 第三步：够不够建子目录

- **同一阶段 ≥ 3~4 篇** → 建子目录
  （例：逻辑优化 4 篇 → `logical/`，物理优化 4 篇 → `physical/`）
- **只有 1 篇** → 留在上级目录，**不建单文件目录**
  （例：`10_plan_refinement.md` 是计划改进阶段的唯一一篇，留在 `07_optimize/` 根）
- 建了子目录后，上级 README 必须**按阶段分组**展示，否则读者看不出层次

### 边界归属：第一主语原则

跨层主题（一件事横跨 SQL 层、接口层、引擎层）按**"第一主语是谁"**归位，而不是按"涉及谁"：

| 内容 | 第一主语 | 归属 |
|---|---|---|
| 迭代器怎么调 handler 取行 | SQL 层执行器 | `query/09_executor_iterator.md` |
| `position`/`ref`/`rnd_pos` 的接口语义 | 存储引擎接口 | `server/handler.md` |
| `row_search_mvcc`、游标推进 | InnoDB | `innodb/row_search.md` |
| DML 的 buffer row id 两阶段读 | SQL 层 DML | `query/09_dml.md` |

一份内容只在**一处**详写，其他篇用交叉引用指向它——不要因为"这里也相关"就复制一遍。跨层主题在文首加一行"边界"说明谁讲什么。

**例外：跨层端到端特性不走按层归位，进 `feat/`。** 判据（满足任一）：

1. **同时改动多个层**：分区表同时改 SQL 层（语法、元数据、优化器、MDL）与引擎层（存储、分区 DDL）；全文索引、GIS 同理。
2. **可能被多个引擎实现**：按引擎拆篇会让 N 个引擎变成 N 篇，且每篇都要复述一遍 server 层。正确做法是一篇 + "引擎差异"小节（例：分区表只 InnoDB/NDB 两家，NDB 走 `HA_USE_AUTO_PARTITION` 自动分区并保留 `nodegroup_id`，未声明 native 分区能力的引擎直接被拒）。
3. **读者需要一次读完闭环**：分区表的 DML 路径横跨 `ph_write_row`（SQL 层算分区号）与 `set_partition`（引擎换 `dict_table_t`），中间只隔一层虚调用，拆开则两篇都断头。

不属于此类的仍按层归位：纯引擎内部机制（buffer pool、redo）→ `innodb/`；纯 SQL 层机制（MDL、优化器）→ `server/`；**两层接口本身** → `server/handler.md`（它属于"接口"而非"特性"）。

> **★ `feat/` 篇的骨架另有要求**（两种形态，按"两层耦合方式"选），见 [`feat/README.md`](feat/README.md)「篇内骨架」。
> 这是全库**唯一**的目录级骨架规范；除它之外，单篇骨架一律以 [`_template.md`](_template.md) 为准。

### 编号规则

- 主链用数字前缀表示**时间顺序**（`01_` → `09_`）
- 子目录内**编号全局连续、不重置**：`07_optimize/logical/02_` 一直排到 `physical/09_`
  - 这样跨篇引用"见 03 篇"**永远有效**，移动文件不必改引用
  - 用**路径**区分命名空间（写 `07_optimize/01_cost_model`），不靠"移出去平级"来避免编号冲突
- 横切内容（`runtime/`、EXPLAIN）不参与主链编号语义，README 里单独分组

### 命名

- 全库**英文文件名**、小写、下划线分隔
- 名字要能看出**内容归属**：`04_logical_join.md` / `06_join_order.md` 一眼能分清逻辑优化和物理优化；反例是曾经的 `06_join_order_access.md`，把 greedy + range + const 全挤在一起，看不出是物理优化

### 收尾检查清单

新增或移动文档后，这四件必做：

1. **改上级 README** 的文件索引（含新篇的一句话内容摘要）
2. **改被影响篇的交叉引用**（尤其是拆分/合并时，原篇的指向会失效）
3. **跑链接校验**——相对路径层级最易错（`../` 还是 `../../`）：
   ```bash
   cd .codebuddy/knowledge_for_mysql
   for f in $(find . -name "*.md"); do d=$(dirname "$f"); \
     for l in $(grep -o "](\([^)#]*\.md\)[^)]*)" "$f" | sed 's/](//;s/)$//;s/#.*//'); do \
       [ -e "$d/$l" ] || echo "BROKEN: $f -> $l"; done; done
   ```
4. **写好本篇的「参考」章节**（论文 / 官方文档 / 内核月报，按相关性取舍）

---

## 准入标准

> 上一章讲**放哪**（组织），本章讲**够不够格进、要写成什么样**（质量）。
>
> 本章每一条都来自真实返工——不是假想的规范。

### 零、先明确：什么是好文档

**目标是内核月报式的深度剖析**——讲透设计思想与核心算法，**不是**教科书式的"是什么 / 有哪些 / 怎么配置"罗列。

读者是"想真正理解 MySQL 设计的人"，不是"想查某个函数在第几行的人"。

一篇合格的文档要能回答三个问题，按重要性排序：

| # | 问题 | 落在哪一章 |
|---|---|---|
| **1** | **怎么设计的？为什么这么设计？用了什么思想理论？**（权衡取舍、被否决的方案、论文溯源、他库对比） | 「理论基础」——本章是文档价值的核心 |
| **2** | **重点难点是什么，怎么讲清？**（反直觉处、隐含假设、复杂状态机） | 「核心实现」——**贴关键代码 + 逐段解释算法**，只给结论等于没写透 |
| **3** | **怎么演进到今天的？**（各版本变了什么，以及**为什么变**） | 「概述·版本演进」记变化清单；「理论基础」记为什么变 |

**加一条结构要求：模块要闭环。** 读者读完这一篇（或这个子目录），对该模块应形成完整认知，不留"讲了一半"的断口——上游谁调它、下游它调谁、边界在哪，都要交代（哪怕只一句交叉引用）。

> **不写源码位置索引。** 读者会自己看代码，不需要文档告诉他"在哪个文件第几行"。函数名/结构名照常出现——它们是**指称用的术语**（讨论 `choose_table_order` 做了什么），不是坐标。也不要专门列"源码位置速查表"。
>
> **执行细则**（完整版见 [`_template.md` ⑧](_template.md)，两处重复时以 template 为准）：① **行号一律不写**——版本一变就失效，而且会把注意力从"为什么"拉到"在哪"；② 文件名最多在首次交代"这个函数属于哪个模块"时提一次，之后只用函数名；③ 参考章只列论文/文档/月报/相关文档，**不列源码清单**；④ **不列"关键函数速查"表**（函数 ↔ 一句话职责）——那是把正文抄一遍，且必然漂移，函数名在讲到它的地方就地交代。
>
> **存量（早期文档）里的行号与「关键源码位置速查」章保持现状，不作回溯清理**；新写和重写的篇按上面四条从严。

### 一、三条硬门槛

| 门槛 | 含义 | 不满足的表现 |
|---|---|---|
| **有理论基础** | 讲清设计思想、权衡、理论溯源 | 只有"代码这么写的"，没有"为什么这么写" |
| **讲透实现** | 关键代码贴出来并逐段解释算法 | 只列结论、只给函数名清单 |
| **闭环** | 该模块认知完整，边界与上下游交代清楚 | 读完不知道这一步之后发生什么 |

**篇幅要撑得起。** 同族篇的基线：`redo_log` 2151 / `io` 1660 / `dblwr` 1126 / `ahi` 975 / `buffer_pool` 869 行。
新篇若只有三四百行，通常意味着"核心实现"没展开——**写了骨架没写血肉**。判据：关键函数贴全了吗、
每个分支都解释了吗、"为什么"都答了吗。（不是凑字数，是别在讲透之前收尾。）

**说的东西必须真实存在**——函数名、结构成员、默认值都经本仓库源码核实（见「三、事实核验」）。

**基线版本是本仓库的 MySQL 8.0.39。** 其他版本只在"版本演进"里作为对比出现。

### 二、重点难点怎么讲

一篇里通常只有**一两个真正的难点**，把它们讲透比均匀铺开更有价值。哪些算难点：

- **反直觉**：排序/物化/聚合发生在 `Init()` 而非 `Read()`；迭代器不拥有行缓冲
- **易混淆的命名**：handler 的 `ref`（行位置）与 EXPLAIN 的 `ref` 列（访问类型）毫无关系；PSI batch mode 是**监控**批量不是计算批量
- **有隐含假设**：ROR-intersection 的选择率乘数把不同列当独立相乘——这既是设计简化，也是 bad case 的根因
- **状态机复杂**：NLJ 的 NULL 补全为什么必须区分"第一条内层行"

讲透的标准：**解释清"为什么不得不这样"**，而不只是"它是这样"。

> 设计里的简化假设带来的**失效场景**、以及**阈值下的退化行为**（如 `eq_range_index_dive_limit` 只缓解 index dive、不缓解 SEL_ARG 树构造），写在「理论基础 → 设计思想与权衡」里——它们是权衡的另一半，跟产生它们的设计决策放在一起才讲得通。

### 二·补：讲清复杂性的三种手段

难点只靠文字往往讲不透。以下三件是本库反复验证有效的手段：

> **本节是摘要，执行细则见 [`_template.md` ⑪⑫⑬](_template.md)**——两处重复时以 template 为完整版。

**① 复杂处必须配图**（文字说不清的一律画图）

| 类型 | 用什么 |
|---|---|
| 结构关系（继承树、结构体嵌套、内存/文件布局） | ASCII 树 / 分块图 |
| 状态与流转（状态机、生命周期、调用链） | ASCII 缩进链，或 mermaid `stateDiagram` |
| 并发与时序（多线程协作、加锁顺序、事件唤醒） | mermaid `sequenceDiagram` |
| 分层与数据流（跨层路径、缓冲区读写） | ASCII 分层框图 |

> **ASCII 优先**：纯文本、diff 友好、任何环境可读；mermaid 只在"节点多、连线复杂"时用。
> ✗ 常见错法：用大段文字描述树状继承关系；用列表罗列状态转移。

**② 涉及 C++ 类体系就加一章「★ 本机制里的 C++ 运用」**

判据：有继承体系 / 模板 / 所有权转移 / RAII / C 与 C++ 边界。写法是"为什么用、代价是什么"，
**不是语法教学**。参考 [`server/infra/io_cache.md`](server/infra/io_cache.md) 与 [`server/dd/dd.md`](server/dd/dd.md) 的同名章。
（C 结构如何被安全地包进 C++ 类——这条最能体现工程权衡，有就必写。）

**③ 面向二次开发：Misc 里给"动手改"需要的信息**

- **扩展点**：加一种新类型/新分支要改哪几处
- **坑与已知缺陷**：源码 TODO/FIXME 注释、**社区边界澄清**（"社区版没有 X"）
- **怎么观测**：status 变量、PFS instrument、`SHOW ENGINE` 段落、debug 开关

### 三、事实核验：六类高频错误

**这六类是本知识库真实踩过的坑**（虽然定位精度要求降低了，但"说的东西必须真实存在"这条不放松）：

#### A. 函数名不存在（最高频）

来源：注释里的**伪代码**、**历史版本**名字、**其他分支**（MariaDB/Percona）名字。

| 写错的名字 | 8.0.39 实际 | 错因 |
|---|---|---|
| `find_min_ror_intersection_scan` | `get_best_ror_intersect` + `find_intersect_order` | 只是伪代码注释 |
| `Optimize_table_order::best()` | `choose_table_order()` | 5.x 历史名字 |
| `prepare_unique` | 已内联进 `IndexMergeIterator::Init()` | **头注释里的方法名**，代码已重构 |
| `MRR_HANDLER_BUFFER` | `HANDLER_BUFFER` | MariaDB 的名字 |
| `ha_autocommit_or_rollback` | `trans_commit_stmt` | 已删除 |

> ★ **注释里出现的函数名不算存在**——头注释常年不更新，`index_merge.h` 至今写着"implemented in `prepare_unique`"，而那个方法早就没了。每个函数名都要 grep 确认。

#### B. 结构成员已删除或从不存在

`POSITION::sjmat_lookup_tables`（8.0.39 已删）、`IndexMergeIterator::m_dedup`（不存在，是优化器侧局部变量）、`TableRowIterator::m_expected_rows`（在各子类上，不在基类）。

> 贴结构体时打开头文件逐字段对照，不凭记忆列字段。

#### C. 默认值凭印象

`subquery_to_derived` 默认 **OFF**（常被误认为 ON）；`KEY_COMPARE_COST` 是 **0.05**（不是 0.1）。

> 必须从 `sys_vars.cc` / `opt_costconstants.cc` 的定义处读出。

#### D. 实现机制的版本前提变了

8.0.39 的**多表** UPDATE 延迟缓冲用**临时表**，`open_cached_file + position()` 只用于**单表** UPDATE 改扫描键的快路径。旧资料混为一谈。

> 写"X 用 Y 实现"时，确认这条路径在 8.0.39 真的还走 Y。

#### E. 把别家特性当社区特性

| 特性 | 实际归属 |
|---|---|
| `inlist2join` | Percona Server |
| table elimination | MariaDB |
| 列相关性 / dependency 统计 | PolarDB 在做，社区**无** |
| 向量执行、列式加速 | HeatWave（商业闭源），社区**无** |
| SQL 层并行查询 | 社区**无**（只有 InnoDB 内部 `Parallel_reader`） |

> 读商业版/其他分支资料时逐个 grep 确认。**社区没有的，宁可写"没有"也不能含糊带过。**

#### F. 把注释/文档当实现

头注释描述的算法可能已被重构（见 A 的 `prepare_unique`）。**以实际代码为准，注释只作线索。**

### 四、外部材料（月报 / 论文 / 博客）怎么消化

**外部材料提供视角与理论，源码提供事实。**

这几轮的经验：**月报的函数名和行号几乎全部对不上 8.0.39**（多基于 5.7 或某分支）。流程固定三步：

1. **读材料，提取"问题视角"与"理论脉络"**：它关注什么问题、指出什么 bad case、背后是哪篇论文的思路
2. **回源码核实机制与函数名**
3. **写文档时以源码为准，并显式记录纠正**

案例：月报《Index-Merge 代价估算原理》给了极好的问题视角（为什么 index merge 常选错），但它的 4 个函数名在 8.0.39 全不存在。最终文档保留了**算法思路和 bad case 根因**，函数名全部换成源码实际的。

> **禁止**照抄外部材料的函数名/公式而不核实。这是本知识库返工最多的原因。

### 五、什么不该进

| 不该进 | 处理 |
|---|---|
| 无法落到源码的传闻、性能玄学 | 不写 |
| 别家分支/商业版的私有实现细节 | 只在「理论基础 → 他库对比」里提及，明确标注归属 |
| 从外部材料抄来但没核实的内容 | 核实后再写 |
| 已被本版本删除的机制 | 只进"版本演进"作为沿革 |
| 同一内容在第二处重写 | 交叉引用指向唯一详写处 |

### 六、"社区没有 X" 这类澄清怎么处理

读者常问相邻概念（向量执行？并行查询？IN list 转 join？）。这类澄清**有价值**，但要：

1. **给证据**（"全库搜 `inlist2join` 返回 0 匹配"），不是断言
2. **说清替代品**——社区用什么解决同类问题
3. **说清谁有**——哪个分支/商业版的特性
4. **不为"没有"单独造一篇**——挂在最相关的篇里

> 例外：`innodb/parallel_scan.md` 单独成篇，因为有真实的 `Parallel_reader` 可讲，"社区无 SQL 层并行查询"只是其中一节。

### 七、提交前自检

**内容五条**（组织四条见上一章「收尾检查清单」）：

1. **「理论基础」写了吗**——设计思想、权衡、理论溯源、演进动机
2. **关键代码贴了并逐段解释了吗**——只给结论等于没写透
3. **模块闭环吗**——上下游、边界都交代了吗
4. **难点讲透了吗**——能解释"为什么不得不这样"，不只是"它是这样"
5. **函数名/成员/默认值核实过吗**——不照抄外部材料、不信注释里的函数名

> 单篇文档模板见 [`_template.md`](_template.md)；参考资料（论文 / 官方文档 / 内核月报 / 书籍）
> 在**每篇文档末尾的「参考」章节**，不在全局维护索引。
