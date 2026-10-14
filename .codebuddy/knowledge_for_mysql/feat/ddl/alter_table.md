# ★ ALTER TABLE 深度剖析（server ↔ InnoDB 跨层全链路）

> 基于 MySQL 8.0.39 源码，剖析 ALTER TABLE 的 **server 侧编排主线**：`Sql_cmd_alter_table::execute` → `mysql_alter_table` → `Alter_info`（意图描述）/ `Alter_table_ctx`（执行期上下文）/ `mysql_prepare_alter_table`（意图 → 具体动作）→ **算法决策与 fail-fast** → INPLACE 或 COPY 分支 → `mysql_recreate_table`。
>
> **边界**：ALTER 是最复杂的 DDL，本篇只保留**编排主线与两层接口骨架**；各算法的深水区已独立成篇，不在此重复：
>
> | 内容 | 去哪看 |
> |---|---|
> | COPY 全链路（`copy_data_between_tables` 逐段、为何一定不 Online） | [`copy_ddl.md`](copy_ddl.md) |
> | INPLACE 全链路（MDL 升降级、引擎四件套、**row log 为什么 apply 两次**） | [`inplace_ddl.md`](inplace_ddl.md) |
> | INSTANT（行版本号三层级、WL#11250 两个坑） | [`instant_ddl.md`](instant_ddl.md) |
> | 8.0 新版 DDL 引擎与并行构建（Loader/Builder/Btree_load） | [`ddl_engine.md`](ddl_engine.md) |
> | 原子 DDL（DDL log、post-DDL replay、崩溃恢复） | [`atomic_ddl.md`](atomic_ddl.md) |
> | 三种算法的选择策略与 MDL 三段式骨架 | [`ddl.md`](ddl.md) |
>
> CREATE TABLE 的完整链路（含引擎侧建表空间 / 建 B-tree 根页）见 [`create_table.md`](create_table.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [★ ALTER TABLE 全链路](#-alter-table-全链路server-侧--引擎侧)
  - [完整调用链](#完整调用链)
  - [`Alter_info`：ALTER 的"意图描述"](#alter_infoalter-的意图描述)
  - [`Alter_table_ctx`：执行期上下文](#alter_table_ctx执行期上下文)
  - [`mysql_prepare_alter_table()`：翻译"意图 → 具体动作"](#mysql_prepare_alter_table翻译意图--具体动作)
  - [★ 算法决策：为什么会落到 INPLACE 或 COPY（含 fail-fast）](#算法决策inplace-与-copy-的判定含-fail-fast)
  - [INPLACE vs COPY 的 server 侧差异](#inplace-vs-copy-的-server-侧差异)
  - [阶段一：`open_tables` 与外键 MDL 大集合](#阶段一open_tables-与外键-mdl-集合的作用范围)
  - [阶段二：`mysql_prepare_alter_table` 之后，新表定义怎么落地](#阶段二新表定义的构建)
  - [`mysql_inplace_alter_table()`：INPLACE 的 server 侧执行主体（全流程）](#mysql_inplace_alter_tableinplace-的-server-侧执行主体全流程)
  - [收尾：binlog / 提交 / 回滚与失败清理](#收尾binlog--提交--回滚与失败清理)
  - [`mysql_recreate_table()`：ALTER 的薄包装](#mysql_recreate_tablealter-的薄包装)
  - [`notify_alter_table`：MDL 升 X 太晚的补救钩子](#notify_alter_tablemdl-升-x-太晚的补救钩子)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

ALTER TABLE 的 server 侧编排：把用户的 ALTER 语句解析成**意图**（`Alter_info`），翻译成**具体动作**（`mysql_prepare_alter_table`），**选定执行算法**，然后分流给 INPLACE 或 COPY 两条路。

### 用途

本篇覆盖的是**三种算法共用的公共前段**——从语句解析到算法分岔口为止。它不对应某个具体算法（COPY/INPLACE/INSTANT 各有自己的篇），分岔之后的内容交给各篇。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 之前 | ALTER 基本等价于"建新表 + 拷数据 + 换名"（即后来的 COPY 算法），server 层无算法选择 |
| 5.6 | **新 ALTER API**（WL#5534）引入 `prepare/inplace/commit_inplace_alter_table` 三段式，配合 online DDL（WL#5526），ALTER 首次有了"引擎内执行"这条路 |
| 8.0 | 原子 DDL：ALTER 纳入 Prepare / Perform / Commit / Post-DDL 四阶段（见 [`atomic_ddl.md`](atomic_ddl.md)）；`ALGORITHM=INSTANT` 加入（8.0.12+，8.0.29 支持 DROP COLUMN） |
| 8.0（新版） | 重建类 ALTER 的构建方式换成新 DDL 引擎（`storage/innobase/ddl/`），8.0.27+ 支持并行；server 侧编排骨架未变 |

---

## 理论基础

### 设计思想与权衡

**为什么要有"意图 → 动作"两层描述**：`Alter_info` 表达的是**用户要什么**（加一列、改类型、重建），`Alter_inplace_info` 表达的是**引擎要做什么**（哪些索引要建、哪些要删、要不要重建）。两者解耦，使 parser 的语义不必下推到引擎，引擎也不必理解 SQL。

**为什么算法决策放在 server 而不是引擎**：server 持有 DDL 的完整语义（能不能只改元数据、要不要拷数据），引擎只回答"我能不能 INPLACE、需要什么锁"（`check_if_supported_inplace_alter` 的六档返回值）。所以"选算法"是 server 的事，引擎只有否决权。

**为什么显式指定 `ALGORITHM` 做不到就直接报错**：这是**声明式约束**而非开关——目的是防止用户以为走 INPLACE，实际被静默降级成 COPY 把表锁死。做不到即 fail-fast（官方文档口径，源码见下方决策点）。

**代价与保守性**：server 的等价性判定是**保守**的——`IS_EQUAL_*` 只有三档、InnoDB 连 `PACK_LENGTH` 差异都不接受（见 [`ddl.md`](ddl.md)「算法判定的真正链条」）。结果是很多理论上可 INPLACE 的操作落到 COPY，这是"宁可慢、不可错"的取舍。

**简化假设的失效场景**：判定链条假设"引擎的返回值可信"。若引擎返回了与实际不符的档位（如声称 NO_LOCK 但实际需要追平），后果是并发 DML 丢数据——这也是 [`inplace_ddl.md`](inplace_ddl.md) 强调六档语义约束的原因。

### 算法与数据结构

| 结构 | 角色 | 说明 |
|---|---|---|
| `Alter_info` | 意图描述 | 由 parser 填：请求标志位（`Alter_info::ALTER_*`）+ `request` 列表（Drop/Add/Change 等）+ `requested_algorithm` / `requested_lock` |
| `Alter_table_ctx` | 执行期上下文 | 构造时算出新旧库表名、临时名（`#sql-` 系列）、是否需要 rename、涉及的表空间；供后续各阶段共享 |
| `Alter_inplace_info` | 引擎动作描述 | 传给引擎四件套：`create_info` + `alter_info` + 要建/删的索引数组 + `online` 标志 |

---

## ★ ALTER TABLE 全链路（server 侧 + 引擎侧）


### 完整调用链

```
Sql_cmd_alter_table::execute()                       sql/sql_alter.cc
  └─ mysql_alter_table()                             sql/sql_table.cc
       ├─ 前置拒绝：log 表 / START TRANSACTION / WITH VALIDATION 误用 /
       │            ALGORITHM=INSTANT + LOCK=xxx / NDF 默认值与 binlog 格式
       ├─ open_tables() + Alter_table_prelocking_strategy
       ├─ handle_drop_functional_index() / handle_rename_functional_index()
       ├─ lock_trigger_names() / tablespace 名校验
       ├─ Alter_table_ctx alter_ctx(thd, table_list, tables_opened, new_db, new_name);
       ├─ dd::Schema_MDL_locker ×2 + acquire(old_table_def)
       ├─ 改名时：target_mdl_request / target_db_mdl_request 拿 X / IX
       ├─ 外键 MDL 大集合（collect_fk_parents_for_new_fks / _for_all_fks /
       │                   collect_fk_children / collect_fk_names_for_dropped_fks）
       ├─ if (is_simple_rename_or_index_change()) → simple_rename_or_index_change()  ← 快路径
       ├─ prep_alter_part_table()
       ├─ prepare_check_constraints_for_alter()
       ├─ mysql_prepare_alter_table()   → prepare_fields_and_keys()   ★
       ├─ check_primary_key_alter_restrictions() / is_alter_geometry_column_valid()
       ├─ 算法决策：old_alter_table / is_inplace_alter_impossible() / partition_changed
       ├─ ALTER TABLE ... ENGINE=同引擎 → flags |= ALTER_RECREATE
       ├─ create_table_impl(..., internal_tmp_table=true, no_ha_table=true,
       │                    do_not_store_in_dd=(新引擎支持原子 DDL), ...)
       │     → 得到"新表版本"的 dd::Table（可能只在内存里）+ key_info/fk_key_info
       ├─ if (requested_algorithm != COPY):
       │     Alter_inplace_info + fill_alter_inplace_info()
       │     open_table_uncached() → altered_table
       │     if (ha_alter_info.handler_flags == 0) → is_noop, goto end_inplace_noop
       │     check_if_supported_inplace_alter()      ← 引擎决策（见 4.6）
       │     if (use_inplace) { mysql_inplace_alter_table(); goto end_inplace; }
       └─ /* todo: ALTER TABLE using copy algorithm. */   ← COPY 路径就在本函数体内
             fk_check_copy_alter_table() → MDL 升级 → FK 名锁 →
             ha_create_table(tmp_path) → open_table_uncached →
             copy_data_between_tables() → 备份改名 → 换表 → quick_rm_table(备份) →
             binlog → commit → post_ddl → reopen
```

`Sql_cmd_alter_table::execute()` 本身只做四件事：权限（`ALTER_ACL`；`DROP PARTITION`/`RENAME TO` 额外要 `DROP_ACL`）、复制 `HA_CREATE_INFO`/`Alter_info`、忽略 DATA/INDEX DIRECTORY 并告警、开 slow log。

> 顺带：`CREATE INDEX` / `DROP INDEX` 也是转成 `mysql_alter_table()`：
> ```cpp
> /*
>   CREATE INDEX and DROP INDEX are implemented by calling ALTER
>   TABLE with proper arguments.
> */
> ```

### 前置拒绝：`mysql_alter_table` 开头这段决定"哪些 ALTER 根本不会开始"

`mysql_alter_table` 是 DDL 里最大的函数（**2098 行**，sql_table.cc:16295-18392）。它的开头约 100 行是一串**前置拒绝**——这些检查不通过，DDL 连表都不打开就失败：

```cpp
// ① 日志表（mysql.slow_log / mysql.general_log）
enum_log_table_type table_kind =
    query_logger.check_if_log_table(table_list, false);
if (table_kind != QUERY_LOG_NONE) {
  if (query_logger.is_log_table_enabled(table_kind)) {
    my_error(ER_BAD_LOG_STATEMENT, MYF(0), "ALTER");      // 日志开启时禁止 ALTER
    return true;
  }
  if ((create_info->used_fields & HA_CREATE_USED_ENGINE) &&
      (!create_info->db_type ||
       !(create_info->db_type->flags & HTON_SUPPORT_LOG_TABLES))) {
    my_error(ER_UNSUPORTED_LOG_ENGINE, MYF(0));           // 改成不支持日志表的引擎
    return true;
  }
  if (alter_info->flags & Alter_info::ALTER_PARTITION) {
    my_error(ER_WRONG_USAGE, MYF(0), "PARTITION", "log table");
    return true;
  }
}

// ② ALTER 不能带 START TRANSACTION
if (create_info->m_transactional_ddl) {
  my_error(ER_NOT_ALLOWED_WITH_START_TRANSACTION, MYF(0),
           "with ALTER TABLE command.");
  return true;
}

// ③ WITH VALIDATION 只能用于 ADD / CHANGE COLUMN
if (alter_info->with_validation != Alter_info::ALTER_VALIDATION_DEFAULT &&
    !(alter_info->flags &
      (Alter_info::ALTER_ADD_COLUMN | Alter_info::ALTER_CHANGE_COLUMN))) {
  my_error(ER_WRONG_USAGE, MYF(0), "ALTER", "WITH VALIDATION");
  return true;
}

// ④ ALGORITHM=INSTANT 不能配 LOCK 子句
if (alter_info->requested_algorithm ==
        Alter_info::ALTER_TABLE_ALGORITHM_INSTANT &&
    alter_info->requested_lock != Alter_info::ALTER_TABLE_LOCK_DEFAULT) {
  my_error(ER_WRONG_USAGE, MYF(0), "ALGORITHM=INSTANT",
           "LOCK=NONE/SHARED/EXCLUSIVE");
  return true;
}
```

同段还有两条容易忽略的：

- **NDF 默认值 `DEFAULT (func())` 与 binlog 格式**：statement 格式下只**警告并继续**（`push_warning(ER_BINLOG_UNSAFE_SYSTEM_FUNCTION)`），mixed / row 格式下**直接报 `ER_BINLOG_UNSAFE_SYSTEM_FUNCTION` 失败**。这条与直觉相反——通常认为 row 更安全，但这里因为要记录表达式的求值结果才判为 unsafe。
- **② 是"原子 DDL ≠ 事务性 DDL"的源码体现**：`m_transactional_ddl`（`START TRANSACTION` 语义）与 ALTER 直接互斥。原子说的是这条 DDL 自身全成或全败，不是说它能被包进用户事务。

> 源码里还给 ① 留了注释：`TODO: this design is obsolete and will be removed.`——日志表的特殊处理是历史遗留。

### `Alter_info`：ALTER 的"意图描述"

★ 源码自己给的定位注释（说明它只是**解析层意图**，不是执行层动作）：

```cpp
/*
  These flags are set by the parser and describes the type of
  operation(s) specified by the ALTER TABLE statement.

  They do *not* describe the type operation(s) to be executed
  by the storage engine. For example, we don't yet know the
  type of index to be added/dropped.
*/
enum Alter_info_flag : ulonglong { ... };
```

**容器成员**（全是"待办清单"）：

| 成员 | 职责 |
|---|---|
| `drop_list` | DROP COLUMN / KEY / FOREIGN KEY / CHECK CONSTRAINT |
| `alter_list` | SET DEFAULT / DROP DEFAULT / RENAME COLUMN / 列可见性 |
| `key_list` | **CREATE 和 ALTER 共用**的键定义（含 FK） |
| `alter_rename_key_list` | RENAME INDEX |
| `alter_index_visibility_list` | 索引 VISIBLE/INVISIBLE |
| `alter_constraint_enforcement_list` | ALTER CHECK ... ENFORCED |
| `create_list` | **CREATE 和 ALTER 共用**的列定义；ALTER 时 `change` 非空的项表示"改某列" |
| `cf_appliers` | 延迟应用到 `Create_field` 的回调 |

**标量成员**：`flags`（41 个 `Alter_info_flag` 位）、`keys_onoff`、`partition_names`/`num_parts`、`requested_algorithm`、`requested_lock`、`with_validation`、`new_db_name`/`new_table_name`。

标志位本身（摘几个有代表性的）：

```cpp
enum Alter_info_flag : ulonglong {
  /// Set for ADD [COLUMN]
  ALTER_ADD_COLUMN = 1ULL << 0,
  /// Set for DROP [COLUMN]
  ALTER_DROP_COLUMN = 1ULL << 1,
  /// Set for CHANGE [COLUMN] | MODIFY [CHANGE]
  /// Set by mysql_recreate_table()
  ALTER_CHANGE_COLUMN = 1ULL << 2,
  /// Set for ADD INDEX | ADD KEY | ADD PRIMARY KEY | ADD UNIQUE KEY | ...
  ALTER_ADD_INDEX = 1ULL << 3,
  /// Set for DROP PRIMARY KEY | DROP FOREIGN KEY | DROP KEY | DROP INDEX
  ALTER_DROP_INDEX = 1ULL << 4,
  /// Set for RENAME [TO]
  ALTER_RENAME = 1ULL << 5,
  /// Set for ORDER BY                       ← ★ 被一票否决的那一位
  ALTER_ORDER = 1ULL << 6,
  /// Set for table_options
  ALTER_OPTIONS = 1ULL << 7,
  ...
  /// Set for FORCE
  /// Set for ENGINE(same engine)
  /// Set by mysql_recreate_table()
  ALTER_RECREATE = 1ULL << 10,
  ...
  /// Set for ADD FOREIGN KEY
  ADD_FOREIGN_KEY = 1ULL << 21,
  /// Set for DROP FOREIGN KEY
  DROP_FOREIGN_KEY = 1ULL << 22,
};
```

★ 两个细节：① `ALTER_ORDER = 1ULL << 6` 就是「算法决策」节里 `is_inplace_alter_impossible` 一票否决的那一位——server 侧判定直接读这些 parser 位；② `ALTER_CHANGE_COLUMN` 与 `ALTER_RECREATE` 的注释都写着 *"Set by mysql_recreate_table()"*——**并非所有标志位都来自 parser**，有些是 server 内部改写时设的（`OPTIMIZE TABLE` / `ENGINE=同引擎` 会被改成 `ALTER TABLE ... FORCE`，见 `mysql_recreate_table`）。

> ⚠️ **无 `alter_recreate_kind` 这个成员**（源码无直接证据）；语义上最接近的是 `flags` 里的 `ALTER_RECREATE`。

### `Alter_table_ctx`：执行期上下文

定位：`/** Runtime context for ALTER TABLE. */`。它是"这次 ALTER 的原名/新名/临时名/路径/外键快照"的容器。

| 成员 | 职责 |
|---|---|
| `db`/`table_name`/`alias` | 原库/原表/原别名 |
| `new_db`/`new_name`/`new_alias` | 新库/新表/新别名 |
| `is_table_renamed()` 等 | ★ **靠指针比较**而非 strcmp（注释："we want to see if it's a RENAME later just by comparing the pointers, avoiding the need for strcmp"） |
| `tmp_name[80]` | 临时表名：`snprintf(tmp_name, ..., "%s-%lx_%x", tmp_file_prefix, current_pid, thd->thread_id())` → 即 **`#sql-<pid>_<tid>`**；`lower_case_table_names` 时转小写（注释 `/* Safety fix for InnoDB */`） |
| `path`/`new_path`/`tmp_path` | 原表路径 / 新表路径 / 临时表路径 |
| `datetime_field` + `error_if_not_empty` | 表非空时新增 NOT NULL DATE/DATETIME/GEOMETRY 要报错的位掩码 |
| `fk_info`/`fk_count` | **旧表已有外键的快照**，供新表版本复用 |
| `target_mdl_request`/`target_db_mdl_request` | 新表名 X 锁、新库 IX 锁 |

> ⚠️ **它没有任何 `.frm` 成员**——8.0 已无 `.frm`，对应的是 `dd::Table` + SDI；缓存失效由 `tdc_remove_table()` / `close_all_tables_for_name()` 完成。

```cpp
class Alter_table_ctx {
 public:
  /**
     @return true if the table is moved to another database, false otherwise.
  */
  bool is_database_changed() const { return (new_db != db); }

  /**
     @return true if the table name is changed, false otherwise.
  */
  bool is_table_name_changed() const { return (new_name != table_name); }

  /**
     @return true if the table is renamed (i.e. its name or database changed),
             false otherwise.
  */
  bool is_table_renamed() const {
    return is_database_changed() || is_table_name_changed();
  }

 public:
  typedef uint error_if_not_empty_mask;
  static const error_if_not_empty_mask DATETIME_WITHOUT_DEFAULT = 1 << 0;
  static const error_if_not_empty_mask GEOMETRY_WITHOUT_DEFAULT = 1 << 1;

  Create_field *datetime_field;
  error_if_not_empty_mask error_if_not_empty;   // ★ 表非空时的"不能为空"位掩码
  const char *db;
  const char *table_name;
  const char *new_db;
  const char *new_name;
  char tmp_name[80];            // 临时表名：#sql-<pid>_<tid>

  /* Used to remember which foreign keys already existed in the table. */
  FOREIGN_KEY *fk_info;         // ★ 旧表外键快照，供新表版本复用
  uint fk_count;
  uint fk_max_generated_name_number;

  /**
    Metadata lock request on table's new name when this name or database
    are changed.
  */
  MDL_request target_mdl_request;
  ...
};
```

★ `error_if_not_empty` 这个位掩码值得单独记：它记录"表非空就不能做"的情形（新增 NOT NULL 的 DATETIME / GEOMETRY 列）。它在 [`instant_ddl.md`](instant_ddl.md) 里还有第二个落点——引擎判定 INSTANT 时，`else if (ha_alter_info->error_if_not_empty) break;` 会**因此放弃 INSTANT 退回 INPLACE**（因为要检查数据就得拷表）。同一个字段，server 与引擎两处消费。

### `mysql_prepare_alter_table()`：翻译"意图 → 具体动作"

它的 doc 注释自己说明了会**改写入参**：

```
  Prepares alter_info->create_list and alter_info->key_list with
  columns and keys of the new table.
```

分步：

```cpp
bool mysql_prepare_alter_table(THD *thd, const dd::Table *src_table,
                               TABLE *table, HA_CREATE_INFO *create_info,
                               Alter_info *alter_info,
                               Alter_table_ctx *alter_ctx) {
  uint64_t used_fields = create_info->used_fields;

  // ① 从 TABLE_SHARE 继承未显式指定的表选项
  create_info->init_create_options_from_share(table->s, used_fields);

  // ② 没显式指定 AUTO_INCREMENT 但有自增列 → 取当前值
  if (!(used_fields & HA_CREATE_USED_AUTO) && table->found_next_number_field) {
    /* Table has an autoincrement, copy value to new table */
    table->file->info(HA_STATUS_AUTO);
    create_info->auto_increment_value = table->file->stats.auto_increment_value;
  }

  // ③ ★ 主体：把 alter_info 的"待办清单"翻译成 create_list / key_list
  if (prepare_fields_and_keys(thd, src_table, table, create_info, alter_info,
                              alter_ctx, used_fields))
    return true;

  table->file->update_create_info(create_info);   // ④ 引擎回填

  /* Get the autoextend_size value for the old table if the user did not
  specify it on the command line */
  if (src_table && src_table->engine() == "InnoDB") {
    ulonglong autoextend_size{};
    dd::get_implicit_tablespace_options(thd, src_table, &autoextend_size);
    if (!create_info->m_implicit_tablespace_autoextend_size_change) {
      create_info->m_implicit_tablespace_autoextend_size = autoextend_size;
    }
  }

  /* NDB storage engine misuses handler::update_create_info() to implement
  special handling of table comments which are used to specify and store
  some NDB-specific table options. In the process it might report error.
  In order to detect and correctly handle such situation we need to call
  THD::is_error(). */
  if (thd->is_error()) return true;               // ★ 引擎可能在 ④ 里报错

  // ⑤ 表选项位合并：先清掉旧位，再让新值并入
  if ((create_info->table_options &
       (HA_OPTION_PACK_KEYS | HA_OPTION_NO_PACK_KEYS)) ||
      (used_fields & HA_CREATE_USED_PACK_KEYS))
    db_create_options &= ~(HA_OPTION_PACK_KEYS | HA_OPTION_NO_PACK_KEYS);
  ...
```

对应阅读要点：

1. ① 就是"ALTER 不写 `ROW_FORMAT` 就保留原值"的实现——**未指定的选项从 `TABLE_SHARE` 继承**，而不是取默认值。
2. ② 自增必须显式搬运（注释 "copy value to new table"）：否则重建后自增值从 1 开始，后续插入撞重复键。
3. ③ `prepare_fields_and_keys()` 是真正的翻译器（见下）。
4. ④ `update_create_info` 是引擎的回填机会——注释特意指出 **NDB 拿它解析表注释并可能报错**，所以紧接着要查 `thd->is_error()`。
5. ⑤ 位合并的通用模式是"先 `&= ~` 清旧位，再并入新值"——同样是**显式指定才覆盖，否则继承**。

#### `prepare_fields_and_keys()`

★ 一个易忽略但重要的细节：**它对四个 list 全部先拷一份再改**：

```cpp
/*
  Original Alter_info::drop_list is used by foreign key handling code and
  storage engines. check_if_field_used_by_generated_column_or_default()
  also needs original Alter_info::drop_list. So this function should not
  modify original list but rather work with its copy.
*/
```

列侧的关键检查：

- 命中 drop 时：若该列被生成列/默认值依赖、或用于分区函数 → 不允许
- 命中 CHANGE/MODIFY：**不允许改 STORED 属性**（`ER_UNSUPPORTED_ACTION_ON_GENERATED_COLUMN, "Changing the STORED status"`）
- 新增 DATE/DATETIME NOT NULL 且无默认 → 置 `DATETIME_WITHOUT_DEFAULT`（原文：*"raise the error_if_not_empty flag to allow ALTER TABLE only if the table to be altered is empty"*）
- 函数索引的隐藏生成列**按列名排序追加到末尾**，还要考虑主备版本兼容

键侧有个**单位换算**陷阱（源码给了具体例子）：

```
  In case of TEXTs we check the data type maximum length *in bytes*
  to key part length measured *in characters* (i.e. key_part_length
  divided to mbmaxlen). This is because it's OK to have:
  CREATE TABLE t1 (a tinytext, key(a(254)) character set utf8);
  In case of this example:
  - data type maximum length is 255.
  - key_part_length is 1016 (=254*4, where 4 is mbmaxlen)
```

**server 侧的硬否决表** `is_inplace_alter_impossible()`：

```cpp
/* At the moment we can't handle altering temporary tables without a copy. */
if (table->s->tmp_table) return true;
if (alter_info->flags & (Alter_info::ALTER_ORDER | Alter_info::ALTER_KEYS_ONOFF)) return true;
/*
  Check constraints are evaluated in the server, if any check constraint
  (re-)evaluation is required then it can't be added/enforced inplace.
*/
if (is_any_check_constraints_evaluation_required(alter_info)) return true;
if (create_info->db_type != table->s->db_type()) return true;
if (!table->s->mysql_version) return true;   // 老 frm 兼容
```

真正的 INPLACE/INSTANT 能力由引擎 `check_if_supported_inplace_alter()` 回答（映射见 4.6）。

### 算法决策：INPLACE 与 COPY 的判定（含 fail-fast）

server 的判定不是"猜"，而是**三个条件的或**（`mysql_alter_table`）：

```cpp
/*
  - old_alter_table system variable is set without in-place requested using
    the ALGORITHM clause.
  - Or if in-place is impossible for given operation.
  - Changes to partitioning needs to be handled using table copying
    algorithm unless the engine supports partitioning changes using
    in-place API (because it supports auto-partitioning or simply
    can do partitioning changes using in-place using mark-up in
    partition_info object).
*/
if ((thd->variables.old_alter_table &&
     alter_info->requested_algorithm !=
         Alter_info::ALTER_TABLE_ALGORITHM_INPLACE &&
     alter_info->requested_algorithm !=
         Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) ||
    is_inplace_alter_impossible(table, create_info, alter_info) ||
    (partition_changed &&
     !(table->s->db_type()->partition_flags() & HA_USE_AUTO_PARTITION) &&
     !new_part_info)) {
  // ★ 落到 COPY 分支
  if (alter_info->requested_algorithm ==
      Alter_info::ALTER_TABLE_ALGORITHM_INPLACE) {
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), "ALGORITHM=INPLACE",
             "ALGORITHM=COPY");
    return true;          // ★ fail-fast：做不到就报错，绝不静默降级
  }
  if (alter_info->requested_algorithm ==
      Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) {
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), "ALGORITHM=INSTANT", ...);
```

三个条件任一成立即走 COPY：

1. **`old_alter_table` 开关**（且未显式指定 INPLACE/INSTANT）——用于强制对比测试；
2. **`is_inplace_alter_impossible()` 判定不可 INPLACE**——server 侧的一票否决清单；
3. **分区变更**，且引擎不支持 `HA_USE_AUTO_PARTITION`（如 InnoDB 原生分区外的情形）。

★ **fail-fast**：若用户显式写了 `ALGORITHM=INPLACE`/`INSTANT` 却落到 COPY 分支，**直接报 `ER_ALTER_OPERATION_NOT_SUPPORTED` 而不降级**。这就是"ALGORITHM 子句是声明式约束而非开关"在源码里的落点——防止用户以为走 INPLACE 实际被静默降级成 COPY 把表锁死。

`is_inplace_alter_impossible` 里 server 侧一票否决的典型情形：

```cpp
static bool is_inplace_alter_impossible(TABLE *table,
                                        HA_CREATE_INFO *create_info,
                                        const Alter_info *alter_info) {
  /* At the moment we can't handle altering temporary tables without a copy. */
  if (table->s->tmp_table) return true;          // ① 临时表：一律 COPY

  /*
    ALTER TABLE tbl_name ORDER BY ... 总是走 copy（现有 API 无法把排序下推给引擎）；
    ENABLE/DISABLE KEYS 是 MyISAM/Heap 特有，与其它操作组合时不能 inplace；
    存储生成列在 server 求值，因此不能 inplace 加/改。
  */
  if (alter_info->flags &
      (Alter_info::ALTER_ORDER | Alter_info::ALTER_KEYS_ONOFF))
    return true;                                  // ②
  ...
```

★ 条件 ① 值得单独记：**临时表的 ALTER 一定走 COPY**——这与 [`instant_ddl.md`](instant_ddl.md) 的 FR11（临时表不支持 INSTANT）是同一条约束链上的两个落点。

### 快路径：`simple_rename_or_index_change`——根本不进算法判定的那些 ALTER

有一类 ALTER **压根不走上面那套三条件判定**，而是提前走快路径：

```cpp
static bool is_simple_rename_or_index_change(const Alter_info *alter_info) {
  return (!(alter_info->flags &
            ~(Alter_info::ALTER_RENAME | Alter_info::ALTER_KEYS_ONOFF)) &&
          alter_info->requested_algorithm !=
              Alter_info::ALTER_TABLE_ALGORITHM_COPY);
}
```

条件读法：`flags` 除了 `ALTER_RENAME`（改名）和 `ALTER_KEYS_ONOFF`（ENABLE/DISABLE KEYS）之外**不能有任何其它位**，且未显式要求 COPY。

★ 这里有个和上一节**相互印证**的点：`ALTER_KEYS_ONOFF` 在 `is_inplace_alter_impossible` 里是**一票否决 INPLACE** 的，但单独使用时却能走快路径——两者并不矛盾，因为快路径在算法判定之前就返回了。源码注释也点明了：*"ENABLE/DISABLE KEYS is a MyISAM/Heap specific operation that is not supported for in-place in combination with other operations. **Alone, it will be done by simple_rename_or_index_change()**."*

⇒ 结论：**`ALTER TABLE t ENABLE KEYS` 单独执行走快路径（很快）；一旦与其它操作写在同一个 ALTER 里，就会掉到 COPY**。这是写复合 ALTER 语句时最容易踩的坑之一。

### INPLACE vs COPY 的 **server 侧**差异

> 引擎侧差异（三件套语义、row log）见 **4.6（inplace 四件套）** 与 **七（row log）**。下面只列 **server 层额外要做的事**。

#### INPLACE：`mysql_inplace_alter_table()`

**① MDL 升降级**（server 做的，不是引擎）——online 窗口就在这里打开：

```cpp
/*
  Upgrade to EXCLUSIVE lock if:
  - This is requested by the storage engine
  - Or the storage engine needs exclusive lock for just the prepare phase
  - Or requested by the user
*/
if (...) {
  if (wait_while_table_is_used(thd, table, HA_EXTRA_FORCE_REOPEN)) goto cleanup;
  /*
    Get rid of all TABLE instances belonging to this thread
    except one to be used for in-place ALTER TABLE.
    This is mostly needed to satisfy InnoDB assumptions/asserts.
  */
  close_all_tables_for_name(thd, table->s, false, table);
```

prepare 之后**降级**：

```cpp
if (inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
    alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_SHARED)
  table->mdl_ticket->downgrade_lock(MDL_SHARED_NO_WRITE);
else {
  assert(inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE);
  table->mdl_ticket->downgrade_lock(MDL_SHARED_UPGRADABLE);
}
```

**② ★ DD 替换（server 侧的原子替换核心）**：

```cpp
/*
  Replace table definition in the data-dictionary.

  Note that any error after this point is really awkward for storage engines
  which don't support atomic DDL. Changes to table in SE are already
  committed and can't be rolled back. ...
  For engines supporting atomic DDL error is business-as-usual situation.
*/
altered_table_def->set_schema_id(table_def->schema_id());
altered_table_def->set_name(alter_ctx->alias);
altered_table_def->copy_triggers(table_def);
if (thd->dd_client()->drop(table_def)) goto cleanup2;
...
if ((db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
  /*
    For engines supporting atomic DDL we have delayed storing new
    table definition in the data-dictionary so far in order to avoid
    conflicts between old and new definitions on foreign key names.
    Since the old table definition is gone we can safely store new
    definition now.
  */
  if (thd->dd_client()->store(altered_table_def)) goto cleanup2;
} else {
  if (thd->dd_client()->update(altered_table_def)) goto cleanup2;
  ... trans_commit_stmt(thd) || trans_commit_implicit(thd) ...
}
```

**③ 提交后通知引擎**（★ 必须在提交之后）：

```cpp
if (db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) {
  if (trans_commit_stmt(thd) || trans_commit_implicit(thd)) goto cleanup2;
  /* Call SE DDL post-commit hook. */
  if (db_type->post_ddl) db_type->post_ddl(thd);
  /*
    Finally we can tell SE supporting atomic DDL that the changed table
    in the data-dictionary.
  */
  ... new_table_list.table->file->ha_notify_table_changed(ha_alter_info);
}
```

> 例外：OPTIMIZE TABLE 要 `/* Disable GTID and SPCO for OPTIMIZE TABLE to avoid deadlock. */`

#### COPY：内联在 `mysql_alter_table()` 里

server 侧**额外**要做的事：

1. **`fk_check_copy_alter_table()`**（INPLACE 不做）——COPY **不允许重命名 FK 列**：*"Copy algorithm doesn't support proper renaming of columns in the foreign key yet"*
2. **MDL 强制 SNW**（`LOCK=NONE` 直接拒绝：*"COPY algorithm doesn't work with concurrent writes."*）
3. **建临时表到磁盘 + 拷数据**（`copy_data_between_tables()` 逐行走 SQL 层 INSERT），其中：

```cpp
bool mysql_trans_prepare_alter_copy_data(THD *thd) {
  /*
    Turn off recovery logging since rollback of an alter table is to
    delete the new table so there is no need to log the changes to it.
    ...
  */
```

4. **三段改名（备份 → 换表 → 删备份）**：

```cpp
snprintf(backup_name, sizeof(backup_name), "%s2-%lx-%x", tmp_file_prefix, current_pid, thd->thread_id());
```

★ 备份改名失败时有一段"**重试 20 次**"的 hack：

```
// Restore the backup of the original table to its original name.
// If the operation fails, we need to retry it to avoid leaving
// the dictionary inconsistent.
//
// This hack might become unnecessary once InnoDB stops acquiring
// gap locks on DD tables (which might cause deadlocks).
uint retries = 20;
```

5. **触发器 / check 约束名迁移**（schema 内唯一名冲突，必须先把旧版本改名）：

```cpp
/*
  Since trigger names have to be unique per schema, we cannot
  create them while both the old and the tmp version of the
  table exist.
*/
if (backup_table->has_trigger()) {
  new_dd_table->copy_triggers(backup_table);
  backup_table->drop_all_triggers();
  update = true;
}
```

⇒ INPLACE 只需 `copy_triggers` 一次，COPY 要"迁移 + 清空"两次。

**对照表**：

| 关注点 | INPLACE | COPY |
|---|---|---|
| 新表版本 | 只建 `dd::Table`（可在内存）+ `open_table_uncached` | 还要 `ha_create_table()` 真正落地 + 拷数据 |
| FK 列改名 | 允许 | 禁止 |
| 触发器/约束名 | `copy_triggers` 一次 | 迁移 + `drop_all_triggers` 两次 |
| 换表 | 仅 RENAME 时 | 必然（备份名 → 新名 三段式） |
| 统计/行数 | 无 | `my_ok(copied+deleted)` |
| no-op 特判 | `handler_flags == 0` → 跳过引擎调用 | 无 |

---

### 阶段一：`open_tables` 与外键 MDL 集合的作用范围

`mysql_alter_table` 在前置检查后第一件事不是拿 X 锁，而是**打开表**，且用一个 ALTER 专用的 prelocking 策略：

```cpp
Alter_table_prelocking_strategy alter_prelocking_strategy;
bool error = open_tables(thd, &table_list, &tables_opened, 0,
                         &alter_prelocking_strategy);
// open_tables/open_table 的完整剖析（含 backoff 重试机制）见 ../../server/table.md「1. 开表」
```

`Alter_table_prelocking_strategy` 的三个方法（`sql_base.cc`）**全部返回 false**：

```cpp
bool Alter_table_prelocking_strategy::handle_routine(...) { return false; }
bool Alter_table_prelocking_strategy::handle_table(...)   { return false; }
bool Alter_table_prelocking_strategy::handle_view(...)    { return false; }
```

注释写明：*Unlike in DML, we do not process triggers here*——DML 要为触发器/函数预展开锁集合，ALTER 只打开目标表本身。真正让 ALTER "锁一堆无关表"的是**随后的外键 MDL 收集**（非临时表时执行，四个来源汇入 `MDL_request_list` 一次获取）：

```cpp
Foreign_key_parents_invalidator fk_invalidator;
if (table->s->tmp_table == NO_TMP_TABLE) {
  MDL_request_list mdl_requests;

  // ① 新加的外键 → 父表（SU）
  if (collect_fk_parents_for_new_fks(thd, ..., MDL_SHARED_UPGRADABLE, ...))

  // ② 现有全部外键 → 父表（SU）；simple_rename/index_change 免收
  if (!is_simple_rename_or_index_change(alter_info)) {
    if (collect_fk_parents_for_all_fks(thd, old_table_def, nullptr,
                                       MDL_SHARED_UPGRADABLE, ...))

    if (create_info->db_type != table->s->db_type()) {
      /* 换引擎可能"唤醒"孤儿外键：新引擎下旧表变成某孤儿 FK 的父表，
         需要锁那些子表；换引擎后旧表不可能再是任何 FK 的父表 */
      if (collect_fk_children(thd, table_list->db, table_list->table_name,
                              create_info->db_type, MDL_SHARED_UPGRADABLE, ...))
    } else {
      if (collect_fk_children(thd, old_table_def, MDL_SHARED_UPGRADABLE, ...))
    }

    // ③ 改名时：新名字也可能接上孤儿外键 → 子表（SU）
    if (alter_ctx.is_table_renamed() &&
        collect_fk_children(thd, alter_ctx.new_db, alter_ctx.new_alias, ...))

    // ④ 要 DROP 的外键 → 锁外键名字本身
    if (collect_fk_names_for_dropped_fks(thd, table_list->db, alter_info, ...))
  }
}
```

锁级别全部是 `MDL_SHARED_UPGRADABLE`——这就是**"DDL 会等一堆看似无关的表"的根源**：你 `ALTER TABLE t`，但 `t` 的每个 FK 父表、（换引擎/改名时的）FK 子表都要拿 SU。若某父表上有长事务（哪怕只是 SELECT），你的 ALTER 在这里就会卡 `Waiting for table metadata lock`。四条收集各有讲究：

1. **新增 FK → 父表 SU**：要在 ALTER 期间校验父表定义（列类型/索引）。
2. **现有 FK → 父表 SU**：ALTER 可能删列/删索引破坏 FK，检查时需稳定读父表定义；`simple_rename_or_index_change` 路径不碰列，免收（对应快路径节）。
3. **换引擎/改名 → 子表 SU**：**孤儿外键**（父表还不存在或引擎不支持时的悬空 FK）会被引擎切换"激活"——新引擎支持 FK 后，那些引用本表的外键突然生效，所以要把子表锁住；改名后新名字同理。
4. **外键名字**：FK 名在 schema 内唯一，DROP FK 时先锁名字，防并发 `CREATE TABLE` 抢注同名 FK。**要 ADD 的外键名此刻不锁**——注释说明它落在哪个库取决于后面选的算法（COPY 建在新库中间表、INPLACE 建在原库），名字到 `mysql_inplace_alter_table` 里才锁。

还有一条 LOCK TABLES 特例：在此模式下要求所有父表已持 `MDL_SHARED_READ_ONLY` 以上，否则报 `ER_TABLE_NOT_LOCKED`——否则 ALTER 结束后子表持 WRITE 锁、父表没锁，FK 检查无从谈起。

### 阶段二：新表定义的构建

`mysql_prepare_alter_table`（意图 → 动作的翻译，见前文专节）之后还有一串**定案与兜底**，然后才建中间表：

```cpp
// ① GIPK 限制：mandatory primary key 模式下不允许删 PK 等操作
if (check_primary_key_alter_restrictions(thd, create_info->db_type,
                                         alter_info, table)) return true;

// ② 空间列改 SRID 但有空间索引 → 拒绝（索引会失效）
if (!is_alter_geometry_column_valid(alter_info)) return true;

// ③ 库级默认字符集应用到 create_info
if (set_table_default_charset(thd, create_info, *schema)) return true;

// ④ 强制退 COPY 的三个条件（这是算法决策的 server 侧兜底）
if ((thd->variables.old_alter_table &&          // 老参数 old_alter_table=1
     ...requested_algorithm != INPLACE && != INSTANT) ||
    is_inplace_alter_impossible(table, create_info, alter_info) ||  // 操作 InnoDB 不支持
    (partition_changed && !(...HA_USE_AUTO_PARTITION) && !new_part_info)) {
  // 用户显式要 INPLACE/INSTANT → fail-fast 报 ER_ALTER_OPERATION_NOT_SUPPORTED
  alter_info->requested_algorithm = Alter_info::ALTER_TABLE_ALGORITHM_COPY;
}

// ⑤ pre-5.6.4 老 temporal 类型升级（avoid_temporal_upgrade 关闭时）
if (check_temporal_upgrade) upgrade_old_temporal_types(thd, alter_info);

// ⑥ ALTER TABLE ... ENGINE=同引擎 是"求重建"的惯用法 → 强制置 ALTER_RECREATE
if (create_info->db_type == table->s->db_type() &&
    create_info->used_fields & HA_CREATE_USED_ENGINE)
  alter_info->flags |= Alter_info::ALTER_RECREATE;
```

随后进入建中间表段。三个关键动作：

```cpp
// ① 先拿中间表名 #sql-xxx 的 MDL X（STATEMENT duration）
MDL_REQUEST_INIT(&tmp_name_mdl_request, MDL_key::TABLE, alter_ctx.new_db,
                 alter_ctx.tmp_name, MDL_EXCLUSIVE, MDL_STATEMENT);
thd->mdl_context.acquire_lock(&tmp_name_mdl_request, ...);

// ② create_table_impl 建中间表——注意 binlog 关闭 + 禁止中间提交
Disable_binlog_guard binlog_guard(thd);
Implicit_substatement_state_guard substatement_guard(...);
error = create_table_impl(..., alter_ctx.tmp_name, ..., &non_dd_table_def, ...);

// ③ 原子性取决于新旧两个引擎
bool atomic_replace = (new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
                      (old_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
bool no_ha_table = true;   // 记住：引擎对象还没建
```

三个细节都值得记住：**中间表名的 MDL X 必须先拿**（它也是普通表名，别的会话可能正巧引用它）；**建中间表关 binlog**（`#sql-` 中间表不能进复制，最终只 binlog 一条 ALTER 语句）；**`non_dd_table_def`**——目标引擎支持原子 DDL 时，新表定义**暂不写入磁盘 DD**，一直攥在内存里直到 rename 阶段才 `store()`，这既省了失败回滚，也避免新旧两版 FK 定义在 DD 里打架。

### `mysql_inplace_alter_table()`：INPLACE 的 server 侧执行主体（全流程）

前文 INPLACE 节拆了"降级与两次 apply"，这里把整个函数从头到尾串一遍（`sql_table.cc:13332` 起）：

```text
mysql_inplace_alter_table()
├─ ① 升锁（按六档返回值决定升多高）
│    EXCLUSIVE_LOCK / *_AFTER_PREPARE+LOCK_TABLES / 用户要 X
│      → wait_while_table_is_used(X) + close_all_tables_for_name(留一个)
│    *_AFTER_PREPARE（非 LOCK_TABLES）
│      → upgrade_shared_lock(X) + tdc_remove_table(NOT_OWN_KEEP_SHARE)
│    SHARED_LOCK / 用户 LOCK=SHARED → 升 SNW
├─ ② 补锁 FK 名字（ADD 的外键此刻才锁名字；改名则连新名 FK 一起）
│    + FK 名冲突预检（ER_FK_DUP_NAME，趁还没开始昂贵阶段）
├─ ③ lock_tables + error_if_not_empty 检查（几何列/DATETIME 无默认值
│    要求空表；空表时 NO_LOCK 降为 SHARED_LOCK，且断言不许 INSTANT）
├─ ④ online 标记：NO_LOCK* 且用户没要锁 → ha_alter_info->online = true
├─ ⑤ ha_prepare_inplace_alter_table（引擎第一件套）
│    成功后按 *_AFTER_PREPARE 降锁：SNW 或 SU ← 这就是 online 窗口开启
├─ ⑥ ha_inplace_alter_table（引擎第二件套；InnoDB 的 row log 主阶段）
├─ ⑦ wait_while_table_is_used(PREPARE_FOR_RENAME) ← 升回 X，短暂阻塞点
│    + collect_and_lock_fk_tables_for_complex_alter_table（重收 FK 相关表）
│    + alter_table_drop_histograms（直方图失效即删）
├─ ⑧ ha_commit_inplace_alter_table(commit=true)（引擎第三件套）
├─ ⑨ 关掉所有旧 TABLE 实例；rollback_needs_dict_cache_reset = true
├─ ⑩ DD 替换：altered_table_def 补 schema_id/名字/可见性 + copy_triggers
│    → dd_client()->drop(旧定义) → store(新定义)（原子 DDL 引擎才延迟 store，
│      非原子引擎这里 update + trans_commit_stmt/implicit 落盘）
│    ★ 注释明言：这之后出错，对非原子引擎"非常尴尬"——引擎侧已提交无法回滚
├─ ⑪ 非原子引擎：reopen + ha_notify_table_changed + tdc_remove_table(ALL)
├─ ⑫ 改名：mysql_rename_table(原子引擎带 NO_DD_COMMIT)
├─ ⑬ adjust_fks_for_complex_alter_table（FK 落到最终名字）
├─ ⑭ write_bin_log(…, HTON_SUPPORTS_ATOMIC_DDL)   ← 收尾 binlog
├─ ⑮ update_referencing_views_metadata（引用视图的元数据刷新）
├─ ⑯ trans_commit_stmt/implicit（原子引擎在此提交）
├─ ⑰ post_ddl（新旧两个引擎都调；InnoDB 在这里清 DDL log）
└─ ⑱ LOCK_TABLES 模式：释放旧名锁、新名锁降级 SNRW；fk_invalidator.invalidate
```

三个流程要点：

- **两次升 X、一次降级**：⑦ 的 `wait_while_table_is_used` 是 INPLACE 真正的第二个短阻塞点（第一个在 ① prepare 期升 X）。`*_AFTER_PREPARE` 档在 ⑤ 后降锁开 online 窗口、⑦ 再升回去——这就是"INPLACE ≠ 一直持 X"的完整形态。
- **FK 名字是延迟锁的**：阶段一收的是父/子表 SU 与要 DROP 的 FK 名；要 ADD 的 FK 名到 ② 才锁——因为名字落点取决于算法（INPLACE 在原库，COPY 在中间表所在库）。
- **原子 DDL 与否在 ⑩ 分岔**：支持原子 DDL 的引擎，DD 更新走 `drop 旧 + store 新` 且不中间提交，出错整体回滚；不支持的非原子引擎在 ⑩ 直接 `trans_commit_stmt` 落盘 DD——所以注释说这之后的错误对非原子引擎"无解"，只能希望旧定义兼容新引擎。这是 [`atomic_ddl.md`](atomic_ddl.md) "引擎四件套在 server 侧的提交序" 的第一现场。

### 收尾：binlog / 提交 / 回滚与失败清理

COPY 路的收尾（INPLACE 的上面已含）是**备份名三段式**，所有 rename 型表替换都长这样：

```text
 COPY 路成功链：
   数据拷完 → close_temporary_table(不删) → wait_while_table_is_used(X)
   → 旧表 rename 成备份名 #sql-2-<pid>-<tid>（拿该名的 MDL X，防 I_S 打开）
   → [atomic_replace 才在此 store 新定义，否则更早已提交]
   → 中间表 #sql-xxx rename 成正式名 → 调整 FK/CC/触发器名 → histograms
   → quick_rm_table(备份名) ← 删旧表（Fn_IS_TMP）
   → write_bin_log → update_referencing_views_metadata → commit → post_ddl
```

**失败清理链**（`err_with_mdl` 前的回滚段）才是细节密集区：

```cpp
if (mysql_rename_table(..., FN_FROM_IS_TMP | ...) ||  // 中间表 → 正式名失败
    adjust_fks_for_complex_alter_table(...) ||
    (!atomic_replace && ... trans_intermediate_ddl_commit(...))) {
  if (!atomic_replace) {
    ...
    // ① 删掉中间表（引擎对象 + DD），不留垃圾
    (void)quick_rm_table(thd, new_db_type, alter_ctx.new_db,
                         alter_ctx.tmp_name, FN_IS_TMP);

    // ② 把备份名恢复回正式名——失败要重试 20 次
    // 注释原文：This hack might become unnecessary once InnoDB stops
    // acquiring gap locks on DD tables (which might cause deadlocks).
    uint retries = 20;
    while (retries-- &&
           mysql_rename_table(thd, old_db_type, alter_ctx.db, backup_name,
                              ..., FN_FROM_IS_TMP | NO_FK_CHECKS |
                              NO_FK_RENAME | NO_CC_RENAME))
      ;
  }
  goto err_with_mdl;
}
```

`quick_rm_table` 与"重试 20 次"两处各有故事：

- **`quick_rm_table`**（vs 正常 `rm_table`）：不做完整 DROP 流程（不通知 binlog、不跑引擎 drop），只按路径直接删引擎对象 + DD 记录，专用于"清理自己刚建的东西"。失败路径上所有 `quick_rm_table` 都 `(void)` 忽略返回值——**清理失败不掩盖原始错误**，残骸留给崩溃恢复（非原子引擎）或 DD 校验。
- **重试 20 次**：恢复备份名时若 `mysql_rename_table` 失败（注释点名原因：InnoDB 对 DD 表的 gap lock 可能造成死锁），就重试最多 20 次。注意这是**尽力而为**——20 次全失败时 `retries` 耗尽循环退出，照样 `goto err_with_mdl`，表停留在备份名状态，靠用户看到错误后再处理。这是"非原子 ALTER 失败可能留下 `#sql-2-*` 残骸"的代码出处。
- **binlog 的写法**：`write_bin_log(thd, true, query, length, atomic_replace && !is_noop)`——最后一个参数是否进**事务 cache**。原子 DDL 引擎的 ALTER 与 DD 事务同生共死所以进 trx cache；no-op ALTER（无引擎调用）没有活跃事务，只能进 stmt cache，注释特意解释了这一点。
- **成功删备份**：`quick_rm_table(..., FN_IS_TMP | (atomic_replace ? NO_DD_COMMIT : 0))`——原子路径下删备份也是 DD 事务的一部分，随 `trans_commit_stmt/implicit` 一起生效；这正是"看到旧表残留 `#sql-2-*` = 事务没走完"的诊断依据。

---
> ★ **ALTER 的三条执行路已各自独立成篇**（本篇保留 server 侧编排骨架与 4.5 的两路对照）：
> - **COPY 路**：server 编排 + `copy_data_between_tables` 逐段 + 为什么一定不 Online → [`copy_ddl.md`](copy_ddl.md)
> - **INPLACE 路**：MDL 升降级 + 引擎四件套 + row log 两次 apply 与短暂阻塞点 → [`inplace_ddl.md`](inplace_ddl.md)
> - **INSTANT 路**：行版本号与三层级 version 辨析 → [`instant_ddl.md`](instant_ddl.md)

### `mysql_recreate_table()`：ALTER 的薄包装

函数前注释原文：*\"Recreates tables by calling mysql_alter_table().\"*，参数 `table_copy` = *\"Recreate the table by using ALTER TABLE COPY algorithm\"*。

实现极短——就是给 `mysql_alter_table()` 预设 flag：

```c
  table_list->set_lock({TL_READ_NO_INSERT, THR_DEFAULT});
  table_list->mdl_request.set_type(MDL_SHARED_NO_WRITE);
  create_info.row_type = ROW_TYPE_NOT_USED;
  create_info.default_table_charset = default_charset_info;
  alter_info.flags = (Alter_info::ALTER_CHANGE_COLUMN | Alter_info::ALTER_RECREATE);
  if (table_copy) alter_info.requested_algorithm = Alter_info::ALTER_TABLE_ALGORITHM_COPY;
  return mysql_alter_table(thd, NullS, NullS, &create_info, table_list, &alter_info);
```

**什么时候用**（`sql_admin.cc`）：

1. **`OPTIMIZE TABLE` 的旧路径**：`res = (specialflag & SPECIAL_NO_NEW_FUNC) ? mysql_recreate_table(thd, first_table, true) : mysql_admin_table(..., &handler::ha_optimize, ...)` —— 即 `--skip-new` 模式走 `table_copy=true`，正常模式走 `ha_optimize`
2. **表需要"升级/重建"时**：另外两处调用点，都包在 `Disable_binlog_guard` 里（注释 *\"binlogging is done by caller if wanted\"*），且要求先 `open_temporary_tables` 或断言 `!table->table->s->tmp_table`（注释：*\"Temporary table are always created by current server so they never require upgrade.\"*）

`sql_alter.h` 里 `ALTER_RECREATE` 的注释明确：*\"Set for FORCE / Set for ENGINE(same engine) / Set by mysql_recreate_table()\"*。

### `notify_alter_table`：MDL 升 X 太晚的补救钩子

- 是 **`handlerton` 的函数指针成员**（`sql/handler.h`）：
  `typedef bool (*notify_alter_table_t)(THD *thd, const MDL_key *mdl_key, ha_notification_type notification_type);`
  与 `notify_exclusive_mdl_t` / `notify_rename_table_t` / `notify_truncate_table_t` 同族
- **存在理由**（`handler.h` 注释大意）：*\"This hook is necessary because for ALTER TABLE upgrade to X metadata lock happens fairly late during the execution process, so it can be expensive to abort ALTER TABLE operation at this stage by returning failure from notify_exclusive_mdl() hook.\"*
  ⇒ MDL 升 X 发生得太晚，那时失败代价太大，所以在 ALTER **执行前 / 执行后**各给引擎一次 pre/post 通知
- **谁调用**：`handler.cc` 的 `notify_table_ddl_helper()`（plugin 遍历器），按 `params->ddl_type` 分派 `HA_ALTER_DDL` → `hton->notify_alter_table(...)`
- ★ **InnoDB 没有实现**：`storage/` 全目录 grep `notify_alter_table` **只有 NDB**（`ha_ndbcluster.cc` 里 `ndbcluster_hton->notify_alter_table = ndbcluster_notify_alter_table;`）。InnoDB 未注册该钩子
- ⚠️ **引入版本：源码无直接证据**（没找到版本标注/WL 号），不要写死"8.0 引入"

> 顺带纠正一个 5.x 遗留概念：**`ha_alter_table()` 在 8.0.39 全仓 0 命中**，该符号已不存在。8.0 里 server 与引擎 ALTER 的接缝是：`handler::check_if_supported_inplace_alter()`（算法判定）+ inplace 四件套（执行）+ `notify_alter_table`（可选通知）+ COPY 路径三函数（`mysql_trans_prepare_alter_copy_data` / `write_row` / `mysql_trans_commit_alter_copy_data`）。

---

---

## 可观测性

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `old_alter_table` | OFF | Global/Session | 强制走 COPY 算法（用于性能对比）；**显式指定 `ALGORITHM=INPLACE/INSTANT` 时不生效**——此时按 fail-fast 直接报错 |

> `ALGORITHM` / `LOCK` 是**语句级子句**而非系统变量：写在 `ALTER TABLE` 语句末尾，用逗号与表名、列定义分隔（`ALTER TABLE t ADD PRIMARY KEY (c), ALGORITHM=INPLACE;`）。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 说明 |
|--------|------|------|
| 这次 ALTER 走了哪个算法 | 显式指定 + fail-fast | 指定 `ALGORITHM=X` 后若报 `ER_ALTER_OPERATION_NOT_SUPPORTED`，即该算法不可行——**用报错反推**是最可靠的判定方式 |
| 默认模式下走了哪个 | `SHOW PROCESSLIST` / PFS | State：`copy to tmp table` 即 COPY；INPLACE 各阶段 State 不同（见 [`ddl.md`](ddl.md) 观测章） |
| 是否发生 rebuild（换表） | SQL | rebuild 会分配新 `TABLE_ID`；重建后 `INNODB_TABLES.TOTAL_ROW_VERSIONS` 归零（见 [`instant_ddl.md`](instant_ddl.md)） |
| 判定链条为什么这么选 | 源码路径 | `is_inplace_alter_impossible`（server 一票否决）→ `check_if_supported_inplace_alter`（引擎六档返回值）→ 见 [`ddl.md`](ddl.md)「算法判定的真正链条」 |

---

## Misc

### 面向二次开发

**扩展点**：新增一种 ALTER 语义要动三处——`Alter_info::ALTER_*` 加请求标志位；`mysql_prepare_alter_table` 加"意图 → 动作"的翻译分支；判定侧决定它是 INSTANT / INPLACE / COPY（server 侧在 `is_inplace_alter_impossible` 加否决，或交给引擎 `check_if_supported_inplace_alter` 回答）。落地实现分别在 [`instant_ddl.md`](instant_ddl.md) / [`inplace_ddl.md`](inplace_ddl.md) / [`copy_ddl.md`](copy_ddl.md)。

### 坑与已知缺陷

| 坑 | 说明 |
|---|---|
| 临时表 ALTER 一律 COPY | `is_inplace_alter_impossible` 对 `table->s->tmp_table` 直接返回 true；与 INSTANT 不支持临时表（FR11）是同一条约束链 |
| 显式 ALGORITHM 做不到即报错 | fail-fast 设计，不静默降级；生产脚本里写了 `ALGORITHM=INPLACE` 又遇到不支持的操作会直接失败 |
| `ALGORITHM=INSTANT` 不能与 `LOCK` 子句同用 | 报 `ER_WRONG_USAGE`，只能用 `LOCK=DEFAULT`（见 [`instant_ddl.md`](instant_ddl.md)） |
| server 判定偏保守 | `IS_EQUAL_*` 只有三档、InnoDB 连 `PACK_LENGTH` 差异都不接受 ⇒ 部分理论可 INPLACE 的操作落到 COPY |

### 易混淆概念

| 概念 | 辨析 |
|---|---|
| 本篇 vs 三篇算法篇 | 本篇是**公共前段**（意图 → 动作 → 算法分岔）；分岔之后的实现在 `instant_ddl.md` / `inplace_ddl.md` / `copy_ddl.md` |
| `Alter_info` vs `Alter_inplace_info` | 前者是"用户要什么"（parser 填），后者是"引擎要做什么"（传给四件套） |
| `ha_alter_table` | **8.0 已不存在**的旧接口；现在只有四件套三段式。InnoDB 也未实现 `notify_alter_table` 钩子（见下节） |

---

## 参考

**相关文档**
- 上层骨架（三种算法判定、MDL 三段式、DDL 观测、binlog 与复制）见 [`ddl.md`](ddl.md)
- CREATE TABLE（含引擎侧建表空间 / 建 B-tree 根页）见 [`create_table.md`](create_table.md)
- COPY 全链路见 [`copy_ddl.md`](copy_ddl.md)；INPLACE 全链路与 row log 见 [`inplace_ddl.md`](inplace_ddl.md)
- INSTANT 见 [`instant_ddl.md`](instant_ddl.md)；新版 DDL 引擎与并行构建见 [`ddl_engine.md`](ddl_engine.md)
- 原子 DDL 见 [`atomic_ddl.md`](atomic_ddl.md)；生产在线变更工具见 [`online_ddl_tools.md`](online_ddl_tools.md)

> 本篇由 `ddl.md` 的 ALTER TABLE 章迁出而成；原篇的 WL / bug 参考仍保留在 [`ddl.md`](ddl.md)「参考」章，不在此重复。
