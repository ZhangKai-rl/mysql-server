# 索引的生命周期与可观测：DDL 与运维

> 基于 MySQL 8.0.39 源码。本篇讲 **索引的"生老病死"**（DDL 生命周期）与**怎么观测它**（`SHOW INDEX` / `I_S` / PFS / sys schema）。
>
> **边界**：批量建页的 `Btree_load` 算法细节见 [`btr.md`](btr.md)；online DDL 的通用框架（row log、DDL log）见 [`../innodb/ddl.md`](../innodb/ddl.md)；统计数值本身怎么算见 [`stats.md`](stats.md)；索引类型见 [`types.md`](types.md)。本篇聚焦**索引对象的生命周期操作**与**运维观测入口**。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 索引的创建
    - [从 `ALTER TABLE` 到 `ddl::Loader` 的完整链路](#从-alter-table-到-ddlloader-的完整链路)
    - [三种算法：COPY / INPLACE / INSTANT 与索引](#三种算法copy--inplace--instant-与索引)
    - [在线建索引的 row log](#在线建索引的-row-log)
    - [并行构建与临时文件](#并行构建与临时文件)
  - 索引的销毁与变更
    - [DROP INDEX：三段式（标记 → 日志 → post_ddl 释放）](#drop-index三段式标记--日志--post_ddl-释放)
    - [RENAME / 可见性 / DISABLE KEYS / 主键增删](#rename--可见性--disable-keys--主键增删)
  - 索引的重建
    - [OPTIMIZE TABLE 的真相](#optimize-table-的真相)
  - 可观测
    - [`SHOW INDEX`：Cardinality 与缓存](#show-indexcardinality-与缓存)
    - [`I_S.STATISTICS` 列与语义](#isstatistics-列与语义)
    - [InnoDB 侧的三张 I_S 表](#innodb-侧的三张-is-表)
    - [PFS 与 sys schema：无用与冗余索引](#pfs-与-sys-schema无用与冗余索引)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引的生命周期 = **创建（建树 + 回填并发 DML）→ 使用（DML 维护）→ 变更/删除（标记 + 延迟释放）→ 重建**。本篇覆盖首尾三段与全程的可观测入口。

### 用途

- 理解 `ADD INDEX` 为什么"online"、什么时候不 online、失败会怎样；
- 理解 `DROP INDEX` 为什么"瞬间返回"却还有后续 IO；
- 定位索引相关问题：基数、空间、是否被使用、是否冗余。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 在线 DDL（`ADD INDEX` 不阻塞 DML）、row log |
| 8.0.12+ | INSTANT 算法（**但索引操作全不支持 INSTANT**） |
| 8.0.27+ | `innodb_ddl_threads` 并行构建索引（默认 4） |
| 8.0.30 | GIPK、`ALTER INDEX ... VISIBLE/INVISIBLE` 相关 DD 支持 |

---

## 理论基础

### 设计思想与权衡

**1. 在线建索引 = 扫描 + 排序 + 批量装载 + 日志回放。** 四条流水并行：一致性读扫描聚簇索引 → 归并排序 → 自底向上批量建页 → 回放扫描期间的并发 DML（row log）。这样 `ADD INDEX` 只在开始和结束短暂持锁。

**2. "延迟释放"是 DDL 原子性的代价。** `DROP INDEX` 不能立即释放 B-tree（否则 DDL 回滚无法恢复），所以走"打标记 → 写 DDL log → 提交后 post_ddl 重放日志真正释放"三段式。

**3. 统计与结构是分离的。** `OPTIMIZE TABLE` 是"重建结构 + 重算统计"两件事；只统计陈旧时 `ANALYZE` 就够，不必重建。

**4. 可观测分三层**：DD 元数据（`I_S.STATISTICS`：索引定义/可见性/表达式）、引擎内存状态（`INNODB_TABLESTATS`：行数/页数/修改计数）、PFS 使用统计（`table_io_waits_summary_by_index_usage`：索引到底被用过没）。三层回答不同问题。

### 理论溯源

- Online index build（Oracle/MySQL 的经典实现）：扫描 + merge + 日志回放三段式；
- ARIES 的"逻辑 undo/redo"思想在 DDL log 上的应用（DDL 也要可回滚）。

---

## 核心实现

### 索引的创建

#### 从 `ALTER TABLE` 到 `ddl::Loader` 的完整链路

```
CREATE INDEX / ALTER TABLE ADD INDEX
  └─ mysql_alter_table()                       sql/sql_table.cc:16295
      ├─ fill_alter_inplace_info()             计算 handler_flags / 索引增删数量
      ├─ open_table_uncached()                 打开"新定义"的 TABLE
      ├─ (handler_flags == 0 → no-op：ADD/DROP 同索引会抵消)
      ├─ check_if_supported_inplace_alter()    → ha_innobase（handler0alter.cc:958）
      └─ mysql_inplace_alter_table()
          ├─ ha_prepare_inplace_alter_table()  → prepare_inplace_alter_table_impl (5407)
          │     ├─ ddl::create_index()         造 dict_index_t（:4897）
          │     ├─ row_log_allocate(二级索引)  （:4934）
          │     └─ row_log_allocate(聚簇)      （:4962，仅 rebuild）
          ├─ ha_inplace_alter_table()          → inplace_alter_table_impl (6105)
          │     └─ ddl::Context::build()       (ddl0ctx.cc:512)
          │          └─ Loader::build_all()    (ddl0loader.cc:484)
          │               ├─ Cursor::scan(m_builders)   并行扫描聚簇索引 → 临时文件
          │               └─ Loader::load()             (290) 并行排序 + 装载
          │                    └─ Builder::btree_build → Btree_load::build（btr0load.cc:1304）
          │                         └─ Builder::finalize → Flush_observer::flush
          │                                              → row_log_apply（回放）
          ├─ wait_while_table_is_used()        升级到 EXCLUSIVE（短暂）
          └─ ha_commit_inplace_alter_table()  → commit_inplace_alter_table_impl (7414)
```

`Loader::load()` 的并行方式（`ddl0loader.cc:290-345`）：

```cpp
const bool sync = m_ctx.m_max_threads <= 1;     // innodb_ddl_threads <= 1 则同步
for (auto builder : m_builders) {
  /* RTrees are built during the scan phase, using row by row insert. */
  if (!builder->is_spatial_index()) { builder->set_next_state(); add_task(Task{builder}); }
}
if (!sync) {
  for (size_t i = 1; i < m_ctx.m_max_threads; ++i) threads.push_back(std::thread{fn, i});
}
auto err = m_taskq->execute();                  // ★ 当前用户线程也参与执行
```

**R-tree 是例外**：空间索引在扫描阶段就逐行插入（无法排序后自底向上构建），且 **redo 不禁用**。

#### 三种算法：COPY / INPLACE / INSTANT 与索引

InnoDB 的 flag 分类（`handler0alter.cc:114-172`）决定了算法：

```cpp
INNOBASE_ONLINE_CREATE  = ADD_INDEX | ADD_UNIQUE_INDEX | ADD_SPATIAL_INDEX;
INNOBASE_ALTER_REBUILD  = ADD_PK_INDEX | DROP_PK_INDEX | CHANGE_CREATE_OPTION | ... | RECREATE_TABLE;
INNOBASE_ALTER_NOREBUILD= INNOBASE_ONLINE_CREATE | 外键操作 | DROP_INDEX | RENAME_INDEX | ...;
INNOBASE_INSTANT_ALLOWED= 全是列操作（改列名/加删虚拟列/加删存储列）;   // ★ 无索引相关 flag
```

| 索引 DDL | INSTANT | INPLACE | 备注 |
|---|---|---|---|
| `ADD INDEX` / `ADD UNIQUE` / `ADD SPATIAL` / `ADD FULLTEXT` | **否** | **是**（online，返回 `HA_ALTER_INPLACE_NO_LOCK`） | |
| `DROP INDEX` / `DROP UNIQUE` | **否** | **是**（`DROP_INDEX` ∈ NOREBUILD） | |
| `RENAME INDEX` | **否** | **是**（纯字典改名，不动数据） | |
| `ALTER INDEX ... VISIBLE/INVISIBLE` | — | 实质 no-op（只改 DD `is_visible`） | **瞬时** |
| `ADD / DROP PRIMARY KEY` | **否** | **是，但触发 rebuild**（`ADD_PK_INDEX` ∈ REBUILD） | 整表重建，所有二级索引一起重建 |
| `ALTER TABLE ... DISABLE/ENABLE KEYS` | — | **InnoDB 不支持**（普通表返回 `HA_ERR_WRONG_COMMAND`） | 只改 DD `keys_disabled`，索引仍被维护 |

**不是 online（`LOCK=NONE`）的例外**：

1. `ADD/DROP PRIMARY KEY` → 走 rebuild（虽仍用 row log，但重建全表）；
2. 同一条语句里混了任何触发 rebuild 的变更；
3. 只加/删外键且 `foreign_key_checks=ON`（`handler0alter.cc:1017` 直接 NOT_SUPPORTED）；
4. 修改 encryption 属性、字段数超限；
5. **表为空时 server 强制降级**（`sql_table.cc:13501`：ONLINE → SHARED，因为空表时不允许并发插入）。

#### 在线建索引的 row log

`row_log_t`（`row0log.cc:184`）是"扫描期间的并发 DML 暂存区"，本质是一个**内存 block + 临时文件**的日志：

```cpp
struct row_log_t {
  ddl::Unique_os_file_descriptor file;   // 溢出到临时文件
  ib_mutex_t mutex;                      // 保护 error / max_trx / tail
  dict_table_t *table;                   // 非 NULL 表示 rebuild（而非建二级索引）
  bool same_pk;                          // rebuild 时 PK 定义是否未变
  dberr_t error;
  trx_id_t max_trx;
  row_log_buf_t tail;                    // 写端
  row_log_buf_t head;                    // 读端
  const char *path;                      // 临时文件目录（innodb_tmpdir）
};
```

DML 侧钩子（`row0log.ic:48`）——每一条二级索引 DML 都先问一句"我是不是正在被 online build"：

```cpp
switch (dict_index_get_online_status(index)) {
  case ONLINE_INDEX_COMPLETE:  return false;        // 正常索引，不记日志
  case ONLINE_INDEX_CREATION:  row_log_online_op(index, tuple, trx_id); break;
  case ONLINE_INDEX_ABORTED:   break;               // 已中止：不记日志且跳过操作
}
```

**`innodb_online_alter_log_max_size` 超限的后果**（默认 128MB，最小 64KB）：**不是锁表、不是降级，而是本次 DDL 失败**——

```cpp
    if (byte_offset + srv_sort_buf_size >= srv_online_max_size) goto write_failed;
    ...
    write_failed:
      index->type |= DICT_CORRUPT;      // 二级索引：标记损坏
      // rebuild 侧：log->error = DB_ONLINE_LOG_TOO_BIG;
```

回放（`row_log_apply`，`row0log.cc:3800`）：

```cpp
rw_lock_x_lock(dict_index_get_lock(index));   // ★ 短暂 X 锁索引（此时阻塞 DML）
error = row_log_apply_ops(trx, index, &dup, stage);
if (error != DB_SUCCESS) { index->type |= DICT_CORRUPT; ... }
else dict_index_set_online_status(index, ONLINE_INDEX_COMPLETE);   // ★ 索引对外可见
rw_lock_x_unlock(...);
row_log_free(log);
```

回放时机在 `Builder::finalize()`（`ddl0builder.cc:1949`）：**先 `Flush_observer::flush()` 把批量装载期间未记 redo 的脏页全部刷盘，再写 `MLOG_INDEX_LOAD` 恢复 redo，最后回放 row log**。顺序不能颠倒——否则回放产生的 redo 与未刷盘的批量装载页会冲突。

#### 并行构建与临时文件

- **`innodb_ddl_threads`（默认 4，范围 1-64）**：并行扫描（受 `innodb_parallel_read_threads`）+ 并行排序归并。`<=1` 时同步执行。
- **`innodb_ddl_buffer_size`（默认 1 MiB，范围 64KB-4GB）**：是**总量**，被 `Context::scan_buffer_size()` 均分给 `n_threads × n_indexes` 套（sort buffer + IO buffer）——**同时建多个索引时每个索引拿到的内存会变少**。
- **临时文件** `ddl::file_t`（`ddl0impl.h:67`）：目录取 `innodb_tmpdir`；排序缓冲满 → `key_buffer_sort()` → `serialize()` 以 append 方式写临时文件并记录每个有序段的偏移，之后 `Merge_file_sort` 多路归并。row log 也复用同一套临时文件机制。

---

### 索引的销毁与变更

#### DROP INDEX：三段式（标记 → 日志 → post_ddl 释放）

| 阶段 | 做什么 | 释放页？ |
|---|---|---|
| PREPARE | `index->to_be_dropped = 1`；若被外键引用则报 `HA_ERR_DROP_INDEX_FK` | 否 |
| INPLACE/COMMIT | `log_ddl->write_free_tree_log()` 往 `mysql.innodb_ddl_log` 插一条记录；`btr_drop_ahi_for_index()`；`dict_index_remove_from_cache()` | **否**（只打日志） |
| **post_ddl**（提交后） | `Log_DDL::post_ddl()` → `replay_free_tree_log()` → `btr_free_if_exists()` → `btr_free_but_not_root()` + `btr_free_root()` | **是**（同步释放） |

```cpp
// log0ddl.cc:1625
void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no, ulint index_id) {
  ...
  btr_free_if_exists(page_id_t(space_id, page_no), page_size, index_id, &mtr);
}
```

> **结论：DROP INDEX 的 B-tree 空间是提交后由 post_ddl 重放 DDL log 时"立即物理释放"的，不是 purge 回收。** purge 回收的是**行记录**（delete-mark 记录与回滚段历史），与"整棵索引树的段"无关。

最后一个细节：`btr_free_but_not_root` 用 `fseg_free_step*()` **分批**释放（每批一个 mtr），避免单个 mtr 过大——**这就是大索引 DROP 时有短暂 IO 抖动的原因**。

#### RENAME / 可见性 / DISABLE KEYS / 主键增删

- **RENAME INDEX**：`rename_index_in_cache()`（`handler0alter.cc:5260`）只改内存里的 `dict_index_t::name`，不动数据、不改 root page、**极快且 online**。
- **`ALTER INDEX ... VISIBLE/INVISIBLE`**：**InnoDB handler 完全没有对应实现**（grep `set_visible` 在 handler0alter.cc 只命中 `ALTER_COLUMN_VISIBILITY`）。可见性纯粹是 **DD 元数据**（`mysql.indexes.is_visible`），**不触碰任何页**——这是它瞬时的原因（详见 [`types.md`](types.md)）。
- **`DISABLE/ENABLE KEYS`**：`ha_innobase::enable_indexes/disable_indexes` 对普通 InnoDB 表直接返回 `HA_ERR_WRONG_COMMAND`（只对 intrinsic 临时表实现）。server 层退化为"只更新 DD 的 `keys_disabled`"，**索引仍被完整维护**。
- **`ADD/DROP PRIMARY KEY`**：触发 rebuild（整表重建 + 所有二级索引重建，因为二级索引叶子里存的是 PK 值）。`same_pk=false` 时 row log 要记录完整旧 PK + `DB_TRX_ID`/`DB_ROLL_PTR`，回放开销更大。优化：若新旧 PK 顺序一致（`innobase_pk_order_preserved`），聚簇索引可**跳过文件排序**（`skip_pk_sort`）。

---

### 索引的重建

#### OPTIMIZE TABLE 的真相

```cpp
// ha_innodb.cc:18125
int ha_innobase::optimize(THD *, HA_CHECK_OPT *) {
  if (innodb_optimize_fulltext_only) {
    /* 只对 FTS：同步 cache + optimize，不重建表 */
    fts_sync_table(m_prebuilt->table, false, true, false);
    fts_optimize_table(m_prebuilt->table);
    return HA_ADMIN_OK;
  } else {
    return HA_ADMIN_TRY_ALTER;     // ★ 让 server 去做 recreate + analyze
  }
}
```

server 收到 `HA_ADMIN_TRY_ALTER` 后（`sql_admin.cc:1237`）：提示 "Table does not support optimize, doing recreate + analyze instead" → `mysql_recreate_table()`（`ALTER TABLE ... FORCE` 等价路径，`RECREATE_TABLE` ∈ REBUILD）→ 之后再做一次 `ha_innobase::analyze()`。

**结论：`OPTIMIZE TABLE`（除 FTS）是"真 rebuild + analyze"，不是只做 analyze。**

**什么时候真的需要重建**：

1. `DATA_FREE` 显著（碎片）；
2. 页填充率低——删除/更新造成页内空洞，而 `btr_page_reorganize()` **只在插入放不下时本地重排、不跨页搬迁**，所以空洞不会自动消除；
3. 变更 `ROW_FORMAT`/`KEY_BLOCK_SIZE`/`TABLESPACE`（`innobase_need_rebuild()`）；
4. FTS 表清理已删除文档残留（`fts_optimize_table()`）。

只是统计陈旧 → **`ANALYZE TABLE` 即可，不必重建**。

---

### 可观测

#### `SHOW INDEX`：Cardinality 与缓存

**实现位置纠正**：8.0 的 `SHOW INDEX` **不在** `sql/sql_show.cc`，而是被重写为对系统视图 `SHOW_STATISTICS` 的查询（`sql/dd/info_schema/show.cc:756` 的 `build_show_keys_query()`）。

`Cardinality` 列来自 `INTERNAL_INDEX_COLUMN_CARDINALITY(...)`（`sql/item_func.cc:9483`），底层 `Table_statistics::read_stat()`（`sql/dd/info_schema/table_stats.cc:443`）：

```cpp
if (!is_persistent_statistics_expired(thd, cached_timestamp)) {
  return table_stat_data;        // ★ 缓存未过期：直接用 mysql.index_stats
}
if (hton_implements_get_statistics) read_stat_from_SE(...)   // 走引擎
else                             read_stat_by_open_table(...)  // 退化：开表再取
```

> **运维要点**：`SHOW INDEX` 的 `Cardinality` **默认最多陈旧 24 小时**（`information_schema_stats_expiry`）。想立刻取最新值：`SET information_schema_stats_expiry=0`。

其他列：`Sub_part`（前缀长度，经 `GET_DD_INDEX_SUB_PART_LENGTH` 做字节/字符换算）、`Collation`（`A`/`D`，降序索引）、`Index_type`（`BTREE`/`FULLTEXT`/`SPATIAL`）、`Comment`（`disabled` = DISABLE KEYS 生效中）、`Packed`（**恒 NULL**）。

#### `I_S.STATISTICS` 列与语义

`Statistics_base`（`statistics.cc:43`）的列与来源：

| 列 | 来源表达式 | 语义要点 |
|---|---|---|
| `NON_UNIQUE` | `IF(idx.type='PRIMARY' OR idx.type='UNIQUE',0,1)` | |
| `COLUMN_NAME` | `IF(col.hidden='SQL', NULL, col.name ...)` | **功能索引时 NULL**（隐藏列） |
| `COLLATION` | `CASE icu.order WHEN 'DESC' THEN 'D' WHEN 'ASC' THEN 'A' END` | 降序索引 |
| `SUB_PART` | `GET_DD_INDEX_SUB_PART_LENGTH(...)` | 前缀长度；非前缀为 NULL |
| `INDEX_TYPE` | `CASE WHEN idx.type='SPATIAL' THEN 'SPATIAL' ... ELSE idx.algorithm END` | |
| `COMMENT` | `IF(... INTERNAL_KEYS_DISABLED(tbl.options),'disabled','')` | |
| **`IS_VISIBLE`** | `IF(idx.is_visible,'YES','NO')` | 8.0 新增 |
| **`EXPRESSION`** | `IF(col.hidden='SQL', col.generation_expression_utf8, NULL)` | **功能索引的表达式**：`WHERE EXPRESSION IS NOT NULL` 可筛出功能索引 |
| `CARDINALITY` | `INTERNAL_INDEX_COLUMN_CARDINALITY(...)` | 仅 `Statistics` 视图有 |

过滤条件 `IS_VISIBLE_DD_OBJECT(tbl.hidden, idx.hidden OR icu.hidden, idx.options)`——GIPK 在 `show_gipk_in_create_table_and_information_schema=OFF` 时由此隐藏。

#### InnoDB 侧的三张 I_S 表

均在 `storage/innobase/handler/i_s.cc`（**访问需 PROCESS 权限**）：

| 表 | 关键列 | 语义 |
|---|---|---|
| `INNODB_TABLESTATS` | `NUM_ROWS`、`CLUST_INDEX_SIZE`（**页**）、`OTHER_INDEX_SIZE`（**页**）、**`MODIFIED_COUNTER`**、`STATS_INITIALIZED` | 表级统计；`MODIFIED_COUNTER / NUM_ROWS` 判断统计新鲜度；二级索引总空间的权威来源 |
| `INNODB_INDEXES` | `INDEX_ID`、`NAME`、`TYPE`（**位掩码**）、`N_FIELDS`（含系统列）、`PAGE_NO`、`MERGE_THRESHOLD` | **无 `n_diff` 列**（5.7 的 `INNODB_INDEX_STATS` 已移除）——看基数要查 `mysql.innodb_index_stats` |
| `INNODB_TABLESPACES` | `FILE_SIZE`、`ALLOCATED_SIZE`、`AUTOEXTEND_SIZE` | 物理文件大小；与 `stat_index_size × innodb_page_size` 对比可判断"文件是否虚胖" |

#### PFS 与 sys schema：无用与冗余索引

**`performance_schema.table_io_waits_summary_by_index_usage`**（`table_tiws_by_index_usage.cc:49`）：列 `COUNT_STAR` / `SUM_TIMER_WAIT` / `_READ/_WRITE/_FETCH/_INSERT/_UPDATE/_DELETE` 各统计族。**`INDEX_NAME IS NULL` 的那一行代表"全表扫描"**。

**`sys.schema_unused_indexes`**（`scripts/sys_schema/views/p_s/schema_unused_indexes.sql`）：

```sql
FROM performance_schema.table_io_waits_summary_by_index_usage t
INNER JOIN information_schema.statistics s ON ...
WHERE t.index_name IS NOT NULL      -- 排除全表扫描行
  AND t.count_star = 0              -- ★ 从未产生任何 IO 等待
  AND t.object_schema != 'mysql'
  AND t.index_name != 'PRIMARY'     -- 排除主键
  AND s.NON_UNIQUE = 1              -- 只保留非唯一二级索引
  AND s.SEQ_IN_INDEX = 1            -- 去重
```

**`sys.schema_redundant_indexes`**（`views/i_s/schema_redundant_indexes.sql`）三种冗余判定：

1. **列完全相同**且（非唯一度更高）或（非唯一度相同但索引名字典序更大）；
2. **非唯一前缀**：`LOCATE(CONCAT(redundant.index_columns, ','), dominant.index_columns) = 1` 且 `redundant.non_unique = 1`；
3. **唯一索引作为另一索引的前缀**：`LOCATE(CONCAT(dominant.index_columns, ','), redundant.index_columns) = 1` 且 `dominant.non_unique = 0`。

（尾逗号保证是完整列边界的前缀匹配，避免 `a` 误匹配 `ab`。）

**`Handler_read_*` 状态变量**（`mysqld.cc:9786`，判断索引 vs 全表的经典依据）：

| 变量 | 含义 |
|---|---|
| `Handler_read_key` | 按 key 定位一行（等值/ref 访问） |
| `Handler_read_next` | 按索引顺序读下一行（range/index scan） |
| `Handler_read_rnd_next` | **全表扫描**（不看索引） |
| `Handler_read_first` / `_last` / `_prev` / `_rnd` | 索引首/尾/前一行/按 rowid 随机读 |

---

## 相关的系统变量/状态变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `innodb_ddl_threads` | **4**（1-64） | 并行构建索引的线程数；`<=1` 同步 |
| `innodb_ddl_buffer_size` | **1 MiB**（64KB-4GB） | 总量，均分给 `线程数 × 索引数` 套缓冲 |
| `innodb_online_alter_log_max_size` | **128 MB**（64KB-∞） | row log 上限；**超限 = 本次 DDL 失败**（索引标 corrupt），不是降级 |
| `innodb_sort_buffer_size` | 1 MiB | 同时是 row log 的 block 大小 |
| `innodb_tmpdir` | 无（用 tmpdir） | 排序临时文件与 row log 临时文件的目录 |
| `innodb_optimize_fulltext_only` | OFF | ON 时 `OPTIMIZE` 只做 FTS optimize，不重建表 |
| `information_schema_stats_expiry` | **86400**（24h） | `I_S`/`SHOW` 统计缓存过期；`0` = 每次实时取 |
| `sql_require_primary_key` | OFF | 开启后 InnoDB 建无 PK 表报 `ER_TABLE_WITHOUT_PK` |
| `sql_generate_invisible_primary_key` | OFF | GIPK（详见 [`types.md`](types.md)） |
| `show_gipk_in_create_table_and_information_schema` | **ON** | 是否在 SHOW/I_S 显示 GIPK |

---

## Misc

### 索引 DDL 总表

| 操作 | 算法 | online | 关键函数 | 主要变量 |
|---|---|---|---|---|
| `ADD INDEX` | INPLACE | **是** | `ddl::Loader::build_all` → `Btree_load::build` → `row_log_apply` | `innodb_ddl_threads`、`innodb_ddl_buffer_size`、`innodb_online_alter_log_max_size` |
| `ADD SPATIAL` | INPLACE | 是（但 redo 不禁用、逐行插入） | `RTree_inserter` | 同上 |
| `ADD FULLTEXT` | INPLACE | 是 | `Builder::State::FTS_SORT_AND_BUILD` | 同上 |
| `DROP INDEX` | INPLACE | 是 | `write_free_tree_log` → post_ddl `replay_free_tree_log` → `btr_free_if_exists` | — |
| `RENAME INDEX` | INPLACE | 是 | `rename_index_in_cache` | — |
| `ALTER INDEX VISIBLE/INVISIBLE` | 纯 DD | **瞬时** | 无 InnoDB 实现 | — |
| `ADD/DROP PRIMARY KEY` | INPLACE（rebuild） | 是（但重建全表） | `row_log_table_apply` | `innodb_online_alter_log_max_size` |
| `OPTIMIZE TABLE` | recreate + analyze | 是 | `HA_ADMIN_TRY_ALTER` → `mysql_recreate_table` + `analyze` | `innodb_optimize_fulltext_only` |
| `ANALYZE TABLE` | — | 是 | `ha_innobase::analyze` → `dict_stats_update(PERSISTENT)` | 见 [`stats.md`](stats.md) |

### 可观测入口 × 数据来源 × 能回答什么

| 入口 | 数据来源 | 能回答 |
|---|---|---|
| `SHOW INDEX` / `I_S.STATISTICS` | DD `mysql.indexes` + `index_column_usage` | 有哪些索引、列顺序、是否唯一/降序/前缀/可见、功能索引表达式 |
| `Cardinality` | `stat_n_rows / rec_per_key`（有缓存） | 近似基数（注意 24h 缓存） |
| `mysql.innodb_index_stats` | `dict_stats_save` 写入 | **权威 n_diff / sample_size** |
| `I_S.INNODB_TABLESTATS` | 引擎内存 `dict_table_t` | 行数/页数、**统计新鲜度（MODIFIED_COUNTER）** |
| `I_S.INNODB_INDEXES` | 引擎内存 `dict_index_t` | TYPE 位掩码、root page、MERGE_THRESHOLD |
| PFS `table_io_waits_summary_by_index_usage` | PFS 事件聚合 | **索引到底被用过没**（`count_star=0`） |
| `sys.schema_unused_indexes` / `schema_redundant_indexes` | PFS + I_S | 无用索引 / 冗余索引清单 |
| `Handler_read_*` | server 状态变量 | 索引访问 vs 全表扫描的整体比例 |

### 易混淆点

- **`DROP INDEX` 不是 purge 回收**：purge 回收行记录，整树释放靠 post_ddl 重放 `FREE_TREE_LOG`。
- **`OPTIMIZE TABLE` 是 rebuild + analyze**，不是只 analyze；只统计陈旧用 `ANALYZE`。
- **`ALTER INDEX INVISIBLE` 瞬时**（纯 DD），但 **DML 仍完整维护该索引**。
- **`INNODB_INDEXES` 没有 `N_DIFF_FIELDS`**（5.7 已移除），基数在 `mysql.innodb_index_stats`。
- **`innodb_online_alter_log_max_size` 超限 = DDL 失败**，不是"自动降级为锁表"。
- **`SHOW INDEX` 的 Cardinality 有 24h 缓存**，`information_schema_stats_expiry=0` 才实时。
- **`DISABLE KEYS` 对 InnoDB 不生效**（只改 DD `keys_disabled`，索引照维护）。

### 一句话总结

索引的"生"是扫描 + 排序 + 批量装载 + row log 回放四条流水并行，"死"是标记 + DDL log 延迟到提交后释放，变更多为纯字典操作；观测则要分清三层——DD 元数据看定义、引擎内存看统计与页数、PFS 看实际使用。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.12 InnoDB and Online DDL*
- *MySQL 8.0 Reference Manual → 15.6.2.6 Optimizing InnoDB DDL Operations*
- *MySQL 8.0 Reference Manual → 27.4 The INFORMATION_SCHEMA STATISTICS Table*
- *MySQL 8.0 Reference Manual → 28 MySQL sys Schema*

**相关文档**
- online DDL 通用框架（row log、DDL log）：[`../innodb/ddl.md`](../innodb/ddl.md)
- 批量建页 `Btree_load` 算法：[`btr.md`](btr.md)
- 统计采样与 `rec_per_key`：[`stats.md`](stats.md)
- 索引类型（不可见索引、GIPK、降序/前缀/功能/多值）：[`types.md`](types.md)
- 索引访问优化：`access.md`
- 优化器侧：[`../server/query/07_optimize/`](../server/query/07_optimize/)
