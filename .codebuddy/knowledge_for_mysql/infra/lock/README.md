# lock：锁与同步（跨层主题）

> MySQL/InnoDB 的锁横跨 `include/`（原语封装）、`mysys/`（原语实现）、`sql/`（MDL 等 server 层锁）、`innodb/`（行锁 + 内部 latch）四个模块，故按**主题**独立成目录，不按层拆散。本篇是对 MySQL 8.0 **全部锁的盘点**。

## 一、最重要的分类：并发原语 vs 事务锁

MySQL 里的"锁"其实是**两类完全不同的东西**，读任何锁相关代码前先分清它属于哪一类：

| 维度 | **并发原语**（concurrency primitives） | **事务锁 / 语义锁**（transactional / semantic locks） |
|------|-------------------------------------------|------------------------------------------------------|
| 本质 | 并发编程**基础设施** | 数据库**语义**的一部分 |
| 解决什么问题 | 多线程如何**安全访问共享内存**（临界区互斥、内存可见性） | 事务之间如何**协调对数据库对象的访问**（表结构、数据行） |
| 持有者 | 线程 | 事务 |
| 生命周期 | 临界区（微秒级，`mutex_enter` → `mutex_exit`） | 可跨整个事务（毫秒~秒级，提交/回滚才释放） |
| 保护对象 | 内存对象：缓存条目、队列、引用计数 | 数据库对象：表元数据、索引记录、间隙 |
| 语义 | 无业务含义，纯技术手段 | 有业务含义：可表达"并发 DDL 与 DML 互斥"、"幻读防护" |
| 死锁处理 | 一般不检测（靠加锁顺序规约避免；个别如 latch 有内部调试检测） | **检测并回滚受害者**（InnoDB 死锁检测、MDL wait-for graph） |
| 典型例子 | **同步原语**：mutex / rwlock / cond / InnoDB latch；**无锁结构**：RCU / LF_HASH / `ut_lock_free_hash_t` / `Seq_lock` / `Link_buf` | MDL / 行锁 / 间隙锁 / 表锁 / AUTOINC 锁 |

**一句话判据**：问"这把锁在保护**内存**还是保护**数据库对象**？由**线程**持还是**事务**持？"——保护内存、线程持、临界区级 → 并发原语；保护表/行、事务持、跨事务 → 事务锁。

**并发原语下再分两个子类**（这就是 LF_HASH 这类无锁容器不该叫"同步原语"的原因）：

| 子类 | 是什么 | 例子 |
|---|---|---|
| **同步原语**（synchronization primitives） | 互斥/同步**机制**：临界区互斥、等待通知 | mutex / rwlock / cond / sem / event / latch |
| **无锁数据结构**（lock-free data structures） | 用 atomic/CAS 搭的并发**容器**，对用户透明地并发增删查 | LF_HASH / `ut_lock_free_hash_t` / RCU / `Seq_lock` / `Link_buf` / MPMC 队列（文档在 [`../infra/structure/`](../structure/)） |

两者共同点：线程持、保护内存、临界区级；区别：同步原语是"加锁/解锁"的动作，无锁结构是"**用原子操作取代锁**"的成品容器。

**为什么不能混**：

1. **层次不同**：原语在事务系统**之下**——InnoDB 的行锁实现（`lock_sys` 的 hash 表、等待队列）本身就靠 latch/mutex 保护。
2. **语义不同**：原语没有回滚概念；事务锁有（事务回滚 → 行锁释放）。
3. **死锁处理不同**：原语死锁是**编程 bug**；事务锁死锁是**运行时正常现象**（检测 + 回滚）。

## 二、锁的类型学（MySQL 官方 LOCK ORDER 视角）

### 2.0 多维总分类视图（先看全景，再进细节）

MySQL 的锁可以沿三个正交维度切分，任何一把锁都落在下面的格子里：

| | **并发原语**（线程持） | **事务锁**（事务持） |
|---|---|---|
| **标准库 / mysys** | `std::mutex`/`std::shared_mutex`、`mysql_mutex_t`/`rwlock`/`prlock`/`cond`、`my_atomic`、RCU | `THR_LOCK` 排队锁（LOCK TABLES 表锁语义） |
| **sql（server 层）** | THD 族 / binlog 组提交 / GTID 复制 / table cache / DD 缓存 / XA / 时区等实例（A2） | **MDL 框架**（18 namespace，含备份锁、用户级锁）+ `Global_read_lock`（FTWRL） |
| **innodb（引擎层）** | `PolicyMutex`/`rw_lock_t`/`os_event`/`latch_t` 等自研原语 + 100+ 全局实例（A3） | `lock_t` 体系（表锁 IS/IX/S/X/AUTOINC + 行锁 record/gap/next-key/insert intention + 谓词锁） |

**生命周期维度**（事务锁专用）：语句级（MDL_STATEMENT）→ 事务级（MDL_TRANSACTION、InnoDB 行锁）→ 显式级（LOCK TABLES、GET_LOCK）→ 实例级（FTWRL、备份锁）。

> **盘点口径**：只覆盖 **MySQL（server 层）+ InnoDB**，三个层次（标准库/mysys、sql、innodb）的**原语类型 + 全局实例 + 事务锁体系**；其他引擎（MyISAM/ARCHIVE/CSV 等）与插件自带锁（半同步/组复制/审计等）**不纳入**。每页 per-page latch 等"按对象实例化"的锁按类型计数（如 `buf_block_lock` 每页一把，盘点按类型）；**cond 变量是 mutex 的伴生物**，在 A2 单独列清单。

### 2.1 MySQL 官方 LOCK ORDER 的类型学

`sql/debug_lock_order.cc`（8.0 的锁序图工具，配合 `LOCK_ORDER` 编译选项）是官方维护的**全锁清单**，它把同步原语按 PFS 命名分成 6 类：

| 锁序类型 | 实现 | PFS 前缀 | 例子 |
|---------|------|---------|------|
| `mutex` | `mysql_mutex_t` | `wait/synch/mutex/sql/` | `LOCK_open` |
| `rwlock` | `mysql_rwlock_t` | `wait/synch/rwlock/sql/` | `global_sid_lock` |
| `prlock` | `mysql_prlock_t`（**priority rwlock**：读者永不阻塞读者，即使写者排队） | `wait/synch/prlock/sql/` | `MDL_context::m_LOCK_waiting_for` |
| `sxlock` | InnoDB `rw_lock_t`（S/SX/X 三态） | `wait/synch/sxlock/innodb/` | `dict_operation_lock` |
| `cond` | `mysql_cond_t` | `wait/synch/cond/sql/` | `COND_open`（与 `LOCK_open` BIND） |

> 说明：`file`（OS 文件锁 `my_lock`，外部锁定）在 LOCK ORDER 里是第 6 类，但它是 **MyISAM 外部锁定的遗留**，8.0 已基本移除——按"只考虑 MySQL+InnoDB"口径不纳入。

> 注意 **prlock ≠ 普通 rwlock**：`mysql_rwlock_t` 写者排队后读者会被间接阻塞；`mysql_prlock_t`（priority）读者永远不被写者阻塞——MDL 的等待队列（`m_LOCK_waiting_for`）必须用它，否则死锁检测会卡死。InnoDB 的 `rw_lock_t` 则独有 **SX（shared-exclusive）** 态（供 B-tree 热点页分裂：阻止新读、不阻止在途读）。

**锁序校验的两套体系与分层**（这是"锁序"主题的全局地图）：

| 体系 | 归属 | 覆盖范围 | 坐标系 | 详细剖析 |
|---|---|---|---|---|
| **LatchDebug**（latch level 金字塔） | **InnoDB 专属**（UNIV_DEBUG） | 仅 InnoDB 自研原语（100+ 全局实例 + 页锁） | `latch_level_t` 枚举（sync0types.h 头部有整幅 ASCII 金字塔图） | [`primitives/innodb_sync.md`](primitives/innodb_sync.md)「LatchDebug 的锁序金字塔」 |
| **LOCK ORDER 工具**（`sql/debug_lock_order.cc`） | **server 层跨层**（`WITH_LOCK_ORDER` 构建） | server + InnoDB **所有 PFS 锁** | **PFS instrument 名**（`wait/synch/...`） | [`lock_order.md`](lock_order.md) |
| **PFS 锁 instrument** | 两者共享的**锁命名层** | `wait/synch/mutex\|rwlock\|sxlock\|cond\|prlock` + 事务锁表 `wait/lock/*` | PFS 名 | 观测见各篇「可观测性」章 |

**为什么是两套而不是一套**：LatchDebug 只能管 InnoDB 自己（它不认识 mysys 的 `mysql_mutex_t`）；LOCK ORDER 工具以 PFS 名为通用坐标，才能把 server 与引擎的加锁关系画进**同一张全局有向图**（配 `lock_order_dependencies.txt` 手工声明允许的依赖，运行时判环 = 死锁风险）。演进方向：从"InnoDB 内部金字塔"走向"以 PFS 名为通用坐标的全局图"。

**事务锁侧收敛为四大体系**（这是盘点的主线）：

```
MDL 框架（sql/mdl.*）      ─┬─ 常规元数据锁（TABLE/SCHEMA/GLOBAL/COMMIT...）
                            ├─ ★ BACKUP_LOCK namespace  → 备份锁（LOCK INSTANCE FOR BACKUP）
                            └─ ★ USER_LEVEL_LOCK namespace → 用户级锁（GET_LOCK）
THR_LOCK 排队锁（mysys/thr_lock.cc）→ LOCK TABLES 表锁
InnoDB lock_t（lock0lock.cc） ─┬─ 表锁（IS/IX/S/X + AUTOINC）
                               └─ 行锁（record/gap/next-key/insert intention）+ 谓词锁
Global_read_lock（sql_class.h）→ 全局读锁（FTWRL）
```

**收敛性洞察**：server 层的事务锁看似很多（元数据锁、备份锁、用户级锁……），其实**都收敛在 MDL 一个框架**里（不同 namespace）；引擎侧收敛在 `lock_t` 一个结构里（`type_mode` 区分表锁/行锁）。真正独立的小类是 THR_LOCK 表锁和 FTWRL 全局读锁。

## 三、盘点 A：并发原语（线程持、保护内存）

### A1. 原语谱系：两棵从 OS 派生的"族谱树"

**server 与 innodb 各自独立地从 OS 原语往上叠了 2~4 层**——先理清谱系（哪些类型衍生自哪个基础），再看实例（A2/A3），才不会漏。所有"锁类型"都能挂到这两棵树上。

#### 谱系树 1：server 侧（mysys + PFS 埋点）

```
L0  OS 原语        pthread_mutex_t / pthread_rwlock_t / pthread_cond_t / sem_t / Windows SRWLock / futex
                    │  （mysys 做平台适配 + 可关闭的埋点开关）
L1  可移植封装      my_mutex_t / my_rwlock_t / my_cond_t        （mysys/my_thr_init.cc、thr_rwlock.cc）
                    │  （mysql_thread.h 包一层 PSI 埋点）
L2  PFS 埋点版      mysql_mutex_t / mysql_rwlock_t / mysql_prlock_t / mysql_cond_t
                    │  ★ mysql_prlock_t = priority rwlock（读者永不阻塞，与 mysql_rwlock_t 同源不同实现）
L3  实例            LOCK_open、THD::LOCK_thd_data、Global_sid_lock、COND_open ...（见 A2 全清单）
```

旁支（不经过 L1/L2 的独立族）：

| 族 | 位置 | 说明 |
|----|------|------|
| `std::mutex`/`std::shared_mutex`/`std::atomic` | C++ 标准库 | 8.0 **新模块直接用标准库**（XA `m_xa_lock`、DD 内部）——不带 PFS 埋点 |
| `my_atomic_*` | `include/atomic/` | 原子操作封装 |
| `MyRcuLock<T>`（RCU） | `include/my_rcu_lock.h` | 读多写少全局指针保护 ✅ [rcu.md](primitives/rcu.md) |
| `LF_HASH`（无锁哈希） | `include/lf.h` + `mysys/lf_hash.cc` | 无锁可扩展哈希（split-ordered list + hazard pointer），MDL_map/ACL cache 用 ✅ [hash.md](../structure/hash.md) |

#### 谱系树 2：InnoDB 侧（实现 + 策略两维模板）

```
L0  OS 原语        futex（**裸 syscall，无封装层**）/ std::atomic / pthread（cond + mutex）
                    ⚠️ sem_t 在 8.0.39 已完全不用；无 ut0futex.h / FutexWait
                    │
L1  底层等待         os_event（manual-reset 事件，包 pthread_cond）/ sync0arr（wait array）
                    ⚠️ 「FutexWait」不存在（旧资料/月报常见），futex 由 ib0mutex.h 直接 syscall
                    │
L2  实现模板        TTASFutexMutex<Policy>（TTAS 自旋 + futex 睡眠）
  （ib0mutex.h）    TTASEventMutex<Policy>（TTAS 自旋 + os_event 睡眠）
                    OSTrackMutex<Policy>（跟踪包装）
                    PolicyMutex = 按 UT_LOCKS_USE_FUTEX 选上述之一
                    rw_lock_t（sync0rw.h:375，S / X / SX 三态读写锁）
                    │  ★ 模板参数 <Policy> 是"横切关注"的注入点（见下）
L3  策略层          NoPolicy（release 默认空实现）
  （sync0policy.h） GenericPolicy（per-instance 统计 spins/waits/calls）
                    BlockMutexPolicy（聚合统计——buf block mutex 太多，不能逐个计）
                    MutexDebug（UNIV_DEBUG 下：锁序检查、魔法数、持有者跟踪）
                    │
L4  实例            dict_operation_lock、buf_pool->LRU_list_mutex、index->lock、
                    lock_sys->latches 分片、trx_sys->serialisation_mutex ...（见 A3 全清单）
```

InnoDB 旁支（不挂主干的独立原语）：

| 族 | 位置 | 说明 |
|----|------|------|
| `latch_t` | `sync0types.h` | UNIV_DEBUG 下所有可校验 latch 的**调试基类**（持 `latch_id_t`、`get_level()` 查金字塔锁序），`rw_lock_t` 在 DEBUG 下继承它——**不是** buf 页 latch（那是 `buf_block_t::lock`，一个 `rw_lock_t`）。声明式校验是 `ut::Stateful_latching_rules`（目前唯一用户是 `buf_page_t::io_fix` 状态机）。见 [`primitives/innodb_sync.md`](primitives/innodb_sync.md) |
| `Seq_lock` | `ut0seq_lock.h` | 序号锁（seqlock）：读者无锁读计数器，引 HPL-2012-68 ✅ [seq_lock.md](primitives/seq_lock.md) |
| `ut_lock_free_hash_t`（无锁哈希） | `ut0lock_free_hash.h` | 开放寻址 + 链表数组扩容 + 256 分片无锁引用计数，bp 每索引页数统计 ✅ [hash.md](../structure/hash.md) |
| latch 序与调试 | `lock0latches.cc`（latch_level + `Shard_latches_guard`）、`sync0debug.cc` | 按级别加锁防死锁 + 锁序调试 |

**谱系的阅读价值**：①server 侧 `mysql_*_t` 都是 L1 的 PFS 包装——所以 server 锁都能被 `performance_schema.mutex_instances` 看到，而 InnoDB 用 L2/L3 自研实现，只有通过 `sync0sync.cc` 注册 PFS key 的那批（A3）才可见；②InnoDB 的 Policy 模板把"锁机制"（L2）与"统计/调试"（L3）正交分解——这是它比 server 侧先进的地方（server 侧埋点硬编码在 L2 里）；③`mysql_prlock_t` 与 `mysql_rwlock_t` 同源（L1）不同实现——前者读者永不被写者阻塞。

### A2. server 层原语实例清单（mutex 为主 + 少量 rwlock / prlock / cond）

这些是 A1 各族的**实例**，`sql/debug_lock_order.cc` 的锁序图是权威清单。**server 层 99% 是 mutex**（`mysql_mutex_t`），rwlock/prlock 只有极少数（见 A2.2）——所以下面主表按域列 mutex 实例，非 mutex 的单独抽出来。★ = 高频/重要：

| 域 | 锁实例 |
|----|--------|
| **THD per-thread 族** | ★ `LOCK_thd_query`（查询状态/KILL）、★ `LOCK_thd_data`（THD 数据字段）、`LOCK_thd_sysvar`（session 变量更新，`sql_class.h:1211`）、`LOCK_thd_protocol`、`LOCK_thd_security_ctx`（安全上下文）、`LOCK_current_cond`（当前条件变量）、`LOCK_query_plan`（8.0.24+，EXPLAIN 查询计划）；`TABLE_SHARE::LOCK_ha_data`（per-table 引擎数据） |
| **table cache** | ★ `LOCK_open`（全局 + 条件 `COND_open`）、`LOCK_table_cache`（老表缓存锁）+ **16 分片 `Table_cache::m_lock`**（`table_cache.h:93`，每个分片独立） |
| **DD 数据字典** | DD 对象缓存 `Shared_multi_map` 的分片 `mysql_mutex_t`（`sql/dd/impl/cache/shared_multi_map.h:118`，PSI key `key_object_cache_mutex`） |
| **XA / 2PC** | `XID_STATE::m_xa_lock`（`sql/xa.h:319`，**std::mutex**——XID 状态 hash 的全局保护） |
| **时区缓存** | `tz_LOCK`（`sql/tztime.cc:1064`，时区表/命名时区缓存） |
| **binlog / 组提交** | `LOCK_log`（binlog 文件句柄）、`LOCK_index`、★ `LOCK_commit`/`LOCK_sync`/`LOCK_group_commit`（组提交队列）、`LOCK_binlog_end_pos`（binlog 位点通知 + 条件 `update_cond`） |
| **GTID / 复制** | ★ `Global_sid_lock`（全局 GTID **rwlock**，`rpl_gtid.h`）、`LOCK_active_mi`、`LOCK_replica_list`（`mysqld.cc:1581`）、`LOCK_status`（状态变量数组）、`LOCK_compress_gtid_table`、`LOCK_reset_gtid_table`、`LOCK_replica_net_timeout`、`LOCK_replica_trans_dep_tracker`（并行复制 WRITESET 依赖）、`LOCK_sql_replica_skip_counter`、`key_mta_temp_table_LOCK`（MTA 并行复制临时表）、channel 三 rwlock（`channel_lock`/`channel_map_lock`/`channel_to_filter_lock`）、per-channel `mi->channel_wrlock`/`mi->data_lock`、per-`Rpl_info` 的 `LOCK_rpl_info`、`rli->data_lock`/`rli->run_lock`/`rli->log_space_lock`、**relay log 自己的锁**（`MYSQL_RELAY_LOG` 类自带 `LOCK_log`/`LOCK_index`/`LOCK_log_end_pos`——与 binlog 的 `MYSQL_BIN_LOG::LOCK_*` 同名但独立实例） |
| **连接 / 资源计数** | `LOCK_connection_count`（+`COND_connection_count`）、`LOCK_user_conn`、★ `LOCK_thd_list`（THD 全局链表）、★ `LOCK_manager`（THD manager 主锁 +`COND_manager`，`sql_manager.cc:59`）、`LOCK_prepared_stmt_count`（`mysqld.cc:1575`）、`LOCK_uuid_generator`（server_uuid，`mysqld.h:707`）、`LOCK_scheduler_state`（event scheduler）、`LOCK_plugin` |
| **系统变量** | `LOCK_global_system_variables`、`LOCK_system_variables_hash`（**rwlock**） |
| **账号/安全** | `LOCK_crypt`（DES 加密，`mysqld.h:708`）、`LOCK_default_password_lifetime`、`LOCK_mandatory_roles`、`LOCK_password_history`、`LOCK_password_reuse_interval` |
| **错误日志** | `LOCK_error_messages`（`mysqld.cc:1072`，错误消息资源保护）、`LOCK_error_log`（错误日志文件）、`LOCK_log_throttle_qni`（日志节流）、`LOCK_collect_instance_log` |
| **TLS / 连接** | `LOCK_tls_ctx_options`、`LOCK_admin_tls_ctx_options`（8.0.16+，TLS 上下文选项）、`LOCK_delegate_connection_mutex`（X 协议连接） |
| **binlog 加密** | `LOCK_rotate_binlog_master_key`（binlog 主密钥轮换） |
| **启动 / 监听** | `LOCK_server_started`、`LOCK_socket_listener_active`、`LOCK_start_signal_handler`、`LOCK_handler_count` |
| **杂项** | `LOCK_sql_rand`（`RAND()` 种子）、`LOCK_offline_mode`（OFFLINE MODE）、`LOCK_event_queue`（event scheduler 队列）、`LOCK_plugin_install`（plugin 安装，与 `LOCK_plugin` 分工）、`LOCK_tc`（事务协调）、`LOCK_cost_const`（代价模型常量）、`LOCK_global_conn_mem_limit`（8.0.28+ 全局连接内存）、`LOCK_keyring_operations`（keyring）、`MDL_wait::LOCK_wait_status`（MDL 等待状态）、hostname 缓存分片锁（`sql/hostname_cache.cc`） |
| **ACL cache** | 8.0 双层：MDL 的 **`ACL_CACHE` namespace**（见 B1）+ `sql_auth_cache.cc` 的 cache 读写锁 |
| **MDL 内部** | ★ `MDL_context::m_LOCK_waiting_for`（**prlock**，见 A2.2） |

#### A2.2 server 层的 rwlock / prlock / cond 实例（少数，单独列出）

**读写锁（`mysql_rwlock_t`，仅 5 个）**：

| 锁 | 域 | 说明 |
|----|----|------|
| `Global_sid_lock`（=`global_sid_lock`） | GTID | 全局 GTID 集合读写锁（`rpl_gtid.h`） |
| `channel_lock` / `channel_map_lock` / `channel_to_filter_lock` | 复制 | channel 映射/过滤器三个 rwlock |
| `LOCK_system_variables_hash` | 系统变量 | 系统变量 hash 表 |

**priority 读写锁（`mysql_prlock_t`，仅 1 个）**：

| 锁 | 说明 |
|----|------|
| `MDL_context::m_LOCK_waiting_for` | MDL 等待队列（读者永不阻塞，死锁检测依赖此特性） |

**条件变量（`mysql_cond_t`，伴生 mutex，共 11 个）**：`COND_open`（与 `LOCK_open`）、`COND_manager`（与 `LOCK_manager`）、`COND_connection_count`、`COND_thr_lock`、`COND_server_started`、`COND_socket_listener_active`、`COND_start_signal_handler`、`COND_handler_count`、`COND_cache_status_changed`、`COND_compress_gtid_table`、`COND_active`。

### A3. InnoDB 原语实例清单（mutex 为主 + 15 个 rwlock）

权威清单 = `sync0sync.cc` 的 PFS key 注册表（100+ mutex + 15 rwlock）。**InnoDB 的 rwlock（`rw_lock_t`）单独列在 A3.2**，主表只列 mutex。按域归类（★ = 全局级热点，详细机制见各模块文档）：

| 域 | 锁实例 | 详细机制文档 |
|----|--------|------------|
| **全局 DDL 闸门** | ★ `dict_operation_lock`（**rw_lock_t**：DDL 拿 X，正常操作拿 S——`debug_lock_order.cc` 里叫 "sxlock/innodb/dict_operation_lock"） | `innodb/ddl.md` |
| **lock_sys（锁系统自身）** | `lock_sys_table_mutex`、`lock_sys_page_mutex`（8.0.35+ 分片）、`lock_wait_mutex`、`lock_sys_global_rw_lock` | `innodb_trx_lock.md` |
| **redo log（log_sys）** | ★ `log_checkpointer_mutex`、`log_writer_mutex`、`log_flusher_mutex`、`log_write_notifier_mutex`、`log_flush_notifier_mutex`、`log_closer_mutex`、`log_limits_mutex`、`log_files_mutex`、`log_cmdq_mutex`、`log_sn_mutex` + `log_sn_lock`（LSN 序号锁）、`log_sys_arch_mutex` | `innodb/redo_log.md` |
| **buffer pool** | ★ `LRU_list_mutex`、`free_list_mutex`、`flush_state_mutex`、`zip_mutex`/`zip_hash`、`chunks_mutex`、per-block `buffer_block_mutex`、per-page `buf_block_lock`（rwlock） | `innodb/buffer_pool.md` |
| **trx_sys / MVCC** | ★ `trx_sys_mutex`、`trx_sys_shard_mutex`、`serialisation_mutex`（read view 生成与事务串行化，`trx0sys.h:508`）、`trx_mutex`、`trx_pool`/`trx_pool_manager`、`temp_pool_manager`、`trx_undo_mutex`、`trx_i_s_cache_lock`（rwlock） | `innodb/trx.md`、`mvcc.md` |
| **purge** | `purge_sys_pq_mutex`、`trx_purge_latch`（rwlock） | `innodb/undo_log.md` |
| **fil / 表空间** | ★ `fil_system_mutex` + 68 个 `Fil_shard::mutex`、per-space `fil_space_latch`（rwlock）、`file_open_mutex` | `innodb/io.md` |
| **AIO 子系统** | `os_aio_array::m_mutex`（每个 AIO 数组的槽位保护，"the mutex protecting the aio array"，`os0file.cc:715`） | `innodb/io.md` |
| **★ 索引树锁（"索引锁"）** | `dict_index_t::lock`（`dict0mem.h:1266`，`rw_lock_t`，**每个索引一把**，创建于 `dict0dict.cc:2561` 的 `rw_lock_create(index_tree_rw_lock_key, ..., LATCH_ID_INDEX_TREE)`）——B-tree 搜索持 S、修改（`BTR_MODIFY_LEAF`）持 X、页分裂 SMO 用 SX；保护的是**索引这个内存结构**（树高/分裂/合并），不是记录 | `index/btr.md` |
| **AHI** | `btr_search_latch`（AHI 分片 rwlock）、`ahi_enabled_mutex` | `index/ahi.md` |
| **dict / DD 引擎侧** | `dict_sys_mutex`、`dict_table_mutex`、`dict_persist_dirty_tables`、`dict_table_stats`（rwlock）、`index_online_log`（online DDL row log） | `innodb/dd.md` |
| **dblwr** | `dblwr_mutex` | `innodb/dblwr.md` |
| **FTS** | `fts_bg_threads`/`fts_delete`/`fts_optimize`/`fts_doc_id`/`fts_pll_tokenize` + `fts_cache_rw_lock`/`fts_cache_init_rw_lock` | `innodb/fts.md` |
| **change buffer** | `ibuf_mutex`、`ibuf_bitmap`、`ibuf_pessimistic_insert` | `index/ibuf.md` |
| **undo/rseg** | `undo_spaces_lock`（rwlock）、`rsegs_lock`（rwlock）、`trx_sys_rseg`、`undo_space_rseg`、`temp_space_rseg` | `innodb/undo_log.md` |
| **autoinc** | `autoinc_mutex`、`autoinc_persisted_mutex`、`ddl_autoinc_mutex` | `feat/auto_increment.md` |
| **恢复（recv）** | `recv_sys_mutex`、`recv_writer_mutex` | `innodb/recovery.md` |
| **R-tree（空间）** | `rtr_active`/`rtr_match`/`rtr_path`/`rtr_ssn` | `server/datatype/gis.md` |
| **clone / 归档** | `clone_sys`/`clone_task`/`clone_snapshot`、`page_sys_arch`/`page_sys_arch_oper`/`page_sys_arch_client` | — |
| **sync 自身注册表** | `mutex_list_mutex`、`rw_lock_list_mutex`（InnoDB 自维护的锁注册表，供 `sync0debug` 调试） | — |
| **srv / 杂项** | `srv_sys_mutex`、`srv_threads`、`srv_innodb_monitor`、`srv_misc_tmpfile`、`srv_monitor_file`、`page_cleaner_mutex`、`recalc_pool_mutex`、`flush_list_mutex`、`master_key_id`、`hash_table_locks`、`hash_table_mutex`（普通 hash 表）、`lock_free_hash_mutex`、`dict_foreign_err`、`sync_array_mutex`、`event_mutex`/`event_manager`、`parser_mutex`、`parallel_read`、`row_drop_list`、`zip_pad`、`page_zip_stat_per_index` | — |

#### A3.2 InnoDB 的 rwlock（`rw_lock_t`，S/SX/X 三态）实例（15 个）

InnoDB 的读写锁是自研的 `rw_lock_t`（不是 server 的 `mysql_rwlock_t`），独占 **SX（shared-exclusive）** 态。全部实例（`sync0sync.cc` 的 `UNIV_PFS_RWLOCK` 段）：

| rwlock 实例 | 保护对象 | 每对象一把？ |
|-------------|---------|-------------|
| ★ `dict_operation_lock` | **全局 DDL 闸门**：DDL 拿 X，正常操作拿 S | 全局唯一 |
| `lock_sys_global_rw_lock` | lock_sys 全局状态 | 全局唯一 |
| `index_tree_rw_lock`（=`dict_index_t::lock`） | 索引树结构（B-tree 搜索/分裂） | 每索引 |
| `buf_block_lock` | 每个 buf 页 | 每页 |
| `btr_search_latch` | AHI 分片 | 每分片（8 个） |
| `fil_space_latch` | 表空间 | 每表空间 |
| `undo_spaces_lock` / `rsegs_lock` | undo 表空间 / 回滚段 | 全局 |
| `trx_purge_latch` | purge 队列 | 全局 |
| `dict_table_stats` | 表统计信息 | 每表 |
| `hash_table_locks` | 内存 hash 表 | 每 hash 表 |
| `index_online_log` | online DDL 的 row log | 每索引 |
| `fts_cache_rw_lock` / `fts_cache_init_rw_lock` | FTS 缓存 | 全局 |
| `trx_i_s_cache_lock` | information_schema 的 trx 缓存 | 全局 |

> 另有 `buf_block_debug_latch`（仅 `UNIV_DEBUG`）不纳入。与 server 侧对比：**server 只有 5 个 rwlock，InnoDB 有 15 个**——因为 InnoDB 的树/页/表空间等"按对象实例化"的结构天然适合读写锁（读者并发高、写者少），server 层多是全局单例更适合 mutex。

### A4. 无锁容器全景清单（盘点盲区的补课）

> ★ 本节是初版盘点的"补漏"。复盘：初版以 `debug_lock_order.cc` 锁序图 + `sync0sync.cc` 的 PFS key 注册表 + mutex/rwlock 实例清单为锚点——**这三个锚点只登记同步原语**。无锁容器的本质是"用原子操作取代锁"，它们不注册任何 PFS 锁、不进锁序图，**用"锁清单"盘点锁，无锁结构天然在盲区**（当时 `ut_lock_free_hash_t` 只在旁支表留了 4 个字，`LF_HASH` 完全没出现）。正确的盘点对象是"**并发基础设施**"（同步原语 + 无锁容器），无锁容器要靠 grep `lock_free|lock-free|wait-free|lockless` 主动找。

| 结构 | 位置 | 类型 | 用户/用途 | 文档状态 |
|---|---|---|---|---|
| `LF_HASH` | `include/lf.h` + `mysys/lf_hash.cc` | 无锁哈希（split-ordered list + hazard pointer，有论文原型） | MDL_map / Acl_cache / PFS 对象表 | ✅ [hash.md](../structure/hash.md) |
| `ut_lock_free_hash_t` | `ut0lock_free_hash.h` | 无锁哈希（开放寻址 + 链表数组，无论文原型） | `buf_stat_per_index`（bp 每索引页数） | ✅ [hash.md](../structure/hash.md) |
| `ut_lock_free_cnt_t` | 同上 | 无锁引用计数（256 分片按 CPU 摊派） | 上述 hash 的数组节点配套 | ✅（同上篇内） |
| `MyRcuLock<T>`（RCU） | `include/my_rcu_lock.h` | RCU（读计数等零） | `ssl_acceptor_context_data` | ✅ [rcu.md](primitives/rcu.md) |
| `Link_buf<Position>` | `ut0link_buf.h` | 无锁环形"链缓冲"（乱序 add_link + tail 沿链推进） | redo `recent_written` / `recent_closed`（`log0sys.h`，cache line 对齐） | ✅ [link_buf.md](../structure/link_buf.md) |
| `Seq_lock<data_t>` | `ut0seq_lock.h` | 序号锁（回调式 seqlock，引 HPL-2012-68） | `mt_fast_modulo_t`（hash 表快速取模的 {mod,inv} 对） | ✅ [seq_lock.md](primitives/seq_lock.md)（锁模式，归锁原语） |
| `ib_counter_t` | `ut0counter.h` | 分片计数器（RDTSC 散槽 + cache line 隔离，读 fuzzy） | `srv0srv.h` 5 个 typedef + 全局统计 | ✅ [counter.md](../structure/counter.md) |
| `Counter::Shards` | `ut0counter.h` | 分片计数器第二代（真原子 + Pad，读精确） | bp `m_n_page_gets`、no-logging mtr、Parallel_reader | ✅（同上 counter.md） |
| `mpmc_bq` | `ut0mpmcbq.h` | 有界 MPMC 队列（Vyukov 算法，引 1024cores.net） | dblwr `Segments`/`Batch_segments`、FTS `Docq` | ✅ [queue.md](../structure/queue.md) |
| `Integrals_lockfree_queue` | `sql/containers/integrals_lockfree_queue.h` | server 层无锁队列（仅整型元素，虚拟索引 + MSB 占用位） | **唯一用户** `Bgc_ticket_manager`（binlog 组提交 ticket，8.0.28 开发周期） | ✅ [queue.md](../structure/queue.md) |
| `Bgc_ticket_manager` | `sql/binlog/group_commit/bgc_ticket_manager.h` | lock-free ticket 分配（63 位值 + 1 位 MSB 锁 + 计数对齐放行） | binlog 组提交 | ✅ binlog.md「BGC Ticket 系统」章已补底层实现 |

**口径外**（按"只考虑 MySQL server + InnoDB"不纳入，仅留线索）：TempTable 引擎的 `lock_free_pool`（`storage/temptable/`，属其他引擎）；MySQL Router 的 `mpmc_queue`/`mpsc_queue` 与组复制插件的 `gcs_mpsc_queue`（属组件）。

**盘点已按"类型一篇一主题"重组**（2026-09-20）：hash 三实现合 [hash.md](../structure/hash.md)、两个 MPMC 队列合 [queue.md](../structure/queue.md)、两代计数器合 [counter.md](../structure/counter.md)、`Seq_lock` 归锁原语 [seq_lock.md](primitives/seq_lock.md)、`Link_buf` 单篇 [link_buf.md](../structure/link_buf.md)；`Bgc_ticket_manager` 机制归位 binlog.md。

## 四、盘点 B：事务锁（事务持、保护数据库对象）

### B1. MDL 框架（`sql/mdl.h`）—— server 层事务锁的主框架 ✅ [mdl.md](transactional/mdl.md)

一个框架、**18 个 namespace**（`MDL_key::enum_mdl_namespace`，`mdl.h:400-421`）：

| namespace | 用途 |
|-----------|------|
| `GLOBAL` | 全局资源（如全库 flush、表空间全局） |
| ★ **`BACKUP_LOCK`** | 备份锁（`LOCK INSTANCE FOR BACKUP`） |
| `TABLESPACE` / `SCHEMA` / `TABLE` / `FUNCTION` / `PROCEDURE` / `TRIGGER` / `EVENT` | 各类数据库对象的结构保护 |
| `COMMIT` | 提交顺序锁（binlog 组提交顺序与引擎提交互斥） |
| ★ **`USER_LEVEL_LOCK`** | 用户级锁（`GET_LOCK`/`RELEASE_LOCK`，`item_func.cc:5220` 的 `User_level_lock`） |
| `LOCKING_SERVICE` | 锁服务（`GET_LOCK` 的插件接口版本） |
| `SRID` / `COLUMN_STATISTICS` / `RESOURCE_GROUPS` / `FOREIGN_KEY` / `CHECK_CONSTRAINT` | 空间参考系 / 直方图 / 资源组 / 外键 / 检查约束 |
| `ACL_CACHE` | ACL 缓存（8.0 权限缓存优化） |

> **收敛性**：备份锁和用户级锁都不是独立锁系统——`sql_backup_lock.cc` 的 `acquire_exclusive_backup_lock`（底层是 MDL **S** 锁，⚠️ 命名反转，见 B4 下方专节）和 `GET_LOCK` 都只是 MDL 框架的一个 namespace。写类操作（DDL/PURGE BINLOG 等）自动拿的是"S 备份锁"（底层 MDL **IX**；`binlog.cc` 的 `Shared_backup_lock_guard`，拿不到报 `ER_CANNOT_PURGE_BINLOG_WITH_BACKUP_LOCK`）。

### B2. 表锁体系（`THR_LOCK` 排队锁，`mysys/thr_lock.cc` + `sql/lock.cc`）✅ [thr_lock.md](transactional/thr_lock.md)

- `mysql_lock_tables` → 底层 `THR_LOCK`：通用排队读/写锁（**写优先**、并发插入的 4 处锁升级、防饥饿靠 `max_write_lock_count`）
- 承载：**`LOCK TABLES`** 语句（server 语义，InnoDB 表也支持但锁由引擎 `lock_t` 实现）——THR_LOCK 是 MySQL 3.x 就有、沿用至今的锁框架；InnoDB 的表锁不在这里（走 `lock_t` 的 `LOCK_TABLE`，见 B3）

**★ 三个必须记住的点**：

| 点 | 说明 |
|---|---|
| **InnoDB 上它是空壳** | `ha_innobase::lock_count()` 返回 **0**，`store_lock()` 不追加任何 `THR_LOCK_DATA` → `thr_multi_lock()` 拿到空数组。排队/冲突检测/优先级对 InnoDB 一行都不起作用；真正互斥由 MDL + InnoDB `lock_t` 分担 |
| **但 `thr_lock_type` 仍是跨层意图协议** | 被 `store_lock()` 翻译成 InnoDB 的 `select_lock_type`（LOCK_NONE/S/X）与 `select_mode`（SKIP LOCKED / NOWAIT），并派生 MDL 类型（`mdl_type_for_dml`） |
| **它完全没有死锁检测** | 用 `sort_locks()` 按 `(THR_LOCK* 地址, -type)` 全序 + "必须经 `thr_multi_lock()` 一次性申请"来预防；失败即整体回滚，无 victim 选择 |

### B3. InnoDB `lock_t` 体系（`storage/innobase/lock/lock0lock.cc`）✅ [innodb_trx_lock.md](transactional/innodb_trx_lock.md)

**关键结构**：`lock_t` **一统表锁与行锁**（`lock_get_type_low` 按 `type_mode` 区分 `LOCK_TABLE`/`LOCK_REC`，`lock0priv.ic:44`）——这就是为什么文档叫 `innodb_trx_lock.md` 而不是 `innodb_row_lock.md`。

| 形态 | 锁 | 说明 |
|------|----|------|
| `LOCK_TABLE` | 意向锁 **IS/IX** | 表级意向锁：行锁的"电梯"，DDL 判断表上是否有行锁 |
| `LOCK_TABLE` | 表级 **S/X** | `LOCK TABLES`、DDL 的表锁 |
| `LOCK_TABLE` | **AUTOINC 锁** | 自增列三档模式（跨层特性，详见 [`../feat/auto_increment.md`](../../feat/auto_increment.md)） |
| `LOCK_REC` | **record / gap / next-key / insert intention** | 行级四标志（`LOCK_REC_NOT_GAP`/`LOCK_GAP`/`LOCK_INSERT_INTENTION` 组合） |
| `PRDT_REC`/`PRDT_PAGE` | **谓词锁** | R-tree 空间索引（`lock0prdt.cc`） |
| — | 等待与死锁检测 | `lock0wait.cc` + `lock_deadlock_*`（waits-for graph + DFS 回滚受害者） |

### B4. 全局锁二件套（"锁整个实例"的两把锁）✅ [global_lock.md](transactional/global_lock.md)

MySQL 语境里的"**全局锁**"通常指这两把（加上只读模式的配合）。**两把都不是独立锁系统**，而是 MDL 框架的组合用法：FTWRL = `GLOBAL` S + `COMMIT` S，备份锁 = `BACKUP_LOCK` S：

| 全局锁 | SQL | 实现 | 锁什么 |
|--------|-----|------|--------|
| **全局读锁** | `FLUSH TABLES WITH READ LOCK` | `Global_read_lock`（`sql_class.h:829`）：`lock_global_read_lock`（拿 MDL 全局 + 表锁）+ `make_global_read_lock_block_commit`（阻塞新提交）两阶段 | 全库只读：阻塞 DDL/DML/提交 |
| **备份锁** | `LOCK INSTANCE FOR BACKUP`（8.0） | MDL 的 `BACKUP_LOCK` namespace（见 B1） | 语义上的 X 备份锁底层是 MDL **S**（⚠️ 命名反转，见下方专节）——阻塞写类操作（DDL/账户管理/PURGE BINLOG），**纯读不受影响**；备份会话持锁期间连它自己也不能 PURGE BINLOG（`Shared_backup_lock_guard` 的 owns 检查）；比 FTWRL（连读都堵）粒度更准 |

#### ★ 备份锁的命名反转：语义 X = MDL S，语义 S = MDL IX

`sql_backup_lock.cc:164-175` 有一段源码里少见的"为什么选这两个锁类型"的完整推导（原文翻译）：

> IX 与 S **互不兼容**，但各自与自身兼容；且 **IX 优先级低于 S**。备份是**稀有操作**，不应被高频的 DDL（用 IX）饿死——所以用 S 锁给备份，IX 锁给 DDL。因此 **S 锁应视为 Exclusive Backup Lock，IX 锁应视为 Shared Backup Lock**。

| 语义名 | 函数 | 底层 MDL 类型 | 谁在拿 |
|---|---|---|---|
| **Exclusive Backup Lock** | `acquire_exclusive_backup_lock`（`sql_backup_lock.cc:177`） | **`MDL_SHARED`** | `LOCK INSTANCE FOR BACKUP` 语句（`Sql_cmd_lock_instance::execute:59`，需 `BACKUP_ADMIN` 权限；XtraBackup 等备份工具执行的就是它） |
| **Shared Backup Lock** | `acquire_shared_backup_lock`（`:183`） | **`MDL_INTENTION_EXCLUSIVE`** | 写类语句自动拿：DDL、`TRUNCATE`、`ANALYZE`、`INSTALL/UNINSTALL COMPONENT`、`PURGE BINARY LOGS` 等 |

**实际效果**：纯读（SELECT）**不碰** `BACKUP_LOCK` namespace，完全不受备份影响；写类语句自动拿 IX，与备份的 S 冲突被阻塞——这就是"`LOCK INSTANCE FOR BACKUP` **阻塞写、放行读**"的精确实现。

三个细节：

1. **防饿死是选 S 的直接原因**：若反过来让备份拿 IX、DDL 拿 S，高频 DDL 的 S 请求会因优先级高不断插队，备份永远排不上——让"稀有方"拿高优先级的 S、高频方拿 IX 让位，备份先被满足。
2. ★ **锁对"实例状态"不对"会话豁免"**：`Shared_backup_lock_guard` 构造（`:81-90`）先查 `owns_equal_or_stronger_lock(BACKUP_LOCK, MDL_SHARED)`——**同会话已持 X 备份锁时，连它自己也不允许 PURGE BINARY LOG**（注释原文：*PURGE BINARY LOG is not allowed even when instance is locked for backup by the same session*）。
3. `LOCK TABLES` 模式下的自动获取在 `acquire_backup_lock_in_lock_tables_mode`（`sql_base.cc:5600`）：遍历表链，凡是 DDL/LOCK TABLES 类 MDL 请求（且非 `MDL_SHARED_READ_ONLY`）就拿一次 IX——一次遍历只拿一次即返回。

> **回看截图日志**（内部内核的备份 debug 输出，开源 8.0.39 全库无此字符串）：`global table version changed. start_global_table_version:0, end_global_table_version:5` 是内部增强——备份开始/结束各读一次"全局表版本"，不一致说明备份期间有 DDL 改过结构。开源的思路不同：**不靠版本号事后检测，而是用 S 备份锁直接把 DDL 挡在备份窗口外**（上文机制）。两种思路 = "阻塞保证一致" vs "并行 + 检测回补"。

> **为什么 8.0 加 Backup Lock**：FTWRL 太重（阻塞一切提交），XtraBackup 只需要"备份期间文件集不变"。Backup Lock 用 MDL 精确表达这个语义：拿 X 锁的备份工具阻塞 DDL/DML，但 InnoDB 内部后台活动不受影响。
>
> 第三件套：`SET GLOBAL read_only=1` 本身不是锁（是个变量），但它和 `super_read_only` 配合才构成完整的"实例级只读"。

### B5. 两层协作

MDL 与 InnoDB 行锁**独立但配合**：一个 DML 同时持 MDL SW（元数据层）+ InnoDB 行锁（数据层）；DDL 拿 MDL X 前等所有 SW 释放，再在引擎层检查/等待行锁。分工细节见 [`mdl.md`](transactional/mdl.md) 的 Misc。

## 五、盘点 C：容易误判为"锁"的东西

| 东西 | 是什么 | 类别 |
|------|--------|------|
| `trx->mutex`、`lock_sys->wait_mutex` | 保护事务/锁表内部数据的互斥锁 | 原语**实例**（A 类） |
| `LOCK_open`、`LOCK_plugin` 等 server 全局锁 | table cache / plugin list 的互斥锁 | 原语实例 |
| **索引树锁**（`index->lock`）/ 页锁（`buf_block_lock`） | 保护**索引结构/页**的 latch | 原语（A3）——与行锁正交：行锁锁"记录"（`lock_t`），索引树锁锁"索引这个内存结构"（`rw_lock_t`）。`innodb_row_lock_current_waits` 背后常是树锁而非行锁 |
| `read_only`/`super_read_only` | 变量，不是锁 | — |
| wait-for graph / 死锁检测 | 事务锁的**伴生机制** | 附属于 B3/B1 |

## 六、文档覆盖状态与待补清单

```
lock/
├── README.md                        ← 本篇：分类 + 类型学 + 全量盘点
├── lock_order.md                    ✅ LOCK ORDER 工具（server 层跨层）：PSI 钩子替换 + 有向图 + Tarjan SCC 判环 + 依赖文件语法（ARC/BIND/NODE）+ 官方工作流
├── primitives/                      ← 同步原语（锁原语，只放锁原语）
│   ├── innodb_sync.md               ✅ os_event（manual-reset + signal_count 防丢信号）+ sync0arr（event 已嵌入被等对象，只剩诊断/兜底/死锁检测）+ PolicyMutex 模板族（TTAS 自旋 + futex 三态两后端）+ rw_lock_t lock_word 单字三态编码 + LatchDebug/Stateful_latching_rules/LOCK ORDER 三套校验
│   ├── mysys_primitives.md          ✅ server 侧三层封装（native/my/mysql_mutex）+ PFS 埋点宏链（m_psi 无条件内嵌换 ABI）+ SAFE_MUTEX（CMake Debug 注入，与 PFS 正交）+ prlock（为 MDL 手搓的强读者优先锁）
│   └── rcu.md                       ✅ RCU 锁模式（读计数等零 + 写者等读者退出）
└── transactional/                   ← 事务锁
    ├── global_lock.md               ✅ 全局锁二件套：FTWRL（`Global_read_lock` = GLOBAL S + COMMIT S **两段式**，含三线程死锁反例）+ 备份锁（`BACKUP_LOCK` namespace，**语义 X = MDL S 的命名反转**推导）；`SET GLOBAL read_only` 为何复用 GRL；两把锁的阻塞范围对比
    ├── innodb_trx_lock.md           ✅ lock_t 一统表锁/行锁：表锁机制（意向锁/count_by_mode/AUTOINC）、行锁四形态深入剖析（源码版冲突矩阵）、隐含锁物化、等待与唤醒、死锁检测、锁与 MVCC/半一致性读边界；第四种锁语义**谓词锁**（R-tree）、**AUTOINC** 三档模式与语句级释放；核心函数完整代码剖析（加锁决策主链/冲突判定/授予/入队/挂起）
    ├── mdl.md                       ✅ MDL：**18 namespace 与两套策略**（scoped/object）、四件套关系图、**fast path 与 obtrusive/unobtrusive 的延迟精算**（`m_fast_path_state` 位布局 + 物化）、双矩阵（granted + **4 张 waiting 优先级矩阵**）、`can_grant_lock` 三级短路、`reschedule_waiters` 公平调度、wait-for graph 死锁检测（BFS+DFS、静态权重 victim、深度 32 即判死锁）、锁升级 SU→SNW→X 与 online/instant DDL、用户级锁 GET_LOCK
    └── thr_lock.md                  ✅ server 层表锁 THR_LOCK：两级结构（THR_LOCK + THR_LOCK_DATA）、13 种 `thr_lock_type` 优先级编码、**排序代替死锁检测**、写优先与防饥饿（默认关闭）、MyISAM 并发插入 4 处锁升级、InnoDB `lock_count()==0` 空壳与跨层意图协议
```

## 七、归属判据（本目录与相邻目录的边界）

- **锁/同步机制本身** → 本目录（同步原语进 `primitives/`，事务锁进 `transactional/`）
- **无锁容器/数据结构本身** → [`../infra/structure/`](../structure/)（按类型一篇一主题：`hash.md`、`list.md`、`link_buf.md` 等，不分层）——**锁原语目录只放锁原语**（2026-09-20 用户拍板的组织原则）
- **用锁实现的机制，但不是"锁"本身** → 各模块目录：Buffer Pool 的 latch 协议 → `innodb/buffer_pool.md`；`LOCK_open` 只是 table cache 的一个实现细节 → `server/table.md`
- **跨层特性里用到锁** → `feat/`（如 AUTOINC 锁），本篇从锁的视角链接过去
- **锁的实例清单**（A2/A3 的全局 mutex/latch）→ 本篇盘点 + 指向各自模块文档，不复制详细机制
