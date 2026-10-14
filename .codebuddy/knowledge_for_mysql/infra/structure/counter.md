# MySQL 分片计数器：ib_counter_t 与 Counter::Shards

> 基于 MySQL 8.0.39 源码，剖析 InnoDB 的两代分片计数器（同文件 `ut0counter.h`）：**`ib_counter_t`**（2012 Sunny Bains 创建，RDTSC 散槽 + 每槽独占 cache line + 非原子 `+=`，读 fuzzy）与 **`Counter::Shards`**（8.0 后期，`std::atomic::fetch_add` + `Pad` 对齐，读精确）。两者解决同一问题——多核下高频计数的 **cache line 乒乓（false sharing）**——但走了两条路线：第一代"写最快、读模糊"，第二代"写稍慢、读精确"。
>
> **边界**：本篇讲计数器容器本身。最大用户 `srv_stats_t`（InnoDB 全局性能统计）的字段清单与 `SHOW GLOBAL STATUS` / `SHOW ENGINE INNODB STATUS` 的读出路径见 [`../../innodb/monitor.md`](../observability/innodb_monitor.md)；`buf_pool_stat_t::m_n_page_gets`（第二代 Shards 用户）见 [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [两代对比](#两代对比)
- [误差分析（fuzzy 到底差多少）](#误差分析fuzzy-到底差多少)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL/InnoDB 的高频统计场景（行操作计数、日志字节数、页读取次数）有一个共同特征：**多核高频并发累加、读时只需近似值**。直接 `std::atomic::fetch_add` 的代价是 MESI 协议下那条 cache line 在所有核之间乒乓；普通 `uint64_t` 又会数据竞争。分片计数器是第三条路：**把逻辑计数器拆成 N 个独立 slot（每个独占一条 cache line），各核写自己的 slot，读时求和**。

### 用途

| 实例 | 版本 | 场景 |
|---|---|---|
| `srv_stats_t`（5 个 typedef） | 第一代 `ib_counter_t` | InnoDB 全局统计：`n_rows_read` 等行操作（64 槽 RDTSC）+ `data_written`/`log_writes`/`n_lock_wait_time` 等（单槽） |
| `buf_pool_stat_t::m_n_page_gets` | 第二代 `Counter::Shards<64>` | buffer pool 页 get 计数（同 struct 其他字段仍 `std::atomic`，按热度选型） |
| `Mtr::m_count_nologging_mtr` | 第二代 `Counter::Shards<128>` | no-logging mtr 计数（`ALTER INSTANCE DISABLE INNODB REDO_LOG` 配套，**要精确**判断能否安全操作 redo） |
| `Parallel_reader` 记录计数 | 第二代 `Counter::Shards<MAX_THREADS>` | 并行扫描各线程扫的行数 |

**★★★ 四个实例的真实定义与调用点**

**① `srv_stats_t`（`srv0srv.h:71`，★ 只有行级计数用 64 槽）**：

```cpp
struct srv_stats_t {
  typedef ib_counter_t<ulint, 64> ulint_ctr_64_t;              /* ★ 唯一的多槽 */
  typedef ib_counter_t<lsn_t, 1, single_indexer_t> lsn_ctr_1_t;
  typedef ib_counter_t<ulint, 1, single_indexer_t> ulint_ctr_1_t;
  typedef ib_counter_t<lint, 1, single_indexer_t> lint_ctr_1_t;
  typedef ib_counter_t<int64_t, 1, single_indexer_t> int64_ctr_1_t;
  ...
  ulint_ctr_64_t n_rows_read;        n_rows_updated;  n_rows_deleted;  n_rows_inserted;
  ulint_ctr_64_t n_system_rows_*;    n_sampled_pages_read;  n_sampled_pages_skipped;
  /* ★ 以上都是 64 槽 —— 用户线程直接递增，并发最高 */
  ulint_ctr_1_t data_written;  log_writes;  buf_pool_reads;  dblwr_writes; ...
  int64_ctr_1_t n_lock_wait_time;
  /* ★ 以上都是单槽 —— I/O 与锁等待，写者少 */
};
```
```cpp
/* ha_innodb.cc:10328 —— ★ 带 index 的 add：用 MySQL 连接线程 id 当分片键，省一次 RDTSC */
srv_stats.n_rows_read.add(thd_get_thread_id(m_prebuilt->trx->mysql_thd), 1);
```
```cpp
/* srv0srv.cc:1611 —— ★ 读点：隐式转换，代码里看不到函数名 */
export_vars.innodb_rows_read = srv_stats.n_rows_read;
```

> ★★ **规律很清晰**：只有"用户 SQL 行级 / 采样页计数"用 64 槽（高频、用户线程写）；**所有 I/O 与锁等待类指标都是单槽**。

**② `buf_pool_stat_t::m_n_page_gets`（`buf0buf.h:2175`，⚠️ 真名核查）**：

```cpp
struct buf_pool_stat_t {
  using Shards = Counter::Shards<64>;

  /** Number of page gets performed; ... this field is NOT protected
  by the buffer pool mutex */
  Shards m_n_page_gets;

  /** Number of read operations. */
  std::atomic<uint64_t> n_pages_read;      /* ★ 次热字段：裸 atomic，不分片 */
  std::atomic<uint64_t> n_pages_written;
```
```cpp
/* buf0buf.cc:3246 —— 分片键 = page_no() */
Counter::inc(buf_pool->stat.m_n_page_gets, page_id.page_no());
```
```cpp
/* buf0buf.cc:505 —— 多 buffer pool 实例汇总用 add(dst, src) */
Counter::add(tot_stat->m_n_page_gets, buf_stat->m_n_page_gets);
/* srv0srv.cc:1614 —— 导出 */
export_vars.innodb_buffer_pool_read_requests = Counter::total(stat.m_n_page_gets);
```
> ⚠️ 真名是 **`m_n_page_gets`**（不是 `buf_page_gets`）；定义在 **`buf0buf.h`**（`buf0stats.h` 里只有 `buf_stat_per_index_t`）。
> ★ 注释 "NOT protected by the buffer pool mutex" 与同结构 `buf0buf.h:2199` 的 "Protected by LRU_list_mutex." 形成完美对照——**源码会显式标注每个字段靠什么保护**。

**③ `mtr_t::Logging::m_count_nologging_mtr`（`mtr0mtr.h:336`，★ 唯一改用 seq_cst 的实例）**：

```cpp
    void init() {
      m_state.store(ENABLED);
      /* We use sharded counter and force sequentially consistent counting
      which is the general default for c++ atomic operation. If we try to
      optimize it further specific to current operations, we could use
      Release-Acquire ordering ... However, sharding looks to be good enough
      for now and we should go for non default memory ordering only with some
      visible proof for improvement. */
      m_count_nologging_mtr.set_order(std::memory_order_seq_cst);   /* ★★ 全仓唯一 */
      Counter::clear(m_count_nologging_mtr);
    }
```
```cpp
/* mtr0mtr.cc:990 —— ★ 为什么"要精确"：这是安全判据，不是统计 */
int mtr_t::Logging::wait_no_log_mtr(THD *thd) {
  auto wait_cond = [&](bool, bool &result) {
    if (Counter::total(m_count_nologging_mtr) == 0) { result = false; return 0; }
```
> ★★★ `total() == 0` 才允许把 redo logging 从 DISABLED 切回 ENABLED。**若读偏小（漏掉一个在跑的 mtr），系统会在还有未落盘页时宣布 crash safe → 数据丢失**。所以必须是"不丢更新"的原子计数 + seq_cst。统计指标读错无所谓，这里读错就是 bug。

**④ 并行校验计数（`row0mysql.cc:4378`，⚠️ 不在 `row0pread.cc`）**：

```cpp
  using Shards = Counter::Shards<Parallel_reader::MAX_THREADS>;   /* = 256 */
  Shards n_recs;  Shards n_dups;  Shards n_corrupt;
  Counter::clear(n_dups);  Counter::clear(n_recs);  Counter::clear(n_corrupt);
  ...
    const auto id = ctx->thread_id();     /* ★ 分片键 = 线程 id */
    Counter::inc(n_recs, id);
    if (cmp > 0) Counter::inc(n_corrupt, id);
    else if (...) Counter::inc(n_dups, id);
  ...
  if (Counter::total(n_dups) > 0) { ... err = DB_DUPLICATE_KEY; }
  *n_rows = Counter::total(n_recs);
```
```cpp
/* row0mysql.cc:4382 —— 并行 count(*)：用 for_each 而非 total()，因为要逐个过滤 */
    Counter::for_each(n_recs, [=](const Counter::Type n) {
      if (n > 0) { *n_rows += n; }
    });
```
> ★★ 分片数 = `MAX_THREADS`（256）= **并发度**，即"最多多少并行线程就分多少片"。`total()` 就是 `for_each` 的一个特例。
> ⚠️ `row0pread.cc` 全文 grep `Counter` **0 命中**——计数在 `row0mysql.cc`；`row0pread.h:106` 只提供 `MAX_THREADS{256}`、`:712` 提供 `thread_id()`。

### 版本演进

| 版本 | 变化 |
|---|---|
| 2012（5.6 开发期） | Sunny Bains 创建 `ut0counter.h` 的 `ib_counter_t`（第一代：RDTSC 散槽 + 非原子 `+=`） |
| 8.0 后期 | 同文件新增第二代 `Counter::Shards`（真原子 + Pad 对齐，默认 128 片）；新代码（buf_pool 页 get、no-logging mtr、并行扫描）用第二代，`srv_stats_t` 老代码未动 | **未检索到编号** |

> ★ **WL 检索结论（如实注明）**：`ut0counter.h` 全文件 grep `WL#` / `worklog` **零命中**；在 MySQL Worklog 库中也未检索到与"分片计数器"对应的独立 WL。判断：两代计数器都是**作为其它特性的内部构件**随同引入的（第一代随 5.6/5.7 的 InnoDB 并发优化，第二代随 8.0 的并行扫描 / no-logging mtr），**没有单独的 Worklog**——不是漏查。

**★ 从代码风格取证的年代证据**（★ 无 git log 时的替代手段）：

| 证据 | 位置 | 说明 |
|---|---|---|
| 创建署名 | `ut0counter.h:32` | `Created 2012/04/12 by Sunny Bains` |
| 版权跨年 | `ut0counter.h:3` | `Copyright (c) 2012, 2024` —— 文件从 2012 活到 8.0.39 发布年 |
| 第一代风格 | `:59` `:148` | `UNIV_NOTHROW` 宏、`ulint/lint/lsn_t` 老 typedef、`template <typename,int> class Indexer` **模板模板参数**（C++03 老派） |
| 第二代风格 | `:221` 起 | `namespace Counter{}`、`using Type = uint64_t`、`constexpr auto`、`std::array/std::function`、**`noexcept` 取代 `UNIV_NOTHROW`**、`size_t COUNT` 非类型参数（不再是 `int N`） |
| **API 断层** | `:194` vs `:333` | 第一代 `operator Type()` 隐式转换取值；第二代**显式 `Counter::total()`** |
| 支撑设施年代 | `ut0cpu_cache.h:3` | `Copyright (c) 2020, 2024` —— `ut::INNODB_CACHE_LINE_SIZE` **2020 年**才抽出为公共常量 |
| 现代 C++ 规范 | `:227` `:230` | `static_assert` + `using Pad = byte[...]` + `{}` 值初始化 |

> ★★ **结论**：第一代（2012，Sunny Bains）是 5.6 时代 NUMA/多核优化产物；第二代是 8.0 中后期（伴随 2020 年的 `ut0cpu_cache.h`）的现代 C++ 重写。**二者共存于同一文件、同一版本，从未替换**。

---

## 理论基础

### 设计思想与权衡

#### 一、问题：cache line 乒乓（false sharing）

多核 CPU 的缓存一致性以 **cache line（通常 64 字节）** 为单位。N 个核往同一个原子变量 `fetch_add` 时，那条 line 要在所有核之间来回传输（MESI 的 invalidate + 抢回），**核越多越慢**。这不是锁的问题——原子变量没有锁，但硬件的缓存协议照样让它成为瓶颈。

#### 二、解法：分片 + cache line 隔离

把逻辑计数器拆成 N 个 slot，每个 slot **独占一条 cache line**；不同核写不同 slot 时互不 invalidate；读时把 N 个 slot 累加。思路与 Linux 内核 `percpu_counter`、Java `LongAdder` 同源。

#### 三、两条路线：非原子换性能 vs 原子换精确

| | 第一代 `ib_counter_t` | 第二代 `Counter::Shards` |
|---|---|---|
| 写 | 普通 `+=`（非原子，靠分片隔离保安全） | `std::atomic::fetch_add`（真原子） |
| 读 | 遍历 N 槽求和（非原子，**fuzzy**） | 原子 load 求和（**精确**） |
| 定位 | **写最快**（`+=` 比 `fetch_add` 快数倍，无需 MESI RMW） | **写稍慢、读精确** |

**"第二代是升级"的说法不严谨**：它是另一条路线。`fetch_add` 要发 Read-Modify-Write 协议消息，比 `+=` 慢；换来的是读精确。选哪代取决于"读要不要精确"。

**★ 两代的写路径源码对照**

```cpp
/* ut0counter.h:145 —— 第一代：普通 -=（非原子） */
  void sub(size_t index, Type n) UNIV_NOTHROW {
    size_t i = m_policy.offset(index);
    ut_ad(i < UT_ARR_SIZE(m_counter));
    m_counter[i] -= n;                       /* ★ 三步 read-modify-write，非原子 */
  }
```
```cpp
/* ut0counter.h:262 —— 第二代：真原子 */
template <size_t COUNT>
inline Type add(Shards<COUNT> &shards, size_t id, size_t n) {
  auto &shard_arr = shards.m_arr;
  auto order = shards.m_memory_order;
  return (shard_arr[id % shard_arr.size()].m_n.fetch_add(n, order));
}
```

**★ 分片数的定义**（`ut0counter.h:51` / `:247`）：

```cpp
/** Default number of slots to use in ib_counter_t */
constexpr uint32_t IB_N_SLOTS = 64;
...
template <size_t COUNT = 128>
struct Shards {
  std::array<Shard, COUNT> m_arr{};
  std::memory_order m_memory_order{Memory_order};
  void set_order(std::memory_order memory_order) { m_memory_order = memory_order; }
};
```
> ★ 第一代默认 **64** 槽，第二代默认 **128** shard。第二代的 memory_order **可运行时覆盖**（`set_order()`），默认 `std::memory_order_relaxed`。

**★ 第一代的槽选择：RDTSC 散列**（`ut0counter.h:55`）：

```cpp
template <typename Type, int N>
struct generic_indexer_t {
  /** @return offset within m_counter */
  static size_t offset(size_t index) UNIV_NOTHROW {
    return (((index % N) + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type)));
  }
};

/** Use the result of my_timer_cycles(), which mainly uses RDTSC for cycles,
to index into the counter array. */
template <typename Type = ulint, int N = 1>
struct counter_indexer_t : public generic_indexer_t<Type, N> {
  enum { fast = 1 };
```
> ★★ `+ 1` 就是**槽 0 占位**——数组第一块 cache line *永不存放有效计数*，这样 `(index % N)` 永远不会取到 0，避免了"槽 0 与数组前面的数据 false sharing"。
> ★ 用 RDTSC（CPU 周期计数）而非 thread id，是因为它**极低开销**（不用调 `pthread_self()`）且分布足够随机。

**★ 第一代的存储与自检**（`ut0counter.h:217`）：

```cpp
  /** Slot 0 is unused. */
  Type m_counter[(N + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type))];
```
```cpp
  bool validate() UNIV_NOTHROW {
#ifdef UNIV_DEBUG
    size_t n = (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type));
    /* Check that we aren't writing outside our defined bounds. */
    for (size_t i = 0; i < UT_ARR_SIZE(m_counter); i += n) {
      for (size_t j = 1; j < n - 1; ++j) {
        ut_ad(m_counter[i + j] == 0);        /* ★ 每个槽的 padding 区必须仍是 0 */
      }
    }
#endif
    return (true);
  }
```
> ★ `(N + 1)` 因子 + debug 期验证 padding 区未被写 —— 这是"严谨"的由来。

**★★ 第二代的对齐：Pad 在计数器之前**（`ut0counter.h:225`，★ 一个容易看错的点）：

```cpp
using N = std::atomic<Type>;
static_assert(ut::INNODB_CACHE_LINE_SIZE >= sizeof(N),
              "Atomic counter size > ut::INNODB_CACHE_LINE_SIZE");
using Pad = byte[ut::INNODB_CACHE_LINE_SIZE - sizeof(N)];

/** Counter shard. */
struct Shard {
  /** Separate on cache line. */
  Pad m_pad;                                /* ★★ padding 在【前面】 */
  /** Sharded counter. */
  N m_n{};                                  /* ★ 计数器在【后面】 */
};
```
> ★★ 布局是 `[Padding 56B][atomic 8B]`，所以每个 `m_n` 落在**自己 cache line 的最后 8 字节**，`m_n` 之间被 padding 隔开 —— **相邻 shard 之间没有 false sharing**。
> ★ 但代价是：**最后一个 shard 的 `m_n` 之后没有尾部 padding**——若 `Shards` 对象后面紧跟被频繁写的数据，末尾 shard 会与之共享 cache line。这是第二代相对第一代 `(N+1)` 因子的小瑕疵。

**★ 第一代的读：遍历求和，注释明说"不 100% 准确"**（`ut0counter.h:198`）：

```cpp
  /* @return total value - not 100% accurate, since it is not atomic. */
  operator Type() const UNIV_NOTHROW {
    Type total = 0;
    for (size_t i = 0; i < N; ++i) {
      total += m_counter[m_policy.offset(i)];
    }
    return (total);
  }
```

---

## 核心实现

### 第一代：ib_counter_t

```cpp
constexpr uint32_t IB_N_SLOTS = 64;

template <typename Type, int N = IB_N_SLOTS,
          template <typename, int> class Indexer = default_indexer_t>
class ib_counter_t {
  void add(Type n) UNIV_NOTHROW {
    size_t i = m_policy.offset(m_policy.get_rnd_index());
    m_counter[i] += n;                       /* 非原子 +=，fuzzy 的来源 */
  }
  void add(size_t index, Type n) UNIV_NOTHROW { /* 有稳定索引（如 thread_id）免 RDTSC */ }
  operator Type() const UNIV_NOTHROW {      /* 聚合：遍历 N 槽求和（非快照） */
    Type total = 0;
    for (size_t i = 0; i < N; ++i) total += m_counter[m_policy.offset(i)];
    return (total);
  }
 private:
  Indexer<Type, N> m_policy;
  /** Slot 0 is unused. */
  Type m_counter[(N + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type))];
};
```

#### 三个关键设计

1. **槽间距 = 整条 cache line，slot 0 占位**：`offset(index) = ((index % N) + 1) * (cacheline / sizeof(Type))`——**+1 跳过 slot 0**（注释 "Slot 0 is unused"），每槽独占一条 cache line，彻底消除伪共享。`validate()` 在 UNIV_DEBUG 下校验 padding 恒为 0（防误写越界破坏隔离）。
2. **RDTSC 当散列源**（`counter_indexer_t`，默认）：`get_rnd_index()` 返回 `my_timer_cycles()`（RDTSC 时间戳）——**不查线程号**，多核 RDTSC 低位的抖动天然把并发写散到不同槽；返回 0 时回退 `ut::this_thread_hash`。`add(index, n)` 版让调用方传显式下标（`ha_innodb.cc` 传 `thd_get_thread_id`），省一次 RDTSC。
3. **N 是模板参数**（默认 64，另有 `single_indexer_t` 单槽版）：`srv_stats_t` 同时实例化 `ib_counter_t<ulint, 64>`（高频热路径）和 `ib_counter_t<ulint, 1, single_indexer_t>`（低频/已串行化场景塌缩单槽，省 64 倍内存和读累加成本）。

### 第二代：Counter::Shards

```cpp
namespace Counter {
using Type = uint64_t;
using N = std::atomic<Type>;
static_assert(ut::INNODB_CACHE_LINE_SIZE >= sizeof(N), "...");

using Pad = byte[ut::INNODB_CACHE_LINE_SIZE - sizeof(N)];  // 64-8 = 56 字节

struct Shard {
  Pad m_pad;     // 56 字节填充在前
  N m_n{};       // 8 字节 atomic 在后 —— 整个 Shard 正好 64 字节
};

template <size_t COUNT = 128>
struct Shards {
  std::array<Shard, COUNT> m_arr{};
  std::memory_order m_memory_order{std::memory_order_relaxed};  // 可 set_order 改
};

inline Type add(Shards<COUNT> &shards, size_t id, size_t n) {
  return shard_arr[id % shard_arr.size()].m_n.fetch_add(n, order);
}
inline Type total(const Shards<COUNT> &shards) noexcept { /* 遍历求和 */ }
// 全套：add/sub/inc/dec/get/total/clear/copy/add(dst,src)/for_each
}
```

**false sharing 的消除手法与第一代同源不同形**：每个 `Shard` 结构体正好 64 字节（56 填充 + 8 原子），相邻 shard 的 `m_n` 间隔固定 64 字节必落不同 cache line。**`std::atomic` 本身不引起 false sharing**——根源是"多变量挤同 cache line 被不同核频繁写"，Pad 正是为此。

> 一个反差细节：`Shard`/`Shards` 均无 `alignas(INNODB_CACHE_LINE_SIZE)`，只保证 8 字节对齐；若数组起始地址 mod 64 落在 8-55 间，**首个 shard 的 `m_n` 会跨两条 cache line**（相邻 shard 间仍无伪共享）。第一代的 slot 0 占位 + (N+1) 因子反而更严谨——新代码不总是处处胜过老代码。

---

## 两代对比

| 维度 | 第一代 `ib_counter_t` | 第二代 `Counter::Shards` |
|---|---|---|
| 写 | 普通 `+=`（非原子） | `fetch_add`（原子，MESI RMW） |
| 读 | 非原子累加，**fuzzy 近似** | 原子 load，**精确** |
| 槽选择 | RDTSC 散列 / 显式 index | 显式 shard key（`id % COUNT`） |
| 默认分片 | 64（单槽版 N=1） | 128 |
| 对齐 | slot 0 占位 + (N+1) 因子（严谨） | Pad 结构体（首个 shard 有跨行瑕疵） |
| API | inc/add/dec/sub/operator Type/operator[] | add/sub/inc/dec/get/total/clear/copy/for_each |
| memory_order | 无（非原子） | 可调（默认 relaxed） |
| 替换关系 | **未被替换**——`srv_stats_t` 仍在用 | 新功能按需引入，两代并存 |

**★ `ut0counter.h` 全貌**（★ 文件仅 **373 行**，两代共存于同一个文件）：

| 行区间 | 内容 |
|---|---|
| 1-33 | GPL 头 + **`@file` 注释（含创建署名）** |
| 50-51 | `IB_N_SLOTS = 64` |
| 53-62 | `generic_indexer_t` |
| 64-91 | `counter_indexer_t`（RDTSC 版） |
| 93-111 | `single_indexer_t` |
| 113 | `#define default_indexer_t counter_indexer_t` |
| **115-218** | **第一代 `ib_counter_t`** |
| **220-371** | **第二代 `namespace Counter`** |

**★★★ 两代 API 哲学的分水岭**（★ 一句话就能分辨）：

- 第一代取值用 **`operator Type()` 隐式转换** —— 所以源码里 `export_vars.innodb_rows_read = srv_stats.n_rows_read;` **看不出是在求和**
- 第二代用 **显式自由函数 `Counter::total()`**

```cpp
/* ut0counter.h:194 —— 第一代：隐式转换，调用点看不出来 */
  /* @return total value - not 100% accurate, since it is not atomic. */
  operator Type() const UNIV_NOTHROW {
    Type total = 0;
    for (size_t i = 0; i < N; ++i) {
      total += m_counter[m_policy.offset(i)];
    }
    return (total);
  }
```
```cpp
/* ut0counter.h:329 —— 第二代：显式函数 */
template <size_t COUNT>
inline Type total(const Shards<COUNT> &shards) noexcept {
  Type n = 0;
  for_each(shards, [&](const Type count) { n += count; });
  return (n);
}
```

**★ 第一代完整定义**（`ut0counter.h:115`，★ 头部注释就是"fuzzy"的唯一出处）：

```cpp
/** Class for using fuzzy counters. The counter is not protected by any
mutex and the results are not guaranteed to be 100% accurate but close
enough. Creates an array of counters and separates each element by the
ut::INNODB_CACHE_LINE_SIZE bytes */
template <typename Type, int N = IB_N_SLOTS,
          template <typename, int> class Indexer = default_indexer_t>
class ib_counter_t {
 public:
  ib_counter_t() { memset(m_counter, 0x0, sizeof(m_counter)); }
  ~ib_counter_t() { ut_ad(validate()); }              /* ★ 仅 debug 生效 */

  static bool is_fast() { return (Indexer<Type, N>::fast); }

  bool validate() UNIV_NOTHROW {
#ifdef UNIV_DEBUG
    size_t n = (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type));
    /* Check that we aren't writing outside our defined bounds. */
    for (size_t i = 0; i < UT_ARR_SIZE(m_counter); i += n) {
      for (size_t j = 1; j < n - 1; ++j) {
        ut_ad(m_counter[i + j] == 0);                 /* ★ 只查 padding 被写脏，不查值 */
      }
    }
#endif
    return (true);
  }
```

> ★ `validate()` 的职责常被误解：它**只探测越界写**（padding 区必须为 0），**不检查计数一致性**，且 `UNIV_DEBUG` 外整段被裁掉。

**★ 第一代写路径的两个重载**（★ 第二个省一次 RDTSC）：

```cpp
  void add(Type n) UNIV_NOTHROW {
    size_t i = m_policy.offset(m_policy.get_rnd_index());   /* ① 无 index：跑一次 RDTSC */
    ut_ad(i < UT_ARR_SIZE(m_counter));
    m_counter[i] += n;
  }

  /** Use this if you can use a unique identifier, saves a
  call to get_rnd_index(). */
  void add(size_t index, Type n) UNIV_NOTHROW {
    size_t i = m_policy.offset(index);                      /* ② 有 index：直接算 */
    ut_ad(i < UT_ARR_SIZE(m_counter));
    m_counter[i] += n;
  }
```

**★ RDTSC 的 plan B**（`ut0counter.h:73`，★ 常被忽略）：

```cpp
  static size_t get_rnd_index() UNIV_NOTHROW {
    size_t c = static_cast<size_t>(my_timer_cycles());
    if (c != 0) return (c);
    else {
      /* We may go here if my_timer_cycles() returns 0,
      so we have to have the plan B for the counter. */
#if !defined(_WIN32)
      return ut::this_thread_hash;    /* ★ 退化成"按线程分片" */
#else
      LARGE_INTEGER cnt;
      QueryPerformanceCounter(&cnt);
      return static_cast<size_t>(cnt.QuadPart);
#endif
    }
  }
```
> ★ `ut::this_thread_hash` 是 `thread_local` 常量（`os0thread.h:73`）——**同一线程恒定命中同一槽**，分片退化但依然无竞争。

**★ `single_indexer_t`（N=1 专用）**：

```cpp
template <typename Type = ulint, int N = 1>
struct single_indexer_t {
  enum { fast = 0 };                     /* ★ 非 fast */

  static size_t offset(size_t index [[maybe_unused]]) UNIV_NOTHROW {
    static_assert(N == 1);
    return ((ut::INNODB_CACHE_LINE_SIZE / sizeof(Type)));   /* 恒返回 1 个跨距 */
  }
  static size_t get_rnd_index() UNIV_NOTHROW {
    static_assert(N == 1);
    return (1);                          /* ★★ "随机数"恒为 1 → 所有线程挤同一槽 */
  }
};
```

**★★ 第二代完整定义**（`ut0counter.h:220`，⚠️ 两处真名纠错）：

```cpp
namespace Counter {
using Type = uint64_t;          /* ★ 元素类型被钉死为 uint64_t（第一代是模板参数） */
using N = std::atomic<Type>;    /* ★ 这里的 N 是"原子类型"，不是分片数 */

static_assert(ut::INNODB_CACHE_LINE_SIZE >= sizeof(N),
              "Atomic counter size > ut::INNODB_CACHE_LINE_SIZE");   /* ★ 唯一 static_assert */

using Pad = byte[ut::INNODB_CACHE_LINE_SIZE - sizeof(N)];   /* ⚠️ 是 using 别名，不是 struct */
/* Pad = byte[64-8] = 56 字节 */

/** Counter shard. */
struct Shard {
  /** Separate on cache line. */
  Pad m_pad;
  /** Sharded counter. */
  N m_n{};
};                              /* sizeof = 56 + 8 = 64 = 一条 cache line */

using Function = std::function<void(const Type)>;

/** Relaxed order by default. */
constexpr auto Memory_order = std::memory_order_relaxed;   /* ⚠️ 真名是 Memory_order，不是 Relaxed */

template <size_t COUNT = 128>
struct Shards {
  std::array<Shard, COUNT> m_arr{};
  std::memory_order m_memory_order{Memory_order};   /* ★ 每个实例自带一份 */
  void set_order(std::memory_order memory_order) { m_memory_order = memory_order; }
};
```

> ⚠️ **两点纠错**：① 不存在 `struct Pad`（是 `using` 类型别名）；② 第二代**完全没有 `alignas`**——"每个 Shard 占 64B"靠 padding 保证，但"数组首地址落在 cache line 起点"并不由这段代码保证。
> ★ 为什么不用 `alignas`？`ut0cpu_cache.h:56` 的注释给出了官方理由：*"Use this, as opposed to alignas(), to avoid problems with allocators which do not handle over-aligned types."*

**★ 第二代的聚合三件套**（★ `add(dst, src)` 是多 buffer pool 汇总的关键）：

```cpp
template <size_t COUNT>
inline void add(Shards<COUNT> &dst, const Shards<COUNT> &src) noexcept {
  size_t i{0};
  for_each(src, [&](const Type count) {
    dst.m_arr[i++].m_n.fetch_add(count, dst.m_memory_order);
  });
}
```

**★ `INNODB_CACHE_LINE_SIZE` 的真名与值**（⚠️ 不在 `ut0ut.h` 也不在 `ut0cpu.h`——后者根本不存在）：

```cpp
/* storage/innobase/include/ut0cpu_cache.h:39 */
namespace ut {
/** CPU cache line size */
#ifdef __powerpc__
constexpr size_t INNODB_CACHE_LINE_SIZE = 128;   /* ★ PowerPC 放宽到 128 */
#else
constexpr size_t INNODB_CACHE_LINE_SIZE = 64;
#endif
```
> ★ 是 `constexpr size_t`（不是宏/枚举），在 **namespace `ut`** 内。`ut0cpu_cache.h` 本身是 2020 年才抽出来的公共常量（版权行 `Copyright (c) 2020, 2024`）——这本身就是"第二代较晚"的证据。

---

## 误差分析（fuzzy 到底差多少）

第一代的 fuzzy 误差有两个来源，**量级不固定**：

1. **读窗口漏写**（次要）：`operator Type()` 遍历 64 槽耗时 200ns-1μs，期间往"已读槽"的写被漏掉。1μs 内全库约发生几次写，漏 0-1 次为主——**个位数级**。
2. **同槽写-写竞争丢更新**（主要）：`m_counter[i] += n` 是 read-modify-write 三步非原子，同槽多线程并发会丢更新（A、B 都 load 100，都 store 101，本该 102）。

| 并发写线程 T（N=64） | 每槽竞争 | 误差量级 |
|---|---|---|
| 64 | 1 | 几乎零（每槽独占） |
| 256 | 4 | 每秒差几十到几百 |
| 1024 | 16 | 丢更新明显，随时间累积 |

但**相对总计数（百万/千万级）的比率仍极低（10^-4 以下）**。且速率比累计值更准：两次 fuzzy 快照相减，误差方向随机近似抵消——所以 `SHOW ENGINE INNODB STATUS` 给的 inserts/s 比单次 `Innodb_rows_read` 累计值误差更小。

**选型判据**：值要参与正确性判断（阈值、归零）→ 第二代 `Shards` 或 `std::atomic`；纯统计展示、容忍近似 → 第一代 `ib_counter_t`（写最快）。`mtr0mtr.h` 的 no-logging mtr 计数选第二代正是因为"要精确判断能否安全操作 redo"。

**★★ 丢更新的本质**（`ut0counter.h:145`，★ 三步非原子的经典例子）：

```
线程 A: load m_counter[i] → 100
线程 B: load m_counter[i] → 100
线程 A: store 100 + 1 = 101
线程 B: store 100 + 1 = 101      ← 本该 102，丢了 1 次
```

> ★ 这正是 `fetch_add` 要避免的：它用 **MESI Read-Modify-Write** 独占该 cache line 完成整个读改写周期，期间别的核无法插入。代价是比普通 `+=` 慢数倍。

**★ 为什么速率比累计值更准**：两次 fuzzy 快照相减时，两个时刻的误差**方向随机**，一阶近似下部分抵消；而单次累计值的误差是**单向累积**的（只可能少算，不可能多算）。所以 `SHOW ENGINE INNODB STATUS` 里的 `inserts/s` 比单次 `Innodb_rows_read` 更可信。

**★ 一个实践推论**：`ib_counter_t` 的误差**恒为负**（只丢不增）。所以用它做"是否达到 N 次"的判断时，**实际次数只会 ≥ 读到的值**——这意味着用它做"上限检查"是**安全的**（不会误判为已达上限），做"下限检查"才是危险的。

**★★★ fuzzy 的三重源码来源**（`ut0counter.h:194`，★ 逐条对应代码）：

```cpp
  /* @return total value - not 100% accurate, since it is not atomic. */
  operator Type() const UNIV_NOTHROW {
    Type total = 0;
    for (size_t i = 0; i < N; ++i) {
      total += m_counter[m_policy.offset(i)];   /* ★ 三重 fuzzy 都在这两行里 */
    }
    return (total);
  }
```

| # | 机制 | 源码 | 后果 |
|---|---|---|---|
| ① | **非原子 RMW 写** | `:153` `m_counter[i] += n` | 同槽并发写**丢失更新**，误差**系统性偏小**；且非原子读改写是**数据竞争（UB）** |
| ② | **非原子读** | `:198` `total += m_counter[...]` | 裸数组读，编译器可缓存/重排，读到陈旧值 |
| ③ | **遍历有先后** | `:197` `for (i=0;i<N;++i)` | 得到的是"不同时刻的拼接快照"，i=0 与 i=N-1 之间可能相差任意长时间 |

> ★ 遍历范围 `i ∈ [0,N)`，经 `offset(i) = ((i%N)+1)*stride` 覆盖 **slot 1..N**，恰好等于写路径用到的全部槽（slot 0 永不用）。

**★ 官方承认 fuzzy 的两处注释**（★ 全仓 grep `fuzzy` 在 `ut0counter.h` 只命中 `:115` 一处）：

```cpp
/* ut0counter.h:115 —— 第一处，"fuzzy" 这个词的唯一出处 */
/** Class for using fuzzy counters. The counter is not protected by any
mutex and the results are not guaranteed to be 100% accurate but close
enough. ... */
```
```cpp
/* ut0counter.h:193 —— 第二处，读路径上的承认（注意是 /* */ 不是 /** */，doxygen 不显示） */
  /* @return total value - not 100% accurate, since it is not atomic. */
```
> ★ 另有一处同类承认在 `srv0mon.h:715`：*"Note: This is not 100% accurate because of the inherent race, we ignore…"*

**★★ 单槽（N=1）反而更 fuzzy**（★ 直觉相反）：

```cpp
  static size_t get_rnd_index() UNIV_NOTHROW { return (1); }   /* ★ "随机数"恒为 1 */
```
> ★ 所有线程无差别地写同一个槽，`m_counter[i] += n` 就是彻头彻尾的非原子并发自增，**丢失更新概率达到最大**。`srv_stats_t` 里 `data_written` / `log_writes` / `buf_pool_reads` 等十几个单槽字段都是这种情况——敢这么做是因为**写者少**（多为后台线程）且指标容忍误差。

**★★ 第二代为什么"精确"——但要区分"精确"与"一致快照"**（`ut0counter.h:325`）：

```cpp
template <size_t COUNT>
inline void for_each(const Shards<COUNT> &shards, Function &&f) noexcept {
  for (const auto &shard : shards.m_arr) {
    f(shard.m_n);      /* ★ std::atomic<uint64_t> 隐式转 uint64_t = 一次原子 load */
  }
}
```

> ★★ 三点：
> ① 每个槽是 `std::atomic`，读是**原子 load**，不会读到撕裂值，也不存在 UB；
> ② **不丢更新** ⇒ 任何时刻求和必然等于"已完成 fetch_add 之和"，**单调、可重复**；
> ③ ⚠️ 但它**不是"原子快照"**——遍历 COUNT 个槽期间其他线程仍在并发写，两次 `total()` 之间若有写，结果依然会变。它保证的是"每个被读到的值都是某个真实瞬时值"，不是"所有槽在同一瞬间被冻结"。
> ⇒ 想要真强一致，只有 `mtr` 那一个用例做了：`set_order(std::memory_order_seq_cst)`。

**★ memory_order 的全链路**（4 个位置）：

| 作用 | 位置 |
|---|---|
| 定义 | `ut0counter.h:244` `constexpr auto Memory_order = std::memory_order_relaxed;` |
| 存到实例 | `:252` `std::memory_order m_memory_order{Memory_order};` |
| 改写接口 | `:256` `void set_order(std::memory_order)` |
| **唯一改写点** | **`mtr0mtr.h:263`** `set_order(std::memory_order_seq_cst)` |

**★ 第一代 `sub()` 没有任何下溢保护**（★ 第二代有）：

```cpp
  void sub(Type n) UNIV_NOTHROW {
    size_t i = m_policy.offset(m_policy.get_rnd_index());
    ut_ad(i < UT_ARR_SIZE(m_counter));
    m_counter[i] -= n;        /* ⚠️ 减到负数也无人报警 */
  }
```
```cpp
/* 第二代对照（ut0counter.h:281）—— debug 下有断言 */
template <size_t COUNT>
inline Type sub(Shards<COUNT> &shards, size_t id, size_t n) {
  ut_ad(get(shards, id) >= n);
  ...
```

### ★ 真名核查

| 项 | 结论 |
|---|---|
| 文件路径 | `storage/innobase/include/ut0counter.h`（两代**同文件**） |
| 创建信息 | `Created 2012/04/12 by Sunny Bains`（`ut0counter.h:33`） |
| 默认分片数 | 第一代 `IB_N_SLOTS = 64`；第二代 `Shards<COUNT = 128>` |
| 第二代 memory_order | 默认 `std::memory_order_relaxed`，**可 `set_order()` 运行时覆盖** |
| 槽选择 | 第一代 `counter_indexer_t`（RDTSC，`:66`）；第二代 `id % shard_arr.size()`（显式 shard key） |
| 占位技巧 | 第一代 `offset() = ((index % N) + 1) * (CACHE_LINE / sizeof(Type))` —— **`+1` 即槽 0 占位** |
| 对齐 | 第一代 `(N+1)` 因子 + debug 校验 padding 未写；第二代 `Shard { Pad m_pad; N m_n; }` —— **padding 在计数器之前** |

> ⚠️ 注意 `ib_counter_t` 的模板默认是 `counter_indexer_t`（RDTSC 版）；另有 `single_indexer_t`（`:95`，N=1 单槽版，`offset()` 直接返回 `CACHE_LINE / sizeof(Type)`，即只有槽 1）。

**★ 三个核心表达式的原文对照**（★ 一眼看清两代的分片与填充策略差异）：

```cpp
/* ── 第一代 ───────────────────────────────────────────── */
/* ut0counter.h:60 —— 槽位偏移：+1 跳过 slot 0，乘跨距保证每槽独占 cache line */
static size_t offset(size_t index) UNIV_NOTHROW {
  return (((index % N) + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type)));
}
/* ut0counter.h:216 —— 数组大小 N+1 份跨距，slot 0 永不存放有效计数 */
/** Slot 0 is unused. */
Type m_counter[(N + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type))];
/* ut0counter.h:153 —— 写：非原子 += */
m_counter[i] += n;

/* ── 第二代 ───────────────────────────────────────────── */
/* ut0counter.h:230 —— 填充：56 字节前置 padding（64 - sizeof(atomic<uint64>)=8） */
using Pad = byte[ut::INNODB_CACHE_LINE_SIZE - sizeof(N)];
struct Shard { Pad m_pad; N m_n{}; };      /* sizeof = 64 */
/* ut0counter.h:271 —— 分片路由 + 原子 RMW，返回旧值 */
return (shard_arr[id % shard_arr.size()].m_n.fetch_add(n, order));
/* ut0counter.h:325 —— 读：std::atomic 隐式 load（原子） */
f(shard.m_n);
```

> ★ 两代在"隔离"这件事上的**手法不同**：第一代靠**跨距乘法**（数组里每个有效槽相隔一条 cache line），第二代靠**结构体填充**（每个元素自身就是 64 字节）。
> ★ 第一代的代价是内存（4160 B/计数器），第二代的代价是 `fetch_add` 的 RMW 开销。

---

## Misc

### 面向二次开发

- **高频近似计数** → 第一代默认 64 槽；**低频/已串行化** → 单槽版（`single_indexer_t`）省内存；**要精确读** → 第二代 `Shards`。
- 第一代有稳定索引时用 `add(index, n)` 免 RDTSC；第二代写要传 shard key（`id % COUNT` 分发）。
- 第二代默认 relaxed；需要更强序时 `set_order`。

**坑与已知缺陷**（源码证据）：

1. 第一代非原子 `+=` 会**丢更新**（类注释自认 "not 100% accurate but close enough"）；`validate()` 只 debug 校验越界。
2. 第一代的 `operator++`/`sum()`/`common_indexer_t` **不存在**（旧资料常见错误）——增量入口是 `inc()/add(n)`，聚合是 `operator Type()`。
3. 第二代首个 shard 可能跨 cache line（缺 `alignas`，见上文）。
4. 第二代 `sub()` 有 `ut_ad(get(shards, id) >= n)` 断言——下溢在 release 下静默回绕。
5. **第一代 `sub()` 连这个断言都没有**（`ut0counter.h:173`）——减到负数无任何保护。

**★★★ 真名勘误全表**（★ 本篇收割中逐一核实）

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `INNODB_CACHE_LINE_SIZE` 在 `ut0ut.h` 或 `ut0cpu.h` | **都错** → `ut::INNODB_CACHE_LINE_SIZE`，在 **`ut0cpu_cache.h:39/41`**；**不存在 `ut0cpu.h`** |
| 第一代读路径是 `value()` / `total()` | **都不存在** → 唯一取值入口是 **`operator Type()`**（`:194`）与 `operator[]`（`:204`） |
| 第一代有 `operator+=` / `operator-=` / `reset()` | **都不存在** → 只有 `inc/add/dec/sub/operator Type/operator[]/validate/is_fast` |
| `buf_pool_stat_t` 在 `buf0stats.h` | **错** → 在 **`buf0buf.h:2175`**；`buf0stats.h` 里只有 `buf_stat_per_index_t` |
| `buf_page_gets` 字段 | 真名 **`m_n_page_gets`**，类型 `Counter::Shards<64>` |
| `Mtr::m_count_nologging_mtr` | 真名 **`mtr_t::Logging::m_count_nologging_mtr`**（`mtr0mtr.h:339`） |
| Parallel_reader 计数在 `row0pread.cc` | **该文件 grep `Counter` 0 命中** → 在 **`row0mysql.cc:4382/4435`** |
| 第二代用 `alignas` 对齐 | **没有** → 用前置填充别名 `using Pad = byte[56]` |
| 存在 `struct Pad` | **不是 struct**，是 `using` 类型别名（`:230`） |
| 第二代默认内存序叫 `Counter::Relaxed` | 真名 **`Counter::Memory_order`**（`:244`） |
| `UT_ARR_SIZE` 在 `ut0counter.h` | 在 **`univ.i:526`** |
| `srv_stats_t` 5 个 typedef 都是多槽 | **只有 `ulint_ctr_64_t` 是 64 槽**，其余 4 个全是 `N=1, single_indexer_t` |
| 单槽（N=1）就"不 fuzzy"了 | **仍然 fuzzy，且更严重**——所有线程打同一槽，`+=` 是非原子 RMW |
| 第二代 `total()` 是"原子快照" | **不是** → 遍历期间仍有并发写；它保证的是"不丢更新"，不是"同一瞬间冻结" |

**★ 两代 API 的完整清单**（★ 便于对照记忆）：

```cpp
// 第一代 ib_counter_t（ut0counter.h:115-218）
  void inc();  void add(Type n);  void add(size_t index, Type n);
  void dec();  void sub(Type n);  void sub(size_t index, Type n);
  operator Type() const;                /* ★ 求和，fuzzy */
  Type operator[](size_t index) const;  /* 单槽读 */
  static bool is_fast();                /* counter_indexer_t=1 / single_indexer_t=0 */
  bool validate();                      /* 仅 debug：查 padding 未被写脏 */

// 第二代 namespace Counter（ut0counter.h:220-371）
  Type add(Shards&, size_t id, size_t n);   /* 返回旧值 */
  Type sub(Shards&, size_t id, size_t n);   /* 有 ut_ad 下溢断言 */
  Type inc(Shards&, size_t id);  Type dec(Shards&, size_t id);
  Type get(const Shards&, size_t id);       /* 单槽 atomic load */
  Type total(const Shards&);                /* 求和，精确 */
  void clear(Shards&);  void copy(Shards&dst, const Shards&src);
  void add(Shards&dst, const Shards&src);   /* ★ 多实例汇总 */
  void for_each(const Shards&, Function&&); /* total() 是它的特例 */
```

**★★ 新增一个分片计数器的完整改动清单**（★ 推荐只用第二代——全仓 `ib_counter_t<` 只剩 `srv0srv.h:72-76` 一处）：

| 步骤 | 位置（参照现有实现） | 改动 |
|---|---|---|
| 1 | 目标头文件，如 `buf0buf.h:2176` | `using Shards = Counter::Shards<64>;` + 声明成员 `Shards m_xxx;` |
| 2 | 同文件 `static void copy()`，`buf0buf.h:2216` | 加 `Counter::copy(dst.m_xxx, src.m_xxx);` |
| 3 | 同文件 `void reset()`，`buf0buf.h:2240` | 加 `Counter::clear(m_xxx);` |
| 4 | 递增点 `.cc`，如 `buf0buf.cc:3246` | `Counter::inc(obj.m_xxx, 分片键);` |
| 5 | 读取点，如 `srv0srv.cc:1614` | `Counter::total(obj.m_xxx)` |
| 6 | 多实例汇总，如 `buf0buf.cc:505` | `Counter::add(tot.m_xxx, one.m_xxx);` |
| 7 | （可选）提序 `mtr0mtr.h:263` | 仅当"计数决定正确性"才 `set_order(seq_cst)`；统计类保持 relaxed |

> ★ **分片键的选型**：优先选**天然分散**的量——`page_no()`（buf pool）、`thread_id()`（并行扫描）、数组下标。`srv_stats` 用的是 `thd_get_thread_id()`。
> ★ **分片数的选型**：全仓只有 4 个 `Counter::Shards` 实例——`buf0buf.h:2176`(64) / `mtr0mtr.h:336`(128) / `row0mysql.cc:4382`(256) / `:4435`(256)。规律：**与并发度对齐**（并行扫描 = MAX_THREADS），或取 64/128 这类 2 的幂。

**★ 一个容易漏的点**：`Counter::Shards` **不能用 `memcpy` 拷贝**——必须用 `Counter::copy()` / `Counter::add()`（逐槽 `store` / `fetch_add`）。`buf_pool_stat_t::copy()`（`buf0buf.h:2216`）就是这么写的。

### 易混淆概念

- **`ib_counter_t`（模糊，非原子写）≠ `Counter::Shards`（精确，原子写）**：同文件两代，不是新旧替换关系，按"读要不要精确"选。
- **`Counter`（分片计数器）≠ `ib_counter_t` 里的 `counter_indexer_t`**：后者只是第一代选择槽位的策略对象。
- **分片计数器 ≠ 原子计数器**：前者解决的是 **cache 乒乓**（性能），后者解决的是**数据竞争**（正确性）；`Shards` 两者都要（原子 + 分片），`ib_counter_t` 只要前者（分片即安全，近似即正确）。
- **`srv_stats`（全局统计结构）≠ `buf_pool->stat`（`buf_pool_stat_t`）**：前者按全库统计（第一代），后者按 pool 统计（第二代 + 原子混合）。

**★★★ 选型的源码依据**（★ 同一个 `buf_pool_stat_t` 里就有四种选择的活样本）：

```cpp
struct buf_pool_stat_t {
  using Shards = Counter::Shards<64>;
  /** Number of page gets performed; ... this field is NOT protected
  by the buffer pool mutex */
  Shards m_n_page_gets;                      /* ① 最热：分片 + 填充 */

  /** Number of read operations. */
  std::atomic<uint64_t> n_pages_read;        /* ② 次热：裸 atomic，不分片不填充 */
  std::atomic<uint64_t> n_pages_written;

  /** Number of read ahead pages that are evicted without being accessed.
  Protected by LRU_list_mutex. */
  uint64_t n_ra_pages_evicted;               /* ③ 已有锁：普通变量 */
};
```

| 场景 | 选择 | 源码证据 |
|---|---|---|
| 已有 mutex 保护 | 普通 `uint64_t` | `buf0buf.h:2199` *"Protected by LRU_list_mutex."* |
| 无锁、中低频写 | `std::atomic<uint64_t>` | `buf0buf.h:2184` |
| 无锁、高频多线程写、仅做统计 | `Counter::Shards<N>` | `buf0buf.h:2181` |
| 无锁、高频、结果**决定正确性** | `Shards` + `set_order(seq_cst)` | `mtr0mtr.h:263` |
| 历史遗留 / 统计容忍误差 | `ib_counter_t` | `srv0srv.h:72` |

> ★★ **判据就是"热度 + 是否决定正确性"**。最热的 `m_n_page_gets` 用 Shards，次热的 `n_pages_read` 就退回裸 atomic——后者写频率低，共享 cache line 也无所谓。

**★★ `m_counter` 为什么是 `N+1`**（★ 这个 +1 常被误读）：

```cpp
/* ut0counter.h:216 */
  /** Slot 0 is unused. */
  Type m_counter[(N + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type))];
/* ut0counter.h:60 */
    return (((index % N) + 1) * (ut::INNODB_CACHE_LINE_SIZE / sizeof(Type)));
```

> ★ 因果链：`offset() = ((index % N) + 1) × stride ∈ [1×stride, N×stride]` ⇒ 最大下标 = N×stride ⇒ 数组必须 ≥ (N+1)×stride。
> ★ **为什么要 +1？** slot 0 若被使用，它会和前一个成员 `m_policy` 共享 cache line；留空一整条 cache line 作"头部隔离带"，保证第一个有效槽从独立 cache line 开始。
> ★ **代价**：`ulint_ctr_64_t` = (64+1) × 8 × 8B = **4160 字节/个计数器**。以 `srv_stats_t` 里 10 个 64 槽字段算，仅这部分就 ~41 KB——这是"用空间换无锁"的账单。
> ★ `UT_ARR_SIZE` 定义在 **`univ.i:526`**（不是 `ut0counter.h`）。

**★ `single_indexer_t` 全仓只有 4 个使用点**（`srv0srv.h:73`）：4 个 `*_ctr_1_t` typedef，服务于 `srv_stats_t` 里所有**非行级**指标。语义是"我知道会撞车，但写者少、能忍"。对照 `counter_indexer_t::fast = 1` / `single_indexer_t::fast = 0`，`is_fast()`（`:127`）就是把这枚举值透出去。

---

## 参考

**经典算法**
- Linux 内核 `percpu_counter`、Java `LongAdder`——分片计数器的同源实现（源码无引用注释，为从实现反推的对应关系）。

**源码**
- `storage/innobase/include/ut0counter.h`（两代全部实现，inline 在头文件）
- `storage/innobase/include/srv0srv.h`（`srv_stats_t` 5 个 typedef）、`storage/innobase/handler/ha_innodb.cc`（`n_rows_read.add(thd_get_thread_id, 1)` 写路径）
- `storage/innobase/include/buf0buf.h`（`m_n_page_gets` 第二代用户）、`storage/innobase/include/mtr0mtr.h`（no-logging mtr 计数）、`storage/innobase/row/row0mysql.cc`（Parallel_reader 计数）

**相关文档**
- 统计值的读出路径（`SHOW GLOBAL STATUS` 的 `Innodb_*`、`SHOW ENGINE INNODB STATUS` 的 ROW OPERATIONS 段）见 [`../../innodb/monitor.md`](../observability/innodb_monitor.md)
- 数据结构归属与全量清单见 [`../README.md`](../README.md)
