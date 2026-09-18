# InnoDB 崩溃恢复深度解析

> 基于 MySQL 8.0.39 源码，涵盖两阶段恢复模型（redo 前滚 / undo 回滚）、恢复入口与扫描驱动、块流→记录流的解析状态机、mtr 两遍扫描、`recv_sys_t` 恢复上下文、hash 聚合与按页幂等应用、文件级 redo、clone/MEB 分支、`innodb_force_recovery` 退化档位。
>
> **边界**：本篇讲**崩溃恢复**——即 redo 与 undo 如何把库带回一致状态。redo 自身的格式、LSN 序号、checkpoint 推进、无锁写路径见 [`redo_log.md`](redo_log.md)；事务提交状态裁决见 [`trx.md`](trx.md)；undo 的管理与 purge 见 [`undo_log.md`](undo_log.md)。

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [主链路](#主链路)
    - [checkpoint：恢复的起点](#checkpoint恢复的起点)
    - [recv_sys_t：恢复上下文的完整结构](#recv_sys_t恢复上下文的完整结构)
  - 扫描与解析（redo 前滚）
    - [恢复入口：recv_recovery_from_checkpoint_start](#恢复入口recv_recovery_from_checkpoint_start)
    - [扫描驱动：recv_recovery_begin](#扫描驱动recv_recovery_begin)
    - [扫描主循环：recv_scan_log_recs](#扫描主循环recv_scan_log_recs)
    - [解析：mtr 分组与两遍扫描](#解析mtr-分组与两遍扫描)
  - 应用与协同
    - [应用：hash 聚合与按页重放](#应用hash-聚合与按页重放)
    - [单页重放：recv_recover_page_func 的幂等](#单页重放recv_recover_page_func-的幂等)
    - [redo 与 undo 的协同](#redo-与-undo-的协同)
  - 特殊分支
    - [文件级 redo 与 DDL 在恢复期的处理](#文件级-redo-与-ddl-在恢复期的处理)
    - [clone / MEB 数据目录的恢复分支](#clone--meb-数据目录的恢复分支)
- [相关的系统变量/状态变量](#相关的系统变量/状态变量)
- [Misc](#Misc)
- [参考](#参考)

---

## 概述

### 是什么

崩溃恢复是 InnoDB 启动时把数据文件从"崩溃瞬间的磁盘状态"带回"事务一致状态"的过程，分两阶段：**redo 前滚**（把库恢复到崩溃那一刻的物理状态）→ **undo 回滚**（撤销当时尚未提交的事务）。

### 用途

一个反直觉的事实是：**事务提交成功返回后，数据文件里可能一行都没改**。提交只保证 redo 落盘（`innodb_flush_log_at_trx_commit=1`），数据页仍是 buffer pool 里的脏页，可能几秒后才被刷盘——这正是 WAL 的精髓（见 [`redo_log.md`](redo_log.md)）。于是宕机后磁盘必然处于"redo 有、数据页没更新"的中间态，恢复就是补齐这一差距，并清理那些没提交却已部分落地的修改。

它要同时兑现 ACID 的两个字母：

- **D**（持久性）：已提交事务的修改重启后依然存在 → 由 redo 前滚保证
- **A**（原子性）：未提交事务的修改不残留 → 由 undo 回滚保证

恢复位于 `srv_start` 之内，**在引擎对外提供服务之前**；未完成时 mysqld 不接受连接。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | 恢复期页 hash 为单级全局 `recv_sys->addr_hash`；redo 为固定 2 个循环文件；解析按 block 粒度进行 |
| 8.0 | 页 hash 改为**两级** `recv_sys->spaces`（按 space_id 分桶，每桶自带 heap）；写路径无锁化后，恢复侧改以 LSN 水位与块头 `epoch_no` 判定数据完整性；新增 `recv_writer` 线程专职恢复期腾页；新增两遍扫描 `saved_recs` 优化 |
| 8.0.28 | redo record 新格式（新增 67–76 号类型），恢复需同时兼容旧格式 |
| 8.0.29 | prepared undo 状态拆分：`TRX_UNDO_PREPARED` 取 6，值 5 保留为 `TRX_UNDO_PREPARED_80028` 兼容旧版本数据 |
| 8.0.30 | redo 改为 32 个文件循环（`#innodb_redo/#ib_redoN`），扫描需跨文件定位；checkpoint 页固定在第一个文件；`recv_find_max_checkpoint` 需先定位 `checkpoint_lsn` 落在哪个文件 |

---

## 理论基础

### 设计思想与权衡

**1. Repeating history：连未提交事务一起重放**

ARIES 的核心主张是**重演历史**——redo 阶段不区分事务是否已提交，把日志里的修改**全部**重放，恢复到崩溃那一刻的物理状态，再由 undo 阶段清理未提交的。

被放弃的方案是"只重放已提交事务"。它看似更快，实则要求 redo 阶段就能判定事务边界，而 redo 里**根本没有 begin/commit 标记**（`mlog_id_t` 76 种类型里没有任何事务边界类型）。要做到就得在 redo 里额外维护事务表，或在扫描时先做预分析——前者拖慢正常写入，后者只是把复杂度搬进恢复路径。MySQL 选 repeating history：**正常运行零负担，恢复时多做的功由 undo 阶段承担**。

代价：恢复时间与"崩溃时活跃事务量"正相关，而不仅是已提交数据量。

**2. ★ checkpoint 不记脏页表：用"多扫 redo"换"checkpoint 极轻量"**

ARIES 原型的 checkpoint 记录 **DPT（Dirty Page Table）**与**事务表**，使 Analysis 阶段能精确定位重放起点。而 MySQL 的 checkpoint 页**只写一个有效字段 `checkpoint_lsn`**（8.0.30 起连 `checkpoint_no` 都废弃了；详见 [`redo_log.md`](redo_log.md) checkpoint 章），**没有 DPT**。

后果是恢复起点必然保守：必须从 `checkpoint_lsn` 一路扫到日志尾，`checkpoint_age` 有多大就扫多少 redo。这是**用恢复时间换正常运行时的写放大**——checkpoint 从"可能要刷一批元数据"的重操作，退化成"写两个 header 页 + 一次 fsync"。

这条权衡直接解释了 redo_log.md 里那套阈值模型的动机：`adaptive_flush_min_age` / `max_age` / `aggressive_checkpoint_min_age` 三级加速刷脏，本质都是**压小 checkpoint_age 以缩短未来的恢复时间**。缓解手段是幂等重放（第 4 条），它让"多扫"不至于变成"多做"。

**3. 先收集后应用：把随机 IO 聚成批量 IO**

扫描全程只解析，把 redo 按 (space_id, page_no) 聚进 hash，扫完再按页批量应用。若边扫边应用，同一页的修改会随日志顺序被反复读入 buffer pool——日志顺序是"写入顺序"，页顺序是"物理顺序"，两者不相关，等于把顺序扫描退化成随机访问。

代价：**hash 必须能装下 checkpoint_age 内所有被修改的页**，这是恢复期最大的内存不确定性。撑不住时的退化路径是中途触发 apply 腾空间（见「应用」一节），并连带禁掉 change buffer 操作。

**4. 幂等重放：靠页上的 FIL_PAGE_LSN 去重**

每个数据页头记 `FIL_PAGE_LSN`——最后修改它的 redo 的 end_lsn。应用时若 record 的 start_lsn < 页 LSN，说明该修改已落盘，跳过。

这让恢复**可重入**：恢复中再崩，重启重跑结果一致。也是"多扫 redo"能被接受的关键。

隐含假设是**页 LSN 可信**。若页因半页写损坏，LSN 也不可信，此时先靠 doublewrite buffer 修复页（见 [`buffer_pool.md`](buffer_pool.md)），不是恢复逻辑的职责。

**5. mtr 原子性在恢复侧的兑现**

mtr 是一组页修改的原子单位。恢复时，多记录 mtr 若不见结尾标记 `MLOG_MULTI_REC_END`，**整组丢弃**——宁可少应用一个完整 mtr，也绝不应用半个。这由两遍扫描保证：第一遍只确认"整组都在缓冲里"，第二遍才入 hash。

**6. 退化路径：innodb_force_recovery**

恢复本身失败（页损坏、redo 损坏、内存不够）时，MySQL 提供 0–6 共 7 档强制恢复，逐级**放弃更多恢复工作**换取能启动。它们是上述每条设计的"反向开关"，也印证了各阶段的独立性。

> 失效场景集中在此：checkpoint_age 长期偏大 ⇒ 恢复时间不可控（第 2 条）；hash 撑爆 ⇒ 中途 apply + 禁 ibuf（第 3 条）；页损坏且 doublewrite 也失效 ⇒ 只能靠 `innodb_force_recovery` 降级启动（第 4 条）。

### 理论溯源

**ARIES**（Mohan, Haderle, Lindsay, Pirahesh, Schwarz — *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*, ACM TODS 1992）。三阶段模型与 MySQL 的对应：

| ARIES | MySQL 8.0 对应 | 说明 |
|---|---|---|
| Analysis | **被省略**（由 checkpoint 取代） | checkpoint 不记 DPT/事务表，故无 Analysis 阶段；起点直接取 `checkpoint_lsn` |
| Redo | `recv_scan_log_recs` + `recv_apply_hashed_log_recs` | repeating history，重放全部含未提交 |
| Undo | `trx_rollback_or_clean_recovered` | 扫 undo 表空间重建事务列表，回滚未提交者 |

**WAL**（Gray & Reuter, *Transaction Processing*）：恢复的全部前提是"先日志后数据"。MySQL 的兑现点有二：提交时 redo 必先落盘；刷脏页前必先确保对应 redo 已落盘。

**Physiological logging**：物理到页、页内逻辑。恢复时按页重放、页内按 record 语义解释，兼顾日志体积与幂等性——这是"页 LSN 去重"能成立的前提（纯逻辑日志无法用页 LSN 判断是否需要重放）。

### 算法与数据结构

| 结构 | 选择理由 |
|---|---|
| **两级 hash**：`unordered_map<space_id, Space>`，每桶内 `unordered_map<page_no, recv_addr_t*>` | 按表空间粒度整体释放——DROP/删表空间的 redo 直接丢整桶；每桶独立 `mem_heap_t`，避免全局 heap 长期膨胀 |
| **解析缓冲**（2MB，可扩容） | 块是 512B 定长、record 变长且**可跨块**；缓冲把块数据区拼成连续 record 流，让解析器不必处理块边界 |
| **四个 lsn 游标** | `checkpoint_lsn`（起点）→ `scanned_lsn`（已读入）→ `recovered_lsn`（已解析）→ 应用进度，各司其职 |
| **`saved_recs` 缓存**（上限 8K 条） | 两遍扫描时避免二次解析：以 32B × 8K ≈ 256KB 固定内存，换 1G redo 扫描**约 1.8 倍**提速（源码注释给出的实测） |
| **批量读** `RECV_SCAN_SIZE`（64KB） | 一次读 128 个 block，摊薄扫描 IO |
| **`recv_max_page_lsn`** | 记录应用中见到的最大页 LSN；收尾时与 `m_scanned_lsn` 对账，两者不匹配说明扫描漏了数据 |

### 他库对比与演进动机

| | MySQL / Oracle | PostgreSQL |
|---|---|---|
| 恢复阶段 | 前滚 + **显式回滚**（两阶段） | 只前滚，**无回滚阶段** |
| 未提交数据 | 前滚照样写入，随后由 undo 撤销 | 前滚照样写入，靠 **clog 提交标记 + MVCC 可见性**让它们"看不见"，由 vacuum 清理 |
| 半页写防护 | **doublewrite buffer** | **full-page writes**（checkpoint 后首次改页写整页进 WAL） |
| 恢复起点 | 最近的 checkpoint | 最近 checkpoint 的 redo point |

分歧根源在 MVCC 实现：MySQL/Oracle 用**独立 undo 段**存旧版本，恢复必须显式回滚才能回收；PostgreSQL 把旧版本留在堆表内、用 clog + 事务可见性判定，恢复只需前滚。代价是 PG 需要 vacuum 兜底，MySQL 则在恢复路径上多一个阶段。

**为什么演进到今天**：8.0 的重写（redo 无锁化 + 32 文件）改变了恢复侧的两个前提——完整性判定从"块是否写满"变成"LSN 水位是否推进"（并发 mtr 会留下空洞），扫描从"在两个大文件里循环"变成"跨 32 个文件定位"。`recv_sys->spaces` 两级 hash、`epoch_no` 校验、`saved_recs` 优化都是这一轮重写的配套产物。

---

## 核心实现

### 主链路

```
srv_start（mysqld 启动，引擎对外服务之前）
  │
  ├─ recv_recovery_from_checkpoint_start(log, flushed_lsn)
  │    ├─ buf_flush_init_flush_rbt()            恢复期 flush_list 用红黑树加速插入
  │    ├─ recv_find_max_checkpoint()            遍历所有文件×两个槽位，取 checkpoint_lsn 最大者
  │    ├─ recv_init_crash_recovery()            置 recv_needed_recovery，建 recv_sys，起 recv_writer
  │    ├─ recv_recovery_begin(log, checkpoint_lsn)
  │    │    └─ while (!finished)
  │    │         ├─ recv_read_log_seg()         一次读 64KB
  │    │         └─ recv_scan_log_recs()        扫描 → 解析 → 入 spaces hash
  │    │              ├─ recv_sys_add_to_parsing_buf()  逐块喂进解析缓冲
  │    │              ├─ recv_parse_log_recs()          → recv_single_rec / recv_multi_rec
  │    │              └─ recv_apply_hashed_log_recs()   hash 内存超阈值时中途应用腾空间
  │    ├─ log_start(log, checkpoint_lsn, recovered_lsn)
  │    └─ log_files_next_checkpoint()           把 checkpoint 也写进另一个 header（双保险）
  │
  ├─ 脏页刷盘 → 引擎进入运行态（此时未提交事务的修改仍在）
  │
  ├─ recv_recovery_from_checkpoint_finish()     停 recv_writer、回传 DD 动态元数据
  └─ trx_rollback_or_clean_recovered()         ★ undo 阶段：重建事务列表，回滚未提交事务
```

注意 `trx_rollback_or_clean_recovered` 由 **`srv_start`** 调用，不在 `recv_recovery_from_checkpoint_finish` 内部——前滚（redo）与回滚（undo）在代码上是两个独立调用点。

### checkpoint：恢复的起点

> checkpoint 的**推进机制**（三上限、触发条件、与刷脏的阈值协同）见 [`redo_log.md`](redo_log.md) checkpoint 章。本节讲**恢复视角**关心的四件事：它凭什么当起点、它在文件里的位置、8.0.30 多文件下怎么找、以及它的质量如何决定恢复耗时。

**1. 凭什么当起点**：`last_checkpoint_lsn` 的语义是"所有 `oldest_modification <` 它的脏页都已刷盘"。因此 checkpoint 之前的 redo 所描述的修改**已经不在内存里**，重放它们没有意义（幂等机制也会跳过）。恢复只需从 `checkpoint_lsn` 扫到日志尾。

**2. 在文件里的位置**：每个 redo 文件头都是 2KB（4 个 block），block 1 = `LOG_CHECKPOINT_1`、block 3 = `LOG_CHECKPOINT_2`，两个槽位**交替写**以防写一半崩溃。但源码注释明写 "This field is only defined in the first log file"——**只有第一个文件的 checkpoint 槽位会被真正写入**（log0constants.h:164-176）。8.0.30 起 header 里**只有 `checkpoint_lsn` 一个有效字段**，`checkpoint_no` 已废弃。

**3. 8.0.30 多文件下怎么找**：`recv_find_max_checkpoint` 仍**遍历所有 32 个文件**逐个读，靠三道过滤保证正确：① `checkpoint_lsn == 0` 跳过（未写过的槽位）；② `file.contains(checkpoint_lsn)` 校验 lsn 必须落在该文件的 `[m_start_lsn, m_end_lsn)` 区间，否则报错并跳过该文件；③ 取**全局 lsn 最大**者。这样即使第一个文件损坏或槽位陈旧，也不会误用一个不属于该文件的 lsn。

**4. 质量决定恢复耗时**：因为 checkpoint **不记 DPT**（理论基础第 2 条），恢复只能保守地从 `checkpoint_lsn` 扫到日志尾，`checkpoint_age` 就是**恢复要扫的 redo 量**。redo_log.md 里那套 `adaptive_flush_*` / `aggressive_checkpoint_min_age` 阈值模型与 `log_free_check` 用户线程闸，本质都是在运行时把 age 压住，**给未来的崩溃恢复定时**。

**逐行剖析 `recv_find_max_checkpoint`（两个重载）**：

单文件版——在一个文件里读两个槽位，取 lsn 大者：

```cpp
[[nodiscard]] static bool recv_find_max_checkpoint(
    log_t &, Log_file_handle &file_handle,
    Log_checkpoint_location &checkpoint) {
  bool found = false;
  checkpoint = {};                                    // ① 输出参数先清零

  for (auto checkpoint_header_no : {Log_checkpoint_header_no::HEADER_1,
                                    Log_checkpoint_header_no::HEADER_2}) {
    Log_checkpoint_header checkpoint_header;
    const dberr_t err = log_checkpoint_header_read(   // ② 读槽位（内含 checksum 校验）
        file_handle, checkpoint_header_no, checkpoint_header);
    if (err != DB_SUCCESS) {
      ut_a(err == DB_CORRUPTION);                     // ③ IO 错误直接崩；只有 CRC 损坏才允许跳过
      continue;
    }

    const lsn_t checkpoint_lsn = checkpoint_header.m_checkpoint_lsn;
    if (checkpoint_lsn == 0) {
      continue;                                       // ④ 槽位从未写过 → 跳过
    }                                                 //    （非第一个文件的槽位都走这里）

    if (!found || checkpoint_lsn > checkpoint.m_checkpoint_lsn) {
      ut_a(checkpoint_lsn >= LOG_START_LSN);          // ⑤ lsn 必须合法
      found = true;
      checkpoint.m_checkpoint_file_id = file_handle.file_id();
      checkpoint.m_checkpoint_header_no = checkpoint_header_no;
      checkpoint.m_checkpoint_lsn = checkpoint_lsn;   // ⑥ ★ 比的是 lsn，不是 checkpoint_no
    }
  }
  return found;                                       // ⑦ 没找到任何有效槽位 → false
}
```

全文件版——遍历所有 redo 文件，再取全局最大：

```cpp
static bool recv_find_max_checkpoint(log_t &log,
                                     Log_checkpoint_location &checkpoint) {
  bool found = false;
  checkpoint = {};

  log_files_for_each(log.m_files, [&](const Log_file &file) {   // ① 遍历全部 32 个文件
    auto file_handle = file.open(Log_file_access_mode::READ_ONLY);
    ut_a(file_handle.is_open());                                // ② 打不开直接崩

    Log_checkpoint_location checkpoint_in_file;

    if (!recv_find_max_checkpoint(log, file_handle, checkpoint_in_file)) {
      return;                                                   // ③ 该文件无有效 checkpoint
    }

    if (!file.contains(checkpoint_in_file.m_checkpoint_lsn)) {  // ④ ★ 关键校验
      ib::error(ER_IB_MSG_RECOVERY_CHECKPOINT_OUTSIDE_LOG_FILE, //    lsn 必须落在本文件区间
                ...);                                           //    否则报错并跳过该文件
      return;
    }

    if (!found ||
        checkpoint_in_file.m_checkpoint_lsn > checkpoint.m_checkpoint_lsn) {
      found = true;
      checkpoint = checkpoint_in_file;                          // ⑤ 取全局 lsn 最大者
    }
  });

  return found;
}
```

> ★ **常见错误**：很多资料（含基于 5.7 的月报）写"取 `checkpoint_no` 大者"。8.0.30 起 `Log_checkpoint_header` 已无该字段（log0types.h:236 只剩 `m_checkpoint_lsn`），**8.0.39 比的是 lsn**。

### 恢复入口：recv_recovery_from_checkpoint_start

```cpp
dberr_t recv_recovery_from_checkpoint_start(log_t &log, lsn_t flush_lsn) {
  buf_flush_init_flush_rbt();

  if (srv_force_recovery >= SRV_FORCE_NO_LOG_REDO) {
    /* We leave redo log not started and this is read-only mode. */
    ut_a(log.sn == 0);
    ut_a(srv_read_only_mode);
    return DB_SUCCESS;
  }

  recv_recovery_on = true;
  ...
  if (!recv_find_max_checkpoint(log, checkpoint)) {
    ib::error(ER_IB_MSG_RECOVERY_CHECKPOINT_NOT_FOUND);
    return DB_ERROR;
  }
  ...
  log.last_checkpoint_lsn.store(checkpoint.m_checkpoint_lsn);
  ...
  if (checkpoint_lsn != flush_lsn) {
    if (!recv_needed_recovery) {
      if (srv_read_only_mode) { return DB_ERROR; }   // 只读模式不能做恢复
      recv_init_crash_recovery();
    }
  }

  err = recv_recovery_begin(log, checkpoint_lsn);
  ...
  recovered_lsn = recv_sys->recovered_lsn;
  log.recovered_lsn = recovered_lsn;
  ...
  if ((recv_sys->found_corrupt_log && srv_force_recovery == 0) ||
      recv_sys->found_corrupt_fs) {
    return DB_ERROR;
  }
  ...
  err = log_start(log, checkpoint_lsn, recovered_lsn, false);
  ...
  if (!srv_read_only_mode) {
    log.next_checkpoint_header_no =
        log_next_checkpoint_header(checkpoint.m_checkpoint_header_no);
    err = log_files_next_checkpoint(log, checkpoint_lsn);
  }
  recv_sys->apply_log_recs = true;
  return DB_SUCCESS;
}
```

逐段说清：

- **第一道闸 `SRV_FORCE_NO_LOG_REDO`（=6）**：直接跳过整个前滚，且断言 `log.sn == 0` 与只读模式——**6 档不是"少做一点"，是彻底不做 redo，必须只读启动**
- **`recv_find_max_checkpoint` 失败即 DB_ERROR**：找不到可用 checkpoint 就拒绝启动。8.0.30 后 checkpoint_lsn 必须能定位到某个 redo 文件（`log.m_files.find` 找不到直接 `ut_error`）
- **`checkpoint_lsn != flush_lsn` 是"需要恢复"的判据**：`flush_lsn` 来自 redo 文件头记录的干净关闭点。相等说明上次是干净关闭，无需前滚；不等才 `recv_init_crash_recovery()`。这也解释了为什么**正常重启也有"recovery"代码路径**——它只是走完发现无事可做
- **只读模式的三重拦截**：需要恢复但 `srv_read_only_mode` → DB_ERROR；`recovered_lsn < checkpoint_lsn` 且非只读 → `ut_error`
- **`found_corrupt_log && srv_force_recovery == 0` → 失败**：发现坏 redo 时，只有显式设了强制恢复档位才放行
- **收尾再写一次 checkpoint**：把同一个 `checkpoint_lsn` 写进**另一个** checkpoint header，使两个 header 内容一致。这是防御——万一下一次 checkpoint 写一半崩溃，还有一个完好的 header 可用
- **`apply_log_recs = true` 后返回**：此时 hash 里的 redo 已应用完毕，后续用户事务的 redo 由 io-handler 边读边应用

### 扫描驱动：recv_recovery_begin

```cpp
static dberr_t recv_recovery_begin(log_t &log, const lsn_t checkpoint_lsn) {
  recv_sys->parse_start_lsn = 0;
  recv_sys->bytes_to_ignore_before_checkpoint = 0;
  recv_sys->checkpoint_lsn = checkpoint_lsn;
  recv_sys->scanned_lsn = checkpoint_lsn;
  recv_sys->recovered_lsn = checkpoint_lsn;
  recv_sys->previous_recovered_lsn = checkpoint_lsn;
  recv_sys->last_block_first_mtr_boundary = 0;
  recv_sys->scanned_epoch_no = 0;
  ...
  ulint max_mem =
      UNIV_PAGE_SIZE * (buf_pool_get_n_pages() -
                        (recv_n_pool_free_frames * srv_buf_pool_instances));

  lsn_t start_lsn =
      ut_uint64_align_down(checkpoint_lsn, OS_FILE_LOG_BLOCK_SIZE);

  bool finished = false;
  while (!finished) {
    const lsn_t end_lsn =
        recv_read_log_seg(log, log.buf, start_lsn, start_lsn + RECV_SCAN_SIZE);
    if (end_lsn == 0) { return DB_ERROR; }
    if (end_lsn == start_lsn) { break; }   // 刚写完一个文件、下一个还没建

    finished = recv_scan_log_recs(log, max_mem, log.buf, end_lsn - start_lsn,
                                  start_lsn, &log.m_scanned_lsn);
    start_lsn = end_lsn;
  }
  return DB_SUCCESS;
}
```

要点：

- **起点向下对齐到 block 边界**：`checkpoint_lsn` 可能在块中间，先 `align_down` 到 512 边界再读
- **`parse_start_lsn = 0` 是"未找到解析起点"的哨兵**：源码注释明说 8.0 起 checkpoint_lsn 可能指向某条 record 的中间，必须先找到"第一个起始 lsn ≥ checkpoint_lsn 的 record group"才能开始解析
- **`max_mem` 由 buffer pool 规模推导**：`UNIV_PAGE_SIZE × (总页数 − 预留空闲帧)`。这是 hash 的内存上限——预留 `recv_n_pool_free_frames` 是为了保证应用阶段还有帧可用于读页
- **`end_lsn == start_lsn` 的处理**：刚写完一个 redo 文件、下一个文件尚未创建时发生，直接 break 而不是报错

### 扫描主循环：recv_scan_log_recs

这是恢复的**核心状态机**。每个 block 依次过三道"abrupt end"判据，然后确立解析起点、喂进解析缓冲：

```cpp
  do {
    Log_data_block_header block_header;
    log_data_block_header_deserialize(log_block, block_header);   // ① 反序列化 12B 块头

    const uint32_t expected_hdr_no =
        log_block_convert_lsn_to_hdr_no(scanned_lsn);             // ② 由 lsn 推算期望块号

    if (block_header.m_hdr_no != expected_hdr_no) {               // ③ 判据一：块号不符
      /* Garbage or an incompletely written log block. ... We simply
      treat this as an abrupt end of the redo log. */
      finished = true;
      break;
    }

    if (!log_block_checksum_is_ok(log_block)) {                   // ④ 判据二：crc32 失败
      /* ... treat this as an abrupt end of the redo log. */
      finished = true;
      break;
    }

    const auto data_len = block_header.m_data_len;                // ⑤ 本块有效数据字节数

    if (scanned_lsn + data_len > recv_sys->scanned_lsn &&         // ⑥ 判据三：epoch_no 非法
        recv_sys->scanned_epoch_no > 0 &&                         //    仅当已有基准时才比
        !log_block_epoch_no_is_valid(block_header.m_epoch_no,
                                     recv_sys->scanned_epoch_no)) {
      /* Garbage from a log buffer flush which was made
      before the most recent database recovery */                 //    上一轮恢复遗留的旧块
      finished = true;
      break;
    }
```

**三道停扫判据**（这是 8.0 与 5.7 最大的差别）：

1. **`hdr_no` 不匹配**：块头里的块号（`hdr_no` 由 lsn 换算）与按 `scanned_lsn` 推算的期望值不符 → 垃圾或写了一半的块
2. **checksum 失败**：同上。注释明确写了原因——"killing the server while it was writing this log block"
3. **`epoch_no` 非法**：这一条是 8.0 独有。无锁并发写让 log buffer 里可能有空洞，也可能残留**上一次恢复之前**刷下去的旧块；`epoch_no` 由 lsn 高位推算，能识别"这个块属于上一轮"，从而避免把陈旧数据当新数据解析

> ★ 5.7 时代"回放到 `data_len < 512` 的块即最后一块"的判据在 8.0 **已不成立**：`data_len < 512` 只说明"这个块没写满"，但并发写下一个块可能已被写了。8.0 只认上面三条。

接着是**解析起点的确立**——这是整个恢复最容易看漏的一步：

```cpp
    if (!recv_sys->parse_start_lsn &&                          // ① 起点尚未确立（0 = 哨兵）
        block_header.m_first_rec_group > 0) {                  // ② 且本块内有新 mtr 起始
      recv_sys->parse_start_lsn =                              // ③ 起点 = 块 lsn + 组内偏移
          scanned_lsn + block_header.m_first_rec_group;

      if (recv_sys->parse_start_lsn < recv_sys->checkpoint_lsn) {   // ④ 回退到 checkpoint 之前了
        /* We start to parse log records even before checkpoint_lsn, from
        the beginning of the log block which contains the checkpoint_lsn.
        ... checkpoint_lsn could potentially point to the middle of some
        log record. We need to find the first group of log records that
        starts at or after checkpoint_lsn. ...
        However, we don't want to report missing tablespaces for space_id
        in log records before checkpoint_lsn. Hence we need ... a counter
        of bytes to ignore. */
        recv_sys->bytes_to_ignore_before_checkpoint =
            recv_sys->checkpoint_lsn - recv_sys->parse_start_lsn;
        ...
      }

      recv_sys->scanned_lsn = recv_sys->parse_start_lsn;
      recv_sys->recovered_lsn = recv_sys->parse_start_lsn;
      recv_track_changes_of_recovered_lsn();
    }
```

**为什么必须回退**：`checkpoint_lsn` 可能落在某条 record 的中间，直接从这个 lsn 开始解析会把半条 record 当完整 record。正确做法是回到**该块内第一个 record group 的起点**（`first_rec_group` 给出的偏移）。因为 `first_rec_group` 只在"块内有新 mtr 起始"时非零，所以要一路扫到第一个非零的块。

**`bytes_to_ignore_before_checkpoint` 的作用**：回退后 `parse_start_lsn < checkpoint_lsn`，中间那截 record 属于 checkpoint 之前的修改（已落盘）。解析它们只为"找到 group 边界"，**不能真的应用**（否则会重放已落盘的修改，且可能因表空间已不存在而误报）。这个字段就是"要忽略的字节数"，应用阶段用 `recv_update_bytes_to_ignore_before_checkpoint(len)` 逐条递减。

然后是**喂进解析缓冲 + 内存压力处理**：

```cpp
      if (recv_sys->len + 4 * OS_FILE_LOG_BLOCK_SIZE >= recv_sys->buf_len) {  // ① 缓冲快满
        if (!recv_sys_resize_buf()) {                          // ② 先尝试扩容（2MB 起）
          recv_sys->found_corrupt_log = true;                  // ③ 扩不动 → 标记损坏
          if (srv_force_recovery == 0) {                       // ④ 只有强制恢复档位才放行
            ib::error(ER_IB_MSG_724);
            return true;
          }
        }
      }

      if (!recv_sys->found_corrupt_log) {
        more_data = recv_sys_add_to_parsing_buf(log_block, scanned_lsn);  // ⑤ 数据区拼进解析缓冲
      }                                                        //    （跳过 12B 头、截断 4B 尾）

      recv_sys->scanned_lsn = scanned_lsn;                     // ⑥ 推进"已扫描"水位
      recv_sys->scanned_epoch_no = block_header.m_epoch_no;    // ⑦ 更新 epoch 校验基准
```

块数据区被**连续拼进 `recv_sys->buf`**（`recv_sys_add_to_parsing_buf` 里跳过 12B 头、截断 4B 尾做 memcpy），解析器看到的是连续 record 流，块边界完全透明。缓冲快满时先尝试扩容（2MB 起，可增长），扩不动则置 `found_corrupt_log`——此时**只有设了 `innodb_force_recovery` 才继续**。

循环末尾是**扫描与应用的耦合点**：

```cpp
    if (data_len < OS_FILE_LOG_BLOCK_SIZE) {
      finished = true;                          // ① 块没写满 → 已读入的这一段到此为止
      break;
    } else {
      log_block += OS_FILE_LOG_BLOCK_SIZE;      // ② 推进到下一块
    }
  } while (log_block < buf + len);              // ③ 只在本次读入的 64KB 内循环

  if (more_data && !recv_sys->found_corrupt_log) {
    recv_parse_log_recs();                      // ④ 解析缓冲里新追加的数据

    if (recv_heap_used() > max_memory) {        // ⑤ hash 内存超阈值
      recv_apply_hashed_log_recs(log, false);   //    → 中途应用腾空间（allow_ibuf=false）
    }

    if (recv_sys->recovered_offset > recv_sys->buf_len / 4) {   // ⑥ 已消费超过 1/4
      recv_reset_buffer();                      //    → 左移腾出尾部空间
    }
  }
```

- `data_len < 512` 在这里是**"这批日志读完了"**的判据（在读进来的 64KB 范围内），与前面说的"不能作为 8.0 的停扫判据"并不矛盾：它只在已读入的段内生效，真正的日志尾由三道 abrupt end 判据确定
- `recv_heap_used() > max_memory` → 中途 apply，`allow_ibuf = false`（见下）
- `recovered_offset > buf_len / 4` → 把已解析部分左移，避免缓冲被已消费数据占满

### 解析：mtr 分组与两遍扫描

`recv_parse_log_recs` 循环调用 `recv_single_rec`（单记录 mtr）或 `recv_multi_rec`（多记录 mtr）。核心在 `recv_multi_rec` 的**两遍扫描**：

**第一遍：只确认"整组是否齐全"，顺手缓存解析结果**

```cpp
static bool recv_multi_rec(byte *ptr, byte *end_ptr) {
  /* Check that all the records associated with the single mtr
  are included within the buffer */
  ulint n_recs = 0;
  ulint total_len = 0;

  for (;;) {
    mlog_id_t type = MLOG_BIGGEST_TYPE;
    byte *body;
    page_no_t page_no = 0;
    space_id_t space_id = 0;

    ulint len =
        recv_parse_log_rec(&type, ptr, end_ptr, &space_id, &page_no, &body);

    if (recv_sys->found_corrupt_log) { ... return true; }        // ② 已标记损坏 → 放弃
    else if (len == 0) { return true; }              // ③ 数据不够解析出下一条 → 等下一块
    else if ((*ptr & MLOG_SINGLE_REC_FLAG)) {
      recv_sys->found_corrupt_log = true;            // ④ 多记录组里混进单记录标记 → 结构错乱
      return true;
    }
    else if (recv_sys->found_corrupt_fs) { return true; }        // ⑤ 与文件系统不一致

    recv_sys->save_rec(n_recs, space_id, page_no, type, body, len);  // ⑥ ★ 缓存解析结果
    total_len += len;                                //    第二遍靠它免解析
    ++n_recs;
    ptr += len;                                      // ⑦ 指针前移，继续找结尾标记

    if (type == MLOG_MULTI_REC_END) { break; }       // ⑧ 见到结尾 → 整组齐全
  }

  lsn_t new_recovered_lsn =
      recv_calc_lsn_on_data_add(recv_sys->recovered_lsn, total_len);  // ⑨ 组结束后的 lsn

  if (new_recovered_lsn > recv_sys->scanned_lsn) {   // ⑩ 组正好填满一个块
    /* The log record filled a log block, and we require
    that also the next log block should have been scanned in */
    return true;                                     //    → 必须等下一块也扫进来才确认完整
  }
```

- `len == 0` = 缓冲里的数据不够解析出下一条 → 返回，等下一块追加后再试（跨块 record 的自然处理）
- 见到 `MLOG_SINGLE_REC_FLAG` 说明结构错乱 → 判定损坏
- **关键**：`new_recovered_lsn > scanned_lsn` 时返回 —— 这条 mtr 正好填满一个块，必须等**下一个块也扫进来**才能确认完整。这正是 8.0 "不能凭 data_len < 512 判断结尾"的配套措施
- `save_rec(n_recs, ...)` 把解析结果存进 `saved_recs`，供第二遍直接取用

**第二遍：真正加入 hash**

```cpp
  /* Add all the records to the hash table */
  ptr = recv_sys->buf + recv_sys->recovered_offset;   // ① 回到本组起始位置

  for (ulint i = 0; i < n_recs; i++) {
    lsn_t old_lsn = recv_sys->recovered_lsn;         // ② 记下本条起始 lsn
    ...
    /* Avoid parsing if we have the record saved already. */
    if (!recv_sys->get_saved_rec(i, space_id, page_no, type, body, len)) {
      len = recv_parse_log_rec(&type, ptr, end_ptr, &space_id, &page_no, &body);
    }                                                // ③ ★ 缓存命中就免解析（1.8x 提速来源）
    ...
    recv_sys->recovered_offset += len;               // ④ 消费偏移前移
    recv_sys->recovered_lsn = recv_calc_lsn_on_data_add(old_lsn, len);  // ⑤ 解析水位前移

    const bool apply = !recv_update_bytes_to_ignore_before_checkpoint(len);  // ⑥ checkpoint 前的字节不计

    switch (type) {
      case MLOG_MULTI_REC_END:                       // ⑦ 结尾标记：更新块边界记录后返回
        recv_track_changes_of_recovered_lsn();
        return false;

      case MLOG_FILE_DELETE:
      case MLOG_FILE_CREATE:
      case MLOG_FILE_RENAME:
      case MLOG_FILE_EXTEND:
      case MLOG_TABLE_DYNAMIC_META:
        /* These were already handled by recv_parse_or_apply_log_rec_body(). */
        break;                                       // ⑧ 文件级/元数据：解析阶段已执行，不入 hash

      default:
        if (!apply) { break; }                       // ⑨ 属于 checkpoint 之前的字节 → 跳过

        if (recv_recovery_on                         // ⑩ 只在恢复开启时入 hash
            && (space_id == TRX_SYS_SPACE ||         //    系统表空间总是有效
                fil_tablespace_lookup_for_recovery(space_id))) {  // 其他需表空间仍存在
          recv_add_to_hash_table(type, space_id, page_no, body, ptr + len,
                                 old_lsn, new_recovered_lsn);
        }
    }
    ptr += len;
  }
  return false;                                      // ⑪ false = 这组已完整处理
```

四个要点：

1. **`get_saved_rec` 命中就不重新解析**——这就是 1.8 倍提速的来源；超过 `MAX_SAVED_MLOG_RECS`(8K) 则静默退回重新解析
2. **`apply = !recv_update_bytes_to_ignore_before_checkpoint(len)`**：checkpoint 之前那截字节被逐条扣减，扣完之前 `apply = false`，记录被解析但不入 hash
3. **`MLOG_FILE_*` 与 `MLOG_TABLE_DYNAMIC_META` 不入 hash**：它们在解析阶段（`recv_parse_or_apply_log_rec_body`）就已执行完毕，这里跳过
4. **表空间存在性过滤**：`space_id == TRX_SYS_SPACE || fil_tablespace_lookup_for_recovery(space_id)` —— 表空间已不存在的 redo 直接丢弃（对应 `recv_sys->missing_ids`）

### recv_sys_t：恢复上下文的完整结构

恢复全程的**唯一全局状态**是单例 `recv_sys`。它既是"扫描器状态机"，又是"待应用 redo 的容器"。

**嵌套类型（8.0 相对 5.7 的最大变化）**：

```cpp
struct Space {                        // 每个表空间独立一套
  mem_heap_t *m_heap;                 // 该表空间的 redo 记录体存于此 heap
  Pages m_pages;                      // page_no → recv_addr_t*（该页待应用的 redo 链）
};
using Pages  = std::unordered_map<page_no_t, recv_addr_t *>;
using Spaces = std::unordered_map<space_id_t, Space>;
```

> ⚠️ **纠正**：8.0 **没有** `recv_sys->addr_hash`——那是 5.7 的单级全局页 hash。8.0 改成两级 `spaces`：先按 `space_id` 分桶，每桶自带 heap 与 `page_no → recv_addr_t*` 映射。收益是**按表空间粒度整体释放**：DROP/删表空间的 redo 直接丢整桶，不必逐页清理。

**成员表**：

| 成员 | 类型 | 作用 |
|---|---|---|
| `mutex` | ib_mutex_t | 保护 `apply_log_recs` / `n_addrs` / 每个 `recv_addr` 的 state |
| `writer_mutex` | ib_mutex_t | 协调 `recv_writer_thread` 与恢复线程 |
| `flush_start` / `flush_end` | os_event_t | 唤醒 page cleaner 替恢复腾页 / 等它做完 |
| `flush_type` | buf_flush_t | `BUF_FLUSH_LRU`（腾 free block）/ `BUF_FLUSH_LIST`（全刷） |
| `apply_log_recs` | bool | 是否允许把 redo 应用到页（io-handler 据此决定是否 apply） |
| `apply_batch_on` | bool | 是否正在跑一批 apply |
| `buf` | byte* | **解析缓冲**——块数据区连续拼入，块头/块尾已剥离 |
| `buf_len` | size_t | 缓冲大小，初始 `RECV_PARSING_BUF_SIZE`（2MB），可扩容 |
| `len` | ulint | buf 中有效字节数 |
| `parse_start_lsn` | lsn_t | 能开始解析并入 hash 的 lsn；**0 = 尚未找到合适起点** |
| `checkpoint_lsn` | lsn_t | 本次恢复使用的 checkpoint lsn（从文件读） |
| `bytes_to_ignore_before_checkpoint` | ulint | 到 checkpoint_lsn 前要忽略的数据字节数 |
| `scanned_lsn` | lsn_t | 日志**已扫描**（读入 buf）到的 lsn |
| `scanned_epoch_no` | uint32_t | 已扫描到的 epoch_no（`epoch_no` 合法性校验的基准） |
| `recovered_offset` | ulint | buf 中未解析记录的起始偏移（消费后左移腾空间） |
| `recovered_lsn` | lsn_t | 已**解析**到的 lsn |
| `previous_recovered_lsn` | lsn_t | 上一值；用于检测 `recovered_lsn` 跨块 |
| `last_block_first_mtr_boundary` | lsn_t | `recovered_lsn` 所属块的 `first_rec_group` 应有值（0 = 未知） |
| `found_corrupt_log` | bool | 发现坏块/坏记录，或解析缓冲溢出 |
| `found_corrupt_fs` | bool | 扫描/应用时发现与文件系统不一致 |
| `is_cloned_db` / `is_meb_db` | bool | 识别出是 clone / MEB 的数据目录 |
| `dblwr_state` | bool | MEB 恢复前的 doublewrite 状态（MEB 期间禁用 dblwr，结束后还原） |
| `spaces` | Spaces* | **页 hash 表**：space_id → page_no 两级聚合待应用 redo |
| `n_addrs` | ulint | hash 表中未处理的地址数（受 mutex 保护；满则中途 apply 腾空间） |
| `dblwr` | dblwr::recv::DBLWR* | 恢复期专用的 doublewrite 页，恢复完成后销毁 |
| `metadata_recover` | MetadataRecover* | 扫描期收集并合并各表持久动态元数据 |
| `keys` | Encryption_Keys* | 每个表空间的加密 key（space_id / lsn / key / iv） |
| `missing_ids` | Missing_Ids | redo 应用期间被忽略的表空间（文件已不在） |
| `deleted` | Missing_Ids | 被显式删除的表空间 |
| `saved_recs` | Mlog_records | 两遍扫描优化：缓存已解析记录，避免二次解析 |

**相关常量**：

| 常量 | 值 | 含义 |
|---|---|---|
| `RECV_PARSING_BUF_SIZE` | 2MB | 解析缓冲初始大小（须能容纳多次 `RECV_SCAN_SIZE`） |
| `RECV_SCAN_SIZE` | `4 * UNIV_PAGE_SIZE`（64KB） | 前滚扫描的块读大小 |
| `MAX_SAVED_MLOG_RECS` | `8 * 1024` | `saved_recs` 上限；源码注释称 1G redo 的恢复扫描**提速约 1.8 倍** |

### 应用：hash 聚合与按页重放

`recv_apply_hashed_log_recs` 遍历两级 hash，对每页调 `recv_apply_log_rec`。先看单页的调度（`recv_apply_log_rec` 逐行剖析）：

```cpp
static void recv_apply_log_rec(recv_addr_t *recv_addr) {
  if (recv_addr->state == RECV_DISCARDED) {          // ① 整桶被丢弃（表空间已 DROP）
    ut_a(recv_sys->n_addrs > 0);
    --recv_sys->n_addrs;
    return;
  }

  bool found;
  const page_id_t page_id(recv_addr->space, recv_addr->page_no);

  const page_size_t page_size =
      fil_space_get_page_size(recv_addr->space, &found);   // ② 表空间是否还在

  if (!found || recv_sys->missing_ids.find(recv_addr->space) !=
                    recv_sys->missing_ids.end()) {
    /* Tablespace was discarded or dropped after changes were
    made to it. ... We can't apply this redo log out of order. */
    recv_addr->state = RECV_PROCESSED;               // ③ 放弃应用，但计入"已处理"
    --recv_sys->n_addrs;

    if (recv_sys->deleted.find(recv_addr->space) == recv_sys->deleted.end()) {
      recv_sys->missing_ids.insert(recv_addr->space);  // ④ 非显式删除 → 记入 missing_ids
    }

  } else if (recv_addr->state == RECV_NOT_PROCESSED) {
    mutex_exit(&recv_sys->mutex);                    // ⑤ ★ 读页是 IO，必须先放开全局锁

    if (buf_page_peek(page_id)) {                    // ⑥ 页已在 buffer pool
      mtr_start(&mtr);
      block = buf_page_get(page_id, page_size, RW_X_LATCH, UT_LOCATION_HERE, &mtr);  // ⑦ X latch
      buf_block_dbg_add_level(block, SYNC_NO_ORDER_CHECK);
      recv_recover_page(false, block);               // ⑧ 应用该页的 redo 链
      mtr_commit(&mtr);
    } else {
      recv_read_in_area(page_id);                    // ⑨ ★ 不在内存 → 预读整个 area
    }

    mutex_enter(&recv_sys->mutex);                   // ⑩ 重新拿锁
  }
}
```

三个要点：

- **④ `missing_ids` vs `deleted` 的区分**：`deleted` 是显式删除的表空间（安全忽略）；其余"找不到"的情况要记进 `missing_ids`，后续该表空间的 redo 一律跳过——并防止"乱序应用"（注释：can't apply this redo log out of order）
- **⑤⑩ 锁的收放**：读页涉及 IO，必须先 `mutex_exit` 再 `mutex_enter`，否则全局锁会被 IO 长时间占住
- **⑨ `recv_read_in_area` 是恢复期的 IO 优化**：不只读目标页，而是读**一整个 area**（相邻 64 页量级）。因为 hash 里待恢复的页往往成片相邻，这把随机读变成顺序读，是恢复速度的关键之一

再看批量应用的主函数 `recv_apply_hashed_log_recs`：

```cpp
dberr_t recv_apply_hashed_log_recs(log_t &log, bool allow_ibuf) {
  for (;;) {
    mutex_enter(&recv_sys->mutex);
    if (!recv_sys->apply_batch_on) { break; }
    mutex_exit(&recv_sys->mutex);
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
  }

  if (!allow_ibuf) { recv_no_ibuf_operations = true; }

  recv_sys->apply_log_recs = true;
  recv_sys->apply_batch_on = true;

  auto batch_size = recv_sys->n_addrs;
  ...
  for (const auto &space : *recv_sys->spaces) {
    bool dropped;
    if (space.first == TRX_SYS_SPACE) {
      dropped = false;
    } else {
      dberr_t err = fil_tablespace_open_for_recovery(space.first);
      if (err == DB_SUCCESS) { dropped = false; }
      else if (err == DB_CORRUPTION) {
        /* Page couldn't be recovered from doublewrite, we cannot proceed
        with recovery. Skip applying redos and abort the startup. */
        return err;
      } else {
        /* Tablespace was dropped. ... */
        dropped = true;
      }
    }

    for (auto pages : space.second.m_pages) {
      if (dropped) { pages.second->state = RECV_DISCARDED; }
      recv_apply_log_rec(pages.second);
      ...
    }
  }

  while (recv_sys->n_addrs != 0) {      // 等所有页处理完
    mutex_exit(&recv_sys->mutex);
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    mutex_enter(&recv_sys->mutex);
  }

  if (!allow_ibuf) {
    /* Flush all the file pages to disk and invalidate them in the buffer pool */
    mutex_enter(&recv_sys->writer_mutex);
    buf_flush_wait_LRU_batch_end();
    os_event_reset(recv_sys->flush_end);
    recv_sys->flush_type = BUF_FLUSH_LIST;
    os_event_set(recv_sys->flush_start);
    os_event_wait(recv_sys->flush_end);
    buf_pool_invalidate();
    mutex_exit(&recv_sys->writer_mutex);
    recv_no_ibuf_operations = false;
  }

  recv_sys->apply_log_recs = false;
  recv_sys->apply_batch_on = false;
  recv_sys_empty_hash();
  return DB_SUCCESS;
}
```

逐段：

- **开头等 `apply_batch_on` 落**：同一时刻只允许一批 apply，避免并发重入
- **`allow_ibuf = false` 的含义**：中途 apply（腾内存）时传 false → 置 `recv_no_ibuf_operations`。因为此时页可能还没恢复到最新状态，change buffer 合并会写坏页
- **按表空间打开/丢弃**：`fil_tablespace_open_for_recovery` 返回 `DB_CORRUPTION` 表示页连 doublewrite 都救不回来 → **直接放弃启动**；返回其他错误表示表空间已被 DROP → 整桶标 `RECV_DISCARDED` 跳过
- **进度打印**：每 10% 或每个 `PRINT_INTERVAL` 打一条 `ib::info`，这是恢复期间错误日志里那些百分比的来源
- **`!allow_ibuf` 收尾要 `buf_pool_invalidate()`**：中途 apply 时 page cleaner 可能已把"未完全恢复"的页写出去，因此最后必须**全刷 + 清空 buffer pool**，确保所有页下次从磁盘重新读入完整版本

**恢复期的辅助线程 `recv_writer_thread`** 负责腾页：每 100ms 通过 `flush_start` / `flush_end` 事件驱动 page cleaner 从 **LRU 尾部**刷页（`BUF_FLUSH_LRU`）保证有空闲帧；恢复结束阶段改用 `BUF_FLUSH_LIST` 全刷。它保证恢复不会因 buffer pool 无空闲帧而卡死。

### 单页重放：recv_recover_page_func 的幂等

单页应用是幂等性的落点：

```cpp
  mtr_start(&mtr);
  mtr_set_log_mode(&mtr, MTR_LOG_NONE);     // ① ★ 重放不再产生 redo（redo 是输入不是输出）
  ...
  /* Read the newest modification lsn from the page */
  lsn_t page_lsn = mach_read_from_8(page + FIL_PAGE_LSN);   // ② 页头记录的"已落盘版本"LSN

  /* It may be that the page has been modified in the buffer
  pool: read the newest modification LSN there */
  lsn_t page_newest_lsn = buf_page_get_newest_modification(&block->page);  // ③ 内存中的更新版本
  if (page_newest_lsn) { page_lsn = page_newest_lsn; }      // ④ 取两者较新者作为比较基准

  for (auto recv : recv_addr->rec_list) {                   // ⑤ 按 lsn 顺序遍历该页的 redo 链
    ...
    if (recv->type == MLOG_INIT_FILE_PAGE) {                // ⑥ 特例：页被重新初始化
      page_lsn = page_newest_lsn;
      memset(FIL_PAGE_LSN + page, 0, 8);      //    页上旧 LSN 已无意义 → 清零重来
      ...
    }

    if (recv->start_lsn >= page_lsn && undo::is_active(recv_addr->space)) {  // ⑦ ★ 幂等判据
      if (!modification_to_page) {
        modification_to_page = true;
        start_lsn = recv->start_lsn;          //    记录本页首个真正应用的 lsn
      }
      recv_parse_or_apply_log_rec_body(recv->type, buf, buf_end,
                                       recv_addr->space, recv_addr->page_no,
                                       block, &mtr, ULINT_UNDEFINED, LSN_MAX);  // ⑧ 真正改页
      end_lsn = recv->start_lsn + recv->len;
      mach_write_to_8(FIL_PAGE_LSN + page, end_lsn);                    // ⑨ 回写页头 LSN
      mach_write_to_8(UNIV_PAGE_SIZE - FIL_PAGE_END_LSN_OLD_CHKSUM + page, end_lsn);  // ⑩ 页尾 LSN
      ...
    }
  }
```

四个关键点：

1. **`mtr_set_log_mode(&mtr, MTR_LOG_NONE)`**：重放过程中对页的修改**不再产生 redo**——redo 是恢复的输入，不是输出。这是恢复能收敛的前提
2. **双重 LSN 来源**：页头的 `FIL_PAGE_LSN` 只反映"已刷盘的版本"；若该页在 buffer pool 里已被修改（本次恢复中由更早的 redo 改过），要用 `buf_page_get_newest_modification` 的值。取两者中**较新**的作为比较基准
3. **`recv->start_lsn >= page_lsn` 是幂等判据**：严格小于才跳过。注意 `undo::is_active(recv_addr->space)` 这一条件——undo 表空间的重放另有约束
4. **`MLOG_INIT_FILE_PAGE` 特例**：这条 redo 表示"页被重新初始化"，此时页上旧的 LSN 已无意义，先清零再重放

应用完后 `recv_max_page_lsn` 记录见到的最大页 LSN，`recv_recovery_from_checkpoint_start` 收尾时与 `m_scanned_lsn` 对账——若扫描到的 LSN 还小于已见过的页 LSN，说明**扫描漏了数据**，报错。

### redo 与 undo 的协同

为什么必须先 redo 后 undo？三层递进：

**1. undo 页本身受 redo 保护**：undo 就是 buffer pool 里的普通页，写 undo 记录同样产生 redo（`MLOG_UNDO_INSERT`、`MLOG_UNDO_INIT`、`MLOG_UNDO_HDR_REUSE`、`MLOG_UNDO_HDR_CREATE`）。所以"redo 保住 undo"是两阶段能成立的前提——崩溃后 undo 段先被前滚重建，才有东西可回滚。

**2. redo 里没有事务边界**：redo 只有一条条页级/记录级修改，`mlog_id_t` 76 种类型里**没有任何 begin/commit 类型**。事务的提交状态完全靠 **undo 段头的状态位**判定：

| 状态 | 值 | 含义 |
|---|---|---|
| `TRX_UNDO_ACTIVE` | 1 | 活跃事务 |
| `TRX_UNDO_CACHED` | 2 | 缓存待复用 |
| `TRX_UNDO_TO_FREE` | 3 | insert undo 段可释放 |
| `TRX_UNDO_PREPARED_80028` | 5 | 8.0.29 之前版本的 prepared |
| `TRX_UNDO_PREPARED` | 6 | prepared（8.0.29+） |
| `TRX_UNDO_PREPARED_IN_TC` | 7 | 已被事务协调器处理的 prepared |

**3. 因此顺序不可交换**：必须先靠 redo 把 undo 页恢复到崩溃瞬间，**才读得出谁没提交**，然后才轮到 `trx_rollback_or_clean_recovered` 回滚。这就是 ARIES "Redo before Undo" 在 MySQL 的兑现。

**4. 2PC 下由 binlog 裁决**：处于 `TRX_UNDO_PREPARED` 的事务，恢复后提交还是回滚取决于 binlog 中有无对应 XID——见 [`trx.md`](trx.md) 与 [`../server/replication/binlog.md`](../server/replication/binlog.md)。

**5. 收尾**：回滚/提交后 undo 由 purge 异步清理，见 [`undo_log.md`](undo_log.md)。

> redo 侧的同一主题（含 undo 类 redo 的类型编号）另见 [`redo_log.md`](redo_log.md)「redo 与 undo 的协同」节。

### 文件级 redo 与 DDL 在恢复期的处理

`MLOG_FILE_*` 系列（文件的创建/删除/重命名/扩展）**不进 hash**——它们不是页修改，无法按页聚合。因此它们在**解析阶段就被执行**：`recv_parse_or_apply_log_rec_body` 在解析 record body 时直接作用于文件系统，所以 `recv_multi_rec` 第二遍的 switch 里这些类型是 `break`（已在解析时处理过）。

同理还有 `MLOG_TABLE_DYNAMIC_META`——它更新的是表的动态元数据，在解析阶段就并入 `recv_sys->metadata_recover`，最终由 `recv_recovery_from_checkpoint_finish` 回传给 DD（`MetadataRecover*` 返回值）。这也是恢复收尾前**禁止 checkpoint** 的原因：

```cpp
  /* Disallow checkpoints until recovery is finished, and changes gathered
  in recv_sys->metadata_recover (dict_metadata) are transferred to
  dict_table_t objects (happens in srv0start.cc). */
```

顺序上的天然安全性：扫描期只处理文件操作，扫完才统一应用页级 redo —— 避免了"先删了文件、后面又要读页"。

`recv_sys->deleted` 记下被显式删除的表空间，`missing_ids` 记下 redo 提到但文件已不存在的表空间；应用时按这两张表决定跳过哪些页。DDL 侧细节见 [`ddl.md`](ddl.md)。

### clone / MEB 数据目录的恢复分支

- **`is_cloned_db`**：识别出是 clone 过来的数据目录。这类目录的 redo 与页可能来自不同时点，需 clone 插件配合走专门流程
- **`is_meb_db`**：识别出是 MySQL Enterprise Backup 的备份目录。MEB 恢复期间**临时禁用 doublewrite**（`dblwr_state` 保存原状态），因为 MEB 自己做页完整性保证；恢复结束后还原

`recv_scan_log_recs` 与 `recv_apply_hashed_log_recs` 都有 `UNIV_HOTBACKUP` 分支（MEB 编译时走另一套逻辑，如 `meb_scan_log_recs`），社区版编译走 `#ifndef UNIV_HOTBACKUP` 分支。

**上下游闭环**：上游 = 崩溃后重启，`srv_start` 调 `recv_recovery_from_checkpoint_start`；下游 = 前滚后脏页刷盘、引擎进入运行态，随后 `trx_rollback_or_clean_recovered` 回滚未提交事务，purge 线程接管 undo 清理。redo 的产生与落盘见 [`redo_log.md`](redo_log.md)，undo 的管理见 [`undo_log.md`](undo_log.md)。

---

## 相关的系统变量/状态变量

### 系统变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_force_recovery` | 0 | Global（只读） | 强制恢复档位 0–6，**非 0 时 InnoDB 以只读方式启动**，且不可在线调回 0 |

七档语义（常量定义在 `srv0srv.h` 的匿名 enum，注释明写"bigger number includes all precautions of lower numbers"）：

| 档位 | 常量 | 放弃的恢复工作 |
|---|---|---|
| 1 | `SRV_FORCE_IGNORE_CORRUPT` | 容忍损坏页，继续跑 |
| 2 | `SRV_FORCE_NO_BACKGROUND` | 不起主线程（purge 崩溃时用它） |
| 3 | `SRV_FORCE_NO_TRX_UNDO` | 恢复后**不回滚**事务 |
| 4 | `SRV_FORCE_NO_IBUF_MERGE` | 不做 change buffer 合并 |
| 5 | `SRV_FORCE_NO_UNDO_LOG_SCAN` | 启动时不看 undo 日志——**未提交事务也被当作已提交** |
| 6 | `SRV_FORCE_NO_LOG_REDO` | **不做 redo 前滚**（`recv_recovery_from_checkpoint_start` 开头直接返回） |

> 这张表就是"恢复各阶段相互独立"的直接证据：从 1 到 6 依次放弃页校验、后台线程、undo 回滚、ibuf 合并、undo 扫描、redo 前滚。

---

## Misc

### 恢复进度的四把尺子（易混淆）

```
checkpoint_lsn  ──►  scanned_lsn  ──►  recovered_lsn  ──►  应用完成
   恢复起点           已读入 buf        已解析入 hash
```

外加 `parse_start_lsn`（真正能开始解析的起点，由 `first_rec_group` 修正 checkpoint_lsn 得到）。四者**不可互相替代**：`scanned_lsn` 领先 `recovered_lsn`（已读入未解析完），`recovered_lsn` 领先应用进度（已解析未应用）。

### 易混淆的命名：`recv_no_ibuf_operations`

名字暗示"不允许 change buffer 操作"，实际语义更宽——源码注释自称"the variable name is misleading"：它真正表示**恢复仍在运行、尚不能对日志文件做操作**。置位时机是 hash 过满被迫中途 apply，此时页可能处于未完全恢复状态，故连 ibuf 合并都需禁止。

### 配套的全局标志

| 标志 | 含义 |
|---|---|
| `recv_recovery_on` | 正在应用 redo；**后台回滚线程运行时为 false**（前滚结束、回滚阶段不置位） |
| `recv_needed_recovery` | `recv_init_crash_recovery` 已调用，本次启动确实走了恢复 |
| `recv_lsn_checks_on` | 开启"页 LSN 是否在未来"的检查，由 `recv_recovery_from_checkpoint_start` 置位 |
| `recv_is_making_a_backup` / `recv_is_from_backup` | MEB 备份/恢复标志 |

### 恢复期的两个"看起来像 bug"的正常现象

- **正常重启也走恢复代码**：`srv_start` 总会调 `recv_recovery_from_checkpoint_start`，只是 `checkpoint_lsn == flush_lsn` 时无需前滚
- **恢复日志里的百分比**：来自 `recv_apply_hashed_log_recs` 的每 10% 进度打印（`batch_size = n_addrs`，即待恢复页数），不是 redo 字节数百分比

---

## 参考

**论文**

- Mohan, Haderle, Lindsay, Pirahesh, Schwarz. *ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging*. ACM TODS 17(1), 1992.
- Gray & Reuter. *Transaction Processing: Concepts and Techniques*. Morgan Kaufmann, 1993.（WAL 与两阶段恢复的经典表述）

**官方文档**

- *MySQL 8.0 Reference Manual → InnoDB Recovery*
- *MySQL 8.0 Reference Manual → Forcing InnoDB Recovery*（`innodb_force_recovery` 各档位语义）

**内核月报 / 技术文章**

- 内核月报《MySQL 崩溃恢复》(2015-06)
- 《MySQL 的恢复》(code0xff.org, 2022-12，含 undo log 所有类型)
- 《MySQL redo log 恢复原理》(StoneDB 技术分享会 #5, 2023-08)

**相关文档**

- redo 的产生、格式、LSN 与 checkpoint（上游）见 [`redo_log.md`](redo_log.md)
- undo 的管理与 purge（下游）见 [`undo_log.md`](undo_log.md)
- prepared 事务的提交/回滚裁决见 [`trx.md`](trx.md)，binlog 侧见 [`../server/replication/binlog.md`](../server/replication/binlog.md)
- 脏页刷盘、doublewrite（半页写防护）见 [`buffer_pool.md`](buffer_pool.md)
- DDL 的文件级操作见 [`ddl.md`](ddl.md)

> ⚠️ 月报/博客的函数名与行号多基于 5.7 或 8.0 早期版本，与 8.0.39 对不上——只取问题视角，机制一律以本篇源码为准。
