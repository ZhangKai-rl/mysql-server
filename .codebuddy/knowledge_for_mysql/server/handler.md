# 存储引擎接口：handler / handlerton / prebuilt 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 handler 与 handlerton 的分层（含 **slot 机制**、回调表、**flags**）、InnoDB 的 prebuilt 缓存（结构/所有权/生命周期），以及两个关键 handler 回调：**`external_lock`**（语句边界 + 2PC 事务注册，含 **`FLUSH TABLES ... FOR EXPORT` 的 quiesce 表静默分支**深度剖析）与 **`extra`**（杂项开关通道，含 `skip_alter_undo` 深度剖析）。两者构成"server 层向引擎传达执行意图"的两条通道：前者管语句/事务边界，后者管细粒度行为开关。
>
> DDL 如何经这两层接口落地，见 [`../feat/ddl/ddl.md`](../feat/ddl/ddl.md) 的「DDL 的引擎接口」章。

## 目录

- [概述](#概述)
- [handler vs handlerton](#handler-vs-handlerton)
- [row_prebuilt_t：server ↔ InnoDB 的桥接对象](#row_prebuilt_tserver--innodb-的桥接对象)
- [handler::external_lock：语句边界与事务注册](#handlerexternal_lock语句边界与事务注册)
  - [quiesce 分支：`FLUSH TABLES ... FOR EXPORT` 怎么让表静下来](#quiesce-分支flush-tables--for-export-是怎么让表静下来的)（停 purge / merge change buffer / 刷脏移出 BP / 写 .cfg 与 .cfp）
- [handler::extra：杂项开关通道（含 skip_alter_undo 深度剖析）](#handlerextra杂项开关通道含-skip_alter_undo-深度剖析)
- [格式图谱：行 / 列 / 键的多种表示](#格式图谱行--列--键的多种表示)
- [行定位：position / ref / rnd_pos](#行定位position--ref--rnd_pos)
- [Misc](#misc)
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
| 定义 | `struct handlerton`（`sql/handler.h`） | `class handler`（`sql/handler.h`） |
| 承载操作 | 事务（`commit_t`/`rollback_t`/`prepare_t`/`recover_t`）、`create_t`（创建 handler）、`flush_logs_t`、`show_status_t`、`alter_tablespace_t`、DD（`dict_*`/`sdi_*`） | 单表读写：`rnd_next`/`index_read`/`ha_update_row`/`ha_delete_row`/`position`/`rnd_pos`；单表 DDL：`create`/`drop_table`/`rename_table`/`truncate` + inplace alter 四件套 |
| 存活位置 | 插件注册时创建，全局 | `TABLE::file` 指向 |

关系：`handlerton::create()` 创建 `handler` 对象。`TABLE::file` 是 `handler*`，对 InnoDB 就是 `ha_innobase`（`ha_innobase : public handler`）。**handler 通过成员 `handlerton *ht`（handler.h）回指所属引擎——这是两层之间唯一的运行时连接点。**

一句话分工：**`handlerton` 回答"这个引擎能干什么、怎么提交"，`handler` 回答"这张表怎么读写、怎么改结构"**。

### handlerton：引擎的"身份证 + 回调表"

⚠️ **勘误**：`struct handlerton` **只有 239 行**（`sql/handler.h:2624-2862`），不是"几百行到 3500"。

```cpp
// sql/handler.h:2611
/**
  handlerton is a singleton structure - one instance per storage engine -
  to provide access to storage engine functionality that works on the
  "global" level (unlike handler class that works on a per-table basis).

  usually handlerton instance is defined statically in ha_xxx.cc as

  static handlerton { ... } xxx_hton;

  savepoint_*, prepare, recover, and *_by_xid pointers can be 0.
*/
struct handlerton {
  SHOW_COMP_OPTION state;
  enum legacy_db_type db_type;   // ★ 历史上靠这个数字在 .frm 里找引擎，8.0 已废弃
  uint slot;                     // ★ 引擎在 THD::ha_data[] 里的下标，由 ha_initialize_handlerton 分配
  uint savepoint_offset;         // ★ 静态初始化时=需要字节数；之后被改写成全局 savepoint 区内的偏移

  close_connection_t close_connection;   // ★ 只有 thd->ha_data[slot] 非 0 才被调用
  kill_connection_t kill_connection;
  pre_dd_shutdown_t pre_dd_shutdown;     // ★ 关库前停掉所有会访问 DD 的后台线程
  savepoint_set_t savepoint_set;
  savepoint_rollback_t savepoint_rollback;
  savepoint_release_t savepoint_release;

  /* ==================== 2PC 事务回调 ==================== */
  commit_t commit;               // int (*)(handlerton*, THD*, bool all)
  rollback_t rollback;           // ★ all=false 表示只回滚语句
  prepare_t prepare;             // ★ == nullptr 即视为不支持 2PC
  recover_t recover;             // 崩溃恢复时枚举处于 PREPARED 的 XA 事务
  recover_prepared_in_tc_t recover_prepared_in_tc;
  commit_by_xid_t commit_by_xid;
  rollback_by_xid_t rollback_by_xid;
  set_prepared_in_tc_t set_prepared_in_tc;

  /* ==================== DDL / 元数据（★ 只有"通知"，没有执行体） ==================== */
  create_t create;               // ★ 这是"造 handler 对象"，不是 CREATE TABLE
  drop_database_t drop_database;
  panic_t panic;
  flush_logs_t flush_logs;       // ★ binlog group commit 的 flush 阶段按引擎遍历调用
  show_status_t show_status;
  ...
  notify_alter_table_t notify_alter_table;      // ★ 通知型
  notify_rename_table_t notify_rename_table;    // ★ 通知型
  notify_truncate_table_t notify_truncate_table; // ★ 通知型

  /** Global handler flags. */
  uint32 flags{0};               // ★ HTON_* 位图，8.0 起类内初始化为 0

  binlog_func_t binlog_func;     // ★ 由 handler init 填充，非引擎静态初始化
  ...
  const char **file_extensions;
  ...
  post_ddl_t post_ddl;           // void (*)(THD*) —— atomic DDL 用
  post_recover_t post_recover;
  Clone_interface_t clone_interface;
  uint32 license;                // ★ 引擎许可证标志（社区版 vs 企业版插件）
  void *data;                    // ★ 引擎私有数据挂点
  ...
  /* ==================== 二级引擎 ==================== */
  SecondaryEngineFlags secondary_engine_flags;   // ★ 为空 = 不支持 hypergraph
  secondary_engine_modify_access_path_cost_t
      secondary_engine_modify_access_path_cost;
  ...
  Page_track_t page_track;       // ★ 最后一个成员（:2861）
};
```

> ⚠️ **不存在** `handlerton::drop_table` / `rename_table` / `truncate` / `alter_table`——8.0 只有三个**通知型**钩子 `notify_*`。
> ⚠️ **不存在** `prepare_pfs` 成员（全仓 0 命中）——PFS 事务注册走 `MYSQL_START_TRANSACTION()`，在 `trans_register_ha()` 内完成。
> ⚠️ **不存在** `commit_one_phase`（5.x 遗留注释里还提它，与代码不一致）。

**`HTON_*` flags 完整清单**（`sql/handler.h:2864-2949`）：

```c
#define HTON_NO_FLAGS 0
#define HTON_CLOSE_CURSORS_AT_COMMIT (1 << 0)
#define HTON_ALTER_NOT_SUPPORTED (1 << 1)  // Engine does not support alter
#define HTON_CAN_RECREATE (1 << 2)         // Delete all is used for truncate
#define HTON_HIDDEN (1 << 3)               // Engine does not appear in lists
/*
  Bit 4 was occupied by BDB-specific HTON_FLUSH_AFTER_RENAME flag and is no
  longer used.
*/
#define HTON_NOT_USER_SELECTABLE (1 << 5)
#define HTON_TEMPORARY_NOT_SUPPORTED (1 << 6)
#define HTON_SUPPORT_LOG_TABLES (1 << 7)
#define HTON_NO_PARTITION (1 << 8)
#define HTON_NO_BINLOG_ROW_OPT (1 << 9)    // ★ 不裁剪 read/write_set（如 P_S 虚表）
#define HTON_SUPPORTS_EXTENDED_KEYS (1 << 10)
#define HTON_SUPPORTS_FOREIGN_KEYS (1 << 11)
#define HTON_SUPPORTS_ATOMIC_DDL (1 << 12)
#define HTON_SUPPORTS_PACKED_KEYS (1 << 13)
#define HTON_IS_SECONDARY_ENGINE (1 << 14)
#define HTON_SUPPORTS_SECONDARY_ENGINE (1 << 15)
#define HTON_SUPPORTS_TABLE_ENCRYPTION (1 << 16)
constexpr const decltype(handlerton::flags) HTON_SUPPORTS_ENGINE_ATTRIBUTE{1 << 17};
constexpr const decltype(handlerton::flags) HTON_SUPPORTS_GENERATED_INVISIBLE_PK{1 << 18};
#define HTON_SECONDARY_ENGINE_SUPPORTS_DDL (1 << 19)
#define HTON_NO_TRIGGER_SUPPORT (1 << 20)
#define HTON_SUPPORTS_EXTERNAL_SOURCE (1 << 21)
```

> ★ bit 17/18 **不是 `#define` 而是 `constexpr`**（类型安全改造，类型为 `decltype(handlerton::flags)` = `uint32`）；**bit 4 废弃**（原 BDB）；bit 22-31 空闲。

**`ha_initialize_handlerton()`**（`sql/handler.cc:773`）——handlerton 是 **server malloc 出来的**，引擎的 `xxx_init()` 只负责"填字段"：

```cpp
int ha_initialize_handlerton(st_plugin_int *plugin) {
  hton = static_cast<handlerton *>(my_malloc(key_memory_handlerton_objects,
                                             sizeof(handlerton),
                                             MYF(MY_WME | MY_ZEROFILL)));
  hton->slot = HA_SLOT_UNDEF;
  plugin->data = hton;
  if (plugin->plugin->init && plugin->plugin->init(hton)) { ... goto err; }
  //  ★ 引擎的 xxx_init(hton) 在这里填 commit/prepare/rollback/flags/...

  switch (hton->state) {
    case SHOW_OPTION_YES: {
      /* 分配/复用 slot */
      for (fslot = 0; fslot < se_plugin_array.size(); fslot++)
        if (!se_plugin_array[fslot]) break;      // ★ 复用空闲 slot（支持卸载重装的循环）
      hton->slot = (fslot < se_plugin_array.size()) ? fslot : se_plugin_array.size();
      ...
      tmp = hton->savepoint_offset;              // ★ 引擎静态写的"需要多少字节"
      hton->savepoint_offset = savepoint_alloc_size;
      savepoint_alloc_size += tmp;               // ★ 改写成全局 savepoint 区内的偏移
      if (hton->prepare) total_ha_2pc++;         // ★★ 只数"实现了 prepare 的引擎"
      break;
    }
  }
  switch (hton->db_type) {
    case DB_TYPE_INNODB: innodb_hton = hton; break;   // ★ 全局单例指针在此绑定
    ...
  }
  reload_optimizer_cost_constants();
  return 0;
  ...
}
```

```cpp
struct handlerton {                        // handler.h
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

这就是 `thd->get_ha_data(ht->slot)->ha_info` 的来源——每个引擎在 THD 里有一块 per-connection + per-engine 内存，**2PC 注册的 `Ha_trx_info` 就存在这里**（`ha_info + (all ? 1 : 0)`，handler.cc）。

**回调表分类**：

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

inline bool ddl_is_atomic(const handlerton *hton) {   // handler.h
  return (hton->flags & HTON_SUPPORTS_ATOMIC_DDL) != 0;
}
```

`ddl_is_atomic()` 在 DDL 路径里被到处使用。InnoDB 的设置（ha_innodb.cc）：

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
class handler {                            // handler.h
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

**创建走工厂**（ha_innodb.cc）：

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

**inplace alter 四件套**（handler.h）——即「DDL 全流程」章 MDL 三段式的实现载体：

| 接口 | MDL 阶段 | 干什么 |
|------|---------|--------|
| `check_if_supported_inplace_alter` | 决策 | 返回 `enum_alter_inplace_result` → 决定算法与 MDL 级别 |
| `prepare_inplace_alter_table` | ① X 准备 | 建 new_table、分配 row log |
| `inplace_alter_table` | ② S/NONE 执行 | bulk load 重建，DML 并发 |
| `commit_inplace_alter_table` | ③ X 提交 | apply row log + rename 切换 |

**原子 DDL 的真正机制**藏在 handler DDL 接口的注释里（handler.h）：

```cpp
/* truncate 的注释 handler.h */
@note  Changes to dd::Table object done by this method will be saved
       to data-dictionary only if storage engine supports atomic DDL
       (i.e. has HTON_SUPPORTS_ATOMIC_DDL flag set).
```

即：handler 的 DDL 方法会**修改传入的 `dd::Table` 对象**，但**只有引擎声明支持原子 DDL，这些修改才会被持久化到 DD**。原因——持久化随 DDL 事务提交，不支持原子 DDL 的引擎无法保证"DD 修改"与"引擎内部状态"一起回滚，所以 server 干脆不接受。

**`truncate` 的默认实现也印证**（handler.h）：

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

### 是什么（`row_prebuilt_t`）

`row_prebuilt_t`（`row0mysql.h`，注释 "save CPU time"）：缓存 InnoDB 访问一张表所需的全部上下文，避免每条语句重建。字段包括 `trx_t* trx`、`dict_index_t* index`、`dtuple_t* search_tuple`/`m_stop_tuple`、`mysql_row_templ_t* mysql_template`、`btr_pcur_t* pcur`、`ins_node_t*`/`upd_node_t*`（query graph）、锁读控制字段 `select_lock_type`/`select_mode`/`row_read_type`。

### 所有权与嵌套

`m_prebuilt` 是 `ha_innobase` 的成员（`ha_innodb.h`），prebuilt 是 **per-table** 的（每个 ha_innobase 实例一个）：

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

> **完整剖析见下节「生命周期：创建、绑定与释放」（`row0mysql.cc`）**——`row_create_prebuilt`（152 行）+ `row_prebuilt_free`（73 行）+ 事务切换 + blob_heap 管理 + 时序表 + 内存归属表。

一句话版：**创建**在 `ha_innobase::open`（`ha_innodb.cc:7397` 调 `row_create_prebuilt`）、**销毁**在 `ha_innobase::close`（`:7691` 调 `row_prebuilt_free`）；双魔数 `magic_n`/`magic_n2`（创建 `ROW_PREBUILT_ALLOCATED` / 释放 `ROW_PREBUILT_FREED`）是 use-after-free 的防呆（poka-yoke 思想，同 `THD_SENTRY_MAGIC` / `MEM_BLOCK_MAGIC_N`）。

★ 一个常被忽略的推论：因为 `row_read_type`（半一致性读的 `DID` 标记）存在 prebuilt 里、而 **prebuilt 与 handler 同生命周期**，所以 `DID` 能**跨两次 `row_search_mvcc` 调用保留**（但每次返回时会按 `did_semi_consistent_read` 重设，`row0sel.cc`，非恒为 DID）。

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

函数注释（`ha_innodb.cc` 起）原文：MySQL 每使用一个新表开始处理 SQL 语句时都会执行 external lock，InnoDB 用它来 ①把 THD 指针存进 handle ②告知 InnoDB 新语句开始，必须存 savepoint 到事务 handle，以便出错时回滚语句。

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

### quiesce 分支：`FLUSH TABLES ... FOR EXPORT` 是怎么让表"静下来"的

步骤表里"quiesce switch"那一行展开讲。这个分支服务于 `FLUSH TABLES t FOR EXPORT`：让 .ibd 处于**可直接拷贝**的状态，并生成 `.cfg` 元数据文件供 `ALTER TABLE ... IMPORT TABLESPACE` 使用。

触发条件由 `external_lock` 的 switch 承接：

```cpp
switch (m_prebuilt->table->quiesce) {
  case QUIESCE_START:
    /* Check for FLUSH TABLE t WITH READ LOCK; */
    if (!srv_read_only_mode && sql_command == SQLCOM_FLUSH &&
        lock_type == F_RDLCK) {      // ★ 三个条件同时满足：非只读 + 是 FLUSH 命令 + 读锁
      if (dict_table_is_discarded(m_prebuilt->table)) {
        ib_senderrf(trx->mysql_thd, IB_LOG_LEVEL_ERROR, ER_TABLESPACE_DISCARDED,
                    table->s->table_name.str);
        return HA_ERR_NO_SUCH_TABLE;
      }

      row_quiesce_table_start(m_prebuilt->table, trx);

      /* Use the transaction instance to track UNLOCK TABLES. It can be done
         via START TRANSACTION; too implicitly. */
      ++trx->flush_tables;            // ★ 用 trx 计数跟踪"还有几张表处于静默"
    }
    break;

  case QUIESCE_COMPLETE:
    /* Check for UNLOCK TABLES; implicit or explicit or trx interruption. */
    if (trx->flush_tables > 0 &&
        (lock_type == F_UNLCK || trx_is_interrupted(trx))) {  // ★ UNLOCK TABLES 或事务被中断都算收尾
      row_quiesce_table_complete(m_prebuilt->table, trx);

      ut_a(trx->flush_tables > 0);
      --trx->flush_tables;
    }
    break;

  case QUIESCE_NONE:
    break;
}
```

真正的"静默"在 `row_quiesce_table_start`：

```cpp
void row_quiesce_table_start(dict_table_t *table, trx_t *trx) {
  ut_a(srv_n_purge_threads > 0);        // ★ 前提：purge 线程存在——下面要停它
  ut_ad(!srv_read_only_mode);
  ut_ad(fil_space_get(table->space) != nullptr);

  if (trx_purge_state() != PURGE_STATE_DISABLED) {
    trx_purge_stop();                   // ★① 停 purge —— 注意是**全局停**，不是只停这张表
  }

  for (ulint count = 0;
       ibuf_merge_space(table->space) != 0 && !trx_is_interrupted(trx);
       ++count) {                       // ★② 把该表空间的 change buffer 全部 merge 完才退出
    if (!(count % 20)) {
      ib::info(ER_IB_MSG_1017) << "Merging change buffer entries for " << table->name;
    }
  }

  if (!trx_is_interrupted(trx)) {
    if (dd_is_table_in_encrypted_tablespace(table)) {
      mutex_enter(&master_key_id_mutex); // ★③ 加密表：锁住 master_key_id，防止期间 key rotation
    }

    buf_LRU_flush_or_remove_pages(table->space, BUF_REMOVE_FLUSH_WRITE, trx);
    // ★④ 刷脏页并**从 buffer pool 移除**——移除是关键，见下

    if (dd_is_table_in_encrypted_tablespace(table)) {
      mutex_exit(&master_key_id_mutex);
    }

    if (trx_is_interrupted(trx)) {
      ib::warn(ER_IB_MSG_1018) << "Quiesce aborted!";
    } else if (row_quiesce_write_cfg(table, trx->mysql_thd) != DB_SUCCESS) {
      ib::warn(ER_IB_MSG_1019) << "There was an error writing to the meta data file";  // ★⑤ 写 .cfg
    } else if (row_quiesce_write_cfp(table, trx->mysql_thd) != DB_SUCCESS) {
      ib::warn(ER_IB_MSG_1020) << "There was an error writing to the encryption info file"; // ★⑥ 写 .cfp
    } else {
      ib::info(ER_IB_MSG_1021) << "Table " << table->name << " flushed to disk";
    }
  }

  dberr_t err = row_quiesce_set_state(table, QUIESCE_COMPLETE, trx);  // ★⑦ 状态推进，等 UNLOCK 收尾
  ut_a(err == DB_SUCCESS);
}
```

**为什么必须做这七步**（缺任何一步，拷走的 .ibd 都不自洽）：

1. **停 purge**：purge 会在后台清理 delete-marked 记录并写页，不停的话拷贝出的文件可能缺这段变更。代价是**整个实例的 purge 停摆**——history list 持续变长、undo 无法回收，直到 `UNLOCK TABLES`。这是该命令最重的副作用，也是它只适合短暂备份窗口的原因。
2. **merge change buffer**：二级索引的变更可能还缓存在 change buffer 里未落盘，必须全部 merge 进该表空间的索引页。循环到 `ibuf_merge_space` 返回 0 为止。
3. **`master_key_id_mutex`**：加密表要拿这把锁，防止写 `.cfp` 期间发生 key rotation，导致 `.cfp` 里的密钥与 `.ibd` 实际用的不一致。
4. **`buf_LRU_flush_or_remove_pages(..., BUF_REMOVE_FLUSH_WRITE, ...)`**：把该表空间的脏页刷盘**并从 buffer pool 移除**。移除是点睛之笔——之后对该表的访问都要重新从磁盘读，不会命中 buffer pool 里的旧副本；同时保证拷走的文件自洽。（对比 `BUF_REMOVE_NONE`：只刷不移除，不能用于导出。）
5. **写 `.cfg`**：表结构、列、索引的序列化元数据（`IB_EXPORT_CFG_VERSION_V7`），`IMPORT TABLESPACE` 时靠它校验并重建 dict 对象；含 instant drop 列的元数据与列默认值。
6. **写 `.cfp`**：加密表专用，存放传输用密钥供导入端解密。
7. **状态推进到 `QUIESCE_COMPLETE`**：表示"已静默"，等 `UNLOCK TABLES` 走 `row_quiesce_table_complete` 收尾——先等 `table->quiesce == QUIESCE_COMPLETE`（事务被 kill 时要等操作收尾），然后删掉 `.cfg` 与 `.cfp`（元数据只在导出窗口内有意义），最后 `trx_purge_run()` **恢复全局 purge**。

> ★ **术语易混**：这里的 **quiesce 是"表静默"（没有写入）**，与 [`../feat/ddl/ddl_engine.md`](../feat/ddl/ddl_engine.md) 讲的 **quiescence detection（并发终止检测，"没有活动在进行"）** 只是英文名撞车，机制毫无关系。源码里 grep `quiesce` 命中的绝大多数是前者。

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

**完整源码**（`storage/innobase/handler/ha_innodb.cc:3025-3045`，只有 15 行）：

```cpp
/** Registers an InnoDB transaction with the MySQL 2PC coordinator, so that
 the MySQL XA code knows to call the InnoDB prepare and commit, or rollback
 for the transaction. This MUST be called for every transaction for which
 the user may call commit or rollback. Calling this several times to register
 the same transaction is allowed, too. */
void innobase_register_trx(handlerton *hton, THD *thd, trx_t *trx) {
  const ulonglong trx_id = static_cast<ulonglong>(trx_get_id_for_print(trx));

  trans_register_ha(thd, false, hton, &trx_id);
  //  ★ false = STMT 作用域，【无条件】注册：保证语句级 commit/rollback 能通知到 InnoDB

  if (!trx_is_registered_for_2pc(trx) &&
      thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
    //  ★ 只在"真正的多语句事务"里再注册 SESSION 作用域：
    //    OPTION_NOT_AUTOCOMMIT → autocommit=0
    //    OPTION_BEGIN          → 显式 BEGIN/START TRANSACTION
    trans_register_ha(thd, true, hton, &trx_id);
  }

  trx_register_for_2pc(trx);   // ★ 置 trx->is_registered = true（InnoDB 侧本地标志）
}
```

> ⚠️ **勘误**：**不存在 `thd_register_ha()`**（全仓 0 命中）。真实函数是 `trans_register_ha()`，且它定义在 **`sql/handler.cc:1296`**，不是 `sql/transaction.cc`（后者只有一行注释提到它）。
>
> ⚠️ **trx ↔ THD 的绑定不在这里**——发生在 `innobase_trx_allocate()` 的 `trx->mysql_thd = thd;`（`ha_innodb.cc:2794`）。`register_trx` 只做"向 2PC 协调器登记"这一件事。

```cpp
// ha_innodb.cc:2887
static inline void trx_register_for_2pc(trx_t *trx) { trx->is_registered = true; }
// ha_innodb.h:1270
inline bool trx_is_registered_for_2pc(const trx_t *trx) {
  return (trx->is_registered == 1);
}
```

**`trans_register_ha()`（`sql/handler.cc:1282`）**：

```cpp
void trans_register_ha(THD *thd, bool all, handlerton *ht_arg,
                       const ulonglong *trxid [[maybe_unused]]) {
  Ha_trx_info *ha_info;
  Transaction_ctx *trn_ctx = thd->get_transaction();
  Transaction_ctx::enum_trx_scope trx_scope =
      all ? Transaction_ctx::SESSION : Transaction_ctx::STMT;

  if (all) { thd->server_status |= SERVER_STATUS_IN_TRANS; ... }

  ha_info = thd->get_ha_data(ht_arg->slot)->ha_info + (all ? 1 : 0);
  //  ★ Ha_trx_info 的存储空间在 THD 侧：每个 slot 预分配 2 个（[0]=STMT, [1]=SESSION）

  if (ha_info->is_started()) return;   // ★ 幂等：已注册直接返回
                                       //   is_started() == (m_ht != nullptr)

  trn_ctx->register_ha(trx_scope, ha_info, ht_arg);
  trn_ctx->set_ha_trx_info(trx_scope, ha_info);

  if (ht_arg->prepare == nullptr) trn_ctx->set_no_2pc(trx_scope, true);
  //  ★★ 引擎没实现 prepare → 整个事务作用域标记 no_2pc，后续跳过 tc_log->prepare()

#ifdef HAVE_PSI_TRANSACTION_INTERFACE
  if (thd->m_transaction_psi == nullptr && ht_arg->db_type != DB_TYPE_BINLOG && ...) {
    const XID *xid = trn_ctx->xid_state()->get_xid();
    bool autocommit = !thd->in_multi_stmt_transaction_mode();
    thd->m_transaction_psi = MYSQL_START_TRANSACTION(
        &thd->m_transaction_state, xid, trxid, thd->tx_isolation,
        thd->tx_read_only, autocommit);
    //  ★★ trxid（来自 innobase_register_trx 的 &trx_id）在这里落地 —— 这就是"prepare_pfs"的现代位置
  }
#endif
}
```

**`class Ha_trx_info`（`sql/transaction_info.h:393`）——注册链表节点**：

```cpp
class Ha_trx_info {
 public:
  void register_ha(Transaction_ctx::THD_TRANS *trans, handlerton *ht_arg) {
    assert(m_flags == 0); assert(m_ht == nullptr); assert(m_next == nullptr);
    m_ht = ht_arg;
    m_flags = (int)TRX_READ_ONLY;   // ★ 默认乐观假设只读
    if (trans->m_ha_list != this) {
      m_next = trans->m_ha_list;    // ★ 头插法
      trans->m_ha_list = this;
    }
  }
  void set_trx_read_write() { m_flags |= (int)TRX_READ_WRITE; }
  bool is_trx_read_write() const { return m_flags & (int)TRX_READ_WRITE; }
  bool is_started() const { return m_ht != nullptr; }   // ★ 幂等判据
  void coalesce_trx_with(const Ha_trx_info &stmt_trx) {
    if (stmt_trx.is_trx_read_write()) set_trx_read_write();   // ★ STMT→SESSION 的 rw 合并
  }
  handlerton *ht() const { return m_ht; }
 private:
  enum { TRX_READ_ONLY = 0, TRX_READ_WRITE = 1 };
  Ha_trx_info *m_next;
  handlerton *m_ht;
  uchar m_flags;
};
```

**`Transaction_ctx` 的组织（`sql/transaction_info.h:53`）**：

```cpp
  enum enum_trx_scope { STMT = 0, SESSION };
  struct THD_TRANS {
    bool m_no_2pc;            // ★ 有任一注册引擎 prepare==nullptr 就置 true
    int m_rw_ha_count;        // ★ ha_check_and_coalesce_trx_read_only() 的返回值
    Ha_trx_info *m_ha_list;   // ★ 单向链表头
   private:
    unsigned int m_unsafe_rollback_flags;
    static unsigned int const MODIFIED_NON_TRANS_TABLE = 0x01;
    static unsigned int const CREATED_TEMP_TABLE = 0x02;
    static unsigned int const DROPPED_TEMP_TABLE = 0x04;
   public:
    bool cannot_safely_rollback() const { return m_unsafe_rollback_flags > 0; }
    ...
  };
 private:
  THD_TRANS m_scope_info[2];   // ★ [0]=STMT, [1]=SESSION
```

```
THD
 └─ m_transaction : Transaction_ctx
      └─ m_scope_info[2] : THD_TRANS
           ├─ [0] STMT    { m_no_2pc, m_rw_ha_count, m_ha_list ─┐
           └─ [1] SESSION { m_no_2pc, m_rw_ha_count, m_ha_list ─┤
                                                                 │
THD::ha_data[slot].ha_info[2] : Ha_trx_info  ←──────────────────┘（节点存储于此）
      ├─ [0] → 挂进 STMT 链表
      └─ [1] → 挂进 SESSION 链表
```

**★ 1PC / 2PC 的判定点**（`sql/handler.cc:1772`）——就这一行：

```cpp
    if (ha_info->is_started())
      rw_ha_count = ha_check_and_coalesce_trx_read_only(thd, ha_info, all);
    trn_ctx->set_rw_ha_count(trx_scope, rw_ha_count);
    ...
    if (!trn_ctx->no_2pc(trx_scope) && (trn_ctx->rw_ha_count(trx_scope) > 1))
      error = tc_log->prepare(thd, all);      // ★ 2PC
    ...
    if (error || (error = tc_log->commit(thd, all))) { ... }   // ★ 1PC / 2PC 第二阶段
```

```cpp
// sql/handler.cc:1361 —— 数"真正改过数据"的引擎
static uint ha_check_and_coalesce_trx_read_only(THD *thd,
                                                Ha_trx_info_list &ha_list,
                                                bool all) {
  unsigned rw_ha_count = 0;
  for (auto const &ha_info : ha_list) {
    if (ha_info.is_trx_read_write()) ++rw_ha_count;
    if (!all) {
      Ha_trx_info *ha_info_all = &thd->get_ha_data(ha_info.ht()->slot)->ha_info[1];
      if (ha_info_all->is_started()) ha_info_all->coalesce_trx_with(ha_info);
      //  ★ STMT 的 rw 标志向上合并到 SESSION
    } else if (rw_ha_count > 1) {
      break;   // ★ 已确定要 2PC，提前跳出（所以返回值未必是精确个数）
    }
  }
  return rw_ha_count;
}
```

> ★★ 典型场景：只改 InnoDB 且**开了 binlog** → binlog 也是事务参与者（有自己的 prepare/commit），`rw_ha_count == 2` → **2PC**；**关掉 binlog** → `rw_ha_count == 1` → 纯 **1PC**。
>
> ★ `ha_prepare_low()`（`handler.cc:2294`）是真遍历引擎逐个调 `ht->prepare()` 的地方，且**只读引擎不调 prepare**（除非是 XA）。`MYSQL_BIN_LOG::prepare()`（`binlog.cc:8017`）会先设 `thd->durability_property = HA_IGNORE_DURABILITY`——prepare 阶段不刷 redo，fsync 推迟到 binlog group commit 的 flush 阶段。

```cpp
void innobase_register_trx(handlerton *hton, THD *thd, trx_t *trx) {   // ha_innodb.cc
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

**注册的用途**（权威注释在 `sql/handler.cc`）：

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

（handler.cc 等多处）

**binlog 自己也注册**（binlog.cc）：

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

**`handler::ha_extra()`（`sql/handler.cc:8490`）——它不是纯转发**：

```cpp
int handler::ha_extra(enum ha_extra_function operation) {
  if (operation == HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER) {
    // This operation should be called only for active multi-valued index
    assert(inited == INDEX &&
           (table->key_info[active_index].flags & HA_MULTI_VALUED_KEY));
    // This unique filter uses only row id to weed out duplicates. ...
    if (!m_unique &&
        (!(m_unique = new (*THR_MALLOC) Unique_on_insert(ref_length)) ||
         m_unique->init())) {
      destroy(m_unique);
      return HA_ERR_OUT_OF_MEM;
    }
    m_unique->reset(true);      // ★ 按 ref_length 建 rowid 去重器（多值索引专用）
    return 0;
  } else if (operation == HA_EXTRA_DISABLE_UNIQUE_RECORD_FILTER) {
    if (m_unique) {
      m_unique->cleanup();
      destroy(m_unique);        // ★ 与 ha_index_end 的 reset 不同：这里彻底销毁
      m_unique = nullptr;
    }
  }
  return extra(operation);      // ★ 其余全部转发给引擎虚函数
}
```

```cpp
// sql/handler.h:5598 —— 引擎默认实现：什么都不做
  virtual int extra(enum ha_extra_function operation [[maybe_unused]]) {
    return 0;
  }
```

**`enum ha_extra_function`（`include/my_base.h:184-424`）——与 DML/DDL 直接相关的关键成员**：

```c
  HA_EXTRA_NORMAL = 0,
  HA_EXTRA_QUICK = 1,           /* DELETE QUICK：B-tree 删除不做叶子合并 */
  HA_EXTRA_NO_READCHECK = 5,    /* SQL 总是先读后写，MyISAM 的 read-check 无意义 */
  HA_EXTRA_KEYREAD = 7,         /* ★ 覆盖索引：只须读索引列，不必回表 */
  HA_EXTRA_NO_KEYREAD = 8,
  HA_EXTRA_RESET_STATE,         /* 重置 handler 到 open 后的状态 */
  HA_EXTRA_IGNORE_DUP_KEY,      /* ★ INSERT IGNORE 等：不要因 dup key 停事务 */
  HA_EXTRA_NO_IGNORE_DUP_KEY,
  HA_EXTRA_PREPARE_FOR_UPDATE,  /* ★ 多表 UPDATE：动态记录表要丢掉 read cache */
  HA_EXTRA_KEYREAD_PRESERVE_FIELDS, /* ★ InnoDB 专有：keyread 时只覆盖 key 列，不整行 memcpy */
  HA_EXTRA_WRITE_CAN_REPLACE,   /* ★ REPLACE：write_row 撞冲突可直接覆盖 */
  HA_EXTRA_WRITE_CANNOT_REPLACE,
  HA_EXTRA_DELETE_CANNOT_BATCH, /* ★ 有 AFTER DELETE 触发器 → 不能批量删 */
  HA_EXTRA_UPDATE_CANNOT_BATCH, /* ★ 有 AFTER UPDATE 触发器 → 不能批量改 */
  HA_EXTRA_INSERT_WITH_UPDATE,  /* ★ INSERT ... ON DUPLICATE KEY UPDATE */
  HA_EXTRA_EXPORT,              /* FLUSH TABLES ... FOR EXPORT */
  HA_EXTRA_NO_READ_LOCKING,
  HA_EXTRA_BEGIN_ALTER_COPY,    /* ★★ ALTER TABLE COPY 算法：开始向中间表灌数据 */
  HA_EXTRA_END_ALTER_COPY,      /* ★★ 灌数据结束（与 BEGIN 配对） */
  HA_EXTRA_NO_AUTOINC_LOCKING,
  HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER,
  HA_EXTRA_DISABLE_UNIQUE_RECORD_FILTER
};

/* Compatible option, to be deleted in 6.0 */
#define HA_EXTRA_PREPARE_FOR_DELETE HA_EXTRA_PREPARE_FOR_DROP
```

> ★ 带显式 `= 数字` 的成员有空洞（`NORMAL=0, QUICK=1, NOT_USED=2, NO_READCHECK=5, ...`），从 `HA_EXTRA_FLUSH` 起改为隐式递增。
>
> ⚠️ **`HA_EXTRA_NO_*` 的真实成员**：`NO_READCHECK`(5)、`NO_KEYREAD`(8)、`NO_USER_CHANGE`(9)、`NO_WAIT_LOCK`(13)、`NO_KEYS`(16)、`NO_ROWS`、`NO_IGNORE_DUP_KEY`、`NO_IGNORE_NO_KEY`、`NO_READ_LOCKING`、`NO_AUTOINC_LOCKING`。**不存在** `HA_EXTRA_NO_WRITE_CAN_REPLACE`——对应的反面叫 `HA_EXTRA_WRITE_CANNOT_REPLACE`。

**批量接口（与 `extra` 常一起出现）**：

```cpp
// sql/handler.cc:4784
void handler::ha_start_bulk_insert(ha_rows rows) {
  assert(table_share->tmp_table != NO_TMP_TABLE || m_lock_type == F_WRLCK);
  estimation_rows_to_insert = rows;   // ★ 估算值存在 handler 上，供引擎决定批量策略
  start_bulk_insert(rows);
}
int handler::ha_end_bulk_insert() {
  estimation_rows_to_insert = 0;      // ★ 清回 0，标志批量结束
  return end_bulk_insert();
}

// sql/handler.h:6578 —— server 默认：无批量优化
  virtual void start_bulk_insert(ha_rows) {}
  virtual int end_bulk_insert() { return 0; }

// sql/handler.h:5365 —— ★ 语义反直觉：返回 true 表示"我【不用】批量"
  /** @retval false  Bulk update used by handler
      @retval true   Bulk update not used, normal operation used */
  virtual bool start_bulk_update() { return true; }
  virtual bool start_bulk_delete() { return true; }
```

> ⚠️ **不存在** `ha_start_bulk_update()` / `ha_start_bulk_delete()`（只有 insert 有 `ha_` 包装）。update/delete 的批量是直接调 virtual，且默认返回 `true` = 走普通路径。

**`position()` 与 `cmp_ref()`**——一个纯虚、一个有默认实现：

```cpp
// sql/handler.h:5541
  /*
    If HA_PRIMARY_KEY_REQUIRED_FOR_POSITION is set, then it sets ref
    (reference to the row, aka position, with the primary key given in
    the record). Otherwise it set ref to the current row.
  */
  virtual void position(const uchar *record) = 0;   // ★★ 纯虚！server 无默认实现

// :5893
  virtual int cmp_ref(const uchar *ref1, const uchar *ref2) const {
    return memcmp(ref1, ref2, ref_length);          // ★ server 唯一默认实现：定长 memcmp
  }
```

> ★ InnoDB **覆写了 `cmp_ref`**（按 PK 字段的**语义顺序**比较），因为 InnoDB 的 `ref` 是 PK 经 `key_copy()` 序列化的结果，字节序 ≠ 索引序（变长字段空格填充、数字大端编码等）。

**`key_copy()` / `key_restore()`**（`sql/key.cc:136` / `:183`）——ref 的序列化与反序列化：

```cpp
void key_copy(uchar *to_key, const uchar *from_record, const KEY *key_info,
              uint key_length) {
  if (key_length == 0) key_length = key_info->key_length;   // ★ 0 = 拷全部 key part
  for (key_part = key_info->key_part; (int)key_length > 0; key_part++) {
    // note: for key format, 如果可空每个 key part 前都有一个 null byte
    if (key_part->null_bit) {                       // ★ 可空列：先写 1 字节 NULL 标志
      bool key_is_null = from_record[key_part->null_offset] & key_part->null_bit;
      *to_key++ = (key_is_null ? 1 : 0);
      key_length--;
    }
    if (key_part->key_part_flag & HA_BLOB_PART ||
        key_part->key_part_flag & HA_VAR_LENGTH_PART) {
      key_length -= HA_KEY_BLOB_LENGTH;             // ★ 变长：留 2 字节长度前缀
      length = min<uint>(key_length, key_part->length);
      key_part->field->get_key_image(to_key, length, Field::itRAW);
      to_key += HA_KEY_BLOB_LENGTH;
    } else {
      length = min<uint>(key_length, key_part->length);
      size_t bytes = key_part->field->get_key_image(to_key, length, Field::itRAW);
      if (bytes < length)
        cs->cset->fill(cs, (char *)to_key + bytes, length - bytes, ' ');
        //                 ★ 定长列尾部补空格（PAD SPACE 语义）
    }
    to_key += length;
    key_length -= length;
  }
}
```

```cpp
void key_restore(uchar *to_record, const uchar *from_key, const KEY *key_info,
                 uint key_length) {
  for (key_part = key_info->key_part; (int)key_length > 0; key_part++) {
    if (key_part->null_bit) {                       // ★ 读回 NULL 标志写回 record 的 null bitmap
      if (*from_key++) to_record[key_part->null_offset] |= key_part->null_bit;
      else to_record[key_part->null_offset] &= ~key_part->null_bit;
      key_length--;
    }
    if (key_part->key_part_flag & HA_BLOB_PART) {
      uint blob_length = uint2korr(from_key);       // ★ 2 字节长度前缀（大端）
      field->set_ptr_offset(..., (ulong)blob_length, from_key);  // ★ BLOB 直接指向 key buffer，零拷贝
      ...
    } else if (key_part->key_part_flag & HA_VAR_LENGTH_PART) {
      field->move_field_offset(ptrdiff);            // ★ 临时把 field 指针"挪"到 to_record 上
      field->set_key_image(from_key, length);
      field->move_field_offset(-ptrdiff);           // ★ 还原
      ...
    } else {
      memcpy(to_record + key_part->offset, from_key + used_uneven_bits,
             (size_t)length - used_uneven_bits);    // ★ 定长列：直接 memcpy 到记录内偏移
    }
    ...
  }
}
```

`copy_data_between_tables`（`sql/sql_table.cc`）里对**目标表**调用：

```cpp
to->file->ha_external_lock(thd, F_WRLCK);       // 通知"要往这张表写"
alter_table_manage_keys(...);                    // 需要先 external lock
to->file->ha_start_bulk_insert(...);             // 批量插入优化
to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY);   // ★ 设 skip_alter_undo=1，插入不记 undo
while (!(error = iterator->Read())) { ... }      // 逐行读源表 → 逐行写目标表
```

**`ha_extra` 是什么**：handler 的通用**杂项开关/提示通道**（传枚举不传数据），告诉引擎"怎么做"而非"做什么"。`my_base.h` 的 `enum ha_extra_function` 头部注释（160-181 行）直接按引擎给枚举分类——除通用项外很多是**引擎专属**（只 MyISAM / 只 NDB / 只 MERGE / **只 InnoDB 的 `HA_EXTRA_EXPORT`** / 只分区的 `HA_EXTRA_SECONDARY_SORT_ROWID`）。这决定了它天然是"旁路"：各引擎各取所需，`default` 分支什么都不做。

**`ha_innobase::extra` 的实现**（ha_innodb.cc）就是一个 switch，把枚举翻译成 prebuilt/table 上的标志：

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

**它是 `dict_table_t` 上的 1-bit 标志**（`storage/innobase/include/dict0mem.h:2048`）：

```cpp
  /** true if the table is an intermediate table during copy alter
  operation or a partition/subpartition which is required for copying
  data and skip the undo log for insertion of row in the table.
  This variable will be set and unset during extra(), or during the
  process of altering partitions */
  unsigned skip_alter_undo : 1;
```

**开关就是那对 `HA_EXTRA_*`**（`ha_innodb.cc:18410-18417`）：

```cpp
    case HA_EXTRA_BEGIN_ALTER_COPY:
      m_prebuilt->table->skip_alter_undo = 1;
      break;
    case HA_EXTRA_END_ALTER_COPY:
      alter_stats_rebuild(m_prebuilt->table, m_prebuilt->table->name.m_name,
                          m_user_thd);
      m_prebuilt->table->skip_alter_undo = 0;
      break;
```

server 侧的配对调用在 `sql/sql_table.cc`：

```cpp
  to->file->ha_extra(HA_EXTRA_BEGIN_ALTER_COPY);     // :18619  灌数据前
  ...
  to->file->ha_extra(HA_EXTRA_END_ALTER_COPY);       // :18726  灌完
```

分区表走另一条路（`ha_innopart.cc:2923-2931`），按分区逐个置位：

```cpp
  if (operation == HA_EXTRA_BEGIN_ALTER_COPY ||
      operation == HA_EXTRA_END_ALTER_COPY) {
    ...
      table_part->skip_alter_undo = (operation == HA_EXTRA_BEGIN_ALTER_COPY);
  }
```

**★ 免掉的到底是什么——落点在 `btr0cur.cc:2791`**：

```cpp
  if (!(flags & BTR_KEEP_SYS_FLAG) && !index->table->is_intrinsic()) {
    /* Roll_ptr is zero during copy alter table.
    So pretend to be freshly inserted row. */
    if (index->table->skip_alter_undo) {
      ut_ad(roll_ptr == 0);
      roll_ptr = trx_undo_build_roll_ptr(true, 0, 0, 0);   // ★ 不是真跳过，是填一个"假的"
      ut_ad(roll_ptr == (1ULL << 55));
    }
    row_upd_index_entry_sys_field(entry, index, DATA_ROLL_PTR, roll_ptr);
  }
```

> ★★ **关键澄清**：`skip_alter_undo` **不是"完全不写 undo"**——它是**不分配真正的 undo 记录**，而是把每条记录的 `roll_ptr` 填成一个**哨兵值** `1ULL << 55`（`trx_undo_build_roll_ptr(true, 0, 0, 0)`：insert-flag=1、rseg/space/page/offset 全 0）。语义是"这行假装是刚插入的、没有历史版本"。
>
> ★ 为什么安全：COPY 算法的中间表是**本次 DDL 私有的**，任何并发事务都看不见它；DDL 成功则整体提交、失败则整表丢弃——都不可能回滚"单条插入"。所以历史版本（用于 MVCC 与回滚）根本不需要。

**配套的不变式断言**（遍布 row/undo/purge 路径）——它们保证"跳过 undo"的表永远不会走需要历史版本的分支：

```cpp
  storage/innobase/row/row0undo.cc:172    ut_ad(!node->table->skip_alter_undo);
  storage/innobase/row/row0umod.cc:1241   ut_ad(!node->table->skip_alter_undo);
  storage/innobase/row/row0uins.cc:343    ut_ad(!node->table->skip_alter_undo);
  storage/innobase/row/row0upd.cc:420/537/826/901  ut_ad(!index->table->skip_alter_undo);
  storage/innobase/row/row0purge.cc:707/1103       ut_ad(!node->table->skip_alter_undo);
```

> ★ 也就是说：**UPDATE / DELETE / PURGE 都断言"这张表没在 skip_alter_undo"**——因为中间表在灌数据期间只做 INSERT。这是一条很强的一致性约束。

**唯一的非 INSERT 例外：FTS**（`row0mysql.cc:1690`）——即便跳过 undo，全文索引 doc id 仍要登记：

```cpp
    if (table->skip_alter_undo) {
      if (trx->fts_trx == nullptr) trx->fts_trx = fts_trx_create(trx);
      fts_trx_table_t ftt;
      ftt.table = table;
      ftt.fts_trx = trx->fts_trx;
      fts_add_doc_from_tuple(&ftt, doc_id, node->row);
    }
```

**分区表的另外两处**（`handler0alter.cc:8064` / `:8125`）：

```cpp
    part->skip_alter_undo = true;                    // :8064  新建分区置位
    ...
        m_new_table_parts[i]->skip_alter_undo = false;   // :8125  结束后复位
```

> ★ 总结：`skip_alter_undo` 换来的收益是——COPY DDL 灌数据时**每条插入都不分配 undo record、不写 redo 的 undo 部分**，只保留聚簇/二级索引的插入。代价是记录里的 `roll_ptr` 是哨兵值，因此该表在灌数据期间**只能 INSERT，不能 UPDATE/DELETE/PURGE**（由四处断言守住）。

置 1 后在**插入路径**被消费（row0ins.cc），是**双重优化**，不只是免 undo：

```cpp
// 聚簇索引插入 row0ins.cc
flags = index->table->is_temporary() ? BTR_NO_LOCKING_FLAG : 0;
if (index->table->skip_alter_undo) {
  trx_id = thr_get_trx(thr)->id;
  flags |= BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG;   // ★ 免 undo + 免锁
}

// 二级索引插入 row0ins.cc（同样两个 flag）
if (index->table->skip_alter_undo) {
  flags |= BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG;
}
```

- `BTR_NO_UNDO_LOG_FLAG`：不写 undo 记录（连带省掉对应 redo）
- `BTR_NO_LOCKING_FLAG`：不做行锁检查

**为什么敢免锁**：中间表在 MDL X 保护下**只有当前 ALTER 线程能访问**，加锁纯属浪费。
**为什么敢免 undo**：ALTER 一旦失败整张中间表就 drop 掉，不需要单行回滚（`my_base.h`："Begin of insertion into intermediate table during copy alter operation"）。

**免 undo 的连带处理——roll_ptr**（btr0cur.cc）：

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
- autoinc 保留逻辑直接跳过（ha_innodb.cc）：中间表无需保留自增值间隙
- 分区表：需对每个分区设置，且 `HA_EXTRA_END_ALTER_COPY` 时对每个分区调 `alter_stats_rebuild`（ha_innopart.cc）

**收益**：COPY 最大的两项单行开销（undo 写入 + 锁检查）被完全消除——这是 COPY 能实用的关键优化，部分抵消了它走 SQL 层的劣势。

必须放在拷贝前：只有 external_lock 之后 InnoDB 事务才注册好、语句才开始，后续 bulk insert 和逐行写入才能正常工作。**COPY 算法的"逐行"本质就在最后那个 `iterator->Read()` 循环里**——完全走 SQL 层，这也是它比 INPLACE 的 `ddl::Builder` bulk load 慢的根本原因。

---

## row_prebuilt_t：server ↔ InnoDB 的桥接对象

> 完整定义见 `include/row0mysql.h`。它是 `ha_innobase::m_prebuilt` 的类型，**不属于 handler 类继承体系**，却是 handler 每次操作真正依赖的"工作现场"。

### 为什么需要它

每次 DML 都要做一批"昂贵但可复用"的准备工作：把 MySQL 行格式 ↔ InnoDB 行格式的**转换模板**建出来、分配持久游标、构造查询图（query graph）、缓存 range 边界。若每行都重做一遍，开销无法接受。

`row_prebuilt_t` 就是这些**跨行复用的状态集合**：建一次（`build_template`），之后所有行共用。这也是它名字里 "prebuilt" 的含义。

### 字段分组

| 组 | 字段 | 说明 |
|---|---|---|
| **句柄** | `table` / `index` / `trx` | `dict_table_t` / 当前索引 / 当前事务 |
| **★ 行格式转换** | `mysql_template`（`mysql_row_templ_t*`）、`template_type`、`n_template`、`null_bitmap_len` | 由 `ha_innobase::build_template` 创建；`template_type` 取 `ROW_MYSQL_WHOLE_ROW` / `REC_FIELDS` / `DUMMY_TEMPLATE` / `NO_TEMPLATE` |
| **读路径控制** | `read_just_key`、`need_to_access_clustered`、`hint_need_to_fetch_extra_cols`、`index_usable` | ★ `read_just_key` = **覆盖索引**（`HA_EXTRA_KEYREAD`，不必回表）；`need_to_access_clustered` = 二级索引取不到全部列、必须回表 |
| **游标与 range** | `pcur` / `clust_pcur`、`search_tuple` / `m_stop_tuple` / `m_stop_tuple_found` | `btr_pcur_t` 持久游标；`[search_tuple, m_stop_tuple]` 就是一次 range 的两个边界 |
| **执行节点** | `ins_node` / `upd_node`、`ins_graph` / `upd_graph` / `sel_graph` | InnoDB 的执行节点与查询图（`que_fork_t`） |
| **自增** | `autoinc_last_value` / `_increment` / `_offset` / `autoinc_error`、`no_autoinc_locking` | 见 [`../feat/auto_increment.md`](../feat/auto_increment.md) |
| **锁** | `select_lock_type`、`sql_stat_start` | `select_lock_type` 由 `external_lock` 设置（见上文）；`sql_stat_start` 在 `row_search_mvcc` 中被置 false |
| **DML 语义开关** | `on_duplicate_key_update`、`replace` | 对应 `HA_EXTRA_INSERT_WITH_UPDATE` / `HA_EXTRA_WRITE_CAN_REPLACE` |
| **缓冲区** | `ins_upd_rec_buff`、`default_rec`、`mv_data`、`heap` / `cursor_heap` | MySQL→InnoDB 的转换缓冲、默认值行、多值列数据 |

> **未在本表列出的字段各有归属**（不是缺口，是分流）：`old_vers_heap`（MVCC 旧版本堆）→ [`../innodb/mvcc.md`](../innodb/mvcc.md)；`m_lob_undo`（LOB 的 MVCC undo 缓存）→ [`../innodb/physical/lob.md`](../innodb/physical/lob.md)；`fts_doc_id` / `fts_doc_id_in_read_set` / `m_fts_limit` / `in_fts_query` → [`../feat/fts.md`](../feat/fts.md) 与 [`../index/inverted.md`](../index/inverted.md)；`srch_key_val1/2` / `srch_key_val_len`（`rows_in_range()` 的 **INT 键转换缓冲**，因 MySQL 小端↔InnoDB 大端而需要）→ 上面「生命周期」节；`fetch_cache[]` / `n_fetch_cached` / `fetch_cache_first` / `fetch_direction` / `m_no_prefetch` → 下节「fetch cache」；`innodb_api_buf` / `innodb_api_rec` / `innodb_api_rec_size`（**仅 InnoDB API 路径**用，`cursor_heap` 同此）→ 本文件不涉及。其余为纯状态位（`no_read_locking`、`m_read_virtual_key`、`ins_sel_stmt`、`new_rec_lock` 等），语义即字段名。

### 两个值得单说的字段

**① `clust_index_was_generated`**——用户没定义主键、InnoDB 自动生成 row_id 聚簇索引时置 true。它与 `byte row_id[DATA_ROW_ID_LEN]`（最后取到的行的 row_id）配套。★ 这正是 [`../innodb/physical/record.md`](../innodb/physical/record.md) 中「隐藏系统列与 DB_ROW_ID」在 handler 侧的对应物：那篇讲 row_id 怎么生成与持久化，这里讲它怎么被缓存与回传。

**② `m_is_reading_range` 用了 RAII guard**：

```cpp
class row_is_reading_range_guard_t : private ut::bool_scope_guard_t {
 public:
  explicit row_is_reading_range_guard_t(row_prebuilt_t &prebuilt)
      : ut::bool_scope_guard_t(prebuilt.m_is_reading_range) {}
};
row_is_reading_range_guard_t get_is_reading_range_guard() {
  ut_ad(!m_is_reading_range);        // 断言作用域不嵌套
  return row_is_reading_range_guard_t(*this);
}
```

即"是否处于 `read_range_first/next` 中"这个状态由作用域自动管理，且显式断言不嵌套——比手工 bool 置位/复位更不容易漏。

### mysql_row_templ_t：转换模板的每一项（`row0mysql.h`）

⚠️ **勘误**：`mysql_row_templ_t` 在 **`row0mysql.h:491-540`**（不是 1800-2000）；`row_prebuilt_t` 在 **`:553-979`**。

```cpp
// storage/innobase/include/row0mysql.h:451
/* A struct describing a place for an individual column in the MySQL
row format which is presented to the table handler in ha_innobase.
This template struct is used to speed up row transformations between
Innobase and MySQL. */

struct mysql_row_templ_t {
  ulint col_no;                 /*!< column number of the column */
  ulint rec_field_no;           /*!< field number of the column in an
                                Innobase record in the current index;
                                not defined if template_type is
                                ROW_MYSQL_WHOLE_ROW */
  ulint clust_rec_field_no;     /*!< field number of the column in an
                                Innobase record in the clustered index */
  ulint icp_rec_field_no;       /*!< field number of the column in an
                                Innobase record in the current index;
                                only defined for columns that could be
                                used to evaluate a pushed down index
                                condition and/or end-range condition */
  ulint mysql_col_offset;       /*!< offset of the column in the MySQL row format */
  ulint mysql_col_len;          /*!< length of the column in the MySQL row format */
  ulint mysql_mvidx_len;        /*!< index length on multi-value array */
  ulint mysql_null_byte_offset; /*!< MySQL NULL bit byte offset in a MySQL record */
  ulint mysql_null_bit_mask;    /*!< bit mask to get the NULL bit,
                                zero if column cannot be NULL */
  ulint type;                   /*!< column type in Innobase mtype numbers DATA_CHAR... */
  ulint mysql_type;             /*!< MySQL type code; this is always < 256 */
  ulint mysql_length_bytes;     /*!< if mysql_type == DATA_MYSQL_TRUE_VARCHAR,
                                whether to use 1 or 2 bytes to store the
                                data length at the start of row */
  ulint charset;                /*!< MySQL charset-collation code of the column */
  ulint mbminlen;               /*!< minimum length of a char, in bytes */
  ulint mbmaxlen;               /*!< maximum length of a char, in bytes */
  ulint is_unsigned;            /*!< integer type and != 0 → unsigned */
  ulint is_virtual;             /*!< if a column is a virtual column */
  ulint is_multi_val;           /*!< if a column is a Multi-Value Array virtual column */
};
```

> ★ 全部是 `ulint`。注意 `mysql_null_byte_offset` **只在 `field->is_nullable()` 分支里赋值**（`build_template_field` :8289），非可空列它是未初始化的残留值，靠 `mysql_null_bit_mask == 0` 判断"不会为 NULL"。

**模板类型是 `constexpr uint32_t`，不是 enum**（`row0mysql.h:1043`）：

```cpp
constexpr uint32_t ROW_MYSQL_WHOLE_ROW = 0;
constexpr uint32_t ROW_MYSQL_REC_FIELDS = 1;
constexpr uint32_t ROW_MYSQL_NO_TEMPLATE = 2;
/** dummy template used in row_scan_and_check_index */
constexpr uint32_t ROW_MYSQL_DUMMY_TEMPLATE = 3;

/* Values for hint_need_to_fetch_extra_cols */
constexpr uint32_t ROW_RETRIEVE_PRIMARY_KEY = 1;
constexpr uint32_t ROW_RETRIEVE_ALL_COLS = 2;

/* Values for row_read_type */
constexpr uint32_t ROW_READ_WITH_LOCKS = 0;
constexpr uint32_t ROW_READ_TRY_SEMI_CONSISTENT = 1;
constexpr uint32_t ROW_READ_DID_SEMI_CONSISTENT = 2;
```

> ⚠️ **`ROW_MYSQL_DYNAMIC` / `ROW_MYSQL_COMMON` / `ROW_MYSQL_CLUST` 全都不存在**（5.1/5.5 旧枚举名，8.0 全仓 0 命中）。`template_type` 是 **2-bit 位域**，正好容纳 0~3。

**`build_template_field()`：每一项怎么填**（`ha_innodb.cc:8239`，核心段）：

```cpp
static mysql_row_templ_t *build_template_field(
    row_prebuilt_t *prebuilt, dict_index_t *clust_index, dict_index_t *index,
    TABLE *table, const Field *field, ulint i, ulint v_no) {
  templ = prebuilt->mysql_template + prebuilt->n_template++;   // ★ 连续数组，不是链表

  if (innobase_is_v_fld(field)) { templ->is_virtual = true; col = &dict_table_get_nth_v_col(index->table, v_no)->m_col; }
  else { templ->is_virtual = false; col = index->table->get_col(i); }

  if (!templ->is_virtual) {
    templ->col_no = i;
    templ->clust_rec_field_no = dict_col_get_clust_pos(col, clust_index);
    if (index->is_clustered()) templ->rec_field_no = templ->clust_rec_field_no;
    else                       templ->rec_field_no = index->get_col_pos(i);
  } else { ... }
  templ->icp_rec_field_no = ULINT_UNDEFINED;   /* Set in set_templ_icp(). */

  if (field->is_nullable()) {
    templ->mysql_null_byte_offset = field->null_offset();
    templ->mysql_null_bit_mask = (ulint)field->null_bit;
  } else {
    templ->mysql_null_bit_mask = 0;            // ★ 非可空：mask 为 0 即"永不为 NULL"
  }

  templ->mysql_col_offset = (ulint)get_field_offset(table, field);
  templ->mysql_col_len = (ulint)field->pack_length();
  templ->type = col->mtype;
  templ->mysql_type = (ulint)field->type();
  if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR)
    templ->mysql_length_bytes = field->get_length_bytes();
  templ->charset = dtype_get_charset_coll(col->prtype);
  templ->mbminlen = col->get_mbminlen();
  templ->mbmaxlen = col->get_mbmaxlen();
  templ->is_unsigned = col->prtype & DATA_UNSIGNED;

  if (!index->is_clustered() && templ->rec_field_no == ULINT_UNDEFINED)
    prebuilt->need_to_access_clustered = true;    // ★ 二级索引里没这列 → 要回表
  if (dict_index_is_spatial(index))
    prebuilt->need_to_access_clustered = true;    // ★ 空间索引也要回表

  if (prebuilt->mysql_prefix_len < templ->mysql_col_offset + templ->mysql_col_len)
    prebuilt->mysql_prefix_len = templ->mysql_col_offset + templ->mysql_col_len;
    //  ★★ 要拷贝的 MySQL 行前缀长度 —— row_search_mvcc 里最小化 memcpy 的关键

  if (DATA_LARGE_MTYPE(templ->type)) prebuilt->templ_contains_blob = true;
  if (templ->type == DATA_POINT)     prebuilt->templ_contains_fixed_point = true;
  return (templ);
}
```

**ICP 专属判定完全不看 read_set**（`ha_innodb.cc:8221`）：

```cpp
inline bool build_template_needs_field_in_icp(
    const dict_index_t *index, const row_prebuilt_t *prebuilt,
    bool contains, ulint i, bool is_virtual) {
  ut_ad(contains == dict_index_contains_col_or_prefix(index, i, is_virtual));
  return (index == prebuilt->index ? contains
                                   : dict_index_contains_col_or_prefix(
                                         prebuilt->index, i, is_virtual));
}
```

> ★ ICP 只看该列是否被**正在扫描的那个索引**（`prebuilt->index`）包含——因为下推条件始终相对 `prebuilt->index`，而 `index` 参数可能是聚簇索引（whole_row 场景）。

**`row_prebuilt_t` 里与模板相关的位域与字段**（`row0mysql.h:599-640`）：

```cpp
  unsigned template_type : 2;              /* ROW_MYSQL_WHOLE_ROW / REC_FIELDS /
                                             DUMMY_TEMPLATE / NO_TEMPLATE */
  unsigned n_template : 10;                /* 模板项数（位域上限 1023） */
  unsigned null_bitmap_len : 10;           /* MySQL 行首 NULL 位图的字节数 */
  unsigned need_to_access_clustered : 1;   /* ★ 二级索引回表标志 */
  unsigned templ_contains_blob : 1;
  unsigned templ_contains_fixed_point : 1;
  mysql_row_templ_t *mysql_template;       /* ★ 数组，容量 = table->s->fields */
  ulint mysql_prefix_len;                  /* ★ 要拷贝的行前缀长度 */
  ulint n_rows_fetched;
  byte *fetch_cache[MYSQL_FETCH_CACHE_SIZE];   /* ★ = 8 */
```

> ⚠️ 常见误称更正：成员名是 **`row_read_type`**（不是 `row_lock_type`）、**`mysql_row_len`**（不是 `row_len`）、**`fetch_cache[]`**（不存在 `fetch_size`）；**不存在** `n_rows_updated`（那是全局 `srv_stats` 的成员，`include/srv0srv.h:136`）。
>
> ⚠️ **`clust_templ_for_sec` 不是 `row_prebuilt_t` 的成员**——8.0 已删除该字段，它现在是 `row0sel.cc` 里的局部 bool / 形参（`:2515`、`:2746`、`:4456` 等）。

`row_prebuilt_t::mysql_template` 是 `mysql_row_templ_t` 数组，**每列一项**，由 `ha_innobase::build_template` 填充。它存在的意义是：把"查字典才能知道的布局信息"预先算成数字，转换时直接 `memcpy`。

```cpp
struct mysql_row_templ_t {
  ulint col_no;                   // 列号
  ulint rec_field_no;             // 当前索引记录中的字段号
  ulint clust_rec_field_no;       // ★ 聚簇索引记录中的字段号（回表时用）
  ulint icp_rec_field_no;         // ★ ICP / end-range 用的字段号（只为可能下推的列定义）
  ulint mysql_col_offset;         // MySQL 行格式中的偏移
  ulint mysql_col_len;            // MySQL 行格式中的长度
  ulint mysql_mvidx_len;          // 多值数组索引长度
  ulint mysql_null_byte_offset;   // MySQL NULL 位图字节偏移
  ulint mysql_null_bit_mask;      // NULL 位掩码（0 = 该列不可能为 NULL）
  ulint type;                     // InnoDB mtype（DATA_CHAR...）
  ulint mysql_type;               // MySQL 类型码（恒 < 256）
  ulint mysql_length_bytes;       // true VARCHAR 用 1 还是 2 字节存长度
  ulint charset, mbminlen, mbmaxlen;
  ulint is_unsigned, is_virtual, is_multi_val;
};
```

★ **这个结构的本质是"三重坐标映射"**——同列在三种表示里的位置都预先记下：

| 坐标 | 字段 | 用在哪 |
|---|---|---|
| **MySQL 行格式** | `mysql_col_offset` / `mysql_col_len` / `mysql_null_byte_offset` / `mysql_null_bit_mask` | 读写 `TABLE->record[0]` |
| **当前索引记录** | `rec_field_no` | 从当前扫描的索引记录取列 |
| **聚簇索引记录** | `clust_rec_field_no` | 回表后从聚簇记录取列 |

转换一行时不需要任何字典查找，纯数字索引 + `memcpy`——**这就是 "prebuilt 省 CPU" 的全部秘密**。

两个易忽略的细节：

- `icp_rec_field_no` **只为"可能参与 ICP 下推或 end-range 判断"的列定义**，其余列不填——避免为用不到的列付代价；
- `mysql_length_bytes` 记录 MySQL 行格式里 true VARCHAR 的长度前缀是 1 还是 2 字节；**注意注释强调：key value（键值）格式恒用 2 字节**，与行内表示不同。

#### template_type：四种模板各用于什么

```cpp
constexpr uint32_t ROW_MYSQL_NO_TEMPLATE = 2;
/* dummy template used in row_scan_and_check_index */
constexpr uint32_t ROW_MYSQL_DUMMY_TEMPLATE = 3;
```

| 类型 | 含义 | 谁在用 |
|---|---|---|
| `ROW_MYSQL_WHOLE_ROW` | 模板含**全部列** | `build_template(true)`：写路径（`write_row` / `update_row`）、并行扫描、ALTER 重建、采样 |
| `ROW_MYSQL_REC_FIELDS` | 模板只含**本次需要的列** | `build_template(false)`：读路径（`index_read` / `rnd_next` / `general_fetch`） |
| `ROW_MYSQL_DUMMY_TEMPLATE` | **空模板**（`n_template = 0`），只校验不取行 | `CHECK TABLE`、`row_scan_and_check_index`（`row0sel.cc` 注释 "CHECK TABLE: fetch the row"） |
| `ROW_MYSQL_NO_TEMPLATE` | 不需要模板 | HANDLER 等场景 |

★ **`build_template(true/false)` 这个参数，就是 `read_set` 优化在 InnoDB 侧的落地**：读路径只把 `read_set` 里的列建进 `mysql_template` 数组（`n_template` 更小），取行时直接跳过无关列。这是 `SELECT a FROM t` 比 `SELECT *` 快的深层原因之一——不只是少了 IO，**每行还少做了 N-1 次格式转换**。

#### 构建、复用与失效

- `build_template(bool whole_row)`（`ha_innodb.cc:8377`）填充 `mysql_template` 数组并设 `n_template`；`reset_template()`（`:3166-3190`）**不释放数组、不重置 `n_template`**，只清复用前提位（`read_just_key`、`keep_other_fields_on_keyread`、`m_end_range`），并在 `idx_cond` 为真时把 `template_type` 置 `ROW_MYSQL_NO_TEMPLATE`（`:3188`）迫使下一次重建；
- ★ **复用判定的真实位置**（一处常见错误：以为在 `row0mysql.cc`）：`can_reuse_mysql_template()` 是 `ha_innobase` 的**虚函数**——基类默认直接 `return false`（`ha_innodb.h:640`，普通表永不复用）；唯一的 override 在分区表 `ha_innopart`（`ha_innopart.h:1164`，返回状态位 `m_reuse_mysql_template`）。该状态位只在"**同一 statement 内切换到另一个分区**"时被置 true（`ha_innopart.cc:1350-1352`，条件是 `sql_stat_start == 0`），在初始化/`index_init` 等入口一律置 false（`:1138`、`:3972`、`:4096`）。复用的前提假设是"各分区表定义/字段布局完全一致"；
- ★ debug 下的"**重建前后逐字节比对**"防线（`ha_innodb.cc:10232-10242`，位于 `index_read` 内、`:10227` 的常规重建之后）：只在"允许复用"时启用——备份 `n_template` 与旧指针、把 `mysql_template` **置 nullptr 强制重分配**、`build_template(false)` 重建一遍，然后断言 ① `n_template` 相等 ② **`memcmp` 整块逐字节相等**（`:10238-10240`），最后 `ut::free` 新缓冲、恢复旧指针。它把"模板该失效却被复用"的隐蔽数据错乱挡在开发/测试阶段；之所以可行，靠 `:8443` 的 memset 消除 padding 干扰 + `:8436` 的"指针为空才分配"。该块被 `!defined(UNIV_DEBUG_VALGRIND)` 排除（Valgrind 对未初始化字节告警）。

#### build_template：主函数逐行（`ha_innodb.cc:8377-8675`）

**① 开头：`whole_row` 的四种来源 + 三个布尔开关**

```cpp
void ha_innobase::build_template(bool whole_row) {           // :8377 值传递，函数内可改写
  dict_index_t *index;           // :8378 本次扫描用哪个索引
  dict_index_t *clust_index;     // :8379 聚簇索引
  ulint n_fields;                // :8380 表字段数
  bool fetch_all_in_key = false;        // :8381 只取索引内所有列
  bool fetch_primary_key_cols = false;  // :8382 至少取主键列
  ulint i;                       // :8383

  if (m_prebuilt->select_lock_type == LOCK_X) {              // :8385
    /* We always retrieve the whole clustered index record if we
    use exclusive row level locks, for example, if the read is
    done in an UPDATE statement. */
    whole_row = true;                                        // :8390 ★来源一：X 锁
  } else if (!whole_row) {                                   // :8391
    if (m_prebuilt->hint_need_to_fetch_extra_cols == ROW_RETRIEVE_ALL_COLS) {  // :8392
      if (m_prebuilt->read_just_key) {                       // :8396
        /* MySQL has instructed us that it is enough to fetch the
        columns in the key; looks like MySQL can set this flag also
        when there is only a prefix of the column in the key: in
        that case we retrieve the whole column from the clustered index */
        fetch_all_in_key = true;                             // :8404 ★来源二：key-read 分支
      } else {
        whole_row = true;                                    // :8406 ★来源三：需额外列却非 key-read
      }
    } else if (m_prebuilt->hint_need_to_fetch_extra_cols ==
               ROW_RETRIEVE_PRIMARY_KEY) {                   // :8408-8409
      /* We must at least fetch all primary key cols. Note that if the
      clustered index was internally generated by InnoDB on the row id
      (no primary key was defined), then row_search_for_mysql() will
      always retrieve the row id to a special buffer in the
      m_prebuilt struct. */
      fetch_primary_key_cols = true;                         // :8417 ★来源四
    }
  }
```

★ 决策树是 **X 锁 > 调用方 `whole_row` > `hint_need_to_fetch_extra_cols`**。★ 最值得记住的是"来源一"：只要拿**排他行锁**（`UPDATE` 里的读、`SELECT ... FOR UPDATE`），就**无条件升级为取整行**——这是"锁定读比普通读慢"的深层原因之一：不是锁慢，而是**锁定读无法享受"只取需要的列"这一优化**。★ `ROW_RETRIEVE_ALL_COLS` **不等于** `whole_row`：只有在**没有** `read_just_key` 时才升级（`:8405-8407`）；有 `read_just_key` 则改为 `fetch_all_in_key`（只取键内列，但键内**所有**列都要）。★ `read_just_key` 由 SQL 层 `HA_EXTRA_KEYREAD` 设定、`reset_template()` 清零（`:3178`）——**本函数只读不写**它。

**② 索引选定 + 模板空间分配 + 状态复位（`:8421-8458`）**

```cpp
  clust_index = m_prebuilt->table->first_index();             // :8421 聚簇索引（InnoDB 第一索引）
  index = whole_row ? clust_index : m_prebuilt->index;        // :8423 整行→聚簇；否则→当前扫描索引
  m_prebuilt->need_to_access_clustered = (index == clust_index);  // :8425 初值

  /* Either m_prebuilt->index should be a secondary index, or
  it should be the clustered index. */
  ut_ad(index->is_clustered() == (index == clust_index));     // :8429 二级/聚簇二选一

  n_fields = (ulint)table->s->fields; /* number of columns */  // :8434

  if (!m_prebuilt->mysql_template) {                          // :8436 ★只在首次分配（随 prebuilt 复用）
    m_prebuilt->mysql_template = (mysql_row_templ_t *)ut::malloc_withkey(
        UT_NEW_THIS_FILE_PSI_KEY, n_fields * sizeof(mysql_row_templ_t));   // :8437-8438
  }

#if defined(UNIV_DEBUG) && !defined(UNIV_DEBUG_VALGRIND)
  /* zero-filling for compare contents for debug */
  memset(m_prebuilt->mysql_template, 0, n_fields * sizeof(mysql_row_templ_t));  // :8443 为复用比对清 padding
#endif

  m_prebuilt->template_type =
      whole_row ? ROW_MYSQL_WHOLE_ROW : ROW_MYSQL_REC_FIELDS;  // :8446-8447
  m_prebuilt->null_bitmap_len = table->s->null_bytes;         // :8448

  /* Prepare to build m_prebuilt->mysql_template[]. */
  m_prebuilt->templ_contains_blob = false;                    // :8451
  m_prebuilt->templ_contains_fixed_point = false;             // :8452
  m_prebuilt->mysql_prefix_len = 0;                           // :8453
  m_prebuilt->n_template = 0;                                 // :8454 ★计数从 0 重启（覆写而非追加）
  m_prebuilt->idx_cond_n_cols = 0;                            // :8455
```

模板数组按**全列数**一次性分配、在 `row_prebuilt_t` 生命周期内**复用**（`:8436` 的 `if (!mysql_template)`）——所以三个建模板循环是**覆写**，`n_template` 每次从 0 重启，随后由 `build_template_field` 内部 `n_template++` 自增（见下节）。★ `need_to_access_clustered` 在此只赋初值，之后**只在 `build_template_field` 内**被置 true，共两处（`:8320` 二级索引取不到该列、`:8325` 空间索引）。★ `:8443` 那个 debug memset 不是为性能，而是为 `:10239` 的 `memcmp` 比对**消除 padding 干扰**（否则未初始化的 padding 字节会让比对假阳性）。

**③ 三个建模板循环：骨架**

整函数的分流**不是**按"索引是不是聚簇"，而是按**有没有 ICP 下推**——进入判据只有一个：`active_index != MAX_KEY && active_index == pushed_idx_cond_keyno`（`:8460`）：

| 循环 | 位置 | 进入条件 | 干什么 |
|---|---|---|---|
| **①** | `:8464` | ICP 分支内 | 收集**参与 ICP / end_range 的列**（`build_template_needs_field_in_icp` 为真） |
| **②** | `:8558` | 同一分支内 | 补齐**不需要 ICP 的列**（同一判据取反） |
| **③** | `:8606` | `else`（无 ICP） | 一视同仁按需建模板（唯一能降级 whole_row 的地方） |
| **④** | `:8669` | 收尾（与否 ICP 无关） | 需要回表时把 `rec_field_no` 整体改指聚簇记录 |

★ 循环①②**必须**把 ICP 列连续压到模板前段——`:8516`/`:8552` 断言 `idx_cond_n_cols == n_template`，因为下游 `row_search_idx_cond_check` 只扫前 `idx_cond_n_cols` 项（省去遍历全模板）。★ 循环①与②的唯一差别是 `build_template_field` 的第 7 实参 `v_no`：①传死值 `0`（ICP 项不允许虚拟列，`:8513` 的 `ut_ad(!templ->is_virtual)` 兜底），②传 `num_v`。两者在"跳过虚拟列"时都会推进局部计数器 `num_v`（`:8504`、`:8584`），保证 `i - num_v` 始终是**物理列号**。

**循环①（ICP 列）完整代码（`:8464-8554`）**

```cpp
  if (active_index != MAX_KEY && active_index == pushed_idx_cond_keyno) {  // :8460
    ulint num_v = 0;                                         // :8461 局部虚拟列计数器
    /* Push down an index condition or an end_range check. */
    for (i = 0; i < n_fields; i++) {                         // :8464 【循环①】
      bool index_contains;
      if (innobase_is_v_fld(table->field[i])) {              // :8467
        index_contains = dict_index_contains_col_or_prefix(index, num_v, true);   // :8468
      } else {
        index_contains = dict_index_contains_col_or_prefix(index, i - num_v, false);  // :8470-8471
      }

      /* When join_read_always_key() invokes this code via
      handler::ha_index_init() and ha_innobase::index_init(), end_range is
      not yet initialized. Because of that, we must always check for
      index_contains, instead of the subset field->part_of_key.is_set(...)
      which would be acceptable if end_range == NULL. */
      bool is_v = innobase_is_v_fld(table->field[i]);        // :8489
      if (build_template_needs_field_in_icp(index, m_prebuilt, index_contains,
                                            is_v ? num_v : i - num_v, is_v)) {  // :8490-8491
        /* Needed in ICP */
        const Field *field;
        mysql_row_templ_t *templ;
        if (whole_row) {                                     // :8496
          field = table->field[i];                           // :8497
        } else {
          field = build_template_needs_field(
              index_contains, m_prebuilt->read_just_key, fetch_all_in_key,
              fetch_primary_key_cols, index, table, i, num_v);       // :8499-8501
          if (!field) {                                      // :8502
            if (innobase_is_v_fld(table->field[i])) num_v++; // :8503-8504
            continue;                                        // :8506
          }
        }
        templ = build_template_field(m_prebuilt, clust_index, index, table,
                                     field, i - num_v, 0);   // :8510-8511 ★v_no 传 0
        ut_ad(!templ->is_virtual);                           // :8513 ICP 项必为普通列
        m_prebuilt->idx_cond_n_cols++;                       // :8515
        ut_ad(m_prebuilt->idx_cond_n_cols == m_prebuilt->n_template);  // :8516 ★ICP 列必须连续排最前
        auto column_position = i - num_v;                    // :8518
        set_templ_icp(templ, index, m_prebuilt->index, column_position);  // :8520 填 icp_rec_field_no
        ut_ad(templ->icp_rec_field_no != ULINT_UNDEFINED);   // :8522
        /* Index condition pushdown can be used on all columns of a
        secondary index, and on the PRIMARY KEY columns. On the clustered
        index, it must never be used on other than PRIMARY KEY columns,
        because those columns may be stored off-page, and we will not
        fetch externally stored columns before checking the index condition. */
      }
      if (innobase_is_v_fld(table->field[i])) num_v++;       // :8546-8547
    }

    ut_ad(m_prebuilt->idx_cond_n_cols > 0);                  // :8551
    ut_ad(m_prebuilt->idx_cond_n_cols == m_prebuilt->n_template);  // :8552
    num_v = 0;                                               // :8554 ★复位，供循环②重新编号
```

**循环②（同分支内的非 ICP 列）完整代码（`:8558-8599`）**

```cpp
    /* Include the fields that are not needed in index condition pushdown. */
    for (i = 0; i < n_fields; i++) {                         // :8558 【循环②】
      mysql_row_templ_t *templ;
      bool index_contains;
      if (innobase_is_v_fld(table->field[i])) {              // :8562
        index_contains = dict_index_contains_col_or_prefix(index, num_v, true);   // :8563
      } else {
        index_contains = dict_index_contains_col_or_prefix(index, i - num_v, false);  // :8565-8566
      }
      bool is_v = innobase_is_v_fld(table->field[i]);        // :8569
      if (!build_template_needs_field_in_icp(index, m_prebuilt, index_contains,
                                             is_v ? num_v : i - num_v, is_v)) {  // :8571-8572 ★判据取反
        /* Not needed in ICP */
        const Field *field;
        if (whole_row) {
          field = table->field[i];                           // :8577
        } else {
          field = build_template_needs_field(
              index_contains, m_prebuilt->read_just_key, fetch_all_in_key,
              fetch_primary_key_cols, index, table, i, num_v);       // :8579-8581
          if (!field) {                                      // :8582
            if (innobase_is_v_fld(table->field[i])) num_v++; // :8583-8584
            continue;                                        // :8586
          }
        }
        templ = build_template_field(m_prebuilt, clust_index, index, table,
                                     field, i - num_v, num_v);   // :8590-8591 ★v_no 传 num_v
        if (templ->is_virtual) num_v++;                      // :8593-8594
      }
    }
    m_prebuilt->idx_cond = true;                             // :8599
```

★ 两个循环的 `whole_row` 分支都直接取 `table->field[i]`——若该列恰是虚拟列会命中 `:8513` 断言，这是"整行 + ICP"组合的隐含前提（ICP 下推本身只针对普通列）。

**循环③（无 ICP）完整代码（`:8606-8662`）**

```cpp
  } else {                                                   // :8600
    mysql_row_templ_t *templ;
    ulint num_v = 0;                                         // :8602 独立计数器
    /* No index condition pushdown */
    m_prebuilt->idx_cond = false;                            // :8604

    for (i = 0; i < n_fields; i++) {                         // :8606 【循环③】
      const Field *field;
      bool is_virtual = innobase_is_v_fld(table->field[i]);  // :8608

      if (whole_row) {                                       // :8610
        /* Even this is whole_row, if the search is on a virtual
        column, and read_just_key is set, and field is not in this
        index, we will not try to fill the value since they are not
        stored in such index nor in the cluster index. */
        if (is_virtual && m_prebuilt->read_just_key &&
            !dict_index_contains_col_or_prefix(m_prebuilt->index, num_v, true)) {  // :8617-8619
          /* Turn off ROW_MYSQL_WHOLE_ROW */
          m_prebuilt->template_type = ROW_MYSQL_REC_FIELDS;  // :8621 ★整行模板被降级
          num_v++;                                           // :8622
          continue;                                          // :8623
        }
        field = table->field[i];                             // :8626
      } else {
        bool contain;
        if (innobase_is_v_fld(table->field[i])) {            // :8630
          contain = dict_index_contains_col_or_prefix(index, num_v, true);   // :8631
        } else {
          contain = dict_index_contains_col_or_prefix(index, i - num_v, false);  // :8633
        }
        field = build_template_needs_field(
            contain, m_prebuilt->read_just_key, fetch_all_in_key,
            fetch_primary_key_cols, index, table, i, num_v); // :8636-8638
        if (!field) {                                        // :8639
          if (is_virtual) num_v++;                           // :8640-8641
          continue;                                          // :8643
        }
      }

      templ = build_template_field(m_prebuilt, clust_index, index, table, field,
                                   i - num_v, num_v);        // :8647-8648
      /* Virtual columns may have to be read from the secondary index
      before evaluating an end-range condition in
      row_search_end_range_check(). Set ICP field number for virtual column. */
      auto scan_index = m_prebuilt->index;                   // :8653
      bool is_sec_idx = (scan_index != nullptr && !scan_index->is_clustered());  // :8654
      if (is_virtual && is_sec_idx) {                        // :8656
        set_templ_icp(templ, index, scan_index, num_v);      // :8657 供 end_range 用
      }
      if (templ->is_virtual) num_v++;                        // :8660-8661
    }
  }
```

★ 循环③是**唯一能把 `template_type` 从 `ROW_MYSQL_WHOLE_ROW` 降级为 `ROW_MYSQL_REC_FIELDS`** 的地方（`:8621`），条件是"整行 + `read_just_key` + 虚拟列不在该索引中"——这类虚拟列既不属于二级索引也不在聚簇索引里，根本无处取值。★ 即使整条 SQL 没走 ICP，只要扫描的是二级索引，虚拟列的 end-range 检查仍需从索引里取值，所以这里也要 `set_templ_icp`（`:8657`）。

**④ 收尾循环：需要回表时切换 `rec_field_no`（`:8666-8675`）**

```cpp
  if (index != clust_index && m_prebuilt->need_to_access_clustered) {  // :8666
    /* Change rec_field_no's to correspond to the clustered index record */
    for (i = 0; i < m_prebuilt->n_template; i++) {           // :8669 【循环④】只后处理，不改 n_template
      mysql_row_templ_t *templ = &m_prebuilt->mysql_template[i];
      templ->rec_field_no = templ->clust_rec_field_no;       // :8672
    }
  }
}                                                            // :8675 返回类型 void
```

★ 这是"**延迟决策**"：三个建模板循环事先不知道扫描期间是否真的会回表，于是 `build_template_field` 同时写好两个位置——`clust_rec_field_no`（在聚簇索引里的字段号）与 `rec_field_no`（在当前索引里的字段号，可能是 `ULINT_UNDEFINED`）；一旦最终 `need_to_access_clustered` 为真，就在最后**统一**把取值位置改指聚簇记录。★ 循环④**只改 `rec_field_no`**，不重算 `col_no`/`mysql_col_offset` 等其它字段。★ 本函数内**没有** `ut_ad(n_template <= n_fields)`；相关断言是 `:8429`（索引类型）、`:8269`（聚簇定位）、`:8513/:8516/:8522`（ICP 约束）、`:8551-8552`（ICP 计数）。

> **逐列判定与字段填充在两个辅助函数里**：`build_template_needs_field`（要不要这一列）与 `build_template_field`（模板项怎么填）——下两节逐个给全。

#### build_template_needs_field：这一列到底要不要（`ha_innodb.cc:8165-8218`）

纯**过滤谓词**（不写任何东西）：返回 `Field*` = 该列要进 `mysql_template`；返回 `nullptr` = 跳过、`n_template` 不增长——取行时完全跳过它。

```cpp
/** Determines if a field is needed in a m_prebuilt struct 'template'.
 @return field to use, or NULL if the field is not needed */
static const Field *build_template_needs_field(
    bool index_contains,       // :8166 索引是否含该列（dict_index_contains_col_or_prefix 的结果）
    bool read_just_key,        /* :8169 true when MySQL calls ha_innobase::extra
                                with HA_EXTRA_KEYREAD; it is enough to read just
                                columns defined in the index */
    bool fetch_all_in_key,     // :8175 true=fetch all fields in the index
    bool fetch_primary_key_cols,   // :8178 true=fetch the primary key columns
    dict_index_t *index, const TABLE *table,
    ulint i,                   // :8183 ★MySQL 字段下标（table->field[i] 直接用）
    ulint num_v)               // :8184 已累计的虚拟列个数
{
  const Field *field = table->field[i];                      // :8186

  if (!index_contains) {                                     // :8188 ① 索引里没这一列
    if (read_just_key) {
      /* If this is a 'key read', we do not need
      columns that are not in the key */
      return (nullptr);                                      // :8193 ★key-read → 直接放弃
    }
  } else if (fetch_all_in_key) {                             // :8195 ② 在索引里且要求全取
    /* This field is needed in the query */
    return (field);                                          // :8198
  }

  if (bitmap_is_set(table->read_set, static_cast<uint>(i)) ||
      bitmap_is_set(table->write_set, static_cast<uint>(i))) {    // :8201-8202 ③ SQL 层读/写集合包含它
    /* This field is needed in the query */
    return (field);                                          // :8205
  }

  ut_ad(i >= num_v);                                         // :8208
  if (fetch_primary_key_cols &&
      dict_table_col_in_clustered_key(index->table, i - num_v)) {  // :8209-8210 ④ 该列属于聚簇键
    /* This field is needed in the query */
    return (field);                                          // :8213
  }

  /* This field is not needed in the query, skip it */
  return (nullptr);                                          // :8218 兜底：不进模板
}
```

★ 两个 `i` 空间要分清：`table->field[i]` 用 **MySQL 下标**，而 `dict_table_col_in_clustered_key` 用**物理列号 `i - num_v`**（含 `ut_ad(i >= num_v)` 守护）。★ 本函数**没有** `is_virtual` 参数——虚拟列只以 `num_v` 形式参与归一化；`is_virtual` 参数属于下一个函数。★ `dict_index_contains_col_or_prefix` 在聚簇索引上**无条件返回 true**（`dict0dict.cc:866-868`），所以"回表语义"天然由它兜住。

#### build_template_needs_field_in_icp：ICP 专属判定（`ha_innodb.cc:8223-8237`）

```cpp
/** Determines if a field is needed in a m_prebuilt struct 'template'.
 @return whether the field is needed for index condition pushdown */
inline bool build_template_needs_field_in_icp(
    const dict_index_t *index,      /* 建模板所用的索引（可能是聚簇） */
    const row_prebuilt_t *prebuilt, /* 持有真正被扫描的 prebuilt->index */
    bool contains,                  /* 调用者预先算好的 contains */
    ulint i,                        /* ★InnoDB 列号：非虚拟传 i-num_v，虚拟传 num_v */
    bool is_virtual) {
  ut_ad(contains == dict_index_contains_col_or_prefix(index, i, is_virtual));  // :8232 自校验

  return (index == prebuilt->index ? contains
                                   : dict_index_contains_col_or_prefix(
                                         prebuilt->index, i, is_virtual));  // :8234-8236
}
```

★ **与 `build_template_needs_field` 的分工差异**：本函数**只看"索引是否 contains 该列"**（不与 `read_set`/`write_set` 交互），回答的问题是"这一列的 `icp_rec_field_no` 需不需要算"——因为**索引条件下推评估的是索引记录里的值，与 SQL 层是否 SELECT 该列无关**；而另一个函数回答"该列的值要不要拷回 MySQL 行缓冲"。两者在调用点（`:8499` 与 `:8490`）是**与关系**。★ `index == prebuilt->index ? contains : 重算` 的短路语义：只有"建模板用的索引"与"即将扫描的索引"相同时才能复用调用者算好的 `contains`，否则必须针对**真正被扫描的索引**重新判定。

#### build_template_field：模板每一项怎么填（`ha_innodb.cc:8241-8344`）

这是"**模板项构造器**"：输入一个已通过 `build_template_needs_field` 过滤的 `Field*`，输出填好的 `mysql_row_templ_t`——**前面四核心函数的 `templ->mysql_col_offset`/`mysql_col_len`/`mysql_type`/`mysql_length_bytes` 等参数，全部在这里算出**。

```cpp
/** Adds a field to a m_prebuilt struct 'template'.
 @return the field template */
static mysql_row_templ_t *build_template_field(
    row_prebuilt_t *prebuilt,  /*!< in/out: template */
    dict_index_t *clust_index, /*!< in: InnoDB clustered index */
    dict_index_t *index,       /*!< in: InnoDB index to use */
    TABLE *table,              /*!< in: MySQL table object */
    const Field *field,        /*!< in: field in MySQL table */
    ulint i,                   /* ★已减 num_v 的物理列号，非 table->field 下标 */
    ulint v_no) {              /* 虚拟列序号（v_cols 下标） */
  mysql_row_templ_t *templ;
  const dict_col_t *col;

  ut_ad(clust_index->table == index->table);                 // :8253 两索引同表

  templ = prebuilt->mysql_template + prebuilt->n_template++;  // :8255 ★占用槽位 + 后置自增
  UNIV_MEM_INVALID(templ, sizeof *templ);                    // :8256 调试下标记未初始化（release no-op）

  if (innobase_is_v_fld(field)) {                            // :8258 虚拟列（gcol_info && !stored_in_db）
    templ->is_virtual = true;                                // :8259
    col = &dict_table_get_nth_v_col(index->table, v_no)->m_col;  // :8260 ★按 v_no 取
  } else {
    templ->is_virtual = false;                               // :8262
    col = index->table->get_col(i);                          // :8263 ★按物理列号取
  }

  if (!templ->is_virtual) {                                  // :8266
    templ->col_no = i;                                       // :8267 ★仅非虚拟列写 col_no
    templ->clust_rec_field_no = dict_col_get_clust_pos(col, clust_index);  // :8268 聚簇记录中的字段号
    ut_a(templ->clust_rec_field_no != ULINT_UNDEFINED);      // :8269 ★硬断言：必须能定位
    if (index->is_clustered()) {                             // :8271
      templ->rec_field_no = templ->clust_rec_field_no;       // :8272
    } else {
      templ->rec_field_no = index->get_col_pos(i);           // :8274 二级索引内位置，可能 UNDEFINED
    }
  } else {                                                   // :8276 虚拟列
    templ->clust_rec_field_no = v_no;                        // :8277 ★虚拟列用 v_no 作字段号
    if (index->is_clustered()) {
      templ->rec_field_no = templ->clust_rec_field_no;       // :8279
    } else {
      templ->rec_field_no = index->get_col_pos(v_no, false, true);  // :8281 is_virtual=true
    }
  }

  /* Set in set_templ_icp(). */
  templ->icp_rec_field_no = ULINT_UNDEFINED;                 // :8286 ★占位，由 set_templ_icp 写

  if (field->is_nullable()) {                                // :8288
    templ->mysql_null_byte_offset = field->null_offset();    // :8289 = m_null_ptr - record[0]
    templ->mysql_null_bit_mask = (ulint)field->null_bit;     // :8291 位掩码 1/2/4/…/128
  } else {
    templ->mysql_null_bit_mask = 0;                          // :8293 ★0 即"没有 NULL 位"的唯一判据
  }

  templ->mysql_col_offset = (ulint)get_field_offset(table, field);  // :8296 列值在 MySQL 行缓冲里的偏移
  templ->mysql_col_len = (ulint)field->pack_length();        // :8297 列在 MySQL 行格式中的长度（含长度前缀）
  if (templ->is_virtual && innobase_is_multi_value_fld(field)) {   // :8298 多值索引虚拟列
    templ->mysql_mvidx_len = static_cast<ulint>(field->key_length());  // :8299
    templ->is_multi_val = true;                              // :8300
  } else {
    templ->mysql_mvidx_len = 0;                              // :8302
    templ->is_multi_val = false;                             // :8303
  }
  templ->type = col->mtype;                                  // :8305 ★InnoDB mtype：直接搬运，不推导
  templ->mysql_type = (ulint)field->type();                  // :8306 MySQL 类型码（<256）

  if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR) {        // :8308 ★本函数唯一的类型分流
    templ->mysql_length_bytes = field->get_length_bytes();   // :8309 1 或 2
  } else {
    templ->mysql_length_bytes = 0;                           // :8311
  }

  templ->charset = dtype_get_charset_coll(col->prtype);      // :8314 从 prtype 取 charset-collation 号
  templ->mbminlen = col->get_mbminlen();                     // :8315
  templ->mbmaxlen = col->get_mbmaxlen();                     // :8316
  templ->is_unsigned = col->prtype & DATA_UNSIGNED;          // :8317

  if (!index->is_clustered() && templ->rec_field_no == ULINT_UNDEFINED) {  // :8319 ★二级索引里没有该列
    prebuilt->need_to_access_clustered = true;               // :8320 必须回表
  }

  /* For spatial index, we need to access cluster index. */
  if (dict_index_is_spatial(index)) {                        // :8324
    prebuilt->need_to_access_clustered = true;               // :8325 ★空间索引一律回表
  }

  if (prebuilt->mysql_prefix_len <
      templ->mysql_col_offset + templ->mysql_col_len) {      // :8328-8329 维护"行缓冲最右边界"
    prebuilt->mysql_prefix_len = templ->mysql_col_offset + templ->mysql_col_len;  // :8330
  }

  if (DATA_LARGE_MTYPE(templ->type)) {                       // :8333 大字段
    prebuilt->templ_contains_blob = true;                    // :8334 供 row0sel 选逐字段拷贝路径
  }

  if (templ->type == DATA_POINT) {                           // :8337 ★只认 DATA_POINT，不含 VAR_POINT
    /* We set this only when it's DATA_POINT, but not
    DATA_VAR_POINT */
    prebuilt->templ_contains_fixed_point = true;             // :8340 定长点类型需特殊 NULL 处理
  }

  return (templ);                                            // :8343
}
```

**模板字段 → 赋值来源全表**（`mysql_row_templ_t` 定义在 `row0mysql.h:491-540`）：

| 模板字段 | 赋值表达式 | 行号 | 备注 |
|---|---|---|---|
| `col_no` | `i`（物理列号） | :8267 | ★**仅非虚拟列**赋值 |
| `clust_rec_field_no` | 非虚拟 `dict_col_get_clust_pos(col, clust_index)`；虚拟 `v_no` | :8268 / :8277 | 聚簇记录中的字段号 |
| `rec_field_no` | 非虚拟 `index->is_clustered() ? clust : index->get_col_pos(i)`；虚拟 `...get_col_pos(v_no,false,true)` | :8271-8282 | 可能 `ULINT_UNDEFINED` → 触发回表 |
| `icp_rec_field_no` | `ULINT_UNDEFINED` | :8286 | ★占位，真值由 `set_templ_icp()` 在调用点写 |
| `mysql_col_offset` | `get_field_offset(table, field)` | :8296 | MySQL 行缓冲内偏移 |
| `mysql_col_len` | `field->pack_length()` | :8297 | 含 VARCHAR 长度前缀 |
| `mysql_null_byte_offset` | `field->null_offset()` | :8289 | ★NOT NULL 列**不赋值** |
| `mysql_null_bit_mask` | `(ulint)field->null_bit`；否则 `0` | :8291 / :8293 | ★`0` 是唯一判据 |
| `type` | `col->mtype` | :8305 | ★只搬运，不推导（推导在建表期 `get_innobase_type_from_mysql_type`） |
| `mysql_type` | `(ulint)field->type()` | :8306 | MySQL 类型码 |
| `mysql_length_bytes` | `mysql_type == DATA_MYSQL_TRUE_VARCHAR ? get_length_bytes() : 0` | :8308-8312 | 1 或 2 |
| `charset` / `mbminlen` / `mbmaxlen` | `dtype_get_charset_coll(col->prtype)` / `col->get_mbminlen()` / `col->get_mbmaxlen()` | :8314-8316 | 与 `charset.md` 的 prtype 下沉呼应 |
| `is_unsigned` | `col->prtype & DATA_UNSIGNED` | :8317 | |
| `is_multi_val` / `mysql_mvidx_len` | 多值索引虚拟列专用 | :8298-8304 | |

★ 一个消费侧约定：`mysql_null_byte_offset` 只在 `mysql_null_bit_mask != 0` 时有意义——使用侧正是这样守卫的（`row0sel.cc` 里一律 `if (templ->mysql_null_bit_mask)` 包住对 offset 的读取）。★ `n_template` 是 **10 位位域**（`row0mysql.h`，`unsigned n_template : 10`），单模板上限 1023 项；数组本身按 `table->s->fields` 分配（`:8437`），`n_template++` 不做溢出检查——这是已知硬上限。

**收尾一环：`set_templ_icp`（`ha_innodb.cc:8351-8371`）**——`icp_rec_field_no` 的唯一写入者：

```cpp
/** Set Index Condition Push down (ICP) field number in template. */
static void set_templ_icp(mysql_row_templ_t *templ, const dict_index_t *index,
                          const dict_index_t *scan_index, ulint col_position) {
  if (scan_index == nullptr || templ == nullptr) {           // :8353 空参早退
    return;
  }

  bool is_virtual = templ->is_virtual;                       // :8357
  auto icp_field_no = templ->rec_field_no;                   // :8358 默认沿用模板项自己的字段号

  if (scan_index != index) {                                 // :8360 建模板索引 ≠ 被扫描索引
    /* First, try to find the column position without prefix. */
    icp_field_no = scan_index->get_col_pos(col_position, false, is_virtual);  // :8362
  }

  /* Try any prefix of the column. Used in end_range comparison. */
  if (icp_field_no == ULINT_UNDEFINED) {                     // :8366 无前缀找不到 → 退而求"前缀列"
    ut_ad(!scan_index->is_clustered());                      // :8367 聚簇索引不该走到这
    icp_field_no = scan_index->get_col_pos(col_position, true, is_virtual);   // :8368
  }
  templ->icp_rec_field_no = icp_field_no;                    // :8370
}
```

★ 它处理的是"**建模板用的索引**"与"**真正被扫描的索引**"不一致的情况：默认沿用 `rec_field_no`，不一致时到扫描索引里重查位置；查不到完整列就退一步找**前缀列**（`end_range` 比较允许用前缀）。★ `ut_ad(!scan_index->is_clustered())` 说明"前缀列"只可能出现在二级索引里。

#### read_set / write_set 是什么（为什么 `SELECT a` 比 `SELECT *` 快）

**server 侧声明**（`sql/table.h:1593`）——注意 `write_set` **没有任何独立注释块**：

```cpp
  /**
    The read set contains the set of columns that the execution engine needs to
    process the query. In particular, it is used to tell the storage engine
    which columns are needed. For virtual generated columns, the underlying base
    columns are also added, since they are required in order to calculate the
    virtual generated columns.
    ...
    Set during resolving; every field that gets resolved, sets its own bit
    in the read set. In some cases, we switch the read set around during
    various phases; note that it is a pointer.
    ...
   */
  MY_BITMAP *read_set{nullptr};

  MY_BITMAP *write_set{nullptr};     // ★ 8.0.39 里【没有】独立注释
```

```cpp
// sql/table.h:1905 —— 切换与辅助
  void mark_columns_needed_for_update(THD *thd, bool mark_binlog_columns);
  void mark_columns_needed_for_delete(THD *thd);
  inline void column_bitmaps_set_no_signal(MY_BITMAP *r, MY_BITMAP *w) {
    read_set = r; write_set = w;
  }
  inline void use_all_columns() { column_bitmaps_set(&s->all_set, &s->all_set); }
  inline void default_column_bitmaps() {
    read_set = &def_read_set; write_set = &def_write_set;
  }
```

**`MY_BITMAP` 只有 4 个字段**（`include/my_bitmap.h:41`）：

```cpp
typedef uint32 my_bitmap_map;
struct MY_BITMAP {
  my_bitmap_map *bitmap{nullptr};
  uint n_bits{0};                    /* number of bits occupied by the above */
  my_bitmap_map last_word_mask{0};
  my_bitmap_map *last_word_ptr{nullptr};
};
```

**★ 唯一进入 InnoDB 的地方**（`ha_innodb.cc:8201`，`build_template_needs_field()`）：

```cpp
static const Field *build_template_needs_field(
    bool index_contains, bool read_just_key, bool fetch_all_in_key,
    bool fetch_primary_key_cols, dict_index_t *index, const TABLE *table,
    ulint i, ulint num_v) {
  const Field *field = table->field[i];

  if (!index_contains) {
    if (read_just_key) {
      /* If this is a 'key read', we do not need columns that are not in the key */
      return (nullptr);
    }
  } else if (fetch_all_in_key) {
    return (field);              /* This field is needed in the query */
  }

  if (bitmap_is_set(table->read_set, static_cast<uint>(i)) ||
      bitmap_is_set(table->write_set, static_cast<uint>(i))) {
    return (field);              /* This field is needed in the query */
  }                              // ★★ handler 接口 ↔ 引擎 关于列裁剪的【全部】契约

  ut_ad(i >= num_v);
  if (fetch_primary_key_cols &&
      dict_table_col_in_clustered_key(index->table, i - num_v)) {
    return (field);
  }
  return (nullptr);              /* not needed, skip */
}
```

> ★ 注意它**同时看 read_set 和 write_set**——UPDATE 需要写集列。返回 `nullptr` ⇒ 该列不进 `mysql_template` ⇒ `row_search_mvcc` 不回表读它。

**完整链路（为什么 `SELECT a` 更快）**：

```
rnd_init / index_init
  └─ change_active_index() → build_template(false)          ha_innodb.cc:10521
       └─ 对每一列 build_template_needs_field()
            └─ bitmap_is_set(read_set, i) || bitmap_is_set(write_set, i)   :8201
       ⇒ 产出 template_type / n_template / mysql_prefix_len / need_to_access_clustered
  └─ row_search_mvcc() → row_sel_store_mysql_rec()
       只遍历 mysql_template[0..n_template)，按 templ->rec_field_no 取值、
       按 templ->mysql_col_offset/len 写回 buf；mysql_prefix_len 决定 memcpy 长度
```

**性能差异的三个层次**：
1. `n_template` 小 ⇒ 逐列转换（字符集/类型换算）次数少
2. `mysql_prefix_len` 短 ⇒ 每行 `memcpy` 的字节数少
3. 若所有需要的列都在二级索引里，`need_to_access_clustered == false` ⇒ **完全不回表**

```cpp
// build_template 末尾：二级索引回表时把字段号整体换成聚簇索引的字段号
  if (index != clust_index && m_prebuilt->need_to_access_clustered) {
    /* Change rec_field_no's to correspond to the clustered index record */
    for (i = 0; i < m_prebuilt->n_template; i++) {
      mysql_row_templ_t *templ = &m_prebuilt->mysql_template[i];
      templ->rec_field_no = templ->clust_rec_field_no;
    }
  }
```

> ★ 这段就是 8.0 里**取代老版 `clust_templ_for_sec` 语义**的机制——不再另建一套模板，而是把已有模板的字段号整体改写。⚠️ `clust_templ_for_sec` 已不是 `row_prebuilt_t` 的成员，它只是 `row0sel.cc` 里的局部 bool / 形参（`:2515`、`2746`、`4456` 等）。
>
> ⚠️ **勘误**：`mark_columns_needed_for_delete()` / `..._for_update()`（`sql/table.cc:5565` / `:5642`）里的 `HA_REQUIRES_KEY_COLUMNS_FOR_DELETE` / `HA_PRIMARY_KEY_REQUIRED_FOR_DELETE` 两个分支，**对 InnoDB 都不成立**（InnoDB 有 `pcur`）。对 InnoDB 真正生效的只有最后两句：`mark_generated_columns(true)` 与 `mark_check_constraint_columns(true)`。

**它是什么**：`TABLE` 上的两个位图 `MY_BITMAP read_set` / `write_set`，**每列一个 bit**，标记"**本次语句**需要读 / 要写哪些列"。

**谁设置它**：优化器在 prepare/resolve 阶段。语句里出现过的列（SELECT 列表、WHERE、JOIN 条件、ORDER BY、GROUP BY、写语句要更新的列…）会被 `mark_column_used()` 打进位图。所以：

| 语句 | `read_set` 内容 |
|---|---|
| `SELECT a FROM t WHERE b > 1` | 只有 `a`、`b` 两列 |
| `SELECT * FROM t` | 全部列 |
| `UPDATE t SET a = 1 WHERE b = 2` | `read_set` 含 `b`，`write_set` 含 `a` |

**谁用它**：三层都会看它——

1. **server 层**：判断某列是否"被本次查询用到"（决定是否要为该列做转换/校验）；
2. **InnoDB `build_template(false)`** ——上面第 ③ 步，决定模板里有哪些列；
3. **`row_search_mvcc` 取行时**只转换模板里的列。

★ 所以 `SELECT a FROM t` 比 `SELECT *` 快的**完整因果链**是：

```
查询只用到 a
  → read_set 只有 a 这一位
  → build_template(false) 只建 1 个 mysql_row_templ_t（n_template = 1）
  → 取每行时只做 1 次字段转换 + 1 次 memcpy
  → 其余列连"从 rec 里解析 offset"这一步都省了
```

不只是"少读 IO"——**每行的 CPU 转换开销也按列数线性下降**。这在宽表（几十列）上差异极大。

**两个易踩的坑**：

- `write_set` 也参与判断：所以 `UPDATE` 即使只改一列，模板里也要有被改的列（否则写不回去）；
- **锁定读会绕过这个优化**——见上，`LOCK_X` 直接 `whole_row = true`，`read_set` 此时不起作用。

### ★ clust_templ_for_sec

这个标志属于 `row_search_mvcc` / `row_sel_store_mysql_rec` 的**执行细节**，已移至 [`../innodb/row_search.md`](../innodb/row_search.md) 的「**★ clust_templ_for_sec：扫描二级索引、但模板基于主键**」一节——那里有**完整链路**：`row_sel_store_mysql_rec` 的 `template_col` 映射与 `sec_field_no` 反查、`row_sel_store_mysql_field` 的字段号借出-归还与三条取数路径、end-range 两处使用点、以及与 ICP `icp_rec_field_no` 的同源对照。

此处只留一句提要：**扫描二级索引、但模板按主键建**（`index != clust_index && need_to_access_clustered`）。因为二级索引记录里隐含主键列，模板中的 PK 列可以**直接从二级索引记录抠出来**，剩下的才回表——这是"覆盖索引 / 回表"二分之外的第三种形态。

### fetch cache：顺序扫描时的批量缓存

**数据与常量**（`row0mysql.h`）：

```cpp
constexpr uint32_t MYSQL_FETCH_CACHE_SIZE = 8;        // 自建缓存槽数
/* After fetching this many rows, we start caching them in fetch_cache */
constexpr uint32_t MYSQL_FETCH_CACHE_THRESHOLD = 4;   // "攒够 4 行才开缓存"的自适应阈值
constexpr uint32_t ROW_PREBUILT_FETCH_MAGIC_N = 465765687;  // 每槽前后各 4 字节的腐败探测魔数

  byte *fetch_cache[MYSQL_FETCH_CACHE_SIZE];   // 8 个槽：每槽 = [4B 魔数][mysql_row_len 行副本][4B 魔数]
  ulint fetch_cache_first;    // 消费者下标
  ulint n_fetch_cached;       // 生产者计数（同时是隐含写下标）
  ulint n_rows_fetched;       // 定位后已返回行数（兼作阈值判据）
  bool m_no_prefetch;         // 整体关停开关（分区表用）
```

★ 缓存的是"**已转成 MySQL 行格式的副本**"（不是 InnoDB rec、不是指针）——所以跨 mtr/页闩使用**无悬垂风险**（与 BLOB 指针相反，见下）。★ 取满 4 行才开缓存的动机：**顺序扫描**才值得缓存；随机点查缓存命中率低，缓存反而浪费内存与拷贝——这是"用前几行判断这是不是顺序扫描"的自适应策略。

**分配：`row_sel_prefetch_cache_init`（`row0sel.cc:3639-3669`，惰性、一次 malloc、首尾魔数）**

```cpp
static inline void row_sel_prefetch_cache_init(row_prebuilt_t *prebuilt) {
  ulint i, sz;
  byte *ptr;

  /* We use our own prefetch cache only if the server didn't provide one. */
  ut_ad(row_sel_get_record_buffer(prebuilt) == nullptr);   // ★ 只有 server 没给 Record_buffer 才自建

  /* Reserve space for the magic number. */
  sz = UT_ARR_SIZE(prebuilt->fetch_cache) * (prebuilt->mysql_row_len + 8);  // 8×(行长+8)
  ptr = static_cast<byte *>(ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sz));

  for (i = 0; i < UT_ARR_SIZE(prebuilt->fetch_cache); i++) {
    /* A user has reported memory corruption in these
    buffers in Linux. Put magic numbers there to help
    to track a possible bug. */
    mach_write_to_4(ptr, ROW_PREBUILT_FETCH_MAGIC_N);      // :3660 槽首魔数
    ptr += 4;
    prebuilt->fetch_cache[i] = ptr;                        // :3663 指向数据区（跳过首魔数）
    ptr += prebuilt->mysql_row_len;
    mach_write_to_4(ptr, ROW_PREBUILT_FETCH_MAGIC_N);      // :3666 槽尾魔数
    ptr += 4;
  }
}
```

★ 分配大小 = `8 × (mysql_row_len + 8)`；`fetch_cache[i]` 指向"首魔数后 4 字节"，所以 `fetch_cache[0] - 4` 就是 malloc 基址（释放路径据此还原，见「生命周期」节）。★ **没有配对的 free 函数**（全仓库 `prefetch_cache` 只出现在 `row0sel.cc`）——释放内联在 `row_prebuilt_free`。★ 唯一调用点是 `row_sel_fetch_last_buf`（`:3690`），即"第一次真要写缓存槽"时才分配；★ 8.0.29+ 若 server（executor）提供了 `Record_buffer`，则**完全不用 `fetch_cache[]`**（保持全 nullptr），槽由 `Record_buffer::add_record()` 线性分配——`fetch_cache[0] != nullptr` 是"自建块存在"的唯一判据。

**入队/出队：FIFO + 整批回绕**

```cpp
// :3705 入队
static inline void row_sel_enqueue_cache_row_for_mysql(byte *mysql_rec,
                                                       row_prebuilt_t *prebuilt) {
  /* For non ICP code path the row should already exist in
  the next fetch cache slot. */
  if (prebuilt->idx_cond) {                                  // ICP：候选行先在调用方 buf 求值，需拷入槽
    byte *dest = row_sel_fetch_last_buf(prebuilt);           // 取下一个空槽（必要时触发 init）
    ut_memcpy(dest, mysql_rec, prebuilt->mysql_prefix_len);  // 只拷有效前缀（末尾未请求列不拷）
  }
  ++prebuilt->n_fetch_cached;                                // ★ 唯一入队计数点
}

// :3569 出队（节选：三条拷贝路径 + 回绕）
static inline void row_sel_dequeue_cached_row_for_mysql(byte *buf, row_prebuilt_t *prebuilt) {
  ...
  cached_rec = record_buffer
                   ? record_buffer->record(prebuilt->fetch_cache_first)   // server 缓冲优先
                   : prebuilt->fetch_cache[prebuilt->fetch_cache_first];  // 自建数组兜底
  if (UNIV_UNLIKELY(prebuilt->keep_other_fields_on_keyread)) {
    row_sel_copy_cached_fields_for_mysql(buf, cached_rec, prebuilt);   // key-read：只覆盖缓存里有的列
  } else if (prebuilt->mysql_prefix_len > 63) {                        // 长行：逐列拷
    ut_memcpy(buf, cached_rec, prebuilt->null_bitmap_len);             // 先整块拷 NULL 位图
    for (i = 0; i < prebuilt->n_template; i++) {
      templ = prebuilt->mysql_template + i;
      if (templ->is_virtual) { ... continue; }                         // 虚拟列常规不拷
      row_sel_copy_cached_field_for_mysql(buf, cached_rec, templ);     // 逐列拷（避免复制长 VARCHAR 未用尾部）
    }
  } else {
    ut_memcpy(buf, cached_rec, prebuilt->mysql_prefix_len);            // 短行快路径：一次 memcpy
  }

  prebuilt->n_fetch_cached--;        // :3625 消费一行
  prebuilt->fetch_cache_first++;     // :3626 消费者下标推进
  if (prebuilt->n_fetch_cached == 0) {
    /* All the prefetched records have been returned.
    Rewind so that we can insert records at the beginning ... */
    prebuilt->fetch_cache_first = 0;                 // :3632 整批耗尽 → 回绕
    if (record_buffer != nullptr) record_buffer->clear();   // ★ clear 保留 out_of_range 标志（reset 会清）
  }
}
```

★ 命中缓存是 **memcpy 到调用方 `buf`**（≤63 字节整块拷、长行逐列拷）——**不是"指针切换"**（"切换"只发生在源码行 `record_buffer ? ... : fetch_cache[first]` 这个取槽三元表达式上）。★ 回绕用 `clear()` 而非 `reset()`：**保留 `m_out_of_range`**，供 `row_search_mvcc` 在缓存耗尽后直接返回 `DB_RECORD_NOT_FOUND`（`:4608-4616`）；回绕同时满足生产者断言 `ut_ad(prebuilt->fetch_cache_first == 0)`（`:3698`）。

**状态机（`row_search_mvcc`）**：三个计数器的语义非常干净——

| 计数器 | 何时变 | 语义 |
|---|---|---|
| `n_fetch_cached` | **只**在 enqueue(`+1`) / dequeue(`-1`) | 未消费行数（生产者计数） |
| `fetch_cache_first` | 每次 dequeue `+1`，整批耗尽归 0 | 消费者下标 |
| `n_rows_fetched` | dequeue 命中 `+1`、真取行 `+1`（带 1e9 回绕保护） | 兼作"≥4 才预取"判据 |

- **清零点**：`direction == 0`（语句第一行请求）时三计数器全清（`:4553-4557`）；**滚动游标反向**时也清（`:4576-4592`，★ 且"反向时缓存必须已空"是 `ut_error` 断言；另有 `ut_ad(record_buffer == nullptr)`——滚动游标不会拿到 Record_buffer）。
- **命中即返回**：`n_fetch_cached > 0` → dequeue + `goto func_exit`（`:4593-4599`），**整个 B-tree descent 不发生**。
- **硬不变量**：缓存耗尽后 `fetch_cache_first == 0`（`:4606` 的 `ut_ad`）。
- **灌满机制**（`:5612-5637`）：`n_rows_fetched >= 4`（或 `record_buffer != nullptr`）且 `can_prefetch_records()` 时，在**同一次 mtr/页闩**内连续转换后续行直到灌满（`goto next_rec`）——★ 这就是省 CPU 的原理：**8 行共享一次定位 + 一个 mtr**，摊薄 B-tree descent、latch 与行转换成本。首批第一行直写调用方 `buf`（零拷贝返回），其余写缓存槽（`next_buf` 机制）。

**启用条件（`row_prebuilt_t::can_prefetch_records()`，`row0mysql.cc:4801`）**：

```cpp
return select_lock_type == LOCK_NONE && !m_no_prefetch &&
       !templ_contains_blob && !templ_contains_fixed_point &&
       !clust_index_was_generated && !used_in_HANDLER && !innodb_api &&
       template_type != ROW_MYSQL_DUMMY_TEMPLATE && !in_fts_query;
```

即**上锁读（`select_lock_type != LOCK_NONE`，如 UPDATE 内的读）/ 模板含 BLOB 或定长 POINT / 无主键自增聚簇索引 / HANDLER 语句 / InnoDB API / CHECK TABLE（dummy 模板）/ FTS 查询 / 分区扫描（`m_no_prefetch`）** 全部禁用。★ **二级索引回表不在禁用名单**——那正是它的主场（`need_to_access_clustered` 不参与判断）。

**★ 与 BLOB 的关系（为什么 BLOB 是硬性排除项）**：

BLOB 在 MySQL 行格式里是"长度 + **8 字节数据指针**"（`row_mysql_store_blob_ref`），指针指向 `blob_heap`——而 `row_sel_store_mysql_rec` **每次行转换都会 `mem_heap_empty(blob_heap)`**（`:2912-2914`），即 **prebuilt 只保留"一行"的 BLOB 内容**。若缓存 8 行，前一行的 BLOB 指针会在后几行转换时失效 → 所以有 `!templ_contains_blob`；源码原话（`:5628-5630`）：*"Since we keep space in prebuilt only for the BLOBs of a single row, we cannot cache rows in the case there are BLOBs in the fields to be fetched."*。**反推**：预取缓存装的是"**按值拷贝的整行副本**"（VARCHAR 数据整段拷进槽），所以跨 mtr 使用没有任何悬垂风险——这是与 BLOB 指针相反的设计。

**释放**：见「生命周期」节（`row_prebuilt_free`，`:995-1013` 逐槽校验首尾魔数后 `ut::free`）；语句级清理（`HA_EXTRA_FLUSH` / `end_stmt`）**只清 `blob_heap`、不动 fetch cache**——缓存内容靠下一条语句的计数器清零实现"逻辑作废 + 就地覆写"。

### 生命周期：创建、绑定与释放（`row0mysql.cc`）

`row_prebuilt_t` 结构体本体 **427 行 / 约 84 个成员**（`row0mysql.h:553-979`，含 6 个字段分组分隔段），它的生老病死由四个函数掌管，共约 316 行源码：

| 函数 | 位置 | 职责 |
|---|---|---|
| `row_create_prebuilt` | `:805-952`（152 行） | 创建：建堆 + zalloc + 建 dtuple/游标 |
| `row_update_prebuilt_trx` | `:1030-1048`（24 行） | 事务切换时重绑 `trx` |
| `row_prebuilt_free` | `:957-1028`（73 行） | 释放：逐个 free 附属结构 + 主堆 |
| `row_mysql_prebuilt_free_blob_heap` | `:147-155` | 单独释放 `blob_heap`（跨语句存活） |

**魔数（use-after-free 的第一道防线）**：

```cpp
constexpr uint32_t ROW_PREBUILT_ALLOCATED = 78540783;
constexpr uint32_t ROW_PREBUILT_FREED = 26423527;
```

双魔数 `magic_n` / `magic_n2`（`row0mysql.h`）在创建时都置 `ALLOCATED`（`:885-886`），释放时都置 `FREED`（`:963-964`）——`row_search_mvcc` 入口的 `ut_a(prebuilt->magic_n == ROW_PREBUILT_ALLOCATED)`（`:4501`）就是第一道防线，**双份**是为了同时防悬垂与越界/错位写。

#### `row_create_prebuilt`（`:805-952`，完整）

```cpp
row_prebuilt_t *row_create_prebuilt(dict_table_t *table, ulint mysql_row_len) {  // :805-807
  DBUG_TRACE;
  row_prebuilt_t *prebuilt;
  mem_heap_t *heap;              // :813 ★ 唯一的主堆
  dict_index_t *clust_index;
  dict_index_t *temp_index;
  dtuple_t *ref;
  ulint ref_len;
  uint srch_key_len = 0;         // :818 搜索键转换缓冲大小
  ulint search_tuple_n_fields;

  search_tuple_n_fields =       // :821 ★ 预留 2 倍列数：前一半存 start key、后一半存 stop key
      2 * (table->get_n_cols() + dict_table_get_n_v_cols(table));   // :822 含虚拟列

  clust_index = table->first_index();                              // :824 第一索引即聚簇索引

  /* Make sure that search_tuple is long enough for clustered index */
  ut_a(2 * table->get_n_cols() >= clust_index->n_fields);          // :827 ★ 断言容量足够

  ref_len = dict_index_get_n_unique(clust_index);                  // :829 聚簇唯一键长度

  /* PREBUILT_HEAP_INITIAL_SIZE：一次性预估足够大的堆，减少 malloc 次数
     （sizeof(*prebuilt) + DTUPLE_EST_ALLOC(search_tuple) + ... + sizeof(*pcur) + sizeof(*clust_pcur)） */
#define PREBUILT_HEAP_INITIAL_SIZE  ...                            // :831-847 见源码宏定义

  /* :849-871 计算 rows_in_range() 用的 InnoDB 格式 INT 搜索键缓冲：
     MySQL 小端 / InnoDB 大端且翻转符号位，故 INT 键需转换缓冲；start+end 两份 ⇒ 取所有索引中的最大值 */
  for (temp_index = table->first_index(); temp_index; temp_index = temp_index->next()) {  // :857
    uint temp_len = 0;                                             // :861
    for (uint i = 0; i < temp_index->n_uniq; i++) {                // :862
      ulint type = temp_index->fields[i].col->mtype;               // :863
      if (type == DATA_INT) temp_len += temp_index->fields[i].fixed_len;   // :864-865 只统计整数列
    }
    srch_key_len = std::max(srch_key_len, temp_len);               // :868 ★ 取最大值
  }
  ut_a(srch_key_len <= MAX_SRCH_KEY_VAL_BUFFER);                   // :871

  heap = mem_heap_create(PREBUILT_HEAP_INITIAL_SIZE + 2 * srch_key_len,   // :879 ★ 建主堆
                         UT_LOCATION_HERE);

  prebuilt = static_cast<row_prebuilt_t *>(                        // :882-883 ★ 实例本身也从主堆 zalloc（全部成员清零）
      mem_heap_zalloc(heap, sizeof(*prebuilt)));

  prebuilt->magic_n = ROW_PREBUILT_ALLOCATED;                      // :885 ★ 双魔数
  prebuilt->magic_n2 = ROW_PREBUILT_ALLOCATED;                     // :886

  prebuilt->table = table;                                         // :888 只存指针（引用计数由调用方持有）
  prebuilt->sql_stat_start = true;                                 // :890
  prebuilt->heap = heap;                                           // :891

  prebuilt->srch_key_val_len = srch_key_len;                       // :893
  if (prebuilt->srch_key_val_len) {                                // :894
    prebuilt->srch_key_val1 = static_cast<byte *>(
        mem_heap_alloc(prebuilt->heap, 2 * prebuilt->srch_key_val_len));   // :895-896
    prebuilt->srch_key_val2 = prebuilt->srch_key_val1 + prebuilt->srch_key_val_len;  // :897-898
  } else {
    prebuilt->srch_key_val1 = nullptr;                             // :900
    prebuilt->srch_key_val2 = nullptr;                             // :901
  }

  prebuilt->pcur = static_cast<btr_pcur_t *>(                      // :904-905 ★ 游标也来自主堆（非 new）
      mem_heap_zalloc(prebuilt->heap, sizeof(btr_pcur_t)));
  prebuilt->clust_pcur = static_cast<btr_pcur_t *>(                // :906-907
      mem_heap_zalloc(prebuilt->heap, sizeof(btr_pcur_t)));
  prebuilt->pcur->reset();                                         // :908 ★ 复位（清 old_rec_buf/位置状态）
  prebuilt->clust_pcur->reset();                                   // :909

  prebuilt->select_lock_type = LOCK_NONE;                          // :911
  prebuilt->select_mode = SELECT_ORDINARY;                         // :912

  prebuilt->search_tuple = dtuple_create(heap, search_tuple_n_fields);      // :914 ★ start key
  prebuilt->m_stop_tuple = dtuple_create(heap, search_tuple_n_fields);      // :915 ★ end key
  ut_ad(!prebuilt->m_stop_tuple_found);                            // :916 zalloc 保证 false
  ut_ad(!prebuilt->is_reading_range());                            // :917

  ref = dtuple_create(heap, ref_len);                              // :919 ★ 聚簇唯一键 dtuple
  dict_index_copy_types(ref, clust_index, ref_len);                // :921
  prebuilt->clust_ref = ref;                                       // :923

  prebuilt->autoinc_error = DB_SUCCESS;                            // :925
  prebuilt->autoinc_offset = 0;                                    // :926
  /* Default to 1, we will set the actual value later in ha_innobase::get_auto_increment(). */
  prebuilt->autoinc_increment = 1;                                 // :930 步长默认 1
  prebuilt->autoinc_last_value = 0;                                // :932

  /* During UPDATE and DELETE we need the doc id. */
  prebuilt->fts_doc_id = 0;                                        // :935

  prebuilt->mysql_row_len = mysql_row_len;                         // :937 ★ 保存 MySQL 行长
  prebuilt->ins_sel_stmt = false;                                  // :939
  prebuilt->session = nullptr;                                     // :940
  prebuilt->fts_doc_id_in_read_set = false;                        // :942
  prebuilt->blob_heap = nullptr;                                   // :943 ★ BLOB 堆懒创建
  prebuilt->no_read_locking = false;                               // :945
  prebuilt->no_autoinc_locking = false;                            // :946
  prebuilt->m_no_prefetch = false;                                 // :948
  prebuilt->m_read_virtual_key = false;                            // :949

  return prebuilt;                                                 // :951 无失败分支（OOM 由堆层终止）
}
```

**七个要点**（逐个都是易错点）：

1. ★ **只建 1 个堆 `heap`**（`:879`），prebuilt 实例本身也从它 `mem_heap_zalloc`（`:883`）——整个对象与 `search_tuple`/`m_stop_tuple`/`clust_ref`/`srch_key_val*`/`pcur`/`clust_pcur` **共用这一个堆**，最后 `mem_heap_free(heap)` 一次性回收。
2. ★ **`mysql_template` 不在创建时分配**（本函数零提及），首次 `build_template` 才 `ut::malloc_withkey`（`ha_innodb.cc:8436-8439`）——**它也不在主堆上**，所以 `row_prebuilt_free` 里要单独 `ut::free`（`:973`）。
3. ★ **三个"懒创建、不属于主堆"的 heap**：`blob_heap`（BLOB 拷贝，`row0sel.cc:2766/2868`）、`old_vers_heap`（MVCC 旧版本，`row0sel.cc:710/740`）、`cursor_heap`（**仅 InnoDB API 使用**，`api0api.cc`，`ha_innobase` 路径永远是 nullptr）。
4. ★ **游标用 `mem_heap_zalloc` + `btr_pcur_t::reset()`**（`:904-909`），**不是** `btr_pcur_create_for_mysql()`（那个走 `ut::new`，见 `btr0pcur.h`）。
5. ★ **执行图全部懒建**：`ins_graph`/`upd_graph`/`sel_graph`/`ins_node`/`upd_node` 靠 zalloc 得 nullptr，首用时才在 `heap` 上建。
6. ★ **`m_mysql_handler` / `m_mysql_table` 不在这里设置**——它们在调用方 `ha_innobase::open`（`ha_innodb.cc:7402-7403`）；`default_rec` 也在那里（`:7399`）。
7. **`search_tuple_n_fields = 2 × 列数（含虚拟列）`**（`:821`）——**两倍**是因为同一对 dtuple 要同时承载 start key 与 stop key；`:827` 的断言保证它容得下聚簇索引的全部字段。

#### `row_update_prebuilt_trx`：事务切换时重绑（`:1030-1048`）

```cpp
void row_update_prebuilt_trx(row_prebuilt_t *prebuilt, trx_t *trx) {   // :1030
  ut_a(trx->magic_n == TRX_MAGIC_N);                    // :1031 trx 有效性
  ut_a(prebuilt->magic_n == ROW_PREBUILT_ALLOCATED);    // :1032 prebuilt 未释放
  ut_a(prebuilt->magic_n2 == ROW_PREBUILT_ALLOCATED);   // :1033

  prebuilt->trx = trx;                                  // :1035 ★ 更新当前事务句柄

  if (prebuilt->ins_graph) prebuilt->ins_graph->trx = trx;   // :1037-1038 同步已建执行图
  if (prebuilt->upd_graph) prebuilt->upd_graph->trx = trx;   // :1041-1042
  if (prebuilt->sel_graph) prebuilt->sel_graph->trx = trx;   // :1045-1046
}
```

★ 只改 `trx` 与**已建**执行图的 `->trx`——**不动** `search_tuple`/`m_stop_tuple`（那两个 dtuple 只存索引键值，不含 `trx_id`）。★ 调用链是"**同一 `TABLE` 句柄被不同连接复用**"的场景：`ha_innobase::external_lock`（`ha_innodb.cc:18638`）第一句 `update_thd(thd)`（`:18644`）→ `check_trx_exists` → 若 `m_prebuilt->trx` 不同则 `row_update_prebuilt_trx(m_prebuilt, trx)`（`:3004-3006`）。

#### `row_prebuilt_free`：释放顺序（`:957-1028`）

```cpp
void row_prebuilt_free(row_prebuilt_t *prebuilt, bool dict_locked) {   // :957
  DBUG_TRACE;

  ut_a(prebuilt->magic_n == ROW_PREBUILT_ALLOCATED);    // :960 ★ 释放前先校验（防重复释放/野指针）
  ut_a(prebuilt->magic_n2 == ROW_PREBUILT_ALLOCATED);   // :961

  prebuilt->magic_n = ROW_PREBUILT_FREED;               // :963 ★ 置 FREED：此后任何误用都会被断言捕获
  prebuilt->magic_n2 = ROW_PREBUILT_FREED;              // :964

  ut_a(!prebuilt->is_reading_range());                  // :968 处于 read_range 中时先崩，好过写脏内存

  prebuilt->pcur->reset();                              // :970 复位游标（清 old_rec_buf）
  prebuilt->clust_pcur->reset();                        // :971

  ut::free(prebuilt->mysql_template);                   // :973 ★ 模板不在主堆上，单独 free（nullptr 安全）

  if (prebuilt->ins_graph) que_graph_free_recursive(prebuilt->ins_graph);   // :975-976
  if (prebuilt->sel_graph) que_graph_free_recursive(prebuilt->sel_graph);   // :979-980
  if (prebuilt->upd_graph) que_graph_free_recursive(prebuilt->upd_graph);   // :983-984

  if (prebuilt->blob_heap) row_mysql_prebuilt_free_blob_heap(prebuilt);     // :987-988 ★ 懒建，可能为空
  if (prebuilt->old_vers_heap) mem_heap_free(prebuilt->old_vers_heap);      // :991-992 ★ 懒建，可能为空

  if (prebuilt->fetch_cache[0] != nullptr) {            // :995 ★ 预取缓存是「独立 buf + 首尾魔数」
    byte *base = prebuilt->fetch_cache[0] - 4;          // :996 fetch_cache[i] 前 4 字节是魔数
    byte *ptr = base;
    for (ulint i = 0; i < MYSQL_FETCH_CACHE_SIZE; i++) { // :999 逐行校验 8 行
      ulint magic1 = mach_read_from_4(ptr);             // :1000
      ut_a(magic1 == ROW_PREBUILT_FETCH_MAGIC_N);       // :1001 行首魔数
      ptr += 4;
      byte *row = ptr;
      ut_a(row == prebuilt->fetch_cache[i]);            // :1005 断言行指针与数组一致（防越界/错位）
      ptr += prebuilt->mysql_row_len;                   // :1006 跳过一行
      ulint magic2 = mach_read_from_4(ptr);             // :1008
      ut_a(magic2 == ROW_PREBUILT_FETCH_MAGIC_N);       // :1009 行尾魔数
      ptr += 4;
    }
    ut::free(base);                                     // :1013 释放整块
  }

  if (prebuilt->rtr_info) rtr_clean_rtr_info(prebuilt->rtr_info, true);     // :1016-1017 R-tree 搜索信息

  if (prebuilt->table) {                                // :1020
    ut_ad(!prebuilt->table->is_fts_aux());              // :1021
    dd_table_close(prebuilt->table, nullptr, nullptr, dict_locked);   // :1022 ★ 递减表引用计数
  }

  prebuilt->m_lob_undo.destroy();                       // :1025 释放 LOB MVCC undo 缓存

  mem_heap_free(prebuilt->heap);                        // :1027 ★ 最后释放主堆：prebuilt 自身 + 所有堆附属结构一次性回收
}
```

★ **释放顺序有讲究**："模板 → 三个执行图 → blob_heap → old_vers_heap → fetch cache → rtr_info → 表引用 → 主堆"，**主堆放最后**（因为 prebuilt 自身就住在里面）。★ `fetch_cache` 用的是**带首尾魔数的独立 buf**（`ROW_PREBUILT_FETCH_MAGIC_N`，每行前后各 4 字节）——释放时逐行校验魔数并断言指针一致，这是防"预取缓存越界/错位"的又一道防线。★ `dict_locked` 参数透传给 `dd_table_close`：`false`（`ha_innobase::close` 传的，`ha_innodb.cc:7691`）表示内部要自己加字典锁，`true` 表示调用方已持锁、跳过加锁避免死锁。★ `m_lob_undo` 是 LOB 的 MVCC undo 缓存（见 [`../innodb/physical/lob.md`](../innodb/physical/lob.md)）。

#### `row_mysql_prebuilt_free_blob_heap`：为什么 BLOB 堆要单独管理（`:147-155`）

```cpp
void row_mysql_prebuilt_free_blob_heap(row_prebuilt_t *prebuilt) {
  DBUG_TRACE;
  DBUG_PRINT("row_mysql_prebuilt_free_blob_heap",
             ("blob_heap freeing: %p", prebuilt->blob_heap));
  mem_heap_free(prebuilt->blob_heap);                   // :153 释放整个堆（内部 BLOB 拷贝随之释放）
  prebuilt->blob_heap = nullptr;                        // :154 ★ 置空：既防二次释放，也是"当前无 blob 堆"的判据
}
```

★ **它必须独立成函数的原因**：`blob_heap` **不属于主堆**，而且它的生命周期**比一次查询更长**——批量预取时记录被缓存在 `fetch_cache[]` 里，其中指向 BLOB 数据的指针必须指向一块"**在返回给 MySQL 之前始终有效**"的内存。所以它在三个时机被释放（**都不是**每行结束）：`extra(HA_EXTRA_FLUSH)`（`ha_innodb.cc:18371`）、`end_stmt`（`:18435`，每条语句结束）、`row_prebuilt_free`（`row0mysql.cc:988`）。★ 本函数**只释放堆本身**，不清理 `fetch_cache[]`、也不重置 `n_fetch_cached` / `m_temp_read_shared`（那些由 `row_search_mvcc` 在语句起点重置，`row0sel.cc:4553-4556`）。

#### 生命周期时序

| 阶段 | 位置 | 调用 | 做什么 | 涉及 heap |
|---|---|---|---|---|
| **打开** | `ha_innodb.cc:7397` | `row_create_prebuilt(ib_table, table->s->reclength)` | 建 `heap`；zalloc prebuilt；建 `search_tuple`/`m_stop_tuple`/`clust_ref`/`pcur`/`clust_pcur`/`srch_key_val*`；置双魔数 | **创建 `heap`（唯一）** |
| 打开后补设 | `ha_innodb.cc:7399-7418` | 直接赋值 | `default_rec`、`m_mysql_table`、`m_mysql_handler` | — |
| **事务绑定/切换** | `ha_innodb.cc:18644`→`:3005` | `external_lock`→`update_thd`→`row_update_prebuilt_trx` | `prebuilt->trx = trx`，同步三个执行图 | — |
| 语句起点 | `row0sel.cc:4553-4556` | `row_search_mvcc` | 重置 `n_rows_fetched`/`n_fetch_cached`/`fetch_cache_first`/`m_end_range` | — |
| 读旧版本（懒建） | `row0sel.cc:710`,`740` | `row_sel_build_prev_vers(_committed)_for_mysql` | 懒创建 `old_vers_heap`，语句间 `mem_heap_empty` 复用 | 懒建 `old_vers_heap` |
| 读 BLOB（懒建） | `row0sel.cc:2766`,`2868` | `row_sel_store_mysql_field` | 懒创建 `blob_heap`，拷 BLOB 数据 | 懒建 `blob_heap` |
| 批量预取 | `row0sel.cc:3628-3719` | `row_sel_prefetch_cache_init` 等 | 用 `fetch_cache[]`（带首尾魔数的独立 buf）缓存多行 | 独立 buf（非 heap） |
| 语句结束 | `ha_innodb.cc:18434` | `end_stmt`→`row_mysql_prebuilt_free_blob_heap` | 释放 `blob_heap` | 释放 `blob_heap` |
| 刷新 | `ha_innodb.cc:18370` | `extra(HA_EXTRA_FLUSH)`→ 同上 | 同上 | 释放 `blob_heap` |
| **关闭** | `ha_innodb.cc:7691` | `close`→`row_prebuilt_free(m_prebuilt, false)` | 逐个 free 附属结构，**最后** `mem_heap_free(heap)` | 释放 `old_vers_heap`/`blob_heap`/fetch cache/`heap` |

#### heap 与"非堆内存"归属表

| 名称 | 创建位置 | 装什么 | 释放位置 |
|---|---|---|---|
| **`heap`**（主堆） | `row0mysql.cc:879` | prebuilt 自身、`search_tuple`、`m_stop_tuple`、`clust_ref`、`srch_key_val1/2`、`pcur`、`clust_pcur`、懒建的 ins/upd/sel 执行图 | `row0mysql.cc:1027` `mem_heap_free` |
| `old_vers_heap` | 懒建 `row0sel.cc:710/740` | MVCC 一致性读构建的旧版本记录 | `row0mysql.cc:992`（语句间用 `mem_heap_empty` 复用） |
| `blob_heap` | 懒建 `row0sel.cc:2766/2868` | SELECT 时 BLOB/TEXT/GEOMETRY 的数据拷贝 | `row0mysql.cc:153`；调用点 `:988`、`ha_innodb.cc:18371`、`:18435` |
| `cursor_heap` | 仅 InnoDB API（`api0api.cc:1677/1713/1766`） | InnoDB API 的 `innodb_api_buf` | API 层自行归还——**`ha_innobase` 路径永远是 nullptr** |
| **`mysql_template`**（非堆） | 懒建 `ha_innodb.cc:8436-8439` `ut::malloc_withkey` | `mysql_row_templ_t[n_fields]` | `row0mysql.cc:973` `ut::free` |
| **`fetch_cache` buf**（非堆） | 懒建 `row0sel.cc:3690` `row_sel_prefetch_cache_init` | `MYSQL_FETCH_CACHE_SIZE`(8) 行，每行前后各 4 字节魔数 | `row0mysql.cc:995-1013` 校验后 `ut::free` |

**一句话总结**：`row_create_prebuilt` 用一个 zalloc 的主堆承载 prebuilt 骨架与几乎所有附属结构，`trx` / `mysql_template` / `blob_heap` / `old_vers_heap` 分别由事务切换、`build_template`、读取路径懒建；`row_prebuilt_free` 按"模板 → 图 → 堆 → 缓存 → 表引用 → 主堆"顺序释放，开头把双魔数置 `ROW_PREBUILT_FREED` 作 use-after-free 防线；`blob_heap` 因须**跨越批量预取**存活而独立管理。

### 与 handler 的关系

`row_prebuilt_t` 由 `ha_innobase::open` 时创建、`close` 时释放，**每个 `ha_innobase` 实例一份**（不像 `TABLE_SHARE` 那样跨连接共享）。所以它是"**会话私有、表私有**"的：同一张表在两个连接里各有各的 prebuilt、各有各的持久游标状态。

## Misc

### handler 接口覆盖地图（哪些方法在本篇、哪些在别处）

| handler 方法 | 源码（`ha_innodb.cc`） | 本篇覆盖 | 主归属（若不在本篇） |
|---|---|---|---|
| `external_lock` | `:18638`（约 287 行） | ✅ 专节（9 步表 + quiesce 深剖 + 2PC 注册） | — |
| `extra` | `:18300` 附近 | ✅ 专节（含 `skip_alter_undo` 深剖） | — |
| `position` | `:11249` | ✅ 完整实现 + ref_length/row_id 机制 | — |
| `rnd_pos` | `:10863` | ✅ 完整实现 + DATA_SYS 特例 | — |
| `reset_template` | `:3166` | ✅ 生命周期节 | — |
| `is_record_buffer_wanted` | `:23843` | ✅ fetch cache 节 | — |
| **`open`** | `:7177`（约 500 行） | ⚠️ 只讲"创建 prebuilt"那一环 | ★ [`dd/innodb_dict.md`](dd/innodb_dict.md)（`dict_table_t` 加载）+ [`table.md`](table.md)（`open_table` 流程） |
| `close` | `:7680` | ⚠️ 只讲"释放 prebuilt"那一环 | 关闭链路归 `table.md` / `dd/innodb_dict.md` |
| **`info` / `info_low`** | `:17525` / `:17208`（约 400 行） | ❌ 仅一句提及 | ★ [`../index/stats.md`](../index/stats.md)（`rec_per_key` / 行数统计） |
| `records_in_range` | `:16697` | ⚠️ 键读链路里提过调用点 | ★ [`query/07_optimize/physical/08_range_optimizer.md`](query/07_optimize/physical/08_range_optimizer.md) 等优化器文档 |
| `index_init` / `index_end` | `:10056` / `:10069` | ⚠️ 仅提及 | ★ [`../innodb/row_search.md`](../innodb/row_search.md)（索引切换与游标） |
| `write_row` / `update_row` / `delete_row` | `:9004` / `:9758` / `:9917` | ❌ | ★ [`../innodb/row_ops.md`](../innodb/row_ops.md)（DML 状态机） |
| `index_read` / `rnd_next` | — | ❌ | ★ [`../innodb/row_search.md`](../innodb/row_search.md)（读路径入口） |
| `start_stmt` / `end_stmt` | `:18489` / `:18433` | ⚠️ `external_lock` 节内提及 | — |
| `truncate_impl` | `:15263` | ❌ | ★ DDL 文档（[`../feat/ddl/`](../feat/ddl/)） |

> **本篇的边界**：只讲"**接口层本身**"（handler/handlerton 抽象、prebuilt 缓存、两条意图通道 `external_lock`/`extra`）——DML 与读路径的**流程**归 `row_ops.md` / `row_search.md`，统计信息归 `index/stats.md`，表打开/关闭的完整链路归 `dd/innodb_dict.md` / `table.md`。

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

#### `ha_*` 包装层：server 与引擎之间那层做了什么

调用方写 `table->file->ha_rnd_next(buf)`，引擎实现的是 `rnd_next(buf)`（**无 `ha_` 前缀**）。中间这层就在 `handler::ha_rnd_next`——它不只是转发：

```cpp
int handler::ha_rnd_next(uchar *buf) {
  int result;
  DBUG_EXECUTE_IF("ha_rnd_next_deadlock", return HA_ERR_LOCK_DEADLOCK;);
  DBUG_TRACE;
  assert(table_share->tmp_table != NO_TMP_TABLE || m_lock_type != F_UNLCK);
  assert(inited == RND);          // ★ 契约：必须先 rnd_init 且已加锁（临时表除外）

  // Set status for the need to update generated fields
  m_update_generated_read_fields = table->has_gcol();   // ★ 表有虚拟生成列就置位

  // ★ 引擎调用被 PSI 埋点包住：MAX_KEY 表示全表扫（无索引）
  MYSQL_TABLE_IO_WAIT(PSI_TABLE_FETCH_ROW, MAX_KEY, result,
                      { result = rnd_next(buf); })     // ← 真正调引擎的 rnd_next

  if (!result && m_update_generated_read_fields) {
    // ★ 虚拟生成列在"读到行之后"才计算，引擎不负责算
    result = update_generated_read_fields(buf, table);
    m_update_generated_read_fields = false;
  }
  table->set_row_status_from_handler(result);
  return result;
}
```

⇒ 这层承担了引擎实现不管的三件事：

1. **PSI 埋点** —— `performance_schema` 的 `table_io_waits_*` 就计在这里，引擎侧无感
2. **虚拟生成列的读取后计算** —— 引擎返回的是"未计算生成列的行"，`update_generated_read_fields` 在拿到 buf 之后补算
3. **调用契约的断言** —— `inited == RND`（必须先 init）、锁状态（临时表豁免）

**命名约定**：`ha_` 前缀 = **server 侧包装**（`ha_rnd_next` / `ha_index_read` / `ha_rnd_pos`），**无前缀 = 引擎实现**（`rnd_next` / `index_read` / `rnd_pos`）。⇒ 看到 `ha_xxx` 就知道里面还裹着 PSI 与前后处理，不是直接调引擎。

#### 状态谁维护：`inited` 与 `m_start_of_scan` 是**两回事**

**`inited` 是 server 层的状态机**（`sql/handler.h:4508`）——注意它是**匿名枚举成员**，且 8.0.39 有 **4 态**（不是 3 态）：

```cpp
 public:
  /**
    End value for a range scan. If this is NULL the range scan has no
    end value. Should also be NULL when there is no ongoing range scan.
    Used by the read_range() functions and also evaluated by pushed
    index conditions.
  */
  key_range *end_range;                      // ★ 真正的"是否有范围上界"开关
  uint errkey;                               /* Last dup key */
  uint active_index;                         // 在用的 index 编号
  /** Length of ref (1-8 or the clustered key length) */
  uint ref_length;
  enum { NONE = 0, INDEX, RND, SAMPLING } inited;
  //     ★ INDEX 在 RND【之前】；SAMPLING 是 8.0 TABLESAMPLE 新增的第 4 态
```

```cpp
// sql/handler.h:4466（私有存储）
  /*
    Storage space for the end range value. Should only be accessed using
    the end_range pointer. The content is invalid when end_range is NULL.
  */
  key_range save_end_range;

// :4652 构造函数初值
        end_range(nullptr),
        ref_length(sizeof(my_off_t)),   // ★ 默认 8 字节，引擎可在 ha_open 里改
        inited(NONE),

// :4685 析构断言
  virtual ~handler(void) {
    assert(inited == NONE);   // ★ 漏调 ha_rnd_end/ha_index_end 会在 debug 直接崩
  }
```

**四个 `ha_*` 各自维护状态迁移**（`sql/handler.cc`）：

```cpp
// :2948
int handler::ha_rnd_end() {
  assert(inited == RND);
  inited = NONE;
  end_range = nullptr;
  m_record_buffer = nullptr;      // ★ 多行预读缓冲失效
  return rnd_end();
}

// :2885
int handler::ha_index_init(uint idx, bool sorted) {
  assert(inited == NONE);         // ★ 与 rnd 不同：index init 不允许重复调用
  if (!(result = index_init(idx, sorted))) inited = INDEX;
  mrr_have_range = false;
  end_range = nullptr;
  return result;
}

// :2905
int handler::ha_index_end() {
  assert(inited == INDEX);
  inited = NONE;
  end_range = nullptr;
  m_record_buffer = nullptr;
  if (m_unique) m_unique->reset(false);   // ★ 多值索引去重器复位（不释放）
  return index_end();
}
```

> ⚠️ **勘误**：**server 层 `handler` 没有 `m_start_of_scan`**。全仓仅 3 处命中，都在 InnoDB，是 `ha_innobase` 自己的成员（`storage/innobase/handler/ha_innodb.h:666`）：
> ```cpp
> // ha_innodb.cc:10802
> int ha_innobase::rnd_init(bool scan) {
>   int err = change_active_index(table_share->primary_key);
>   if (!scan) m_prebuilt->row_read_type = ROW_READ_WITH_LOCKS;
>   m_start_of_scan = true;        // ★ InnoDB 自己的"还没取第一行"标志
>   return err;
> }
>
> int ha_innobase::rnd_next(uchar *buf) {
>   ha_statistic_increment(&System_status_var::ha_read_rnd_next_count);
>   if (m_start_of_scan) {         // ★ 第一次走 index_first，之后走 general_fetch
>     error = index_first(buf);
>     if (error == HA_ERR_KEY_NOT_FOUND) error = HA_ERR_END_OF_FILE;
>     m_start_of_scan = false;
>   } else {
>     error = general_fetch(buf, ROW_SEL_NEXT, 0);
>   }
>   return error;
> }
> ```

**`end_range` 的置位与消费**（`handler.cc:7436` / `:7406`）：

```cpp
void handler::set_end_range(const key_range *range,
                            enum_range_scan_direction direction) {
  if (range) {
    save_end_range = *range;                  // ★ 值拷进私有存储
    end_range = &save_end_range;              // ★ 公开指针指向它 —— 非空即"有上界"
    range_key_part = table->key_info[active_index].key_part;
    key_compare_result_on_equal =
        (range->flag == HA_READ_BEFORE_KEY) ? 1
        : (range->flag == HA_READ_AFTER_KEY ? -1 : 0);   // ★ 开/闭区间预计算
    m_virt_gcol_in_end_range = key_has_vcol(range_key_part, range->length);
  } else
    end_range = nullptr;
  ...
}
```

```cpp
    result = ha_index_next(table->record[0]);
    if (result) return result;
    if (compare_key(end_range) > 0) {         // ★ 越界判定
      unlock_row();                           // ★ 越界行也要主动放锁
      result = HA_ERR_END_OF_FILE;
    }
```

> ★ `ha_rnd_next()` **完全不读 `end_range`**（只有 `assert(inited == RND)`）；`end_range` 只在 `ha_index_init/end`、`ha_rnd_init/end` 里被无条件清成 `nullptr`，由 `set_end_range()` 置位，在 range 扫描里消费。
>
> ★ 统一收尾入口 `ha_index_or_rnd_end()`（`handler.h:4766`）：`inited == INDEX ? ha_index_end() : inited == RND ? ha_rnd_end() : 0`。
>
> ★ `ha_rnd_pos()` 的 `assert(inited == RND)` 被**故意注释掉**——`rnd_pos` 可在 `rnd_init(scan=false)` 之外被调用（如 dup update 回查）。而 `ha_index_read_idx_map()` 反过来要求 `assert(inited == NONE)`（自己开自己关，出口再 assert 一次）。

这两个最容易混，但它们**正交**。先看 `ha_rnd_init` 里的两个断言：

```cpp
int handler::ha_rnd_init(bool scan) {
  DBUG_EXECUTE_IF("ha_rnd_init_fail", return HA_ERR_TABLE_DEF_CHANGED;);
  int result;
  DBUG_TRACE;
  // ① 要么临时表，要么已加锁：读表必须持有表锁，临时表是连接私有的所以豁免
  assert(table_share->tmp_table != NO_TMP_TABLE || m_lock_type != F_UNLCK);
  // ② 允许"未初始化"或"已是 RND 且又是 scan"；禁止从 INDEX / SAMPLING 切过来
  assert(inited == NONE || (inited == RND && scan));

  inited = (result = rnd_init(scan)) ? NONE : RND;   // 包装层据结果记状态
  end_range = nullptr;
  return result;
}
```

**② 为什么放行"已 RND 且又是 scan"？** handler.h 的接口注释给了答案：

> *"rnd_init() can be called two times without rnd_end() in between (it only makes sense if scan=1). then the second call should prepare for the new table scan (e.g if rnd_init allocates the cursor, second call should position it to the start of the table, **no need to deallocate and allocate it again**)"*

⇒ 即**允许"不 end 就重新 init"来重开一次扫描**（游标不重新分配，只拨回开头）。所以断言放行"已 RND + scan=1"；而 `inited == INDEX`（正在索引扫描）或 `SAMPLING` 则不被允许——**扫描模式不能半路切换**。

**`rnd` 是什么**：handler 接口的历史命名。`rnd_*` 系列 = **不按索引顺序的全表扫描**，与 `index_*` 系列相对（状态变量 `ha_read_rnd_next_count` 计的就是它）。源码未给出缩写释义，按"读整张表"理解即可。

**两个状态的分工**（★ 关键区分）：

| | `inited`（server 侧 `handler` 成员） | `m_start_of_scan`（引擎侧 `ha_innobase` 成员） |
|---|---|---|
| 取值 | `NONE / INDEX / RND / SAMPLING` 四值 | bool |
| 管什么 | **当前是哪种扫描模式** | **同一次扫描内是不是第一行** |
| 生命周期 | 从 `rnd_init` 到 `rnd_end`，**贯穿整次扫描** | 只在**第一次** `rnd_next` 时为 true |

⇒ **`inited` 不是"是否第一次读"**——它在整次扫描期间一直是 `RND`。"是否第一次读"是 `m_start_of_scan`。

**`m_start_of_scan` 怎么一步步传下去、最终到哪有用**：

```cpp
// ① ha_innobase::rnd_init —— 置位
m_start_of_scan = true;

// ② ha_innobase::rnd_next —— 消费
if (m_start_of_scan) {
  error = index_first(buf);                        // ★ 首次：定位第一行
  if (error == HA_ERR_KEY_NOT_FOUND) {
    error = HA_ERR_END_OF_FILE;                    // 空表 → 转成"读到头"
  }
  m_start_of_scan = false;
} else {
  error = general_fetch(buf, ROW_SEL_NEXT, 0);     // ★ 之后：续读
}
```

⇒ 它决定的是**走 `index_first` 还是 `general_fetch`**，而这最终体现为传给 `row_search_mvcc` 的 `mode`：

| `m_start_of_scan` | 走的方法 | 传给 `row_search_mvcc` 的 mode | PHASE 3 分支 |
|---|---|---|---|
| `true`（首次） | `index_first` | `PAGE_CUR_G` | **open**（从 root 下降定位） |
| `false`（续读） | `general_fetch` | `PAGE_CUR_UNSUPP` | **restore**（恢复已存位置） |

⇒ 这与 [`row_search.md`](../innodb/row_search.md) 里"入口 → mode → PHASE 3 打开/恢复"**是同一条链的两端**。

#### 错误码转换：是**两层**，不是一层

**server 侧的第二级：`HA_ERR_*` → `ER_*` 的映射**（`sql/handler.cc:4199`，`handler::print_error()`，节选）：

```cpp
void handler::print_error(int error, myf errflag) {
  THD *thd = current_thd;
  Foreign_key_error_handler foreign_key_error_handler(thd, this);

  int textno = ER_GET_ERRNO;          // ★ 兜底：未识别的引擎错误码
  switch (error) {
    case EACCES:  textno = ER_OPEN_AS_READONLY; break;   // ★ 前三个是 OS errno
    case EAGAIN:  textno = ER_FILE_USED; break;
    case ENOENT: { ... my_error(ER_FILE_NOT_FOUND, ...); } break;  // ★ case 内直接报

    case HA_ERR_KEY_NOT_FOUND:
    case HA_ERR_NO_ACTIVE_RECORD:
    case HA_ERR_RECORD_DELETED:
    case HA_ERR_END_OF_FILE:
      textno = ER_KEY_NOT_FOUND;      // ★ 扫描结束/记录消失统一表现为 "key not found"
      break;

    case HA_ERR_FOUND_DUPP_KEY: {
      uint key_nr = table ? get_dup_key(error) : -1;
      if ((int)key_nr >= 0) {
        print_keydup_error(table,
            key_nr == MAX_KEY ? nullptr : &table->key_info[key_nr], errflag);
        return;                       // ★ 能定位到具体 key → 走"带 key 名/值"的精细报错
      }
      textno = ER_DUP_KEY;
      break;
    }
    case HA_ERR_OUT_OF_MEM:
      textno = ER_OUT_OF_RESOURCES;   // ★ server 层 OOM
      break;
    case HA_ERR_SE_OUT_OF_MEMORY:
      my_error(ER_ENGINE_OUT_OF_MEMORY, errflag, table->file->table_type());
      return;                         // ★ 引擎层 OOM：直接报，带引擎名，不走尾部统一 my_error
    case HA_ERR_LOCK_WAIT_TIMEOUT: textno = ER_LOCK_WAIT_TIMEOUT; break;
    case HA_ERR_LOCK_DEADLOCK:     textno = ER_LOCK_DEADLOCK; break;
    case HA_ERR_ROW_IS_REFERENCED: {                      // 1451
      String str;
      thd->push_internal_handler(&foreign_key_error_handler);
      get_error_message(error, &str);                     // ★ 向引擎要"哪张子表引用了它"
      my_error(ER_ROW_IS_REFERENCED_2, errflag, str.c_ptr_safe());
      thd->pop_internal_handler();
      return;
    }
    case HA_ERR_NO_REFERENCED_ROW: { ... my_error(ER_NO_REFERENCED_ROW_2, ...); return; }
    ...
    default: {
      /* The error was "unknown" to this function.
         Ask handler if it has got a message for this error */
      String str;
      bool temporary = get_error_message(error, &str);    // ★★ 交给引擎自己描述
      if (!str.is_empty()) {
        const char *engine = table_type();
        if (temporary)
          my_error(ER_GET_TEMPORARY_ERRMSG, errflag, error, str.ptr(), engine);
        else
          my_error(ER_GET_ERRMSG, errflag, error, str.ptr(), engine);
      } else {
        char errbuf[MYSQL_ERRMSG_SIZE];
        my_error(ER_GET_ERRNO, errflag, error,
                 my_strerror(errbuf, MYSQL_ERRMSG_SIZE, error));
      }
      return;
    }
  }
  if (textno != ER_FILE_NOT_FOUND)    // ★ ENOENT 已在 case 内报过，避免重复
    my_error(textno, errflag, table_share->table_name.str, error);
}
```

**第三级：引擎自己的描述文本**（`handler::get_error_message()`）——server 默认**空实现**：

```cpp
// sql/handler.cc:4501
/**
  Return an error message specific to this handler.
  @return true if this is a temporary error
*/
bool handler::get_error_message(int error [[maybe_unused]],
                                String *buf [[maybe_unused]]) {
  return false;   // ★ server 默认：没有消息、也不是临时错误 —— 引擎必须覆写
}
```

**"可忽略 / 致命"的两级判定**（`handler.cc:4139` / `:4164`）——决定了 `IGNORE` 能否吞掉错误：

```cpp
bool handler::is_ignorable_error(int error) {
  switch (error) {
    case 0:                            // 不是错误
    case HA_ERR_FOUND_DUPP_KEY:        // 重复键可显式忽略
    case HA_ERR_FOUND_DUPP_UNIQUE:
    case HA_ERR_ROW_IS_REFERENCED:     // FK 违规可忽略
    case HA_ERR_NO_REFERENCED_ROW:
      return true;
  }
  return false;
}

bool handler::is_fatal_error(int error) {
  if (is_ignorable_error(error)) return false;   // ★ 第一层：可忽略 ⇒ 必然非致命
  switch (error) {
    /* Deadlock and lock timeout cause transaction/statement rollback so
       that THD::is_fatal_sub_stmt_error will be set. ... */
    case HA_ERR_LOCK_WAIT_TIMEOUT:
    case HA_ERR_LOCK_DEADLOCK:
      return false;                              // ★ 第二层：非致命但会触发回滚
    case HA_ERR_NULL_IN_SPATIAL:
      return false;
  }
  return true;                                   // ★ 默认致命
}
```

**错误文本脱敏：`Foreign_key_error_handler`**（`sql/error_handler.h:415` / `error_handler.cc:498`）：

```cpp
bool Foreign_key_error_handler::handle_condition(THD *, uint sql_errno, ...) {
  const TABLE_SHARE *share = m_table_handler->get_table_share();

  if (sql_errno == ER_NO_REFERENCED_ROW_2) {
    for (TABLE_SHARE_FOREIGN_KEY_INFO *fk = share->foreign_key; ...) {
      if (check_table_access(m_thd, TABLE_OP_ACLS, &table, true, 1, true)) {
        my_error(ER_NO_REFERENCED_ROW, MYF(0));   // ★ 无权限 → 降级为不带父表名的模糊报错
        return true;
      }
    }
  } else if (sql_errno == ER_ROW_IS_REFERENCED_2) { ... ER_ROW_IS_REFERENCED ... }
  return false;   // ★ 返回 false = 未处理，原错误继续向上传播（带表名细节）
}
```

> ★ 机制本质是 C++ try/throw/catch 的模拟（源码 doxygen 明说）：`THD::push_internal_handler()` = try，`my_error()` = throw，`handle_condition()` = catch。

**`HA_ERR_*` 的真实形态**（`include/my_base.h:818-1000`）——是一组 `#define` **宏**，不是 enum：

```c
#define HA_ERR_FIRST 120
#define HA_ERR_KEY_NOT_FOUND 120      /* Didn't find key on read or update */
#define HA_ERR_FOUND_DUPP_KEY 121     /* Duplicate key on write */
#define HA_ERR_RECORD_CHANGED 123     /* Uppdate with is recoverable */
#define HA_ERR_CRASHED 126            /* Indexfile is crashed */
#define HA_ERR_OUT_OF_MEM 128
#define HA_ERR_NOT_A_TABLE 130        /* not a MYI file - no signature */
#define HA_ERR_END_OF_FILE 137        /* end in next/prev/first/last */
#define HA_ERR_FOUND_DUPP_UNIQUE 141  /* Duplicate unique on write */
#define HA_ERR_LOCK_WAIT_TIMEOUT 146
#define HA_ERR_LOCK_DEADLOCK 149
#define HA_ERR_NO_REFERENCED_ROW 151  /* Cannot add a child row */
#define HA_ERR_ROW_IS_REFERENCED 152  /* Cannot delete a parent row */
#define HA_ERR_NO_SUCH_TABLE 155
#define HA_ERR_TABLE_DEF_CHANGED 159  /* The table changed in storage engine */
#define HA_ERR_RECORD_IS_THE_SAME 169 /* row not actually updated */
#define HA_ERR_FK_DEPTH_EXCEEDED 192  /* FK cascade depth exceeded */
#define HA_ERR_SE_OUT_OF_MEMORY 194   /* Out of memory in storage engine */
#define HA_ERR_LAST 209
#define HA_ERR_ERRORS (HA_ERR_LAST - HA_ERR_FIRST + 1)   /* = 90 */
```

> ⚠️ 三个命名不规则的成员**没有** `HA_ERR_` 前缀：`HA_WRONG_CREATE_OPTION`(140)、`HA_FTS_INVALID_DOCID`(182)、`HA_MISSING_CREATE_OPTION`(193)。另 **129 号是空洞**（历史遗留）。

InnoDB 内部用 `dberr_t`（`DB_*`），server 只认 `HA_ERR_*`。但转换**分两处**，混在一起讲就会自相矛盾：

**第一层：调用场景自己定语义**（在各 `ha_innobase` 方法里，先按上下文处理）

```cpp
// general_fetch（续读）：
case DB_SUCCESS:          error = 0; break;
case DB_RECORD_NOT_FOUND: error = HA_ERR_END_OF_FILE; break;   // 续读到头 = 正常结束
case DB_END_OF_INDEX:     error = HA_ERR_END_OF_FILE; break;   // 树扫完
// rnd_next 首次走 index_first：
if (error == HA_ERR_KEY_NOT_FOUND) error = HA_ERR_END_OF_FILE; // 空表 → "读到头"
```

⇒ 同一个 `DB_RECORD_NOT_FOUND`：**续读**语境是"读到头"（正常结束），**定位**语境是"没找到"（`HA_ERR_KEY_NOT_FOUND`，错误）。**语义由上下文决定**。

**第二层：兜底映射 `convert_error_code_to_mysql`**（其余错误码统一转换）

| `dberr_t` | `HA_ERR_*` | 副作用 |
|---|---|---|
| `DB_SUCCESS` | `0` | — |
| `DB_DUPLICATE_KEY` | `HA_ERR_FOUND_DUPP_KEY` | 注释警示：MySQL 会**重入存储层**取重复键信息，此时表句柄/事务信息可能已不可用 |
| `DB_DEADLOCK` / `DB_FORCED_ABORT` | `HA_ERR_LOCK_DEADLOCK` | ★ `thd_mark_transaction_to_rollback(thd, 1)` —— **标记整事务回滚** |
| `DB_LOCK_WAIT_TIMEOUT` | 锁等待超时 | ★ `thd_mark_transaction_to_rollback(thd, row_rollback_on_timeout)` —— **是否整事务回滚可配** |
| `DB_READ_ONLY` | `HA_ERR_TABLE_READONLY`（`srv_force_recovery` 时为 `HA_ERR_INNODB_FORCED_RECOVERY`） | — |
| `DB_MISSING_HISTORY` | `HA_ERR_TABLE_DEF_CHANGED` | — |
| `DB_INTERRUPTED` | `HA_ERR_GENERIC` | `thd_set_kill_status()` |
| `DB_RECORD_NOT_FOUND` | `HA_ERR_NO_ACTIVE_RECORD` | ⚠️ 见下方说明 |
| `DB_ERROR` / `default` | `HA_ERR_GENERIC` | — |

⇒ ★ 两个必须记住的点：

1. **`DB_RECORD_NOT_FOUND` 两层映射不同**：第一层（场景）常转成 `HA_ERR_END_OF_FILE` / `HA_ERR_KEY_NOT_FOUND`，兜底函数里却是 `HA_ERR_NO_ACTIVE_RECORD`。**它通常走不到兜底**——各方法已先按场景处理掉了。（这也是我一度写混的地方，两处必须分开看。）
2. **转换不只是改枚举值，还带副作用**：`DB_DEADLOCK` 标记**整事务回滚**，`DB_LOCK_WAIT_TIMEOUT` 则取决于 `row_rollback_on_timeout`（可只回滚语句）。⇒ 这正是"**死锁回滚整事务、超时可配**"的机制来源，与 [`row_search.md`](../innodb/row_search.md)（半一致读）和 [`mvcc.md`](../innodb/mvcc.md) 里讲的行为对得上。

---

## 格式图谱：行 / 列 / 键的多种表示

> MySQL ↔ InnoDB 之间有大量格式，转换就发生在 handler 这一层。本章是"格式"的完整图谱，理解它才能看懂 `position`/`key_copy`/回表等一切转换。

#### ★ 转换链完整盘点（这条链分散在 5 篇，此处是总索引）

| 环节 | 写方向（INSERT / 定位） | 读方向（SELECT / 输出） | 深度剖析在哪 |
|---|---|---|---|
| 查询键 → dtuple | — | `row_sel_convert_mysql_key_to_innobase` | [`row_search.md`](../innodb/row_search.md)（本篇 1020 有精简版） |
| record[0] → dtuple（整行） | `row_mysql_convert_row_to_innobase` | — | [`row_search.md`](../innodb/row_search.md) |
| **每列编码 / 解码** | `row_mysql_store_col_in_innobase_format` | `row_sel_field_store_in_mysql_format_func` | 写 [`data_repr.md`](../infra/structure/data_repr.md)；读 [`row_search.md`](../innodb/row_search.md) |
| dtuple → rec_t | `rec_convert_dtuple_to_rec` | — | [`record.md`](../innodb/physical/record.md)（写方向） |
| rec_t 字段解析 | — | `rec_get_offsets` | [`row_search.md`](../innodb/row_search.md) |
| rec_t → record[0]（整行） | — | `row_sel_store_mysql_rec` | [`row_search.md`](../innodb/row_search.md) |
| **record[0] ⇄ 引擎键** | `key_copy` / `get_key_image` | `key_restore` | **本篇**（key_copy 节） |
| dtuple → MySQL 行（构建） | `row_build` / `row_build_index_entry` | — | [`record_format.md`](../index/record_format.md) |
| 字节级原语 | `mach_write_*` | `mach_read_*` | [`encoding.md`](infra/encoding.md) |
| 字节布局（长度前缀 / NULL 位图） | — | — | [`record.md`](../innodb/physical/record.md) |

**三个必须成对理解的对称点**（单看一边会像怪代码）：

1. **整数符号位**：写方向"倒序 + `^0x80` 置位" ⇄ 读方向"倒序 + `^128` 还原"——XOR 自反，同一招（offset binary，见 [`data_repr.md`](../infra/structure/data_repr.md)）。
2. **VARCHAR 长度头**：行格式 1 或 2 字节 ⇄ **键格式恒 2 字节**（写侧 `row_format_col` 参数、读侧 `DATA_MYSQL_TRUE_VARCHAR` 分支）。
3. ★★ **UCS2 的"半个空格"**：写方向 `col_len &= ~1` 把奇数砍成偶数 ⇄ 读方向 `len & 1` 补回一个 `0x20`。**根因只在写侧**，读侧那个 `if` 单独看无法理解。

> 本篇承担"**图谱 + 导航**"；值级编码细节分派到 [`data_repr.md`](../infra/structure/data_repr.md)（写）与 [`row_search.md`](../innodb/row_search.md)（读），字节布局归 [`record.md`](../innodb/physical/record.md)。

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

`rec_t` 是**字节数组指针**（`rem0types.h`），指向字段数据起始（origin），记录头/长度列表全部在**低地址方向**。REDUNDANT 与 COMPACT/DYNAMIC/COMPRESSED 的记录头布局不同：

```
COMPACT/DYNAMIC/COMPRESSED（new-style，5 字节头）：
[变长长度列表(倒序)][NULL位图][字段数(仅instant)][5B头][字段数据...]
                       5B头 = info bits(4b)+n_owned(4b)+heap_no(13b)+status(3b)+next_offset(2B)

REDUNDANT（old-style，6 字节头）：
[偏移数组(倒序，每字段1/2B，最高位=NULL标记)][6B头][字段数据...]
                       6B头 = next_offset(2B)+short_flag(1b)+n_fields(10b)+heap_no(13b)+n_owned(4b)+info_bits(4b)
```

**系统列**（大端存储，`data0type.h`）：`DB_ROW_ID`（6B，仅无主键）、`DB_TRX_ID`（6B，MVCC）、`DB_ROLL_PTR`（7B，undo 指针）。追加在聚簇索引用户列之后。

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

> ★ 最容易混的两点：①VARCHAR 在 record 里长度头 1/2 字节，但**键里恒 2 字节**；②整数在 InnoDB 是**大端 + 符号位翻转**，在 record[0] 是**小端**——转换时逐字节倒序 + 还原符号位（`row_sel_field_store_in_mysql_format`，逐分支剖析见 [`../innodb/row_search.md`](../innodb/row_search.md)「★ row_sel_field_store_in_mysql_format_func」）。

### 两套 null 位图 + 1-based 偏移

**record[0] 与 rec_t 的 null 约定是两套**：

- **record[0]**：新 .frm 格式 `null_field_first = true`（`table.cc`），null 位图在**行首 byte 0**，字段数据从 **byte 1** 开始——这就是 `Fieldstart = 1` 的来源。旧格式 null 位图在行尾。
- **rec_t**：REDUNDANT 无独立 bitmap（嵌偏移数组最高位）；COMPACT 独立 bitmap 在 5 字节头之后（origin 之前，反向排布）。

`table.cc` 的 `-1` 就是为此：

```cpp
record = (uchar *)outparam->record[0] - 1; /* Fieldstart = 1 */
outparam->null_flags = (uchar *)record + 1;/* record[0] 开头 */
```

净效果 `null_flags = record[0]`；那个 `-1` 是把局部变量退到"偏移 0"基准，让字段偏移（1-based）与 null 位图（byte 0）对齐。配套地，`rec_buff_length = ALIGN_SIZE(reclength + 1)`（`table.cc`）多出的 `+1` 就是给 null 位图 byte 0 留的。

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

`ref` 是 `handler` 类的成员 `uchar *ref`（`handler.h`），长度 `ref_length`（`handler.h`，注释 "1-8 or the clustered key length"）。

**它存的不是行数据，而是"能重新定位到这一行的钥匙"**——不同引擎完全不同：

| 引擎 | ref 内容 | 长度 |
|---|---|---|
| InnoDB 有主键 | 主键列的**引擎键格式**（`key_copy` 产出） | `key_info->key_length` |
| InnoDB 无主键 | 6 字节隐藏 `DB_ROW_ID` | `DATA_ROW_ID_LEN`=6 |
| MyISAM | 文件偏移 `my_off_t` | `sizeof(my_off_t)` |

> ⚠ **ref 不是行数据，是从 record 派生的"定位键"**——它回答"怎么再找到这一行"，不回答"这一行长什么样"。

**ref 缓冲区的分配**（`handler.cc`，`ha_open` 里）：一次 `2 * ALIGN_SIZE(ref_length)` 字节，前半给 `ref`、后半给 `dup_ref`。`ref_length` 由引擎在 open 时定（InnoDB 有主键 = `key_info->key_length`，无主键 = 6）。

### 三种格式：别混淆

> 完整图谱（rec_t / dtuple / record[0] / key / addon 五种 + 双向转换链）见上「格式图谱」章。这里只列与 ref 直接相关的三种：

rowid 链路涉及三种"键/列"格式，容易混：

| 格式 | 谁产出 | 在哪用 | 特点 |
|---|---|---|---|
| **record 列存储格式** | 引擎读行填 `record[0]` | SQL 层计算/比较 | VARCHAR 有长度前缀、BLOB 是"长度+外置指针"、CHAR 定长无前缀 |
| **引擎键格式** | `key_copy` | `index_read`/`position` 的 ref | 可空 keypart 前 1 字节 NULL 标记、变长 2 字节长度头、定长空格填充 |
| **addon 格式** | `Field::pack` | filesort 排序记录 | 紧凑序列化，见 [runtime/01](query/runtime/01_filesort.md) 1.3 |

`position()` 的 `ref` 是**引擎键格式**——既不是 record 格式，也不是 InnoDB 内部格式。

### position 实现（`ha_innodb.cc`）

```c
if (m_prebuilt->clust_index_was_generated) {
  /* 无主键：ref = 6 字节隐藏 row id */
  len = DATA_ROW_ID_LEN;                            // data0type.h，= 6
  memcpy(ref, m_prebuilt->row_id, len);
} else {
  /* 有主键：ref = 主键列的引擎键格式 */
  KEY *key_info = table->key_info + table_share->primary_key;
  key_copy(ref, (uchar *)record, key_info, key_info->key_length);  // ← 关键
  len = key_info->key_length;
}
```

**无主键的 `DB_ROW_ID` 是什么**：InnoDB 为无主键表自动生成的隐藏聚簇键，全局单调递增（`dict_sys_get_new_row_id`，`dict0boot.ic`），6 字节大端存储（`mach_write_to_6`）。`m_prebuilt->row_id` 是当前扫描行的 row id 缓存。

**完整函数（`ha_innobase.cc:11249-11280`）与两个配套机制**：

```cpp
/**
Store a reference to the current row to 'ref' field of the handle.
Note that in the case where we have generated the clustered index for the
table, the function parameter is illogical: we MUST ASSUME that 'record'
is the current 'position' of the handle, because if row ref is actually
the row id internally generated in InnoDB, then 'record' does not contain
it. We just guess that the row id must be for the record where the handle
was positioned the last time.
@param[in]      record  row in MySQL format */
void ha_innobase::position(const uchar *record) {
  uint len;

  DBUG_TRACE;
  assert(m_prebuilt->trx == thd_to_trx(ha_thd()));
  assert(table_share->is_missing_primary_key() ==
         (bool)m_prebuilt->clust_index_was_generated);   // :11254 无 PK ⟺ 自建聚簇索引

  if (m_prebuilt->clust_index_was_generated) {           // :11257 ★ 判据不是 dict_index_is_clust
    /* No primary key was defined for the table and we
    generated the clustered index from row id: the
    row reference will be the row id, not any key value
    that MySQL knows of */
    len = DATA_ROW_ID_LEN;                               // :11263 = 6
    memcpy(ref, m_prebuilt->row_id, len);                // :11265 ★ 从 prebuilt 缓存拷，不碰 record！
  } else {
    /* Copy primary key as the row reference */
    KEY *key_info = table->key_info + table_share->primary_key;      // :11268
    key_copy(ref, (uchar *)record, key_info, key_info->key_length);  // :11269 整键拷贝（含 NULL 标志位）
    len = key_info->key_length;                          // :11270
  }

  /* We assume that the 'ref' value len is always fixed for the same table. */
  if (len != ref_length) {                               // :11276 一致性兜底（不一致就打错误日志）
    log_errlog(ERROR_LEVEL, ER_INNODB_DIFF_IN_REF_LEN, (ulong)len, (ulong)ref_length);
  }
}
```

★★ **最容易被忽略的一点**：无主键表下 `record` 参数**根本没用**——MySQL 行格式里没有 row id 列，只能靠**预缓存**（函数头注释自己都说这是 "illogical" 的）。两个配套机制：

- **`prebuilt->row_id` 的刷新**：`row_sel_store_row_id_to_prebuilt()`（`row0sel.cc:2451-2481`）在**每次取到行时**从聚簇记录的 `DATA_ROW_ID` 系统列抠出（`rec_get_nth_field`），触发条件只有 `clust_index_was_generated`（`row0sel.cc:4162-4166`），**与 `read_just_key` 无关**。
- **判据与 `ref_length` 同源**：`clust_index_was_generated` 由 open 时的 `row_table_got_default_clust_index()`（`row0mysql.cc:2696-2704`）设置，定义是"**聚簇索引第 0 列类型为 `DATA_SYS`**"；同一串判定定死 `ref_length`：有 PK 时 = `table->key_info[primary_key].key_length`，无 PK 时 = `DATA_ROW_ID_LEN`（`ha_innodb.cc:7512/7532`）。★ `position()` 里**没有**任何 `read_just_key` 分支——`HA_EXTRA_KEYREAD` 只影响 `build_template` 的建项，不影响 ref 构造。

### position() 何时调 key_copy（`sql/key.cc`）

`position()` 的实质就是调一次 **`key_copy`**：把 record[0] 里主键各列的值，按引擎键格式拼进 `ref` 缓冲。**键格式本身的逐字节剖析不在这里**——完整源码（`key_copy` 47 行 + `key_restore` 72 行）与键格式约定（NULL 标记 1 字节、变长段恒 2 字节小端长度头、定长段空格填充、`get_key_image` 各类型差异）在 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md) 的「键写（server 侧）」节，此处不重复。

本层只需记住**调用时机**与**调用者**：

- `position()`（`ha_innodb.cc` / `ha_innopart.cc`）——filesort rowid 模式、`ref` 访问、`handler::position` 协议；
- `get_auto_increment`（`sql/handler.cc:4016-4025`，传 `next_number_key_offset` 以便生成"带前缀"的自增键）；
- 去重迭代器（`sql/iterators/composite_iterators.cc`）、松散索引扫描（`group_index_skip_scan.cc`）、DD 原子键（`dd/impl/raw/object_keys.cc`）、权限/复制系统表（`sql/auth/*`、`rpl_write_set_handler.cc`）等。

> ⚠️ **两处旧说法要纠正**：① **不是"二级索引插入都调 `key_copy`"**——InnoDB 内 `key_copy` 只在 `position()` 里被调（`ha_innodb.cc:11269`、`ha_innopart.cc`），二级索引项是引擎侧 `row_build_index_entry*` 从聚簇 dtuple 直接建的，**根本不经过 MySQL 键格式**；② `row_sel_convert_mysql_key_to_innobase` 的逐行剖析在 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md)「键的读方向」节（本篇下面 `rnd_pos` 只讲链路）。

#### key_restore：反向——把键还原成 record

`key_copy` 是 record → 键，**`key_restore` 是键 → record**——完整源码与逐行注释见 [`../infra/structure/data_repr.md`](../infra/structure/data_repr.md)「键写（server 侧）」节。真实调用点只有三处（**全在 server 层，`storage/innobase` 内 0 处**）：**分区裁剪**（`sql/sql_partition.cc`）、**松散索引扫描**（`group_index_skip_scan.cc`）、**权限表读取**（`sql/auth/sql_user_table.cc`）。

> ⚠️ **纠正一个流传很广的说法**：`key_restore` **不是"覆盖索引不回表"的实现**。覆盖索引靠的是 `m_prebuilt->read_just_key`（`ha_innodb.cc`）——只取索引列、不回表；`storage/innobase` 内对 `key_restore` **0 处调用**。原注释 "return a key as a result row" 指的是把索引键当作**结果行**输出（用于上述三处场景），不是"索引覆盖查询"。

### rnd_pos 回表（`ha_innodb.cc`）

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

无主键时，`row_sel_convert_mysql_key_to_innobase` 有专门分支（`row0sel.cc`）：跳过 MySQL key 解析，直接把 6 字节当 `DB_ROW_ID` 塞进 search_tuple 第一列。

**完整实现（`ha_innobase.cc:10859-10889`）**：

```cpp
int ha_innobase::rnd_pos(uchar *buf, uchar *pos) {
  DBUG_TRACE;
  DBUG_DUMP("key", pos, ref_length);                   // :10871

  ha_statistic_increment(&System_status_var::ha_read_rnd_count);

  ut_a(m_prebuilt->trx == thd_to_trx(ha_thd()));

  /* Note that we assume the length of the row reference is fixed
  for the table, and it is == ref_length */
  int error = index_read(buf, pos, ref_length, HA_READ_KEY_EXACT);   // :10880 ★ 一次精确查找

  if (error != 0) {
    DBUG_PRINT("error", ("Got error: %d", error));
  } else {
    m_start_of_scan = false;
  }
  return error;
}
```

**`HA_READ_KEY_EXACT` 在 InnoDB 侧是两层翻译**：

```cpp
// :10087-10092 第一层：定位模式
case HA_READ_KEY_EXACT:
  /* this does not require the index to be UNIQUE */
case HA_READ_KEY_OR_NEXT:
  return (PAGE_CUR_GE);                 // 定位到 >= key 的第一条

// :10272-10277 第二层：命中后仍要逐列比对
if (find_flag == HA_READ_KEY_EXACT) {
  match_mode = ROW_SEL_EXACT;           // 保证结果"等于"而非"大于等于"
}
```

★ 所以 ref 能**唯一定位一行**：有 PK 时是 PK 全列、无 PK 时是 6 字节 row id（唯一）。

**无主键表的 `DATA_SYS` 特例（完整代码）**：

```cpp
// row0sel.cc:2286-2297
if (UNIV_UNLIKELY(dfield_get_type(dfield)->mtype == DATA_SYS)) {
  /* A special case: we are looking for a position in the
  generated clustered index which InnoDB automatically added
  to a table with no primary key: the first and the only
  ordering column is ROW_ID which InnoDB stored to the key_ptr
  buffer. */
  ut_a(key_len == DATA_ROW_ID_LEN);                     // :2293 长度必须 6
  dfield_set_data(dfield, key_ptr, DATA_ROW_ID_LEN);    // :2295 6 字节原样作为 ROW_ID 搜索键
  dtuple_set_n_fields(tuple, 1);                        // :2297 单字段元组
  return;
}
```

★ 搜索元组的类型是从索引复制来的（`dict_index_copy_types`），自建聚簇索引第 0 列类型为 `DATA_SYS` → 走直拷分支。**`DATA_ROW_ID` 是隐藏系统列、不出现在 MySQL 行格式里**——这与 `position()` 必须预缓存 `row_id` 是同一件事的两面。**ref 的比较也特判**（`:19912-19915`）：`cmp_ref()` 在 `clust_index_was_generated` 时直接 `memcmp(ref1, ref2, DATA_ROW_ID_LEN)`。

> ★ 一个不对称之处：`rnd_pos` **不**断言 `clust_index_was_generated`（只依赖 `ref_length`）——若 `position()` 写 ref 的路径与 open 时定下的 `ref_length` 不一致，靠 `position()` 的 `:11276` 打 `ER_INNODB_DIFF_IN_REF_LEN` 错误日志（历史上"ref 长度写错"类 bug 都以它为线索）。

### rowid 全链路闭环

**关键澄清：`ref` 不是扫描时自动填的**——filesort 在 rowid 模式下，每读到一行后**显式调 `position()`**（`filesort.cc`）：

```
(1) 扫描：ha_rnd_next/ha_index_next → InnoDB row_search_mvcc 返回行 → 行落到 record[0]
(2) 填 ref：filesort.cc  table->file->position(record[0])   ← 显式，非自动
              ├ 有主键：key_copy → ref = 主键引擎键格式
              └ 无主键：memcpy → ref = 6 字节 DB_ROW_ID
(3) 进排序记录：filesort.cc  memcpy(to, table->file->ref, ref_length)
(4) 排序：ref 作为 sort key 之后的 payload，随行移动
(5) 回表：SortBufferIndirectIterator::Read → ha_rnd_pos(record[0], ref)
(6) 定位：rnd_pos → index_read → row_search_mvcc → 原始行重新定位
```

前置：rowid 模式先 `prepare_for_position()`（把主键列加进 `read_set`，否则 `position` 拷不到主键值）。

### ⚠ 与 EXPLAIN 的 ref 区分

EXPLAIN 输出里的 `ref` 列（访问类型 `const`/`ref`/`eq_ref`）是**另一个概念**——优化器说的"用什么索引/常量匹配行"，与 handler 的 `ref` 缓冲区（行位置）**无关**，勿混淆。

### 谁用它

- **filesort 的 indirect 模式**：排序结果只存 `ref`，读结果时 `ha_rnd_pos()` 回表（见 [query/runtime/01_filesort.md](query/runtime/01_filesort.md) 1.3）
- **DML 两阶段读**：`position()` 记行 ID → `rnd_pos()` 回表改（见 [query/10_dml.md](query/10_dml.md)）
- **index merge 的 ROR**：`IndexRangeScanIterator` 按 rowid 输出，靠 `position()` 填 `file->ref`
- **WeedoutIterator**：semijoin 去重的键就是外层表的 row ID

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → Alternative Storage Engines*（引擎插件架构）
- *MySQL Internals Manual → The Handler Interface*

**相关文档**
- 上游（SQL 层如何调用）见 [query/09_executor_iterator.md](query/09_executor_iterator.md)
- 下游（InnoDB 如何实现）见 [innodb/row_search.md](../innodb/row_search.md)
