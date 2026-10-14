# I_S 的 DD 半边：Data Dictionary 系统视图

> **边界**：8.0 的 `INFORMATION_SCHEMA` 有**两套并存的机制**，本篇讲其中**被 `sql/dd/` 接管的那半边**：
> - **本篇**：`sql/dd/impl/system_views/`（43 张）+ `sql/dd/info_schema/`——**8.0 里绝大多数的 I_S 表**（`COLUMNS` / `TABLES` / `STATISTICS` / `PARTITIONS` …）属于这一类，它们**是真正的视图**。
> - **[`../../feat/i_s.md`](../../infra/observability/i_s.md)**：另一半——`ST_SCHEMA_TABLE` 插件机制 + InnoDB 那 26 张 `INNODB_*` 表（**临时表物化**，不是视图）。两篇互指，合起来才是完整的 I_S。
> - **`SHOW` 语句的重写**落在本篇（它重写成的就是这些视图的查询），`SHOW STATUS/VARIABLES` 重写成 PFS 的部分在 [`../infra/variables.md`](../infra/variables.md)。
> - **[`dd.md`](dd.md) 讲"元数据怎么存"，本篇讲"元数据怎么暴露出去"**——两篇互补不重叠。以下均**不在本篇展开**（见 `dd.md`）：DD 表体系与 `mysql.ibd` 自举、对象模型（`dd::Table`/`Column`/`Index`）、`Dictionary_client` 与三层 registry 缓存、`se_private_data` 全键名表、`dd::Properties` 的落盘格式、SDI、DD 升级。本篇涉及这些概念时只取**消费视角**（如"把 `se_private_data` 传给 UDF"），机制一律不重复。

**一句话机制**：每张 I_S 表都是一条 `CREATE OR REPLACE VIEW` 语句——视图定义在 C++ 里用 builder API 拼出来，启动时执行这条 DDL 落进 DD；查询时 server 把它当普通视图，直接查底层的 `mysql.*` DD 表。

---

## 目录

- [概述](#概述)
  - [是什么](#是什么)
  - [版本演进](#版本演进)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
  - [一、机制：视图定义在 C++ 里拼出来](#一机制视图定义在-c-里拼出来)
  - [二、43 张视图清单](#二43-张视图清单)
  - [三、生命周期：注册、创建、版本](#三生命周期注册创建版本)
  - [四、深挖：TABLES 与 STATISTICS](#四深挖tables-与-statistics)
  - [五、深挖：PARTITIONS 与 COLUMN_STATISTICS](#五深挖partitions-与-column_statistics)
  - [六、深挖：FILES 与存储程序类](#六深挖files-与存储程序类)
  - [七、深挖：KEY_COLUMN_USAGE 与 KEYWORDS](#七深挖key_column_usage-与-keywords)
  - [八、深挖：5 张 innodb_* 与 8 张 *_extensions](#八深挖5-张-innodb_-与-8-张-_extensions)
  - [九、选型判据：为什么有些表不做成视图](#九选型判据为什么有些表不做成视图)
  - [十、SHOW 语句重写成视图查询](#十show-语句重写成视图查询)
- [已知限制与坑](#已知限制与坑)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

**I_S 表在 8.0 是真的视图**——不是"看起来像表"，而是由一条真正的 `CREATE VIEW` DDL 建出来的、持久存在于 DD 里的视图。

这与插件式 I_S 表（InnoDB 那 26 张）**是两种截然不同的实现**：

| | **DD 系统视图（本篇）** | **插件 I_S 表**（见 `feat/i_s.md`） |
|---|---|---|
| 本质 | **真视图**（`CREATE VIEW` 落进 DD） | 内存临时表（HEAP），查询时现场物化 |
| 定义方式 | C++ builder 拼出 CREATE VIEW 语句 | `ST_FIELD_INFO` 数组 |
| 取数 | server 查视图 → 查底层 `mysql.*` DD 表 | 插件的 `fill_table()` 回调 |
| 权限 | 视图定义里的 `CAN_ACCESS_*` UDF（**逐行粒度**） | 插件自己手写 `check_global_access`（**整表粒度**） |
| 版本 | 全局 `IS_VERSION` | 各自的 `plugin_version` |
| 例子 | `I_S.COLUMNS`、`TABLES`、`STATISTICS` | `INNODB_TRX`、`INNODB_BUFFER_PAGE` |

`System_view_definition` 接口只有**一个**方法，注释就把用途写死了：

```cpp
/*
  The purpose of this interface is to prepare the DDL statements
  necessary to create a I_S table.
*/
class System_view_definition {
 public:
  virtual ~System_view_definition() = default;
  virtual String_type build_ddl_create_view() const = 0;
};
```

### 版本演进

`IS_DD_VERSION` 的历史变更注释是权威来源（8.0.3 起，当前 80030）。要点：

| IS 版本 | 发布 | 变化 |
|---|---|---|
| 80012 | 8.0.12 | 新增 `VIEW_TABLE_USAGE` / `VIEW_ROUTINE_USAGE`；`STATISTICS` 新增 `EXPRESSION` |
| 80013 | 8.0.13 | 新增 `ST_UNITS_OF_MEASURE` |
| 80016 | 8.0.16 | 新增 `CHECK_CONSTRAINTS`；`SCHEMATA` 增 `DEFAULT_ENCRYPTION` |
| 80019 | （未发布） | WL#10895 新增 **6 张角色视图** |
| 80020 → 800201 | 8.0.19 误发 / 8.0.20 纠正 | 版本号"事故"被原样记在注释里，`static_assert` 为这个畸形值留了例外 |
| 80022 | 8.0.22 | 新增 `SCHEMATA_EXTENSIONS` |
| 80030 | 当前 | utf8mb3 重命名；6 张视图在 `show_gipk_in_create_table_and_information_schema=OFF` 时隐藏 GIPK |

⚠️ **8.0 全系列没有废弃/删除任何 I_S 视图**，只有新增与列定义变更。

**定义与那段"不许用未来版本号"的 `static_assert`**：

```cpp
/* sql/dd/info_schema/metadata.h */
static const uint IS_DD_VERSION = 80030;
static_assert((IS_DD_VERSION <= MYSQL_VERSION_ID) ||
                  ((IS_DD_VERSION == 800201) && (MYSQL_VERSION_ID >= 80020)),
              "This release can not use a version number from the future");
```

> ★ 那个 `IS_DD_VERSION == 800201` 的例外，就是 8.0.19 误发版本号事故留下的疤痕——**注释里明文记着**（见下表 80020 / 800201 两行）。

**版本变更历史注释的原文**（`metadata.h`，节选最有信息量的几段）：

```cpp
/**
  The version of the current information_schema system views.

  This version number is stored on disk in the data dictionary.
  Every time the information_schema structure changes,
  this version number must change.

  The numbering to use is the MySQL version number
  of the first MySQL version that published a given database schema.
  The format is Mmmdd with M=Major, m=minor, d=dot,
  so that MySQL 8.0.4 is encoded as 80004.
  ...
  80019: Not published (see below)
  ------------------------------------
  Changes from version 80018:

  - WL#10895 INFORMATION_SCHEMA views for Roles.
    Adds new system view definitions for roles.
       INFORMATION_SCHEMA.APPLICABLE_ROLES;
       INFORMATION_SCHEMA.ADMINISTRABLE_ROLE_AUTHORIZATIONS;
       INFORMATION_SCHEMA.ENABLED_ROLES;
       INFORMATION_SCHEMA.ROLE_TABLE_GRANTS;
       INFORMATION_SCHEMA.ROLE_COLUMN_GRANTS;
       INFORMATION_SCHEMA.ROLE_ROUTINE_GRANTS;

  80020: Published by mistake in server version 8.0.19. To correct this,
  we set the IS version number to 800201 in mysql server version 8.0.20.
  Then, in server version 8.0.21, we're back on track with IS_version 80021.
  ------------------------------------
  ...
  800201: Published in 8.0.20
  ------------------------------------
  Changes from version 80020:

  - Bug#30263373: INCORRECT OUTPUT FROM TABLE_FUNCTION_JSON::PRINT()
    This bug updates the character set of columns returned from JSON_TABLE
    expressions in INFORMATION_SCHEMA views.

  80021: Published in 8.0.21
  ------------------------------------
  Changes from version 800201:

     - WL#13562 CREATE/ALTER USER COMMENT 'JSON'

     - WL#13341: Store options for secondary engines.

     - Bug#30766181 and Bug#30216864 updates I_S.KEY_COLUMN_USAGE and
       I_S.TABLE_CONSTRAINTS views.

  80022: Published in 8.0.22
  ------------------------------------
  Changes from version 80021:

  - WL#13369: Added new I_S view 'SCHEMATA_EXTENSIONS'.
  ...
  80024..80028: Not published.
  ----------------------------------------------------------------------------
  There are no changes from version 80023. Hence server versions 80024..80028
  uses I_S version 80023.
  ...
  80030: Current.
  ------------------------------------
  Changes from version 80029:
  - Bug #33787300 Rename utf8_xxx collations to utf8mb3_xxx

  - WL#13784: Following INFORMATION_SCHEMA tables are modified to skip listing
              generated invisible primary key and key column when system
              variable show_gipk_in_create_table_and_information_schema = OFF,
                INFORMATION_SCHEMA.COLUMNS
                INFORMATION_SCHEMA.COLUMNS_EXTENSIONS
                INFORMATION_SCHEMA.KEY_COLUMN_USAGE
                INFORMATION_SCHEMA.STATISTICS
                INFORMATION_SCHEMA.TABLE_CONSTRAINTS
                INFORMATION_SCHEMA.TABLE_CONSTRAINTS_EXTENSIONS

  80031: Next IS version number after the previous is public.
  ------------------------------------
  Changes from version 80030:
*/
```

> ★ 注意"未发布版本沿用上一版本号"的做法（80014/80015 用 80013，80024~80028 用 80023）——**`IS_DD_VERSION` 不等于服务器版本**，它是"I_S 结构最后一次变更的发布版本"。WL#13341（8.0.21）是 `*_extensions` 的来由，WL#13784 是 GIPK（见 [`../../index/gipk.md`](../../index/gipk.md)）。

---

## 设计思想与理论基础

**① 用视图做"元数据 API"。**

元数据已经存在 `mysql.*` DD 表里了。要把它暴露成 SQL 标准的 `I_S.COLUMNS`（列名大写、有 `DATA_TYPE`/`CHARACTER_SET_NAME` 这类派生列），最省事的做法就是**写一条视图定义**：把 DD 表的列映射成标准列名，派生列用 SQL 表达式算。

好处是**零执行期代码**——不需要 `fill_table`、不需要遍历、不需要加锁协议（视图查询由常规 SQL 引擎处理，一致性由事务/MDL 保证）。代价是**灵活性差**：只能表达 SELECT 能算出来的东西。

**② 视图定义在 C++ 里拼，而不是写 SQL 文件。**

因为要用 `enum` 给字段编号、要复用 `fs_name_collation()` 这类依赖运行时全局变量的表达式拼接。代价是可读性差——COLUMNS 的定义就是一大串字符串拼接的 `CASE WHEN`。

**③ 与插件 I_S 的分工判据，不是"元数据 vs 运行时状态"，而是"能否用一条 SELECT 表达"。**

见「八」——`INNODB_TABLES` 是纯元数据却**不能**做成视图，这是最好的反证。

---

## 核心实现

### 一、机制：视图定义在 C++ 里拼出来

#### 1.1 三层类继承

```cpp
class System_view_definition_impl : public System_view_definition {
 public:
  virtual const String_type &view_name() const { return m_view_name; }
  virtual void set_view_name(const String_type &name) { m_view_name = name; }

  /**
    Get collation clause to append to view definition for some
    view columns based on lower_case_table_names.

    @return Empty string if lctn=0, other wise " COLLATE utf8mb3_tolower_ci".
  */
  static const String_type fs_name_collation() {
    if (lower_case_table_names != 0) return " COLLATE utf8mb3_tolower_ci";
    return "";
  }

  String_type build_ddl_create_view() const override = 0;
 private:
  String_type m_view_name;
};
```

- 第一层只管名字，外加一个 `fs_name_collation()`
- 第二层 `System_view_select_definition_impl` 是真正的 builder（`add_field` / `add_from` / `add_where` / `add_cte_expression` / `add_distinct` / `add_star`）
- 第三层 `System_view_union_definition_impl` —— **写了但完全没被使用**：

```cpp
// This class is unused and kept here for future use.
class System_view_union_definition_impl : public System_view_definition_impl {
```

⚠️ 注意 `fs_name_collation()` 是 **static**，但所有调用点都写成 `m_target_def.fs_name_collation()`（借实例调静态成员），且它依赖全局 `lower_case_table_names`——所以**视图 DDL 在运行时才定型**，同一份二进制在不同 `lctn` 下会生成不同的 CREATE VIEW。

#### 1.2 builder 的内部结构

```cpp
  // Map of field_names and the ordinal position in SELECT projection.
  typedef std::map<String_type, int> Field_numbers;
  // Map of field ordinal position and their view column definition.
  typedef std::map<int, String_type> Field_definitions;
  // List of FROM clause definition in the SELECT
  typedef std::vector<String_type> From_clauses;
  // List of WHERE clause definition in the SELECT
  typedef std::vector<String_type> Where_clauses;
```

注意：**没有 `add_join()`**——JOIN 是以字符串形式塞进 `add_from()` 的（如 `add_from("JOIN mysql.schemata sch ON tbl.schema_id=sch.id")`）。也没有 `add_group_by` / `add_order_by`。

`add_field()` 的实现（含查重断言与 `*` 特例）：

```cpp
  virtual void add_field(int field_number, const String_type &field_name,
                         const String_type &field_definition,
                         bool add_quotes = false) {
    // Make sure the field_number and field_name are not added twise.
    assert(m_field_numbers.find(field_name) == m_field_numbers.end() &&
           m_field_definitions.find(field_number) == m_field_definitions.end());

    m_field_numbers[field_name] = field_number;

    Stringstream_type ss;
    if (field_name == "*") {
      ss << " * ";
    } else {
      if (add_quotes) {
        assert(field_definition.find('\'') == String_type::npos);
        ss << '\'' << field_definition << '\'';
      } else
        ss << field_definition;

      ss << " AS " << field_name;
    }

    m_field_definitions[field_number] = ss.str();
  }
```

（注释里的 "twise" 是源码原文就有的拼写错误。注意这个断言是 `assert`，**release 构建下重复插入会静默覆盖**。）

#### 1.3 ★ `build_ddl_create_view()`：CREATE OR REPLACE

```cpp
  String_type build_ddl_create_view() const override {
    Stringstream_type ss;
    ss << "CREATE OR REPLACE DEFINER=`mysql.infoschema`@`localhost` VIEW "
       << "information_schema." << view_name() << " AS " + build_select_query();

    return ss.str();
  }
```

两个关键点：

- **`CREATE OR REPLACE`** —— 这就是重启/升级时"重跑全部 43 条 DDL"能**幂等**生效的原因（不需要先 DROP）
- **DEFINER 固定 `mysql.infoschema@localhost`**

`build_select_query()` 的拼装顺序：`[CTE] SELECT [DISTINCT] <投影，逗号换行分隔> FROM <多条，第一条加 FROM> [WHERE <多条，第一条加 WHERE>]`。

#### ★ 字段顺序按枚举，不是按插入顺序

因为 `m_field_definitions` 是 `std::map<int, String_type>`，`begin()→end()` 按 key（即 enum 值）**升序**迭代。

有硬证据：`Tables_base()` 构造时先 add `TABLE_CATALOG/SCHEMA/NAME/TYPE/ENGINE/VERSION/ROW_FORMAT`，再 add `CREATE_TIME/UPDATE_TIME/...`；而派生类 `Tables::Tables()` **后** add 的是 `TABLE_ROWS/AVG_ROW_LENGTH/DATA_LENGTH/.../AUTO_INCREMENT`。最终 DDL 里这些统计列却出现在 `ROW_FORMAT` 之后、`CREATE_TIME` 之前——与枚举 `FIELD_TABLE_ROWS=7 … FIELD_AUTO_INCREMENT=12` 完全吻合，**与构造顺序无关**。

⇒ 这个设计很巧：**允许多级继承分片填充定义，顺序仍由枚举统一裁决**。

**`add_field()` 的实现就是证据**——注意它是 `m_field_definitions[field_number] = ...`，即**用枚举当下标写进 map**，不是 `push_back`：

```cpp
/* sql/dd/impl/system_views/system_view_definition_impl.h
   —— System_view_select_definition_impl::add_field() */
  virtual void add_field(int field_number, const String_type &field_name,
                         const String_type &field_definition,
                         bool add_quotes = false) {
    // Make sure the field_number and field_name are not added twise.
    assert(m_field_numbers.find(field_name) == m_field_numbers.end() &&
           m_field_definitions.find(field_number) == m_field_definitions.end());

    m_field_numbers[field_name] = field_number;

    Stringstream_type ss;
    if (field_name == "*") {
      ss << " * ";
    } else {
      if (add_quotes) {
        assert(field_definition.find('\'') == String_type::npos);
        ss << '\'' << field_definition << '\'';
      } else
        ss << field_definition;

      ss << " AS " << field_name;
    }

    m_field_definitions[field_number] = ss.str();   /* ★ enum 下标 */
  }
```

容器定义（对比 FROM / WHERE 用 `vector`，即按调用顺序）：

```cpp
  // Map of field_names and the ordinal position in SELECT projection.
  typedef std::map<String_type, int> Field_numbers;

  // Map of field ordinal position and their view column definition.
  typedef std::map<int, String_type> Field_definitions;   /* ★ 有序，key 升序 */

  // List of FROM clause definition in the SELECT
  typedef std::vector<String_type> From_clauses;           /* ★ 按调用顺序 */

  // List of WHERE clause definition in the SELECT
  typedef std::vector<String_type> Where_clauses;

  Field_numbers m_field_numbers;
  Field_definitions m_field_definitions;
  From_clauses m_from_clauses;
  Where_clauses m_where_clauses;
```

生成 DDL 时**直接迭代 map，没有额外排序逻辑**，所以输出顺序 = key 升序 = 枚举顺序：

```cpp
    if (!m_add_star) {
      // Output view column definitions
      for (Field_definitions::const_iterator field = m_field_definitions.begin();
           field != m_field_definitions.end(); ++field) {
        if (field != m_field_definitions.begin()) ss << ",\n";
        ss << "  " << field->second;
      }
    } else
      ss << "*";
```

> ★ DD **物理表**那一侧是同一套手法（`Object_table_definition_impl` 里也是 `typedef std::map<int, String_type> Element_definitions;` + `(*element_definitions)[element_number] = element_definition;`）——**不是数组、不是 push_back**。两侧一致。

> ⚠️ 真名提醒：`add_field()` 在**派生类** `System_view_select_definition_impl` 上，**基类** `System_view_definition_impl` 只有 `view_name()` / `set_view_name()` / `fs_name_collation()` 三个成员，没有 `add_field()`。

⚠️ 顺带一条：多条 WHERE 是拼在一起的，第一条必须以不带 `AND` 的开头（通常是 `CAN_ACCESS_*`），第二及以后自带 `AND`。

#### 1.4 `m_target_def` vs `m_actual_def`

**系统视图侧只有 `m_target_def`，不存在 `m_actual_def`**。`m_actual_def` 只存在于 DD **物理表**体系 `Object_table_impl`：

```cpp
int Object_table_impl::field_number(const String_type &field_label) const {
  /*
    During upgrade, we must get the position of the field label from
    the actual definition. Otherwise, we get the position from the
    the target definition.
  */
  if (bootstrap::DD_bootstrap_ctx::instance().is_dd_upgrade())
    return m_actual_def.field_number(field_label);
  return m_target_def.field_number(field_label);
}
```

DD 物理表需要"目标定义 vs 磁盘实际定义"对比以支持升级时列位置漂移；**系统视图不需要**，因为它的版本管理是**整体粗粒度**的——版本变了就整包删掉重跑 `CREATE OR REPLACE VIEW`，不逐列比对。

### 二、43 张视图清单

`sql/dd/impl/system_views/` 下每个 `.cc` 对应一张（按文件名字母序）：

| | | |
|---|---|---|
| `administrable_role_authorizations` | `applicable_roles` | `character_sets` |
| `check_constraints` | `collation_charset_applicability` | `collations` |
| `column_statistics` | **`columns`** | `columns_extensions` |
| `enabled_roles` | `events` | `files` |
| `innodb_datafiles` | `innodb_fields` | `innodb_foreign` |
| `innodb_foreign_cols` | `innodb_tablespaces_brief` | `key_column_usage` |
| `keywords` | `parameters` | `partitions` |
| `referential_constraints` | `resource_groups` | `role_column_grants` |
| `role_routine_grants` | `role_table_grants` | `routines` |
| `schemata` | `schemata_extensions` | `st_geometry_columns` |
| `st_spatial_reference_systems` | `st_units_of_measure` | **`statistics`** |
| `table_constraints` | `table_constraints_extensions` | **`tables`** |
| `tables_extensions` | `tablespaces_extensions` | `triggers` |
| `user_attributes` | `view_routine_usage` | `view_table_usage` |
| `views` | | |

** ★ 计数的准确口径**（源码实测，容易记错）：

| 口径 | 数量 |
|---|---|
| `sql/dd/impl/system_views/` 下 `.h` 文件 | 45（含 2 个基础设施：`system_view_definition_impl.h`、`system_view_impl.h`） |
| 视图类头文件 / `.cc` | **43** |
| `register_view<>()` 调用 | **44** |
| 用户可见的 I_S 视图 | **43** |

差额来自 `statistics.h` 一个文件里定义了**两个类**——`Statistics` 与 `Show_statistics`，后者 `hidden()` 返回 `true`：

```cpp
/* sql/dd/impl/system_views/statistics.h —— 隐藏的那一个 */
class Show_statistics : public Statistics {
 public:
  ...
  // This view definition is hidden from user.
  bool hidden() const override { return true; }
};
```

MTR 用例里写的也是 44（`mysql-test/suite/information_schema/t/i_s_schema_definition_debug.test`）：

```
# Total number of system views in MySQL server.
let $expected_system_view_count = 44;
```

> ⚠️ 所以"43 张"只在**用户可见**的意义上成立；说"注册 43 行"或"43 个 .cc"都不准（前者 44，后者 43）。源码里**没有**任何 `static_assert` 或注释写死这个数字（grep `\b43\b` in `sql/dd` 命中 0）。

**`register_view<>()` 的完整 44 条**（`sql/dd/impl/system_registry.cc`，`System_views::init()`）：

```cpp
void System_views::init() {
  dd::System_views::Types is = dd::System_views::Types::INFORMATION_SCHEMA;
  dd::System_views::Types non_dd_based_is =
      dd::System_views::Types::NON_DD_BASED_INFORMATION_SCHEMA;

  register_view<dd::system_views::Enabled_roles>(non_dd_based_is);
  register_view<dd::system_views::Applicable_roles>(non_dd_based_is);
  register_view<dd::system_views::Administrable_role_authorizations>(
      non_dd_based_is);
  register_view<dd::system_views::Character_sets>(is);
  register_view<dd::system_views::Check_constraints>(is);
  register_view<dd::system_views::Collations>(is);
  register_view<dd::system_views::Collation_charset_applicability>(is);
  register_view<dd::system_views::Columns>(is);
  register_view<dd::system_views::Columns_extensions>(is);
  register_view<dd::system_views::Column_statistics>(is);
  register_view<dd::system_views::Events>(is);
  register_view<dd::system_views::Files>(is);
  register_view<dd::system_views::Innodb_datafiles>(is);
  register_view<dd::system_views::Innodb_foreign>(is);
  register_view<dd::system_views::Innodb_foreign_cols>(is);
  register_view<dd::system_views::Innodb_fields>(is);
  register_view<dd::system_views::Innodb_tablespaces_brief>(is);
  register_view<dd::system_views::Key_column_usage>(is);
  register_view<dd::system_views::Keywords>(is);
  register_view<dd::system_views::Parameters>(is);
  register_view<dd::system_views::Partitions>(is);
  register_view<dd::system_views::Referential_constraints>(is);
  register_view<dd::system_views::Resource_groups>(is);
  register_view<dd::system_views::Role_column_grants>(non_dd_based_is);
  register_view<dd::system_views::Role_routine_grants>(non_dd_based_is);
  register_view<dd::system_views::Role_table_grants>(non_dd_based_is);
  register_view<dd::system_views::Routines>(is);
  register_view<dd::system_views::Schemata>(is);
  register_view<dd::system_views::Schemata_extensions>(is);
  register_view<dd::system_views::Show_statistics>(is);
  register_view<dd::system_views::St_spatial_reference_systems>(is);
  register_view<dd::system_views::St_units_of_measure>(is);
  register_view<dd::system_views::St_geometry_columns>(is);
  register_view<dd::system_views::Statistics>(is);
  register_view<dd::system_views::Table_constraints>(is);
  register_view<dd::system_views::Table_constraints_extensions>(is);
  register_view<dd::system_views::Tables>(is);
  register_view<dd::system_views::Tables_extensions>(is);
  register_view<dd::system_views::Tablespaces_extensions>(is);
  register_view<dd::system_views::Triggers>(is);
  register_view<dd::system_views::View_routine_usage>(is);
  register_view<dd::system_views::View_table_usage>(is);
  register_view<dd::system_views::Views>(is);
  register_view<dd::system_views::User_attributes>(non_dd_based_is);
}
```

其中 `non_dd_based_is` 共 **7 张**：`Enabled_roles`、`Applicable_roles`、`Administrable_role_authorizations`、`Role_column_grants`、`Role_routine_grants`、`Role_table_grants`、`User_attributes`。

**分两批的原因**（`System_views::Types` 的注释原文）：

```cpp
  /*
    Classification of system views.

    Both of these types represent server INFORMATION_SCHEMA tables. The
    difference is that NON_DD_BASED_INFORMATION_SCHEMA indicates that the
    system view is based on ACL tables like role_edges, default_roles etc.
    NON_DD_BASED_INFORMATION_SCHEMA are created after creation of ACL
    tables defined in mysql_system_tables.sql and not with regular
    INFORMATION_SCHEMA tables that are created during bootstrap.
  */
  enum class Types { INFORMATION_SCHEMA, NON_DD_BASED_INFORMATION_SCHEMA };
```

（不逐个展开——机制相同，逐张复述是凑数。下面只深挖有代表性的几张。）

### 三、生命周期：注册、创建、版本

#### 3.1 注册：手写的 43 行 `register_view<X>()`

```cpp
void System_views::init() {
  dd::System_views::Types is = dd::System_views::Types::INFORMATION_SCHEMA;
  dd::System_views::Types non_dd_based_is =
      dd::System_views::Types::NON_DD_BASED_INFORMATION_SCHEMA;

  register_view<dd::system_views::Enabled_roles>(non_dd_based_is);
  ...
  register_view<dd::system_views::Character_sets>(is);
  ...
}
```

`X::instance()` 是函数内静态单例，**注册即构造**——构造函数里就把 `add_field`/`add_from` 全跑完了。

#### ★ 两类视图：为什么要分两批

```cpp
  /*
    Classification of system views.

    Both of these types represent server INFORMATION_SCHEMA tables. The
    difference is that NON_DD_BASED_INFORMATION_SCHEMA indicates that the
    system view is based on ACL tables like role_edges, default_roles etc.
    NON_DD_BASED_INFORMATION_SCHEMA are created after creation of ACL
    tables defined in mysql_system_tables.sql and not with regular
    INFORMATION_SCHEMA tables that are created during bootstrap.
  */
  enum class Types { INFORMATION_SCHEMA, NON_DD_BASED_INFORMATION_SCHEMA };
```

即：**角色类视图**（`APPLICABLE_ROLES` / `ENABLED_ROLES` / `ADMINISTRABLE_ROLE_AUTHORIZATIONS` / `ROLE_*_GRANTS` / `USER_ATTRIBUTES`）依赖 `mysql.role_edges`、`mysql.user` 这些 ACL 表，必须等 `mysql_system_tables.sql` 建完才能建。

#### 3.2 执行：遍历 + `execute_query`

```cpp
  for (dd::System_views::Const_iterator it =
           dd::System_views::instance()->begin(sv_type);
       it != dd::System_views::instance()->end();
       it = dd::System_views::instance()->next(it, sv_type)) {
    const dd::system_views::System_view_definition *view_def =
        (*it)->entity()->view_definition();

    // Build the CREATE VIEW DDL statement and execute it.
    if (view_def == nullptr ||
        dd::execute_query(thd, view_def->build_ddl_create_view())) {
      error = true;
      break;
    }
  }
```

**时机**由 `Dictionary_impl::init` 的五种 `dd_init` 分支决定（`DD_INITIALIZE_SYSTEM_VIEWS` / `DD_INITIALIZE_NON_DD_BASED_SYSTEM_VIEWS` / 升级路径等）。

执行时会**强制改会话状态**：

```cpp
  // Force use of utf8mb3 charset.
  ...
  thd->variables.character_set_client = m_client_cs;
  thd->variables.collation_connection = m_connection_cl;
  thd->update_charset();
```
```cpp
  /*
    Set tx_read_only to false to allow installing system views even
    if the server is started with --transaction-read-only=true.
  */
  thd->variables.transaction_read_only = false;
  thd->tx_read_only = false;
```

外层还有 `Disable_binlog_guard`（不写 binlog）。

#### 3.3 存哪：`dd::View` + `SYSTEM_VIEW`

```cpp
std::unique_ptr<View> v(dd::create_object<View>());
v->set_system_view(true);
```
```cpp
  void set_system_view(bool system_view) override {
    m_type =
        system_view ? enum_table_type::SYSTEM_VIEW : enum_table_type::USER_VIEW;
  }
```

即：**和普通视图共用 `mysql.tables` + DD 视图元数据**，只是 `type='SYSTEM VIEW'`。

⚠️ **但列元数据是从 `ST_SCHEMA_TABLE` 列表反推的，不是从视图 DDL 反推**：

```cpp
  ST_FIELD_INFO *fields_info = schema_table->fields_info;
  for (; fields_info->field_name; fields_info++) {
    dd::Column *col_obj = view_obj->add_column();
    col_obj->set_name(fields_info->field_name);
```

⇒ 这解释了一个坑：**改了视图 SQL 里的列类型而 `IS_VERSION` 不变时，用户侧看到的列类型会滞后**。

#### 3.4 ★ options：区分两套 I_S 的标记

```cpp
  dd::Properties *view_options = &view_obj->options();
  if (version != UNKNOWN_PLUGIN_VERSION)
    view_options->set(PLUGIN_VERSION_STRING, version);
  else
    view_options->set(SERVER_I_S_TABLE_STRING, true);
```

- **插件 I_S 表** → `options.plugin_version`
- **server 系统视图** → `options.server_i_s_table = true`

运行时正是用"**有没有 plugin_version**"来区分两者：

```cpp
    if (!view_options->exists(PLUGIN_VERSION_STRING)) {
      // Remove metadata from DD as I_S version is changed.
      error = dd::info_schema::remove_I_S_view_metadata(thd, view->name());
```

#### 3.5 hidden：43 张里只有一张

基类默认 `hidden() == false`。**唯一 override 成 true 的是 `SHOW_STATISTICS`**：

```cpp
/*
 The class represents system view definition used by SHOW STATISTICS when
*/
class Show_statistics : public Statistics {
 public:
  // This view definition is hidden from user.
  bool hidden() const override { return true; }
};
```

它与 `STATISTICS` 共用 `Statistics_base`，只多投影两列（`INDEX_ORDINAL_POSITION` / `COLUMN_ORDINAL_POSITION`）——**是给 `SHOW INDEX/KEYS` 内部用的，不对用户暴露**。用户直接查会报 `ER_NO_SYSTEM_VIEW_ACCESS`（除非带 `OPTION_SELECT_FOR_SHOW`）。

⚠️ **`I_S.FILES` 不是 hidden**——它走的是完全不同的 PROCESS 权限分支，两者是同一 `if` 块里并列的两条独立规则（见「六」）。

#### 3.6 版本：全局 `IS_VERSION` + 粗粒度重建

```cpp
static const uint IS_DD_VERSION = 80030;
static_assert((IS_DD_VERSION <= MYSQL_VERSION_ID) ||
                  ((IS_DD_VERSION == 800201) && (MYSQL_VERSION_ID >= 80020)),
              "This release can not use a version number from the future");
```

存在 DD properties 表的 `IS_VERSION` 键。重启时的判据与流程（函数头注释即设计说明）：

```
  Does following during server restart.

  - If I_S version is not changed, do nothing.
  - Else, remove all I_S metadata that belongs to server.
  - Store new server I_S metadata.
  - Recreate I_S system views, which will update old I_S metadata
    because we execute CREATE OR REPLACE VIEW.
```

⚠️ 一个边界：**DDSE 只读时不允许重建 I_S**（`check_if_server_ddse_readonly`）。

对比：插件 I_S 表**不走 `IS_VERSION`**，各自用 `plugin_version`，不匹配时单独删掉重建。

### 四、深挖：TABLES 与 STATISTICS

#### 4.1 `TABLES`

**JOIN**：`mysql.tables` → `schemata` → `catalogs` → `LEFT JOIN collations`（表默认排序规则）；统计派生类再加 `LEFT JOIN tablespaces` + `LEFT JOIN table_stats`。

**`TABLE_TYPE` 直接取 DD 枚举**，视图不做任何映射——DD 物理列定义就是：

```cpp
  m_target_def.add_field(FIELD_TYPE, "FIELD_TYPE",
                         "type ENUM('BASE TABLE', 'VIEW', 'SYSTEM VIEW')"
                         "NOT NULL");
```

**两处"视图才做不了、要在 SQL 里判"**：

```cpp
  m_target_def.add_field(FIELD_ENGINE, "ENGINE",
                         "IF(tbl.type = 'BASE TABLE', tbl.engine, NULL)");
  m_target_def.add_field(
      FIELD_VERSION, "VERSION",
      "IF(tbl.type = 'VIEW', NULL, 10 /* FRM_VER_TRUE_VARCHAR */)");
```

⚠️ `VERSION` 是硬编码的 **10**，注释直接标了来源宏 `FRM_VER_TRUE_VARCHAR`——**5.7 FRM 时代的遗留**。

**`CREATE_OPTIONS` 靠 UDF**——因为 `options` 是 `dd::Properties` 的 key=value 串，SQL 解析不了：

```cpp
      "IF (tbl.type = 'VIEW', NULL,"
      "  GET_DD_CREATE_OPTIONS(tbl.options,"
      "  IF(IFNULL(tbl.partition_expression, 'NOT_PART_TBL')='NOT_PART_TBL',"
      "     0, 1), IF(sch.default_encryption='YES',1,0)))"
```

**统计列走 UDF + 缓存表**：

```cpp
  m_target_def.add_field(FIELD_TABLE_ROWS, "TABLE_ROWS",
                         "IF (tbl.type = 'VIEW', NULL,"
                         "INTERNAL_TABLE_ROWS(sch.name, tbl.name,"
                         "  IF(ISNULL(tbl.partition_type), tbl.engine, ''),"
                         "  tbl.se_private_id, tbl.hidden != 'Visible', "
                         "  ts.se_private_data,"
                         "  COALESCE(stat.table_rows, 0),"
                         "  COALESCE(CAST(stat.cached_time as UNSIGNED), 0)))");
```

两点值得注意：

- 分区表传 engine 时**故意传 `''`**（`IF(ISNULL(tbl.partition_type), tbl.engine, '')`）——分区表统计要由 SE 按分区算
- 传 `ts.se_private_data` 的注释：*"which is used by SE to read the SE specific tablespace metadata ... E.g., InnoDB would read the SE specific space_id from se_private_data column"*

还有一条 Bug workaround 注释：

```cpp
  /*
    stat.<value> and stat.cached_time should be passed directly to UDFs
    and UDF implementation should handle NULL value. Using IFNULL() as a
    workaround until Bug#26389402 is fixed.
  */
```

**WHERE**：`CAN_ACCESS_TABLE(sch.name, tbl.name)` + `IS_VISIBLE_DD_OBJECT(tbl.hidden)`。

#### 4.2 `STATISTICS`

**Cardinality**（只有 `Statistics` 派生类加）：

```cpp
  m_target_def.add_field(
      FIELD_CARDINALITY, "CARDINALITY",
      "INTERNAL_INDEX_COLUMN_CARDINALITY(sch.name, tbl.name, idx.name,"
      "col.name, idx.ordinal_position,"
      "icu.ordinal_position,"
      "IF(ISNULL(tbl.partition_type), tbl.engine, ''),"
      "tbl.se_private_id,"
      "tbl.hidden != 'Visible' OR idx.hidden OR icu.hidden,"
      "COALESCE(stat.cardinality, CAST(-1 AS UNSIGNED)),"
      "COALESCE(CAST(stat.cached_time as UNSIGNED), 0))");

  m_target_def.add_from(
      "LEFT JOIN mysql.index_stats stat"
      "  ON tbl.name=stat.table_name"
      " AND sch.name=stat.schema_name"
      " AND idx.name=stat.index_name"
      " AND col.name=stat.column_name");
```

★ **缓存未命中时传 `-1`（`CAST(-1 AS UNSIGNED)`）而不是 0** —— 这是"未知"的哨兵值。

**`SUB_PART`（前缀索引长度）** 不能直接取 `icu.length`，要按列类型转换：

```cpp
  m_target_def.add_field(FIELD_SUB_PART, "SUB_PART",
                         "GET_DD_INDEX_SUB_PART_LENGTH(icu.length,"
                         "col.type, col.char_length, col.collation_id,"
                         "idx.type)");
```

**两个容易踩的细节**：

```cpp
  m_target_def.add_field(FIELD_NULLABLE, "NULLABLE",
                         "IF (col.is_nullable = 1, 'YES','')");
```
（是**空串**不是 `'NO'`——这是 SQL 标准对 `NULLABLE` 的要求。）

```cpp
  m_target_def.add_field(FIELD_INDEX_TYPE, "INDEX_TYPE",
                         "CASE WHEN idx.type = 'SPATIAL' THEN 'SPATIAL' "
                         "WHEN idx.algorithm = 'SE_PRIVATE' THEN '' "
                         "ELSE idx.algorithm END ");
```
（SPATIAL 看 `type`；**SE 私有索引**（InnoDB 内部 clustered index）映射为**空串**。）

**函数索引的处理**是全篇最绕的：

```cpp
      FIELD_COLUMN_NAME, "COLUMN_NAME",
      "IF (col.hidden = 'SQL', NULL, col.name COLLATE utf8mb3_tolower_ci)"
      ...
      FIELD_EXPRESSION, "EXPRESSION",
      "IF (col.hidden = 'SQL', col.generation_expression_utf8, NULL)"
```

即：函数索引（`col.hidden='SQL'`）**不显示列名，改显示 `EXPRESSION`**。

### 五、深挖：PARTITIONS 与 COLUMN_STATISTICS

#### 5.1 `PARTITIONS`

**`PARTITION_METHOD`** —— DD 存的是内部枚举，视图负责翻成 SQL 标准写法：

```cpp
  m_target_def.add_field(FIELD_PARTITION_METHOD, "PARTITION_METHOD",
                         "CASE tbl.partition_type"
                         "  WHEN 'HASH' THEN 'HASH' "
                         "  WHEN 'RANGE' THEN 'RANGE' "
                         "  WHEN 'LIST' THEN 'LIST' "
                         "  WHEN 'AUTO' THEN 'AUTO' "
                         "  WHEN 'KEY_51' THEN 'KEY' "
                         "  WHEN 'KEY_55' THEN 'KEY' "
                         "  WHEN 'LINEAR_KEY_51' THEN 'LINEAR KEY' "
                         "  WHEN 'LINEAR_KEY_55' THEN 'LINEAR KEY' "
                         "  WHEN 'LINEAR_HASH' THEN 'LINEAR HASH' "
                         "  WHEN 'RANGE_COLUMNS' THEN 'RANGE COLUMNS' "
                         "  WHEN 'LIST_COLUMNS' THEN 'LIST COLUMNS' "
                         "  ELSE NULL END");
```

（`_51`/`_55` 是 5.1/5.5 的 KEY 分区算法差异，对外统一叫 `KEY`。）

**`PARTITION_EXPRESSION`** —— **没有用任何函数处理**，而是直接取 DD 里的 `*_utf8` 列：

```cpp
  m_target_def.add_field(FIELD_PARTITION_EXPRESSION, "PARTITION_EXPRESSION",
                         "tbl.partition_expression_utf8");
```

DD 里同时存在 `partition_expression`（BLOB，原始字节）与 `partition_expression_utf8`（转码后的 LONGTEXT）——**视图只取 utf8 版本**，这就是全部的"处理"。

**分区树的自 JOIN**（分区 → 子分区 → 各自表空间）：

```cpp
  m_target_def.add_from(
      "LEFT JOIN mysql.table_partitions part ON "
      "part.table_id=tbl.id");
  m_target_def.add_from(
      "LEFT JOIN mysql.table_partitions sub_part "
      "ON sub_part.parent_partition_id=part.id");
  ...
  m_target_def.add_where("AND part.parent_partition_id IS NULL");
```

★ **分区统计不走缓存**——给 UDF 传的缓存值位置**直接是 `0, 0`**，即每个分区**实时问 SE**：

```cpp
                         "  IF(sub_part.name IS NULL, "
                         "     IF(part.name IS NULL, tbl.se_private_data, "
                         "part_ts.se_private_data),"
                         "     sub_part_ts.se_private_data), 0, 0,"
                         "  IFNULL(sub_part.name, part.name))"
```

#### 5.2 ★ `COLUMN_STATISTICS`：43 张里最薄的一张

**完全没有 UDF、没有 CASE、没有 JOIN、没有缓存**：

```cpp
Column_statistics::Column_statistics() {
  m_target_def.set_view_name(view_name());

  m_target_def.add_field(FIELD_SCHEMA_NAME, "SCHEMA_NAME", "SCHEMA_NAME");
  m_target_def.add_field(FIELD_TABLE_NAME, "TABLE_NAME", "TABLE_NAME");
  m_target_def.add_field(FIELD_COLUMN_NAME, "COLUMN_NAME", "COLUMN_NAME");
  m_target_def.add_field(FIELD_HISTOGRAM, "HISTOGRAM", "HISTOGRAM");
  m_target_def.add_from("mysql.column_statistics");
  m_target_def.add_where("CAN_ACCESS_TABLE(SCHEMA_NAME, TABLE_NAME)");
}
```

**`HISTOGRAM` 是 DD 物理表里的一个真 JSON 列**：

```cpp
  m_target_def.add_field(FIELD_HISTOGRAM, "FIELD_HISTOGRAM",
                         "histogram JSON NOT NULL");
```

⇒ 这带来三条与其他视图**本质不同**的性质：

1. 直方图由 **`ANALYZE TABLE ... UPDATE HISTOGRAM` 显式生成并持久化**，不是缓存也不是估算 ⇒ **不受 `information_schema_stats_expiry` 影响**
2. 它是唯一一张**不 JOIN `mysql.tables`** 的"表级"视图——三个名字就是 DD 表自己的列，所以 `CAN_ACCESS_TABLE` 直接吃列名
3. 没有 `TABLE_CATALOG` 列（不 JOIN `schemata`/`catalogs`）

（对比：`TABLES.TABLE_ROWS` 走 `mysql.table_stats` + `cached_time`，`STATISTICS.CARDINALITY` 走 `mysql.index_stats` + `cached_time`——**这两个会过期**。）

### 六、深挖：FILES 与存储程序类

#### 6.1 `FILES`：半张插件表

**它是唯一被解析器按名字硬编码特殊对待的 I_S 表**——整个 `sql_parse.cc` 里只有这一处：

```cpp
        /*
          Stop users from accessing I_S.FILES if they do not have
          PROCESS privilege.
        */
        if (!strcmp(ptr->table_name, "FILES") &&
            check_global_access(thd, PROCESS_ACL))
          return nullptr;
```

**实现上它是"半张插件表"**——FROM 只 JOIN 两张 DD 表，但**绝大多数列是 `INTERNAL_TABLESPACE_*` UDF**（共 14 个：ID / TYPE / LOGFILE_GROUP_NAME / FREE_EXTENTS / TOTAL_EXTENTS / EXTENT_SIZE / INITIAL_SIZE / MAXIMUM_SIZE / AUTOEXTEND_SIZE / VERSION / ROW_FORMAT / DATA_FREE / STATUS / EXTRA）：

```cpp
  m_target_def.add_from("mysql.tablespaces ts");
  m_target_def.add_from(
      "JOIN mysql.tablespace_files tsf "
      "ON ts.id=tsf.tablespace_id");
```

所有 UDF 签名一致：`(ts.name, tsf.file_name, ts.engine, ts.se_private_data)`——**DD 只提供"坐标"，数据由 SE 现算**。

一批"历史包袱列"直接硬写 NULL：`TABLE_SCHEMA`、`FULLTEXT_KEYS`、`DELETED_ROWS`、`UPDATE_COUNT`、`TABLE_ROWS`、`CHECKSUM` 等。

`FILE_NAME` 是唯一纯 SQL 拼装的列，带一条明确的"已知不一致"注释：

```cpp
  /*
    Fix for Bug#26518545 might change below code. Mainly because the
    difference in DD not containing './' and filename maintained by InnoDB
    containing './' might change after the fix.
   */
```

#### 6.2 存储程序类：`ROUTINES` / `EVENTS` / `TRIGGERS`

**共同难点 ①：定义文本存两份。**

```cpp
  m_target_def.add_field(FIELD_DEFINITION, "FIELD_DEFINITION",
                         "definition LONGBLOB");
  m_target_def.add_field(FIELD_DEFINITION_UTF8, "FIELD_DEFINITION_UTF8",
                         "definition_utf8 LONGTEXT");
```

视图一律取 `*_utf8` —— 这就是 I_S 能把程序体当 utf8mb3 列直接吐出来的原因，省掉了运行时转码。

**共同难点 ②：要还原"创建时的客户端环境"，所以 JOIN 一大堆字符集表。** `ROUTINES` 一共 8 条 FROM：`coll_client` / `cs_client` / `coll_conn` / `coll_db` 是 INNER JOIN，`coll_result` / `cs_result` 是 **LEFT JOIN**（因为 PROCEDURE 没有返回类型）。

**★ 共同难点 ③：权限过滤分两层。** `ROUTINES` 是唯一把 `CAN_ACCESS_ROUTINE` **用两次**的：

```cpp
  m_target_def.add_field(
      FIELD_ROUTINE_DEFINITION, "ROUTINE_DEFINITION",
      "IF (CAN_ACCESS_ROUTINE(sch.name, rtn.name, rtn.type, rtn.definer, TRUE),"
      "    rtn.definition_utf8, NULL)");
  ...
  m_target_def.add_where(
      "CAN_ACCESS_ROUTINE(sch.name, rtn.name, rtn.type, "
      "rtn.definer, FALSE)");
```

WHERE 里一次决定**行是否可见**，投影里一次决定 **`ROUTINE_DEFINITION` 是否给 NULL**——第 5 个参数 `TRUE/FALSE` 就是这个开关。

**`EVENTS` 独有的难点：时区**（DD 一律存 UTC）：

```cpp
  m_target_def.add_field(FIELD_EXECUTE_AT, "EXECUTE_AT",
                         "CONVERT_TZ(evt.execute_at,'+00:00', evt.time_zone)");
  ...
  m_target_def.add_field(FIELD_INTERVAL_VALUE, "INTERVAL_VALUE",
                         "CONVERT_INTERVAL_TO_USER_INTERVAL(evt.interval_value,"
                         "evt.interval_field)");
```

**`TRIGGERS`**：要排除视图上的触发器（`WHERE tbl.type != 'VIEW'`），且 `ACTION_CONDITION` 恒 NULL。

### 七、深挖：KEY_COLUMN_USAGE 与 KEYWORDS

#### 7.1 `KEY_COLUMN_USAGE`：43 张里 SQL 最复杂的一张

**唯一用了 `LATERAL` + 内联 `UNION ALL`**：

```cpp
  m_target_def.add_from(
      ", LATERAL (SELECT"
      "    idx.name AS CONSTRAINT_NAME,"
      "    ... "
      "    FROM mysql.indexes idx"
      "    JOIN mysql.index_column_usage icu ON icu.index_id = idx.id"
      "    WHERE idx.table_id = tbl.id"
      "      AND idx.type IN ('PRIMARY', 'UNIQUE')"
      "  UNION ALL SELECT"
      "    fk.name COLLATE utf8mb3_tolower_ci AS CONSTRAINT_NAME,"
      "    ... "
      "    FROM mysql.foreign_keys fk"
      "    JOIN mysql.foreign_key_column_usage fkcu ON fkcu.foreign_key_id = "
      "fk.id"
      "    WHERE fk.table_id = tbl.id"
      ") constraints");
```

设计意图很清晰：**主键/唯一键 与 外键 是 DD 里两套完全无关的表**，用 LATERAL 子查询把它们 UNION 成统一的 `constraints` 行源，再统一 JOIN `columns`。

对比 `REFERENTIAL_CONSTRAINTS` 就极其朴素——只 JOIN 4 张表、**零 CASE、零 UDF**，所有值都是 DD 原生列/枚举。

#### 7.2 ★ `KEYWORDS`：构建期代码生成 + 运行期 JSON_TABLE

**它不是硬编码的**，数据来源是**生成的** `keyword_list[]`。生成器 `gen_keyword_list.cc` 读的是词法器的 `symbols[]` **加上 yacc 文件**：

```cpp
  for (size_t i = 0; i < array_elements(symbols); i++) {
    const SYMBOL *sym = &symbols[i];

    if (sym->group != SG_KEYWORDS && sym->group != SG_HINTABLE_KEYWORDS)
      continue;  // Function or optimizer hint name.

    if (!isalpha(sym->name[0])) continue;  // Operator.

    bool is_reserved = keyword_tokens.count(sym->tok) == 0;
```

★ **`RESERVED` 的判定标准**：在 yacc 里**没有**以 `<lexer.keyword>` 语义类型声明的 token 就是保留字（生成器用正则扫 `%token <lexer.keyword> NAME NUMBER`）。这是一个非常"源码级"的判据。

**视图构造**：把整个 `keyword_list[]` 在 C++ 侧序列化成一个 JSON 数组字面量塞进 FROM，再用 `JSON_TABLE` 展开：

```cpp
  ss << "JSON_TABLE('[";
  for (auto x : keyword_list)
    ss << "[\"" << x.word << "\"," << x.reserved << "],";
  ss.seekp(ss.tellp() - static_cast<std::streamoff>(1));  // remove last ','
  ss << "]', '$[*]' COLUMNS(word VARCHAR(" << MAX_WORD_SIZE
     << ") CHARSET utf8mb4 PATH '$[0]',"
     << "reserved INT PATH '$[1]')) AS j";
```

⇒ **`I_S.KEYWORDS` 的 CREATE VIEW 语句是一条约 2000 个关键字那么长的巨型 JSON 字面量**。`MAX_WORD_SIZE = 128` 是硬上限，debug 下有 assert 兜底。

### 八、深挖：5 张 `innodb_*` 与 8 张 `*_extensions`

#### 8.1 5 张 `innodb_*`：名字最坑的一组

它们**是视图**（从 DD 读，走本篇机制），而 `INNODB_TRX` / `INNODB_BUFFER_PAGE` 那些**是插件表**（走 `feat/i_s.md` 的机制）。**名字都叫 `INNODB_*`，机制完全不同。**

| 视图 | 数据来源 |
|---|---|
| `innodb_datafiles` | `mysql.tablespace_files` |
| `innodb_fields` | `mysql.columns` 等 |
| `innodb_foreign` / `innodb_foreign_cols` | `mysql.foreign_keys` / `mysql.foreign_key_column_usage` |
| `innodb_tablespaces_brief` | `mysql.tablespaces` |

> **考古点**：`INNODB_SYS_FOREIGN` / `INNODB_SYS_DATAFILES` 这套 5.x 的表在 8.0 已移除（插件侧只剩三个孤儿 `extern` 声明，见 [`../../feat/i_s.md`](../../infra/observability/i_s.md)），**取而代之的正是这里从 DD 读的视图**。

**定义源码实证**（两张代表，FROM 全部是 `mysql.*` DD 表，过滤条件统一为 `engine='INNODB'` + `se_private_id IS NOT NULL` + `hidden='Visible'`）：

```cpp
/* sql/dd/impl/system_views/innodb_datafiles.cc */
Innodb_datafiles::Innodb_datafiles() {
  m_target_def.set_view_name(view_name());

  m_target_def.add_field(
      FIELD_SPACE, "SPACE",
      "GET_DD_TABLESPACE_PRIVATE_DATA(ts.se_private_data, 'id')");
  m_target_def.add_field(FIELD_PATH, "PATH", "ts_files.file_name");

  m_target_def.add_from("mysql.tablespace_files ts_files");
  m_target_def.add_from(
      "JOIN mysql.tablespaces ts ON "
      "ts.id=ts_files.tablespace_id");

  m_target_def.add_where("ts.se_private_data IS NOT NULL");
  m_target_def.add_where("AND ts.engine='InnoDB'");
  m_target_def.add_where("AND ts.name<>'mysql'");
  m_target_def.add_where("AND ts.name<>'innodb_temporary'");
}
```

```cpp
/* sql/dd/impl/system_views/innodb_fields.cc */
Innodb_fields::Innodb_fields() {
  m_target_def.set_view_name(view_name());

  m_target_def.add_field(
      FIELD_INDEX_ID, "INDEX_ID",
      "GET_DD_INDEX_PRIVATE_DATA(idx.se_private_data, 'id')");
  m_target_def.add_field(FIELD_NAME, "NAME", "col.name");
  m_target_def.add_field(FIELD_POS, "POS", "fld.ordinal_position - 1");

  m_target_def.add_from("mysql.index_column_usage fld");
  m_target_def.add_from("JOIN mysql.columns col ON fld.column_id=col.id");
  m_target_def.add_from("JOIN mysql.indexes idx ON fld.index_id=idx.id");
  m_target_def.add_from("JOIN mysql.tables tbl ON tbl.id=idx.table_id");

  m_target_def.add_where("NOT tbl.type = 'VIEW'");
  m_target_def.add_where("AND tbl.hidden = 'Visible'");
  m_target_def.add_where("AND NOT fld.hidden");
  m_target_def.add_where("AND tbl.se_private_id IS NOT NULL");
  m_target_def.add_where("AND tbl.engine='INNODB'");
}
```

两个值得一提的细节：

- **`GET_DD_*_PRIVATE_DATA(se_private_data, 'id')`** —— 空间 ID / 索引 ID 存在 `se_private_data` 这个 Properties 串里，SQL 解析不了，只能靠 UDF 取。这正是「工程技法 3」说的"UDF 是视图的逃生舱"
- **`POS = ordinal_position - 1`** —— I_S 用 0-based，DD 里是 1-based，视图里做转换

`innodb_tablespaces_brief` 的 `SPACE_TYPE` 更典型——靠 `flags` 的 bit 11（2048）判 General/Single：

```cpp
  m_target_def.add_field(
      FIELD_SPACE_TYPE, "SPACE_TYPE",
      "IF(GET_DD_TABLESPACE_PRIVATE_DATA(ts.se_private_data, 'id')=0, 'System', "
      "   IF((GET_DD_TABLESPACE_PRIVATE_DATA(ts.se_private_data, 'flags')&2048)>>11 != 0, "
      "      'General', 'Single'))");
```

`innodb_foreign` 甚至把 `GROUP BY` 塞进了 `add_where()`（因为 `add_where` 只是 push_back 字符串，不做语义检查）：

```cpp
  m_target_def.add_where("GROUP BY fk.id");
```

⇒ 这一组正是「九、选型判据」里"InnoDB 元数据被劈成两半"的**直接实证**：进了 DD 的那部分做成这 5 张视图。

#### 8.2 5 张 `*_extensions`（8.0.21+）：`engine_attribute` 的暴露口

> ⚠️ **纠错**：不是 8 张，是 **5 张**（`sql/dd/impl/system_views/` 下 grep `*_extensions*` 实测）：
> `columns_extensions` / `schemata_extensions` / `table_constraints_extensions` / `tables_extensions` / `tablespaces_extensions`。
> 其中含 `engine_attribute` 的是 4 张（`schemata_extensions` 暴露的是 schema 的其它属性，没有 engine_attribute——schema 粒度本来也没有引擎属性）。

**它们为什么存在**——源码注释直说：小版本不能改 I_S 视图的列集，所以新列只能开新视图：

```cpp
/**
   The class representing INFORMATION_SCHEMA.TABLES_EXTENSIONS system
   view definition.

   Contains additional metadata about tables which cannot be added to
   INFORMATION_SCHEMA.TABLES in a minor release.
*/
```

`tables_extensions` 的完整定义（直接取 `mysql.tables` 的两列）：

```cpp
/* sql/dd/impl/system_views/tables_extensions.cc */
namespace {
enum {
  FIELD_TABLE_CATALOG,
  FIELD_TABLE_SCHEMA,
  FIELD_TABLE_NAME,
  FIELD_ENGINE_ATTRIBUTE,
  FIELD_SECONDARY_ENGINE_ATTRIBUTE
};

const dd::String_type s_view_name{STRING_WITH_LEN("TABLES_EXTENSIONS")};
const dd::system_views::Tables_extensions *s_instance =
    new dd::system_views::Tables_extensions(s_view_name);
}  // namespace

Tables_extensions::Tables_extensions(const dd::String_type &n) {
  m_target_def.set_view_name(n);

  m_target_def.add_field(FIELD_TABLE_CATALOG, "TABLE_CATALOG",
                         "cat.name" + m_target_def.fs_name_collation());
  m_target_def.add_field(FIELD_TABLE_SCHEMA, "TABLE_SCHEMA",
                         "sch.name" + m_target_def.fs_name_collation());
  m_target_def.add_field(FIELD_TABLE_NAME, "TABLE_NAME",
                         "tbl.name" + m_target_def.fs_name_collation());

  // SELECT extension fields
  m_target_def.add_field(FIELD_ENGINE_ATTRIBUTE, "ENGINE_ATTRIBUTE",
                         "tbl.engine_attribute");
  m_target_def.add_field(FIELD_SECONDARY_ENGINE_ATTRIBUTE,
                         "SECONDARY_ENGINE_ATTRIBUTE",
                         "tbl.secondary_engine_attribute");

  m_target_def.add_from("mysql.tables tbl");
  m_target_def.add_from("JOIN mysql.schemata sch ON tbl.schema_id=sch.id");
  m_target_def.add_from("JOIN mysql.catalogs cat ON cat.id=sch.catalog_id");

  m_target_def.add_where("CAN_ACCESS_TABLE(sch.name, tbl.name)");
  m_target_def.add_where("AND IS_VISIBLE_DD_OBJECT(tbl.hidden)");
}
```

> ★ 两个工程细节：① 这里的 `enum` 放在**文件级匿名 namespace**里（不是类内 `enum_fields`），但照样当 `add_field` 的编号用；② 视图名用 `const dd::String_type s_view_name{STRING_WITH_LEN(...)}` 在 `.cc` 里定义，避免"头文件里有静态 std::string"的初始化顺序问题。

**`engine_attribute` 是真正的列，不是 Properties**——DD 物理表里的定义就是 MySQL 原生 `JSON` 列：

```cpp
/* sql/dd/impl/tables/tables.cc */
  m_target_def.add_field(FIELD_ENGINE_ATTRIBUTE, "FIELD_ENGINE_ATTRIBUTE",
                         "engine_attribute JSON");
  m_target_def.add_field(FIELD_SECONDARY_ENGINE_ATTRIBUTE,
                         "FIELD_SECONDARY_ENGINE_ATTRIBUTE",
                         "secondary_engine_attribute JSON");
```

DD 对象接口返回的是 `LEX_CSTRING`（不是 `Properties&`），impl 成员是 `dd::String_type`：

```cpp
/* sql/dd/types/table.h */
  virtual LEX_CSTRING engine_attribute() const = 0;
  virtual void set_engine_attribute(LEX_CSTRING attrs) = 0;
  virtual LEX_CSTRING secondary_engine_attribute() const = 0;
  virtual void set_secondary_engine_attribute(LEX_CSTRING attrs) = 0;
```

```cpp
/* sql/dd/impl/types/table_impl.h —— 成员区，与 m_options/m_se_private_data 对比鲜明 */
  Properties_impl m_se_private_data;          /* ★ Properties 串 */
  // SE-specific json attributes
  dd::String_type m_engine_attribute;         /* ★ 独立列 */
  dd::String_type m_secondary_engine_attribute;
```

DD 版本记录也印证它是 8.0.21 新增的列（`sql/dd/dd_version.h`）：

```
  - WL#13341 adds new columns
      mysql.tables.engine_attribute
      mysql.tables.secondary_engine_attribute
      mysql.columns.engine_attribute
      mysql.columns.secondary_engine_attribute
      mysql.indexes.engine_attribute
      mysql.indexes.secondary_engine_attribute
      mysql.tablespaces.engine_attribute
```

> ⚠️ `Tablespace` **只有** `engine_attribute`，**没有** `secondary_engine_attribute`（表空间没有 secondary engine 概念）。

语法与落库路径（`sql_yacc.yy` → `parse_tree_nodes.cc` → `dd_table.cc`）：

```yacc
        | ENGINE_ATTRIBUTE_SYM opt_equal json_attribute
          {
            $$ = make_table_engine_attribute(YYMEM_ROOT, $3);
          }
```

```cpp
/* sql/parse_tree_nodes.cc */
PT_create_table_option *make_table_engine_attribute(MEM_ROOT *mem_root,
                                                    LEX_CSTRING attr) {
  return new (mem_root) PT_attribute<LEX_CSTRING, PT_create_table_option>(
      attr, +[](LEX_CSTRING a, Table_ddl_parse_context *pc) {
        pc->create_info->engine_attribute = a;
        pc->create_info->used_fields |= HA_CREATE_USED_ENGINE_ATTRIBUTE;
        pc->alter_info->flags |=
            (Alter_info::ALTER_OPTIONS | Alter_info::ANY_ENGINE_ATTRIBUTE);
        return false;
      });
}
```

★ 由此必须区分 DD 里的**三种"自定义数据"**——这是理解 DD 扩展性的关键，也极易混淆：

| | 位置 | 谁写 / 谁用 | 特点 |
|---|---|---|---|
| `options` | 对象的 `options` 列（`dd::Properties`） | server 自己 | **有白名单**（见 `dd.md`） |
| `se_private_data` | 对象的 `se_private_data` 列（`dd::Properties`） | **存储引擎自管** | **无白名单**；InnoDB 全键名表见 [`dd.md`](dd.md) |
| **`engine_attribute`** | **独立列**（不在 Properties 里） | **用户**通过 `ALTER TABLE ... ENGINE_ATTRIBUTE='...'` 设置 | 类型安全、可被 SQL 直接查；透传给引擎 / secondary engine |

⚠️ 判据很直接：**前两者是 Properties 串（弱类型、SQL 不可解析），第三个是真正的列**。`GET_DD_CREATE_OPTIONS` 那类 UDF 存在的原因，正是因为前两者 SQL 解析不了（见「四」）。

### 九、选型判据：为什么有些表不做成视图

#### 判据不是"像不像元数据"，而是"能否用一条 SELECT 从 `mysql.*` 算出来"

三条源码证据：

**① DD 眼里两者是同一种东西**——都是 `dd::View` + `type=SYSTEM_VIEW` + `schema=information_schema`，区别**只在 options**（`plugin_version` vs `server_i_s_table`）。**"视图 vs 插件表"这个划分对 DD 根本不存在**，真正的区别在**数据从哪来**。

**② 解析器走完全不同的分发路径**：

```cpp
    if (ptr->is_system_view) {
      ...  // 视图：展开视图定义
    } else {
      schema_table = find_schema_table(thd, ptr->table_name);
      // → 物化成临时表
```

**③ 权限模型的根本差异**：视图把权限**下推到 `CAN_ACCESS_*` UDF**（`CAN_ACCESS_TABLE` / `_COLUMN` / `_ROUTINE` / `_TRIGGER` / `_EVENT` / `_USER`），因为必须在**逐行**粒度做（同一张 `I_S.COLUMNS` 里 A 能看到 a 表、看不到 b 表）；插件表的 `fill_table` 是 C++ 回调，天然只能**整表**粒度。

**★ "只能被系统视图调用"的门禁**（`sql/item_create.cc`）——这是这类函数最关键的一处代码：

```cpp
/* 宏：Like SQL_FN, but for functions that may only be referenced from system views. */
#define SQL_FN_INTERNAL(F, N) \
  &Internal_function_factory<Instantiator<F, N>>::s_singleton

template <typename Instantiator_fn>
class Internal_function_factory : public Create_func {
 public:
  Item *create_func(THD *thd, LEX_STRING function_name,
                    PT_item_list *item_list) override {
    if (!thd->parsing_system_view && !thd->is_dd_system_thread() &&
        DBUG_EVALUATE_IF("skip_dd_table_access_check", false, true)) {
      my_error(ER_NO_ACCESS_TO_NATIVE_FCT, MYF(0), function_name.str);
      return nullptr;                            /* ★ 用户手写会被拒 */
    }
    ...
  }
};
```

注册表里的完整清单（`sql/item_create.cc`）：

```cpp
    {"CAN_ACCESS_DATABASE", SQL_FN_INTERNAL(Item_func_can_access_database, 1)},
    {"CAN_ACCESS_TABLE", SQL_FN_INTERNAL(Item_func_can_access_table, 2)},
    {"CAN_ACCESS_COLUMN", SQL_FN_INTERNAL(Item_func_can_access_column, 3)},
    {"CAN_ACCESS_VIEW", SQL_FN_INTERNAL(Item_func_can_access_view, 4)},
    {"CAN_ACCESS_TRIGGER", SQL_FN_INTERNAL(Item_func_can_access_trigger, 2)},
    {"CAN_ACCESS_ROUTINE", SQL_FN_LIST_INTERNAL(Item_func_can_access_routine, 5)},
    {"CAN_ACCESS_EVENT", SQL_FN_INTERNAL(Item_func_can_access_event, 1)},
    {"CAN_ACCESS_USER", SQL_FN_INTERNAL(Item_func_can_access_user, 2)},
    {"CAN_ACCESS_RESOURCE_GROUP",
     SQL_FN_INTERNAL(Item_func_can_access_resource_group, 1)},
    {"GET_DD_COLUMN_PRIVILEGES",
     SQL_FN_INTERNAL(Item_func_get_dd_column_privileges, 3)},
```

> ⚠️ **术语纠正**：它们严格说是 **native / internal function**（工厂是 `Internal_function_factory`），**不是** `CREATE FUNCTION` 那种 UDF（那是 `Create_udf_func`）。源码注释里自称 "UDF's"，但机制完全不同。

**`CAN_ACCESS_TABLE` 的实现只有 5 行**——真正的判定在 `check_table_and_trigger_access()`，且用的是**调用者**的 `Security_context`：

```cpp
/* sql/item_func.cc */
longlong Item_func_can_access_table::val_int() {
  if (check_table_and_trigger_access(args, false, &null_value)) return 1;
  return 0;
}

static bool check_table_and_trigger_access(Item **args, bool check_trigger_acl,
                                           bool *null_value) {
  String schema_name;
  String *schema_name_ptr = args[0]->val_str(&schema_name);
  String table_name;
  String *table_name_ptr = args[1]->val_str(&table_name);
  ...
  // Skip INFORMATION_SCHEMA database
  if (is_infoschema_db(schema_name_ptr->ptr())) return true;
  ...
  // Check access
  Access_bitmask db_access = 0;
  if (check_access(thd, SELECT_ACL, schema_name_ptr->ptr(), &db_access, nullptr,
                   false, true))
    return false;

  Table_ref table_list;
  table_list.db = schema_name_ptr->ptr();
  table_list.table_name = table_name_ptr->ptr();
  table_list.grant.privilege = db_access;

  if (check_trigger_acl == false) {
    if (db_access & TABLE_OP_ACLS) return true;
    if (check_grant(thd, TABLE_OP_ACLS, &table_list, true, 1, true))
      return false;
  } else {  // Trigger check.
    if (check_table_access(thd, TRIGGER_ACL, &table_list, false, 1, true))
      return false;
  }
  return true;
}
```

★ 这解释了一个反直觉的现象：**视图的 DEFINER 是 `mysql.infoschema`，但过滤结果按调用者权限**。因为 `check_access()` / `check_grant()` 取的是 `current_thd` 的 `Security_context`——`CAN_ACCESS_*` 是"在视图体内以调用者身份做判断"的钩子。

配套账号（`scripts/mysql_system_users.sql`）与"底层 mysql.* 免权限检查"的代码，共同构成这个模型的两端：

```sql
-- Create an user that is definer for information_schema view
CREATE USER 'mysql.infoschema'@localhost IDENTIFIED WITH caching_sha2_password
 AS '$A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED'
 ACCOUNT LOCK;
GRANT SELECT ON *.* TO 'mysql.infoschema'@localhost;
GRANT SYSTEM_USER ON *.* TO 'mysql.infoschema'@localhost;
```

```cpp
/* sql/auth/sql_authorization.cc —— 正因为底层 mysql.* 被整体放行，
   行级过滤才必须落到 CAN_ACCESS_* 上 */
  /*
    Optimizer internal tables and the DD tables used under the
    INFORMATION_SCHEMA system views does not need any privilege checking.
  */
  if (all_tables->is_internal() ||
      (all_tables->referencing_view &&
       all_tables->referencing_view->is_system_view)) {
    all_tables->set_privileges(privilege);
    return false;
  }
```

#### ★ 反例：`INNODB_TABLES` / `INNODB_INDEXES` 是元数据，但不可能是视图

**它们是插件表**（`st_mysql_plugin`，`type = MYSQL_INFORMATION_SCHEMA_PLUGIN`，`fill_table = i_s_innodb_tables_fill_table`），**内容确实是纯元数据**（`TABLE_ID` / `NAME` / `FLAG` / `N_COLS` / `SPACE` / `ROW_FORMAT` / `INSTANT_COLS` / `TOTAL_ROW_VERSIONS`）。

**为什么不能走视图——硬证据在填充函数**：它扫的是 **InnoDB 自己的 SYS_TABLES 系统表（B-tree）**，不是 `mysql.tables`：

```cpp
static int i_s_innodb_tables_fill_table(THD *thd, Table_ref *tables, Item *) {
  btr_pcur_t pcur;
  const rec_t *rec;
  mem_heap_t *heap;
  mtr_t mtr;
  ...
  while (rec) {
    dict_table_t *table_rec;
    dd_process_dd_tables_rec_and_mtr_commit(heap, rec, &table_rec, dd_tables,
                                            &mdl_on_tab, &mtr);
```

填充值直接取自 `dict_table_t` 内存结构：

```cpp
  OK(fields[INNODB_TABLES_ID]->store(longlong(table->id), true));
  OK(field_store_string(fields[INNODB_TABLES_NAME], table->name.m_name));
  OK(fields[INNODB_TABLES_FLAG]->store(table->flags));
  OK(fields[INNODB_TABLES_NUM_COLUMN]->store(table->n_cols));
  OK(fields[INNODB_TABLES_SPACE]->store(table->space));
```

⇒ **`table->id`、`flags`、`n_cols`、`space`、`instant_cols`、`current_row_version` 这些物理/内部属性，DD 的 `mysql.tables` 里根本没有对应列**（DD 只存 `se_private_id` / `se_private_data` 这种不透明 blob）。要"用视图表达"就得让 SQL 去解析 `se_private_data`——不可行。

**这带来一条很好的归纳**：**InnoDB 的元数据被劈成两半**——
- 进了 DD 的那部分 → 做成 5 张 `innodb_*` **系统视图**（`innodb_datafiles` / `innodb_fields` / `innodb_foreign` / `innodb_foreign_cols` / `innodb_tablespaces_brief`）
- 只在 InnoDB SYS_* 里的那部分 → 做成 `INNODB_*` **插件表**

版本机制也印证：插件表用**自己的** `i_s_innodb_plugin_version_postfix`，与全局 `IS_VERSION` 无关。

### 十、SHOW 语句重写成视图查询

`sql/dd/info_schema/` 下的 `show.h` / `show.cc` 提供了一批构造函数，把 `SHOW` 在**语法分析阶段**就改写成对这些视图的 `SELECT`：

```cpp
Query_block *build_show_character_set_query(...);
Query_block *build_show_collation_query(...);
Query_block *build_show_databases_query(...);
Query_block *build_show_tables_query(...);
Query_block *build_show_columns_query(...);
Query_block *build_show_keys_query(...);
Query_block *build_show_triggers_query(...);
Query_block *build_show_procedures_query(...);
Query_block *build_show_events_query(...);
```

⚠️ 两个例外不走视图：

- **临时表**的 `SHOW COLUMNS` / `SHOW KEYS` —— 临时表不在 DD 里，走 `schema_tables[]` 的 `TMP_TABLE_COLUMNS` / `TMP_TABLE_KEYS`（插件机制）
- **`SHOW STATUS` / `SHOW VARIABLES`** —— 重写成 **PFS 表**查询

（详见 [`../../feat/i_s.md`](../../infra/observability/i_s.md) 第四章的三分天下。）

另外 `SHOW INDEX/KEYS` 用的是 hidden 的 `SHOW_STATISTICS` 视图（见 3.5），不是 `STATISTICS`。

---

## 已知限制与坑

| 坑 | 说明 |
|---|---|
| **同名 `INNODB_*` 两套机制** | `innodb_fields`（视图，从 DD 读）与 `INNODB_BUFFER_PAGE`（插件表，现场遍历）名字相近但实现完全不同。判据：查的是"能否用 SELECT 表达" |
| **改视图 SQL 而 `IS_VERSION` 不变 → 列类型滞后** | 因为 DD 里的列元数据是从 `ST_SCHEMA_TABLE` 反推的，不是从视图 DDL 反推 |
| **派生列是查询时算的** | `DATA_TYPE`、`CHARACTER_SET_NAME`、`PARTITION_METHOD` 等靠 SQL 表达式/CASE 算 |
| **统计值可能过期** | `TABLES.TABLE_ROWS` / `STATISTICS.CARDINALITY` 走 `mysql.*_stats` 缓存，受 `information_schema_stats_expiry` 影响；但 **`COLUMN_STATISTICS.HISTOGRAM` 不受影响**（持久化的 JSON） |
| **分区统计不走缓存** | `PARTITIONS` 每个分区实时问 SE |
| **`add_field` 的查重是 assert** | release 构建下重复插入会静默覆盖 |
| **`SHOW` 的重写有例外** | 临时表的 SHOW COLUMNS/KEYS 与 SHOW STATUS/VARIABLES 不走视图 |

### 性能相关

源码里**没有**形如"this is slow / expensive"的通用性能警告。但可以从代码结构客观列举开销来源：

- 每列一次 `INTERNAL_*` UDF 调用（`FILES` 单张 14 个，`TABLES` 9 个）
- 存储程序类视图 8 条 FROM
- `KEYWORDS` 的 FROM 是一个数万字符的 JSON 字面量 + `JSON_TABLE`，每次解析视图定义都要扫一遍
- 10~11 分支的 `CASE WHEN`
- 嵌套 `SUBSTRING_INDEX(SUBSTRING_INDEX(...))`

一处罕见的"为优化器选类型"的权衡注释（角色视图的递归 CTE）：

```cpp
      /*
        The use of CHAR(17000) enables use of MEDIUMTEXT datatype for
        'role_path' column.
        ...
        Using TEXT enable about ~2k SHA2 values to be stored in
        'role_path'. And using MEDIUMTEXT enables us to hold about ~500k
        SHA2 values.
      */
```

即：`CAST(... AS CHAR(17000))` 的长度是**故意写大的**，只为让优化器把列类型推成 MEDIUMTEXT（能存约 50 万个 SHA2 值做环路检测）而非 TEXT（约 2 千个）。

### 特权 UDF 的访问控制

`INTERNAL_*` / `CAN_ACCESS_*` 这些 UDF 普通用户不能调：

```cpp
    if (!thd->parsing_system_view && !thd->is_dd_system_thread() &&
        DBUG_EVALUATE_IF("skip_dd_table_access_check", false, true)) {
      my_error(ER_NO_ACCESS_TO_NATIVE_FCT, MYF(0), function_name.str);
```

`thd->parsing_system_view` 由语法层设置（正在解析系统视图时才放行）。

---

## Misc

### 工程技法

1. **`enum` 编号 + `std::map<int,...>` 有序迭代**——允许多级继承分片填充定义，字段顺序由枚举统一裁决（与构造顺序解耦）
2. **`CREATE OR REPLACE`** —— 让 43 条 DDL 可幂等重跑，省掉 DROP 与版本 diff
3. **UDF 承载 SQL 表达不了的逻辑**——`GET_DD_CREATE_OPTIONS` 解析 Properties 串、`INTERNAL_*` 取运行时统计、`CAN_ACCESS_*` 做逐行权限。它们是"视图的逃生舱"
4. **`fs_name_collation()` 延迟到运行时**——同一份二进制在不同 `lctn` 下生成不同 DDL
5. **CTE + `DISTINCT` 表达递归角色图**；为环路检测把路径压成 SHA2 串并用 `CHAR(17000)` 骗优化器选中 MEDIUMTEXT
6. **`JSON_TABLE` 把编译期数组变成行源**（`KEYWORDS`）——用 SQL 手段绕过"视图不能引用 C++ 数组"的限制
7. **分两批创建**（DD-based / non-DD-based）——因为角色视图依赖 ACL 表，必须等 `mysql_system_tables.sql`

### 扩展点：新增一张 I_S 视图

| # | 改动 |
|---|---|
| 1 | `sql/dd/impl/system_views/` 加 `.cc` + `.h`，继承 `System_view_select_definition_impl` |
| 2 | 构造函数 `set_view_name()` + 一串 `add_field(枚举, 列名, SQL 表达式)` + `add_from` / `add_where` |
| 3 | 在 `System_views::init()` 里 `register_view<X>()`（**漏这步视图不存在**），并选对 `Types` |
| 4 | **递增 `IS_DD_VERSION`**（否则重启时不会重建，用户看到的还是旧定义） |
| 5 | 如需被 `SHOW` 使用，在 `info_schema/show.cc` 加对应的 `build_show_*_query()` |

| 6 | 改 MTR：`i_s_schema_definition_debug.test` 的 `$expected_system_view_count`（现为 **44**）+ `i_s_schema_assert_and_fill_table_names.inc` 的视图名清单 + 重录 `i_s_schema_definition_debug.result` |
| 7 | **`sql/CMakeLists.txt`** 加 `.cc`（不加不参与编译） |
| — | ❌ **不需要**改 `scripts/mysql_system_tables.sql`（该文件只有 3 处对 I_S 的查询，无视图定义） |

★ 最容易漏的是**第 4 步**——43 张可见视图共用同一个 `IS_VERSION`，漏改会导致"改了代码但线上没生效"。

**为什么漏改不生效——重启时的完整重建逻辑**（`sql/dd/info_schema/metadata.cc`）：

```cpp
bool update_server_I_S_metadata(THD *thd) {
  bool error = false;
  dd::Dictionary_impl *d = dd::Dictionary_impl::instance();

  // Stop if I_S version is same and no DD upgrade was done.
  uint actual_version = d->get_actual_I_S_version(thd);

  if (d->get_target_I_S_version() == actual_version &&
      !dd::bootstrap::DD_bootstrap_ctx::instance().dd_upgrade_done())
    return false;                                  /* ★★★ 版本号相同就直接返回 */
  ...
  /*
    Stage 1:
    Remove all server I_S metadata from DD tables.
  */
  std::vector<const dd::View *> sch_views;
  if (thd->dd_client()->fetch_schema_components(sch_obj, &sch_views))
    return true;

  for (const dd::View *view : sch_views) {
    const dd::Properties *view_options = &view->options();
    if (!view_options->exists(PLUGIN_VERSION_STRING)) {
      error = dd::info_schema::remove_I_S_view_metadata(thd, view->name());
      if (error) break;
      // Update status of views referencing a system view.
      ...
    }
  }

  /*
    Stage 2:
    1) Store server I_S tables
    2) Recreate system view.
    3) Update the target IS version in DD.
  */
  error = error || dd::info_schema::store_server_I_S_metadata(thd) ||
          dd::info_schema::create_system_views(thd);

  return dd::end_transaction(thd, error);
}
```

> ★★★ **判定完全靠单个 `IS_VERSION` 数字，没有任何"逐视图比对定义文本"的逻辑**。视图体改了但忘记 bump `IS_DD_VERSION` ⇒ 重启后**不会重建**——这就是第 4 步最致命的原因。

版本号存在哪（`IS_DD_VERSION` 的唯一使用点）：

```cpp
uint Dictionary_impl::get_target_I_S_version() {
  return dd::info_schema::IS_DD_VERSION;
}

uint Dictionary_impl::get_actual_I_S_version(THD *thd) {
  bool exists = false;
  uint version = 0;
  bool error [[maybe_unused]] = tables::DD_properties::instance().get(
      thd, "IS_VERSION", &version, &exists);
  ...
  return version;
}
```

即存成 `mysql.dd_properties` 里的一行 `IS_VERSION`。

### 易混淆点

| 混淆 | 澄清 |
|---|---|
| I_S 表都是"虚拟的临时表"？ | ❌ 只有插件表是；**DD 系统视图是真视图**（`CREATE VIEW` 建的） |
| `innodb_fields` 与 `INNODB_BUFFER_PAGE` 同机制？ | ❌ 前者视图（DD），后者插件表（现场遍历） |
| "元数据 → 视图"是判据？ | ❌ 真正判据是**能否用一条 SELECT 从 `mysql.*` 算出来**。`INNODB_TABLES` 是元数据却只能是插件表 |
| `I_S.FILES` 是 hidden 视图？ | ❌ 它不 hidden，走的是**独立的 PROCESS 权限分支**；43 张里唯一 hidden 的是 `SHOW_STATISTICS` |
| I_S 里的统计值都受 `stats_expiry` 影响？ | ❌ `COLUMN_STATISTICS.HISTOGRAM` 是持久化的 JSON，不受影响 |
| `I_S.KEYWORDS` 是硬编码表？ | ❌ 构建期从 `symbols[]` + yacc 生成，运行期用 `JSON_TABLE` 展开 |

### ★★★ 真名勘误与数字核实

| 常见说法 | 8.0.39 实际 |
|---|---|
| 8 张 `*_extensions` | **5 张**：`columns_extensions` / `schemata_extensions` / `table_constraints_extensions` / `tables_extensions` / `tablespaces_extensions`（其中含 `engine_attribute` 的是 4 张，`schemata_extensions` 没有） |
| "43 行 `register_view<X>()`" | **44 行**。差额是 `SHOW_STATISTICS`（`hidden()=true`）。故"用户可见 43"成立，"注册 43 行"不成立；MTR 里写的也是 44 |
| `System_view_definition_impl::add_field()` | 基类**没有** `add_field()`。它在派生类 **`System_view_select_definition_impl`** 上 |
| `Object_table_definition_impl` 用数组存字段 | 是 `std::map<int, String_type>`（按 enum 有序），不是数组也不是 `push_back` |
| `dd::View::enum_view_type` | **不存在**。是 `dd::enum_table_type { INVALID_TABLE, BASE_TABLE, USER_VIEW, SYSTEM_VIEW }`（`sql/dd/types/abstract_table.h`） |
| `TABLE_SHARE::is_system_view` | **不存在**。是 **`Table_ref::is_system_view`**（`sql/table.h`） |
| `CAN_ACCESS_*` 是 UDF | 严格说是 **native/internal function**，工厂 `Internal_function_factory`，不是 `Create_udf_func`。源码注释自称 "UDF's" |
| `options` 白名单叫 `is_valid_option` / `VALID_*` | 真名 `default_valid_option_keys`（各 `*_impl.cc` 顶部）+ `Properties_impl::m_keys`；判据是 `m_keys.empty() ⇒ 任意 key 合法` |
| `IS_DD_VERSION` 在 `system_registry.cc` | 只在 **`sql/dd/info_schema/metadata.h`**（值 **80030**，不是 80039） |
| `INNODB_TABLESPACE_BRIEF`（单数） | `innodb_tablespaces_brief.h` 的**注释**写单数，`view_name()` 是 **复数 `INNODB_TABLESPACES_BRIEF`**——以 `view_name()` 为准 |
| 新增视图要改 `mysql_system_tables.sql` | **不需要**（该文件只有 3 处对 I_S 的查询）；要改的是 `sql/CMakeLists.txt` 与 MTR 用例 |

### ★ 权限模型的一句话总结

视图以 `mysql.infoschema@localhost` 为 DEFINER（账号被 `ACCOUNT LOCK` 且只有 SELECT + SYSTEM_USER），**底层 `mysql.*` DD 表在 I_S 视图下免权限检查**，于是**行级过滤的全部责任落进视图体里的 `CAN_ACCESS_*`**——而这些函数只能在 `thd->parsing_system_view` 为真时被调用，用户手写会得到 `ER_NO_ACCESS_TO_NATIVE_FCT`。三处代码合起来才构成完整的权限模型，缺一块就会误判。

---

## 参考

- MySQL 8.0 Reference Manual → *INFORMATION_SCHEMA Tables* / *Data Dictionary*
- MySQL 8.0 Source Code → `sql/dd/impl/system_views/`、`sql/dd/info_schema/`、`sql/dd/impl/system_registry.cc`、`sql/gen_keyword_list.cc`
- [../../feat/i_s.md](../../infra/observability/i_s.md)（另一半：插件机制与 InnoDB 26 张表）
- [dd.md](dd.md)（DD 缓存 / 升级 / SDI / `se_private_data` / `dd::Properties`）
- [statistics.md](statistics.md)（统计的四套体系）
