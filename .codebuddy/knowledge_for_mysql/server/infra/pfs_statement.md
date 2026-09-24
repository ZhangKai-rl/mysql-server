# PFS Statement：语句统计体系

> 本篇讲 Performance Schema 的**语句（statement）统计**：`events_statements_*` 四张表各自是什么、统计字段怎么累加、延迟直方图怎么分桶、`QUERY_SAMPLE_TEXT` 保留哪条 SQL、以及 sys schema 视图与 PFS 原生分位数的语义差异。
>
> **与 [`../query/14_sql_digest.md`](../query/14_sql_digest.md) 的分工**：那篇讲**指纹怎么算**（归一化 / 哈希 / 反解析 / digest 槽管理），本篇讲**统计怎么聚合**（表族 / 统计字段 / 直方图 / 采样 / 汇总维度）。交界处只有 `events_statements_summary_by_digest` 一张表。

## 目录

- [一句话机制](#一句话机制)
- [设计思想与理论基础](#设计思想与理论基础)
- [一、instrumentation 生命周期](#一instrumentation-生命周期)
- [二、四张表：current / history / history_long / summary](#二四张表current--history--history_long--summary)
- [三、统计字段：PFS_statement_stat](#三统计字段pfs_statement_stat)
- [四、延迟直方图：450 桶与 QUANTILE_95/99/999](#四延迟直方图450-桶与-quantile_9599999)
- [五、QUERY_SAMPLE：保留哪一条 SQL](#五query_sample保留哪一条-sql)
- [六、汇总表矩阵：按什么维度聚合](#六汇总表矩阵按什么维度聚合)
- [七、sys schema 的 p95 与 PFS 的 QUANTILE_95 不是一回事](#七sys-schema-的-p95-与-pfs-的-quantile_95-不是一回事)
- [八、为什么查不到数据：三层开关与过滤时机](#八为什么查不到数据三层开关与过滤时机)
  - [8.1 consumer 是全局的还是 per-thread 的](#81-consumer-是全局的还是-per-thread-的两者都是)
  - [8.2 过滤发生在 start 时一次算完](#82--过滤发生在-start-时一次算完这是零开销的关键)
  - [8.3 setup_actors：默认全开，但删掉默认行 = 全禁](#83-setup_actors默认全开但删掉默认行--全禁)
- [九、PFS 表是怎么实现的（引擎层）](#九pfs-表是怎么实现的引擎层)
  - [9.3 get_row_count 返回的是容量上限](#93-get_row_count-返回的是容量上限不是真实行数)
  - [9.4 UNIQUE KEY USING HASH 是装饰](#94-unique-key--using-hash-是装饰实际是-on-扫描)
  - [9.5 TRUNCATE TABLE = 清零内存](#95-truncate-table--清零内存不是删行)
- [十、其它事件表族与统一事件模型](#十其它事件表族与统一事件模型)
- [十一、内存与 sizing](#十一内存与-sizing)
  - [11.2 autosize 的档位不是按内存算](#112--autosize-的档位不是按内存算)
- [十二、与慢日志：两套完全独立的机制](#十二与慢日志两套完全独立的机制)
- [十三、复制线程与后台线程](#十三复制线程与后台线程)
- [十四、已知限制与坑](#十四已知限制与坑)
- [参考](#参考)

---

## 一句话机制

> 每条语句从**网络层收到包的那一刻**开始被 instrument（先注册成 `statement/abstract/new_packet`，随后两次 refine 成具体类型），执行结束时把耗时、行数、错误、临时表等统计**同时累加到多个汇总槽**，并写进一个 **450 桶的指数直方图**——分位数不存储，读表时才从直方图算出来。

---

## 设计思想与理论基础

### 为什么起点在网络层而不是 dispatch_command

这是整个 statement instrumentation 最反直觉的设计。客户端命令的语句**不是在 `dispatch_command()` 里 start 的**，而是在网络层收到新包的回调里：

```cpp
/*
  We need to:
  - start a new STATEMENT event
  - start a new STAGE event, within this statement,
  - start recording SOCKET WAITS events, within this stage.
  The proper order is critical to get events numbered correctly,
  and nested in the proper parent.
*/
thd->m_statement_psi = MYSQL_START_STATEMENT(
    &thd->m_statement_state, stmt_info_new_packet.m_key, thd->db().str,
    thd->db().length, thd->charset(), nullptr);
```

原因直接写在注册注释里——**因为包类型还没解析出来，只能先用一个抽象类型占位**：

```cpp
/*
  When a new packet is received,
  it is instrumented as "statement/abstract/new_packet".
  Based on the packet type found, it later mutates to the
  proper narrow type, for example
  "statement/abstract/query" or "statement/com/ping".
  ...
  the parser, which mutates the statement type to an even more
  narrow classification, for example "statement/sql/select".
*/
stmt_info_new_packet.m_flags = PSI_FLAG_MUTABLE;
```

**这个设计的收益**：连"等待收包"的时间也能被计入语句耗时，而且 socket wait 有了正确的父事件。

### refine：两次收窄

| 阶段 | 事件名 | 位置 |
|---|---|---|
| 收包 | `statement/abstract/new_packet` | 网络层回调 |
| 知道命令类型 | `statement/abstract/query` 或 `statement/com/*` | `dispatch_command()` 入口 |
| 解析后 | `statement/sql/select` 等 | `mysql_execute_command()` |

refine 的实现极简——只换 class：

```cpp
/* mutate EVENTS_STATEMENTS_CURRENT.EVENT_NAME */
pfs->m_class = klass;
```

### 三条并行的统计口径

| 口径 | 含义 | 载体 |
|---|---|---|
| **单次事件** | 这一条语句自己的明细 | `events_statements_current/history/history_long` |
| **按形状聚合** | 同一 SQL 指纹的累计 | `events_statements_summary_by_digest` |
| **按来源聚合** | 同一用户/host/线程/程序 | `events_statements_summary_by_*_by_event_name` |

三者**在语句结束时同时写入**，互不替代。

---

## 一、instrumentation 生命周期

### 1.1 起点与终点

```cpp
locker = PSI_STATEMENT_CALL(get_thread_statement_locker)(state, key, charset, sp_share);
if (likely(locker != nullptr)) {
  PSI_STATEMENT_CALL(start_statement)(locker, db, db_len, src_file, src_line);
}
```

终点：

```cpp
static inline void inline_mysql_end_statement(
    struct PSI_statement_locker *locker, Diagnostics_area *stmt_da) {
#ifdef HAVE_PSI_STAGE_INTERFACE
  PSI_STAGE_CALL(end_stage)();
#endif
  if (likely(locker != nullptr)) {
    PSI_STATEMENT_CALL(end_statement)(locker, stmt_da);
  }
}
```

注意 **先结束 stage 再结束 statement**——这正是 stage/statement 父子链正确的关键。

`do_command()` 出口有硬断言保证配对：

```cpp
out:
  /* The statement instrumentation must be closed in all cases. */
  assert(thd->m_digest == nullptr);
  assert(thd->m_statement_psi == nullptr);
```

### 1.2 嵌套语句：一个显式栈

`PFS_thread` 上就是一个栈，不是单行：

```cpp
/** Size of @c m_events_statements_stack. */
uint m_events_statements_count;
PFS_events_statements *m_statement_stack;
```

栈深上限 = `performance_schema_max_statement_stack`，默认 **10**：

```cpp
#ifndef PFS_STATEMENTS_STACK_SIZE
#define PFS_STATEMENTS_STACK_SIZE 10
#endif
```

超限时不报错，只**丢掉这一层的 EVENT 记录**（但仍参与 digest/summary 聚合），计数器通过状态变量 `Performance_schema_nested_statement_lost` 暴露。

SQL 层每一层嵌套都自己 save/restore（以存储过程为例）：

```cpp
sql_digest_state digest_state;
sql_digest_state *parent_digest = thd->m_digest;
thd->m_digest = &digest_state;
parent_locker = thd->m_statement_psi;
thd->m_statement_psi = MYSQL_START_STATEMENT(...);
err_status = i->execute(thd, &ip);
MYSQL_END_STATEMENT(thd->m_statement_psi, thd->get_stmt_da());
thd->m_statement_psi = parent_locker;
thd->m_digest = parent_digest;
```

同样的模式出现在：触发器、预编译语句、`sp_instr.cc`、`sql_cursor.cc`、事件调度器、`sql_partition.cc` 等。传进去的 `sp_share` 会记到 `OBJECT_TYPE/SCHEMA/OBJECT_NAME`，所以能看出"这条语句属于哪个存储过程"。

### 1.3 父子链：statement / stage / wait / transaction

`pfs_start_statement_vc()` 一次性把四者关系钉死：

```cpp
/* New stages will have this statement as parent */
child_stage->m_nesting_event_id = event_id;
child_stage->m_nesting_event_type = EVENT_TYPE_STATEMENT;

PFS_events_statements *parent_statement = nullptr;
if (pfs_thread->m_events_statements_count > 0) {
  parent_statement = pfs - 1;                    // 栈里上一层
  parent_event = parent_statement->m_event_id;
  parent_level = parent_statement->m_nesting_event_level + 1;
}

if (parent_transaction->m_state == TRANS_STATE_ACTIVE &&
    parent_transaction->m_event_id > parent_event) {
  parent_event = parent_transaction->m_event_id;  // 事务抢走父级
  parent_type = parent_transaction->m_event_type;
}
```

要点：

- 父级取 `pfs - 1`（栈里上一层语句），`NESTING_EVENT_LEVEL = 父 level + 1`
- **如果事务事件更新（event_id 更大且事务 active），事务会抢走父级**
- 反过来，stage 结束时 wait 会重新挂回父语句

`NESTING_EVENT_ID / NESTING_EVENT_TYPE` 两列就来自这些字段。

---

## 二、四张表：current / history / history_long / summary

### 2.1 `events_statements_current` —— 每线程一个**栈**

扫表逻辑是二维游标 `(thread_index, stack_index)`：

```cpp
if (safe_events_statements_count == 0) {
  /* Display the last top level statement, when completed */
  if (m_pos.m_index_2 >= 1) continue;
} else {
  /* Display all pending statements, when in progress */
  if (m_pos.m_index_2 >= safe_events_statements_count) continue;
}
statement = &pfs_thread->m_statement_stack[m_pos.m_index_2];
```

行为：

- 有语句在跑 → 每个线程最多 10 行（当前层 + 各层嵌套）
- 空闲（栈空）→ 只显示 index 0，即**最后一条已完成的顶层语句**

> 这解释了为什么在空闲连接上 `SELECT * FROM events_statements_current` 仍能看到"最后一条 SQL"。

### 2.2 `history` / `history_long` —— 环形缓冲，无 LRU

per-thread history：

```cpp
uint index = thread->m_statements_history_index;
/*
  A concurrent thread executing TRUNCATE TABLE EVENTS_STATEMENTS_CURRENT
  could alter the data that this thread is inserting,
  causing a potential race condition.
  We are not testing for this and insert a possibly empty record,
  to make this thread (the writer) faster.
  This is ok, the readers of m_statements_history will filter this out.
*/
copy_events_statements(&thread->m_statements_history[index], statement);
index++;
if (index >= events_statements_history_per_thread) {
  index = 0;
  thread->m_statements_history_full = true;
}
```

global history_long：

```cpp
uint index = events_statements_history_long_index.m_u32++;
index = index % events_statements_history_long_size;
if (index == 0) events_statements_history_long_full = true;
```

**没有显式淘汰/LRU，纯环形覆盖**（写指针回绕即为 full，覆盖最老的一条）。

容量由两个 `READ_ONLY` 变量控制，默认 `-1`（自动 sizing）：

| 变量 | 范围 | 自动档位（small / medium / large） |
|---|---|---|
| `performance_schema_events_statements_history_size` | -1..1024 | 5 / 10 / 10（**每线程**） |
| `performance_schema_events_statements_history_long_size` | -1..1M | 100 / 1000 / 10000（**全局**） |

写入由两个 consumer 开关控制：

```cpp
if (thread->m_flag_events_statements_history) {
  insert_events_statements_history(thread, pfs);
}
if (thread->m_flag_events_statements_history_long) {
  insert_events_statements_history_long(pfs);
}
```

### 2.3 `events_statements_summary_by_digest` —— 按指纹聚合

聚合 key 与槽管理见 [`../query/14_sql_digest.md`](../query/14_sql_digest.md) 第五章。这里是**唯一一张带 `QUERY_SAMPLE_*` 和 `QUANTILE_*` 的表**。

---

## 三、统计字段：PFS_statement_stat

### 3.1 结构体字段与列名映射

```cpp
struct PFS_statement_stat {
  PFS_single_stat m_timer1_stat;        // COUNT_STAR / SUM/MIN/AVG/MAX_TIMER_WAIT
  ulonglong m_error_count{0};           // SUM_ERRORS
  ulonglong m_warning_count{0};         // SUM_WARNINGS
  ulonglong m_rows_affected{0};         // SUM_ROWS_AFFECTED
  ulonglong m_lock_time{0};             // SUM_LOCK_TIME
  ulonglong m_rows_sent{0};             // SUM_ROWS_SENT
  ulonglong m_rows_examined{0};         // SUM_ROWS_EXAMINED
  ulonglong m_created_tmp_disk_tables{0};   // SUM_CREATED_TMP_DISK_TABLES
  ulonglong m_created_tmp_tables{0};        // SUM_CREATED_TMP_TABLES
  ulonglong m_select_full_join{0};          // SUM_SELECT_FULL_JOIN
  ulonglong m_select_full_range_join{0};    // SUM_SELECT_FULL_RANGE_JOIN
  ulonglong m_select_range{0};              // SUM_SELECT_RANGE
  ulonglong m_select_range_check{0};        // SUM_SELECT_RANGE_CHECK
  ulonglong m_select_scan{0};               // SUM_SELECT_SCAN
  ulonglong m_sort_merge_passes{0};         // SUM_SORT_MERGE_PASSES
  ulonglong m_sort_range{0};                // SUM_SORT_RANGE
  ulonglong m_sort_rows{0};                 // SUM_SORT_ROWS
  ulonglong m_sort_scan{0};                 // SUM_SORT_SCAN
  ulonglong m_no_index_used{0};             // SUM_NO_INDEX_USED
  ulonglong m_no_good_index_used{0};        // SUM_NO_GOOD_INDEX_USED
  ulonglong m_cpu_time{0};                  // SUM_CPU_TIME
  ulonglong m_max_controlled_memory{0};     // MAX_CONTROLLED_MEMORY
  ulonglong m_max_total_memory{0};          // MAX_TOTAL_MEMORY
  ulonglong m_count_secondary{0};           // COUNT_SECONDARY
};
```

单位换算在**读表时**做（存储单位 ≠ 显示单位）：

```cpp
m_lock_time = stat->m_lock_time * MICROSEC_TO_PICOSEC;
m_cpu_time  = stat->m_cpu_time  * NANOSEC_TO_PICOSEC;
```

### 3.2 `AVG_TIMER_WAIT` 不存储

```cpp
struct PFS_single_stat {
  ulonglong m_count;
  ulonglong m_sum;
  ulonglong m_min;
  ulonglong m_max;
  PFS_single_stat() { m_count = 0; m_sum = 0; m_min = ULLONG_MAX; m_max = 0; }

  inline void aggregate_value(ulonglong value) {
    m_count++;
    m_sum += value;
    if (unlikely(m_min > value)) m_min = value;
    if (unlikely(m_max < value)) m_max = value;
  }
};
```

**没有 `m_avg`**——`AVG_TIMER_WAIT` 是读时 `SUM/COUNT` 算的。`MIN` 初始 `ULLONG_MAX`，靠 `has_timed_stats()`（`m_min <= m_max`）判断是否有效。

未开计时时只计数：

```cpp
if (pfs_flags & STATE_FLAG_TIMED) {
  stat->aggregate_value(wait_time);
  stat->m_cpu_time += cpu_time;
} else {
  stat->aggregate_counted();
}
```

内存字段取 **max 而不是 sum**：

```cpp
void aggregate_memory_size(size_t controlled_size, size_t total_size) {
  if (controlled_size > m_max_controlled_memory) m_max_controlled_memory = controlled_size;
  if (total_size > m_max_total_memory) m_max_total_memory = total_size;
}
```

### 3.3 诊断字段是 SQL 层**主动 push** 进来的

`SUM_SELECT_SCAN` / `SUM_NO_INDEX_USED` 这些**不是从 THD 某个成员"读"出来的**，而是执行过程中由 SQL 层调 PSI 接口推进来的，走两套宏：

```cpp
#define INC_STATEMENT_ATTR_BODY(LOCKER, ATTR, VALUE)   /* += count */
#define SET_STATEMENT_ATTR_BODY(LOCKER, ATTR, VALUE)   /* 直接赋 1 */
```

SQL 层调用点都在 `THD` 的状态计数方法里，`status_var` 与 PFS 同步：

```cpp
void THD::inc_status_select_scan() {
  status_var.select_scan_count++;
  PSI_STATEMENT_CALL(inc_statement_select_scan)(m_statement_psi, 1);
}
void THD::set_status_no_index_used() {
  server_status |= SERVER_QUERY_NO_INDEX_USED;
  PSI_STATEMENT_CALL(set_statement_no_index_used)(m_statement_psi);
}
```

语义区别（容易误读）：

| 字段 | 语义 |
|---|---|
| `SUM_SELECT_SCAN` | 发生全表扫描的**语句次数**（一次语句扫两张表仍只 +1） |
| `SUM_NO_INDEX_USED` / `SUM_NO_GOOD_INDEX_USED` | **布尔式**——本语句是否触发过（每次置 1），汇总后是"触发过的语句数" |

这也是 sys schema 用 `IF(SUM_NO_GOOD_INDEX_USED > 0 OR SUM_NO_INDEX_USED > 0, '*', '')` 做 `full_scan` 标记的原因。

### 3.4 ERRORS / WARNINGS / ROWS_AFFECTED 来自 Diagnostics_area

语句结束时按 `Diagnostics_area` 状态 switch：

```cpp
case Diagnostics_area::DA_OK:
  pfs->m_rows_affected = da->affected_rows();
  pfs->m_warning_count = da->last_statement_cond_count();
  break;
case Diagnostics_area::DA_ERROR:
  pfs->m_sql_errno = da->mysql_errno();
  memcpy(pfs->m_sqlstate, da->returned_sqlstate(), SQLSTATE_LENGTH);
  pfs->m_error_count++;
  break;
```

- `SUM_ERRORS` = **出错的语句条数**（每个 DA_ERROR 语句 +1），不是错误总数
- `SUM_WARNINGS` = **警告条数之和**
- `MESSAGE_TEXT / MYSQL_ERRNO / RETURNED_SQLSTATE` **只保存在 EVENT 行**（current/history），不进 digest 汇总

---

## 四、延迟直方图：450 桶与 QUANTILE_95/99/999

### 4.1 分位数是存在的（常见误解）

⚠️ **8.0.39 的 `events_statements_summary_by_digest` 有 `QUANTILE_95` / `QUANTILE_99` / `QUANTILE_999` 三列**。表定义原文：

```cpp
"  QUANTILE_95 BIGINT unsigned not null,\n"
"  QUANTILE_99 BIGINT unsigned not null,\n"
"  QUANTILE_999 BIGINT unsigned not null,\n"
```

另有两张独立直方图表：`events_statements_histogram_by_digest` 与 `events_statements_histogram_global`（含 `BUCKET_NUMBER` / `BUCKET_TIMER_LOW` / `BUCKET_TIMER_HIGH` / `COUNT_BUCKET` / `COUNT_BUCKET_AND_LOWER`）。

### 4.2 450 桶、指数分桶

```cpp
/** Number of buckets used in histograms. */
#define NUMBER_OF_BUCKETS 450

struct PFS_histogram {
  void increment_bucket(uint bucket_index) { m_bucket[bucket_index]++; }
 private:
  std::atomic<ulonglong> m_bucket[NUMBER_OF_BUCKETS];
};
```

桶边界生成规则（注释即设计说明）：

```cpp
/**
  Histogram base bucket timer, in picoseconds.
  Currently defined as 10 micro second.
*/
#define BUCKET_BASE_TIMER (10 * 1000 * 1000)

/**
  Bucket factor.
  histogram_timer[i+1] = BUCKET_BASE_FACTOR * histogram_timer[i]
  The value is chosen so that BUCKET_BASE_FACTOR ^ 50 = 10,
  which corresponds to a 4.7 percent increase for each bucket,
  or a power of 10 increase for 50 buckets.
*/
#define BUCKET_BASE_FACTOR 1.0471285480508996

void PFS_histogram_timers::init() {
  double current_bucket_timer = BUCKET_BASE_TIMER;
  m_bucket_timer[0] = 0;
  for (bucket_index = 1; bucket_index < NUMBER_OF_BUCKETS; bucket_index++) {
    m_bucket_timer[bucket_index] = current_bucket_timer;
    current_bucket_timer *= BUCKET_BASE_FACTOR;
  }
  m_bucket_timer[NUMBER_OF_BUCKETS] = UINT64_MAX;   // 最后一桶 = 无穷
}
```

即：**第 0 桶 = 0，第 1 桶起 10µs，每桶 ×1.0471，50 桶放大 10 倍；450 桶放大约 10⁹ 倍**。

**为什么用指数分桶**：延迟是长尾分布，等宽分桶在 ms~s 区间会失去分辨率。指数分桶保证**相对误差恒定（约 4.7%）**，这正是分位数想要的。

写入路径（**只有 timed 语句才进直方图**）：

```cpp
if (pfs_flags & STATE_FLAG_TIMED) {
  digest_stat->m_stat.aggregate_value(wait_time);
  const ulong bucket_index = normalizer->bucket_index(wait_time);
  digest_stat->m_histogram.increment_bucket(bucket_index);
  global_statements_histogram.increment_bucket(bucket_index);
}
```

即使没有 digest 记录，**全局直方图仍然更新**。

### 4.3 分位数是**读时计算**的

```cpp
ulonglong count_star = 0;
for (index = 0; index < NUMBER_OF_BUCKETS; index++) {
  count_star += histogram->read_bucket(index);
}
const ulonglong count_95  = ((count_star * 95)  + 99)  / 100;
const ulonglong count_99  = ((count_star * 99)  + 99)  / 100;
const ulonglong count_999 = ((count_star * 999) + 999) / 1000;
// ...扫桶找第一个累计 >= 目标的位置
m_row.m_p95 = g_histogram_pico_timers.m_bucket_timer[index_95 + 1];
```

返回的是**该桶的上界**（不是插值）。

⚠️ **性能含义**：每次读 digest 表的每一行都要扫 450 个桶，全表扫是 **O(行数 × 450)**。这是 `SELECT * FROM events_statements_summary_by_digest` 较慢的主要原因之一（另一原因是 `DIGEST_TEXT` 的懒计算还原）。

---

## 五、QUERY_SAMPLE：保留哪一条 SQL

### 5.1 三选一策略（不是单纯的"最慢"或"最后"）

```cpp
bool get_sample_query = (digest_stat->m_query_sample_length == 0);  // ① 第一条

if (!get_sample_query) {
  get_sample_query = new_max_wait;                                   // ② 最慢的那条
  if (!get_sample_query) {
    if (pfs_param.m_max_digest_sample_age > 0) {                     // ③ 老化重采
      get_sample_query = (digest_stat->get_sample_age() >
                          pfs_param.m_max_digest_sample_age * 1000000);
    }
  }
}
```

| 条件 | 行为 |
|---|---|
| **① 本 digest 还没有样本** | 无脑采 |
| **② 新语句耗时打破历史最大** | 替换（`new_max_wait` 由 `wait_time > get_sample_timer_wait()` 判定） |
| **③ 样本"过期"** | 用当前这条替换（**不管快慢**），默认 **60 秒** |

所以实际观察到的是：**"最近 60 秒内最后一次执行的样本，或者是历史最慢那条（若它还没老化）"**。

`performance_schema_max_digest_sample_age = 0` 时退化为"只采一次（第一条），之后永不更新"。

> 想要"永远保留最慢那条"就设成 0；想"尽量看到最近 SQL"就调小（如 5 秒）。

### 5.2 一个并发保护细节

```cpp
if (get_sample_query) {
  /* Get exclusive access otherwise abort. */
  if (digest_stat->inc_sample_ref() == 0) {
    ...  memcpy 覆盖 ...
  }
  digest_stat->dec_sample_ref();
}
```

`inc_sample_ref()` 是**乐观独占**：读表线程会持引用，此刻若正被读（`!= 0`）就**放弃本次更新**（注释 "Get exclusive access otherwise abort."），避免读到半截文本。

`QUERY_SAMPLE_TIMER_WAIT` 是**那条样本语句自己的耗时**，`QUERY_SAMPLE_SEEN` 是它被记录时该 digest 的 `m_last_seen`。

### 5.3 为什么只能采样：定长预分配

每个 digest record 只有**一块固定长度、独占的 sample 缓冲区**，init 时一次性切好，之后只是 memcpy 覆盖：

```cpp
const size_t sqltext_size = pfs_max_sqltext * sizeof(char);
statements_digest_query_sample_text_array =
    PFS_MALLOC_ARRAY(&builtin_memory_digest_sample_sqltext, digest_max,
                     sqltext_size, char, MYF(MY_ZEROFILL));
```

内存是 `digest_max × pfs_max_sqltext` 的定长数组（默认档位 ≈ 10000 × 1024 B ≈ 10 MB），**不存在按需增长的路径**。要保留全部 SQL 原文就得变长分配，与 PFS 的定长预分配模型冲突。

### 5.4 `SQL_TEXT` vs `QUERY_SAMPLE_TEXT`

| | `SQL_TEXT` | `QUERY_SAMPLE_TEXT` |
|---|---|---|
| 位置 | `events_statements_current/history/history_long` | `events_statements_summary_by_digest` |
| 粒度 | **每条语句**一份 | **每个 digest** 一份 |
| 内存 | `history_long_size × 1024` + 每线程 `history_size × 1024` + 每线程栈 `10 × 1024` | `digest_max × 1024` |
| 写入者 | `pfs_set_statement_text_vc()` | `pfs_end_statement_vc()` |

两者都受 `performance_schema_max_sql_text_length`（默认 1024，`READ_ONLY`）限制，超了就截断（置 `m_*_truncated`）。

一条长 SQL 会**同时存在于 current / history / history_long 三份缓冲里**（`copy_events_statements()` 里再 memcpy 一次）。

---

## 六、汇总表矩阵：按什么维度聚合

| 表名 | 聚合维度 |
|---|---|
| `events_statements_summary_by_digest` | `SCHEMA_NAME` + `DIGEST` |
| `events_statements_summary_global_by_event_name` | `EVENT_NAME`（全局） |
| `events_statements_summary_by_thread_by_event_name` | `THREAD_ID` + `EVENT_NAME` |
| `events_statements_summary_by_account_by_event_name` | `USER` + `HOST` + `EVENT_NAME` |
| `events_statements_summary_by_user_by_event_name` | `USER` + `EVENT_NAME` |
| `events_statements_summary_by_host_by_event_name` | `HOST` + `EVENT_NAME` |
| `events_statements_summary_by_program` | 存储程序身份（`OBJECT_TYPE`/schema/object） |
| `events_statements_histogram_by_digest` | `SCHEMA_NAME` + `DIGEST` + `BUCKET_NUMBER` |
| `events_statements_histogram_global` | `BUCKET_NUMBER` |

⚠️ 一个关键区别：**`by_thread` / `by_account` / `by_user` / `by_host` 是读时聚合的**，不是预先维护的数组：

```cpp
PFS_connection_statement_visitor visitor(klass);
PFS_connection_iterator::visit_thread(thread, &visitor);
m_row.m_stat.set(m_normalizer, &visitor.m_stat);
```

而 `global_by_event_name` 是常驻数组，`pfs_end_statement_vc()` 直接聚合。

---

## 七、sys schema 的 p95 与 PFS 的 QUANTILE_95 不是一回事

sys 的 `statements_with_runtimes_in_95th_percentile` 是**纯 SQL 视图**，自己算分位数：

```sql
SELECT s2.avg_us avg_us,
       IFNULL(SUM(s1.cnt)/NULLIF((SELECT COUNT(*) FROM
         performance_schema.events_statements_summary_by_digest), 0), 0) percentile
  FROM sys.x$ps_digest_avg_latency_distribution AS s1
  JOIN sys.x$ps_digest_avg_latency_distribution AS s2
    ON s1.avg_us <= s2.avg_us
 GROUP BY s2.avg_us
HAVING ... > 0.95
 ORDER BY percentile LIMIT 1;
```

| | PFS `QUANTILE_95` | sys 的 "95th percentile" |
|---|---|---|
| 对象 | **每次执行的延迟** | 各 digest 的**平均延迟** |
| 语义 | "95% 的请求比这个值快" | "平均延迟排在所有 digest 的前 5%" |
| 来源 | `PFS_histogram` 450 桶 | 自连接累计分布，**O(n²)** |
| 单位 | 皮秒 | 微秒 |

⚠️ **两者数值通常不相等，不要混用。** 8.0.39 上看真实 p95 延迟应直接读 `events_statements_summary_by_digest.QUANTILE_95`。

（sys 的 `statement_analysis` 视图自身**没有用** `QUANTILE_*` 列，是纯"均值/最大值"视角。）

---

## 八、为什么查不到数据：三层开关与过滤时机

**instrument 全开了却查不到数据**是 PFS 最高频的困惑。答案是三层开关串联，而且源码注释把顺序写得极直白：

```cpp
  /*
    IF to collect PFS data.
  */
  if (flag_global_instrumentation) {
    if (klass->m_enabled) {
      /*
        WHERE to collect PFS data.
      */
      if (flag_thread_instrumentation) {
        if (pfs_thread->m_enabled) {
          pfs_flags |= STATE_FLAG_BASE | STATE_FLAG_THREAD;
          if (flag_events_statements_current) {
            pfs_flags |= STATE_FLAG_EVENT;
          }
        }
      }
      /*
        WHAT PFS data to collect.
      */
      if (pfs_flags != 0) {
        if (klass->m_enabled && klass->m_timed) {
          pfs_flags |= STATE_FLAG_TIMED;
        }
        if (flag_statements_digest) { pfs_flags |= STATE_FLAG_DIGEST; }
      }
    }
  }
```

| 层 | 表 | 管什么 | 代码里的形态 |
|---|---|---|---|
| **① IF** | `setup_instruments` | 这个 instrument 采不采、计不计时 | `klass->m_enabled` / `klass->m_timed` |
| **② WHERE** | `setup_consumers` | 采到的数据存到哪 | `flag_events_statements_current`、`flag_statements_digest`、`flag_global/thread_instrumentation` |
| **③ 对谁** | `setup_actors` | 前台线程的 user/host 过滤 | `pfs_thread->m_enabled` |

⚠️ `setup_objects` **不参与** statement 是否采集的判定——它只作用于 table/routine 类对象级 instrument（默认屏蔽 `mysql`/`performance_schema`/`information_schema` 里的表与存储程序）。

### 8.1 consumer 是全局的还是 per-thread 的？——两者都是

`setup_consumers` 每行只是一个**全局 bool 指针**加两个隐藏刷新标志：

```cpp
static row_setup_consumers all_setup_consumers_data[COUNT_SETUP_CONSUMERS] = {
    {{...("events_statements_current")},      &flag_events_statements_current,      false, false},
    {{...("events_statements_history")},      &flag_events_statements_history,      false, true},
    {{...("statements_digest")},              &flag_statements_digest,              false, false}};
```

修改时"改全局 bool + 触发重算"：

```cpp
if (m_row->m_instrument_refresh) { update_instruments_derived_flags(); }
if (m_row->m_thread_refresh)     { update_thread_derived_flags(); }
```

关键区别：

- **`events_statements_current` / `statements_digest` / `events_statements_cpu` 是纯全局 bool**，代码里直接用。
- **`events_statements_history` / `_history_long` 是「全局 flag × 本线程 `m_history`」的与**，结果缓存在 `thread->m_flag_events_statements_history` 里：

```cpp
void PFS_thread::set_history_derived_flags() {
  if (m_history) {
    m_flag_events_statements_history      = flag_events_statements_history;
    m_flag_events_statements_history_long = flag_events_statements_history_long;
  } else {
    m_flag_events_statements_history = false;
    ...
  }
}
```

所以 consumer 打开后，已存在的连接靠 `update_thread_derived_flags()` **立刻重算**（不必重连）。

### 8.2 ★ 过滤发生在 start 时一次算完——这是"零开销"的关键

`pfs_start_statement_vc()` 把决策固化进 `state->m_pfs_flags`；**真正的事件对象只在 `STATE_FLAG_EVENT` 置位时才从栈里取槽并初始化计时器**。若 `pfs_flags == 0`，连计时器都不取、栈都不压。

未启用时调用侧几乎零成本（编译期可整体消失）：

```cpp
#define MYSQL_START_STATEMENT(STATE, K, DB, DB_LEN, CS, SPS) NULL
```
```cpp
locker = PSI_STATEMENT_CALL(get_thread_statement_locker)(...);
if (likely(locker != nullptr)) {
  PSI_STATEMENT_CALL(start_statement)(locker, db, db_len, src_file, src_line);
}
```

end 阶段只做"往哪几张表塞"，用 per-thread 派生 flag。

**refine 阶段还可以中途放弃**：

```cpp
if (unlikely(klass == nullptr) || !klass->m_enabled) {
  /* pop statement stack */
  /* The performance schema gives up. */
  pfs_flags = 0;
}
```

### 8.3 `setup_actors`：默认全开，但删掉默认行 = 全禁

默认行是 `%` 通配：

```cpp
/* Enable all users on all hosts by default */
insert_setup_actor(&any_user, &any_host, &any_role, true, true);
```

匹配是**四级回退**（user+host → user+`%` → `%`+host → `%`+`%`），且**全不命中 = 全禁**：

```cpp
*enabled = false;
*history = false;
```

⚠️ 所以"删掉默认 `%` 行又没加自己的行"等于把所有前台线程禁掉——这是另一个高频坑。

**后台线程豁免 `setup_actors`**：

```cpp
} else {
  /* There is no setting for background threads */
  enabled = true;
  history = true;
}
```

---

## 九、PFS 表是怎么实现的（引擎层）

### 9.1 是一个真 handler，但只实现"游标 + 少量写"

```cpp
pfs_hton->flags = HTON_ALTER_NOT_SUPPORTED | HTON_TEMPORARY_NOT_SUPPORTED |
                  HTON_NO_PARTITION | HTON_NO_BINLOG_ROW_OPT;
```

不支持 ALTER、不支持临时表、不支持分区、**不产生行格式 binlog**。

### 9.2 不是视图也不是真存储：数据常驻内存数组，表只是位置游标

```cpp
int table_esms_by_digest::rnd_next() {
  digest_stat = &statements_digest_stat_array[m_pos.m_index];
  if (digest_stat->m_lock.is_populated()) {
    if (digest_stat->m_first_seen != 0) { return make_row(digest_stat); }
  }
  return HA_ERR_RECORD_DELETED;
}
```

游标直接索引内存数组，没有行物化。`rnd_end()` 是 `delete m_table`——**一次扫描 = 一个游标对象，扫完即销毁**。

### 9.3 `get_row_count()` 返回的是容量上限，不是真实行数

```cpp
ha_rows table_esms_by_digest::get_row_count() { return digest_max; }
ha_rows table_events_statements_current::get_row_count() {
  return global_thread_container.get_row_count() * statement_stack_max;
}
ha_rows table_events_statements_history_long::get_row_count() {
  return events_statements_history_long_size;
}
```

⚠️ 优化器拿到的是**数组槽位数**，空槽在 `rnd_next` 里被跳过。所以 `EXPLAIN` 的 rows 经常远大于实际返回行数——这是 PFS 表特有的估算偏差来源。

### 9.4 `UNIQUE KEY ... USING HASH` 是装饰，实际是 O(n) 扫描

表定义里确实写了 `UNIQUE KEY (SCHEMA_NAME, DIGEST) USING HASH`，但 PFS 侧**没有任何 B-tree/哈希表结构**。真身是"键读取器 + 扫描过滤器"：

```cpp
int table_esms_by_digest::index_next() {
  for (m_pos.set_at(&m_next_pos); m_pos.m_index < digest_max; m_pos.next()) {
    digest_stat = &statements_digest_stat_array[m_pos.m_index];
    if (digest_stat->m_first_seen != 0) {
      if (m_opened_index->match(digest_stat)) {
        if (!make_row(digest_stat)) { ... return 0; }
      }
    }
  }
  return HA_ERR_END_OF_FILE;
}
```

- SQL 层的 `key_info` 是真的（优化器能生成 `index_read`）
- 但执行方式是**从当前位置线性扫描 + `match()` 过滤**，复杂度仍是 O(n)
- 收益：避免对不匹配的行执行 `make_row()`（省掉整行构造、分位数计算、DIGEST_TEXT 还原）

### 9.5 `TRUNCATE TABLE` = 清零内存，不是删行

```cpp
int ha_perfschema::truncate(dd::Table *) { return delete_all_rows(); }
int table_esms_by_digest::delete_all_rows() { reset_esms_by_digest(); return 0; }
```

与普通表的四点区别：

1. **不走行删除**：没有事务、没有 undo/redo，直接 reset 数组
2. **不释放也不缩容**：`digest_max` 不变，`get_row_count()` 仍返回 `digest_max`
3. **不写 binlog**（`HTON_NO_BINLOG_ROW_OPT`）；且**从复制线程执行会被静默忽略**：

```cpp
int ha_perfschema::delete_all_rows() {
  if (!PFS_ENABLED()) { return 0; }
  if (is_executed_by_slave()) { return 0; }   // ← 静默忽略
```

4. **语义随表而变**：summary 表 = 计数器归零并丢弃 QUERY_SAMPLE；current/history 表 = 清空事件

---

## 十、其它事件表族与统一事件模型

### 10.1 统一模型：四类事件共享同一个头部

```cpp
/** An event record. */
struct PFS_events {
  ulonglong m_thread_internal_id;       // THREAD_ID
  ulonglong m_event_id;                 // EVENT_ID（线程内单调递增）
  ulonglong m_end_event_id;             // END_EVENT_ID（0 = 未结束）
  enum_event_type m_event_type;
  ulonglong m_nesting_event_id;         // NESTING_EVENT_ID
  enum_event_type m_nesting_event_type;
  uint m_nesting_event_level;
  PFS_instr_class *m_class;
  ulonglong m_timer_start;
  ulonglong m_timer_end;
};
```

父子链形成 **transaction ⇄ statement → stage → wait** 的树。同一线程内 event_id 单调递增，所以"父 event_id < 子 event_id"是天然偏序——这正是 1.3 里判定"事务是否抢走父级"的依据。

### 10.2 `events_transactions_*`

状态机（与表列 ENUM 一一对应）：

```cpp
enum enum_transaction_state {
  TRANS_STATE_ACTIVE = 1,
  TRANS_STATE_COMMITTED = 2,
  TRANS_STATE_ROLLED_BACK = 3
};
```

GTID / XA 由专门接口回填，**只在 `STATE_FLAG_EVENT` 时生效**（即 `events_transactions_current` consumer 打开时）：

```cpp
void pfs_set_transaction_gtid_v1(...) {
  if (state->m_flags & STATE_FLAG_EVENT) {
    pfs->m_sid = *static_cast<const rpl_sid *>(sid);
    pfs->m_gtid_spec = *static_cast<const Gtid_specification *>(gtid_spec);
  }
}
```

### 10.3 `events_stages_*`：进度是可选特性

```cpp
bool is_progress() const { return m_flags & PSI_FLAG_STAGE_PROGRESS; }
```

只有带该 flag 的 stage 才有 `WORK_COMPLETED` / `WORK_ESTIMATED`，非进度 stage 在表里输出 NULL。

会报告进度的（由调用点反推）：**ALTER TABLE 的拷表阶段**（`mysql_stage_set_work_estimated(psi, from->file->stats.records)`）、**buffer pool dump/load**、**clone**（`clone0monitor` 的 `ESTIMATE_WORK → COMPLETE_WORK`）、Group Replication 的 stage monitor。

> ⚠️ 源码中没有"哪些 stage 有进度"的注册表可枚举，只能由调用点反推。

### 10.4 `events_waits_*`：父级是 wait 栈上一层

```cpp
PFS_events_waits *parent_event = wait - 1;
wait->m_nesting_event_id = parent_event->m_event_id;
wait->m_nesting_event_type = parent_event->m_event_type;
```

IDLE 是明确的例外（源码注释：*"IDLE events are waits, but by definition we know that such waits happen outside of any STAGE and STATEMENT, so they have no parents."*）。

---

## 十一、内存与 sizing

### 11.1 从哪分配、能否观测

所有 PFS 数组走 `PFS_MALLOC_ARRAY`，每个类有统一前缀名，可在 `memory_summary_global_by_event_name` 里看到 `memory/performance_schema/events_statements_summary_by_digest` 这类行。

⚠️ **限制**：内置内存类**只做全局统计，不做 per-session**：

```cpp
klass->m_class.m_flags = PSI_FLAG_ONLY_GLOBAL_STAT;
/* Builtin stats are only global, not counting per session usage. */
```

所以 `memory_summary_by_thread_by_event_name` 里看不到 PFS 自身开销。

另一条观测路径：`SHOW ENGINE PERFORMANCE_SCHEMA STATUS` 逐项输出 `<对象>.size` / `.count` / `.memory` 并累加 `total_memory`。

### 11.2 ★ autosize 的档位**不是按内存算**

三档由**三个 hint 相对出厂默认值的倍数**决定：

```cpp
static PFS_sizing_data *estimate_hints(PFS_global_param *param) {
  if ((param->m_hints.m_max_connections <= MAX_CONNECTIONS_DEFAULT) &&
      (param->m_hints.m_table_definition_cache <= TABLE_DEF_CACHE_DEFAULT) &&
      (param->m_hints.m_table_open_cache <= TABLE_OPEN_CACHE_DEFAULT)) {
    /* The my.cnf used is either unchanged, or lower than factory defaults. */
    return &small_data;
  }
  if (... <= DEFAULT * 2) { return &medium_data; }
  /* Looks like a server in production. */
  return &large_data;
}
```

| 档位 | history（每线程） | history_long（全局） | digest |
|---|---|---|---|
| small | 5 | 100 | 1000 |
| medium | 10 | 1000 | 5000 |
| large | 10 | 10000 | 10000 |

`apply_heuristic()` 只填**用户没显式设过**的项（`if (p->m_xxx_sizing < 0)`）。

---

## 十二、与慢日志：两套完全独立的机制

慢日志走 `log_slow_statement` → `log_slow_do`，PFS 走 `MYSQL_START/END_STATEMENT`，**两条链路无代码交集**：关慢日志不影响 PFS，反之亦然。

判定逻辑（PFS 完全不参与）：

```cpp
bool warn_no_index =
    ((thd->server_status & (SERVER_QUERY_NO_INDEX_USED | SERVER_QUERY_NO_GOOD_INDEX_USED)) &&
     opt_log_queries_not_using_indexes && ...);
bool log_this_query =
    ((thd->server_status & SERVER_QUERY_WAS_SLOW) || warn_no_index) &&
    (thd->get_examined_row_count() >= thd->variables.min_examined_row_limit);
```

| 维度 | 慢日志 | PFS statement |
|---|---|---|
| 触发 | `long_query_time` + `log_queries_not_using_indexes` + `min_examined_row_limit` | instrument + consumer + actor |
| 粒度 | 只记**超阈值**的语句，文本形式 | 记**所有**被采集语句，结构化列 + digest 聚合 |
| 分布 | 无 | 450 桶直方图 + QUANTILE_95/99/999 |

**`log_throttle_queries_not_using_indexes`** 只限流"未用索引"这一类（分钟级窗口，超限打一条 `throttle: N ... suppressed.` 汇总），**不限流普通慢查询**；且 **PFS 侧的 `NO_INDEX_USED` 计数永远完整**（不受影响）。

---

## 十三、复制线程与后台线程

### 13.1 复制 SQL 线程的语句**会**进 PFS，用专用 instrument

applier 不走 `dispatch_command`，而是自己显式开/关 locker：

```cpp
thd->m_statement_psi = MYSQL_START_STATEMENT(
    &thd->m_statement_state, stmt_info_rpl.m_key, ...);
```

注册的 instrument 名与网络侧对称：

```cpp
  /*
    Statements processed from the relay log are initially instrumented as
    "statement/abstract/relay_log". The parser will mutate the statement type to
    a more specific classification, for example "statement/sql/insert".
  */
  stmt_info_rpl.m_name = "relay_log";
  stmt_info_rpl.m_flags = PSI_FLAG_MUTABLE;
```

（网络侧对应的是 `statement/abstract/new_packet`。）

⚠️ **不受 `setup_actors` 约束**——复制线程是后台线程，没有 user/host，走 8.3 那条"后台线程恒开"。所以默认配置下**复制线程恒被采集**（仍受 `setup_instruments` + `setup_consumers` 控制）。

慢日志侧则还要 `log_slow_replica_statements=ON`。

### 13.2 其它

- **事件调度器**：`thread/sql/event_scheduler` / `thread/sql/event_worker`
- **复制线程**：`thread/sql/replica_io`、`replica_sql`、`replica_worker`、`replica_monitor`（`PSI_FLAG_THREAD_SYSTEM`）
- **srv_session**（插件/组件会话）：显式 `MYSQL_START_STATEMENT(stmt_info_new_packet.m_key)` 后走 `dispatch_command`，正常 refine 链

> ⚠️ **clone 没有专用 thread instrument**（源码无证据）——它的可观测性是通过 **stage 进度**（`mysql_stage_set_work_estimated` / `_completed`）实现的。

---

## 十四、已知限制与坑

| 限制 | 说明 |
|---|---|
| **读 digest 表很贵** | 每行要做 450 桶扫描 + `DIGEST_TEXT` 还原，且 `make_row()` 无条件执行 |
| **digest 溢出静默** | 数组满后新 digest 全部并入 `DIGEST=NULL` 那一行；唯一指标是 `Performance_schema_digest_lost` |
| **嵌套超限静默** | 栈深超 10 的嵌套语句不产生 EVENT 记录，只看 `Performance_schema_nested_statement_lost` |
| **history 无 LRU** | 纯环形覆盖，最老的先丢 |
| **非 timed 语句不进直方图** | 只 `aggregate_counted()`，不贡献分位数 |
| **布尔式诊断字段** | `SUM_NO_INDEX_USED` 是"语句数"不是"次数"，不要当计数用 |
| **AVG 不存储** | `AVG_TIMER_WAIT` 读时算；`MIN` 初始 `ULLONG_MAX` 靠 `m_min <= m_max` 判定有效 |
| **`SQL_TEXT` 三份拷贝** | 同一条长 SQL 在 current/history/history_long 各存一份 |

---

## 参考

**源码**
- `storage/perfschema/pfs.cc` —— `pfs_start_statement_vc` / `pfs_end_statement_vc` / `pfs_digest_end_vc`
- `storage/perfschema/pfs_stat.h` —— `PFS_single_stat` / `PFS_statement_stat`
- `storage/perfschema/pfs_histogram.{h,cc}` —— 450 桶直方图
- `storage/perfschema/pfs_instr.h` —— `PFS_thread::m_statement_stack`
- `storage/perfschema/table_esms_by_digest.cc` / `table_esmh_by_digest.cc` —— 汇总表与直方图表
- `sql/sql_parse.cc` / `sql/conn_handler/init_net_server_extension.cc` —— 打点位置
- `sql/sys_vars.cc` —— 全部 `performance_schema_*` 变量定义
- `scripts/sys_schema/views/p_s/` —— sys 视图定义

**相关文档**
- 指纹计算、归一化、digest 槽管理：[`../query/14_sql_digest.md`](../query/14_sql_digest.md)
- PFS 框架总论（instrumentation / AOP / 零开销探针）：[`pfs.md`](pfs.md)
- 变量体系：[`variables.md`](variables.md)
