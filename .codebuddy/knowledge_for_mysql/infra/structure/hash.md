# MySQL 哈希全盘点：一个问题的三种答案

> 基于 MySQL 8.0.39 源码，盘点 MySQL 的**全部自研哈希实现**，并按设计维度糅合对照：server 无锁 `LF_HASH`（split-ordered list + hazard pointer）、InnoDB 无锁 `ut_lock_free_hash_t`（开放寻址 + 链表数组 + 引用计数）、InnoDB 有锁经典 `hash_table_t`（链地址法 + 分片 rw_lock）。三者回答同一个问题——"高并发下怎么维护一张全局字典"——却走了三条路：**论文方案 / 教科书方案 / 经典教科书**。核心结论：**通用性越强，理论原型越重；写越少，锁越少**。
>
> **边界**：本篇讲哈希容器本身（跨层数据结构，不属锁原语——分类见 [`../lock/README.md`](../lock/README.md)）。用户视角在别处详写：lock_sys（`rec_hash`/`prdt_hash` 及其 `locksys::Latches` 分片锁保护）见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)，buffer pool `page_hash` 见 [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)，AHI 见 [`../../innodb/ahi.md`](../../innodb/ahi.md)，数据字典见 [`../../server/dd/dd.md`](../../server/dd/dd.md)；LF_HASH 的最大用户 MDL_map 见 [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md)；读者无锁的 RCU（`MyRcuLock`）见 [`../lock/primitives/rcu.md`](../lock/primitives/rcu.md)；`mt_fast_modulo_t` 依赖的 `Seq_lock` 见 [`../lock/primitives/seq_lock.md`](../lock/primitives/seq_lock.md)。

## 目录

- [总览：一个问题，三种答案](#总览一个问题三种答案)
- [结构组织：桶怎么摆](#结构组织桶怎么摆)
- [键与元素：装什么、怎么装](#键与元素装什么怎么装)
- [查找：怎么找](#查找怎么找)
- [插入与扩容：装满了怎么办](#插入与扩容装满了怎么办)
- [删除与内存回收](#删除与内存回收)
- [并发正确性协议](#并发正确性协议)
- [遍历能力](#遍历能力)
- [用户全景：谁用了谁、为什么](#用户全景谁用了谁为什么)
- [可观测性](#可观测性)
- [选择指南](#选择指南)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [面向二次开发](#面向二次开发)
- [坑与已知缺陷汇总](#坑与已知缺陷汇总)
- [易混淆概念汇总](#易混淆概念汇总)
- [参考](#参考)

---

## 总览：一个问题，三种答案

### 全量盘点：MySQL 8.0.39 的自研哈希

| 实现 | 层 | 并发性 | 结构 | 键值 | 用户 |
|---|---|---|---|---|---|
| `LF_HASH` | server（mysys） | **无锁**（lock-free） | split-ordered list（一条有序链 + dummy 分区间） | 任意对象（隐藏头 + 三回调） | MDL_map / Acl_cache / PFS / connection_delay |
| `ut_lock_free_hash_t` | InnoDB | **无锁**（lock-free，optimize 上锁） | 开放寻址 + 链表数组 | 仅整数对 `(uint64, int64)` | `buf_stat_per_index`（bp 每索引页数） |
| `hash_table_t` | InnoDB | **有锁**（调用方锁 / 内建分片 rw_lock） | 经典链地址法（侵入式宏） | 任意（调用方算 hash + 元素自带链指针） | lock_sys `rec_hash`/`prdt_hash`/`prdt_page_hash`、bp `page_hash`/`zip_hash`、dict `table_hash`/`table_id_hash`、AHI、`ha0storage` |

**为什么只有三个**：8.0 里 mysys 的通用 `HASH`（旧 `my_hash_*` 系列，5.x 时代的 server 层通用有锁哈希）**已被删除**——`include/lf.h` 的批注直言 `my_hash_get_key` 曾改名 "to force build error (since signature changed)"，意图是阻止任何人继续用旧风格。所以 **"server 层自研哈希"在 8.0.39 只剩 LF_HASH 一个**，其余散落各处的是专用结构（keycache 内部 `HASH_LINK`、`strings/` 的 `my_hash_sort_simple` 排序函数、temptable 的 `indexed_cells` 哈希索引），不属于通用容器，不纳入本篇。

### 三实现一张光谱

```
                  通用性 ─────────────────────────────────► 专用性
               任意对象                          整数对               任意对象
               无锁                              无锁                 有锁
              LF_HASH ────────────────────────► ut_lock_free_hash_t ─► hash_table_t
        论文方案（两篇明星论文）         教科书方案（Knuth+工程组合）     经典教科书（Knuth）
        写者从不阻塞读者                读者拿 RAII handle         调用方锁/分片锁
        扩容零迁移                    扩容显式迁移               无扩容（用户层处理）
```

### 三者回答同一个问题的分叉决策树

```
要一张并发字典
  ├─ 键是任意对象、server 层跨模块复用、需要元素级并发增删？
  │     → LF_HASH（无锁论文方案，代价：读者显式 pin、4 pin 上限）
  ├─ 键是整数对、高频 inc/dec、值归零即删？
  │     → ut_lock_free_hash_t（无锁教科书方案，代价：optimize 上锁、忙等读者）
  └─ 写多、要稳定遍历、已有外层锁协议、元素是复杂链表结构？
        → hash_table_t（有锁经典，代价：自己管锁、无扩容）
```

### 版本演进

| 版本 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 1995 | — | — | 随 InnoDB 诞生（Heikki Tuuri），链地址法 + 调用方锁，至今结构未变 |
| 5.0/5.1 | 引入（最初服务 Query Cache 等全局容器） | — | — |
| 5.x | — | — | 引入 `hash_create_sync_obj` 分片锁（SYNC_RW_LOCK 模式）；`ib_create` 工厂化 |
| 5.6+ | PFS 各对象表改用 LF_HASH（10 处 `lf_hash_init3`） | — | — |
| 2015（5.7） | — | Vasil Dimov 同时创建 `ut0lock_free_hash.h` 与 `buf0stats.h`——为"每索引 bp 页数"统计而生，**无锁 hash 与用户同源** | — |
| 8.0 | 最大用户变成 MDL_map（5.6 分区设计退役、LF_HASH 上位）；connection_control、ACL cache 接入 | 延续使用，仅同步 PFS/PSI/锁序注册 | `mt_fast_modulo_t`（Seq_lock 包裹）替换裸取模，适配并发 resize；page_hash 的 confirm 两段式协议定型 |

### 理论出身：为什么论文不对称

三个 hash 的**理论出身不对称**——这是理解一切差异的钥匙：

| | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 论文原型 | ★ **有**：split-ordered list（Shalev & Shavit, JACM 2006）+ hazard pointers（Michael, IEEE TPDS 2004）——两篇都是无锁数据结构领域的"明星论文" | ★ **没有**：开放寻址 + 线性探测是 Knuth TAOCP Vol.3 教科书内容；链表数组扩容 + 分片引用计数是经典技术的**工程组合** | **教科书**：链地址法是 Knuth TAOCP Vol.3 基础 |
| 诚实性 | ⚠️ 源码里无引用注释（grep `Shavit`/`Maged Michael` 无命中），从实现的 bit-reversed 排序、dummy 节点、pin 数组反推 | 源码无任何论文引用。**不要为对称硬安论文** | 无 |

**为什么 InnoDB 不用 LF_HASH 而自研 ut_lock_free_hash_t**：① LF_HASH 是 mysys 的通用容器（键值任意、带回调），对"只存整数对、高频 inc/dec"的 bp 统计是杀鸡用牛刀；② 引用计数的 `del_when_zero` 语义正是"页数归零即删条目"所需的；③ 整数 key 让开放寻址 + 位掩码哈希成为最优解（无碰撞链、无对象构造）。

**为什么写多场景仍用有锁的 hash_table_t**：无锁 hash 的 CAS 重试在写多时退化（竞争风暴），而 hash_table_t 持锁挂链是 O(1) 无重试；且 lock_sys / bp 的元素是复杂链表结构 + 已有外层锁协议。`lf.h` 批注原话："innodb hash_table_t 适合写多；LF_HASH 是一条链，用 dummy node 区分 cell/bucket"。

**同题不同解的根因**：LF_HASH 要服务"任意对象、跨模块通用"的 server 层容器需求，所以选择论文化的通用方案；`ut_lock_free_hash_t` 只服务一个"整数对、高频 inc/dec"的统计需求，所以选择教科书方案的最小实现；`hash_table_t` 服务"复杂对象 + 写多 + 已有锁协议"的引擎内部需求，所以选择连扩容都不做的经典原版。

### 阅读路径

本篇按**设计维度**组织，每个维度下三实现并排对照。只想读懂某一个实现：按目录找它的 `###` 小节连续读（每节的"是什么"部分自包含）；想理解"为什么这样设计"：从任一维度的"对照"小表出发，回看三个小节。

---

## 结构组织：桶怎么摆

三个哈希最根本的分野：**哈希值落到哪里、冲突怎么放**。

### LF_HASH：一条有序链（split-ordered list）

LF_HASH（**L**ock-**F**ree **Hash**）是 mysys 自研的**无锁可扩展哈希表**：`mysys/lf_hash.cc`（算法主体）+ `mysys/lf_alloc-pin.cc`（无锁分配器 + hazard pointer）+ `mysys/lf_dynarray.cc`（wait-free 动态数组），接口全在 `include/lf.h`。

```
LF_HASH（哈希表本体）
  ├── LF_DYNARRAY array     bucket 头数组（哨兵层，wait-free 可增长）
  ├── LF_ALLOCATOR alloc    无锁固定大小分配器（元素池，只增不减）
  │     └── LF_PINBOX       pin 管理器（版本化空闲栈）
  ├── LF_PINS               每线程一套 hazard pointer（4 pin + purgatory）
  └── LF_SLIST              链表节点（lf_hash.cc 私有，元素隐藏头）
```

核心洞察：**整个哈希只有一条全局有序单链表**，bucket 只是这条链上由 dummy（哨兵）节点分割出的区间。`calc_hash()` 把哈希值 `& INT_MAX32`（只用低 31 位，**最高位留给 odd/even 区分**），节点里存的是 **bit-reversed** 后的值：

```c
node->hashnr = my_reverse_bits(hashnr) | 1;   /* 普通节点：奇数 */
dummy->hashnr = my_reverse_bits(bucket) | 0;  /* 哨兵节点：偶数 */
```

整条链按 reversed hash 排序。于是**扩容 = `size *= 2`**：旧 bucket i 拆成 i 和 i+size，但 reversed 序下新区间天然是旧区间的二分裂——**链表顺序完全不变，不需要搬任何数据**（详见「插入与扩容」）。代价：哈希值浪费 1 位（31 位有效）+ 全局一条链，random_match 注释承认有"随机 dive 落到链尾"的尾部扫描风险。

算法与数据结构：

| 结构 | 组成 | 复杂度 |
|---|---|---|
| `LF_DYNARRAY` | 4 层 × 256 元素的递归数组（`LF_DYNARRAY_LEVELS=4`），永不 realloc（"so no pointer into the array may ever become invalid"） | 寻址 O(1)，扩展 CAS 逐层分配 |
| `LF_ALLOCATOR` | `top` 空闲单链栈（`atomic<uchar*>`）+ pinbox + ctor/dtor | 分配 O(1) CAS |
| `LF_SLIST` | `link`（低位=DELETED 标志）+ `hashnr`（reversed）+ `key`/`keylen`，数据紧随其后 | 遍历 O(链长) |
| 查找 | bit-reversed 有序单链，`my_reverse_bits` 用 256 项查表 | O(平均链长)，最坏 O(n) |

理论溯源：**Split-Ordered Lists: Lock-Free Extensible Hash Tables**（Shalev & Shavit, JACM 2006，bit-reversed 序、dummy 哨兵、无 rehash 扩容）与 **Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects**（Michael, IEEE TPDS 2004，pin 数组 + 延迟回收 + ABA 防护）。（⚠️ 两篇均非源码自述，见总览「理论出身」的诚实性注记。）

### ut_lock_free_hash_t：开放寻址 + 数组链表

与"一条有序链"完全相反：**开放寻址 + 线性探测**，元素是 `key_val_t` 数组里的槽位，没有链：

```c
struct key_val_t {
  key_val_t() : m_key(UNUSED), m_val(NOT_FOUND) {}
  std::atomic<uint64_t> m_key;
  std::atomic<int64_t> m_val;
};
```

`guess_position(key) = hash_uint64(key) & (size-1)`，从该位置**环形线性探测**到 `start + size`。数组装满了怎么办？**数组串联成单向链表**（`ut_lock_free_list_node_t`），扩容 = 追加一个新数组到链表尾（详见「插入与扩容」）：

```
ut_lock_free_hash_t
  ├── std::atomic<arr_node_t*> m_data        数组链表头
  ├── ib_mutex_t m_optimize_latch            唯一一把锁（仅 optimize 用）
  ├── hollow_t *m_hollow_objects             待销毁的空壳数组节点
  └── bool m_del_when_zero                   值归零自动删
        │
  arr_node_t = ut_lock_free_list_node_t<key_val_t>（数组节点）
        ├── unique_ptr<T[]> m_base           元素数组
        ├── atomic<next_t> m_next            下一个数组（CAS 追加）
        ├── atomic_bool m_pending_free       标记"等待读者退出"
        └── ut_lock_free_cnt_t m_n_ref       引用计数（256 分片）
```

**为什么 InnoDB 在这里选开放寻址**：键是整数（无碰撞链、无对象构造），位掩码哈希是 O(1) 期望；`del_when_zero` 的"归零即删"语义正命中"每索引页数"统计。全实现在头文件 `ut0lock_free_hash.h`（1044 行，无 .cc），2015 Vasil Dimov 创建。

### hash_table_t：经典链地址法

教科书原版：**桶数组 + 每桶一条侵入式链**。桶是 `hash_cell_t`，链节点就是元素自己（元素自带一个"链指针"成员）：

```cpp
struct hash_cell_t {
  void *node; /*!< hash chain node, NULL if none */
};

class hash_table_t {
 public:
  hash_table_t(size_t n) {
    const auto prime = ut::find_prime(n);
    cells = ut::make_unique<hash_cell_t[]>(prime);
    set_n_cells(prime);
    hash_table_clear(this);   /* memset 全部 cell 为 0 */
  }
  enum hash_table_sync_t type = HASH_TABLE_SYNC_NONE;  // 默认无内建同步！
  bool adaptive = false;      // 仅 debug，AHI 用
 private:
  std::atomic<size_t> n_cells;                 // 桶数，原子读
  ut::mt_fast_modulo_t n_cells_fast_modulo;    // 快速取模器（Seq_lock 包着）
 public:
  ut::unique_ptr<hash_cell_t[]> cells;         // 桶数组
  size_t n_sync_obj = 0;                       // rw_lock 个数（2 的幂），0 <=> SYNC_NONE
  rw_lock_t *rw_locks = nullptr;               // 分片锁数组
  mem_heap_t *heap = nullptr;                  // 仅 AHI 用（ha_node_t 堆）
};
```

两个不寻常的设计：

1. **素数桶**：`ut::find_prime(n)` 不是查表，是"**构造 + 试除**"——故意把桶数推到 2 的幂附近之外，再乘打散常数、逐个试除找素数：

```cpp
uint64_t find_prime(uint64_t n) {
  constexpr auto random1 = 1.0412321;
  constexpr auto random2 = 1.1131347;
  constexpr auto random3 = 1.0132677;
  n += 100;
  pow2 = 1;
  while (pow2 * 2 < n) pow2 = 2 * pow2;        // pow2 <= n 的最大 2 的幂
  if ((double)n < 1.05 * (double)pow2)          // 离下界 2 的幂太近就乘 random1
    n = (uint64_t)((double)n * random1);
  pow2 = 2 * pow2;
  if ((double)n > 0.95 * (double)pow2)          // 离上界 2 的幂太近就乘 random2
    n = (uint64_t)((double)n * random2);
  if (n > pow2 - 20) n += 30;
  n = (uint64_t)((double)n * random3);
  for (;; n++) {                                 // 朴素试除法找素数
    i = 2;
    while (i * i <= n) { if (n % i == 0) goto next_n; i++; }
    break;
  next_n:;
  }
  return n;
}
```

动机：避免"低位 hash 位直接决定桶"的偏斜；素数模 + 非 2 幂让桶分布更均匀。

2. **`n_cells_fast_modulo` 用 `Seq_lock` 包**：`{mod, inv}` 对要一起读、且 resize 线程写它时读者正并发读——`mt_fast_modulo_t` 的注释原文："It can be read without latches in parallel to set_n_cells, and as it is a complex object, it is not set atomically. Because of this the multi-threaded version is used."——这是 seqlock 在 InnoDB 的**唯一实例化点**（见 [`../lock/primitives/seq_lock.md`](../lock/primitives/seq_lock.md)）。

### 对照：结构组织

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 冲突解决 | 全局一条有序链 | 线性探测 | 每桶一条链 |
| 桶/槽形态 | bucket = 链上区间（dummy 分割） | (key,val) 槽数组 | hash_cell_t 桶头 |
| 哈希值处理 | & INT_MAX32 + bit-reversed | & (size-1) 位掩码（2 的幂） | % n_cells（素数，快速取模） |
| 节点在哪 | LF_SLIST 隐藏头 + 数据紧随 | 槽位内嵌 (key,val) | 元素自带链指针（侵入） |
| 缓存友好 | 差（链式，指针跳跃） | 好（连续槽） | 中（链式但短） |

---

## 键与元素：装什么、怎么装

"装什么"决定了整个设计的分叉——这是三实现差异的第一因。

### LF_HASH：任意对象 + 隐藏头 + 三回调

元素是**任意类型**。LF_HASH 在用户元素前塞一个隐藏头 `LF_SLIST`：

```c
struct LF_SLIST {
  std::atomic<LF_SLIST *> link; /* next 指针，最低位 = DELETED 标志 */
  uint32 hashnr;                /* reversed hash number, for sorting */
  const uchar *key;
  size_t keylen;
  /* data is stored here, directly after the keylen */
};
const int LF_HASH_OVERHEAD = sizeof(LF_SLIST);
```

`LF_HASH_OVERHEAD`（通常 32 字节）是每个元素的固定前缀。分配块布局 = `[0, OVERHEAD)` 隐藏头 + `[OVERHEAD, ...)` 用户对象。**非平凡对象**（如 MDL_lock 含 mutex/rwlock）靠三个回调解决生命周期：

```c
lf_hash_init2(&m_locks, sizeof(MDL_lock), LF_HASH_UNIQUE, 0, 0, mdl_locks_key,
              &my_charset_bin, &murmur3_adapter,
              &mdl_lock_cons, &mdl_lock_dtor, &mdl_lock_reinit);
```

| 回调 | 触发时机 | MDL 的实现 |
|---|---|---|
| **constructor** | 对象第一次 `my_malloc` 出来后（初始化昂贵部分） | `mdl_lock_cons`：`new (arg + LF_HASH_OVERHEAD) MDL_lock()` placement new |
| **destructor** | 对象真正 free 前（destroy 路径） | `mdl_lock_dtor`：显式析构 |
| **initialize/reinit** | insert 时替代 memcpy（dst=块+OVERHEAD，src=insert 参数） | `mdl_lock_reinit`：`dst->reinit(src)` |

`mdl_lock_reinit` 的注释给出复用必须重置的理由："Otherwise it is possible that we will end up in situation when 'new' (actually reused) MDL_lock object inserted in LF_HASH will inherit some values from old object."

本体结构 `LF_HASH`（分配器、回调、原子桶数全在）：

```c
struct LF_HASH {
  LF_DYNARRAY array;             /* hash itself */
  LF_ALLOCATOR alloc;            /* allocator for elements */
  hash_get_key_function get_key;
  CHARSET_INFO *charset;
  lf_hash_func *hash_function;
  lf_cmp_func *cmp_function;
  uint key_offset, key_length;
  uint element_size;             /* size of memcpy'ed area on insert */
  uint flags;                    /* LF_HASH_UNIQUE */
  std::atomic<int32> size;       /* 桶数，翻倍增长 */
  std::atomic<int32> count;      /* 元素数 */
  lf_hash_init_func *initialize; /* 非平凡对象用；NULL = memcpy 语义 */
};
```

`size`/`count` 是原子但允许普通读（32 位读不会撕裂，顶多陈旧——MDL 注释原文："Non-atomic read of LF_HASH::count which happens above should be OK as LF_HASH code does them too + preceding atomic operation provides memory barrier"）。

**任意对象的代价**：链式遍历、对象构造、缓存不友好——这正是"通用性越强，理论原型越重"的落点之一。

### ut_lock_free_hash_t：仅整数对 + del_when_zero

**`static_assert` 级别的约束**——只存 `(uint64_t key, int64_t val)` 整数对。没有对象、没有回调、没有键提取：槽位本身就是 (key,val) 两个原子。代价为零对象生命周期管理，换来开放寻址的缓存友好与无锁 CAS 的可行性。

**`del_when_zero` 把引用计数语义做进容器**：`inc`/`dec` 到 0 自动转 `DELETED`——这不是通用 hash 该有的，但正命中 `buf_stat_per_index` 的需求（页数归零 = 索引在 bp 无页 = 删条目）。为唯一用户定制一个开关，而不是做纯 KV 存储：

```c
bool update_tuple(key_val_t *t, int64_t val_to_set, bool is_delta) {
  int64_t cur_val = t->m_val.load(std::memory_order_relaxed);
  for (;;) {
    if (cur_val == GOTO_NEXT_ARRAY) return false;
    int64_t new_val;
    if (is_delta && cur_val != NOT_FOUND && cur_val != DELETED) {
      if (m_del_when_zero && cur_val + val_to_set == 0) new_val = DELETED;
      else new_val = cur_val + val_to_set;
    } else new_val = val_to_set;
    if (t->m_val.compare_exchange_strong(cur_val, new_val)) return true;
    /* CAS 失败时 cur_val 已被刷新为最新值，重试 */
  }
}
```

### hash_table_t：侵入链指针 + 宏

元素是**任意类型但必须自带一个"链指针"成员**（如 `buf_page_t::hash`、`lock_t::hash`）。没有隐藏头、没有回调——**宏在编译期知道链指针叫什么**：

```cpp
#define HASH_INSERT(TYPE, NAME, TABLE, HASH_VALUE, DATA)                 \
  do {                                                                   \
    hash_cell_t *cell3333; TYPE *struct3333;                             \
    const uint64_t hash_value3333 = HASH_VALUE;                          \
    hash_assert_can_modify(TABLE, hash_value3333);   /* debug: 持 X 锁 */ \
    (DATA)->NAME = NULL;                                                 \
    cell3333 = hash_get_nth_cell(TABLE, hash_calc_cell_id(hash_value3333, TABLE)); \
    if (cell3333->node == NULL) { cell3333->node = DATA; }               \
    else {                                                               \
      struct3333 = (TYPE *)cell3333->node;                               \
      while (struct3333->NAME != NULL) struct3333 = (TYPE *)struct3333->NAME; \
      struct3333->NAME = DATA;        /* 追加到链尾 */                    \
    }                                                                    \
  } while (0)
```

**注意是尾插不是头插**（循环走到链尾才挂）——有锁场景下尾插保证桶内顺序稳定（这对"同一 page 的锁队列按到达顺序排列"有意义）。宏方案的本质：TYPE 和 NAME 是编译期参数，免去函数指针回调，代价是类型安全全靠宏纪律（无模板）。

**hash 值完全由调用者提供**：hash0hash 里**没有 `hash_calc_hash` 函数**（8.0.39 grep 全仓 0 命中，勿引用），调用者自己算好（`page_id.hash()`、`ut::hash_string()`、`dtuple_hash()`）再交给 `hash_calc_cell_id`。

### 对照：键与元素

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 键值类型 | 任意（隐藏头 32B 前缀） | 仅 `(uint64,int64)` 对 | 任意（元素自带链指针） |
| 生命周期 | 三回调（ctor/dtor/reinit） | 无（POD 整数） | 无（元素由用户全权管理） |
| 键提取 | `hash_get_key_function` 回调 | key 就是 key | 调用者自己算 hash 值 |
| 通用性代价 | 链式遍历 + 对象构造 | 无（但只服务整数对） | 宏纪律（无类型安全） |

---

## 查找：怎么找

### LF_HASH：3 pin 滑动窗口 + helping

主链路（以一次 `lf_hash_search` 为例）：

```
lf_hash_search(hash, pins, key, keylen)
  ├─ hashnr = calc_hash(key) & INT_MAX32         ① 只用低 31 位
  ├─ bucket = hashnr % size                      ② 定位 bucket
  ├─ el = lf_dynarray_lvalue(array, bucket)      ③ 无锁取 bucket 头指针
  ├─ el->load() == nullptr → initialize_bucket   ④ dummy 惰性初始化（递归父 bucket）
  └─ my_lsearch(el, ..., reversed(hashnr)|1, ...) ⑤ 沿链查找
       └─ 找到 → lf_pin(pins, 2, curr)           ⑥ pin[2] 持有返回对象
            返回 found+1（跳过 LF_SLIST 头）
  调用方用完 → lf_hash_search_unpin(pins)         ⑦ 释放 pin[2]（唯一 unpin 点）
```

遍历主体 `my_lfind` 用 **3 个 pin 的滑动窗口**（pin[0]=next、pin[1]=curr、pin[2]=prev/结果）保护遍历途中的节点不被回收：

```c
retry:
  cursor->prev = head;
  do { cursor->curr = (LF_SLIST *)(*cursor->prev);
       lf_pin(pins, 1, cursor->curr);
  } while (*cursor->prev != cursor->curr && LF_BACKOFF);
  for (;;) {
    if (unlikely(!cursor->curr)) return 0;          /* end of the list */
    do { link = cursor->curr->link.load();
         cursor->next = PTR(link);                   /* 抹掉 DELETED 位 */
         lf_pin(pins, 0, cursor->next);
    } while (link != cursor->curr->link && LF_BACKOFF);
    if (*cursor->prev != cursor->curr) { goto retry; }  /* 前驱校验 */
    if (!DELETED(link)) {
      if (cur_hashnr >= hashnr) {
        if (cur_hashnr > hashnr) return 0;
        if (r >= 0) return !r;       /* cmp_function 或 my_strnncoll */
      }
      cursor->prev = &(cursor->curr->link);
      lf_pin(pins, 2, cursor->curr);
    } else {
      /* we found a deleted node - be nice, help the other thread
         and remove this deleted node */
      if (atomic_compare_exchange_strong(cursor->prev, &cursor->curr, cursor->next))
        lf_pinbox_free(pins, cursor->curr);
      else { (void)LF_BACKOFF; goto retry; }
    }
    cursor->curr = cursor->next;
    lf_pin(pins, 1, cursor->curr);
  }
```

注释明说 "pins[0..2] are used, they are NOT removed on return"。三个机制点：① **滑动窗口**——扫描自低向高，符合"只向上复制"协议（扫描方从低到高扫）；② **每步"读 link → pin → 重读校验"**——解 ABA；③ **helping**——路过带 DELETED 标志的节点时**顺手帮忙摘链**，保证"置 DELETED 标志次数 == 摘链次数"不变量。

`lf_hash_search` 返回三态：元素指针（**found+1，跳过 LF_SLIST 头**）/ `NULL`（未找到）/ `MY_LF_ERRPTR`（OOM 哨兵）。API 注释给出**持有窗口**的精确语义：

> Uses pins[0..2]. On return pins[0..1] are removed and pins[2] is used to pin object found. It is also not removed in case when object is not found/error occurs... So calling **lf_hash_unpin() is mandatory** after call to this function in case of both success and failure.

即：**search 返回后到 `lf_hash_search_unpin` 之间的整个使用期，pin[2] 一直 pin 住返回对象，期间对象绝不被释放**。MDL 中这个窗口覆盖 `m_rwlock` 加锁/解锁全程（mdl.cc 注释："We can't unpin object earlier as lf_hash_delete() might have been called for it already and so LF_ALLOCATOR is free to deallocate it once unpinned"）。

### ut_lock_free_hash_t：环形探测 + CAS 占槽 + 跨数组跟随

主链路（以 `inc(key)` 为例）：

```
inc(key)
  └─ insert_or_update(key, +1, is_delta=true, m_data)
       ├─ begin_access() 拿引用 handle（pending_free 则重试）
       ├─ insert_or_get_position_in_array：guess_position → 环形探测
       │     ├─ 命中 key → 返回 tuple
       │     ├─ 遇 UNUSED → CAS 占槽（冲突则看是否被抢先插成同 key）
       │     └─ 探测整圈仍无 → 返回 NULL（数组满）
       ├─ update_tuple(t, +1, true)：CAS 更新 val（GOTO_NEXT_ARRAY 则返回 false）
       ├─ 数组满或 GOTO → 沿 m_next 走下一数组
       ├─ 无下一数组 → grow() 追加（翻倍 / deleted>75% 不翻倍）
       └─ grow 由本线程完成 → optimize()（唯一上锁处，合并数组）
```

```c
for (size_t i = start; i < end; i++) {
  const size_t cur_pos = i & (arr_size - 1);   // 环形
  key_val_t *cur_tuple = &arr[cur_pos];
  const uint64_t cur_key = cur_tuple->m_key.load(std::memory_order_relaxed);
  if (cur_key == key) return cur_tuple;
  if (cur_key == UNUSED) {
    uint64_t expected = UNUSED;
    if (cur_tuple->m_key.compare_exchange_strong(expected, key)) return cur_tuple;
    if (expected == key) return cur_tuple;     // CAS 失败但被抢先插成同 key
    /* 否则是别人插了别的 key，继续探测下一槽 */
  }
  /* AVOID 槽直接跳过 */
}
return nullptr;   // 整圈满，无空槽
```

`get` 的跨数组跟随（注意 `GOTO_NEXT_ARRAY` 的防重排注释）：

```c
int64_t get(uint64_t key) const override {
  arr_node_t *arr = m_data.load();
  for (;;) {
    auto [handle, tuple] = get_tuple(key, &arr);
    if (tuple == nullptr) return NOT_FOUND;
    int64_t v = tuple->m_val.load(std::memory_order_relaxed);
    if (v == DELETED) return NOT_FOUND;
    else if (v != GOTO_NEXT_ARRAY) return v;
    arr = arr->m_next.load();   /* 防重排：val 是 GOTO 时 m_next 必已存在 */
  }
}
```

`get_tuple` 里 `begin_access()` 失败（数组 `pending_free`）则 `*arr = m_data.load()` **从链表头重试**——读者可能被"踢回"重来。

### hash_table_t：无锁 peek + 锁后 confirm 两段式

有锁哈希最体现工程技巧的地方。**SYNC_RW_LOCK 模式下，先无锁算锁、加锁，再确认锁没换**：

```cpp
static inline rw_lock_t *hash_lock_s_confirm(rw_lock_t *hash_lock,
                                             hash_table_t *table, uint64_t hash_value) {
  ut_ad(rw_lock_own(hash_lock, RW_LOCK_S));
  rw_lock_t *hash_lock_tmp = hash_get_lock(table, hash_value);
  while (hash_lock_tmp != hash_lock) {
    rw_lock_s_unlock(hash_lock);
    hash_lock = hash_lock_tmp;
    rw_lock_s_lock(hash_lock, UT_LOCATION_HERE);
    hash_lock_tmp = hash_get_lock(table, hash_value);
  }
  return hash_lock;
}
// hash_lock_x_confirm 完全同构，只是 S 换成 X。
```

运作机制（hash0hash.h 的 `n_cells` 字段注释是协议的依据）：

- **peek 阶段**：线程**无锁**读 `n_cells`（原子）算锁下标 `i = (hash % n_cells) % n_sync_obj`，对 `rw_locks[i]` 加 S/X 锁。此时桶数可能已被 resize 改掉，`i` 可能不是"当前"正确分片。
- **confirm 阶段**：持锁后**重新**按同一 hash_value 算锁下标（`hash_get_lock` 读新的 n_cells），若与手里这把不同——说明期间发生了 resize（cell_id、锁下标整体位移）——则解锁、换正确的锁、再算一次，直到"手里锁 == 按当前 n_cells 算出的锁"。
- **正确性保证**：resize 线程改 n_cells 前必须持有**全部分片 X 锁**，因此 confirm 循环拿到的锁，其保护的 cell 集合与 n_cells 一致。

**把"锁开销"从每次探测中省掉**：只有真正命中候选才上锁 confirm。典型调用（`Buf_fetch::lookup`，buf0buf.cc）：

```cpp
m_hash_lock = buf_page_hash_lock_get(m_buf_pool, m_page_id);
rw_lock_s_lock(m_hash_lock, UT_LOCATION_HERE);
/* If not own LRU_list_mutex, page_hash can be changed. */
m_hash_lock = buf_page_hash_lock_s_confirm(m_hash_lock, m_buf_pool, m_page_id);
```

查找本体用 `HASH_SEARCH` 宏（算 cell_id → 取链头 → 循环 `TEST` 匹配 → 不匹配 `HASH_GET_NEXT` 走链），debug 下 `hash_assert_can_search` 断言持对应分片锁。

**分片锁的创建与映射**（多对一，不是一桶一锁）：

```cpp
void hash_create_sync_obj(hash_table_t *table, latch_id_t id, size_t n_sync_obj) {
  ut_a(n_sync_obj > 0);
  ut_a(ut_is_2pow(n_sync_obj));
  table->type = HASH_TABLE_SYNC_RW_LOCK;
  table->rw_locks = static_cast<rw_lock_t *>(ut::malloc_withkey(..., n_sync_obj * sizeof(rw_lock_t)));
  for (size_t i = 0; i < n_sync_obj; i++)
    rw_lock_create(hash_table_locks_key, table->rw_locks + i, id);
  table->n_sync_obj = n_sync_obj;
}
// ib_create（ha0ha.cc）——工厂函数分两条路：
//   n_sync_obj == 0 → AHI 路径：只建 heap（MEM_HEAP_FOR_BTR_SEARCH），不建锁
//   n_sync_obj > 0  → 必为 MEM_HEAP_FOR_PAGE_HASH："We create a hash table
//                     protected by rw_locks for buf_pool->page_hash"
//   ⇒ SYNC_RW_LOCK 全仓库唯一用户是 page_hash

static inline uint64_t hash_get_sync_obj_index(hash_table_t *table, uint64_t hash_value) {
  return ut_2pow_remainder(hash_calc_cell_id(hash_value, table), table->n_sync_obj);
}
// = (hash % n_cells) % n_sync_obj —— 先算 cell_id 再对锁数取余
```

由于锁数远小于桶数，同一把锁覆盖多个 cell（步长 n_sync_obj）；且"落同一桶的节点必然被同一把锁保护"（hash0hash.h 顶部注释图：n_cells=1024、n_sync_obj=4，Shard 0 覆盖 cells 0,4,8,12…）。

### 对照：查找

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 路径 | 有序链 walk | 环形线性探测（可跨数组） | 桶链 walk（HASH_SEARCH） |
| 并发保护 | 3 pin 滑动窗口（无锁） | 引用 handle（无锁） | 分片 S 锁 + confirm |
| ABA 防护 | 读-pin-重读循环 | CAS 期望值携带状态 | 锁（无 ABA） |
| 最坏复杂度 | O(全局链长)，尾部扫描风险 | O(探测长度)，负载高退化 | O(桶链长)，桶数固定时键偏斜退化 |
| 未找到语义 | NULL / ERRPTR 两态 | NOT_FOUND 魔法值 | 链尾 NULL |

---

## 插入与扩容：装满了怎么办

扩容哲学是三个哈希**差异最大**的维度：零迁移 / 显式迁移 / 无扩容。

### LF_HASH：零迁移（dummy 惰性插入）

split-ordered list 的结构红利：**扩容只是把 `size` CAS 翻倍，数据一个不搬**。`lf_hash_insert` 尾部：

```c
node = (LF_SLIST *)lf_alloc_new(pins);
if (unlikely(!node)) return -1;
uchar *extra_data = (uchar *)(node + 1);  /* 数据紧随隐藏头 */
if (hash->initialize) (*hash->initialize)(extra_data, (const uchar *)data);
else memcpy(extra_data, data, hash->element_size);
node->key = hash_key(hash, (uchar *)(node + 1), &node->keylen);
hashnr = calc_hash(hash, node->key, node->keylen);
bucket = hashnr % hash->size;
el = lf_dynarray_lvalue(&hash->array, bucket);
if (el->load() == nullptr && unlikely(initialize_bucket(...))) { lf_pinbox_free(...); return -1; }
node->hashnr = my_reverse_bits(hashnr) | 1;   /* normal node */
if (linsert(el, cmp_function, charset, node, pins, hash->flags)) {
  lf_pinbox_free(pins, node);
  return 1;                                    /* 唯一键冲突 */
}
/* ★ resize 就在 insert 尾部 */
csize = hash->size;
if ((hash->count.fetch_add(1) + 1.0) / csize > MAX_LOAD) {  /* MAX_LOAD = 1.0 */
  atomic_compare_exchange_strong(&hash->size, &csize, csize * 2);
}
return 0;
```

linsert 的 CAS 循环（注释原文要点）：`my_lfind` 定位 → 唯一键冲突则返回 0 → 否则 `node->link = cursor.curr` → CAS 前驱指针插入。注释给出一个重要警告：**linsert 返回的冲突节点指针不可靠**——"cursor.curr is not pinned here and the pointer is unreliable, the object may disappear anytime. But if it points to a dummy node, the pointer is safe, because dummy nodes are never freed"。

**resize 完全惰性**：新 bucket 的 dummy 由后续访问触发 `initialize_bucket` 插入（含**递归初始化父 bucket**，因为搜索路径必须连续可走）：

```c
static int initialize_bucket(LF_HASH *hash, std::atomic<LF_SLIST *> *node,
                             uint bucket, LF_PINS *pins) {
  uint parent = my_clear_highest_bit(bucket);   /* 递归父 bucket */
  ...
  dummy->hashnr = my_reverse_bits(bucket) | 0;  /* dummy node */
  dummy->keylen = 0;
  if ((cur = linsert(el, cmp_function, charset, dummy, pins, LF_HASH_UNIQUE))) {
    my_free(dummy);
    dummy = cur;                                /* 复用并发者已插入的 dummy */
  }
  atomic_compare_exchange_strong(node, &tmp, dummy);
  /* CAS 失败（linsert 成功之后）无害 */
```

### ut_lock_free_hash_t：追加数组 + 显式迁移 + optimize 合并

开放寻址装满了必须搬数据，所以是**三段式**：

1. **追加**：数组满 → `grow()` CAS 把新数组（翻倍；`deleted > 75%` 则保持同大小）挂到链表尾。为什么 deleted 多不翻倍（注释）：删除过多时翻倍只会摊大稀疏数组，不如复用现有大小让 `DELETED` 槽被新插入覆盖。关键不变量（注释原文）：`m_next` "begins its life as NULL and is only changed once to some real value. Never changed to another value after that"——所以读 `m_next` 无 CAS 竞争，配合 `GOTO_NEXT_ARRAY` 标记即可安全后移。
2. **迁移**：`copy_to_another_array` 把旧数组元素逐个搬到新数组，旧槽 `m_val` CAS 成 `GOTO_NEXT_ARRAY` 标记"已搬走，去下一数组找"：

```c
void copy_to_another_array(arr_node_t *src, arr_node_t *dst) {
  for (size_t i = 0; i < src->m_n_base_elements; i++) {
    key_val_t *t = &src->m_base[i];
    uint64_t k = t->m_key.load(std::memory_order_relaxed);
    if (k == UNUSED && t->m_key.compare_exchange_strong(k, AVOID)) continue;  // 阻止新插入
    int64_t v = t->m_val.load(std::memory_order_relaxed);
    bool copied = false;
    for (;;) {
      if (v != DELETED || copied) { insert_or_update(k, v, false, dst, false); copied = true; }
      if (t->m_val.compare_exchange_strong(v, GOTO_NEXT_ARRAY)) break;   // 迁移完成
      /* CAS 失败：v 已刷新，重试（这次 insert_or_update 是 update 而非 insert） */
    }
  }
}
```

两个关键点（注释原文）：① `copied` 标志——"若已复制过，即使 val 变成 DELETED 也要继续同步最新值到 dst，否则并发的 delete 会丢失"；② 只能由 grow 成功的那个线程执行（其他线程的 insert/update 会命中旧数组的 tuple）。
3. **合并**：`optimize()` 把多数组收敛回单数组，恢复 O(1)。**这是唯一上锁处**：

```c
void optimize() {
  mutex_enter(&m_optimize_latch);
  for (;;) {
    arr_node_t *arr = m_data.load();
    arr_node_t *next = arr->m_next.load();
    if (next == nullptr) break;
    copy_to_another_array(arr, next);
    arr->m_pending_free.store(true);
    ut_a(m_data.compare_exchange_strong(expected, next));  // 摘头节点
    arr->await_release_of_old_references();                 // 忙等读者归零
    arr->m_base.reset();                                    // 释放元素数组
    m_hollow_objects->push_back(arr);                       // 空壳延迟销毁
  }
  mutex_exit(&m_optimize_latch);
}
```

`m_optimize_latch` 的注释是重要的设计取舍：迁移/摘除/释放"可以无锁实现，但复杂度大增，grow 不是热操作，选简单与可维护性"。

### hash_table_t：无扩容（用户层 resize 三姿势）

`hash_table_t` **没有内建扩容 API**。`set_n_cells` 全仓库只有 **2 个调用者**：构造函数 + `buf_pool_resize_hash`。装满了只能由**用户层**处理，8.0.39 有三种姿势：

1. **swap 偷换**（`buf_pool_resize_hash`，最精致）：建临时表 → 逐桶搬迁 → **不释放 page_hash 本体**，把新表的 `cells` 和 `n_cells` "偷换"给老表再删临时表：

```cpp
/* Concurrent threads may be accessing buf_pool->page_hash->n_cells,
n_sync_obj and try to latch rw_locks[i] while we are resizing. Therefore we
never deallocate page_hash, instead we overwrite its n_cells and cells with
the new values "stolen" from the temporary new_hash_table. */
std::swap(buf_pool->page_hash->cells, new_hash_table->cells);
/* ...两个 set_n_cells 交换... */
```

**这正是 confirm 协议存在的根因**——resize 期间读者正并发读 n_cells、锁 rw_locks。

2. **建新迁移**（`lock_sys_resize`）：`Global_exclusive_latch_guard` 下用 `HASH_MIGRATE` 宏整表搬迁到新表（注释："我们要在 cell 间重排 lock，并改变用于 shard 分片的 hash 函数参数，所以必须阻止所有人访问 lock sys 队列、甚至计算 shard id"）。
3. **整体重建**（AHI `btr_search_sys_resize`、`dict_resize`）：X 锁全部 part / mutex → `mem_heap_free` + delete 旧表 + `ib_create` 新表 → 全量回填。AHI 在 `btr_search_enabled` 时报错拒绝（"hash index hash table is not empty"）。

### 对照：插入与扩容

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 插入 | CAS 挂链（linsert 循环） | CAS 占槽 / 追加数组 | 宏尾插（持 X 锁） |
| 扩容触发 | load > 1.0（insert 尾部惰性） | 探测整圈无 UNUSED | 无内建，用户层判 |
| 扩容方式 | size CAS 翻倍 + dummy 惰性，**零迁移** | grow 追加 + copy 迁移 + optimize | swap 偷换 / 建新迁移 / 整体重建 |
| 扩容代价 | 几乎零（一个整数 CAS） | 显式迁移 O(size) | 停机式（全 X 锁 / 全局 guard） |
| 为什么 | 有序链的结构红利 | 开放寻址必须搬数据 | 写多场景，扩容低频可接受 |

---

## 删除与内存回收

"读者持指针期间，写者怎么安全释放内存"——这是无锁哈希最难的问题，两个无锁实现给了两种经典答案；有锁实现根本不用回答。

### LF_HASH：逻辑删除 + helping + purgatory（hazard pointer）

删除分两步（`lf_hash_delete`）：**① 逻辑删除**——link 最低位打 `DELETED` 标志（CAS）；**② 物理删除**——prev 指针跨过 curr（CAS），成功则 `lf_pinbox_free`：

```c
for (;;) {
  if (!my_lfind(head, cmp_func, cs, hashnr, key, keylen, &cursor, pins)) {
    res = 1; /* not found */
    break;
  } else {
    /* ① 逻辑删除：link 最低位打 DELETED 标志 */
    if (atomic_compare_exchange_strong(&cursor.curr->link, &cursor.next,
                                       SET_DELETED(cursor.next))) {
      /* ② 物理删除：prev 直接跨过 curr */
      if (atomic_compare_exchange_strong(cursor.prev, &cursor.curr, cursor.next)) {
        lf_pinbox_free(pins, cursor.curr);      /* 进 purgatory，不立即 free */
      } else {
        /* somebody already "helped" us and removed the node ?
           Let's check if we need to help that someone too! */
        my_lfind(head, ...);                    /* 帮别人摘下一个 DELETED */
      }
      res = 0;
      break;
    }
  }
}
```

两步 CAS 的价值：**读者永远看到自洽的链表**——打标志期间节点仍在链上（读路径跳过 DELETED 节点），摘链后读者看到的是新链。摘链失败说明别人已帮忙，则 re-find 检查是否还要帮"另一个 DELETED"摘链，维持"置标志次数 == 摘链次数"不变量。

**内存回收用 hazard pointer 的 purgatory 摊销**：`lf_pinbox_free` 并不真正 free，只是入**本线程 purgatory**（延迟释放栈）；每累计 `LF_PURGATORY_SIZE`(=10) 个才 `lf_pinbox_real_free` 扫一次**全部线程的全部 pin**——仍被 pin 的放回新 purgatory，其余批量 CAS 回分配器空闲栈。**写者从不阻塞读者**。

`lf_alloc-pin.cc` 头部注释是机制最完整的自述：

> It works as follows: every thread ... has a small array of pointers. They're called "pins". Before using an object its address must be stored in this array (pinned). When an object is no longer necessary its address must be removed from this array (unpinned). When a thread wants to free() an object it scans all pins of all threads to see if somebody has this object pinned. If yes - the object is not freed (but stored in a "purgatory"). To reduce the cost of a single free() pins are not scanned on every free() but only added to (thread-local) purgatory. On every LF_PURGATORY_SIZE free() purgatory is scanned and all unpinned objects are freed. **Pins are used to solve ABA problem.**

配套设计（`LF_PINS` / `LF_PINBOX` / `LF_ALLOCATOR`）：

```c
#define LF_PINBOX_PINS 4
#define LF_PURGATORY_SIZE 10

struct LF_PINS {
  std::atomic<void *> pin[LF_PINBOX_PINS];  /* 4 个 hazard pointer */
  LF_PINBOX *pinbox;
  void *purgatory;          /* 本线程延迟释放栈 */
  uint32 purgatory_count;
  std::atomic<uint32> link;
  /* we want sizeof(LF_PINS) to be 64 to avoid false sharing */
  char pad[...];            /* static_assert(sizeof(LF_PINS) == 64) */
};
```

三个关键设计：**4 个 pin 是最小充分集**（pin[0..2] 遍历窗口 + 结果持有，`LF_REQUIRE_PINS(3)` 编译期强制，pin[3] 留给 allocator）；**pin 协议只向上复制**（pin[N] 只能拷到 pin[M>N]，扫描方从低到高扫）；**版本化 pinbox 栈解 ABA**（`pinstack_top_ver` 高 16 位版本 + 低 16 位索引，空闲 pins 组成无锁栈；`LF_PINS` 整体 `static_assert(sizeof == 64)` 硬性防伪共享）。**语义是"每使用者一套"**——MDL 里每 `MDL_context` 一个 `LF_PINS *m_pins`（mdl.h 注释原文："Thread's pins (a.k.a. hazard pointers) ... NULL if pins are not yet allocated"）。

pin 协议（源码注释原文，读代码前必背）：

> 3. Pin the PTR in a loop: `do { LOCAL_PTR = PTR; pin(PTR, PIN_NUMBER); } while (LOCAL_PTR != PTR)`
> 4. It is guaranteed that after the loop has ended, LOCAL_PTR points to an object ... that will never be freed.
> 5. When done working with the object, remove the pin: unpin(PIN_NUMBER)
> 7. Don't keep the object pinned longer than necessary - the number of pins you have is limited (and small)

第 3 条是解 ABA 的核心：**先读地址、再 pin、再重读校验**——若 pin 时对象已被释放并复用，重读会发现地址变了，重试。

分配器 `LF_ALLOCATOR`：`top` 空闲单链栈 + `constructor`/`destructor`。`lf_alloc_new` 的核心：CAS 弹栈顶（用 pin[0] 防 ABA），栈空则 `my_malloc` + constructor；回收路径 `lf_pinbox_free` **只是入 purgatory**（用 `free_ptr_offset` 处的字段链入——对 LF_HASH 就是 `offsetof(LF_SLIST, key)`，对象已脱离链表，key 字段无人再读）。

### ut_lock_free_hash_t：DELETED 复用 + 引用计数 + hollow

删除是 `update_tuple` 里 `del_when_zero` 归零转 `DELETED`（或 `del` 显式置），**槽位不回收，insert 复用**。真正的"释放"发生在数组摘除时，用**引用计数 + RAII handle + hollow 延迟销毁**：

```c
class ut_lock_free_cnt_t {
  class handle_t {                     // RAII：构造 fetch_add(1)，析构 fetch_sub(1)
    explicit operator bool() const noexcept { return m_counter != nullptr; }
  };
  handle_t reference() { return handle_t{&m_cnt[n_cnt_index()]}; }
  void await_release_of_old_references() const {
    for (size_t i = 0; i < m_cnt.size(); i++)
      while (m_cnt[i].load()) std::this_thread::yield();   // 忙等归零
  }
  std::array<ut::Cacheline_aligned<std::atomic<uint64_t>>, 256> m_cnt;
};
```

- `begin_access()` 先 `m_n_ref.reference()` 拿 handle（RAII，析构自动 `fetch_sub(1)`），再检查 `m_pending_free`——为 true 则拒绝访问（返回空 handle），调用方从链表头重试；
- 数组被摘除时先置 `m_pending_free=true`（关闸阻止新读者），再 `await_release_of_old_references()` **自旋等引用归零**，然后释放 `m_base`；
- 摘下的空壳 `arr_node_t` 进 `m_hollow_objects`，**哈希表析构时才真正销毁**。

`ut_lock_free_cnt_t` 是"按 CPU 分片"的引用计数（256 个 cacheline 对齐原子计数器，`n_cnt_index()` 用 `os_getcpu()` 取片 `% 256`——注释："同一 CPU 总取同一计数器，落本地缓存，NUMA 下免去跨 CPU 缓存同步"）。

### hash_table_t：宏摘链（有锁，无回收问题）

`HASH_DELETE` 宏：链上线性找前驱 → 摘链 → debug 下 `HASH_INVALIDATE` 把摘下的节点链指针打成毒值：

```cpp
#ifdef UNIV_HASH_DEBUG
#define HASH_ASSERT_VALID(DATA) ut_a((void *)(DATA) != (void *)-1)
#define HASH_INVALIDATE(DATA, NAME) *(void **)(&DATA->NAME) = (void *)-1
#else
#define HASH_ASSERT_VALID(DATA) do {} while (0)
#define HASH_INVALIDATE(DATA, NAME) do {} while (0)
#endif
```

**元素内存由用户全权管理**——hash_table 从不分配/释放元素，这是"侵入式 + 有锁"的自然结果：锁保证了没有并发读者，释放顺序就完全由用户掌控（如 `lock_t` 的归还、`buf_page_t` 的复用）。`hash_table_clear` 只 `memset` 桶头（不清 heap），由构造函数与 AHI `btr_search_disable` / `ha_storage_empty` 调用。

### 对照：删除与内存回收

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 删除 | 逻辑位 + 物理摘链（两步 CAS） | CAS 置 DELETED（复用）/ GOTO_NEXT_ARRAY | 宏摘链（持 X 锁） |
| 读者保护 | hazard pointer（pin + purgatory） | 引用计数 handle（RAII） | 锁（无并发读者） |
| 写者代价 | 扫 pins（**从不阻塞读者**） | **忙等读者归零** | 持锁（阻塞读者） |
| 读者代价 | 显式 pin/unpin + 4 pin 上限 | 拿 handle（自动析构） | 零（锁外无指针） |
| 内存 | 池复用，不还 OS | 空壳延迟销毁 | 用户管理 |

---

## 并发正确性协议

### LF_HASH：CAS + pin 协议（完全无锁）

读/写/删**全无锁**，正确性靠三件：① 所有链修改都是 CAS（失败重试 `LF_BACKOFF`）；② 每步"读 → pin → 重读校验"解 ABA；③ helping 维持"置标志 == 摘链"不变量。注意 "lock-free 不是 wait-free"：CAS 失败会重试；只有 `LF_DYNARRAY` 的扩展自嘲 "wait-free, not lock-free ;-)"（文件头注释原文："Analog of DYNAMIC_ARRAY that never reallocs (so no pointer into the array may ever become invalid)"，`lf_dynarray_lvalue` 用 CAS 逐层分配缺失层，`lf_dynarray_iterate` 是 purgatory 扫描的引擎）。

### ut_lock_free_hash_t：CAS + 魔法值状态机

读/写/删无锁，但 **optimize 用 mutex**（唯一上锁处）。并发正确性靠**魔法值把生命周期编码进键值本身**（注释把完整转移表列全了，读任何 CAS 前先对这张表）：

| 特殊值 | 作用 |
|---|---|
| `UNUSED = UINT64_MAX`（key） | 空槽，初始状态 |
| `AVOID = UNUSED-1`（key） | 禁用槽：迁移时把 `UNUSED` CAS 成 `AVOID` 阻止新插入，搜索遇之即停 |
| `NOT_FOUND = INT64_MAX`（val） | 初始值 / 刚插入未写完 |
| `DELETED = NOT_FOUND-1`（val） | 已删除，insert 可复用 |
| `GOTO_NEXT_ARRAY = DELETED-1`（val） | 已迁移到下一数组，搜索沿 `m_next` 继续 |

key 合法转移：`UNUSED → 真实key`、`UNUSED → AVOID`（其余禁止）；val 合法转移：`NOT_FOUND → 真实值/DELETED`、`真实值 → 另一真实值/DELETED/GOTO_NEXT_ARRAY`、`DELETED → 真实值/GOTO_NEXT_ARRAY`。**代价**：真实 key 不能用 `UINT64_MAX`/`-1`，真实 val 不能用 `INT64_MAX` 附近三个值。

### hash_table_t：分片锁 + confirm 两段式（可选内建锁）

两种模式：

- **SYNC_NONE（默认）**：hash_table 只是数据，锁是**用户的既有锁**——最普遍的形态：lock_sys 三表用 `locksys::Latches` 分片锁（512 表分片 + 512 页分片 + 全局闸门，**分片与 hash cell 同键映射**保证"锁队列在哪个 cell 哪个 shard 就护着它"——详写见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)）；dict 两表用 `dict_sys->mutex`（查找路径 `ut_ad(dict_sys_mutex_own())`）；AHI 用 8 个 part latch（与 hash_table 并列放 cache line 上）；zip_hash 用 `zip_hash_mutex` 独占。
- **SYNC_RW_LOCK**：**唯一用户是 buf_pool 的 page_hash**（见「查找」），配合"无锁 peek + 锁后 confirm"。

**锁序约定**：`hash_lock_x_all` 按数组**升序**获取全部分片 X 锁（防死锁），用于 resize/清空；`hash_unlock_x_all_but` 保留一把释放其余；`hash_lock_has_all_x` 仅 debug，作为 `set_n_cells` 的前置断言。

### 对照：并发协议

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 读 | 无锁（pin 保护） | 无锁（handle） | S 锁（或无锁 peek） |
| 写 | 无锁（CAS） | 无锁（CAS，optimize 除外） | X 锁 |
| 完全无锁？ | 是 | 否（optimize mutex） | 否 |
| 写多场景 | CAS 重试退化 | 同左 | **适合**（挂链 O(1)，无重试风暴） |

---

## 遍历能力

| 维度 | LF_HASH | ut_lock_free_hash_t | hash_table_t |
|---|---|---|---|
| 遍历 | `lf_hash_random_match`：随机 dive + 回绕，`rand_val=0` + 全匹配回调退化成全表遍历 | **无遍历 API** | 有锁时稳定遍历（HASH_GET_FIRST/NEXT 全桶扫，`HASH_SEARCH_ALL`） |
| 说明 | ① 随机取样（MDL 近似 LRU 淘汰）② 全遍历（Acl_cache flush）——一个 API 覆盖两种需求；**`lf_hash_iterate` 不存在**（全仓 0 匹配） | 统计场景不需要 | 唯一支持"稳定迭代"的哈希 |

`lf_hash_random_match` 的去偏置设计（注释原文）：随机 dive 后若没找到且 `hashnr != 0`，**回绕到链表头再找到起点为止**——既不偏向 bucket 头也不偏向全局头；`my_lfind_match` 里 dummy 是安全重启点（"thanks to the fact that dummy nodes are never deleted we can save it as a safe place to restart iteration if ever needed"）。

---

## 用户全景：谁用了谁、为什么

### LF_HASH（server）

| 用户 | 文件 | 用法 |
|---|---|---|
| **MDL_map::m_locks** | sql/mdl.cc | 全服务器 MDL_lock 容器；search/insert/delete/random_match 全用，最复杂用户 |
| **Acl_cache::m_cache** | sql/auth/sql_auth_cache.cc | ACL 权限缓存（Acl_map）；`random_match(0, cache_flusher)` 循环清空 |
| **connection_delay** | plugin/connection_control/connection_delay.cc | 失败连接延迟黑名单；search/insert/delete + 遍历填 I_S 表 |
| **PFS（10 处）** | storage/perfschema/pfs_user/host/account/digest/... | 各对象注册表，lookup 为主 |

**澄清**（旧资料常见错误）：TC_LOG、group replication、clone、table_def_cache（TDC 有自己的 cache）、InnoDB 的 `dict`（用经典链地址法 `hash_table_t`）都**不**用 LF_HASH。**演进动机**：8.0 里 MDL_map 从"分区 + mutex"改成 LF_HASH（见 [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md)），是对"高并发只读查找"的终极答案——查找零互斥，插入/删除只在链表局部 CAS。

### ut_lock_free_hash_t（InnoDB）

唯一用户 `buf_stat_per_index_t`（`buf0stats.h`，全局单例）：key=index_id、val=页数，构造 `(1024, true)`。页面加入/移出 buffer pool 的**热路径**（`buf0buf.cc` 的 `buf_stat_per_index->inc(...)`）会高频调用 `inc`/`dec`，多 buffer pool 实例并发访问，用全局 mutex 会成为瓶颈，因此引入无锁 hash。三个要点：① `initial_size=1024`（2 的幂，`ut_a(ut_is_2pow)` 断言）；② `del_when_zero=true`；③ `should_skip` 跳过 insert buffer（`is_ibuf`）、临时表空间、index_id 高 32 位非 0 的索引。

### hash_table_t（InnoDB 的写多主力）

| 用户 | 表 | 键 | 同步 |
|---|---|---|---|
| lock_sys | `rec_hash` / `prdt_hash` / `prdt_page_hash` | `page_id.hash()` | SYNC_NONE + `locksys::Latches` 512 分片（同键映射，详写见锁文档） |
| buf_pool | `page_hash` | `page_id.hash()` | **唯一 SYNC_RW_LOCK**（`srv_n_page_hash_locks` 把分片锁，上限 `MAX_PAGE_HASH_LOCKS`） |
| buf_pool | `zip_hash` | frame 索引 | SYNC_NONE + `zip_hash_mutex`（存 `buf_block_t*`，供 buddy allocator） |
| dict_sys | `table_hash` / `table_id_hash` | 表名 hash / 表 id | SYNC_NONE + `dict_sys->mutex` |
| AHI | `btr0sea.h` 的 `hash_table`（`adaptive=true`） | `(space,index,prefix)` 折叠 | SYNC_NONE + 8 个 part latch（全 nowait 哲学，见 [`../../innodb/ahi.md`](../../innodb/ahi.md)） |
| `ha0storage` | 字符串去重存储 | 字符串 hash | SYNC_NONE（单线程启动期用） |

**AHI 的查找路径**（`btr_search_guess_on_hash`，全 nowait 哲学）：`btr_search_s_lock_nowait(index)` 拿 part S 锁 → `ha_search_and_get_data` 查桶（沿 ha_node_t 链按 hash_value 匹配）→ `buf_block_from_ahi(rec)` 得块 → `buf_page_get_known_nowait` 固定块 → 释放 S 锁 → 验证 space/index_id 与 `btr_search_check_guess`。注释明确：S 锁期间 AHI 项不会被移除，因此块一定还在缓冲池。**哈希桶和 part latch 一起放在 cache line 上**（`search_part_t` 的 `alignas(INNODB_CACHE_LINE_SIZE)`）。AHI 的 `heap` 字段存 ha_node_t（`hash_get_heap` 有 `ut_ad(table->n_sync_obj == 0)`——heap 只允许出现在无锁模式）。

**为什么 lock_sys 不用无锁 hash（架构约束，源码无直接注释，从约束反推）**：① lock 对象是**复杂链表结构**（`lock_t` 内嵌 hash 链指针 + 同一 page 的锁要合并成一条 queue + 与 512 shard 互斥映射一致）——远超 ut_lock_free_hash_t "整数 key→整数 value" 的表达能力；② lock_sys_resize 要"改变用于 shard 分片的 hash 函数参数"（停机式），无锁哈希的并发扩容在这里反而是麻烦；③ 行锁是**写多**场景（每把锁都要入队/出队），`lf.h` 批注自己就说了 hash_table_t 适合写多。

---

## 可观测性

三个哈希本体都**没有内置埋点**——观测只能经由用户：

| 我想看 | 手段 |
|---|---|
| LF_HASH 相关内存 | `performance_schema.memory_summary_*` 的 `key_memory_lf_node`（`lf_alloc_new` 的 `my_malloc` 用该 key） |
| LF_HASH 用户行为 | MDL 锁对象总数无直接计数器，行为推断：`performance_schema.metadata_locks` 行数 + PFS 的 `Performance_schema_metadata_lock_lost`；ACL cache flush 走 general log 观察（低频操作） |
| LF_HASH 容器本身 | DBUG `LF_BACKOFF` 相关调试点（仅 debug 构建） |
| ut_lock_free_hash_t 的查找/碰撞 | 编译期宏 `UT_HASH_IMPLEMENT_PRINT_STATS` 定义后 `print_stats()` 输出搜索次数/迭代次数/平均迭代——注释警告：原子统计"性能影响巨大"，非原子"偏差大但几乎无开销" |
| ut_lock_free_hash_t 内存 | `mem_key_ut_lock_free_hash_t` / `mem_key_buf_stat_per_index_t`（PFS memory_summary）；`lock_free_hash_mutex`（`m_optimize_latch` 的 PSI key） |
| hash_table_t | 无任何埋点。lock_sys 侧经「InnoDB Monitor 锁段」（`SHOW ENGINE INNODB STATUS`）与 PFS `data_locks` 表间接可见（见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)）；bp 侧经 `INFORMATION_SCHEMA.INNODB_BUFFER_PAGE` 按 `INDEX_ID` 聚合（间接） |

排查要点：LF_HASH 的问题几乎都表现为**用户行为异常**（MDL 锁对象不回收 = 淘汰条件未触发，见 [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md) 的 `remove_random_unused` 阈值 `unused > 1000 && unused > count * 0.25`）。

---

## 选择指南

- 要存**任意类型对象**、元素级并发增删、需要随机取样/全遍历、server 层跨模块复用 → **`LF_HASH`**（读者显式 pin，写者从不阻塞读者）
- 要存**整数键值对**、高频 inc/dec、值归零即删 → **`ut_lock_free_hash_t`**（读者 RAII handle，写者忙等归零）
- **写多、要稳定遍历、键是内部整数/对象、已有外层锁协议或元素是复杂链表** → **`hash_table_t`**（分片 rw_lock 或复用调用方锁）
- 两个无锁都不满足（如需要遍历、或 CAS 重试风暴）→ 回退 `hash_table_t`

**三实现的"同题不同解"再论**：

| | `LF_HASH` | `ut_lock_free_hash_t` | `hash_table_t` |
|---|---|---|---|
| 服务对象 | server 层任意全局容器（对象有构造/析构/键提取） | 一个统计需求（整数对、高频 inc/dec） | InnoDB 内部写多索引（锁/页/字典） |
| 因此需要 | 任意键值 + 回调生命周期 | 位掩码 + CAS 整数 + 归零即删 | 侵入链指针 + 稳定遍历 + 写不退化 |
| 因此选择 | split-ordered list + hazard pointer（论文方案） | 开放寻址 + 引用计数（教科书方案） | 链地址 + 调用方锁/分片锁（经典教科书） |
| 读者代价 | 显式 pin/unpin（协议纪律） | 拿 handle（RAII 自动） | 无锁 peek 或 S 锁 confirm |
| 写者代价 | 扫 pins（从不阻塞读者） | 摘除时忙等读者归零 | 持锁（X 锁或调用方锁） |
| 扩容代价 | 零迁移（惰性 dummy） | 显式迁移 + optimize 上锁 | 无扩容（用户层处理） |

与其他结构的关系：三个 hash 是**键空间**型结构（并发/有锁字典）；`Link_buf` 是**位置空间**型无锁结构（有序完成跟踪，见 [`link_buf.md`](link_buf.md)）；有界 MPMC 队列见 [`queue.md`](queue.md)；分片计数器见 [`counter.md`](counter.md)。

---

## ★ 本机制里的工程实现技法

### 一、扩容三哲学：零迁移 / 显式迁移 / 不迁移

| 技法 | 落在哪 | 为什么（收益） | 代价 |
|---|---|---|---|
| 有序链 + 惰性 dummy | LF_HASH | 扩容只 CAS 一个整数，**数据永不搬动**（split-ordered list 的结构红利） | 哈希值废 1 位、全局一条链尾部扫描风险 |
| 追加数组 + 显式迁移 | ut_lock_free_hash_t | 开放寻址必须搬数据，但每次只搬"新旧交界"那一批 | 迁移期跨数组查找、optimize 上锁 |
| 无扩容 + 用户层三姿势 | hash_table_t | 写多场景扩容低频，把复杂度交给最懂场景的用户 | resize 停机式（swap 偷换 / 迁移 / 重建） |

### 二、内存回收三方案：写者扫 pins / 写者等归零 / 根本不用

"读者持指针期间写者安全释放"的三种答案：hazard pointer 的**写者扫 pins 从不阻塞读者**（读者付显式 pin 的代价）；引用计数的**读者拿 RAII handle 免显式管理**（写者付忙等归零的代价）；有锁的**锁保证无并发读者**（回收顺序完全由用户掌控）。同仓库的第四个答案 RCU（`MyRcuLock`，见 [`../lock/primitives/rcu.md`](../lock/primitives/rcu.md)）是"整体快照替换、读多写极少"的路数：读者只需计数 inc/dec，但**写者要等全体读者退出**——与 hazard pointer 的"写者从不阻塞读者"正相反，适合完全不同的场景。

### 三、两段式协议：把锁开销从探测路径剔除

LF_HASH 的"读 link → pin → 重读校验"与 hash_table_t 的"无锁 peek → 锁后 confirm"是**同一种思想的两个实例**：先用廉价的无锁/弱手段拿到候选，再用强手段（重读校验 / 上锁确认）验证候选仍有效——把昂贵操作从每次探测中省掉，只有"真命中"才付全价。

### 四、逻辑删除与物理删除分离 + helping

| 技法 | 落在哪 | 为什么（收益） | 代价 |
|---|---|---|---|
| link 低位 DELETED 标志 | LF_HASH | 读者永远看到自洽链表，删除可两步完成 | 每步读都要抹位；被删节点短暂残留链上 |
| helping（路过顺手摘除） | `my_lfind` 的 DELETED 分支 | 维持"置标志次数 == 摘链次数"不变量，链不残留垃圾 | **任意读线程可能承担删除线程的工作**（读路径做写操作） |
| 毒值 HASH_INVALIDATE | hash_table_t（debug） | 把"删除后仍被访问"的悬垂 bug 变成必现崩溃 | 仅 debug |

### 五、pin 数量最小化与协议纪律

| 技法 | 落在哪 | 为什么（收益） | 代价 |
|---|---|---|---|
| 3 pin 滑动窗口 | `my_lfind` | 遍历 + 结果持有共 3 个 pin 就够（编译期 `LF_REQUIRE_PINS(3)` 强制） | 协议复杂：只能向上复制、返回时 0/1 不摘 2 必须摘 |
| `do{读;pin;重读校验}` 循环 | 所有 pin 点 | 解 ABA 的标准模板 | 每步两次读 + 一次写 |
| `sizeof(LF_PINS) == 64` 硬断言 | 结构定义 | 防伪共享 | 结构内部布局被 ABI 钉死 |
| 版本化空闲栈 | `pinstack_top_ver` 高 16 版本 + 低 16 索引 | 一个原子字同时解 ABA 与回收复用 | 16 位版本回绕风险（理论存在） |

### 六、random_match 一石二鸟

随机 dive + 回绕：① 随机取样（MDL 近似 LRU 淘汰，match 只有一行 `return (lock->m_fast_path_state.load() == 0);`）② `rand_val=0` 退化成全表遍历（Acl_cache flush、connection_delay 填表）——**一个 API 覆盖两种需求**。代价：可能找不到目标（返回 NULL，调用方重试）。

### 七、分片锁多对一 + 外部锁映射一致性

1 把 rw_lock 保护多桶（`(hash % n_cells) % n_sync_obj` 步长映射），在"锁粒度"与"锁数量"间折中；lock_sys 更进一步——**分片必须按 cell_id 算**，保证"同桶 lock queue 同 shard"的硬约束（容器锁与业务锁必须对齐的典型案例，详见锁文档）。

### 八、swap 偷换：并发 resize 下的结构置换

`buf_pool_resize_hash` 不释放 page_hash 本体，只偷换 `cells`/`n_cells`——配合 confirm 协议，实现"读者无感"的桶数变更。这是"原子指针交换"替代"整体重建"的经典手法（与 RCU 的整体快照替换同源，见 [`../lock/primitives/rcu.md`](../lock/primitives/rcu.md)）。

### 九、魔法值状态机：把生命周期编码进数据本身

ut_lock_free_hash_t 用三个特殊 key + 三个特殊 val 把"空/删除/迁移"塞进整数本身，省掉每 tuple 的状态字段；LF_HASH 用 link 低位 DELETED 位、hashnr 的 odd/even 位同理——**用整数的位/值域承载状态**，是无锁结构"一个原子变量多种语义"的通用手法（同族的还有 queue.md 里 MSB 占用位、ticket 的 63+1 编码）。

---

## 面向二次开发

**LF_HASH**：新增用户 = 决定"元素是否平凡"→ 平凡走 `lf_hash_init`（memcpy 语义），非平凡走 `lf_hash_init2/3` + 三回调（ctor 初始化昂贵成员、reinit 重置、dtor 析构）。**新增遍历能力不可行**（无 `lf_hash_iterate`），只能用 `random_match(0, 全匹配)` 变通——若遍历是高频操作，选别的容器。

**ut_lock_free_hash_t**：新用户实现 `ut_hash_interface_t` 即可，但注意它**只支持整数键值**——通用对象容器请用 `LF_HASH`。`del_when_zero` 语义：只有"值归零即删"的引用计数场景才开 true，否则 `dec` 到负数会累积 `(key, 负数)`。`initial_size` 必须 2 的幂（`ut_a(ut_is_2pow)` 断言），因为位掩码取模。

**hash_table_t**：新用户元素必须自带链指针成员；hash 值自己算（`page_id.hash()` / `ut::hash_string()`）；锁自己管（SYNC_NONE）或走 `ib_create` + `hash_create_sync_obj`（SYNC_RW_LOCK，需 2 的幂锁数 + 注册 latch_id）。`heap` 字段只在 SYNC_NONE 下合法（`hash_get_heap` 有 `ut_ad(n_sync_obj == 0)`）。resize 三姿势按场景选：读者无感要 swap 偷换；要改分片参数要 HASH_MIGRATE 停机式；可接受整体停摆直接重建。

**社区边界澄清**：LF_HASH 是 MySQL 社区版与 MariaDB 共有的历史组件（源自 MySQL AB 时代）；Percona 未改其核心。**源码中无论文引用注释**——引用两篇论文是本文从实现特征反推的对应关系，请勿声称"源码引用了论文"。

---

## 坑与已知缺陷汇总

### LF_HASH

1. **pin 数量上限 65536**（`LF_PINBOX_MAX_PINS`，16 位索引）：`lf_pinbox_get_pins` 超限返回 `nullptr`，调用方必须检查（connection_delay 检查了）。文件头 `/* QQ: TODO multi-pinbox */` 承认这是未竟事项。
2. **`put_pins` 的死锁警告**（注释原文带 XXX）：put_pins 会清空 purgatory，**若有线程 pin 着你 purgatory 里的对象又等你做某事 → 死锁**——"only free pins when all work is done and nobody can wait for you"。这是 MDL_context 在连接关闭时才 put pins 的原因。
3. **random_match 可能失败**：随机 dive 落到链尾、或目标被并发销毁/复用 → 返回 NULL，调用方要重试（MDL 的淘汰逻辑就是这么写的）。
4. **内存只增不减**：普通节点删除后进 purgatory → 空闲栈 → 池复用，除非 destructor（destroy 路径）否则**不归还 OS**。PFS 注释抱怨过：`the usage of LF_HASH introduces some memory allocation... to use a lock-free, malloc-free hash code table`——分配器池空时 insert 热路径会 malloc，与 PFS 的 malloc-free 设计冲突。
5. **文件头 @todo**："try to get rid of dummy nodes? / for non-unique hash, count only _distinct_ values"——非唯一哈希的 count 语义是未竟事项。
6. **linsert 的冲突节点指针不可靠**（注释原文）——API 只返回错误码不返回已有对象，是刻意设计。
7. **`lf_hash_destroy` 非线程安全**（`lf_alloc_destroy` 注释直言 "not thread safe"，还幽默一句 "don't put your cat in a microwave"）：必须保证无并发访问（MDL_map::destroy 前置 `@pre It must be empty.`）。**不存在 `lf_hash_clear`**。

### ut_lock_free_hash_t

1. **`optimize` 用 mutex**：并非 100% 无锁（注释自述"选简单与可维护性"）。
2. **`await_release_of_old_references` 忙等**：注释承认"若太慢可改 lazy deletion list"——目前是 `yield()` 自旋。
3. **并发 del/inc 顺序未定义**（`del` 的完整注释）：调用方要接受或上层阻止。
4. **`guess_position` 注释**：承认 hash 函数可能碰撞太多（"if this one turns out to generate too many collisions" 就换）。
5. **内存不归还**：空壳对象进 `m_hollow_objects` 延迟到析构才释放；数组节点只增不减。
6. **非线程安全析构**：构造/析构要求单线程（`buf_stat_per_index_t` 在 bp 初始化/关闭时单线程调用）。

### hash_table_t

1. **SYNC_NONE 模式"锁是约定不是机制"**：新代码极易漏锁，`hash_assert_can_modify` 只在 debug 抓到。
2. **无扩容**：`set_n_cells` 要求持全部 X 锁（注释原文），等于"扩容必须停机式"；三姿势都重量级。
3. **链可退化为 O(n)**：桶数固定 + 键偏斜时单链变长；素数桶只是缓解不是消除。
4. **resize 窗口的 confirm 复杂度**：偷换 n_cells 期间读者反复换锁重试（最坏自旋）。

---

## 易混淆概念汇总

- **`LF_HASH`（无锁，任意对象）≠ `ut_lock_free_hash_t`（无锁，整数对）≠ `hash_table_t`（有锁，链地址）**：三个哈希的定位差异见「选择指南」再论表。
- **LF_HASH 的"无锁"是 lock-free 不是 wait-free**：CAS 失败会重试（`LF_BACKOFF`）；只有 `LF_DYNARRAY` 的扩展自嘲 "wait-free, not lock-free ;-)"。
- **`MY_LF_ERRPTR`（OOM 哨兵）≠ `NULL`（未找到）**：调用方必须区分。
- **hazard pointer 的 pin ≠ 引用计数**：pin 只是"我暂时在用"的声明，不增减对象计数——所以有 pin 数上限（4 个）；ut_lock_free_hash_t 的 handle 才是真引用计数。
- **逻辑删除 ≠ 物理删除**：DELETED 标志后节点还在链上（读者跳过），摘链才真离链，free 还要等 purgatory 扫描。
- **`lf_hash_iterate` / `lf_hash_clear` / `LF_SLIST`（在 lf.h 中）不存在**——旧文档/口误常见；遍历只能靠 `random_match` 变相实现。
- **`ut_lock_free_hash_t`（无锁）≠ `hash_table_t`（hash0hash，链地址 + 可选分片 rw_lock）**：后者才是 `lock_sys`（`rec_hash`/`prdt_hash`）、`dict_sys`、AHI 用的；前者只在 bp 统计。
- **`ut_lock_free_cnt_t`（引用计数）≠ `ut_lock_free_hash_t`（hash）**：前者是后者数组节点的配套计数器，定义在同一头文件。
- **`hash_cell_t` 桶数组 ≠ 链节点**：链就是元素自己的 `NAME` 指针串起来的，没有独立节点对象（与 LF_HASH 的 `LF_SLIST` 隐藏头不同）。
- **"锁后 confirm"只存在于 SYNC_RW_LOCK 模式**：SYNC_NONE 模式 `hash_get_first`/`hash_get_next` 只是裸指针遍历，锁由调用方语境保证。
- **`hash_calc_hash` / `hash_table_find` 在 8.0.39 均不存在**——hash 值由调用者算，勿引用这两个符号（旧资料常见错误）。
- **`hash_get_heap` 的 heap ≠ 元素内存**：只存 AHI 的 ha_node_t 索引节点，元素本体在缓冲池。
- **`buf0stats.h` 的 `buf_stat_per_index`（全局单例）≠ `buf_pool->stat`（`buf_pool_stat_t`）**：前者按索引统计页数（用无锁 hash），后者按 pool 统计 gets/reads 等计数（用 `Counter`）。

---

## 参考

**论文 / 经典算法**
- O. Shalev, N. Shavit. *Split-Ordered Lists: Lock-Free Extensible Hash Tables*. JACM 2006.（LF_HASH 的 bit-reversed 排序、dummy 哨兵、无 rehash 扩容出处；⚠️ 源码无引用注释，为从实现反推）
- M. M. Michael. *Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects*. IEEE TPDS 2004.（LF_HASH 的 4 pin + 延迟回收 + ABA 防护出处；同上）
- D. E. Knuth. *The Art of Computer Programming, Vol. 3*（开放寻址 + 线性探测与链地址法的经典出处；同上）
- 无锁引用计数的按 CPU 分片手法参考 M. Herlihy & N. Shavit *The Art of Multiprocessor Programming* 的 combining/snapshot 思想（同上）

**官方文档**
- *MySQL 8.0 Reference Manual → How MySQL Uses Memory*（`key_memory_lf_node` 相关）

**相关文档**
- LF_HASH 最大用户 MDL_map 的用法（placement new 三回调、随机淘汰阈值 `unused > 1000 && > count*0.25`、LF_HASH 取代 5.6 分区的演进）见 [`../lock/transactional/mdl.md`](../lock/transactional/mdl.md)
- `hash_table_t` 的用户视角：lock_sys（`rec_hash`/`prdt_hash` 及其 `locksys::Latches` 分片锁）见 [`../lock/transactional/innodb_trx_lock.md`](../lock/transactional/innodb_trx_lock.md)；buffer pool `page_hash` 见 [`../../innodb/buffer_pool.md`](../../innodb/buffer_pool.md)；AHI 见 [`../../innodb/ahi.md`](../../innodb/ahi.md)
- 同一问题的另一个方案 RCU（`MyRcuLock`，整体快照 + 读计数等零）见 [`../lock/primitives/rcu.md`](../lock/primitives/rcu.md)；`mt_fast_modulo_t` 依赖的 `Seq_lock` 见 [`../lock/primitives/seq_lock.md`](../lock/primitives/seq_lock.md)
- 数据结构归属与全量清单见 [`../README.md`](../README.md)
