# CHECK TABLE 全链路深度解析

> 基于 MySQL 8.0.39 源码。本篇讲 **CHECK TABLE 从 SQL 语法到引擎逐页校验的完整链路**：server 编排（ADMIN 语句家族 → `handler::ha_check` 契约 → `HA_ADMIN_*` 返回码 → `Msg_type` 结果表）↔ InnoDB 实现（`ha_innobase::check` 三段流程 + 行级扫描器）。
>
> **边界**：B-tree 结构校验算法（`btr_validate_index`/`btr_validate_level` 逐项校验表）在本篇「B-tree 结构校验算法」节（[`index/btr.md`](../index/btr.md) 只留互指）；ANALYZE/OPTIMIZE 的引擎侧深挖分别在 [`../index/stats.md`](../index/stats.md) 与 [`index/operations.md`](../index/operations.md)；升级检查体系（`check_for_upgrade` 的 DD 版本写入）见 [`server/dd/dd.md`](../server/dd/dd.md)「升级前体检」节。

## 是什么

`CHECK TABLE t [QUICK|FAST|MEDIUM|EXTENDED|CHANGED|FOR UPGRADE]` 是 MySQL 的**表诊断语句**：server 层打开表、调引擎的 `check()` 实现对每个索引做"结构校验（B-tree 逐项对错判断）+ 行级检查（记录计数、重复键检测）"，把结果聚合成 4 列（Table/Op/Msg_type/Msg_text）结果集返回。**只诊断不修复**（InnoDB 没有 repair 实现，见 [REPAIR 边界](#repair-table-与-for-upgrade)）。

## 这条操作经过了什么（全景调用栈）

```
mysql_execute_command (sql/sql_parse.cc)
  └─ Sql_cmd_check_table::execute            sql/sql_admin.cc:1755
       ├─ check_table_access                 sql/sql_authorization.cc:2334
       └─ mysql_admin_table                  sql/sql_admin.cc:699   ← 四语句共用框架
            ├─ (构造结果集 4 列)              :728
            ├─ open_and_lock_tables           :823/:857  (MDL_SHARED_READ + TL_READ_NO_INSERT)
            └─ (table->file->*operator_func)  :1129   ← 成员函数指针分派
                 └─ handler::ha_check         sql/handler.cc:4704
                      ├─ check_table_for_old_types   :4718 (版本<当前时)
                      ├─ handler::ha_check_for_upgrade :4720
                      └─ handler::check (虚)  :4724
                           └─ ha_innobase::check       ha_innodb.cc:18156
            └─ send_result: switch(result_code) :1175  → 输出 Msg_type/Msg_text
```

## 版本演进

| 版本 | 变化 | 设计依据 |
|---|---|---|
| 3.x~5.x | `CHECK TABLE` 语句与四档位图（`T_QUICK/FAST/MEDIUM/EXTEND`）确立（语义继承自早期存储引擎时代） | **早于 MySQL Worklog 体系，无编号** |
| 5.x | `CHECK TABLE ... FOR UPGRADE`：`TT_FOR_UPGRADE` 写进 `sql_flags`（与四档位图分离），供 `mysql_upgrade` 客户端逐表体检 | 早于 Worklog 体系，无编号 |
| 8.0.0 | 事务性数据字典：升级检查机制（`is_checked_for_upgrade`/`last_checked_for_upgrade` 版本号写 DD、升级沙箱 schema） | [WL#9553](https://dev.mysql.com/worklog/task/?id=9553) *Upgrading the transactional data dictionary tables* |
| 8.0.16 | `mysql_upgrade` 客户端退役、升级任务并入 server（`upgrade_system_schemas` 拼大 `CHECK TABLE ... FOR UPGRADE` 从 `Msg_text` 抠 `ALTER TABLE` 建议执行） | 未找到对应 WL 编号（详见 [`server/dd/dd.md`](../server/dd/dd.md)） |

## 设计思想与权衡

**1. "诊断"与"修复"分离是 InnoDB 引擎哲学的投影**。InnoDB 有 redo 前滚 + MVCC 回滚保证完整性，**物理损坏意味着 bug 而非可修状态**——"修复"的正确姿势是重建（`ALTER TABLE ENGINE=InnoDB`，见 REPAIR 边界）。所以 InnoDB 只有 `check()` 没有 `repair()`，这是**能力位 `HA_CAN_REPAIR` 显式声明**的体现（`handler::repair` 默认实现断言引擎没声明该能力，InnoDB 未声明所以 `repair()` 永远到不了引擎层）。

**2. InnoDB 是单档实现**。InnoDB 的 B-tree 是自描述结构（页内目录槽 + 页间 node pointer），结构校验只有"整树有效/无效"两态——四档里只有 `T_QUICK` 生效（跳过结构校验只数行），`T_FAST`/`T_CHANGED`/`T_EXTEND` 对 InnoDB 无差别。

**3. 结构校验失败不落 corrupted 标记的不对称**（`ha_innobase::check` 里最反直觉的一处）：`btr_validate_index` 返回 false 只 warning + `continue`；`dict_set_corrupted` 只由行级扫描失败/计数不符触发。源码没有注释解释原因；合理的工程推理：`dict_set_corrupted` **不可逆**（落标后开表直接拒查），结构校验在 SX 锁下可能受瞬时 IO 错误/恢复期边界状态影响产生不确定结论，而行级扫描（逐条读记录）的结果更确定、且顺带产出损坏定位——把"不可逆的落标"留给更确定的证据。

**4. 他库对比**：PostgreSQL 没有 CHECK TABLE（完整性由 WAL + MVCC 保证，诊断靠第三方扩展 amcheck 的 `bt_index_check`）；SQL Server 有 `DBCC CHECKDB`（引擎内置、可带 `REPAIR_ALLOW_DATA_LOSS` 修复选项——与 InnoDB 只诊断形成对照）；Oracle 无等价物（`ANALYZE TABLE VALIDATE STRUCTURE` 近似）。

## ADMIN 语句家族：`mysql_admin_table` 一框架四兄弟

CHECK/REPAIR/OPTIMIZE/ANALYZE 各有独立 `Sql_cmd` 子类（`sql/sql_admin.h:177/207/192/64`），但 `execute()` 全部收敛到**同一个静态函数 `mysql_admin_table()`**（`sql/sql_admin.cc:699`），通过 **C++ 成员函数指针** `operator_func`（`int (handler::*)(THD *, HA_CHECK_OPT *)`）分派——**没有 handlerton 层 check 回调**，多引擎分派全走 handler 虚函数。

| 语句 | execute | operator_func | lock_type | open_for_modify | binlog |
|---|---|---|---|---|---|
| CHECK | `:1755` | `&handler::ha_check` | `TL_READ_NO_INSERT` | false | 无 |
| REPAIR | `:1804` | `&handler::ha_repair` | `TL_WRITE` | true | **有**（`:1819`） |
| OPTIMIZE | `:1776` | `&handler::ha_optimize` | `TL_WRITE` | true | 有（`:1791`） |
| ANALYZE | `:1716` | `&handler::ha_analyze` | `TL_READ_NO_INSERT` | false | 无 |

CHECK 的完整调用（`sql/sql_admin.cc:1765`）：

```cpp
  res = mysql_admin_table(thd, first_table, &thd->lex->check_opt, "check",
                          lock_type, false, false, HA_OPEN_FOR_REPAIR, nullptr,
                          &handler::ha_check, 1, m_alter_info, true);
```

- `"check"` → 结果集 `Op` 列；`check_view=1` 允许对 view 执行；`HA_OPEN_FOR_REPAIR` 允许在损坏场景下打开（InnoDB 不依赖它）；
- `lock_type` 决定 MDL：`>= TL_WRITE_ALLOW_WRITE` 取 `MDL_SHARED_NO_READ_WRITE`，否则 **`MDL_SHARED_READ`**（`:778`）——CHECK/ANALYZE 是 `TL_READ_NO_INSERT` + `MDL_SHARED_READ`，REPAIR/OPTIMIZE 是 `TL_WRITE` 写锁。

### `mysql_admin_table` 主函数的关键流程（sql_admin.cc:699-1544，约 800 行）

**整体职责**：ADMIN 语句族的统一编排——逐表"打开 → 分派 handler 虚函数 → 聚合结果输出"，外加 view 物化/crashed 警告/FOR UPGRADE/TRY_ALTER 重建/TDCM 清理等旁路。骨架（关键环节）：

```cpp
static bool mysql_admin_table(THD *thd, Table_ref *tables, HA_CHECK_OPT *check_opt,
                              const char *operator_name, thr_lock_type lock_type, ...) {
  Disable_autocommit_guard autocommit_guard(thd);      // 防 DD 更新后自动提交
  ... /* 结果集 4 列构造（Table/Op/Msg_type/Msg_text）*/
  close_thread_tables(thd);                            // 关预打开表，简化权限检查
  /* gtid_next 为 ASSIGNED/ANONYMOUS 时提前置 skip_gtid_rollback：
     该语句失败也写 binlog，失败路径 trans_rollback_stmt 会释放 GTID 所有权 */
  if (gtid_rollback_must_be_skipped) thd->skip_gtid_rollback = true;

  for (table = tables; table; table = table->next_local) {   // ★ 逐表循环
    table->set_lock({lock_type, THR_DEFAULT});               // THR_LOCK 层
    table->mdl_request.set_type((lock_type >= TL_WRITE_ALLOW_WRITE)
                                    ? MDL_SHARED_NO_READ_WRITE
                                    : MDL_SHARED_READ);      // MDL 层换算
    {
      /* 每轮只开一个表：临时断 next_global/next_local 链，
         避免 open_and_lock_tables 顺链打开全部 */
      save_next_global = table->next_global; table->next_global = nullptr;
      ...
      /* CHECK 允许视图（check_view==1 且非 ALTER_ADMIN_PARTITION），
         否则 required_type = BASE_TABLE */
      if (alter_info->flags & Alter_info::ALTER_ADMIN_PARTITION || check_view != 1)
        table->required_type = dd::enum_table_type::BASE_TABLE;
      open_error = open_temporary_tables(thd, table);
      if (!open_error) open_error = open_and_lock_tables(thd, table, 0);
      if (!open_error && need_to_acquire_shared_backup_lock)
        acquire_shared_backup_lock(thd, ...);               // admin 语句要显式拿 backup lock
      /* 视图总是物化：resolve_derived + setup_materialized_derived */
      if (!open_error && table->is_view()) { ... }
      table->next_global = save_next_global; ...
    }
    ...
    if (!table->table) {                                     // open 失败
      ... result_code = HA_ADMIN_FAILED 或 HA_ADMIN_CORRUPT（非损坏类错误才 FAILED）;
      goto send_result;
    }
    if (table->is_view()) { result_code = HA_ADMIN_OK; goto send_result; }  // ★ 视图：直接 OK，不调 check
    if (table->schema_table) { result_code = HA_ADMIN_NOT_IMPLEMENTED; goto send_result; }
    /* 只读表（HA_READ_ONLY）+ open_for_modify（REPAIR/OPTIMIZE/ANALYZE）→ ER_OPEN_AS_READONLY；
       CHECK（open_for_modify=false）不受限 */
    ...
    /* CHECK 且表 crashed：先打一行 warning "Table is marked as crashed" */
    if (table->table->s->crashed && operator_func == &handler::ha_check) { ... }
    /* TT_FOR_UPGRADE：走 check_for_upgrade 的 DD 版本检查（见 FOR UPGRADE 节）*/
    if (check_opt && (check_opt->sql_flags & TT_FOR_UPGRADE) != 0) {
      Check_result cr = check_for_upgrade(thd, snam, tnam, [&]() {
        return (table->table->file->*operator_func)(thd, check_opt);
      });
      result_code = cr.second; if (cr.first) goto err;
    } else {
      result_code = (table->table->file->*operator_func)(thd, check_opt);   // ★ 核心分派点
    }
  send_result:
    thd->clear_error();
    if (send_analyze_table_errors(thd, operator_name, table_name)) goto err; // 引擎 warning 逐条输出
    protocol->start_row(); store(Table); store(Op);
  send_result_message:
    switch (result_code) { ... }                            // HA_ADMIN_* → Msg_type/Msg_text（见结果聚合节）
    ...
    /* 收尾：TDCM（fatal_error/ANALYZE 时 tdc_remove_table）、提交、关表、放 MDL */
    if (protocol->end_row()) goto err;
  }
  my_eof(thd);
  if (gtid_rollback_must_be_skipped) thd->skip_gtid_rollback = false;   // 恢复
  return false;
err:
  ... trans_rollback_stmt + trans_rollback + invalidate_dict + close + release;
  return true;
}
```

几个容易被忽略的细节：

1. **逐表断链打开**：每轮临时断开 `next_global/next_local` 只开一张表——避免锁住整条表链；
2. **视图上的 CHECK 本质**：`is_view()` 直接返回 `status/OK` **不调 `handler::check`**——真正的"检查"发生在 `setup_materialized_derived` 物化过程（视图定义可解析即 OK）；
3. **`share->crashed` 与 `HA_OPEN_FOR_REPAIR`**：普通语句打开损坏表会置 crashed（可能触发自动修复），CHECK/REPAIR 带了 `HA_OPEN_FOR_REPAIR` → **不自动置位**，让用户显式看到状态；
4. **REPAIR 的 binlog 在 `mysql_admin_table` 返回后统一写**（`:1819`，"we write after unlocking the table"）——CHECK 不写 binlog；
5. **GTID 技巧**：`skip_gtid_rollback` 保证"失败也写 binlog"的语句在回滚路径不丢 GTID 所有权。

### `handler::ha_check` 完整（sql/handler.cc:4704-4725）：分派前的版本壳

```cpp
int handler::ha_check(THD *thd, HA_CHECK_OPT *check_opt) {
  int error;
  assert(table_share->tmp_table != NO_TMP_TABLE || m_lock_type != F_UNLCK);

  /* 已是最新版本 + FOR UPGRADE → 直接短路返回 HA_ADMIN_OK */
  if ((table->s->mysql_version >= MYSQL_VERSION_ID) &&
      (check_opt->sql_flags & TT_FOR_UPGRADE))
    return 0;

  if (table->s->mysql_version < MYSQL_VERSION_ID) {   // 旧版本表才做升级检查
    mysql_mutex_lock(&LOCK_global_system_variables);
    const bool check_temporal_upgrade = !avoid_temporal_upgrade;
    mysql_mutex_unlock(&LOCK_global_system_variables);
    if ((error = check_table_for_old_types(table, check_temporal_upgrade)))  // 旧 temporal 类型
      return error;
    error = ha_check_for_upgrade(check_opt);          // 引擎级升级检查
    if (error && (error != HA_ADMIN_NEEDS_CHECK)) return error;
    if (!error && (check_opt->sql_flags & TT_FOR_UPGRADE)) return 0;
  }
  return check(thd, check_opt);                       // ★ 最终虚调用 → ha_innobase::check
}
```

**分工**：`ha_check` 只做 server 层的版本/temporal 前置检查 + 虚调用转发；`TT_FOR_UPGRADE` 的 DD 版本号写入不在它这层（在 `mysql_admin_table` 的 `check_for_upgrade` 包裹层）。

## check_opt 四档：两套位图的传递链

yacc 归约（`sql/sql_yacc.yy:9639-9674`）把选项填进 `LEX::check_opt`（`HA_CHECK_OPT`，`handler.h:3635`——只有 `flags`/`sql_flags`/`key_cache` 三个字段）：

- **`flags` 位图**（`include/my_check_opt.h:40-54`，历史引擎语义位被 server 层沿用）：`T_QUICK`(1<<15) / `T_FAST`(1<<10) / `T_MEDIUM`(1<<14) / `T_EXTEND`(1<<9) / `T_CHECK_ONLY_CHANGED`(1<<5)；**默认 `T_MEDIUM`**（`:9648` `%empty` 规则）；
- **`sql_flags` 位图**（`include/myisam.h:312-313`）：`TT_FOR_UPGRADE=2`、`TT_USEFRM=1`——`FOR UPGRADE` 写的是 **sql_flags 不是 flags**，与四档完全分离；
- 传递：`PT_check_table_stmt::make_cmd`（`parse_tree_nodes.cc:3199`）按位或合并进 `lex->check_opt`。

**InnoDB 只认 `T_QUICK` 一档**：跳过 `btr_validate_index` 结构校验、只做行级扫描（`ha_innodb.cc:18224`）；`T_FAST`/`T_MEDIUM`/`T_EXTEND`/`T_CHECK_ONLY_CHANGED` 对 InnoDB 行为完全相同（都做完整校验）。

## handler 契约与 `HA_ADMIN_*` 返回码

虚函数默认实现（`sql/handler.h:6566-6577`）：`check()` 默认返回 `HA_ADMIN_NOT_IMPLEMENTED`，`repair()` 默认**断言引擎没声明 `HA_CAN_REPAIR`** 后返回 `HA_ADMIN_NOT_IMPLEMENTED`。

`HA_ADMIN_*` 全清单（`handler.h:173-189`）：`OK=0` / `ALREADY_DONE=1` / `NOT_IMPLEMENTED=-1` / `FAILED=-2` / **`CORRUPT=-3`** / `INTERNAL_ERROR=-4` / `INVALID=-5` / `REJECT=-6` / **`TRY_ALTER=-7`**（OPTIMIZE 触发重建）/ `WRONG_CHECKSUM=-8` / `NOT_BASE_TABLE=-9` / `NEEDS_UPGRADE=-10` / `NEEDS_ALTER=-11` / `NEEDS_CHECK=-12` / `STATS_UPD_ERR=-13` / `NEEDS_DUMP_UPGRADE=-14`。

## InnoDB 侧：`ha_innobase::check` 三段流程（`ha_innodb.cc:18156` 完整）

**整体职责**：CHECK TABLE 的引擎实现——前置门禁 → 对每个 committed 索引做"结构校验（可选）+ 行级扫描计数/查重复"→ 汇总返回 `HA_ADMIN_OK/CORRUPT`。输入 `(THD*, HA_CHECK_OPT*)`，输出是 handler 返回码 + 逐条 `push_warning`。

### ① 前置门禁（18156-18194）

```cpp
  if (m_prebuilt->mysql_template == nullptr) {
    /* Build the template; we will use a dummy template
    in index scans done in checking */
    build_template(true);                 // 只为让 mysql_template 指针非空，
  }                                       // 真正扫描前会被临时覆盖为 dummy（见④）
  if (dict_table_is_discarded(m_prebuilt->table)) {
    ib_senderrf(thd, IB_LOG_LEVEL_ERROR, ER_TABLESPACE_DISCARDED, ...);
    return HA_ADMIN_CORRUPT;              // ① 表空间被 discard：直接损坏
  } else if (m_prebuilt->table->ibd_file_missing) {
    ib_senderrf(thd, IB_LOG_LEVEL_ERROR, ER_TABLESPACE_MISSING, ...);
    return HA_ADMIN_CORRUPT;              // ② .ibd 丢失
  }
  m_prebuilt->trx->op_info = "checking table";   // PROCESSLIST 可见（收尾清空）
```

### ② 已标记 corrupted 短路 + 隔离级别强制提升（18196-18215）

```cpp
  if (m_prebuilt->table->is_corrupted()) {
    /* Now that the table is already marked as corrupted,
    there is no need to check any index of this table */
    return HA_ADMIN_CORRUPT;              // 已落标 → 不再逐索引检查
  }

  old_isolation_level = m_prebuilt->trx->isolation_level;
  /* We must run the index record counts at an isolation level
  >= READ COMMITTED, because a dirty read can see a wrong number
  of records in some index; to play safe, we use always
  REPEATABLE READ here */
  m_prebuilt->trx->isolation_level = TRX_ISO_REPEATABLE_READ;   // ★ 计数在脏读下会数错（注释原文见上）
```

### ③ 逐索引循环：结构校验段（18217-18244）

```cpp
  for (index = m_prebuilt->table->first_index(); index != nullptr;
       index = index->next()) {
    if (!index->is_committed()) continue;          // 在线 DDL 中的索引跳过

    if (!(check_opt->flags & T_QUICK) && !index->is_corrupted()) {
      /* Enlarge the fatal lock wait timeout during CHECK TABLE. */
      srv_fatal_semaphore_wait_extend.fetch_add(1);   // ★ 大表校验持锁久，
      bool valid = btr_validate_index(index, m_prebuilt->trx, false);  // 防 semaphore
      srv_fatal_semaphore_wait_extend.fetch_sub(1);   //   等待自杀（±1 成对）
      if (!valid) {
        is_ok = false;
        push_warning_printf(thd, Sql_condition::SL_WARNING, ER_NOT_KEYFILE,
                            "InnoDB: The B-tree of index %s is corrupted.",
                            index->name());
        continue;                                   // ★ 只 warning 不落 corrupted 标记
      }
    }
```

- `btr_validate_index(index, trx, false)`：`lockout=false` → 索引锁加 **SX**（非 X）——排他写者、允许并发读（逐项校验算法见 [btr.md](../index/btr.md)）；
- `srv_fatal_semaphore_wait_extend` 与统计逐层扫描同款防御；
- **结构校验失败只 `continue`，不 `dict_set_corrupted`**——不对称设计（原因推理见 [设计思想与权衡](#设计思想与权衡)）。

### ④ 行级扫描段（18246-18288）：dummy template 登场

```cpp
    m_prebuilt->index = index;                     // 绕过 change_active_index()
    m_prebuilt->index_usable = m_prebuilt->index->is_usable(m_prebuilt->trx);
    if (!m_prebuilt->index_usable) {
      // is_corrupted → HA_ERR_INDEX_CORRUPT；否则 HA_ERR_TABLE_DEF_CHANGED
      // （"Insufficient history for index %s"——历史版本不足）；都 is_ok=false + continue
    }
    m_prebuilt->sql_stat_start = true;
    m_prebuilt->template_type = ROW_MYSQL_DUMMY_TEMPLATE;  // ★ dummy template：
    m_prebuilt->n_template = 0;                            //   不转 MySQL 行格式，
    m_prebuilt->need_to_access_clustered = false;          //   整 rec 原样拷 buf
    m_prebuilt->clear_search_tuples();
    m_prebuilt->select_lock_type = LOCK_NONE;              // 非锁读（consistent read）

    size_t max_threads = thd_parallel_read_threads(m_prebuilt->trx->mysql_thd);
    if (dict_index_is_spatial(index)) {
      ret = row_count_rtree_recs(m_prebuilt, &n_rows, &n_dups);   // R-tree 专用
    } else {
      ret = row_scan_index_for_mysql(m_prebuilt, index, max_threads, true, &n_rows);
    }
```

**dummy template**（`ROW_MYSQL_DUMMY_TEMPLATE=3`，`row0mysql.h:1049`）：check/COUNT(*) 专用占位模式——跳过 `row_sel_store_mysql_rec`，把整条 InnoDB rec 原样拷进 buf、记录偏移写前 4 字节（`row0sel.cc:5731-5744`），扫描器用 `rec = buf + mach_read_from_4(buf)` 取回。check 只需数记录和比相邻键，不需要完整行——省掉模板列映射开销。

### ⑤ 结果判定（18290-18326）：计数比对 + 落 corrupted 标记

```cpp
    if (ret == DB_INTERRUPTED || thd_killed(m_user_thd)) break;  // shutdown 不报错
    if (ret != DB_SUCCESS) {
      push_warning_printf(thd, Sql_condition::SL_WARNING, ER_NOT_KEYFILE,
                          "InnoDB: The B-tree of index %s is corrupted.", ...);
      is_ok = false;
      dict_set_corrupted(index);                        // ★ 行级失败 → 才落标
    }
    if (index == m_prebuilt->table->first_index()) {
      n_rows_in_table = n_rows;                         // 聚簇索引行数为基准
    } else if (!(index->type & DICT_FTS) && (n_rows != n_rows_in_table) &&
               (!index->is_multi_value()) &&
               (!dict_index_is_spatial(index) || (n_rows < n_rows_in_table) ||
                (n_dups < n_rows - n_rows_in_table))) {
      push_warning_printf(thd, Sql_condition::SL_WARNING, ER_NOT_KEYFILE,
                          "InnoDB: Index '%-.200s' contains %lu entries, should be %lu.",
                          index->name(), (ulong)n_rows, (ulong)n_rows_in_table);
      is_ok = false;
      dict_set_corrupted(index);                        // 计数不符 → 也落标
    }
```

计数比对规则：二级索引（FTS/多值除外）行数 ≠ 聚簇基准 → 报错；**空间索引特殊**（`n_rows >= n_rows_in_table` 合法——R-tree 页内可有重复 MBR 项），仅当 `n_rows < 基准` 或重复数超过差值时报错。**不对称设计总结**：结构校验失败 → 只 warning 不落标；行级扫描失败/计数不符 → warning + `dict_set_corrupted`。

### ⑥ 收尾（18328-18348）

恢复隔离级别 → **仅 DEBUG/AHI_DEBUG 编译**下非 QUICK 时额外 `btr_search_validate()` 校验整个 AHI → 清 `op_info` → `return is_ok ? HA_ADMIN_OK : HA_ADMIN_CORRUPT`。

### 七个被主流程压缩掉的接口（逐个补齐）

**① R-tree 计数 `row_count_rtree_recs`**（row0sel.cc:6109）：空间索引无全序，无法用 `PAGE_CUR_G` 顺序游标——构造首字段 `DATA_GIS_MBR` 的**无穷大 MBR**（`tmp_mbr[j*2]=DBL_MAX`/`+1=-DBL_MAX` 覆盖全域）搜索元组，用 `PAGE_CUR_WITHIN`（空间包含判定）逐条遍历；`rtr_info->is_dup` 指针挂在局部变量上，底层遍历遇重复 MBR 置位 → `n_dups++`。错误处理与 B-tree 串行 loop **同款"忽略"**（`default fall through → DB_END_OF_INDEX → DB_SUCCESS`）。

**② `dict_set_corrupted` 的完整语义**（dict0dict.cc:4163）：幂等置位 `index->type |= DICT_CORRUPT`（**只在字典缓存**，注释"cache only"——为了索引自身字典损坏时仍能强制加载以修复）；非只读非临时表时把 `(space, index_id)` 经 `PM_INDEX_CORRUPTED` persister 写 **redo log**（`log_write_up_to` 保证位不丢失）+ `dict_table_mark_dirty` 持久化。**置位后**：`is_usable()` 返回 false → DML/查询拒绝使用该索引；CHECK 入口短路（表已 corrupted 直接返回）。**没有 `dict_reset_corrupted`**（全库 0 命中）——corruption 位运行时**不可清除**，唯一恢复是重建索引/表重新加载字典对象。

**③ `index->is_usable` 与 "Insufficient history" 的真义**（dict0mem.cc:604）：

```cpp
bool dict_index_t::is_usable(const trx_t *trx) const {
  if (!is_clustered() && dict_index_is_online_ddl(this)) return false;  // 在线 DDL 创建中
  if (is_corrupted()) return false;                                     // 已损坏
  /* Check if the specified transaction can see this index. */
  return (table->is_temporary() || trx_id == 0 ||
          !MVCC::is_view_active(trx->read_view) ||
          trx->read_view->changes_visible(trx_id, table->name));        // ★ MVCC 可见性
}
```

CHECK 里 `!index_usable` 的两个分支：`is_corrupted()` → `HA_ERR_INDEX_CORRUPT`（且 is_ok=false）；否则 → `HA_ERR_TABLE_DEF_CHANGED` "Insufficient history"。**"Insufficient history"不是 undo 历史不足**，而是 **MVCC 读视图可见性**：该索引的 `trx_id`（创建事务）晚于当前 read view 低水位——CHECK 强制 RR 后，另一个事务刚提交的建索引对这个 read view 不可见（"历史不足以看到该索引"）。

**④ `is_committed` 与 `online_status` 是两套正交状态**：`is_committed()` 读独立的 `uncommitted` 位（索引定义是否已写 DD）；`dict_index_is_online_ddl()` 读 2-bit `online_status`（`ONLINE_INDEX_COMPLETE/CREATION/ABORTED/ABORTED_DROPPED`）。CHECK 主循环先以 `!is_committed()` 粗粒度跳过（在线 DDL 中、B-tree 未构建完的索引不参与整个 CHECK）；`btr_validate_index` 再以 `dict_index_is_online_ddl()` 细粒度二次跳过（覆盖"已提交但 online_status 非 COMPLETE"的边界）——两层共同保证中间态索引不参与结构校验。

## B-tree 结构校验算法：`btr_validate_index` / `btr_validate_level`

**整体职责**：验证一棵 B-tree 的全部结构不变量——页链、键序、node pointer、段标记。`btr_validate_index`（编排）从 root 起**逐层**调 `btr_validate_level`（单层校验，430 行大函数）；后者**逐页**扫描该层所有页。输入 `(index, trx, lockout)`，输出 bool（valid）；错误走 warning + `ret=false`（release）或 `ut_a` abort（debug）。

### 编排：`btr_validate_index`（btr0btr.cc:4509）

```cpp
bool btr_validate_index(dict_index_t *index, const trx_t *trx, bool lockout) {
  /* Full Text index are implemented by auxiliary tables, not the B-tree */
  if (dict_index_is_online_ddl(index) || (index->type & DICT_FTS)) {
    return (true);                        // ① 在线 DDL / FTS 辅助表：不是真 B-tree，跳过
  }
  if (dict_index_is_spatial(index)) {
    return (btr_validate_spatial_index(index, trx));   // ② R-tree 走专用校验（独立算法）
  }
  mtr_t mtr;
#ifdef UNIV_DEBUG
  { /* Check the FSEG_NOT_FULL_N_USED field stored in the segment inode. */
    ... fseg_inode_get(seg_header, ...);
    ut_ad(fsi.verify_not_full_n_used());  // ③ debug 下先校验段 inode 的空闲页计数
  }
#endif /* UNIV_DEBUG */
  mtr_start(&mtr);
  if (!srv_read_only_mode) {
    if (lockout) {
      mtr_x_lock(dict_index_get_lock(index), &mtr, UT_LOCATION_HERE);   // DDL 用 X
    } else {
      mtr_sx_lock(dict_index_get_lock(index), &mtr, UT_LOCATION_HERE);  // CHECK 用 SX
    }
  }
  bool ok = true;
  page_t *root = btr_root_get(index, &mtr);
  ulint n = btr_page_get_level(root);        // 树高度（root 页 PAGE_LEVEL）
  for (ulint i = 0; i <= n; ++i) {
    if (!btr_validate_level(index, trx, n - i, lockout)) {   // 从顶层 n 校验到叶层 0
      ok = false;
      break;                                 // 任一层失败即停（后续层意义不大）
    }
  }
  mtr_commit(&mtr);
  return (ok);
}
```

### 单层校验：`btr_validate_level`（btr0btr.cc:4036，430 行骨架化）

**阶段一：沿最左 node pointer 下钻到目标层**：

```cpp
static bool btr_validate_level(dict_index_t *index, const trx_t *trx,
                               ulint level, bool lockout) {
  mtr_start(&mtr);
  /* 与 validate_index 相同的 index->lock X/SX 获取（每层重复获取——索引锁不跨层持有）*/
  block = btr_root_block_get(index, RW_SX_LATCH, &mtr);   // 页级一律 SX：防页内增删、允许读
  page = buf_block_get_frame(block);
  seg = page + PAGE_HEADER + PAGE_BTR_SEG_TOP;            // 非叶段头（叶层校验时换成 LEAF）

  /* ★ 表 flags 与表空间 flags 必须一致（防"字典说 DYNAMIC、表空间存 COMPACT"）*/
  const fil_space_t *space = fil_space_get(index->space);
  if (!table_page_size.equals_to(space_page_size)) {
    ib::warn(ER_IB_MSG_38) << "Flags mismatch: ...";
    return (false);
  }

  while (level != btr_page_get_level(page)) {   // 还没到目标层：继续下钻
    const rec_t *node_ptr;
    if (fseg_page_is_free(seg, ...)) { ... ret = false; }   // 下钻途中的页被段标记 free
    ut_a(!page_is_leaf(page));                  // 未到目标层却已是叶子 → 树结构矛盾
    page_cur_set_before_first(block, &cursor);
    page_cur_move_to_next(&cursor);
    node_ptr = page_cur_get_rec(&cursor);       // ★ 总是取最左 node pointer（infimum 后第一条）
    offsets = rec_get_offsets(node_ptr, index, offsets, ULINT_UNDEFINED, ..., &heap);
    savepoint2 = mtr_set_savepoint(&mtr);
    block = btr_node_ptr_get_child(node_ptr, index, offsets, &mtr);
    page = buf_block_get_frame(block);
    /* For R-Tree, since record order might not be the same as
    linked index page in the lower level, we need to traverse
    backwards to get the first page rec in this level. ... */
    if (dict_index_is_spatial(index)) {
      left_page_no = btr_page_get_prev(page, &mtr);
      while (left_page_no != FIL_NULL) {        // ★ R-tree：沿 FIL_PAGE_PREV 回溯到最左页
        /* To obey latch order of tree blocks, we should release the
        right_block once to obtain lock of the uncle block. */
        mtr_release_block_at_savepoint(&mtr, savepoint2, block);
        savepoint2 = mtr_set_savepoint(&mtr);
        block = btr_block_get(left_page_id, ..., RW_SX_LATCH, ..., &mtr);
        page = buf_block_get_frame(block);
        left_page_no = btr_page_get_prev(page, &mtr);
      }
    }
  }
  /* Now we are on the desired level. Loop through the pages on that level. */
  if (level == 0) {
    /* Leaf pages are managed in their own file segment. */
    seg -= PAGE_BTR_SEG_TOP - PAGE_BTR_SEG_LEAF;   // 叶层校验用 LEAF 段头判断 free
  }
```

**阶段二：逐页校验链（loop 标签，每页一套）**：

```cpp
loop:
  mem_heap_empty(heap);
  offsets = offsets2 = nullptr;
  /* index->lock 再次 X/SX（每页重新获取——mtr commit 后索引锁已放）*/
  ...
  ut_a(block->page.id.space() == index->space);

  if (fseg_page_is_free(seg, ...)) {
    btr_validate_report1(index, level, block);
    ib::warn(ER_IB_MSG_40) << "Page is marked as free";
    ret = false;                              // ① 页被段标记 free
  } else if (btr_page_get_index_id(page) != index->id) {
    ib::error(ER_IB_MSG_41) << "Page index id ... != data dictionary index id ...";
    ret = false;                              // ② 页头 PAGE_INDEX_ID 与字典不一致
  } else if (!page_validate(page, index)) {
    btr_validate_report1(index, level, block);
    ret = false;                              // ③ 页内结构（目录槽/堆/记录格式）
  } else if (level == 0 && !btr_index_page_validate(block, index)) {
    /* We are on level 0. Check that the records have the right
    number of fields, and field lengths are right. */
    ret = false;                              // ④ 叶层记录字段数/字段长度
  }

  ut_a(btr_page_get_level(page) == level);    // 页的 PAGE_LEVEL 必须等于目标层
  ut_a(!page_is_empty(page) ||
       (level == 0 && page_get_page_no(page) == dict_index_get_page(index)));
                                              // ★ 只有"根页兼叶页"允许空页
  right_page_no = btr_page_get_next(page, &mtr);
  left_page_no = btr_page_get_prev(page, &mtr);

  if (right_page_no != FIL_NULL) {            // 有右兄弟：校验兄弟链 + 跨页键序
    savepoint = mtr_set_savepoint(&mtr);
    right_block = btr_block_get(..., RW_SX_LATCH, ..., &mtr);
    right_page = buf_block_get_frame(right_block);

    if (btr_page_get_prev(right_page, &mtr) != page_get_page_no(page)) {
      fputs("InnoDB: broken FIL_PAGE_NEXT or FIL_PAGE_PREV links\n", stderr);
      ret = false;                            // ⑤ 右页的 FIL_PAGE_PREV 必须指回本页
    }
    if (page_is_comp(right_page) != page_is_comp(page)) {
      fputs("InnoDB: 'compact' flag mismatch\n", stderr);
      ret = false;
      goto node_ptr_fails;                    // ⑥ compact 标志必须一致（格式不同无法比较）
    }
    /* ★ 跨页键序：本页最大键（supremum 前）必须 < 右页最小键（infimum 后）*/
    rec = page_rec_get_prev(page_get_supremum_rec(page));
    right_rec = page_rec_get_next(page_get_infimum_rec(right_page));
    ... offsets = rec_get_offsets(rec, ...); offsets2 = rec_get_offsets(right_rec, ...);
    /* For spatial index, we cannot guarantee the key ordering across pages,
    so skip the record compare verification for now. ... */
    if (!dict_index_is_spatial(index) &&
        cmp_rec_rec(rec, right_rec, offsets, offsets2, index, false) >= 0) {
      fputs("InnoDB: records in wrong order on adjacent pages\n", stderr);
      rec_print(...); rec_print(...);         // 把两条记录打出来定位
      ret = false;                            // ⑦ >= 0 即本页最大键 ≥ 右页最小键 → 乱序
    }
  }

  if (level > 0 && left_page_no == FIL_NULL) {
    ut_a(REC_INFO_MIN_REC_FLAG &              // ⑧ 最左非叶页首记录带 MIN_REC 标记
         rec_get_info_bits(page_rec_get_next(page_get_infimum_rec(page)), ...));
  }

  /* Similarly skip the father node check for spatial index for now, ...
  We will add special validation mechanism for R-tree later (WL #7520) */
  if (!dict_index_is_spatial(index) &&
      block->page.id.page_no() != dict_index_get_page(index)) {
    /* Check father node pointers */
    /* ★ 从本页首记录定位父页 node_ptr：两次 btr_page_get_father_node_ptr_for_validate
       （首记录一次、supremum 前一次），校验父指针的两个方向 */
    btr_cur_position(index, page_rec_get_next(page_get_infimum_rec(page)), block, &node_cur);
    offsets = btr_page_get_father_node_ptr_for_validate(offsets, heap, &node_cur, ..., &mtr);
    father_page = btr_cur_get_page(&node_cur);
    node_ptr = btr_cur_get_rec(&node_cur);
    parent_page_no = page_get_page_no(father_page);
    parent_right_page_no = btr_page_get_next(father_page, &mtr);
    rightmost_child = page_rec_is_supremum(page_rec_get_next(node_ptr));

    btr_cur_position(index, page_rec_get_prev(page_get_supremum_rec(page)), block, &node_cur);
    offsets = btr_page_get_father_node_ptr_for_validate(offsets, heap, &node_cur, ..., &mtr);

    if (node_ptr != btr_cur_get_rec(&node_cur) ||
        btr_node_ptr_get_child_page_no(node_ptr, offsets) != block->page.id.page_no()) {
      fputs("InnoDB: node pointer to the page is wrong\n", stderr);
      ret = false;                            // ⑨ 父页里指向本页的 node_ptr 页号必须正确
      goto node_ptr_fails;
    }
    if (!page_is_leaf(page)) {
      node_ptr_tuple = dict_index_build_node_ptr(index, page_rec_get_next(page_get_infimum_rec(page)), ...);
      if (cmp_dtuple_rec(node_ptr_tuple, node_ptr, index, offsets)) {
        ib::error(ER_IB_MSG_42) << "Node ptrs differ on levels > 0";
        ret = false;                          // ⑩ 非叶层：node_ptr 元组 == 本页首记录构建的元组
        goto node_ptr_fails;
      }
    }
    if (left_page_no == FIL_NULL) {           // 最左页：node_ptr 是父页 infimum 后第一条
      ut_a(node_ptr == page_rec_get_next(page_get_infimum_rec(father_page)));
      ut_a(btr_page_get_prev(father_page, &mtr) == FIL_NULL);
    }
    if (right_page_no == FIL_NULL) {          // 最右页：node_ptr 是父页 supremum 前最后一条
      ut_a(node_ptr == page_rec_get_prev(page_get_supremum_rec(father_page)));
      ut_a(btr_page_get_next(father_page, &mtr) == FIL_NULL);
    } else {
      /* 右兄弟的父指针也要校验：right_node_ptr 与父页里下一条 node_ptr 一致；
         右页是最右子（rightmost_child）时父页要翻页——先放右块锁再拿父右页锁，
         保持"自顶向下、自左向右"的 latch 顺序（!lockout 即 SX 锁下）*/
      right_node_ptr = page_rec_get_next(node_ptr);
      if (!lockout && rightmost_child) {
        mtr_release_block_at_savepoint(&mtr, savepoint, right_block);
        btr_block_get(parent_right_page_no, ..., RW_SX_LATCH, ...);
        right_block = btr_block_get(right_page_no, ..., RW_SX_LATCH, ...);
      }
      ... /* 校验 right_node_ptr 链（三种情况各报 "node pointer N to the right page is wrong"）*/
    }
  }

node_ptr_fails:
  /* Commit the mini-transaction to release the latch on 'page'.
  Re-acquire the latch on right_page, which will become 'page'
  on the next loop.  The page has already been checked. */
  mtr_commit(&mtr);                           // ★ 逐页一个 mtr：本页校验完即放锁

  if (trx_is_interrupted(trx)) {
    /* On interrupt, return the current status. */   // 被 kill：立即返回当前结论
  } else if (right_page_no != FIL_NULL) {
    mtr_start(&mtr);
    /* lockout=false 时按 parent_page_no/parent_right_page_no 预拿父页锁
       （保持 latch 顺序），再拿右页——右页成为下一轮的 'page' */
    block = btr_block_get(right_page_no, ..., RW_SX_LATCH, ...);
    page = buf_block_get_frame(block);
    goto loop;                                // ★ 沿页链右移，直到无右兄弟
  }
  mem_heap_free(heap);
  return (ret);
}
```

**三个并发/结构设计点**（骨架里被压缩掉的要点）：

1. **逐页一个 mtr**：每页校验完 `mtr_commit` 释放本页 SX 锁，下一轮重新拿右页——校验不持有整层锁链，避免长事务式 latch 累积；页链稳定性靠外层 `index->lock` 的 X/SX（冻结 SMO）。
2. **latch 顺序守则**：拿父页/父右页/右兄弟时先 `mtr_release_block_at_savepoint` 放掉旧块，再按"自顶向下、自左向右"拿新块——与搜索路径的 latch coupling 同一守则（校验在 `lockout=false` 的 SX 锁下更要遵守，否则与并发 SMO 死锁）。
3. **spatial 的三处跳过**（源码注释原文）：跨页键序（页链序 ≠ 键序）、父指针检查（"Search parent from root is very costly for R-tree"）、最左页判定——R-tree 专用校验算法留给 `btr_validate_spatial_index`（注释指向 WL#7520）。

**五个被调子函数族**（btr_validate_level 的主体依赖它们）：

1. **父指针定位 `btr_page_get_father_node_ptr_for_validate` 是薄封装**（`btr0btr.cc:718`）——真实现 `btr_page_get_father_node_ptr_func`（:638）四步：① `dict_index_build_node_ptr` 用子页记录构造"node_ptr 搜索元组"；② `btr_cur_search_to_nth_level(index, level+1, ..., PAGE_CUR_LE)` 向上搜到父层；③ 校验定位到的记录带 `REC_STATUS_NODE_PTR` 状态位；④ **child page_no 与子页号不符 → `ib::fatal` 强制崩溃**（"dump + drop + reimport the table"）。**校验版与运行时版的唯一差异是 `latch_mode`**：校验传 `BTR_CONT_SEARCH_TREE`（父页 S-latch，只读）、运行时传 `BTR_CONT_MODIFY_TREE`（父页 X-latch）——共用同一函数体，无"不持锁"的独立版本。

2. **报告函数 `btr_validate_report1/2`**（:4003/4018）：纯日志输出到 `ib::error`（`ER_IB_MSG_1225/1226`），本身不校验——report1 报单页（page_no + index + table + 非叶层附 level），report2 报**两页之间**（多一个 block 参数，用于"右兄弟左指针不指回"等跨页场景）。

3. **页内深度校验 `page_validate`**（`page0page.cc:2126`）十步：compact 标志一致 → `page_simple_validate_new/old` 浅校验（堆/目录不重叠、owned 计数、n_recs/n_heap 对账）→ `PAGE_MAX_TRX_ID` 越界 → 分配 **UNIV_PAGE_SIZE bitmap 做记录字节重叠检测**（"Record overlaps another"）→ 逐条 `page_rec_validate` + node_ptr 标志 → MIN_REC_FLAG 位置 → **记录升序校验 `cmp_rec_rec`**（spatial 非叶/多值放行相等）→ data_size 累加 → 目录槽 owned 复核 → n_slots/n_recs/data_size 对账 → free list 校验。**深 vs 浅**：浅校验不依赖索引定义（对垃圾页也 robust），深校验额外做键序/node_ptr 标志/min_rec/字节重叠等**索引语义**检查。

4. **叶层字段校验 `btr_index_page_validate`**（`btr0btr.cc:3957`）：整页遍历壳，逐条调 `btr_index_rec_validate`（:3827）——ibuf 索引直接放行（change buffer 记录来自任意索引）；校验 compact 标志一致、**字段数**（REDUNDANT 格式 `rec_get_n_fields_old == n`，唯一例外旧 SYS_INDEXES 表允许 n-1）、**字段长度**（定长字段 `len == fixed_size`、前缀索引 `len <= prefix_len`，DATA_POINT 在 R-tree 索引有特殊长度）。失败可 `dump_on_error` 打十六进制。

5. **R-tree 专用 `btr_validate_spatial_index`**（:4473）：**强制 `mtr_x_lock`**（`lockout=true` 硬编码——R-tree 校验期间不允许并发分裂），逐层 `btr_validate_level(index, trx, n-i, true)`。为什么独立（WL#7520）：R-tree **页链序 ≠ MBR 键序**（普通 B-tree 校验假设页链按键序）、父指针检查昂贵（父节点存**包含所有子 MBR 的最小包围盒**而非最小键）——真正的 MBR 包含关系校验在 `page_validate` 的 spatial 非叶分支（`page_is_spatial_non_leaf` 比较参数 + `rtr_equal_nodeptrs` 允许相等）。

**五个调用点**（CHECK TABLE 只是其中之一）：`ha_innobase.cc:18229`（CHECK TABLE）、`ddl0loader.cc:519`（DDL 并行构建后自检）、`row0import.cc:4322`（IMPORT TABLESPACE 后校验）、`row0upd.cc:2122/2407`（`ut_a` debug 断言，update 路径）、`btr_print_index` 末尾（打印后自校验）。

## 行级扫描器：`row_scan_index_for_mysql`（row0mysql.cc:4586，完整贴码）

**整体职责**：CHECK TABLE 的行级检查核心——非锁读整索引扫描 + 计数 + 相邻记录判序/查重复。输入 `(prebuilt, index, max_threads, check_keys, *n_rows)`，`check_keys=true` 是 CHECK 模式（false 是 COUNT(*) 模式，跳过全部键比较）。**第一步先判断能不能并行**（聚簇索引 + 多线程时走 `parallel_check_table`），不能才落进下面的单线程 loop。

### 入口：跳过判定 + 并行分派（4586-4641）

```cpp
dberr_t row_scan_index_for_mysql(row_prebuilt_t *prebuilt, dict_index_t *index,
                                 size_t max_threads, bool check_keys, ulint *n_rows) {
  *n_rows = 0;
  /* Don't support RTree Leaf level scan */
  ut_ad(!dict_index_is_spatial(index));

  if (index->is_clustered()) {
    /* The clustered index of a table is always available.
    During online ALTER TABLE that rebuilds the table, the clustered index
    in the old table will have index->online_log pointing to the new table. ... */
  } else if (dict_index_is_online_ddl(index) || (index->type & DICT_FTS)) {
    /* Full Text index are implemented by auxiliary tables, not the B-tree.
    We also skip secondary indexes that are being created online. */
    return (DB_SUCCESS);                     // ① 在线 DDL 二级 / FTS：不扫（与 btr_validate 同款跳过）
  }

  if (prebuilt->trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&   // ② 非脏读（一致读）
      prebuilt->select_lock_type == LOCK_NONE &&                     // ③ 非锁读
      index->is_clustered() &&                                       // ④ 聚簇索引（二级索引按叶链走、可能回表，切分无意义）
      (check_keys || prebuilt->trx->mysql_n_tables_locked == 0) &&   // ⑤ 无 LOCK TABLES
      !prebuilt->ins_sel_stmt) {                                     // ⑥ 非 INSERT..SELECT
    auto n_threads = Parallel_reader::available_threads(max_threads, false);
    if (n_threads > 1) {
      /* No INSERT INTO ... SELECT and non-locking selects only. */
      trx_assign_read_view(prebuilt->trx);    // 并行线程共享一个 read view
      if (!check_keys) {
        return row_mysql_parallel_select_count_star(trx, indexes, n_threads, n_rows);
      }
      return parallel_check_table(trx, index, n_threads, n_rows);   // ★ check 走这里
    } else if (n_threads == 1) {
      Parallel_reader::release_threads(n_threads);   // 单线程可用：落回串行 loop
    }
  }
```

即：**check 的并行只发生在聚簇索引上**，且 `innodb_parallel_read_threads > 1`；否则落回下面的串行 loop。

### 单线程扫描 loop（4646-4778，完整）

```cpp
  bool contains_null;
  rec_t *rec = nullptr;
  ulint matched_fields;
  dtuple_t *prev_entry = nullptr;            // 上一条记录转成的 dtuple（相邻比较的基准）
  ...
  ulint cnt = 1000;
  ulint bufsize = std::max(UNIV_PAGE_SIZE, prebuilt->mysql_row_len);
  auto buf = ut::malloc_withkey(..., bufsize);   // dummy template 的输出缓冲
  auto heap = mem_heap_create(100, ...);

  auto ret = row_search_for_mysql(buf, PAGE_CUR_G, prebuilt, 0, 0);  // 定位首条

loop:
  /* Check thd->killed every 1,000 scanned rows */
  if (--cnt == 0) {
    if (trx_is_interrupted(prebuilt->trx)) { ret = DB_INTERRUPTED; goto func_exit; }
    cnt = 1000;                              // 每 1000 行查一次 kill（成本/响应折中）
  }

  switch (ret) {
    case DB_SUCCESS:
      break;
    case DB_DEADLOCK:
    case DB_LOCK_TABLE_FULL:
    case DB_LOCK_WAIT_TIMEOUT:
    case DB_INTERRUPTED:
      goto func_exit;                        // 真错误：直接退出（不吞）
    default: {
      const char *doing = check_keys ? "CHECK TABLE" : "COUNT(*)";
      ib::warn(ER_IB_MSG_1005) << doing << " on index " << index->name
                               << " of table " << index->table->name
                               << " returned " << ret;
    }
      /* fall through (this error is ignored by CHECK TABLE) */
      [[fallthrough]];
    case DB_END_OF_INDEX:
      ret = DB_SUCCESS;                      // ★ 扫描正常结束（见下方"行为不对称"）
    func_exit:
      ut::free(buf);
      mem_heap_free(heap);
      return (ret);
  }

  *n_rows = *n_rows + 1;                     // 计数（check 与 COUNT(*) 都要）

  if (!check_keys) {
    goto next_rec;                           // COUNT(*) 模式：跳过键比较
  }
  /* else this code is doing handler::check() for CHECK TABLE */

  /* row_search... returns the index record in buf, record origin offset
  within buf stored in the first 4 bytes, because we have built a dummy template */
  rec = buf + mach_read_from_4(buf);         // ★ dummy template 取 rec（见下）

  offsets = rec_get_offsets(rec, index, offsets_, ULINT_UNDEFINED, ..., &heap);

  if (prev_entry != nullptr) {               // 从第二条起做相邻比较
    matched_fields = 0;
    auto cmp = prev_entry->compare(rec, index, offsets, &matched_fields);
    contains_null = false;
    /* In a unique secondary index we allow equal key values if
    they contain SQL NULLs */
    const auto n_ordering = dict_index_get_n_ordering_defined_by_user(index);
    for (ulint i = 0; i < n_ordering; ++i) {
      if (UNIV_SQL_NULL == dfield_get_len(dtuple_get_nth_field(prev_entry, i))) {
        contains_null = true; break;
      }
    }
    const char *msg;
    if (cmp > 0) {                           // ★ 升序索引里相邻记录逆序 → 乱序
      ret = DB_INDEX_CORRUPT;
      msg = "index records in a wrong order in ";
    not_ok:
      ib::error(ER_IB_MSG_1006) << msg << index->name << " of table "
                                << index->table->name << ": " << *prev_entry
                                << ", " << rec_offsets_print(rec, offsets);
      /* Continue reading */                 // 打 error log 后继续扫（ret 保持错误码）
    } else if (dict_index_is_unique(index) && !contains_null &&
               matched_fields >= n_ordering) {
      ret = DB_DUPLICATE_KEY;                // ★ 全排序键相等 → 重复键
      msg = "duplicate key in ";
      goto not_ok;
    }
  }

  {
    /* Empty the heap on each round.  But preserve offsets[] for the
    row_rec_to_index_entry() call, by copying them into a separate
    memory heap when needed. */
    if (UNIV_UNLIKELY(offsets != offsets_)) {   // offsets 溢出小栈缓冲时单独拷贝
      ... tmp_heap = mem_heap_create(rec_offs_get_n_alloc(offsets) * sizeof *offsets, ...);
      offsets = mem_heap_dup(tmp_heap, offsets, size);
    }
    mem_heap_empty(heap);                    // ★ 每轮清 heap（prev_entry 在上轮 heap 里，
                                             //   清零前先把 offsets 拷走——生命周期管理）
    prev_entry = row_rec_to_index_entry(rec, index, offsets, heap);  // 本条转 dtuple 供下轮比较
    if (UNIV_LIKELY_NULL(tmp_heap)) mem_heap_free(tmp_heap);
  }

next_rec:
  ret = row_search_for_mysql(buf, PAGE_CUR_G, prebuilt, 0, ROW_SEL_NEXT);  // 取下一条
  goto loop;
}
```

**heap 生命周期技巧**（每轮结尾）：`prev_entry` 存活在 `heap` 上，下一轮 `mem_heap_empty(heap)` 会销毁它——但 `row_rec_to_index_entry` 需要上一轮的 `offsets`（可能指向 heap），所以先检查 `offsets != offsets_`（溢出栈缓冲时）把它 `mem_heap_dup` 到临时 heap 保护起来，再清主 heap。单条记录的 offsets 超过 `REC_OFFS_NORMAL_SIZE`（大记录）才走这条路径。

### ★ 行为不对称：单线程"忽略"、并行"报错"

上一段代码里藏着一个反直觉事实——**单线程扫描发现的乱序/重复键不会报给 `ha_innobase::check`**：

1. 循环内 `not_ok` 设置 `ret = DB_INDEX_CORRUPT`/`DB_DUPLICATE_KEY`，打 `ib::error` 日志后 `/* Continue reading */` 继续；
2. 下一轮 loop 开头 `switch(ret)`：这两个错误码**不在** `DB_SUCCESS/DB_DEADLOCK/...` 的 case 列表里 → 走 `default` → 打 warning → `[[fallthrough]]` 到 `DB_END_OF_INDEX` → **`ret = DB_SUCCESS` 直接返回**；
3. 注释原文明说："fall through (**this error is ignored by CHECK TABLE**)"。

**后果**：`innodb_parallel_read_threads` 决定 CHECK TABLE 的行为——**>1 走并行路径，`parallel_check_table` 汇总 `n_dups>0 → DB_DUPLICATE_KEY`、`n_corrupt>0 → DB_INDEX_CORRUPT` 正常返回错误 → `ha_innobase::check` 落 `dict_set_corrupted`；=1 走串行路径，乱序/重复键只写 error log、结果 `HA_ADMIN_OK`**。即：同一张坏表，默认配置（并行）报 Corrupt，关闭并行后显示 OK——这解释了"CHECK TABLE 结果随 `innodb_parallel_read_threads` 变化"的现象。

**验证结论（逐行核对）**：① 串行 loop 里能 `goto func_exit` 提前返回的只有 `DB_DEADLOCK/DB_LOCK_TABLE_FULL/DB_LOCK_WAIT_TIMEOUT/DB_INTERRUPTED`（+每 1000 行的 kill 探测）——`DB_INDEX_CORRUPT`/`DB_DUPLICATE_KEY` **100% 落 default 被吞**，不存在漏看的其他路径；② `ha_innobase::check` 的 `ret != DB_SUCCESS → dict_set_corrupted` 在串行模式下对乱序/重复键**永不触发**。不对称行为确凿成立。

**第二重不对称**：并行分派条件里有硬条件 `index->is_clustered()`（`row0pread.h:102` 注释 "Secondary index scans are not supported currently"）——**二级索引永远走串行 loop**。所以默认配置下：聚簇索引损坏 → 报错落标；**二级索引同性质的乱序/重复键损坏 → 同样被忽略、不落标**（二级索引触发 `dict_set_corrupted` 只剩两条路：`btr_validate_index` 结构校验失败和行数不一致比对）。

**历史溯源**：串行分支用旧错误码 `ER_IB_MSG_1005/1006` 且注释 "this error is ignored by CHECK TABLE"——说明"忽略"是**并行特性引入之前的既有行为**；并行版是后来新增（新错误码 `ER_IB_ERR_INDEX_RECORDS_WRONG_ORDER/ER_IB_ERR_FOUND_N_*`、汇总时把 counter 转成错误返回）——两套逻辑并存导致语义不对称。源码内无 TODO/FIXME 提及，git 历史已被重写无法追溯原始 commit（本仓库 git log 只到 licence 更新），官方 bug 条目无源码级线索。

### 取行语义：`n_rows` 是 MVCC 可见口径，不是物理记录数

**LOCK_NONE → consistent read**：`row_search_for_mysql` 在 `select_lock_type == LOCK_NONE` 时走一致读路径（`row0sel.cc:5344` 起）：隔离级别为 READ UNCOMMITTED 时直接读最新版本，否则聚簇记录经 `lock_clust_rec_cons_read_sees` 判定、不可见则 `row_sel_build_prev_vers_for_mysql` 沿 undo 回溯到 read view 可见旧版本（旧版本不存在 → `goto next_rec` 跳过不计数）。

**delete-marked 在扫描层跳过**（`row0sel.cc:5431`）：MVCC 判定之后统一 `rec_get_deleted_flag → goto next_rec`——不进入相邻比较、不计入 `n_rows`。

**二级索引的可见性**（`row0sel.cc:5385-5412`）：即使 `need_to_access_clustered=false`（CHECK 设的），二级 rec 仍先经 `lock_sec_rec_cons_read_sees` 判定——不可见则 `goto requires_clust_rec` 回聚簇沿版本链确认（`row_sel_get_clust_rec_for_mysql`），最终不可见/delete-marked 都跳过。**二级扫描口径与聚簇一致：read view 可见的活记录**。

**read view 赋值点**：串行路径在 `row_search_mvcc` 内 `trx_assign_read_view`（`row0sel.cc:4861`，`sql_stat_start==true` 时）；并行路径在 `row_scan_index_for_mysql` 入口 `trx_assign_read_view`（`row0mysql.cc:4621`）——**并行 worker 共享同一个 read view**。生命周期覆盖整个 CHECK 语句。

**计数口径的后果**：`ha_innobase::check` 的 "Index contains N entries, should be M" 里 N/M 分别是二级/聚簇的**可见活记录计数**（口径一致，正常恒等）；结构校验 `btr_validate_level` **不做行计数、不参与该比对**。计数差异只在"二级项与聚簇行在 MVCC 语义下不同步"时出现（如 purge 竞态清理了聚簇 delete-mark 但二级项残留、或崩溃恢复下 delete-mark 状态不一致）——这正是该比对捕获"二级索引损坏"的设计意图；单纯的 delete-marked 堆积未 purge 时两侧都一致跳过，**不会误报**。

### 相邻比较的精确语义：比较对象是"可见相邻"，降序方向内建

**`prev_entry->compare` 比较的字段数**：`row_rec_to_index_entry_low` 里 `dtuple_set_n_fields_cmp(entry, dict_index_get_n_unique_in_tree(index))`（`row0row.cc:590`）——即 **n_unique_in_tree**（二级索引 = 用户列 + 追加 PK，聚簇 = 主键列），不是 n_fields。所以两个二级项即使用户列相同，只要追加 PK 不同，compare 也不等（这是正确口径：树中位置由 n_unique_in_tree 决定）。

**降序索引的判乱序方向正确**（`rem0cmp.cc` 确认）：`cmp_data` 的第三个参数 `is_asc = index->get_field(i)->is_ascending`，所有非零返回值统一 `return (is_asc ? cmp : -cmp)`——**方向反转内建在 compare 返回值里**。降序字段物理上大值在前，`prev`（前一条）应 ≥ `rec`（后一条），compare 因 is_asc=false 反转后返回正值——所以 `cmp > 0` 判乱序对升序/降序**一致成立**，混排索引（有的升有的降）也正确（值相等时无论升降序都返回 0，不影响 `matched_fields` 计数）。

**判重阈值 `n_ordering = n_user_defined_cols`**（`dict0dict.ic:746`）：只数用户显式列，不含追加 PK——"唯一二级索引上两个不同 PK 同用户键的损坏记录"比较时 `matched_fields` 停在 `n_ordering`（第一个 PK 字段处不等），仍 ≥ `n_ordering` → 判重，正确。

**部分 NULL 的边界**：多列唯一索引 (a,b) 上两条 (1,NULL)：compare 对 NULL 与 NULL 返回 0（`cmp_data` 的 `len1==len2==UNIV_SQL_NULL → 0`），`matched_fields==2==n_ordering`，但 `contains_null`（遍历 prev 前 n_ordering 字段发现 NULL）→ 放行不判重——符合 SQL "NULL != NULL"。降序列的 NULL 排序方向（is_asc=false 时 NULL 排最后）由 cmp_data 内建处理，与判重无交互。

**"物理相邻 vs 可见相邻"不误报**：串行路径（`row_search_mvcc` 的 delete-marked 跳过）与并行路径（`Scan_ctx::check_visibility`，`row0pread.cc:511-515` 跳过 delete-marked、聚簇先做 MVCC 判定）都在**扫描层过滤**，callback/循环体收不到 delete-marked 记录——`prev_entry` 保存的永远是上一条**可见**记录，delete-marked 键序残留不会参与比较。

### 转换核心：`row_rec_to_index_entry_low`（row0row.cc:564，完整）

**整体职责**：把物理 rec 转成"typed dtuple"——dtuple 的每个 dfield **零拷贝指向 rec 内部字节**，类型信息从索引定义拷贝。`row_rec_to_index_entry`（外层，:622）先 `rec_copy` 整条 rec 到 heap 再调 `_low`——**拷贝的原因**：rec 指向 `row_search_for_mysql` 复用的 buf，下一轮会被覆盖，`prev_entry` 必须指向独立副本才能跨轮保留。

```cpp
dtuple_t *row_rec_to_index_entry_low(const rec_t *rec, const dict_index_t *index,
                                     const ulint *offsets, mem_heap_t *heap) {
  /* 注意：这里不能 rec_offs_validate——row0ddl.cc 可能用不同 header 格式调用 */
  rec_len = rec_offs_n_fields(offsets);              // = 物理字段数（含系统列）

  entry = dtuple_create(heap, rec_len);
  dtuple_set_n_fields_cmp(entry, dict_index_get_n_unique_in_tree(index));
  /* ★ n_fields_cmp = 树内唯一键字段数：compare 循环的上界——系统列不参与键序 */
  ut_ad(rec_len == dict_index_get_n_fields(index) ||
        rec_len == dict_index_get_n_unique(index) + 1 ||        // 非叶 node_ptr 记录
        (index->table->id == DICT_INDEXES_ID && rec_len == ... - 1));  // 旧 SYS_INDEXES

  dict_index_copy_types(entry, index, rec_len);     // 类型信息固化进 dtuple（typed tuple 的关键）

  for (i = 0; i < rec_len; i++) {
    dfield = dtuple_get_nth_field(entry, i);
    field = rec_get_nth_field_instant(rec, offsets, i, index, &len);  // ★ instant 感知取值
    dfield_set_data(dfield, field, len);             // 零拷贝：dfield.data 指向 rec 内部
    if (rec_offs_nth_extern(index, offsets, i)) dfield_set_ext(dfield);  // extern 只打标不拷贝
  }
  return (entry);
}
```

**instant 列的取值**（`rec_get_nth_field_instant`，`rem0rec.ic:1015`）：`has_row_versions()` 时先 `get_field_off_pos(n)` 做逻辑→物理列号映射；字段长度标记 `UNIV_SQL_ADD_COL_DEFAULT` → 返回 **`index->get_nth_default(n)` 缓存的默认值**（rec 内没存）；`UNIV_SQL_INSTANT_DROP_COL` → 长度 0 返回 nullptr。

### 两个比较入口：`cmp_rec_rec`（结构校验）vs `dtuple_t::compare`（行级）

check 的"顺序/重复"检测是**两条独立比较链**：

| 维度 | `cmp_rec_rec`（结构校验跨页） | `dtuple_t::compare`（行级相邻） |
|---|---|---|
| 左操作数 | `rec_t *`（本页最后一条，frame 内稳定） | `dtuple_t *prev_entry`（已 rec_copy 到 heap） |
| 类型来源 | 每次循环现取 `index->get_col/get_field` | dtuple 的 dfield type（已固化） |
| 比较上界 | 遇 `n_unique_in_tree` 提前返回 0 | `n_fields_cmp`（构造时设好） |
| MIN_REC 标志 | `cmp_btree_recs` 优化：min rec 恒小于其他 | `REC_INFO_MIN_REC_FLAG` 同样处理 |
| 为什么用这个 | 两记录都在各自 page frame 里稳定驻留，无需转换 | 必须跨轮保留 prev（buf 会被覆盖），dtuple 形态天然适配 |

注意一个易混点：`btr_validate_level:4256` 调用 `cmp_rec_rec(rec, right_rec, offsets, offsets2, index, false)` 的最后一个 `false` 是 **`spatial_index_non_leaf`**（第 5 形参），不是 `nulls_unequal`——后者在 `cmp_rec_rec` 内联封装里硬编码 false（NULL 与 NULL 按 `cmp_data` 语义处理）。

### prev_entry 的生命周期内存细节（串行 vs 并行）

**串行版**（row0mysql.cc:4756-4777）：每轮 `mem_heap_empty(heap)` 释放上一轮 prev_entry 数据 + 溢出 offsets；但**当前轮的 offsets 可能正指向 heap**（字段数 > `REC_OFFS_NORMAL_SIZE=100` 溢出栈缓冲时）——先检查 `offsets != offsets_`，溢出则 `mem_heap_dup` 到临时 `tmp_heap` 保住，再清主 heap，`row_rec_to_index_entry` 用保住的 offsets 重建 prev_entry，最后释放 tmp_heap。栈缓冲场景（`offsets == offsets_`）无需中转。

**并行版**（row0mysql.cc:4472-4539）：每线程独立 `heaps[id]`（4096B）+ `prev_tuples[id]` + `prev_blocks[id]`——**heap 按"块"清不是按"行"清**（`prev_blocks[id] != block` 才 `mem_heap_empty`）：同一 page 内多条 rec 的 prev 累积不释放（省每行清堆开销），跨页才清。offsets 直接传 `nullptr`（无栈缓冲、始终 heap 分配），所以**无需 tmp_heap 中转**（清完立即 `rec_get_offsets` 重算）。

### read view 生命周期与并发正确性闭环

**read view 赋值点与幂等**：串行路径在 `row_search_mvcc` 内 `trx_assign_read_view`（row0sel.cc:4861，`sql_stat_start==true && LOCK_NONE` 时；CHECK 在 18271 置 `sql_stat_start=true`、18278 置 `LOCK_NONE`）；并行路径在 `row_scan_index_for_mysql` 入口（4621）。`trx_assign_read_view` 幂等（"已 active 就不重建"，trx0trx.cc:2364）——**这就是 RR 提升的根基**：CHECK 全程复用同一个 read view，各索引行数口径一致（若 RC 会每次语句重建，多索引计数对不上）。并行 worker 共享同一 `trx_t*` 的 read view 指针（无 clone——read view 创建后内容只读，天然线程安全）。

**三处扫描的可见性口径**：聚簇行级用 `lock_clust_rec_cons_read_sees` + 版本链回溯；二级行级用 `lock_sec_rec_cons_read_sees`（**保守粗筛**：用页头 `PAGE_MAX_TRX_ID` 而非记录 trx_id——二级记录没有 trx_id）；**结构校验不做任何 MVCC 判定**（delete-marked 记录也参与键序/页链校验）——但 delete-mark **不改变键值**，所以不会误报乱序。

**并发正确性闭环**（三个结论 + 证据）：① **SX 锁挡 SMO**——分裂/合并必须持 index X/SX，结构校验的 SX 锁保证页链冻结；② **read view 挡 DML**——行级扫描靠快照隔离并发增删改；③ **SX 锁也挡 purge**——purge 的物理删除先取 index SX 锁（`row0purge.cc:323`）再 `btr_cur_pessimistic_delete`，与结构校验互斥，不会撕裂页链。三者分工：结构校验看物理（锁保证稳定）、行级看可见（快照保证一致）。

**kill 中断的响应差异**：行级每 1000 行查 `trx_is_interrupted`（立即 `goto func_exit`）；结构校验每页链节点 commit mtr 后查（**先报告已发现的损坏再响应中断**）；`ha_innobase::check` 对 `DB_INTERRUPTED` 是 `break`（不 warning、不落标——"could happen during shutdown"），与真损坏区分。

### 并行版：`parallel_check_table`（row0mysql.cc:4432，完整）

```cpp
static dberr_t parallel_check_table(trx_t *trx, dict_index_t *index,
                                    size_t n_threads, ulint *n_rows) {
  ut_a(n_threads > 1);
  using Shards = Counter::Shards<Parallel_reader::MAX_THREADS>;
  Shards n_recs{};        // 每线程独立的分片计数器（无锁累加，避免竞争）
  Shards n_dups{};
  Shards n_corrupt{};
  Counter::clear(n_dups); Counter::clear(n_recs); Counter::clear(n_corrupt);
  ...
  Heaps heaps; Tuples prev_tuples; Blocks prev_blocks;
  for (size_t i = 0; i < n_threads; ++i) {
    heaps.push_back(mem_heap_create(4096, ...));   // 每线程独立 heap/prev_tuple/prev_block
  }

  Parallel_reader reader(n_threads);
  Parallel_reader::Scan_range full_scan;
  Parallel_reader::Config config(full_scan, index);

  auto err = reader.add_scan(trx, config, [&](const Parallel_reader::Ctx *ctx) {
    /* worker() → dequeue() → ctx->traverse() → 对每条 rec 调这个 callback */
    const auto rec = ctx->m_rec;
    const auto block = ctx->m_block;
    const auto id = ctx->thread_id();
    Counter::inc(n_recs, id);

    auto heap = heaps[id];

    if (ctx->m_start) {
      /* Starting scan of a new range. We need to reset the previous tuple
      because we don't know what the value of the previous last tuple was. */
      prev_tuples[id] = nullptr;              // ★ 新 range 起点重置：跨 range 比较会误报乱序
    }
    auto prev_tuple = prev_tuples[id];
    auto offsets = rec_get_offsets(rec, index, nullptr, ..., &heap);

    if (prev_tuple != nullptr) {
      ulint matched_fields = 0;
      auto cmp = prev_tuple->compare(rec, index, offsets, &matched_fields);
      /* 与串行版完全相同的判定：含 NULL 放行、cmp>0 乱序、全排序键相等重复 */
      bool contains_null = false;
      const auto n_ordering = dict_index_get_n_ordering_defined_by_user(index);
      for (size_t i = 0; i < n_ordering; ++i) {
        if (UNIV_SQL_NULL == dfield_get_len(dtuple_get_nth_field(prev_tuple, i))) {
          contains_null = true; break;
        }
      }
      if (cmp > 0) {
        Counter::inc(n_corrupt, id);
        ib::error(ER_IB_ERR_INDEX_RECORDS_WRONG_ORDER) << ...;
        /* Continue reading */                 // ★ 并行版不退出：数完所有分片再汇总
      } else if (dict_index_is_unique(index) && !contains_null &&
                 matched_fields >= dict_index_get_n_ordering_defined_by_user(index)) {
        Counter::inc(n_dups, id);
        ib::error(ER_IB_ERR_INDEX_DUPLICATE_KEY) << ...;
      }
    }

    if (prev_blocks[id] != block || prev_blocks[id] == nullptr) {
      mem_heap_empty(heap);                    // 换页才清 heap（offsets 指向 heap，翻页即失效）
      offsets = rec_get_offsets(rec, index, nullptr, ..., &heap);
      prev_blocks[id] = block;
    }
    prev_tuples[id] = row_rec_to_index_entry(rec, index, offsets, heap);
    return DB_SUCCESS;
  });

  if (err == DB_SUCCESS) { prev_tuples.resize(n_threads); prev_blocks.resize(n_threads); err = reader.run(n_threads); }
  if (err == DB_OUT_OF_RESOURCES) {
    ib::warn(ER_INNODB_OUT_OF_RESOURCES)
        << "Resource not available to create threads for parallel scan."
        << " Falling back to single thread mode.";
    err = reader.run(0);                       // ★ 资源不足：退化为"单线程但走并行代码"
  }
  for (auto heap : heaps) mem_heap_free(heap);

  if (Counter::total(n_dups) > 0) {
    ib::error(ER_IB_ERR_FOUND_N_DUPLICATE_KEYS) << "Found " << Counter::total(n_dups)
                                                << " duplicate rows in " << index->name;
    err = DB_DUPLICATE_KEY;                    // ★ 汇总后才返回错误（与串行的"忽略"相反）
  }
  if (Counter::total(n_corrupt) > 0) {
    ib::error(ER_IB_ERR_FOUND_N_RECORDS_WRONG_ORDER) << "Found " << Counter::total(n_corrupt)
                                                     << " rows in the wrong order in " << index->name;
    err = DB_INDEX_CORRUPT;
  }
  *n_rows = Counter::total(n_recs);
  return err;
}
```

与串行版的差异总结：① **错误处理相反**——串行遇到乱序/重复打日志后"忽略"（返回 `DB_SUCCESS`），并行数完汇总返回错误（触发 `dict_set_corrupted`）；② **heap 管理**——串行每轮清 heap，并行只在**换页**时清（`prev_blocks[id] != block`，因为 offsets/prev_tuple 按页存活）；③ 分片边界重置 `prev_tuples[id]=nullptr` 防跨 range 误报；④ `DB_OUT_OF_RESOURCES` 时 `reader.run(0)` 退化为单线程但**保留并行代码的错误语义**（此时乱序/重复仍会报错）。

**并行机制对接的四个细节**（`row0pread.cc/h`）：

1. **分片算法：按子树切**，不是按固定页链段——`Scan_ctx::partition → create_ranges` 从根广度优先遍历非叶层，每个 node_ptr 生成一个 `Range`（叶子层的持久游标对 `[start, end)`）；`full_scan`（CHECK 用）即整棵树切成"根的孩子数"个子树。树太浅（深度 < `SPLIT_THRESHOLD=3`）不切；`ranges > n_threads` 时多余部分标 `m_split=true` 由 worker 动态再拆（`Ctx::split`）。
2. **`ctx->m_start` 的精确语义**：`traverse_recs` 进入新遍历范围时置 true，**每条回调成功返回后置 false**；因不可见跳过或越界 break 时保持 true——即"当前是新范围的第一条**可见**记录"。callback 用它重置 `prev_tuples[id]=nullptr`，防跨子树误比。
3. **`Counter::Shards<256>`**（`ut0counter.h`）：cache-line 对齐的 `std::atomic` 数组，按 `thread_id` 取模分片 `fetch_add(relaxed)` 无锁累加；`Counter::total` 是非原子快照（注释 "not 100% accurate"）——对 CHECK 的汇总判定足够。
4. **`reader.run(0)` 退化语义**：`DB_OUT_OF_RESOURCES` 时走 `m_sync=true`，不创建 OS 线程、在**当前调用线程**直接跑 `worker(thread_ctx[0])`（thread_id=0）——注意它**保留并行代码的错误语义**（此时乱序/重复键仍会报错落标，与串行 loop 的"忽略"不同）。

### dummy template 的写入点（row0sel.cc:5731-5744）

串行 loop 里 `rec = buf + mach_read_from_4(buf)` 的前提是 `row_search_for_mysql` 在 `ROW_MYSQL_DUMMY_TEMPLATE` 模式下不走 MySQL 行格式转换：

```cpp
    if (UNIV_UNLIKELY(prebuilt->template_type == ROW_MYSQL_DUMMY_TEMPLATE)) {
      /* CHECK TABLE: fetch the row */
      if (result_rec != rec && !prebuilt->need_to_access_clustered) {
        offsets = rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED, ..., &heap);
        result_rec = rec;
      }
      memcpy(buf + 4, result_rec - rec_offs_extra_size(offsets), rec_offs_size(offsets));
      mach_write_to_4(buf, rec_offs_extra_size(offsets) + 4);   // ★ 前 4 字节 = rec 在 buf 内的偏移
    }
```

即：整条 rec（含 extra-size 前缀）原样拷进 buf，记录偏移写前 4 字节——扫描器按 `buf + mach_read_from_4(buf)` 取回原始 rec，跳过 `row_sel_store_mysql_rec` 的列映射开销。`ROW_MYSQL_DUMMY_TEMPLATE` 在 row0sel.cc 有**两处判断点**：`row_search_no_mvcc`（:4170，HANDLER/API 等非 MVCC 扫描用）与 `row_search_mvcc`（:5731，CHECK 走这处）。与 `ROW_MYSQL_WHOLE_ROW` 的核心差异就是"零转换"：不做 MySQL 行格式的字段拷贝/转换，整条物理 rec 直通 handler，由 `ha_innobase::check` 自己 `rec_get_offsets` 解析。

`Parallel_reader` 的分片机制（按子树切区间、`ctx->m_start`、后台线程）→ [`../innodb/parallel_scan.md`](../innodb/parallel_scan.md)「并行 CHECK TABLE」节。

## 结果表与 Msg_type 聚合

四列在 `mysql_admin_table` 开头构造（`:728-741`：Table/Op/Msg_type/Msg_text，`Msg_text` 最大 `SQL_ADMIN_MSG_TEXT_SIZE=128K`）。`Msg_type` 三路产生：① **警告聚合** `send_analyze_table_errors`（`:329`）——把引擎 `push_warning` 收集的每条条件逐行 dump；② 主状态行 `switch(result_code)`（`:1175-1446`）；③ 早期失败行内联写（open 失败/view/只读表等）。

**警告聚合 `send_analyze_table_errors`**（sql_admin.cc:329-347，完整）：

```cpp
static bool send_analyze_table_errors(THD *thd, const char *operator_name,
                                      const char *table_name) {
  Diagnostics_area::Sql_condition_iterator it = thd->get_stmt_da()->sql_conditions();
  const Sql_condition *err;
  Protocol *protocol = thd->get_protocol();
  while ((err = it++)) {                              // 遍历 DA 里每条 condition
    protocol->start_row();
    protocol->store(table_name, system_charset_info);                          // Table
    protocol->store(operator_name, system_charset_info);                       // Op
    protocol->store_string(warning_level_names[err->severity()].str,           // Msg_type
                           warning_level_names[err->severity()].length, ...);
    protocol->store(err->message_text(), system_charset_info);                 // Msg_text
    if (protocol->end_row()) return true;
  }
  thd->get_stmt_da()->reset_condition_info(thd);      // 发完清空 condition 区
  return false;
}
```

- condition 来源是 `thd->get_stmt_da()->sql_conditions()`——引擎 `push_warning` 追加进语句 DA 的每条 condition（**不是** `my_message`：push_warning 追加列表不影响主错误位，my_message 设主错误）；
- `warning_level_names[]`（sql_error.cc:722-725）= `{"Note", "Warning", "Error", "?"}`，下标即 `Sql_condition::enum_severity_level`——注意这里**首字母大写**，与 switch 里的小写 "note"/"status"/"error" 不一致（8.0 既存现象）。

**主状态行 switch(result_code)**（:1175-1446）完整映射：

| 返回码 | Msg_type | Msg_text | fatal_error |
|---|---|---|---|
| `HA_ADMIN_NOT_IMPLEMENTED` | note | ER_CHECK_NOT_IMPLEMENTED（含 operator_name） | 否 |
| `HA_ADMIN_NOT_BASE_TABLE` | note | ER_BAD_TABLE_ERROR（含 db.table） | 否 |
| `HA_ADMIN_OK` | status | "OK" | 否 |
| `HA_ADMIN_FAILED` | status | "Operation failed" | 否 |
| `HA_ADMIN_REJECT` | status | "Operation need committed state"（并置 open_for_modify=false） | 否 |
| `HA_ADMIN_ALREADY_DONE` | status | "Table is already up to date" | 否 |
| `HA_ADMIN_CORRUPT` | **error** | "Corrupt" | **是** |
| `HA_ADMIN_INVALID` | error | "Invalid argument" | 否 |
| `HA_ADMIN_TRY_ALTER` | note | recreate+analyze 子流程（唯一 `goto send_result_message` 回跳分支） | 否 |
| `HA_ADMIN_WRONG_CHECKSUM` | note | ER_VIEW_CHECKSUM | 否 |
| `HA_ADMIN_NEEDS_UPGRADE/NEEDS_ALTER` | error | ER_TABLE_NEEDS_UPGRADE（可修复）/ER_TABLE_NEEDS_REBUILD（不可修复，按 `HA_CAN_REPAIR` 分） | **是** |
| `HA_ADMIN_STATS_UPD_ERR` | status | "Unable to write table statistics to DD tables" | 否 |
| `HA_ADMIN_NEEDS_DUMP_UPGRADE` | error | "Table upgrade required... dump/reload"（前有 `assert(false)`，8.0 正常到不了） | **是** |
| default | error | "Unknown - internal error %d during operation" | **是** |

`fatal_error=true` 的四个（CORRUPT/NEEDS_UPGRADE/NEEDS_DUMP_UPGRADE/default）在收尾触发 `tdc_remove_table(TDC_RT_REMOVE_UNUSED)`。即：CHECK TABLE 的"每行一个 warning/error"就是引擎 `push_warning` 的逐条呈现——InnoDB 发现 N 个索引损坏，结果集就有 N+1 行（N 个 error + 1 个主状态行）。

## CHECKSUM TABLE：不在 ADMIN 四兄弟框架里的第五个语句

`CHECKSUM TABLE tbl [QUICK|EXTENDED]` 语义上是"诊断家族"的一员，但**实现上不走 `mysql_admin_table`**——语法归约直接置 `SQLCOM_CHECKSUM`（`sql_yacc.yy:9512`，没有 `Sql_cmd` 类），`sql_parse.cc:3666` 分派到独立入口 `mysql_checksum_table`（`sql_table.cc:18785`）：2 列结果集（Table/Checksum）、**逐表单独打开**（"Open one table after the other to keep lock time as short as possible"）、调用 `t->file->checksum()` 虚函数（`handler.h:5814` 默认返回 0）。

**InnoDB 没有覆写 `checksum()` 也不声明 `HA_HAS_CHECKSUM`**（`ha_innodb.cc:2973-2983` 的 `m_int_table_flags` 不含该标志）——但**不是返回 NULL**，而是走 `mysql_checksum_table` 的四种路径（`sql/sql_table.cc:18841-18915`）：

```cpp
    if (!t) {
      protocol->store_null();                        // ① 表不存在
    } else {
      if (t->file->ha_table_flags() & HA_HAS_CHECKSUM &&
          !(check_opt->flags & T_EXTEND))
        protocol->store((ulonglong)t->file->checksum());   // ② 引擎内置 checksum（InnoDB 不在此列）
      else if (!(t->file->ha_table_flags() & HA_HAS_CHECKSUM) &&
               (check_opt->flags & T_QUICK))
        protocol->store_null();                      // ③ 无内置 checksum 且 QUICK：跳过算 NULL
      else {
        /* ④ 默认/EXTEND：逐行 ha_rnd_next + checksum_crc32 累加（InnoDB 走这里）*/
        ... protocol->store((ulonglong)crc);
      }
    }
```

即 **CHECKSUM TABLE 在 InnoDB 表上默认是逐行全表扫描 + CRC32 累加**（与 CHECK TABLE 一样 O(行数)，且默认档是重操作）；`T_QUICK` 才输出 NULL。`handler::checksum()` 默认返回 0 对 InnoDB **根本不生效**（该虚函数只在 `HA_HAS_CHECKSUM` 分支被调）。另外注意：`mysql_checksum_table` 是**逐表单独打开**（"Open one table after the other to keep lock time as short as possible"，断链开一张处理完再开下一张，`TL_READ` 锁），与 `mysql_admin_table` 一次打开整条表链不同。InnoDB 的完整性由 redo + 页级 checksum（`FIL_PAGE_SPACE_OR_CHKSUM`）保证——表级 CRC32 是逐行算出来的值，与页级校验是两套东西。

## REPAIR TABLE 与 FOR UPGRADE

- **InnoDB 没有 `repair()`**（`ha_innodb.cc` 无 `::repair`，继承默认返回 `HA_ADMIN_NOT_IMPLEMENTED`；默认实现断言 `!(ha_table_flags() & HA_CAN_REPAIR)`——InnoDB 未声明该能力位，`repair()` 永远到不了引擎层）→ REPAIR TABLE 输出 `note` + ER_CHECK_NOT_IMPLEMENTED。
- **InnoDB 的"修复"= 重建**（源码依据）：`ha_innobase::optimize` 返回 `HA_ADMIN_TRY_ALTER`（`ha_innodb.cc:18122-18148`，注释原文 "mapped to ALTER TABLE tablename ENGINE=InnoDB, which rebuilds the table"）→ server `mysql_admin_table` `case HA_ADMIN_TRY_ALTER`（`:1237`）关表 + `mysql_recreate_table` 重建。所以惯例"InnoDB 靠 `ALTER TABLE ... ENGINE=InnoDB` 修复"的源码依据就是这条 TRY_ALTER 路径。
- **FOR UPGRADE**：`sql_flags & TT_FOR_UPGRADE` 时不走普通 operator_func（`mysql_admin_table:1107`），走 `check_for_upgrade`（`sql_admin.cc:635`，完整）：

```cpp
template <typename CHECK_FUNC>
static Check_result check_for_upgrade(THD *thd, dd::String_type &sname,
                                      dd::String_type &tname, CHECK_FUNC &&cf) {
  /* Check_result = pair<bool, int>：first=是否致命（goto err），second=HA_ADMIN_* */
  dd::cache::Dictionary_client *dc = thd->dd_client();
  const dd::Table *t = nullptr;
  if (dc->acquire(sname, tname, &t)) return {true, HA_ADMIN_FAILED};  // DD 拿不到=致命

  if (is_checked_for_upgrade(*t)) {
    /* last_checked_for_upgrade_version_id == MYSQL_VERSION_ID → 已检查过 */
    return {false, HA_ADMIN_ALREADY_DONE};             // 短路："Table is already up to date"
  }
  int result_code = cf();                              // ★ 调引擎 check()（lambda）
  if (result_code != HA_ADMIN_OK && result_code != HA_ADMIN_ALREADY_DONE) {
    return {false, result_code};                       // 检查失败：非致命，走 send_result 输出
  }
  Check_result error{true, result_code};               // 以下 DD 写失败都是致命

  if (acquire_shared_global_read_lock(thd, thd->variables.lock_wait_timeout))
    return error;                                      // ① 全局读锁 GRL
  if (dd::sdi_utils::mdl_lock(thd, MDL_key::SCHEMA, sname, "", MDL_INTENTION_EXCLUSIVE))
    return error;                                      // ② schema MDL IX（为 acquire_for_modification 准备）
  if (dd::sdi_utils::mdl_lock(thd, MDL_key::TABLE, sname, tname))
    return error;                                      // ③ table MDL X（更新版本号需要）
  dd::Table *c = nullptr;
  if (dc->acquire_for_modification(t->id(), &c)) return error;
  c->mark_as_checked_for_upgrade();                    // 写 last_checked_for_upgrade = MYSQL_VERSION_ID
  if (dc->update(c)) return error;                     // 落库 mysql.tables
  return {false, result_code};
}
```

三个要点：① **短路键**是 DD 字段 `last_checked_for_upgrade_version_id == MYSQL_VERSION_ID`（`table.h:465`，初始 0 = 每张表都会被检查一次）；② 通过后才拿**三把锁**（GRL → schema IX → table X，顺序固定）写版本号——**先检查后写标记**，检查失败不写；③ 引擎 `check()` 以 lambda 传入，所以 FOR UPGRADE 的引擎侧仍是完整 CHECK（只是多一层 DD 标记）。InnoDB 未覆写 `check_for_upgrade`（基类返回 0），真正版本判断在 `ha_check` 的 `check_table_for_old_types`（`handler.cc:4718`）。机制全貌见 [`server/dd/dd.md`](../server/dd/dd.md) 升级前体检节。

## 锁与事务语义

| 层 | CHECK 拿什么 |
|---|---|
| MDL | `MDL_SHARED_READ`（`lock_type < TL_WRITE_ALLOW_WRITE` 分支） |
| thr_lock | `TL_READ_NO_INSERT`（thr_lock 层语义；对 InnoDB 无实际约束作用——InnoDB 并发靠引擎内锁） |
| 引擎 trx | InnoDB 自决提 RR + 每索引 SX 锁（`lockout=false`）；server **不传**隔离级别 |

观测：慢日志由 `log_slow_admin_statements` 控制（`enable_slow_log = opt_log_slow_admin_statements`）；PROCESSLIST state 用 `op_info="checking table"`。耗时构成：结构校验 O(页数) 串行 SX；行级扫描 O(记录数) 可并行（`innodb_parallel_read_threads`）。

## 被否掉的方案 / 已知限制

- **InnoDB 不做 repair 是主动选择**（非缺失）：完整性由 redo/MVCC 保证，"修复"在 InnoDB 世界观里就是重建（见设计权衡 1）；
- **CHECK 期间 SX 锁阻塞 SMO**：结构校验加索引 SX 锁，分裂/合并（SMO）会被挡——大表 CHECK 期间写路径的树结构调整排队，但普通插入/更新不受影响（SX 与 S 兼容）；
- **结构校验无法并行**：只有行级扫描走 `Parallel_reader`，`btr_validate_index` 串行单事务；
- **`T_FAST`/`T_CHANGED` 对 InnoDB 无意义**：这两档依赖"未正常关闭"标记，InnoDB 的崩溃安全模型下没有该语义，直接忽略；
- **AHI 校验只在 debug 编译**：release 版 CHECK 不校验自适应哈希索引。

## 常见误解

1. **"CHECK TABLE 会修复表"**——不修，只诊断；InnoDB 的修复路径是 `ALTER TABLE ENGINE=InnoDB` 重建（TRY_ALTER 链）。
2. **"QUICK 是快速档"**——InnoDB 里 QUICK 是"跳过结构校验"（只做行级扫描），比非 QUICK 更浅。
3. **"InnoDB 有 REPAIR TABLE"**——没有，`note` 提示不支持。
4. **"结构校验失败的表会被标记 corrupted"**——不标；只有行级扫描失败/计数不符才 `dict_set_corrupted`（不对称设计）。
5. **"CHECK 会阻塞写"**——server 层 `MDL_SHARED_READ` 不挡 DML；引擎侧 SX 锁只挡 SMO（分裂/合并），普通插入/更新照常。
6. **"dirty read 下也能 CHECK"**——InnoDB 强制提 RR，就是为了记录计数正确。

## 参考

- [`index/btr.md`](../index/btr.md) 树校验节（`btr_validate_index`/`btr_validate_level` 逐项校验 + SX 锁协议 + 打印族）
- [`../index/stats.md`](../index/stats.md)（ANALYZE 引擎侧）、[`index/operations.md`](../index/operations.md)（OPTIMIZE 的真相）
- [`server/dd/dd.md`](../server/dd/dd.md)（升级检查体系 / `upgrade_system_schemas`）
- `sql/sql_admin.cc`（`mysql_admin_table`）、`sql/handler.h`（HA_ADMIN_* / 虚函数契约）、`storage/innobase/handler/ha_innodb.cc:18156`（`ha_innobase::check`）、`storage/innobase/row/row0mysql.cc:4586`（扫描器）
