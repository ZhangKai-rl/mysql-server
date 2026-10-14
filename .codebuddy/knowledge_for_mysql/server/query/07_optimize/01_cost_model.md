# 01 代价模型与统计信息

> 代价模型（物理优化的判据）+ 统计信息（代价模型的输入）。两者是 ②③ 物理优化的前提。
>
> ★ 本篇讲的是**经典优化器**的代价模型（`Cost_model_table` / `Cost_model_server`，可从 `mysql.server_cost` / `mysql.engine_cost` 配置）。hypergraph 优化器用的是**另一套**（编译期 `constexpr` 常量，全部为 0.1），见 [18 篇第四章](physical/18_hypergraph_advanced.md)——两者的量纲不可直接比较。

## 目录

- [零、总览](#零总览)
- [设计思想与理论基础](#设计思想与理论基础)
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

## 设计思想与理论基础

### 设计思想与权衡

#### 1. 为什么是"半经验公式"而不是纯理论模型

理论（Selinger 式）只提供公式的**结构**（IO + CPU、选择率连乘），具体参数几乎全部是**基准测试 + 线上回归反推**。源码里有大量直白的自我承认：

| 位置 | 原文 |
|---|---|
| `handler::estimate_in_memory_buffer` | "if the size of the table/index is less than 20 percent (**pick any number**) of the memory buffer" |
| `guess_rec_per_key` | "a = records matched by first key part (**1% of all records?**)" |
| `find_worst_seeks` | "This is mostly a **crutch** to mitigate that we don't estimate the cache effects of ref accesses properly" |
| `get_sort_and_sweep_cost` | 把 `key_compare_cost` 砍掉 10 倍："Until this constant is adjusted we introduce a constant that is **more realistic**" |
| InnoDB `scan_time` | "we pretend that a sequential read takes the same time as a random disk read… which **would be physically realistic**" |

**为什么接受半经验**：存储栈（buffer pool / OS cache / SSD / NVMe）不可观测，且优化器必须在**编译时**出结果、不能做任何探测。所以设计取向是——**先保证"计划排序正确"，再谈绝对精度**。`pick any number`、`crutch`、`more realistic` 这些词反复出现，就是这个取向的自白。

#### 2. 代价常数是怎么标定的

思路是**先定一个锚，再用相对 benchmark 比值推其余项**：

| 常数 | 默认值 | 标定依据（注释原文） |
|---|---|---|
| `ROW_EVALUATE_COST` | **0.1** | 锚（"评估一行"），无 benchmark 注释 |
| `KEY_COMPARE_COST` | **0.05** | 无注释（量纲直觉 = 半行评估） |
| `MEMORY_TEMPTABLE_CREATE_COST` | **1.0** | "by benchmark found to be as costly as **writing 10 rows**" |
| `MEMORY_TEMPTABLE_ROW_COST` | **0.1** | "equivalent to evaluating a row in the join engine" |
| `DISK_TEMPTABLE_CREATE_COST` | **20.0** | "Creating a MyISAM table is **20 times slower** than creating a Memory table" |
| `DISK_TEMPTABLE_ROW_COST` | **0.5** | "we do not have benchmarks for very large tables, so setting this factor **conservatively** to be 5 times slower" |
| `MEMORY_BLOCK_READ_COST` | **0.25** | 无任何标定注释（是 fiat） |
| `IO_BLOCK_READ_COST` | **1.0** | **整个模型的单位锚** |

编译期硬编码（不可配）的 seek 常数则用**自洽方程**标定：平均一次 seek（跳过 `BLOCKS_IN_AVG_SEEK = 128` 块）总代价 = 1.0，解出 `DISK_SEEK_BASE_COST = 0.9`、`DISK_SEEK_PROP_COST = 0.1/128`（且带 `@todo` 承认未核实）。

> 注意：**默认值需与 `mysql_system_tables.sql` 的 `default_value` 列两处同步**，源码顶部注释专门提醒了这一点。

#### 3. 为什么代价单位是"抽象代价单位"而不是毫秒

源码明确：

> **Our cost units are "random disk seeks"**…

- **好处**：硬件无关（同一套常数在 HDD / SSD / NVMe 上语义一致）；常数变动时计划的**相对排序**保持稳定，回归测试好做
- **代价**：① 回答不了"这条 SQL 跑多久"；② 在 SSD 上 IO 与 CPU 的真实比例完全失真（这正是 `engine_cost` 可配的动机）；③ `worst_seeks` 这类人造上限本质就是**单位失真的补丁**

#### 4. index dive 与统计估算为什么并存

**dive 是按 range 数线性增长的**（每个等值 range 一次 B 树双向下潜），**统计是 O(1)** 查 `rec_per_key`：

- 纯 dive：`IN (...)` 有上万个值时，优化期开销可能超过执行期
- 纯统计：等值谓词是最常见的点查，用"全局平均 `rec_per_key`"估每个值，在倾斜列上错得离谱

所以做成**按数量切换**（`eq_range_index_dive_limit = 200`），并允许 session + hint 覆盖、`0` 强制 dive。

> ⚠️ **200 在源码中没有任何选择理由的注释**——应视为"经验阈值，无官方推导"，不要臆造理由。

**超阈值后的四类偏差**：① 每个 range 都用同一个全局平均 `rec_per_key`（uniformity 假设，倾斜列上高频值被低估）；② **`x IS NULL` 被显式排除**（注释：该值数量"likely to be very different"，所以永远走 dive）；③ `rec_per_key` 本身是采样外推的，误差复合；④ 无 `rec_per_key` 时退化为 `rows = 1`（FORCE INDEX 分支），会极端低估。

#### 5. 为什么 8.0 引入直方图

**本质**：把"均匀分布假设"从**整列**细化到**桶内**——选择率从 `1/NDV` 变成 `freq(bucket)/num_distinct(bucket)`。

> **关键理解**："均分"这一步**没有消失，只是作用域缩小了**。所以直方图能捕捉偏斜，但不能消除误差。

**为什么不默认开启、也不自动更新**：

1. **增量更新根本没实现**——源码原话："As of now, this means 'when the histogram was created' (**incremental updates are not supported**)"
2. **构建是重操作**：全表扫描 + 内存有界采样，采样率由 `histogram_generation_max_mem_size`（默认 20MB）反推
3. **能靠索引精确得到的就不建**："If it is [covered by a single-part unique index], we don't want to create histogram statistics"
4. **过期风险用下限对冲而非自动刷新**：`get_selectivity()` 返回至少 **0.001**，注释详述了理由（采样 1000 页时漏掉选择率 0.001 值的概率约 1/e）与代价（大表上 0.1% 仍是很多行）

**桶数默认 100**："A value of 100 is chosen because **the gain in accuracy above this point seems to be generally low**"（上限 1024）。

**最值得整段引用的设计理由**是 equi-height 用 GEE 估桶内 NDV 的那段注释——它把"为什么这么估、为什么不改、未来怎么办"全写了：

- 知道**页级采样破坏 GEE 假设**（GEE 是为均匀随机采样设计的），但**接受**，因为"we would rather **underestimate** than overestimate the number of distinct values"
- 明确**拒绝** HyperLogLog / Count-Min，理由是"would require updating a sketch **on every table update**"——与"不自动更新"是同一个决策的两面

#### 6. condition filtering 的设计动机

它估的是 **"post read filtering"**：访问方法只覆盖部分谓词，其余谓词在读出行之后才评估，这部分过滤**不被任何访问方法反映**，必须单独估。

**为什么必须估**：不估就等价于 `filter = 1.0`，而 `filter` 是**逐表相乘进 fanout** 的——N 表 join 时误差**连乘放大**。

**"计算本身有代价，所以只在有意义时算"**（源码注释原话）。启用条件恰好暴露了设计考量：

- `2b` 不是 join order 的最后一张表（"filtering only reduces the number of rows sent to the next step… therefore has **no effect** on the last table"）
- `2c` 在子查询中（需要估算可能物化的子查询行数）
- `2d` 在含 semijoin nest 的查询块（去重策略的代价依赖输出大小）
- **单表查询直接忽略**："since single table optimization performance is so important"

**它基于两个假设**：

1. **谓词间独立**：AND 连乘（`P(A and B) = P(A) * P(B)`）、OR 容斥
2. **无统计时按算子类型给固定系数**：`COND_FILTER_EQUALITY = 0.1`、`INEQUALITY = 0.3333`、`BETWEEN = 0.1111`

并有**小表放宽**（`filter = max(1/records, default_filter)`）与**下界保护**（`filter >= 1/records`、fanout 下限 0.05）。

> **fanout 下限 0.05 的 TODO 是极强的设计证据**：它本身会造成"不同 join order 估出不同结果集大小"的不自洽，但因为"去掉后 DBT-3 出现不良效果"就保留了——**纯经验回归驱动**。

#### 7. InnoDB 统计为什么用随机采样

默认值：**persistent 20 页、transient 8 页**；表级 `STATS_SAMPLE_PAGES` 范围 [1, 65535]（注释："65535 pages, 16kb each means to sample 1GB, which is impractical"）。

**为什么不全表扫描**（源码给了两条理由）：

1. **锁竞争**：采样路径需持有索引 `SX_LOCK`；扫超过 **1e6 页**时，宁可走无 `SX_LOCK` 的全扫
2. **规模不经济**：`n_sample_pages × n_uniq` 超过总叶页数时，采样反而比全扫**更慢且更差** → 自动改走全扫

⇒ **采样不是教条，是成本权衡。**

**持久 vs 瞬态**：持久（20 页、分层随机采样、落盘）保证重启不丢 + 实例间计划稳定（避免主备漂移）；瞬态（8 页、仅内存）为无持久统计的场景提供廉价兜底。

**自动更新按"变更行数比例"触发**：持久 > `n_rows/10`、瞬态 > `16 + n_rows/16`（`+16` 是避免"计数器表"被反复触发，注释明说）。因为统计的**相对误差取决于"变更量/总量"**，按比例才是与精度相关的正确量纲。另有 **10 秒 `MIN_RECALC_INTERVAL`** 节流。

⇒ **"统计滞后"是被设计接受的**：10% 阈值 + 10 秒节流 + 后台线程，三层都是"宁可滞后，不可影响前台"。

### 失效场景与已知短板（源码自认）

#### 列独立性 / 零相关性假设

最坦率的一处在 `EstimateFieldSelectivity()` 的 doc comment：

> Assumes **equal distribution and zero correlation** between the two fields… If there are multiple ones, we choose the one with the **largest** selectivity (least selective)…
> - Databases generally tend to **underestimate** join cardinality (due to assuming uncorrelated relations); if we're wrong, it would better be towards **overestimation**…
> - **Overestimating** the number of rows generally leads to safer choices that are a little slower for few rows (e.g., hash join). **Underestimating**, however, leads to choices that can be **catastrophic** for many rows (e.g., nested loop against table scans).

**"没有相关性检测，只能靠取最大选择性来补偿"** —— 这是对"社区版无列相关性统计"最直接的源码回答。全库**没有任何**列相关性/多列联合分布的统计结构（直方图是**单列**的）。

**后果**：列相关时（如 `country='CN' AND city='Beijing'`）AND 连乘会**严重低估行数** ⇒ 优化器以为 index merge / nested loop 很便宜 ⇒ 实际行数爆炸。这正是"错选 index merge / 错选 join order"的机理。

#### 源码自认的短板清单

| 位置 | 原文 |
|---|---|
| `find_worst_seeks` | "This is mostly a **crutch**…" |
| `EstimateCostForRefAccess` | "This is still a **very primitive, and rather odd**, cost model." |
| `disk_seek_prop_cost` | `@todo Check that the BLOCKS_IN_AV_SEEK is correct…` |
| fanout 下限 | `TODO: Should evaluate whether this restriction makes sense… some unwanted effects on DBT-3 was observed when removing it` |
| equi-height GEE | 承认采样率 1% 时"the estimate could be off by a **factor 10** about 1/3 of the time" |
| Singleton NDV | `TODO… If the histogram is based on sampling, then this estimate is potentially off by a factor 1/sampling_rate` |
| ROR 选择率 | `FIXME: …the estimation is **probably wrong**` |

#### 统一哲学

> **所有不确定处的默认取向都是"宁可高估行数"。**

直方图 0.001 下限、GEE 宁可低估 NDV（→ 高估选择率）、取最大选择性、`btr_estimate` 的 ×2 补偿与 `n_rows/2` 截断——全都是同一套。

### 理论溯源

- **Selinger《Access Path Selection in a Relational DBMS》(SIGMOD 1979)**：代价 = IO + CPU、选择率连乘，是这套公式结构的来源
- **直方图**：equi-height + **GEE（Guaranteed Error Estimator）** 估桶内 NDV，来自论文；源码注释给出了 GEE 公式与误差界的推导
- **InnoDB 采样外推**：基于"层间不同值比例相同"的假设外推到叶层

### 他库对比与演进动机

| 机制 | 引入版本 | 为什么引入 |
|---|---|---|
| 持久化统计 | 5.6 | 解决重启/主备计划漂移 |
| **代价常数表化**（`mysql.server_cost` / `engine_cost`） | **5.7**（非 8.0） | 使 DBA 能不重编译地适配 SSD / 大内存 / 特定引擎；注释明说理想是"不同存储设备各一套常数"，但当前 `MAX_STORAGE_CLASSES = 1`（因为"we… does not have a way to know which storage device a given table is stored on"） |
| condition filtering | 5.7 | 解决多表 join 行数估算的连乘放大 |
| **直方图** | **8.0** | 第一个"按值分布"的统计结构，打破均匀分布假设 |
| 直方图接入 condition filtering | 8.0 | "有直方图用直方图，没有就退回启发式常数" |

** hypergraph 侧去掉了 `worst_seeks` 上限**——旧上限是"防止全表扫被过度选中"，那个前提建立在**只有 nested-loop** 的世界里；有了 hash join 之后全表扫不再天然危险，于是这个补丁被撤销。

⇒ **代价模型的每一代人造常数，都绑定着当时的执行器能力。**

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

它的本体（最基础的代价公式，值得单独看）：

```cpp
// handler.cc:6055
Cost_estimate handler::table_scan_cost() {
  /*
    This function returns a Cost_estimate object. The function should be
    implemented in a way that allows the compiler to use "return value
    optimization" to avoid creating the temporary object for the return value
    and use of the copy constructor.
  */
  const double io_cost = scan_time() * table->cost_model()->page_read_cost(1.0);
  Cost_estimate cost;
  cost.add_io(io_cost);
  return cost;
}

// scan_time() 的默认实现（handler.h:4979，虽已标记 deprecated 但仍是默认路径）
virtual double scan_time() {
  return ulonglong2double(stats.data_file_length) / IO_SIZE + 2;
}
```

★ 于是**全表扫描的 io 代价 = `scan_time()` × `page_read_cost(1.0)`**，两项各管一段：

- **`scan_time()` = `data_file_length / IO_SIZE + 2`**——即"页数 + 2"。★ 那个 **`+2` 是常数余量**：即使空表也不为 0，避免"全扫代价为 0"在与其它 plan 比较时产生退化（代价全 0 会让 plan 选择失去区分度）。
- **`page_read_cost(1.0)`**——"读一页的加权代价"，已含 buffer 池命中率的折算（见 1.2 节 `buffer_block_read_cost` / `io_block_read_cost` 的加权）。所以引擎层给的是"页数"，命中率折算发生在 cost_model 这一层。

两点设计约定：

1. ★ **io 与 cpu 严格分离**：`table_scan_cost` 只 `add_io`，**完全不写 cpu**。因为引擎只知道"我要读多少页"，不知道 server 层要对每行做多少次表达式求值——后者由上层按行数乘 `ROW_EVALUATE_COST` 追加。这就是"引擎提供 io、server 补 cpu"的分工，也是 `Cost_estimate` 要分成多个维度而非单一数字的根本原因。
2. 源码注释特意要求**写成允许 RVO 的形式**（先算局部量再 `Cost_estimate cost; ...; return cost;`），避免返回临时对象触发拷贝构造——代价计算在优化器里被调用极频繁。

（`add_io` 那侧：`assert(!is_max_cost())` 防的是在 `DBL_MAX`（`set_max_cost()` 表达的"不可行"哨兵）上继续累加——`DBL_MAX + x` 仍是 `DBL_MAX`，累加无意义，见上文第 3 点。）

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

#### ★ 完整代价常量表（8.0.39 全部 8 个）

> 源码坐标：`[C1] sql/opt_costconstants.h/.cc`、`[C2] sql/opt_costmodel.h/.cc`、`[C12] sql/opt_costconstantcache.cc`

| 常量名 | 所在表 | 默认值 | 含义 / 使用点 |
|---|---|---|---|
| `row_evaluate_cost` | `mysql.server_cost` | **0.1** | 对一行求值 WHERE 的 CPU 代价 |
| `key_compare_cost` | `mysql.server_cost` | **0.05** | 一次键值比较（索引比较、range 边界比较） |
| `memory_temptable_create_cost` | `mysql.server_cost` | **1.0** | 创建内存内部临时表（注释：约等于写 10 行） |
| `memory_temptable_row_cost` | `mysql.server_cost` | **0.1** | 内存临时表读写一行 |
| `disk_temptable_create_cost` | `mysql.server_cost` | **20.0** | 创建磁盘临时表（注释：比 Memory 慢 20 倍） |
| `disk_temptable_row_cost` | `mysql.server_cost` | **0.5** | 磁盘临时表读写一行（保守取 5 倍慢） |
| `memory_block_read_cost` | `mysql.engine_cost` | **0.25** | 从缓冲池随机读一块 |
| `io_block_read_cost` | `mysql.engine_cost` | **1.0** | 从磁盘随机读一块（也是 `disk_seek_base_cost()` 的基准） |

```cpp
// [C1] sql/opt_costconstants.cc —— 默认值唯一来源
const double Server_cost_constants::ROW_EVALUATE_COST = 0.1;
const double Server_cost_constants::KEY_COMPARE_COST = 0.05;
const double Server_cost_constants::MEMORY_TEMPTABLE_CREATE_COST = 1.0;
const double Server_cost_constants::MEMORY_TEMPTABLE_ROW_COST = 0.1;
const double Server_cost_constants::DISK_TEMPTABLE_CREATE_COST = 20.0;
const double Server_cost_constants::DISK_TEMPTABLE_ROW_COST = 0.5;

const double SE_cost_constants::MEMORY_BLOCK_READ_COST = 0.25;
const double SE_cost_constants::IO_BLOCK_READ_COST = 1.0;

// 常量写入口：名字大小写不敏感，值必须 > 0
cost_constant_error Server_cost_constants::set(const LEX_CSTRING &name,
                                               double value) {
  if (name.str == nullptr || name.length == 0) return UNKNOWN_COST_NAME;
  if (value <= 0.0) return INVALID_COST_VALUE;      // ★ 代价常量不允许 0 / 负值
  if (my_strcasecmp(&my_charset_utf8mb3_general_ci, "ROW_EVALUATE_COST",
                    name.str) == 0) {
    m_row_evaluate_cost = value;
    return COST_CONSTANT_OK;
  }
  return UNKNOWN_COST_NAME;
}
```

★ 引擎常量多一层"默认值 / 非默认值"语义——这是 `engine_cost` 表能同时表达 `engine_name='default'` 与具体引擎名的关键：

```cpp
// [C1] —— 非默认值无条件覆盖；新默认值只在"当前仍是默认值"时才生效
void SE_cost_constants::update_cost_value(double *cost_constant,
                                          bool *cost_constant_is_default,
                                          double new_value,
                                          bool new_value_is_default) {
  if (!new_value_is_default) {          // 用户显式配置 → 强制生效
    *cost_constant = new_value;
    *cost_constant_is_default = false;
  } else {                              // 只是改 default 行 → 已被覆盖则不动
    if (*cost_constant_is_default) *cost_constant = new_value;
  }
}
```

系统表覆盖路径（`[C12]`，读表必须**自建 THD**，因为当前 THD 可能已开关过表）：

```cpp
  Table_ref tables[2] = {Table_ref("mysql", "server_cost", TL_READ),
                         Table_ref("mysql", "engine_cost", TL_READ)};
  if (!open_and_lock_tables(thd, tables, MYSQL_LOCK_IGNORE_TIMEOUT)) {
    read_server_cost_constants(thd, tables[0].table, cost_constants);
    read_engine_cost_constants(thd, tables[1].table, cost_constants);
  }
// server_cost 列序：cost_name, cost_value, last_update, comment
//   → 只有 cost_value 非 NULL 的行才调用 update_server_cost_constant()
// engine_cost 列序：engine_name, device_type, cost_name, cost_value, last_update, comment
```

**三个要点**：

1. 常量是**引用计数缓存**对象，`FLUSH OPTIMIZER_COSTS` 只是换版本——正在跑的查询仍持有旧快照。
2. ★ `MAX_STORAGE_CLASSES = 1`（注释明说"当前版本无法得知表在哪种设备上"）——**每引擎只有一组常量**，`engine_cost.device_type` 实际只能填 0。
3. ★ `io_block_read_cost=1.0` 是**整个模型的标尺**：`disk_seek_base_cost() = 0.9 × io_block_read_cost`。所以调小它会同时缩小所有扫描代价，**无法只压 IO 不动 seek**。

```cpp
// [C2] —— 顺序 seek 的两个硬编码系数（不是可配置常量）
constexpr const double DISK_SEEK_BASE_COST{0.9};        // 相当于半圈旋转
constexpr const int BLOCKS_IN_AVG_SEEK{128};
constexpr const double DISK_SEEK_PROP_COST{0.1 / BLOCKS_IN_AVG_SEEK};
```

#### 三层结构与 `Cost_estimate`

```
   Cost_model_constants            ← 常量集合（server + 每引擎一组，引用计数缓存）
        ├── Server_cost_constants  （server 层，全局唯一）
        └── SE_cost_constants × MAX_STORAGE_CLASSES(=1)
                     │
   ┌─────────────────┴───────────────────────────────┐
Cost_model_server（每查询）              Cost_model_table（每 TABLE）
   row_evaluate_cost(rows)                 io_block_read_cost(blocks)
   key_compare_cost(keys)                  buffer_block_read_cost(blocks)
   tmptable_create_cost(type)              page_read_cost(pages)
   tmptable_readwrite_cost(...)            disk_seek_cost(seek_blocks)
```

```cpp
// [C5] sql/handler.h —— 比较口径只含三项，mem_cost 被刻意排除
class Cost_estimate {
  double io_cost, cpu_cost, import_cost;
  double mem_cost;      ///< 内存占用（字节），不参与比较
 public:
  double total_cost() const { return io_cost + cpu_cost + import_cost; }
  void multiply(double m) {
    io_cost *= m; cpu_cost *= m; import_cost *= m;
    /* Don't multiply mem_cost */    // 内存不随行数线性放大（如一个 hash table）
  }
  Cost_estimate &operator+=(const Cost_estimate &other) {
    io_cost += other.io_cost;     cpu_cost += other.cpu_cost;
    import_cost += other.import_cost;
    mem_cost += other.mem_cost;   // 累加，只是不进 total_cost()
    return *this;
  }
};
```

⇒ 两张代价相同的计划**只有内存差异时不会互相淘汰**（`mem_cost` 只用于 `EXPLAIN FORMAT=JSON` 与临时表选择）。

8.0.39 的三个扫描代价入口都在 `handler` 上（**不是** `Cost_model_table`——这点与 5.7 不同）：

```cpp
// [C19] sql/handler.cc
Cost_estimate handler::table_scan_cost() {
  const double io_cost = scan_time() * table->cost_model()->page_read_cost(1.0);
  Cost_estimate cost;
  cost.add_io(io_cost);              // ★ 只记 io，不记 cpu
  return cost;
}
Cost_estimate handler::index_scan_cost(uint index, double ranges, double rows) {
  const double io_cost = index_only_read_time(index, rows) *
                         table->cost_model()->page_read_cost_index(index, 1.0);
  Cost_estimate cost; cost.add_io(io_cost); return cost;
}
```

⇒ 三个函数都只 `add_io()`：CPU 侧的 `row_evaluate_cost` 是**调用方**（连接顺序搜索）另外累加的，所以单看 `table_scan_cost()` 会低估。

#### 缓冲区命中率插值与 InnoDB 的注入点

```cpp
// [C2] sql/opt_costmodel.cc —— 纯线性插值
double Cost_model_table::page_read_cost(double pages) const {
  const double in_mem = m_table->file->table_in_memory_estimate();  // [0,1]
  const double pages_in_mem = pages * in_mem;
  const double pages_on_disk = pages - pages_in_mem;
  return buffer_block_read_cost(pages_in_mem) + io_block_read_cost(pages_on_disk);
}
// page_read_cost_index() 完全同构，只是换成 index_in_memory_estimate(index)
```

⇒ `in_mem=1` 单价 0.25、`in_mem=0` 单价 1.0，**跨度正好 4 倍**——这就是"内存表 vs 磁盘表代价差 4 倍"的全部来源，没有二次项。

`in_mem` 的启发式（引擎没给真值时）：

```cpp
// [C19] —— 源码注释原文的 20%/100% 分段
double handler::estimate_in_memory_buffer(ulonglong table_index_size) const {
  longlong memory_buf_size = get_memory_buffer_size();   // InnoDB: srv_buf_pool_curr_size
  if (memory_buf_size <= 0) memory_buf_size = 100 * 1024 * 1024;  // 没报就假设 100MB
  const double table_index_in_memory_limit = 0.2;
  const double percent_of_mem = static_cast<double>(table_index_size) / memory_buf_size;
  if (percent_of_mem < table_index_in_memory_limit) return 1.0;   // < 20% → 全在内存
  else if (percent_of_mem > 1.0) return 0.0;                      // > buffer → 全不在
  else return 1.0 - (percent_of_mem - table_index_in_memory_limit) /
                        (1.0 - table_index_in_memory_limit);
}
```

InnoDB 通过 override `scan_time()` / `read_time()` 把统计塞进代价模型（**它不 override `table_scan_cost()`**）：

```cpp
// [C7] ha_innodb.cc
double ha_innobase::scan_time() {
  /* 注释原文：MySQL 过于偏向全表扫描，所以我们假装顺序读和随机读一样贵，
     即下面不除以 10（虽然物理上那样更真实）。 */
  if (m_prebuilt == nullptr)   // 派生表：还没开就先要统计
    return (ulonglong2double(stats.data_file_length) / IO_SIZE + 2);
  return ((double)m_prebuilt->table->stat_clustered_index_size);  // ★ 单位"页"
}

double ha_innobase::read_time(uint index, uint ranges, ha_rows rows) {
  if (index != table->s->primary_key)
    return (handler::read_time(index, ranges, rows));
  if (rows <= 2) return ((double)rows);
  double time_for_scan = scan_time();
  if ((total_rows = estimate_rows_upper_bound()) < rows) return (time_for_scan);
  return (ranges + (double)rows / (double)total_rows * time_for_scan);
}
longlong ha_innobase::get_memory_buffer_size() const { return srv_buf_pool_curr_size; }
```

⇒ `scan_time()` 返回页数的**无单位数字**，被 `table_scan_cost()` 直接当页数乘单价。所以 `innodb_stats_persistent_sample_pages` 导致的 `stat_clustered_index_size` 变化会**线性**传导到全表扫描代价。

⚠️ 一个乐观结论的来源：`estimate_in_memory_buffer()` 的分母是**整块 buffer pool**，表大小用 `data_file_length`（含碎片）。30GB buffer pool + 6GB 表 → `percent_of_mem=0.2` → `in_mem=1.0`（"整表常驻"）。
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

#### ★ `innodb_rec_per_key()` 完整代码

> 源码坐标：`[C7] storage/innobase/handler/ha_innodb.cc` · `[C6] sql/key.h`

```cpp
rec_per_key_t innodb_rec_per_key(const dict_index_t *index, ulint i,
                                 ha_rows records) {
  rec_per_key_t rec_per_key;
  uint64_t n_diff;

  ut_a(index->table->stat_initialized);
  ut_ad(i < dict_index_get_n_unique(index));
  ut_ad(!dict_index_is_spatial(index));

  if (records == 0) {
    /* "Records per key" 对空表无意义；返回 1.0 对优化器最方便 */
    return (1.0);
  }

  n_diff = index->stat_n_diff_key_vals[i];      // ← 采样外推得到的 NDV（见 5.x）

  if (n_diff == 0) {
    rec_per_key = static_cast<rec_per_key_t>(records);   // 退化为"全表一个值"
  } else if (srv_innodb_stats_method == SRV_STATS_NULLS_IGNORED) {
    uint64_t n_null;
    uint64_t n_non_null;
    n_non_null = index->stat_n_non_null_key_vals[i];
    /* 理论上 stat_n_non_null_key_vals[i] 应 ≤ records；统计值可能有偏差，
       这里保证 NULL 数不为负 */
    if (records < n_non_null)
      n_null = 0;
    else
      n_null = records - n_non_null;

    if (n_diff <= n_null) {
      /* NULL 值数量 ≥ 不同值数量 → 表里几乎全是 NULL */
      rec_per_key = 1.0;
    } else {
      // ★ nulls_ignored：把 NULL 行从分子分母里同时剔除
      rec_per_key =
          static_cast<rec_per_key_t>(records - n_null) / (n_diff - n_null);
    }
  } else {
    rec_per_key = static_cast<rec_per_key_t>(records) / n_diff;  // nulls_equal/unequal
  }

  if (rec_per_key < 1.0) {
    /* 小于 1.0 无意义，只可能是统计不精确导致 */
    rec_per_key = 1.0;
  }
  return (rec_per_key);
}
```

写入 `KEY` 的位置（`ha_innobase::info_low()` 内，**同时写浮点版和整型版**）：

```cpp
/* innodb_rec_per_key() 用 index->stat_n_diff_key_vals[] 与传入的
   index->table->stat_n_rows。两者都由 ANALYZE 或后台统计线程计算，
   可能来自不同时刻，这是可接受的。 */
const rec_per_key_t rec_per_key =
    innodb_rec_per_key(index, (ulint)j, index->table->stat_n_rows);
key->set_records_per_key(j, rec_per_key);        // ★ 浮点数组（优化器真正使用）

/* 下面是遗留代码，一旦确认浮点版 rec_per_key 工作正常就应连同本注释一起删掉 */
ulong rec_per_key_int =
    static_cast<ulong>(innodb_rec_per_key(index, (ulint)j, stats.records));
/* 由于 MySQL 似乎过于偏向全表扫描，我们假装索引选择性比实际好 2 倍 */
rec_per_key_int = rec_per_key_int / 2;
if (rec_per_key_int == 0) rec_per_key_int = 1;
key->rec_per_key[j] = rec_per_key_int;
```

```cpp
// [C6] sql/key.h —— 双数组：rec_per_key(ulong) 遗留 + rec_per_key_float(double) 新版
rec_per_key_t records_per_key(uint key_part_no) const {
  /* 引擎给了浮点版就用浮点版，否则用整型版 */
  if (rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN)
    return rec_per_key_float[key_part_no];
  return (rec_per_key[key_part_no] != 0)
             ? static_cast<rec_per_key_t>(rec_per_key[key_part_no])
             : REC_PER_KEY_UNKNOWN;
}
void set_records_per_key(uint key_part_no, rec_per_key_t rec_per_key_est) {
  assert(rec_per_key_est == REC_PER_KEY_UNKNOWN || rec_per_key_est >= 1.0);  // ★ 下界 1.0
  rec_per_key_float[key_part_no] = rec_per_key_est;
}
```

**三个防御性下界**（共同保证"等值条件永远不会估出 0 行"）：空表返回 1.0、`n_diff==0` 时返回 `records`、`rec_per_key < 1.0` 一律夹到 1.0。

⚠️ 两处常被误解的地方：

1. **`SRV_STATS_NULLS_IGNORED` 不是"忽略 NULL 列"**——它是**把 NULL 当成一个不存在的值域**，从样本空间整体移除。所以 `innodb_stats_method=nulls_ignored` 会让含大量 NULL 的列估值**显著变低**（更偏索引）。
2. **整型版与浮点版不同源**：整型版用 `stats.records`（MySQL 侧），浮点版用 `stat_n_rows`（InnoDB 侧）——这也是遗留路径估值会漂的原因之一。
3. 另注：`SHOW INDEX` 的 `Cardinality` 走的就是 `n_rows / innodb_rec_per_key(...)`（FTS/SPATIAL 索引硬编码为 `stat_n_rows`）。
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

### 5.5 REF01：NDV 外推公式（`dict0stats.cc:1604`）

源码里的算法头注释（`storage/innobase/dict/dict0stats.cc:53-115`）就是 REF01 的推导原型：

```cpp
/* Sampling algorithm description @{

The algorithm is controlled by one number - N_SAMPLE_PAGES(index),
let it be A, which is the number of leaf pages to analyze for a given index
for each n-prefix (if the index is on 3 columns, then 3*A leaf pages will be
analyzed).

Let the total number of leaf pages in the table be T.
Level 0 - leaf pages, level H - root.

For each n-prefix: start from the root level and full scan subsequent lower
levels until a level that contains at least A*10 distinct records is found.
Lets call this level LA.

After finding the appropriate level LA with >A*10 distinct records ...
divide it into groups of equal records and pick A such groups. ...
After selecting A records as described above, dive below them to find A leaf
pages and analyze them, finding the total number of distinct records. ...

Let the number of different key values found in each leaf page i be Pi (i=1..A).
Let N_DIFF_AVG_LEAF be (P1 + P2 + ... + PA) / A.
Let the number of different key values on level LA be N_DIFF_LA.
Let the total number of records on level LA be TOTAL_LA.
Let R be N_DIFF_LA / TOTAL_LA, we assume this ratio is the same on the
leaf level.
Let the number of leaf pages be N.
Then the total number of different key values on the leaf level is:
N * R * N_DIFF_AVG_LEAF.
See REF01 for the implementation.
@} */
```

公式落地只有 8 行，但**四个量（A / LA / N / Pi）各自的来源**才是坑，逐个下钻。

#### 5.5.1 落地：`dict_stats_index_set_n_diff()` 完整

```cpp
// storage/innobase/dict/dict0stats.cc:1604
/** Set dict_index_t::stat_n_diff_key_vals[] and stat_n_sample_sizes[].
@param[in]      n_diff_data     input data to use to derive the results
@param[in,out]  index           index whose stat_n_diff_key_vals[] to set */
static inline void dict_stats_index_set_n_diff(const n_diff_data_t *n_diff_data,
                                               dict_index_t *index) {
  for (ulint n_prefix = dict_index_get_n_unique(index); n_prefix >= 1;
       n_prefix--) {
    /* n_diff_all_analyzed_pages can be 0 here if
    all the leaf pages sampled contained only
    delete-marked records. In this case we should assign
    0 to index->stat_n_diff_key_vals[n_prefix - 1], which
    the formula below does. */

    const n_diff_data_t *data = &n_diff_data[n_prefix - 1];

    ut_ad(data->n_leaf_pages_to_analyze > 0);
    ut_ad(data->n_recs_on_level > 0);

    uint64_t n_ordinary_leaf_pages;   // ★ N：普通（非 BLOB 外部）叶页数

    if (data->level == 1) {
      /* If we know the number of records on level 1, then
      this number is the same as the number of pages on
      level 0 (leaf). */
      // ★ 采样层恰好是 level 1 ⇒ 该层记录数 == 叶页数，N 精确
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
      // ★ T = index->stat_n_leaf_pages（含 BLOB 外部页的总页数）
      //   D = data->n_leaf_pages_to_analyze（本次分析的普通叶页）
      //   E = data->n_external_pages_sum（这 D 页指向的外部页）
      //   ⇒ 用"普通:外部 = D:E"的比例，从 T 反推普通叶页数
      n_ordinary_leaf_pages =
          index->stat_n_leaf_pages * data->n_leaf_pages_to_analyze /
          (data->n_leaf_pages_to_analyze + data->n_external_pages_sum);
    }

    /* See REF01 for an explanation of the algorithm */
    // ★ NDV = N × R × N_DIFF_AVG_LEAF
    index->stat_n_diff_key_vals[n_prefix - 1] =
        n_ordinary_leaf_pages                                  // N
        * data->n_diff_on_level / data->n_recs_on_level        // R = N_DIFF_LA / TOTAL_LA
        * data->n_diff_all_analyzed_pages / data->n_leaf_pages_to_analyze;  // N_DIFF_AVG_LEAF

    index->stat_n_sample_sizes[n_prefix - 1] = data->n_leaf_pages_to_analyze;
    // ★ 落进 mysql.innodb_index_stats.sample_size 的就是这个 A
  }
}
```

即 **`NDV ≈ N × R × N_DIFF_AVG_LEAF`**：

| 项 | 含义 |
|---|---|
| `N` | 普通（非溢出）叶页数。若采样层是 level 1 则精确（= level 1 记录数）；否则 `T × D/(D+E)` 剔除 BLOB 外部页 |
| `R` | 该层 distinct 密度，**假设"非叶层密度 = 叶层密度"** |
| `N_DIFF_AVG_LEAF` | 采样页平均 distinct 值数 `(P1+...+PA)/A` |

#### 5.5.2 采样层 LA 怎么定，A 个叶页怎么选（都**不是**"取前 A 个"）

`dict_stats_analyze_index_low()`（`dict0stats.cc:1668`）关键段——**先全扫到底层定 LA，再随机下潜**：

```cpp
static bool dict_stats_analyze_index_low(uint64_t &n_sample_pages,
                                         dict_index_t *index) {
  ...
  const uint64_t n_diff_required = n_sample_pages * 10;   // ★ A*10：LA 的"够格"判据
  ...
  root_level = btr_height_get(index, &mtr);
  n_uniq = dict_index_get_n_unique(index);

  /* If the tree has just one level (and one page) or if the user
  has requested to sample too many pages then do full scan. */
  // ★ 特例：树只有 1 层，或 A*列数 > min(叶页数, 1e6) ⇒ 直接全扫叶层，
  //   此时 NDV 精确、sample_size = 叶页数
  if (root_level == 0 || n_sample_pages * n_uniq >
                            std::min<ulint>(index->stat_n_leaf_pages, 1e6)) {
    ...
    (void)dict_stats_analyze_index_level(
        index, 0 /* leaf level */, index->stat_n_diff_key_vals, &total_recs,
        &total_pages, nullptr /* boundaries not needed */, wait_start_time,
        &mtr);
    for (ulint i = 0; i < n_uniq; i++) {
      index->stat_n_sample_sizes[i] = total_pages;
    }
    ...
    return true;
  }
  ...
  /* total_recs is also used to estimate the number of pages on one
  level below, so at the start we have 1 page (the root) */
  total_recs = 1;

  level = root_level;
  level_is_analyzed = false;

  // ★ 从最大前缀（整键）往最小前缀逐层算——因为下潜层对本层所有前缀都有效
  for (n_prefix = n_uniq; n_prefix >= 1; n_prefix--) {
    /* descend into the tree, searching for "good enough" level */
    for (;;) {
      ut_ad(level > 0);
      ut_ad(!level_is_analyzed);

      /* Do not scan if this would read too many pages. */
      if (total_recs > n_sample_pages) {
        /* step one level back */
        level++;                        // ★ 本层页数已超 A ⇒ 退回上一层当 LA
        level_is_analyzed = true;
        break;
      }

      const uint64_t prev_total_recs = total_recs;
      if (!dict_stats_analyze_index_level(
              index, level, n_diff_on_level, &total_recs, &total_pages,
              n_diff_boundaries, wait_start_time, &mtr)) {
        /* The other index->lock waiter is near to timeout.
        We should reduce requested pages for safety. */
        ut_ad(prev_total_recs <= n_sample_pages);
        n_sample_pages = prev_total_recs / 2;   // ★ 收缩重试，见 5.5.4
        succeeded = false;
        goto end;
      }

      level_is_analyzed = true;

      // ★ 停止下潜的两个条件：已到 level 1，或本层 NDV 已 ≥ A*10
      if (level == 1 || n_diff_on_level[n_prefix - 1] >= n_diff_required) {
        break;
      }

      level--;
      level_is_analyzed = false;
    }
  found_level:
    n_diff_data_t *data = &n_diff_data[n_prefix - 1];

    data->level = level;
    data->n_recs_on_level = total_recs;                     // ★ R 的分母 TOTAL_LA
    data->n_diff_on_level = n_diff_on_level[n_prefix - 1];  // ★ R 的分子 N_DIFF_LA
    data->n_leaf_pages_to_analyze =
        std::min(n_sample_pages, n_diff_on_level[n_prefix - 1]);
    // ★ A = min(采样页数, 层内 NDV)：不能比该层的"不同值组数"还多
    ...
  }
  ...
```

`dict_stats_analyze_index_for_n_prefix()`（`dict0stats.cc:1425`）——**A 个叶页是"分段 + 段内随机"选出来的**：

```cpp
static bool dict_stats_analyze_index_for_n_prefix(
    dict_index_t *index, ulint n_prefix, const boundaries_t *boundaries,
    n_diff_data_t *n_diff_data,
    std::chrono::steady_clock::time_point &wait_start_time, mtr_t *mtr) {
  ...
  n_diff_data->n_diff_all_analyzed_pages = 0;   // ★ Σ Pi
  n_diff_data->n_external_pages_sum = 0;        // ★ Σ Ei

  for (i = 0; i < n_diff_data->n_leaf_pages_to_analyze; i++) {
    /* there are n_diff_on_level elements
    in 'boundaries' and we divide those elements
    into n_leaf_pages_to_analyze segments, ...
    then we select a random record from each segment and dive below it */
    const uint64_t n_diff = n_diff_data->n_diff_on_level;
    const uint64_t n_pick = n_diff_data->n_leaf_pages_to_analyze;

    // ★ 把 [0, n_diff_on_level) 均分成 A 段，第 i 段 = [left, right]
    const uint64_t left = n_diff * i / n_pick;
    const uint64_t right = n_diff * (i + 1) / n_pick - 1;

    ut_a(left <= right);
    ut_a(right <= last_idx_on_level);

    // ★ 段内**随机**取一个"不同键组"⇒ 覆盖全键域，不是取前 A 个
    const uint64_t rnd = ut::random_from_interval(left, right);
    const uint64_t dive_below_idx = boundaries->at(rnd);

    /* seek to the record with index dive_below_idx */
    while (rec_idx < dive_below_idx && pcur.is_on_user_rec()) {
      pcur.move_to_next_user_rec(mtr);
      rec_idx++;
    }
    ...
    if ((i + 1) % 100 == 0 &&
        dict_stats_index_long_waiters(index, wait_start_time)) {
      /* long waiters exist. abort. */
      break;
    }
    ...
    // ★ 下潜到叶页：得到该页上 n_prefix 前缀的不同键数 Pi 与外部页数 Ei
    dict_stats_analyze_index_below_cur(pcur.get_btr_cur(), n_prefix,
                                       &n_diff_on_leaf_page, &n_external_pages);

    /* We adjust n_diff_on_leaf_page here to avoid counting
    one value twice - once as the last on some page and once
    as the first on another page. ...
    our algo would (correctly) get an estimate that there are
    2 distinct records per page (average). Having 4 pages below
    non-boring records, it would (wrongly) estimate the number of
    distinct records to be 8. */
    if (n_diff_on_leaf_page > 0) {
      n_diff_on_leaf_page--;    // ★ 相邻页首尾同值只算一次
    }

    n_diff_data->n_diff_all_analyzed_pages += n_diff_on_leaf_page;  // ★ Σ Pi
    n_diff_data->n_external_pages_sum += n_external_pages;          // ★ Σ Ei
  }

  pcur.close();

  if (i < n_diff_data->n_leaf_pages_to_analyze) {
    /* return how much progressed */
    n_diff_data->n_leaf_pages_to_analyze = i;  // ★ 中途退出：回传真实进度
    return false;
  }

  return true;
}
```

`n_diff_data_t` 各量来源（`dict0stats.cc:1370`）：

```cpp
struct n_diff_data_t {
  ulint level;                          // 停止下潜的层（>=1，不扫叶层 0）
  uint64_t n_recs_on_level;             // R 的分母 TOTAL_LA（analyze_index_level 的 total_recs）
  uint64_t n_diff_on_level;             // R 的分子 N_DIFF_LA（analyze_index_level 的 n_diff[]）
  uint64_t n_leaf_pages_to_analyze;     // A = min(n_sample_pages, n_diff_on_level)
  uint64_t n_diff_all_analyzed_pages;   // Σ Pi（for_n_prefix 内累计）
  uint64_t n_external_pages_sum;        // Σ Ei（for_n_prefix 内累计）
};
```

#### 5.5.3 层内一次扫描同时算出所有前缀的 NDV（`analyze_index_level`）

`dict_stats_analyze_index_level()`（`dict0stats.cc:793`）三个统计量的累加点：

```cpp
static bool dict_stats_analyze_index_level(
    dict_index_t *index, ulint level, uint64_t *n_diff, uint64_t *total_recs,
    uint64_t *total_pages, boundaries_t *n_diff_boundaries,
    std::chrono::steady_clock::time_point &wait_start_time, mtr_t *mtr) {
  ...
  /* elements in the n_diff array are 0..n_uniq-1 (inclusive) */
  memset(n_diff, 0x0, n_uniq * sizeof(n_diff[0]));
  // ★ n_diff[0..n_uniq-1]：一次扫描同时统计**所有前缀**的层内 NDV
  ...
  *total_recs = 0;   // ★ R 的分母
  *total_pages = 0;  // ★ 层内页数

  for (; pcur.is_on_user_rec(); pcur.move_to_next_user_rec(mtr)) {
    ...
    rec_is_last_on_page = page_rec_is_supremum(page_rec_get_next_const(rec));

    /* increment the pages counter at the end of each page */
    if (rec_is_last_on_page) {
      (*total_pages)++;          // ★ 累加点 1
      ...
    }

    /* Skip delete-marked records on the leaf level. ... */
    if (level == 0 &&
        (srv_stats_include_delete_marked
             ? 0
             : rec_get_deleted_flag(rec, page_is_comp(pcur.get_page())))) {
      ...
      continue;                  // ★ delete-marked 记录不计入 total_recs
    }
    rec_offsets = rec_get_offsets(rec, index, rec_offsets, n_uniq,
                                  UT_LOCATION_HERE, &heap);

    (*total_recs)++;             // ★ 累加点 2

    if (prev_rec != nullptr) {
      ulint matched_fields;

      prev_rec_offsets = rec_get_offsets(prev_rec, index, prev_rec_offsets,
                                         n_uniq, UT_LOCATION_HERE, &heap);

      cmp_rec_rec_with_match(rec, prev_rec, rec_offsets, prev_rec_offsets,
                             index, false, false, &matched_fields);
      // ★ matched_fields = 相邻两条记录相等的前缀列数

      for (i = matched_fields; i < n_uniq; i++) {
        // ★ 前缀 i+1 出现了新值 ⇒ n_diff[i]++；一次比较同时更新多个前缀
        if (n_diff_boundaries != nullptr) {
          uint64_t idx;
          idx = *total_recs - 2;
          n_diff_boundaries[i].push_back(idx);   // ★ 记录每个前缀的分组右边界（供随机下潜）
        }
        n_diff[i]++;             // ★ 累加点 3
      }
    } else {
      /* this is the first non-delete marked record */
      for (i = 0; i < n_uniq; i++) {
        n_diff[i] = 1;           // ★ 首记录：所有前缀各 1 个不同值
      }
    }
    ...
  }
  ...
```

#### 5.5.4 重试收缩：长等待者让路，采样页数**减半**重试

```cpp
// dict0stats.cc:1987 —— 外层 while：失败就用收缩后的 N 重试
static void dict_stats_analyze_index(dict_index_t *index) {
  uint64_t n_sample_pages = N_SAMPLE_PAGES(index);   // 初值 = 表选项或全局（20）
  while (n_sample_pages > 0 &&
         !dict_stats_analyze_index_low(n_sample_pages, index)) {
    /* aborted. retrying. */
    ib::warn(ER_IB_MSG_STATS_SAMPLING_TOO_LARGE)
        << "Detected too long lock waiting around " << index->table->name << "."
        << index->name
        << " stats_sample_pages. Retrying with the smaller value "
        << n_sample_pages << ".";

    /* Certain delay is needed for waiters to lock the index next. */
    std::this_thread::sleep_for(std::chrono::milliseconds(100));  // ★ 让等待者先拿锁
  }
}
```

```cpp
// dict0stats.cc:1888 / 1948 —— 内层两个收缩点
      if (!dict_stats_analyze_index_level(...)) {
        ut_ad(prev_total_recs <= n_sample_pages);
        n_sample_pages = prev_total_recs / 2;   // ★ 收缩点 1：按"已扫页数"减半
        succeeded = false;
        goto end;
      }
      ...
    if (!dict_stats_analyze_index_for_n_prefix(...)) {
      ut_ad(data->n_leaf_pages_to_analyze <= n_sample_pages);
      n_sample_pages = data->n_leaf_pages_to_analyze / 2;  // ★ 收缩点 2：按实际进度减半
      succeeded = false;
      goto end;
    }
```

⇒ **`innodb_stats_persistent_sample_pages=20` 只是"请求值"**；长等待者出现时实际采样页数可能被砍到 10、5、2…，`sample_size` 落库的就是最终的 A。这也是"同一张表两次 ANALYZE 的 `n_diff_pfxNN` 不同"的常见原因之一。

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

#### 6.1.1 先纠三个常见误称

| 常见说法 | 8.0.39 源码真相 |
|---|---|
| 枚举叫 `dict_stats_recalc_type` | 实际是 `dict_stats_upd_option_t`，**4 个值**（见下） |
| 持久统计有"表级 + 索引级"三种 `STATS_PERSISTENT` | **只有全局 + 表级两级**。`dict_index_t` 没有 `is_persistent()`/`stats_persistent`；`is_persistent()` 只属于 `dd::Table` |
| 有 `dict_stats_update_if_needed()` | 不存在。触发入口是 `ha_innobase::info_low()` 里的 `dict_stats_update()`；自动更新在 `row_update_statistics_if_needed()` |

**四种模式枚举（不是三种）** `storage/innobase/include/dict0stats.h:43`：

```cpp
enum dict_stats_upd_option_t {
  DICT_STATS_RECALC_PERSISTENT,  // 精确重算 + 落盘（无持久存储时告警并退化为 transient）
  DICT_STATS_RECALC_TRANSIENT,   // 快速瞬时重算，不落盘
  DICT_STATS_EMPTY_TABLE,        // 全部置 0/1（等价"空表"）；persistent 表会落盘
  DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY  // 仅当内存未初始化时从磁盘读回
};
```

#### 6.1.2 分派：`dict_stats_update()` 完整

```cpp
// storage/innobase/dict/dict0stats.cc:2814
dberr_t dict_stats_update(dict_table_t *table, dict_stats_upd_option_t stats_upd_option) {
  ut_ad(!dict_sys_mutex_own());

  if (table->ibd_file_missing) {
    // ★ .ibd 丢失：清空统计。注意这里**仍可能写盘**（取决于后续调用方），
    //   但空表统计意义明确，故直接 empty 后返回
    if (!dict_table_is_discarded(table)) {
      ib::warn(ER_IB_MSG_224)
          << "Cannot calculate statistics for table " << table->name
          << " because the .ibd file is missing. " << TROUBLESHOOTING_MSG;
    }
    dict_stats_empty_table(table);
    return (DB_TABLESPACE_DELETED);
  } else if (srv_force_recovery >= SRV_FORCE_NO_IBUF_MERGE) {
    // ★ 高 force_recovery 下不算统计：坏索引可能让统计扫描直接崩
    dict_stats_empty_table(table);
    return (DB_SUCCESS);
  }

  switch (stats_upd_option) {
    dberr_t err;

    case DICT_STATS_RECALC_PERSISTENT:
      if (srv_read_only_mode) {
        break;   // ★ 只读模式写不了 mysql.innodb_*_stats ⇒ break 落到 transient
      }

      /* wakes the last purge batch for exact recalculation */
      if (trx_sys->rseg_history_len.load() > 0) {
        srv_wake_purge_thread_if_not_active();
      }

      /* Persistent recalculation requested, called from
      1) ANALYZE TABLE, or
      2) the auto recalculation background thread, or
      3) open table if stats do not exist on disk and auto recalc is enabled */

      /* InnoDB internal tables (e.g. SYS_TABLES) cannot have
      persistent stats enabled */
      ut_a(strchr(table->name.m_name, '/') != nullptr);   // 名字必为 "db/table"

      err = dict_stats_update_persistent(table);   // ★ 只算，不落盘

      if (err != DB_SUCCESS) {
        return (err);
      }

      return (dict_stats_save(table, nullptr));    // ★ 落盘在这里

    case DICT_STATS_RECALC_TRANSIENT:
      break;   // ★ 直接落到函数尾走 transient

    case DICT_STATS_EMPTY_TABLE:
      dict_stats_empty_table(table);

      /* If table is using persistent stats, then save the stats on disk */
      if (dict_stats_is_persistent_enabled(table)) {
        return (dict_stats_save(table, nullptr));  // ★ 空表统计也落盘
      }
      return (DB_SUCCESS);

    case DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY:
      /* fetch requested, either fetch from persistent statistics
      storage or use the old method */
      if (table->stat_initialized) {
        return (DB_SUCCESS);    // ★ 内存里已有 ⇒ 什么都不做（"ONLY_IF_NOT_IN_MEMORY"）
      }
      ut_a(strchr(table->name.m_name, '/') != nullptr);

      /* Create a dummy table object with the same name and
      indexes, suitable for fetching the stats into it. */
      dict_table_t *t = dict_stats_table_clone_create(table);
      // ★ 先克隆一份：读回失败时不能污染真实 table 的统计

      err = dict_stats_fetch_from_ps(t);

      t->stats_last_recalc = table->stats_last_recalc;
      t->stat_modified_counter = 0;

      switch (err) {
        case DB_SUCCESS:
          dict_table_stats_lock(table, RW_X_LATCH);
          dict_stats_copy(table, t);   // ★ 读回成功才拷贝回真实 table
          dict_stats_assert_initialized(table);
          dict_table_stats_unlock(table, RW_X_LATCH);
          dict_stats_table_clone_free(t);
          return (DB_SUCCESS);
        case DB_STATS_DO_NOT_EXIST:
          // ★ 磁盘上没统计：
          dict_stats_table_clone_free(t);
          if (srv_read_only_mode) {
            break;
          }
          if (dict_stats_auto_recalc_is_enabled(table)) {
            // ★ 自动重算开着 ⇒ 立刻做一次精确重算并落盘（递归回本函数）
            return (dict_stats_update(table, DICT_STATS_RECALC_PERSISTENT));
          }
          ib::info(ER_IB_MSG_225)
              << "Trying to use table " << table->name
              << " which has persistent statistics enabled,"
                 " but auto recalculation turned off and the"
                 " statistics do not exist in " TABLE_STATS_NAME_PRINT
                 " and " INDEX_STATS_NAME_PRINT ". ...";
          break;   // ★ 否则退化为瞬态统计
        default:
          dict_stats_table_clone_free(t);
          ib::error(ER_IB_MSG_226) << "Error fetching persistent statistics ...";
          break;
      }
      /* no "default:" in order to produce a compilation warning
      about unhandled enumeration value */
  }

  // ★ 所有 break 的落点：RECALC_TRANSIENT / 只读 / fetch 失败 都执行瞬态统计
  dict_table_stats_lock(table, RW_X_LATCH);
  dict_stats_update_transient(table);
  dict_table_stats_unlock(table, RW_X_LATCH);

  return (DB_SUCCESS);
}
```

`dict_stats_update_persistent()`（`dict0stats.cc:2009`）——**只算不落盘**，注意行数取自聚簇索引全列 NDV：

```cpp
static dberr_t dict_stats_update_persistent(dict_table_t *table) {
  dict_index_t *index;

  dict_table_stats_lock(table, RW_X_LATCH);   // ★ stat_* 成员加 X latch

  /* analyze the clustered index first */
  index = table->first_index();

  if (index == nullptr || index->is_corrupted() ||
      (index->type | DICT_UNIQUE) != (DICT_CLUSTERED | DICT_UNIQUE)) {
    /* Table definition is corrupt */
    dict_table_stats_unlock(table, RW_X_LATCH);
    dict_stats_empty_table(table);
    return (DB_CORRUPTION);
  }

  ut_ad(!dict_index_is_ibuf(index));

  dict_stats_analyze_index(index);   // ★ 核心采样（REF01，见 5.5）

  ulint n_unique = dict_index_get_n_unique(index);

  table->stat_n_rows = index->stat_n_diff_key_vals[n_unique - 1];
  // ★ 表行数 = 聚簇索引"全列前缀"的 NDV（这就是 innodb_table_stats.n_rows 的来源）

  table->stat_clustered_index_size = index->stat_index_size;

  /* analyze other indexes from the table, if any */
  table->stat_sum_of_other_index_sizes = 0;

  for (index = index->next(); index != nullptr; index = index->next()) {
    ut_ad(!dict_index_is_ibuf(index));

    if (index->type & DICT_FTS || dict_index_is_spatial(index)) {
      continue;   // ★ 全文/空间索引不参与
    }

    dict_stats_empty_index(index);

    if (dict_stats_should_ignore_index(index)) {
      continue;
    }

    if (!(table->stats_bg_flag & BG_STAT_SHOULD_QUIT)) {
      // ★ 后台线程被要求退出时跳过剩余二级索引（统计可能"算一半"）
      dict_stats_analyze_index(index);
    }

    table->stat_sum_of_other_index_sizes += index->stat_index_size;
  }

  table->stats_last_recalc = std::chrono::steady_clock::now();
  table->stat_modified_counter = 0;   // ★ 与 10% 阈值配合
  table->stat_initialized = true;

  dict_stats_assert_initialized(table);
  dict_table_stats_unlock(table, RW_X_LATCH);
  return (DB_SUCCESS);
}
```

`dict_stats_update_transient()`（`dict0stats.cc:684`）与逐索引实现（`:626`）：

```cpp
static void dict_stats_update_transient(dict_table_t *table) {
  dict_index_t *index;
  ulint sum_of_index_sizes = 0;

  index = table->first_index();

  if (dict_table_is_discarded(table)) {
    dict_stats_empty_table(table);   // DISCARD TABLESPACE 的表
    return;
  } else if (index == nullptr) {
    /* Table definition is corrupt */
    ib::warn(ER_IB_MSG_221) << "Table " << table->name
                            << " has no indexes. Cannot calculate statistics.";
    dict_stats_empty_table(table);
    return;
  }

  for (; index != nullptr; index = index->next()) {
    ut_ad(!dict_index_is_ibuf(index));
    if (index->type & DICT_FTS || dict_index_is_spatial(index)) continue;

    dict_stats_empty_index(index);
    if (dict_stats_should_ignore_index(index)) continue;

    dict_stats_update_transient_for_index(index);   // ★ 逐索引估大小 + NDV
    sum_of_index_sizes += index->stat_index_size;
  }

  index = table->first_index();
  table->stat_n_rows =
      index->stat_n_diff_key_vals[dict_index_get_n_unique(index) - 1];

  table->stat_clustered_index_size = index->stat_index_size;
  table->stat_sum_of_other_index_sizes = sum_of_index_sizes - index->stat_index_size;

  table->stats_last_recalc = std::chrono::steady_clock::now();
  table->stat_modified_counter = 0;
  table->stat_initialized = true;
}
```

```cpp
static void dict_stats_update_transient_for_index(dict_index_t *index) {
  if (srv_force_recovery >= SRV_FORCE_NO_TRX_UNDO &&
      (srv_force_recovery >= SRV_FORCE_NO_LOG_REDO || !index->is_clustered())) {
    /* If we have set a high innodb_force_recovery level, do not calculate
    statistics, as a badly corrupted index can cause a crash in it. */
    dict_stats_empty_index(index);
  } else {
    mtr_t mtr;
    ulint size;

    mtr_start(&mtr);
    dict_disable_redo_if_temporary(index->table, &mtr);
    mtr_s_lock(dict_index_get_lock(index), &mtr, UT_LOCATION_HERE);

    size = btr_get_size(index, BTR_TOTAL_SIZE, &mtr);   // ★ 整棵树页数
    if (size != ULINT_UNDEFINED) {
      index->stat_index_size = size;
      size = btr_get_size(index, BTR_N_LEAF_PAGES, &mtr);  // ★ 叶页数
    }
    mtr_commit(&mtr);

    switch (size) {
      case ULINT_UNDEFINED:
        dict_stats_empty_index(index);
        return;
      case 0:
        /* The root node of the tree is a leaf */
        size = 1;
    }

    index->stat_n_leaf_pages = size;

    /* We don't handle the return value since it will be false
    only when some thread is dropping the table ... */
    btr_estimate_number_of_different_key_vals(index);
    // ★ 瞬态 NDV 全靠 btr_estimate_number_of_different_key_vals()：
    //   对每个前缀列做随机下潜，**不用** REF01 外推、不做分层采样
  }
}
```

#### 6.1.3 三种"模式"的真实分层：全局 / 表级（+ 采样页数覆盖）

表级标志成员 `storage/innobase/include/dict0mem.h:2246`：

```cpp
/** The two bits below are set in the 'stat_persistent' member. They
have the following meaning:
1. _ON=0, _OFF=0, no explicit persistent stats setting for this table,
the value of the global srv_stats_persistent is used ...
2. _ON=0, _OFF=1, persistent stats are explicitly disabled for this table ...
3. _ON=1, _OFF=0, persistent stats are explicitly enabled for this table ...
4. _ON=1, _OFF=1, not allowed, we assert if this ever happens. */
#define DICT_STATS_PERSISTENT_ON (1 << 1)
#define DICT_STATS_PERSISTENT_OFF (1 << 2)

  uint32_t stat_persistent;      // 见上面的位含义
  uint32_t stats_auto_recalc;    // STATS_AUTO_RECALC=1/0
  ulint stats_sample_pages;      // 表级 STATS_SAMPLE_PAGES；0 表示用全局
```

判定优先级 —— `storage/innobase/include/dict0stats.ic:67`：

```cpp
static inline bool dict_stats_is_persistent_enabled(const dict_table_t *table) {
  /* Because of the nature of this check (non-locking) it is possible
  that a table becomes: PS-disabled immediately after this function has
  returned true or PS-enabled immediately after ... This is acceptable.
  Avoiding this would mean that we would have to protect the ::stat_persistent
  with dict_table_stats_lock() ... too big performance penalty,
  especially when this function is called from row_update_statistics_if_needed().*/

  /* we rely on this read to be atomic */
  uint32_t stat_persistent = table->stat_persistent;

  // ★ 优先级：表级显式 ON > 表级显式 OFF > 全局 srv_stats_persistent
  if (stat_persistent & DICT_STATS_PERSISTENT_ON) {
    ut_ad(!(stat_persistent & DICT_STATS_PERSISTENT_OFF));
    return true;
  } else if (stat_persistent & DICT_STATS_PERSISTENT_OFF) {
    return false;
  } else {
    return (srv_stats_persistent);
  }
}

static inline void dict_stats_set_persistent(dict_table_t *table, bool ps_on,
                                            bool ps_off) {
  if (ps_on && ps_off) {
    ps_off = false;   // ★ 同时给 ON/OFF 时 ON 胜出
  }
  uint32_t stat_persistent = 0;
  if (ps_on) stat_persistent |= DICT_STATS_PERSISTENT_ON;
  if (ps_off) stat_persistent |= DICT_STATS_PERSISTENT_OFF;
  table->stat_persistent = stat_persistent;
}
```

赋值来源（`storage/innobase/handler/ha_innodb.cc:2907` / `:14463` 一带）：

```cpp
  if (innodb_table->is_temporary()) {
    /* Temp tables do not use persistent stats. */
    ps_on = false;
    ps_off = true;    // ★ 临时表强制 transient
  } else {
    ps_on = create_info->table_options & HA_OPTION_STATS_PERSISTENT;
    ps_off = create_info->table_options & HA_OPTION_NO_STATS_PERSISTENT;
  }
  dict_stats_set_persistent(innodb_table, ps_on, ps_off);
  // 打开既有表路径同理，用 table_share->db_create_options
```

采样页数：全局默认值（`storage/innobase/srv/srv0srv.cc:578`）与取值宏（`dict0stats.cc:130`）：

```cpp
unsigned long long srv_stats_transient_sample_pages = 8;    // 瞬态默认 8
bool srv_stats_persistent = true;                           // 全局默认开
unsigned long long srv_stats_persistent_sample_pages = 20;  // 持久默认 20
bool srv_stats_auto_recalc = true;
```

```cpp
/* Gets the number of leaf pages to sample in persistent stats estimation */
#define N_SAMPLE_PAGES(index)                                    \
  static_cast<uint64_t>((index)->table->stats_sample_pages != 0  \
                            ? (index)->table->stats_sample_pages \
                            : srv_stats_persistent_sample_pages)
// ★ 表级 STATS_SAMPLE_PAGES 非 0 时覆盖全局；只被 dict_stats_analyze_index() 使用

/* number of distinct records on a given level that are required to stop
descending to lower levels and fetch N_SAMPLE_PAGES(index) records
from that level */
#define N_DIFF_REQUIRED(index) (N_SAMPLE_PAGES(index) * 10)   // ★ 即 5.5 里的 A*10
```

sysvar 注册（`ha_innodb.cc:22203`）：`innodb_stats_transient_sample_pages`（默认 8，范围 [1, ~0ULL]）、`innodb_stats_persistent`（默认 ON）、`innodb_stats_persistent_sample_pages`（默认 20）。

### 6.2 存储格式

`mysql.innodb_index_stats` 是**行式**，`stat_name` ∈ {`size`, `n_leaf_pages`, `n_diff_pfx01`, `n_diff_pfx02`, ...}，配 `stat_value` 与 `sample_size`；主键 `(database_name, table_name, index_name, stat_name)`。

`mysql.innodb_table_stats` 存 `(n_rows, clustered_index_size, sum_of_other_index_sizes)`。

#### 6.2.1 表定义在源码里的位置（不是 5.7 的 `dict_stats_create_persistent_table`）

8.0 起这两张表通过 DDSE 接口以 `dd::Object_table` 声明 —— `storage/innobase/handler/ha_innodb.cc:12907`（`innobase_ddse_dict_init`）：

```cpp
static bool innobase_ddse_dict_init(
    dict_init_mode_t dict_init_mode, uint, List<const dd::Object_table> *tables,
    List<const dd::Plugin_tablespace> *tablespaces) {
  ...
  /* Changing these values would change the specification of innodb statistics
  tables. */
  static constexpr size_t DB_NAME_FIELD_SIZE = 64;     // = dict_name::MAX_DB_CHAR_LEN
  static constexpr size_t TABLE_NAME_FIELD_SIZE = 199; // = dict_name::MAX_TABLE_CHAR_LEN

  std::ostringstream db_name_field;
  db_name_field << "database_name VARCHAR(" << DB_NAME_FIELD_SIZE << ") NOT NULL";
  std::string db_field = db_name_field.str();
  ...
  dd::Object_table *innodb_table_stats = dd::Object_table::create_object_table();
  innodb_table_stats->set_hidden(false);   // ★ 用户可见、可手工 UPDATE
  def = innodb_table_stats->target_table_definition();
  def->set_table_name("innodb_table_stats");
  def->add_field(0, "database_name", db_field.c_str());
  def->add_field(1, "table_name", table_field.c_str());
  def->add_field(2, "last_update",
                 "last_update TIMESTAMP NOT NULL \n"
                 "  DEFAULT CURRENT_TIMESTAMP \n"
                 "  ON UPDATE CURRENT_TIMESTAMP");
  def->add_field(3, "n_rows", "n_rows BIGINT UNSIGNED NOT NULL");
  def->add_field(4, "clustered_index_size",
                 "clustered_index_size BIGINT UNSIGNED NOT NULL");
  def->add_field(5, "sum_of_other_index_sizes",
                 "sum_of_other_index_sizes BIGINT UNSIGNED NOT NULL");
  def->add_index(0, "index_pk", "PRIMARY KEY (database_name, table_name)");

  dd::Object_table *innodb_index_stats = dd::Object_table::create_object_table();
  innodb_index_stats->set_hidden(false);
  def = innodb_index_stats->target_table_definition();
  def->set_table_name("innodb_index_stats");
  def->add_field(0, "database_name", db_field.c_str());
  def->add_field(1, "table_name", table_field.c_str());
  def->add_field(2, "index_name", "index_name VARCHAR(64) NOT NULL");
  def->add_field(3, "last_update",
                 "last_update TIMESTAMP NOT NULL"
                 "  DEFAULT CURRENT_TIMESTAMP"
                 "  ON UPDATE CURRENT_TIMESTAMP");
  /*
          There are at least: stat_name='size'
                  stat_name='n_leaf_pages'
                  stat_name='n_diff_pfx%'
  */
  // ★ stat_name 的取值集合在源码注释里就写死了
  def->add_field(4, "stat_name", "stat_name VARCHAR(64) NOT NULL");
  def->add_field(5, "stat_value", "stat_value BIGINT UNSIGNED NOT NULL");
  def->add_field(6, "sample_size", "sample_size BIGINT UNSIGNED");
  def->add_field(7, "stat_description", "stat_description VARCHAR(1024) NOT NULL");
  def->add_index(0, "index_pk",
                 "PRIMARY KEY (database_name, table_name, index_name, stat_name)");
  ...
  tables->push_back(innodb_table_stats);
  tables->push_back(innodb_index_stats);
  ...
}
```

实际 `SHOW CREATE TABLE`：

```sql
CREATE TABLE `innodb_table_stats` (
  `database_name` varchar(64) COLLATE utf8mb3_bin NOT NULL,
  `table_name` varchar(199) COLLATE utf8mb3_bin NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `n_rows` bigint unsigned NOT NULL,
  `clustered_index_size` bigint unsigned NOT NULL,
  `sum_of_other_index_sizes` bigint unsigned NOT NULL,
  PRIMARY KEY (`database_name`,`table_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 COLLATE=utf8mb3_bin STATS_PERSISTENT=0 ROW_FORMAT=DYNAMIC;

CREATE TABLE `innodb_index_stats` (
  `database_name` varchar(64) COLLATE utf8mb3_bin NOT NULL,
  `table_name` varchar(199) COLLATE utf8mb3_bin NOT NULL,
  `index_name` varchar(64) COLLATE utf8mb3_bin NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `stat_name` varchar(64) COLLATE utf8mb3_bin NOT NULL,
  `stat_value` bigint unsigned NOT NULL,
  `sample_size` bigint unsigned DEFAULT NULL,
  `stat_description` varchar(1024) COLLATE utf8mb3_bin NOT NULL,
  PRIMARY KEY (`database_name`,`table_name`,`index_name`,`stat_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 COLLATE=utf8mb3_bin STATS_PERSISTENT=0 ROW_FORMAT=DYNAMIC;
```

⇒ **两张表自己都 `STATS_PERSISTENT=0`**：否则"统计自己的统计"会形成递归。

#### 6.2.2 `stat_name` 三类的生成点（`dict_stats_save()`，`dict0stats.cc:2187`）

```cpp
static dberr_t dict_stats_save(dict_table_t *table_orig,
                               const index_id_t *only_for_index) {
  ...
  table = dict_stats_snapshot_create(table_orig);   // ★ 对 stat_* 做一致性快照（读时锁定）
  ...
  rw_lock_x_lock(dict_operation_lock, UT_LOCATION_HERE);  // ★ 全程持 dict_operation_lock X
  ...
  // ★ 表级统计：DELETE + INSERT（PROCEDURE TABLE_STATS_SAVE）
  ret = dict_stats_exec_sql(pinfo,
                            "PROCEDURE TABLE_STATS_SAVE () IS\n"
                            "BEGIN\n"
                            "DELETE FROM \"" TABLE_STATS_NAME "\"\n"
                            "WHERE\n"
                            "database_name = :database_name AND\n"
                            "table_name = :table_name;\n"
                            "INSERT INTO \"" TABLE_STATS_NAME "\"\n"
                            "VALUES\n"
                            "(\n"
                            ":database_name,\n"
                            ":table_name,\n"
                            ":last_update,\n"
                            ":n_rows,\n"
                            ":clustered_index_size,\n"
                            ":sum_of_other_index_sizes\n"
                            ");\n"
                            "END;",
                            nullptr);
  ...
  trx_t *trx = trx_allocate_for_background();
  if (srv_read_only_mode) {
    trx_start_internal_read_only(trx, UT_LOCATION_HERE);
  } else {
    trx_start_internal(trx, UT_LOCATION_HERE);
  }

  /* Below we do all the modifications in innodb_index_stats in a single
  transaction for performance reasons. Modifying more than one row in a
  single transaction may deadlock with other transactions if they
  lock the rows in different order. ... To prevent deadlocks we always lock
  the rows in the same order - the order of the PK, which is
  (database_name, table_name, index_name, stat_name). This is why below we
  sort the indexes by name and then for each index, do the mods ordered by
  stat_name. */
  // ★ 按索引名排序 = 按 PK 顺序加锁，避免并发 DROP TABLE 的 DELETE 与之死锁
  for (index = table->first_index(); index != nullptr; index = index->next()) {
    indexes[index->name] = index;
  }

  for (it = indexes.begin(); it != indexes.end(); ++it) {
    index = it->second;

    if (only_for_index != nullptr &&
        index->space != only_for_index->m_space_id &&
        index->id != only_for_index->m_index_id) {
      continue;   // ★ dict_stats_update_for_index() 只刷一个索引时用
    }
    if (dict_stats_should_ignore_index(index)) continue;
    ut_ad(!dict_index_is_ibuf(index));

    for (ulint i = 0; i < index->n_uniq; i++) {
      char stat_name[16];
      char stat_description[1024];

      // ★ n_diff_pfxNN 在此生成
      snprintf(stat_name, sizeof(stat_name), "n_diff_pfx%02" ULINTPFS, i + 1);

      /* craft a string that contains the column names */
      // ★ stat_description = 逗号拼起来的列名前缀，如 "a"、"a,b"
      snprintf(stat_description, sizeof(stat_description), "%s",
               index->fields[0].name());
      for (j = 1; j <= i; j++) {
        size_t len = strlen(stat_description);
        snprintf(stat_description + len, sizeof(stat_description) - len, ",%s",
                 index->fields[j].name());
      }

      ret = dict_stats_save_index_stat(
          index, now, stat_name, index->stat_n_diff_key_vals[i],
          &index->stat_n_sample_sizes[i], stat_description, trx);
      if (ret != DB_SUCCESS) goto end;
    }

    // ★ n_leaf_pages（sample_size 传 NULL）
    ret = dict_stats_save_index_stat(index, now, "n_leaf_pages",
                                     index->stat_n_leaf_pages, nullptr,
                                     "Number of leaf pages in the index", trx);
    if (ret != DB_SUCCESS) goto end;

    // ★ size（sample_size 传 NULL）
    ret = dict_stats_save_index_stat(index, now, "size", index->stat_index_size,
                                     nullptr, "Number of pages in the index", trx);
    if (ret != DB_SUCCESS) goto end;
  }

  trx_commit_for_mysql(trx);   // ★ 整批 innodb_index_stats 一次提交
end:
  trx_free_for_background(trx);
  rw_lock_x_unlock(dict_operation_lock);
  dict_stats_snapshot_free(table);
  return (ret);
}
```

索引级仍是 DELETE + INSERT（`dict0stats.cc:2092`）：

```cpp
static dberr_t dict_stats_save_index_stat(dict_index_t *index, lint last_update,
                                          const char *stat_name,
                                          uint64_t stat_value, uint64_t *sample_size,
                                          const char *stat_description, trx_t *trx) {
  ...
  if (sample_size != nullptr) {
    pars_info_add_ull_literal(pinfo, "sample_size", *sample_size);
  } else {
    // ★ size / n_leaf_pages 的 sample_size 写 NULL
    pars_info_add_literal(pinfo, "sample_size", nullptr, UNIV_SQL_NULL,
                          DATA_FIXBINARY, 0);
  }
  ...
  ret = dict_stats_exec_sql(pinfo,
                            "PROCEDURE INDEX_STATS_SAVE () IS\n"
                            "BEGIN\n"
                            "DELETE FROM \"" INDEX_STATS_NAME "\"\n"
                            "WHERE\n"
                            "database_name = :database_name AND\n"
                            "table_name = :table_name AND\n"
                            "index_name = :index_name AND\n"
                            "stat_name = :stat_name;\n"
                            "INSERT INTO \"" INDEX_STATS_NAME "\"\n"
                            "VALUES\n"
                            "(\n"
                            ":database_name,\n"
                            ":table_name,\n"
                            ":index_name,\n"
                            ":last_update,\n"
                            ":stat_name,\n"
                            ":stat_value,\n"
                            ":sample_size,\n"
                            ":stat_description\n"
                            ");\n"
                            "END;",
                            trx);
  ...
}
```

| stat_name | 生成处 | `stat_value` | `sample_size` | `stat_description` |
|---|---|---|---|---|
| `n_diff_pfx01`…`n_diff_pfx%02d` | `snprintf(..., "n_diff_pfx%02", i+1)` | `stat_n_diff_key_vals[i]` | `stat_n_sample_sizes[i]`（= A） | 列名前缀 `a[,b,...]` |
| `n_leaf_pages` | 直接字面量 | `stat_n_leaf_pages` | `NULL` | `"Number of leaf pages in the index"` |
| `size` | 直接字面量 | `stat_index_size` | `NULL` | `"Number of pages in the index"` |

#### 6.2.3 落盘走的是 InnoDB **内部 SQL 解析器**，完全绕开 MySQL SQL 层

```cpp
// storage/innobase/dict/dict0stats.cc:179
/** Executes a given SQL statement using the InnoDB internal SQL parser. */
static dberr_t dict_stats_exec_sql(pars_info_t *pinfo, const char *sql,
                                   trx_t *trx) {
  dberr_t err;
  bool trx_started = false;

  ut_ad(rw_lock_own(dict_operation_lock, RW_LOCK_X));
  ut_ad(!dict_sys_mutex_own());

  if (trx == nullptr) {
    trx = trx_allocate_for_background();   // ★ 自建后台事务
    trx_started = true;
    if (srv_read_only_mode) {
      trx_start_internal_read_only(trx, UT_LOCATION_HERE);
    } else {
      trx_start_internal(trx, UT_LOCATION_HERE);
    }
  }

  err = que_eval_sql(pinfo, sql, trx);   // ★ InnoDB 内部 SQL 解析器，直接以 B-tree 层读写
  //   pinfo is freed here

  if (!trx_started && err == DB_SUCCESS) {
    return (DB_SUCCESS);   // ★ 外部 trx：成功时不在此提交，由调用方统一提交
  }

  if (err == DB_SUCCESS) {
    trx_commit_for_mysql(trx);   // ★ 自建 trx 时用 trx_commit_for_mysql 提交
  } else {
    trx->op_info = "rollback of internal trx on stats tables";
    trx->dict_operation_lock_mode = RW_X_LATCH;
    trx_rollback_to_savepoint(trx, nullptr);
    trx->dict_operation_lock_mode = 0;
    trx->op_info = "";
    ut_a(trx->error_state == DB_SUCCESS);
  }

  if (trx_started) {
    trx_free_for_background(trx);
  }

  return (err);
}
```

⇒ **`ANALYZE TABLE` 不走 MySQL SQL 层去 INSERT**：`que_eval_sql` 直接把 SQL 文本当"过程"在 B-tree 层执行。这也是为什么 `innodb_table_stats` 可以被用户在事务外直接 `UPDATE`（InnoDB 下次读回时信任它），但必须同时改 `last_update` 之类的语义一致性。

#### 6.2.4 读回：`dict_stats_fetch_from_ps()`（`dict0stats.cc:2674`）

```cpp
static dberr_t dict_stats_fetch_from_ps(dict_table_t *table) {
  index_fetch_t index_fetch_arg;
  trx_t *trx;
  pars_info_t *pinfo;
  dberr_t ret;

  ut_ad(!dict_sys_mutex_own());

  /* Initialize all stats to dummy values before fetching because if
  the persistent storage contains incomplete stats (e.g. missing stats
  for some index) then we would end up with (partially) uninitialized
  stats. */
  dict_stats_empty_table(table);   // ★ 先清空，防"部分读到"

  trx = trx_allocate_for_background();

  /* Use 'read-uncommitted' so that the SELECTs we execute
  do not get blocked in case some user has locked the rows we are SELECTing */
  trx->isolation_level = TRX_ISO_READ_UNCOMMITTED;   // ★ RU：不被用户锁住的统计表行阻塞

  if (srv_read_only_mode) {
    trx_start_internal_read_only(trx, UT_LOCATION_HERE);
  } else {
    trx_start_internal(trx, UT_LOCATION_HERE);
  }
  ...
  pars_info_bind_function(pinfo, "fetch_table_stats_step",
                          dict_stats_fetch_table_stats_step, table);

  index_fetch_arg.table = table;
  index_fetch_arg.stats_were_modified = false;
  pars_info_bind_function(pinfo, "fetch_index_stats_step",
                          dict_stats_fetch_index_stats_step, &index_fetch_arg);

  ret = que_eval_sql(pinfo,
                     "PROCEDURE FETCH_STATS () IS\n"
                     "found INT;\n"
                     "DECLARE FUNCTION fetch_table_stats_step;\n"
                     "DECLARE FUNCTION fetch_index_stats_step;\n"
                     "DECLARE CURSOR table_stats_cur IS\n"
                     "  SELECT\n"
                     "  n_rows,\n"
                     "  clustered_index_size,\n"
                     "  sum_of_other_index_sizes\n"
                     "  FROM \"" TABLE_STATS_NAME "\"\n"
                     "  WHERE\n"
                     "  database_name = :database_name AND\n"
                     "  table_name = :table_name;\n"
                     "DECLARE CURSOR index_stats_cur IS\n"
                     "  SELECT\n"
                     "  index_name,\n"
                     "  stat_name,\n"
                     "  stat_value,\n"
                     "  sample_size\n"
                     "  FROM \"" INDEX_STATS_NAME "\"\n"
                     "  WHERE\n"
                     "  database_name = :database_name AND\n"
                     "  table_name = :table_name;\n"
                     "BEGIN\n"
                     "OPEN table_stats_cur;\n"
                     "FETCH table_stats_cur INTO\n"
                     "  fetch_table_stats_step();\n"
                     "IF (SQL % NOTFOUND) THEN\n"
                     "  CLOSE table_stats_cur;\n"
                     "  RETURN;\n"
                     "END IF;\n"
                     "CLOSE table_stats_cur;\n"
                     "OPEN index_stats_cur;\n"
                     "found := 1;\n"
                     "WHILE found = 1 LOOP\n"
                     "  FETCH index_stats_cur INTO\n"
                     "    fetch_index_stats_step();\n"
                     "  IF (SQL % NOTFOUND) THEN\n"
                     "    found := 0;\n"
                     "  END IF;\n"
                     "END LOOP;\n"
                     "CLOSE index_stats_cur;\n"
                     "END;",
                     trx);

  trx_commit_for_mysql(trx);
  trx_free_for_background(trx);

  if (!index_fetch_arg.stats_were_modified) {
    return (DB_STATS_DO_NOT_EXIST);   // ★ 一行索引统计都没读到 ⇒ 视为"不存在"
  }

  return (ret);
}
```

`stat_name` 的解析（`dict_stats_fetch_index_stats_step`，`dict0stats.cc:2455`）：

```cpp
#define PFX "n_diff_pfx"
#define PFX_LEN 10

  if (stat_name_len == 4 /* strlen("size") */
      && native_strncasecmp("size", stat_name, stat_name_len) == 0) {
    index->stat_index_size = (ulint)stat_value;
    arg->stats_were_modified = true;
  } else if (stat_name_len == 12 /* strlen("n_leaf_pages") */
             && native_strncasecmp("n_leaf_pages", stat_name, stat_name_len) == 0) {
    index->stat_n_leaf_pages = (ulint)stat_value;
    arg->stats_were_modified = true;
  } else if (stat_name_len > PFX_LEN /* e.g. stat_name=="n_diff_pfx01" */
             && native_strncasecmp(PFX, stat_name, PFX_LEN) == 0) {
    num_ptr = stat_name + PFX_LEN;

    if (stat_name_len != PFX_LEN + 2 || num_ptr[0] < '0' || num_ptr[0] > '9' ||
        num_ptr[1] < '0' || num_ptr[1] > '9') {
      ...   // ★ 格式非法的行静默忽略
      return true;
    }

    n_pfx = (num_ptr[0] - '0') * 10 + (num_ptr[1] - '0');

    if (n_pfx == 0 || n_pfx > n_uniq) {
      ...   // ★ 越界（比如 DDL 缩了索引列数）也静默忽略
      return true;
    }

    index->stat_n_diff_key_vals[n_pfx - 1] = stat_value;

    if (sample_size != UINT64_UNDEFINED) {
      index->stat_n_sample_sizes[n_pfx - 1] = sample_size;
    } else {
      index->stat_n_sample_sizes[n_pfx - 1] = 0;
    }

    index->stat_n_non_null_key_vals[n_pfx - 1] = 0;

    arg->stats_were_modified = true;
  } else {
    /* silently ignore rows with unknown stat_name, the
    user may have developed her own stats */
    // ★ 用户自定义行不会让 ANALYZE 报错，也不会被采用
  }
```

⇒ **"用户可改统计"是官方支持的行为**：读回侧对未知 `stat_name` 静默忽略、对非法/越界 `n_diff_pfxNN` 也忽略；只要 `stat_value` 合法就会被采信。

#### 6.2.5 RENAME 时的同步

`dict_stats_rename_table()` / `dict_stats_rename_index()`（`dict0stats.cc:3284`）都是在两张统计表上跑 `UPDATE ... SET database_name/table_name/index_name = 新名`：

```cpp
static inline dberr_t dict_stats_rename_table_in_table_stats(...) {
  ...
  ret = dict_stats_exec_sql(pinfo,
                            "PROCEDURE RENAME_TABLE_IN_TABLE_STATS () IS\n"
                            "BEGIN\n"
                            "UPDATE \"" TABLE_STATS_NAME "\" SET\n"
                            "database_name = :new_dbname_utf8mb3,\n"
                            "table_name = :new_tablename_utf8mb3\n"
                            "WHERE\n"
                            "database_name = :old_dbname_utf8mb3 AND\n"
                            "table_name = :old_tablename_utf8mb3;\n"
                            "END;\n",
                            nullptr);
  return (ret);
}

dberr_t dict_stats_rename_table(const char *old_name, const char *new_name,
                                char *errstr, size_t errstr_sz) {
  ...
  /* skip innodb_table_stats and innodb_index_stats themselves */
  if (strcmp(old_name, TABLE_STATS_NAME) == 0 ||
      strcmp(old_name, INDEX_STATS_NAME) == 0 || ...) {
    return (DB_SUCCESS);
  }
  ...
  rw_lock_x_lock(dict_operation_lock, UT_LOCATION_HERE);

  ulint n_attempts = 0;
  do {
    n_attempts++;
    ret = dict_stats_rename_table_in_table_stats(...);
    if (ret == DB_DUPLICATE_KEY) {
      // ★ 目标名已有残留行（例如上次改名失败）⇒ 先删目标行再改
      dict_stats_delete_from_table_stats(new_db_utf8mb3, new_table_utf8mb3);
    }
    if (ret == DB_STATS_DO_NOT_EXIST) {
      ret = DB_SUCCESS;   // ★ 本来就没统计 ⇒ 视为成功
    }
    if (ret != DB_SUCCESS) {
      rw_lock_x_unlock(dict_operation_lock);
      std::this_thread::sleep_for(std::chrono::milliseconds(200));
      rw_lock_x_lock(dict_operation_lock, UT_LOCATION_HERE);
    }
  } while ((ret == DB_DEADLOCK || ret == DB_DUPLICATE_KEY ||
            ret == DB_LOCK_WAIT_TIMEOUT) && n_attempts < 5);
  // ★ 最多重试 5 次、每次间隔 200ms，失败就静默放弃（统计只是"估算"，
  //   不值得因它让 RENAME 失败）；随后对 innodb_index_stats 重复同样循环
  ...
}
```

⇒ 与 `TABLE_SHARE`/TDC 的失效机制对照：**统计表的失效是"改名/删除同步"而非版本号**——RENAME 同步改统计表行名，DROP 时删行，改表结构（列/索引变化）时由 `dict_stats_update` 重算覆盖（`n_diff_pfxNN` 的行数按 `index->n_uniq` 生成，旧的多余行会被 DELETE 覆盖到新值或残留但被读回侧忽略）。

### 6.3 自动更新阈值：`row_update_statistics_if_needed()` 完整

```cpp
// storage/innobase/row/row0mysql.cc:1126
/** Updates the table modification counter and calculates new estimates
 for table and index statistics if necessary. */
static inline void row_update_statistics_if_needed(dict_table_t *table) {
  uint64_t counter;
  uint64_t n_rows;

  if (!table->stat_initialized) {
    DBUG_EXECUTE_IF("test_upd_stats_if_needed_not_inited",
                    fprintf(stderr, "test_upd_stats_if_needed_not_inited"
                                    " was executed\n"););
    return;   // ★ 未初始化先不算
  }

  counter = table->stat_modified_counter++;   // ★ 每次 DML 后自增
  n_rows = dict_table_get_n_rows(table);

  if (dict_stats_is_persistent_enabled(table)) {
    // ★ persistent：修改量 > 10% 且 auto_recalc 开启 ⇒ 塞进后台 recalc 池（**异步**）
    if (counter > n_rows / 10 /* 10% */
        && dict_stats_auto_recalc_is_enabled(table)) {
      dict_stats_recalc_pool_add(table);
      table->stat_modified_counter = 0;
    }
    return;
  }

  /* Calculate new statistics if 1 / 16 of table has been modified
  since the last time a statistics batch was run.
  We calculate statistics at most every 16th round, since we may have
  a counter table which is very small and updated very often. */

  // ★ transient：counter > 16 + n_rows/16（6.25%），且至少 16 次改动
  if (counter > 16 + n_rows / 16 /* 6.25% */) {
    ut_ad(!dict_sys_mutex_own());
    /* this will reset table->stat_modified_counter to 0 */
    dict_stats_update(table, DICT_STATS_RECALC_TRANSIENT);   // ★ 同步立即重算
  }
}
```

- **持久统计**：修改行数 **> 10%** 加入 recalc pool（后台线程执行）
- **瞬态统计**：`> 6.25% + 16`（+16 避免极小的"计数器表"被过度触发）
- 表级 `STATS_AUTO_RECALC=1/0` **优先**于全局 `innodb_stats_auto_recalc`（默认 true）
- `stat_modified_counter` 语义 = **自上次统计重算以来的修改次数**；命中阈值即清零，`dict_stats_update_persistent/transient()` 结尾也清零

后台线程消费点：`dict_stats_recalc_pool_add()` 定义在 `storage/innobase/dict/dict0stats_bg.cc:119`；`dict_stats` 后台线程循环里 `dict_stats_recalc_pool_get()` 取出 → `dict_stats_update(table, DICT_STATS_RECALC_PERSISTENT)`，失败则再 `dict_stats_recalc_pool_add()` 重新排队（`:317` 附近）——因此一次后台统计失败不会永久丢统计。

### 6.4 谁真正触发统计计算：`ha_innobase::info_low()` 的三个 HA_STATUS 分支

```cpp
// storage/innobase/handler/ha_innodb.cc:17208
int ha_innobase::info_low(uint flag, bool is_analyze) {
  ...
  ib_table = m_prebuilt->table;

  if (flag & HA_STATUS_TIME) {
    // ★ 唯一会真正触发统计计算的入口
    if (is_analyze || innobase_stats_on_metadata) {
      dict_stats_upd_option_t opt;
      dberr_t ret;

      m_prebuilt->trx->op_info = "updating table statistics";

      if (dict_stats_is_persistent_enabled(ib_table)) {
        if (is_analyze) {
          opt = DICT_STATS_RECALC_PERSISTENT;   // ★ ANALYZE TABLE → 精确重算 + 落盘
        } else {
          /* This is e.g. 'SHOW INDEXES', fetch
          the persistent stats from disk. */
          opt = DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY;   // ★ SHOW INDEX/STATUS → 只读回
        }
      } else {
        opt = DICT_STATS_RECALC_TRANSIENT;      // ★ 非持久表 → 瞬时重算
      }

      ut_ad(!dict_sys_mutex_own());
      ret = dict_stats_update(ib_table, opt);

      if (ret != DB_SUCCESS) {
        m_prebuilt->trx->op_info = "";
        return HA_ERR_GENERIC;
      }
      m_prebuilt->trx->op_info = "returning various info to MySQL";
    }

    stats.update_time = (ulong)std::chrono::system_clock::to_time_t(
        ib_table->update_time.load());
  }

  if (flag & HA_STATUS_VARIABLE) {
    // ★ 只读内存里的 stat_*，**不触发计算**
    ulint stat_clustered_index_size;
    ulint stat_sum_of_other_index_sizes;

    if (!(flag & HA_STATUS_NO_LOCK)) {
      dict_table_stats_lock(ib_table, RW_S_LATCH);   // ★ 与 stat_* 修改者互斥
    }

    ut_a(ib_table->stat_initialized);

    n_rows = ib_table->stat_n_rows;
    stat_clustered_index_size = ib_table->stat_clustered_index_size;
    stat_sum_of_other_index_sizes = ib_table->stat_sum_of_other_index_sizes;

    if (!(flag & HA_STATUS_NO_LOCK)) {
      dict_table_stats_unlock(ib_table, RW_S_LATCH);
    }

    if (n_rows == 0 && !(flag & HA_STATUS_TIME) &&
        table_share->table_category != TABLE_CATEGORY_TEMPORARY) {
      n_rows++;   // ★ 优化器视角：0 行补 1，避免"空表"导致左连接/anti-join 误判
    }

    stats.records = (ha_rows)n_rows;
    stats.deleted = 0;

    calculate_index_size_stats(ib_table, n_rows, stat_clustered_index_size,
                               stat_sum_of_other_index_sizes, &stats);
    ...
  }

  if (flag & HA_STATUS_CONST) {
    // ★ 填充每索引 records_per_key（供优化器）——见第 4 章
    ...
  }
```

⇒ **读语义要分清**：`HA_STATUS_VARIABLE`（`records`/`deleted`）与 `HA_STATUS_CONST`（`records_per_key`）都只是**读内存**；真正"算统计"的只有 `HA_STATUS_TIME` 且 (`is_analyze` 或 `innodb_stats_on_metadata=ON`)。因此 `SELECT` 走 `info()` 默认分支时**不会**因为读统计而变慢——这正是"`innodb_stats_on_metadata` 曾让 `SHOW TABLE STATUS` 卡顿"的开关所在。

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

> 源码坐标：`[C9] storage/innobase/btr/btr0cur.cc` · `[C7] storage/innobase/handler/ha_innodb.cc` · `[C18] sql/range_optimizer/index_range_scan_plan.cc`

```
        root
        /  \                path1 = 下潜到 range_start 的完整路径
      ...   ...             path2 = 下潜到 range_end   的完整路径
      /         \           自 root 向下逐层比较 slot1/slot2：
    [L1]      [L1]           - 两条路径同页同记录 → 未分叉，继续向下一层
    /   \     /   \          - 首次 nth_rec 不同 → diverged（同页内数记录差）
  slot1  ...  slot2          - 落在不同页 → diverged_lot（扫该层中间页，最多 10 页后外推）
```

**入口** `ha_innobase::records_in_range()`：

```cpp
  heap = mem_heap_create(2 * (key->actual_key_parts * sizeof(dfield_t) + sizeof(dtuple_t)), ...);
  range_start = dtuple_create(heap, key->actual_key_parts);
  range_end   = dtuple_create(heap, key->actual_key_parts);
  // MySQL 的 key_range → InnoDB 的 dtuple
  row_sel_convert_mysql_key_to_innobase(range_start, ..., min_key ? min_key->key : nullptr, ...);
  row_sel_convert_mysql_key_to_innobase(range_end,   ..., max_key ? max_key->key : nullptr, ...);
  mode1 = convert_search_mode_to_innobase(min_key ? min_key->flag : HA_READ_KEY_EXACT);
  mode2 = convert_search_mode_to_innobase(max_key ? max_key->flag : HA_READ_KEY_EXACT);

  if (mode1 != PAGE_CUR_UNSUPP && mode2 != PAGE_CUR_UNSUPP) {
    if (dict_index_is_spatial(index))
      n_rows = rtr_estimate_n_rows_in_range(index, range_start, mode1);  // 空间索引只下潜一次
    else
      n_rows = btr_estimate_n_rows_in_range(index, range_start, mode1, range_end, mode2);
  } else {
    n_rows = HA_POS_ERROR;              // 无法估算，交给调用方退化
  }
  ...
  /* 优化器会把 0 行当成"确定为空"并直接返回 Empty set；加锁读也要真正做一次搜索
     以加 next-key 锁，所以这里 +1，避免 MySQL 做出该假设 */
  if (n_rows == 0) n_rows = 1;
```

★ **双向下潜 + 边界是否计入**：

```cpp
// [C9] btr_estimate_n_rows_in_range_low()
  if (dtuple_get_n_fields(tuple1) > 0) {
    btr_cur_search_to_nth_level(index, 0, tuple1, mode1,
                                BTR_SEARCH_LEAF | BTR_ESTIMATE, &cursor, 0, ..., &mtr);
    /* 例如树里最大记录是 5、条件是 x > 3：cursor 落在 5 → 左边界要计入；
       条件是 x > 7：cursor 落在最右叶页的 supremum → 左边界不计入 */
    should_count_the_left_border = !page_rec_is_supremum(btr_cur_get_rec(&cursor));
  } else {
    btr_cur_open_at_index_side(true, index, BTR_SEARCH_LEAF | BTR_ESTIMATE, &cursor, 0, ..., &mtr);
    should_count_the_left_border = false;      // 无左边界（x < 123）
  }

  if (dtuple_get_n_fields(tuple2) > 0) {
    btr_cur_search_to_nth_level(index, 0, tuple2, mode2, BTR_SEARCH_LEAF | BTR_ESTIMATE, ...);
    const rec_t *rec = btr_cur_get_rec(&cursor);
    should_count_the_right_border =
        (mode2 == PAGE_CUR_LE && cursor.low_match >= dtuple_get_n_fields(tuple2)) ||
        (mode2 == PAGE_CUR_L && !page_rec_is_infimum(rec));
    /* 注意：对 "WHERE col <= 'foo'"，MySQL 传进来的是 min_key=NULL +
       max_key='foo' flag=HA_READ_AFTER_KEY(PAGE_CUR_G)，cursor 落在目标右侧第一条，
       因此右边界不计入 */
  } else {
    btr_cur_open_at_index_side(false, index, BTR_SEARCH_LEAF | BTR_ESTIMATE, ...);
    should_count_the_right_border = false;
  }
```

**路径逐层对比 + 三态状态机**：

```cpp
  auto diverged = false;        // 两条路径不再经过同一页
  auto diverged_lot = false;    // 不再相邻 → 可以扫中间页
  divergence_level = 1000000;
  for (i = 0;; ++i) {
    slot1 = &path1[i];  slot2 = &path2[i];
    if (slot1->nth_rec == ULINT_UNDEFINED || slot2->nth_rec == ULINT_UNDEFINED) {
      /* 到达叶层 */
      if (is_n_rows_exact) {
        if (last1->page_no == last2->page_no && last1->nth_rec == last2->nth_rec) {
          /* 两条路径收敛到同一记录：两个边界都算也不能重复计数 */
          if (should_count_the_left_border && should_count_the_right_border) n_rows = 1;
          else n_rows = 0;                       // 例如 [3,3)
        } else {
          if (should_count_the_left_border)  n_rows++;
          if (should_count_the_right_border) n_rows++;
        }
      }
      if (i > divergence_level + 1 && !is_n_rows_exact)
        n_rows = n_rows * 2;                     // ★ 树高 > 1 时算法倾向低估，乘 2 补偿
      /* 不要让估算值超过全表估算行数的 1/2 */
      if (n_rows > table_n_rows / 2 && !is_n_rows_exact) {
        n_rows = table_n_rows / 2;
        if (n_rows == 0) n_rows = table_n_rows;
      }
      return (n_rows);
    }
    if (!diverged && slot1->nth_rec != slot2->nth_rec) {
      if (slot1->page_no != slot2->page_no || slot1->page_level != slot2->page_level) {
        if (nth_attempt >= rows_in_range_max_retries)   // 默认 4 次
          return (rows_in_range_arbitrary_ret_val);     // 放弃，返回常数 10
        return btr_estimate_n_rows_in_range_low(..., nth_attempt + 1);
      }
      diverged = true;
      if (slot1->nth_rec < slot2->nth_rec) {
        n_rows = slot2->nth_rec - slot1->nth_rec - 1;   // 不数两个边界，故 -1
        if (n_rows > 0) { diverged_lot = true; divergence_level = i; }
      }
    } else if (diverged && !diverged_lot) {
      if (slot1->nth_rec < slot1->n_recs || slot2->nth_rec > 1) {
        diverged_lot = true;  divergence_level = i;  n_rows = 0;
        if (slot1->nth_rec < slot1->n_recs) n_rows += slot1->n_recs - slot1->nth_rec;
        if (slot2->nth_rec > 1)             n_rows += slot2->nth_rec - 1;
      }
    } else if (diverged_lot) {
      n_rows = btr_estimate_n_rows_in_range_on_level(index, slot1, slot2, n_rows,
                                                     &is_n_rows_exact);
    }
  }
```

**扫页上限与外推**：

```cpp
// [C9] btr_estimate_n_rows_in_range_on_level()
  *is_n_rows_exact = true;
  if (slot1->nth_rec <= slot1->n_recs) n_rows += slot1->n_recs - slot1->nth_rec;  // 左页右半
  if (slot2->nth_rec > 1)              n_rows += slot2->nth_rec - 1;              // 右页左半

  /* 只是一个估算，不要为了它读太多页伤性能：读满这么多页还没到 slot2 就用已扫页均值外推 */
  constexpr uint32_t N_PAGES_READ_LIMIT = 10;
  do {
    block = buf_page_get_gen(page_id, page_size, RW_S_LATCH, nullptr,
                             Page_fetch::POSSIBLY_FREED, ..., &mtr);  // 不持 index->lock
    page = buf_block_get_frame(block);
    if (!fil_page_index_page_check(page) ||
        btr_page_get_index_id(page) != index->id ||
        btr_page_get_level(page) != level) {
      mtr_commit(&mtr);
      goto inexact;                       // 页被复用给别的东西了
    }
    n_pages_read++;
    if (page_id.page_no() != slot1->page_no) n_rows += page_get_n_recs(page);
    page_id.set_page_no(btr_page_get_next(page, &mtr));
    mtr_commit(&mtr);
    if (n_pages_read == N_PAGES_READ_LIMIT || page_id.page_no() == FIL_NULL)
      goto inexact;                       // 读太多 / 到层尾也没遇到 slot2 → 树变了
  } while (page_id.page_no() != slot2->page_no);
  return (n_rows);

inexact:
  *is_n_rows_exact = false;
  if (n_pages_read > 0)
    /* 本层页数 ≈ n_rows_on_prev_level，乘以"已扫页的平均记录数"外推 */
    n_rows = n_rows_on_prev_level * n_rows / n_pages_read;
  else
    n_rows = 10;                          // 连 slot1 所在页都没扫到就变了
  return (n_rows);
```

#### ★ 何时改用统计估算：`eq_range_index_dive_limit`

```cpp
// [C18] sql/range_optimizer/index_range_scan_plan.cc
  param->use_index_statistics = eq_ranges_exceeds_limit(
      tree, &range_count, thd->variables.eq_range_index_dive_limit);

static bool eq_ranges_exceeds_limit(const SEL_ROOT *keypart, uint *count, uint limit) {
  if (limit == 0) return false;   // 0 = 永远用 index dive
  if (limit == 1) return true;    // 只要有等值 range 就直接用统计
  for (SEL_ARG *keypart_range = keypart->root->first(); keypart_range;
       keypart_range = keypart_range->next) {
    /* 计入条件：1) 无 min/max flag（不是 <, <= 之类）
                 2) 上下界同值（不是 BETWEEN a AND b）
                 3) "x IS NULL" 不计入 —— NULL 数量在统计里通常不准 */
    if (!keypart_range->min_flag && !keypart_range->max_flag &&
        !keypart_range->cmp_max_to_min(keypart_range) &&
        !keypart_range->is_null_interval()) {
      ...
      if (*count >= limit) return true;
    }
  }
  return false;
}
```

★ **调这个变量时最常见的判断错误**：它**只统计纯等值路径**——`BETWEEN` 与 `IS NULL` 都不计入。所以 `IN (1..300)`（默认 limit=200）会切换到统计，而 `col BETWEEN 1 AND 300` **仍然走 dive**。

**三处兜底值得注意**：`N_PAGES_READ_LIMIT=10` 触发外推会置 `is_n_rows_exact=false`，随后触发「×2 低估补偿」与「不超过全表 1/2」两道夹逼；树在两次下潜之间变了最多重试 4 次，然后直接返回常数 **10**（这个值非常粗糙，是 `IN (...)` 大列表偶尔估崩的根源之一）。

⇒ 最后一条：**任何区间估算都不会给出 0 行**（`records_in_range` 末尾 `n_rows==0 → 1`）。

**为什么 index dive 能在优化阶段对每个候选索引都跑一遍**：它只下潜 2 次 + 最多扫 10 个叶页，代价是 **O(树高)**，与区间大小无关。
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

只有两种，且**类型是 `const` 成员、构造后不可变**（`sql/histograms/histogram.h:312`）：

```cpp
class Histogram {
 public:
  /// All supported histogram types in MySQL.
  enum class enum_histogram_type { EQUI_HEIGHT, SINGLETON };

  /// String representation of the JSON field "histogram-type".
  static constexpr const char *histogram_type_str() { return "histogram-type"; }
  static constexpr const char *data_type_str() { return "data-type"; }
  static constexpr const char *collation_id_str() { return "collation-id"; }
  static constexpr const char *singleton_str() { return "singleton"; }
  static constexpr const char *equi_height_str() { return "equi-height"; }

 protected:
  double m_sampling_rate;          // ★ 采样率（0.0~1.0），GEE 修正要用
  double m_null_values_fraction;   // ★ NULL 占比；初值 INVALID_NULL_VALUES_FRACTION = -1.0
  const CHARSET_INFO *m_charset;
  size_t m_num_buckets_specified;  // ★ 用户 ANALYZE 时指定的桶数（可能 > 实际桶数）
  ...
 private:
  MEM_ROOT *m_mem_root;
  const enum_histogram_type m_hist_type;   // ★ const：类型不可变
  const Value_map_type m_data_type;        // ★ const
  LEX_CSTRING m_database_name;
  LEX_CSTRING m_table_name;
  LEX_CSTRING m_column_name;

  bool get_raw_selectivity(Item **items, size_t item_count, enum_operator op,
                           double *selectivity) const;
  ...
 public:
  Histogram() = delete;                          // ★ 无默认构造
  Histogram(const Histogram &other) = delete;    // ★ 禁拷贝：只能 clone(MEM_ROOT*)
  virtual ~Histogram() = default;
  ...
  virtual std::string histogram_type_to_str() const = 0;
  virtual size_t get_num_buckets() const = 0;
  virtual size_t get_num_distinct_values() const = 0;
  virtual bool histogram_to_json(Json_object *json_object) const = 0;
  static Histogram *json_to_histogram(MEM_ROOT *mem_root, ...);
  virtual Histogram *clone(MEM_ROOT *mem_root) const = 0;
  bool store_histogram(THD *thd) const;
  bool get_selectivity(Item **items, size_t item_count, enum_operator op,
                       double *selectivity) const;
  double get_non_null_values_fraction() const {
    return 1.0 - get_null_values_fraction();
  }
};
```

选型（**函数名是自由函数 `build_histogram()`，不是 `generate_histogram()`**，`histogram.cc:441`）：

```cpp
template <class T>
Histogram *build_histogram(MEM_ROOT *mem_root, const Value_map<T> &value_map,
                           size_t num_buckets, const std::string &db_name,
                           const std::string &tbl_name,
                           const std::string &col_name) {
  Histogram *histogram = nullptr;

  /*
    If the number of buckets specified is greater or equal to the number
    of distinct values, we create a Singleton histogram. Otherwise we create
    an equi-height histogram.
  */
  // ★ 判据：num_buckets >= value_map.size()（= distinct 值个数）；等号归 Singleton
  if (num_buckets >= value_map.size()) {
    Singleton<T> *singleton = Singleton<T>::create(
        mem_root, db_name, tbl_name, col_name, value_map.get_data_type());
    if (singleton == nullptr) return nullptr;   // ★ create 内部 catch bad_alloc
    if (singleton->build_histogram(value_map, num_buckets))
      return nullptr;
    histogram = singleton;
  } else {
    Equi_height<T> *equi_height = Equi_height<T>::create(
        mem_root, db_name, tbl_name, col_name, value_map.get_data_type());
    if (equi_height == nullptr) return nullptr;
    if (equi_height->build_histogram(value_map, num_buckets))
      return nullptr;
    histogram = equi_height;
  }

  assert(histogram != nullptr);
  assert(histogram->get_num_buckets_specified() == num_buckets);  // ★ 指定桶数原样记录
  assert(histogram->get_num_buckets() <= num_buckets);            // ★ 实际桶数 <= 指定桶数
  assert(histogram->get_character_set() != nullptr);
  assert(histogram->get_null_values_fraction() >= 0.0);
  assert(histogram->get_null_values_fraction() <= 1.0);
  return histogram;
}
```

⇒ **失败回退**：`create()` 失败（OOM）或 `build_histogram()` 返回 true 都回 `nullptr`，调用方 `update_histogram()` 抛 `ER_UNABLE_TO_BUILD_HISTOGRAM`（`histogram.cc:1467`）。注意 `Singleton` 分支要求 `num_buckets >= distinct`，**当 distinct 值巨大时永远走 Equi-height**。

参数（已核实）：
- **默认 100 桶，上限 1024**（`sql_yacc.yy:182-188`；`DEFAULT_NUMBER_OF_HISTOGRAM_BUCKETS=100`，`MAX_NUMBER_OF_HISTOGRAM_BUCKETS=1024`）
- `histogram_generation_max_mem_size` 默认 **20000000 字节（20 MB）**，范围 [1000000, SIZE_MAX]，session 级（`sys_vars.cc:3369`）

**采样率由内存预算反推**（`histogram.cc:1443`）：

```cpp
// 在 update_histogram() 内
  /*
    Caclulate how many rows we can fit into memory permitted by
    histogram_generation_max_mem_size.
  */
  double rows_in_memory = thd->variables.histogram_generation_max_mem_size /
                          static_cast<double>(row_size_bytes);   // 能装进内存的行数

  table->fetch_number_of_rows();                                 // ★ 忽略其错误
  ha_rows rows_in_table = std::max(1ULL, tbl->file->stats.records);  // ★ 至少 1，防除零

  double sample_percentage = rows_in_memory / rows_in_table * 100.0;  // 反推采样百分比
  sample_percentage = std::min(sample_percentage, 100.0);             // 上限 100%

  // Read data from the table into the Value_maps we have prepared.
  if (fill_value_maps(resolved_fields, sample_percentage, tbl, value_maps))
    return true; /* purecov: deadcode */
```

即 `sample_percentage = min(内存可容行数 / 表行数, 1.0) × 100%` —— **内存不够时降低采样率，而不是丢弃已采到的值**。采样主循环（`histogram.cc:1089`）：

```cpp
static bool fill_value_maps(
    const std::vector<Field *, Histogram_key_allocator<Field *>> &fields,
    double sample_percentage, const TABLE *table,
    value_map_collection &value_maps) {
  assert(sample_percentage > 0.0);
  assert(sample_percentage <= 100.0);

  std::random_device rd;
  std::uniform_int_distribution<int> dist;
  int sampling_seed = dist(rd);   // ★ 每次 ANALYZE 种子不同

  DBUG_EXECUTE_IF("histogram_force_sampling", {
    sampling_seed = 1;
    sample_percentage = 50.0;     // 测试钩子
  });

  void *scan_ctx = nullptr;

  for (auto &value_map : value_maps)
    value_map.second->set_sampling_rate(sample_percentage / 100.0);  // ★ 写回 Value_map（GEE 用）

  /* This is not a tablesample request. */
  bool tablesample = false;

  if (table->file->ha_sample_init(scan_ctx, sample_percentage, sampling_seed,
                                  enum_sampling_method::SYSTEM, tablesample)) {
    return true;
  }

  auto handler_guard = create_scope_guard([table, scan_ctx]() {
    table->file->ha_sample_end(scan_ctx);
  });

  // Read the data from each column into its own Value_map.
  int res = table->file->ha_sample_next(scan_ctx, table->record[0]);
  while (res == 0) {
    ...   // 按 field_type_to_value_map_type 分派 val_str/val_real/val_int/... 灌入 Value_map
  }
```

采样接口（`sql/handler.h:707` / `sql/handler.cc:3044`）——注意**常量名是 `enum_sampling_method::SYSTEM`**（没有 `TABLE_SAMPLE`/`Sampling_strategy`）：

```cpp
enum class enum_sampling_method { SYSTEM, NONE };   // sql/handler.h:707
```

```cpp
int handler::ha_sample_init(void *&scan_ctx, double sampling_percentage,
                            int sampling_seed,
                            enum_sampling_method sampling_method,
                            const bool tablesample) {
  assert(sampling_percentage >= 0.0);
  assert(sampling_percentage <= 100.0);
  assert(inited == NONE);
  m_random_number_engine.seed(sampling_seed);   // ★ 随机数引擎播种
  m_sampling_percentage = sampling_percentage;
  int result = sample_init(scan_ctx, sampling_percentage, sampling_seed,
                           sampling_method, tablesample);   // 转调引擎实现
  inited = (result != 0) ? NONE : SAMPLING;
  return result;
}

// 默认实现（InnoDB 会覆盖成页级 SYSTEM 采样）
int handler::sample_init(void *&scan_ctx [[maybe_unused]], double, int,
                         enum_sampling_method, const bool) {
  return rnd_init(true);
}

int handler::sample_next(void *scan_ctx [[maybe_unused]], uchar *buf) {
  int res = rnd_next(buf);
  std::uniform_real_distribution<double> rnd(0.0, 1.0);
  while (!res && rnd(m_random_number_engine) > (m_sampling_percentage / 100.0))
    res = rnd_next(buf);   // ★ 默认引擎实现是"逐行按概率筛"
  return res;
}
```

**值 → 频次** 与字符串截断（`sql/histograms/value_map.cc:90`）：

```cpp
template <class T>
bool Value_map<T>::add_values(const T &value, const ha_rows count) {
  try {
    auto res = m_value_map.emplace(value, count);   // ★ key=值，value=绝对频次
    if (!res.second) res.first->second += count;    // 已存在则累加
  } catch (const std::bad_alloc &) {
    // Out of memory.
    return true;   // ★ OOM 时直接失败，让整个 ANALYZE 报错（不静默丢值）
  }
  return false;
}

template <>
bool Value_map<String>::add_values(const String &value, const ha_rows count) {
  /*
    We only consider the substring. That is, if the strings differs after
    character number HISTOGRAM_MAX_COMPARE_LENGTH, they will be considered
    equal. */
  // ★ 长字符串只比前 HISTOGRAM_MAX_COMPARE_LENGTH 个字符：不同后缀会被当作同一值
  String substring = value.substr(0, HISTOGRAM_MAX_COMPARE_LENGTH);
  auto found = m_value_map.find(substring);
  if (found == m_value_map.end()) {
    try {
      char *string_data = substring.dup(&m_mem_root);   // 拷贝进本 Value_map 的 MEM_ROOT
      if (string_data == nullptr) return true;
      String string_dup(string_data, substring.length(), substring.charset());
      m_value_map.emplace(string_dup, count);
    } catch (const std::bad_alloc &) {
      return true;
    }
  } else {
    found->second += count;
  }
  return false;
}
```

`element_overhead()`（`value_map.h:314`）—— 内存预算里"每行多大"的估算口径：

```cpp
  size_t element_overhead() const override {
    // TODO: Replace this with sizeof(value_map_type::node_type) when we have
    // full C++17 support.
    return sizeof(typename value_map_type::value_type) +
           sizeof(typename value_map_type::key_type) + 32;   // ★ 每元素 = value + key + 32B
  }
```

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
bool Histogram::get_selectivity(Item **items, size_t item_count,
                                enum_operator op, double *selectivity) const {
  if (get_raw_selectivity(items, item_count, op, selectivity)) return true;

  /*
    We return a selectivity of at least 0.001 in order to avoid returning very
    low estimates in the following cases:

    1) We miss a value or underestimate its frequency during sampling. With our
       current histogram format this causes "holes" between buckets where we
       estimate a selectivity of zero.

    2) We miss a range of values. With our format we are particularly vulnerable
       around the min and max of the distribution as the sampled min is likely
       greater than the true min and the sampled max likely smaller than the
       true max.

    3) Within-bucket heuristics produce very low estimates. This can for example
       happen for range-queries within a bucket. Another example is if we have
       many infrequent values and one highly frequent value in a bucket.

    4) The histogram has gone stale. While the usual assumption is that the
       value distribution remains nearly constant this assumption fails in some
       common use cases. Consider for example a date column where the current
       date is inserted.

    The reason for the choice of 0.001 for the lower bound is that we typically
    sample fewer than 1000 pages with the default settings. With a sample of
    1000 pages the probablity of missing a value or range of values with a
    selectivity of 0.001 is around 1/e (~0.368) as the size of the table goes to
    infinity in the worst case when the values of interest are concentrated on
    few pages.

    The cost of using a minimum selectivity of 0.001 is that we may sometimes
    over-estimate the selectivity. For very large tables 0.1% of the rows is
    still a lot in absolute terms -- 1000 rows for a table with 1 million rows,
    and 1 million rows for a table with 1 billion rows.

    We could improve this estimate by considering the actual number of pages
    sampled when the histogram was constructed.
  */
  const double minimum_selectivity = 0.001;   // ★ 全局下限：即使桶里说 0，也返回 0.001
  *selectivity = std::max(*selectivity, minimum_selectivity);
  return false;
}
```

四条理由对应四类误差：**采样漏值 / 采样漏区间（尤其 min-max 两端）/ 桶内启发式过小 / 直方图过期**。代价是**可能高估**——对 10 亿行表，0.1% 就是 100 万行。

**`get_raw_selectivity`（`histogram.cc:2047`）——规范化 + 算子合成**：

```cpp
bool Histogram::get_raw_selectivity(Item **items, size_t item_count,
                                    enum_operator op, double *selectivity) const {
  // Do some sanity checking first
  switch (op) {
    case enum_operator::EQUALS_TO:
    case enum_operator::GREATER_THAN:
    case enum_operator::LESS_THAN:
    case enum_operator::LESS_THAN_OR_EQUAL:
    case enum_operator::GREATER_THAN_OR_EQUAL:
    case enum_operator::NOT_EQUALS_TO:
      assert(item_count == 2);
      /*
        Verify that one side of the predicate is a column/field, and that the
        other side is a constant value.
        Make sure that we have the constant item as the right side argument of
        the predicate internally.
      */
      if (items[0]->const_item() && items[1]->type() == Item::FIELD_ITEM) {
        // ★ 常量在左：翻转算子（> 变 <，<= 变 >=），交换参数后**递归回 get_selectivity**
        switch (op) {
          case enum_operator::GREATER_THAN:          op = enum_operator::LESS_THAN; break;
          case enum_operator::LESS_THAN:             op = enum_operator::GREATER_THAN; break;
          case enum_operator::LESS_THAN_OR_EQUAL:    op = enum_operator::GREATER_THAN_OR_EQUAL; break;
          case enum_operator::GREATER_THAN_OR_EQUAL: op = enum_operator::LESS_THAN_OR_EQUAL; break;
          default: break;
        }
        Item *items_flipped[2];
        items_flipped[0] = items[1];
        items_flipped[1] = items[0];
        return get_selectivity(items_flipped, item_count, op, selectivity);
        // ★ 注意递归回**带 0.001 下限**的 get_selectivity，而不是 get_raw_selectivity
      } else if (items[0]->type() != Item::FIELD_ITEM || !items[1]->const_item()) {
        return true;   // ★ 两边都不是"字段 op 常量" ⇒ 让调用方走默认估计
      }
      break;
    case enum_operator::BETWEEN:
    case enum_operator::NOT_BETWEEN:
      assert(item_count == 3);
      if (items[0]->type() != Item::FIELD_ITEM || !items[1]->const_item() ||
          !items[2]->const_item()) return true;
      break;
    case enum_operator::IN_LIST:
    case enum_operator::NOT_IN_LIST:
      assert(item_count >= 2);
      if (items[0]->type() != Item::FIELD_ITEM) return true;
      for (size_t i = 1; i < item_count; ++i)
        if (!items[i]->const_item()) return true;   // ★ IN 列表必须全常量
      break;
    case enum_operator::IS_NULL:
    case enum_operator::IS_NOT_NULL:
      assert(item_count == 1);
      if (items[0]->type() != Item::FIELD_ITEM) return true;
  }

  assert(items[0]->type() == Item::FIELD_ITEM);

  const TYPELIB *typelib = nullptr;
  const Item_field *item_field = down_cast<const Item_field *>(items[0]);
  if (item_field->field->real_type() == MYSQL_TYPE_ENUM ||
      item_field->field->real_type() == MYSQL_TYPE_SET) {
    // ★ ENUM/SET 取出 typelib，供"字符串 → 序号"映射
    const Field_enum *field_enum = down_cast<const Field_enum *>(item_field->field);
    typelib = field_enum->typelib;
  }

  switch (op) {
    case enum_operator::LESS_THAN:
    case enum_operator::EQUALS_TO:
    case enum_operator::GREATER_THAN: {
      return get_selectivity_dispatcher(items[1], op, typelib, selectivity);
    }
    case enum_operator::LESS_THAN_OR_EQUAL: {   // ★ a<=x 用 1 - (a>x) 合成
      double greater_than_selectivity;
      if (get_selectivity_dispatcher(items[1], enum_operator::GREATER_THAN,
                                     typelib, &greater_than_selectivity)) return true;
      *selectivity = std::max(get_non_null_values_fraction() - greater_than_selectivity, 0.0);
      return false;
    }
    case enum_operator::GREATER_THAN_OR_EQUAL: {
      double less_than_selectivity;
      if (get_selectivity_dispatcher(items[1], enum_operator::LESS_THAN,
                                     typelib, &less_than_selectivity)) return true;
      *selectivity = std::max(get_non_null_values_fraction() - less_than_selectivity, 0.0);
      return false;
    }
    case enum_operator::NOT_EQUALS_TO: {
      double equals_to_selectivity;
      if (get_selectivity_dispatcher(items[1], enum_operator::EQUALS_TO,
                                     typelib, &equals_to_selectivity)) return true;
      *selectivity = std::max(get_non_null_values_fraction() - equals_to_selectivity, 0.0);
      return false;
    }
    case enum_operator::BETWEEN: {
      double less_than_selectivity, greater_than_selectivity;
      if (get_selectivity_dispatcher(items[1], enum_operator::LESS_THAN,
                                     typelib, &less_than_selectivity) ||
          get_selectivity_dispatcher(items[2], enum_operator::GREATER_THAN,
                                     typelib, &greater_than_selectivity)) return true;
      *selectivity = this->get_non_null_values_fraction() -
                     (less_than_selectivity + greater_than_selectivity);
      *selectivity = std::max(0.0, *selectivity);   // ★ 防 BETWEEN 3 AND 0 的负值
      return false;
    }
    case enum_operator::NOT_BETWEEN: {
      double less_than_selectivity, greater_than_selectivity;
      if (get_selectivity_dispatcher(items[1], enum_operator::LESS_THAN, typelib,
                                     &less_than_selectivity) ||
          get_selectivity_dispatcher(items[2], enum_operator::GREATER_THAN, typelib,
                                     &greater_than_selectivity)) return true;
      *selectivity = std::min(less_than_selectivity + greater_than_selectivity,
                              get_non_null_values_fraction());
      return false;
    }
    case enum_operator::IN_LIST: {
      *selectivity = 0.0;
      for (size_t i = 1; i < item_count; ++i) {   // ★ 逐值累加（忽略重复值）
        double equals_to_selectivity;
        if (get_selectivity_dispatcher(items[i], enum_operator::EQUALS_TO, typelib,
                                       &equals_to_selectivity)) return true;
        *selectivity += equals_to_selectivity;
        if (*selectivity >= get_non_null_values_fraction()) break;   // 早停
      }
      *selectivity = std::min(*selectivity, get_non_null_values_fraction());
      return false;
    }
    case enum_operator::NOT_IN_LIST: {
      *selectivity = this->get_non_null_values_fraction();
      for (size_t i = 1; i < item_count; ++i) {
        double equals_to_selectivity;
        if (get_selectivity_dispatcher(items[i], enum_operator::EQUALS_TO, typelib,
                                       &equals_to_selectivity)) {
          if (items[i]->null_value) {
            // ★ NOT IN (..., NULL, ...) ⇒ 结果恒为 false ⇒ 选择率 0
            *selectivity = 0.0;
            return false;
          }
          return true;
        }
        *selectivity -= equals_to_selectivity;
        if (*selectivity <= 0.0) break;
      }
      *selectivity = std::max(*selectivity, 0.0);
      return false;
    }
    case enum_operator::IS_NULL:
      *selectivity = this->get_null_values_fraction();
      return false;
    case enum_operator::IS_NOT_NULL:
      *selectivity = 1.0 - this->get_null_values_fraction();
      return false;
  }
  ...
}
```

**类型派发** `get_selectivity_dispatcher()`（`histogram.cc:1876`）：按 `get_data_type()` 分派——STRING 先比对字符集（不一致直接失败）并 `substr(0, HISTOGRAM_MAX_COMPARE_LENGTH)`；INT/ENUM/SET 走 `val_int`；ENUM/SET 若 `data_type()==MYSQL_TYPE_VARCHAR` 先 `find_type2`/`find_set` 转序号（且**只支持 EQUALS_TO**）；UINT/DOUBLE/DECIMAL/DATE/TIME/DATETIME 各自取值后进 `apply_operator()`。

**Singleton（精确查表，无插值）** —— `singleton.cc:407`：

```cpp
template <class T>
double Singleton<T>::get_equal_to_selectivity(const T &value) const {
  // Find the first histogram bucket where the value is not less than the value.
  const auto found = std::lower_bound(m_buckets.begin(), m_buckets.end(), value,
                                      Histogram_comparator());
  if (found == m_buckets.end()) return 0.0;
  if (Histogram_comparator()(value, found->value) == 0) {
    if (found == m_buckets.begin())
      return found->cumulative_frequency;   // 首桶：累计频率即自身频率
    else {
      const auto previous = std::prev(found, 1);
      return found->cumulative_frequency - previous->cumulative_frequency;  // ★ 差分
    }
  }
  return 0.0;   // ★ 值不在表中 ⇒ 0.0（精确语义），随后被 0.001 下限兜住
}

template <class T>
double Singleton<T>::get_less_than_selectivity(const T &value) const {
  const auto found = std::lower_bound(m_buckets.begin(), m_buckets.end(), value,
                                      Histogram_comparator());
  if (found == m_buckets.begin())
    return 0.0;
  else {
    const auto previous = std::prev(found, 1);
    return previous->cumulative_frequency;   // ★ 严格小于：前一桶累计频率
  }
}

template <class T>
double Singleton<T>::get_greater_than_selectivity(const T &value) const {
  const auto found = std::upper_bound(m_buckets.begin(), m_buckets.end(), value,
                                      Histogram_comparator());
  if (found == m_buckets.begin())
    return get_non_null_values_fraction();
  else {
    const auto previous = std::prev(found, 1);
    return get_non_null_values_fraction() - previous->cumulative_frequency;
  }
}
```

**Equi-height（桶内线性插值）** —— `equi_height.cc:702`（桶结构 `equi_height_bucket.h:54` 四成员：`m_lower_inclusive` / `m_upper_inclusive` / `m_cumulative_frequency` / `m_num_distinct`）：

```cpp
template <class T>
double Equi_height<T>::get_equal_to_selectivity(const T &value) const {
  // Find the first bucket where the upper inclusive value is not less than value.
  const auto found = std::lower_bound(m_buckets.begin(), m_buckets.end(), value,
                                      Histogram_comparator());
  if (found == m_buckets.end()) return 0.0;
  if (Histogram_comparator()(value, found->get_lower_inclusive())) return 0.0;
  // ★ value 落在桶下界之下（桶间隙）⇒ 0

  double bucket_frequency;
  if (found == m_buckets.begin()) {
    bucket_frequency = found->get_cumulative_frequency();
  } else {
    const auto previous = std::prev(found, 1);
    bucket_frequency = found->get_cumulative_frequency() -
                       previous->get_cumulative_frequency();   // ★ 桶频率 = 累计差分
  }
  return (bucket_frequency / found->get_num_distinct());
  // ★ = 桶频率 / 桶内估计 distinct 数（桶内均分假设）
}

template <class T>
double Equi_height<T>::get_less_than_selectivity(const T &value) const {
  const auto found = std::lower_bound(m_buckets.begin(), m_buckets.end(), value,
                                      Histogram_comparator());
  if (found == m_buckets.end()) return get_non_null_values_fraction();

  double previous_bucket_cumulative_frequency;
  double found_bucket_frequency;
  if (found == m_buckets.begin()) {
    previous_bucket_cumulative_frequency = 0.0;
    found_bucket_frequency = found->get_cumulative_frequency();
  } else {
    const auto previous = std::prev(found, 1);
    previous_bucket_cumulative_frequency = previous->get_cumulative_frequency();
    found_bucket_frequency = found->get_cumulative_frequency() -
                             previous->get_cumulative_frequency();
  }

  if (Histogram_comparator()(found->get_lower_inclusive(), value)) {
    // ★ a < value <= b：桶内线性插值
    const double distance = found->get_distance_from_lower(value);   // ∈ [0,1]
    return previous_bucket_cumulative_frequency + (found_bucket_frequency * distance);
  } else {
    return previous_bucket_cumulative_frequency;   // ★ value <= a ⇒ 本桶贡献 0
  }
}

template <class T>
double Equi_height<T>::get_greater_than_selectivity(const T &value) const {
  const auto found = std::upper_bound(m_buckets.begin(), m_buckets.end(), value,
                                      Histogram_comparator());
  if (found == m_buckets.end()) return 0.0;

  double found_bucket_frequency;
  if (found == m_buckets.begin()) {
    found_bucket_frequency = found->get_cumulative_frequency();
  } else {
    const auto previous = std::prev(found, 1);
    found_bucket_frequency = found->get_cumulative_frequency() -
                             previous->get_cumulative_frequency();
  }
  double next_buckets_frequency =
      get_non_null_values_fraction() - found->get_cumulative_frequency();  // 本桶之后的全部

  if (Histogram_comparator()(value, found->get_lower_inclusive())) {
    // ★ value < a ⇒ 整桶都大于 value
    return found_bucket_frequency + next_buckets_frequency;
  } else {
    const double distance = found->get_distance_from_upper(value);   // a <= value < b
    return distance * found_bucket_frequency + next_buckets_frequency;
  }
}
```

**桶内插值算式**（`equi_height_bucket.cc`）：
- 整型（主模板 / longlong / ulonglong）：`(value-lower)/(upper-lower+1.0)`（`:203`）
- double：`(value-lower)/(upper-lower)`（`:210`）
- String：`my_strnxfrm` 做字节序变换 + 去公共前缀后拼成 64bit 整数再插值（`:287-346`）
- MYSQL_TIME：折成秒+微秒差比值（`:349-397`）
- my_decimal：转 double 后比值（`:400-421`）
- `get_distance_from_upper` 主模板 = `1.0 - get_distance_from_lower`；longlong/ulonglong 另写 `(upper-value)/(upper-lower+1.0)`（`:435-452`）

**桶内 NDV 的 GEE 修正**（`equi_height.cc:307`，函数名是 `EstimateDistinctValues`，不是成员 `get_estimated_distinct_values`）：

```cpp
static ha_rows EstimateDistinctValues(double sampling_rate,
                                      ha_rows bucket_distinct_values,
                                      ha_rows bucket_unary_values) {
  // Singleton buckets can only contain one distinct value.
  if (bucket_distinct_values == 1) return 1;

  // GEE estimate for non-singleton buckets.
  assert(sampling_rate > 0.0);
  assert(bucket_distinct_values >= bucket_unary_values);
  return static_cast<ha_rows>(
             std::round(std::sqrt(1.0 / sampling_rate) * bucket_unary_values)) +  // ★ sqrt(1/s)*u
         bucket_distinct_values - bucket_unary_values;                            // ★ + d - u
}
```

`d = bucket_distinct_values`（样本内去重值数）、`u = bucket_unary_values`（样本内只出现一次的值数）、`s = sampling_rate`。高频值 `(d-u)` 不放大，低频值 `u` 按 `sqrt(1/s)` 放大——**故意比 `1/s` 保守**（低估 NDV → 高估选择率 → 偏安全）。

> ⚠️ GEE 是为**均匀随机采样**设计的，而 MySQL 用**页级采样**；源码注释承认最坏情况下会低估 NDV 近 `1/s` 倍。

**有直方图 vs 无直方图的本质区别**：把"均匀假设"从**整列**细化到**桶内**，从而能捕捉偏斜分布。

### 8.4 读写与生命周期

- **加载**：`read_histograms()`（`sql/sql_base.cc:585`）在 `get_table_share()` 里被调用（`:833`）：逐列 `find_histogram()` 从 DD 取，`histogram->clone(&share->mem_root)` 后存进 `TABLE_SHARE::m_histograms`（key = `column->ordinal_position()-1`，即 `field_index`）。
- **DD 层取直方图**：`find_histogram()`（`histogram.cc:1751`）用 `dd::Column_statistics::create_name()` 拼名，`client->acquire<dd::Column_statistics>()` 后返回 `column_statistics->histogram()`。
- **SHARE 层查**：`TABLE_SHARE::find_histogram()`（`sql/table.cc:3947`）在 `std::map<uint, const Histogram*>` 里按 field_index 查；`m_histograms` 随 SHARE 析构删除（`table.cc:551`）。
- **写盘**：`Histogram::store_histogram()`（`histogram.cc:1581`）加 MDL 写锁 → `acquire_for_modification` → 有则 `set_histogram(this)` + `update`，无则 `create_object<dd::Column_statistics>` + `set_histogram`。
- **JSON 落库**：`dd::Column_statistics_impl::store_attributes()`（`sql/dd/impl/types/column_statistics_impl.cc:132`）：

```cpp
bool Column_statistics_impl::store_attributes(Raw_record *r) {
  Json_object json_object;
  m_histogram->histogram_to_json(&json_object);   // ★ 虚调用，落到 Singleton/Equi_height 的序列化
  Json_wrapper wrapper(&json_object);
  wrapper.set_alias();
  ...
```

- **序列化字段**：基类（`histogram.cc:387`）写 `last-updated`（UTC 微秒）/`histogram-type`/`sampling-rate`/`number-of-buckets-specified`/`null-values`/`collation-id`；子类追加 `data-type` 与 `buckets`。Singleton bucket = `[值, 累计频率]`（`singleton.cc:190`）；Equi-height bucket = `[lower, upper, 累计频率, num_distinct]`（`equi_height_bucket.h:116`）。

### 8.5 优化器调用点（**不在 `opt_statistics.cc`**）

直方图选择率的唯一入口是 `sql/item_cmpfunc.cc:232` 的静态函数 `get_histogram_selectivity()`：

```cpp
static bool get_histogram_selectivity(THD *thd, const Field *field, Item **args,
                                      size_t arg_count,
                                      histograms::enum_operator op,
                                      Item_func *item_func,
                                      const TABLE_SHARE *table_share,
                                      double *selectivity) {
  const histograms::Histogram *histogram =
      table_share->find_histogram(field->field_index());   // ★ 从 TABLE_SHARE 缓存取
  if (histogram != nullptr) {
    if (!histogram->get_selectivity(args, arg_count, op, selectivity)) {
      if (unlikely(thd->opt_trace.is_started()))
        write_histogram_to_trace(thd, item_func, *selectivity);   // ★ 写 optimizer trace
      return false;   // false = 成功用上直方图
    }
  }
  return true;   // true = 无直方图/失败 ⇒ 调用方退回默认估计
}
```

trace 辅助（`item_cmpfunc.cc:156`）：

```cpp
static void write_histogram_to_trace(THD *thd, Item_func *item,
                                     const double selectivity) {
  Opt_trace_object obj(&thd->opt_trace, "histogram_selectivity");
  obj.add("condition", item).add("histogram_selectivity", selectivity);
}
```

**被谁调用**：旧优化器路径经各 `Item_func_*::get_filtering_effect()`（见第 9 章）→ `calculate_condition_filter()`；hypergraph 路径经 `Item::get_filtering_effect()`（`EstimateSelectivity()` 的兜底分支）。全库共 11 处：`item_cmpfunc.cc` 的 2474 / 2524 / 2555 / 2585 / 2623 / 3074 / 4929 / 6050 / 6223 / 7388，以及 `Item_equal` 里的 6890。

---

## 九、条件过滤 condition filtering

### 9.1 calculate_condition_filter（`sql_planner.cc:1243`）完整

**短路条件**：必须 `condition_fanout_filter=on`（**默认 on**，`sys_vars.cc:196-211`）且满足 2a~2g 之一（有 join buffer / 不是最后一张表 / 子查询 / semijoin / ORDER BY|GROUP BY + LIMIT / EXPLAIN / 未开 BIG_SELECTS）。行数 < 1 或该表无谓词 → 直接 1.0。

**三步，信息源优先级递减**：

```
1) range optimizer 的行数估计（quick_rows / records）   ← 最准
2) 索引统计 rec_per_key
3) Guesstimates                                          ← 最差
```

```cpp
// sql/sql_planner.cc:1243
float calculate_condition_filter(const JOIN_TAB *const tab,
                                 const Key_use *const keyuse,
                                 table_map used_tables, double fanout,
                                 bool is_join_buffering, bool write_to_trace,
                                 Opt_trace_object &parent_trace) {
  const THD *thd = tab->join()->thd;
  TABLE *const table = tab->table();
  const table_map remaining_tables =
      ~used_tables & ~tab->table_ref->map() & tab->join()->all_table_map;
  // ★ 短路条件：不满足 2a~2g 任何一条就直接放行 1.0
  if (!(thd->optimizer_switch_flag(
           OPTIMIZER_SWITCH_COND_FANOUT_FILTER) &&  // 1) 开关
       (is_join_buffering ||                         // 2a 有 join buffer
        remaining_tables != 0 ||                     // 2b 不是最后一张表
        tab->join()->query_block->master_query_expression()->outer_query_block() != nullptr || // 2c 子查询
        !tab->join()->query_block->sj_nests.empty() ||   // 2d semijoin
        ((!tab->join()->order.empty() || !tab->join()->group_list.empty()) &&
         tab->join()->query_expression()->select_limit_cnt != HA_POS_ERROR) || // 2e ORDER/GROUP+LIMIT
        thd->lex->is_explain() ||                    // 2f EXPLAIN
        !Overlaps(thd->variables.option_bits, OPTION_BIG_SELECTS)  // 2g 未开 BIG_SELECTS
        )))
    return COND_FILTER_ALLPASS;

  // No filtering is calculated if we expect less than one row to be fetched
  if (fanout < 1.0 || tab->found_records < 1.0 || tab->records() < 1.0)
    return COND_FILTER_ALLPASS;   // ★ 不足一行不估

  /*
    cond_set has the column bit set for each column involved in a predicate.
    If no bits are set, there are no predicates on this table.
  */
  if (bitmap_is_clear_all(&table->cond_set)) return COND_FILTER_ALLPASS;

  assert(bitmap_is_clear_all(&table->tmp_set));   // ★ tmp_set 借用作"已计入字段"集合

  float filter = COND_FILTER_ALLPASS;

  Opt_trace_context *const trace = &tab->join()->thd->opt_trace;
  Opt_trace_disable_I_S disable_trace(trace, !write_to_trace);
  Opt_trace_array filtering_effect_trace(trace, "filtering_effect");

  /*
    ★ 第 1 步：If ref/range access, the condition is already included in the
    record estimate. The fields used by the ref/range access method
    shall not contribute to the filtering estimate.
  */
  if (keyuse) {
    const KEY *key = table->key_info + keyuse->key;
    if (keyuse[0].keypart == FT_KEYPART) {   // 全文索引：整键字段全排除
      for (uint i = 0; i < key->actual_key_parts; i++)
        bitmap_set_bit(&table->tmp_set, key->key_part[i].field->field_index());
    } else {
      const Key_use *curr_ku = keyuse;
      while (curr_ku->table_ref == tab->table_ref &&         // 1)
             curr_ku->key == keyuse->key &&                  // 1)
             curr_ku->keypart_map & keyuse->bound_keyparts)  // 2)
      {
        // ★ ref 访问实际用到的 keypart（bound_keyparts 子集）才算"已被访问方法用掉"
        bitmap_set_bit(&table->tmp_set,
                       key->key_part[curr_ku->keypart].field->field_index());
        curr_ku++;
      }
    }
  } else if (tab->range_scan())
    get_fields_used(tab->range_scan(), &table->tmp_set);

  /*
    Early exit if the only conditions for the table refers to columns
    used by the access method.
  */
  if (bitmap_is_subset(&table->cond_set, &table->tmp_set)) {
    assert(filter == COND_FILTER_ALLPASS);
    goto cleanup;   // ★ 谓词全被访问方法覆盖 ⇒ 无需再叠过滤系数
  }

  /*
    ★ 第 2 步：If the range optimizer has made row estimates for predicates that
    are not used by the chosen access method, the estimate from the
    range optimizer is used as filtering effect for those fields.
  */
  if (!table->quick_keys.is_clear_all()) {
    char buf[MAX_FIELDS / 8];
    my_bitmap_map *bitbuf = static_cast<my_bitmap_map *>(static_cast<void *>(&buf));
    MY_BITMAP fields_current_quick;

    for (uint keyno = 0; keyno < table->s->keys; keyno++) {
      if (table->quick_keys.is_set(keyno)) {   // ★ 该索引有 range 估行
        bitmap_init(&fields_current_quick, bitbuf, table->s->fields);
        const KEY *key = table->key_info + keyno;
        for (uint i = 0; i < table->quick_key_parts[keyno]; i++)
          bitmap_set_bit(&fields_current_quick,
                         key->key_part[i].field->field_index());

        if (bitmap_is_overlapping(&table->tmp_set, &fields_current_quick))
          continue;   // ★ 字段已被计入 ⇒ 跳过，防重复折算

        bitmap_union(&table->tmp_set, &fields_current_quick);

        const float selectivity = static_cast<float>(table->quick_rows[keyno]) /
                                  static_cast<float>(tab->records());
        // ★ range optimizer 的估行 / 表行数
        // Cannot possible access more rows than there are in the table
        filter *= std::min(selectivity, 1.0f);
      }
    }
  }

  /*
    ★ 第 3 步：Filtering effect for predicates that can be gathered from
    the range optimizer is now reflected in 'filter' ...
  */
  if (tab->join()->where_cond &&
      !bitmap_is_subset(&table->cond_set, &table->tmp_set))
  {
    /*
      Get filtering effect for predicates that are not already
      reflected in 'filter'. The below call gets this filtering effect
      based on index statistics and guesstimates.
    */
    filter *= tab->join()->where_cond->get_filtering_effect(   // ★ 递归遍历 WHERE 树
        tab->join()->thd, tab->table_ref->map(), used_tables, &table->tmp_set,
        static_cast<double>(tab->records()));
  }

  /*
    Cost calculations and picking the right join order assumes that a
    positive number of output rows from each joined table.
  */
  filter = max(filter, 1.0f / tab->records());   // ★ 至少 1 行匹配

  /*
    For large tables, the restriction above may still give very small
    numbers when calculating fan-out. ...
  */
  if ((filter * fanout) < 0.05F) filter = 0.05F / static_cast<float>(fanout);
  // ★ fan-out（filter × fanout）再兜底 0.05

cleanup:
  filtering_effect_trace.end();
  parent_trace.add("final_filtering_effect", filter);

  // Clear tmp_set so it can be used elsewhere
  bitmap_clear_all(&table->tmp_set);   // ★ 必须归还，否则污染后续计算
  assert(filter >= 0.0F && filter <= 1.0F);
  return filter;
}
```

⇒ 三个关键反直觉点：① `tmp_set` 是**借用**的，`goto cleanup` 与正常出口都必须清空；② ref/range **已经用过**的字段一律不重复计入（否则同一谓词被折两次）；③ 最终有两级下限（`1/records` 与 `0.05/fanout`）。

### 9.2 各算子的默认系数（已核实，`item.h:98-115`）

| 常量 | 值 | 含义 |
|---|---|---|
| `COND_FILTER_ALLPASS` | **1.0** | 无过滤（未知谓词/已读表） |
| `COND_FILTER_EQUALITY` | **0.1** | `col1 = col2` |
| `COND_FILTER_INEQUALITY` | **0.3333** | `col1 > col2` |
| `COND_FILTER_BETWEEN` | **0.1111** | `BETWEEN` |

**核心折算公式** `Item_field::get_cond_filter_default_probability`（`sql/item.cc:7948`）：

```cpp
float Item_field::get_cond_filter_default_probability(
    double max_distinct_values, float default_filter) const {
  assert(max_distinct_values >= 1.0);

  // Some field types have a limited number of possible values
  switch (field->real_type()) {
    case MYSQL_TYPE_ENUM: {
      // ENUM can only have the values defined in the typelib
      const uint enum_values = static_cast<Field_enum *>(field)->typelib->count;
      max_distinct_values = min(static_cast<double>(enum_values), max_distinct_values);
      break;
    }
    case MYSQL_TYPE_BIT: {
      // BIT(N) can have no more than 2^N distinct values
      const uint bits = static_cast<Field_bit *>(field)->field_length;
      const double combos = pow(2.0, (int)bits);
      max_distinct_values = min(combos, max_distinct_values);
      break;
    }
    default:
      break;
  }
  // ★ filter = max(1/去重数, 默认系数)
  return max(static_cast<float>(1 / max_distinct_values), default_filter);
}
```

**注意 `max_distinct_values` 实参是 `rows_in_table`**——所以 1) 对**小表** `1/records` 会超过 0.1（5 行表 → 0.2，自动放宽）；2) 对 `records >= 10` 的表，`=` 恒为 0.1；3) 唯一索引路径才会真正用到 `1/records` 上限。

`COND_FILTER_*` 宏本体（`sql/item.h:102`；**不存在 `Condition_filter_parameters` 结构体**）：

```cpp
#define COND_FILTER_ALLPASS 1.0f          // 全放行
/// Filtering effect for equalities: col1 = col2
#define COND_FILTER_EQUALITY 0.1f         // 等值默认 0.1
/// Filtering effect for inequalities: col1 > col2
#define COND_FILTER_INEQUALITY 0.3333f    // 不等默认 1/3
/// Filtering effect for between: col1 BETWEEN a AND b
#define COND_FILTER_BETWEEN 0.1111f       // BETWEEN 默认 ~1/9
/**
   Value is out-of-date, will need recalculation.
   This is used by post-greedy-search logic which changes the access method and
  thus makes obsolete the filtering value calculated by best_access_path(). ...
*/
#define COND_FILTER_STALE -1.0f           // ★ 脏值标记（不是估计值！）
/**
   A special subcase of the above:
   - if this is table/index/range scan, and
   - rows_fetched is how many rows we will examine, and ...
*/
#define COND_FILTER_STALE_NO_CONST -2.0f  // ★ 脏值·特例
```

**默认实现与组合算子**：

```cpp
// sql/item.h:1986 —— 基类：不认识的条件一律放行
  virtual float get_filtering_effect(THD *thd [[maybe_unused]],
                                     table_map filter_for_table [[maybe_unused]],
                                     table_map read_tables [[maybe_unused]],
                                     const MY_BITMAP *fields_to_ignore [[maybe_unused]],
                                     double rows_in_table [[maybe_unused]]) {
    // Filtering effect cannot be calculated for a table already read.
    assert((read_tables & filter_for_table) == 0);
    return COND_FILTER_ALLPASS;
  }
```

```cpp
// sql/item.cc:7936 —— 裸列 "WHERE col" ≈ "col <> 0"
float Item_field::get_filtering_effect(THD *, table_map filter_for_table,
                                       table_map,
                                       const MY_BITMAP *fields_to_ignore,
                                       double rows_in_table) {
  if (used_tables() != filter_for_table ||
      bitmap_is_set(fields_to_ignore, field->field_index()))
    return COND_FILTER_ALLPASS;
  return 1.0f - get_cond_filter_default_probability(rows_in_table,
                                                    COND_FILTER_EQUALITY);
}
```

```cpp
// sql/item_func.cc:891 —— 各算子共用的前置判定
const Item_field *Item_func::contributes_to_filter(
    table_map read_tables, table_map filter_for_table,
    const MY_BITMAP *fields_to_ignore) const {
  assert((read_tables & filter_for_table) == 0);
  assert(functype() != MULT_EQUAL_FUNC);   // ★ Item_equal 不走这里（它有自己的实现）
  if ((used_tables() & ~read_tables) != filter_for_table) return nullptr;
  // ★ 必须"只依赖当前这一张表"，否则无法归因
  Item_field *usable_field = nullptr;
  bool found_comparable = (arg_count == 1);
  for (uint i = 0; i < arg_count; i++) {
    const Item::Type arg_type = args[i]->real_item()->type();
    if (arg_type == Item::SUBSELECT_ITEM) {
      if (args[i]->const_for_execution()) { found_comparable = true; continue; }
      ...   // ★ 依赖子查询 ⇒ 无法估计，返回 nullptr
    }
    ...
  }
  ...
}
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

**AND/OR/NOT 组合**（`sql/item_cmpfunc.cc:5921 / 5976 / 326`）：

```cpp
float Item_cond_and::get_filtering_effect(THD *thd, table_map filter_for_table,
                                          table_map read_tables,
                                          const MY_BITMAP *fields_to_ignore,
                                          double rows_in_table) {
  if (!(used_tables() & filter_for_table))
    return COND_FILTER_ALLPASS;  // No conditions below this apply to the table
  float filter = COND_FILTER_ALLPASS;
  List_iterator<Item> it(list);
  Item *item;
  /*
    Calculated as "Conjunction of independent events":
       P(A and B ...) = P(A) * P(B) * ...
  */
  while ((item = it++))
    filter *= item->get_filtering_effect(thd, filter_for_table, read_tables,
                                         fields_to_ignore, rows_in_table);
  return filter;   // ★ 连乘
}

float Item_cond_or::get_filtering_effect(THD *thd, table_map filter_for_table,
                                         table_map read_tables,
                                         const MY_BITMAP *fields_to_ignore,
                                         double rows_in_table) {
  if (!(used_tables() & filter_for_table))
    return COND_FILTER_ALLPASS;
  float filter = 0.0f;   // ★ OR 从 0 起累
  List_iterator<Item> it(list);
  Item *item;
  while ((item = it++)) {
    const float cur_filter = item->get_filtering_effect(
        thd, filter_for_table, read_tables, fields_to_ignore, rows_in_table);
    /*
      Calculated as "Disjunction of independent events":
      P(A or B)  = P(A) + P(B) - P(A) * P(B)
    */
    filter = filter + cur_filter - (filter * cur_filter);   // ★ 容斥
  }
  return filter;
}

// sql/item_cmpfunc.cc:326
float Item_func_not::get_filtering_effect(...) {
  const float filter = args[0]->get_filtering_effect(...);
  return 1.0 - filter;   // ★ NOT 直接取反
}
```

**各算子（直方图优先，失败回退默认系数）**：

```cpp
// sql/item_cmpfunc.cc:7379
float Item_func_eq::get_filtering_effect(THD *thd, table_map filter_for_table,
                                         table_map read_tables,
                                         const MY_BITMAP *fields_to_ignore,
                                         double rows_in_table) {
  const Item_field *fld =
      contributes_to_filter(read_tables, filter_for_table, fields_to_ignore);
  if (!fld) return COND_FILTER_ALLPASS;
  double selectivity;
  if (!get_histogram_selectivity(thd, fld->field, args, arg_count,
                                 histograms::enum_operator::EQUALS_TO, this,
                                 fld->field->table->s, &selectivity))   // ★ 先试直方图
    return static_cast<float>(selectivity);
  return fld->get_cond_filter_default_probability(rows_in_table,
                                                  COND_FILTER_EQUALITY); // ★ 回退 0.1
}
```

`Item_func_ne`（`:2465`）用 `NOT_EQUALS_TO`，回退 `1.0f - get_cond_filter_default_probability(..., COND_FILTER_EQUALITY)`；`Item_func_ge/gt/le/lt`（`:2503/2594/2564/2534`）先特判 `FT_FUNC`（MATCH 复用 BETWEEN 估计），否则直方图对应算子（`GREATER_THAN_OR_EQUAL` / `GREATER_THAN` / `LESS_THAN_OR_EQUAL` / `LESS_THAN`），回退 `COND_FILTER_INEQUALITY`（0.3333）。

```cpp
// sql/item_cmpfunc.cc:6036
float Item_func_isnull::get_filtering_effect(THD *thd, table_map filter_for_table,
                                             table_map read_tables,
                                             const MY_BITMAP *fields_to_ignore,
                                             double rows_in_table) {
  if (cache_used) {   // ★ 常量折叠已缓存结果
    return cached_value ? COND_FILTER_ALLPASS : 0.0f;
  }
  const Item_field *fld =
      contributes_to_filter(read_tables, filter_for_table, fields_to_ignore);
  if (!fld) return COND_FILTER_ALLPASS;
  double selectivity;
  if (!get_histogram_selectivity(thd, fld->field, args, arg_count,
                                 histograms::enum_operator::IS_NULL, this,
                                 fld->field->table->s, &selectivity))
    return static_cast<float>(selectivity);   // ★ = null_values_fraction
  return fld->get_cond_filter_default_probability(rows_in_table,
                                                  COND_FILTER_EQUALITY);
}
```

`Item_func_isnotnull`（`:6215`）用 `IS_NOT_NULL`，回退 `1.0f - ...`。

```cpp
// sql/item_cmpfunc.cc:4817 / 4838 —— IN 的两条路径
float Item_func_in::get_single_col_filtering_effect(
    Item_ident *fieldref, table_map filter_for_table,
    const MY_BITMAP *fields_to_ignore, double rows_in_table) {
  if ((fieldref->used_tables() != filter_for_table) ||   // 1)
      bitmap_is_set(fields_to_ignore,
                    static_cast<Item_field *>(fieldref->real_item())
                        ->field->field_index()))   // 2)
    return COND_FILTER_ALLPASS;
  const Item_field *fld = (Item_field *)fieldref->real_item();
  return fld->get_cond_filter_default_probability(rows_in_table,
                                                  COND_FILTER_EQUALITY);
}

float Item_func_in::get_filtering_effect(THD *thd, table_map filter_for_table,
                                         table_map read_tables,
                                         const MY_BITMAP *fields_to_ignore,
                                         double rows_in_table) {
  assert((read_tables & filter_for_table) == 0);
  if ((used_tables() & ~read_tables) != filter_for_table || dep_subq_in_list)
    return COND_FILTER_ALLPASS;
  /*
    No matter how many row values are input the filtering effect
    shall not be higher than in_max_filter (currently 0.5).
  */
  const float in_max_filter = 0.5f;   // ★ 硬上限：IN 最多认为过滤掉一半
  float filter = COND_FILTER_ALLPASS;
  if (args[0]->type() == Item::ROW_ITEM) {
    ...
    if (single_rowval_filter != COND_FILTER_ALLPASS)
      filter = min((arg_count - 1) * single_rowval_filter, in_max_filter);
      // ★ (#值 × 单值选择率) 封顶 0.5，用加法近似（忽略重复值）
  } else if (args[0]->real_item()->type() == Item::FIELD_ITEM) {
    if (args[0]->type() == FIELD_ITEM) {
      const Item_field *item_field = down_cast<const Item_field *>(args[0]);
      histograms::enum_operator op =
          (negated ? histograms::enum_operator::NOT_IN_LIST
                   : histograms::enum_operator::IN_LIST);
      double selectivity;
      if (!get_histogram_selectivity(thd, item_field->field, args, arg_count,
                                     op, this, item_field->field->table->s,
                                     &selectivity))
        return static_cast<float>(selectivity);   // ★ 直方图能算就用直方图
    }
    Item_ident *fieldref = static_cast<Item_ident *>(args[0]);
    const float tmp_filt = get_single_col_filtering_effect(
        fieldref, filter_for_table, fields_to_ignore, rows_in_table);
    if (tmp_filt != COND_FILTER_ALLPASS)
      filter = min((arg_count - 1) * tmp_filt, in_max_filter);
  }
  if (negated && filter != COND_FILTER_ALLPASS) filter = 1.0f - filter;  // NOT IN 取反
  assert(filter >= 0.0f && filter <= 1.0f);
  return filter;
}
```

```cpp
// sql/item_cmpfunc.cc:6906 —— Item_equal：索引统计 → 直方图 → 0.1 三级回退
float Item_equal::get_filtering_effect(THD *thd, table_map filter_for_table,
                                       table_map read_tables,
                                       const MY_BITMAP *fields_to_ignore,
                                       double rows_in_table) {
  if (!(used_tables() & filter_for_table)) return COND_FILTER_ALLPASS;
  float filter = COND_FILTER_ALLPASS;
  bool found_comparable = false;
  if (m_const_arg != nullptr) found_comparable = true;   // 多重等式带常量
  List_iterator<Item_field> it(fields);
  Item_field *cur_field;
  while ((cur_field = it++)) {
    if (cur_field->used_tables() & read_tables) {
      found_comparable = true;   // 字段来自已读表 ⇒ 可比
    } else if (cur_field->used_tables() == filter_for_table) {
      if (bitmap_is_set(fields_to_ignore, cur_field->field->field_index())) {
        found_comparable = true;
      } else {
        float cur_filter = cur_field->get_cond_filter_default_probability(
            rows_in_table, COND_FILTER_EQUALITY);   // 默认 0.1
        // Use index statistics if available for this field
        if (!cur_field->field->key_start.is_clear_all()) {
          const TABLE *tab = cur_field->field->table;
          for (uint j = 0; j < tab->s->keys; j++) {
            if (cur_field->field->key_start.is_set(j) &&
                tab->key_info[j].has_records_per_key(0)) {
              cur_filter = static_cast<float>(
                  tab->key_info[j].records_per_key(0) / rows_in_table);   // ★ rec_per_key
              break;
            }
          }
          if (cur_filter >= 1.0) cur_filter = 1.0f;
        } else if (m_const_arg != nullptr) {
          /*
            If index statistics is not available, see if we can use any
            available histogram statistics.
          */
          const histograms::Histogram *histogram =
              cur_field->field->table->s->find_histogram(
                  cur_field->field->field_index());
          if (histogram != nullptr) {
            std::array<Item *, 2> items{{cur_field, m_const_arg}};
            double selectivity;
            if (!histogram->get_selectivity(
                    items.data(), items.size(),
                    histograms::enum_operator::EQUALS_TO, &selectivity)) {
              if (unlikely(thd->opt_trace.is_started())) {
                Item_func_eq *eq_func =
                    new (thd->mem_root) Item_func_eq(cur_field, m_const_arg);
                write_histogram_to_trace(thd, eq_func, selectivity);
              }
              cur_filter = static_cast<float>(selectivity);
            }
          }
        }
        filter *= cur_filter;   // ★ 多字段连乘
      }
    }
  }
  return found_comparable ? filter : COND_FILTER_ALLPASS;
}
```

⇒ `Item_equal` **不走** `Item_func::contributes_to_filter()`（`item_func.cc:899` 有 `assert` 拦），因为它要跨字段取最小值而非单一字段。

### 9.3 hypergraph 的 estimate_selectivity

8.0.39 的 `EstimateSelectivity()` 在 **`sql/join_optimizer/estimate_selectivity.cc`**（旧优化器侧已无此函数）。四级优先级：**索引 rec_per_key → 唯一索引 cap → 直方图 → `get_filtering_effect` 兜底**。

```cpp
// sql/join_optimizer/estimate_selectivity.cc:76
static double EstimateFieldSelectivity(Field *field, double *selectivity_cap,
                                       string *trace) {
  const TABLE *table = field->table;
  double selectivity = -1.0;
  for (uint j = 0; j < table->s->keys; j++) {   // ★ 遍历表上所有索引
    KEY *key = &table->key_info[j];
    if (field->key_start.is_set(j)) {           // 该字段是此索引的**首列**
      if (key->has_records_per_key(0)) {
        double field_selectivity =
            static_cast<double>(table->key_info[j].records_per_key(0)) /
            table->file->stats.records;
        // ★ 一级：sel = rec_per_key[0] / stats.records
        selectivity = std::max(selectivity, field_selectivity);
        // ★ 多个候选索引取**最大**（最不 selective）：宁可高估行数
      }
      const bool single_row = Overlaps(actual_key_flags(key), HA_NOSAME) &&
                              key->actual_key_parts == 1;   // 唯一单列索引
      if (single_row) {
        *selectivity_cap = std::min(*selectivity_cap, 1.0 / table->file->stats.records);
        // ★ 二级：全局上限 1/records，保证 EQ_REF 估行与真实一致
      }
    }
  }

  // Look for a histogram if there was no suitable index.
  if (selectivity == -1.0) {
    // ★ 三级：直方图回退
    const histograms::Histogram *const histogram =
        field->table->s->find_histogram(field->field_index());
    if (histogram != nullptr && !empty(*histogram)) {
      /* ... SELECT ... WHERE tab.field=<expression> ... */
      selectivity = histogram->get_non_null_values_fraction() /
                    std::max<double>(1.0, histogram->get_num_distinct_values());
      ...
    }
  }
  /* ... rec_per_key 与 rows_per_table 不同步可能 >1.0，强制回 1.0 ... */
  return std::min(selectivity, 1.0);
}
```

```cpp
// sql/join_optimizer/estimate_selectivity.cc:168
double EstimateSelectivity(THD *thd, Item *condition, string *trace) {
  // If the item is a true constant, we can say immediately whether it passes
  // or filters all rows. (Actually, calling get_filtering_effect() below
  // would crash if used_tables() is zero, which it is for const items.)
  if (condition->const_item()) {
    return (condition->val_int() != 0) ? 1.0 : 0.0;   // ★ 常量：真 1.0 / 假 0.0
  }

  // For field = field (e.g. t1.x = t2.y), we try to use index information ...
  double selectivity_cap = 1.0;
  if (is_function_of_type(condition, Item_func::EQ_FUNC)) {
    Item_func_eq *eq = down_cast<Item_func_eq *>(condition);
    if (eq->source_multiple_equality != nullptr &&
        eq->source_multiple_equality->const_arg() == nullptr) {
      // ... 统一用多重等式信息，保证不同 join 顺序的选择率一致
      condition = eq->source_multiple_equality;
    } else {
      Item *left = eq->arguments()[0];
      Item *right = eq->arguments()[1];
      if (left->type() == Item::FIELD_ITEM &&
          right->type() == Item::FIELD_ITEM) {   // ★ field = field
        double selectivity = -1.0;
        for (Field *field : {down_cast<Item_field *>(left)->field,
                             down_cast<Item_field *>(right)->field}) {
          selectivity = std::max(
              selectivity,
              EstimateFieldSelectivity(field, &selectivity_cap, trace));   // 两边取最大
        }
        if (selectivity >= 0.0) {
          selectivity = std::min(selectivity, selectivity_cap);   // ★ 施加唯一索引上限
          ...
          return selectivity;
        }
      } else if (left->type() == Item::FIELD_ITEM) {
        // field = <anything>：忽略估计值本身，但**仍要取 cap**
        // （否则 EQ_REF 与同条件下的其他 join 顺序估行不一致）
        EstimateFieldSelectivity(down_cast<Item_field *>(left)->field,
                                 &selectivity_cap, trace);
      } else if (right->type() == Item::FIELD_ITEM) {
        EstimateFieldSelectivity(down_cast<Item_field *>(right)->field,
                                 &selectivity_cap, trace);
      }
    }
  }

  // For multi-equalities, we do the same thing. ...
  if (is_function_of_type(condition, Item_func::MULT_EQUAL_FUNC)) {
    Item_equal *equal = down_cast<Item_equal *>(condition);
    // These should have been expanded early, before we get here.
    assert(equal->const_arg() == nullptr);
    double selectivity = -1.0;
    for (Item_field &field : equal->get_fields()) {
      selectivity = std::max(
          selectivity,
          EstimateFieldSelectivity(field.field, &selectivity_cap, trace));
    }
    if (selectivity >= 0.0) {
      selectivity = std::min(selectivity, selectivity_cap);
      ...
      return selectivity;
    }
  }

  // Index information did not help us, so use Item::get_filtering_effect().
  // ...(说明旧优化器 API 与 hypergraph 的适配问题)
  table_map used_tables = condition->used_tables() & ~PSEUDO_TABLE_BITS;
  table_map this_table = IsolateLowestBit(used_tables);   // ★ 伪造"最低位表"当当前表
  MY_BITMAP empty;
  my_bitmap_map bitbuf[bitmap_buffer_size(MAX_FIELDS) / sizeof(my_bitmap_map)];
  bitmap_init(&empty, bitbuf, MAX_FIELDS);
  double selectivity = condition->get_filtering_effect(
      thd, this_table, used_tables & ~this_table,
      /*fields_to_ignore=*/&empty,
      /*rows_in_table=*/1000.0);   // ★★ 硬编码 1000.0 行表！

  selectivity = std::min(selectivity, selectivity_cap);   // ★ 仍施加唯一索引上限
  ...
  return selectivity;
}
```

**为什么是 1000.0**：`rows_in_table` 只被 `get_cond_filter_default_probability()` 用作 `1/max_distinct_values` 的分母；`COND_FILTER_EQUALITY=0.1` 等默认常量远大于 `1/1000`，所以对普通字段结果与表大小无关（恒为默认常量）。⇒ **hypergraph 的兜底路径无法感知真实表大小**，唯一索引的真实上限由独立的 `selectivity_cap` 承担。

**调用方只有 4 处**（都在 `sql/join_optimizer/join_optimizer.cc` 的**建图阶段**，算完缓存进 `CachedPropertiesForPredicate`，代价搜索阶段不再重算）：

| 行号 | 用途 |
|---|---|
| `:5427` | `HavingCondition`：`filter_path.set_num_output_rows(root_path->num_output_rows() * EstimateSelectivity(...))` |
| `:6246` | 构造 `Predicate p`：`p.selectivity = EstimateSelectivity(thd, eq_item, trace)`（含子查询提升出的等值谓词） |
| `:6327` | `CreateJoinEdges()` 对 `edge.expr->equijoin_conditions` 逐个算 |
| `:6353` | 同函数对 `edge.expr->join_conditions`（非等值 join 条件）算 |

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

