# Buffer Pool 深度解析

> 基于 MySQL 8.0.39 源码，涵盖 控制块状态机、LRU 中点替换、预读、脏页刷盘（page cleaner）、doublewrite、自适应刷脏。
>
> **边界**：本篇讲 buffer pool 内部机制；redo 刷盘与 LSN 体系见 [`redo_log.md`](redo_log.md)，B-tree 页内操作见 [`btr.md`](btr.md)，AHI 见 btr.md「自适应哈希索引」节，崩溃恢复见 redo_log.md。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [关键源码位置速查](#关键源码位置速查)
- [参考](#参考)

---

## 概述

### 是什么

InnoDB 的缓冲池（buffer pool）：一块预分配的内存区域，把表空间数据页（默认 16KB）缓存其中，作为"磁盘页在内存中的代理"。每个页槽位配一个控制块（`buf_page_t` / `buf_block_t`）记录"这个槽现在装哪一页、脏否、被谁 pin、I/O 进行否"。

### 用途

在 DRAM 与磁盘之间削峰：读命中则免一次磁盘 I/O；写则合并（页在内存改脏，延迟批量刷盘），配合 redo 的 WAL 保证崩溃一致。buffer pool 是 InnoDB 吞吐的命门——几乎所有页访问都要先过它。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.6 | 多 buffer pool instance（`innodb_buffer_pool_instances`），每实例独立 LRU/page_hash/flush_list，降低单 mutex 争用 |
| 5.7 | 在线 resize（chunk 增删，`innodb_buffer_pool_size` 动态生效，无需重启）；dump/load 预热（`innodb_buffer_pool_dump_now`） |
| 8.0 | flush 无锁化（8.0.19 起 `flush_list_mutex` 替代 `buf_pool mutex` 扫 flush_list，配 hazard pointer）；page cleaner 协调者/worker 模型（一个 coordinator 派发 slot + N 个 worker）；io_fix 用 `Stateful_latching_rules` 做声明式 latch 协议校验；doublewrite 8.0.20 重构为独立文件双池 |

---

## 理论基础

### 设计思想与权衡

**问题本质**：磁盘页远多于内存槽位，必须做"装谁、淘汰谁、何时刷脏"三件事的调度。这三件事的耦合点就是控制块的状态字段。

**权衡一：slot-based 而非对象式**。InnoDB 选了固定页大小的槽位（`buf_block_t::frame`），靠 `page_hash` 把 `(space_id, page_no)` 解耦到任意槽——对比"每页一个 malloc 对象"的对象式（CMU BusTub 教学版即如此）。代价是固定 16KB 槽对压缩页浪费（COMPRESSED 页只有 1-8KB，仍占一个 16KB 槽或走 buddy），收益是分配/回收/对齐极快、可预分配大块、内存碎片可控。**失效场景**：压缩表密集时 buddy 切分频繁，`zip_free` 链压力上升——这是 COMPRESSED 行格式代价的另一半。

**权衡二：LRU 中点插入而非纯 LRU**。纯 LRU 有"全表扫描一次性灌满缓冲池把热页冲走"的经典病（table scan pollution）。InnoDB 把 LRU 分 old/new 两段，**新读入页插 old 区头部而非 new 区头部**，必须"活过 `innodb_old_blocks_time`（默认 1 秒）且被再次访问"才升 young。**被否决的方案**：纯 LRU（全表扫描污染）、CLOCK/二次机会（InnoDB 的"再次访问 + 时间窗"本质是二次机会的变体，但保留了时间维度做更精确的过滤）。**退化阈值**：`innodb_old_blocks_pct` 调到 100 则退化为纯 LRU（无 old 区隔离），全表扫描会冲热页。

**权衡三：异步刷脏 + WAL 解耦**。刷脏不卡用户线程——page cleaner 后台批量异步刷，用户线程只在"free page 供给不上"时才被迫单页同步刷（性能悬崖）。**代价**：必须用 redo 的 WAL 兜底——脏页写盘前，该页所有 redo 必须先 fsync 落盘（`buf_flush_write_block_low` 先 `log_write_up_to`），否则崩溃恢复时数据文件有新版本而 redo 缺失，无法重放。这条 WAL 约束是"延迟刷脏"这条决策的**不可分割的另一半**。

**权衡四：声明式 latch 协议而非人工记锁**。io_fix 状态转换的正确性依赖"改之前拿对了锁"，8.0 把这层隐性知识用 `Stateful_latching_rules` 模板显式化（FSM + latch 集合表），debug 版自动校验。**代价**：仅 debug 有效（release 编译掉），但它把"哪条转换要哪些锁"集中一处，避免散落各处的人肉记忆出错——这是 8.0 把 io_fix 从 buf_pool mutex 粗粒度保护细化成精确 latch 协议的配套工程。

**汇聚点模式**：三条刷脏路径（flush list 批刷 / LRU 批刷 / 用户线程单页同步刷）全部汇聚到 `buf_flush_write_block_low`——WAL 约束、checksum、doublewrite 写入只需在一处实现，观测/埋点（DBUG `ib_buf`）也只此一处。

**生产者-消费者**：page cleaner coordinator 算"要刷多少页"并派发 slot，worker 线程消费 slot 执行刷盘。coordinator 兼任 worker 0。

### 理论溯源

- **slot-based buffer manager**：术语与骨架直接源自 Gray & Reuter《Transaction Processing: Concepts and Techniques》（1992），源码 `buf0buf.cc` 注释明确引用。三要素（frame 槽位 / 控制块 / 页表 page_hash）即该书描述的"经典缓冲管理器"。
- **WAL（Write-Ahead Logging）**：`buf_flush_write_block_low` 写数据页前先 `log_write_up_to(flush_to_lsn, wait=true)`，是 ARIES（Mohan et al., 1992）的 no-force 规则落地：缓冲池不强制事务提交即刷脏（no-force），但刷脏前必须保证对应 redo 已 force。
- **二次机会 LRU**：old/new 分段 + `old_blocks_time` 时间窗，是"第二次机会"算法（O'Neil et al., 1993, LRU-K 的思想前身）的简化版——InnoDB 只看"首次访问后是否在时间窗内被再次访问"，而非完整的 K 次访问历史。代价是抗扫描污染能力弱于 LRU-K/ARC，但实现极简且状态机清晰。
- **buddy allocator**：压缩页 `zip.data` 内存由 `buf_pool->zip_free` 按 2 的幂大小分桶管理，源自 Buddy 系统算法，便于按页大小（1K/2K/4K/8K）快速分配回收与合并。
- **Stateful_latching_rules**：`ut0stateful_latching_rules.h` 是 InnoDB 自研的声明式 FSM+latch 协议校验框架，设计上对标形式化锁协议验证思想（把"转换需持锁"建模为状态图的边）。

### 算法与数据结构

slot-based buffer manager 的通用概念到 InnoDB 实现的对应（骨架源自 Gray & Reuter《Transaction Processing》）：

| 通用概念 | InnoDB 实现 |
|---------|------------|
| 槽位 frame | `buf_block_t::frame`（`byte*`，16KB 页帧） |
| 控制块 | `buf_block_t`（内含 `buf_page_t page` + `BPageLock lock` + `frame`） |
| 页表 page table | `buf_pool_t::page_hash`（按 `(space_id, page_no)` 索引，侵入式链地址法） |
| 空闲链表 | `buf_pool->free`（`NOT_USED` 态块） |
| 替换策略 | LRU list（old/new 两段、midpoint insertion） |

对照"固定映射式"（页号直接映射内存位置，无页表，无法动态装载）与"对象/变长式"（malloc 级粒度，内存碎片难控）——InnoDB 选 slot-based 是在分配效率、对齐、碎片可控与页大小灵活性间的折中。CMU 15-445 BusTub 是最简教学实现（`pages[]` + `page_table_` + `replacer_` + `free_list_`）。

核心数据结构：

- **page_hash**：按 `(space_id, page_no)` 的侵入式链地址法哈希表，节点是 `buf_page_t*`（`buf_page_t::hash` 是 next ptr）。`buf_block_t::page` 必须是首成员，让 hash 能统一指向 `buf_page_t`（裸压缩页）或 `buf_block_t`（有 frame 的页）。
- **flush_list**：脏页链表，按 `oldest_modification` 升序插入（首脏时定序），page cleaner 按序扫描刷盘。8.0.19+ 用 hazard pointer（`flush_hp`）+ relaxed order 降低 `flush_list_mutex` 持锁粒度。
- **LRU list**：双向链表，`LRU_old` 指针把表分成 new（头段，热）与 old（尾段，冷）两段。`buf_LRU_old_adjust_len` 在每次插入/移除后维护指针位置，使 old 段长度恒为 `LRU_old_ratio/BUF_LRU_OLD_RATIO_DIV`。
- **unzip_LRU**：压缩页解压帧的独立 LRU（只有 `FILE_PAGE` 且 `zip.data!=null` 的块进），与主 LRU 协同淘汰解压帧（内存紧张时先丢解压帧保留压缩副本，变 `ZIP_DIRTY`）。

### 他库对比与演进动机

| 维度 | InnoDB | PostgreSQL | Oracle |
|---|---|---|---|
| 替换算法 | LRU 中点（old/new + 时间窗） | CLOCK 近似（`clock_sweep`） | LRU + 多 pool（按对象大小分池） |
| 刷脏 | page cleaner 异步 + 自适应（LSN age 因子） | bgwriter + checkpointer | DBWn 进程组 |
| 预读 | 线性（连续 extent）+ 随机（extent 内驻留比例） | effective_io_concurrency 顾问 | 自动/手动 |

**演进动机**：5.6 多 instance 是为打破单 `buf_pool mutex` 争用（多核扩展性）；5.7 在线 resize 因 DBA 抱怨"改 buffer pool 要停机"；8.0 flush 无锁化因大内存下扫 flush_list 持 `buf_pool mutex` 过久拖慢前台——每次演进都对应一个被实测的瓶颈。8.0.20 doublewrite 重构因旧版 shared tablespace 双写区在 truncate 表空间时竞争，改独立文件双池可并发。

---

## 核心实现

### 主链路

读者视角（页怎么进 buffer pool）与写者视角（脏页怎么出去）两条主线：

```
读：buf_page_get_gen → page_hash 查 → 命中(buf_fix++) / 未命中(buf_LRU_get_free_block 取槽 → buf_read_page_low 异步读 → buf_page_io_complete)
写：mtr commit → buf_flush_note_modification(首脏入 flush_list) → ... → page cleaner: pc_flush_slot → buf_flush_batch → buf_flush_page → buf_flush_write_block_low(WAL+checksum+dblwr::write)
```

下面逐环节展开。

### 页面读取：`buf_page_get_gen`

入口 `buf_page_get_gen`（buf0buf.cc:4365）。`Page_fetch mode` 枚举决定行为（`NORMAL`/`SCAN`/`IF_IN_POOL`/`IF_IN_POOL_OR_WATCH`…），核心两分支：

**命中**：在 `page_hash` 找到 bpage → `buf_fix_count++`（bufferfix，钉页防淘汰，**不阻塞 I/O 状态查询**，与 io_fix 区分）→ 按 `rw_latch` 加 S/SX/X 锁 frame → 返回 block。

**未命中**：调 `buf_LRU_get_free_block` 取一个 free 槽 → `buf_page_init_for_read` 建 hash 项、置 `io_fix=BUF_IO_READ` → `buf_read_page_low`（buf0rea.cc:67）提交异步读 AIO → 当前线程等 `buf_bpage->io_fix` 变 NONE。

读完成后 IO handler 线程回调 `buf_page_io_complete`（buf0buf.cc:5608）：`io_fix` READ→NONE、`buf_fix_count--`、若被改脏则 `buf_flush_note_modification` 入 flush_list。

`buf_fix_count` vs `io_fix` 的关键区别——这是最容易混的两个"钉页"机制：

| 机制 | 谁置 | 防什么 | 是否阻塞 I/O 状态查询 |
|---|---|---|---|
| `buf_fix_count` | 读页线程（buf_page_get） | 防 LRU 淘汰/重定位 | 否（`buf_page_can_relocate` 只看 io_fix 与 fix_count，但查询 io_fix 可不持 block mutex） |
| `io_fix` | 发起 I/O 的线程 | 防"读盘/刷盘进行中被别人动这页" | 是（改 io_fix 要持规定 latch） |

### LRU 中点替换

LRU 被一个指针 `buf_pool->LRU_old` 分两段：**new 段**（头，热页，再次访问移到头）、**old 段**（尾，冷页，候选淘汰区）。常量（buf0lru.h）：

- `BUF_LRU_OLD_RATIO_DIV = 1024`（比例分母）
- `BUF_LRU_OLD_RATIO_MIN = 51`（最小 old 比例 ≈ 5%，对应 `innodb_old_blocks_pct` 最小 5）
- `BUF_LRU_OLD_MIN_LEN = 8*1024/16 = 512` 页（LRU 短于此 `LRU_old` 不初始化，退化为纯 LRU）

**新页插入**：`buf_LRU_add_block`（buf0lru.cc:1719 起）把新读入页插入 `LRU_old` 位置（old 段头部），标记 `old=true`，再调 `buf_LRU_old_adjust_len`（:1457）微调 `LRU_old` 指针使 old 段长度回到 `LRU_old_ratio/BUF_LRU_OLD_RATIO_DIV`。

**升 young**：页被再次访问时，若其"首次访问时间"距今 ≥ `innodb_old_blocks_time`（默认 1000ms），`buf_LRU_make_block_young`（:1734）把它摘除移到 new 段头部。否则留在 old 段——这是抗全表扫描污染的核心：扫描页活不过 1 秒就不会进 new 段挤热页。

```c
// buf_LRU_make_block_young：摘除后插到 LRU 头(new 段)
void buf_LRU_make_block_young(buf_page_t *bpage) {
  if (bpage->old) buf_pool->stat.n_pages_made_young++;
  buf_LRU_remove_block(bpage);
  buf_LRU_add_block_low(bpage, false);   // false=放 new 段头
}
```

**淘汰**：`buf_LRU_scan_and_free_block`（:1157）从 old 段尾部扫，跳过 io_fixed/buf_fixed 的页，找到可淘汰的干净页 `buf_LRU_free_page` 释放回 free list。`BUF_LRU_SEARCH_SCAN_THRESHOLD=100` 控制 LRU 末尾扫描窗口。

**兜底**：`buf_LRU_get_free_block`（:1300+）取不到 free 且 LRU 扫不出干净页时，被迫单页同步刷脏（`buf_flush_single_page_from_LRU`）——用户线程阻塞等刷完，对应状态变量 `Innodb_buffer_pool_wait_free`，是性能悬崖信号。

### 预读（read-ahead）

两种预读都在 `buf0rea.cc`，由 `buf_page_get_gen` 未命中路径或 io handler 触发：

**线性预读 `buf_read_ahead_linear`**（:334）：检测顺序访问。当一个 extent（`read_ahead_area`，通常 64 页）的**边界页**（首或尾）被访问时，检查该 extent 内连续访问的页数是否 ≥ 阈值：

```c
threshold = std::min(static_cast<page_no_t>(64 - srv_read_ahead_threshold),
                     buf_pool->read_ahead_area);
```

`srv_read_ahead_threshold` 默认 56，即连续访问一个 extent 内 ≥(64-56)=8 页才触发预读下一个 extent。阈值越小越激进。同时检测访问方向（asc/desc），只往同方向预读。

**随机预读 `buf_read_ahead_random`**（:157）：检测局部密集驻留。当一个 extent 内已有 ≥ `BUF_READ_AHEAD_RANDOM_THRESHOLD`（:57，动态 `5 + read_ahead_area/8`）个页在 buffer pool 时，把整个 extent 剩余页预读。受 `srv_random_read_ahead` 开关控制（**8.0 默认关闭**，线性预读默认开）。

两者都过 `BUF_READ_AHEAD_PEND_LIMIT`（:64，值 2）限流：pending 读过多时不预读，防 IO-fixed 块灌满 buffer pool。

### 脏页生成与 flush_list

mtr commit 时若页被改，`buf_flush_note_modification`（buf0flu.cc）设 `oldest_modification`（首脏时，置为 mtr end_lsn）并插入 flush_list；后续修改只更新 `newest_modification`。flush_list 按 `oldest_modification` 升序——这是 page cleaner "刷老不刷新"与 checkpoint 推进的依据。

### 脏页刷新（Flush）

三条路径全部汇聚到 `buf_flush_page`（:1277）→ `buf_flush_write_block_low`（:1174，统一汇聚点）：WAL 约束 + checksum + `dblwr::write`。

刷脏类型 `buf_flush_t`（buf0types.h:68）：

| 值 | 类型 | 发起者 | IO |
|---|---|---|---|
| 0 | `BUF_FLUSH_LRU` | page cleaner / `buf_LRU_get_free_block` | async |
| 1 | `BUF_FLUSH_LIST` | page cleaner（主路径） | async |
| 2 | `BUF_FLUSH_SINGLE_PAGE` | 用户线程（free page 不足兜底） | sync |

`sync=true` 只允许 SINGLE_PAGE（断言 :1292）。写页前先保证 newest_lsn 之前的 redo 已落盘（WAL）。

**路径1：flush list 批刷**（async, type=1，主路径）：
```
buf_flush_page_coordinator_thread (buf0flu.cc:3183)   ← 算刷脏量、派发 slot、唤醒 worker
└─ buf_flush_page_cleaner_thread (:3606)             ← worker（coordinator 兼任 worker 0）
   └─ pc_flush_slot (:2934)
      └─ buf_flush_do_batch (:2074) → buf_flush_batch (:1951)
         └─ buf_do_flush_list_batch (:1881)
            └─ buf_flush_page_and_try_neighbors (:1626)  ← 邻居页合并(innodb_flush_neighbors)
               └─ buf_flush_try_neighbors (:1482)
                  └─ buf_flush_page (:1277)  ← 置 IO_FIX(BUF_IO_WRITE)、SX lock、flush_state_mutex 统计
                     └─ buf_flush_write_block_low (:1174)  ← ★ 汇聚点(WAL+checksum+dblwr)
```

**路径2：LRU 批刷**（async, type=0）：coordinator 周期性 + `buf_LRU_get_free_block` 触发 → `buf_flush_LRU_list_batch`（:1762）→ 后半段同路径1。

**路径3：用户线程单页同步刷**（sync, type=2）：`buf_flush_single_page_from_LRU`（:2159）→ `buf_flush_page(..., BUF_FLUSH_SINGLE_PAGE, sync=true)`。频繁发生 = free page 供给不上，对应 `Innodb_buffer_pool_wait_free`。

### 刷脏前的 WAL：redo 先落盘

`buf_flush_write_block_low`（:1174）写数据页前先同步等 redo fsync（:1205-1221）：

```c
const lsn_t flush_to_lsn = bpage->get_newest_lsn();  // = newest_modification
if (log_sys->flushed_to_disk_lsn.load() < flush_to_lsn) {
    wait_stats = log_write_up_to(*log_sys, flush_to_lsn, true);  // true=同步等 fsync
}
```

- **WAL/ARIES**：数据页新版本写盘前，对应 redo 必须先落盘，否则崩溃恢复无法重放/回滚。
- **用 newest 而非 oldest**：要保证该页**所有**修改的 redo 落盘，水位必须推到 newest（最大 LSN）。`oldest_modification` 是首脏 LSN（进 flush_list 依据），`newest_modification` 是最近修改 LSN。
- 优化：先查 `flushed_to_disk_lsn < flush_to_lsn`，redo 已 flush 到更新位置就不调（避免无谓等待 + 污染 log 线程调用计数器）。

### 自适应刷脏（Adaptive Flushing）

page cleaner coordinator 每秒（`innodb_flush_sync` 周期）调 `Adaptive_flush::page_recommendation`（buf0flu.cc:2291 命名空间）算"本轮回刷多少页"。核心是按 **LSN age 因子**非线性的刷脏速率：

```c
// buf0flu.cc:2541-2547（简化）
lsn_age_factor = (age * 100.0) / limit_for_dirty_page_age;
return (static_cast<ulint>(((srv_max_io_capacity / srv_io_capacity) *
                            (lsn_age_factor * sqrt(lsn_age_factor))) /
                           7.5));
```

- `age` = current_lsn − last_checkpoint_lsn（redo 产生的速度）；`limit_for_dirty_page_age` 是脏页年龄上限。
- 用 `sqrt`（开方）做非线性：age 越接近上限刷得越凶，但前期平缓，避免抖动。
- `srv_adaptive_flushing_lwm`（低水位）：age 低于它不启用自适应刷脏；超过 `limit_for_dirty_page_age` 即使关了 adaptive flushing 也强制刷（防 checkpoint age 撞顶）。
- `srv_max_io_capacity` / `srv_io_capacity`：IO 容量上下限，自适应值在这区间。

**sync flush 模式**：当 checkpoint age 接近软上限，`log_request_sync_flush` 唤醒 page cleaner 走 sync flush（`is_sync_flush=true`），按 `lsn_avg_rate` + `buf_flush_lsn_scan_factor` 收紧 `lsn_limit`，优先刷最老脏页推 checkpoint。

### doublewrite

刷脏经 `dblwr::write`（buf0dblwr.cc）走 doublewrite：先把页写到 doublewrite buffer（独立文件双池，8.0.20+），再写数据文件。

**为什么必须 doublewrite——防半页写（torn page）**：InnoDB 页 16KB，操作系统/磁盘扇区通常 4KB（甚至 512B）。一次 16KB 写若中途崩溃，只有部分扇区落盘，页撕裂。若有 doublewrite：数据文件里的撕裂页可从 doublewrite buffer 的完整副本恢复；若无，撕裂页无 redo 保护（redo 只记逻辑修改，假设页本身完整），崩溃恢复会因 checksum 校验失败而无法重放。

8.0.20+ 重构：doublewrite 从 shared tablespace 的固定区域改为独立文件（`#ib_16384_0.dblwr` 等），双池（active + ready）轮换，允许并发写不同 instance，避免 truncate 表空间时的双写区竞争。`m_dblwr_id`（buf_page_t 字段）在写时分配 instance 序号，IO 完成时据此回收。

### 压缩页（ROW_FORMAT=COMPRESSED）三态与刷盘

同一 COMPRESSED 页在 buffer pool 三种形态：

| 状态 | zip.data | block->frame | 脏? | 在哪个 list |
|---|---|---|---|---|
| `BUF_BLOCK_ZIP_PAGE` | 压缩副本 | 无（解压帧已淘汰） | 干净 | zip_clean |
| `BUF_BLOCK_ZIP_DIRTY` | 压缩副本 | 无 | 脏 | flush_list |
| `BUF_BLOCK_FILE_PAGE` | 压缩副本 + 解压帧 | 有 | 脏/干净 | LRU (+flush_list 若脏) |

- `ZIP_PAGE`/`ZIP_DIRTY` 是裸 `buf_page_t`（非 `buf_block_t`，无 frame）。`ZIP_DIRTY` = 压缩表脏页的"省内存精简形态"：解压帧被 LRU 回收（`buf0lru.cc:1872` `b->state = b->is_dirty() ? BUF_BLOCK_ZIP_DIRTY : BUF_BLOCK_ZIP_PAGE`），只留脏的压缩副本等刷盘。
- 能这么干因压缩页修改是 `page_zip_*` **双写增量维护**（改记录时同步更新 zip.data），释放 frame 后 zip.data 仍是最新。

刷盘分支（`buf_flush_write_block_low` :1223-1257）：
- **ZIP_DIRTY**（:1232）：直接对 `zip.data` 写 `FIL_PAGE_LSN` + `verify_zip_checksum`（只校验不重算）。不调 `buf_flush_init_for_writing`，压缩数据已增量维护好。
- **FILE_PAGE**（:1243）：`frame = zip.data`（优先压缩版省 I/O），无才用 `block->frame`；调 `buf_flush_init_for_writing`（:998）按页类型更新 checksum——压缩页走 `buf_flush_update_zip_checksum`（:1044）只更新 `zip.data` checksum，**不重新压缩**。

### buf_page_t 状态机与 buf_page_in_file

`buf_page_in_file(bpage)`（buf0buf.ic:346）：控制块是否映射到表空间页（有合法 `(space_id,page_no)` 磁盘身份）。是 `state` 字段的规范解释者。

8 态两类划分（buf0buf.h:129-152）：

| in_file = true | in_file = false |
|---|---|
| `FILE_PAGE` 普通文件页（含解压帧） | `NOT_USED` free list |
| `ZIP_PAGE` 仅压缩页、干净（zip_clean） | `READY_FOR_USE` 刚从 free list 拿到未装页 |
| `ZIP_DIRTY` 仅压缩页、脏（flush_list） | `MEMORY` 纯内存对象（释放过渡态，buf0lru.cc:2322） |
| | `REMOVE_HASH` 正在释放、需先摘 AHI 的过渡态（buf0lru.cc:2265） |

- `POOL_WATCH`：watch[] 哨兵，`buf_page_in_file` 中直接 `ut_error`，出现即 bug（详见下文 Misc「Buffer Pool Watch」）。
- 只有 in_file 态，`id`、`oldest/newest_modification`、list 节点才有意义。
- 唯一容忍的例外：`REMOVE_HASH` 瞬态可进 `buf_flush_ready_for_flush_common`（:722-728），debug 版要求不持 `LRU_list_mutex`；BUF_FLUSH_LIST 下 REMOVE_HASH 不可刷（:756）。

生命周期：
```
NOT_USED ──buf_LRU_get_free_block──> READY_FOR_USE
  ──buf_page_init_for_read/buf_page_create──> FILE_PAGE
FILE_PAGE ──buf_LRU_free_page──> (有 AHI 先 REMOVE_HASH) ──> MEMORY ──> NOT_USED
```

### io_fix 状态机与 latch 协议校验

`buf_page_t` 有两个独立状态维度（勿混）：`state`（8 态，页身份/生命周期）vs `io_fix`（buf0types.h:99，4 态，I/O 进行中）。

io_fix 四态：`BUF_IO_NONE`(无I/O)/`READ`(异步读中)/`WRITE`(异步写中)/`PIN`(钉住防重定位，`buf_page_set_sticky` buf0buf.ic:480)。

`ut::Stateful_latching_rules<buf_io_fix, 3>`（ut0stateful_latching_rules.h:114）声明式定义转换图 + 每条转换要求的 latch 集合，debug 版自动校验（FSM + 形式化 latch 协议验证，8.0 无锁化重构一环）：

```
NONE ──{0,2}──> READ        发起读
READ ──{0,2}──> NONE        读完成
NONE ──{0,1,2}──> WRITE     发起写
WRITE ──{0,1,2}──> NONE     写完成
NONE ──{0}──> PIN           置 sticky
PIN  ──{0}──> NONE          取消 sticky
```

latch 编号（`get_owned_latches` buf0buf.cc:5519-5527）：
- **0** = block mutex（`buf_page_get_mutex`，页级字段）
- **1** = `flush_state_mutex`（buf0buf.h:2204，保护刷脏批次状态 `init_flush`/`n_flush`/`no_flush`，**非** flush_list 链表）
- **2** = io_responsibility 责任令牌（当前线程是否负责该页 I/O，供 io 完成线程免锁检查）

**写比读多 latch 1**：发起/完成写动 `n_flush` 计数与 `no_flush` 事件（flush_state_mutex 保护）；读不碰刷脏计数。**PIN 只要 latch 0**：只钉住防重定位，不碰 I/O 也不碰刷脏。

校验两类：
- `on_transition_to`（:5558）：`set_io_fix` 改值前（buf0buf.cc:5597 在 :5598 store 之前）校验当前线程持有该转换要求的 latch。
- `assert_latches_let_distinguish`（:5547）：读 io_fix 判断"是 A 还是 B"时，校验持有 latch 足以阻止 io_fix 从 A/B 集合内变到集合外（否则读到的值瞬间失效）。如 `is_io_fix_write()`（:5567）调用它。

纯 debug（`ut_ad`/`ut_d`/`#ifdef UNIV_DEBUG`），release 编译掉零开销。

### DBUG 观测与调试

DBUG 库（源自 Fred Fish dbug 包）条件打印：`DBUG_PRINT(keyword, args)`（my_dbug.h:181）经 `_db_keyword_` 匹配才输出；仅 debug build（WITH_DEBUG=1）有效，release 宏为空（:255）零开销。

开关：
- 启动：`mysqld --debug='d,ib_buf:t:i:F:N:o,/tmp/mysqld.trace'`（`d`=keyword 列表，`t`=函数进出 trace，`i`=线程号，`F/N`=源文件/行号，`o`=输出文件）
- 运行期（debug build）：`SET GLOBAL debug='+d,ib_buf';` / `'-d,ib_buf';`

`ib_buf` keyword 打印点（页面全生命周期）：

| 位置 | 输出 | 含义 |
|------|------|------|
| buf0buf.cc:5067 | `create page S:P` | 页创建 |
| buf0rea.cc:104+ | read-ahead 系列 | 预读 |
| buf0flu.cc:1537 | `flush S:low..high` | 邻居页合并范围 |
| **buf0flu.cc:1183** | `flush <sync\|async> <type> page S:P` | **每页写盘前（汇聚点）** |
| buf0flu.cc:1980/1992 | `flush N completed` | batch 汇总 |
| buf0lru.cc:873 | `evict page S:P state N` | 驱逐 |
| buf0lru.cc:1801 | `free page S:P` | 释放 |

调试要点：`flush sync 2` 高频 → 单页同步刷，free page 不足（性能悬崖）；同一 `S:P` 高频 flush → 热点页/checkpoint 压力；type=1 应占绝对多数。release build 替代：`innodb_metrics` 的 `buffer_flush_*`/`buffer_LRU_batch_*`、`SHOW ENGINE INNODB STATUS` BUFFER POOL 段。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_buffer_pool_size` | 128MB | Global | buffer pool 总大小，可在线 resize（chunk 增删） |
| `innodb_buffer_pool_instances` | 1（或按 size 自动） | Global | buffer pool 实例数，每实例独立 LRU/page_hash |
| `innodb_old_blocks_pct` | 37 | Global | LRU old 段占比（5-95），调 100 退化为纯 LRU |
| `innodb_old_blocks_time` | 1000(ms) | Global | 页须在 old 段驻留 ≥此值且被再次访问才升 young |
| `innodb_read_ahead_threshold` | 56 | Global | 线性预读阈值（extent 内连续访问页数 ≥ 64-threshold 才触发） |
| `innodb_random_read_ahead` | OFF | Global | 随机预读开关（默认关，线性预读默认开） |
| `innodb_adaptive_flushing` | ON | Global | 自适应刷脏（按 LSN age 因子动态调刷脏速率） |
| `innodb_adaptive_flushing_lwm` | 10(%) | Global | 自适应刷脏低水位，age 低于它不启用 |
| `innodb_flush_neighbors` | 1 | Global | 邻居页合并（0=关，1=同 extent，2=连续 extent） |
| `innodb_io_capacity` / `innodb_io_capacity_max` | 200 / 2000（视磁盘） | Global | IO 容量上下限，自适应刷脏值在此区间 |
| `innodb_flush_sync` | ON | Global | checkpoint age 接近上限时启用 sync flush |
| `innodb_buffer_pool_dump_at_shutdown` | ON | Global | 关机时 dump 热页列表用于预热 |
| `innodb_buffer_pool_load_at_startup` | ON | Global | 启动时 load dump 文件预热 |

### 状态变量

| 变量名 | 说明 |
|--------|------|
| `Innodb_buffer_pool_wait_free` | 用户线程被迫单页同步刷等待次数（性能悬崖信号，应为 0） |
| `Innodb_buffer_pool_pages_dirty` | 脏页数 |
| `Innodb_buffer_pool_pages_total` / `_free` / `_data` | 总页/空闲/数据页数 |
| `Innodb_buffer_pool_read_requests` / `Innodb_buffer_pool_reads` | 逻辑读/物理读次数（命中率 = 1 - reads/requests） |
| `Innodb_buffer_pool_pages_made_young` / `_not_young` | 升 young / 未升 young 计数（调 old_blocks_pct/time 的依据） |
| `Innodb_buffer_pool_bytes_dirty` / `_data` | 脏页/数据页字节数 |

---

## Misc

### Buffer Pool Watch（哨兵机制）

purge 删 secondary index 的 delete-marked 记录时，页若不在 buffer pool，`buf_pool_watch_set`（buf0buf.cc:2926，仅 purge 线程用）在 page_hash 插一个 `watch[]` 哨兵占位，把删除缓冲到 change buffer（`ibuf_insert IBUF_OP_DELETE`）避免 purge 读盘阻塞 OLTP。页读入时 change buffer merge 删除 + `watch_remove` 替换哨兵。

哨兵状态机：`buf_pool->watch[]`（`BUF_POOL_WATCH_SIZE` = 最大 purge 线程数）。`POOL_WATCH`=空闲态（不在 hash/无 page_id/zip.data=null/buf_fix=0）；`ZIP_PAGE`=激活态（在 page_hash 占位/有 page_id/buf_fix>=1/zip.data 仍 null 伪装压缩页无数据）。`buf_page_in_file` 对 `POOL_WATCH` 直接 `ut_error`：空闲哨兵不在任何 hash/list，正常路径拿不到，流到此处是 use-after-free bug，`ut_error` 防御暴露。区分激活哨兵 vs 真实压缩页靠 `buf_pool_watch_is_sentinel`（地址在 watch[] 范围 + zip.data=null）。

### slot-based 布局细节

`buf_chunk_init`（buf0buf.cc:993）：一次分配整块 chunk，头部放 `buf_block_t` 控制块数组，后面紧跟 frame 数组；循环 `buf_block_init` 后 `block++`、`frame += UNIV_PAGE_SIZE`，控制块与 frame 一一对应连续排列。`buf_block_t::page` 必须是首成员（buf0buf.h:1700），让 page_hash 能统一指向 `buf_page_t`（裸压缩页）或 `buf_block_t`（有 frame 的页）。CMU 15-445 BusTub 是最简教学实现（`pages[]` + `page_table_` + `replacer_` + `free_list_`）。

### 术语辨析

- **`state` vs `io_fix`**：前者管"页身份/在哪个 list"（8 态），后者管"I/O 进行否"（4 态），两个正交维度，勿混。
- **`buf_fix_count` vs `io_fix`**：前者读页线程钉页防淘汰（不阻塞 I/O 状态查询），后者发起 I/O 的线程置位（改它要持规定 latch）。
- **`flush_list_mutex` vs `flush_state_mutex`**：前者保护 flush_list 链表本身（节点增删），后者保护刷脏批次状态（`init_flush`/`n_flush`/`no_flush` 计数与事件）——io_fix WRITE 转换要的是后者。
- **`oldest_modification` vs `newest_modification`**：前者首脏 LSN（进 flush_list 依据，恒定），后者最近修改 LSN（WAL 刷脏水位用此，单调增）。

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `storage/innobase/include/buf0buf.ic:346` | `buf_page_in_file` 定义 |
| `storage/innobase/include/buf0buf.h:129` | `buf_page_state` 8 态枚举 |
| `storage/innobase/buf/buf0buf.cc:4365` | `buf_page_get_gen` 读页主入口 |
| `storage/innobase/buf/buf0buf.cc:993` | `buf_chunk_init` chunk/控制块布局 |
| `storage/innobase/buf/buf0buf.cc:5608` | `buf_page_io_complete` IO 完成回调 |
| `storage/innobase/buf/buf0buf.cc:5475` | io_fix 状态机 + latch 协议校验器 |
| `storage/innobase/buf/buf0buf.cc:5519` | `get_owned_latches`：latch 0/1/2 映射 |
| `storage/innobase/buf/buf0buf.cc:5587` | `set_io_fix`：校验挂载点(:5597 在 store 前) |
| `storage/innobase/buf/buf0lru.cc:1457` | `buf_LRU_old_adjust_len` 中点指针维护 |
| `storage/innobase/buf/buf0lru.cc:1734` | `buf_LRU_make_block_young` 升 young |
| `storage/innobase/buf/buf0lru.cc:1157` | `buf_LRU_scan_and_free_block` 淘汰扫描 |
| `storage/innobase/buf/buf0lru.cc:1872` | 压缩页解压帧淘汰→ZIP_DIRTY 形态转换 |
| `storage/innobase/buf/buf0lru.cc:2265` | 释放页先置 REMOVE_HASH |
| `storage/innobase/buf/buf0rea.cc:67` | `buf_read_page_low` 异步读 |
| `storage/innobase/buf/buf0rea.cc:157` | `buf_read_ahead_random` 随机预读 |
| `storage/innobase/buf/buf0rea.cc:334` | `buf_read_ahead_linear` 线性预读 |
| `storage/innobase/buf/buf0rea.cc:57` | `BUF_READ_AHEAD_RANDOM_THRESHOLD` 动态阈值 |
| `storage/innobase/buf/buf0flu.cc:1174` | `buf_flush_write_block_low` 刷脏统一汇聚点 |
| `storage/innobase/buf/buf0flu.cc:1205` | 刷脏前 WAL：`log_write_up_to` 同步等 redo fsync |
| `storage/innobase/buf/buf0flu.cc:998` | `buf_flush_init_for_writing` checksum 更新 |
| `storage/innobase/buf/buf0flu.cc:1232` | BUF_BLOCK_ZIP_DIRTY 刷盘分支 |
| `storage/innobase/buf/buf0flu.cc:1243` | BUF_BLOCK_FILE_PAGE 刷盘分支 |
| `storage/innobase/buf/buf0flu.cc:1277` | `buf_flush_page`：置 IO_FIX、SX lock、batch 统计 |
| `storage/innobase/buf/buf0flu.cc:1626` | `buf_flush_page_and_try_neighbors` 邻居合并 |
| `storage/innobase/buf/buf0flu.cc:2159` | `buf_flush_single_page_from_LRU` 单页同步刷 |
| `storage/innobase/buf/buf0flu.cc:2291` | `Adaptive_flush` 命名空间（自适应刷脏） |
| `storage/innobase/buf/buf0flu.cc:2541` | 自适应刷脏公式（sqrt 非线性） |
| `storage/innobase/buf/buf0flu.cc:2934` | `pc_flush_slot` worker 刷脏入口 |
| `storage/innobase/buf/buf0flu.cc:3183` | `buf_flush_page_coordinator_thread` coordinator |
| `storage/innobase/include/buf0lru.h:59` | `BUF_LRU_OLD_MIN_LEN=512` / `BUF_LRU_OLD_RATIO_DIV=1024` |
| `storage/innobase/include/buf0types.h:68` | `buf_flush_t`：LRU=0/LIST=1/SINGLE_PAGE=2 |
| `storage/innobase/include/buf0types.h:99` | `buf_io_fix` 4 态枚举 |
| `storage/innobase/include/ut0stateful_latching_rules.h:114` | `Stateful_latching_rules` 模板 |
| `include/my_dbug.h:181` | `DBUG_PRINT` 宏（release :255 为空） |

---

## 参考

**经典文献**
- Jim Gray & Andreas Reuter. *Transaction Processing: Concepts and Techniques*. Morgan Kaufmann, 1992. —— slot-based buffer manager 骨架，`buf0buf.cc` 注释直接引用。
- C. Mohan, Don Haderle, Bruce Lindsay, Hamid Pirahesh, Peter Schwarz. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollback Using Write-Ahead Logging*. ACM TODS 17(1), 1992. —— no-force/steal 规则，刷脏前 WAL 的理论依据。
- Elizabeth J. O'Neil, Patrick E. O'Neil, Gerhard Weikum. *The LRU-K Page Replacement Algorithm For Database Disk Buffering*. SIGMOD 1993. —— InnoDB 中点 LRU 是其简化版（只看首次访问+时间窗，非完整 K 次历史）。
- Mark S. Massagli & Scott K. Warren. *Buddy System* 内存分配算法——压缩页 `zip_free` 按页大小分桶管理。

**官方文档**
- *MySQL 8.0 Reference Manual → The InnoDB Storage Engine → Buffer Pool*（架构、LRU、预读、刷脏、自适应哈希）
- *MySQL 8.0 Reference Manual → InnoDB Initialization and Configuration → Configuring Buffer Pool Size*
- WorkLog: [WL#10314](https://dev.mysql.com/worklog/task/?id=10314) InnoDB: separate double write buffer (8.0.20 doublewrite 重构)
- WorkLog: [WL#11652](https://dev.mysql.com/worklog/task/?id=11652) InnoDB: Reduce lock contention on flush list (8.0.19 flush 无锁化)

**相关文档**
- 上游（buffer pool 谁调用）见 [`../server/handler.md`](../server/handler.md)（handler 取行下推到引擎）
- redo 刷盘与 LSN 体系、崩溃恢复见 [`redo_log.md`](redo_log.md)
- B-tree 页内操作与 AHI 见 [`btr.md`](btr.md)
- 行读取主循环 `row_search_mvcc` 见 [`row_search.md`](row_search.md)
