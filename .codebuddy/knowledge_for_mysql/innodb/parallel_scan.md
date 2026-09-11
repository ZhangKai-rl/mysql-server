# InnoDB 并行扫描与 SQL 层并行查询

> 澄清一个常见误解：**标准 MySQL 8.0.39 没有 SQL 层并行查询执行器**——普通 `SELECT` 没有并行执行计划、没有并行 join。它有的只是 **InnoDB 内部的并行扫描**（`Parallel_reader`），用途极窄。真正的 SQL 层并行查询（PolarDB/TXSQL/Aurora）是怎么实现的，见第四节。

## 目录

- [一、结论先行](#一结论先行)
- [二、SQL 层：没有并行执行计划](#二sql-层没有并行执行计划)
- [三、InnoDB Parallel_reader 深度剖析](#三innodb-parallel_reader-深度剖析)
- [四、SQL 层并行查询怎么实现（以 PolarDB 为例）](#四sql-层并行查询怎么实现以-polardb-为例)
- [五、标准 MySQL 为什么不做](#五标准-mysql-为什么不做)
- [关键源码位置速查](#关键源码位置速查)

---

## 一、结论先行

| 维度 | 标准 MySQL 8.0.39 |
|------|-------------------|
| SQL 层并行执行计划 | **无**（没有 `ParallelIterator`、并行 join、Exchange 算子） |
| 普通 SELECT 并行加速 | **否** |
| InnoDB 内部并行扫描 | 有，但只用于 `COUNT(*)` / `CHECK TABLE` / 二级引擎装载 |
| 并行扫描开关 | `innodb_parallel_read_threads`（默认 4，最大 256） |

真正的 SQL 层并行查询（并行扫描 + 并行 join + 并行聚合 + Exchange 算子）是 **PolarDB / TXSQL / Aurora** 的私有特性，社区版没有。

---

## 二、SQL 层：没有并行执行计划

全库搜索 `ParallelIterator` / `parallel_scan` / `parallel_query`：
- `sql/` 层**没有任何** `ParallelIterator`、并行 join 迭代器
- `sql/` 里的 `parallel` 绝大多数是**复制 MTS**（`replica_parallel_workers`，`sys_vars.cc:4219`），与查询执行无关
- 唯一与"并行读"相关的接口是 handler 层的 `parallel_scan_*`（`handler.h:4821`），但它不是 SELECT 执行计划，而是给二级引擎的装载通道

结论：普通 `SELECT` 的执行仍是**单线程迭代器模型**（AccessPath → RowIterator，见 [`server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)）。

---

## 三、InnoDB Parallel_reader 深度剖析

> 文件：`storage/innobase/include/row0pread.h`、`storage/innobase/row/row0pread.cc`。

### 3.1 核心数据结构

```cpp
// row0pread.h:105
constexpr static size_t MAX_THREADS{256};          // innodb_parallel_read_threads 上限
constexpr static size_t MAX_RESERVED_THREADS{16};  // 数据加载额外保留额度
constexpr static size_t MAX_TOTAL_THREADS{272};    // 能 spawn 的绝对上限

using F = std::function<dberr_t(const Ctx *)>;     // 逐行回调

// Config（row0pread.h:173）
struct Config {
  const Scan_range m_scan_range;   // [start, end)，全 null = 全表
  dict_index_t *m_index{};         // 要扫的（聚簇）索引
  size_t m_read_level{0};          // 从哪个 B+ 树层读（0=叶子）
  size_t m_partition_id{...max()}; // 分区 ID
};

// Scan_range：左闭右开 [m_start, m_end)，两个裸指针不拥有内存
struct Scan_range { const dtuple_t *m_start{}; const dtuple_t *m_end{}; };
```

**三个线程常量的含义**：
- `MAX_THREADS=256`：`innodb_parallel_read_threads` 变量上限
- `MAX_RESERVED_THREADS=16`：给数据加载（二级引擎）**额外保留**的额度——普通扫描把 256 用满时，数据加载还能再拿 16 个
- `MAX_TOTAL_THREADS=272`：spawn 的绝对上限

### 3.2 核心算法：B+ 树按子树切分

这是整个模块的灵魂，头文件注释（`row0pread.h:60`）总结三件事：

1. 找 B+ 树的**左路径**（scan start）和**右路径**（scan end）
2. 在**某一层**沿兄弟链接从左到右，把**每个子树根**切成一个 scan 单元
3. 子树数比线程多时，把多余的子树标成 `to_be_split`，由先跑完的线程动态再切

#### (1) create_ranges（row0pread.cc:1038）—— 核心递归

```cpp
dberr_t Parallel_reader::Scan_ctx::create_ranges(
    const Scan_range &scan_range, page_no_t page_no, size_t depth,
    const size_t split_level, Ranges &ranges, mtr_t *mtr) {
  // 定位起点：PAGE_CUR_LE 找 start 的"小于等于"位置（左路径落点）
  if (start != nullptr)
    page_cur_search(block, index, start, PAGE_CUR_LE, &page_cursor);
  else { page_cur_set_before_first(block, &page_cursor); page_cur_move_to_next(&page_cursor); }

  // 从左到右遍历该层的 node pointer（每个指向一个子树根）
  while (!page_cur_is_after_last(&page_cursor)) {
    const auto rec = page_cur_get_rec(&page_cursor);
    // 右边界检查：node pointer >= end 就停（右路径终点）
    if (end != nullptr && end->compare(rec, index, offsets) <= 0) break;

    if (at_level > m_config.m_read_level) {
      auto child_page_no = btr_node_ptr_get_child_page_no(rec, offsets);
      if (depth < split_level) {
        // 还没降到目标切分层 → 递归下沉一层
        create_ranges(scan_range, child_page_no, depth + 1, split_level, ranges, mtr);
        page_cur_move_to_next(&page_cursor); continue;
      }
      // 到了目标切分层：从子树根一路降到叶子，取 range 起点
      level_page_cursor = start_range(child_page_no, mtr, start, savepoints);
    } else {
      level_page_cursor = page_cursor;   // 已在读层
    }

    if (!page_rec_is_supremum(...)) create_range(ranges, level_page_cursor, mtr);
    // 子树已建持久游标，释放沿途 S latch
    for (auto &sp : savepoints) mtr->release_block_at_savepoint(...);

    start = nullptr;                    // 只有第一个 range 用原始 start
    page_cur_move_to_next(&page_cursor);  // 下一个子树
  }
}
```

**算法逐段解释**：

1. **定位起点**：在 root 页上 `PAGE_CUR_LE` 找 `start` 的"小于等于"位置（左路径在 root 层的落点），无 start 则从第一条用户记录（跳过 infimum）开始。
2. **从左到右遍历 node pointer**：每条 node pointer 指向一个子树根。
3. **右边界**：`end->compare(rec) <= 0` 说明已越过右路径终点，停止。
4. **分层决定**：`depth < split_level` 时递归下沉；否则 `start_range` 从该子树根一路降到叶子，取 range 左边界。
5. **create_range 建持久游标**：包装成 `Iter` 并接到上一个 range 的右边界，形成**左闭右开区间链**。
6. **立即释放子树 latch**：持久游标已 `store_position`，无需再持有子树路径 latch——这是"切分一次性完成、之后各扫各的"的关键。

#### (2) create_range —— 链成左闭右开区间

```cpp
void Parallel_reader::Scan_ctx::create_range(Ranges &ranges, page_cur_t &leaf_page_cursor, mtr_t *mtr) {
  auto iter = create_persistent_cursor(leaf_page_cursor, mtr);  // 复制记录 + 存位置
  if (!ranges.empty()) {
    ranges.back().second = iter;   // 上一个 range 的"右边界" = 当前 range 的"左边界"
  }
  ranges.push_back(Range(iter, std::make_shared<Iter>()));  // 新空右边界
}
```

最终 `ranges` 形如：

```
[iter0, iter1)  [iter1, iter2)  [iter2, iter3)  ...  [iterN, end/null)
```

每个 Ctx 扫 `[自己的 start, 下一个 Ctx 的 start)`。

`create_persistent_cursor`（:552）关键动作：落在 infimum 上则前进到首条用户记录 → `copy_row` 整行拷贝到自己的 heap（因为后续要释放原页 latch）→ `store_position` 保存位置。

#### (3) 切分粒度控制（SPLIT_THRESHOLD=3）

代码里**没有**"每线程分几个叶子块"的直接参数，粒度由三机制控制：

1. **`split_level` 决定切在哪层**：`add_scan` 传 0（root 层切），`Ctx::split` 传 1（root 下一层切）。
2. **`SPLIT_THRESHOLD=3`**（`row0pread.cc:50`）在 `create_contexts`（:1256）里决定"要不要动态再切"：

```cpp
if (ranges.size() > n) {
  split_point = (ranges.size() / n) * n;   // 子树比线程多：只把多出的"尾巴"标可拆
} else if (m_depth < SPLIT_THRESHOLD) {
  split_point = n;                          // 树浅不拆（小表拆分反而更贵）
}
// 否则 split_point=0：全部标可拆（树深但子树少，解决负载不均）
```

3. **动态再切 `Ctx::split()`**（:148）：把一个大 range 再 `partition(..., 1)` 往下切一层，子 Ctx 入队被别的线程取走。

### 3.3 线程队列与同步

**全局并发限制**（`available_threads`，:103）：

```cpp
size_t Parallel_reader::available_threads(size_t n_required, bool use_reserved) {
  auto active = s_active_threads.fetch_add(n_required, SEQ_CST);  // 先乐观预留
  auto max_threads = use_reserved ? MAX_THREADS + MAX_RESERVED_THREADS : MAX_THREADS;
  if (active < max_threads) { ... return 能给的数; }
  s_active_threads.fetch_sub(n_required, ...);  // 超了全退
  return 0;
}
```

算法：先原子 `fetch_add` 预留，再按 `max - active` 计算真正能给的数，多余退回。返回 0 表示单线程回退。

**worker 主循环**（`worker`，:822）是纯消费者：

```cpp
for (;;) {
  auto ctx = dequeue();          // 从 run queue 取任务
  if (ctx == nullptr) break;
  if (ctx->m_split) {
    ctx->split();                // 需要动态切分：切成子 Ctx 再入队
  } else {
    ctx->traverse();             // 真正扫这个子树
  }
  // m_n_completed == m_ctx_id 判断所有任务完成
}
```

**同步模型**（8.0.39 已改，非早期版本的 `Synthetic_sync`）：
- `m_mutex` + `m_ctxs`：任务队列互斥
- `os_event_t m_event` + `m_sig_count`：`os_event_reset` + `os_event_wait_time_low` 的 reset-then-wait 配对，生产者（spawn 放行 / split 产生新任务 / 完成）`os_event_set` 唤醒
- `m_sync`：`max_threads==0` 时跳过线程创建，当前线程直接跑 worker

**关键：线程间无记录级数据交换**。每个线程从共享 `m_ctxs` 队列独立 `dequeue` 一个 Ctx，用自己的持久游标扫自己的子树区间，结果写自己的分片计数器。唯一共享的是：任务队列、全局原子错误状态、事件、索引 tree latch。

### 3.4 三个实际消费者

**(1) 并行 COUNT**（`row_mysql_parallel_select_count_star`，row0mysql.cc:4377）：

```cpp
using Shards = Counter::Shards<Parallel_reader::MAX_THREADS>;  // 256 个分片原子计数器
Shards n_recs;
Parallel_reader reader(n_threads);
reader.add_scan(trx, config, [&](const Parallel_reader::Ctx *ctx) {
  Counter::inc(n_recs, ctx->thread_id());   // 每个线程累加自己的 shard
  return DB_SUCCESS;
});
reader.run(n_threads);
Counter::for_each(n_recs, [&](auto n) { *n_rows += n; });   // 最后顺序累加
```

`Counter::Shards`（`ut0counter.h:220`）是**按 cache line 对齐的分片原子计数器**，每个线程用 `thread_id` 作 shard id，**无共享竞争**（每线程一个 cache line）。

**(2) 并行 CHECK TABLE**（`parallel_check_table`，row0mysql.cc:4432）：三个分片计数器（行数/重复键/乱序），每线程维护 `prev_tuple` 比较相邻记录顺序；`ctx->m_start`（range 起点）用于**重置 prev_tuple**——并行切分后 range 间不连续，不能拿上个 range 的最后一条和当前 range 第一条比较。

**(3) `Parallel_reader_adapter`**（row0pread-adapter.cc）：给二级引擎批量喂数据。`process_rows` 把 InnoDB rec 转 MySQL 行（`row_sel_store_mysql_rec`）进 2MB 缓冲（`ADAPTER_SEND_BUFFER_SIZE`），缓冲满/新 range 时 `send_batch` 上送 `m_load_fn`。

### 3.5 触发条件（row_scan_index_for_mysql，row0mysql.cc:4586）

```cpp
if (prebuilt->trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&  // ① RC/RR
    prebuilt->select_lock_type == LOCK_NONE &&                    // ② 非锁定读
    index->is_clustered() &&                                      // ③ 聚簇索引
    (check_keys || prebuilt->trx->mysql_n_tables_locked == 0) &&  // ④ check 或无 LOCK TABLES
    !prebuilt->ins_sel_stmt) {                                    // ⑤ 非 INSERT...SELECT
```

**二级索引不支持**（三处证据）：头文件注释明说 "Secondary index scans are not supported"；MVCC 可见性判断里直接 `ut_error`（:505）——因为二级索引记录不含完整事务信息，读视图判定要回表，破坏"每线程独立持久游标 + 独立 MVCC"的模型。

`innodb_parallel_read_threads`（`ha_innodb.cc:1107`）默认 **4**、范围 **[1, 256]**。

---

## 四、SQL 层并行查询怎么实现（以 PolarDB 为例）

> 以下基于阿里云《PolarDB 并行查询深入剖析》（2022/01）。**这是 PolarDB 私有特性，非标准 MySQL 源码**，这里只讲它的设计思路，作为"如果要 SQL 层并行查询该怎么改"的参考。
>
> ⚠️ **术语澄清**：本节的"plan slice"是 PolarDB 的**并行执行计划分片**（一组 worker 并行执行）。MySQL 社区源码里也有个叫 `REF_SLICE`（`REF_SLICE_WIN_wn`）的东西，但那是**物化临时表的编号**（窗口函数/物化多次读中间结果用，见 [`server/query/07_optimize/10_plan_refinement.md`](../server/query/07_optimize/10_plan_refinement.md)），**与并行毫无关系**——两者名字都带 "slice"，含义完全不同，勿混淆。

### 4.1 两步走优化

PolarDB **没有完全重写 MySQL 优化器**，而是"两步走"：

1. 先完成 MySQL 原生的**串行优化**
2. 在串行优化结果之上，再做**并行拆分与并行优化**

原因：MySQL 优化器内部各子步骤耦合深（递归 join ordering、semi-join 策略），不适合完全侵入式改造。

### 4.2 physical plan clone + refix

并行计划不是从零构造，而是从串行计划**克隆 + 修正**：

- **clone**：根据并行计划描述，把串行物理计划的子 slice 结构克隆到各 worker 线程
- **refix**：修改串行计划变成 leader 侧计划——去掉已下推的执行结构，替换成"从 collector table 读 worker 传上来的数据"

### 4.3 Exchange 算子（数据分发）

PQ2.0 引入 **Exchange 算子**，在 plan slice 之间传递中间结果，三种分发方式：

| 方式 | 作用 |
|------|------|
| **Gather** | 多个 worker 的结果收集回 leader |
| **Shuffle / Repartition** | 按 key 重新分发到下一组 worker（如 group by key shuffle） |
| **Broadcast** | 小表广播到各 worker |

实现用 **lock-free shared ring buffer** 做流水线数据传输。Exchange 是**计划片段的切分点**——并行优化后计划空间产生带 Exchange Enforcer 的物理算子树，按代价选最优，用 Enforcer 作切分点构建多个 plan slice。

### 4.4 plan slice 与 worker

执行计划被拆成多个 **plan slice**，每个 slice 由一组 worker 并行完成，slice 间通过 Exchange 传递中间结果。

**并行扫描**：PolarDB 底层是共享存储，不能按物理分片分配，采用**逻辑分区**——在 B+ 树层面切成**远多于 worker 数的细粒度小分片**，worker 用 **round robin** 抢分片，做到"能者多劳"、避免数据倾斜。

**并行 join** 两种：
- **Parallel Hash Join**：build 阶段多 worker 向共享 lock-free hash table 插入，probe 阶段并行搜索
- **Partition Hash Join**：先按 join key shuffle 到 partition，每个 partition 各自构建小 hash table，co-located join

**子查询并行**：相关子查询用 Pushdown Exec（随外层表下推，每 worker 执行次数等比例减少）；非相关用 Pushdown Shared（提前并行物化成临时表，外层并行读）。

### 4.5 并行度（DOP）

- 用户设置 `max_parallel_degree = xxx` 作为**上限**
- 并行优化器基于统计信息和代价决定实际并行度
- **自适应执行**：资源不足时退回串行 / 降低并行度 / 排队等资源

### 4.6 与 Volcano 模型的关系

**不是替代，是扩展**：每个 plan slice 内部仍沿用 MySQL 的迭代器/QEP_TAB 结构执行，slice 之间通过 Exchange 连接。即在 MySQL 原有 Volcano/Iterator 模型之上，增加并行计划拆分、多 worker 克隆、Exchange 分发、多阶段流水线调度。

---

## 五、标准 MySQL 为什么不做

标准 MySQL 8.0.39 的"并行"停在 **InnoDB 内部对全表扫描类操作做并行扫描 + 给二级引擎喂数据**，原因可归纳：

1. **优化器耦合深**：join ordering / semi-join 等递归逻辑复杂，做并行代价模型的侵入成本高（PolarDB 也选择"两步走"而非重写）
2. **执行器是单线程迭代器**：AccessPath → RowIterator 是单线程模型，要加并行得引入 Exchange/plan slice 整套机制
3. **存储引擎并行已有窄场景**：`Parallel_reader` 覆盖了 COUNT/CHECK 这类"最痛"的全表扫描，普通 OLTP 查询的并行收益有限
4. **云厂商承担了这部分**：真正需要并行查询的分析场景，由 PolarDB/Aurora/TXSQL 等云厂商用私有特性覆盖

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/handler.h:4821/4898` | `parallel_scan_init` / `parallel_scan` 接口（给二级引擎） |
| `storage/innobase/include/row0pread.h:105/148/173` | 常量 / `Scan_range` / `Config` |
| `storage/innobase/row/row0pread.cc:103` | `available_threads`（全局并发限制） |
| `storage/innobase/row/row0pread.cc:1038/981/1021` | `create_ranges`（切分核心）/ `start_range` / `create_range` |
| `storage/innobase/row/row0pread.cc:50/1256/148` | `SPLIT_THRESHOLD` / `create_contexts` / `Ctx::split` |
| `storage/innobase/row/row0pread.cc:822` | `worker`（消费者循环） |
| `storage/innobase/row/row0pread.cc:468` | `check_visibility`（MVCC，二级索引 ut_error） |
| `storage/innobase/handler/ha_innodb.cc:1107` | `innodb_parallel_read_threads`（默认 4） |
| `storage/innobase/handler/ha_innodb.cc:16606/18156` | `records`（COUNT）/ `check`（CHECK TABLE） |
| `storage/innobase/row/row0mysql.cc:4377/4432/4586` | 并行 COUNT / CHECK / 触发条件 |
| `storage/innobase/row/row0pread-adapter.cc:39/172/147` | adapter 批大小 / `process_rows` / `send_batch` |
| `storage/innobase/include/ut0counter.h:220` | `Counter::Shards`（分片原子计数器） |

---

## 参考

**内核月报**
- **2019/10《Parallel Index Scans, One is Better Than Two》** —— 二级索引并行范围扫描的可扩展性问题（PolarDB 研究）

**阿里云技术文章**
- **《PolarDB 并行查询深入剖析》（遥凌，2022/01）** —— 两步走优化、physical plan clone、Exchange 算子、plan slice（第四节的直接来源）

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Configuration → `innodb_parallel_read_threads`*

**相关文档**
- 单线程迭代器执行模型见 [`../server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)
- handler 接口见 [`../server/handler.md`](../server/handler.md)
