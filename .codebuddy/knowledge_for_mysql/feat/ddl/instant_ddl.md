# INSTANT DDL 深度剖析（server ↔ InnoDB 跨层全链路）

> 基于 MySQL 8.0.39 源码，剖析 INSTANT 算法：为什么它能做到"只改元数据、已有行一个字节都不改"，行版本号机制（表级 / 行级 / 物理三个层级极易混淆的辨析）、V1→V2 两种记录格式的字节级布局、加列与删列的实现差异，以及官方 WL#11250 的设计与两个坑。
>
> **边界**：本篇**写完两层**——server 侧如何判定 INSTANT 可行性（`check_if_supported_inplace_alter` 与 `IS_EQUAL_*` 三档容忍度、各引擎分歧）、引擎侧如何用行版本号让不同时代的行共存。算法选择策略与 MDL 三段式的公共骨架见 [`ddl.md`](ddl.md)；另两种算法的跨层链路见 [`copy_ddl.md`](copy_ddl.md)、[`inplace_ddl.md`](inplace_ddl.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
  - [主链路：从判定到提交](#主链路从判定到提交)
  - [行版本号机制](#行版本号机制)
  - [提交路径与元数据](#提交路径与元数据)
  - [限制与官方两个"坑"](#限制与官方两个坑)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`ALGORITHM=INSTANT`：**只改元数据，已有行一个字节都不改**。加列/删列时，新列不落盘到老行上，靠"行版本号"让不同时代的行共存——老行定格在旧版本，读取时按版本号推导它实际有哪些列，缺失的列用 DD 里存的默认值填充。

### 用途

把最常见的加列操作从 O(表大小) 降到 O(1)：大表 `ADD COLUMN` 从"几分钟到几小时"变成"秒级"，且不占用额外磁盘、不需要重建表。

### 版本演进

| 版本 | 变化 |
|------|------|
| 8.0.12 | INSTANT 引入，仅支持 **ADD COLUMN**（记录格式 **V1**：记录头存字段数 `n_fields`）。官方 WL#11250 摘要首句 *"Porting Tecent's patch on Instant Add Column"*——源自腾讯贡献的补丁 |
| 8.0.29 | 支持 **DROP COLUMN**，记录格式升级为 **V2**（记录头存 1 字节 `row_version`）；老表升级后仍按 V1 语义解析（`is_upgraded_instant()`） |
| 8.0.29+ | 支持面继续扩大（如列重命名等元数据级操作） |

**★ 版本演进的直接证据是 `Instant_Type` 枚举的取值数**（`dict0inst.h:39`）——8.0.39 共 **5 个值**，不止 ADD/DROP：

```cpp
/** Flags indicating if current operation can be done instantly */
enum class Instant_Type : uint16_t {
  /** Impossible to alter instantly */
  INSTANT_IMPOSSIBLE,          // == 0，也是 handler_trivial_ctx 的"非 INSTANT"哨兵值
  /** Can be instant without any change */
  INSTANT_NO_CHANGE,           // flags 全被 INNOBASE_INPLACE_IGNORE 吃掉
  /** Adding or dropping virtual columns only */
  INSTANT_VIRTUAL_ONLY,        // 8.0.29：只动虚拟列
  /** ADD/DROP COLUMN which can be done instantly, including adding/dropping
  stored column only (or along with adding/dropping virtual columns) */
  INSTANT_ADD_DROP_COLUMN,     // 8.0.12 ADD / 8.0.29 DROP
  /** Column rename */
  INSTANT_COLUMN_RENAME        // 8.0.28+：列改名
};
using Columns = std::vector<Field *>;
```

> ⚠️ **只有 `INSTANT_ADD_DROP_COLUMN` 会 `current_row_version++`**（`dict0inst.cc:248`）——这就是"行版本号只在 instant 加/删列时递增"的源码证据。改名/虚拟列/无变更这三个分支**不消耗行版本号**。

**★ 8.0.29 的隐藏列命名在 8.0.32 又变过一次**（`dict0mem.h:84`）：

```cpp
/** Format of INSTANTLY DROPPED column names. */
constexpr char INSTANT_DROP_SUFFIX_8_0_29[] = "_dropped_v";
constexpr char INSTANT_DROP_PREFIX_8_0_32[] = "!hidden!_dropped_";
```

```cpp
static void set_dropped_column_name(std::string &name, uint32_t version,
                                    uint32_t phy_pos) {
  std::ostringstream new_name;
  new_name << INSTANT_DROP_PREFIX_8_0_32 << "v" << version << "_p" << phy_pos
           << "_" << name;
  name = new_name.str();
  name.resize(std::min<size_t>(name.size(), NAME_CHAR_LEN));  // ★ 超 64 字符硬截断
}
```

⇒ 最终格式 `!hidden!_dropped_v{version}_p{phy_pos}_{原列名}`。⚠️ **硬截断**意味着长列名反复 drop 有撞车风险（撞车报 `ER_IB_HIDDEN_NAME_CONFLICT`）。

---

## 理论基础

### 设计思想与权衡

**要解决的问题**：`ADD COLUMN` 的朴素做法（COPY/INPLACE rebuild）要重建整张表，代价 O(表大小)。但对"只在表尾加一列、且带默认值"这种变更，物理上**没有任何一行需要真的存新值**——它对所有已存在的行都取同一个默认值。

**核心取舍：把"列值"从记录里移出去，改成"按版本推导"**：

- 记录里不存新列的值；
- 表级维护"当前行版本号" `current_row_version`，每次 instant 变更 +1；
- **新插入的行**才带新版本号（才会存新列）；老行版本号定格；
- 读取时用该行的版本号去查"这一版有哪些列"（列的 `v_added` / `v_dropped`），缺失的列取 DD 里的默认值。

**"本来可以那样做，但没有"的对比**：也可以像 COPY 那样把默认值物理写进每一行（语义等价），代价是 O(n) 重写；INSTANT 选择"惰性"——**把迁移成本推迟到每一行下次被 UPDATE 时才发生**，甚至推迟时也不落盘（用 `REC_OFFS_DEFAULT` 标记，见 Misc 的坑一）。

**代价**：

- **读取路径变复杂**：每次解析记录都要先取 `row_version`，再查该版本的列集合与 nullable 数（`get_nullable_fields_for_rec`），是一个常量级但确实存在的开销。
- **记录头多占 1 字节**（V2），或 1–2 字节（V1）。
- **语义约束**：新列只能加到末尾（前面插列会让"按位置推导"失效）；`INSTANT` 不支持压缩表、有全文索引的表、DD 表空间的表等（详见限制）。
- **退化阈值**：`INSTANT` 次数不是无限的——`row_version` 是 1 字节（上限 255 左右），且每次 instant 都让记录解析多一层历史；官方 FR13 指出 **rebuild / TRUNCATE / OPTIMIZE 后记录格式统一为最新、版本归零**。

**★★ 哪些操作能 INSTANT，答案是一个 6-bit 白名单**（`handler0alter.cc:148`）：

```cpp
/** Operation allowed with ALGORITHM=INSTANT */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
    Alter_inplace_info::ALTER_COLUMN_NAME |          // 列改名
    Alter_inplace_info::ADD_VIRTUAL_COLUMN |         // 加虚拟列
    Alter_inplace_info::DROP_VIRTUAL_COLUMN |        // 删虚拟列
    Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER | // 虚拟列换序
    Alter_inplace_info::ADD_STORED_BASE_COLUMN |     // ★ 加实体列
    Alter_inplace_info::ALTER_STORED_COLUMN_ORDER |  // 实体列换序
    Alter_inplace_info::DROP_STORED_COLUMN;          // ★ 删实体列
```

> ★ **`HA_ALTER_FLAGS` 是 64-bit 位图，共 50 个 bit（bit0..bit49）**。白名单外任何一位置位 → 立即 `INSTANT_IMPOSSIBLE`。

**★★ 为什么加主键不能 INSTANT**（这是最常被问的一条）：`ADD_PK_INDEX` 被归入 `INNOBASE_ALTER_REBUILD`，而 **INSTANT 永不 rebuild**——两条规则直接冲突：

```cpp
/** Operations for rebuilding a table in place */
static const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_ALTER_REBUILD =
    Alter_inplace_info::ADD_PK_INDEX | Alter_inplace_info::DROP_PK_INDEX |  // ★
    Alter_inplace_info::CHANGE_CREATE_OPTION | ... | Alter_inplace_info::RECREATE_TABLE;

[[nodiscard]] static bool innobase_need_rebuild(
    const Alter_inplace_info *ha_alter_info) {
  if (is_instant(ha_alter_info)) {
    return (false);            // ★★ INSTANT 一律 return false
  }
  ...
}
```

> ★★ 两条一致性断言把"rebuild 与 instant 互斥"钉死：`ut_ad(ctx == nullptr || !(ctx->need_rebuild() && is_instant(ha_alter_info)))`（`handler0alter.cc:1608`）。

**★★ 压缩表 / 全文索引 / 临时表 / 系统表为什么不行**——全部由 `support_instant_add_drop()` 一句话决定（`dict0dict.ic:1370`）：

```cpp
inline bool dict_table_t::support_instant_add_drop() const {
  return (
      !DICT_TF_GET_ZIP_SSIZE(flags) && space != dict_sys_t::s_dict_space_id &&
      !DICT_TF2_FLAG_IS_SET(this, DICT_TF2_FTS_HAS_DOC_ID) && !is_temporary() &&
      !DICT_TF2_FLAG_IS_SET(this, DICT_TF2_FTS) && !is_system_table);
}
```

| 条件 | 排除 |
|---|---|
| `!DICT_TF_GET_ZIP_SSIZE(flags)` | **压缩表**（`ROW_FORMAT=COMPRESSED` / `KEY_BLOCK_SIZE`） |
| `space != dict_sys_t::s_dict_space_id` | **DD 表空间**里的表 |
| `!DICT_TF2_FTS_HAS_DOC_ID` | **有全文索引**（带隐藏 `FTS_DOC_ID` 列） |
| `!is_temporary()` | 临时表 |
| `!DICT_TF2_FTS` | 带 FTS 索引 |
| `!is_system_table` | 系统表 |

**★ 升级线程一律不走 INSTANT**（`handler0alter.cc:835`）：

```cpp
  /* During upgrade, if columns are added in system tables, avoid instant */
  if (current_thd->is_server_upgrade_thread()) {
    return (Instant_Type::INSTANT_IMPOSSIBLE);
  }
```

**★ INSTANT 不需要 row log**——`inplace_alter_table_impl()` 在创建 `Alter_stage`（row log 的载体）**之前**就 return 了：

```cpp
  if (!(ha_alter_info->handler_flags & INNOBASE_ALTER_DATA) ||
      is_instant(ha_alter_info)) {
    return all_ok();        // ★★ row log 根本没机会被创建
  }
  ...
  ctx->m_stage = ut::new_withkey<Alter_stage>(UT_NEW_THIS_FILE_PSI_KEY, pk);
```

> ⚠️ **`handler0alter.cc` 全文 "row_log" 0 命中**——row log 的实现与调用全在 `row0log.cc`，`handler0alter.cc` 只通过 `ctx->m_stage` 与 `clust_index->online_log` 间接驱动，INSTANT 两条都不触碰。

### 算法与数据结构

| 机制 | 落点 | 为什么这么选 |
|---|---|---|
| 版本化列元数据 | 每列的 `v_added` / `v_dropped` + `se_private_data` | 用"区间"表达列的存在范围，读取时 `v_added ≤ row_version < v_dropped` 即存在，O(1) 判定 |
| `nullables[version]` 数组 | `dict_index_t::create_nullables()` | 不同版本的行可空列数不同，预计算每个版本的 nullable 数避免逐行遍历列 |
| 默认值外置 | DD 的 `DD_INSTANT_COLUMN_DEFAULT` | 老行不存值，读时按列取默认值；这是"零重写"的关键 |
| 1 字节版本号 | 记录头前 `rec-6`（COMPACT/DYNAMIC） | 版本号上限远小于 256，单字节足够；紧贴记录头便于 O(1) 定位 |

---

## 核心实现

### 主链路：从判定到提交

```
ALTER TABLE ... ADD COLUMN
  ├─ server: check_if_supported_inplace_alter()
  │     → HA_ALTER_INPLACE_INSTANT 表示可 INSTANT（判定链条见 ddl.md）
  ├─ server: 仍需走 prepare/commit_inplace_alter_table 两段（MDL 短暂升级）
  └─ 引擎: commit_instant_ddl()（dict0inst.cc）
        ├─ dd_copy_private()            复制私有元数据
        ├─ populate_to_be_instant_columns()
        ├─ commit_instant_drop_col() / commit_instant_add_col()
        ├─ m_dict_table->current_row_version++   ★ 唯一"改表"动作
        └─ 更新 DD 中每列的 se_private_data（v_added / v_dropped / 默认值）
```

**整个过程不扫描任何一行、不分配任何新页**——这就是"秒级"的全部原因。对比 INPLACE rebuild（[`inplace_ddl.md`](inplace_ddl.md)）要扫描-排序-建树，COPY（[`copy_ddl.md`](copy_ddl.md)）要逐行 INSERT。

### 行版本号机制

#### 三个 version，别混淆

提到 instant 的 "version"，其实涉及**三个不同层级**，极易混淆：

| 层级 | 字段 | 类型 | 何时变 | 作用 |
|---|---|---|---|---|
| 表定义 | `dict_table_t::version` | uint64_t | **任何 DDL** | 表定义元数据版本 |
| 行时代 | `dict_table_t::current_row_version` | uint32_t | **仅 instant add/drop** | 行的"时代"计数 |
| 物理行 | 记录头 row_version | uint8_t | 行 INSERT 时定格 | 该行属于哪个时代 |

**`version`（表定义元数据版本，dict0mem.h）**：注释原文 "metadata version number of dd::Table::se_private_data()"。它是**整张表定义**（所有列/索引/选项）的版本号，**任何 DDL（不只 instant）都会 +1**。用途是检测表定义是否被并发修改——dict0dict.cc：

```cpp
/* If the metadata version is bigger than the one in table, it could be that
   an ALTER TABLE has been rolled back, so metadata in new version should be ignored. */
if (table->version != metadata->get_version()) {
  return get_dirty;
}
```

每次 open 表时 `ha_innodb.cc` 从 DD 取最新值：`ib_table->version = dd_get_version(table_def)`。

**`current_row_version`（行版本号，dict0mem.h）**：注释原文 "Current row version in case columns are added/dropped INSTANTly"。**只有 instant add/drop column 才递增**（dict0inst.cc）。它是"行时代"的计数——每发生一次 instant 结构变更，表进入新"时代"。`has_row_versions()` 的定义就是 `current_row_version > 0`。

**行记录头 row_version（1 字节）**：物理行里存的值 = **该行 INSERT 时 `current_row_version` 的快照**。老行定格在旧时代，新行用新时代。

```
dict_table_t::version              （表定义版本：任何 DDL +1）
dict_table_t::current_row_version  （行版本：仅 instant add/drop +1）
        │ 新行 INSERT 时快照
        ▼
物理行.row_version = 当时的 current_row_version
```

**列级还有 `v_added` / `v_dropped`**：每个 `dict_col_t` 记录自己"在哪一版被 instant 加进来 / 删掉"。读取一行时，用该行 `row_version` 与列的 `v_added`/`v_dropped` 比较，判断这列在这行里是否存在（`v_added ≤ row_version < v_dropped` 才存在），不存在的列用默认值填充。`create_nullables`（dict0mem.cc）正是遍历所有列、按每个版本的 v_added/v_dropped 累加出 `nullables[version]` 数组。

> 回答"也涉及 dict 的 version 吧"：是的，但要区分——`dict_table_t::version` 是**表定义元数据版本**（任何 DDL 都变），`current_row_version` 才是**行版本**（仅 instant 变），两者是独立字段。

#### 两种记录格式：V1（n_fields）→ V2（row_version）

| | **V1**（8.0.12–8.0.28） | **V2**（8.0.29+，当前） |
|---|---|---|
| 记录头存什么 | **字段数 n_fields**（`rec_set_n_fields`） | **1 字节 row_version**（`rec_set_instant_row_version_new`） |
| 状态枚举 | `Rec_instant_state::REC_IS_INSTANT` | `Rec_instant_state::REC_IS_VERSIONED` |
| 支撑的 DDL | 仅 ADD COLUMN | ADD + DROP COLUMN |
| 判定依据 | `has_instant_cols()` | `has_row_versions()` |

**为什么从 V1 演进到 V2**：V1 存"本行实际字段数"，只够支撑"末尾加列"（新列不落老行，老行字段数少）。要支持 **DROP COLUMN**（被删的列还要保留在老行里）和更复杂的版本语义，字段数就不够表达了，于是引入 **row_version**——版本号能承载"这行是哪个时代插入的"，据此推导出该行完整列集合。

**★★ 8.0.39 实际是"6 种行"而非 3 种**——源码里有一张官方状态表（`rem/rec.h:724`，引用淘宝月报）：

```cpp
/* For INSTANT ADD/DROP, we may have following 5 types of rec for table :
  http://mysql.taobao.org/monthly/2023/12/02/
  +----------------------------------------------------------------------------+
  |              SCENARIO                         |        STATE               |
  |----------------------------------+------------+---------+------------------|
  |            Row property          | V <= 8.0.28| Bit set | Stored on row    |
  +----------------------------------+------------+---------+------------------+
  | INSERTED before 1st INSTANT ADD  |     Y      | NONE    | N/A              |
  |----------------------------------+------------+---------+------------------|
  | INSERTED after 1st INSTANT ADD   |     Y      | INSTANT | # of fields      |
  |----------------------------------+------------+---------+------------------|
  | INSERTED before INSTANT ADD/DROP |     Y      | VERSION | Version = 0      |
  |----------------------------------+------------+---------+------------------|
  | INSERTED before INSTANT ADD/DROP |     N      | NONE    | N/A              |
  |----------------------------------+------------+---------+------------------|
  | INSERTED after INSTANT ADD/DROP  |     Y/N    | VERSION | row version      |
  +----------------------------------------------------------------------------+
*/
enum REC_INSERT_STATE {
  INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION,
  INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION,
  INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION,
  INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION,
  INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION,
  INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION,
  NONE
};
```

> ★★ **最反直觉的一格**：第三行"升级后、新实现首次 instant 之前插入的行"——**VERSION 位置 1 但 version = 0**。所以"VERSION 位为 1"不等于"做过 V2 instant"，还必须看 version 是否 > 0。
> ⚠️ 第五行（升级表，bit = NONE）与第一行（老实现前，bit = NONE）**在 info bits 上无法区分**，只能靠 `index->table->has_instant_cols()` 判断。
> ⚠️ 表里说的是 "5 types"（注释原文），但枚举有 **6 个值**——多出来的是 `INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION`（从没做过 instant 的表，短路返回）与哨兵 `NONE`。

**★★ 六种行各自的"解析配方"**（★ 这才是这张表的真正内容——同一个函数对六种行走出六条不同路径）：

| 状态 | `row_version` | 行内真实列数 | NULL 位图大小 `n_null` | 逐列判定 |
|---|---|---|---|---|
| `INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION` | `UINT8_UNDEFINED` | 全部 | `index->n_nullable` | 无，全列都在 |
| `INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION`（V1 老行） | `UINT8_UNDEFINED` | `index->get_instant_fields()` | `index->get_nullable_before_instant_add_drop()` | `i >= non_default_fields` → 打 DEFAULT |
| `INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION`（V1 新行） | `UINT8_UNDEFINED` | 行内 `n_fields`（1–2 字节解码） | `index->calculate_n_instant_nullable(n_fields)` | `i >= non_default_fields` → 打 DEFAULT |
| `INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION`（V2 老行） | 0 | 全部（按 v0） | `index->get_nullable_before_instant_add_drop()` | `is_added_after(0)` / `is_dropped_in_or_before(0)` |
| `INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION` | 0 | 全部 | `get_nullable_in_version(0)` | 同上（version 强制 0） |
| `INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION`（V2 新行） | `> 0` | 按版本 | `get_nullable_in_version(v)` | `is_added_after(v)` / `is_dropped_in_or_before(v)` |

> ★★ **V1 与 V2 的本质差别就在这里**：V1 只能表达"**尾部**少了若干列"（`i >= non_default_fields` 一个线性判据）；V2 能表达"**任意位置**缺列 + 任意位置有多余的已删列"（两个逐列判据）。这就是为什么 V1 撑不起 DROP COLUMN。

**★★ 解析的总入口 `rec_init_null_and_len_comp()` 全文**（`rem/rec.h:985`，★ 上面那张表的代码实现）：

```cpp
static inline enum REC_INSERT_STATE rec_init_null_and_len_comp(
    const rec_t *rec, const dict_index_t *index, const byte **nulls,
    const byte **lens, uint16_t *n_null, uint16_t &non_default_fields,
    uint8_t &row_version) {
  non_default_fields = static_cast<uint16_t>(dict_index_get_n_fields(index));
  row_version = UINT8_UNDEFINED;

  /* Position nulls */
  *nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);      /* ★ 先落在 rec-6（info bits 后一字节） */

  const enum REC_INSERT_STATE rec_insert_state = get_rec_insert_state(index, rec, false);

  switch (rec_insert_state) {
    case INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION: {
      ut_ad(!rec_get_instant_flag_new(rec));
      ut_ad(!rec_new_is_versioned(rec));
      *n_null = index->n_nullable;                 /* ① 无 instant：全量 */
    } break;

    case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION:
    case INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION: {
      /* Read the row version from record */
      row_version = (uint8_t)(**nulls);            /* ★ 读 1 字节版本号 */
      ut_ad(is_valid_row_version(row_version));
      /* Reposition nulls to skip version */
      *nulls -= 1;
      *n_null = index->get_nullable_in_version(row_version);   /* ② V2：查 nullables[] */
    } break;

    case INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION: {
      uint16_t length;
      non_default_fields =
          rec_get_n_fields_instant(rec, REC_N_NEW_EXTRA_BYTES, &length);
      ut_ad(length == 1 || length == 2);
      /* Reposition nulls to skip number of fields */
      *nulls -= length;                            /* ★ 退 1 或 2 字节 */
      *n_null = index->calculate_n_instant_nullable(non_default_fields);
    } break;                                       /* ③ V1 新行：按行内列数反推 */

    case INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION: {
      *n_null = index->get_nullable_before_instant_add_drop();
      non_default_fields = index->get_instant_fields();
    } break;                                       /* ④ V1 老行：不额外占字节 */

    case INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION: {
      *n_null = index->get_nullable_before_instant_add_drop();
    } break;                                       /* ⑤ V2 老行：按 v0 解析 */
    default:
      ut_ad(false);
  }

  /* Position lens */
  *lens = *nulls - UT_BITS_IN_BYTES(*n_null);     /* ★ 变长长度列表在 NULL 位图更低地址 */

  return (rec_insert_state);
}
```

> ★★ **`*nulls` 的初始位置 `rec - 6` 是理解全篇的钥匙**——它既是 V1 的"n_fields 低位字节"，也是 V2 的"row_version 字节"。六种行的差别，本质上就是"从这个位置往低地址再退几个字节、以及 NULL 位图该开多大"。
> ★ `*lens = *nulls - UT_BITS_IN_BYTES(*n_null)`：变长长度列表紧邻 NULL 位图的**低地址侧**，且**倒序**存放（读取用 `*lens--`）。

**★★ 逐列判定：`rec_init_offsets_comp_ordinary()` 的状态机**（`rem/rec.cc:535`，★ 六种行在这里真正分流）：

```cpp
  do {
    /* Fields are stored on disk in version they are added in and are
    maintained in fields_array in the same order. Get the right field. */
    const dict_field_t *field = index->get_physical_field(i);   /* ★ 物理序 */
    const dict_col_t *col = field->col;
    uint64_t len;

    switch (rec_insert_state) {
      case INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION:
        ut_ad(!index->has_instant_cols_or_row_versions());
        break;

      case INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION: {
        /* Record has to be interpreted in v0. */
        row_version = 0;
      }
        [[fallthrough]];
      case INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION:
      case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION: {
        if (col->is_dropped_in_or_before(row_version)) {
          /* This columns is dropped before or on this row version so its data
          won't be there on row. So no need to store the length. Instead, store
          offs ORed with REC_OFFS_DROP to indicate the same. */
          len = offs | REC_OFFS_DROP;
          goto resolved;
          /* NOTE : Existing rows, which have data for this column, would still
          need to process this column, so don't skip and store the correct
          length there. Though it will be skipped while fetching row. */
        } else if (col->is_added_after(row_version)) {
          /* This columns is added after this row version. In this case no need
          to store the length. Instead store only if it is NULL or DEFAULT value. */
          len = rec_get_instant_offset(index, i, offs);
          goto resolved;
        }
      } break;

      case INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION:
      case INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION: {
        ut_ad(!is_valid_row_version(row_version));
        if (i >= non_default_fields) {
          /* This would be the case when column doesn't exists in the row. */
          len = rec_get_instant_offset(index, i, offs);
          goto resolved;
        }
        /* Note : Even if the column has been dropped, this row in V1 would
        definitely have the value of this column. */
      } break;
      default:
        ut_ad(false);
    }
    ... 正常的 NULL 位图 / 变长长度读取 ...
  resolved:
    rec_offs_base(offsets)[i + 1] = len;
  } while (++i < rec_offs_n_fields(offsets));

  *rec_offs_base(offsets) = (rec - (lens + 1)) | REC_OFFS_COMPACT | any_ext;
```

> ★★★ **顺序不可交换**：先判 `is_dropped_in_or_before`（打 `REC_OFFS_DROP`），再判 `is_added_after`（打 `REC_OFFS_DEFAULT` / `REC_OFFS_SQL_NULL`），最后才读 NULL 位图与变长长度。**被跳过的列不消耗 `nulls` / `lens`**——这正是老行字节流能正确对齐的原因。
> ★★ **`is_dropped_in_or_before` 的注释里有个容易被忽略的事实**：已 DROP 的列对**老行**来说数据还在行上，所以"仍然要处理这一列、存正确的长度"——只是**取数时**跳过。这就是 `dropped_col_count` 在 REDUNDANT 里要单独修正下标的理由。
> ★ 循环变量 `i` 是**物理位置**（`get_physical_field(i)`），不是逻辑列序。V2 下 `get_physical_field()` 内部要经 `fields_array` 做一次翻译。
> ★ 最后一行 `rec - (lens + 1)` 反推出整个 extra 区的大小（变长长度 + NULL 位图 + version/n-fields + 5 字节头）。

**★ 状态判定总入口 `get_rec_insert_state()`**（`rem/rec.h:763`）：

```cpp
static inline enum REC_INSERT_STATE get_rec_insert_state(
    const dict_index_t *index, const rec_t *rec, bool temp) {
  if (!index->has_instant_cols_or_row_versions()) {
    return INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION;
  }
  /* Position just before info-bits where version will be there if any */
  const byte *v_ptr = (byte *)rec -
      ((temp ? REC_N_TMP_EXTRA_BYTES : REC_N_NEW_EXTRA_BYTES) + 1);
  const bool is_versioned =
      (temp) ? rec_new_temp_is_versioned(rec) : rec_new_is_versioned(rec);
  const uint8_t version = (is_versioned) ? (uint8_t)(*v_ptr) : UINT8_UNDEFINED;
  const bool is_instant = (temp) ? rec_get_instant_flag_new_temp(rec)
                                 : rec_get_instant_flag_new(rec);

  if (is_versioned && is_instant) {      // ★ 两位同时置 1 = 表损坏
    ib::error() << "Record has both instant and version bit set in Table '"
                << index->table_name << "', Index '" << index->name()
                << "'. This indicates that the table may be corrupt. Please "
                   "run CHECK TABLE before proceeding.";
  }
  ut_ad(!is_versioned || !is_instant);

  if (is_versioned) {
    if (version == 0) {                  // ★ 升级表
      ut_ad(index->has_instant_cols());
      rec_insert_state = INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION;
    } else {
      ut_ad(index->has_row_versions());
      rec_insert_state = INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION;
    }
  } else if (is_instant) {
    rec_insert_state = INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION;   // V1
  } else if (index->table->has_instant_cols()) {
    rec_insert_state = INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION;
  } else {
    rec_insert_state = INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION;
  }
```

**⚠️ 三个函数真名核查**：**不存在 `rec_new_is_instant()`**（真名 `rec_get_instant_flag_new()`，`rem/rec.h:631`）；**不存在 `rec_init_null_and_len_old()`**（REDUNDANT 走 `rec_init_offset_old_1byte()` / `_2byte()`）；**不存在 `rec_offs_any_default()`**（只有 `rec_offs_nth_default_low()` + `rec_offs_nth_default()`）。

**★ 常量分散在两个文件**（这是 8.0.39 的目录重排结果）：

```cpp
// rem/rec.h:114 —— info bits 两位
constexpr uint32_t REC_INFO_VERSION_FLAG = 0x40UL;   // VERSION 位
constexpr uint32_t REC_INFO_INSTANT_FLAG = 0x80UL;   // INSTANT 位（V1）

// rem/rec.h:128 —— 记录头字节数
constexpr uint32_t REC_N_OLD_EXTRA_BYTES = 6;        // REDUNDANT
constexpr int32_t  REC_N_NEW_EXTRA_BYTES = 5;        // COMPACT/DYNAMIC
constexpr uint32_t REC_N_TMP_EXTRA_BYTES = 1;        // TEMP（row log / sort buffer）

// rem/rec.h:209 —— n_fields 编码
static const uint8_t REC_N_FIELDS_TWO_BYTES_FLAG = 0x80;
static const uint8_t REC_N_FIELDS_ONE_BYTE_MAX = 0x7F;

// rem0types.h:44 —— 上限
constexpr uint32_t REC_MAX_N_FIELDS = 1024 - 1;
constexpr uint32_t REC_MAX_N_USER_FIELDS = REC_MAX_N_FIELDS - DATA_N_SYS_COLS * 2;
```

> ⚠️ **`include/rec0rec.h` 不存在**（全仓 0 匹配）。8.0.39 已重排：新增 `storage/innobase/rem/rec.h`（41.9KB）与 `rem/rec.cc`，绝大多数 instant 定义从 `rem0rec.h/.ic` 搬到了这里。
> ⚠️ **`REC_OFFS_COMPACT` 与 `REC_OFFS_SQL_NULL` 同为 `1U << 31`**（`rec.h:45/47`）——靠"只出现在 `offsets[0]` 上"来区分。`REC_OFFS_DEFAULT = 1 << 29`，`REC_OFFS_DROP = 1 << 28`（后者是 instant DROP 专用）。

#### 字节级布局（COMPACT / DYNAMIC 行格式）

记录头固定 5 字节（`REC_N_NEW_EXTRA_BYTES = 5`，rec.h），`rec` 指针指向记录头起点。instant 字段插在 **NULL 位图与记录头之间**，紧贴记录头（`rec - 6`）：

```
① 普通记录（无 instant）
低地址 ───────────────────────────────────────────────────────▶ 高地址
┌──────────────────┬──────────┬─────────────────┬────────────┐
│ 变长字段长度列表    │ NULL位图  │ 记录头 (5B)       │ 列数据       │
└──────────────────┴──────────┴─────────────────┴────────────┘
                                ↑ rec            ↑ rec+5

② V1（instant ADD COLUMN，8.0.12–8.0.28）：记录头前加 n_fields（1–2 字节）
┌──────────────────┬──────────┬─────────────────┬─────────────────┬────────────┐
│ 变长字段长度列表    │ NULL位图  │ n_fields (1–2B)  │ 记录头 (5B)       │ 列数据       │
└──────────────────┴──────────┴─────────────────┴─────────────────┴────────────┘
                                ↑ rec-6 / rec-7   ↑ rec
  n_fields ≤ 127（REC_N_FIELDS_ONE_BYTE_MAX=0x7F）：1 字节
  n_fields > 127：2 字节，高字节最高位 0x80 作"两字节"标志（REC_N_FIELDS_TWO_BYTES_FLAG）

③ V2（instant ADD/DROP COLUMN，8.0.29+）：记录头前加 row_version（固定 1 字节）
┌──────────────────┬──────────┬──────────────┬─────────────────┬────────────┐
│ 变长字段长度列表    │ NULL位图  │ row_version  │ 记录头 (5B)       │ 列数据       │
│                  │          │    (1B)      │                 │            │
└──────────────────┴──────────┴──────────────┴─────────────────┴────────────┘
                                ↑ rec-6       ↑ rec
```

三态靠记录头 info bits 里的**两个独立标志位**区分（rem0rec.ic）：

```
                    INSTANT_FLAG   VERSION_FLAG
普通（REC_IS_SIMPLE）      0              0
V1（REC_IS_INSTANT）       1              0
V2（REC_IS_VERSIONED）     0              1
```

两者写在**同一位置**（记录头前 1 字节）：

```cpp
byte *ptr = rec - (REC_N_NEW_EXTRA_BYTES + 1);   // = rec - 6，两种格式写同一位置
```

---

#### ★★ 读侧表示：`offsets[]` 数组与四个标记位

物理记录里**没有任何"default 标记"**——instant 的全部语义活在**内存**的 `offsets[]` 数组里。这是"instant 格式"最容易被误解的一点：

```cpp
/* rem/rec.h:44 —— ★ 全部 REC_OFFS_* 都在这里（不在 rem0rec.h，那里 0 命中） */
/* Compact flag ORed to the extra size returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_COMPACT  = 1U << 31;
/* SQL NULL flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_SQL_NULL = 1U << 31;
/* External flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_EXTERNAL = 1 << 30;
/* Default value flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_DEFAULT  = 1 << 29;
/* An INSTANT DROP flag in offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_DROP     = 1 << 28;
/* Mask for offsets returned by rec_get_offsets() */
constexpr uint32_t REC_OFFS_MASK     = REC_OFFS_DROP - 1;
```

> ★★ **`REC_OFFS_COMPACT` 与 `REC_OFFS_SQL_NULL` 同为 `1<<31` 却不冲突**——前者只出现在 `base[0]`（extra_size 槽）上，后者只出现在 `base[1..n]`（各列结束偏移槽）上。
> ★ `REC_OFFS_MASK = 0x0FFFFFFF`——剥掉高 4 个标记位后才是纯偏移值。

**offsets 数组的完整布局**：

```
offsets[0]            = n_alloc（数组容量）
offsets[1]            = n_fields（列数）
offsets[2] = (ulint)rec    ┐ 仅 UNIV_DEBUG
offsets[3] = (ulint)index  ┘ （rec_offs_make_valid 填充）
── rec_offs_base(offsets) = offsets + REC_OFFS_HEADER_SIZE ──
base[0]     = extra_size | REC_OFFS_COMPACT | [REC_OFFS_EXTERNAL]
base[1..n]  = 第 0..n-1 列的【结束偏移】| 标记位
```

```cpp
/* rem/rec.h:173 —— ★★ REC_OFFS_HEADER_SIZE 分编译模式！ */
#ifdef UNIV_DEBUG
constexpr uint32_t REC_OFFS_HEADER_SIZE = 4;    /* debug：多 2 槽存 rec/index 指针 */
#else
constexpr uint32_t REC_OFFS_HEADER_SIZE = 2;
#endif
constexpr uint32_t REC_OFFS_NORMAL_SIZE = 100;  /* 栈上初试容量 */
constexpr uint32_t REC_OFFS_SMALL_SIZE  = 10;
```

**★★ `REC_OFFS_DEFAULT` 的唯一生产者**（★ 写的是内存 offsets，不是磁盘）：

```cpp
/* rem/rec.h:496 */
static inline uint64_t rec_get_instant_offset(const dict_index_t *index,
                                              ulint n, uint64_t offs) {
  ut_ad(index->has_instant_cols_or_row_versions());
  ulint length;
  index->get_nth_default(n, &length);      /* ★ 先看默认值的长度 */

  if (length == UNIV_SQL_NULL) {
    return (offs | REC_OFFS_SQL_NULL);     /* ★ 默认值是 NULL → 打 SQL_NULL */
  } else {
    return (offs | REC_OFFS_DEFAULT);      /* ★ 默认值非 NULL → 打 DEFAULT */
  }
}
```

> ★★★ **这是"默认值 vs NULL"的分岔点**：instant 列的默认值**本身是 NULL** 时打 `REC_OFFS_SQL_NULL`（不是 `REC_OFFS_DEFAULT`）。所以"这列是 instant 缺列"≠"这列带 DEFAULT 标记"——**两者都可能**。
> ★ 两种情况都**不推进 `offs`**——该列在行上不占任何字节。

**★★ 标记位 → 长度语义的翻译（唯一翻译点）**：

```cpp
/* include/rem0lrec.h:141 */
static inline ulint rec_get_nth_field_offs_low(const ulint *offsets, ulint n,
                                               ulint *len) {
  if (n == 0) offs = 0;
  else        offs = rec_offs_base(offsets)[n] & REC_OFFS_MASK;

  length = rec_offs_base(offsets)[1 + n];

  if (length & REC_OFFS_SQL_NULL)      length = UNIV_SQL_NULL;
  else if (length & REC_OFFS_DEFAULT)  length = UNIV_SQL_ADD_COL_DEFAULT;
  else if (length & REC_OFFS_DROP)     length = UNIV_SQL_INSTANT_DROP_COL;
  else { length &= REC_OFFS_MASK; length -= offs; }    /* ★ 相邻两槽相减得长度 */

  *len = length;
  return (offs);
}
```

```cpp
/* include/univ.i:460 —— 三个哨兵值 */
constexpr uint32_t UNIV_SQL_NULL             = UINT32_UNDEFINED;                  // 0xFFFFFFFF
constexpr auto     UNIV_SQL_ADD_COL_DEFAULT  = UNIV_SQL_NULL - 1;                 // 0xFFFFFFFE
constexpr auto     UNIV_NO_INDEX_VALUE       = UNIV_SQL_ADD_COL_DEFAULT - 1;      // 0xFFFFFFFD
const uint32_t     UNIV_MULTI_VALUE_ARRAY_MARKER = UNIV_NO_INDEX_VALUE - 1;       // 0xFFFFFFFC
constexpr auto     UNIV_SQL_INSTANT_DROP_COL = UNIV_MULTI_VALUE_ARRAY_MARKER - 1; // 0xFFFFFFFB
```

> ★ 检查顺序固定：**SQL_NULL → DEFAULT → DROP**。所以 `rec_offs_nth_default()` 返回真时，`rec_offs_nth_sql_null()` 必为假。
> ★ 因为 DEFAULT/DROP 列不推进偏移，它们的 `rec_offs_nth_size()` 恒为 **0**。

**★★ 默认值从哪来（零拷贝）**：

```cpp
/* include/rem0rec.ic:1006 —— ★ 读侧取得列值的最终落点 */
static inline const byte *rec_get_nth_field_instant(const rec_t *rec,
                                                    const ulint *offsets,
                                                    ulint n,
                                                    const dict_index_t *index,
                                                    ulint *len) {
  if (index->has_row_versions()) {
    /* Fields are stored in record in order of the version they are added in. */
    n = index->get_field_off_pos(n);          /* ★ 逻辑列号 → 物理位 */
  }
  ulint off = rec_get_nth_field_offs(nullptr, offsets, n, len);

  if (*len != UNIV_SQL_ADD_COL_DEFAULT && *len != UNIV_SQL_INSTANT_DROP_COL) {
    return (rec + off);                       /* ① 正常：返回行内指针 */
  }
  if (*len == UNIV_SQL_INSTANT_DROP_COL) {
    *len = 0;
    return nullptr;                           /* ② 已删列：返回空 */
  }
  return (index->get_nth_default(n, len));    /* ★★ ③ 缺列：直接返回 dict_col_t::instant_default->value */
}
```

> ★★★ **"补默认值"是零拷贝的**：返回的指针直接指向 `dict_col_t::instant_default->value`（常驻 `table->heap`），**没有任何 memcpy**。所以整个 `row_sel_store_mysql_field()` 里**看不到**针对 DEFAULT 的分支——`row_sel_field_store_in_mysql_format()` 无差别地把它当普通数据写进 MySQL 行。

**★ `rec_get_offsets()` 的反直觉点**（★ 这是"instant 行看起来有满列"的根因）：

```cpp
/* rem/rec.cc:364 */
  if (dict_table_is_comp(index->table)) {
    switch (UNIV_EXPECT(rec_get_status(rec), REC_STATUS_ORDINARY)) {
      case REC_STATUS_ORDINARY:
        n = dict_index_get_n_fields(index);      /* ★★ COMPACT：恒等于索引当前列数！ */
        break;
      ...
  } else {
    n = rec_get_n_fields_old(rec, index);        /* REDUNDANT：读行内 n_fields 并上抬 */
  }
  ...
  ulint size = n + (1 + REC_OFFS_HEADER_SIZE);  /* +1 是 base[0] 的 extra_size 槽 */
  rec_offs_set_n_fields(offsets, n);
  rec_init_offsets(rec, index, offsets);
```

> ★★★ **`rec_get_offsets()` 返回的 `n_fields` 与行内实际列数无关**，永远是当前索引的列数。缺的列靠 `REC_OFFS_DEFAULT` / `REC_OFFS_DROP` 标记"虚拟补齐"——**上层代码完全不需要知道这行是老行还是新行**。这就是 instant DDL 能做到"零数据重写"的设计核心。

**★★ `rec_validate()` 里的一条硬不变式**：

```cpp
      default:
        /* Normal field with non null value */
        /* INSTANT ADD columns will come (physically) after original ones */
        ut_a(n_defaults == 0);        /* ★★ 出现第一个 DEFAULT 列后，不许再有真实数据列 */
        ...
  ut_a(rec_offs_comp(offsets) ||
       (n_fields - n_inst_dropped) <= rec_get_n_fields_old_raw(rec) + n_defaults);
  if (len_sum != rec_offs_data_size(offsets)) { ... return false; }
```

> ★★ `ut_a(n_defaults == 0)` 断言的是"**instant ADD 的列物理上一定排在尾部**"——一旦某个列被打成 DEFAULT，后面就不能再有真实数据列了（对 V1/V2 都成立，因为 ADD 只能追加）。
> ★ 末尾还校验 `len_sum == rec_offs_data_size(offsets)`，即所有真实数据列长度之和必须精确等于数据区大小。

#### 核心函数逐行解析

**写 V1 的 `rec_set_n_fields`**（rem0rec.ic）：

```cpp
static inline uint8_t rec_set_n_fields(rec_t *rec, ulint n_fields) {
  byte *ptr = rec - (REC_N_NEW_EXTRA_BYTES + 1);   // ① 定位：记录头(5B)前 1 字节 = rec-6
  ut_ad(n_fields < REC_MAX_N_FIELDS);               // ② 断言：字段数 < 1023（REC_MAX_N_FIELDS=1024-1）

  if (n_fields <= REC_N_FIELDS_ONE_BYTE_MAX) {      // ③ 单字节路径：≤127（0x7F）
    *ptr = static_cast<byte>(n_fields);             //    直接写，最高位恒 0，天然区分
    return (1);                                     //    报告占用 1 字节
  }

  --ptr;                                            // ④ 双字节路径：>127，指针再前移
  *ptr++ = static_cast<byte>(n_fields & 0xFF);      //    先写低字节（小端）
  *ptr   = static_cast<byte>(n_fields >> 8);        //    再写高字节（暂不含标志位）
  ut_ad((*ptr & 0x80) == 0);                        // ⑤ 断言高字节最高位空闲（≤1023 保证）
  *ptr |= REC_N_FIELDS_TWO_BYTES_FLAG;              // ⑥ 最高位置 0x80 = "这是两字节"标志
  return (2);                                       //    报告占用 2 字节
}
```

逐行要点：① `rec-6` 是记录头前紧贴的 1 字节，V1/V2 共用这个位置；③ 单字节靠"最高位为 0"区分；④-⑥ 双字节时**低字节在前**（小端），高字节最高位 `0x80` 作"两字节"标志供读取识别。

**写 V2 的 `rec_set_instant_row_version_new`**（rem0rec.ic）：

```cpp
static inline void rec_set_instant_row_version_new(rec_t *rec, uint8_t row_version) {
  ut_ad(is_valid_row_version(row_version));          // ① 断言版本号合法
  byte *ptr = rec - (REC_N_NEW_EXTRA_BYTES + 1);     // ② 与 n_fields 同一位置 rec-6
  *ptr = static_cast<byte>(row_version);             // ③ 固定 1 字节，直接写
}
```

对比可见 V2 比 V1 简单得多：**固定 1 字节，无变长判断**——因为 row_version 上限远小于 256，单字节够用。

**"上限"具体是多少？往下拆 `is_valid_row_version`**（这句断言里藏着 INSTANT 的硬约束）：

```cpp
// dict0mem.h
const uint8_t MAX_ROW_VERSION = 64;

static inline bool is_valid_row_version(const uint8_t version) {
  /* NOTE : 0 is also a valid row versions for rows which are inserted after
  upgrading from earlier INSTANT implemenation */
  if (version <= MAX_ROW_VERSION) {
    return true;
  }

  return false;
}
```

★ **`MAX_ROW_VERSION = 64`**——即一张表最多积累 **64 个行版本**。

⚠️ **这个函数有个同名近亲，别搞混**：`dict0dd.cc` 里还有一个 `dd_is_valid_row_version(uint32_t)`，判定是 `version != UINT32_UNDEFINED && version > 0 && version <= MAX_ROW_VERSION`。两者**对 0 的判定恰好相反**：

| 函数 | 参数 | 0 是否合法 | 用在哪 |
|---|---|---|---|
| `is_valid_row_version` | `uint8_t`（**行头里的版本**） | **合法**（源码注释：从早期 INSTANT 实现升级后插入的行，版本就是 0） | 行内版本号判定、`rec_set_instant_row_version_new` 的断言 |
| `dd_is_valid_row_version` | `uint32_t`（**DD 里的列版本**） | **不合法** | DD 元数据 `v_added` / `v_dropped` 的断言（`ut_ad(dd_is_valid_row_version(...))`） |

⇒ 差异来自作用域不同：**行里的版本号允许为 0**（那是升级前就存在的老行，本就没有版本概念），而 **DD 里显式记录的"第几版加/删"必须 > 0**（0 留给"非 instant 列"这个语义）。读代码时看到两个 `is_valid_row_version` 一定要先看参数类型。

**超过 64 会怎样**——判定点在 `check_if_supported_inplace_alter`（`handler0alter.cc`），两个上限并列检查：

```cpp
      } else if (!((m_prebuilt->table->n_def +
                    get_num_cols_added(ha_alter_info)) < REC_MAX_N_FIELDS)) {
        if (requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) {
          my_error(ER_INNODB_INSTANT_ADD_NOT_SUPPORTED_MAX_FIELDS, MYF(0),
                   m_prebuilt->table->name.m_name);
          return HA_ALTER_ERROR;
        }
        /* INSTANT can't be done any more. Fall back to INPLACE. */
        break;
      } else if (!is_valid_row_version(m_prebuilt->table->current_row_version +
                                       1)) {
        ut_ad(is_valid_row_version(m_prebuilt->table->current_row_version));
        if (requested_algorithm == Alter_info::ALTER_TABLE_ALGORITHM_INSTANT) {
          my_error(ER_INNODB_MAX_ROW_VERSION, MYF(0),
                   m_prebuilt->table->name.m_name);
          return HA_ALTER_ERROR;
        }
        /* INSTANT can't be done any more. Fall back to INPLACE. */
        break;
      }
```

两个上限是**独立的两个门槛**，命中任意一个都不能再 INSTANT：

| 上限 | 常量 | 报错（显式 `ALGORITHM=INSTANT`） |
|---|---|---|
| 字段数 | `REC_MAX_N_FIELDS` | `ER_INNODB_INSTANT_ADD_NOT_SUPPORTED_MAX_FIELDS` |
| 行版本号 | `MAX_ROW_VERSION = 64` | `ER_INNODB_MAX_ROW_VERSION` |

三点值得记：

1. **判定用的是 `current_row_version + 1`**——是**预判**"做完这次还能不能合法"，而不是先做再检查。所以最后一次成功的 instant 会让版本号正好停在 64。
2. ★ **两条分支都是同一个模式**：用户显式写了 `ALGORITHM=INSTANT` → **报错**（`HA_ALTER_ERROR`）；没写 → **静默降级 INPLACE**（`break` 出 switch）。这正是 ALGORITHM 子句"声明式约束"语义的实例（见 [`copy_ddl.md`](copy_ddl.md)）。
3. `ut_ad(is_valid_row_version(current_row_version))` 断言**当前版本一定合法**——能走到这个分支说明当前是 64（合法），+1 后 65 越界。⇒ 也就是说，**累计 64 次 instant add/drop 之后，第 65 次必须走 rebuild**（INPLACE/COPY），届时表被重写、版本号归零。这就是"INSTANT 用多了要重建表"的源码依据。

**读取侧的 `rec_get_n_fields_instant`**（rem0rec.ic）：

```cpp
ptr = rec - (extra_bytes + 1);                        // ① 同一位置 rec-6
if ((*ptr & REC_N_FIELDS_TWO_BYTES_FLAG) == 0) {      // ② 最高位 0 → 单字节
  *length = 1;
  return (*ptr);                                      //    直接返回字段数
}
*length = 2;                                          // ③ 最高位 1 → 两字节
n_fields = ((*ptr-- & REC_N_FIELDS_ONE_BYTE_MAX) << 8);  // ④ 高字节去掉 0x80 标志
n_fields |= *ptr;                                     // ⑤ 拼上低字节
```

写入和读取互为镜像：写时"低字节前 + 高字节最高位做标志"，读时"最高位判单双 + 去标志 + 拼低字节"。

#### REDUNDANT（旧格式）与四种行格式小结

上面画的都是 **COMPACT / DYNAMIC（NEW-style）**。REDUNDANT（OLD-style）同样支持 instant version，但布局不同：

- 记录头 `REC_N_OLD_EXTRA_BYTES = 6`，比新格式多 1 字节；
- version 写在 `rec - (6+1) = rec - 7`（`rec_set_instant_row_version_old`）；
- **字段寻址是"偏移数组"而非"变长长度列表"**：读取字段 N 时，若该行 versioned 需额外跳过 1 字节：

```cpp
uint32_t version_length = 0;
if (rec_old_is_versioned(rec)) {
  version_length = 1;                                  // version 占 1 字节
}
return mach_read_from_1(rec - (REC_N_OLD_EXTRA_BYTES + version_length + n));  // rec-7-n
```

布局（REDUNDANT + version）：

```
低地址 ──────────────────────────────────────────────────────▶ 高地址
┌───────────────┬──────────────┬─────────────────┬────────────┐
│ 字段偏移数组    │ row_version  │ 记录头 (6B)       │ 列数据       │
│  (变长)       │   (1B)       │                 │            │
└───────────────┴──────────────┴─────────────────┴────────────┘
                                ↑ rec            ↑ rec+6
```

写入端在 `rec_convert_dtuple_to_rec_old`（rem0rec.cc）：

```cpp
if (store_version) {
  rec_set_instant_row_version_old(rec, index->table->current_row_version);  // 写 rec-7
  rec_old_set_versioned(rec, true);   // 旧格式的 versioned 标志位
} else {
  rec_old_set_versioned(rec, false);
}
```

| 格式 | 记录头大小 | version 位置 | 字段寻址 |
|---|---|---|---|
| REDUNDANT（OLD） | 6B | rec-7 | 偏移数组 |
| COMPACT（NEW） | 5B | rec-6 | 变长长度列表 |
| DYNAMIC（NEW） | 5B | rec-6 | 变长长度列表（+ off-page 外部存储） |
| TEMP（临时/row log） | 1B | rec-2 | 省略记录头 |

TEMP 格式（`REC_N_TMP_EXTRA_BYTES=1`）用于 row log / sort buffer，只保留 info bits 1 字节，instant n_fields 存在 `rec - 2`。

#### 记录状态与读写判定

`rec_convert_dtuple_to_rec_comp` 返回三种状态（rem0rec.cc）：

```cpp
enum class Rec_instant_state { REC_IS_SIMPLE, REC_IS_VERSIONED, REC_IS_INSTANT };

switch (rec_state) {
  case Rec_instant_state::REC_IS_SIMPLE:    rec_new_reset_instant_version(rec); break;  // 无 instant
  case Rec_instant_state::REC_IS_VERSIONED: rec_new_set_versioned(rec);        break;  // V2：存版本号
  case Rec_instant_state::REC_IS_INSTANT:   rec_new_set_instant(rec);          break;  // V1：存字段数
}
```

**★★ `is_store_version()` 的真身**（⚠️ 上面那版是 8.0.29 的旧实现。8.0.39 多了 **UPDATE 未物化默认值**这个分支——这是本篇"坑一"的机制来源）：

```cpp
/* rem/rem0rec.cc:246 */
bool is_store_version(const dict_index_t *index, size_t n_tuple_fields) {
  if (!index->has_instant_cols_or_row_versions()) return false;

  ut_ad(index->table->has_row_versions() ||
        index->table->is_upgraded_instant());
  ut_ad(n_tuple_fields > 0);

  /* From an UPDATE when not materializing INSTANT defaults */
  if (n_tuple_fields < index->n_fields) {          /* ★★ dtuple 列数 < 索引列数 */
    ut_ad(index->has_instant_cols());
    return false;                                  /* → 退回 V1，写 n_fields */
  }
  return true;                                     /* → 走 V2，写 row_version */
}
```

完整的写入判定链：

```
index->has_instant_cols_or_row_versions()
        │
        ├─ is_store_version(index, n_fields) == true
        │     → rec_set_instant_row_version_new(rec, table->current_row_version)
        │     → nulls -= 1                       // 版本字节在 nulls 前面
        │     → REC_IS_VERSIONED                 // V2：1 字节 row_version
        │
        └─ false（n_fields < index->n_fields，即 UPDATE 未物化默认值）
              └─ index->is_tuple_instant_format(n_fields)
                    → rec_set_n_fields(rec, n_fields)     // 返回占 1 还是 2 字节
                    → nulls -= n_fields_len
                    → REC_IS_INSTANT              // V1：1~2 字节 n_fields
              └─ 否则 → 保持 REC_IS_SIMPLE
```

**★ 写入侧的 nullable 数必须和上面这条链一致**：`rec_get_converted_size_comp_prefix_low()` 里预留头字节的逻辑与之严格对应——V2 加 1 字节，V1 加 `rec_get_n_fields_length(n_fields)` 字节。**两者不匹配就会踩坏 NULL 位图**。

**★★ `materialize_instant_default()`：什么时候"不"物化默认值**（★ "坑一"的直接答案）：

```cpp
/* btr/btr0cur.cc:3614 */
bool materialize_instant_default(const dict_index_t *index, const rec_t *rec) {
  /* If the record is versioned, the old instant add defaults must already have
  been materialized. */
  if (rec_new_is_versioned(rec)) return true;

  /* If table has version (i.e. INSTANT ADD/DROP is done), this is gauranteed
  that the maximum row will fit within the limit or else INSTANT ADD/DROP would
  have failed. So it is safe to materialize it. */
  if (index->has_row_versions()) return true;

  /* The record is not versioned. If the index has instant default from older
  version, we must not materialize it. */
  if (index->has_instant_cols()) return false;      /* ★★ V1 升级表的老行：不物化 */
  return true;
}
```

注释原文解释了动机（这是"坑一"最完整的一手材料）：

> *"For upgraded table with INSTANT ADD column done in previous implementaion, don't materialize INSTANT columns for existing rows during update. The reason is, earlier INSTANT ADD column is allowed even if after adding that column, max size of a possible row may cross the row_size_limit. This creates an issue with the UPDATE/ROLLBACK. Let's say during UPDATE all INSTANT ADD COLS are materialized, updated row is fitting the row_size_limit. But if a rollback is done and INSTANT columns are kept materialised row may exceed row_size_limit. Thus ROLLBACK will fail. So for these rows, we keep the old behavior."*

> ★★★ 所以：**V1 升级表的老行在 UPDATE 时不物化默认值**（`n_fields` 保持原样 → 走 V1 分支），而 **V2 表的行一律物化**。差别在于 V1 时代允许"加列后最大行超过 `row_size_limit`"，V2 则在 DDL 时就拦住了（`is_instant_add_drop_possible()` 判行大小）。

**★ `dtuple_t::ignore_trailing_default()`：反向裁剪**（★ V1 兼容路径的入口）：

```cpp
/* data/data0data.cc:797 */
void dtuple_t::ignore_trailing_default(const dict_index_t *index) {
  ut_a(index->has_instant_cols());
  ut_a(!index->has_row_versions());

  /* It's necessary to check all the fields that could be default.
  If it's from normal update, it should be OK to keep original
  default values in the physical record as is, however,
  if it's from rollback, it may rollback an update from default
  value to non-default. To make the rolled back record as is,
  it has to check all possible default values. */
  for (; n_fields > index->get_instant_fields(); --n_fields) {
    const dict_col_t *col = index->get_field(n_fields - 1)->col;
    ut_a(!col->is_instant_added());
    ut_a(!col->is_instant_dropped());
    const dfield_t *dfield = dtuple_get_nth_field(this, n_fields - 1);
    ulint len = dfield_get_len(dfield);
    ut_ad(col->instant_default != nullptr);
    if (len != col->instant_default->len ||
        (len != UNIV_SQL_NULL &&
         memcmp(dfield_get_data(dfield), col->instant_default->value, len) != 0)) {
      break;                                       /* ★ 遇到第一个"不等于默认值"的列就停 */
    }
  }
}
```

> ★★ 它**从尾部往回裁掉**等于默认值的字段，把 `n_fields` 减到 `index->get_instant_fields()`。于是 `is_store_version()` 返回 false → 走 V1 写 `n_fields`。**这就是 rollback 时能还原成"老行格式"的机制**。
> ⚠️ **不存在 `dtuple_create_instant_fields()` / `dtuple_set_instant_defaults()`**——instant 新列的默认值**从不**在写入侧填进 dtuple，而是在**读侧**由 `rec_get_nth_field_instant()` 直接返回。

**★★ UPDATE 与 undo：一个重大勘误**（★ 与多数文章的写法相反）：

```cpp
/* trx/trx0rec.cc:1371 —— 写 undo：把"列号"换成"物理位置" */
      if (index->has_row_versions() && !is_virtual) {
        /* Write physical position of field in UNDO */
        auto phy_pos = index->get_field(pos)->col->get_col_phy_pos();
        ut_ad(phy_pos == fld->field_phy_pos);
        ut_ad(!index->get_field(pos)->col->is_instant_dropped());
        ptr += mach_write_compressed(ptr, phy_pos);   /* ★★ V2 表：写 phy_pos */
      } else {
        ptr += mach_write_compressed(ptr, pos);       /* 普通表：写逻辑列号 */
      }
      /* Save the old value of field */
      field = rec_get_nth_field_instant(rec, offsets, pos, index, &flen);  /* ★ 缺的列这里直接拿 instant_default */

/* trx/trx0rec.cc:1856 —— 读 undo：phy_pos → 逻辑位 */
      if (index->has_row_versions()) {
        auto log_pos = index->fields_array[field_no];   /* ★ 经 fields_array 反查 */
        upd_field_set_field_no(upd_field, log_pos, index);
      } else {
        upd_field_set_field_no(upd_field, field_no, index);
      }
```

> ★★★ **undo 记录里根本不存 row version**——`trx_undo_rec_get_row_version()` / `trx_undo_rec_set_row_version()` **都不存在**（全 `storage/innobase/trx/` 目录 grep `row_version` 只命中 4 处，全是 `index->has_row_versions()`；`trx0undo.cc` 零命中）。**undo 侧唯一的 instant 适配就是把"列号"从逻辑位置改成物理位置（phy_pos）**，这样 rollback 才能按行的物理布局还原。
> ★ 也没有任何 instant 专用的 `TRX_UNDO_*` 常量。

**★ `row_upd_store_row()`：存进 undo 前不是"裁剪"而是"补齐"**：

```cpp
/* row/row0upd.cc:1979 */
  node->row = row_build(ROW_COPY_DATA, clust_index, rec, offsets, nullptr,
                        nullptr, nullptr, ext, node->heap);
```

> ★ `row_build(ROW_COPY_DATA, ...)` 内部对每一列调 `rec_get_nth_field_instant()`，**缺的列自动被 `dict_col_t::instant_default` 补上**。所以 `node->row` 是一份**逻辑完整**的 dtuple。这与 `ignore_trailing_default()` 的"裁剪"是**两个方向**——一个为 undo 保存完整旧值，一个为写回时还原老行格式。

**★ `row_upd_rec_in_place()` 为什么要先存后取两个 bit**：

```cpp
  if (rec_offs_comp(offsets)) {
    /* Keep the INSTANT/VERSION bit of prepared physical record */
    const bool is_instant = rec_get_instant_flag_new(rec);
    const bool is_versioned = rec_new_is_versioned(rec);
    rec_set_info_bits_new(rec, update->info_bits);     /* ★ 会覆盖整个 info bits 字节 */
    if (is_versioned)      rec_new_set_versioned(rec);
    else if (is_instant) { ut_ad(index->table->has_instant_cols());
                           rec_new_set_instant(rec); }
    else                   rec_new_reset_instant_version(rec);
    ut_a(!(rec_get_instant_flag_new(rec) && rec_new_is_versioned(rec)));
  }
  ...
    /* Updating default value for instantly added columns must not be done
    in-place. See also row_upd_changes_field_size_or_external() */
    ut_ad(!rec_offs_nth_default(index, offsets, field_no));   /* ★ 更新默认值列必须走悲观路径 */
```

> ★★ **INSTANT/VERSION 两个 bit 就住在 info bits 那个字节里**，而 `rec_set_info_bits_new()` 会整体覆盖它——所以必须先读出来再写回去。
> ★ 配套 `row_upd_changes_field_size_or_external()`：目标列带 `REC_OFFS_DEFAULT` 时把 `old_len` 强设为 `UNIV_SQL_ADD_COL_DEFAULT`，**强制悲观更新**（默认值不在行内，任何更新都会让记录变长）。

**读取侧按版本解析**：`rec_init_null_and_len_comp`（rem0rec.cc）从记录头解析出 `row_version`（初始 `UINT8_UNDEFINED`），`get_nullable_fields_for_rec` 用它取该版本的 nullable 数——`index->get_nullable_in_version(rec_version)`，这就是 `create_nullables` 按版本维护的数组的用途：

```cpp
static size_t get_nullable_fields_for_rec(const dict_index_t *index,
                                          const size_t n_fields,
                                          uint8_t &rec_version) {
  const bool is_valid_version = is_valid_row_version(rec_version);
  size_t nullable_fields = 0;

  if (!is_valid_version) {
    /* Invalid version. Temp record for redundnat format. Record being logged
    doesn't have version. It will be stored in version=0 */
    rec_version = 0;                                        // ① 无版本号 → 按版本 0 处理
    nullable_fields = index->get_nullable_in_version(rec_version);
    return nullable_fields;
  }

  if (index->has_row_versions()) {
    /* Table has version. New records will be stored in latest format. */
    nullable_fields = index->get_nullable_in_version(rec_version);   // ② V2：按版本号查表
  } else if (index->has_instant_cols()) {
    /* Table has no version. New records will be written as of old way. */
    nullable_fields =
        index->get_n_nullable_before(static_cast<uint32_t>(n_fields)); // ③ V1：按字段数反推
  } else {
    /* Table has no version no instant. All the fields will be there. */
    nullable_fields = index->n_nullable;                    // ④ 普通记录：全量
  }

  return nullable_fields;
}
```

★ 这四个分支就是本篇所有"版本机制"的收口：② 是 V2 路径（按 `row_version` 查 `nullables[]`），③ 是 V1 路径（没有版本号，只能按记录里的 `n_fields` 反推），① 处理 REDUNDANT/temp 这类"记录里没有版本"的格式，④ 是没做过 instant 的普通表。**同一个函数把三种记录格式统一成一个"取 nullable 数"的动作**，上层解析无需关心版本细节。

### 提交路径与元数据

#### 实现入口与分支

`dict0inst.cc`（文件头注释 "Instant DDL interface implementation"），`Instant_ddl_impl<Table>` 同时实例化 `dd::Table` 与 `dd::Partition`（分区表也支持，但 `dd_part_is_first` 只让第一个分区做元数据变更）。

`commit_instant_ddl()` 按 `Instant_Type` 分支：

| Instant_Type | 处理 |
|---|---|
| `INSTANT_NO_CHANGE` | `dd_commit_inplace_no_change(false)` |
| `INSTANT_COLUMN_RENAME` | 同上 + `innobase_discard_table`（重载表定义） |
| `INSTANT_VIRTUAL_ONLY` | 处理 FTS_DOC_ID 隐藏列 + discard table |
| `INSTANT_ADD_DROP_COLUMN` | `dd_copy_private` → drop/add → **`current_row_version++`** |
| `INSTANT_IMPOSSIBLE` | `ut_ad(0)` |

`INSTANT_ADD_DROP_COLUMN` 关键步骤：

```cpp
bool Instant_ddl_impl<Table>::commit_instant_ddl() {
  // ★ Instant_Type 由引擎在 check_if_supported_inplace_alter 里塞进 handler_trivial_ctx
  Instant_Type type =
      static_cast<Instant_Type>(m_ha_alter_info->handler_trivial_ctx);

  switch (type) {
    case Instant_Type::INSTANT_NO_CHANGE:
      dd_commit_inplace_no_change(false);
      break;

    case Instant_Type::INSTANT_COLUMN_RENAME:
      dd_commit_inplace_no_change(false);
      if (!dd_table_is_partitioned(m_new_dd_tab->table()) ||
          dd_part_is_first(reinterpret_cast<dd::Partition *>(m_new_dd_tab))) {
        dd_update_v_cols(&m_new_dd_tab->table(), m_dict_table->id);
      }
      row_mysql_lock_data_dictionary(m_trx, UT_LOCATION_HERE);
      innobase_discard_table(m_thd, m_dict_table);      // 重载表定义
      row_mysql_unlock_data_dictionary(m_trx);
      break;

    case Instant_Type::INSTANT_VIRTUAL_ONLY:
      // FTS_DOC_ID 隐藏列若被删掉要补回
      if (dd_find_column(&m_old_dd_tab->table(), FTS_DOC_ID_COL_NAME) &&
          !dd_find_column(&m_new_dd_tab->table(), FTS_DOC_ID_COL_NAME)) {
        dd::Column *col = dd_add_hidden_column(...);
        dd_set_hidden_unique_index(m_new_dd_tab->table().add_index(),
                                   FTS_DOC_ID_INDEX_NAME, col);
      }
      dd_commit_inplace_no_change(true);
      ...
      innobase_discard_table(m_thd, m_dict_table);
      break;

    case Instant_Type::INSTANT_ADD_DROP_COLUMN:
      trx_start_if_not_started(m_trx, true, UT_LOCATION_HERE);
      dd_copy_private(*m_new_dd_tab, *m_old_dd_tab);

      /* Fetch the columns which are to be added or dropped */
      populate_to_be_instant_columns();

      ut_ad(!m_cols_to_add.empty() || !m_cols_to_drop.empty());

      if (!m_cols_to_drop.empty()) {
        /* INSTANT DROP —— 注意顺序：先 DROP 再 ADD */
        if (commit_instant_drop_col()) return true;
      }
      if (!m_cols_to_add.empty()) {
        /* INSTANT ADD */
        if (commit_instant_add_col()) return true;
      }

      /* Update the current row version in dictionary cache */
      m_dict_table->current_row_version++;               // ★ 版本递增

      ut_ad(dd_table_has_instant_cols(m_new_dd_tab->table()));

      // 每个索引的私有元数据记下事务 id
      for (auto dd_index : *m_new_dd_tab->indexes()) {
        dd::Properties &p = dd_index->se_private_data();
        p.set(dd_index_key_strings[DD_INDEX_TRX_ID], m_trx->id);
      }

      row_mysql_lock_data_dictionary(m_trx, UT_LOCATION_HERE);
      innobase_discard_table(m_thd, m_dict_table);
      row_mysql_unlock_data_dictionary(m_trx);
      break;

    case Instant_Type::INSTANT_IMPOSSIBLE:
      ...
  }
}
```

★ 三点值得记：① `Instant_Type` 是从 `handler_trivial_ctx` 取回的——它由引擎在 `check_if_supported_inplace_alter` 里写入（`ha_alter_info->handler_trivial_ctx = instant_type_to_int(instant_type)`，见 [`inplace_ddl.md`](inplace_ddl.md)），**判定与提交靠这个整数字段接力**；② 只有 `INSTANT_ADD_DROP_COLUMN` 分支会 `current_row_version++`——**这就是"行版本号只在 instant 加/删列时递增"的源码证据**（对应「三个 version 的辨析」）；③ 每个分支末尾都要 `innobase_discard_table()` 重载表定义，因为 DD 元数据已经变了、缓存必须失效。

但 switch 里每个分支调用的仍是封装层。继续下钻两个最核心的实现。

#### 深入：`dd_drop_instant_columns`——删列的真相是"留一个隐藏副本"

```cpp
for (const auto &column : cols_to_drop) {
  ut_ad(!innobase_is_v_fld(column));

  /* Get column to be dropped from old table def */
  dd::Column *col_to_drop = dd_find_column(old_dd_table, column->field_name);
  dd::Properties &private_data = col_to_drop->se_private_data();

  // ① 取物理位置：优先从 se_private_data，没有则从内存 dict_col_t 取
  uint32_t phy_pos = UINT32_UNDEFINED;
  if (!private_data.exists(dd_column_key_strings[DD_INSTANT_PHYSICAL_POS])) {
    ut_ad(!dd_table_has_row_versions(*old_dd_table));
    const dict_col_t *col = new_dict_table->get_col_by_name(column->field_name);
    phy_pos = col->get_phy_pos();
  } else {
    private_data.get(DD_INSTANT_PHYSICAL_POS, &phy_pos);
  }

  // ② ★ 构造隐藏列的新名字：把版本号与物理位置编进名字
  std::string dropped_col_name(col_to_drop->name().c_str());
  set_dropped_column_name(dropped_col_name,
                          new_dict_table->current_row_version + 1, phy_pos);

  /* Add this column as an SE_HIDDEN column in new table def. NOTE: This call
  will update the DD_INSTANT_VERSION_DROPPED for the column as well. */
  dd::Column *dropped_col = dd_add_hidden_column(
      new_dd_table, dropped_col_name.c_str(),
      col_to_drop->char_length(), col_to_drop->type());
  if (dropped_col == nullptr) {
    // ★ 名字撞车：新表里已经有一个同名列
    ib::info(ER_IB_HIDDEN_NAME_CONFLICT, dropped_col_name.c_str(), ...)
        << "If you have any conflicting user column please rename it.";
    return true;
  }

  // ③ 把原列的完整类型信息抄到隐藏列上
  dd::Properties &private_data = dropped_col->se_private_data();
  if (dd_column_is_added(col_to_drop)) {
    private_data.set(DD_INSTANT_VERSION_ADDED,
                     dd_column_get_version_added(col_to_drop));  // 继承 v_added
  }
  private_data.set(DD_INSTANT_VERSION_DROPPED,
                   new_dict_table->current_row_version + 1);      // ★ v_dropped
  private_data.set(DD_INSTANT_PHYSICAL_POS, phy_pos);

  dropped_col->set_nullable(col_to_drop->is_nullable());
  dropped_col->set_char_length(col_to_drop->char_length());
  dropped_col->set_numeric_scale(col_to_drop->numeric_scale());
  dropped_col->set_unsigned(col_to_drop->is_unsigned());
  dropped_col->set_collation_id(col_to_drop->collation_id());
  dropped_col->set_type(col_to_drop->type());
  /* ENUM / SET 连枚举元素都要抄过来 */
  if (col_to_drop->type() == dd::enum_column_types::ENUM ||
      col_to_drop->type() == dd::enum_column_types::SET) {
    for (const auto *source_elem : col_to_drop->elements()) {
      dropped_col->add_element()->set_name(source_elem->name());
    }
  }
}

instant_update_table_cols_count(new_dict_table, 0, cols_to_drop.size());
```

★ 三点是理解 instant DROP 的关键：

1. **删列不是删除，是"改名 + 转成 `SE_HIDDEN` 列"保留在新表定义里**。原因很实在：老行里还实实在在存着这个字段的字节，解析老行时必须知道它的**类型与物理位置**，否则整行都解不开。
2. **隐藏列的名字里编码了版本号与物理位置**（`set_dropped_column_name(name, v, phy_pos)`）——这样多次删同名列也不会撞车。但用户若正好有个列叫这个名字就会冲突 → 报 `ER_IB_HIDDEN_NAME_CONFLICT`，源码还提示"请重命名冲突的用户列"。
3. **类型信息要完整抄一遍**（含 ENUM/SET 的元素列表），因为隐藏列将来还要用于解析老行数据——这也解释了为什么反复 instant 加删列会让表定义越来越臃肿。

#### 深入：`populate_to_be_instant_columns_low`——待增删列的计算逻辑

`commit_instant_ddl` 里调 `populate_to_be_instant_columns()` 得到 `m_cols_to_add` / `m_cols_to_drop`，它的底层实现是**比对新旧两表的字段列表**（而不是直接读 `Alter_info` 的 drop/add 列表）：

```cpp
static void populate_to_be_instant_columns_low(
    const Alter_inplace_info *ha_alter_info, const TABLE *old_table,
    const TABLE *altered_table, Columns &cols_to_add, Columns &cols_to_drop) {
  /* Collect all renamed columns */
  std::vector<renamed_fields_t> renamed_fields;
  for (size_t i = 0; i < old_table->s->fields; i++) {
    const char *field_name = old_table->field[i]->field_name;
    std::string new_name;
    if (is_renamed(ha_alter_info, field_name, new_name)) {
      renamed_fields.push_back(std::pair(field_name, new_name));
    }
  }

  for (size_t i = 0; i < old_table->s->fields; i++) {
    Field *old_table_field = old_table->field[i];
    const char *old_field_name = old_table_field->field_name;

    /* Skip virtual column from old table */
    if (innobase_is_v_fld(old_table_field)) continue;

    /* Skip if this column is being renamed */
    auto it = std::find_if(renamed_fields.begin(), renamed_fields.end(),
                           [&old_field_name](renamed_fields_t &element) {
                             return strcmp(element.first.c_str(), old_field_name) == 0;
                           });
    if (it != renamed_fields.end()) continue;    // ★ 改名的列不算"删一列 + 加一列"

    /* Look for this column in new table */
    bool found = false;
    for (size_t j = 0; j < altered_table->s->fields; j++) {
      Field *new_table_field = altered_table->field[j];
      if (innobase_is_v_fld(new_table_field)) continue;

      if (strcmp(old_field_name, new_table_field->field_name) == 0) {
        /* Check if this column is in drop list of alter_info. */
        if (is_dropped(ha_alter_info, old_field_name)) {
          /* This column is being dropped */
          cols_to_drop.push_back(old_table_field);
          /* But this column is present in new table as well which is possible
          if a new column with same name is being added or an existing column
          is being renamed to this name. */
        }
        found = true;
        break;
      }
    }
    if (!found) cols_to_drop.push_back(old_table_field);   // 新表没有 = 被删
  }
  // 反向遍历新表字段，旧表没有的 → cols_to_add
}
```

★ 三条特判值得记：

1. **改名的列要排除**——否则 `RENAME COLUMN a TO b` 会被误判成"删 a + 加 b"，白白消耗一次行版本号（这也解释了为什么 `INSTANT_COLUMN_RENAME` 在 `commit_instant_ddl` 里是独立分支、不进 ADD_DROP）。
2. **虚拟列跳过**（`innobase_is_v_fld`）——它不占物理行空间。
3. **"新表也有同名列"要细分**：可能是"删了又加同名列"，也可能是"另一列改名成这个名字"——源码注释明确列了这两种可能。

⇒ 这就是 INSTANT 对"改名 + 删列 + 加列"这类组合判定特别绕的根源，也是限制表里那些"不支持的组合"的由来。

#### 深入：`is_instant_add_drop_possible`——行大小限制的计算

限制那节说"加列后行大小可能超限就不允许"，源码实现是**构造 dummy 列试算**，不实际分配任何东西：

```cpp
/* Get maximum permissible size on page */
size_t page_rec_max;
size_t page_ptr_max;
get_permissible_max_size(dict_table, index, page_rec_max, page_ptr_max);

/* Get the maximum size of a valid rec in current table */
size_t current_max_size;
bool res = dict_index_validate_max_rec_size(dict_table, index, true,
                                            page_rec_max, page_ptr_max,
                                            current_max_size);

if (res) {
  /* Table is already in a state where possible row size can go beyond
  permissible size limit. Don't allow INSTANT ADD */
  return false;                       // ① 当前表已处于"可能超限"状态 → 一律拒绝
}

for (auto field : cols_to_add) {
  if (innobase_is_v_fld(field)) continue;   // 虚拟列不占物理行空间，跳过

  /* Create a dummy dict_cot_t and dict_field_t just to calculate size */
  dict_col_t dummy_col;                     // ② 纯粹为了算大小而造的临时对象
  dummy_col.mtype = mtype;
  dummy_col.prtype = prtype;
  dummy_col.len = col_len;
  ulint mbminlen, mbmaxlen;
  dtype_get_mblen(mtype, prtype, &mbminlen, &mbmaxlen);
  dummy_col.set_mbminmaxlen(mbminlen, mbmaxlen);
  dict_field_t dummy_field;
  dummy_field.col = &dummy_col;
  ...                                        // 算出 field_max_size 再与 page_rec_max 比较
}
```

★ 判定分两段：先看**现有表**是否已经处于"可能超限"状态（`dict_index_validate_max_rec_size` 返回 true 就直接拒绝一切 instant ADD），再逐列试算新列加进去后的最坏情况。虚拟列 `innobase_is_v_fld` 跳过——它不占物理行空间。

> 这也回答了一个常见疑问：**为什么一张已经很宽的表再做 instant ADD 会失败**——不是因为这一列本身大，而是当前表已经踩在 `page_rec_max` 的边缘，判定直接返回 false。

加列与删列各自的实现（`_low` 版本，被 `dd::Table` / `dd::Partition` 两个模板实例共用）：

```cpp
bool Instant_ddl_impl<Table>::commit_instant_add_col_low() {
  ut_ad(!m_dict_table->is_temporary());
  ut_a(is_instant_add_drop_possible(m_ha_alter_info, m_old_table,
                                    m_altered_table, m_dict_table));

  /* To remember old default values if exist */
  dd_copy_table_columns(m_ha_alter_info, m_new_dd_tab->table(),
                        m_old_dd_tab->table(), m_dict_table);

  /* Then add all new default values */
  if (dd_add_instant_columns(&m_old_dd_tab->table(), &m_new_dd_tab->table(),
                             m_dict_table, m_cols_to_add))
    return true;

  /* Keep the metadata for newly added virtual columns if exist */
  dd_update_v_cols(&m_new_dd_tab->table(), m_dict_table->id);

  return false;
}

bool Instant_ddl_impl<Table>::commit_instant_drop_col_low() {
  ut_ad(!m_dict_table->is_temporary());
  ut_a(is_instant_add_drop_possible(...));

  /* Copy columns metadata */
  dd_copy_table_columns(m_ha_alter_info, m_new_dd_tab->table(),
                        m_old_dd_tab->table(), m_dict_table);

  /* Update metadata of columns to be dropped */
  return dd_drop_instant_columns(&m_old_dd_tab->table(), &m_new_dd_tab->table(),
                                 m_dict_table, m_cols_to_drop ...);
}

// ★ 分区表只有第一个分区真正改元数据
template <>
bool Instant_ddl_impl<dd::Partition>::commit_instant_add_col() {
  if (dd_part_is_first(m_new_dd_tab)) {
    if (commit_instant_add_col_low()) return true;
  }
  return false;
}
```

★ 两点：① 两个 `_low` 函数结构对称——**都是"先拷已有列元数据，再写新增/待删列的元数据"**，全程只动 DD，不碰任何行；② 分区表的 `commit_instant_add_col` 特化里 `dd_part_is_first()` 保证**只有第一个分区做元数据变更**（其余分区共享同一份列定义），普通表则直接调 `_low`。

但到这里只是**封装层**。真正写元数据在 `dd_add_instant_columns`（dict0dd.cc）里——下面深入这一层。

#### 深入一层：`dd_add_instant_columns` 写入 DD 的内容

**第一段：新列的物理位置怎么算出来**

```cpp
uint32_t total_cols =
    new_dict_table->total_col_count + new_dict_table->get_n_sys_cols();
uint32_t next_phy_pos = 0;
for (size_t i = 0; i < total_cols; i++) {
  dict_col_t *col = new_dict_table->get_col(i);

  if (col->get_phy_pos() == UINT32_UNDEFINED) {
    ut_ad(col == new_dict_table->get_sys_col(DATA_ROW_ID));  // 无主键时的 ROW_ID
    continue;
  }
  if (col->has_prefix_phy_pos()) {
    /* Column prefix part of clustered index. It appears twice. */
    next_phy_pos += 2;                                        // ★ 前缀列占两个位置
    continue;
  }
  next_phy_pos++;
}
```

即：新列只能**接在现有列之后**（这就是"只能加到末尾"的物理原因），前缀索引列因为出现两次要占 2 个位置。

**第二段：逐列填充 `se_private_data`（核心）**

```cpp
for (const auto new_column : cols_to_add) {
  Field *field = new_column;
  ut_ad(!innobase_is_v_fld(field));      // ★ 虚拟列不占物理位置，不走这里

  /* The MySQL type code has to fit in 8 bits
  in the metadata stored in the InnoDB change buffer. */
  ut_ad(field->charset() == nullptr ||
        field->charset()->number <= MAX_CHAR_COLL_NUM);

  dd::Column *column = dd_find_column(new_dd_table, field->field_name);
  dd::Properties &se_private = column->se_private_data();

  se_private.set(dd_index_key_strings[DD_TABLE_ID], new_dict_table->id);

  /* Set Version Added */
  se_private.set(dd_column_key_strings[DD_INSTANT_VERSION_ADDED],
                 new_dict_table->current_row_version + 1);     // ★ v_added

  /* Set physical position on row */
  se_private.set(dd_column_key_strings[DD_INSTANT_PHYSICAL_POS],
                 next_phy_pos + cols_added++);                 // ★ 物理位置

  /* Set Default NULL */
  if (field->is_real_null()) {
    se_private.set(dd_column_key_strings[DD_INSTANT_COLUMN_DEFAULT_NULL], true);
    continue;                                                  // ★ NULL 只记标记
  }

  set_col_default(field, se_private);                          // 非 NULL：编码后存
}

instant_update_table_cols_count(new_dict_table, cols_to_add.size(), 0);
```

★ 三个要点：① **`DD_INSTANT_VERSION_ADDED = current_row_version + 1`** —— 新列从**下一版**起存在（随后 `commit_instant_ddl` 里 `current_row_version++` 正好补上这一格），这正是读取侧判定 `v_added ≤ row_version < v_dropped` 的依据；② **物理位置连续递增**（`next_phy_pos + cols_added++`）；③ **默认值是 NULL 时只写一个布尔标记**，不存值——省空间。

**第三段：非 NULL 默认值是怎么编码的**（`set_col_default` lambda 的关键行）

```cpp
// 构造一个"临时 dict_col_t"来承载类型信息
dict_col_t col;
/* Set a fake col_pos, since this should be useless */
dict_mem_fill_column_struct(&col, 0, mtype, prtype, col_len, true,
                            UINT32_UNDEFINED,
                            new_dict_table->current_row_version, 0);
dfield_t dfield;
col.copy_type(dfield_get_type(&dfield));

// MySQL 行格式 → InnoDB 行格式
row_mysql_store_col_in_innobase_format(&dfield, reinterpret_cast<byte *>(&buf),
                                       true, mysql_data, size,
                                       dict_table_is_comp(new_dict_table));

// 再编码成可存进 DD 的字符串
DD_instant_col_val_coder coder;
size_t length = 0;
const char *value = coder.encode(reinterpret_cast<byte *>(dfield.data),
                                 dfield.len, &length);

se_private.set(dd_column_key_strings[DD_INSTANT_COLUMN_DEFAULT],
               dd::String_type(value, length));
```

★ 这条链路解释了一个观测现象：**`INNODB_COLUMNS.DEFAULT_VALUE` 里显示的是"内部二进制格式"而不是 SQL 字面量**——因为存的就是 InnoDB 内部格式经 coder 编码后的结果，不是 `DEFAULT 5` 这样的文本。源码注释也提醒 `set_col_default` 的类型推导必须**与 `dd_fill_dict_table()` 保持一致**（"Keep this same with the code in dd_fill_dict_table(), except FTS check"），否则读出来的默认值会解析错。

> 反向操作是 `dd_match_default_value()`：IMPORT 时用 `coder.decode()` 还原并与内存里的 `col->instant_default` 逐字节比对——不一致就拒绝（见「坑二：`.cfg` 缺失时 IMPORT 会失败」）。

#### 列元数据存在哪（DD se_private_data）

每列的 `se_private_data`（dict0dd.cc）记录：

| key | 含义 |
|---|---|
| `DD_INSTANT_VERSION_ADDED` | = `current_row_version + 1`，该列从哪个版本开始存在 |
| `DD_INSTANT_VERSION_DROPPED` | = `current_row_version + 1`，该列从哪个版本起被删 |
| `DD_INSTANT_PHYSICAL_POS` | 物理位置 |
| `DD_INSTANT_COLUMN_DEFAULT` / `_DEFAULT_NULL` | instant 加列时**老行的默认值** |

**删除列不是真删**：改名成 SE_HIDDEN 列保留（`set_dropped_column_name`），保证老版本行仍能被正确解析。

**★ 这些 key 的定义不在 `sql/dd/`**（该目录 0 命中），真身在 `storage/innobase/include/dict0dd.h`——**表级 1 个、列级 5 个、分区级 1 个**：

```cpp
enum dd_table_keys {
  DD_TABLE_AUTOINC,
  DD_TABLE_DATA_DIRECTORY,
  DD_TABLE_VERSION,
  DD_TABLE_DISCARD,
  /** Columns before first instant ADD COLUMN, used only for V1 */
  DD_TABLE_INSTANT_COLS,        // ★ 表级："首次 instant ADD 前的列数"
  DD_TABLE__LAST
};

enum dd_column_keys {
  /** Default value when it was added instantly */
  DD_INSTANT_COLUMN_DEFAULT,
  /** Default value is null or not */
  DD_INSTANT_COLUMN_DEFAULT_NULL,
  /** Row version when this column was added instantly */
  DD_INSTANT_VERSION_ADDED,
  /** Row version when this column was dropped instantly */
  DD_INSTANT_VERSION_DROPPED,
  /** Column physical position on row when it was created */
  DD_INSTANT_PHYSICAL_POS,      // ★ 解决 DROP + ADD 复用物理槽位
  DD_COLUMN__LAST
};

enum dd_partition_keys {
  DD_PARTITION_ROW_FORMAT,
  /** Columns before first instant ADD COLUMN.
  This is necessary for each partition because different partition
  may have different instant column numbers, especially, for a
  newly truncated partition, it can have no instant columns.
  So partition level one should be always >= table level one. */
  DD_PARTITION_INSTANT_COLS,
  DD_PARTITION_DISCARD,
  DD_PARTITION__LAST
};
```

```cpp
/** InnoDB private key strings for dd::Table. @see dd_table_keys */
const char *const dd_table_key_strings[DD_TABLE__LAST] = {
    "autoinc", "data_directory", "version", "discard", "instant_col"};

/** InnoDB private key strings for dd::Column, @see dd_column_keys */
const char *const dd_column_key_strings[DD_COLUMN__LAST] = {
    "default", "default_null", "version_added", "version_dropped",
    "physical_pos"};

/** InnoDB private key strings for dd::Partition. @see dd_partition_keys */
const char *const dd_partition_key_strings[DD_PARTITION__LAST] = {
    "format", "instant_col", "discard"};
```

> ★ **数组顺序必须与枚举严格一致**——这是靠下标访问的，插错一个就全错位。

**★★ 默认值编码：`DD_instant_col_val_coder` 是 hex 编码，体积翻倍**（`dict0dd.h:342` 注释原文）：

```cpp
/** Class to decode or encode a stream of default value for instant table.
The decode/encode are necessary because that the default values would b
kept as InnoDB format stream, which is in fact byte stream. However,
to store them in the DD se_private_data, it requires text(char).
So basically, the encode will change the byte stream into char stream,
by splitting every byte into two chars, for example, 0xFF, would be split
into 0x0F and 0x0F. So the final storage space would be double. */
class DD_instant_col_val_coder {
 public:
  const char *encode(const byte *stream, size_t in_len, size_t *out_len);
  const byte *decode(const char *stream, size_t in_len, size_t *out_len);
 private:
  void cleanup() { ut::delete_arr(m_result); }
  byte *m_result;
};

const char *DD_instant_col_val_coder::encode(const byte *stream, size_t in_len,
                                             size_t *out_len) {
  cleanup();
  m_result = ut::new_arr_withkey<byte>(UT_NEW_THIS_FILE_PSI_KEY,
                                       ut::Count{in_len * 2});   // ★ 1 字节 → 2 字符
  char *result = reinterpret_cast<char *>(m_result);
  for (size_t i = 0; i < in_len; ++i) {
    uint8_t v1 = ((stream[i] & 0xF0) >> 4);
    uint8_t v2 = (stream[i] & 0x0F);
    result[i * 2] = (v1 < 10 ? '0' + v1 : 'a' + v1 - 10);
    result[i * 2 + 1] = (v2 < 10 ? '0' + v2 : 'a' + v2 - 10);
  }
  *out_len = in_len * 2;
  return result;
}
```

> ★★ **这就是 `INNODB_COLUMNS.DEFAULT_VALUE` 显示"内部二进制格式"而不是 SQL 字面量的根因**——存的是 InnoDB 行内字节流经 hex 编码后的结果。
> ⚠️ 编码是**小写 hex**（`'a' + v - 10`），不是大写。

**★ 三个"是否 instant"的判定函数各不相同**（常混用）：

```cpp
// V1 判据：表级 "instant_col" 键存在
inline bool dd_table_is_upgraded_instant(const dd::Table &table) {
  if (table.is_temporary()) return false;
  return (table.se_private_data().exists(
      dd_table_key_strings[DD_TABLE_INSTANT_COLS]));
}

// V2 判据：第一个非虚拟列带 physical_pos
inline bool dd_table_has_row_versions(const dd::Table &table) {
  if (table.is_temporary()) return false;
  bool has_row_version = false;
  for (const auto column : table.columns()) {
    if (column->is_virtual()) continue;
    if (column->se_private_data().exists(
            dd_column_key_strings[DD_INSTANT_PHYSICAL_POS])) {
      has_row_version = true;
    }
    break;      // ★ 只看一列就够
  }
  return has_row_version;
}

// V1 ∪ V2
inline bool dd_table_has_instant_cols(const dd::Table &table) {
  bool instant_v1 = dd_table_is_upgraded_instant(table);
  bool instant_v2 = dd_table_has_row_versions(table);
  bool instant = instant_v1 || instant_v2;
  ut_ad(!instant || dd_instant_columns_consistent(table));
  return (instant);
}
```

> ⚠️ 还有 `dd_table_has_instant_add_cols()` / `dd_table_has_instant_drop_cols()`（遍历全部列）——**共 5 个不同函数**，别混。

**★★ `dict_col_t` 的版本成员真名不是 `v_added` / `v_dropped`**（`dict0mem.h:488`）：

```cpp
 private:
  /* Position of column on physical row.
  If column prefix is part of PK, it appears twice on row. First 2 bytes are
  for prefix position and next 2 bytes are for column position on row. */
  uint32_t phy_pos{UINT32_UNDEFINED};

  /* Row version in which this column was added INSTANTly to the table */
  uint8_t version_added{UINT8_UNDEFINED};     // ⚠️ 不是 v_added

  /* Row version in which this column was dropped INSTANTly from the table */
  uint8_t version_dropped{UINT8_UNDEFINED};   // ⚠️ 不是 v_dropped
```

```cpp
  bool is_visible_in_version(uint8_t version) const {   // ★ 可见性判定的唯一入口
    ut_ad(is_valid_row_version(version));
    return (!is_added_after(version) && !is_dropped_in_or_before(version));
  }
```

> ★ `phy_pos` 的 **bit15 是"有前缀"标志**：`has_prefix_phy_pos()` 返回 `phy_pos & 0x8000`，高 16 位是前缀位置、低 15 位是列位置。
> ⚠️ `create_nullables()` 的**注释**里仍写着 `v_added` / `v_dropped`，是过时注释——真实成员名已改。

#### row version 的配套内存结构

`dict_index_t::create_nullables(current_row_version)`（dict0mem.cc）为**每个 row version** 维护独立的 nullable 计数（`nullables[MAX_ROW_VERSION + 1]`）——因为不同版本的行，列的可空性不同：

```cpp
void dict_index_t::create_nullables(uint32_t current_row_version) {
  ut_ad(is_clustered());
  ut_ad(is_valid_row_version(current_row_version));

  memset(nullables, 0, (MAX_ROW_VERSION + 1) * sizeof(nullables[0]));

  auto update_nullable = [&](size_t start_version, bool is_increment) {
    ut_ad(is_valid_row_version(start_version));
    // ★ 从该列"生效"的版本起，一路累加/递减到当前版本
    for (size_t i = start_version; i <= current_row_version; i++) {
      ut_ad(is_increment || nullables[i] > 0);

      if (is_increment) {
        ++nullables[i];
      } else {
        --nullables[i];
      }
    }
  };

  for (uint32_t i = 0; i < n_def; i++) {
    dict_field_t *field = get_field(i);
    ...  // 按列的 v_added / v_dropped 调用 update_nullable
  }
}
```

★ 这是**预计算而非逐行遍历**：每列只在它"生效"的版本区间上累加一次，读取时 O(1) 拿到该版本的 nullable 数——这正是"引入版本化却不牺牲读取性能"的关键。另外 `nullables[MAX_ROW_VERSION + 1]` 的数组大小本身就是**版本号上限的硬约束**（与 1 字节 `row_version` 呼应，见理论基础里的退化阈值）。

**★ `create_nullables()` 全文**（★ 区间累加的完整语义，含三个系统列的排除）：

```cpp
/* dict/dict0mem.cc:622 */
void dict_index_t::create_nullables(uint32_t current_row_version) {
  ut_ad(is_clustered());
  ut_ad(is_valid_row_version(current_row_version));

  memset(nullables, 0, (MAX_ROW_VERSION + 1) * sizeof(nullables[0]));   /* 65 个 uint32 清零 */

  auto update_nullable = [&](size_t start_version, bool is_increment) {
    ut_ad(is_valid_row_version(start_version));
    for (size_t i = start_version; i <= current_row_version; i++) {   /* ★★ 从"生效版本"一路到当前版本 */
      ut_ad(is_increment || nullables[i] > 0);
      if (is_increment) ++nullables[i];
      else              --nullables[i];
    }
  };

  for (uint32_t i = 0; i < n_def; i++) {
    dict_field_t *field = get_field(i);
    /* In case of redo recovery, names are not populated */
    ut_ad(field->name != nullptr || strcmp(name, RECOVERY_INDEX_TABLE_NAME) == 0);

    if (field->name != nullptr && (strcmp(field->name, "DB_ROW_ID") == 0 ||
                                   strcmp(field->name, "DB_TRX_ID") == 0 ||
                                   strcmp(field->name, "DB_ROLL_PTR") == 0)) {
      continue;                                    /* ★ 3 个系统列永不计入 nullable */
    }
    if (field->col->prtype & DATA_NOT_NULL) {
      continue;                                    /* NOT NULL 列不计入 */
    }

    /* For each version increment by 1 starting from field->col->v_added */
    size_t start_from = 0;
    if (field->col->is_instant_added()) {
      start_from = field->col->get_version_added();  /* ★ ADD 的列：从 v_added 起才存在 */
    }
    update_nullable(start_from, true);

    /* For each version decrement by 1 starting from field->col->v_dropped */
    if (field->col->is_instant_dropped()) {
      update_nullable(field->col->get_version_dropped(), false);  /* ★ DROP 的列：从 v_dropped 起不再存在 */
    }
  }
}
```

> ★★ **`nullables` 是 `uint32_t nullables[65]`，不是 `std::vector`**——固定数组，下标即 row_version（0..64）。这是"从版本号 O(1) 拿到 nullable 数"的物理基础。

---

#### ★★★ "row version → 列集合"的推导三件套

这是整个 V2 机制的心脏。8.0.39 **没有** `get_n_fields_in_version()` 这样的函数（全仓 0 匹配），"某版本可见哪些列"由下面三处协作完成：

**① 逐列判定：`dict_col_t::is_visible_in_version()`**（`dict0mem.h:781`）

```cpp
  /** Check if column is dropped before the given version. */
  bool is_dropped_in_or_before(uint8_t version) const {
    if (!is_instant_dropped()) return false;
    return (get_version_dropped() <= version);
  }

  /** Check if column is added after the current version. */
  bool is_added_after(uint8_t version) const {
    if (!is_instant_added()) return false;
    return (get_version_added() > version);
  }

  /** Check if a column is visible in given version. */
  bool is_visible_in_version(uint8_t version) const {
    return (!is_added_after(version) && !is_dropped_in_or_before(version));
  }
```

> ★★★ **整个 V2 的语义就压缩在这一行**：列在版本 v 可见 ⟺ **不是"加得比我晚"** 且 **不是"删得比我早"**。

**② 版本号 → nullable 数：`nullables[]` 数组**（见上）

**③ 物理位 ↔ 逻辑位：`fields_array`**（`dict0mem.h:1182`）

```cpp
  /** Array of field pos sorted as per their physical pos in record. Only
  needed for clustered index having INSTANT ADD/DROP columns. */
  std::vector<uint16_t> fields_array;      /* fields_array[phy_pos] = 逻辑 field index */

  void create_fields_array() {
    fields_array.resize(n_def);
    for (uint32_t i = 0; i < n_def; i++) {
      dict_field_t *field = get_field(i);
      size_t pos = field->get_phy_pos();
      fields_array[pos] = i;               /* ★ 物理位 → 逻辑位 */
    }
  }

  dict_field_t *get_physical_field(size_t pos) const {
    if (has_row_versions()) return get_field(fields_array[pos]);   /* ★ V2 必须翻译 */
    return get_field(pos);                                          /* V1/普通表 物理位==逻辑位 */
  }

  uint16_t get_field_off_pos(ulint pos) const {
    return get_field(pos)->get_phy_pos();            /* 逻辑 → 物理 */
  }

  uint16_t get_field_phy_pos(ulint pos, uint8_t version) const {
    uint16_t phy_pos = get_field(pos)->get_phy_pos();
    if (version == UINT8_UNDEFINED) return phy_pos;
    uint16_t res = phy_pos;
    for (size_t i = 0; i < phy_pos; i++) {
      if (get_field(fields_array[i])->col->is_dropped_in_or_before(version)) {
        res--;                                       /* ★ 减掉该版本前已删的列，得紧凑物理位 */
      }
    }
    return res;
  }
```

**★★ `dict_col_t` 的四个 instant 成员**（★ 每列记四个版本号/位置）：

```cpp
struct dict_col_t {
  /** Default value when this column was added instantly. */
  dict_col_default_t *instant_default{nullptr};

  unsigned ind : 10;              /* 逻辑位置（create table 中的定义顺序），instant 后不变 */

 private:
  /* Position of column on physical row. INSTANT DROP 后此值不变 —— 这正是
     "逻辑位 vs 物理位"分离的根 */
  uint32_t phy_pos{UINT32_UNDEFINED};
  /* Row version in which this column was added INSTANTly */
  uint8_t version_added{UINT8_UNDEFINED};
  /* Row version in which this column was dropped INSTANTly */
  uint8_t version_dropped{UINT8_UNDEFINED};
};
```

> ★★ **`ind`（逻辑位）与 `phy_pos`（物理位）的分离是 V2 能支持 DROP 的根本**：列被 instant DROP 后，`phy_pos` **保持不变**（老行上数据还在那个位置），但 `ind` 仍是原来的逻辑序号。读取时靠 `fields_array` 做双向翻译。
> ⚠️ `version_added` / `version_dropped` 的"未设置"哨兵是 `UINT8_UNDEFINED`（255），而 `MAX_ROW_VERSION = 64`——所以 `is_instant_added()` 必须额外判 `> 0`，**不能只判 `!= UINT8_UNDEFINED`**。

**★★ `dict_table_t` 的 instant 成员**（★ 三个计数器 + 两个标志）：

```cpp
  /** Number of non-virtual columns before first instant ADD COLUMN,
  including the system columns. Used only when table has instant ADD in V1. */
  unsigned n_instant_cols : 10;              /* ★ V1 专用 */

  /** Current row version in case columns are added/dropped INSTANTly */
  uint32_t current_row_version{0};           /* ★ V2 核心：0 = 无版本 */

  uint32_t initial_col_count{0};             /* 建表时的列数 */
  uint32_t current_col_count{0};             /* 当前"可见"列数（drop 后减少） */
  uint32_t total_col_count{0};               /* 含已 dropped 的总列数（只增不减） */
  bool m_upgraded_instant{false};            /* ★ V1 标记 */
```

```cpp
  bool has_row_versions() const {                    /* ★ 只表示 V2 */
    if (current_row_version > 0) {
      ut_ad(has_instant_add_cols() || has_instant_drop_cols());
      return true;
    }
    return false;
  }
  bool has_instant_cols() const {                    /* ★★ 只表示 V1！不含 V2 */
    if (is_upgraded_instant() || (n_instant_cols < n_cols)) return true;
    return false;
  }
  uint32_t get_n_instant_add_cols() const   { return total_col_count - initial_col_count; }
  uint32_t get_n_instant_drop_cols() const  { return total_col_count - current_col_count; }
```

> ★★★ **最容易踩的坑**：`dict_table_t::has_instant_cols()` **只表示 V1**，一个只做过 8.0.32+ instant ADD/DROP 的表，它返回 **false**。要问"这张表是不是 instant 表"必须用 `has_instant_cols() || has_row_versions()`；要问"这条记录要不要按 instant 解析"必须用 **`dict_index_t::has_instant_cols_or_row_versions()`**（它内部是 `(!is_clustered() → false) || row_versions || instant_cols`，且**二级索引恒为 false**）。

**★ `dict_index_t` 的 instant 成员与标志位落地**（`dict0dict.cc:2619`）：

```cpp
  new_index->instant_cols = false;
  new_index->row_versions = false;
  new_index->set_instant_nullable(new_index->n_nullable);

  if (new_index->is_clustered()) {
    if (new_index->table->has_row_versions()) {
      new_index->row_versions = true;                 /* ★ V2 */
    }
    if (new_index->table->has_instant_cols()) {
      ut_ad(new_index->table->is_upgraded_instant()); /* ★★ V1 必然是升级表 */
      new_index->instant_cols = true;
      const size_t n_instant_fields = new_index->get_instant_fields();
      size_t new_n_nullable = new_index->calculate_n_instant_nullable(n_instant_fields);
      new_index->set_instant_nullable(new_n_nullable);
    }
  }
```

> ★ `ut_ad(new_index->table->is_upgraded_instant())` 这条断言说明：**8.0.39 里 V1 只有"升级上来的表"这一种来源**——新做的 instant ADD/DROP 一律走 V2。

**★ `get_instant_fields()`：V1 列数的反推**（`dict0mem.ic:90`）：

```cpp
inline uint32_t dict_index_t::get_instant_fields() const {
  ut_ad(has_instant_cols());
  size_t n_inst_cols_v1 = table->get_n_instant_added_col_v1();
  size_t n_drop_cols    = table->get_n_instant_drop_cols();
  size_t n_add_cols     = table->get_n_instant_add_cols();
  size_t n_instant_fields =
      n_fields + n_drop_cols - n_add_cols - n_inst_cols_v1;   /* ★ 现字段数 + 已删 - 已加 - V1新增 */
  return n_instant_fields;
}
```

**★ REDUNDANT 里的 `dropped_col_count` 修正**（★ 这是 REDUNDANT 独有的一个下标陷阱）：

```cpp
      if (col->is_dropped_in_or_before(row_version)) {
        offs = offs | REC_OFFS_DROP;
        dropped_col_count++;                 /* ★ 累计，但 DROP 列自身不减 */
      } else {
        offs = rec_1_get_field_end_info_low(rec, i - dropped_col_count);  /* ★ 下标要减掉 */
      }
```

> ★★ 已 DROP 的列在**老行**上仍占字节，所以读 `rec_1_get_field_end_info_low()` 时下标要 `i - dropped_col_count`；但 DROP 列自己**不**减（它的数据还在）。两个分支减的数量不同，这是 REDUNDANT 比 COMPACT 多出来的复杂度。

### 限制与官方两个"坑"

#### 限制（源码依据）

1. **表级**：`dict_table_t::support_instant_add_drop()`（dict0dict.ic）——非压缩表、非 DD 表空间、无 FTS_DOC_ID、非临时表、无全文索引、非系统表；
2. **行大小**：`is_instant_add_drop_possible`（dict0inst.cc）若加列后可能超出 `page_rec_max`，直接不允许；
3. **只能加到末尾**：Instant 语义下新列不落老行，只能追加（表无主键时末尾隐式主键会导致失败）；
4. 升级线程不做 instant（`current_thd->is_server_upgrade_thread()`）；
5. 列被外键引用时不允许 instant rename。

#### 官方设计（WL#11250）的口径

> 以下来自官方 [WL#11250 *Support Instant Add Column*](https://dev.mysql.com/worklog/task/?id=11250) 原文，是对上一节（源码依据的限制）的**官方口径补充**。

**来源**：官方摘要第一句就是 *"Porting Tecent's patch on Instant Add Column."*（官方原文把 Tencent 拼成了 Tecent）⇒ **Instant Add Column 源自腾讯的贡献补丁**，不是 Oracle 原创。

**官方列的三条限制**（*Some restrictions of this approach* 原文）：

1. *Columns can only be added last, it cannot be added in between existing columns*
2. *Instantly added columns cannot be PK, unless a table rebuild*
3. *Instant add column does not and will not support compressed table format* —— ⚠️ **"will not"（将来也不）**，这是官方明确的设计取舍，不是待办

**其余功能需求里的限制**：

| FR | 内容 |
|---|---|
| FR10 | **有全文索引的表不支持** |
| FR11 | **临时表总是走 `ALGORITHM=COPY`**，不在本 WL 范围 |
| FR12 | **DD 表空间里的表不支持** |
| FR13 | rebuild 后记录格式统一为最新：包括所有需 rebuild 的 ALTER（如 DROP COLUMN）、**TRUNCATE TABLE**、新建/截断分区、OPTIMIZE TABLE |
| FR14 | **EXCHANGE PARTITION**：分区或表任一方是 instant ⇒ 直接拒绝，报 `ER_PARTITION_EXCHANGE_DIFFERENT_OPTION` |
| FR15 | 需要"检查数据"的操作不能 INSTANT（如新增定义为 NOT NULL 的 geometry 列） |
| FR1 | `ALGORITHM=INSTANT` **不能与 `LOCK=NONE/SHARED/EXCLUSIVE` 同用**（报 `ER_WRONG_USAGE`），`LOCK=DEFAULT` 可以 |

**★ FR1 的源码落点**（`sql_table.cc:16370`）——注意 parser **不校验**，校验只在这里：

```cpp
  // LOCK clause doesn't make any sense for ALGORITHM=INSTANT.
  if (alter_info->requested_algorithm ==
          Alter_info::ALTER_TABLE_ALGORITHM_INSTANT &&
      alter_info->requested_lock != Alter_info::ALTER_TABLE_LOCK_DEFAULT) {
    my_error(ER_WRONG_USAGE, MYF(0), "ALGORITHM=INSTANT",
             "LOCK=NONE/SHARED/EXCLUSIVE");
    return true;
  }
```

> ⚠️ **`parse_tree_nodes.cc` 只做赋值**（`requested_algorithm = algo;`），任何互斥校验都不做。

**★ INSTANT 的 MDL 语义：保持开表时的 SU，commit 前才升 X**（`handler.h:6068` 契约原文）：

```cpp
       For HA_ALTER_INPLACE_INSTANT we keep shared upgradable metadata lock
       which was acquired at table open time.
    ...
    *) We ensure that no other connection uses the table by upgrading our
       lock on it to exclusive.
    *) a) ... handler::ha_commit_inplace_alter_table() is called ...
       For instant algorithm this is the step during which SE changes are done.
```

对应 server 侧 swtich（`sql_table.cc:13528`）——**INSTANT 分支什么都不做，`ha_alter_info->online` 保持 false**：

```cpp
    case HA_ALTER_INPLACE_EXCLUSIVE_LOCK:
    case HA_ALTER_INPLACE_SHARED_LOCK_AFTER_PREPARE:
    case HA_ALTER_INPLACE_SHARED_LOCK:
    case HA_ALTER_INPLACE_INSTANT:      // ★ 不置 online
      break;
```

**★ prepare 对 INSTANT 完全 no-op**（`handler0alter.cc:5477`）：

```cpp
  if (is_instant(ha_alter_info)) {
    Instant_Type type = innobase_support_instant(ha_alter_info, indexed_table,
                                                 this->table, altered_table);
    if (type == Instant_Type::INSTANT_ADD_DROP_COLUMN) {
      ut_a(is_valid_row_version(indexed_table->current_row_version + 1));
    }                                     // ★ 只做一个断言
    return false;                         // ★★ 不开 trx、不建 ctx、不建 row log
  }
```

```cpp
  if (!(ha_alter_info->handler_flags & ~INNOBASE_INPLACE_IGNORE) ||
      is_instant(ha_alter_info)) {        // commit 侧同样短路
    assert(!ctx0);                        // ★ INSTANT 一定没有 inplace ctx
    MONITOR_ATOMIC_DEC(MONITOR_PENDING_ALTER_TABLE);
    ha_alter_info->group_commit_ctx = nullptr;
    return false;                         // ★ 不 X 锁表、不起 trx
  }
```

**★ `Instant_ddl_impl` 是栈上对象，不是 new**（`handler0alter.cc:1611`）：

```cpp
  if (is_instant(ha_alter_info)) {
    ut_ad(!res);
    Instant_ddl_impl<dd::Table> executor(      // ★ 栈上构造
        ha_alter_info, m_user_thd, m_prebuilt->trx, m_prebuilt->table, table,
        altered_table, old_dd_tab, new_dd_tab,
        altered_table->found_next_number_field != nullptr
            ? &m_prebuilt->table->autoinc : nullptr);
    if (executor.commit_instant_ddl()) return true;   // ★★ 唯一执行入口
  }
```

**★★ 显式 `ALGORITHM=INSTANT` 是 fail-fast 而非静默降级**（`sql_table.cc:17454`）：

```cpp
    if (alter_info->requested_algorithm ==
            Alter_info::ALTER_TABLE_ALGORITHM_INSTANT &&
        inplace_supported != HA_ALTER_INPLACE_INSTANT &&
        inplace_supported != HA_ALTER_ERROR) {
      ha_alter_info.report_unsupported_error("ALGORITHM=INSTANT",
                                             "ALGORITHM=COPY/INPLACE");
      close_temporary_table(thd, altered_table, true, false);
      goto err_new_table_cleanup;
    }
```

> ★ `HA_ALTER_ERROR` 是被排除的——引擎已自己 `my_error()` 过，不能再覆盖。
> ★ 引擎侧的四道降级全是同一模式：`ALGORITHM=INSTANT` → `return HA_ALTER_ERROR`（报错）；没写 → `break`（**静默降级 INPLACE**）：字段数、行版本、行大小、`error_if_not_empty`。
> ⚠️ 还有一条：**显式 `ALGORITHM=INPLACE` 时也不给 INSTANT**（`handler0alter.cc:1038`，契约见 `handler.h:6168`）。

#### ★ 坑一：UPDATE 后再 ROLLBACK，行可能变大到无法回滚

官方第 7 节给的示例（`row_format=compact`、4K 页）：

```
create table t2(id int primary key, c1 varchar(4000), c2 varchar(4000), c3 varchar(1000));
insert into t2 values(1, repeat('a',4000), repeat('b',4000), repeat('c',1));
alter table t2 add column d1 varchar(500) not null default repeat('d',500);
begin;
update t2 set c1 = repeat('x',200) where id=1;   -- 或 set c4='x'
rollback;
```

原行**没有**存储那 500 字节默认值（instant 不改动已有记录）。但 **ROLLBACK 时要把默认值补回原行** ⇒ 行突然变大，可能放不进一页、部分字段要转外部存储；**REDUNDANT + 4K 页时甚至会因行过大导致回滚失败并崩溃**。

官方因此要求：*it needs to keep the default values as not stored in record when possible* —— 即更新后的记录**也**不存默认值，只用 `REC_OFFS_DEFAULT`（`1 << 29`）在 offsets 数组里标记，读时再从 `dict_col_t::instant_default` 取。

⇒ 连带规则：**只要 instant 加的列被更新，就必须走 delete + insert，不能 inplace update**。

**★ `REC_OFFS_DEFAULT` 的完整取值链条**（这是"零重写"读路径的核心）：

```cpp
// ① rem/rec.h:469 —— 打标记：默认值为常量 → DEFAULT 位；为 NULL → SQL_NULL 位
static inline uint64_t rec_get_instant_offset(const dict_index_t *index,
                                              ulint n, uint64_t offs) {
  ut_ad(index->has_instant_cols_or_row_versions());
  ulint length;
  index->get_nth_default(n, &length);
  if (length == UNIV_SQL_NULL) {
    return (offs | REC_OFFS_SQL_NULL);
  } else {
    return (offs | REC_OFFS_DEFAULT);      // ★
  }
}
```

```cpp
// ② rem0lrec.h:134 —— 长度字段变魔法值
  if (length & REC_OFFS_SQL_NULL) {
    length = UNIV_SQL_NULL;
  } else if (length & REC_OFFS_DEFAULT) {
    length = UNIV_SQL_ADD_COL_DEFAULT;     // ★★ 魔数，不是真实字节数
  } else if (length & REC_OFFS_DROP) {
    length = UNIV_SQL_INSTANT_DROP_COL;
  } else {
    length &= REC_OFFS_MASK;
    length -= offs;
  }
```

```cpp
// ③ rem0rec.ic:1006 —— COMPACT 取默认值（★ 最终回落到 col->instant_default）
static inline const byte *rec_get_nth_field_instant(const rec_t *rec,
                                                    const ulint *offsets,
                                                    ulint n,
                                                    const dict_index_t *index,
                                                    ulint *len) {
  if (index->has_row_versions()) {
    /* Fields are stored in record in order of the version they are added in.
    Get the field number on physical record. */
    n = index->get_field_off_pos(n);       // ★ 逻辑位 → 物理位（fields_array）
  }
  ulint off = rec_get_nth_field_offs(nullptr, offsets, n, len);
  if (*len != UNIV_SQL_ADD_COL_DEFAULT && *len != UNIV_SQL_INSTANT_DROP_COL) {
    return (rec + off);                    // 真实数据就在记录里
  }
  if (*len == UNIV_SQL_INSTANT_DROP_COL) {
    *len = 0;
    return nullptr;                        // ★ instant DROP 掉的列 → 空
  }
  return (index->get_nth_default(n, len)); // ★★ 老行：回落到默认值
}
```

```cpp
// ④ dict0mem.h:1595 —— 默认值的唯一来源
  const byte *get_nth_default(ulint nth, ulint *length) const {
    ut_ad(nth < get_n_total_fields());
    const dict_col_t *col = get_physical_field(nth)->col;   // ★ 物理位取 field
    if (col->instant_default == nullptr) { *length = 0; return (nullptr); }
    *length = col->instant_default->len;
    return (col->instant_default->value);
  }
```

```cpp
// ⑤ dict0mem.h:475 —— 承载结构
struct dict_col_default_t {
  dict_col_t *col;      /* Pointer to the column itself */
  byte *value;          /* Default value in bytes */
  size_t len;           /* Length of default value；可为 UNIV_SQL_NULL */
  bool operator==(const dict_col_default_t &other);
  bool operator!=(const dict_col_default_t &other);
};
```

> ⚠️ **REDUNDANT 走另一条路**（`rec_get_nth_field_old_instant`，`rem0rec.ic:969`）：先读 `row_version`，再 `index->get_field_phy_pos(n, version)` 换算物理位，然后 `if (n < rec_get_n_fields_old_raw(rec))` 才取记录内数据，否则回落默认值。

**★ `is_store_version()` 揭示了 UPDATE 的特殊性**（`rem0rec.cc:246`）：

```cpp
bool is_store_version(const dict_index_t *index, size_t n_tuple_fields) {
  if (!index->has_instant_cols_or_row_versions()) return false;
  ut_ad(index->table->has_row_versions() || index->table->is_upgraded_instant());
  ut_ad(n_tuple_fields > 0);
  /* From an UPDATE when not materializing INSTANT defaults */
  if (n_tuple_fields < index->n_fields) {
    //                       ★ UPDATE 只给部分列 → 不记 version，退回 V1 记列数
    ut_ad(index->has_instant_cols());
    return false;
  }
  return true;
}
```

> ★★ 所以：**完整行的 INSERT 写 1 字节 version；部分列的 UPDATE 写 1–2 字节列数**。这是 V1 与 V2 在 8.0.39 里**并存**的直接证据——不是"升级后就只用 V2"。

#### ★ 坑二：`.cfg` 缺失时 IMPORT 会失败

列数（`instant_col`）存在 `dd::Table::se_private_data`。若 `.cfg` 文件缺失，就没有"第一次 instant ADD COLUMN 前有多少列"这个信息 ⇒ 官方要求 IMPORT 后**校验聚簇索引**，而 instant 表的记录会被判定为 corrupted，**IMPORT 直接失败**（官方说明要等 WL#9761 的 SDI EXPORT/IMPORT 到位才能根治）。

**另两个实现要点**：

- **redo**：新风格记录（COMPACT/DYNAMIC）记 redo 时必须带上"第一次 instant ADD COLUMN 前的列数"，否则恢复时无法解析；旧风格（REDUNDANT）记录本身就有字段数，**不需要额外日志**
- **I_S 展示**：`INNODB_TABLES.INSTANT_COLS`、`INNODB_COLUMNS.HAS_DEFAULT` / `DEFAULT_VALUE`

**★ 坑二的真实机制是"四象限分派"**（`row0import.cc:1823`）——不只是"`.cfg` 缺失就失败"：

```cpp
dberr_t row_import::set_instant_info_v2(THD *thd, const dd::Table *dd_table) {
  bool src_has_row_versions = has_row_versions();      // 来自 .cfg
  bool dst_has_row_versions = m_table->has_row_versions();  // 来自 DD

  /* None of the table has INSTANT columns. Return success. */
  if (!src_has_row_versions && !dst_has_row_versions) return (DB_SUCCESS);

  /* Only target table has INSTANT columns, ERROR */
  if (!src_has_row_versions && dst_has_row_versions) {
    ib_errf(thd, IB_LOG_LEVEL_ERROR, ER_TABLE_SCHEMA_MISMATCH,
            "The .cfg file indicates no INSTANT column in the source table"
            " whereas the metadata in data dictionary says there are instant"
            " columns in the target table");
    return (DB_ERROR);
  }

  /* ★ 只有源表有 → 用 .cfg 去"补齐"目标表的 instant 元数据 */
  if (src_has_row_versions && !dst_has_row_versions) {
    return (adjust_instant_metadata_in_taregt_table(thd, dd_table));
  }

  /* 两边都有 → 必须逐项匹配 */
  if (src_has_row_versions && dst_has_row_versions) {
    return (match_instant_metadata_in_target_table(thd));
  }
```

```cpp
  /** Check if table being imported has INSTANT ADD/DROP columns
  @return true if table has INSTANT ADD/DROP columns */
  bool has_row_versions() {
    return (m_total_column_count > m_initial_column_count ||
            m_total_column_count > m_current_column_count);
  }
```

> ★★ **`has_row_versions()` 的判据是"总列数 > 初始列数 或 > 当前列数"**——`total` 含被 instant drop 的列（见 `instant_update_table_cols_count` 里 `total_col_count` 只增不减）。

**匹配要逐列比对 6 项**（`row0import.cc:1639`）：

```cpp
    if (!cfg_col->is_version_added_match(target_col) ||
        !cfg_col->is_version_dropped_match(target_col) ||
        (cfg_col->ind != target_col->ind) ||
        (cfg_col->get_phy_pos() != target_col->get_phy_pos())) {
      ib_errf(thd, IB_LOG_LEVEL_ERROR, ER_TABLE_SCHEMA_MISMATCH,
              "Instant metadata didn't match for column %s", col_name);
      return (DB_ERROR);
    }
    ...
    /* If instant_default values are different, error */
    if (*target_col->instant_default != *cfg_col->instant_default) {
      ib_errf(thd, IB_LOG_LEVEL_ERROR, ER_TABLE_SCHEMA_MISMATCH,
              "The metadata in the data dictionary and the .cfg file contain"
              " different default values for column %s!", col_name);
      return (DB_ERROR);
    }
```

```cpp
bool dd_match_default_value(const dd::Column *dd_col, const dict_col_t *col) {
  ut_ad(col->instant_default != nullptr);
  const dd::Properties &private_data = dd_col->se_private_data();
  if (private_data.exists(dd_column_key_strings[DD_INSTANT_COLUMN_DEFAULT])) {
    dd::String_type value;
    const byte *default_value;
    DD_instant_col_val_coder coder;
    private_data.get(dd_column_key_strings[DD_INSTANT_COLUMN_DEFAULT], &value);
    default_value = coder.decode(value.c_str(), value.length(), &len);
    match = col->instant_default->len == len &&
            memcmp(col->instant_default->value, default_value, len) == 0;
    return match;
  } else if (private_data.exists(
                 dd_column_key_strings[DD_INSTANT_COLUMN_DEFAULT_NULL])) {
    return (col->instant_default->len == UNIV_SQL_NULL);
  }
  return true;     // ★ 服务端没记默认值 → 一律放行，后续可补
}
```

> ★ 分区表还有一道额外校验 `match_col_default_values()`（不同分区可能需要的默认值个数不同，源码注释里给了 p1 需 10 个 / p2 需 2 个的例子）。

**⚠️ 勘误：instant 没有专属 redo 类型**——`MLOG_REC_*` 枚举里**没有任何含 INSTANT 的项**。instant 元数据的持久化靠三条：

1. 记录内 1 字节（`rec_set_instant_row_version_new/old` 或 `rec_set_n_fields`）+ info bits 两位；
2. **online DDL 的 row log**：序列化成 TEMP 格式时额外写 1~2 字节（`row0log.cc:798` 注释原文 *"Write instant metadata in 1 or 2 bytes"*）；
3. "首次 instant ADD 前的列数"持久化在 **DD 的 `se_private_data["instant_col"]`**（`DD_TABLE_INSTANT_COLS`），不在 redo。

```cpp
    /* Write instant metadata in 1 or 2 bytes */
    if (index->has_instant_cols_or_row_versions()) {
      byte *temp_rec = b + extra_size;
      rec_set_info_bits_new_temp(temp_rec, rec_get_info_bits(rec, false));
      if (rec_has_version) {
        rec_new_temp_set_versioned(temp_rec, true);
        byte *temp_version = temp_rec - 2;
        memcpy(temp_version, &rec_version, sizeof(uint8_t));  // ★ 版本写在 info-bits 前一字节
      } else {
        rec_new_temp_set_versioned(temp_rec, false);
      }
    }
```

---

## 可观测性

### 如何用 SQL 查询 instant 状态

**判断一张表是否做过 instant DDL**：查 `information_schema.INNODB_TABLES` 的 `TOTAL_ROW_VERSIONS`（它就是 `current_row_version`）：

```sql
SELECT NAME, N_COLS, INSTANT_COLS, TOTAL_ROW_VERSIONS
FROM   information_schema.INNODB_TABLES
WHERE  NAME = 'your_db/your_table';
```

- **`TOTAL_ROW_VERSIONS > 0`** → 做过 instant ADD/DROP（值 = 累计次数，每次 instant add/drop +1，`OPTIMIZE TABLE`/重建表后重置为 0）；
- **`INSTANT_COLS`** → instant 列数，但**有坑**：只有从 8.0.28 及更早升级上来的表（V1 格式，`is_upgraded_instant()`）才有值，8.0.29+ 的 V2 表恒为 0（源码 `table->is_upgraded_instant() ? table->get_instant_cols() : 0`）。**所以判断"做过没有"要看 `TOTAL_ROW_VERSIONS`，不要看 `INSTANT_COLS`**。

**看哪些列是 instant 加的**：

```sql
SELECT t.NAME, c.NAME AS col, c.HAS_DEFAULT, c.DEFAULT_VALUE
FROM   information_schema.INNODB_TABLES  t
JOIN   information_schema.INNODB_COLUMNS c ON c.TABLE_ID = t.TABLE_ID
WHERE  t.NAME = 'your_db/your_table'
  AND  c.HAS_DEFAULT = 1;      -- 有 instant 默认值 → instant ADD 的列
```

`HAS_DEFAULT=1` 表示该列带 instant 默认值（`column->instant_default != nullptr`），即 instant ADD 的列；instant DROP 的列默认**不显示**（源码里只有 `DBUG show_dropped_column` 调试开关才露出）。

**★ I_S 表定义与填充代码**（⚠️ 不在 `dict0dd.cc`，在 `storage/innobase/handler/i_s.cc`）：

```cpp
#define INNODB_TABLES_INSTANT_COLS 8
    {STRUCT_FLD(field_name, "INSTANT_COLS"),
     STRUCT_FLD(field_length, MY_INT32_NUM_DECIMAL_DIGITS),
     STRUCT_FLD(field_type, MYSQL_TYPE_LONG), ...},

#define INNODB_TABLES_TOTAL_ROW_VERSIONS 9
    {STRUCT_FLD(field_name, "TOTAL_ROW_VERSIONS"),
     STRUCT_FLD(field_length, MY_INT32_NUM_DECIMAL_DIGITS),
     STRUCT_FLD(field_type, MYSQL_TYPE_LONG), ...},
```

```cpp
  OK(fields[INNODB_TABLES_INSTANT_COLS]->store(
      table->is_upgraded_instant() ? table->get_instant_cols() : 0));   // ★★ V2 恒 0

  OK(fields[INNODB_TABLES_TOTAL_ROW_VERSIONS]->store(
      table->current_row_version));                                     // ★ V2 次数
```

```cpp
  if (column->instant_default != nullptr) {       // ★ HAS_DEFAULT 完全由它决定
    OK(fields[SYS_COLUMN_HAS_DEFAULT]->store(1));
    OK(field_blob_store(fields[SYS_COLUMN_DEFAULT_VALUE],
                        column->instant_default));
  } else {
    OK(fields[SYS_COLUMN_HAS_DEFAULT]->store(0));
```

> ★ `DEFAULT_VALUE` 存的是 **`DD_instant_col_val_coder::encode()` 后的 hex 字符串**（`field_blob_store()` 里现算），长度上限 `65536 * 4`。
> ★ **`UNIV_DEBUG` 构建下还有 5 个额外列**：`INNODB_TABLES` 的 `INITIAL_COLUMN_COUNTS` / `CURRENT_COLUMN_COUNTS` / `TOTAL_COLUMN_COUNTS`，`INNODB_COLUMNS` 的 `VERSION_ADDED` / `VERSION_DROPPED` / `PHYSICAL_POS`。**生产版本看不到**——排障时值得知道它们的存在。

**★ 三个列计数器的语义**（`dict0mem.h:2150`）：

```cpp
  /** Initial non-virtual column count */
  uint32_t initial_col_count{0};
  /** Current non-virtual column count */
  uint32_t current_col_count{0};
  /** Total non-virtual column count */
  uint32_t total_col_count{0};
```

```cpp
static void instant_update_table_cols_count(dict_table_t *dict_table,
                                            uint32_t n_added_column,
                                            uint32_t n_dropped_column) {
  dict_table->current_col_count += n_added_column;
  dict_table->current_col_count -= n_dropped_column;
  dict_table->total_col_count += n_added_column;   // ★★ total 只增不减 → 被删列永久占位
  ut_ad(dict_table->total_col_count >= dict_table->current_col_count);
}
```

| 派生量 | 公式 |
|---|---|
| instant ADD 的列数 | `total_col_count - initial_col_count` |
| instant DROP 的列数 | `total_col_count - current_col_count` |
| `has_row_versions()` | `current_row_version > 0` |

> ★★ **`total_col_count` 只增不减**是"反复 instant 加删列会让表定义越来越臃肿"的量化证据——每次 ADD 都让 `total` 涨，DROP 只是让它与 `current` 拉开差距，空间永不回收（直到 rebuild 表）。

### 观测对象 → 手段 速查

| 我想看 | 手段 | 说明 |
|--------|------|------|
| 某表做过几次 instant 变更 | SQL | `INNODB_TABLES.TOTAL_ROW_VERSIONS`（重建表后归零） |
| 哪些列是 instant 加的 | SQL | `INNODB_COLUMNS.HAS_DEFAULT=1` |
| 这次 ALTER 走没走 INSTANT | SQL / 返回 | 显式 `ALGORITHM=INSTANT` 时不可行会报错（fail-fast）；默认模式下看 `TOTAL_ROW_VERSIONS` 是否 +1 |
| instant 列被 UPDATE 的代价 | 源码规则 | 该类 UPDATE 强制走 delete+insert（坑一），无法 inplace update |

---

## Misc

### 面向二次开发

**扩展点**：新增一种可 INSTANT 的变更类型要动 `Instant_Type` 枚举 + `commit_instant_ddl()` 分支 + `check_if_supported_inplace_alter` 的判定；若涉及新的列元信息，需在 `se_private_data` 加 key 并在加载路径（dict0dd.cc）解析。

### 坑与已知缺陷

| 坑 | 说明 |
|---|---|
| `INSTANT_COLS` 对 8.0.29+ 表恒为 0 | 该列仅对 V1（升级上来的）表有意义，判断请用 `TOTAL_ROW_VERSIONS` |
| instant 列的 UPDATE 不能 inplace | 必须 delete+insert（坑一），写入放大 |
| 版本不是无限的 | `row_version` 1 字节；且每次 instant 都增加一层解析历史，FR13 说明 rebuild/TRUNCATE/OPTIMIZE 会归零 |
| IMPORT 依赖 `.cfg` | 缺失即失败（坑二） |
| 压缩表永不 INSTANT | 官方原话 "will not"，是设计取舍 |

### 易混淆概念

| 概念 | 辨析 |
|---|---|
| `dict_table_t::version` vs `current_row_version` | 前者表定义版本（任何 DDL +1），后者行版本（仅 instant +1），独立字段 |
| V1 `n_fields` vs V2 `row_version` | 都写在记录头前同一位置（`rec-6`），靠 info bits 的两个标志位区分；V1 仅支持加列，V2 支持加/删列 |
| INSTANT vs "只改元数据"的其他操作 | INSTANT 特指靠**行版本号**让老行保持原样的加/删列；普通的元数据级 ALTER（如改注释）不产生版本号 |
| `records` vs `offsets` | 物理记录里**没有** default 标记；`REC_OFFS_DEFAULT` / `REC_OFFS_DROP` 只活在内存的 `offsets[]` 数组里 |
| `REC_OFFS_DEFAULT` vs `REC_OFFS_SQL_NULL` | instant 列的默认值**本身就是 NULL** 时打的是 `REC_OFFS_SQL_NULL`，不是 `REC_OFFS_DEFAULT`（见 `rec_get_instant_offset`） |

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `include/rec0rec.h` | **不存在**（0 命中）→ `rem/rec.h` + `rem/rem0rec.cc` + `include/rem0rec.{h,ic}` + `include/rem0lrec.h` + `rem/rem0wrec.cc` |
| `storage/innobase/rem/record.cc` | **不存在** → `rem/rec.cc`（只放 offsets 解析）/ `rem/rem0rec.cc`（放转换函数） |
| `storage/innobase/ddl/ddl0rec.cc` | **不存在**（ddl 目录只有 10 个 `ddl0*.cc`，无 rec） |
| `storage/innobase/ddl/ddl0dd.cc` | **不存在** → `storage/innobase/dict/dict0inst.{h,cc}`（"Instant DDL interface"） |
| `storage/innobase/dd/dd0dd.cc` | **不存在** → `storage/innobase/dict/dict0dd.{h,cc}` |
| `REC_OFFS_*` 在 `rem0rec.h` | 全在 **`rem/rec.h:44-55`**（`rem0rec.h` 里 0 命中） |
| `REC_OFFS_HEADER_SIZE` = 2 | **分编译模式**：`UNIV_DEBUG` 下 = **4**（多 2 槽存 rec/index 指针），否则 = 2 |
| `rec_offs_n_alloc()` | **不存在** → getter `rec_offs_get_n_alloc()`，setter 才是 `rec_offs_set_n_alloc()` |
| `rec_offs_any_default()` | **不存在**（0 命中）→ 只有 `rec_offs_nth_default(index, offsets, n)` |
| `rec_offs_nth_offset()` | **不存在** → `rec_get_nth_field_offs()` / `rec_get_nth_field_offs_low()` |
| `rec_init_offsets_temp()` | **不存在** → `rec_init_offsets_comp_ordinary(rec, /*temp=*/true, ...)` + `rec_init_null_and_len_temp()` |
| `rec_set_instant_row_version_new_temp()` | **不存在** → temp 侧只有 `rec_new_temp_set_versioned()` |
| `rec_is_alter_metadata_temp()` / `rec_set_alter_metadata_temp()` | **均不存在** |
| `rec_validate_rec()` | **不存在** → `rec_validate(rec, offsets)` + static 的 `rec_validate_old(rec)` |
| `rec_copy_prefix_to_dtuple_low()` | **不存在** → 只有 `rec_copy_prefix_to_dtuple()` |
| `row_rec_to_fields()` | **不存在**（0 命中）→ `row_sel_store_mysql_rec()` → `row_sel_store_mysql_field()` → `rec_get_nth_field_instant()` |
| `row_merge_insert_index_tuples()` | **不存在** → 只剩 `srv0srv.h:837` 一句过时注释；现为 `ddl::Builder`（`ddl0builder.cc`） |
| `struct instant_field_t` / `Instant_field` | **不存在** → 拆成 `dict_col_default_t`（默认值）+ `dict_col_t` 的 `version_added`/`version_dropped`/`phy_pos` |
| `dict_index_t::instant_fields`（vector） | **不存在** → 只有方法 `get_instant_fields()`，且是**算出来的** |
| `dict_index_t::n_instant_cols` | **不存在** → `n_instant_cols` 只在 `dict_table_t`；索引侧是 `n_instant_nullable` + 标志位 `instant_cols` |
| `versioned_fields` / `n_versioned_fields` | **不存在** → 只是 `mtr0log.cc` 里的局部变量名 |
| `row_versions` 是 `std::vector<uint32_t>` | **不是** → `dict_table_t::current_row_version`（uint32）+ `dict_index_t::nullables[65]`（uint32 数组）+ `fields_array`（vector<uint16_t>）三件套 |
| `dict_index_add_instant_field()` | **不存在** → `dict_index_t::add_field()` + `dict_index_add_col()` |
| `get_n_fields_in_version()` | **不存在**（0 命中）→ `is_visible_in_version(v)` 逐列判定 + `get_nullable_in_version(v)` |
| `dict_index_build_instant_fields()` / `_versioned_fields()` | **不存在** → `dict_index_build_internal_clust()` 里的 `create_nullables()` + `create_fields_array()` |
| `is_instant_add_possible()` / `is_instant_drop_possible()` | **不存在** → `Instant_ddl_impl<Table>::is_instant_add_drop_possible()`，且**只判行大小**不判语义 |
| `dd_set_instant_table()` | **不存在** → 只有反向的 `dd_clear_instant_table()` / `dd_clear_instant_part()` |
| `dict_table_get_n_instant_cols()` | **不存在** → `dict_table_t::get_instant_cols()`（注意返回值**减掉了系统列**） |
| `dict_table_t::n_mysql_handles_opened` | **不存在** → 改名 `std::atomic<uint64_t> n_ref_count` |
| `MAX_ROW_VERSION` = 63 | **64**（`dict0mem.h:323`），数组 `nullables[65]` |
| `SQL_NULL` / `ADD_COL_DEFAULT` / `INSTANT_DROP_COL` 的值 | `0xFFFFFFFF` / `0xFFFFFFFE` / `0xFFFFFFFB`（注意中间还夹着 `UNIV_NO_INDEX_VALUE` 与 `UNIV_MULTI_VALUE_ARRAY_MARKER`） |
| instant 定义在 `dict0dict.h` | 该文件对 "instant" **0 匹配**；全在 `dict0mem.h` |
| `dtuple_create_instant_fields()` / `dtuple_set_instant_defaults()` | **均不存在** → 默认值**不在写入侧填进 dtuple**，而是读侧由 `rec_get_nth_field_instant()` 零拷贝返回；唯一相关的 dtuple 方法是**反向**的 `ignore_trailing_default()` |
| `trx_undo_rec_get_row_version()` / `trx_undo_rec_set_row_version()` | **均不存在**——**undo 里根本不存 row version**，只把列号改成 `phy_pos` |
| `rec_get_default_value()` / `dict_col_get_default()` | **均不存在** → `dict_index_t::get_nth_default()` / `dict_col_t::set_default()` / 成员 `instant_default` |
| `rec_new_is_instant()` | **不存在** → `rec_get_instant_flag_new()` |
| `rec_init_null_and_len_old()` | **不存在** → REDUNDANT 走 `rec_init_offset_old_1byte()` / `_2byte()` |
| 一次 UPDATE 必然物化 instant 默认值 | **不是** → `materialize_instant_default()`：**V1 升级表的老行返回 false**（不物化，防 ROLLBACK 超 row_size_limit） |
| `is_store_version()` 只看表属性 | 还要看 `n_tuple_fields < index->n_fields`（UPDATE 未物化默认值 → 退回 V1） |
| 物理记录里有 default 标记 | **没有** → `REC_OFFS_DEFAULT` 只在读侧 offsets 数组里 OR 进去，唯一生产者是 `rec_get_instant_offset()` |
| `table->has_instant_cols()` 表示"这张表是 instant 表" | **只表示 V1**；V2 要用 `has_row_versions()`。判断"记录要不要按 instant 解析"只能用 `index->has_instant_cols_or_row_versions()`（且二级索引恒 false） |

**★ 四个"结构性事实"**

1. **`offsets[]` 才是 instant 的真正载体，物理记录只是"少写了几个列"**。`rec_get_offsets()` 对 COMPACT 恒返回 `dict_index_get_n_fields(index)`（**与行内实际列数无关**），缺的列靠 `REC_OFFS_DEFAULT` / `REC_OFFS_DROP` 在内存 offsets 里"虚拟补齐"。于是上层所有代码——比较、拷贝、回传 MySQL——**都不知道这行是老行**。这是零重写能成立的根本原因，也解释了为什么 `row_sel_store_mysql_field()` 里找不到任何 DEFAULT 分支。

2. **V1 与 V2 的分水岭不是"存什么"而是"能表达什么"**。两者都写在 `rec-6` 那一个字节（靠 info bits 的两个 bit 区分含义），但 V1 只有 `i >= non_default_fields` 这一个**线性**判据（只能表达"尾部少列"），V2 有 `is_added_after(v)` / `is_dropped_in_or_before(v)` 两个**逐列**判据（能表达任意位置缺列 + 任意位置有多余的已删列）。这就是为什么 V1 撑不起 DROP COLUMN，以及为什么 8.0.29 必须引入版本号。

3. **`ind`（逻辑位）与 `phy_pos`（物理位）的分离是 V2 支持 DROP 的根本**。列被 instant DROP 后 `phy_pos` **不变**（老行数据还在那个位置），`ind` 也不变；读取时靠 `fields_array` 双向翻译，REDUNDANT 里还要额外用 `dropped_col_count` 修正下标。三个原子判定压成一行：`is_visible_in_version(v) = !is_added_after(v) && !is_dropped_in_or_before(v)`。

4. **"坑一"的机制链条完整闭合**：V1 时代允许"加列后最大行超过 `row_size_limit`"，所以 `materialize_instant_default()` 对 **V1 升级表的老行返回 false** → UPDATE 不物化默认值 → `is_store_version()` 因 `n_tuple_fields < index->n_fields` 返回 false → 走 V1 写 `n_fields`；rollback 时 `ignore_trailing_default()` 从尾部裁掉等于默认值的字段，把 `n_fields` 减回 `get_instant_fields()`，还原成老行格式。**V2 不存在这个问题**，因为 `is_instant_add_drop_possible()` 在 DDL 阶段就拦住了超长行。

> 注：`rem/rec.h`、`rem/rem0rec.cc`、`include/dict0mem.h` 等文件中的部分 `// note:` / `// ques:` / `// 中文` 注释为本工作区人工批注，非 Oracle 上游原文。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → 17.12 InnoDB and Online DDL*（INSTANT 的官方定位与 8.0.12 引入说明）

**WorkLog**
- [WL#11250](https://dev.mysql.com/worklog/task/?id=11250) *Support Instant Add Column* —— 官方设计，含记录格式、`se_private_data` 键与"回滚时行变大"的坑；摘要首句为 *"Porting Tecent's patch on Instant Add Column"*（官方原文如此），即该特性源自腾讯贡献

**相关文档**
- 上层：server 侧 DDL 编排、算法选择策略、MDL 三段式、原子 DDL 见 [`ddl.md`](ddl.md)
- 另两条路：COPY 见 [`copy_ddl.md`](copy_ddl.md)、INPLACE 见 [`inplace_ddl.md`](inplace_ddl.md)
- 记录物理格式与 offsets 层的落地见 [`../../innodb/physical/record.md`](../../innodb/physical/record.md)
