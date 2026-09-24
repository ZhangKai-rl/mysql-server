# MySQL 有界 MPMC 无锁队列：一个算法的两种裁剪

> 基于 MySQL 8.0.39 源码，将 server 侧 `Integrals_lockfree_queue` 与 InnoDB 侧 `mpmc_bq` 两个有界 MPMC 无锁队列糅合为一篇：两者都是 **Vyukov 式单调 ticket 环形队列**（单调索引 + 槽状态交接），但按"需求半径"做了两种裁剪——server 版**只装整型**、用 `Null`/`Erased` 哨兵值 + head/tail 的 MSB 占用位，带迭代器与软删除全套 API；InnoDB 版**装任意类型**、用槽 seq 与 ticket 差值判空满，是最小 MPMC。核心结论：**需求半径决定 API 半径**。
>
> **边界**：本篇讲两个队列容器本身。`Integrals_lockfree_queue` 的唯一用户 `Bgc_ticket_manager`（binlog 组提交 ticket 机制）见 [`../../server/replication/binlog.md`](../../server/replication/binlog.md)；`mpmc_bq` 的用户（dblwr 刷写段池、FTS 并行建索引）只在篇内简述。

## 目录

- [总览：一个算法，两种裁剪](#总览一个算法两种裁剪)
- [元素与容量：装什么、多大](#元素与容量装什么多大)
- [索引与空满判定：怎么知道空还是满](#索引与空满判定怎么知道空还是满)
- [互斥与交接：谁写哪个槽](#互斥与交接谁写哪个槽)
- [push/pop 逐段剖析](#pushpop-逐段剖析)
- [状态与语义：无锁的代价](#状态与语义无锁的代价)
- [高级 API 与遍历](#高级-api-与遍历)
- [用户全景](#用户全景)
- [可观测性](#可观测性)
- [选择指南](#选择指南)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [面向二次开发](#面向二次开发)
- [坑与已知缺陷汇总](#坑与已知缺陷汇总)
- [易混淆概念汇总](#易混淆概念汇总)
- [参考](#参考)

---

## 总览：一个算法，两种裁剪

### 全量盘点

| 实现 | 层 | 元素 | 空满判定 | 互斥手段 | API | 用户 |
|---|---|---|---|---|---|---|
| `Integrals_lockfree_queue` | server（`sql/containers/`） | **仅整型**（static_assert） | 虚拟索引：`head==tail` 空 / `tail==head+capacity` 满 | head/tail 的 **MSB 占用位** CAS | 迭代器、`erase_if`、`front/back` | `Bgc_ticket_manager::m_sessions_per_ticket`（容量 1024） |
| `mpmc_bq` | InnoDB（`ut0mpmcbq.h`） | 任意类型（`T` 拷贝进槽） | 槽 seq 与 ticket 差值：`diff==0` / `diff<0` | 槽 seq CAS + ticket 发号 | 只有 enqueue/dequeue/empty | dblwr `Segments`/`Batch_segments`、FTS `Docq` |

**一句话**：**mpmc_bq 是"能装指针的最小 MPMC"，Integrals_lockfree_queue 是"装整型的全套 API MPMC"**——一个给引擎热路径（对象池），一个给 SQL 层机制（ticket 计数）。

### 同族异路对比

| 维度 | `Integrals_lockfree_queue`（server） | `mpmc_bq`（InnoDB） |
|---|---|---|
| 元素 | **仅整型**（static_assert） | 任意类型（`T` 拷贝进槽） |
| 空满判定 | 虚拟索引：`head==tail` 空 / `tail==head+capacity` 满 | 槽 seq 与 ticket 差值：`diff==0` / `diff<0` |
| 状态编码 | `Null`/`Erased` 哨兵值 | 槽 seq 本身 |
| 互斥手段 | head/tail 的 **MSB 占用位** CAS | 槽 seq CAS + ticket 发号 |
| 取模 | `% capacity`（**不要求 2 的幂**） | 位掩码（**必须 2 的幂**） |
| 高级 API | 迭代器、`erase_if` 软删除、`front/back` | 无（只有 enqueue/dequeue/empty） |
| 状态返回 | thread_local `enum_queue_state`（`get_state()`） | bool 返回值 |

### 版本演进

| 版本 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 8.0.28 | 与 Bgc_ticket_manager 一同引入（组复制视图变更屏障的配套设施）——**不是 8.0.27**（旧资料常见错误）。⚠️ 判据不能用文件头版权：该文件头写的是 `Copyright (c) 2020, 2024`（含后续修改年份），**不是**引入年份 | 早已存在（dblwr 无锁化、FTS 并行建索引的底座） |

### 理论出身

- `mpmc_bq` 的源码注释**自述出处**：Dmitry Vyukov 的 1024cores.net 博客 *Bounded MPMC queue*——**博客而非论文**，但这是源码自述，诚实照录。
- `Integrals_lockfree_queue` **无源码论文引用**——Vyukov 式单调 ticket 环形队列为并发工程常见手法；MSB 占用位 + 哨兵值 CAS 是自旋锁与值交换的常规组合。**不要硬安论文**。

### 阅读路径

本篇按**设计维度**组织，每个维度下两个实现并排对照。只想读懂某一个实现：按目录找它的 `###` 小节连续读；想理解"为什么这样设计"：从任一维度的"对照"小表出发。

---

## 元素与容量：装什么、多大

### Integrals_lockfree_queue：仅整型 + 哨兵值 + 容量任意

文件头注释原文：

> Lock-free, fixed-size bounded, multiple-producer (MP), multiple-consumer (MC), circular FIFO queue **for integral elements**.
> Monotonically ever increasing virtual indexes are used to keep track of the head and tail pointer... Head is the pointer to the virtual index of the first position that holds an element to be popped, if any. Tail is the pointer to the virtual index of the first available position to push an element to, if any.

关键约束直接写在类型参数上——**只装整型**：

```cpp
template <typename T, T Null = std::numeric_limits<T>::max(), T Erased = Null,
          typename I = container::Padded_indexing<T>,
          typename A = std::nullptr_t>
class Integrals_lockfree_queue {
  static_assert(std::is_integral<T>::value,
    "class `Integrals_lockfree_queue` requires an integral type...");
```

**为什么只支持整型**：元素槽是 `std::atomic<T>`，靠与 `Null`/`Erased` 哨兵值的 CAS 交接——这要求 T 平凡可原子读写，而哨兵作为模板值参数只能是整型常量。整型队列零对象生命周期管理，是无锁的前提。

数据结构：

```cpp
 private:
  size_t m_capacity{0};                        /* 队列容量 */
  array_type m_array;                          /* Atomics_array<T>：std::atomic<T> 数组，可带 padding */
  atomic_type m_head{0};                       /* Aligned_atomic<unsigned long long>：缓存行对齐 */
  atomic_type m_tail{0};
```

- `I` 索引布局：默认 `Padded_indexing`（槽间 padding 防伪共享）或 `Interleaved_indexing`；
- 构造：`m_array{size, null_value}`——所有槽初始化为 `Null`；
- **无"空槽哨兵"**：成员关系完全由 head/tail 决定（"No evaluation of the value held in the given position is made"）；
- 容量 **不要求 2 的幂**（`% capacity` 取模），但构造时固定不可变。

### mpmc_bq：任意类型 + 槽 seq + 容量必须 2 的幂

```cpp
/** Multiple producer consumer, bounded queue
 Implementation of Dmitry Vyukov's MPMC algorithm
 http://www.1024cores.net/home/lock-free-algorithms/queues/bounded-mpmc-queue */
template <typename T>
class mpmc_bq {
 private:
  struct Cell {
    std::atomic<size_t> m_pos;             /* 槽的 seq 状态 */
    T m_data;
  };
  Pad m_pad0;
  Cell *const m_ring;
  size_t const m_capacity;                 /* n_elems - 1，作位掩码 */
  Pad m_pad1;
  std::atomic<size_t> m_enqueue_pos;       /* 单调 ticket，64 位回绕才溢出 */
  Pad m_pad2;
  std::atomic<size_t> m_dequeue_pos;
  Pad m_pad3;
```

- **元素拷贝语义**（`T const&` 拷进槽，"it will be copied"），生命周期由使用者管理；
- **容量必须 2 的幂**（`ut_a((n_elems >= 2) && ((n_elems & (n_elems - 1)) == 0))`），位掩码 `pos & (n_elems-1)` 取下标；
- **队列头尾四个 Pad 防伪共享**（ring / enqueue_pos / dequeue_pos 各被 cache line 隔开）。

### 对照：元素与容量

| 维度 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 元素 | 仅整型（哨兵值 CAS 交接的前提） | 任意类型（拷贝语义） |
| 槽初始值 | `Null` 哨兵 | `m_pos = i`（初始 seq） |
| 容量 | 任意（`% capacity`） | 2 的幂（位掩码） |
| 防伪共享 | Padded_indexing 槽间 padding | 4 个 Pad 隔开关键字段 |

---

## 索引与空满判定：怎么知道空还是满

### Integrals_lockfree_queue：虚拟索引三合一

head/tail 存的是**单调递增的 64 位虚拟索引**（永不回退），真实下标靠 `translate(from) = from % capacity` 换算。收益：

1. **空满判定无需牺牲槽位**：空 = `head == tail`，满 = `tail == head + capacity`——传统环形队列要留一个空位（head==tail 无法区分空满），虚拟索引版满装；
2. **ABA 防护**：虚拟索引单调（2^63-1 才回绕），CAS 的期望值天然携带"代数"，绕回导致的 ABA 实际不可达；
3. **并发推进**：抢占 head/tail 用 CAS + 占用位，失败方 yield 重试。

### mpmc_bq：槽 seq 与 ticket 差值

**状态编码在槽 seq 里**（无显式状态常量）：构造时 `m_ring[i].m_pos = i`；入队判 `diff = seq - pos == 0` 空、`< 0` 满；出队判 `diff = seq - (pos+1) == 0` 有元素、`< 0` 空。出队归还时 `m_pos = pos + capacity + 1` 推进一圈，下圈生产者才看到 `diff == 0`。

注释给出序列即 ticket 的原理："m_enqueue_pos only wraps at MAX(m_enqueue_pos), instead we use the capacity to convert the sequence to an array index. This is why the ring buffer must be a size which is a power of 2. This also allows the sequence to double as a ticket/lock."

### 对照：索引与空满判定

| 维度 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 索引 | 虚拟索引（head/tail 单调递增） | ticket（enqueue/dequeue 单调递增） |
| 空判 | `head == tail` | `diff = seq - pos == 0` 且 CAS 抢占失败 |
| 满判 | `tail == head + capacity` | `diff < 0` |
| 满装？ | 是（无需留空位） | 是（Vyukov 算法本身可满装） |
| ABA | 虚拟索引单调天然防 | seq 单调 + 64 位回绕不可达 |

---

## 互斥与交接：谁写哪个槽

### Integrals_lockfree_queue：MSB 占用位（把锁塞进指针）

head/tail 是 `Aligned_atomic<unsigned long long>`，**最高位是"此指针正在被某线程改写"的占用标志**：

```cpp
set_bit = 1ULL << 63;   /* 占用位 */
clear_bit = set_bit - 1;
```

CAS 成功者把新值**先置上占用位**存回（其他线程的 CAS 因期望值不匹配失败并 yield），写完槽位值后再存回清位值。这就是无锁互斥的手段——**与 `BgcTicket` 的"63 位值 + 1 位 MSB 锁"同构**（ticket 也这么干，见 binlog.md）。

### mpmc_bq：ticket CAS 发号（不需要占用位）

生产者抢 `m_enqueue_pos` 的 CAS 就是"发号"——**ticket 只负责发号，生产者抢到 ticket 后不必等前序生产者写完**（流水化），消费者只见 release 后的数据。槽的归属完全由"seq == pos"判定，不需要在指针上叠占用位。

### 对照：互斥手段

| 维度 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 指针互斥 | MSB 占用位（值+锁合一） | CAS 发号即互斥 |
| 槽交接 | 哨兵值 CAS（Null↔数据） | 槽 seq 状态判定（无值交换） |
| 握手 | 槽值承担"数据就绪"信号 | release store seq 发布 |

---

## push/pop 逐段剖析

### Integrals_lockfree_queue：抢 tail → 槽交接 → 清占用位

主链路（以 push 为例）：

```
push(to_push)
  ├─ assert(to_push != Null && != Erased)      ← 哨兵值不可入队
  ├─ clear_state()                             ← 清线程本地状态
  └─ 循环：
       ├─ 读 tail/head（清占用位）
       ├─ tail == head + capacity → 置 NO_SPACE_AVAILABLE，返回
       ├─ new_tail = tail+1 | set_bit          ← 置占用位
       ├─ CAS(m_tail, tail, new_tail) 失败 → yield 重试
       ├─ 槽 CAS(Null → to_push) 失败 → yield 重试（等并发 pop 清成 Null）
       ├─ new_tail &= clear_bit；store(m_tail) ← 清占用位 = 宣告完成
       └─ 返回 *this（链式；结果查 get_state()）
```

push 完整代码：

```cpp
push(value_type to_push) {
  assert(to_push != Null && to_push != Erased);
  this->clear_state();
  for (; true;) {
    auto tail = this->m_tail->load(std::memory_order_acquire) & clear_bit;
    auto head = this->m_head->load(std::memory_order_relaxed) & clear_bit;
    if (tail == head + this->m_capacity) {
      this->state() = enum_queue_state::NO_SPACE_AVAILABLE;
      return (*this);
    }
    auto new_tail = tail + 1;
    new_tail |= set_bit;                       /* Set the occupied bit */
    if (this->m_tail->compare_exchange_strong(tail, new_tail, std::memory_order_release)) {
      auto &current = this->m_array[this->translate(tail)];
      for (; true;) {
        T null_ref{Null};
        if (current.compare_exchange_strong(null_ref, to_push,
                                            std::memory_order_acquire)) {
          new_tail &= clear_bit;               /* Unset the occupied bit */
          this->m_tail->store(new_tail, std::memory_order_seq_cst);
          break;
        }
        std::this_thread::yield();             /* 并发 pop 尚未把槽清成 Null */
      }
      break;
    }
    std::this_thread::yield();                 /* 并发 push 抢先或占用位未清 */
  }
  return (*this);
}
```

pop 完整代码：

```cpp
T pop() {
  this->clear_state();
  for (; true;) {
    auto head = this->m_head->load(std::memory_order_acquire) & clear_bit;
    auto tail = this->m_tail->load(std::memory_order_relaxed) & clear_bit;
    if (head == tail) {
      this->state() = enum_queue_state::NO_MORE_ELEMENTS;
      break;
    }
    auto new_head = head + 1;
    new_head |= set_bit;
    if (this->m_head->compare_exchange_strong(head, new_head, std::memory_order_release)) {
      auto &current = this->m_array[this->translate(head)];
      for (; true;) {
        value_type value = current.load();
        if (value != Null &&
            current.compare_exchange_strong(value, Null, std::memory_order_release)) {
          new_head &= clear_bit;
          this->m_head->store(new_head, std::memory_order_seq_cst);
          if (value == Erased) {
            break;                             /* 被擦除的元素：继续 pop 下一个 */
          }
          return value;
        }
        std::this_thread::yield();             /* 并发 push 尚未写入槽值 */
      }
    }
    std::this_thread::yield();
  }
  return Null;
}
```

两个交接细节：① push 等槽变成 `Null`（等并发 pop 清完）才写；② pop 等槽变成非 `Null`（等并发 push 写完）才读。**槽值本身承担了"数据就绪"的握手**，head/tail 的占用位只保护指针的推进。

### mpmc_bq：ticket 发号 → 写槽 → release 发布

```cpp
  [[nodiscard]] bool enqueue(T const &data) {
    size_t pos = m_enqueue_pos.load(std::memory_order_relaxed);
    Cell *cell;
    for (;;) {
      cell = &m_ring[pos & m_capacity];
      size_t seq = cell->m_pos.load(std::memory_order_acquire);
      intptr_t diff = (intptr_t)seq - (intptr_t)pos;
      if (diff == 0) {                     /* 槽空：本圈轮到该 ticket */
        if (m_enqueue_pos.compare_exchange_weak(pos, pos + 1,
                                                std::memory_order_relaxed))
          break;                           /* 抢到 ticket 即独占该槽 */
      } else if (diff < 0) {
        return false;                      /* 满：前圈元素未消费 */
      } else {
        pos = m_enqueue_pos.load(std::memory_order_relaxed);   /* 别人推进了，重试 */
      }
    }
    cell->m_data = data;
    cell->m_pos.store(pos + 1, std::memory_order_release);     /* 发布给消费者 */
    return true;
  }

  [[nodiscard]] bool dequeue(T &data) {
    size_t pos = m_dequeue_pos.load(std::memory_order_relaxed);
    Cell *cell;
    for (;;) {
      cell = &m_ring[pos & m_capacity];
      size_t seq = cell->m_pos.load(std::memory_order_acquire);
      auto diff = (intptr_t)seq - (intptr_t)(pos + 1);
      if (diff == 0) {                     /* 有元素：生产者已发布 */
        if (m_dequeue_pos.compare_exchange_weak(pos, pos + 1,
                                                std::memory_order_relaxed))
          break;
      } else if (diff < 0) {
        return false;                      /* 空 */
      } else {
        /* Under normal circumstances this branch should never be taken. */
        pos = m_dequeue_pos.load(std::memory_order_relaxed);
      }
    }
    data = cell->m_data;
    cell->m_pos.store(pos + m_capacity + 1, std::memory_order_release);  /* 归还槽位 */
    return true;
  }
```

要点：① **ticket 只负责发号**，生产者抢到 ticket 后不必等前序生产者写完（流水化），消费者只见 release 后的数据；② **满装无空位浪费**——`capacity()` 返回 `n_elems`（Vyukov 算法本身可满装，不像传统留一空位）；③ 出队归还时 `m_pos = pos + capacity + 1` 推进一圈，下圈生产者才看到 `diff == 0`；④ dequeue 的 `diff > 0` 分支注释 "should never be taken"。

### 对照：push/pop

| 维度 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 抢占 | MSB 占用位 CAS（指针级） | ticket CAS（发号级） |
| 写槽 | 哨兵值 CAS 交接（等 Null / 等非 Null） | 直接赋值 + release store seq |
| 关键差异 | push/pop 在**指针和槽两个位置**各有一重 CAS | 只有一个 CAS（ticket），槽归属由 seq 判定 |

---

## 状态与语义：无锁的代价

### Integrals_lockfree_queue：结果基于线程本地视角

文件头注释直接承认：

> Being a lock-free structure, the queue may be changing at the same time as operations access both pointers and values... The operation results and returning states are always based on the thread local view of the queue state... **Therefore, extra validations, client-side serialization and/or retry mechanisms may be needed**.

即：操作线程安全（无内存问题），但**返回结果可能已过时**——push 成功不代表元素此刻还在（可能被并发 pop），pop 返回 Null 不代表此刻为空。这是与带锁队列最本质的语义差异，调用方必须接受或自行串行化。

状态传递用 **thread_local** 的 `enum_queue_state`（每线程每队列一份，源码 TODO 注明"queues 用得多了要垃圾回收这份 map"）——返回值保持纯数据（pop 返回 T、push 返回引用链式），错误经状态传递。

### mpmc_bq：bool 返回值

语义更简单直接：bool 返回入队/出队是否发生。代价是"过时"问题由返回值语义自行承担（enqueue 返回 true 不代表元素还在——但对象池场景下这正是期望语义）。

### 对照：状态语义

| 维度 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 返回 | 纯数据（链式 + T） | bool |
| 状态 | thread_local enum_queue_state | 无 |
| 过时语义 | 显式（get_state 自查） | 隐式（调用方默认接受） |

---

## 高级 API 与遍历

### Integrals_lockfree_queue：全套 API

- `get_state()` / `clear_state()`：thread_local 状态表。
- `erase_if(predicate)`：把命中谓词的槽 CAS 成 `Erased`（软删除，不移动 head/tail）；pop 遇到 `Erased` 自动跳过继续。**Erased 元素不影响空满判定**——`is_empty()` 只看 head==tail，即使队头是 `Erased` 也返回 false（`pop()` 会返回 Null 并置 `NO_MORE_ELEMENTS`，`front()` 把 Erased 映射成 Null）。
- **Iterator**：forward iterator 持 `m_current`（虚拟索引）+ `m_parent`；`begin()`/`end()` 是取值瞬间的 head/tail **快照**。头注释列了 4 种无锁迭代异常场景——迭代永不结束（总有 push 先于 end 发生）、起点是 Null（begin 后被并发 pop）、中间遇 Null/Erased、两轮迭代间 pop+push 超 capacity 导致元素与虚拟索引错位。注释结论："If one of the above scenarios is harmful to your use-case, an additional serialization mechanism may be needed"。

### mpmc_bq：只有三板斧

只有 `enqueue` / `dequeue` / `empty`——对象池场景（借出/归还）不需要遍历和软删除。

### 对照：API

| 维度 | Integrals_lockfree_queue | mpmc_bq |
|---|---|---|
| 遍历 | 弱一致迭代器（快照式） | 无 |
| 删除 | erase_if 软删除（Erased 第三态） | 无 |
| 查看 | front/back | 无 |

---

## 用户全景

### Integrals_lockfree_queue（唯一用户）

**`Bgc_ticket_manager::m_sessions_per_ticket`**（`sql/binlog/group_commit/bgc_ticket_manager.h`）——容量 1024，记录每个"已关闭分配"的 binlog 组提交 ticket 对应的会话总数：

```cpp
using queue_value_type = std::uint64_t;
using queue_type = container::Integrals_lockfree_queue<queue_value_type>;
static constexpr size_t max_concurrent_tickets = 1024;
queue_type m_sessions_per_ticket{max_concurrent_tickets};
```

### mpmc_bq（全仓仅 2 个实例化点）

1. **dblwr 刷写段池**（buf0dblwr.cc）——传 `Segment*`/`Batch_segment*` 对象池：

```cpp
using Segments = mpmc_bq<Segment *>;
using Batch_segments = mpmc_bq<Batch_segment *>;
```

刷写线程 `dequeue` 借 segment → 写盘 → `enqueue` 归还，**无锁对象池**。容量 `max(2, ut_2_power_up(SYNC_PAGE_FLUSH_SLOTS))`。

2. **FTS 并行建索引**（ddl0fts.cc）——传待解析文档：

```cpp
/** Must be a power of 2. */
static constexpr size_t DOC_ITEM_QUEUE_SIZE = 64;
using Docq = mpmc_bq<FTS::Doc_item *>;
```

父线程投递 `Doc_item*`，N 个解析子线程取文档做分词/词频统计——**1 生产 N 消费**的并行文档解析。

（澄清：`row0pread.h` 只是 `#include "ut0mpmcbq.h"`，**并未实例化**。）

---

## 可观测性

两个队列**都没有内置统计或 PFS 埋点**——观测只能经由用户：

| 我想看 | 手段 |
|---|---|
| Bgc_ticket_manager 的队列行为 | 无直接观测；binlog 组提交的 ticket 计数对齐协议见 [`../../server/replication/binlog.md`](../../server/replication/binlog.md) |
| dblwr 段池的借还 | 无直接观测；doublewrite 写入统计见 [`../../innodb/dblwr.md`](../../innodb/dblwr.md) |
| FTS 并行建索引队列 | 无直接观测 |

---

## 选择指南

- **装指针/任意对象、引擎热路径、只要入队出队** → `mpmc_bq`（容量 2 的幂）
- **装整型、需要迭代/软删除/前后查看、容量任意** → `Integrals_lockfree_queue`（注意结果基于线程本地视角）
- 需要强一致遍历或精确计数语义 → 回退带锁队列

**同族异路的根因**：两个队列是同一年代的同族产物（Vyukov 式 ticket 环），但服务对象不同：`mpmc_bq` 给 InnoDB 热路径当"指针对象池"（段池、文档队列），只需要 enqueue/dequeue 两板斧；`Integrals_lockfree_queue` 给 SQL 层机制当"整型计数管道"（ticket 会话数），需要迭代、软删除、前后查看等全套 API。**需求半径决定 API 半径**，与三个 hash 的"通用性越强、理论原型越重"同理。

与其他结构的关系：两个队列是**有界 MPMC（数据通道）**；`Link_buf` 是无锁完成跟踪（见 [`link_buf.md`](link_buf.md)）；三个 hash 是键空间字典（见 [`hash.md`](hash.md)）；分片计数器见 [`counter.md`](counter.md)。

---

## ★ 本机制里的工程实现技法

### 一、虚拟索引的"三合一"（Integrals）

| 收益 | 机制 |
|---|---|
| 满装无空位 | `tail == head + capacity` 判满，无需哨兵槽 |
| ABA 防护 | 单调索引携带代数，2^63 回绕不可达 |
| 并发推进 | CAS 期望值含代数，失败方 yield 重试 |

### 二、MSB 占用位：把锁塞进指针（Integrals）

与 `BgcTicket` 的 63 位值 + 1 位锁同构——**一个原子变量同时承载"值"与"临界区状态"**，省掉独立 mutex。代价：所有读方必须先 `& clear_bit` 抹掉占用位，且占用位未清时其他 CAS 必失败（自旋）。

### 三、哨兵值 CAS 握手：槽值即"数据就绪"信号（Integrals）

push/pop 的槽交接本质是**生产消费握手**：槽从 `Null`→`数据`（生产完成）→`Null`（消费完成），head/tail 只发号不确认。`Erased` 是第三态（软删除），pop 自动跳过。**三态槽值编码了元素的完整生命周期**。

### 四、线程本地状态代替返回码（Integrals）

无锁结构的"结果可能过时"语义下，bool 返回码会误导（"push 成功"不代表"元素还在"）；`thread_local` 状态表把"这次调用我看到了什么"留给调用方自己读（`get_state()`），返回值保持纯数据。Bgc_ticket_manager 的 `coalesce()` 正是用 `do { pop(); } while (get_state() != NO_MORE_ELEMENTS)` 循环清空队列。

### 五、序列即 ticket：一个数承担发号与状态（mpmc_bq）

槽的 `m_pos` 一个字段同时承担"发号依据"（diff 判空满）与"发布信号"（release store）——与 Integrals 的"指针占用位 + 槽哨兵值"两处状态不同，mpmc_bq 把状态压缩到一个 seq 里，代价是容量必须 2 的幂、元素槽不能有第三态。

### 六、cache line 对齐是共同的纪律

Integrals 用 `Padded_indexing`（槽间 padding）+ `Aligned_atomic`（head/tail 缓存行对齐）；mpmc_bq 用 4 个 `Pad` 隔开 ring/enqueue_pos/dequeue_pos——**高频无锁件的防伪共享是标准动作**。

---

## 面向二次开发

**Integrals_lockfree_queue**：新用户元素必须是整型（static_assert 强制）；哨兵值默认取 T 最大值，业务值若可能撞上须显式改模板参数；容量不要求 2 的幂但固定不可变。**语义纪律**：操作结果基于线程本地视角——push 成功 ≠ 元素还在；pop 返回 Null ≠ 队列为空。必要时外层串行化（Bgc_ticket_manager 就用 `AtomicBgcTicketGuard` 串行化前后 ticket 操作）。

**mpmc_bq**：容量 2 的幂、元素可拷贝、生命周期自管；析构**不清理元素**。

---

## 坑与已知缺陷汇总

### Integrals_lockfree_queue

1. **push/pop 自旋无上限**：抢占失败 yield 重试，高并发下最坏退化（无 backoff 策略）。
2. **迭代器弱一致**（头注释 4 场景）：需要强一致遍历的调用方自行串行化。
3. **thread_local 状态表的内存**（TODO 原文："garbage collect this if queues start to be used more dynamically"）——队列动态创建多时泄漏。
4. **Erased 元素的空满语义反直觉**：`is_empty()` 不评估槽值，队头是 Erased 时 `is_empty()==false` 但 `pop()` 返回 Null。

### mpmc_bq

1. **满装语义下空满判定依赖 64 位 seq 不回绕**（回绕需 2^64 次操作，实际不可达）；dequeue 的 `diff > 0` 分支注释 "should never be taken"。
2. **容量必须 2 的幂**（构造断言），否则位掩码寻址错位。
3. **析构不清理元素**：元素生命周期由使用者管理。

---

## 易混淆概念汇总

- **`Integrals_lockfree_queue`（server，整型）≠ `mpmc_bq`（InnoDB，任意类型）**：见[总览](#总览一个算法两种裁剪)对比表。
- **`Null`（空哨兵）≠ `Erased`（软删除）**：默认二者相同（`Erased = Null`），此时 `erase_if` 不生成（`enable_if_t<M != F>`）。
- **`get_state()` 的"状态" ≠ 队列的全局状态**：是**本线程上次操作**的结果，其他线程的并发修改不影响它。
- **虚拟索引 ≠ 数组下标**：head/tail 是虚拟索引，`translate()` 才映射到 `m_array` 下标。
- **mpmc_bq ≠ `Link_buf`**：一个装数据（队列），一个装完成跟踪（link）；名字都带"buffer"但语义完全不同。

---

## 参考

**论文 / 博客（源码注释自述）**
- D. Vyukov. *Bounded MPMC queue*（1024cores.net 博客，`mpmc_bq` 注释引用）
- `Integrals_lockfree_queue` 无源码论文引用——Vyukov 式单调 ticket 环形队列为并发工程常见手法，不要硬安论文。

**源码**
- `sql/containers/integrals_lockfree_queue.h`（全部实现，含无锁语义的头注释）
- `sql/binlog/group_commit/bgc_ticket_manager.h`（唯一实例化 `m_sessions_per_ticket`，容量 1024）
- `storage/innobase/include/ut0mpmcbq.h`（`mpmc_bq` 全部实现）
- `storage/innobase/buf/buf0dblwr.cc` / `ddl/ddl0fts.cc`（mpmc_bq 两个实例化点）

**相关文档**
- `Integrals_lockfree_queue` 唯一用户 `Bgc_ticket_manager`（发号/叫号/计数对齐协议/GR 视图变更屏障）见 [`../../server/replication/binlog.md`](../../server/replication/binlog.md) 的「BGC Ticket 系统」章
- 无锁三小件中与 mpmc_bq 同篇的 `Seq_lock`、`ib_counter_t` 已拆分：seqlock 见 [`../lock/primitives/seq_lock.md`](../lock/primitives/seq_lock.md)，计数器见 [`counter.md`](counter.md)
- 数据结构归属与全量清单见 [`../README.md`](../README.md)
