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

### 版本演进

- **8.0.30+ redo 格式统一**：8.0.39 里 `MLOG_*` 存在"旧 `_8027` 型"与"新统一型"两套并存——旧型只用于**读**旧 redo，真正生效的是 67–76 这组统一型（COMPACT 与否由 log 内 index flag 携带）。**不存在** `MLOG_COMP_REC_INSERT` 这类不带后缀的旧名。

---

## 理论基础

### 设计思想与权衡

**1. 每个索引独立两段，root 页是非叶段的第一页。** 好处：DROP INDEX 时按段整体释放、不需要逐页收集；坏处：小索引也要 2 个段 inode + 若干 frag 页（`btr_create` 里两次 `fseg_create`）。

**2. root page no 恒定。** 树长高（`btr_root_raise_and_insert`）时**不换 root**——分配新页承载旧 root 内容、root 原地清空并 level+1。这使 DD 里的 `root` 键永不过期，是元数据与物理的稳定锚点。

**3. redo 是"物理逻辑"（physiological）**：不是字节 diff，也不是纯逻辑操作——是**"在页内重放一次操作"**（带 cursor 偏移 + 记录尾段）。恢复时必须**先读页**（payload 依赖页现状），换来的是 redo 体积小、且能容忍页内偏移变化。

**4. bulk load 不写逐页 redo，靠 DDL log。** 批量建索引时树页是"新造的"，逐页写 redo 无意义（恢复也用不上）——只写一条 `MLOG_INDEX_LOAD` 标记 + DDL log 记录"这棵树若没提交就删掉"。

**5. 空间释放是分批小步的。** `fseg_free_step` 每次只释放一个 extent / 一个 frag page / 一个 inode，跨多个 mtr 循环——单个 mtr 不能持有过多页锁、不能产生过大 redo。

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
| `PAGE_BTR_SEG_LEAF` / `PAGE_BTR_SEG_TOP` | 74 / 84 | 两段头，**仅 root 有效**；所有索引页都预留这 56 字节段头空间，是"每个索引页都能成为 root"的前提 |

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
