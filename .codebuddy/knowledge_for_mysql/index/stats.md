# 索引统计：采样、持久化与优化器衔接

> 基于 MySQL 8.0.39 源码。本篇讲 **索引基数（cardinality）是怎么算出来的、存在哪、怎么喂给优化器，以及为什么它不准**。
>
> **边界**：B-tree 的行数估算（`btr_estimate_n_rows_in_range` 的 `BTR_ESTIMATE` 搜索路径）见 [`btr.md`](btr.md)；优化器怎么用选择率算代价见 [`../server/query/07_optimize/`](../server/query/07_optimize/)；索引类型见 [`types.md`](types.md)。本篇聚焦**统计本身的产生与传递**。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [统计存在哪：`dict_table_t` / `dict_index_t` 的统计成员](#统计存在哪dict_table_t--dict_index_t-的统计成员)
    - [两套模式：persistent 与 transient](#两套模式persistent-与-transient)
  - 采样的产生
    - [持久化采样：分层下潜 + 分段随机](#持久化采样分层下潜--分段随机)
    - [外推公式（REF01）](#外推公式ref01)
    - [transient 采样：随机页 + 补偿项](#transient-采样随机页--补偿项)
  - 触发与生命周期
    - [何时重算：10% 判据与后台线程](#何时重算10-判据与后台线程)
    - [`dict_stats_save`：写回与死锁规避](#dict_stats_save写回与死锁规避)
  - 喂给优化器
    - [`rec_per_key` 的传递链](#rec_per_key-的传递链)
    - [index dive 与 statistics 模式的切换](#index-dive-与-statistics-模式的切换)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引统计是一组**估计值**：每张表的行数、每个索引的叶子页数、每个 n 列前缀的"不同键值数（`n_diff`）"。优化器用它算选择率与代价。

**最重要的一点**：MySQL 的索引基数**从来不是数出来的**，是**采少量页 + 两个比例假设外推**出来的。

### 用途

- `rec_per_key = stat_n_rows / stat_n_diff_key_vals[i]` → 优化器算 ref 访问的 fanout；
- range 扫描的行数估计（index dive 或 `records_per_key`）；
- `SHOW INDEX` 的 `Cardinality`、`SHOW TABLE STATUS` 的 `Rows`；
- 全表扫描代价（`stats.records`）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 引入**持久化统计**（此前只有 transient） |
| 8.0 | 统计表 `mysql.innodb_table_stats`/`innodb_index_stats` 纳入 DD；`innodb_analyze_is_persistent` 已移除 |
| 8.0 | `eq_range_index_dive_limit` 默认 200：等值 range 超限时用统计代替 index dive |

---

## 理论基础

### 设计思想与权衡

**1. 精确统计的代价不可接受。** 全表 `COUNT(DISTINCT)` 需要扫全表 + 排序，对大表是分钟级。InnoDB 选择了**采样 + 外推**，把代价压到 O(采样页数)。

**2. 分层下潜是为了让样本"均匀覆盖键值域"。** transient 模式纯随机挑页，样本可能聚堆；persistent 模式先在非叶层找到"不同值足够多（≥ A×10）"的层 LA，把该层的不同值**均分成 A 段、每段随机取一个**再下潜——保证样本分散在整棵树的键值域上。

**3. 确定性优先于精确性。** 默认排除 delete-marked 记录（`innodb_stats_include_delete_marked=OFF`），源码注释说明原因：否则"DELETE 后立刻 ANALYZE"与"等 purge 清完再 ANALYZE"结果不同——非确定。代价是 DELETE 后统计与可见行数不一致。

**4. 统计是"启发式"而非"事务性"的。** `stat_modified_counter` 明确注释"not protected by any latch ... not even reversed on rollback"。统计的更新不参与事务语义。

### 理论溯源

- 经典数据库统计：System R 的优化器统计思想（Selinger et al., 1979），用估计的选择率驱动访问路径选择；
- 采样估计：统计学上的**随机抽样外推**，误差来源即"样本代表性"。

### 算法与数据结构

三个核心数组，长度均为 `n_uniq`（下标 i 对应"前 i+1 列组成的前缀"）：`stat_n_diff_key_vals[]`、`stat_n_sample_sizes[]`、`stat_n_non_null_key_vals[]`。

---

## 核心实现

### 主线与基础构件

#### 统计存在哪：`dict_table_t` / `dict_index_t` 的统计成员

> **事实纠正**：8.0.39 里**没有** `dict_stats_t` 结构——统计直接是 `dict_table_t`/`dict_index_t` 的成员。

表级（`dict0mem.h:2238-2305`）：`stat_initialized`、`stats_last_recalc`、`stat_persistent`、`stats_auto_recalc`、`stats_sample_pages`、`stat_n_rows`、`stat_clustered_index_size`、`stat_sum_of_other_index_sizes`、`stat_modified_counter`。

索引级（`dict0mem.h:1213-1236`）：

```cpp
uint64_t *stat_n_diff_key_vals;      // 每个 n 列前缀的不同键值数估计
uint64_t *stat_n_sample_sizes;       // 该值采样了多少叶子页
uint64_t *stat_n_non_null_key_vals;  // 非 NULL 值数（仅 nulls_ignored 采集）
ulint stat_index_size;               // 索引总页数
ulint stat_n_leaf_pages;             // 叶子页数
```

其中 `stat_index_size` / `stat_n_leaf_pages` 是**精确值**（`btr_get_size()` 读 segment inode），只有 `n_diff` 是估计值。

#### 两套模式：persistent 与 transient

```cpp
// storage/innobase/include/dict0stats.ic:69-98
static inline bool dict_stats_is_persistent_enabled(const dict_table_t *table) {
  uint32_t stat_persistent = table->stat_persistent;
  if (stat_persistent & DICT_STATS_PERSISTENT_ON)       return true;
  else if (stat_persistent & DICT_STATS_PERSISTENT_OFF) return false;
  else return (srv_stats_persistent);      // 全局默认 true
}
```

| 维度 | persistent（默认 ON） | transient |
|---|---|---|
| 算法 | `dict_stats_analyze_index_low()` 分层采样 | `btr_estimate_number_of_different_key_vals()` 随机页采样 |
| 采样页数 | `stats_sample_pages` 或 `srv_stats_persistent_sample_pages`（**默认 20**） | `srv_stats_transient_sample_pages`（**默认 8**） |
| 落盘 | 写 `mysql.innodb_table_stats` + `innodb_index_stats` | 仅内存，关表/重启即失 |
| 重算执行者 | 后台 `dict_stats_thread`（异步） | 前台用户线程（同步） |
| 锁 | 需 **SX** 索引锁（扫非叶层） | 只需 **S** 索引锁 |

物理载体（`ha_innodb.cc:12979-13023` 的 DD 表定义）：

| 表 | 主键 | 关键列 |
|---|---|---|
| `mysql.innodb_table_stats` | `(database_name, table_name)` | `n_rows`、`clustered_index_size`、`sum_of_other_index_sizes`、`last_update` |
| `mysql.innodb_index_stats` | `(database_name, table_name, index_name, stat_name)` | `stat_name`、`stat_value`、`sample_size`、`stat_description` |

`stat_name` 的三种取值：`n_diff_pfx01…n_diff_pfxNN`（不同键值数，`stat_description` 是逗号分隔的列名）、`n_leaf_pages`、`size`。

---

### 采样的产生

#### 持久化采样：分层下潜 + 分段随机

算法总纲（源码注释 `dict0stats.cc:53-116`，A = `N_SAMPLE_PAGES(index)`）：

```
从 root 层起逐层下扫，直到某层 LA 的不同值数 ≥ A*10（或到达 level 1 强制停）
把 LA 层的不同值分成 A 组，每组随机取一条记录，下潜到叶子页
统计每个叶子页的不同值数 Pi
n_diff = N_ordinary_leaf_pages × (n_diff_on_level / n_recs_on_level) × (ΣPi / A)
```

`dict_stats_analyze_index_low()`（`dict0stats.cc:1668-1982`）关键分支：

```cpp
const uint64_t n_diff_required = n_sample_pages * 10;     // ① A*10，默认 200

// ② 全表扫描捷径：单页树，或 A*n_uniq 超过叶子页数（上限 1e6）
if (root_level == 0 ||
    n_sample_pages * n_uniq > std::min<ulint>(index->stat_n_leaf_pages, 1e6)) {
  dict_stats_analyze_index_level(index, 0, index->stat_n_diff_key_vals, ...);
  return true;                                            // 结果精确
}

for (n_prefix = n_uniq; n_prefix >= 1; n_prefix--) {       // ③ 长前缀 → 短前缀
  if (level_is_analyzed && (n_diff_on_level[n_prefix-1] >= n_diff_required || level == 1))
    goto found_level;                                     // ④ 复用上一层结果
  for (;;) {
    if (total_recs > n_sample_pages) {                    // ⑤ 页数上限保护
      level++; level_is_analyzed = true; break;
    }
    if (!dict_stats_analyze_index_level(index, level, n_diff_on_level,
                                        &total_recs, &total_pages,
                                        n_diff_boundaries, wait_start_time, &mtr)) {
      n_sample_pages = prev_total_recs / 2;               // ⑥ 锁让步：采样页数减半
      succeeded = false; goto end;
    }
    if (level == 1 || n_diff_on_level[n_prefix-1] >= n_diff_required) break;
    level--;
  }
found_level:
  dict_stats_analyze_index_for_n_prefix(index, n_prefix, &n_diff_boundaries[n_prefix-1],
                                        data, wait_start_time, &mtr);
}
```

分段随机的实现（`dict_stats_analyze_index_for_n_prefix`，`dict0stats.cc:1483-1531`）：

```cpp
// 把 n_diff_on_level 个不同值均分成 n_pick 段，每段随机取一个
const uint64_t left  = n_diff * i       / n_pick;
const uint64_t right = n_diff * (i + 1) / n_pick - 1;
const uint64_t rnd   = ut::random_from_interval(left, right);
const uint64_t dive_below_idx = boundaries->at(rnd);
... 下潜到叶子页，统计该页不同值数
if (n_diff_on_leaf_page > 0) n_diff_on_leaf_page--;   // ★ 减 1：避免跨页重复计数
```

**下潜时的"boring record"优化**（`dict_stats_analyze_index_below_cur`）：若某非叶页上所有键都相同（下潜时 `*n_diff == 1`），则**不再下潜**——其下所有叶子页的不同值数必然也是 1。

**锁让步机制**（重要）：若别的线程等这个索引锁快超时，采样放弃并把 `n_sample_pages` 减半，外层 `dict_stats_analyze_index()` 睡眠 100ms 后重试：

```cpp
while (n_sample_pages > 0 && !dict_stats_analyze_index_low(n_sample_pages, index)) {
  ib::warn(ER_IB_MSG_STATS_SAMPLING_TOO_LARGE) << "Detected too long lock waiting ...";
  std::this_thread::sleep_for(std::chrono::milliseconds(100));
}
```

持续锁竞争 → 采样页数一路减半 → **基数系统性偏低**。这是生产上"统计突然变差"的一条真实路径（error log 里可见该警告）。

#### 外推公式（REF01）

`dict_stats_index_set_n_diff()`（`dict0stats.cc:1604-1659`）：

```cpp
index->stat_n_diff_key_vals[n_prefix - 1] =
    n_ordinary_leaf_pages                                       // 叶子页数（剔除 BLOB 外部页）
  * data->n_diff_on_level / data->n_recs_on_level               // R = LA 层的不同值比例
  * data->n_diff_all_analyzed_pages / data->n_leaf_pages_to_analyze;  // 每页平均不同值数
```

其中第一个因子 `n_ordinary_leaf_pages`（普通叶页数，剔除 BLOB 外部页）**本身就是一次推导**：

```cpp
if (data->level == 1) {
  /* level 1 的记录数 == level 0 的页数，直接用（此时不用 E 修正） */
  n_ordinary_leaf_pages = data->n_recs_on_level;
} else {
  /* 采样 D 个普通叶页、发现它们共指向 E 个外部页 → ordinary:external ≈ D:E
     → ordinary 占比 ≈ D/(D+E)；总页数 T（含外部页）→ ordinary ≈ T * D / (D + E) */
  n_ordinary_leaf_pages = index->stat_n_leaf_pages * data->n_leaf_pages_to_analyze /
                          (data->n_leaf_pages_to_analyze + data->n_external_pages_sum);
}
```

**外部页计数的来源**：`dict_stats_scan_page` 的扫描循环里对每条记录调 `lob::btr_rec_get_externally_stored_len()` 累加（`dict0stats.cc:1166-1169、1216-1219`），经 `dict_stats_analyze_index_below_cur` 汇总到 `n_diff_data->n_external_pages_sum`。**失效场景**：公式假设外部页在整棵树均匀分布——若大 BLOB 行在键空间聚集，采样页恰好碰/碰不到 BLOB 页会导致 E 高估（distinct 低估）或低估（反向高估）；下潜途中撞"整页同键"提前返回时该采样点的 E 计 0，稀释比例；`level == 1` 快捷路径完全不用 E（隐含外部页可忽略）。另注：`stat_n_leaf_pages` **含**外部页（`btr0cur.cc:5745` 注释"our sample actually represents also the pages used for external storage"），所以必须先做此修正。

**逐层扫描的三个细节**（`dict_stats_analyze_index_level`，`dict0stats.cc:793-1086`）：

1. **跨页比较**：相邻记录比较跨越页边界（页尾 vs 下页首）——页尾记录必须 `rec_copy_prefix_to_buf` **拷贝前缀**，因为翻页后 `move_to_next_user_rec` 会释放上一页 latch，指针失效；
2. **每 100 页检查长等待者**（仅 level≠0，持 SX 锁时）：`dict_stats_index_long_waiters()`——索引锁上有等待者**且**距上次无等待已超过 `srv_fatal_semaphore_wait_threshold` 的一半，立即中止（防 ANALYZE 触发 semaphore wait 自杀）；上层减半采样页数重试；
3. **level == 0 提前释放 SX 锁**：全叶层扫描是全表量级操作，继续持 SX 会长时间阻塞 SMO；叶层间靠 `page_get_prev/next` 链接，不需要树形稳定即可正确走链表——代价是扫描期间树结构可能变化（统计退化为近似值）。

**全表扫描捷径的 1e6 上限防什么**（`dict0stats.cc:1729-1773` 注释原文）：level>0 的逐层扫描需要 `dict_index_get_lock` 的 **SX 锁**；若 `n_sample_pages * n_uniq` 超过 100 万，即使比叶页总数少，逐层扫描也要在 SX 锁下跑太久——宁可释放 SX 用较慢的全叶扫描。走捷径时 `stat_n_sample_sizes[i] = total_pages` = 真实扫过的叶页数（即结果是**精确计数**，可通过 `sample_size` 判断统计可信度）。

三个假设，每一个都可能不成立——这就是基数漂移的根源（见 [Misc](#misc)）。

#### transient 采样：随机页 + 补偿项

`btr_estimate_number_of_different_key_vals()`（`btr0cur.cc:5557-5786`）：

```cpp
for (i = 0; i < n_sample_pages; i++) {
  btr_cur_open_at_rnd_pos(index, BTR_SEARCH_LEAF, &cursor, ...);   // 随机挑一个叶子页
  ... 逐记录 cmp_rec_rec_with_match(rec, next_rec, ..., stats_null_not_equal, &matched_fields)
  for (j = matched_fields; j < n_cols; j++) n_diff[j]++;            // 前缀增量计数
}
// 外推
n_diff[j] = BTR_TABLE_STATS_FROM_SAMPLE(n_diff[j], index, n_sample_pages, total_external_size, not_empty_flag);
add_on = index->stat_n_leaf_pages / (10 * (n_sample_pages + total_external_size));
if (add_on > n_sample_pages) add_on = n_sample_pages;
index->stat_n_diff_key_vals[j] += add_on;                          // ★ 补偿项
```

```cpp
// btr0cur.cc:149-163
constexpr uint64_t BTR_TABLE_STATS_FROM_SAMPLE(value, index, sample, ext_size, not_empty) {
  return (value * index->stat_n_leaf_pages + sample - 1 + ext_size + not_empty) / (sample + ext_size);
}
```

`add_on` 的推导（源码注释原文）：当树大于约 `10 * n_sample_pages + ext_size` 时，随机采的几页很可能**一个键边界都看不见**（`n_diff[j]≈0`，外推结果≈0），但至少采样页本身就展示了 ≥ `n_sample_pages` 个不同键值——于是按"树每扩大 10 倍采样规模，就补一个键值"的经验比例补偿，并封顶 `n_sample_pages`（不超出样本直接证明的数量）。

`BTR_TABLE_STATS_FROM_SAMPLE` 宏的另外两个技巧（`btr0cur.cc:150-162`）：

- 分子加 `sample - 1` 是**整数向上取整**：`⌈a/b⌉ = (a + b - 1) / b`，避免小样本截断成 0；
- 加 `not_empty_flag`（首采样页非空则置 1）：非空表保证结果 ≥ 1，避免把非空表估成 0 行。

还有一处单页修正（`btr0cur.cc:5722-5735`）：当该前缀在树内已唯一（`n_cols == n_unique_in_tree`）且页有邻居时 `n_diff[n_cols-1]++`——"第一条记录必然与前页最后一条不同"，修复"每页一条大记录时严重低估行数"的老问题。

**分区表统计**：ANALYZE 是**逐分区独立做完整统计**（`ha_innopart::info_low` 循环每个分区调 `update_table_stats` → `dict_stats_update(DICT_STATS_RECALC_PERSISTENT)`，每个分区是独立 `dict_table_t`、在 `mysql.innodb_table_stats` 各占一行，子表名形如 `t#p#p0`）；行数/页数的**聚合发生在读取时**（`HA_STATUS_VARIABLE` 分支对各分区 `stat_n_rows` 求和）。`ha_innopart.cc:3522` 的 TODO（"Only analyze the PK for all partitions..."）证实当前实现每个分区**所有索引**都完整采样，无跨分区共享采样。

---

### 触发与生命周期

#### 何时重算：10% 判据与后台线程

> **事实纠正**：不存在 `dict_stats_update_if_needed()`；实际是 `row_update_statistics_if_needed()`（`row0mysql.cc:1124-1162`）。

```cpp
counter = table->stat_modified_counter++;        // 后置自增：counter 是自增前的旧值
n_rows = dict_table_get_n_rows(table);           // 注意：n_rows 本身也是估计值

if (dict_stats_is_persistent_enabled(table)) {
  if (counter > n_rows / 10 /* 10% */ && dict_stats_auto_recalc_is_enabled(table)) {
    dict_stats_recalc_pool_add(table);           // ★ 只入队，不在此重算
    table->stat_modified_counter = 0;
  }
  return;                                        // persistent 永不在前台同步重算
}
/* transient：6.25%（16 + n_rows/16）*/
if (counter > 16 + n_rows / 16) {
  dict_stats_update(table, DICT_STATS_RECALC_TRANSIENT);   // ★ 前台同步采样，代价落在用户线程
}
```

三条关键性质：

1. **分母 `n_rows` 本身是估计值**——用一个估计值判断另一个估计值是否过期；
2. **小表陷阱**：`n_rows=5` 时 `n_rows/10 == 0`，第二次 DML 就满足 `counter > 0`；
3. **persistent 表只入队**，真正采样交给后台线程 `dict_stats_thread`，前台 DML 永不被阻塞。

后台线程（`dict0stats_bg.cc:354-389`）：

```cpp
constexpr std::chrono::seconds MIN_RECALC_INTERVAL{10};    // 节流：10 秒
...
os_event_wait_time(dict_stats_event, MIN_RECALC_INTERVAL);  // 无事件也至少 10 秒自醒一次（防丢 event）
dict_stats_process_entry_from_recalc_pool(thd);             // ★ 一次只处理一个表
```

处理时若 `now - stats_last_recalc < MIN_RECALC_INTERVAL` 就把表**塞回队尾**本次不算。

**滞后性根因**：10% 阈值 + 10 秒节流 + 一次一个表 → 繁忙实例上统计可能长时间滞后。

首次打开表时 `dict_stats_init()` 用 `DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY` 从磁盘拉取；若磁盘没有且 `auto_recalc` 开着就现算并写盘。`dict_stats_deinit()` 在最后一次 close 时置 `stat_initialized=false`——**这就是 `FLUSH TABLE` 能刷新统计的机制**。

> ⚠️ 表还开着时 `DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY` 直接 `return`（`dict0stats.cc:2895`）——磁盘上的新统计不会被已打开的表读到。

#### `dict_stats_save`：写回与死锁规避

`dict_stats_save()`（`dict0stats.cc:2187-2355`）用 InnoDB 内建 SQL 解析器执行 DELETE+INSERT：

```cpp
table = dict_stats_snapshot_create(table_orig);   // 影子 dict_table_t，后续不必持表锁
// 表级：独立事务，DELETE 再 INSERT
// 索引级：单独一个后台事务，所有索引的所有 stat_name 一次提交
index_map_t indexes((ut_strcmp_functor()), ...);  // ★ 按索引名排序
for (index = table->first_index(); ...) indexes[index->name] = index;
for (it = indexes.begin(); ...) { ... n_diff_pfx01..NN, n_leaf_pages, size ... }
```

**为什么要按名字排序**：`DROP TABLE` 会一次 `DELETE FROM innodb_index_stats WHERE database_name=.. AND table_name=..` 锁多行；若统计写入顺序与它不一致就死锁。排序保证与 PK 顺序 `(database_name, table_name, index_name, stat_name)` 一致。

`dict_stats_update()` 在 persistent 重算前还会**唤醒 purge**（`dict0stats.cc:2851`）：

```cpp
if (trx_sys->rseg_history_len.load() > 0) srv_wake_purge_thread_if_not_active();
```

---

### 喂给优化器

#### `rec_per_key` 的传递链

```
dict_index_t::stat_n_diff_key_vals[i]  +  dict_table_t::stat_n_rows
        ↓  innodb_rec_per_key()                       ha_innodb.cc:17023
        ↓  KEY::set_records_per_key(j, rec_per_key)   ha_innodb.cc:17457（info_low 的 HA_STATUS_CONST 段）
        ↓  KEY::records_per_key(j)                    sql/key.h:237（优先浮点版 rec_per_key_float）
        ↓  sql_planner.cc best_access_path 的 fanout 估算 / opt_statistics.cc
```

`innodb_rec_per_key()` 核心：

```cpp
n_diff = index->stat_n_diff_key_vals[i];
if (n_diff == 0) rec_per_key = records;
else if (srv_innodb_stats_method == SRV_STATS_NULLS_IGNORED) { ... n_null = records - n_non_null; ... }
else rec_per_key = (rec_per_key_t)records / n_diff;     // ★ 核心公式
if (rec_per_key < 1.0) rec_per_key = 1.0;               // 下限
```

`info_low()` 里还有两处容易踩坑的细节：

```cpp
// ① 优化器永不看到 0 行
if (n_rows == 0 && !(flag & HA_STATUS_TIME) && ...) n_rows++;

// ② legacy 整型数组被 /2（"Since MySQL seems to favor table scans too much..."）
rec_per_key_int = static_cast<ulong>(innodb_rec_per_key(index, j, stats.records));
rec_per_key_int = rec_per_key_int / 2;
key->rec_per_key[j] = rec_per_key_int;
```

实际生效的是**浮点版**（`records_per_key()` 优先读它），上式只是 legacy fallback。

#### index dive 与 statistics 模式的切换

`records_in_range`（`ha_innodb.cc:16694`）→ `btr_estimate_n_rows_in_range_low()`：两次下潜记录 path1/path2，自顶向下比对；跨多页时调 `btr_estimate_n_rows_in_range_on_level()`，**最多读 10 页**（`N_PAGES_READ_LIMIT`）然后外推，并对高树 ×2 修正、被 `table_n_rows/2` 上限裁剪；并发导致树变化时重试 4 次后**返回常数 10**。最后 `records_in_range` 把 0 改成 1（避免优化器直接返回 Empty set）。

优化器侧的切换（`handler.cc:6266-6279`）：

```cpp
if ((range.range_flag & UNIQUE_RANGE) && !(range.range_flag & NULL_RANGE))
  rows = 1;                                              // 唯一键：最多一行
else if (range.range_flag & SKIP_RECORDS_IN_RANGE && ...) {
  rows = table->key_info[keyno].records_per_key(keyparts_used - 1);   // ★ 用统计，零 I/O
} else {
  rows = this->records_in_range(keyno, min_endp, max_endp);           // ★ index dive
}
```

`SKIP_RECORDS_IN_RANGE` 由 `eq_ranges_exceeds_limit(tree, &range_count, thd->variables.eq_range_index_dive_limit)` 决定——**默认 200**，超过就用统计。

---

## 相关的系统变量/状态变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `innodb_stats_persistent` | **ON** | 全局默认；可被表级 `STATS_PERSISTENT` 覆盖 |
| `innodb_stats_persistent_sample_pages` | **20** | 持久化采样叶子页数；表级 `STATS_SAMPLE_PAGES` 优先 |
| `innodb_stats_transient_sample_pages` | **8** | 仅 transient 模式 |
| `innodb_stats_auto_recalc` | **ON** | 仅 persistent 表生效（10% 判据） |
| `innodb_stats_include_delete_marked` | **OFF** | 只影响 persistent 采样；**transient 无效** |
| `innodb_stats_method` | `nulls_equal` | `nulls_equal`/`nulls_unequal`/`nulls_ignored` |
| `innodb_stats_on_metadata` | **OFF** | `SHOW INDEX`/`SHOW TABLE STATUS` 是否触发 transient 重算 |
| `eq_range_index_dive_limit` | **200** | 等值 range 超限时用 `records_per_key` 代替 dive（`0` = 永远 dive） |
| `information_schema_stats_expiry` | 86400（24h） | `I_S`/`SHOW` 统计缓存的过期时间 |

表级覆盖：`CREATE/ALTER TABLE ... STATS_PERSISTENT=n | STATS_AUTO_RECALC=n | STATS_SAMPLE_PAGES=n`。

---

## Misc

### 统计项总表

| 统计项 | 计算方法 | 持久化 | 影响谁 |
|---|---|---|---|
| `stat_n_leaf_pages` | `btr_get_size(BTR_N_LEAF_PAGES)` —— **精确** | `n_leaf_pages` | 外推基数、缓存比例 |
| `stat_index_size` | `btr_get_size(BTR_TOTAL_SIZE)` —— **精确** | `size` | 索引空间估算 |
| `stat_n_diff_key_vals[n]` | 分层采样 + REF01 外推 / 随机页 + 补偿 | `n_diff_pfxNN` | **`rec_per_key` → 优化器选择率** |
| `stat_n_sample_sizes[n]` | 实际下潜页数 | `sample_size` | 仅可观测 |
| `stat_n_non_null_key_vals[n]` | 仅 transient + `nulls_ignored` | **不落盘** | `rec_per_key` 的 nulls_ignored 分支 |
| `stat_n_rows` | = 聚簇索引 `stat_n_diff_key_vals[n_uniq-1]` | `n_rows` | `stats.records` → 全表扫描代价 |

### 为什么索引基数会漂移：源码级根因

| # | 根因 | 证据 |
|---|---|---|
| 1 | 采样规模极小（20 页 / 8 页）对百万页表是 2e-5 采样率 | `srv_stats_persistent_sample_pages=20` |
| 2 | 外推假设 `R` 在 LA 层与叶子层相同——键分布倾斜时崩塌 | `dict_stats_index_set_n_diff` 注释 "we assume this ratio is the same" |
| 3 | "A 页平均不同值数代表全表"——页间密度方差被 ×N 放大 | 同上 |
| 4 | transient 的 `add_on` 是拍脑袋常数 | `btr0cur.cc:5761` |
| 5 | `stat_n_rows` 本身就是估计值，却是 `rec_per_key` 的分子 | `dict_stats_update_transient` dict0stats.cc:728 |
| 6 | `stat_modified_counter` 无锁、rollback 不回滚 | dict0mem.h:2299 |
| 7 | 10% 阈值 + `n_rows` 是估计值；小表 `n_rows/10==0` 频繁触发 | row0mysql.cc:1144 |
| 8 | 后台线程 10 秒节流 + 一次一个表 | dict0stats_bg.cc:51 |
| 9 | 树变化或锁等待时**静默放弃**，未算完的前缀保留旧值 | dict0stats.cc:1821/1888/1971 |
| 10 | 锁竞争使采样页数减半重试 → 系统性偏低 | dict0stats.cc:1894/1991 |
| 11 | delete-marked 默认排除，DELETE 后未 purge 前不一致 | dict0stats.cc:921/1352 |
| 12 | `stat_n_rows` 与 `n_diff` 可能算于不同时刻，且 `HA_STATUS_NO_LOCK` 下不持锁读 | ha_innodb.cc:17432 注释 "This is acceptable" |
| 13 | persistent 不保存 `stat_n_non_null_key_vals` → `nulls_ignored` 退化成 `rec_per_key=1.0` | dict0stats.cc:2660 |
| 14 | range 估计被 `table_n_rows/2` 裁剪，`table_n_rows` 又是估计值 → 误差传递 | btr0cur.cc:5417 |
| 15 | 并发下 `records_in_range` 重试 4 次后返回常数 **10** | btr0cur.cc:5183/5445 |
| 16 | "boring record" 假定整页同键的叶子页 n_diff==1，页分裂后可能低估 | dict0stats.cc:63/1318 |

### 易混淆点

- **`innodb_stats_method` 对 persistent 采样几乎无效**：`dict_stats_scan_page()` 里 `nulls_unequal` **硬编码 `false`**（dict0stats.cc:1185）；且 `stat_n_non_null_key_vals` 不落盘（fetch 时置 0）→ `nulls_ignored` + persistent 会让 `rec_per_key` 恒为 1.0，反而让优化器以为索引极好。
- **`dict_stats_update_if_needed` / `dict_stats_t` / `dict_stats_update_persistent_for_index` / `innodb_analyze_is_persistent` 在 8.0.39 都不存在**（凭记忆写必错）。
- **`n_rows` 是估出来的**，不是 `COUNT(*)`；`SHOW TABLE STATUS` 的 Rows 同理。
- **`SHOW INDEX` 的 Cardinality 有缓存**：默认最多陈旧 `information_schema_stats_expiry`（24h）；立刻取最新值用 `SET information_schema_stats_expiry=0`。
- **`INNODB_INDEXES` 没有 `N_DIFF_FIELDS` 列**（5.7 的 `INNODB_INDEX_STATS` 已移除），看 `n_diff` 要查 `mysql.innodb_index_stats`。

### 实践建议（对应源码）

1. 单表提高采样：`ALTER TABLE t STATS_SAMPLE_PAGES=200;`（落到 `dict_table_t::stats_sample_pages`，由 `N_SAMPLE_PAGES()` 优先使用）。
2. 大批量 DML 后显式 `ANALYZE TABLE`（且它会先唤醒 purge）。
3. 用 `I_S.INNODB_TABLESTATS.MODIFIED_COUNTER / NUM_ROWS` 判断统计新鲜度。
4. 怀疑采样被降级时看 error log 的 `ER_IB_MSG_STATS_SAMPLING_TOO_LARGE`。
5. range 计划长期偏：调 `eq_range_index_dive_limit`（0 = 永远 dive）。

### 可观测入口

| 入口 | 数据来源 |
|---|---|
| `SHOW INDEX` 的 `Cardinality` | `stats.records / KEY::records_per_key(j)` ≈ `stat_n_diff_key_vals[j]`（经 `INTERNAL_INDEX_COLUMN_CARDINALITY` + 缓存） |
| `mysql.innodb_index_stats` | 权威：`n_diff_pfxNN` / `n_leaf_pages` / `size` + `sample_size` |
| `I_S.INNODB_TABLESTATS` | `NUM_ROWS`、`CLUST_INDEX_SIZE`（页）、`OTHER_INDEX_SIZE`（页）、**`MODIFIED_COUNTER`** |
| `I_S.INNODB_INDEXES` | 无统计列（只有 `TYPE`/`N_FIELDS`/`PAGE_NO`/`MERGE_THRESHOLD`）；访问需 PROCESS 权限 |

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.8.10.1 Configuring Persistent Optimizer Statistics Parameters*
- *MySQL 8.0 Reference Manual → 17.8.10.2 Configuring Non-Persistent Optimizer Statistics Parameters*
- *MySQL 8.0 Reference Manual → 13.7.3.1 ANALYZE TABLE Statement*

**论文**
- Selinger, P. et al. *Access Path Selection in a Relational Database Management System*. SIGMOD, 1979.（用统计驱动访问路径选择的经典）

**相关文档**
- 行数估算的 B-tree 搜索路径：[`btr.md`](btr.md)
- 索引类型与 `n_uniq` 粒度：[`types.md`](types.md)
- 索引访问优化（ICP/MRR 依赖的 `rec_per_key`）：[`access.md`](access.md)
- 优化器代价与访问路径：[`../server/query/07_optimize/`](../server/query/07_optimize/)
- 索引 DDL 与可观测：[`operations.md`](operations.md)
