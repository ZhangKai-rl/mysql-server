# 索引类型总览：语义、存储与源码标志

> 基于 MySQL 8.0.39 源码。本篇讲 **MySQL 有哪些索引类型、每种类型的语义与存储结构、以及在 `dict_index_t` 里怎么表达**。
>
> **边界**：B-tree 的结构与操作（搜索/分裂/合并/node pointer）见 [`btr.md`](btr.md)；索引访问时的优化（ICP/MRR/覆盖索引）见 [`access.md`](access.md)；索引承载的约束（唯一性检查、外键）见 [`constraint.md`](constraint.md)；R-tree 的空间语义见 [`../server/datatype/gis.md`](../server/datatype/gis.md)，全文倒排见 [`../feat/fts.md`](../feat/fts.md)。本篇只做**类型谱系与类型间差异**，各类型的完整机制指向专篇。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - 主线与基础构件
    - [类型标志体系：`dict_index_t::type`](#类型标志体系dict_index_ttype)
    - [`n_uniq` 与 `n_fields`：唯一性的粒度](#n_uniq-与-n_fields唯一性的粒度)
  - B-tree 族：主结构与三种基础类型
    - [聚簇索引（主键 / IOT）](#聚簇索引主键--iot)
    - [二级索引](#二级索引)
    - [唯一索引](#唯一索引)
  - B-tree 变体：列级修饰带来的类型
    - [降序索引](#降序索引)
    - [前缀索引](#前缀索引)
    - [功能索引与多值索引](#功能索引与多值索引)
  - 非 B-tree：专用存储结构
    - [全文索引（倒排）](#全文索引倒排)
    - [空间索引（R-tree）](#空间索引r-tree)
  - 元数据型
    - [不可见索引与隐藏索引](#不可见索引与隐藏索引)
- [相关的系统变量/状态变量](#相关的系统变量状态变量)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 的"索引类型"有两层含义，理解这两层的分离是本篇的核心：

- **语义层**（用户可见，DDL 里写的东西）：主键、唯一、二级、降序、前缀、功能、多值、全文、空间、不可见；
- **实现层**（存储结构）：**B-tree**（绝大多数）、**R-tree**（空间）、**倒排**（全文）、Hash（仅内存 AHI 与 NDB 引擎）。

一个语义类型绑定一种存储结构：全文 → 倒排，空间 → R-tree，其余全部 → B-tree。

### 用途

- 加速访问（B-tree 系）；
- 承载约束（PRIMARY/UNIQUE，见 [`constraint.md`](constraint.md)）；
- 支持专门的检索形态（全文的 `MATCH AGAINST`、空间的 `MBRContains` 等）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0 | **降序索引**真正被支持（此前 `DESC` 在索引定义里被解析器忽略） |
| 8.0.13+ | **功能索引**（functional index，表达式索引） |
| 8.0.17 | **多值索引**（`CAST(... AS ... ARRAY)`） |
| 8.0.30 | GIPK（`sql_generate_invisible_primary_key`，需显式开启） |
| 8.0 全程 | 不可见索引（SQL 层元数据 `is_visible`） |

---

## 理论基础

### 设计思想与权衡

**1. 索引组织表（IOT）的取舍。** InnoDB 把聚簇索引的叶子页直接当作"行数据"：整张表就是主键的 B-tree。收益是主键访问零额外跳转；代价是**所有二级索引都要带上主键值**（叶子记录 = 键列 + 主键列），主键越长、越宽，二级索引越膨胀。这解释了"为什么推荐短整型自增主键"。

**2. 语义与实现的分离。** 唯一性**不**通过额外存储结构表达（`dict_index_t` 里没有"唯一索引专用的页格式"），而是靠 `n_uniq` + 插入前的重复检测（见 [`constraint.md`](constraint.md)）。这让"唯一索引"和"普通二级索引"的物理布局完全一致。

**3. 一种新类型 = 一次"降解"。** 功能索引被降解为"隐藏虚拟生成列上的普通二级索引"；多值索引被降解为"隐藏 `Field_typed_array` 虚拟列上的二级索引 + 一行产生多条索引记录"。InnoDB 侧**没有**为它们写专用的 B-tree 代码——这是最小的实现代价。

**4. 专用结构只给无法降解的需求。** 全文（倒排：词 → 文档列表）和空间（R-tree：MBR 层次包络）无法用有序 B-tree 表达，才引入完全不同的存储与插入路径（这二者的 DDL 构建也不走 `Builder` 批量建页）。

### 理论溯源

- Bayer & McCreight，*Organization and Maintenance of Large Ordered Indexes*（1972）：B-tree 起源。
- Guttman，*R-Trees: A Dynamic Index Structure for Spatial Searching*（SIGMOD 1984）：R-tree 与 MBR 层次结构——空间索引的直接来源。
- 倒排索引（信息检索经典结构）：全文索引的来源，见 [`../feat/fts.md`](../feat/fts.md)。

### 算法与数据结构

- 索引记录：B-tree 记录 = 键列 +（二级索引追加主键列）；R-tree 记录 = MBR + 主键列；全文 = 辅助表里的 `(word, first_doc_id, ilist)`。
- 类型判定的统一入口是 `dict_index_t::type` 位域（10 bit）+ 少量布尔/枚举成员。

### 他库对比

| | InnoDB | PostgreSQL | MyISAM |
|---|---|---|---|
| 聚簇 | IOT（主键即数据） | 堆表（索引指向 ctid） | 堆表 |
| 二级索引 | 存主键值 | 存 ctid | 存文件偏移 |
| 空间 | R-tree | GiST/SP-GiST | R-tree |
| 全文 | 倒排（辅助表） | tsvector + GIN | 倒排 |
| 函数索引 | 8.0 支持（虚拟列降解） | 支持（表达式索引） | 不支持 |

---

## 核心实现

### 主线与基础构件

#### 类型标志体系：`dict_index_t::type`

```cpp
// storage/innobase/include/dict0mem.h:91-116
constexpr uint32_t DICT_CLUSTERED   = 1;    // 聚簇索引
constexpr uint32_t DICT_UNIQUE      = 2;    // 唯一索引
constexpr uint32_t DICT_IBUF        = 8;    // change buffer 自身的 B-tree
constexpr uint32_t DICT_CORRUPT     = 16;   // 损坏标记（持久化在 SYS_INDEXES.TYPE）
constexpr uint32_t DICT_FTS         = 32;   // 全文；"can't be combined with the other flags"
constexpr uint32_t DICT_SPATIAL     = 64;   // 空间；同上
constexpr uint32_t DICT_VIRTUAL     = 128;  // 索引含虚拟列（功能索引/多值索引都带）
constexpr uint32_t DICT_SDI         = 256;  // 仅内存结构的 SDI 索引
constexpr uint32_t DICT_MULTI_VALUE = 512;  // 多值索引
constexpr uint32_t DICT_IT_BITS     = 10;   // type 位域宽度
```

要点：

- 可组合位：`DICT_CLUSTERED`/`DICT_UNIQUE`/`DICT_VIRTUAL`/`DICT_MULTI_VALUE`；
- 语义互斥：`DICT_FTS`/`DICT_SPATIAL`（源码注释明确），其中 FTS 用**等值**判定（`index->type == DICT_FTS`），SPATIAL 用**位与**（`dict_index_is_spatial()`）；
- 判定函数全是自由函数，**不是** `dict_index_t` 的成员方法：

| 函数 | 实现 |
|---|---|
| `dict_index_t::is_clustered()` | `type & DICT_CLUSTERED` |
| `dict_index_t::is_multi_value()` | `type & DICT_MULTI_VALUE` |
| `dict_index_is_unique()` | `type & DICT_UNIQUE` |
| `dict_index_is_spatial()` | `type & DICT_SPATIAL` |
| `dict_index_has_virtual()` | `type & DICT_VIRTUAL` |
| `dict_index_has_desc()` | 遍历 `fields[i].is_ascending`，任一为 0 即 true |
| `dict_index_is_auto_gen_clust()` | **`type == DICT_CLUSTERED`（等值！唯一能区分"自动生成"的地方）** |

SQL → InnoDB 的映射在 `ha_innobase::create_index`（`ha_innodb.cc:11959-12126`）：`HA_SPATIAL → DICT_SPATIAL`、`HA_FULLTEXT → DICT_FTS`、`HA_NOSAME → DICT_UNIQUE`、`primary_key → DICT_CLUSTERED`、`HA_REVERSE_SORT → is_ascending=false`、`innobase_is_v_fld → DICT_VIRTUAL`、`innobase_is_multi_value_fld → DICT_MULTI_VALUE`。

#### `n_uniq` 与 `n_fields`：唯一性的粒度

```cpp
// storage/innobase/include/dict0dict.ic:688-716
static inline ulint dict_index_get_n_unique(const dict_index_t *index) {
  return static_cast<uint16_t>(index->n_uniq);
}
static inline ulint dict_index_get_n_unique_in_tree(const dict_index_t *index) {
  if (index->is_clustered()) {
    return (dict_index_get_n_unique(index));
  }
  return (static_cast<uint16_t>(dict_index_get_n_fields(index)));   // 二级索引：全部字段
}
```

- `n_uniq` = **逻辑唯一性**（不含多版本）；
- `n_unique_in_tree` = **B-tree 物理唯一性**：对二级索引返回 `n_fields`——承认"同一逻辑键在 B-tree 里可能有多条物理记录"（多次 UPDATE / delete-mark 后重插都会在二级索引留下记录）。这正是二级索引不存 `DB_TRX_ID`、必须靠页级粗筛 + 回表精判的根因（见 [`../innodb/mvcc.md`](../innodb/mvcc.md)）。

---

#### 索引键的比较与编码层：降序 / collation / 前缀 / NULL

所有索引类型的比较都汇聚到 `cmp_data` / `cmp_whole_field`，这一层决定了"索引里的顺序到底是什么"。

**① 降序索引：比较时取反，不是存储时编码取反**（已确认无任何写入端取反代码）。传递路径：`dict_field_t::is_ascending`（0=DESC）→ `cmp_dtuple_rec_with_match_low` 把它作为 `cmp_data` 的参数 → `cmp_whole_field` 末尾统一取反（`rem0cmp.cc:319-383`）：

```cpp
  if (!is_asc) { cmp = -cmp; }     // ← 降序：比较结果取反
  return (cmp);
```

逐字节比较路径同样在每个分支里 `is_ascending ? 1 : -1`。证据链的另一端：MIN_REC_FLAG 的处理注释明说 "independent of any ASC/DESC flags"——最左节点指针的"最小"语义不受 DESC 影响，这只有在"顺序由比较器决定、物理布局统一"的模型下才成立。**附带限制**：change buffering 只支持列全升序的索引（`rem0cmp.cc:675-676` 注释）。

**② 字符串比较与 collation**：入口 `innobase_mysql_cmp`（`rem0cmp.cc:75-116`）→ 实际比较函数是 `cs->coll->strnncollsp`（**不存在** `charset_compare` 这个符号）：

```cpp
if ((prtype & DATA_MYSQL_TYPE_MASK) == MYSQL_TYPE_STRING &&
    cs->pad_attribute == NO_PAD) {
  /* CHAR 字段取回时要剥尾空格；内部索引比较不走 Field 类，这里补上 */
  a_length = cs->cset->lengthsp(cs, (const char *)a, a_length);
  b_length = cs->cset->lengthsp(cs, (const char *)b, b_length);
}
return (cs->coll->strnncollsp(cs, a, a_length, b, b_length));
```

- **NO PAD（utf8mb4_0900_ai_ci 等）+ CHAR**：先 `lengthsp` 去尾空格再比；**PAD SPACE**：交给 `strnncollsp`（"sp" = space padding）；
- 关键差异：NO PAD 下 **VARCHAR 尾空格显著**（`'a'` ≠ `'a '`），PAD SPACE 下相等；同一索引里 CHAR 与 VARCHAR 的语义组合由此产生；
- 补位字符逻辑在 `cmp_get_pad_char`（`rem0cmp.cc:699-728`）：binary charset 不补（`ULINT_UNDEFINED` = 更长者更大），否则 `pad = 0x20` 短侧补空格逐字节比；
- `cmp_cols_are_equal` 要求两列 `dtype_get_charset_coll` 相同才可比较。

**③ 前缀索引**：`dict_field_t::prefix_len`（字节数；UTF-8 下 MySQL 传 `mbmaxlen × 前缀字符数`）。写入时截断（`row0row.cc:308-314`）：

```cpp
if (ind_field->prefix_len) {
  len = dtype_get_at_most_n_mbchars(     // ★ 保证不切断多字节字符
      col->prtype, col->mbminmaxlen, ind_field->prefix_len, len, ...);
  dfield_set_len(dfield, len);
}
```

比较只比前缀 → 索引顺序只反映前缀；"前缀相等但整列不同"由 **ICP / 回表后的 WHERE 兜底**（源码无针对前缀列的专门回表分支）。server 层 `sql/table.cc:758-761` 注释确认：**前缀索引仍可作覆盖索引**用于只比较/只输出前缀的场景，但输出的是截断值。

**④ NULL 的编码与比较**：可空列在记录头 nulls 位图占 1 位（大小 `UT_BITS_IN_BYTES(index->n_nullable)`，`rem0rec.cc:175-244`），**NULL 字段不占数据区、不写长度**。比较（`cmp_data`，`rem0cmp.cc:397-411`）：

```cpp
if (len1 == UNIV_SQL_NULL || len2 == UNIV_SQL_NULL) {
  if (len1 == len2) return (0);
  /* We define the SQL null to be the smallest possible value of a field. */
  return ((len1 == UNIV_SQL_NULL) == is_asc ? -1 : 1);
}
```

**NULL 最小**，且降序下翻转为最大。注意 `rem0cmp.cc:48-65` 文件头注释写的 "SQL null is bigger" 已过时，与代码矛盾——以代码为准。唯一索引与 NULL：`!nulls_equal` 时 entry 任一 unique 列为 NULL → 直接不冲突（`NULL != NULL`，`row0ins.cc:1905-1911`）。

### B-tree 族：主结构与三种基础类型

#### 聚簇索引（主键 / IOT）

**语法与三种形态：**

```sql
CREATE TABLE t (a INT PRIMARY KEY, b INT);          -- 用户主键
CREATE TABLE t (a INT);                             -- 无主键 → GEN_CLUST_INDEX（6B DB_ROW_ID）
-- sql_generate_invisible_primary_key=ON 时：
CREATE TABLE t (a INT);                             -- 生成 my_row_id（bigint unsigned AUTO_INCREMENT INVISIBLE）
```

**构建要点**（`dict_index_build_internal_clust`，`dict0dict.cc:2960-3158`）：

```cpp
if (dict_index_is_unique(index)) {
  new_index->n_uniq = new_index->n_def;      // 用户主键：n_uniq = 主键列数，不需要 DB_ROW_ID
} else {
  new_index->n_uniq = 1 + new_index->n_def;  // 自动生成簇：+1 = 追加的 DB_ROW_ID
}
...
if (!dict_index_is_unique(index)) {
  dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROW_ID), 0, true);  // 只有自动生成簇才加
}
dict_index_add_col(new_index, table, table->get_sys_col(DATA_TRX_ID), 0, true);    // 总是加
if (!table->is_intrinsic()) {
  dict_index_add_col(new_index, table, table->get_sys_col(DATA_ROLL_PTR), 0, true);
}
// 把表里剩下的所有列追加进来 —— IOT 的实现证据：
for (size_t i = 0; i < table->get_n_user_cols(); i++) {
  if (indexed[col->ind]) continue;
  dict_index_add_col(new_index, table, col, 0, true);
}
```

记录内部布局：`[用户主键列...] [DB_ROW_ID] [DB_TRX_ID] [DB_ROLL_PTR] [其余全部用户列...]`。前缀列不参与 `indexed[]` 标记，因此其完整值仍会被追加——**聚簇索引永远存完整行**。

`trx_id_offset`：若主键列全定长且无前缀，可算出 `DB_TRX_ID` 的固定字节偏移，读可见性时免解析变长列；否则置 0。

**三种主键形态对照：**

| | 用户主键 | GEN_CLUST_INDEX | GIPK |
|---|---|---|---|
| `type` | `DICT_CLUSTERED\|DICT_UNIQUE`=3 | `DICT_CLUSTERED`=1 | `DICT_CLUSTERED\|DICT_UNIQUE`=3 |
| `is_auto_gen_clust()` | false | **true** | false |
| 记录含 `DB_ROW_ID` | 否 | **是** | 否 |
| 二级索引回表用 | 用户 PK | 6B ROW_ID | `my_row_id` |

> **事实纠正**：GIPK 在 8.0.39 默认 **OFF**（`sys_vars.cc:7307`，`DEFAULT(false)`），需 `SET sql_generate_invisible_primary_key=ON`；分区表不支持（`sql_gipk.cc:113-120`）。它是**服务层**生成的普通主键，到 InnoDB 就是普通用户主键。

#### 二级索引

**构建要点**（`dict_index_build_internal_non_clust`，`dict0dict.cc:3163-3255`）——"二级索引叶子存主键值"的实现：

```cpp
new_index = dict_mem_index_create(..., index->n_fields + 1 + clust_index->n_uniq);  // 预留容量
...
for (i = 0; i < clust_index->n_uniq; i++) {        // ★ 追加聚簇索引的 n_uniq 个列
  field = clust_index->get_field(i);
  if (!indexed[field->col->ind]) {
    dict_index_add_col(new_index, table, field->col, field->prefix_len, field->is_ascending);
  } else if (dict_index_is_spatial(index)) {
    dict_index_add_col(new_index, table, field->col, field->prefix_len, field->is_ascending);
  }
}
if (dict_index_is_unique(index)) {
  new_index->n_uniq = index->n_fields;      // 唯一：只算用户键列
} else {
  new_index->n_uniq = new_index->n_def;     // 非唯一：键列 + 追加 PK 列
}
```

注意 3229-3230 行传入的是**聚簇索引字段的 `is_ascending`**：追加的主键列沿用聚簇索引方向（通常 ASC），即使二级索引自身是 DESC。

**为什么不存 `DB_TRX_ID`**：只追加了 `n_uniq` 个主键列，没有 `DATA_TRX_ID`/`DATA_ROLL_PTR`。于是二级索引的 MVCC 只能：① 页级粗筛 `PAGE_MAX_TRX_ID`；② 否则回表用聚簇记录的 `DB_TRX_ID` 精判（详见 [`../innodb/mvcc.md`](../innodb/mvcc.md)）。

#### 唯一索引

结构上与非唯一二级索引**完全相同**，唯一性由 `n_uniq`（不含追加 PK）+ 插入前重复检测表达。

**与 change buffer 的互斥**（`ibuf_should_try`，`ibuf0ibuf.ic:114-130`）：

```cpp
return (innodb_change_buffering != IBUF_USE_NONE && ibuf->max_size != 0 &&
        index->space != dict_sys_t::s_dict_space_id &&
        !index->is_clustered() && !dict_index_is_spatial(index) &&
        !dict_index_has_desc(index) &&
        index->table->quiesce == QUIESCE_NONE &&
        (ignore_sec_unique || !dict_index_is_unique(index)) &&      // ★
        srv_force_recovery < SRV_FORCE_NO_IBUF_MERGE);
```

`ignore_sec_unique` 由 `btr_op != BTR_INSERT_OP` 决定：

| 操作 | `btr_op` | `ignore_sec_unique` | 唯一索引能否缓存 |
|---|---|---|---|
| INSERT（需查重） | `BTR_INSERT_OP` | false | **不能**（必须读页查重） |
| INSERT（忽略唯一检查） | `BTR_INSERT_IGNORE_UNIQUE_OP` | true | 能 |
| DELETE-MARK | `BTR_DELMARK_OP` | true | **能** |
| PURGE DELETE | `BTR_DELETE_OP` | true | **能** |

语义原因：INSERT 必须**先确认不存在相同键**，缓存而不读页就无法查重；DELETE-MARK/PURGE 本来就要定位已存在的记录，缓存只是延迟 I/O，不破坏约束。

---

### B-tree 变体：列级修饰带来的类型

#### 降序索引

标志字段是 `dict_field_t::is_ascending`（**0=DESC，1=ASC**），不是 `is_descending`：

```cpp
// storage/innobase/include/dict0mem.h:913-938
struct dict_field_t {
  dict_col_t *col;
  id_name_t name;
  unsigned prefix_len : 12;
  unsigned fixed_len : 10;
  unsigned is_ascending : 1;     /*!< 0=DESC, 1=ASC */
};
```

**降序的实现在比较器里——把每列比较结果取反**（`rem0cmp.cc:770-861`）：

```cpp
const bool is_ascending = dict_index_is_ibuf(index) || index->fields[cur_field].is_ascending;
...
if (dtuple_f_len == UNIV_SQL_NULL) {
  if (rec_f_len == UNIV_SQL_NULL) goto next_field;
  ret = is_ascending ? -1 : 1;              // ★ NULL 最小：DESC 时反号
  goto order_resolved;
}
...
if (dtuple_byte < rec_byte) {
  ret = is_ascending ? -1 : 1;              // ★ 逐字节比较结果反号
} else if (dtuple_byte > rec_byte) {
  ret = is_ascending ? 1 : -1;
}
```

关键点：

1. **不是"改变扫描方向"**，而是物理顺序本身被翻转——`KEY(a DESC)` 上 forward scan 得到的就是降序序列，无需反向扫描；
2. **SQL NULL 的位置也翻转**：DESC 列里 NULL 变"最大"；
3. **最左 node pointer 的 `REC_INFO_MIN_REC_FLAG` 独立于方向**（`rem0cmp.cc:615-626`）：它恒被定义为小于任何其他 node pointer，否则降序索引的树结构约定会冲突；
4. **不能用 change buffer**：`ibuf_should_try` 里 `!dict_index_has_desc(index)`，因为 ibuf 里存的是升序语义的紧凑元组，没有记录每列方向。

#### 前缀索引

`dict_field_t::prefix_len`（12 bit，**字节数**；UTF-8 下 MySQL 传的是 `mbmaxlen × 前缀字符数`）。判定"是否前缀"靠比较 `key_part->length` 与列 `pack_length()`，而非 `HA_PART_KEY_SEG`（注释说后者"does not seem to be properly set by MySQL"）。数值类型不允许前缀（`ER_WRONG_TYPE_FOR_COLUMN_PREFIX_IDX_FLD`），多值列不支持前缀。

三个连带影响：

1. **聚簇索引仍存完整列**（`dict0dict.cc:3107-3112`：`prefix_len != 0` 的列不标记 `indexed`，于是完整值在 3124 行被再次追加）；
2. **`trx_id_offset` 失效**（`dict0dict.cc:3067-3072`：主键含前缀列时强制置 0）；
3. **不能做覆盖索引、不能消除 ORDER BY**——索引里只有前 N 字节，超出部分既取不到也不保证有序。

#### 功能索引与多值索引

**功能索引**在服务层被降解为"隐藏虚拟生成列上的二级索引"（`sql/sql_table.cc:7742-7891`）：

```cpp
cr->hidden = dd::Column::enum_hidden_type::HT_HIDDEN_SQL;
cr->stored_in_db = false;
gcol_info->set_field_stored(false);       // VIRTUAL，不落盘
cr->gcol_info = gcol_info;                // 表达式 = KEY((a+b)) 里的 a+b
```

列名是"key 名 + key part 编号的哈希"。限制：**不能作为 PRIMARY KEY**（`ER_FUNCTIONAL_INDEX_PRIMARY_KEY`）、**不能是 FULLTEXT/SPATIAL**、不能是 BLOB/TEXT/GEOMETRY/JSON。InnoDB 侧只看到 `DICT_VIRTUAL`，**没有专门的"功能索引"代码路径**。

**多值索引**在功能索引基础上再进一步：隐藏列是 `Field_typed_array`（`MYSQL_TYPE_TYPED_ARRAY`，`sql/field.h:4134-4162`），索引额外带 `DICT_MULTI_VALUE`。**一行产生 N 条索引记录**（N = 数组元素数）——这是它与普通二级索引最本质的区别：

```cpp
// storage/innobase/row/row0ins.cc:3260-3287
static dberr_t row_ins_sec_index_multi_value_entry(dict_index_t *index, dtuple_t *entry,
                                                   uint32_t &multi_val_pos, que_thr_t *thr) {
  Multi_value_entry_builder_insert mv_entry_builder(index, entry);
  for (dtuple_t *mv_entry = mv_entry_builder.begin(multi_val_pos);
       mv_entry != nullptr; mv_entry = mv_entry_builder.next()) {
    err = row_ins_sec_index_entry(index, mv_entry, thr, false);   // ★ 逐元素复用普通插入
    if (err != DB_SUCCESS) {
      multi_val_pos = mv_entry_builder.last_multi_value_position();  // 失败时记断点，下次续传
      return (err);
    }
  }
  multi_val_pos = 0;
  return (err);
}
```

builder 的注释点明实现要点："It simply **replace the pointers** to the multi-value field data for each different value"——不重建 tuple，只换指针，零拷贝。UPDATE 场景用 `Multi_value_entry_builder_normal`，靠 `bitset` 只处理变化的元素（新旧数组里相同的元素不必删了再插）。一个多值索引**只能有一个多值列**（`dict0mem.h:1643` 注释 "Only one multi-value field"）。

**数据载体与比较**：

```cpp
// storage/innobase/include/data0data.h:384-408
struct multi_value_data {
  const void **datap;      // 指向各元素的指针数组
  uint32_t *data_len;      // 每个元素的长度
  uint64_t *conv_buf;      // 整型元素的转换缓冲
  uint32_t num_v;          // 元素个数
  uint32_t num_alc;        // 已分配容量
  Bitset *bitset;          // UPDATE 时标记"哪些元素需要处理"
};
```

比较不走普通路径——`cmp_dtuple_rec_with_match_low`（`rem0cmp.cc:661-673`）对多值字段用 `mv_data->has(type, rec_b_ptr, rec_f_len)`（判断该记录的值是否在数组内），而非逐字节 `cmp_data`。另有两个特殊长度标记（`data0data.h:581-595`）：

- `s_multi_value_virtual_col_length_marker = 0xFF`：多值虚拟列的长度标记，用于识别 undo 里的多值列；
- `s_multi_value_no_index_value = 0x0`（无值，映射 `UNIV_NO_INDEX_VALUE`）、`s_multi_value_null = 0x1`（NULL，映射 `UNIV_SQL_NULL`）。

**与优化器的交互**（识别点全在 range optimizer 的 `get_func_mm_tree`，`range_analysis.cc:553`）：

1. **哪些谓词能用**：`Item_func_member_of`（注意**没有** `Item_func_json_member_of` 这个类名）、`Item_func_json_contains`、`Item_func_json_overlaps`。唯一判定是 predicand 为 `Field_typed_array`（`predicand->returns_array()`）。
2. **`MEMBER OF` 被降级成等值**（`range_analysis.cc:621-651`）：把 predicand 求值为 JSON、`field->coerce_json_value()` 类型强转后，**临时置 `table->const_table = true`**，伪装成"field op 常量"的 `EQ_FUNC` 谓词交给通用路径 `get_mm_parts` 生成 SEL_TREE。
3. **`JSON_OVERLAPS`/`JSON_CONTAINS` 生成 N 个等值 range 的 OR**（`get_func_mm_tree_from_json_overlaps_contains`，463-535 行）：每个非 null 元素一次 `coerce_json_value` + `get_mm_parts(EQ_FUNC)`，逐个 `tree_or()` 并入；去重就在这里做（`wr.remove_duplicates`，字符串数组按列 charset）；前导 `J_NULL` 跳过（null 元素不进索引）。**即"每个数组元素一个 range"**。
4. **ref vs range**：等值 `5 MEMBER OF (...)` 最终走 **ref**（`Index lookup`，测试基线 `explain_tree.result` 证实）——server 传来的是**单值 key**，`ha_innobase::index_read` 对多值索引**没有任何特殊分支**，照常 B-tree 定位（每个元素本来就是一条独立物理记录）；`JSON_OVERLAPS/CONTAINS`（N 个 range 的 OR）走 range。
5. **AND 合并的语义坑**（`range_optimizer/tree.cc:960-965`）：两个不相交 range 对多值索引**不代表恒假**——`1 member of(f) AND 2 member of(f)` 对 `f=[1,2]` 为真，必须走 `and_all_keys` 而非普通的矛盾消解。
6. **同一行命中两次的问题与 unique filter**：一行 `[1,2]` 在 range 1 和 range 2 各命中一次。`RefIterator::Init()` / `IndexRangeScanIterator::Init()` / DS-MRR 的 `dsmrr_init` 都会在 `HA_MULTI_VALUED_KEY` 时发 `HA_EXTRA_ENABLE_UNIQUE_RECORD_FILTER`——创建 `Unique_on_insert(ref_length)`，**按 rowid 去重**（"same row returned twice" 消除）；Index Merge 场景则禁用 filter（由 merge 自身去重）。
7. **HA_MULTI_VALUED_KEY 的连带限制**（`ha_innobase::index_flags`，`ha_innodb.cc:6391`）：关闭 `HA_READ_ORDER`（不能按索引序扫描）与 `HA_KEYREAD_ONLY`（不能只读索引列——虚拟列值不在二级索引里）→ **不能覆盖索引、不能 ORDER BY 走索引**；`handler::compare_key` 对 MV 索引跳过比较（MRR 期间虚拟列未更新，TODO 注明"Disable MRR on MV index"）。
8. **EXPLAIN 的打印**：range 输出 `<值> MEMBER OF (<原始 JSON 表达式>)`（`append_range`，`range_optimizer.cc:1536`——从 `gcol_info->expr_item` 剥掉 `CAST(... AS ... ARRAY)` 外壳）；代价按"N 个普通 range 叠加"走通用模型，无多值专用公式。

---

### 非 B-tree：专用存储结构

#### 全文索引（倒排）

`dict_index_t` 里只有元描述（`n_uniq = 0`，不参与 B-tree 排序/唯一语义），真正的倒排数据在 **11 张辅助表**（每张是独立的 InnoDB 表）：

```cpp
// storage/innobase/include/fts0fts.h:105-109
constexpr size_t FTS_NUM_AUX_INDEX  = 6;    // 每个 FTS 索引一套 index_1..index_6
constexpr size_t FTS_NUM_AUX_COMMON = 5;    // 整表一套：being_deleted(/_cache)、config、deleted(/_cache)
```

- **6 张 INDEX 表**：按首字符值分档（`fts_index_selector[] = {9,"index_1"}, {65,"index_2"}, ...`），主键 `(word, first_doc_id)`，`ilist` 是 BLOB，存 `[doc_id, position...]` 的变长编码——**这就是倒排列表**；另有 `last_doc_id`/`doc_count` 用于跳过与统计。
- **5 张 COMMON 表**：`deleted`/`deleted_cache`（已删但倒排项未清理的 doc_id）、`being_deleted`/`being_deleted_cache`、`config`（含 `FTS_SYNCED_DOC_ID` 等）。
- **`FTS_DOC_ID`**：未显式定义时由 DD 层自动加隐藏列 + **基表上真正的 B-tree 唯一索引** `FTS_DOC_ID_INDEX`（`dict_index_t::hidden = true`），它不占 11 张辅助表的名额。
- **插入路径完全绕过 B-tree**：`row0ins.cc:3568` 的 `if (node->index->type != DICT_FTS)` 把 FTS 索引跳过，改走 `fts_trx_add_op(trx, table, doc_id, FTS_INSERT/FTS_DELETE, ...)` → 先写 FTS cache，由 sync/optimize 异步刷到辅助表。

#### 空间索引（R-tree）

记录第 0 字段是 **MBR**（不是原始几何数据）：

```cpp
// storage/innobase/row/row0ins.cc:3323-3346
double mbr[SPDIMS * 2];                    // 2 维 × (min,max) = 4 个 double
get_mbr_from_store(srs, dptr, dlen, SPDIMS, mbr, srid);
dfield_write_mbr(field, mbr);
```

与 B-tree 的四处本质差异：

1. **非叶子节点只有 2 个字段**（MBR + page no），常量 `DICT_INDEX_SPATIAL_NODEPTR_SIZE = 1`；
2. **搜索模式是 `PAGE_CUR_RTREE_INSERT`**（插入位置由"MBR 最小扩大代价"决定，不是有序位置）；
3. **插入后必须上溯扩大祖先 MBR**（`rtr_ins_enlarge_mbr`）——B-tree 插入不改变祖先键值，R-tree 改变；
4. **DDL 构建不能自底向上排序建页**，只能用 `RTree_inserter` 批量 tuple 逐条插入。

**插入路径与 B-tree 的完整差异**（`row0ins.cc:2872-2897`）——这段是 R-tree 插入的骨架：

```cpp
if (dict_index_is_spatial(index)) {
  rtr_init_rtr_info(&rtr_info, false, &cursor, index, false);   // ① R-tree 搜索上下文（B-tree 没有）
  rtr_info_update_btr(&cursor, &rtr_info);
  btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_RTREE_INSERT, ...);   // ② 不是 PAGE_CUR_LE

  if (mode == BTR_MODIFY_LEAF && rtr_info.mbr_adj) {
    mtr_commit(&mtr);
    rtr_clean_rtr_info(&rtr_info, true);
    rtr_init_rtr_info(&rtr_info, false, &cursor, index, false);
    rtr_info_update_btr(&cursor, &rtr_info);
    mtr_start(&mtr);
    search_mode &= ~BTR_MODIFY_LEAF;
    search_mode |= BTR_MODIFY_TREE;                              // ③ 上溯调整 → 必须升级为悲观模式
    btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_RTREE_INSERT, ...);
    mode = BTR_MODIFY_TREE;
  }
} else {
  btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_LE, search_mode, ...);   // B-tree 常规
}
...
if (cursor.flag == BTR_CUR_INSERT_TO_IBUF) {
  ut_ad(!dict_index_is_spatial(index));      // ④ 空间索引绝不可能进 ibuf
  goto func_exit;
}
```

插入完成后再上溯扩大祖先 MBR（`row0ins.cc:3019-3021`）：

```cpp
if (err == DB_SUCCESS && dict_index_is_spatial(index) && rtr_info.mbr_adj) {
  err = rtr_ins_enlarge_mbr(&cursor, &mtr);      // 三种插入路径后都要调
}
```

R-tree 核心函数（`gis0rtree.cc` / `gis0rtree.ic`）：`rtr_page_cal_mbr`（算页 MBR）、`rtr_split_page_move_rec_list`（**按 MBR 分组分裂**，非中点）、`rtr_ins_enlarge_mbr`、`rtr_adjust_upper_level`、`rtr_merge_and_update_mbr`。

限制：列必须 `NOT NULL`（源码侧 `row0ins.cc:3405-3407`：几何数据长度不足返回 `DB_CANT_CREATE_GEOMETRY_OBJECT`；SQL 侧 `ER_SPATIAL_CANT_HAVE_NULL`）；不支持虚拟列/功能索引；不能进 change buffer（`ibuf_should_try` + `row0ins.cc:2831` 不设 `BTR_INSERT` + `row0ins.cc:2914` 断言 + `btr0cur.cc:826` 断言）。

---

### 元数据型

#### 不可见索引与隐藏索引

**两个层次，容易混淆：**

| | DD `is_hidden()`（SE/DD 隐藏） | DD `is_visible()=false`（SQL 不可见） |
|---|---|---|
| 例子 | `FTS_DOC_ID_INDEX`、`GEN_CLUST_INDEX` | `ALTER INDEX k INVISIBLE` |
| 进 `keys_in_use` | **否**（`dd_table_share.cc:1552` 直接 continue） | **是** |
| 进 `visible_indexes` | 否 | **否** |
| 优化器可见 | 否 | 否（除非 `use_invisible_indexes=on`） |
| DML 是否维护 | 是 | **是** |

```cpp
// sql/table.cc:497-502
Key_map TABLE_SHARE::usable_indexes(const THD *thd) const {
  Key_map usable_indexes(keys_in_use);
  if (!thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES))
    usable_indexes.intersect(visible_indexes);
  return usable_indexes;
}
```

**关键事实**：`dict_index_t` **没有** `is_visible` 成员——InnoDB 完全不知道 SQL 层的 INVISIBLE（全文检索 `storage/innobase` 下 `invisible` 无相关命中）。不可见索引在 InnoDB 的索引链表里与可见索引无区别，**DML 照常维护它**。这正是它作为"软删除灰度"手段的意义：先设 INVISIBLE 观察，确认无影响再真 DROP，有问题立刻改回 VISIBLE（无需重建数据）。

注意区分：`dict_index_t::hidden`（`dict0mem.h:1199`）是 **SE-hidden** 语义，只对 `GEN_CLUST_INDEX` 和 `FTS_DOC_ID_INDEX` 为 true——**与 SQL 的 INVISIBLE 无关**。

---

### 类型总表

| 类型 | 存储结构 | B-tree? | `type` 位 | `n_uniq` | change buffer | 备注 |
|---|---|---|---|---|---|---|
| 主键（用户定义） | B-tree（IOT） | ✅ | `DICT_CLUSTERED\|DICT_UNIQUE` | 主键列数 | ❌ | 记录含 TRX_ID/ROLL_PTR |
| GEN_CLUST_INDEX | B-tree（IOT） | ✅ | `DICT_CLUSTERED`（等值判定） | 1 | ❌ | 含 `DB_ROW_ID` |
| GIPK 主键 | B-tree（IOT） | ✅ | `DICT_CLUSTERED\|DICT_UNIQUE` | 1 | ❌ | 服务层生成 `my_row_id` |
| 二级索引（非唯一） | B-tree（键+PK） | ✅ | 无 Cluster/Unique | `n_def`（含 PK） | ✅ | 无 TRX_ID |
| 二级索引（唯一） | B-tree（键+PK） | ✅ | `DICT_UNIQUE` | 用户键列数 | ⚠️ INSERT 不可，DELMARK/DELETE 可 | 布局同上 |
| 降序索引 | B-tree（物理序翻转） | ✅ | 同二级 + `is_ascending=0` | 同二级 | ❌ | NULL 变最大 |
| 前缀索引 | B-tree（存前 N 字节） | ✅ | 同二级 + `prefix_len` | 同二级 | ✅（若不唯一/不降序） | 不能覆盖/消排序 |
| 功能索引 | B-tree（隐藏虚拟列） | ✅ | `\|DICT_VIRTUAL` | 同二级 | ✅ | 不能 PK/FULLTEXT/SPATIAL |
| 多值索引 | B-tree（每行 N 条） | ✅ | `\|DICT_VIRTUAL\|DICT_MULTI_VALUE` | 同二级 | ✅ | 1 个多值列；无前缀 |
| 全文索引 | **倒排**（11 辅助表） | ❌（辅助表内是 B-tree） | `DICT_FTS`（等值） | **0** | ❌ | 异步 cache |
| 空间索引 | **R-tree** | ❌ | `DICT_SPATIAL` | 同二级 | ❌ | NOT NULL；非叶 2 字段 |
| 不可见索引 | 同基础类型 | 同 | 同基础类型 | 同 | 同 | 仅 DD `is_visible=false` |

---

## 相关的系统变量/状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `sql_generate_invisible_primary_key` | **OFF** | Global/Session | GIPK：无主键表自动加 `my_row_id`（分区表不支持） |
| `use_invisible_indexes` | OFF | optimizer_switch | 调试用：让优化器临时"看见"不可见索引（常量 `OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES = 1ULL<<19`） |
| `innodb_change_buffering` | all | Global | 影响非唯一二级索引的 INSERT 缓存（唯一索引的 INSERT 恒不缓存） |

---

## Misc

### 易混淆点

- **"类型"是语义，"存储结构"是实现**：`UNIQUE` 不是一种存储结构，唯一索引与非唯一二级索引的物理布局完全相同，区别只在 `n_uniq` 和插入前的检查。
- **`dict_index_t` 没有 `has_virtual`/`is_visible` 成员**：是否含虚拟列是函数 `dict_index_has_virtual()`；索引级 INVISIBLE 是 SQL/DD 概念，InnoDB 无感知。
- **降序字段名是 `is_ascending`（0=DESC）**，不是 `is_descending`。
- **`prefix_len` 是字节数**，UTF-8 下等于 `mbmaxlen × 前缀字符数`。
- **隐藏 ≠ 不可见**：`dict_index_t::hidden`（SE 隐藏，如 GEN_CLUST_INDEX）与 DD `is_visible`（SQL 不可见）是两个无关机制。
- **GIPK 默认 OFF**（8.0.30 引入但需显式开启），不要当成默认行为。

### 一句话总结

十种索引类型里，**八种都是 B-tree**（区别只在 `dict_index_t` 的标志位与列级修饰：唯一/降序/前缀/虚拟/多值），只有全文（倒排）和空间（R-tree）换了存储结构；新类型大多通过"降解为已有类型"（功能索引 → 虚拟列索引、多值索引 → 虚拟列索引 + 多条记录）实现，因此 InnoDB 的 B-tree 代码只有一套。

---

## 参考

**论文**
- Bayer, R. & McCreight, E. *Organization and Maintenance of Large Ordered Indexes*. Acta Informatica, 1972.
- Guttman, A. *R-Trees: A Dynamic Index Structure for Spatial Searching*. SIGMOD, 1984.（空间索引）
- Comer, D. *The Ubiquitous B-Tree*. ACM Computing Surveys, 1979.

**官方文档**
- *MySQL 8.0 Reference Manual → 15.6.2 Indexes*（各索引类型的官方语义）
- *MySQL 8.0 Reference Manual → 13.1.15 CREATE INDEX Statement*（降序、前缀、功能、多值、不可见语法）
- *MySQL 8.0 Reference Manual → 15.6.2.4 InnoDB Full-Text Indexes*、*15.6.2.5 Spatial Indexes*

**相关文档**
- B-tree 结构与操作全套：[`btr.md`](btr.md)
- 索引访问优化（回表/覆盖索引/ICP/MRR）：[`access.md`](access.md)
- 索引承载的约束（唯一性检查/外键）：[`constraint.md`](constraint.md)
- 全文检索完整机制：[`../feat/fts.md`](../feat/fts.md)
- 空间类型与 R-tree：[`../server/datatype/gis.md`](../server/datatype/gis.md)
- 功能索引/多值索引所在的生成列家族：[`../feat/generated_columns.md`](../feat/generated_columns.md)
- 索引记录格式与 `n_uniq` 的可见性含义：[`../innodb/physical/record.md`](../innodb/physical/record.md)、[`../innodb/mvcc.md`](../innodb/mvcc.md)
