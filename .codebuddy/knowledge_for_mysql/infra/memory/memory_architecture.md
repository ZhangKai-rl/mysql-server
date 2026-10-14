# MySQL 全局内存架构总览：进程内存地图与预算视角

> 本篇是内存主题的**架构篇**：回答"MySQL 进程的内存由哪些区构成、每块谁在分配、由哪个参数控制、怎么算总量、OOM 了怎么查"。
> 分配器本身（mem_heap / MEM_ROOT / buf_buddy / jemalloc 的算法与实现）见 [allocator.md](allocator.md)；
> Buffer Pool 的页状态机与 LRU/刷盘机制见 [buffer_pool.md](../../innodb/buffer_pool.md)；
> PFS 框架原理见 [pfs.md](../observability/pfs/pfs.md)。
>
> 基于本仓库 MySQL 8.0.39 源码（`storage/innobase`、`sql` 等模块）。

## 目录

- [摘要](#摘要)
- [一、进程内存地图全景](#一进程内存地图全景)
- [二、InnoDB 全局常驻区](#二innodb-全局常驻区)
- [三、Server 层内存区](#三server-层内存区)
- [四、内存预算：总量怎么估算](#四内存预算总量怎么估算)
- [五、OOM 排查路线图（PFS 观测）](#五oom-排查路线图pfs-观测)
- [六、每块内存落到哪个分配器：与分配器矩阵的关系](#六每块内存落到哪个分配器与分配器矩阵的关系)
- [七、设计哲学（内存架构部分）](#七设计哲学内存架构部分)
- [八、版本演进与常见误区澄清](#八版本演进与常见误区澄清)
- [参考](#参考)

---

## 摘要

MySQL 进程内存按**生命周期与归属**分为三类：**全局常驻区**（Buffer Pool、redo 体系、dict/lock/trx sys 等，启动一次性分配、运行期不归还）、**每连接区**（THD 内存、线程栈、sort/join buffer、binlog cache 等，随连接数线性放大）、**瞬态区**（mem_heap 临时计算、DDL 归并排序、压缩 buffer 等，用完即弃）。三者叠加构成进程 RSS 的主体。内存观测的唯一权威入口是 Performance Schema 的 `memory_summary_*` 表族，其数据来自每处分配点的 PSI key 埋点。官方口径下总内存没有单一公式，只能按组件累加——最容易"乘出"的部分是每连接开销 × `max_connections`，最容易被忽视的是 PFS 自身只增不减。

---

## 一、进程内存地图全景

### 1.1 总图

```
mysqld 进程内存
│
├─ 全局常驻区（启动分配，运行期不归还，重启前不变）
│   ├─ Buffer Pool（最大头）
│   │   ├─ chunk 内存（ut::malloc_large_page_withkey → mmap/大页）
│   │   ├─ page_hash / zip_hash 表
│   │   ├─ watch 数组（purge 线程 + 1）
│   │   └─ AHI hash node（从 BP 免费页借，见 2.1.5）
│   ├─ redo 体系（log_sys）
│   │   ├─ log buffer（16MB 默认，malloc）
│   │   ├─ write-ahead buffer（8KB）
│   │   ├─ recent_written / recent_closed（Link_buf，1MB/2MB）
│   │   └─ flush/write events（各 2048 个 os_event）
│   ├─ dict_sys（数据字典 cache，hash cell + 表对象堆）
│   ├─ lock_sys（3 个 hash + 100K waiting slots）
│   ├─ trx_sys（MVCC 1024 read view 槽 + rsegs 数组）
│   ├─ srv_sys / purge_sys / ibuf / dblwr 结构（KB~MB 级）
│   └─ Server：table cache、table def cache、hostname cache、
│              MDL LF_HASH、sp_cache、PFS 自身
│
├─ 每连接区（× 连接数，连接关闭才释放；thread cache 保留期不释放）
│   ├─ 线程栈（thread_stack，默认 1MB 级）
│   ├─ THD 对象 + main_mem_root（语句级复用）
│   ├─ net buffer ×2（net_buffer_length → 按需涨到 max_allowed_packet）
│   ├─ sort buffer / join buffer(hash join) / read buffer
│   ├─ binlog trx cache + stmt cache（各 binlog_cache_size 起步）
│   └─ TempTable 共享块（从全局共享块池借一块，RAM 优先）
│
└─ 瞬态区（查询/DDL 期间分配，结束即弃）
    ├─ mem_heap（InnoDB 内部临时结构：row0sel/row0vers/row0upd…）
    ├─ ddl0merge 归并排序 buffer（online DDL）
    ├─ FTS doc cache（随表缓存淘汰）
    ├─ 压缩页 zlib 瞬态 buffer（dblwr TPC）
    └─ MDL_lock / trx_t 等按需 malloc 的对象
```

### 1.2 三个分类的生命周期语义

**⚠️ 一个必须先纠正的事实**：**MySQL 8.0.39 没有任何"计算 mysqld 总内存"的函数**。在 `sql/mysqld.cc` 里检索 `total memory` / `total_memory` / `memory_used` / `estimate.*memory` **全部 0 命中**。官方唯一"总内存"口径是 sys schema 视图对 PFS 的求和：

```sql
-- scripts/sys_schema/views/p_s/memory_global_total.sql
CREATE OR REPLACE
  ALGORITHM = TEMPTABLE                       -- ★ 查这个视图本身还要再吃一份临时表内存
  DEFINER = 'mysql.sys'@'localhost'
  SQL SECURITY INVOKER
VIEW memory_global_total (
  total_allocated
) AS
SELECT format_bytes(SUM(CURRENT_NUMBER_OF_BYTES_USED)) total_allocated
  FROM performance_schema.memory_summary_global_by_event_name;
```

> ★★ 所以"总内存 = ?"的答案是 **`SUM(alloc_size - free_size)`**，而**不是**任何引擎上报的量。这也意味着：所有未埋 PSI key 的分配（见 §5）**不在这个总和里**。

**三类内存的判据不是"大小"，而是"谁来分配 + 何时归还"**：

```cpp
// 全局常驻：启动时一次分配，运行期不归还，只在 shutdown 释放
  buf_pool_ptr = (buf_pool_t *)ut::zalloc_withkey(
      UT_NEW_THIS_FILE_PSI_KEY, n_instances * sizeof *buf_pool_ptr);
  //                            ★ 实例数组本身也是常驻，PSI key = UT_NEW_THIS_FILE_PSI_KEY

// 每连接：THD 构造时创建，release_resources() 归还，与连接同生死
  main_mem_root(key_memory_thd_main_mem_root,                     // memory/sql/thd::main_mem_root
                global_system_variables.query_alloc_block_size),  // 默认 8192，不预分配
  //  ★ 构造时是 0 字节（MEM_ROOT 懒分配），第一次查询才真正 malloc

// 瞬态：用完即弃，典型是 mem_heap
  mem_heap_t *heap = mem_heap_create(...);
  ...
  mem_heap_free(heap);
```

**★ 三者的边界并不干净**——最容易误判的两处：

1. **`recv_sys`**：崩溃恢复期间是"大户"（`buf_len = min(RECV_PARSING_BUF_SIZE, srv_log_buffer_size)` + 每个 space 一个 `mem_heap_create(256, ..., MEM_HEAP_FOR_RECV_SYS)`，随脏页数增长），但恢复结束就 `recv_sys_close()` 释放——**它不是常驻项**。
2. **`trx_pools`**：名义上是"事务对象池"（瞬态），实际 **`Pool` 构造函数立刻 `zalloc(4 MiB)` 且从不归还**——**行为上属于常驻区**：

```cpp
  Pool(size_t size) : m_end(), m_start(), m_size(size), m_last() {
    m_start = reinterpret_cast<Element *>(
        ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, m_size));   // ★ 一次性全部分配
    m_last = m_start;
    m_end = &m_start[m_size / sizeof(*m_start)];
    /* Note: Initialise only a small subset, even though we have
    allocated all the memory. This is required only because PFS
    (MTR) results change if we instantiate too many mutexes up
    front. */
    init(std::min(size_t(16), size_t(m_end - m_start)));
  }
  ...
  bool add_pool(size_t n_pools) {   // 池满 → 再追加一个 4 MiB 块
    pool = ut::new_withkey<PoolType>(UT_NEW_THIS_FILE_PSI_KEY, m_size);
    if (pool != nullptr) m_pools.push_back(pool);
```

```cpp
typedef Pool<trx_t, TrxFactory, TrxPoolLock> trx_pool_t;
typedef PoolManager<trx_pool_t, TrxPoolManagerLock> trx_pools_t;
static trx_pools_t *trx_pools;
/** Size of on trx_t pool in bytes. */
static const ulint MAX_TRX_BLOCK_SIZE = 1024 * 1024 * 4;
```

> ⚠️ **`trx_pools` 不是数组**，是 `PoolManager`（内部 `std::vector<PoolType*>`）。每个 pool 固定 4 MiB，**只增不减**。

| 类别 | 分配时机 | 释放时机 | 失控时的典型表现 |
|------|----------|----------|------------------|
| 全局常驻 | `srv_start` → `srv0start.cc` 启动序列 | 进程退出 | RSS 基线高，与连接数无关 |
| 每连接 | 连接建立/语句执行 | 连接关闭；进 thread cache 则**保留** | 连接数 × 每连接开销，最容易 OOM |
| 瞬态 | 语句/事务/DDL 期间 | 语句/事务/DDL 结束 | 大事务/大 DDL 的尖峰，难以在 `ps` 里抓到 |

> 官方文档（memory-use）明确："线程结束时，若进入 thread cache 保留，内存不释放"——这是 thread cache 的一个隐蔽副作用：缓存线程不仅仅是缓存 OS 线程，还连带保留了它的 THD mem_root 等内存。

---

## 二、InnoDB 全局常驻区

### 2.1 Buffer Pool：chunk 机制与大页分配

#### 2.1.1 内存从哪来

BP 内存不是一块连续 malloc，而是按 **chunk** 分块分配，每 chunk 用 `ut::malloc_large_page_withkey` 尝试 **OS 大页（huge page）**，失败自动回退普通页对齐分配：

```cpp
// buf0buf.cc:883-915（buf_pool_t::allocate_chunk）
chunk->mem = ut::malloc_large_page_withkey(
    ut::make_psi_memory_key(mem_key_buf_buf_pool), mem_size,
    ut::fallback_to_normal_page_t{});
```

```cpp
// ut0new.h:1361-1365（回退逻辑）
if (large_pages_enabled) {
  large_page_mem = malloc_large_page_withkey(key, size);   // mmap + MAP_HUGETLB
}
return large_page_mem ? large_page_mem : malloc_page_withkey(key, size);
```

BP 体积远超 glibc malloc 的 `MMAP_THRESHOLD`（128KB），所以无论大页与否最终都走 **mmap 匿名映射**路径。释放端 `deallocate_chunk()`（buf0buf.cc:917-926）走 `ut::free_large_page` 即 munmap。

**调用栈（启动）**：

```
srv_start（srv0start.cc）
  └─ srv_start_inner（srv0start.cc:1774）
       └─ buf_pool_init(srv_buf_pool_size, srv_buf_pool_instances)   // buf0buf.cc:1453
            ├─ 按 instance 并行建池（每 instance 一条 std::thread，1509-1512）
            └─ buf_pool_create()                                    // buf0buf.cc:1200
                 └─ 逐 chunk：allocate_chunk() → buf_chunk_init()
                      （chunk 内布局见 buffer_pool.md「slot-based 布局细节」）
```

#### 2.1.2 chunk 内布局与"隐藏内存"

chunk 的**内部布局**（`buf_block_t` 控制块数组放头部、frame 按 16KB 对齐排列、`buf_block_t::page` 必须是首成员）见 [buffer_pool.md](../../innodb/buffer_pool.md)「slot-based 布局细节」。

内存总账只需记住一个数：`mem_size()`（buf0buf.ic:53-61）通过 `ut::large_page_allocation_size` 反查实际分配大小——chunk 实际占用比 `innodb_buffer_pool_size` 名义值**多出 block 描述符区**，这是预算时容易被忽略的常数开销（约每页几百字节量级）。

**★ `buf_chunk_t` 真实定义在 `include/buf0buf.ic:52`**（⚠️ `buf0buf.h` 只有前向声明；**`buf/buf0chunk.cc` 这个文件不存在**）：

```cpp
struct buf_chunk_t {
  ulint size;          /* size of frames[] and blocks[] */
  unsigned char *mem;  /* pointer to the memory area which was allocated
                       for the frames */
  buf_block_t *blocks; /* array of buffer control blocks */
  size_t mem_size() const { return ut::large_page_allocation_size(mem); }
  bool madvise_dump();
  bool madvise_dont_dump();
  bool contains(const buf_block_t *ptr) const { ... }
};
```

> ★ 只有 **3 个数据字段**——**frame 数组没有独立指针字段**，它是 `ut_align(chunk->mem, UNIV_PAGE_SIZE)` 之后跳过描述符区的那部分，布局完全由 `buf_chunk_init()` 运行时算出来。

**★ `buf_chunk_init()` 的"预留 + 扣减"两步**（`buf0buf.cc:990`）——这就是"隐藏内存"的全部：

```cpp
  /* Round down to a multiple of page size, although it already should be. */
  mem_size = ut_2pow_round(mem_size, UNIV_PAGE_SIZE);
  /* Reserve space for the block descriptors. */
  mem_size += ut_2pow_round(                              // ★ ①先按页数预留描述符区
      (mem_size / UNIV_PAGE_SIZE) * (sizeof *block) + (UNIV_PAGE_SIZE - 1),
      UNIV_PAGE_SIZE);
  ...
  chunk->blocks = (buf_block_t *)chunk->mem;              // ★ 描述符数组 = mem 起点
  frame = (byte *)ut_align(chunk->mem, UNIV_PAGE_SIZE);   // ★ frame 起点 = 16K 对齐
  chunk->size = ut::large_page_allocation_size(chunk->mem, ...) / UNIV_PAGE_SIZE
                - (frame != chunk->mem);
  /* Subtract the space needed for block descriptors. */
  {
    ulint size = chunk->size;
    while (frame < (byte *)(chunk->blocks + size)) {      // ★ ②frame 逐个后移直到越过描述符区
      frame += UNIV_PAGE_SIZE;
      size--;
    }
    chunk->size = size;                                   // ★ 真正能用的页数被砍掉了
  }
  block = chunk->blocks;
  for (i = chunk->size; i--;) {
    buf_block_init(buf_pool, block, frame);               // ★ blocks[i] ↔ 第 i 个 frame
    UT_LIST_ADD_LAST(buf_pool->free, &block->page);
    block++;
    frame += UNIV_PAGE_SIZE;
  }
```

```
chunk->mem ──► ┌──────────────────────────────────────────┐
 (= blocks)    │ buf_block_t 描述符数组 (chunk->size 个)   │ ← 预留：上取整到 16K
  frame ──────►├──────────────────────────────────────────┤ ← ut_align(mem, 16K)
               │ frame[0] | frame[1] | ... | frame[n-1]   │ ← 每个 UNIV_PAGE_SIZE
               └──────────────────────────────────────────┘
```

> ★★ **预算公式**：`chunk 实际字节 = mem_size`（已含描述符），但**可用页数 `chunk->size` < `mem_size / 16K`**。差额 = 描述符区。所以"BP = 128 GiB"实际 mmap 出来的比 128 GiB 多。

**大页分配与回退**（⚠️ **`os_mem_alloc_large()` 在 8.0.39 已不存在**，只剩两处注释提及）：

```cpp
  chunk->mem = static_cast<uint8_t *>(ut::malloc_large_page_withkey(
      ut::make_psi_memory_key(mem_key_buf_buf_pool), mem_size,
      ut::fallback_to_normal_page_t{}));
```

```cpp
inline void *malloc_large_page_withkey(PSI_memory_key_t key, std::size_t size,
                                       fallback_to_normal_page_t,
                                       bool large_pages_enabled = os_use_large_pages) noexcept {
  void *large_page_mem = nullptr;
  if (large_pages_enabled) {
    large_page_mem = malloc_large_page_withkey(key, size);
  }
  return large_page_mem ? large_page_mem : malloc_page_withkey(key, size);
  //  ★★ 回退逻辑就这一行三目：大页 mmap 失败 → 普通页对齐 mmap
}
```

```cpp
  static inline void *alloc(std::size_t size) {
    auto total_len = round_to_next_multiple(
        size + page_allocation_metadata::len, large_page_default_size);
    //  ★★ 每个 chunk 都向上取整到大页尺寸整数倍 + 一段元数据头
    auto mem = large_page_aligned_alloc(total_len);
    if (unlikely(!mem)) return nullptr;
    page_allocation_metadata::datalen(mem, total_len);
    page_allocation_metadata::page_type(mem, Page_type::large_page);
    return static_cast<uint8_t *>(mem) + page_allocation_metadata::len;
  }
```

> ★ `large_page_aligned_alloc()` 在 Linux 是 `mmap(..., MAP_ANON)`，macOS 是 `mmap(..., VM_FLAGS_SUPERPAGE_SIZE_2MB)`，Windows 是 `VirtualAlloc`。元数据头额外吃掉约一个 CPU_PAGE_SIZE。

**还有两个易漏的常驻项**（都在 `buf_pool_init()` 里）：

```cpp
  buf_chunk_map_reg =
      ut::new_withkey<buf_pool_chunk_map_t>(UT_NEW_THIS_FILE_PSI_KEY);
  ...
  btr_search_sys_create(buf_pool_get_curr_size() / sizeof(void *) / 64);   // ★ AHI cell 数组
  buf_stat_per_index = ut::new_withkey<buf_stat_per_index_t>(
      ut::make_psi_memory_key(mem_key_buf_stat_per_index_t));
```

#### 2.1.3 配套结构

| 结构 | 分配点 | 大小 |
|------|--------|------|
| `page_hash` | `buf_pool_create` buf0buf.cc:1318-1320 | `2 × curr_size` 个 cell，16 把分片锁 |
| `zip_hash` | 同上 1322 | `2 × curr_size` 个 cell |
| `watch` 数组 | 同上 1335-1336 | `srv_n_purge_threads + 1` 个哨兵页 |

`page_hash` 是 `ib_create(..., MEM_HEAP_FOR_PAGE_HASH)` 分配的——即 hash cell 走 malloc，而 hash node 挂到 heap 后走 BUFFER 型分配（从 BP 免费页借），详见 allocator.md 的 MEM_HEAP_BTR_SEARCH 死锁规避。

#### 2.1.4 参数体系与约束

- `innodb_buffer_pool_size`：默认 128MB，最小 5MB，最大 `LLONG_MAX`（`srv0srv.cc:425-429`）。**8.0 没有 5GB 上限**（见 8.2 澄清）。
- `innodb_buffer_pool_chunk_size`：默认 128MB，最小 1MB，READONLY（`ha_innodb.cc:22307-22314`）。
- `innodb_buffer_pool_instances`：默认 0 自动。自动规则（`innodb_init_params`，`ha_innodb.cc:4563-4601`）：BP ≥ `BUF_POOL_SIZE_THRESHOLD = 1GB`（`srv0start.h:65-67`）才允许多实例，否则强制 1 个。resize 校验同样以 1GB 为界：多实例时 BP < 1GB 直接拒绝（`innodb_buffer_pool_size_validate`，`ha_innodb.cc:20476-20536`）。
- 分配失败：`srv0start.cc:1774-1780` 检查 `buf_pool_init` 返回值，失败打印 `ER_IB_MSG_1131`（Cannot allocate memory for the buffer pool）并中止启动。

#### 2.1.5 AHI：内存从 BP 免费页"借"

AHI 是 InnoDB 里唯一一个**不属于 BP 却从 BP 拿内存**的结构：hash cell 数组本身是 malloc 的（cell 数 = BP 页数 / 512，`buf_pool_init` 尾部经 `btr_search_sys_create` 分配，buf0buf.cc:1541），但 **hash node 从 BP 免费页分配**（`MEM_HEAP_FOR_BTR_SEARCH` → `free_block_ptr` 原子取，取不到返回 NULL）。关闭 `innodb_adaptive_hash_index` 省下的是 node 占用的 BP 页——这是"BP 内存被 AHI 吃掉"说法的源码依据。哈希键自适应、锁分片等机制见 [ahi.md](../../index/ahi.md)「开关、内存与监控」。

#### 2.1.6 resize 的临时内存

`buf_pool_resize()`（buf0buf.cc:2137）在线调整大小。预算视角的关键点是 `buf_pool_resize_hash()`（buf0buf.cc:2008-2082）：构造一个 **2×新大小** 的临时 hash 表，`std::swap` 后释放旧表——**resize 期间 page_hash 需要双份内存**，预算紧张时这可能成为压垮骆驼的最后一根稻草。resize 后 dict 的 hash 也要跟着调：`dict_sys_resize()`（dict0dict.cc:4603-4609）。chunk 增删协议（withdraw / 七锁 / `buf_resize_thread`）见 [buffer_pool.md](../../innodb/buffer_pool.md)「`buf_dump` / `buf_resize`：两个按需线程」。

### 2.2 redo 体系内存（log_sys）

`log_t`（单例 `log_sys`，`include/log0sys.h`）在 `log_sys_create()`（log0log.cc:584）里一次性把整条 redo 管线的内存分配齐：

```
log_sys_create（log0log.cc:584）
  ├─ log_allocate_buffer(log)              // 660：log buffer，malloc
  ├─ log_allocate_write_ahead_buffer(log)  // 661：write-ahead 8KB
  ├─ log_allocate_recent_written(log)      // 662：Link_buf 1MB
  ├─ log_allocate_recent_closed(log)       // 663：Link_buf 2MB
  ├─ log_allocate_flush_events(log)        // 664：2048 个 os_event
  └─ log_allocate_write_events(log)        // 665：2048 个 os_event
```

各块明细：

| 块 | 分配路径 | 大小/参数 |
|----|----------|-----------|
| `log.buf` | `ut::aligned_array_pointer::alloc_withkey(log_buffer_memory_key)`（log0log.cc:1339-1351），即 **OS 堆 malloc**，PFS 里叫 `memory/innodb/log0log` | `innodb_log_buffer_size` 默认 16MB，最小 256KB，最大 `UINT32_MAX`（≈4GB，`log0constants.h:484-491`） |
| write-ahead buf | 同函数 | `INNODB_LOG_WRITE_AHEAD_SIZE_DEFAULT` = 8KB |
| `recent_written`/`recent_closed` | Link_buf 分配 | 默认 1MB / 2MB（`log0constants.h:493-509`） |
| flush/write events | `os_event_create()` × `srv_log_flush_events`（默认 2048） | 每个 event 一个 malloc 对象 |

两点值得注意：

1. **log buffer 不是 BP 的一部分**，走的是普通 malloc 路径，所以它不会出现在 BP 相关统计里，而是一个独立的 PFS event `memory/innodb/log0log`。
2. **log buffer 实际峰值可能略超配置值**：超大 mtr 时自动按 1.382 倍扩容；SET GLOBAL 缩容在现有数据超限时失败。resize 的三锁协议与 buffer 环形定址机制见 [redo_log.md](../../innodb/redo_log.md)「log buffer 的在线变更」。

**⚠️ 勘误：`innodb_log_recent_written_size` / `_closed_size` 的单位是"slot 数"，不是字节**：

```cpp
template <typename Position>
Link_buf<Position>::Link_buf(size_t capacity)
    : m_capacity(capacity), m_tail(0) {
  if (capacity == 0) { m_links = nullptr; return; }
  ut_a((capacity & (capacity - 1)) == 0);          // ★ 必须是 2 的幂
  m_links = ut::new_arr_withkey<std::atomic<Distance>>(
      UT_NEW_THIS_FILE_PSI_KEY, ut::Count{capacity});   // ★ Distance = lsn_t = 8 字节
  for (size_t i = 0; i < capacity; ++i) m_links[i].store(0);
}
```

| 变量 | 默认值（slots） | 实际字节 |
|---|---|---|
| `innodb_log_recent_written_size` | `1024 * 1024` | **8 MiB** |
| `innodb_log_recent_closed_size` | `2 * 1024 * 1024` | **16 MiB** |

> ★★ 这两个加起来 **24 MiB**，是 redo 体系里除 log buffer 外最大的一块，但常被称为"1MB / 2MB"（单位搞错）。

```cpp
constexpr ulong INNODB_LOG_BUFFER_SIZE_DEFAULT = 16 * 1024 * 1024UL;
constexpr ulong INNODB_LOG_RECENT_WRITTEN_SIZE_DEFAULT = 1024 * 1024;
constexpr ulong INNODB_LOG_RECENT_CLOSED_SIZE_DEFAULT = 2 * 1024 * 1024;
constexpr ulong INNODB_LOG_EVENTS_DEFAULT = 2048;
constexpr ulong INNODB_LOG_WRITE_AHEAD_SIZE_DEFAULT = 8192;
```

```cpp
static void log_allocate_buffer(log_t &log) {
  ut_a(srv_log_buffer_size >= INNODB_LOG_BUFFER_SIZE_MIN);
  log.buf.alloc_withkey(ut::make_psi_memory_key(log_buffer_memory_key),
                        ut::Count{srv_log_buffer_size});
}
static void log_calc_buf_size(log_t &log) {
  log.buf_size = srv_log_buffer_size;                 // ★ 原样使用，没有取 2 的幂
  log.buf_size_sn = log_translate_lsn_to_sn(log.buf_size);
}
static void log_allocate_flush_events(log_t &log) {
  const size_t n = srv_log_flush_events;              // 默认 2048
  ut_a((n & (n - 1)) == 0);                           // ★ 必须是 2 的幂
  log.flush_events = ut::new_arr_withkey<os_event_t>(
      UT_NEW_THIS_FILE_PSI_KEY, ut::Count{n});
  for (size_t i = 0; i < n; ++i) log.flush_events[i] = os_event_create();
}
```

> ⚠️ **`log_t` 在 8.0.39 里没有 `flush_buf` 字段**（8.0 已改为 log_writer/log_flusher 线程模型），只有 `write_ahead_buf`。内存成员共 6 块：`buf` / `recent_written` / `recent_closed` / `write_ahead_buf` / `write_events[]` / `flush_events[]`。

**在线 resize 的临时内存**（`log0log.cc:1230`）：

```cpp
bool log_buffer_resize_low(log_t &log, size_t new_size, lsn_t end_lsn) {
  ...
  /* Save the contents. */
  byte *tmp_buf = ut::new_arr_withkey<byte>(UT_NEW_THIS_FILE_PSI_KEY,
                                            ut::Count(end_lsn - start_lsn));
  for (auto i = start_lsn; i < end_lsn; i += OS_FILE_LOG_BLOCK_SIZE)
    std::memcpy(&tmp_buf[i - start_lsn], &log.buf[i % log.buf_size],
                OS_FILE_LOG_BLOCK_SIZE);
  /* Re-allocate log buffer. */
  srv_log_buffer_size = static_cast<ulong>(new_size);
  log_deallocate_buffer(log);
  log_allocate_buffer(log);
  /* Restore the contents. */ ... ut::delete_arr(tmp_buf);
```

### 2.3 数据字典 cache（dict_sys）

dict_sys 的**内存身份**：`dict_init()`（dict0dict.cc:1009-1038）一次 `zalloc` 出 `dict_sys_t` + 两个 hash 表（cell 数 = `buf_pool_get_curr_size() / (512 × UNIV_WORD_SIZE)`，随 BP 大小联动，BP resize 后经 `dict_sys_resize()` 重建）；每个 `dict_table_t`/`dict_index_t` 在各自的 `table->heap`（mem_heap）上分配，`dict_sys->size` 记账字节数。**随表数量增长**：表对象在 LRU 上，字典对象随 table cache 淘汰释放。

注意：不存在 `innodb_dictionary` 系统变量——8.0 的权威 DD 缓存主体在 Server 层 `sql/dd/cache/`（见 3.8），dict_sys 只缓存 SE private 元数据。dict_sys 的结构与"两套字典并存"机制见 [innodb_dict.md](../../server/dd/innodb_dict.md)。

**⚠️ 常见误解：dict cache 用 mem_heap 装** —— 不对。它**分两层**，增长机制完全不同：

```cpp
void dict_init(void) {
  dict_sys = static_cast<dict_sys_t *>(
      ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sizeof(*dict_sys)));
  ...
  dict_sys->table_hash = ut::new_<hash_table_t>(
      buf_pool_get_curr_size() / (DICT_POOL_PER_TABLE_HASH * UNIV_WORD_SIZE));
  dict_sys->table_id_hash = ut::new_<hash_table_t>(
      buf_pool_get_curr_size() / (DICT_POOL_PER_TABLE_HASH * UNIV_WORD_SIZE));
  //   ★ 普通 hash table（heap == nullptr），不是 mem_heap
}
/** buffer pool max size per table  hash table fixed size in bytes */
constexpr uint32_t DICT_POOL_PER_TABLE_HASH = 512;
```

| 层 | 分配方式 | 随什么增长 |
|---|---|---|
| **哈希桶数组**（`table_hash` / `table_id_hash`） | `ut::new_<hash_table_t>(n)`，n = `BP字节数 / (512 × sizeof(void*))` | **只随 BP 大小**，不随表数量；只在 BP resize 时重建 |
| **真正的表/索引对象** | 每张 `dict_table_t` 自带 `mem_heap_t *heap` | **随表数量**，靠 `table_LRU` 驱逐 |

> ★★ 这就是"dict cache 无上限"说法的源码依据：**哈希桶数组不限制表数**（64 位下 BP=128 GiB → 各 32768 cell → 各 256 KiB），真正的上限靠 `table_LRU` 的 LRU 驱逐。`dict_sys->size` 记的是第二层的字节数，也就是 `SHOW ENGINE INNODB STATUS` 里 `Dictionary memory allocated` 那一行的值。

```cpp
  /* BP resize 后重建（dict0dict.cc:4600） */
  ut::delete_(dict_sys->table_hash);
  ut::delete_(dict_sys->table_id_hash);
  dict_sys->table_hash = ut::new_<hash_table_t>(
      buf_pool_get_curr_size() / (DICT_POOL_PER_TABLE_HASH * UNIV_WORD_SIZE));
  ...
  for (auto table : dict_sys->table_LRU) { ... HASH_INSERT(...); }
```

### 2.4 lock sys

lock_sys 的**内存身份**：`lock_sys_create()`（lock0lock.cc:300-331）一次 `zalloc` 出 `lock_sys_t` + `waiting_threads` 槽位数组（`srv_max_n_threads = 100 × 1024` 个 `srv_slot_t`，**固定 ≈ 800KB~1MB，与并发无关**）+ 三个行锁 hash 表。整体 MB 级、基本不随负载增长；真正的增长点是**每行锁的 `lock_t`**（高并发长事务下动态 malloc，PFS 的 lock 相关 key 统计）。槽位协议与 512+512 分片锁机制见 [innodb_trx_lock.md](../lock/transactional/innodb_trx_lock.md)。

**★ `lock_sys_t` 与 `waiting_threads[]` 是一次 zalloc 打包分配的**（`lock0lock.cc:299`）：

```cpp
void lock_sys_create(ulint n_cells) {   // n_cells = srv_lock_table_size
  ulint lock_sys_sz;
  lock_sys_sz = sizeof(*lock_sys) + srv_max_n_threads * sizeof(srv_slot_t);
  lock_sys = static_cast<lock_sys_t *>(
      ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, lock_sys_sz));
  new (lock_sys) lock_sys_t{};
  void *ptr = &lock_sys[1];                                  // ★ 紧跟 lock_sys_t 之后
  lock_sys->waiting_threads = static_cast<srv_slot_t *>(ptr);
  lock_sys->last_slot = lock_sys->waiting_threads;
  ...
}
```

**★ `rec_hash` 的 cell 数是硬编码的 `5 × BP 页数`**（`ha_innodb.cc:5090`）——这是一条**随 BP 线性增长**的常驻内存，很容易被漏算：

```cpp
  srv_lock_table_size = 5 * (srv_buf_pool_size / UNIV_PAGE_SIZE);
```

> BP = 128 MiB / 16K = 8192 页 → `rec_hash` = 40960 cells ≈ 320 KiB 桶数组（不含 node）。
> ⚠️ `prdt_hash` / `prdt_page_hash` **不在 `lock_sys_create()` 里初始化**（走 lazy 路径）。

```cpp
struct lock_sys_t {
  locksys::Latches latches;       // ★ 512 page shard + 512 table shard + 分片 global_latch
  hash_table_t *rec_hash;         // 行锁哈希（按 page_id）
  hash_table_t *prdt_hash;        // predicate 锁
  hash_table_t *prdt_page_hash;   // predicate page 锁
  char pad2[ut::INNODB_CACHE_LINE_SIZE];   // ★ 避免 false sharing
  Lock_mutex wait_mutex;
  srv_slot_t *waiting_threads;    // ★ 锁等待槽位数组
  srv_slot_t *last_slot;
  ...
};
```

> ⚠️ **`lock_sys` 里没有 table hash**——表锁是用**链表**（挂在 `dict_table_t::locks` 上）组织的，不是哈希表。

### 2.5 trx sys 与 MVCC

trx_sys 的**内存身份**：`trx_sys_create()`（trx0sys.cc:561-575）一次 `zalloc` 出 `trx_sys_t`，其中 `mvcc` 预分配**固定 1024 个 read view 槽**——长事务并发再多也不膨胀。`trx_t` 本身按事务 `trx_create_low()` 动态分配、事务结束归还（trx pool 复用）。事务机制见 [trx.md](../../innodb/trx.md)。

**⚠️ 但请回看 §1.2**：`trx_pools` 每块 **4 MiB 一次性 zalloc 且从不归还**，并发事务多时按 4 MiB 台阶增长。所以"trx_t 动态分配"只是**从池里取**，不代表内存会回落。

```cpp
void trx_sys_create(void) {
  trx_sys = static_cast<trx_sys_t *>(
      ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sizeof(*trx_sys)));
  ...
  trx_sys->mvcc = ut::new_withkey<MVCC>(UT_NEW_THIS_FILE_PSI_KEY, 1024);   // ★ 1024 read view
  new (&trx_sys->rw_trx_ids)
      trx_ids_t(ut::allocator<trx_id_t>(mem_key_trx_sys_t_rw_trx_ids));    // ★ 按需增长
  for (auto &shard : trx_sys->shards) new (&shard) Trx_shard{};            // TRX_SHARDS_N 个
}
```

> ★ `trx_sys_t` 里 `Trx_shard shards[TRX_SHARDS_N]` 是**内联数组**（不是指针），一次 zalloc 全包；各字段之间还插了 `char padN[ut::INNODB_CACHE_LINE_SIZE]` 做 **cache line 隔离**——这是 InnoDB 全局结构的常见手法，代价是结构体比字段之和大得多。

### 2.6 change buffer：几乎"没有"独立内存

ibuf 的**内存身份**：独立 malloc 的只有 `ibuf_t` 结构本身（KB 级，`ibuf_init_at_db_start`，ibuf0ibuf.cc:458-474）。**B-tree 本体、root page、各表空间的 bitmap 全部在 Buffer Pool 内**——`innodb_change_buffer_max_size`（默认 25，0-50）限制的是它在 **BP 内的占比**，不是额外分配内存。预算视角：change buffer 不是独立内存区，而是 BP 的一个"内部配额"。机制见 [ibuf.md](../../index/ibuf.md)。

**★ 源码证据链**（⚠️ 真名 `ibuf_init_at_db_start()`，**不存在 `ibuf_init()`**）：

```cpp
void ibuf_init_at_db_start(void) {
  ibuf = static_cast<ibuf_t *>(
      ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, sizeof(ibuf_t)));   // ★ 唯一一笔堆分配

  /* At startup we initialize ibuf to have a maximum of
  CHANGE_BUFFER_DEFAULT_SIZE in terms of percentage of the buffer pool size. */
  ibuf->max_size = ((buf_pool_get_curr_size() / UNIV_PAGE_SIZE) *
                    CHANGE_BUFFER_DEFAULT_SIZE) / 100;                // ★ 单位是"页数"
  ...
  block = buf_page_get(page_id_t(IBUF_SPACE_ID, FSP_IBUF_TREE_ROOT_PAGE_NO),
                       univ_page_size, RW_X_LATCH, UT_LOCATION_HERE, &mtr);
  //  ★★ B-tree root 是系统表空间的一页，通过 buf_page_get 从 BP 借 frame
  root = buf_block_get_frame(block);
  ...
  ibuf->index = dict_mem_index_create("innodb_change_buffer", "CLUST_IND", ...);
  ibuf->index->page = FSP_IBUF_TREE_ROOT_PAGE_NO;
}

void ibuf_max_size_update(ulint new_val) {
  ulint new_size = ((buf_pool_get_curr_size() / UNIV_PAGE_SIZE) * new_val) / 100;
  mutex_enter(&ibuf_mutex);
  ibuf->max_size = new_size;      // ★ 只是改一个数字，不分配任何内存
  mutex_exit(&ibuf_mutex);
}
```

> ★★ `innodb_change_buffer_max_size` 的 update 回调（`ha_innodb.cc:20729`）就调 `ibuf_max_size_update()`——**改这个参数不产生任何内存分配/释放**。它纯粹是"允许 ibuf 占 BP 的百分之几"的软配额。

```cpp
static MYSQL_SYSVAR_ULONG(
    change_buffer_max_size, srv_change_buffer_max_size, PLUGIN_VAR_RQCMDARG,
    "Maximum on-disk size of change buffer in terms of percentage"
    " of the buffer pool.",
    nullptr, innodb_change_buffer_max_size_update, CHANGE_BUFFER_DEFAULT_SIZE,
    0, 50, 0);                     // ★ 范围 [0,50]，默认 25
```

### 2.7 doublewrite

dblwr 的**内存身份**：每实例 `Double_write` 的 `m_buffer`/`m_buf_pages` 容量均为 `dblwr::n_pages`（默认 64）个物理页——**约 1MB/实例级别的小型 malloc 内存**（实例数 = `max(4, BP 实例数 × 2)`），dblwr 文件本体在磁盘。TPC 页面压缩的 zlib 上下文是瞬态每页 buffer，不常驻。写流程与批量结构见 [dblwr.md](../../innodb/dblwr.md)。

### 2.8 瞬态 mem_heap 的使用大户

InnoDB 内部临时计算全靠 mem_heap（Bump Allocator，算法见 allocator.md）。本仓库 `mem_heap_create` 调用点统计：`row0sel.cc`(17)、`row0vers.cc`(10)、`row0upd.cc`(9)、`row0mysql.cc`(9)、`fts0fts.cc`(9)、`btr0btr.cc`(8)、`row0log.cc`(7)、`ibuf0ibuf.cc`(7)……

要点：

1. **BUFFER 型 heap 的大块直接借 BP 页**（`mem_heap_create_block` → `buf_block_alloc`，分配分支机制见 [allocator.md](allocator.md)）——AHI、recv_sys（崩溃恢复）、lock heap 的大块分配**消耗 BP 页**。
2. **8.0 没有全局 mem pool**：`mem_comm_pool` 全库 0 匹配。5.7 已移除 InnoDB 自研内存池，8.0 全部落到 malloc / BP 两条路径。
3. **DDL 归并排序**在 `ddl/ddl0merge.cc`（`ddl::Merge_file_sort`；**8.0.30 起 `row0merge.cc` 已不存在**），排序 I/O buffer 按 `merge_io_buffer_size(n_buffers)` 每线程分配（ddl0merge.cc:482-489）——大表 DDL 的瞬态内存尖峰主要来自这里。
4. FTS 每表一个 doc cache（`fts_cache_create`，fts0fts.cc:526-536，`mem_heap_create(512)`），随表缓存淘汰释放。

### 2.9 其他常驻小块

| 对象 | 分配 | 量级 |
|------|------|------|
| `srv_sys` | `ut::zalloc_withkey(sizeof(srv_sys_t) + n_sys_threads × sizeof(srv_slot_t))`（srv0srv.cc:1157-1160） | KB~MB |
| purge_sys | `trx_purge_sys_mem_create`（trx0purge.cc:241-251） | KB 级 |
| os_event 大户 | redo 4096 个、每 BP 实例 `no_flush[]`、trx/lock timeout event | 每 event 约百字节 |
| Parallel_reader | 每线程小 mem_heap（row0pread.cc:559-560 等） | 线程数 × 页大小级 |

---

## 三、Server 层内存区

### 3.1 table cache / table definition cache

**⚠️ 这是最容易搞混的一对**——两个独立缓存，缓存的对象完全不同：

| | table cache | table_definition_cache |
|---|---|---|
| 缓存对象 | **`TABLE`**（已打开的表句柄，含 handler 实例、记录缓冲） | **`TABLE_SHARE`**（表定义/元数据，全局唯一） |
| 分片 | **16 个实例**（`Table_cache_manager::m_table_cache[]`，按 `thread_id % instances` 选） | **全局一份**，由 `LOCK_open` 保护 |
| 锁 | 每个实例自己的 `m_lock` | `LOCK_open` |
| 连接点 | `TABLE_SHARE::cache_element[]` 数组（每实例一个槽位） | — |

```cpp
class Table_cache_manager {
 public:
  static const int MAX_TABLE_CACHES = 64;              // 数组骨架
  static const int DEFAULT_MAX_TABLE_CACHES = 16;      // ★ 实际只用前 16 个
  Table_cache *get_cache(THD *thd) {
    return &m_table_cache[thd->thread_id() % table_cache_instances];
  }
 private:
  Table_cache m_table_cache[MAX_TABLE_CACHES];
};
```

```cpp
using Table_definition_cache =
    malloc_unordered_map<std::string,
                         std::unique_ptr<TABLE_SHARE, Table_share_deleter>>;
Table_definition_cache *table_def_cache;    // ★ 全局唯一；不存在 "tdc_hash" 这个类/变量
```

**★ `table_open_cache` 是软限制**（源码注释原文就写着 "soft limit"）：

```cpp
static bool fix_table_cache_size(sys_var *, THD *, enum_var_type) {
  /*
    table_open_cache parameter is a soft limit for total number of objects
    in all table cache instances. Once this value is updated we need to
    update value of a per-instance soft limit on table cache size.
  */
  table_cache_size_per_instance = table_cache_size / table_cache_instances;
  return false;
}
```

```cpp
void Table_cache::free_unused_tables_if_necessary(THD *thd) {
  if (m_table_count > table_cache_size_per_instance && m_unused_tables) {
    //                                                 ★★ m_unused_tables 为空就直接放弃回收
    mysql_mutex_lock(&LOCK_open);
    while (m_table_count > table_cache_size_per_instance && m_unused_tables) {
      TABLE *table_to_free = m_unused_tables;
      remove_table(table_to_free);
      intern_close_table(table_to_free);
      thd->status_var.table_open_cache_overflows++;    // ★ Table_open_cache_overflows
    }
    mysql_mutex_unlock(&LOCK_open);
  }
}
```

> ★★ **实战结论**：比较对象是"**每实例配额**"（`table_open_cache / 16`）而非 `table_open_cache`；且**正在使用的 TABLE 不计入回收**——所以实际打开数可以超过配置值，`Open_tables` 状态变量可能 > `table_open_cache`。

TDC 的淘汰同理（只淘汰"未被引用"的 share）：

```cpp
  /* Free cache if too big */
  while (table_def_cache->size() > table_def_size && oldest_unused_share->next)
    table_def_cache->erase(to_string(oldest_unused_share->table_cache_key));
```

> ⚠️ **`table_open_cache_instances` 是 READONLY**（必须重启）；`table_open_cache` / `table_definition_cache` 可在线改，且**两者都是 PFS 的 sizing hint**（`sys_var::PARSE_EARLY` 注释明说）——**改大会让 PFS 自身预分配变大**。

**内存归属**（总账上最容易混淆的一点）：**`TABLE` 对象在 THD 的 mem_root 上**（每次打开表分配，语句/连接结束释放）；**`TABLE_SHARE` 在 table def cache 内**（跨连接共享，常驻）。

- `table_open_cache`：默认 4000，运行时更新重算 `table_cache_size_per_instance`（sys_vars.cc:5122-5131）。
- `table_definition_cache`：默认 400，最小 400 最大 512K（sys_vars.cc:5081-5090）。
- `table_open_cache_instances`：1-64，默认 16（sys_vars.cc:5133-5146）。

官方口径：table def cache "不占文件描述符、比表缓存更省空间"，大表数量场景调大它能减少开表成本。分片结构与查找/淘汰机制见 [table.md](../../server/table.md)。

### 3.2 thread cache 与每连接固定开销

**★ thread cache 只缓存 OS 线程，**不缓存任何业务内存**——这是本节最大的误解**：

```cpp
class Per_thread_connection_handler : public Connection_handler {
  /**
    List of pending channel info objects to be picked by idle
    threads. Protected by LOCK_thread_cache.
  */
  static std::list<Channel_info *> *waiting_channel_info_list;
  //  ★★ 整个 thread cache 的全部"家当"就是一个 std::list<Channel_info*>
  //      Channel_info 只描述 socket/管道通道，不含 THD

  static ulong blocked_pthread_count;   // = 状态变量 Threads_cached
  static ulong max_blocked_pthreads;    // = thread_cache_size 本身（sysvar 直接绑它）
  Channel_info *block_until_new_connection();
};
```

**★ 决定性证据：`THD` 在进入 cache 之前就被 `delete` 了**（`connection_handler_per_thread.cc:245`）：

```cpp
  for (;;) {
    THD *thd = init_new_thd(channel_info);   // ★ 每轮（=每个新连接）都 NEW 一个 THD
    ...
    while (thd_connection_alive(thd)) { if (do_command(thd)) break; }
    end_connection(thd);
    close_connection(thd, 0, false, false);
    thd->release_resources();                // ★ 释放 THD 全部业务内存
    thd_manager->remove_thd(thd);
    delete thd;                              // ★★★ THD 在这里被销毁
    ...
    channel_info = Per_thread_connection_handler::block_until_new_connection();
    //  ★★★ 线程裸奔着进入 cache：没有 THD、没有 mem_root、没有 binlog cache
    if (channel_info == nullptr) break;
    pthread_reused = true;
```

```cpp
Channel_info *Per_thread_connection_handler::block_until_new_connection() {
  mysql_mutex_lock(&LOCK_thread_cache);
  if (blocked_pthread_count < max_blocked_pthreads && !shrink_cache) {
    blocked_pthread_count++;
    while (!connection_events_loop_aborted() && !wake_pthread && !shrink_cache)
      mysql_cond_wait(&COND_thread_cache, &LOCK_thread_cache);   // ★ 就睡在这里
    blocked_pthread_count--;
    ...
```

> ★★ **结论**：thread cache 对内存预算的意义**只有一个**——避免 `pthread_create` 与线程栈（默认 8 MB 虚拟）的创建销毁开销。**它不会让"每连接内存"被复用或省掉**。`Threads_cached` 高 ≠ 内存没释放。

```cpp
static Sys_var_ulong Sys_thread_cache_size(
    "thread_cache_size", "How many threads we should keep in a cache for reuse",
    GLOBAL_VAR(Per_thread_connection_handler::max_blocked_pthreads),
    CMD_LINE(REQUIRED_ARG, OPT_THREAD_CACHE_SIZE), VALID_RANGE(0, 16384),
    DEFAULT(0), BLOCK_SIZE(1), NO_MUTEX_GUARD, NOT_IN_BINLOG, nullptr,
    ON_UPDATE(modify_thread_cache_size));
//  ★ 默认 0（8.0 编译期写死，与 5.7 的"自动计算"不同）
```

**每连接固定开销的构成**（`THD::THD()`，`sql_class.cc:626`）：

| 项 | 时机 | 说明 |
|---|---|---|
| `THD` 对象 + `new LEX` + `Protocol_text/binary` + `Dictionary_client` + `Transaction_ctx` | 连接建立即分配 | 固定 |
| `main_mem_root`（PSI key `memory/sql/thd::main_mem_root`） | **懒分配**，块大小 `query_alloc_block_size`（8192） | **首次查询才 malloc** |
| `variables`（System_variables）+ `status_var` | 构造时 `memset` | 两块固定结构体，约 1-2 KB |
| `mdl_context` | `THD()` 里 `mdl_context.init(this)` | 含 ticket/hash，运行期增长 |
| `ha_data[handlerton]` | 构造时建数组，**内容懒创建** | binlog 的 `binlog_cache_mngr` 挂这里 |
| net buffer | 初始 `net_buffer_length`（16 KiB） | 可涨到 `max_allowed_packet` |

> ★ `memset(&variables, 0, sizeof(variables))` 之后由 `plugin_thdvar_init()` 填充——**每个连接一份 complete 的 system variables 副本**。

**binlog cache：是"初始 buffer"不是"固定占用"**：

```cpp
bool IO_CACHE_binlog_cache_storage::open(const char *dir, const char *prefix,
                                         my_off_t cache_size, my_off_t max_cache_size) {
  if (open_cached_file(&m_io_cache, dir, prefix, cache_size, MYF(MY_WME)))
    return true;                            // ★ 初始内存 buffer = binlog_cache_size
  m_max_cache_size = max_cache_size;
  m_io_cache.end_of_file = max_cache_size;  // ★★ IO_CACHE 的 end_of_file 就是内存上限
  return false;
}
bool Binlog_cache_storage::open(my_off_t cache_size, my_off_t max_cache_size) {
  const char *LOG_PREFIX = "ML";            // ★ 临时文件前缀，落在 mysql_tmpdir
  if (m_file.open(mysql_tmpdir, LOG_PREFIX, cache_size, max_cache_size)) return true;
```

> ★★ 一个 `binlog_cache_mngr` 内含**两个**完整 cache（`stmt_cache` + `trx_cache`，各内嵌一个 `IO_CACHE`）。所以**每连接峰值 ≈ `binlog_cache_size + binlog_stmt_cache_size`**（默认各 32 KiB），事务内可涨到 `max_binlog_cache_size`，再超就落盘 `ML*` 临时文件。
> ⚠️ **`max_binlog_cache_size` 默认 ≈ ULLONG_MAX（实际无限）**——这是最容易踩 OOM 的一个"无上限"变量。若 `binlog_cache_size > max_binlog_cache_size`，server 会强制 clamp（`check_binlog_cache_size()`）。
> ★ 额外增长项：`cache_state_map`（每个 SAVEPOINT 一条）、`m_pending`（未 flush 的 `Rows_log_event`，大事务时可达数十 MB）。

```cpp
class binlog_cache_mngr {
  bool init() {
    return stmt_cache.open(binlog_stmt_cache_size, max_binlog_stmt_cache_size) ||
           trx_cache.open(binlog_cache_size, max_binlog_cache_size);
  }
  binlog_stmt_cache_data stmt_cache;   // 非事务 cache
  binlog_trx_cache_data  trx_cache;    // 事务 cache
};
```

**sort buffer 是"上限"不是"固定值"**（`Filesort_buffer`，`filesort_utils.h:44`——**不在 `filesort.cc`**）：

```cpp
  Filesort_buffer() : m_next_rec_ptr(nullptr), m_current_block_end(nullptr),
                      m_max_size_in_bytes(0), m_current_block_size(0),
                      m_space_used_other_blocks(0) {}   // ★ 构造不分配任何内存
  void set_max_size(size_t max_size, size_t record_length) {
    m_max_size_in_bytes = max_size;      // ★★ 是 maximum，不是 fixed
  }
```

```cpp
bool Filesort_buffer::allocate_block(size_t num_bytes) {
  size_t next_block_size;
  if (m_current_block_size == 0) {
    next_block_size = MIN_SORT_MEMORY;   // ★ 首块固定 32 KiB，不是 sort_buffer_size
  } else {
    next_block_size = m_current_block_size + m_current_block_size / 2;   // ★ 之后每块 ×1.5
  }
  ...
  size_t space_left = (space_used > m_max_size_in_bytes)
                          ? 0 : m_max_size_in_bytes - space_used;
  next_block_size = min(max(next_block_size, num_bytes), space_left);   // ★ 受上限约束
  if (next_block_size < num_bytes) return true;   // 满了 → filesort 转落盘
  return allocate_sized_block(next_block_size);
}
```

> ★★ **所以 `sort_buffer_size=4M` 但只排 100 行，实际只吃 32 KiB**——这是"配了大 sort_buffer 但内存没涨"的原因。额外开销是 `m_record_pointers`（每行一个 8 字节指针，源码注释估算最坏 33%）。

`thread_cache_size`（sys_vars.cc:5161-5166）默认 0，自动默认 `8 + max_connections/100`（上限 100，mysqld.cc:5055-5070），控制 `Per_thread_connection_handler::max_blocked_pthreads`。**缓存线程保留其全部内存**（官方文档明示），所以 thread cache 不只是省线程创建开销，还意味着这部分内存不会随连接关闭归还系统。

THD 的内存核心是 `main_mem_root`（`sql/sql_class.h:4347`）：

```cpp
// sql/sql_class.cc:706-707
main_mem_root(key_memory_thd_main_mem_root,
              global_system_variables.query_alloc_block_size),
```

语句结束 `main_mem_root.Clear()`（sql_class.cc:1470）——即 THD mem_root 是**语句级复用**的 arena：解析、优化、执行产生的临时对象全在它上面，语句边界整体回收。这是 MEM_ROOT 在 server 层最大的使用场景。

### 3.3 hostname cache

**8.0.30+ 已无 `Host_cache` 类**，替换为 `hostname_cache`（`sql/hostname_cache.cc:193-197`）：`malloc_unordered_map`（按 IP 字符串）+ LRU list。`host_cache_size` 默认 128，自动默认 `128 + max_connections`（上限 2000，mysqld.cc:5059-5076）。

### 3.4 binlog cache

每连接**两份**：trx cache + stmt cache。**内存身份**：`binlog_cache_size` / `binlog_stmt_cache_size` 默认 32KB 起步，内存写满自动 spill 到磁盘临时文件（`Binlog_cache_storage::open(cache_size, max_cache_size)`）；`max_binlog_cache_size` / `max_binlog_stmt_cache_size` 默认约 16EB（不限制）。**大事务的内存尖峰**就是 trx cache：一个事务产生的 binlog 事件先堆在内存，超过 `max_binlog_cache_size` 才报 ER_TRANS_CACHE_FULL。写入/截断/压缩机制见 [binlog.md](../../server/replication/binlog.md)。

### 3.5 sort buffer / join buffer / read buffer

- sort buffer：`sort_buffer_size` 默认 256KB，**8.0 是"预算"而非"预分配"**——32KB 起 ×1.5 块式增长，小排序只付 32KB（机制见 [01_filesort.md](../../server/query/runtime/01_filesort.md)）。
- join buffer：**8.0 中 BNL/BKA join cache 已删除**；`join_buffer_size`（默认 256KB）现在是 **hash join build 端的内存上限**，超限落盘（机制见 [05_join_buffer.md](../../server/query/runtime/05_join_buffer.md)）。
- read buffer：`read_buffer_size` / `read_rnd_buffer_size`，官方口径"每次顺序扫描分配一个读缓冲"。

这些 buffer 的共性：**按连接数线性放大**，是内存预算里"每连接 × N 项"的主体。

### 3.6 TempTable 引擎

内部临时表（8.0 默认引擎 TempTable）用**独立于 malloc 的自有分配体系**：

- 参数：`temptable_max_ram`（默认 1GB）、`temptable_max_mmap`（默认 1GB）、`temptable_use_mmap`——**全局共享预算**（一个会话吃满 1GB，其他会话也被迫转 mmap/磁盘）。
- **池化复用**：全局共享块池，每连接借一个共享 Block、连接关闭归还——TempTable 的"每连接内存"不是纯 malloc。
- PFS 记账：`memory/temptable/physical_ram` / `physical_disk`。

分配策略（RAM → MMAP 两级降级、块粒度判超限）与临时表机制见 [07_temptable.md](../../server/query/runtime/07_temptable.md)「内存分配与多级阈值」。

### 3.7 PFS 自身：只增不减

官方文档明确：Performance Schema **动态增量分配、直到重启才释放**。参数全部 READONLY：`performance_schema_max_memory_classes`、`max_table_instances`、`max_digest_length`（默认 1024）、`max_statement_stack`。长期运行的内存"只涨不跌"，PFS 自身是常见贡献者之一——它记录的内存事件越多，占的统计内存越多。

### 3.8 其余 server 层内存

| 对象 | 位置 | 说明 |
|------|------|------|
| DD cache | `sql/dd/cache/` Shared_dictionary_cache | 8.0 权威字典缓存主体，与 InnoDB dict_sys 并存 |
| sp cache | `sql/sp_cache.cc:42-87`，每 THD 一份 | `stored_program_cache` 参数限制 |
| MDL map | `sql/mdl.cc:247-248` LF_HASH | `MDL_lock` 按需 malloc，空闲对象按低水位随机驱逐 |
| 线程栈 | `thread_stack`（mysqld.cc:7723-7732 设栈大小） | 每连接 1MB 级，**虚拟内存**占比大，物理页按需映射 |
| key buffer | `key_buffer_size` 默认 8MB（MyISAM） | 按需 mmap |

---

## 四、内存预算：总量怎么估算

官方文档（memory-use）**没有给出单一公式**，口径是组件累加。结合源码，可整理成三部分：

```
总内存 ≈ 全局常驻 + 每连接 × 有效连接数 + PFS 自身
```

**全局常驻（与连接数无关）**：

```
innodb_buffer_pool_size（含 chunk 描述符开销）
+ innodb_log_buffer_size + write-ahead 8KB + Link_buf 3MB + 4096 × event
+ dict_sys（随表数量增长）
+ lock_sys（≈ 1MB + 行锁动态部分）
+ trx_sys / srv_sys / purge_sys / dblwr（≈ 数 MB）
+ table_def_cache × TABLE_SHARE 平均大小（大表数量场景可观）
+ host_cache / MDL / sp_cache / TempTable 全局池
```

**每连接（× 并发连接数）**：

```
thread_stack（虚拟内存 1MB+，物理按需）
+ THD 对象 + main_mem_root 当前块
+ 2 × net_buffer_length（可涨至 max_allowed_packet）
+ sort_buffer_size + join_buffer_size（hash join）
+ read_buffer_size + read_rnd_buffer_size（按需）
+ binlog_cache_size + binlog_stmt_cache_size（大事务可涨到 max_binlog_cache_size）
+ max_digest_length（每会话）
+ TempTable 共享块（池化，按并发需求增长）
```

**要点**：

1. **最容易乘出的部分是每连接**：官方口径提示 `max_connections × (sort + read + read_rnd + join + stack + binlog + ~2MB)`。一台 `max_connections=1000` 的机器，即使全部是默认值，每连接开销也能达到 ~10MB 虚拟/数 MB 物理，千连接就是 GB 级。
2. **thread cache 保留内存**：连接归还 thread cache 后不释放，实际并发低于峰值时内存不回落。
3. **PFS 只增不减**：长期运行的实例，PFS 统计内存随事件种类增长。
4. **resize 的瞬时双份**：BP 在线 resize 时 page_hash 有 2× 新大小的临时表；dict/lock sys 重建同理。预算评估要按峰值（resize 期间）算。
5. **虚拟内存与 RSS 的差异**：线程栈、BP（延迟物理映射）都是虚拟内存大头。官方提示 `ps` 会虚高。

---

## 五、OOM 排查路线图（PFS 观测）

### 5.1 数据入口

所有内存分配的记账入口是 `performance_schema.memory_summary_global_by_event_name`（实现 `storage/perfschema/table_mems_global_by_event_name.cc`）。sys 库提供聚合视图（`scripts/sys_schema/views/p_s/`）：`memory_global_by_current_bytes`、`memory_global_total`、`memory_by_user_by_current_bytes`、`memory_by_host_by_current_bytes`、`memory_by_thread_by_current_bytes`。

**⚠️ 三个命名勘误**：类名是 **`table_mems_*`**（不是 `table_memory_summary_*`）；统计结构**不存在 `PFS_memory_stat`**，实为三套；`pfs_memory.cc` 里**没有** `*_vc` 函数（全在 `pfs.cc`）。

```cpp
struct PFS_memory_safe_stat {      // 每线程桶，非原子
  bool m_used;
  size_t m_alloc_count;            // → COUNT_ALLOC
  size_t m_free_count;             // → COUNT_FREE
  size_t m_alloc_size;             // → SUM_NUMBER_OF_BYTES_ALLOC
  size_t m_free_size;              // → SUM_NUMBER_OF_BYTES_FREE
  size_t m_alloc_count_capacity;   // ★ 「离历史最高还差多少」
  size_t m_free_count_capacity;    // ★ 「离历史最低还差多少」
  size_t m_alloc_size_capacity;
  size_t m_free_size_capacity;
  ...
};
struct PFS_memory_shared_stat {    // 全局/account/user/host 桶，成员全 std::atomic
  std::atomic<bool> m_used;  std::atomic<size_t> m_alloc_count; ...
};
struct PFS_memory_monitoring_stat {// ★ 只在「读表那一刻」构造，用于归一化输出 10 列
  size_t m_alloc_count, m_free_count, m_alloc_size, m_free_size;
  size_t m_alloc_count_capacity, ...;
  size_t m_missing_free_count_capacity, m_missing_free_size_capacity;
  ssize_t m_low_count_used, m_high_count_used;   // ★ 有符号
  ssize_t m_low_size_used, m_high_size_used;
  void normalize(bool global);
};
```

**源码里的不变量**（`pfs_stat.h:894` 注释原文）：

```
CURRENT_COUNT_USED = m_alloc_count - m_free_count
LOW_COUNT_USED  + m_free_count_capacity  = CURRENT_COUNT_USED
CURRENT_COUNT_USED + m_alloc_count_capacity = HIGH_COUNT_USED
CURRENT_SIZE_USED = m_alloc_size - m_free_size
LOW_SIZE_USED  + m_free_size_capacity  = CURRENT_SIZE_USED
CURRENT_SIZE_USED + m_alloc_size_capacity = HIGH_SIZE_USED
```

**★ 最终算式**（`PFS_memory_monitoring_stat::normalize`，`pfs_stat.cc:421`）：

```cpp
  const ssize_t current_count = m_alloc_count - m_free_count;   // CURRENT_COUNT_USED
  m_low_count_used  = current_count - m_free_count_capacity;    // LOW_COUNT_USED
  m_high_count_used = current_count + m_alloc_count_capacity;   // HIGH_COUNT_USED
  const ssize_t current_size = m_alloc_size - m_free_size;      // ★ CURRENT_NUMBER_OF_BYTES_USED
  m_low_size_used   = current_size - m_free_size_capacity;
  m_high_size_used  = current_size + m_alloc_size_capacity;
  if (global) {                     // ★ 只有 global 表把低水位夹到 0
    if (m_low_count_used < 0) m_low_count_used = 0;
    if (m_low_size_used  < 0) m_low_size_used  = 0;
  }
```

> ★★ **实战结论**：`CURRENT_NUMBER_OF_BYTES_USED = m_alloc_size - m_free_size` 就是"当前占用内存"，`sys.memory_global_total` 求的就是这一列的和。**水位列（LOW/HIGH）是"capacity 记账"的近似，不是逐时刻采样**——`HIGH_*` 单调不减、`LOW_*` 单调不增，都依赖"父桶余量冒泡"是否及时。

**★★ 五道闸门：任何一道关闭，分配就完全不进表**（`pfs.cc:8319`）：

```cpp
PSI_memory_key pfs_memory_alloc_vc(PSI_memory_key key, size_t size, PSI_thread **owner) {
  if (!flag_global_instrumentation) { *owner_thread = nullptr; return PSI_NOT_INSTRUMENTED; }   // ① 全局 PFS 关
  PFS_memory_class *klass = find_memory_class(key);
  if (klass == nullptr)              { ... return PSI_NOT_INSTRUMENTED; }                       // ② key 未注册
  if (!klass->m_enabled)             { ... return PSI_NOT_INSTRUMENTED; }                       // ③ instrument 未 ENABLED
  if (flag_thread_instrumentation && !klass->is_global()) {
    PFS_thread *pfs_thread = my_thread_get_THR_PFS();
    if (unlikely(pfs_thread == nullptr)) { ... return PSI_NOT_INSTRUMENTED; }                   // ④ 非 PFS 线程
    if (!pfs_thread->m_enabled)          { ... return PSI_NOT_INSTRUMENTED; }                   // ⑤ 该线程 instrumentation 关
    ... /* 进 by_thread 桶，高水位增量向 account/user/host 冒泡 */
  } else {
    stat->count_global_alloc(size);   // ★ 只进全局桶
  }
}
```

> ★ `is_global()` = `m_flags & PSI_FLAG_ONLY_GLOBAL_STAT` —— **带此 flag 的 instrument（如 `memory/innodb/buf_buf_pool`）在 by_thread/account/host/user 表里必然看不到**。

**还有三个"统计会撒谎"的源码证据**：

```cpp
// ① 未埋点：MyISAM 排序缓冲直接用 PSI_NOT_INSTRUMENTED（sort.cc:163）
    if ((sort_keys = (uchar **)my_malloc(PSI_NOT_INSTRUMENTED,
// ② 同一块内存被另一线程接管 → 净用量被强行夹到 0（pfs_stat.h:1089）
  void count_free(size_t size) {
    if (m_size >= size) m_size -= size;
    else m_size = 0;   // ★ "Thread 2 releases more memory that it is supposed to own"
  }
// ③ 线程断开把"净贡献"直接塞进全局桶，导致跨桶水位失衡（pfs_stat.cc:707 注释原文）
      - resulting in unbalanced alloc / free watermarks, globally.
```

**5 张表的 DDL 是硬编码在源码里的**（`table_mems_*.cc`），全部无 NULL 列；`COUNT_ALLOC/COUNT_FREE/SUM_*` 是 `BIGINT unsigned`（只增），`CURRENT_*/LOW_*/HIGH_*` 是 `BIGINT`（**有符号**，可为负）：

```cpp
Plugin_table table_mems_global_by_event_name::m_table_def(
    "performance_schema",
    "memory_summary_global_by_event_name",
    "  EVENT_NAME VARCHAR(128) not null,\n"
    "  COUNT_ALLOC BIGINT unsigned not null,\n"
    "  COUNT_FREE BIGINT unsigned not null,\n"
    "  SUM_NUMBER_OF_BYTES_ALLOC BIGINT unsigned not null,\n"
    "  SUM_NUMBER_OF_BYTES_FREE BIGINT unsigned not null,\n"
    "  LOW_COUNT_USED BIGINT not null,\n"
    "  CURRENT_COUNT_USED BIGINT not null,\n"
    "  HIGH_COUNT_USED BIGINT not null,\n"
    "  LOW_NUMBER_OF_BYTES_USED BIGINT not null,\n"
    "  CURRENT_NUMBER_OF_BYTES_USED BIGINT not null,\n"
    "  HIGH_NUMBER_OF_BYTES_USED BIGINT not null,\n"
    "  PRIMARY KEY (EVENT_NAME)\n",
    " ENGINE=PERFORMANCE_SCHEMA", nullptr);
```

> ★ `TRUNCATE` global 表会顺带 reset **全部 5 张**内存表（`delete_all_rows()` 调 4 个 `reset_memory_by_*` + `reset_memory_global()`），且用的是 **`rebase()` 而非 `reset()`**——保留"当前净用量"，只清掉水位余量：

```cpp
void PFS_memory_safe_stat::rebase() {
  if (!m_used) return;
  size_t base = std::min<size_t>(m_alloc_count, m_free_count);
  m_alloc_count -= base;  m_free_count -= base;
  base = std::min<size_t>(m_alloc_size, m_free_size);
  m_alloc_size -= base;   m_free_size -= base;
  m_alloc_count_capacity = 0; m_free_count_capacity = 0;
  m_alloc_size_capacity = 0;  m_free_size_capacity = 0;
}
```

### 5.2 关键 event 清单（按内存量级排序的典型场景）

| event | 对应结构 | 说明 |
|-------|----------|------|
| `memory/innodb/buf_buf_pool` | BP chunk | 唯一入口，`PSI_FLAG_ONLY_GLOBAL_STAT`，恒等于 BP 配置量 |
| `memory/innodb/log0log` | log buffer | 独立于 BP 的 redo 缓冲 |
| `memory/innodb/mem0mem` | mem_heap 各块 | 瞬态分配的主入口 |
| `memory/innodb/btr0sea` | AHI hash cell | node 部分体现在 BP 免费页减少上 |
| `memory/innodb/dict0dict` | 字典对象 | 随表数量增长 |
| `memory/innodb/lock0lock` | 锁对象 | 随并发事务增长 |
| `memory/sql/THD::main_mem_root` | 每连接语句内存 | 连接数放大器 |
| `memory/sql/TABLE` / `TABLE_SHARE` | 表缓存 | 随打开表增长 |
| `memory/temptable/physical_ram` / `physical_disk` | TempTable | 大 GROUP BY/UNION 的内存/落盘 |
| `memory/sql/Filesort_buffer` 等 | sort/join | 重排序 SQL 尖峰 |

InnoDB 的 key 注册表在 `ut/ut0new.cc:47-89`（`pfs_info[]`），声明清单在 `include/ut0new.h:210-230`；server 层 key 在 `sql/psi_memory_key.h`（100+ 个）。

### 5.3 排查顺序

1. `SELECT * FROM sys.memory_global_by_current_bytes;`——找出当前 top 内存区。
2. 若 top 是每连接类（THD/sort/binlog），对比 `Threads_connected` 判断是"连接数多"还是"单连接吃得多"；后者查 `memory_by_thread_by_current_bytes` 定位具体线程 → 关联 `processlist` 找 SQL。
3. 若 top 是 mem_heap / ddl，找正在跑的 DDL 或大查询（`ddl0merge` 归并尖峰）。
4. 若全部 event 加起来 << RSS，检查**未插桩部分**：线程栈（虚拟内存）、OS 大页、TempTable mmap、MyISAM key buffer——它们不都在 memory_summary 里。
5. 排除 PFS 自身只增不减的假象：对比 `memory_summary_*` 总量与 `ps` RSS 的差值。

### 5.4 官方给出的"释放"途径

- `FLUSH TABLES`：关闭未用表，释放 table cache 内存（在用表等线程结束）。
- `FLUSH PRIVILEGES`：释放 GRANT/USER 语句的缓存（REVOKE 本身不释放）。

---

## 六、每块内存落到哪个分配器：与分配器矩阵的关系

架构视角与分配器视角的交叉点（详细算法见 allocator.md）：

| 内存区 | 分配路径 | 分配器 |
|--------|----------|--------|
| BP chunk | `ut::malloc_large_page_withkey`（大页→回退）| mmap（OS） |
| page_hash cell / AHI cell / dict hash cell | `ib_create` → malloc | jemalloc/glibc |
| log buffer | `aligned_array_pointer::alloc_withkey` | jemalloc/glibc |
| mem_heap 小块（<8KB） | `ut::malloc_withkey` | jemalloc/glibc |
| mem_heap 大块（BUFFER 型）| `buf_block_alloc` → `block = frame` | **Buffer Pool 免费页** |
| MEM_ROOT（THD/解析器）| `my_malloc` | jemalloc/glibc |
| TempTable | 自有 RAM/MMAP 分配器 | 自研（非标准 malloc） |
| 压缩页 frame | `buf_buddy_alloc` → `zip_free[]` | Binary Buddy（BP 内） |
| sort/join buffer | THD mem_root / my_malloc | MEM_ROOT → jemalloc |
| 线程栈 | pthread 栈 | mmap |

**关键结论**：MySQL 内存的"底"只有三条——**OS 堆（jemalloc）、mmap 匿名映射、Buffer Pool 免费页**。mem_heap 的 BUFFER 模式把后两者打通（临时大块借 BP 页），这是 InnoDB 内存架构最独特的一环：**BP 既是数据缓存，也是临时内存的供应池**。

**分配原语的真名**（⚠️ 三个"常见名"都不存在）：

| 常见说法 | 8.0.39 真名 |
|---|---|
| `os_mem_alloc_large()` | **已不存在**（只在 `os0proc.cc:47` / `os0proc.h:44` 的注释里残留）→ `ut::malloc_large_page_withkey()` |
| `buf_pool_init` 里有"大页失败回退 if 分支" | **没有**——回退被封装进三参重载的一行三目里 |
| `ut::malloc_withkey` 返回失败会抛异常 | **不会**——它是 `noexcept` 返回 `nullptr`；抛异常的是上层 `ut::new_*` / `ut::allocator` |

```cpp
// ① 大页（带回退）—— BP 走这条
inline void *malloc_large_page_withkey(PSI_memory_key_t key, std::size_t size,
                                       fallback_to_normal_page_t,
                                       bool large_pages_enabled = os_use_large_pages) noexcept {
  void *large_page_mem = nullptr;
  if (large_pages_enabled) large_page_mem = malloc_large_page_withkey(key, size);
  return large_page_mem ? large_page_mem : malloc_page_withkey(key, size);
}
// ② 普通（PFS 记账版）
inline void *malloc_withkey(PSI_memory_key_t key, std::size_t size) noexcept {
  using impl = detail::select_malloc_impl_t<WITH_PFS_MEMORY, false>;
  using malloc_impl = detail::Alloc_<impl>;
  return malloc_impl::alloc<false>(size, key());     // pfs=on → Allocpfs::alloc
}
// ③ 普通（零初始化）
inline void *zalloc_withkey(PSI_memory_key_t key, std::size_t size) noexcept {
  ... return malloc_impl::alloc<true>(size, key());
}
```

```cpp
// 回退路径：page_aligned_alloc（Linux = mmap，Windows = VirtualAlloc）
inline void *page_aligned_alloc(size_t n_bytes) {
  void *ptr = mmap(nullptr, n_bytes, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANON, -1, 0);
  if (unlikely(ptr == (void *)-1)) { ib::log_warn(...); }
  return (ptr != (void *)-1) ? ptr : nullptr;
}
```

**★ 内存分配失败的行为：InnoDB 与 server 层不一致**（这是 OOM 排查的关键差异）：

```cpp
// server 层 my_malloc：返回 nullptr + 报错（只有 MY_FAE 才 exit(1)）
  if (point == nullptr) {
    set_my_errno(errno);
    if (my_flags & MY_FAE) error_handler_hook = my_message_stderr;
    if (my_flags & (MY_FAE + MY_WME))
      my_error(EE_OUTOFMEMORY, MYF(ME_ERRORLOG + ME_FATALERROR), size);
    if (my_flags & MY_FAE) exit(1);
  }
  return (point);
```

```cpp
// InnoDB ut::allocator：抛 C++ 异常
  pointer allocate(size_type n_elements, const_pointer hint = nullptr) {
    if (unlikely(n_elements > max_size())) throw std::bad_array_new_length();
    auto ptr = Allocator_base::allocate_impl(n_elements * sizeof(T));
    if (unlikely(!ptr)) throw std::bad_alloc();      // ★★ 抛异常，不返回 NULL
    return static_cast<pointer>(ptr);
  }
```

> ⚠️ **"InnoDB 分配失败会 sleep 重试 60 次"已成历史**——`alloc_max_retries = 60`（`ut0new.cc:43`）**只有定义与 extern 声明，全仓库无任何使用点**。
> ★ 进 error log 时 `EE_OUTOFMEMORY` 被改写成 **`ER_SERVER_OUT_OF_RESOURCES`**（`sql/mysqld.cc:3896`）。
> ⚠️ **不存在 `my_malloc_error_handler` / `my_error_register`**（`mysqld.cc` 检索 0 命中）；**不存在 `oom_score_adj` 自调节**（全仓 0 命中）——要靠 systemd `OOMScoreAdjust=` 或 cgroup。

---

## 七、设计哲学（内存架构部分）

1. **常驻不归还**：BP 运行期不收缩、PFS 只增不减、thread cache 保留内存——数据库内存的第一原则是"稳定基线优先"，避免频繁向 OS 归还/再申请的系统调用与页表抖动。代价是灵活性差，适合长驻服务进程。
2. **延迟物理映射**：BP chunk 用 mmap，物理页在首次 touch 时才映射（内核月报 2021/01 指出）。启动快、冷区不占物理页。
3. **池化复用**：mem_heap/MEM_ROOT 大块切分（减少 malloc 调用）、TempTable 共享块池（16K 块全局复用）、trx_t 池化——高频小对象不直接进出 OS 堆。
4. **分片降竞争**：BP 多实例（1GB 阈值）、page_hash 16 锁、table cache 16 实例、AHI 8 分区——内存结构的锁分片与内存分区同步演进。
5. **借页而非买页**：AHI、recv_sys 从 BP 免费页借内存而非 malloc——内存总量封顶，不会出现"AHI 无限膨胀吃光系统"。
6. **可观测性前置**：所有分配点带 PSI key（ut0new.cc 注册表），内存问题可从 SQL 层定位到模块——这是 8.0 相对 5.7 最重要的内存工程能力。

---

## 八、版本演进与常见误区澄清

### 8.1 版本演进

- **5.6 之前**：InnoDB 自研内存分配器 + 额外内存池（glibc 当时的分配器多线程性能差），`innodb_use_sys_malloc` / `innodb_additional_mem_pool_size` 控制。5.6 弃用，5.7 删除（内核月报 2021/01 追溯）。
- **8.0**：全部落到系统 malloc + BP 两条路；PFS 内存埋点成为标准观测手段；`mem_comm_pool` 等旧结构彻底移除。
- **8.0.30+**：DDL 归并排序迁至 `ddl/ddl0merge.cc`；Server 层 `Host_cache` 被 `hostname_cache` 取代；BNL/BKA join cache 删除，`join_buffer_size` 转为 hash join 上限。

### 8.2 常见误区（对照本仓库 8.0.39 源码核实）

| 误区 | 事实 |
|------|------|
| "8.0 的 `innodb_buffer_pool_size` 有 5GB 上限" | **无**。上限 `LLONG_MAX`；5MB 是最小值，1GB 是实例数自动配置阈值（`srv0start.h:65-67`） |
| "`innodb_log_buffer_size` 最大 4096(4K 或 4M)" | **无此限制**。上限 `UINT32_MAX`（≈4GB，`log0constants.h:484-491`） |
| "InnoDB 有全局 mem pool" | 8.0 **没有**。`mem_comm_pool` 全库 0 匹配，5.7 已移除 |
| "DDL 排序在 row0merge.cc" | 8.0.30 起在 `ddl/ddl0merge.cc`（`ddl::Merge_file_sort`） |
| "AHI 是独立 malloc 的内存区" | hash cell 是 malloc，**node 从 BP 免费页借**（MEM_HEAP_FOR_BTR_SEARCH） |
| "change buffer 是独立内存" | 树/bitmap 全在 BP 内，`innodb_change_buffer_max_size` 是 BP 内配额 |
| "join_buffer_size 控制 BNL join cache" | 8.0 已删除 BNL/BKA cache，该参数是 hash join 内存上限 |
| "内存涨上去就不回来 = 泄漏" | 多为常驻区 + PFS 只增不减 + thread cache 保留的设计行为 |

---

## 参考

**源码（本仓库 8.0.39）**
- `storage/innobase/buf/buf0buf.cc`（chunk 分配 / resize）、`include/buf0buf.h` / `buf0buf.ic`（buf_pool_t / buf_chunk_t）
- `storage/innobase/log/log0log.cc`（log_sys_create / log_allocate_buffer）、`include/log0sys.h`、`include/log0constants.h`
- `storage/innobase/dict/dict0dict.cc`（dict_init）、`storage/innobase/lock/lock0lock.cc`（lock_sys_create）
- `storage/innobase/trx/trx0sys.cc`、`trx0trx.cc`、`trx0purge.cc`
- `storage/innobase/ibuf/ibuf0ibuf.cc`、`storage/innobase/buf/buf0dblwr.cc`、`storage/innobase/btr/btr0sea.cc`
- `storage/innobase/ut/ut0new.cc`（PFS key 注册表）、`include/ut0new.h`、`mem/memory.cc`
- `storage/innobase/ddl/ddl0merge.cc`（DDL 归并排序）
- `sql/table_cache.h`、`sql/sql_class.cc`、`sql/hostname_cache.cc`、`sql/binlog_ostream.h`、`sql/filesort.cc`、`sql/iterators/hash_join_buffer.cc`、`sql/mdl.cc`、`sql/sp_cache.cc`、`sql/sys_vars.cc`、`sql/mysqld.cc`
- `storage/temptable/include/temptable/allocator.h`、`constants.h`、`src/handler.cc`、`src/block.cc`
- `storage/perfschema/table_mems_global_by_event_name.cc`、`scripts/sys_schema/views/p_s/`

**外部材料**
- MySQL 8.0 Reference Manual: [10.12.3.1 How MySQL Uses Memory](https://dev.mysql.com/doc/refman/8.0/en/memory-use.html)
- 数据库内核月报（阿里）：尚灿芳《MySQL · 源码阅读 · Innodb 内存管理解析》2021/01/06，<http://mysql.taobao.org/monthly/2021/01/06/>
