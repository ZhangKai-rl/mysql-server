# GIS / GeoJSON 与 JSON 的关系

> 基于 MySQL 8.0.39 源码。本文是**占位/概览文档**，只回答"GIS 与前面讲的 JSON 是什么关系"，GIS 内部实现（R-Tree、MBR、空间运算、gcalc）待后续展开。

## 目录

- [一句话结论](#一句话结论)
- [GeoJSON 相关函数](#geojson-相关函数)
- [几何值的内部存储格式](#几何值的内部存储格式)
- [空间列 vs JSON 列](#空间列-vs-json-列)
- [GeoJSON ↔ JSON 的转换路径](#geojson--json-的转换路径)
- [关键源码位置速查](#关键源码位置速查)

---

## 一句话结论

**GeoJSON 与 JSON 共享解析器（rapidjson / `Json_dom` / `Json_wrapper`），但存储与执行是两个完全独立的子系统**：

- JSON 是 **JSONB 文档类型**（树形、有路径表达式、只能靠生成列建普通索引）；
- GeoJSON 只是 **geometry 的外部文本表示**，几何值内部用自研的 **SRID + WKB** 二进制格式，落 InnoDB 为 `DATA_GEOMETRY`，可以建 **R-Tree 空间索引**。

两者的耦合点只有"一进一出"两个函数：`ST_GeomFromGeoJSON`（用 JSON 解析器读入）和 `ST_AsGeoJSON`（构造 `Json_dom` 输出）。

---

## GeoJSON 相关函数

| 函数 | 类 | 位置 |
|------|-----|------|
| `ST_GeomFromGeoJSON` | `Item_func_geomfromgeojson` | 声明 `item_geofunc.h:375`；`val_str` `item_geofunc.cc:863` |
| └ 解析子过程 | `parse_object` / `get_positions` / `get_linestring` / `get_polygon` / `parse_crs_object` | `item_geofunc.cc:1064 / 1251 / 1513 / 1545 / 1660` |
| `ST_AsGeoJSON` | `Item_func_as_geojson : public Item_json_func` | 声明 `item_geofunc.h:491`；`val_json` `item_geofunc.cc:2360` |
| └ 转换过程 | `geometry_to_json` / `append_geometry` | `item_geofunc.cc:2326 / 2179` |
| `ST_GeomFromText` / `ST_GeomFromWKB` | `Item_func_geometry_from_text` / `..._from_wkb` | `item_geofunc.h:199 / 260`；`item_geofunc.cc:421 / 678` |
| `ST_AsText` / `ST_AsWKT` | `Item_func_as_wkt` | `item_geofunc.h:321`；`item_geofunc.cc:3134` |
| `ST_AsWKB` / `ST_AsBinary` | `Item_func_as_wkb` | `item_geofunc.h:331`；`item_geofunc.cc:3253` |
| 名字注册 | — | `item_create.cc:1562 (ST_ASGEOJSON), 1594 (ST_GEOMFROMGEOJSON)` |

---

## 几何值的内部存储格式

**自研的 SRID + WKB**（内部常称 swkb），与 JSONB 完全无关：

```
4 字节 SRID(小端) + 1 字节字节序(wkb_ndr) + 4 字节 wkbType + 几何数据
```

| 位置 | 内容 |
|------|------|
| `sql/spatial.h:50-55` | `SRID_SIZE=4`、`WKB_HEADER_SIZE=1+4`、`GEOM_HEADER_SIZE` |
| `sql/spatial.h:1112/1138` | `write_geometry_header` |
| `sql/spatial.h:384` / `:593` | `Geometry::wkb_parser` / `Geometry::construct`（实现 `spatial.cc:325`） |
| `sql/gis/wkb.h` / `wkb.cc` | 新 `gis::` 子系统的 WKB 读写 |

---

## 空间列 vs JSON 列

| | 空间列 | JSON 列 |
|---|--------|---------|
| 字段类 | `Field_geom : Field_blob`（`field.h:3929`） | `Field_json : Field_blob`（`field.h:3986`） |
| `type()` | `MYSQL_TYPE_GEOMETRY` | `MYSQL_TYPE_JSON` |
| 字符集 | 固定 `my_charset_bin` | 底层 bin，对外伪装 utf8mb4 |
| 存储内容 | 定长头 + 二进制 WKB 几何结构 | 二进制 JSONB 文档树 |
| 校验 | 存入前做 WKB 合法性校验（`field.cc:7519`） | 存入前做 JSON 语法校验（`field.cc:7633`） |
| 索引 | **R-Tree 空间索引**（`SPATIAL INDEX`） | 不能直接建索引，需生成列/函数索引/多值索引 |
| InnoDB 映射 | `DATA_GEOMETRY`（`data0type.h:92`；`ha_innodb.cc:7991`） | `DATA_BLOB`（`ha_innodb.cc:7996`） |

---

## GeoJSON ↔ JSON 的转换路径

**输出方向**（`ST_AsGeoJSON`）—— **构造 `Json_dom`，不是拼文本**：
`Item_func_as_geojson` 继承 `Item_json_func`，实现 `val_json`；`geometry_to_json` 用 `wkb_parser` 扫描内部 swkb，递归 `append_geometry` 填充 `Json_object`（`item_geofunc.cc:2343/2353`），最后经 `Item_json_func::val_str` → `wr.to_string()` 序列化成文本（`item_json_func.cc:1283`）。

**输入方向**（`ST_GeomFromGeoJSON`）—— **走标准 JSON 解析器**：
`item_geofunc.cc:911` 调 `get_json_wrapper`（`item_json_func.cc:1086`）→ `json_value`（`:1066`）→ `parse_json`（`:132`）→ `Json_dom::parse`（`json_dom.cc:562`，内部 rapidjson），得到 DOM 后手工翻译成 WKB。

**即：GeoJSON 只是交换格式，几何值从不以 JSONB 形式存储。**

---

## 关键源码位置速查

| 位置 | 说明 |
|------|------|
| `sql/item_geofunc.cc:863` | `ST_GeomFromGeoJSON` |
| `sql/item_geofunc.cc:2360` | `ST_AsGeoJSON::val_json` |
| `sql/item_geofunc.cc:2326 / 2179` | `geometry_to_json` / `append_geometry` |
| `sql/spatial.h:50-55 / 384 / 593` | swkb 布局常量 / wkb_parser / construct |
| `sql/field.h:3929` / `field.cc:7519` | `Field_geom` / WKB 校验 |
| `storage/innobase/handler/ha_innodb.cc:7991` | `MYSQL_TYPE_GEOMETRY → DATA_GEOMETRY` |
| `storage/innobase/rem/rem0cmp.cc:372/432/717` | 几何类型的比较/排序特殊处理 |
| `sql/gis/` | 新版 GIS 子系统（待展开） |
