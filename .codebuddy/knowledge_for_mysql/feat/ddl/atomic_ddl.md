# 原子 DDL 深度剖析（server ↔ InnoDB 跨层全链路）

> 基于 MySQL 8.0.39 源码，剖析原子 DDL 的完整闭环：server 侧编排与崩溃安全的五条机制、失败时的回滚路径、引擎侧的 `dd::Table` 条件持久化，以及 DDL log（`mysql.innodb_ddl_log`）的**两个相反方向的语义**（CREATE 类"写后即删"用于回滚、DROP 类"提交后 replay"用于前滚）。
>
> **边界**：原子性是 DDL 的**横切属性**（所有 DDL 类型都受它保障），故独立成篇；本篇**写完两层**（server 编排 + 引擎 DDL log）。各操作类型的编排骨架见 [`ddl.md`](ddl.md)；三条执行路见 [`copy_ddl.md`](copy_ddl.md)、[`inplace_ddl.md`](inplace_ddl.md)、[`instant_ddl.md`](instant_ddl.md)；被 DDL log 延迟执行的典型物理操作——B-tree 释放（btr_free）与 .ibd 删除——见 [`ddl.md`](ddl.md) 的 DROP / TRUNCATE 章。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路：四阶段与 DDL log 的两个方向](#主链路四阶段与-ddl-log-的两个方向)
  - [server 侧编排：为什么这些改动能在同一事务](#server-侧编排同事务提交的可行性)
  - [崩溃安全的五条机制](#崩溃安全的五条机制)
  - [引擎侧一半：`dd::Table` 修改的条件持久化](#引擎侧一半ddtable-修改的条件持久化)
  - [DDL log：存储、记录类型与幂等性](#ddl-log存储记录类型与幂等性)
  - [失败与崩溃恢复路径](#失败与崩溃恢复路径)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

原子 DDL 把一次 DDL 关联的**数据字典更新、存储引擎操作、binlog 写入**合并成单个原子操作：要么全提交（三处一致持久化），要么全回滚——即使操作过程中服务器 halt。

### 用途

解决的是 DDL 中断造成的"三处不一致"：数据字典（DD）、存储引擎、binlog 三者状态对不上，或留下孤儿文件（.ibd 已建但 DD 里没记录，反之亦然）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | 无原子 DDL。元数据散落在 `.frm` 文件、非事务系统表、引擎自有字典中 ⇒ DDL 必须**中间提交** ⇒ 中断即留下不一致；`DROP TABLE t1,t2` 中 t2 不存在时 t1 仍被删掉 |
| 8.0 | **新数据字典**（元数据事务化）+ 原子 DDL：[WL#7743](https://dev.mysql.com/worklog/task/?id=7743)（SE API 改造，把 `dd::Table` 传给引擎）、[WL#9525](https://dev.mysql.com/worklog/task/?id=9525)（引擎元数据回写 DD）、[WL#9536](https://dev.mysql.com/worklog/task/?id=9536)（crash-safe DDL，即 DDL log 机制） |
| 8.0.21 | `CREATE TABLE ... SELECT` 在 row-based 复制下**记为一个事务**（此前是两个：建表 + 插数据，中间崩溃会复制出空表），因此可与 GTID 复制共存 |

> ★ **原子 DDL ≠ 事务性 DDL**：DDL 语句仍然隐式结束当前会话事务（相当于先 `COMMIT`），不能在事务里被 `ROLLBACK` 回滚。原子性指的是"这条 DDL 自身要么全成要么全败"，不是"DDL 可以参与用户事务"。

**8.0.21 "CTAS 记为一个事务" 的源码落点**（`sql_insert.cc:3166`，`Query_result_insert::send_eof()` 之外）：

```cpp
  if (!table->s->tmp_table) {
    thd->get_stmt_da()->set_overwrite_status(true);
    error = trans_commit_stmt(thd) || trans_commit_implicit(thd);   // ★ 一次提交同时带上"建表"与"INSERT 的行"
    thd->get_stmt_da()->set_overwrite_status(false);
  }
  ...
  if (!error && m_post_ddl_ht) {
    m_post_ddl_ht->post_ddl(thd);      // ★ 成功路径 post_ddl
  }
```

失败侧（`:3304`）：

```cpp
  if (!(create_info->options & HA_LEX_CREATE_TMP_TABLE)) {
    trans_rollback_stmt(thd);
    /* Rollback transaction both to clear THD::transaction_rollback_request
    (if it is set) and to synchronize DD state in cache and on disk (as
    statement rollback doesn't clear DD cache of modified uncommitted objects). */
    trans_rollback_implicit(thd);
    if (m_post_ddl_ht) m_post_ddl_ht->post_ddl(thd);   // ★ 失败路径也调
  }
```

> ★ **它的 GTID 安全性由 `HTON_SUPPORTS_ATOMIC_DDL` 直接决定**（`binlog.cc:10676`）：

```cpp
    is_create_table_atomic =
        get_default_handlerton(this, lex->create_info->db_type)->flags &
        HTON_SUPPORTS_ATOMIC_DDL;
  ...
  if (is_create_table_select && !is_create_temporary_table &&
      !is_create_table_atomic) {
    /* CREATE ... SELECT (without TEMPORARY) for engines not supporting
    atomic DDL is unsafe because if binlog_format=row it will be
    logged as a CREATE TABLE followed by row events, re-executed
    non-atomically as two transactions, and then written to the
    slave's binary log as two separate transactions with the same
    GTID. */
    return handle_gtid_consistency_violation(
        this, ER_GTID_UNSAFE_CREATE_SELECT, ER_RPL_GTID_UNSAFE_STMT_CREATE_SELECT);
  }
```

**⚠️ 只有 InnoDB 与 NDB 设置了 `HTON_SUPPORTS_ATOMIC_DDL`**（`storage/` 全目录 grep）：

```cpp
// ha_innodb.cc:5174
  HTON_SUPPORTS_FOREIGN_KEYS | HTON_SUPPORTS_ATOMIC_DDL |
// ha_ndbcluster.cc:12661
  HTON_SUPPORTS_FOREIGN_KEYS | HTON_SUPPORTS_ATOMIC_DDL;
```

> MyISAM / MEMORY / ARCHIVE / CSV / BLACKHOLE / PERFORMANCE_SCHEMA 等**均未设置**。宏本体在 `handler.h:2915`，值 `1 << 12`；配套内联判定 `ddl_is_atomic(hton)`（`handler.h:2957`）。

---

## 理论基础

### 设计思想与权衡

**为什么 8.0 之前做不到**：元数据在 `.frm` 文件和 MyISAM 系统表里，文件与非事务表都**无法随事务回滚**，所以 DDL 不得不分步提交。8.0 把元数据搬进 InnoDB 事务表（`mysql.*`）后，"元数据变更"本身变成了普通 DML，`dd_client()->store()` 天然带 undo/redo——**原子性的地基是数据字典的事务化**，不是 DDL log 本身。

**核心手法：不可回滚的物理操作不立即做**——但要注意它有两种**相反方向**，这是本篇最容易被讲漏的一点：

| | **CREATE 类**（CREATE TABLE、ADD INDEX） | **DROP 类**（DROP TABLE、TRUNCATE、rebuild） |
|---|---|---|
| log 写在哪个阶段 | **Prepare**（执行期） | **Commit**（提交时） |
| log 的用途 | **为了回滚**（undo 语义） | **为了前滚**（redo 语义） |
| 写入后立刻做什么 | 主事务**立即 DELETE 掉这条记录** | 不删，留给 post-DDL |
| DDL 成功 | DELETE 随主事务提交生效 ⇒ log 表干净 ⇒ **什么都不用做** | post-DDL **replay 真正执行**删文件 / free 树 |
| DDL 失败或崩溃 | DELETE 未生效 ⇒ log 保留 ⇒ replay **清理刚建的 .ibd / 刚建的树** | log 保留 ⇒ 恢复后 replay 前滚 |

源码里 CREATE 方向的那句删除，注释写得直白（log0ddl.cc，紧跟 `insert_free_tree_log` 之后）：

```cpp
/* Delete this operation if the create trx is committed */
err = delete_by_id(trx, id, false);
```

两个方向复用同一张表、同一套 replay 代码，**仅靠"写在哪个阶段"和"是否被主事务删除"区分语义**——这是整个机制最精巧的地方。

**代价与约束**：

- **DDL 仍隐式提交当前事务**（原子 ≠ 事务性），用户无法把 DDL 包进自己的事务。
- **`innodb_ddl_log` 的 redo 无视 `innodb_flush_log_at_trx_commit` 强制刷盘**（WL#9536 FR5）：否则会出现"数据文件被 DDL 改了，但 DDL log 的 redo 没落盘"，恢复/回滚时出错。这是一条容易被忽略的隐含开销。
- **引擎必须自己实现 replay**，server 无法单独保证原子性——目前只有 InnoDB 支持；不支持的引擎被"豁免"（`HTON_SUPPORTS_ATOMIC_DDL` 未置位），其 DDL 仍可能留下不一致。

### 理论溯源与算法

DDL log 就是 WAL/"意图日志"思想在 DDL 场景的落地：与 ARIES 的对应关系是——**元数据用 undo**（事务化 DD 表，靠 undo 回滚），**物理操作用意图日志 + 延迟执行/撤销**（不可回滚的操作不进事务，只记意图）。

**幂等性是硬性要求**（WL#9536 原文：*"All operations will be implemented in an idempotent way"*）——因为 replay 可能重复执行（崩溃恢复后再次 replay）。

**简化假设的失效场景**：DDL 期间服务器 halt ⇒ log 残留。WL#9536 FR2a 的约束是"崩溃恢复完成后、开始任何新 DDL 之前，`DDL_LOG` 表必须为空"——这条不变量是所有后续 DDL 正确性的前提。

**★ 幂等性的三种落地方式**（8.0.39 各有一处源码）：

| 方式 | 位置 | 机制 |
|---|---|---|
| **页面内容编码** | `btr_free_root_check` / `btr_free_root_invalidate` | 释放后把 root 页的 `PAGE_INDEX_ID` 写成 `BTR_FREED_INDEX_ID(0)`；重放时比对不上就跳过 |
| **"没找到 = 已完成"** | `row_drop_tablespace` / `DDTableBuffer::remove` / `DDL_Log_Table::remove` | 容忍 `DB_TABLESPACE_NOT_FOUND`、定位到 infimum 就跳过，**恒返回 `DB_SUCCESS`** |
| **无操作容忍** | `fil_op_replay_rename_for_ddl` | 找不到 space 直接 `return true`（成功） |

```cpp
// btr0btr.cc:788 —— ★ 页面内容编码（不需要额外的"已处理"记录表）
/** PAGE_INDEX_ID value for freed index B-trees */
static const space_index_t BTR_FREED_INDEX_ID = 0;   // ⚠️ file-scope static，头文件未导出

[[nodiscard]] static buf_block_t *btr_free_root_check(
    const page_id_t &page_id, const page_size_t &page_size,
    space_index_t index_id, mtr_t *mtr) {
  ut_ad(!fsp_is_system_temporary(page_id.space()));
  ut_ad(index_id != BTR_FREED_INDEX_ID);
  buf_block_t *block =
      buf_page_get(page_id, page_size, RW_X_LATCH, UT_LOCATION_HERE, mtr);
  buf_block_dbg_add_level(block, SYNC_TREE_NODE);
  if (fil_page_index_page_check(block->frame) &&
      index_id == btr_page_get_index_id(block->frame)) {
    /* This should be a root page.
    It should not be possible to reassign the same
    index_id for some other index in the tablespace. */
    ut_ad(page_is_root(block->frame));
  } else {
    block = nullptr;      // ★ 已释放过（PAGL_INDEX_ID==0）→ 返回 NULL → 幂等跳过
  }
  return (block);
}

void btr_free_if_exists(const page_id_t &page_id, const page_size_t &page_size,
                        space_index_t index_id, mtr_t *mtr) {
  buf_block_t *root = btr_free_root_check(page_id, page_size, index_id, mtr);
  if (root == nullptr) {
    return;                                  // ★★ 幂等核心
  }
  btr_free_but_not_root(root, mtr->get_log_mode());
  btr_free_root(root, mtr);
  btr_free_root_invalidate(root, mtr);       // ★ 写 BTR_FREED_INDEX_ID，闭环
}
```

> ⚠️ **`btr_free_if_exists()` 不开 mtr**——由调用方（`replay_free_tree_log`）传入 `mtr_t*`。
> ⚠️ **临时表不做 invalidate**：`btr_create()` 回滚路径里 `if (!index->table->is_temporary())` 才调 `btr_free_root_invalidate()`（`btr0btr.cc:893`）——临时表空间用完即弃，无需为崩溃重放做幂等标记。

```cpp
// dict0dict.cc:5389 —— DDTableBuffer::remove：无论删没删到都返回 DB_SUCCESS
dberr_t DDTableBuffer::remove(table_id_t id) {
  ...
  pcur.open(m_index, 0, m_search_tuple, PAGE_CUR_LE,
            BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE, &mtr, UT_LOCATION_HERE);
  if (!page_rec_is_infimum(pcur.get_rec()) &&
      pcur.get_low_match() == m_index->n_uniq) {   // ★ 不存在就整块跳过
    btr_cur_pessimistic_delete(&error, false, pcur.get_btr_cur(), ...);
    ut_ad(error == DB_SUCCESS);
  }
  mtr.commit();
  return (DB_SUCCESS);      // ★★ 恒成功 —— 幂等契约
}
```

```cpp
// row0mysql.cc:3729 —— 容忍"已经删过了"
dberr_t row_drop_tablespace(space_id_t space_id, const char *filepath) {
  if (!fil_space_exists_in_mem(space_id, nullptr, true, false)) {
    /* Force a delete of any discarded or temporary files. */
    if (fil_delete_file(filepath)) { ... }
  } else {
    err = fil_delete_tablespace(space_id, BUF_REMOVE_NONE);
    if (err != DB_SUCCESS && err != DB_TABLESPACE_NOT_FOUND) {   // ★★ 容忍点
      ib::error(ER_IB_MSG_991) << "Failed to delete the datafile ...";
    }
  }
  return (err);
}
```

> ★ `Fil_shard::space_delete()` 里 `ut_a(err == DB_TABLESPACE_NOT_FOUND)`——**只允许这一种失败码**，上层才能安全吞掉。
> ★ 删文件前会**先写一条 `MLOG_FILE_DELETE` redo**（`Fil_shard::space_delete` 内 `fil_op_write_log`），注释原文：*"Before deleting the file, write a log record about it, so that InnoDB crash recovery will expect the file to be gone."*

**★ 崩溃注入点是幂等性的测试保证**——`DBUG_INJECT_CRASH` 是 InnoDB 自己的包装（`ut0dbg.h:85`），把 `prefix` + 计数器拼成 `"prefix_%u"` 再喂给 `DBUG_EXECUTE_IF`，**release 版展开为空**：

```cpp
#ifdef UNIV_DEBUG
#define DBUG_INJECT_CRASH(prefix, count)            \
  do {                                              \
    char buf[64];                                   \
    snprintf(buf, sizeof buf, prefix "_%u", count); \
    DBUG_EXECUTE_IF(buf, DBUG_SUICIDE(););          \
  } while (0)
#define DBUG_INJECT_CRASH_WITH_LOG_FLUSH(prefix, count)                \
  do { /* 先 flush redo 再自杀：构造"redo 已落盘但 log 未删"的崩溃点 */ \
    ... DBUG_EXECUTE_IF(buf, log_buffer_flush_to_disk(); DBUG_SUICIDE();); \
  } while (0)
#else
#define DBUG_INJECT_CRASH(prefix, count)                 // ★ release 完全为空
#endif
```

> ★ 全仓共 **20 处 / 10 个名字**，其中 `ddl_log_crash_after_replay` 出现在 5 个 replay 函数末尾（1657/1720/1748/1801/1811）——**这正是"回放完了、还没来得及删记录就崩"的注入点**，下次启动必须重放一遍。
> ⚠️ **`RENAME_TABLE_LOG` 与 `REMOVE_CACHE_LOG` 没有任何崩溃注入点**——源码注释明说原因：*"Other RENAME_TABLE and REMOVE CACHE doesn't touch the data files at all, so would be skipped"*。
> ⚠️ `ddl_log_crash_reset()` 只重置 **11 个**计数器——`crash_before/after_alter_encrypt_space_log_counter` **没被重置**（源码如此）。

---

## 核心实现

### 主链路：四阶段与 DDL log 的两个方向

官方（WL#9536 / 文档 15.1.1）把一次 DDL 划分为四阶段，`ALTER TABLE` 这类操作**Prepare 与 Perform 可以执行多次**（多个子事务）：

```
① Prepare  创建所需对象（文件、B-tree），写 DDL log（CREATE 方向：为回滚做准备）
② Perform  执行实际操作（如填充索引树、拷贝数据）
③ Commit   更新数据字典并提交 DD 事务（DROP 方向在此写 log）
④ Post-DDL 回放并删除 DDL log —— 真正执行文件重命名/删除、B-tree 释放
```

各操作在这四阶段的分布（WL#9536 §1–§6）：

| 操作 | Prepare 写 | Commit 写 | Post-DDL 做什么 |
|---|---|---|---|
| CREATE TABLE | `DELETE`（.ibd）+ 每棵树 `FREE` + `REMOVE_CACHE` | — | 成功时记录已被删，无操作；失败时清理刚建对象 |
| DROP TABLE | — | `DELETE`（file-per-table）或每索引 `FREE`（共享表空间） | 删文件 / free 索引树 |
| ALTER ... ADD INDEX | 每索引 `FREE` | — | 同上（失败时清理） |
| ALTER ... DROP INDEX | — | `FREE` | free 索引树 |
| ALTER 重建表 | 同 CREATE（新表） | 同 DROP（旧表）+ `DROP`（动态元数据） | 删旧文件 + 清动态元数据 |
| TRUNCATE | `RENAME` + 同 CREATE | 同 DROP | 删旧表文件 |
| RENAME TABLE | `RENAME`（为回滚） | — | 回滚时改回原名 |

**★ "四阶段"的权威出处是 `handler.h:6015-6132` 的三阶段契约注释**，其中 Phase 3 才区分原子/非原子——这是原子 DDL 编排最本质的差别：

```cpp
    Phase 3 : Final
    ===============

    a) For engines which don't support atomic DDL:

       *) Update the SQL-layer data-dictionary by replacing description of old
          version of the table with its new version. This change is immediately
          committed.
       *) Inform the storage engine about this change by calling the
          handler::ha_notify_table_changed() method.
       *) Process the RENAME clause by calling handler::ha_rename_table() and
          updating the data-dictionary accordingly. Again this change is
          immediately committed.
       *) Destroy the Alter_inplace_info and handler_ctx objects.

    b) For engines which support atomic DDL:

       *) Update the SQL-layer data-dictionary by replacing description of old
          version of the table with its new version.
       *) Process the RENAME clause by calling handler::ha_rename_table() and
          updating the data-dictionary accordingly.
       *) Commit the statement/transaction.
       *) Finalize atomic DDL operation by calling handlerton::post_ddl() hook
          for the storage engine.
       *) Additionally inform the storage engine about completion of ALTER TABLE
          for the table by calling the handler::ha_notify_table_changed() method.
       *) Destroy the Alter_inplace_info and handler_ctx objects.
```

> ★★ **顺序差异**：非原子 = `更新 DD（立即提交）→ notify → rename（立即提交）`；原子 = `更新 DD（不提交）→ rename → 提交事务 → post_ddl → notify`。**notify 在原子路径里排到 post_ddl 之后**。

源码对应（`sql_table.cc:13793`，ALTER INPLACE 提交段）：

```cpp
  if (db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) {
    ...
    Implicit_substatement_state_guard guard(thd, mode);
    /* Commit ALTER TABLE. Needs to be done here and not in the callers
    (which do it anyway) to be able notify SE about changed table. */
    if (trans_commit_stmt(thd) || trans_commit_implicit(thd)) goto cleanup2;
    /* Call SE DDL post-commit hook. */
    if (db_type->post_ddl) db_type->post_ddl(thd);
    ...
    new_table_list.table->file->ha_notify_table_changed(ha_alter_info);   // ★ notify 在 post_ddl 之后
```

> ⚠️ **勘误**：Phase 2 里还有一条关键注释常被漏读——*“Engines that support atomic DDL only prepare for the commit during this step but do not finalize it. Real commit happens later when the whole statement is committed. Also in some situations statement might be rolled back after call to commit_inplace_alter_table() for such storage engines.”* 这就是 `rollback_needs_dict_cache_reset` 存在的原因。

**`post_ddl` 的全部 server 侧调用点**（★ 覆盖 commit 与 rollback 两条路）：

| 位置 | 场景 |
|---|---|
| `sql_table.cc:3742` | DROP 成功（`for (handlerton *hton : *post_ddl_htons) hton->post_ddl(thd);`） |
| `sql_table.cc:3786` | DROP 失败（回滚后仍调，让 InnoDB 清残留文件） |
| `sql_table.cc:10356` | CREATE TABLE（★ **成功和失败都调**，只有 `m_transactional_ddl` 时跳过） |
| `sql_table.cc:13822` / `:13869` | ALTER INPLACE 提交后 / 回滚后 |
| `sql_table.cc:18196-18199` | ALTER COPY 成功（**新旧两个引擎都调**） |
| `sql_table.cc:18296` / `:18338-18341` | ALTER COPY 两个失败出口 |
| `sql_table.cc:11564` | SECONDARY_LOAD/UNLOAD 的 cleanup lambda（scope guard 保证两条路都执行） |
| `sql_insert.cc:3184` / `:3308` | CTAS 成功 / 失败 |
| `transaction.cc:297` / `:430` | `trans_commit()` / `trans_rollback()` 内部（★ `m_transactional_ddl.post_ddl()`） |

> ⚠️ **勘误：CREATE TABLE 成功时不是"不调"**。源码是 `if (!create_info->m_transactional_ddl && post_ddl_ht) post_ddl_ht->post_ddl(thd);`——**无条件调用**，注释说的 "mostly relevant for case when statement is rolled back" 只是说**主要**对回滚有意义。只有 `CREATE TABLE ... START TRANSACTION` 场景跳过（交给 `trans_commit`/`trans_rollback` 内的 `Transactional_ddl_context::post_ddl()`）。

```cpp
// sql_table.cc:11562 —— 二级引擎用 scope guard 把"无论 commit 还是 rollback"写死
  auto cleanup = [thd, hton]() {
    hton->post_ddl(thd);
    return thd->locked_tables_mode && thd->locked_tables_list.reopen_tables(thd);
  };
  auto rollback_guard = create_scope_guard([thd, cleanup] {
    trans_rollback_stmt(thd); trans_rollback_implicit(thd); cleanup();
  });
```

### server 侧编排：同事务提交的可行性

**为什么 DD 变更能在同一事务**：DD 本身就是 InnoDB 表（`mysql.*`），`dd_client()->store()` 就是对它们的普通 DML ⇒ 天然有 undo/redo。

**为什么引擎操作也在同一事务**：引擎的 `ha_create_table()` / `ha_rename_table()` 等回调可以修改 `dd::Table`（填 `se_private_data`），这些修改通过调用点之后的 `dd_client()->update()` 落回同一事务。

**为什么 `.ibd` 的删建必须延后到提交后**：那是**不可回滚的物理操作**，只能在提交成功后由 `post_ddl` 钩子 replay DDL log 执行。

**为什么 binlog 要分 trx cache / stmt cache**：

```cpp
/*
  If the SE supports atomic DDL, we can use the trx binlog cache.
  Otherwise we must use the statement cache.
*/
```

### 崩溃安全的五条机制

1. **DD 变更 = 同一事务里的 InnoDB DML**
2. ★ **`do_not_store_in_dd` 技巧**（ALTER 专用）——新表版本的 `dd::Table` 先只存在于内存，**直到旧定义被 drop 之后才 `store()`**：

```cpp
/*
  For engines supporting atomic DDL we have delayed storing new
  table definition in the data-dictionary so far in order to avoid
  conflicts between old and new definitions on foreign key names.
  Since the old table definition is gone we can safely store new
  definition now.
*/
```

这样绕开了"新旧外键同名冲突"。`create_table_impl()` 里有三条 assert 约束这个前提：

```cpp
assert(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
assert(no_ha_table);
assert(!dd::get_dictionary()->get_dd_table(db, table_name));
```

3. **`atomic_replace` 双引擎判定**（COPY 路径）：

```cpp
/*
  Atomic replacement of the table is possible only if both old and new
  storage engines support DDL atomicity.
*/
bool atomic_replace = (new_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
                      (old_db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
```

4. **提交后钩子 `post_ddl` + 缓存复位**：

```cpp
/*
  InnoDB requires additional SE dictionary cache invalidation if we roll back
  after successful call to handler::ha_commit_inplace_alter_table().
*/
if (rollback_needs_dict_cache_reset) {
  if (db_type->dict_cache_reset != nullptr)
    db_type->dict_cache_reset(alter_ctx->db, alter_ctx->table_name);
}
```

5. **备库上的特殊开关**（避免 slave info 表参与 DDL 事务）：

```cpp
/**
  RAII class to control the atomic DDL commit on slave.
  A slave context flag responsible to mark the DDL as committed is
  raised and kept for the entirety of DDL commit block.
  While DDL commits the slave info table won't take part
  in its transaction.
*/
class Disable_slave_info_update_guard { ... };
```

### 引擎侧一半：`dd::Table` 修改的条件持久化

原子 DDL 常被理解为"DDL log + 提交后 replay"（见「DDL log」节）。但还有一半藏在 handler 接口的注释里（handler.h 多处重复强调）：

```cpp
/* truncate 的注释 handler.h */
@note  Changes to dd::Table object done by this method will be saved
       to data-dictionary only if storage engine supports atomic DDL
       (i.e. has HTON_SUPPORTS_ATOMIC_DDL flag set).

/* rename_table 的注释 handler.h */
Storage engines which support atomic DDL (i.e. having
HTON_SUPPORTS_ATOMIC_DDL flag set) are allowed to adjust this object.
```

**机制**：引擎的 handler 方法会**修改传入的 `dd::Table` 对象**（填 se_private_data、改表空间信息等），但这些修改**只有引擎声明支持原子 DDL 时才会被持久化到数据字典**。

**为什么**：持久化是随 DDL 事务提交的。不支持原子 DDL 的引擎，DDL 失败时无法保证"DD 修改"与"引擎内部状态"一起回滚，所以 server 干脆**不接受它对 DD 的修改**——避免出现"DD 已改、引擎没改"的不一致。

这也解释了两个默认实现（handler.h）：

```cpp
virtual int truncate(dd::Table *table_def) { return HA_ERR_WRONG_COMMAND; }
virtual int optimize(THD *, HA_CHECK_OPT *) { return HA_ADMIN_NOT_IMPLEMENTED; }
```

- `optimize` 不实现 → InnoDB 返回 `HA_ADMIN_TRY_ALTER`，让 server 改写为 `ALTER TABLE ... ENGINE=InnoDB`（这正是 OPTIMIZE 的真实实现路径）
- `truncate` 不实现 → 走 `HTON_CAN_RECREATE` 的 recreate 路径

### DDL log：存储、记录类型与幂等性

#### 存储与 schema

`mysql.innodb_ddl_log` 是**硬编码在数据字典表空间（`mysql.ibd`）里的隐藏 InnoDB 表**，不允许对它做 DDL，非 debug 版本也不能直接用 SQL 访问（官方文档 15.1.1 明确说明）。schema 对应：

```sql
CREATE TABLE mysql.innodb_ddl_log (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  thread_id BIGINT UNSIGNED NOT NULL,
  type INT UNSIGNED NOT NULL,   -- FREE / DELETE / RENAME / DROP ...
  space_id INT UNSIGNED,
  page_no INT UNSIGNED,
  index_id BIGINT UNSIGNED,
  table_id BIGINT UNSIGNED,
  old_file_path VARCHAR(512),
  new_file_path VARCHAR(512),
  KEY(thread_id)
);
```

DDL log 记录通过 `DDL_Log_Table::insert` 插入（log0ddl.h），插入本身产生 undo + redo——这个 undo 是为了 DDL 事务失败时回滚对 `innodb_ddl_log` 表自身的修改，属元数据层面，与用户数据无关。

**replay 的分发**（`Log_DDL::replay`）——一张表、八种类型、一个 switch：

```cpp
dberr_t Log_DDL::replay(DDL_Record &record) {
  if (srv_print_ddl_logs) {
    ib::info(ER_IB_MSG_654) << "DDL log replay : " << record;   // ★ innodb_print_ddl_logs 的落点
  }

  switch (record.get_type()) {
    case Log_Type::FREE_TREE_LOG:
      replay_free_tree_log(record.get_space_id(), record.get_page_no(),
                           record.get_index_id());
      break;
    case Log_Type::DELETE_SPACE_LOG:
      replay_delete_space_log(record.get_space_id(), record.get_old_file_path());
      break;
    case Log_Type::RENAME_SPACE_LOG:
      replay_rename_space_log(record.get_space_id(), record.get_old_file_path(),
                              record.get_new_file_path());
      break;
    case Log_Type::DROP_LOG:
      replay_drop_log(record.get_table_id());
      break;
    case Log_Type::RENAME_TABLE_LOG:
      replay_rename_table_log(record.get_old_file_path(),
                              record.get_new_file_path());
      break;
    case Log_Type::REMOVE_CACHE_LOG:
      replay_remove_cache_log(record.get_table_id(), record.get_new_file_path());
      break;
    case Log_Type::ALTER_ENCRYPT_TABLESPACE_LOG:
    case Log_Type::ALTER_UNENCRYPT_TABLESPACE_LOG:
      err = replay_alter_encrypt_space_log(record);
      break;
    default:
      ut_error;                    // ★ 未知类型直接崩：新增类型必须同步改这里
  }

  return (err);
}
```

★ 两个细节：① `srv_print_ddl_logs` 就在这里生效（每 replay 一条打一条），是 `innodb_print_ddl_logs` 输出的直接来源；② `default: ut_error`——**新增一种 `Log_Type` 却忘记加分支会直接崩**，扩展时必须同步改这个 switch（见 Misc）。

回放函数本身也处处体现幂等，例如 `replay_free_tree_log` 开头就把"文件已不在"当作已完成：

```cpp
void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no,
                                   ulint index_id) {
  bool found;
  const page_size_t page_size(fil_space_get_page_size(space_id, &found));

  /* Skip if it is a single table tablespace and the .ibd file is missing */
  if (!found) {                    // ★ 表空间都没了 → 没什么可 free 的，直接返回
    if (srv_print_ddl_logs) {
      ib::info(ER_IB_MSG_655) << "DDL log replay : FREE tablespace " << space_id
                              << " is missing.";
    }
    return;
  }
```

**线程隔离**：`thread_id` 列让每个线程只访问/删除**自己的**记录，因此无需行锁（log0ddl.h 注释）。

#### 存储层：`innodb_ddl_log` 表结构与读写本体

**这张表本身是 hidden 的 DD 表**（`ha_innodb.cc:13025`），只有两个索引：

```cpp
innodb_ddl_log->set_hidden(true);
def->add_field(0, "id", "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT");
def->add_field(1, "thread_id", "thread_id BIGINT UNSIGNED NOT NULL");
def->add_field(2, "type", "type INT UNSIGNED NOT NULL");
def->add_field(3, "space_id", "space_id INT UNSIGNED");
def->add_field(4, "page_no",  "page_no INT UNSIGNED");
def->add_field(5, "index_id", "index_id BIGINT UNSIGNED");
def->add_field(6, "table_id", "table_id BIGINT UNSIGNED");
def->add_field(7, "old_file_path", "old_file_path VARCHAR(512) ...");
def->add_field(8, "new_file_path", "new_file_path VARCHAR(512) ...");
def->add_index(0, "index_pk", "PRIMARY KEY(id)");
def->add_index(1, "index_k_thread_id", "KEY(thread_id)");
```

★ **只有 `PK(id)` 与 `KEY(thread_id)` 两个索引**——这直接决定了回放的检索方式只有两种：按 id 扫全表、或按 thread_id 认领（见后文 `replay_all` vs `replay_by_thread_id`）。它是 hidden DD 表，**不进 binlog、不参与普通 SQL**。

**★ 写入不走 SQL**——这是最反直觉的一点。不存在 `Log_DDL::insert()`，实际是 `DDL_Log_Table::insert()`（`log0ddl.cc:462`），它**手工构造 tuple 直接调 B-tree 插入原语**：

```cpp
dberr_t DDL_Log_Table::insert(const DDL_Record &record) {
  dict_index_t *index = m_table->first_index();
  uint32_t flags = BTR_NO_LOCKING_FLAG;
  ...
  create_tuple(record);
  entry = row_build_index_entry(m_tuple, nullptr, index, m_heap);
  error = row_ins_clust_index_entry_low(flags, BTR_MODIFY_LEAF, index,
                                        index->n_uniq, entry, m_thr, false);
  if (error == DB_FAIL) { error = ...(BTR_MODIFY_TREE)...; }
  index = index->next();                                  // 二级索引（thread_id）
  error = row_ins_sec_index_entry_low(flags, BTR_MODIFY_LEAF, index, ...);
```

三点：

1. **绕过解析器、锁、外键、duplicate check**（`BTR_NO_LOCKING_FLAG`），聚簇 + 二级索引各写一次。DDL log 的写入必须尽量轻，因为它嵌在 DDL 事务里高频发生。
2. `create_tuple()`（`:346`）**手工填系统列**：`buf = 0xFF × 8` 给 `DB_ROW_ID`/`DB_ROLL_PTR`，`mach_write_to_6(buf, m_trx->id)` 给 `DB_TRX_ID`——又一次手工构造记录。
3. 事务由构造参数决定：`DDL_Log_Table(trx)` 内部 `start_query_thread()`（`:333`）建一个 dummy query graph（`pars_complete_graph_for_exec`）；**无 trx 时 `m_thr == nullptr`，不能 insert**。

**★ 删除是打 delete-mark，不是物理删除**——这就是"删 log 也要事务"的根因：

```cpp
// DDL_Log_Table::remove(ulint id)，log0ddl.cc:730
pcur.open(clust_index, 0, m_tuple, PAGE_CUR_LE,
          BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE, &mtr, ...);
if (!rec_get_deleted_flag(...)) {
  err = btr_cur_del_mark_set_clust_rec(flags, ..., m_thr, m_tuple, &mtr);   // ★ 打标记
}
...
if (search_result == ROW_NOT_FOUND) { ut_d(ut_error); ut_o(return DB_CORRUPTION); }
err = btr_cur_del_mark_set_sec_rec(flags, btr_cur, true, m_thr, &mtr);
```

打标记要写 undo ⇒ **必须持有 trx 与 `m_thr`**。二级索引找不到记录直接判 `DB_CORRUPTION`（因为两个索引必须同在）。

配套两条：

- `remove(const DDL_Records&)`（`:813`）逐条判 `if (record->get_deletable())` —— ★ **`m_deletable` 为 false 的记录被故意保留**（这正是 alter-encrypt 那条"跨重启续做"记录不被清掉的原因）。
- `Log_DDL::delete_by_ids()`（`:1539`）自己开 background trx（`trx_allocate_for_background` + `ddl_operation = true` + `trx_commit_for_mysql`），★ 遇 `DB_TOO_MANY_CONCURRENT_TRXS` **最多重试 10 次**（`DELETE_IDS_RETRIES_MAX = 10`，`:1537`）。而 `delete_by_id()`（`:1442`）若调用时持着 dict 锁也要先 `dict_sys_mutex_exit()`，且靠 `DDL_Log_Table` 局部对象的析构 `stop_query_thread()`——**必须在 trx commit 之前析构**。

**★ `next_id()` 借用了表自身的 AUTO_INCREMENT 计数器**（`log0ddl.cc:835`）：

```cpp
inline uint64_t Log_DDL::next_id() {
  dict_table_autoinc_lock(dict_sys->ddl_log);
  autoinc = dict_table_autoinc_read(dict_sys->ddl_log);
  ++autoinc;
  dict_table_autoinc_update_if_greater(dict_sys->ddl_log, autoinc);
  dict_table_autoinc_unlock(dict_sys->ddl_log);
  return (autoinc);
}
```

⇒ id 不是另设的序列，而是 **`innodb_ddl_log` 这张表的 autoinc**（内存计数器 + autoinc mutex），`update_if_greater` 会经 `dict_table_autoinc_log()` 持久化到 `mysql.innodb_dynamic_metadata`。重启时由 `srv0start.cc:2356` 的 `metadata_applier` 恢复，并对未被 `ha_innobase::open()` 初始化过的表补 `++table->autoinc`。

**★ `skip()`：四种情况完全不写 DDL log**（`log0ddl.cc:847`）

```cpp
inline bool Log_DDL::skip(const dict_table_t *table, THD *thd) {
  return (recv_recovery_on || thread_local_ddl_log_replay ||
          (table != nullptr && table->is_temporary()) ||
          thd_is_bootstrap_thread(thd));
}
```

| 条件 | 含义 |
|---|---|
| `recv_recovery_on` | 正在做 redo 恢复——此时"回放"本身就是恢复动作，不能再产生新 log |
| `thread_local_ddl_log_replay` | 正在 replay DDL log（回放动作本身不记 log，否则无限递归） |
| `table->is_temporary()` | 临时表不需要崩溃恢复 |
| `thd_is_bootstrap_thread` | 初始化建系统表时不记 |

#### 记录类型

**★ `Log_Type` 实际只有 7 种**（`log0ddl.h:43`）——`SMALLEST_LOG` 与 `BIGGEST_LOG` 是同值哨兵，不是独立类型：

```cpp
enum class Log_Type : uint32_t {
  /** Smallest log type */
  SMALLEST_LOG = 1,

  /** Drop an index tree */
  FREE_TREE_LOG = 1,

  /** Delete a file */
  DELETE_SPACE_LOG,

  /** Rename a file */
  RENAME_SPACE_LOG,

  /** Drop the entry in innodb_table_metadata */
  DROP_LOG,

  /** Rename table in dict cache. */
  RENAME_TABLE_LOG,

  /** Remove a table from dict cache */
  REMOVE_CACHE_LOG,

  /** Alter Encrypt a tablespace */
  ALTER_ENCRYPT_TABLESPACE_LOG,

  /** Alter Unencrypt a tablespace */
  ALTER_UNENCRYPT_TABLESPACE_LOG,

  /** Biggest log type */
  BIGGEST_LOG = ALTER_UNENCRYPT_TABLESPACE_LOG
};
```

> ⚠️ `SMALLEST_LOG = 1` 与 `FREE_TREE_LOG` **同值**；`BIGGEST_LOG` 就是 `ALTER_UNENCRYPT_TABLESPACE_LOG = 7`。底层 `uint32_t` 是因为表里 `type` 列是 `INT UNSIGNED`（4 字节）。

**⚠️ `write_*_log()` 只有 7 个**——**`write_alter_unencrypt_space_log()` 不存在**（全仓 0 命中）。加密/解密共用 `write_alter_encrypt_space_log(space_id, Encryption::Progress type, existing_rec)`，在 `insert_alter_encrypt_space_log()` 里按 `type` 分派：

```cpp
  if (type == Encryption::Progress::ENCRYPTION) {
    record.set_type(Log_Type::ALTER_ENCRYPT_TABLESPACE_LOG);
  } else {
    ut_ad(type == Encryption::Progress::DECRYPTION);
    record.set_type(Log_Type::ALTER_UNENCRYPT_TABLESPACE_LOG);
  }
```

**★ `DDL_Record` 的 `m_deletable` 是唯一的"控制位"**（默认 `true`，构造 `log0ddl.cc:159`）：

```cpp
class DDL_Record {
 public:
  ...
  void set_deletable(bool deletable) { m_deletable = deletable; }
  bool get_deletable() const { return (m_deletable); }
  /** Get encryption operation type */
  Encryption::Progress get_encryption_type() const {
    auto type = get_type();
    if (type == Log_Type::ALTER_ENCRYPT_TABLESPACE_LOG) {
      return Encryption::Progress::ENCRYPTION;
    }
    ut_ad(type == Log_Type::ALTER_UNENCRYPT_TABLESPACE_LOG);
    return Encryption::Progress::DECRYPTION;
  }
  std::ostream &print(std::ostream &out) const;      // ⚠️ 不存在 to_string()
 private:
  ulint m_id;
  Log_Type m_type;
  ulint m_thread_id;
  space_id_t m_space_id;
  page_no_t m_page_no;
  ulint m_index_id;
  table_id_t m_table_id;
  char *m_old_file_path;      /* DELETE 的待删路径 / RENAME 的旧路径 */
  char *m_new_file_path;      /* RENAME 的新路径；★ REMOVE_CACHE 存的是表名 */
  mem_heap_t *m_heap;         /* ★ 惰性创建：只有设了路径才分配 */
  bool m_deletable;
};
```

> ★ 未设置的字段保持哨兵值 `ULINT_UNDEFINED` / `SPACE_UNKNOWN` / `FIL_NULL`——`create_tuple()` 和 `print()` 都靠这些哨兵判断"该列要不要写/要不要打印"。**留空 = 该列写 SQL NULL**，这就是"一条记录只填自己关心的列"的实现方式。
> ⚠️ **不对称**：`REMOVE_CACHE` 把表名放在 **`new_file_path`** 列（不是 `old_file_path`），`replay_remove_cache_log` 也从 `get_new_file_path()` 取。

`Log_Type` 枚举（log0ddl.h，uint32_t 节省表空间）：

| 类型 | 值 | 含义 | 对应 replay |
|------|----|------|-------------|
| `FREE_TREE_LOG` | 1 | 释放索引 B-tree | `replay_free_tree_log`（`btr_free_if_exists`） |
| `DELETE_SPACE_LOG` | 2 | 删除表空间文件 | `replay_delete_space_log`（`fil_delete_tablespace`） |
| `RENAME_SPACE_LOG` | 3 | 重命名表空间文件 | `replay_rename_space_log` |
| `DROP_LOG` | 4 | 删除持久化的动态元数据条目 | `replay_drop_log` |
| `RENAME_TABLE_LOG` | 5 | dict cache 中重命名表 | `replay_rename_table_log` |
| `REMOVE_CACHE_LOG` | 6 | 从 dict cache 移除表 | `replay_remove_cache_log` |
| `ALTER_ENCRYPT_TABLESPACE_LOG` | 7 | 表空间加密 | `replay_alter_encrypt_space_log` |
| `ALTER_UNENCRYPT_TABLESPACE_LOG` | 8 | 表空间解密 | 同上 |

> `DROP_LOG` 清理的是表的**动态元数据**（autoinc、update_time 等"非事务持久"信息）。源码注释里写 `innodb_table_metadata`、官方 WL 与文档写 `innodb_dynamic_metadata`——同一处的两种叫法，见 Misc。

#### 写入本体：`write_*` / `insert_*` 两层结构

上表是"有哪些记录类型"，但**写一条 log 到底做了什么**要往下看一层。所有 `write_*` 都遵循同一套骨架，以 `write_drop_log` 为例（`log0ddl.cc`）：

```cpp
dberr_t Log_DDL::write_drop_log(trx_t *trx, const table_id_t table_id) {
  if (skip(nullptr, trx->mysql_thd)) {        // ① 可跳过（见下）
    return (DB_SUCCESS);
  }

  trx->ddl_operation = true;                  // ② ★ 标记本事务是 DDL 事务

  uint64_t id = next_id();                    // ③ 单调递增的记录 id
  ulint thread_id = thd_get_thread_id(trx->mysql_thd);   // ④ 归属线程（post_ddl 按它认领）

  DBUG_INJECT_CRASH("ddl_log_crash_before_drop_log", crash_before_drop_log_counter++);

  dberr_t err;
  err = insert_drop_log(trx, id, thread_id, table_id);   // ⑤ 真正的落盘
  if (err != DB_SUCCESS) return err;

  DBUG_INJECT_CRASH("ddl_log_crash_after_drop_log", crash_after_drop_log_counter++);
  return err;
}
```

第二层 `insert_*` 才是落盘，且**每一条都要先放掉 dict mutex**：

```cpp
dberr_t Log_DDL::insert_drop_log(trx_t *trx, uint64_t id, ulint thread_id,
                                 const table_id_t table_id) {
  ut_ad(trx->ddl_operation);
  ut_ad(dict_sys_mutex_own());                // ★ 进入时持有
  trx_start_if_not_started(trx, true, UT_LOCATION_HERE);
  dict_sys_mutex_exit();                      // ★ ★ 立刻释放：insert 是 IO，不能持全局字典锁
  ...
  DDL_Record record;
  record.set_id(id);
  record.set_thread_id(thread_id);
  record.set_type(Log_Type::DROP_LOG);
  record.set_table_id(table_id);
  ...
}
```

⇒ 于是 `write_*`（构造参数、标记事务、崩溃注入）与 `insert_*`（落盘，且**内部统一释放 `dict_sys` mutex**）分工明确。这也解释了调用方为什么可以放心在持 dict 锁的路径上写 log——**释放动作被封装在 `insert_*` 里了**。

**★ `is_drop` 参数：同一个函数两种相反语义**

`write_delete_space_log` 是最能说明"DDL log 是回滚预案"的例子——它有个 `bool is_drop` 参数，两条分支含义完全相反：

```cpp
  if (is_drop) {
    /* DROP：写一条"删这个表空间"的预案，留着等崩溃恢复/post_ddl 真正删 */
    err = insert_delete_space_log(trx, id, thread_id, space_id, file_path, dict_locked);
  } else {
    /* CREATE：先写预案（trx=nullptr！），紧接着立刻把它删掉 */
    err = insert_delete_space_log(nullptr, id, thread_id, space_id, file_path, dict_locked);
    ...
    err = delete_by_id(trx, id, dict_locked);          // ★ 写完马上删
    ut_ad(err == DB_SUCCESS || err == DB_TOO_MANY_CONCURRENT_TRXS);
  }
```

| | `is_drop = true`（DROP 场景） | `is_drop = false`（CREATE 场景） |
|---|---|---|
| 插入时传的 trx | 当前事务 `trx` | **`nullptr`** |
| 之后 | 保留，等回放/post_ddl | **立刻 `delete_by_id` 删掉** |
| 语义 | "万一崩了，请把这个空间删掉" | "提交成功 ⇒ 预案作废，删掉它" |

★ **这就是"DDL log 是回滚预案而非执行计划"最直接的源码证据**：CREATE 一条 `.ibd` 时，先登记一条"删除它"的预案（`trx=nullptr` 表示这条记录**不属于当前事务**、独立提交，所以即便当前事务回滚它也在）；等事务**提交成功**，立刻把预案删掉。于是：

- 提交成功 ⇒ 预案被删 ⇒ log 表干净，什么都不用做；
- 崩溃/失败 ⇒ 预案还在 ⇒ recovery 重放它，把刚建的 `.ibd` 删掉。

（这也是为什么 `delete_by_id` 的返回值允许 `DB_TOO_MANY_CONCURRENT_TRXS`——删预案只是清理，并发限制导致的失败可以容忍。）

另外三条通用约定：`skip()` 使整条 log 可跳过（如 `innodb_print_ddl_logs` 未开、或临时表路径）；`trx->ddl_operation = true` 是后续 2PC / redo 策略的开关；**每个 `write_*` 前后都成对埋了 `DBUG_INJECT_CRASH` 注入点**（`*_before_*` / `*_after_*`），这是验证"写 log 与执行动作之间任意时刻崩溃都能恢复"的手段。

#### 生命周期与两个方向

**方向 A（CREATE 类）——写后即删，为回滚**：

执行期由子事务 INSERT 一条记录，随即在主事务里 DELETE 掉它（源码注释：*"Delete this operation if the create trx is committed"*）：

```cpp
err = insert_free_tree_log(nullptr, index, id, thread_id);
...
/* Delete this operation if the create trx is committed */
err = delete_by_id(trx, id, false);
```

- 主事务**提交** ⇒ 那条 DELETE 生效 ⇒ log 表里没有记录 ⇒ post-DDL 无事可做
- 主事务**回滚/崩溃** ⇒ DELETE 不生效 ⇒ 记录仍在 ⇒ replay 执行清理（删刚建的 .ibd、free 刚建的树）

**方向 B（DROP 类）——提交时写，为前滚**：

提交阶段写入记录，成功后由 `Log_DDL::post_ddl(thd)` 按 `thread_id` 找出记录并 `replay` 真正执行，完成后 `delete_by_ids` 删除已处理记录。函数本体很短，但每一行都有讲究：

```cpp
dberr_t Log_DDL::post_ddl(THD *thd) {
  if (skip(nullptr, thd)) {
    return (DB_SUCCESS);
  }

  // ★ 只读模式 / force_recovery 到一定级别 → 直接跳过（见 Misc 的坑）
  if (srv_read_only_mode || srv_force_recovery >= SRV_FORCE_NO_UNDO_LOG_SCAN) {
    return (DB_SUCCESS);
  }

  ulint thread_id = thd_get_thread_id(thd);

  if (srv_print_ddl_logs) {
    ib::info(ER_IB_MSG_660) << "DDL log post ddl : begin for thread id : " << thread_id;
  }

  thread_local_ddl_log_replay = true;          // ★ 标记"本线程正在 replay"

  dberr_t err = replay_by_thread_id(thread_id);

  thread_local_ddl_log_replay = false;

  if (srv_print_ddl_logs) {
    ib::info(ER_IB_MSG_661) << "DDL log post ddl : end for thread id : " << thread_id;
  }

  return (err);
}
```

★ 三个细节：① `srv_force_recovery >= SRV_FORCE_NO_UNDO_LOG_SCAN` 时**直接返回**——强制启动模式下 DDL log 不被 replay，会留下孤儿文件（Misc 已列此坑）；② `thread_local_ddl_log_replay` 是**线程局部标志**，置位期间本线程再写 DDL log 会被区别对待（避免 replay 自身产生新记录）；③ 那两条 `ib::info` 正是官方示例里的 `DDL log post ddl : begin/end for thread id : 7`。

崩溃恢复走 `Log_DDL::recover()`，与 post_ddl 同构但换用 `replay_all()`（不再按线程区分）：

```cpp
dberr_t Log_DDL::recover() {
  if (srv_read_only_mode || srv_force_recovery > 0) {   // ★ 门槛更严：> 0 就跳过
    return (DB_SUCCESS);
  }

  ib::info(ER_IB_MSG_662) << "DDL log recovery : begin";

  thread_local_ddl_log_replay = true;
  s_in_recovery = true;                                  // ★ 全局"恢复中"标志

  dberr_t err = replay_all();

  thread_local_ddl_log_replay = false;
  s_in_recovery = false;

  ib::info(ER_IB_MSG_663) << "DDL log recovery : end";
```

#### 边界场景：崩溃恢复中延迟执行的操作

加密/解密表空间的 DDL log 是典型——`replay_alter_encrypt_space_log` 明确只在恢复期调用：

```cpp
dberr_t Log_DDL::replay_alter_encrypt_space_log(DDL_Record &record) {
  /* Normal operation, we shouldn't come here during post_ddl */
  ut_ad(is_in_recovery());

  error = replace_and_insert(&record);           // ★ 换一条记录，thread_id 改 ULINT_MAX
  if (error != DB_SUCCESS) return error;
  ut_ad(record.get_thread_id() == ULINT_MAX);

  /* We could have resume encryption execution one by one for each tablespace
  from here by calling SQL API to run the query. But then it would be blocking
  server bootstrap. We need to resume this encryption in BG thread so we need
  to just make a note of this space and operation here and don't do any real
  operation. */
  ts_encrypt_ddl_records.push_back(&record);    // 记入待恢复列表

  /* Make sure not to delete this record till resume operation finishes.
  This is to make sure that if there is a crash before that, we can resume
  encryption in the next restart. */
  record.set_deletable(false);                  // ★ 不可删除，保证下次还能恢复
  return error;
}

// 换记录：后台事务删旧的、插新的，并把 thread_id 置为 ULINT_MAX（无人认领）
static dberr_t replace_and_insert(DDL_Record *record) {
  trx_t *trx = trx_allocate_for_background();
  trx_start_internal(trx, UT_LOCATION_HERE);
  trx->ddl_operation = true;

  /* update the thread_id for the record */
  record->set_thread_id(ULINT_MAX);              // ★ 脱离原前台线程

  {
    DDL_Log_Table ddl_log(trx);
    /* Remove old record and insert the new record */
    err = ddl_log.remove(record->get_id());
    ...
    err = ddl_log.insert(*record);
  }
  trx_commit_for_mysql(trx);
  trx_free_for_background(trx);
  return err;
}
```

★ 这是一套完整的"**延迟恢复**"设计，两处配合缺一不可：

1. `thread_id = ULINT_MAX` —— 这条记录不再属于任何前台线程，因此不会被某个 DDL 的 `post_ddl` 按 thread_id 取走、也不会被它删掉；
2. `set_deletable(false)` —— 保证它在后台线程完成前不会被清理，**若后台还没做完就再次崩溃，下次启动仍能接着做**。

注释也点明了动机：不能在这里直接跑加密恢复，那会**阻塞 server bootstrap**，所以只记一笔、交给后台线程。这是"DDL log 不只用于即时回放，也用于跨重启续做"的例证。

**post_ddl 在 commit 和 rollback 之后都会被调用**（log0ddl.h 注释：*"Replay and clean DDL logs after DDL transaction commits or rollbacks"*）：rollback 时方向 A 的记录因 DELETE 未生效而保留，replay 执行清理；方向 B 根本没有记录（提交没发生），replay 是空操作。调用者是 server 层 `hton->post_ddl(thd)`（如 TRUNCATE 走 `Sql_cmd_truncate_table::cleanup_base`）→ `innobase_post_ddl`。

#### replay 的幂等性

设计约束由 WL#9536 提出（*"All operations will be implemented in an idempotent way"*），8.0.39 的落地在 `btr_free_if_exists` 的两个辅助函数上。

**校验侧**（`btr_free_root_check`）——回放前先确认这页确实是那棵树的根页：

```cpp
[[nodiscard]] static buf_block_t *btr_free_root_check(
    const page_id_t &page_id, const page_size_t &page_size,
    space_index_t index_id, mtr_t *mtr) {
  ut_ad(index_id != BTR_FREED_INDEX_ID);

  buf_block_t *block =
      buf_page_get(page_id, page_size, RW_X_LATCH, UT_LOCATION_HERE, mtr);

  if (fil_page_index_page_check(block->frame) &&
      index_id == btr_page_get_index_id(block->frame)) {
    /* This should be a root page.
    It should not be possible to reassign the same
    index_id for some other index in the tablespace. */
    ut_ad(page_is_root(block->frame));
  } else {
    block = nullptr;      // ★ 不是索引页 / index_id 不匹配 → 直接跳过，不做任何事
  }

  return (block);
}
```

**标记侧**（`btr_free_root_invalidate`）——释放成功后把根页的 `PAGE_INDEX_ID` 抹掉：

```cpp
static void btr_free_root_invalidate(buf_block_t *block, mtr_t *mtr) {
  ut_ad(page_is_root(block->frame));

  btr_page_set_index_id(buf_block_get_frame(block),
                        buf_block_get_page_zip(block), BTR_FREED_INDEX_ID, mtr);
}
```

★ 两者配合就是幂等的全部：第一次回放——校验通过 → 释放 → 打上 `BTR_FREED_INDEX_ID`；第二次回放——`index_id == btr_page_get_index_id(...)` 不再成立（开头那条 `ut_ad(index_id != BTR_FREED_INDEX_ID)` 也保证不会拿已释放的 id 来比对）→ 直接跳过。**不需要额外的"已处理"记录表，幂等性直接编码在页面内容里**。

> WL §0.1 的伪代码（`IF PAGE_TYPE=FIL_PAGE_INDEX AND PAGE_LEVEL=0 AND PAGE_INDEX_ID=index_id THEN ...`）语义与此一致，但那不是 8.0.39 的真实实现——真实实现如上。

`DELETE_SPACE_LOG` 同理：文件不存在就直接删除 log 记录，不报错。

### 崩溃恢复主链路：`recover()` / `replay_all()` / `replay()`

**调用时机**：`ha_post_recover()` → `innobase_post_recover()` → `Log_DDL::recover()`。在 server 启动序列里，它位于 **redo/binlog 恢复之后、XA 恢复之前**（`mysqld.cc:6893`）。

#### ★ 双重门槛：一处容易读错的空转

```cpp
// ha_innodb.cc:4130
static void innobase_post_recover() {
  if (srv_force_recovery < SRV_FORCE_NO_TRX_UNDO) {     // ① 外层：< 3 才进来
    dberr_t err = log_ddl->recover();
    ...

// log0ddl.cc:1943
dberr_t Log_DDL::recover() {
  if (srv_read_only_mode || srv_force_recovery > 0)     // ② 内层：> 0 就直接返回！
    return (DB_SUCCESS);
  thread_local_ddl_log_replay = true;
  s_in_recovery = true;
  dberr_t err = replay_all();
```

★ 外层放 `force_recovery < 3` 的记录进来，内层却又要求 `force_recovery == 0`。⇒ **`force_recovery = 1` 或 `2` 时，`post_recover` 进来了但 DDL 恢复整个空转**，崩溃遗留的 DDL log 记录**永久留在表里**（不回放、也不清理）。这是"为什么开着强制恢复启动后，表里有陈年 `innodb_ddl_log` 记录"的原因。

#### ★ 扫描顺序：降序 = undo 语义

```cpp
// DDL_Log_Table::search_all()，log0ddl.cc:627
  /** Scan the index in decreasing order. */
  pcur.open_at_side(false, index, BTR_SEARCH_LEAF, true, 0, &mtr);
  for (; move == true; move = pcur.move_to_prev(&mtr)) {
```

★ **逆序回放**（从最大 id 往回走）。这与事务回滚的 LIFO 语义一致：后登记的预案先撤。运行期的 `post_ddl` 走 `replay_by_thread_id()`（按 thread_id 精确认领本线程的记录），同样是 `rbegin() → rend()` 逆序。

两条检索方式的分工：

| | 检索 | 用途 |
|---|---|---|
| `recover()` → `replay_all()` | `search_all()`，**不看 thread_id** | 崩溃恢复：所有残留记录都归它处理 |
| `post_ddl()` → `replay_by_thread_id()` | `search(thread_id)`，只认领本线程 | 正常运行：DDL 提交后回放自己那批 |

⇒ 这也解释了 `replace_and_insert` 里把 `thread_id` 改成 `ULINT_MAX` 的用意：**这个 id 永远匹配不到任何活线程**，所以运行期 `post_ddl` 不会认领它，只有下一次 `replay_all()`（重启）会捡起它——配合 `set_deletable(false)` 实现"跨重启续做"。

#### 各记录类型的实际动作

`replay()`（`log0ddl.cc:1576`）按 `record.get_type()` 分派。★ 注意**六个分支返回 `void`（无错误码）**，只有 `ALTER_ENCRYPT` 返回 `err`；`default: ut_error`。

| 类型 | 分支 | 实际动作 |
|---|---|---|
| `FREE_TREE_LOG` | `:1625` | `fil_space_get_page_size` 拿不到就 return（.ibd 已删则跳过）；否则持 dict mutex + mtr 调 `btr_free_if_exists` |
| `DELETE_SPACE_LOG` | `:1660` | undo 表空间走 `undo::ddl_mutex` + `ut_a(undo_space->is_empty())`；有 THD 时 `dict_sdi_remove_from_cache`；`fil_update_partition_name` 修正分区大小写后 `row_drop_tablespace` |
| `RENAME_SPACE_LOG` | `:1723` | `fil_op_replay_rename_for_ddl` |
| `DROP_LOG` | `:1805` | **只删** `DDTableBuffer` 里的动态元数据，不碰数据文件 |
| `RENAME_TABLE_LOG` | `:1814` | ★ `is_in_recovery()` 时**直接 return**（崩溃恢复里啥也不做） |
| `REMOVE_CACHE_LOG` | `:1874` | ★ 同上——只清内存 cache，重启后 cache 本就是空的 |

★ 两条 `is_in_recovery()` 直接 return 很值得记：**不是所有 DDL log 在崩溃恢复里都有意义**——纯内存态的动作（改 cache、改表名映射）重启后已自然失效，重做无意义甚至有害。

```cpp
// log0ddl.cc:1814 —— RENAME_TABLE：恢复期直接跳过
void Log_DDL::replay_rename_table_log(const char *old_name,
                                      const char *new_name) {
  if (is_in_recovery()) {
    if (srv_print_ddl_logs) {
      ib::info(ER_IB_MSG_657) << "DDL log replay : in recovery,"
                              << " skip RENAME TABLE";
    }
    return;
  }
  ...
  err = row_rename_table_for_mysql(old_name, new_name, nullptr, trx, true);
```

```cpp
// log0ddl.cc:1874 —— REMOVE_CACHE：恢复期直接跳过
void Log_DDL::replay_remove_cache_log(table_id_t table_id,
                                      const char *table_name) {
  if (is_in_recovery()) {
    if (srv_print_ddl_logs) {
      ib::info(ER_IB_MSG_659) << "DDL log replay : in recovery,"
                              << " skip REMOVE CACHE";
    }
    return;
  }
  dict_table_t *table = dd_table_open_on_id_in_mem(table_id, false);
  if (table != nullptr) {
    ut_ad(strcmp(table->name.m_name, table_name) == 0);
    dict_sys_mutex_enter();
    dd_table_close(table, nullptr, nullptr, true);
    btr_drop_ahi_for_table(table);          // ★ 先摘 AHI，再摘缓存
    dict_table_remove_from_cache(table);
    dict_sys_mutex_exit();
  }
}
```

**★ `replay_drop_log` 只删动态元数据，不碰数据文件**：

```cpp
void Log_DDL::replay_drop_log(const table_id_t table_id) {
  mutex_enter(&dict_persist->mutex);
  ut_d(dberr_t error =) dict_persist->table_buffer->remove(table_id);
  ut_ad(error == DB_SUCCESS);
  mutex_exit(&dict_persist->mutex);
  DBUG_INJECT_CRASH("ddl_log_crash_after_replay", crash_after_replay_counter++);
}
```

**★ `replay_delete_space_log` 的 undo 表空间特判**（`log0ddl.cc:1660`）：

```cpp
void Log_DDL::replay_delete_space_log(space_id_t space_id,
                                      const char *file_path) {
  if (fsp_is_undo_tablespace(space_id)) {
    /* Serialize this delete with all undo tablespace DDLs. */
    mutex_enter(&undo::ddl_mutex);
    ...
    undo::Tablespace *undo_space = undo::spaces->find(space_num);
    if (undo_space != nullptr) {
      ut_a(undo_space->is_empty());        // ★ 硬断言：恢复期删 undo 空间必须已空
      undo::spaces->drop(undo_space);
    }
    ...
  }
  /* Update filename with correct partition case, of needed. */
  std::string path_str(file_path);
  std::string space_name;
  fil_update_partition_name(space_id, 0, false, space_name, path_str);
  file_path = path_str.c_str();
  row_drop_tablespace(space_id, file_path);
```

**⚠️ `fil_op_replay_rename_for_ddl()` 的返回值语义反转**（`fil0fil.cc:9844`）——**返回 `true` 表示成功**：

```cpp
bool fil_op_replay_rename_for_ddl(const page_id_t &page_id,
                                  const char *old_name, const char *new_name) {
  fil_space_t *space = fil_space_get(space_id);
  if (space == nullptr && !fil_system->open_for_recovery(space_id)) {
    ib::info(ER_IB_MSG_350)
        << "Can not find space with space ID " << space_id
        << " when replaying the DDL log "
        << "rename from '" << old_name << "' to '" << new_name << "'";
    return true;      // ★★ 找不到 = "目标状态已达成或无从失败" → 返回 true（成功）
  }
  return fil_op_replay_rename(page_id, old_name, new_name);
}
```

调用方印证（`log0ddl.cc:1741`）：`if (!ret && srv_print_ddl_logs)` —— 只有 `false` 才打印失败。

★ `RENAME_SPACE_LOG` 还有一处**语义反转**：`fil0fil.cc:9849` 找不到 space 时 `return true`（表示"无需做、当作成功"），但调用方写的是 `if (!ret && srv_print_ddl_logs)`——把 `false` 当失败来打印。读这段代码时别被返回值方向绕进去。

#### 幂等性：为什么重复回放是安全的

1. `btr_free_if_exists`（`btr0btr.cc:1010`）：`btr_free_root_check` 返回 nullptr 即 return——**已释放的树再放一次是 no-op**。
2. `DDTableBuffer::remove`（`dict0dict.cc:5403`）：`pcur.get_low_match() != n_uniq` 就跳过，**恒返回 `DB_SUCCESS`**，所以 `replay` 里那条 `ut_ad(error == DB_SUCCESS)` 才成立。
3. `row_drop_tablespace`（`row0mysql.cc:3752`）：`err != DB_SUCCESS && err != DB_TABLESPACE_NOT_FOUND` 才报错——**容忍"已经删过了"**。

★ 幂等性是**必须的**而非优化：每条 replay 之后都埋了 `DBUG_INJECT_CRASH("ddl_log_crash_after_replay", ...)`（1657/1720/1748/1801/1811），这正是"**回放完了、但还没来得及删记录就崩**"的注入点——下次启动必须重放一遍。

#### 清理与失败处理

`replay_all`（`:1485`）：任一条 `err != DB_SUCCESS` **立即 break**，**跳过 `delete_by_ids`** ⇒ 记录留在表里，留给下次启动重放；全部成功才 `delete_by_ids(records)`（含 10 次重试）。★ 注意 `replay_by_thread_id`（`:1513`）**忽略 replay 返回值**，只负责收集后统一删记录——所以运行期单条回放失败不会中断，但记录仍会被清掉（这是运行期与恢复期的一个行为差异）。

### 失败与崩溃恢复路径

统一的"三件套"，在 `mysql_create_table()` / `mysql_inplace_alter_table()` / `mysql_alter_table()` 三处**逐字重复**：

```cpp
trans_rollback_stmt(thd);
/*
  Full rollback in case we have THD::transaction_rollback_request
  and to synchronize DD state in cache and on disk (as statement
  rollback doesn't clear DD cache of modified uncommitted objects).
*/
trans_rollback(thd);
```

外加：

- **post-DDL 钩子负责清理引擎残留文件**（CREATE 场景原注释：*"In case of CREATE TABLE post-DDL hook is mostly relevant for case when statement is rolled back. In such cases it is responsibility of this hook to cleanup files which might be left after failed table creation attempt."*）
- **非原子引擎要手工补做 DD 删除**（因为它的 DD 已经提交了）
- **INPLACE 失败**：先让引擎回滚 `ha_commit_inplace_alter_table(..., false, ...)`，再 `trans_rollback`，再 `dict_cache_reset`，再 `reopen_tables()`
- **COPY 失败且非原子**：`quick_rm_table()` 删临时表 + **重试 20 次**把备份表改回原名
- **`trans_intermediate_ddl_commit()` 本身就是"带回滚的提交"**：

```cpp
static bool trans_intermediate_ddl_commit(THD *thd, bool error) {
  Implicit_substatement_state_guard substatement_guard(thd);
  if (error) {
    trans_rollback_stmt(thd);
    trans_rollback(thd);
    return true;
  }
  return trans_commit_stmt(thd) || trans_commit(thd);
}
```

**崩溃恢复**：`Log_DDL::recover`（log0ddl.cc）调用 `replay_all()` 重放所有残留记录：

- DD 事务已提交（redo 与 binlog 里有 commit）⇒ 视为成功 ⇒ replay **前滚**完成物理操作
- DD 事务未提交 ⇒ InnoDB 重放 DD 的 redo 时把该事务回滚 ⇒ 方向 A 的 log 记录保留 ⇒ replay **清理**残留对象 ⇒ 表恢复原状

**原子性闭环**：物理操作通过 DDL log 延迟到"确定已提交"之后（或为回滚而预留），元数据修改通过 undo 可回滚——两者结合保证"全成功或全回滚"。关键判据是 **DDL log 记录的存在性由 undo 决定**：未提交 ⇒ 记录还在（DELETE 未生效）；已提交 ⇒ 记录已消失（DELETE 生效，对 CREATE 方向）或需要前滚（对 DROP 方向）。

---

### `post_ddl`：为什么"不可逆动作"要等到提交之后

#### handler 层的语义契约

```cpp
// sql/handler.h:2269
/**
  Perform post-commit/rollback cleanup after DDL statement (e.g. in
  case of DROP TABLES really remove table files from disk).

  @note This hook will be invoked after DDL commit or rollback only
        for storage engines supporting atomic DDL.

  @note Problems during execution of this method should be reported to
        error log and as warnings/notes to user. Since this method is
        called after successful commit of the statement we can't fail
        statement with error.
*/
typedef void (*post_ddl_t)(THD *thd);
```

★ 三条契约：① 只在 **commit 或 rollback 之后**调用；② 只对支持原子 DDL 的引擎；③ **不能让语句失败**——已经提交完了，只能写 error log / warning。⇒ `post_ddl` 是**尽力而为的清理**，不是事务的一部分。

配套的顺序契约（`handler.h:6119`，ALTER 场景）：

```
*) Update the SQL-layer data-dictionary ...
*) Process the RENAME clause by calling handler::ha_rename_table() ...
*) Commit the statement/transaction.
*) Finalize atomic DDL operation by calling handlerton::post_ddl() hook ...
*) Additionally inform the storage engine ... handler::ha_notify_table_changed()
```

⇒ **post_ddl 严格在 commit 之后、notify 之前**。这个顺序就是"原子 DDL"的骨架：DD 变更与 binlog 在事务内提交，不可逆的物理动作（删文件、重命名）推到事务外。

#### InnoDB 的实现：只做一件事

```cpp
// ha_innodb.cc:5100（注意：不在 handler0alter.cc 里）
static void innobase_post_ddl(THD *thd) {
  /* During upgrade, etc., the log_ddl may haven't been
  initialized and there is nothing to do now. */
  if (log_ddl != nullptr) {
    dberr_t err = log_ddl->post_ddl(thd);
    /* If this fails, do not continue startup. */
    if (err != DB_SUCCESS) {
      ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_DDL_LOG_FAIL_POST_DDL);
    }
  }
}

// log0ddl.cc:1906
  ulint thread_id = thd_get_thread_id(thd);
  thread_local_ddl_log_replay = true;
  dberr_t err = replay_by_thread_id(thread_id);
  thread_local_ddl_log_replay = false;
```

★ 一个值得注意的张力：handler 契约说"不能让语句失败"，而 InnoDB 的选择是 **`ib::fatal`**（不继续启动/运行）。因为 DDL log 回放失败意味着磁盘状态与 DD 不一致，继续跑只会更糟——**契约允许"报错"，但 InnoDB 认为这个错严重到必须中止**。

#### server 侧调用点：CREATE 成功时反而不调

选引擎的条件（`sql_table.cc:1173`）：`if ((db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) && db_type->post_ddl) *post_ddl_ht = ...`

| 场景 | 位置 | 时机 |
|---|---|---|
| **DROP 成功** | `sql_table.cc:3716` | `write_bin_log()` → `mysql_bin_log.commit(thd, true)` → **然后** post_ddl |
| **DROP 失败** | `sql_table.cc:3786` | `trans_rollback_stmt` + `trans_rollback` **之后**同样调用（注释：与成功路径保持一致） |
| **CREATE TABLE** | `sql_table.cc:10349` | ★ **成功时不调**（无文件要删），**只在回滚/出错时调**；且 `m_transactional_ddl` 时也跳过 |
| ALTER | `sql_table.cc:13822` / `13869` | 提交后 / 回滚后（回滚侧还要判 `HTON_SUPPORTS_ATOMIC_DDL`） |
| RENAME | `sql_table.cc:18296`、`18338` | `err_with_mdl` 路径，**新旧两个引擎都调** |

★ `CREATE` 那条最反直觉——源码注释解释了原因：*"In case of CREATE TABLE post-DDL hook is mostly relevant for case when statement is rolled back"*。CREATE 成功时新文件本来就该留着，没什么要清理；只有失败时才是"把刚建的 .ibd 删掉"（即回放那条 `DELETE_SPACE` 预案）。

★ 二级引擎那处用了 scope guard 把"无论 commit 还是 rollback 都要执行"写死（`sql_table.cc:11562`）：

```cpp
  // Cleanup that must be done regardless of commit or rollback.
  auto cleanup = [thd, hton]() { hton->post_ddl(thd); ... };
  auto rollback_guard = create_scope_guard([thd, cleanup] {
    trans_rollback_stmt(thd); trans_rollback_implicit(thd); cleanup();
  });
```

#### ★ `trx->ddl_operation`：DDL 事务强制刷 redo

```cpp
// trx0trx.h:925
  bool ddl_operation;   /*!< True if this trx involves dd table change */
  bool ddl_must_flush;  /*!< True if this trx involves dd table change, and must flush */

// trx0trx.cc:1820（提交路径）
  if (trx->ddl_operation || trx->ddl_must_flush) {
    auto wait_stats = log_write_up_to(*log_sys, lsn, true);      // ★ 强制同步刷
  } else {
    trx_flush_log_if_needed_low(lsn);                            // 按 flush_log_at_trx_commit
  }
```

★ **DDL 事务无视 `innodb_flush_log_at_trx_commit = 0`**——必须等 redo 落盘。因为 DDL 的后果（文件删除等）一旦发出就不可撤销，若 redo 没落盘而崩溃，DD 已改、引擎未改，一致性就破了。

`trx0trx.cc:2113` 还有一处配套：`flush_log_later` 时把 `ddl_must_flush = ddl_operation`（注释：*"Remember current ddl_operation, because trx_init() later will set ddl_operation to false. And the final flush is even later."*）——因为 trx 对象会被复用，标记要转存到 `ddl_must_flush` 上，保证最终那次刷盘仍按 DDL 标准执行。

崩溃恢复侧的两条：

- `trx0roll.cc:690`：`if (all || trx->ddl_operation)` —— DDL 事务**即便不是 full rollback 也强制回滚**。
- `trx0trx.cc:843`：`if (trx->ddl_operation || is_prepared)` → 只 resurrect 表级 IX 锁、**不进后台回滚线程**（注释：*"We don't rollback DDL or XA prepared transaction in background"*）。

#### `HTON_SUPPORTS_ATOMIC_DDL` 串起 DD + binlog + 引擎

```cpp
// handler.h:2909
/**
  Engine supports atomic DDL. That is rollback of transaction for DDL
  statement will also rollback all changes in SE, commit of transaction
  of DDL statement will make it durable.
*/
#define HTON_SUPPORTS_ATOMIC_DDL (1 << 12)
```

这个 flag 决定四件事：

1. 是否注册 `post_ddl_ht`（`sql_table.cc:1173`、`2990`）；
2. **DD 改动是否延迟提交**——非原子引擎立刻 `trans_intermediate_ddl_commit`（`sql_table.cc:1161`），原子引擎则留在同一事务里；
3. 是否允许 SE 修改并持久化 `dd::Table*`（`handler.h:6420` rename / `6750` truncate / `6837` create）；
4. binlog 侧：`binlog.cc:10676` `is_create_table_atomic = ... & HTON_SUPPORTS_ATOMIC_DDL`（决定 CREATE TABLE 的 binlog 写法）。

⇒ **三者一致的完整机制**：DD 改动（InnoDB 表）与 binlog 在同一个 `trans_commit_stmt`/`trans_commit_implicit`（含 2PC）里提交；引擎内部的**不可逆副作用**先写进 `innodb_ddl_log`（与 DD 同事务），提交**成功之后**才由 `post_ddl` 回放。于是：

- **提交前崩溃** ⇒ 事务整体回滚 + `Log_DDL::recover()` 回放残留记录；
- **提交后崩溃** ⇒ 恢复时 `replay_all()` 补做（靠幂等性扛住）。

## 可观测性

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_print_ddl_logs` | OFF | Global | 把 DDL log 的写入/删除/replay 打到 stderr（错误日志）；★ 还需 `log_error_verbosity=3` 才可见 |
| `innodb_ddl_log_crash_reset_debug` | — | Global（debug） | 重置崩溃注入计数器，仅 MTR 用 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 说明 |
|--------|------|------|
| 一次 DDL 写了哪些 log | 变量 | `SET GLOBAL innodb_print_ddl_logs=1` + `log_error_verbosity=3`，看错误日志 |
| DDL log 表内容 | — | **不可直接查**（隐藏表，非 debug 版本禁止访问），只能靠上面那个变量 |
| 崩溃后是否有残留 | 启动日志 | WL#9536 FR3：只读模式下若恢复出残留记录，server 会输出提示消息 |
| DDL 三处是否一致 | 对比 | DD（`information_schema`）／引擎（`.ibd` 文件）／binlog 三处对照 |

### ★ 读懂 `innodb_print_ddl_logs` 输出（官方文档示例）

```
SET GLOBAL innodb_print_ddl_logs=1;
CREATE TABLE t1 (c1 INT) ENGINE = InnoDB;

[Note] InnoDB: DDL log insert : [DDL record: DELETE SPACE, id=18, thread_id=7, space_id=5, old_file_path=./test/t1.ibd]
[Note] InnoDB: DDL log delete : by id 18
[Note] InnoDB: DDL log insert : [DDL record: REMOVE CACHE, id=19, thread_id=7, table_id=1058, new_file_path=test/t1]
[Note] InnoDB: DDL log delete : by id 19
[Note] InnoDB: DDL log insert : [DDL record: FREE, id=20, thread_id=7, space_id=5, index_id=132, page_no=4]
[Note] InnoDB: DDL log delete : by id 20
[Note] InnoDB: DDL log post ddl : begin for thread id : 7
[Note] InnoDB: DDL log post ddl : end for thread id : 7
```

这段输出是「写后即删」机制的**直接证据**：三条 insert（DELETE SPACE / REMOVE CACHE / FREE）各自紧跟一条 `delete by id`——成功路径上这些记录全部被主事务删除，`post ddl` 区间内没有任何 replay。**若哪天看到 insert 后面没有对应的 delete，说明那个操作被保留下来等 replay（失败或崩溃清理路径）。**

---

## Misc

### 面向二次开发

**扩展点**：新增一种需要延迟执行的物理操作类型，要动三处——`Log_Type` 枚举加值、`write_*_log`（写意图）+ `replay_*_log`（执行）、`Log_DDL::replay` 的 switch 加分支。**replay 必须实现为幂等**（否则崩溃恢复重复执行会出错）。

### 坑与已知缺陷

| 坑 | 说明 |
|---|---|
| `DROP TABLE` 语义变化导致复制失败 | 8.0 起"部分成功"不再可能：`DROP TABLE t1,t2` 中 t2 不存在则整句失败。5.7 主库上部分完成的语句同步到 8.0 备库会失败 → 官方建议一律加 `IF EXISTS` |
| `innodb_force_recovery > 0` 会跳过 DDL log 恢复 | 强制启动模式下残留 DDL log 不被 replay，可能留下孤儿文件 |
| DDL 期间 halt 一定留 log | 正常情况 `DDL_LOG` 表应为空（FR2a）；非空即说明上次 DDL 异常终止 |
| 只读模式下的残留提示 | FR3：server 应提示恢复出的不完整 DDL 事务或残留记录；未提交事务按"未提交"处理，不阻止启动 |

### 易混淆概念

| 概念 | 辨析 |
|---|---|
| 原子 DDL vs 事务性 DDL | 原子＝这条 DDL 自身全成或全败；**不等于** DDL 能进用户事务——DDL 仍隐式提交 |
| `innodb_table_metadata` vs `innodb_dynamic_metadata` | 同一处东西的两种叫法：源码注释（log0ddl.h）写 `innodb_table_metadata`，官方 WL#9536 与文档写 `innodb_dynamic_metadata`，指的都是存放 autoinc 等动态元数据的那张非事务持久表 |
| DDL log vs redo log | DDL log 记的是**延迟/撤销用的物理操作意图**（普通表 + 强制刷盘 redo）；redo 是引擎的物理日志。前者是 DDL 语义层，后者是存储层 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 15.1.1 Atomic Data Definition Statement Support*（四阶段定义、支持的语句范围、行为变化、`innodb_ddl_log` schema 与 `innodb_print_ddl_logs` 示例）

**WorkLog**
- [WL#9536](https://dev.mysql.com/worklog/task/?id=9536) *InnoDB_New_DD: Support crash-safe DDL* —— 本篇主干依据：四阶段、DDL_LOG schema、四种操作的 log 写入时机、`FR5` 强制刷盘、幂等性要求、`innodb_print_ddl_logs` 的引入
- [WL#9525](https://dev.mysql.com/worklog/task/?id=9525) *InnoDB_New_DD: update InnoDB specific metadata into new DD for DDL* —— 引擎把 `se_private_data` / `se_private_id` / `options` 回写进 `dd::Table` 的官方依据（对应"引擎侧一半"）；它自述依赖 WL#7743
- [WL#7743](https://dev.mysql.com/worklog/task/?id=7743) *New data dictionary: changes to DDL-related parts of SE API* —— 把 `dd::Table` 传给引擎的 SE API 改造，原子 DDL 的前置

**相关文档**
- 上层：server 侧 DDL 编排骨架、算法选择策略与 MDL 三段式见 [`ddl.md`](ddl.md)
- 三条执行路：COPY 见 [`copy_ddl.md`](copy_ddl.md)、INPLACE 见 [`inplace_ddl.md`](inplace_ddl.md)、INSTANT 见 [`instant_ddl.md`](instant_ddl.md)
- 被 DDL log 延迟执行的物理操作（B-tree 释放 btr_free、.ibd 删除）见 [`ddl.md`](ddl.md) 的 DROP / TRUNCATE 章
