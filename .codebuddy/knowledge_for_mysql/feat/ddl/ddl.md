# DDL 深度剖析（server ↔ InnoDB 跨层全链路）

> **归位**：DDL 天然横跨 server 与引擎两层，按"跨层主题归 `feat/`、**一篇写完两层**"的原则放这里。
>
> ★ **本篇是融合结构，不是"server 侧一篇 + 引擎侧一篇"**——按**主题**编排，每个主题章内同时给出两层视角与接缝：

| 章 | 主题 | 两层视角如何融合 |
|---|---|---|
| 二 | DDL 全流程与三种算法 | 引擎侧骨架：COPY / INPLACE / INSTANT 的选择策略 + **MDL 锁三段式** |
| 三 | CREATE TABLE | ★ 已独立成篇 → [`create_table.md`](create_table.md)：server 侧编排（→ `ha_create_table()` 接缝）+ **引擎侧落地**（`create_table_info_t` 三段式 / 建 .ibd / `btr_create` 建根页 / 统计置空） |
| 四 | ALTER TABLE | ★ 已独立成篇 → [`alter_table.md`](alter_table.md)：server 侧 `mysql_alter_table()` 编排主线；COPY / INPLACE / INSTANT / 并行构建各自分篇 |
| 五 | DROP / TRUNCATE / RENAME | server 侧 `mysql_rm_table()` + **InnoDB 实现** + 为什么不记 undo + btr_free |
| 六 | ★★ 原子 DDL | **server 侧编排七步** + **引擎侧** `dd::Table` 条件持久化 + DDL log + 回滚路径 |
| 七–九 | row log / 并行 DDL / Instant DDL | 引擎侧专深（online DDL 的并发控制、8.0.27+ 并行构建、行版本号机制） |
| 十–十三 | 行为速查 / binlog / 数据结构 / 调用栈 | 速查与参考 |

> 原 `innodb/ddl.md`（1404 行）已**完整并入，未删减**，并按主题**拆散嵌入**对应章节（不再是独立的一块）。单层通用机制（如 `row_search_mvcc`、MDL 通用语义）仍留在按层目录，本篇只写跨层主题自身。
>
> **边界**：本篇只讲 **MySQL 原生 DDL**（server 编排 + 引擎实现）。生产上用第三方工具做在线变更的机制（gh-ost 的 binlog 订阅与 magic lock cut-over、pt-osc 的触发器方案）见同目录的 [`online_ddl_tools.md`](online_ddl_tools.md)。

## 目录

- [总览：一条 DDL 的两层视角](#总览一条-ddl-的两层视角)（server 侧七步 / 引擎侧视角 / 理论基础）
- [DDL 全流程、三种算法与 MDL 三段式](#ddl-全流程三种算法与-mdl-三段式引擎侧骨架)
  - [MDL 锁的三段式（online DDL 的骨架）](#mdl-锁的三段式online-ddl-的骨架)
    - [Online 到底指什么：三个必须分清的结论](#online-的语义边界)（★ `INPLACE ≠ online`；row log 只服务 INPLACE rebuild，INSTANT 不需要）
  - [★ 算法判定的真正链条](#-算法判定的真正链条纠正一个常见误解)（★ `mysql_compare_tables` **不参与**判定；`IS_EQUAL_*` 只有三个；InnoDB 连 PACK_LENGTH 都不接受）
- **★ CREATE TABLE** → 已独立成篇 [`create_table.md`](create_table.md)（server 侧编排 + 引擎侧落地；★ 新表统计必为 0 行）
- **★ ALTER TABLE** → 已独立成篇 [`alter_table.md`](alter_table.md)（server 侧编排主线；三种算法与并行构建各自分篇）
- [DROP / TRUNCATE / RENAME（server 侧 + 引擎侧）](#drop--truncate--renameserver-侧--引擎侧)
  - [mysql_rename_table：★ 先 SE 后 DD](#mysql_rename_table-先-se-后-dd)（反直觉，WL#7743 解释）
  - [B-tree 的物理释放（btr_free）](#b-tree-的物理释放btr_free)
  - [mysql_rm_table_no_locks](#mysql_rm_table_no_locks差别不是谁持锁而是是否自己取锁)（DROP DATABASE 也直接调它）
- **★★ 原子 DDL**（server 编排七步 + 引擎 DDL log + 崩溃恢复闭环）→ 已独立成篇：见 [`atomic_ddl.md`](atomic_ddl.md)
- **★ 以下主题已迁出为独立篇**（本篇保留跨层公共骨架、算法选择与原子性）：
  - [create_table.md](create_table.md) —— **CREATE TABLE 全链路**：server 侧编排（`mysql_prepare_create_table` / `rea_create_base_table` / `ha_create_table` 接缝）+ **引擎侧落地**（`create_table_info_t` 三段式 / 建 .ibd / `btr_create` 建根页 / 统计置空）
  - [alter_table.md](alter_table.md) —— **ALTER TABLE 编排主线**：`Alter_info` / `Alter_table_ctx` / `mysql_prepare_alter_table`、INPLACE vs COPY 的 server 侧差异、`mysql_recreate_table`
  - [inplace_ddl.md](inplace_ddl.md) —— **INPLACE 全链路**：MDL 升降级、引擎四件套、**online DDL 与 row log**（为什么 apply 两次、短暂阻塞点从哪来）
  - [ddl_engine.md](ddl_engine.md) —— **8.0 新版 DDL 引擎与并行构建**（200 KB）：Loader/Builder/Btree_load、★★ 新旧对比表、★ 归并并行建树串行、**★ 任务队列的同步原语与终止判定**（quiescence 终止检测、防丢唤醒、锁外执行）、`Parallel_reader` 动态分片
  - [instant_ddl.md](instant_ddl.md) —— **Instant DDL**：行版本号三层级辨析、★ 官方 WL#11250 设计与两个坑
  - [copy_ddl.md](copy_ddl.md) —— **COPY 全链路**：server 编排 + `copy_data_between_tables` 逐段 + 为何一定不 Online
- [DDL 行为速查：四维评估与实战陷阱](#ddl-行为速查四维评估与实战陷阱)
- [DDL 与 binlog / 复制](#ddl-与-binlog--复制) / [关键数据结构](#关键数据结构全览) / [核心调用栈与系统变量](#核心调用栈与相关系统变量引擎侧)
- [★ DDL 如何观测](#ddl-如何观测要点)
  - [三个"看不到"及原因](#三个看不到及其真正原因)（★ **consumer 默认 OFF**，不是 instrument）
  - [进度为什么不准](#进度估算的偏差来源)（源码注释自认 wild guess / factor of 6）
  - [DDL 卡住：用 `metadata_locks` 诊断](#ddl-卡住用-metadata_locks-诊断)（★ 无 wait-for graph 表、无 VICTIM/TIMEOUT 列）
  - [State 与计数器](#state-与计数器ddl-相关速查)（State 不依赖 instrument；`innodb_metrics` 的 ddl 计数器默认 disabled）
  - [并行 DDL 观测](#并行-ddl-的观测手段)（`thread/innodb/ib_ddl`；★ PROCESSLIST 看不到）
  - [★★ `mysql.innodb_ddl_log` 不能直接查](#-mysqlinnodb_ddl_log-不能直接查)（用 `innodb_print_ddl_logs`，需配 `log_error_verbosity=3`）/ [崩溃恢复的两个坑](#-崩溃恢复的两个坑)
- [Misc](#misc) / [参考](#参考)

---

## 总览：一条 DDL 的两层视角

### server 侧：一条 DDL 的七步编排

```
① 拿 MDL（S→X / SU），锁名（表 + FK 父子表 + FK 名 + check 约束名 + trigger 名）
② 在【当前语句事务】里做 DD 变更
     dd_client()->store() / update() / drop()   ← 对 mysql.* DD 表的普通 DML
     非原子引擎例外：立即 trans_intermediate_ddl_commit()
③ 调引擎：ha_create_table() / ha_delete_table() / ha_rename_table() /
           ha_prepare|inplace|commit_inplace_alter_table()
   ★ 引擎可以在这些回调里修改 dd::Table（填 se_private_data 等）
④ 写 binlog：write_bin_log(..., is_trans = HTON_SUPPORTS_ATOMIC_DDL)
⑤ 一次提交：trans_commit_stmt() || trans_commit_implicit()
⑥ 提交后：hton->post_ddl(thd) —— 引擎 replay DDL log 做 .ibd 删除等滞后物理操作
⑦ 提交后通知：ha_notify_table_changed() / tdc_remove_table() / reopen
```

**为什么是这个顺序**：DD 本身就是 InnoDB 表 ⇒ ② 天然有 undo/redo；⑥ 的 `.ibd` 删建是**不可回滚的物理操作**，只能在提交后由 `post_ddl` 补做。

> ⚠️ **三处命名纠正**（8.0.39 实测，别按旧资料写）：
> - **`dd::execute_ddl()` 不存在**——原子 DDL 在 server 侧**没有独立编排入口**，逻辑分散在各 `mysql_*()` 里，靠 `HTON_SUPPORTS_ATOMIC_DDL` 一个 flag 做所有分岔
> - **`mysql_copy_alter_table()` 不存在**——COPY 算法**内联在 `mysql_alter_table()` 函数体里**（源码里就留了 `/* todo: ALTER TABLE using copy algorithm. */` 作标记）
> - **`Alter_table_handler` 不存在**——对应物是 `Alter_inplace_info` + `inplace_alter_handler_ctx`

---


### 引擎侧视角：本篇要回答什么



#### 是什么

8.0 原子 DDL 是"DDL 要么完全成功、要么完全回滚"的机制。InnoDB 侧的核心载体是 `mysql.innodb_ddl_log` 表和 `Log_DDL` 类（log0ddl.cc）：DDL 执行期把"破坏性物理操作"的意图写成日志记录，事务提交后再真正执行。

#### 用途

解决 5.7 及之前 DDL 中途崩溃导致的不一致（如 `#sql-xxx.ibd` 孤儿文件、表空间状态错乱）。把 free B-tree、删 .ibd、改名等不可回滚的物理操作，用"先记录意图、提交后执行"的方式纳入事务原子性。

#### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | DDL 非原子：崩溃可能留下孤儿 .ibd / 数据字典与实际文件不一致 |
| 8.0 | 引入 `mysql.innodb_ddl_log` 表 + `Log_DDL`（[WL#7977](https://dev.mysql.com/worklog/task/?id=7977) Atomic DDL）；`HTON_SUPPORTS_ATOMIC_DDL`；物理操作延迟到提交后 replay |
| 8.0.20+ | DDL log 类型扩展（ALTER ENCRYPT TABLESPACE 等）；`innodb_print_ddl_logs` 调试开关 |

---


### 理论基础


#### 设计模式

- **意图日志 / 延迟执行（intent log + deferred execution）**：执行期只向 DDL log 表写入"意图"记录，真正的物理操作推迟到事务提交后。破坏性操作一旦失败无法回滚，必须先记录、后执行
- **命令模式（Command）**：`DDL_Record` 是命令对象，携带类型 + 参数；`Log_DDL::replay` 按 `Log_Type` 分发到 `replay_free_tree_log` / `replay_delete_space_log` / `replay_drop_log` 等具体执行者
- **日志先行（Write-Ahead）**：物理操作执行前日志必须已持久化，与 ARIES 一致

#### 相关论文

- **ARIES**（Mohan et al., 1992）：do-undo-redo 恢复模型。原子 DDL 用 DDL log（redo 性质）记录物理操作意图，配合元数据 undo，是 ARIES 思想在 DDL 上的工程化应用

#### 算法与数据结构

- **DDL log 记录**：`DDL_Record` 类（log0ddl.h:78），字段 id / thread_id / type / space_id / page_no / index_id / table_id / file_path 等
- **幂等释放**：根页 `PAGE_INDEX_ID` 置 0（`BTR_FREED_INDEX_ID`），replay 前校验 index_id 匹配才释放——崩溃恢复重复 replay 安全
- **extent 级释放**：B-tree 释放粒度是文件段的一个 extent，而非逐页，一个 mtr 一个 extent 控制 mini-transaction 大小

#### 类似实现对比

【待补充】PostgreSQL 的 DDL 事务化（DDL 与 DML 同事务）、Oracle 的原子 DDL 机制对比

#### 历史背景

5.7 的 DDL 分两阶段（执行 + 元数据更新）且不原子，崩溃恢复靠 `innodb_force_recovery` 人工清理孤儿文件。8.0 引入原子 DDL 的动机：MySQL 官方 DD（data dictionary）表重构后（与 DDL 直接相关的设计见 [WL#7743](https://dev.mysql.com/worklog/task/?id=7743) *New data dictionary: changes to DDL-related...*；自举见 [WL#6394](https://dev.mysql.com/worklog/task/?id=6394) *Bootstrap code for new DD*），元数据本身存储在 InnoDB 表里，DDL 的元数据修改天然有 undo；顺势把物理操作也纳入原子性，形成完整方案。

---


---

## DDL 全流程、三种算法与 MDL 三段式（引擎侧骨架）

### DDL 全流程

本章给出 DDL 的**骨架**：先把"server 层怎么选算法、MDL 怎么拿放"讲清，后面各章（DDL log、row log、DROP/TRUNCATE）都是这条骨架上的某个环节。

### 三种算法与选择策略

⚠️ **勘误**：
- **不存在 `enum_alter_table_change_level`**（`ALTER_TABLE_METADATA_CHANGE` 等，全仓 0 命中，是 5.6 遗留物）
- **不存在 `need_copy_table` / `is_inplace` 变量**——真名是 `bool use_inplace = true;`（`sql_table.cc:17382`）
- **不存在 `ha_innobase::check_if_supported_alter()`**——INSTANT 判定内嵌在 `check_if_supported_inplace_alter()` 里，靠返回 `HA_ALTER_INPLACE_INSTANT` 表达
- **不存在 `is_inplace_alter()` / `is_instant_alter()`**——真名 `is_inplace_alter_impossible()` / `is_simple_rename_or_index_change()` / `innobase_need_rebuild()`
- **不存在 `HA_ONLINE_*` 标志**，`table_flags()` 与算法选择完全无关
- 算法**分散判定**，没有集中的 `switch (requested_algorithm)`

**第一步：server 层强制 COPY 的四条件**（`is_inplace_alter_impossible()`，`sql/sql_table.cc:12779`）：

```cpp
static bool is_inplace_alter_impossible(TABLE *table, HA_CREATE_INFO *create_info,
                                        const Alter_info *alter_info) {
  /* At the moment we can't handle altering temporary tables without a copy. */
  if (table->s->tmp_table) return true;                 // ① 临时表 → 必 COPY

  if (alter_info->flags &
      (Alter_info::ALTER_ORDER | Alter_info::ALTER_KEYS_ONOFF))
    return true;                                        // ② ORDER BY / ENABLE|DISABLE KEYS → COPY

  if (is_any_check_constraints_evaluation_required(alter_info)) return true;  // ③ CHECK 约束需重算 → COPY

  if (create_info->db_type != table->s->db_type()) return true;   // ④ 换引擎 → COPY

  if (!table->s->mysql_version) return true;            // ⑤ 5.0 之前的老表 → COPY

  /* If we are changing the SRID modifier of a column, we must do a COPY.
     But not if we are changing to the NULL SRID. */
  for (const Create_field &new_field_def : alter_info->create_list) {
    if (new_field_def.field != nullptr &&
        new_field_def.field->type() == MYSQL_TYPE_GEOMETRY) {
      const Field_geom *field_geom = down_cast<const Field_geom *>(new_field_def.field);
      if (field_geom->get_srid() != new_field_def.m_srid &&
          new_field_def.m_srid.has_value())
        return true;                                    // ⑥ 改 GEOMETRY 列 SRID（非 NULL）→ COPY
    }
  }
  return false;
}
```

**第二步：`check_if_supported_inplace_alter()` 的返回值枚举**（`sql/handler.h:191`）：

```cpp
enum enum_alter_inplace_result {
  HA_ALTER_ERROR,
  HA_ALTER_INPLACE_NOT_SUPPORTED,
  HA_ALTER_INPLACE_EXCLUSIVE_LOCK,
  HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE,
  HA_ALTER_INPLACE_SHARED_LOCK,
  HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE,
  HA_ALTER_INPLACE_NO_LOCK,
  HA_ALTER_INPLACE_INSTANT
};
```

**第三步：server 侧的决策 switch**（`sql_table.cc:17451`）：

```cpp
    enum_alter_inplace_result inplace_supported =
        table->file->check_if_supported_inplace_alter(altered_table, &ha_alter_info);

    switch (inplace_supported) {
      case HA_ALTER_INPLACE_EXCLUSIVE_LOCK:
        // If SHARED lock and no particular algorithm was requested, use COPY.
        if (alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_SHARED &&
            alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_DEFAULT) {
          use_inplace = false;        // ★★ LOCK=SHARED + 未指定算法 → 反而走 COPY（COPY 只需 SNW）
        }
        break;
      case HA_ALTER_INPLACE_NOT_SUPPORTED:
        if (alter_info->requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INPLACE) {
          ha_alter_info.report_unsupported_error("ALGORITHM=INPLACE", "ALGORITHM=COPY");
          goto err_new_table_cleanup;
        }
        if (alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_NONE) {
          ha_alter_info.report_unsupported_error("LOCK=NONE", "LOCK=SHARED");
          goto err_new_table_cleanup;
        }
        use_inplace = false;          // ★★ 唯一的 INPLACE→COPY 降级出口（DEFAULT 语义）
        break;
      case HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE:
      case HA_ALTER_INPLACE_NO_LOCK:
      case HA_ALTER_INPLACE_INSTANT:
        /*
          Note that any instant operation is also in fact in-place operation.
          ... If instant algorithm has some downsides to in-place algorithm and user
          explicitly asks for ALGORITHM=INPLACE it is responsibility of storage
          engine to fallback to in-place algorithm execution ...
        */
        break;                        // ★ ALGORITHM=INPLACE 时允许静默升级为 INSTANT
      case HA_ALTER_ERROR:
      default:
        goto err_new_table_cleanup;
    }

    if (use_inplace) {
      mysql_inplace_alter_table(...);
      goto end_inplace;
    }
```

**★ InnoDB 的返回值全集**（`handler0alter.cc:944`）——**只有 5 种，另外 3 种永不返回**：

| 返回 | 触发条件 |
|---|---|
| `HA_ALTER_ERROR` | 只读 / `srv_force_recovery`；显式 INSTANT 但字段数/row_version/行大小超限 |
| `HA_ALTER_INPLACE_NOT_SUPPORTED` | raw 设备、列数超限、改加密、**白名单外 flag**、ADD FK 且 `foreign_key_checks=ON`、NULL→NOT NULL 且非严格模式、单独 DROP PK、虚拟列混操作、已有 FTS 且需 rebuild |
| `HA_ALTER_INPLACE_INSTANT` | `instant_type ∈ {NO_CHANGE, VIRTUAL_ONLY, COLUMN_RENAME, ADD_DROP_COLUMN}` |
| `HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE` | 函数末尾 `online == true` |
| `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE` | 函数末尾 `online == false`（新增 AUTO_INCREMENT 列、虚拟列上建索引、加空间索引、建 FULLTEXT 索引等） |
| `EXCLUSIVE_LOCK` / `SHARED_LOCK` / `NO_LOCK` | **InnoDB 永不返回** |

```cpp
  return online ? HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE
                : HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE;
```

**★ 主闸门**（`handler0alter.cc:1030`）——任何不在白名单里的标志一律 COPY：

```cpp
  if (ha_alter_info->handler_flags &
      ~(INNOBASE_INPLACE_IGNORE | INNOBASE_ALTER_NOREBUILD | INNOBASE_ALTER_REBUILD)) {
    if (ha_alter_info->handler_flags & Alter_inplace_info::ALTER_STORED_COLUMN_TYPE) {
      ha_alter_info->unsupported_reason = innobase_get_err_msg(
          ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_COLUMN_TYPE);
    }
    return HA_ALTER_INPLACE_NOT_SUPPORTED;
  }
```

**`Alter_info::flags` 是 `ulonglong`，41 个标志位**（`sql/sql_alter.h:206`）：

```cpp
  enum Alter_info_flag : ulonglong {
    ALTER_ADD_COLUMN = 1ULL << 0,
    ALTER_DROP_COLUMN = 1ULL << 1,
    ALTER_CHANGE_COLUMN = 1ULL << 2,
    ALTER_ADD_INDEX = 1ULL << 3,
    ALTER_DROP_INDEX = 1ULL << 4,
    ALTER_RENAME = 1ULL << 5,
    ALTER_ORDER = 1ULL << 6,              // ★ 这个必导致 COPY
    ALTER_OPTIONS = 1ULL << 7,
    ALTER_KEYS_ONOFF = 1ULL << 9,         // ★ 这个必导致 COPY
    ALTER_RECREATE = 1ULL << 10,          // ALTER TABLE FORCE / ENGINE / OPTIMIZE
    ...
    ALTER_DISCARD_TABLESPACE = 1ULL << 27,
    ALTER_IMPORT_TABLESPACE = 1ULL << 28,
    ALTER_INDEX_VISIBILITY = 1ULL << 29,
    ...
    ALTER_COLUMN_VISIBILITY = 1ULL << 40  // ★ 最高位
  };

  enum enum_alter_table_algorithm {
    ALTER_TABLE_ALGORITHM_DEFAULT,   // INPLACE if supported, copy otherwise.
    ALTER_TABLE_ALGORITHM_INPLACE,   // In-place if supported, error otherwise.
    ALTER_TABLE_ALGORITHM_INSTANT,   // Instant if supported, error otherwise.
    ALTER_TABLE_ALGORITHM_COPY
  };

  enum enum_alter_table_lock {
    ALTER_TABLE_LOCK_DEFAULT,
    ALTER_TABLE_LOCK_NONE,
    ALTER_TABLE_LOCK_SHARED,
    ALTER_TABLE_LOCK_EXCLUSIVE
  };
```

ALTER TABLE 有三种算法，描述的是**执行逻辑**（数据怎么搬），与"是否 Online"（是否阻塞 DML）是**两个正交维度**：

| 算法 | 执行位置 | 数据怎么搬 | 是否 Online | 典型场景 |
|------|---------|-----------|------------|---------|
| **COPY** | server 层 | 建临时表 `#sql-xxx` → `copy_data_between_tables` 逐行走 SQL 层 INSERT | **一定不 Online**（全程 MDL X） | 改列类型、删主键、改表字符集 |
| **INPLACE** | InnoDB 层 | 引擎内部处理；可能只改元数据，也可能 bulk load **重建表** | **不一定**（看引擎返回值） | 加索引、加列、OPTIMIZE |
| **INSTANT** | 元数据 | 只改数据字典，不搬数据不重建 | 是 | 8.0 加列、改默认值、重命名表 |

选择策略（`mysql_alter_table`，sql/sql_table.cc）：

1. 用户显式指定 `ALGORITHM` → 用用户的；不支持就报错（**不会偷偷降级**）
2. 未指定 → 问引擎 `check_if_supported_inplace_alter`，优先 INPLACE，不支持则 COPY
3. 强制走 COPY 的条件：`old_alter_table=ON`（且未指定 INPLACE/INSTANT）、`is_inplace_alter_impossible`、分区变更且引擎不支持 auto-partition

引擎通过 `check_if_supported_inplace_alter` 返回 `enum_alter_inplace_result`，**这个返回值直接决定了 MDL 锁级别**：

| 返回值 | 含义 | MDL |
|--------|------|-----|
| `HA_ALTER_INPLACE_NOT_SUPPORTED` | 不支持 inplace | 走 COPY（全程 X） |
| `HA_ALTER_INPLACE_EXCLUSIVE_LOCK` | 支持 inplace 但需排他 | X（**不 Online**） |
| `HA_ALTER_INPLACE_SHARED_LOCK` | 共享锁 | S |
| `HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE` | prepare 后降级为共享 | X → S |
| `HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE` | prepare 后**不锁** | X → 无（**这就是 Online**） |
| `HA_ALTER_INPLACE_INSTANT` | 只改元数据 | 几乎不锁 |

注意 `*_AFTER_PREPARE` 后缀：**prepare 阶段需要 exclusive，prepare 之后可以降级**——这就是 online DDL 的 MDL 降级机制，也是"DDL 期间 DML 为什么能并发"的答案。

### MDL 锁的三段式（online DDL 的骨架）

所有 DDL（不论哪种算法）都遵循这个 MDL 流转：

```
① 开始：拿 MDL EXCLUSIVE，做准备工作
        （建临时表结构 / 分配 row log / 打开新表）
② 执行：降级为 MDL SHARED（或 NO），执行真正的 DDL
        （重建表 / 建索引）—— 此期间 DML 并发 ✅
③ 提交：升级回 MDL EXCLUSIVE，apply row log + 切换可见性
        （rename 换表 / 让新索引生效）—— 此期间阻塞 DML ❌
```

三段分别对应 handler 的三个回调：

```
① prepare_inplace_alter_table()     ← MDL X
② inplace_alter_table()             ← MDL S / NONE（online 窗口）
③ commit_inplace_alter_table()      ← 升回 MDL X（切换窗口）
```

**`online` 标志怎么定的**（引擎返回值 + 用户 `LOCK` 子句共同决定）：

```cpp
switch (inplace_supported) {
  case HA_ALTER_INPLACE_NO_LOCK:
  case HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE:
    switch (alter_info->requested_lock) {
      case Alter_info::ALTER_TABLE_LOCK_DEFAULT:
      case Alter_info::ALTER_TABLE_LOCK_NONE:
        ha_alter_info->online = true;     // ★ 只有这两档、且未要求 SHARED/EXCLUSIVE 才 online
        break;
      case Alter_info::ALTER_TABLE_LOCK_SHARED:
      case Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE:
        break;                             // 用户主动要求加锁 → 不 online
    }
    break;
  case HA_ALTER_INPLACE_EXCLUSIVE_LOCK:
  case HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE:
  case HA_ALTER_INPLACE_SHARED_LOCK:
  case HA_ALTER_INPLACE_INSTANT:
    break;
}
```

**降级本身**（prepare 之后、执行之前）：

```cpp
/*
  Downgrade the lock if storage engine has told us that exclusive lock was
  necessary only for prepare phase (unless we are not under LOCK TABLES) and
  user has not explicitly requested exclusive lock.
*/
if ((inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
     inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE) &&
    !(thd->locked_tables_mode == LTM_LOCK_TABLES ||
      thd->locked_tables_mode == LTM_PRELOCKED_UNDER_LOCK_TABLES) &&
    (alter_info->requested_lock != Alter_info::ALTER_TABLE_LOCK_EXCLUSIVE)) {
  /* If storage engine or user requested shared lock downgrade to SNW. */
  if (inplace_supported == HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE ||
      alter_info->requested_lock == Alter_info::ALTER_TABLE_LOCK_SHARED)
    table->mdl_ticket->downgrade_lock(MDL_SHARED_NO_WRITE);   // SNW：可读、禁写
  else {
    assert(inplace_supported == HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE);
    table->mdl_ticket->downgrade_lock(MDL_SHARED_UPGRADABLE); // ★ SU：读写都放行
  }
}
```

★ 三个条件缺一不可才降级：① 引擎返回 `*_AFTER_PREPARE` 系；② 不在 `LOCK TABLES` 模式下（那时降级会破坏用户显式持有的表锁语义）；③ 用户没显式要求 EXCLUSIVE。

★ **降到哪一档**是"INPLACE ≠ online"的直接证据：`SHARED_LOCK_AFTER_PREPARE` 降到 `MDL_SHARED_NO_WRITE`（**可读但禁写**），只有 `NO_LOCK_AFTER_PREPARE` 才降到 `MDL_SHARED_UPGRADABLE`（**读写都放行**，即真正的 online）。引擎返回中间几档时，降完仍是 SNW，写操作照样被挡。

**升回 X**（执行完、提交前）：

```cpp
// Upgrade to EXCLUSIVE before commit.
if (wait_while_table_is_used(thd, table, HA_EXTRA_PREPARE_FOR_RENAME))
  goto rollback;
```

★ 注意它调的是 **`wait_while_table_is_used`** 而非 `mdl_ticket->upgrade_lock()`——server 侧要等的是"所有还开着这张表的线程退出"。这才是"DDL 收尾被长事务堵住"的**真正来源**（MDL 等待只是其中一环），也是 [`inplace_ddl.md`](inplace_ddl.md) 那个"第二个短暂阻塞点"在 server 侧的落点。

**为什么必须"先升回 X 再切换"**：见「online DDL 与 row log」章——核心是"切换哪个结构对外可见"必须排他；且第二次 apply 要"追平"（`head.total == tail.total`），只要 DML 还在写 row log 就永远追不平。

**Online DDL 并不是绝对不锁表**：如果第 ① 步拿不到 MDL X（表上有慢查询/长事务），后续**所有**操作（包括 SELECT）都会被阻塞，表现为典型的"锁表"。所以线上 DDL 仍要避开长事务。

#### Online 的语义边界

**server → handler 的"要干什么"位图**：`Alter_inplace_info::HA_ALTER_FLAGS`（`sql/handler.h:3191`，50 个标志位）。摘关键几个：

```cpp
  // Add non-unique, non-primary index
  static const HA_ALTER_FLAGS ADD_INDEX = 1ULL << 0;
  static const HA_ALTER_FLAGS DROP_INDEX = 1ULL << 1;
  static const HA_ALTER_FLAGS ADD_UNIQUE_INDEX = 1ULL << 2;
  static const HA_ALTER_FLAGS DROP_UNIQUE_INDEX = 1ULL << 3;
  static const HA_ALTER_FLAGS ADD_PK_INDEX = 1ULL << 4;
  static const HA_ALTER_FLAGS DROP_PK_INDEX = 1ULL << 5;

  // Add column
  static const HA_ALTER_FLAGS ADD_VIRTUAL_COLUMN = 1ULL << 6;
  static const HA_ALTER_FLAGS ADD_STORED_BASE_COLUMN = 1ULL << 7;   // ★ 普通 ADD COLUMN
  static const HA_ALTER_FLAGS ADD_STORED_GENERATED_COLUMN = 1ULL << 8;
  static const HA_ALTER_FLAGS ADD_COLUMN =
      ADD_VIRTUAL_COLUMN | ADD_STORED_BASE_COLUMN | ADD_STORED_GENERATED_COLUMN;

  static const HA_ALTER_FLAGS DROP_VIRTUAL_COLUMN = 1ULL << 9;
  static const HA_ALTER_FLAGS DROP_STORED_COLUMN = 1ULL << 10;
  static const HA_ALTER_FLAGS ALTER_COLUMN_NAME = 1ULL << 11;
  static const HA_ALTER_FLAGS ALTER_STORED_COLUMN_TYPE = 1ULL << 13;
  static const HA_ALTER_FLAGS ALTER_COLUMN_EQUAL_PACK_LENGTH = 1ULL << 14;
  static const HA_ALTER_FLAGS ALTER_COLUMN_NOT_NULLABLE = 1ULL << 18;
  static const HA_ALTER_FLAGS ALTER_COLUMN_DEFAULT = 1ULL << 19;
  static const HA_ALTER_FLAGS RECREATE_TABLE = 1ULL << 37;   // FORCE / ENGINE / OPTIMIZE
  static const HA_ALTER_FLAGS ADD_SPATIAL_INDEX = 1ULL << 38;
  static const HA_ALTER_FLAGS ALTER_COLUMN_VISIBILITY = 1ULL << 49;
```

**★ InnoDB 自己的六组 flag 白名单**（`handler0alter.cc:109`）——这是"Online 语义边界"的源码定义：

```cpp
/** Operations for creating secondary indexes (no rebuild needed) */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ONLINE_CREATE =
    Alter_inplace_info::ADD_INDEX | Alter_inplace_info::ADD_UNIQUE_INDEX |
    Alter_inplace_info::ADD_SPATIAL_INDEX;

/** Operations for rebuilding a table in place */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ALTER_REBUILD =
    Alter_inplace_info::ADD_PK_INDEX | Alter_inplace_info::DROP_PK_INDEX |
    Alter_inplace_info::CHANGE_CREATE_OPTION
    /* CHANGE_CREATE_OPTION needs to check innobase_need_rebuild() */
    | Alter_inplace_info::ALTER_COLUMN_NULLABLE |
    Alter_inplace_info::ALTER_COLUMN_NOT_NULLABLE |
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN
    /* ADD_STORED_BASE_COLUMN needs to check innobase_need_rebuild() */
    | Alter_inplace_info::RECREATE_TABLE;

/** Operations for altering a table that InnoDB does not care about */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INPLACE_IGNORE =
    Alter_inplace_info::ALTER_COLUMN_DEFAULT |
    Alter_inplace_info::ALTER_COLUMN_COLUMN_FORMAT |
    Alter_inplace_info::ALTER_COLUMN_STORAGE_TYPE |
    Alter_inplace_info::ALTER_RENAME | Alter_inplace_info::CHANGE_INDEX_OPTION |
    Alter_inplace_info::ADD_CHECK_CONSTRAINT |
    Alter_inplace_info::DROP_CHECK_CONSTRAINT |
    Alter_inplace_info::SUSPEND_CHECK_CONSTRAINT |
    Alter_inplace_info::ALTER_COLUMN_VISIBILITY;

/** Operation allowed with ALGORITHM=INSTANT */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME |
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN |
    Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER |
    Alter_inplace_info::ADD_STORED_BASE_COLUMN |
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |
    Alter_inplace_info::DROP_STORED_COLUMN;

/** Operations that InnoDB cares about and can perform without rebuild */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ALTER_NOREBUILD =
    INNOBASE_ONLINE_CREATE | INNOBASE_FOREIGN_OPERATIONS |
    Alter_inplace_info::DROP_INDEX | Alter_inplace_info::DROP_UNIQUE_INDEX |
    Alter_inplace_info::RENAME_INDEX | Alter_inplace_info::ALTER_COLUMN_NAME |
    Alter_inplace_info::ALTER_COLUMN_EQUAL_PACK_LENGTH |
    Alter_inplace_info::ALTER_INDEX_COMMENT |
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN |
    Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER |
    Alter_inplace_info::ALTER_COLUMN_INDEX_LENGTH;
```

> ★★ **三个集合的关系**：`IGNORE`（无所谓）∪ `NOREBUILD`（不用重建）∪ `REBUILD`（要重建）= 全部 INPLACE 可达；**不在这个并集里的任何标志 → `HA_ALTER_INPLACE_NOT_SUPPORTED` → COPY**。

**`online` 布尔量在什么条件下被置 false**（即 LOCK=NONE 不可用，只能 LOCK=SHARED）：

```cpp
      if (key_part->field->is_flag_set(AUTO_INCREMENT_FLAG)) {
        /* We cannot assign an AUTO_INCREMENT column values during online ALTER. */
        ha_alter_info->unsupported_reason =
            innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_AUTOINC);
        online = false;                      // ① 新增 AUTO_INCREMENT 列
      }
      if (key_part->field->is_virtual_gcol()) {
        ha_alter_info->unsupported_reason =
            innobase_get_err_msg(ER_UNSUPPORTED_ALTER_ONLINE_ON_VIRTUAL_COLUMN);
        online = false;                      // ② 新增虚拟列上建索引
      }
  if (ha_alter_info->handler_flags & Alter_inplace_info::ADD_SPATIAL_INDEX) {
    ha_alter_info->unsupported_reason =
        innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_GIS);
    online = false;                          // ③ 加空间索引
  }
    for (uint i = 0; i < ha_alter_info->index_add_count; i++) {
      const KEY *key = &ha_alter_info->key_info_buffer[ha_alter_info->index_add_buffer[i]];
      if (key->flags & HA_FULLTEXT) {
        ha_alter_info->unsupported_reason =
            innobase_get_err_msg(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_FTS);
        online = false;                      // ④ 建 FULLTEXT 索引
        break;
      }
    }
  } else if (((ha_alter_info->handler_flags & Alter_inplace_info::ADD_PK_INDEX) ||
              innobase_need_rebuild(ha_alter_info)) &&
             (innobase_fulltext_exist(altered_table) ||
              innobase_spatial_exist(altered_table))) {
    online = false;                          // ⑤ 需 rebuild 且表上有 FTS/SPATIAL
    if (m_prebuilt->table->fts) {
      ha_alter_info->unsupported_reason = innobase_get_err_msg(ER_INNODB_FT_LIMIT);
      return HA_ALTER_INPLACE_NOT_SUPPORTED;  // ★ 已有 FTS 还要 rebuild → 直接 COPY
    }
  }
```

**`unsupported_reason` 的作用**（`sql/handler.cc:5013`）——决定报错信息的详细程度：

```cpp
void Alter_inplace_info::report_unsupported_error(const char *not_supported,
                                                  const char *try_instead) {
  if (unsupported_reason == nullptr)
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED, MYF(0), not_supported, try_instead);
  else
    my_error(ER_ALTER_OPERATION_NOT_SUPPORTED_REASON, MYF(0), not_supported,
             unsupported_reason, try_instead);
}
```

> ★ 这就是"为什么有的 ALTER 只说 `ALGORITHM=INPLACE is not supported`，有的却给出具体原因（如 `Cannot change column type INPLACE`）"——取决于 InnoDB 有没有设 `unsupported_reason`。

**INSTANT 的降级清单**（`handler0alter.cc:1078`）——显式 `ALGORITHM=INPLACE` 会阻止降级为 INSTANT：

```cpp
      case Instant_Type::INSTANT_ADD_DROP_COLUMN:
        if (ha_alter_info->alter_info->requested_algorithm ==
            Alter_info::ALTER_TABLE_ALGORITHM_INPLACE) {
          /* Still fall back to INPLACE since the behaviour is different */
          break;                             // ① 显式 INPLACE → 不降级
        } else if (!((m_prebuilt->table->n_def + get_num_cols_added(ha_alter_info))
                     < REC_MAX_N_FIELDS)) {
          ... break;                         // ② 字段数超限 → INPLACE
        } else if (!is_valid_row_version(m_prebuilt->table->current_row_version + 1)) {
          my_error(ER_INNODB_MAX_ROW_VERSION, MYF(0), ...);
          break;                             // ③ row_version 溢出 → INPLACE
        } else if (!Instant_ddl_impl<dd::Table>::is_instant_add_drop_possible(...)) {
          my_error(ER_INNODB_INSTANT_ADD_DROP_NOT_SUPPORTED_MAX_SIZE, MYF(0));
          break;                             // ④ 行大小可能超限 → INPLACE
        } else if (ha_alter_info->error_if_not_empty) {
          /* In this case, it can't be instant because the table may not be empty. */
          break;                             // ⑤ 表可能非空 → INPLACE
        }
        [[fallthrough]];
      case Instant_Type::INSTANT_NO_CHANGE:
      case Instant_Type::INSTANT_VIRTUAL_ONLY:
      case Instant_Type::INSTANT_COLUMN_RENAME:
        ha_alter_info->handler_trivial_ctx = instant_type_to_int(instant_type);
        return HA_ALTER_INPLACE_INSTANT;
```

> ★ `handler_trivial_ctx` 是 `Alter_inplace_info` 里专门给 handler 存"轻量上下文"的 `uint`（`sql/handler.h`），InnoDB 用它存 `Instant_Type`。这也是 INSTANT 判定"没有独立函数"的证据。

"Online" 有严格含义——**DDL 执行期间并发 DML 能否进来**（不阻塞业务读写）。它和"用哪种算法"是**两个独立维度**：

| 算法 | 是否 online | 原因 |
|---|---|---|
| COPY | **必然不 online** | 全程 MDL X，DML 进不来 |
| INPLACE | **不一定** | 取决于引擎的六档返回值（见上表） |
| INSTANT | **必然 online** | 只改元数据，prepare/main 是 no-op，SU 锁下允许并发读写 |

**结论一：`INPLACE ≠ online`。** INPLACE 说的是"不建 server 层临时表、在引擎内就地做"；online 说的是"并发 DML 能不能进"，两回事。引擎返回 `EXCLUSIVE_LOCK` / `SHARED_LOCK` / `SHARED_LOCK_AFTER_PREPARE` 时，操作**仍然是 INPLACE，但照样阻塞**。所以 `ALTER TABLE ... ALGORITHM=INPLACE` 完全可能锁住业务（典型如某些改列操作）。

**结论二：row log 只服务于 INPLACE rebuild，不是 online 的必需品。** 它的作用是"重建表期间把并发 DML 转发到新表"，启用条件是 `ctx->online && ctx->need_rebuild()`。因此：
- COPY 不需要——DML 根本进不来，没有转发的必要；
- **INSTANT 不需要**——源码有条断言 `ut_ad(!(ctx->need_rebuild() && is_instant(ha_alter_info)))`，即 **instant 与 rebuild 互斥**；INSTANT 不搬数据、不建新表，压根不存在"旧表写入要转发给新表"的问题。

**结论三：online 不等于"全程不加锁"。** 即便 online，头尾仍各有一个短暂排他阶段（① prepare 拿 X、③ commit 升回 X）。③ 的阻塞不可避免：切换哪个结构对外可见必须排他，且第二次 apply 要"追平"（`head.total == tail.total`）——只要 DML 还在往 row log 里写，就永远追不平。

> ⚠️ **一个极易读反的点**：[`inplace_ddl.md`](inplace_ddl.md) 里讲"做过 instant add column 的表，row log 记录这类表的 record 时需额外记 info bits"——那**不是说 INSTANT 用 row log**，而是行格式被 instant 改过之后，**后续**再做 online DDL 时 row log 的格式要跟着扩展（行记录头多了 `INSTANT_FLAG`，temp 记录解析要靠它判断）。

### COPY 与 INPLACE 表重建（rebuild）为何"像"

它们**确实像**——结果形态都是"建一张新表 → 搬数据 → rename 替换"，都需要额外空间：

| | COPY | INPLACE 表重建（rebuild） |
|---|---|---|
| 执行层 | server 层 | InnoDB 层 |
| 数据搬运 | `copy_data_between_tables` 逐行走 SQL 层 | 扫描聚簇索引 + `ddl::Builder` bulk load 构建索引树 |
| DML | **全程阻塞** | **online**（靠 row log 转发） |
| row log | 不用（DML 进不来） | 必须用 |
| 临时文件 | server 层 `#sql-xxxx` | InnoDB 层 `ibXXXXXX`（在 `--tmpdir` 下） |
| rename | 有 | 有（都换了表） |
| 额外空间 | 需要 | **也需要**（会建 InnoDB 临时数据文件） |

**关键认知**：INPLACE 描述的是"**表**"而不是"数据文件"——只要不创建 server 层临时表就是 INPLACE。但很多 INPLACE DDL 仍会重建表（增加主键、重建主键、删列、调整列顺序、改 ROW_FORMAT、OPTIMIZE 等），都要创建 InnoDB 临时数据文件、需要额外空间。

所以"像"在于结果形态（换表 + 额外空间），区别在于**搬运方式 + 是否 online**。

### ★ 算法判定的真正链条（纠正一个常见误解）

⚠️ **容易搞错的一点**：`mysql_compare_tables()`（名字很像"决定走哪条路"的函数）**其实完全不参与 ALTER 的算法判定**。它的**全仓唯一调用点是 `sql/sql_partition_admin.cc`（EXCHANGE PARTITION）**，失败报 `ER_TABLES_DIFFERENT_METADATA`。

**真正的判定链**是：

```
mysql_prepare_alter_table()          ← 把 ALTER 意图翻译成 handler_flags（Alter_inplace_info 位集）
        ↓
is_inplace_alter_impossible()        ← SQL 层一票否决（有些操作天生不能 inplace）
        ↓
handler::check_if_supported_inplace_alter()   ← 引擎层拍板（sql/handler.cc 有默认实现，InnoDB 有覆盖版）
        ↓
HA_ALTER_INPLACE_INSTANT / NOT_SUPPORTED / ...
```

**`IS_EQUAL_*` 只有三个宏**（`sql/field.h`）：

```c
#define IS_EQUAL_NO            0
#define IS_EQUAL_YES           1
#define IS_EQUAL_PACK_LENGTH   2
```

`Field::is_equal()` 的文档注释原文：*\"IS_EQUAL_YES if there is no change.\"* / *\"IS_EQUAL_PACK_LENGTH if the data are unchanged, but the length requirements have changed\"* / *\"IS_EQUAL_NO if there is an incompatible change requiring copy.\"*

⚠️ 不存在 `IS_EQUAL_GEOM`（全仓 0 命中）；而且这三个是**枚举值不是位标志**，`changes |=` 只是累加。

**第一步**（`mysql_prepare_alter_table`）：

```
case IS_EQUAL_NO:              → ALTER_VIRTUAL_COLUMN_TYPE / ALTER_STORED_COLUMN_TYPE
case IS_EQUAL_YES:             → 不置任何 flag（注释：Such a change ... can ALWAYS be carried out
                                  by simply updating data-dictionary without even informing
                                  storage engine.）
case IS_EQUAL_PACK_LENGTH:     → 若字符集变了 且 该列在索引里或在分区表达式里
                                     → ALTER_STORED_COLUMN_TYPE
                                   否则 → Alter_inplace_info::ALTER_COLUMN_EQUAL_PACK_LENGTH
```

**第二步**（`handler::check_if_supported_inplace_alter`，INSTANT 的判定就在这）：

```c
  uint table_changes = (ha_alter_info->handler_flags &
                        Alter_inplace_info::ALTER_COLUMN_EQUAL_PACK_LENGTH)
                           ? IS_EQUAL_PACK_LENGTH
                           : IS_EQUAL_YES;
  if (table->file->check_if_incompatible_data(create_info, table_changes) == COMPATIBLE_DATA_YES)
    return HA_ALTER_INPLACE_INSTANT;
  return HA_ALTER_INPLACE_NOT_SUPPORTED;
```

★ **各引擎对 `table_changes` 的容忍度不同**——这就是"VARCHAR 扩长能不能 INSTANT"的真正分歧点：

| 引擎 | 对 `IS_EQUAL_PACK_LENGTH` |
|---|---|
| **InnoDB** | `if (table_changes != IS_EQUAL_YES) { ... }` ⇒ **连 PACK_LENGTH 也不接受**（所以 InnoDB 扩 VARCHAR 不是 INSTANT） |
| MyISAM / Heap | `table_changes == IS_EQUAL_NO \|\| table_changes & IS_EQUAL_PACK_LENGTH` → 不兼容（注释 `// Not implemented yet`） |
| Archive | `table_changes != IS_EQUAL_YES` |

#### 附：`mysql_compare_tables()` 的职责

⚠️ **勘误（重要）**：**`mysql_compare_tables()` 不是 ALTER 算法判定的核心**。它**只服务 `ALTER TABLE ... EXCHANGE PARTITION`**，全仓库唯一调用点是 `sql/sql_partition_admin.cc:248`。ALTER 真正的差异计算是 **`fill_alter_inplace_info()`**（`sql_table.cc:11933`）。

而且它**返回值是 `bool`（true=出错）**，三态通过**出参 `*metadata_equal`** 输出；`IS_EQUAL_*` 是局部变量 `changes` 由 `Field::is_equal()` 逐字段累积的位掩码。

```cpp
/**
  Compare two tables to see if their metadata are compatible.
  One table specified by a TABLE instance, the other using Alter_info
  and HA_CREATE_INFO.
  ...
  @retval true   error
  @retval false  success
*/
bool mysql_compare_tables(THD *thd, TABLE *table, Alter_info *alter_info,
                          HA_CREATE_INFO *create_info, bool *metadata_equal) {
  uint changes = IS_EQUAL_NO;          // ★ 累加器，初值 0
  *metadata_equal = false;

  /*
    Create a copy of alter_info.
    To compare definitions, we need to "prepare" the definition - transform it
    from parser output to a format that describes the table layout...
    Since we would like to keep mysql_compare_tables() idempotent (not altering
    any of the arguments) we create a copy of alter_info here ...
  */
  Alter_info tmp_alter_info(*alter_info, thd->mem_root);

  if (mysql_prepare_create_table(thd, "", "", create_info, &tmp_alter_info,
                                 table->file, false, &key_info_buffer, &key_count,
                                 &fk_key_info_buffer, &fk_key_count, nullptr, 0,
                                 nullptr, 0, 0, false))
    return true;

  /* Some very basic checks. */
  if (table->s->fields != alter_info->create_list.elements ||
      table->s->db_type() != create_info->db_type || table->s->tmp_table ||
      (table->s->row_type != create_info->row_type))
    return false;                      // ★ 列数/引擎/临时表/ROW_TYPE 任一不同 → 不等

  tmp_new_field_it.init(tmp_alter_info.create_list);
  for (Field **f_ptr = table->field; *f_ptr; f_ptr++) {
    Field *field = *f_ptr;
    const Create_field *tmp_new_field = tmp_new_field_it++;

    if (tmp_new_field->is_virtual_gcol() != field->is_virtual_gcol()) {
      my_error(ER_UNSUPPORTED_ACTION_ON_GENERATED_COLUMN, MYF(0),
               "Exchanging partitions for non-generated columns");
      return false;
    }
    /* Check that NULL behavior is the same. */
    if ((tmp_new_field->flags & NOT_NULL_FLAG) != field->is_flag_set(NOT_NULL_FLAG))
      return false;
    /* Check if field was renamed */
    if (my_strcasecmp(system_charset_info, field->field_name, tmp_new_field->field_name))
      return false;

    uint field_changes = field->is_equal(tmp_new_field);   // ★★ 三态来源
    if (field_changes != IS_EQUAL_YES) return false;       // ★★ EXCHANGE 只接受完全相等
    changes |= field_changes;
  }

  /* Check if changes are compatible with current handler. */
  if (table->file->check_if_incompatible_data(create_info, changes))
    return false;                      // ★ 交给 handler 复核（InnoDB: ha_innodb.cc:20234）
  ...  // 双向索引名集合 + 索引 algorithm/flags/user_defined_key_parts/每部分 length 与 fieldnr
  *metadata_equal = true;
  return false;
}
```

**`IS_EQUAL_*` 三态**（`sql/field.h:250`）：

```cpp
/*
  Some defines for exit codes for ::is_equal class functions.
*/
#define IS_EQUAL_NO 0
#define IS_EQUAL_YES 1
#define IS_EQUAL_PACK_LENGTH 2
```

| 值 | 含义 | `mysql_compare_tables` 中 | `fill_alter_inplace_info` 中 |
|---|---|---|---|
| `IS_EQUAL_NO` (0) | 类型不兼容，必须改数据 | 拒绝 | → `ALTER_*_COLUMN_TYPE` |
| `IS_EQUAL_YES` (1) | 完全相同或元数据兼容（如 ENUM 加成员） | **唯一接受** | 不设任何标志 |
| `IS_EQUAL_PACK_LENGTH` (2) | 打包长度兼容（VARCHAR(300)→VARCHAR(400)） | **也拒绝** | → `ALTER_COLUMN_EQUAL_PACK_LENGTH` |

**★ `fill_alter_inplace_info()` 里三态的翻译**（`sql_table.cc:12049`）——这才是"算法判定"真正的差异计算：

```cpp
      switch (field->is_equal(new_field)) {
        case IS_EQUAL_NO:
          /* New column type is incompatible with old one. */
          if (field->is_virtual_gcol())
            ha_alter_info->handler_flags |= Alter_inplace_info::ALTER_VIRTUAL_COLUMN_TYPE;
          else
            ha_alter_info->handler_flags |= Alter_inplace_info::ALTER_STORED_COLUMN_TYPE;
          break;
        case IS_EQUAL_YES:
          /*
            New column is the same as the old one or the fully compatible with
            it (for example, ENUM('a','b') was changed to ENUM('a','b','c')).
            Such a change if any can ALWAYS be carried out by simply updating
            data-dictionary without even informing storage engine.
            No flag is set in this case.
          */
          break;
        case IS_EQUAL_PACK_LENGTH:
          /*
             New column type differs from the old one, but has compatible packed
             data representation. ... (for example, VARCHAR(300) is changed to
             VARCHAR(400)).
             If the collation has changed, and the column is part of an index
             or it is part of a partition expression, we must mark this as a
             change in stored column type ...
           */
          if (field->charset() != new_field->charset &&
              (is_field_part_of_index(*field, *new_field, ha_alter_info) ||
               is_field_part_of_partition_expression(*field, ha_alter_info))) {
            ha_alter_info->handler_flags |= Alter_inplace_info::ALTER_STORED_COLUMN_TYPE;
          } else {
            ha_alter_info->handler_flags |= Alter_inplace_info::ALTER_COLUMN_EQUAL_PACK_LENGTH;
          }
          break;
      }
```

> ★★ **`IS_EQUAL_YES` 的"不设任何标志"是 INSTANT 能成立的基础**——`handler_flags == 0` 时 `mysql_alter_table()` 会走 no-op 短路（`sql_table.cc:17425`）：
> ```cpp
>     if (ha_alter_info.handler_flags == 0) {
>       /* No-op ALTER, no need to call handler API functions. ... */
>       is_noop = true;
>       goto end_inplace_noop;
>     }
> ```

- **唯一真正输出**是一个 `bool *metadata_equal`，**没有位掩码输出**；内部的 `uint changes` 是局部变量，只喂给 `check_if_incompatible_data()`
- **幂等设计**（函数开头注释，值得原文引用）：因为 `mysql_prepare_create_table()` 是**破坏性**的（会"in-place"修改参数：初始化所有列默认值、去掉重复列、把 parser 的 key 定义转成 `KEY`/`KEY_PART_INFO` 数组、回写 `create_info->used_fields`），为了让 `mysql_compare_tables()` 保持幂等，它先 `Alter_info tmp_alter_info(*alter_info, thd->mem_root);` **复制一份**再传给 `mysql_prepare_create_table`
  - ⚠️ **半幂等**：只复制了 `Alter_info`，**没复制 `HA_CREATE_INFO`**——而后者同样会被修改。所以对 `alter_info` 幂等、对 `create_info` **不**幂等
- **比较范围**（容易想多）：
  - ✅ 字段数 / `db_type` / `row_type` / 是否临时表；逐字段比 `NOT_NULL_FLAG`、列名（不允许改名）、`field->is_equal() == IS_EQUAL_YES`
  - ✅ 索引**双向**比较（防一边多）：比 `algorithm`、`flags & HA_KEYFLAG_MASK`、`user_defined_key_parts`，逐个 key_part 比 `length` 与 `fieldnr - 1`
  - ❌ **不做外键比较**（虽然传了 `&fk_key_info_buffer`，但函数里再没读过）
  - ❌ **不做分区比较**（由调用方 EXCHANGE PARTITION 自己处理）
  - ❌ **不做表选项比较**（除 `db_type`/`row_type`/`tmp_table` 三个外，`charset`/`comment`/`row_format`/`auto_increment` 一概不比）
- 一个反直觉点：`if (field_changes != IS_EQUAL_YES) return false;` ⇒ `IS_EQUAL_NO` 和 `IS_EQUAL_PACK_LENGTH` 都直接返回，**这条路径永远不可能把 `IS_EQUAL_PACK_LENGTH` 传给 `check_if_incompatible_data`**

---

## ★ CREATE TABLE / ALTER TABLE：已独立成篇

> 这两条链路体量较大，已从本篇迁出为独立篇。**本篇保留跨操作共享的骨架**（三种算法判定、MDL 三段式、原子性、行为速查与观测），操作链路本身去独立篇看。

### CREATE TABLE → [`create_table.md`](create_table.md)

**server 侧编排**（`mysql_create_table` → `mysql_prepare_create_table` 全部合法性检查 → `rea_create_base_table` DD 写入 → `ha_create_table` 接缝）+ **引擎侧落地**（`ha_innobase::create` → `create_table_info_t` 三段式 → `dict_build_tablespace_for_table` 建 .ibd → `dict_create_index_tree_in_mem` → `btr_create` 建根页 → 统计置空）。

★ 两个反直觉点：① CREATE 也会写 DDL log——`REMOVE_CACHE_LOG`（回滚要摘字典缓存）和建树后的 `FREE_TREE_LOG`；② 新表统计**必然是 0 行**（`DICT_STATS_EMPTY_TABLE`），这正是 **COPY ALTER 之后统计为空**的根因。

### ALTER TABLE → [`alter_table.md`](alter_table.md)

只保留 **server 侧编排主线**（`Alter_info` 意图描述 / `Alter_table_ctx` 执行期上下文 / `mysql_prepare_alter_table`、INPLACE vs COPY 的 server 侧差异、`mysql_recreate_table`、`notify_alter_table`）；算法深水区已分篇：

| 算法 | 去哪看 |
|---|---|
| COPY（`copy_data_between_tables` 逐段、为何一定不 Online） | [`copy_ddl.md`](copy_ddl.md) |
| INPLACE（MDL 升降级、引擎四件套、row log 为何 apply 两次） | [`inplace_ddl.md`](inplace_ddl.md) |
| INSTANT（行版本号三层级、WL#11250 两个坑） | [`instant_ddl.md`](instant_ddl.md) |
| 8.0 新版 DDL 引擎与并行构建 | [`ddl_engine.md`](ddl_engine.md) |

> 接缝归属：`rea_create_base_table` / `ha_create_table` 是 CREATE 与 ALTER 共用的接缝，随 CREATE 主链路放在 [`create_table.md`](create_table.md)，ALTER 侧链接回去。

---

## DROP / TRUNCATE / RENAME（server 侧 + 引擎侧）


### `mysql_rm_table()` 与 `Drop_tables_ctx`

多表 DROP 靠一个**分组状态机**：

```cpp
class Drop_tables_ctx {
  Prealloced_array<Table_ref *, 1> base_atomic_tables;
  Prealloced_array<Table_ref *, 1> base_non_atomic_tables;
  Prealloced_array<Table_ref *, 1> tmp_trans_tables;
  Prealloced_array<Table_ref *, 1> tmp_non_trans_tables;
  Prealloced_array<Table_ref *, 1> nonexistent_tables;
  Prealloced_array<Table_ref *, 1> views;
  ...
  enum { NO_GTID_MANY_TABLE_GROUPS, NO_GTID_SINGLE_TABLE_GROUP,
         GTID_MANY_TABLE_GROUPS, GTID_SINGLE_TABLE_GROUP } gtid_and_table_groups_state;
```

分组维度：**原子/非原子 × 临时/普通 × 事务/非事务 × 存在/不存在 × 视图 × GTID**。

★ 一个很人性化的设计——**无 `IF EXISTS` 且有不存在的表 → 一张都不删**：

> *"This gives us nice 'atomic' (succeed or don't drop anything) behavior for most common failure scenario even for tables which don't support atomic DDL."*

**单表删除顺序**（`drop_base_table()`）：

```cpp
int error = ha_delete_table(thd, hton, path, table->db, table->table_name, table_def, ...);
...
if (adjust_fk_children_for_parent_drop(...) || adjust_fk_parents(...)) return true;
...
bool result = dd::drop_table(thd, table->db, table->table_name, *table_def);
if (!atomic) result = trans_intermediate_ddl_commit(thd, result);
result |= mark_referencing_views_invalid(...);
```

⇒ **先 `ha_delete_table()`（引擎删 .ibd）→ FK 调整 → `dd::drop_table()`（DD 清理）→ 视图失效标记**。

**binlog 与提交顺序**（★ 原子/非原子完全不同）：

| | binlog | 提交 |
|---|---|---|
| **非原子组** | **每删一张就立刻写一条单表 DROP 并 commit** | 逐表 commit |
| **原子组** | 所有表删完后**写一条多表 DROP**，用 trx cache | 一次提交（DD + SE + binlog 原子） |

非原子组"每表一提交"的理由（注释原文）：*"we drop each such table individually and write to binlog a single-table DROP TABLE statement corresponding to this action right after it… This increases chances of SE, data-dictionary and binary log being in sync if crash occurs."*

且**先处理非原子组再处理原子组**：*"in order to avoid situations when DROP TABLES for mixed set of tables will fail and leave changes to atomic, 'transactional' tables around"*

### TRUNCATE 的两条执行路径

由 **`HTON_CAN_RECREATE`** 决定：

```cpp
if (hton->flags & HTON_CAN_RECREATE) {
  /*
    The storage engine can truncate the table by creating an
    empty table with the same structure.
  */
  ...
  if (ha_create_table(thd, path, table_ref->db, table_ref->table_name,
                      &create_info, true, false, table_def) != 0) return;
  // Binlog only if truncate-by-recreate succeeds.
  m_error = false; binlog_stmt = true; return;
}

assert((hton->flags & HTON_CAN_RECREATE) == false);
/*
  The engine does not support truncate-by-recreate.
  Attempt to use the handler truncate method.
*/
const Truncate_result tr = handler_truncate_base(thd, table_ref, table_def);
```

**为什么必须两条**：

- **重建路径**（`HTON_CAN_RECREATE`，InnoDB 有）：用**同一个 `dd::Table`** 重新 `ha_create_table()`，等于"drop + create"但保留 DD 对象与 id；天然纳入原子 DDL 事务
- **handler truncate 路径**：引擎只实现了 `handler::truncate()`（逐行/段级删除），能力更弱，返回 `Truncate_result::{OK, FAILED_BUT_BINLOG, FAILED_SKIP_BINLOG, FAILED_OPEN}`

其一处细节：非事务表 truncate 失败也**必须记 binlog**（注释原文：*"All effects of a TRUNCATE TABLE operation are committed even if truncation fails in the case of non transactional tables."*）

### `mysql_rename_table()`：★ 先 SE 后 DD

这是最反直觉的一点，源码注释解释了为什么改过顺序：

```cpp
int error = file->ha_rename_table(from_base, to_base, from_table_def, to_table_def);
...
/*
  Note that before WL#7743 we have renamed table in the data-dictionary
  before renaming it in storage engine. However with WL#7743 engines
  supporting atomic DDL are allowed to update dd::Table object describing
  new version of table in handler::rename_table(). Hence it should saved
  after this call.
  So to avoid extra calls to DD layer and to keep code simple the
  renaming of table in the DD was moved past rename in SE for all SEs.
  ...
*/
bool result = thd->dd_client()->update(to_table_def);
```

⇒ **因为引擎可以在 `rename_table()` 里改 `dd::Table`，所以 DD 的 update 必须放在 SE rename 之后。**

**flag 体系**（server 侧复用同一个函数做多种"改名"的关键）：

| flag | 含义 |
|---|---|
| `FN_TO_IS_TMP` / `FN_FROM_IS_TMP` | 目标/源是临时名（`#sql-` 前缀） |
| `NO_DD_COMMIT` | 不中间提交，留给调用方（原子事务） |
| `NO_FK_RENAME` / `NO_CC_RENAME` | 不改 FK 名 / check 约束名（临时改名阶段用，避免名字冲突） |
| `NO_FK_CHECKS` | 临时关闭外键检查 |

```cpp
/*
  Only SEs which support atomic DDL are allowed not to commit
  changes to the data-dictionary.
*/
assert(!(flags & NO_DD_COMMIT) || (base->flags & HTON_SUPPORTS_ATOMIC_DDL));
```

---


### DROP/TRUNCATE 不记数据 undo 的原因

**源码证据 ①：`ha_delete_table()` 调用点前后没有任何 undo 相关调用**（`sql_table.cc:2968`）：

```cpp
  int error = ha_delete_table(thd, hton, path, table->db, table->table_name,
                              table_def, !drop_ctx.drop_database);
  // 【注】单纯一个 SE 删除调用；没有 row_undo / trx_undo_* / undo_no 之类的配套
```

**源码证据 ②：InnoDB 侧走的是"DDL 事务"，标记为 `TRX_DICT_OP_TABLE`**（`row0mysql.cc`）：

```cpp
  switch (trx_get_dict_operation(trx)) {
    case TRX_DICT_OP_NONE:
      trx_set_dict_operation(trx, TRX_DICT_OP_TABLE);   // ★ 字典操作，不是普通 DML
    case TRX_DICT_OP_TABLE:
      break;
    case TRX_DICT_OP_INDEX:
      ut_ad(strstr(table->name.m_name, "/fts_") != nullptr || ...);
  }
```

**源码证据 ③：崩溃安全由 DDL log 承担，不靠 undo**——见"三种 DDL log 的决策矩阵"。`Log_DDL::skip()` 里对临时表直接返回 `DB_SUCCESS`（临时表不需要崩溃安全）。

**源码证据 ④：TRUNCATE 通过"文件 rename + 整文件删除"绕过逐行删除**（`ha_innodb.cc:14560`）——它不 DELETE 每一行，而是：
- 把 `t.ibd` rename 成临时名（写 `RENAME_SPACE_LOG` + redo `MLOG_FILE_RENAME`）
- drop 旧表（写 `DELETE_SPACE_LOG` 指向临时文件）
- create 新表（新 space）

> ★★ **真正的答案**：DROP/TRUNCATE 的"回滚"**不是把数据恢复出来，而是把"文件还在"这件事恢复**。`DELETE_SPACE_LOG` 记录的正是"要删哪个文件"——提交后 `post_ddl()` 才真正删文件；若崩溃在提交前，replay 时按记录删；若崩溃在提交后但删文件前，replay 也会删。**数据从未进入 undo log，因为它根本不需要被"撤销到某个旧版本"，只需要"要么整个文件在，要么整个文件不在"**。
>
> ★ 这也是为什么 **`DROP TABLE` 在大表上很快**：它不逐行写 undo，只做 (a) dict cache 摘除、(b) `fil_delete_tablespace()` 整文件删除、(c) 几条 DDL log 插入。
>
> ⚠️ **唯一的例外是"非 file-per-table"**：表在系统/共享/通用表空间时不能删整个文件，只能 `btr_free_if_exists()` 逐页释放 B 树——所以写 `FREE_TREE_LOG`（每索引一条），且**这个过程是 redo 保护的**（`mtr_start` / `mtr_commit`），仍然不写 undo。

**`row_drop_table_for_mysql()` 里"表还被占用"的兜底**（`row0mysql.cc:3960`）——这是唯一会"延迟删除"的场景：

```cpp
  if (table->n_foreign_key_checks_running > 0) {
    auto added = row_add_table_to_background_drop_list(save_tablename);
    if (added) {
      ib::info(ER_IB_MSG_993) << "You are trying to drop table " << table->name
                              << " though there is a foreign key check"
                                 " running on it. Adding the table to the"
                                 " background drop queue.";
      /* We return DB_SUCCESS to MySQL though the drop will happen lazily later */
      err = DB_SUCCESS;
    } else {
      err = DB_ERROR;
    }
    goto funct_exit;
  }
```

> ★ 所以"DROP TABLE 卡住"一种可能是表进了 **background drop list**，由后台线程延迟删。


DROP TABLE 和 TRUNCATE TABLE **不记录用户数据的 undo log**——它们不像 `DELETE FROM t` 那样为每一行产生 undo record。原因：

1. **没有逐行 delete**：DROP/TRUNCATE 是物理操作（删表空间文件、free btree root），不是逐行删除，没有逐行 undo 的产生点
2. **不需要 MVCC 旧版本**：被删或被清空的表数据不需要一致性读的历史版本——表没了或被重建了，没有 read view 会访问旧数据
3. **DDL 隐式提交不可回滚**：DDL 语句本身是原子事务，用户层面不可回滚，不存在"回滚到 DDL 前"的需求

因此 DDL 的原子性不靠数据 undo，靠 DDL log + dd 元数据 undo（详见下文）。

---


### DROP TABLE 的 InnoDB 实现

**入口链**：`ha_innobase::delete_table()` → `innobase_basic_ddl::delete_impl()` → `row_drop_table_for_mysql()`

```cpp
// ha_innodb.cc:15337
int ha_innobase::delete_table(const char *name, const dd::Table *table_def) {
  if (table_def != nullptr &&
      dict_sys_t::is_dd_table_id(table_def->se_private_id())) {
    my_error(ER_NOT_ALLOWED_COMMAND, MYF(0));
    return HA_ERR_UNSUPPORTED;                 // ★ DD 表不允许删
  }
  THD *thd = ha_thd();
  trx_t *trx = check_trx_exists(thd);
  if (table_def != nullptr && table_def->is_persistent()) {
    innobase_register_trx(ht, thd, trx);
  }
  return innobase_basic_ddl::delete_impl(thd, name, table_def, nullptr);
}
```

```cpp
// ha_innodb.cc:14228 —— 关键两步
  error = row_drop_table_for_mysql(norm_name, trx, true, handler);
  //                                              ★ nonatomic=true

  if (error == DB_SUCCESS && file_per_table) {
    dd::Object_id dd_space_id = dd_first_index(dd_tab)->tablespace_id();
    if (dd_drop_tablespace(client, dd_space_id)) error = DB_ERROR;
    //  ★ InnoDB 只删自己的；DD 侧 tablespace 记录由 server 层 dd_drop_tablespace 删
  }
```

**`row_drop_table_for_mysql()` 的关键步骤**（`row0mysql.cc:3771`，387 行）：

1. **锁字典** `row_mysql_lock_data_dictionary(trx)` → `locked_dictionary = true; nonatomic = true;`
2. **打开表**：先查 dict cache（`dict_table_check_if_in_cache_low`），miss 则 `dd_table_open_on_name(..., DICT_ERR_IGNORE_INDEX_ROOT | DICT_ERR_IGNORE_CORRUPT）`
3. **FTS aux 表加 X 锁**：`fts_lock_all_aux_tables()`
4. **等待后台统计停**：`dict_stats_wait_bg_to_stop_using_table()`
5. **清统计**：`dict_stats_drop_table()`
6. **检查 FK / 表锁**：有并发 FK check → 进 background drop list 延迟删
7. **清 AHI + 从 dict cache 摘除**：`row_drop_table_from_cache()`
8. **写 DDL log**（决策矩阵见上节）
9. **`row_drop_tablespace()` 删文件**（replay 时）

```cpp
static inline dberr_t row_drop_table_from_cache(dict_table_t *table, trx_t *trx) {
  /* Remove the pointer to this table object from the list
  of modified tables by the transaction because the object
  is going to be destroyed below. */
  trx->mod_tables.erase(table);

  if (!table->is_intrinsic()) {
    btr_drop_ahi_for_table(table);        // ★ 先清自适应哈希
    dict_table_remove_from_cache(table);  // ★ 从 dict cache 摘除
  } else {
    while (dict_index_t *index = UT_LIST_GET_FIRST(table->indexes)) {
      rw_lock_free(&index->lock);
      UT_LIST_REMOVE(table->indexes, index);
      dict_mem_index_free(index);
    }
    dict_mem_table_free(table);
  }
  return DB_SUCCESS;
}
```

```cpp
/** Drop a tablespace as part of dropping or renaming a table.
This deletes the fil_space_t if found and the file on disk. */
dberr_t row_drop_tablespace(space_id_t space_id, const char *filepath) {
  /* If the tablespace is not in the cache, just delete the file. */
  if (!fil_space_exists_in_mem(space_id, nullptr, true, false)) {
    if (fil_delete_file(filepath)) { ... }      // ★ 直接 unlink
  } else {
    err = fil_delete_tablespace(space_id, BUF_REMOVE_NONE);   // ★ 整文件删除
  }
  return err;
}
```

> ⚠️ **`dict_table_remove_from_cache()` 在 `dict0dict.cc:1974`，不在 `dict0crea.cc`**；`dict_index_remove_from_cache()` 在 `dict0dict.cc:2727`。
>
> ⚠️ **`ddl::drop_table()` 是另一条路**（`ddl0ddl.cc:291`），只被 `dict_table_close_and_drop()` 调用（内部清理，如 FTS aux 表）：
> ```cpp
> dberr_t drop_table(trx_t *trx, dict_table_t *table) noexcept {
>   ut_ad(!srv_read_only_mode);
>   /* There must be no open transactions on the table. */
>   ut_a(table->get_ref_count() == 0);
>   return row_drop_table_for_mysql(table->name.m_name, trx, false, nullptr);
>   //                                            ★ nonatomic=false
> }
> ```
> ⚠️ **`ddl::drop_tablespace()` 不存在**——`namespace ddl` 里只有 `drop_table()` 与 `drop_indexes()`。

**`ddl::drop_indexes()` —— 表还被占用时的降级**（`ddl0ddl.cc:468`）：

```cpp
void drop_indexes(trx_t *trx, dict_table_t *table, bool locked) noexcept {
  ut_ad(rw_lock_own(dict_operation_lock, RW_LOCK_X));
  /* the caller should have an open handle to the table */
  ut_ad(table->get_ref_count() >= 1);

  if (!locked && table->get_ref_count() > 1) {
    /* Mark the secondary indexes as aborted/offline. */
    mark_secondary_indexes(trx, table);   // ★ 只打标记，等下次 close 或 crash recovery 再删
  } else {
    drop_secondary_indexes(trx, table);   // ★ 直接删
  }
}
```

**★ `fil_delete_tablespace()` 与 `fil_discard_tablespace()` 的区别**（`fil0fil.cc:4814` 注释原文）：

```cpp
/** Discards a single-table tablespace. The tablespace must be cached in the
memory cache. Discarding is like deleting a tablespace, but

 1. We do not drop the table from the data dictionary;

 2. We remove all insert buffer entries for the tablespace immediately;
    in DROP TABLE they are only removed gradually in the background;

 3. When the user does IMPORT TABLESPACE, the tablespace will have the
    same id as it originally had.

 4. Free all the pages in use by the tablespace if rename=true.
*/
dberr_t fil_discard_tablespace(space_id_t space_id) {
  dberr_t err = fil_delete_tablespace(space_id, BUF_REMOVE_NONE);
  ...
}
```

> ★★ **DISCARD vs DROP 的两点关键差别**：DISCARD **不动 DD**（表定义还在），且**立即清 ibuf**（DROP 是后台渐进清）。这就是 `ALTER TABLE ... DISCARD TABLESPACE` 后还能 `IMPORT` 回来的原因。

**TRUNCATE 时的文件截断**（`Fil_shard::space_truncate()`，`fil0fil.cc:4678`）——注意 InnoDB 的 TRUNCATE 不走这个（走 rename+drop+create），这是给 `Fil_shard` 通用路径用的：

```cpp
  /* Step-2: Mark the tablespace pages in the buffer pool as stale by bumping
   the version number of the space. Those stale pages will be ignored and freed
   lazily later. This includes AHI ... */
  space->bump_version();          // ★ 用版本号让 buffer pool 里的旧页全部失效

  /* Step-3: Truncate the tablespace ... */
  space->size = file.size = size_in_pages;
  bool success = os_file_truncate(file.name, file.handle, 0);
  if (success) {
    os_offset_t size = size_in_pages * UNIV_PAGE_SIZE;
    success = os_file_set_size(file.name, file.handle, 0, size, true);
    if (success) space->stop_new_ops = false;
  }
```


入口 `row_drop_table_for_mysql`（row0mysql.cc:3771），核心步骤：

1. 打开表、加 dict 锁、`table->to_be_dropped = true`（:3874）
2. 移除表上所有锁（`lock_remove_all_on_table` :3974）、停用 stats
3. **写 DDL log**（关键，按表空间类型分三条路）：

```
4028:4127:storage/innobase/row/row0mysql.cc
  if (!table->is_temporary() && !file_per_table) {
    err = log_ddl->write_free_tree_log(trx, index, true);   // 共享表空间：释放索引 btree（每个 index 一条）
  }
  ...
  if (!is_temp) {
    log_ddl->write_drop_log(trx, table_id);                  // 记录 drop（清理 innodb_table_metadata）
  }
  ...
  err = log_ddl->write_delete_space_log(trx, nullptr, space_id, filepath, true, true);  // 独立表空间：删 .ibd
```

4. `index->page = FIL_NULL`（:4049）标记所有索引不可用（逻辑删除）
5. `row_drop_table_from_cache`（:4108）从 dict cache 移除
6. 提交后由 post_ddl 统一 replay

#### ★ 三种 DDL log 的决策矩阵（这段的真正核心）

⚠️ **勘误（源码基线）**：
- **`sql/log_ddl.cc` / `sql/log_ddl.h` 不存在**！全库 `sql/` 下对 `Log_DDL` grep = 0 命中。真正的实现在 **`storage/innobase/include/log0ddl.h` + `storage/innobase/log/log0ddl.cc`**
- **不存在 `enum class ddl_log_action_code`**——真名 `enum class Log_Type : uint32_t`（`log0ddl.h:45`）
- **不存在 `write_drop_table_log()` / `write_drop_space_log()` / `execute_ddl_log_entry()` / `recover_from_crash()` / `disable()`**——真名 `write_drop_log()` / `write_delete_space_log()` / `replay()` / `recover()` / `skip()`

**action type 全集（8 种）**（`log0ddl.h:43`）：

```cpp
/** DDL log types defined as uint32_t because it costs 4 bytes in
mysql.innodb_ddl_log. */
enum class Log_Type : uint32_t {
  /** Smallest log type */
  SMALLEST_LOG = 1,

  /** Drop an index tree */
  FREE_TREE_LOG = 1,          // ★ 值=1：释放某 index 的 B 树（space_id + page_no + index_id）

  /** Delete a file */
  DELETE_SPACE_LOG,           // ★ 值=2：删除 .ibd 文件（space_id + old_file_path）

  /** Rename a file */
  RENAME_SPACE_LOG,           // ★ 值=3：重命名文件（old_file_path → new_file_path）

  /** Drop the entry in innodb_table_metadata */
  DROP_LOG,                   // ★ 值=4：删 innodb_dynamic_metadata 里的表元数据项（table_id）

  /** Rename table in dict cache. */
  RENAME_TABLE_LOG,           // ★ 值=5：dict cache 里改表名（复用 old/new_file_path 存表名）

  /** Remove a table from dict cache */
  REMOVE_CACHE_LOG,           // ★ 值=6：把表从 dict cache 摘掉（table_id + new_file_path 存表名）

  /** Alter Encrypt a tablespace */
  ALTER_ENCRYPT_TABLESPACE_LOG,   // ★ 值=7
  /** Alter Unencrypt a tablespace */
  ALTER_UNENCRYPT_TABLESPACE_LOG, // ★ 值=8

  BIGGEST_LOG = ALTER_UNENCRYPT_TABLESPACE_LOG
};
```

**`mysql.innodb_ddl_log` 表结构**（InnoDB 向 DD 注册，`ha_innodb.cc:13025`）：

```cpp
  dd::Object_table *innodb_ddl_log = dd::Object_table::create_object_table();
  innodb_ddl_log->set_hidden(true);                 // ★ 隐藏表，SHOW TABLES 看不到
  def = innodb_ddl_log->target_table_definition();
  def->set_table_name("innodb_ddl_log");
  def->add_field(0, "id", "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT");
  def->add_field(1, "thread_id", "thread_id BIGINT UNSIGNED NOT NULL");
  def->add_field(2, "type", "type INT UNSIGNED NOT NULL");
  def->add_field(3, "space_id", "space_id INT UNSIGNED");
  def->add_field(4, "page_no", "page_no INT UNSIGNED");
  def->add_field(5, "index_id", "index_id BIGINT UNSIGNED");
  def->add_field(6, "table_id", "table_id BIGINT UNSIGNED");
  def->add_field(7, "old_file_path",
                 "old_file_path VARCHAR(512) COLLATE UTF8MB3_BIN");
  def->add_field(8, "new_file_path",
                 "new_file_path VARCHAR(512) COLLATE UTF8MB3_BIN");
  def->add_index(0, "index_pk", "PRIMARY KEY(id)");
  //  ★ 聚簇索引按 id 单调自增 → replay 时按倒序（后写的先回滚）
  def->add_index(1, "index_k_thread_id", "KEY(thread_id)");
  //  ★ post_ddl 按 thread_id 抓"本线程"全部记录
```

**★ 决策矩阵的源码**（`row0mysql.cc:4028-4127`）：

```cpp
  if (!table->is_temporary() && !file_per_table) {
    dict_sys_mutex_exit();
    for (dict_index_t *index = table->first_index();
         err == DB_SUCCESS && index != nullptr; index = index->next()) {
      err = log_ddl->write_free_tree_log(trx, index, true);
    }
    dict_sys_mutex_enter();
    //  ★★ 只有"非临时 且 非 file-per-table"才逐索引写 FREE_TREE_LOG
    //      因为不能删整个文件，只能逐页把 B 树释放掉
  }
  ...
  err = row_drop_table_from_cache(table, trx);

  if (!is_temp) {
    log_ddl->write_drop_log(trx, table_id);        // ★ 非临时表必写 DROP_LOG
  }

  /* Do not attempt to drop known-to-be-missing tablespaces,
  nor system or shared general tablespaces. */
  if (is_discarded || is_temp || shared_tablespace ||
      fsp_is_system_or_temp_tablespace(space_id)) {
    goto funct_exit;                               // ★ 这 4 类不写 DELETE_SPACE_LOG
  }

  ut_ad(file_per_table);
  err = log_ddl->write_delete_space_log(trx, nullptr, space_id, filepath, true, true);
  //  ★★ 能走到这里的一定是 file-per-table → 整文件删除
```

| 表类型 | FREE_TREE_LOG(1) | DROP_LOG(4) | DELETE_SPACE_LOG(2) | 释放方式 |
|---|---|---|---|---|
| **file-per-table 普通表** | ❌ | ✅ | ✅ | **整文件删除**（`row_drop_tablespace` → `fil_delete_tablespace`） |
| **系统/共享/通用表空间中的表** | ✅（每索引一条） | ✅ | ❌ | **逐页释放**（`btr_free_if_exists`） |
| **临时表** | ❌ | ❌ | ❌ | `skip()` 直接返回 `DB_SUCCESS` |
| **已 DISCARD 的表** | 视是否共享 | ✅ | ❌ | 文件已不在，不删 |
| **CREATE TABLE 回滚路径** | ✅（insert 后立刻 `delete_by_id`） | — | ✅（insert 后立刻 `delete_by_id`） | 提交即失效、回滚才生效 |

**★ 核心技巧："提交即失效，回滚才生效"**（`write_free_tree_log()`，`log0ddl.cc:853`）：

```cpp
  if (is_drop_table) {
    /* Drop index case, if committed, will be redo only */
    err = insert_free_tree_log(trx, index, id, thread_id);
    //  ★ DROP：只 insert，提交后由 post_ddl 真正释放
  } else {
    /* This is the case of building index during create table
    scenario. The index will be dropped if ddl is rolled back */
    err = insert_free_tree_log(nullptr, index, id, thread_id);
    //  ★ CREATE：用独立后台 trx 先写（立即提交，不随用户 trx）

    /* Delete this operation if the create trx is committed */
    err = delete_by_id(trx, id, false);
    //  ★★ 立刻在"用户自己的 trx"里删掉这条记录：
    //     - 用户 trx 提交 → 删除生效 → 记录消失 → post_ddl 找不到 → 不释放
    //     - 用户 trx 回滚 → 删除被撤销 → 记录还在 → replay 时释放 B 树
  }
```

> ★★ 这是整个 DDL log 设计里最精妙的一步：**把"撤销动作"的生命周期绑定到"用户事务"的提交/回滚上**，无需额外的协调者。

**`skip()` 的四种豁免**（`log0ddl.cc:847`）：

```cpp
inline bool Log_DDL::skip(const dict_table_t *table, THD *thd) {
  return (recv_recovery_on ||                              // ① redo 恢复期间不写
          thread_local_ddl_log_replay ||                   // ② 自己正在 replay，禁递归写
          (table != nullptr && table->is_temporary()) ||   // ③ 临时表不需要崩溃安全
          thd_is_bootstrap_thread(thd));                   // ④ bootstrap 阶段不写
}
```

**回放：崩溃恢复 vs 运行时**（`log0ddl.cc:1943` / `:1906`）：

```cpp
dberr_t Log_DDL::recover() {
  if (srv_read_only_mode || srv_force_recovery > 0) return DB_SUCCESS;
  thread_local_ddl_log_replay = true;    // ★ 置位 → skip() 为真，replay 期间不再写新 log
  s_in_recovery = true;
  dberr_t err = replay_all();            // ★ 全表倒序扫描 + 逐条 replay
  thread_local_ddl_log_replay = false;
  s_in_recovery = false;
  return err;
}

dberr_t Log_DDL::post_ddl(THD *thd) {
  if (skip(nullptr, thd)) return DB_SUCCESS;
  ulint thread_id = thd_get_thread_id(thd);
  thread_local_ddl_log_replay = true;
  dberr_t err = replay_by_thread_id(thread_id);   // ★ 只回放"本线程"的记录
  thread_local_ddl_log_replay = false;
  return err;
}
```

```cpp
dberr_t Log_DDL::replay(DDL_Record &record) {
  switch (record.get_type()) {
    case Log_Type::FREE_TREE_LOG:
      replay_free_tree_log(record.get_space_id(), record.get_page_no(),
                           record.get_index_id());
      break;
    case Log_Type::DELETE_SPACE_LOG:
      replay_delete_space_log(record.get_space_id(), record.get_old_file_path());
      break;
    case Log_Type::RENAME_SPACE_LOG:  ... break;
    case Log_Type::DROP_LOG:          replay_drop_log(record.get_table_id()); break;
    case Log_Type::RENAME_TABLE_LOG:  ... break;
    case Log_Type::REMOVE_CACHE_LOG:  ... break;
    case Log_Type::ALTER_ENCRYPT_TABLESPACE_LOG:
    case Log_Type::ALTER_UNENCRYPT_TABLESPACE_LOG:
      err = replay_alter_encrypt_space_log(record); break;
  }
  return err;
}
```

> ⚠️ **持久化表走 `btr_free_if_exists()`，不是 `btr_free()`**——`btr_free()`（`btr0btr.cc:1026`）注释明确是 *"Free an index tree in a **temporary** tablespace"*，且 `MTR_LOG_NO_REDO`。
>
> ```cpp
> void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no,
>                                    ulint index_id) {
>   ...
>   dict_sys_mutex_enter();              // ★ 保护 AHI 的并发清理
>   mtr_start(&mtr);
>   btr_free_if_exists(page_id_t(space_id, page_no), page_size, index_id, &mtr);
>   mtr_commit(&mtr);
>   dict_sys_mutex_exit();
> }
> ```

上面三条写入分支合起来是一张**按表空间类型分岔的决策矩阵**（`row0mysql.cc:4028-4127`）：

| 场景 | `FREE_TREE_LOG`（逐索引） | `DELETE_SPACE_LOG`（整空间） | `DROP_LOG`（表） |
|---|---|---|---|
| **独立表空间**（file_per_table）非临时 | ❌ 不写 | ✅ | ✅ |
| **共享表空间**（system/general）非临时 | ✅ 每个索引一条 | ❌（空间不删） | ✅ |
| **临时表** | ❌ | ❌ | ❌ |

逻辑就是：**独立表空间回滚时直接删整个 .ibd，逐索引 free 是冗余的**（与 CREATE 篇 `dict_create_index_tree_in_mem` 的 FIXME 同一个道理）；共享表空间的 .ibd 是公用的，只能逐棵树 free；临时表崩溃随 tmp 文件消失，一条都不用写。另外 `is_discarded`（`DISCARD TABLESPACE` 之后的表）也不写 `DELETE_SPACE_LOG`——空间早已不在。

还有五个细节决定了这段代码的正确性，光看调用链看不到：

1. ★ **写 `FREE_TREE_LOG` 前先放 `dict_sys` mutex**（:4029 `dict_sys_mutex_exit()` → 写完 :4034 再拿回来）——写 DDL log 是 IO，绝不能持着全局字典锁做；这也是为什么逐索引写要包在这对进出之间。
2. **`index->page = FIL_NULL`**（:4049）在拿 X latch 下改——**摘掉根页号**就是"标记索引不可用"，此后任何访问这个索引的路径都会失败；`is_temporary()` 时还要 `dict_drop_temporary_table_index` 直接清临时页。
3. ★ **`filepath` 必须在 `row_drop_table_from_cache` **之前**算好**（:4096-4105）——一旦表从 dict cache 移除，`dict_table_t` 就没了，`data_dir_path`/表名都拿不到。这就是为什么"收集 space_id/table_id/filepath"这一段要在释放对象前完成。
4. ★ **TRUNCATE 的表名来自 `trunc_name`**（:4089）——TRUNCATE 走 rename 路径时 `.ibd` 文件名已被改名，drop 旧表时要删的是**改名后的文件**；`table->trunc_name.m_name` 取完置 nullptr（所有权转移，funct_exit 统一 free）。
5. **写入顺序 `DROP_LOG` 在前、`DELETE_SPACE_LOG` 在后**（:4115 → :4126）——恢复重放时先清表的元数据残留（`innodb_table_metadata`），再删文件；反过来的话文件没了、元数据记录还在。

`dict operation` 的前置 switch（:4015-4026）也值得一提：`TRX_DICT_OP_NONE → 置为 TABLE`；若之前已被标成 `TRX_DICT_OP_INDEX`，断言这只能是 **FTS 辅助表**或 `#sql-` 内部表在 drop——即"一个事务里既删索引又删表"的两种合法情形。

---


### TRUNCATE TABLE：rename + drop + create

⚠️ **勘误**：**不存在 `mysql_truncate_table()` / `truncate_t::operator()` / `Sql_cmd_truncate_table::handler_truncate()` / `recreate_table()`**。真实的是：**两个 file-static 自由函数** `handler_truncate_base()`（`sql_truncate.cc:168`）、`handler_truncate_temporary()`（:260）；recreate 路径**内联展开**在 `truncate_base()` 里。

**★ 两条路径的判据是 `HTON_CAN_RECREATE`，不是 `HTON_SUPPORTS_ATOMIC_DDL`**（`sql_truncate.cc:545`）：

```cpp
  if (hton->flags & HTON_CAN_RECREATE) {
    // 【注】★ 路径 1：recreate（InnoDB 走这条）
    binlog_is_trans = (hton->flags & HTON_SUPPORTS_ATOMIC_DDL);

    if (mysql_audit_table_access_notify(thd, table_ref) != 0) return;

    char path[FN_REFLEN + 1];
    build_table_filename(path, sizeof(path) - 1, table_ref->db,
                         table_ref->table_name, "", 0);

    // Attempt to reconstruct the table
    if (ha_create_table(thd, path, table_ref->db, table_ref->table_name,
                        &create_info, true, false, table_def) != 0) {
      return;                   // ★ 第 6 参 true = recreate
    }

    // Binlog only if truncate-by-recreate succeeds.
    m_error = false;
    binlog_stmt = true;
    return;
  }

  assert((hton->flags & HTON_CAN_RECREATE) == false);
  /* The engine does not support truncate-by-recreate.
     Attempt to use the handler truncate method. */
  const Truncate_result tr = handler_truncate_base(thd, table_ref, table_def);
  switch (tr) {
    case Truncate_result::OK:
      m_error = false;
      [[fallthrough]];
    case Truncate_result::FAILED_BUT_BINLOG:
      binlog_stmt = true;
      binlog_is_trans = table_ref->table->file->has_transactions();
      [[fallthrough]];
    case Truncate_result::FAILED_SKIP_BINLOG:
      close_all_tables_for_name(thd, table_ref->table->s, false, nullptr);
      break;
    case Truncate_result::FAILED_OPEN:
      break;
  }
```

```cpp
enum class Truncate_result {
  OK = 0,
  FAILED_BUT_BINLOG,
  FAILED_SKIP_BINLOG,
  FAILED_OPEN
};
```

**InnoDB 的 rename → drop → create 三步**（`ha_innodb.cc:14560`）：

```cpp
int innobase_truncate<Table>::truncate() {
  /* Rename tablespace file to avoid existing file in create. */
  if (m_file_per_table) {
    error = rename_tablespace();      // ★① t.ibd → #sql-ib<id>-<n>.ibd
  }
  if (error != 0) return error;
  ...
  dd_table_close(m_table, m_thd, nullptr, false);
  m_table = nullptr;

  error = innobase_basic_ddl::delete_impl(m_thd, m_name, m_dd_table, nullptr);
  //  ★② DROP 旧表（会写 DELETE_SPACE_LOG 指向改名后的临时文件）
  if (error != 0) return error;

  if (m_dd_table->is_persistent()) {
    m_dd_table->set_se_private_id(dd::INVALID_OBJECT_ID);
    for (auto dd_index : *m_dd_table->indexes()) {
      dd_index->se_private_data().clear();      // ★★ 清空所有索引的 se_private_data（含 root！）
    }
  }

  m_trx->in_truncate = true;
  error = innobase_basic_ddl::create_impl(
      m_thd, m_name, m_form, &m_create_info, m_dd_table, m_file_per_table,
      false, true, m_flags, m_flags2, ...);
  //  ★③ CREATE 新表（新 space，新 root page no）
  m_trx->in_truncate = false;
  ...
}
```

```cpp
// rename_tablespace() —— ha_innodb.cc:14655
  char *temp_name = dict_mem_create_temporary_tablename(
      m_table->heap, m_table->name.m_name, m_table->id);
  ...
  err = fil_rename_tablespace_check(m_table->space, old_path, new_path.c_str(), false);
  if (err == DB_SUCCESS) {
    dict_sys_mutex_enter();
    err = fil_rename_tablespace(m_table->space, old_path, temp_name, new_path.c_str());
    //  ★ 改名的同时会写 RENAME_SPACE_LOG + 写 redo MLOG_FILE_RENAME
    dict_sys_mutex_exit();
    if (err == DB_SUCCESS) {
      m_table->trunc_name.m_name = temp_name;
      //  ★ row_drop_table_for_mysql 用 trunc_name 拼出要删的文件路径
    }
  }
```

> ★★ **`trunc_name` 的作用**：`row_drop_table_for_mysql()` 里
> ```cpp
>   /* Table space file name has been renamed in TRUNCATE. */
>   table_name = table->trunc_name.m_name;
>   if (table_name == nullptr) { table_name = table->name.m_name; }
>   else { table->trunc_name.m_name = nullptr; }
> ```
> 所以 TRUNCATE 的 DELETE_SPACE_LOG 删的是**改名后的临时文件**（`#sql-ib...`），不是原 `t.ibd`（那个已经被 rename 成临时文件了，逻辑上等价）。

**TRUNCATE 的 MDL 是 X 名锁，不升级到表级**（`sql_truncate.cc:304`）：

```cpp
bool Sql_cmd_truncate_table::lock_table(THD *thd, Table_ref *table_ref) {
  /* Lock types are set in the parser. */
  assert(table_ref->lock_descriptor().type == TL_WRITE);
  /* The handler truncate protocol dictates a exclusive lock. */
  assert(table_ref->mdl_request.type == MDL_EXCLUSIVE);
  ...
  if (thd->locked_tables_mode) {
    ...
    m_ticket_downgrade = table->mdl_ticket;
    /* Close if table is going to be recreated. */
    if (table->s->db_type()->flags & HTON_CAN_RECREATE)
      close_all_tables_for_name(thd, table->s, false, nullptr);
    return false;
  }
  /* Acquire an exclusive lock. */
  if (lock_table_names(thd, table_ref, nullptr,
                       thd->variables.lock_wait_timeout, 0))
    return true;
  /* Table is already locked exclusively. Remove cached instances. */
  tdc_remove_table(thd, TDC_RT_REMOVE_ALL, table_ref->db, table_ref->table_name, false);
  return false;
}
```

> ★ TRUNCATE 可以修"打不开的坏表"——注释原文：*"As long as the table format file tbl_name.frm is valid, the table can be re-created as an empty table with TRUNCATE TABLE, even if the data or index files have become corrupted."*

**FK 父表禁止 TRUNCATE**（`sql_truncate.cc:123`）：

```cpp
static bool fk_truncate_illegal_if_parent(THD *thd, Table_ref *table_list,
                                          dd::Table *table_def) {
  for (const dd::Foreign_key_parent *fk_p : table_def->foreign_key_parents()) {
    if (my_strcasecmp(table_alias_charset, fk_p->child_schema_name().c_str(),
                      table_list->db) == 0 &&
        my_strcasecmp(table_alias_charset, fk_p->child_table_name().c_str(),
                      table_list->table_name) == 0)
      continue;                     // ★ 自引用 FK 允许
    my_error(ER_TRUNCATE_ILLEGAL_FK, MYF(0), fk_info_str(thd, fk_p));
    return true;
  }
  return false;
}
```

**`ha_truncate()` 的基类默认实现返回 `HA_ERR_WRONG_COMMAND`**（`sql/handler.h:6751`）：

```cpp
  virtual int truncate(dd::Table *table_def [[maybe_unused]]) {
    return HA_ERR_WRONG_COMMAND;
  }
```

> ⚠️ **InnoDB 没有覆盖 `truncate()`**——因为它有 `HTON_CAN_RECREATE`，server 从不走这条回退路径。


TRUNCATE 在 InnoDB 走 `HTON_CAN_RECREATE` 路径（ha_innodb.cc:5174），本质是 drop + recreate，从 `innobase_truncate::truncate()`（ha_innodb.cc:14558）看三步：

```
14564:14623:storage/innobase/handler/ha_innodb.cc
  if (m_file_per_table) {
    error = rename_tablespace();      // ① 旧 .ibd 重命名临时名（避免 create 文件冲突）
  }
  ...
  error = innobase_basic_ddl::delete_impl(...);   // ② 删旧表 → row_drop_table_for_mysql，写 DROP 类 DDL log
  ...
  error = innobase_basic_ddl::create_impl(...);   // ③ 建新空表，写 CREATE 类 DDL log
```

设计要点：

- **先 rename 而非直接删**：create 失败时旧文件（临时名）还在，有恢复余地。rename 本身通过 `RENAME_SPACE_LOG` 类型的 DDL log 记录
- TRUNCATE 的"undo"= DROP 阶段 DDL log + CREATE 阶段 DDL log + 两阶段 dd 元数据 undo，全部在同一个 DDL 事务内，提交后 post_ddl 统一 replay
- `m_trx->in_truncate = true`（:14612）走原子 truncate 日志；`m_dd_table->set_se_private_id(INVALID)`（:14596）使 table_id 重新分配
- TRUNCATE vs DROP+CREATE 区别：TRUNCATE 保留外键定义、可保留 autoinc（`m_keep_autoinc`）、复用同一 DDL 事务

---


### B-tree 的物理释放（btr_free）


DROP/TRUNCATE 时 B-tree 到底怎么删，取决于表空间类型。

#### 独立表空间（file-per-table）/ 一般表空间：整文件删除，不逐页释放

`row_drop_table_for_mysql` 只写 `DELETE_SPACE_LOG`（row0mysql.cc:4126），提交后：

```
Log_DDL::replay_delete_space_log (log0ddl.cc:1659)
  → fil_delete_tablespace (fil0fil.cc:4658)
    → Fil_shard::space_delete (:4493)
      → os_file_delete (:4641)    ← 整个 .ibd 删除，B-tree 所有页随文件消失
```

注意 `row0mysql.cc:4028` 的判定 `if (!table->is_temporary() && !file_per_table)`——**独立表空间根本不写 FREE_TREE_LOG，一个页都不会释放**。

#### 系统表空间 / 共享表空间：逐页释放 B-tree（FREE_TREE_LOG）

文件被其他表共享不能删，必须真正逐页释放。执行期只写 log：

```
4028:4055:storage/innobase/row/row0mysql.cc
  if (!table->is_temporary() && !file_per_table) {
    err = log_ddl->write_free_tree_log(trx, index, true);  // 每个 index 一条 FREE_TREE_LOG
  }
  ...
  index->page = FIL_NULL;  // dict cache 标记索引不可用
```

`write_free_tree_log`（log0ddl.cc:853）→ `insert_free_tree_log`（920）在 `innodb_ddl_log` 表 insert（space_id / root_page_no / index_id）。提交后 replay：

```
1583:1650:storage/innobase/log/log0ddl.cc
  switch (record.get_type()) {
    case Log_Type::FREE_TREE_LOG:
      replay_free_tree_log(record.get_space_id(), record.get_page_no(), record.get_index_id());
      break;
  ...
void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no, ulint index_id) {
  btr_free_if_exists(page_id_t(space_id, page_no), page_size, index_id, &mtr);
```

`btr_free_if_exists`（btr0btr.cc:1010）分四步：

1. `btr_free_root_check`（809）：读根页，校验是 index 页且 `PAGE_INDEX_ID` == index_id，防止崩溃恢复时重复/错误释放
2. `btr_free_but_not_root`（958）：释放除根页外的整棵树
3. `btr_free_root`（771）：释放根页
4. `btr_free_root_invalidate`（795）：根页 `PAGE_INDEX_ID` 写 0（`BTR_FREED_INDEX_ID`），幂等标记

`btr_free_but_not_root`（958-1003）：B-tree 的 LEAF segment 与 TOP segment 是两个独立文件段，分两轮释放：

- leaf_loop：循环 `fseg_free_step(root+PAGE_BTR_SEG_LEAF, ...)`，每个 mtr 释放叶子段的一个 extent
- top_loop：循环 `fseg_free_step_not_header(root+PAGE_BTR_SEG_TOP, ...)` 释放非叶段（不含头页）

> **完整源码剖析见 [`../../index/btr.md`](../../index/btr.md)「树的创建、分配、释放与截断」章**——`btr_free_if_exists` 四步、`btr_free_but_not_root` 的 leaf/top 双循环、`btr_free_root`、`btr_free_root_invalidate`、`btr_free_root_check` 在那里都贴了源码并逐行解释（连带 `btr_create`、`btr_truncate`）。本篇只记与 DDL 相关的三点：

1. **一个 mtr 只释放一个 extent**：`fseg_free_step` 每放一小步就 `mtr_commit`，大索引的释放是**多次 mtr 提交**——否则 mtr 的 memo 栈会无限膨胀、redo 也写不下。这正是 DROP 大表时释放过程"慢慢进行"的原因。
2. **删前校验 + 删后作废构成幂等**：`btr_free_root_check` 校验 `PAGE_INDEX_ID == index_id`（防止页已被新索引复用而误删），释放后 `btr_free_root_invalidate` 把它置 0。**崩溃恢复重放时校验失败即跳过，不需要额外的"已处理"记录表**——幂等性直接编码在页面内容里（见 [`atomic_ddl.md`](atomic_ddl.md)「幂等性」节）。
3. **临时表走 `btr_free`（`MTR_LOG_NO_REDO`）且省掉 check 与 invalidate 两步**：临时表不参与崩溃恢复，也就不需要"重复回放安全"，这两步纯属多余。

`fseg_free_step`（fsp0fsp.cc:3658）：一个 mini-transaction 内释放 segment 的一个分配单元（1 个完整 extent 或 1 个 frag 页），返回 false 表示未释放完，外层循环继续。核心逻辑：

- 有整 extent：`fseg_free_extent`（3591）把 extent 从 FSEG_FULL / FSEG_NOT_FULL / FSEG_FREE 链摘除 → `fsp_free_extent`（1896）`xdes_init` 复位描述符 → `flst_add_last(FSP_FREE)` 归还表空间空闲区（`space->free_len++`）；frag extent 归还 FSP_FREE_FRAG
- 只有 frag 页：`fseg_free_page_low`（3725）逐页释放
- 全部释放完：`fsp_free_seg_inode`（3720）归还 segment inode 页
- AHI：`fseg_free_extent` 内按页 `btr_search_drop_page_hash_when_freed`（3622）摘自适应哈希索引

`btr_free_root`（771-786）：先 `btr_search_drop_page_hash_index` 摘 AHI，再循环 `fseg_free_step` 释放根页所在段。

**设计思想**：free B-tree 是破坏性物理操作，页被释放后若 DDL 回滚无法恢复。用 DDL log 记录意图、提交后执行，配合元数据 undo 可回滚。崩溃恢复重放 FREE_TREE_LOG，靠 `btr_free_root_invalidate` 的 index_id=0 标记保证幂等。释放粒度是 extent 而非逐页，整 extent 一次性归还 FSP_FREE，XDES 描述符直接复位，开销小。

补充：临时表清空走 `row_delete_all_rows`（row0mysql.cc:2491）——`btr_free`（btr0btr.cc:1026，MTR_LOG_NO_REDO）+ `btr_create` 重建根页，不经过 DDL log。

---

### `mysql_rm_table_no_locks()`：差别不是"谁持锁"，而是"是否自己取锁"

**一句话**：`mysql_rm_table()` = **加锁** + 特殊场景 binlog；`mysql_rm_table_no_locks()` = **排序分组 + SE 删除 + DD 删除 + binlog + 提交/回滚**。

| 维度 | `mysql_rm_table()`（:1550） | `mysql_rm_table_no_locks()`（:3119） |
|---|---|---|
| 谁取锁 | **它取**：`lock_table_names()`、`lock_trigger_names()`、`acquire_shared_backup_lock()`、`rm_table_do_discovery_and_lock_fk_tables()`、`lock_check_constraint_names()` | **不取**。注释原文 *"@note This function assumes that metadata locks have already been taken."* |
| 调用者 | 用户 `DROP TABLE`（`Sql_cmd_drop_table`） | `mysql_rm_table()`、`DROP DATABASE`（`sql_db.cc:809`） |
| binlog | 仅函数内特例（存储函数里的 `DROP TEMPORARY`） | **主体 binlog 全在这里**（`Drop_tables_query_builder`） |
| 输出参数 | 无 | `dropped_non_atomic_flag` / `post_ddl_htons` / `fk_invalidator` / `safe_to_release_mdl` |

**★ `Drop_tables_ctx`（:1729）——分组不是按 engine，而是三个正交维度**：

```cpp
class Drop_tables_ctx {
  /* Parameters of DROP TABLES statement. */
  const bool if_exists;
  const bool drop_temporary;
  const bool drop_database;

  /* Different table groups of tables to be dropped. */
  Prealloced_array<Table_ref *, 1> base_atomic_tables;      // 1.b 支持原子 DDL 的基表
  Prealloced_array<Table_ref *, 1> base_non_atomic_tables;  // 1.a 不支持原子 DDL 的基表
  Prealloced_array<Table_ref *, 1> tmp_trans_tables;        // 2.b 事务型临时表
  Prealloced_array<Table_ref *, 1> tmp_trans_tables_to_binlog;
  Prealloced_array<Table_ref *, 1> tmp_non_trans_tables;    // 2.a 非事务型临时表
  Prealloced_array<Table_ref *, 1> tmp_non_trans_tables_to_binlog;
  Prealloced_array<Table_ref *, 1> nonexistent_tables;      // 1.d / 2.c
  Prealloced_array<Table_ref *, 1> views;                   // 1.c（只有 DROP DATABASE 才填）
  Prealloced_array<Table_ref *, 1> dropped_non_atomic;

  enum {
    NO_GTID_MANY_TABLE_GROUPS,
    NO_GTID_SINGLE_TABLE_GROUP,
    GTID_MANY_TABLE_GROUPS,
    GTID_SINGLE_TABLE_GROUP
  } gtid_and_table_groups_state;
};
```

**分组判据**（`rm_table_sort_into_groups()`，:1964）：

```cpp
      assert(!(hton->flags & HTON_SUPPORTS_FOREIGN_KEYS) ||
             (hton->flags & HTON_SUPPORTS_ATOMIC_DDL));
      //  ★★ 所有支持 FK 的引擎都被假定为支持原子 DDL —— 这是 drop/rename 里大量 assert(atomic) 的根

      if (hton->flags & HTON_SUPPORTS_ATOMIC_DDL || thd->is_plugin_fake_ddl())
        drop_ctx->base_atomic_tables.push_back(table);      // ★ 原子组
      else
        drop_ctx->base_non_atomic_tables.push_back(table);  // ★ 非原子组
```

> ★★ **唯一决定"逐表删 + 逐表提交 + 逐表 binlog" 还是 "整组一个事务" 的判据是 `HTON_SUPPORTS_ATOMIC_DDL`**。

**binlog 有 6 个写入点**（全部在 `mysql_rm_table_no_locks()` 里）：

| 点 | 场景 | 方式 |
|---|---|---|
| A | DROP DATABASE 中的非原子表 | `mysql_bin_log.write_event(&qinfo)` 直写 |
| B | DROP TABLE 中的非原子表 | 单表 `DROP TABLE ... /* generated by server */` + 立即 `trans_commit_stmt` |
| C | **原子组**（含不存在表） | **一条多表 DROP TABLE**，落入 **trx cache** |
| D | GTID_MANY 组 | 最后一次性补写所有表 |
| E | 非事务临时表 | `DROP TEMPORARY TABLE`，走 **stmt cache**（事务之前） |
| F | 事务型临时表 | `DROP TEMPORARY TABLE` |

**★ 8.0 原子性的落点**（:3423）：

```cpp
      Drop_tables_query_builder built_query(
          thd, false /* no TEMPORARY */, drop_ctx.if_exists,
          /* stmt or trx cache. */
          drop_ctx.has_base_atomic_tables(), false /* db exists */);

      built_query.add_array(drop_ctx.base_atomic_tables);
      built_query.add_array(drop_ctx.nonexistent_tables);
      thd->thread_specific_used = true;

      if (built_query.write_bin_log()) goto err_with_rollback;
      //  ★ binlog 写入点 C：一条多表 DROP TABLE，落入 trx cache

      if (drop_ctx.has_no_gtid_single_table_group() ||
          drop_ctx.has_gtid_single_table_group()) {
        /*
          This is fully atomic multi-table DROP TABLES.
          Commit changes to SEs, data-dictionary and binary log/or
          and mark GTID as executed/update slave info tables atomically.
        */
        error = (trans_commit_stmt(thd) || trans_commit_implicit(thd));
        //  ★★★ SE(含 InnoDB DDL log) + DD + binlog 一次提交
      }
```

> ⚠️ **纠正一个流传的错误**：**server 层没有 `ddl_log_state` 句柄可传**（全仓 grep `ddl_log_state` = 0 命中）。8.0 的 DDL 日志已下沉到 InnoDB（`log0ddl.cc`）。原子性靠"**DD 变更 + InnoDB DDL log + binlog 在同一事务里一次提交**"，不是两套提交。

**`drop_base_table()` 的真实顺序**（:2800）——**SE 先删，DD 后删**：

```cpp
  int error = ha_delete_table(thd, hton, path, table->db, table->table_name,
                              table_def, !drop_ctx.drop_database);
  //  ★ 步骤 1：先在 SE 里删表（→ handler::delete_table）。注意：这里没有任何 undo 相关调用

  if (atomic && hton->post_ddl) post_ddl_htons->insert(hton);
  //  ★ post_ddl 钩子（InnoDB 用来在提交/回滚后清理 ddl log）

  if (adjust_fk_children_for_parent_drop(thd, table->db, table->table_name,
                                         table_def, hton) ||
      adjust_fk_parents(thd, table->db, table->table_name, false, nullptr))
    return true;                                 // 步骤 2：清子表 FK 的 UNIQUE_CONSTRAINT_NAME

  bool result = dd::drop_table(thd, table->db, table->table_name, *table_def);
  //  ★ 步骤 3：从 DD 删除（SE 已删完！）

  if (!atomic) result = trans_intermediate_ddl_commit(thd, result);
  //  ★ 步骤 4：非原子引擎 → 立刻单独提交 DD 变更（无 undo 可回滚，只能"尽快落盘"降低不一致窗口）

  result |= mark_referencing_views_invalid(thd, table, ...);
```

| | `mysql_rm_table()`（外壳） | `mysql_rm_table_no_locks()`（内核） |
|---|---|---|
| MDL | **自己取**：`lock_table_names()` + `lock_trigger_names()` + `rm_table_do_discovery_and_lock_fk_tables()` + `lock_check_constraint_names()`；LOCK TABLES 模式下还要 `find_table_for_mdl_upgrade()` + `wait_while_table_is_used(..., HA_EXTRA_FORCE_REOPEN)` | **不取**，文档注释：*\"@note This function assumes that metadata locks have already been taken. It is also assumed that the tables have been removed from TDC.\"* |
| 前置检查 | XA_IDLE / XA_PREPARED 拒绝；`decide_logging_format()`；拒绝删日志表 | 无 |
| 收尾 | 释放 MDL、处理 LOCK TABLES 退出（`reopen_tables` / `unlock_locked_tables` / 逐个 `safe_to_release_mdl`） | 无 |

`mysql_rm_table` 做完上述准备后才调 `mysql_rm_table_no_locks(...)`。

⚠️ **`mysql_rm_table_no_locks()` 的调用者不止 `mysql_rm_table`**——**DROP DATABASE 路径也直接调它**（传 `drop_database=true`）。

它比 `mysql_rm_table` 多 4 个参数：`drop_database`（DROP DATABASE 语义，视图/binlog 特殊处理）、`dropped_non_atomic_flag`（是否删了不支持原子 DDL 的引擎的表）、`post_ddl_htons`（需要 post-DDL 回调的 handlerton 集合）、`fk_invalidator`（父表 DD 缓存失效集合）。

---


---


---

> ★ **以下主题已独立成篇**（本篇保留跨层公共骨架、算法选择与 MDL 三段式）：
> - **★★ 原子 DDL**（server 编排七步 + 引擎 DDL log + 崩溃恢复闭环，所有 DDL 类型共用的横切保障）→ [`atomic_ddl.md`](atomic_ddl.md)
> - **8.0 新版 DDL 引擎与并行构建**：`storage/innobase/ddl/` 全景、Loader/Builder/Btree_load、任务队列的同步与终止检测、`Parallel_reader` 动态分片 → [`ddl_engine.md`](ddl_engine.md)
> - **INPLACE 的 online 支柱 row log**（为什么 apply 两次、短暂阻塞点从哪来）→ [`inplace_ddl.md`](inplace_ddl.md)
> - **INSTANT 的行版本号机制**（表级/行级/物理三层级辨析、WL#11250）→ [`instant_ddl.md`](instant_ddl.md)

## DDL 行为速查：四维评估与实战陷阱


> 参考月报《云原生数据库 PolarDB MySQL 8.0.2 DDL 介绍》（2023/09）的组织思路，但**所有结论均回社区版 8.0.39 源码核实**，并标注哪些是 PolarDB 私有增强。

### DDL 分类的四个维度

月报给的框架很实用：判断一个 DDL 的影响，光知道"COPY/INPLACE/INSTANT"不够，用户真正关心的是**四个正交维度**：

| 维度 | 关心的问题 | 对应本文概念 |
|------|-----------|-------------|
| **是否锁表**（允许并发 DML） | 会不会影响业务写入 | Online 与否（`check_if_supported_inplace_alter` 返回值决定 MDL 级别） |
| **是否重建表** | 要多久（是否随表规模增长） | `need_rebuild` / `new_clustered` |
| **是否只改元数据** | 能否秒级完成 | INSTANT（`innobase_support_instant`） |
| **是否支持并行** | 能否多线程加速 | `innodb_ddl_threads` 等（见下） |

**关键认知**：Online DDL 只在"改元数据"这一步申请表互斥锁（一般 <1s），期间允许读写；非 Online 的 DDL 全程锁表。

### 具体操作 × 四维对照（源码依据）

InnoDB 用三组 `HA_ALTER_FLAGS` 常量（`handler0alter.cc:114-172`）决定每个操作走哪条路：

```cpp
/** Operations for rebuilding a table in place */
INNOBASE_ALTER_REBUILD =
    ADD_PK_INDEX | DROP_PK_INDEX | CHANGE_CREATE_OPTION
    /* CHANGE_CREATE_OPTION needs to check innobase_need_rebuild() */
  | ALTER_COLUMN_NULLABLE | ALTER_COLUMN_NOT_NULLABLE
  | ALTER_STORED_COLUMN_ORDER | DROP_STORED_COLUMN
  | ADD_STORED_BASE_COLUMN
    /* ADD_STORED_BASE_COLUMN needs to check innobase_need_rebuild() */
  | RECREATE_TABLE;

/** Operation allowed with ALGORITHM=INSTANT */
INNOBASE_INSTANT_ALLOWED =
    ALTER_COLUMN_NAME | ADD_VIRTUAL_COLUMN | DROP_VIRTUAL_COLUMN
  | ALTER_VIRTUAL_COLUMN_ORDER | ADD_STORED_BASE_COLUMN
  | ALTER_STORED_COLUMN_ORDER | DROP_STORED_COLUMN;

/** ...can perform without rebuild */
INNOBASE_ALTER_NOREBUILD =
    ADD_INDEX | ADD_UNIQUE_INDEX | ADD_SPATIAL_INDEX
  | DROP_INDEX | DROP_UNIQUE_INDEX | RENAME_INDEX
  | ALTER_COLUMN_NAME | ALTER_COLUMN_EQUAL_PACK_LENGTH
  | ADD/DROP_VIRTUAL_COLUMN | ALTER_COLUMN_INDEX_LENGTH | ...;
```

由此得到四维对照表：

| 操作 | 重建表 | 秒级 | 锁表 | 并行 | 说明 |
|------|--------|------|------|------|------|
| **ADD COLUMN** | 否/是¹ | ✅/❌ | 否 | 否 | `ADD_STORED_BASE_COLUMN` |
| **DROP COLUMN** | 否/是¹ | ✅/❌ | 否 | 否 | `DROP_STORED_COLUMN` |
| **RENAME COLUMN** | 否 | ✅ | 否 | 否 | `ALTER_COLUMN_NAME`，纯元数据 |
| ADD/DROP VIRTUAL COLUMN | 否 | ✅ | 否 | 否 | 虚拟列不存数据 |
| **ADD INDEX**（二级） | 否 | ❌ | **否**（LOCK=NONE） | **✅** | `INNOBASE_ONLINE_CREATE`，见「并行 DDL」 |
| **DROP INDEX** | 否 | ✅ | 否 | 否 | `DROP_INDEX`，只改元数据 |
| RENAME INDEX | 否 | ✅ | 否 | 否 | `RENAME_INDEX` |
| **ADD / DROP PRIMARY KEY** | **是** | ❌ | **是** | ✅ | `ADD_PK_INDEX`/`DROP_PK_INDEX`，代价最高 |
| **改列类型**（MODIFY/CHANGE） | **是** | ❌ | 视情况 | ✅ | `ALTER_STORED_COLUMN_TYPE` |
| **改列顺序** | 是/否¹ | ✅/❌ | 否 | — | `ALTER_STORED_COLUMN_ORDER` |
| 改 NULL/NOT NULL | **是** | ❌ | 视情况 | ✅ | `ALTER_COLUMN_NULLABLE`/`NOT_NULLABLE` |
| **改列默认值 / 列可见性** | 否 | ✅ | 否 | 否 | 属 `INNOBASE_INPLACE_IGNORE`，InnoDB 不关心 |
| **OPTIMIZE / ALTER ENGINE** | **是** | ❌ | 否 | **✅** | `RECREATE_TABLE` |
| 改表选项（ROW_FORMAT 等） | 是/否¹ | — | 视情况 | ✅ | `CHANGE_CREATE_OPTION` |

**¹ 重要发现——"加/删列有时秒回、有时跑很久"的源码解释**：

`ADD_STORED_BASE_COLUMN`、`DROP_STORED_COLUMN`、`ALTER_STORED_COLUMN_ORDER` **同时出现在** `INNOBASE_ALTER_REBUILD` 和 `INNOBASE_INSTANT_ALLOWED` 两个集合里。源码注释直接点明：

```cpp
Alter_inplace_info::ADD_STORED_BASE_COLUMN
/* ADD_STORED_BASE_COLUMN needs to check innobase_need_rebuild() */
```

即**加/删列既可能走 INSTANT，也可能退化为 REBUILD**，由 `innobase_need_rebuild()` 判定。退化条件就是「Instant DDL 深度剖析」里那几条限制（压缩表、行大小超限、非末尾加列、有全文索引等）。**踩中任一条 → 从秒级退化为全表重建**，这正是生产上 ADD COLUMN 表现不一致的根因。


月报重点提示的坑，社区版源码依据在 `sql/field.h:4291-4294`（`Field_json` 的同类实现，VARCHAR 语义一致）：

```cpp
uint32 get_length_bytes() const override {
  assert(m_elt_type == MYSQL_TYPE_VARCHAR);
  return field_length > 255 ? 2 : 1;      // ★ 长度字节数：≤255 用 1 字节，>255 用 2 字节
}
```

`Field_varstring` 用成员 `length_bytes`（field.h:3584，注释 "Store number of bytes used to store length (1 or 2)"）记录该值。

**后果**：把 VARCHAR 从 **≤255 字节扩到 ≥256 字节**会改变 `length_bytes`（1→2），行格式变化 → **无法只改元数据，退化为 COPY（全程锁表）**。

**建议**（月报给的）：varchar 是变长存储、磁盘只存实际长度，建表时**直接把最大长度设到 256 以上**，避免后续扩展踩坑。不确定时可显式 `ALGORITHM=INPLACE` 试，不支持会直接报错而非偷偷降级：

```
ALTER TABLE t ALGORITHM=INPLACE, CHANGE COLUMN c1 c1 VARCHAR(256);
ERROR 0A000: ALGORITHM=INPLACE is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

### 实战陷阱二：Instant ADD/DROP COLUMN 的表级限制

社区版的判断在 `dict_table_t::support_instant_add_drop()`（`dict0dict.ic:1371-1376`）：

```cpp
inline bool dict_table_t::support_instant_add_drop() const {
  return (
      !DICT_TF_GET_ZIP_SSIZE(flags) &&                         // 非压缩表（ROW_FORMAT≠COMPRESSED）
      space != dict_sys_t::s_dict_space_id &&                   // 非 DD 表空间
      !DICT_TF2_FLAG_IS_SET(this, DICT_TF2_FTS_HAS_DOC_ID) &&   // 无 FTS_DOC_ID
      !is_temporary() &&                                        // 非临时表
      !DICT_TF2_FLAG_IS_SET(this, DICT_TF2_FTS) &&              // 无全文索引
      !is_system_table);                                        // 非系统表
}
```

**即：压缩表、全文索引表、临时表、系统表 → 不支持 Instant**。这点与月报一致（月报还提到"需已有主键""只能加到末尾"——Instant 的语义是新列不落已有行、用默认值，故只能加在末尾；若表无主键，末尾的隐式主键会导致失败）。

**不满足时**：加列退化为 **INPLACE 全表重建**（允许并发 DML、可用并行 DDL 加速）。

### 实战陷阱三：INSTANT 只支持白名单内的操作

社区版 `innobase_support_instant`（handler0alter.cc:821-907）明确了 INSTANT 的操作范围。`INNOBASE_INSTANT_ALLOWED`（148 行起）白名单之外的操作一律 `INSTANT_IMPOSSIBLE`：

```cpp
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME |
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |
    Alter_inplace_info::DROP_VIRTUAL_COLUMN | ...;
```

`INSTANT_OPERATION` 分类（840-849）与几个"不支持"的边界（871-904）：
- `COLUMN_RENAME_ONLY`：仅列重命名 → 需 `ok_to_rename_column`；**若列被外键引用则不允许 INSTANT**（777-801）
- `VIRTUAL_ADD_DROP_ONLY`：仅虚拟列增删
- `VIRTUAL_ADD_DROP_WITH_RENAME`：**"Not supported yet in INPLACE. So not supporting here as well."**（886）→ 不支持
- `INSTANT_ADD` / `INSTANT_DROP`：加/删存储列 → 需 `table->support_instant_add_drop()`

另外 `innobase_need_rebuild`（922）：`if (is_instant(ha_alter_info)) return false;` —— INSTANT 不重建表。

### 实战陷阱四：任何"无锁变更"工具在切表时都要拿 MDL-X

月报强调的一点，值得记住：**gh-ost / pt-osc / DMS 无锁变更，在最后"切表（修改元数据）"时同样至少拿 MDL-X**，Online DDL 也一样。所以：
- 执行 DDL 仍需确保不被大事务/大查询堵塞（否则 MDL-X 等待会连带阻塞后续所有请求，甚至连接打满）
- "无锁"是相对的（仅指数据迁移阶段）

### 社区版 vs PolarDB 差异一览

| 能力 | 社区版 8.0.39 | PolarDB |
|------|--------------|---------|
| 并行 DDL | ✅ 有（`innodb_ddl_threads` 等，8.0.27+） | ✅ 有，调优到 15-20 倍 |
| Instant ADD/DROP COLUMN | ✅ 有（8.0.12+ ADD、8.0.29+ DROP） | ✅ 有 |
| 分区级 MDL | ❌ **社区版无** | ✅ `loose_partition_level_mdl_enabled`，DDL 不影响不涉及分区的 DML |
| 非阻塞 DDL | ❌ 社区版无 | ✅ PolarDB 特性（见月报 2022/10） |

---

---

## DDL 与 binlog / 复制


### 统一入口

```cpp
int write_bin_log(THD *thd, bool clear_error, const char *query,
                  size_t query_length, bool is_trans) {
  int error = 0;
  if (mysql_bin_log.is_open()) {
    int errcode = 0;
    if (clear_error) thd->clear_error();
    else errcode = query_error_code(thd, true);
    error = thd->binlog_query(THD::STMT_QUERY_TYPE, query, query_length,
                              is_trans, false, false, errcode);
  }
  return error;
}
```

⇒ **`is_trans` 的语义 = 用 trx cache 还是 stmt cache**。这个值不是猜的，而是由引擎能力直接算出（建表准备路径）：

```cpp
/*
  If the SE supports atomic DDL, we can use the trx binlog cache.
  Otherwise we must use the statement cache.
*/
if (binlog_to_trx_cache != nullptr)
  *binlog_to_trx_cache =
      (create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
```

★ 同一个 `HTON_SUPPORTS_ATOMIC_DDL` 标志，在 DDL 里一共决定三件事：① DD 变更能否与引擎操作同处一个事务；② 是否启用 DDL log 机制（见 [`atomic_ddl.md`](atomic_ddl.md)）；③ **binlog 写 trx cache 还是 stmt cache**。

- 写 **trx cache** ⇒ binlog 记录随 DDL 事务一起提交，**崩溃时与 DD/引擎状态同生同灭**（这是原子 DDL 能把 binlog 也纳入原子的原因）；
- 写 **stmt cache** ⇒ 语句级提交，DDL 中途失败时 binlog 可能已经落盘，三处状态就对不上了——这正是 5.7 时代"DDL 中断留下不一致"的成因之一。

### DDL 与行格式的冲突

1. **DDL 只有 `Query_log_event`（语句文本）**——`THD::STMT_QUERY_TYPE`。行格式描述"行的前后像"，无法描述"表结构本身变了"
2. **CTAS 的行顺序不确定**：

```cpp
/*
  CREATE TABLE...IGNORE/REPLACE SELECT... can be unsafe, unless
  ORDER BY PRIMARY KEY clause is used in SELECT statement.
  TODO: Check if the order of the output of the select statement is
  deterministic. Waiting for BUG#42415
*/
```

3. **CTAS + AUTO_INCREMENT 不确定**（`mysql_prepare_create_table()`）：

```cpp
/*
 CREATE TABLE[with auto_increment column] SELECT is unsafe as the rows
 inserted in the created table depends on the order of the rows fetched
 from the select tables. This order may differ on master and slave.
*/
```

4. **非确定性函数做默认值**：STMT 下只告警、ROW/MIXED 下直接 `ER_BINLOG_UNSAFE_SYSTEM_FUNCTION`
5. **`TEMPORARY` 与 ROW 冲突**：临时表在备库不存在 → RBR 下干脆不记；但 LIKE 的 Case 2 必须改写语句带上 `ENGINE=`
6. **GIPK 语句重写**：因为 `@@sql_generate_invisible_primary_key` 的值不进 binlog
7. **`DROP TABLE` 也要重写**：`Drop_tables_query_builder` 生成合成的 DROP 语句

### 关于"DDL 强制 statement"

⚠️ **并不是由某个 `force_stmt` 标志实现**——DDL 路径上根本不调用 `binlog_query(THD::ROW_QUERY_TYPE, ...)`，一律走 `STMT_QUERY_TYPE` 写 `Query_log_event`。

MIXED 下 `decide_logging_format()` 只影响**临时表相关**的 DDL 是否切到 ROW（进而决定是否写 binlog），不会把 DDL 本身变成行事件。这就是 CREATE/DROP/TRUNCATE 必须在**拿锁之前**先调一次 `decide_logging_format()` 的原因：

```cpp
/*
  DROP tables need to have their logging format determined if
  in MIXED mode and dropping a TEMP table.
*/
if (thd->decide_logging_format(tables)) return true;
```

> （`LEX::binlog_row_based_if_mixed` 在 `sql_table.cc` 内无直接引用，只在 `sql_class.h` / `item_create.h` 的注释中出现。）

---


---

## 关键数据结构全览


| 结构 | 文件 | 职责 | 关键成员 |
|---|---|---|---|
| **`Alter_info`** | `sql_alter.h` | **CREATE/ALTER 的"意图描述"**（parser 产物） | `create_list`、`key_list`、`drop_list`、`alter_list`、`alter_rename_key_list`、`alter_index_visibility_list`、`flags`(41 位)、`requested_algorithm`、`requested_lock`、`with_validation`、`new_db_name`/`new_table_name` |
| **`Alter_table_ctx`** | `sql_alter.h` | **执行期上下文** | 原名/新名、`tmp_name`(`#sql-pid_tid`)、`path`/`tmp_path`、`fk_info` 快照、`target_mdl_request` |
| **`Create_field`** | `create_field.h` | 一列的"待创建/待修改"描述 | `field_name`、`change`、`after`、`sql_type`、`flags`、`charset`、`gcol_info`、`hidden`、`field`（ALTER 时指向旧 Field） |
| **`Key_spec`** | `key_spec.h` | **parser 产物的索引定义** | `type`(PRIMARY/UNIQUE/MULTIPLE/FULLTEXT/SPATIAL/FOREIGN)、`columns`、`name`、`generated` |
| **`Key_part_spec`** | `key_spec.h` | 一个索引列（含函数索引） | `m_field_name`、`m_prefix_length`、`m_expression` |
| **`Foreign_key_spec`** | `key_spec.h` | FK 定义 | `ref_db`/`ref_table`/`ref_columns`、`delete_opt`/`update_opt`/`match_opt` |
| **`Alter_drop`** | `sql_alter.h` | DROP 子句 | `name` + `drop_type {KEY, COLUMN, FOREIGN_KEY, CHECK_CONSTRAINT, ANY_CONSTRAINT}` |
| **`Alter_column`** | `sql_alter.h` | SET/DROP DEFAULT、RENAME COLUMN、可见性 | `name`、`def`、`m_new_name`、`Type` |
| **`HA_CREATE_INFO`** | `handler.h` | 表级选项集合 | `table_charset`、`comment`、`compress`、`encrypt_type`、`data_file_name`、`max_rows`/`auto_increment_value`、`table_options`、`used_fields`(位图)、`db_type`、`row_type`、`m_hidden`、`m_transactional_ddl` |
| ★ **`Alter_inplace_info`** | `handler.h` | **给引擎看的"这次到底要改什么"** | `handler_flags`（ADD_INDEX/DROP_INDEX/ADD_PK_INDEX/ADD_VIRTUAL_COLUMN… 位图）、`create_info`、`alter_info`、`index_drop_buffer`/`index_add_buffer`、`online`、`error_if_not_empty` |
| **`inplace_alter_handler_ctx`** | `handler.h` | 引擎在 inplace 各阶段间的私有上下文 | 纯虚基类 |
| **`Table_exists_result`** | `sql_table.cc`(static) | 命名冲突检查 | `m_table_exists`、`m_error` |
| **`Drop_tables_ctx`** | `sql_table.cc`(static) | 多表 DROP 分组状态机 | 见 3.1 |
| **`Drop_tables_query_builder`** | `sql_table.cc`(static) | 合成 DROP 的 binlog 语句 | `m_built_query` |

> ★ **`Alter_info` 与 `Alter_inplace_info` 是双层设计**：前者是"用户说了什么"，后者是"server 算出来要做什么"（`handler_flags` 位图）。**引擎只看后者。**

---


---

## 核心调用栈与相关系统变量（引擎侧）

### 核心调用栈

```
TRUNCATE TABLE（server 层）
  Sql_cmd_truncate_table::execute (sql_truncate.cc:745)
  → ha_innobase::truncate_impl (ha_innodb.cc:15261)
  → innobase_truncate<dd::Table>::exec (ha_innodb.cc:14748)
  → innobase_truncate<Table>::truncate (ha_innodb.cc:14558)
    → rename_tablespace (14653)：fil_rename_tablespace → RENAME_SPACE_LOG
    → innobase_basic_ddl::delete_impl (14226)
      → row_drop_table_for_mysql (row0mysql.cc:3771)
        → write_free_tree_log (4032, 仅共享表空间) → insert_free_tree_log (log0ddl.cc:920)
        → write_drop_log (4115)
        → write_delete_space_log (4126, 仅独立表空间)
    → innobase_basic_ddl::create_impl (14109)：dict_create_index_tree_in_mem → btr_create (btr0btr.cc:832)

提交后 post_ddl：
  Log_DDL::post_ddl (log0ddl.cc:1905) → replay_by_thread_id (1930)
  → replay (1576) 按类型分发
    → FREE_TREE_LOG: replay_free_tree_log (1625)
        → btr_free_if_exists (btr0btr.cc:1010)
          → btr_free_root_check (809) → btr_free_but_not_root (958) → btr_free_root (771) → btr_free_root_invalidate (795)
            → fseg_free_step (fsp0fsp.cc:3658)
              → fseg_free_extent (3591) → fsp_free_extent (1896) → FSP_FREE 列表
    → DELETE_SPACE_LOG: replay_delete_space_log (1659)
        → fil_delete_tablespace (fil0fil.cc:4658) → Fil_shard::space_delete (4493) → os_file_delete (4641)

崩溃恢复：
  Log_DDL::recover (log0ddl.cc:1942) → replay_all (1952) → replay (1576) → 同上
```

---

### 相关的系统变量/状态变量

#### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_print_ddl_logs` | OFF | Global | 把 DDL log 的 insert/delete/replay 全流程打印到错误日志（`srv_print_ddl_logs`，ha_innodb.cc:23090），调试原子 DDL 崩溃问题 |

#### 状态变量

无专门状态变量。

---


---

## DDL 如何观测（要点）

> 通用观测**手段**（PFS 表结构、`information_schema`、`SHOW PROCESSLIST`、错误日志）见 [`../../infra/observability/README.md`](../../infra/observability/README.md) 及该目录下各篇；本篇只记 **DDL 特有的观测点与坑**。

### 三个"看不到"及其真正原因

| 现象 | 真正原因（源码依据） |
|---|---|
| **看不到 ALTER 进度** | 不是 instrument 没开——`stage/innodb/alter table%` 带 `PSI_FLAG_STAGE_PROGRESS`，`register_stage_class()` 里 progress 类 stage 默认就是 ENABLED/TIMED。真正原因是 **stage consumer 默认 OFF**（`performance_schema_consumer_events_stages_current` 默认 false，且 `READ_ONLY NOT_VISIBLE`，只能启动时指定） |
| **PFS 查不到 `altering table`，但 PROCESSLIST 有** | `stage/sql/altering table` 的 flags=0 ⇒ 默认 `ENABLED=NO`；但 `pfs_start_stage_v1()` 里 `/* Always update column threads.processlist_state. */` ⇒ **State 的更新不依赖 instrument 开关** |
| **并行 DDL 时用户连接那行进度不动** | 各 builder 的 `Alter_stage` 是在 `thread/innodb/ib_ddl` **工作线程**里 `begin_phase_*` 的，用户线程只在 `Loader::~Loader()` → `Alter_stage::aggregate()` 后才拿到汇总值 |

⇒ **看进度的第一步永远是打开 stage consumer**（`events_stages_current` / `_history` / `_history_long`），启动时指定 `--performance-schema-consumer-events-stages-current=ON`。

### 进度估算的偏差来源

源码注释自己承认是估算：`WORK_ESTIMATED` 由行数等推得，注释原文用了 "wild guess" 与 "factor of 6" 这类措辞。所以进度条只能当参考，不能用来推算剩余时间。

### DDL 卡住：用 `metadata_locks` 诊断

- `LOCK_STATUS='PENDING'` = 该 MDL 请求正在排队；这是判断"DDL 被挡住"的直接依据。
- ⚠️ MySQL **没有在 SQL 层暴露 MDL wait-for graph 的表**——把同一对象上的其它持锁者关联出来只是辅助，是否真阻塞要按 MDL 兼容矩阵人工判断（`LOCK_TYPE` 有 11 种取值，含 `SHARED_UPGRADABLE`、`SHARED_NO_WRITE` 等）。
- ⚠️ 表里**没有 `VICTIM` 也没有 `TIMEOUT`** 列：死锁受害者与锁等待超时只表现为客户端收到 `ER_LOCK_DEADLOCK` / `ER_LOCK_WAIT_TIMEOUT`。
- `OWNER_THREAD_ID` 为 NULL 的行是内部/无主锁，忽略即可。

### State 与计数器（DDL 相关速查）

- `copy to tmp table` 是 **SQL 层唯一带进度数字**的 DDL stage（COPY 算法拷数据）；INPLACE 的三个阶段（`preparing for alter table` / `altering table` / `committing alter table to storage engine`）flags=0，默认不进 PFS。
- ⚠️ 旧资料里的 `waiting for handler insert` **在 8.0.39 源码中不存在**（只有 `waiting for handler commit`）。
- `Waiting for table metadata lock` 由 `MDL_context::acquire_lock()` → `MDL_wait::timed_wait()` → `set_proc_info()` 设置，即"该连接的 MDL 请求进了等待队列"；对 DDL 就是拿 `MDL_EXCLUSIVE` / `SHARED_NO_READ_WRITE` 时被长事务或未提交 SELECT 挡住。
- `innodb_metrics` 的 `ddl` 子系统有 5 个计数器（`ddl_pending_alter_table`、`ddl_online_create_index`、`ddl_background_drop_tables`、`ddl_sort_file_alter_table`、`ddl_log_file_alter_table`），但**全是 `MONITOR_NONE` ⇒ 默认 disabled**，需 `SET GLOBAL innodb_monitor_enable='ddl'`；它们**不并入 `SHOW STATUS`**，唯一出口是 `innodb_metrics`（且需要 `PROCESS` 权限）。

### 并行 DDL 的观测手段

`Loader::load()` 里实际线程数 = `m_max_threads`，其中 **1 个是用户连接线程**，额外 `m_max_threads - 1` 个是裸 `std::thread`：

- ✅ **PFS 能看到**：以 `ddl_thread_key` 注册 ⇒ `performance_schema.threads` 中 `NAME = 'thread/innodb/ib_ddl'`，每个线程各有一条 `events_stages_current`。
- ❌ **`SHOW PROCESSLIST` 看不到**：源码显式 `current_thd = nullptr;`，不是 THD，没有 `PROCESSLIST_ID`。

### ★★ `mysql.innodb_ddl_log` 不能直接查

`ha_innodb.cc` 里 `innodb_ddl_log->set_hidden(true);` ⇒ 它是 **DD 隐藏系统表**，SQL 层不可 `SELECT`，也不出现在 `SHOW TABLES` / `information_schema.tables`。

替代观测是开 `innodb_print_ddl_logs`，但有个**必踩的坑**：`log0ddl.cc` 里这些打印点**全是 `ib::info`**（`INFORMATION_LEVEL` = 3），而 `log_error_verbosity` **默认是 2**，过滤规则是 `prio > verbosity → DROP` ⇒ **只设 `innodb_print_ddl_logs=ON` 不会有任何输出**，必须同时把 `log_error_verbosity` 设为 3。打开后 insert / delete / replay / post_ddl 全链路都会打印。

> 详见 [`../../infra/observability/logging/error_log.md`](../../infra/observability/logging/error_log.md)「★ `innodb_print_ddl_logs` 的陷阱」。

### ★ 崩溃恢复的两个坑

- `Log_DDL::recover()` 在 `srv_read_only_mode || srv_force_recovery > 0` 时**直接 return**（不回放也不清理）⇒ **一旦用 `innodb_force_recovery > 0` 启动，DDL 残留最可能留下来**；`post_ddl()` 在 `srv_force_recovery >= SRV_FORCE_NO_UNDO_LOG_SCAN` 时同样跳过。
- 错误日志里会**成对出现** `DDL log recovery : begin` … `: end`，中间每条回放在开了 `innodb_print_ddl_logs` 时有 `DDL log replay : <record>`。
- **残留确认**：DDL 崩了而进程没重启时，`ddl_pending_alter_table` / `ddl_online_create_index` 会卡住不降（正常结束和回滚分支都有 `DEC`）——这是很有用的残留信号。

---

## Misc


### `sql_table.cc` 里暴露痛点的 TODO（26 处，挑最有信息量的）

| 位置 | 原文 | 痛点 |
|---|---|---|
| `mysql_prepare_create_table()` | *"TODO: Check if the old .FRM limitations still make sense with the new DD."* | 8.0 去掉 .frm 后仍保留 MAX_FIELDS / 4G rows 等老限制 |
| `mysql_alter_table()` COPY 段 | `/* todo: ALTER TABLE using copy algorithm. */` | COPY **没有独立函数**，内联在函数体里 |
| `mysql_alter_table()` 删备份失败处 | *"This hack might become unnecessary once InnoDB stops acquiring gap locks on DD tables (which might cause deadlocks)."* | 重试 20 次是绕死锁权宜 |
| `mysql_inplace_alter_table()` 末尾 | *"TODO: May move the opening of the table and the call to ha_notify_table_changed() here..."* | 通知时机仍不完美 |
| `Drop_tables_ctx` | *"TODO: consider splitting into 2 orthogonal enum/bools."* | GTID × 表分组膨胀成 4 值 enum |
| FK children 获取处（两处） | *"TODO: In the long term, extend the Dictionary_client to support..."* | `fetch_fk_children_uncached_uncommitted_normalized()` 是性能/正确性折中 |
| InnoDB 字典缓存失效处（4 处） | *"TODO: Simply removing entries from InnoDB internal cache breaks..."* | server 层直接剔 InnoDB 内部缓存项有副作用 |

### 几个"看似简单实则复杂"

1. **`CREATE TABLE ... LIKE` 用 ALTER 的代码实现**——调 `mysql_prepare_alter_table()` 把源表"导出"成 `Alter_info`，再走标准 CREATE
2. **函数索引 = 隐藏生成列**，且必须**按列名排序放到表末尾**，还要考虑主备版本兼容
3. **键前缀长度的单位换算**（字节 vs 字符×mbmaxlen）
4. **`sort_keys()` 之后再处理外键**——为了自引用外键查找父键时行为可预测
5. **分区表即使 `no_ha_table` 也要开一次表**：*"For partitioned tables we can't find some problems with table until table is opened."*
6. **ALTER 的中间临时表要拿 `MDL_EXCLUSIVE` 锁临时名**：*"This ensures that concurrent I_S queries won't try to open it."*
7. **no-op ALTER 特判**：`handler_flags == 0` 时（如 ADD 和 DROP 同名索引互相抵消）跳过引擎调用，且忽略 LOCK 子句
8. **`Disable_binlog_guard`**：创建中间表**不记 binlog**，只记最终那条 ALTER

### server 侧与 handler 接口的关键约定

| 约定 | 证据 |
|---|---|
| **`handler::create()` 可以修改 `dd::Table`** | `ha_create_table()` 的 post-create `dd_client()->update()` |
| **`handler::rename_table()` 也可以，所以 DD update 必须在其后** | `mysql_rename_table()` 的 WL#7743 长注释 |
| **`handler::ha_create()` 前必须先有 `dd::Table`** | *"ha_create_table() won't work correctly if dd::Table object is not stored in the data-dictionary."* |
| **临时表 / DD 自身表不走 post-create update** | 同一处的排除条件 |
| **`handler::update_create_info()` 可能被引擎用来报错** | `mysql_prepare_alter_table()` 里的 NDB 注释 |
| **`post_ddl` 契约** | 提交和回滚**都要**调用，负责清理残留文件（DDL log replay 在此发生） |
| **`dict_cache_reset` 契约** | 仅在"`ha_commit_inplace_alter_table()` 已成功但随后回滚"时调用 |

---


### 引擎侧待优化点


对比云厂商（PolarDB / AliSQL）的增强，社区版 8.0.39 在 DDL 与并行方面仍有明显可优化空间：

| 优化点 | 社区版现状 | 云厂商增强 | 价值 |
|--------|-----------|-----------|------|
| **分区级 MDL** | ❌ DDL 锁整张表（**所有分区**） | ✅ PolarDB `loose_partition_level_mdl_enabled`：DDL 只锁涉及分区，不影响其他分区的 DML | 大分区表 DDL 可用性（**投入产出比最高**） |
| **并行构建 B+树** | ❌ 建索引第③阶段**单线程** | 部分优化 | 大表建索引第三阶段瓶颈 |
| **重建表扫描并行** | ❌ 单线程（主键有序，设计如此） | — | 重建表扫描阶段无法加速 |
| **并行扫描二级索引** | ❌ 仅支持聚簇索引 | ✅ AliSQL 支持 | 二级索引扫描场景 |
| **CHECK TABLE 并行** | ⚠️ 仅第二次扫描并行，第一次（结构校验）单线程 | — | 大表 CHECK 耗时 |
| **非阻塞 DDL** | ❌ 无 | ✅ PolarDB 非阻塞 DDL | 彻底避免 MDL 等待堆积 |
| **并行 DDL 加速比** | 有机制（8.0.27+） | 调优到 15-20 倍 | 大表 DDL 耗时 |

**优先级建议**：若目标是提升大表 DDL 可用性，**分区级 MDL** 投入产出比最高——它直接消除"只改一个分区却锁住全表"的痛点，且不依赖并行度调优，也不受"重建表扫描必须串行"这类架构约束限制。

---


### 引擎侧 Misc


#### 原子 DDL 与 binlog 的关系

server 层通过 `binlog_query`/GTID 保证 DDL 在 binlog 中有记录。崩溃恢复时：DDL 事务已写 binlog（已提交）→ DDL log 记录存在 → replay 完成物理操作；未写 binlog（未提交）→ undo 回滚。binlog 提交与否是"该不该 replay"的判据。

#### 原子 DDL 与 InnoDB 其他机制的关系

- clone/备份：`fil_truncate_tablespace`（fil0fil.cc:4738）用于 undo 表空间 truncate（srv0tmp.cc:143），与本机制无关
- SDI 表：删除表空间前需对 SDI 表加 MDL（`dd_sdi_acquire_exclusive_mdl`），防止 purge 并发操作 SDI（row0mysql.cc:3854）

---


---

## 参考

### 官方文档与 WorkLog

- *MySQL 8.0 Reference Manual → InnoDB and Online DDL*（各操作支持的 ALGORITHM / LOCK 矩阵，判定的"官方口径"）
- *MySQL 8.0 Reference Manual → Atomic Data Definition Statement Support*
- *MySQL 8.0 Reference Manual → CREATE / ALTER / DROP / TRUNCATE / RENAME TABLE*（语法与行为细节）
- *MySQL 8.0 Reference Manual → [Online DDL Memory Management](https://dev.mysql.com/doc/refman/8.0/en/online-ddl-memory-management.html)* — ★ `innodb_ddl_buffer_size`（8.0.27 引入，默认 1 MB）在 online DDL 各阶段如何分配临时缓冲；本篇 8.10（内存预算三函数）的"官方口径"
- 官方博客 [MySQL 8.0 – InnoDB Parallel Threads for Online DDL](https://dev.mysql.com/blog-archive/mysql80-innodb-parallel-threads-ddl/)（2021-12）— ★ 8.0.27 并行 DDL 的官方发布说明：默认 `innodb_ddl_threads=4` / `innodb_ddl_buffer_size=1048576`，并给出 `ALTER TABLE ... ADD INDEX` 的实测对比。⇒ 本篇 9.（并行 DDL）的官方依据（该重构在源码中**无 WL 号**，此博客是主要官方来源）
- 官方博客 [MySQL 8.0: Data Dictionary Architecture and Design](https://dev.mysql.com/blog-archive/mysql-8-0-data-dictionary-architecture-and-design/)（2016-10）— ★ 官方对"为什么要 atomic and crash-safe DDL"的设计说明：*"We do want to provide atomic and crashsafe DDL, and this requires changes to the MySQL server DDL code, and the InnoDB code... The MySQL server code will remove all implicit commits and implement clear atomic semantics for DDL statements."* ⇒ 与本篇 6.（原子 DDL）互为印证。（⚠️ 该文早于 8.0 GA 一年半，部分架构后续版本才实现——引用时以本仓库 8.0.39 源码为准。`server/dd/dd.md` 已通读并核对过）
- WorkLog：
  - [WL#7977](https://dev.mysql.com/worklog/task/?id=7977) *Atomic DDL* —— 本篇 6.（原子 DDL）的官方设计
  - [WL#7743](https://dev.mysql.com/worklog/task/?id=7743) *New data dictionary: changes to DDL-related...* —— ★ 官方说明"为了让 InnoDB 摆脱内部数据字典"而改 DDL；本篇 5.3 `mysql_rename_table()` **先 SE 后 DD** 的依据（源码那段长注释直接以 *"before WL#7743"* 起头）
  - [WL#9525](https://dev.mysql.com/worklog/task/?id=9525) *InnoDB_New_DD: update InnoDB specific metadata into new DD for DDL* —— ★ 官方设计"引擎在 DDL 时把 `se_private_data` / `se_private_id` / `options` 回写进 `dd::Table` / `dd::Index` / `dd::Tablespace`"；本篇 3.7（★ 引擎可改 `dd::Table`）与 [`atomic_ddl.md`](atomic_ddl.md) 的官方依据。它自述依赖 WL#7743（要先支持把 `dd::Table` 传给 InnoDB）
  - [WL#11250](https://dev.mysql.com/worklog/task/?id=11250) *Support Instant Add Column* —— ★ Instant DDL 篇的官方设计（见 [`instant_ddl.md`](instant_ddl.md)）。摘要首句即 **"Porting Tecent's patch on Instant Add Column"**（官方原文如此，Tencent 被拼成 Tecent）⇒ 该特性源自**腾讯贡献**。含记录格式设计（`REC_INFO_INSTANT_FLAG 0x80`）、四个新增 `se_private_data` 键、★ **回滚时行变大的坑**（见 [`instant_ddl.md`](instant_ddl.md)）
  - [WL#7957](https://dev.mysql.com/worklog/task/?id=7957) *Add MDL for tablespaces* —— ⚠️ **与原子 DDL 无关**（是给 tablespace 引入 MDL namespace）
  - ⚠️ **已核实不存在的编号**：WL#8347、WL#6041（官方均返回 *"Unable to Locate Task"*）——从旧资料看到请勿引用
  - ⚠️ `storage/innobase/ddl/` 的并行 DDL 重构：**源码无 WL 号**（该目录 grep `WL #` / `Work Log` 均 0 命中）⇒ 改用上面的官方博客

### MySQL 8.0.39 源码

**server 侧**

- `sql/sql_table.cc`（**773 KB**）：`mysql_create_table` / `mysql_create_table_no_lock` / `create_table_impl` / `rea_create_base_table` / `rea_create_tmp_table` / `mysql_prepare_create_table` / `prepare_create_field` / `mysql_alter_table` / `mysql_prepare_alter_table` / `prepare_fields_and_keys` / `mysql_inplace_alter_table` / `mysql_compare_tables` / `mysql_recreate_table` / `mysql_rename_table` / `mysql_rm_table` / `mysql_rm_table_no_locks` / `drop_base_table` / `write_bin_log` / `trans_intermediate_ddl_commit`
- `sql/sql_alter.h` / `.cc`：`Alter_info` / `Alter_table_ctx` / `Alter_drop` / `Alter_column` / `Sql_cmd_alter_table`
- `sql/sql_cmd_ddl_table.h` / `.cc`：`Sql_cmd_create_table::execute` / `populate_table`
- `sql/handler.h` / `.cc`（**315 KB**）：`HA_CREATE_INFO` / `Alter_inplace_info` / `ha_create_table` / `check_if_supported_inplace_alter` / `notify_alter_table_t`
- `sql/sql_rename.cc`（35 KB）/ `sql/sql_truncate.cc`（27 KB，`Sql_cmd_truncate_table::truncate_base`）/ `sql/sql_view.cc`（72 KB）/ `sql/sql_admin.cc`（`mysql_recreate_table` 的两个调用点）
- `sql/field.h` / `field.cc`：`IS_EQUAL_NO` / `IS_EQUAL_YES` / `IS_EQUAL_PACK_LENGTH` 与 `Field::is_equal()`
- `sql/create_field.h`、`sql/key_spec.h`、`sql/structs.h`
- `sql/dd/dd_table.cc`：`dd::create_table` / `dd::create_tmp_table` / `dd::drop_table`

**引擎侧**

- `storage/innobase/ddl/`（**200 KB，8.0.27+ 新架构**）：`ddl0builder.cc`（Builder 状态机）/ `ddl0loader.cc`（任务队列）/ `ddl0par-scan.cc`（并行扫描）/ `ddl0merge.cc`（多趟 2 路归并）/ `ddl0buffer.cc` / `ddl0file-reader.cc` / `ddl0ctx.cc` / `ddl0ddl.cc` / `ddl0fts.cc` / `ddl0rtree.cc`
- `storage/innobase/include/ddl0*.h`（13 个头）
- `storage/innobase/handler/handler0alter.cc`（inplace 四件套、`ddl::Context` 构造点）
- `storage/innobase/handler/ha_innodb.cc`（**845 KB**）：`thd_ddl_threads` / `thd_ddl_buffer_size` / `check_if_incompatible_data`
- `storage/innobase/row/row0log.cc`（row log）/ `row0pread.cc`（Parallel_reader）/ `dict/dict0dd.cc`
- ⚠️ `row0merge.cc` **在本版本已不存在**（索引构建已迁至上面的 `ddl/`），仅 `srv0srv.h` / `ut0stage.h` / `lob0lob.h` 残留注释中的历史引用
- **老 `row0merge.cc` 的获取方式**（本篇 8.2/8.3 的全部依据）：该文件已在提交 `e701897fdd1`（2021-05-04）被删除，但 git 里可完整取出——
  ```
  git show e701897fdd1^:storage/innobase/row/row0merge.cc      # 3975 行 / 130791 字节
  git log --diff-filter=D --format="%h %ad %s" --date=short --all -- '*row0merge*'   # 定位删除提交
  ```
  ⚠️ `row_merge_buf_t` / `merge_file_t` / `row_merge_dup_t` 的**结构体定义原文**在被一并删除的 `row0merge.h` 里（工作区无），8.2 的字段列表是**由使用点反推**的

### 内核月报 / 技术文章（阿里云 RDS 数据库内核组，`mysql.taobao.org/monthly/`）

**Online DDL 与算法**

- [MySQL · 源码阅读 · 白话Online DDL](http://mysql.taobao.org/monthly/2021/03/06/)（2021/03）— ★ COPY/INPLACE/INSTANT 对比、**MDL 三段式（X→S→X）**、"INPLACE 不一定 Online"、"INPLACE 也需要额外空间"
- [MySQL · 引擎特性 · 8.0 · DDL的那些事](http://mysql.taobao.org/monthly/2020/05/05/)（2020/05）— server 层 `dd::create_table` → `dd::Table`（`sql/dd/types/table.h` + `impl/table_impl.*`）的创建路径，`sql/dd/impl/cache` 的角色
- [云原生数据库PolarDB MySQL 8.0.2 DDL介绍](http://mysql.taobao.org/monthly/2023/09/03/)（2023/09）— ★ **四维评估框架**（锁表 / 重建表 / 秒级 / 并行）、VARCHAR 256 边界、Instant 加列限制、"无锁变更"工具切表也要 MDL-X。⚠️ **该文讲 PolarDB**：并行 DDL 非 PolarDB 独有（社区版 8.0.27+ 有 `innodb_ddl_threads`）；**分区级 MDL 是 PolarDB 私有**，社区版无
- [PolarDB ·引擎特性· DDL中MDL锁的优化和演进](http://mysql.taobao.org/monthly/2023/05/)（2023/05）

**原子 DDL / DDL log**

- [MySQL · 源码分析 · 原子DDL的实现过程](http://mysql.taobao.org/monthly/2018/03/02/)（2018/03）— 背景、8.0 DD 重构、`Dictionary_client` / `Shared_dictionary_cache` / `Storage_adapter`
- [MySQL · 源码分析 · 8.0 原子DDL的实现过程续](http://mysql.taobao.org/monthly/2018/07/)（2018/07）
- [MySQL · 源码分析 · DDL log与原子DDL的实现](http://mysql.taobao.org/monthly/2021/07/)（2021/07）
- [数据库内核月报 2015/05/06](http://mysql.taobao.org/monthly/2015/05/06/) — ★ **非原子 DDL 的经典 bug**：`ALTER TABLE ... ADD KEY` 成功但 `DROP KEY` 引擎失败 ⇒ server 层回滚 FRM 而 InnoDB 已加上该 key，**两层字典不一致**。这正是 8.0 原子 DDL 要解决的核心场景

**row log / Instant / 并行**

- [MySQL · 源码分析 · Row log分析](http://mysql.taobao.org/monthly/2022/03/02/)（2022/03）— ★ row log 四段格式、三种操作各记什么、**temp 格式**、instant add column 对 row log 的扩展
- [MySQL · 特性分析 · instant add column功能解析](http://mysql.taobao.org/monthly/2020/03/01/)（2020/03）— ★ Instant 记录头格式（**`rem0rec.cc` 源码注释里直接引用这篇**）、V1 时代的 n_fields 存储、读取时按字段数解析
- [MySQL · 引擎特性 · 8.0 Instant Add Column功能解析](http://mysql.taobao.org/monthly/2020/03/01/)（2020/03）
- [一文读懂 MySQL 并行查询 & DDL](http://www.uml.org.cn/sjjm/202409294.asp?artid=26485)（阿里技术，2024-09，8.0.37 基准）— 并行扫描（B+树子树分片、预分片+二次分片）、**并行建索引 vs 并行重建表的三阶段粒度差异**、参数配比、"并行查询 & DDL 仅限 InnoDB 层"

**MDL 与 rename**

- [MySQL · BUG分析 · Rename table 死锁分析](https://www.kancloud.cn/taobaomysql/monthly/140086)（2016/03）— ★ rename 卡在 **`fil_rename_tablespace`**（InnoDB 文件层）而非 MDL 等待点；真实死锁是 `fil_system` 的 IO 同步缺陷（设 `stop_ios` 后等 `n_pending` 归零，master 线程提交剩余 IO 时被挡，又没唤醒 IO helper 线程，三方互等）。与本篇 5.3 `mysql_rename_table()` **先 SE 后 DD** 直接相关
- [内核月报 MDL 加锁源码分析](http://mysql.taobao.org/monthly/2018/02/01/)（2018/02）— MDL 加锁路径，与本篇 2.3（三段式）互补；更完整的 MDL 剖析见 [`../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)

### 内部链接

- [`../server/dd/dd.md`](../../server/dd/dd.md)（DD 表体系与对象模型、缓存、自举、升级）
- [`../server/table.md`](../../server/table.md)（`TABLE_SHARE` / `TABLE` / 开表与表缓存）
- [`../server/handler.md`](../../server/handler.md)（server ↔ 引擎接口总览）
- [`../infra/lock/transactional/mdl.md`](../../infra/lock/transactional/mdl.md)（MDL 通用机制，本篇只写 DDL 场景下的三段式）
- [`../innodb/undo_log.md`](../../innodb/undo_log.md)（undo 本身；DDL 为何不记数据 undo 见本篇 5.4）
- [`../innodb/physical/tablespace.md`](../../innodb/physical/tablespace.md)（表空间碎片度量；B-tree 释放见本篇 5.7）
