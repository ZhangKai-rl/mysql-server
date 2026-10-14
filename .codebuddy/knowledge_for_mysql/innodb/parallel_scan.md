# InnoDB 并行扫描与 SQL 层并行查询

> **一句话**：标准 MySQL 8.0.39 **没有 SQL 层并行查询**——普通 `SELECT` 没有并行执行计划、没有并行 join。社区版只有 **InnoDB 内部的并行扫描**（`Parallel_reader`），把一棵 B+ 树按**子树**切成若干区间，交给多个后台线程各扫一段，用途极窄（并行 `COUNT(*)`、`CHECK TABLE`、二级引擎装载、DDL 扫描、直方图统计）。

> **边界**：本篇讲 `Parallel_reader` 的**切分算法、线程模型与消费者**，以及"社区为什么不做 SQL 层并行、别人怎么做"。
> - 单线程迭代器执行模型（社区 SELECT 的实际路径）见 [`../server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)；
> - handler 层的 `parallel_scan_*` 接口见 [`../server/handler.md`](../server/handler.md)；
> - B+ 树搜索与游标体系见 [`../index/btr.md`](../index/btr.md)；
> - InnoDB 内部执行模型（fork/thr/node）见 [`query_graph.md`](query_graph.md)。

## 目录

- [概述](#概述)
  - 是什么
  - 用途
  - 版本演进
- [理论基础](#理论基础)
  - 为什么社区只有"引擎内并行扫描"
  - 为什么按子树切分，而不是按行/按页/按 key 范围
  - 为什么切分必须一次性完成（单 mtr + S 锁）
  - 为什么只支持聚簇索引
  - 为什么需要动态再切
- [核心实现](#核心实现)
  - 数据结构与三个线程常量
  - ★ 切分算法：create_ranges 逐段
  - create_range：左闭右开的区间链
  - 粒度控制：split_level / SPLIT_THRESHOLD / 动态 split
  - 线程队列与同步
  - 三个实际消费者
  - 触发条件与限制
- [他库对比](#他库对比)
  - SQL 层并行查询怎么实现（以 PolarDB 为例）
- [可观测性](#可观测性)
- [Misc](#misc)
  - ★ 本机制里的工程实现技法
  - 失效与退化场景
  - 易混淆点
- [参考](#参考)

---

## 概述

### 是什么

`Parallel_reader`（`storage/innobase/include/row0pread.h` / `row/row0pread.cc`）是 InnoDB 内部的**并行全表/范围扫描框架**：把一棵 B+ 树切成若干**子树区间**（Range），每个区间由一个线程用**自己的持久游标**独立扫描，逐行回调同一个 `std::function`（`F`）。

### 用途

| 场景 | 入口 |
|---|---|
| 并行 `COUNT(*)` | `row_mysql_parallel_select_count_star`（`row0mysql.cc`） |
| 并行 `CHECK TABLE` | `parallel_check_table` |
| 二级引擎（HeatWave 等）批量装载 | `Parallel_reader_adapter`（`row0pread-adapter.cc`） |
| DDL 建二级索引 / 表重建 | `handler0alter.cc` |
| 分区表扫描 | `ha_innopart.cc`（每分区一路 scan，完整剖析见 [`../feat/partitioning.md`](../feat/partitioning.md) §7.3） |
| 直方图统计 | `row0pread-histogram.cc` |
| DDL 扫描 | `ddl0par-scan.cc` |

★ 注意这张表：**没有一项是普通 `SELECT`**。这就是"社区没有 SQL 层并行查询"的直接证据。

### 版本演进

| 版本 | 变化 | 为什么 |
|------|------|--------|
| 8.0.14 | `Parallel_reader` 引入，先服务并行 `CHECK TABLE` 与二级引擎装载 | 云上二级引擎需要高速批量喂数据 |
| 8.0.17+ | 扩展到并行 `COUNT(*)`、DDL 扫描、直方图统计 | 覆盖更多"全表扫描类"操作 |
| 8.0.39（本库基线） | 线程模型已改为 `m_mutex` + `os_event` + `m_sync`（早期版本的 `Synthetic_sync` 已移除）；`MAX_THREADS=256` | 简化同步、支持 `max_threads==0` 的同步回退 |
| 8.4 / 9.x | 本库未跟踪到 `Parallel_reader` 的结构性变化 | —— |

---

## 理论基础

### 为什么社区只有"引擎内并行扫描"

三个约束叠加，决定了 MySQL 的并行只可能做成"引擎内、窄场景"：

| # | 约束 | 说明 |
|---|---|---|
| 1 | **执行器是单线程迭代器模型** | AccessPath → RowIterator 的契约是"单线程、一次一行"（见 [`09_executor_iterator`](../server/query/09_executor_iterator.md)）。要做 SQL 层并行，必须引入 Exchange 算子、plan slice、多 worker 克隆一整套机制——**不是加参数，是换执行模型** |
| 2 | **优化器耦合深** | join ordering 是递归 + 剪枝，semi-join 有五种策略，代价模型与统计紧耦合。侵入式改造成本极高——**连 PolarDB 都选择"先串行优化、再并行拆分"的两步走**，而非重写 |
| 3 | **OLTP 定位** | 短查询（走索引、返回少量行）的并行收益为负：切分、线程唤醒、结果合并的开销超过扫描本身。真正痛的是**全表扫描类**操作，而这正好是 `Parallel_reader` 覆盖的范围 |

★ **结论**：并行不是"没做"，而是**只做在"规则、无状态、可独立切分"的场景**。这类操作的共同特征是：不需要 join、不需要全局顺序、结果可归约（计数、校验、批量上送）——**可归约性**才是能否并行的真正判据。

### 为什么按子树切分，而不是按行 / 按页 / 按 key 范围

切分单位的选择是这个模块最核心的设计决策：

| 候选 | 为什么不行 |
|---|---|
| **按行号均分** | ① 总行数未知（要扫完才知道）；② MVCC 下"可见行数"随 ReadView 变化，切分后各段实际工作量不等；③ 行号到物理位置的映射本身就是 B+ 树搜索 |
| **按 page 号均分** | 页在 B+ 树的逻辑顺序（兄弟链）与物理页号**不一致**——按页号切会破坏顺序扫描的局部性，且跨层页混杂（叶子与非叶子页号交错） |
| **按 key 范围均分** | 键分布倾斜时负载严重不均（如 `WHERE status IN (0,1)` 里 99% 是 0）；且分段点可能落在页中间，无法直接定位 |
| **★ 按子树** | ① **同层子树的大小天然接近**（B+ 树平衡），负载均衡；② 子树**内部物理连续**，顺序扫描局部性好；③ 切分只需遍历**上层少数页**（下层不必读），成本低；④ 子树根就是天然的分界点，持久游标可直接落在上面 |

★ 更深一层：按子树切分让"切分"与"扫描"两个阶段**彻底解耦**——切分阶段只在树的上层走一遍（持有 S latch），扫描阶段各线程在自己的子树上跑（不需要任何 latch 协调）。这是"**一次切分、各自扫描**"模型成立的前提。

### 为什么切分必须一次性完成（单 mtr + S 锁）

`create_ranges` 用**一个 mtr** 递归遍历整棵树，源码注释说得很直白：

```
Do a breadth first traversal of the B+Tree using recursion. We want to
set up the scan ranges in one pass. This guarantees that the tree structure
cannot change while we are creating the scan sub-ranges.

Once we create the persistent cursor (Range) for a sub-tree we can release
the latches on all blocks traversed for that sub-tree.
```

两个要点：

1. **为什么必须一次完成**：range 的边界是"子树根上的持久游标"。若切分期间树发生了分裂/合并，之前确定的边界可能失效（子树被拆走或合并）。所以切分期间要 `index_s_lock()`（索引 S 锁，阻止 SMO）。
2. **为什么可以尽早放 latch**：一个子树的持久游标建好后（`create_range` 内部 `store_position`），该子树的边界就"固化"了——后续扫描靠游标自己的 position 重新定位，**不需要再持有路径上的 latch**。所以代码用 `mtr->release_block_at_savepoint()` 立即释放这条路径上的所有块。

★ 这是"长事务 mtr"里的标准技巧：**用一个 mtr 保证原子性，用 savepoint 尽早缩小 latch 持有范围**。若不做这个释放，一棵 4 层树有几百个子树，单个 mtr 会持有整棵树路径的 S latch——既浪费 buffer pool 的 latch 资源，也会阻塞并发写。

### 为什么只支持聚簇索引

头文件注释明说 "Secondary index scans are not supported"，MVCC 可见性判定处直接 `ut_error`。原因不是"没实现"，而是**模型不允许**：

- 二级索引记录**不含 `trx_id` / `roll_ptr`**，判定一行是否对当前 ReadView 可见，必须**回聚簇索引**取版本链（见 [`../index/transaction.md`](../index/transaction.md)）；
- `Parallel_reader` 的模型是"每个线程持有自己的持久游标 + 独立做 MVCC 判定，线程间零通信"；
- 一旦要回表，就变成"扫二级索引的线程 A 要去读线程 B 正在扫的聚簇子树"——**破坏独立性与局部性**，还可能引发线程间等待。

★ 所以限制的本质是：**二级索引的可见性判定依赖聚簇索引，无法在不破坏"每线程独立"前提下并行**。

### 为什么需要动态再切（to_be_split）

静态切分有个绕不开的问题：**切分阶段不知道每个子树有多少工作量**。同层子树大小接近只是"页数接近"，而实际工作量还取决于：记录数（页内填充率不同）、过滤率（有 WHERE 时）、MVCC 可见行数。

于是设计成两级：

1. **静态切一次**：按 `split_level` 切出 N 个 range；
2. **动态再切**：把可能"过大"的 range 标成 `m_split=true`，先跑完的线程拿到它后调 `Ctx::split()` 再切一层，子 range 入队让别人取。

★ 这本质上是 **work stealing 的简化版**——不做真正的抢占式偷取，而是"把大任务切成小任务放回共享队列"。代价是切分本身要重新持 S 锁遍历一遍子树（`split()` 里 `index_s_lock()` → `partition`），所以只对"确实太大"的 range 做——这就是 `split_point` 的作用。

---

## 核心实现

### 数据结构与三个线程常量

```cpp
// row0pread.h
constexpr static size_t MAX_THREADS{256};          // innodb_parallel_read_threads 上限
constexpr static size_t MAX_RESERVED_THREADS{16};  // 数据加载额外保留额度
constexpr static size_t MAX_TOTAL_THREADS{MAX_THREADS + MAX_RESERVED_THREADS};

using F = std::function<dberr_t(const Ctx *)>;     // 逐行回调

struct Config {
  const Scan_range m_scan_range;   // [start, end)，全 null = 全表
  dict_index_t *m_index{};         // 要扫的（聚簇）索引
  size_t m_read_level{0};          // 从哪个 B+ 树层读（0=叶子）
  size_t m_partition_id{...max()}; // 分区 ID
};

struct Scan_range { const dtuple_t *m_start{}; const dtuple_t *m_end{}; };  // 左闭右开，不拥有内存
```

三个线程常量：

- `MAX_THREADS = 256`：`innodb_parallel_read_threads` 的上限；
- `MAX_RESERVED_THREADS = 16`：**给数据加载（二级引擎）额外保留**的额度——普通扫描把 256 用满时，数据加载仍能再拿 16 个（避免装载通道被并行扫描饿死）；
- `MAX_TOTAL_THREADS = 272`：能 spawn 的绝对上限。

⚠️ **勘误**：
- **核心类就叫 `class Parallel_reader`**（`row0pread.h:103`），不是 `Parallel_reader_adapter`——后者（`row0pread-adapter.h:43`）只是外层封装，内部持有 `Parallel_reader m_parallel_reader;`（`:208`）。
- **不存在 `MIN_THREADS` / `DEFAULT_NUM_THREADS`**——默认 4、最小 1 写死在 sysvar 注册处 `ha_innodb.cc:1107-1112`。
- **`Ctx` / `Scan_ctx` 是 `class`**，`Thread_ctx` 是 `struct`。

**类设计总注释**（`row0pread.h:58`）——这是理解整套机制最权威的一段：

```cpp
/** The core idea is to find the left and right paths down the B+Tree.These
paths correspond to the scan start and scan end search. Follow the links
at the appropriate btree level from the left to right and split the scan
on each of these sub-tree root nodes.

If the user has set the maximum number of threads to use at say 4 threads
and there are 5 sub-trees at the selected level then we will split the 5th
sub-tree dynamically when it is ready for scan.

We want to allow multiple parallel range scans on different indexes at the
same time. To achieve this split out the scan  context (Scan_ctx) from the
execution context (Ctx). The Scan_ctx has the index  and transaction
information and the Ctx keeps track of the cursor for a specific thread
during the scan.

To start a scan we need to instantiate a Parallel_reader. A parallel reader
can contain several Scan_ctx instances and a Scan_ctx can contain several
Ctx instances. Its' the Ctx instances that are eventually executed.

...

Note: The Ctx instances are in a virtual list. Each Ctx instance has a
range to scan. The start point of this range instance is the end point
of the Ctx instance scanning values less than its start point. A Ctx
will scan from [Start, End) rows. We use std::shared_ptr to manage the
reference counting, this allows us to dispose of the Ctx instances
without worrying about dangling pointers.

NOTE: Secondary index scans are not supported currently. */
```

**三层上下文的关系**：

```
Parallel_reader（一个扫描器实例）
  ├─ Scan_ctx #0（一次 add_scan：一个索引/一个分区 = 一份 trx + 回调）
  │    ├─ Ctx（一个 Range = 一个线程一次真正扫的范围）◀── 被 enqueue 进 run queue
  │    ├─ Ctx
  │    └─ Ctx
  └─ Scan_ctx #1（分区表：每个分区一个）
```

> ★★ **"多个 Scan_ctx"是为分区表设计的**——分区是互相独立的索引，一个 `Parallel_reader` 可以一次扫完所有分区，所有 Ctx 共享同一个 run queue 与同一批 worker 线程。

**`Scan_range` 与 `Config` 完整定义**（`row0pread.h:147` / `:172`）：

```cpp
  struct Scan_range {
    Scan_range() : m_start(), m_end() {}       // ★ 全 nullptr == (-inf, +inf) 全表扫描

    /** Start of the scan, can be nullptr for -infinity. */
    const dtuple_t *m_start{};                 // ★ 闭端：包含该 key，PAGE_CUR_GE 定位

    /** End of the scan, can be null for +infinity. */
    const dtuple_t *m_end{};                   // ★ 开端：不包含
  };

  struct Config {
    Config(const Scan_range &scan_range, dict_index_t *index,
           size_t read_level = 0,
           size_t partition_id = std::numeric_limits<size_t>::max())
        : m_scan_range(scan_range),
          m_index(index),
          m_is_compact(dict_table_is_comp(index->table)),
          //  ★ 只用来读 delete flag —— 不是"禁止压缩表"的判断！
          m_page_size(dict_tf_to_fsp_flags(index->table->flags)),
          m_read_level(read_level),
          //  ★ 0 = 叶子层；非 0 表示只读内部节点（直方图采样用）
          m_partition_id(partition_id) {}
    ...
  };
```

**`Scan_ctx::Iter` 与 `Range`——左闭右开的类型级证据**（`row0pread.h:486`）：

```cpp
  /** Boundary of the range to scan. */
  struct Iter {
    mem_heap_t *m_heap{};
    const ulint *m_offsets{};
    const rec_t *m_rec{};              // ★ 起始记录（深拷到 m_heap）
    /** Tuple representation inside m_rec, for two Iter instances in a range
    m_tuple will be [first->m_tuple, second->m_tuple). */
    const dtuple_t *m_tuple{};         // ★★★ 左闭右开的直接证据
    btr_pcur_t *m_pcur{};              // ★ 持久化游标（placement new 在 m_heap 上）
  };

  using Savepoint = std::pair<ulint, buf_block_t *>;   // ★ 用于中途释放某个 block 的 S latch
  using Savepoints = std::vector<Savepoint, ut::allocator<Savepoint>>;

  /** The first cursor should read up to the second cursor [f, s). */
  using Range = std::pair<std::shared_ptr<Iter>, std::shared_ptr<Iter>>;
  using Ranges = std::vector<Range, ut::allocator<Range>>;
  //  ★ 一次 partition 产出的所有相邻 Range，首尾相接
```

**`Thread_ctx`（每 worker 线程一份）**（`row0pread.h:217`）：

```cpp
  struct Thread_ctx {
    size_t m_thread_id{std::numeric_limits<size_t>::max()};
    void *m_callback_ctx{};            // ★ 调用方（adapter）自己的线程私有状态
    mem_heap_t *m_blob_heap{};         // ★★ per-thread BLOB heap
    State m_state{State::UNKNOWN};     // THREAD / CTX / PAGE 三态
    PCursor *m_pcursor{};              // ★ 由 Ctx::traverse() 在栈上创建后挂进来

    void create_blob_heap() noexcept {
      ut_a(m_blob_heap == nullptr);
      m_blob_heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);
    }
    void savepoint() noexcept;                      // 存位置 + commit mtr
    dberr_t restore_from_savepoint() noexcept;
  };
```

**`Ctx`（执行上下文）**（`row0pread.h:684`）：

```cpp
class Parallel_reader::Ctx {
 private:
  size_t m_id{};
  /** If true then split the context at the block level. */
  bool m_split{};                      // ★★ 这就是注释里说的 "to_be_split"，真实成员名
  Scan_ctx::Range m_range{};
  Scan_ctx *m_scan_ctx{};
 public:
  Thread_ctx *m_thread_ctx{};          // ★ worker dequeue 后赋值
  const buf_block_t *m_block{};
  const rec_t *m_rec{};
  size_t m_n_pages{};
  bool m_first_rec{true};
  ulint *m_offsets{};
  /** Start of a new range to scan. */
  bool m_start{};                      // ★ adapter 据此判断"新 range 开始"→ flush 批次

  dberr_t traverse();                  // ★ 一个 Ctx 的扫描主流程
  dberr_t traverse_recs(PCursor *pcursor, mtr_t *mtr);
  bool move_to_next_node(PCursor *pcursor);
  dberr_t split();                     // ★ 动态再切分
};
```

> ⚠️ **不存在 `Ctx::init()` / `Ctx::start()`**——`Ctx` 只有构造/析构 + 上面 4 个方法。
> ⚠️ **`to_be_split` 只出现在注释里**（`row0pread.h:87`），真实符号是 `Ctx::m_split`（`row0pread.h:768`）。
> ⚠️ **`Scan_range` 里没有 `m_heap_no`**（`m_heap_no` 属于 `lock0priv.h:697` 的 `RecID`，与本机制无关）。

**`Scan_ctx` 的索引 S 锁是引用计数式的**（`row0pread.cc:132`）——★ 这是"允许非加锁线程解锁"的关键：

```cpp
void Parallel_reader::Scan_ctx::index_s_lock() {
  if (m_s_locks.fetch_add(1, std::memory_order_acquire) == 0) {
    //  ★ 只有 0→1 的那个线程真正加锁
    auto index = m_config.m_index;
    /* The latch can be unlocked by a thread that didn't originally lock it. */
    rw_lock_s_lock_gen(dict_index_get_lock(index), true, UT_LOCATION_HERE);
    //                                              ★ 第 2 个参数 = 允许由别的线程解锁
  }
}

void Parallel_reader::Scan_ctx::index_s_unlock() {
  if (m_s_locks.fetch_sub(1, std::memory_order_acquire) == 1) {
    auto index = m_config.m_index;
    rw_lock_s_unlock_gen(dict_index_get_lock(index), true);
  }
}
```

> ★★ 因为切分（用户线程）与扫描（worker 线程）可能是**不同线程**，所以必须用引用计数 + "允许他人解锁"的 latch。若用普通 S 锁，worker 线程解锁会断言失败。

### ★ 切分算法：create_ranges 逐段

```cpp
dberr_t Parallel_reader::Scan_ctx::create_ranges(
    const Scan_range &scan_range, page_no_t page_no, size_t depth,
    const size_t split_level, Ranges &ranges, mtr_t *mtr) {
  ut_ad(index_s_own());
  Savepoint savepoint({mtr->get_savepoint(), nullptr});
  auto block = block_get_s_latched(page_id, mtr, __LINE__);   // ① 逐页 S latch
  savepoint.second = block;

  // ② 定位起点：start 的 PAGE_CUR_LE 落点（左路径）
  if (start != nullptr) {
    page_cur_search(block, index, start, PAGE_CUR_LE, &page_cursor);
    if (page_cur_is_after_last(&page_cursor)) return DB_SUCCESS;
    else if (page_cur_is_before_first(&page_cursor)) page_cur_move_to_next(&page_cursor);
  } else {
    page_cur_set_before_first(block, &page_cursor);
    page_cur_move_to_next(&page_cursor);      /* Skip the infimum record. */
  }

  const auto at_leaf  = page_is_leaf(buf_block_get_frame(block));
  const auto at_level = btr_page_get_level(buf_block_get_frame(block));

  // ③ 从左到右遍历该层的 node pointer（每个指向一个子树根）
  while (!page_cur_is_after_last(&page_cursor)) {
    const auto rec = page_cur_get_rec(&page_cursor);

    // ④ 右边界：node pointer >= end 就停
    if (end != nullptr && end->compare(rec, index, offsets) <= 0) break;

    if (at_level > m_config.m_read_level) {                    // ⑤ 还在目标层之上
      auto child_page_no = btr_node_ptr_get_child_page_no(rec, offsets);
      if (depth < split_level) {                               // ⑥ 还没到切分层 → 递归下沉
        create_ranges(scan_range, child_page_no, depth + 1, split_level, ranges, mtr);
        page_cur_move_to_next(&page_cursor);
        continue;
      }
      level_page_cursor = start_range(child_page_no, mtr, start, savepoints);  // ⑦ 降到叶子取起点
    } else {
      level_page_cursor = page_cursor;                          // 已在读层（或 root 即叶）
    }

    if (!page_rec_is_supremum(page_cur_get_rec(&level_page_cursor)))
      create_range(ranges, level_page_cursor, mtr);            // ⑧ 建持久游标

    // ⑨ 子树边界已固化：释放这条路径上的所有 S latch
    for (auto &sp : savepoints) mtr->release_block_at_savepoint(sp.first, sp.second);
    savepoints.clear();

    if (m_depth == 0 && depth == 0) m_depth = savepoints.size();  // 记录树深
    if (at_level == m_config.m_read_level) break;
    start = nullptr;                    // 只有第一个 range 用原始 start
    page_cur_move_to_next(&page_cursor);
  }
  /* 收尾：释放本层 savepoint */
}
```

**逐段解释**：

1. **① 逐页 S latch**：`block_get_s_latched` 拿页的 S 锁，配合外层 `index_s_lock()`，确保切分期间无 SMO。
2. **② 定位起点**：有 `start` 则用 `PAGE_CUR_LE` 找"小于等于 start"的位置（左路径在该层的落点）；无 start 则从第一条用户记录开始（跳过 infimum）。
3. **③ 遍历 node pointer**：每一条 node pointer 指向一个子树根——**这就是"按子树切分"的执行点**。
4. **④ 右边界**：`end->compare(rec) <= 0` 说明已越过右路径终点，停止（不做无谓的遍历）。
5. **⑤-⑦ 分层决策**：`at_level`（当前页层号）vs `m_config.m_read_level`（目标读层，默认 0 = 叶子）：
   - `at_level > m_read_level`：还需下行。`depth < split_level` → **递归下沉一层**（继续细分）；否则调 `start_range` 从该子树根一路降到叶子，取 range 的左边界。
   - 已在目标层（或 root 就是叶子）：直接用当前页游标。
6. **⑧ `create_range`**：把叶子游标包装成持久游标（`Iter`），接到区间链上。
7. **⑨ 立即释放 latch**（★ 关键优化）：持久游标已 `store_position`，该子树的路径 latch 可以全部释放——这是"一个 mtr 跑完全树却不会长期持有大量 latch"的原因。
8. **`m_depth`**：记录树深（首次 depth==0 时取 savepoints 数量），供 `SPLIT_THRESHOLD` 判定用。

**递归的形状**：`split_level` 决定"切到第几层"。`add_scan` 传 0（在 root 层切），`Ctx::split()` 传 1（在 root 的下一层切，即再细一层）。

### create_range：左闭右开的区间链

```cpp
void Parallel_reader::Scan_ctx::create_range(Ranges &ranges, page_cur_t &leaf_page_cursor, mtr_t *mtr) {
  auto iter = create_persistent_cursor(leaf_page_cursor, mtr);  // 复制记录 + 存位置
  if (!ranges.empty()) {
    ranges.back().second = iter;   // 上一个 range 的"右边界" = 当前 range 的"左边界"
  }
  ranges.push_back(Range(iter, std::make_shared<Iter>()));  // 新 range：右边界暂空
}
```

最终 `ranges` 形如：

```
[iter0, iter1)  [iter1, iter2)  [iter2, iter3)  ...  [iterN, end/null)
```

每个 `Ctx` 扫 `[自己的 start, 下一个 Ctx 的 start)`——**左闭右开，天然不重不漏**，无需任何跨线程协调。

`create_persistent_cursor` 的关键动作：落在 infimum 上则前进到首条用户记录 → `copy_row` 把整行拷贝到自己的 heap（**必须拷**，因为随后要释放原页 latch）→ `store_position` 保存位置。

**完整源码**（`row0pread.cc:552`）：

```cpp
std::shared_ptr<Parallel_reader::Scan_ctx::Iter>
Parallel_reader::Scan_ctx::create_persistent_cursor(
    const page_cur_t &page_cursor, mtr_t *mtr) const {
  ut_ad(index_s_own());

  std::shared_ptr<Iter> iter = std::make_shared<Iter>();
  iter->m_heap = mem_heap_create(sizeof(btr_pcur_t) + (srv_page_size / 16),
                                 UT_LOCATION_HERE);

  auto rec = page_cursor.rec;
  const bool is_infimum = page_rec_is_infimum(rec);
  if (is_infimum) rec = page_rec_get_next(rec);          // ★ 跳过 infimum

  if (page_rec_is_supremum(rec)) {
    /* Empty page, only root page can be empty. */
    ut_a(!is_infimum ||
         page_cursor.block->page.id.page_no() == m_config.m_index->page);
    return (iter);                                        // ★ 空页（只可能是空 root）
  }

  void *ptr = mem_heap_alloc(iter->m_heap, sizeof(btr_pcur_t));
  ::new (ptr) btr_pcur_t();                               // ★ placement new 在 iter 自己的 heap 上
  iter->m_pcur = reinterpret_cast<btr_pcur_t *>(ptr);

  iter->m_pcur->init(m_config.m_read_level);              // ★★ 记住 read_level：恢复时降到第 N 层

  /* Make a copy of the rec. */
  copy_row(rec, iter.get());                              // ★ 深拷起始记录

  iter->m_pcur->open_on_user_rec(page_cursor, PAGE_CUR_GE,
                                 BTR_ALREADY_S_LATCHED | BTR_SEARCH_LEAF);
  //                              ★★ PAGE_CUR_GE = 定位到 >= 该 key 的第一条 = 左闭
  //                                  BTR_ALREADY_S_LATCHED：页面已被本 mtr S-latch

  iter->m_pcur->store_position(mtr);                      // ★★ 持久化：之后可 commit mtr、跨线程恢复
  iter->m_pcur->set_fetch_type(Page_fetch::SCAN);

  return (iter);
}
```

```cpp
// row0pread.cc:523 —— copy_row 只比较 key 列
void Parallel_reader::Scan_ctx::copy_row(const rec_t *rec, Iter *iter) const {
  iter->m_offsets = rec_get_offsets(rec, m_config.m_index, nullptr, ULINT_UNDEFINED,
                                    UT_LOCATION_HERE, &iter->m_heap);
  auto rec_len = rec_offs_size(iter->m_offsets);
  auto copy_rec = static_cast<rec_t *>(mem_heap_alloc(iter->m_heap, rec_len));
  memcpy(copy_rec, rec, rec_len);                 // ★ 深拷到 iter->m_heap
  iter->m_rec = copy_rec;

  auto tuple = row_rec_to_index_entry_low(iter->m_rec, m_config.m_index,
                                          iter->m_offsets, iter->m_heap);
  /* We have copied the entire record but we only need to compare the
  key columns when we check for boundary conditions. */
  const auto n_compare = dict_index_get_n_unique_in_tree(m_config.m_index);
  dtuple_set_n_fields_cmp(tuple, n_compare);      // ★★ 只比较 key 列 → 边界判断更快
  iter->m_tuple = tuple;
}
```

**`partition()` 给最后一个 Range 封口**（`row0pread.cc:1196`）：

```cpp
  mtr.start();
  mtr.set_log_mode(MTR_LOG_NO_REDO);
  err = create_ranges(scan_range, m_config.m_index->page, 0, split_level, ranges, &mtr);
  //                                    ★ 从 root 页号 index->page 开始，depth=0

  if (err == DB_SUCCESS && scan_range.m_end != nullptr && !ranges.empty()) {
    auto &iter = ranges.back().second;      // ★ 最后一个 Range 的右端点此时还是空壳
    ut_a(iter->m_heap == nullptr);
    iter->m_heap = mem_heap_create(sizeof(btr_pcur_t) + (srv_page_size / 16), ...);
    iter->m_tuple = dtuple_copy(scan_range.m_end, iter->m_heap);
    /* Do a deep copy. */
    for (size_t i = 0; i < dtuple_get_n_fields(iter->m_tuple); ++i) {
      dfield_dup(&iter->m_tuple->fields[i], iter->m_heap);
    }
  }                                          // ★★ 把用户给的 m_end 填进去 → 右开边界生效
  mtr.commit();
```

**左闭右开的源码证据总表**：

| 证据 | 位置 | 内容 |
|---|---|---|
| 类型定义 | `row0pread.h:502` | `m_tuple will be [first->m_tuple, second->m_tuple).` |
| Range 定义 | `row0pread.h:516` | `The first cursor should read up to the second cursor [f, s).` |
| 全局说明 | `row0pread.h:96` | `A Ctx will scan from [Start, End) rows.` |
| **左闭** | `row0pread.cc:589` | `open_on_user_rec(page_cursor, PAGE_CUR_GE, ...)` |
| **左闭** | `row0pread.cc:1008` | `page_cur_search(block, index, key, PAGE_CUR_GE, &page_cursor)` |
| **右开（切分）** | `row0pread.cc:1116` | `if (end != nullptr && end->compare(rec, index, offsets) <= 0) break;` |
| **右开（扫描）** | `row0pread.cc:738` | `auto ret = end_tuple->compare(...); if (ret <= 0) break;` |
| 末区间封口 | `row0pread.cc:1210` | `iter->m_tuple = dtuple_copy(scan_range.m_end, iter->m_heap);` |
| 空壳断言 | `row0pread.cc:1030`、`:1213` | `ut_a(ranges.back().second->m_heap == nullptr)` |

> ★★ 两处右开判断（`create_ranges` 切分时与 `traverse_recs` 扫描时）用的是**完全相同的表达式** `compare(...) <= 0 → break`。这保证了"切分时认为越界"与"扫描时认为越界"严格一致。
>
> ★ 边界判断**不用 MVCC**：源码注释解释得很清楚——"key 值只在 delete 或 update 时变化，而 update 本质是 delete+insert，两种情况都只打 delete-mark，原始 key 值保留在页上"，所以最新（非 MVCC）版本足以判断是否越界。

### 粒度控制：split_level / SPLIT_THRESHOLD / 动态 split

代码里**没有**"每线程分几个叶子块"的直接参数。粒度由三个机制共同决定：

**① `split_level` 决定切在哪层**（见上文：`add_scan` 传 0，`Ctx::split` 传 1）。

**② `create_contexts` 的 `split_point`**（决定哪些 range 标记"待再切"）：

```cpp
const auto n = std::max(max_threads(), size_t{1});
if (ranges.size() > n) {
  split_point = (ranges.size() / n) * n;   // 子树比线程多：只把多出的"尾巴"标为可拆
} else if (m_depth < SPLIT_THRESHOLD) {
  split_point = n;                          // 树浅：不拆
}
// 否则 split_point = 0：全部标可拆
...
for (auto range : ranges) create_context(range, i >= split_point);
```

三条分支的语义：

| 条件 | split_point | 含义 |
|---|---|---|
| `ranges.size() > n` | `(size/n)*n` | 子树比线程多：前 k×n 个直接扫（已足够均衡），**只把多出的尾部标为可拆**——避免无谓的二次切分 |
| `m_depth < SPLIT_THRESHOLD(3)` | `n` | **树浅不拆**：源码注释 "If the tree is not very deep then don't split. For smaller tables it is more expensive to split because we end up traversing more blocks" |
| 其他（树深但子树少） | `0` | **全部标可拆**：子树数少于线程数，必须靠动态再切把负载摊开（解决负载不均） |

★ 注意 `SPLIT_THRESHOLD=3` 的判据是**树深**而非表大小：`m_depth` 在 `create_ranges` 里记录。树深 < 3 意味着子树数量本来就不多、每个子树也不大，再切一次要额外遍历更多块，**不划算**。

**③ 动态再切 `Ctx::split()`**：

```cpp
dberr_t Parallel_reader::Ctx::split() {
  Scan_range scan_range(m_range.first->m_tuple, m_range.second->m_tuple);
  /* S lock so that the tree structure doesn't change while we are
  figuring out the sub-trees to scan. */
  m_scan_ctx->index_s_lock();
  Parallel_reader::Scan_ctx::Ranges ranges{};
  m_scan_ctx->partition(scan_range, ranges, 1);      // 再切一层（split_level=1）
  if (!ranges.empty()) ranges.back().second = m_range.second;   // 末段接原右边界
  for (auto &range : ranges) err = m_scan_ctx->create_context(range, false);
  m_scan_ctx->index_s_unlock();
}
```

子 range 的 `m_split=false`（不再递归），入队后由**别的线程**取走——这就是"先跑完的线程帮慢的线程干活"。

### 线程队列与同步

⚠️ **勘误**：
- **队列不是无锁队列**。虽然 `row0pread.h:56` 确实 `#include "ut0mpmcbq.h"`（MPMC 无锁队列），但**并行扫描没用它**——真实实现是 `std::list<std::shared_ptr<Ctx>>` + **`ib_mutex_t m_mutex`**。
- **不存在 `Parallel_reader::start_threads()`**，真名 `parallel_read()`（`row0pread.cc:1288`）。

```cpp
// row0pread.h:410、427、430
  using Ctxs = std::list<std::shared_ptr<Ctx>, ut::allocator<std::shared_ptr<Ctx>>>;
  Ctxs m_ctxs{};              // ★ 全局 run queue，worker 从这里 dequeue 拿活
  mutable ib_mutex_t m_mutex; // ★ 普通 InnoDB mutex
  os_event_t m_event{};       // ★ worker 空闲等待 / 有活唤醒
  std::atomic_size_t m_ctx_id{};        // ★ 已创建的 Ctx 总数（含动态 split 新产生的）
  std::atomic_size_t m_n_completed{};   // ★ 已完成数
  std::atomic<dberr_t> m_err{DB_SUCCESS};
  static std::atomic_size_t s_active_threads;  // ★ 全局（跨实例）已占用线程数
```

```cpp
// row0pread.cc:792
void Parallel_reader::enqueue(std::shared_ptr<Ctx> ctx) {
  mutex_enter(&m_mutex);
  m_ctxs.push_back(ctx);          // ★ 尾插
  mutex_exit(&m_mutex);
}

std::shared_ptr<Parallel_reader::Ctx> Parallel_reader::dequeue() {
  mutex_enter(&m_mutex);
  if (m_ctxs.empty()) {
    mutex_exit(&m_mutex);
    return (nullptr);             // ★ 空队列 → worker 据此去睡
  }
  auto ctx = m_ctxs.front();
  m_ctxs.pop_front();             // ★ FIFO
  mutex_exit(&m_mutex);
  return (ctx);                   // ★ shared_ptr 传出，生命周期安全
}
```

**★ `worker()` 主循环全文**（`row0pread.cc:822`）：

```cpp
void Parallel_reader::worker(Parallel_reader::Thread_ctx *thread_ctx) {
  dberr_t err{DB_SUCCESS};
  dberr_t cb_err{DB_SUCCESS};

  if (m_start_callback) {
    thread_ctx->m_state = State::THREAD;
    cb_err = m_start_callback(thread_ctx);   // ★ 线程级 start 回调（adapter 在此建线程缓冲）
    if (cb_err != DB_SUCCESS) { err = cb_err; set_error_state(cb_err); }
  }

  /* Wait for all the threads to be spawned as it's possible that we could
  abort the operation if there are not enough resources to spawn all the
  threads. */
  if (!m_sync) {
    os_event_wait_time_low(m_event, std::chrono::microseconds::max(),
                           m_sig_count);    // ★★ 等 parallel_read() 的 os_event_set：全部就绪才开工
  }

  for (;;) {
    size_t n_completed{};
    int64_t sig_count = os_event_reset(m_event);   // ★ 先 reset 再干活，避免丢唤醒

    while (err == DB_SUCCESS && cb_err == DB_SUCCESS && !is_error_set()) {
      auto ctx = dequeue();                        // ★ 取一个 Ctx
      if (ctx == nullptr) break;                   // 队列空 → 本轮结束
      auto scan_ctx = ctx->m_scan_ctx;
      if (scan_ctx->is_error_set()) break;

      ctx->m_thread_ctx = thread_ctx;              // ★ 绑定执行线程

      if (ctx->m_split) {                          // ★★ 动态再切分触发点
        err = ctx->split();                        // 不扫，而是切成若干子 Range 再 enqueue
        /* Tell the other threads that there is work to do. */
        os_event_set(m_event);                     // ★ 唤醒睡着的兄弟来抢新活
      } else {
        if (m_start_callback) {
          thread_ctx->m_state = State::CTX;
          cb_err = m_start_callback(thread_ctx);
        }
        if (cb_err == DB_SUCCESS && err == DB_SUCCESS) {
          err = ctx->traverse();                   // ★★ 真正扫描这个 Range
        }
        if (m_finish_callback) {
          thread_ctx->m_state = State::CTX;
          cb_err = m_finish_callback(thread_ctx);
        }
      }

      /* Check for trx interrupted (useful in the case of small tables). */
      if (err == DB_SUCCESS && trx_is_interrupted(ctx->trx())) {
        err = DB_INTERRUPTED;
        scan_ctx->set_error_state(err);
        break;
      }
      ++n_completed;                               // ★ 本地计数，批量加到全局
    }

    if (cb_err != DB_SUCCESS || err != DB_SUCCESS || is_error_set()) break;

    m_n_completed.fetch_add(n_completed, std::memory_order_relaxed);

    if (m_n_completed == m_ctx_id) {               // ★★ 全部 Ctx 完成（含动态产生的）
      os_event_set(m_event);                       // 唤醒其它 worker 一起退出
      break;
    }

    if (!m_sync) {
      os_event_wait_time_low(m_event, std::chrono::microseconds::max(),
                             sig_count);           // ★ 队列空但活没干完 → 睡等别人 split 出新活
    }
  }

  if (err != DB_SUCCESS && !is_error_set()) set_error_state(err);
  if (is_error_set()) os_event_set(m_event);       // ★ 出错要把睡着的兄弟全部叫醒

  if (m_finish_callback) {
    thread_ctx->m_state = State::THREAD;
    cb_err = m_finish_callback(thread_ctx);        // ★ 线程级 finish（adapter 在此 flush 尾包）
    if (cb_err != DB_SUCCESS) { err = cb_err; set_error_state(cb_err); }
  }

  ut_a(is_error_set() || (m_n_completed == m_ctx_id && is_queue_empty()));
  //  ★★ 退出不变式：要么出错，要么全部完成且队列空
}
```

**★ 起线程：`parallel_read()`**（`row0pread.cc:1288`）：

```cpp
void Parallel_reader::parallel_read() {
  if (m_ctxs.empty()) return;               // ★ 空表：一个线程都不起

  if (m_sync) {                             // ★ n_threads==0：调用者线程自己扫
    auto ptr = ut::new_withkey<Thread_ctx>(UT_NEW_THIS_FILE_PSI_KEY, 0);
    if (ptr == nullptr) { set_error_state(DB_OUT_OF_MEMORY); return; }
    m_thread_ctxs.push_back(ptr);
    /* Set event to indicate to ::worker() that no threads will be spawned. */
    os_event_set(m_event);
    worker(m_thread_ctxs[0]);               // ★ 单线程复用同一套 worker 逻辑
    return;
  }

  ut_a(m_n_threads > 0);
  m_thread_ctxs.reserve(m_n_threads);

  for (size_t i = 0; i < m_n_threads; ++i) {
    try {
      auto ptr = ut::new_withkey<Thread_ctx>(UT_NEW_THIS_FILE_PSI_KEY, i);
      if (ptr == nullptr) { set_error_state(DB_OUT_OF_MEMORY); return; }
      m_thread_ctxs.emplace_back(ptr);
      m_parallel_read_threads.emplace_back(
          os_thread_create(parallel_read_thread_key, i + 1,
                           &Parallel_reader::worker, this, m_thread_ctxs[i]));
      m_parallel_read_threads.back().start();
    } catch (...) {
      err = DB_OUT_OF_RESOURCES;
      set_error_state(err);                 // ★ 线程不够 → 已起的 worker 自行退出
      break;
    }
  }

  os_event_set(m_event);                    // ★★ 放行所有 worker：正式开工
}
```

**`run()` / `spawn()` / 同步模式**（`row0pread.cc:1351`）：

```cpp
dberr_t Parallel_reader::spawn(size_t n_threads) noexcept {
  m_n_threads = n_threads;
  if (max_threads() > m_n_threads) {
    release_unused_threads(max_threads() - m_n_threads);  // ★ 申请 4 个只用 2 个 → 归还 2 个配额
  }
  parallel_read();
  return DB_SUCCESS;
}

dberr_t Parallel_reader::run(size_t n_threads) {
  m_err = DB_SUCCESS;                       // 可能是 DB_OUT_OF_RESOURCES 后的重试
  if (n_threads == 0) m_sync = true;        // ★ 同步模式：不起线程
  const auto err = spawn(n_threads);
  if (m_sync) {
    ut_a(m_n_threads == 0);
    return is_error_set() ? m_err.load() : DB_SUCCESS;
  } else {
    join();                                 // ★★ 用户线程阻塞等所有 worker
    ...
  }
}
```

> ★★ **`m_sync` 是"退化串行"的实现**：`run(0)` 让调用者线程直接跑 `worker()`，整条代码路径完全复用（包括动态 split），只是没有并发。所有消费者在 `DB_OUT_OF_RESOURCES` 时都走这条降级：
> ```cpp
>   if (err == DB_OUT_OF_RESOURCES) {
>     ib::warn(ER_INNODB_OUT_OF_RESOURCES)
>         << "Resource not available to create threads for parallel scan."
>         << " Falling back to single thread mode.";
>     err = reader.run(0);
>   }
> ```

**全局线程配额**（`row0pread.cc:103`）——乐观先占、不够再退：

```cpp
size_t Parallel_reader::available_threads(size_t n_required, bool use_reserved) {
  auto max_threads = MAX_THREADS;              // 256
  constexpr auto SEQ_CST = std::memory_order_seq_cst;
  auto active = s_active_threads.fetch_add(n_required, SEQ_CST);   // ★ 乐观先占

  if (use_reserved) max_threads += MAX_RESERVED_THREADS;           // +16 → 272

  if (active < max_threads) {
    const auto available = max_threads - active;
    if (n_required <= available) return n_required;                 // 全够
    const auto release = n_required - available;
    s_active_threads.fetch_sub(release, SEQ_CST);
    return available;                                              // ★ 只给部分
  }
  s_active_threads.fetch_sub(n_required, SEQ_CST);
  return 0;                                                        // ★ 一个都拿不到
}
```

> ⚠️ **`srv_parallel_read_threads` 是死变量**——全仓仅 2 处命中（定义 `srv0srv.cc:213`、声明 `srv0srv.h:391`），**没有任何读取点**，也没有 sysvar 赋值。真正的全局配额是 `Parallel_reader::s_active_threads`。

**`Ctx::traverse()`（一个 Range 的扫描主流程）**（`row0pread.cc:618`）：

```cpp
dberr_t Parallel_reader::Ctx::traverse() {
  /* Take index lock if the requested read level is on a non-leaf level as the
  index lock is required to access non-leaf page.  */
  if (m_scan_ctx->m_config.m_read_level != 0) {
    m_scan_ctx->index_s_lock();     // ★★ 只有读非叶子层才加索引锁；叶子层扫描完全不加
  }

  mtr_t mtr;
  mtr.start();
  mtr.set_log_mode(MTR_LOG_NO_REDO);  // ★ 不写 redo，但脏页仍进 flush list

  auto &from = m_range.first;         // ★ 划分阶段建好的持久游标

  PCursor pcursor(from->m_pcur, &mtr, m_scan_ctx->m_config.m_read_level);
  pcursor.restore_position();         // ★★ 从 store_position 保存的位置恢复（跨线程、跨 mtr）

  m_thread_ctx->m_pcursor = &pcursor;
  dberr_t err = traverse_recs(&pcursor, &mtr);

  if (mtr.is_active()) mtr.commit();
  m_thread_ctx->m_pcursor = nullptr;

  if (m_scan_ctx->m_config.m_read_level != 0) m_scan_ctx->index_s_unlock();
  return (err);
}
```

**`traverse_recs()` 的页内前进与边界**（`row0pread.cc:654`，节选）：

```cpp
  const auto &end_tuple = m_range.second->m_tuple;   // ★ 右开边界
  ...
    if (page_cur_is_after_last(cur)) {               // ★ 当前页扫完（游标在 supremum 上）
      ...
      if (!(m_n_pages % TRX_IS_INTERRUPTED_PROBE) && trx_is_interrupted(trx())) {
        err = DB_INTERRUPTED; break;                 // ★ 每 5 万页才探一次 kill
      }
      if (pcursor->is_after_last_on_page() && !move_to_next_node(pcursor)) break;
      ++m_n_pages;
    }
    ...
    if (end_tuple != nullptr) {
      auto ret = end_tuple->compare(rec, index, offsets);
      /* Note: The range creation doesn't use MVCC. Therefore it's possible
      that the range boundary entry could have been deleted. */
      if (ret <= 0) break;                           // ★★ 右开：rec >= end_tuple 立即停止
    }
    bool skip{};
    if (page_is_leaf(cur->block->frame)) {           // ★ 非叶子层不做可见性判断
      skip = !m_scan_ctx->check_visibility(rec, offsets, heap, mtr);
    }
    if (!skip) {
      m_rec = rec; m_offsets = offsets; m_block = cur->block;
      err = m_scan_ctx->m_f(this);                   // ★★ 每条可见记录回调一次
      if (err != DB_SUCCESS) break;
      m_start = false;
    }
    page_cur_move_to_next(cur);                      // ★★ 页内前进
```

**页内前进的完整下钻链**（并行扫描页内遍历的唯一一行就是 `page_cur_move_to_next`）：

```
page_cur_move_to_next(cur)              page0cur.ic:136   cur->rec = page_rec_get_next(cur->rec)
 └─ page_rec_get_next(rec)              page0page.ic:628
     └─ page_rec_get_next_low(rec,comp) page0page.ic:598  page_align + 越界检查
         └─ rec_get_next_offs(rec,comp) rem0rec.ic:161    mach_read_from_2(rec - REC_NEXT)
```

```cpp
// rem0rec.ic:161 —— 为什么必须用 page_rec_get_next 而不是直接读偏移
static inline ulint rec_get_next_offs(const rec_t *rec, ulint comp) {
  ulint field_value = mach_read_from_2(rec - REC_NEXT);   // 记录头前 2 字节
  if (comp) {
    /* Check if the result offset is still on the same page. We allow
    `field_value` to be interpreted as negative 16bit integer. */
    if (field_value == 0) return (0);
    return (ut_align_offset(rec + field_value, UNIV_PAGE_SIZE));  // ★ COMPACT：相对增量，可为负
  } else {
    return (field_value);                                          // ★ REDUNDANT：绝对页内偏移
  }
}
```

> ★★ **两种行格式语义不同**（compact 是有符号相对增量，redundant 是绝对偏移），且 `rec_get_next_offs` 不做任何边界检查——所以必须用 `page_rec_get_next_low()` 补齐 `page_align()` + `offs >= UNIV_PAGE_SIZE` 报 `ut_error`。
>
> ⚠️ **不存在 `rec_get_next()` / `page_rec_next_get()`**（全仓 0 命中）。
>
> ★ **infimum / supremum 从不显式判断**——靠 `page_cur_t` 的两个不变量：进页先 `page_cur_set_before_first()`（落在 infimum）再 `page_cur_move_to_next()` 跳过；页内循环终止条件 `page_cur_is_after_last(cur)` ⟺ `page_rec_is_supremum(cur->rec)`。

**翻页：只走同层 `FIL_PAGE_NEXT`，从不爬父节点**（`row0pread.cc:360`）：

```cpp
dberr_t PCursor::move_to_user_rec() noexcept {
  auto cur = m_pcur->get_page_cur();
  const auto next_page_no = btr_page_get_next(page_cur_get_page(cur), m_mtr);  // ★ 同层右兄弟
  if (next_page_no == FIL_NULL) { m_mtr->commit(); return DB_END_OF_INDEX; }
  ...
  if (m_read_level == 0) {
    block = buf_page_get_gen(page_id_t(page_id.space(), next_page_no),
                             block->page.size, RW_S_LATCH, nullptr,
                             Page_fetch::SCAN, UT_LOCATION_HERE, m_mtr);
    //  ★ 叶子层：直接 S latch
  } else {
    /* Read IO should be waited for. But s-latch should be nowait,
    to avoid deadlock opportunity completely. */
    block = buf_page_get_gen(..., RW_NO_LATCH, nullptr, Page_fetch::SCAN, ...);
    bool success = buf_page_get_known_nowait(
        RW_S_LATCH, block, Cache_hint::KEEP_OLD, __FILE__, __LINE__, m_mtr);
    btr_leaf_page_release(block, RW_NO_LATCH, m_mtr);
    if (!success) return DB_LOCK_NOWAIT;      // ★ 非叶子层用 nowait 抢 S latch，避免死锁
  }
  btr_leaf_page_release(page_cur_get_block(cur), RW_S_LATCH, m_mtr);  // ★ 放旧页
  page_cur_set_before_first(block, cur);
  page_cur_move_to_next(cur);                 // ★ 跳过 infimum
  return DB_SUCCESS;
}
```

> ★★ **`row0pread.cc` 里一次都没调用 `btr_page_get_father_node_ptr()`**。子树间移动全靠同层链表，**从不做"向上爬父节点"**——这正是叶子层并行扫描不需要索引锁的根本原因。
>
> ⚠️ `btr_page_get_father_node_ptr()` **不在 `btr0btr.h`**，是 `btr0btr.cc:638` 的 `static` 函数 + 两个 `inline` wrapper（`:711`/`:718`），外部不可调用。

**页边界主动让出**（`PCursor::move_to_next_block()`，`row0pread.cc:426`）：

```cpp
  if (rw_lock_get_waiters(dict_index_get_lock(index))) {
    /* There are waiters on the index tree lock. Store and restore
    the cursor position, and yield so that scanning a large table
    will not starve other threads. */
    savepoint();                        // ★ commit mtr，放开所有 latch
    std::this_thread::yield();          // ★★ 让出 CPU
    err = restore_from_savepoint();
  } else {
    err = move_to_user_rec();
  }

  while (err == DB_LOCK_NOWAIT) {
    savepoint();
    /* Forces to restore from index root. */
    m_pcur->m_block_when_stored.clear();   // ★ 清空 hint，强制从 root 重新下降
    err = restore_from_savepoint();
    ut_ad(n_retries++ < 10);
  }
```

**恢复定位：乐观 / 悲观两条路**（`btr0pcur.cc:148`）：

```cpp
  /* Try optimistic restoration. */
  if (m_block_when_stored.run_with_hint([&](buf_block_t *hint) {
        return hint != nullptr &&
               btr_cur_optimistic_latch_leaves(hint, m_modify_clock,
                                               &latch_mode, &m_btr_cur,
                                               location.filename, location.line, mtr);
      })) {
    //  ★★ 乐观：block 仍有效 + modify_clock 未变 → 直接复用原页，不重新搜索
    m_pos_state = BTR_PCUR_IS_POSITIONED;
    if (m_rel_pos == BTR_PCUR_ON) return (true);   // 精确落在原记录
    if (is_on_user_rec()) m_pos_state = BTR_PCUR_IS_POSITIONED_OPTIMISTIC;
    return (false);
  }

  /* If optimistic restoration did not succeed, open the cursor anew */
  tuple = dict_index_build_data_tuple(index, m_old_rec, m_old_n_fields, heap);
  switch (m_rel_pos) {
    case BTR_PCUR_ON:     mode = PAGE_CUR_LE; break;   // 原来"在记录上" → <= 该 key 的最后一条
    case BTR_PCUR_AFTER:  mode = PAGE_CUR_G;  break;   // ★ 原来"在某条之后" → > 该 key 的第一条
    case BTR_PCUR_BEFORE: mode = PAGE_CUR_L;  break;
  }
  open_no_init(index, tuple, mode, latch_mode, 0, mtr, location);
  //  ★★ 从 root 重新下降 —— 且降到 m_read_level 层，不是一律降到叶子层
```

> ★★ **"停在指定层"靠 `m_read_level`**：`open_no_init()`（`btr0pcur.h:609`）传给 `btr_cur_search_to_nth_level()` 的是 **`m_read_level`** 而不是参数 `level`。这就是直方图采样能"只读 level 1"的原因。
>
> ★ `store_position()` 存的是 **(block hint + modify_clock + old_rec 排序前缀 + rel_pos)** 四元组，不是整行（`dict_index_copy_rec_order_prefix`，只拷 `n_unique_in_tree` 个字段）。
>
> ⚠️ **`m_block_when_stored` 不是裸指针**，是 `buf::Block_hint`（`btr0pcur.h:483`），内部带"指针是否仍有效"的校验，必须经 `run_with_hint()` 使用。
> ⚠️ **不存在 `m_withdraw_clock`**（全库 0 命中），只有 `m_modify_clock`。

**可见性：不加行锁，走 MVCC 回捞**（`row0pread.cc:468`）：

```cpp
bool Parallel_reader::Scan_ctx::check_visibility(const rec_t *&rec, ulint *&offsets,
                                                 mem_heap_t *&heap, mtr_t *mtr) {
  if (!m_trx) {
    /* Do nothing */                    // ★ trx==nullptr（DDL 非 online）→ 不做 MVCC
  } else if (m_trx->read_view != nullptr) {
    auto view = m_trx->read_view;
    if (m_config.m_index->is_clustered()) {
      trx_id_t rec_trx_id;
      if (m_config.m_index->trx_id_offset > 0) {
        rec_trx_id = trx_read_trx_id(rec + m_config.m_index->trx_id_offset);  // 快路径
      } else {
        rec_trx_id = row_get_rec_trx_id(rec, m_config.m_index, offsets);
      }
      if (m_trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
          !view->changes_visible(rec_trx_id, table_name)) {
        row_vers_build_for_consistent_read(rec, mtr, m_config.m_index, &offsets,
                                           view, &heap, heap, &old_vers, nullptr, nullptr);
        rec = old_vers;                 // ★★ 走 undo 回捞，而不是加行锁等待
        if (rec == nullptr) return (false);
      }
    } else {
      /* Secondary index scan not supported yet. */
      ut_error;                         // ★★ 二级索引并行扫描未实现，直接 ut_error
    }
  }
  if (rec_get_deleted_flag(rec, m_config.m_is_compact)) {
    return (false);                     // ★ delete-marked → 跳过
  }
  return (true);
}
```

> ⚠️ **`row0pread.cc` 完全没有 `lock_clust_rec_cons_read_sees`，也不加任何行锁**——并行扫描是纯 MVCC 一致性读。
>
> ⚠️ **mtr 用的是 `MTR_LOG_NO_REDO` 不是 `MTR_LOG_NONE`**（`row0pread.cc:627/347/1203`）：不写 redo，但脏页仍进 flush list。
>
> ★ **`Page_fetch::SCAN` 的两个实际效果**（`buf0buf.cc`）：① **不做随机预读**（`:4038-4044`）；② **不 `buf_page_make_young_if_needed`，不污染 LRU 头部**（`:4316-4320`、`:4454-4456`）。页**不要求**已在 buffer pool，会正常发起 IO。
>
> ⚠️ **并行扫描不禁用 AHI**（`row0pread.cc` 里无 `btr_search_disable`）。

**全局并发限制**（`available_threads`）：

```cpp
size_t Parallel_reader::available_threads(size_t n_required, bool use_reserved) {
  auto active = s_active_threads.fetch_add(n_required, SEQ_CST);   // 先乐观预留
  auto max_threads = use_reserved ? MAX_THREADS + MAX_RESERVED_THREADS : MAX_THREADS;
  if (active < max_threads) { /* 返回能给的数 */ }
  s_active_threads.fetch_sub(n_required, ...);   // 超了全退
  return 0;
}
```

先原子 `fetch_add` 预留、再按 `max - active` 计算真正能给的数、多余退回。返回 0 表示**回退为单线程**（`m_sync` 路径）。

**worker 主循环**是纯消费者：

```cpp
for (;;) {
  auto ctx = dequeue();          // 从共享 run queue 取任务
  if (ctx == nullptr) break;
  if (ctx->m_split) ctx->split();   // 需要再切：切成子 Ctx 再入队
  else ctx->traverse();              // 真正扫这个子树
}
```

**同步模型**（8.0.39，早期版本的 `Synthetic_sync` 已移除）：

- `m_mutex` + `m_ctxs`：任务队列互斥；
- `os_event_t m_event` + `m_sig_count`：`os_event_reset` + `os_event_wait_time_low` 的 reset-then-wait 配对；生产者在"spawn 放行 / split 产生新任务 / 任务完成"时 `os_event_set` 唤醒；
- `m_sync`：`max_threads == 0` 时跳过线程创建，当前线程直接跑 worker（同步回退）。

★ **线程间无记录级数据交换**：每个线程独立 `dequeue` 一个 Ctx，用自己的持久游标扫自己的区间，结果写自己的分片计数器。唯一共享的是：任务队列、全局原子错误状态、事件、索引 tree latch。

### 三个实际消费者

**(1) 并行 COUNT**（`row_mysql_parallel_select_count_star`）：

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

`Counter::Shards`（`ut0counter.h`）是**按 cache line 对齐的分片原子计数器**，每线程用 `thread_id` 作 shard id，**无共享竞争**。

**(2) 并行 CHECK TABLE**（`parallel_check_table`）：三个分片计数器（行数 / 重复键 / 乱序），每线程维护 `prev_tuple` 比较相邻记录顺序。★ `ctx->m_start`（range 起点）用于**重置 prev_tuple**——并行切分后 range 之间不连续，**不能**拿上个 range 的最后一条与当前 range 第一条比较（否则误报"乱序"）。

**(3) `Parallel_reader_adapter`**：给二级引擎批量喂数据。`process_rows` 把 InnoDB rec 转成 MySQL 行（`row_sel_store_mysql_rec`）进 2MB 缓冲（`ADAPTER_SEND_BUFFER_SIZE`），缓冲满或遇到新 range 时 `send_batch` 上送 `m_load_fn`。

⚠️ **勘误："三个消费者"是错的，真实有 5 个**。全仓 `Parallel_reader` 只有 **5 个构造点**：

| # | 构造点 | 消费者 | 扫什么 |
|---|---|---|---|
| 1 | `row0mysql.cc:4389` | `row_mysql_parallel_select_count_star()` | 聚簇索引全表计数 |
| 2 | `row0mysql.cc:4458` | `parallel_check_table()`（static） | 聚簇索引全扫，查乱序 + 唯一键重复 |
| 3 | `ddl/ddl0par-scan.cc:186` | `Parallel_cursor::scan()` | DDL 建二级索引 / 重建表 |
| 4 | `row0pread-histogram.h:161`（成员） | `Histogram_sampler` | `read_level` 层抽样 |
| 5 | `row0pread-adapter.h:208`（成员） | `Parallel_reader_adapter` | 喂回 server 层（社区版无调用者） |

> ★★ **逻辑备份（mysqldump / `SELECT *`）不是消费者**——它走普通 `rnd_next`，不经过 `Parallel_reader`。这是最常见的误解。

**消费者 ① 完整源码**（`row0mysql.cc:4377`）：

```cpp
dberr_t row_mysql_parallel_select_count_star(
    trx_t *trx, std::vector<dict_index_t *> &indexes, size_t n_threads,
    ulint *n_rows) {
  ut_a(n_threads > 1);
  ut_a(!indexes.empty());
  using Shards = Counter::Shards<Parallel_reader::MAX_THREADS>;
  Shards n_recs;
  Counter::clear(n_recs);

  const Parallel_reader::Scan_range FULL_SCAN;   // ★ 全 nullptr = (-inf, +inf) 全表

  Parallel_reader reader(n_threads);

  dberr_t err{DB_SUCCESS};

  for (auto index : indexes) {       // ★ 分区表会 push 多个 index（每分区一个聚簇索引）
    Parallel_reader::Config config(FULL_SCAN, index);
    err = reader.add_scan(trx, config, [&](const Parallel_reader::Ctx *ctx) {
      Counter::inc(n_recs, ctx->thread_id());    // ★ 每条记录：只做本线程 shard 自增
      return DB_SUCCESS;
    });
    if (err != DB_SUCCESS) break;
  }

  if (err == DB_SUCCESS) err = reader.run(n_threads);   // ★ spawn + join

  if (err == DB_OUT_OF_RESOURCES) {
    ib::warn(ER_INNODB_OUT_OF_RESOURCES)
        << "Resource not available to create threads for parallel scan."
        << " Falling back to single thread mode.";
    err = reader.run(0);            // ★★ 退化为同步单线程
  }

  if (err == DB_SUCCESS) {
    Counter::for_each(n_recs, [=](const Counter::Type n) { if (n > 0) *n_rows += n; });
  }
  return err;
}
```

**消费者 ② `parallel_check_table()` 的回调本体**（`row0mysql.cc:4432`）：

```cpp
  auto err = reader.add_scan(trx, config, [&](const Parallel_reader::Ctx *ctx) {
    const auto rec = ctx->m_rec;
    const auto block = ctx->m_block;
    const auto id = ctx->thread_id();

    Counter::inc(n_recs, id);

    if (ctx->m_start) {
      /* Starting scan of a new range. We need to reset the previous tuple
      because we don't know what the value of the previous last tuple was. */
      prev_tuples[id] = nullptr;    // ★★ 新子树起点：无法跨子树比较顺序，重置 prev
    }

    auto prev_tuple = prev_tuples[id];
    auto offsets = rec_get_offsets(rec, index, nullptr, ULINT_UNDEFINED,
                                   UT_LOCATION_HERE, &heap);

    if (prev_tuple != nullptr) {
      ulint matched_fields = 0;
      auto cmp = prev_tuple->compare(rec, index, offsets, &matched_fields);

      bool contains_null = false;
      const auto n_ordering = dict_index_get_n_ordering_defined_by_user(index);
      for (size_t i = 0; i < n_ordering; ++i) {
        const auto nth_field = dtuple_get_nth_field(prev_tuple, i);
        if (UNIV_SQL_NULL == dfield_get_len(nth_field)) { contains_null = true; break; }
      }

      if (cmp > 0) {                                     // ★ 乱序 → 索引损坏
        Counter::inc(n_corrupt, id);
        ib::error(ER_IB_ERR_INDEX_RECORDS_WRONG_ORDER) << "Index records in a wrong order ...";
      } else if (dict_index_is_unique(index) && !contains_null &&
                 matched_fields >= n_ordering) {
        Counter::inc(n_dups, id);                        // ★ 唯一索引重复键（NULL 不算）
        ib::error(ER_IB_ERR_INDEX_DUPLICATE_KEY) << "Duplicate key ...";
      }
    }

    if (prev_blocks[id] != block || prev_blocks[id] == nullptr) {
      mem_heap_empty(heap);                              // ★ 换页才清 heap
      offsets = rec_get_offsets(rec, index, nullptr, ULINT_UNDEFINED,
                                UT_LOCATION_HERE, &heap);
      prev_blocks[id] = block;
    }
    prev_tuples[id] = row_rec_to_index_entry(rec, index, offsets, heap);
    return DB_SUCCESS;
  });
```

> ★ 每个线程一套 `heaps` / `prev_tuples` / `prev_blocks`（按 `thread_id` 索引），**完全无共享**。

**消费者 ⑤ `Parallel_reader_adapter`：如何喂回 server 层**：

```cpp
// row0pread-adapter.h:45
  static constexpr size_t ADAPTER_SEND_BUFFER_SIZE = 2 * 1024 * 1024;   // 每线程 2MB

  using Load_fn  = handler::Load_cbk;      // bool(void *cookie, uint nrows, void *rowdata, uint64 partition_id)
  using End_fn   = handler::Load_end_cbk;  // void(void *cookie)
  using Init_fn  = handler::Load_init_cbk; // bool(void *cookie, ulong ncols, ulong row_len, col_offsets, null_byte_offsets, null_bitmasks)
```

```cpp
// row0pread-adapter.cc:39 —— 批大小 = 2MB / 行长
Parallel_reader_adapter::Parallel_reader_adapter(size_t max_threads, ulint rowlen)
    : m_parallel_reader(max_threads) {
  m_batch_size = ADAPTER_SEND_BUFFER_SIZE / rowlen;
}

// :172 —— 每条记录都走这里
dberr_t Parallel_reader_adapter::process_rows(const Parallel_reader::Ctx *reader_ctx) {
  auto reader_thread_ctx = reader_ctx->thread_ctx();
  auto ctx = reader_thread_ctx->get_callback_ctx<Thread_ctx>();
  auto blob_heap = reader_thread_ctx->m_blob_heap;   // ★★ 每线程独立 blob heap

  /* Start of a new range, send what we have buffered. */
  if ((reader_ctx->m_start && n_pending > 0) || is_buffer_full(ctx)) {
    err = send_batch(reader_thread_ctx, ctx->m_partition_id, n_pending);
    if (blob_heap != nullptr) mem_heap_empty(blob_heap);
  }

  offsets = rec_get_offsets(reader_ctx->m_rec, reader_ctx->index(), offsets, ...);
  const auto next_rec = ctx->m_n_read % m_batch_size;
  const auto buffer_loc = &ctx->m_buffer[0] + next_rec * m_mysql_row.m_max_len;

  if (row_sel_store_mysql_rec(buffer_loc, m_prebuilt, reader_ctx->m_rec,
                              nullptr, true, reader_ctx->index(),
                              reader_ctx->index(), offsets, false, nullptr,
                              blob_heap)) {
    //                          ★ InnoDB 物理记录 → MySQL 行格式，写入本线程 buffer
    ++ctx->m_n_read;
    ctx->m_partition_id = reader_ctx->partition_id();
  } else { err = DB_ERROR; }
  return err;
}
```

> ★ ★ **`row_prebuilt_t` 在多线程间共享**——源码里明说这是"脆弱代码"：
> ```cpp
>   /** Prebuilt to use for conversion to MySQL row format.
>   NOTE: We are sharing this because we don't convert BLOBs yet. There are
>   data members in row_prebuilt_t that cannot be accessed in multi-threaded
>   mode e.g., blob_heap.
>
>   row_prebuilt_t is designed for single threaded access and to share
>   it among threads is not recommended unless "you know what you are doing".
>   This is very fragile code as it stands.
>
>   To solve the blob heap issue in prebuilt we use per thread m_blob_heaps. */
>   row_prebuilt_t *m_prebuilt{};
> ```
> ★★ **BLOB 不禁用并行扫描**——改为每线程 `create_blob_heap()` 绕开共享 `prebuilt->blob_heap`。

### 触发条件与限制

```cpp
if (prebuilt->trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&  // ① RC/RR（RU 不做）
    prebuilt->select_lock_type == LOCK_NONE &&                    // ② 非锁定读
    index->is_clustered() &&                                      // ③ 聚簇索引
    (check_keys || prebuilt->trx->mysql_n_tables_locked == 0) &&  // ④ check 或无 LOCK TABLES
    !prebuilt->ins_sel_stmt) {                                    // ⑤ 非 INSERT...SELECT
```

五个条件各自的原因：

- ① RU 隔离级别下不做（历史包袱 + 语义）；
- ② 加锁读会引入锁与 MVCC 的交互，破坏"独立扫描"；
- ③ 聚簇索引（见「为什么只支持聚簇索引」）；
- ④ `LOCK TABLES` 下不做（表锁语义与并发扫描冲突）；
- ⑤ `INSERT...SELECT` 会边读边写（Halloween 风险），不做。

**二级索引不支持**（三处证据）：头文件注释明说 "Secondary index scans are not supported"；MVCC 可见性判断里直接 `ut_error`。

`innodb_parallel_read_threads`（`ha_innodb.cc`）默认 **4**、范围 **[1, 256]**。

**完整准入判断**（`row_scan_index_for_mysql()`，`row0mysql.cc:4586`）——这是全仓唯一的"是否启用并行"总闸门：

```cpp
dberr_t row_scan_index_for_mysql(row_prebuilt_t *prebuilt, dict_index_t *index,
                                 size_t max_threads, bool check_keys,
                                 ulint *n_rows) {
  /* Don't support RTree Leaf level scan */
  ut_ad(!dict_index_is_spatial(index));        // 限制①：空间索引
  ...
  } else if (dict_index_is_online_ddl(index) || (index->type & DICT_FTS)) {
    return (DB_SUCCESS);                       // 限制②：online DDL 二级索引 / FTS
  }

  DBUG_EXECUTE_IF("ib_disable_parallel_read", goto skip_parallel_read;);  // 调试开关

  if (prebuilt->trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&   // 限制③ RU 禁用
      prebuilt->select_lock_type == LOCK_NONE &&                     // 限制④ 加锁读禁用
      index->is_clustered() &&                                       // 限制⑤ 只聚簇索引
      (check_keys || prebuilt->trx->mysql_n_tables_locked == 0) &&   // 限制⑥ LOCK TABLES
      !prebuilt->ins_sel_stmt) {                                     // 限制⑦ INSERT...SELECT
    auto n_threads = Parallel_reader::available_threads(max_threads, false);
    //                                                  ★ use_reserved=false → 上限 256

    if (n_threads > 1) {
      trx_start_if_not_started_xa(prebuilt->trx, false, UT_LOCATION_HERE);
      trx_assign_read_view(prebuilt->trx);    // ★★ 必须先建 read view，worker 才有一致视图
      ...
      if (!check_keys) return row_mysql_parallel_select_count_star(...);   // COUNT(*)
      return parallel_check_table(trx, index, n_threads, n_rows);          // CHECK TABLE
    } else if (n_threads == 1) {
      /* If there is a single thread available then we do a sync scan. */
      Parallel_reader::release_threads(n_threads);   // ★ 只拿到 1 个 → 还回去走串行
    }
  }

skip_parallel_read:
  ...
  //  ★ 以下为串行 fallback 扫描
```

**限制全集**（含源码位置）：

| 路径 | 限制 | 判断表达式 | 位置 |
|---|---|---|---|
| COUNT/CHECK | 空间索引 | `ut_ad(!dict_index_is_spatial(index))` | `row0mysql.cc:4592` |
| | Online DDL 二级索引 / FTS | `dict_index_is_online_ddl(index) \|\| (index->type & DICT_FTS)` | `:4602` |
| | RU 隔离级别 | `isolation_level > TRX_ISO_READ_UNCOMMITTED` | `:4611` |
| | 加锁读 | `select_lock_type == LOCK_NONE` | `:4612` |
| | 非聚簇索引 | `index->is_clustered()` | `:4612` |
| | LOCK TABLES | `check_keys \|\| mysql_n_tables_locked == 0` | `:4613` |
| | INSERT...SELECT | `!prebuilt->ins_sel_stmt` | `:4614` |
| | 线程数 | `n_threads > 1` | `:4615` |
| DDL 建索引 | **虚拟列** | `has_virtual_columns() \|\| has_fts_indexes()` → 单线程 | `ddl0par-scan.cc:51` |
| | **FTS 索引** | 同上 | `ddl0par-scan.cc:51` |
| | 需要保序 / 空间索引 | `is_skip_file_sort() \|\| is_spatial_index()` → 单线程 | `ddl0par-scan.cc:109` |
| | online DDL 隔离级别 | `ut_a(!m_online \|\| isolation_level == REPEATABLE_READ)` | `ddl0par-scan.cc:99` |
| 采样 | 只支持 SYSTEM 采样法 | `sampling_method != SYSTEM` → return 0 | `ha_innodb.cc:10699` |
| | **并行读不支持采样** | `available_threads(1, false)` **硬编码 1** | `ha_innodb.cc:10731` |
| 全局 | 二级索引并行扫描 | `/* Secondary index scan not supported yet. */ ut_error;` | `row0pread.cc:506` |

> ⚠️ **不存在 `Parallel_reader::is_table_supported()`**（全仓 0 命中）——前置校验散落在各调用点。
> ⚠️ **几个流传的"禁用"其实不存在**：**压缩行格式不禁用**（`row0pread.cc` 全文 grep `compressed/zip` 零命中，`m_is_compact` 只用来读 delete flag）；**临时表不禁用**；**BLOB 不禁用**；**AHI 不禁用**；**buffer pool 太小不禁用**。

**sysvar 定义与全部读取点**（`ha_innodb.cc:1107`）：

```cpp
static MYSQL_THDVAR_ULONG(parallel_read_threads, PLUGIN_VAR_RQCMDARG,
                          "Number of threads to do parallel read.", nullptr,
                          nullptr, 4,                   /* Default. */
                          1,                            /* Minimum. */
                          Parallel_reader::MAX_THREADS, /* Maximum. */
                          0);
```

```cpp
ulong thd_parallel_read_threads(THD *thd) {
  return THDVAR(thd, parallel_read_threads);   // ★ session 级（THDVAR），不是 GLOBAL
}
```

| 读取点 | 用途 |
|---|---|
| `ha_innodb.cc:16659` | `ha_innobase::records()` → COUNT(*) |
| `ha_innodb.cc:18280` | `ha_innobase::check()` → CHECK TABLE |
| `handler0alter.cc:1473` | `ha_innobase::parallel_scan_init()` |
| `handler0alter.cc:10002` | `ha_innopart::parallel_scan_init()` |
| `ha_innopart.cc:3185` | `ha_innopart::records()` → 分区 COUNT(*) |
| `ddl/ddl0par-scan.cc:105` | `Parallel_cursor::scan()` → DDL 建索引 |

> ★ `ha_innodb.cc:10731`（采样）**故意不读这个变量**，硬编码 `available_threads(1, false)`。

**COUNT(*) 为什么一定要走 `records()`**（`sql/opt_sum.cc:352`）：

```cpp
    ulonglong table_flags = tl->table->file->ha_table_flags();
    if ((table_flags & HA_STATS_RECORDS_IS_EXACT) && tables_filled &&
        !tl->table->force_index) {
      error = tl->fetch_number_of_rows();   // ★ MyISAM / InnoDB 临时表走这条（不并行）
    } else {
      delay_ha_records_to_exec_phase |=
          !(table_flags & HA_COUNT_ROWS_INSTANT) || tl->table->force_index;
      //  ★★ InnoDB 普通表：既无 HA_COUNT_ROWS_INSTANT 也无 HA_STATS_RECORDS_IS_EXACT → true
      have_exact_count = false;
    }
```

```cpp
// ha_innodb.cc:6338 —— 只有临时表才有 HA_STATS_RECORDS_IS_EXACT
  /* Temporary table provides accurate record count */
  if (table_share != nullptr &&
      table_share->table_category == TABLE_CATEGORY_TEMPORARY) {
    flags |= HA_STATS_RECORDS_IS_EXACT;
  }
```

> ★★ 所以 `SELECT COUNT(*) FROM t`（InnoDB 普通表）的执行路径是：优化期 `delay_ha_records_to_exec_phase=true` → 执行期 `UnqualifiedCountIterator::Read()` → `get_exact_record_count()` → `ha_records()` → `ha_innobase::records()` → 并行扫描。**`FORCE INDEX` 会挡掉这条路。**
>
> ⚠️ **`innodb_parallel_read_threads` 完全不参与 CBO 代价估算**——`sql/sql_optimizer.cc`、`opt_costmodel.cc`、`opt_range.cc`、`sql_planner.cc` 里 grep `parallel` **零命中**。
>
> ⚠️ **`ha_innobase::info()` / `ANALYZE TABLE`（cardinality）不用并行扫描**（走 `dict_stats_update()` 随机采样）；但 **`ANALYZE TABLE ... UPDATE HISTOGRAM` / `TABLESAMPLE` 用**（且只申请 1 个线程）。

---

## 他库对比

### SQL 层并行查询怎么实现（以 PolarDB 为例）

**★ 先说社区版 8.0.39 的事实**：`sql/handler.h:4834` 确实有 `parallel_scan_init()` / `parallel_scan()` / `parallel_scan_end()` 三个虚函数，`ha_innodb.h` / `ha_innopart.h` 也确实 override 了，但 **`sql/` 下零调用点**——这条路是给 HeatWave/RAPID 类的 "load" 准备的，**社区版是死代码**。

```cpp
// sql/handler.h:4821
  virtual int parallel_scan_init(void *&scan_ctx [[maybe_unused]],
                                 size_t *num_threads [[maybe_unused]],
                                 bool use_reserved_threads [[maybe_unused]]) {
    return 0;
  }
  virtual int parallel_scan(void *scan_ctx [[maybe_unused]],
                            void **thread_ctxs [[maybe_unused]],
                            Load_init_cbk init_fn [[maybe_unused]],
                            Load_cbk load_fn [[maybe_unused]],
                            Load_end_cbk end_fn [[maybe_unused]]) {
    return 0;
  }
  virtual void parallel_scan_end(void *scan_ctx [[maybe_unused]]) { return; }
```

```cpp
// sql/handler.h:4859 —— 三个回调的签名
  using Load_init_cbk = std::function<bool(
      void *cookie, ulong ncols, ulong row_len, const ulong *col_offsets,
      const ulong *null_byte_offsets, const ulong *null_bitmasks)>;
  using Load_cbk = std::function<bool(void *cookie, uint nrows, void *rowdata,
                                      uint64 partition_id)>;
  using Load_end_cbk = std::function<void(void *cookie)>;
```

**handler 层入口的实现**（`handler0alter.cc:1452`）：

```cpp
int ha_innobase::parallel_scan_init(void *&scan_ctx, size_t *num_threads,
                                    bool use_reserved_threads) {
  ...
  trx_assign_read_view(trx);                  // ★ 并行扫描统一走一致性读

  size_t max_threads = thd_parallel_read_threads(m_prebuilt->trx->mysql_thd);
  max_threads = Parallel_reader::available_threads(max_threads, use_reserved_threads);
  //                                            ★ use_reserved=true 时上限 +16 → 272

  if (max_threads == 0) return (HA_ERR_GENERIC);   // ★ 配额耗尽 → 直接失败，不退化

  auto adapter = ut::new_withkey<Parallel_reader_adapter>(
      UT_NEW_THIS_FILE_PSI_KEY, max_threads, row_len);

  Parallel_reader::Scan_range full_scan{};
  Parallel_reader::Config config(full_scan, m_prebuilt->table->first_index());
  //                                          ★★ 永远只扫聚簇索引

  dberr_t err = adapter->add_scan(trx, config, [=](const Parallel_reader::Ctx *ctx) {
    return (adapter->process_rows(ctx));
  });

  scan_ctx = adapter;
  *num_threads = max_threads;
  build_template(true);
  adapter->set(m_prebuilt);     // ★ 把列偏移/null-bitmask 元数据 + start/finish 回调挂上
  return (0);
}
```

> ★ 分区版 `ha_innopart::parallel_scan_init()`（`handler0alter.cc:10000`）逐分区 `add_scan`，把**分区 id 放进 `Config`**（`Config config(FULL_SCAN, first_index(), 0, i)`），回吐时 `Load_cbk` 的 `partition_id` 参数就是这么来的。

---



> 以下基于阿里云《PolarDB 并行查询深入剖析》（2022/01）。**这是 PolarDB 私有特性，非标准 MySQL 源码**，这里只讲设计思路，作为"如果要做 SQL 层并行该怎么改"的参考。

**两步走优化**：PolarDB 没有重写优化器，而是①先完成 MySQL 原生串行优化，②在其结果之上做并行拆分与并行优化。原因与社区相同——优化器内部耦合深（递归 join ordering、semi-join 策略），侵入式改造不划算。

**physical plan clone + refix**：并行计划不是从零构造，而是从串行计划**克隆 + 修正**：

- **clone**：把串行物理计划的子 slice 结构克隆到各 worker 线程；
- **refix**：把串行计划改成 leader 侧计划——去掉已下推的执行结构，替换成"从 collector table 读 worker 传上来的数据"。

**Exchange 算子**（数据分发）：三种方式——`Gather`（结果收集回 leader）、`Shuffle/Repartition`（按 key 重分发，如 group by key shuffle）、`Broadcast`（小表广播）。实现用 lock-free shared ring buffer 做流水线传输。Exchange 同时是**计划片段的切分点**。

**plan slice 与 worker**：执行计划拆成多个 slice，每个 slice 由一组 worker 并行完成，slice 间通过 Exchange 传递中间结果。

- **并行扫描**：底层是共享存储，不能按物理分片分配，采用**逻辑分区**——在 B+ 树层面切成**远多于 worker 数的细粒度小分片**，worker 用 **round robin** 抢分片（对比社区的"按子树切 + 动态再切"，思路同源但粒度更细）；
- **并行 join**：Parallel Hash Join（build 阶段多 worker 向共享 lock-free hash table 插入）与 Partition Hash Join（先按 join key shuffle，各自构建小 hash table）；
- **子查询并行**：相关子查询用 Pushdown Exec；非相关用 Pushdown Shared（提前并行物化成临时表）。

**并行度（DOP）**：用户设 `max_parallel_degree` 作上限，并行优化器基于统计与代价决定实际并行度，支持自适应执行（资源不足时退回串行 / 降并行度 / 排队）。

**与 Volcano 模型的关系**：不是替代，是扩展——每个 slice 内部仍沿用 MySQL 的迭代器结构，slice 之间通过 Exchange 连接。

---

## 可观测性

| 想看什么 | 手段 |
|---|---|
| 当前并行度 | `innodb_parallel_read_threads`（默认 4，范围 1–256）；设为 **1** 即关闭并行（同步回退） |
| 是否真的并行了 | 没有直接的 status 变量——`Parallel_reader` 不计数。**间接**：`CHECK TABLE`/`COUNT(*)` 大表时对比设 1 与设 8 的耗时 |
| 线程资源 | `s_active_threads` 是全局原子量，不暴露给用户；超限时**静默回退单线程**（不报错） |
| 用在哪些场景 | 只有 7 类入口（见「用途」表），普通 `SELECT` 不走——用 `EXPLAIN` 看不到任何并行痕迹 |
| 二级引擎装载 | `Parallel_reader_adapter` 的批大小 `ADAPTER_SEND_BUFFER_SIZE`（2MB） |

★ **最大的观测问题是"静默"**：超线程上限、非聚簇索引、加锁读、RU 隔离级别——这些情况**都不会报错**，只是静默退回单线程。想确认有没有并行，只能靠耗时对比。

---

## Misc

### ★ 本机制里的工程实现技法

1. **单 mtr + savepoint 逐步放锁**：用一个 mtr 保证切分的原子性（树结构不变），用 `release_block_at_savepoint` 在每个子树边界固化后立即释放路径 latch——兼顾"原子性"与"不长期持锁"。
2. **左闭右开的区间链**：`ranges.back().second = iter` 让相邻区间共享同一个游标对象，既保证不重不漏，又省掉一次拷贝。
3. **持久游标 + `copy_row`**：游标要跨 latch 释放存活，所以整行必须拷进自己的 heap（`store_position` 只存位置，行内容另存）。
4. **分片原子计数器**（`Counter::Shards`）：按 cache line 对齐，每线程一个 shard，消除多核竞争（聚合只在最后做一次）。
5. **乐观预留 + 退回**的并发额度分配（`available_threads`）：先 `fetch_add` 再判断是否超限，避免"检查-再使用"的竞态。
6. **work stealing 的简化版**：不做抢占式偷取，而是"大任务切成小任务放回共享队列"——实现简单，代价是切分要重新持 S 锁。

### 失效与退化场景

| 场景 | 表现 | 判据 |
|---|---|---|
| 树深 < 3 | **不动态再切**（小表切分比扫描还贵） | `m_depth < SPLIT_THRESHOLD(3)` |
| 二级索引 | **完全不支持**（MVCC 判定处 `ut_error`） | `index->is_clustered()` |
| 加锁读 / RU / `LOCK TABLES` / `INSERT...SELECT` | 静默退回单线程 | 五个触发条件 |
| 全局活跃线程超 256 | 静默回退单线程（`available_threads` 返回 0） | `s_active_threads` |
| `innodb_parallel_read_threads=1` | 同步执行，不创建线程 | `m_sync` |
| 数据倾斜（某子树记录远多于其他） | 尾部 range 被标"可拆"，靠动态再切摊平；若仍不均，总耗时取决于**最慢的那个 range** | `split_point` 逻辑 |
| 切分期间遇到 SMO | 不会（持有 `index_s_lock()`），但会**阻塞并发 DDL/写入** | S 锁语义 |
| 分区表 | 按分区切（`m_partition_id`），每个分区独立扫描 | `ha_innopart.cc`；分区作为分片单位的完整机制见 [`../feat/partitioning.md`](../feat/partitioning.md) |

### 易混淆点

- ⚠️ **术语澄清**：PolarDB 资料的 "plan slice" 指**并行执行计划分片**；MySQL 社区源码里的 `REF_SLICE`（`REF_SLICE_WIN_wn`）是**物化临时表的编号**（窗口函数/物化多次读中间结果用，见 [`../server/query/07_optimize/10_plan_refinement.md`](../server/query/07_optimize/10_plan_refinement.md)）——**两者名字都带 slice，含义毫无关系**。
- ⚠️ **`sql/` 里的 `parallel` 绝大多数是复制 MTS**（`replica_parallel_workers`），与查询执行无关。全库搜索 `ParallelIterator` / 并行 join 迭代器：**0 命中**。
- ⚠️ `handler.h` 的 `parallel_scan_init` / `parallel_scan` **不是 SELECT 执行计划**，而是给**二级引擎**的装载通道。
- ⚠️ `MAX_THREADS=256` 是**变量上限**，不是"会用 256 个线程"——默认 4，且实际受 `available_threads` 全局额度限制。
- ⚠️ 并行 `CHECK TABLE` 的"乱序"检测**必须按 range 重置 `prev_tuple`**，否则跨 range 误报。

---

## 参考

**内核月报**

- **2019/10《Parallel Index Scans, One is Better Than Two》**——二级索引并行范围扫描的可扩展性问题（PolarDB 研究）

**阿里云技术文章**

- **《PolarDB 并行查询深入剖析》（遥凌，2022/01）**——两步走优化、physical plan clone、Exchange 算子、plan slice（「他库对比」的直接来源）

**官方文档**

- *MySQL 8.0 Reference Manual → InnoDB Configuration → `innodb_parallel_read_threads`*

**源码**

- `storage/innobase/include/row0pread.h`——常量、`Config`、`Scan_range`、`Ctx`、`Scan_ctx`
- `storage/innobase/row/row0pread.cc`——`create_ranges`（切分核心）、`start_range`、`create_range`、`Ctx::split`、`create_contexts`、`available_threads`、`worker`
- `storage/innobase/include/row0pread-adapter.h` / `row0pread-adapter.cc`——二级引擎装载适配器
- `storage/innobase/row/row0mysql.cc`——并行 COUNT / CHECK / 触发条件

**相关文档**

- 单线程迭代器执行模型：[`../server/query/09_executor_iterator.md`](../server/query/09_executor_iterator.md)
- handler 接口：[`../server/handler.md`](../server/handler.md)
- B+ 树搜索与游标：[`../index/btr.md`](../index/btr.md)
- 二级索引可见性（为什么必须回表）：[`../index/transaction.md`](../index/transaction.md)
- InnoDB 内部执行模型：[`query_graph.md`](query_graph.md)
