# 索引的物理存储与持久化：根页、段区、redo 与恢复

> 基于 MySQL 8.0.39 源码。本篇讲 **"一个索引占用哪些物理资源、怎么落盘、怎么在崩溃后恢复"**——根页的恒定、两段式分配、索引页布局、空间回收、索引操作的 redo 记录体系、bulk load 的 redo 豁免与 DDL log 兜底。
>
> **边界**：B 树**页内操作**（搜索/分裂/合并/更新）见 [`btr.md`](btr.md)；**页的通用物理结构**（FIL header / 目录槽 / infimum-supremum）见 [`../innodb/physical/page_structure.md`](../innodb/physical/page_structure.md)；**表空间/段/区管理**的通用机制见 [`../innodb/physical/tablespace.md`](../innodb/physical/tablespace.md)；**redo 的整体体系**（LSN/mtr/落盘）见 [`../innodb/redo_log.md`](../innodb/redo_log.md)；**恢复的整体流程**见 [`../innodb/recovery.md`](../innodb/recovery.md)——本篇只取"索引"这一视角，物理结构的篇内布局可互补但以页结构篇为准。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [根页：恒定的物理锚点](#根页恒定的物理锚点)
    - [段与区：每个索引两个段](#段与区每个索引两个段)
    - [索引页布局：PAGE_HEADER 逐字段](#索引页布局page_header-逐字段)
    - [树高度：现算不缓存](#树高度现算不缓存)
  - 持久化：索引的 redo 体系
    - [MLOG 完整清单](#mlog-完整清单)
    - [物理逻辑（physiological）特性：先读页再重放](#物理逻辑physiological特性先读页再重放)
    - [SMO 的列表级 redo](#smo-的列表级-redo)
    - [bulk load：`MTR_LOG_NO_REDO` + `MLOG_INDEX_LOAD`](#bulk-loadmtr_log_no_redo--mlog_index_load)
  - 回收与恢复
    - [空间回收：分批的 `fseg_free_step`](#空间回收分批的-fseg_free_step)
    - [DDL log：索引树的崩溃回滚](#ddl-log索引树的崩溃回滚)
    - [恢复视角：hash 聚合与"页是全新的"](#恢复视角hash-聚合与页是全新的)
  - 特殊场景
    - [压缩页 / 加密 / punch hole](#压缩页--加密--punch-hole)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

索引的物理存在 = **一棵 B 树**（root page 起）+ **两个段**（非叶段 + 叶子段）+ **一整套 redo 记录**（所有结构变化的持久化）+ **DDL log**（整棵树级操作的崩溃回滚）。索引的元数据侧（root page no 存哪）见 [`metadata.md`](metadata.md)，本篇从"页号"向下走到"字节"。

### 用途

- 理解索引页的布局（`PAGE_INDEX_ID` 等字段）；
- 理解 DROP INDEX 为什么慢（分批释放）；
- 理解建索引为什么不写 redo 也安全（DDL log 兜底）；
- 理解崩溃恢复时索引 redo 怎么被应用。

**★ 源码对"这个模块干什么"的自述**（`storage/innobase/btr/btr0btr.cc:80`，★ 文件头三段注释）：

```cpp
/*
Latching strategy of the InnoDB B-tree
--------------------------------------
A tree latch protects all non-leaf nodes of the tree. Each node of a tree
also has a latch of its own. ...
Node pointers
-------------
Leaf pages of a B-tree contain the index records stored in the
tree. On levels n > 0 we store 'node pointers' to pages on level
n - 1. For each page there is exactly one node pointer stored:
thus the our tree is an ordinary B-tree, not a B-link tree. ...
File page allocation
--------------------
In the root node of a B-tree there are two file segment headers.
The leaf pages of a tree are allocated from one file segment, to
make them consecutive on disk if possible. From the other file segment
we allocate pages for the non-leaf levels of the tree.
*/
```
> ★★ 最后一段就是"两段"设计的原始出处：根页上放两个 file segment header，叶子段追求物理连续，非叶段单独分配。
> ★ 中间那句 "**ordinary B-tree, not a B-link tree**" 也很关键——MySQL 的 B 树**没有**兄弟指针链（Bl-link 的 `link` 指针指的是 node pointer 结构，不是页级 sibling link），这与 PostgreSQL 的 B-link 树是根本差异。

**★ 版本演进：8.0 引入的四处改动**（★ 均已核实）：

| 变化 | 源码证据 |
|---|---|
| `dict_index_t::tree`（`btr_btr_t`）**已移除** | `dict0mem.h` 中 grep `tree` / `btr_btr` 仅命中注释；替代物是 `page` + `btr_root_block_get()` |
| 页类型细分 | `btr0btr.cc:905`：新增 `FIL_PAGE_RTREE` / `FIL_PAGE_SDI`，不再只有 `FIL_PAGE_INDEX` |
| extent 状态新增 `XDES_FSEG_FRAG = 5`（lease 机制） | `fsp0fsp.h:340` |
| `PAGE_MAX_TRX_ID` 更新**不再写 redo** | `page0page.cc:204` 注释：*"It is not necessary to write this change to the redo log, as during a database recovery we assume that the max trx id of every page is the maximum trx id assigned before the crash."* |

**★ 8.0 的 instant ADD/DROP 对物理层零影响**（`dict0mem.h:1127`，★ 这是"根页恒定"设计的最大红利）：

```cpp
  /** number of total fields in the index (including INSTANT dropped fields) */
  unsigned n_total_fields : 10;
  /** true if the index is clustered index and it has some instant columns */
  unsigned instant_cols : 1;
  /** true if the index is clustered index and table has row versions */
  unsigned row_versions : 1;
```
```cpp
// dict0mem.h:1180
  /** Array of field pos sorted as per their physical pos in record. Only
  needed for clustered index having INSTANT ADD/DROP columns. */
  std::vector<uint16_t> fields_array;
  /** Number of nullable columns in each version. ... */
  uint32_t nullables[MAX_ROW_VERSION + 1] = {0};
```
> ★★ instant ADD/DROP COLUMN 只改 `dict_index_t` 的**内存元数据**（`fields_array` / `nullables[]` 行版本），**物理页与根页号完全不动**——正因根页恒定，元数据 DDL 才能彻底免于重建 B 树。

**★ `btr_truncate()`：8.0 新增的"不动根页的清空"**（`btr0btr.cc:1041`）：

```cpp
  page_no_t root_page_no = index->page;      /* ★ 直接读 index->page */
  ...
  /* Mark that we are going to truncate the index tree
  We use PAGE_MAX_TRX_ID as it should be always 0 for clustered index. */
  mlog_write_ull(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), IB_ID_MAX, &mtr);
  mtr.commit();
  ...
  /* Free all except the root, we don't want to change it. */
  btr_free_but_not_root(block, MTR_LOG_ALL);
```
> ★★ 巧思：借 `PAGE_MAX_TRX_ID`（聚簇索引恒为 0）写 `IB_ID_MAX` 当"正在 truncate"的标记位——**先落盘意图、再做动作**的两阶段协议，崩溃后 `btr_truncate_recover()` 靠这个标记接着做完。同样是"根页恒定"的红利：标记可以放心写在根页上，因为根页号全程不变。

### 版本演进

- **8.0.30+ redo 格式统一**：8.0.39 里 `MLOG_*` 存在"旧 `_8027` 型"与"新统一型"两套并存——旧型只用于**读**旧 redo，真正生效的是 67–76 这组统一型（COMPACT 与否由 log 内 index flag 携带）。**不存在** `MLOG_COMP_REC_INSERT` 这类不带后缀的旧名。

> ★ **WL 检索结论（如实注明）**：在 `storage/innobase/btr|fsp|mtr|lob/` 与 `include/page0*|fsp0*` 全树 grep `WL#`，**仅 1 处命中**——`fsp/fsp0file.cc:603` 的 `TODO: Enable following after WL#11063`，与本篇主题（索引根页/段/页布局/redo）无直接关系。判断：索引的物理布局与段管理是 **InnoDB 自 4.x 延续至今的骨架**，**早于 MySQL Worklog 体系（Worklog 自 5.6 起才系统化使用）**，因此**没有对应编号**——不是漏查。
>
> ★ 但 redo 格式的版本化本身有编号：8.0.28 的"Row versioning header"与 8.0.30 的 `innodb_redo_log_capacity`（见 `log0types.h` 的 `Log_format` 枚举），对应 `MLOG_*_8027` → 67-76 的换号。这两条若要追 WL，应查 redo 主题而非索引主题（见 [`../innodb/redo_log.md`](../innodb/redo_log.md)）。

---

## 理论基础

### 设计思想与权衡

**1. 每个索引独立两段，root 页是非叶段的第一页。** 好处：DROP INDEX 时按段整体释放、不需要逐页收集；坏处：小索引也要 2 个段 inode + 若干 frag 页（`btr_create` 里两次 `fseg_create`）。

**2. root page no 恒定。** 树长高（`btr_root_raise_and_insert`）时**不换 root**——分配新页承载旧 root 内容、root 原地清空并 level+1。这使 DD 里的 `root` 键永不过期，是元数据与物理的稳定锚点。

**3. redo 是"物理逻辑"（physiological）**：不是字节 diff，也不是纯逻辑操作——是**"在页内重放一次操作"**（带 cursor 偏移 + 记录尾段）。恢复时必须**先读页**（payload 依赖页现状），换来的是 redo 体积小、且能容忍页内偏移变化。

**4. bulk load 不写逐页 redo，靠 DDL log。** 批量建索引时树页是"新造的"，逐页写 redo 无意义（恢复也用不上）——只写一条 `MLOG_INDEX_LOAD` 标记 + DDL log 记录"这棵树若没提交就删掉"。

**5. 空间释放是分批小步的。** `fseg_free_step` 每次只释放一个 extent / 一个 frag page / 一个 inode，跨多个 mtr 循环——单个 mtr 不能持有过多页锁、不能产生过大 redo。

**★ 五条的源码锚点**：

```cpp
/* ② 根页恒定：dict_index_t 只存 (space, page)，永不迁移 */
  unsigned space : 32;
  unsigned page : 32;                    /* dict0mem.h:1081-1085 */
```
```cpp
/* ⑤ 分批释放，源码注释直接给出理由（fsp0fsp.cc:3655） */
/** Frees part of a segment. This function can be used to free a segment by
 repeatedly calling this function in different mini-transactions. Doing
 the freeing in a single mini-transaction might result in too big a
 mini-transaction.                                    ★★ 官方理由
 @return true if freeing completed */
bool fseg_free_step(fseg_header_t *header, bool ahi, mtr_t *mtr)
```
```cpp
/* ③ 物理逻辑：11 字节头 = 定位到页，之后才是页内偏移 + 内容（mtr0log.h:61） */
/* Size of initial info on REDO log
  1   byte  for LOG TYPE
  3-5 bytes for SPACE ID
  3-5 bytes for PAGE OFFSET */
constexpr size_t REDO_LOG_INITIAL_INFO_SIZE = 11;
```
```cpp
/* ④ bulk load 豁免：MTR_LOG_NO_REDO 时 mlog_open 静默丢弃（mtr0log.cc:72） */
  /* If no logging is requested, we may return now */
  if (!mlog_open(mtr, REDO_LOG_INITIAL_INFO_SIZE, log_ptr)) {
    return;
  }
```
```cpp
/* ① 两段：文件头注释原话（btr0btr.cc:80，最后一段） */
/* In the root node of a B-tree there are two file segment headers.
The leaf pages of a tree are allocated from one file segment, to
make them consecutive on disk if possible. From the other file segment
we allocate pages for the non-leaf levels of the tree. */
```

**★ 第 2 条的另半边：root 恒定的三个"下游红利"**（★ 把分散的证据串起来）：

| 红利 | 源码 |
|---|---|
| **元数据永不过期** | `se_private_data["root"]` / `SYS_INDEXES.PAGE_NO` 全程无需更新 |
| **instant ADD/DROP 免重建** | `dict0mem.h:1127`：`instant_cols` / `row_versions` 只改内存元数据，物理页不动 |
| **`btr_truncate` 可原地清空** | `btr0btr.cc:1041`：借根页的 `PAGE_MAX_TRX_ID` 打标记，*"we don't want to change it"* |

> ★ 反过来说：**如果 root 不恒定，这三项都要重写**——这就是"锚点"设计的真正价值。

### ★ 为什么每个索引是"两段"而不是一段

每个索引创建时调两次 `fseg_create`，拿到 `PAGE_BTR_SEG_LEAF`（叶子段）与 `PAGE_BTR_SEG_TOP`（非叶段）。为什么要分开：

| 约束 | 说明 |
|---|---|
| **两段的增长模式完全不同** | 叶子段随数据量**线性**增长；非叶段随树高**对数**增长（一个非叶页能索引上千个叶子页）。混在一个段里会让空间分配策略无法各自优化 |
| **按段整体释放 = DROP INDEX 的代价可控** | 删索引时按段回收（`fseg_free_step`），不需要遍历收集页——否则删一个亿级行的索引要扫完整棵树 |
| **root 页要同时承载两个段头** | root 页的 `PAGE_BTR_SEG_LEAF` / `PAGE_BTR_SEG_TOP` 两个段头——这也是每个索引页都要预留段头空间的原因（任何页都可能成为新 root） |

★★ **段头的精确位置**（`page0types.h:90`，⚠️ 原文"各占 20 字节"是错的，段头是 **10 字节**）：

```cpp
/** file segment header for the leaf pages in a B-tree: defined only on the root
 page of a B-tree, but not in the root of an ibuf tree */
constexpr uint32_t PAGE_BTR_SEG_LEAF = 36;                    /* 相对 PAGE_HEADER 的偏移 */
constexpr uint32_t PAGE_BTR_SEG_TOP = 36 + FSEG_HEADER_SIZE;  /* = 46 */
/** start of data on the page */
constexpr uint32_t PAGE_DATA = PAGE_HEADER + 36 + 2 * FSEG_HEADER_SIZE;  /* = 94 */
```
```cpp
/* fsp0types.h:80 —— 段头定长 10 字节 */
constexpr uint32_t FSEG_HDR_SPACE = 0;      /* space id of the inode */
constexpr uint32_t FSEG_HDR_PAGE_NO = 4;    /* page number of the inode */
constexpr uint32_t FSEG_HDR_OFFSET = 8;     /* byte offset of the inode */
constexpr uint32_t FSEG_HEADER_SIZE = 10;
```
> ★ 换算成物理页偏移：`PAGE_HEADER == FIL_PAGE_DATA == 38`，所以 LEAF 段头在 **74**、TOP 段头在 **84**，两个共 20 字节，`PAGE_DATA`（用户记录区起点）从 **94** 开始。

**★★★ 一个必须纠正的认知：段头不在 `dict_index_t` 里**

`dict_index_t` **没有** `seg` 字段——只有 `space`（32 位）与 `page`（32 位，根页号）。段头是**磁盘上根页页头的物理字段**，每次访问要 `btr_root_get()` 拿根页再取偏移：

```cpp
/* btr0btr.cc:397 —— 分配时按 level 选段 */
  root = btr_root_get(index, mtr);                 /* ★ SX latch 只为读段头 */
  if (level == 0) {
    seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_LEAF;   /* ★ 叶子页 → leaf 段 */
  } else {
    seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_TOP;    /* ★ 内节点页 → top 段 */
  }
```

> ★ `btr_root_get()` 用 **SX latch 而非 X**——源码注释：*"SX lock doesn't block reading user data by other threads. And block the segment list access by others."* 段头读写不与用户记录读互斥。

**★ 两段是真实存在的两个资源池**（`btr0btr.cc:486`，统计口径直接证明）：

```cpp
  if (flag == BTR_N_LEAF_PAGES) {
    seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_LEAF;
    fseg_n_reserved_pages(seg_header, &n, mtr);      /* "叶子页数" = leaf 段保留页数 */
  } else if (flag == BTR_TOTAL_SIZE) {
    seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_TOP;
    n = fseg_n_reserved_pages(seg_header, &dummy, mtr);
    seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_LEAF;
    n += fseg_n_reserved_pages(seg_header, &dummy, mtr);  /* "总页数" = top + leaf */
  }
```

**★ 建索引时是先建 TOP 段再建 LEAF 段**（`btr0btr.cc:867`，★ 顺序与直觉相反）：

```cpp
  } else {
    block = fseg_create(space, 0, PAGE_HEADER + PAGE_BTR_SEG_TOP, mtr);  /* ① 先建非叶段，返回的那页就是根页 */
  }
  ...
    if (!fseg_create(space, page_no, PAGE_HEADER + PAGE_BTR_SEG_LEAF, mtr)) {  /* ② 再建叶子段，段头写进根页 */
```
> ★ 因为 `fseg_create(space, 0, ...)` 的第二个参数 0 表示"新段的第一个页由它分配"，那页**就成为根页**——所以必须先建 TOP 段才能拿到根页号。

**★ 源码里点名 "two segments" 的 FIXME**（`dict0crea.cc:432`，★ 两段是索引的两份物理资源）：

```cpp
    /* FIXME: Now writing ddl log after the index has been created,
    so if server crashes before the redo log gets persisted,
    there is no way to find the resources(two segments, etc.)
    allocated to this index. Since this is a rare case, living
    with it is acceptable */
```

代价：再小的索引也要占 2 个段 inode + 若干 frag 页——**索引多的小表会有明显的元数据开销**。

### ★ bulk load 豁免 redo：用 DDL 的原子性换持久性

第 4 条说批量建索引"不写逐页 redo，只写一条 `MLOG_INDEX_LOAD` + DDL log"。这背后的推理值得展开——**它是一次"持久性责任转移"**：

- 常规 DML 写 redo，是因为事务提交后数据必须持久化：崩溃时靠 redo 把"已提交但未落盘"的修改补回来；
- 批量建索引的页是**新造的**，且整个 DDL 是**原子的**：要么全部完成并可见，要么（崩溃/失败）整棵树被 DDL log 删掉。不存在"部分完成且已提交"这种中间态需要 redo 去修复；
- 所以 redo 的持久性责任被**转移给了 DDL log 的原子性**——崩溃恢复时看到 `MLOG_INDEX_LOAD` 标记就知道"这棵树属于一个未完成的 DDL"，直接丢弃即可。

★ 这是一个通用模式：**当操作本身是原子的（要么全有要么全无），就不需要为它的中间产物付持久性成本**。对比：online DDL 的 row log 不能豁免 redo（因为并发 DML 是已提交的事务，必须持久化）。

**★★ 源码实现（含三处真名纠正）**

```cpp
/* btr0load.cc:305 —— Page_load::init() */
  mtr->set_log_mode(MTR_LOG_NO_REDO);          /* ★★ 不是 MTR_LOG_NONE！ */
  mtr->set_flush_observer(m_flush_observer);   /* ★★ 换来的代价：脏页必须被 observer 记账并强制刷盘 */
```

> ★★ **`MTR_LOG_NO_REDO` ≠ `MTR_LOG_NONE`**（`mtr0types.h:41`）：

```cpp
enum mtr_log_t {
  MTR_LOG_ALL = 0,
  /** Log no operations and dirty pages are not added to the flush list */
  MTR_LOG_NONE = 1,
  /** Don't generate REDO log but add dirty pages to flush list */
  MTR_LOG_NO_REDO = 2,        /* ★ bulk load 用的是它 */
  MTR_LOG_SHORT_INSERTS = 3,
  MTR_LOG_MODE_MAX = 4
};
```
> ★ `MTR_LOG_NO_REDO` **仍入 flush list**，靠 `Flush_observer` 保证 DDL 提交前全部落盘。

**★★★ 关键豁免边界：空间分配仍必须记 redo**（`btr0load.cc:323`，★ 这个细节容易漏）：

```cpp
    /* We commit redo log for allocation by a separate mtr,
    because we don't guarantee pages are committed following
    the allocation order, and we will always generate redo log
    for page allocation, even when creating a new tablespace. */
    alloc_mtr.start();
    ...
    new_block = btr_page_alloc(m_index, 0, FSP_UP, m_level, &alloc_mtr, mtr);
    ...
    alloc_mtr.commit();       /* ★ 独立的 alloc_mtr，照常记 redo */
```
> ★ 豁免的只是**页内容**，不是**页分配**——段/区/extent 的元数据改动照常记 redo。

**★ 收尾只写一条 `MLOG_INDEX_LOAD`**（`ddl0builder.cc:1905`）：

```cpp
void Builder::write_redo(const dict_index_t *index) noexcept {
  mtr_t mtr;
  mtr.start();
  byte *log_ptr{};
  if (mlog_open(&mtr, 11 + 8, log_ptr)) {
    log_ptr = mlog_write_initial_log_record_low(MLOG_INDEX_LOAD, index->space,
                                                index->page, log_ptr, &mtr);
    mach_write_to_8(log_ptr, index->id);      /* ★ body 只有 8 字节 index_id */
    mlog_close(&mtr, log_ptr + 8);
  }
  mtr.commit();
}
```
> ★ 落点是**根页** `(space, index->page)`——整棵树"我加载完了"的唯一一条 redo。Doxygen 契约三要素：① 单页 redo 被禁用；② 这些页**已全部落盘**；③ 恢复时遇到它就无需重放该树任何页。

**★ DDL 兜底的真名**（⚠️ 不是 `ddl0log.cc`，那个文件不存在）：

```cpp
/* storage/innobase/include/log0ddl.h:407 */
/** Class to write and replay ddl logs */
class Log_DDL {
 public:
  /** Write DDL log for freeing B-tree ... */
  dberr_t write_free_tree_log(trx_t *trx, const dict_index_t *index,
                              bool is_drop_table);   /* ★ 崩溃时把半成品索引的两段整棵树回收 */
```

**⚠️ 三处真名纠正**：`btr0bulk.cc` / `Btr_bulk` **不存在**（8.0.39 的 bulk load 在 `btr0load.cc` 的 `Page_load`/`Btree_load` + `ddl/ddl0builder.cc` 的 `Builder`）；日志模式是 `MTR_LOG_NO_REDO`；DDL 兜底是 `log/log0ddl.cc` 的 `Log_DDL`。

**★ 临时表连 `btr_create` 都不记 redo**（`dict0crea.cc:417`）：

```cpp
  if (index->table->is_temporary()) {
    mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);
  }
  page_no = btr_create(index->type, index->space, index->id, index, &mtr);
```

### ★ 为什么空间释放必须分批小步（深化）

`fseg_free_step` 一次只释放一个 extent / frag page / inode，跨多个 mtr 循环。约束是 **mtr 的原子性边界**：

- 一个 mtr 持有的页锁数量与它产生的 redo 量是绑定的（mtr 提交时一次性写 redo）；
- 若一次释放整个段（可能上万个 extent），单 mtr 会持有巨量页锁、产生极大的单条 redo——既可能撑爆 redo buffer，也会长时间阻塞其他操作。

所以"分批小步"是**把一个大原子操作切成一串小原子操作**——代价是释放过程中若崩溃，需要靠 DDL log 继续完成（幂等重试）。

**★ 调用方的两个循环，每轮一个新 mtr**（`btr0btr.cc:954`，`btr_free_but_not_root()`）：

```cpp
leaf_loop:
  mtr_start(&mtr);
  mtr_set_log_mode(&mtr, log_mode);
  page_t *root = block->frame;
  finished = fseg_free_step(root + PAGE_HEADER + PAGE_BTR_SEG_LEAF, true, &mtr);
  mtr_commit(&mtr);                    /* ★★ 一轮 = 一个 mtr，立刻提交 */
  if (!finished) {
    goto leaf_loop;                    /* ★ 未完则重开 mtr */
  }
top_loop:
  mtr_start(&mtr);
  mtr_set_log_mode(&mtr, log_mode);
  root = block->frame;
  finished = fseg_free_step_not_header(root + PAGE_HEADER + PAGE_BTR_SEG_TOP,
                                       true, &mtr);   /* ★ 保留根页（段头还在上面） */
  mtr_commit(&mtr);
  if (!finished) {
    goto top_loop;
  }
```

> ★ 顺序也是先叶子段后非叶段，且非叶段用 `fseg_free_step_not_header()`——**因为段头就写在根页上，根页必须最后单独释放**。

**★ merge 阈值**（`btr0cur.h:609`，★ 默认 50% 但可 per-index 设）：

```cpp
#define BTR_CUR_PAGE_COMPRESS_LIMIT(index) \
  ((UNIV_PAGE_SIZE * (ulint)((index)->merge_threshold)) / 100)
```
> ★ `merge_threshold` 是 `dict_index_t` 里的 **6 位**字段（`dict0mem.h:1089`），默认 `DICT_INDEX_MERGE_THRESHOLD_DEFAULT = 50`。16 KB 页 → 阈值 8 KB。
> ★ 保护：`page_get_n_recs(page) < 2` 也触发 merge 判定之外，还有"页内至少留 2 条记录"的兜底（`btr0cur.ic:148`），避免树高反复抖动。

---

## 核心实现

### 主线与基础构件

#### 根页：恒定的物理锚点

`dict_index_t::page` 的三处赋值：

```cpp
// dict0crea.cc:394-451（节选）
dberr_t dict_create_index_tree_in_mem(dict_index_t *index, trx_t *trx) {
  mtr_start(&mtr);
  if (index->table->is_temporary()) mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);
  page_no = btr_create(index->type, index->space, index->id, index, &mtr);
  index->page = page_no;                    // ← root page no
  index->trx_id = trx->id;
  mtr_commit(&mtr);
  if (page_no == FIL_NULL) { err = DB_OUT_OF_FILE_SPACE; }
  else { err = log_ddl->write_free_tree_log(trx, index, false); }  // DDL log 兜底
  return err;
}
```

- **持久化位置**：`se_private_data["root"]`（见 [`metadata.md`](metadata.md)）；`SYS_INDEXES.PAGE_NO` 只在升级路径读；
- **root page no 恒定**的两处证据：`btr_lift_page_up()`（页合并时把 child 拷进 father，root 物理页仍是原来那一页）；`btr_root_raise_and_insert()`（长高时"分配新页承载旧 root 内容 → root 原地清空并 level+1"）——**都不改 `index->page`**；
- 其他赋值点：`row0mysql.cc:2518`（SDI/临时场景）、`btr0btr.cc:4675`（SDI 索引）、`ibuf0ibuf.cc:526`（ibuf 固定 `FSP_IBUF_TREE_ROOT_PAGE_NO`）、`row0import.cc`（IMPORT 从 .cfg 恢复）。

**★★ `dict_index_t` 里根页号的真身**（`include/dict0mem.h:1082`，⚠️ 不是 `page_no_t`）：

```cpp
struct dict_index_t {
  /** id of the index */
  space_index_t id;
  ...
  /** space where the index tree is placed */
  unsigned space : 32;      /* ⚠️ 32 位位域，不是 space_id_t */
  /** index tree root page number */
  unsigned page : 32;       /* ★★ 32 位位域，不是 page_no_t。
                                 这是整个 B-tree 唯一的"恒定物理锚点" */
  /** In the pessimistic delete, if the page data size drops below this limit
  in percent, merging it to a neighbor is tried */
  unsigned merge_threshold : 6;
  unsigned type : DICT_IT_BITS;
  /** position of the trx id column in a clustered index record, if the fields
  before it are known to be of a fixed size, 0 otherwise */
  unsigned trx_id_offset : MAX_KEY_LENGTH_BITS;
  ...
  /** read-write lock protecting the upper levels of the index tree */
  rw_lock_t lock;           /* ★ 是内嵌对象不是指针；保护"树的上层结构"即所有非叶层 */
```
> ⚠️ **真名核查**：`dict_index_t::tree`（`btr_btr_t`）**已在 5.7 移除**，8.0.39 不存在。

**★ 读取器 `dict_index_get_page()`**（`include/dict0dict.ic:791`，⚠️ **不存在** `dict_index_get_root()`，全仓 grep 0 命中）：

```cpp
/** Gets the page number of the root of the index tree.
 @return page number */
static inline page_no_t dict_index_get_page(
    const dict_index_t *index) {
  ut_ad(index);
  ut_ad(index->magic_n == DICT_INDEX_MAGIC_N);
  return (index->page);     /* ★ 把位域提升为 page_no_t 返回 */
}
```

**★ 根页号的持久化位置**（`include/dict0boot.h:207`，★ SYS_INDEXES 第 8 列）：

```cpp
enum dict_fld_sys_indexes_enum {
  DICT_FLD__SYS_INDEXES__TABLE_ID = 0,
  DICT_FLD__SYS_INDEXES__ID = 1,
  DICT_FLD__SYS_INDEXES__DB_TRX_ID = 2,
  DICT_FLD__SYS_INDEXES__DB_ROLL_PTR = 3,
  DICT_FLD__SYS_INDEXES__NAME = 4,
  DICT_FLD__SYS_INDEXES__N_FIELDS = 5,
  DICT_FLD__SYS_INDEXES__TYPE = 6,
  DICT_FLD__SYS_INDEXES__SPACE = 7,
  DICT_FLD__SYS_INDEXES__PAGE_NO = 8,        /* ★ 根页号，4 字节 */
  DICT_FLD__SYS_INDEXES__MERGE_THRESHOLD = 9, /* ★ 8.0 新增的列 */
  DICT_NUM_FIELDS__SYS_INDEXES = 10
};
```
```cpp
/* dict/dict0load.cc:436 —— 装载侧证据 */
  field = rec_get_nth_field_old(nullptr, rec, DICT_FLD__SYS_INDEXES__PAGE_NO, &len);
  if (len != 4) { ... }      /* ★ 4 字节，与 unsigned page:32 精确对应 */
```

**★ `btr_root_block_get()`：根页的唯一取页路径**（`btr/btr0btr.cc:151`）：

```cpp
buf_block_t *btr_root_block_get(const dict_index_t *index, ulint mode, mtr_t *mtr) {
  const space_id_t space_id = dict_index_get_space(index);
  const page_id_t page_id(space_id, dict_index_get_page(index));
  const page_size_t page_size(dict_table_page_size(index->table));

  buf_block_t *block =
      btr_block_get(page_id, page_size, mode, UT_LOCATION_HERE, index, mtr);

  btr_assert_not_corrupted(block, index);
#ifdef UNIV_BTR_DEBUG
  if (!dict_index_is_ibuf(index)) {
    const page_t *root = buf_block_get_frame(block);
    ut_a(btr_root_fseg_validate(FIL_PAGE_DATA + PAGE_BTR_SEG_LEAF + root, space_id));
    ut_a(btr_root_fseg_validate(FIL_PAGE_DATA + PAGE_BTR_SEG_TOP + root, space_id));
  }
#endif
  return (block);
}
```
> ★★ **根页定位 = `(index->space, index->page)` 两个内存字段直接拼出 `page_id`，没有任何"遍历/查找"**——这就是为什么根页号必须恒定：它是 O(1) 的物理地址。
> ★ `UNIV_BTR_DEBUG` 下还会顺带校验根页上的两个 segment header。

**★ 根页的"物理自证"：`page_is_root()`**（`include/page0page.ic:254`）：

```cpp
static inline bool page_is_root(const page_t *page) {
  /* Check that this is an index page and both the PREV and NEXT
  pointers are FIL_NULL, because the root page does not have any
  siblings. */
  return (fil_page_index_page_check(page) &&
          *reinterpret_cast<const uint64_t *>(page + FIL_PAGE_PREV) ==
              std::numeric_limits<uint64_t>::max());
}
```
> ★★ 根页在物理上可自证：**`FIL_PAGE_PREV`/`FIL_PAGE_NEXT` 两个 4B 拼成一个 u64 == 全 0xff**。`btr_create` 里 `btr_page_set_next/prev(FIL_NULL)` 就建立了这个不变量。所以"根页恒定"不只是设计约定，还能被恢复端验证。

**★ `btr_root_get()` 用 SX 而非 X**（`btr0btr.cc:181`，★ 一个加锁折中）：

```cpp
page_t *btr_root_get(const dict_index_t *index, mtr_t *mtr) {
  /* Intended to be used for segment list access.
  SX lock doesn't block reading user data by other threads.
  And block the segment list access by others.*/
  return (buf_block_get_frame(btr_root_block_get(index, RW_SX_LATCH, mtr)));
}
```
> ★ 这正是"根页同时是索引页又是段头载体"带来的折中：并发的用户记录读不被阻塞，只阻塞其它也要动段链表的人。

#### 段与区：每个索引两个段

`btr_create`（`btr0btr.cc:832-952`）核心流程：

```cpp
if (type & DICT_IBUF) {
  /* ibuf 树：只 1 个段，header 在 ibuf header page 上 */
  block = fseg_create(space, 0, IBUF_HEADER + IBUF_TREE_SEG_HEADER, mtr);
  block = fseg_alloc_free_page(..., IBUF_TREE_ROOT_PAGE_NO, FSP_UP, mtr);
} else {
  /* 段1 = 非叶(TOP)段；page=0 => 新分配一页，该页即 root，
     段头写在 root 的 PAGE_HEADER + PAGE_BTR_SEG_TOP */
  block = fseg_create(space, 0, PAGE_HEADER + PAGE_BTR_SEG_TOP, mtr);
}
...
if (!(type & DICT_IBUF)) {
  /* 段2 = 叶子段；段头写在 root 页的 PAGE_HEADER + PAGE_BTR_SEG_LEAF */
  if (!fseg_create(space, page_no, PAGE_HEADER + PAGE_BTR_SEG_LEAF, mtr)) {
    btr_free_root(block, mtr);              // 回滚：释放刚建的 TOP 段
    return FIL_NULL;
  }
}
/* 页类型：FIL_PAGE_RTREE / FIL_PAGE_SDI / 其余 FIL_PAGE_INDEX */
btr_page_set_index_id(page, page_zip, index_id, mtr);
```

- **root 页同时属于 TOP 段**（`fseg_create(space, 0, ...)` 时 `page==0` ⇒ 新分配的页属于新建段）；叶子段头写在 root 页的 `PAGE_BTR_SEG_LEAF`（root 页本身不属于叶子段）；
- `fseg_create_general`（`fsp0fsp.cc:2277`）预留 2 个 extent（inode 1 个 + 段 1 个），段 inode 存在表空间的 INODE 页；段先吃 frag page（32 slot），超了才整 extent 分配；
- **FSEG_HEADER（10B）**：`FSEG_HDR_SPACE`(4B) + `FSEG_HDR_PAGE_NO`(4B) + `FSEG_HDR_OFFSET`(2B)，指向段 inode。

**★ 段头的定义**（`include/fsp0types.h:80`，⚠️ **不是 16 字节，是 10 字节**）：

```cpp
/** On a page of any file segment, data
may be put starting from this offset */
constexpr uint32_t FSEG_PAGE_DATA = FIL_PAGE_DATA;  /* = 38 */

/** space id of the inode */
constexpr uint32_t FSEG_HDR_SPACE = 0;
/** page number of the inode */
constexpr uint32_t FSEG_HDR_PAGE_NO = 4;
/** byte offset of the inode */
constexpr uint32_t FSEG_HDR_OFFSET = 8;
/** Length of the file system  header, in bytes */
constexpr uint32_t FSEG_HEADER_SIZE = 10;       /* ★★ 10 不是 16 */
```
> ★ 段头 = inode 的"三级地址"：`space_id(4B) + page_no(4B) + offset(2B)`。根页上放两份：`PAGE_BTR_SEG_LEAF`(36) 与 `PAGE_BTR_SEG_TOP`(46)。

**★ 区大小**（`include/fsp0types.h:57`，★ 段→区→页的中间层）：

```cpp
/** File space extent size in pages
page size | file space extent size
----------+-----------------------
   4 KiB  | 256 pages = 1 MiB
   8 KiB  | 128 pages = 1 MiB
  16 KiB  |  64 pages = 1 MiB
  32 KiB  |  64 pages = 2 MiB
  64 KiB  |  64 pages = 4 MiB
*/
#define FSP_EXTENT_SIZE \
  static_cast<page_no_t>(((UNIV_PAGE_SIZE <= (16384) ? (1048576 / UNIV_PAGE_SIZE) \
      : ((UNIV_PAGE_SIZE <= (32768)) ? (2097152 / UNIV_PAGE_SIZE) \
                                     : (4194304 / UNIV_PAGE_SIZE)))))
```
> ★ 默认 16KiB 页 → **1 extent = 64 pages = 1MiB**。

**★★ 段(inode)的完整结构**（`include/fsp0fsp.h:191`，★ 段/区/页三层的源码级区分）：

```cpp
/*
偏移         字段                   说明
─────────────────────────────────────────────
  0         FSEG_ID (8B)          段 ID，0 表示未使用
  8         FSEG_NOT_FULL_N_USED  NOT_FULL 链表中已用页数
 12         FSEG_FREE             空闲 extent 链表头 (16B)
 28         FSEG_NOT_FULL         部分使用 extent 链表头 (16B)
 44         FSEG_FULL             满 extent 链表头 (16B)
 60         FSEG_MAGIC_N (4B)     魔数 97937874
 64         FSEG_FRAG_ARR         32 个碎片页槽位 (32×4B)
*/
constexpr uint32_t FSEG_ARR_OFFSET = FSEG_PAGE_DATA + FLST_NODE_SIZE;
constexpr uint32_t FSEG_ID = 0;
constexpr uint32_t FSEG_NOT_FULL_N_USED = 8;
constexpr uint32_t FSEG_FREE = 12;
constexpr uint32_t FSEG_NOT_FULL = 12 + FLST_BASE_NODE_SIZE;
constexpr uint32_t FSEG_FULL = 12 + 2 * FLST_BASE_NODE_SIZE;
constexpr uint32_t FSEG_MAGIC_N = 12 + 3 * FLST_BASE_NODE_SIZE;
constexpr uint32_t FSEG_FRAG_ARR = 16 + 3 * FLST_BASE_NODE_SIZE;
#define FSEG_FRAG_ARR_N_SLOTS (FSP_EXTENT_SIZE / 2)     /* ★ 32 个槽 */
constexpr uint32_t FSEG_FRAG_SLOT_SIZE = 4;
#define FSEG_INODE_SIZE \
  (16 + 3 * FLST_BASE_NODE_SIZE + FSEG_FRAG_ARR_N_SLOTS * FSEG_FRAG_SLOT_SIZE)
constexpr uint32_t FSEG_MAGIC_N_VALUE = 97937874;
```
> ★★ 一个 inode 约 **192 字节**：8B seg_id + 8B n_used + 3×16B extent 链表头 + 4B magic + 32×4B 碎片页槽。它"拥有"的区通过 `FSEG_FREE`/`NOT_FULL`/`FULL` **三条链表**组织；不足一个区的零散页放在 `FSEG_FRAG_ARR` 的 32 个槽里。

**★ XDES 状态：区归谁**（`include/fsp0fsp.h:320`，★ 三层区分一句话）：

```cpp
enum xdes_state_t {
  XDES_NOT_INITED = 0,   /** extent descriptor is not initialized */
  XDES_FREE = 1,         /** extent is in free list of space */
  XDES_FREE_FRAG = 2,    /** extent is in free fragment list of space */
  XDES_FULL_FRAG = 3,    /** extent is in full fragment list of space */
  XDES_FSEG = 4,         /** extent belongs to a segment */
  XDES_FSEG_FRAG = 5     /** fragment extent leased to segment  ★ 8.0 新增 */
};
/** How many bits are there per page */
constexpr uint32_t XDES_BITS_PER_PAGE = 2;   /* free bit + clean bit */
```
> ★★ **段(segment)** = `fseg_inode_t`，逻辑归属单位，由 inode 的 3 条链表引用若干 extent；**区(extent)** = 64 个连续页，由 `xdes_t` 描述，state 决定"属于 FSP 还是属于某个段"；**页(page)** = 16KiB，其归属由所在 extent 的 `XDES_ID` 间接决定（碎片页则由 `FSEG_FRAG_ARR` 直接记页号）。

**★★ 段头写入：三次 `mlog_write_ulint`，没有 `fseg_header_put()`**（`fsp/fsp0fsp.cc:2401`，⚠️ 该函数不存在）：

```cpp
  // 写 (space_id, page_no, offset) 三级地址
  mlog_write_ulint(header + FSEG_HDR_OFFSET, page_offset(inode), MLOG_2BYTES, mtr);
  mlog_write_ulint(header + FSEG_HDR_PAGE_NO,
                   page_get_page_no(page_align(inode)), MLOG_4BYTES, mtr);
  mlog_write_ulint(header + FSEG_HDR_SPACE, space_id, MLOG_4BYTES, mtr);
```
> ★ 三次写入**全带 redo**（`mlog_write_ulint`）。写在 `byte_offset` 处，`btr_create` 传的是 `PAGE_HEADER+PAGE_BTR_SEG_TOP` 或 `+PAGE_BTR_SEG_LEAF`。

**★ 建段先预留 2 个 extent**（`fsp/fsp0fsp.cc:2330`，★ CREATE INDEX 空间不足的源头）：

```cpp
  if (!has_done_reservation) {
    fsp_reserve_t alloc_type =
        (fsp_is_undo_tablespace(space_id) ? FSP_UNDO : FSP_NORMAL);
    // 创建 segment 时，一般默认预留 2 个 extent，
    // 但如果当前的文件小于 1 个 extent 时，则只预留 2 个 page。
    if (!fsp_reserve_free_extents(&n_reserved, space_id, 2, alloc_type, mtr)) {
      return nullptr;    /* ★ 空间不足 → 返回 NULL → btr_create 返回 FIL_NULL */
    }
  }
  ...
  seg_id = mach_read_from_8(space_header + FSP_SEG_ID);
  mlog_write_ull(space_header + FSP_SEG_ID, seg_id + 1, mtr);   /* ★ 段号自增 */
  mlog_write_ull(inode + FSEG_ID, seg_id, mtr);
  ...
  flst_init(inode + FSEG_FREE, mtr);
  flst_init(inode + FSEG_NOT_FULL, mtr);
  flst_init(inode + FSEG_FULL, mtr);       /* ★ 新段三条链表全空，只靠碎片页起步 */
```
> ★ `fseg_create()` 是薄封装，传 `has_done_reservation=false`，所以 `btr_create` 路径上每次都由 `fseg_create_general` 内部现预留。

**★ `btr_create()` 两处 `fseg_create` 的分工**（`btr/btr0btr.cc:832`，★ 关键差异）：

```cpp
  } else {
    block = fseg_create(space, 0, PAGE_HEADER + PAGE_BTR_SEG_TOP, mtr);
  }
  page_no = block->page.id.page_no();      /* ★ 根页号在此刻确定（典型是 4） */
  ...
    if (!fseg_create(space, page_no, PAGE_HEADER + PAGE_BTR_SEG_LEAF, mtr)) {
      /* Not enough space for new segment, free root segment before return. */
      btr_free_root(block, mtr);
      ...
      return (FIL_NULL);
    }
```
> ★★ 第 1 处 `page=0` ⇒ 段自己分配新页，**该页即根页**，段头写 `PAGE_BTR_SEG_TOP`；第 2 处 `page=page_no`（已存在的根页）⇒ 建叶子段，段头写 `PAGE_BTR_SEG_LEAF`。
> ★ 于是根页上同时挂着两个段头：`SEG_LEAF`(36,10B) + `SEG_TOP`(46,10B)。**根页属于 TOP 段，不属于叶子段**。
> ★ 之后 `btr_page_set_level(page, nullptr, 0, mtr)` ⇒ 新根页 level=0（既是根也是叶子）；`btr_page_set_index_id(page, page_zip, index_id, mtr)` ⇒ `PAGE_INDEX_ID = index_id`。

**★ inode 页的容量**（`include/fsp0fsp.h:250`）：

```cpp
static inline uint32_t FSP_SEG_INODES_PER_PAGE(page_size_t page_size) {
  return (page_size.physical() - FSEG_ARR_OFFSET - 10) / FSEG_INODE_SIZE;
}
```
> ★ 默认 16KiB 页 → 约 **85 个 inode/页**。一个索引 2 个段，所以一张 INODE 页够约 42 个索引用。

root 页上的段头偏移（16K 页，绝对偏移）：

| 常量 | 相对 PAGE_HEADER | 绝对偏移 | 含义 |
|---|---|---|---|
| `PAGE_BTR_SEG_LEAF` | 36 | 74 | 叶子段 FSEG_HEADER |
| `PAGE_BTR_SEG_TOP` | 46 | 84 | 非叶段 FSEG_HEADER |
| `PAGE_DATA` | 56 | 94 | 记录区起点 |

> 所有索引页（含非 root）都预留了这 56 字节的段头空间——这是"**每个索引页都能成为 root**"的前提。

**索引在表空间里的组织**：同一表空间多索引共存，每索引独立两段、页号交错是常态，靠页头 `PAGE_INDEX_ID`（8B）区分归属；file-per-table 下所有索引 + SDI 索引在一个 `.ibd`；SDI 索引是 `DICT_SDI` 类型的独立树（页类型 `FIL_PAGE_SDI`）。

#### 索引页布局

索引页 = FIL Header(38B) + PAGE_HEADER(56B) + 记录堆 + Page Directory（自页尾向前）+ FIL Trailer(8B)。

> PAGE_HEADER 的**逐字段 layout 以 [`../innodb/physical/page_structure.md`](../innodb/physical/page_structure.md) 为准**（页结构是那一篇的第一主语，含 12 类页的完整 layout）。本篇只补**索引语义**相关的字段：

| 字段 | 偏移 | 索引语义（本篇视角） |
|---|---|---|
| `PAGE_INDEX_ID` | 66 | 所属 index id（8B）——同一表空间多索引页号交错，靠它区分归属 |
| `PAGE_MAX_TRX_ID` | 56 | 可能修改本页的最大 trx_id，**仅二级索引/ibuf 用**（聚簇恒 0——故 `btr_truncate` 借它打标记） |
| `PAGE_LEVEL` | 64 | 层号，叶层=0；root 页恒定（长高不换根） |
| `PAGE_BTR_SEG_LEAF` / `PAGE_BTR_SEG_TOP` | 74 / 84 | 两段头（各 10B），**仅 root 有效**；所有索引页都预留这 20 字节段头空间，是"每个索引页都能成为 root"的前提 |

**★ PAGE_HEADER 全字段原文**（`page0types.h:45`，★ 相对 `PAGE_HEADER == FIL_PAGE_DATA == 38` 的偏移）：

```cpp
typedef byte page_header_t;
/** index page header starts at this offset */
constexpr uint32_t PAGE_HEADER = FSEG_PAGE_DATA;          /* = FIL_PAGE_DATA = 38 */

constexpr uint32_t PAGE_N_DIR_SLOTS = 0;   /* 2B 页目录槽数 */
constexpr uint32_t PAGE_HEAP_TOP = 2;      /* 2B 堆顶偏移 */
constexpr uint32_t PAGE_N_HEAP = 4;        /* 2B 记录数，bit15 = compact 格式标志 */
constexpr uint32_t PAGE_FREE = 6;          /* 2B 已删除记录链表头 */
constexpr uint32_t PAGE_GARBAGE = 8;       /* 2B 垃圾字节数 */
constexpr uint32_t PAGE_LAST_INSERT = 10;  /* 2B 最后插入位置（"同方向插入"快速路径） */
constexpr uint32_t PAGE_DIRECTION = 12;    /* 2B */
constexpr uint32_t PAGE_N_DIRECTION = 14;  /* 2B 连续同方向插入次数 */
constexpr uint32_t PAGE_N_RECS = 16;       /* 2B 用户记录数 */
constexpr uint32_t PAGE_MAX_TRX_ID = 18;   /* 8B 仅二级索引/ibuf 用 */
/** end of private data structure of the page header which are set in a page create */
constexpr uint32_t PAGE_HEADER_PRIV_END = 26;             /* ★ 0..25 是"page create 时设置"的 */
constexpr uint32_t PAGE_LEVEL = 26;        /* 2B 叶层=0 */
constexpr uint32_t PAGE_INDEX_ID = 28;     /* 8B */
constexpr uint32_t PAGE_BTR_SEG_LEAF = 36;
constexpr uint32_t PAGE_BTR_SEG_TOP = 36 + FSEG_HEADER_SIZE;   /* 46 */
constexpr uint32_t PAGE_DATA = PAGE_HEADER + 36 + 2 * FSEG_HEADER_SIZE;  /* 94 */
```

```cpp
/* --- 常量 --- */
constexpr ulint PAGE_HEAP_NO_INFIMUM = 0;      /* infimum 的 heap_no 恒为 0 */
constexpr ulint PAGE_HEAP_NO_SUPREMUM = 1;     /* supremum 恒为 1 */
constexpr ulint PAGE_HEAP_NO_USER_LOW = 2;     /* 用户记录从 2 起 */
enum cursor_direction_t : uint8_t {
  PAGE_LEFT = 1, PAGE_RIGHT = 2, PAGE_SAME_REC = 3,
  PAGE_SAME_PAGE = 4, PAGE_NO_DIRECTION = 5
};
```

> ★ `PAGE_HEADER_PRIV_END = 26` 这个分界线有意思：**0..25 是"建页时设置、之后可变"的运行时字段**，26 起（`PAGE_LEVEL`/`PAGE_INDEX_ID`/段头）是**建页后不再改**的静态字段（注释明说 *"should not be written to after page creation"*）。

**★★ "根页恒定"的完整链条**：

```
dict_index_t::page (内存, 恒定) ──→ 根页 ──┬─ PAGE_BTR_SEG_LEAF (74) ─→ 叶子段 inode ─→ 所有叶子页
                                           └─ PAGE_BTR_SEG_TOP  (84) ─→ 非叶段 inode ─→ 所有内节点页
                                      每页都写入同一个 index->id 到自己的 PAGE_INDEX_ID (66)
```

> ★ 所以"索引"在物理上 = 根页 + 两段 + 所有 `PAGE_INDEX_ID == index->id` 的页。**每页自带 index_id**，恢复/校验时可反向核对归属。

```cpp
/* btr0btr.cc:151 —— 根页获取 */
  const page_id_t page_id(space_id, dict_index_get_page(index));   /* ★ (space, page) 永不迁移 */
  buf_block_t *block = btr_block_get(page_id, page_size, mode, ...);
```

**★★ `PAGE_HEADER` 全部字段**（`include/page0types.h:45`，⚠️ 8.0 已全部改成 `constexpr uint32_t` 并**移到 `page0types.h`**）：

```cpp
/** index page header starts at this offset */
constexpr uint32_t PAGE_HEADER = FSEG_PAGE_DATA;   /* = FIL_PAGE_DATA = 38 */
                                                   /* ⚠️ 不存在 PAGE_HEADER_SIZE 常量 */
/** number of slots in page directory */
constexpr uint32_t PAGE_N_DIR_SLOTS = 0;
/** pointer to record heap top */
constexpr uint32_t PAGE_HEAP_TOP = 2;
/** number of records in the heap, bit 15=flag: new-style compact page format */
constexpr uint32_t PAGE_N_HEAP = 4;
/** pointer to start of page free record list */
constexpr uint32_t PAGE_FREE = 6;
/** number of bytes in deleted records */
constexpr uint32_t PAGE_GARBAGE = 8;
/** pointer to the last inserted record, or NULL if this info has been reset by
 a delete, for example */
constexpr uint32_t PAGE_LAST_INSERT = 10;
/** last insert direction: PAGE_LEFT, ... */
constexpr uint32_t PAGE_DIRECTION = 12;
/** number of consecutive inserts to the same direction */
constexpr uint32_t PAGE_N_DIRECTION = 14;
/** number of user records on the page */
constexpr uint32_t PAGE_N_RECS = 16;
/** highest id of a trx which may have modified a record on the page; trx_id_t;
defined only in secondary indexes and in the insert buffer tree */
constexpr uint32_t PAGE_MAX_TRX_ID = 18;
/** end of private data structure of the page header which are set in a page create */
constexpr uint32_t PAGE_HEADER_PRIV_END = 26;
/** level of the node in an index tree; the leaf level is the level 0.
This field should not be written to after page creation. */
constexpr uint32_t PAGE_LEVEL = 26;          /* ★ 2B；根页 level = 树高-1 */
/** index id where the page belongs. This field should not be written to after
 page creation. */
constexpr uint32_t PAGE_INDEX_ID = 28;       /* ★ 8B；页 → 索引的物理归属 */
/** file segment header for the leaf pages in a B-tree: defined only on the root
 page of a B-tree, but not in the root of an ibuf tree */
constexpr uint32_t PAGE_BTR_SEG_LEAF = 36;
constexpr uint32_t PAGE_BTR_IBUF_FREE_LIST = PAGE_BTR_SEG_LEAF;
constexpr uint32_t PAGE_BTR_SEG_TOP = 36 + FSEG_HEADER_SIZE;   /* = 46 */
/** start of data on the page */
constexpr uint32_t PAGE_DATA = PAGE_HEADER + 36 + 2 * FSEG_HEADER_SIZE;  /* = 94 */
/** offset of the page infimum record on an old-style page */
#define PAGE_OLD_INFIMUM (PAGE_DATA + 1 + REC_N_OLD_EXTRA_BYTES)
/** offset of the page infimum record on a new-style compact page */
#define PAGE_NEW_INFIMUM (PAGE_DATA + REC_N_NEW_EXTRA_BYTES)
#define PAGE_NEW_SUPREMUM (PAGE_DATA + 2 * REC_N_NEW_EXTRA_BYTES + 8)
```
> ★ `PAGE_DATA = 38 + 36 + 20 = 94`。之后才是 infimum/supremum 与用户记录。
> ★ `PAGE_OLD_*` 是 REDUNDANT 行格式，`PAGE_NEW_*` 是 COMPACT/DYNAMIC——8.0 建表默认 COMPACT，只有老表或 sys 系统表（如 `SYS_INDEXES`）才走 `OLD_*`。
> ★ 两个 "should not be written to after page creation" 字段（`PAGE_LEVEL` / `PAGE_INDEX_ID`）**在 `PAGE_HEADER_PRIV_END`(26) 之后**——见下面的 `page_create_low()`。

**★ `FIL_PAGE_DATA = 38`**（`include/fil0types.h:109`，★ PAGE_HEADER 的物理基址）：

```cpp
/** start of the data on the page */
constexpr uint32_t FIL_PAGE_DATA = 38;
/** size of the page trailer */
constexpr uint32_t FIL_PAGE_DATA_END = 8;
```
> ★ 可用区间是 `[38, UNIV_PAGE_SIZE - 8]`。

**★★ `page_create_low()` 只清 26 字节"私有区"**（`page/page0page.cc:304`，★ 一个精妙的细节）：

```cpp
  memset(page + PAGE_HEADER, 0, PAGE_HEADER_PRIV_END);   /* ★ 只清 [38, 64) */
  page[PAGE_HEADER + PAGE_N_DIR_SLOTS + 1] = 2;          /* 2 个槽：infimum+supremum */
  page[PAGE_HEADER + PAGE_DIRECTION + 1] = PAGE_NO_DIRECTION;

  if (comp) {
    page[PAGE_HEADER + PAGE_N_HEAP] = 0x80;              /* ★ page_is_comp() 读这一位 */
    page[PAGE_HEADER + PAGE_N_HEAP + 1] = PAGE_HEAP_NO_USER_LOW;
    page[PAGE_HEADER + PAGE_HEAP_TOP + 1] = PAGE_NEW_SUPREMUM_END;
    memcpy(page + PAGE_DATA, infimum_supremum_compact, sizeof infimum_supremum_compact);
    ...
  } else {
    memcpy(page + PAGE_DATA, infimum_supremum_redundant, ...);
  }
```
> ★★ `PAGE_LEVEL`(26) 与 `PAGE_INDEX_ID`(28) **不在这里清**——它们"不在 page create 的私有区"，由调用方 `btr_create` 显式设置（`btr_page_set_level` / `btr_page_set_index_id`）。这正是注释 "should not be written to after page creation" 的另一面。
> ★ `PAGE_N_HEAP` 最高位 `0x80` 是 compact 标志位，低 15 位是 heap 记录数，初始 2。

**★ 压缩页的 level 不走 mlog**（`page/page0page.cc:381`，★ 与非压缩页的对比）：

```cpp
  page = page_create_low(block, true, page_type);
  mach_write_to_2(PAGE_HEADER + PAGE_LEVEL + page, level);
  mach_write_to_8(PAGE_HEADER + PAGE_MAX_TRX_ID + page, max_trx_id);
  /* ★ 用 mach_write_* 而不是 mlog_write_* —— 因为整页会被 page_zip_compress
     写进整页重做记录，无需逐字段 redo */
  if (!page_zip_compress(page_zip, page, index, page_zip_level, mtr)) {
    ut_error;   /* The compression of a newly created page should always succeed. */
  }
```

**★ `PAGE_MAX_TRX_ID` 更新不写 redo**（`page/page0page.cc:192`，★ 8.0 的优化）：

```cpp
  /* It is not necessary to write this change to the redo log, as
  during a database recovery we assume that the max trx id of every
  page is the maximum trx id assigned before the crash. */
  if (page_zip) {
    mach_write_to_8(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), trx_id);
    page_zip_write_header(page_zip, ..., 8, mtr);
  } else if (mtr) {
    mlog_write_ull(page + (PAGE_HEADER + PAGE_MAX_TRX_ID), trx_id, mtr);
  ...
```
> ★★ 与 `PAGE_LEVEL` **必须写 redo** 形成鲜明对比：`btr_page_set_level()` 走 `mlog_write_ulint(..., MLOG_2BYTES, mtr)`。原因：level 是树结构的一部分，错了整棵树就错；而 max_trx_id 只用于 MVCC 可见性判断，恢复时可以"保守取崩溃前最大 trx id"。

#### 树高度：现算不缓存

```cpp
// btr0btr.cc:196-219
ulint btr_height_get(dict_index_t *index, mtr_t *mtr) {
  root_block = btr_root_block_get(index, RW_S_LATCH, mtr);
  height = btr_page_get_level(buf_block_get_frame(root_block));  // ← 读 PAGE_LEVEL
  return height;
}
```

`dict_index_t` **没有高度字段**——每次读 root 页现算。上限：`BTR_MAX_NODE_LEVEL = 45`（超过拒绝插入）、`BTR_MAX_LEVELS = 100`（路径数组容量）。唯一缓存高度的是 ibuf 的私有结构。

**★★ `btr_height_get()` 完整实现**（`btr/btr0btr.cc:192`，★ 三条关键事实）：

```cpp
ulint btr_height_get(dict_index_t *index, mtr_t *mtr) {
  ulint height;
  buf_block_t *root_block;

  ut_ad(srv_read_only_mode ||
        mtr_memo_contains_flagged(
            mtr, dict_index_get_lock(index),
            MTR_MEMO_S_LOCK | MTR_MEMO_X_LOCK | MTR_MEMO_SX_LOCK) ||
        index->table->is_intrinsic());

  /* S latches the page */
  root_block = btr_root_block_get(index, RW_S_LATCH, mtr);
  height = btr_page_get_level(buf_block_get_frame(root_block));

  /* Release the S latch on the root page. */
  mtr->memo_release(root_block, MTR_MEMO_PAGE_S_FIX);

  ut_d(sync_check_unlock(&root_block->lock));
  return (height);
}
```
> ★★ ① **高度 == 根页的 `PAGE_LEVEL`**，因为叶子层被定义为 0；② **只读 1 个页（根页）**，代价是 1 次 `buf_page_get` + 1 次 S latch + 1 次 2 字节读——**与树高/页数无关，不存在"逐级递减"**（"递减"的说法来自 `btr_validate_level` 的逐层下钻，不是高度查询）；③ 读完立即 `memo_release` 放掉根页 latch，刻意做成"轻量瞬时读取"。
> ★ 前置 `ut_ad` 要求持 `index->lock` 的 S/X/SX，保证读到的高度不被并发 SMO 撕裂。

**★ `btr_page_get_level()` 本身不读页**（`include/btr0btr.ic:94`）：

```cpp
static inline ulint btr_page_get_level(const page_t *page) {
  ulint level;
  ut_ad(page);
  level = mach_read_from_2(page + PAGE_HEADER + PAGE_LEVEL);
  ut_ad(level <= BTR_MAX_NODE_LEVEL);
  return (level);
}
```
> ★ 参数已经是 buffer pool 里的 page frame 指针。"每次读页"发生在调用方 `btr_root_block_get` → `buf_page_get_gen`。所以"现算" = **每次都从物理页的 2 字节字段现读，而不是从内存对象缓存读**。
> ⚠️ **不存在** `page_header_set_level()`，level 的 setter 只有 `btr_page_set_level()`。

**★ `BTR_MAX_NODE_LEVEL` 的来历**（`btr0btr.ic:41`，★ 一个真实的 bug 故事）：

```cpp
/** NOTE - Changing this from the original number of 50 to 45 as
insert_debug.test was failing in ASAN build because of a stack overflow issue.
It was found that rtr_info_t was taking up a lot of stack space in the function
btr_insert_on_non_leaf_level_func which is part of the recursive stack trace. */
/** Maximum B-tree page level (not really a hard limit). Used in debug
 assertions in btr_page_set_level and btr_page_get_level */
constexpr uint32_t BTR_MAX_NODE_LEVEL = 45;
```
> ★ "not really a hard limit"——45 只是 assert 用。真正触发限制的是 `row0ins.cc:2379` 的 `btr_height_get(index) >= BTR_MAX_NODE_LEVEL` 报错路径。原值是 50，因 ASAN 下递归爆栈改成 45。

**★★ "完全没有缓存"的 grep 证据**（★ 负证据）：

```cpp
/* storage/innobase/include/dict0mem.h —— grep "height" → 0 命中 */
/* 真实读取点：dict/dict0stats.cc:1721 */
  root_level = btr_height_get(index, &mtr);
```
```cpp
/* dict0stats.cc:1818 —— ★ 统计采样中"再调一次"判断树是否被并发改形 */
    if (root_level != btr_height_get(index, &mtr)) {
      /* Just quit if the tree has changed beyond
      recognition here. The old stats from previous
      runs will remain in the values that we have
      ...
```
> ★★ 统计采样过程中**再调一次** `btr_height_get` 来检测"树是否被并发 SMO 改形"——这正说明高度是每次现算的瞬时值，**没有任何缓存/版本号可以比对**。

**★ `dict_index_t` 里确实没有物理容量字段**（★ 一个常见误解）：

```cpp
  /** approximate index size in database pages */
  ulint stat_index_size;         /* ★ 是"估算值"（persistent stats 抽样算的），不是实算 */
  /** approximate number of leaf pages in the index tree */
  ulint stat_n_leaf_pages;       /* ★ 同上，估算值 */
```
> ★ 真实的物理大小要看 `fseg_n_reserved_pages()`（读 inode 的三条链表实算）。`stat_*` 只是给优化器用的估值。

---

### 持久化：索引的 redo 体系

#### MLOG 完整清单

8.0.39 两套并存（`mtr0types.h:63-274`）：旧 `_8027` 型只用于读旧 redo；**真正生效的是统一型 67–76**：

| 类型 | 值 | 语义 |
|---|---|---|
| `MLOG_REC_INSERT` | 67 | 插入（COMP/REDUNDANT 由 log 内 index flag 区分） |
| `MLOG_REC_CLUST_DELETE_MARK` | 68 | 聚簇 delete-mark |
| `MLOG_REC_DELETE` | 69 | 物理删除 |
| `MLOG_REC_UPDATE_IN_PLACE` | 70 | 原地更新（R-tree MBR 扩大也用它） |
| `MLOG_LIST_END_COPY_CREATED` | 71 | 列表拷贝到新页（分裂） |
| `MLOG_PAGE_REORGANIZE` | 72 | 页重组织（**非压缩页 payload 为 0**） |
| `MLOG_ZIP_PAGE_REORGANIZE` | 73 | 压缩页重组织（payload 仅 1B z_level） |
| `MLOG_ZIP_PAGE_COMPRESS_NO_DATA` | 74 | "请重压缩"（不带镜像） |
| `MLOG_LIST_END_DELETE` / `MLOG_LIST_START_DELETE` | 75 / 76 | 删尾/删头列表（payload 仅 2B 记录偏移） |

**页创建类**：`MLOG_PAGE_CREATE`(19) / `MLOG_COMP_PAGE_CREATE`(37)（无 payload，恢复时整页重建空页）、`MLOG_PAGE_CREATE_RTREE`/`MLOG_COMP_PAGE_CREATE_RTREE`(57/58)、`MLOG_PAGE_CREATE_SDI`/`MLOG_COMP_PAGE_CREATE_SDI`(63/64)——由 `page_create_write_log` 按页类型分派。**bulk load 标记**：`MLOG_INDEX_LOAD`(61)。

**node pointer 与 R-tree MBR 不新增 redo 类型**：node ptr 是普通记录，插入走 `MLOG_REC_INSERT`；R-tree MBR 原地扩大复用 `MLOG_REC_UPDATE_IN_PLACE`（`rtr_update_mbr_field_in_place`，`gis0rtree.cc:181`，注释明说"今后可能需要新类型"）。

**★★ 枚举原文片段**（`mtr0types.h:59`，★ 三处最值得记住的注释）：

```cpp
enum mlog_id_t {
  /** if the mtr contains only one log record for one page,
  i.e., write_initial_log_record has been called only once,
  this flag is ORed to the type of that first log record */
  MLOG_SINGLE_REC_FLAG = 128,       /* ★ OR 到类型字节上；解析时 & ~128 还原 */

  /** one byte is written */
  MLOG_1BYTE = 1,                   /* ★ 值即长度：1/2/4/8 既是类型也是字节数 */
  MLOG_2BYTES = 2,
  MLOG_4BYTES = 4,
  MLOG_8BYTES = 8,

  MLOG_REC_INSERT_8027 = 9,         /* ★ 8.0.27 及之前的格式，冻结保留 */
  ...
  /** this means that a file page is taken into use and the prior
  contents of the page should be ignored: in recovery we must not
  trust the lsn values stored to the file page.
  Note: it's deprecated because it causes crash recovery problem
  in bulk create index, and actually we don't need to reset page
  lsn in recv_recover_page_func() now. */
  MLOG_INIT_FILE_PAGE = 29,         /* ★★ 已废弃：注释明说给 bulk create index 造成崩溃恢复问题 */
  ...
  /** this means that a file page is taken into use.
  We use it to replace MLOG_INIT_FILE_PAGE. */
  MLOG_INIT_FILE_PAGE2 = 59,        /* ★★ 现行"页被启用"记录 */
  ...
  /** notify that an index tree is being loaded without writing
  redo log about individual pages */
  MLOG_INDEX_LOAD = 61,             /* ★ bulk load 豁免标记 */
  ...
  MLOG_REC_INSERT = 67,             /* ★ 8.0.28+ 统一新格式 */
  ...
  MLOG_LIST_START_DELETE = 76,
  /** biggest value (used in assertions) */
  MLOG_BIGGEST_TYPE = MLOG_LIST_START_DELETE
};
```

**★★★ 为什么会有 `_8027` 后缀**（★ 本篇最重要的版本证据）：

> **8.0.27 及之前的 redo 格式被冻结保留**（值 9-18、38-46、52-53），**8.0.28 起改用无后缀的新号段（67-76）**。
> 原因：8.0.28 引入的 **instant add column** 需要 redo 里携带额外元数据，新格式统一处理；旧格式只用于**向后兼容**（读旧 redo）。
> ★ 解析入口真名：`mlog_parse_index_8027()`（`mtr0log.cc:414`）。

**★ redo 文件格式版本枚举**（`log0types.h:137`，★ 与 MLOG 换号同代）：

```cpp
enum class Log_format : uint32_t {
  LEGACY = 0,
  VERSION_5_7_9 = 1,
  VERSION_8_0_1 = 2,     /* Remove MLOG_FILE_NAME and MLOG_CHECKPOINT, introduce MLOG_FILE_OPEN */
  VERSION_8_0_3 = 3,
  VERSION_8_0_19 = 4,    /* Expand ulint compressed form */
  VERSION_8_0_28 = 5,    /* Row versioning header  ★ 与 MLOG_*_8027 → 67-76 同代 */
  VERSION_8_0_30 = 6,    /* innodb_redo_log_capacity */
  CURRENT = VERSION_8_0_30
};
```

**★ `MLOG_SINGLE_REC_FLAG` 怎么剥离**（`mtr0log.cc:132`）：

```cpp
  *type = (mlog_id_t)((ulint)*ptr & ~MLOG_SINGLE_REC_FLAG);   /* 去 128 位 */
  ptr++;
  *space = mach_parse_compressed(&ptr, end_ptr);   /* ★ 每条记录自带 space_id */
  *page_no = mach_parse_compressed(&ptr, end_ptr); /* ★ 和 page_no → 才能做 hash 聚合 */
```

**★ 分类速查**：

| 类别 | 类型（值） |
|---|---|
| 索引页结构 | `MLOG_PAGE_CREATE`(19)、`MLOG_COMP_PAGE_CREATE`(37)、`_RTREE`(57/58)、`_SDI`(63/64)、`MLOG_PAGE_REORGANIZE*`(18/46/72)、`MLOG_INIT_FILE_PAGE2`(59) |
| 索引记录 DML | `MLOG_REC_INSERT*`(9/38/67)、`MLOG_REC_CLUST_DELETE_MARK*`(10/39/68)、`MLOG_REC_SEC_DELETE_MARK`(11/40)、`MLOG_REC_UPDATE_IN_PLACE*`(13/41/70)、`MLOG_REC_DELETE*`(14/42/69)、`MLOG_LIST_*`(15/16/17/43/44/45/71/75/76) |
| 压缩页 | `MLOG_ZIP_*`(48-53、73、74) |
| 通用字节写 | `MLOG_1BYTE`…`MLOG_8BYTES`(1/2/4/8)、`MLOG_WRITE_STRING`(30) |
| **非索引**（undo / 文件管理 / DDL） | 20-25、33/34/35/65、`MLOG_INDEX_LOAD`(61)、`MLOG_TABLE_DYNAMIC_META`(62) |

#### 物理逻辑（physiological）特性：先读页再重放

**redo 里存的是"cursor 偏移 + 记录尾段字节流"，不是 offsets 数组，也不是整页 diff**（`page_cur_insert_rec_write_log`，`page0cur.cc:860-1050`）：

```cpp
/* 计算 insert_rec 与 cursor_rec 第一个不同的字节 i */
...
mach_write_to_2(log_ptr, page_offset(cursor_rec));   /* 2B：cursor 在页内的偏移 */
...
log_ptr += mach_write_compressed(log_ptr, 2 * (rec_size - i));  /* 尾段长度 */
memcpy(log_ptr, ins_ptr, rec_size);                  /* 从 i 到记录尾的字节 */
```

恢复时 `page_cur_parse_insert_rec` 用 **dummy index**（`mlog_parse_index` 从 log 里重建的临时 `dict_index_t`，因为恢复时 DD 还没加载完）把这段拼回完整记录，再 `page_cur_insert_rec` 插到 cursor 之后——**是"页内操作重放"，不是字节覆盖**。

**为什么必须"先读页"**：payload 依赖页现状（cursor 内容做差分、heap/目录槽状态），所以 `recv_recover_page_func` 必须先把页读进 buffer pool 再应用。

**`MLOG_PAGE_REORGANIZE` 不是整页重写**：非压缩页 payload 为 0（`log0recv.cc:2113` 注释原文），恢复时**重新执行一遍页整理**（`btr_parse_page_reorganize` → `btr_page_reorganize_block`）——碎片压实结果由恢复时现算，不传字节镜像。

#### SMO 的列表级 redo

**为什么列表级**：分裂要搬 N 条记录到新页，逐条完整 redo 每条都要重复 "type + space + page + index 元数据"；列表级 = **一条 `MLOG_LIST_END_COPY_CREATED` 头（含 index 元数据 + 4B 总长）+ N 条 short 形式 insert**。

`page_copy_rec_list_end_to_created_page`（`page0cur.cc:2080`）关键：

```cpp
page_copy_rec_list_to_created_page_write_log(new_page, index, mtr, log_ptr); // 头：留 4B
log_mode = mtr_set_log_mode(mtr, MTR_LOG_SHORT_INSERTS);   // ← 短形式：省略 cursor 偏移
do {
  insert_rec = rec_copy(heap_top, rec, offsets);
  page_cur_insert_rec_write_log(insert_rec, rec_size, prev_rec, index, mtr);
  ...
} while (!page_rec_is_supremum(rec));
mach_write_to_4(log_ptr, log_data_len);                    // 回填 4B 总长度
```

**列表删除**（`page_delete_rec_list_write_log`）：payload 仅 **2B 记录偏移**，注释明说 "Individual deletes are not logged"——整批删除只记一个边界偏移。

**分裂的 redo 全部由子调用产生**（`btr_page_split_and_insert` 自身不直接写）：`btr_page_create` → 页创建类；`btr_attach_half_pages` → node ptr 插入（`MLOG_REC_INSERT`）+ `btr_node_ptr_set_child_page_no`（`MLOG_4BYTES`）+ 页链重接（`MLOG_2BYTES`）；`page_move_rec_list_*` → 目标页 `MLOG_LIST_END_COPY_CREATED`、源页 `MLOG_LIST_START/END_DELETE`；压缩表 → `MLOG_ZIP_PAGE_COMPRESS`（带镜像）。

**★ 恢复侧对列表级 redo 的解析**（`log/log0recv.cc:2037`，★ "列表级"的源码定义）：

```cpp
    case MLOG_LIST_END_DELETE:
    case MLOG_LIST_START_DELETE:

      ut_ad(!page || fil_page_type_is_index(page_type));   /* ★ 只能作用于索引页 */

      if (nullptr != (ptr = mlog_parse_index(ptr, end_ptr, &index))) {
        ut_a(!page || page_is_comp(page) == dict_table_is_comp(index->table));
        ptr = page_parse_delete_rec_list(type, ptr, end_ptr, block, index, mtr);
                                     /* ★ 一条 redo 删"一整段记录链表"，不是逐条 delete */
      }
      break;
    ...
    case MLOG_LIST_END_COPY_CREATED:
      ut_ad(!page || fil_page_type_is_index(page_type));
      if (nullptr != (ptr = mlog_parse_index(ptr, end_ptr, &index))) {
        ptr = page_parse_copy_rec_list_to_created_page(ptr, end_ptr, block, index, mtr);
                                     /* ★ 页分裂的另一半：整段拷进"刚创建的页"，
                                        与 MLOG_PAGE_CREATE 原子配对 */
      }
      break;
```

**★★ 一个 mtr 的多条 redo 如何成为原子单元**（`mtr/mtr0mtr.cc:782`，★ 最重要的机制）：

```cpp
  const auto n_recs = m_impl->m_n_log_recs;
  ...
  if (n_recs <= 1) {
    ut_ad(n_recs == 1);
    /* Flag the single log record as the only record in this mini-transaction. */
    *m_impl->m_log.front()->begin() |= MLOG_SINGLE_REC_FLAG;   /* ★ 128 位 OR 进首字节 */
  } else {
    /* Because this mini-transaction comprises multiple log records, append
    MLOG_MULTI_REC_END at the end. */
    mlog_catenate_ulint(&m_impl->m_log, MLOG_MULTI_REC_END, MLOG_1BYTE);  /* ★ 1B 哨兵 31 */
    ++len;
  }
```
> ★★ 单条 → 首字节 OR 上 `MLOG_SINGLE_REC_FLAG(128)`，恢复时看到该位就知道"这一条即一整个 mtr"；多条 → 末尾追加 1 字节 `MLOG_MULTI_REC_END(31)` 作结束哨兵。恢复时以它为界，保证 SMO 的 `PAGE_CREATE + LIST_END_COPY_CREATED + 指针更新` **要么全做要么全不做**。
> ⚠️ **真名核查**：`MLOG_SINGLE_REC` 不是记录类型，是标志位；`MLOG_MULTI_REC` **不存在**（只有 `_END`）。

**★ `mtr_log_t` 与 `MTR_LOG_NO_REDO`**（`include/mtr0types.h:41`）：

```cpp
/** Logging modes for a mini-transaction */
enum mtr_log_t {
  /** Default mode: log all operations modifying disk-based data */
  MTR_LOG_ALL = 0,
  /** Log no operations and dirty pages are not added to the flush list */
  MTR_LOG_NONE = 1,
  /** Don't generate REDO log but add dirty pages to flush list */
  MTR_LOG_NO_REDO = 2,          /* ★★ bulk load 用的：不产生 redo，但脏页仍进 flush list */
  /** Inserts are logged in a shorter form */
  MTR_LOG_SHORT_INSERTS = 3,    /* ★ SMO 列表级用的短形式 */
  MTR_LOG_MODE_MAX = 4
};
```
> ★ `NO_REDO` 与 `NONE` 的区别：**前者脏页仍进 flush list**（要刷盘但不记 redo），后者连 flush list 都不进。

**★ `mlog_write_initial_log_record()`：11 字节头**（`mtr/mtr0log.cc:72`）：

```cpp
void mlog_write_initial_log_record(const byte *ptr, mlog_id_t type, mtr_t *mtr) {
  ut_ad(type <= MLOG_BIGGEST_TYPE);
  ut_ad(type > MLOG_8BYTES);
  /* If no logging is requested, we may return now */
  if (!mlog_open(mtr, REDO_LOG_INITIAL_INFO_SIZE, log_ptr)) {
    return;                      /* ★★ MTR_LOG_NO_REDO / NONE 时静默丢弃整条 redo */
  }
  log_ptr = mlog_write_initial_log_record_fast(ptr, type, log_ptr, mtr);
  mlog_close(mtr, log_ptr);
}
```
```cpp
/* include/mtr0log.h:61 —— ★ 这就是"物理逻辑"的全部寻址信息 */
/* Size of initial info on REDO log
  1   byte  for LOG TYPE
  3-5 bytes for SPACE ID
  3-5 bytes for PAGE OFFSET */
constexpr size_t REDO_LOG_INITIAL_INFO_SIZE = 11;
```
```cpp
/* mtr0log.ic:169 —— "fast" 的含义：由被改地址反推页框 */
  page = (const byte *)ut_align_down(ptr, UNIV_PAGE_SIZE);
  space = mach_read_from_4(page + FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID);
  offset = mach_read_from_4(page + FIL_PAGE_OFFSET);
  ...
  /* We don't redo log the double write buffer */
  if (space == TRX_SYS_SPACE && offset >= FSP_EXTENT_SIZE &&
      offset < 3 * FSP_EXTENT_SIZE) {
    return (log_ptr);            /* ★ doublewrite buffer 不记 redo */
  }
```
> ⚠️ **"physiological" 这个词全仓 0 命中**——MySQL 从不使用它。等价语义就落在这 11 字节头里：**定位到页（space_id + page_no）+ 页内逻辑修改（offset + 内容）**，没有整页镜像。

**★★ "先读页再重放"的真正代码**（`log/log0recv.cc:1060`）：

```cpp
static void recv_apply_log_rec(recv_addr_t *recv_addr) {
  ...
  const page_size_t page_size = fil_space_get_page_size(recv_addr->space, &found);
                                     /* ★ 先拿到页尺寸：压缩页物理尺寸 != 逻辑尺寸 */
  ...
  } else if (recv_addr->state == RECV_NOT_PROCESSED) {
    mutex_exit(&recv_sys->mutex);

    if (buf_page_peek(page_id)) {    /* ★ 分支 A：已在 buffer pool → 直接用 */
      mtr_t mtr;
      mtr_start(&mtr);
      buf_block_t *block = buf_page_get(page_id, page_size, RW_X_LATCH, ..., &mtr);
      recv_recover_page(false, block);
      mtr_commit(&mtr);
    } else {
      recv_read_in_area(page_id);    /* ★★ 分支 B：真正的"先读页"，按 64 页 area 批量预读 */
    }
    mutex_enter(&recv_sys->mutex);
  }
}
```

**★ `recv_recover_page_func()`：幂等过滤靠页 LSN**（`log/log0recv.cc:2533`）：

```cpp
  recv_addr = recv_get_rec(block->page.id.space(), block->page.id.page_no());
  if (recv_addr == nullptr || recv_addr->state == RECV_BEING_PROCESSED ||
      recv_addr->state == RECV_PROCESSED) {
    mutex_exit(&recv_sys->mutex);
    return;                          /* ★ 三态 state 判定，不是某个 is_already_processed() */
  }
  ...
  /* Read the newest modification lsn from the page */
  lsn_t page_lsn = mach_read_from_8(page + FIL_PAGE_LSN);   /* ★★ 只有先读页才能拿到 */
  lsn_t page_newest_lsn = buf_page_get_newest_modification(&block->page);
  if (page_newest_lsn) page_lsn = page_newest_lsn;
  ...
    if (recv->start_lsn >= page_lsn && undo::is_active(recv_addr->space)) {
      ...
      recv_parse_or_apply_log_rec_body(recv->type, buf, buf_end,
                                       recv_addr->space, recv_addr->page_no,
                                       block, &mtr, ULINT_UNDEFINED, LSN_MAX);
                                     /* ★ block != nullptr ⇒ 真正写入刚读进来的页框 */
      end_lsn = recv->start_lsn + recv->len;
      mach_write_to_8(FIL_PAGE_LSN + page, end_lsn);
      mach_write_to_8(UNIV_PAGE_SIZE - FIL_PAGE_END_LSN_OLD_CHKSUM + page, end_lsn);
      if (page_zip) mach_write_to_8(FIL_PAGE_LSN + page_zip->data, end_lsn);
    }
```
> ★★ **幂等核心**：`recv->start_lsn >= page_lsn` 说明该修改已落盘，跳过；否则才重放。
> ⚠️ **真名核查**：`recv_page_is_already_processed()` **不存在**。真名是 `recv_page_is_brand_new()`（`log0recv.cc:2507`），语义也不同——它判定"该页全新、历史内容一律忽略"（首条 redo 是 `MLOG_INIT_FILE_PAGE(2)` 时为真）。
> ⚠️ `recv_recover_page()` 是**宏包装**（`log0recv.h:152`），`UNIV_HOTBACKUP` 下退化为一参版本。

**★ `MLOG_INIT_FILE_PAGE` 破坏"页 LSN 可信"这一前提**（`log0recv.cc:2699`）：

```cpp
    if (recv->type == MLOG_INIT_FILE_PAGE) {
      page_lsn = page_newest_lsn;   /* ★ 页被重新初始化，磁盘 FIL_PAGE_LSN 不可信 */
      memset(FIL_PAGE_LSN + page, 0, 8);
      memset(UNIV_PAGE_SIZE - FIL_PAGE_END_LSN_OLD_CHKSUM + page, 0, 8);
      if (page_zip) memset(FIL_PAGE_LSN + page_zip->data, 0, 8);
    }
```

#### bulk load：`MTR_LOG_NO_REDO` + `MLOG_INDEX_LOAD`

**装载期逐页内容不写 redo**（`Page_load::init`，`btr0load.cc:297-345`）：

```cpp
mtr->start();
if (!dict_index_is_online_ddl(m_index)) mtr->x_lock(dict_index_get_lock(m_index), ...);
mtr->set_log_mode(MTR_LOG_NO_REDO);          // ★ 记录写入不产生 redo
mtr->set_flush_observer(m_flush_observer);
if (m_page_no == FIL_NULL) {
  /* 页分配用独立 mtr 写 redo（fsp_reserve_free_extents + btr_page_alloc），
     因为"分配顺序"必须持久化，即使建新表空间也要记 */
  ...
}
```

注意：`page_create()` 本会写 `MLOG_PAGE_CREATE`，但 mtr 是 `MTR_LOG_NO_REDO`，`mlog_open()` 返回 false——**页创建 redo 在 bulk load 中被抑制**。

**装载完成后写一条 `MLOG_INDEX_LOAD`**（`ddl0builder.cc:1905-1922`，payload = 8B index id），恢复时**只校验长度、不做任何页修改、不进 hash 表**（`log0recv.cc:1643-1659`，`recv_add_to_hash_table` 里 `ut_ad(type != MLOG_INDEX_LOAD)`）——它是一条"告知"记录（主要给 MEB 备份用）。**含义：这条 redo 之后的崩溃，恢复不能靠 redo 重建该索引，必须靠 DDL log 回滚。**

---

### 回收与恢复

#### 空间回收：分批的 `fseg_free_step`

DROP INDEX 的释放链：`btr_free_if_exists` → `btr_free_but_not_root`（叶子段 + 非叶段）→ `btr_free_root` → `btr_free_root_invalidate`（`PAGE_INDEX_ID := BTR_FREED_INDEX_ID` 防二次释放）。

```cpp
// btr0btr.cc:958-1003（节选）
static void btr_free_but_not_root(buf_block_t *block, mtr_log_t log_mode) {
leaf_loop:
  mtr_start(&mtr);  mtr_set_log_mode(&mtr, log_mode);
  finished = fseg_free_step(root + PAGE_HEADER + PAGE_BTR_SEG_LEAF, true, &mtr);
  mtr_commit(&mtr);
  if (!finished) goto leaf_loop;       // 一次只释放一个 extent / 一个 frag page
top_loop:
  ... fseg_free_step_not_header(root + PAGE_HEADER + PAGE_BTR_SEG_TOP, ...) ...
}
```

`fseg_free_step`（`fsp0fsp.cc:3660-3741`）**每次调用只做一件事**：① 有 extent → `fseg_free_extent()` 返回 false；② 无 extent → 释放一个 frag page；③ 无 frag → `fsp_free_seg_inode()` 返回 true（完成）。**分批的原因**：单个 mtr 不能持有过多页锁/产生过大 redo，分批让 `fil_space_t::latch` 周期性释放。

幂等保护：`btr_free_root_check` 校验 `fil_page_index_page_check(frame) && index_id == btr_page_get_index_id(frame)`——DDL log 重放不会二次释放。

**★★ "每次只做一件事"的源码**（`fsp/fsp0fsp.cc:3655`，⚠️ **不存在 `FSEG_FREE_PAGE_LIMIT`**）：

```cpp
  descr = fseg_get_first_extent(inode, space_id, page_size, mtr);
  if (descr != nullptr) {
    /* Free the extent held by the segment */
    page = xdes_get_offset(descr);
    fseg_free_extent(inode, space_id, page_size, page, ahi, mtr);
    return false;        /* ★★ 分支 A：一次释放一整个 extent（16KiB 页 = 64 页 = 1MiB） */
  }

  /* Free a frag page */
  n = fseg_find_last_used_frag_page_slot(inode, mtr);
  if (n == ULINT_UNDEFINED) {
    /* Freeing completed: free the segment inode */
    fsp_free_seg_inode(space_id, page_size, inode, mtr);
    return true;         /* ★ 全空 → 释放 inode 本身，完成 */
  }
  fseg_free_page_low(inode, page_id_t(space_id, fseg_get_nth_frag_page_no(inode, n, mtr)),
                     page_size, ahi, mtr);
                         /* ★★ 分支 B：只释放 1 个碎片页（4B 槽位里记的那个页号） */
  ...
  return false;
```
> ★★ **每步粒度只有两种**："1 个 extent(64 页)" 或 "1 个碎片页"。返回 `false` = 未完成，调用方再起一个 mtr。
> ⚠️ **真名核查**：`fsp/fsp0fseg.cc` 这个文件**不存在**，全部段代码在 `fsp/fsp0fsp.cc`。

**★ 幂等保护：inode 已空就当"已完成"**（`fsp0fsp.cc:3680`）：

```cpp
  inode = fseg_inode_try_get(header, space_id, page_size, mtr, &iblock);
  if (inode == nullptr) {
    ib::info(ER_IB_MSG_424)
        << "Double free of inode from " << page_id_t(space_id, header_page);
    return true;         /* ★★ 崩溃重放时若已释放过，直接返回"完成" */
  }
```

**★ 分批的官方理由**（`include/fsp0fsp.h:671`，★ 注释原话）：

```cpp
/** Frees part of a segment. This function can be used to free a segment
 by repeatedly calling this function in different mini-transactions.
 Doing the freeing in a single mini-transaction might result in
 too big a mini-transaction.
 @return true if freeing completed */
```
```cpp
/* fsp0fsp.h:197 —— inode 结构注释里的另一条约束 */
Segment inode which is created for each segment in a tablespace. NOTE: in
purge we assume that a segment having only one currently used page can be
freed in a few steps, so that the freeing cannot fill the file buffer with
bufferfixed file pages.
```
> ★★ 两个理由：① 单个 mtr 太大会持有过多页锁、产生过大 redo；② **防止 buffer pool 被 bufferfixed 页塞满**（purge 路径的假设）。

**★ 两个段用不同函数**（`btr/btr0btr.cc:954`，★ 关键差异）：

```cpp
  finished = fseg_free_step(root + PAGE_HEADER + PAGE_BTR_SEG_LEAF, true, &mtr);
                                       /* ★ 叶子段：会把根页也释放掉（根页是叶子段的第一页） */
  ...
  finished = fseg_free_step_not_header(root + PAGE_HEADER + PAGE_BTR_SEG_TOP, true, &mtr);
                                       /* ★ 非叶段：保留根页 */
```
```cpp
/* fsp0fsp.cc:3788 —— _not_header 的两个关键差异 */
  n = fseg_find_last_used_frag_page_slot(inode, mtr);
  if (n == ULINT_UNDEFINED) {
    ut_error;            /* ★ 差异 1：不允许"碎片页已空"（段头页本身还在 frag 里） */
  }
  page_no = fseg_get_nth_frag_page_no(inode, n, mtr);
  if (page_no == page_get_page_no(page_align(header))) {
    return true;         /* ★★ 差异 2：剩最后一页恰好是根页时，停手 —— 根页留给 btr_free_root() */
  }
```

**★ `btr_free_root` 在同一个 mtr 里跑完**（`btr0btr.cc:765`，★ 与分批的对比）：

```cpp
static void btr_free_root(buf_block_t *block, mtr_t *mtr) {
  header = buf_block_get_frame(block) + PAGE_HEADER + PAGE_BTR_SEG_TOP;
  while (!fseg_free_step(header, true, mtr)) {
    /* Free the entire segment in small steps. */
  }
}
```
> ★ 非叶段通常就根页自己，很小，所以**不分批**；叶子段才需要"每个 mtr 一步"。
> ★ `btr_free_root_invalidate()` 把 `PAGE_INDEX_ID` 写 `BTR_FREED_INDEX_ID(=0)`，`btr_free_root_check` 再看到就判定"不再是匹配的 B-tree 页"而安全返回——**"index_id 不会被重新分配给同表空间的另一个索引"是幂等重放的基石**。

**TRUNCATE 不是"重置 root"，而是重建表**（`ha_innodb.cc:14560-14652`）：rename 表空间 → 删旧表 → 清空 `se_private_data` → `create_impl` 新建。`btr_truncate()`（原地重置 root 版本）只用于 DD 内部表，靠"复用聚簇索引恒为 0 的 `PAGE_MAX_TRX_ID` 字段"做崩溃恢复标记（`PAGE_MAX_TRX_ID := IB_ID_MAX`，`btr_truncate_recover` 检查重做）。

#### DDL log：索引树的崩溃回滚

建索引时 `log_ddl->write_free_tree_log(trx, index, /*is_drop_table=*/false)`（`log0ddl.cc:853`）：

```cpp
if (skip(index->table, trx->mysql_thd)) return DB_SUCCESS;   // 恢复中/临时表跳过
if (index->type & DICT_FTS) return DB_SUCCESS;               // FTS 无索引树
if (dict_index_get_online_status(index) != ONLINE_INDEX_COMPLETE) return DB_SUCCESS;
if (is_drop_table) { insert_free_tree_log(trx, index, id, thread_id); }
else { insert_free_tree_log(nullptr, index, id, thread_id);
       delete_by_id(trx, id, false);   // 建表 trx 提交 ⇒ 删除这条记录 }
```

**语义**：建索引成功提交 ⇒ 删掉这条 DDL log；失败/崩溃 ⇒ log 留下 ⇒ post_ddl 回放 `replay_free_tree_log` → `btr_free_if_exists`（幂等）删树。已知 FIXME（`dict0crea.cc:434-441` 注释）：`btr_create` 之后才写 DDL log，若在此之前崩溃，段/页会泄漏（官方称"rare case, acceptable"）。

**★ DDL log 有 8 种操作类型，不是 3 种**（`include/log0ddl.h:43`，⚠️ 常见说法是 3 种）：

```cpp
/** DDL log types defined as uint32_t because it costs 4 bytes in
mysql.innodb_ddl_log. */
enum class Log_Type : uint32_t {
  SMALLEST_LOG = 1,
  /** Drop an index tree */
  FREE_TREE_LOG = 1,        /* ★★ 索引树崩溃回滚用的就是它 */
  /** Delete a file */
  DELETE_SPACE_LOG,
  /** Rename a file */
  RENAME_SPACE_LOG,
  /** Drop the entry in innodb_table_metadata */
  DROP_LOG,
  /** Rename table in dict cache. */
  RENAME_TABLE_LOG,
  /** Remove a table from dict cache */
  REMOVE_CACHE_LOG,
  /** Alter Encrypt a tablespace */
  ALTER_ENCRYPT_TABLESPACE_LOG,
  /** Alter Unencrypt a tablespace */
  ALTER_UNENCRYPT_TABLESPACE_LOG,
  BIGGEST_LOG = ALTER_UNENCRYPT_TABLESPACE_LOG
};
```

**★ `DDL_Record` 落库的是"根页三元组"**（`include/log0ddl.h:77` / `log/log0ddl.cc:920`）：

```cpp
  DDL_Record record;
  record.set_id(id);
  record.set_thread_id(thread_id);
  record.set_type(Log_Type::FREE_TREE_LOG);
  record.set_space_id(index->space);      /* ★ 表空间 */
  record.set_page_no(index->page);        /* ★★ 根页号 */
  record.set_index_id(index->id);         /* ★ 索引 id（btr_free_if_exists 会校验） */
```
```cpp
/* log0ddl.cc:920 的断言 */
  ut_ad(index->page != FIL_NULL);         /* ★ 必须有根页 */
```
> ★ 这张表就叫 **`mysql.innodb_ddl_log`**（InnoDB 引擎的普通表，所以写它本身是 redo 保护的）。

**★ 完整实现与"先 insert 再 delete"的技巧**（`log/log0ddl.cc:853`）：

```cpp
dberr_t Log_DDL::write_free_tree_log(trx_t *trx, const dict_index_t *index,
                                     bool is_drop_table) {
  if (skip(index->table, trx->mysql_thd)) return DB_SUCCESS;
  if (index->type & DICT_FTS) {
    ut_ad(index->page == FIL_NULL);       /* ★ FTS 索引没有 B-tree 根页 */
    return DB_SUCCESS;
  }
  if (dict_index_get_online_status(index) != ONLINE_INDEX_COMPLETE) {
    /* To skip any previously aborted index. This is because this kind
    of index should be already freed in previous post_ddl. It's improper
    to log it and may free it again later, which may trigger some
    double free page problem. */
    return DB_SUCCESS;                    /* ★★ 与 online_status 的直接耦合 */
  }
  ...
  if (is_drop_table) {
    /* Drop index case, if committed, will be redo only */
    err = insert_free_tree_log(trx, index, id, thread_id);   /* 传 trx：同生共死 */
  } else {
    /* This is the case of building index during create table scenario. ... */
    err = insert_free_tree_log(nullptr, index, id, thread_id);  /* 独立后台 trx 立刻落库 */
    ...
    /* Delete this operation if the create trx is committed */
    err = delete_by_id(trx, id, false);   /* ★★ 用同一个 trx 把记录删掉 */
  }
```
> ★★ **技巧**：先 `insert` 一条 FREE_TREE_LOG，再用**同一个 trx** 把它 `delete` 掉。若 trx 提交 ⇒ 记录消失 ⇒ 不回滚；若崩溃 ⇒ delete 未提交 ⇒ 记录留存 ⇒ 重启后 post_ddl 回放把树删掉。**这是"用事务的原子性反过来做 DDL 的撤销日志"**。

**★ 回放：两条线在这里合流**（`log/log0ddl.cc:1625`）：

```cpp
void Log_DDL::replay_free_tree_log(space_id_t space_id, page_no_t page_no,
                                   ulint index_id) {
  ...
  /* This is required by dropping hash index afterwards. */
  dict_sys_mutex_enter();                 /* ★ 必须持 dict_sys mutex */
  mtr_t mtr;
  mtr_start(&mtr);
  btr_free_if_exists(page_id_t(space_id, page_no), page_size, index_id, &mtr);
  mtr_commit(&mtr);
  dict_sys_mutex_exit();
  DBUG_INJECT_CRASH("ddl_log_crash_after_replay", crash_after_replay_counter++);
}
```
> ★★ **DDL log 重放（第 2 条线）实际执行的就是 `btr_free_if_exists()`（第 1 条线的 redo 分批释放）**。`DBUG_INJECT_CRASH` 专门在"重放后"埋崩溃点，验证这条链路的幂等性。

**★ 何时不写 DDL log**（`log/log0ddl.cc:847`）：

```cpp
inline bool Log_DDL::skip(const dict_table_t *table, THD *thd) {
  return (recv_recovery_on || thread_local_ddl_log_replay ||
          (table != nullptr && table->is_temporary()) ||
          thd_is_bootstrap_thread(thd));
}
```
> ★ **恢复期间不写**（因为正在做的就是在重放 DDL log）；**临时表不写**（临时表空间随进程消失，不需要崩溃恢复）。

**★ 写入点：共享表空间时才需要逐索引记**（`row/row0mysql.cc:4026`）：

```cpp
  if (!table->is_temporary() && !file_per_table) {
    for (dict_index_t *index = table->first_index();
         err == DB_SUCCESS && index != nullptr; index = index->next()) {
      err = log_ddl->write_free_tree_log(trx, index, true);
    }
    ...
  }
```
> ★ file-per-table 时删表直接删文件，**不需要**逐索引记 FREE_TREE_LOG。

#### 恢复视角：hash 聚合与"页是全新的"

**Scan 阶段**（`recv_parse_log_rec`）：取出 `(type, space_id, page_no)`，body 解析时 `block == nullptr` → 所有 `page_*_parse` 走 `if (!block) return rec_end;` 分支（**只校验长度不修改**）；`recv_add_to_hash_table` 按 `(space, page_no)` 聚合进 `rec_list`。

**Apply 阶段**（`recv_apply_hashed_log_recs` → `recv_recover_page_func`，`log0recv.cc:2540`）：

- **LSN 过滤**：只应用 `start_lsn > 页上 FIL_PAGE_LSN`（已落盘的跳过）；
- **先读页**：`buf_page_get` 读进 buffer pool 再重放（physiological 特性使然）；
- **"页是全新的"**（`recv_page_is_brand_new`）：rec_list 第一条是 `MLOG_INIT_FILE_PAGE2` ⇒ **忽略该页磁盘上的旧内容**——新分配页由 `fsp_init_file_page` 盖章，stale 页不会被误用；
- **表空间没了**（`recv_sys->missing_ids`）：该页 redo 直接丢弃（表被 DROP 后的 redo）；
- **页类型不匹配**：`MLOG_PAGE_CREATE*` 显式 "Allow anything in page_type when creating a page"（建页可覆盖任意旧类型）；其余断言失败置 `found_corrupt_log`；
- **恢复中不校验 B 树结构**：`btr_validate_index` 只在 CHECK TABLE / IMPORT / build 完成后调用（`ha_innodb.cc:18229`），恢复路径只做 redo 记录级健全性检查。

**恢复时不产生新 redo**：`mtr_set_log_mode(&mtr, MTR_LOG_NONE)`（`log0recv.cc:2620`）。

**★ 聚合的两级 hash 结构**（`log0recv.h:379`，★ space → page 两级，不是一张大表）：

```cpp
struct recv_sys_t {
  using Pages = std::unordered_map<page_no_t, recv_addr_t *, ...>;  /* ★ 一级：space 内按 page_no */
  struct Space {
    mem_heap_t *m_heap;     /* 该表空间的 redo 记录统一在此 heap 分配 */
    Pages m_pages;
  };
  using Spaces = std::unordered_map<space_id_t, Space, ...>;        /* ★ 二级：按 space_id */
```
```cpp
/* log0recv.h:255 —— 单页的 redo 链表 */
struct recv_addr_t {
  recv_addr_state state;
  space_id_t space;
  page_no_t page_no;        /* ★ 聚合键就是 (space, page_no) */
  List rec_list;            /* ★ UT_LIST，追加到尾部 → 天然按扫描(=LSN)顺序 */
};
```

**★ 聚合代码**（`log0recv.cc:2401`）：

```cpp
static void recv_add_to_hash_table(mlog_id_t type, space_id_t space_id,
                                   page_no_t page_no, byte *body, byte *rec_end,
                                   lsn_t start_lsn, lsn_t end_lsn) {
  ut_ad(type != MLOG_FILE_DELETE);
  ut_ad(type != MLOG_FILE_CREATE);
  ut_ad(type != MLOG_FILE_RENAME);
  ut_ad(type != MLOG_FILE_EXTEND);
  ut_ad(type != MLOG_DUMMY_RECORD);
  ut_ad(type != MLOG_INDEX_LOAD);       /* ★★ 这 6 类不进 hash：扫描阶段直接处理掉 */
  ...
  auto it = space->m_pages.find(page_no);
  if (it != space->m_pages.end()) {
    recv_addr = it->second;
  } else {
    recv_addr = ...; recv_addr->state = RECV_NOT_PROCESSED;
    UT_LIST_INIT(recv_addr->rec_list);
    space->m_pages.insert(it, Value{page_no, recv_addr});
    ++recv_sys->n_addrs;
  }
  UT_LIST_ADD_LAST(recv_addr->rec_list, recv);   /* ★ 追加到该页 redo 链表尾 */
  /* body 按 RECV_DATA_BLOCK_SIZE 分块存进 heap（不超过 UNIV_PAGE_SIZE） */
```

**★★★ "页是全新的"判定完整源码**（`log0recv.cc:2507`，真名 `recv_page_is_brand_new`）：

```cpp
bool recv_page_is_brand_new(buf_block_t *block) {
  mutex_enter(&recv_sys->mutex);
  recv_addr_t *recv_addr;
  recv_addr = recv_get_rec(block->page.id.space(), block->page.id.page_no());
  if (recv_addr == nullptr) {
    /* no redo log treated as brand new */
    mutex_exit(&recv_sys->mutex);
    return true;                 /* ① 该页没有任何 redo → 当它是全新的 */
  }
  auto recv = UT_LIST_GET_FIRST(recv_addr->rec_list);
  if (recv == nullptr) { ... return true; }
  if (recv->type == MLOG_INIT_FILE_PAGE2 || recv->type == MLOG_INIT_FILE_PAGE) {
    mutex_exit(&recv_sys->mutex);
    return true;                 /* ② 第一条 redo 是"页被启用" → 不依赖旧镜像 */
  }
  mutex_exit(&recv_sys->mutex);
  return false;
}
```

> ★★ **判定看的是"该页第一条 redo 的类型"，不是页头 LSN**。这解释了为什么 bulk load 豁免 redo 期间分配的页在恢复时不会被误用——它们根本没有 redo（走分支①）。

**★ 幂等核心：LSN 过滤**（`log0recv.cc:2699`）：

```cpp
    if (recv->start_lsn >= page_lsn       /* ★ 只重放 LSN 大于页已有 LSN 的记录 */
#ifndef UNIV_HOTBACKUP
        && undo::is_active(recv_addr->space)
#endif
    ) {
```
> ★ `page_lsn` 取 `FIL_PAGE_LSN` 与 buffer pool 里 `buf_page_get_newest_modification()` 的较大者。

---

### 特殊场景

#### 压缩页 / 加密 / punch hole

- **压缩页**（`ROW_FORMAT=COMPRESSED`）：压缩是页级附加表示（`page_zip` + modification log），B-tree 逻辑不变。机制三层：
  - **modification log 延迟压缩**：`page_zip_des_t`（`data`/`m_end`/`n_blobs`/`m_nonempty`/`ssize`）在压缩页 trailer 的未压缩区维护修改日志——索引修改（`page_zip_write_rec`、`page_zip_write_node_ptr`）先追加进 mod log 并置 `m_nonempty=true`，**不立即重压缩**；只有 `page_zip_available()` 判定空间不足时 B-tree 层才触发 `btr_page_reorganize → page_zip_reorganize → page_zip_compress` 全页 zlib 重压缩（成功后 `m_nonempty=false`，失败 `dict_index_zip_failure` 回滚页面）；
  - **`zip_pad` 自适应填充**：`zip_pad_info_t{pad, success, failure, n_rounds}`——每 128 次压缩一轮（`ZIP_PAD_ROUND_LEN`），失败率超 `innodb_compression_failure_threshold_pct` 则 `pad += 128`，连续 5 轮达标（`ZIP_PAD_SUCCESSFUL_ROUND_LIMIT`）则 `pad -= 128`；`dict_index_zip_pad_optimal_page_size` 把"页面可用空间上限"压到 `UNIV_PAGE_SIZE - pad`——**让页面在还留有 pad 空间时就分裂/重组**，保证新半页一定能压缩成功（压缩失败只能靠分裂回退、代价远高于提前预留）；
  - **KEY_BLOCK_SIZE 链**：`get_zip_shift_size`（KBS→log2 shift）→ `create_table_info_t::check_create_options`（合法值 1/2/4/8/16，临时表禁用，要求 `innodb_file_per_table`）→ **`dict_table_t::flags` 的 4-bit `ZIP_SSIZE` 字段**（`DICT_TF_GET_ZIP_SSIZE`；**8.0 没有 `dict_tf_t`/`zip_ssize` 成员**）；无 KBS 的 `ROW_FORMAT=COMPRESSED` 默认 `zip_ssize_max - 1`。
  - **压缩页分裂**：搬记录**不能用** `page_move_rec_list_*`（新页重压缩可能失败），改走 `page_zip_copy_recs` 整页字节复制 + `page_delete_rec_list_*`（删除必成功）；插入失败 `n_iterations++` 回 `func_start` 重分裂，二次失败且新页为压缩页时改插空页避免无限分裂。
  - **redo 差异**：常见路径成功后补一条 `MLOG_ZIP_PAGE_COMPRESS_NO_DATA`（"请重压缩"，恢复时**不重组织、等这条再重压缩**，期间压缩/非压缩短暂不一致是允许的）；压缩失败/整页拷贝才写带镜像的 `MLOG_ZIP_PAGE_COMPRESS`；小改走 `MLOG_ZIP_WRITE_NODE_PTR`（payload = 解压页偏移 2B + 压缩页偏移 2B + 页号 4B）/`MLOG_ZIP_WRITE_BLOB_PTR`/`MLOG_ZIP_WRITE_HEADER`；`MLOG_ZIP_PAGE_REORGANIZE` 由 `btr_page_reorganize_low` 在**重压缩成功后**记录（1B z_level），恢复端 `btr_parse_page_reorganize(compressed=true)` 读 level 后 `btr_page_reorganize_block(true, level, ...)` 重压缩；`page_zip_validate`（UNIV_ZIP_DEBUG）在重组/分裂前后重新解压比对。
- **加密**：完全在 fil/os 层（`fil_io_set_encryption` → `os_aio`），**索引/B-tree 层无感知**；顺序是**先压缩后加密**（注释：先加密会导致压缩失败）；恢复时 page 0 先解密取密钥。
- **punch hole**：fil/os 层的逐页写优化（写前打洞让全零区变 sparse），与索引段**没有直接关系**；DROP INDEX 释放页**不会主动 punch hole**，`.ibd` 物理大小不减——真正的空间回收只在 file-per-table 的 DROP/TRUNCATE（整文件删除）。

**★★★ 但 punch hole 与索引页类型**有关系**（`os0file.cc:1345`，★ 一条明确的排除规则）：

```cpp
  /* The page must be at least twice as large as the file system
  block size if we are to save any space. Ignore R-Tree pages for now,
  they repurpose the same 8 bytes in the page header. No point in
  compressing if the file system block size >= our page size. */   /* ★★ 源码原话 */

  if (page_type == FIL_PAGE_RTREE || block_size == ULINT_UNDEFINED ||
      compression.m_type == Compression::NONE || src_len < block_size * 2) {
    *dst_len = src_len;
    return (src);          /* ★ 原样返回：不压缩、不打洞 */
  }
```

> ★★★ **R-Tree 页被显式排除在透明页压缩之外**——因为它们复用了 FIL 头那 8 字节作 SSN（`FIL_RTREE_SPLIT_SEQ_NUM`）。这与 rtree.md 里"R-Tree 页不参与 punch-hole 压缩"是同一件事的两面。

**★ 第二个关闭条件**（`fil0fil.cc:2415`，★ 整表空间级）：

```cpp
  if (!IORequest::is_punch_hole_supported() || !punch_hole ||
      file.block_size >= srv_page_size) {
    fil_no_punch_hole(&file);     /* ★ 文件系统块 ≥ 页大小时整体关闭 */
  } else {
    file.punch_hole = punch_hole;
  }
```

**★ `IORequest` 的标志位**（`include/os0file.h:267`，⚠️ **不存在 `PUNCH_HOLE_PORTION`**）：

```cpp
class IORequest {
 public:
  /** Flags passed in the request, they can be ORred together. */
  enum {
    UNSET = 0, READ = 1, WRITE = 2,
    /** Request for a doublewrite page IO */
    DBLWR = 4,
    /** Data file */
    DATA_FILE = 8,
    /** Log file request*/
    LOG = 16,
    /** Disable partial read warnings */
    DISABLE_PARTIAL_IO_WARNINGS = 32,
    /** Do not to wake i/o-handler threads, but the caller will do ... */
    DO_NOT_WAKE = 64,
    /** Ignore failed reads of non-existent pages */
    IGNORE_MISSING = 128,
    /** Use punch hole if available, only makes sense if
    compression algorithm != NONE. Ignored if not set */
    PUNCH_HOLE = 256,        /* ★★ 真名就叫 PUNCH_HOLE */
    /** Force raw read, do not try to compress/decompress.
    This can be used to force a read and write without any
    compression e.g., for redo log, merge sort temporary files
    and the truncate redo log. */
    NO_COMPRESSION = 512,    /* ★ redo log / 临时排序文件 / truncate redo 强制裸读写 */
    /** Row log used in online DDL */
    ROW_LOG = 1024,
    /** We optimise cases where punch hole is not done if the compressed length
    of the page is the same as the original size of the page. Ignore such
    optimisations if this flag is set. */
    DISABLE_PUNCH_HOLE_OPTIMISATION = 2048
  };
```
```cpp
  /** @return true if punch hole is supported */
  static bool is_punch_hole_supported() {
    DBUG_EXECUTE_IF("ignore_punch_hole", return (true););
#if defined(HAVE_FALLOC_PUNCH_HOLE_AND_KEEP_SIZE) || defined(_WIN32)
    return (true);
#else
    return (false);
#endif
  }
```

**★ punch hole 的 Linux 实现**（`os/os0file.cc:2056`，★ `KEEP_SIZE` 是关键）：

```cpp
static dberr_t os_file_punch_hole_posix(os_file_t fh, os_offset_t off,
                                        os_offset_t len) {
#ifdef HAVE_FALLOC_PUNCH_HOLE_AND_KEEP_SIZE
  const int mode = FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE;
                             /* ★★ KEEP_SIZE：打洞但文件大小不变
                                —— 这是"透明页压缩"能让 16k 页只占 <16k 物理块的核心 */
  int ret = fallocate(fh, mode, off, len);
  if (ret == 0) return (DB_SUCCESS);
  ...
  if (errno == ENOTSUP) {
    return (DB_IO_NO_PUNCH_HOLE);   /* ★ 文件系统不支持 → 上层关闭 punch hole 回退为普通写 */
  }
```
```cpp
/* os0file.cc:1696 —— 写盘流程：先写压缩后数据，再把尾部打洞 */
    offset += len;
    return (os_file_punch_hole(fh, offset, src_len - len));
                             /* ★ 文件逻辑大小仍是 src_len，物理占用只剩 len */
```

**★★ 表压缩 vs 页压缩：两套完全独立的机制**（★ 最易混淆的一点）：

| | 表压缩 | 页压缩（透明压缩） |
|---|---|---|
| 语法 | `ROW_FORMAT=COMPRESSED` + `KEY_BLOCK_SIZE` | `COMPRESSION=zlib` / `lz4` |
| 载体 | `page_zip_des_t`（BP 内 `buf_page_t::zip`） | `Compression::Type`（`fil_space_t::compression_type`） |
| 判定 | `DICT_TF_GET_ZIP_SSIZE(flags)` | `fil_space_t::is_compressed()` |
| 尺寸 | 物理尺寸 = `(1K/2) << zip_ssize` | 逻辑尺寸不变，靠 punch hole 省物理块 |
| 标志 | `DICT_TF_ZIP_SSIZE` 位段 | `IORequest::PUNCH_HOLE` |

```cpp
/* include/dict0dict.ic:449 —— 表压缩的判定 */
  if (DICT_TF_GET_ZIP_SSIZE(flags)) {
    return (REC_FORMAT_COMPRESSED);      /* ★ 由 SYS_TABLES.TYPE 的 ZIP_SSIZE 位段决定 */
  }
```
```cpp
/* os/file.h:50 —— 页压缩的算法枚举（★ 不在 os0file.h，在 os/file.h） */
struct Compression {
  enum Type : uint8_t {
    /* Note: During recovery we don't have the compression type
    because the .frm file has not been read yet. Therefore
    we write the recovered pages out without compression. */  /* ★★ 恢复时一律不压缩写出 */
    NONE = 0, ZLIB = 1, LZ4 = 2
  };
  struct meta_t {
    uint8_t m_version;
    Type m_algorithm;
    uint16_t m_original_type;
    uint16_t m_original_size;
    uint16_t m_compressed_size;    /* ★ 解压靠它还原 */
  };
```

**★ 加密字段在 `fil_space_t` 里**（`include/fil0fil.h:504`，⚠️ **不存在 `encryption_type`**）：

```cpp
  /** Compression algorithm */
  Compression::Type compression_type;      /* ★ 页压缩，表空间级 */
  /** Encryption metadata */
  Encryption_metadata m_encryption_metadata;  /* ★ master key id + 算法 + 表空间密钥 */
  /** Encryption is in progress */
  Encryption::Progress encryption_op_in_progress{Encryption::Progress::NONE};
  ...
  [[nodiscard]] bool is_compressed() const noexcept {
    return compression_type != Compression::NONE;
  }
  [[nodiscard]] bool is_encrypted() const noexcept {
    return FSP_FLAGS_GET_ENCRYPTION(flags);   /* ★★ 是否加密是 FSP header 的 flag 位 */
  }
```
> ⚠️ punch hole 能力在 **`fil_node_t`**（`fil0fil.h:219`）而不是 `fil_space_t`：`bool punch_hole;` + `size_t block_size;`——**每个文件节点单独探测一次文件系统能力**。

**★ 为什么 `innodb_page_size > 16k` 时禁止 COMPRESSED**（`include/univ.i:300`，★ 源码级原因）：

```cpp
/** log2 of largest compressed page size (1<<14 == 16384 bytes).
A compressed page directory entry reserves 14 bits for the start offset
and 2 bits for flags. This limits the uncompressed page size to 16k. ... */
constexpr uint32_t UNIV_ZIP_SIZE_SHIFT_MAX = 14;
constexpr uint32_t UNIV_ZIP_SIZE_MIN = 1 << UNIV_ZIP_SIZE_SHIFT_MIN;
constexpr uint32_t UNIV_ZIP_SIZE_MAX = 1 << UNIV_ZIP_SIZE_SHIFT_MAX;
```
> ★★ **压缩页目录项只有 14 bit 存偏移** —— 这是硬限制，与性能无关。
> ⚠️ **不存在** `PAGE_ZIP_MIN_SIZE` 这个名字。

**★ 相关页类型**（`fil0fil.h:1205`）：

```cpp
constexpr page_type_t FIL_PAGE_INDEX = 17855;      /* B-tree 索引页 */
constexpr page_type_t FIL_PAGE_RTREE = 17854;      /* R-tree 索引页 */
constexpr page_type_t FIL_PAGE_COMPRESSED = 14;
constexpr page_type_t FIL_PAGE_ENCRYPTED = 15;
constexpr page_type_t FIL_PAGE_COMPRESSED_AND_ENCRYPTED = 16;
constexpr page_type_t FIL_PAGE_ENCRYPTED_RTREE = 17;
constexpr page_type_t FIL_PAGE_SDI_BLOB = 18;
```

> ⚠️ 真名：`os_file_punch_hole()`（`os0file.h:1764`）——**不存在 `buf_page_free_punch_hole`**。
> ★ 压缩后没变小就干脆不打洞：`DISABLE_PUNCH_HOLE_OPTIMISATION = 2048` 标志。

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `FSP_FIRST_INODE_PAGE_NO` | 首个 INODE 页（root page no 的下界，`dd_write_index` 断言） |
| `BTR_FREED_INDEX_ID` = 0 | 释放后 `PAGE_INDEX_ID` 的哨兵值 |
| `BTR_MAX_NODE_LEVEL` = 45 / `BTR_MAX_LEVELS` = 100 | 树高上限 / 路径数组容量 |
| `FSEG_HEADER_SIZE` = 10 | 段头长度（SPACE 4 + PAGE_NO 4 + OFFSET 2） |
| `index->merge_threshold` = 50 | 页合并阈值（与空间回收的触发相关） |
| `FIL_PAGE_INDEX` = 17855 | 索引页类型 |

**★ 可调的系统变量**（逐个核实）：

| 变量 | 文件:行 | 默认 | 备注 |
|---|---|---|---|
| `innodb_page_size` | `ha_innodb.cc:22574` | 16 KiB | `READONLY` + `NOPERSIST`，`UNIV_PAGE_SIZE_DEF = 1 << UNIV_PAGE_SIZE_SHIFT_DEF` |
| `innodb_fill_factor` | `ha_innodb.cc:22477` | **100** | 范围 [10,100]；"Percentage of B-tree page filled during **bulk insert**"——★ 只影响 bulk load，不影响普通插入的页填充 |
| `innodb_compression_level` | `ha_innodb.cc:22252` | **6** | 范围 [0,9]，`DEFAULT_COMPRESSION_LEVEL = 6`（`page0zip.h:59`） |
| `innodb_page_cleaners` | `ha_innodb.cc:22135` | **4** | READONLY，范围 [1,64]；会被引擎侧钳到 `srv_buf_pool_instances`（`ha_innodb.cc:5084`） |
| `innodb_merge_threshold_set_all_debug` | `ha_innodb.cc:22053` | 50 | ⚠️ **仅 debug 构建**（`UNIV_DEBUG`），范围 [1,50]，可批量覆盖所有索引的 `merge_threshold` |

```cpp
/* ha_innodb.cc:22477 */
static MYSQL_SYSVAR_LONG(fill_factor, ddl::fill_factor, PLUGIN_VAR_RQCMDARG,
                         "Percentage of B-tree page filled during bulk insert",
                         nullptr, nullptr, 100, 10, 100, 0);
```
> ★ 引擎侧声明：`storage/innobase/include/ddl0ddl.h:58` 的 `extern long fill_factor;`（命名空间 `ddl`）。

> ⚠️ **`innodb_fill_factor` 确实存在**（不是"无效变量"）——但它**只作用于 bulk load / DDL 建索引路径**，普通 INSERT 的页分裂不看它。这是最容易误解的一点。

**★ 补齐其余变量（逐个核实注册原文）**：

```cpp
static MYSQL_SYSVAR_ULONG(page_size, srv_page_size,
                          PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY |
                              PLUGIN_VAR_NOPERSIST,
                          "Page size to use for all InnoDB tablespaces.",
                          nullptr, nullptr, UNIV_PAGE_SIZE_DEF,
                          UNIV_PAGE_SIZE_MIN, UNIV_PAGE_SIZE_MAX, 0);
```
```cpp
static MYSQL_SYSVAR_BOOL(
    file_per_table, srv_file_per_table, PLUGIN_VAR_NOCMDARG,
    "Stores each InnoDB table to an .ibd file in the database dir.", nullptr,
    nullptr, true);
```
```cpp
static MYSQL_SYSVAR_UINT(
    compression_level, page_zip_level, PLUGIN_VAR_RQCMDARG,
    "Compression level used for compressed row format.  0 is no compression"
    ", 1 is fastest, 9 is best compression and default is 6.",
    nullptr, nullptr, DEFAULT_COMPRESSION_LEVEL, 0, 9, 0);

static MYSQL_SYSVAR_BOOL(
    log_compressed_pages, page_zip_log_pages, PLUGIN_VAR_OPCMDARG,
    "Enables/disables the logging of entire compressed page images."
    " InnoDB logs the compressed pages to prevent corruption if"
    " the zlib compression algorithm changes."
    " When turned OFF, InnoDB will assume that the zlib"
    " compression algorithm doesn't change.",
    nullptr, nullptr, true);
                          /* ★★ 这条解释了为什么默认 on：防止 zlib 版本差异导致损坏 */
```
```cpp
static MYSQL_SYSVAR_ENUM(
    default_row_format, innodb_default_row_format, PLUGIN_VAR_RQCMDARG,
    "The default ROW FORMAT for all innodb tables created without explicit"
    " ROW_FORMAT. Possible values are REDUNDANT, COMPACT, and DYNAMIC."
    " The ROW_FORMAT value COMPRESSED is not allowed",
    nullptr, nullptr, DEFAULT_ROW_FORMAT_DYNAMIC,
    &innodb_default_row_format_typelib);
```
```cpp
/** Note we cannot use rec_format_enum because we do not allow
COMPRESSED row format for innodb_default_row_format option. */  /* ★★ 源码注释直说 */
enum default_row_format_enum {
  DEFAULT_ROW_FORMAT_REDUNDANT = 0,
  DEFAULT_ROW_FORMAT_COMPACT = 1,
  DEFAULT_ROW_FORMAT_DYNAMIC = 2,      /* ★ 故意不含 COMPRESSED */
};
```
```cpp
static MYSQL_SYSVAR_ULONG(
    compression_failure_threshold_pct, zip_failure_threshold_pct,
    PLUGIN_VAR_OPCMDARG,
    "If the compression failure rate of a table is greater than this number"
    " more padding is added to the pages to reduce the failures. A value of"
    " zero implies no padding",
    nullptr, nullptr, 5, 0, 100, 0);

static MYSQL_SYSVAR_ULONG(
    compression_pad_pct_max, zip_pad_max, PLUGIN_VAR_OPCMDARG,
    "Percentage of empty space on a data page that can be reserved"
    " to make the page compressible.",
    nullptr, nullptr, 50, 0, 75, 0);
```
```cpp
static MYSQL_SYSVAR_ULONG(undo_tablespaces, srv_undo_tablespaces,
                          PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_NOPERSIST,
                          "Number of undo tablespaces to use. (deprecated)",
                          nullptr, innodb_undo_tablespaces_update,
                          FSP_IMPLICIT_UNDO_TABLESPACES,
                          FSP_MIN_UNDO_TABLESPACES, FSP_MAX_UNDO_TABLESPACES, 0);
```

**★★★ 三个"已删除"的变量（8.0 全仓 0 命中）**：

| 变量 | 状态 |
|---|---|
| `innodb_file_format`（Antelope/Barracuda） | **8.0.39 已彻底删除**，全仓 grep 0 命中 |
| `innodb_large_prefix` | **不存在**，全仓 grep 0 命中 |
| `innodb_encrypt_tables` | **不存在** → 改为 `innodb_redo_log_encrypt` / `innodb_undo_log_encrypt` / `ENCRYPTION` 子句 |

**★ 状态变量**（★ 与"不存在"的对照）：

| 状态变量 | 文件:行 | 取值来源 |
|---|---|---|
| `Innodb_page_size` | `ha_innodb.cc:1247` + `srv0srv.cc:1654` | `UNIV_PAGE_SIZE` |
| `Innodb_buffer_pool_pages_data` | `ha_innodb.cc:1142` + `srv0srv.cc:1631` | `LRU_len` |
| `Innodb_pages_created` | `include/srv0srv.h:1184` | `buf_pool->stat.n_pages_created` |
| `Innodb_pages_read` | `include/srv0srv.h:1185` | `buf_pool->stat.n_pages_read` |
| ~~`Innodb_num_indexes`~~ | **不存在** | 8.0 改由 `information_schema.INNODB_INDEXES` 提供 |
| ~~`Innodb_page_compressed_*`~~ | **不存在** | 透明页压缩无 status 计数器 |

```cpp
    {"page_size", (char *)&export_vars.innodb_page_size, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},      /* ha_innodb.cc:1247 —— 暴露点 */
```
```cpp
  export_vars.innodb_page_size = UNIV_PAGE_SIZE;   /* srv0srv.cc:1654 —— 赋值点 */
```

### ★★★ 真名勘误全表（本轮新增）

| 常见说法 | 8.0.39 真相 |
|---|---|
| `dict_index_t::page` 是 `page_no_t` / `uint32` | **都不是** → 位域 `unsigned page : 32`（`space` 同理） |
| 根页号由 `dict_index_get_root()` 获取 | **不存在该函数**（全仓 0 命中）→ `dict_index_get_page(index)` + `btr_root_block_get()` |
| `dict_index_t` 里有 `tree`（`btr_btr_t`） | **已在 5.7 移除**；`lock` 存在但是内嵌值 `rw_lock_t lock` |
| 树高缓存在 `dict_index_t` 里 | **完全没有**（grep `height` 0 命中） |
| `btr_height_get` 会沿树逐级递减 | **不会** → 只读根页 1 页，1 次 IO，与树高无关 |
| `FSEG_HEADER_SIZE` 是 16 字节 | **10 字节**（`FSEG_HDR_SPACE 4 + PAGE_NO 4 + OFFSET 2`） |
| 有 `fseg_header_put()` / `fseg_header_get()` | **都不存在** → 三次 `mlog_write_ulint` 直接写 |
| `fseg_free_step` 每步释放固定页数（`FSEG_FREE_PAGE_LIMIT`） | **该常量不存在** → 粒度只有"1 个 extent(64页)"或"1 个碎片页" |
| 段代码在 `fsp/fsp0fseg.cc` | **该文件不存在** → 全在 `fsp/fsp0fsp.cc` |
| `PAGE_HEADER` 是一组 `#define`，在 `page0page.h` | 8.0 全部改成 `constexpr uint32_t`，**在 `page0types.h`** |
| `page_header_set_level()` | **不存在** → `btr_page_set_level()`（`btr0btr.ic`） |
| `PAGE_LEVEL` 根页是 0 | 叶子=0，**根页 level = 树高-1** |
| 段 == 区 | **三级**：段(`fseg_inode_t`,逻辑) → 区(64页/1MiB,`xdes_t`) → 页 |
| `physiological` 是源码用词 | **全仓 0 命中**，MySQL 从不使用 |
| `MLOG_SINGLE_REC` 是记录类型 | **不是** → `MLOG_SINGLE_REC_FLAG = 128`，是打在首字节的标志位 |
| `MLOG_MULTI_REC` / `MLOG_MULTI_REC_END` | 只有 `MLOG_MULTI_REC_END = 31`，没有 `MLOG_MULTI_REC` |
| `MLOG_*` 常量在 `mtr0log.h` | **一个都没有** → 全在 `include/mtr0types.h` 的 `enum mlog_id_t` |
| `MLOG_LIST_END_DELETE` 是老编号 | 新值是 **75**（`MLOG_LIST_START_DELETE = 76`）；老编号改名 `*_8027 = 15/16` |
| `recv_recover_page()` 是独立函数 | **是宏包装**（`log0recv.h:152`） |
| 有 `recv_page_is_already_processed()` | **不存在** → `recv_page_is_brand_new()`，且语义不同 |
| `innodb_file_format`（Antelope/Barracuda）还在 | **8.0 已彻底删除** |
| `innodb_large_prefix` 还在 | **不存在** |
| `innodb_encrypt_tables` 还在 | **不存在** → `innodb_redo_log_encrypt` / `innodb_undo_log_encrypt` |
| 有 `Innodb_num_indexes` 状态变量 | **不存在** → `information_schema.INNODB_INDEXES` |
| 有 `Innodb_page_compressed_*` 状态变量 | **不存在** |
| `ROW_FORMAT=COMPRESSED` 已废弃 | 未废弃但被**明确排斥**：`innodb_default_row_format` 的 enum 故意不含它 |
| 压缩只有一种 | **两种独立**：表压缩(`page_zip_des_t`) vs 页压缩(`Compression::Type` + punch hole) |
| `IORequest::PUNCH_HOLE_PORTION` | **不存在** → `PUNCH_HOLE = 256` / `DISABLE_PUNCH_HOLE_OPTIMISATION = 2048` |
| `PAGE_ZIP_MIN_SIZE` | **不存在** → `UNIV_ZIP_SIZE_MIN` / `UNIV_ZIP_SIZE_MAX` |
| `fil_space_t` 里有 `encryption_type` | **不存在** → `m_encryption_metadata` + `FSP_FLAGS_GET_ENCRYPTION(flags)` |
| `fil_space_t` 里有 punch hole 能力 | 在 **`fil_node_t`**（`punch_hole` / `block_size`） |
| DDL log 有 3 种操作类型 | **8 种**（`FREE_TREE_LOG` / `DELETE_SPACE_LOG` / `RENAME_SPACE_LOG` / `DROP_LOG` / `RENAME_TABLE_LOG` / `REMOVE_CACHE_LOG` / `ALTER_ENCRYPT_TABLESPACE_LOG` / `ALTER_UNENCRYPT_TABLESPACE_LOG`） |
| `struct Compression` 在 `os0file.h` | 在 **`storage/innobase/os/file.h:51`** |
| 存在 `mlog0mlog.h` | **不存在** |

**★ "一句话总结"的源码依据**（`include/mtr0log.h:61`，★ 源码没有专门的总结注释，这是最能概括设计意图的一段）：

```cpp
/* Size of initial info on REDO log
  1   byte  for LOG TYPE
  3-5 bytes for SPACE ID
  3-5 bytes for PAGE OFFSET */
constexpr size_t REDO_LOG_INITIAL_INFO_SIZE = 11;
```
> ★★ 索引的持久化 = **物理定位到页（space_id + page_no）+ 页内逻辑修改（offset + 内容）+ mtr 原子边界（SINGLE_REC_FLAG / MULTI_REC_END）**。恢复时必须先把页读进 buffer pool，用页头 `FIL_PAGE_LSN` 与 redo 的 `start_lsn` 比较做幂等过滤，再逐条应用。

---

## Misc

### 易混淆点

- **8.0.39 不存在 `MLOG_COMP_REC_INSERT` 这类不带 `_8027` 的旧名**——统一型是 67–76，COMPACT 与否由 log 内 index flag 携带。
- **`MLOG_PAGE_REORGANIZE` 不是整页重写**：payload 为 0，恢复时重新执行页整理。
- **bulk load 期间连 `MLOG_PAGE_CREATE` 都被抑制**（`MTR_LOG_NO_REDO` 下 `mlog_open()` 返回 false）；只有页分配用独立 mtr 写 redo。
- **root page no 恒定**：长高/降高都不换 root；`btr_truncate` 的原地重置只用于 DD 内部表。
- **TRUNCATE 是重建表**（新表空间、新 root），不是"重置 root"。
- **聚簇索引的 `PAGE_MAX_TRX_ID` 恒 0**——`btr_truncate` 正是借这个"无用字段"打崩溃恢复标记。
- **DROP INDEX 不回收 `.ibd` 物理大小**：页释放只在 FSP 层（链表/位图），punch hole 不参与。
- **node ptr 与 R-tree MBR 不新增 redo 类型**：分别复用 `MLOG_REC_INSERT` 与 `MLOG_REC_UPDATE_IN_PLACE`。
- **根页属于 TOP 段，不属于叶子段**：`btr_create` 第 1 次 `fseg_create(space, 0, ...)` 时 `page==0` ⇒ 新分配的页（即根页）归新建的非叶段；叶子段的段头只是**写在**根页上（`PAGE_BTR_SEG_LEAF`）。⇒ 释放时叶子段用 `fseg_free_step`（会把根页也释放），非叶段用 `fseg_free_step_not_header`（保留根页）。
- **段 ≠ 区 ≠ 页**：段 = `fseg_inode_t`（192B，含 3 条 extent 链表 + 32 个碎片页槽）；区 = 64 个连续页（1MiB，由 `xdes_t` 描述，state 决定归 FSP 还是归段）；页的归属由所在 extent 的 `XDES_ID` 间接决定。**段是逻辑归属单位，区是物理分配单位**。
- **`dict_index_t::page` 是 32 位位域**，不是 `page_no_t`——读的时候由 `dict_index_get_page()` 提升。且 **不存在 `dict_index_get_root()`**。
- **树高看似"应该缓存"却完全没有**：`grep height dict0mem.h` = 0 命中。而且 `btr_height_get()` **只读根页一页**（不是逐级下钻），代价是 1 次 IO、与树高无关。`dict0stats.cc:1818` 甚至会**再调一次**来判断树是否被并发改形。
- **`fseg_free_step` 没有"每步固定页数"常量**：`FSEG_FREE_PAGE_LIMIT` 不存在。粒度只有两种——1 个 extent 或 1 个碎片页。
- **表压缩与页压缩是两套完全独立的机制**：表压缩 = `ROW_FORMAT=COMPRESSED` + `DICT_TF_ZIP_SSIZE` + `page_zip_des_t`（BP 内 `buf_page_t::zip`）；页压缩 = `COMPRESSION=zlib/lz4` + `Compression::Type` + punch hole。**互不相关，可同时启用**。
- **恢复期重放的页一律不压缩写出**：`os/file.h:52` 注释 *"During recovery we don't have the compression type because the .frm file has not been read yet"*——因为 DD 还没加载、不知道压缩算法。
- **`PAGE_MAX_TRX_ID` 的更新不写 redo，但 `PAGE_LEVEL` 必须写**：前者只用于 MVCC 可见性（恢复时可保守取崩溃前最大 trx id），后者是树结构的一部分（错了整棵树就错）。
- **`page_create_low()` 只清 26 字节私有区**：`PAGE_LEVEL`(26) 与 `PAGE_INDEX_ID`(28) 在 `PAGE_HEADER_PRIV_END` 之后，由 `btr_create` 显式设置——这正是注释 "should not be written to after page creation" 的另一面。
- **DDL log 的"先 insert 再 delete"**：用**同一个 trx** 先插一条 `FREE_TREE_LOG` 再删掉它。trx 提交 ⇒ 记录消失；崩溃 ⇒ delete 未提交 ⇒ 记录留存 ⇒ 重启回放删树。**用事务的原子性反过来做 DDL 的撤销日志**。
- **punch hole 能力在 `fil_node_t` 不在 `fil_space_t`**：每个文件节点单独探测一次文件系统能力（`punch_hole` + `block_size`）。且 `block_size >= srv_page_size` 时整体关闭。

### ★★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `dict_index_t` 有 `seg` / `seg_header` 字段 | **不存在**。只有 `space`(32) 与 `page`(32) 两个字段；段头在**根页页头偏移 36/46**（物理偏移 74/84） |
| 两个段头"各占 20 字节" | **错**。`FSEG_HEADER_SIZE = 10`（space 4 + page_no 4 + offset 2），两个共 20 |
| 建索引先建叶子段 | **反了**。先 `fseg_create(...PAGE_BTR_SEG_TOP...)` 拿到根页，再建 LEAF 段（`btr0btr.cc:867,889`） |
| `btr0bulk.cc` / `Btr_bulk` 类 | **不存在**。bulk load 在 `btr0load.cc`（`Page_load`/`Btree_load`）+ `ddl/ddl0builder.cc`（`Builder`） |
| bulk load 用 `MTR_LOG_NONE` | **错**。是 **`MTR_LOG_NO_REDO`**（不记 redo 但**仍入 flush list**，靠 `Flush_observer` 强制落盘） |
| `ddl0log.cc` / DDL log | 文件名**不存在**。是 `log/log0ddl.cc` + `include/log0ddl.h` 的 `Log_DDL` 类 |
| `recv_page_is_new()` | 真名 **`recv_page_is_brand_new()`**（`log0recv.cc:2507`） |
| `MLOG_INIT_FILE_PAGE`(29) 是现行写法 | **已废弃**，注释明说"给 bulk create index 造成崩溃恢复问题"；替代者 `MLOG_INIT_FILE_PAGE2`(59) |
| `MLOG_PAGE_CREATE` 是唯一的建页 redo | 有 **4 类**：`MLOG_PAGE_CREATE`(19)、`MLOG_COMP_PAGE_CREATE`(37)、`_RTREE`(57/58)、`_SDI`(63/64) |
| `buf_page_free_punch_hole()` | **不存在** → `os_file_punch_hole()`（`os0file.h:1764`） |
| `innodb_fill_factor` 不存在/无效 | **存在**（`ha_innodb.cc:22477`，默认 100，范围 [10,100]），但**只影响 bulk load** |
| `PAGE_RTREE_LEVEL` | **不存在**（见 rtree.md）——本篇的 `PAGE_LEVEL` 是 B-Tree/R-Tree 共用的 |
| 恢复时 redo 按 LSN 全局排序 | **不是**。按 `(space_id, page_no)` 两级 hash 分桶，桶内链表按扫描顺序 |
| "页是全新的"看页头 LSN | **不是**。看**该页第一条 redo 的类型**是否是 `MLOG_INIT_FILE_PAGE2`，或干脆无 redo |

**★ 三个结构性事实**

1. **"每个索引两段"的物理落点是根页，不是 `dict_index_t`**。`dict_index_t` 只有 `space` + `page`；段头是根页页头的两个 10 字节字段。`btr_page_alloc_low()` 每次分配都要 `btr_root_get()` 拿根页再按 `level` 选段——**这个"回根页读段头"的动作是两段设计的运行时代价**。而 `btr_root_get()` 用 **SX latch**（非 X）来避免阻塞用户数据的读。

2. **`_8027` 后缀是 8.0.28 的分水岭**。redo 记录类型被劈成两代：8.0.27 及之前的格式（值 9-18、38-46、52-53）**冻结保留只用于读旧 redo**，8.0.28 起启用无后缀新号段（67-76），动因是 instant add column 需要 redo 携带额外元数据。`Log_format::VERSION_8_0_28 = 5` 与之同代。

3. **punch hole 排除 R-Tree 页**——因为 R-Tree 复用了 FIL 头那 8 字节作 SSN。这条把本篇（透明页压缩）与 rtree.md（SSN）连起来了：一个为省空间而"借"页头字段的设计，反过来让这些页无法参与透明压缩。

> 注：部分 `/* 【注】... */` 与 `★` 注释为本工作区人工批注，非 Oracle 上游原文。

### 崩溃安全的两条线

```
索引的内容安全 = 逐页 redo（MLOG_*，physiological 重放）
索引的存在安全 = DDL log（整棵树级：建了没提交就删、删了没提交就补删）
bulk load 例外  = 内容不写 redo（MTR_LOG_NO_REDO + MLOG_INDEX_LOAD 标记），
                  存在性仍由 DDL log 兜底
```

理解索引持久化的关键就是这个**双层结构**：redo 保证"已提交的页操作不丢"，DDL log 保证"未提交的整树操作不残留"——两者各管一层，缺一不可。

### 一句话总结

索引的物理存在以 **root page 为锚**：一个恒定的页号（元数据桥）→ 两段式分配（空间管理单位）→ 索引页布局（`PAGE_INDEX_ID` 归属）→ 逐页 physiological redo（内容持久化）→ DDL log（整树生命周期）——从 DD 里的一个 `"root"` 键走到字节，再走回崩溃恢复。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.6.1.2 InnoDB and MySQL Data Dictionary*（root page 存于 SDI）
- *MySQL 8.0 Reference Manual → 15.11 InnoDB File-Format*（页布局）

**相关文档**
- 索引元数据（root page 在 DD 里的存储）：[`metadata.md`](metadata.md)
- B 树页内操作：[`btr.md`](btr.md)
- 页通用物理结构：[`../innodb/physical/page_structure.md`](../innodb/physical/page_structure.md)
- 表空间/段/区通用管理：[`../innodb/physical/tablespace.md`](../innodb/physical/tablespace.md)
- redo 整体体系：[`../innodb/redo_log.md`](../innodb/redo_log.md)
- 崩溃恢复整体流程：[`../innodb/recovery.md`](../innodb/recovery.md)
- 索引 DDL 生命周期：[`operations.md`](operations.md)
- R-tree 的 MBR redo：[`rtree.md`](rtree.md)
