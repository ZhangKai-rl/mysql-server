# InnoDB doublewrite（dblwr）深度解析

> 基于 MySQL 8.0.39 源码，核心文件 `storage/innobase/buf/buf0dblwr.cc`（3640 行）+ `include/buf0dblwr.h`（552 行）。涵盖：**为什么必须有**（torn page 与 redo 的能力边界）、**文件布局**（无文件头的扁平页数组、批量区与 SYNC 槽位、奇偶文件功能切分）、**批量与单页两条写路径的完整源码流程**、**崩溃恢复时如何用它修页**、**加密帧为什么单独存在**、**O_DIRECT_NO_FSYNC 下哪些 fsync 被跳过**、**参数与监控**、**已知问题与 TODO**。

> **边界**：本篇讲 **doublewrite 本身**。刷脏（page cleaner、flush list、邻接刷盘、io_capacity 自适应）见 [`buffer_pool.md`](buffer_pool.md)；redo 的**写/刷路径**与格式见 [`redo_log.md`](redo_log.md)；dblwr 之下的 `os_file` 原语与 AIO 见 [`io.md`](io.md)；表空间元数据与 I/O 分发见 [`fil.md`](fil.md)；崩溃恢复整体见 [`recovery.md`](recovery.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [整体类结构](#整体类结构)
    - [文件布局](#文件布局)
  - 写路径
    - [批量写完整流程](#批量写完整流程)
    - [同步单页 flush](#同步单页-flush)
    - [`write_to_datafile`](#write_to_datafile)
    - [加密帧 `get_encrypted_frame`](#加密帧-get_encrypted_frame)
  - 崩溃恢复
    - [崩溃恢复时如何使用 dblwr](#崩溃恢复时如何使用-dblwr)
  - 工程细节
    - [O_DIRECT 与 fsync 的完整交互](#o_direct-与-fsync-的完整交互)
    - [`force_flush` 的四个调用点](#force_flush-的四个调用点)
  - 治理
    - [监控与状态变量](#监控与状态变量)
    - [reduced 模式（DETECT_ONLY）](#reduced-模式detect_only)
- [相关的系统变量](#相关的系统变量)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

doublewrite 是 InnoDB 防 **torn page（半页写）** 的机制：脏页在写到真实位置之前，**先整页顺序写到一个专门的文件并 fsync**，确认这份"完整副本"落盘后，才把页写到数据文件的真实位置。崩溃恢复时若发现数据文件里的页损坏，就从 dblwr 副本覆盖回去。

一句话：**用一次额外的写，换取"页本身是完整的"这个前提**。

### 用途

| 场景 | dblwr 的角色 |
|------|-------------|
| 正常刷脏 | 每个脏页物理上被写**两次**（dblwr 一次 + 数据文件一次） |
| 崩溃恢复 | 数据文件里的页 checksum 损坏 → 从 dblwr 副本恢复 |
| 文件系统/设备不保证原子写 | **唯一的防线** |

### 版本演进

| 版本 | 变化 |
|------|------|
| 4.1/5.0 | dblwr 位于**系统表空间内**的固定区域（2 个 extent，各 64 页），与系统表空间共享文件 |
| 5.6/5.7 | 增加 `innodb_doublewrite_dir` 可换目录（5.7）；仍是系统表空间内的固定区 |
| **8.0.20** | **★ 重构为独立文件**：`#ib_<page_size>_<id>.dblwr`，多实例、多文件，与系统表空间解耦 |
| 8.0.30+ | 引入 **reduced 模式**（`DETECT_ONLY`）——只记 `(space_id, page_no, lsn)` 三元组，写入量大幅减少，但**只能检测不能修复** |

> **为什么演进**：8.0.20 的独立文件化是为了①让 dblwr 可以放到更快的设备（`innodb_doublewrite_dir`）；②解除对系统表空间固定区的竞争（truncate 表空间时会争用双写区）；③支持多实例并发写。

---

## 理论基础

### 设计思想与权衡

#### 一、为什么必须有：torn page 与 redo 的能力边界

InnoDB 页 16KB，而设备/文件系统的原子写单位通常是 **4KB（甚至 512B）**。一次 16KB 写在中途断电，可能前 8KB 是新的、后 8KB 是旧的——**页撕裂**。

**★ 关键：redo 修不了这种损坏。**

| | 能修什么 | 为什么修不了 torn page |
|---|---|---|
| **redo** | 页"逻辑内容"的丢失（缺一次 UPDATE） | redo 是**页内偏移 + 内容**的物理日志，它假设"页的其他部分是对的"。在撕裂的页上重放 redo，等于在错误的基底上打补丁 |
| **dblwr** | 页"物理完整性" | 保存了整页的完整副本，直接覆盖 |

> 这就是 [`redo_log.md`](redo_log.md) 里"redo 防不了半页写"的完整论证。两者是**分工**关系，不是替代：redo 管"内容对不对"，dblwr 管"页本身完不完整"。

#### 二、代价清单（权衡的另一面）

| 代价 | 说明 |
|------|------|
| **写放大 2×** | 每个脏页物理写两次 |
| **fsync 次数增加** | dblwr 文件写完要 fsync（非 O_DIRECT 时），数据文件写完还要 fsync |
| **同步阻塞** | `Segment::write` 是**同步** `os_file_write_retry`，不走 AIO |
| **段池竞争** | 批量段是无锁队列，取不到就 `yield` 自旋；上一批未完成时新批要等 event |

> 正因为写放大，`DETECT_ONLY`（reduced）模式才存在——它把"写入 dblwr"的量从整页降到 16 字节/页。

#### 三、什么时候可以关掉

dblwr 的存在源于**块设备契约不保证多块写原子性**。若设备提供了 16KB 原子写，dblwr 就可以关：

```cpp
// storage/innobase/fsp/fsp0sysspace.cc
    if (fil_fusionio_enable_atomic_write(it->m_handle)) {
      if (dblwr::is_enabled) {
        ib::info(ER_IB_MSG_456) << "FusionIO atomic IO enabled,"
                                   " disabling the double write buffer";
        dblwr::g_mode = dblwr::Mode::OFF;
      }
      it->m_atomic_write = true;
    }
```

`fil_fusionio_enable_atomic_write`（`fil0fil.cc`）**要求 `O_DIRECT`** 且通过 ioctl 设置。

> ★ **注意：原子写与 dblwr 是互斥的**——`fil0fil.cc` 只在 `!dblwr::is_enabled` 时才尝试启用原子写。社区版默认不敢依赖设备原子写，因为通用块存储契约里没有这一条（详见 [`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)「块语义的最小契约」）。

#### 四、什么情况下自动跳过 dblwr

`dblwr::write` 的跳过条件（`buf0dblwr.cc`）：

```cpp
  if (srv_read_only_mode || fsp_is_system_temporary(space_id) ||
      !dblwr::is_enabled || Double_write::s_instances == nullptr ||
      mtr_t::s_logging.dblwr_disabled) {
    /* Skip the double-write buffer since it is not needed. Temporary
    tablespaces are never recovered, therefore we don't care about
    torn writes. */
```

| 条件 | 理由 |
|------|------|
| 只读模式 | 根本不写 |
| **临时表空间** | **临时表空间从不参与崩溃恢复**，撕裂也无所谓 |
| `innodb_doublewrite=OFF` | 用户显式关闭 |
| **redo logging 被禁用/受限**（`mtr_t::s_logging.dblwr_disabled`） | 无法恢复，写了也没意义 |

### 理论溯源

| 理论 | 落点 |
|------|------|
| **ARIES 的 "no-force / steal" + WAL** | redo 负责内容一致性；但 ARIES **假设页写入是原子的**（"write of a page is atomic"在某些表述中作为假设）。真实设备不满足 → dblwr 是**补足这个假设**的工程手段 |
| **端到端论证（Saltzer et al.）** | "完整性检查必须在最终使用点做"——页 checksum 在数据文件侧校验，损坏才回退到 dblwr |
| **副本冗余（可靠的单一副本 vs 两份副本）** | dblwr 本质是"先写可靠副本、再写目标"的两阶段提交思想 |

---

## 核心实现

### 整体类结构

`buf0dblwr.cc` 中定义的类（共 9 个）：

| 行号 | 类 | 职责 |
|---|---|---|
| 62 (h) | `dblwr::Buffer` | 页对齐的写缓冲（`ut::aligned_zalloc`，Windows AIO 要求对齐） |
| 141 | `dblwr::File` | 一个 `.dblwr` 文件：`m_id` / `m_name` / `m_pfs`，静态 `s_n_pages` |
| 204 | `dblwr::recv::Page` | 恢复时从 dblwr 文件读出的一个页 |
| 234 | `dblwr::recv::Page_entry` | reduced 模式恢复条目 `(space_id, page_no, lsn)` |
| 249 | `dblwr::recv::Pages` | 恢复期容器 |
| **430** | **`Double_write`** | **dblwr 实例**（每个 BP 实例 + 每种 flush 类型各一个） |
| **949** | **`Segment`** | **dblwr 文件内一段连续物理区间**（单页 flush slot 就是一个 Segment） |
| **1013** | **`Batch_segment : Segment`** | 批量写段，带 batch id / 引用计数 |
| **1143** | **`Reduced_double_write : Double_write`** | DETECT_ONLY 模式实现 |

关系：

```
dblwr::File  (1..n 个 .dblwr 文件)
   ▲ m_file
Segment ──────────► 文件内容中的 [m_start, m_end) 字节区间
   ▲ 继承
Batch_segment  ── m_dblwr ──► Double_write (实例)
                                  ▲ 继承
                          Reduced_double_write

Double_write::s_instances  (std::vector<Double_write*>*)
Double_write::s_files      (std::vector<dblwr::File>)
Double_write::s_LRU_batch_segments / s_flush_list_batch_segments  (mpmc_bq<Batch_segment*>*)
Double_write::s_single_segments   (mpmc_bq<Segment*>*)     ← 512 个单页槽位
```

#### `Segment`：文件内的一段区间

`buf0dblwr.cc`：

```cpp
class Segment {
 public:
  Segment(dblwr::File &file, page_no_t start, uint32_t n_pages)
      : m_file(file),
        m_phy_size(univ_page_size.physical),
        m_start(start * m_phy_size),
        m_end(m_start + (n_pages * m_phy_size)) {}

  virtual ~Segment = default;

  void write(const void *ptr, uint32_t len) noexcept {
    ut_a(len <= m_end - m_start);
    IORequest req(IORequest::WRITE | IORequest::DO_NOT_WAKE);
    req.dblwr;

    auto err = os_file_write_retry(req, m_file.m_name.c_str, m_file.m_pfs,
                                   ptr, m_start, len);
    ut_a(err == DB_SUCCESS);
  }

  /** Flush the segment to disk. */
  void flush noexcept { os_file_flush(m_file.m_pfs); }

  dblwr::File &m_file;
  uint32_t m_phy_size{};
  os_offset_t m_start{};
  os_offset_t m_end{};
};
```

**三个关键点**：

1. **偏移在构造时算死**（`m_start = start_page_no * phy_size`），运行时不重算。
2. **`write` 是同步的**——`os_file_write_retry` 直接系统调用，**不经过 `fil_io` / AIO**。所以带 `DO_NOT_WAKE`（不需要唤醒 AIO 线程）+ `req.dblwr`（标记，fil 层据此跳过压缩/punch hole）。
3. **`os_file_write_retry` 无限重试**（`os0file.cc`）：`DB_IO_ERROR` 时每 10 秒重试一次、**永不放弃**（打 `ER_INNODB_IO_WRITE_ERROR_RETRYING`）。盘短暂不可用时 InnoDB 会 **hang 住重试而不是崩**。

> **矫正一处常见说法**：`Segment::start` 方法**不存在**；`Double_write::init` 也**不存在**（初始化入口是 `create_v2`）。

#### `Double_write`：一个 dblwr 实例

关键成员（`buf0dblwr.cc`）：

```cpp
 protected:
  using Segments = mpmc_bq<Segment *>;
  using Instances = std::vector<Double_write *>;
  using Batch_segments = mpmc_bq<Batch_segment *>;

  uint16_t m_id{};                       // 实例 ID
  ib_mutex_t m_mutex;                    // 保护 m_buf_pages
  os_event_t m_event;                    // 等本实例上一批完成
  std::atomic_bool m_batch_running{false};
  Buffer m_buffer;                       // ★ 批量写缓冲（容量 = dblwr::n_pages 页）
  Buf_pages m_buf_pages;                 // 本批的页（含各自的 e_block）

  static Batch_segments *s_LRU_batch_segments;
  static Batch_segments *s_flush_list_batch_segments;
  static Segments *s_single_segments;
  /* ... reduced 版本 ... */
  static std::vector<Batch_segment *> s_segments;   // 按 batch id 索引
 public:
  static std::vector<dblwr::File> s_files;
  static Instances *s_instances;         // 全局实例表
  unsigned long long m_bytes_written{};
```

**实例数与初始化**：

```cpp
// buf0dblwr.cc —— dblwr::open
  Double_write::s_n_instances = std::max(4UL, srv_buf_pool_instances * 2);
```

**★ 没有 `Double_write::init`**。初始化入口是 `create_v2`（`buf0dblwr.cc`），由 `dblwr::open` 调用：

```cpp
dberr_t Double_write::create_v2 noexcept {
  ut_a(!s_files.empty);
  ut_a(s_instances == nullptr);

  s_instances = ut::new_withkey<Instances>(UT_NEW_THIS_FILE_PSI_KEY);
  if (s_instances == nullptr) return DB_OUT_OF_MEMORY;

  for (uint32_t i = 0; i < s_n_instances; ++i) {
    auto ptr = ut::new_withkey<Double_write>(UT_NEW_THIS_FILE_PSI_KEY, i,
                                             dblwr::n_pages);
    if (ptr == nullptr) { err = DB_OUT_OF_MEMORY; break; }
    s_instances->push_back(ptr);
  }
  ...
}
```

构造函数——**`m_buffer` 与 `m_buf_pages` 容量都是 `dblwr::n_pages` 个物理页，即"一个 batch 最多 `n_pages` 页"**：

```cpp
Double_write::Double_write(uint16_t id, uint32_t n_pages) noexcept
    : m_id(id), m_buffer(n_pages), m_buf_pages(n_pages) {
  ut_a(n_pages == dblwr::n_pages);
  mutex_create(LATCH_ID_DBLWR, &m_mutex);
  m_event = os_event_create;
}
```

**实例的选择**：`s_instances` **前半服务 LRU、后半服务 flush list**：

```cpp
  [[nodiscard]] static Double_write *instance(buf_flush_t flush_type,
                                              uint32_t buf_pool_index,
                                              bool is_reduced) noexcept {
    auto midpoint = s_instances->size / 2;
    auto i = midpoint > 0 ? buf_pool_index % midpoint : 0;
    if (flush_type == BUF_FLUSH_LIST) {
      i += midpoint;
    }
    return (is_reduced ? s_r_instances->at(i) : s_instances->at(i));
  }
```

**启动/关闭时机**（`srv0start.cc`）：`dblwr::open` 在**系统表空间打开之后、redo 初始化（`log_sys_init`）之前**：

```cpp
  if (dblwr::is_enabled && ((err = dblwr::open) != DB_SUCCESS)) {
    return srv_init_abort(err);
  }

  lsn_t new_files_lsn;
  err = log_sys_init(create_new_db, flushed_lsn, new_files_lsn);
```

> 这个顺序很重要：dblwr 必须**早于 redo**，因为恢复时要先用它修页。

---

### 文件布局

#### 文件名

`dblwr_file_open`（`buf0dblwr.cc`）：

```cpp
  file.m_name = std::string(dir_name) + OS_PATH_SEPARATOR + "#ib_";
  file.m_name += std::to_string(srv_page_size) + "_" + std::to_string(id);
  file.m_name += dot_ext[extension];
```

即 `#ib_<page_size>_<id>.dblwr`；reduced 为 `.bdblwr`。默认目录 `"."`（datadir），由 `innodb_doublewrite_dir` 覆盖。

#### ★ 没有文件头——扁平的页数组

`.dblwr` 文件是**纯物理页数组，0 字节文件头**。整个布局由 `dblwr::open`（`buf0dblwr.cc`）在启动时算出来：

```cpp
  Double_write::s_n_instances = std::max(4UL, srv_buf_pool_instances * 2);

  if (dblwr::n_files == 0) dblwr::n_files = 2;
  if (dblwr::n_pages == 0) dblwr::n_pages = srv_n_write_io_threads;

  if (Double_write::s_n_instances < dblwr::n_files) {
    segments_per_file = 1;
    Double_write::s_files.resize(Double_write::s_n_instances);
  } else {
    Double_write::s_files.resize(dblwr::n_files);
    segments_per_file = (Double_write::s_n_instances / dblwr::n_files) + 1;
  }

  dblwr::File::s_n_pages = dblwr::n_pages * segments_per_file;   // 每个文件的批量区页数
```

**两个区域**：

**区域 1：批量写区**（`create_batch_segments`）——每个文件 `segments_per_file` 个段，每段 `n_pages` 页：

```cpp
  for (auto &file : s_files) {
    for (uint32_t i = 0; i < total_pages; i += dblwr::n_pages, ++id) {
      auto s = ut::new_withkey<Batch_segment>(..., id, file, i, dblwr::n_pages);
      ...
      Batch_segments *segments{};
      if (s_files.size > 1) {
        segments = file.is_for_lru ? s_LRU_batch_segments
                                     : s_flush_list_batch_segments;
      } else {
        segments = is_odd(id) ? s_LRU_batch_segments : s_flush_list_batch_segments;
      }
      segments->enqueue(s);
      s_segments.push_back(s);
    }
  }
```

第 `id` 个 batch segment 的偏移：`m_start = (id % segments_per_file) * n_pages * phy_size`。

**区域 2：单页同步 flush 槽位**（`create_single_segments`）——位于**批量区之后（文件尾部）**，每个 Segment **只有 1 页**：

```cpp
  const auto start = dblwr::File::s_n_pages;      // ← 起点 = 批量写区之后

  for (uint32_t i = start; i < start + n_pages; ++i) {
    auto s = ut::new_withkey<Segment>(..., file, i, 1UL);
    s_single_segments->enqueue(s);
  }
```

```cpp
/** DBLWR file pages reserved per instance for single page flushes. */
constexpr uint32_t SYNC_PAGE_FLUSH_SLOTS = 512;    // buf0dblwr.cc
```

槽位分布规则：

| 文件数 | SYNC 槽位在哪 |
|--------|--------------|
| 1 个文件 | 512 页全在文件 0 尾部；批量段按 `is_odd(id)` 交替分给 LRU / flush list 队列 |
| N 个文件 | **只在奇数 id（LRU）文件**上，每个 `512/(N/2)` 页，合计恒为 512 |

#### ★ `n_files = 2` 的含义：奇偶文件功能切分（不是轮换）

```cpp
  bool is_for_lru const { return is_odd; }   // 奇数 id → LRU
```

| 文件 | 承载 |
|------|------|
| **奇数 id** | **LRU 批量写段** + **全部单页 SYNC 槽位** |
| **偶数 id** | **flush list 批量写段** |

> `segments_per_file = s_n_instances / n_files + 1` 里的 **"+1" 是并发余量**：实例数 8 但段数 10，保证 `dequeue` 不至于一直饿死。

**布局图**（`n_files=2`、`srv_buf_pool_instances=4`）：

```
s_n_instances = max(4, 4*2) = 8
segments_per_file = 8/2 + 1 = 5
File::s_n_pages = n_pages * 5

文件 0  (#ib_16384_0.dblwr, id=0, 偶数 → flush list)
┌──────────────────────────────────────────────────┐
│ seg0 │ seg1 │ seg2 │ seg3 │ seg4 │  (无 SYNC 区)  │
│ 每段 n_pages 页 → s_flush_list_batch_segments      │
└──────────────────────────────────────────────────┘
 0                                    File::s_n_pages

文件 1  (#ib_16384_1.dblwr, id=1, 奇数 → LRU)
┌──────────────────────────────────────────────────┬──────────────────┐
│ seg0..seg4 → s_LRU_batch_segments                 │ SYNC 槽位 512 个 │
│                                                  │ (每槽 1 页)      │
└──────────────────────────────────────────────────┴──────────────────┘
                                      File::s_n_pages
                                      ↑ SYNC 槽位起点
```

---

### 批量写完整流程

#### 调用链

```
buf_flush_page / buf_flush_try_neighbors
  └─ buf_flush_write_block_low                     buf0flu.cc
       └─ dblwr::write(flush_type, bpage, sync)      buf0dblwr.cc
            ├─[跳过分支] Double_write::write_to_datafile(bpage, sync, nullptr)
            └─[异步批量] dblwr::get_encrypted_frame          :2413
                        Double_write::submit                 :669
                          └─ Double_write::enqueue           :598
                               └─ [缓冲满] flush_to_disk     :549
                                    ├─ wait_for_pending_batch:527
                                    └─ write_pages           :2246
                                         ├─ write_dblwr_pages:2160  (同步写 dblwr)
                                         └─ write_data_pages :2192  (异步 AIO 写数据文件)
                                              └─ write_to_datafile :1603

... AIO 完成 ...
buf_page_io_complete
  └─ dblwr::write_complete(bpage, flush_type)        buf0dblwr.cc
       └─ Double_write::write_complete             :2559
            └─ Batch_segment::write_complete       :1058
                 └─ [最后一个] → fil_flush_file_spaces + 归还段
```

#### 入口 `dblwr::write` 的两分支

`buf0dblwr.cc`：

```cpp
    if (!sync && flush_type != BUF_FLUSH_SINGLE_PAGE) {
      MONITOR_INC(MONITOR_DBLWR_ASYNC_REQUESTS);
      ut_d(bpage->release_io_responsibility);
      Double_write::submit(flush_type, bpage, e_block);
      err = DB_SUCCESS;
    } else {
      MONITOR_INC(MONITOR_DBLWR_SYNC_REQUESTS);
      /* Disable batch completion in write_complete. */
      bpage->set_dblwr_batch_id(std::numeric_limits<uint16_t>::max);
      err = Double_write::sync_page_flush(bpage, e_block);
    }
```

| 分支 | 条件 | 行为 |
|------|------|------|
| **批量（batched）** | `!sync && flush_type != BUF_FLUSH_SINGLE_PAGE` | 页进 batch buffer，攒一批后**一次写 dblwr（+可选 fsync）**，再异步写数据文件 |
| **同步单页** | `sync==true` 或 `BUF_FLUSH_SINGLE_PAGE` | 抢一个 SYNC 槽位 → 写 dblwr → fsync → 同步写数据文件 → `fil_flush` |

> **★ 所有 `BUF_FLUSH_SINGLE_PAGE` 走同步分支**，即使调用方请求 async。`buf0lru.cc` 的注释明确记录了这一点并标注为待确认的 TODO（见 [Misc](#misc)）。

#### `enqueue`：缓冲满了就强制刷盘

`buf0dblwr.cc`：

```cpp
  void enqueue(buf_flush_t flush_type, buf_page_t *bpage,
               const file::Block *e_block) noexcept {
    ...
    for (;;) {
      mutex_enter(&m_mutex);

      if (m_buffer.append(frame, len)) {     // ① 能放下就追加
        break;
      }

      if (flush_to_disk(flush_type)) {       // ② 放不下 → 强制刷盘
        auto success = m_buffer.append(frame, len);   // ③ 刷完必成功
        ut_a(success);
        break;
      }
      ut_ad(!mutex_own(&m_mutex));           // ④ 上一批在跑：已释放 mutex 并等过 event，重来
    }

    m_buf_pages.push_back(bpage, e_block);
    mutex_exit(&m_mutex);
  }
```

`Buffer::append`（`buf0dblwr.h`）是**整页 memcpy**：

```cpp
  bool append(const void *ptr, size_t n_bytes) noexcept {
    if (m_next + m_phy_size > m_ptr + m_n_bytes) return false;   // 放不下，什么都不拷
    memcpy(m_next, ptr, n_bytes);
    m_next += m_phy_size;
    return true;
  }
```

> 所以 dblwr 批量写 = **把页内容复制到实例私有对齐缓冲**，攒满后一次写。压缩页也占一整页槽位（只拷 `n_bytes`，但按 `m_phy_size` 前进）。

#### `flush_to_disk` 与"上一批未完成"的等待

`buf0dblwr.cc`：

```cpp
  bool wait_for_pending_batch noexcept {
    auto sig_count = os_event_reset(m_event);
    std::atomic_thread_fence(std::memory_order_acquire);

    if (m_batch_running.load(std::memory_order_acquire)) {
      mutex_exit(&m_mutex);
      MONITOR_INC(MONITOR_DBLWR_FLUSH_WAIT_EVENTS);
      os_event_wait_low(m_event, sig_count);
      sig_count = os_event_reset(m_event);
      return true;
    }
    return false;
  }

  bool flush_to_disk(buf_flush_t flush_type) noexcept {
    if (wait_for_pending_batch) {
      ut_ad(!mutex_own(&m_mutex));
      return false;          // ← 有 batch 在跑，返回 false 让调用者重试
    }
    MONITOR_INC(MONITOR_DBLWR_FLUSH_REQUESTS);
    write_pages(flush_type);
    ut_a(m_buffer.empty);
    ut_a(m_buf_pages.empty);
    return true;
  }
```

> **同一个 dblwr 实例同一时刻只有一个 batch 在飞**（`m_batch_running`）。这是 dblwr 在高并发刷脏时的主要阻塞点，`MONITOR_DBLWR_FLUSH_WAIT_EVENTS` 就是它的计数。

#### `write_pages`：先 dblwr，再数据文件

`buf0dblwr.cc`：

```cpp
uint16_t Double_write::write_dblwr_pages(buf_flush_t flush_type) noexcept {
  ut_a(!m_buffer.empty);

  Batch_segment *batch_segment{};
  auto segments = flush_type == BUF_FLUSH_LRU ? s_LRU_batch_segments
                                              : s_flush_list_batch_segments;

  while (!segments->dequeue(batch_segment)) {     // 无锁队列取空闲段
    std::this_thread::yield;
  }

  batch_segment->start(this);                     // m_batch_running = true
  batch_segment->write(m_buffer);                 // ★ 同步写 dblwr 文件

  m_bytes_written += m_buffer.size;
  m_buffer.clear;

#ifndef _WIN32
  if (is_fsync_required) {
    batch_segment->flush;                       // fsync dblwr 文件
  }
#endif

  batch_segment->set_batch_size(m_buf_pages.size);   // m_uncompleted = n
  return batch_segment->id;
}
```

```cpp
void Double_write::write_data_pages(buf_flush_t flush_type,
                                    uint16_t batch_id) noexcept {
  for (uint32_t i = 0; i < m_buf_pages.size; ++i) {
    const auto bpage = std::get<0>(m_buf_pages.m_pages[i]);
    bpage->set_dblwr_batch_id(batch_id);
    ut_d(bpage->take_io_responsibility);
    auto err = write_to_datafile(bpage, false, std::get<1>(m_buf_pages.m_pages[i]));
    ...
  }

  srv_stats.dblwr_writes.inc;          // ← Innodb_dblwr_writes +1（每 batch 一次）

  m_buf_pages.clear;
  os_aio_simulated_wake_handler_threads;   // 唤醒 AIO 线程
}
```

> **★ "写完 dblwr 后把页写到数据文件"发生在 `write_data_pages`，不是 `write_complete` 回调里。** `write_complete` 是数据文件写**完成之后**的收尾。

#### `write_complete`：收尾与 fsync 摊薄

`buf0dblwr.cc`：

```cpp
        if (batch_segment->write_complete) {     // 引用计数减到 0（最后一条 AIO 完成）
          batch_segment->completed;              // m_batch_running=false; os_event_set

          srv_stats.dblwr_pages_written.add(batch_segment->batch_size);

          batch_segment->reset;

          Batch_segments *segments{ ... };

          fil_flush_file_spaces;                 // ★ fsync 数据文件（批量！）

          while (!segments->enqueue(batch_segment)) {   // 归还段
            std::this_thread::yield;
          }
        }
```

**★ 这是 dblwr 在慢速存储上最重要的价值**：把一整批页的随机 fsync **摊成一次**（`fil_flush_file_spaces` 扫 unflushed 链表统一 fsync）。在云盘上 fsync 是最贵的系统调用，这个摊薄直接决定刷脏吞吐。

---

### 同步单页 flush

`Double_write::sync_page_flush`（`buf0dblwr.cc`）完整七步：

```cpp
dberr_t Double_write::sync_page_flush(buf_page_t *bpage,
                                      file::Block *e_block) noexcept {
  Segment *segment{};

  while (!s_single_segments->dequeue(segment)) {      // ① 从 512 个 slot 抢一个
    std::this_thread::yield;
  }

  single_write(segment, bpage, e_block);              // ② 同步写 dblwr

#ifndef _WIN32
  if (is_fsync_required) {
    segment->flush;                                 // ③ fsync dblwr 文件
  }
#endif

  auto err = write_to_datafile(bpage, true, e_block); // ④ 同步写数据文件

  if (err == DB_SUCCESS) {
    fil_flush(bpage->id.space);                     // ⑤ fsync 该表空间
  } else {
    if (e_block != nullptr) os_free_block(e_block);
  }

  while (!s_single_segments->enqueue(segment)) {      // ⑥ 归还 slot
    UT_RELAX_CPU;
  }

  buf_page_io_complete(bpage, true);                  // ⑦ 完成 IO（true = 同时淘汰出 LRU）
  return DB_SUCCESS;
}
```

| 问题 | 答案 |
|------|------|
| slot 怎么分配 | 从 `s_single_segments`（`mpmc_bq`，容量 `ut_2_power_up(512)=512`）`dequeue`；空则 `yield` 自旋 |
| 写完 dblwr 后怎么写数据文件 | `write_to_datafile(bpage, /*sync=*/true, ...)` → `fil_io(..., sync=true, ...)` **同步** |
| 是否 `fil_flush` | **是** |

> **单页路径不参与 batch 机制**：调用方在 `dblwr::write` 里预先 `set_dblwr_batch_id(UINT16_MAX)`，所以随后的 `write_complete` 里 `batch_id == max`，不会去动 `Batch_segment`。

---

### `write_to_datafile`

`buf0dblwr.cc`：

```cpp
dberr_t Double_write::write_to_datafile(const buf_page_t *in_bpage, bool sync,
                                        const file::Block *e_block) noexcept {
  ...
  uint32_t type = IORequest::WRITE;
  if (sync) {
    type |= IORequest::DO_NOT_WAKE;
  }

  IORequest io_request(type);
  io_request.set_encrypted_block(e_block);
  io_request.set_original_size(bpage->size.physical);

  auto err = fil_io(io_request, sync, bpage->id, bpage->size, 0, len, frame, bpage);
  ut_a(err == DB_SUCCESS || err == DB_TABLESPACE_DELETED || err == DB_PAGE_IS_STALE);
  return err;
}
```

**IORequest 标志小结**：

| 位置 | 标志 | 说明 |
|------|------|------|
| 写 dblwr 文件（`Segment::write`） | `WRITE \| DO_NOT_WAKE` + **`req.dblwr`** | 同步系统调用，不唤醒 AIO 线程；带 dblwr 标记 |
| 写数据文件（批量） | `WRITE` | **异步 AIO**；循环结束后统一 `os_aio_simulated_wake_handler_threads` |
| 写数据文件（单页） | `WRITE \| DO_NOT_WAKE` | 同步 |

> **`req.dblwr` 标记很关键**：`fil0fil.cc` 用 `req_type.is_dblwr` 做断言，且 dblwr 写**绕过 `Fil_shard::do_io` 的压缩/加密注入**——所以压缩与加密必须在 dblwr 之前手工做完（见下节）。

---

### 崩溃恢复时如何使用 dblwr

**★ `Double_write::recover` 不存在**。恢复用的是 `dblwr::recv::` 命名空间下的一套独立结构，入口 `dblwr::recv::Pages::recover`。

#### 时序（按实际执行顺序）

| 步骤 | 位置 | 做什么 |
|------|------|--------|
| ① | `log0recv.cc`（`recv_sys_init`） | 创建空的 `dblwr::recv::DBLWR` |
| ② | `fsp0sysspace.cc`（`read_lsn_and_check_flags`） | **`load` + `reduced_load`**——扫描 dblwr 目录，把 `.dblwr` / `.bdblwr` 全部读进内存。**在 redo 扫描之前** |
| ③ | `fil0fil.cc`（`fil_tablespace_open_for_recovery`） | 每个表空间打开时 `recover(space)`——只修属于它的页 |
| ④ | `log0recv.cc`（`recv_init_crash_recovery`） | **`recv_sys->dblwr->recover`**（space=nullptr → 修所有） |
| ⑤ | `fsp0file.cc`（`restore_from_doublewrite`） | page 0 损坏时单独修复 |
| ⑥ | `fil0fil.cc` | `check_missing_tablespaces` |
| ⑦ | `log0recv.cc`（`recv_sys_finish`） | `recovered` → `dblwr::reset_files` 截断 dblwr 文件到配置大小 |

```cpp
// log0recv.cc
static void recv_init_crash_recovery {
  recv_needed_recovery = true;
  ib::info(ER_IB_MSG_726);
  ib::info(ER_IB_MSG_727);

  recv_sys->dblwr->recover;      // ← 在 spawn recv_writer 之前

  if (srv_force_recovery < SRV_FORCE_NO_LOG_REDO) { ... }
}
```

**★ dblwr 目录扫描用正则**（`buf0dblwr.cc`），且**不依赖当前配置**：

```cpp
  /* The number of buffer pool instances can change. Therefore we must:
    1. Scan the doublewrite directory for all *.dblwr files and load
       their contents.
    2. Reset the file sizes after recovery is complete. */

  std::string rexp{"#ib_([0-9]+)_([0-9]+)\\"};
  rexp.append(dot_ext[DWR]);
  const std::regex regex{rexp};
```

用 `max(srv_buf_pool_instances, ids.back+1)` 个文件 id 去尝试打开——这样 `innodb_buffer_pool_instances` 变过也能覆盖全部历史文件。

#### torn page 的三个判定 case

`dblwr::recv::Pages::dblwr_recover_page`（`buf0dblwr.cc`）：

```cpp
  /* Read in the page from the data file to compare. */
  auto err = fil_io(request, true, page_id, page_size, 0, page_size.physical,
                    buffer.begin, nullptr);

  BlockReporter data_file_page(true, buffer.begin, page_size,
                               fsp_is_checksum_disabled(space->id));

  if (data_file_page.is_corrupted) {
    ib::info(ER_IB_MSG_DBLWR_1315) << "Database page corruption or"
        << " a failed file read of page " << page_id << ". Trying to recover it from the"
        << " doublewrite file.";

    const bool dblwr_corrupted = is_dblwr_page_corrupted(page, space, page_no, &dblwr_err);

    if (dblwr_corrupted) {
      /* 数据页坏了 AND dblwr 副本也坏了 → dump 两份后 fatal */
      ib::fatal(UT_LOCATION_HERE, ER_IB_MSG_DBLWR_1306);
    }

  } else {
    bool data_page_zeroes = buf_page_is_zeroes(buffer.begin, page_size);
    bool dblwr_zeroes = buf_page_is_zeroes(page, page_size);
    const bool dblwr_corrupted = is_dblwr_page_corrupted(page, space, page_no, &dblwr_err);

    if (data_page_zeroes && !dblwr_zeroes && !dblwr_corrupted) {
      /* Database page contained only zeroes, while a valid copy is
      available in dblwr buffer. */
    } else {
      return false;      // 数据页是好的，不修
    }
  }
  ...
  err = fil_io(write_request, true, page_id, page_size, 0, page_size.physical,
               const_cast<byte *>(page), nullptr);
  ib::info(ER_IB_MSG_DBLWR_1308) << "Recovered page " << page_id << " from the doublewrite buffer.";
```

| case | 条件 | 行为 |
|------|------|------|
| **1. 数据页 checksum 损坏** | `is_corrupted` | 用 dblwr 副本覆盖；**若 dblwr 副本也损坏 → `ib::fatal`** |
| **2. 数据页全 0 且 dblwr 副本非 0 且完好** | `data_page_zeroes && !dblwr_zeroes && !dblwr_corrupted` | 覆盖（应对"文件扩展了但内容未落盘"） |
| **3. 其余** | — | 不修（数据页是好的） |

**比对前要先解密 + 解压**（`is_dblwr_page_corrupted`）：

```cpp
  if (dblwr_page.is_encrypted) {
    *err = en.decrypt(req_type, page, z_page_size, nullptr, 0);
    if (*err != DB_SUCCESS) {
      corrupted = true;
    } else {
      if (page_type == FIL_PAGE_COMPRESSED) {
        *err = os_file_decompress_page(true, page, nullptr, 0);
        ...
      }
    }
  }
```

> 因为 dblwr 文件里存的是**加密后（可能还压缩过）的页**，必须先还原才能算 checksum 比对。

---

### 加密帧 `get_encrypted_frame`

`buf0dblwr.cc`（核心片段）：

```cpp
  IORequest type(IORequest::WRITE);
  void *frame{};
  uint32_t len{};

  Double_write::prepare(bpage, &frame, &len);
  ulint n = len;

  /* Transparent page compression (TPC) is disabled if punch hole is not
  supported. A similar check is done in Fil_shard::do_io. */
  const bool do_compression =
      space->is_compressed && !bpage->size.is_compressed &&
      IORequest::is_punch_hole_supported && node->punch_hole;

  if (do_compression) {
    /* @note Compression needs to be done before encryption. */
    type.compression_algorithm(space->compression_type);
    compressed_block = os_file_compress_page(type, frame, &n);
  }

  type.get_encryption_info.set(space->m_encryption_metadata);
  auto e_block = os_file_encrypt_page(type, frame, n);

  if (compressed_block != nullptr) {
    file::Block::free(compressed_block);
  }
  return e_block;
```

#### ★ 为什么需要"单独的加密帧"

调用处的注释一句话点破（`buf0dblwr.cc`）：

```cpp
    /* Encrypt the page here, so that the same encrypted contents are written
    to the dblwr file and the data file. */
```

四个理由：

1. **必须保证 dblwr 副本与数据文件副本字节级一致**。若各自独立加密，AES-CBC 的 IV / key rotation 可能不同 → 两份副本不一致 → 恢复出来的页解密失败或被判损坏。
2. **dblwr 写绕过了 fil 层**。`Segment::write` 直接 `os_file_write_retry`，不经过 `Fil_shard::do_io`，而压缩/加密/punch hole 逻辑都在 fil 层。所以必须提前手工做完。
3. **同一个 `file::Block` 被复用 3 次**：`enqueue` 拷进 dblwr 缓冲 → `write_data_pages` 传给 `write_to_datafile` → `IORequest::set_encrypted_block` 让 os/AIO 层在 IO 结束后 `os_free_block` 释放。**同一段物理内存天然一致**。
4. **`file::Block::m_size` 可以 ≠ UNIV_PAGE_SIZE**（压缩后变小）；`Buffer::append` 只拷 `n_bytes` 但按 `m_phy_size` 前进，所以压缩页在 dblwr 文件里仍占一整页槽位。

`file::Block` 定义在 `include/os0file.h`（**不在 `dblwr` 命名空间**）：

```cpp
struct Block {
  byte *m_ptr;
  /** Size of the data in memory block. This may be not UNIV_PAGE_SIZE if the
  data was compressed before encryption. */
  size_t m_size;
  byte pad[ut::INNODB_CACHE_LINE_SIZE];      // 避免 false sharing
  std::atomic<bool> m_in_use;
};
```

---

### O_DIRECT 与 fsync 的完整交互

#### `is_fsync_required`

`buf0dblwr.cc`：

```cpp
  [[nodiscard]] static bool is_fsync_required noexcept {
    /* srv_unix_file_flush_method is a dynamic variable. */
    return srv_unix_file_flush_method != SRV_UNIX_O_DIRECT &&
           srv_unix_file_flush_method != SRV_UNIX_O_DIRECT_NO_FSYNC;
  }
```

> **★ `O_DIRECT` 和 `O_DIRECT_NO_FSYNC` 都不 fsync dblwr 文件**。只有 `fsync` / `O_DSYNC` / `littlesync` / `nosync` 才 fsync。

#### 五处 fsync 点及其在 `O_DIRECT_NO_FSYNC` 下的行为

| # | 位置 | 代码 | NO_FSYNC 下 |
|---|------|------|------------|
| 1 | `buf0dblwr.cc`（批量） | `if (is_fsync_required) batch_segment->flush;` | **跳过** |
| 2 | `buf0dblwr.cc`（reduced） | 同上 | **跳过** |
| 3 | `buf0dblwr.cc`（单页） | `if (is_fsync_required) segment->flush;` | **跳过** |
| 4 | `buf0dblwr.cc`（batch 完成） | `fil_flush_file_spaces;` | **跳过**（除文件变大） |
| 5 | `buf0dblwr.cc`（单页） | `fil_flush(bpage->id.space);` | **跳过**（除文件变大） |

第 4、5 点的短路在 fil 层（`fil0fil.cc`）：

```cpp
static inline bool fil_disable_space_flushing(const fil_space_t *space) {
#ifndef _WIN32
  if (space->purpose == FIL_TYPE_TABLESPACE &&
      srv_unix_file_flush_method == SRV_UNIX_O_DIRECT_NO_FSYNC) {
    return true;
  }
#endif
  if (space->purpose == FIL_TYPE_TEMPORARY) return true;
  return false;
}
```

`Fil_shard::space_flush` 里的实际逻辑（`fil0fil.cc`）：

```cpp
    /* Skip flushing if the file size has not changed since
    last flush was done and the flush mode is O_DIRECT_NO_FSYNC */
    if (disable_flush && (file.flush_size == file.size)) {
      continue;
    }
```

> 即：`O_DIRECT_NO_FSYNC` 下**只有文件被扩展（size 变了）时才 fsync**，普通页写一律不 fsync（依赖 O_DIRECT 绕过 page cache）。完整机制见 [`fil.md`](fil.md)「刷盘与 fsync 并发去重」。

#### dblwr 文件本身也开 O_DIRECT

`os0file.cc`：

```cpp
  if ((!read_only || type == OS_CLONE_DATA_FILE) && *success &&
      (type == OS_DATA_FILE || type == OS_CLONE_DATA_FILE ||
       type == OS_DBLWR_FILE) &&
      (srv_unix_file_flush_method == SRV_UNIX_O_DIRECT ||
       srv_unix_file_flush_method == SRV_UNIX_O_DIRECT_NO_FSYNC)) {
    os_file_set_nocache(file.m_file, name, mode_str);
  }
```

> 这也是为什么 dblwr 的写缓冲必须 `ut::aligned_zalloc` 页对齐。

---

### `force_flush` 的四个调用点

实例级（`buf0dblwr.cc`）：

```cpp
  void force_flush(buf_flush_t flush_type) noexcept {
    for (;;) {
      mutex_enter(&m_mutex);
      if (!m_buf_pages.empty && !flush_to_disk(flush_type)) {
        ut_ad(!mutex_own(&m_mutex));
        continue;                 // 有 batch 在跑 → 等完再试
      }
      break;
    }
    mutex_exit(&m_mutex);
  }
```

静态级刷 regular + reduced 两个实例；命名空间级 `dblwr::force_flush_all` 遍历所有 BP 实例的 LRU 与 flush list。

| # | 位置 | 场景 |
|---|------|------|
| 1 | `buf0flu.cc` | 刷 flush list 时拿不到 `rw_lock` 的 SX 锁 → **先把 dblwr 里挂着的页强制刷出去再阻塞等待**（解开潜在的 latch 等待环） |
| 2 | `buf0flu.cc` | `buf_flush_end`：一个 flush batch 结束 |
| 3 | `ha_innodb.cc` | `doublewrite_update`：**动态切换 `innodb_doublewrite` 模式前**刷掉所有半满的 dblwr 缓冲 |
| 4 | `buf0dblwr.cc` | `UNIV_DEBUG` 下 `dblwr::Force_crash` 命中（配合 `DBUG_SUICIDE` 测 torn page 恢复） |

第 1 点的完整上下文（`buf0flu.cc`）：

```cpp
    if (flush_type == BUF_FLUSH_LIST && is_uncompressed &&
        !rw_lock_sx_lock_nowait(rw_lock, BUF_IO_WRITE, UT_LOCATION_HERE)) {
      if (!fsp_is_system_temporary(bpage->id.space) && dblwr::is_enabled) {
        dblwr::force_flush(flush_type, buf_pool_index(buf_pool));
      } else {
        buf_flush_sync_datafiles;
      }
      rw_lock_sx_lock_gen(rw_lock, BUF_IO_WRITE, UT_LOCATION_HERE);   // 阻塞式
    }
```

---

### 监控与状态变量

#### `MONITOR_DBLWR_*`

`srv0mon.h`：

| 计数器 | 累加点 | 语义 |
|--------|--------|------|
| `MONITOR_DBLWR_ASYNC_REQUESTS` | `buf0dblwr.cc` | 走**异步批量**路径的页请求数 |
| `MONITOR_DBLWR_SYNC_REQUESTS` | `buf0dblwr.cc` | 走**同步单页**路径的页请求数 |
| `MONITOR_DBLWR_FLUSH_REQUESTS` | `buf0dblwr.cc` | 实际发起的批量 flush 次数 |
| `MONITOR_DBLWR_FLUSH_WAIT_EVENTS` | `buf0dblwr.cc` | ★ **因上一批还在跑而阻塞等待的次数**（dblwr 的主要阻塞指标） |

#### 状态变量（★ 累加时机值得注意）

| 变量 | 位置 | 语义 |
|------|------|------|
| `Innodb_dblwr_writes` | `buf0dblwr.cc` `srv_stats.dblwr_writes.inc` | **每写完一个 batch +1**（不是每页），在 `write_data_pages` 里 |
| `Innodb_dblwr_pages_written` | `buf0dblwr.cc` `add(batch_segment->batch_size)` | **整个 batch 全部 AIO 完成时才一次性加** |

> 所以这两个值是**阶梯式增长**的，不是平滑的。用它们算"平均每个 batch 多少页"= `pages_written / writes`，可以反推 `innodb_doublewrite_pages` 是否合理。

shutdown 时还会打印累计写入字节数：`"Bytes written to disk by DBLWR (ON)"` / `"(REDUCED)"`（`buf0dblwr.cc`）。

---

### reduced 模式（DETECT_ONLY）

8.0.30+ 引入。**只记 16 字节的三元组，不记页面内容**：

```cpp
const uint32_t REDUCED_BATCH_PAGE_SIZE = 8192;

/* 20-byte header.
Fields        : [batch id][checksum][data len][batch type][unused  ]
Field Width   : [4 bytes ][4 bytes ][2 bytes ][  1 byte  ][ 9 bytes] */
constexpr const uint32_t REDUCED_HEADER_SIZE = 4 + 4 + 2 + 1 + 9;     // 20
constexpr const uint32_t REDUCED_ENTRY_SIZE =
    sizeof(space_id_t) + sizeof(page_no_t) + sizeof(lsn_t);            // 16
constexpr const uint32_t REDUCED_MAX_ENTRIES =
    (REDUCED_BATCH_PAGE_SIZE - REDUCED_HEADER_SIZE) / REDUCED_ENTRY_SIZE;  // 510
```

**★ 只能检测，不能修复**（`buf0dblwr.cc` 注释）：

> *"We cannot recover page because the entire page is not logged only an entry of space_id, page_id, LSN is logged. So we abort the server. It is expected that the user restores from backup."*

`reduced_recover`发现损坏直接 `ib::fatal(ER_REDUCED_DBLWR_PAGE_FOUND)`。

**与 regular 的协同**：恢复时若发现 reduced 里记录的 LSN **比 regular dblwr 的页还新**，说明 regular 副本是旧的，**不能拿来恢复** → 直接 fatal：

```cpp
  lsn_t dblwr_lsn = mach_read_from_8(page + FIL_PAGE_LSN);
  if (found && reduced_lsn != LSN_MAX && reduced_lsn > dblwr_lsn) {
    ib::fatal(UT_LOCATION_HERE, ER_REDUCED_DBLWR_PAGE_FOUND, ...);
  }
```

reduced 使用独立的 `.bdblwr` 文件，**只有 1 个**，段从 `s_segments.size` 开始编号（保证 batch id 大于 `s_regular_last_batch_id`，靠 `is_reduced_batch_id` 区分）。

---

## 相关的系统变量

**★ 注意：dblwr 的参数全部定义在 `storage/innobase/handler/ha_innodb.cc`（`sql/sys_vars.cc` 里 0 条匹配）**——这是 InnoDB 与 server 层参数分离的体现。

| 变量 | 默认 | 范围 | 属性 |
|------|------|------|------|
| `innodb_doublewrite` | `ON` | `OFF` / `ON` / **`DETECT_ONLY`** / **`DETECT_AND_RECOVER`** / `FALSE` / `TRUE` | **动态**（但 ON↔OFF 必须重启） |
| `innodb_doublewrite_dir` | 空（→ datadir） | 路径 | 只读 |
| `innodb_doublewrite_pages` | **0** → 实际 = `innodb_write_io_threads` | [0, 512] | 只读 |
| `innodb_doublewrite_files` | **0** → 实际 = 2 | [0, 256] | 只读 |
| `innodb_doublewrite_batch_size` | 0 | [0, 256] | 只读，**★ 8.0.39 中完全未被使用（no-op）** |

**几个易错点**：

1. **`innodb_doublewrite` 的 6 个取值**里，`ON` = `TRUE` = `DETECT_AND_RECOVER`（三者等价），`OFF` = `FALSE`。
2. **热切换只能 `ON ↔ DETECT_ONLY`**。`doublewrite_update`（`ha_innodb.cc`）对 `ON↔OFF` 直接报错要求重启。
3. **`innodb_doublewrite_pages` / `files` 的源码初值**（`n_pages{64}` / `n_files{1}`）**会被 sysvar 的 0 覆盖**，然后在 `dblwr::open` 里二次兜底为 `write_io_threads` / `2`。
4. **`innodb_doublewrite_batch_size` 是空转的**——批量大小实际由 `innodb_doublewrite_pages` 决定。`dblwr::batch_size` 在代码中除了同名成员函数外无任何引用。
5. **相对路径会被加 `#` 前缀**（`ha_innodb.cc`）——InnoDB 内部表示"相对 datadir"的约定。

---

## Misc

### 已知问题与 TODO（源码注释里的）

| 问题 | 位置 | 注释 |
|------|------|------|
| **★ 单页 flush 被迫转同步** | `buf0lru.cc` | *"if double write is used, dblwr::write forces all single page flush to sync IO. ... For bulk flush async trigger could be better for performance and seems to be the case in 5.7. **Need to validate if 8.0 forcing sync flush is intentional** - No functional impact."* |
| **legacy v1 dblwr 的 FIXME** | `srv0start.cc` | *"We always create the legacy double write buffer to preserve the expected page ordering of the system tablespace. **FIXME: Try and remove this requirement.**"* |
| **异步写的临时修复** | `buf0dblwr.cc` | *"... This is a temp fix to address this situation. Ideally we should handle these errors in single place possibly by one function."* |
| **两处死声明** | `buf0dblwr.cc` | `void write(buf_flush_t)` 与 `static dberr_t start` 声明了但**无定义** |

### 常见误解

| 误解 | 正确 |
|------|------|
| "redo 能修半页写" | **不能**。redo 假设页的其他部分是对的；torn page 只能靠 dblwr |
| "单页刷盘不走 doublewrite" | **走**，用文件尾部的 `SYNC_PAGE_FLUSH_SLOTS`（512 个）。只有三类跳过：OFF / 临时表空间 / 只读 |
| "dblwr 文件有文件头" | **没有**，是扁平的物理页数组，布局全在启动时算 |
| "`innodb_doublewrite_files=2` 是双缓冲轮换" | **不是**，是**奇偶文件功能切分**（奇数 LRU + 全部 SYNC 槽位，偶数 flush list） |
| "`O_DIRECT` 下 dblwr 会 fsync" | **不会**。`is_fsync_required` 对 `O_DIRECT` 与 `O_DIRECT_NO_FSYNC` **都返回 false** |
| "`Innodb_dblwr_pages_written` 每页 +1" | 不是，**整个 batch 完成时一次性 `add(batch_size)`** |
| "dblwr 写走 AIO" | **不是**，`Segment::write` 是同步 `os_file_write_retry`；只有**写数据文件**才是异步 AIO |
| "参数在 `sys_vars.cc`" | 全在 `ha_innodb.cc` |

---

## 参考

> 本文结论基于 MySQL 8.0.39 源码逐行核实；涉及的函数与类型可直接在本仓库检索。

**相关文档**

- 刷脏（dblwr 的上游）：[`buffer_pool.md`](buffer_pool.md)
- redo 与 dblwr 的分工（为什么 redo 防不了半页写）：[`redo_log.md`](redo_log.md)
- dblwr 之下的 I/O 原语与 AIO：[`io.md`](io.md)　|　表空间与 fsync 去重：[`fil.md`](fil.md)
- 崩溃恢复整体：[`recovery.md`](recovery.md)
- 块设备"最小契约"如何逼出 dblwr：[`../cloud/cloud_storage.md`](../cloud/cloud_storage.md)
