# Seq_lock：回调式序号锁（seqlock）

> 基于 MySQL 8.0.39 源码，剖析 InnoDB 的序号锁 `Seq_lock<T>`（`ut0seq_lock.h`）：**读者完全无锁**（不写任何共享行），写者靠外层互斥保证单写；源码注释引用 HP 报告 **HPL-2012-68**（Hans-J. Boehm《Can Seqlocks Get Along With Programming Language Memory Models?》Figure 6），是 InnoDB 无锁件中少数**有论文出处的**。
>
> **边界**：本篇讲 `Seq_lock` 这个**同步原语本身**（seqlock 是"锁模式"——读多写极少、多字段一致快照——与 RCU 同属读者无锁的锁原语家族，故归入 `primitives/`）；它唯一的用户 `mt_fast_modulo_t`（hash 表快速取模）与 `hash_table_t` 的连接见 [`../../structure/hash.md`](../../structure/hash.md) 第三篇。

## 目录

- [概述](#概述)
- [核心实现](#核心实现)
- [唯一实例：mt_fast_modulo_t](#唯一实例mt_fast_modulo_t不是-lsn)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

```cpp
/** @file include/ut0seq_lock.h
 Implements a sequential lock structure for non-locking atomic read/write
 operations on a complex structure. */
```

seqlock（序号锁）：**读者完全无锁**（不写任何共享行），写者靠外层互斥保证单写。InnoDB 版是**回调式**——不暴露 `T*`，而是 `read(Op&&)` / `write(Op&&)` 把私有 `m_value` 作为参数传给 lambda：

```cpp
/** A class that allows to read value of variable of some type T atomically ...
The type T has to be composed of std::atomic fields only. That is because
read(op_r) might read it in parallel to write(op_w). ...
Inspired by https://www.hpl.hp.com/techreports/2012/HPL-2012-68.pdf Figure 6. */
template <typename T>
class Seq_lock : private Non_copyable {
 public:
  template <typename Op> void write(Op &&op) {
    const auto old = m_seq.load(std::memory_order_relaxed);
    /* The odd value means someone else is executing the write operation
    concurrently, and this is not allowed. */
    ut_ad((old & 1) == 0);
    m_seq.store(old + 1, std::memory_order_relaxed);      /* 奇数 = 写进行中 */
    std::atomic_thread_fence(std::memory_order_release);
    op(m_value);
    m_seq.store(old + 2, std::memory_order_release);      /* 偶数 = 稳定窗口 */
  }

  template <typename Op> auto read(Op &&op) const {
    int try_count = 0;
    while (true) {
      const auto seq_before = m_seq.load(std::memory_order_acquire);
      if ((seq_before & 1) == 1) {
        /* Someone is currently writing ... Try a few times to read the seq
        value, if this not help, try to yield execution. */
        if ((++try_count & 7) == 0) std::this_thread::yield();
        continue;
      }
      auto res = op(m_value);
      std::atomic_thread_fence(std::memory_order_acquire);
      const auto seq_after = m_seq.load(std::memory_order_relaxed);
      if (seq_before == seq_after) return res;            /* 前后一致 = 拿到连贯快照 */
    }
  }
 private:
  T m_value;
  /** Sequence count. Even when the value is ready for read, odd when the value
  is being written to. */
  std::atomic<uint64_t> m_seq{0};
};
```

### 协议

⚠️ **先纠正名字**：真身是 **`ut::Seq_lock<T>`**，在 **`storage/innobase/include/ut0seq_lock.h:49`**（**不是 `include/seq_lock.h` / `sql/seq_lock.h`**）。**不存在 `begin_read()` / `end_read()` / `seqlock_read()`**——只有两个成员：`write(op)` 和 `read(op)`。

**完整实现**（`ut0seq_lock.h:49`）——★ 读写都是回调式：

```cpp
template <typename T>
class Seq_lock : private Non_copyable {     // ★ 私有继承 Non_copyable：禁拷贝/禁赋值
 public:
  Seq_lock() = default;
  template <typename... Args>
  Seq_lock(Args... args) : m_value{std::forward<Args>(args)...} {}

  /** Writes a new value for the variable of type T. The op can use
  memory_order_relaxed stores.
  NOTE: The user needs to synchronize all calls to this method. */
  template <typename Op>
  void write(Op &&op) {                     // ★★ 写也是回调式
    const auto old = m_seq.load(std::memory_order_relaxed);
    /* The odd value means someone else is executing the write operation
    concurrently, and this is not allowed. */
    ut_ad((old & 1) == 0);                  // ★ 进入时必须为偶数；写者之间需外部互斥（本类不保证）
    m_seq.store(old + 1, std::memory_order_relaxed);   // ① 置奇数 = "正在写"
    /* This fence is meant to synchronize with the fence in read(), whenever
    op() in read() happens to load-from any of the values stored by our op().
    Then it would follow that load to seq_after will happen-after our
    first fetch_add(1), which is all we need. See 32.9.1 of C++17 draft. */
    std::atomic_thread_fence(std::memory_order_release);   // ② release fence：阻止 op() 的写上移到 ① 之前
    op(m_value);                            // ③ 执行用户回调（可用 relaxed store）
    m_seq.store(old + 2, std::memory_order_release);       // ④ 回到偶数，写完成
  }

  /* Reads the value of the stored value of type T using operation op().
  ... The op() can't assume the data stored inside T is logically consistent.
  Calls to this method don't need to be synchronized. */
  template <typename Op>
  auto read(Op &&op) const {                // ★ const；返回类型由 op 推导
    int try_count = 0;
    while (true) {
      const auto seq_before = m_seq.load(std::memory_order_acquire);  // ① acquire 读起始序号
      if ((seq_before & 1) == 1) {          // ② 奇数：有写者正在写，本次不读
        /* Someone is currently writing to the stored value. Try a few times to
        read the seq value, if this not help, try to yield execution. */
        if ((++try_count & 7) == 0) {
          std::this_thread::yield();        // ★ 每 8 次忙等才 yield 一次（背压退避）
        }
        continue;
      }
      auto res = op(m_value);               // ③ 执行用户回调（可用 relaxed load）
      std::atomic_thread_fence(std::memory_order_acquire);            // ④ acquire fence：阻止 op() 的读被下移到 ⑤ 之后
      const auto seq_after = m_seq.load(std::memory_order_relaxed);   // ⑤
      if (seq_before == seq_after) {
        return res;                         // ★★ 前后序号相同且为偶数 → 读到一致快照
      }
      /* The begin and end seq number is different ... Let's just try again.
      We do not want to yield here, as the new seq value is already set. */
    }
  }

 private:
  /** Stored value. */
  T m_value;
  /** Sequence count. Even when the value is ready for read, odd when the value
  is being written to. */
  std::atomic<uint64_t> m_seq{0};           // ★ 唯一的同步变量
};
```

> ★★ **协议要点**：`m_seq` **偶数 = 稳定可读，奇数 = 正在写**。读端"读前记序号 → 读 → 读后记序号 → 相同则接受"，不同则**重试**（不 yield，因为新序号已经设好了）。
> ★ **写者之间不互斥**——`ut_ad((old & 1) == 0)` 只是断言，注释明说 "The user needs to synchronize all calls to this method"。
> ★ 两个 `atomic_thread_fence`：写侧 **release**（防止 op 的写上移），读侧 **acquire**（防止 op 的读下移）。**注意用的是 fence 不是 load/store 的内存序**——因为 op 内部用的是 relaxed。

**★ 正确性证明的官方推导**（`read()` 后那段长注释）值得抄：

> "Suppose that one of the loads inside `op()` saw value written by some writer `w` … Since `seq_before` was assigned using **acquire load**, it synchronizes-with the store from which it loads, meaning that `op()` has to happen-after `w`'s final increment of seq. Thus we see that if `op()` sees any part of the value written by `w`, then all the other parts of value it sees are at least as fresh."

**基类 `Non_copyable`**（`ut0class_life_cycle.h:41`）：

```cpp
class Non_copyable {
 public:
  Non_copyable(const Non_copyable &) = delete;
  Non_copyable &operator=(const Non_copyable &) = delete;
 protected:
  Non_copyable() = default;
  ~Non_copyable() = default;  /// Protected non-virtual destructor
};
```

**写者**：seq 偶数 → +1（奇数，宣告进入）→ release fence → 写数据（relaxed）→ +1（偶数，宣告完成）；**内部没有任何锁**，注释明说 "The user needs to synchronize all calls to this method"——写者互斥由外层保证。

**读者**：奇数则自旋（每 8 次 yield 一次），偶数则 relaxed 读 + acquire fence + 复验 seq，不等则整个重读。

**正确性论证**（注释全文保留在源码）：若 `op()` 读到某写者 w 的任一字段，acquire fence 与 w 的 release fence 同步 → `seq_after` happen-after w 的首次 +1；又 `seq_after` 为偶数且等于 `seq_before` → 全部字段来自同一连贯版本。

---

## 核心实现

### 与内核 seqlock 的差异

| | Linux 内核 seqlock | InnoDB `Seq_lock` |
|---|---|---|
| 读者重试时 | **会写**（`read_seqbegin` 失败时 `read_seqretry` 会 spin 在 seq 上） | **零写**：读路径不写任何共享行，无 cache line 抖动 |
| 形式 | 裸 API（begin/retry/end） | **回调式**：`read(Op&&)` 把值传给 lambda，不暴露 `T*` |
| 写者互斥 | 内核 spinlock | **无内建锁**，`ut_ad((old & 1) == 0)` 抓多写者并发（外层保证） |

### 三个使用前提（选它必须同时满足）

**① T 必须只由 `std::atomic` 字段构成** —— doxygen 原话：*"The type `T` has to be composed of `std::atomic` fields only. That is because `read(op_r)` might read it in parallel to `write(op_w)`."*

**② 写者之间必须外部互斥** —— `ut_ad((old & 1) == 0)` 只是 debug 断言，注释：*"The user needs to synchronize all calls to this method."*

**③ 读操作必须幂等/可重试** —— `read()` 的 `while(true)` 重试时 `op()` 会被**重复执行**。

---

## 唯一实例：`ut::mt_fast_modulo_t`

⚠️ **勘误**：真名是 **`ut::mt_fast_modulo_t`**（`storage/innobase/include/ut0math.h:235`），带 `ut::` 命名空间。

```cpp
/** A class that allows to atomically set new modulo value for fast modulo
computations. */
class mt_fast_modulo_t : private Non_copyable {
 public:
  mt_fast_modulo_t() : m_data{0ULL, 0ULL} {}
  explicit mt_fast_modulo_t(uint64_t mod)
      : m_data{mod, fast_modulo_t::precompute_inv(mod)} {}

  fast_modulo_t load() {                     // ★ 读：lambda 只读 const data_t&
    return m_data.read([](const data_t &stored_data) {
      return fast_modulo_t{stored_data.m_mod.load(std::memory_order_relaxed),
                           stored_data.m_inv.load(std::memory_order_relaxed)};
    });
  }

  void store(uint64_t new_mod) {             // ★ 写：lambda 拿 data_t&，用 relaxed store
    const fast_modulo_t new_fast_modulo{new_mod};
    const auto inv = new_fast_modulo.get_inverse();  // ★ 逆元在锁外预先算好，缩短临界区
    m_data.write([&](data_t &data) {
      data.m_mod.store(new_mod, std::memory_order_relaxed);
      data.m_inv.store(inv, std::memory_order_relaxed);
    });
  }

 private:
  struct data_t {
    std::atomic<uint64_t> m_mod;
    std::atomic<uint64_t> m_inv;
  };                              // ★★ 满足前提①：全是 atomic

  Seq_lock<data_t> m_data;
};
```

**★ 为什么必须用 seqlock**（`hash0hash.h:479` 的官方注释直说）：

```cpp
  /** Utility to calculate the modulo n_cells fast. It is set together with
  n_cells. It can be read without latches in parallel to set_n_cells, and as it
  is a complex object, it is not set atomically. Because of this the
  multi-threaded version is used. */
  ut::mt_fast_modulo_t n_cells_fast_modulo;
```

> ★★ `data_t` 是**复合对象**（`m_mod` + `m_inv` 两个原子量）。单个 `store` **无法原子地同时更新两者**——读端可能在 `m_mod` 已更新而 `m_inv` 尚未更新时读到**"撕裂"的组合**（mod 与逆元不匹配 → 取模结果错误）。seqlock 让读者无锁重试到前后 seq 一致，从而拿到**配对的 `(mod, inv)`**。

**全仓只有 3 处使用点**：

```cpp
// (1) hash0hash.h:439 —— 读写封装
  const ut::fast_modulo_t get_n_cells_fast_modulo() {
    return n_cells_fast_modulo.load();       // ★ 无 latch 也能安全 peek
  }
  void set_n_cells(size_t n) {
    ut_ad(type == HASH_TABLE_SYNC_NONE || hash_lock_has_all_x(this));   // ★★ 写端必须在所有 shard 的 X-latch 下
    n_cells.store(n);
    n_cells_fast_modulo.store(n);
  }
```

```cpp
// (2) hash0hash.ic:59 —— 实际取模
static inline uint64_t hash_calc_cell_id(uint64_t hash_value,
                                         hash_table_t *table) {
  return hash_value % table->get_n_cells_fast_modulo();
}
```

> ★ `hash_calc_cell_id` 又被 **`locksys::Latches::Page_shards::get_shard()`**（`lock0latches.cc:36`）复用——这是 seqlock 落到**锁分片**的关键路径：
> ```cpp
>   return hash_calc_cell_id(lock_rec_hash_value(page_id), lock_sys->rec_hash) % SHARDS_COUNT;
>   //     ★ 先算 hash cell id（走 seqlock 保护的 fast_modulo），再 % 512 归并到 shard
>   //       这样"同一 hash 桶 = 同一 lock queue"的两个 page 必然落在同一 shard
> ```

**配套的 `fast_modulo_t::compute`（全程无除法）**（`ut0math.h:197`）：

```cpp
  /** Computes the value of x % mod. */
  uint64_t compute(uint64_t x) const {
    uint64_t hi;
    (void)multiply_uint64(x, m_inv, hi);    // ★ x * M 的 128 位结果，取高 64 位 ≈ floor(x/mod)
    const uint64_t guess = hi * m_mod;
    const uint64_t rest = x - guess;
    return rest - (m_mod <= rest) * m_mod;  // ★ 结果落在 [0,2mod)，再至多减一次 mod
  }

  /** Precomputes the inverse needed for fast modulo operations. */
  static uint64_t precompute_inv(uint64_t mod) {
    if (mod == 1) return ~uint64_t{0};      // pedantic matter
    else return divide_128(1, 0, mod);      // ★ = floor(2^64 / mod)
  }
```

**★ 与内核 seqlock 的差异**：内核 seqlock 通常保护"任意数据结构 + 自旋等待写者完成"；MySQL 这版要求 **T 全是 atomic**（否则并发读写非原子字段就是 UB），且**读侧遇奇数不等待而是重试**（只在连续 8 次失败后才 `yield()`）。

1. **读多写极少**：读者反复重试的代价靠"几乎无写"摊薄
2. **单写者**：写者互斥自备，`ut_ad` 只 debug 抓
3. **T 全 atomic 字段**：`read` 可能与 `write` 并行读 `m_value`，非原子字段会撕裂

> 读者**活锁**风险（源码证据）：写者持续写入时读者反复重试，靠每 8 次 yield 缓解——单写者 + 短临界区是隐含前提。

---

## 唯一实例：mt_fast_modulo_t（不是 LSN）

全仓 `Seq_lock<` 仅一处实例化——**快速取模**（`ut0math.h`）：

```cpp
/** A class that allows to atomically set new modulo value for fast modulo computations. */
class mt_fast_modulo_t : private Non_copyable {
 public:
  fast_modulo_t load() {
    return m_data.read([](const data_t &stored_data) {
      return fast_modulo_t{stored_data.m_mod.load(std::memory_order_relaxed),
                           stored_data.m_inv.load(std::memory_order_relaxed)};
    });
  }
  void store(uint64_t new_mod) {
    const fast_modulo_t new_fast_modulo{new_mod};
    const auto inv = new_fast_modulo.get_inverse();
    m_data.write([&](data_t &data) {
      data.m_mod.store(new_mod, std::memory_order_relaxed);
      data.m_inv.store(inv, std::memory_order_relaxed);
    });
  }
 private:
  struct data_t {
    std::atomic<uint64_t> m_mod;
    std::atomic<uint64_t> m_inv;
  };
  Seq_lock<data_t> m_data;
};
```

**为什么这里需要 seqlock**（hash0hash.h 注释原文）：`{m_mod, m_inv}` 两个字段必须同版本（单 `atomic<size_t>` 装不下），且"**It can be read without latches in parallel to set_n_cells**"——哈希表（`hash_table_t` 的快速取模）每次探测都要无锁读快速取模参数，写者只在 `set_n_cells`（"can be used only when holding x-latches on all shards"）改它。完美命中 seqlock 的"读多写极少 + 多字段一致快照 + 单写者"三个前提。详见 [`../../structure/hash.md`](../../structure/hash.md) 第三篇。

**澄清两个易错点**：① `sn_t` 只是 `uint64_t` 的 typedef（log0types.h），**不是** seqlock 用户；② `log_sn_lock`（PFS rwlock key）是 `log.sn` 的 **debug-only rwlock**，与 seqlock 无关——`sn` 的并发协议是最高位自旋锁位（`SN_LOCKED`），单 8 字节直接原子即可，不需要 seqlock。

---

## Misc

### 面向二次开发

- 新用户必须满足三前提（读多写极少、单写者、T 全 atomic 字段）；写者互斥自备，`ut_ad((old & 1) == 0)` 会抓住多写者并发。

### 易混淆概念

- **`Seq_lock` ≠ 内核 seqlock 的直接移植**：内核版读重试会写；本版回调式、读路径零写，但写并发时读者可能活锁。
- **`Seq_lock`（锁模式）≠ `Link_buf` / `ib_counter_t`（数据结构）**：它是"读多写少的多字段一致快照"锁原语，与 RCU（[`rcu.md`](rcu.md)）同属"读者无锁"家族；其余无锁件在 [`../../structure/`](../../structure/)。
- **`log_sn_lock` 不是 seqlock**：是 debug rwlock；`sn` 用最高位自旋锁位。

---

## 参考

**论文（源码注释自述）**
- H.-J. Boehm. *Can Seqlocks Get Along With Programming Language Memory Models?* HP Labs **HPL-2012-68**（`Seq_lock` 注释引用的 Figure 6，且内存序证明引 C++17 草案 §32.9.1）

**源码**
- `storage/innobase/include/ut0seq_lock.h`（全部实现 inline）
- `storage/innobase/include/ut0math.h`（`mt_fast_modulo_t` 实例）、`hash0hash.h`（seqlock 选型注释）

**相关文档**
- 唯一用户 `mt_fast_modulo_t` 的宿主 `hash_table_t`（快速取模热路径、`set_n_cells` 的持锁前提）见 [`../../structure/hash.md`](../../structure/hash.md) 第三篇
- 同族读者无锁方案 RCU（`MyRcuLock`）见 [`rcu.md`](rcu.md)；锁盘点与归属判据见 [`../README.md`](../README.md)
