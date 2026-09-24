# 索引 × 分区：本地索引、逐分区构建与裁剪

> 基于 MySQL 8.0.39 源码。本篇讲 **分区表上索引的一切**：本地索引语义（每分区一棵树）与 `dd::Partition_index` 元数据、per-partition `dict_table_t` 与游标切换、唯一键必须含分区键的约束、逐分区的索引 DDL、分区裁剪与索引访问的配合、统计聚合（索引基数**不求和**）、EXCHANGE/TRUNCATE/REORG 的索引处理。
>
> **边界**：分区的整体机制（分区函数/裁剪算法/`partition_info`）见 [`../feat/partitioning.md`](../feat/partitioning.md)；分区裁剪的 range 优化细节见 [`../server/query/07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md)。本篇只取"索引"视角。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [本地索引：`dd::Partition_index` 与 per-partition `dict_table_t`](#本地索引ddpartition_index-与-per-partition-dict_table_t)
    - [全局索引不存在 + 唯一键必须含分区键](#全局索引不存在--唯一键必须含分区键)
  - 读路径
    - [裁剪在前、索引在后；逐分区索引搜索](#裁剪在前索引在后逐分区索引搜索)
    - [等值键的单分区定位与 index dive 求和](#等值键的单分区定位与-index-dive-求和)
  - 写路径与维护
    - [ADD/DROP INDEX：逐分区 inplace + 独立 row_log](#adddrop-index逐分区-inplace--独立-row_log)
    - [统计聚合：行数求和、基数取最大分区](#统计聚合行数求和基数取最大分区)
    - [EXCHANGE / TRUNCATE / DROP / REORG 的索引处理](#exchange--truncate--drop--reorg-的索引处理)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

分区表的索引是**本地索引（local index）**：每个索引在每个分区一棵独立的 B 树，引擎侧每个分区是一个独立的 `dict_table_t`（带自己的 `dict_index_t` 链表、自己的 tablespace）。SQL 层看到的"一个索引"在物理上是 **N 份**（N = 分区数）。

### 用途

- 理解分区裁剪如何与索引访问叠加（先裁剪、后逐分区索引搜索）；
- 理解分区表索引统计的怪癖（基数只取最大分区）；
- 理解分区 DDL（ADD INDEX/EXCHANGE/TRUNCATE）的索引代价。

### 版本演进

- 8.0 的 `ha_innopart`（原生 InnoDB 分区 handler）取代了 5.x 的 `ha_partition`（通用分区壳）——5.x 的 `m_partitions_to_use`、共享索引定义等概念在 8.0 全部消失；
- **8.0.39 无全局索引**（无语法、无 flag、无 DD 实体）。

---

## 理论基础

### 设计思想与权衡

**1. 本地索引 = 把"一个索引"拆成 N 份独立树。** 好处：分区裁剪天然"索引级"生效（不访问的分区连索引树都不碰）、锁/change buffer/purge 按 space 天然隔离；代价：**唯一性只能分区内保证**（跨分区唯一必须靠"唯一键含分区键"这一约束）、DDL 要逐分区做 N 遍、统计聚合有失真。

**2. 唯一性约束是补丁**：因为每个分区树只保证分区内唯一，InnoDB 分区表**唯一键必须包含全部分区函数字段**（`ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF`，DDL 期拒绝）——用"唯一键含分区键"把跨分区唯一性退化成分区内唯一性。

**3. prebuilt 是共享壳、per-partition 状态各自保存**（`set_partition`/`update_partition`）：不为一分区各建一套完整查询上下文，而是切换 `m_prebuilt->table/index` 指针——性能与内存的折中。

**4. 裁剪在索引优化之前**：先决定"哪些分区可能命中"（分区字段伪索引跑 `get_mm_tree`），再做各分区的索引优化——两个优化器解耦，分区裁剪永远先做。

---

## 核心实现

### 主线与基础构件

#### 本地索引：`dd::Partition_index` 与 per-partition `dict_table_t`

**DD 层**：`mysql.index_partitions` 表**每 (分区, 索引) 一行**（`dd::Partition_index`，Weak_object，生命周期跟随 `dd::Partition`），每行携带自己的 `options`/`se_private_data`/`tablespace_id`——**root page no 存在每行的 `se_private_data` 里**（键名与普通索引相同：`"root"`，`dd_index_key_strings[]` 注释明说 "for dd::Index **or dd::Partition_index**"）。

写入点：`dd_write_index` 模板**显式实例化到 `dd::Partition_index`**（`dict0dd.cc:2615-2639`）：

```cpp
template <typename Index>
static void dd_write_index(dd::Object_id dd_space_id, Index *dd_index,
                           const dict_index_t *index) {
  dd_index->set_tablespace_id(dd_space_id);
  dd::Properties &p = dd_index->se_private_data();
  p.set(dd_index_key_strings[DD_INDEX_ROOT], index->page);   // 每分区 root page
  ...
}
template void dd_write_index<dd::Index>(...);
template void dd_write_index<dd::Partition_index>(...);      // 分区索引专用实例
```

**引擎侧**：`Ha_innopart_share` 持 `dict_table_t **m_table_parts`（每分区一个**完全独立**的 InnoDB 表对象，独立 `dict_index_t` 链表、独立 tablespace）+ `dict_index_t **m_index_mapping`（`m_index_count * part_id + keynr` 的 (分区, MySQL 键号) → InnoDB 索引映射）。**没有共享索引定义**。

游标切换核心（`ha_innopart::set_partition`，`ha_innopart.cc:1267-1311`）：

```cpp
void ha_innopart::set_partition(uint part_id) {
  if (m_pcur_parts != nullptr) {
    m_prebuilt->pcur = &m_pcur_parts[m_pcur_map[part_id]];
  }
  m_prebuilt->ins_node = part.m_ins_node;    // 每分区独立 insert node（change buffer）
  m_prebuilt->upd_node = part.m_upd_node;
  m_prebuilt->table = m_part_share->get_table_part(part_id);   // 切表
  m_prebuilt->index = innopart_get_index(part_id, active_index); // 切索引
}
```

配套 `update_partition` 把 per-partition 状态存回 `m_parts[part_id]`。

#### 全局索引不存在 + 唯一键必须含分区键

**全局索引在 8.0.39 完全不存在**（三重证据）：`HA_CAN_GLOBAL_INDEX`/`global_index` 全库 0 命中；`sql_yacc.yy` 的 `GLOBAL_SYM` 只用于权限/SET GLOBAL，索引语法无 GLOBAL 选项；`dd::Index` 无 global 属性。引擎分区能力位（`ha_innodb.cc:4488`）：`HA_CAN_EXCHANGE_PARTITION | HA_CANNOT_PARTITION_FK | HA_TRUNCATE_PARTITION_PRECLOSE`。

**唯一键约束**：`fix_partition_func()`（`sql_partition.cc:1580-1586`）→ `check_primary_key()`（主键无条件强制）/ `check_unique_keys()`（受 `HA_CAN_PARTITION_UNIQUE` 豁免——**只有 NDB 设此 flag**，InnoDB 不设）→ 报 `ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF`（1503）：

```
"A %-.192s must include all columns in the table's partitioning function
 (prefixed columns are not considered)."
```

**后果**：唯一键（含主键）必须包含全部分区函数字段（前缀列不算）；非唯一索引无限制。引擎侧不做跨分区唯一性检查——该约束在 DDL 期挡死。

---

### 读路径

#### 裁剪在前、索引在后；逐分区索引搜索

**顺序已确认**：`JOIN::make_join_plan` 在搜索 join order **之前**调 `prune_table_partitions()`（`sql_optimizer.cc:502`）→ `prune_partitions()`（`partition_pruning.cc:251-326`）内部用**"分区字段伪索引"**（`range_par->using_real_indexes = false`）跑 `get_mm_tree()`——与真实索引优化器解耦。结果 = `part_info->read_partitions` bitmap。

**传给引擎的机制**：8.0 **没有** `set_partition_info`/`m_partitions_to_use`（5.x 概念，全库 0 命中）；对应物是 `part_info->read_partitions` bitmap（遍历循环以它为界）+ 执行期的 `m_part_spec`（`part_id_range`）。

**range 访问 = 对每个分区分别做索引定位**，多分区时 ordered scan 逐分区归并。`partition_scan_set_up`（`partition_handler.cc:2219`）：等值键经 `get_partition_set` 算 `m_part_spec`；`start_part == end_part` → 单分区不需要归并，否则 `m_ordered_scan_ongoing = m_ordered`。

ha_innopart 的逐分区原语族（`ha_innopart.cc:1787-1999`），全部是 `set_partition → ha_innobase::xxx → update_partition` 三明治：`index_read_map_in_part` / `index_next_in_part` / `index_next_same_in_part` / `index_last_in_part` / `index_prev_in_part` / `read_range_first_in_part` / `read_range_next_in_part` 等。

#### 等值键的单分区定位与 index dive 求和

**等值键命中单分区**（执行期、每次扫描 setup 时）：`get_partition_set()`（`sql_partition.cc:3668-3727`）：

```cpp
if ((index < MAX_KEY) && key_spec &&
    key_spec->flag == (uint)HA_READ_KEY_EXACT &&        // 等值查找
    part_info->some_fields_in_PF.is_set(index)) {       // 索引含分区字段（预计算 bitmap）
  ...
  if (part_info->all_fields_in_PF.is_set(index)) {      // 含全部分区字段
    get_full_part_id_from_key(...);                     // ★ 算出唯一分区
    prune_partition_set(table, part_spec);
    return;
  }
```

预计算 bitmap（`set_up_partition_key_maps`，`sql_partition.cc:1187`）：`all_fields_in_PF`/`some_fields_in_PF` 逐键检查分区字段覆盖——EQ_REF/ref 的键覆盖全部分区字段时 `m_part_spec` 收窄为**单分区**（一次 `index_read_map_in_part`），否则回退全分区扫描。**优化器不为每个 probe 值重新裁剪——这是 handler 内的精确裁剪**。

**index dive 逐分区求和**（`ha_innopart::records_in_range`，`ha_innopart.cc:3272-3364`）：

```cpp
n_rows = btr_estimate_n_rows_in_range(index, range_start, mode1, range_end, mode2);
for (part_id = m_part_info->get_next_used_partition(part_id); ...) {
  index = m_part_share->get_index(part_id, keynr);      // 每分区自己的索引
  /* Individual partitions can be discarded we need to check each partition */
  if (index == nullptr || dict_table_is_discarded(index->table) || ...) {
    n_rows = HA_POS_ERROR; ...
  }
  int64_t n = btr_estimate_n_rows_in_range(index, ...);
  n_rows += n;                                          // ★ 逐分区 dive 并求和
}
```

优化器每评估一个 range，就对每个 used 分区下沉 B 树估算并**求和**。

---

### 写路径与维护

#### ADD/DROP INDEX：逐分区 inplace + 独立 row_log

`ha_innopart` 的四个 inplace 入口在 `handler0alter.cc`（10098/10281/10418/10484），核心是"**每分区一份 ctx、每分区一个 prebuilt**，循环调 `ha_innobase` 的模板实现"：

```cpp
// handler0alter.cc:10281-10416（节选）
ctx_parts = new (thd->mem_root) ha_innopart_inplace_ctx(m_tot_parts);
ctx_parts->prebuilt_array[0] = m_prebuilt;
for (uint i = 1; i < m_tot_parts; i++) {
  tmp_prebuilt = row_create_prebuilt(m_part_share->get_table_part(i), ...);
  ctx_parts->prebuilt_array[i] = tmp_prebuilt;           // 每分区独立 prebuilt
}
for (uint i = 0; i < m_tot_parts; ++oldp, ++newp) {
  m_prebuilt = ctx_parts->prebuilt_array[i];
  set_partition(i);
  res = prepare_inplace_alter_table_impl<dd::Partition>(   // 单分区走 ha_innobase 逻辑
      altered_table, ha_alter_info, old_part, new_part);
  update_partition(i);
  ctx_parts->ctx_array[i] = ha_alter_info->handler_ctx;
}
```

- **每分区独立 row_log**（online DDL 状态独立）：impl 内每个新索引 `row_log_allocate` 一次（`handler0alter.cc:4934/4962`），各分区 ctx 独立 → N 份 row_log；
- **commit 只经第一个分区**（group commit）：`commit_inplace_alter_table` 里 `ctx_array[0]` 走 `ha_innobase::commit_inplace_alter_table_impl<dd::Table>`；**rollback 逐分区**；
- 完成后 `m_part_share->set_table_part(i, ctx->prebuilt->table)` 装回新表对象；
- `check_if_supported_inplace_alter`（10098）的限制：FK/FTS 不支持（`ER_FOREIGN_KEY_ON_PARTITIONED`/`ER_FULLTEXT_NOT_SUPPORTED_WITH_PARTITIONING`）；KEY() 分区时 ADD/DROP PK 不支持 INPLACE；`ALTER ... PARTITION` 类按 `alter_parts::need_copy` 分流（RANGE/LIST ADD、DROP 不需拷数据）。

#### 统计聚合：行数求和、基数取最大分区

`ha_innopart::info_low`（`ha_innopart.cc:3484-3789`）三类聚合：

- **行数/页数（`HA_STATUS_VARIABLE`）：逐分区求和**（3559-3572）——`n_rows += ib_table->stat_n_rows`，同时记录 `biggest_partition`；
- **索引统计（`HA_STATUS_CONST`）：不求和、不加权，只取最大分区的索引**（3674-3789）——`stat_n_diff_key_vals[]` 不做跨分区合成，`rec_per_key` 用**最大分区**的行数算（而非总和）。源码 TODO 自陈这一取舍："Only analyze the PK for all partitions, then the secondary indexes only for the largest partition!"

> ★ 分区统计的完整剖析（`info_low` 逐段源码、"分区分布不均 → 计划突变"的后果）见 [`../feat/partitioning.md`](../feat/partitioning.md)——**分区是跨层特性，那里是权威**；统计本身怎么采集（persistent/transient 采样、`n_diff_pfxNN`、基数漂移）见 [`stats.md`](stats.md)。本篇只保留"索引 × 分区"的聚合规则这一层。

**ANALYZE PARTITION**：`set_altered_partitions()`（`partition_handler.cc:1274`）把 `read_partitions` 只置位指定分区 → `update_table_stats` 只跑这些分区；`stats.update_time` 取各分区 max。

**隔离性**：行锁/隐式锁挂在 per-partition `dict_index_t` 上（`set_partition` 切指针后 `row_sel`/`lock` 全部作用于该分区 space）——**锁天然按分区隔离，无跨分区锁**；change buffer 的 `m_ins_node`、purge 按 space 同样每分区独立。

#### EXCHANGE / TRUNCATE / DROP / REORG 的索引处理

| 操作 | 索引处理 |
|---|---|
| **EXCHANGE PARTITION** | 物理文件三次 rename 交换**整棵树**（全部索引随 tablespace 走）+ DD 层逐索引交换 `tablespace_id` 与 `se_private_data`（含 root page，`handler0alter.cc:10993-11036`）；索引按名字排序对齐，`ut_ad(part_indexes.size() == swap_indexes.size())` 校验同构 |
| **TRUNCATE PARTITION** | 每分区 `innobase_truncate<dd::Partition>` **整空间重建**（聚簇 + 全部二级索引树释放重建），autoinc 取各分区 max |
| **DROP PARTITION** | 走 `alter_parts`（`m_to_drop` 记录待删），不拷数据（`HA_ALTER_INPLACE_NO_LOCK_AFTER_PREPARE`）；提交时旧分区 tablespace（含全部索引树）整体删除，`mysql.index_partitions` 行随 DD 事务删 |
| **REORGANIZE/COALESCE** | `need_copy()==true`：新分区提前建**全新的空 `dict_table_t`**（含全部索引树），`copy_partitions()` 逐行插入、**索引随行插入逐棵构建**（不复用旧树）；提交后 drop 旧分区 |

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF`（1503） | 唯一键必须含全部分区字段 |
| `HA_CAN_PARTITION_UNIQUE` | 豁免唯一键检查的引擎 flag（**仅 NDB**） |
| `innopart_partition_flags()` | `HA_CAN_EXCHANGE_PARTITION \| HA_CANNOT_PARTITION_FK \| HA_TRUNCATE_PARTITION_PRECLOSE` |
| `part_info->read_partitions` | 裁剪结果 bitmap（遍历以它为界） |
| `m_part_spec`（`part_id_range`） | 执行期每扫描的分区区间（等值键 = 单分区） |
| `innodb_compression_failure_threshold_pct` 等 | （压缩相关，见 [`physical_storage.md`](physical_storage.md)） |

---

## Misc

### 易混淆点

- **8.0 没有全局索引**（无 flag/无语法/无 DD 实体）；也没有 5.x 的 `m_partitions_to_use`/共享索引定义——每分区独立 `dict_table_t`。
- **索引基数统计不求和**：`rec_per_key` 只取**最大分区**的索引 + `max_rows`——分区表基数可能严重失真（尤其数据分布不均时）。
- **唯一性约束在 DDL 期挡死**（1503），引擎不做跨分区唯一性检查——每分区树只保证分区内唯一。
- **分区裁剪在索引优化之前**，且用"分区字段伪索引"独立跑 range 树——与真实索引优化器完全解耦。
- **EQ_REF 的单分区定位是执行期按当前键值算的**（`get_partition_set`），不是优化器对每个 probe 值重新裁剪。
- **分区表不支持 FK 与 FTS**（`ER_FOREIGN_KEY_ON_PARTITIONED` / `ER_FULLTEXT_NOT_SUPPORTED_WITH_PARTITIONING`）。
- **锁/change buffer/purge 按 space 天然隔离**——per-partition 独立空间是这一切的基础。

### 一句话总结

分区表的索引 = "一个 SQL 索引"投影成"每分区一棵独立树"：DD 用 `dd::Partition_index`（每分区一行、每行一个 root page）表达，引擎用 per-partition `dict_table_t` + prebuilt 指针切换承载，读路径靠"先裁剪、后逐分区索引搜索 + dive 求和"，维护靠"逐分区 inplace/独立 row_log"——而**唯一性只能分区内保证**这一事实，倒逼出"唯一键必须含分区键"的 DDL 约束。理解分区索引的关键是**"N 份独立"**：树独立、空间独立、锁独立、统计独立（但基数只取一份）。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 26.6.1 Partitioning Keys, Primary Keys, and Unique Keys*
- *MySQL 8.0 Reference Manual → 26.2 Partitioning Types*（分区键约束的完整规则）

**相关文档**
- 分区整体机制：[`../feat/partitioning.md`](../feat/partitioning.md)
- 分区裁剪 range 优化：[`../server/query/07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md)
- 索引统计聚合基础：[`stats.md`](stats.md)
- 索引 DDL 执行：[`operations.md`](operations.md)
- `dd::Partition_index` 与 `se_private_data`：[`metadata.md`](metadata.md)
- 索引构造链路：[`construction.md`](construction.md)
