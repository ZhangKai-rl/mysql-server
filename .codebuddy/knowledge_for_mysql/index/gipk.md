# GIPK（Generated Invisible Primary Key）深度解析

> 基于 MySQL 8.0.39 源码。GIPK 是 8.0.30 引入的机制：开启 `sql_generate_invisible_primary_key` 后，CREATE TABLE 若没有显式主键，服务器自动补一个隐藏的自增主键列 `my_row_id`。
>
> **边界**：本篇讲 GIPK 的完整生命周期——生成条件、物理形态、可见性、数据字典打标、binlog/复制、ALTER 限制。主键与索引的通用机制见 [`types.md`](types.md) 与 [`constraint.md`](constraint.md)；`sql_require_primary_key`（另一套主键策略）见下文「与 sql_require_primary_key 的关系」节的对照。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [可观测性](#可观测性)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

GIPK = **Generated Invisible Primary Key**。一句话：**服务器帮你生成一个 `bigint unsigned NOT NULL AUTO_INCREMENT INVISIBLE` 的列 `my_row_id`，并以它为 PRIMARY KEY**。

```sql
SET SESSION sql_generate_invisible_primary_key = ON;
CREATE TABLE t (f1 INT);
-- 实际等价于
CREATE TABLE t (
  my_row_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT INVISIBLE,
  f1 INT,
  PRIMARY KEY (my_row_id)
);
```

**★ 第一个必须破除的误解**：GIPK **不是**虚拟生成列。`generate_invisible_primary_key()` 里 `Create_field::init()` 的第 15 个实参（对应 `Value_generator *fld_gcol_info`）传的是 `nullptr`——**没有 `gcol_info`**，它是 8 字节真实落盘的 STORED 列。名字里的 "Generated" 指"服务器自动生成"，不是 SQL 的 `GENERATED ALWAYS AS (expr)`。

**★★ 第二个必须破除的误解**：`my_row_id` 默认是**可见**的。`show_gipk_in_create_table_and_information_schema` 的默认值为 **`true`(ON)**，即 `SHOW CREATE TABLE` 默认就会把它显示出来。想隐藏得显式设 OFF。这与多数人"GIPK 是隐藏的"的直觉相反。

### 用途

解决的核心问题：**无主键表在 Row-Based Replication 下的性能与正确性**。

- **复制性能**：从库 apply 一个 UPDATE/DELETE 的 row event 时，若表无主键也无唯一键，只能全表扫描定位行。GIPK 保证每张表都有主键，把 O(N) 降到 O(log N)。
- **复制正确性**：无主键表在主从之间行顺序可能不同，GIPK 给每行一个稳定标识。
- **云原生场景**：让"用户不必关心主键"与"系统必须有主键"两个诉求同时成立。

在链路中的位置：**纯 server 层特性**——只在 `mysql_create_table()` / `create_table_from_items()` 里往 `Alter_info` 塞一个 `Create_field` + 一个 `Key_spec`，之后走完全标准的建表路径。

```
CREATE TABLE (SQL)
  → mysql_create_table()                        sql/sql_table.cc
      → validate_and_generate_invisible_primary_key()    ★ 唯一的生成入口
          → validate_invisible_primary_key_generation()  （四条禁令）
          → generate_invisible_primary_key()             （造列 + 造主键）
      → 正常建表路径 ...
      → ★ 重写 binlog 语句（store_create_info + write_bin_log）
```

> ★ 存储引擎**完全无感**：整个 `storage/` 目录 grep `gipk|invisible_primary` **零命中**（唯一命中是 perfschema 的版本号文件，与逻辑无关）。InnoDB 只通过 `HTON_SUPPORTS_GENERATED_INVISIBLE_PK` 声明"我支持"，之后收到的就是一个普通主键。

### 版本演进

| 版本 | 变化 | WL |
|---|---|---|
| 8.0.30 | **GIPK 引入**：`sql_generate_invisible_primary_key` / `show_gipk_in_create_table_and_information_schema`；I_S 六个视图支持隐藏 GIPK | [WL#13784](https://dev.mysql.com/worklog/task/?id=13784) |
| 8.0.30 | binlog 侧：`Table_map_log_event::TM_GENERATED_INVISIBLE_PK_F` 标志位；GIPK 表建表语句重写 | 同 WL#13784 |
| 8.0.32+ | **主从 GIPK 不对等**的行复制：主库无 GIPK、从库有 GIPK（或反之）时列映射偏移处理 | [WL#14639](https://dev.mysql.com/worklog/task/?id=14639) |
| 8.0.32+ | `REQUIRE_TABLE_PRIMARY_KEY_CHECK = GENERATE` 可按通道设置（从库自行生成 GIPK） | [WL#15419](https://dev.mysql.com/worklog/task/?id=15419) |

> ★ **WL#13784 的标题就是 GIPK 的官方定义**：*"Create auto_increment PK on table creation when no PK is specified"*。
>
> ⚠️ **`sql/sql_gipk.cc` 里没有任何 `WL#` 或 `8.0.30` 注释**（grep 零命中）——版本号与 WL 编号来自 `sql/dd/info_schema/metadata.h` 与 MTR 用例，不要写成"sql_gipk.cc 里写了 WL#13784"。

**★ 8.0.30 的源码实证**（★ 五条独立证据）：

```cpp
/* sql/rpl_utility.cc:1431 —— ★ 唯一的版本门槛判定，写死 80030 */
bool is_immediate_server_gipk_ready(THD &thd) {
  return thd.variables.immediate_server_version != UNDEFINED_SERVER_VERSION &&
         thd.variables.immediate_server_version >= 80030;
}
```
```cpp
/* sql/dd/info_schema/metadata.h:242 —— I_S 版本号锚定 80030 */
static const uint IS_DD_VERSION = 80030;
```
```cpp
/* sql/log_event.cc:11521 —— mysqlbinlog 只对 >=80030 的 source 打印 GIPK 标志 */
    if (print_event_info->immediate_server_version != UNDEFINED_SERVER_VERSION &&
        print_event_info->immediate_server_version >= 80030)
      my_b_printf(&print_event_info->head_cache,
                  "# has_generated_invisible_primary_key=%d\n",
                  has_generated_invisible_primary_key());
```

---

## 理论基础

### 设计思想与权衡

**核心命题：GIPK 是一次"服务器替用户做决定"，因此它必须在"侵入性"上极度克制。**

具体体现在四个决策上，每一个都有明确的"本来可以那样做，但没有"：

**决策 1：造一个真实列，而不是在引擎里加隐式 rowid。**

本来可以像 InnoDB 那样用一个内部 rowid（不占列名空间、不进 DD）。但那样：① 用户无法引用，`SELECT my_row_id` 做不到；② 无法被外键引用；③ 主从之间不可见，复制无法对齐。**代价**：占 8 字节/行，且占用列数配额。

**决策 2：用 `HT_HIDDEN_USER`（INVISIBLE）而不是 `HT_HIDDEN_SQL`（函数索引那种完全隐藏）。**

```cpp
/* sql/dd/types/column.h:95 —— 四种 hidden 类型 */
  enum class enum_hidden_type {
    /// The column is visible (a normal column)
    HT_VISIBLE = 1,
    /// The column is completely invisible to the server
    HT_HIDDEN_SE = 2,
    /// The column is visible to the server, but hidden from the user.
    /// This is used for i.e. implementing functional indexes.
    HT_HIDDEN_SQL = 3,
    /// User table column marked as INVISIBLE by using the column visibility
    /// attribute. Column is hidden from the user unless it is explicitly
    /// referenced in the statement. Column is visible to the server.
    HT_HIDDEN_USER = 4
  };
```
> ★★ 差别就在这句注释：**`HT_HIDDEN_USER` 的列 `hidden from the user unless it is explicitly referenced`**——`SELECT *` 不返回，但 `SELECT my_row_id` 可以。`HT_HIDDEN_SQL`（函数索引）是 SQL 层完全不可见。GIPK 选了前者，因为它是"用户的表"的一部分。

**决策 3：不传播变量，而是重写 binlog 语句。**（★ 这是最重要的一条，源码注释逐字说明）

```cpp
      /*
        If primary key is generated for a table then we create version of CREATE
        TABLE statement which includes generated key and invisible column
        definitions in explicit form by calling store_create_info(). This is
        necessary to correctly binlog/replicate such statements, as we don't
        write to binary log value of @@sql_generate_invisible_primary_key
        variable, but rely on logging what really has been done instead.
      */
```
> ★★ 为什么：`sql_generate_invisible_primary_key` 注册为 `NOT_IN_BINLOG`。若只记用户原句，从库（可能 OFF）就造不出这个列 ⇒ 主从表结构分叉。所以**把"实际做了什么"完整写进 binlog**。
> ★ 代价：建表时要额外 `open_table()` + `store_create_info()` 生成一份完整 DDL——**DDL 路径上多一次开表**。

**决策 4：对系统线程无条件关闭。**

```cpp
bool is_generate_invisible_primary_key_mode_active(THD *thd) {
  return (thd->variables.sql_generate_invisible_primary_key &&
          !thd->is_dd_system_thread() && !thd->is_initialize_system_thread());
}
```
> ★ bootstrap / `--initialize` 时不能给 `mysql.*` 系统表塞主键——它们的结构是硬编码的，多一列就会崩。

**★ 代价要落到具体后果**：

1. **8 字节/行的空间开销**，且因为它是主键，InnoDB 下**成为聚簇索引**——直接影响全表扫描性能与二级索引体积。
2. **占用列数配额**（与 `CREATE TABLE` 的列数上限共享）。
3. **`CREATE TABLE ... SELECT` 在 SBR 下被禁**：自增值的分配顺序在两端不确定，MIXED 下会被强制降级为 ROW。
4. **分区表完全不支持**（报 `ER_NOT_SUPPORTED_YET`），因为分区键必须是主键的一部分，而 GIPK 主键是单列自增。
5. **DDL 多一次开表**（binlog 重写）。

### 理论溯源

GIPK 没有对应的学术论文——它是**工程约束驱动的产物**。可追溯的三条脉络：

1. **RBR 的行定位问题**：无主键表在从库 apply row event 时只能全表扫描。这是 GIPK 存在的根本动机，在 `sql_require_primary_key` 的错误文案里被明确写了出来（见「与 sql_require_primary_key 的关系」）。
2. **INVISIBLE 列**（8.0.23，WL#10905）：GIPK 复用了这套 `HT_HIDDEN_USER` 基础设施。没有 INVISIBLE 列，GIPK 就得自己造一套"部分隐藏"机制。
3. **DD options 机制**：GIPK 用 `dd::Table/Index/Column::options()` 里的 `"gipk"` 键做标记，这是 DD 通用的键值扩展点（有白名单校验）。

> ★ 源码落点：`dd::Column::enum_hidden_type` 的 `HT_HIDDEN_USER` 注释明确写着 "User table column marked as INVISIBLE by using the column visibility attribute" —— GIPK 就是把它当成基础设施用。

### 算法与数据结构

**GIPK 没有任何算法**——它是教科书意义的"简单机制"。值得说的只有**判定逻辑**：

```cpp
/* sql/sql_gipk.cc:443 —— ★ 识别 GIPK 列的六个条件（合取） */
static bool is_generated_invisible_primary_key_column(Field *field) {
  return ((field->field_index() == 0) &&                    /* ① 必须是第 0 列 */
          is_generated_invisible_primary_key_column_name(field->field_name) &&  /* ② 名叫 my_row_id（大小写不敏感） */
          field->real_type() == MYSQL_TYPE_LONGLONG &&      /* ③ BIGINT */
          field->is_unsigned() &&                           /* ④ UNSIGNED */
          field->is_flag_set(NOT_NULL_FLAG) &&              /* ⑤ NOT NULL */
          (field->auto_flags & Field::NEXT_NUMBER) &&       /* ⑥ AUTO_INCREMENT */
          field->is_hidden_by_user());                      /* ⑦ INVISIBLE */
}
```
> ★★ **这是"结构推断"式判定，不是"标记读取"**。含义很微妙：**用户手写一模一样的列定义，也会被认成 GIPK**，从而套上全部 ALTER 限制。MTR 用例明确验证了这个行为。
> ★ 反过来也一样：`ALTER TABLE t ALTER my_row_id SET VISIBLE` 之后，条件 ⑦ 不成立 ⇒ **这一列不再是 GIPK**，所有限制随之解除。这是"改可见性"成为唯一被允许的 ALTER 的原因。

数据结构上，GIPK 只往两个既有容器里塞东西：

| 容器 | 塞入 | 代码 |
|---|---|---|
| `alter_info->create_list` | `Create_field`（`push_front`，永在第 0 位） | `sql/sql_gipk.cc:167` |
| `alter_info->key_list` | `Key_spec(KEYTYPE_PRIMARY, ...)` | `sql/sql_gipk.cc:176` |

### 他库对比与演进动机

| 数据库 | 无主键表怎么处理 | 与 MySQL 的差异 |
|---|---|---|
| **PostgreSQL** | 无隐式主键；表本来就是 heap，行由 `ctid` 定位 | 不需要 GIPK——它没有"复制靠主键定位"这个约束 |
| **Oracle** | 有 `ROWID` 伪列（物理地址） | 同样没有"必须有主键"的复制需求 |
| **MySQL** | 8.0.30 起 GIPK；8.0 之前只有 InnoDB 的内部 6 字节 rowid（用户不可见、不进 DD、不能被复制利用） | **GIPK 的本质差异**：它是**一个真实的、SQL 可见的、可复制的列**，不是内部物理地址 |
| **MariaDB** | 无对应特性（`sql_generate_invisible_primary_key` 不存在） | ⚠️ 属 MySQL 私有特性 |

> ★ 关键洞察：MySQL **必须**有 GIPK，是因为它的复制是"逻辑复制 + 行定位靠索引"。PG/Oracle 没有这个约束，所以它们不需要。GIPK 是**复制架构的产物**，不是存储架构的产物。

---

## 核心实现

### 主链路

```
CREATE TABLE t (...)
  │
  ├─ mysql_create_table()                              sql/sql_table.cc
  │    │
  │    ├─【候选判定】is_candidate_table_for_invisible_primary_key_generation()
  │    │     ├─ 引擎支持？HTON_SUPPORTS_GENERATED_INVISIBLE_PK  ← 只有 InnoDB
  │    │     └─ 已有显式 PRIMARY KEY？→ 直接 false（静默不生成）
  │    │
  │    ├─【模式判定】is_generate_invisible_primary_key_mode_active()
  │    │     └─ 变量 ON 且 非 DD/initialize 系统线程
  │    │
  │    ├─【校验+生成】validate_and_generate_invisible_primary_key()
  │    │     ├─ validate_invisible_primary_key_generation()   四条禁令
  │    │     └─ generate_invisible_primary_key()              造列 + 造主键
  │    │
  │    ├─ 正常建表路径（走标准流程）
  │    │
  │    └─【binlog 重写】if (is_pk_generated)
  │          ├─ open_table() / open_temporary_table()
  │          ├─ store_create_info(..., for_show_create_stmt=false)   ★ 必含 GIPK
  │          └─ write_bin_log(generated_query)
  │
  └─【DD 打标】dd_table.cc: table/index/column 各写 options["gipk"]=true
```

**★ ALTER TABLE 不在这条链上**——全仓库只有 **2 处**调用 `validate_and_generate_invisible_primary_key()`：

| 调用点 | 所在函数 | 路径 |
|---|---|---|
| `sql/sql_table.cc:10163` | `mysql_create_table()` | 普通 CREATE TABLE / CREATE TEMPORARY TABLE |
| `sql/sql_insert.cc:2741` | `create_table_from_items()` | CREATE TABLE ... SELECT |

### 一、生成的触发与条件

**★★ 候选判定**（`sql/sql_gipk.cc:64`，★ 只有两个条件，其它都不在这里查）：

```cpp
bool is_candidate_table_for_invisible_primary_key_generation(
    const HA_CREATE_INFO *create_info, Alter_info *alter_info) {
  // Check PK generation is supported for the table's storage engine.
  if (!is_generating_invisible_pk_supported_for_se(create_info->db_type))
    return false;                      /* ① 引擎门槛：只有 InnoDB */

  // Check if primary key is specified for the table.
  const Mem_root_array<Key_spec *> &kl = alter_info->key_list;
  if (std::any_of(kl.begin(), kl.end(), [](const Key_spec *ks) {
        return (ks->type == KEYTYPE_PRIMARY);
      }))
    return false;                      /* ② 已有显式 PRIMARY KEY → 静默不生成 */

  return true;
}
```
> ★★ 注意**不在这里检查**的：分区、AUTO_INCREMENT 列、同名列——那些在 validate 阶段，且是**报错**而非静默。所以"已有主键"与"已有自增列"的后果完全不同：前者静默跳过，后者报错失败。

```cpp
/* sql/sql_gipk.cc:52 —— ★ 引擎门槛的真相 */
static bool is_generating_invisible_pk_supported_for_se(
    handlerton *se_handlerton) {
  // Invisible PK generation is supported for only InnoDB tables for now.
  return (ha_check_storage_engine_flag(se_handlerton,
                                       HTON_SUPPORTS_GENERATED_INVISIBLE_PK));
}
```
```cpp
/* sql/handler.h:2932 */
/** Engine supports Generated invisible primary key. */
constexpr const decltype(
    handlerton::flags) HTON_SUPPORTS_GENERATED_INVISIBLE_PK{1 << 18};
```
> ★ 全仓库只有 InnoDB 声明了这个 flag（`storage/innobase/handler/ha_innodb.cc:5177`）。**MyISAM 静默不生成**，不报错。

**★★ 四条禁令**（`sql/sql_gipk.cc:90`，★ 官方 Doxygen 逐条列出）：

```cpp
/**
  Validate and generate invisible primary key for a candidate table (table
  being created).

  Primary key is generated if,
    a) Table is a non-partitioned table. Generating invisible primary
       key is not supported for partitioned tables for now.
    b) Table does *not* have a column with auto_increment attribute.
    c) Table does *not* have a column with name "my_row_id".
    d) Table is *not* created using CREATE TABLE ... SELECT in
       binlog_format=STATEMENT mode.
  Otherwise an error is reported in the validation of phase.
*/
```

```cpp
  // CREATE TABLE ... SELECT
  if (!thd->lex->query_block->field_list_is_empty()) {
    /*
      Mark statement as unsafe so that decide_logging_format() knows that it
      needs to use row format when binlog_format=MIXED
    */
    thd->lex->set_stmt_unsafe(LEX::BINLOG_STMT_UNSAFE_CREATE_SELECT_WITH_GIPK);
                                    /* ★ MIXED 模式下强制降级为 ROW */
    /*
      Report an error when binlog_format=STATEMENT.

      Generating invisible primary key for the CREATE TABLE SELECT in SBR mode
      is unsafe. This operation can *not* be replicated safely. Order in which
      auto-increment values generated for the my_row_id column is
      non-deterministic, so replicating this operation is not safe using SBR.
    */
    if (thd->variables.binlog_format == BINLOG_FORMAT_STMT) {
      my_error(ER_CREATE_SELECT_WITH_GIPK_DISALLOWED_IN_SBR, MYF(0));
      return true;
    }
  }

  /*
    Generating invisible PK is *not* supported for the partitioned tables for
    now..
  */
  if (thd->lex->part_info != nullptr) {
    my_error(ER_NOT_SUPPORTED_YET, MYF(0),
             "generating invisible primary key for the partitioned tables");
    return true;
  }

  for (Create_field cr_field : alter_info->create_list) {
    if (is_generated_invisible_primary_key_column_name(cr_field.field_name)) {
      my_error(ER_GIPK_COLUMN_EXISTS, MYF(0));
      return true;                  /* ★ 列名冲突 */
    }
    if (cr_field.auto_flags & Field::NEXT_NUMBER) {
      my_error(ER_GIPK_FAILED_AUTOINC_COLUMN_EXISTS, MYF(0));
      return true;                  /* ★ 已有 AUTO_INCREMENT 列 */
    }
  }
```
> ★ 列名比较用 `my_strcasecmp(system_charset_info, ...)` —— **大小写不敏感**，`My_Row_Id` / `MY_ROW_ID` 都算冲突。

**★ 生成场景总表**：

| 场景 | 是否生成 | 证据 |
|---|---|---|
| `CREATE TABLE`（InnoDB，无 PK） | ✅ 生成 | `sql/sql_table.cc:10160` |
| `CREATE TEMPORARY TABLE` | ✅ **会**生成 | `sql/sql_table.cc:10214`（临时表也走 binlog 重写分支） |
| `CREATE TABLE ... SELECT` | ✅ 生成；SBR 下报错，MIXED 降级 ROW | `sql/sql_insert.cc:2738` |
| `CREATE TABLE ... LIKE` | ❌ **不生成** | 走 `mysql_create_table_no_lock()` 绕过 `mysql_create_table()`（`sql/sql_table.cc:11030`） |
| 已有显式 PRIMARY KEY | ❌ 静默不生成 | `sql/sql_gipk.cc:71` |
| 非 InnoDB 引擎 | ❌ 静默不生成 | `sql/sql_gipk.cc:67` |
| 分区表 | ❌ 报 `ER_NOT_SUPPORTED_YET` | `sql/sql_gipk.cc:117` |
| bootstrap / --initialize | ❌ 不生成 | `sql/sql_gipk.cc:59` |
| **ALTER TABLE** | ❌ **从不生成** | 全仓库仅 2 处调用，都不在 ALTER 路径 |

> ★★ `CREATE TABLE ... LIKE` 不生成是一个真实的坑：`sql_require_primary_key=ON` 时 `CREATE TABLE t3 LIKE t1` 会报 `ER_TABLE_WITHOUT_PK`。

### 二、生成什么：一个真实的物理列

**★★★ 核心函数**（`sql/sql_gipk.cc:153`，★ 全篇最重要的一段）：

```cpp
static bool generate_invisible_primary_key(THD *thd, Alter_info *alter_info) {
  /*
    Create primary key column "my_row_id bigint unsigned NOT NULL AUTO_INCREMENT
    INVISIBLE" and add it as the first column in the column list.
  */
  Create_field *cr_field = new (thd->mem_root) Create_field();
  if (cr_field == nullptr) return true;  // OOM

  if (cr_field->init(
          thd, gipk_column_name, MYSQL_TYPE_LONGLONG, nullptr, nullptr,
          (UNSIGNED_FLAG | NOT_NULL_FLAG | AUTO_INCREMENT_FLAG), nullptr,
          nullptr, &EMPTY_CSTR, nullptr, nullptr, nullptr, false, 0, nullptr,
          nullptr, {}, dd::Column::enum_hidden_type::HT_HIDDEN_USER, false))
    return true;
                /* ★★ 逐实参对齐 Create_field::init() 签名：
                   1  thd
                   2  fld_name      = gipk_column_name ("my_row_id")
                   3  fld_type      = MYSQL_TYPE_LONGLONG        (BIGINT, 8 字节)
                   4  display_width = nullptr                    (不指定显示宽度)
                   5  fld_decimals  = nullptr                    (decimals=0)
                   6  fld_type_modifier = UNSIGNED|NOT_NULL|AUTO_INCREMENT
                   7  fld_default_value = nullptr                (无 DEFAULT)
                   8  fld_on_update_value = nullptr              (无 ON UPDATE)
                   9  fld_comment   = &EMPTY_CSTR                (空注释)
                  10  fld_change    = nullptr                    (非 CHANGE/MODIFY)
                  11  fld_interval_list = nullptr                (非 ENUM/SET)
                  12  fld_charset   = nullptr                    (数值类型无字符集)
                  13  has_explicit_collation = false
                  14  fld_geom_type = 0
                  15  fld_gcol_info = nullptr   ★★★ 无生成列信息 → 不是 generated column
                  16  fld_default_val_expr = nullptr
                  17  srid          = {}
                  18  hidden        = HT_HIDDEN_USER   ★★ 不是 HT_HIDDEN_SQL
                  19  is_array_arg  = false                                   */
  if (alter_info->create_list.push_front(cr_field)) return true;
                                    /* ★ push_front：GIPK 列永远是第 0 列 */

  // Create primary key and add it the key list.
  List<Key_part_spec> key_parts;
  Key_part_spec *key_part_spec = new (thd->mem_root)
      Key_part_spec({gipk_column_name, strlen(gipk_column_name)}, 0, ORDER_ASC);
  if (key_part_spec == nullptr || key_parts.push_back(key_part_spec))
    return true;  // OOM

  Key_spec *key = new (thd->mem_root)
      Key_spec(thd->mem_root, KEYTYPE_PRIMARY, NULL_CSTR,
               &default_key_create_info, false, true, key_parts);
                                    /* ★ NULL_CSTR = 让服务器自动赋名 "PRIMARY" */
  if (key == nullptr || alter_info->key_list.push_back(key))
    return true;  // OOM

  return false;
}
```

**★ 列名常量**（`sql/sql_gipk.cc:36`，★ 硬编码，无任何变量可改）：

```cpp
/* Generated invisible primary key column name. */
const char *gipk_column_name = "my_row_id";
```

**★ 属性汇总**：

| 属性 | 值 |
|---|---|
| 列名 | `my_row_id`（硬编码） |
| 类型 | `MYSQL_TYPE_LONGLONG` = BIGINT，**8 字节** |
| 修饰符 | `UNSIGNED \| NOT NULL \| AUTO_INCREMENT` |
| `auto_flags` | `Field::NEXT_NUMBER`（由 `AUTO_INCREMENT_FLAG` 置位，`create_field.cc:241`） |
| `hidden` | `HT_HIDDEN_USER` (=4) |
| `gcol_info` | **`nullptr`** —— 不是生成列，是 STORED 普通列 |
| 位置 | 永远第 0 列 |
| 主键 | 单列 `PRIMARY KEY`，ASC，整列 |

**★ AUTO_INCREMENT 的三个字段不是 GIPK 设的，是反推出来的**：

```cpp
/* sql/table.cc:2224 —— ★ 建表后由 find_ref_key() 反推 */
  if (share->found_next_number_field) {
    Field *reg_field = *share->found_next_number_field;
    if ((int)(share->next_number_index = (uint)find_ref_key(
                  share->key_info, share->keys, share->default_values,
                  reg_field, &share->next_number_key_offset,
                  &share->next_number_keypart)) < 0) {
```
> ★ 因为 GIPK 列是第 0 列且主键是它的单列索引，所以 `next_number_index = 0`、`offset = 0`、`keypart = 0`。

### 三、可见性：GIPK 在哪儿可见

**★★ 唯一的开关**（`sql/sys_vars.cc:7315`，★ 默认 **ON**）：

```cpp
static Sys_var_bool Sys_show_gipk_in_create_table_and_information_schema(
    "show_gipk_in_create_table_and_information_schema",
    "When set, if a primary key is generated for a table then SHOW commands "
    "and INFORMATION_SCHEMA tables shows generated invisible primary key "
    "definition.",
    SESSION_VAR(show_gipk_in_create_table_and_information_schema),
    CMD_LINE(OPT_ARG), DEFAULT(true), NO_MUTEX_GUARD, NOT_IN_BINLOG,
                                    /* ★★ DEFAULT(true) —— 默认显示！ */
    ON_CHECK(nullptr), ON_UPDATE(nullptr));
```

**★ `SHOW CREATE TABLE`：`skip_gipk` 是局部变量，不是函数**（`sql/sql_show.cc:2028`）：

```cpp
  /*
    When building CREATE TABLE statement for the SHOW CREATE TABLE (i.e.
    for_show_create_stmt = true), skip generated invisible primary key
    if system variable 'show_gipk_in_create_table_and_information_schema' is set
    to OFF.
  */
  bool skip_gipk =
      (for_show_create_stmt &&
       table_has_generated_invisible_primary_key(table) &&
       !thd->variables.show_gipk_in_create_table_and_information_schema);
                        /* ★ 三条件：是给 SHOW CREATE 用的 + 有 GIPK + 变量 OFF */
  Field **first_field = table->field;
  assert(!table_has_generated_invisible_primary_key(table) ||
         is_generated_invisible_primary_key_column_name(
             (*first_field)->field_name));
  if (skip_gipk) first_field++;       /* ★ 指针 ++ 跳过第 0 列 */

  for (ptr = first_field; (field = *ptr); ptr++) {
```
```cpp
  key_info = table->key_info;
  if (skip_gipk) key_info++;          /* ★ 索引也跳过第 0 个（PRIMARY） */
  for (uint i = skip_gipk ? 1 : 0; i < share->keys; i++, key_info++) {
```
> ★ 连 `AUTO_INCREMENT=N` 表选项也一起隐藏（`sql_show.cc:2405` 的 `!skip_gipk` 条件）。
> ★★ **`skip_gipk` 只在 `for_show_create_stmt == true` 时生效**。binlog 重写传的是 `false`，所以**写进 binlog 的语句永远包含完整 GIPK 定义**——这是复制正确性的保证。

**★ `SHOW COLUMNS` / `DESCRIBE` 同样跳过第 0 列**（`sql/sql_show.cc:4028`）：

```cpp
  if (table_has_generated_invisible_primary_key(show_table) &&
      !thd->variables.show_gipk_in_create_table_and_information_schema) {
    assert(is_generated_invisible_primary_key_column_name((*ptr)->field_name));
    ptr++;                            /* ★ SHOW COLUMNS 跳过 GIPK 列 */
  }
```

**★ `SHOW INDEX` / `SHOW KEYS` 跳过 PRIMARY**（`sql/sql_show.cc:4258`）：

```cpp
  if (table_has_generated_invisible_primary_key(show_table) &&
      !thd->variables.show_gipk_in_create_table_and_information_schema) {
    assert((key_info->user_defined_key_parts == 1) &&
           (is_generated_invisible_primary_key_column_name(
               key_info->key_part->field->field_name)));
    i++;                              /* ★ 索引序号 +1 */
    key_info++;
  }
```

**★★ `SELECT *` 不返回，但可显式引用**（★ 这是 `HT_HIDDEN_USER` 的直接后果）：

```cpp
/* sql/sql_base.cc:8412 —— ★ SELECT * 的过滤点 */
  auto is_non_participant_column = [using_fields](Field *field) {
    return (field != nullptr &&
            (field->is_hidden_by_system() ||
             ((using_fields == nullptr) && field->is_hidden_by_user())));
  };
                /* ★ using_fields == nullptr ⇒ SELECT * / 隐式引用 ⇒ INVISIBLE 列被排除
                   using_fields 非空 ⇒ 显式写了列名 ⇒ 不被排除 */
```
> ★★ 所以 `SELECT * FROM t` 看不到 `my_row_id`，但 `SELECT my_row_id FROM t` **是合法的**。
> ★ 显式引用后，内部临时表还会把它变回 VISIBLE（`sql/sql_tmp_table.cc:209`）：
> ```cpp
>   /*
>     Invisible column is explicitly referred in the column list. Mark it as
>     VISIBLE column in the internal temporary table.
>   */
>   if (new_field->is_hidden_by_user())
>     new_field->set_hidden(dd::Column::enum_hidden_type::HT_VISIBLE);
> ```

### 四、DD 打标与 I_S 过滤

**★★ DD 三处打标**（`sql/dd/dd_table.cc:2197`，★ 唯一写入处）：

```cpp
  /*
    If table definition has a generated invisible primary key then set "gipk"
    option for the table, primary key and key column.

    At this point we use these "gipk" flags only in INFORMATION_SCHEMA
    implementation to filter out information about the GIPK in
    show_gipk_in_create_table_and_information_schema=OFF mode.
  */
  if (table_def_has_generated_invisible_primary_key(
          thd, create_info->db_type, create_fields, keys, keyinfo)) {
    // Set "gipk" option for the table.
    table_options->set("gipk", true);        /* ★ ① 表级 */

    // Set "gipk" option for the primary key.
    for (dd::Index *idx : *tab_obj->indexes()) {
      if (idx->type() == dd::Index::IT_PRIMARY) {
        idx->options().set("gipk", true);    /* ★ ② 索引级 */
        break;
      }
    }
    // ques: what id gipk?????             /* ⚠️ Oracle 未清理的开发疑问注释 */
    // Set "gipk" option for the column added for GIPK.
    for (dd::Column *col : *tab_obj->columns()) {
      if (is_generated_invisible_primary_key_column_name(col->name().c_str())) {
        col->options().set("gipk", true);    /* ★ ③ 列级（按名字匹配） */
        break;
      }
    }
  }
```
> ★ option key 字符串**就是 `"gipk"`**（不是 `"gipk_column"`）。三处值都是 `true`。
> ⚠️ 源码注释里也把它叫 "generated **implicit** primary key"（`index_impl.cc:68`、`column_impl.cc:70`），与 "invisible" 混用——Oracle 自身的术语不一致。

```cpp
/* 三处 option 白名单，否则 set() 会被拒绝 */
/* sql/dd/impl/types/abstract_table_impl.cc:92 */  "gipk"
/* sql/dd/impl/types/index_impl.cc:68 */           "gipk" /* generated implicit primary key */
/* sql/dd/impl/types/column_impl.cc:70 */          "gipk" /* generated implicit primary key column */
```

**★★ I_S 过滤的真身：`IS_VISIBLE_DD_OBJECT` 是一个内部 SQL 函数**（⚠️ 不是宏）：

```cpp
/* sql/item_func.h:2508 */
class Item_func_is_visible_dd_object : public Item_int_func {
  ...
  const char *func_name() const override { return "is_visible_dd_object"; }
```
```cpp
/* sql/item_create.cc:1710 —— 注册为内部函数，变参 1~3 个 */
    {"IS_VISIBLE_DD_OBJECT",
     SQL_FN_INTERNAL_V(Item_func_is_visible_dd_object, 1, 3)},
```
```cpp
/* sql/item_func.cc:9208 —— ★ 解析 options 里的 "gipk" */
  // Skip generated invisible primary key and key columns.
  if (arg_count == 3 && !args[2]->is_null() &&
      !thd->variables.show_gipk_in_create_table_and_information_schema) {
    String options;
    String *options_ptr = args[2]->val_str(&options);
    if (options_ptr != nullptr) {
      std::unique_ptr<dd::Properties> p(
          dd::Properties::parse_properties(options_ptr->c_ptr_safe()));
      if (p.get()) {
        if (p->exists("gipk")) {
          bool gipk_value = false;
          p->get("gipk", &gipk_value);
          if (gipk_value) return 0;      /* ★ 判定"不可见" → 该行被过滤掉 */
        }
      }
```
> ★★ Doxygen 明写：*"GIPK and key column are skipped even for SHOW EXTENDED command"* —— **`SHOW EXTENDED` 救不了 GIPK**，只有把变量设 ON 才行。

**★ 受影响的 I_S 视图**（六个，WL#13784 定义）：

```cpp
/* sql/dd/info_schema/metadata.h:227 */
  - WL#13784: Following INFORMATION_SCHEMA tables are modified to skip listing
              generated invisible primary key and key column when system
              variable show_gipk_in_create_table_and_information_schema = OFF,
                INFORMATION_SCHEMA.COLUMNS
                INFORMATION_SCHEMA.COLUMNS_EXTENSIONS
                INFORMATION_SCHEMA.KEY_COLUMN_USAGE
                INFORMATION_SCHEMA.STATISTICS
                INFORMATION_SCHEMA.TABLE_CONSTRAINTS
                INFORMATION_SCHEMA.TABLE_CONSTRAINTS_EXTENSIONS
```

```cpp
/* sql/dd/impl/system_views/columns.cc:142 —— I_S.COLUMNS 的 WHERE 子句 */
  m_target_def.add_where(
      "AND IS_VISIBLE_DD_OBJECT(tbl.hidden, col.hidden NOT IN ('Visible', "
      "'User'), col.options)");       /* ★ 第 3 参携带 "gipk" */
```
> ★ `I_S.TABLES` 里连 AUTO_INCREMENT 值都要隐藏（`tables.cc:149`）。

### 五、binlog 与复制

**★★ 为什么要重写语句**（★ 已在「设计思想」给出注释，这里给完整实现）：

```cpp
/* sql/sql_table.cc:10208 —— ★ 完整的重写 + 落 binlog */
      if ((create_table->table == nullptr && !create_table->is_view()) &&
          is_pk_generated) {
        /*
          Open table to generate CREATE TABLE statement. For non-temporary
          table we already have exclusive lock here.
        */
        if (create_info->options & HA_LEX_CREATE_TMP_TABLE) {
          result = open_temporary_table(thd, create_table);   /* ★ 临时表也走这里 */
        } else {
          Open_table_context ot_ctx(thd, MYSQL_OPEN_REOPEN);
          result = open_table(thd, create_table, &ot_ctx);
          tdc_remove_table(thd, TDC_RT_REMOVE_NOT_OWN, create_table->db,
                           create_table->table_name, false);
        }

        if (!result) {
          char buf[2048];
          String query(buf, sizeof(buf), system_charset_info);
          query.length(0);
          result = store_create_info(thd, create_table, &query, create_info,
                                     true /* show_database */,
                                     false /* SHOW CREATE TABLE */);
                    /* ★★ 第 6 参 for_show_create_stmt=false ⇒ skip_gipk 恒 false
                       ⇒ 生成的语句必定含 my_row_id + PRIMARY KEY
                       ⇒ 不受 show_gipk_in_create_table_and_information_schema 影响 */
          assert(result == 0);

          // Write generated CREATE TABLE statement to binlog.
          result = write_bin_log(thd, true, query.ptr(), query.length(), is_trans);
```
> ⚠️ `is_pk_generated` 是 `mysql_create_table()` 的**局部 bool**（`sql_table.cc:10051`），不在 `TABLE` / `TABLE_SHARE` 里。

**★ binlog 里的 GIPK 标志位**：

```cpp
/* sql/log_event.h:2363 */
    /**
      Table has generated invisible primary key. MySQL generates primary key
      while creating a table if sql_generate_invisible_primary_key is "ON" and
      table is PK-less.
    */
    TM_GENERATED_INVISIBLE_PK_F = (1U << 2)
```
```cpp
/* sql/log_event.cc:10763 —— source 侧写入 */
  if (table_has_generated_invisible_primary_key(m_table))
    m_flags |= TM_GENERATED_INVISIBLE_PK_F;
```

**★★ 从库会自己开 GIPK 吗？—— 默认不会**（★ 变量是 `NOT_IN_BINLOG`）：

```cpp
/* sql/rpl_replica.cc:6050 —— ★ 只有通道设了 GENERATE 才就地打开 */
  thd->variables.sql_generate_invisible_primary_key = false;
  if (thd->rpl_thd_ctx.get_rpl_channel_type() != GR_APPLIER_CHANNEL &&
      thd->rpl_thd_ctx.get_rpl_channel_type() != GR_RECOVERY_CHANNEL &&
      Relay_log_info::PK_CHECK_GENERATE ==
          rli->get_require_table_primary_key_check()) {
    thd->variables.sql_generate_invisible_primary_key = true;
  }
```
```cpp
/* sql/rpl_rli.h:276 —— 四档策略 */
    PK_CHECK_STREAM = 1,    /* 跟随主库 sql_require_primary_key */
    PK_CHECK_ON = 2,        /* 强制要求 */
    PK_CHECK_OFF = 3,       /* 不强制 */
    PK_CHECK_GENERATE = 4   /* ★ 从库自行生成 GIPK */
```
> ★ **Group Replication 通道禁止 GENERATE**：`sql/rpl_replica.cc:9743` 报 `ER_REQUIRE_TABLE_PRIMARY_KEY_CHECK_GENERATE_WITH_GR`。

**★★ 主从 GIPK 不对等时的列映射**（★ 最复杂的部分）：

```cpp
/* sql/rpl_utility.cc:795 —— ★ 推断 source 是否有 GIPK */
bool table_def::compute_source_table_gipk_info(THD &thd, TABLE *table) {
  m_is_gipk_set = true;

  // Check if source signals the use of GIPKs on tables
  if (is_immediate_server_gipk_ready(thd)) {
    m_is_gipk_on_table =
        (m_flags & Table_map_log_event::TM_GENERATED_INVISIBLE_PK_F);
    return false;                    /* ★ source >= 8.0.30：直接读标志位，准确 */
  }

  // If the replica has no GIPK assume the source doesn't also
  if (!table_has_generated_invisible_primary_key(table)) {
    m_is_gipk_on_table = false;
    return false;
  }

  // column difference = number of columns in source - replica
  longlong column_difference =
      static_cast<longlong>(size()) - static_cast<longlong>(table->s->fields);

  if (0 == column_difference) {
    m_is_gipk_on_table = true;
    return false;              /* ★ 列数相同 ⇒ 认为两边都有 GIPK */
  }
  if (-1 == column_difference) {
    m_is_gipk_on_table = false;
    return false;              /* ★ 从库多 1 列 ⇒ 认为只有从库有 GIPK */
  }

  m_is_gipk_on_table = false;
  m_is_gipk_set = false;
  return true;                 /* ★ 无法判定 → ER_REPLICATION_INCOMPATIBLE_TABLE_WITH_GIPK */
}
```
> ★★ source < 8.0.30 时**只能靠列数差猜**——这就是"source 太老 + 从库有 GIPK + 列数对不上"会报错的根因。

```cpp
/* sql/rpl_record.cc:781 —— ★ source 有 GIPK / 从库没有 → 直接吃掉第 0 列字节 */
const uchar *translate_beginning_of_raw_data(
    const uchar *raw_data, MY_BITMAP const *column_image, size_t column_count,
    Bit_reader &null_bits, table_def *tabledef, bool source_has_gipk,
    bool replica_has_gipk) {
  if (source_has_gipk && !replica_has_gipk) {
    if (bitmap_is_set(column_image, 0)) {
      null_bits.get();
    }
    const uchar *ptr = raw_data + ((column_count + 7) / 8);
    uint32 first_column_len = tabledef->calc_field_size(0, ptr);
    return ptr + first_column_len;
  }
  return raw_data + (column_count + 7) / 8;
}
```

**★ 一个实践要点：GIPK 不会用于从库行定位**（★ 因为 PK 本来就被排除）：

```cpp
/* sql/log_event.cc:8351 —— 候选索引筛选 */
      if (!((keyinfo->flags & (HA_NOSAME | HA_NULL_PART_KEY)) == HA_NOSAME) ||
          (key == table->s->primary_key) ||                /* ★★ PK 被无条件跳过 */
          ((slave_rows_search_algorithms_options & SLAVE_ROWS_INDEX_SCAN) &&
           keyinfo->is_functional_index()) ||
          keyinfo->flags & HA_MULTI_VALUED_KEY || !keyinfo->is_visible) {
        continue;
      }
```
> ★ GIPK 就是 PK，所以**永远不会被选中做行定位**。这意味着 GIPK 场景下 `slave_rows_search_algorithms` 通常退化为 TABLE_SCAN / HASH_SCAN——**GIPK 带来的主键并不能直接用于复制行定位**，它的价值在别处（表结构完整性、避免全表扫描的上游）。

### 六、ALTER 限制

**★★ 三条限制**（`sql/sql_gipk.h:126`，★ 官方 Doxygen 原文）：

```cpp
/**
  Check ALTER restrictions on primary key and column.

  Following ALTER restrictions are applicable on primary key and column,

    *) When sql_generate_invisible_primary_key is enabled then, primary key
       is allowed to drop only if new table definition has a primary key.

    *) Generated invisible primary key is allowed to drop only if primary key
       column is also dropped. This restriction is applicable irrespective of
       sql_generate_invisible_primary_key's state.

    *) CHANGE/MODIFY OR ALTER operations on generated invisible primary key
       columns are *not* allowed except ALTER operation to change column
       visibility attribute. This restriction is applicable irrespective of
       sql_generate_invisible_primary_key's state.
*/
```

**限制 1：GIPK 模式下不能"删了 PK 不加新的"**（`sql/sql_gipk.cc:273`）：

```cpp
  if (is_generate_invisible_primary_key_mode_active(thd)) {
    Mem_root_array<Key_spec *> &key_list = alter_info->key_list;
    if (std::count_if(key_list.begin(), key_list.end(), [](const Key_spec *ks) {
          return (ks->type == KEYTYPE_PRIMARY);
        }) == 0) {
      /*
        When GIPK mode is active, dropping existing primary key without adding
        a new primary key in not supported for now. But this restriction will be
        relaxed eventually by automagically generating a primary key.
      */
      my_error(ER_NOT_SUPPORTED_YET, MYF(0), "...");
      return true;
    }
  }
```
> ★ 注释里那句 *"will be relaxed eventually by automagically generating a primary key"* —— **8.0.39 还没实现**自动生成。

**限制 2：删 GIPK 主键必须先删列**（`sql/sql_gipk.cc:309`）：

```cpp
    bool is_gipk_dropped = false;
    if ((alter_info->flags & Alter_info::ALTER_DROP_COLUMN) &&
        std::any_of(alter_info->drop_list.begin(), alter_info->drop_list.end(),
            [](const Alter_drop *d) {
              return (d->type == Alter_drop::COLUMN &&
                      is_generated_invisible_primary_key_column_name(d->name));
            })) {
      /*
        MySQL automatically drops the key when all its columns (or single
        column) is dropped. We stick to this behavior for GIPK column for
        consistency sake, even though it might be not intuitive.
      */
      is_gipk_dropped = true;      /* ★ DROP COLUMN my_row_id → 连带删 PK */
    } else if ((alter_info->flags & Alter_info::ALTER_DROP_INDEX) && ...) {
      /*
        Generated invisible primary key column should be dropped to drop
        primary key.
      */
      my_error(ER_DROP_PK_COLUMN_TO_DROP_GIPK, MYF(0));   /* ★ 只 DROP PK → 报错 */
      return true;
    }
```
> ★ 注释自白 *"even though it might be not intuitive"* —— 官方承认这个行为不直观。

**限制 3：GIPK 列只允许改可见性**（`sql/sql_gipk.cc:352`）：

```cpp
    Create_field *cr_field = alter_info->create_list.head();
    /*
      First column should be "my_row_id unsigned BIGINT NOT NULL
      AUTO_INCREMENT". Changing only visibility attribute is allowed.
    */
    if (!is_generated_invisible_primary_key_column_name(cr_field->field_name) ||
        (cr_field->change != nullptr &&
         !is_generated_invisible_primary_key_column_name(cr_field->change)) ||
        cr_field->sql_type != MYSQL_TYPE_LONGLONG ||
        !(cr_field->auto_flags & Field::NEXT_NUMBER) ||
        !(cr_field->flags & UNSIGNED_FLAG) || cr_field->is_nullable) {
      my_error(ER_GIPK_COLUMN_ALTER_NOT_ALLOWED, MYF(0));
      return true;
    }

    for (const Alter_column *alter_column : alter_info->alter_list) {
      if (alter_column->change_type() == Alter_column::Type::SET_COLUMN_VISIBLE)
        continue;                    /* ★★ 唯一豁免：SET VISIBLE */

      if (is_generated_invisible_primary_key_column_name(alter_column->name)) {
        my_error(ER_GIPK_COLUMN_ALTER_NOT_ALLOWED, MYF(0));
        return true;
      }
    }
```
> ★★ **`ALTER TABLE t ALTER my_row_id SET VISIBLE` 是解除所有限制的钥匙**——因为 `is_generated_invisible_primary_key_column()` 的判据里有 `is_hidden_by_user()`，改 VISIBLE 后这一列就"不再是 GIPK"了。

**★ 附带机制：GIPK 列被挤走时自动搬回第 0 位**（`sql/sql_gipk.cc:224`）：

```cpp
  /*
    Generated invisible primary key column is neither dropped nor altered
    but new columns are added before generated invisible primary key column.
    Generated invisible primary key column must be at the first position.
  */
  fld_it.remove();
  prepared_create_list->push_front(fld);
```
> ★ 场景：`ALTER TABLE t ADD COLUMN f1 INT FIRST` 之后，GIPK 列被挤到第 2 位，这里搬回第 1 位。

---

## 可观测性

### 系统变量与状态变量

| 变量名 | 默认值 | 作用域 | 说明 |
|---|---|---|---|
| `sql_generate_invisible_primary_key` | **OFF** | GLOBAL+SESSION（`SESSION_VAR`） | 是否自动生成 GIPK。`NOT_IN_BINLOG`、`ON_CHECK(check_session_admin)`（需 SESSION_VARIABLES_ADMIN） |
| `show_gipk_in_create_table_and_information_schema` | **ON** | GLOBAL+SESSION（`SESSION_VAR`） | SHOW / I_S 是否显示 GIPK。`NOT_IN_BINLOG`，无权限门槛 |
| `sql_require_primary_key` | OFF | GLOBAL+SESSION（`HINT_UPDATEABLE`） | ⚠️ **另一套机制**，不是 GIPK 的一部分（见下节） |

```cpp
/* sql/sys_vars.cc:7307 —— ★ 两个 GIPK 变量是紧挨着注册的 */
static Sys_var_bool Sys_sql_generate_invisible_primary_key(
    "sql_generate_invisible_primary_key",
    "When set, if a table is created without a primary key then server "
    "generates invisible auto-increment column as a primary key for the table.",
    SESSION_VAR(sql_generate_invisible_primary_key), CMD_LINE(OPT_ARG),
    DEFAULT(false), NO_MUTEX_GUARD, NOT_IN_BINLOG,      /* ★★ NOT_IN_BINLOG */
    ON_CHECK(check_session_admin), ON_UPDATE(nullptr));
```

### 观测对象 → 手段 速查

| 我想看 | 手段 | 入口 |
|---|---|---|
| 某表有没有 GIPK | SQL | `SHOW CREATE TABLE t`（默认就能看到 `my_row_id`） |
| 隐藏 GIPK 看原始定义 | SQL | `SET SESSION show_gipk_in_create_table_and_information_schema=OFF;` 再 `SHOW CREATE TABLE` |
| I_S 里 GIPK 是否被过滤 | SQL | `information_schema.COLUMNS` / `STATISTICS` / `KEY_COLUMN_USAGE` |
| binlog 里 GIPK 语句是否被重写 | SQL | `mysqlbinlog` 看 CREATE TABLE 语句；`>= 8.0.30` 还会打印 `# has_generated_invisible_primary_key=` |
| 复制通道的 GIPK 策略 | SQL | `REQUIRE_TABLE_PRIMARY_KEY_CHECK`（`STREAM` / `ON` / `OFF` / `GENERATE`） |
|  mysqldump 是否导出 GIPK | SQL | `--skip-generated-invisible-primary-key`（内部设 `show_gipk...=OFF`） |

---

## Misc

### 与 `sql_require_primary_key` 的关系（★ 极易混淆，必须澄清）

| 维度 | `sql_generate_invisible_primary_key` (GIPK) | `sql_require_primary_key` |
|---|---|---|
| 语义 | **帮你生成一个**隐藏自增 PK | **拒绝**创建/变更无 PK 的表 |
| 默认值 | `false` | `false` |
| 是否进 binlog | **`NOT_IN_BINLOG`** | **`IN_BINLOG`** |
| `HINT_UPDATEABLE` | 无 | **有**（可用 optimizer hint 覆盖） |
| 报错码 | `ER_GIPK_*` 系列 | `ER_TABLE_WITHOUT_PK` |
| 复制语义 | 靠**重写 CREATE TABLE 语句**传递 | 靠**变量值本身进 binlog**传递 |

```cpp
/* sql/sql_table.cc:8203 —— sql_require_primary_key 的检查点 */
  // If the table is created without PK, we must check if this has
  // been disabled and return error. Limit the effect of sql_require_primary_key
  // to only those SEs that can participate in replication.
  if (!primary_key && !thd->is_dd_system_thread() &&
      !thd->is_initialize_system_thread() &&
      (file->ha_table_flags() &
       (HA_BINLOG_ROW_CAPABLE | HA_BINLOG_STMT_CAPABLE)) != 0 &&
      thd->variables.sql_require_primary_key) {
    my_error(ER_TABLE_WITHOUT_PK, MYF(0));
    return true;
  }
```
> ★★ 两者**天然兼容**：开了 GIPK 后 `!primary_key` 就不成立了。但 `CREATE TABLE ... LIKE` 不生成 GIPK，于是 `sql_require_primary_key=ON` 时会把它拦下。

### 相关错误码

| 错误码名 | 抛出位置 | 文案 |
|---|---|---|
| `ER_GIPK_COLUMN_EXISTS` | `sql/sql_gipk.cc:130` | Failed to generate invisible primary key. Column 'my_row_id' already exists. |
| `ER_GIPK_FAILED_AUTOINC_COLUMN_EXISTS` | `sql/sql_gipk.cc:135` | Failed to generate invisible primary key. Auto-increment column already exists. |
| `ER_GIPK_COLUMN_ALTER_NOT_ALLOWED` | `sql/sql_gipk.cc:375`、`:388` | Altering generated invisible primary key column 'my_row_id' is not allowed. |
| `ER_DROP_PK_COLUMN_TO_DROP_GIPK` | `sql/sql_gipk.cc:336` | Please drop primary key column to be able to drop generated invisible primary key. |
| `ER_CREATE_SELECT_WITH_GIPK_DISALLOWED_IN_SBR` | `sql/sql_gipk.cc:108` | ...disallowed when binlog_format=STATEMENT... |
| `ER_NOT_SUPPORTED_YET`（复用 ×3） | `sql/sql_gipk.cc:118`、`:287`、`:304` | 分区表 / 无新 PK 就 DROP PK / 给 GIPK 表分区 |
| `ER_REPLICATION_INCOMPATIBLE_TABLE_WITH_GIPK` | `sql/rpl_utility.cc:492` | ...Replication is unsupported when the source server is older than 8.0.30... |
| `ER_REQUIRE_TABLE_PRIMARY_KEY_CHECK_GENERATE_WITH_GR` | `sql/rpl_replica.cc:9745` | On a Group Replication channel, setting REQUIRE_TABLE_PRIMARY_KEY_CHECK to 'GENERATE' is not allowed. |

> ⚠️ **`ER_GIPK_*` 系列只有 3 个**（`_COLUMN_EXISTS`、`_FAILED_AUTOINC_COLUMN_EXISTS`、`_COLUMN_ALTER_NOT_ALLOWED`）；另外 2 个相关错误码**不带** `ER_GIPK_` 前缀。不存在 `ER_GIPK_NOT_SUPPORTED` / `ER_GIPK_PARTITION` 等。

### ★★★ 真名勘误

| 常见说法 | 8.0.39 真相 |
|---|---|
| GIPK 是虚拟生成列 / VIRTUAL generated column | **错** → 是普通 STORED 物理列，`fld_gcol_info` 传 `nullptr`，**没有 `gcol_info`** |
| hidden 类型是 `HT_HIDDEN_SQL` | **错** → 是 **`HT_HIDDEN_USER`**(=4)。`HT_HIDDEN_SQL`(=3) 是函数索引用的 |
| `show_gipk_in_create_table_and_information_schema` 默认 OFF | **错** → 默认 **`ON`**（`DEFAULT(true)`），即默认**显示** |
| `sql_generate_invisible_primary_key` 默认 ON | **错** → 默认 **OFF** |
| 两个变量是 GLOBAL 作用域 | **错** → 都是 `SESSION_VAR`（GLOBAL+SESSION 双值） |
| 公开 API 叫 `is_generated_invisible_primary_key()` | **半错** → 确实存在但它是 `sql_gipk.cc:466` 的 **static**（只接受 `const KEY*`），头文件未声明。对外判表用 **`table_has_generated_invisible_primary_key(TABLE*)`** |
| 模式开关叫 `is_gipk_mode_active()` | **错** → 真名 **`is_generate_invisible_primary_key_mode_active(THD*)`** |
| 有个函数叫 `skip_gipk()` | **错** → 是 `store_create_info()` 里的**局部 bool 变量** |
| `is_pk_generated` 在 `TABLE_SHARE` / `TABLE` 里 | **错** → 是 `mysql_create_table()` 的**局部 bool** |
| `IS_VISIBLE_DD_OBJECT` 是个宏 | **错** → 是内部 SQL 函数，C++ 类 `Item_func_is_visible_dd_object`，1~3 参 |
| 支持分区表 | **错** → 有 `part_info` 直接报 `ER_NOT_SUPPORTED_YET` |
| 所有引擎都支持 | **错** → 只有带 `HTON_SUPPORTS_GENERATED_INVISIBLE_PK`(1<<18) 的引擎，实际**仅 InnoDB** |
| InnoDB 里有 GIPK 专用代码 | **错** → `storage/` 下 grep `gipk` **零命中**，纯 server 层 |
| ALTER TABLE 无主键时会补生成 | **错** → 全仓库只有 2 处调用，都不在 ALTER 路径 |
| `CREATE TABLE ... LIKE` 会生成 | **错** → 绕过 `mysql_create_table()`，**不生成** |
| 临时表不生成 | **错** → **会生成** |
| `sql_require_primary_key` 与 GIPK 冲突 | **错** → 互补；但 CTAS-LIKE 不生成 PK，于是 `require_primary_key=ON` 时报 `ER_TABLE_WITHOUT_PK` |
| 已有 AUTO_INCREMENT 列时仍生成 | **错** → 报 `ER_GIPK_FAILED_AUTOINC_COLUMN_EXISTS` |
| 列名比较区分大小写 | **错** → 用 `my_strcasecmp(system_charset_info, ...)`，**大小写不敏感** |
| GIPK 列完全隐藏不可引用 | **错** → `SELECT *` 不返回，但 **`SELECT my_row_id` 合法** |
| `SHOW EXTENDED` 能让 GIPK 显形 | **错** → Doxygen 明说 "skipped even for SHOW EXTENDED" |
| GIPK 主键可用于从库行定位 | **错** → PK 在候选索引筛选时被无条件跳过 |
| 列名可自定义 | **错** → 硬编码 `"my_row_id"`，无任何变量 |
| `sql/sql_gipk.cc` 里有 WL#13784 注释 | **错** → 该文件 grep `WL#` **零命中**；WL 编号来自 `dd/info_schema/metadata.h` 与 MTR |

> ★ 一个官方未清理的开发残留：`sql/dd/dd_table.cc:2217` 有一行 `// ques: what id gipk?????`。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → CREATE TABLE → Generated Invisible Primary Keys*
- *MySQL 8.0 Reference Manual → Replication → REQUIRE_TABLE_PRIMARY_KEY_CHECK*

**WorkLog**
- [WL#13784](https://dev.mysql.com/worklog/task/?id=13784) *Create auto_increment PK on table creation when no PK is specified*（GIPK 本体 + I_S 隐藏）
- [WL#14639](https://dev.mysql.com/worklog/task/?id=14639) *Row-based replication between tables that differ in the existence of a generated invisible primary key*
- [WL#15419](https://dev.mysql.com/worklog/task/?id=15419) *Make the replica_generate_invisible_primary_key option settable per channel*

**相关文档**
- 主键与索引类型见 [`types.md`](types.md)、[`constraint.md`](constraint.md)
- 隐藏列体系（函数索引 / 多值索引用的 `HT_HIDDEN_SQL`）见 [`../server/query/07_optimize/13_functional_mv_index.md`](../server/query/07_optimize/13_functional_mv_index.md)
- 复制行定位与 `slave_rows_search_algorithms` 见 [`../server/replication/`](../server/replication/)
- CREATE TABLE 完整流程见 [`../feat/ddl/create_table.md`](../feat/ddl/create_table.md)

> ⚠️ 本篇所有源码结论均基于 8.0.39 实测；GIPK 版本演进较快，8.0.30~8.0.39 之间有增量（如 WL#14639 / WL#15419），更高版本请重新核实。
