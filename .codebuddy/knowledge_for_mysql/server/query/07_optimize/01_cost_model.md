# 01 代价模型与统计信息

> 代价模型（物理优化的判据）+ 统计信息（代价模型的输入）。两者是 ②③ 物理优化的前提。

## 目录

- [零、总览](#零总览)
- [一、代价模型架构](#一代价模型架构)
- [二、IO 代价：page_read_cost 与 in_mem](#二io-代价page_read_cost-与-in_mem)
- [三、handler 层接口与引擎实现](#三handler-层接口与引擎实现)
- [四、rec_per_key](#四rec_per_key)
- [五、InnoDB 统计采集算法](#五innodb-统计采集算法)
- [六、持久化 vs 瞬态 + 自动更新](#六持久化-vs-瞬态--自动更新)
- [七、index dive（records_in_range）](#七index-diverange_rows)
- [八、直方图](#八直方图)
- [九、条件过滤 condition filtering](#九条件过滤-condition-filtering)
- [十、无统计时的兜底估算](#十无统计时的兜底估算)

---

## 零、总览

```
统计信息（输入）                     代价模型（计算）              选择（输出）
表行数 / NDV / rec_per_key    →   page_read_cost              →  最小代价计划
直方图 / in_mem 比例              row_evaluate_cost
```

统计数据分两类：

| 类别 | 内容 | 存储 |
|------|------|------|
| 表/索引统计 | `stats.records`、`rec_per_key[]`、NDV、页数 | `mysql.innodb_table_stats` / `innodb_index_stats` |
| 列统计（直方图） | histogram JSON、采样率、NULL 比例、NDV | Data Dictionary `dd::Column_statistics`（经 `information_schema.COLUMN_STATISTICS` 暴露） |

---

## 一、代价模型架构

### 1.1 三层结构

| 类 | 位置 | 职责 |
|---|---|---|
| `Cost_model_server` | `opt_costmodel.h:52` | 与具体表无关，持有 server 级常量 |
| `Cost_model_table` | `opt_costmodel.h:240` | 表级：委托 server 常量 + 本引擎 `SE_cost_constants` + 本表 |
| `Cost_estimate` | **`handler.h:3709`**（不在 opt_costmodel.h） | 代价累加器 |

`Cost_model_table` 不自己存全局常量，而是**委托**：`row_evaluate_cost()`/`key_compare_cost()` 转发给 `m_cost_model_server`，只有 IO 类接口用 `m_se_cost_constants`。挂载点是 `TABLE::m_cost_model`，优化器里统一形如 `table->cost_model()->xxx()`。

**初始化链**（`opt_costmodel.cc:45-77`）：每次优化新查询都要重新 `init()`——因为常量可能被 `FLUSH OPTIMIZER_COSTS` 换掉。

### 1.2 Cost_estimate：total_cost 公式

```cpp
// sql/handler.h:3709
class Cost_estimate {
  double io_cost;      double cpu_cost;
  double import_cost;  double mem_cost;   // 内存字节数
 public:
  /// Returns sum of time-consuming costs, i.e., not counting memory cost
  double total_cost() const { return io_cost + cpu_cost + import_cost; }
};
```

**关键**：

1. **`total_cost() = io + cpu + import`，`mem_cost` 严格不参与比较**。它是"内存预算"维度（字节数），只用于判断能否用 hash join / 内存临时表，不参与 plan 优劣比较。
2. `multiply()` 只乘 io/cpu/import，**注释明确写 `/* Don't multiply mem_cost */`**——mem 是绝对值不是代价率。
3. `set_max_cost()` 把 `io_cost = DBL_MAX` 表达"不可行"；后续算术有 `assert(!is_max_cost())`。

典型写入：`table_scan_cost()` 只写 io 维度，CPU 由上层单独追加（如 `sql_planner.cc:693` 的 `prefix_rowcount * row_evaluate_cost(cur_fanout)`）。

### 1.3 全部常量与默认值（已核实）

**server 级 6 个**（`opt_costconstants.cc:56-85`）：

| 常量 | 默认值 | 行 |
|---|---|---|
| `ROW_EVALUATE_COST` | **0.1** | :56 |
| `KEY_COMPARE_COST` | **0.05** | :59 |
| `MEMORY_TEMPTABLE_CREATE_COST` | **1.0** | :65（注释：基准测得"建内存临时表 ≈ 写 10 行"） |
| `MEMORY_TEMPTABLE_ROW_COST` | **0.1** | :71 |
| `DISK_TEMPTABLE_CREATE_COST` | **20.0** | :76（注释：MyISAM 建表比 Memory 慢 20 倍） |
| `DISK_TEMPTABLE_ROW_COST` | **0.5** | :85 |

**引擎级 2 个**（`opt_costconstants.cc:148-152`）：

| 常量 | 默认值 | 行 |
|---|---|---|
| `MEMORY_BLOCK_READ_COST` | **0.25** | :149 |
| `IO_BLOCK_READ_COST` | **1.0** | :152 |

**编译期硬编码 3 个**（`opt_costmodel.h:41-43`，**不可配置**）：

```cpp
constexpr const double DISK_SEEK_BASE_COST{0.9};
constexpr const int BLOCKS_IN_AVG_SEEK{128};
constexpr const double DISK_SEEK_PROP_COST{0.1 / BLOCKS_IN_AVG_SEEK};
```

标定条件：**平均一次 seek（跳过 128 块）总代价 = 1.0**，即 `0.9 + (0.1/128)*128 = 1.0`。

### 1.4 系统表与配置方式

- server 常量 → **`mysql.server_cost`**；engine 常量 → **`mysql.engine_cost`**（建表 `scripts/mysql_system_tables.sql:529-582`）
- `default_value` 是 **GENERATED VIRTUAL 列**（SQL 里硬编码默认值，所以源码注释说"改默认值要同步改这里"）
- 用法：`UPDATE mysql.server_cost SET cost_value=X WHERE cost_name='row_evaluate_cost';` + `FLUSH OPTIMIZER_COSTS`（`cost_value=NULL` 表示用默认）
- `engine_cost` 主键 `(cost_name, engine_name, device_type)`：可配 `engine_name='default'`（全局）与具体 `engine_name='InnoDB'`（覆盖）。**引擎特定值永远优先**（`opt_costconstants.cc:196-215`）
- 加载：`opt_costconstantcache.cc:395-407`，`reload()` 由 `FLUSH OPTIMIZER_COSTS` 触发
- 值必须 `> 0.0`，否则 `INVALID_COST_VALUE`

> `MAX_STORAGE_CLASSES = 1`（`opt_costconstants.h:58`）——每引擎当前只有一套常量（device_type 维度预留未启用）。

---

## 二、IO 代价：page_read_cost 与 in_mem

### 2.1 公式（`opt_costmodel.cc:79`）

```cpp
double Cost_model_table::page_read_cost(double pages) const {
  const double in_mem = m_table->file->table_in_memory_estimate();
  const double pages_in_mem = pages * in_mem;
  const double pages_on_disk = pages - pages_in_mem;
  return buffer_block_read_cost(pages_in_mem) + io_block_read_cost(pages_on_disk);
}
```

即线性插值混合模型：

```
cost(pages) = pages × [ in_mem × 0.25 + (1 - in_mem) × 1.0 ]
```

> **命名陷阱**：`buffer_block_read_cost()` 用的是 `memory_block_read_cost` 常量（0.25）——"buffer 命中"≡"从内存块读"。

`page_read_cost_index(index, pages)`（`:346`）同理，但用 `index_in_memory_estimate(index)`（二级索引自己的 in_mem 率）。**区别很重要**：`read_cost`/`table_scan_cost` 用表级 in_mem（要回表），`index_scan_cost` 用索引级。

### 2.2 in_mem 的两个来源（`handler.cc:5950-5995`）

**① 引擎主动上报** `stats.table_in_mem_estimate`（InnoDB 在 `ha_innodb.cc:17409` 设 `pct_cached`）。

**② 否则启发式** `estimate_in_memory_buffer`（`handler.cc:5997-6052`）：

```cpp
longlong memory_buf_size = get_memory_buffer_size();
if (memory_buf_size <= 0) memory_buf_size = 100 * 1024 * 1024;   // 兜底 100 MB
const double table_index_in_memory_limit = 0.2;
const double percent_of_mem = (double)table_index_size / memory_buf_size;
if (percent_of_mem < 0.2)       in_mem_est = 1.0;      // 表 < buffer 的 20% → 全内存
else if (percent_of_mem > 1.0)  in_mem_est = 0.0;      // 表比 buffer 大 → 全磁盘
else in_mem_est = 1.0 - (percent_of_mem - 0.2) / (1.0 - 0.2);   // 线性插值
```

分段线性，**拐点默认 20%**，缓冲区大小未知时兜底 **100 MB**。

### 2.3 disk_seek_cost

```cpp
double disk_seek_cost(double seek_blocks) const {
  return disk_seek_base_cost() + disk_seek_prop_cost() * seek_blocks;
}
```
base/prop 都要再乘 `io_block_read_cost(1.0)`，所以引擎调 `io_block_read_cost` 会让 seek 代价整体按比例变化。

---

## 三、handler 层接口与引擎实现

### 3.1 旧接口 → 新接口的包装

旧接口返回裸 `double`（页数），新接口返回 `Cost_estimate`。转换统一为 **`pages × page_read_cost(1.0)`**（`handler.cc:6054-6105`）：

```cpp
Cost_estimate handler::table_scan_cost() {
  const double io_cost = scan_time() * table->cost_model()->page_read_cost(1.0);
  Cost_estimate cost; cost.add_io(io_cost); return cost;
}
Cost_estimate handler::index_scan_cost(uint index, double ranges, double rows) {
  const double io_cost = index_only_read_time(index, rows) *
                         table->cost_model()->page_read_cost_index(index, 1.0);
  ...
}
Cost_estimate handler::read_cost(uint index, double ranges, double rows) {
  const double io_cost = read_time(index, ranges, rows) *
                         table->cost_model()->page_read_cost(1.0);
  ...
}
```

**三者都只写 io_cost**（注释说这样写是为了让编译器做 RVO），CPU 由调用方追加。

默认基线公式：
- `scan_time()` = `data_file_length / IO_SIZE + 2`（页数 + 2 固定启动开销）
- `read_time()` = `ranges + rows`（**完全不看页数**）
- `index_only_read_time()`（`handler.cc:5940`）：
  ```cpp
  keys_per_block = (stats.block_size/2 / (key_length + ref_length)) + 1;   // 假设索引页半满
  read_time = ceil(records / keys_per_block);
  ```

### 3.2 InnoDB 实现

**`ha_innobase::scan_time()`**（`ha_innodb.cc:16864`）：

```cpp
/* Since MySQL seems to favor table scans too much over index searches,
   we pretend that a sequential read takes the same time as a random
   disk read, that is, we do not divide the following by 10, which
   would be physically realistic. */
return ((double)stat_clustered_index_size);
```

**= 聚簇索引的页数，故意不除以 10**——物理上顺序读应比随机读便宜约 10 倍，但注释明说这是为了压低优化器对全表扫的偏好。（`m_prebuilt==nullptr` 时退回通用公式。）

**`ha_innobase::read_time()`**（`ha_innodb.cc:16898`）三个分支：

| 情况 | 公式 |
|---|---|
| 二级索引（`index != primary_key`） | 退回通用 `ranges + rows` |
| 聚簇索引且 `rows <= 2` | `rows`（1~2 页） |
| 聚簇索引且更多行 | `ranges + (rows / total_rows) × scan_time()` |

即"按比例切分全表扫页数，每个 range 再加一次 seek"。若 `rows` 超过表总行数上界，直接给全表扫代价。

---

## 四、rec_per_key

### 4.1 定义与两份数组

`KEY` 里有**两份**数组（`sql/key.h:161` / `:195`）：

- `ulong *rec_per_key` —— legacy 整数版
- `rec_per_key_t *rec_per_key_float`（`typedef float rec_per_key_t`）—— 浮点版，**主用**

```cpp
rec_per_key_t records_per_key(uint key_part_no) const {
  if (rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN)   // -1.0f
    return rec_per_key_float[key_part_no];
  return (rec_per_key[key_part_no] != 0) ? rec_per_key[key_part_no] : REC_PER_KEY_UNKNOWN;
}
```

**语义**：`rec_per_key[i]` = "前 i+1 个 key part 组成的键前缀，平均每个不同键对应多少行"。对 `(a,b,c)`：`rec_per_key[0]`=每个 a 值的平均行数，`rec_per_key[2]`=每个 `(a,b,c)` 的平均行数。断言 `rec_per_key >= 1.0`（`key.h:265`）。

### 4.2 InnoDB 计算（`innodb_rec_per_key`，`ha_innodb.cc:17034`）

主公式 **`rec_per_key = records / n_diff`**（`records = stat_n_rows`，`n_diff[i] = stat_n_diff_key_vals[i]`）。四条路径：

| 情况 | 结果 |
|---|---|
| `records == 0`（空表） | 1.0（避免除零） |
| `n_diff == 0` | `records`（全是同一个值） |
| `innodb_stats_method = nulls_ignored` | 先剔 NULL：`(records-n_null)/(n_diff-n_null)`；若 NULL 比 NDV 还多 → 1.0 |
| 其余 | `records / n_diff` |

最后统一 `max(rec_per_key, 1.0)`——采样误差会让比值 < 1，但"每个键值少于 1 行"无意义。

### 4.3 legacy 整数版的历史遗留（`ha_innodb.cc:17412`）

```cpp
const rec_per_key_t rec_per_key = innodb_rec_per_key(index, j, index->table->stat_n_rows);
key->set_records_per_key(j, rec_per_key);          // 浮点版：不除 2

/* legacy */
ulong rec_per_key_int = innodb_rec_per_key(index, j, stats.records);
rec_per_key_int = rec_per_key_int / 2;             // ★ 故意除以 2
key->rec_per_key[j] = rec_per_key_int;
```

注释：*"Since MySQL seems to favor table scans too much over index searches, we pretend index selectivity is 2 times better than our estimate"*。因为 `records_per_key()` 优先读浮点版，**现代 8.0 实际生效的是未除 2 的浮点值**；整数版只在引擎没填浮点版时才生效。

**何时更新**：首次打开表（`dict_stats_init`）、`ANALYZE TABLE`、自动重算（见第六节）。

---

## 五、InnoDB 统计采集算法

### 5.1 采样参数（已核实）

```cpp
// storage/innobase/srv/srv0srv.cc:578-582
unsigned long long srv_stats_transient_sample_pages = 8;
bool  srv_stats_persistent = true;
bool  srv_stats_include_delete_marked = false;
unsigned long long srv_stats_persistent_sample_pages = 20;
bool  srv_stats_auto_recalc = true;
```

- **persistent 默认 20 页**，**transient 默认 8 页**
- `N_SAMPLE_PAGES(index)` 优先用表级 `stats_sample_pages`（`STATS_SAMPLE_PAGES=N`，范围 [1,65535]）
- `N_DIFF_REQUIRED(index) = N_SAMPLE_PAGES * 10`（找"至少含 A×10 个 distinct 值"的层）

### 5.2 算法总纲（源码自带伪代码，`dict0stats.cc:764`）

```
dict_stats_analyze_index_low(N):
  for each n_prefix:                      # ★ 从最长前缀往短前缀走
    search for good enough level:         # 逐层下降
      dict_stats_analyze_index_level()
      if n_diff_on_level >= N*10 or level==1: stop
    dict_stats_analyze_index_for_n_prefix(that level)   # 采样 N 个叶页
      dict_stats_analyze_index_below_cur()              # 下潜到叶页
```

**关键优化（跨前缀复用）**：从最长前缀往短前缀走。若对 n 列前缀"首个含 D 个 distinct 的层"是 L，则对 n-1 列该层只可能是 L 或更低，所以可以接着往下走，不必回到 root。

**层下降停止条件**：`n_diff >= A*10` **或** 已到 `level==1`（**永不下降到第 0 层/叶层**）。

**保护**：若下一层将扫超过 A 个页（利用"第 L 层页数 = 第 L+1 层记录数"），退回上一层。

### 5.3 层内一次扫描算所有前缀的 NDV（`dict_stats_analyze_index_level`）

```cpp
cmp_rec_rec_with_match(rec, prev_rec, ..., &matched_fields);
for (i = matched_fields; i < n_uniq; i++) {
  n_diff_boundaries[i].push_back(*total_recs - 2);
  n_diff[i]++;       // 对前缀长度 > matched_fields 的所有 i，这是新 distinct 值
}
```

对**相邻两条记录**比较得到 `matched_fields` = 前缀匹配列数；只匹配 k 列时，所有前缀长度 `> k` 的都算新 distinct 值。**一次 O(记录数) 扫描同时得到 `n_diff[0..n_uniq-1]`**，无需为每列单独扫。

### 5.4 分层随机采样（`dict_stats_analyze_index_for_n_prefix`）

```cpp
const uint64_t left  = n_diff * i / n_pick;
const uint64_t right = n_diff * (i + 1) / n_pick - 1;
const uint64_t rnd = ut::random_from_interval(left, right);
const uint64_t dive_below_idx = boundaries->at(rnd);
```

把该层所有 distinct 组**均分成 A 段**，每段内**随机取一个组**，取该组最后一条记录下潜到它下面的叶页。保证采样点**均匀覆盖整个键值域**。

**跨页重复计数修正**：每个采样页的 distinct 数减 1（页尾值与下一页页首值相同会被重复计入）。

### 5.5 REF01：NDV 外推公式（`dict0stats.cc:1619`）

```cpp
index->stat_n_diff_key_vals[n_prefix - 1] =
    n_ordinary_leaf_pages                                        // N
  * data->n_diff_on_level / data->n_recs_on_level                // R = N_DIFF_LA/TOTAL_LA
  * data->n_diff_all_analyzed_pages / data->n_leaf_pages_to_analyze;  // N_DIFF_AVG_LEAF
```

即 **`NDV ≈ N × R × N_DIFF_AVG_LEAF`**：

| 项 | 含义 |
|---|---|
| `N` | 普通（非溢出）叶页数。若采样层是 level 1 则精确（= level 1 记录数）；否则 `T × D/(D+E)` 剔除 BLOB 外部页 |
| `R` | 该层 distinct 密度，**假设"非叶层密度 = 叶层密度"** |
| `N_DIFF_AVG_LEAF` | 采样页平均 distinct 值数 `(P1+...+PA)/A` |

### 5.6 五个误差来源（为什么是"非精确统计"）

1. **只采 A 个叶页**（默认 20）外推到全部 N 页——假设"各叶页 distinct 密度均匀"，数据倾斜时误差极大
2. **采样层被迫截断**：到 `level==1` 或"下层页数 > A"就停，此时 `n_diff < A*10`，R 的分母很小
3. **R 的可移植假设**："非叶层密度 = 叶层密度"（源码明说 "we assume this ratio is the same"）
4. **外部页剔除本身是估计**
5. **delete-marked 记录被跳过**，且 `stat_n_rows` 与 `stat_n_diff_key_vals` 采集时点可能不同

另外有**重试收缩**：遇锁等待时 `n_sample_pages` 减半重试（`dict0stats.cc:1894`），所以**实际采样页数可能远小于配置的 20**。

---

## 六、持久化 vs 瞬态 + 自动更新

### 6.1 三种模式

| 维度 | persistent（默认） | transient |
|---|---|---|
| 触发 | `DICT_STATS_RECALC_PERSISTENT` | `DICT_STATS_RECALC_TRANSIENT` |
| NDV 算法 | 上面的分层采样（A=20） | `btr_estimate_number_of_different_key_vals`（随机采 8 个叶页，`dict0stats.cc:675`） |
| 落盘 | 是 | 否（仅内存） |
| 重启 | 从 `mysql.innodb_*_stats` 读回 | 重算 |

`innodb_stats_persistent` 默认 **true**。

### 6.2 存储格式

`mysql.innodb_index_stats` 是**行式**，`stat_name` ∈ {`size`, `n_leaf_pages`, `n_diff_pfx01`, `n_diff_pfx02`, ...}，配 `stat_value` 与 `sample_size`；主键 `(database_name, table_name, index_name, stat_name)`。

`mysql.innodb_table_stats` 存 `(n_rows, clustered_index_size, sum_of_other_index_sizes)`。

### 6.3 自动更新阈值（`row0mysql.cc:1124`）

```cpp
counter = table->stat_modified_counter++;
n_rows  = dict_table_get_n_rows(table);
if (dict_stats_is_persistent_enabled(table)) {
  if (counter > n_rows / 10 && dict_stats_auto_recalc_is_enabled(table)) {   // ★ 10%
    dict_stats_recalc_pool_add(table);
    table->stat_modified_counter = 0;
  }
  return;
}
if (counter > 16 + n_rows / 16) { ... }   // 瞬态：6.25% + 16 行
```

- **持久统计**：修改行数 **> 10%** 加入 recalc pool（后台线程执行）
- **瞬态统计**：`> 6.25% + 16`（+16 避免极小的"计数器表"被过度触发）
- 表级 `STATS_AUTO_RECALC=1/0` **优先**于全局 `innodb_stats_auto_recalc`（默认 true）

---

## 七、index dive（range_rows）

### 7.1 双向下潜 + 路径对比（`btr0cur.cc:5204`）

1. 两次独立 `btr_cur_search_to_nth_level(..., BTR_ESTIMATE)`，每次把**从 root 到 leaf 的整条路径**记进 `path1`/`path2`
2. 无边界时用 `btr_cur_open_at_index_side` 直接定位最左/最右，**不计入边界**（精确处理开闭区间）
3. 逐层对比两条路径，三阶段：

| 阶段 | 条件 | 算法 |
|---|---|---|
| **0 未分叉** | `slot1->nth_rec == slot2->nth_rec` | 继续往下一层 |
| **1 刚分叉** | 首次 `nth_rec` 不同 | `n_rows = slot2->nth_rec - slot1->nth_rec - 1`；若间隔 > 0 标记 `diverged_lot` |
| **2 大量分叉** | `diverged_lot` | 左右边界各贡献本层剩余/前置记录数，再调 `btr_estimate_n_rows_in_range_on_level` **真扫中间页** |

### 7.2 扫页上限与外推（`btr0cur.cc:5041`）

```cpp
constexpr uint32_t N_PAGES_READ_LIMIT = 10;
...
n_rows += page_get_n_recs(page);
if (n_pages_read == N_PAGES_READ_LIMIT || page_id.page_no() == FIL_NULL) goto inexact;
...
inexact:
  n_rows = n_rows_on_prev_level * n_rows / n_pages_read;   // 已扫页平均记录数 × 该层总页数
  *is_n_rows_exact = false;
```

沿层链表向右扫，累加每页记录数，**最多扫 10 页**；提前中断则按"已扫页平均记录数 × 该层总页数"外推。一页都没扫到直接返回 **10**。

### 7.3 两条重要修正（`btr0cur.cc:5407-5430`）

```cpp
if (i > divergence_level + 1 && !is_n_rows_exact) n_rows = n_rows * 2;   // ★ ×2
if (n_rows > table_n_rows / 2 && !is_n_rows_exact) n_rows = table_n_rows / 2;  // ★ 上限截断
```

1. **树高 > 1 且估计不精确时 ×2**——承认算法系统性低估
2. **非精确估计上限截断为表总行数的一半**——防止 range 估出比全表还多

**重试**：两次下潜之间树被改则重试，最多 4 次（`rows_in_range_max_retries = 4`），仍失败返回硬编码 **10 行**（`rows_in_range_arbitrary_ret_val`）。

---

## 八、直方图

### 8.1 类型与选型

```cpp
enum class enum_histogram_type { EQUI_HEIGHT, SINGLETON };   // histogram.h:312
```

选型（`histogram.cc:442`）：**`桶数 >= distinct 值数` → Singleton（每桶一个值，精确）；否则 → Equi-height（近似）**。

参数（已核实）：
- **默认 100 桶，上限 1024**（`sql_yacc.yy:182-188`；`DEFAULT_NUMBER_OF_HISTOGRAM_BUCKETS=100`，`MAX_NUMBER_OF_HISTOGRAM_BUCKETS=1024`）
- `histogram_generation_max_mem_size` 默认 **20000000 字节（20 MB）**，范围 [1000000, SIZE_MAX]，session 级（`sys_vars.cc:3369`）

**采样率由内存预算反推**（`histogram.cc:1415`）：
```
sample_percentage = min( (max_mem_size / row_size_bytes) / rows_in_table , 1.0 ) × 100%
```
走的是 `ha_sample_init`（**页级 SYSTEM 采样**，不是行级均匀），采样率存进直方图供后续 GEE 修正。

### 8.2 Equi-height 分桶算法

**第 1 步：二分搜索"最大桶容量"**（`equi_height.cc:192`）

```cpp
ha_rows upper_bucket_values = 2 * total_values / (max_buckets - 1) + 1;   // 上界（有理论保证）
const int max_search_steps = 10;    // ★ 最多二分 10 次
while (upper_bucket_values > lower_bucket_values + 1 && search_step < max_search_steps) {
  ha_rows bucket_values = (upper + lower) / 2;
  if (FitsIntoBuckets(value_map, bucket_values, max_buckets)) upper = bucket_values;
  else lower = bucket_values;
}
```

`FitsIntoBuckets`（`:128`）贪心判定：按值有序扫描，当前桶加不下就开新桶；**但"单值桶"允许超限**（一个值本身频率就超过 max 也不拆开）。

**第 2 步：贪心装桶**（`:417`）——三个继续条件：
```cpp
if (next != value_map.end() &&
    distinct_values_remaining > empty_buckets_remaining &&        // 给后面每个值留一个桶
    bucket_values + next->second <= bucket_max_values) continue;  // 等高约束
```

每个桶存 `[lower, upper]`（闭区间）+ `cumulative_frequency`（**累积**频率）+ `num_distinct`。

**桶内 NDV 的 GEE 估计**（`:220-250`）：

```
GEE = sqrt(1/s) × u + (d - u)
```

`s`=采样率，`d`=样本内 distinct 数，`u`=样本内只出现一次的值数。直觉：高频值（d-u）不放大，低频值（u）按 `sqrt(1/s)` 放大——**故意比 `1/s` 保守**（低估 NDV → 高估选择率 → 偏安全）。

> ⚠️ GEE 为**均匀随机采样**设计，而 MySQL 用**页级采样**，源码注释承认"最坏情况下会低估 NDV 近 1/s 倍"。

### 8.3 选择率估算

**入口 `get_selectivity`（`histogram.cc:2001`）——下限钳到 0.001**：

```cpp
if (get_raw_selectivity(items, item_count, op, selectivity)) return true;
const double minimum_selectivity = 0.001;
*selectivity = std::max(*selectivity, minimum_selectivity);
```

注释列了 4 个理由：采样漏值、桶间空洞、桶内启发式、直方图过期；选 0.001 是因为默认采样 < 1000 页时漏掉一个 selectivity=0.001 的值概率约 1/e。

`get_raw_selectivity`（`:2047`）做规范化：要求一侧是 `Item::FIELD_ITEM`、另一侧是常量；**常量在左边时翻转算子**（`>` ↔ `<`）并递归。

**Singleton（精确查表）**：`lower_bound` 二分定位，命中则 `freq = cum[i] - cum[i-1]`；**没命中 → 0.0**（精确值不在表中就是 0）。`< v` 取前一桶累积频率；`> v` 用 `non_null_fraction - 前一桶累积频率`。

**Equi-height（桶内线性插值）**：

```
sel(col = v) = freq(bucket(v)) / num_distinct(bucket(v))        # 桶内均分
sel(col < v) = cum[bucket_before] + freq(bucket(v)) × distance(v)
其中 distance(v) = (v - lower) / (upper - lower) ∈ [0,1]
```

- 之前所有桶的贡献**精确**（用累积频率）
- 当前桶**按 v 在 `[a,b]` 内的相对位置线性切分**（桶内均匀分布假设）
- v 落在桶间隙（`< lower`）→ 当前桶贡献 0

> **有直方图 vs 无直方图的本质区别**：把"均匀假设"从**整列**细化到**桶内**，从而能捕捉偏斜分布。

---

## 九、条件过滤 condition filtering

### 9.1 calculate_condition_filter（`sql_planner.cc:1243`）

**短路条件**：必须 `condition_fanout_filter=on`（**默认 on**，`sys_vars.cc:196-211`）且满足 2a~2g 之一（有 join buffer / 不是最后一张表 / 子查询 / semijoin / ORDER BY|GROUP BY + LIMIT / EXPLAIN / 未开 BIG_SELECTS）。行数 < 1 或该表无谓词 → 直接 1.0。

**三步，信息源优先级递减**（`sql_planner.h:211-222`）：

```
1) range optimizer 的行数估计（quick_rows / records）   ← 最准
2) 索引统计 rec_per_key
3) Guesstimates                                          ← 最差
```

**第 1 步**：排除访问方法已用掉的列（ref 用 `bound_keyparts`、range 用 `get_fields_used`）——已反映在记录估计里，不重复计入。

**第 2 步**（`:1411`）：优先采用 range optimizer 的估计
```cpp
const float selectivity = table->quick_rows[keyno] / (float)tab->records();
filter *= std::min(selectivity, 1.0f);
```

**第 3 步**（`:1460`）：剩余谓词交给 `get_filtering_effect`。最后 `filter = max(filter, 1.0f / tab->records())`（**至少 1 行匹配**）。

### 9.2 各算子的默认系数（已核实，`item.h:98-115`）

| 常量 | 值 | 含义 |
|---|---|---|
| `COND_FILTER_ALLPASS` | **1.0** | 无过滤（未知谓词/已读表） |
| `COND_FILTER_EQUALITY` | **0.1** | `col1 = col2` |
| `COND_FILTER_INEQUALITY` | **0.3333** | `col1 > col2` |
| `COND_FILTER_BETWEEN` | **0.1111** | `BETWEEN` |

**核心折算公式** `Item_field::get_cond_filter_default_probability`（`item.cc:7948`）：

```cpp
switch (field->real_type()) {
  case MYSQL_TYPE_ENUM: max_distinct_values = min(typelib->count, max_distinct_values); break;
  case MYSQL_TYPE_BIT:  max_distinct_values = min(pow(2.0, bits), max_distinct_values); break;
}
return max(1.0f / max_distinct_values, default_filter);
```

即 **`filter = max(1/records, default_filter)`**：

- 对**小表**，`1/records` 会超过 0.1（5 行的表 → 0.2），从而**自动放宽**启发式
- `records >= 10` 时 `=` 恒为 0.1；`records < 10` 时为 `1/records`
- ENUM 按值域大小、BIT(N) 按 `2^N` 收紧上界

**各算子对照表**：

| Item 类型 | 默认系数 | 先用直方图? | 位置 |
|---|---|---|---|
| `Item_func_eq` (`=`) | 0.1 | 是 | `item_cmpfunc.cc:7379` |
| `Item_func_ne` (`<>`) | 0.9 | 是 | `:2465` |
| `>/</>=/<=` | 0.3333 | 是 | `:2503-2630` |
| `IS NULL` | 0.1 | 是 | `:6036` |
| `IS NOT NULL` | 0.9 | 是 | `:6220` |
| 裸列 `WHERE col` | 0.9 | 否 | `item.cc:7936` |
| `IN` | `min(值个数 × 单值率, 0.5)` | 是 | `:4856` |
| `Item_equal`（多等值） | `rec_per_key[0]/records` 或 0.1 | 否 | `:6949` |
| `BETWEEN` | 0.1111 | — | `item.h:108` |

**AND/OR 组合**：

```cpp
// AND：独立事件相乘
filter *= item->get_filtering_effect(...);                    // item_cmpfunc.cc:5921
// OR：P(A)+P(B)-P(A)P(B)
filter = filter + cur_filter - (filter * cur_filter);          // :5976
```

**IN 列表的硬上限**（`:4860`）：`in_max_filter = 0.5`——**IN 列表最多认为过滤掉一半**，用加法近似（忽略重复值）。

### 9.3 hypergraph 的 estimate_selectivity

8.0.39 的 `estimate_selectivity` 在 **`sql/join_optimizer/estimate_selectivity.cc`**（旧优化器侧已无此函数）：

```cpp
// :50-135
// 1) 索引优先：sel = rec_per_key[0] / stats.records，多个候选取【最大】（最不 selective）
selectivity = std::max(selectivity, field_selectivity);
// 2) 唯一单列索引：硬上限 1/records
if (single_row) *selectivity_cap = std::min(*selectivity_cap, 1.0 / records);
// 3) 回退直方图：sel = non_null_fraction / num_distinct_values
// 4) 再回退：get_filtering_effect(..., rows_in_table = 1000.0)   ← ★ 硬编码 1000
```

**注意第 4 步硬编码 `rows_in_table = 1000.0`**，于是 `get_cond_filter_default_probability(1000, 0.1) = max(0.001, 0.1) = 0.1`。

多个候选取**最大值**（最不 selective）的原因：宁可高估行数——高估导致保守但安全的计划，低估会导致 nested loop 灾难。

---

## 十、无统计时的兜底估算

| 兜底 | 值 | 位置 | 说明 |
|---|---|---|---|
| `guess_rec_per_key` | 首列 **1%**，整键 **10**（唯一键 1） | `opt_statistics.cc:61-120` | 无 rec_per_key 时的插值 |
| `MATCHING_ROWS_IN_OTHER_TABLE` | **10** | `sql_planner.cc:91` | `distinct_keys_est = records / 10` |
| `handler::records_in_range` 默认 | **10 行** | `handler.h:5530` | 引擎不实现 range 估行 |
| `rows_in_range_arbitrary_ret_val` | **10** | `btr0cur.cc:5188` | index dive 重试 4 次仍失败 |
| `btr_estimate_..._on_level` 兜底 | **10** | `btr0cur.cc:5173` | 一页都没读成功 |
| `N_PAGES_READ_LIMIT` | **10 页** | `btr0cur.cc:5087` | index dive 最多扫 10 页 |
| range 估行上限 | `table_n_rows / 2` | `btr0cur.cc:5421` | 非精确估计最多占全表一半 |
| 树高>1 低估补偿 | **×2** | `btr0cur.cc:5407` | 算法系统性低估 |
| `COND_FILTER_EQUALITY` | **0.1** | `item.h:104` | `=` 无统计 |
| `in_max_filter` | **0.5** | `item_cmpfunc.cc:4860` | IN 过滤上限 |
| `filter` 下界 | `1 / records` | `sql_planner.cc:1481` | 至少 1 行 |
| 直方图 selectivity 下界 | **0.001** | `histogram.cc:2042` | 对冲采样漏值 |
| in_mem 阈值 / 缓冲兜底 | **20%** / **100MB** | `handler.cc:6025/6018` | page_read_cost 启发式 |
| 空表 rec_per_key | **1.0** | `ha_innodb.cc:17044` | |

---


## 参考

**论文**
- **Selinger et al.《Access Path Selection in a Relational DBMS》(SIGMOD 1979)** —— 代价公式、选择性估算
- **Poosala et al.《Improved Histograms for Selectivity Estimation of Range Predicates》(SIGMOD 1996)** —— 等深直方图
- **GEE（Guaranteed Error Estimator）** —— 桶内 NDV 外推 `sqrt(1/s)·u + (d-u)`，见 `equi_height.cc:220` 注释

**官方文档**
- *MySQL 8.0 Reference Manual → The Optimizer Cost Model*（`mysql.server_cost` / `engine_cost`）
- *MySQL 8.0 Reference Manual → Histogram Statistics*
- *MySQL 8.0 Reference Manual → Configuring Optimizer Statistics for InnoDB*（persistent / transient）

**内核月报**
- 2022/10《统计信息采集》
- **2020/12《统计信息的现状和发展》** —— InnoDB 统计信息管理框架缺陷（默认采样 20 页、时效性、一致性风险）

