# InnoDB 原子 DDL 与 DDL Log 深度解析

> 基于 MySQL 8.0.39 源码，涵盖原子 DDL 机制、`mysql.innodb_ddl_log` 表与 DDL log 类型、DDL log 生命周期（执行期写日志 / 提交后 replay / 崩溃恢复）、DROP TABLE 与 TRUNCATE TABLE 的 InnoDB 实现、B-tree 物理释放（btr_free 机制）。本文件内容由 undo_log.md 的"DDL 与 Undo"章节独立而成——DDL log 只是因"DDL 不记数据 undo"而生，其机制本身与 undo 无直接关系。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [为什么 DROP/TRUNCATE 不记数据 undo](#为什么-droptruncate-不记数据-undo)
- [DDL log 的存储与记录类型](#ddl-log-的存储与记录类型)
- [DDL log 生命周期](#ddl-log-生命周期)
- [DROP TABLE 流程](#drop-table-流程)
- [TRUNCATE TABLE：rename + drop + create](#truncate-tablerename--drop--create)
- [B-tree 的物理释放（btr_free 机制）](#b-tree-的物理释放btr_free-机制)
- [核心调用栈](#核心调用栈)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)

---

## 概述

### 是什么

8.0 原子 DDL 是"DDL 要么完全成功、要么完全回滚"的机制。InnoDB 侧的核心载体是 `mysql.innodb_ddl_log` 表和 `Log_DDL` 类（log0ddl.cc）：DDL 执行期把"破坏性物理操作"的意图写成日志记录，事务提交后再真正执行。

### 用途

解决 5.7 及之前 DDL 中途崩溃导致的不一致（如 `#sql-xxx.ibd` 孤儿文件、表空间状态错乱）。把 free B-tree、删 .ibd、改名等不可回滚的物理操作，用"先记录意图、提交后执行"的方式纳入事务原子性。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | DDL 非原子：崩溃可能留下孤儿 .ibd / 数据字典与实际文件不一致 |
| 8.0 | 引入 `mysql.innodb_ddl_log` 表 + `Log_DDL`（WL#7957 原子 DDL，WL#8347 DROP/TRUNCATE）；`HTON_SUPPORTS_ATOMIC_DDL`；物理操作延迟到提交后 replay |
| 8.0.20+ | DDL log 类型扩展（ALTER ENCRYPT TABLESPACE 等）；`innodb_print_ddl_logs` 调试开关 |

---

## 理论基础

### 设计模式

- **意图日志 / 延迟执行（intent log + deferred execution）**：执行期只向 DDL log 表写入"意图"记录，真正的物理操作推迟到事务提交后。破坏性操作一旦失败无法回滚，必须先记录、后执行
- **命令模式（Command）**：`DDL_Record` 是命令对象，携带类型 + 参数；`Log_DDL::replay` 按 `Log_Type` 分发到 `replay_free_tree_log` / `replay_delete_space_log` / `replay_drop_log` 等具体执行者
- **日志先行（Write-Ahead）**：物理操作执行前日志必须已持久化，与 ARIES 一致

### 相关论文

- **ARIES**（Mohan et al., 1992）：do-undo-redo 恢复模型。原子 DDL 用 DDL log（redo 性质）记录物理操作意图，配合元数据 undo，是 ARIES 思想在 DDL 上的工程化应用

### 算法与数据结构

- **DDL log 记录**：`DDL_Record` 类（log0ddl.h:78），字段 id / thread_id / type / space_id / page_no / index_id / table_id / file_path 等
- **幂等释放**：根页 `PAGE_INDEX_ID` 置 0（`BTR_FREED_INDEX_ID`），replay 前校验 index_id 匹配才释放——崩溃恢复重复 replay 安全
- **extent 级释放**：B-tree 释放粒度是文件段的一个 extent，而非逐页，一个 mtr 一个 extent 控制 mini-transaction 大小

### 类似实现对比

【待补充】PostgreSQL 的 DDL 事务化（DDL 与 DML 同事务）、Oracle 的原子 DDL 机制对比

### 历史背景

5.7 的 DDL 分两阶段（执行 + 元数据更新）且不原子，崩溃恢复靠 `innodb_force_recovery` 人工清理孤儿文件。8.0 引入原子 DDL 的动机：MySQL 官方 DD（data dictionary）表重构（WL#6041）后，元数据本身存储在 InnoDB 表里，DDL 的元数据修改天然有 undo；顺势把物理操作也纳入原子性，形成完整方案。

---

## 为什么 DROP/TRUNCATE 不记数据 undo

DROP TABLE 和 TRUNCATE TABLE **不记录用户数据的 undo log**——它们不像 `DELETE FROM t` 那样为每一行产生 undo record。原因：

1. **没有逐行 delete**：DROP/TRUNCATE 是物理操作（删表空间文件、free btree root），不是逐行删除，没有逐行 undo 的产生点
2. **不需要 MVCC 旧版本**：被删或被清空的表数据不需要一致性读的历史版本——表没了或被重建了，没有 read view 会访问旧数据
3. **DDL 隐式提交不可回滚**：DDL 语句本身是原子事务，用户层面不可回滚，不存在"回滚到 DDL 前"的需求

因此 DDL 的原子性不靠数据 undo，靠 DDL log + dd 元数据 undo（详见下文）。

---

## DDL log 的存储与记录类型

`mysql.innodb_ddl_log` 是普通 InnoDB 表。DDL log 记录通过 `DDL_Log_Table::insert` 插入（log0ddl.h:257），插入本身产生 undo + redo——这个 undo 是为了 DDL 事务失败时回滚对 `innodb_ddl_log` 表自身的修改，是元数据层面保证，与用户数据无关。

记录类型（`Log_Type` 枚举，log0ddl.h:45，uint32_t 节省表空间）：

| 类型 | 值 | 含义 | 对应 replay |
|------|----|------|-------------|
| `FREE_TREE_LOG` | 1 | 释放索引 B-tree | `replay_free_tree_log`（btr_free_if_exists） |
| `DELETE_SPACE_LOG` | 2 | 删除表空间文件 | `replay_delete_space_log`（fil_delete_tablespace） |
| `RENAME_SPACE_LOG` | 3 | 重命名表空间文件 | `replay_rename_space_log`（fil_op_replay_rename_for_ddl） |
| `DROP_LOG` | 4 | 删除 innodb_table_metadata 条目 | `replay_drop_log` |
| `RENAME_TABLE_LOG` | 5 | dict cache 中重命名表 | `replay_rename_table_log` |
| `REMOVE_CACHE_LOG` | 6 | 从 dict cache 移除表 | `replay_remove_cache_log` |
| `ALTER_ENCRYPT_TABLESPACE_LOG` | 7 | 表空间加密 | `replay_alter_encrypt_space_log` |
| `ALTER_UNENCRYPT_TABLESPACE_LOG` | 8 | 表空间解密 | 同上 |

`DDL_Log_Table` 的设计要点：**线程只访问/删除自己的记录**（按 thread_id 区分），无需行锁（log0ddl.h:237 注释）。

---

## DDL log 生命周期

### 阶段 1：执行期（DDL 事务未提交）——只写 log，不执行物理操作

向 `innodb_ddl_log` 表 insert 物理操作记录。此时物理操作本身不执行。`insert_free_tree_log` 注释说得很明确："if committed, will be redo only"（log0ddl.cc:884）——free tree 要等事务提交后通过 replay 才执行。所以 DDL 失败可以干净回滚：物理操作还没做，只需 undo 回滚元数据修改和 ddl_log 表的 insert。

### 阶段 2：提交后 post_ddl——真正执行物理操作

`Log_DDL::post_ddl(thd)`（log0ddl.cc:1905）被调用，`replay_by_thread_id`（1506）按线程 ID 查找该 DDL 写入的所有记录，`replay`（1576）根据类型执行物理操作，完成后 `delete_by_ids` 删除已处理记录。

**post_ddl 在 commit 和 rollback 之后都会被调用**（注释原文 "Replay and clean DDL logs after DDL transaction commits or rollbacks"，log0ddl.h:484）。调用者是 server 层 `hton->post_ddl(thd)`（如 TRUNCATE 走 `Sql_cmd_truncate_table::cleanup_base` sql_truncate.cc:399）→ `innobase_post_ddl`（ha_innodb.cc:5101）。rollback 场景下 replay 是空操作（见下）。

### 阶段 3：崩溃恢复——replay_all

`Log_DDL::recover`（log0ddl.cc:1942）调用 `replay_all()` 重放所有残留 DDL log：

- DDL 事务已提交（binlog 有记录）→ DDL log 记录存在 → replay 确保物理操作完成
- DDL 事务未提交 → undo 回滚该事务 → `innodb_ddl_log` 的 insert 被撤销（记录消失）→ 无 log 可 replay → 表恢复原状

### 原子性闭环的判定逻辑

原子 DDL 的本质：**物理操作通过 DDL log 延迟到提交后执行，元数据修改通过 undo 可回滚**，两者结合保证 DDL 要么完全成功（提交 + 物理操作 replay 完成），要么完全回滚（undo 回滚元数据，物理操作从未执行）。

关键闭环（回答"为什么这样实现原子性"）：

1. **执行期只写意图**：不可回滚的破坏性操作（free B-tree / 删 .ibd / rename 文件）执行期一律不做，只向 `innodb_ddl_log` insert 记录（普通表 insert，产生 undo + redo，随 DDL 事务持久化）
2. **DDL log 记录的存在性由 undo 决定**：DDL 事务未提交则记录被 undo 撤销；已提交则记录随事务落盘。因此"日志记录是否存在"本身就是"该 DDL 是否已提交"的可靠判据
3. **post_ddl / 崩溃恢复都基于该判据执行物理操作**：查得到记录 → 该 DDL 已提交 → 执行（且物理操作本身产生的 redo 保证崩溃后不丢）；查不到 → 从未提交 → 物理操作从未执行，状态天然一致
4. **为什么破坏性操作必须延迟**：若执行期就 free B-tree，DDL 失败回滚时这些页已归还 FSP_FREE 且可能被其他事务覆盖，物理上无法恢复。延迟使"回滚"永远只需撤销日志记录，无需撤销物理操作

---

## DROP TABLE 流程

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

---

## TRUNCATE TABLE：rename + drop + create

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

## B-tree 的物理释放（btr_free 机制）

DROP/TRUNCATE 时 B-tree 到底怎么删，取决于表空间类型。

### 独立表空间（file-per-table）/ 一般表空间：整文件删除，不逐页释放

`row_drop_table_for_mysql` 只写 `DELETE_SPACE_LOG`（row0mysql.cc:4126），提交后：

```
Log_DDL::replay_delete_space_log (log0ddl.cc:1659)
  → fil_delete_tablespace (fil0fil.cc:4658)
    → Fil_shard::space_delete (:4493)
      → os_file_delete (:4641)    ← 整个 .ibd 删除，B-tree 所有页随文件消失
```

注意 `row0mysql.cc:4028` 的判定 `if (!table->is_temporary() && !file_per_table)`——**独立表空间根本不写 FREE_TREE_LOG，一个页都不会释放**。

### 系统表空间 / 共享表空间：逐页释放 B-tree（FREE_TREE_LOG）

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

`fseg_free_step`（fsp0fsp.cc:3658）：一个 mini-transaction 内释放 segment 的一个分配单元（1 个完整 extent 或 1 个 frag 页），返回 false 表示未释放完，外层循环继续。核心逻辑：

- 有整 extent：`fseg_free_extent`（3591）把 extent 从 FSEG_FULL / FSEG_NOT_FULL / FSEG_FREE 链摘除 → `fsp_free_extent`（1896）`xdes_init` 复位描述符 → `flst_add_last(FSP_FREE)` 归还表空间空闲区（`space->free_len++`）；frag extent 归还 FSP_FREE_FRAG
- 只有 frag 页：`fseg_free_page_low`（3725）逐页释放
- 全部释放完：`fsp_free_seg_inode`（3720）归还 segment inode 页
- AHI：`fseg_free_extent` 内按页 `btr_search_drop_page_hash_when_freed`（3622）摘自适应哈希索引

`btr_free_root`（771-786）：先 `btr_search_drop_page_hash_index` 摘 AHI，再循环 `fseg_free_step` 释放根页所在段。

**设计思想**：free B-tree 是破坏性物理操作，页被释放后若 DDL 回滚无法恢复。用 DDL log 记录意图、提交后执行，配合元数据 undo 可回滚。崩溃恢复重放 FREE_TREE_LOG，靠 `btr_free_root_invalidate` 的 index_id=0 标记保证幂等。释放粒度是 extent 而非逐页，整 extent 一次性归还 FSP_FREE，XDES 描述符直接复位，开销小。

补充：临时表清空走 `row_delete_all_rows`（row0mysql.cc:2491）——`btr_free`（btr0btr.cc:1026，MTR_LOG_NO_REDO）+ `btr_create` 重建根页，不经过 DDL log。

---

## 核心调用栈

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

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_print_ddl_logs` | OFF | Global | 把 DDL log 的 insert/delete/replay 全流程打印到错误日志（`srv_print_ddl_logs`，ha_innodb.cc:23090），调试原子 DDL 崩溃问题 |

### 状态变量

无专门状态变量。

---

## Misc

### 原子 DDL 与 binlog 的关系

server 层通过 `binlog_query`/GTID 保证 DDL 在 binlog 中有记录。崩溃恢复时：DDL 事务已写 binlog（已提交）→ DDL log 记录存在 → replay 完成物理操作；未写 binlog（未提交）→ undo 回滚。binlog 提交与否是"该不该 replay"的判据。

### 原子 DDL 与 InnoDB 其他机制的关系

- clone/备份：`fil_truncate_tablespace`（fil0fil.cc:4738）用于 undo 表空间 truncate（srv0tmp.cc:143），与本机制无关
- SDI 表：删除表空间前需对 SDI 表加 MDL（`dd_sdi_acquire_exclusive_mdl`），防止 purge 并发操作 SDI（row0mysql.cc:3854）

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `row0mysql.cc:3771` | `row_drop_table_for_mysql`：DROP TABLE InnoDB 侧实现 |
| `row0mysql.cc:4032` | `write_free_tree_log`：DROP 记录释放 btree（仅共享表空间） |
| `row0mysql.cc:4049` | `index->page = FIL_NULL`：dict cache 逻辑删除索引 |
| `row0mysql.cc:4115` | `write_drop_log`：DROP 记录 table_id |
| `row0mysql.cc:4126` | `write_delete_space_log`：DROP 记录删 .ibd（仅独立表空间） |
| `row0mysql.cc:2491` | `row_delete_all_rows`：临时表删全行 = btr_free + btr_create |
| `log0ddl.h:45` | `Log_Type` 枚举：8 种 DDL log 类型 |
| `log0ddl.h:78` | `DDL_Record`：DDL log 记录类 |
| `log0ddl.h:237` | `DDL_Log_Table`：innodb_ddl_log 表封装（线程隔离，无行锁） |
| `log0ddl.cc:853` | `write_free_tree_log`：写 FREE_TREE_LOG |
| `log0ddl.cc:920` | `insert_free_tree_log`：DDL 事务写 innodb_ddl_log |
| `log0ddl.cc:1576` | `Log_DDL::replay`：按类型 replay 物理操作 |
| `log0ddl.cc:1625` | `replay_free_tree_log`：FREE_TREE_LOG replay → btr_free_if_exists |
| `log0ddl.cc:1659` | `replay_delete_space_log`：DELETE_SPACE_LOG replay → fil_delete_tablespace |
| `log0ddl.cc:1506` | `replay_by_thread_id`：按线程 ID 找该 DDL 的记录并 replay |
| `log0ddl.cc:1905` | `Log_DDL::post_ddl`：提交/回滚后 replay（commit 和 rollback 后都被调用） |
| `log0ddl.cc:1942` | `Log_DDL::recover`：崩溃恢复 replay_all |
| `ha_innodb.cc:5101` | `innobase_post_ddl`：InnoDB 侧 post_ddl 钩子实现 |
| `sql_truncate.cc:399` | `Sql_cmd_truncate_table::cleanup_base`：commit 后调 hton->post_ddl |
| `btr0btr.cc:1010` | `btr_free_if_exists`：幂等释放持久化索引树 |
| `btr0btr.cc:958` | `btr_free_but_not_root`：释放 B-tree 非根页（先 LEAF 段后 TOP 段） |
| `btr0btr.cc:771` | `btr_free_root`：释放根页所在段 |
| `btr0btr.cc:795` | `btr_free_root_invalidate`：根页 PAGE_INDEX_ID 置 0（幂等标记） |
| `btr0btr.cc:1026` | `btr_free`：临时表空间释放索引树（MTR_LOG_NO_REDO） |
| `fsp0fsp.cc:3658` | `fseg_free_step`：一个 mtr 释放 segment 一个 extent/frag 页 |
| `fsp0fsp.cc:3591` | `fseg_free_extent`：extent 从段链摘除 → 归还表空间 |
| `fsp0fsp.cc:1896` | `fsp_free_extent`：extent 挂入 FSP_FREE 列表（free_len++） |
| `fil0fil.cc:4658` | `fil_delete_tablespace`：删除表空间（删 .ibd） |
| `fil0fil.cc:4493` | `Fil_shard::space_delete`：缓存清理 + os_file_delete |
| `ha_innodb.cc:5174` | `HTON_CAN_RECREATE`：InnoDB TRUNCATE 走 recreate |
| `ha_innodb.cc:14558` | `innobase_truncate::truncate`：rename+drop+create |
| `ha_innodb.cc:23090` | `innodb_print_ddl_logs` 系统变量注册 |
| `sql_truncate.cc:468` | `Sql_cmd_truncate_table::truncate_base`：server 层 TRUNCATE |
