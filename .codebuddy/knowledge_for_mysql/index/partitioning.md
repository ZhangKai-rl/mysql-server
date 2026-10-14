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
    - [★ key_restore：从 key_buff 二进制还原分区字段值](#-key_restore从-key_buff-二进制还原分区字段值)
    - [判例一：ref 等值下分区号漂移 → 静默归零](#判例一ref-等值下分区号漂移--静默归零-0-行真实-bug-模式)
    - [判例二：绑定标记残留 → 跨语句污染](#判例二绑定标记残留--跨语句污染社区-bug-121400)
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

### ★ "唯一键必须含分区键"这个约束是怎么推出来的

第 2 条说了结论（唯一键必须包含全部分区函数字段，否则 DDL 期报 `ER_UNIQUE_KEY_NEED_ALL_FIELDS_IN_PF`）。完整的推理链是这样的——它不是"限制"，而是一次**退化**：

```
本地索引：每个分区一棵独立树，只保证【分区内】唯一
   ↓ 问题：全局唯一怎么保证？
若要全局唯一，必须在【跨分区】范围内检查（每次 DML 都要查所有分区 → 破坏分区独立性）
   ↓ 退化手段：让"相同的唯一键值"必然落在【同一个分区】
若唯一键 ⊇ 分区键，则两行的唯一键值相同 ⇒ 分区键值也相同 ⇒ 分区函数算出同一个分区
   ↓ 结论
分区内唯一 == 全局唯一 ✓
```

★ **关键一步是"唯一键 ⊇ 分区键 ⇒ 同值必同分区"**：因为分区键的值决定了行落在哪个分区，只要唯一键包含了分区键，拥有相同唯一键值的行就只能去同一个分区——于是"分区内唯一"自动等价于"全局唯一"，不需要任何跨分区协调。

这也解释了为什么这个限制**无法绕过**：它是本地索引模型下保证全局唯一的**唯一廉价手段**。

### ★ 为什么社区没有全局索引（8.0.39 无）

全局索引 = 一棵跨所有分区的共享树，能直接保证全局唯一。8.0.39 **没有**（无语法、无 flag、无 DD 实体）。为什么不做：

| 约束 | 说明 |
|---|---|
| **与本地索引的独立性冲突** | 全局索引让"分区独立"失效：`DROP PARTITION`、`TRUNCATE PARTITION`、`EXCHANGE PARTITION` 这些**分区级瞬间操作**都要回头维护全局索引（删除一行要在全局树里删），退化成与数据量成正比的操作 |
| **分区级 DDL 的价值正是"只动一个分区"** | 全局索引会让分区表的最大优势（分区级维护）消失，那分区表就没意义了 |
| **本地索引 + 含分区键已经能覆盖大多数场景** | 上面的退化手段已解决唯一性；代价只是唯一键要带上分区键（多数业务唯一键本来就含时间/租户这类分区键） |

★ 所以"没有全局索引"是**分区表设计目标的必然结果**，不是待办——这也意味着：**需要跨分区唯一、又不方便把分区键放进唯一键的场景，MySQL 分区表不适合**（应改用分库分表或应用层保证）。

### ★ 为什么统计只取最大分区（深化）

`HA_STATUS_CONST` 的索引统计**不求和、不加权，只取最大分区**（见「统计聚合」节与 [`../feat/partitioning.md`](../feat/partitioning.md)）。为什么：

- 约束：跨分区合成 `n_diff`（不同值数）在数学上不成立——两个分区各自有 100 个不同值，合并后可能是 100（完全重叠）也可能是 200（完全不重叠），没有额外信息无法判断；
- 源码 TODO 自陈这一取舍："Only analyze the PK for all partitions, then the secondary indexes only for the largest partition!"；
- 后果：**分区数据分布不均时，优化器拿到的是"最大分区的基数"，其他分区的估算全部失真**——这是分区表计划突变的经典根因（见 [`stats.md`](stats.md)）。

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

#### ★ key_restore：从 key_buff 二进制还原分区字段值

`get_full_part_id_from_key` 不是直接对 key 字节跑分区函数，而是**先把 key_buff 的二进制"解包"回记录字段**，再让分区函数从 record 字段读值。完整函数：

```cpp
void get_full_part_id_from_key(const TABLE *table, uchar *buf, KEY *key_info,
                               const key_range *key_spec,
                               part_id_range *part_spec) {
  bool result;
  partition_info *part_info = table->part_info;
  uchar *rec0 = table->record[0];
  longlong func_value;
  DBUG_TRACE;

  key_restore(buf, key_spec->key, key_info, key_spec->length);
  // ★ key_restore = Field::unpack 的批量包装：把 key_buff 二进制按 key
  //   布局还原成记录格式，写进 buf（ref 扫描时 buf 就是 record[0]）
  if (likely(rec0 == buf)) {
    result = part_info->get_partition_id(part_info, &part_spec->start_part,
                                         &func_value);
  } else {
    Field **part_field_array = part_info->full_part_field_array;
    set_field_ptr(part_field_array, buf, rec0);   // 临时把分区字段指针指到 buf
    result = part_info->get_partition_id(part_info, &part_spec->start_part,
                                         &func_value);
    set_field_ptr(part_field_array, rec0, buf);   // 恢复
  }
  part_spec->end_part = part_spec->start_part;
  if (unlikely(result)) part_spec->start_part++;
  // ★ 分区函数计算失败（如值越界）→ start++ 制造 start > end → 空集
}
```

逐段解释：

1. **`key_restore`**：把 `key_spec->key`（= `TABLE_REF::key_buff`）的**二进制 key 格式**按 `key_info` 的字段布局 unpack 成 record 格式写进 `buf`。ref 扫描时 `buf` 就是 `record[0]`，所以接下来的分区函数直接读 record 字段。
2. **`rec0 == buf` 的两种形态**：`buf == record[0]` 时直接算；否则（range 扫描用临时 buffer）用 `set_field_ptr` 临时把 `full_part_field_array` 的分区字段指针指到 `buf`，算完恢复——分区函数的 Item 树不重建，只换数据源指针。
3. **`result → start_part++`**：分区函数返回错误（值不在任何分区/越界）时，`start_part` 自增超过 `end_part`，**制造空集**——这是**第二条静默归零路径**（不经过 `prune_partition_set`，直接在"算出分区"这一步把结果变成空）。

**这条"key_buff 二进制 → key_restore 解包 → record 字段 → 分区函数"的链**意味着：**key_buff 内容的任何偏差都会直接变成分区号偏差**。而 key_buff 是 ref 访问的 `store_key::copy()` 写的——它的静默失败语义（截断被洗白成 OK、常量 part `(void)` 扔返回值、分配即脏）见 [`../server/query/07_optimize/physical/07_access_method.md`](../server/query/07_optimize/physical/07_access_method.md)「copy() 的静默语义」一节——两篇互为因果的两半。

#### 判例：ref 等值下分区号漂移 → 静默归零 0 行（真实 bug 模式）

**现象**：`SELECT count(*) FROM t PARTITION (p_x) WHERE tran_date = '<p_x 内的日期>'` 短连接循环执行，count 在 0 与正确值间跳跃，`Handler_read_key` 0/1 交替。

**分区侧的因果链**（两条归零路径 + 一层伪装）：

**路径 A——`prune_partition_set` 求交为空**（完整函数）：

```cpp
void prune_partition_set(const TABLE *table, part_id_range *part_spec) {
  int last_partition = -1;
  uint i = part_spec->start_part;
  partition_info *part_info = table->part_info;

  if (i)
    i = bitmap_get_next_set(&part_info->read_partitions, i - 1);
  else
    i = bitmap_get_first_set(&part_info->read_partitions);

  part_spec->start_part = i;

  for (; i <= part_spec->end_part;
       i = bitmap_get_next_set(&part_info->read_partitions, i)) {
    if (last_partition == -1) part_spec->start_part = i;   // 第一个命中
    last_partition = i;                                     // 持续记录命中
  }
  if (last_partition == -1)                                 // ★ 一个都没命中
    part_spec->start_part = part_spec->end_part + 1;        //   → 制造空集
  else
    part_spec->end_part = last_partition;
}
```

逐段解释：把 `[start_part, end_part]`（get_partition_set 算出的区间）与 `read_partitions` bitmap 求交，收缩成"连续命中区间"。`last_partition == -1` 表示**区间内一个分区都不在 read_partitions 里** → `start = end + 1` 制造空集。**没有任何错误返回**——空集就是结果。

**路径 B——`get_full_part_id_from_key` 的分区函数失败**（上文代码最后一行）：`if (unlikely(result)) part_spec->start_part++;`——漂移值越出所有分区定义时同样制造空集。

**空集 → EOF 伪装"key not found"**（`partition_scan_set_up` 完整函数）：

```cpp
int Partition_helper::partition_scan_set_up(uchar *buf, bool idx_read_flag) {
  if (idx_read_flag)
    get_partition_set(m_table, buf, m_handler->active_index, &m_start_key,
                      &m_part_spec);
  else {
    m_part_spec.start_part = 0;
    m_part_spec.end_part = m_tot_parts - 1;
  }
  if (m_part_spec.start_part > m_part_spec.end_part) {
    /*
      We discovered a partition set but the set was empty so we report
      key not found.
    */
    DBUG_PRINT("info", ("scan with no partition to scan"));
    return HA_ERR_END_OF_FILE;              // ★ 空分区集 → 伪造成"没找到 key"
  }
  if (m_part_spec.start_part == m_part_spec.end_part) {
    m_ordered_scan_ongoing = false;         // 单分区，无需归并
  } else {
    uint start_part = m_part_info->get_first_used_partition();
    if (start_part == MY_BIT_NONE) return HA_ERR_END_OF_FILE;
    if (start_part > m_part_spec.start_part)
      m_part_spec.start_part = start_part;
    m_ordered_scan_ongoing = m_ordered;     // 多分区 ordered scan 归并
  }
  assert(m_part_spec.start_part < m_tot_parts);
  assert(m_part_spec.end_part < m_tot_parts);
  return 0;
}
```

**关键伪装**（`:2231-2238` 注释原文）：分区集为空时返回 `HA_ERR_END_OF_FILE`——与"key 真的不存在"**完全相同的错误码**。上层迭代器只会看到"查无此行"，等价于数据不存在：**0 行、无任何告警**。`Handler_read_key=0` 的观测含义是扫描根本没发起引擎读（EOF 在 setup 就返回了）。

**串起来**：

```
key_buff 脏（store_key 静默失败/分配残留，见 07 篇）
  → key_restore 解包出"漂移的日期值"
  → 路径 A：分区函数算出 P ≠ 实际分区 → prune 求交空 → start=end+1
  → 路径 B：漂移值越出分区定义 → result → start++（同样空集）
  → partition_scan_set_up：start > end → HA_ERR_END_OF_FILE（伪装 key not found）
  → 0 行、无报错（Handler_read_key=0）
```

**`PARTITION (p_x)` 是放大器**：显式分区子句把 `read_partitions` 钉死为单分区位图——漂移的 P 只要 ≠ p_x，路径 A 交集必空；不加子句时 read_partitions 往往覆盖多分区，漂移 P 可能落进其中，返回**别的分区的 0 行**（更隐蔽）。

**规避**：等值改区间（`tran_date >= '...' AND tran_date < '...'`）。`get_partition_set` 的快速路径要求 `flag == HA_READ_KEY_EXACT`（上文代码）——区间条件不走它，改由**优化期 range 裁剪**（见 [`../server/query/07_optimize/12_partition_pruning.md`](../server/query/07_optimize/12_partition_pruning.md) 的 `prune_partitions` 链）基于常量表达式算分区，完全绕开 store_key 的静默路径。

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

#### 判例二：绑定标记残留 → 跨语句污染（社区 bug #121400）

上一个判例的成因是 **key_buff 脏**（值漂移）；这一条的成因完全不同——**不是值错，而是"分区字段是否已绑定"这个状态没被复位**，污染了同一 `TABLE` 对象上的后续查询。两者**出口相同**（空集 → `HA_ERR_END_OF_FILE` 伪装成 key not found），成因独立。

##### 前置知识：index extension（二级索引尾部的隐藏主键列）

InnoDB 的二级索引在**物理上**隐含了主键列——记录尾部追加 PK 的 `n_uniq` 列，用于回表定位：

```cpp
// storage/innobase/dict/dict0dict.cc  (dict_index_build_internal_non_clust)
  new_index = dict_mem_index_create(table->name.m_name, index->name, index->space,
                                    index->type,
                                    index->n_fields + 1 + clust_index->n_uniq);
  new_index->n_user_defined_cols = index->n_fields;
  ...
  /* Add to new_index the columns necessary to determine the clustered
  index entry uniquely */
  for (i = 0; i < clust_index->n_uniq; i++) {
    field = clust_index->get_field(i);
    if (!indexed[field->col->ind]) {
      dict_index_add_col(new_index, table, field->col, field->prefix_len, field->is_ascending);
```

server 层也知道这件事（`HTON_SUPPORTS_EXTENDED_KEYS`），会把 PK 的 keypart 拷到二级索引 `KEY::key_part[]` 的**尾部**，于是出现三个不同的"列数"：

```cpp
// sql/key.h
  /** How many key_parts */
  uint user_defined_key_parts{0};
  /** How many key_parts including hidden parts */
  uint actual_key_parts{0};
  /** Key parts allocated for primary key parts extension but not used ... */
  uint unused_key_parts{0};
```

```cpp
// sql/table.cc  (add_pk_parts_to_sk)
      *current_key_part = *pk_key_part;
      sk->actual_key_parts++;
      sk->unused_key_parts--;
```

优化器按 `actual_key_parts()` 枚举 keypart 并生成 `Key_use`（受 `optimizer_switch` 的 `use_index_extensions` 控制），`calculate_key_len()` 同样按它累加长度：

```cpp
// sql/sql_select.cc
uint actual_key_parts(const KEY *key_info) {
  return current_thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INDEX_EXTENSIONS)
             ? key_info->actual_key_parts
             : key_info->user_defined_key_parts;
}
```

**关键的不对称**：`KEY::key_length` 只统计**非 hidden** 的元素：

```cpp
// sql/dd_table_share.cc
  keyinfo->key_length = 0;
  for (const dd::Index_element *idx_elem : idx_obj->elements()) {
    // Skip hidden index elements
    if (!idx_elem->is_hidden()) keyinfo->key_length += idx_elem->length();
  }
```

于是"ref 绑定了扩展列"时，`key_spec->length > key_info->key_length`——这正是后面走慢路径的判据。

##### 慢路径：`set_PF_fields_in_key` 打标记

`get_partition_set` 快速路径要求 `key_spec->length == key_info->key_length`；不满足就进慢路径，按剩余 `key_length` 逐 keypart 前进，给"参与分区函数的字段"点亮 `GET_FIXED_FIELDS_FLAG`：

```cpp
// sql/sql_partition.cc  (set_PF_fields_in_key)
  for (key_part = key_info->key_part; (int)key_length > 0; key_part++) {   // ★ 无 keypart 上界
    ...
    if (key_part->field->is_flag_set(FIELD_IN_PART_FUNC_FLAG)) {
      found_part_field = true;
      key_part->field->set_flag(GET_FIXED_FIELDS_FLAG);
    }
```

注意循环**没有 keypart 数量上限**——它会一路走完 `key_length` 覆盖到的所有 keypart，**包括扩展列**。如果扩展列恰好是分区键（典型：`PARTITION BY LIST(MOD(pk_col, N))` + 二级索引），标记就打在了隐藏列对应的 `Field` 上。

##### 不对称：唯一的复位入口只清用户定义部分

```cpp
// sql/sql_partition.cc  (clear_indicator_in_key_fields)
static void clear_indicator_in_key_fields(KEY *key_info) {
  uint key_parts = key_info->user_defined_key_parts;                       // ★ 只清用户定义
  for (i = 0, key_part = key_info->key_part; i < key_parts; i++, key_part++)
    key_part->field->clear_flag(GET_FIXED_FIELDS_FLAG);
}
```

它有个配对函数 `set_indicator_in_key_fields()`，两者都用 `user_defined_key_parts`（注释明写 "always used in pairs"），那对是自洽的。但 **`set_PF_fields_in_key()` 没有对称的 clear**——唯一的复位就是上面这个只清用户定义的版本。结果：**扩展列上的 `GET_FIXED_FIELDS_FLAG` 永远清不掉**。

受害点是 `check_part_func_bound`，它只检查 flag 是否点亮，不检查是谁点亮的：

```cpp
// sql/sql_partition.cc
static bool check_part_func_bound(Field **ptr) {
  for (; *ptr; ptr++) {
    if (!(*ptr)->is_flag_set(GET_FIXED_FIELDS_FLAG)) {
      result = false;
      break;
    }
  }
  return result;
}
```

一旦它误判"分区函数的所有字段都已绑定"，就会走 `get_full_part_id_from_key()` 用**当前不完整的键值**算出分区，把 `part_spec` 钉到单分区或空集。

##### 为什么能跨语句残留（TABLE 缓存复用）

`Field` 数组是 **per-TABLE 的克隆**，不是 per-TABLE_SHARE：

```cpp
// sql/table.cc  (open_table_from_share)
  for (i = 0; i < share->fields; i++, field_ptr++) {
    Field *new_field = share->field[i]->clone(root);
    ...
```

而且 TABLE 拿到的是一份私有的 `key_info` 副本，其中 `key_part->field` 改指自己的 Field 克隆。所以 flag 打在 TABLE 私有的 Field 上。

而 table cache **命中时直接复用 TABLE，不重建 Field**：

```cpp
// sql/sql_base.cc  (open_table)
    table = tc->get_table(thd, key, key_length, &share);
  if (table) {
    ...
    table->file->ha_extra(HA_EXTRA_RESET_STATE);   // ★ 只重置 handler，不碰 Field flag
    thd->status_var.table_open_cache_hits++;
    goto table_found;                              // ★ 不走 open_table_from_share
```

`FLUSH TABLES` 之所以能自愈，是因为它销毁 TABLE 对象本身：

```cpp
// sql/sql_base.cc  (close_cached_tables)
    refresh_version++;
    table_cache_manager.free_all_unused_tables();   // 销毁未使用的 TABLE 与其 Field 克隆
```

下次访问 cache miss → 重新 `clone()` Field → flag 回到干净状态。

##### 完整失效链

```
① 二级索引 ref 绑定了 index extension 追加的隐藏 PK 列（该列同时是分区键）
   → calculate_key_len() 按 actual_key_parts 算长度，覆盖到扩展列
② key_spec->length != key_info->key_length（后者只含非 hidden 元素）
   → get_partition_set 走慢路径
③ set_PF_fields_in_key() 无 keypart 上界 → 对扩展列 set_flag(GET_FIXED_FIELDS_FLAG)
④ 唯一复位 clear_indicator_in_key_fields() 只遍历 user_defined_key_parts → 扩展列 flag 残留
⑤ flag 在 per-TABLE 的 Field 克隆上；table cache 复用不重建 Field → 跨语句存活
⑥ 下一条语句：check_part_func_bound() 误判"PF 全绑定"
   → get_full_part_id_from_key() 用不完整键值算分区 → start_part==end_part（单分区）或空集
⑦ 空集 → partition_scan_set_up() 返回 HA_ERR_END_OF_FILE → 伪装成"key not found"
   → 结果变少（1 而非 2）或为 0，且 Handler_read_key 为 0
```

##### 与判例一的对比

| | 判例一（key_buff 脏） | 判例二（标记残留） |
|---|---|---|
| 污染源 | `store_key::copy()` 静默失败 / buffer 分配即脏 | `GET_FIXED_FIELDS_FLAG` 未被复位 |
| 出错环节 | **值**错了（key_restore 解包出漂移值） | **状态**错了（误判字段已绑定） |
| 是否跨语句 | 单语句内（同一 key_buff） | **跨语句**（TABLE 缓存复用） |
| 自愈方式 | 无（需改 key_buff 写入） | `FLUSH TABLES` 重建 TABLE |
| 出口 | 同：空集 → `HA_ERR_END_OF_FILE` 伪装 key not found | 同左 |

两条链说明同一件事：**分区定位失败在 MySQL 里是"静默"的**——空集不报错，只是变成"查无此行"。

##### 观测与规避

- 观测：`Handler_read_key` 与 `Handler_read_next` 均为 0，但表里有数据；同一条 SQL **前后两次执行结果不同**；`FLUSH TABLES` 后恢复正常——这三条同时出现，基本就是这条链。
- 规避一：让 ref 不绑定扩展列（避免把分区键通过 index extension 绑进二级索引查找）。
- 规避二：`SET optimizer_switch='use_index_extensions=off'`——`actual_key_parts()` 退化为 `user_defined_key_parts`，ref 不再延伸进扩展列。
- 规避三：等值改区间，走优化期 range 裁剪，完全绕开 `get_partition_set` 这条执行期路径（同判例一）。

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
