# InnoDB 表空间碎片度量（DATA_FREE / FREE_EXTENTS）深度解析

> 基于 MySQL 8.0.39 源码，涵盖表空间空闲空间的三种查询入口、`DATA_FREE` 与 `FREE_EXTENTS` 的计算链路与差异、两层碎片（extent 级 / 页内）的辨析、以及碎片整理。
>
> **边界**：本篇聚焦"碎片如何度量与计算"；表空间的物理组织（extent/segment/页分配、`fsp`/`fseg` 空间管理）、页内合并（`MERGE_THRESHOLD`）另见相关主题（后续补充）。

## 目录

- [概述](#概述)
- [查询方法](#查询方法)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

InnoDB 表空间碎片，指"已分配给表空间、但当前未被数据占用的空间"。对外可见的度量有两个入口：

1. `information_schema.TABLES.DATA_FREE`（= `SHOW TABLE STATUS` 的 `Data_free` 列）：估算值，单位字节；
2. `information_schema.FILES.FREE_EXTENTS`（5.7 为 `INNODB_SYS_TABLESPACES.FREE_EXTS`）：空闲 extent 的个数。

两者的数据来源是同一个底层事实——`fil_space_t` 里的 free extent 链表（`free_len`），但加工口径不同。

### 用途

回答运维最关心的问题："这个表到底浪费了多少磁盘空间？删掉数据能不能瘦身？" 它也是判断是否需要 `OPTIMIZE TABLE` / `ALTER TABLE ... ENGINE=InnoDB` 重建的依据之一。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | 用 `INNODB_SYS_TABLESPACES.FREE_EXTS / TOTAL_EXTS` 看空闲 extent；`TABLES.DATA_FREE` 已存在 |
| 8.0 | 数据字典（DD）系统视图化：`INNODB_SYS_TABLESPACES` 被 `FILES` 视图取代，`FREE_EXTENTS` 经 `INTERNAL_TABLESPACE_FREE_EXTENTS()` 函数读取；`TABLES.DATA_FREE` 改经 `INTERNAL_DATA_FREE()` 读取 |

---

## 查询方法

最常用的是按表看 `TABLES.DATA_FREE`（估算值，单位字节）：

```sql
SELECT table_schema, table_name, engine,
       ROUND(data_length/1024/1024, 2)  AS data_mb,       -- 聚簇索引占用
       ROUND(index_length/1024/1024, 2) AS index_mb,      -- 二级索引占用
       ROUND(data_free/1024/1024, 2)    AS data_free_mb,  -- 空闲（可复用）字节
       ROUND(data_free / (data_length + index_length + data_free) * 100, 2) AS frag_pct
FROM   information_schema.tables
WHERE  table_schema = 'your_db'
  AND  engine = 'InnoDB'
ORDER  BY data_free DESC;
```

`frag_pct` 是"碎片占表总分配空间的比例"的近似值，用来圈定哪些表值得重建。注意 `DATA_FREE` 是估算值、且表很小（不足 1 个 extent）时恒为 0，`frag_pct` 只作相对参考。

快速看单个表，用 `SHOW TABLE STATUS`，关注 `Data_free` 列（就是上面的 `DATA_FREE`）：

```sql
SHOW TABLE STATUS FROM your_db LIKE 't%';
```

要按表空间看空闲 extent 个数（精确值），查 `FILES` 视图：

```sql
SELECT FILE_NAME, ENGINE, FREE_EXTENTS, TOTAL_EXTENTS,
       FREE_EXTENTS * EXTENT_SIZE AS free_bytes   -- 空闲字节 = extent 个数 × extent 大小
FROM   information_schema.FILES
WHERE  ENGINE = 'InnoDB' AND FILE_TYPE = 'TABLESPACE';
```

5.7 没有 `FILES` 视图，对应的是 `INNODB_SYS_TABLESPACES`：

```sql
SELECT SPACE, NAME, FREE_EXTS, TOTAL_EXTS
FROM   information_schema.INNODB_SYS_TABLESPACES;
```

碎片整理靠重建表：

```sql
OPTIMIZE TABLE t;              -- 对 InnoDB 等价于重建表
ALTER TABLE t ENGINE=InnoDB;   -- 重建后收缩 file-per-table 的 .ibd
```

> 三种入口的口径差异（估算 vs 精确）见「核心实现」；`DATA_FREE` 大不代表能立刻释放磁盘，见「理论基础 · 设计思想与权衡」。

---

## 理论基础

### 设计思想与权衡

**为什么"碎片量"是个估算值，而不是一个精确计数器？**

因为 InnoDB 不需要、也不维护"每张表当前有多少字节没被数据占用"这样一个精确数字——那需要额外的全局记账，且每次 DML 都要维护。它选择在**被查询时现场估算**：拿表空间的 free extent 链表长度，扣掉为 undo 和清理操作预留的余量，得出"还能再塞多少新数据"的近似值。

这个设计的代价就是：
- **不是精确值**：返回的是"可插入空间的上界估算"，且 `size < 1 extent`（64 页）的小表直接返回 0；
- **获取有成本**：要拿 InnoDB 内部 latch，所以 `info_low()` 在 `HA_STATUS_NO_LOCK` 时跳过（省 CPU，Bug#38185）；
- **语义被误读**：`DATA_FREE` 大，不代表 `.ibd` 能立刻收缩——file-per-table 下文件只增不减，释放的 extent 挂回 `FSP_FREE` 复用，物理文件大小不变。

**两个层面碎片的取舍**：InnoDB 的 B-tree 页默认填充到 15/16（约 93.75%），预留 1/16 给后续 UPDATE 就地更新，避免频繁页分裂。这一层"页内空隙"是**有意为之的写放大缓冲**，不在 `DATA_FREE` 统计范围内；`DATA_FREE` 只统计段外整块空闲 extent。二者本质不同，运维上不应混用。

**离散 vs 顺序删除的本质差异**：extent 释放走 `fseg_free_extent`（fsp0fsp.cc:3593），**要求整 extent 内 64 页全空才挂回 `FSP_FREE` 链表**。离散删除下大量"半空"页，extent 不会释放 → `DATA_FREE` 几乎不动；顺序删除容易让整 extent 全空，触发释放 → `DATA_FREE` 增大。注意三个细节：

1. **物理 `.ibd` 只增不减**：`size_in_header` 不回缩，extent 释放后只供后续 INSERT 复用，文件不会物理收缩；只有 `OPTIMIZE TABLE` 重建才真正"瘦 `.ibd`"；
2. **释放是异步的**：purge 延迟执行，不会删除后立即看到 `DATA_FREE` 变化；
3. **段预占 extent 池**：段持有的页面比 free extent 多，释放回链表的速度可能比预期慢。

所以 `DATA_FREE` 看不出来的是**页内空隙**（fill factor 下降带来的 IO 读放大）——这正是 `OPTIMIZE TABLE` 要解决的核心问题。

### `OPTIMIZE TABLE` 真实实现

`ha_innobase::optimize` (ha_innodb.cc:18125) **不做任何"优化"**，只返回 `HA_ADMIN_TRY_ALTER`：

```cpp
int ha_innobase::optimize(THD *, HA_CHECK_OPT *) {
  if (innodb_optimize_fulltext_only) {
    // 只整理 FTS 索引（不重建表、不阻塞 DML、不缩 .ibd）
    fts_optimize_table(m_prebuilt->table);
    return HA_ADMIN_OK;
  }
  return HA_ADMIN_TRY_ALTER;   // 关键：让 server 改写为 ALTER TABLE t ENGINE=InnoDB
}
```

server 层 `Sql_cmd_optimize_table::execute` 拿到 `HA_ADMIN_TRY_ALTER` 后，改走 `mysql_alter_table()` 路径，本质是 `ALTER TABLE t ENGINE=InnoDB`：

- 8.0 默认 `ALGORITHM=INPLACE, LOCK=NONE`（online DDL），**期间允许并发 DML 和 SELECT**；
- online 期间产生的 DML 暂存到内部 row log，最后 apply；`innodb_online_alter_log_max_size`（默认 128 MB）不够时 online alter 失败；
- commit/rename 阶段有毫秒级 MDL 阻塞。

因此 `OPTIMIZE TABLE` 中间**能写入**（DML 受 row log 容量约束），结尾有短暂阻塞；想强制阻塞 DML 用 `ALGORITHM=COPY`，但违背了 online 初衷。

### 理论溯源

- **空间管理粒度 = extent**：与 Oracle 的 extent 概念同源。释放/分配都以 extent（64 页 = 1MB @16KB）为单位，free extent 用 `FSP_FREE` 链表串联，避免逐页跟踪的高开销。
- **预留余量（reserve）**：`fsp_get_available_space_in_free_extents` 里扣掉 `2 extent + 1%` 给 undo 段和清理（purge/cleaning）操作——这是对"空间统计不能误导写入"的一种保守策略，宁可少报，不让上层以为空间充足而写入失败。

### 算法与数据结构

- `fil_space_t::free_len`：free extent 链表长度；
- `fil_space_t::free_limit`：已初始化页的边界（之上的页还能切出完整 extent）；
- `fil_space_t::size_in_header`：表空间头记录的 size（页数）；
- 计算复杂度 O(1)：纯内存读 + 常数次算术，不做任何扫描。

### 他库对比

- **PostgreSQL**：表碎片通常看 `pgstattuple` 扩展（精确到死元组/空闲空间，但需全表扫描）；InnoDB 的 `DATA_FREE` 是 O(1) 估算，精度换速度。
- **Oracle**：`dba_free_space` / `dba_segments` 能精确列出每个表空间的空闲块；InnoDB 不做这么细的记账。

---

## 核心实现

### 主链路

```
information_schema.TABLES.DATA_FREE
  → tables.cc 视图定义：DATA_FREE = INTERNAL_DATA_FREE(...)
  → Item_func_internal_data_free::val_int()                 (sql/item_func.cc)
    → retrieve_table_statistics()
      → ha_innobase::info_low()                             (ha_innodb.cc)  [HA_STATUS_VARIABLE_EXTRA 时]
        → calculate_delete_length_stat()
          → fsp_get_available_space_in_free_extents()       (fsp0fsp.cc)
        → stats->delete_length = avail_space * 1024
  → table_stats.cc：obj->set_data_free(stats.delete_length)  → DATA_FREE 列

information_schema.FILES.FREE_EXTENTS
  → files.cc 视图定义：FREE_EXTENTS = INTERNAL_TABLESPACE_FREE_EXTENTS(...)
  → Item_func_internal_tablespace_free_extents::val_int()
    → retrieve_tablespace_statistics()
      → innobase_get_tablespace_statistics()
        → stats->m_free_extents = space->free_len            → FREE_EXTENTS 列
        → stats->m_total_extents = space->size_in_header / extent_pages
```

> 关键命名陷阱：InnoDB 填的是 `ha_statistics::delete_length`（头文件注释即 "Free bytes"），server 层在 `table_stats.cc` 用 `set_data_free(stats.delete_length)` 映射成 `DATA_FREE`。**引擎的 `delete_length` 就是外部看到的 `DATA_FREE`**，一个东西两个名字。

### 核心计算：fsp_get_available_space_in_free_extents

```cpp
uintmax_t fsp_get_available_space_in_free_extents(const fil_space_t *space) {
  ulint size_in_header = space->size_in_header;              // 表空间头记录的 size（页数）
  if (size_in_header < FSP_EXTENT_SIZE) return 0;            // 不足一个 extent 的小表：报 0

  // free_limit 以上的页还没初始化，按 64 页/extent 切
  ulint n_free_up = (size_in_header - space->free_limit) / FSP_EXTENT_SIZE;
  if (n_free_up > 0) {
    n_free_up--;                                             // 第一个 extent 是 XDES 描述页
    n_free_up -= n_free_up / (page_size.physical() / FSP_EXTENT_SIZE); // 每 256MB 一组再让出 XDES
  }

  // 预留：2 extent + 总 extent 数的 1%，给 undo 段和清理操作
  ulint reserve = 2 + ((size_in_header / FSP_EXTENT_SIZE) * 2) / 200;
  ulint n_free = space->free_len + n_free_up;                // free extent 链表 + 未初始化部分
  if (reserve > n_free) return 0;

  return (n_free - reserve) * FSP_EXTENT_SIZE * (page_size.physical() / 1024); // 单位 KiB
}
```

逐段解释：

1. **`size_in_header < FSP_EXTENT_SIZE` 返回 0**：表太小、连一个 extent（64 页）都不满时，没有"整块空闲 extent"可言，直接报 0。这也是为什么新表 `DATA_FREE` 常常是 0。
2. **`n_free_up`**：`free_limit` 是表空间已初始化到哪个页的边界，它之上的页还没被任何段用掉，理论上都能切成完整 extent 继续分配。减 1 是因为第一个 extent 的首页要做 XDES（extent descriptor）页；再按 `page_size / FSP_EXTENT_SIZE`（16KB 页 = 256）减一组，因为每 256 个 extent（256MB）构成一个"区组"，组首 extent 的首页也是 XDES 页。
3. **`reserve`**：保守预留 `2 extent + 1%`，给 undo 段和清理（cleaning/purge）操作兜底，防止上层拿到的"空闲量"误导写入。
4. **结果单位 KiB**：`FSP_EXTENT_SIZE`（64 页）× 每页 KiB 数，所以调用方再 `* 1024` 转字节。

### FREE_EXTENTS：更"纯"的视角

```cpp
// innobase_get_tablespace_statistics (ha_innodb.cc)
stats->m_free_extents = space->free_len;                                  // 直接就是链表长度
stats->m_total_extents = space->size_in_header / extent_pages;
stats->m_data_free = fsp_get_available_space_in_free_extents(space) * 1024;
```

`FREE_EXTENTS` 不加预留、不掺未初始化区，就是 free extent 链表长度。想要"到底空闲了多少个完整 extent"时用它最直接；要"预估还能写多少字节"用 `DATA_FREE`。

---

## 相关的系统变量/状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_file_per_table` | ON | Global | 决定碎片是否隔离到每个 `.ibd`；ON 时重建表才能收缩单表文件，OFF 时表数据进共享表空间 `ibdata1`，无法按表收缩 |
| `innodb_page_size` | 16384 | Global | 决定一个 extent 的字节数（64 页 × page_size），间接影响 `DATA_FREE` 的粒度与 `XDES` 分组阈值 |

> `MERGE_THRESHOLD` 等页内合并参数属于页内碎片机制，本篇不展开，后续补充。

---

## Misc

### DATA_FREE / FREE_EXTENTS / 页内碎片 三者辨析

| 对象 | 单位 | 口径 | 是否估算 |
|------|------|------|----------|
| `TABLES.DATA_FREE` | 字节 | free extent + 未初始化区，扣 2 extent + 1% 预留 | 是（估算可写空间） |
| `FILES.FREE_EXTENTS` | extent 个数 | 仅 free extent 链表长度 | 否（精确链表长度） |
| 页内空隙（fill factor） | — | B-tree 页 15/16 填充预留 | 不对外暴露 |

### 易混淆命名

- `ha_statistics::delete_length`（"Free bytes"）== 外部 `DATA_FREE`，**不是**"删除的行数/字节数"；
- `FREE_EXTENTS`（8.0 `FILES` 视图）与 5.7 的 `FREE_EXTS`（`INNODB_SYS_TABLESPACES`）是同一概念，8.0 改名并迁到 `FILES`。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → INFORMATION_SCHEMA TABLES Table*（`DATA_FREE` 列说明）
- *MySQL 8.0 Reference Manual → INFORMATION_SCHEMA FILES Table*（`FREE_EXTENTS` / `TOTAL_EXTENTS` 列说明）

**相关文档**
- 表空间物理组织（extent / segment / fsp / fseg 分配）后续补充
- B-tree 页内合并与 `MERGE_THRESHOLD` 后续补充
