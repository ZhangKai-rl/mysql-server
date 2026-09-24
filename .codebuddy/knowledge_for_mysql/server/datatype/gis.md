# GIS / 空间数据：几何类型、R-Tree 索引与 GeoJSON

> 基于 MySQL 8.0.39 源码。GIS 是**两个独立子系统**的组合：① 几何值的存储（自研 SRID+WKB 二进制）+ R-Tree 空间索引（InnoDB 层）；② GeoJSON 只是几何值的**外部文本交换格式**（复用 JSON 解析器）。
>
> **边界**：R-Tree 空间索引使用的**谓词锁**（`LOCK_PREDICATE` / `LOCK_PRDT_PAGE`）第一主语是"锁"，已在 [`../../infra/lock/transactional/innodb_trx_lock.md`](../../infra/lock/transactional/innodb_trx_lock.md)「谓词锁」节完整剖析，本篇不重复。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [一、几何类型与存储格式](#一几何类型与存储格式)
- [二、InnoDB R-Tree 索引结构](#二innodb-r-tree-索引结构)
- [三、SPATIAL 索引的 range 扫描](#三spatial-索引的-range-扫描)
- [四、MBR 假阳性与精确过滤](#四mbr-假阳性与精确过滤)
- [五、GeoJSON 转换](#五geojson-转换)
- [完整调用栈](#完整调用栈)
- [★ 工程实现技法](#-工程实现技法)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

**GeoJSON 与 JSON 共享解析器，但存储与执行是完全独立的子系统**：

- JSON 是 JSONB 文档类型（树形、路径表达式、靠生成列建普通索引）
- GeoJSON 只是 geometry 的**外部文本表示**；几何值内部用自研的 **SRID + WKB** 二进制格式，落 InnoDB 为 `DATA_GEOMETRY`，可建 **R-Tree 空间索引**

两者耦合点只有"一进一出"两个函数：`ST_GeomFromGeoJSON`（用 JSON 解析器读入）和 `ST_AsGeoJSON`（构造 `Json_dom` 输出）。

★ 关键点：**几何值从不以 JSONB 形式存储**。R-Tree 索引键是**几何的 MBR（最小包围盒）**，不是几何本身。

## 理论基础

### R-Tree：多维数据的平衡树

R-Tree 是 B-Tree 在**多维空间**上的推广（Guttman 1984）：

- B-Tree 按键值**一维排序**组织；R-Tree 按**空间位置**组织（每个节点存其子树所有对象的 MBR）
- 查询不是"找到 key"，而是"找到与查询区域相交的所有对象"（空间查询）

### MBR 近似：为什么需要"假阳性过滤"

R-Tree 索引只存 MBR（最小包围盒），**不存完整几何**。MBR 是近似：两个几何的 MBR 相交，**不代表几何本身真正相交**（例如两个对角分布的 L 形多边形，MBR 重叠但几何不相交）。

所以 R-Tree 扫描得到的是**候选集**，必须做**精确过滤**（recheck）——这是 GIS 索引与普通 B-Tree 索引最本质的差异（普通索引的 key 匹配是精确的，GIS 索引的 MBR 匹配是近似的）。

### Quadratic Split：R-Tree 的分裂策略

R-Tree 页满时如何分裂成两页？Guttman 提出三种：Exhaustive / **Quadratic（平方）** / Linear（线性）。InnoDB 用 Quadratic Split：

1. **pick_seeds**：选两个"合并后面积增量最大"的记录作为两组种子（让两组尽量分开）
2. **pick_next**：每次选"加入两组的面积增量差最大"的记录，分配到增量较小的一组

---

## 一、几何类型与存储格式

### 几何类型体系

`sql/spatial.h` 的 `Geometry` 基类，派生：

| 类 | feature_dimension | 说明 |
|---|---|---|
| `Gis_point` | 0 | 点，2 个 double |
| `Gis_line_string` | 1 | 线，点序列 |
| `Gis_polygon` | 2 | 面，外环 + 内环（`Gis_polygon_ring`） |
| `Gis_multi_point` / `...line_string` / `...polygon` | — | 集合 |
| `Gis_geometry_collection` | — | 混合集合 |

### 存储格式：SRID + WKB（swkb）

```
4 字节 SRID(小端) + 1 字节字节序(wkb_ndr) + 4 字节 wkbType + 几何数据
```

| 常量 | 值 | 位置 |
|---|---|---|
| `SRID_SIZE` | 4 | `sql/spatial.h` |
| `WKB_HEADER_SIZE` | 1+4=5 | 同上 |
| `GEOM_HEADER_SIZE` | SRID_SIZE + WKB_HEADER_SIZE = 9 | 同上 |

`wkbType` 枚举：`wkb_point=1, wkb_linestring=2, wkb_polygon=3, wkb_multipoint=4, ... wkb_geometrycollection=7`。

内部用位域 `Flags_t` 紧凑编码（`bo:1, dim:2, nomem:1, geotype:5, nbytes:30, props:12, zm:2, unused:11`，共 64 位）。

### 字段映射

| | 空间列 | JSON 列 |
|---|---|---|
| 字段类 | `Field_geom : Field_blob` | `Field_json : Field_blob` |
| `type()` | `MYSQL_TYPE_GEOMETRY` | `MYSQL_TYPE_JSON` |
| 存储 | 定长头 + WKB | JSONB 文档树 |
| 校验 | 存入前 WKB 合法性校验 | JSON 语法校验 |
| 索引 | **R-Tree 空间索引** | 生成列/函数索引/多值索引 |
| InnoDB 映射 | `DATA_GEOMETRY` | `DATA_BLOB` |

---

## 二、InnoDB R-Tree 索引结构

> InnoDB 源码在 `storage/innobase/`（注意不是 `storage/innodb/`），GIS 集中在 `storage/innobase/gis/` 与 `storage/innobase/include/gis0*.h`。

### MBR：32 字节固定键

```cpp
typedef struct rtr_mbr {
  double xmin, xmax;   // x 轴最小/最大
  double ymin, ymax;   // y 轴最小/最大
} rtr_mbr_t;
```

- 内存表示：`rtr_mbr_t`（`sql/gis/rtree_support.h`）
- 页内物理布局：`SPDIMS * 2` 个 double（`xmin,xmax,ymin,ymax`），共 **32 字节**（`DATA_MBR_LEN = SPDIMS * 2 * sizeof(double)`）

### 与 B-Tree 的页结构差异

| 维度 | B-Tree | R-Tree |
|---|---|---|
| 节点存什么 | 索引 key（一维有序） | **子节点 MBR**（多维包围盒） |
| 记录顺序 | 按 key 排序 | 无排序，由分裂算法决定 |
| 方向性 | ASC/DESC | 无方向 |
| 分裂 | 中位数分裂 | Quadratic Split |

三个关键点：

1. **MBR 是记录的第 0 字段**（`gis0rtree.ic` 注释 "The mbr address is in the first field"）。非叶子节点的 MBR 由 `rtr_page_cal_mbr` 遍历子页所有记录、合并其 MBR 得到（初始化为 `xmin=DBL_MAX, xmax=-DBL_MAX` 后取 min/max）。

2. **叶子 MBR 与父 MBR 是 "WITHIN" 关系**（`btr0btr.cc`）：
   ```cpp
   // For spatial index, the MBR in the parent rec could be different
   // with that of first rec of child, their relationship should be "WITHIN"
   if (dict_index_is_spatial(index)) {
     ut_a(!cmp_dtuple_rec_with_gis(tuple, btr_cur_get_rec(&cursor), offsets,
                                   PAGE_CUR_WITHIN, index->rtr_srs.get()));
   }
   ```
   父节点的 MBR 是子页所有 MBR 的**并集**，所以父 MBR 一定**包含**子页第一个记录的 MBR。

3. **SSN（Split Sequence Number）**：每个索引维护 `rtr_ssn`，**仅在页分裂时递增**，用于检测搜索期间页是否发生分裂（搜索走一条路径，若中途页分裂了，路径可能失效，要靠 SSN 重新检测）。这是 R-Tree 并发搜索的核心机制（类比 B-Tree 的 `PAGE_CUR_LE` 位置保存，但 R-Tree 的搜索路径更复杂）。

### 搜索路径结构 rtr_info_t

`rtr_info_t`（`gis0type.h`）维护 R-Tree 搜索的中间态：

| 字段 | 作用 |
|---|---|
| `path` | 匹配的非叶子页栈（DFS 搜索路径） |
| `parent_path` | 插入用的父路径 |
| `matches` | 叶子层匹配记录缓存（`matched_rec_t`） |
| `mbr` / `search_mode` | 搜索 MBR 与模式 |

### Quadratic Split 的完整流程

```
页满 → rtr_page_split_and_insert（gis0rtree.cc）
  ├─ 分配新页
  ├─ 设置 SSN（新页继承旧 SSN，旧页分配新 SSN）
  ├─ split_rtree_node（gis0geo.cc）
  │     ├─ pick_seeds：选面积增量最大的两个种子
  │     └─ pick_next：循环分配剩余记录（增量差最大者优先）
  ├─ 移动记录、更新锁（lock_rtr_move_rec_list）
  └─ 上推 MBR 扩大（rtr_ins_enlarge_mbr）
```

---

## 三、SPATIAL 索引的 range 扫描

### 三层映射：SQL 算子 → handler → 页游标

**① SQL 空间算子 → `HA_READ_MBR_*`**（`range_analysis.cc`）：

```cpp
SP_EQUALS_FUNC     -> HA_READ_MBR_EQUAL
SP_DISJOINT_FUNC   -> HA_READ_MBR_DISJOINT
SP_INTERSECTS_FUNC -> HA_READ_MBR_INTERSECT
SP_TOUCHES_FUNC    -> HA_READ_MBR_INTERSECT
SP_CROSSES_FUNC    -> HA_READ_MBR_INTERSECT
SP_WITHIN_FUNC     -> HA_READ_MBR_CONTAIN   // 注意：反向！
SP_CONTAINS_FUNC   -> HA_READ_MBR_WITHIN    // 注意：反向！
SP_OVERLAPS_FUNC   -> HA_READ_MBR_INTERSECT
```

★ **`WITHIN` / `CONTAINS` 反向映射**：因为存储引擎对这两个谓词的参数顺序是反向实现的（历史兼容 MyISAM/InnoDB 约定），注释明确说明。

**② `HA_READ_MBR_*` → `PAGE_CUR_*`**（`ha_innodb.cc`）：

```cpp
HA_READ_MBR_CONTAIN    -> PAGE_CUR_CONTAIN
HA_READ_MBR_INTERSECT  -> PAGE_CUR_INTERSECT
HA_READ_MBR_WITHIN     -> PAGE_CUR_WITHIN
HA_READ_MBR_DISJOINT   -> PAGE_CUR_DISJOINT
HA_READ_MBR_EQUAL      -> PAGE_CUR_MBR_EQUAL
```

**③ 迭代器**：`GeometryIndexRangeScanIterator` 与 `IndexRangeScanIterator` 的区别仅在 `Read()`——它对每个 range 调 `ha_index_read_map(min_key=MBR, rkey_func_flag=HA_READ_MBR_*)`，然后 `ha_index_next_same` 读相同 MBR key 的下一条。

### 邻域搜索 rtr_cur_search_with_match

R-Tree 的 DFS 搜索（`gis0sea.cc`）：

```
rtr_cur_search_with_match
  ├─ 非叶子节点：按模式调 cmp_dtuple_rec_with_gis 判断子节点 MBR 是否匹配
  │     ★ CONTAIN/INTERSECT/MBR_EQUAL 三种模式都需同时检查 CONTAIN 和 INTERSECT
  │       （父 MBR 是子页的并集，可能比实际覆盖范围大，需双重判定）
  ├─ 匹配的非叶子节点压入 rtr_info->path
  └─ 叶子匹配记录收集到 rtr_info->matches
```

★ 关键细节（非叶子层的双重判定）：

```cpp
case PAGE_CUR_CONTAIN: case PAGE_CUR_INTERSECT: case PAGE_CUR_MBR_EQUAL:
  cmp = cmp_dtuple_rec_with_gis(tuple, rec, offsets, PAGE_CUR_CONTAIN, srs);
  if (cmp != 0)
    cmp = cmp_dtuple_rec_with_gis(tuple, rec, offsets, PAGE_CUR_INTERSECT, srs);
```

因为父节点 MBR 是"子页所有 MBR 的并集"，可能比子页实际覆盖范围**更大**，单纯 CONTAIN 或 INTERSECT 判断会漏掉或误判，所以非叶子层要双重判定。

### 路径遍历与回溯

`rtr_pcur_getnext_from_path`：从 `path` 栈弹出节点，用 SSN 检测分裂，递归下降到叶子。`rtr_pcur_move_to_next` 在 `matches` 向量耗尽后移动到下一个 path 节点。这是典型的 **R-Tree DFS + 回溯**。

---

## 四、MBR 假阳性与精确过滤

### 为什么必须精确过滤

MBR 相交是**充分但不必要**的条件：两个几何的 MBR 相交，不代表几何本身相交。所以 R-Tree 扫描只是"候选集裁剪"，**必须二次精确过滤**。

### inexact 标志

```cpp
// range_analysis.cc
// R-tree queries are based on bounds, and must be rechecked.
*inexact = true;
```

`SEL_TREE` / `SEL_ROOT` 上的 `inexact` 标志沿树传播，供 hypergraph 优化器判断"是否需要重新执行原始谓词做精确过滤"。

### 只有 MBR 谓词能走 SPATIAL 索引

只有 `SP_EQUALS/DISJOINT/INTERSECTS/TOUCHES/CROSSES/WITHIN/CONTAINS/OVERLAPS` 这些（MBR* 系列与对应 ST_ 系列）才建立 R-Tree range；其它空间函数不用索引（`range_analysis.cc` 注释："We cannot involve spatial indexes for queries that don't use MBREQUALS(), MBRDISJOINT(), etc."）。

---

## 五、GeoJSON 转换

**输出方向**（`ST_AsGeoJSON`）—— **构造 `Json_dom`，不是拼文本**：`Item_func_as_geojson` 继承 `Item_json_func`，实现 `val_json`；`geometry_to_json` 用 `wkb_parser` 扫描内部 swkb，递归 `append_geometry` 填充 `Json_object`，最后序列化成文本。

**输入方向**（`ST_GeomFromGeoJSON`）—— **走标准 JSON 解析器**：`get_json_wrapper` → `json_value` → `parse_json` → `Json_dom::parse`（内部 rapidjson），得到 DOM 后手工翻译成 WKB。

---

## 完整调用栈

### 查询（range 扫描）

```
SQL 谓词（ST_Contains / MBRContains / ST_Intersects ...）
  → Item_func_spatial_relation::val_int
  → get_mm_tree / get_func_mm_tree（range_analysis.cc）
       → 设置 gis_index_read_function = HA_READ_MBR_*（SQL 算子映射）
       → 设置 *inexact = true（MBR 需 recheck）
  → GeometryIndexRangeScanIterator::Read
       → file->ha_index_read_map(min_key=MBR, rkey_func_flag=HA_READ_MBR_*)
  → ha_innobase::index_read（HA_READ_MBR_* → PAGE_CUR_*）
  → row_search_mvcc → rtr_pcur_open → rtr_pcur_open_low
       → rtr_cur_search_with_match
            → cmp_dtuple_rec_with_gis
                 → cmp_gis_field → rtree_key_cmp
                      → mbr_intersect_cmp / mbr_contain_cmp / ...
       → rtr_pcur_move_to_next → rtr_pcur_getnext_from_path
  → 精确过滤（inexact 谓词重新 eval）
```

### 插入

```
ha_innobase::write_row → row_ins_sec_index_entry
  → btr_cur_search_to_nth_level（mode=PAGE_CUR_RTREE_INSERT）
       → rtr_cur_search_with_match（选面积增量最小的子树）
  → rtr_ins_enlarge_mbr（上推 MBR 扩大）
  → 页满 → rtr_page_split_and_insert
       → split_rtree_node（Quadratic Split：pick_seeds / pick_next）
```

---

## ★ 工程实现技法

### 1. MBR 作为"第 0 字段"的统一处理

R-Tree 页内记录的第一个字段固定是 MBR，这样 `cmp_dtuple_rec_with_gis` 能统一取 `tuple 第 0 字段` 与 `rec 第 0 字段` 比较，复用普通索引的比较框架。

### 2. SSN（Split Sequence Number）解决并发分裂

搜索走一条 DFS 路径，若中途页分裂，路径可能失效。InnoDB 用 `rtr_ssn`（只在分裂时递增）检测："搜索过程中 SSN 变了，说明路径失效，要重走"。这是 R-Tree 并发搜索的核心，类比 B-Tree 的乐观位置保存。

### 3. 父 MBR 的 WITHIN 断言

`btr0btr.cc` 里 `ut_a(... PAGE_CUR_WITHIN ...)` 用**断言**保证"父 MBR 包含子页第一个记录的 MBR"这一不变量——因为父 MBR 是子页并集，违反它说明分裂/上推逻辑有 bug。

### 4. WITHIN/CONTAINS 反向映射的历史包袱

`WITHIN` 映射到 `HA_READ_MBR_CONTAIN`、`CONTAINS` 映射到 `HA_READ_MBR_WITHIN`——存储引擎层的参数顺序与 SQL 层相反。这是历史兼容（MyISAM/InnoDB 约定）的产物，靠注释维持，是典型的"语义在注释里、不在类型系统里"。

---

## 可观测性

### 怎么看 R-Tree 索引

```sql
-- 建空间索引
CREATE TABLE t (g GEOMETRY NOT NULL, SPATIAL INDEX(g));

-- 看执行计划：R-Tree 走 "Using where"（因 inexact 需 recheck）
EXPLAIN SELECT * FROM t WHERE MBRContains(..., g);

-- 精确过滤的体现：EXPLAIN 里 FILTER 节点的条件含原始空间谓词
```

### 相关变量

| 变量 | 作用 |
|---|---|
| 无专用开关 | R-Tree 索引不依赖 optimizer_switch，只要有 SPATIAL INDEX 就走 |

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 新增空间函数 | `item_geofunc*` + `item_create.cc` 注册 |
| 新增空间关系算子 | `range_analysis.cc` 的 SQL 算子 → `HA_READ_MBR_*` 映射 + `gis0sea.cc` 的搜索模式 |
| 改分裂策略 | `gis0geo.cc` 的 `pick_seeds` / `pick_next`（Quadratic Split） |

### 已知缺陷

- **MBR 近似导致的假阳性**：R-Tree 扫描结果必须 recheck，大量几何精确相交时性能差。
- **WITHIN/CONTAINS 反向映射**是历史包袱，容易踩坑。
- **只有 MBR 谓词能走索引**：非 MBR 空间函数退化为全表扫。

### 社区边界澄清

- R-Tree 空间索引是社区版完整能力（InnoDB 实现，非云厂商私有）。
- 谓词锁（`LOCK_PREDICATE`）是 R-Tree 的并发控制，社区版有（见锁篇）。
- 空间函数（ST_*）是 OGC 标准的子集实现。

---

## 参考

**论文 / 理论**

- **Guttman《R-Trees: A Dynamic Index Structure for Spatial Searching》(SIGMOD 1984)** —— R-Tree 的原始论文，含 Quadratic Split 算法
- **OGC Simple Features** —— WKB/WKT、空间关系的标准定义

**官方文档**

- MySQL 8.0 Reference Manual, "Spatial Data Types" / "Optimizing Spatial Analysis" / "Spatial Indexes"

**源码出处**

- `storage/innobase/include/gis0rtree.h` / `gis0type.h` / `gis0geo.h` —— R-Tree 结构与分裂
- `storage/innobase/gis/gis0sea.cc` —— 邻域搜索
- `storage/innobase/gis/gis0geo.cc` —— Quadratic Split
- `sql/spatial.h` —— 几何类型体系
- `sql/range_optimizer/geometry_index_range_scan.cc` —— SPATIAL range 扫描
- `sql/range_optimizer/range_analysis.cc` —— SQL 算子 → MBR 映射

**相关文档**

- [`../../infra/lock/transactional/innodb_trx_lock.md`](../../infra/lock/transactional/innodb_trx_lock.md) —— R-Tree 的谓词锁
- [`json.md`](json.md) —— JSON 子系统（GeoJSON 复用其解析器）
