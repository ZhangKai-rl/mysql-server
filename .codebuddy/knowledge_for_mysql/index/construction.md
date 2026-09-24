# 索引的构造：KEY 类与从 DDL 到 `dict_index_t` 的链路

> 基于 MySQL 8.0.39 源码。本篇讲 **server 层 KEY 类的完整对象模型**，以及**索引对象从 SQL 语句到引擎内存对象的完整构造链路**：`Key_spec`（解析器）→ `KEY`/`KEY_PART_INFO`（server 中间表示）→ `dd::Index`（持久化）→ 开表回读 → `ddl::Index_defn` → `dict_index_t`（引擎）。
>
> **边界**：`dd::Index` ↔ `dict_index_t` 的元数据映射细节见 [`metadata.md`](metadata.md)；索引 DDL 的执行（COPY/INPLACE/INSTANT 三算法、row log）见 [`operations.md`](operations.md)；`dict_index_t` 的构建细节（`dict_index_build_internal_*`）见 [`metadata.md`](metadata.md)。本篇聚焦**KEY 类本身**与**构造链路的三条路径**。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [KEY：server 层索引表示的全部成员](#keyserver-层索引表示的全部成员)
    - [KEY_PART_INFO 与两级 flags 体系](#key_part_info-与两级-flags-体系)
    - [`Key_spec`：解析器产出](#key_spec解析器产出)
  - 构造路径
    - [CREATE TABLE：`prepare_key` → KEY → `dd::Index`](#create-tableprepare_key--key--ddindex)
    - [ALTER ADD INDEX：反解合并 → `ddl::Index_defn`](#alter-add-index反解合并--ddlindex_defn)
    - [开表回读：DD → KEY（`fill_index_from_dd` + 后处理）](#开表回读dd--keyfill_index_from_dd--后处理)
  - 运行期
    - [TABLE 层拷贝与 rec_per_key](#table-层拷贝与-rec_per_key)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

KEY 是**索引在 server 层的 SQL 表示**（`sql/key.h` 的 `KEY` + `KEY_PART_INFO`），是"SQL 语句里的索引定义"与"引擎里的 `dict_index_t`"之间的**共享中间表示**：建表时由解析产物 `Key_spec` 构造、开表时由 DD 回读构造；引擎（`handler` 接口、`ddl::Index_defn`）和优化器（`records_per_key`、`usable_key_parts`）都直接消费它。

### 用途

- 建表/加索引：DDL → KEY → DD → 引擎；
- 开表：DD → KEY → TABLE_SHARE/TABLE；
- 优化器：ref 代价估算、`part_of_sortkey` 判定、ICP/MRR 能力判定。

### 版本演进

- **5.x 的独立存储层 `struct KEY_PART` 已消失**：8.0 中 `KEY_PART_INFO` 本身就是传给引擎的接口（全树搜 `struct KEY_PART` 只在 range optimizer 有一个同名无关结构）；引擎侧再转成 `ddl::Index_field`。

---

## 理论基础

### 设计思想与权衡

**1. KEY 是"事实来源的投影"而非事实来源。** 建表时 KEY 从语句构造、开表时从 DD 构造——同一结构两处来源，导致**构造期与开表期的字段填充不对称**（见 Misc 的不对称表）。理解 KEY 必须记住它总在某个"阶段"。

**2. 两级 flags 的分工**：`KEY::flags` 描述"键是什么"（唯一/全文/空间/生成），`KEY_PART_INFO::key_part_flag` 描述"段是什么"（降序/前缀/变长/BLOB）。`actual_flags` 是第二份"实际属性"——开表时二级索引追加了隐藏 PK part 后 `actual_flags |= HA_NOSAME`，而 `flags` 不变（DD 里是 IT_MULTIPLE，运行期"实际唯一"）。

**3. 构造期的字段缺失是刻意的**：CREATE 阶段 `null_bit/null_offset/store_length` 不填（开表后处理才补算）——因为建表路径不需要精确记录布局，省一次全量计算；临时表路径才用 `init_from_field` 现场补齐。

**4. 主键提升不重命名 KEY**：唯一键被提升为主键时，DD 里 `dd::Index` 类型仍为 `IT_UNIQUE`（列标 `CK_PRIMARY`），开表侧才把 `share->primary_key` 指向它——`SHOW CREATE` 仍显示 `UNIQUE KEY`（只有 `name == "PRIMARY"` 才打印 `PRIMARY KEY`）。

---

## 核心实现

### 主线与基础构件

#### KEY：server 层索引表示的全部成员

`sql/key.h:114-358`，全部成员：

| 成员 | 语义 | 构造期/开表期的关键差异 |
|---|---|---|
| `name` | 索引名 | 建表：PK 强制 `"PRIMARY"`，匿名自动命名（首列名） |
| `key_part` | `KEY_PART_INFO` 数组指针 | 建表 `sql_calloc` 分配；开表 `mem_root.ArrayAlloc` |
| `flags` | 键属性位图（**只描述用户定义 part**） | 见下节 |
| `actual_flags` | **实际**（含隐藏扩展 part）属性 | 建表 `= flags`；开表可被 `add_pk_parts_to_sk` 追加 `HA_NOSAME` |
| `user_defined_key_parts` | 用户可见 part 数 | 开表 = 跳过 hidden 元素计数 |
| `actual_key_parts` | 含扩展 PK part 的总数 | 开表 `add_pk_parts_to_sk` 递增 |
| `unused_key_parts` | 为扩展 PK 预留未用的槽 | 仅开表（预置 `primary_key_parts`） |
| `usable_key_parts` | 可用于 ref 的前缀 part 数 | 遇降序段即停（`setup_key_part_field`） |
| `key_length` | 键总字节长（含 null 标记与长度字节） | 建表只累加 part 长度；开表后处理补 null/blob 字节 |
| `block_size` / `algorithm` / `is_algorithm_explicit` | KEY_BLOCK_SIZE / 算法 | `algorithm` 是 `enum ha_key_alg` |
| `parser` / `parser_name` | FTS 解析器插件 | 开表时插件锁定 |
| `rec_per_key` / `rec_per_key_float` | 每前缀"平均记录数"双数组 | 见运行期节 |
| `is_visible` | 优化器可见性 | → `share->visible_indexes` |
| `table` | 所属 TABLE（仅 TABLE 层拷贝置位） | |
| `comment` / `engine_attribute` / `secondary_engine_attribute` | 注解/引擎属性 | |
| `is_functional_index()` | 功能索引判定（遍历 part 的 `field->is_field_for_functional_index()`） | key.cc:50-57 |
| `m_in_memory_estimate` | 索引驻留内存比例（info_low 填入） | |

> 不存在：`KEY::uses_uint8_key`、`free_key()`、`TABLE_SHARE::new_keys()`——KEY 数组随 `share->mem_root` 整体释放；**不存在克隆 KEY 的 `key_copy` 重载**（克隆是 TABLE 打开时的 memcpy）。

#### KEY_PART_INFO 与两级 flags 体系

```cpp
class KEY_PART_INFO { /* sql/key.h:57-91 */
  Field *field;
  uint offset;          /* 记录内偏移 */
  uint null_offset;     /* null_bit 所在字节偏移 */
  uint16 length;        /* 段字节长（不含 null 标记与长度字节） */
  uint16 store_length;  /* length + null(1) + blob 长度(2) */
  uint16 fieldnr;       /* 列序号（UNIREG，1-based） */
  uint16 key_part_flag; /* 0 或 HA_REVERSE_SORT */
  uint8 type;           /* ha_base_keytype */
  uint8 null_bit;       /* 字节内的 null 位 */
  bool bin_cmp;         /* 可否平凡二进制比较 */
  void init_from_field(Field *fld);   // 临时表/排序键专用
};
```

`key_part_flag` 全部位：`HA_SPACE_PACK`(1, MyISAM)、`HA_PART_KEY_SEG`(4, 前缀段)、`HA_VAR_LENGTH_PART`(8)、`HA_NULL_PART`(16, **仅 MyISAM 内部 keyseg**——server 用 `null_bit/null_offset` 表达可空)、`HA_BLOB_PART`(32)、`HA_REVERSE_SORT`(128, 降序)、`HA_BIT_PART`(1024)。

`KEY::flags` 全部位（关键几个）：`HA_NOSAME`(1)、`HA_PACK_KEY`/`HA_BINARY_PACK_KEY`（**唯一显式持久化进 DD options "flags" 的位**，其余可由 DD 对象属性重建）、`HA_FULLTEXT`(1<<7)、`HA_SPATIAL`(1<<10)、`HA_GENERATED_KEY`(1<<13, FK 自动生成)、`HA_KEY_HAS_PART_KEY_SEG`(1<<16, **SQL 层内部位不入 DD**)、`HA_KEY_RENAMED`(1<<17, ALTER 改名标记)、`HA_VIRTUAL_GEN_KEY`(1<<18)、`HA_MULTI_VALUED_KEY`(1<<19, **仅开表设置**——建表阶段由 `count_keys` 校验但不在 KEY 置位)。

#### `Key_spec`：解析器产出

`sql/key_spec.h`：`Key_spec`（209-240）含 `keytype type`（KEYTYPE_PRIMARY/UNIQUE/MULTIPLE/FULLTEXT/SPATIAL）、`KEY_CREATE_INFO key_create_info`（algorithm/is_algorithm_explicit/block_size/parser_name/comment/is_visible）、`columns`（`Key_part_spec` 列表）、`name`、`generated`（FK 自动生成）。`Key_part_spec` 含 `m_is_ascending`、`m_prefix_length`、`m_expression`（功能索引表达式）。

---

### 构造路径

#### CREATE TABLE：`prepare_key` → KEY → `dd::Index`

`mysql_prepare_create_table`（`sql_table.cc:7941`）里索引相关六步：

1. `prepare_create_field` 所有列；
2. **功能索引表达式解析 → 添加隐藏生成列**到 create_list（`add_functional_index_to_create_list`，7763）——此后 `fieldnr` 把隐藏 gcol 计为普通列；
3. `calculate_field_offsets`（8101）——此后 part 的 `offset` 才有意义；
4. `count_keys`（4782-4858）：计数、剔冗余 FK 键、**校验引擎能力**（`HA_DESCENDING_INDEX` 于 4832、多值于 4836-4842）；
5. `sql_calloc` 分配 KEY 数组（8165）；
6. 每个非 FK `Key_spec` 调 `prepare_key`（7183），再 `sort_keys`（PK 排第一）。

**`prepare_key`** 要点：PK 强制命名 `"PRIMARY"`；按 `key->type` 置 `HA_NOSAME/HA_FULLTEXT/HA_SPATIAL`；`user_defined_key_parts = actual_key_parts = columns.size()`；`usable_key_parts` 赋 key_number 占位（开表重算）；循环调 `prepare_key_column`；最后 `actual_flags = flags`。

**`prepare_key_column`**（4860-5276）每个 part 的计算：按名在 create_list 找 `Create_field`（功能索引跳隐藏列）→ `fieldnr = field`、`offset = sql_field->offset`、**降序** `key_part_flag |= HA_REVERSE_SORT`（5134，唯一写入点）→ 前缀截断/校验 → `length` → `HA_PACK_KEY/HA_BINARY_PACK_KEY` → 前缀段置 `HA_PART_KEY_SEG` + 键级 `HA_KEY_HAS_PART_KEY_SEG` → `key_length += key_part_length`。

> CREATE 阶段**不设置** `null_bit/null_offset/store_length/type/bin_cmp`——开表后处理才补算。

**KEY → dd::Index**（`fill_dd_indexes_from_keyinfo`，`dd_table.cc:968`）：逐 KEY 建 `dd::Index`（name/algorithm/visible/type/comment）；`dd_get_new_index_type` 靠 `name == "PRIMARY"` 判 `IT_PRIMARY`；elements 由 `fill_dd_index_elements_from_key_parts`（834-924）填充：**`set_length(key_part->length)`、降序 `set_order(ORDER_DESC)`**；**主键提升发生在这里**——`is_candidate_primary_key` 选中后把列标 `CK_PRIMARY`，但 `dd::Index` 类型仍 `IT_UNIQUE`。

#### ALTER ADD INDEX：反解合并 → `ddl::Index_defn`

1. `mysql_prepare_alter_table` → `prepare_fields_and_keys`（14642）：把**旧表每个 KEY 反解为 `Key_spec`**（15062-15075：`HA_REVERSE_SORT → ORDER_DESC`；功能索引 part 用 `gcol_info->expr_item` 重建表达式），与新 `alter_info->key_list` 合并；
2. 后续**完全复用 CREATE 路径**（`create_table_impl` → `mysql_prepare_create_table`）产出新 KEY 数组；
3. `fill_alter_inplace_info`（11933）新旧 KEY 对比：新键不在旧表 → `add_added_key`（进 `index_add_buffer`）；
4. InnoDB inplace：`prepare_inplace_alter_table_dict` → `innobase_create_key_defs` → **`innobase_create_index_def`**（`handler0alter.cc:2618`，KEY → `ddl::Index_defn`）：

```cpp
index_def->m_n_fields = n_fields;              // user_defined_key_parts
index_def->m_name = mem_heap_strdup(heap, key->name);
if (key_clustered) index_def->m_ind_type = DICT_CLUSTERED | DICT_UNIQUE;
else if (key->flags & HA_FULLTEXT) index_def->m_ind_type = DICT_FTS;
else if (key->flags & HA_SPATIAL)  index_def->m_ind_type = DICT_SPATIAL;
else index_def->m_ind_type = (key->flags & HA_NOSAME) ? DICT_UNIQUE : 0;
for (i = 0; i < n_fields; i++) {
  innobase_create_index_field_def(altered_table, &key->key_part[i],
                                  &index_def->m_fields[i], new_clustered);
  if (index_def->m_fields[i].m_is_v_col) index_def->m_ind_type |= DICT_VIRTUAL;
}
```

`innobase_create_index_field_def`（2544）：`m_is_multi_value = innobase_is_multi_value_fld(field)`、`m_is_ascending = !(key_part_flag & HA_REVERSE_SORT)`、`m_col_no = fieldnr - 虚拟列数`。最终 `ddl::create_index` 构造 `dict_index_t`。

**COPY 算法**则直接 `handler::create(..., key_info, key_count)` 把 KEY 数组传给引擎；**内部临时表**不经过上述链，在 `create_tmp_table` 用 `KEY_PART_INFO::init_from_field` 现场构造（此时补全 `null_bit/store_length/type/bin_cmp`）。

#### 开表回读：DD → KEY（`fill_index_from_dd` + 后处理）

分配（`fill_indexes_from_dd`，`dd_table_share.cc:1465`）：统计 keys/key_parts（**跳过 hidden 索引与 hidden 元素**）；**为扩展 PK 预留空间**；`mem_root.ArrayAlloc<KEY>` + key_part + 双 rec_per_key 数组。

`fill_index_from_dd`（1274-1427）关键：

```cpp
keyinfo->algorithm = dd_get_old_index_algorithm_type(idx_obj->algorithm());
keyinfo->is_visible = idx_obj->is_visible();
keyinfo->user_defined_key_parts = 0;
for (const dd::Index_element *idx_ele : idx_obj->elements())
  if (!idx_ele->is_hidden()) keyinfo->user_defined_key_parts++;
switch (idx_obj->type()) {            // IT_MULTIPLE→0; IT_FULLTEXT→HA_FULLTEXT;
  ...                                 // IT_SPATIAL→HA_SPATIAL; PRIMARY/UNIQUE→HA_NOSAME
}
```

元素回填（`fill_index_element_from_dd`，1210-1243）：`length`、`fieldnr = column().ordinal_position()`、`field = share->field[fieldnr-1]`、**`offset = field->offset(share->default_values)`（绝对偏移，含 null 头——与建表期的列内偏移不同！）**、`bin_cmp` 判定、`ORDER_DESC → HA_REVERSE_SORT`；`field->is_array() → flags |= HA_MULTI_VALUED_KEY`（1263-1264，多值的**唯一**设置点）。

后处理（`open_table_share` 内，305-467）：候选唯一键提升为主键（324-344）；**逐 part 补算 null/blob 信息**（349-376：`null_offset/null_bit/store_length/key_length += HA_KEY_NULL_LENGTH` 等）；`setup_key_part_field`（table.cc:723：置 Field key 位图、算 `usable_parts` 遇降序段停）；`actual_flags = flags`（440）；**`add_pk_parts_to_sk`**（table.cc:801-867：把不在二级键中的 PK part 追加为隐藏 part，`actual_key_parts++`、`unused_key_parts--`，可行时 `actual_flags |= HA_NOSAME`）。

---

### 运行期

#### TABLE 层拷贝与 rec_per_key

**TABLE::key_info 与 TABLE_SHARE::key_info 不共享而是拷贝**：`open_table_from_share` → `create_key_part_field_with_prefix_length`（`table.cc:2793`）在 TABLE 的 mem_root 上 memcpy 整个 KEY+KEY_PART_INFO 数组，把 `key_part->field` 重指到 TABLE 自己的 field、前缀 part 生成"截断字段"、`key_info->table = table`。但 **rec_per_key 数组内存仍在 share->mem_root 上**（TABLE 拷贝的只是指针）——统计更新跨同一 share 的多个 TABLE 实例共享。

**填入**（`ha_innobase::info_low`，`HA_STATUS_CONST` 分支，`ha_innodb.cc:17412`）：

```cpp
if (!key->supports_records_per_key()) continue;   // 两数组皆非空才继续
for (j = 0; j < key->actual_key_parts; j++) {
  if (key->flags & (HA_FULLTEXT | HA_SPATIAL)) { key->set_records_per_key(j, 1.0f); continue; }
  rec_per_key = innodb_rec_per_key(index, j, index->table->stat_n_rows);
  key->set_records_per_key(j, rec_per_key);
  /* 兼容旧接口：ulong 数组故意除以 2（认为选择性被低估） */
}
```

`innodb_rec_per_key` = `records / n_diff`（`stat_n_diff_key_vals`）。`supports_records_per_key()` 在**建表/ALTER 准备期的 KEY 数组上为 false**（从未 `set_rec_per_key_array`）——InnoDB 对 altered_table 调 info_low 时跳过。**消费点**：经典优化器 ref 代价（`sql_select.cc:5133`）、扇出（`sql_planner.cc:462`）、新代价模型（`cost_model.cc:372`）、BKA、MRR 代价。

---

## 相关的系统变量/状态变量

| 项 | 说明 |
|------|------|
| `share->keys / primary_key` | 索引总数 / 主键号（可能指向被提升的唯一键） |
| `share->visible_indexes` | 不可见索引位图（`is_visible` 汇总） |
| `HA_DESCENDING_INDEX` / `HA_MULTI_VALUED_KEY_SUPPORT` | 引擎能力位（`count_keys` 校验） |
| `key_length` | 键总字节长（含 null 标记与长度字节） |
| `rec_per_key` / `rec_per_key_float` | 每前缀平均记录数（引擎统计 → 优化器代价） |

---

## Misc

### 构造期 vs 开表期的字段不对称表

| 字段 | CREATE 阶段（`prepare_key_column`） | 开表阶段（`fill_index_from_dd` + 后处理） |
|---|---|---|
| `offset` | 列内偏移（不含 null bitmap 头） | **绝对偏移**（`field->offset(default_values)`，含 null 头） |
| `null_bit` / `null_offset` | **不填** | 后处理补算 |
| `store_length` | 不填 | `length + null + blob 长度` |
| `type` / `bin_cmp` | 不填 | 填 |
| `HA_MULTI_VALUED_KEY` | **不置位**（只校验） | `field->is_array()` 推导置位 |
| `usable_key_parts` | 占位赋 key_number | `setup_key_part_field` 实算（遇降序停） |

### 易混淆点（本轮核实的事实）

- **不存在 `struct KEY_PART`（存储层）**——8.0 里 `KEY_PART_INFO` 就是传给引擎的接口；引擎侧转成 `ddl::Index_field`。
- **不存在 `create_key_infos` / `fill_dd_indexes_for_keyinfo`**——实际是 `prepare_key`/`prepare_key_column` 与 `fill_dd_indexes_from_keyinfo`。
- **不存在 `KEY::uses_uint8_key`、`free_key`、`TABLE_SHARE::new_keys`**——内存随 mem_root 释放。
- **主键提升不重命名 KEY**：DD 里 `dd::Index` 类型仍 `IT_UNIQUE`，`SHOW CREATE` 显示 `UNIQUE KEY`；只有 `name == "PRIMARY"` 才显示 `PRIMARY KEY`。
- **`HA_NULL_PART` 是 MyISAM 内部 keyseg 位**——server 的 KEY_PART_INFO 不置它（可空由 `null_bit/null_offset` 表达）。
- **`HA_PACK_KEY`/`HA_BINARY_PACK_KEY` 是唯一显式持久化进 DD options 的 flags**（其余位由 DD 对象属性重建）；但 **InnoDB 不实现 pack key**（见 [`record_format.md`](record_format.md)）。
- **`HA_MULTI_VALUED_KEY` 只在开表设置**——建表阶段仅校验。

### 一句话总结

KEY 类是整个索引构造链的**公共车站**：三条构造路径（DDL 语句、ALTER、开表回读）在这里交汇，两份消费方（DD 持久化、引擎 `dict_index_t`）从这里出发。理解它要抓住两件事：**构造期与开表期的字段填充不对称**（同一结构两个来源），**`flags` vs `actual_flags` 的双层属性**（DD 语义 vs 运行期真实形态——隐藏 PK part 使二级索引"实际唯一"）。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 15.1.20.9 Secondary Indexes and Generated Columns*（功能索引）
- *MySQL 8.0 Reference Manual → 13.1.8 CREATE INDEX*

**相关文档**
- `dd::Index` ↔ `dict_index_t` 映射：[`metadata.md`](metadata.md)
- 索引 DDL 执行：[`operations.md`](operations.md)
- 索引记录格式（KEY 层描述的下游字节形态）：[`record_format.md`](record_format.md)
- 引擎侧 `dict_index_t` 构建（`dict_index_build_internal_*`）：[`metadata.md`](metadata.md)
- 索引类型语义：[`types.md`](types.md)
- 能力位与覆盖索引：[`access.md`](access.md)
