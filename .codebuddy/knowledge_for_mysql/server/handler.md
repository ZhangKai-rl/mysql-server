# 存储引擎接口：handler / handlerton / prebuilt 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 handler 与 handlerton 的分层（含 **slot 机制**、回调表、**flags**）、InnoDB 的 prebuilt 缓存（结构/所有权/生命周期），以及两个关键 handler 回调：**`external_lock`**（语句边界 + 2PC 事务注册）与 **`extra`**（杂项开关通道，含 `skip_alter_undo` 深度剖析）。两者构成"server 层向引擎传达执行意图"的两条通道：前者管语句/事务边界，后者管细粒度行为开关。
>
> DDL 如何经这两层接口落地，见 [`../innodb/ddl.md`](../innodb/ddl.md) 的「DDL 的引擎接口」章。

## 目录

- [概述](#概述)
- [handler vs handlerton](#handler-vs-handlerton)
- [prebuilt：结构、所有权、生命周期](#prebuilt结构所有权生命周期)
- [handler::external_lock：语句边界与事务注册](#handlerexternallock语句边界与事务注册)
- [handler::extra：杂项开关通道（含 skip_alter_undo 深度剖析）](#handlerextra杂项开关通道含-skip_alter_undo-深度剖析)
- [格式图谱：行 / 列 / 键的多种表示](#格式图谱行--列--键的多种表示)
- [行定位：position / ref / rnd_pos](#行定位position--ref--rnd_pos)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

SQL 层通过两层接口操作存储引擎：引擎级 `handlerton`（per-engine，全局一份）与表级 `handler`（per-table，每打开一张表一个实例）。InnoDB 的 handler 实现是 `ha_innobase`，内部用 `row_prebuilt_t`（prebuilt）缓存访问一张表所需的全部上下文以省 CPU。

### 用途

解耦 SQL 层与存储引擎：SQL 层只面向 handler/handlerton 抽象接口，引擎插件化接入。

---

## handler vs handlerton

| 维度 | handlerton | handler |
|------|-----------|---------|
| 粒度 | 引擎级（per-engine，全局唯一） | 表级（per-table，每打开一张表一个） |
| 定义 | `struct handlerton`（`sql/handler.h:2624`） | `class handler`（`sql/handler.h:4418`） |
| 承载操作 | 事务（`commit_t`/`rollback_t`/`prepare_t`/`recover_t`）、`create_t`（创建 handler）、`flush_logs_t`、`show_status_t`、`alter_tablespace_t`、DD（`dict_*`/`sdi_*`） | 单表读写：`rnd_next`/`index_read`/`ha_update_row`/`ha_delete_row`/`position`/`rnd_pos`；单表 DDL：`create`/`drop_table`/`rename_table`/`truncate` + inplace alter 四件套 |
| 存活位置 | 插件注册时创建，全局 | `TABLE::file` 指向 |

关系：`handlerton::create()` 创建 `handler` 对象。`TABLE::file` 是 `handler*`，对 InnoDB 就是 `ha_innobase`（`ha_innobase : public handler`）。**handler 通过成员 `handlerton *ht`（handler.h:4432）回指所属引擎——这是两层之间唯一的运行时连接点。**

一句话分工：**`handlerton` 回答"这个引擎能干什么、怎么提交"，`handler` 回答"这张表怎么读写、怎么改结构"**。

### handlerton：引擎的"身份证 + 回调表"

```cpp
struct handlerton {                        // handler.h:2624
  SHOW_COMP_OPTION state;                  // 引擎是否可用
  enum legacy_db_type db_type;             // 历史遗留，注释明说"正在淘汰"
  uint slot;                               // ★ THD 里的 per-engine 内存槽位
  uint savepoint_offset;                   // per-savepoint 存储区
  /* ... 几十个回调 ... */
  uint32 flags{0};                         // 全局能力标志
  const char **file_extensions;            // 文件扩展名数组
};
```

**`slot` 机制**（2636-2644 注释）是最易忽略但极关键的设计：

```cpp
/*
  Each storage engine has it's own memory area (actually a pointer)
  in the thd, for storing per-connection information.
  It is accessed as  thd->ha_data[xxx_hton.slot]
  slot number is initialized by MySQL after xxx_init() is called.
*/
uint slot;
```

这就是 `thd->get_ha_data(ht->slot)->ha_info` 的来源——每个引擎在 THD 里有一块 per-connection + per-engine 内存，**2PC 注册的 `Ha_trx_info` 就存在这里**（`ha_info + (all ? 1 : 0)`，handler.cc:1323）。

**回调表分类**（2656-2753）：

| 类别 | 回调 | 说明 |
|------|------|------|
| 事务 / XA | `commit`、`rollback`、`prepare`、`recover`、`commit_by_xid`、`rollback_by_xid`、`set_prepared_in_tc` | 内部 XA 与 2PC |
| savepoint | `savepoint_set`、`savepoint_rollback`、`savepoint_release` | 语句级回滚（external_lock 里"存 savepoint"用这些） |
| **handler 工厂** | `create` | ★ 构造 handler 实例 |
| 表空间 | `alter_tablespace`、`get_tablespace`、`upgrade_tablespace`、`is_valid_tablespace_name` | 表空间级 DDL |
| **数据字典** | `dict_init`、`dict_recover`、`dict_cache_reset`、`ddse_dict_init`、`dict_get/set_server_version` | 8.0 DD 集成（原子 DDL 前提） |
| SDI | `sdi_create`、`sdi_get`、`sdi_set`、`sdi_delete` | 序列化字典信息（存于 .ibd） |
| 其他 | `flush_logs`、`show_status`、`binlog_func`、`push_to_engine` | 运维与 secondary engine |

`dict_*` 与 `sdi_*` 这一整类是 **8.0 才有**的——对应"元数据从 MyISAM 迁到 InnoDB DD"，是原子 DDL 的前提。

### handlerton 的 flags：直接决定 DDL 行为

```cpp
#define HTON_ALTER_NOT_SUPPORTED (1 << 1)  // 引擎不支持 alter
#define HTON_CAN_RECREATE (1 << 2)         // Delete all is used for truncate
#define HTON_HIDDEN (1 << 3)               // 引擎不出现在列表中
#define HTON_SUPPORTS_ATOMIC_DDL (1 << 12) // 引擎支持原子 DDL
#define HTON_IS_SECONDARY_ENGINE (1 << 14)

inline bool ddl_is_atomic(const handlerton *hton) {   // handler.h:2957
  return (hton->flags & HTON_SUPPORTS_ATOMIC_DDL) != 0;
}
```

`ddl_is_atomic()` 在 DDL 路径里被到处使用。InnoDB 的设置（ha_innodb.cc:5173）：

```cpp
innobase_hton->flags = HTON_SUPPORTS_EXTENDED_KEYS |
                       HTON_SUPPORTS_FOREIGN_KEYS |
                       HTON_SUPPORTS_ATOMIC_DDL |      // ★ 原子 DDL
                       HTON_CAN_RECREATE |             // ★ TRUNCATE 走 recreate
                       HTON_SUPPORTS_SECONDARY_ENGINE |
                       HTON_SUPPORTS_TABLE_ENCRYPTION |
                       HTON_SUPPORTS_GENERATED_INVISIBLE_PK;
```

- **`HTON_SUPPORTS_ATOMIC_DDL`**：正是 `copy_data_between_tables` 里那个分岔判断（`to->file->ht->flags & HTON_SUPPORTS_ATOMIC_DDL`）的源头
- **`HTON_CAN_RECREATE`**：注释 "Delete all is used for truncate"——TRUNCATE 走"删光重建"的依据（rename + drop + create）

### handler：一张表的操作句柄

```cpp
class handler {                            // handler.h:4418
 protected:
  TABLE_SHARE *table_share;                // 表定义（可被多个 TABLE 共享）
  TABLE *table;                            // 当前 TABLE 实例
 public:
  handlerton *ht;                          // ★ 指回所属引擎
  uchar *ref;                              // 当前行位置（position() 产出）
  ha_statistics stats;                     // 统计（data_file_length / data_free）
  uint active_index;                       // 当前索引编号
  uint ref_length;
  enum { NONE=0, INDEX, RND, SAMPLING } inited;   // 扫描状态机
  const Item *pushed_cond;                 // 下推条件（ICP）
};
```

- `table_share`（表定义，多 TABLE 共享）与 `table`（当前实例）**handler 同时持有**
- `stats` 即 `ha_statistics`——前面算 `DATA_FREE` 时看到的 `stats->delete_length` 就从这里来（`ha_innobase::info_low` 填充）
- `inited` 是扫描状态机，`rnd_init`/`index_init` 设置它，决定后续读路径

**创建走工厂**（ha_innodb.cc:1764）：

```cpp
static handler *innobase_create_handler(handlerton *hton, TABLE_SHARE *table,
                                        bool partitioned, MEM_ROOT *mem_root) {
  if (partitioned) {
    ha_innopart *file = new (mem_root) ha_innopart(hton, table);
    if (file && file->init_partitioning(mem_root)) { destroy(file); return nullptr; }
    return (file);
  }
  // 非分区：new (mem_root) ha_innobase(hton, table)
}
```

**分区表返回 `ha_innopart`，普通表返回 `ha_innobase`**——这是 handler 层唯二的多态产物（都由 `mem_root` 分配）。

### DDL 如何落到这两层接口

**inplace alter 四件套**（handler.h:6175 起）——即「DDL 全流程」章 MDL 三段式的实现载体：

| 接口 | MDL 阶段 | 干什么 |
|------|---------|--------|
| `check_if_supported_inplace_alter`（6175） | 决策 | 返回 `enum_alter_inplace_result` → 决定算法与 MDL 级别 |
| `prepare_inplace_alter_table`（6257） | ① X 准备 | 建 new_table、分配 row log |
| `inplace_alter_table`（6294） | ② S/NONE 执行 | bulk load 重建，DML 并发 |
| `commit_inplace_alter_table`（6352） | ③ X 提交 | apply row log + rename 切换 |

**原子 DDL 的真正机制**藏在 handler DDL 接口的注释里（handler.h:6750、6419、6836）：

```cpp
/* truncate 的注释 handler.h:6750 */
@note  Changes to dd::Table object done by this method will be saved
       to data-dictionary only if storage engine supports atomic DDL
       (i.e. has HTON_SUPPORTS_ATOMIC_DDL flag set).
```

即：handler 的 DDL 方法会**修改传入的 `dd::Table` 对象**，但**只有引擎声明支持原子 DDL，这些修改才会被持久化到 DD**。原因——持久化随 DDL 事务提交，不支持原子 DDL 的引擎无法保证"DD 修改"与"引擎内部状态"一起回滚，所以 server 干脆不接受。

**`truncate` 的默认实现也印证**（handler.h:6754）：

```cpp
virtual int truncate(dd::Table *table_def) { return HA_ERR_WRONG_COMMAND; }
virtual int optimize(THD *, HA_CHECK_OPT *) { return HA_ADMIN_NOT_IMPLEMENTED; }
```

这解释了两个已知现象：OPTIMIZE 对 InnoDB 返回 `HA_ADMIN_TRY_ALTER`（引擎不实现，让 server 改走 ALTER）；TRUNCATE 走 `HTON_CAN_RECREATE` 路径。

### 调用方向的不对称：向下接口化、向上直调

理解这两层接口，关键在**两个方向的调用机制完全不同**：

| 方向 | 通道 | 机制 | 能否替换 |
|------|------|------|---------|
| **SQL → 引擎**（向下） | handler 虚函数 + handlerton 函数指针 | 接口多态（vtable + 函数指针表） | 可插件替换 |
| **引擎 → SQL**（向上） | 直接调用 SQL 层函数 | 直接 C++ 符号引用 | 不可替换（同一可执行文件） |

**向下（SQL 调引擎）是两条路径，不是只有 handlerton**：

```
SQL 层
 ├─ 表级读写  ──▶ handler 虚函数    (ha_innobase 重写的 rnd_next/index_read/write_row…)
 └─ 引擎级事务 ──▶ handlerton 函数指针 (commit/rollback/create/flush_logs…)
```

- 表级数据读写 → `handler` 虚函数（`ha_innobase` 重写的 `rnd_next`/`index_read`/`write_row`/`update_row`/`delete_row`…）
- 引擎级事务/生命周期 → `handlerton` 函数指针（`commit`/`rollback`/`prepare`/`create`/`flush_logs`…）

两者合起来才是 SE 暴露给 SQL 层的完整接口。因为都是"接口多态"，SQL 层不直接依赖 InnoDB 的具体符号，引擎才能插件化（`mysql_declare_plugin(innobase)` 注册）。**handlerton 只承载"引擎能干什么/怎么提交"，数据读写主链路走 handler 虚函数，别把两者混成"都走 handlerton"。**

**向上（引擎回调 SQL）是直接符号调用，不是 `extern "C"`**：

- InnoDB 直接调用 `my_error()`（`include/my_sys.h`，普通 `extern` 声明）、`push_warning()`/`push_warning_printf()`、`THD` 的成员方法（`dict0dd.cc` 里大量出现）
- 8.0 的 InnoDB 已是 C++（`.cc`），与 SQL 层最终链接进同一个 `mysqld`，符号直接可见，无需 C 链接
- `extern "C"` 是 InnoDB **早期纯 C 时代**调用 C++ 函数的历史遗留，现在只剩零星几处（如 `ha_innodb.cc` 的 `thd_start_time`，且实际被 `FIXME` 注释掉未真调），**并非当前机制**

一句话：**向下靠"接口"（多态解耦，可换引擎），向上靠"符号"（直接链接，同一进程）**——这才是 handler/handlerton 设计的核心。

### 是什么

`row_prebuilt_t`（`row0mysql.h:553`，注释 "save CPU time"）：缓存 InnoDB 访问一张表所需的全部上下文，避免每条语句重建。字段包括 `trx_t* trx`、`dict_index_t* index`、`dtuple_t* search_tuple`/`m_stop_tuple`、`mysql_row_templ_t* mysql_template`、`btr_pcur_t* pcur`、`ins_node_t*`/`upd_node_t*`（query graph）、锁读控制字段 `select_lock_type`/`select_mode`/`row_read_type`。

### 所有权与嵌套

`m_prebuilt` 是 `ha_innobase` 的成员（`ha_innodb.h:646`），prebuilt 是 **per-table** 的（每个 ha_innobase 实例一个）：

```
THD
 └─ TABLE（每张打开的表）
     └─ TABLE::file = handler*（对 InnoDB 是 ha_innobase 对象）
         └─ ha_innobase::m_prebuilt = row_prebuilt_t*
             ├─ trx_t* trx
             ├─ dict_index_t* index
             ├─ btr_pcur_t* pcur（持久游标）
             └─ row_read_type / select_lock_type / select_mode
```

### 生命周期

- 创建：`row_create_prebuilt`（`row0mysql.cc:805`），在 `ha_innobase::open`（`ha_innodb.cc:7397`）。
- 销毁：`row_prebuilt_free`（`row0mysql.cc:957`），在 `ha_innobase::close`（`ha_innodb.cc:7691`）。
- 防呆：`magic_n` 创建置 `ROW_PREBUILT_ALLOCATED`（78540783）、释放置 `ROW_PREBUILT_FREED`（26423527），防 use-after-free（poka-yoke 思想，同 `THD_SENTRY_MAGIC`/`MEM_BLOCK_MAGIC_N`）。

因为 `row_read_type` 存在 prebuilt 里、prebuilt 与 handler 同生命周期，所以半一致性读的 `DID` 标记能跨两次 `row_search_mvcc` 调用保留（但每次返回时会按 `did_semi_consistent_read` 重设，`row0sel.cc:6075-6080`，非恒为 DID）。

---

## handler::external_lock：语句边界与事务注册

> **归属**：`external_lock` 是 **`handler` 的虚函数**（表级，`TABLE::file` 指向的实例每打开一张表一个），**不是 `handlerton`**（引擎级全局一份）。
>
> **为何单独成章**：handler/handlerton 有上百个函数，本篇不按函数罗列。单独给 `external_lock` 一节，是因为它承载了一个**横切机制**——「语句边界 + 事务注册（2PC 参与者报到）」，与 `rnd_next`/`index_read` 等"数据访问函数"不在同一层面（那些搬数据，这个决定 InnoDB 事务如何被 server 层纳入 2PC）。

### 它不是"加锁"

名字极具误导性：**`external_lock` 不加任何锁**。它是 SQL 层通知引擎"语句要开始了、这张表要被使用了"的 hook。对照三个真正的锁就更清楚（三者**层次与锁对象完全不同**）：

| 锁 | 层次 | 锁对象 | 谁加 | 何时 |
|----|------|--------|------|------|
| **MDL** | server 层 | **表定义 / 元数据** | `mysql_alter_table` 等 server 逻辑 | 语句/DML 开始前早已获取 |
| **InnoDB 行锁** | InnoDB 层 | 记录 | DML 执行时（`row_search_mvcc` / `row_ins`） | 逐行，不是这里 |
| **InnoDB 表锁** | InnoDB 层（`lock_sys`） | **`dict_table_t`（表）** | `row_lock_table()` → `lock_table()` | **仅 `LOCK TABLES` 显式请求** |

**关键**：MySQL 平时几乎不用 InnoDB 表锁——InnoDB 是行锁引擎，靠**意向锁**（IS/IX）与行锁配合，只有显式 `LOCK TABLES` 才加真正的表锁（`lock_t` 的 `type_mode | LOCK_TABLE`）。

函数注释（`ha_innodb.cc:18623` 起）原文：MySQL 每使用一个新表开始处理 SQL 语句时都会执行 external lock，InnoDB 用它来 ①把 THD 指针存进 handle ②告知 InnoDB 新语句开始，必须存 savepoint 到事务 handle，以便出错时回滚语句。

### `ha_innobase::external_lock` 做了什么

`lock_type != F_UNLCK`（开始）分支按顺序：

| 步骤 | 作用 |
|------|------|
| `update_thd(thd)` | 把 THD 绑定进 handle |
| `sql_stat_start = true` / `reset_template()` | 标记语句开始、重置行模板 |
| quiesce switch | 处理 `FLUSH TABLE WITH READ LOCK` 的静默态 |
| `F_WRLCK → select_lock_type = LOCK_X` | 写表 → 后续读走当前读 |
| **`innobase_register_trx(ht, thd, trx)`** | 注册事务到 2PC 协调器（见下） |
| `F_RDLCK` 分支 | 按隔离级别决定 `LOCK_NONE` / `LOCK_S` |
| `row_lock_table()` | **唯一真加 InnoDB 表锁**：仅 `LOCK TABLES` + `innodb_table_locks=ON` + autocommit=0 |
| `n_mysql_tables_in_use++` / `m_mysql_has_locked = true` | 计数 |
| `++trx->will_lock` | 事务未启动且需要锁时 |
| `TrxInInnoDB::begin_stmt(trx)` | 语句开始（内含 savepoint） |

`F_UNLCK` 分支做反向：`end_stmt` → 计数减 1 → 降到 0 时若 autocommit=1 则 `innobase_commit` 提交。

补充：注释（18834 起）解释了 `row_lock_table` 为何这么保守——4.1.9 起 **AUTOCOMMIT=1 时 `LOCK TABLES` 不加 InnoDB 表锁**，因为加完马上释放纯属浪费且极易死锁，所以只在用户显式请求表锁时才加。

### 读锁类型决策

`F_RDLCK` 时按此表决定 `m_prebuilt->select_lock_type`：

```
+--------------------+----------------+-----------------+------+
|                    | < SERIALIZABLE | = SERIALIZABLE  | DD表 |
+--------------------+----------------+-----------------+------+
| 普通 SELECT        | NONE           | S               | NONE |
| SELECT FOR SHARE   | S              | S               | NONE |
| SELECT FOR UPDATE  | X              | X               | X    |
+--------------------+----------------+-----------------+------+
```

三档语义：`LOCK_NONE` = 快照读（走 read view）、`LOCK_S` = 当前读共享锁、`LOCK_X` = 当前读排他锁。这个 `select_lock_type` 一路传到 `row_search_mvcc`，决定加不加锁、加什么锁。

### `innobase_register_trx`：注册到 MySQL 的 2PC 协调器

```cpp
void innobase_register_trx(handlerton *hton, THD *thd, trx_t *trx) {   // ha_innodb.cc:3031
  const ulonglong trx_id = trx_get_id_for_print(trx);

  trans_register_ha(thd, false, hton, &trx_id);      // ① 语句级注册

  if (!trx_is_registered_for_2pc(trx) &&
      thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
    trans_register_ha(thd, true, hton, &trx_id);     // ② 事务级（2PC）注册，仅显式事务
  }

  trx_register_for_2pc(trx);                          // ③ InnoDB 内部打标记
}
```

**注册到哪里**：注册到 **THD 的事务管理器**（`thd->get_transaction()` 的 `ha_list`），即 MySQL 的**内部 XA / 2PC 协调器**。

**为什么**：内部 2PC 里 server 层（TC_LOG，开 binlog 时是 `MYSQL_BIN_LOG`）是协调者、InnoDB 是参与者。注册后，server 在 `ha_commit_trans` / `ha_rollback_trans` 时才知道要回调 InnoDB 的 `prepare` / `commit` / `rollback`——这一步就是"参与者报到"。

三个调用的分工：
- **① `all=false`（语句级）**：告诉 MySQL "InnoDB 参与了当前语句"，用于语句级回滚
- **② `all=true`（事务级/2PC）**：只有**显式事务**（`BEGIN` 或 `autocommit=0`）才做——只有显式事务需要完整 2PC 的 prepare + commit
- **③ `trx_register_for_2pc(trx)`**：InnoDB 内部标记"已注册 2PC"，保证幂等（注释明说 "Calling this several times to register the same transaction is allowed"）

**注册的用途**（权威注释在 `sql/handler.cc:1194`）：

> in order to be a part of a transaction, the engine must "register" itself. This is done by invoking `trans_register_ha()`… **Only storage engines registered for the transaction/statement will know when to commit/rollback it.**

即：**没注册的引擎，提交/回滚时根本不会被通知**。

**后续哪里用到**：`ha_commit_trans` / `ha_rollback_trans` 遍历注册链表逐个回调：

```cpp
auto ha_list = trn_ctx->ha_trx_info(trx_scope);
for (auto const &ha_info : ha_list) {
    auto ht = ha_info.ht();     // 取出注册的 handlerton
    // → 调用 ht->prepare / ht->commit / ht->rollback
}
```

（handler.cc:1892、1991、2298、2343 等多处）

**binlog 自己也注册**（binlog.cc:9421-9422）：

```cpp
if (trx) trans_register_ha(thd, true, binlog_hton, nullptr);
trans_register_ha(thd, false, binlog_hton, nullptr);
```

binlog 作为"伪引擎"注册进同一个 ha_list——这正是**内部 2PC 里 binlog 参与提交回调**的实现方式，与"binlog 作协调者（TC_LOG）"是同一件事的两面。

**副作用**（注释原文指出）：因为 `trans_register_ha` 嵌在 `external_lock` 里，**某些 DDL 会意外开启事务**（server 视角）。

> COPY DDL 里对目标表调 `ha_external_lock(F_WRLCK)` 的完整上下文见下一章。

---

## handler::extra：杂项开关通道（含 skip_alter_undo 深度剖析）

> **归属**：`extra` 是 **`handler` 的虚函数**（表级），与 `external_lock` **平级**，不是 handlerton 的。
>
> **为何单独成章**：它与 `external_lock` 共同构成"server 层把执行意图传达给引擎"的**两条通道**——`external_lock` 管**语句/事务边界**，`extra` 管**细粒度行为开关**。两者都是理解 DDL（尤其 COPY）如何在引擎侧生效的关键。

### 与 COPY DDL 的关系（`extra` 的主要用武之地）

`copy_data_between_tables`（`sql/sql_table.cc`）里对**目标表**调用：

```cpp
to->file->ha_external_lock(thd, F_WRLCK);       // 通知"要往这张表写"
alter_table_manage_keys(...);                    // 需要先 external lock
to->file->ha_start_bulk_insert(...);             // 批量插入优化
to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY);   // ★ 设 skip_alter_undo=1，插入不记 undo
while (!(error = iterator->Read())) { ... }      // 逐行读源表 → 逐行写目标表
```

**`ha_extra` 是什么**：handler 的通用**杂项开关/提示通道**（传枚举不传数据），告诉引擎"怎么做"而非"做什么"。`my_base.h:184` 的 `enum ha_extra_function` 头部注释（160-181 行）直接按引擎给枚举分类——除通用项外很多是**引擎专属**（只 MyISAM / 只 NDB / 只 MERGE / **只 InnoDB 的 `HA_EXTRA_EXPORT`** / 只分区的 `HA_EXTRA_SECONDARY_SORT_ROWID`）。这决定了它天然是"旁路"：各引擎各取所需，`default` 分支什么都不做。

**`ha_innobase::extra` 的实现**（ha_innodb.cc:18353）就是一个 switch，把枚举翻译成 prebuilt/table 上的标志：

| operation | 效果 | 落点 |
|---|---|---|
| `HA_EXTRA_FLUSH` | 释放 blob heap | `row_mysql_prebuilt_free_blob_heap` |
| `HA_EXTRA_RESET_STATE` | 重置模板 + 清 replace/ondup | `reset_template()` |
| `HA_EXTRA_KEYREAD` / `NO_KEYREAD` | 只读索引列（index-only scan） | `read_just_key` |
| `HA_EXTRA_INSERT_WITH_UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` | `on_duplicate_key_update` |
| `HA_EXTRA_WRITE_CAN_REPLACE` | REPLACE 语义 | `replace` |
| `HA_EXTRA_NO_READ_LOCKING` | 读不加锁（DD/ACL 表） | `no_read_locking` |
| **`HA_EXTRA_BEGIN_ALTER_COPY`** | **COPY 中间表：不记 undo、不加锁** | `table->skip_alter_undo = 1` |
| **`HA_EXTRA_END_ALTER_COPY`** | **拷完重算统计并复位** | `alter_stats_rebuild` + 置 0 |
| `HA_EXTRA_NO_AUTOINC_LOCKING` | 不加 autoinc 锁 | `no_autoinc_locking` |
| default | 什么都不做 | — |

**一个重要的实现约束**（源码警告注释 18363、18388）：

```cpp
/* Warning: since it is not sure that MySQL calls external_lock
before calling this function, the trx field in m_prebuilt can be
obsolete!  ... CAREFUL HERE, OR MEMORY CORRUPTION MAY OCCUR! */
```

`extra()` **可能在 `external_lock()` 之前被调用**，此时 `m_prebuilt->trx` 可能已过期。所以它只敢改简单标志、**绝不能碰 trx**——这是它设计得如此"轻"的原因。

### 深入：`skip_alter_undo` 到底免掉了什么

置 1 后在**插入路径**被消费（row0ins.cc），是**双重优化**，不只是免 undo：

```cpp
// 聚簇索引插入 row0ins.cc:3206
flags = index->table->is_temporary() ? BTR_NO_LOCKING_FLAG : 0;
if (index->table->skip_alter_undo) {
  trx_id = thr_get_trx(thr)->id;
  flags |= BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG;   // ★ 免 undo + 免锁
}

// 二级索引插入 row0ins.cc:3104（同样两个 flag）
if (index->table->skip_alter_undo) {
  flags |= BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG;
}
```

- `BTR_NO_UNDO_LOG_FLAG`：不写 undo 记录（连带省掉对应 redo）
- `BTR_NO_LOCKING_FLAG`：不做行锁检查

**为什么敢免锁**：中间表在 MDL X 保护下**只有当前 ALTER 线程能访问**，加锁纯属浪费。
**为什么敢免 undo**：ALTER 一旦失败整张中间表就 drop 掉，不需要单行回滚（`my_base.h:411`："Begin of insertion into intermediate table during copy alter operation"）。

**免 undo 的连带处理——roll_ptr**（btr0cur.cc:2787）：

```cpp
/* Roll_ptr is zero during copy alter table. So pretend to be freshly inserted row. */
if (index->table->skip_alter_undo) {
  ut_ad(roll_ptr == 0);
  roll_ptr = trx_undo_build_roll_ptr(true, 0, 0, 0);   // == (1ULL << 55)
}
```

不写 undo → roll_ptr 为 0；但行系统列需要一个"看起来像新插入"的值，于是构造 type=insert 的空 roll_ptr（`1ULL << 55`）。

**作用域有严格限制——只对 INSERT 生效**：update（row0upd.cc）、undo（row0uins.cc）、modified-undo（row0umod.cc）、undo apply（row0undo.cc）、purge（row0purge.cc）路径里**到处是 `ut_ad(!index->table->skip_alter_undo)` 断言**，走到这些路径就断言失败。

原因：COPY 中间表是**一次性构建的空表**，只会 insert；MDL X 下无人 update/delete。一旦出现 update/delete/undo/purge，说明用错场景——断言正是用来抓这个的。

**其他连带影响**：
- autoinc 保留逻辑直接跳过（ha_innodb.cc:19746）：中间表无需保留自增值间隙
- 分区表：需对每个分区设置，且 `HA_EXTRA_END_ALTER_COPY` 时对每个分区调 `alter_stats_rebuild`（ha_innopart.cc:2923）

**收益**：COPY 最大的两项单行开销（undo 写入 + 锁检查）被完全消除——这是 COPY 能实用的关键优化，部分抵消了它走 SQL 层的劣势。

必须放在拷贝前：只有 external_lock 之后 InnoDB 事务才注册好、语句才开始，后续 bulk insert 和逐行写入才能正常工作。**COPY 算法的"逐行"本质就在最后那个 `iterator->Read()` 循环里**——完全走 SQL 层，这也是它比 INPLACE 的 `ddl::Builder` bulk load 慢的根本原因。

---

## Misc

### handler vs handlerton 易混淆

- handlerton：引擎级，一个引擎一份，管事务/恢复/创建 handler。
- handler：表级，每打开一张表一个，管单表读写。
- 二者是不同层级，不是同一个东西的两种叫法。

### prebuilt 与 THD/事务的关系

prebuilt 挂在 ha_innobase（handler）下，不直接挂在 THD；但 prebuilt 内 `trx_t* trx` 指向当前事务，事务挂在 THD。所以 prebuilt 生命周期（open→close）与表句柄一致，而 trx 生命周期与语句/事务一致。

### 读路径的上下游

- **SQL 层怎么调进来**：迭代器 → `ha_rnd_next`/`ha_index_next` 见 [query/09_executor_iterator.md](query/09_executor_iterator.md)
- **InnoDB 侧怎么取行**：`row_search_mvcc`、direction、游标推进见 [innodb/row_search.md](../innodb/row_search.md)
- **DML 的两阶段读**：buffer row id → `rnd_pos` 回表见 [query/10_dml.md](query/10_dml.md)

---

## 格式图谱：行 / 列 / 键的多种表示

> MySQL ↔ InnoDB 之间有大量格式，转换就发生在 handler 这一层。本章是"格式"的完整图谱，理解它才能看懂 `position`/`key_copy`/回表等一切转换。

### 五种核心格式 + 双向转换链

```
                      (server 层)
  SQL 值 ──Field::store──▶ record[0]  ──key_copy/get_key_image──▶ MySQL key value
                              │   (row format)                        (恒2B长度头)
                              │
        INSERT 方向            │ row_insert_for_mysql
                              │  └ row_mysql_convert_row_to_innobase
                              ▼
                        dtuple_t (dfield: data+len+type)  ◀── row_sel_convert_mysql_key_to_innobase ── 查询键
                              │
                              │ rec_convert_dtuple_to_rec  [INSERT]
                              ▼
                          rec_t (页内物理记录)
                              │
        SELECT 方向            │ rec_get_offsets 解析
                              │  └ row_sel_store_mysql_rec [回表]
                              ▼
                        record[0] ──Protocol::store──▶ 客户端 text/binary
```

| 格式 | 是什么 | 关键函数 |
|---|---|---|
| **record[0]** | server 层内存行（`TABLE::record[0]`） | `row_sel_store_mysql_rec`（来）、`row_insert_for_mysql`（去） |
| **rec_t** | InnoDB 磁盘行（页内物理记录） | `rec_convert_dtuple_to_rec`（来）、`rec_get_offsets`（解析） |
| **dtuple** | InnoDB 逻辑元组（未压缩字段值） | `row_mysql_convert_row_to_innobase`、`row_sel_convert_mysql_key_to_innobase` |
| **MySQL key** | 引擎键格式（`position`/`index_read` 用） | `key_copy` / `get_key_image` |
| **addon** | filesort 排序记录 | `Field::pack`（见 runtime/01 1.3） |

### rec_t 的物理结构（格式源头）

`rec_t` 是**字节数组指针**（`rem0types.h:40`），指向字段数据起始（origin），记录头/长度列表全部在**低地址方向**。REDUNDANT 与 COMPACT/DYNAMIC/COMPRESSED 的记录头布局不同：

```
COMPACT/DYNAMIC/COMPRESSED（new-style，5 字节头）：
[变长长度列表(倒序)][NULL位图][字段数(仅instant)][5B头][字段数据...]
                       5B头 = info bits(4b)+n_owned(4b)+heap_no(13b)+status(3b)+next_offset(2B)

REDUNDANT（old-style，6 字节头）：
[偏移数组(倒序，每字段1/2B，最高位=NULL标记)][6B头][字段数据...]
                       6B头 = next_offset(2B)+short_flag(1b)+n_fields(10b)+heap_no(13b)+n_owned(4b)+info_bits(4b)
```

**系统列**（大端存储，`data0type.h:173`）：`DB_ROW_ID`（6B，仅无主键）、`DB_TRX_ID`（6B，MVCC）、`DB_ROLL_PTR`（7B，undo 指针）。追加在聚簇索引用户列之后。

**NULL 与变长长度**：REDUNDANT 没有独立 null bitmap，null 位嵌在偏移数组每项最高位；COMPACT 有独立 null bitmap（在 5B 头之后）+ 变长长度列表（倒序，1 或 2 字节）。

### dtuple vs rec_t：逻辑元组 vs 物理记录

| | dtuple（dfield） | rec_t 字段 |
|---|---|---|
| 数据 | `data` 指针 + `len`（未编码裸值） | 字节串中的一段（靠 offsets 反查） |
| NULL | `len == UNIV_SQL_NULL` | REDUNDANT 偏移最高位 / COMPACT null bitmap |
| 变长长度 | 隐式 `len` | 显式编码进长度列表 |
| 类型 | 每字段带 `dtype_t` | 不携带，靠 `dict_index_t` |

**为什么分开**：INSERT 用 dtuple（先填值再压成 rec），读取用 rec（解析成 dtuple 或直接转 record[0]），比较用 dtuple vs rec（不必先物化）。

### 同一字段的多种表示（以 VARCHAR(100) 为例）

| 表示 | 格式 | 长度头 |
|---|---|---|
| record[0] | `[长度头][数据]` | 1 或 2 字节（`field_length<256` 决定） |
| rec_t（COMPACT） | 数据在字段区，长度进变长长度列表 | 1 或 2 字节（len<128 用 1） |
| MySQL key | `[可选1B null标记][长度头][数据]` | **恒 2 字节** |
| 网络 | text: length-coded；binary: 2B 长度 + 数据 | text 变长 / binary 定 2B |
| dtuple | `data` 指针 + `len` | 无 |

> ★ 最容易混的两点：①VARCHAR 在 record 里长度头 1/2 字节，但**键里恒 2 字节**；②整数在 InnoDB 是**大端 + 符号位翻转**，在 record[0] 是**小端**——转换时逐字节倒序 + 还原符号位（`row_sel_field_store_in_mysql_format`，`row0sel.cc:2505`）。

### 两套 null 位图 + 1-based 偏移

**record[0] 与 rec_t 的 null 约定是两套**：

- **record[0]**：新 .frm 格式 `null_field_first = true`（`table.cc:1487`），null 位图在**行首 byte 0**，字段数据从 **byte 1** 开始——这就是 `Fieldstart = 1` 的来源。旧格式 null 位图在行尾。
- **rec_t**：REDUNDANT 无独立 bitmap（嵌偏移数组最高位）；COMPACT 独立 bitmap 在 5 字节头之后（origin 之前，反向排布）。

`table.cc:2963-2965` 的 `-1` 就是为此：

```cpp
record = (uchar *)outparam->record[0] - 1; /* Fieldstart = 1 */
outparam->null_flags = (uchar *)record + 1;/* record[0] 开头 */
```

净效果 `null_flags = record[0]`；那个 `-1` 是把局部变量退到"偏移 0"基准，让字段偏移（1-based）与 null 位图（byte 0）对齐。配套地，`rec_buff_length = ALIGN_SIZE(reclength + 1)`（`table.cc:1867`）多出的 `+1` 就是给 null 位图 byte 0 留的。

### 四种 row format 差异

| | REDUNDANT | COMPACT | DYNAMIC | COMPRESSED |
|---|---|---|---|---|
| 记录头 | 6B + 倒序偏移数组 | 5B + null bitmap + 变长长度列表 | 同 COMPACT | 同 COMPACT + 页压缩 |
| NULL | 偏移最高位（定长 NULL 仍占满） | 独立 bitmap | 同 COMPACT | 同 COMPACT |
| 长字段 off-page | 768B 前缀 + 20B ref | 768B 前缀 + 20B ref | **仅 20B ref（无前缀）** | 同 DYNAMIC + 压缩 |
| 索引列前缀上限 | 768B | 768B | 3072B | 3072B |

> DYNAMIC 与 COMPACT 的关键区别：长字段（>40B，`BTR_EXTERN_LOCAL_STORED_MAX_SIZE`）整段移到 off-page，记录内只留 20 字节引用，不再存 768 字节前缀。

---

## 行定位：position / ref / rnd_pos

> 本节讲 handler 接口的**行定位语义**——从"扫描出一行"到"回表定位"的 rowid 全链路。这是理解 filesort 的 indirect 模式、DML 两阶段读、index merge 回表的基础。

### ref 是什么：行位置的序列化

`ref` 是 `handler` 类的成员 `uchar *ref`（`handler.h:4434`），长度 `ref_length`（`handler.h:4506`，注释 "1-8 or the clustered key length"）。

**它存的不是行数据，而是"能重新定位到这一行的钥匙"**——不同引擎完全不同：

| 引擎 | ref 内容 | 长度 |
|---|---|---|
| InnoDB 有主键 | 主键列的**引擎键格式**（`key_copy` 产出） | `key_info->key_length` |
| InnoDB 无主键 | 6 字节隐藏 `DB_ROW_ID` | `DATA_ROW_ID_LEN`=6 |
| MyISAM | 文件偏移 `my_off_t` | `sizeof(my_off_t)` |

> ⚠ **ref 不是行数据，是从 record 派生的"定位键"**——它回答"怎么再找到这一行"，不回答"这一行长什么样"。

**ref 缓冲区的分配**（`handler.cc:2823`，`ha_open` 里）：一次 `2 * ALIGN_SIZE(ref_length)` 字节，前半给 `ref`、后半给 `dup_ref`。`ref_length` 由引擎在 open 时定（InnoDB 有主键 = `key_info->key_length`，无主键 = 6）。

### 三种格式：别混淆

> 完整图谱（rec_t / dtuple / record[0] / key / addon 五种 + 双向转换链）见上「格式图谱」章。这里只列与 ref 直接相关的三种：

rowid 链路涉及三种"键/列"格式，容易混：

| 格式 | 谁产出 | 在哪用 | 特点 |
|---|---|---|---|
| **record 列存储格式** | 引擎读行填 `record[0]` | SQL 层计算/比较 | VARCHAR 有长度前缀、BLOB 是"长度+外置指针"、CHAR 定长无前缀 |
| **引擎键格式** | `key_copy`（`sql/key.cc:136`） | `index_read`/`position` 的 ref | 可空 keypart 前 1 字节 NULL 标记、变长 2 字节长度头、定长空格填充 |
| **addon 格式** | `Field::pack` | filesort 排序记录 | 紧凑序列化，见 [runtime/01](query/runtime/01_filesort_and_temptable.md) 1.3 |

`position()` 的 `ref` 是**引擎键格式**——既不是 record 格式，也不是 InnoDB 内部格式。

### position 实现（`ha_innodb.cc:11249`）

```c
if (m_prebuilt->clust_index_was_generated) {
  /* 无主键：ref = 6 字节隐藏 row id */
  len = DATA_ROW_ID_LEN;                            // data0type.h:179，= 6
  memcpy(ref, m_prebuilt->row_id, len);
} else {
  /* 有主键：ref = 主键列的引擎键格式 */
  KEY *key_info = table->key_info + table_share->primary_key;
  key_copy(ref, (uchar *)record, key_info, key_info->key_length);  // ← 关键
  len = key_info->key_length;
}
```

**无主键的 `DB_ROW_ID` 是什么**：InnoDB 为无主键表自动生成的隐藏聚簇键，全局单调递增（`dict_sys_get_new_row_id`，`dict0boot.ic:36`），6 字节大端存储（`mach_write_to_6`）。`m_prebuilt->row_id` 是当前扫描行的 row id 缓存。

### key_copy：引擎键格式怎么来（`sql/key.cc:136`）

`position()` 里的 `key_copy` 把 record 里的主键列，转成引擎键格式。逐段：

```cpp
for (key_part = key_info->key_part; key_length > 0; key_part++) {
  if (key_part->null_bit) {                         // 可空 keypart：先写 1 字节 NULL 标记
    bool key_is_null = from_record[key_part->null_offset] & key_part->null_bit;
    *to_key++ = (key_is_null ? 1 : 0);              // 1=NULL, 0=非NULL
  }
  if (key_part->key_part_flag & (HA_BLOB_PART | HA_VAR_LENGTH_PART)) {
    // 变长 keypart：预留 2 字节长度头，再写数据
    field->get_key_image(to_key, length, Field::itRAW);
    to_key += HA_KEY_BLOB_LENGTH;                   // 跳过长度头位置（引擎回填）
  } else {
    // 定长 keypart：get_key_image 拷值，不足处空格填充
    field->get_key_image(to_key, length, Field::itRAW);
  }
  to_key += length; key_length -= length;
}
```

**引擎键格式的关键约定**：可空 keypart 前有 1 字节 NULL 标记；变长 keypart 有 2 字节长度头；定长 keypart 空格填充；各 keypart 按 KEY 定义顺序拼接，**无额外长度信息**——引擎按 `KEY_PART_INFO` 静态布局解析。

**键格式的真正产出者是 `get_key_image`**（`key_copy` 逐 keypart 调它），各类型把 record 格式转成键格式的方式不同：

```cpp
// Field_string::get_key_image（field.cc:6529）：CHAR = memcpy + 空格填充，无长度头
memcpy(buff, ptr, bytes);
if (bytes < length)
  cset->fill(buff + bytes, length - bytes, pad_char);   // 不足空格填充

// Field_varstring::get_key_image（field.cc:6825）：VARCHAR = 恒 2 字节长度头 + 数据
int2store(buff, f_length);                       // ★ 键里恒 2 字节长度头
memcpy(buff + HA_KEY_BLOB_LENGTH, pos, f_length);
```

> ★ **关键差异**：VARCHAR 在 **record 里**长度头是 1 或 2 字节（`length_bytes` 按 `field_length<256` 决定），但在**键里恒 2 字节**（注释原文 "always stored with a 2 byte prefix. Just like blob keys"）。这是"引擎键格式"与"record 格式"最实质的差异之一。

**回表时的反向解析 `row_sel_convert_mysql_key_to_innobase`**（`row0sel.cc:2260`）：`index_read` 把 MySQL 键格式转成 InnoDB 的 `search_tuple`（dtuple），逐 keypart 解析：

```cpp
if (!(dfield_type & DATA_NOT_NULL)) {           // 可空 keypart：第 1 字节 NULL 标记
  data_offset = 1;
  if (*key_ptr != 0) { dfield_set_null(dfield); is_null = true; }
}
if (DATA_LARGE_MTYPE(type)) {                   // 变长：读 2 字节小端长度
  data_len = key_ptr[data_offset] + 256 * key_ptr[data_offset + 1];
  data_offset += 2;
}
```

> **`key_copy` 不只服务 `position`**——它是"引擎键格式"的通用产出函数，`ref` 访问、`index_read`、UPDATE 构造旧键、二级索引插入等所有需要"把列值拼成引擎键"的地方都调它。这里因 `position` 用到而出现，但它属于"索引键"这个更宽的概念。

### rnd_pos 回表（`ha_innodb.cc:10863`）

```cpp
int ha_innobase::rnd_pos(uchar *buf, uchar *pos) {
  return index_read(buf, pos, ref_length, HA_READ_KEY_EXACT);  // 精确等值回表
}
```

`pos` 就是 ref（主键引擎键格式，或 6 字节 row id）。回表链路：

```
ha_rnd_pos → ha_innobase::rnd_pos → index_read(HA_READ_KEY_EXACT)
  → row_sel_convert_mysql_key_to_innobase()   // MySQL 键格式 → InnoDB search_tuple
  → row_search_mvcc()                          // B+ 树等值定位
```

无主键时，`row_sel_convert_mysql_key_to_innobase` 有专门分支（`row0sel.cc:2286`）：跳过 MySQL key 解析，直接把 6 字节当 `DB_ROW_ID` 塞进 search_tuple 第一列。

### rowid 全链路闭环

**关键澄清：`ref` 不是扫描时自动填的**——filesort 在 rowid 模式下，每读到一行后**显式调 `position()`**（`filesort.cc:987`）：

```
(1) 扫描：ha_rnd_next/ha_index_next → InnoDB row_search_mvcc 返回行 → 行落到 record[0]
(2) 填 ref：filesort.cc:987  table->file->position(record[0])   ← 显式，非自动
              ├ 有主键：key_copy → ref = 主键引擎键格式
              └ 无主键：memcpy → ref = 6 字节 DB_ROW_ID
(3) 进排序记录：filesort.cc:1554  memcpy(to, table->file->ref, ref_length)
(4) 排序：ref 作为 sort key 之后的 payload，随行移动
(5) 回表：SortBufferIndirectIterator::Read → ha_rnd_pos(record[0], ref)
(6) 定位：rnd_pos → index_read → row_search_mvcc → 原始行重新定位
```

前置：rowid 模式先 `prepare_for_position()`（把主键列加进 `read_set`，否则 `position` 拷不到主键值）。

### ⚠ 与 EXPLAIN 的 ref 区分

EXPLAIN 输出里的 `ref` 列（访问类型 `const`/`ref`/`eq_ref`）是**另一个概念**——优化器说的"用什么索引/常量匹配行"，与 handler 的 `ref` 缓冲区（行位置）**无关**，勿混淆。

### 谁用它

- **filesort 的 indirect 模式**：排序结果只存 `ref`，读结果时 `ha_rnd_pos()` 回表（见 [query/runtime/01_filesort_and_temptable.md](query/runtime/01_filesort_and_temptable.md) 1.3）
- **DML 两阶段读**：`position()` 记行 ID → `rnd_pos()` 回表改（见 [query/10_dml.md](query/10_dml.md)）
- **index merge 的 ROR**：`IndexRangeScanIterator` 按 rowid 输出，靠 `position()` 填 `file->ref`
- **WeedoutIterator**：semijoin 去重的键就是外层表的 row ID

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/handler.h` | `class handler`（表级接口）、`struct handlerton`（引擎级） |
| `sql/handler.h:1474` | `create_t`：handlerton 创建 handler |
| `sql/handler.h:4434` | `handler::ref`（行位置缓冲区） |
| `sql/handler.h:4506` | `ref_length` |
| `sql/handler.h:5541` | `position()` 纯虚接口 |
| `sql/handler.cc:2969` | `handler::ha_rnd_next`（server/InnoDB 分界面） |
| `storage/innobase/handler/ha_innodb.h:646` | `ha_innobase::m_prebuilt` |
| `storage/innobase/handler/ha_innodb.cc:7397` | `row_create_prebuilt`（open 时） |
| `storage/innobase/handler/ha_innodb.cc:7691` | `row_prebuilt_free`（close 时） |
| `storage/innobase/handler/ha_innodb.cc:11249` | `ha_innobase::position`（ref=主键/隐藏 row id） |
| `storage/innobase/handler/ha_innodb.cc:10863` | `rnd_pos`（pos=主键，index_read 回表） |
| `sql/key.cc:136` | `key_copy`（record 列 → 引擎键格式） |
| `storage/innobase/include/row0mysql.h:553` | `row_prebuilt_t` 结构 |
| `storage/innobase/row/row0mysql.cc:805` | `row_create_prebuilt` 定义 |
| `storage/innobase/row/row0mysql.cc:957` | `row_prebuilt_free` 定义 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Alternative Storage Engines*（引擎插件架构）
- *MySQL Internals Manual → The Handler Interface*

**相关文档**
- 上游（SQL 层如何调用）见 [query/09_executor_iterator.md](query/09_executor_iterator.md)
- 下游（InnoDB 如何实现）见 [innodb/row_search.md](../innodb/row_search.md)
