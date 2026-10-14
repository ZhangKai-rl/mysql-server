# 索引的构造：KEY 与从 DDL 到 `dict_index_t` 的三条链路

> **一句话**：`KEY` 是"索引在 server 层的 SQL 表示"，也是整个索引构造链的**投影层**——SQL 语句、DD 持久化、引擎内存对象三方的语义互不对等，全靠它做转换。理解它的钥匙是两件事：**构造期与开表期的字段填充不对称**（同一结构两个来源），**`flags` 与 `actual_flags` 的双层属性**（持久化语义 vs 运行期真实形态）。

> **边界**：本篇讲 `KEY` 类本身与**三条构造链路**。
> - `dd::Index` ↔ `dict_index_t` 的元数据映射细节见 [`metadata.md`](metadata.md)；
> - 索引 DDL 的**执行**（COPY/INPLACE/INSTANT 三算法、row log、online）见 [`operations.md`](operations.md)；
> - `dict_index_t` 的构建（`dict_index_build_internal_*`）见 [`metadata.md`](metadata.md)；
> - 索引记录的**字节形态**（KEY 描述的下游产物）见 [`record_format.md`](record_format.md)；
> - 索引类型语义见 [`types.md`](types.md)。

## 目录

- [概述](#概述)
  - 是什么
  - 用途
  - 版本演进
- [理论基础](#理论基础)
  - 为什么需要两套索引描述：四处不对等
  - 为什么构造期与开表期字段不对称
  - 为什么要有 flags 与 actual_flags 两份
  - 为什么主键提升不重命名 KEY
  - 投影关系全景
- [核心实现](#核心实现)
  - 基础构件：KEY / KEY_PART_INFO / Key_spec
  - 构造路径一：CREATE TABLE
  - 构造路径二：ALTER ADD INDEX
  - 构造路径三：开表回读
  - ★ 功能索引：投影设计的最佳例证
  - 运行期：TABLE 层拷贝与 rec_per_key
- [可观测性](#可观测性)
- [Misc](#misc)
  - ★ 本机制里的工程实现技法
  - 扩展点：加一种索引属性要动哪些地方
  - 失效与退化场景
  - 构造期 vs 开表期不对称表
  - 易混淆点（源码核实清单）
- [参考](#参考)

---

## 概述

### 是什么

`KEY` + `KEY_PART_INFO`（`sql/key.h`）是索引在 **server 层的中间表示**。它不是事实来源，而是**投影**：

- 建表时由解析器产物 `Key_spec` 构造；

> 📌 **关于 `sql/key_spec.cc`**（避免误判为缺口）：这个文件名在本知识库其他篇里从未出现，容易让人以为"KEY 规格翻译"没人讲。实际上 **`key_spec.cc` 只有 175 行，它并不做翻译**——只实现了 `Key_part_spec::operator==`、`foreign_key_prefix()`、`Key_part_spec::resolve_expression()`（函数索引的表达式 `fix_fields` 入口）、`set_name_and_prefix_length()` 和 `Foreign_key_spec::validate()`（外键校验）。
>
> **真正把 `Key_spec` 翻译成 `KEY` + `KEY_PART_INFO` 的是 `sql/sql_table.cc` 里的 `static bool prepare_key()`**（逐列调 `prepare_key_column()`）。该翻译过程正是本篇核心实现的内容。
>
> 另注：8.0.13 之后**不存在** `Key_function_spec` / `Key_json_member_spec` 这两个类——函数索引与多值索引统一用 `Key_part_spec` 持有 expression 表示（`has_expression()`），多值索引用 `get_expression()->returns_array()` 判定。
- 开表时由 DD（`dd::Index`）回读构造；
- 引擎侧（`handler` 接口、`ddl::Index_defn`）与优化器（`records_per_key`、`usable_key_parts`）都直接消费它。

### 用途

| 场景 | 链路 |
|---|---|
| 建表 / 加索引 | DDL → `Key_spec` → KEY → `dd::Index` → 引擎 |
| 开表 | `dd::Index` → KEY → `TABLE_SHARE`/`TABLE` |
| 优化器 | ref 代价估算、`part_of_sortkey` 判定、ICP/MRR 能力判定 |

### 版本演进

| 版本 | 变化 | 为什么 |
|------|------|--------|
| 5.x | 存在独立的存储层 `struct KEY_PART`（server `KEY_PART_INFO` 与它分离） | 老的双层结构 |
| **8.0** | **5.x 的 `struct KEY_PART` 已消失**——`KEY_PART_INFO` 本身就是传给引擎的接口；引擎侧再转成 `ddl::Index_field` | 去掉一层无谓的转换 |
| 8.0.13 | 功能索引（functional index）：表达式 → 隐藏生成列 → 普通索引 | 让"表达式索引"复用已有索引机制，不新增索引类型 |
| 8.0.17 | 多值索引（`HA_MULTI_VALUED_KEY`，JSON 数组） | 配合 JSON 部分更新 |
| 8.0 全程 | 不可见索引（`is_visible`）、降序索引（`HA_REVERSE_SORT`）纳入 KEY 描述 | SQL 语义持续丰富，KEY 作为投影层跟着扩 |
| 8.4 / 9.x | **本库未跟踪到 `KEY` 类与三条构造链路的结构性变化**（8.0 起稳定） | —— 若发现变化请补入本节 |

---

## 理论基础

### 为什么需要两套索引描述：四处不对等

索引在 MySQL 里有两套描述：server 层的 `KEY` 与引擎层的 `dict_index_t`。这不是冗余，而是因为**两者的语义天生对不齐**——四处不对等决定了必须有一个投影层：

| # | 不对等 | 具体表现 | 为什么不能合并 |
|---|---|---|---|
| 1 | **抽象层级** | server 层不能知道 InnoDB 内部结构；`handler` 接口是硬边界 | 合并就绑定了单一引擎，违反可插拔存储引擎的设计前提 |
| 2 | **语义丰富度** | SQL 层的索引语义比引擎丰富：功能索引是**表达式**、前缀索引只索引前 N 字节、降序、不可见、多值 | 引擎里这些要么被降解（功能索引 → 隐藏生成列上的普通索引），要么由 server 层单独记账（`is_visible`） |
| 3 | **使用方位置** | 优化器在 **server 层**，每候选访问路径都要问"这个前缀有多少不同值"，不能每次下沉到引擎 | 需要一份 server 层常驻的统计与能力描述（`rec_per_key`、`usable_key_parts`） |
| 4 | **生命周期** | `dd::Index` 是**持久化**的（DD 表），`dict_index_t` 是**内存**对象（开表构建、关表释放），`KEY` 介于两者之间 | 三者的生命周期不同，不能共用一个对象 |

★ **结论**：`KEY` 存在的理由是"**SQL 语义**与**引擎存储**之间的语义落差"。落差不是设计失误，而是分层架构的必然产物——所以 `KEY` 上到处是"两边各让一步"的痕迹（隐藏生成列、隐藏 PK part、双份 flags）。

### 为什么构造期与开表期字段不对称

这是本篇最反直觉的一点：同一个 `KEY_PART_INFO`，建表时与开表时**填的字段不一样**（`offset` 的含义甚至都不同）。原因有两层：

**约束一：记录布局在建表期还没最终确定。** 行的 null bitmap 头与各列的记录内偏移，要等所有列都确定、跑完 `calculate_field_offsets` 之后才算得出来。所以：

- CREATE 阶段：`prepare_key_column` 只填"语句里能直接读到的"（`fieldnr`、列内 `offset`、`length`）；
- `null_bit` / `null_offset` / `store_length` / `type` / `bin_cmp`：**算不出来，索性不填**，留给开表后处理。

**约束二：两条路径的目的不同，需要的精度不同。**

| 路径 | 目的 | 需要什么 |
|---|---|---|
| 建表 | 生成 `dd::Index` 持久化 + 让引擎建索引 | 只需要"哪些列、什么顺序、多长、升降序" |
| 开表 | 真正读写行、做 ref 比较、算 ICP | 需要精确的字节布局（null 位、store_length、能否二进制比较） |

★ 这是"**同一结构服务两个目的**"的典型取舍：如果建表期也全量计算，要多跑一遍与开表同样的布局计算（而那份结果马上就被丢弃——建表路径不读写行）。代价是**同一个字段在两个阶段语义不同**（`offset` 一个是列内偏移、一个是含 null 头的绝对偏移），这正是最容易踩的坑。

### 为什么要有 flags 与 actual_flags 两份

`KEY::flags` 描述"**用户声明的键属性**"，`actual_flags` 描述"**运行期的真实形态**"。两者会分叉，因为：

**引擎会在开表时扩展二级索引**——`add_pk_parts_to_sk()` 把不在二级键里的主键列追加为**隐藏 key part**，追加完成后该二级索引"实际唯一"（`actual_flags |= HA_NOSAME`），但 `flags` 不变（DD 里它仍是 `IT_MULTIPLE`）。

为什么要分两份而不是合并成一份：

- 若只留 `flags`（DD 语义）：运行期"实际唯一"这个事实丢失 → 优化器无法利用"扩展后唯一"做优化；
- 若只留 `actual_flags`（运行期形态）：`SHOW CREATE TABLE` / DD 里用户的原始声明被改写 → **用户 DDL 语义不可逆地丢失**（`UNIQUE` 和 `MULTIPLE` 在 DDL 里是两回事）。

★ 所以分两份是**"持久化语义"与"运行期形态"必须分离**的必然结果。同理，`user_defined_key_parts` / `actual_key_parts` / `unused_key_parts` 三个计数也是为同一目的存在（分别记"用户声明的段数""含隐藏扩展的总段数""为扩展预留但未用的槽"）。

### 为什么主键提升不重命名 KEY

没有显式 PRIMARY KEY 但有 NOT NULL UNIQUE 键时，MySQL 会把该唯一键**提升**为主键（`is_candidate_primary_key`）。但 DD 里 `dd::Index` 的类型**仍是 `IT_UNIQUE`**，只是列标成 `CK_PRIMARY`；开表侧把 `share->primary_key` 指向它。

★ 约束：`SHOW CREATE TABLE` 是**从 DD 重建**的。若提升时把类型改成 `IT_PRIMARY`，用户看到的 DDL 就与当初写的不一致（`UNIQUE KEY` 变成 `PRIMARY KEY`）——**用户的 DDL 语义被系统改写了**。保持 DD 原样 + 运行期用指针表达提升，是最小侵入。副作用：`SHOW CREATE` 仍显示 `UNIQUE KEY`（只有名字叫 `"PRIMARY"` 才会打印 `PRIMARY KEY`）。

### 投影关系全景

```
CREATE TABLE:   Key_spec ──prepare_key──► KEY ──fill_dd_indexes_from_keyinfo──► dd::Index ──┐
                                           │                                                 │
ALTER:          旧 KEY ──反解──► Key_spec ─┘（合并新定义后复用 CREATE 路径）                    │
                                           │                                                 │
                                           └──► handler::create / ddl::Index_defn ──► dict_index_t
                                                                                              │
开表:           dd::Index ──fill_index_from_dd──► KEY（+ 后处理补全）──────────────────────────┘
```

三条入、两处出（`dd::Index` 持久化、`dict_index_t` 引擎对象），`KEY` 是唯一的交汇点。

---

## 核心实现

### 基础构件：KEY / KEY_PART_INFO / Key_spec

#### KEY 的全部成员

⚠️ **勘误**：`KEY` **不在 `sql/sql_class.h`**（那个文件 grep `key_info` / `KEY *` 均 0 命中），真身在 **`sql/key.h:114`**。

```cpp
class KEY {
 public:
  /** Tot length of key */
  uint key_length{0};
  /** dupp key and pack flags */
  ulong flags{0};                    // ★ 第一级 flags：用户可见属性
  /** dupp key and pack flags for actual key parts */
  ulong actual_flags{0};             // ★ 第二级 flags：叠加"PK 扩展后变 unique"
  /** How many key_parts */
  uint user_defined_key_parts{0};    // DDL 里写的列数
  /** How many key_parts including hidden parts */
  uint actual_key_parts{0};          // = user_defined + PK 扩展附加数
  /**
     Key parts allocated for primary key parts extension but
     not used due to some reasons(no primary key, duplicated key parts)
  */
  uint unused_key_parts{0};          // 为 PK 扩展预留但没用上的槽位
  /** Should normally be = actual_key_parts */
  uint usable_key_parts{0};          // ★ 见下：建表期是"原始序号"，开表期才是 filesort 可用前缀
  uint block_size{0};
  enum ha_key_alg algorithm { HA_KEY_ALG_SE_SPECIFIC };
  /**
    A flag which indicates that index algorithm for this key was explicitly
    specified by user. So, for example, it should be mentioned in SHOW CREATE
    TABLE output.
  */
  bool is_algorithm_explicit{false};
  /** Fulltext [pre]parser */
  plugin_ref parser{nullptr};
  /** Fulltext [pre]parser name */
  LEX_CSTRING parser_name{nullptr, 0};

  KEY_PART_INFO *key_part{nullptr};  // 指向全表共享的 KEY_PART_INFO 大数组中的一段
  /** Name of key */
  const char *name{nullptr};         // PRIMARY 恒为 "PRIMARY"

  /**
    Array of AVG(number of records with the same field value) for 1st ... Nth
    key part. 0 means 'not known'.
  */
  ulong *rec_per_key{nullptr};       // ★ 整数版统计，0 = 未知

  bool is_functional_index() const;

  LEX_CSTRING engine_attribute{nullptr, 0};
  LEX_CSTRING secondary_engine_attribute{nullptr, 0};

 private:
  double m_in_memory_estimate{0.0};
  /**
    ... This is the same information as stored in the above rec_per_key array
    but using float values instead of integer values. If the storage engine
    has supplied values in this array, these will be used. ...
  */
  rec_per_key_t *rec_per_key_float{nullptr};   // ★ float 版，优先于整数版
 public:
  /**
    True if this index is visible to the query optimizer. The optimizer may
    only use visible indexes.
  */
  bool is_visible{false};            // ★ 类内默认 false，但 KEY_CREATE_INFO 默认 true
  TABLE *table{nullptr};
  LEX_CSTRING comment{nullptr, 0};

  bool has_records_per_key(uint key_part_no) const {
    return ((rec_per_key_float && rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN) ||
            (rec_per_key && rec_per_key[key_part_no] != 0));
  }
  rec_per_key_t records_per_key(uint key_part_no) const {
    if (rec_per_key_float[key_part_no] != REC_PER_KEY_UNKNOWN)
      return rec_per_key_float[key_part_no];
    return (rec_per_key[key_part_no] != 0)
               ? static_cast<rec_per_key_t>(rec_per_key[key_part_no])
               : REC_PER_KEY_UNKNOWN;
  }
  void set_records_per_key(uint key_part_no, rec_per_key_t rec_per_key_est) {
    assert(rec_per_key_est == REC_PER_KEY_UNKNOWN || rec_per_key_est >= 1.0);
    rec_per_key_float[key_part_no] = rec_per_key_est;
  }
  void set_rec_per_key_array(ulong *rec_per_key_arg,
                             rec_per_key_t *rec_per_key_float_arg) {
    rec_per_key = rec_per_key_arg;
    rec_per_key_float = rec_per_key_float_arg;
  }
};
```

```cpp
typedef float rec_per_key_t;
/**
  If an entry for a key part in KEY::rec_per_key_float[] has this value,
  then the storage engine has not provided a value for it and the rec_per_key
  value for this key part is unknown.
*/
#define REC_PER_KEY_UNKNOWN -1.0f
#define IN_MEMORY_ESTIMATE_UNKNOWN -1.0
```

> ⚠️ 勘误：**不存在 `KEY::ext_key_parts`**（全仓 0 命中）。三件套是真 `user_defined_key_parts` / `actual_key_parts` / `unused_key_parts`。
> ⚠️ **不存在 `KEY::set_records`**——真名 `set_records_per_key()` / `records_per_key()` / `has_records_per_key()` / `set_rec_per_key_array()` / `move_rec_per_key()`。
> ★ **统计有两份**：`rec_per_key`（ulong，0=未知）与 `rec_per_key_float`（float，-1.0=未知）。**优先用 float 版**。

**★ `usable_key_parts` 的两副面孔**（这是本篇最容易踩的坑）：

```cpp
// 构造期（sql_table.cc:7320）—— 存的是"KEY 在 DDL 里的原始序号"
  key_info->usable_key_parts = key_number;
```

```cpp
// sort_keys 官方注释自证
    /*
      Prefer original key order. usable_key_parts contains here
      the original key position.
    */
    return a.usable_key_parts < b.usable_key_parts;
```

```cpp
// 开表期（dd_table_share.cc:451）—— 真值：filesort 可用的非 DESC 前缀长度
      keyinfo->usable_key_parts = usable_parts;  // Filesort
```

```cpp
// usable_parts 的累加规则（table.cc:764）
  if (!(key_part->key_part_flag & HA_REVERSE_SORT) && *usable_parts == key_part_n)
    (*usable_parts)++;  // For FILESORT
```

> ★★ 语义：从第一个 key part 起**连续的非 DESC 前缀长度**——一旦遇到 DESC 就停止增长。

**`KEY::is_functional_index()`**（`key.cc:50`）：

```cpp
bool KEY::is_functional_index() const {
  for (uint i = 0; i < actual_key_parts; ++i) {      // ★ 遍历到 actual_key_parts
    if (key_part[i].field->is_field_for_functional_index()) {
      return true;
    }
  }
  return false;
}
```

> ★ **不查 flags 位**，而是查 key part 的 `Field` 是否被打上"功能索引隐藏列"标记。

| 成员 | 语义 | 构造期 / 开表期差异 |
|---|---|---|
| `name` | 索引名 | 建表：PK 强制 `"PRIMARY"`，匿名按首列名自动命名 |
| `key_part` | `KEY_PART_INFO` 数组指针 | 建表 `sql_calloc`；开表 `mem_root.ArrayAlloc` |
| `flags` | 键属性位图（**只描述用户声明的 part**） | 见下 |
| `actual_flags` | **实际**（含隐藏扩展 part）属性 | 建表 `= flags`；开表可被 `add_pk_parts_to_sk` 追加 `HA_NOSAME` |
| `user_defined_key_parts` | 用户可见 part 数 | 开表 = 跳过 hidden 元素计数 |
| `actual_key_parts` | 含扩展 PK part 的总数 | 开表 `add_pk_parts_to_sk` 递增 |
| `unused_key_parts` | 为扩展 PK 预留未用的槽 | 仅开表（预置 `primary_key_parts`） |
| `usable_key_parts` | 可用于 ref 的前缀 part 数 | 遇降序段即停（`setup_key_part_field`） |
| `key_length` | 键总字节长（含 null 标记与长度字节） | 建表只累加 part 长度；开表后处理补 null/blob 字节 |
| `block_size` / `algorithm` / `is_algorithm_explicit` | KEY_BLOCK_SIZE / 算法 | `algorithm` 是 `enum ha_key_alg` |
| `parser` / `parser_name` | FTS 解析器插件 | 开表时插件锁定 |
| `rec_per_key` / `rec_per_key_float` | 每前缀"平均记录数"双数组 | 见「运行期」 |
| `is_visible` | 优化器可见性 | → `share->visible_indexes` |
| `table` | 所属 TABLE（仅 TABLE 层拷贝置位） | |
| `comment` / `engine_attribute` / `secondary_engine_attribute` | 注解 / 引擎属性 | |
| `is_functional_index()` | 功能索引判定（遍历 part 的 `field->is_field_for_functional_index()`） | |
| `m_in_memory_estimate` | 索引驻留内存比例（`info_low` 填入） | |

> 源码核实：**不存在** `KEY::uses_uint8_key`、`free_key()`、`TABLE_SHARE::new_keys()`——KEY 数组随 `share->mem_root` 整体释放；也**不存在**克隆 KEY 的 `key_copy` 重载（克隆是开表时的 memcpy）。

#### KEY_PART_INFO 与两级 flags

```cpp
class KEY_PART_INFO { /* sql/key.h */
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

**`key_part_flag` 全部位**：`HA_SPACE_PACK`(1, MyISAM)、`HA_PART_KEY_SEG`(4, 前缀段)、`HA_VAR_LENGTH_PART`(8)、`HA_NULL_PART`(16, **仅 MyISAM 内部 keyseg**——server 用 `null_bit`/`null_offset` 表达可空)、`HA_BLOB_PART`(32)、`HA_REVERSE_SORT`(128, 降序)、`HA_BIT_PART`(1024)。

**`KEY::flags` 关键位**：`HA_NOSAME`(1)、`HA_PACK_KEY`/`HA_BINARY_PACK_KEY`（**唯一显式持久化进 DD options "flags" 的位**，其余可由 DD 对象属性重建）、`HA_FULLTEXT`(1<<7)、`HA_SPATIAL`(1<<10)、`HA_GENERATED_KEY`(1<<13, FK 自动生成)、`HA_KEY_HAS_PART_KEY_SEG`(1<<16, **SQL 层内部位不入 DD**)、`HA_KEY_RENAMED`(1<<17, ALTER 改名标记)、`HA_VIRTUAL_GEN_KEY`(1<<18)、`HA_MULTI_VALUED_KEY`(1<<19, **仅开表设置**——建表阶段由 `count_keys` 校验但不在 KEY 置位)。

#### Key_spec：解析器产物

⚠️ **勘误**：`Key_part_spec` **不在 `sql/field.h`**（grep 0 命中），真身在 **`sql/key_spec.h:90`**。

```cpp
// parser获取的结构, 在 mysql_prepare_create_table prepare_key变成 key + key_part_info
class Key_spec {
 public:
  const keytype type;                         // PRIMARY/UNIQUE/MULTIPLE/FULLTEXT/SPATIAL/FOREIGN
  const KEY_CREATE_INFO key_create_info;      // algorithm / block_size / parser_name / comment / is_visible / engine attrs
  Mem_root_array<Key_part_spec *> columns;    // ★ 不是 List，是 Mem_root_array
  LEX_CSTRING name;                           // 可为 {nullptr,0}：未命名时由 prepare_key 自动生成
  const bool generated;                       // true = 为支持 FK 自动生成的索引 → KEY 上带 HA_GENERATED_KEY
  /**
    A flag to determine if we will check for duplicate indexes.
    This typically means that the key information was specified
    directly by the user (set by the parser) or a column
    associated with it was dropped.
  */
  const bool check_for_duplicate_indexes;

  Key_spec(MEM_ROOT *mem_root, keytype type_par, const LEX_CSTRING &name_arg,
           const KEY_CREATE_INFO *key_info_arg, bool generated_arg,
           bool check_for_duplicate_indexes_arg, List<Key_part_spec> &cols)
      : type(type_par), key_create_info(*key_info_arg), columns(mem_root),
        name(name_arg), generated(generated_arg),
        check_for_duplicate_indexes(check_for_duplicate_indexes_arg) {
    columns.reserve(cols.elements);
    List_iterator<Key_part_spec> it(cols);
    Key_part_spec *column;
    while ((column = it++)) columns.push_back(column);
  }

  virtual ~Key_spec() = default;              // ★ virtual：Foreign_key_spec 继承它
};
```

```cpp
enum keytype {
  KEYTYPE_PRIMARY, KEYTYPE_UNIQUE, KEYTYPE_MULTIPLE,
  KEYTYPE_FULLTEXT, KEYTYPE_SPATIAL, KEYTYPE_FOREIGN
};

enum enum_order : int { ORDER_NOT_RELEVANT = 1, ORDER_ASC, ORDER_DESC };

class KEY_CREATE_INFO {
 public:
  enum ha_key_alg algorithm = HA_KEY_ALG_SE_SPECIFIC;
  bool is_algorithm_explicit = false;
  ulong block_size = 0;
  LEX_CSTRING parser_name = {NullS, 0};
  LEX_CSTRING comment = {NullS, 0};
  bool is_visible = true;          // ★★ 与 KEY::is_visible 默认 false 相反！
  LEX_CSTRING m_engine_attribute = EMPTY_CSTR;
  LEX_CSTRING m_secondary_engine_attribute = EMPTY_CSTR;
};
```

> ★★ **`KEY::is_visible` 默认 false，`KEY_CREATE_INFO::is_visible` 默认 true**。`prepare_key()` 里用后者覆盖前者，所以**建表默认可见**。

**`Key_part_spec` 的三个构造形态**（★ 前两个是函数索引，第三个是普通列）：

```cpp
class Key_part_spec {
 public:
  Key_part_spec(Item *expression, enum_order order)          // ① 匿名表达式
      : m_is_ascending((order == ORDER_DESC) ? false : true),
        m_is_explicit(order != ORDER_NOT_RELEVANT),
        m_field_name(nullptr), m_prefix_length(0),
        m_expression(expression), m_has_expression(true) {}

  Key_part_spec(const char *column_name, Item *expression, enum_order order)  // ② 具名表达式
      : ..., m_field_name(column_name), ..., m_has_expression(true) {}

  Key_part_spec(LEX_CSTRING column_name, uint prefix_length, enum_order order)  // ③ 普通列 + 前缀
      : ..., m_field_name(column_name.str), m_prefix_length(prefix_length),
        m_expression(nullptr), m_has_expression(false) {}

  bool operator==(const Key_part_spec &other) const {
    return get_prefix_length() == other.get_prefix_length() &&
           is_ascending() == other.is_ascending() &&
           !my_strcasecmp(system_charset_info, get_field_name(),
                          other.get_field_name());
    //  ★★ 表达式不参与比较！只比 前缀长度 + 方向 + 列名
  }

  bool resolve_expression(THD *thd) {
    assert(has_expression());
    if (get_expression()->fixed) return false;
    get_expression()->allow_array_cast();     // ★ 多值索引需要 array cast
    return get_expression()->fix_fields(thd, &m_expression);
  }
  void set_name_and_prefix_length(const char *name, uint prefix_length);
  bool has_expression() const { return m_has_expression; }
  ...
};
```

> ★ `operator==` 只比列名/前缀/方向——**函数索引生成隐藏列名后才走这个比较**，所以同一个表达式建两次不会被判重复。

**`Foreign_key_spec`**：继承 `Key_spec`，但**不进 `KEY[]`**：

```cpp
class Foreign_key_spec : public Key_spec {
 public:
  const LEX_CSTRING ref_db, orig_ref_db, ref_table, orig_ref_table;
  Mem_root_array<Key_part_spec *> ref_columns;
  const fk_option delete_opt, update_opt;
  const fk_match_opt match_opt;
  const bool has_explicit_name;

  Foreign_key_spec(...)
      : Key_spec(mem_root, KEYTYPE_FOREIGN, name_arg, &default_key_create_info,
                 false,
                 false,  // We don't check for duplicate FKs.
                 cols), ...
};
```

```cpp
class FOREIGN_KEY {         // ★ 最终产物不是 KEY
 public:
  const char *name;
  const char *unique_index_name;
  uint key_parts;
  LEX_CSTRING *key_part;
  LEX_CSTRING *fk_key_part;
  LEX_CSTRING ref_db, ref_table;
  fk_option delete_opt, update_opt;
  fk_match_opt match_opt;
};
```

> ★ FK 的两条路：① `Foreign_key_spec` → `FOREIGN_KEY`（不进 `KEY[]`）；② 若子表缺索引，自动生成一个 `Key_spec(generated=true)` → 进 `KEY[]` 且带 `HA_GENERATED_KEY`。

**`Key_spec` ↔ `KEY` 字段对照**（★ 哪些是 KEY 独有的）：

| Key_spec 侧 | KEY 侧 | 转换点 |
|---|---|---|
| `type` | `flags`（HA_NOSAME/FULLTEXT/SPATIAL）+ `algorithm` | `sql_table.cc:7278-7311 / 7332-7374` |
| `key_create_info.is_visible` | `KEY::is_visible` | `:7322` |
| `key_create_info.comment` | `comment` + `HA_USES_COMMENT` | `:7247-7266` |
| `columns.size()` | `user_defined_key_parts` / `actual_key_parts` | `:7317-7318` |
| `Key_part_spec::get_prefix_length()` | `KEY_PART_INFO::length` + `HA_PART_KEY_SEG` | `:5136-5189, 5268-5272` |
| `Key_part_spec::is_ascending()` | `key_part_flag & HA_REVERSE_SORT` | `:5134` |
| `generated` | `flags |= HA_GENERATED_KEY` | `:7314` |
| **无** | `key_length` / `key_part` / `actual_flags` / `unused_key_parts` / `usable_key_parts` / `rec_per_key*` / `parser` | 建表期算 / **开表期才填** |
| **无** | `KEY_PART_INFO::field / null_offset / null_bit / store_length / type / bin_cmp` | **建表期完全不填**，`init_from_field()` 在开表期填 |

> ★★ **一句话**：`Key_spec` 只承载 **DDL 文本**能表达的信息；`KEY` 额外承载 **引擎/开表期计算**产物。这就是"构造期与开表期字段不对称"的根因。

`sql/key_spec.h`：`Key_spec` 含 `keytype type`（`KEYTYPE_PRIMARY/UNIQUE/MULTIPLE/FULLTEXT/SPATIAL`）、`KEY_CREATE_INFO key_create_info`（algorithm / block_size / parser_name / comment / is_visible）、`columns`（`Key_part_spec` 列表）、`name`、`generated`（FK 自动生成）。`Key_part_spec` 含 `m_is_ascending`、`m_prefix_length`、`m_expression`（功能索引表达式）。

★ `Key_spec` 是**纯语法层**产物：它只记"用户写了什么"，不含任何字节布局信息——这正是不对称的源头之一。

### 构造路径一：CREATE TABLE

⚠️ **勘误**：**不存在 `MAX_KEY_NAME` 宏**。索引名长度用 `check_string_char_length(key->name, "", NAME_CHAR_LEN, ...)` → 报 `ER_TOO_LONG_IDENT`。

**`mysql_prepare_create_table()` 的 KEY 数组生成段**（`sql_table.cc:8146`）：

```cpp
  uint key_parts;
  Mem_root_array<bool> redundant_keys(thd->mem_root,
                                      alter_info->key_list.size(), false);
  if (count_keys(alter_info->key_list, key_count, &key_parts, fk_key_count,
                 &redundant_keys, file->ha_table_flags()))
    return true;
  if (*key_count > file->max_keys()) {
    my_error(ER_TOO_MANY_KEYS, MYF(0), file->max_keys());
    return true;
  }

  /*
    Make KEY objects for the keys in the new table.
  */
  KEY *key_info;
  (*key_info_buffer) = key_info = (KEY *)sql_calloc(sizeof(KEY) * (*key_count));
  KEY_PART_INFO *key_part_info =
      (KEY_PART_INFO *)sql_calloc(sizeof(KEY_PART_INFO) * key_parts);
  //  ★★ 全表 KEY_PART_INFO 一次性 calloc，由 prepare_key 里的 (*key_part_info)++ 逐个切给每个 KEY

  Mem_root_array<const KEY *> keys_to_check(thd->mem_root);
  if (keys_to_check.reserve(*key_count)) return true;

  uint key_number = 0;
  bool primary_key = false;

  // First prepare non-foreign keys so that they are ready when
  // we prepare foreign keys.
  for (size_t i = 0; i < alter_info->key_list.size(); i++) {
    if (redundant_keys[i]) continue;              // Skip redundant keys
    const Key_spec *key = alter_info->key_list[i];

    if (key->type == KEYTYPE_PRIMARY) {
      if (primary_key) {
        my_error(ER_MULTIPLE_PRI_KEY, MYF(0));    // ★ "只能有一个 PRIMARY" 在这里查
        return true;
      }
      primary_key = true;
    }

    if (key->type != KEYTYPE_FOREIGN) {
      if (prepare_key(thd, error_schema_name, error_table_name, create_info,
                      &alter_info->create_list, key, key_info_buffer, key_info,
                      &key_part_info, keys_to_check, key_number, file,
                      &auto_increment))
        return true;
      key_info++;
      key_number++;                               // ★ 这个就是 usable_key_parts 的"原始序号"
    }
  }
  ...
  /* 重复索引检测：延后批量做 */
  for (dup_check_key = keys_to_check.begin(); dup_check_key != keys_to_check.end();
       dup_check_key++) {
    if (check_duplicate_key(thd, error_schema_name, error_table_name,
                            *dup_check_key, *key_info_buffer, *key_count, alter_info))
      return true;
  }

  if (!primary_key && check_promoted_index(file, *key_info_buffer, *key_count))
    return true;

  if (auto_increment > 0) { my_error(ER_WRONG_AUTO_KEY, MYF(0)); return true; }

  /* Sort keys in optimized order */
  std::sort(*key_info_buffer, *key_info_buffer + *key_count, sort_keys());
```

**★ `prepare_key()`：Key_spec → KEY 的唯一入口**（`sql_table.cc:7183`）。关键段：

```cpp
  /* Create the key name based on the first column (if not given) */
  if (key->type == KEYTYPE_PRIMARY)
    key_info->name = primary_key_name;            // ★ 规则 1：PK 恒为 "PRIMARY"
  else if (key->name.str)
    key_info->name = key->name.str;               // ★ 规则 2：用户指定名
  else {
    ...
    key_info->name =
        make_unique_key_name(sql_field->field_name, *key_info_buffer, key_info);
    //  ★ 规则 3：首列名，冲突则 c → c_2 → c_3 …
  }
  if (key->type != KEYTYPE_PRIMARY &&
      check_if_keyname_exists(key_info->name, *key_info_buffer, key_info)) {
    my_error(ER_DUP_KEYNAME, MYF(0), key_info->name);
    return true;
  }
```

```cpp
  switch (key->type) {                            // ★ keytype → HA_* flags 映射表
    case KEYTYPE_MULTIPLE: break;
    case KEYTYPE_FULLTEXT:
      if (!(file->ha_table_flags() & HA_CAN_FULLTEXT)) {
        my_error(ER_TABLE_CANT_HANDLE_FT, MYF(0)); return true;
      }
      key_info->flags |= HA_FULLTEXT;
      if (key->key_create_info.parser_name.str) {
        key_info->parser_name = key->key_create_info.parser_name;
        key_info->flags |= HA_USES_PARSER;
      } else key_info->parser_name = NULL_CSTR;
      break;
    case KEYTYPE_SPATIAL:
      if (!(file->ha_table_flags() & HA_CAN_RTREEKEYS)) {
        my_error(ER_TABLE_CANT_HANDLE_SPKEYS, MYF(0)); return true;
      }
      if (key->columns.size() != 1) { my_error(ER_TOO_MANY_KEY_PARTS, MYF(0), 1); return true; }
      key_info->flags |= HA_SPATIAL;
      break;
    case KEYTYPE_PRIMARY:
    case KEYTYPE_UNIQUE:
      key_info->flags |= HA_NOSAME;               // ★ PRIMARY 与 UNIQUE 都走 HA_NOSAME
      break;
  }
  if (key->generated) key_info->flags |= HA_GENERATED_KEY;
```

```cpp
  key_info->user_defined_key_parts = key->columns.size();
  key_info->actual_key_parts = key_info->user_defined_key_parts;   // ★ 构造期两者相等
  key_info->key_part = *key_part_info;
  key_info->usable_key_parts = key_number;                         // ★ 借用字段存原始序号
  key_info->is_algorithm_explicit = false;
  key_info->is_visible = key->key_create_info.is_visible;
  ...
  for (size_t column_nr = 0; column_nr < key->columns.size();
       column_nr++, (*key_part_info)++) {
    if (prepare_key_column(..., key->columns[column_nr], column_nr, key_info,
                           *key_part_info, file, auto_increment, &ft_key_charset))
      return true;
  }
  key_info->actual_flags = key_info->flags;       // ★★ 构造期两级 flags 相等（PK 扩展是开表期的事）

  if (key_info->key_length > file->max_key_length() && key->type != KEYTYPE_FULLTEXT) {
    my_error(ER_TOO_LONG_KEY, MYF(0), file->max_key_length());
    if (thd->is_error()) return true;             // May be silenced - see Bug#20629014
  }
```

> ⚠️ **不存在 `KEY::operator=` / `KEY::init_from_key_spec`**——转换就是 `prepare_key()` 逐字段手工搬。

**`sort_keys()`**：决定 `KEY[]` 的最终顺序（★ 之后才进 DD）：

```cpp
/*
   Sort keys according to the following properties, in decreasing order of
   importance:
   - PRIMARY KEY
   - UNIQUE with all columns NOT NULL
   - UNIQUE without partial segments
   - UNIQUE
   - without fulltext columns
   - without virtual generated columns

   This allows us to ... be sure that, if there is no PK, the set of UNIQUE
   keys candidate for promotion starts at number 0, and we can choose #0 as PK
   (it is required that PK has number 0).
*/
struct sort_keys {
  bool operator()(const KEY &a, const KEY &b) const {
    if ((a.flags ^ b.flags) & HA_NOSAME) return a.flags & HA_NOSAME;
    if (a.flags & HA_NOSAME) {
      if ((a.flags ^ b.flags) & HA_NULL_PART_KEY) return b.flags & HA_NULL_PART_KEY;
      // Sort PRIMARY KEY before other UNIQUE NOT NULL.
      if (a.name == primary_key_name) return true;   // ★★ 靠指针相等识别主键，不是 strcmp
      if (b.name == primary_key_name) return false;
      if ((a.flags ^ b.flags) & HA_KEY_HAS_PART_KEY_SEG)
        return b.flags & HA_KEY_HAS_PART_KEY_SEG;
    }
    if ((a.flags ^ b.flags) & HA_FULLTEXT) return b.flags & HA_FULLTEXT;
    if ((a.flags ^ b.flags) & HA_VIRTUAL_GEN_KEY) return b.flags & HA_VIRTUAL_GEN_KEY;
    return a.usable_key_parts < b.usable_key_parts;  // ★ 印证 usable_key_parts 的临时语义
  }
};
```

**★ KEY → `dd::Index` 的投影**（`fill_dd_indexes_from_keyinfo`，`dd_table.cc:967`）：

```cpp
    dd::Index *idx_obj = tab_obj->add_index();     // ordinal_position 由 add_index() 顺序决定
    idx_obj->set_name(key->name);
    idx_obj->set_algorithm(dd_get_new_index_algorithm_type(key->algorithm));
    idx_obj->set_algorithm_explicit(key->is_algorithm_explicit);
    idx_obj->set_visible(key->is_visible);
    idx_obj->set_type(dd_get_new_index_type(key));
    idx_obj->set_generated(key->flags & HA_GENERATED_KEY);
    ...
    /*
      Most of flags in KEY::flags bitmap can be easily calculated from other
      attributes ... so we avoid storing this redundant information in DD.

      HA_PACK_KEY and HA_BINARY_PACK_KEY are special in this respect. ...
      they, unlike other flags, do not reflect immanent property of key or its
      parts, but rather reflect our decision to apply certain optimization
      in the specific case. So it is better to store these flags explicitly
      in DD in order to avoid problems with binary compatibility ...
    */
    idx_options->set("flags", (key->flags & (HA_PACK_KEY | HA_BINARY_PACK_KEY)));
    //  ★★ KEY::flags 里只有这两个 bit 被持久化进 DD；其余全部开表期重算
    ...
    fill_dd_index_elements_from_key_parts(
        tab_obj, idx_obj, key->user_defined_key_parts, key->key_part, file,
        key == primary_key_info);   // ★ 只投影 user_defined_key_parts 个（PK 扩展不入 DD）
```

> ★★ **这是"投影层"最核心的一处设计**：`KEY::flags` 有 20+ 个位，但**只有 2 个**进 DD（因为它们无法从其他属性推导）。其余全部靠开表期重算——这也解释了为什么会出现 `actual_flags` 这个"运行期真实形态"。

```cpp
static dd::Index::enum_index_type dd_get_new_index_type(const KEY *key) {
  if (key->flags & HA_FULLTEXT) return dd::Index::IT_FULLTEXT;
  if (key->flags & HA_SPATIAL)  return dd::Index::IT_SPATIAL;
  if (key->flags & HA_NOSAME) {
    /*
      mysql_prepare_create_table() marks PRIMARY KEY by assigning
      KEY::name special value. We rely on this here and in several
      other places in server (e.g. in sort_keys()).
    */
    if (key->name == primary_key_name) return dd::Index::IT_PRIMARY;  // ★ 再次印证
    else return dd::Index::IT_UNIQUE;
  }
  return dd::Index::IT_MULTIPLE;
}
```

**`fill_dd_index_elements_from_key_parts()`**（`dd_table.cc:829`）：

```cpp
    dd::Index_element *idx_elem = idx_obj->add_element(const_cast<dd::Column *>(key_col_obj));
    idx_elem->set_length(key_part->length);       // ★ 前缀长度入 DD（0 = 全列）

    /*
      Set index order: DD 里存的 ORDER 其实是"问 handler 出来的"，不是直接从 KEY 拷贝
    */
    if (file->index_flags(idx_obj->ordinal_position() - 1, key_part_no, false) & HA_READ_ORDER)
      idx_elem->set_order(key_part->key_part_flag & HA_REVERSE_SORT
                              ? dd::Index_element::ORDER_DESC
                              : dd::Index_element::ORDER_ASC);
    else
      idx_elem->set_order(dd::Index_element::ORDER_UNDEF);
```

> ★ 列的 `dd::Column::column_key()`（CK_PRIMARY/CK_UNIQUE/CK_MULTIPLE）**不是**在 `fill_dd_columns_from_create_fields()` 里设的，而是**由索引反向写回**（`dd_table.cc:871-901`）。调用顺序：先建列（:2164）→ 后建索引（:2175）。

**`check_promoted_index()`**（★ 主键提升的守卫）：

```cpp
static bool check_promoted_index(const handler *file, const KEY *key_info_buffer,
                                 uint key_count) {
  bool has_unique_key = false;
  const KEY *end = key_info_buffer + key_count;
  for (const KEY *k = key_info_buffer; k < end && !has_unique_key; ++k)
    if (!(k->flags & HA_NULL_PART_KEY) && (k->flags & HA_NOSAME)) {
      has_unique_key = true;          // ★ 第一个 UNIQUE NOT NULL 索引 = PK 候选（排序保证它在 #0）
      if (!k->is_visible) {
        my_error(ER_PK_INDEX_CANT_BE_INVISIBLE, MYF(0));
        return true;                  // ★★ 会被提升为 PK 的索引不能 INVISIBLE
      }
    }
  if (!has_unique_key && (file->ha_table_flags() & HA_REQUIRE_PRIMARY_KEY)) {
    my_error(ER_REQUIRES_PRIMARY_KEY, MYF(0));
    return true;
  }
  return false;
}
```

`mysql_prepare_create_table`（`sql/sql_table.cc`）里索引相关的六步：

1. `prepare_create_field` 处理所有列；
2. **功能索引表达式解析 → 添加隐藏生成列**到 create_list（`add_functional_index_to_create_list`）——此后 `fieldnr` 把隐藏 gcol 当作普通列计数；
3. `calculate_field_offsets`——**此后 part 的 `offset` 才有意义**；
4. `count_keys`：计数、剔冗余 FK 键、**校验引擎能力**（`HA_DESCENDING_INDEX`、多值索引支持）；
5. `sql_calloc` 分配 KEY 数组；
6. 每个非 FK `Key_spec` 调 `prepare_key`，再 `sort_keys`（PK 排第一）。

**`prepare_key`** 要点：PK 强制命名 `"PRIMARY"`；按 `key->type` 置 `HA_NOSAME` / `HA_FULLTEXT` / `HA_SPATIAL`；`user_defined_key_parts = actual_key_parts = columns.size()`；`usable_key_parts` 赋 key_number 占位（开表重算）；循环调 `prepare_key_column`；最后 `actual_flags = flags`。

**`prepare_key_column`** 每个 part 的计算：按名在 create_list 找 `Create_field`（功能索引跳隐藏列）→ `fieldnr = field`、`offset = sql_field->offset` → **降序** `key_part_flag |= HA_REVERSE_SORT`（**唯一写入点**）→ 前缀截断/校验 → `length` → `HA_PACK_KEY`/`HA_BINARY_PACK_KEY` → 前缀段置 `HA_PART_KEY_SEG` + 键级 `HA_KEY_HAS_PART_KEY_SEG` → `key_length += key_part_length`。

> CREATE 阶段**不设置** `null_bit` / `null_offset` / `store_length` / `type` / `bin_cmp`——开表后处理才补算。

**KEY → `dd::Index`**（`fill_dd_indexes_from_keyinfo`，`sql/dd_table.cc`）：逐 KEY 建 `dd::Index`（name / algorithm / visible / type / comment）；`dd_get_new_index_type` 靠 `name == "PRIMARY"` 判 `IT_PRIMARY`；elements 由 `fill_dd_index_elements_from_key_parts` 填充（`set_length(key_part->length)`、降序 `set_order(ORDER_DESC)`）；**主键提升发生在这里**——`is_candidate_primary_key` 选中后把列标 `CK_PRIMARY`，但 `dd::Index` 类型仍 `IT_UNIQUE`。

### 构造路径二：ALTER ADD INDEX

1. `mysql_prepare_alter_table` → `prepare_fields_and_keys`：把**旧表每个 KEY 反解为 `Key_spec`**（`HA_REVERSE_SORT → ORDER_DESC`；功能索引 part 用 `gcol_info->expr_item` 重建表达式），与新 `alter_info->key_list` 合并；
2. 后续**完全复用 CREATE 路径**（`create_table_impl` → `mysql_prepare_create_table`）产出新 KEY 数组；
3. `fill_alter_inplace_info` 新旧 KEY 对比：新键不在旧表 → `add_added_key`（进 `index_add_buffer`）；
4. InnoDB inplace：`prepare_inplace_alter_table_dict` → `innobase_create_key_defs` → **`innobase_create_index_def`**（KEY → `ddl::Index_defn`）：

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

`innobase_create_index_field_def`：`m_is_multi_value = innobase_is_multi_value_fld(field)`、`m_is_ascending = !(key_part_flag & HA_REVERSE_SORT)`、`m_col_no = fieldnr - 虚拟列数`。最终 `ddl::create_index` 构造 `dict_index_t`。

> **注意这里的"语义降解"**：`HA_REVERSE_SORT`（server 的降序）转成 `m_is_ascending`，`HA_MULTI_VALUED_KEY` 转成 `m_is_multi_value`——SQL 层的位在引擎侧各有对应，但表达形式完全不同。

**COPY 算法**则直接 `handler::create(..., key_info, key_count)` 把 KEY 数组传给引擎。**内部临时表**不经过上述链路，在 `create_tmp_table` 用 `KEY_PART_INFO::init_from_field` 现场构造（此时**补全** `null_bit` / `store_length` / `type` / `bin_cmp`——因为临时表没有"先建后开"的两阶段）。

### 构造路径三：开表回读

**分配**（`fill_indexes_from_dd`，`sql/dd_table_share.cc:1460`）：统计 keys / key_parts（**跳过 hidden 索引与 hidden 元素**）；**为扩展 PK 预留空间**（这就是 `unused_key_parts` 的由来）；`mem_root.ArrayAlloc<KEY>` + key_part + 双 rec_per_key 数组。

⚠️ **两个重要勘误**：

1. **`share->key_info` 的分配点在 `fill_indexes_from_dd()`（`dd_table_share.cc:1526`），不是 `open_table_from_share()`**——后者只做 share→TABLE 的 memcpy。
2. **不存在 `fill_keyinfo_from_dd_index()`**——真名是 `fill_indexes_from_dd`(1465) → `fill_index_from_dd`(1274) → `fill_index_elements_from_dd`(1246) → `fill_index_element_from_dd`(1210)。
3. `open_binary_frm()`（`table.cc:1418`）里那段含 `add_pk_parts_to_sk` 的代码是 **5.7 frm 升级遗留路径**，不是正常开表路径——别把它当主路径读。

```cpp
static bool fill_indexes_from_dd(THD *thd, TABLE_SHARE *share, const dd::Table *tab_obj) {
  uint32 primary_key_parts = 0;

  bool use_extended_sk = ha_check_storage_engine_flag(
      share->db_type(), HTON_SUPPORTS_EXTENDED_KEYS);   // ★★ extended keys 在此一次性静态决定

  for (const dd::Index *idx_obj : tab_obj->indexes()) {
    if (idx_obj->is_hidden()) continue;                 // ★ 跳过隐藏索引
    share->keys++;
    uint key_parts = 0;
    for (const dd::Index_element *idx_ele : idx_obj->elements()) {
      if (!idx_ele->is_hidden()) key_parts++;           // ★ 跳过隐藏元素
    }
    share->key_parts += key_parts;
    if (idx_obj->ordinal_position() == 1) primary_key_parts = key_parts;
    //  ★ 1 号索引（= PK 或被提升的候选键）的 part 数，决定每个 SK 预留多少 unused slot
  }

  if (share->keys) {
    uint total_key_parts = share->key_parts;
    if (use_extended_sk)
      total_key_parts += (primary_key_parts * (share->keys - 1));
    //  ★★ 为每个 SK 多分配 primary_key_parts 个 KEY_PART_INFO 槽（= "PK 扩展"预留）

    if (!(rec_per_key = (ulong *)share->mem_root.Alloc(total_key_parts * sizeof(ulong))))
      return true;
    if (!(rec_per_key_float = (rec_per_key_t *)share->mem_root.Alloc(
              total_key_parts * sizeof(rec_per_key_t))))
      return true;
    if (!(share->key_info = share->mem_root.ArrayAlloc<KEY>(share->keys)))   // ★★ 真正的分配点
      return true;
    if (!(key_part = share->mem_root.ArrayAlloc<KEY_PART_INFO>(total_key_parts)))
      return true;
    ...
    do {
      KEY *keyinfo = &share->key_info[key_nr];
      keyinfo->key_part = key_part;
      keyinfo->set_rec_per_key_array(rec_per_key, rec_per_key_float);
      keyinfo->set_in_memory_estimate(IN_MEMORY_ESTIMATE_UNKNOWN);

      fill_index_elements_from_dd(share, index_at_pos[key_nr], key_nr);

      key_part += keyinfo->user_defined_key_parts;
      rec_per_key += keyinfo->user_defined_key_parts;
      rec_per_key_float += keyinfo->user_defined_key_parts;

      keyinfo->actual_key_parts = keyinfo->user_defined_key_parts;   // ★ 先令两者相等
      if (use_extended_sk && key_nr && !(keyinfo->flags & HA_NOSAME)) {
        keyinfo->unused_key_parts = primary_key_parts;               // ★★ 预留槽位数
        key_part += primary_key_parts;
        rec_per_key += primary_key_parts;
        rec_per_key_float += primary_key_parts;
        share->key_parts += primary_key_parts;
      }

      for (uint kp = 0; kp < keyinfo->actual_key_parts; ++kp) {
        keyinfo->rec_per_key[kp] = 0;                                // ★ 开表时全是"未知"
        keyinfo->set_records_per_key(kp, REC_PER_KEY_UNKNOWN);
      }
      key_nr++;
    } while (key_nr < share->keys);
  }
  return false;
}
```

**`fill_index_from_dd()` 的 flags 是整体赋值，不是 `|=`**：

```cpp
  switch (idx_obj->type()) {
    case dd::Index::IT_MULTIPLE:  keyinfo->flags = 0;          break;   // ★ 注意是 `=`
    case dd::Index::IT_FULLTEXT:  keyinfo->flags = HA_FULLTEXT; break;
    case dd::Index::IT_SPATIAL:   keyinfo->flags = HA_SPATIAL;  break;
    case dd::Index::IT_PRIMARY:
    case dd::Index::IT_UNIQUE:    keyinfo->flags = HA_NOSAME;   break;
  }
  if (idx_obj->is_generated()) keyinfo->flags |= HA_GENERATED_KEY;

  /* key_length 从 DD 长度累加（后面 prepare_share 还会为 NULL/BLOB +HA_KEY_NULL_LENGTH/BLOB_LENGTH） */
  for (const dd::Index_element *idx_elem : idx_obj->elements())
    if (!idx_elem->is_hidden()) keyinfo->key_length += idx_elem->length();
```

```cpp
  /* 还原 DD 里持久化的 HA_PACK_KEY / HA_BINARY_PACK_KEY */
  uint32 stored_flags = 0;
  idx_options.get("flags", &stored_flags);
  assert((stored_flags & ~(HA_PACK_KEY | HA_BINARY_PACK_KEY)) == 0);
  /* ...8.0.12 前遗留：若 SE 不支持则清掉这两个 bit（否则 ALTER 算法分析会误判）... */
  if (ha_check_storage_engine_flag(share->db_type(), HTON_SUPPORTS_PACKED_KEYS) == 0)
    stored_flags &= ~(HA_PACK_KEY | HA_BINARY_PACK_KEY);
  keyinfo->flags |= stored_flags;
```

**元素的 `fieldnr` 口径与构造期不同**：

```cpp
static void fill_index_element_from_dd(TABLE_SHARE *share,
                                       const dd::Index_element *idx_elem_obj,
                                       KEY_PART_INFO *keypart) {
  keypart->length = idx_elem_obj->length();
  keypart->store_length = keypart->length;
  // fieldnr
  keypart->fieldnr = idx_elem_obj->column().ordinal_position();   // ★ 1-based，含隐藏列
  Field *field = keypart->field = share->field[keypart->fieldnr - 1];
  // offset
  keypart->offset = field->offset(share->default_values);         // ★ 绝对偏移（含 null 头）
  // key type
  keypart->bin_cmp = ((field->real_type() != MYSQL_TYPE_VARCHAR &&
                       field->real_type() != MYSQL_TYPE_STRING) ||
                      (field->charset()->state & MY_CS_BINSORT));
  if (idx_elem_obj->order() == dd::Index_element::ORDER_DESC)
    keypart->key_part_flag |= HA_REVERSE_SORT;                    // ★ 降序还原
}
```

> ★★ **构造期的 `fieldnr` 是 `Create_field` 在 `create_list` 里的下标；开表期是 `dd::Column::ordinal_position()`（1-based，含隐藏列）。两者口径不同**——而 `check_duplicate_key()` 敢直接比 `fieldnr`，是因为"both KEY objects come from `mysql_prepare_create_table()`"。

**后处理段（`prepare_share()`，`dd_table_share.cc:292`）—— 四步的源码**：

```cpp
      uint usable_parts = 0;
      ...
      for (uint i = 0; i < keyinfo->user_defined_key_parts; key_part++, i++) {
        Field *field = key_part->field;
        key_part->type = field->key_type();                       // ① 补 type（构造期不填）
        if (field->is_nullable()) {
          key_part->null_offset = field->null_offset(share->default_values);
          key_part->null_bit = field->null_bit;                   // ② 补 null 信息
          key_part->store_length += HA_KEY_NULL_LENGTH;
          keyinfo->flags |= HA_NULL_PART_KEY;                     // ★ 构造期由 prepare_key_column 设；开表期重算
          keyinfo->key_length += HA_KEY_NULL_LENGTH;
        }
        if (field->type() == MYSQL_TYPE_BLOB ||
            field->real_type() == MYSQL_TYPE_VARCHAR ||
            field->type() == MYSQL_TYPE_GEOMETRY) {
          key_part->store_length += HA_KEY_BLOB_LENGTH;
          if (i + 1 <= keyinfo->user_defined_key_parts) keyinfo->key_length += HA_KEY_BLOB_LENGTH;
        }
        key_part->init_flags();
        if (field->is_virtual_gcol()) keyinfo->flags |= HA_VIRTUAL_GEN_KEY;

        setup_key_part_field(share, handler_file, primary_key, keyinfo, key, i,
                             &usable_parts, true);                // ③ 算 usable_parts
        ...
        if (field->key_length() != key_part->length)
          key_part->key_part_flag |= HA_PART_KEY_SEG;             // ★ 开表期重算"前缀段"
      }

      /*
        KEY::flags is fully set-up at this point so we can copy it to
        KEY::actual_flags.
      */
      keyinfo->actual_flags = keyinfo->flags;                     // ★ A：flags 稳定后才快照

      if (primary_key < MAX_KEY && key != primary_key &&
          (ha_option & HA_PRIMARY_KEY_IN_READ_INDEX))
        key_part += add_pk_parts_to_sk(keyinfo, key, share->key_info, primary_key,
                                       share, handler_file, &usable_parts,
                                       use_extended_sk);           // ★★ PK 扩展的唯一入口

      key_part += keyinfo->unused_key_parts;                       // 跳过未用槽位
      keyinfo->usable_key_parts = usable_parts;  // Filesort        // ★ B：真值
```

> ★★ **顺序很关键**：`actual_flags = flags` 必须在 `add_pk_parts_to_sk()` **之前**——因为后者只改 `actual_flags` 不改 `flags`。这就是"两级 flags"的全部秘密：**`flags` = 用户声明（进 DD），`actual_flags` = 运行期真实形态（含 PK 扩展带来的唯一性）**。

**`HTON_SUPPORTS_EXTENDED_KEYS`**（⚠️ **不存在 `HA_EXTENDED_KEYS`**）：

```cpp
/**
  Engine supports extended keys. The flag allows to
  use 'extended key' feature if the engine is able to
  do it (has primary key values in the secondary key).
  Note that handler flag HA_PRIMARY_KEY_IN_READ_INDEX is
  actually partial case of HTON_SUPPORTS_EXTENDED_KEYS.
*/
#define HTON_SUPPORTS_EXTENDED_KEYS (1 << 10)      // sql/handler.h:2903
```

> ⚠️ **`handler::ha_index_init()` 里没有 extended keys 协商**——它只设 `active_index`。extended keys 完全在 **TABLE_SHARE 打开期**一次性静态决定（`table.cc:1531` / `dd_table_share.cc:1473`），运行期不再协商。声明方：`ha_innodb.cc:5173`（InnoDB）、`temptable/src/plugin.cc:59`（TempTable）。

**`fill_index_from_dd`** 关键：

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

元素回填（`fill_index_element_from_dd`）：`length`、`fieldnr = column().ordinal_position()`、`field = share->field[fieldnr-1]`、**`offset = field->offset(share->default_values)`（绝对偏移，含 null 头——与建表期的列内偏移不同！）**、`bin_cmp` 判定、`ORDER_DESC → HA_REVERSE_SORT`；`field->is_array() → flags |= HA_MULTI_VALUED_KEY`（多值的**唯一**设置点）。

**后处理**（`open_table_share` 内）四步：

1. 候选唯一键提升为主键；
2. **逐 part 补算 null/blob 信息**（`null_offset` / `null_bit` / `store_length`、`key_length += HA_KEY_NULL_LENGTH` 等）；
3. `setup_key_part_field`（置 Field 的 key 位图、算 `usable_parts`，遇降序段停）；
4. `actual_flags = flags`，然后 **`add_pk_parts_to_sk`**。

#### add_pk_parts_to_sk 逐段：隐藏 PK part 的追加规则

```cpp
uint add_pk_parts_to_sk(KEY *sk, uint sk_n, KEY *pk, uint pk_n, TABLE_SHARE *share,
                        handler *handler_file, uint *usable_parts, bool use_extended_sk) {
  uint max_key_length = sk->key_length;
  /*
    Secondary key becomes unique if the key does not exceed
    key length limitation(MAX_KEY_LENGTH) and key parts
    limitation(MAX_REF_PARTS) and PK parts are added to SK.
  */
  bool is_unique_key = use_extended_sk;
  ...
  for (; pk_part < pk->user_defined_key_parts; pk_part++) {
    if (sk->actual_key_parts >= MAX_REF_PARTS) { is_unique_key = false; break; }
    bool pk_field_is_in_sk = /* 该 PK 列是否已在 SK 中（且长度一致） */;
    if (!pk_field_is_in_sk) {
      if (max_key_length + pk_key_part->length > MAX_KEY_LENGTH) { is_unique_key = false; break; }
      max_key_length += pk_key_part->length;
      /* Do not add key part if SK is a unique key or if use_index_extensions is OFF. */
      if ((sk->flags & HA_NOSAME) || !use_extended_sk) continue;
      *current_key_part = *pk_key_part;
      setup_key_part_field(...);
      sk->actual_key_parts++;  sk->unused_key_parts--;
      sk->set_records_per_key(sk->actual_key_parts - 1, REC_PER_KEY_UNKNOWN);
      current_key_part++;
    }
  }
  if (is_unique_key) sk->actual_flags |= HA_NOSAME;
  /* 超限的 PK part 清理位图 */
  for (; pk_part < pk->user_defined_key_parts; pk_part++) {
    fld->part_of_key.clear_bit(sk_n);
    fld->part_of_sortkey.clear_bit(sk_n);
  }
}
```

★ 四条规则（也是四个失效点）：

1. **已经是唯一键则不追加**（`sk->flags & HA_NOSAME`）——它本来就唯一，无需扩展；
2. **`use_index_extensions` 开关关掉则完全不追加**（优化器扩展失效，见下）；
3. **段数超过 `MAX_REF_PARTS` 或总长超过 `MAX_KEY_LENGTH`（3072 字节）则停止追加，且 `is_unique_key = false`**——这是"二级索引实际唯一"**不成立的典型场景**（长主键 / 多列主键）；
4. 超限后**还要清理**被放弃的 PK part 的 `part_of_key` / `part_of_sortkey` 位图——否则优化器会以为这些列仍被该索引覆盖。

### ★ 功能索引：投影设计的最佳例证

功能索引把"SQL 语义比引擎丰富"这一点体现得最彻底：

```
CREATE INDEX idx ON t ((col1 + col2))     -- SQL 层：索引一个表达式
```

引擎里没有"表达式索引"这种东西。MySQL 的做法是**降解为已有机制**：

1. 建表准备阶段，`add_functional_index_to_create_list` 为每个功能索引 part **创建一列隐藏的生成列**（`gcol_info` 里存表达式）；
2. 此后 `fieldnr` 把该隐藏列当作普通列，走**完全相同的** `prepare_key_column` 路径；
3. 引擎侧看到的就是"一个虚拟列上的普通索引"（`DICT_VIRTUAL`）；
4. 开表回读时反向：功能索引 part 用 `gcol_info->expr_item` 把表达式**重建**回 `Key_spec`（这样 `SHOW CREATE` 才能显示原始表达式）。

★ **为什么必须这么做**（约束）：如果为表达式索引新增一种索引类型，就要改引擎的索引构建、记录格式、比较器、统计采集……**成本不可控**。降解成"隐藏生成列 + 普通索引"后，**引擎侧零改动**——这是 MySQL 一贯的做法（GIPK 隐藏主键、多值索引的隐藏列都是同一套路）。

**① 隐藏列的命名规则**（`sql_table.cc:7690`）：

```cpp
static const char *make_functional_index_column_name(
    std::string_view key_name, unsigned key_part_number,
    const List<Create_field> &fields, MEM_ROOT *mem_root) {
  for (unsigned count = 0;; ++count) {
    string name("!hidden!");           // ★ 前缀 !hidden! = SQL 层隐藏列标志
    name += key_name;

    string suffix("!");
    suffix += to_string(key_part_number);
    suffix += '!';
    suffix += to_string(count);

    /* If the name is so long that we hit the NAME_CHAR_LEN limit, truncate the
    index name part, so that there is enough room to add the suffix with the
    key part number and the counter. (If we had truncated the counter, we
    could loop forever because the generated name is the same in each
    iteration.) */
    name.resize(min(name.size(), NAME_CHAR_LEN - suffix.size()));
    name.append(suffix);

    if (column_name_exists(fields, name)) continue;
    return strmake_root(mem_root, name.data(), name.size());
  }
}
```

> ★ 格式 `!hidden!{索引名}!{part序号}!{冲突计数}`。**截断的是索引名部分而不是计数部分**——否则会死循环。

**② 三步降解**（`add_functional_index_to_create_list`，`sql_table.cc:7763`）：

```cpp
  // A functional index cannot be a primary key
  if (key_spec->type == KEYTYPE_PRIMARY) {
    my_error(ER_FUNCTIONAL_INDEX_PRIMARY_KEY, MYF(0));
    return nullptr;
  }

  /* ...（为匿名功能索引自动命名：functional_index → functional_index_2 → _3 …）... */

  const char *field_name = make_functional_index_column_name(...);
  Item *item = kp->get_expression();
  Create_field *cr = generate_create_field(thd, item, &tmp_table);

  if (is_blob(cr->sql_type)) {
    my_error(ER_FUNCTIONAL_INDEX_ON_LOB, MYF(0));   // ★ 表达式结果不能是 BLOB
    return nullptr;
  }

  cr->field_name = field_name;
  cr->field = nullptr;
  cr->hidden = dd::Column::enum_hidden_type::HT_HIDDEN_SQL;   // ★ SQL 层隐藏
  cr->stored_in_db = false;                                    // ★ 虚拟，不落盘

  Value_generator *gcol_info = new (thd->mem_root) Value_generator();
  gcol_info->expr_item = kp->get_expression();   // ★ 表达式 Item 树直接挂上
  gcol_info->set_field_stored(false);
  gcol_info->set_field_type(cr->sql_type);
  cr->gcol_info = gcol_info;

  /* ...（按列名排序插入 create_list 末尾，保证主从一致）... */

  alter_info->flags |= Alter_info::ALTER_ADD_COLUMN;
  kp->set_name_and_prefix_length(field_name, 0);   // ★★ 关键：把 Key_part_spec 从"表达式"改写成"指向隐藏列"
  return cr;
```

> ★ 8.0.18 起隐藏列按列名排序放在末尾（`is_not_slave_or_master_sorts_functional_index_columns_last()`，`master_version >= 80018`）——为了主从复制一致。

**③ `Value_generator` 的不变量**（`sql/field.h:483`）：

```cpp
class Value_generator {
 public:
  /**
    Item representing the generation expression.
    This is non-NULL for every Field of a TABLE, if that field is a generated
    column.
    Contrast this with the Field of a TABLE_SHARE, which has expr_item==NULL
    even if it's a generated column; that makes sense, as an Item tree cannot
    be shared.
  */
  Item *expr_item{nullptr};
  /**
    Text of the expression. ... the text read from the DD is put into the
    Value_generator::expr_str of the Field of the TABLE_SHARE; then this
    expr_str is used as source to produce expr_item for the Field of every
    TABLE derived from this TABLE_SHARE.
  */
  LEX_STRING expr_str{nullptr, 0};
```

> ★★ **`TABLE_SHARE::field` 只有 `expr_str`（文本），`TABLE::field` 才有 `expr_item`（Item 树）**——Item 树不可共享，所以每个 TABLE 都要重建。这正是"运行期拷贝"必须重建表达式的原因。

**④ DD 里怎么存**（`dd_table.cc:611`）——**`dd::Index` 上没有功能索引标记**，它被降解成"一个 `hidden='SQL'` 的虚拟生成列"：

```cpp
    col_obj->set_hidden(field.hidden);        // HT_HIDDEN_SQL 落到这里
    ...
    if (field.gcol_info) {
      /*
        It is important to normalize the expression's text into the DD, to
        make it independent from sql_mode. For example, 'a||b' means 'a OR b'
        or 'CONCAT(a,b)', depending on if PIPES_AS_CONCAT is on. Using
        Item::print(), we get self-sufficient text containing 'OR' or
        'CONCAT'. If sql_mode later changes, it will not affect the column.
       */
      col_obj->set_virtual(!field.stored_in_db);
      field.gcol_info->print_expr(thd, &gc_expr);
      col_obj->set_generation_expression(...);
      col_obj->set_generation_expression_utf8(...);   // ★ 另存一份 utf8 版专供 I_S
    }
```

**⑤ 多值索引（`HA_MULTI_VALUED_KEY`）**——⚠️ **`prepare_key()` 里没有多值分支**：

```cpp
/** Multi-valued key */
#define HA_MULTI_VALUED_KEY (1 << 19)              // my_base.h:551
/** Supports multi-valued index */
#define HA_MULTI_VALUED_KEY_SUPPORT (1LL << 55)    // handler.h:512
```

唯一的建表校验在 `count_keys()`（`sql_table.cc:4830`）：

```cpp
          if (kp->has_expression() && kp->get_expression()->returns_array()) {
            if (mv_key_parts++) {                  // ★ 一个索引只允许一个多值 part
              my_error(ER_NOT_SUPPORTED_YET, MYF(0),
                       "more than one multi-valued key part per index");
              return true;
            }
            if (!(se_index_flags & HA_MULTI_VALUED_KEY_SUPPORT)) {
              my_error(ER_CHECK_NOT_IMPLEMENTED, MYF(0), "multi-valued indexes");
              return true;
            }
            if (kp->is_explicit()) {               // ★ 禁止显式 ASC/DESC
              my_error(ER_WRONG_USAGE, MYF(0), "multi-valued index",
                       "explicit index order");
              return true;
            }
          }
```

**`HA_MULTI_VALUED_KEY` 的唯一置位点是开表期**（`dd_table_share.cc:1263`）：

```cpp
    fill_index_element_from_dd(share, idx_elem_obj, keyinfo->key_part + i);
    if (keyinfo->key_part[i].field->is_array())
      keyinfo->flags |= HA_MULTI_VALUED_KEY;      // ★★ 列是 typed array ⇒ 索引是多值
```

> ★★ **DD 不持久化"多值"这一属性**（`dd_table.cc` 全文检索 `multi_valued` 0 命中），靠开表时从列的 `is_array()` 反推。InnoDB 侧再反推 `DICT_MULTI_VALUE`（`dict0dd.cc:3009`）。
> ⚠️ 也不存在 `dd::Index::IT_MULTI_VALUED`——`enum_index_type` 只有 PRIMARY/UNIQUE/MULTIPLE/FULLTEXT/SPATIAL。

**⑥ 不可见索引**——⚠️ **不存在 `HA_INVISIBLE_KEY` 宏**（0 命中），是独立 bool + DD 列：

```cpp
  /** True if this index is visible to the query optimizer. ... */
  bool is_visible{false};                          // sql/key.h:202
```

```cpp
  m_target_def.add_field(FIELD_IS_VISIBLE, "FIELD_IS_VISIBLE",
                         "is_visible BOOL NOT NULL");   // sql/dd/impl/tables/indexes.cc:78
```

```cpp
Key_map TABLE_SHARE::usable_indexes(const THD *thd) const {
  Key_map usable_indexes(keys_in_use);
  if (!thd->optimizer_switch_flag(OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES))
    usable_indexes.intersect(visible_indexes);     // ★★ 唯一生效点
  return usable_indexes;
}
```

> ★ `OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES = 1ULL << 19`（`sql_const.h:218`），**默认 OFF**。要"假装索引不存在"就是靠这个。

**⑦ ALTER 期的可见性切换**（`fill_alter_inplace_info`，`sql_table.cc:12351`）：

```cpp
    new_key->is_visible = alter_index_visibility->is_visible();
    ha_alter_info->handler_flags |= Alter_inplace_info::RENAME_INDEX;
    //  ★★ 伪装成 RENAME，以此触发元数据更新（因为改可见性不需要重建索引）
    ha_alter_info->add_altered_index_visibility(table_key, new_key);
```

### 运行期：TABLE 层拷贝与 rec_per_key

**`TABLE::key_info` 与 `TABLE_SHARE::key_info` 不共享而是拷贝**：`open_table_from_share` → `create_key_part_field_with_prefix_length` 在 TABLE 的 mem_root 上 memcpy 整个 KEY + KEY_PART_INFO 数组，把 `key_part->field` 重指到 TABLE 自己的 field、前缀 part 生成"截断字段"、`key_info->table = table`。

★ 但 **`rec_per_key` 数组的内存仍在 `share->mem_root` 上**（TABLE 拷贝的只是指针）——统计更新跨同一 share 的多个 TABLE 实例共享。这是"每 TABLE 要自己的 Field 指针"与"统计应共享"两种需求的折中。

**填入**（`ha_innobase::info_low` 的 `HA_STATUS_CONST` 分支）：

```cpp
if (!key->supports_records_per_key()) continue;   // 两数组皆非空才继续
for (j = 0; j < key->actual_key_parts; j++) {
  if (key->flags & (HA_FULLTEXT | HA_SPATIAL)) { key->set_records_per_key(j, 1.0f); continue; }
  rec_per_key = innodb_rec_per_key(index, j, index->table->stat_n_rows);
  key->set_records_per_key(j, rec_per_key);
  /* 兼容旧接口：ulong 数组故意除以 2（认为选择性被低估） */
}
```

`innodb_rec_per_key` = `records / n_diff`（`stat_n_diff_key_vals`）。

★ **`supports_records_per_key()` 在建表 / ALTER 准备期的 KEY 数组上为 false**（从未 `set_rec_per_key_array`）——InnoDB 对 `altered_table` 调 `info_low` 时会跳过。这是"构造期 KEY 是残缺投影"的又一体现。

**消费点**：经典优化器 ref 代价、扇出估算、新代价模型、BKA、MRR 代价。

---

## 可观测性

| 想看什么 | 手段 |
|---|---|
| 用户声明的索引形态 | `SHOW CREATE TABLE`（**从 DD 重建**——看到的是 `flags` 而非 `actual_flags`） |
| 索引的元数据与隐藏 part | `I_S.STATISTICS`（注意它会显示扩展后的 `actual_key_parts` 吗——不显示隐藏 part，隐藏对 SQL 不可见）、`I_S.INNODB_INDEXES` |
| 引擎侧真实形态 | `I_S.INNODB_INDEXES` / `INNODB_FIELDS`（看 `dict_index_t` 的段构成，含追加的 PK part） |
| 统计（`rec_per_key` 的来源） | `mysql.innodb_index_stats` 的 `n_diff_pfxNN`；采集算法见 [`stats.md`](stats.md) |

⚠️ **勘误**：**不存在 `print_key()`**——索引文本由 `store_create_info()`（`sql_show.cc:1947`）内联生成，属性子句在 **`store_key_options()`**（`sql_show.cc:2604`）。

**`store_key_options()`**（★ SHOW CREATE 的属性输出）：

```cpp
static void store_key_options(THD *thd, String *packet, TABLE *table, KEY *key_info) {
  if (!foreign_db_mode) {
    /*
      Send USING clause only if key algorithm was explicitly specified
      at the table creation time.
    */
    if (key_info->is_algorithm_explicit) {
      if (key_info->algorithm == HA_KEY_ALG_BTREE) packet->append(STRING_WITH_LEN(" USING BTREE"));
      if (key_info->algorithm == HA_KEY_ALG_HASH)  packet->append(STRING_WITH_LEN(" USING HASH"));
      if (key_info->algorithm == HA_KEY_ALG_RTREE) {
        /* We should send USING only in non-default case: non-spatial rtree. */
        assert(!(key_info->flags & HA_SPATIAL));
        packet->append(STRING_WITH_LEN(" USING RTREE"));
      }
    }
    if ((key_info->flags & HA_USES_BLOCK_SIZE) &&
        table->s->key_block_size != key_info->block_size) {
      packet->append(STRING_WITH_LEN(" KEY_BLOCK_SIZE="));
      end = longlong10_to_str(key_info->block_size, buff, 10);
      packet->append(buff, (uint)(end - buff));
    }
    if (key_info->flags & HA_USES_COMMENT) {
      packet->append(STRING_WITH_LEN(" COMMENT "));
      append_unescaped(packet, key_info->comment.str, key_info->comment.length);
    }
    if (!key_info->is_visible)
      packet->append(STRING_WITH_LEN(" /*!80000 INVISIBLE */"));   // ★ 不可见索引
    if (key_info->engine_attribute.length > 0) {
      packet->append(STRING_WITH_LEN(" /*!80021 ENGINE_ATTRIBUTE "));
      append_unescaped(packet, key_info->engine_attribute.str, ...);
      packet->append(STRING_WITH_LEN(" */"));
    }
    /* ... SECONDARY_ENGINE_ATTRIBUTE 同款 ... */
  }
}
```

**索引正文段**（★ 功能索引如何还原成表达式）：

```cpp
    for (uint j = 0; j < key_info->user_defined_key_parts; j++, key_part++) {
      if (j) packet->append(',');
      if (key_part->field) {
        // If this fields represents a functional index, print the expression
        // instead of the column name.
        if (key_part->field->is_field_for_functional_index()) {
          StringBuffer<STRING_BUFFER_USUAL_SIZE> s;
          s.set_charset(system_charset_info);
          key_part->field->gcol_info->print_expr(thd, &s);
          packet->append("(");
          packet->append(s);
          packet->append(")");            // ★★ 打印 ((col1 + col2)) 而不是 !hidden!...
        } else {
          append_identifier(thd, packet, key_part->field->field_name, ...);
        }
      }
      /* 前缀长度：除以 mbmaxlen 换回"字符数" */
      if (key_part->field &&
          (key_part->length != table->field[key_part->fieldnr - 1]->key_length() &&
           !(key_info->flags & (HA_FULLTEXT | HA_SPATIAL)))) {
        packet->append_parenthesized((long)key_part->length /
                                     key_part->field->charset()->mbmaxlen);
      }
      if (key_part->key_part_flag & HA_REVERSE_SORT)
        packet->append(STRING_WITH_LEN(" DESC"));
    }
```

> ★ 注意用的是 **`user_defined_key_parts`**——PK 扩展出来的隐藏 part 不显示。

**`SHOW INDEX` 的两列新字段**（`get_schema_tmp_table_keys_record`，`sql_show.cc:4242`）：

```cpp
      // is_visible column
      const char *is_visible = key_info->is_visible ? "YES" : "NO";
      table->field[TMP_TABLE_KEYS_IS_VISIBLE]->store(is_visible, strlen(is_visible), cs);

      // Expression for functional key parts
      if (key_part->field != nullptr &&
          key_part->field->is_field_for_functional_index()) {
        Value_generator *gcol = key_info->key_part->field->gcol_info;
        table->field[TMP_TABLE_KEYS_EXPRESSION]->store(
            gcol->expr_str.str, gcol->expr_str.length, cs);
        //  ⚠ 8.0.39 的粗糙点：始终取 key_info->key_part[0] 的 gcol，
        //     对多列功能索引不准
      } else {
        table->field[TMP_TABLE_KEYS_EXPRESSION]->set_null();
      }
```

> ⚠️ 隐藏列在 COLUMN_NAME 上显示 **NULL**（`is_hidden_by_system()` 判据）。
> ★ `Cardinality = 总行数 / records_per_key(j)`；**SHOW INDEX 不主动刷统计**（只调 `HA_STATUS_VARIABLE`，不含 `HA_STATUS_CONST`），所以新表可能是 `REC_PER_KEY_UNKNOWN`。

**`I_S.STATISTICS` 视图定义**（`sql/dd/impl/system_views/statistics.cc:55`）：

```cpp
  m_target_def.add_field(FIELD_COLUMN_NAME, "COLUMN_NAME",
      "IF (col.hidden = 'SQL', NULL, col.name COLLATE utf8mb3_tolower_ci)");
  m_target_def.add_field(FIELD_COLLATION, "COLLATION",
      "CASE WHEN icu.order = 'DESC' THEN 'D' WHEN icu.order = 'ASC' THEN 'A' ELSE NULL END");
  m_target_def.add_field(FIELD_IS_VISIBLE, "IS_VISIBLE",
      "IF (idx.is_visible, 'YES', 'NO')");
  m_target_def.add_field(FIELD_EXPRESSION, "EXPRESSION",
      "IF (col.hidden = 'SQL', col.generation_expression_utf8, NULL)");
```

```cpp
  /* Cardinality 是独立子查询，来自 mysql.index_stats（持久化缓存），不是实时 info() */
  m_target_def.add_field(FIELD_CARDINALITY, "CARDINALITY",
      "INTERNAL_INDEX_COLUMN_CARDINALITY(sch.name, tbl.name, idx.name, col.name, "
      "idx.ordinal_position, icu.ordinal_position, ..., "
      "COALESCE(stat.cardinality, CAST(-1 AS UNSIGNED)), "
      "COALESCE(CAST(stat.cached_time as UNSIGNED), 0))");
  m_target_def.add_from("LEFT JOIN mysql.index_stats stat ON ...");
```

**相关 sysvar**（★ 全部在 `sql/sys_vars.cc`，与索引构造直接相关）：

```cpp
// sql_const.h:218 —— optimizer_switch 的一个 bit
#define OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES (1ULL << 19)
```

| 变量 | 位置 | 与索引的关系 |
|---|---|---|
| `optimizer_switch`（`use_index_extensions` 默认 ON；`use_invisible_indexes` 默认 OFF） | `sys_vars.cc:3476` | ★ 前者控制 PK 扩展是否被优化器利用 |
| `eq_range_index_dive_limit`（默认 200） | `sys_vars.cc:3925` | 决定是用统计还是 index dive |
| `range_optimizer_max_mem_size`（默认 8388608） | `sys_vars.cc:3339` | 超限则放弃 range 优化 |
| `default_storage_engine` | `sys_vars.cc:5344` | 决定 `file->get_default_index_algorithm()` 是谁 |
| `show_gipk_in_create_table_and_information_schema`（默认 true） | `sys_vars.cc:7315` | GIPK 是否在 SHOW / I_S 里显示 |
| `key_buffer_size` | `sys_vars.cc:2353` | **仅 MyISAM** 索引块缓存 |

> ⚠️ **`innodb_stats_*` 不在 `sql/sys_vars.cc`**（本文件检索 0 命中），全部在 `storage/innobase/handler/ha_innodb.cc`。

**★ 扩展点：加一种索引属性要动的地方**（13 处）：

| # | 位置 | 做什么 | 参照 |
|---|---|---|---|
| 1 | `my_base.h` `HA_XXX` 位 | 加 `KEY::flags` 位 | `HA_MULTI_VALUED_KEY (1<<19)` |
| 2 | `sql/key.h` `class KEY` | 加成员 | `is_visible` |
| 3 | `sql/key_spec.h` `Key_spec` / `KEY_CREATE_INFO` | parser 载体 | `is_visible` |
| 4 | `sql_table.cc:7183` `prepare_key()` | `Key_spec → KEY` + 置位 | `:7322` |
| 5 | `dd_table.cc:968` `fill_dd_indexes_from_keyinfo()` | 写 DD（列 or `idx_options`） | `set_visible`、`idx_options->set("flags",…)` |
| 6 | `dd/types/index.h` + `impl/types/index_impl.*` + `impl/tables/indexes.cc` | 独立列：接口+实现+物理列+store/restore | `is_visible` |
| 7 | `dd_table_share.cc` `fill_index_from_dd()` | DD → KEY 回填 | `:1300` |
| 8 | `sql_show.cc` `store_create_info()` + `store_key_options()` | SHOW CREATE | `:2641` |
| 9 | `sql_show.cc` `get_schema_tmp_table_keys_record()` + 列定义 | SHOW KEYS | `:4389` |
| 10 | `dd/impl/system_views/statistics.cc` | I_S.STATISTICS | `:82` |
| 11 | `dict0dd.cc` `dd_fill_one_dict_index()` | KEY → dict_index_t | `:3006` |
| 12 | `handler.h` `HTON_XXX` + 能力校验 | SE 是否支持 | `HA_MULTI_VALUED_KEY_SUPPORT` |
| 13 | `sql_table.cc` ALTER 路径 | 保留/变更该属性 | `:15141`、`:12362` |
| 功能索引的隐藏列 | `SHOW CREATE TABLE` 不显示隐藏生成列；看 `I_S.COLUMNS` 的隐藏标记或 `SHOW EXTENDED` |
| 索引是否可见 / 被优化器用 | `share->visible_indexes`；`EXPLAIN` 的 `possible_keys` |
| 扩展是否生效 | `optimizer_switch` 的 **`use_index_extensions`**——关掉后二级索引不再追加隐藏 PK part。★ 开关在优化器侧的**三个消费点**与「代价估算用 `flags`、type 落地用 `actual_key_flags`」的口径分歧（及其对 eq_ref 判定的影响）见 [`../server/query/07_optimize/physical/07_access_method.md`](../server/query/07_optimize/physical/07_access_method.md) |

★ **观测上的坑**：`SHOW CREATE TABLE` 看到的是 `flags`（用户声明），而优化器用的是 `actual_flags`（含隐藏扩展）。**两者不一致是常态**——排查"为什么这个二级索引被当成唯一"时，要看 `actual_flags`，不能只看 DDL。

---

## Misc

### ★ 本机制里的工程实现技法

1. **位域 flags 的两级分工**：`KEY::flags`（键是什么）与 `KEY_PART_INFO::key_part_flag`（段是什么）分开，避免一个位域承载两种粒度；且只有真正无法从 DD 属性重建的位（`HA_PACK_KEY` 系列）才显式持久化。
2. **为扩展预留槽位**：开表分配时按 `primary_key_parts` 多分配 key_part 槽（`unused_key_parts`），避免 `add_pk_parts_to_sk` 时重新分配数组（KEY 数组在 mem_root 上不能 realloc）。
3. **拷贝而非共享**：TABLE 层 memcpy 整个 KEY 数组并重指 `field`——因为每个 TABLE 有自己的 Field 实例；但 `rec_per_key` 指针仍指向 share 的内存，实现"结构私有、统计共享"。
4. **延迟计算**：建表期跳过所有依赖最终行布局的字段（`null_bit`/`store_length`/`type`/`bin_cmp`），把一次全量计算推迟到真正需要读写行的开表期。
5. **降解而非扩展**（功能索引 / 多值索引 / GIPK）：新增 SQL 语义时不新增引擎索引类型，而是降解为"隐藏列 + 已有机制"，把改动限制在 server 层。

### 扩展点：加一种索引属性要动哪些地方

以"新增一个键级属性（类似 `is_visible`）"为例：

| # | 位置 | 改动 |
|---|---|---|
| 1 | `sql/key.h` | `KEY` 加成员（或 `flags` 加位——需确认是否要持久化） |
| 2 | `sql/key_spec.h` + parser | `Key_spec`/`KEY_CREATE_INFO` 加语法承载（否则用户没法写） |
| 3 | `sql/sql_table.cc` | `prepare_key` 里从 `Key_spec` 填进 KEY |
| 4 | `sql/dd_table.cc` | `fill_dd_indexes_from_keyinfo` 写进 `dd::Index`（**不写则开表丢失**） |
| 5 | `sql/dd_table_share.cc` | `fill_index_from_dd` 回读（与第 4 步**必须对称**） |
| 6 | `sql/dd/types/index.h` + DD 表 | 若需持久化，加属性与升级逻辑（DD 升级是硬约束，见 [`../server/dd/dd.md`](../server/dd/dd.md)） |
| 7 | 引擎侧 | 若引擎需要感知，`innobase_create_index_def` / `innobase_create_index_field_def` 转成 `ddl::Index_defn` 字段 |

★ **最容易漏的是第 4 / 5 步的对称性**：写进 DD 却没回读（或反之）在编译期无任何提示，表现为"建表时生效、重启后丢失"。

### 失效与退化场景

| 场景 | 后果 | 判据 |
|---|---|---|
| 主键过长 / 段数过多 | 二级索引**不会**变成"实际唯一"（`add_pk_parts_to_sk` 提前 break） | `max_key_length + part->length > MAX_KEY_LENGTH`(3072) 或 `actual_key_parts >= MAX_REF_PARTS` |
| `use_index_extensions=OFF` | **完全不追加**隐藏 PK part，索引扩展优化失效（ref 访问少一层） | `optimizer_switch` |
| 二级索引本身是 UNIQUE | 不追加（`flags & HA_NOSAME`）——它已唯一 | `add_pk_parts_to_sk` 的 `continue` |
| 降序段出现在前缀 | `usable_key_parts` 停在该段——**后续段不能用于 ref**（8.0 的已知限制） | `setup_key_part_field` |
| 建表期 KEY 调 `info_low` | `supports_records_per_key()` 为 false → **跳过统计填入**（此时 `rec_per_key` 不可用） | 未 `set_rec_per_key_array` |
| 前缀索引 | 生成"截断字段"，`bin_cmp` 可能为 false（不能直接二进制比较） | `create_key_part_field_with_prefix_length` |
| 功能索引 | 表达式依赖隐藏生成列；**该列隐藏**，SQL 里不能直查 | `is_field_for_functional_index()` |

### 构造期 vs 开表期不对称表

| 字段 | CREATE 阶段（`prepare_key_column`） | 开表阶段（`fill_index_from_dd` + 后处理） |
|---|---|---|
| `offset` | 列内偏移（不含 null bitmap 头） | **绝对偏移**（`field->offset(default_values)`，含 null 头） |
| `null_bit` / `null_offset` | **不填** | 后处理补算 |
| `store_length` | 不填 | `length + null + blob 长度` |
| `type` / `bin_cmp` | 不填 | 填 |
| `HA_MULTI_VALUED_KEY` | **不置位**（只校验） | `field->is_array()` 推导置位 |
| `usable_key_parts` | 占位赋 key_number | `setup_key_part_field` 实算（遇降序停） |

### 易混淆点（源码核实清单）

- **不存在 `struct KEY_PART`（存储层）**——8.0 里 `KEY_PART_INFO` 就是传给引擎的接口；引擎侧转成 `ddl::Index_field`。
- **不存在 `create_key_infos` / `fill_dd_indexes_for_keyinfo`**——实际是 `prepare_key` / `prepare_key_column` 与 `fill_dd_indexes_from_keyinfo`。
- **不存在 `KEY::uses_uint8_key`、`free_key`、`TABLE_SHARE::new_keys`**——内存随 mem_root 释放。
- **主键提升不重命名 KEY**：DD 里 `dd::Index` 类型仍 `IT_UNIQUE`，`SHOW CREATE` 显示 `UNIQUE KEY`；只有 `name == "PRIMARY"` 才显示 `PRIMARY KEY`。
- **`HA_NULL_PART` 是 MyISAM 内部 keyseg 位**——server 的 `KEY_PART_INFO` 不置它（可空由 `null_bit` / `null_offset` 表达）。
- **`HA_PACK_KEY` / `HA_BINARY_PACK_KEY` 是唯一显式持久化进 DD options 的 flags**（其余位由 DD 对象属性重建）；但 **InnoDB 不实现 pack key**（见 [`record_format.md`](record_format.md)）。
- **`HA_MULTI_VALUED_KEY` 只在开表设置**——建表阶段仅校验。
- **`rec_per_key` 数组内存属于 share，不属于 TABLE**——TABLE 只拷贝指针。

---

## 参考

**官方文档**

- *MySQL 8.0 Reference Manual → 15.1.20.9 Secondary Indexes and Generated Columns*（功能索引）
- *MySQL 8.0 Reference Manual → 13.1.8 CREATE INDEX*
- *MySQL 8.0 Reference Manual → 10.3.9 Optimizer Switch*（`use_index_extensions`）

**源码**

- `sql/key.h`——`KEY` / `KEY_PART_INFO`
- `sql/key_spec.h`——`Key_spec` / `Key_part_spec`
- `sql/sql_table.cc`——`mysql_prepare_create_table`、`prepare_key`、`prepare_key_column`、`count_keys`、`add_functional_index_to_create_list`
- `sql/dd_table.cc`——`fill_dd_indexes_from_keyinfo`
- `sql/dd_table_share.cc`——`fill_index_from_dd`、`fill_index_element_from_dd`
- `sql/table.cc`——`setup_key_part_field`、`add_pk_parts_to_sk`、`create_key_part_field_with_prefix_length`
- `sql/sql_const.h`——`MAX_KEY_LENGTH`

**相关文档**

- `dd::Index` ↔ `dict_index_t` 映射：[`metadata.md`](metadata.md)
- 索引 DDL 执行：[`operations.md`](operations.md)
- 索引记录格式：[`record_format.md`](record_format.md)
- 索引类型语义：[`types.md`](types.md)
- 统计采集（rec_per_key 的来源）：[`stats.md`](stats.md)
- DD 升级约束：[`../server/dd/dd.md`](../server/dd/dd.md)
