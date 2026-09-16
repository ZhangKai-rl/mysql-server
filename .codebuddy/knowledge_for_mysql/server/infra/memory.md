# MySQL 内存分配器矩阵：mem_heap / MEM_ROOT / buf_buddy / jemalloc

> 本篇聚焦 **分层内存分配器矩阵**。PFS 可观测性见 [pfs.md](pfs.md)；读源码通用知识地图与共享术语/论文见 [README.md](README.md)。
> 基于 MySQL 8.0 源码（`storage/innobase`、`mysys` 等模块）。

## 目录

- [摘要](#摘要)
- [一、InnoDB 内存分配器矩阵](#一innodb-内存分配器矩阵)
- [二、可观测性与内存管理的交汇](#二可观测性与内存管理的交汇)
- [三、内存管理模块涉及的设计模式](#三内存管理模块涉及的设计模式)
- [四、核心术语中英文对照（内存管理）](#四核心术语中英文对照内存管理)
- [五、学术论文索引（内存分配器）](#五学术论文索引内存分配器)
- [六、设计哲学（内存部分）](#六设计哲学内存部分)
- [附录：推荐阅读文件清单（内存部分）](#附录推荐阅读文件清单内存部分)

---

## 摘要

MySQL 的**分层内存分配器矩阵**（mem_heap、MEM_ROOT、buf_buddy、jemalloc）管理从临时查询结构到 Buffer Pool 页面的所有内存：mem_heap 是 InnoDB 的 Bump Allocator（指针碰撞分配器，O(1) 极速）、MEM_ROOT 是 Server 层的 Arena Allocator（块级回收）、buf_buddy 是二叉伙伴分配器（管理压缩页 frame）、底层由 jemalloc 支撑多核可扩展性。背后是 Buddy Allocator、Arena/Bump Allocator、Segregated-fit Allocator 等理论在数据库内存管理中的具体落地。

---
## 一、InnoDB 内存分配器矩阵

### 2.1 全景架构

InnoDB 不是用一个通用分配器管理所有内存，而是根据生命周期、大小和底层来源，构建了一个 **分层协作的分配器矩阵**：

```
┌──────────────────────────────────────────────────────┐
│  事务生命周期 / 临时计算                               │
│  ┌──────────────────────────────────────────────┐    │
│  │  mem_heap (InnoDB 引擎内部)                    │    │
│  │  类型: Bump/Arena Allocator                   │    │
│  │  来源: 小块→ut::malloc, 大块→Buffer Pool 页面  │    │
│  │  特色: MEM_HEAP_BTR_SEARCH 死锁规避            │    │
│  │        MEM_NO_MANS_LAND Debug 越界检测         │    │
│  │        Scoped_heap RAII 封装                  │    │
│  └──────────────────────────────────────────────┘    │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │  MEM_ROOT (Server 层)                          │    │
│  │  类型: Arena Allocator                        │    │
│  │  来源: my_malloc                              │    │
│  │  特色: 指数增长 block size                     │    │
│  │        Mem_root_allocator STL 适配            │    │
│  └──────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────┤
│  Buffer Pool 内部分配                                 │
│  ┌──────────────────────────────────────────────┐    │
│  │  buf_buddy                                    │    │
│  │  类型: Binary Buddy Allocator                 │    │
│  │  来源: Buffer Pool 预分配页面（BUF_BLOCK_MEMORY）│    │
│  │  理论: Knowlton 1965 (CACM), Knuth 1968       │    │
│  │  场景: 压缩页 frame 分配                       │    │
│  └──────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────┤
│  OS 堆分配（最终所有上层都落到这里）                   │
│  ┌──────────────────────────────────────────────┐    │
│  │  ut::malloc_withkey (PFS 埋点包装)             │    │
│  │  类型: Instrumented Wrapper（无新分配算法）    │    │
│  │  作用: 每次分配向 PFS 报告 key + size           │    │
│  └──────────────────────┬───────────────────────┘    │
│  ┌──────────────────────┴───────────────────────┐    │
│  │  jemalloc / glibc malloc (OS 分配器)          │    │
│  │  类型: Segregated-fit Allocator               │    │
│  │  思想: size class + thread cache + arena      │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### 2.1.1 主链路：一次分配落到哪个分配器

上面的全景图是**静态分层**，主链路是**一次 `mem_heap_alloc` 沿哪条路径落到最底层**：

```
mem_heap_alloc(heap, n)                       ← bump 分配入口
  │
  ├─ 当前 block 剩余空间够？
  │   ├─ 够 → 直接 bump（O(1) 推 free 指针）
  │   │       buf = block + free + MEM_NO_MANS_LAND
  │   │       mem_block_set_free(free + MEM_SPACE_NEEDED(n))
  │   │
  │   └─ 不够 → mem_heap_add_block(heap, n)
  │             │  新块大小 = 2×上一块，封顶标准值
  │             ▼
  │         mem_heap_create_block(heap, size, type)
  │             │
  │             ├─ type==DYNAMIC 或 size < 半页(8KB)
  │             │     → ut::malloc_withkey(KEY, len)
  │             │         → jemalloc / glibc malloc → PFS 统计
  │             │
  │             └─ 大块（BUFFER 类型）:
  │                 ├─ type & BTR_SEARCH
  │                 │     → free_block_ptr 原子取（AHI 死锁规避，失败返 NULL）
  │                 └─ 否则
  │                       → buf_block_alloc
  │                           → Buffer Pool free list（一整个 16KB 页）
  ▼
返回 buf
```

**这条链路的三个关键分叉**：

- **bump vs add_block**：绝大多数分配走 bump 分支（O(1)，推 free 指针）；只有当前 block 装不下才 `add_block`（低频）
- **小块 vs 大块**（`create_block` 内）：`< 半页(8KB)` 走 `malloc`，因为不值得为几 KB 临时数据浪费一整个 16KB BP 页
- **BTR_SEARCH 特殊路径**：持有 AHI X-latch 时不能从 BP 分配（可能触发 LRU 驱逐 → 又要 AHI latch → 死锁），所以改成从预留的 `free_block_ptr` 原子取，取不到返回 NULL 而非崩溃

> 这条链路把 2.2~2.5 四个分配器串成一条线：`mem_heap_alloc` 是统一入口，`create_block` 是分流点，最终落到 `jemalloc/glibc`（malloc 路径）或 `Buffer Pool`（页面路径）。

### 2.2 mem_heap —— InnoDB 的 Bump Allocator（指针碰撞分配器）

#### 2.2.1 核心思想与数据结构

mem_heap 是 InnoDB 内部使用最广泛的临时内存分配器。本质是 **Bump Allocator（指针碰撞分配器）**——维护一个 `free` 偏移量指针，分配时直接向前推进，O(1) 完成。

```cpp
// mem_heap_alloc 的核心逻辑（mem0mem.ic:162-165）
buf = (byte *)block + free + MEM_NO_MANS_LAND;           // 计算返回地址
mem_block_set_free(block, free + MEM_SPACE_NEEDED(n));   // 推进 free 指针
```

数据结构：heap 是一个 `mem_block_t` 的非空线性链表：

```
mem_heap_t (heap)
  │
  ├── block 1（也是 heap 的根节点）
  │     ├── mem_block_info_t 头部（含 magic_n, type, len, free, start 等）
  │     ├── [0xCE×16] alloc_1 用户数据 [0xDF×16] + (对齐填充)
  │     ├── [0xCE×16] alloc_2 用户数据 [0xDF×16] + (对齐填充)
  │     ├── [0xCE×16] alloc_3 用户数据 [0xDF×16] + (对齐填充)
  │     └── ...剩余空闲空间...
  │
  └── block 2（不够用了，追加的）
        └── 同上结构
```

**理论来源：**Bump Allocator 是最古老、最基本的分配器策略，无特定论文——它从 1960s 编译器实现中自然产生，在 Knuth 的 *The Art of Computer Programming* Vol.1 (1968) 中有形式化描述。它的设计哲学是**以放弃单块释放的能力，换取极致的分配速度和零碎片**。

#### 2.2.2 三种 Heap Type —— 核心设计

mem_heap 最独特的设计是三种类型决定了**内存来源**和**分配失败行为**的分叉：

```cpp
// mem0mem.h
constexpr uint32_t MEM_HEAP_DYNAMIC    = 0;   // 走 ut::malloc
constexpr uint32_t MEM_HEAP_BUFFER     = 1;   // 大块走 Buffer Pool
constexpr uint32_t MEM_HEAP_BTR_SEARCH = 2;   // 标志位：用于 AHI 死锁规避

// 组合体：
constexpr uint32_t MEM_HEAP_FOR_BTR_SEARCH = MEM_HEAP_BTR_SEARCH | MEM_HEAP_BUFFER; // = 3
constexpr uint32_t MEM_HEAP_FOR_PAGE_HASH  = MEM_HEAP_DYNAMIC;                       // = 0
constexpr uint32_t MEM_HEAP_FOR_RECV_SYS   = MEM_HEAP_BUFFER;                        // = 1
```

**分支逻辑在 `mem_heap_create_block`（memory.cc:248-350）中：**

```cpp
mem_block_t *mem_heap_create_block(mem_heap_t *heap, ulint n, ulint type) {
  len = MEM_BLOCK_HEADER_SIZE + MEM_SPACE_NEEDED(n);

  // 关键分支：
  if (type == MEM_HEAP_DYNAMIC || len < UNIV_PAGE_SIZE / 2) {
    // 路径 A：走 ut::malloc
    // 条件1：DYNAMIC 类型无论多大都走 malloc
    // 条件2：即使 BUFFER 类型，小于半页(~8KB)也不值得浪费 BP 页面
    block = ut::malloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, len);
  } else {
    // 路径 B：从 Buffer Pool 分配一个完整页面
    len = UNIV_PAGE_SIZE;

    if ((type & MEM_HEAP_BTR_SEARCH) && heap) {
      // 子路径 B1：AHI 场景——从预留的 free_block 取
      buf_block = heap->free_block_ptr->load();
      if (!buf_block) return nullptr;  // 优雅失败！
      heap->free_block_ptr->store(nullptr);
    } else {
      // 子路径 B2：正常从 BP free list 取
      buf_block = buf_block_alloc(nullptr);
    }

    block = (mem_block_t *)buf_block->frame;
  }
}
```

**完整分配策略表：**

| Type | 值 | 小块 (<8KB) | 大块 (≥8KB) | 失败时 | 典型场景 |
|------|-----|------------|------------|--------|---------|
| `MEM_HEAP_DYNAMIC` | 0 | `ut::malloc` | `ut::malloc` | fatal 崩溃 | 通用临时 heap、DDL |
| `MEM_HEAP_BUFFER` | 1 | `ut::malloc` | BP 页面 (`buf_block_alloc`) | fatal 崩溃 | recovery、lock heap |
| `MEM_HEAP_FOR_BTR_SEARCH` | 3 | `ut::malloc` | 从 `free_block_ptr` 原子取 | **返回 NULL** | AHI hash table |

**为什么小块即使 type 是 BUFFER 也走 malloc？** 不值得为几 KB 的临时分配浪费一整个 16KB Buffer Pool 页面——那可是一个数据页缓存的代价。

#### 2.2.3 MEM_HEAP_BTR_SEARCH —— 死锁规避的精妙设计

这是 InnoDB 中极具工程智慧的设计。AHI（Adaptive Hash Index，自适应哈希索引）在插入新条目时需要持有 AHI latch，而扩展 heap 又可能需要从 Buffer Pool 分配页面。如果 BP 满了需要 LRU 驱逐，驱逐调用链是：

```
Thread A 持有 AHI X-latch
  → mem_heap_alloc 需要更多空间
  → mem_heap_create_block
  → buf_block_alloc  ← 从 BP 分配
  → 但 BP 满了！
  → buf_LRU_get_free_block
  → buf_LRU_scan_and_free_block
  → buf_LRU_free_page
  → btr_search_drop_page_hash_index  ← 需要 AHI latch！
  → 自己等自己 → 死锁 💀
```

**解法：预留在先（InnoDB 三步法）**

1. **预留在先（获取 AHI latch 之前）**：调用 `btr_search_check_free_space_in_heap()`，检查 `free_block_for_heap` 原子变量是否为空，为空就从 BP 分配一个页面存进去。此时还没持有 AHI latch，不会有死锁。
2. **用时不分配（持有 latch 后）**：`mem_heap_create_block` 发现 `type & MEM_HEAP_BTR_SEARCH` 且 `heap` 不为空时，不从 BP 新分配，而是从 `heap->free_block_ptr` 原子取走预留页面。如果取不到，**返回 NULL 而非崩溃**，调用方优雅处理。
3. **用后补充**：释放 AHI latch 后，下次操作前会再次调用 `btr_search_check_free_space_in_heap` 重新补充。

```cpp
// btr0sea.cc:155-184
static inline void btr_search_check_free_space_in_heap(dict_index_t *index) {
  auto &free_block_for_heap = btr_get_search_part(index).free_block_for_heap;
  if (free_block_for_heap.load() == nullptr) {
    const auto block = buf_block_alloc(nullptr);  // 还没拿 latch，安全
    buf_block_t *expected = nullptr;
    if (!free_block_for_heap.compare_exchange_strong(expected, block)) {
      buf_block_free(block);  // 并发 race：别人已经设了，归还这个
    }
  }
}
```

这个设计确保了：AHI latch 持有者永远不会为了获取 latch 而去拿自己已经持有的 latch——经典的锁顺序死锁规避。

#### 2.2.4 Debug 机制：No Man's Land（金丝雀字节 / 哨兵字节）

仅在 Debug 模式（`UNIV_DEBUG`）下启用。每次 `mem_heap_alloc` 在返回给用户的数据前后各放置 16 字节的哨兵（canary bytes）：

```
┌──────────────┬─────────────────┬──────────────┬──────────┐
│  0xCE × 16   │  用户数据 (N 字节)  │  0xDF × 16   │ (对齐填充) │
│  前方哨兵区   │                  │  后方哨兵区   │          │
└──────────────┴─────────────────┴──────────────┴──────────┘
```

```cpp
// mem0mem.h:99-108
#ifdef UNIV_DEBUG
  constexpr int MEM_NO_MANS_LAND = 16;
#else
  constexpr int MEM_NO_MANS_LAND = 0;    // Release 下完全消除
#endif

const byte MEM_NO_MANS_LAND_BEFORE_BYTE = 0xCE;  // 前方填充字节
const byte MEM_NO_MANS_LAND_AFTER_BYTE  = 0xDF;  // 后方填充字节
```

释放时 `validate_no_mans_land()` 逐字节校验：

```cpp
// mem0mem.ic:189-198
static inline void validate_no_mans_land(
    byte *no_mans_land_begin, byte mem_no_mans_land_byte) {
  for (byte *it = no_mans_land_begin;
       it < no_mans_land_begin + MEM_NO_MANS_LAND; ++it) {
    ut_a(*it == mem_no_mans_land_byte);  // 被篡改了？直接 assert
  }
}
```

**这是经典的 canary byte / guard byte / fence 机制**，是发现 buffer overflow 和 underflow 的第一道防线。

**业界类似实现：**

| 实现 | 来源 | 值 | 说明 |
|------|------|-----|------|
| **mem_heap** | InnoDB | `0xCE` / `0xDF` × 16 | 前后哨兵，Debug-only |
| **Windows Debug Heap** | Microsoft CRT | `0xFD` | 分配块前后 guard |
| **Red zone** | AddressSanitizer | 不可访问页 | 运行时检测越界 |
| **Electric Fence** | Bruce Perens | 保护页 | 经典的 malloc 调试库 |

**为什么选 0xCE 和 0xDF？** 这两个值没有特殊数学含义。选择原则：非零（避免漏检零值写入）、前后不同（便于区分方向）、在 hex dump 中连续出现时非常显眼——正常数据不可能连续 16 字节完全相同。

#### 2.2.5 Scoped_heap —— RAII 封装

`Scoped_heap` 将 C 风格的 `mem_heap_t` 封装为 RAII 对象，底层用 `std::unique_ptr` + 自定义 deleter 实现自动释放：

```cpp
// mem0mem.h:440-507
struct Scoped_heap {
  struct mem_heap_free_functor {
    void operator()(mem_heap_t *heap) { mem_heap_free(heap); }
  };
  using Ptr = std::unique_ptr<mem_heap_t, mem_heap_free_functor>;
  Ptr m_ptr{};

  // 禁止拷贝和移动（所有权唯一）
  Scoped_heap(Scoped_heap &&) = delete;
  Scoped_heap(const Scoped_heap &) = delete;

  // 构造即创建 MEM_HEAP_DYNAMIC 类型的 heap
  Scoped_heap(size_t n, ut::Location location) noexcept
      : m_ptr(mem_heap_create(n, location, MEM_HEAP_DYNAMIC)) {}
};
```

使用者：`ddl0impl-builder.h`、`ddl0impl-cursor.h`、`lock0lock.cc`、`btr0cur.cc` 等 C++ 代码中需要临时 heap 的地方。

### 2.2.6 跨分配器内存复用能力分析

一个常见的误解是"bump allocator 完全没有内存复用"。准确地说：bump allocator **放弃了任意位置的单块复用能力**，但保留了**粗粒度的快照级复用**。

#### 各分配器复用能力总览

| 分配器 | 单块级复用 | block级复用 | 复用机制 | 设计哲学 |
|--------|-----------|------------|---------|---------|
| **buf_buddy** | ✅ 完整 | ✅ 完整 | free list + buddy 合并（`zip_free[]`） | 长生命周期、按需归还 |
| **jemalloc** | ✅ 完整 | ✅ 完整 | thread cache (tcache) + arena → bin → slab | 通用 malloc，高性能 |
| **mem_heap** | ❌（仅栈顶 `free_top`） | ✅（`mem_heap_empty`） | bump allocator，无 free list | 短生命周期临时分配，用完整体释放 |
| **MEM_ROOT** | ❌ | ✅（`ClearForReuse`） | 同 bump，但 block 可保留复用 | 一次查询的临时内存 |

#### mem_heap 的三种复用粒度

```
粒度1：mem_heap_free_top —— 栈顶单块回退
  适用场景：迭代构建最终结果
  例：row_merge_buf_add() 先分配缓冲区，失败时 free_top 回退
  → free 指针回退，下次分配 "覆盖" 之前的数据（栈式复用）

粒度2：mem_heap_empty —— 整块重置
  适用场景：多次执行同一操作
  例：lock_rec_other_has_expl_req() 在遍历事务时重置临时 heap
  → free 指针回到 start，复用整个 block 的空间

粒度3：mem_heap_free —— 完全销毁
  适用场景：操作结束，不再需要
  → 整个 heap 归还，没有任何复用
```

#### 为什么 bump allocator 不做细粒度复用？

| 如果要做 | 需要付出的代价 | 为什么 bump 不做 |
|---------|-------------|-----------------|
| 记录每个空闲位置 | free list（链表管理开销） | 违背了"两条指令完成分配"的极简目标 |
| 合并相邻空闲块 | 合并逻辑（CPU 开销） | 对 microsecond 级的临时操作不可接受 |
| 按大小匹配合适空洞 | size class / best-fit 搜索 | bump 只需要追 free 指针，不需要搜索 |
| 线程安全 | 锁 / atomic | 一般 heap 由单线程持用，省去同步开销 |

**核心洞察：** bump allocator 不是"做不到"细粒度复用，而是**刻意不做**。它用"放弃复用"换来了极致的分配速度和零碎片——这对于 InnoDB 内部频繁的、短生命周期的临时分配（一次 B-tree 操作、一次 undo 构建）是正确权衡。补偿性手段是粗粒度的 `free_top` / `empty`，允许在一个操作的生命周期内重复利用同一块内存。

### 2.3 MEM_ROOT —— Server 层的 Arena Allocator

#### 2.3.1 核心思想

MEM_ROOT 是 MySQL Server 层（非 InnoDB）的 arena 分配器。思想与 mem_heap 同源（bump allocate + 批量释放），但设计上有显著差异以适应 Server 层的不同需求。

#### 2.3.2 主链路：一次 Server 层分配落到哪

与 mem_heap 的主链路（2.1.1）**并行**，但底层只有 `my_malloc` 一条路，**没有 Buffer Pool 分支**——这是 Server 层与引擎层分配器的根本区别：

```
MEM_ROOT::Alloc(length)                        my_alloc.h:146
  │  ALIGN_SIZE 对齐
  ├─ 快路径：当前 block 剩余够（free_end - free_start >= length）?
  │   ├─ 够 → bump（O(1)）                     my_alloc.h:160-164
  │   │       ret = m_current_free_start
  │   │       m_current_free_start += length
  │   │
  │   └─ 不够 → AllocSlow(length)              my_alloc.h:167
  │
  ▼
AllocSlow(length)                               my_alloc.cc:108
  ├─ length >= m_block_size 或 SINGLE_CHUNKS?
  │   ├─ 是 → AllocBlock(length, length)        my_alloc.cc:116-123
  │   │        大块单独分配，插到倒数第二位置（不干扰后续小块）
  │   │
  │   └─ 否 → ForceNewBlock(length)             my_alloc.cc:144
  │             └─ AllocBlock(ALIGN_SIZE(m_block_size), length)
  │                  → 新块成为 current block
  │             → bump 分配                       my_alloc.cc:147-149
  ▼
AllocBlock(wanted_length, minimum_length)       my_alloc.cc:58
  ├─ 容量检查：m_max_capacity 超限 → EE_CAPACITY_EXCEEDED / nullptr
  ├─ bytes_to_alloc = length + ALIGN_SIZE(sizeof(Block))
  ├─ new_block = my_malloc(m_psi_key, bytes_to_alloc, MYF(...))   my_alloc.cc:90
  │    → 底层 malloc → PFS 统计（唯一落点，不碰 Buffer Pool）
  ├─ m_block_size += m_block_size/2             my_alloc.cc:103  ← 指数增长
  └─ 返回 new_block
```

**与 mem_heap 主链路的两个关键差异**：

- **底层落点单一**：mem_heap 大块会分流到 `buf_block_alloc`（Buffer Pool），MEM_ROOT 永远只走 `my_malloc`——Server 层不持有 BP，也没有"拿一整个 16KB 页换临时数据"的问题
- **慢路径两类分叉**：mem_heap 慢路径只有"add_block（2× 增长）"一条；MEM_ROOT 慢路径多一条"**大块单独分配**"（`length >= m_block_size` 时单独开块、插倒数第二），避免偶发的大请求（如 2MB）把后续所有小分配都拖进大块浪费空间

> 完整对比见 2.3.3 的表格；这里只强调主链路上的分流差异。

#### 2.3.3 与 mem_heap 的详细对比

| 维度 | mem_heap | MEM_ROOT |
|------|---------|----------|
| **所属层次** | InnoDB 引擎 | MySQL Server 层 |
| **底层分配接口** | `ut::malloc_withkey` + Buffer Pool | `my_malloc` |
| **Buffer Pool 集成** | ✅ 大块直接从 BP 拿页面，统一管理 | ❌ 无关（Server 层不碰 BP） |
| **Block 增长策略** | 固定大小（初始 size 决定） | **指数增长**：`m_block_size += m_block_size/2`（每次 +50%），保证 O(1) 次 malloc 调用 |
| **增长哲学** | "使用者知道自己要多少"——InnoDB 内部场景高度可预测（一次 B-tree 搜索、一次 undo 构建），block size 在创建 heap 时就确定 | "自适应用户的任意需求"——Server 层面对无法预知的 SQL 复杂度（简单的 `SELECT 1` 到 20 表 JOIN），block size 随分配次数自动扩大 |
| **单块释放能力** | ✅ 支持 `mem_heap_free_top`（LIFO 栈式释放）。因为 block 内分配按地址线性递增，可精确回退 `free` 指针 | ❌ 完全不支持任何单块释放。`Clear()` 或 `ClearForReuse()` 只能全部清空 |
| **清空复用** | `mem_heap_empty`：回退 free 指针，可释放首 block 之外的 block | `Clear()`：释放所有 block；`ClearForReuse()`：保留首 block，返回 OS 的仅有一个 |
| **容量限制** | 无 | `m_max_capacity`，超限可触发 `EE_CAPACITY_EXCEEDED` |
| **多 block 维护策略** | block 链表通过 `prev` 单向链接，追加 block 时直接 `new_block->prev = m_current_block`，结构简单（因为 block 大小固定，不需要复杂的插入位置计算） | 大块分配时插入倒数第二位置（`new_block->prev = m_current_block->prev; m_current_block->prev = new_block`），将大块与当前活跃块隔离，避免后续小分配浪费空间 |
| **Debug 机制** | `MEM_NO_MANS_LAND`（自定义哨兵字节） | 依赖 Valgrind/ASan（`MEM_ROOT_SINGLE_CHUNKS` 模式：每分配一个独立 chunk） |
| **C++ 集成** | `Scoped_heap`（RAII 封装） | `Mem_root_allocator<T>`（STL allocator，支持 `std::vector` 等容器） |
| **AHI 死锁规避** | ✅ `MEM_HEAP_BTR_SEARCH` + `free_block_ptr` | ❌ 无此概念 |
| **PFS 埋点** | 通过 `ut::malloc_withkey` → `PSI_memory_key` | 通过 `my_malloc` + `PSI_memory_key` |
| **代码位置** | `storage/innobase/mem/` + `include/mem0mem.*` | `mysys/my_alloc.cc` + `include/my_alloc.h` |

#### 2.3.4 MEM_ROOT 的指数增长与独立大块分配

```cpp
// my_alloc.cc:101-103
// 每次成功分配后，下次 block size 增大 50%
m_block_size += m_block_size / 2;

// AllocSlow 中：大请求不干扰小请求
if (length >= m_block_size || MEM_ROOT_SINGLE_CHUNKS) {
    // 单独分配一个 block，插入倒数第二位置
    new_block->prev = m_current_block->prev;
    m_current_block->prev = new_block;
}
```

这个设计保证：偶尔来一个巨大的分配（如 2MB）不会导致后续所有小分配都从 2MB 块中分配（那会浪费大量空间），而是隔离到独立 block。

#### 2.3.5 MEM_ROOT 的衍生容器生态（Server 层独有，mem_heap 没有）

**这是 MEM_ROOT 与 mem_heap 最深层的差异**：MEM_ROOT 之上长出了一整套容器生态，而 mem_heap 只有原始分配函数族 + `Scoped_heap` RAII。原因在于两层代码风格——Server 层是 C++ 类层次重灾区（解析树、优化器、executor），天然需要 STL 风格泛型容器；InnoDB 核心是历史 C 风格，直接 `mem_heap_alloc` 分配字节 + 手动 offset 管理。

生态分四个层级：

```
① 方法级      MEM_ROOT::ArrayAlloc<T>(num, args...)    my_alloc.h:182
② allocator级 Mem_root_allocator<T>                    sql/mem_root_allocator.h
③ 容器级      Mem_root_array<T> / _YY<T>               sql/mem_root_array.h
              mem_root_deque<T>                        include/mem_root_deque.h
④ 适配 STL    std::vector/deque/list/set<T, Mem_root_allocator<T>>
```

**① ArrayAlloc —— 分配并构造 num 个 T**

```cpp
// my_alloc.h:182-189
T *ArrayAlloc(size_t num, Args... args) {
  static_assert(alignof(T) <= 8, "MEM_ROOT only returns 8-aligned memory.");
  T *ret = static_cast<T *>(Alloc(num * sizeof(T)));   // 一次性 Alloc
  // ... 然后逐个 placement new 构造
}
```

`ArrayAlloc` 是 MEM_ROOT **方法级**的便捷入口：一次 `Alloc` 拿到整块内存，再 placement new 构造每个元素。`mem_root_deque` 内部就用它分配 Block 数组（`m_root->ArrayAlloc<Block>(...)`）。

**② Mem_root_allocator —— 适配 STL 的桥**

```cpp
// sql/mem_root_allocator.h:101-110
pointer allocate(size_type n, ...) {
  pointer p = static_cast<pointer>(m_memroot->Alloc(n * sizeof(T)));  // 走 MEM_ROOT
  if (p == nullptr) throw std::bad_alloc();
  return p;
}
void deallocate(pointer, size_type) {}   // ★ 空操作：MEM_ROOT 不支持单块释放
```

`deallocate` 是空操作——这是适配器模式里的"退化适配"：STL 容器期望能逐个释放，但 MEM_ROOT 只能整体释放，所以干脆什么都不做，等 MEM_ROOT 生命周期结束统一回收。

`rebind`（`mem_root_allocator.h:155-158`）是关键机制：容器内部要分配的不是 `T`，而是节点类型（如 `std::list` 要分配 `_List_node`），`rebind` 提供"从 `Allocator<T>` 推导 `Allocator<InternalNode>`"的类型配方，并把底层 `m_memroot` 传递过去。

**③ 两个容器类 + 一个 deque（对比见下表）**

| | Mem_root_array | Mem_root_array_YY | mem_root_deque |
|---|---|---|---|
| 定义 | `mem_root_array.h:426` | `mem_root_array.h:61` | `mem_root_deque.h:110` |
| 内存布局 | 连续（`m_array` 裸指针） | 连续 | 分块（1KB/块）+ 物理索引 |
| 构造/析构 | 有 | **无**（Bison `%union` 要求 POD） | 有 |
| 用途 | 通用顺序容器 | 解析树语法栈 | 需两端插入/删除 |

**Mem_root_array 的内存布局**（好调试的原因）：

```cpp
// mem_root_array.h:409-413
protected:
  MEM_ROOT *m_root;
  Element_type *m_array;   // ★ 连续内存裸指针
  size_t m_size;
  size_t m_capacity;
```

`m_array` 指向连续内存，gdb 一条命令就能全打印：`p *arr.m_array@arr.m_size`。

**mem_root_deque 的分块 + 物理索引**（难调试的原因）：

```
m_blocks     Block 数组（每个 Block 存 block_elements 个元素，约 1KB，2 的幂）
m_begin_idx  物理起始索引
m_end_idx    物理结束索引（one-past-end）

元素定位（mem_root_deque.h:616）:
  get(physical_idx) = m_blocks[physical_idx / block_elements]
                        .elements[physical_idx % block_elements]
```

元素分散在多个 1KB 块里，靠 `m_begin_idx + 逻辑索引` 换算成物理索引，再二次间接寻址定位。gdb 的 libstdc++ pretty-printer 不认识这个自造类，所以只显示一堆内部字段，看不到元素值。

> **调试 mem_root_deque 的三个技巧**：
> 1. `p deq[i]` —— `operator[]` 是 const（`mem_root_deque.h:169`），gdb 能直接调
> 2. `p deq.size()` / `deq.front()` / `deq.back()`
> 3. 手动解引用：`p deq.m_blocks[X].elements[Y]`，其中物理索引 `= m_begin_idx + i`，`X = phys / block_elements`，`Y = phys % block_elements`

**为什么解析树偏爱 `_YY` 和 `mem_root_deque`**：

- `Mem_root_array_YY<PT_table_reference *>`（`from_clause`、`join_table_list`）：Bison 语法栈是 `union`，成员必须是 POD（无构造/析构），所以用无 ctor 的 `_YY` 版
- `mem_root_deque<List_item *>`（`many_values`）：VALUES 列表要 `push_back` 追加，也可能 `push_front`，deque 比 array 灵活
- 全部走 MEM_ROOT：**解析树生命周期 = 一次语句**，随 `THD::mem_root` 语句结束一次性释放，无需逐个 delete

### 2.4 buf_buddy —— Binary Buddy Allocator（二叉伙伴分配器）

#### 2.4.1 理论基础

二叉伙伴系统由 K. C. Knowlton 于 1965 年在 **CACM（Communications of the ACM）** 发表 *"A Fast Storage Allocator"* 中首次提出。Knuth 在 *The Art of Computer Programming* Vol.1 (1968) 中做了形式化完善。

#### 2.4.2 核心算法

内存按 2 的幂次划分为不同级别。分配时找最小满足块，不够就向上一级分裂；释放时检查伙伴是否也空闲，如果是则合并：

```
伙伴地址公式:  buddy_addr = addr XOR (1 << order)

例: order=3 (块大小 2³=8 单位)
    addr=0x00 → buddy=0x08
    addr=0x08 → buddy=0x00
```

#### 2.4.3 MySQL 中的实现

```cpp
// buf_pool_t 中的 zip_free 数组
UT_LIST_BASE_NODE_T(buf_buddy_free_t, list) zip_free[BUF_BUDDY_SIZES_MAX];

// 分配接口
static inline byte *buf_buddy_alloc(buf_pool_t *buf_pool, ulint size) {
    return buf_buddy_alloc_low(buf_pool, buf_buddy_get_slot(size));
}
```

通过 `BUF_BUDDY_STAMP_FREE` 魔数（写在 page 的 `space_id` 字段位置复用）判断伙伴是否空闲，避免引入额外元数据。

#### 2.4.4 使用场景

**仅用于压缩页（compressed page）frame 的分配**。压缩表的一个页面压缩后可能只有 1KB/2KB/4KB/8KB，不需要完整 16KB Buffer Pool 页面。buddy allocator 从 BP 中切割出合适大小的块，避免大量空间浪费。

### 2.5 jemalloc —— 底层 OS 分配器

#### 2.5.1 理论渊源

| 论文/工作 | 会议 | 年份 | 贡献 |
|----------|------|------|------|
| Hoard | **ASPLOS** | 2000 | per-thread heap + global heap，消除多线程 false sharing 和 memory blowup |
| jemalloc | **BSDCan** | 2006 | 产品级可扩展 malloc，多 arena 设计 |
| Slab Allocator | **USENIX Summer TC** | 1994 | 按对象大小预分配缓存（size class 思想源头）|

#### 2.5.2 size class 是什么？

size class（大小分级）是 segregated-fit 分配器的核心机制，也是 jemalloc 超越简单 buddy allocator 的关键设计。

**问题：** 假设程序连续分配 3、7、15 字节的对象，然后全部释放。下次再分配 5 字节，从哪里拿？

```
bump allocator 的回答：  从头开始推指针（无复用）
buddy allocator 的回答：  向上取 2^n（3→4，7→8，15→16），释放后合并回大块，
                          下次分配 5→8，重新从大块切割
size class 的回答：       3、5、7 都归入 "8 字节" 这个 size class，
                          释放的 3、7 字节块直接放入 8 字节 bin 的空闲链表，
                          下次分配 5 字节，直接从这个链表取——零切割、零搜索
```

**一句话定义：** size class 是将内存请求按**预设的分档**向上取整（round-up），同一档内的所有分配/释放共享一个 free list。释放的块不需要分裂或合并，直接作为同档内存复用。

| 概念对比 | 工作机制 | 碎片 | 复用效率 |
|---------|---------|------|---------|
| **bump allocator** | 无分类，线性推指针 | 零外部碎片 | 无单块复用 |
| **buddy allocator** | 按 2^n 分裂/合并 | 内部碎片（7→8 浪费 1 字节） | 需合并后才能复用为大块 |
| **size class (jemalloc)** | 按预定义档位向上取整，每档独立 free list | 内部碎片可控（档位设计均衡） | 释放后立即可被同档分配复用 |

**size class 与 buddy 的 2^n 有什么区别？**

buf_buddy 也按 2 的幂次分 size，但它服务于**分裂/合并**——buddy 的意义在于"两个相邻的 4K 空闲块可以合并成 8K"。size class 服务于**对象缓存**——意义在于"所有 5~8 字节的分配都用同一个 bin，释放后可以被下一个 5~8 字节请求直接拿走，不需要任何操作"。jemalloc 通常有几十个精细的 size class（不是简单的 2^n，而是 8, 16, 32, 48, 64, 80, 96...），这样内部碎片被控制在 25% 以内。

**理论来源：** Bonwick (Sun) 在 USENIX Summer TC 1994 发表的 *"The Slab Allocator: An Object-Caching Kernel Memory Allocator"* 首次系统化地提出了这个思想——为每种对象大小预分配 slab，同类对象复用同一 slab 内的槽位。jemalloc 将此思想从内核态带到用户态通用 malloc。

#### 2.5.3 jemalloc 的核心设计

```
请求 size
    │
    ▼
根据 size class 映射到 bin
    │
    ├── 小对象 (≤ 特定阈值)
    │   └── thread cache (tcache) → 无锁分配
    │
    ├── 中等对象
    │   └── arena → bin → slab → 有锁但竞争低
    │
    └── 大对象
        └── mmap → 直接映射，free 时 munmap
```

MySQL 编译时可选链接 jemalloc 或 glibc malloc。jemalloc 的优势：多核扩展性好、碎片更少、支持精确的内存用量统计（对 PFS 汇总有利）。

### 2.6 各分配器的理论渊源汇总

| 分配器 | 算法类型 | 核心论文 | 会议/出处 | 年份 |
|--------|---------|---------|----------|------|
| **mem_heap** | Bump/Arena | 无特定论文（1960s 编译器实践） | Knuth TAOCP Vol.1 | 1968 |
| **MEM_ROOT** | Arena（指数增长） | 无特定论文 | — | — |
| **buf_buddy** | Buddy Allocator | Knowlton, "A Fast Storage Allocator" | **CACM** | 1965 |
| **jemalloc** | Segregated-fit | Evans, "A Scalable Concurrent malloc(3)" | **BSDCan** | 2006 |
| **jemalloc 思想源泉** | Multi-thread scalable | Berger et al., "Hoard" | **ASPLOS** | 2000 |
| **size class 思想源泉** | Object-caching | Bonwick, "The Slab Allocator" | **USENIX Summer TC** | 1994 |

---

## 二、可观测性与内存管理的交汇

### 3.1 ut::malloc_withkey —— 两条线的唯一交汇点

`ut::malloc_withkey` 是 PFS 可观测性框架与 InnoDB 内存管理系统之间的桥梁。它**不实现任何新的分配算法**，而是对底层 `malloc`（jemalloc 或 glibc）做 PFS 埋点包装：

```cpp
// ut0new.h:619-626
inline void *malloc_withkey(PSI_memory_key_t key, std::size_t size) noexcept {
  using impl = detail::select_malloc_impl_t<WITH_PFS_MEMORY, false>;
  using malloc_impl = detail::Alloc_<impl>;
  return malloc_impl::alloc<false>(size, key());
}
```

每次分配时向 PFS 报告：哪个 `key`（模块标识）、分配了多少字节。这使得运维者可以通过 SQL 查询精确知道每个 InnoDB 子模块的内存占用。

### 3.2 全链路观测

```
mem_heap_create_block
  ├── 小块（<8KB）→ ut::malloc_withkey(KEY_mem_heap, size) → jemalloc → PFS 统计
  └── 大块（≥8KB, BUFFER 类型）→ buf_block_alloc → BP free list → PFS 统计

MEM_ROOT::AllocBlock
  └── my_malloc(PSI_key, size) → jemalloc → PFS 统计

buf_buddy_alloc
  └── zip_free[] 空闲链表 → buddy 内部管理 → buddy 自身统计
```

在 PFS 中查询效果：

```sql
SELECT event_name, current_alloc, high_alloc
FROM memory_summary_global_by_event_name
WHERE event_name LIKE '%innodb%'
ORDER BY current_alloc DESC;
-- memory/innodb/buf_buf_pool         8.5 GB
-- memory/innodb/mem_heap             1.2 GB
-- memory/innodb/btr_search           800 MB
-- memory/innodb/row0merge            500 MB
```

---
## 三、内存管理模块涉及的设计模式

#### 4.2.1 RAII（资源获取即初始化）—— Scoped_heap

**这不是 GoF 23 种模式之一，但属于 C++ 最重要的惯用法（idiom），优先级甚至高于 GoF 模式。**

```cpp
// mem0mem.h:440-507
struct Scoped_heap {
  struct mem_heap_free_functor {
    void operator()(mem_heap_t *heap) { mem_heap_free(heap); }
  };
  using Ptr = std::unique_ptr<mem_heap_t, mem_heap_free_functor>;
  Ptr m_ptr{};

  // 构造 = 获取资源
  Scoped_heap(size_t n, ut::Location location) noexcept
      : m_ptr(mem_heap_create(n, location, MEM_HEAP_DYNAMIC)) {}

  // 析构 = 自动释放资源（无需手动调用 mem_heap_free）
  ~Scoped_heap() = default;
};
```

**为什么需要 RAII？** C 风格的 `mem_heap_t` 需要手动调用 `mem_heap_free()`，在以下场景极易泄漏：
- 提前 `return` 的多个退出路径
- C++ 异常抛出（`ut_a` 断言失败会抛异常）
- 多层嵌套调用中某一层失败需要回滚

`Scoped_heap` 保证无论函数如何退出（正常 return、异常抛出、断言失败），heap 都会被释放。**在 InnoDB 的 C++ 代码中，这是内存安全的基石。**

**理论来源：** Bjarne Stroustrup, *"The C++ Programming Language"*, 第 4 版, 2013. RAII 是 C++ 最重要的资源管理思想，比 GoF 模式更基础、更常用。

**中文关键词检索：** "RAII C++" / "资源获取即初始化" / "scope-based resource management"

#### 4.2.2 Adapter（适配器模式）—— Mem_root_allocator 适配 STL

**定义（GoF）：** 将一个类的接口转换成客户希望的另外一个接口。Adapter 模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

**MySQL 中的实现：** `Mem_root_allocator<T>` 将 MEM_ROOT（arena allocator，有自己的 `AllocBlock()` / `Alloc()` 接口）适配为 STL 标准的 `std::allocator<T>` 接口：

```cpp
// 适配前：MEM_ROOT 无法直接用于 std::vector
// 适配后：
std::vector<Item *, Mem_root_allocator<Item *>> 
    items(Mem_root_allocator<Item *>(thd->mem_root));
// 这个 vector 的所有内存都从 MEM_ROOT 分配，而非 new/delete
```

**适配了什么？** STL 容器期望 `allocate()` / `deallocate()` 接口，MEM_ROOT 提供的是 `Alloc()` / `Clear()`，`Mem_root_allocator` 充当了中间的翻译层。注意 MEM_ROOT 不支持单块 `deallocate`，所以适配器在这个方法上做了"空操作"处理——这是适配器模式中常见的"退化适配"。

**中文关键词检索：** "适配器模式" / "Adapter Pattern" / "STL allocator 自定义"

#### 4.2.3 Strategy（策略模式）—— mem_heap 三种 type 的行为分支

**mysql 中的实现：** mem_heap 通过 `type` 参数选择不同的内存来源策略——这是策略模式在 C 语言中不用 vtable 的简化实现：

```cpp
constexpr uint32_t MEM_HEAP_DYNAMIC    = 0;  // 策略A：走 ut::malloc
constexpr uint32_t MEM_HEAP_BUFFER     = 1;  // 策略B：大块走 Buffer Pool
constexpr uint32_t MEM_HEAP_BTR_SEARCH = 2;  // 策略C：死锁安全模式（标志位）

// 在 mem_heap_create_block 中根据 type 选择分配路径：
if (type == MEM_HEAP_DYNAMIC || len < UNIV_PAGE_SIZE / 2) {
  // 策略A/B 的小块路径：ut::malloc_withkey → jemalloc/glibc
}
if (type == MEM_HEAP_BTR_SEARCH) {
  // 策略C：从预留 free_block_ptr 原子取，失败返回 NULL 而非崩溃
}
```

虽然用 `if/else` 而非虚函数实现，但这本质上是策略模式——**同一接口（mem_heap_alloc），不同策略（DYNAMIC/BUFFER/BTR_SEARCH），编译时通过 type 常量选择**。为什么不用 vtable？因为 type 在 heap 创建时就确定且不变，用常量比函数指针少一次间接寻址。

#### 4.2.4 Decorator（装饰器模式）—— ut::malloc_withkey

**定义（GoF）：** 动态地给一个对象添加一些额外的职责。就增加功能来说，Decorator 模式比生成子类更灵活。

**MySQL 中的实现：** `ut::malloc_withkey` 是对底层 `malloc`（jemalloc 或 glibc）的装饰——在不改变 `malloc` 接口的前提下，增加了 PFS 统计功能。这不是面向对象的装饰器，而是**函数级装饰（function-level decoration）：**

```
调用者 → ut::malloc_withkey(size, key) 
           ├── 下层 malloc(size)      ← 原始功能完全不变（被装饰者）
           └── PFS 报告 (key, size)   ← 装饰层增加的功能
```

在 OOP 中，装饰器通常通过组合 + 委托实现；在 C 中，通过包装函数实现。效果相同：**不修改底层代码，不改变接口，透明地增加功能。**

---

## 四、核心术语中英文对照（内存管理）

| 英文 | 中文标准译法 | 说明 |
|------|-------------|------|
| **Bump Allocator** | 指针碰撞分配器 / 线性分配器 | 推 `free` 指针，O(1) |
| **Arena Allocator** | 区域分配器 / 竞技场分配器 | 从一大块中逐次切分 |
| **Buddy Allocator** | 伙伴分配器 | 2^n 分裂/合并 |
| **Slab Allocator** | Slab 分配器 / 板层分配器 | 同大小对象复用槽位 |
| **Segregated-fit Allocator** | 分级适配分配器 | 按 size class 分 bin |
| **Free-list Allocator** | 空闲链表分配器 | 维护空闲块链表 |
| **Stack Allocator** | 栈式分配器 | LIFO，只能释放最后分配的块 |
| **Canary Byte / Guard Byte** | 金丝雀字节 / 哨兵字节 / 守卫字节 | 检测越界写入的魔数 |
| **Red Zone** | 红区 | ASan/Valgrind 的边界检测 |
| **External Fragmentation** | 外部碎片 | malloc 长期运行产生 |
| **Size Class** | 大小分级 / 大小类 | 将请求向上取整到预设档位，同档共享 free list，释放立即可复用。**存在于 jemalloc 层**（buf_buddy 的 2^n 服务于分裂/合并，不是对象缓存意义上的 size class） |
| **Huge Page** | 大页 / 巨页 | 2MB/1GB 页，减少 TLB miss |
| **jemalloc** | — | 不翻译，直接用英文 |
| **Buffer Pool (BP)** | 缓冲池 | InnoDB 核心缓存，也是 mem_heap 大块的来源 |

## 五、学术论文索引（内存分配器）

| 论文标题 | 作者 | 会议/期刊 | 年份 | MySQL 关联 |
|---------|------|----------|------|-----------|
| *A Fast Storage Allocator* | K. C. Knowlton | **CACM** | 1965 | **buf_buddy 的直接理论来源**：二叉伙伴系统 |
| *The Art of Computer Programming, Vol.1* | Donald Knuth | Addison-Wesley | 1968 | buddy system 的形式化分析、bump allocator 描述 |
| *The Slab Allocator: An Object-Caching Kernel Memory Allocator* | Jeff Bonwick (Sun) | **USENIX Summer TC** | 1994 | jemalloc size class 的思想先祖 |
| *Hoard: A Scalable Memory Allocator for Multithreaded Applications* | Berger et al. | **ASPLOS** | 2000 | per-thread heap 消除竞争，jemalloc 的设计源头 |
| *A Scalable Concurrent malloc(3) Implementation for FreeBSD* | Jason Evans | **BSDCan** | 2006 | **jemalloc 论文**，MySQL 底层可选链接 |
| *TCMalloc* | Google | 工业项目 | 2005 | 与 jemalloc 同期竞品，类似 thread-caching |

## 六、设计哲学（内存部分）

| 设计原则 | 思想/理论来源 | MySQL 中的实现 |
|---------|-------------|---------------|
| **极速分配** | Bump Allocator（Knuth 1968） | mem_heap：推 `free` 指针，O(1) |
| **BP 集成** | 自定义设计 | mem_heap `MEM_HEAP_BUFFER` 类型 |
| **死锁规避** | 锁顺序理论 | `MEM_HEAP_BTR_SEARCH` + 预留 `free_block_ptr` |
| **越界检测** | Canary/Guard Bytes（经典调试技术） | `MEM_NO_MANS_LAND`（0xCE/0xDF） |
| **伙伴分裂合并** | Buddy System (Knowlton, CACM 1965) | buf_buddy：压缩页 frame |
| **多核可扩展** | Hoard (ASPLOS 2000) / jemalloc (BSDCan 2006) | 底层 jemalloc |
| **指数增长 vs 固定大小** | 分配器自适应策略 | MEM_ROOT：`m_block_size *= 1.5`（自适应用户需求）；mem_heap：固定 block（场景可预测） |
| **栈式释放 vs 全量清空** | 生命周期管理策略 | mem_heap：`mem_heap_free_top`（LIFO）；MEM_ROOT：`Clear()` / `ClearForReuse()`（all-or-nothing） |
| **细粒度复用 vs 粗粒度复用** | 分配器设计取舍 | buf_buddy/jemalloc：free list 支持任意位置归还再分配；mem_heap/MEM_ROOT：放弃单块复用，换 O(1) 分配速度 + 零碎片，仅在 block/快照级复用 |
| **RAII 安全** | C++ 惯用法 | Scoped_heap / Mem_root_allocator |
| **Strategy（策略）** | GoF 1994 行为型模式 | mem_heap type：DYNAMIC/BUFFER/BTR_SEARCH 三种分配策略 |
| **Adapter（适配器）** | GoF 1994 结构型模式 | `Mem_root_allocator<T>`：将 arena 分配器适配为 STL allocator 接口 |
| **Decorator（装饰器）** | GoF 1994 结构型模式 | `ut::malloc_withkey`：对 malloc 透明增加 PFS 统计功能（函数级装饰） |
| **大页 TLB 优化** | OS 内存管理 | `large_page_alloc`：BP 底层用 `mmap(MAP_HUGETLB)` |

---

## 附录：推荐阅读文件清单（内存部分）

| 目的 | 文件路径 | 大小 |
|------|---------|------|
| mem_heap 数据结构 | `storage/innobase/include/mem0mem.h` | 526 行 |
| mem_heap 内联实现 | `storage/innobase/include/mem0mem.ic` | 530 行 |
| mem_heap block 分配（三种 type 分支） | `storage/innobase/mem/memory.cc` | 485 行 |
| MEM_ROOT 实现 | `mysys/my_alloc.cc` | 304 行 |
| MEM_ROOT 头文件 | `include/my_alloc.h` | 420 行 |
| buf_buddy 实现 | `storage/innobase/buf/buf0buddy.cc` | ~400 行 |
| buf_buddy 内联 | `storage/innobase/include/buf0buddy.ic` | ~91 行 |
| AHI 死锁规避（free_block 预留机制） | `storage/innobase/btr/btr0sea.cc` | — |
| Scoped_heap RAII 封装 | `storage/innobase/include/mem0mem.h:440-507` | — |
| temptable allocator（分层分配器） | `storage/temptable/include/temptable/allocator.h` | — |
| large_page_alloc（大页分配器） | `storage/innobase/include/detail/ut/page_alloc.h` | — |
| large_page_alloc (Linux 实现) | `storage/innobase/include/detail/ut/large_page_alloc-linux.h` | — |

---

*本文基于 MySQL 8.0 源码分析撰写。PFS 部分见 [pfs.md](pfs.md)，读源码通用知识地图见 [README.md](README.md)。*
