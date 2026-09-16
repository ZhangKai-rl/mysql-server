# InnoDB 内部字典（dict_sys）

> **定位**：引擎私有的运行时字典——`dict_table_t` / `dict_index_t` 对象模型、`dict_sys_t` 缓存、DICT_HDR 页、`SYS_*` 内部表，以及**它如何从 server 层 DD 加载**。
> **边界**：server 层 DD（权威元数据）见 [`dd.md`](dd.md)；表对象（`TABLE`/`TABLE_SHARE`）见 [`../table.md`](../table.md)；`DB_ROW_ID` 计数器与 DICT_HDR 布局见 [`../../innodb/physical/record.md`](../../innodb/physical/record.md)；DDL 执行见 [`../../innodb/ddl.md`](../../innodb/ddl.md)。

> 说明：本工作副本源码带少量中文批注（`// note:` / ASCII 图），引用时已与上游注释区分。

---

## 概述

### 是什么

`dict_sys_t`（全局单例，`dict0dict.h:1006`）持有所有 `dict_table_t`，每个 `dict_table_t` 又持有若干 `dict_index_t`。

★ **它是 DD 的投影，不是独立真相**——崩溃后整份丢弃，启动时**并不重建**，而是**惰性按需构建**（见第 6 节）。

### 为什么两层都要

| | server 层 DD | InnoDB dict |
|---|---|---|
| 语义 | SQL 语义（列名、类型、字符集、表达式） | 物理语义（`phy_pos`、root 页号、space id、instant 版本） |
| 持久 | DD 表（mysql.ibd） | 无（运行态 + 少量"动态元数据"） |

粒度不同，所以各存一份，由加载逻辑对齐。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.x | InnoDB 自己就是字典权威（`SYS_TABLES` 等持久化） |
| **8.0** | DD 成为权威；**`SYS_*` 退化为升级兼容**（仅 `srv_is_upgrade_mode` 时读，`dict0dict.h:1031` 注释 "they're only for upgrade"） |
| 8.0.29 | instant ADD/DROP 元数据进 dict（`current_row_version` / `nullables[]` 版本表） |

---

## 1. `dict_table_t`（`dict0mem.h:1939`）

### 1.1 标识与定位

| 字段 | 行号 | 说明 |
|---|---|---|
| `id` | 1994 | `dd::Table/Partition::se_private_id()` |
| `name` | 2008 | `db/tbl` 形式；分区带 `#p#` / `#P#` |
| `space` / `dd_space_id` | 2021 / 2024 | space id（引擎）vs `dd::Tablespace::id`（DD），**两者不同** |
| `flags` | 2034 | 行格式 / zip_ssize / atomic_blob / data_dir / shared_space |
| `flags2` | 2047 | TEMPORARY / FTS / AUX / INTRINSIC / DISCARDED / ENCRYPTION… |
| `heap` | 2005 | 成员均从此 heap 分配；**从 heap 分配后必须同步 `dict_sys->size`** |

### 1.2 ★ instant 的核心：三组计数器

```cpp
/* 传统计数（dict0mem.h:2071-2095） */
unsigned n_def : 10;           // 已定义的非虚列数
unsigned n_cols : 10;          // 非虚列数（含系统列）
unsigned n_instant_cols : 10;  // 第一次 instant ADD 之前的非虚列数（V1 用）
unsigned n_t_cols : 10;        // 总列数（含虚列）
unsigned n_v_cols : 10;        // 虚列数
unsigned n_m_v_cols : 10;      // 多值虚列数

/* V2 instant 计数（来自 dd_table_get_column_counters()，dict0mem.h:2151-2166） */
uint32_t current_row_version;   // 当前行版本
uint32_t initial_col_count;     // 初始非虚列数
uint32_t current_col_count;     // 当前非虚列数
uint32_t total_col_count;       // 总非虚列数（含 instant drop 的）
bool m_upgraded_instant;        // 从 V1 instant 升级而来
```

派生判定（`dict0mem.h:2548-2565`）：

```cpp
get_n_instant_add_cols()  = total_col_count - initial_col_count;
get_n_instant_drop_cols() = total_col_count - current_col_count;
has_instant_add_cols()    = get_n_instant_add_cols()  > 0;
has_instant_drop_cols()   = get_n_instant_drop_cols() > 0;
has_row_versions()        = current_row_version > 0;   // 断言：此时必有 add 或 drop
```

★ 这就是 [`record.md`](../../innodb/physical/record.md) 里 instant 行版本状态机读的那些计数的**来源**——不是解析器算的，是字典里存的。

#### dd version：`dd_table_has_row_versions`

这套 instant 计数的**持久侧**在 `dd::Table`/`dd::Partition` 的 `se_private_data`（见 [`dd.md`](dd.md) 2.4 节：`version_added` / `version_dropped` / `physical_pos` / `instant_col`）。判断入口是 `dd_table_has_row_versions()`（`dict0dd.ic`）：

```cpp
// 表有 instant 行版本 ⇔ se_private_data 里有 instant_col 或任一行有版本
inline bool dd_table_has_row_versions(const dd::Table &table);
```

它的典型用法（`handler0alter.cc:510`）：

```cpp
if (!dd_table_has_row_versions(*old_dd_tab)) {
  return;   // 没有 instant 历史 → 不需要迁移 physical_pos
}
// Copy col phy pos from old DD table to new DD table
inherit_instant_metadata(&old_part->table(), &new_part->table());
```

★ 即 **instant 元数据在 ALTER 时的继承**：新建表对象要把旧的 `physical_pos` 等五个键搬过去，否则行版本解析会错位。这也是 `TRUNCATE PARTITION` 会抹掉分区级 `instant_col`（分区值恒 >= 表级）的同一套机制的另一面。

### 1.3 其它重要成员

| 成员 | 行号 | 说明 |
|---|---|---|
| `cols` / `v_cols` / `s_cols` | 2110 / 2113 / 2120 | 非虚列数组 / 虚列数组 / 存储生成列（仅 create/copy alter 期做 FK 检查） |
| `indexes` | 2145 | 索引链表，**第一个必须是聚簇索引**（`first_index()` @2487） |
| `foreign_set` / `referenced_set` | 2202 / 2205 | 作为子表 / 父表的外键 |
| `fts_doc_id_index` | 2142 | FTS 专用 |
| `is_dd_table` | 2480 | DD 表标记；注释：用于"对 DD 表做无锁读" |
| `n_ref_count` | 2420 | 引用计数；**归零才允许淘汰/删除** |
| `n_rec_locks` | 2412 | 本表记录锁计数 |
| `locks` / `count_by_mode[]` | 2425 / 2438 | 表锁链表与按模式计数（避免遍历） |
| `can_be_evicted` / `ddl_not_evictable` / `explicitly_non_lru` | 2099 / 2103 / 2484 | 缓存可淘汰性 |
| `dirty_status` / `dirty_dict_tables` | 2169 / 2174 | 动态元数据脏状态（`METADATA_DIRTY/BUFFERED/CLEAN`） |
| `stat_*` | 2220-2325 | 统计（持久化统计另存 `mysql.innodb_table_stats`） |
| `vc_templ` / `temp_prebuilt` | 2464 / 2474 | 虚拟列模板 / 并行读的 per-reader prebuilt |

### 1.4 autoinc 系列（`dict0mem.h:2328-2391`）

`autoinc_lock`（预分配的 AUTOINC 锁实例）、`autoinc_mutex`、`autoinc`（下一个值）、`autoinc_persisted`（已写入 redo/DDTableBuffer 的值）、`autoinc_field_no`、`autoinc_trx`（原子指针，快速判断本 trx 是否已持 AUTOINC）。详见 [`../feat/auto_increment.md`](../../feat/auto_increment.md)。

### 1.5 来自 DD vs 运行态

**来自 DD**（每次 open 重填）：`id`、`name`/`space`/`tablespace`/`data_dir_path`、`flags`/`flags2` 大部分位、全部列定义、`initial/current/total_col_count`、`current_row_version`、`n_instant_cols`（升级表）、`version`、`autoinc` 初值与 `autoinc_persisted`、索引 id/root/space/trx_id。

**纯运行态**：`heap`、`n_def`/`n_t_def`/`n_v_def`、`n_ref_count`/`n_rec_locks`/`locks`/`count_by_mode`、可淘汰性各标志、所有 `stat_*`、`dirty_status`、`autoinc` 运行时值、`fts`/`vc_templ`、`is_dd_table`（现算）。

---

## 2. `dict_index_t`（`dict0mem.h:1065`）

### 2.1 定位与类型位

```cpp
space_index_t id;      // 1067
unsigned space : 32;   // 1082
unsigned page  : 32;   // 1085  root 页号
unsigned type  : DICT_IT_BITS;  // 1092（10 bit）
```

`type` 位（`dict0mem.h:96-115`）：`DICT_CLUSTERED=1` / `UNIQUE=2` / `IBUF=8` / `CORRUPT=16` / `FTS=32` / `SPATIAL=64` / `VIRTUAL=128` / `SDI=256` / `MULTI_VALUE=512`。

### 2.2 ★ instant 三件套

```cpp
unsigned n_fields : 10;         // 1128 字段数
unsigned n_total_fields : 10;   // 1131 含 INSTANT dropped 字段
unsigned n_nullable : 10;       // 1134
unsigned n_instant_nullable : 10; // 1138 第一次 instant ADD 之前的 nullable 数
unsigned instant_cols : 1;      // 1157 聚簇且有 instant 列
unsigned row_versions : 1;      // 1160 聚簇且表有行版本

dict_field_t *fields;                          // 1178
std::vector<uint16_t> fields_array;            // 1182 按物理位置排序（仅 instant 聚簇需要）
uint32_t nullables[MAX_ROW_VERSION + 1];       // 1186 每个行版本的 nullable 数（MAX_ROW_VERSION=64）
```

`n_fields` 在有 row_versions 时会减掉 drop 列（`dict0dict.cc:2486`：`n_fields = n_def - get_n_instant_drop_cols()`）。`nullables[]` 由 `create_nullables()`（`dict0mem.cc:622`）按每列的 `version_added`/`version_dropped` 区间累加生成。

### 2.3 ★ `is_usable()`：可用性的三个条件

```cpp
// dict0mem.cc:604
bool dict_index_t::is_usable(const trx_t *trx) const {
  if (!is_clustered() && dict_index_is_online_ddl(this)) return false;   // ① 正在 online DDL
  if (is_corrupted()) return false;                                       // ② 损坏
  return (table->is_temporary() || trx_id == 0 ||                          // ③ MVCC 可见
          !MVCC::is_view_active(trx->read_view) ||
          trx->read_view->changes_visible(trx_id, table->name));
}
```

③ 常被忽略：**索引自身带 `trx_id`**（创建它的事务），若对当前 read view 不可见则不能用——避免看到尚未提交的 DDL 建的索引。

`is_corrupted()` 只看 `type & DICT_CORRUPT`（`dict0mem.h:1327`）；置位通过 `dict_set_corrupted()`（`dict0dict.cc:4163`）写 `PM_INDEX_CORRUPTED` 动态元数据 redo，并强制 `log_write_up_to` 确保不丢。

### 2.4 其它

`online_log`（1211，online DDL 的 row log）、`lock`（1266，保护 B-tree 上层的 rw_lock）、`merge_threshold`、`trx_id`（1259）、`srid`（空间索引）、`search_info`。

★ `get_sys_col_pos(type)`（`dict0mem.cc:716`）：**聚簇索引返回物理位置**（`dict_col_get_clust_pos`），二级索引返回逻辑位置（`get_col_pos`）——所以常见写法是 `ut_ad(get_sys_col_pos(DATA_ROLL_PTR) == trx_id_pos + 1)`（物理上 TRX_ID 紧邻 ROLL_PTR）。

---

## 3. `dict_col_t`（`dict0mem.h:489`）

完整字段与 ★ `phy_pos` 位域编码见前面已填充的内容。补充一个易错点：

```cpp
set_prefix_phy_pos(uint16_t prefix_pos) { phy_pos = prefix_pos; phy_pos <<= 16; phy_pos |= 0x8000; }
set_col_phy_pos(uint16_t pos) { ut_ad(has_prefix_phy_pos()); phy_pos |= pos; }   // ★ 必须先调前者
```

即**物理位置的设置顺序有断言保护**：先设前缀位置（置 bit15 标志），再设列位置。

instant drop 列的命名约定（`dict0mem.h:84-86`）：8.0.29 用后缀 `_dropped_v`、8.0.32 起用前缀 `!hidden!_dropped_`。

`dict_field_t`（`dict0mem.h:913`）：`col` / `name` / `prefix_len:12` / `fixed_len:10` / `is_ascending:1`；`get_phy_pos()` 在 `prefix_len != 0` 时返回前缀位置。

★ `fixed_len` 的填充有个细节（`dict.cc:120`）：`> DICT_MAX_FIXED_COL_LEN(768)` 就视为变长（`fixed_len = 0`）。

---

## 4. dict cache（`dict_sys_t`）

### 4.1 结构（`dict0dict.h:1006`）

```cpp
DictSysMutex mutex;              // 1009  唯一一把，串行化 DDL 与字典读取
row_id_t row_id;                 // 1017
hash_table_t *table_hash;        // 1024  key = 表名（ut::hash_string）
hash_table_t *table_id_hash;     // 1026  key = 表 id（ut::hash_uint64）
size_t size;                     // 1028
dict_table_t *sys_tables, *sys_columns, *sys_indexes, *sys_fields, *sys_virtual;  // 1033-1037 仅升级
dict_table_t *table_stats, *index_stats, *ddl_log, *dynamic_metadata;             // 1040-1046 常驻
Table_LRU_list_base table_LRU;       // 1050  可淘汰
Table_LRU_list_base table_non_LRU;   // 1052  不可淘汰
```

★ **8.0.39 的 dict cache 未分片**——只有一把 `mutex`（`ib_mutex_t`），`hash_table_t` 自身无锁（`HASH_TABLE_SYNC_NONE`）。配合全局 `dict_operation_lock`（rw_lock，**DDL/淘汰时必须先 x-lock 它再进 dict mutex**）。

### 4.2 淘汰

触发者是 **master thread**（`srv0srv.cc:2002` `srv_master_evict_from_table_cache`，周期性检查 `dict_make_room_in_cache`）。

能否淘汰的三条件（`dict0dict.cc:1264` `dict_table_can_be_evicted`）：

1. `n_ref_count == 0`
2. 无任何表锁/记录锁（`lock_table_has_locks()`）
3. 所有索引的 AHI `search_info->ref_count == 0`

且表本身必须在 `table_LRU`（`can_be_evicted == true`）。

★ **哪些表住在 `table_non_LRU`（永不淘汰）**：

| 类别 | 进入时机 | 为什么不能淘汰 |
|---|---|---|
| **系统 DD 表** | `dict_sys` 常驻指针（`sys_tables` / `table_stats` / `ddl_log` / `dynamic_metadata`…） | 崩溃恢复与 DDL 期间 DD 尚未就绪，必须能直接拿到 |
| **被外键引用的表** | `dict_foreign_add_to_cache()` | 外键检查要顺着 `referenced_table` 指针找过去；淘汰会导致悬空指针 |
| **有全文索引的表** | `fts_optimize_add_table()` | FTS 后台优化线程随时要访问 |
| `explicitly_non_lru` 置位的表 | `dict_table_prevent_eviction()`（如 DDL 期间） | DDL 期间表不能被抽走 |

共同点：**这些表的指针被别的结构长期持有**，淘汰即悬空。所以不是"缓存策略选择"，而是**正确性约束**。

#### 淘汰的一个反直觉后果

表被淘汰时要**回写动态元数据**（见第 8 节）——因为 autoinc 计数器只活在内存 + `mysql.innodb_dynamic_metadata` 里，淘汰即"最后一次保存机会"。

★ **容量上限来自 server 层参数**：`ha_innodb.cc:2471` `innobase_get_table_cache_size()` 直接 `return table_def_size;`——即 **InnoDB dict cache 的大小由 `table_definition_cache`（TDC 的参数）决定**，不是独立参数。

### 4.3 ★ 与 server 层 TDC 的分工

| | InnoDB dict cache | server TDC |
|---|---|---|
| 内容 | `dict_table_t`（物理元数据） | `TABLE_SHARE`（SQL 元数据） |
| 容量 | 取 `table_def_size` | `table_definition_cache` |
| 淘汰 | master thread 定期 | LRU |

打开顺序：`ha_innobase::open()` 拿到的已经是**构造好的 `TABLE` + `TABLE_SHARE` + `dd::Table`**（TDC 已读 DD），InnoDB 只做"翻译成 `dict_table_t`"。所以 **TDC miss 才付 DD 读取代价，之后再触发 `dd_open_table()`；dict cache miss 不会导致 TDC miss**。

一致性校验点：`ha_innodb.cc:7310` 列数不一致 → 报 `ER_IB_MSG_556` 并给聚簇索引打 `DICT_CORRUPT`。

### 4.4 ★ 第四个缓存：session 级 intrinsic 表缓存

除全局 dict cache 外，还有一个**每会话**的小缓存——只为 **intrinsic 临时表**（`table->is_intrinsic`，即内部临时表）服务：

```cpp
// ha_innodb.cc:2076
static inline void add_table_to_thread_cache(dict_table_t *table,
                                             mem_heap_t *heap, THD *thd) {
  dict_table_add_system_columns(table, heap);
  dict_table_set_big_rows(table);
  innodb_session_t *&priv = thd_to_innodb_session(thd);
  priv->register_table_handler(table->name.m_name, table);
}

// sess0sess.h:105
void register_table_handler(const std::string &table_name, dict_table_t *table) {
  m_open_tables.insert(table_cache_t::value_type(
      table_name, new dict_intrinsic_table_t(table)));
}
```

即 `innodb_session_t::m_open_tables` 是一个 `map<表名, dict_intrinsic_table_t*>`。三个特点：

1. **不进全局 dict cache**——intrinsic 表不需要 MDL、不需要 DDL 可见性，进全局缓存反而要参与锁/淘汰/DD 交互，纯属负担；
2. **随会话销毁**（`~innodb_session_t` 遍历 delete）；
3. `dict_intrinsic_table_t` 是 `dict_table_t` 的包装（handler 形式），对内部表提供 handler 接口。

★ 这就是"内部临时表开表"的快速路径：`ha_innobase::open` 里 intrinsic 表先查这个 map，命中就绕过了 dd_open_table 全链路。

### 4.5 ★ index corrupt 的完整闭环

`is_usable()` 三条件里的 `is_corrupted()` 是入口。完整闭环：

```
标记：dict_set_corrupted(index)          dict0dict.cc
        → 置 index->type |= DICT_CORRUPT
        → dict_table_mark_dirty(table)   加入动态元数据脏链
        → CorruptedIndexPersister::write 序列化（PM_INDEX_CORRUPTED + 12B/索引）
        → Persister::write_log           写 MLOG_TABLE_DYNAMIC_META redo
        → checkpoint 时快照进 mysql.innodb_dynamic_metadata

恢复：redo 扫描 parseMetadataLog → table_buffer
        → dict_table_load_dynamic_metadata（开表时）
        → dict_table_apply_dynamic_metadata → 索引标记为 corrupt

使用：dict_index_t::is_usable() → is_corrupted() → false
        → 优化器放弃该索引（查询可用，索引不可用）
```

★ **corrupt 标记是"可恢复的降级"**：表还能打开（所以才能被 load 回来）、数据还能查（全表扫描），只是索引被禁用。`ALTER TABLE ... FORCE`（重建索引）或 `CHECK TABLE` 修复后清除。这也是它走"动态元数据"通道而不是写死 DD 的原因——它必须**跟着表的内存对象走**，且恢复成本低、变化频繁。

---

### 4.6 ★ 缓存对象的三条生死路径（remove / resize / 一致性校验）

#### `dict_table_remove_from_cache_low`（`dict0dict.cc:1882`）—— 淘汰的完整步骤

```cpp
ut_a(table->get_ref_count() == 0);      // ① 引用计数归零才允许
ut_a(table->n_rec_locks.load() == 0);   // 无记录锁
ut_ad(dict_sys_mutex_own());

/* ② 动态元数据三态处理（★ 淘汰即最后一次保存机会，见第 8 节） */
switch (table->dirty_status.load()) {
  case METADATA_DIRTY:
    dict_table_persist_to_dd_table_buffer(table);   // 回写
    [[fallthrough]];
  case METADATA_BUFFERED:
    UT_LIST_REMOVE(dict_persist->dirty_dict_tables, table);   // 摘脏链
    break;
  case METADATA_CLEAN: break;
}

/* ③ 外键关系清理：foreign_set 移除、referenced_set 里被引用的指针置 null */
/* ④ 逐个索引 dict_index_remove_from_cache_low(...) */
/* ⑤ 双哈希删除：HASH_DELETE(name_hash) + HASH_DELETE(id_hash) */
/* ⑥ 按 can_be_evicted 从 table_LRU 或 table_non_LRU 摘除 */
/* ⑦ 释放 vc_templ（虚拟列模板） */
dict_sys->size -= mem_heap_get_size(table->heap) + strlen(name) + 1;  // ⑧ 记账
dict_mem_table_free(table);              // ⑨ 物理释放
```

#### `dict_index_remove_from_cache_low`（`dict0dict.cc:2644`）—— 为什么必须持 dict mutex

本仓库注释总结了持锁的四个原因：

```
删除 dict_index_t（remove dict_index_t from dict_sys lru cache）时必须持有 dict sys mutex：
  - 保护 AHI 引用计数：等待所有 AHI 引用归零
  - 防止 use-after-free：其他线程可能通过 block->ahi.index 访问索引
  - 保护表的索引链表：防止并发修改 table->indexes
  - 保护全局字典状态：更新 dict_sys->size
```

核心动作：`btr_search_await_no_reference(table, index, !lru_evict)`（★ **等 AHI 引用归零**，`lru_evict=true` 时不等待——淘汰场景 AHI 反正要失效）→ `rw_lock_free(&index->lock)` → 压缩统计 `page_zip_stat_per_index.erase` → `UT_LIST_REMOVE(table->indexes, index)` → 虚列索引列表清理。

#### `dict_resize`（`dict0dict.cc:4597`）—— buffer pool resize 时重建哈希

```cpp
void dict_resize() {
  dict_sys_mutex_enter();
  ut::delete_(dict_sys->table_hash);      // ① 删旧表
  ut::delete_(dict_sys->table_id_hash);

  dict_sys->table_hash = ut::new_<hash_table_t>(
      buf_pool_get_curr_size() / (DICT_POOL_PER_TABLE_HASH * UNIV_WORD_SIZE));
  // ② ★ 哈希桶数 = buffer pool 大小 / (DICT_POOL_PER_TABLE_HASH × 8)
  //     —— 字典哈希表大小跟着 buffer pool 走

  for (auto table : dict_sys->table_LRU) { ... HASH_INSERT × 2 }       // ③ 重灌
  for (auto table : dict_sys->table_non_LRU) { ... HASH_INSERT × 2 }
  dict_sys_mutex_exit();
}
```

★ **字典哈希大小与 buffer pool 成比例**——因为"表对象个数"与"缓存能装下的表个数"正相关，buffer pool 越大表越多，哈希桶要跟着扩，否则退化成长链。

#### ★ `dd_table_match`（`dict0dd.cc:250`）—— 缓存命中后的"权威一致性校验"

开表在 dict cache 命中时，**不是直接返回缓存对象，而是先和 DD 权威对账**（`ha_innodb.cc:7249`）：

```cpp
bool dd_table_match(const dict_table_t *table, const Table *dd_table) {
  if (dd_table == nullptr || table->is_temporary()) return true;   // 临时表无元数据

  if (dd_table->se_private_id() != table->id) {                    // ① id 不一致
    ib::warn(ER_IB_MSG_166) << "Table id in InnoDB is " << table->id
        << " while the id in global DD is " << dd_table->se_private_id();
    match = false;
  }
  if (dict_table_is_discarded(table)) return match;                // ② 已 discard 不查索引

  for (const auto dd_index : dd_table->indexes()) {                // ③ 逐索引比对
    const dict_index_t *index = dd_find_index(table, dd_index);
    if (!dd_index_match(index, dd_index)) match = false;
  }
  ...
}
```

不一致的后果（`ha_innodb.cc:7249-7255`）：

```cpp
if (!dd_table_match(ib_table, table_def)) {
  dict_set_corrupted(ib_table->first_index());   // ★ 打损坏标记
  dict_table_remove_from_cache(ib_table);        // 移除缓存
  ib_table = nullptr;                             // 强制走重新加载
}
```

★ **这就是"字典不一致"的防御闭环**：缓存里的 `dict_table_t` 若与 DD 权威不符（如 DDL 在别的连接改了定义、缓存没跟上），不是"将错就错"，而是**判缓存失效 → 标损坏 → 移除 → 重新从 DD 加载**。第 4.5 节的 index corrupt 闭环与这里是同一个防御体系的两半：corrupt 是"索引损坏"的记录机制，`dd_table_match` 是"缓存与权威不符"的检测机制。

## 5. 从 DD 加载（核心链路）

```
ha_innobase::open(name, ..., const dd::Table *table_def)      ha_innodb.cc:7177
  ├─ session 私有缓存 lookup（intrinsic 表）                              :7205
  ├─ dict_table_check_if_in_cache_low(name) 命中 → 校验/刷新后 acquire     :7208-7254
  └─ 未命中 → dd_open_table(client, table, name, table_def, thd)   dict0dd.cc:5448
        └─ dd_open_table_one()                                   dict0dd.cc:5075
             ├─ ★ dd_fill_dict_table()                           dict0dd.cc:3808
             │     ├─ dd_table_get_column_counters()  → 四个 instant 计数   :3942
             │     ├─ dict_mem_table_create(...)                          :3944
             │     ├─ table->id = dd_tab->se_private_id()                 :3954
             │     ├─ flags/flags2 逐位（data_dir/discarded/compact/file_per_table/
             │     │     atomic_blobs/zip/fts/temporary/encryption）       :3956-4045
             │     ├─ fill_dict_columns()  → dict_col_t                :4050 → :3641
             │     └─ dd_fill_instant_columns_default()（有 instant 时）   :4059
             ├─ ★ dd_fill_dict_index()                            dict0dd.cc:3200
             │     ├─ 无 PK → "GEN_CLUST_INDEX"（DICT_CLUSTERED, n_uniq=0） :3232
             │     └─ 各索引 dd_fill_one_dict_index()                  :2953
             ├─ autoinc 初始化                                          :5129-5144
             ├─ 回填每个索引的 page/space/id/trx_id（se_private_data）      :5151-5251
             ├─ dict_table_add_to_cache()                               :5283
             ├─ dict_table_load_dynamic_metadata()                      :5292
             └─ dd_table_load_fk(...)                                   :5314
```

### ★ `se_private_data` 的 key（DD 与 InnoDB 的私有通道）

| 层级 | key |
|---|---|
| 表 | `autoinc` / `data_directory` / `version` / `discard` / `instant_col` |
| 分区 | `format` / `instant_col` / `discard` |
| 列 | `default` / `default_null` / `version_added` / `version_dropped` / `physical_pos` |
| 索引 | `id` / `space_id` / `table_id` / `root` / `trx_id` |

★ 这就是"DD 是权威、dict 是投影"的**具体通道**：InnoDB 把只能自己知道的物理信息（root 页号、`physical_pos`、instant 版本）塞进 `se_private_data` 存回 DD。

### 一致性断言

```cpp
// dict0dd.cc:2508（及 row0import.cc:1190）
ut_ad(dict_table_is_partition(table) == dd_table_is_partitioned(*dd_table));
```

其它：`dict0dd.cc:3771`（`current_row_version` 逐列累计校验）、`:3461`（instant 默认列数）、`:5234`（`root > 1`）、`ha_innodb.cc:14099`（`dd_table_match`）。

---

## 6. DICT_HDR 与启动（★ 反直觉结论）

### DICT_HDR 现在只剩"ID 计数器"

| 能力 | 5.x | **8.0.39** |
|---|---|---|
| 表/索引元数据持久化 | DICT_HDR → `SYS_*` | **移交 DD** |
| table/index/space id 分配 | DICT_HDR | **仍然是**（`dict_hdr_get_new_id`，`dict0boot.cc:70`） |
| 全局 row id | DICT_HDR_ROW_ID | **仍然是**（`dict_hdr_flush_row_id`，`:151`） |
| `SYS_*` root 页号 | DICT_HDR | **仅 `srv_is_upgrade_mode` 时读** |

字段（`dict0boot.h:113-135`）：`ROW_ID=0` / `TABLE_ID=8` / `INDEX_ID=16` / `MAX_SPACE_ID=24` / `MIX_ID_LOW=28`(废弃) / `TABLES=32` / `TABLE_IDS=36` / `COLUMNS=40` / `INDEXES=44` / `FIELDS=48` / `FSEG_HEADER=56`。

★ `dict_hdr_create()` 只在 **bootstrap 建库时**调一次（`dict0boot.cc:172`），且 `TABLE_ID` 起点是 `DICT_MAX_DD_TABLES = 1024`——**给 DD 表预留了 1024 个 id**，说明 id 空间已与 DD 表共存。

### ★ 启动时不重建 dict cache

`dict_boot()`（`dict0boot.cc:205`）只做四件事：

1. `dict_init()`（分配 `dict_sys`、两把哈希表、`dict_operation_lock`）；
2. 读 row_id 并**向上对齐 +256**（`DICT_HDR_ROW_ID_WRITE_MARGIN`，补偿"每 256 次才落盘"）；
3. `if (srv_is_upgrade_mode)` 才载入 `SYS_*` 四表（含 13 条 `static_assert` 校验列数）；
4. `ibuf_init_at_db_start()`。

**缓存条目此后按需构建**——不是启动时遍历 DD 灌满。恢复期间（无 THD）走 `dd_table_open_on_id_low()`（`dict0dd.cc:486`），注释说明理由：*"During server startup, while recovering XA transaction we don't have THD..."*。

---

## 7. `SYS_*` 四表（仅升级用）

`dict_sys` 里 5 个句柄（含 `sys_virtual`），注释即 "only for upgrade"（`dict0dict.h:1031`）。

| 表 | 列数 | 字段数 | 固定 id |
|---|---|---|---|
| SYS_TABLES | 8 | 10 | 1 |
| SYS_COLUMNS | 7 | 9 | 2 |
| SYS_INDEXES | 8 | 10 | 3 |
| SYS_FIELDS | 3 | 5 | 4 |
| SYS_TABLE_IDS | — | 2 | 5 |

`dict0boot.cc:242-255` 有 **13 条 `static_assert`** 钉死这些数字（"Be sure these constants do not ever change"）——升级路径对格式变化零容忍。

`dict0load.cc` 中的加载函数（`dict_load_table_one` / `_low` / `dict_load_columns` / `dict_load_indexes` / `dict_load_fields` / `dict_load_virtual` / `dict_load_foreigns`）**全部只在升级路径上被调用**。

---

## 8. 特殊 `dict_table_t`

| 类型 | 要点 |
|---|---|
| **分区** | 每分区一个 `dict_table_t`，分隔符 `#p#`（8.0.17 前 `#P#`）、子分区 `#sp#`（`dict0types.h:71-88`）；判定 `dict_table_is_partition()` 看名字；打开时若 `se_private_id() == INVALID_OBJECT_ID` 则遍历 `leaf_partitions()` 匹配 |
| **虚拟列** | `n_v_cols` / `v_cols` / `vc_templ`；列上 `prtype & DATA_VIRTUAL`；索引类型 `DICT_VIRTUAL` / `DICT_MULTI_VALUE` |
| **FTS 辅助表** | 命名 `fts_<table_id>_*`；`flags2 & DICT_TF2_AUX` + `parent_id`；索引 `DICT_FTS` |
| **临时 / intrinsic** | `DICT_TF2_TEMPORARY` / `INTRINSIC`；★ **intrinsic 表不入 dict cache**，走 session 私有缓存（`innodb_session_t::lookup_table_handler()`）；系统列只有 2 个（无 ROLL_PTR），本地 `sess_row_id` / `sess_trx_id` |
| **DD 表自身** | `is_dd_table`（用于"对 DD 表做无锁读"、跳过 GAP 锁）；space id `0xFFFFFFFE`；`dict0dict.h:1083` 有 `s_dd_table_ids` 集合；`dict_sys` 常驻 `table_stats` / `index_stats` / `ddl_log` / `dynamic_metadata` 四个句柄 |

#### mysql.ibd 的硬编码创建（`dd_create_hardcoded`）

普通表空间的创建走 `CREATE TABLESPACE` 的 SQL 链路，但 **mysql.ibd 必须先于一切存在**（否则无从读取 DD 表定义），所以它用一套专用函数（`ha_innodb.cc:5397`）：

```
dd_create_hardcoded(s_dict_space_id, "mysql.ibd")
  ├─ fil_ibd_create()            // 建物理文件，初始 7 个页
  ├─ fsp_header_init()           // 初始化 Page 0（FSP_HDR）
  │    ├─ 写 FIL_PAGE_TYPE = FIL_PAGE_TYPE_FSP_HDR
  │    ├─ 写 FSP Header 全部字段
  │    └─ fsp_fill_free_list()   // 初始化 extent 0 的 XDES：
  │          Page 0 (FSP_HDR, 已用) → extent 0 进 FSP_FREE_FRAG
  │          Page 1 (IBUF_BITMAP, 已用)
  └─ btr_sdi_create_index()      // ★ 建 SDI B-tree（mysql.ibd 自带 SDI）
       └─ fsp_sdi_write_root_to_page()  // 把 SDI 根页号写回 Page 0
```

Page 0 的完整布局（16K 页；本工作副本 `dict0dict.h:1110` 有现成的中文注释图，可对照）：

```
[0, 38)      FIL Header（38B）
[38, 150)    FSP Header（112B）   ← 逐字段偏移见 page_structure.md
[150, 10390) XDES Array（256 × 40B）  ← 描述 extent 0~255
[10390, ...) Encryption Info（约 260B）
[≈10654, +8) SDI Header（8B）= SDI_VERSION(4) + SDI_ROOT_PAGE_NUM(4)
[≈10662, +5) Encryption Progress Info
...
[16376, 16384) FIL Trailer（8B）
```

页用途：Page 0 = FSP_HDR、Page 1 = IBUF_BITMAP、Page 2 = 第一个 INODE 页、`Page 3+` = SDI B-tree 根页与各 DD 表的聚簇/二级索引根页及数据页。

★ **mysql.ibd 里住着两套表**：

| 归属 | 张数 | 例子 |
|---|---|---|
| **MySQL DD 表** | 30（INERT 1 + CORE 22 + SECOND 7） | `tables` / `columns` / `indexes` / `schemata` / **`dd_properties`** …（逐张见 [`dd.md`](dd.md)；★ `dd_properties` 是 MySQL 的 INERT DD 表，不是 InnoDB 自有） |
| **InnoDB 自有 DDSE 表** | 4 | `innodb_dynamic_metadata`(1) / `innodb_table_stats`(1) / `innodb_index_stats`(1) / `innodb_ddl_log`(2，括号内为索引数) |

`dict0dd.h:294-330` 的 `innodb_dd_table[]` 把这张清单**按名字 + 索引数硬编码**，供 `dict_sys_t::s_dd_table_ids`（`dict0dict.cc:142`）与 `is_dd_table_id()` 在崩溃恢复早期（DD 尚未就绪）按 space_id 直接定位。

#### 动态元数据：DD 之外的一小块持久状态

有些 InnoDB 私有状态**不适合放 DD**（改得太频繁：每次插入都要动），于是单独走"动态元数据"通道——`mysql.innodb_dynamic_metadata` + redo。典型内容：**autoinc 计数器**（`PM_TABLE_AUTO_INC`）、**索引损坏标志**（`PM_INDEX_CORRUPTED`）。

三态机（`table_dirty_status`）：

```
METADATA_CLEAN ──(mark_dirty)──► METADATA_DIRTY ──(persist_to_dd_table_buffer)──► METADATA_BUFFERED
       ▲                                                                                  │
       └──────────────────────(checkpoint 回写完成)──────────────────────────────────────┘
```

**读方向：`dict_table_load_dynamic_metadata()`**（`dict0dict.cc:4080`，加载表时调用）：

```cpp
void dict_table_load_dynamic_metadata(dict_table_t *table) {
  ut_ad(dict_sys != nullptr);
  ut_ad(dict_sys_mutex_own());        // ① 调用者必须已持 dict_sys->mutex
  ut_ad(!table->is_temporary());

  DDTableBuffer *table_buffer = dict_persist->table_buffer;
  mutex_enter(&dict_persist->mutex);

  uint64_t version;
  const auto readmeta = table_buffer->get(table->id, &version);   // ② 先查内存 buffer

  if (!readmeta.empty()) {
    PersistentTableMetadata metadata(table->id, version);
    dict_table_read_dynamic_metadata(readmeta.data(), readmeta.size(), &metadata);
    bool is_dirty = dict_table_apply_dynamic_metadata(table, &metadata);

    /* 若 !is_dirty，两种可能：
       1. 首次加载该表，且"损坏索引标记"指向的索引已被 drop → 当前应是 METADATA_CLEAN
       2. 第二次应用，内存态已是最新 → 当前应是 METADATA_BUFFERED
       两种情况都不必改 dirty_status */
    if (is_dirty) {
      UT_LIST_ADD_LAST(dict_persist->dirty_dict_tables, table);
      table->dirty_status.store(METADATA_BUFFERED);                // ③ 注意是 BUFFERED 不是 DIRTY
      ut_d(table->in_dirty_dict_tables_list = true);
    }
  }
  mutex_exit(&dict_persist->mutex);
}
```

★ ③ 是个易忽略的细节：加载后状态置 **`METADATA_BUFFERED`**（而非 `DIRTY`）——因为这份数据**刚从持久层读出来、与磁盘一致**，只是"仍在脏链里待命"。标记成 DIRTY 会白写一次盘。

另一处值得注意：`dict_table_apply_dynamic_metadata` 返回的 `is_dirty` 决定要不要挂链，注释里把"首次加载 + 损坏索引已 drop"与"重复加载"两种不挂的情况分开讲清楚——**这是"索引损坏标记"（`PM_INDEX_CORRUPTED`）与 autoinc 共用同一通道造成的特殊分支**。

```cpp
// dict0dict.cc:4125
void dict_table_mark_dirty(dict_table_t *table) {
  ut_ad(!table->is_temporary());
  /* 关闭的 flush 阶段之后不再登记——这些数据只在 recovery 时有用 */
  ut_ad(srv_shutdown_state.load() < SRV_SHUTDOWN_FLUSH_PHASE);

  mutex_enter(&dict_persist->mutex);
  switch (table->dirty_status.load()) {
    case METADATA_DIRTY:
      break;                                    // 已在脏链，无需操作
    case METADATA_CLEAN:
      UT_LIST_ADD_LAST(dict_persist->dirty_dict_tables, table);   // ① 挂进脏链
      ut_d(table->in_dirty_dict_tables_list = true);
      [[fallthrough]];
    case METADATA_BUFFERED:
      table->dirty_status.store(METADATA_DIRTY);
      ++dict_persist->num_dirty_tables;
      dict_persist_update_log_margin();         // ★ ② 约束 checkpoint
  }
  ut_ad(table->in_dirty_dict_tables_list);
  mutex_exit(&dict_persist->mutex);
}
```

★ ② `dict_persist_update_log_margin()` 是整套机制的**关键联动**：它压低 `log_sys->dict_max_allowed_checkpoint_lsn`，确保"标注脏的这段 redo"不会被 checkpoint 丢掉。这正是 [`auto_increment.md`](../../feat/auto_increment.md) 里 `dict_table_autoinc_log` 那段长注释（"the redo logs for current change won't be counted into current checkpoint…"）所指的东西——两处从不同角度描述同一件事。

回写（`dict_table_persist_to_dd_table_buffer`，`:4230`）先做**双重检查**（并发 checkpoint 可能已把状态改成非 DIRTY，直接返回），再 `dict_table_persist_to_dd_table_buffer_low()` 写 `mysql.innodb_dynamic_metadata`。

三个调用时机：

| 时机 | 位置 | 说明 |
|---|---|---|
| DML 之后 | `row0ins.cc:2601` / `row0upd.cc:2986` | autoinc 推进后立即登记 |
| **表被淘汰时** | `dict0dict.cc:1903`（`dict_table_remove_from_cache_low` 内） | ★ 淘汰前必须回写，否则内存态丢失 |
| checkpoint | `dict_persist_to_dd_table_buffer()`（`:4250`，遍历 `dirty_dict_tables`） | 批量刷 |

★ 这解释了一个反直觉现象：**表被从 dict cache 淘汰时还要写一次磁盘**——因为动态元数据只活在内存 + 这张表里，淘汰即"最后一次保存机会"。

#### ★ 持久化走 redo，不走上层表：`MLOG_TABLE_DYNAMIC_META`

动态元数据为什么不直接写 `mysql.innodb_dynamic_metadata` 表，而是先写 redo？因为**写这张表要拿表锁，而"元数据变化"发生在引擎最底层**（插入一行就要推进 autoinc）——底层代码持有上层系统表的锁，极易与 DDL 形成环。redo 是引擎最底层的设施，任何位置都能写，不存在锁层次问题（月报 2019/12 的原文表述）。

于是 8.0 引入一个专用逻辑 redo 类型：

```cpp
// mtr0types.h:247
MLOG_TABLE_DYNAMIC_META = 62,
```

写入（`Persister::write_log`，`dict0dict.cc:5480`）：

```cpp
void Persister::write_log(table_id_t id,
                          const PersistentTableMetadata &metadata,
                          mtr_t *mtr) const {
  ulint size = get_write_size(metadata);
  /* Both table id and version would be written in a compressed format,
  each of which would cost 1..11 bytes, and MLOG_TABLE_DYNAMIC_META costs
  1 byte. */
  static constexpr uint8_t metadata_log_header_size = 23;

  if (!mlog_open_metadata(mtr, metadata_log_header_size + size, log_ptr))
    return;   // 全局 redo 关闭时（如部分恢复场景）直接跳过

  log_ptr = mlog_write_initial_dict_log_record(
      MLOG_TABLE_DYNAMIC_META, id, metadata.get_version(), log_ptr, mtr);
  ulint consumed = write(metadata, log_ptr, size);   // 各 persister 自己的序列化
  mlog_close(mtr, log_ptr);
}
```

★ **压缩设计**：table id 与 version 都是变长编码（1..11 字节），autoinc 计数器这种高频写入的元数据，一次 mtr 只多几十字节。`Persister` 是按类型分发的抽象——`AutoIncPersister` / `CorruptedIndexPersister` / `UpdateTimePersister` 各自实现 `write()`（序列化）与 `read()`（反序列化）。

`CorruptedIndexPersister::write`（`:5515`）的格式就是个例子：

```cpp
mach_write_to_1(buffer, PM_INDEX_CORRUPTED);   // 标记
mach_write_to_1(buffer, num);                  // 损坏索引数
for (...) { mach_write_to_4(space_id); mach_write_to_8(index_id); }  // 12B/个
```

#### ★ checkpoint 刷数据页，不刷元数据

一个常被忽略的事实：**checkpoint 只负责把脏页刷盘，不负责元数据**。autoinc 等动态元数据的变化只写在 redo + 内存里，checkpoint 推进后这部分 redo 就没有了——所以崩溃恢复时**元数据状态只能靠"从某起点 apply redo"一步步重建**。

这正是 **DD Buffer Table（`mysql.innodb_dynamic_metadata`）存在的理由**：它把"上次 checkpoint 以来的活跃元数据"**在 checkpoint 时快照一份**（`dict_persist_to_dd_table_buffer()`），于是：

```
没有 buffer table：恢复要从"上次完整状态"apply 全部 redo
有 buffer table：  恢复 = 读 DD 表（表结构）+ 读 buffer table（元数据快照）
                   + apply 快照之后的 redo（短）
```

★ **DD Buffer Table 与 checkpoint 的分工**：checkpoint 是**数据页**的快照，DD Buffer Table 是**活跃元数据**的快照。有了后者，recovery 才可以从 checkpoint 开始而不是从头。

恢复四步（对应 iwiki 那篇的流程）：

```
1. 从 DD 系统表读表结构（mysql.tables 等，正常路径）
2. 从 DD Buffer Table 读动态元数据快照（innodb_dynamic_metadata）
3. 应用 checkpoint 之后的 redo（MLOG_TABLE_DYNAMIC_META 解析）
4. 两者合并重建 dict_table_t 的完整内存状态
```

**redo 端的实现**（`log0recv.cc:2867`）值得注意：`MLOG_TABLE_DYNAMIC_META` **不走页级 apply**：

```cpp
case MLOG_TABLE_DYNAMIC_META:
  *page_no = FIL_NULL; *space_id = SPACE_UNKNOWN;   // 不是页日志
  new_ptr = mlog_parse_initial_dict_log_record(ptr, end_ptr, type, &id, &version);
  new_ptr = recv_sys->metadata_recover->parseMetadataLog(id, version, new_ptr, end_ptr);
  // → 解析进内存 buffer（dict_persist->table_buffer），等表加载时被
  //   dict_table_load_dynamic_metadata() 消费
```

即：**recovery 扫描 redo 时直接把元数据解析进内存 `table_buffer`**（一个 `table_id → metadata` 的 map），后续加载每张表时由 `dict_table_load_dynamic_metadata()` 取用——这就是上文"读方向"的输入来源。整条链是：

```
写入：dict_table_mark_dirty → (mtr) Persister::write_log → redo
快照：checkpoint 时 dict_persist_to_dd_table_buffer → mysql.innodb_dynamic_metadata
恢复：redo 扫描 parseMetadataLog → table_buffer → dict_table_load_dynamic_metadata
```

#### `mysql.innodb_ddl_log`：物理资源的原子性

DD 解决的是**元数据**的原子性（server 层 DD 事务），但 DDL 还会动**物理资源**（`.ibd` 文件、B+ 树、表空间）——这些不归 DD 事务管。于是有了第三样东西：

★ **两种原子性分工**：

| | 保障对象 | 机制 |
|---|---|---|
| **DD 事务** | 元数据（表/列/索引定义） | server 层 DD 表写入，随事务提交/回滚 |
| **`innodb_ddl_log`** | 物理资源（文件、索引树、缓存项） | 把"已做的物理动作"记入一张 InnoDB 表，崩溃后回放 |

**`Log_Type` 共 8 种**（`log0ddl.cc`，`SMALLEST_LOG` ~ `BIGGEST_LOG` 之间）：

| Log_Type | 含义 | 回放时要做什么 |
|---|---|---|
| `FREE_TREE_LOG` | 已建好一棵索引树 | 释放这棵树（建索引中途崩溃） |
| `DELETE_SPACE_LOG` | 已创建一个表空间文件 | 删除该 `.ibd` 文件 |
| `RENAME_SPACE_LOG` | 已改名表空间 | 改回原名 |
| `DROP_LOG` | 已删除 | 继续完成删除 |
| `RENAME_TABLE_LOG` | 已 RENAME | 改回 |
| `REMOVE_CACHE_LOG` | 已把表放进 dict cache | 从 cache 移除 |
| `ALTER_ENCRYPT_TABLESPACE_LOG` / `ALTER_UNENCRYPT_TABLESPACE_LOG` | 已改加密状态 | 反向改回 |

**为什么它能做到原子**：`innodb_ddl_log` 本身就是一张 InnoDB 表，写入走普通事务——所以它**与 DD 事务同生共死**：

```
DDL 执行中：  物理动作 → 插一条 DDL_Record（与 DD 同事务提交）
DDL 成功：    DDL_Log_Table::remove(records)  删除这些记录
DDL 崩溃：    记录残留 → 重启后 Log_DDL::recover() 扫描并回放
```

实现要点（`log0ddl.cc`）：

- `DDL_Log_Table` 构造时直接取 `dict_sys->ddl_log`（常驻指针，不走哈希查找——DD 可能还没就绪）；
- 插入用 **`BTR_NO_LOCKING_FLAG`**（不加记录锁：DDL 期间表已被 MDL 保护，无并发 DML），且每 64 次 `log_free_check()` 防 redo 打满（`log0ddl.cc:470`）；
- 一次 DDL 的多条记录靠 **id 串联**：`search(thread_id, …)` 先用二级索引（`first_index()->next()`）按 thread_id 查出所有 id，再用聚簇索引按 id 取回完整记录（`:666`）；
- 记录字段：`id` / `thread_id` / `type` / `space_id` / `page_no` / `index_id` / `table_id` / `old_file_path` / `new_file_path`。

★ 一句话概括：**DD 让"元数据"原子，DDL log 让"文件和树"原子，两者靠同一事务提交绑定**。这也是为什么 8.0 能宣称"crash-safe DDL"。

##### 回放与幂等性（`Log_DDL::recover` / `replay_all` / `post_ddl`）

三个入口对应三种时机：

| 入口 | 时机 | 行为 |
|---|---|---|
| `recover()`（`log0ddl.cc:1953`） | **崩溃恢复** | `replay_all()`：`search_all` 扫全部残留记录 → 逐个 `replay()` → 成功后 `delete_by_ids` 删除 |
| `post_ddl()`（`:1906`） | **正常 DDL 提交后** | ★ `replay_by_thread_id(thread_id)`——**只回放本线程的记录**（`thread_local_ddl_log_replay = true`），把"删树/删文件"这类收尾动作真正执行掉 |
| `Log_DDL::shutdown` | 关闭 | 清理 |

★ **幂等性设计**：回放成功即删记录（`delete_by_ids`），失败留下记录等下次重启再试——**残留记录 = 未完成的收尾动作**，回放多少次都以"删到没有"为止。

一个特殊的反例（`:1799`）：加密相关记录 `record.set_deletable(false)`——**故意保留不回放即删**：

```cpp
/* Make sure not to delete this record till resume operation finishes.
   This is to make sure that if there is a crash before that, we can
   resume encryption in the next restart. */
record.set_deletable(false);
```

因为"加密 tablespace"要交给后台线程异步完成，记录必须活到加密真正完成；中途崩溃也能在下一次重启 resume。**可删除性（deletable）本身是幂等协议的一部分**。

`replay()` 对每条记录做的是"**把物理世界恢复到记录期望的状态**"：`FREE_TREE_LOG` → 若树还在就删、已删则跳过（幂等）；`DELETE_SPACE_LOG` → 若文件还在就删。每条 replay 都是"检查-执行"而非"盲目重放"——这正是幂等的实现方式。

---

## 9. 与其它模块的接口

| 对接方 | 接口 |
|---|---|
| **handler / prebuilt** | `row_prebuilt_t::table` / `index`；`row_create_prebuilt()`；`ha_innobase::open` 里 `m_share` 必须在 `dict_table_t` 之后分配、之前释放（注释：`m_share` 持有 index 指针而无 pin） |
| **锁** | `dict_table_t::locks` / `count_by_mode[LOCK_NUM]` / `n_rec_locks`；`lock_table_has_locks()`（`lock0lock.cc:6297`）决定能否淘汰；autoinc 用 `autoinc_lock` 与 `count_by_mode[LOCK_AUTO_INC]` |
| **事务** | `trx_t::mod_tables`（本事务改过的表集合，注释：表对象在事务运行期间不会被销毁）；`dict_index_t::trx_id` 供 `is_usable()` 做 MVCC 判断 |
| **redo / 动态元数据** | `dict_persist_t` + `Persister` 族（`CorruptedIndexPersister` / `AutoIncPersister`），写 `MLOG_TABLE_DYNAMIC_META`，最终落 `mysql.innodb_dynamic_metadata`（`DDTableBuffer`）；`dirty_status` 三态 |
| **DDL log** | `Log_DDL::write_free_tree_log(dict_index_t*)` / `write_drop_log(table_id)` / `write_remove_cache_log(dict_table_t*)` 等（`log0ddl.cc`），参数直接是 dict 对象 |
| **统计** | `dict_stats_update()`；持久化存 `mysql.innodb_table_stats` / `innodb_index_stats`；触发点 `row_update_statistics_if_needed()` |

---

## Misc

### 容易误解的概念

- **dict 不是持久化的真相**：除 ID 计数器与"动态元数据"外不写 redo，崩溃即丢，从 DD 重建；
- **`dict_table_t` 不等于一张"表"**：分区表每个分区一个；
- **8.0 的 `SYS_*` 只是升级兼容**，不要拿 5.x 资料当 8.0 现状；
- **`dict_sys` 未分片**（只有一把 mutex），元数据极多时可能成为瓶颈——这与 server 层 TDC 同样未分片形成呼应；
- **启动不重建缓存**，是惰性构建。

### 待补清单

（全部完成，暂无待补）

## 参考

- *MySQL 8.0 Reference Manual → InnoDB and MySQL Data Dictionary*
- 源码：`storage/innobase/dict/`（`dict0boot.cc` / `dict0dd.cc` / `dict0load.cc` / `dict0dict.cc` / `dict0mem.cc` / `dict0stats.cc`）
- [InnoDB：DDL（1）— Skywalker](https://zhuanlan.zhihu.com/p/446044092)
- 内核月报：[MySQL 中的元数据管理](http://mysql.taobao.org/monthly/2023/10/03/)（dict cache 的 LRU/non_LRU 部分） / [MySQL 表定义缓存](http://mysql.taobao.org/monthly/2015/08/10/)
- 内核月报：[动态元信息持久化](http://mysql.taobao.org/monthly/2019/12/01/)（autoinc 持久化背景） / [DDL log 与原子 DDL](http://mysql.taobao.org/monthly/2021/07/01/)
