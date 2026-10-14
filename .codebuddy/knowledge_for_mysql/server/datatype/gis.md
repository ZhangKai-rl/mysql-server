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

**★ 一句话源码定位**（`sql/sql_optimizer.cc:1974`，★ R-Tree 不存数据这件事优化器自己知道）：

```cpp
          Cannot do full index scan on rtree index. It is not supported by
          Innodb as it's rtree index does not store data, but only the
          type POINT). Index scans on rtrees are probabaly not supported
```

**★ 模块划分**（`sql/gis/` 59 个头文件，官方 `@file` 定位）：

| 文件 | 定位 |
|---|---|
| `geometries.h` | "the geometry class hierarchy used by the server as the internal representation" |
| `geometries_cs.h` | Cartesian / Geographic 两套具体子类 |
| `wkb.h` | "the interface of the WKB parser ... and the parser for the internal geometry representation" |
| `rtree_support.h` | "functions that storage engines can call to do geometrical operations" ★ `rtr_mbr_t` 在此 |
| `relops.h` + `*_functor.h` | 关系运算（within/intersects/equals/crosses/overlaps/touches/covered_by…） |
| `setops.h` + `*_functor.h` | 集合运算（union/intersection/difference/symdifference） |
| `functor.h` | "Each GIS function is split in two: a functor class (for internal use) and a ..." |

> ⚠️ **不存在** `sql/gis/geometry_codec.h`。

### 版本演进

| 版本 | 变化 | WL |
|---|---|---|
| 5.7 | **InnoDB GEOMETRY 类型**（先有类型，才有索引与函数） | [WL#6455](https://dev.mysql.com/worklog/task/?id=6455) |
| 5.7 | **R-tree 索引的 DML 支持**（分裂 / 收缩，算法承自 MyISAM `rt_split.cc`） | [WL#6745](https://dev.mysql.com/worklog/task/?id=6745) |
| 5.7 | **谓词锁**：多维数据没有"下一个键"，gap lock 无从定义 | [WL#6609](https://dev.mysql.com/worklog/task/?id=6609) |
| 5.7 | **R-tree 搜索与定位**（整合 6745 + 6609 后整体发布） | [WL#6968](https://dev.mysql.com/worklog/task/?id=6968) |
| — | 多维（>2 维）索引：**规划过但未实现** | [WL#6618](https://dev.mysql.com/worklog/task/?id=6618)（未落地） |
| 8.0 | **新增 SRS（空间参考系）体系**：`mysql.st_spatial_reference_systems` 数据字典表，SRID 可被校验、`ST_Transform` 可用；几何类体系重构为 `gis::Geometry` + Cartesian/Geographic 双特化（Boost.Geometry 驱动） | — |

> ★ 8.0 的**几何类体系重构**在源码里没有对应的 WL 编号可查（`sql/gis/` 全目录 grep `WL#` 零命中）。从 `geometries.h:31` 的注释可看出重构的真正动因是 **Boost.Geometry 依赖编译期类型特征**区分坐标系，所以必须有两套具体子类——这是技术约束驱动的，不是规划出来的特性。
>
> ★ WL#6745 里明确说明了"复用 MyISAM"的决策：*"we will adopt all MyISAM's MRB (minimum bounding box) manipulation functions"*、*"The function we can reuse from MySQL/MyISAM is a set of APIs in sp_key.c. For example, sp_mbr_from_wkb()"*、*"rt_split.c"*。这印证了本篇「实现承自 MyISAM」的说法。

---

## 理论基础

### R-Tree：多维数据的平衡树

R-Tree 是 B-Tree 在**多维空间**上的推广（Guttman 1984）：

- B-Tree 按键值**一维排序**组织；R-Tree 按**空间位置**组织（每个节点存其子树所有对象的 MBR）
- 查询不是"找到 key"，而是"找到与查询区域相交的所有对象"（空间查询）

**⚠️ 源码里没有任何论文引用**（★ 核证结果）：

```
$ grep -rn 'Guttman' .          → 0 matching results
$ grep -rn 'WL *#?[0-9]\{4,\}' sql/gis/ storage/innobase/gis/   → 0 matching results
```
> ★ `storage/innobase/gis/gis0rtree.cc:28-32` 的文件头只写 `InnoDB R-tree interfaces / Created 2013/03/27 Allen Lai and Jimmy Yang`。**Guttman 1984 是背景知识，不是源码里能找到的东西**——写文档时必须区分"算法出处"与"MySQL 源码证据"。

**★ 但"R-Tree 是 InnoDB 的一等索引类型"在源码里是明确的**：

```cpp
/* storage/innobase/include/fil0fil.h:1211 —— ⚠️ 真值 17854，不是常见的 17856 */
/** R-tree node */
constexpr page_type_t FIL_PAGE_RTREE = 17854;
```
```cpp
/* storage/innobase/include/fil0fil.h:1314 —— ★ R-Tree 页与 B-Tree 页同属 "index page" 大类 */
inline bool fil_page_type_is_index(page_type_t page_type) {
  return page_type == FIL_PAGE_INDEX || page_type == FIL_PAGE_SDI ||
         page_type == FIL_PAGE_RTREE;
}
```
> ★ 同属 index page ⇒ 共享 buffer pool / 压缩 / doublewrite 的索引页路径。**差异发生在更细的分支点**，见「与 B-Tree 的页结构差异」一节。

**★ 页创建时的两处 R-Tree 专属分支**（`storage/innobase/btr/btr0btr.cc:318`）：

```cpp
  uint16_t page_create_type;
  if (dict_index_is_spatial(index)) {
    page_create_type = FIL_PAGE_RTREE;      /* ★ 差异点 1：页类型标记 */
  } else if (dict_index_is_sdi(index)) {
    page_create_type = FIL_PAGE_SDI;
  } else {
    page_create_type = FIL_PAGE_INDEX;
  }
  ...
  /* For Spatial Index, initialize the Split Sequence Number */
  if (dict_index_is_spatial(index)) {
    page_set_ssn_id(block, page_zip, 0, mtr);  /* ★ 差异点 2：R-Tree 页出生即带 SSN=0 */
  }
```

### MBR 近似：为什么需要"假阳性过滤"

R-Tree 索引只存 MBR（最小包围盒），**不存完整几何**。MBR 是近似：两个几何的 MBR 相交，**不代表几何本身真正相交**（例如两个对角分布的 L 形多边形，MBR 重叠但几何不相交）。

所以 R-Tree 扫描得到的是**候选集**，必须做**精确过滤**（recheck）——这是 GIS 索引与普通 B-Tree 索引最本质的差异（普通索引的 key 匹配是精确的，GIS 索引的 MBR 匹配是近似的）。

**★ "近似过滤"在源码里的本体：`MBR::disjoint()`**（`sql/spatial.h:124`）：

```cpp
  int disjoint(const MBR *mbr) const {
    /* The following should be safe, even if we compare doubles */
    return ((mbr->xmin > xmax) || (mbr->ymin > ymax) || (mbr->xmax < xmin) ||
            (mbr->ymax < ymin));
  }

  int intersects(const MBR *mbr) const { return !disjoint(mbr); }
```
> ★★ **4 次比较**完成"是否可能相交"的判定——这就是"假阳性过滤"的全部。它只保证 **disjoint ⇒ 一定不相交**（可以安全剪掉整棵子树），反过来 intersects 只是**必要条件**。
> ⚠️ 源码里**没有**"假阳性过滤"这个措辞，这是本文的概括；可引用的实际机制就是上面这段 + `sql/gis/rtree_support.h` 的 `mbr_intersect_cmp()` / `mbr_contain_cmp()`。

**★ 一个同名陷阱**（`sql/gis/rtree_support.h:64`，★ 源码自己打了警告）：

```cpp
/// Checks if one MBR covers another MBR.
///
/// @warning Despite the name, this function computes the covers relation, not
/// contains.
///
/// @retval true MBR a contains MBR b.
/// @retval false MBR a doesn't contain MBR b.
bool mbr_contain_cmp(const dd::Spatial_reference_system *srs, rtr_mbr_t *a,
                     rtr_mbr_t *b);
```
> ★ `mbr_contain_cmp` **实际算的是 covers 不是 contains**；`mbr_within_cmp` 同理算的是 covered_by。这与下面「WITHIN/CONTAINS 反向映射」是两个独立的命名坑，别混为一谈。

**★ 索引键里只有 MBR，没有几何本身**（★ 32 字节的权威计算）：

```cpp
/* storage/innobase/include/data0type.h:214 */
/** Used as GIS MBR column */
constexpr uint32_t DATA_GIS_MBR = 2048;
/** GIS MBR length*/
constexpr uint32_t DATA_MBR_LEN = SPDIMS * 2 * sizeof(double);   /* = 2*2*8 = 32 */
```
```cpp
/* storage/innobase/include/univ.i:649 —— ★ 2 维硬编码，且注释明说是 MyISAM 遗产 */
/* Dimension of spatial object we support so far. It has its root in
myisam/sp_defs.h. We only support 2 dimension data */
constexpr uint32_t SPDIMS = 2;
```
```cpp
/* storage/innobase/include/data0data.ic:165 —— ★ MBR → 索引键的序列化 */
static inline void dfield_write_mbr(dfield_t *field, const double *mbr) {
  field->ext = 0;
  for (uint i = 0; i < SPDIMS * 2; i++) {
    mach_double_write(static_cast<byte *>(field->data) + i * sizeof(double),
                      mbr[i]);
  }
  field->len = DATA_MBR_LEN;                 /* ★ = 32 */
}
```
> ⚠️ **不存在** `Field_geom::get_key_image()`。R-Tree 键由** InnoDB 侧**从 WKB 计算：`get_mbr_from_store()` → `dfield_write_mbr()`。

### Quadratic Split：R-Tree 的分裂策略

R-Tree 页满时如何分裂成两页？Guttman 提出三种：Exhaustive / **Quadratic（平方）** / Linear（线性）。InnoDB 用 Quadratic Split：

1. **pick_seeds**：选两个"合并后面积增量最大"的记录作为两组种子（让两组尽量分开）
2. **pick_next**：每次选"加入两组的面积增量差最大"的记录，分配到增量较小的一组

**★ 调用点**（`storage/innobase/gis/gis0rtree.cc:978`，★ 参数里的 `0, 2, 2` 就是策略与维度）：

```cpp
  total_data = page_get_data_size(page) + insert_size;
  first_rec_group = split_rtree_node(
      rtr_split_node_array, static_cast<int>(n_recs),
      static_cast<int>(total_data), static_cast<int>(insert_size), 0, 2, 2,
      &buf_pos, SPDIMS, static_cast<uchar *>(first_rec),
      cursor->index->rtr_srs.get());
                  /* 参数 0=quadratic 策略, 2=?, 2=维度, SPDIMS=2, rtr_srs=SRS */
```
> ★ 最后一个参数 `cursor->index->rtr_srs.get()` 值得注意：**分裂要按 SRS 算面积**——笛卡尔是矩形面积，地理是椭球面上的面积。`dict_index_t` 上挂着 `rtr_srs` 指针。
> ⚠️ 调用点附近**没有算法说明注释**（只有 `InnoDB R-tree interfaces` 的文件头）。

**★ 选路也按 SRS 算面积**（`storage/innobase/gis/gis0rtree.cc:1768`）：

```cpp
      switch (mode) {
        case PAGE_CUR_CONTAIN:
        case PAGE_CUR_INTERSECT:
          area += rtree_area_overlapping(index->rtr_srs.get(), range_mbr_ptr,
                                         field, DATA_MBR_LEN) / rec_area;
          break;
        case PAGE_CUR_DISJOINT:
          area += 1;
          area -= rtree_area_overlapping(index->rtr_srs.get(), range_mbr_ptr,
                                         field, DATA_MBR_LEN) / rec_area;
          break;
      }
```

---

## 一、几何类型与存储格式

### 几何类型体系

> ⚠️ **类名已整体重构**：8.0 是 `sql/gis/geometries.h` 里的 `gis::Geometry` 命名空间体系，**不再是 5.7 的 `sql/spatial.h` + `Gis_*` 命名**。
>
> **★★ 但这绝不是说旧类被删了——恰恰相反，它们全都还在，而且删不掉。** 原因见本节末尾「Boost.Geometry 绑定在旧类上」。
>
> **⚠️ 纠正一处常见误传**："`Gis_point` / `Gis_line_string` / `Gis_polygon` 在 8.0 已删除" 是**错的**。8.0.39 中它们**全部仍在**（`sql/spatial.h:1152 / 2099 / 2187`），且仍在被 `sql/item_geofunc.cc:765` 等处实际调用。

**★ 类型枚举**（`sql/gis/geometries.h:52`，★ 底层 `uint32`、取值与 WKB type code 完全一致）：

```cpp
enum class Geometry_type : std::uint32_t {
  kGeometry = 0,
  kPoint = 1,
  kLinestring = 2,
  kPolygon = 3,
  kMultipoint = 4,
  kMultilinestring = 5,
  kMultipolygon = 6,
  kGeometrycollection = 7
  // kMulticurve = 11,  kMultisurface = 12,  kCurve = 13,  kSurface = 14,
};
```
> ★ `kMulticurve` / `kCurve` 等抽象类型**被注释掉**——虽在 SFA-CA 里存在，MySQL 不实现。

**★ 坐标系枚举**（`geometries.h:69`，★ 这是外界常误传为 `Coordinate_type` 的那个）：

```cpp
enum class Coordinate_system {
  /// A Cartesian plane with the same unit in both directions.
  kCartesian = 0,
  /// An ellipsoidal system with longitude and latitude coordinates,
  /// both in the same unit.
  kGeographic = 1
};
```

**★ 类层次**（两层：先按 SFA-CA 抽象分层，再按坐标系特化）：

```
Geometry (抽象，SQL 中不可实例化)
├─ Point ──────────────── Cartesian_point / Geographic_point
├─ Curve (抽象)
│   └─ Linestring ────── Cartesian_linestring / Geographic_linestring
│        └─ Linearring ─ Cartesian_linearring / Geographic_linearring
├─ Surface (抽象)
│   └─ Polygon ───────── Cartesian_polygon / Geographic_polygon
└─ Geometrycollection ── Cartesian_geometrycollection / Geographic_geometrycollection
     ├─ Multipoint ───── Cartesian_multipoint / Geographic_multipoint
     ├─ Multicurve (抽象)
     │   └─ Multilinestring
     └─ Multisurface (抽象)
         └─ Multipolygon
```

> ★ 两套具体子类在 `sql/gis/geometries_cs.h` 定义。**为什么要分两套**？源码注释说得很清楚（`geometries.h:31`）：
> ```
> since Boost.Geometry depends on type traits to know if a geometry is in a
> Cartesian or geographic SRS, there are Cartesian and geographic
> specializations of each instantiable type in the SFA-CA.
> ```
> 即 **Boost.Geometry 靠类型特征（编译期）区分笛卡尔/地理**，所以必须有两套类型，不能只用一个 bool 成员。
> ★ 副产品：迭代器也必须是坐标系特定类型，所以**基类里没有标准的 `begin()`/`end()`**，改用 `operator[]` 作为坐标系无关的访问方式。

**★ 基类 `Geometry` 只有 5 个纯虚函数**（`geometries.h:93`）：

```cpp
class Geometry {
 public:
  virtual Geometry *clone() const = 0;
  virtual Geometry_type type() const = 0;
  virtual Coordinate_system coordinate_system() const = 0;
  virtual bool accept(Geometry_visitor *v) = 0;
  virtual bool is_empty() const = 0;
};
```
> ⚠️ 不存在 `coordinate_type()`，真名是 `coordinate_system()`。
> ★ `is_empty()` 与 `empty()` 语义不同（基类注释明说）：`is_empty` = "点集为空"（SFA-CA 6.1.2.2），`empty` = "没有子几何"。一个 GC 包含另一个空 GC 时，`empty()` 为 false 而 `is_empty()` 为 true。

**★ `Point` 的两个关键细节**（`geometries.h:147`）：

```cpp
  Point() : m_x(std::nan("")), m_y(std::nan("")) {}
  bool is_empty() const override { return (std::isnan(m_x) || std::isnan(m_y)); }
```
> ★ **8.0 用 NaN 表示"空点"**，不是单独的 bool 标志。

```cpp
  /// First coordinate (X or longitude).
  ///
  /// Geographic coordinates are in radians, positive to the East of
  /// Greenwich. Cartesian coordinates are in the SRSs length unit.
  double m_x;
```
> ★★★ **地理坐标系下 `m_x`/`m_y` 存的是弧度，不是度**！对外转度由 `dd::Spatial_reference_system::from_radians()` 负责（见 `get_mbr_from_store()`，`rtree_support.cc:393`）。

**★ 与 OGC 规范的一处关键分歧**（`geometries.h:264`，注释原文）：

```cpp
/// According to the SFA-CA, linestrings have a linear interpolation between
/// points. In MySQL, a linestring represents the geodesic. On a plane, this is
/// linear interpolation, but on an ellipsoid, it's the shortest path along the
/// ellipsoid surface.
```
> ★ **MySQL 的 linestring 表示测地线（geodesic）**，不是 OGC 的线性插值——椭球面上是最短路径。

**⚠️ 文件真名：`sql/gis/geometries.h`**（★ 收割时踩过的坑）：

```
不存在：sql/gis/gis_geometry.h、gis_point.h、gis_linestring.h、gis_polygon.h、
        gis_linearring.h、gis_geometrycollection.h、gis_multi*.h
真  名：sql/gis/geometries.h        ← 7 个几何类全在这一个文件
        sql/gis/geometries_cs.h     ← Cartesian_* / Geographic_* 两套特化
```
> ★ 命名模式是「按**操作**分文件（area / distance / intersects / within … 各一个 `*_functor.h`）+ 按**类型**只分 `geometries*.h`」。99 个文件里没有一个是按几何类型单独成文件的。

**★ `Linearring` 是独立类，不是 typedef**（`sql/gis/geometries.h:320`）：

```cpp
/// A ring-shaped linestring.
///
/// Linearring is a non-instantiable type in SQL. It is used to represent
/// polygon rings.
///
/// The start and end point of the linestring must be the same (assumed, but not
/// enforced, by the implementation).
class Linearring : public Linestring {
 public:
  bool accept(Geometry_visitor *v) override = 0;
  static Linearring *create_linearring(Coordinate_system coordinate_system);
};
```
> ★ 闭合性**"假定但不强制"** —— 非法环要靠 `ST_IsValid` 检查。

**★ 工厂方法：绕开"基类不知道坐标系"的通用手法**（★ 7 个可实例化类型各有一个）：

```cpp
  /// Creates a subclass of Linestring from a Coordinate_system
  static Linestring *create_linestring(Coordinate_system coordinate_system);
```
```cpp
  static Polygon *create_polygon(Coordinate_system coordinate_system);
  static Multipoint *create_multipoint(Coordinate_system coordinate_system);
  static Multipolygon *create_multipolygon(Coordinate_system coordinate_system);
```
> ⚠️ `Geometrycollection` 有**两个**工厂：`CreateGeometrycollection()`（老式大驼峰）和 `create_geometrycollection()`（新式）—— 8.0 重构期的历史残留，两个都还在。

**★ 坐标系特化的真身**（`sql/gis/geometries_cs.h:44`）：

```cpp
/// A Cartesian 2d point.
class Cartesian_point : public Point {
 public:
  Cartesian_point() = default;
  Cartesian_point(double x, double y) : Point(x, y) {}
  Coordinate_system coordinate_system() const override {
    return Coordinate_system::kCartesian;
  }
  Cartesian_point *clone() const override { return new Cartesian_point(*this); }
};

/// A geographic (ellipsoidal) 2d point.
class Geographic_point : public Point {
 public:
  Coordinate_system coordinate_system() const override {
    return Coordinate_system::kGeographic;
  }
  Geographic_point *clone() const override {
    return new Geographic_point(*this);
  }
};
```
```cpp
/// A Cartesian 2d linestring.
class Cartesian_linestring : public Linestring {
 protected:
  std::vector<Cartesian_point, Malloc_allocator<Cartesian_point>> m_points;
 public:
  Cartesian_linestring()
      : m_points(Malloc_allocator<Cartesian_point>(
            key_memory_Geometry_objects_data)) {}
                                    /* ★ PSI key，几何内存可观测 */
  iterator begin() noexcept { return m_points.begin(); }
  iterator end() noexcept { return m_points.end(); }
                                    /* ★★ begin()/end() 在这里才出现 */
```
> ★★ 印证 `geometries.h:38-41` 的说明：**因为迭代器类型必须绑定坐标系，公共基类无法提供 `begin()`/`end()`**，只能在特化类里给。
> ★ 内存用 `std::vector<..., Malloc_allocator<...>>` + PSI key `key_memory_Geometry_objects_data`。

**★★★ Boost.Geometry 绑定在旧类上——这就是旧类删不掉的原因**（`sql/gis_bg_traits.h:52`，★ 本节最重要的发现）：

```cpp
/* This file defines all boost geometry traits. */
#include <boost/geometry/core/access.hpp>
#include <boost/geometry/core/coordinate_system.hpp>
#include <boost/geometry/core/tags.hpp>
#include <boost/geometry/geometries/concepts/point_concept.hpp>
...
#include "sql/spatial.h"     /* ★★ include 的是 5.7 遗留头文件 */

namespace boost { namespace geometry { namespace traits {
template <> struct tag<Gis_point> {
  typedef boost::geometry::point_tag type;
};
template <> struct coordinate_system<Gis_point> {
  typedef boost::geometry::cs::cartesian type;   /* ★ 硬编码 cartesian */
};
template <std::size_t Dimension> struct access<Gis_point, Dimension> {
  static inline double get(Gis_point const &p) { return p.get<Dimension>(); }
  static inline void set(Gis_point &p, double const &value) { p.set<Dimension>(value); }
};
template <> struct tag<Gis_line_string> { typedef boost::geometry::linestring_tag type; };
template <> struct tag<Gis_polygon>    { typedef boost::geometry::polygon_tag type; };
template <> struct tag<Gis_polygon_ring> { typedef boost::geometry::ring_tag type; };
template <> struct tag<Gis_multi_point>     { typedef boost::geometry::multi_point_tag type; };
template <> struct tag<Gis_multi_line_string> { ... };
template <> struct tag<Gis_multi_polygon>   { ... };
}}}
```
> ★★★ **8.0 的 Boost.Geometry traits 全部注册在旧 `Gis_point` / `Gis_line_string` / `Gis_polygon` / `Gis_polygon_ring` / `Gis_multi_*` 上，不是新的 `gis::` 类**。这是"两套体系并存"的物理原因：新的 `gis::` 类负责 SQL 层语义与 SRS，旧的 `Gis_*` 类作为 **Boost.Geometry 适配器**继续活着。
> ★ 注意 `Gis_point` 的 `coordinate_system` 被硬编码为 `cartesian` —— 它是"无 SRS 的裸 WKB 视图"，地理语义由 `gis::Geographic_point` 承担。

**★ 旧 `Geometry` 与新 `gis::Geometry` 是并列关系，无继承**（★ 两处头文件互不 include）：

| | 旧 `Geometry` | 新 `gis::Geometry` |
|---|---|---|
| 位置 | `sql/spatial.h:213` | `sql/gis/geometries.h:100` |
| 定位 | 5.7 遗留的 **WKB 内存视图**（BG adapter） | 8.0 新架构的内部表示 |
| 有 SRID 吗 | 无（裸 WKB） | 无（`srid()` 不存在，由 `dd::Spatial_reference_system*` 单独传） |
| 谁在用 | Boost.Geometry、`ST_AsGeoJSON`、`Field_geom` 校验 | SQL 层几何运算 |
> ★ 两者通过 `sql/gis/geometries.cc` 转换。

**★ 两套类型的运行时分派**（`sql/gis/wkb.cc:389`）：

```cpp
  if (srs == nullptr || srs->is_cartesian()) {
      Wkb_parser<Cartesian_point, Cartesian_linestring, Cartesian_linearring,
                 Cartesian_polygon, Cartesian_geometrycollection,
                 Cartesian_multipoint, Cartesian_multilinestring,
                 Cartesian_multipolygon> parser(...);
  } else if (srs->is_geographic()) {
      Wkb_parser<Geographic_point, Geographic_linestring, Geographic_linearring,
                 Geographic_polygon, Geographic_geometrycollection,
                 Geographic_multipoint, Geographic_multilinestring,
                 Geographic_multipolygon> parser(...);
  }
```
> ★ **`srs == nullptr`（SRID 0）与 `is_cartesian()` 走同一套** —— SRID 0 就是笛卡尔。

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

> ⚠️ **"内部用位域 `Flags_t` 紧凑编码（bo:1, dim:2, ...）"是 5.7 的说法**——8.0.39 的 `sql/gis/` 里**不存在**这个 `Flags_t` 位域结构。8.0 的内部表示就是上面那套 `gis::Geometry` 对象树，序列化回 swkb 时按需计算长度。

**★ 权威格式常量**（`sql/spatial.h:50`）：

```cpp
const uint GEOM_DIM = 2;
const uint SRID_SIZE = 4;
const uint SIZEOF_STORED_DOUBLE = 8;
const uint POINT_DATA_SIZE = (SIZEOF_STORED_DOUBLE * 2);
const uint WKB_HEADER_SIZE = (1 + 4);
const uint GEOM_HEADER_SIZE = (SRID_SIZE + WKB_HEADER_SIZE);   /* = 9 */
```
> ⚠️ 真名 `SRID_SIZE`，**不存在 `SPATIAL_SRID_SIZE`**。

**★★ 写路径**（`sql/gis/wkb.cc:499`，`write_geometry()` —— 就两步，没有任何额外头部）：

```cpp
bool write_geometry(const dd::Spatial_reference_system *srs, Geometry &geometry,
                    String *str) {
  Wkb_size_visitor wkb_size;
  geometry.accept(&wkb_size);                       /* ① 先遍历算长度 */
  size_t geometry_size = sizeof(std::uint32_t) + wkb_size.size();  // SRID + WKB.
  str->set_charset(&my_charset_bin);
  str->length(geometry_size);
  char *buffer = &((*str)[0]);
  int4store(buffer, srs == nullptr ? 0 : srs->id());   /* ② 写 4 字节小端 SRID */
  buffer += sizeof(std::uint32_t);
  Wkb_visitor wkb(srs, buffer, wkb_size.size());
  geometry.accept(&wkb);                            /* ③ 追加纯 WKB */
  return false;
}
```

**★ 读路径**（`sql/gis/wkb.cc:429`，★ SRID 恒小端，与 WKB 自身的字节序标志无关）：

```cpp
bool parse_srid(const char *str, std::size_t length, srid_t *srid) {
  unsigned char *begin = pointer_cast<unsigned char *>(const_cast<char *>(str));
  if (length < sizeof(srid_t)) return true;
  *srid = uint4korr(begin);  // Always little-endian.
  return false;
}
```
```cpp
  srid_t srid;
  if (parse_srid(str->ptr(), str->length(), &srid)) { ... }
  if (srid != 0 && fetcher.acquire(srid, srs)) return true;
  /* ★ SRID=0 是特例：不查 SRS 字典，直接当笛卡尔 */
  *geometry = gis::parse_wkb(thd, *srs, str->ptr() + sizeof(srid_t),
                             str->length() - sizeof(srid_t), true);
  /*                          ★ 跳过 4 字节 SRID        ★ ignore_axis_order=true */
```

**★★ 存储格式与标准 WKB 的唯一差别**（`sql/gis/wkb.h:61`，官方注释）：

> *"If ignore_axis_order is true, the string is assumed to be on the geometry storage format, not general WKB. The storage format differs from ordinary WKB by having a **fixed x=longitude, y=latitude mapping**."*

> ★ 即：内部格式固定轴序 x=经度 / y=纬度，**不理会 SRS 定义的轴序**。

**★★★ 存储引擎侧的权威实现** `get_mbr_from_store()`（`sql/gis/rtree_support.cc:354`，★ 这条注释是"4 字节小端 SRID"最硬的证据）：

```cpp
/// The geometry is expected to be on the storage format (SRID + WKB). ...
/// The SRID of the SRS parameter must match the SRID stored in the first four
/// bytes of the geometry string.
int get_mbr_from_store(const dd::Spatial_reference_system *srs, const uchar *store,
                       uint size, uint n_dims, double *mbr, gis::srid_t *srid) {
  if (srid != nullptr) *srid = uint4korr(store);
  std::unique_ptr<gis::Geometry> g = gis::parse_wkb(
      current_thd, srs, pointer_cast<const char *>(store) + sizeof(gis::srid_t),
      size - sizeof(gis::srid_t), true);
  if (srs == nullptr || srs->is_cartesian()) {
    gis::Cartesian_box box;
    gis::box_envelope(g.get(), srs, &box);
    mbr[0] = box.min_corner().x();  ...
  } else {
    gis::Geographic_box box;
    gis::box_envelope(g.get(), srs, &box);
    mbr[0] = srs->from_radians(box.min_corner().x());   /* ★★ 弧度 → SRS 单位 */
    ...
  }
  if (std::isnan(mbr[0])) {
    // The geometry is empty, so there is no bounding box. Return a box that
    // covers the entire domain.
    mbr[0] = std::numeric_limits<double>::lowest();
    mbr[1] = std::numeric_limits<double>::max();
    ...
  }
```
> ★★ 三点：① **"SRID stored in the first four bytes"** 实锤；② 地理分支要 `from_radians()` 把内部弧度转回 SRS 单位（度）；③ **空几何的 MBR 是全域 `±DBL_MAX`，不是 0/0/0/0**。

> ⚠️ **另一处需要纠正的误传**："`Geometry::create_from_wkb()` / `as_wkb()` 在 8.0 已不存在" 是**错的**。它们**仍在**（`sql/spatial.h:600 / :604`），且仍被实际调用：
> ```cpp
> /* sql/item_geofunc.cc:764 —— 旧 5.7 路径并未被清理 */
>   Geometry_buffer buff;
>   Geometry *g = Geometry::create_from_wkb(current_thd, &buff, wkb->ptr(),
>                                           wkb->length(), &temp, true);
> ```
> ★ 准确的说：**新路径**是 `gis::parse_wkb()` / `gis::write_geometry()`；**旧路径仍在**，服务于 Boost.Geometry adapter 与 GeoJSON 转换。

**★ 8.0 新解析器的真身：`Wkb_parser` 是 8 参数模板**（`sql/gis/wkb.cc:104`）：

```cpp
template <typename Point_t, typename Linestring_t, typename Linearring_t,
          typename Polygon_t, typename Geometrycollection_t,
          typename Multipoint_t, typename Multilinestring_t,
          typename Multipolygon_t>
class Wkb_parser {
 private:
  uchar *m_begin;  uchar *m_end;
  Coordinate_system m_coordinate_system;
  double m_angular_unit;      /* ★ SRS 角单位 → 弧度 */
  double m_prime_meridian;    /* ★ SRS 本初子午线 → Greenwich */
  bool m_positive_north;  bool m_positive_east;
  bool m_swap_axes;           /* ★ SRS 定义为 lat-long 时置位 */
  THD *m_thd;
```
> ★ 一次实例化绑定**全套** `Cartesian_*` 或全套 `Geographic_*`（见 `wkb.cc:391` 的两处分派）。
> ⚠️ 与旧 `Geometry::wkb_parser`（`sql/spatial.h:384`，嵌套类）**同名但不同物**——两个都在。

**★★ 磁盘格式 vs 内存格式的完整差异**（`sql/gis/wkb.cc:132`，★ 源码注释逐条写明）：

```cpp
  double transform_x(double x) {
    switch (m_coordinate_system) {
      case Coordinate_system::kCartesian:
        // The on-disk and in-memory format is x in SRS direction and unit.
        break;
      case Coordinate_system::kGeographic:
        // The on-disk format is x = longitude, in the SRS direction and unit,
        // and with the SRS meridian.
        // The in-memory format is x = longitude (Easting) in radians with the
        // meridian at Greenwich.
        if (!m_positive_east) x *= -1.0;
        x += m_prime_meridian;  // Both are in the SRS angular unit
        x *= m_angular_unit;    // Convert to radians
        break;
```
```cpp
  double transform_y(double y) {
    ...
      case Coordinate_system::kGeographic:
        // The in-memory format is y = latitude (Northing) in radians.
        if (!m_positive_north) y *= -1.0;
        y *= m_angular_unit;  // Convert to radians
                                    /* ★ 纬度无子午线偏移 */
```

| | 磁盘（存储格式） | 内存 |
|---|---|---|
| 单位 | SRS 的角单位（如度） | **恒弧度** |
| 本初子午线 | SRS 定义的 | **归化到 Greenwich** |
| 轴向 | SRS 定义（可 lat-long） | **恒 long-lat（x=经度）** |

**★ 逐字段解析：字节布局的"读侧"证据**（`sql/gis/wkb.cc:165`）：

```cpp
  Byte_order parse_byte_order() {
    if (m_begin + 1 > m_end) throw std::exception();
    switch (*(m_begin++)) {
      case 0: return Byte_order::XDR;      /* 大端 */
      case 1: return Byte_order::NDR;      /* 小端 */
    }
    throw std::exception();
  }

  std::uint32_t parse_uint32(Byte_order bo) {
    if (bo == Byte_order::NDR) i = uint4korr(m_begin);
    else i = load32be(m_begin);
    m_begin += 4;  return i;
  }

  Geometry_type parse_geometry_type(Byte_order bo) {
    std::uint32_t wkb_type = parse_uint32(bo);
    Geometry_type type = static_cast<Geometry_type>(wkb_type);
    if (!is_valid_type_or_subtype(type, Geometry_type::kGeometry))
      throw std::exception();
    return type;
  }

  Point_t parse_point(Byte_order bo) {
    double x = parse_double(bo);
    double y = parse_double(bo);
    if (!std::isfinite(x) || !std::isfinite(y)) throw std::exception();
                                    /* ★ 拒收 NaN/Inf */
    if (m_swap_axes)
      return Point_t(transform_x(y), transform_y(x));
    else
      return Point_t(transform_x(x), transform_y(y));
  }
```
> ★ 一个 Point 的裸 WKB = 1(字节序) + 4(类型) + 8(x) + 8(y) = **21 字节**；swkb 再加 4 字节 SRID = **25 字节**。
> ★ `enum class Byte_order : std::uint8_t { XDR = 0, NDR = 1 }`（`wkb.cc:48`）。

**★ 子类型判定（GEOMETRY 列能存任意类型的机制）**（`sql/gis/wkb.cc:78`）：

```cpp
static bool is_subtype_of(Geometry_type sub, Geometry_type super) {
  return (super == Geometry_type::kGeometry || sub == super ||
          (super == Geometry_type::kGeometrycollection &&
           (sub == Geometry_type::kMultipoint ||
            sub == Geometry_type::kMultilinestring ||
            sub == Geometry_type::kMultipolygon)));
}
```
> ★ `kGeometry(0)` 作 super 时接受任何类型；`Multi*` 是 `Geometrycollection` 的子类型（SFA-CA 语义）——所以 `GEOMCOLLECTION` 列能存 `Multi*`。

**★ 解析后的两道后处理**（`sql/gis/wkb.cc:455`）：

```cpp
  // Flip polygon rings so that the exterior ring is counter-clockwise and
  // interior rings are clockwise.
  gis::Ring_flip_visitor rfv(semi_major, semi_minor);
  (*geometry)->accept(&rfv);
  ...
  gis::Coordinate_range_visitor crv(*srs);
  if ((*geometry)->accept(&crv)) {
    if (crv.longitude_out_of_range()) {
      my_error(ER_GEOMETRY_PARAM_LONGITUDE_OUT_OF_RANGE, MYF(0), func_name,
               crv.coordinate_value(), (*srs)->from_radians(-M_PI),
               (*srs)->from_radians(M_PI));
      return true;
    }
    if (crv.latitude_out_of_range()) { ... -M_PI_2, M_PI_2 ... }
```
> ★ ① 环方向规范化（外环逆时针、内环顺时针）；② 经纬度范围校验 `[-π,π]` / `[-π/2,π/2]`。

**★ 写侧：两遍扫描 + 恒小端**（`sql/gis/wkb_visitor.cc:30`）：

```cpp
bool Wkb_visitor::visit(Point *pt) {
  *m_wkb_current_position = '\x01';  // Little-endian
  m_wkb_current_position++;
  int4store(m_wkb_current_position, static_cast<uint32>(pt->type()));
  m_wkb_current_position += sizeof(std::uint32_t);
  if (m_srs != nullptr && m_srs->is_geographic()) {
    float8store(m_wkb_current_position,
                m_srs->from_normalized_longitude(pt->x()));
    m_wkb_current_position += sizeof(double);
    float8store(m_wkb_current_position,
                m_srs->from_normalized_latitude(pt->y()));
    ...
```
> ★★ **MySQL 写出的 WKB 恒为小端**（硬编码 `'\x01'`），与输入字节序无关。写出时用 `from_normalized_longitude/latitude` 把内存弧度转回 SRS 单位——与读侧 `transform_x/y` 互为逆运算。

**★ `Field_geom`：几何值进记录就是一个 BLOB**（`sql/field.h:3929`，⚠️ **没有** pack/unpack）：

```cpp
class Field_geom final : public Field_blob {
 private:
  const std::optional<gis::srid_t> m_srid;   /* ★ 8.0 新增：列级 SRID 约束 */
 public:
  enum geometry_type geom_type;
  enum ha_base_keytype key_type() const final { return HA_KEYTYPE_VARBINARY2; }
  enum_field_types type() const final { return MYSQL_TYPE_GEOMETRY; }
  bool match_collation_to_optimize_range() const final { return false; }
```
> ★ 完全继承 `Field_blob`，只 override 了 `store*` / `sql_type` / `is_equal`。**几何列在记录里 = 长度前缀 + swkb 字节串**。
> ⚠️ **不存在** `Field_geom::pack()` / `unpack()`。

```cpp
/* sql/field.cc:7519 —— ★ 写入校验 5 条 */
type_conversion_status Field_geom::store_internal(const char *from,
                                                  size_t length,
                                                  const CHARSET_INFO *cs) {
  // Check that the given WKB
  // 1. is at least 13 bytes long (length of GEOMETRYCOLLECTION EMPTY)
  // 2. isn't marked as bad geometry data
  // 3. isn't shorter than empty geometrycollection
  // 4. is a valid geometry type
  // 5. is well formed
  if (length < 13 ||
      from == Geometry::bad_geometry_data.ptr() ||
      length < SRID_SIZE + WKB_HEADER_SIZE + sizeof(uint32) ||
      !Geometry::is_valid_geotype(uint4korr(from + SRID_SIZE + 1)) ||
      !Geometry::is_well_formed(from, length, ...)) {
    Field_blob::reset();
    my_error(ER_CANT_CREATE_GEOMETRY_OBJECT, MYF(0));
    return TYPE_ERR_BAD_VALUE;
  }
```
> ★ `uint4korr(from + SRID_SIZE + 1)` —— 精确定位"跳过 4 字节 SRID 和 1 字节字节序后的 4 字节类型码"，是字节布局的**直接证据**。
> ★ 最小 13 字节 = 4(SRID) + 5(header) + 4(`GEOMETRYCOLLECTION EMPTY` 的 num_geoms=0)。

```cpp
/* include/field_types.h:87 —— ⚠️ 不在 mysql_com.h */
  MYSQL_TYPE_GEOMETRY = 255
```

### 字段映射

| | 空间列 | JSON 列 |
|---|---|---|
| 字段类 | `Field_geom : Field_blob` | `Field_json : Field_blob` |
| `type()` | `MYSQL_TYPE_GEOMETRY` | `MYSQL_TYPE_JSON` |
| 存储 | 定长头 + WKB | JSONB 文档树 |
| 校验 | 存入前 WKB 合法性校验 | JSON 语法校验 |
| 索引 | **R-Tree 空间索引** | 生成列/函数索引/多值索引 |
| InnoDB 映射 | `DATA_GEOMETRY` | `DATA_BLOB` |

**★ `Field_geom` 真身**（`sql/field.h:3929`，★ 注意它是 `final` 且继承 `Field_blob`——几何列物理上就是 BLOB）：

```cpp
class Field_geom final : public Field_blob {
 private:
  const std::optional<gis::srid_t> m_srid;   /* ★ 列级 SRID 约束；nullopt = 不限制 */
 public:
  enum geometry_type geom_type;              /* ★★ 字段映射的核心成员 */
  enum_field_types type() const final { return MYSQL_TYPE_GEOMETRY; }
  geometry_type get_geometry_type() const final { return geom_type; }
  std::optional<gis::srid_t> get_srid() const { return m_srid; }
};
```

**★ SQL 类型枚举**（`sql/field.h:718`，★ 与 `gis::Geometry_type` 的 1..7 完全对齐）：

```cpp
  enum geometry_type {
    GEOM_GEOMETRY = 0,  GEOM_POINT = 1,  GEOM_LINESTRING = 2,  GEOM_POLYGON = 3,
    GEOM_MULTIPOINT = 4, GEOM_MULTILINESTRING = 5, GEOM_MULTIPOLYGON = 6,
    GEOM_GEOMETRYCOLLECTION = 7
  };
```
> ★★ **所有 7 种几何类型在 Field 层统一是 `MYSQL_TYPE_GEOMETRY`**，靠 `geom_type` 区分——这就是"字段映射"的全部内容。

**★ `SHOW CREATE TABLE` / DD 怎么展示**（`sql/field.cc:7475`）：

```cpp
void Field_geom::sql_type(String &res) const {
  switch (geom_type) {
    case GEOM_POINT:   res.set(STRING_WITH_LEN("point"), cs); break;
    case GEOM_LINESTRING: res.set(STRING_WITH_LEN("linestring"), cs); break;
    ...
    case GEOM_GEOMETRYCOLLECTION:
      res.set(STRING_WITH_LEN("geomcollection"), cs); break;   /* ★ 不是 geometrycollection */
    default: res.set(STRING_WITH_LEN("geometry"), cs);
  }
}
```
> ★ 注意是 **`geomcollection`**（缩写），不是 `geometrycollection`。

**★★ 写路径的校验链**（`sql/field.cc:7539`，★ 五道检查 + SRID 校验）：

```cpp
  if (length < 13 ||                                                   // 1 最短 GEOMETRYCOLLECTION EMPTY
      from == Geometry::bad_geometry_data.ptr() ||                     // 2 坏数据标记
      length < SRID_SIZE + WKB_HEADER_SIZE + sizeof(uint32) ||         // 3
      !Geometry::is_valid_geotype(uint4korr(from + SRID_SIZE + 1)) ||  // 4 合法类型
      !Geometry::is_well_formed(from, length,
                                geometry_type_to_wkb_type(geom_type),
                                Geometry::wkb_ndr)) {                  // 5 结构良好
    my_error(ER_CANT_CREATE_GEOMETRY_OBJECT, MYF(0));
    return TYPE_ERR_BAD_VALUE;
  }
  if (get_srid().has_value()) {                    /* ★ 列级 SRID 约束 */
    gis::srid_t geometry_srid = uint4korr(from);
    if (geometry_srid != get_srid().value()) {
      my_error(ER_WRONG_SRID_FOR_COLUMN, MYF(0), field_name, ...);
      return TYPE_ERR_BAD_VALUE;
    }
  }
```
> ★ **`Geometry::wkb_ndr`** —— 内部格式**强制小端**，不接受大端 WKB。
> ★ `from + SRID_SIZE` 是字节序字节，`+1` 是 type code。
> ★ 数字/小数**不能隐式转几何**——`store(double)` / `store(longlong)` / `store_decimal()` 三个重载全部直接报 `ER_CANT_CREATE_GEOMETRY_OBJECT`。
> ⚠️ 读路径**没有 `val_str()` 覆写**——沿用 `Field_blob::val_str()`，吐出的就是原始 swkb 字节。

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

**★★ 一个极易踩的坑：SQL 层 `MBR` 与 InnoDB 侧 `rtr_mbr_t` 字段顺序不同**！

```cpp
/* sql/spatial.h:83 —— SQL 层 */
struct MBR {
  double xmin, ymin, xmax, ymax;          /* ★ xmin, ymin, xmax, ymax */
```
```cpp
/* sql/gis/rtree_support.h:39 —— InnoDB 侧 */
typedef struct rtr_mbr {
  /// minimum on x
  double xmin;
  /// maximum on x
  double xmax;
  /// minimum on y
  double ymin;
  /// maximum on y
  double ymax;
} rtr_mbr_t;                              /* ★ xmin, xmax, ymin, ymax */
```
> ★★ 跨层拷贝**必须重排**，走 `rtr_write_mbr()` / `rtr_read_mbr()` 转换，不能直接 memcpy。
> ★ 参照 `gis0rtree.cc:1694` 的 `double coords[4]{xmin, xmax, ymin, ymax}`。

**★ SQL 层 `MBR` 的几个设计细节**（`sql/spatial.h:83`）：

```cpp
  MBR() {
    xmin = ymin = DBL_MAX;
    xmax = ymax = -DBL_MAX;      /* ★ "空 MBR"：min=+MAX, max=-MAX（不是 0） */
  }

  void add_xy(double x, double y) {
    /* Not using "else" for proper one point MBR calculation */
    if (x < xmin) xmin = x;
    if (x > xmax) xmax = x;
    if (y < ymin) ymin = y;
    if (y > ymax) ymax = y;      /* ★★ 刻意不用 else —— 单点也能同时更新 min 和 max */
  }
```
> ★ 默认构造是 min=+DBL_MAX / max=-DBL_MAX，这样 `add_xy` 第一个点就能同时正确收敛 min 和 max。

**★ 32 字节的两个组成常量**（★ 真名核查，都在 InnoDB 侧）：

| 常量 | 值 | 文件:行 |
|---|---|---|
| `SPDIMS` | 2 | `storage/innobase/include/univ.i:651` |
| `SPLEN` | 8 | `storage/innobase/include/gis0geo.h:36`（MyISAM 遗留） |
| `DATA_MBR_LEN` | **32** | `storage/innobase/include/data0type.h:216` |
| `DATA_GIS_MBR` | 2048 | `storage/innobase/include/data0type.h:214` |

**★ 8.0 新增的"计算用 MBR"：`gis::Box`**（`sql/gis/box.h:38`，★ 同样二分为坐标系）：

```cpp
/// A 2d box with sides parallel to the coordinate system grid.
///
/// Used by computations on minimum bounding boxes (MBRs).
class Box {
 public:
  virtual Coordinate_system coordinate_system() const = 0;
  virtual Point const &min_corner() const = 0;
  virtual Point &min_corner() = 0;
  virtual Point const &max_corner() const = 0;
  virtual Point &max_corner() = 0;
};
```
```cpp
/// A Geographic 2d box.
///
/// Unlike polygons, the sides of the box are not the shortest distance between
/// the endpoints. A box side will follow the latitude line, while a linestring
/// (or polygon segment) between the same points won't.
class Geographic_box : public Box {
```
> ★★ `gis::Box` **不用于索引键**（索引键用 `rtr_mbr_t` / `DATA_MBR_LEN`），只用于 `gis::` 层的几何计算。
> ★★ 地理 box 的边**沿纬线走，不是测地线** —— 所以地理 MBR 是**保守包围**（比真正的最小包围盒略大）。
> ⚠️ **疑似源码 bug**：`sql/gis/box.h:122` 的非 const 版 `min_corner()` 返回的是 `m_max_corner`（对照 `Cartesian_box:89` 返回 `m_min_corner`）。

**★ 空几何的 MBR 是 NaN**（`sql/gis/mbr_utils.h:54`，★ 已知坑）：

```cpp
/// Checks if an MBR is empty.
///
/// By default, BG box coordinates are NaN. If a geometry is empty, its box will
/// have all NaN coordinates.
bool mbr_is_empty(Box const &mbr);
```

### 与 B-Tree 的页结构差异

| 维度 | B-Tree | R-Tree |
|---|---|---|
| 节点存什么 | 索引 key（一维有序） | **子节点 MBR**（多维包围盒） |
| 记录顺序 | 按 key 排序 | 无排序，由分裂算法决定 |
| 方向性 | ASC/DESC | 无方向 |
| 分裂 | 中位数分裂 | Quadratic Split |

**★★★ 但物理层面几乎没有差别**——这是最容易想反的一点。逐项核实：

**① 页类型**（`storage/innobase/include/fil0fil.h:1208`）：

```cpp
/** B-tree node */
constexpr page_type_t FIL_PAGE_INDEX = 17855;
/** R-tree node */
constexpr page_type_t FIL_PAGE_RTREE = 17854;      /* ★ = INDEX - 1 */
```
```cpp
/* btr0btr.cc:331 —— 建页时的分派 */
  if (dict_index_is_spatial(index))      page_create_type = FIL_PAGE_RTREE;
  else if (dict_index_is_sdi(index))     page_create_type = FIL_PAGE_SDI;
  else                                   page_create_type = FIL_PAGE_INDEX;
```

**② 不存在 `PAGE_RTREE_LEVEL` 之类新页头字段**——完全复用 `PAGE_LEVEL`（`page0types.h:84`），叶子同样是 0。

**③ R-Tree 唯一"额外"的字段不在 page header，而是抢了 FIL header 的 8 字节**（`fil0types.h:103`）：

```cpp
/** This overloads FIL_PAGE_FILE_FLUSH_LSN for RTREE Split Sequence Number */
constexpr uint32_t FIL_RTREE_SPLIT_SEQ_NUM = FIL_PAGE_FILE_FLUSH_LSN;
```
> ★★ 这就是 SSN（Split Sequence Number）。它是并发搜索检测"我路径上的页被分裂过"的关键（见 rtree.md）。
> ★ 副作用（`buf0buf.cc:5149` 注释）：**R-Tree 页不参与 punch-hole 透明压缩**，因为那 8 字节被占用了。

**④ 记录物理格式与 B-Tree 完全共用**（COMPACT/DYNAMIC、node pointer 同样 4 字节子页号）。差异只在记录**内容**：

| | B-Tree 二级索引 | R-Tree（SPATIAL） |
|---|---|---|
| 非叶记录 | key 字段 + **PK 字段** + 子页号(4B) | **MBR(32B) + 子页号(4B)**，**无 PK** |
| 叶记录 | key + PK | MBR(32B) + PK 字段 |

```cpp
/* storage/innobase/include/dict0dict.ic:727 —— 非叶只有 2 个字段 */
  if (dict_index_is_spatial(index)) {
    /* For spatial index, on non-leaf page, we have only
    2 fields(mbr+page_no). So, except page no field,
    there's one field there. */
    return (DICT_INDEX_SPATIAL_NODEPTR_SIZE);
  }
```

**⑤ 真正的差异在比较函数与搜索算法**（`page0cur.cc:397`）：

```cpp
  /* If the mode is for R-tree indexes, use the special MBR
  related compare functions */
  if (dict_index_is_spatial(index) && mode > PAGE_CUR_LE) {
    /* For leaf level insert, we still use the traditional
    compare function for now */
    if (mode == PAGE_CUR_RTREE_INSERT && page_is_leaf(page)) {
      mode = PAGE_CUR_LE;       /* ★★ 叶层插入退化成普通 LE */
    } else {
      rtr_cur_search_with_match(block, (dict_index_t *)index, tuple, mode,
                                cursor, rtr_info);
      return;
    }
  }
```
> ★ 分派条件就是 `mode > PAGE_CUR_LE`（B-Tree 只用到 1..4，≥7 即 R-Tree）。

**⑥ 三处"R-Tree 被排除的优化"**（★ 都是 B-Tree 有而 R-Tree 没有的）：

```cpp
/* page0cur.cc:380 —— adaptive hash 快捷路径 */
  if (page_is_leaf(page) && (mode == PAGE_CUR_LE) &&
      !dict_index_is_spatial(index) && ...
```
```cpp
/* page0cur.cc:1944 —— 顺序插入方向统计 */
  if (!dict_index_is_spatial(index)) {
    ... page_header_set_field(page, page_zip, PAGE_DIRECTION, PAGE_NO_DIRECTION);
        page_header_set_field(page, page_zip, PAGE_N_DIRECTION, 0);
  }
```
> ★ R-Tree **不维护 `PAGE_DIRECTION` / `PAGE_N_DIRECTION`**（顺序插入优化）——MBR 插入位置与记录顺序无关。

**⑦ 一个放宽**：B-Tree 里非叶 node pointer 必须严格递增，R-Tree **允许两个 MBR 完全相同的 child**（`page0page.cc:2295`）：

```cpp
      /* For spatial index, on nonleaf leavel, we
      allow recs to be equal. */
      bool rtr_equal_nodeptrs =
          (ret == 0 && dict_index_is_spatial(index) && !page_is_leaf(page));
```

> ★★ **一句话**：R-Tree 与 B-Tree 在物理层是"同一个引擎的两套索引"，差异集中在**比较函数 + 搜索算法 + 分裂策略**，不在页结构。

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

**★ SSN 的读写**（`storage/innobase/include/page0page.ic:93`）：

```cpp
/** Returns the RTREE SPLIT SEQUENCE NUMBER (FIL_RTREE_SPLIT_SEQ_NUM). */
static inline node_seq_t page_get_ssn_id(const page_t *page) {
  return (static_cast<node_seq_t>(
      mach_read_from_8(page + FIL_RTREE_SPLIT_SEQ_NUM)));
}
```
> ★ `node_seq_t` 是 `uint32_t`（`gis0type.h:57`）——虽然槽位有 8 字节，但只用了 4 字节。

**★ 非叶记录构造：`rtr_index_build_node_ptr`**（`storage/innobase/gis/gis0rtree.cc:131`，★ 与 B-Tree 的关键差异）：

```cpp
dtuple_t *rtr_index_build_node_ptr(const dict_index_t *index,
                                   const rtr_mbr_t *mbr, const rec_t *rec,
                                   page_no_t page_no, mem_heap_t *heap) {
  ut_ad(dict_index_is_spatial(index));
  n_unique = DICT_INDEX_SPATIAL_NODEPTR_SIZE;    /* ★ = 1 */

  tuple = dtuple_create(heap, n_unique + 1);

  /* For rtree internal node, we need to compare page number fields. */
  dtuple_set_n_fields_cmp(tuple, n_unique + 1);
                    /* ★★ B-Tree 节点指针比较时不含 child page no；
                       R-Tree 必须含，否则同 MBR 的两个子节点无法区分 */

  /* Write page no field */
  buf = static_cast<byte *>(mem_heap_alloc(heap, 4));
  mach_write_to_4(buf, page_no);                 /* ★ child page no 定长 4 字节 */
  field = dtuple_get_nth_field(tuple, n_unique);
  dfield_set_data(field, buf, 4);
  dtype_set(dfield_get_type(field), DATA_SYS_CHILD, DATA_NOT_NULL, 4);

  /* Set mbr as index entry data */
  field = dtuple_get_nth_field(tuple, 0);
  buf = static_cast<byte *>(mem_heap_alloc(heap, DATA_MBR_LEN));  /* ★ 32 字节 */
  rtr_write_mbr(buf, mbr);
  dfield_set_data(field, buf, DATA_MBR_LEN);
```
```cpp
/* storage/innobase/include/dict0dict.h:712 */
/** The number of fields in the nonleaf page of spatial index, except
the page no field. */
constexpr uint32_t DICT_INDEX_SPATIAL_NODEPTR_SIZE = 1;
```
> ⚠️ **不存在** `RTR_NODE_REC_SIZE`。

**★ R-Tree 专属搜索模式（B-Tree 只用到 1..6）**（`storage/innobase/include/page0types.h:186`）：

```cpp
  /* These search mode is for search R-tree index. */
  PAGE_CUR_CONTAIN = 7,
  PAGE_CUR_INTERSECT = 8,
  PAGE_CUR_WITHIN = 9,
  PAGE_CUR_DISJOINT = 10,
  PAGE_CUR_MBR_EQUAL = 11,
  PAGE_CUR_RTREE_INSERT = 12,
  PAGE_CUR_RTREE_LOCATE = 13,
  PAGE_CUR_RTREE_GET_FATHER = 14
};
```
```cpp
/* storage/innobase/include/gis0rtree.h:56 */
/* Define it for rtree search mode checking. */
static inline bool RTREE_SEARCH_MODE(page_cur_mode_t mode) {
  return mode >= PAGE_CUR_CONTAIN && mode <= PAGE_CUR_RTREE_GET_FATHER;
}
```

**★ 索引标志与能力位**（★ 真名核查）：

```cpp
/* storage/innobase/include/dict0mem.h:106 —— ⚠️ 不是 DICT_RTREE */
/** SPATIAL index; can't be combined with the other flags */
constexpr uint32_t DICT_SPATIAL = 64;
```
```cpp
/* storage/innobase/include/dict0dict.ic:126 */
static inline ulint dict_index_is_spatial(const dict_index_t *index) {
  return (index->type & DICT_SPATIAL);
}
```
```cpp
/* include/my_base.h:95 —— ⚠️ 属于 enum ha_key_alg，不是 ha_rkey_function */
enum ha_key_alg {
  HA_KEY_ALG_SE_SPECIFIC = 0,
  HA_KEY_ALG_BTREE = 1,   /* B-tree. */
  HA_KEY_ALG_RTREE = 2,   /* R-tree, for spatial searches */
  HA_KEY_ALG_HASH = 3,    /* HASH keys (HEAP, NDB). */
  HA_KEY_ALG_FULLTEXT = 4 /* FULLTEXT. */
};
```
```cpp
/* sql/handler.h:290 */
#define HA_CAN_RTREEKEYS (1 << 17)
```
> ★ `HA_CAN_RTREEKEYS` 在 `ha_innodb.cc:2978` 声明。

**★ redo 也是分叉的**（`storage/innobase/page/page0page.cc:262`）：

```cpp
    case FIL_PAGE_INDEX:
      type = comp ? MLOG_COMP_PAGE_CREATE : MLOG_PAGE_CREATE;
      break;
    case FIL_PAGE_RTREE:
      type = comp ? MLOG_COMP_PAGE_CREATE_RTREE : MLOG_PAGE_CREATE_RTREE;
      break;
```
```cpp
/* storage/innobase/log/log0recv.cc:2165 —— 回放端 */
      page_parse_create(block, type == MLOG_COMP_PAGE_CREATE_RTREE,
                        FIL_PAGE_RTREE);
```

**★★ 两处"R-Tree 被完全关闭的 B-Tree 优化"**（★ 补充前面⑥没列全的）：

```cpp
/* storage/innobase/include/btr0sea.ic:48 —— ★ 自适应哈希索引 */
  if (dict_index_is_spatial(index) || !btr_search_enabled) {
    return;      /* ★★ R-Tree 完全不建 AHI（MBR 无全序，无法哈希） */
  }
```
```cpp
/* storage/innobase/include/ibuf0ibuf.ic:122 —— ★ change buffer */
  return (innodb_change_buffering != IBUF_USE_NONE && ibuf->max_size != 0 &&
          index->space != dict_sys_t::s_dict_space_id &&
          !index->is_clustered() && !dict_index_is_spatial(index) &&
          !dict_index_has_desc(index) && ...
```

**★ 记录复制必须"逐条插入"以保持顺序**（`page0page.cc:570`）：

```cpp
    if (dict_index_is_spatial(index)) {
      ...
      /* For spatial index, we need to insert recs one by one
      to keep recs ordered. */
      rtr_page_copy_rec_list_end_no_locks(new_block, block, rec, index, heap,
                                          rec_move, max_to_move, &num_moved,
```
```cpp
  if (dict_index_is_spatial(index) && rec_move) {
    lock_rtr_move_rec_list(new_block, block, rec_move, num_moved);
                    /* ★ R-Tree 用谓词锁迁移，而非 B-Tree 的 lock_move_rec_list_* */
  } else if (!dict_table_is_locking_disabled(index->table)) {
    lock_move_rec_list_end(new_block, block, rec);
  }
```

**★ SPATIAL 与 FULLTEXT 一起被排除在"仅索引扫描"之外**（`sql/sql_optimizer.cc:7775`）：

```cpp
      if (table->key_info[key].flags & (HA_FULLTEXT | HA_SPATIAL))
        continue;  // ToDo: ft-keys in non-ft queries.   SerG
```

### 搜索路径结构 rtr_info_t

`rtr_info_t`（`gis0type.h`）维护 R-Tree 搜索的中间态：

| 字段 | 作用 |
|---|---|
| `path` | 匹配的非叶子页栈（DFS 搜索路径） |
| `parent_path` | 插入用的父路径 |
| `matches` | 叶子层匹配记录缓存（`matched_rec_t`） |
| `mbr` / `search_mode` | 搜索 MBR 与模式 |

**★ 完整定义**（`storage/innobase/include/gis0type.h:108`）：

```cpp
typedef struct rtr_info {
  rtr_node_path_t *path;        /*!< vector holding matching pages */
  rtr_node_path_t *parent_path; /*!< vector holding parent pages during search */
  matched_rec_t *matches;       /*!< struct holding matching leaf records */
  ib_mutex_t rtr_path_mutex;
  buf_block_t *tree_blocks[RTR_MAX_LEVELS + RTR_LEAF_LATCH_NUM];
  ulint tree_savepoints[RTR_MAX_LEVELS + RTR_LEAF_LATCH_NUM];
  rtr_mbr_t mbr;                /*!< the search MBR */
  que_thr_t *thr;
  mem_heap_t *heap;
  btr_cur_t *cursor;
  dict_index_t *index;
  bool need_prdt_lock;          /*!< whether we will need predicate lock */
  bool need_page_lock;
  bool allocated;
  bool mbr_adj;                 /*!< whether mbr will need to be enlarged */
  bool fd_del;
  const dtuple_t *search_tuple;
  page_cur_mode_t search_mode;
  bool *is_dup;                 /* TODO: temporary fix, will be removed later */
} rtr_info_t;
```

**★ 路径上的一个节点**（`gis0type.h:57`，★ SSN 就存在这里）：

```cpp
typedef struct node_visit {
  page_no_t page_no;    /*!< the page number */
  node_seq_t seq_no;    /*!< the SSN (split sequence number) */
  ulint level;          /*!< the page's index level */
  page_no_t child_no;   /*!< child page num if for parent recording */
  btr_pcur_t *cursor;   /*!< cursor structure if we positioned */
  double mbr_inc;       /*!< whether this node needs to be enlarged */
} node_visit_t;
```

> ★ `RTR_MAX_LEVELS = 100`（同 `BTR_MAX_LEVELS`）；`RTR_LEAF_LATCH_NUM = 3` —— 叶子层可能同时 latch **左/中/右 3 页**（因为有分裂/收缩可能）。

**★★ 为什么要"双路径"**（`gis0sea.cc:131`，★ 源码注释给出精确条件）：

```cpp
  /* Whether need to track parent information. Only need so
  when we do tree altering operations (such as index page merge) */
  need_parent = ((my_latch_mode == BTR_MODIFY_TREE ||
                  my_latch_mode == BTR_CONT_MODIFY_TREE) &&
                 mode == PAGE_CUR_RTREE_LOCATE);
```

> ★★★ `path` 用于**自上而下**定位，`parent_path` 用于**自下而上**回溯（改祖先 MBR / 删父指针）。**只有 `BTR_MODIFY_TREE` / `BTR_CONT_MODIFY_TREE` + `PAGE_CUR_RTREE_LOCATE` 时才填 `parent_path`**——纯搜索（`BTR_SEARCH_LEAF`）不需要，省开销。
> ★ `parent_path` 里每个节点持有**独立的 `btr_pcur_t`**（堆分配，`gis0sea.cc:1348`），供回溯时精确定位父记录。

**★ 叶子命中记录的"影子缓冲"**（`gis0type.h:84`）：

```cpp
typedef struct matched_rec {
  byte *bufp;
  byte rec_buf[UNIV_PAGE_SIZE_MAX * 2];  /*!< buffer used to copy matching rec */
  buf_block_t block;                     /*!< the shadow buffer block */
  ...
  bool valid;   /*!< whether result in matched_recs or this search is valid */
} matched_rec_t;
```
> ★★ 命中记录被**拷贝**到一个 2×页大小的影子缓冲区——因为 R-Tree 搜索要**提前释放页 latch**，不能让游标继续指着页。这就是"R-Tree 先收集一批命中、再逐条返回"的实现基础。

**★ SSN 的用途**（`gis0sea.cc:274`，★ 并发分裂检测）：

```cpp
    /* If there are splits, push the split page. */
    if (page_ssn > path_ssn) {
      page_no_t next_page_no = btr_page_get_next(page, mtr);
      rtr_non_leaf_stack_push(rtr_info->path, next_page_no, path_ssn, level, 0,
                              nullptr, 0);
```
> ★ **页的当前 SSN > 路径里记的 SSN ⇒ 我上次访问后这页被分裂过 ⇒ 分裂出去的右兄弟也要补进搜索路径。** 这就是 R-Tree 搜索必须记 SSN 的原因。

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

**★★ `rtr_info_t` 真名位置：`include/gis0type.h:109`**（⚠️ **不是** `gis0rtree.h`）：

```cpp
/* Node Sequence Number. Only updated when page splits */
typedef uint32_t node_seq_t;

/* RTree internal non-leaf Nodes to be searched, from root to leaf */
typedef struct node_visit {
  page_no_t page_no;  /*!< the page number */
  node_seq_t seq_no;  /*!< the SSN (split sequence number */
  ulint level;        /*!< the page's index level */
  page_no_t child_no; /*!< child page num if for parent recording */
  btr_pcur_t *cursor; /*!< cursor structure if we positioned
                      FIXME: there is no need to use whole
                      btr_pcur_t, just the position related
                      members */
  double mbr_inc;     /*!< whether this node needs to be enlarged for insertion */
} node_visit_t;

typedef std::vector<node_visit_t, ut::allocator<node_visit_t>> rtr_node_path_t;
```
> ★★ **"路径"是 `vector<node_visit_t>`，不是定长数组**，也不是严格 LIFO 的栈（回溯时从头重新扫描）。
> ★ 与 `sql/spatial.h` 侧的 `Rtree_info` **是两个不同东西**，无同名冲突。

**★ `rtr_info_t` 的其余关键成员**（`gis0type.h:109`）：

```cpp
/* Maximum index level for R-Tree, this is consistent with BTR_MAX_LEVELS */
constexpr uint32_t RTR_MAX_LEVELS = 100;

/* Number of pages we latch at leaf level when there is possible Tree
modification (split, shrink), we always latch left, current and right pages */
constexpr uint32_t RTR_LEAF_LATCH_NUM = 3;      /* ★★ 叶层同时 latch 3 页 */

typedef struct rtr_info {
  rtr_node_path_t *path;          /*!< vector holding matching pages */
  rtr_node_path_t *parent_path;   /*!< vector holding parent pages during search */
  matched_rec_t *matches;         /*!< struct holding matching leaf records */
  ib_mutex_t rtr_path_mutex;
  buf_block_t *tree_blocks[RTR_MAX_LEVELS + RTR_LEAF_LATCH_NUM];
  ulint tree_savepoints[RTR_MAX_LEVELS + RTR_LEAF_LATCH_NUM];
  rtr_mbr_t mbr;                  /*!< the search MBR */
  que_thr_t *thr;
  mem_heap_t *heap;
  btr_cur_t *cursor;
  dict_index_t *index;
  bool need_prdt_lock;            /*!< whether we will need predicate lock */
  bool need_page_lock;            /*!< whether we will need predicate page lock */
  ...
  /* TODO: This is for a temporary fix, will be removed later */
  bool *is_dup;
} rtr_info_t;
```

**★★ 回溯的核心：分裂检测与右兄弟补入**（`storage/innobase/gis/gis0sea.cc:265`，★ R-Tree 特有的"分裂补偿式回溯"）：

```cpp
    page = buf_block_get_frame(block);
    page_ssn = page_get_ssn_id(page);

    /* If there are splits, push the split page.
    Note that we have SX lock on index->lock, there
    should not be any split/shrink happening here */
    if (page_ssn > path_ssn) {        /* ★★ 路径里记的 SSN < 当前页 SSN
                                          ⇒ 搜索期间该页被分裂过
                                          ⇒ 漏掉的那一半在右兄弟页 */
      page_no_t next_page_no = btr_page_get_next(page, mtr);
      rtr_non_leaf_stack_push(rtr_info->path, next_page_no, path_ssn, level, 0,
                              nullptr, 0);       /* ★ 把右兄弟补回 path */

      if (!srv_read_only_mode && mode != PAGE_CUR_RTREE_INSERT &&
          mode != PAGE_CUR_RTREE_LOCATE) {
        ut_ad(rtr_info->thr);
        lock_place_prdt_page_lock({space, next_page_no}, index, rtr_info->thr);
      }
      new_split = true;
```
> ★★ **B-Tree 用 latch 钉住路径所以不需要这个**；R-Tree 因为搜索路径可被并发分裂打乱，必须在回溯时靠 SSN 主动补偿。

**★ 命中记录在"影子缓冲块"里读**（`gis0sea.cc:432`，★ 一个反直觉的实现细节）：

```cpp
    cursor->m_btr_cur.page_cur.rec = rec.r_rec;
    cursor->m_btr_cur.page_cur.block = &rtr_info->matches->block;
                    /* ★★ 不在原页上读，而在 matches->block 影子块里读 */
    return (true);
  }
  mutex_exit(&rtr_info->matches->rtr_match_mutex);
  /* Fetch the next page */
  return (rtr_pcur_getnext_from_path(tuple, mode, &cursor->m_btr_cur, cur_level,
                                     cursor->m_latch_mode, false, mtr));
```
> ★ `matched_rec_t` 里有 `byte rec_buf[UNIV_PAGE_SIZE_MAX * 2]` —— 匹配记录被**拷贝**到这个缓冲，因为原页可能已被修改/释放。

**★ 单页内的回溯重试**（`gis0sea.cc:1196`）：

```cpp
  /* Page has changed, for R-Tree, the page cannot be shrunk away,
  so we search the page and its right siblings */
  page = buf_block_get_frame(block);
  page_ssn = page_get_ssn_id(page);
  ...
  /* Check the page SSN to see if it has been split, if so, search
  the right page */
  if (!ret && page_ssn > path_ssn) {
    page_no = btr_page_get_next(page, mtr);
    goto search_again;               /* ★ goto 重搜右兄弟 */
  }
```

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

只有 `SP_EQUALS/DISJOINT/INTERSECTS/TOUCHES/CROSSES/WITHIN/CONTAINS/OVERLAPS` 这 8 个 Functype 才建立 R-Tree range；其它空间函数不用索引（`range_analysis.cc` 注释："We cannot involve spatial indexes for queries that don't use MBREQUALS(), MBRDISJOINT(), etc."）。

**★ Functype 全集**（`sql/item_func.h:208`）：

```cpp
    SP_EQUALS_FUNC,      /* ★ MBREquals  / ST_Equals */
    SP_DISJOINT_FUNC,    /* ★ MBRDisjoint / ST_Disjoint */
    SP_INTERSECTS_FUNC,  /* ★ MBRIntersects / ST_Intersects */
    SP_TOUCHES_FUNC,     /* ★ MBRTouches / ST_Touches */
    SP_CROSSES_FUNC,     /* ★ ST_Crosses */
    SP_WITHIN_FUNC,      /* ★ MBRWithin / ST_Within */
    SP_CONTAINS_FUNC,    /* ★ MBRContains / ST_Contains */
    SP_COVEREDBY_FUNC,   /* ★ MBRCoveredBy / ST_CoveredBy —— ⚠️ 不在 range 白名单 */
    SP_COVERS_FUNC,      /* ★ MBRCovers / ST_Covers —— ⚠️ 不在 range 白名单 */
    SP_OVERLAPS_FUNC,    /* ★ MBROverlaps / ST_Overlaps */
    SP_STARTPOINT,       /* ★ 以下 6 个是访问器而非谓词，永远不能走索引 */
    SP_ENDPOINT,
    SP_EXTERIORRING,
    SP_POINTN,
```

**★★ 白名单只有 8 个，`COVERS` / `COVEREDBY` 被排除**（`sql/range_optimizer/range_analysis.cc:1428`，★ 可引用的硬证据）：

```cpp
  if (key_part->image_type == Field::itMBR) {
    // @todo: use is_spatial_operator() instead?
    switch (type) {
      case Item_func::SP_EQUALS_FUNC:
      case Item_func::SP_DISJOINT_FUNC:
      case Item_func::SP_INTERSECTS_FUNC:
      case Item_func::SP_TOUCHES_FUNC:
      case Item_func::SP_CROSSES_FUNC:
      case Item_func::SP_WITHIN_FUNC:
      case Item_func::SP_CONTAINS_FUNC:
      case Item_func::SP_OVERLAPS_FUNC:
        break;                       /* ★★ 只有这 8 个 */
      default:
        /*
          We cannot involve spatial indexes for queries that
          don't use MBREQUALS(), MBRDISJOINT(), etc. functions.
        */
        goto end;
    }
  }
```
> ★★ **`ST_Covers` / `ST_CoveredBy` / `MBRCovers` / `MBRCoveredBy` 不能走 SPATIAL 索引** —— 这是最反直觉的一条。它们在 `is_spatial_operator()` 里（16 项全集），但**不在** range 白名单里。
> ★ 源码自己也承认这是重复代码（`@todo: use is_spatial_operator() instead?`），且已经漂移了。

**★ `is_spatial_operator()` 是 16 项全集**（`range_analysis.cc:1039`）：

```cpp
/**
  Test whether a comparison operator is a spatial comparison
  operator, i.e. Item_func::SP_*.

  Used to check if range access using operator 'op_type' is applicable
  for a non-spatial index.
*/
static bool is_spatial_operator(Item_func::Functype op_type) {
  switch (op_type) {
    case Item_func::SP_EQUALS_FUNC:      case Item_func::SP_DISJOINT_FUNC:
    case Item_func::SP_INTERSECTS_FUNC:  case Item_func::SP_TOUCHES_FUNC:
    case Item_func::SP_CROSSES_FUNC:     case Item_func::SP_WITHIN_FUNC:
    case Item_func::SP_CONTAINS_FUNC:    case Item_func::SP_COVEREDBY_FUNC:
    case Item_func::SP_COVERS_FUNC:      case Item_func::SP_OVERLAPS_FUNC:
    case Item_func::SP_STARTPOINT:       case Item_func::SP_ENDPOINT:
    case Item_func::SP_EXTERIORRING:     case Item_func::SP_POINTN:
    case Item_func::SP_GEOMETRYN:        case Item_func::SP_INTERIORRINGN:
      return true;
    default:
      return false;
  }
}
```
> ★ 这个函数的用途是**反过来**的：在非空间索引上**禁用**空间谓词。
```cpp
      if (key_part->image_type != Field::itMBR &&
          is_spatial_operator(cond_func->functype()))
        continue;
```

**★ MBR* 与 ST_* 是同一个 Functype**（★ 这条纠正了常见误解）：

```cpp
class Item_func_mbrcontains final : public Item_func_spatial_relation {
  enum Functype functype() const override { return SP_CONTAINS_FUNC; }
  enum Functype rev_functype() const override { return SP_WITHIN_FUNC; }
  const char *func_name() const override { return "mbrcontains"; }
};
```
```cpp
class Item_func_st_contains final : public Item_func_spatial_relation {
  enum Functype functype() const override { return SP_CONTAINS_FUNC; }
  enum Functype rev_functype() const override { return SP_WITHIN_FUNC; }
  const char *func_name() const override { return "st_contains"; }
};
```
> ★★ **`ST_Contains` 与 `MBRContains` 的 Functype 完全相同** —— 所以 `ST_Contains` / `ST_Within` / `ST_Intersects` **都能走 SPATIAL 索引**（先在 MBR 层面粗筛，再精确计算）。
> ★ `rev_functype()` 用于参数交换时的语义反转。

**★ `itMBR` 的设置处**（`sql/range_optimizer/range_optimizer.cc:413`）：

```cpp
        key_parts->image_type = (part < key_info->user_defined_key_parts &&
                                 key_info->flags & HA_SPATIAL)
                                    ? Field::itMBR
```
> ⚠️ **`get_mm_spatial()` 已不存在**（8.0 拆包后全仓 grep 0 命中）。SPATIAL 走通用的 `get_mm_parts()` + `is_spatial_operator()`。

---

## 五、GeoJSON 转换

**输出方向**（`ST_AsGeoJSON`）—— **构造 `Json_dom`，不是拼文本**：`Item_func_as_geojson` 继承 `Item_json_func`，实现 `val_json`；`geometry_to_json` 用 `wkb_parser` 扫描内部 swkb，递归 `append_geometry` 填充 `Json_object`，最后序列化成文本。

**输入方向**（`ST_GeomFromGeoJSON`）—— **走标准 JSON 解析器**：`get_json_wrapper` → `json_value` → `parse_json` → `Json_dom::parse`（内部 rapidjson），得到 DOM 后手工翻译成 WKB。

**⚠️ 文件真名**：**不存在** `sql/gis/geojson.h` / `geojson.cc`。两个方向的实现**全在 `sql/item_geofunc.cc`**（`ST_AsGeoJSON` 在 `:2360`，`ST_GeomFromGeoJSON` 在 `:863`）；`geometry_to_json()` 声明在 `sql/item_json_func.h:1189`。

**★ 关键：两个 Item 类的基类不同**（★ 决定了"复用"的方式）：

```cpp
/* sql/item_geofunc.h:491 —— 输出方向 */
class Item_func_as_geojson : public Item_json_func {
 private:
  int m_max_decimal_digits;
  bool m_add_bounding_box;      /* ★ bbox */
  bool m_add_short_crs_urn;     /* ★ "EPSG:4326" */
  bool m_add_long_crs_urn;      /* ★ "urn:ogc:def:crs:EPSG::4326" */
  uint32 m_geometry_srid;
  ...
  bool val_json(Json_wrapper *wr) override;
```
```cpp
/* sql/item_geofunc.h:375 —— 输入方向 */
class Item_func_geomfromgeojson : public Item_geometry_func {   /* ★ 不是 Item_json_func */
 public:
  enum enum_handle_coordinate_dimension {
    reject_document,            /* ★ options=1 */
    strip_now_accept_future,    /* ★ options=2 */
    strip_now_reject_future,    /* ★ options=3 */
    strip_now_strip_future      /* ★ options=4 */
  };
  ...
  String *val_str(String *) override;   /* ★ 输出 swkb 字符串，不是 JSON */
```
> ★★ `ST_AsGeoJSON` 继承 `Item_json_func`（产出就是 `Json_wrapper` DOM，由 JSON 层序列化）；`ST_GeomFromGeoJSON` 继承 `Item_geometry_func`（只**借** JSON 解析器读入，产出是 swkb）。**"复用"是单向的。**

**★ GeoJSON 字符串常量 = 类型映射表**（`sql/item_geofunc.cc:821`）：

```cpp
const char *Item_func_geomfromgeojson::TYPE_MEMBER = "type";
const char *Item_func_geomfromgeojson::CRS_MEMBER = "crs";
const char *Item_func_geomfromgeojson::GEOMETRY_MEMBER = "geometry";
const char *Item_func_geomfromgeojson::PROPERTIES_MEMBER = "properties";
const char *Item_func_geomfromgeojson::FEATURES_MEMBER = "features";
const char *Item_func_geomfromgeojson::GEOMETRIES_MEMBER = "geometries";
const char *Item_func_geomfromgeojson::COORDINATES_MEMBER = "coordinates";
const char *Item_func_geomfromgeojson::SHORT_EPSG_PREFIX = "EPSG:";
const char *Item_func_geomfromgeojson::LONG_EPSG_PREFIX = "urn:ogc:def:crs:EPSG::";
const char *Item_func_geomfromgeojson::CRS84_URN = "urn:ogc:def:crs:OGC:1.3:CRS84";
const char *Item_func_geomfromgeojson::POINT_TYPE = "Point";
const char *Item_func_geomfromgeojson::LINESTRING_TYPE = "LineString";
const char *Item_func_geomfromgeojson::FEATURE_TYPE = "Feature";
const char *Item_func_geomfromgeojson::FEATURECOLLECTION_TYPE = "FeatureCollection";
const char *Item_func_geomfromgeojson::GEOMETRYCOLLECTION_TYPE = "GeometryCollection";
```

**★★ 默认 SRID 是 4326，且是"先占位后回填"**（`sql/item_geofunc.cc:863`，★ 一个巧妙的实现细节）：

```cpp
  if (buf->reserve(GEOM_HEADER_SIZE, 512)) { ... }
  buf->set_charset(&my_charset_bin);
  buf->length(0);
  q_append(static_cast<uint32>(4326), buf);     /* ★ 先写默认 4326 占位 */
  ...
  if (parse_object(root_obj, &rollback, &collection_buffer, false,
                   &result_geometry)) { ... }

  // Set the correct SRID for the geometry data.
  if (m_user_provided_srid) {
    write_at_position(0, m_user_srid, buf);      /* ★ 优先级 1：显式 srid 参数 */
  } else if (m_srid_found_in_document > -1) {
    ...
    write_at_position(0, static_cast<uint32>(m_srid_found_in_document), buf);
                                                 /* ★ 优先级 2：文档内 CRS */
  }
  bool return_result = result_geometry->as_wkb(buf, false);
```
> ★ SRID 优先级：**显式参数 > 文档内 CRS > 默认 4326**（GeoJSON 规范的默认 CRS84）。
> ★ 因为真正的 SRID 要等文档解析完才知道，所以先 `q_append(4326)` 预留 4 字节，最后 `write_at_position(0, ...)` 回填。
> ★ `get_json_wrapper(args, 0, buf, func_name(), &wr)` + `wr.to_dom()` → `Json_object` —— **复用 JSON 解析器的直接证据**。

**★ bbox 与 crs 的输出**（`sql/item_geofunc.cc:2083` / `:2106`）：

```cpp
static bool append_bounding_box(MBR *mbr, Json_object *geometry) {
  assert(GEOM_DIM == 2);
  Json_array *bbox_array = new (std::nothrow) Json_array();
  if (... || geometry->add_alias("bbox", bbox_array) ||
      bbox_array->append_alias(new (std::nothrow) Json_double(mbr->xmin)) ||
      bbox_array->append_alias(new (std::nothrow) Json_double(mbr->ymin)) ||
      bbox_array->append_alias(new (std::nothrow) Json_double(mbr->xmax)) ||
      bbox_array->append_alias(new (std::nothrow) Json_double(mbr->ymax))) { ... }
```
> ★ `bbox` 顺序 `[xmin, ymin, xmax, ymax]`，符合 GeoJSON spec 1.0。写的是**舍入后**的坐标。

```cpp
/**
  ... the long CRS URN is preferred as mentioned in the GeoJSON specification:
  "OGC CRS URNs such as "urn:ogc:def:crs:OGC:1.3:CRS84" shall be preferred
  over legacy identifiers such as "EPSG:4326""
*/
static bool append_crs(Json_object *geometry, bool add_short_crs_urn,
                       bool add_long_crs_urn, uint32 geometry_srid) {
  ...
  crs_object->add_alias("type", new (std::nothrow) Json_string("name"))
  crs_object->add_alias("properties", crs_properties)
  crs_properties->add_alias("name", new (std::nothrow) Json_string(crs_name))
```
> ★ 结构：`{"crs": {"type": "name", "properties": {"name": "EPSG:4326"}}}`。长格式优先于短格式。
> ★ **SRID == 0 时完全不输出 crs**（`geometry_srid > 0` 判断）。

**★ options 的 3 bit 语义**（`sql/item_geofunc.cc:2394`）：bit0(1)=bbox，bit1(2)=短 CRS URN，bit2(4)=长 CRS URN（覆盖 bit1）。
> ★ `maxdecimaldigits` 默认 `INT_MAX32`（不限制）；`my_double_round(x, digits, true, false)`，且 `10 → 10.00`（补零）。
> ★ `GEOMETRYCOLLECTION` 用 `"geometries"` 成员，`Multi*` 用 `"coordinates"`。crs 只在 root object 输出一次，bbox 每层都输（除空 GC）。

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

**★★ 反向映射本体**（`sql/range_optimizer/range_analysis.cc:1657`，★ 注释原文是"经验性补偿"的自白）：

```cpp
    case Item_func::SP_TOUCHES_FUNC:
      tree->root->set_gis_index_read_function(HA_READ_MBR_INTERSECT);
                                    /* ★ TOUCHES → INTERSECT（保守放大） */
      break;
    case Item_func::SP_CROSSES_FUNC:
      tree->root->set_gis_index_read_function(HA_READ_MBR_INTERSECT);
                                    /* ★ CROSSES → INTERSECT，同样保守放大 */
      break;
    case Item_func::SP_WITHIN_FUNC:
      /*
        Adjust the rkey_func_flag as it's assumed and observed that both
        MyISAM and Innodb implement this function in reverse order.
      */
      tree->root->set_gis_index_read_function(HA_READ_MBR_CONTAIN);
                                    /* ★★ WITHIN → 发给引擎的却是 MBR_CONTAIN */
      break;
    case Item_func::SP_CONTAINS_FUNC:
      /*
        Adjust the rkey_func_flag as it's assumed and observed that both
        MyISAM and Innodb implement this function in reverse order.
      */
      tree->root->set_gis_index_read_function(HA_READ_MBR_WITHIN);
                                    /* ★★ CONTAINS → 发给引擎的却是 MBR_WITHIN */
      break;
    case Item_func::SP_OVERLAPS_FUNC:
      tree->root->set_gis_index_read_function(HA_READ_MBR_INTERSECT);
      break;
```
> ★★★ 注释里 **"assumed and observed"**（假设并观察到）——这是**经验性补偿**，不是设计。两个引擎"将错就错"保持了一致，所以没人敢改。**"observed" 这个词本身就说明没人真正验证过语义，只是测出来是这样的。**
> ★ 附带：`TOUCHES` / `CROSSES` / `OVERLAPS` 都退化成 `INTERSECT` —— 因为索引只是粗筛，放大是安全的（结果不漏，只是多召回，最终靠精确计算过滤）。

**★ 引擎侧忠实转发，不再反转**（`storage/innobase/handler/ha_innodb.cc:10085`）：

```cpp
page_cur_mode_t convert_search_mode_to_innobase(ha_rkey_function find_flag) {
  switch (find_flag) {
    ...
    case HA_READ_MBR_CONTAIN:
      return (PAGE_CUR_CONTAIN);      /* ★ 一一对应，引擎侧不再反转 */
    case HA_READ_MBR_INTERSECT:
      return (PAGE_CUR_INTERSECT);
    case HA_READ_MBR_WITHIN:
      return (PAGE_CUR_WITHIN);
    case HA_READ_MBR_DISJOINT:
      return (PAGE_CUR_DISJOINT);
    case HA_READ_MBR_EQUAL:
      return (PAGE_CUR_MBR_EQUAL);
```
> ⚠️ **不存在** `rtr_get_mbr_mode()`（全仓 grep 0 命中）。

```cpp
/* include/my_base.h:78 —— MBR 谓词在 ha_rkey_function 里 */
  HA_READ_MBR_CONTAIN,         /* Minimum Bounding Rectangle contains */
  HA_READ_MBR_INTERSECT,       /* Minimum Bounding Rectangle intersect */
  HA_READ_MBR_WITHIN,          /* Minimum Bounding Rectangle within */
  HA_READ_MBR_DISJOINT,        /* Minimum Bounding Rectangle disjoint */
  HA_READ_MBR_EQUAL,           /* Minimum Bounding Rectangle equal */
```
```cpp
/* include/my_base.h:1103 —— ★ GEOM_FLAG 告诉引擎"这是 rtree 区间" */
  /**
    This flag means that the index is an rtree index, and the interval is
    specified using HA_READ_MBR_XXX defined in enum ha_rkey_function.
  */
  GEOM_FLAG = 1 << 7,
```

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

**★ 唯一可调的 GIS 系统变量**（`sql/sys_vars.cc:3080`）：

```cpp
static Sys_var_ulong Sys_max_points_in_geometry(
    "max_points_in_geometry", "Maximum number of points in a geometry",
    HINT_UPDATEABLE SESSION_VAR(max_points_in_geometry), CMD_LINE(OPT_ARG),
    VALID_RANGE(3, 1024 * 1024L), DEFAULT(64 * 1024), BLOCK_SIZE(1));
```
> ★ `max_points_in_geometry`：默认 **65536**，范围 [3, 1048576]，**SESSION 级且支持 hint**。这是 `ST_Simplify` / `ST_Buffer` 等会产生大量点的函数的**防爆内存**闸门。
> ⚠️ **不存在** `innodb_rtree_*` 系列变量（grep 仅命中 `dict0dd.h:321` 的表名和一个 .test 文件名）。

**★ 恒为 YES 的只读变量**（`sql/sys_vars.cc:6009`）：

```cpp
static Sys_var_have Sys_have_geometry(
    "have_geometry", "have_geometry",
    READ_ONLY NON_PERSIST GLOBAL_VAR(have_geometry), NO_CMD_LINE);
```

**★ 其它相关常量**（★ 非变量，但排障时常被当成配置找）：

| 常量 | 值 | 文件:行 |
|---|---|---|
| `RTR_MAX_LEVELS` | 100 | `storage/innobase/include/gis0type.h:101` |
| `RTR_LEAF_LATCH_NUM` | 3 | `gis0type.h:106`（叶层同时 latch 左/当前/右） |
| `GEO_DATA_HEADER_SIZE` | 4 | `storage/innobase/include/gis0rtree.h:62` |
| `SPLEN` | 8 | `storage/innobase/include/gis0geo.h:36`（MyISAM 遗留） |
| longitude/latitude 校验 | — | ⚠️ **无独立变量**，在 `sql/gis/longitude_range_normalizer.h` 与 SRS 中做 |

---

## Misc

### 扩展点

| 想做什么 | 要动的地方 |
|---|---|
| 新增空间函数 | `item_geofunc*` + `item_create.cc` 注册 |
| 新增空间关系算子 | `range_analysis.cc` 的 SQL 算子 → `HA_READ_MBR_*` 映射 + `gis0sea.cc` 的搜索模式 |
| 改分裂策略 | `gis0geo.cc` 的 `pick_seeds` / `pick_next`（Quadratic Split） |
| **扩到 3 维** | ⚠️ 硬阻塞在 `SPDIMS = 2`（`univ.i:651`）+ `DATA_MBR_LEN = SPDIMS*2*8`。改这一处会牵动 MBR 键长、页格式、比较函数 |

### 已知缺陷

- **MBR 近似导致的假阳性**：R-Tree 扫描结果必须 recheck，大量几何精确相交时性能差。
- **WITHIN/CONTAINS 反向映射**是历史包袱，容易踩坑。
- **只有 MBR 谓词能走索引**：非 MBR 空间函数退化为全表扫。
- **`ST_Covers` / `ST_CoveredBy` 不能走索引**（虽然 `ST_Within` / `ST_Contains` 可以）——白名单只有 8 个 Functype。
- **R-Tree 不建 AHI、不进 change buffer**（`btr0sea.ic:48` / `ibuf0ibuf.ic:122`）——B-Tree 的这两个优化它都用不上。
- **R-Tree 不能做全索引扫描**（`sql_optimizer.cc:1974` 注释：*"it's rtree index does not store data"*）。
- **R-Tree 页不参与 punch-hole 透明压缩**（`buf0buf.cc:5149` 注释）——因为 SSN 占用了 `FIL_PAGE_FILE_FLUSH_LSN` 那 8 字节。

**★ 源码里明确标注的 TODO / FIXME**（★ 逐条 grep 实证）：

| 位置 | 原文 | 性质 |
|---|---|---|
| `storage/innobase/dict/dict.cc:88` | `/* TODO: We suppose the dimension is 2 now. */` | ★ 3D 未实现的官方自认 |
| `sql/range_optimizer/range_analysis.cc:1429` | `// @todo: use is_spatial_operator() instead?` | ★ 重复代码已漂移（COVERS/COVEREDBY） |
| `gis0type.h:67` | `FIXME: there is no need to use whole btr_pcur_t, just the position related members` | 内存冗余 |
| `gis0type.h:144` | `/* TODO: This is for a temporary fix, will be removed later */ bool *is_dup;` | 临时补丁 |
| `sql/spatial.h:384` | `*x = float8get(m_data);  // GIS-TODO: byte order` | ★ 5.7 遗留：旧 parser 硬编码字节序 |
| `sql/sql_optimizer.cc:7776` | `// ToDo: ft-keys in non-ft queries. SerG` | SPATIAL 未参与仅索引扫描 |
| `sql/gis/rtree_support.h:138` | "takes a dimension parameter, but currently only supports ..." | 形参支持 n 维，实现只支持 2 |

**★ 疑似源码 bug**（`sql/gis/box.h:122`，★ 比任何 TODO 都实在）：

```cpp
  /// Returns the minimum corner.
  Cartesian_point const &min_corner() const override { return m_min_corner; }
  Cartesian_point &min_corner() override { return m_min_corner; }   /* Cartesian_box:89 ✓ */
```
```cpp
  Geographic_point const &min_corner() const override { return m_min_corner; }
  Geographic_point &min_corner() override { return m_max_corner; }  /* ★ box.h:122 ✗ */
```
> ★★ 非 const 版 `Geographic_box::min_corner()` 返回的是 **`m_max_corner`**，与 const 版和 `Cartesian_box` 都不一致。这是真实存在的源码缺陷。

**★ 3D / Z 维度不支持的源码证据**（★ 两条）：

```cpp
/* storage/innobase/include/univ.i:649 */
/* Dimension of spatial object we support so far. It has its root in
myisam/sp_defs.h. We only support 2 dimension data */
constexpr uint32_t SPDIMS = 2;
```
```cpp
/* storage/innobase/dict/dict.cc:88 */
  /* TODO: We suppose the dimension is 2 now. */   /* ★ 官方自认 */
```

**★ 地理 vs 笛卡尔：8.0 里地理已是一等公民**（`sql/gis/wkb.cc:391`，★ 源码证据）：

```cpp
  if (srs == nullptr || srs->is_cartesian()) {
    Wkb_parser<Cartesian_point, Cartesian_linestring, Cartesian_linearring,
               Cartesian_polygon, Cartesian_geometrycollection,
               Cartesian_multipoint, Cartesian_multilinestring,
               Cartesian_multipolygon> parser(...);
  } else if (srs->is_geographic()) {
    Wkb_parser<Geographic_point, Geographic_linestring, Geographic_linearring,
               Geographic_polygon, Geographic_geometrycollection,
               Geographic_multipoint, Geographic_multilinestring,
               Geographic_multipolygon> parser(...);
```
> ★ 整套 `Geographic_*` 类型并行存在，不再是"笛卡尔近似"。**这是 8.0 相对 5.7 最大的进步。**
> ★ 引擎侧也按 SRS 分派（`sql/gis/rtree_support.cc:147` 的 `mbr_intersect_cmp` 传 `semi_major_axis/semi_minor_axis`）。

### ★★★ 真名勘误与 5.7 → 8.0 重构对照

**★★ 本篇原来用的是 5.7 的类名体系，8.0.39 已整体重构**：

| 5.7 说法（本篇旧文） | 8.0.39 真相 |
|---|---|
| `sql/spatial.h` 的 `Geometry` 基类 | → **`sql/gis/geometries.h` 的 `gis::Geometry`** |
| `Gis_point` / `Gis_line_string` / `Gis_polygon` / `Gis_multi_*` / `Gis_geometry_collection` | ⚠️ **"已删除"是错的——全都还在**（`sql/spatial.h:1152 / 2099 / 2187 / 2290 / 2333 / 2373 / 2409`）。新体系 `gis::Point` / `Linestring` / `Polygon`… 与其**并存**：新体系负责 SQL 语义与 SRS，旧类作为 **Boost.Geometry 适配器**活着（`sql/gis_bg_traits.h:57` 的 traits 全部注册在旧类上）。**这就是它们删不掉的原因** |
| `feature_dimension`（0/1/2）字段 | **不存在** |
| 内部用位域 `Flags_t`（bo:1, dim:2, geotype:5…） | **不存在**。8.0 是对象树，序列化时按需算长度 |
| `Geometry::create_from_wkb()` / `as_wkb()` | ⚠️ **"已删除"是错的——仍存在**（`sql/spatial.h:604 / :600`），且仍被 `item_geofunc.cc:765` 调用。**新路径**是 `gis::parse_wkb()`（`wkb.h:80`）/ `gis::write_geometry()`（`wkb.h:113`），两套并存 |
| `SPATIAL_SRID_SIZE` | 真名 **`SRID_SIZE` = 4**（`sql/spatial.h:51`） |
| `Coordinate_type::kGeographic/kCartesian` | 真名 **`gis::Coordinate_system`**（`geometries.h:69`） |

**★ 其余真名核查**：

| 项 | 结论 |
|---|---|
| `sql/gis/srid.h` 定义 `class srid_t` | 只有一行实质：`typedef std::uint32_t srid_t;` |
| `rtr_mbr_t` 在 InnoDB `gis0type.h` | **不在** → server 层 `sql/gis/rtree_support.h:39` |
| `rtr_info_t` 在 `gis0rtree.h` | **不在** → `storage/innobase/include/gis0type.h:108` |
| `geom0type.h` | **不存在** → `gis0type.h` |
| `PAGE_RTREE_LEVEL` / `PAGE_RTREE_FIRST` | **不存在**。复用 `PAGE_LEVEL`；唯一额外字段 `FIL_RTREE_SPLIT_SEQ_NUM` 在 **FIL header**（复用 `FIL_PAGE_FILE_FLUSH_LSN`） |
| R-Tree 记录有特殊标志 | **没有**。物理记录格式与 B-Tree 完全共用 |
| `sql/gis/geometry_codec.h` | **不存在** |
| `PAGE_CUR_RTREE_*` 在 `rem0cmp.h` / `page0cur.h` | 真名位置 `storage/innobase/include/page0types.h:176`（`enum page_cur_mode_t`） |
| `FIL_PAGE_RTREE` 值 | `17854`（比 `FIL_PAGE_INDEX` 小 1） |

**★ 三个容易被忽略的事实**

1. **地理坐标内部存弧度，不是度**。`Point` 的 `m_x`/`m_y` 在地理 SRS 下是**弧度**（`geometries.h:229` 注释：*"Geographic coordinates are in radians"*），对外转度由 `srs->from_radians()` 负责——`get_mbr_from_store()` 里落盘 MBR 前那次 `from_radians()` 就是这个转换。

2. **空几何的 MBR 是全域 `±DBL_MAX`，不是 0**。源码注释：*"The geometry is empty, so there is no bounding box. Return a box that covers the entire domain."* 这直接影响"空几何能不能被 SPATIAL 索引命中"的行为。

3. **MySQL 的 linestring 是测地线，不是 OGC 的线性插值**。`geometries.h:264` 注释明说：*"In MySQL, a linestring represents the geodesic ... on an ellipsoid, it's the shortest path along the ellipsoid surface."*

**★★★ 本轮新增的勘误（补充上表）**：

| 常见说法 | 8.0.39 真相 |
|---|---|
| `sql/gis/gis_geometry.h` 定义 `gis::Geometry` | **不存在** → 真名 **`sql/gis/geometries.h`**（7 个几何类**全在这一个文件**） |
| `gis_point.h` / `gis_linestring.h` / `gis_polygon.h` / `gis_multi*.h` / `gis_linearring.h` / `gis_geometrycollection.h` | **全部不存在** → 命名模式是「按**操作**分文件（area/distance/intersects…）+ `geometries*.h` 按**类型**分文件」 |
| `sql/gis/gis_wkb.cc` | **不存在** → 真名 **`sql/gis/wkb.cc`**；写出靠 **`Wkb_visitor`**（`wkb_visitor.cc`），**没有** `write_wkb_header()` 之类函数 |
| `sql/gis/geojson.h` / `geojson.cc` | **不存在** → 两个方向全在 **`sql/item_geofunc.cc`**；`geometry_to_json()` 声明在 `sql/item_json_func.h:1189` |
| `gis::Geometry` 有 `srid()` 成员 | **没有** → 只有 5 个纯虚；SRID 由 `dd::Spatial_reference_system *srs` 单独传 |
| `Linearring` 只是 typedef | **是独立类**（`geometries.h:320`），继承 `Linestring`，非实例化 SQL 类型 |
| 旧 `Geometry` 继承自新 `gis::Geometry` | **并列无继承**（`sql/spatial.h:213` vs `geometries.h:100`，两处头文件互不 include） |
| Boost.Geometry 适配新的 `gis::` 类 | **错** → traits 全部注册在**旧** `Gis_point` / `Gis_line_string` / `Gis_polygon` 上（`gis_bg_traits.h:57` 起） |
| `SPDIMS` / `SPLEN` 在 `sql/` | 在 **InnoDB**：`SPDIMS=2`（`univ.i:651`）、`SPLEN=8`（`gis0geo.h:36`） |
| 源码注释引用 Guttman 1984 | **全仓无 "Guttman" 字样**（0 命中）——算法出处是外部知识，不是源码证据 |
| `Field_geom` 有自己的 `pack()`/`unpack()` | **不存在** → 完全继承 `Field_blob` |
| `MYSQL_TYPE_GEOMETRY` 在 `mysql_com.h` | 在 **`include/field_types.h:87`**，值 **255** |
| MBR 序列化靠 `Field_geom::get_key_image` | **不存在** → 由 InnoDB 侧 `get_mbr_from_store()` → `dfield_write_mbr()` |
| `FIL_PAGE_RTREE = 17856` | **错** → **17854**（`fil0fil.h:1212`）；`FIL_PAGE_INDEX = 17855` |
| 页类型常量叫 `PAGE_RTREE` 在 `page0types.h` | **不存在** → 真名 `FIL_PAGE_RTREE`，在 **`fil0fil.h`** |
| R-Tree 有 `PAGE_RTREE_LEVEL` | **不存在** → 复用 `PAGE_LEVEL = 26` |
| `RTR_NODE_REC_SIZE` | **不存在** → `DICT_INDEX_SPATIAL_NODEPTR_SIZE = 1` + `DATA_MBR_LEN = 32` |
| `dict_index_t` 用 `DICT_RTREE` | **错** → `DICT_SPATIAL = 64`（`dict0mem.h:106`） |
| `HA_KEY_ALG_RTREE` 在 `ha_rkey_function` 里 | **错** → 属 `enum ha_key_alg`，值 **2**（`my_base.h:108`） |
| `get_mm_spatial()` 还在 | **已删除**（0 命中）→ 走 `get_mm_parts()` + `is_spatial_operator()` |
| `rtr_get_mbr_mode()` | **不存在** → `convert_search_mode_to_innobase()`（`ha_innodb.cc:10087`） |
| `ST_Within` / `ST_Contains` 不能走索引 | **错** → 与 `MBRWithin`/`MBRContains` **同一个 Functype**，都能走 |
| 只有 `MBR*()` 能走 SPATIAL | **半错** → `ST_Intersects/Within/Contains/Touches/Crosses/Overlaps` 同样能走；但 **`ST_Covers`/`ST_CoveredBy`/`ST_Equals`/`ST_Disjoint` 不能** |
| `rtr_info_t` 定义在 `gis0rtree.h` | **错** → **`storage/innobase/include/gis0type.h:109`** |
| `innodb_rtree_*` 系统变量 | **不存在**（grep 仅命中表名与 .test 文件名） |
| `MBR` 与 `rtr_mbr_t` 字段顺序一致 | **不同** → `MBR`: xmin,ymin,xmax,ymax；`rtr_mbr_t`: xmin,xmax,ymin,ymax（跨层拷贝必须重排） |
| 地理 MBR 的边是测地线 | **错** → 沿**纬线**走（`box.h:100` 注释），是保守包围 |

> 注：部分 `/* 【注】... */` 与 `★` 注释为本工作区人工批注，非 Oracle 上游原文。

### 社区边界澄清

- R-Tree 空间索引是社区版完整能力（InnoDB 实现，非云厂商私有）。
- 谓词锁（`LOCK_PREDICATE`）是 R-Tree 的并发控制，社区版有（见锁篇）。
- 空间函数（ST_*）是 OGC 标准的子集实现。

**★ 边界：server 层把几何能力以 C API 形式暴露给存储引擎**（`sql/gis/rtree_support.h:27`，★ 官方注释）：

```cpp
///
/// This file declares a set of functions that storage engines can call to do
/// geometrical operations.
///
#include "my_inttypes.h"  // uchar, uint
#include "sql/gis/srid.h"
...
/// Fetches a copy of the dictionary entry for a spatial reference system.
///
/// Spatial reference dictionary cache objects have a limited lifetime,
/// typically until the end of a transaction. This function returns a clone of
/// the dictionary object so that it is valid also after the transaction has
/// ended. This is necessary since InnoDB may do index operations after the
/// transaction has ended.      /* ★★ 设计说明：SRS 必须 clone */
///
/// @return The spatial reference system dictionary entry, or nullptr.
dd::Spatial_reference_system *fetch_srs(gis::srid_t srid);
```
> ★★ **引擎不需要懂 Boost.Geometry** —— 它只调 `mbr_intersect_cmp()` / `mbr_contain_cmp()` / `get_mbr_from_store()` 这些 C 接口。
> ★ SRS 必须 clone 的原因：InnoDB 在**事务结束后**仍可能做索引操作（后台 purge / 分裂），而 DD cache 对象活不过事务。

**★ 一个有趣的血缘事实：InnoDB 的 R-Tree 是 MyISAM 移植来的**（★ 三个文件头自述）：

```cpp
/* storage/innobase/include/gis0geo.h:27 */
/** @file gis0geo.h
 The r-tree define from MyISAM      /* ★★ 自认 */
 *******************************************************/
```
```cpp
/* storage/innobase/include/univ.i:649 */
/* Dimension of spatial object we support so far. It has its root in
myisam/sp_defs.h. We only support 2 dimension data */     /* ★★ 自认 */
```
> ★★ 所以 InnoDB R-Tree 的很多"怪异之处"（`SPLEN`、Quadratic Split 的参数约定、MBR 比较的命名坑）其实是 **MyISAM 遗产**，不是 InnoDB 的设计选择。这解释了为什么 WL#6745 说要"adopt all MyISAM's MRB manipulation functions"。

**★ R-Tree 作者与年代**（`storage/innobase/include/gis0type.h:28`）：

```cpp
/** @file include/gis0type.h
 R-tree header file

 Created 2013/03/27 Jimmy Yang        /* ★ 2013 年，MySQL 5.7 开发期 */
 ***********************************************************************/
```

**⚠️ 源码里能引的"官方线索"只有这些**（`sql/gis/` 与 `storage/innobase/gis/` **均无 WL# 编号**，grep 0 命中）：

| 线索 | 位置 | 原文 |
|---|---|---|
| SFA-CA / OGC 06-103r4 | `sql/gis/geometries.h:32` | `"closely modelled after the hierarchy in SFA-CA (OGC 06-103r4)"` |
| MyISAM 血缘 | `gis0geo.h:27` | `"The r-tree define from MyISAM"` |
| 维度来自 MyISAM | `univ.i:649` | `"It has its root in myisam/sp_defs.h"` |
| R-Tree 作者 | `gis0type.h:31` | `"Created 2013/03/27 Jimmy Yang"` |
| GeoJSON 规范 | `item_geofunc.cc:849` | `"It follows the standard described at http://geojson.org/geojson-spec.html (revision 1.0)"` |

---

## 参考

**论文 / 理论**

- **Guttman《R-Trees: A Dynamic Index Structure for Spatial Searching》(SIGMOD 1984)** —— R-Tree 的原始论文，含 Quadratic Split 算法
- **Kornacker & Banks《High-Concurrency Locking in R-Trees》(VLDB 1995)** —— R-Tree 并发控制（谓词锁 + SSN）的论文
- **OGC Simple Features** —— WKB/WKT、空间关系的标准定义

**官方文档**

- MySQL 8.0 Reference Manual, "Spatial Data Types" / "Optimizing Spatial Analysis" / "Spatial Indexes"

**WorkLog**

- [WL#6455](https://dev.mysql.com/worklog/task/?id=6455) InnoDB GEOMETRY 数据类型支持
- [WL#6745](https://dev.mysql.com/worklog/task/?id=6745) R-tree 索引的 DML 支持（分裂 / 收缩，复用 MyISAM `sp_key.c` / `rt_split.c`）
- [WL#6609](https://dev.mysql.com/worklog/task/?id=6609) 谓词锁（多维数据无"下一个键"，gap lock 不可用）
- [WL#6968](https://dev.mysql.com/worklog/task/?id=6968) R-tree 搜索与定位（整合 6745 + 6609 后发布）
- [WL#6618](https://dev.mysql.com/worklog/task/?id=6618) 多维索引（**未实现**）

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

**★ 源码出处（本轮精确化，★ 全部 grep 实证）**：

| 主题 | 文件 | 关键行 |
|---|---|---|
| 类型体系 | `sql/gis/geometries.h` | 27-46（文件头 / SFA-CA）、50-89（枚举）、93-145（`gis::Geometry`）、147-595（7 个类型类） |
| 坐标系特化 | `sql/gis/geometries_cs.h` | 44-68（`Cartesian/Geographic_point`）、70-120（`Cartesian_linestring`） |
| 8.0 新 WKB | `sql/gis/wkb.cc` | 46-102（字节序 + 类型校验）、104-163（`Wkb_parser` + transform）、165-262（逐字段解析）、359-427（分派）、429-497（SRID+parse_geometry）、499-522（`write_geometry`） |
| WKB 写出 | `sql/gis/wkb_visitor.cc` | 30-128（`Wkb_visitor`，恒小端） |
| 8.0 新 MBR | `sql/gis/box.h` | 38-96（`Box`）、98-129（`Geographic_box`） |
| 引擎契约 | `sql/gis/rtree_support.h` | 27-62（`rtr_mbr_t` + fetch_srs）、64-136（`mbr_*_cmp`） |
| 5.7 遗留 | `sql/spatial.h` | 47-57（布局常量）、59-79（`point_xy`/`wkb_header`）、81-189（`MBR`）、384-395（旧 `wkb_parser`）、600-606（`create_from_wkb/wkt`）、1152-2456（`Gis_*` 系列） |
| Boost 绑定 | `sql/gis_bg_traits.h` | 28-95（全部特化在旧类上） |
| GeoJSON | `sql/item_geofunc.cc` | 821-849（常量）、863-1022（`ST_GeomFromGeoJSON`）、1922-2412（`ST_AsGeoJSON` 全套） |
| `Field_geom` | `sql/field.h` / `sql/field.cc` | 3929-3984 / 7519-7550 |
| R-Tree 页 | `storage/innobase/include/fil0fil.h` | 1211-1212（`FIL_PAGE_RTREE`）、1314-1317 |
| SSN | `fil0types.h` / `page0page.ic` | 103-104 / 93-127 |
| 页创建分派 | `storage/innobase/btr/btr0btr.cc` | 318-358 |
| 节点指针 | `storage/innobase/gis/gis0rtree.cc` | 131-178、978-983、1768-1781 |
| `rtr_info_t` | `storage/innobase/include/gis0type.h` | 57-147 |
| 回溯 | `storage/innobase/gis/gis0sea.cc` | 265-292、432-472、1196-1266 |
| 维度 / MBR 长度 | `univ.i` / `data0type.h` | 649-651 / 212-216 |
| SPATIAL → range | `sql/range_optimizer/range_analysis.cc` | 1039-1072、1089-1097、1428-1447、1657-1690 |
| 模式转换 | `storage/innobase/handler/ha_innodb.cc` | 2974-2982、10085-10111 |
| 系统变量 | `sql/sys_vars.cc` | 3080-3083、6009-6011 |
| 回归测试 | `mysql-test/suite/gis/` | `t/` 下 100+ 用例（可作为行为级参考） |

> ⚠️ **源码里没有的东西**（写文档时勿标为源码证据）：Guttman 论文引用、多值索引式 WL# 编号、`sql/gis/` 下任何 Doxygen `@page`、多值索引的 `ER_MULTI_VALUED_*`。
