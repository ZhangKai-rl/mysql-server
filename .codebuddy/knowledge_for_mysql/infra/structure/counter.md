# MySQL 分片计数器：ib_counter_t 与 Counter::Shards

> 基于 MySQL 8.0.39 源码，剖析 InnoDB 的两代分片计数器（同文件 `ut0counter.h`）：**`ib_counter_t`**（2012 Sunny Bains 创建，RDTSC 散槽 + 每槽独占 cache line + 非原子 `+=`，读 fuzzy）与 **`Counter::Shards`**（8.0 后期，`std::atomic::fetch_add` + `Pad` 对齐，读精确）。两者解决同一问题——多核下高频计数的 **cache line 乒乓（false sharing）**——但走了两条路线：第一代"写最快、读模糊"，第二代"写稍慢、读精确"。
>
> **边界**：本篇讲计数器容器本身。最大用户 `srv_stats_t`（InnoDB 全局性能统计）的字段清单与 `SHOW GLOBAL STATUS` / `SHOW ENGINE INNODB STATUS` 的读出路径见 [`../../innodb/monitor.md`](../../innodb/monitor.md)；`buf_pool_stat_t::m_n_page_gets`（第二代 Shards 用户）见 [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)。

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

### 版本演进

| 版本 | 变化 |
|---|---|
| 2012（5.6 开发期） | Sunny Bains 创建 `ut0counter.h` 的 `ib_counter_t`（第一代：RDTSC 散槽 + 非原子 `+=`） |
| 8.0 后期 | 同文件新增第二代 `Counter::Shards`（真原子 + Pad 对齐，默认 128 片）；新代码（buf_pool 页 get、no-logging mtr、并行扫描）用第二代，`srv_stats_t` 老代码未动 |

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

### 易混淆概念

- **`ib_counter_t`（模糊，非原子写）≠ `Counter::Shards`（精确，原子写）**：同文件两代，不是新旧替换关系，按"读要不要精确"选。
- **`Counter`（分片计数器）≠ `ib_counter_t` 里的 `counter_indexer_t`**：后者只是第一代选择槽位的策略对象。
- **分片计数器 ≠ 原子计数器**：前者解决的是 **cache 乒乓**（性能），后者解决的是**数据竞争**（正确性）；`Shards` 两者都要（原子 + 分片），`ib_counter_t` 只要前者（分片即安全，近似即正确）。
- **`srv_stats`（全局统计结构）≠ `buf_pool->stat`（`buf_pool_stat_t`）**：前者按全库统计（第一代），后者按 pool 统计（第二代 + 原子混合）。

---

## 参考

**经典算法**
- Linux 内核 `percpu_counter`、Java `LongAdder`——分片计数器的同源实现（源码无引用注释，为从实现反推的对应关系）。

**源码**
- `storage/innobase/include/ut0counter.h`（两代全部实现，inline 在头文件）
- `storage/innobase/include/srv0srv.h`（`srv_stats_t` 5 个 typedef）、`storage/innobase/handler/ha_innodb.cc`（`n_rows_read.add(thd_get_thread_id, 1)` 写路径）
- `storage/innobase/include/buf0buf.h`（`m_n_page_gets` 第二代用户）、`storage/innobase/include/mtr0mtr.h`（no-logging mtr 计数）、`storage/innobase/row/row0mysql.cc`（Parallel_reader 计数）

**相关文档**
- 统计值的读出路径（`SHOW GLOBAL STATUS` 的 `Innodb_*`、`SHOW ENGINE INNODB STATUS` 的 ROW OPERATIONS 段）见 [`../../innodb/monitor.md`](../../innodb/monitor.md)
- 数据结构归属与全量清单见 [`../README.md`](../README.md)
