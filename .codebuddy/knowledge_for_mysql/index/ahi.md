# InnoDB 自适应哈希索引（AHI）深度解析

> 基于 MySQL 8.0.39 源码，核心文件 `storage/innobase/btr/btr0sea.cc` / `include/btr0sea.h` / `ha/ha0ha.cc`。涵盖：**它到底是什么**（一个可丢弃的启发式缓存，不是磁盘索引）、**哈希键如何"自适应"**（前缀长度学习算法）、**查询路径的 8 道门禁**、**建/删/改的维护协议与 nowait 哲学**、**★ 分片按 (space_id, index_id) 路由导致的单索引热点问题**、**关闭 AHI 可能 600 秒后 crash 的危险代码**。

> **边界**：本篇讲 **AHI 本身**。B-tree 的游标搜索与页面结构见 [`btr.md`](btr.md)；Buffer Pool 的读页与预读见 [`buffer_pool.md`](../innodb/buffer_pool.md)；change buffer 见 [`ibuf.md`](ibuf.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [整体结构](#整体结构)
  - 构建与维护
    - [★ 哈希键的自适应算法（前缀长度学习）](#-哈希键的自适应算法前缀长度学习)
    - [维护路径（建 / 删 / 改）](#维护路径建--删--改)
  - 读路径
    - [查询路径 `btr_search_guess_on_hash`](#查询路径-btr_search_guess_on_hash)
  - 并发与治理
    - [★ 锁协议与分片](#-锁协议与分片)
    - [开关、内存与监控](#开关内存与监控)
- [相关的系统变量](#相关的系统变量)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

AHI 是 InnoDB 在 **B-tree 之上自动建立的内存哈希索引**。它把"叶子页内的记录"按**键的前缀**做哈希，一次哈希直接定位到 `rec_t*`（记录的内存指针），跳过 B-tree 从根到叶的下降过程。

一句话：**它是 B-tree 的"捷径缓存"，丢了不影响正确性**。

源码里最精确的定义（`include/btr0sea.h`）：

```c
    /** The adaptive hash table, mapping dtuple_hash values to rec_t pointers
    on index pages. For any hash value at most one pointer is hold. Is protected
    by the part's latch. ... */
    alignas(ut::INNODB_CACHE_LINE_SIZE) hash_table_t *hash_table;
```

> **注意两个词**：① `dtuple_hash values → rec_t*`，存的是**内存指针**，不是主键值、不是页号；② **"For any hash value at most one pointer"**——同一个哈希值最多只保留一个指针。

### 用途

| 场景 | AHI 的作用 |
|------|-----------|
| **等值点查**（`=` / `IN` / 唯一键查） | 把 `height` 次页下降 + 页内二分 → 一次哈希 + 一次页 latch |
| 范围扫描 | **几乎无用**（AHI 只定位单条，且每组只存一条） |
| 全表扫描 | 无用（走预读，且会污染 AHI 统计） |

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.5 及更早 | **全局单一 `btr_search_latch`**（一把 rw_lock 保护整张哈希表）→ 高并发点查的著名瓶颈 |
| 5.6 | 引入 `btr_search_info_t` 前缀自适应（n_fields / n_bytes） |
| **8.0.30** | **★ 分片改造**：按 `(space_id, index_id)` 哈希到 `btr_ahi_parts`（默认 8）个 part，每个 part 独立的 `rw_lock_t` + 独立 `hash_table_t`；新增只读变量 `innodb_adaptive_hash_index_parts`（1–512）；`btr_search_info_t` 改名 `btr_search_t`；哈希表以 **0 个内部同步对象**创建（内容完全由 part latch 保护）；`n_fields`/`n_bytes`/`left_side` 打包成 64 位原子结构 `btr_search_prefix_info_t` |
| 8.0.39 | 上述结构稳定 |

> **演进动因**（8.0.30 release notes 记载）：*"Enabling the adaptive hash index on a high-concurrency instance caused temporary AHI search latch contention"*。

---

## 理论基础

### 设计思想与权衡

#### 一、为什么需要它：B-tree 的"结构开销"

当 BP 足够大、页都在内存时，点查的开销主要**不再是磁盘 I/O，而是 B-tree 本身的结构**：

```
一次点查 = height 次页 latch（根→内节点→叶）
         + 每层页内的 directory slot 二分
         + 每层页内的记录比较
```

AHI 的定位就是**消除这个结构开销**——直接用键的哈希值换出记录地址。

#### 二、为什么可以"可丢弃"（这是整个设计的基石）

AHI 不持久化、不参与崩溃恢复、不影响事务语义。它只是"记住上次查到哪"的缓存。所以：

> **一切维护操作都可以"放弃"**——拿不到锁就算了、内存不够就不建、猜错了就走 B-tree。

这是理解 AHI 所有 `nowait` 的钥匙。源码注释说得很直白（`btr0sea.cc`）：

```c
  /* The AHI is supposed to be heuristic for speed-up. When adding a block
  to index, waiting here for the latch would defy the purpose. We will try
  to add the block to index next time. However, for updates this must
  succeed so the index doesn't contain wrong entries. */
```

**唯一例外**：`update=true` 的重建（页分裂/合并后）与删除路径**必须**阻塞等锁——因为不更新会留下**错误的**（悬垂的）哈希项，那就不是"慢"而是"错"了。

#### 三、★ AHI 是"允许有瑕疵、惰性修补"的结构

这是最反直觉的一点。源码注释原文（`btr0sea.cc`）：

```c
/** Updates a hash node reference when it has been unsuccessfully used in a
search which could have succeeded with the used hash parameters. This can
happen because when building a hash index for a page, we do not check
what happens at page boundaries, and therefore there can be misleading
hash nodes. Also, collisions in the hash value can lead to misleading
references. This function lazily fixes these imperfections in the hash
index. */
```

**三个"不保证"**：

1. **不检查页边界**——建 AHI 时只按当前页内记录算哈希，不关心相邻页；
2. **可能有哈希碰撞**——不同的键可能算出同一个 fold；
3. 因此**哈希项可能是"误导性"的**（misleading）。

**对策是双重**：
- 查询时做**三重校验**（space_id + 页上的 index_id + 真正比较记录）；
- 猜错时**惰性修补**（`btr_search_update_hash_ref` 补上正确项）。

> **这与其他索引设计的哲学完全不同**：B-tree 是"必须精确"的，AHI 是"先猜、猜错再补"的。

#### 四、代价清单

| 代价 | 说明 |
|------|------|
| **内存** | 哈希表 + 节点（每节点 24 字节）来自 buffer pool 的空闲帧 |
| **CPU** | 每次搜索都要更新统计（`hash_analysis`、`n_hash_potential`） |
| **latch 争用** | 读路径要拿分片 S 锁；任何页重组/分裂/淘汰都要拿 X 锁 |
| **写路径负担** | INSERT/DELETE/UPDATE 都要维护哈希项 |
| **关闭时的风险** | 见 [核心实现六](#核心实现六开关内存与监控)——可能 600 秒后 crash |

### 他库对比

| 数据库 | 类似机制 |
|--------|---------|
| **MySQL / InnoDB** | AHI（自适应、可丢弃、按前缀哈希） |
| **Oracle** | 无直接对应（靠 buffer cache + 索引本身） |
| **PostgreSQL** | 无（B-tree 本身 + OS page cache） |

> AHI 是 InnoDB 相对独特的设计，源于它"一切皆索引组织表 + 点查密集"的 OLTP 定位。

---

## 核心实现

### 整体结构

#### 1.1 三层结构

```
btr_search_sys（全局单例，btr_search_sys_t*）
  └── parts[] : search_part_t[btr_ahi_parts = 8]
        ├── latch       : rw_lock_t        ← 每个分片一把（cache line 对齐）
        ├── hash_table  : hash_table_t*    ← 每个分片一张（★ 内部 0 个锁）
        └── free_block_for_heap : atomic<buf_block_t*>

hash_table_t（通用哈希表，hash0hash.h）
  └── cells[] : hash_cell_t{ void *node }   ← 链地址法
        └── ha_node_t{ uint64_t hash_value; ha_node_t *next; const rec_t *data; }
```

`btr_search_sys_t` 的定义（`include/btr0sea.h`）：

```c
class btr_search_sys_t {
 public:
  btr_search_sys_t(size_t hash_size);

  class search_part_t {
   public:
    void initialize(size_t hash_size);
    /** The latch protecting the adaptive search part: this latch protects the
    (1) positions of records on those pages where a hash index has been built.
    NOTE: It does not protect values of non-ordering fields within a record from
    being updated in-place! We can use fact (1) to perform unique searches to
    indexes. */
    alignas(ut::INNODB_CACHE_LINE_SIZE) rw_lock_t latch;
    /** The adaptive hash table, mapping dtuple_hash values to rec_t pointers
    on index pages. ... */
    alignas(ut::INNODB_CACHE_LINE_SIZE) hash_table_t *hash_table;
    /** A pointer to a free block that the heap in the hash table may use for
    adding new hash nodes. ... */
    std::atomic<buf_block_t *> free_block_for_heap;
  };

  /** Partitions of the AHI system. */
  ut::unique_ptr_aligned<search_part_t[]> parts;
};

extern btr_search_sys_t *btr_search_sys;
```

**三个成员各自 cache line 对齐**——`latch` 与 `hash_table` 分开占，避免读者注册 latch 时的 false sharing。

> **★ latch 保护什么（关键语义）**：注释明说——**只保护"记录在页内的位置"，不保护记录内容**。正因如此，AHI 能用于唯一索引搜索（跳过 `dict_index_t::lock`），也解释了为什么 UPDATE 只改非排序字段时不需要动 AHI。

#### 1.2 `hash_table_t` 是通用表，但 AHI 不用它的内部锁

`hash_table_t` 同时服务 `buf_pool->page_hash`、`lock_sys->rec_hash` 和 AHI。区分点：

| | `page_hash` | **AHI 的哈希表** |
|---|---|---|
| 同步类型 | `HASH_TABLE_SYNC_RW_LOCK`（内部 `rw_locks[]` 分片锁） | **`HASH_TABLE_SYNC_NONE`**（`n_sync_obj = 0`，内部**完全无锁**） |
| 保护者 | 自身的 rw_locks | 外层 `search_part_t::latch` |

证据（`btr0sea.cc`）：

```c
  hash_table = ib_create((hash_size / btr_ahi_parts), LATCH_ID_HASH_TABLE_MUTEX,
                         0, MEM_HEAP_FOR_BTR_SEARCH);
```

第三个参数 `n_sync_obj = 0`。

**哈希节点**（`include/ha0ha.h`）：

```c
struct ha_node_t {
  /** hash value for the data  */
  uint64_t hash_value;
  /** next chain node or NULL if none */
  ha_node_t *next;
#if defined UNIV_AHI_DEBUG || defined UNIV_DEBUG
  buf_block_t *block;
#endif
  /** pointer to the data */
  const rec_t *data;
};
```

生产编译下 `sizeof(ha_node_t) = 24`。由 `rec_t*` 反查 block 用 `buf_block_from_ahi`。

#### 1.3 ★ 页级字段：`buf_block_t::ahi_t`

> **矫正**：`search_index` / `n_fields` / `n_bytes` / `curr_*` 这些是 **5.6 及更早**的字段名，**8.0.39 全部不存在**。现在打包成 `ahi_t` + 一个 64 位原子结构。

**前缀信息**（`include/buf0buf.h`）：

```c
struct alignas(alignof(uint64_t)) btr_search_prefix_info_t {
  /** recommended prefix: number of bytes in an incomplete field */
  uint32_t n_bytes;
  /** recommended prefix length for hash search: number of full fields */
  uint16_t n_fields;
  /** true or false, depending on whether the leftmost record of several records
  with the same prefix should be indexed in the hash index */
  bool left_side;
  ...
};
```

三个字段打包成 8 字节 → 可**无锁原子读写**（`static_assert(is_always_lock_free)`）。

`buf_block_t::ahi_t` 里**两份** prefix_info（**极易混淆**）：

| 字段 | 含义 | 修改条件 |
|------|------|---------|
| `recommended_prefix_info` | **建议值**（从 index 拷来） | 持 block 的 S/X latch 时 |
| `prefix_info` | **实际生效值** | 只能持 AHI part 的 **X latch** 时 |
| `index` | 该 block 属于哪个索引的 AHI（`atomic<dict_index_t*>`） | 见 assert |
| `n_pointers` | DEBUG 用：指向本页的哈希项数 | X latch |

**不变式**：本 block 放进 AHI 的所有记录，**都是用当前这份 `prefix_info` 折叠出来的**。

`n_hash_helps` 在 `buf_block_t` 上但**不在 `ahi_t` 内**——源码注释解释得很清楚：`n_hash_helps` 应该放进 `ahi_t`，但放外面能让 `made_dirty_with_no_latch` 复用那 8 字节对齐空间，**给这个高频对象省 8 字节**。

#### 1.4 索引级：`btr_search_t`

挂在 `dict_index_t::search_info`（`dict0mem.h`）。字段（`include/btr0sea.h`）：

| 字段 | 含义 |
|------|------|
| `ref_count` | 该索引有多少 block 在 AHI 里（关闭/drop 时要等它归零） |
| `root_guess` | 上次取的 root 页（加速） |
| `hash_analysis` | 分析节流计数（≥ 17 才做一次分析） |
| `last_hash_succ` | ★ 上次 AHI 是否成功——**进入 AHI 查找的前置条件** |
| `n_hash_potential` | 连续"本可用 AHI 成功"的次数（0..105） |
| `prefix_info` | 索引级**推荐**前缀 |

> **注意**：即使 AHI 关闭，`search_info` 也照样创建（`btr0sea.cc`：*"We always create search info whether adaptive hash index is enabled or not."*）。

索引级还有开关 `dict_index_t::disable_ahi`（`dict0mem.h`）——目前只用于 intrinsic 临时表与 SDI 表，因为**它们的 index id 不唯一**，而 AHI 校验依赖 index id。

**★ 完整定义**（⚠️ 真名是 `btr_search_t`，**不存在 `btr_search_info_t`**——只有函数 `btr_search_info_create()` 保留了旧名）：

```cpp
struct btr_search_t {
  /** Number of blocks in this index tree that have search index built i.e.
  block->ahi.index points to this index. */
  std::atomic<size_t> ref_count;

  /** @{ The following fields are not protected by any latch.
  Unfortunately, this means that they must be aligned to the machine word, i.e.,
  they cannot be turned into bit-fields. */

  /** the root page frame when it was last time fetched, or NULL. */
  buf_block_t *root_guess;
  /** when this exceeds BTR_SEARCH_HASH_ANALYSIS, the hash analysis starts; this
  is reset if no success noticed. */
  std::atomic<uint64_t> hash_analysis;
  /** true if the last search would have succeeded, or did succeed, using the
  hash index; NOTE that the value here is not exact: it is not calculated for
  every search, and the calculation itself is not always accurate! */
  bool last_hash_succ;
  /** number of consecutive searches which would have succeeded, or did succeed,
  using the hash index; the range is 0 .. BTR_SEARCH_BUILD_LIMIT + 5. */
  std::atomic<uint64_t> n_hash_potential;
  /** @} */

  std::atomic<btr_search_prefix_info_t> prefix_info;
  static_assert(decltype(prefix_info)::is_always_lock_free);
#ifdef UNIV_SEARCH_PERF_STAT
  std::atomic<ulint> n_hash_succ;
  std::atomic<ulint> n_hash_fail;
  std::atomic<ulint> n_patt_succ;
  std::atomic<ulint> n_searches;
#endif /* UNIV_SEARCH_PERF_STAT */
#ifdef UNIV_DEBUG
  ulint magic_n;
#endif
};
```

> ⚠️ **`n_fields` / `n_bytes` / `left_side` 已不再是独立字段**——8.0.30+ 打包进 `std::atomic<btr_search_prefix_info_t> prefix_info`（见下）。
> ⚠️ `n_hash_succ` / `n_hash_fail` / `n_patt_succ` / `n_searches` **只在 `UNIV_SEARCH_PERF_STAT` 下存在**，默认构建**不编译**。所以"`hash_analysis`、`n_hash_potential`"才是总在生效的统计项。
> ⚠️ **不存在 `n_patt_move`**（UNIV_SEARCH_PERF_STAT 下也没有这个字段）。

**★★ `btr_search_prefix_info_t` 真身不在 `btr0sea.h`，而在 `buf0buf.h:1666`**：

```cpp
/** Structure used by AHI to contain information on record prefixes to be
considered in hash index subsystem. It is meant for using as a single 64bit
atomic value, thus it needs to be aligned properly. */
struct alignas(alignof(uint64_t)) btr_search_prefix_info_t {
  /** recommended prefix: number of bytes in an incomplete field */
  uint32_t n_bytes;
  /** recommended prefix length for hash search: number of full fields */
  uint16_t n_fields;
  /** true or false, depending on whether the leftmost record of several records
  with the same prefix should be indexed in the hash index */
  bool left_side;

  bool equals_without_left_side(const btr_search_prefix_info_t &other) const {
    return n_bytes == other.n_bytes && n_fields == other.n_fields;
  }
  bool operator==(const btr_search_prefix_info_t &other) const { ... }
  bool operator!=(const btr_search_prefix_info_t &other) const { ... }
};
```

> ★★ 整个结构 8 字节（4+2+1+padding），所以 `std::atomic<...>` 能做到 **lock-free 单指令读写**——这正是 8.0.30 重构的核心收益：**一次原子 load 同时拿到 `(n_fields, n_bytes, left_side)` 三元组**，避免三者不一致。源码里 `static_assert(decltype(prefix_info)::is_always_lock_free);` 把这个前提钉死。
> ★ `equals_without_left_side()` 用于 insert 路径——只关心前缀长度、不关心方向。

**★ 索引级推荐 → 页级使用的传递链**：

```
【产生】btr_search_info_update_hash()          btr0sea.cc:407
          └─ info->prefix_info = {n_bytes, n_fields, left_side}   ← 单条 8 字节原子写
【复制到 block】btr_search_update_block_hash_info()   btr0sea.cc:548
          └─ block->ahi.recommended_prefix_info = info->prefix_info.load();
【落到实际使用前缀】btr_search_build_page_hash_index()  btr0sea.cc:1579
          ├─ block->ahi.prefix_info = prefix_info;
          └─ block->ahi.index = index;
【读取用于 fold】
   ├─ rec_hash(..., block->ahi.prefix_info.*, ...)         btr0sea.cc:1136（删除）
   ├─ rec_hash(..., recommended_prefix_info.*, ...)        btr0sea.cc:1480/1503（建立）
   └─ rec_hash(..., block_prefix_info.*, ...)              btr0sea.cc:623（修复）
【查询侧】btr_search_guess_on_hash()   btr0sea.cc:840
   └─ prefix_info = info->prefix_info.load();  （索引级推荐）
```

> ★★★ **不变式**：删除（`btr0sea.cc:1136`）与修复（`btr0sea.cc:623`）**必须**用 `block->ahi.prefix_info`（当初建索引时用的那个），**不能**用 `info->prefix_info`（当前推荐）。若两者不同，说明页即将被重建——修复路径直接跳过，删除路径回到 `for(;;)` 重试。**否则会往哈希表塞入一个永远删不掉的条目，破坏 AHI**。

#### 1.5 阈值常量

| 常量 | 值 | 含义 |
|---|---|---|
| `BTR_SEARCH_PAGE_BUILD_LIMIT` | 16 | 页级：`n_hash_helps > 页内记录数 / 16` |
| `BTR_SEARCH_BUILD_LIMIT` | 100 | 索引级：`n_hash_potential >= 100` |
| `BTR_SEARCH_HASH_ANALYSIS` | 17 | 每 17 次搜索才做一次完整分析（省 CPU） |
| `BTR_SEARCH_ON_PATTERN_LIMIT` | 3 | 连续 3 次才尝试"模式捷径" |
| `BTR_SEARCH_ON_HASH_LIMIT` | 3 | 连续 3 次才尝试哈希捷径 |

**"只给热点页建"的判定**（`btr0sea.cc`）：

```c
  if (info->n_hash_potential >= BTR_SEARCH_BUILD_LIMIT &&
      block->n_hash_helps >
          page_get_n_recs(block->frame) / BTR_SEARCH_PAGE_BUILD_LIMIT) {
    if (!block->ahi.index ||
        block->n_hash_helps > 2 * page_get_n_recs(block->frame) ||
        block->ahi.recommended_prefix_info.load !=
            block->ahi.prefix_info.load) {
      return true;
    }
  }
```

> **推论**：**页越小越容易进 AHI**。一个 1000 条记录的页要被"帮" 62 次以上；一个 16 条记录的页只要 1 次。

#### 1.6 ★ 统计字段故意不加锁

```c
/** Updates the search info of an index about hash successes. NOTE that info
is NOT protected by any semaphore, to save CPU time! Do not assume its fields
are consistent.
```

代价是这些字段**不能是 bit-field**（必须机器字对齐）。这是 AHI 的核心取舍：统计只是启发式，允许"脏"。

---

### ★ 哈希键的自适应算法（前缀长度学习）

#### 2.1 输入：字节级匹配数

叶子层搜索时（`btr0cur.cc`）走的是 `page_cur_search_with_match_bytes`（不是普通的 `_with_match`），多返回 `up_bytes` / `low_bytes`：

```c
  } else if (height == 0 && btr_search_enabled &&
             !dict_index_is_spatial(index)) {
    /* The adaptive hash index is only used when searching
    for leaf pages (height==0), but not in r-trees.
    We only need the byte prefix comparison for the purpose
    of updating the adaptive hash index. */
    page_cur_search_with_match_bytes(block, index, tuple, page_mode, &up_match,
                                     &up_bytes, &low_match, &low_bytes,
```

> **AHI 只在叶子页（height == 0）上构建和查找**；R-tree 完全不支持。

#### 2.2 节流：`btr_search_info_update`

```c
static inline void btr_search_info_update(btr_cur_t *cursor) {
  const auto index = cursor->index;
  if (dict_index_is_spatial(index) || !btr_search_enabled) return;
  if (cursor->flag == BTR_CUR_HASH_NOT_ATTEMPTED) return;

  const auto hash_analysis_value = ++index->search_info->hash_analysis;
  if (hash_analysis_value < BTR_SEARCH_HASH_ANALYSIS) {
    return;      // ← 每 17 次才真正分析一次
  }
  btr_search_info_update_slow(cursor);
}
```

#### 2.3 ★ 核心算法：`btr_search_info_update_hash`

**这是全文最难的一段**，也是"自适应"三个字的全部含义。

**思想**：B-tree 二分定位后，搜索键落在 `low` 和 `up` 两条相邻记录之间。算法反推——**要区分这两条记录，最少需要多少个完整字段 + 多少个字节？** 这个长度就是哈希前缀。

```c
static void btr_search_info_update_hash(btr_cur_t *cursor) {
  ...
  if (dict_index_is_ibuf(index)) {
    /* So many deletes are performed on an insert buffer tree
    that we do not consider a hash index useful on it: */
    return;
  }

  const uint16_t n_unique = dict_index_get_n_unique_in_tree(index);
  const auto info = index->search_info;

  if (info->n_hash_potential != 0) {
    const auto prefix_info = info->prefix_info.load;

    /* 情况 A1：推荐前缀已覆盖全部 unique 列，且本次匹配到了 n_unique */
    if (prefix_info.n_fields == n_unique &&
        std::max(cursor->up_match, cursor->low_match) == n_unique) {
      info->n_hash_potential++;
      return;
    }

    /* 情况 A2：一般情况 —— 检查"本该命中" */
    const bool low_matches_prefix =
        0 >= ut_pair_cmp(prefix_info.n_fields, prefix_info.n_bytes,
                         cursor->low_match, cursor->low_bytes);
    const bool up_matches_prefix =
        0 >= ut_pair_cmp(prefix_info.n_fields, prefix_info.n_bytes,
                         cursor->up_match, cursor->up_bytes);
    if (prefix_info.left_side ? (!low_matches_prefix && up_matches_prefix)
                              : (low_matches_prefix && !up_matches_prefix)) {
      info->n_hash_potential++;
      return;
    }
  }

  /* 情况 B：重设推荐 */
  info->hash_analysis = 0;

  cmp = ut_pair_cmp(cursor->up_match, cursor->up_bytes,
                    cursor->low_match, cursor->low_bytes);
  if (cmp == 0) {
    info->n_hash_potential = 0;
    info->prefix_info = {0, 1, true};
  } else if (cmp > 0) {
    info->n_hash_potential = 1;
    if (cursor->up_match == n_unique) {
      info->prefix_info = {0, n_unique, true};
    } else if (cursor->low_match < cursor->up_match) {
      info->prefix_info = {0, (uint16_t)(cursor->low_match + 1), true};
    } else {
      info->prefix_info = {(uint32_t)(cursor->low_bytes + 1),
                           (uint16_t)cursor->low_match, true};
    }
  } else {
    /* 对称，但 left_side = false */
    ...
  }
}
```

**逐条解释**：

**情况 A（已有推荐，测试本该命中吗）**

AHI 每个"相等前缀组"**只存一条**（最左或最右，由 `left_side` 决定）。所以要让 AHI 能命中，必须**恰好有一条落在组边界上**：

| `left_side` | 存的是 | 命中条件 |
|---|---|---|
| `true` | 每组**最左** | `up` 匹配、`low` **不**匹配 |
| `false` | 每组**最右** | `low` 匹配、`up` **不**匹配 |

若两条都匹配 → 它们之间没有组边界 → 前缀太短 → 不计数。

**情况 B（重设推荐）**——比较 `(up_match, up_bytes)` vs `(low_match, low_bytes)`：

| 情形 | 新前缀 | 含义 |
|------|--------|------|
| `cmp == 0` | `{0, 1, true}` | 落在同一组内部，退化为 1 个字段 |
| `up_match == n_unique` | `{0, n_unique, true}` | 用全部 unique 列 |
| `low_match < up_match` | `{0, low_match+1, true}` | **多加一个完整字段**就能分开 |
| 字段数相同 | `{low_bytes+1, low_match, true}` | **在同一个不完整字段里多取一个字节** |
| `cmp < 0` | 对称，`left_side=false` | — |

> **这就是"自适应"的本质**：查询模式变了（比如从用 2 列变成用 3 列），前缀会自动变长/变短，无需人工干预。

#### 2.4 fold 计算：`rec_hash` 与 `dtuple_hash` 必须逐位一致

- **物理记录** → `rec_hash(rec, offsets, n_fields, n_bytes, seed, index)`（`rem0rec.ic`）
- **查询元组** → `dtuple_hash(tuple, n_fields, n_bytes, seed)`（`data0data.ic`）

两者都从 `seed = btr_hash_seed_for_record(index) = hash_uint64_pair(space, index_id)` 出发，先哈希前 `n_fields` 个完整字段（跳过 SQL NULL），若有 `n_bytes` 再哈希第 `n_fields` 个字段的**前 n_bytes 字节**（截断到 `min(len, n_bytes)`）。

底层 `ut::hash_binary` 在 `len >= 15` 时走 CRC32 加速；`hash_uint64` 用 Tabulation Hashing。

> **为什么 prefix_info 必须严格同步**：只要 `n_fields`/`n_bytes` 不一致，两边算出的 fold 就对不上，AHI 永远命中不了。

`btr_search_get_n_fields`（`btr0sea.cc`）：

```c
[[nodiscard]] inline uint16_t btr_search_get_n_fields(
    btr_search_prefix_info_t prefix_info) {
  return prefix_info.n_fields + (prefix_info.n_bytes > 0 ? 1 : 0);
}
```

**⚠️ 两个关键真名更正**：① **`rec_hash()` / `dtuple_hash()` 的定义体在 `.ic` 文件**（头文件只有声明带分号）；② **`rec_fold()` / `dtuple_fold()` / `ut_fold_binary()` 在 8.0.39 全部不存在**（全仓 0 命中），"fold" 只剩注释里的历史词汇。

**★ `rec_hash()` 完整定义**（`rem0rec.ic:836`）：

```cpp
static inline uint64_t rec_hash(const rec_t *rec, const ulint *offsets,
                                ulint n_fields, ulint n_bytes,
                                uint64_t hash_value,
                                const dict_index_t *index) {
  uint16_t i;
  const byte *data;
  ulint len;
  ulint n_fields_rec;

  ut_ad(rec_offs_validate(rec, nullptr, offsets));
  ut_ad(rec_validate(rec, offsets));
  ut_ad(n_fields > 0 || n_bytes > 0);

  n_fields_rec = rec_offs_n_fields(offsets);
  ut_ad(n_fields <= n_fields_rec);
  ut_ad(n_fields < n_fields_rec || n_bytes == 0);

  if (n_fields > n_fields_rec) {
    n_fields = n_fields_rec;        // 防御：不超出记录实际字段数
  }
  if (n_fields == n_fields_rec) {
    n_bytes = 0;                    // 已哈希全部字段，就不存在"半个字段"
  }

  for (i = 0; i < n_fields; i++) {
    data = rec_get_nth_field_instant(rec, offsets, i, index, &len);
    //  ★ instant ADD COLUMN 兼容版本

    if (len != UNIV_SQL_NULL) {     // ★★ SQL NULL 完全不参与 hash
      hash_value = ut::hash_binary(data, len, hash_value);
    }
  }

  if (n_bytes > 0) {                // 前缀截断
    data = rec_get_nth_field_instant(rec, offsets, i, index, &len);
    if (len != UNIV_SQL_NULL) {
      if (len > n_bytes) len = n_bytes;
      hash_value = ut::hash_binary(data, len, hash_value);
    }
  }

  return hash_value;
}
```

**★ `dtuple_hash()` 完整定义**（`data0data.ic:596`）——**结构必须与 `rec_hash` 逐位一致**：

```cpp
static inline uint64_t dtuple_hash(const dtuple_t *tuple, ulint n_fields,
                                   ulint n_bytes, uint64_t hash_value) {
  const dfield_t *field;
  ulint i;
  const byte *data;
  ulint len;

  for (i = 0; i < n_fields; i++) {
    field = dtuple_get_nth_field(tuple, i);
    data = (const byte *)dfield_get_data(field);
    len = dfield_get_len(field);

    if (len != UNIV_SQL_NULL) {
      hash_value = ut::hash_binary(data, len, hash_value);
    }
  }

  if (n_bytes > 0) {
    field = dtuple_get_nth_field(tuple, i);   // ★ 复用循环变量 i（此时 i == n_fields）
    data = (const byte *)dfield_get_data(field);
    len = dfield_get_len(field);

    if (len != UNIV_SQL_NULL) {
      if (len > n_bytes) len = n_bytes;
      hash_value = ut::hash_binary(data, len, hash_value);
    }
  }

  return (hash_value);
}
```

> ★★ **"逐位一致"的三条硬约束**：① 同样的 `seed`；② 同样的 `n_fields`/`n_bytes`；③ **同样跳过 SQL NULL**（不是"跳过该字段"，是"这一轮不加料"）。第三条最容易被忽略——若一边把 NULL 当 0 字节哈希、另一边跳过，结果立刻分叉。

**★ 种子与底层哈希**：

```cpp
static ulint btr_hash_seed_for_record(const dict_index_t *index) {
  ut_ad(index != nullptr);
  return btr_search_hash_index_id(index);      // = ut::hash_uint64_pair(index->space, index->id)
}

static inline uint64_t hash_uint64_pair(uint64_t n1, uint64_t n2) {
  return hash_uint64(hash_uint64(n1) ^ n2);     // ★ Tabulation Hashing
}

static inline uint64_t hash_binary(const byte *str, size_t len, uint64_t seed) {
  if (len >= 15) {
    /* 长字段快路径：前 8 字节 tab-hash 做种子，其余走硬件 CRC32 */
    const auto h = hash_uint64_pair(seed, detail::read_from_8(str));
    return hash_uint64_pair(h, ut_crc32(str + 8, len - 8));
  }
  /* 短字段：把剩余 <8 字节塞进一个 uint64 再折叠 */
  uint64_t h = seed;
  size_t i = 0;
  if (len & 8) { h = hash_uint64_pair(h, detail::read_from_8(str + i)); i += 8; }
  uint64_t last_part = 0;
  if ((len & 7) == 0) return h;
  if (len & 4) { last_part |= detail::read_from_4(str + i) << (16 + 8); i += 4; }
  if (len & 2) { last_part |= detail::read_from_2(str + i) << 8;         i += 2; }
  if (len & 1) { last_part |= detail::read_from_1(str + i);              i += 1; }
  return hash_uint64_pair(h, last_part);
}
```

> ★ 默认 `seed = 0xacb1f3526e25dd39`（`ut0rnd.h:102`），但 AHI 一律用 `btr_hash_seed_for_record(index)` 覆盖它——**同一索引的所有 fold 共享同一个 seed**，这样不同索引的同名值不会互相污染。

**★ 建索引时"每组只留一条"的落地**（`btr_search_build_page_hash_index`:1490-1523）：

```cpp
    if (hash_value != next_hash_value) {
      /* Insert an entry into the hash index */
      if (prefix_info.left_side) {
        hashes[n_cached] = next_hash_value;   // ★ left_side=true  → 取新组的第一条
        recs[n_cached] = next_rec;
        n_cached++;
      } else {
        hashes[n_cached] = hash_value;        // ★ left_side=false → 取上一组的最后一条
        recs[n_cached] = rec;
        n_cached++;
      }
    }
```

而哈希表层则保证"同一 hash_value 最多一个节点"——`ha_insert_for_hash_func()` 命中同 `hash_value` 时**原地改 `data` 指针**，不新增节点。这两处共同实现了"每个等值前缀组只缓存一条记录"。

**★ 哈希表层：`ha_search_and_get_data()` 只比较 64 位 hash_value，不比较记录内容**：

```cpp
static inline const rec_t *ha_search_and_get_data(hash_table_t *table,
                                                  uint64_t hash_value) {
  hash_assert_can_search(table, hash_value);
  ut_ad(btr_search_enabled);

  for (const ha_node_t *node = ha_chain_get_first(table, hash_value);
       node != nullptr; node = ha_chain_get_next(node)) {
    if (node->hash_value == hash_value) {
      return node->data;      // ★ 返回 rec_t*，不是 node
    }
  }
  return nullptr;
}
```

> ★★ 返回的 `rec_t*` 是**页内物理地址**，调用方立刻用 `buf_block_from_ahi(rec)` 反推 `buf_block_t`。**假命中（hash 冲突 / 前缀折叠）由上层 `btr_search_check_guess()` 兜底**——这就是"允许有瑕疵"的工程落点。

**★ 配套：`ha_remove_a_node_to_page()`（真名，⚠️ 不存在 `ha_remove_all_nodes_to_page`）**——一次只删一条：

```cpp
void ha_remove_a_node_to_page(hash_table_t *table, uint64_t hash_value,
                              const page_t *page) {
  ha_node_t *node;
  hash_assert_can_modify(table, hash_value);
  ut_ad(btr_search_enabled);

  node = ha_chain_get_first(table, hash_value);

  while (node) {
    if (page_align(ha_node_get_data(node)) == page) {
      /* Remove the hash node. It may be a node with a different fold, but we
      don't care about it - we delete any entry in this chain for the
      specified page. */
      ha_delete_hash_node(table, node);
      break;                     // ★★ 命中一条就 break
    } else {
      node = ha_chain_get_next(node);
    }
  }
}
```

> ★ 所以 `btr_search_drop_page_hash_index()` 必须**先把整页的 fold 值去重收集到 `hashes[]`**，再逐条调用——每个不同的 hash_value 各调一次。
> ★ `ha_delete_hash_node()` 用的是 `HASH_DELETE_AND_COMPACT` 宏：摘除后**把链尾节点搬到被删位置**（compact），避免第二次遍历。这也是"节点必须自带 `hash_value` 字段"的原因（搬移后要重新定位 cell）。

**★ `ha_node_t` 与内存布局**：

```cpp
struct ha_node_t {
  uint64_t hash_value;        // ★ 自带 hash_value（搬移所需）
  ha_node_t *next;
#if defined UNIV_AHI_DEBUG || defined UNIV_DEBUG
  buf_block_t *block;         // ⚠️ 仅 debug 下存在，生产构建无此字段
#endif
  const rec_t *data;          // ★ 直接指向页内记录
};
```

> ★ **release 构建 24 字节**（8+8+8）；**debug/AHI_DEBUG 下 32 字节**。源码**没有** `static_assert` 固化这个大小。一个 16KB buffer pool 帧约可放 **680 个** AHI 节点。
> ⚠️ 注意 `ha_delete_hash_node()` 里有一行 `ut_a(del_node->block->frame = page_align(del_node->data));`——**是赋值 `=` 而非比较 `==`**（源码原样，仅在 debug 下编译）。

---

### 查询路径 `btr_search_guess_on_hash`

#### 3.1 触发：8 道门禁

`btr0cur.cc`：

```c
  if (rw_lock_get_writer(btr_get_search_latch(index)) == RW_LOCK_NOT_LOCKED &&
      latch_mode <= BTR_MODIFY_LEAF && index->search_info->last_hash_succ &&
      !index->disable_ahi && !estimate
#ifdef PAGE_CUR_LE_OR_EXTENDS
      && mode != PAGE_CUR_LE_OR_EXTENDS
#endif
      && !dict_index_is_spatial(index)
      && UNIV_LIKELY(btr_search_enabled) && !modify_external &&
      btr_search_guess_on_hash(tuple, mode, latch_mode, cursor,
                               has_search_latch, mtr)) {
    btr_cur_n_sea++;
    return;
  }
  btr_cur_n_non_sea++;
```

| # | 条件 | 说明 |
|---|---|---|
| 1 | `rw_lock_get_writer(...) == RW_LOCK_NOT_LOCKED` | 分片没被 X 锁（不在建/删哈希项）→ **不等** |
| 2 | `latch_mode <= BTR_MODIFY_LEAF` | **SMO（BTR_MODIFY_TREE）不走 AHI** |
| 3 | `last_hash_succ` | 上次成功才继续尝试（自适应熔断） |
| 4 | `!index->disable_ahi` | intrinsic 临时表 / SDI 表 |
| 5 | `!estimate` | 优化器估算不走 |
| 6 | `!dict_index_is_spatial` | R-tree 不走 |
| 7 | `btr_search_enabled` | 全局开关（dirty read，函数内再确认） |
| 8 | `!modify_external` | 要改 BLOB 不走 |

#### 3.2 完整流程

```c
bool btr_search_guess_on_hash(const dtuple_t *tuple, ulint mode,
                              ulint latch_mode, btr_cur_t *cursor,
                              ulint has_search_latch, mtr_t *mtr) {
  if (!btr_search_enabled) return false;

  cursor->flag = BTR_CUR_HASH_NOT_ATTEMPTED;

  if (info->n_hash_potential == 0) return false;          // ① 索引未被认定"值得哈希"

  const auto prefix_info = info->prefix_info.load;
  cursor->ahi.prefix_info = prefix_info;

  if (dtuple_get_n_fields(tuple) < btr_search_get_n_fields(cursor))
    return false;                                          // ② 查询给的字段数不够

  const auto hash_value =
      dtuple_hash(tuple, prefix_info.n_fields, prefix_info.n_bytes,
                  btr_hash_seed_for_record(index));        // ③ 算 fold
  cursor->ahi.ahi_hash_value = hash_value;

  if (!has_search_latch) {
    if (!btr_search_s_lock_nowait(index, UT_LOCATION_HERE))
      return false;                                        // ④ nowait S 锁
  }
  ...
  rec = (rec_t *)ha_search_and_get_data(btr_get_search_table(index), hash_value);
                                                           // ⑤ 哈希查找
  cursor->flag = BTR_CUR_HASH_FAIL;                        // ⑥ 先悲观标失败
  info->last_hash_succ = false;

  if (rec == nullptr) return false;

  buf_block_t *block = buf_block_from_ahi(rec);            // ⑦ rec → block

  if (!has_search_latch) {
    if (!buf_page_get_known_nowait(latch_mode, block, Cache_hint::MAKE_YOUNG,
                                   __FILE__, __LINE__, mtr))
      return false;
    /* ★ 先拿到页 latch，再释放 AHI S-latch */
    latch_guard.rollback;
  }

  if (buf_block_get_state(block) != BUF_BLOCK_FILE_PAGE) {
    ... return false;                                      // ⑧ 页正在被淘汰
  }

  btr_cur_position(index, (rec_t *)rec, block, cursor);    // ⑨ 定位游标

  /* ⑩ ★ 三重校验 */
  if (index->space != block->page.id.space ||
      index->id != btr_page_get_index_id(block->frame) ||
      !btr_search_check_guess(cursor, has_search_latch, tuple, mode, mtr)) {
    ... return false;
  }

  if (info->n_hash_potential < BTR_SEARCH_BUILD_LIMIT + 5) {
    info->n_hash_potential++;
  }
  info->last_hash_succ = true;
  cursor->flag = BTR_CUR_HASH;                             // ⑪ 成功

  if (!has_search_latch && buf_page_peek_if_too_old(&block->page)) {
    buf_page_make_young(&block->page);                     // ⑫ 提升 LRU
  }
  Counter::inc(buf_pool->stat.m_n_page_gets, block->page.id.page_no);
  return true;
}
```

**关键设计点**：

| 步骤 | 设计意图 |
|------|---------|
| ② 字段数校验 | **"部分列查询用不上 AHI"的代码证据**——查询给的字段数必须 ≥ 前缀需要的字段数 |
| ④ nowait | 拿不到 S 锁就走 B-tree，**永不阻塞** |
| ⑥ 先标失败 | 悲观初始化，成功再改 |
| ★ 先 fix 页、后放 AHI 锁 | 顺序不能反：先拿页 latch 保证 block 不会被淘汰，之后才能安全释放 AHI 锁 |
| ⑩ 三重校验 | 因为 AHI 可能有误导项（页边界未检查 + 哈希碰撞），必须真比一次 |
| `m_n_page_gets++` | 源码注释：*"though we did not really fix the page: for user info only"*——纯粹为统计好看 |

**三重校验的最后一重** `btr_search_check_guess`（`btr0sea.cc`）有个重要限制：当调用方**只持有 AHI S-latch 但没有页 latch** 时，只能比较游标下那条记录，**不能看前后记录**，猜不中只能返回 false。

#### 3.3 命中/未命中分别做什么

| | 命中 | 未命中 |
|---|---|---|
| `cursor->flag` | `BTR_CUR_HASH` | `BTR_CUR_HASH_FAIL` 或 `BTR_CUR_HASH_NOT_ATTEMPTED` |
| 后续 | 直接 return，跳过整棵树下降 | 走完整 `btr_cur_search_to_nth_level` |
| 事后 | — | 到叶子层调 `btr_search_info_update` → 可能建 AHI / 补哈希项 |
| 计数 | `btr_cur_n_sea++` | `btr_cur_n_non_sea++` |

#### 3.4 ★ 与 `row_search_mvcc` 的关系（两条路径）

**路径 A（常规）**：`row_search_mvcc` → `btr_pcur_t::open*` → `btr_cur_search_to_nth_level` → `btr_search_guess_on_hash`。

**路径 B（特殊短路）**：`row0sel.cc` PHASE 2——**主动持 AHI S-latch 走"捷径"**：

```c
  if (UNIV_UNLIKELY(direction == 0) && unique_search && btr_search_enabled &&
      index->is_clustered && !prebuilt->templ_contains_blob &&
      !prebuilt->used_in_HANDLER &&
      (prebuilt->mysql_row_len < UNIV_PAGE_SIZE / 8) && !prebuilt->innodb_api) {
    ...
    if (trx->mysql_n_tables_locked == 0 && !prebuilt->ins_sel_stmt &&
        prebuilt->select_lock_type == LOCK_NONE &&
        trx->isolation_level > TRX_ISO_READ_UNCOMMITTED &&
        MVCC::is_view_active(trx->read_view)) {
      ut_a(!trx->has_search_latch);
      rw_lock_s_lock(btr_get_search_latch(index), UT_LOCATION_HERE);
      trx->has_search_latch = true;
      switch (row_sel_try_search_shortcut_for_mysql(...)) { ... }
      rw_lock_s_unlock(btr_get_search_latch(index));
      trx->has_search_latch = false;
    }
  }
```

**适用条件极窄**：唯一键 + 聚簇索引 + 无 BLOB + 行宽 < `UNIV_PAGE_SIZE/8` + 非 HANDLER + 一致性读（有 read view）+ 隔离级别 > RU + 无表锁 + 非 INSERT...SELECT。

> **注释里明确警告**：*"...if we try that, we can deadlock on the adaptive hash index semaphore!"* —— 这正是排除 `mysql_n_tables_locked != 0` 与 `ins_sel_stmt` 的原因。

`trx->has_search_latch` 有专门的 latch-order 校验（`btrsea_sync_check`），`i_s.innodb_trx` 里还有 `trx_adaptive_hash_latched` 列。

#### `row_sel_try_search_shortcut_for_mysql`：三态返回值

上面 `switch` 里的这个函数只有 51 行，但**返回值是三态**——这是它最容易看错的地方（`SEL_RETRY` 不是失败）：

```cpp
static ulint row_sel_try_search_shortcut_for_mysql(
    const rec_t **out_rec, row_prebuilt_t *prebuilt, ulint **offsets,
    mem_heap_t **heap, mtr_t *mtr) {
  dict_index_t *index = prebuilt->index;
  const dtuple_t *search_tuple = prebuilt->search_tuple;
  btr_pcur_t *pcur = prebuilt->pcur;
  trx_t *trx = prebuilt->trx;
  const rec_t *rec;

  // 两个断言界定了捷径的适用面（与外层调用条件呼应）
  ut_ad(index->is_clustered());
  ut_ad(!prebuilt->templ_contains_blob);

  // ★ BTR_SEARCH_LEAF 就是"走 AHI"的定位模式；已持有 search latch 就不重复加
  pcur->open_no_init(index, search_tuple, PAGE_CUR_GE, BTR_SEARCH_LEAF,
                     (trx->has_search_latch) ? RW_S_LATCH : 0, mtr,
                     UT_LOCATION_HERE);
  rec = pcur->get_rec();

  if (!page_rec_is_user_rec(rec)) {
    return (SEL_RETRY);        // 落在 infimum/supremum ⇒ 交给正常流程
  }

  /* As the cursor is now placed on a user record after a search with
  the mode PAGE_CUR_GE, the up_match field in the cursor tells how many
  fields in the user record matched to the search tuple */
  if (pcur->get_up_match() < dtuple_get_n_fields(search_tuple)) {
    return (SEL_EXHAUSTED);    // 匹配字段数不足 ⇒ 这个键根本不存在
  }

  *offsets = rec_get_offsets(rec, index, *offsets, ULINT_UNDEFINED,
                             UT_LOCATION_HERE, heap);

  if (!lock_clust_rec_cons_read_sees(rec, index, *offsets,
                                     trx_get_read_view(trx))) {
    return (SEL_RETRY);        // ★ 不可见 ⇒ 要沿 undo 回溯，捷径做不了，回退
  }

  if (rec_get_deleted_flag(rec, dict_table_is_comp(index->table))) {
    return (SEL_EXHAUSTED);    // delete-mark ⇒ 确定没有这一行
  }

  *out_rec = rec;
  return (SEL_FOUND);
}
```

三态的边界：

| 返回值 | 含义 | 主循环怎么做 |
|---|---|---|
| `SEL_FOUND` | 命中：可见且非 delete-mark | 直接 `row_sel_store_mysql_rec` 转换返回 |
| `SEL_EXHAUSTED` | **确定没有**：键不存在或已 delete-mark | `DB_RECORD_NOT_FOUND` |
| `SEL_RETRY` | **捷径做不了**：落在非用户记录，或不可见需回溯 | `break` 回退正常流程（重新 B-tree 下降 + 完整 MVCC） |

⇒ ★ `SEL_RETRY` 不是错误，是"捷径放弃"。**捷径只覆盖能一次判定的情况**（可见且未被删），任何需要 undo 回溯的都退回主流程——这也解释了为什么外层要求"有活跃 read view 且隔离级别 > RU"。

---

### 维护路径（建 / 删 / 改）

#### 4.1 建：`btr_search_build_page_hash_index`

`btr0sea.cc`。核心逻辑：

```c
  /* 1) 前缀变了先删旧的 */
  if (block->ahi.index && block->ahi.prefix_info.load != prefix_info) {
    btr_search_drop_page_hash_index(block);
  }

  auto rec = page_rec_get_next(page_get_infimum_rec(page));
  ...
  /* 2) 遍历页内记录，只在 hash_value 变化处加项 */
  for (;;) {
    const auto next_rec = page_rec_get_next(rec);
    if (page_rec_is_supremum(next_rec)) {
      if (!prefix_info.left_side) { hashes[n_cached] = hash_value; recs[n_cached] = rec; n_cached++; }
      break;
    }
    const auto next_hash_value = rec_hash(next_rec, ...);
    if (hash_value != next_hash_value) {
      if (prefix_info.left_side) { hashes[n_cached] = next_hash_value; recs[n_cached] = next_rec; }
      else                       { hashes[n_cached] = hash_value;      recs[n_cached] = rec; }
      n_cached++;
    }
    rec = next_rec; hash_value = next_hash_value;
  }

  /* 3) ★ update 决定是否等锁 */
  if (update) {
    btr_search_x_lock(index, UT_LOCATION_HERE);           // 必须成功 → 阻塞等
  } else {
    if (!btr_search_x_lock_nowait(index, UT_LOCATION_HERE)) return;  // 启发式 → 放弃
  }
  ...
  /* 4) 加锁后 double-check btr_search_enabled / prefix_info */
  /* 5) ref_count++（仅首次）；n_hash_helps = 0 */
  /* 6) 批量插入 */
  for (size_t i = 0; i < n_cached; i++) {
    ha_insert_for_hash(table, hashes[i], block, recs[i]);
  }
  MONITOR_ATOMIC_INC(MONITOR_ADAPTIVE_HASH_PAGE_ADDED);
```

**三个关键点**：

1. **★ 只存"组边界"记录**——不是每记录一条，而是**每组一条**（`left_side=true` 存新组第一条，`false` 存旧组最后一条）。这是 AHI **不是完整索引**的原因：它只能定位到某个等值前缀组，之后仍需在页内短距离比较。
2. **临界区外预计算所有 fold**——把 `hashes[]`/`recs[]` 缓存在栈上，只在最后插入时持 X latch。这是缩短持锁时间的关键优化。
3. **`update` 决定等不等锁**——启发式建索引用 nowait（失败就下次再试）；页分裂/合并后的重建**必须**阻塞等（不更新会留下错误项）。

#### 4.2 删：`btr_search_drop_page_hash_index`

`btr0sea.cc`。要点：

- **外面套 `for(;;)` 重试循环**——若在"算 fold"和"拿 X 锁"之间有人用不同 prefix 重建了 AHI，重来。
- **同样临界区外预计算 fold**。
- **用 `ha_remove_a_node_to_page(table, hash_value, page)` 而不是精确删除**——它扫描整条冲突链，删掉**所有指向该 page 的项**（因为链上可能有 fold 不同但指向同页的项）。
- **`force` 参数**只用于 `buf_LRU_free_page` 期间的 `BUF_BLOCK_REMOVE_HASH` 状态。
- 调用前不能持有任何 AHI latch（`ut_ad(!btr_search_own_any(...))`）。

**调用点**：`buf_LRU_free_page`（淘汰页）、页重组/分裂/合并（`btr0btr.cc`）、段区释放（`fsp0fsp.cc`）、压缩页重组（`page0zip.cc`）。

`btr_search_set_block_not_cached`的 assert 精确列举了 AHI 的三种合法调用上下文：

```c
  ut_ad((buf_block_get_state(block) == BUF_BLOCK_FILE_PAGE &&
         mutex_own(&btr_search_enabled_mutex) && !btr_search_enabled) ||
        (buf_block_get_state(block) == BUF_BLOCK_REMOVE_HASH &&
         !btr_search_enabled) ||
        (rw_lock_own(btr_get_search_latch(old_index), RW_LOCK_X) &&
         btr_search_enabled));
```

#### 4.3 改：DML 维护

#### INSERT

**快路径** `btr_search_update_hash_node_on_insert`——只在三个条件同时满足时：

```c
  if (cursor->flag == BTR_CUR_HASH && !prefix_info.left_side &&
      cursor->ahi.prefix_info.equals_without_left_side(prefix_info)) {
    if (!btr_search_x_lock_nowait(index, UT_LOCATION_HERE)) return;
    ...
    ha_search_and_update_if_found(table, cursor->ahi.ahi_hash_value, rec,
                                  block, page_rec_get_next(rec));
```

即：本次是通过 AHI 定位的 + AHI 存的是"组最右" + prefix 一致 → **只需改写已有节点的 `data` 指针**（新插入的记录成了新的组最右）。

**慢路径** `btr_search_update_hash_on_insert`——比较 `hash(rec)` / `hash(ins_rec)` / `hash(next_rec)`，只在**组边界变化**处插入新项。**延迟加锁**：`locked` 标志 + scope guard，只有真要插入时才 `x_lock_nowait`；整页都不需要新项时**一次锁都不加**。

#### DELETE / UPDATE

`btr_search_update_hash_on_delete`用的是**阻塞式 `btr_search_x_lock`**——因为删记录后 AHI 里会留下悬垂指针，**必须**删掉。

调用条件（`btr0cur.cc`）：

```c
    if (!index->is_clustered ||
        row_upd_changes_ord_field_binary(index, update, thr, nullptr, nullptr,
                                         nullptr)) {
      /* Remove possible hash index pointer to this record */
      btr_search_update_hash_on_delete(cursor);
    }
```

> **★ 只有改了"有序字段"才需要动 AHI**——呼应"latch 只保护记录位置"的语义。非排序字段的原地更新不影响 AHI。

#### 页分裂/合并时的迁移

`btr_search_update_hash_on_move`：若新旧 block 的 prefix 一致且推荐值也没变 → 用 `update=true` 重建新 block 的 AHI；否则**强制 drop 旧 block**。

#### 惰性修补

`btr_search_update_hash_ref`——猜错时补上正确项（见[理论基础](#三-ahi-是允许有瑕疵惰性修补的结构)）。

#### 哈希表原语的一个细节

`ha_insert_for_hash_func`（`ha0ha.cc`）：

```c
  while (prev_node != nullptr) {
    if (prev_node->hash_value == hash_value) {
      prev_node->data = data;      // ★ 同 hash_value 直接覆盖，不新增节点
      return true;
    }
    prev_node = prev_node->next;
  }
  node = (ha_node_t *)mem_heap_alloc(hash_get_heap(table), sizeof(ha_node_t));
  if (node == nullptr) {
    return false;                  // ★ 内存不够：静默失败，AHI 少一项而已
  }
```

**两处都体现"可丢弃"**：① 同一 hash_value 只保留一个；② `mem_heap_alloc` 返回 nullptr 时静默返回 false，不报错。

**★ `btr_search_update_hash_ref()` 完整实现——"惰性修补"的三重防御**：

```cpp
static void btr_search_update_hash_ref(buf_block_t *block,
                                       const btr_cur_t *cursor) {
  ut_ad(cursor->flag == BTR_CUR_HASH_FAIL);
  ...
  const auto index = block->ahi.index.load();
  const auto block_prefix_info = block->ahi.prefix_info.load();

  if (!index) return;                          // 该页不在 AHI → 无需修

  const auto info = index->search_info;
  /* Dirty read without latch, will be repeated after we take the x-latch, which
  we take after we have the hash value ready, to reduce time consumed under the
  latch. ... Note that only records folded using the block's current prefix info
  can be in AHI. */
  if (info->n_hash_potential > 0 &&
      block_prefix_info == info->prefix_info.load()) {      // ① 锁外预判
    const auto rec = btr_cur_get_rec(cursor);
    if (!page_rec_is_user_rec(rec)) return;

    const auto hash_value = rec_hash(
        rec, Rec_offsets{}.compute(rec, index), block_prefix_info.n_fields,
        block_prefix_info.n_bytes, btr_hash_seed_for_record(index), index);
    //  ★★ 用「块当前实际使用的 prefix_info」fold，与插入时保持一致

    btr_search_check_free_space_in_heap(cursor->index);

    if (!btr_search_x_lock_nowait(cursor->index, UT_LOCATION_HERE)) {
      return;                                  // ② nowait，抢不到就放弃
    }
    /* ... If it does not match, we can't add the entry to hash table, as it
    would never be deleted and would corrupt the AHI. */
    if (btr_search_enabled && block->ahi.index != nullptr) {
      ut_ad(block->ahi.index == index);
      if (info->n_hash_potential > 0 &&
          block_prefix_info == block->ahi.prefix_info.load()) {   // ③ 持锁后复核
        const auto hash_table = btr_get_search_table(index);
        ha_insert_for_hash(hash_table, hash_value, block, rec);
      }
    }
    btr_search_x_unlock(cursor->index);
  }
}
```

> ★★ **三重防御缺一不可**：① 锁外先算 hash 值（减少持锁时间）；② `x_lock_nowait` 抢不到就放弃；③ 拿到锁后**再校验** `btr_search_enabled` + `block->ahi.index != nullptr` + `block_prefix_info` 未变。
> ★★ 第 ③ 条的理由写在源码注释里：*"If it does not match, we can't add the entry to hash table, **as it would never be deleted and would corrupt the AHI**"*——用一个过期 prefix 算出的 fold，**将来删除时用新 prefix 重算会找不到它**，这条 entry 就成了永久垃圾。

**★ `btr_search_drop_page_hash_index()` 的两个精妙处**：

```cpp
void btr_search_drop_page_hash_index(buf_block_t *block, bool force) {
  for (;;) {
    const auto index = block->ahi.index.load();
    if (index == nullptr) return;
    ...
    /* 先在【锁外】把整页 fold 值算出来并去重 */
    while (!page_rec_is_supremum(rec)) {
      const auto hash_value = rec_hash(rec, offsets.compute(...),
          prefix_info.n_fields, prefix_info.n_bytes, index_hash, index);
      if (hash_value != prev_hash_value || prev_hash_value == 0) {
        hashes[n_cached] = hash_value;   // ★ 相邻相同 fold 只记一次
        n_cached++;
      }
      rec = page_rec_get_next_low(rec, page_rec_is_comp(rec));
      prev_hash_value = hash_value;
    }

    btr_search_x_lock(index, UT_LOCATION_HERE);
    ...
    {
      const auto is_ahi_enabled = btr_search_enabled.load();
      /* We need to read the block->ahi.index after we read btr_search_enabled. ... */
      const auto latched_index = block->ahi.index.load();
      if (latched_index == nullptr) return;
      if (!is_ahi_enabled) {
        if (force) { block->ahi.n_pointers.store(0); btr_search_set_block_not_cached(block); }
        /* 普通页交给 buf_pool_clear_hash_index() 处理，本线程不动 */
        return;
      }
      ut_a(latched_index == index);
    }

    if (block->ahi.prefix_info.load() == prefix_info) {
      const auto hash_table = btr_get_search_table(index);
      for (size_t i = 0; i < n_cached; i++) {
        ha_remove_a_node_to_page(hash_table, hashes[i], page);
      }
      btr_search_set_block_not_cached(block);
      MONITOR_ATOMIC_INC_VALUE(MONITOR_ADAPTIVE_HASH_ROW_REMOVED, n_cached);
      return;
    }
    /* Someone else has meanwhile built a new hash index on the page, with
    different parameters. We need to retry the process of removal. */
  }                                            // ★★ 唯一不 return 的出口：prefix 被改 → 重来
}
```

> ★★ 两点值得记：① **内存序陷阱**——必须先读 `btr_search_enabled` **再**读 `block->ahi.index`，否则会把一个已被 disable 流程清空的 block 再删一遍（源码有一段很长的注释解释）。② **`for(;;)` 唯一的"不返回"出口是 prefix_info 被并发改了**，此时用新 prefix 重算再来一遍。
> ★ `force` 参数**只有一个用途**：来自 `buf_LRU_free_page()`、block 处于 `BUF_BLOCK_REMOVE_HASH` 状态时。此时 AHI 正在被 disable，普通页交给 `buf_pool_clear_hash_index()`，只有 force 才自己清。

---

### ★ 锁协议与分片

#### 5.1 是什么锁

- **类型**：`rw_lock_t`（InnoDB 自研读写锁）
- **位置**：`btr_search_sys->parts[i].latch`
- **PFS/latch**：`LATCH_ID_BTR_SEARCH`，PFS key `btr_search_latch_key`，latch level `SYNC_SEARCH_SYS`

> **PFS 里看到的 `innodb/btr_search_latch` 是 8 个分片锁共用的一个 PFS key**，不是"还有一把全局锁"。

#### 5.2 ★ 路由：按 (space_id, index_id)，不是 page_id

```c
static inline size_t btr_search_hash_index_id(const dict_index_t *index) {
  return ut::hash_uint64_pair(index->space, index->id);
}

static inline btr_search_sys_t::search_part_t &btr_get_search_part(
    const dict_index_t *index) {
  const auto index_slot =
      btr_search_hash_index_id(index) % btr_ahi_parts_fast_modulo;
  return btr_search_sys->parts[index_slot];
}

static inline rw_lock_t *btr_get_search_latch(const dict_index_t *index) {
  return &btr_get_search_part(index).latch;
}
```

> **★ 这是 AHI 最容易被误读、也最重要的一个事实**：
> 路由键是 **索引级**的，因此**同一个索引的所有页全部落在同一把 latch 上**。
> 分片只解决了"多索引/多表之间"的争用，**解决不了单索引热点**。
> 一个热点索引上的高并发点查，在 8.0.39 里仍然全部打在同一把 latch 上。

#### 5.3 8.0 的十项优化（代码证据）

| 优化 | 位置 | 说明 |
|---|---|---|
| ① 分区 8 把锁 | `btr0sea.ic` | 按 (space, index_id) 路由 |
| ② cache line 对齐 | `btr0sea.h` | latch 与 hash_table 各占一条 |
| ③ 读路径 nowait | `btr0sea.cc` | `s_lock_nowait`，抢不到就走 B-tree |
| ④ 建索引 nowait |  | 注释："waiting here ... would defy the purpose" |
| ⑤ 插入/修补 nowait | `:629/1722/1815...` | 全是 `x_lock_nowait` |
| ⑥ 只有必须正确的路径阻塞 | (update) / (delete) | 语义要求不能出错 |
| ⑦ 临界区外预计算 fold | (build) / (drop) | 注释："to reduce time consumed under the latch" |
| ⑧ 拿页 latch 后立刻放 AHI 锁 |  | 缩短持锁窗口 |
| ⑨ 统计字段完全无锁 | `btr0sea.h` | "NOT protected by any semaphore, to save CPU time" |
| ⑩ 加锁前无锁预判 |  / `btr0cur.cc` | dirty check 大概率快速返回 |

#### 5.4 仍是瓶颈的证据

- `mysql-test/lock_order_dependencies.txt` 里 `sxlock/innodb/btr_search_latch` 出现在**上百条** ARC 中——它是 latch 层次图的高位节点，持有后几乎不能再拿别的锁。
- `buf0buf.cc`：`DEBUG_SYNC_C("purge_wait_for_btr_search_latch")`——purge 与 AHI latch 有已知交互。
- `row0sel.cc` 注释：*"we can deadlock on the adaptive hash index semaphore"*。
- **根本原因**：任何页重组/分裂/淘汰都要 X 锁它 → 写密集场景争用仍在。

---

### 开关、内存与监控

#### 6.1 内存来源

AHI 的节点（24 字节/个）来自 **buffer pool 的空闲帧**：

```c
  hash_table->heap->free_block_ptr = &free_block_for_heap;
```

即 `mem_heap` 在 buffer pool 里"借用"一个空闲块。这也是为什么 AHI 内存不必单独配置，且受 BP 大小间接约束。

**总桶数**（`buf0buf.cc`）：

```c
  btr_search_sys_create(buf_pool_get_curr_size / sizeof(void *) / 64);
```

即 `总 cell 数 ≈ BP 字节数 / 512`，平均分给 8 个 part，再经 `ut::find_prime` 取素数。例：128 GiB BP → 262144 个 cell（总）→ 每片 32768。

**★★ 但"借用空闲帧"这件事的真正原因不是省内存，是防死锁**——这是本篇最值得记住的一条工程因果链：

```cpp
static inline void btr_search_check_free_space_in_heap(dict_index_t *index) {
  if (!btr_search_enabled) return;
  ut_ad(!rw_lock_own(btr_get_search_latch(index), RW_LOCK_S));
  ut_ad(!rw_lock_own(btr_get_search_latch(index), RW_LOCK_X));   // ★★ 必须在【无锁】时调用

  auto &free_block_for_heap = btr_get_search_part(index).free_block_for_heap;
  const bool no_free_block = free_block_for_heap.load() == nullptr;

  /* We can't do this check and alloc a block from Buffer pool only when needed
  while inserting new nodes to AHI hash table, as in case the eviction is needed
  to free up a block from LRU, the AHI latches may be required to complete the
  page eviction. The execution can reach the following path: buf_block_alloc ->
  buf_LRU_get_free_block -> buf_LRU_scan_and_free_block ->
  buf_LRU_free_from_common_LRU_list -> buf_LRU_free_page ->
  btr_search_drop_page_hash_index */
  if (no_free_block) {
    const auto block = buf_block_alloc(nullptr);      // ★★ 内存来源：buffer pool 空闲帧
    buf_block_t *expected = nullptr;
    if (!free_block_for_heap.compare_exchange_strong(expected, block)) {
      buf_block_free(block);                          // 别人已备好，归还自己那份
    }
  }
}
```

> ★★ 死锁链：`ha_node_t` 从 `MEM_HEAP_FOR_BTR_SEARCH` heap 分配 → heap 不够要向 BP 申请 16KB 页 → `buf_block_alloc` 可能因 LRU 淘汰调用 `btr_search_drop_page_hash_index()` → 后者要拿 **AHI X latch** → **若此时已持 AHI X latch 就是死锁**。
> ★ 解法：**每个分片常备一个"预留帧" `free_block_for_heap`，在持锁之前先保证它非空**；持锁后的分配只从这个预留帧取，**取不到就 `return nullptr` 静默放弃**——这正是 `ha_insert_for_hash_func()` 能返回 false 的根因。

堆侧消费预留帧的地方（`mem/memory.cc:279`）：

```cpp
    if ((type & MEM_HEAP_BTR_SEARCH) && heap) {
      /* We cannot allocate the block from the buffer pool, but must get the
      free block from free block field of the heap base block. This is because
      we hold the X latch on AHI, and getting a block by eviction from LRU might
      require it too. ... */
      ut_ad(heap->free_block_ptr != nullptr);
      buf_block = static_cast<buf_block_t *>(heap->free_block_ptr->load());
      if (!buf_block) {
        return nullptr;              // ★★ 预留帧耗尽 → 返回 NULL（唯一允许失败的 heap 类型）
      }
      heap->free_block_ptr->store(nullptr);
    } else {
      buf_block = buf_block_alloc(nullptr);
    }
    block = (mem_block_t *)buf_block->frame;    // ★ AHI 节点直接写在 buffer pool 的 16KB 页帧里
```

```cpp
/** this flag can optionally be ORed to MEM_HEAP_BUFFER, in which case
 heap->free_block is used in some cases for memory allocations, and if it's
 NULL, the memory allocation functions can return NULL. */
constexpr uint32_t MEM_HEAP_BTR_SEARCH = 2;
constexpr uint32_t MEM_HEAP_FOR_BTR_SEARCH = MEM_HEAP_BTR_SEARCH | MEM_HEAP_BUFFER;  // = 3
```

> ⚠️ **没有 `buf_buddy` 参与**——AHI 内存 100% 走 `buf_block_alloc()` 拿 BP 帧，不是 buddy 分配器、也不是独立内存池。
> ⚠️ 首个 heap block（4096 字节）来自 `ut::malloc`（OS 堆），**不是** BP；只有后续扩展块才用 BP 帧。
> ⚠️ AHI **没有独立的"已用字节数"计数器**：`buf_block_t::ahi.n_pointers` 是 `uint16_t` 且**只在 debug 下存在**，release 下 `validate()` / `assert_empty()` 全是空函数。生产环境唯一的量化输出是 `SHOW ENGINE INNODB STATUS` 的 `node heap has N buffer(s)`。

**★ PFS 统计严重低估 AHI 真实占用**——`mem_key_ahi` 只覆盖两个对象：

```cpp
PSI_memory_key mem_key_ahi;
    {&mem_key_ahi, "adaptive hash index", 0, 0, PSI_DOCUMENT_ME},
```

```cpp
  btr_search_sys = ut::new_withkey<btr_search_sys_t>(
      ut::make_psi_memory_key(mem_key_ahi), hash_size);      // ① sys 对象本身
  parts = ut::make_unique_aligned<part_type[]>(
      ut::make_psi_memory_key(mem_key_ahi), alignof(part_type), btr_ahi_parts);  // ② parts 数组
```

> ★★ **不覆盖** hash table 的 cell 数组（`ut::new_<hash_table_t>(n)` 走文件默认 key），**更不覆盖** AHI 节点所在的 BP 帧（那算在 buffer pool 账上）。所以 `memory/innodb/adaptive hash index` 的数值**远小于** AHI 真实占用，不要拿它做容量判断。

**★ `hash_size` 公式的确切位置**（`buf0buf.cc:1541`，resize 时 `:2583` 复用同一式）：

```cpp
  btr_search_sys_create(buf_pool_get_curr_size() / sizeof(void *) / 64);
```

```cpp
void btr_search_sys_t::search_part_t::initialize(size_t hash_size) {
  rw_lock_create(btr_search_latch_key, &latch, LATCH_ID_BTR_SEARCH);
  hash_table = ib_create((hash_size / btr_ahi_parts), LATCH_ID_HASH_TABLE_MUTEX,
                         0, MEM_HEAP_FOR_BTR_SEARCH);    // ★ n_sync_obj = 0
  hash_table->heap->free_block_ptr = &free_block_for_heap;   // ★ 预留帧接线点
#if defined UNIV_AHI_DEBUG || defined UNIV_DEBUG
  hash_table->adaptive = true;
#endif
}
```

> ★ **`n_sync_obj = 0` 是关键**：AHI 的 hash table **不带任何内部 rw_lock**，并发保护完全交给外层 `search_part_t::latch`。这与 `buf_pool->page_hash`（`n_sync_obj > 0`，自带 rw_locks 数组）正好相反——**同为 `hash_table_t`，两种用法**。
> ⚠️ 因此 **`ha_clear()` 不适用于 AHI**：它的断言是 `table->type == HASH_TABLE_SYNC_RW_LOCK` 且 `heap == nullptr`，只对 page_hash 成立。AHI 清空走 `hash_table_clear()` + `mem_heap_empty()`。

#### 6.2 ★ 关闭 AHI 的危险代码

`btr_search_await_no_reference`（`btr0sea.cc`）：

```c
  while (index->search_info->ref_count.load != 0 &&
         (force || srv_shutdown_state.load < SRV_SHUTDOWN_CLEANUP)) {
    std::this_thread::sleep_for(std::chrono::milliseconds{10});
    sleep_counter++;

    if (sleep_counter % 500 == 0) {
      ib::error(ER_IB_LONG_AHI_DISABLE_WAIT, sleep_counter / 100,
                index->search_info->ref_count.load, index->name,
                table->name.m_name);
    }
    /* To avoid a hang here we commit suicide if the ref_count doesn't drop to
    zero in 600 seconds. */
    ut_a(sleep_counter < 60000);
  }
```

> **★ 等 600 秒 ref_count 还不归零就 `ut_a` 失败（crash）**，注释原文是 "commit suicide"。
> 相关错误码 `ER_IB_LONG_AHI_DISABLE_WAIT`。这是 AHI 最危险的一段代码，也是"线上动态关 AHI 要谨慎"的源码依据。

#### 6.3 监控

**`SHOW ENGINE INNODB STATUS`** 的 AHI 部分由相关函数输出，典型形态：

```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
Hash table size 276671, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```

**计数器**（`srv0mon.h`）：

| 计数器 | 含义 |
|--------|------|
| `MONITOR_ADAPTIVE_HASH_PAGE_ADDED` | 建页哈希次数 |
| `MONITOR_ADAPTIVE_HASH_PAGE_REMOVED` | 删页哈希次数 |
| `MONITOR_ADAPTIVE_HASH_ROW_ADDED` / `_REMOVED` | 行级增删 |
| `MONITOR_ADAPTIVE_HASH_ROW_UPDATED` | 更新已有节点 |
| `MONITOR_ADAPTIVE_HASH_ROW_REMOVE_NOT_FOUND` | 想删但没找到（说明 AHI 有瑕疵） |

**命中率**：`hash searches/s` / (`hash searches/s` + `non-hash searches/s`)。若命中率很低而非哈希搜索很高 → **AHI 是纯负担，应该关**。

---

## 相关的系统变量

| 变量 | 默认 | 范围 | 属性 |
|------|------|------|------|
| `innodb_adaptive_hash_index` | **ON** | ON/OFF | **动态**（有 update 回调） |
| `innodb_adaptive_hash_index_parts` | **8** | 1–512 | **只读**（必须重启） |

定义（`ha_innodb.cc`）：

```c
static MYSQL_SYSVAR_BOOL(
    adaptive_hash_index, srv_btr_search_enabled, PLUGIN_VAR_OPCMDARG,
    "Enable InnoDB adaptive hash index (enabled by default). "
    " Disable with --skip-innodb-adaptive-hash-index.",
    nullptr, innodb_adaptive_hash_index_update, true);

static MYSQL_SYSVAR_ULONG(
    adaptive_hash_index_parts, btr_ahi_parts,
    PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,
    "Number of InnoDB Adaptive Hash Index Partitions. (default = 8). ",
    nullptr, nullptr, 8, 1, 512, 0);
```

> `parts` 只读是必然的：`btr_ahi_parts_fast_modulo` 在 `btr_search_sys_t` 构造时一次性设定，而 AHI 在 `buf_pool_init` 末尾创建。

**★ `parts` 只读的真实机制**（⚠️ **两个回调都是 `nullptr`**，InnoDB 侧**没有任何**专用校验函数/报错文案）：

```cpp
static MYSQL_SYSVAR_ULONG(
    adaptive_hash_index_parts, btr_ahi_parts,
    PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,     // ★★ READONLY 是唯一的约束来源
    "Number of InnoDB Adaptive Hash Index Partitions. (default = 8). ",
    nullptr, nullptr,                              // ★ on_check = nullptr, on_update = nullptr
    8, 1, 512, 0);                                 // 默认 8，范围 1..512，block_size 0
```

```cpp
      : sys_var(
            chain, name_arg, plugin_var_arg->comment,
            (plugin_var_arg->flags & PLUGIN_VAR_THDLOCAL ? SESSION : GLOBAL) |
                (plugin_var_arg->flags & PLUGIN_VAR_READONLY ? READONLY : 0) |
                ...
```

> ★★ 所以"只能在启动时改"是 **server 通用层的 `sys_var::READONLY`** 拦的，`SET GLOBAL` 会被通用层拒绝，InnoDB 侧并不参与。初始化后才真正写入 fast_modulo：
> ```cpp
> btr_search_sys_t::btr_search_sys_t(size_t hash_size) {
>   parts = ut::make_unique_aligned<part_type[]>(
>       ut::make_psi_memory_key(mem_key_ahi), alignof(part_type), btr_ahi_parts);
>   static_assert(alignof(part_type) >= ut::INNODB_CACHE_LINE_SIZE);   // ★ 每 part 独占 cache line
>   btr_ahi_parts_fast_modulo = ut::fast_modulo_t{btr_ahi_parts};       // ★ 只在构造时写一次
>   for (ulint i = 0; i < btr_ahi_parts; ++i) parts[i].initialize(hash_size);
> }
> ```

**★ 开关变量的真名**（⚠️ **不存在 `srv_innodb_adaptive_hash_index`**，全仓 0 命中）：

```cpp
/** Flag storing if the search system is in enabled state. While it is false,
the AHI data structures can't have new entries added, they can only be
removed. It is changed to false while having all AHI latches X-latched, so any
section that adds entries to AHI data structures must have at least one S-latch. */
std::atomic_bool btr_search_enabled = true;      // ★ 真正的原子开关（热路径无锁读）

/** A value that basically stores the same as btr_search_enabled, but is not
atomic and thus can be used as SYSVAR. */
bool srv_btr_search_enabled = true;              // ★ sysvar 后端存储（sysvar 不能绑 atomic）

/** Protects changes of btr_search_enabled flag. */
static ib_mutex_t btr_search_enabled_mutex;
```

> ★★ **为什么要有两份**：sysvar 的后端变量**不能**是 `std::atomic`，所以必须再存一个非原子副本。两者在 `enable()` / `disable()` 里**成对同步更新**；且**启动时 sysvar 的 on_update 不会被调用**，所以 `btr_search_sys_create()` 里有一行手工同步：`btr_search_enabled = srv_btr_search_enabled;`。

```cpp
static void innodb_adaptive_hash_index_update(THD *, SYS_VAR *, void *,
                                              const void *save) {
  if (*(bool *)save) { btr_search_enable(); } else { btr_search_disable(); }
}
```

**★ `btr_search_disable()` 的五步**（返回 `bool`：原本 enabled 且被本次关闭才 true，供 BP resize 后决定是否恢复）：

```cpp
bool btr_search_disable() {
  mutex_enter(&btr_search_enabled_mutex);
  if (!btr_search_enabled) { mutex_exit(...); return false; }

  btr_search_x_lock_all(UT_LOCATION_HERE);       // ② X 锁全部 AHI 分区
  ut_a(btr_search_enabled);
  btr_search_enabled = false;                    // ③ 在持全 X 锁时改 → 此后只许删不许加
  srv_btr_search_enabled = false;
  btr_search_x_unlock_all();

  buf_pool_clear_hash_index();                   // ④ 清空所有 block 的 ahi.index

  dict_sys_mutex_enter();
  for (auto table : dict_sys->table_LRU)     btr_search_await_no_reference(table);
  for (auto table : dict_sys->table_non_LRU) btr_search_await_no_reference(table);
  dict_sys_mutex_exit();                         // ⑤ 等所有索引 ref_count 归零（含 600s 自杀）

  for (ulint i = 0; i < btr_ahi_parts; ++i) {
    const auto hash_table = btr_search_sys->parts[i].hash_table;
    hash_table_clear(hash_table);
    mem_heap_empty(hash_table->heap);
  }
  mutex_exit(&btr_search_enabled_mutex);
  return true;
}
```

> ⚠️ **`btr_search_enable()` 不一定能开**：若正在 BP resize（`srv_buf_pool_old_size != srv_buf_pool_size`）**静默 return，不报错**。
> ⚠️ `btr_search_sys_resize()`（真名，确实存在）**前提必须是 AHI 已 disabled**，否则 release 下打一条 error 后直接 return。

**★★ 监控出口只有两个，且都没有 status 变量**（⚠️ **不存在任何 `Innodb_adaptive_hash_*` status 变量**，全仓 0 命中——5.6/5.7 时代的 `adaptive_hash_hash_searches` 等已被移除）：

```cpp
  for (ulint i = 0; i < btr_ahi_parts; ++i) {          // ① SHOW ENGINE INNODB STATUS
    auto &part = btr_search_sys->parts[i];
    rw_lock_s_lock(&part.latch, UT_LOCATION_HERE);
    ha_print_info(file, part.hash_table);
    rw_lock_s_unlock(&part.latch);
  }
  fprintf(file, "%.2f hash searches/s, %.2f non-hash searches/s\n",
          (btr_cur_n_sea - btr_cur_n_sea_old) / time_elapsed,
          (btr_cur_n_non_sea - btr_cur_n_non_sea_old) / time_elapsed);
```

```cpp
  fprintf(file, "Hash table size %zu", hash_get_n_cells(table));
  ...
  n_bufs = UT_LIST_GET_LEN(table->heap->base) - 1;      // 首块是 ut::malloc 的常规块
  if (table->heap->free_block_ptr->load()) n_bufs++;
  fprintf(file, ", node heap has %zu buffer(s)\n", n_bufs);
```

> ★★ **`node heap has N buffer(s)` 是 AHI 内存占用的唯一对外量化**：`N ≈ (已挂块数 - 1) + 预留帧`，每块 16KB ⇒ **AHI 每 part ≈ (N-1) × 16KB**。
> ⚠️ `used cells` 只在 `UNIV_DEBUG` 下打印（生产构建为性能裁掉了）。
> ⚠️ `time_elapsed` 用 `steady_clock` **截断到秒** + 0.001 防除零——**两次 SHOW ENGINE 间隔 < 1 秒时速率会被放大约 1000 倍**。

```cpp
    {"adaptive_hash_searches", "adaptive_hash_index", ..., MONITOR_OVLD_ADAPTIVE_HASH_SEARCH},
    {"adaptive_hash_searches_btree", "adaptive_hash_index", ..., MONITOR_OVLD_ADAPTIVE_HASH_SEARCH_BTREE},
    {"adaptive_hash_pages_added", ..., MONITOR_ADAPTIVE_HASH_PAGE_ADDED},
    {"adaptive_hash_pages_removed", ..., MONITOR_ADAPTIVE_HASH_PAGE_REMOVED},
    {"adaptive_hash_rows_added", ..., MONITOR_ADAPTIVE_HASH_ROW_ADDED},
    {"adaptive_hash_rows_removed", ..., MONITOR_ADAPTIVE_HASH_ROW_REMOVED},
    {"adaptive_hash_rows_deleted_no_hash_entry", ..., MONITOR_ADAPTIVE_HASH_ROW_REMOVE_NOT_FOUND},
    {"adaptive_hash_rows_updated", ..., MONITOR_ADAPTIVE_HASH_ROW_UPDATED},
```

```cpp
    case MONITOR_OVLD_ADAPTIVE_HASH_SEARCH:       value = btr_cur_n_sea;      break;
    case MONITOR_OVLD_ADAPTIVE_HASH_SEARCH_BTREE: value = btr_cur_n_non_sea;  break;
```

> ★ 这 8 个计数器经 `I_S.INNODB_METRICS` 暴露（**`i_s.cc` 里无 AHI 专属代码**，靠 srv0mon 通用框架自动挂载）。四个 `btr_cur_n_*` 都是**普通 `ulint` 非原子**，多线程递增存在竞争，只作近似值。

---

## Misc

### 社区边界澄清

| 常被问到 | 8.0.39 实际情况 |
|---------|----------------|
| **`btr_search_fold` / `btr_search_get_n_bytes`** | **不存在**（5.6/5.7 名字），8.0 是 `rec_hash` / `dtuple_hash` |
| **`buf_block_t::search_index` / `n_fields` / `curr_*`** | **不存在**，改为 `ahi_t` + `btr_search_prefix_info_t` |
| **全局 `btr_search_latch`** | **不存在**（8.0.30 已分片）；PFS 里那个名字是 8 个分片共用的 key |
| **`Double_write::recover` 式的 `AHI recover`** | 不存在，AHI 不持久化 |

**★ 真名勘误全表**（经全仓 grep 验证，写文档时勿用左列）：

| 常见写法 | 8.0.39 真相 |
|---|---|
| `btr_search_info_t` | **不存在**。真名 `struct btr_search_t`；只有函数 `btr_search_info_create()` 保留了旧名 |
| `n_fields` / `n_bytes` / `left_side` 是独立字段 | **错**。已打包进 `std::atomic<btr_search_prefix_info_t> prefix_info` |
| `btr_search_prefix_info_t` 在 `btr0sea.h` | **不在**。真身在 **`buf0buf.h:1666`** |
| `btr_search_info->n_patt_move` | **不存在** |
| 全局 `btr_search_hash_table` 指针 | **不存在**。现在是 `btr_search_sys->parts[i].hash_table`，由 `btr_get_search_table()` 取（同名静态函数 `btr_search_hash_table_validate()` 存在但无关） |
| `btr_search_index_num()` / `btr_search_get_index_num()` | **不存在**。真名 `btr_search_hash_index_id()` + `btr_get_search_part()` + `btr_get_search_latch()` |
| `ha_remove_all_nodes_to_page()` | **不存在**。真名 `ha_remove_a_node_to_page()`（一次只删一条） |
| `btr_search_latch_own()` | **不存在**。对应物 `btr_get_search_latch()` + `rw_lock_own()`；只有 debug 下的 `btr_search_own_all/own_any` |
| `btr_search_info_n_hash_orderings` | **不存在**。真名 `btr_search_info_update_hash()`（前缀学习） |
| `btr_search_lazy_changed` | **不存在**（全仓 0 命中）。惰性修正由 `btr_search_update_hash_ref()` 承担 |
| `btr_search_get_hash_index()` | **不存在**。真名 `btr_search_hash_index_id()` |
| `rec_fold()` / `dtuple_fold()` | **不存在**（0 命中）。8.0 一律用 `rec_hash` / `dtuple_hash` |
| `ut_fold_binary()` | **不存在**。真名 `ut::hash_binary()` |
| `rec_hash()` / `dtuple_hash()` 定义在 `.h` | 头文件**只有声明**；定义体在 `rem0rec.ic:836` / `data0data.ic:596` |
| `buf_block_t::ahi.n_recs` | **不存在**。`ahi_t` 只有 `recommended_prefix_info` / `prefix_info` / `index` / `n_pointers`（最后一项仅 debug） |
| 查询路径检查 `block->buf_fix_count` | **不检查**。改用 `buf_page_get_known_nowait()`；`buf_fix_count` 只在 `btr_drop_next_batch()` 出现 |
| `srv_innodb_adaptive_hash_index` | **不存在**（0 命中）。真名 `srv_btr_search_enabled` + `btr_search_enabled` |
| `btr_search_sys_t::get_hash_node_count()` | **不存在**（0 命中） |
| `~btr_search_sys_t()` | **不存在**（类内只有构造函数）。释放由 `btr_search_sys_free()` 手工完成 |
| `Innodb_adaptive_hash_*` status 变量 | **不存在**（0 命中）。只有 SHOW ENGINE 文本 + `I_S.INNODB_METRICS` 计数器 |
| `innodb_adaptive_hash_index_parts` 有 on_check/on_update | **都是 `nullptr`**。约束靠 `PLUGIN_VAR_READONLY` → `sys_var::READONLY` |
| AHI 内存走 `buf_buddy_alloc` | **否**。走 `buf_block_alloc()` 取 BP 空闲帧 |
| `btr_search_disable()` 返回 void | 返回 **`bool`**（原 enabled 且被本次关闭才 true），供 BP resize 后决定是否恢复 |

**★ 三个"结构性事实"**：

1. **AHI 的 `hash_table_t` 用 `n_sync_obj = 0` 创建**——哈希表本身**不带任何内部锁**，保护完全由 `search_part_t::latch` 承担。而 `buf_pool->page_hash` 是 `n_sync_obj > 0`（自带 rw_locks）。**同一个 `hash_table_t`，两种截然不同的用法**。因此 **`ha_clear()` 不适用于 AHI**（它的断言要求 `type == HASH_TABLE_SYNC_RW_LOCK` 且 `heap == nullptr`）。
2. **`btr_search_sys` 故意用裸指针而非 `unique_ptr`**——源码注释：*"destroying it at exit() will destroy its rw_lock after the PFS is deinitialized"*（静态析构晚于 PFS 反初始化会崩）。
3. **`btr_search_guess_on_hash` 全仓只有一个调用点**（`btr0cur.cc:862`）。`row_search_mvcc()` 不直接调它——而是提前拿 AHI 分区 S 锁 → 置 `trx->has_search_latch` → 走 `pcur->open_no_init(..., RW_S_LATCH, ...)`，最终由 `btr_cur_search_to_nth_level` 内部的 AHI 分支自然命中。这是"唯一键等值 + 一致性读"的快捷路径，有 **6 个前置条件**（非顺序扫描 / unique_search / AHI 开 / 聚簇索引 / 无 blob 模板 / `mysql_row_len < UNIV_PAGE_SIZE/8`）。

### 常见误解

| 误解 | 正确 |
|------|------|
| "AHI 是磁盘上的索引" | **不是**。纯内存，`rec_t*` 指针，实例重启即消失 |
| "AHI 每条记录一个哈希项" | **不是**。**每个"相等前缀组"只有一条**（`left_side` 决定最左/最右） |
| "AHI 对所有查询都有用" | **不是**。只对**等值点查**有用；范围扫描、全表扫描无用 |
| "给了部分列也能用 AHI" | **不能**。`dtuple_get_n_fields(tuple) < n_fields` 直接返回 false |
| "AHI 分片后就没有争用了" | **不对**。路由按 (space_id, index_id)，**同一索引的所有页仍在同一把 latch** |
| "AHI 自动管理不需要关注" | 需要。**命中率低时它是纯负担**（维护成本 + latch 争用） |
| "动态关 AHI 是安全操作" | **有风险**。要等 `ref_count` 归零，600 秒不归零会 crash |
| "UPDATE 改任何字段都要维护 AHI" | **不是**。只有改**有序字段**才需要（AHI 只保护记录位置） |
| "AHI 内存要单独配置" | 不需要。节点内存来自 buffer pool 空闲帧 |
| "AHI 用在主键上吗" | 会（聚簇索引也可以建 AHI），但**主键点查本来就走聚簇索引，AHI 收益相对小** |

### 什么时候该关 AHI

| 信号 | 判断 |
|------|------|
| `hash searches/s` 占比很低（如 < 20%） | **关**——纯负担 |
| 负载以范围扫描 / 全表扫描为主 | **关** |
| 写密集（大量 INSERT/UPDATE/DELETE） | 考虑关（维护成本 + X 锁争用） |
| 出现 `btr_search_latch` 等待 / `ER_IB_LONG_AHI_DISABLE_WAIT` | 关（但要做好重启准备） |
| 大表 DDL / 批量导入期间 | 临时关 |

---

## 参考

> 本文结论基于 MySQL 8.0.39 源码逐行核实；涉及的函数与类型可直接在本仓库检索。

**相关文档**

- B-tree 游标搜索与页面结构：[`btr.md`](btr.md)
- Buffer Pool 读页与预读：[`buffer_pool.md`](../innodb/buffer_pool.md)
- change buffer：[`ibuf.md`](ibuf.md)
