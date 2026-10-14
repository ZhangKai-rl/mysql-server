# InnoDB 统计信息

## 概述

### 是什么

InnoDB 统计信息是**优化器估算代价的输入**（行数、索引区分度 `n_diff`、聚簇索引大小等）。它既不是纯内存态、也不完全由 DD 承载，而是横跨三层：

```
采集（dict0stats.cc）  →  内存（dict_table_t / dict_index_t）  →  持久化（mysql.innodb_*_stats）
                              ↓ 供优化器读
                     handler::info() → records_in_range() 等
```

### 为什么放在 `server/dd/` 目录下

> **边界**：本篇是 **DD/持久化视角**——统计存在哪、表级开关怎么存、后台重算与持久化表的关系。★ **采集算法本身**（persistent 分层采样 vs transient 随机页采样、`n_diff_pfxNN`、10% 重算判据、基数漂移根因）的权威剖析在 [`../../index/stats.md`](../../index/stats.md)（索引是第一主语），两处互指不重复详写。

它跟 DD 有两处硬关联：

1. **持久统计存在 `mysql.innodb_table_stats` / `mysql.innodb_index_stats`**——这两张是 **InnoDB 的 DDSE_PROTECTED 表**，物理上就在 `mysql.ibd` 里（见 [`dd.md`](dd.md) 的 30 张 DD 表 + InnoDB 自有 5 张）；
2. **表级统计开关存在 `dd::Table::options()` 里**——`stats_auto_recalc` / `stats_sample_pages` 是表的持久化属性，不是纯运行参数（见下文）。

所以它既属于 DD（持久化位置），又属于引擎（采集算法）。放在这里便于两面对照。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.6 以前 | 只有**瞬态**统计（`stat_n_rows` 采样估算，内存态，重启丢失、每次开表重算） |
| 5.6 | 引入**持久统计**（`innodb_table_stats` / `innodb_index_stats`），`innodb_stats_persistent` 默认开 |
| 8.0 | 统计表迁入 `mysql.ibd`（随 DD 一起）；表级属性进 `dd::Table::options()` |

---

## 1. ★ 四套"统计"的辨析（最易混）

MySQL 里名字带 stats 的东西有四套，**完全不是一回事**：

| 名字 | 是什么 | 谁在用 | 存放位置 |
|---|---|---|---|
| `mysql.innodb_table_stats` / `innodb_index_stats` | ★ **InnoDB 持久统计**（真正生效的那个） | InnoDB | `mysql.ibd`（DDSE_PROTECTED 表） |
| `mysql.table_stats` / `index_stats` | **DD 层**的统计缓存（SECOND 表） | DD 模块 | `mysql.ibd`（DD SECOND 表） |
| `dict_table_t::stat_n_rows` 等 | **运行时内存统计** | 优化器实时读 | 内存 |
| `mysql.column_statistics` | **直方图**（列级分布，另一套机制） | 优化器 | `mysql.ibd`（DD 表） |

⚠️ 排查统计信息问题时**第一件事就是确认看的是哪一套**。`SHOW INDEX` 的 `Cardinality` 来自内存统计（实时估算，每次调用可能变），而 `mysql.innodb_index_stats` 里是持久化的值。

**★ 表名实证**（两套表只差 `innodb_` 前缀，极易看错）：

```cpp
/* storage/innobase/dict/dict0stats.cc —— InnoDB 自己的持久化表 */
#define TABLE_STATS_NAME "mysql/innodb_table_stats"
#define TABLE_STATS_NAME_PRINT "mysql.innodb_table_stats"
#define INDEX_STATS_NAME "mysql/innodb_index_stats"
#define INDEX_STATS_NAME_PRINT "mysql.innodb_index_stats"
```

```cpp
/* sql/dd/impl/tables/table_stats.cc —— DD 的 I_S 统计缓存表（另一套！） */
Table_stats::Table_stats() {
  m_target_def.set_table_name("table_stats");
  m_target_def.add_field(FIELD_TABLE_ROWS, "FIELD_TABLE_ROWS", "table_rows BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_AVG_ROW_LENGTH, "FIELD_AVG_ROW_LENGTH", "avg_row_length BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_DATA_LENGTH, "FIELD_DATA_LENGTH", "data_length BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_INDEX_LENGTH, "FIELD_INDEX_LENGTH", "index_length BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_AUTO_INCREMENT, "FIELD_AUTO_INCREMENT", "auto_increment BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_CHECKSUM, "FIELD_CHECKSUM", "checksum BIGINT UNSIGNED");
  m_target_def.add_field(FIELD_CACHED_TIME, "FIELD_CACHED_TIME", "cached_time TIMESTAMP NOT NULL");
  m_target_def.add_index(INDEX_PK_SCHEMA_ID_TABLE_NAME, "INDEX_PK_SCHEMA_ID_TABLE_NAME",
                         "PRIMARY KEY (schema_name, table_name)");
}
```

`mysql.table_stats` 的字段是 `table_rows` / `data_length` / `auto_increment` ——**明显是 `SHOW TABLE STATUS` 那一套**，跟 InnoDB 的 `n_rows` / `clustered_index_size` 不是同一组数据。

**谁写谁读**（每一项都有代码位置作证）：

| # | 归属 | 存储 | 写入方 | 读取方 |
|---|---|---|---|---|
| ① | InnoDB 内存 dict_stats | `dict_table_t::stat_n_rows`、`dict_index_t::stat_n_diff_key_vals[]` | `dict_stats_update_persistent()` / `dict_stats_update_transient()` | `dict_table_get_n_rows()`、`ha_innobase::info()` |
| ② | InnoDB 持久统计 | `mysql.innodb_table_stats` / `innodb_index_stats` | `dict_stats_save()`（内部 SQL） | `dict_stats_fetch_from_ps()`（内部 SQL） |
| ②′ | **DD 的 I_S 缓存**（与 ② 不通！） | `mysql.table_stats` / `index_stats` | `dd::info_schema::update_table_stats()` | `Table_statistics::read_stat()` |
| ③ | server 直方图 | `mysql.column_statistics`（JSON 列） | `Histogram::store_histogram()`（DD client） | `histograms::find_histogram()` |
| ④ | 临时表/优化器的行数估算 | 进程内存 `handler::stats.records`，不落盘 | `table->file->stats.records = ...` | 代价模型 |

> ★ ② 与 ②′ **互相不通**：InnoDB 用**自己的内部 SQL parser** 读写 `mysql.innodb_*_stats`；DD 用 **Dictionary_client** 读写 `mysql.table_stats` / `index_stats`。而且两者注册层级不同——`Column_statistics` 是 **core** 层 DD 表，`Table_stats` / `Index_stats` 是 **second** 层（`sql/dd/impl/system_registry.cc`）。

② 的读回路径（内部 SQL + 游标，读不到就返回 `DB_STATS_DO_NOT_EXIST`）：

```cpp
static dberr_t dict_stats_fetch_from_ps(dict_table_t *table) {
  /* Initialize all stats to dummy values before fetching because if
     the persistent storage contains incomplete stats (e.g. missing stats
     for some index) then we would end up with (partially) uninitialized
     stats. */
  dict_stats_empty_table(table);

  trx = trx_allocate_for_background();
  /* Use 'read-uncommitted' so that the SELECTs we execute
     do not get blocked in case some user has locked the rows we
     are SELECTing */
  trx->isolation_level = TRX_ISO_READ_UNCOMMITTED;
  ...
  ret = que_eval_sql(pinfo,
                     "PROCEDURE FETCH_STATS () IS\n"
                     "DECLARE CURSOR table_stats_cur IS\n"
                     "  SELECT n_rows, clustered_index_size, sum_of_other_index_sizes\n"
                     "  FROM \"mysql/innodb_table_stats\"\n"
                     "  WHERE database_name = :database_name AND table_name = :table_name;\n"
                     ...
  if (!index_fetch_arg.stats_were_modified) {
    return (DB_STATS_DO_NOT_EXIST);
  }
```

> ★ 注意 `TRX_ISO_READ_UNCOMMITTED`——统计读取**故意用脏读**，避免被用户锁阻塞。这是"统计不是关键数据"的直接体现。

另：直方图（`ANALYZE TABLE ... UPDATE HISTOGRAM`）是**独立于 InnoDB 统计**的一套机制，存在 `mysql.column_statistics`（详见第 4 节末尾的协同分析）。

---

## 2. 表级统计属性存在 DD 的 `options` 里

`stats_auto_recalc` 与 `stats_sample_pages` 是**表属性**，随 `CREATE TABLE` 的 `STATS_AUTO_RECALC` / `STATS_SAMPLE_PAGES` 子句设定，最终落到 `dd::Table::options()`：

```cpp
// dict0dd.cc:6440 —— 从 InnoDB 写回 dd::Table 的 options
table_options->set("stats_auto_recalc", HA_STATS_AUTO_RECALC_DEFAULT);
// ... stats_sample_pages 同理
```

加载时反向读出（`dict0dd.cc:3146`）：

```cpp
dict_stats_auto_recalc_set(
    table, table_share->stats_auto_recalc == HA_STATS_AUTO_RECALC_ON,
    table_share->stats_auto_recalc == HA_STATS_AUTO_RECALC_OFF);
table->stats_sample_pages = table_share->stats_sample_pages;
```

★ 这正好是 [`dd.md`](dd.md#23--options-vs-se_private_data) 里 `options` vs `se_private_data` 分类的一个实例：**统计开关是"引擎理解但由 server 定义"的属性 → 进 `options`**（不是 `se_private_data`，因为它由 SQL 层 `CREATE TABLE` 语法直接指定）。

三态语义（`HA_STATS_AUTO_RECALC_ON/OFF/DEFAULT`）：`DEFAULT` 表示跟随全局变量 `innodb_stats_auto_recalc`。

---

## 3. `dict_stats_update()`：四种更新模式

`dict_stats_update(table, stats_upd_option)`（`dict0stats.cc:2817`）是统计更新的总入口。

**入口处的两个前置检查**（容易被忽略）：

```cpp
if (table->ibd_file_missing) {
  if (!dict_table_is_discarded(table)) {
    ib::warn(ER_IB_MSG_224) << "Cannot calculate statistics for table "
        << table->name << " because the .ibd file is missing. " << ...;
  }
  dict_stats_empty_table(table);
  return DB_TABLESPACE_DELETED;
} else if (srv_force_recovery >= SRV_FORCE_NO_IBUF_MERGE) {
  /* 高 innodb_force_recovery 下不计算统计——损坏的索引可能让它直接崩 */
  dict_stats_empty_table(table);
  return DB_SUCCESS;
}
```

★ 第二条的注释很实在：*a badly corrupted index can cause a crash in it*——统计采集要遍历 B+ 树，**强制恢复模式下宁可给空统计也不冒险遍历**。

### 四种 `dict_stats_upd_option_t`

| 模式 | 触发者 | 行为 |
|---|---|---|
| `DICT_STATS_RECALC_PERSISTENT` | `ANALYZE TABLE` / 后台自动重算 / 开表时统计不存在且 auto recalc 开 | 真采一遍 → 存盘 |
| `DICT_STATS_RECALC_TRANSIENT` | 开表时的瞬态统计 | `break`（走后面的瞬态逻辑） |
| `DICT_STATS_EMPTY_TABLE` | 空表 | 置空；若开了持久统计也存盘 |
| `DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY` | 开表 | **只从持久存储读**，不重算 |

枚举定义的原文（`storage/innobase/include/dict0stats.h`）：

```cpp
enum dict_stats_upd_option_t {
  DICT_STATS_RECALC_PERSISTENT, /* (re) calculate the
                           statistics using a precise and slow
                           algo and save them to the persistent
                           storage, if the persistent storage is
                           not present then emit a warning and
                           fall back to transient stats */
  DICT_STATS_RECALC_TRANSIENT,  /* (re) calculate the statistics
                            using an imprecise quick algo
                            without saving the results
                            persistently */
  DICT_STATS_EMPTY_TABLE,       /* Write all zeros (or 1 where it makes sense)
                                into a table and its indexes' statistics
                                members. The resulting stats correspond to an
                                empty table. If the table is using persistent
                                statistics, then they are saved on disk. */
  DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY /* fetch the stats
                          from the persistent storage if the in-memory
                          structures have not been initialized yet,
                          otherwise do nothing */
};
```

> ⚠️ **第 4 个成员是 `DICT_STATS_EMPTY_TABLE`，不是 `NO_RECALC`**（后者是旧资料里的名字）。"不重算"这一语义由 `FETCH_ONLY_IF_NOT_IN_MEMORY` 里的 `if (table->stat_initialized) return DB_SUCCESS;` 短路表达，不是一个独立枚举值。

各分支的真实调用点（grep 实证）：

| 模式 | 调用点 |
|---|---|
| `RECALC_PERSISTENT` | `dict0stats_bg.cc:320`（后台线程）、`ha_innodb.cc:15250`（ANALYZE）、`handler0alter.cc:7386` |
| `RECALC_TRANSIENT` | `row0mysql.cc:1160` |
| `EMPTY_TABLE` | `ha_innodb.cc:9980`、`:14001` |
| `FETCH_ONLY_IF_NOT_IN_MEMORY` | `dict0stats.ic:155`（`dict_stats_init()`，开表） |

开表入口只有 6 行，判据就是"表的持久统计开关"：

```cpp
/* storage/innobase/include/dict0stats.ic —— dict_stats_init() */
static inline void dict_stats_init(dict_table_t *table) {
  if (table->stat_initialized) return;

  dict_stats_upd_option_t opt;
  if (dict_stats_is_persistent_enabled(table)) {
    opt = DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY;   // ★ 持久：从盘上读
  } else {
    opt = DICT_STATS_RECALC_TRANSIENT;              // ★ 瞬态：就地采一遍
  }
  dict_stats_update(table, opt);
}
```

#### `RECALC_PERSISTENT`（真采一遍）

```cpp
if (srv_read_only_mode) break;

/* wakes the last purge batch for exact recalculation */
if (trx_sys->rseg_history_len.load() > 0) {
  srv_wake_purge_thread_if_not_active();       // ★①
}

/* InnoDB internal tables (e.g. SYS_TABLES) cannot have persistent stats */
ut_a(strchr(table->name.m_name, '/') != nullptr);   // ★②

err = dict_stats_update_persistent(table);
if (err != DB_SUCCESS) return err;
return dict_stats_save(table, nullptr);              // ③ 存盘
```

★① **采集前先唤醒 purge**——注释说 "for exact recalculation"：history list 非空意味着还有未 purge 的删除标记，不清干净会让统计偏大。这是"统计准确性"与"purge 进度"的一次耦合。

★② `ut_a(strchr(name, '/') != nullptr)`：**InnoDB 内部表（SYS_TABLES 等）不能有持久统计**——它们的名字里没有 `/`（不含 schema 前缀），这个断言就是在守护这条规则。

#### `FETCH_ONLY_IF_NOT_IN_MEMORY`（只读取）

```cpp
if (table->stat_initialized) return DB_SUCCESS;      // 内存已有，直接用

dict_table_t *t = dict_stats_table_clone_create(table);  // ★③ 造一个 dummy 表
err = dict_stats_fetch_from_ps(t);                        // 从持久存储读到 clone 里

switch (err) {
  case DB_SUCCESS:
    dict_table_stats_lock(table, RW_X_LATCH);
    dict_stats_copy(table, t);                            // 拷回真身
    dict_table_stats_unlock(table, RW_X_LATCH);
    dict_stats_table_clone_free(t);
    return DB_SUCCESS;
  case DB_STATS_DO_NOT_EXIST:
    dict_stats_table_clone_free(t);
    if (srv_read_only_mode) break;
    if (dict_stats_auto_recalc_is_enabled(table))
      return dict_stats_update(table, DICT_STATS_RECALC_PERSISTENT);  // ★④ 降级重算
    ib::info(ER_IB_MSG_225) << ... "auto recalculation turned off ..."
        "InnoDB will now use transient statistics";
    break;
}
```

★③ **为什么要 clone 一个 dummy 表**：读取过程可能失败（`DB_STATS_DO_NOT_EXIST`），直接往真身上读会把它搞成"半初始化"状态。用 clone 承接读取结果，**成功才在 `stats_lock` 保护下拷贝过去**——这是典型的"先试后提交"写法。

★④ 持久统计不存在 + auto recalc 开 → **递归调 `RECALC_PERSISTENT`** 现场采一遍；若 auto recalc 关，则打印提示并**回退到瞬态统计**。

---

## 4. 采集算法

### 采样页数

```cpp
// dict0stats.cc:131
#define N_SAMPLE_PAGES(index)                                    \
  static_cast<uint64_t>((index)->table->stats_sample_pages != 0  \
                            ? (index)->table->stats_sample_pages \
                            : srv_stats_persistent_sample_pages)
```

即：**表级 `STATS_SAMPLE_PAGES` 优先，否则用全局 `innodb_stats_persistent_sample_pages`**。

### 逐层下降采样

`dict_stats_analyze_index_low()`（`:1668`）从根层开始逐层下降，直到某层的"不同记录数"达到阈值（:136 的注释：*number of distinct records on a given level that are required to stop descending*），然后在该层采样 `N_SAMPLE_PAGES` 个页：

```
dict_stats_analyze_index()            :1987  ── 外层重试循环
  └─ dict_stats_analyze_index_low()   :1668  ── 选层 + 采样
       ├─ dict_stats_analyze_index_level()       :793   统计某层的 n_diff / 总记录 / 总页数
       └─ dict_stats_analyze_index_for_n_prefix():1425  对每个前缀长度算 n_diff
            └─ dict_stats_analyze_index_below_cur() :1240  在游标下方继续扫
```

`dict_stats_analyze_index()` 外层有个**重试循环**：

```cpp
uint64_t n_sample_pages = N_SAMPLE_PAGES(index);
while (n_sample_pages > 0 && !dict_stats_analyze_index_low(n_sample_pages, index)) {
  /* aborted. retrying. */
  ib::warn(ER_IB_MSG_STATS_SAMPLING_TOO_LARGE)
      << "Detected too long lock waiting around " << index->table->name << ".";
  ...
}
```

★ 采集要持锁扫 B+ 树，若等待过久会被中断；**中断后自动重试**（并可通过减小采样页数退让）。这解释了为什么大表 `ANALYZE` 可能耗时很长且日志里出现 `STATS_SAMPLING_TOO_LARGE` 警告。

### 估算内容

| 统计量 | 含义 |
|---|---|
| `stat_n_rows` | 表行数（估算值，非精确） |
| `stat_clustered_index_size` / `stat_sum_of_other_index_sizes` | 聚簇/二级索引占用页数 |
| `stat_n_diff_key_parts[]` | 各前缀长度的**不同值个数**（`n_diff`） |
| `stat_index_size` / `stat_n_leaf_pages` | 索引大小与叶子页数 |

**表级三个量是从聚簇索引"搬"过来的**（`dict_stats_update_persistent()`）——这是理解 n_rows 为什么不精确的关键：

```cpp
/* storage/innobase/dict/dict0stats.cc —— dict_stats_update_persistent() */
  dict_stats_analyze_index(index);                    // ① 先分析聚簇索引

  ulint n_unique = dict_index_get_n_unique(index);

  table->stat_n_rows = index->stat_n_diff_key_vals[n_unique - 1];   // ★★ 行数 = 主键去重后的不同值数
  table->stat_clustered_index_size = index->stat_index_size;

  table->stat_sum_of_other_index_sizes = 0;
  for (index = index->next(); index != nullptr; index = index->next()) {
    if (index->type & DICT_FTS || dict_index_is_spatial(index)) continue;
    dict_stats_empty_index(index);
    if (dict_stats_should_ignore_index(index)) continue;
    if (!(table->stats_bg_flag & BG_STAT_SHOULD_QUIT)) {
      dict_stats_analyze_index(index);
    }
    table->stat_sum_of_other_index_sizes += index->stat_index_size;
  }

  table->stats_last_recalc = std::chrono::steady_clock::now();
  table->stat_modified_counter = 0;
  table->stat_initialized = true;
```

> ★★★ `stat_n_rows = 聚簇索引的 stat_n_diff_key_vals[n_unique - 1]`——**总行数是"主键全列前缀的不同值数"**，不是数出来的。这是 `n_rows` 只能是估算值的**根因**（采样外推 + 假定主键唯一）。

`stat_index_size` 与 `stat_n_leaf_pages` 反倒是**精确值**（B+ 树尺寸直接问出来，不是采样）：

```cpp
  size = btr_get_size(index, BTR_TOTAL_SIZE, &mtr);
  if (size != ULINT_UNDEFINED) {
    index->stat_index_size = size;
    size = btr_get_size(index, BTR_N_LEAF_PAGES, &mtr);
  }
  ...
  index->stat_n_leaf_pages = size;
```

**由采样页外推到全表的换算公式**（`dict_stats_index_set_n_diff()`，对应文件开头 53-116 行的算法总纲）：

```cpp
    const n_diff_data_t *data = &n_diff_data[n_prefix - 1];

    uint64_t n_ordinary_leaf_pages;
    if (data->level == 1) {
      /* If we know the number of records on level 1, then
         this number is the same as the number of pages on level 0 (leaf). */
      n_ordinary_leaf_pages = data->n_recs_on_level;
    } else {
      /* If we analyzed D ordinary leaf pages and found E
         external pages in total linked from those D ordinary
         leaf pages, then this means that the ratio
         ordinary/external is D/E. Then the ratio ordinary/total
         is D / (D + E). Knowing that the total number of pages
         is T (including ordinary and external) then we estimate
         that the total number of ordinary leaf pages is
         T * D / (D + E). */
      n_ordinary_leaf_pages =
          index->stat_n_leaf_pages * data->n_leaf_pages_to_analyze /
          (data->n_leaf_pages_to_analyze + data->n_external_pages_sum);
    }

    /* See REF01 for an explanation of the algorithm */
    index->stat_n_diff_key_vals[n_prefix - 1] =
        n_ordinary_leaf_pages
        * data->n_diff_on_level / data->n_recs_on_level                      // R：该层不同键占比
        * data->n_diff_all_analyzed_pages / data->n_leaf_pages_to_analyze;   // N_DIFF_AVG_LEAF：每叶页平均不同键数

    index->stat_n_sample_sizes[n_prefix - 1] = data->n_leaf_pages_to_analyze;
```

即 **`n_diff = N_ordinary_leaf_pages × R × N_DIFF_AVG_LEAF`**，其中：

| 符号 | 含义 |
|---|---|
| `N_ordinary_leaf_pages` | 去掉外部（溢出）页后的"普通"叶子页数。`level==1` 时 = `n_recs_on_level`；否则 = `stat_n_leaf_pages × D/(D+E)` |
| `R = n_diff_on_level / n_recs_on_level` | 停止层上"不同键占记录数的比例"，**假定叶子层同比例** |
| `N_DIFF_AVG_LEAF = n_diff_all_analyzed_pages / n_leaf_pages_to_analyze` | 采样叶页的平均不同键数 |

**算法总纲**（`dict0stats.cc:53-116` 的注释，讲透了"为什么要这么算"）：

```
Let the number of different key values found in each leaf page i be Pi (i=1..A).
Let N_DIFF_AVG_LEAF be (P1 + P2 + ... + PA) / A.
Let the number of different key values on level LA be N_DIFF_LA.
Let the total number of records on level LA be TOTAL_LA.
Let R be N_DIFF_LA / TOTAL_LA, we assume this ratio is the same on the leaf level.
Let the number of leaf pages be N.
Then the total number of different key values on the leaf level is:
        N * R * N_DIFF_AVG_LEAF.
```

> ★ **这个公式的两个隐含假设就是基数漂移的来源**：① 停止层与叶子层的"不同键比例 R"相同；② 采样的 A 个叶页能代表全部 N 个叶页。数据分布倾斜时两者都会失效。

**采样页数的宏**（优先表级 `stats_sample_pages`，0 才回落全局）：

```cpp
/* Gets the number of leaf pages to sample in persistent stats estimation */
#define N_SAMPLE_PAGES(index)                                    \
  static_cast<uint64_t>((index)->table->stats_sample_pages != 0  \
                            ? (index)->table->stats_sample_pages \
                            : srv_stats_persistent_sample_pages)

/* number of distinct records on a given level that are required to stop
   descending to lower levels and fetch N_SAMPLE_PAGES(index) records
   from that level */
#define N_DIFF_REQUIRED(index) (N_SAMPLE_PAGES(index) * 10)
```

采样太多时会自动降级成**全扫叶子层**（因为全扫反而更快更准）：

```cpp
  /* If the tree has just one level (and one page) or if the user
     has requested to sample too many pages then do full scan.
     For each n-column prefix (for n=1..n_uniq) N_SAMPLE_PAGES(index)
     will be sampled, so in total N_SAMPLE_PAGES(index) * n_uniq leaf
     pages will be sampled. If that number is bigger than the total
     number of leaf pages then do full scan of the leaf level instead
     since it will be faster and will give better results. */
  if (root_level == 0 || n_sample_pages * n_uniq >
                             std::min<ulint>(index->stat_n_leaf_pages, 1e6)) {
    (void)dict_stats_analyze_index_level(index, 0 /* leaf level */,
                                         index->stat_n_diff_key_vals,
                                         &total_recs, &total_pages, nullptr,
                                         wait_start_time, &mtr);
    ...
```

优化器通过 `handler::info()` / `records_in_range()` 读这些值，配合 `n_diff` 估算选择率。

#### ★ n_diff 的估算数学：降序前缀 + 单调性剪枝（`dict0stats.cc:1796-1813`）

`boundaries_t`（`dict0stats.cc:146`，`std::vector<uint64_t>`）是**每个前缀长度"停止下降的层号"**数组。核心算法在 `dict_stats_analyze_index_low` 的主循环，源码注释讲透了优化逻辑：

> If we find that level L is the first one (searching from the root) that contains **at least D distinct keys** when looking at the first n_prefix columns, then: if we look at the first **n_prefix-1** columns then the first level that contains D distinct keys will be **either L or a lower one**. So if we find that the first level containing D distinct keys (on n_prefix columns) is L, we continue **from L** when searching for D distinct keys on n_prefix-1 columns.

```cpp
for (n_prefix = n_uniq; n_prefix >= 1; n_prefix--) {   // ★ 从最长的前缀往短的降序
  ... // 从 boundaries[n_prefix] 记录的那一层开始找"第一个含 >= D 个不同键的层"
  ... // 找到后记入 boundaries[n_prefix-1]，下一轮从它开始
}
```

★ **单调性剪枝**：前缀少一列 ⇒ 不同键数只可能更多 ⇒ "达到 D 个不同键"的层**只会更低或相同**。于是 n_prefix-1 的搜索**从 n_prefix 找到的层 L 继续向下**，不用每次从根重搜——**N 个前缀的总搜索是一次从上往下的单程扫描**，而不是 N 次从根开始的扫描。

`dict_stats_analyze_index_for_n_prefix`（`:1426`）执行单层的具体统计：`pcur.open_at_side(true, ...)` 定位该层最左记录 → 逐记录比较 key 前缀 → 数不同值。★ 注意它**从不扫描叶子层**（`ut_ad(n_diff_data->level > 0)`，`:1456`）——因为 n_diff 只需要"不同键的个数"，高层 node pointer 的 key 就够了，扫叶子是浪费。

`n_diff_required`（目标不同键数）的计算（`:1821` 附近）**与采样页数联动**：需要多少个不同键才能给出可靠的 `n_diff` 估算，取决于采样页数 `N_SAMPLE_PAGES(index)`。

#### ★ `innodb_stats_include_delete_marked` 的判定路径（两个使用点）

```cpp
// srv0srv.cc:580 —— 默认 false
bool srv_stats_include_delete_marked = false;
```

采集时**两个地方**用同一变量：

```cpp
// ① dict0stats.cc:929 —— 逐层扫描时跳过删除标记记录
if (!(srv_stats_include_delete_marked ? 0
                                      : rec_get_deleted_flag(rec, ...))) {
  /* 这条记录计入统计 */
}

// ② dict0stats.cc:1354 —— n_diff 计数模式
srv_stats_include_delete_marked
    ? COUNT_ALL_NON_BORING_INCLUDE_DEL_MARKED
    : COUNT_ALL_NON_BORING_AND_SKIP_DEL_MARKED
```

★ **与 purge 的关系**：默认 `false` 意味着**未 purge 的删除标记记录不统计**（否则统计偏大）。但**只读备库 / purge 滞后的实例**上，大量删除标记长期存在，此时开这个参数让统计更接近实际可读的行数。变量注释原文："Include delete marked records when calculating persistent statistics"。

#### `RECALC_TRANSIENT` 的完整分支（此前只标了 `break`）

`dict_stats_update` 的 switch 之后有一段被忽略的统一收尾：

```cpp
switch (stats_upd_option) {
  case DICT_STATS_RECALC_TRANSIENT:
    break;                        // ← 什么都不做，落到下面
  ...
  /* no "default:" in order to produce a compilation warning
     about unhandled enumeration value */   // ★ 故意不加 default，让编译器
}                                           //    警告"漏了枚举值"

dict_table_stats_lock(table, RW_X_LATCH);
dict_stats_update_transient(table);         // ★★ switch 之后无条件瞬态更新
dict_table_stats_unlock(table, RW_X_LATCH);
return DB_SUCCESS;
```

★ **`break` 不是"什么都不做"**：瞬态路径 = switch 后**无条件** `dict_stats_update_transient(table)`（在统计 X 锁下全表扫描估算）。这也解释了 `FETCH_ONLY_IF_NOT_IN_MEMORY` 的**失败回退**：读取持久统计出错 → `break` → 同样落到 `dict_stats_update_transient`，错误信息正是 *"Using transient stats method instead"*（`:2964`）。

★ 顺带一个工程细节：**switch 故意不加 `default`**（注释明说）——让编译器对"新增枚举值未处理"产生警告，强制后来者在加枚举时考虑每个 case。这是 C++ 里用编译期检查替代运行时兜底的小技巧。

#### 直方图与 InnoDB 统计如何协同

两者**独立运作、各自服务优化器**，不互相喂数据：

| | InnoDB 统计 | 直方图 |
|---|---|---|
| 存哪 | `innodb_table_stats` / `innodb_index_stats` | `mysql.column_statistics` |
| 粒度 | 表级行数 + 索引前缀 n_diff | **单列值分布**（bucket 频数） |
| 触发 | ANALYZE / 后台重算 / 开表 | `ANALYZE TABLE ... UPDATE HISTOGRAM` |
| 优化器用途 | 行数估算、索引选择率（`records_in_range`） | 等值/范围谓词的选择率（比 n_diff 细得多，尤其数据倾斜） |

> ⚠️ **纠错**：一种流传很广的说法是"`records_in_range()` 优先用直方图精确化"——**这不是 8.0.39 的实现**。两条路径服务的环节不同：
>
> - `records_in_range()` / `KEY::records_per_key()` ⇒ **只来自 dict_stats**（`stat_n_rows / stat_n_diff_key_vals[j]`），与直方图**完全无关**
> - 直方图 ⇒ 只服务 **WHERE 条件过滤率**（`Item_func_*::get_filtering_effect()`），不参与索引访问的行数估算

**路径 1：dict_stats → 优化器**（唯一出口在 `ha_innobase::info_low()`）：

```cpp
/* storage/innobase/handler/ha_innodb.cc */
  n_rows = ib_table->stat_n_rows;
  stat_clustered_index_size = ib_table->stat_clustered_index_size;
  stat_sum_of_other_index_sizes = ib_table->stat_sum_of_other_index_sizes;
  ...
  stats.records = (ha_rows)n_rows;                    // ★ handler->stats.records ⇐ stat_n_rows
```

```cpp
        /* innodb_rec_per_key() will use
           index->stat_n_diff_key_vals[] and the value we
           pass index->table->stat_n_rows. Both are
           calculated by ANALYZE and by the background
           stats gathering thread ... */
        const rec_per_key_t rec_per_key =
            innodb_rec_per_key(index, (ulint)j, index->table->stat_n_rows);
        key->set_records_per_key(j, rec_per_key);
```

**路径 2：直方图 → 优化器**（server 侧唯一的桥接函数，只有 ~15 行）：

```cpp
/* sql/item_cmpfunc.cc —— get_histogram_selectivity() */
static bool get_histogram_selectivity(THD *thd, const Field *field, Item **args,
                                      size_t arg_count,
                                      histograms::enum_operator op,
                                      Item_func *item_func,
                                      const TABLE_SHARE *table_share,
                                      double *selectivity) {
  const histograms::Histogram *histogram =
      table_share->find_histogram(field->field_index());
  if (histogram != nullptr) {
    if (!histogram->get_selectivity(args, arg_count, op, selectivity)) {
      if (unlikely(thd->opt_trace.is_started()))
        write_histogram_to_trace(thd, item_func, *selectivity);
      return false;                                    // ★ 拿到了 selectivity
    }
  }
  return true;                                         // ★ 没有直方图
}
```

调用点一律是"有直方图用它，没有就回退默认猜测常量"的模式：

```cpp
float Item_func_ne::get_filtering_effect(...) {
  double selectivity;
  if (!get_histogram_selectivity(thd, fld->field, args, arg_count,
                                 histograms::enum_operator::NOT_EQUALS_TO, this,
                                 fld->field->table->s, &selectivity))
    return static_cast<float>(selectivity);            // ★ 直方图优先

  return 1.0f - fld->get_cond_filter_default_probability(rows_in_table,
                                                         COND_FILTER_EQUALITY);  // 兜底
}
```

> ★ 直方图挂在 **`TABLE_SHARE`** 上（`table_share->find_histogram(field_index)`），这也是为什么 `ANALYZE ... UPDATE HISTOGRAM` 之后要 `tdc_remove_table()` 强制 TABLE_SHARE 失效（见 `sql/sql_admin.cc`）。

**两条路径唯一的交叉点**——直方图算采样率时要用 InnoDB 给的总行数当分母：

```cpp
/* sql/histograms/histogram.cc —— update_histogram() */
  double rows_in_memory = thd->variables.histogram_generation_max_mem_size /
                          static_cast<double>(row_size_bytes);

  table->fetch_number_of_rows();
  ha_rows rows_in_table = std::max(1ULL, tbl->file->stats.records);  // ★ 来自 dict_stats

  double sample_percentage = rows_in_memory / rows_in_table * 100.0;
  sample_percentage = std::min(sample_percentage, 100.0);
```

**不互相写入的实证**（grep 计数）：

| grep | 范围 | 命中 |
|---|---|---|
| `stat_n_leaf_pages` | `sql/` | **0**——server 层从不接触 InnoDB 索引级统计结构 |
| `Histogram::get_filtering_effect` | 全树 | **0**——Histogram 上没有这个方法（真名 `get_selectivity`） |
| InnoDB 代码引用 server 直方图 | `storage/innobase/` | 0（`row0pread-histogram.cc` 是并行扫描的工作量分布直方图，同名异物） |

**一个显式的分工边界**：被**单列唯一索引**覆盖的列不建直方图——因为 `rec_per_key = 1` 已经足够精确：

```cpp
    /* Check if this field is covered by a single-part unique index. If it is, we
       don't want to create histogram statistics for it. */
    if (covered_by_single_part_index(thd, field)) {
      results.emplace(column_name, Message::COVERED_BY_SINGLE_PART_UNIQUE_INDEX);
      continue;
    }
```

**直方图的存储格式是 JSON**（不是自定义二进制），`mysql.column_statistics` 的 `histogram` 列类型为 `JSON`：

```cpp
/* sql/dd/impl/tables/column_statistics.cc */
  m_target_def.set_table_name("column_statistics");
  ...
  m_target_def.add_field(FIELD_HISTOGRAM, "FIELD_HISTOGRAM", "histogram JSON NOT NULL");
  m_target_def.add_index(INDEX_UK_CATALOG_ID_SCHEMA_NAME_TABLE_NAME_COLUMN_NAME,
                         "INDEX_UK_CATALOG_ID_SCHEMA_NAME_TABLE_NAME_COLUMN_NAME",
                         "UNIQUE KEY (catalog_id, schema_name, table_name, column_name)");
```

> ⚠️ **又一处注释与代码不一致**：`sql/histograms/histogram.h` 的注释写 "stored persistently in the system table **mysql.column_stats**"，而实际表名是 **`column_statistics`**（`set_table_name("column_statistics")` + `create_name()` 的断言可证）。

bucket 结构（两类直方图）：

```cpp
/* sql/histograms/singleton.h —— 一个桶 = 一个值 + 累积频率 */
template <class T>
struct SingletonBucket {
  T value;
  double cumulative_frequency;
};
```

```cpp
/* sql/histograms/equi_height_bucket.h —— 一个桶 = 下界 + 上界 + 累积频率 + 桶内不同值数 */
  /// Lower inclusive value contained in this bucket.
  const T m_lower_inclusive;
  /// Upper inclusive value contained in this bucket.
  const T m_upper_inclusive;
  /// The cumulative frequency. 0.0 <= m_cumulative_frequency <= 1.0.
  const double m_cumulative_frequency;
  /// Number of distinct values in this bucket.
  const ha_rows m_num_distinct;
```

★ 协同点总结：两者**服务优化器的不同环节**——dict_stats 决定"走哪个索引、索引范围扫多少行"（`records_per_key` / `records_in_range`），直方图决定"WHERE 条件过滤掉多少行"（`get_filtering_effect`）。所以"统计信息不准"的两个解法**作用面不同**：`ANALYZE` 修的是前者，建直方图修的是后者。

---

## 5. 后台自动重算

### 触发条件

DML 时累计修改计数，**修改超过表大小的 1/16 就加入重算池**（`row0mysql.cc:1145`）：

```cpp
/* Calculate new statistics if 1 / 16 of table has been modified */
if (dict_stats_auto_recalc_is_enabled(table)) {
  dict_stats_recalc_pool_add(table);
  table->stat_modified_counter = 0;
}
```

`recalc_pool` 是一个由 `recalc_pool_mutex` 保护的待处理表集合。

#### ★ 专属线程 `dict_stats_thread`（`dict0stats_bg.cc:357`）

统计重算**不在用户线程里做**，而是由一个独立的后台线程处理：

```cpp
void dict_stats_thread() {
  ut_a(!srv_read_only_mode);
  THD *thd = create_internal_thd();          // ① 后台线程也要有自己的 THD

  while (!SHUTTING_DOWN()) {
    /* Wake up periodically even if not signaled. This is
       because we may lose an event - if the below call to
       dict_stats_process_entry_from_recalc_pool() puts the entry back
       in the list, the os_event_set() will be lost by the subsequent
       os_event_reset(). */
    os_event_wait_time(dict_stats_event, MIN_RECALC_INTERVAL);   // ②

#ifdef UNIV_DEBUG
    while (innodb_dict_stats_disabled_debug) {                   // ③ 测试用开关
      os_event_set(dict_stats_disabled_event);
      if (SHUTTING_DOWN()) break;
      os_event_wait_time(dict_stats_event, std::chrono::milliseconds{100});
    }
#endif
    if (SHUTTING_DOWN()) break;

    dict_stats_process_entry_from_recalc_pool(thd);
    os_event_reset(dict_stats_event);
  }
  destroy_internal_thd(thd);
}
```

★② **为什么要"周期性唤醒"而不纯靠事件通知**：注释讲得很清楚——如果处理函数把条目**放回队列**，期间触发的 `os_event_set()` 会被随后的 `os_event_reset()` 吃掉，事件就丢了。定时兜底唤醒是**对"事件可能丢失"的补偿**。这是一个很典型的并发模式教训：*event + reset 的组合在有"处理失败重入队"的场景下会丢事件*。

① `create_internal_thd()`：后台线程要开表、要访问 DD，必须有 THD（DD 是 server 层的东西）。

#### 单条处理：`dict_stats_process_entry_from_recalc_pool`（`:263`）

```cpp
if (!dict_stats_recalc_pool_get(&table_id)) return;    // pop 一个 table_id

dict_sys_mutex_enter();                                 // ★ ①
table = dd_table_open_on_id(table_id, thd, &mdl, true, true);

if (table == nullptr) {                                 // ② 已被 DROP
  dict_sys_mutex_exit();
  return;
}
if (table->is_corrupted()) { ... return; }              // ③ 损坏表跳过

table->stats_bg_flag = BG_STAT_IN_PROGRESS;             // ★ ④
dict_sys_mutex_exit();

/* ut_time_monotonic() could be expensive, the current function is called
   once every time a table has been changed more than 10% ... */
if (std::chrono::steady_clock::now() - table->stats_last_recalc <
    MIN_RECALC_INTERVAL) {
  /* 距上次重算太近 → 放回队列，本次不重算 */
  dict_stats_recalc_pool_add(table);                    // ★⑤
} else {
  dict_stats_update(table, DICT_STATS_RECALC_PERSISTENT);
}

dict_sys_mutex_enter();
table->stats_bg_flag = BG_STAT_NONE;                    // 清标志
dict_sys_mutex_exit();

/* This call can't be moved into dict_sys->mutex protection,
   since it'll cause deadlock while release mdl lock. */
dd_table_close(table, thd, &mdl, false);                // ★⑥
```

- ★① 整个"开表 + 置标志"在 `dict_sys->mutex` 下做——**为了和 DROP TABLE 等 DDL 互斥**；
- ② `table == nullptr` 是正常情况：表 id 入队后可能已被 DROP（注释：*must have been DROPped after its id was enqueued*）；
- ★④ `stats_bg_flag = BG_STAT_IN_PROGRESS` 相当于给表打上"后台统计进行中"标记，**阻止并发 DDL**。

★ 有个反直觉的点：**后台统计线程要拿 MDL**（`dd_table_open_on_id(..., &mdl, ...)` 传出 mdl ticket）。因为它是通过 server 层的 DD 接口开表的，必须遵守 MDL 协议——这也是为什么统计线程可能和 DDL 相互等待。

#### ★ latch 层级是怎么推导出来的（`dict0stats_bg.cc:218-230`）

`recalc_pool_mutex` 的层级定义为 `SYNC_STATS_AUTO_RECALC`，源码用一整段注释解释了**为什么选这个值**——这是一份很好的 latch 层级推导范例：

| 获取场景 | 持锁上下文 | 对层级的要求 |
|---|---|---|
| ① 后台统计线程 | 先于任何其他 latch 获取，中间不加别的锁 | 任意层级均可 |
| ② `row_update_statistics_if_needed()` | 调用时**未持** `SYNC_DICT`，但函数内部**可能获取** | `<= SYNC_DICT` |
| ③ `row_drop_table_for_mysql()` | **已持** `SYNC_DICT` **和** `SYNC_DICT_OPERATION` 之后才获取 | `< SYNC_DICT` 且 `< SYNC_DICT_OPERATION` |

三个约束取交集 → **必须严格低于 `SYNC_DICT`**，于是选了 `SYNC_STATS_AUTO_RECALC`（在 `sync0types.h:310` 中排在 `SYNC_DICT` 之前）。

推论：**持有 recalc pool 锁时可以去拿 dict 锁，反之不行**——这条约束保证了"DROP TABLE（先拿 dict 锁）→ 想拿 recalc pool 锁"是合法顺序，而反向（统计线程先拿 pool 锁再拿 dict 锁）也合法，两者不会形成环。

`DBUG_EXECUTE_IF("do_not_meta_lock_in_background", return;)` 这个调试点（`:268`）也印证了上面的分析——它可以让后台线程跳过加 MDL，用于测试。

#### ★★★ 完整因果链：为什么"事件丢失"一定会发生

把 ⑤ 和主循环的 ② 连起来看，整条链就通了：

```
1. 表距上次重算 < 10 秒 → dict_stats_recalc_pool_add(table) 把它放回队列（不重算）
2. 这 10 秒里，若其他线程 DML 触发入队 → os_event_set(dict_stats_event) 通知"有活"
3. 线程处理完，回到主循环 → os_event_reset(dict_stats_event)  ← 把这个通知吃掉！
4. 线程 os_event_wait_time(...) —— 若没有超时，就永远睡着了（队列里明明有活）
```

**⑤ 是"事件丢失"的直接原因**：因为有"处理了但没真处理、放回队列"这条路径，期间到达的事件会被随后的 reset 吞掉。

而修复方式也很讲究——等待超时用的就是同一个常量：

```cpp
constexpr std::chrono::seconds MIN_RECALC_INTERVAL{10};   // dict0stats_bg.cc:51

os_event_wait_time(dict_stats_event, MIN_RECALC_INTERVAL);  // 主循环 :367
if (now - table->stats_last_recalc < MIN_RECALC_INTERVAL)   // 单条处理 :311
    dict_stats_recalc_pool_add(table);                      // 放回队列
```

★ **等待超时与最小重算间隔取同一个值（10 秒）**，于是"被放回队列的表"最多 10 秒后必然被重新捡起——这个常量复用让兜底间隔与业务逻辑自洽，不是随手挑的。

#### ★ `dd_table_close` 为什么必须在 dict mutex 之外（`:330-332`）

```cpp
/* This call can't be moved into dict_sys->mutex protection,
   since it'll cause deadlock while release mdl lock. */
```

**释放 MDL 的路径可能反向获取 dict_sys->mutex**（MDL 释放会触发表定义清理等回调），所以"持 dict 锁 → 释放 MDL"这个顺序会与"持 MDL → 拿 dict 锁"形成环。这是理解 InnoDB latch 顺序的一个很好的实例：**看起来无害的"释放锁"操作，本身也可能有加锁顺序约束**。

#### 阈值的一处注释不一致

`dict0stats_bg.cc:305` 的注释说 *called once every time a table has been changed more than **10%***，而触发端 `row0mysql.cc` 的注释写的是 *Calculate new statistics if **1 / 16** of table has been modified*（= 6.25%）。两处注释对不上——**以触发端代码为准**（入队条件由 `row_update_statistics_if_needed()` 决定），`dict0stats_bg.cc` 这处是历史注释未更新。

#### 删除路径：`dict_stats_recalc_pool_del`

线程除了"取表"，还有一条**删表路径**——表被 DROP / TRUNCATE 时，要把它的 id 从 recalc pool 里摘掉（否则后台线程会去开一个已不存在的表）：

```cpp
// dict0stats_bg.cc:169
void dict_stats_recalc_pool_del(const dict_table_t *table) {
  ut_ad(!srv_read_only_mode);
  ut_ad(dict_sys_mutex_own());      // 在 dict 锁下调用
  ... // 遍历 recalc_pool，erase 匹配的 table->id
}
```

调用点（`row0mysql.cc:3901`，DROP TABLE 路径）：

```cpp
if (dict_stats_auto_recalc_is_enabled(table)) {
  dict_stats_recalc_pool_del(table);   // ① 先摘掉待重算的 id
}
/* ② 再从持久存储删掉统计行（innodb_table_stats / innodb_index_stats） */
```

★ 顺序有意义：**先摘队列、再删统计行**——若反过来，后台线程可能在"统计行已删、id 还在队列"的窗口里重复计算。

#### 线程全流程总结

把"入队 → 唤醒 → 取表 → 限流 → 重算 → 放回"和"删除路径"合起来：

```
前台 DML：修改 > 1/16 → dict_stats_recalc_pool_add（去重后 push_back + os_event_set）
前台 DROP/TRUNCATE：     dict_stats_recalc_pool_del（摘 id）+ 删统计行
后台线程：os_event_wait_time(10s) → recalc_pool_get（FIFO 取第一个）
         → 开表 + MDL → 距上次 < 10s ? 放回队列 : dict_stats_update(RECALC_PERSISTENT)
         → 清 bg_flag → 关表
```

### ★ 线程视角：`dict_stats_thread` 的主循环与两个"上下文"

上面讲的是"触发条件与算法"，这一节讲**执行体本身**（`dict0stats_bg.cc`）。

```cpp
constexpr std::chrono::seconds MIN_RECALC_INTERVAL{10};

void dict_stats_thread() {
  THD *thd = create_internal_thd();
  while (!SHUTTING_DOWN()) {
    /* Wake up periodically even if not signaled. This is
    because we may lose an event - if the below call to
    dict_stats_process_entry_from_recalc_pool() puts the entry back
    in the list, the os_event_set() will be lost by the subsequent
    os_event_reset(). */
    os_event_wait_time(dict_stats_event, MIN_RECALC_INTERVAL);
    ...
    dict_stats_process_entry_from_recalc_pool(thd);
    os_event_reset(dict_stats_event);
  }
  destroy_internal_thd(thd);
}
```

★ **为什么必须定时醒（10 秒）而不是纯事件驱动**：源码注释给出了明确理由——表被"放回队列"时会 `os_event_set()`，而循环末尾的 `os_event_reset()` 会把这个信号吃掉，事件就丢了。所以靠 **10 秒超时兜底**。这是与 `buf_dump` / `buf_resize`（纯事件、无限期等待）的**根本差异**。

| 项 | 事实 |
|---|---|
| 事件 | `dict_stats_event` |
| 周期 | **10 秒**（`MIN_RECALC_INTERVAL`，同时兼任"同一张表两次重算的最小间隔"） |
| 唤醒源 | ① `dict_stats_recalc_pool_add()`（**唯一生产点**，末尾 set event）；② 自身放回队列；③ `dict_stats_shutdown()` |
| ★ 退出 | `os_event_set` + **`join()`**；退出阶段是 `SRV_SHUTDOWN_PRE_DD_AND_SYSTEM_TRANSACTIONS`，★ **比 `buf_dump`/`buf_resize`（`SRV_SHUTDOWN_CLEANUP`）更早**——因为它要访问 DD |

**recalc pool 的结构**（★ 纠正：不是定长数组）：

```cpp
static const ulint RECALC_POOL_INITIAL_SLOTS = 128;
typedef std::vector<table_id_t, recalc_pool_allocator_t> recalc_pool_t;
recalc_pool->reserve(RECALC_POOL_INITIAL_SLOTS);
```

- `SIZE_OF_A_RECALC_POOL` **不存在**——128 只是 `reserve()` 的初始预留，**没有硬上限**，`push_back` 会自动扩容。
- 只存 `table_id`（不存表对象）；**去重靠线性扫描**（O(n)），已在队列里的表不会重复加入。

**入队阈值**（`row_update_statistics_if_needed()`，★ `STATS_RECALC_THRESHOLD` 这个常量名不存在）：

```cpp
  counter = table->stat_modified_counter++;
  n_rows = dict_table_get_n_rows(table);
  if (dict_stats_is_persistent_enabled(table)) {
    if (counter > n_rows / 10 /* 10% */ && dict_stats_auto_recalc_is_enabled(table)) {
      dict_stats_recalc_pool_add(table);
      table->stat_modified_counter = 0;
    }
    return;
  }
  /* transient（非持久化）分支：6.25%，同步在用户线程里做 */
  if (counter > 16 + n_rows / 16) {
    dict_stats_update(table, DICT_STATS_RECALC_TRANSIENT);
  }
```

⇒ **持久化统计**走后台线程（10%），**transient 统计**是**用户线程同步**做的（6.25%）——后者不在本章的线程里。

**★ 第二重节流**：取出表后若 `now - table->stats_last_recalc < MIN_RECALC_INTERVAL`（10 秒），**放回队列且不重算**——防止热点小表被反复重算。

**★ 与 DDL 的协调**：取表前先置 `table->stats_bg_flag = BG_STAT_IN_PROGRESS`（在 `dict_sys->mutex` 下），`DROP TABLE` 等 DDL 用 `dict_stats_wait_bg_to_stop_using_table()` 轮询等待，让出方式是 **`DICT_STATS_BG_YIELD`——释放 dict latch → sleep 250ms → 重新加锁**（不是忙等）。

#### ★ 两个容易混淆的"上下文"：internal THD 与 background trx

| | 谁创建 | 生命周期 | 用途 | 是否在 MySQL 事务视图中 |
|---|---|---|---|---|
| **internal THD** | `create_internal_thd()`（线程入口一次） | 整个线程 | 提供 server 层 `THD*`：`dd_table_open_on_id(table_id, thd, &mdl, ...)` 需要它拿 **MDL** 并访问 **DD** | — |
| **background trx** | `trx_allocate_for_background()` + `trx_start_internal()` | **每条内部 SQL 一次**（`exec_sql` 传 nullptr 时现配；`dict_stats_save` 自己再配一个） | `que_eval_sql()` 执行内部 SQL 需要 trx | ★ **不可见**：`trx->sess = trx_dummy_sess` 且**不进 `trx_sys->mysql_trx_list`** |

⇒ 这就是"后台统计的事务为什么在 `SHOW ENGINE INNODB STATUS` / 死锁检测里看不到"的原因——只有 `trx_allocate_for_mysql()` 才挂进那条链。

**internal THD 的创建与销毁**（线程入口一次，退出时才销毁）：

```cpp
/* storage/innobase/dict/dict0stats_bg.cc —— dict_stats_thread() */
  THD *thd = create_internal_thd();
  ...
  while (!srv_read_only_mode && !shutdown_started) {
    ...
    dict_stats_process_entry_from_recalc_pool(thd, false);
    ...
  }
  ...
  destroy_internal_thd(thd);
```

它存在的唯一理由是**开表需要 server 层上下文**——`dd_table_open_on_id(table_id, thd, &mdl, ...)` 要同时拿 MDL 和访问 DD，两者都以 `THD*` 为参数。这是"引擎线程借用 server 基础设施"的典型。

**background trx 是每条内部 SQL 现配的**，与 THD 生命周期完全不同：

```cpp
/* dict_stats_save() —— 现配一个 background trx */
  trx_t *trx = trx_allocate_for_background();

  if (srv_read_only_mode) {
    trx_start_internal_read_only(trx, UT_LOCATION_HERE);
  } else {
    trx_start_internal(trx, UT_LOCATION_HERE);
  }
  ...
  trx_commit_for_mysql(trx);
  trx_free_for_background(trx);
```

```cpp
/* dict_stats_fetch_from_ps() —— 再配一个，且用脏读 */
  trx = trx_allocate_for_background();

  /* Use 'read-uncommitted' so that the SELECTs we execute
     do not get blocked in case some user has locked the rows we
     are SELECTing */
  trx->isolation_level = TRX_ISO_READ_UNCOMMITTED;
  ...
  trx_commit_for_mysql(trx);
  trx_free_for_background(trx);
```

> ★ 两个上下文的分工一句话：**THD 是"以 server 身份开表"的凭证（整线程一份），trx 是"执行内部 SQL"的载体（每条 SQL 一份）**。混淆它们就会误解为什么"统计事务"既需要 MDL（来自 THD）又不在事务列表里（因为 trx 是 background 的）。

#### 为什么内部 SQL 要 "DELETE + INSERT" 而不是 UPDATE

`dict_stats_save()` 里的 `PROCEDURE TABLE_STATS_SAVE ()` 是 **InnoDB 自带解析器**处理的（`que_eval_sql`），不走 server 的 parser/优化器。

> ⚠️ **纠错**：常见解释是"内部解析器不支持 UPDATE，所以只能 DELETE+INSERT"——**这个理由是错的**。内部解析器**明确支持** `UPDATE ... SET ... WHERE ...`：

```cpp
/* storage/innobase/pars/pars0grm.y —— 内部解析器语法 */
statement:
        while_statement ';'
        | for_statement ';'
        | insert_statement ';'
        | delete_statement_searched ';'
        | delete_statement_positioned ';'
        | update_statement_searched ';'      /* ★ 支持 UPDATE */
        | update_statement_positioned ';'    /* ★ 支持 UPDATE */
        | open_cursor_statement ';'
        | fetch_statement ';'
        ...
```

而且**源码里没有任何一行注释解释"为什么不用 UPDATE"**（grep `instead of` 无相关命中）。所以文档只能给**可证实的**两条理由，其余属推断：

1. **幂等**：DELETE 无论行存在与否都成功，随后 INSERT 无条件写入，从而**统一"首次统计"与"更新统计"两条路径**——不需要先 SELECT 判断行是否存在。（推断，但与代码行为一致）
2. **索引统计的 stat 集合会变**：索引变更后 `stat_name` 集合不同（`n_diff_pfx01`…`n_diff_pfxNN`），先按四元组 DELETE 才不会残留旧行。（可证实：同一模式在 `INDEX_STATS_SAVE` 里重复，`DELETE ... WHERE database_name=... AND index_name=... AND stat_name=...`）

**真正写在注释里的理由是"防死锁"**（见第 ④ 点）——按 PK 顺序加锁，这才是可证实的官方说明。

★ 而且它必须用**独立 trx**——`handler0alter.cc` 的注释原文：

> *"Lock waits are not allowed in a data dictionary transaction. (Lock waits are possible on the statistics table, because it is directly accessible by users, not covered by the `dict_operation_lock`.)"*

即 `mysql.innodb_table_stats` 用户可直接查改，可能锁等待，因此不能挂在 DD 事务里。

### 相关参数

| 参数 | 默认 | 作用 |
|---|---|---|
| `innodb_stats_persistent` | ON | 是否启用持久统计 |
| `innodb_stats_auto_recalc` | ON | 是否自动重算（表级 `STATS_AUTO_RECALC` 可覆盖） |
| `innodb_stats_persistent_sample_pages` | 20 | 持久统计的采样页数 |
| `innodb_stats_transient_sample_pages` | 8 | 瞬态统计的采样页数 |
| `innodb_stats_method` | `nulls_equal` | NULL 值在 `n_diff` 里如何计数（`nulls_equal` / `nulls_unequal` / `nulls_ignored`） |
| `innodb_stats_include_delete_marked` | OFF | 采集时是否计入删除标记的记录（**只读备库**上可能需要开） |

**全部 sysvar 的源码定义**（都在 `storage/innobase/handler/ha_innodb.cc`，以插件变量形式注册，`innodb_` 由插件前缀自动拼接）：

```cpp
static MYSQL_SYSVAR_BOOL(
    stats_persistent, srv_stats_persistent, PLUGIN_VAR_OPCMDARG,
    "InnoDB persistent statistics enabled for all tables unless overridden"
    " at table level",
    nullptr, nullptr, true);                                   // ★ 默认 ON

static MYSQL_SYSVAR_BOOL(
    stats_auto_recalc, srv_stats_auto_recalc, PLUGIN_VAR_OPCMDARG,
    "InnoDB automatic recalculation of persistent statistics enabled for all"
    " tables unless overridden at table level (automatic recalculation is only"
    " done when InnoDB decides that the table has changed too much and needs a"
    " new statistics)",
    nullptr, nullptr, true);                                   // ★ 默认 ON

static MYSQL_SYSVAR_ULONGLONG(
    stats_persistent_sample_pages, srv_stats_persistent_sample_pages,
    PLUGIN_VAR_RQCMDARG,
    "The number of leaf index pages to sample when calculating persistent"
    " statistics (by ANALYZE, default 20)",
    nullptr, nullptr, 20, 1, ~0ULL, 0);                        // ★ 默认 20

static MYSQL_SYSVAR_ULONGLONG(
    stats_transient_sample_pages, srv_stats_transient_sample_pages,
    PLUGIN_VAR_RQCMDARG,
    "The number of leaf index pages to sample when calculating transient"
    " statistics (if persistent statistics are not used, default 8)",
    nullptr, nullptr, 8, 1, ~0ULL, 0);                         // ★ 默认 8

static MYSQL_SYSVAR_BOOL(
    stats_include_delete_marked, srv_stats_include_delete_marked,
    PLUGIN_VAR_OPCMDARG,
    "Include delete marked records when calculating persistent statistics",
    nullptr, nullptr, false);                                  // ★ 默认 OFF

static MYSQL_SYSVAR_ENUM(
    stats_method, srv_innodb_stats_method, PLUGIN_VAR_RQCMDARG,
    "Specifies how InnoDB index statistics collection code should"
    " treat NULLs. Possible values are NULLS_EQUAL (default),"
    " NULLS_UNEQUAL and NULLS_IGNORED",
    nullptr, nullptr, SRV_STATS_NULLS_EQUAL, &innodb_stats_method_typelib);
```

对应的 C 变量初值（`storage/innobase/srv/srv0srv.cc`，注释即"两种统计"的官方定义）：

```cpp
/* When estimating number of different key values in an index, sample
this many index pages, there are 2 ways to calculate statistics:
* persistent stats that are calculated by ANALYZE TABLE and saved
  in the innodb database.
* quick transient stats, that are used if persistent stats for the given
  table/index are not found in the innodb database */
unsigned long long srv_stats_transient_sample_pages = 8;
bool srv_stats_persistent = true;
bool srv_stats_include_delete_marked = false;
unsigned long long srv_stats_persistent_sample_pages = 20;
bool srv_stats_auto_recalc = true;
```

**另外两个不在上表但相关的变量**：

```cpp
static MYSQL_SYSVAR_BOOL(
    stats_on_metadata, innobase_stats_on_metadata, PLUGIN_VAR_OPCMDARG,
    "Enable statistics gathering for metadata commands such as"
    " SHOW TABLE STATUS for tables that use transient statistics (off by "
    "default)",
    nullptr, nullptr, false);                                  // ★ 默认 OFF
```

> ⚠️ **一处"代码 vs 变量定义"不一致**（很有迷惑性）：`ha_innodb.cc:339` 写着 `bool innobase_stats_on_metadata = true;`，但 `MYSQL_SYSVAR_BOOL` 的第 7 个实参是 `false`。**插件装载时 sysvar 默认值会覆盖 C 初值**，所以生效默认 = **OFF**。MTR 用例 `innodb_stats_on_metadata_basic.result` 实测为 `0`。判断默认值一律以 sysvar 定义为准。

```cpp
static Sys_var_ulong Sys_information_schema_stats_expiry(
    "information_schema_stats_expiry",
    "The number of seconds after which mysqld server will fetch "
    "data from storage engine and replace the data in cache.",
    SESSION_VAR(information_schema_stats_expiry), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, LONG_TIMEOUT), DEFAULT(24 * 60 * 60), BLOCK_SIZE(1));   // ★ 默认 24 小时
```

> ⚠️ **`innodb_stats_traditional` 在 8.0.39 不存在**（全仓库 grep `stats_traditional` 命中 0）——那是 MariaDB 系的变量，不要写进文档。

★ `innodb_stats_include_delete_marked` 是个容易被忽略的参数：它对应 `srv_stats_include_delete_marked`。在**只读实例**上 purge 可能滞后，开启它能让统计更接近实际——但这属于 TRX_SYS 相关的另一条线索（见 `row0mysql.cc` 对它的判定）。

**"是否需要更新"的唯一判定入口**（注意函数名，旧资料常写错）：

```cpp
/* storage/innobase/row/row0mysql.cc —— row_update_statistics_if_needed() */
  counter = table->stat_modified_counter++;
  n_rows = dict_table_get_n_rows(table);

  if (dict_stats_is_persistent_enabled(table)) {
    if (counter > n_rows / 10 /* 10% */
        && dict_stats_auto_recalc_is_enabled(table)) {
      dict_stats_recalc_pool_add(table);          // ★ 只入队，重算由后台线程做
      table->stat_modified_counter = 0;
    }
    return;
  }

  /* Calculate new statistics if 1 / 16 of table has been modified
     since the last time a statistics batch was run.
     We calculate statistics at most every 16th round, since we may have
     a counter table which is very small and updated very often. */
  if (counter > 16 + n_rows / 16 /* 6.25% */) {
    dict_stats_update(table, DICT_STATS_RECALC_TRANSIENT);   // ★ 就地同步重算
  }
```

> ★ 两个阈值不同：**persistent 是 10%，transient 是 `16 + n_rows/16`（≈6.25%）**。而且 persistent 只"入队"，transient 才"就地算"。

计数器的语义（`dict0mem.h`，注释明说不受 latch 保护）：

```cpp
  /** How many rows are modified since last stats recalc. When a row is
     inserted, updated, or deleted, we add 1 to this number; we calculate
     new estimates for the table and the indexes if the table has changed
     too much, see row_update_statistics_if_needed(). The counter is reset
     to zero at statistics calculation. This counter is not protected by
     any latch, because this is only used for heuristics. */
  uint64_t stat_modified_counter;
```

---

### 5.5 ★ 持久化写入：`dict_stats_save`（`dict0stats.cc:2187`）

采集完的落盘环节，四个值得细看的点：

#### ① 先快照，再写盘

```cpp
table = dict_stats_snapshot_create(table_orig);   // 在统计锁下拷贝一份快照
```

后续所有操作都在**快照**上做——防止写盘过程中统计被并发更新。这是快照模式（读写分离）：写盘读到的是一致视图，与 `FETCH_ONLY_IF_NOT_IN_MEMORY` 里的 dummy clone 是同一思路的两处应用。

#### ② 用"内部 SQL"写，不是直接 B+ 树操作

表级统计的写入方式出乎很多人意料：

```cpp
ret = dict_stats_exec_sql(pinfo,
    "PROCEDURE TABLE_STATS_SAVE () IS\n"
    "BEGIN\n"
    "DELETE FROM innodb_table_stats "
    "WHERE database_name=:database_name AND table_name=:table_name;\n"
    "INSERT INTO innodb_table_stats VALUES (:database_name, :table_name, "
    ":last_update, :n_rows, :clustered_index_size, :sum_of_other_index_sizes);\n"
    "END;", nullptr);
```

即 InnoDB **内部跑一条动态 SQL**（内部解析器 + 查询图执行），对 `mysql.innodb_table_stats` 做 DELETE + INSERT——走的是完整 SQL 栈，不是手写 B+ 树游标。索引级统计走 `dict_stats_save_index_stat()`，每行一个 stat：`n_diff_pfx01`…`n_diff_pfxNN`（`snprintf("n_diff_pfx%02lu", i+1)`）、`n_leaf_pages`、`size`。

#### ③ 内部事务 + 全局串行

```cpp
rw_lock_x_lock(dict_operation_lock, ...);      // ★ 全程持字典操作 X 锁
trx_t *trx = trx_allocate_for_background();    // 内部事务，不占用户 trx
trx_start_internal(trx, ...);
... 写所有行 ...
trx_commit_for_mysql(trx);
```

★ `dict_operation_lock` 全程 X 锁——**统计保存与 DDL 互斥、全局串行**。

#### ④ ★★ 固定加锁顺序防死锁（`dict0stats.cc:2265-2275` 注释）

一次保存要改 `innodb_index_stats` 的**多行**，单事务多行修改会与并发的 `DROP TABLE`（它做 `DELETE FROM innodb_index_stats WHERE ...`）死锁——双方以不同顺序锁行。解法是教科书级的：

```cpp
/* To prevent deadlocks we always lock the rows in the same order - the order
   of the PK, which is (database_name, table_name, index_name, stat_name).
   This is why below we sort the indexes by name and then for each index,
   do the mods ordered by stat_name. */
index_map_t indexes((ut_strcmp_functor()), ...);  // std::map，按索引名排序
```

**所有写者都按主键顺序加锁**（先按索引名排序、再按 `stat_name` 顺序改）——锁序一致 ⇒ 不可能出现环形等待。这是"固定加锁顺序"防死锁的标准范本。

---

## 6. `ANALYZE TABLE` 完整链路

```
ANALYZE TABLE t
  → Sql_cmd_analyze_table::execute              sql/sql_admin.cc
    → ha_innobase::analyze()                    ha_innodb.cc
      → ...
        → dict_stats_update(table, DICT_STATS_RECALC_PERSISTENT)   dict0stats.cc:2817
             ├─ srv_wake_purge_thread_if_not_active()   （history len > 0 时）
             ├─ dict_stats_update_persistent()          （逐索引采样）
             └─ dict_stats_save()                       （写 innodb_table_stats/innodb_index_stats）
```

要点：

- `ANALYZE` **走的是 `RECALC_PERSISTENT`**，即真采一遍 + 存盘，与后台自动重算走同一入口；
- 若 `innodb_stats_persistent = OFF`，`ANALYZE` 只刷新瞬态统计（内存），重启即失；
- `ANALYZE` 期间会持索引锁采样，可能阻塞 DML（这也是有 `STATS_SAMPLING_TOO_LARGE` 重试机制的原因）。

---

## Misc

### 容易误解的概念

| 误解 | 正解 |
|---|---|
| "持久统计重启后一定精确" | 是**上次采集时的估算**，不是实时值；重启后从表里读回 |
| "`SHOW INDEX` 的 Cardinality 就是 `innodb_index_stats` 里的值" | 前者是**实时估算**（可能每次调用都变），后者是持久化的 |
| "统计信息存在 DD 表里就归 DD 管" | 采集/更新全在 InnoDB（`dict0stats*.cc`），DD 只提供存储位置 |
| "内部表（SYS_TABLES）也有持久统计" | 无——`ut_a(strchr(name, '/'))` 断言排除了它们 |
| "`ANALYZE` 会锁表" | 采样时持索引锁、可能重试，但不锁全表；`innodb_stats_auto_recalc` 的后台重算同理 |

### ★★★ 真名勘误

以下均为"常见说法 / 旧资料说法"与 8.0.39 实际的差异，逐条经 grep 核实：

| 常见说法 | 8.0.39 实际 |
|---|---|
| `dict_stats_update_if_needed()` | **不存在**（全树命中 0，只在注释里被提到）。真实入口是 `row_update_statistics_if_needed()`（`row0mysql.cc`） |
| `dict_stats_get_table_stats()` | **不存在**。取行数用 `dict_table_get_n_rows()`（`dict0dict.ic`） |
| `dict_stats_drop` | **不存在**。是 `dict_stats_drop_index()` 与 `dict_stats_drop_table()` |
| `dict_stats_init_on_load` | **不存在**。开表入口是 `dict_stats_init()`（`dict0stats.ic`） |
| 枚举第四项是 `NO_RECALC` | 是 **`DICT_STATS_EMPTY_TABLE`**。"不重算"由 `stat_initialized` 短路表达，不是枚举值 |
| `innodb_stats_traditional` | **8.0.39 不存在**（grep 0 命中）——那是 MariaDB 系变量 |
| `innodb_stats_on_metadata` 已删除 | **仍存在**，默认 **OFF**。注意 `ha_innodb.cc:339` 的 C 初值是 `true`，但 sysvar 定义的默认值 `false` 会在插件装载时覆盖它 |
| `Histogram::get_filtering_effect()` / `estimate_selectivity()` | **Histogram 上没有这两个方法**。真名 `Histogram::get_selectivity()`；server 侧桥接是文件静态 `get_histogram_selectivity()` |
| `enum class HistogramType` | 是嵌套的 `Histogram::enum_histogram_type`，取值 `EQUI_HEIGHT` / `SINGLETON` |
| `m_histogram_type` / `m_data` | 真名 `m_hist_type` / `m_data_type` |
| `dd::Table_statistics` 是 DD 持久化对象 | 真实 DD 类叫 **`dd::Table_stat`**；`dd::info_schema::Table_statistics` 是**另一回事**——I_S 查询用的单表内存缓存类（成员只有 `ha_statistics m_stats`） |
| 直方图存在 `mysql.column_stats` | 实际表名 **`mysql.column_statistics`**（`histogram.h` 的注释写错了） |
| `N_SAMPLE_PAGES` 是常量 | 是**宏**，优先取表级 `stats_sample_pages`，为 0 才回落全局 |
| `dict_stats_analyze_index_low(N)` 传值 | 第二参是 `uint64_t &n_sample_pages`（**引用**），被 abort 时就地减半 |
| 重算阈值统一是 10% | **persistent 是 10%**（`counter > n_rows/10`），**transient 是 `16 + n_rows/16`**（≈6.25%） |

### ★★ 机制纠错（两条，均已在上文就近更正）

1. **"`records_in_range()` 优先用直方图精确化"是错的**——二者服务优化器的不同环节：`records_in_range` / `KEY::records_per_key` 只来自 dict_stats；直方图只服务 `Item_func_*::get_filtering_effect()`（WHERE 条件过滤率）。两者唯一交叉是 `update_histogram()` 用 `tbl->file->stats.records` 当采样率分母。
2. **"内部解析器不支持 UPDATE 所以只能 DELETE+INSERT"是错的**——`pars0grm.y` 明确有 `update_statement_searched`。且源码**没有任何注释**解释该选择；可证实的只有"按 PK 顺序加锁防死锁"那一条。

### 待补清单

（全部完成，暂无待补）

## 参考

**内核月报 / 社区文章**
- [MySQL · 内核分析 · InnoDB 的统计信息](http://mysql.taobao.org/monthly/2020/03/01/)
- [MySQL 深潜 - 统计信息采集](http://mysql.taobao.org/monthly/2022/10/01/)
- [MySQL · 内核特性 · 统计信息的现状和发展](http://mysql.taobao.org/monthly/2020/12/01/)
- [MySQL · 内核特性 · 直方图](http://mysql.taobao.org/monthly/2021/05/01/)
- [MySQL · 性能优化 · CloudDBA SQL 优化建议之统计信息获取](http://mysql.taobao.org/monthly/2017/10/01/)

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Persistent Optimizer Statistics*
- *MySQL 8.0 Reference Manual → ANALYZE TABLE Statement*

**源码**
- `storage/innobase/dict/dict0stats.cc`（采集与更新）
- `storage/innobase/dict/dict0stats_bg.cc`（后台重算池）
- `storage/innobase/dict/dict0stats.ic`（内联访问器）
