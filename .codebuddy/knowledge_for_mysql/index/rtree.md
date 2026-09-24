# R-tree：空间索引的存储结构与操作

> 基于 MySQL 8.0.39 源码。本篇讲 **R-tree 作为索引数据结构的完整实现**：MBR 编码、页布局、插入位置选择、页分裂、MBR 上溯、删除合并、搜索模式族、谓词锁——以及它与 B-tree（[`btr.md`](btr.md)）在每一步的差异。
>
> **边界**：空间数据类型/SRID/空间函数语义见 [`../server/datatype/gis.md`](../server/datatype/gis.md)；R-tree 在索引类型谱系里的位置见 [`types.md`](types.md)；`PAGE_CUR_RTREE_INSERT` 等模式在类型层的差异对照见 [`types.md`](types.md) 的空间索引一节。本篇是"索引存储结构"视角的权威出处。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [页与记录布局：MBR 的精确编码](#页与记录布局mbr-的精确编码)
    - [搜索模式族：7 种几何谓词](#搜索模式族7-种几何谓词)
  - 写路径
    - [插入：两阶段的子树选择](#插入两阶段的子树选择)
    - [页分裂：二次分裂（Quadratic Split）](#页分裂二次分裂quadratic-split)
    - [MBR 上溯：常规插入与分裂两条路](#mbr-上溯常规插入与分裂两条路)
    - [删除与合并](#删除与合并)
  - 并发与特殊机制
    - [谓词锁：R-tree 上没有 gap lock](#谓词锁r-tree-上没有-gap-lock)
    - [SSN 与 rtr_track：并发搜索的恢复基础](#ssn-与-rtr_track并发搜索的恢复基础)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

R-tree 是空间索引的存储结构：**按"最小包围矩形（MBR）的层次包含关系"组织数据**，而非按键值全序。叶子记录 = `MBR + PK`，非叶记录 = `子树 MBR + child page no`。搜索沿"查询区域与哪些子树相交"多路下潜。

### 用途

- `MBRContains` / `MBRIntersects` 等谓词的索引化求值；
- `ST_Contains` 等精确空间谓词的**粗筛**（MBR 相交是必要条件，精筛由几何函数做二次过滤）。

### 版本演进

R-tree 在 5.7 引入（源自 MyISAM 的 `rt_split.cc` 分裂代码），8.0 全程保持同一套核心算法。

---

## 理论基础

### 设计思想与权衡

**1. B-tree 的一切有序性假设在 R-tree 上失效。** 键值没有全序（两个矩形可以相交而不相等），于是：

| | B-tree | R-tree |
|---|---|---|
| 页内记录 | 按键序排列 | **无序**（MBR 间无全序） |
| 页内定位 | 二分 `page_cur_search_with_match` | **线性扫描全页** + 几何谓词比较 |
| 插入位置 | 唯一（键序确定） | 代价选择（最小面积扩大） |
| 插入后祖先 | **不变**（键值不变） | **可能变化**（子树 MBR 可能扩大） |
| 分裂 | 中点/插入点，保持键序 | **几何二分**，两半各自重算 MBR |
| node ptr | 全部键列 + child | **MBR + child**（仅 2 字段） |

**2. 每次插入都可能"向上传播"**——B-tree 插入只在分裂时影响上层，R-tree 插入只要子树 MBR 被扩大就要改写父记录。这是 R-tree 与 B-tree 最本质的结构性差异。

**3. 锁也换了一种。** R-tree 上没有 record gap lock/next-key lock——空间一致性由**谓词锁**（predicate lock，锁一个 MBR 区域而非一行记录）保证。

### 理论溯源

- Guttman, A. *R-Trees: A Dynamic Index Structure for Spatial Searching*. SIGMOD, 1984.——本篇的分裂算法（Quadratic Split）与"最小面积扩大"插入策略直接来自该论文；实现承自 MyISAM 的 `rt_split.cc`。

---

## 核心实现

### 主线与基础构件

#### 页与记录布局：MBR 的精确编码

**MBR 是定长 32 字节 double 数组，不是 varint**（`include/gis0rtree.ic:151-175`）：

```cpp
static inline void rtr_write_mbr(byte *data, const rtr_mbr_t *mbr) {
  const double *my_mbr = reinterpret_cast<const double *>(mbr);
  for (uint i = 0; i < SPDIMS * 2; i++) {
    mach_double_write(data + i * sizeof(double), my_mbr[i]);
  }
}
```

- `SPDIMS = 2`（`univ.i:651`），`DATA_MBR_LEN = SPDIMS * 2 * sizeof(double) = 32`（`data0type.h:216`）；
- 内存布局 `[xmin][xmax][ymin][ymax]`（坐标对交替）；
- 空 MBR 哨兵：`xmin = DBL_MAX, xmax = -DBL_MAX`；
- **SRID 头不进 R-tree 记录**——`get_mbr_from_store` 从"SRID 4B + WKB"的几何值中提取 MBR，SRID 校验信息缓存在 `dict_index_t::rtr_srs`。

两种记录布局：

```
叶子记录 = [ MBR(32B) ] + [ PK 列... ]（+ 系统列）
非叶记录 = [ MBR(32B) ] + [ child page_no(4B, DATA_SYS_CHILD) ]   ← 仅 2 个字段
```

非叶记录由 `rtr_index_build_node_ptr`（`gis0rtree.cc:131-178`）构造，`dtuple_set_n_fields_cmp(tuple, n_unique + 1)`——比较时 MBR 和 child page no 都参与（`DICT_INDEX_SPATIAL_NODEPTR_SIZE = 1` 指"几何字段数"，加上 page no 共 2）。点类型（`DATA_POINT`）在 R-tree 中退化为零面积 MBR，`fixed_len` 仍是 `DATA_MBR_LEN`（`btr0btr.cc:3908-3919` 断言）。

#### 搜索模式族：7 种几何谓词

`page0types.h:176-197` + `gis0geo.h:76-80` 注释（a = 搜索 MBR，b = 记录 MBR）：

| 模式 | 值 | 语义 |
|---|---|---|
| `PAGE_CUR_CONTAIN` | 7 | a contains b |
| `PAGE_CUR_INTERSECT` | 8 | a intersects b |
| `PAGE_CUR_WITHIN` | 9 | a within b |
| `PAGE_CUR_DISJOINT` | 10 | a disjoint b |
| `PAGE_CUR_MBR_EQUAL` | 11 | 两 MBR 全等 |
| `PAGE_CUR_RTREE_INSERT` | 12 | 插入定位（最小面积扩大） |
| `PAGE_CUR_RTREE_LOCATE` | 13 | 删除/更新定位（非叶层被改写为 WITHIN） |
| `PAGE_CUR_RTREE_GET_FATHER` | 14 | 取父 node ptr（精确匹配 child 元组） |

范围判断宏 `RTREE_SEARCH_MODE(mode)`：`mode >= PAGE_CUR_CONTAIN && mode <= PAGE_CUR_RTREE_GET_FATHER`。

**SQL 层映射**（`ha_innodb.cc:10087-10123`，`convert_search_mode_to_innobase`）：`HA_READ_MBR_CONTAIN → PAGE_CUR_CONTAIN` 等 5 种。`MBRContains()` 直接映射；`ST_Contains` 等**精确**谓词由优化器下推为 MBR 模式做**粗筛**，几何函数再做二次过滤。`PAGE_CUR_DISJOINT` 的非叶层搜索要同时检查 "DISJOINT 或 INTERSECT"（`gis0sea.cc:1576-1584`）——必须下潜进入可能与查询区相交的子树才能找到相离的叶子。

比较函数是 `cmp_dtuple_rec_with_gis` → `rtree_key_cmp`（`gis0geo.cc:222-260`），按模式调 `mbr_*_cmp` 系列。

---

### 写路径

#### 插入：两阶段的子树选择

逐层搜索被分流到 `rtr_cur_search_with_match`（`gis0sea.cc:1486-1812`；`btr0cur.cc:1367-1370` 的入口判断是 `dict_index_is_spatial(index) && page_mode >= PAGE_CUR_CONTAIN`）。非叶层 `PAGE_CUR_RTREE_INSERT` 分支（`gis0sea.cc:1560-1613`）：

```cpp
case PAGE_CUR_RTREE_INSERT:
  cmp = cmp_dtuple_rec_with_gis(tuple, rec, offsets, PAGE_CUR_WITHIN,
                                index->rtr_srs.get());
  if (cmp != 0) {
    double area{0.0};
    increase = rtr_rec_cal_increase(tuple, rec, offsets, &area,
                                    index->rtr_srs.get());
    if (increase >= DBL_MAX || (increase == 0 && std::isfinite(area))) {
      increase = DBL_MAX / 2;              // 溢出保护
    }
    if (increase < least_inc) {
      least_inc = increase;
      best_rec = rec;
    } else if (best_rec && best_rec == first_rec) {
      /* if first_rec is set, we will try to avoid it */
      least_inc = increase;
      best_rec = rec;
    }
  }
  break;
```

**两阶段策略**：

1. **先找完全容纳者**：对每条记录先判 `WITHIN`（插入 MBR ⊆ 子树 MBR）——命中第一条就 `break`（注释 "it will break once it finds the first MBR that can accommodate"），**不比面积**；
2. **无容纳者**：遍历整页选**面积扩大最小**的子树（`least_inc`）——Guttman choose-subtree 的最小面积扩大变体。面积计算 `rtree_area_increase`（`sql/gis/rtree_support.cc`）= `join_area - area`，且考虑 SRS（球面/投影坐标）。

两个工程 hack：新记录太大时**尽量避开页首**（`first_rec` 规避，压缩页 MIN_REC 原地更新有坑）；`mbr_adj` 置位（选了"非完全容纳"的子树）时 `BTR_MODIFY_LEAF` 必须**升级为 `BTR_MODIFY_TREE` 重试**（`btr0cur.cc:1373-1381`）——因为父 MBR 要变。

**与 B-tree 的本质差异**：B-tree 二分定位唯一插入点 O(log n)；R-tree 页内无序，线性扫全页，几何谓词比较，且**一次搜索可能多路命中**（多个子树与查询区域相交都要下潜），路径记录在 `rtr_info->path`/`parent_path` 栈里。

#### 页分裂：二次分裂（Quadratic Split）

`rtr_page_split_and_insert`（`gis0rtree.cc:884-1205`）先装入 `rtr_split_node_t` 数组，再调 `split_rtree_node`（`gis0geo.cc:145-220`，源自 MyISAM）：

```cpp
pick_seeds(node, n_entries, &a, &b, n_dim, srs);   // 挑种子
a->n_node = 1;
b->n_node = 2;
...
for (i = n_entries - 2; i > 0; --i) {
  /* Can't write into group 2 */
  if (all_size - (size2 + key_size) < min_size) { mark_all_entries(node, n_entries, 1); break; }
  /* Can't write into group 1 */
  if (all_size - (size1 + key_size) < min_size) { mark_all_entries(node, n_entries, 2); break; }
  pick_next(node, n_entries, g1, g2, &next, &next_node, n_dim, srs);
  ... 归入代价小的组
}
```

- **PickSeeds**：O(n²) 枚举所有记录对，取 `mbr_join_area(a,b) - a.square - b.square` **最大**的两条作两组种子——二次分裂（Quadratic Split），**不是**线性分裂，也**不是** R* 的按轴排序 + 重叠最小化；
- **PickNext**：每轮选"与 g1 并面积 − 与 g2 并面积"绝对差最大（最"偏心"）的记录，归入代价小的组；差为 0 时 `ut_rnd_gen_bool()` 随机；
- 调用参数 `min_size=0, size1=size2=2`：**没有最小填充率约束**（对比 Guttman 论文的 40% 下限），靠总字节数控制。

记录搬迁（`rtr_split_page_move_rec_list`，`gis0rtree.cc:732-875`）：

```cpp
for (cur_split_node = node_array; cur_split_node < end_split_node; ++cur_split_node) {
  if (cur_split_node->n_node != first_rec_group) {       // ① 不属于留在旧页的组
    lock_rec_store_on_page_infimum(block, cur_split_node->key);   // ② 锁暂存到 infimum
    rec = page_cur_insert_rec_low(..., new_page_cursor, ...);     // ③ 逐条复制到新页
    lock_rec_restore_from_page_infimum(new_block, rec, block);    // ④ 锁还原
    rec_move[moved++] = {new_rec, old_rec, false};
  }
}
lock_rtr_move_rec_list(new_block, block, rec_move, moved);        // ⑤ 批量搬锁
/* 再从旧页删除这些记录 */                                        // ⑥
```

要点：`first_rec_group` 保证 **MIN_REC 记录留在旧页最左**；锁逐条经 infimum 暂存再还原；压缩页 `page_zip_compress` 失败 → `page_zip_reorganize` → 仍失败走整页字节拷贝兜底；分裂前后旧页/新页/根页都刷新 **SSN**。

分裂后两半各自的 MBR 由 `rtr_page_cal_mbr`（`gis0rtree.ic:35-91`）重算——初始化为空盒（`DBL_MAX` 哨兵），遍历页上所有记录取各维 min/max **并集**。

#### MBR 上溯：常规插入与分裂两条路

**常规插入（无分裂）：`rtr_ins_enlarge_mbr`**（`gis0rtree.cc:1207-1269`）：

```cpp
for (ulint i = 1; i < btr_cur->tree_height; i++) {
  node_visit = rtr_get_parent_node(btr_cur, i, true);   // 从 parent_path 取该层父记录
  if (node_visit->mbr_inc == 0) continue;               // ★ 该层 MBR 未变则剪枝，不再上溯
  rtr_page_cal_mbr(index, block, &new_mbr, heap);       // 重算子页真实 MBR（非增量放大）
  offsets = rtr_page_get_father_block(...);
  rtr_update_mbr_field(&cursor, offsets, nullptr, page, &new_mbr, nullptr, mtr);
  block = 上一层;
}
```

- **不是递归**：沿搜索时记录的 `rtr_info->parent_path`（下潜时压栈，`mbr_inc` 记录"该子树 MBR 需扩大多少"）自底向上**迭代**；
- **提前终止**：某层 `mbr_inc == 0`（新记录完全被该层 MBR 容纳）就不再继续上层；
- 调用点：`row0ins.cc:3019/3030/3049` 三处（modify/optimistic/pessimistic 插入成功后），前置条件 `dict_index_is_spatial(index) && rtr_info.mbr_adj`。

**分裂路径：`rtr_adjust_upper_level`**（`gis0rtree.cc:558-722`），只在页分裂后被 `rtr_page_split_and_insert` 调用，五步：

1. 在父页定位旧页 node ptr，`rtr_update_mbr_field` 换成分裂后旧页的新 MBR；
2. 把 `parent_path` 中对应层的 `mbr_inc` 清零（父已更新，防 `rtr_ins_enlarge_mbr` 重复上溯）；
3. `rtr_index_build_node_ptr` 构造**新页** node ptr；
4. 用普通 **`PAGE_CUR_LE`** 搜父页插入新 node ptr（node ptr 有 page no 参与 `n_fields_cmp`），optimistic 失败转 pessimistic（可能引发上层再分裂，继续向上）；
5. `lock_prdt_update_parent` 把父页中涉及旧 MBR 的**谓词锁**按新/旧 MBR 拆分；最后重接同层双向链（`FIL_PAGE_PREV/NEXT`，与 B-tree 同模式）。

两者落地都经 `rtr_update_mbr_field`（248-555 行），按情况三分支：MIN_REC 记录走 `rtr_update_mbr_field_in_place` 原地改 + 自组 `MLOG_REC_UPDATE_IN_PLACE` redo；页上只有 1 条记录走 insert+delete 防合并；否则 delete+insert 防分裂。

#### 删除与合并

**`rtr_merge_mbr_changed`**（`gis0rtree.cc:1517-1551`）：比较两个 node ptr 的 MBR 并算**并集**（逐维 min/max）——注意"合并后 MBR 收缩"其实不发生在这一步；真正的收缩由 `rtr_update_mbr_field(child_page ...)` 用**保留页的实际记录**重算。

**`rtr_merge_and_update_mbr`**（`gis0rtree.cc:1553-1576`）：

```cpp
changed = rtr_merge_mbr_changed(cursor, cursor2, offsets, offsets2, &new_mbr);
if (changed) {
  rtr_update_mbr_field(cursor, offsets, cursor2, child_page, &new_mbr, nullptr, mtr);
  // ★ 原子完成：保留记录改写 + cursor2 记录删除（用保留页首记录重构 node ptr）
} else {
  rtr_node_ptr_delete(cursor2, mtr);   // 两 node ptr 完全相同：直接删重复项
}
```

调用现场（`btr0btr.cc:3294-3322`，`btr_merge_blocks` 内）体现 R-tree 与 B-tree 的又一个差异：**B-tree 合并只需删一个 node ptr**（键序天然无损）；R-tree 必须**保留一个 node ptr 并改写其 MBR**——且按"父 node ptr 是否 MIN_REC"决定保留哪个（MIN_REC 必须留）：

```cpp
if (rec_info & REC_INFO_MIN_REC_FLAG) {
  /* 父 node ptr 是最小记录：保留它，删 merge 页的 node ptr */
  rtr_merge_and_update_mbr(&father_cursor, &cursor2, offsets, offsets2, merge_page, mtr);
} else {
  /* 否则保留 merge 页的 node ptr、删父 node ptr——为保持上层记录顺序 */
  rtr_merge_and_update_mbr(&cursor2, &father_cursor, offsets2, offsets, merge_page, mtr);
}
```

**合并条件与 underfill 阈值**：`btr_can_merge_with_page`（`btr0btr.cc:4585`）**B-tree/R-tree 共用**，不做几何检查；触发阈值也是同一个 `BTR_CUR_PAGE_COMPRESS_LIMIT(index) = UNIV_PAGE_SIZE * index->merge_threshold / 100`（默认 50%，可用 `ALTER TABLE ... ALTER INDEX idx MERGE_THRESHOLD=n` 调整）。**R-tree 没有专门的填充率**（Guttman 建议的 40% 下限未实现）。

---

### 并发与特殊机制

#### 谓词锁：R-tree 上没有 gap lock

`lock0lock.h:100` 的设计注释："*Also, each predicate lock (from GIS) is tied to a page, not a record.*"

- 锁标志：`LOCK_PREDICATE = 8192`、`LOCK_PRDT_PAGE = 16384`，各有独立哈希表 `lock_sys->prdt_hash` / `prdt_page_hash`；
- 锁谓词结构（`lock0prdt.h:40`）：`typedef struct lock_prdt { void *data; uint16 op; } lock_prdt_t;`——`data` 指向 MBR，`op` 是 `PAGE_CUR_*` 几何模式；
- **读路径**：serializable 隔离时 `rtr_info->need_prdt_lock` → `lock_init_prdt_from_mbr` 构造 MBR 谓词锁（`btr0cur.cc:1417` 起）；定位类搜索还会 `lock_place_prdt_page_lock` 给相关子页上页锁防收缩；
- **写路径**：插入前 `lock_prdt_insert_check_and_lock`（检查已有谓词锁与插入 MBR 是否冲突，含 `LOCK_INSERT_INTENTION` 语义）；冲突判定 `lock_prdt_has_to_wait` 用 **MBR 相交测试**代替记录位图测试；
- **分裂/丢弃**：`lock_prdt_update_split` / `lock_prdt_update_parent` / `lock_prdt_page_free_from_discard` 随结构变化迁移。

#### SSN 与 rtr_track：并发搜索的恢复基础

**SSN（Split Sequence Number）**：`dict_index_t::rtr_ssn`（`rtr_ssn_t`，mutex + 自增 seq_no），存在**页头**（`page_get_ssn_id`/`page_set_ssn_id`）与**搜索栈节点**（`node_visit_t.seq_no`，下潜前取当前值）。分裂时旧页保留 `current_ssn`、新页与根页写入 `next_ssn`（`gis0rtree.cc:997-1000、1158-1161`）。**用途**：并发搜索游标比对栈中记录的 SSN 与页上 SSN，检测"搜索期间发生了分裂"，决定恢复/重走路径（`rtr_cur_restore_position`、`rtr_rebuild_path`，`gis0sea.cc:55-130、1011-1071`）。

**rtr_track**：`dict_index_t::rtr_track`（`rtr_info_track_t *`），`rtr_active` 向量注册/注销每个活跃搜索游标（`rtr_init_rtr_info` / `rtr_clean_rtr_info`）。消费点：`rtr_check_discard_page`（`gis0sea.cc:1073-1121`）在页被合并/丢弃时遍历所有活跃 `rtr_info`，清理其中引用该页的搜索路径。

---

## 相关的系统变量/状态变量

| 变量/常量 | 默认值 | 说明 |
|------|--------|------|
| `DICT_INDEX_SPATIAL_NODEPTR_SIZE` | 1 | 非叶记录的几何字段数（加 page no 共 2） |
| `SPDIMS` | 2 | 维度数（写死为 2 维，不支持 3D 索引） |
| `DATA_MBR_LEN` | 32 字节 | MBR 编码长度（4 × double） |
| `index->merge_threshold` | 50（%） | 与 B-tree 共用的 underfill 阈值（`DICT_INDEX_MERGE_THRESHOLD_DEFAULT`） |
| `dict_index_t::srid / rtr_srs` | — | SRID 校验信息（不进记录） |
| `dict_index_t::rtr_ssn / rtr_track` | — | 分裂序号分配器 / 活跃搜索游标跟踪 |

---

## Misc

### 操作 × 代码路径对照表

| 操作 | 共用 B-tree 路径 | R-tree 专用 |
|---|---|---|
| 逐层搜索定位 | `btr_cur_search_to_nth_level` 主干（加锁/crab） | 层内换 `rtr_cur_search_with_match`；路径栈；**AHI 禁用**（`btr0cur.cc:1390`） |
| 记录插入 | `btr_cur_optimistic/pessimistic_insert` | 入口模式 `PAGE_CUR_RTREE_INSERT`；`mbr_adj` 强制 BTR_MODIFY_TREE；成功后 `rtr_ins_enlarge_mbr` |
| 页分裂 | 新页分配 `btr_page_alloc`、`btr_page_create` | `split_rtree_node`（二次分裂）+ `rtr_split_page_move_rec_list` + `rtr_adjust_upper_level`；SSN/谓词锁迁移 |
| node ptr 构建 | `dict_index_build_node_ptr`（全键+child） | `rtr_index_build_node_ptr`（MBR+child） |
| 上层键修改 | 无（B-tree 上层键不变） | `rtr_update_mbr_field[_in_place]` |
| 页合并 | `btr_merge_blocks`、`btr_can_merge_with_page` | 合并后 `rtr_merge_and_update_mbr`；`rtr_check_discard_page` 清理活跃搜索 |
| 根页提升 | `btr_lift_page_up`（共用） | 找父亲用 rtree 版；`lock_prdt_rec_move` |
| 范围扫描 | `btr_pcur` | `rtr_pcur_open_low` / `rtr_pcur_move_to_next`（先吐完 `rtr_info->matches` 缓存的命中再翻页） |
| 行数估算 | `btr_estimate_n_rows_in_range` | `rtr_estimate_n_rows_in_range`（只接受 5 种 MBR 模式） |
| redo | 通用 `MLOG_REC_*` | MBR 原地改写自组 `MLOG_REC_UPDATE_IN_PLACE`；`mtr0log.cc:812` 按 `NODEPTR_SIZE` 解析 node ptr |
| DDL 构建 | `Builder` 排序批量建页 | **`RTree_inserter`**（扫描阶段逐条插入，见 [`types.md`](types.md)） |

### 易混淆点

- **分裂是 Quadratic Split**，不是线性分裂、不是 R*；且**没有最小填充率约束**（Guttman 的 40% 未实现），underfill 阈值与 B-tree 共用 `merge_threshold`（50%）。
- **"合并后 MBR 收缩"不在 `rtr_merge_mbr_changed`**——它算的是并集；真实收缩由 `rtr_update_mbr_field(child_page)` 用保留页记录重算。
- **插入的子树选择是两阶段**：先找完全容纳者（WITHIN，第一条命中即停），无容纳者才选最小面积扩大——不是纯粹的"最小扩大"。
- **R-tree 没有 gap/next-key lock**：空间一致性靠谓词锁（锁 MBR 区域）；`LOCK_PREDICATE`/`LOCK_PRDT_PAGE` 是独立于 record lock 的哈希表。
- **AHI 对 R-tree 禁用**（无序键值没有稳定的哈希学习语义）。
- **`PAGE_CUR_RTREE_INSERT` 的搜索不能只看 `PAGE_CUR_LE` 语义**：B-tree 的 GE/G 在 `btr_cur_search_to_nth_level` 里被翻译成 L/LE，R-tree 的 mode **原样透传**到非叶层（`btr0cur.cc:1014-1031`）。

### 一句话总结

R-tree = B-tree 的骨架（页、分裂/合并的物理操作、`btr_cur_search_to_nth_level` 主干）+ 几何语义替换（无序、多路下潜、最小面积扩大、MBR 上溯、二次分裂）+ 一套全新的并发原语（谓词锁、SSN、路径栈恢复）——理解它的关键不是算法本身（Guttman 1984），而是**"每次插入都可能改写祖先"这一条对 B-tree 不变量的破坏**如何在代码里处处设防。

---

## 参考

**论文**
- Guttman, A. *R-Trees: A Dynamic Index Structure for Spatial Searching*. SIGMOD, 1984.（PickSeeds/PickNext、最小面积扩大、40% 填充率——MySQL 只实现了前两者）

**官方文档**
- *MySQL 8.0 Reference Manual → 15.6.2.5 Spatial Indexes*（使用限制：NOT NULL、SRID）
- *MySQL 8.0 Reference Manual → 14.16 Spatial Convenience Functions*（MBR 谓词族）

**相关文档**
- 空间类型/SRID/空间函数语义：[`../server/datatype/gis.md`](../server/datatype/gis.md)
- R-tree 在索引类型谱系中的位置：[`types.md`](types.md)
- B-tree 对照（分裂/合并/搜索的完整实现）：[`btr.md`](btr.md)
- 谓词锁在锁体系中的位置：[`../infra/lock/transactional/innodb_trx_lock.md`](../infra/lock/transactional/innodb_trx_lock.md)
- 空间索引为什么不能进 change buffer：[`ibuf.md`](ibuf.md)
- 全文倒排（另一条非 B-tree 结构）：[`inverted.md`](inverted.md)
