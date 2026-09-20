# Link_buf 无锁环形链缓冲深度解析

> 基于 MySQL 8.0.39 源码，剖析 InnoDB 的 `Link_buf<Position>`（`include/ut0link_buf.h`，2017 Paweł Olchawa 创建，header-only）：**"每槽存一个 link"的环形链缓冲**——多生产者乱序 `add_link` 报告"我的区间完成了"，tail 沿 link 链推进得到"最长连续已完成前缀"。redo 用它解耦两大全局串行约束：`recent_written`（并发写 log buffer vs log writer 写出）与 `recent_closed`（并发 mtr 提交 vs dirty page 进 flush list 的顺序）。
>
> **边界**：本篇讲 Link_buf 这个**无锁容器本身**；redo 怎么用它（`buf_ready_for_write_lsn`、flush order lag 的下游语义）见 [`../../innodb/redo_log.md`](../../innodb/redo_log.md)；同为 InnoDB 无锁结构的 `ut_lock_free_hash_t` 见 [`hash.md`](hash.md)，对比见本文「理论基础·他库对比」。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`Link_buf` 是"**完成跟踪（completion tracking）**"结构。文件头注释原文：

> Link buffer - concurrent data structure which allows:
> - concurrent addition of links
> - single-threaded tracking of connected path created by links
> - limited size of window with holes (missing links)

类注释把语义讲得更完整：

> This data structure is informed about finished concurrent operations and tracks **up to which point in a total order all operations have been finished (there are no holes)**. It also allows to limit the last period in which there might be holes. These holes refer to unfinished concurrent operations, which precede in the total order some operations that are already finished.

即：并发操作**局部乱序完成**，Link_buf 把它们整理成一个单调的"最长连续已完成前缀"。三个并发角色：报告完成（lock-free 多线程）、读最大连续位置（lock-free）、推进 tail（单线程跟踪，但实际上靠"槽级锁"实现了多线程安全推进）。

### 用途

redo 的两个实例（`log0sys.h`，均 `alignas(INNODB_CACHE_LINE_SIZE)` 防伪共享）：

| 实例 | 容量默认 | tail 语义 | 解耦什么 |
|---|---|---|---|
| `recent_written` | 1 MiB（`INNODB_LOG_RECENT_WRITTEN_SIZE_DEFAULT`） | `buf_ready_for_write_lsn`：已写入 log buffer 的连续前缀 | **并发写 log buffer** 与 **log writer 按序写出** |
| `recent_closed` | 2 MiB（`INNODB_LOG_RECENT_CLOSED_SIZE_DEFAULT`） | `dirty_pages_added_up_to_lsn`：mtr 已关闭的连续前缀 | **并发 mtr 提交** 与 **dirty page 按序进 flush list** |

`recent_closed` 的容量还兼作 `log_buffer_flush_order_lag()`——flush list 顺序松弛的**上界**（buf0flu 等用它修正 oldest_modification 的比较）。

### 版本演进

| 版本 | 变化 |
|---|---|
| 2017-08-30（8.0 早期） | Paweł Olchawa 创建 `ut0link_buf.h`——8.0 redo 全面重构（`log_sys` 新体系）的配套：旧版 redo 靠 mutex 串行写 buffer，新版放开并发写，需要"乱序完成→连续推进"的桥梁 |
| 8.0.39 | 沿用，仅微调内存序与注释 |

---

## 理论基础

### 设计思想与权衡

#### 一、为什么不能是普通环形队列

普通 ring buffer 要求生产者**按序**写槽（先写低 LSN 再写高 LSN），否则 reader 无法判断"写到哪了"。但 8.0 redo 的设计目标恰恰是**放开并发写**：mtr 提交时先 `reserve` 一段 LSN 区间，再慢慢把 redo 记录拷进 log buffer——拷贝完成顺序与 LSN 顺序天然不同。于是问题变成：

> **有全序的位置空间（LSN），并发操作乱序完成，如何算出"连续完成到哪了"？**

Link_buf 的答案：每个位置对应一个槽，槽里存"**从该位置起，到 to 为止全部完成**"这个 link；tail 沿 link 链前进。乱序完成只是"链上暂时有洞"，洞补上后 tail 一次跨过去。

#### 二、add_link(from, to)：区间而非逐点

为什么一条 link 覆盖一个**区间**而不是一个位置？因为一次并发操作天然对应连续区间（一次 mtr 写入 [start_lsn, end_lsn)），单次原子 store 记录终点即可覆盖整段——生产者开销 O(1)，且语义精确（"from 起、to 之前全部完成"）。逐点打点的代价是每字节一次原子写，不可能。

#### 三、槽不复位：靠"值 ≤ position 即无效"判定

遍历后的槽**不清理**，旧 link 残留在槽里。`next_position` 的判据是 `next <= position` 视为"无 link"——因为合法 link 必然 `next > position`。这带来两个后果：

1. **好处**：advance_tail 无需回收槽，省掉写者之间的清理竞争；
2. **代价**：任何"向后"读槽都可能看到陈旧值，`advance_tail_until` 必须用"CAS 槽锁 + 二次确认 tail 未动"来防御（源码注释专门讲了这个被调度挂起的场景）。

#### 四、tail 的"单写者幻觉"：槽级锁独占推进权

`m_tail` 用**普通 store** 而非 CAS——但写 tail 前必须先通过"槽 CAS 把 next 改成当前 position"抢到**槽级锁**，抢到者二次确认 tail 未动后才写。所以多线程可以并发调 `advance_tail_until`，谁 CAS 成功谁推进，写 tail 时必然无竞争。这是本结构最巧妙的一点：**把"tail 单写者"的约束下放成"槽锁互斥"，换来 API 的并发友好**。

#### 五、失效场景与退化阈值

1. **capacity 不足**：`has_space` 只检查"position 落在 [tail, tail+capacity) 窗口内"，不满足则调用方自旋等待（redo 是 sleep 20µs 循环 + 唤醒 writer）。窗口过小 → 用户线程频繁自旋（有 `MONITOR_LOG_ON_RECENT_WRITTEN/CLOSED_WAIT_LOOPS` 监控点）。
2. **link 跨度必须 < capacity**：`next_load >= position + m_capacity` 会被当作"缠绕/陈旧"处理，`ut_ad(to - from <= max)` 亦约束。
3. **stop_condition 必须是纯函数**：redo 的 `prev_lsn - write_lsn >= write_max_size` 满足"与 position 单调相关"，否则推进不一致。
4. **tail 可能"回退"**（读者视角）：checkpoint 代码注释 "The oldest_lsn can decrease ... which causes a maximum delay of log.recent_closed_size being suddenly subtracted"——并发读 tail 得到的是"某时刻的已确认值"。

### 理论溯源

- **无明星论文**（诚实说明）：完成跟踪（completion tracking / link buffer）是并发工程中的常见手法，`add_link + advance_tail` 的组合与"链路压缩（path compression）"思想相通（Union-Find 的路径压缩、work-stealing 的 completion 链均同族），但源码无任何论文引用。不要为对称硬安论文——与 `LF_HASH`（有 split-ordered list + hazard pointer 两篇论文原型）不同。

### 算法与数据结构

| 结构 | 组成 | 复杂度 |
|---|---|---|
| `Link_buf` | `atomic<Distance>[] m_links` + `alignas(cache line) atomic<Position> m_tail` | add_link O(1)；advance_tail 均摊 O(1)（每槽最多被跨越一次） |
| 槽编码 | 槽 [p] 存 link 终点；`slot_index = p & (capacity-1)` | — |
| 推进 | 槽 CAS 抢锁 → 二次确认 → relaxed 沿链遍历 → release store tail | 最坏 O(链长) |

### 他库对比与演进动机

| 容器 | 解决什么 | 与 Link_buf 的差异 |
|---|---|---|
| **Link_buf** | **全序位置上**的乱序完成跟踪（completion tracking） | 输出单调"连续完成前缀"；tail 沿有序 link 链推进 |
| `LF_HASH`（[`hash.md`](hash.md)） | **无序键值对**的并发增删查（并发字典） | split-ordered list + hazard pointer |
| `ut_lock_free_hash_t`（[`hash.md`](hash.md)） | 无序整数键值对的并发增删查 | 开放寻址 + 链表数组 + 引用计数 |

三者是三种完全不同的模式：**字典（两个 hash）vs 有序完成跟踪（Link_buf）**。Link_buf 的独特之处在于它利用"全序 + 区间 link"把乱序投影成顺序——这是两个 hash 都不具备的维度。

---

## 核心实现

### 主链路（一次 mtr 写入完成）

```
用户线程 mtr 提交（log_buffer_write_completed）
  ├─ has_space(start_lsn)？        ← 窗口检查：[tail, tail+capacity)
  │    否 → os_event_set(writer_event) + sleep 20µs 循环等
  ├─ atomic_thread_fence(release)  ← 保证 buffer 数据可见（注释：不依赖 add_link 内部序）
  └─ recent_written.add_link_advance_tail(start_lsn, end_lsn)
        ├─ 快路径：tail == start_lsn → 直接 store tail = end_lsn（无锁）
        └─ 慢路径：槽[start_lsn] 存 end_lsn → advance_tail_until
              ├─ 槽 CAS 抢"槽锁"（next → position）
              ├─ 二次确认 tail 未动
              ├─ relaxed 沿链遍历到断链（或 stop_condition）
              └─ release store 新 tail
```

### 数据结构

```cpp
template <typename Position = uint64_t>
class Link_buf {
 private:
  size_t m_capacity;                       /* 环的槽数（Position 单位） */
  std::atomic<Distance> *m_links;          /* 每槽存 link 终点 */
  alignas(ut::INNODB_CACHE_LINE_SIZE) std::atomic<Position> m_tail;  /* 防伪共享 */
};
```

`Distance` = `Position`（`typedef Position Distance;`，注释说"本可做成模板参数，目前没必要"）。构造断言 **capacity 必须是 2 的幂**（`ut_a((capacity & (capacity - 1)) == 0)`）——位掩码寻址免取模，且同余映射良定义；所有槽初始化 0（≤ 任何合法 position = "无 link"）。

### 核心操作逐段剖析

#### `add_link`：区间完成声明

```cpp
inline void Link_buf<Position>::add_link(Position from, Position to) {
  ut_ad(to > from);
  const auto index = slot_index(from);
  m_links[index].store(to);
}
```

#### `next_position`：沿链走一步（槽不复位的关键）

```cpp
inline bool Link_buf<Position>::next_position(Position position, Position &next) {
  next = m_links[slot_index(position)].load(std::memory_order_relaxed);
  return next <= position;   /* true = 无向前 link（旧残留值 ≤ position） */
}
```

#### `has_space`：写入前窗口检查

```cpp
inline bool Link_buf<Position>::has_space(Position position) {
  auto tail = m_tail.load(std::memory_order_acquire);
  if (tail + m_capacity > position) return true;
  auto stop_condition = [](Position, Position) { return false; };
  advance_tail_until(stop_condition, 0);   /* 不够则主动推进一次 tail（不重试）再判 */
  tail = m_tail.load(std::memory_order_acquire);
  return tail + m_capacity > position;
}
```

#### `add_link_advance_tail`：快路径无锁直写 tail

```cpp
inline void Link_buf<Position>::add_link_advance_tail(Position from, Position to) {
  auto position = m_tail.load(std::memory_order_acquire);
  ut_ad(position <= from);
  if (position == from) {
    /* can advance m_tail directly and exclusively, and it is unlock */
    m_tail.store(to, std::memory_order_release);      // 快路径：下一个连续区间
  } else {
    m_links[slot_index(from)].store(to, std::memory_order_release);  // 慢路径：存 link
    auto stop_condition = [&](Position prev_pos, Position) { return (prev_pos > from); };
    advance_tail_until(stop_condition);               // 推进最多不超过自己的区间
  }
}
```

快路径的注释 "it is unlock" 点出设计精髓：当 tail 恰好等于 from 时，本线程独占"下一个连续位置"，**连槽锁都不用抢**。

#### `advance_tail_until`：槽级锁 + 二次确认 + 沿链推进

```cpp
template <typename Stop_condition>
bool Link_buf<Position>::advance_tail_until(Stop_condition stop_condition, uint32_t max_retry) {
  /* multi threaded aware */
  auto position = m_tail.load(std::memory_order_acquire);
  auto from = position;
  uint32_t retry = 0;
  while (true) {
    auto &slot = m_links[slot_index(position)];
    auto next_load = slot.load(std::memory_order_acquire);
    if (next_load >= position + m_capacity) {
      /* 缠绕检测：要么绕回后读到陈旧值，要么 link 非法超长 */
      position = m_tail.load(std::memory_order_acquire);
      if (position != from) { from = position; continue; }
    }
    if (next_load <= position || stop_condition(position, next_load)) {
      return false;                                   /* 无 link 或停止条件命中 */
    }
    /* try to lock as storing the end —— 槽级锁：把 next 改成当前 position */
    if (slot.compare_exchange_strong(next_load, position, std::memory_order_acq_rel)) {
      /* 被调度挂起后，槽可能仍是向前 link 但 tail 已被他人推进（槽不重置）。
         必须二次确认 tail 仍等于 from 才能独家推进。 */
      position = m_tail.load(std::memory_order_acquire);
      if (position == from) {
        /* confirmed. can advance m_tail exclusively */
        position = next_load;
        break;
      }
    }
    retry++;
    if (retry > max_retry) return false;              /* give up */
    UT_RELAX_CPU();
    position = m_tail.load(std::memory_order_acquire);
    if (position == from) return false;               /* no progress? */
    from = position;
  }
  while (true) {                                      /* 第二阶段：无锁沿链遍历 */
    Position next;
    bool stop = next_position(position, next);
    if (stop || stop_condition(position, next)) break;
    position = next;
  }
  ut_a(from == m_tail.load(std::memory_order_acquire));
  m_tail.store(position, std::memory_order_release);  /* unlock */
  return position != from;
}
```

四段关键逻辑：① **缠绕检测**——`next_load >= position + capacity` 只可能是"绕回后读到旧时代的残留"或非法超长 link，此时重读 tail 跟进；② **槽级锁**——CAS 把槽内容从 next 改成当前 position，成功者独家获得推进权；③ **二次确认**——防"CAS 成功后自己被调度挂起、别人已推进 tail"的竞态；④ **两阶段推进**——赢锁后从 next_load 起 relaxed 沿链走（此时无人能改这些槽的语义），到断链为止，release store tail。

`advance_tail()` 就是 `advance_tail_until(恒 false 的 stop_condition)` 的退化版。

#### `validate_no_links`：debug 校验

```cpp
void Link_buf<Position>::validate_no_links(Position begin, Position end) {
  end = std::min(end, begin + m_capacity);   /* capacity 次迭代覆盖全部槽 */
  for (; begin < end; ++begin) {
    ut_a(m_links[slot_index(begin)].load() <= m_tail.load());  /* 残留旧值不算 link */
  }
}
```

### redo 的用法：两个实例解耦两条串行链

#### `recent_written`：并发写 buffer vs 顺序写出

- **生产者**（用户线程，`log_buffer_write_completed`）：先 `has_space(start_lsn)` 自旋（不足则 `os_event_set(log.writer_event)` + sleep 20µs），显式 release fence 后 `add_link_advance_tail(start_lsn, end_lsn)`——注释说明显式 fence 与 add_link 内部序冗余但刻意保留（"Not to rely on internals of Link_buf::add_link"）。
- **消费者**（log writer 线程，`log_advance_ready_for_write_lsn`，注释 "It's used by the log writer thread only"）：调 `advance_tail_until`，**stop_condition 限幅 4kB**：

```cpp
auto stop_condition = [&](lsn_t prev_lsn, lsn_t next_lsn) {
  return prev_lsn - write_lsn >= write_max_size;   /* 默认 4kB，防一次遍历过长 */
};
```

推进成功后 `validate_no_links(previous_lsn, ready_lsn)` 校验 + acquire fence。
- **普通读者**：`log_buffer_ready_for_write_lsn()` = `recent_written.tail()`——无锁读。
- **代推者**：`log_self_write_up_to`（用户线程自写日志路径）、后台 sync——都调 `advance_tail()` 主动整理。

#### `recent_closed`：破坏"dirty page 必须按 mtr 提交顺序进 flush list"的串行约束

- **生产者**：`log_buffer_s_lock_exit_close`（mtr 提交末尾）→ `add_link_advance_tail(start_lsn, end_lsn)`。
- **消费者/推进者**：`log_buffer_x_lock_enter`（X 锁必须等 closed 到 current_lsn，即所有洞补上）、checkpoint 的 `log_update_available_for_checkpoint_lsn`、后台 sync。
- **容量检查**：`log_wait_for_space_in_log_recent_closed`（dirty page 进 flush list **之前**调用，`mtr0mtr.cc`）——保证"dirty page 进 flush list 的延迟有界"（注释原文 "That's because we need to guarantee, that the delay until dirty page is added to flush list is limited"）。
- **下游修正**：`log_buffer_flush_order_lag()` = capacity——flush 列表比较 oldest_modification 时允许的乱序偏差上界（`buf0flu.cc`/`arch0page.cc`）。

#### 为什么需要两个（srv0srv.cc 注释原文）

```cpp
/* Number of slots in a small buffer, which is used to allow concurrent
writes to log buffer. ... */
ulong srv_log_recent_written_size;

/* Number of slots in a small buffer, which is used to break requirement
for total order of dirty pages, when they are added to flush lists. ... */
ulong srv_log_recent_closed_size;
```

一个解耦"**写**"的乱序，一个解耦"**关闭/进 flush list**"的乱序——两条独立的全序约束各自松弛，互不干扰。

### 生命周期

构造（capacity 2 的幂断言 + 槽清零）、移动构造/赋值（接管指针置空 rhs）、拷贝删除、析构 `free()`。redo 在 `log_start()` 初始化两条链：

```cpp
log.recent_written.add_link(0, start_lsn);
log.recent_written.advance_tail();   /* slot 0 被 0->start_lsn 占用，tail 跳到 start_lsn */
log.recent_closed.add_link(0, start_lsn);
log.recent_closed.advance_tail();
```

---

## ★ 本机制里的工程实现技法

### 一、乱序完成 → 连续推进的"投影"模式

| 技法 | 用在哪 | 为什么（收益） | 代价 |
|---|---|---|---|
| 区间 link（add_link from→to） | 生产侧 | 单次 store 覆盖整区间，O(1) 报告 | link 跨度必须 < capacity |
| tail 沿链推进 | 消费侧 | 洞补上后一次跨过，均摊 O(1) | 遍历时槽不清理，靠值判定 |
| stop_condition 模板参数 | 消费侧 | 同一结构服务多种推进策略（writer 限 4kB / 无条件全推） | 要求纯函数且与 position 单调相关 |

### 二、槽级锁把"tail 单写者"下放成并发推进

| 技法 | 用在哪 | 为什么（收益） | 代价 |
|---|---|---|---|
| 槽 CAS 抢锁（next→position） | `advance_tail_until` | 多线程并发推进安全化，API 无单写者限制 | CAS 失败重试 + 放弃路径 |
| 二次确认 tail 未动 | 同上 | 防"被调度挂起后他人已推进"的竞态（槽不重置的必然代价） | 多一次 acquire load |
| 快路径直写 tail（tail==from） | `add_link_advance_tail` | 最常见场景零 CAS | 依赖调用方保证 ut_ad(position <= from) |

### 三、与两个无锁 hash 的家族分工

三个容器合起来覆盖了 InnoDB 无锁结构的三种模式：**字典（`ut_lock_free_hash_t`）、通用字典（`LF_HASH`）、有序完成跟踪（Link_buf）**。共同技法是"**用原子操作 + 状态编码取代锁**"，但 Link_buf 独有的是"**利用全序性把乱序投影成顺序**"——这是"位置空间"型结构（ring/seq/epoch）的通用手法，与"键空间"型结构（hash）形成对照。

---

## 可观测性

| 我想看 | 手段 | 说明 |
|---|---|---|
| 两个 buffer 的等待/推进 | SQL | `innodb_log_recent_written_size` / `innodb_log_recent_closed_size`（READONLY sysvar，只读容量） |
| 窗口不足的自旋次数 | DBUG | `MONITOR_LOG_ON_RECENT_WRITTEN_WAIT_LOOPS` / `MONITOR_LOG_ON_RECENT_CLOSED_WAIT_LOOPS`（`srv0mon` 计数，`INNODB_METRICS` 可见） |
| ready_lsn / closed_lsn | 源码 | `log_buffer_ready_for_write_lsn()`、`log_buffer_dirty_pages_added_up_to_lsn()`——无 SQL 暴露，仅内部/调试 |
| 结构本体 | — | Link_buf 无 PFS 埋点、无 INNODB_METRICS 计数 |

排查要点：redo 写吞吐上不去、`log_writer` 频繁空转时，先看 `MONITOR_LOG_ON_RECENT_WRITTEN_WAIT_LOOPS` 是否飙升——是则说明 `recent_written` 容量（1 MiB 默认）对当前并发不够，或 writer 推进不及时。

---

## Misc

### 面向二次开发

- **新用户**：任何"并发乱序完成 + 需要连续前缀"的场景都适用（Position 只要支持 `+`/`-`/比较，默认 uint64）；容量选 2 的幂。
- **stop_condition 纪律**：必须纯函数、与 position 单调相关；不确定就传恒 false（`advance_tail()`）。
- **link 跨度 < capacity**：违反会触发缠绕检测的误判。

**坑与已知缺陷**（源码证据）：

1. **无 TODO/FIXME 于 ut0link_buf.h**；log0buf.cc 有 `@todo Consider refactoring to extract common logic of two recent buffers to a common class (Links_buffer ?)`。
2. **capacity 不足只靠调用方自旋**：sleep 20µs 循环，无内部等待机制（注释明说由调用方负责）。
3. **槽不复位的陈旧值问题**：CAS + 二次确认路径专为它而设（注释原文已引）。
4. **tail 可"回退"**（checkpoint 注释）：`The oldest_lsn can decrease ... which causes a maximum delay of log.recent_closed_size being suddenly subtracted`——读者须容忍。
5. **validate_no_links 只覆盖 capacity 个位置**（`end = min(end, begin + capacity)`）。

### 易混淆概念

- **Link_buf ≠ ring buffer**：ring buffer 是"槽存数据、按序读写"；Link_buf 是"槽存 link、乱序报告、tail 链式整理"。redo 的 log buffer 本身是 ring（数据），recent_written 是它的**完成跟踪器**（元数据）。
- **`log_closer` 线程不存在**（8.0.39）：`log.closer_event`/`closer_mutex` 只是"有人等 ready_lsn"的事件，等待方是 `log_self_write_up_to`，推进方是 log writer 线程。
- **两个实例各管一条串行链**：`recent_written` = 写的连续性；`recent_closed` = 关闭（进 flush list 前）的连续性。别混。
- **`log_sn_lock`（PFS rwlock key）与 Link_buf 无关**——它是 `log.sn` 的 debug 锁；`sn` 本身的并发协议是最高位自旋锁位（`SN_LOCKED`），也不是 seqlock（seqlock 见 [`../lock/primitives/seq_lock.md`](../lock/primitives/seq_lock.md)）。

---

## 参考

**论文 / 经典算法**
- 完成跟踪（completion tracking / link buffer）是并发工程常见手法，与 Union-Find 路径压缩、work-stealing completion 链同族——**源码无论文引用**，为从实现反推的对应关系，不要硬安论文。

**源码**
- `storage/innobase/include/ut0link_buf.h`（`Link_buf` 全部实现，含类注释的完整语义描述）
- `storage/innobase/log/log0buf.cc` / `log0log.cc` / `log0write.cc`（redo 两个实例的用法）
- `storage/innobase/srv/srv0srv.cc`（两个 sysvar 与注释原文）

**相关文档**
- redo 体系与 `buf_ready_for_write_lsn` 下游语义见 [`../../innodb/redo_log.md`](../../innodb/redo_log.md)
- 同族的无锁哈希：InnoDB 侧 [`hash.md`](hash.md)、server 侧 [`hash.md`](hash.md)
- 无锁结构归属与全量清单见 [`../../README.md`](../../README.md)
