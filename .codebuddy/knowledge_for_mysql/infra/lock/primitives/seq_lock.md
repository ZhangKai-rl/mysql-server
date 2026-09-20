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
