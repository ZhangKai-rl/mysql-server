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

另：直方图（`ANALYZE TABLE ... UPDATE HISTOGRAM`）是**独立于 InnoDB 统计**的一套机制，存在 `mysql.column_statistics`，不在本文范围。

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

★ 这正好是 [`dd.md`](dd.md#22--options-vs-se_private_data) 里 `options` vs `se_private_data` 分类的一个实例：**统计开关是"引擎理解但由 server 定义"的属性 → 进 `options`**（不是 `se_private_data`，因为它由 SQL 层 `CREATE TABLE` 语法直接指定）。

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

★ 协同点：`records_in_range()` 返回的行数估算优先用**直方图**（若有）精确化，否则用 n_diff 的均匀假设——n_diff 是"兜底"，直方图是"精细"。这也是为什么"统计信息不准"的经典解法之一就是建直方图（另一解法是 `ANALYZE` 提高采样页数）。

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

### 相关参数

| 参数 | 默认 | 作用 |
|---|---|---|
| `innodb_stats_persistent` | ON | 是否启用持久统计 |
| `innodb_stats_auto_recalc` | ON | 是否自动重算（表级 `STATS_AUTO_RECALC` 可覆盖） |
| `innodb_stats_persistent_sample_pages` | 20 | 持久统计的采样页数 |
| `innodb_stats_transient_sample_pages` | 8 | 瞬态统计的采样页数 |
| `innodb_stats_method` | `nulls_equal` | NULL 值在 `n_diff` 里如何计数（`nulls_equal` / `nulls_unequal` / `nulls_ignored`） |
| `innodb_stats_include_delete_marked` | OFF | 采集时是否计入删除标记的记录（**只读备库**上可能需要开） |

★ `innodb_stats_include_delete_marked` 是个容易被忽略的参数：它对应 `srv_stats_include_delete_marked`。在**只读实例**上 purge 可能滞后，开启它能让统计更接近实际——但这属于 TRX_SYS 相关的另一条线索（见 `row0mysql.cc` 对它的判定）。

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
