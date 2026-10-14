# 存储程序与视图的 DD 侧持久化

> **边界**：本篇讲 **DD 侧的往返**（存储程序 / 触发器 / 事件 / 视图 / 资源组怎么进 `mysql.*` 表、怎么读回来）。
> - **SQL 层语义**（`sp_head`、指令集、缓存、游标、binlog、递归）见 [`../stored_program.md`](../stored_program.md)——**姊妹篇**，本篇不复述。
> - **读出侧的投影**（`I_S.ROUTINES` / `EVENTS` / `TRIGGERS` 视图的 `CAN_ACCESS_*` 两层权限、`definition_utf8`、`CONVERT_TZ` 时区）见 [`system_views.md`](system_views.md)。**那边是"读"，本篇是"写"，互指**。
> - DD 表体系 / 对象模型 / 缓存 / SDI / 升级见 [`dd.md`](dd.md)；建表往返（`dd_table.cc`）见 `dd.md` §2.6。
> - 触发器/事件的**执行**语义（触发顺序、调度器）不在本篇。

**一句话机制**：SQL 层的语义对象（`sp_head` / `Table_ref` / `Event_parse_data` / `::Trigger`）经各自 `dd::create_*` 翻译成 DD 对象树，落到 `mysql.routines` / `triggers` / `events` / `tables` 四张表——**MDL 与事务提交 100% 在 SQL 层**，DD 侧只负责翻译与 store。

---

## 目录

- [概述](#概述)
  - [是什么](#是什么)
  - [落表总览](#落表总览)
- [设计思想与理论基础](#设计思想与理论基础)
- [核心实现](#核心实现)
  - [一、`dd_routine.cc`：存储程序 → `mysql.routines`](#一dd_routinecc存储程序--mysqlroutines)
  - [二、`dd_trigger.cc`：触发器 → `mysql.triggers`](#二dd_triggercc触发器--mysqltriggers)
  - [三、`dd_event.cc`：事件 → `mysql.events`](#三dd_eventcc事件--mysqlevents)
  - [四、`dd_view.cc`：视图 → `mysql.tables`](#四dd_viewcc视图--mysqltables)
  - [五、`dd_resource_group.cc`：资源组](#五dd_resource_groupcc资源组)
  - [六、横向：五类的共同模式](#六横向五类的共同模式)
- [已知限制与坑](#已知限制与坑)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

`CREATE PROCEDURE / FUNCTION / TRIGGER / EVENT / VIEW / RESOURCE GROUP` 落到 DD 的**唯一**写入路径，共 5 个文件约 72 KB。此前的知识库覆盖几乎为零——`dd_trigger` / `dd_event` / `dd_resource_group` 相关符号**全库 0 命中**。

### 落表总览

| 类别 | DD 写入函数 | 落表 | 从属子表 |
|---|---|---|---|
| 存储过程/函数 | `dd::create_routine` / `alter_routine` | `mysql.routines` | `parameters`、`parameter_type_elements` |
| 触发器 | `dd::create_trigger` | `mysql.triggers` | —（作为 `dd::Table` 的**子对象**） |
| 事件 | `dd::create_event` / `update_event` | `mysql.events` | — |
| **视图** | `dd::create_view` / `update_view` | **`mysql.tables`**（`type='VIEW'`） | `columns`、`view_table_usage`、`view_routine_usage` |
| 资源组 | `dd::create_resource_group` | `mysql.resource_groups` | — |

### ★ 视图没有自己的表

这是本篇最需要先说清的结构事实：**`sql/dd/impl/tables/` 下不存在 `views.cc`**。视图与表共用 `dd::tables::Tables`，靠 `type` 列区分，视图专属列以 `view_*` 前缀挂在**同一行**：

`view_definition`、`view_definition_utf8`、`view_check_option`、`view_is_updatable`、`view_algorithm`、`view_security_type`、`view_definer`、`view_client_collation_id`、`view_connection_collation_id`、`view_column_names`，外加 `KEY(type, view_definer)` 索引。

⚠️ `dd_view.cc` 里那句注释 `// Store info in DD views table.` 是**过时注释**——实际落的是 `mysql.tables`。

---

## 设计思想与理论基础

**① 三类写入器，分工明确。**

| 模式 | 代表 | 行为 |
|---|---|---|
| **被动填充器** | `dd::create_table` | 造对象返回，**不 store**，由调用方决定何时写 |
| **主动写入器** | `create_routine` / `create_view` / `create_event` | **内部直接 `dd_client()->store()`** |
| **子对象挂载器** | `create_trigger` | 改父 `dd::Table`，然后 `update(父)` |

⇒ **MDL 与事务提交 100% 在 SQL 层**，唯一例外是资源组（DD 层自己 `trans_commit`）。

**② 单向依赖：DD 层不认识 SQL 层的运行时对象。**

`dd_routine.cc` 只读 `sp_head` 的字段，**绝不反向构造** `sp_head`——反方向走另一条链（`db_load_routine()`）。`sp_head.cc` 头部注释把两者并列为同一"数据库访问封装层"。

**③ 双份文本是普遍模式（4/4），且来源统一。**

原始列（BLOB）+ utf8 列（TEXT）成对出现，utf8 版**不是写入时转码的**，而是**词法分析阶段增量生成**的（`Lex_input_stream::m_body_utf8`）。routine / trigger / event / view 四类的"体"**都来自 `sp_head`**，是同一个解析产物。

**④ 枚举转换是跨层解耦的屏障。**

每类都有 old（server enum）↔ new（DD enum）转换，DD 落盘的是 `ENUM('INSERT','UPDATE','DELETE')` 这类**字面量**，不受 SQL 层重构影响。而且**默认值归一化点就在转换函数里**，保证 DD 里存的永远是已解析的确定值。

---

## 核心实现

### 一、`dd_routine.cc`：存储程序 → `mysql.routines`

#### 1.1 `fill_dd_routine_info()` 的字段映射

| DD 字段 | 来源 |
|---|---|
| `name` / `definition` / `definition_utf8` | `sp->m_name` / `m_body` / `m_body_utf8` |
| `parameter_str` | `sp->m_params`（注释：`// Set parameter str for show routine operations.`） |
| `is_deterministic` / `sql_data_access` / `security_type` / `comment` | `sp->m_chistics` |
| **`definer`** | **入参 `LEX_USER*`**，不是 `sp_head`（`sp_head` 有 `m_definer_user` 但不用） |
| `sql_mode` | `thd->variables.sql_mode` |
| `client_collation_id` / `connection_collation_id` | `thd->charset()` / `thd->variables.collation_connection` |
| `schema_collation_id` | `get_default_db_collation(schema)`，失败回落 `thd->collation()` |

**两个默认值归一化**（CREATE 时就已解析成确定值）：

```c
enum_sp_data_access sp_daccess =
    (sp->m_chistics->daccess == SP_DEFAULT_ACCESS) ? SP_DEFAULT_ACCESS_MAPPING
                                                   : sp->m_chistics->daccess;
```
```c
enum_sp_suid_behaviour sp_suid = (sp->m_chistics->suid == SP_IS_DEFAULT_SUID)
                                     ? SP_DEFAULT_SUID_MAPPING
                                     : sp->m_chistics->suid;
```

⇒ 所以 `mysql.routines.security_type` 的 `ENUM('DEFAULT','INVOKER','DEFINER')` 里，**`'DEFAULT'` 永远不会由 CREATE 路径写入**。

⚠️ **一个跨类型复用**：`security_type` 用的是 **`View::enum_security_type`**（`View::ST_DEFINER` / `ST_INVOKER`），不是 `Routine` 自己的枚举。

**★★ `fill_dd_routine_info()` 完整源码**（`sql/dd/dd_routine.cc:303`）：

```cpp
static bool fill_dd_routine_info(THD *thd, const dd::Schema &schema,
                                 sp_head *sp, Routine *routine,
                                 const LEX_USER *definer) {
  // Set name.
  routine->set_name(sp->m_name.str);

  // Set definition.
  routine->set_definition(sp->m_body.str);        /* ★ 原字符集正文 → definition LONGBLOB */

  // Set definition_utf8.
  routine->set_definition_utf8(sp->m_body_utf8.str);  /* ★ UTF8 正文 → definition_utf8 LONGTEXT */

  // Set parameter str for show routine operations.
  routine->set_parameter_str(sp->m_params.str);   /* ★ 仅供 SHOW CREATE / I_S，不参与解析 */

  // Set is_deterministic.
  routine->set_deterministic(sp->m_chistics->detistic);   /* ★ 源码拼写就是 detistic */

  // Set SQL data access.
  Routine::enum_sql_data_access daccess;
  enum_sp_data_access sp_daccess =
      (sp->m_chistics->daccess == SP_DEFAULT_ACCESS) ? SP_DEFAULT_ACCESS_MAPPING
                                                     : sp->m_chistics->daccess;
  switch (sp_daccess) {
    case SP_NO_SQL:            daccess = Routine::SDA_NO_SQL; break;
    case SP_CONTAINS_SQL:      daccess = Routine::SDA_CONTAINS_SQL; break;
    case SP_READS_SQL_DATA:    daccess = Routine::SDA_READS_SQL_DATA; break;
    case SP_MODIFIES_SQL_DATA: daccess = Routine::SDA_MODIFIES_SQL_DATA; break;
    default:
      assert(false); /* purecov: deadcode */
      return true;   /* purecov: deadcode */
  }
  routine->set_sql_data_access(daccess);

  // Set security type.
  View::enum_security_type sec_type;      /* ★★ 用的是 dd::View 的枚举 */
  enum_sp_suid_behaviour sp_suid = (sp->m_chistics->suid == SP_IS_DEFAULT_SUID)
                                       ? SP_DEFAULT_SUID_MAPPING
                                       : sp->m_chistics->suid;
  switch (sp_suid) {
    case SP_IS_SUID:     sec_type = View::ST_DEFINER; break;
    case SP_IS_NOT_SUID: sec_type = View::ST_INVOKER; break;
    default:
      assert(false); /* purecov: deadcode */
      return true;   /* purecov: deadcode */
  }
  routine->set_security_type(sec_type);   /* ★ ST_DEFAULT 永远不会从这里写入 */

  // Set definer.
  routine->set_definer(definer->user.str, definer->host.str);

  // Set sql_mode.
  routine->set_sql_mode(thd->variables.sql_mode);

  // Set client collation id.
  routine->set_client_collation_id(thd->charset()->number);
  // Set connection collation id.
  routine->set_connection_collation_id(
      thd->variables.collation_connection->number);

  // Set schema collation id.
  const CHARSET_INFO *db_cs = nullptr;
  if (get_default_db_collation(schema, &db_cs)) {
    assert(thd->is_error());
    return true;
  }
  if (db_cs == nullptr) db_cs = thd->collation();
  routine->set_schema_collation_id(db_cs->number);

  // Set comment.
  routine->set_comment(sp->m_chistics->comment.str ? sp->m_chistics->comment.str
                                                   : "");  /* ★ comment 列 NOT NULL */

  // Fill routine parameters
  return fill_routine_parameters_info(thd, sp, routine);
}
```

```c
/* sql/sp.h:65 —— 两个归一化常量都是宏，不是枚举 */
#define SP_DEFAULT_ACCESS_MAPPING SP_CONTAINS_SQL
#define SP_DEFAULT_SUID_MAPPING SP_IS_SUID
```

```c
/* sql/sql_lex.h:199 —— SQL 层枚举，0 就是"未指定" */
enum enum_sp_suid_behaviour { SP_IS_DEFAULT_SUID = 0, SP_IS_NOT_SUID, SP_IS_SUID };
enum enum_sp_data_access { SP_DEFAULT_ACCESS = 0, SP_CONTAINS_SQL, SP_NO_SQL,
                           SP_READS_SQL_DATA, SP_MODIFIES_SQL_DATA };
```

```cpp
/* sql/dd/types/routine.h:87 —— DD 侧枚举，全部从 1 起（0 留给 NULL） */
  enum enum_routine_type { RT_FUNCTION = 1, RT_PROCEDURE };
  enum enum_sql_data_access {
    SDA_CONTAINS_SQL = 1, SDA_NO_SQL, SDA_READS_SQL_DATA, SDA_MODIFIES_SQL_DATA
  };
/* sql/dd/types/view.h:58 —— security_type 真身在这里 */
  enum enum_security_type { ST_DEFAULT = 1, ST_INVOKER, ST_DEFINER };
```

```c
struct st_sp_chistics {          /* sql/sql_lex.h:2624 */
  LEX_CSTRING comment;
  enum enum_sp_suid_behaviour suid;
  bool detistic;                 /* ★ 原文拼写，全库一致 */
  enum enum_sp_data_access daccess;
};
```

> ⚠️ **`dd_routine.cc` 里没有 `get_sql_data_access()` / `get_security_type()` / `get_parameter_mode()` 这三个辅助函数**（全文件 0 命中）——枚举转换是 `fill_dd_routine_info()` / `alter_routine()` 里的**内联 switch**。该文件只有 4 个 static 函数：`fill_dd_function_return_type`(79) / `fill_parameter_info_from_field`(148) / `fill_routine_parameters_info`(232) / `fill_dd_routine_info`(303)。
> ★ 反方向（DD → `sp_head`）的映射在 `sql/dd_sp.cc` 的 `prepare_sp_chistics_from_dd_routine()`，那里 `security_type()==ST_INVOKER ? SP_IS_NOT_SUID : SP_IS_SUID`，`default:` 回落 `SP_DEFAULT_ACCESS_MAPPING`——与这里**互为逆运算**。

#### ★ 1.2 `type` 不是 fill 出来的

`fill_dd_routine_info()` **完全没有 `set_type()`**。类型由**构造哪种子类**决定：

- `function_impl.cc` 构造函数末尾：`set_type(RT_FUNCTION);`
- `routine_impl.cc` 默认构造：`m_routine_type(RT_PROCEDURE)`

对应 `UNIQUE KEY(schema_id, type, name)`——**唯一键带 type，所以同名的 FUNCTION 与 PROCEDURE 可以共存**。

#### 1.3 `create_routine()`：store 前后有个特殊保护

```c
    enum_check_fields saved_check_for_truncated_fields = thd->check_for_truncated_fields;
    thd->check_for_truncated_fields = CHECK_FIELD_WARN;     // ★
    error = thd->dd_client()->store(func.get());
    thd->check_for_truncated_fields = saved_check_for_truncated_fields;
```

即：**DD 表列宽不够时只警告不报错**（`comment` / `definition` 超长不至于让 CREATE 失败）。⚠️ `dd_view.cc` 与 `dd_event.cc` **没有**这个保护。

**★ `create_routine()` 与 `alter_routine()` 完整源码**：

```cpp
bool create_routine(THD *thd, const Schema &schema, sp_head *sp,
                    const LEX_USER *definer) {
  bool error = false;
  if (sp->m_type == enum_sp_type::FUNCTION) {
    std::unique_ptr<Function> func(schema.create_function(thd));

    // Fill stored function return type.
    fill_dd_function_return_type(thd, sp, func.get());   /* ★ 仅 FUNCTION：写 result_* 九列 */

    if (fill_dd_routine_info(thd, schema, sp, func.get(), definer)) return true;

    // Store routine metadata in DD table.
    enum_check_fields saved_check_for_truncated_fields =
        thd->check_for_truncated_fields;
    thd->check_for_truncated_fields = CHECK_FIELD_WARN;   /* ★ 降级为 WARN */
    error = thd->dd_client()->store(func.get());
    thd->check_for_truncated_fields = saved_check_for_truncated_fields;
  } else {
    std::unique_ptr<Procedure> proc(schema.create_procedure(thd));
    if (fill_dd_routine_info(thd, schema, sp, proc.get(), definer)) return true;

    // Store routine metadata in DD table.
    enum_check_fields saved_check_for_truncated_fields =
        thd->check_for_truncated_fields;
    thd->check_for_truncated_fields = CHECK_FIELD_WARN;
    error = thd->dd_client()->store(proc.get());
    thd->check_for_truncated_fields = saved_check_for_truncated_fields;
    /*  ★★ PROCEDURE 分支同样有一份完整的三行保护 —— 不是只有 FUNCTION 有 */
  }
  return error;
}
```

```cpp
bool alter_routine(THD *thd, Routine *routine, st_sp_chistics *chistics) {
  // Set last altered time.
  routine->set_last_altered(
      dd::my_time_t_to_ull_datetime(thd->query_start_in_secs()));  /* ★ 唯一无条件写入 */

  // Set security type.
  if (chistics->suid != SP_IS_DEFAULT_SUID) {     /* ★ 未指定则不改 */
    View::enum_security_type sec_type;
    switch (chistics->suid) {
      case SP_IS_SUID:     sec_type = View::ST_DEFINER; break;
      case SP_IS_NOT_SUID: sec_type = View::ST_INVOKER; break;
      default: assert(false); return true;
    }
    routine->set_security_type(sec_type);
  }

  // Set sql data access.
  if (chistics->daccess != SP_DEFAULT_ACCESS) {   /* ★ 未指定则不改 */
    ... routine->set_sql_data_access(daccess);
  }

  // Set comment.
  if (chistics->comment.str) routine->set_comment(chistics->comment.str);
  /*  ★ NULL 表示"不改"；空串表示"清空" */

  // Update routine.
  return thd->dd_client()->update(routine);
}
```

> ★★ **`alter_routine()` 不写 `result_*` / `definition*` / 参数**，它只 `update(routine)`。`ALTER PROCEDURE` 改 body 走的是**删 + 重新 `create_routine()`**，不是 `alter_routine()`。这与"CREATE 侧默认值归一化"形成对照：**CREATE 是"未指定 → 归一化为确定值"，ALTER 是"未指定 → 保持原值"**。

`alter_routine()` 只支持改"特征"（`suid` / `daccess` / `comment`），**不支持改体**；未指定的特征保持原值——与 CREATE 侧的 DEFAULT 归一化形成对照。

#### 1.4 `fill_routine_parameters_info()` → `parameters`

★ **存储函数的返回类型也占一行**（第 0 号参数）：

```c
  /*
    The return type of the stored function is listed as first parameter from
    the Information_schema.parameters. Storing return type as first parameter
    for the stored functions.
  */
  if (sp->m_type == enum_sp_type::FUNCTION) {
    dd::Parameter *param = routine->add_parameter();
    fill_parameter_info_from_field(thd, &sp->m_return_field_def, param);
  }
```

它**不 `set_name()`、不 `set_mode()`**——正对应 `mysql.parameters` 里 `mode` / `name` **可为 NULL**（而非 NOT NULL）。

形参遍历 `sp_pcontext` 的变量，`mode` 映射 `MODE_IN/OUT/INOUT → PM_IN/PM_OUT/PM_INOUT`。`ordinal_position` 由集合顺序自动编号，`UNIQUE KEY(routine_id, ordinal_position)`。

⚠️ 一个反例设计：ENUM/SET 元素存进 `parameter_type_elements` 时用的是 **`VARBINARY(1020)`**，不是 VARCHAR——**元素值按原始字节存，不转码**（与"双份 utf8"模式相反）。注释还留了个自问：`// We may have multiple similar element names. Do we plan to deprecate it?`

**★ `fill_routine_parameters_info()` 完整源码**（`sql/dd/dd_routine.cc:232`）：

```cpp
static bool fill_routine_parameters_info(THD *thd, sp_head *sp,
                                         Routine *routine) {
  /*
    The return type of the stored function is listed as first parameter from
    the Information_schema.parameters. Storing return type as first parameter
    for the stored functions.
  */
  if (sp->m_type == enum_sp_type::FUNCTION) {
    // Add parameter.
    dd::Parameter *param = routine->add_parameter();
    /*  ★★ 第 0 号参数：不 set_name()、不 set_mode() → 两列均为 NULL */

    // Fill return type information.
    fill_parameter_info_from_field(thd, &sp->m_return_field_def, param);
  }

  // Fill parameter information of the stored routine.
  sp_pcontext *sp_root_parsing_ctx = sp->get_root_parsing_context();
  assert(sp_root_parsing_ctx != nullptr);
  for (uint i = 0; i < sp_root_parsing_ctx->context_var_count(); i++) {
    sp_variable *sp_var = sp_root_parsing_ctx->find_variable(i);
    Create_field *field_def = &sp_var->field_def;

    dd::Parameter *param = routine->add_parameter();
    param->set_name(sp_var->name.str);

    // Set parameter mode.
    Parameter::enum_parameter_mode mode;
    switch (sp_var->mode) {          /* ★ 内联 switch，不是辅助函数 */
      case sp_variable::MODE_IN:    mode = Parameter::PM_IN; break;
      case sp_variable::MODE_OUT:   mode = Parameter::PM_OUT; break;
      case sp_variable::MODE_INOUT: mode = Parameter::PM_INOUT; break;
      default: assert(false); return true;
    }
    param->set_mode(mode);

    fill_parameter_info_from_field(thd, field_def, param);
  }
  return false;
}
```

```cpp
/* sql/dd/types/parameter.h:56 */  enum enum_parameter_mode { PM_IN = 1, PM_OUT, PM_INOUT };
/* sql/sp_pcontext.h:49      */  enum enum_mode { MODE_IN, MODE_OUT, MODE_INOUT };  /* ★ 从 0 起，差 1 */
```

**★ `mysql.parameters` 表定义**（`sql/dd/impl/tables/parameters.cc:48`，关键列）：

```c
  "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT"
  "routine_id BIGINT UNSIGNED NOT NULL"
  "ordinal_position INT UNSIGNED NOT NULL"        /* ★ 返回类型行 = 0，普通参数从 1 起 */
  "mode ENUM('IN','OUT','INOUT')"                 /* ★★ 可空：返回类型行无 mode */
  "name VARCHAR(64) COLLATE utf8mb3_general_ci"   /* ★★ 可空：返回类型行无 name */
  "data_type ENUM(... 33 种 MYSQL_TYPE_*) NOT NULL"
  "data_type_utf8 MEDIUMTEXT NOT NULL"
  "is_zerofill BOOL" "is_unsigned BOOL" "char_length INT UNSIGNED"
  "numeric_precision INT UNSIGNED" "numeric_scale INT UNSIGNED"
  "datetime_precision INT UNSIGNED" "collation_id BIGINT UNSIGNED"
  "options MEDIUMTEXT"                            /* ★ geometry 子类型 "geom_type" 写在这里 */
  ...
  "PRIMARY KEY(id)"
  "UNIQUE KEY (routine_id, ordinal_position)"
  "KEY (collation_id)"
  "FOREIGN KEY (routine_id) REFERENCES routines(id)"
  "FOREIGN KEY (collation_id) REFERENCES collations(id)"
```

**★ `mysql.parameter_type_elements` 表定义**（★ VARBINARY 的完整注释）：

```c
  "parameter_id BIGINT UNSIGNED NOT NULL"
  "element_index INT UNSIGNED NOT NULL"
  // Fail if the max length of enum/set elements is increased.
  // If it's changed, the corresponding column length must be
  // increased, but this must be treated as a DD table upgrade
  // requiring special care.
  assert(MAX_INTERVAL_VALUE_LENGTH <= 255);          /* ★ 编译期护栏 */

  // Leave room for four bytes per character, which is used
  // by e.g. utf8mb4, i.e. 255 * 4 = 1020 bytes.
  "name VARBINARY(1020) NOT NULL"                    /* ★★ 255 字符 × utf8mb4 4 字节 */
  "PRIMARY KEY(parameter_id, element_index)"
  // We may have multiple similar element names. Do we plan to deprecate it?
  // m_target_def.add_index("UNIQUE KEY(column_id, name)");   /* ★ 被注释掉 */
  "FOREIGN KEY (parameter_id) REFERENCES parameters(id)"
```

> ⚠️ 注意：`m_body_utf8` 是 **utf8mb3**，而 `parameter_type_elements.name` 的 `VARBINARY(1020)` 才是为 **utf8mb4**（255×4）预留的。两者别混。

#### 1.5 `result_*` 九列只为 I_S 服务

`fill_dd_function_return_type()` 填 `result_data_type` / `result_data_type_utf8` / `result_zerofill` / `result_unsigned` / `result_char_length` / `result_numeric_*` / `result_datetime_precision` / `result_collation_id`。

★ 一条注释直接承认这是**冗余**：

```c
/*
  set result char length.
  Note that setting this only affects information schema views, and not any
  actual definitions. When initializing functions/routines, length information
  is read from dd::Parameter and not this field.
*/
```

即：**真正初始化函数时长度从 `dd::Parameter`（第 0 号参数）读，`mysql.routines.result_char_length` 只是给 I_S 看的**。

**★ `fill_dd_function_return_type()` 完整源码**（⚠️ 真名是 `function` 不是 `routine`）：

```cpp
/**
  Helper method for create_routine() to fill return type information of stored
  routine from the sp_head.
  from the sp_head.      /* ★ 原文如此，重复了两遍 —— 8.0.39 真实笔误 */
*/
static void fill_dd_function_return_type(THD *thd, sp_head *sp, Function *sf) {
  Create_field *return_field = &sp->m_return_field_def;
  assert(return_field != nullptr);

  sf->set_result_data_type(get_new_field_type(return_field->sql_type));

  // We need a fake table and share to generate a utf8 string
  // representation of result data type.
  TABLE table;  TABLE_SHARE share;
  table.s = &share;  table.in_use = thd;  table.s->db_low_byte_first = true;
  /*  ★★ 空壳 TABLE / TABLE_SHARE —— 只为给 Field 提供宿主以生成类型字符串 */

  sf->set_result_data_type_utf8(get_sql_type_by_create_field(&table, *return_field));
  sf->set_result_zerofill(return_field->is_zerofill);
  sf->set_result_unsigned(return_field->is_unsigned);

  /*
    set result char length.
    Note that setting this only affects information schema views, and not any
    actual definitions. When initializing functions/routines, length information
    is read from dd::Parameter and not this field.
  */
  sf->set_result_char_length(return_field->max_display_width_in_bytes());

  uint numeric_precision = 0;
  if (get_field_numeric_precision(return_field, &numeric_precision) == false)
    sf->set_result_numeric_precision(numeric_precision);

  uint scale = 0;
  if (get_field_numeric_scale(return_field, &scale) == false ||
      numeric_precision)          /* ★ 注意是 || numeric_precision，与下面 param 版写法不同 */
    sf->set_result_numeric_scale(scale);
  else
    assert(sf->is_result_numeric_scale_null());

  uint dt_precision = 0;
  if (get_field_datetime_precision(return_field, &dt_precision) == false)
    sf->set_result_datetime_precision(dt_precision);

  sf->set_result_collation_id(return_field->charset->number);
}
```

> ⚠️ **`fill_dd_routine_return_type()` 不存在**（全仓 0 命中）——真名 `fill_dd_function_return_type()`，只服务 `dd::Function`。
> ⚠️ 同样的"空壳 TABLE"技法也见于 `fill_parameter_info_from_field()` 与 `fill_dd_view_columns()`——**这是 DD 层复用 SQL 层类型推导的通用手法**。

**★ `mysql.routines` 表定义**（`sql/dd/impl/tables/routines.cc:52`，关键列 + 索引 + 外键）：

```c
  "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT"
  "schema_id BIGINT UNSIGNED NOT NULL"
  "name VARCHAR(64) NOT NULL COLLATE utf8mb3_general_ci"
  "type ENUM('FUNCTION', 'PROCEDURE') NOT NULL"      /* ★ RT_FUNCTION=1, RT_PROCEDURE=2 */
  "result_data_type ENUM(... 33 种 MYSQL_TYPE_*) DEFAULT NULL"   /* ★ PROCEDURE 行此列 NULL */
  "result_data_type_utf8 MEDIUMTEXT NOT NULL"
  "result_is_zerofill BOOL DEFAULT NULL"  "result_is_unsigned BOOL DEFAULT NULL"
  "result_char_length INT UNSIGNED DEFAULT NULL"
  "result_numeric_precision INT UNSIGNED DEFAULT NULL"
  "result_numeric_scale INT UNSIGNED DEFAULT NULL"
  "result_datetime_precision INT UNSIGNED DEFAULT NULL"
  "result_collation_id BIGINT UNSIGNED DEFAULT NULL"
  "definition LONGBLOB"                              /* ★ 原字符集正文 */
  "definition_utf8 LONGTEXT"                         /* ★ UTF8 正文 */
  "parameter_str BLOB"
  "is_deterministic BOOL NOT NULL"
  "sql_data_access ENUM('CONTAINS SQL','NO SQL','READS SQL DATA','MODIFIES SQL DATA') NOT NULL"
  "security_type ENUM('DEFAULT','INVOKER','DEFINER') NOT NULL"   /* ★ DEFAULT 在首位 */
  "definer VARCHAR(288) NOT NULL"
  "sql_mode ..."  "client_collation_id BIGINT UNSIGNED NOT NULL"
  "connection_collation_id BIGINT UNSIGNED NOT NULL"
  "schema_collation_id BIGINT UNSIGNED NOT NULL"
  "created TIMESTAMP NOT NULL"  "last_altered TIMESTAMP NOT NULL"
  "comment TEXT NOT NULL"       "options MEDIUMTEXT"
  "external_language VARCHAR(64) NOT NULL DEFAULT 'SQL'"   /* ★ 8.0 新增，预留给非 SQL 语言 */

  "PRIMARY KEY(id)"
  "UNIQUE KEY(schema_id, type, name)"    /* ★★ 唯一键带 type → 同名 FUNCTION 与 PROCEDURE 共存 */
  "KEY(result_collation_id)" "KEY(client_collation_id)"
  "KEY(connection_collation_id)" "KEY(schema_collation_id)" "KEY(definer)"
  "FOREIGN KEY (schema_id) REFERENCES schemata(id)"
  "FOREIGN KEY (result_collation_id) REFERENCES collations(id)"   /* 等 5 条 collation FK */
```

```cpp
/* sql/dd/impl/tables/routines.cc —— 按 type 决定造哪个子类 */
Routine *Routines::create_entity_object(const Raw_record &r) const {
  Routine::enum_routine_type routine_type =
      (Routine::enum_routine_type)r.read_int(FIELD_TYPE);
  if (routine_type == Routine::RT_FUNCTION)
    return dd::create_object<Function>();
  else
    return dd::create_object<Procedure>();
}
```

**★ 双份文本的真正来源：`Lex_input_stream::m_body_utf8`**（★ 解析期增量生成，不是存储期转码）：

```cpp
  /** UTF8-body buffer created during parsing. */    /* sql/sql_lex.h:3706 */
  char *m_body_utf8;
  char *m_body_utf8_ptr;
  const char *m_cpp_utf8_processed_ptr;
```

```cpp
void Lex_input_stream::body_utf8_start(THD *thd, const char *begin_ptr) {
  size_t body_utf8_length =
      (m_buf_length / thd->variables.character_set_client->mbminlen) *
      my_charset_utf8mb3_bin.mbmaxlen;    /* ★ 按客户端字符集最小字节数估算上界 */
  m_body_utf8 = (char *)thd->alloc(body_utf8_length + 1);
  m_body_utf8_ptr = m_body_utf8;
  *m_body_utf8_ptr = 0;
  m_cpp_utf8_processed_ptr = begin_ptr;
}

void Lex_input_stream::body_utf8_append(const char *ptr, const char *end_ptr) {
  if (!m_body_utf8) return;              /* ★ 非 SP 语句直接返回，不产生第二份文本 */
  ...
  memcpy(m_body_utf8_ptr, m_cpp_utf8_processed_ptr, bytes_to_copy);
  m_body_utf8_ptr += bytes_to_copy;
  *m_body_utf8_ptr = 0;
  m_cpp_utf8_processed_ptr = end_ptr;
}

/* ★ 只有文本字面量才真正做字符集转换；标识符/关键字是 memcpy 原样拷贝 */
void Lex_input_stream::body_utf8_append_literal(THD *thd, const LEX_STRING *txt,
                                                const CHARSET_INFO *txt_cs,
                                                const char *end_ptr) {
  if (!m_cpp_utf8_processed_ptr) return;
  if (!my_charset_same(txt_cs, &my_charset_utf8mb3_general_ci)) {
    thd->convert_string(&utf_txt, &my_charset_utf8mb3_general_ci, txt->str,
                        txt->length, txt_cs);   /* ★ 目标是 utf8mb3 */
  }
  ...
}
```

```cpp
/* sql/sp_head.cc:1772 —— 收割点 */
void sp_head::set_body_start(THD *thd, const char *begin_ptr) {
  m_parser_data.set_body_start_ptr(begin_ptr);
  thd->m_parser_state->m_lip.body_utf8_start(thd, begin_ptr);   /* ★ 起点 */
}

void sp_head::set_body_end(THD *thd) {
  ...
  m_body = to_lex_cstring(body);          /* ★ 第一份：原字符集 */
  lip->body_utf8_append(end_ptr);         /* ★ 把最后一段刷进 utf8 缓冲 */
  body_utf8.str = thd->strmake(lip->get_body_utf8_str(), body_utf8.length);
  m_body_utf8 = to_lex_cstring(body_utf8);/* ★ 第二份：UTF8 */
}
```

#### 1.6 事务与 MDL 都在调用方

`sp_create_routine()`（`sql/sp.cc`）的顺序：`lock_object_name(MDL_key::FUNCTION/PROCEDURE)` → `Auto_releaser` → 存在性检查 → `precheck` → `acquire(schema)` → `dd::create_routine` → **`update_referencing_views_metadata()`（反向依赖维护）** → binlog → `trans_commit_stmt + trans_commit` → `sp_cache_invalidate()`。

失败路径必须 **full rollback**，注释解释了原因：

```
    Full rollback in case we have THD::transaction_rollback_request
    and to synchronize DD state in cache and on disk (as statement
    rollback doesn't clear DD cache of modified uncommitted objects).
```

### 二、`dd_trigger.cc`：触发器 → `mysql.triggers`

#### ★ 2.1 读路径：`load_triggers()`

调用链：`open_table_entry_fini()` → `Table_trigger_dispatcher::check_n_load()` → `dd::load_triggers()`。

**只有 `dd::Table::has_trigger()` 为真才加载**——这是读路径的开关：

```c
if (table != nullptr && table->has_trigger()) {
    Table_trigger_dispatcher *d = Table_trigger_dispatcher::create(entry);
    if (!d || d->check_n_load(thd, *table)) { destroy(d); return true; }
```

★ **`load_triggers()` 本身不做任何 DD 查询**——它遍历 `table.triggers()`，即触发器是随 `dd::Table` 一起从缓存/存储带出来的**子集合**。schema 名/表名由调用方传入，不从 trigger 行读。

读到后做三件事：`lex_string_strmake` 拷进 mem_root → 三个 collation id 转 `CHARSET_INFO`（**为 nullptr 时回落会话变量**，不报错）→ `::Trigger::create_from_dd(...)` 重建 SQL 层对象。

⚠️ 读写对称的一个细节：`client_cs_name` 用的是**字符集名**，`connection_cl_name` / `db_cl_name` 用的是**排序规则名**——与写路径的 `resolve_charset` vs `resolve_collation` 严格对应。

#### 2.2 写路径：`create_trigger()`

```c
  if (dd_client->acquire_for_modification(new_trigger->get_db_name().str,
                                          new_trigger->get_subject_table_name().str, &new_table)) return true;
```

**第一步是 `acquire_for_modification` 拿到 `dd::Table`**。然后按 `FOLLOWS` / `PRECEDES` 插入：

```c
  if (ordering_clause == TRG_ORDER_FOLLOWS)
    dd_trig_obj = new_table->add_trigger_following(referenced_trg, ...);
  else
    dd_trig_obj = new_table->add_trigger_preceding(...);
...
  if (fill_in_dd_trigger_object(new_trigger, dd_trig_obj)) return true;
  return dd_client->update(new_table);     // ★ update 的是 Table
```

★ **`action_order` 不在 `fill_in_dd_trigger_object` 里设**——由 `Table_impl` 在插入后统一重排（`reorder_action_order(at, et)`）。

字段映射：`name` / `definer_user` + `definer_host`（分两列，落库拼成 `definer VARCHAR(288)`）/ `event_type` / `action_timing` / `action_statement`(LONGBLOB) + `action_statement_utf8`(LONGTEXT) / `sql_mode` / 三个 collation id。

三条 `resolve_*` 失败都带同一句注释（说明为什么必须自己报错）：

```c
    // resolve_charset will not cause an error to be reported if the
    // collation was not found, so we must report error here.
```

#### 2.3 `table_has_triggers()`——**没有缓存**

```c
// Only used by NDB
bool table_has_triggers(THD *thd, const char *schema_name,
                        const char *table_name, bool *table_has_trigger) {
  cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
  const Table *table = nullptr;
  if (thd->dd_client()->acquire(schema_name, table_name, &table)) return true;
  *table_has_trigger = (table != nullptr && table->has_trigger());
  return false;
}
```

⚠️ 判定就是 `m_triggers.size() > 0`，**每次 `acquire()`，靠 `Dictionary_client` 的对象缓存**；除声明/定义外**全库无调用点**（NDB 用自己的 `ndb_dd_table_has_trigger`）。

#### ★ 2.4 从属关系：`table_id` 外键，级联在对象层

```c
"UNIQUE KEY (schema_id, name)"
"UNIQUE KEY (table_id, event_type, action_timing, action_order)"    // ★
FOREIGN KEY (table_id)  REFERENCES tables(id)
```

第二条唯一键 = "同表同事件同时机内 `action_order` 唯一"，这正是 `reorder_action_order` 维护的不变式，也直接支撑 `FOLLOWS/PRECEDES` 语义。

★ **级联删除在对象层，不在 InnoDB 层**——`Table_impl::drop_children()` 按 `Triggers::create_key_by_table_id()` 批量删。**源码中没有 `ON DELETE CASCADE`**（DD 表的 FK 是元数据记账，删除靠对象层递归）。

ALTER 时需要"搬家"：`copy_triggers()`，且必须重置主键（注释解释了不重置会怎样），配套 `drop_all_triggers()`。

**★ `create_trigger()` 完整源码**（`sql/dd/dd_trigger.cc:178`）：

```cpp
bool create_trigger(THD *thd, const ::Trigger *new_trigger,
                    enum_trigger_order_type ordering_clause,
                    const LEX_CSTRING &referenced_trigger_name) {
  cache::Dictionary_client *dd_client = thd->dd_client();
  cache::Dictionary_client::Auto_releaser releaser(dd_client);
  Table *new_table = nullptr;

  DBUG_EXECUTE_IF("create_trigger_fail", {
    my_error(ER_LOCK_DEADLOCK, MYF(0));
    return true;
  });

  if (dd_client->acquire_for_modification(          /* ★ 第一步：拿 dd::Table 的修改权 */
          new_trigger->get_db_name().str,
          new_trigger->get_subject_table_name().str, &new_table)) {
    return true;      // Error is reported by the dictionary subsystem.
  }
  assert(new_table != nullptr);

  Trigger *dd_trig_obj;
  if (ordering_clause != TRG_ORDER_NONE) {
    const Trigger *referenced_trg =
        new_table->get_trigger(referenced_trigger_name.str);
    /*
      Checking for presence of a trigger referenced by FOLLOWS/PRECEDES clauses
      is done in Trigger_chain::add_trigger() ... Therefore it just added assert
      to check this invariant.
    */
    assert(referenced_trg != nullptr);
    if (ordering_clause == TRG_ORDER_FOLLOWS)
      dd_trig_obj = new_table->add_trigger_following(
          referenced_trg, get_dd_action_timing(new_trigger),
          get_dd_event_type(new_trigger));
    else
      dd_trig_obj = new_table->add_trigger_preceding(...);
  } else
    dd_trig_obj = new_table->add_trigger(get_dd_action_timing(new_trigger),
                                         get_dd_event_type(new_trigger));

  if (dd_trig_obj == nullptr) return true;

  if (fill_in_dd_trigger_object(new_trigger, dd_trig_obj)) return true;

  return dd_client->update(new_table);   /* ★★ update 的是 Table，不是 Trigger */
}
```

**★★ `fill_in_dd_trigger_object()` 完整源码**（★ 三条 `resolve_*` 的注释**一字不差地相同**，且后两条实际调的是 `resolve_collation`，注释却写 `resolve_charset`——源码原样）：

```cpp
static bool fill_in_dd_trigger_object(const ::Trigger *new_trigger,
                                      Trigger *dd_trig_obj) {
  dd_trig_obj->set_name(...);
  dd_trig_obj->set_definer(new_trigger->get_definer_user().str,
                           new_trigger->get_definer_host().str);

  dd_trig_obj->set_event_type(get_dd_event_type(new_trigger));
  dd_trig_obj->set_action_timing(get_dd_action_timing(new_trigger));

  dd_trig_obj->set_action_statement(new_trigger->get_definition().str);
  dd_trig_obj->set_action_statement_utf8(new_trigger->get_definition_utf8().str);
  dd_trig_obj->set_sql_mode(new_trigger->get_sql_mode());

  const CHARSET_INFO *collation;

  if (resolve_charset(new_trigger->get_client_cs_name().str,
                      system_charset_info, &collation)) {
    // resolve_charset will not cause an error to be reported if the
    // collation was not found, so we must report error here.
    my_error(ER_UNKNOWN_COLLATION, MYF(0), new_trigger->get_client_cs_name().str);
    return true;
  }
  dd_trig_obj->set_client_collation_id(collation->number);

  if (resolve_collation(new_trigger->get_connection_cl_name().str,
                        system_charset_info, &collation)) {
    // resolve_charset will not cause an error to be reported if the
    // collation was not found, so we must report error here.     /* ★★ 注释与函数名对不上 */
    my_error(ER_UNKNOWN_COLLATION, MYF(0), new_trigger->get_connection_cl_name().str);
    return true;
  }
  dd_trig_obj->set_connection_collation_id(collation->number);

  if (resolve_collation(new_trigger->get_db_cl_name().str, system_charset_info,
                        &collation)) {
    // resolve_charset will not cause an error to be reported if the
    // collation was not found, so we must report error here.     /* ★★ 同上 */
    my_error(ER_UNKNOWN_COLLATION, MYF(0), new_trigger->get_db_cl_name().str);
    return true;
  }
  dd_trig_obj->set_schema_collation_id(collation->number);

  return false;
  /*  ★ 本函数不写 id / created / last_altered / action_order */
}
```

**★ `load_triggers()` 完整源码**（★ 遍历 `table.triggers()`，**不做任何 DD 查询**）：

```cpp
bool load_triggers(THD *thd, MEM_ROOT *mem_root, const char *schema_name,
                   const char *table_name, const dd::Table &table,
                   List<::Trigger> *triggers) {
  for (const auto &trigger : table.triggers()) {    /* ★★ 已经是 dd::Table 的子集合 */
    LEX_CSTRING db_name_str = {schema_name, strlen(schema_name)};
    LEX_CSTRING subject_table_name = {table_name, strlen(table_name)};
    ...
    if (lex_string_strmake(mem_root, &definition,
                           trigger->action_statement().c_str(), ...)) return true;
    /*  ★ 全部拷进调用方 mem_root —— DD 对象不持有这些字符串 */

    const CHARSET_INFO *client_cs =
        dd_get_mysql_charset(trigger->client_collation_id());
    if (client_cs == nullptr) client_cs = thd->variables.character_set_client;
    /*  ★★ 三个 collation id 查不到时回落会话变量，不报错 */
    ...
    ::Trigger *trigger_to_add = ::Trigger::create_from_dd(
        mem_root, trigger_name, db_name_str, subject_table_name, definition,
        definition_utf8, trigger->sql_mode(), definer_user, definer_host,
        client_cs_name, connection_cl_name, db_cl_name,
        convert_event_type_from_dd(trigger->event_type()),
        convert_action_time_from_dd(trigger->action_timing()),   /* ★ 真名是 _time_ */
        trigger->action_order(), trigger->created());
    ...
  }
  return false;
}
```

> ⚠️ 真名 **`convert_action_time_from_dd`**（不是 `convert_action_timing_from_dd`）。
> ⚠️ **`dd_trigger.cc` 里没有 `drop_trigger` / `update_trigger`**（全文件只有 8 个函数）。触发器更新统一走 `dd_client->update(new_table)`；`drop_trigger` 是 `dd::Table` 的虚函数，实现在 `table_impl.cc:1015`。

**★ `reorder_action_order()` / `copy_triggers()` 完整源码**（`table_impl.cc`）：

```cpp
void Table_impl::reorder_action_order(Trigger::enum_action_timing at,
                                      Trigger::enum_event_type et) {
  uint new_order = 1;
  for (Trigger *trigger : *triggers()) {
    if (trigger->action_timing() == at && trigger->event_type() == et)
      trigger->set_action_order(new_order++);   /* ★ 只在同一 (timing,event) 组内从 1 连续编号 */
  }
}

void Table_impl::drop_trigger(const Trigger *trigger) {
  dd::Trigger::enum_action_timing at = trigger->action_timing();
  dd::Trigger::enum_event_type et = trigger->event_type();
  m_triggers.remove(dynamic_cast<Trigger_impl *>(const_cast<Trigger *>(trigger)));
  reorder_action_order(at, et);   /* ★ 删后必须重排，否则组内出现空洞 */
}

void Table_impl::drop_all_triggers() { m_triggers.remove_all(); }
```

```cpp
void Table_impl::copy_triggers(const Table *tab_obj) {
  for (const Trigger *trig : tab_obj->triggers()) {
    /*
      Reset the trigger primary key ID, so that a new row is created for them,
      when the object is stored. Following is the issue if we don't do that.
      * dd::Table::copy_triggers() retained the old trigger ID's ...
      * thd->dd_client()->update() ... would insert these triggers with same
        old trigger ID ...
      * After inserting a row, we set dd::Trigger_impl::m_id only if a new id
        m_table->file->insert_id_for_cur_row was generated. The problem here is
        that there was no new row ID generated as we did retain old
        mysql.triggers.id. Hence we end-up marking the dd::Trigger_impl::m_id
        as INVALID_OBJECT_ID.
      * Later if the same object is updated (may be rename operation) then as
        the dd::Trigger_impl::m_id is INVALID_OBJECT_ID, we end-up creating a
        duplicate row which already exists.
      So ... dd::Table::copy_triggers() API now sets the ID's of cloned trigger
      objects to INVALID_OBJECT_ID.
    */
    Trigger_impl *new_trigger =
        new Trigger_impl(*dynamic_cast<const Trigger_impl *>(trig), this);
    new_trigger->set_id(INVALID_OBJECT_ID);   /* ★★ 必须重置主键 */
    m_triggers.push_back(new_trigger);
  }
}
```

**★ 级联在对象层的硬证据：`Table_impl::drop_children()`**（`table_impl.cc:431`）：

```cpp
bool Table_impl::drop_children(Open_dictionary_tables_ctx *otx) const {
  // Note that partition collection has to be dropped first
  // as it has foreign key to indexes.
  return m_check_constraints.drop_items(...) ||
         m_triggers.drop_items(otx, otx->get_table<Trigger>(),
                               Triggers::create_key_by_table_id(this->id())) ||
         m_partitions.drop_items(...) ||
         m_foreign_keys.drop_items(...) ||
         m_indexes.drop_items(...) ||
         Abstract_table_impl::drop_children(otx);
}
```

> ★★ 顺序：`check_constraints → triggers → partitions → foreign_keys → indexes → columns`。注释说明 **partition 必须在 index 之前**（因为 partition 对 index 有外键）。
> ⚠️ **`Triggers::create_key_by_table_id()`**（`triggers.cc:139`）就是这条级联路径用的范围键——它走的是 `UNIQUE KEY(table_id, event_type, action_timing, action_order)`。

**★ `mysql.triggers` 表定义**（`sql/dd/impl/tables/triggers.cc:45`）：

```c
  "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT"
  "schema_id BIGINT UNSIGNED NOT NULL"
  "name VARCHAR(64) NOT NULL COLLATE utf8mb3_general_ci"
  "event_type ENUM('INSERT','UPDATE','DELETE') NOT NULL"
  "table_id BIGINT UNSIGNED NOT NULL"
  "action_timing ENUM('BEFORE','AFTER') NOT NULL"
  "action_order INT UNSIGNED NOT NULL"
  "action_statement LONGBLOB NOT NULL"
  "action_statement_utf8 LONGTEXT NOT NULL"
  "created TIMESTAMP(2) NOT NULL"  "last_altered TIMESTAMP(2) NOT NULL"  /* ★ 注意是 (2) */
  "sql_mode ..."  "definer VARCHAR(288) NOT NULL"
  "client_collation_id BIGINT UNSIGNED NOT NULL"
  "connection_collation_id BIGINT UNSIGNED NOT NULL"
  "schema_collation_id BIGINT UNSIGNED NOT NULL"
  "options MEDIUMTEXT"
  ...
  "PRIMARY KEY(id)"
  "UNIQUE KEY (schema_id, name)"
  "UNIQUE KEY (table_id, event_type, action_timing, action_order)"   /* ★★ */
  "KEY(client_collation_id)" "KEY(connection_collation_id)"
  "KEY(schema_collation_id)" "KEY(definer)"
  "FOREIGN KEY (schema_id) REFERENCES schemata(id)"
  "FOREIGN KEY (table_id) REFERENCES tables(id)"
  /* 加 3 条 collation FK */
```

### 三、`dd_event.cc`：事件 → `mysql.events`

#### ★ 3.1 时区：UTC 化不在 `dd_event.cc`

这是最容易被误解的一点：**`dd_event.cc` 里没有任何时区换算**。UTC 化在 `Event_parse_data` 的 `init_*` 阶段就完成了：

```c
  if ((item_execute_at->get_date(&ltime, TIME_NO_ZERO_DATE))) goto wrong_value;
  bool is_in_dst_gap_ignored;
  ltime_utc = thd->time_zone()->TIME_to_gmt_sec(&ltime, &is_in_dst_gap_ignored);
  if (!ltime_utc) { DBUG_PRINT("error", ("Execute AT after year 2037")); goto wrong_value; }
  check_if_in_the_past(thd, ltime_utc);
  execute_at_null = false;
  execute_at = ltime_utc;
```

三个语义要点：

1. `thd->time_zone()` 是**会话时区**，`TIME_to_gmt_sec` 把本地时间解释成 UTC epoch 秒
2. **时区名单独存一列** `time_zone VARCHAR(64)`，读出时用 `CONVERT_TZ(execute_at, '+00:00', time_zone)` 还原——这正是 `system_views.md` 里 I_S `EVENTS` 视图侧 `CONVERT_TZ` 的**反向**
3. "过去时间"检查也在 UTC 域做

落库时 `my_time_t` ↔ DATETIME 的转换由 `Raw_record::store_time()` / `read_time()` 完成。

#### 3.2 `set_event_attributes()` 的三个"条件更新"

```c
  if (!is_update || event_data->status_changed)
    event->set_event_status(get_enum_event_status(event_data->status));
  ...
  if (event_data->body_changed) {
    event->set_sql_mode(thd->variables.sql_mode);
    event->set_definition_utf8(event_body_utf8);
    event->set_definition(event_body);
  }
```

即 ALTER 时：**只有 `ENABLE/DISABLE` 显式出现才改 status；只有体变了才重写 definition。**

⚠️ `time_zone` 的写入条件**不对称**：`expression` 分支要求 `!is_update || !starts_null`，`execute_at` 分支则**无条件**写。

#### ★ 3.3 `originator` 与 `SLAVESIDE_DISABLED`

```c
/**
  Set the originator id of the event to the server_id if executing on
  the master or set to the server_id of the master if executing on
  the slave. If executing on slave, also set status to SLAVESIDE_DISABLED.
*/
```

从库线程（SQL/Worker/IO）执行 CREATE/ALTER EVENT 时，任何 `ENABLED` / `DISABLED` 都被**强制改写为 `SLAVESIDE_DISABLED`** 并置 `status_changed = true` 让 `set_event_attributes` 真正写库。

⇒ **这就是"事件不会在从库被执行两次"的 DD 层保障**。`originator` = 创建该事件的源服务器 `server_id`。

#### 3.4 枚举转换：3 组 6 个（不是 4 组）

| 方向 | 函数 | 枚举 |
|---|---|---|
| DD→server | `get_old_status` / `get_old_on_completion` / `get_old_interval_type` | **非 static**（对外可见，供 I_S / upgrade 复用） |
| server→DD | `get_enum_event_status` / `get_on_completion` / `get_enum_interval_field` | **static**（文件内私有） |

⇒ 命名风格本身就是"**写入侧封装、读出侧开放**"的设计信号。

两处细节：
- `get_on_completion` 把 `ON_COMPLETION_DEFAULT` 折叠进 `OC_DROP`（默认值归一化）
- 六个函数**共用** `static const char *failsafe_object = "Event status option";`——on_completion / interval_field 出错时日志也说 "Event status option"，**文案不准确**

**★ `set_event_attributes()` 完整源码**（`sql/dd/dd_event.cc:255`，★ 三个"条件更新" + `time_zone` 不对称）：

```cpp
static void set_event_attributes(THD *thd, const dd::Schema &schema,
                                 Event *event, const String_type &event_name,
                                 const String_type &event_body,
                                 const String_type &event_body_utf8,
                                 const LEX_USER *definer,
                                 Event_parse_data *event_data, bool is_update) {
  event->set_name(event_name);
  event->set_definer(definer->user.str, definer->host.str);

  // Set last altered time.
  event->set_last_altered(dd::my_time_t_to_ull_datetime(thd->query_start_in_secs()));

  // Set Event on completion and status.
  event->set_on_completion(get_on_completion(event_data->on_completion));
  if (!is_update || event_data->status_changed)      /* ★ 条件更新 ① */
    event->set_event_status(get_enum_event_status(event_data->status));
  event->set_originator(event_data->originator);     /* ★ originator 无条件写入 */

  // Set Event definition attributes.
  if (event_data->body_changed) {                    /* ★ 条件更新 ② */
    event->set_sql_mode(thd->variables.sql_mode);
    event->set_definition_utf8(event_body_utf8);
    event->set_definition(event_body);
  }

  // Set Event scheduling attributes.
  if (event_data->expression) {
    const String *tz_name = thd->variables.time_zone->get_name();
    if (!is_update || !event_data->starts_null)      /* ★★ 不对称点 A：有守卫 */
      event->set_time_zone(tz_name->ptr());

    event->set_interval_value_null(false);
    event->set_interval_value(event_data->expression);
    event->set_interval_field_null(false);
    event->set_interval_field(get_enum_interval_field(event_data->interval));
    event->set_execute_at_null(true);
    if (!event_data->starts_null) { event->set_starts_null(false); event->set_starts(event_data->starts); }
    else                          { event->set_starts_null(true); }
    if (!event_data->ends_null)   { event->set_ends_null(false);   event->set_ends(event_data->ends); }
    else                          { event->set_ends_null(true); }
  } else if (event_data->execute_at) {
    const String *tz_name = thd->variables.time_zone->get_name();
    event->set_time_zone(tz_name->ptr());            /* ★★ 不对称点 B：无条件写入 */
    event->set_interval_value_null(true);
    event->set_interval_field_null(true);
    event->set_starts_null(true);
    event->set_ends_null(true);
    event->set_execute_at_null(false);
    event->set_execute_at(event_data->execute_at);
  } else
    assert(is_update);     /* ★ 既无 EVERY 又无 AT —— 只允许发生在 ALTER EVENT */

  if (event_data->comment.str != nullptr)
    event->set_comment(String_type(event_data->comment.str));   /* ★ NULL 表示不改 */

  event->set_client_collation_id(thd->variables.character_set_client->number);
  event->set_connection_collation_id(thd->variables.collation_connection->number);

  const CHARSET_INFO *db_cl = nullptr;
  if (get_default_db_collation(schema, &db_cl)) {
    DBUG_PRINT("error", ("get_default_db_collation failed."));
    thd->clear_error();
  }
  db_cl = db_cl ? db_cl : thd->collation();
  event->set_schema_collation_id(db_cl->number);
  /*  ★ 本函数不写 created */
}
```

**★ `create_event()` / `update_event()` 完整源码**：

```cpp
bool create_event(THD *thd, const Schema &schema, const String_type &event_name,
                  const String_type &event_body, const String_type &event_body_utf8,
                  const LEX_USER *definer, Event_parse_data *event_data) {
  std::unique_ptr<dd::Event> event(schema.create_event(thd));
  set_event_attributes(thd, schema, event.get(), event_name, event_body,
                       event_body_utf8, definer, event_data, false);
  return thd->dd_client()->store(event.get());   /* ★ store() */
}

bool update_event(THD *thd, Event *event, const dd::Schema &schema,
                  const dd::Schema *new_schema, const String_type &new_event_name,
                  const String_type &new_event_body,
                  const String_type &new_event_body_utf8,
                  const LEX_USER *definer, Event_parse_data *event_data) {
  // Check whether alter event was given dates that are in the past.
  if (event_data->check_dates(thd, static_cast<int>(event->on_completion())))
    return true;   /* ★★ 绕过转换函数，把 DD 枚举裸转成 int */

  // Update Schema Id if there is a dbname change.
  if (new_schema != nullptr) event->set_schema_id(new_schema->id());

  set_event_attributes(thd, (new_schema != nullptr) ? *new_schema : schema, event,
                       new_event_name != "" ? new_event_name : event->name(),
                       new_event_body, new_event_body_utf8, definer, event_data, true);
  return thd->dd_client()->update(event);        /* ★ update() */
}

/* ★ 调度器执行完事件后的快捷路径：只回写两个字段 */
bool update_event_time_and_status(THD *thd, Event *event,
                                  my_time_t last_executed, ulonglong status) {
  event->set_event_status_null(false);
  event->set_event_status(get_enum_event_status(status));
  event->set_last_executed_null(false);
  event->set_last_executed(last_executed);
  return thd->dd_client()->update(event);
}
```

**★ `originator` / `SLAVESIDE_DISABLED` 的完整代码**（`event_parse_data.cc:518`）：

```cpp
/**
  Set the originator id of the event to the server_id if executing on
  the master or set to the server_id of the master if executing on
  the slave. If executing on slave, also set status to SLAVESIDE_DISABLED.
*/
void Event_parse_data::check_originator_id(THD *thd) {
  /* Disable replicated events on slave. */
  if ((thd->system_thread == SYSTEM_THREAD_SLAVE_SQL) ||
      (thd->system_thread == SYSTEM_THREAD_SLAVE_WORKER) ||
      (thd->system_thread == SYSTEM_THREAD_SLAVE_IO)) {
    if ((status == Event_parse_data::ENABLED) ||
        (status == Event_parse_data::DISABLED)) {
      status = Event_parse_data::SLAVESIDE_DISABLED;   /* ★★ 强制改写 */
      status_changed = true;                           /* ★★ 置位后条件更新①才真正落库 */
    }
    originator = thd->server_id;
  } else
    originator = server_id;
}
```

```cpp
/* sql/event_parse_data.h:44 —— SQL 层枚举 */
  enum enum_status { ENABLED = 1, DISABLED, SLAVESIDE_DISABLED };
  enum enum_on_completion {
    /* On CREATE EVENT, DROP is the DEFAULT as per the docs.
       On ALTER  EVENT, "no change" is the DEFAULT. */
    ON_COMPLETION_DEFAULT = 0, ON_COMPLETION_DROP, ON_COMPLETION_PRESERVE
  };
/* sql/dd/types/event.h:73 —— DD 侧枚举，从 1 起 */
  enum enum_event_status { ES_ENABLED = 1, ES_DISABLED, ES_SLAVESIDE_DISABLED };
  enum enum_on_completion { OC_DROP = 1, OC_PRESERVE };
```

**★ 时区换算的真实位置**（★ `dd_event.cc` 里**没有**任何时区换算）：

```cpp
int Event_parse_data::init_execute_at(THD *thd) {
  ...
  if ((item_execute_at->get_date(&ltime, TIME_NO_ZERO_DATE))) goto wrong_value;

  bool is_in_dst_gap_ignored;
  ltime_utc = thd->time_zone()->TIME_to_gmt_sec(&ltime, &is_in_dst_gap_ignored);
  /*  ★★ 会话时区 → UTC epoch 秒。这就是"UTC 化"的全部所在 */

  if (!ltime_utc) {
    DBUG_PRINT("error", ("Execute AT after year 2037"));
    goto wrong_value;
  }
  check_if_in_the_past(thd, ltime_utc);      /* ★ "过去时间"检查也在 UTC 域做 */
  execute_at_null = false;
  execute_at = ltime_utc;
  return 0;
wrong_value:
  report_bad_value(thd, "AT", item_execute_at);
  return ER_WRONG_VALUE;
}
```

> ★ `init_starts()` / `init_ends()` 同样调 `TIME_to_gmt_sec`，但 **`init_starts()` 不调 `check_if_in_the_past()`**（只 `init_execute_at` 与 `init_ends` 调）——源码事实。
> ★ `init_interval()` **没有**时区换算——interval 是纯数量，且组合型区间（`DAY_SECOND` 等）被折叠成**一个标量** `expression`，所以 `interval_value` 列是单个 INT。

**★ `check_if_in_the_past()` 的三分支语义**（★ 这里有第二处置 `status_changed`）：

```cpp
void Event_parse_data::check_if_in_the_past(THD *thd, my_time_t ltime_utc) {
  if (ltime_utc >= (my_time_t)thd->query_start_in_secs()) return;
  /* We'll come back later when we have the real on_completion value */
  if (on_completion == Event_parse_data::ON_COMPLETION_DEFAULT) return;
  /*  ★ ALTER 未指定 ON COMPLETION 时延迟到 check_dates() 再判 */

  if (on_completion == Event_parse_data::ON_COMPLETION_DROP) {
    switch (thd->lex->sql_command) {
      case SQLCOM_CREATE_EVENT:  push_warning(... ER_EVENT_CANNOT_CREATE_IN_THE_PAST ...); break;
      case SQLCOM_ALTER_EVENT:   my_error(ER_EVENT_CANNOT_ALTER_IN_THE_PAST, MYF(0)); break;
      default: assert(0);
    }
    do_not_create = true;    /* ★ 置位后由 sql/events.cc 决定不创建/不 ALTER */
  } else if (status == Event_parse_data::ENABLED) {
    status = Event_parse_data::DISABLED;
    status_changed = true;   /* ★★ 第二处置位 */
    push_warning(thd, ... ER_EVENT_EXEC_TIME_IN_THE_PAST ...);
  }
}
```

**★ `mysql.events` 表定义**（`sql/dd/impl/tables/events.cc:40`）：

```c
  "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT"
  "schema_id BIGINT UNSIGNED NOT NULL"
  "name VARCHAR(64) NOT NULL COLLATE utf8mb3_general_ci"
  "definer VARCHAR(288) NOT NULL"
  "time_zone VARCHAR(64) NOT NULL"        /* ★★ 时区以名字存，无索引、无外键 */
  "definition LONGBLOB NOT NULL"  "definition_utf8 LONGTEXT NOT NULL"
  "execute_at DATETIME"
  "interval_value INT"
  "interval_field ENUM('YEAR','QUARTER','MONTH','DAY','HOUR','MINUTE','WEEK',
     'SECOND','MICROSECOND','YEAR_MONTH','DAY_HOUR','DAY_MINUTE','DAY_SECOND',
     'HOUR_MINUTE','HOUR_SECOND','MINUTE_SECOND','DAY_MICROSECOND',
     'HOUR_MICROSECOND','MINUTE_MICROSECOND','SECOND_MICROSECOND')"
  "sql_mode ..."  "starts DATETIME"  "ends DATETIME"
  "status ENUM('ENABLED','DISABLED','SLAVESIDE_DISABLED') NOT NULL"
  "on_completion ENUM('DROP','PRESERVE') NOT NULL"
  "created TIMESTAMP NOT NULL"  "last_altered TIMESTAMP NOT NULL"
  "last_executed DATETIME"  "comment VARCHAR(2048) NOT NULL"
  "originator INT UNSIGNED NOT NULL"      /* ★★ triggers 表没有这列 */
  "client_collation_id BIGINT UNSIGNED NOT NULL"
  "connection_collation_id BIGINT UNSIGNED NOT NULL"
  "schema_collation_id BIGINT UNSIGNED NOT NULL"
  "options MEDIUMTEXT"
  ...
  "PRIMARY KEY(id)"  "UNIQUE KEY(schema_id, name)"   /* ★ 只有 1 条唯一键 */
  "KEY(client_collation_id)" "KEY(connection_collation_id)"
  "KEY(schema_collation_id)" "KEY(definer)"
  "FOREIGN KEY (schema_id) REFERENCES schemata(id)"  /* ★ 挂在 schema 下，无 table_id */
```

> ★ `enum_interval_field` 的顺序（`IF_DAY=4` 在 `IF_HOUR=5` 前、`IF_WEEK=7` 夹在 `IF_MINUTE` 与 `IF_SECOND` 之间）与表里 ENUM 字面量顺序**完全一致**——这是映射正确性的关键。

### 四、`dd_view.cc`：视图 → `mysql.tables`

#### 4.1 `create_view` / `update_view`

```c
  if (fill_dd_view_definition(thd, view_obj.get(), view)) return true;
  return thd->dd_client()->store(view_obj.get());
```

系统视图走 `schema.create_system_view(thd)` + `set_hidden(HT_HIDDEN_SYSTEM)`。

★ `update_view()` 是**全量重算而非增量 diff**：

```c
  // Clear the columns, tables and routines since it will be added later.
  new_view->remove_children();
```
```c
void View_impl::remove_children() {
  columns()->remove_all();      /* ① mysql.columns 子集合 */
  m_column_names.clear();       /* ② 显式列名 Properties（不在 columns() 里，是独立第二处状态） */
  m_tables.remove_all();        /* ③ view_table_usage 子集合 */
  m_routines.remove_all();      /* ④ view_routine_usage 子集合 */
}
```

**★ `create_view()` / `update_view()` 完整源码**（`sql/dd/dd_view.cc:575`）：

```cpp
bool update_view(THD *thd, dd::View *new_view, Table_ref *view) {
  // Clear the columns, tables and routines since it will be added later.
  new_view->remove_children();     /* ★★ 全量重算，没有增量 diff */
  new_view->set_last_altered(
      dd::my_time_t_to_ull_datetime(thd->query_start_in_secs()));  /* ★ 只有 update 刷新 */
  if (fill_dd_view_definition(thd, new_view, view)) return true;
  return thd->dd_client()->update(new_view);   /* ★ 不提交事务 */
}

bool create_view(THD *thd, const dd::Schema &schema, Table_ref *view) {
  bool hidden_system_view = false;
  std::unique_ptr<dd::View> view_obj;
  if (dd::get_dictionary()->is_system_view_name(view->db, view->table_name,
                                                &hidden_system_view)) {
    view_obj.reset(schema.create_system_view(thd));      /* ★ I_S 系统视图 */
    if (hidden_system_view)
      view_obj->set_hidden(dd::Abstract_table::HT_HIDDEN_SYSTEM);
  } else
    view_obj.reset(schema.create_view(thd));             /* ★ 普通 CREATE VIEW */

  if (fill_dd_view_definition(thd, view_obj.get(), view)) return true;

  // Store info in DD views table.      /* ★★ 过时注释：实落 mysql.tables */
  return thd->dd_client()->store(view_obj.get());        /* ★ 不提交事务 */
}
```

> ⚠️ **`remove_children()` 只在 `update_view()` 里出现**，`create_view()` 里没有（新对象本来就是空的）。
> ⚠️ `create_view` / `update_view` **都不提交事务**——头文件注释明写 "The caller must rollback both statement and transaction on failure"。只有 `update_view_status()` 在 `commit_dd_changes==true` 时自带提交。

**★ `fill_dd_view_tables()` 完整源码**（`sql/dd/dd_view.cc:386`，★ 三条过滤 + 名字级去重）：

```cpp
static void fill_dd_view_tables(View *view_obj, const Table_ref *view,
                                const Table_ref *query_tables) {
  for (const Table_ref *table = query_tables; table != nullptr;
       table = table->next_global) {      /* ★ 遍历 next_global：整条全局表链 */
    /*
      Skip if table is not directly referred by a view or if table is a
      data-dictionary or temporary table.
    */
    if ((table->referencing_view && table->referencing_view != view) ||  /* ① 别视图展开带入的 */
        get_dictionary()->is_dd_table_name(table->get_db_name(),
                                           table->get_table_name()) ||   /* ② mysql 系统表 */
        is_temporary_table(const_cast<Table_ref *>(table)))              /* ③ 临时表 */
      continue;

    LEX_CSTRING db_name = {table->db, table->db_length};
    LEX_CSTRING table_name = {table->table_name, table->table_name_length};

    // Avoid duplicate entries.     /* ★★ 第四条：名字级去重 */
    {
      bool duplicate_vw = false;
      for (const View_table *vw : view_obj->tables()) {
        if (!strcmp(vw->table_schema().c_str(), db_name.str) &&
            !strcmp(vw->table_name().c_str(), table_name.str)) {
          /*  ★★ 纯 strcmp：不比较 catalog，也不比较 object_id —— 依赖是"名字级" */
          duplicate_vw = true;
          break;
        }
      }
      if (duplicate_vw) continue;
    }

    View_table *view_table_obj = view_obj->add_table();
    view_table_obj->set_table_catalog(Dictionary_impl::default_catalog_name());  /* 恒 "def" */
    view_table_obj->set_table_schema(String_type(db_name.str, db_name.length));
    view_table_obj->set_table_name(String_type(table_name.str, table_name.length));
  }
}
```

**★ `fill_dd_view_routines()` 完整源码**（★ 只有存储函数）：

```cpp
static void fill_dd_view_routines(View *view_obj, Query_tables_list *routines_ctx) {
  // View stored functions. We need only directly used routines.
  for (Sroutine_hash_entry *rt = routines_ctx->sroutines_list.first;
       rt != nullptr && rt != *routines_ctx->sroutines_list_own_last;
       /*  ★★ 上界是 sroutines_list_own_last 而非链表尾 —— 只取本 LEX"自有"的 routine，
            函数体内嵌套调用的 routine 不算作视图的直接依赖 */
       rt = rt->next) {
    View_routine *view_sf_obj = view_obj->add_routine();
    /*
      We should get only stored functions here, as procedures are not directly
      used by views, and thus not stored as dependencies.
    */
    assert(rt->type() == Sroutine_hash_entry::FUNCTION);   /* ★★ 唯一断言 */
    view_sf_obj->set_routine_catalog(Dictionary_impl::default_catalog_name());
    view_sf_obj->set_routine_schema(String_type(rt->db(), rt->m_db_length));
    view_sf_obj->set_routine_name(String_type(rt->name(), rt->name_length()));
  }
  /*  ★ 这里没有去重逻辑 —— sroutines_list 本身已是 hash 去重后的集合 */
}
```

**★ `update_view_status()` 完整源码**（一字不漏）：

```cpp
bool update_view_status(THD *thd, const char *schema_name,
                        const char *view_name, bool status,
                        bool commit_dd_changes) {
  dd::cache::Dictionary_client *client = thd->dd_client();
  dd::cache::Dictionary_client::Auto_releaser releaser(client);
  dd::View *new_view = nullptr;
  if (client->acquire_for_modification(schema_name, view_name, &new_view))
    return true;
  if (new_view == nullptr) return false;   /* ★★ 视图不存在 → 静默"成功" */

  // Update view error status.
  dd::Properties *view_options = &new_view->options();
  view_options->set("view_valid", status);  /* ★ 只改 options 里的布尔键 */

  Implicit_substatement_state_guard substatement_guard(thd);

  if (client->update(new_view)) {
    if (commit_dd_changes) {
      trans_rollback_stmt(thd);
      trans_rollback(thd);
    }
    return true;
  }
  return commit_dd_changes && (trans_commit_stmt(thd) || trans_commit(thd));
  /*  ★★ 自带提交，且仅当 commit_dd_changes==true */
}
```

**★ `mysql.tables` 的 view_* 列完整定义**（`sql/dd/impl/tables/tables.cc:130`）：

```c
  "view_definition LONGBLOB"           /* ★ 客户端原始字节流，不转码 */
  "view_definition_utf8 LONGTEXT"      /* ★ UTF8 转码后，SHOW CREATE VIEW / I_S 读这条 */
  "view_check_option ENUM('NONE','LOCAL','CASCADED')"
  "view_is_updatable ENUM('NO','YES')"
  "view_algorithm ENUM('UNDEFINED','TEMPTABLE','MERGE')"
  "view_security_type ENUM('DEFAULT','INVOKER','DEFINER')"
  "view_definer VARCHAR(288)"          /* ★ 288 = user(32)+'@'+host(255) */
  "view_client_collation_id BIGINT UNSIGNED"
  "view_connection_collation_id BIGINT UNSIGNED"
  "view_column_names LONGTEXT"         /* ★ Properties 序列化，key = "1","2",... */
  ...
  "KEY(view_client_collation_id)" "KEY(view_connection_collation_id)"
  "KEY(type, view_definer)"            /* ★★ 复合索引，供 definer 反向查找 */
```

```cpp
/* tables.cc:263 —— 该索引的使用者，且 type 被钉死为 USER_VIEW */
Object_key *Tables::create_key_by_definer(const String_type &definer) {
  /*
    The system views are managed by the server itself, so an arbitrary user
    should not be able to modify them, or to create additional system views.
    Thus, there should not be a need for validating their definers against
    users that are dropped, created or renamed, and hence, we can leave out
    the system views when searching for user referenced as view definers.
  */
  return new (std::nothrow) View_definer_reference_range_key(
      INDEX_K_TYPE_VIEW_DEFINER, FIELD_TYPE,
      static_cast<uint>(enum_table_type::USER_VIEW), FIELD_VIEW_DEFINER, definer);
}
```

**★ `view_table_usage` / `view_routine_usage` 表定义**（★ 三种排序规则不一致）：

```c
/* view_table_usage —— 三列都用 fs_name_collation()，随 lower_case_table_names 变 */
  "view_id BIGINT UNSIGNED NOT NULL"
  "table_catalog VARCHAR(64) NOT NULL COLLATE <fs_name_collation>"
  "table_schema VARCHAR(64) NOT NULL COLLATE <fs_name_collation>"
  "table_name   VARCHAR(64) NOT NULL COLLATE <fs_name_collation>"
  "PRIMARY KEY(view_id, table_catalog, table_schema, table_name)"
  "KEY (table_catalog, table_schema, table_name)"    /* ★★ 反向索引 */
  "FOREIGN KEY (view_id) REFERENCES tables(id)"
  /*  ★★ table_catalog/schema/name 没有任何外键 —— 名字级依赖的物理体现 */

/* view_routine_usage —— 前两列用 fs_name_collation，routine_name 硬编码 */
  "routine_catalog VARCHAR(64) NOT NULL COLLATE <fs_name_collation>"
  "routine_schema  VARCHAR(64) NOT NULL COLLATE <fs_name_collation>"
  "routine_name    VARCHAR(64) NOT NULL COLLATE utf8mb3_general_ci"  /* ★★ 不同 */
  "PRIMARY KEY(view_id, routine_catalog, routine_schema, routine_name)"
  "KEY (routine_catalog, routine_schema, routine_name)"  /* ★★ 反向索引 */
  "FOREIGN KEY (view_id) REFERENCES tables(id)"
```

```cpp
/* object_table_definition_impl.h:162 —— fs_name_collation 是运行期才确定的 */
  static const CHARSET_INFO *fs_name_collation() {
    if (lower_case_table_names == 0) return &my_charset_utf8mb3_bin;      /* Linux 默认 */
    return &my_charset_utf8mb3_tolower_ci;                                /* Windows/macOS */
  }
```

> ★★ `fs_name_collation()` **读全局变量**，是运行期才确定的——这意味着 `view_table_usage` 的 collation 在建 DD 表时就固化，之后改 `lower_case_table_names` 会导致不匹配。这正是 MySQL 不允许随便改 `lctn` 的原因之一。

**★ 反向依赖维护：`update_referencing_views_metadata()`**（★ 被 `sp_create_routine` 调用的真身）：

```cpp
template <typename T>
static bool update_view_metadata(THD *thd, const char *db,
                                 const char *tbl_or_sf_name,
                                 bool commit_dd_changes,
                                 Uncommitted_tables_guard *uncommitted_tables) {
  if (is_view_metadata_update_needed(thd, db, tbl_or_sf_name)) {
    std::vector<Table_ref *> views;
    if (prepare_view_tables_list<T>(thd, db, tbl_or_sf_name, false,
                                    thd->mem_root, &views))
      return true;      /* ★ 走反向索引：T=View_table → view_table_usage；T=View_routine → view_routine_usage */
    if (views.empty()) return false;

    bool is_drop_operation = (thd->lex->sql_command == SQLCOM_DROP_TABLE ||
                              thd->lex->sql_command == SQLCOM_DROP_VIEW ||
                              thd->lex->sql_command == SQLCOM_DROP_FUNCTION ||
                              thd->lex->sql_command == SQLCOM_DROP_DB ||
                              thd->lex->sql_command == SQLCOM_UNINSTALL_PLUGIN);
    if (is_drop_operation)
      return mark_all_views_invalid<T>(...);  /* ★ DROP：直接把引用方标 invalid */

    return open_views_and_update_metadata(thd, &views, commit_dd_changes,
                                          uncommitted_tables);  /* ★ 重解析 + 重算 */
  }
  return false;
}
```

```cpp
/* sql/sp.cc:867 —— 调用点：★ 只有 FUNCTION 才需要维护视图反向依赖 */
  {
    sp_name spname({sp->m_db.str, sp->m_db.length}, sp->m_name, false);
    if (sp->m_type == enum_sp_type::FUNCTION &&
        update_referencing_views_metadata(thd, &spname)) {
      goto err_with_rollback;
    }
  }
/* sql/sp.cc:972 —— DROP 路径，仅当 mdl_type == MDL_key::FUNCTION */
  if (mdl_type == MDL_key::FUNCTION &&
      update_referencing_views_metadata(thd, name)) { ... }
```

> ★ RENAME 场景会**对新名字再走一遍**（`new_db/new_table_name != nullptr`）——因为可能有视图在 rename 之前就引用了新名。

**★ `fill_dd_view_columns()` 的关键段**（★ `Create_field` 物化 + sql_mode 清零的 RAII）：

```cpp
  // Helper class which takes care restoration of THD::variables.sql_mode and
  // delete handler created for dummy table.
  class Context_handler {
   public:
    Context_handler(THD *thd, handler *file) : m_thd(thd), m_file(file) {
      m_sql_mode = m_thd->variables.sql_mode;
      m_thd->variables.sql_mode = 0;
      /*  ★★ sql_mode 清零：视图列类型推导必须跑在"无 sql_mode 干扰"的中性环境下，
            否则 strict/pad 等模式会让同一视图在不同会话里推导出不同列类型 */
    }
    ~Context_handler() {
      m_thd->variables.sql_mode = m_sql_mode;
      destroy(m_file);
    }
   ...
  };
  ...
  for (Item *item : VisibleFields(*(thd->lex->unit->get_unit_column_types()))) {
    /*  ★ UNION 视图取的是"各 branch 合并后的 type holder item" */
    ...
    Field *tmp_field = ...;                          /* Item → 临时 Field */
    Create_field *cr_field =
        new (thd->mem_root) Create_field(tmp_field, orig_field);  /* ★ 物化成 Create_field */
    ...
    create_fields.push_back(cr_field);
  }
  // Fill view columns information from the Create_field objects.
  return fill_dd_columns_from_create_fields(thd, view_obj, create_fields, file);
  /*  ★★ 复用建表路径的同一函数，落 mysql.columns */
```

**★ `dd_view.cc` 的 3 组 6 个转换函数**（★ 命名约定：`old_*` = DD→老式 ulonglong，`new_*` = 老式→DD）：

```cpp
static ulonglong dd_get_old_view_check_type(dd::View::enum_check_option type);   /* 被 read_view() 用 */
static dd::View::enum_check_option dd_get_new_view_check_type(ulonglong type);   /* 被 fill_dd_view_definition() 用 */
static enum enum_view_algorithm dd_get_old_view_algorithm_type(dd::View::enum_algorithm);
static dd::View::enum_algorithm dd_get_new_view_algorithm_type(enum enum_view_algorithm);
static ulonglong dd_get_old_view_security_type(dd::View::enum_security_type);
static dd::View::enum_security_type dd_get_new_view_security_type(ulonglong);
/*  死代码分支形如：LogErr(ERROR_LEVEL, ER_DD_FAILSAFE, "view check option."); assert(false); */
```

> ⚠️ **`get_view_definition()` 不存在**——读取侧真名是 **`read_view()`**。
> ⚠️ 死代码文案各不相同（"view check option." / "view algorithm." / "view security type."）——**比 `dd_event.cc` 的 6 个共用一句要规范**。

**★ 资源组：`dd::Resource_group` 没有"枚举"**（★ 真名 `resourcegroups::Type`）：

```cpp
/* sql/resourcegroups/resource_group_basic_types.h:31 —— 不在 dd 命名空间！ */
namespace resourcegroups {
enum class Type { SYSTEM_RESOURCE_GROUP = 1, USER_RESOURCE_GROUP };
/*  ★ enum class 且显式从 1 起 —— 让 static_cast<int> 直接等于 DD 列 ENUM 下标 */
```

```cpp
static bool set_resource_group_attributes(
    dd::Resource_group *resource_group,
    const resourcegroups::Resource_group &res_grp_ref) {
  resource_group->set_name(res_grp_ref.name().c_str());
  resource_group->set_resource_group_type(res_grp_ref.type());   /* 直接透传，无转换 */
  resource_group->set_resource_group_enabled(res_grp_ref.enabled());

  const resourcegroups::Thread_resource_control *thr_res_ctrl =
      res_grp_ref.controller();
  resource_group->set_cpu_id_mask(thr_res_ctrl->vcpu_vector());  /* bitset<1024> → '0'/'1' 串 */
  resource_group->set_thread_priority(thr_res_ctrl->priority());

  return false;   /* ★★ 唯一的 return，恒为 false —— 没有失败路径 */
}
```

```c
/* sql/dd/impl/tables/resource_groups.cc:33 —— 全文，无任何外键 */
  "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT"
  "resource_group_name VARCHAR(64) NOT NULL COLLATE utf8mb3_general_ci"
  "resource_group_type enum('SYSTEM','USER') NOT NULL"
  "resource_group_enabled boolean NOT NULL"
  "cpu_id_mask VARCHAR(1024) NOT NULL"      /* ★ 1024 == dd::CPU_MASK_SIZE */
  "thread_priority int NOT NULL"
  "options MEDIUMTEXT"
  "PRIMARY KEY(id)"  "UNIQUE KEY (resource_group_name)"
```

```cpp
const CHARSET_INFO *Resource_groups::name_collation() {
  return &my_charset_utf8mb3_general_ci;   /* ★ 硬编码，与 lctn 无关 */
}
bool Resource_groups::update_object_key(Global_name_key *key, const String_type &name) {
  key->update(FIELD_RESOURCE_GROUP_NAME, name, name_collation());
  return false;   /* ★★ Global_name_key：全局对象，不属于任何 schema */
}
```

> ⚠️ **`dd_resource_group.h` 的注释写的是 "DD table resourcegroup"（单数），而真实表名是 `resource_groups`（复数）**——注释与源码不一致。
> ★ `update_resource_group()` 是三个写函数里**唯一没有** `Implicit_substatement_state_guard` 的（create/drop 都有）。
> ★ `cpu_id_mask` 落库/读回：`m_cpu_id_mask.to_string()` ↔ `std::bitset<CPU_MASK_SIZE>(str)`，`CPU_MASK_SIZE = 1024`。

#### ★ 4.2 依赖关系怎么采集与持久化

**(a) 表依赖** `fill_dd_view_tables()` —— 三条过滤 + 一条去重：

```c
    /*
      Skip if table is not directly referred by a view or if table is a
      data-dictionary or temporary table.
    */
    if ((table->referencing_view && table->referencing_view != view) ||
        get_dictionary()->is_dd_table_name(...) ||
        is_temporary_table(...))
      continue;
```

★ **存的是 `(catalog, schema, name)` 三元组，不是 `table_id`**——即**视图依赖是名字级的**，与触发器（`table_id` 外键）形成鲜明对比。

**(b) 例程依赖** `fill_dd_view_routines()`：

```c
    /*
      We should get only stored functions here, as procedures are not directly
      used by views, and thus not stored as dependencies.
    */
    assert(rt->type() == Sroutine_hash_entry::FUNCTION);
```

★ **只有存储函数会成为视图依赖，存储过程不会**（`assert` 保证）。

**(c) 采集时机**——依赖 `THD::lex` 那一刻指向视图自己的 LEX：

```c
  /*
    Fill view routines information in View object. It is important that
    THD::lex points to the view's LEX at this point, so information about
    directly used routines in it is correct.
  */
```

⇒ 这是**"解析态"依赖，不是"执行态"依赖**。

**(d) 落表与反向索引**

`view_table_usage`：`view_id` + `table_catalog/schema/name`，`PRIMARY KEY(view_id, ...)` + **`KEY(table_catalog, table_schema, table_name)`**。

★ 那个反向索引正是给"**哪些视图依赖这张表**"用的——它支撑 `update_referencing_views_metadata()`（A 节创建 FUNCTION 后调用的那一步）。

⚠️ 两个 catalog/schema 列用 `fs_name_collation()`（文件系统名排序规则），而 `routine_name` 用固定 `utf8mb3_general_ci`。

#### 4.3 视图的列：复用建表那条路

★ `fill_dd_view_columns()` 把 SELECT 列表的 `Item` **先物化成 `Create_field`，再复用 `fill_dd_columns_from_create_fields`**：

```c
  // Fill view columns information from the Create_field objects.
  return fill_dd_columns_from_create_fields(thd, view_obj, create_fields, file);
```

即 **视图列与表列走完全相同的持久化通道，落 `mysql.columns`**。视图"没有自己的列类型系统"——**列类型是从 SELECT 列表的表达式类型推导出来的**。

⇒ 这正是视图列类型会随底层表变化而变化、从而**需要 `view_valid` 标记**的根本原因。

还有个细节：造列时**临时把 `sql_mode` 清零**（`Context_handler` RAII）。

#### 4.4 `update_view_status()`：DD 被运行时状态反向修改

```c
  dd::Properties *view_options = &new_view->options();
  view_options->set("view_valid", status);
```

★ `status` **不是独立列**，是 `options` Properties 里的 `view_valid` 布尔键（CREATE 时置 `true`）。

调用场景：打开视图失败（底层表已不存在/列已消失）时把它写成 `false`，下次快速失败。

⇒ 这是**"DD 反向被 SQL 层运行时状态修改"的少见例子**，且该函数**自带事务提交**（不属于原始 DDL 事务）。视图不存在时**静默成功**（`if (new_view == nullptr) return false;`）。

### 五、`dd_resource_group.cc`：资源组

6.1 KB，简要即可，但有个结构性特点值得记：

```c
"PRIMARY KEY(id)"  "UNIQUE KEY (resource_group_name)"
"resource_group_type enum('SYSTEM', 'USER')"  "resource_group_enabled boolean"
"cpu_id_mask VARCHAR(1024)"  "thread_priority int"
```

- **没有 `schema_id`、没有 collation 三元组、没有外键**——资源组是**全局对象**（用 `Global_name_key` 而非 `Name_key`），这是与其他四类（都是 schema 级或 table 级子对象）在**命名空间**上的根本不同
- `cpu_id_mask` 是位图序列化成的字符串
- ★ **唯一自己提交事务的**（`trans_commit_stmt + trans_commit`），因为它的操作不属于任何 DDL 事务
- `set_resource_group_attributes()` **永远 `return false`**——没有失败路径

### 六、横向：五类的共同模式

#### 6.1 枚举转换

| 文件 | 转换 |
|---|---|
| `dd_routine` | 内联 switch（`SP_* → Routine::SDA_*`；`SP_IS_SUID → View::ST_DEFINER`；`sp_variable::MODE_* → Parameter::PM_*`） |
| `dd_trigger` | 2 组 4 个（`get_dd_*` / `convert_*_from_dd`） |
| `dd_event` | 3 组 6 个 |
| `dd_view` | 3 组 6 个 |
| `dd_resource_group` | **无**（直接透传） |

设计理由：① DD 落盘是 `ENUM('INSERT','UPDATE','DELETE')` 这类**字面量**，不受 SQL 层重构影响；② 默认值归一化点就在转换函数里；③ 穷尽性由 parser 保证（所以是 `assert(false)` 而非返回错误码）。

⚠️ **风格并不统一**：`dd_routine` 用 `assert(false); return true;`（会向上报错），`dd_trigger` 用裸 `assert(false)`，`dd_event` / `dd_view` 用 `LogErr(ER_DD_FAILSAFE) + assert(false)`。

#### 6.2 双份文本：4/4 普遍

| 对象 | 原始列 | utf8 列 |
|---|---|---|
| routine | `definition LONGBLOB` | `definition_utf8 LONGTEXT` |
| trigger | `action_statement LONGBLOB` | `action_statement_utf8 LONGTEXT` |
| event | `definition LONGBLOB` | `definition_utf8 LONGTEXT` |
| view | `view_definition LONGBLOB` | `view_definition_utf8 LONGTEXT` |

且**来源统一**：`sp_head::m_body_utf8`，由 `Lex_input_stream` 在词法扫描时增量生成（`/** UTF8-body buffer created during parsing. */`）。

同一模式还见于"类型描述"：`data_type`(ENUM) + `data_type_utf8`(MEDIUMTEXT)——**机器可读枚举 + 人类可读文本**。

**反例**：`parameter_type_elements.name` 用 `VARBINARY`，不转码。

#### 6.3 definer 与 security_type

| 对象 | definer | security_type |
|---|---|---|
| routine | ✔ `VARCHAR(288) NOT NULL` | ✔（复用 `View::enum_security_type`） |
| trigger | ✔ | ❌（触发器无 SQL SECURITY） |
| event | ✔ | ❌（事件恒以 definer 身份执行） |
| view | ✔（可空，`KEY(type, view_definer)`） | ✔ |
| resource group | ❌ | ❌ |

⇒ **definer 4/5，security_type 2/5**。

都有 `KEY(definer)` 索引——支撑"删用户时找出其创建的所有对象"。

⚠️ **DD 层不做任何权限校验**——五份文件里**没有一处 `check_access`**。校验全在 SQL 层，且都围绕 definer（如 `SYSTEM_USER` 特权隔离：`sctx->can_operate_with(definer, consts::system_user, true)`）。**DD 存的 definer 是权限判定的输入**。

#### 6.4 字符集三元组

| 对象 | client | connection | schema(db) |
|---|---|---|---|
| routine / trigger / event | ✔ | ✔ | ✔ |
| **view** | ✔ | ✔ | **✘** |
| resource group | — | — | — |

为什么存 collation id 而不是字符集名——注释说得最明白：

```c
  // Assign client collation ID. The create option specifies character
  // set name, and we store the default collation id for this character set
  // name, which implicitly identifies the character set.
```

即 **collation id 蕴含字符集**（一个 collation 唯一属于一个 charset）。

#### 6.5 从属关系与级联

| 关系 | 机制 |
|---|---|
| trigger → table | `table_id` FK，**级联靠 `Table_impl::drop_children()`** |
| parameter → routine | `routine_id` FK，`UNIQUE KEY(routine_id, ordinal_position)` |
| parameter_type_element → parameter | `parameter_id` FK，仅 ENUM/SET 才 `drop_children` |
| view_*_usage → view | `view_id` FK，**但依赖本身是名字级** |
| event → schema | `schema_id` FK |
| resource group | 无外键、无父对象 |

★ **所有"级联"都在对象层的 `drop_children()` 递归里**（`Weak_object_impl` 默认空实现 + 各 impl 覆写），`Collection::drop_items()` 先递归子层再删本层。**源码中未出现 `ON DELETE CASCADE`**。

#### 6.6 与 `dd::create_table` 的分工对比

| | 表 | 存储程序/视图/事件 | 触发器 | 资源组 |
|---|---|---|---|---|
| 写入模式 | 被动填充器 | 主动写入器 | 子对象挂载器 | 主动写入器 |
| 输入 | `HA_CREATE_INFO` + `List<Create_field>` + `KEY*` | **语义对象**（`sp_head` / `Table_ref` / `Event_parse_data`） | `::Trigger` | `Resource_group` |
| MDL | 调用方 | 调用方（`sp.cc` 的 `lock_object_name`） | `acquire_for_modification` | — |
| 事务 | 调用方 | 调用方 | 调用方 | **DD 层自己** |

---

## 已知限制与坑

| 坑 | 说明 |
|---|---|
| **视图依赖是名字级** | `view_table_usage` 只存三元组没有 `table_id`——底层表改名/重建后依赖会"找错"或失效，靠 `view_valid` 兜底 |
| **`result_char_length` 是冗余** | 源码注释自陈：只为 I_S，真正初始化从 `dd::Parameter` 读 |
| **级联不在数据库层** | 无 `ON DELETE CASCADE`，全靠对象层 `drop_children()` 递归 |
| **`parameter_type_elements` 不转码** | `VARBINARY` 存原始字节，与"双份 utf8"模式相反 |
| **`table_has_triggers()` 无缓存** | 每次 `acquire()`，且生产代码里无调用点（只 NDB 用） |
| **DD 层无权限校验** | 五份文件零 `check_access`，全在 SQL 层 |

### 有意思的点（源码层面非致命瑕疵）

★ **这 5 个文件里 `TODO` / `FIXME` / `Bug#` / `WL#` **0 命中**（对比：`dd_table.cc` 有 7 处 TODO）。所以只能列出代码层面的瑕疵：

1. `dd_event.cc` 六个转换函数共用 `failsafe_object = "Event status option"`——on_completion / interval_field 出错时日志文案不准确
2. `dd_view.cc` 的 `// Store info in DD views table.` 是**过时注释**（实落 `mysql.tables`）
3. `view_impl.cc` 里 `// NOTE: this is a copy/paste from sp_head::set_definer().` —— 显式承认复制粘贴
4. `dd_resource_group.cc` update 分支注释漏字：`// Full rollback we have THD::transaction_rollback_request.`（另两处是 "in case we have"）
5. `update_event()` 里 `static_cast<int>(event->on_completion())` **绕过转换函数裸转** DD 枚举
6. `parameter_type_elements` 的注释自问：`// We may have multiple similar element names. Do we plan to deprecate it?`

---

## Misc

### 工程技法

1. **三类写入器**（被动填充 / 主动写入 / 子对象挂载）——按"对象是否有独立生命周期"选择
2. **默认值在转换函数里归一化**——保证 DD 里存的永远是已解析的确定值，读回时不需要再推理
3. **双份文本统一由词法层生成**——不在写入时转码，避免多处重复实现
4. **collation id 蕴含字符集**——存一个 id 反推两者
5. **视图列复用建表的列通道**——不为视图另造一套类型系统，代价是列类型需从表达式推导、并引入 `view_valid` 惰性失效
6. **反向索引支撑反向依赖维护**——`view_routine_usage` 的 `KEY(routine_*)` 让"建函数后刷新引用它的视图"成为可能

### 易混淆点

| 混淆 | 澄清 |
|---|---|
| 视图有 `mysql.views` 表？ | ❌ 与表共用 `mysql.tables`，`view_*` 前缀挂同行 |
| 存储函数返回类型存在哪？ | 两处：`routines.result_*`（给 I_S）+ `parameters` 第 0 号参数（**真正权威**） |
| 触发器有独立的 DD 查询？ | ❌ 随 `dd::Table` 带出，`load_triggers()` 不做查询 |
| 时区换算在 `dd_event.cc`？ | ❌ 在 `Event_parse_data::init_*`，`dd_event.cc` 只落已 UTC 化的值 |
| 依赖是 `table_id` 外键？ | 触发器是；**视图是名字级** |
| DD 层会校验权限？ | ❌ 零 `check_access`，全在 SQL 层 |
| 存储过程会成为视图依赖？ | ❌ 只有存储函数（`assert` 保证） |

---

## 参考

- MySQL 8.0 Reference Manual → *Stored Objects* / *Data Dictionary*
- MySQL 8.0 Source Code → `sql/dd/dd_routine.cc`、`dd_trigger.cc`、`dd_event.cc`、`dd_view.cc`、`dd_resource_group.cc`；`sql/dd/impl/tables/` 下各表定义
- [../stored_program.md](../stored_program.md)（SQL 层语义：sp_head / 指令 / 缓存 / binlog）
- [system_views.md](system_views.md)（读出侧：`I_S.ROUTINES` / `EVENTS` / `TRIGGERS` 视图）
- [dd.md](dd.md)（DD 表体系 / 对象模型 / 缓存 / 升级 / 建表往返 §2.6）

---

## ★★ 真名勘误全表

| 常见说法 / 想当然 | 8.0.39 真相 |
|---|---|
| `dd_routine.cc` 有 `get_sql_data_access()` / `get_security_type()` / `get_parameter_mode()` | **都不存在**（全文件 0 命中）。该文件只有 4 个 static 函数，枚举转换是 `fill_dd_routine_info()` / `alter_routine()` 里的**内联 switch** |
| `fill_dd_routine_return_type()` | **不存在** → `fill_dd_function_return_type()`（只服务 `dd::Function`） |
| `Routine::enum_security_type` | **不存在**（`routine.h` 里只有 `enum_routine_type` + `enum_sql_data_access`）。真身在 **`sql/dd/types/view.h:58`**：`ST_DEFAULT=1, ST_INVOKER, ST_DEFINER` |
| `routines.security_type` 是 `ENUM('INVOKER','DEFINER')` | **错**，是 `ENUM('DEFAULT','INVOKER','DEFINER')`，`DEFAULT` 在首位 |
| `parameters.name` 是 `VARCHAR(64) NOT NULL` | **可空**（返回类型行无名字）；`mode` 也**无 NOT NULL** |
| `CHECK_FIELD_WARN` 只在 FUNCTION 分支 | **FUNCTION / PROCEDURE 两分支各一份**（`dd_routine.cc:414-418` 与 `426-430`） |
| `SP_DEFAULT_ACCESS_MAPPING` 是枚举/常量 | 是 **`#define` 宏**（`sql/sp.h:68/71`），映射到 `SP_CONTAINS_SQL` / `SP_IS_SUID` |
| `dd_trigger.cc` 有 `drop_trigger` / `update_trigger` | **都不在 `dd_trigger.cc`**。`drop_trigger` 是 `dd::Table` 的虚函数（实现 `table_impl.cc:1015`）；`update_trigger` **整个源码树无同名函数** |
| `copy_triggers` / `drop_all_triggers` 在 `dd_trigger.cc` | **不在** —— 是 `dd::Table` 接口，实现在 `table_impl.cc:962 / :1011` |
| 第 4 个转换函数叫 `convert_action_timing_from_dd` | **真名 `convert_action_time_from_dd`**（`time` 不是 `timing`） |
| 4 个转换函数都在头文件导出 | `dd_trigger.h` **只导出 3 个**：`create_trigger` / `load_triggers` / `table_has_triggers`。两个 `get_dd_*` 与两个 `convert_*_from_dd` 全是 `static` |
| `fill_in_dd_trigger_object` 三条注释各不相同 | **一字不差地相同**（都写 "resolve_charset will not..."），但第 2、3 条实际调的是 `resolve_collation` |
| `dd_event.cc` 有 `drop_trigger` 式的时区换算 | **没有任何时区换算** —— 全在 `Event_parse_data::init_*` |
| `set_event_attributes` 的 `time_zone` 写入条件对称 | **不对称**：`expression` 分支有 `if (!is_update \|\| !starts_null)` 守卫，`execute_at` 分支**无条件**写 |
| `check_if_in_the_past()` 是 public | 是 **private** |
| `events.time_zone` 有索引/外键 | **都没有** |
| `failsafe_object` 是对象名 | 只是**字符串常量 `"Event status option"`**，被 6 个转换函数的死代码分支共用 |
| `get_old_status` 等 3 个是 static | **非 static 且导出**（`dd_event.h`），供 I_S / upgrade 复用；反向的 3 个才是 `static` |
| `dd_view.cc` 有 `get_view_definition()` | **不存在** → 读取侧真名 **`read_view()`** |
| 视图转换函数叫 `dd_get_view_check_type()` | **真名带 `old_`/`new_` 前缀**，共 6 个：`dd_get_old_view_check_type` / `dd_get_new_view_check_type` / ... |
| `create_view()` / `update_view()` 自己提交事务 | **不提交**（头文件注释要求调用方 rollback）。只有 `update_view_status()` 在 `commit_dd_changes==true` 时自带提交 |
| `create_view()` 也调 `remove_children()` | **只有 `update_view()` 调** |
| `dd::Resource_group::enum_resource_group_type` | **不存在** → 真名 **`resourcegroups::Type`**（`enum class`，定义在 `sql/resourcegroups/` **不在 dd 命名空间**） |
| `resource_groups` 表有 `schema_id` / 外键 | **都没有**，用 `Global_name_key` |
| `set_resource_group_attributes()` 会失败 | **永远 `return false`**（两个调用点的 `if(...) return true` 是死码） |
| `view_table_usage` 用 `name_collation()` | 用 **`fs_name_collation()`**（随 `lower_case_table_names` 变）。而 `view_routine_usage.routine_name` 是**硬编码** `utf8mb3_general_ci` —— 三张表三种规则 |
| `KEY(type, view_definer)` 无特殊语义 | `Tables::create_key_by_definer()` 把前缀 **`type` 钉死为 `USER_VIEW`**，有意排除系统视图 |
| `update_referencing_views_metadata()` 总是被调 | **仅当 `sp->m_type == FUNCTION`**（CREATE 路径 `sp.cc:867`）；DROP 路径仅当 `mdl_type == MDL_key::FUNCTION`（`sp.cc:972`） |
| `dd_resource_group.h` 注释的表名 | 写的是 **`resourcegroup`（单数）**，真实表名是 `resource_groups`（复数）——注释与源码不一致 |
| `m_body_utf8` 是 `sp_head` 自己转码的 | **由 `Lex_input_stream` 在解析期边解析边累积**，`sp_head::set_body_end()` 只是 `strmake` 拷出来 |
| `m_body_utf8` 是 utf8mb4 | **是 utf8mb3**（`my_charset_utf8mb3_*`）。为 utf8mb4 预留的是 `parameter_type_elements.name` 的 `VARBINARY(1020)`（255×4） |

**★ 三个"结构性事实"**

1. **`mysql.routines` 的 `UNIQUE KEY(schema_id, type, name)` 带 `type`，所以同名的 FUNCTION 与 PROCEDURE 可以共存**——而 `type` **不是 `fill_dd_routine_info()` 填的**（它完全没有 `set_type()`），而是由**构造哪种子类**决定：`function_impl.cc` 构造器末尾 `set_type(RT_FUNCTION)`，`routine_impl.cc` 默认构造 `m_routine_type(RT_PROCEDURE)`。

2. **"视图依赖是名字级"有物理证据**：`view_table_usage` 的 `table_catalog`/`table_schema`/`table_name` 三列**没有任何外键指向任何表**（只有 `view_id` 有 FK 指向 `tables(id)`）。所以改基表时靠 `KEY(table_catalog, table_schema, table_name)` 反向索引做**字符串匹配**去反查视图，而不是靠 ID 级联——这正是底层表改名/重建后依赖会"找错"、需要 `view_valid` 兜底的根因。

3. **五类对象的"级联"都在对象层的 `drop_children()` 递归里，源码中未出现 `ON DELETE CASCADE`**——`Table_impl::drop_children()` 的顺序是 `check_constraints → triggers → partitions → foreign_keys → indexes → columns`，注释明确说明 **partition 必须在 index 之前**（因为 partition 对 index 有外键）。触发器走的批量删除键是 `Triggers::create_key_by_table_id()`，它用的是那条 `UNIQUE KEY(table_id, event_type, action_timing, action_order)`。

**★ 两个隐藏的 `status_changed` 置位点**（`dd_event.cc:286` 的条件更新①只在它们置位后才真正落库）：

| 置位点 | 触发条件 | 效果 |
|---|---|---|
| `Event_parse_data::check_originator_id()` | 从库线程执行 CREATE/ALTER EVENT | `ENABLED`/`DISABLED` → **`SLAVESIDE_DISABLED`** |
| `Event_parse_data::check_if_in_the_past()` | `on_completion != DROP` 且状态是 `ENABLED` | `ENABLED` → **`DISABLED`** |

> 前者是"**事件不会在从库被执行两次**"的 DD 层保障；后者是"**过去时间 + PRESERVE 的事件被自动禁用**"。两者都通过 `status_changed` 这一个布尔量打通到 `set_event_attributes()`。
