# ★ CREATE TABLE 深度剖析（server ↔ InnoDB 跨层全链路）

> 基于 MySQL 8.0.39 源码，涵盖 server 侧编排（`Sql_cmd_create_table::execute` → `mysql_create_table` → `mysql_prepare_create_table` → `rea_create_base_table` → `ha_create_table`）与**引擎侧落地**（`ha_innobase::create` → `create_table_info_t` → `row_create_table_for_mysql` → `dict_build_tablespace_for_table` / `dict_create_index_tree_in_mem` → `btr_create`）。
>
> **边界**：本篇只讲 CREATE TABLE 这一条操作链路。
> - 跨操作共享的骨架（三种算法判定、MDL 三段式、DDL 观测）见 [`ddl.md`](ddl.md)
> - 原子 DDL（DDL log 与崩溃恢复闭环）见 [`atomic_ddl.md`](atomic_ddl.md)——CREATE 的建树动作会写 `FREE_TREE_LOG`，其延迟 replay 语义在那边
> - ALTER / DROP / TRUNCATE 见 [`alter_table.md`](alter_table.md) 与 [`ddl.md`](ddl.md)
>
> 本篇的 `rea_create_base_table` / `ha_create_table` 是 CREATE 与 ALTER 共用的接缝，放在这里是因为它们是 CREATE 主链路的一段；ALTER 复用时链接回来。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [主链路：两层视角一张图](#主链路两层视角一张图)
- [建表是被复用的底座：三种算法怎么用 CREATE TABLE 的实现](#建表作为复用底座三种算法对-create-table-实现的复用)
- [server 侧编排](#server-侧编排)
  - [完整调用链](#完整调用链)
  - [`mysql_create_table()`：server 侧的"总指挥"](#mysql_create_tableserver-侧的总指挥)
  - [`mysql_create_table_no_lock()`：薄封装](#mysql_create_table_no_lock薄封装)
  - [`create_table_impl()`：真正的落地方](#create_table_impl真正的落地方)
  - [★ `mysql_prepare_create_table()`：全部"表定义合法性"](#-mysql_prepare_create_table全部表定义合法性)
  - [★ `rea_create_base_table()`：DD 写入 + 调引擎的接缝](#-rea_create_base_tabledd-写入--调引擎的接缝)
  - [`ha_create_table()`：server 与 handler 的合同](#ha_create_tableserver-与-handler-的合同)
- [引擎侧落地](#引擎侧落地)
  - [`ha_innobase::create` → `create_impl`：参数解析与三段式](#ha_innobasecreate--create_impl参数解析与三段式)
  - [`create_table_info_t::create_table`：建表 + 逐个建索引](#create_table_info_tcreate_table建表--逐个建索引)
  - [表空间的创建：`dict_build_tablespace_for_table`](#表空间的创建dict_build_tablespace_for_table)
  - [★ 索引树的创建：`btr_create` 建根页](#-索引树的创建btr_create-建根页)
  - [统计信息初始化：为什么新表统计必然是空的](#统计信息初始化新表统计为空的成因)
  - [引擎侧收尾：`create_table_update_global_dd` / `update_dict`](#引擎侧收尾create_table_update_global_dd--update_dict)
- [变体：CREATE ... LIKE / SELECT](#变体create--like--select)
- [变体：临时表 / DD 系统表 / 内部临时表](#变体临时表--dd-系统表--内部临时表)
- [可观测性](#可观测性)
- [Misc](#misc)

---

## 概述

### 是什么

CREATE TABLE 是**从无到有建对象**的操作：server 侧在 DD 里落一行 `dd::Table` 元数据，引擎侧建出表空间（file-per-table 时是新的 `.ibd`）与**每棵索引树的根页**。它是唯一不涉及"搬旧数据"的 DDL——这决定了它不需要、也不存在"算法选择"（对比 ALTER 的 COPY/INPLACE/INSTANT）。

### 用途

除了作为用户的建表语句，它还是**被三种算法/操作复用的底座**（COPY ALTER 建中间表、INPLACE rebuild 建新表、TRUNCATE 重建空表——详见下一节）。

### 版本演进

| 版本 | 变化 |
|------|------|
| 5.7 | 元数据在 `.frm` + 非事务系统表；建表失败可能留下孤儿 `.ibd` 或 DD 与引擎不一致 |
| 8.0 | **新数据字典 + 原子 DDL**：CREATE TABLE 纳入 Prepare / Perform / Commit / Post-DDL 四阶段，Prepare 阶段写"回滚预留"的 DDL log（[WL#9536](https://dev.mysql.com/worklog/task/?id=9536) §1）；GIPK（隐式主键）、SDI 等元数据特性随之加入 |
| 8.0（持续） | `create_table_info_t` 三段式（解析 / 校验 / 落地）定型，支撑分区表与普通表共用同一套实现（模板实例化） |

---

## 理论基础

### 设计思想与权衡

**权衡一：为什么不把"校验"做成声明式，而散落在四个函数里？**

因为**校验的粒度取决于引擎运行时能力**，不是常量。8.0.39 全部走 `handler` 的虚接口：

| 协商点 | 方法（不是常量） | 位置 | 错误码 |
|---|---|---|---|
| 索引个数 | `file->max_keys()`（InnoDB=64） | `sql/sql_table.cc:8156` | `ER_TOO_MANY_KEYS` |
| 索引列段数 | `file->max_key_parts()` | `:7197` | `ER_TOO_MANY_KEY_PARTS` |
| 单键段字节 | `file->max_key_part_length(create_info)` | `:5150 / 5211` | `ER_TOO_LONG_KEY` |
| 整键字节 | `file->max_key_length()` | `:5149 / 7400` | `ER_TOO_LONG_KEY` |
| 记录长度 | `file->max_record_length()` | `:8364` | `ER_TOO_BIG_ROWSIZE` |
| **列数** | **`MAX_FIELDS`（唯一硬编码）** | `:8051` | `ER_TOO_MANY_FIELDS` |

```cpp
  // 【注】★ 全函数唯一的硬编码常量校验
  if (alter_info->create_list.elements > MAX_FIELDS) {
    my_error(ER_TOO_MANY_FIELDS, MYF(0));
    return true;
  }
  ...
  if (*key_count > file->max_keys()) {          // 【注】索引数是运行时的
    my_error(ER_TOO_MANY_KEYS, MYF(0), file->max_keys());
    return true;
  }
  ...
  if (reclength > file->max_record_length()) {  // 【注】行大小也是运行时的
    my_error(ER_TOO_BIG_ROWSIZE, MYF(0),
             static_cast<long>(file->max_record_length()));
    return true;
  }
```

> ⚠️ **勘误**：**不存在 `MAX_KEY` / `MAX_KEY_LENGTH` 常量判据**（那是 5.x 的 .FRM 遗产）；**不存在 `file->max_supported_key_length()`**。
> ⚠️ **表名/库名长度校验不在 `mysql_prepare_create_table()`**——真身在 `sql/table.cc:3678 check_table_name()`。该函数里的 `ER_TOO_LONG_IDENT` 只服务**索引名**（`prepare_key()` :7205）与**外键名**（`prepare_foreign_key()` :6581）。
> ⚠️ **`ER_TOO_LONG_KEY` 在 3 个不同层级触发**（BLOB 前缀 :5169、普通键段 :5233、整键 :7402），且都写成 `if (thd->is_error()) return true;` —— 允许被 `Key_length_error_handler` / `Strict_error_handler` **吞掉**。DD/系统表建表时正是靠这个白名单机制容忍历史遗留的长键。

**权衡二：为什么 `Create_field → Field` 不在 server 的 prepare 阶段？**

`mysql_prepare_create_table()` **只产出 `KEY[]` + `KEY_PART_INFO[]` + `FOREIGN_KEY[]`**，不造 `Field**`。转换发生在 **DD 侧**：

```cpp
// sql/sql_table.cc:1084 —— rea_create_base_table() 第一行
  std::unique_ptr<dd::Table> table_def_res = dd::create_table(
      thd, sch_obj, table_name, create_info, create_fields, key_info, keys,
      keys_onoff, fk_key_info, fk_keys, check_cons_spec, file);
```

```cpp
// sql/dd/dd_table.cc:2163 —— fill_dd_table_from_create_info() 内
  // Add field definitions
  if (fill_dd_columns_from_create_fields(thd, tab_obj, create_fields, file))
    return true;                       // ★ Create_field → dd::Column（内部 make_field()）
  ...
  // Add index definitions
  fill_dd_indexes_from_keyinfo(thd, tab_obj, keys, keyinfo, create_fields, file);
  ...
  // Only add foreign key definitions for engines that support it.
  if (ha_check_storage_engine_flag(create_info->db_type,
                                   HTON_SUPPORTS_FOREIGN_KEYS)) {
    if (fill_dd_foreign_keys_from_create_fields(tab_obj, fk_keys, fk_keyinfo))
      return true;
  }
```

> ★★ **意义**：`Field` 对象只在**建表成功之后**由 `open_table_from_share()` 从 `dd::Column` 反建。所以**"server 层的表定义"权威副本是 `dd::Table`，不是 `TABLE`**。`TABLE`/`Field**` 只是运行时投影。

**权衡三：为什么"引擎不支持外键"不报错而是静默丢弃？**

```cpp
  bool se_supports_fks =
      (create_info->db_type->flags & HTON_SUPPORTS_FOREIGN_KEYS);
  ...
    if (key->type == KEYTYPE_FOREIGN) {
      if (prepare_foreign_key(thd, create_info, alter_info, error_schema_name,
                              error_table_name, is_partitioned,
                              *key_info_buffer, *key_count, *fk_key_info_buffer,
                              fk_number, se_supports_fks, find_parent_keys,
                              down_cast<Foreign_key_spec *>(key),
                              &fk_max_generated_name_number, fk_key_info))
        return true;

      // ★★ SE 不支持 FK 时，fk_key_info / fk_number 都不递增
      if (se_supports_fks) {
        fk_key_info++;
        fk_number++;
      }
    }
```

> ★ 后果：`CREATE TABLE t (..., FOREIGN KEY ...) ENGINE=MyISAM` **不报错**，外键被静默忽略（这也是经典面试坑）。DD 侧在 `fill_dd_table_from_create_info()` 里**再判一次** `HTON_SUPPORTS_FOREIGN_KEYS`——两处守卫。

**权衡四：`sql_require_primary_key` 的落点**

```cpp
  if (!primary_key && !thd->is_dd_system_thread() &&
      !thd->is_initialize_system_thread() &&
      (file->ha_table_flags() &
       (HA_BINLOG_ROW_CAPABLE | HA_BINLOG_STMT_CAPABLE)) != 0 &&
      thd->variables.sql_require_primary_key) {
    my_error(ER_TABLE_WITHOUT_PK, MYF(0));
    return true;
  }
```

> ★ 三个豁免：DD 系统线程、初始化线程、**不支持 binlog 的引擎**（例如 MEMORY 在某些配置下）。判据是 `HA_BINLOG_ROW_CAPABLE | HA_BINLOG_STMT_CAPABLE`，不是"是不是 InnoDB"。

**权衡五：KEY 数组为什么要排序？**

```cpp
  /* Sort keys in optimized order */
  std::sort(*key_info_buffer, *key_info_buffer + *key_count, sort_keys());
```

> ★ `sort_keys()` 把 PK / UNIQUE / 前缀短的排前面。**这决定了 `dd::Index` 的 `ordinal_position`**，也决定了 InnoDB 侧建索引的先后顺序（聚簇必须第一个）。排序**发生在** `check_duplicate_key()` 之后、`check_promoted_index()` 之前。

**要解决的问题**：一条 CREATE 要**同时在两层创建对象**（DD 里一行元数据 + 引擎里一个表空间和若干棵 B-tree），任何一层留下残留在失败时都是垃圾。

**为什么 `rea_create_base_table` 里"先写 DD、再调引擎"**：引擎要用 DD 已经分配好的 `table_id` / `index_id` 与规范化后的表定义；反过来先建引擎对象就没有 id 可用。而且 DD 写入是同一事务内的 DML，可随事务回滚——把"可回滚的"放在前面、"不可回滚的物理操作"放在后面（且为其写回滚预案），是原子 DDL 的一贯顺序（见 [`atomic_ddl.md`](atomic_ddl.md)）。

**为什么引擎侧要分三段（initialize 解析 → prepare_create_table 校验 → create_table 落地）**：解析与校验分开，是为了让非法选项组合在**进入物理建表之前**被拒绝——一旦 `create_table()` 开始建 `.ibd` 和 B-tree，就不再是纯内存操作了。

**为什么原子 DDL 下 CREATE 也要写 DDL log**：Prepare 阶段为刚建的 `.ibd` 写 `DELETE_SPACE_LOG`、为每棵树写 `FREE_TREE_LOG`、为字典缓存项写 `REMOVE_CACHE_LOG`，随后**主事务立即把它们删除**（源码注释：*"Delete this operation if the create trx is committed"*）。于是：提交成功 ⇒ 删除生效 ⇒ log 表干净、什么都不用做；失败或崩溃 ⇒ 删除未生效 ⇒ replay 清理刚建的对象。**这些 log 是回滚预案，不是执行计划**——方向与 DROP 类相反，完整对照见 [`atomic_ddl.md`](atomic_ddl.md)。

**代价与约束**：

- CREATE 是 **O(1)**（不搬数据），与 ALTER 的 O(n) 有本质区别——这正是它不需要算法选择的原因，也是它被 ALTER 复用的价值（COPY/INPLACE 都要先"建一张新表"）。
- 建表虽快但涉及文件系统操作（建 `.ibd`）与全局 id 分配，仍需 DDL log 兜底。
- **简化假设的失效场景**：目标引擎不支持 `HTON_SUPPORTS_ATOMIC_DDL` 时，CREATE 失败要由 server 的 post-DDL 钩子自己清理残留文件（源码注释明确这是该钩子在 CREATE 场景的主要职责）。

### 算法与数据结构

| 机制 | 落点 | 为什么这么选 |
|---|---|---|
| `create_table_info_t` 三段式 | initialize（解析）→ prepare_create_table（校验）→ create_table（落地） | 把 server 的 `HA_CREATE_INFO` 翻译成 InnoDB flags 位图；三段分离让校验失败不产生任何副作用 |
| 模板实例化 | `create_impl<dd::Table>` / `create_impl<dd::Partition>` | 分区表与普通表元数据类型不同但建表逻辑相同，用模板消除重复 |
| 每索引两个文件段 | `btr_create` 建根页时同时建 LEAF / TOP 段 | B-tree 的叶子层与非叶层独立分配，便于后续按 extent 粒度管理与释放（见 [`ddl.md`](ddl.md) 的 btr_free 章） |
| 表空间形态 | `innodb_file_per_table` 决定建独立 `.ibd` 还是用共享表空间 | 影响后续 DROP 的清理方式（整文件删除 vs 逐页释放），也影响 CREATE 时是否写 `DELETE_SPACE_LOG` |

---

## 主链路：两层视角一张图

CREATE TABLE 是 DDL 里**唯一一个"两层都在建新对象"**的操作——server 侧要在 DD 里落一行 `dd::Table`，引擎侧要真的建出表空间文件和 B-tree 根页。两层之间的接缝是 `ha_create_table()`：

```
server 侧                                     引擎侧（InnoDB）
─────────────────────────────────────────   ────────────────────────────────────────
Sql_cmd_create_table::execute
  → mysql_create_table           选引擎/拿 MDL/收集 FK
    → mysql_create_table_no_lock  薄封装
      → create_table_impl
        ├─ mysql_prepare_create_table   ★ 全部合法性检查（纯 server 侧）
        └─ rea_create_base_table
             ├─ dd::create_table → store()     写 DD 元数据
             └─ ha_create_table  ─────────────→ ha_innobase::create
                  （server 与 handler 的合同）      → innobase_basic_ddl::create_impl
                                                     → create_table_info_t 三段式：
                                                          initialize()
                                                          prepare_create_table()
                                                          create_table()
                                                            → create_table_def()      建 dict_table_t
                                                                → row_create_table_for_mysql
                                                                    → dict_build_table_def
                                                                        → dict_build_tablespace_for_table  ★ 建 .ibd
                                                            → create_index() × N       逐个建索引
                                                                → row_create_index_for_mysql
                                                                    → dict_create_index_tree_in_mem
                                                                        → btr_create     ★ 建根页
                                                     → create_table_update_global_dd()
                                                     → create_table_update_dict()      ★ 统计置空
```

一句话抓住两层的分工差异：**server 侧负责"这张表叫什么、合不合法、DD 里怎么写"；引擎侧负责"数据放哪（表空间）和怎么组织（B-tree）"**。`rea_create_base_table` 里两件事的顺序是先写 DD 再调引擎——因为引擎要用 DD 里已经分配好的 id 与元数据（详见该节）。

---

## 建表作为复用底座：三种算法对 CREATE TABLE 实现的复用

⚠️ **勘误**：`create_table_impl()`（8725–9030）里**只有 `HA_LEX_CREATE_TMP_TABLE` 一个真正的分支**。**`HA_LEX_CREATE_TABLE_LIKE` 和 `HA_LEX_CREATE_FROM_SELECT` 在 `create_table_impl()` 里完全不存在**，它们在更上游的 `Sql_cmd_create_table::execute()`（`sql/sql_cmd_ddl_table.cc:129`）就分流了。

**三条路径的分流点**：

```
Sql_cmd_create_table::execute()                    sql/sql_cmd_ddl_table.cc:129
  ├─ if (!query_block->field_list_is_empty())      ← ★ 用"SELECT 列表是否为空"判定 CTAS，
  │     → Query_result_create + populate_table()      不是检查 HA_LEX_CREATE_FROM_SELECT 位
  └─ else if (create_info.options & HA_LEX_CREATE_TABLE_LIKE)
  │     → mysql_create_like_table()                sql_table.cc:10846
  └─ else
        → mysql_create_table()                     sql_table.cc:10045
              └─ mysql_create_table_no_lock()      :9135
                    └─ create_table_impl()         :8725
```

```cpp
    /* regular create */
    if (create_info.options & HA_LEX_CREATE_TABLE_LIKE) {
      /* CREATE TABLE ... LIKE ... */
      res = mysql_create_like_table(thd, create_table, query_expression_tables,
                                    &create_info);
    } else {
      /* Regular CREATE TABLE */
      res = mysql_create_table(thd, create_table, &create_info, &alter_info);
    }
```

**`create_table_impl()` 里的 `HA_LEX_CREATE_TMP_TABLE` 分支**（`sql_table.cc:8996`）：

```cpp
  /*
    Create table definitions.
    If "no_ha_table" is false also create table in storage engine.
  */
  if (create_info->options & HA_LEX_CREATE_TMP_TABLE) {
    if (rea_create_tmp_table(thd, path, schema, db, table_name, create_info,
                             alter_info->create_list, *key_count, *key_info,
                             keys_onoff,
                             &alter_info->check_constraint_spec_list,
                             file.get(), no_ha_table, is_trans, table_def))
      return true;
  } else {
    if (rea_create_base_table(thd, path, schema, db, table_name, create_info,
                              alter_info->create_list, *key_count, *key_info,
                              keys_onoff, *fk_key_count, *fk_key_info,
                              &alter_info->check_constraint_spec_list,
                              file.get(), no_ha_table, do_not_store_in_dd,
                              part_info, is_trans, table_def, post_ddl_ht))
      return true;
  }
  THD_STAGE_INFO(thd, stage_after_create);
```

**`HA_LEX_CREATE_TMP_TABLE` 在同一函数里的全部判定点**：

| 行号 | 语义 |
|---|---|
| 8755 | 临时表不允许 `SECONDARY_ENGINE` → `ER_SECONDARY_ENGINE` |
| 8866 | 临时表不允许分区 → `ER_PARTITION_NO_TEMPORARY` |
| 8913 | 传给 `check_if_table_exists()` |
| 8925 | `is_whitelisted_table` 计算时排除临时表 |
| 9019 | 建完后若在多语句事务中 → 置 `SERVER_STATUS_IN_TRANS` |

**`HA_LEX_CREATE_IF_NOT_EXISTS` 的三态返回**（`sql/sql_table.cc:8556`）——这是"复用底座"最精妙的一处：

```cpp
struct Table_exists_result {
  /** true if the table already exists */
  bool m_table_exists;
  /** true if my_error() has been called and an error must be propagated. */
  bool m_error;
};
```

| 场景 | `{m_table_exists, m_error}` | 结果 |
|---|---|---|
| 不存在、无错 | `{false, false}` | 继续建 |
| 存在 + `IF NOT EXISTS` | `{true, false}` | **只 push_warning，返回"成功"** |
| 存在 + 无 `IF NOT EXISTS` | `{true, true}` | `ER_TABLE_EXISTS_ERROR` 已报出 |

```cpp
  Table_exists_result ter = check_if_table_exists(
      thd, db, table_name, alias,
      (create_info->options & HA_LEX_CREATE_TMP_TABLE),
      (create_info->options & HA_LEX_CREATE_IF_NOT_EXISTS), internal_tmp_table);
  if (ter.m_error) {
    return true;        // ★ 真失败
  }
  if (ter.m_table_exists) {
    return false;       // ★★ IF NOT EXISTS 的"成功"：此处返回 false，SQL 层面报 Note 1050
  }
```

**⚠️ `check_if_table_exists` 有两个同名不同物**：

| 版本 | 位置 | 签名 | 用在哪 |
|---|---|---|---|
| 三返回版 | `sql/sql_base.cc:2483` | `static bool(THD*, Table_ref*, bool *exists)` | 一般开表 |
| **`Table_exists_result` 版** | **`sql/sql_table.cc:8556`** | 7 参数 | **建表路径** |

```cpp
static bool check_if_table_exists(THD *thd, Table_ref *table, bool *exists) {
  *exists = true;      // ★ 默认"存在"，先乐观地认为有；只有明确查不到才置 false
  assert(thd->mdl_context.owns_equal_or_stronger_lock(
      MDL_key::TABLE, table->db, table->table_name, MDL_SHARED));
  if (dd::table_exists(thd->dd_client(), table->db, table->table_name, exists))
    return true;       // 语义①：出错
  if (*exists) goto end;
  /* Table doesn't exist. Check if some engine can provide it. */
  if (ha_check_if_table_exists(thd, table->db, table->table_name, exists)) {
    my_printf_error(ER_OUT_OF_RESOURCES, "Failed to open '%-.64s', error while "
                    "unpacking from engine", MYF(0), table->table_name);
    return true;       // 语义①：SE 侧解包失败
  }
end:
  return false;
}
```

> ★ 建表版还多一条 **分支 C**：DD 里查不到时还要 `ha_table_exists_in_engine()` 问 SE 一句（engine discovery，NDB 典型场景：其他节点建了表，本节点 DD 还没同步）。
> ★ **DD 系统表名豁免**：`is_dd_table_name()` 为真时，分支 B 和 C **都跳过**（引导时 DD 本身还不可用）。

**CREATE TABLE 不属于任何一种 DDL 算法**——`ALGORITHM` 子句是 `ALTER TABLE` 的语法（语法规则 `alter_algorithm_option` → `ALTER_TABLE_ALGORITHM_*`），CREATE TABLE 不支持它，也不需要：它是从无到有建对象，没有"搬旧数据"的负担，自然没有"选哪种搬法"的问题。三种算法的完整对照见 [`copy_ddl.md`](copy_ddl.md)、[`inplace_ddl.md`](inplace_ddl.md)、[`instant_ddl.md`](instant_ddl.md)。

但反过来——**建表这套实现被三种算法/操作共用**，这是本篇值得独立成篇的第二个理由：

| 复用方 | 复用到哪一层 | 落点 |
|---|---|---|
| **COPY ALTER** | server 层建中间表 `#sql-xxx` | `create_table_impl`（ALTER 路径内调用，带 `no_ha_table`——只建定义、不落地引擎对象），随后 `rea_create_base_table` → `ha_create_table` 与 CREATE 共用同一接缝 |
| **INPLACE rebuild** | 引擎内建 `ctx->new_table` | `row_create_table_for_mysql(ctx->new_table, ...)`——**与 CREATE 引擎侧落地是同一个函数**；本篇「索引树的创建」讲的 `btr_create` 建根页同样发生在这里 |
| **TRUNCATE**（recreate 路径） | rename + drop 后重建空表 | `innobase_basic_ddl::create_impl`——**与 CREATE 引擎侧入口是同一个函数** |

读本篇时记住两条：① `rea_create_base_table` / `ha_create_table` 之所以叫"CREATE 与 ALTER 的共用接缝"，是因为 COPY ALTER 真的走它；② 引擎侧 `row_create_table_for_mysql` 建的表不只来自 CREATE——INPLACE rebuild 在引擎内建的新表也走它，只是那张新表随后由 [`ddl_engine.md`](ddl_engine.md) 讲的 Loader/Builder 填充数据。

---

## server 侧编排



### 完整调用链

```
Sql_cmd_create_table::execute()            sql/sql_cmd_ddl_table.cc
  ├─(SELECT 分支) populate_table() → Query_result_create → mysql_create_table()
  ├─(LIKE 分支)   mysql_create_like_table()               sql/sql_table.cc
  └─(常规分支)    mysql_create_table()                    sql/sql_table.cc
                    ├─ get_viable_handlerton_for_create()
                    ├─ open_tables()  → 拿/升级 MDL-X
                    ├─ collect_fk_parents_for_new_fks() / collect_fk_children() / collect_fk_names_for_new_fks()
                    ├─ prepare_check_constraints_for_create()
                    ├─ promote_first_timestamp_column()
                    ├─ validate_and_generate_invisible_primary_key()   (GIPK)
                    ├─ mysql_create_table_no_lock()      ← 薄封装
                    │    ├─ build_table_filename() / build_tmptable_filename()
                    │    ├─ dd::Schema_MDL_locker + dd_client()->acquire(schema)
                    │    ├─ validate_table_encryption()
                    │    └─ create_table_impl()          ← ★真正干活
                    │         ├─ 至少一个可见列检查 / check_engine / set_table_default_charset
                    │         ├─ get_new_handler()（分区时可能是 partition handler）
                    │         ├─ part_info->check_partition_info()
                    │         ├─ check_if_table_exists()   ← 命名冲突
                    │         ├─ mysql_prepare_create_table()   ← ★★ 全部合法性检查
                    │         └─ rea_create_tmp_table() | rea_create_base_table()
                    │                └─ dd::create_table() → dd_client()->store()
                    │                └─ ha_create_table() → table.file->ha_create()
                    ├─ write_bin_log()
                    ├─ adjust_fk_children_after_parent_def_change() / adjust_fk_parents()
                    ├─ update_referencing_views_metadata()
                    └─ trans_commit_stmt() + trans_commit_implicit() + post_ddl_ht->post_ddl()
```

### `mysql_create_table()`：server 侧的"总指挥"

**① 选引擎 + 拒绝非法组合**

```cpp
handlerton *actual_hton = get_viable_handlerton_for_create(
    thd, create_table->table_name, *create_info);
if (actual_hton == nullptr) return true;
create_info->db_type = actual_hton;
```

★ **引擎默认值在 `execute()` 里才定**（不是 parse 时）：

```cpp
if (!(create_info.used_fields & HA_CREATE_USED_ENGINE))
  create_info.db_type = create_info.options & HA_LEX_CREATE_TMP_TABLE
                            ? ha_default_temp_handlerton(thd)
                            : ha_default_handlerton(thd);
```

**② 拿 MDL**（S → X 升级）

```cpp
/*
  Open or obtain "X" MDL lock on the table being created.
  To check the existence of table, lock of type "S" is obtained on the table
  and then it is upgraded to "X" if table does not exists.
*/
```

**③ 外键 MDL 收集**（只有表不存在且非临时表且引擎支持 FK 时才做）

```cpp
/*
  Do not acquire metadata locks on tables in FK relationships if
  table (or view with the same name) exists. They are not necessary
  as we won't perform any lookups on them or update of their metadata
  in this case.
*/
```

**④ 约束/列预处理** → `prepare_check_constraints_for_create()` → `promote_first_timestamp_column()`（`explicit_defaults_for_timestamp=OFF` 时把第一个 TIMESTAMP 列提升）→ **GIPK**：

```cpp
if (is_generate_invisible_primary_key_mode_active(thd) && ...) {
  if (validate_and_generate_invisible_primary_key(thd, alter_info)) { ... }
  is_pk_generated = true;
}
```

**⑤ 写 binlog**——★ **GIPK 要"重写语句"**：

```cpp
if ((create_table->table == nullptr && !create_table->is_view()) && is_pk_generated) {
  ... store_create_info(thd, create_table, &query, create_info, true, false);
  result = write_bin_log(thd, true, query.ptr(), query.length(), is_trans);
} else {
  result = write_bin_log(thd, true, thd->query().str, thd->query().length, is_trans);
}
```

原因（注释原文）：*"we don't write to binary log value of @@sql_generate_invisible_primary_key variable, but rely on logging what really has been done instead."*

**⑥ 提交或回滚 + post-DDL 钩子**

```cpp
if (!result && !thd->is_plugin_fake_ddl())
  result = trans_commit_stmt(thd) ||
           (create_info->m_transactional_ddl ? false : trans_commit_implicit(thd));

if (result && !thd->is_plugin_fake_ddl()) {
  trans_rollback_stmt(thd);
  /*
    Full rollback in case we have THD::transaction_rollback_request
    and to synchronize DD state in cache and on disk (as statement
    rollback doesn't clear DD cache of modified uncommitted objects).
  */
  trans_rollback(thd);
}

/*
  In case of CREATE TABLE post-DDL hook is mostly relevant for case
  when statement is rolled back. In such cases it is responsibility
  of this hook to cleanup files which might be left after failed
  table creation attempt.
*/
if (!create_info->m_transactional_ddl && post_ddl_ht)
  post_ddl_ht->post_ddl(thd);
```

⇒ **为什么要 `trans_rollback_stmt()` 之后再来一次 `trans_rollback()`**：注释说得很清楚——语句级回滚**不会清掉 DD 缓存里已修改但未提交的对象**，必须完整回滚才能让缓存与磁盘同步。

### `mysql_create_table_no_lock()`：薄封装

**全文**（`sql/sql_table.cc:9131`）——**注意它自己不做建表，只做 5 件事：拼路径 → 取 schema → 加密校验 → 弃用告警 → 委托**：

```cpp
/**
  Simple wrapper around create_table_impl() to be used
  in various version of CREATE TABLE statement.
*/
bool mysql_create_table_no_lock(THD *thd, const char *db,
                                const char *table_name,
                                HA_CREATE_INFO *create_info,
                                Alter_info *alter_info, uint select_field_count,
                                bool find_parent_keys, bool *is_trans,
                                handlerton **post_ddl_ht) {
  KEY *not_used_1;
  uint not_used_2;
  FOREIGN_KEY *not_used_3 = nullptr;
  uint not_used_4 = 0;
  std::unique_ptr<dd::Table> not_used_5;   // ★★ 五个 not_used_ 出参：纯 CREATE 不需要回传
  char path[FN_REFLEN + 1];

  if (create_info->options & HA_LEX_CREATE_TMP_TABLE)
    build_tmptable_filename(thd, path, sizeof(path));
  else {
    bool was_truncated;
    const char *alias = table_case_name(create_info, table_name);
    build_table_filename(path, sizeof(path) - 1 - reg_ext_length, db, alias, "",
                         0, &was_truncated);
    if (was_truncated) {
      my_error(ER_IDENT_CAUSES_TOO_LONG_PATH, MYF(0), sizeof(path) - 1, path);
      return true;
    }
  }

  bool no_ha_table = false;
  if (!opt_initialize && dd::get_dictionary()->is_dd_table_name(db, table_name))
    no_ha_table = true;   // ★ 升级/初始化场景：只造 DD 对象，不在 SE 里真建表

  dd::Schema_MDL_locker mdl_locker(thd);
  dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
  const dd::Schema *schema = nullptr;
  if (mdl_locker.ensure_locked(db) || thd->dd_client()->acquire(db, &schema)) {
    return true;          // Error is reported by the dictionary subsystem.
  }
  if (schema == nullptr) {
    my_error(ER_BAD_DB_ERROR, MYF(0), db);
    return true;
  }

  if (create_info->options & HA_LEX_CREATE_TMP_TABLE) {
    if (create_info->encrypt_type.length) {
      my_error(ER_CANNOT_USE_ENCRYPTION_CLAUSE, MYF(0), "temporary");
      return true;        // ★ 临时表禁止 ENCRYPTION
    }
    if (create_info->m_implicit_tablespace_autoextend_size > 0) {
      my_error(ER_CANNOT_USE_AUTOEXTEND_SIZE_CLAUSE, MYF(0), "temporary");
      return true;        // ★ 临时表禁止 AUTOEXTEND_SIZE
    }
  }

  if (!(create_info->options & HA_LEX_CREATE_TMP_TABLE)) {
    /* Assume table as encrypted, if user did not explicitly state it and
    we have a schema with default encryption enabled. */
    if (!create_info->encrypt_type.length && schema->default_encryption()) {
      create_info->encrypt_type = {strmake_root(thd->mem_root, "Y", 1), 1};
    }
    if (validate_table_encryption(thd, create_info)) return true;

    if (create_info->encrypt_type.str || create_info->tablespace) {
      bool request_type = dd::is_encrypted(create_info->encrypt_type);
      if (schema->default_encryption() != request_type) {
        if (opt_table_encryption_privilege_check) {
          if (check_table_encryption_admin_access(thd)) {
            my_error(ER_CANNOT_SET_TABLE_ENCRYPTION, MYF(0));
            return true;
          }
        } else if (schema->default_encryption() && !request_type) {
          push_warning(thd, Sql_condition::SL_WARNING,
                       WARN_UNENCRYPTED_TABLE_IN_ENCRYPTED_DB,
                       ER_THD(thd, WARN_UNENCRYPTED_TABLE_IN_ENCRYPTED_DB));
        }
      }
    }
  }

  for (const Create_field &sql_field : alter_info->create_list) {
    warn_on_deprecated_float_auto_increment(thd, sql_field);
  }
  if (!thd->lex->query_block->field_list_is_empty()) {   // ★ 只有 CTAS/LIKE 才报这三类
    for (const Create_field &sql_field : alter_info->create_list) {
      warn_on_deprecated_float_precision(thd, sql_field);
      warn_on_deprecated_float_unsigned(thd, sql_field);
      warn_on_deprecated_zerofill(thd, sql_field);
    }
  }

  if (thd->is_plugin_fake_ddl()) no_ha_table = true;

  return create_table_impl(
      thd, *schema, db, table_name, table_name, path, create_info, alter_info,
      false, select_field_count, find_parent_keys, no_ha_table, false, is_trans,
      &not_used_1, &not_used_2, Alter_info::ENABLE, &not_used_3, &not_used_4,
      nullptr, 0, nullptr, 0, &not_used_5, post_ddl_ht);
}
```

> ★★ **五个 `not_used_*` 出参是"复用"的痕迹**：`mysql_create_table_no_lock()` 同时服务 CREATE / CREATE LIKE / CTAS / ALTER 内部临时表，后三者需要回传 `KEY*`/`dd::Table*`，纯 CREATE 用不到但签名统一。

**`build_tmptable_filename()`**（`sql/sql_table.cc:911`，⚠️ 不在 `sql/table.cc`）：

```cpp
size_t build_tmptable_filename(THD *thd, char *buff, size_t bufflen) {
  char *p = my_stpnmov(buff, mysql_tmpdir, bufflen);
  assert(sizeof(my_thread_id) == 4);
  snprintf(p, bufflen - (p - buff), "/%s%lx_%x_%x", tmp_file_prefix,
           current_pid, thd->thread_id(), thd->tmp_table++);
  //  ★ tmp_file_prefix = "#sql"；格式 = #sql<pid十六进制>_<thread_id>_<自增计数器>

  if (lower_case_table_names) {
    /* Convert all except tmpdir to lower case */
    my_casedn_str(files_charset_info, p);   // ★ 只小写文件名部分，tmpdir 保持原样
  }
  size_t length = unpack_filename(buff, buff);
  return length;
}
```

> ⚠️ **`no_ha_table` 的语义不是"只建 .FRM"**——8.0 已无 FRM。`create_table_impl()` 的 doxygen 仍残留 "@param no_ha_table 只创建 .FRM 文件" 的**历史注释**，实际语义是"**只建 DD 定义、不建 SE 表**"。生效条件：①非 `--initialize` 且是 DD 表名；②`thd->is_plugin_fake_ddl()`。

**★ `create_table_impl()` 里 `no_ha_table` 的两条分支**（`sql_table.cc:1093`）：

```cpp
  if (no_ha_table) {
    if (part_info) { /* 分区表要显式 open 一次做完整性校验 ... */ }
    return false;    // ★★ no_ha_table=true 时到此为止：完全不碰 SE
  }
```

```cpp
  if (do_not_store_in_dd) {
    assert(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
    assert(no_ha_table);
    *table_def_ptr = std::move(table_def_res);
  } else {
    bool result = thd->dd_client()->store(table_def_res.get());
    //                        ★ 常规路径：先把 dd::Table store 进 DD（未提交）
    if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
        !thd->is_plugin_fake_ddl())
      result = trans_intermediate_ddl_commit(thd, result);
    //                        ★ 非原子 DDL 引擎：立即中间提交 DD
    if (result) return true;
    if (thd->dd_client()->acquire_for_modification(db, table_name, &table_def))
      return true;
  }
```

> ★★ **顺序很关键**：**先 `dd_client()->store()` 把 dd::Table 写进 DD（未提交），再 `ha_create_table()` 让引擎建表**，引擎建表时把 `se_private_id`/`se_private_data`/`tablespace_id` 回填到同一 `dd::Table`，最后 `ha_create_table()` 内部 `dd_client()->update()` 持久化——三步都在**同一个事务**里，这就是原子 DDL。

源码注释就一句：*"Simple wrapper around create_table_impl() to be used in various version of CREATE TABLE statement."*

它做的外壳工作：

- 拼路径（`build_tmptable_filename()` / `build_table_filename()`），检查 `was_truncated` → `ER_IDENT_CAUSES_TOO_LONG_PATH`
- **DD 表特殊处理**：

```cpp
/*
  Don't create the DD tables in the DDSE unless installing the DD.
*/
bool no_ha_table = false;
if (!opt_initialize && dd::get_dictionary()->is_dd_table_name(db, table_name))
  no_ha_table = true;
```

- 临时表拒绝 `ENCRYPTION` / `AUTOEXTEND_SIZE`；schema 默认加密继承 + `validate_table_encryption()`
- `thd->is_plugin_fake_ddl()` → `no_ha_table = true`
- 最后一行 `return create_table_impl(...)`

### `create_table_impl()`：真正的落地方

签名极具信息量（所有"变体开关"都在这里）：

```cpp
static bool create_table_impl(
    THD *thd, const dd::Schema &schema, const char *db, const char *table_name,
    const char *error_table_name, const char *path, HA_CREATE_INFO *create_info,
    Alter_info *alter_info, bool internal_tmp_table, uint select_field_count,
    bool find_parent_keys, bool no_ha_table, bool do_not_store_in_dd,
    bool *is_trans, KEY **key_info, uint *key_count,
    Alter_info::enum_enable_or_disable keys_onoff, FOREIGN_KEY **fk_key_info,
    uint *fk_key_count, FOREIGN_KEY *existing_fk_info, uint existing_fk_count,
    const dd::Table *existing_fk_table, uint fk_max_generated_name_number,
    std::unique_ptr<dd::Table> *table_def, handlerton **post_ddl_ht);
```

几个关键步骤：

**① 至少一个可见列**

```cpp
bool has_visible_column = false;
for (const Create_field &create_field : alter_info->create_list) {
  if (create_field.hidden == dd::Column::enum_hidden_type::HT_VISIBLE) { has_visible_column = true; break; }
}
if (!has_visible_column) { my_error(ER_TABLE_MUST_HAVE_A_VISIBLE_COLUMN, MYF(0)); return true; }
```

**② 默认字符集与 handler 获取**：`set_table_default_charset()` 之后取一个 handler——注意**它可能是分区 handler**：

```cpp
partition_info *part_info = thd->work_part_info;

std::unique_ptr<handler, Destroy_only<handler>> file(get_new_handler(
    (TABLE_SHARE *)nullptr,
    (part_info ||
     (create_info->db_type->partition_flags &&
      (create_info->db_type->partition_flags() & HA_USE_AUTO_PARTITION))),
    thd->mem_root, create_info->db_type));
```

★ 第二个参数决定是否构造 **partition handler**——用户定义了分区，或引擎自带 `HA_USE_AUTO_PARTITION`。

**③ 自动分区与分区校验**（这段是函数里最"绕"的一部分）：

```cpp
// ① 引擎自带自动分区、而用户没定义分区 → 让 handler 搭默认分区
if (!part_info && create_info->db_type->partition_flags &&
    (create_info->db_type->partition_flags() & HA_USE_AUTO_PARTITION)) {
  Partition_handler *part_handler = file->get_partition_handler();
  assert(part_handler != nullptr);
  /*
    Table is not defined as a partitioned table but the engine handles
    all tables as partitioned. The handler will set up the partition info
    object with the default settings.
  */
  thd->work_part_info = part_info = new (thd->mem_root) partition_info();
  part_handler->set_auto_partitions(part_info);
  part_info->default_engine_type = create_info->db_type;
  part_info->is_auto_partitioned = true;
}

if (part_info) {
  // ② 分区注释合法性、逐分区引擎解析……
  if (part_info->check_partition_info(thd, &engine_type, file.get(),
                                      create_info, false))
    return true;
  part_info->default_engine_type = engine_type;

  // ③ 引擎根本不支持分区 → 报错
  if (!engine_type->partition_flags) {
    my_error(ER_CHECK_NOT_IMPLEMENTED, MYF(0), "native partitioning");
    return true;
  }

  // ④ ★ 校验过程中引擎可能被改 → 重新获取 handler
  if (create_info->db_type != engine_type) {
    file.reset(get_new_handler((TABLE_SHARE *)nullptr, true, thd->mem_root,
                               engine_type));
    create_info->db_type = engine_type;
  }
}
```

★ 最后一个分支很反直觉：**handler 可能被获取两次**。注释解释了原因——分区子句里可能指定了与主表不同的引擎，此时 `check_partition_info` 会解析出真正的 `engine_type`，必须换掉先前那个 handler。这也解释了为什么 `file` 是 `unique_ptr` 而非直接对象。

**④ 命名冲突检查**

```cpp
Table_exists_result ter = check_if_table_exists(
    thd, db, table_name, alias,
    (create_info->options & HA_LEX_CREATE_TMP_TABLE),
    (create_info->options & HA_LEX_CREATE_IF_NOT_EXISTS), internal_tmp_table);
if (ter.m_error) return true;
if (ter.m_table_exists) return false;   // IF NOT EXISTS 时已在里面 push warning
```

`Table_exists_result` 只有两个字段（`m_table_exists` / `m_error`），语义干净：临时表走 `find_temporary_table()`，普通表走 `dd_client()->acquire()`。

**⑤ 白名单抑制键长错误**（系统表/DD 表例外）

```cpp
/* Suppress key length errors if this is a white listed table. */
Key_length_error_handler error_handler;
bool is_whitelisted_table = ...
if (is_whitelisted_table) thd->push_internal_handler(&error_handler);
bool prepare_error = mysql_prepare_create_table(...);
if (is_whitelisted_table) thd->pop_internal_handler();
```

**⑥ 最后分岔**

```cpp
/*
  Create table definitions.
  If "no_ha_table" is false also create table in storage engine.
*/
if (create_info->options & HA_LEX_CREATE_TMP_TABLE) {
  if (rea_create_tmp_table(...)) return true;
} else {
  if (rea_create_base_table(...)) return true;
}
```

**⑦ 收尾：临时表会强制把会话拖进事务**

```cpp
THD_STAGE_INFO(thd, stage_after_create);
if ((create_info->options & HA_LEX_CREATE_TMP_TABLE) &&
    thd->in_multi_stmt_transaction_mode()) {
  /*
    When autocommit is disabled, creating temporary table sets this
    flag to start transaction in any case (regardless of binlog=on/off,
    binlog format and transactional/non-transactional engine) to make
    behavior consistent.
  */
  thd->server_status |= SERVER_STATUS_IN_TRANS;
}
return false;
```

★ 这是一个容易踩的反直觉点：**autocommit 关闭时，创建临时表一定会开启事务**——与 binlog 开关、binlog 格式、引擎是否事务型**全都无关**。注释里写明这样做是为了"make behavior consistent"。后果是：一个看似无副作用的 `CREATE TEMPORARY TABLE` 会隐式开启事务，若之后忘了提交，后续语句都被包在里面。

### ★ `mysql_prepare_create_table()`：全部"表定义合法性"

这是 CREATE 与 ALTER 共用的定义规范化器。分步：

**① 表级属性校验**

- `alter_info->create_list.elements > MAX_FIELDS` → `ER_TOO_MANY_FIELDS`
- `max_rows`/`min_rows` 超 `UINT_MAX32` **截断并告警**
- 表注释非法字符集子串 → `ER_COMMENT_CONTAINS_INVALID_STRING` + `validate_comment_length()`
- ★ 三个**表选项字符串**同样过长度闸（都在最开头）：`CONNECTION` > `CONNECT_STRING_MAXLEN`、`COMPRESSION` > `TABLE_COMMENT_MAXLEN`、`ENCRYPT_TYPE` 超 `ER_WRONG_STRING_LENGTH` —— 顺序是先连接串/压缩/加密、后注释，**任何一个超限立即返回**，不会等到引擎层才发现
- `row_type == ROW_TYPE_DYNAMIC` → `create_info->table_options |= HA_OPTION_PACK_RECORD`

> 这里有一段暴露历史债的注释：
> ```
> Checks which previously were done during .FRM creation.
>
> TODO: Check if the old .FRM limitations still make sense
> with the new DD.
> ```

**② 逐列规范化：`prepare_create_field()` 循环**

★ 顺序有讲究（注释原文解释为什么必须先于函数索引）：

```cpp
/*
  Prepare fields, which must be done before calling
  add_functional_index_to_create_list(). The reason is that
  prepare_create_field() sets several properties of all Create_fields, such as
  character set. We need the character set in order to get the correct
  display width for each Create_field, which is in turn needed to resolve the
  correct data type/length for each hidden generated column added by
  add_functional_index_to_create_list().
*/
```

`prepare_create_field()` 做的事：定字符集（`BINCMP_FLAG` 如 `CHAR(1) BINARY` → 换 `_bin` 排序规则）、默认值从客户端字符集转列字符集、`SET`/`ENUM`/`BIT`/`BLOB` 各自处理、`null_bits++`、`check_column_name()`、SRID 存在性检查。

**③ 函数索引 → 隐藏生成列**

```cpp
// Go through all functional key parts. For each functional key part, resolve
// the expression and add a hidden generated column to the create list.
```

**④ AUTO_INCREMENT / BLOB 计数**（含 `HA_NO_AUTO_INCREMENT` / `HA_NO_BLOBS` 引擎能力检查）；CTAS 带自增列打 unsafe 标记。

**⑤ 索引计数 + 冗余索引剔除 + 上限**

```cpp
/*
  Count keys and key segments.
  Also mark redundant keys to be ignored.
*/
if (*key_count > file->max_keys()) { my_error(ER_TOO_MANY_KEYS, MYF(0), file->max_keys()); return true; }
```

非外键键准备完成后、外键之前，还有三道**逐键检查**（源码顺序）：

```cpp
  // sql_require_primary_key：无 PK 时检查是否被禁止。
  // 注释明说限制范围：只对"能参与复制"的 SE 生效
  if (!primary_key && check_promoted_index(file, *key_info_buffer, *key_count))
    ...;                                        // 无 PK → 找可提升为 PK 的唯一索引

  for (dup_check_key = keys_to_check.begin(); ...) {
    if (check_duplicate_key(thd, ..., *dup_check_key, *key_info_buffer, *key_count, ...))
      ...;                                      // 逐键查重（名字/定义重复）
  }
```

1. **`sql_require_primary_key`**——注释明确写了边界：*"Limit the effect of sql_require_primary_key to only those SEs that can participate in replication"*——GIPK 体系的开关只管"会被复制的引擎"，MyISAM 这类不受约束。
2. ★ **`check_promoted_index`**——建表没写 PRIMARY KEY 时，从这里选一个合适的唯一索引**提升**为聚簇键（这决定了 InnoDB 里"没写 PK 也有聚簇索引"的来源之一；都没有才用隐藏 `DB_ROW_ID`）。
3. **`check_duplicate_key` 逐键跑**——`keys_to_check` 是剔完冗余后的键集合；重名/同定义在这里报 `ER_DUP_FIELDNAME`/`ER_DUP_KEYNAME`，不会拖到引擎层。

**⑥ `Key_spec` → `KEY` + `KEY_PART_INFO`**

```cpp
// First prepare non-foreign keys so that they are ready when
// we prepare foreign keys.
for (size_t i = 0; i < alter_info->key_list.size(); i++) {
  if (redundant_keys[i]) continue;  // Skip redundant keys
  const Key_spec *key = alter_info->key_list[i];   // parser 产物
  if (key->type == KEYTYPE_PRIMARY) { if (primary_key) { my_error(ER_MULTIPLE_PRI_KEY, MYF(0)); return true; } primary_key = true; }
  if (key->type != KEYTYPE_FOREIGN) { if (prepare_key(...)) return true; key_info++; key_number++; }
}
```

**⑦ 排序 + 外键**

```cpp
/* Sort keys in optimized order */
std::sort(*key_info_buffer, *key_info_buffer + *key_count, sort_keys());

/*
  Normal keys are done, now prepare foreign keys.

  We do this after sorting normal keys to get predictable behavior
  when searching for parent keys for self-referencing foreign keys.
*/
```

**⑧ TIMESTAMP NOT NULL 的历史坑**

```cpp
/*
  ... Note that this check must be after KEYs have been created as this
  can cause the NOT_NULL_FLAG to be set.
*/
```

**⑨ 行长检查**

```cpp
/* If fixed row records, we need one bit to check for deleted rows */
if (!(create_info->table_options & HA_OPTION_PACK_RECORD)) create_info->null_bits++;
ulong data_offset = (create_info->null_bits + 7) / 8;
...
if (reclength > file->max_record_length()) {
  my_error(ER_TOO_BIG_ROWSIZE, MYF(0), static_cast<long>(file->max_record_length()));
  return true;
}
```

### ★ `rea_create_base_table()`：DD 写入 + 调引擎的接缝

这是"server 侧原子 DDL"的核心接缝。注释原文极其关键：

```cpp
/**
  Create table definition in the Data Dictionary. The table is also
  created in the storage engine, depending on the 'no_ha_table' argument.
  ...
  @param do_not_store_in_dd    Indicates that we should postpone storing table
                               object in the data-dictionary. Requires SE
                               supporting atomic DDL and no_ha_table flag set.
  ...
  @note For engines supporting atomic DDL the caller must rollback
        both statement and transaction on failure. This must be done
        before any further accesses to DD. @sa dd::create_table().
*/
```

**两种落盘策略**（★ 原子 DDL 的关键开关）：

```cpp
if (do_not_store_in_dd) {
  /*
    Clean up code assumes that SE supports atomic DDL if do_not_store_in_dd
    was requested, so we can simply rollback our changes.

    ha_create_table() won't work correctly if dd::Table object is not stored
    in the data-dictionary.
    ...
  */
  assert(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
  assert(no_ha_table);
  assert(!dd::get_dictionary()->get_dd_table(db, table_name));
  *table_def_ptr = std::move(table_def_res);
  table_def = table_def_ptr->get();
} else {
  bool result = thd->dd_client()->store(table_def_res.get());
  if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
      !thd->is_plugin_fake_ddl())
    result = trans_intermediate_ddl_commit(thd, result);   // 非原子引擎：立刻提交
  if (result) return true;
  if (thd->dd_client()->acquire_for_modification(db, table_name, &table_def)) return true;
}
```

落盘分岔的源码本体（这是整个函数最核心的一段）：

```cpp
std::unique_ptr<dd::Table> table_def_res = dd::create_table(
    thd, sch_obj, table_name, create_info, create_fields, key_info, keys,
    keys_onoff, fk_key_info, fk_keys, check_cons_spec, file);
if (!table_def_res) return true;

if (do_not_store_in_dd) {
  /*
    Clean up code assumes that SE supports atomic DDL if do_not_store_in_dd
    was requested, so we can simply rollback our changes.

    ha_create_table() won't work correctly if dd::Table object is not stored
    in the data-dictionary.
  */
  assert(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
  assert(no_ha_table);
  assert(!dd::get_dictionary()->get_dd_table(db, table_name));

  *table_def_ptr = std::move(table_def_res);     // ★ 交给调用方保管，不落盘
  table_def = table_def_ptr->get();
} else {
  bool result = thd->dd_client()->store(table_def_res.get());   // 落盘

  if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
      !thd->is_plugin_fake_ddl())
    result = trans_intermediate_ddl_commit(thd, result);        // ★ 非原子：立刻中间提交

  if (result) return true;

  if (thd->dd_client()->acquire_for_modification(db, table_name, &table_def))
    return true;
}
```

★ 三条 assert 说明 `do_not_store_in_dd` **不能单独使用**：它要求引擎支持原子 DDL、**且 `no_ha_table` 为真**（不调引擎建表）、且不是 DD 自有表。原因就在注释里——`ha_create_table()` 需要 `dd::Table` 已经 stored 才能正常工作；既然原子路径选择不 store，就只能同时放弃"这时就建引擎表"，把建表推迟到后续提交阶段统一处理。**这就是前面 `create_table_impl` 那个 `do_not_store_in_dd = HTON_SUPPORTS_ATOMIC_DDL` 参数与 `no_ha_table` 必须配套出现的根因。**

⇒ **原子 DDL 引擎 → DD 变更留在当前语句事务里不提交；非原子引擎 → 立刻中间提交**，尽量让 DD 与 SE 同步。

**调引擎，失败时非原子引擎要手工补做 DD 删除**：

```cpp
if (ha_create_table(thd, path, db, table_name, create_info, false, false, table_def)) {
  /*
    Remove table from data-dictionary if it was added and rollback
    won't do this automatically.
  */
  if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
    bool result = dd::drop_table(thd, db, table_name, *table_def);
    if (!thd->is_plugin_fake_ddl()) (void)trans_intermediate_ddl_commit(thd, result);
  }
  return true;
}
```

**★ 决定 binlog 用 trx cache 还是 stmt cache**：

```cpp
/*
  If the SE supports atomic DDL, we can use the trx binlog cache.
  Otherwise we must use the statement cache.
*/
if (binlog_to_trx_cache != nullptr)
  *binlog_to_trx_cache = (create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL);
```

**另外两处容易漏的细节**：

**① 分区表在"不建引擎表"时要试开一次表**

```cpp
if (no_ha_table) {
  if (part_info) {
    /*
      For partitioned tables we can't find some problems with table
      until table is opened. Therefore in order to disallow creation
      of corrupted tables we have to try to open table as the part
      of its creation process.
      In cases when both .FRM and SE part of table are created table
      is implicitly open in ha_create_table() call.
      In cases when we create .FRM without SE part we have to open
      table explicitly.
    */
    TABLE table; TABLE_SHARE share;
    init_tmp_table_share(thd, &share, db, 0, table_name, path, nullptr);
    bool result = open_table_def(thd, &share, *table_def) ||
                  open_table_from_share(thd, &share, "", 0, (uint)READ_ALL, 0,
                                        &table, true, nullptr);
    ...
    if (result) {
      // 非原子引擎：已经提交过了，只能手工从 DD 删掉
      if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
        bool drop_result = dd::drop_table(thd, db, table_name, *table_def);
        (void)trans_intermediate_ddl_commit(thd, drop_result);
      }
      return true;
    }
  }
  return false;
}
```

★ 为什么分区表要额外"试开"：注释说得很清楚——**分区表的有些问题只有把表打开才暴露**（分区函数、分区表达式的合法性）。正常情况下 `ha_create_table()` 会隐式打开表，但 `no_ha_table` 路径没有建引擎表，就只能显式开一次来做校验。

**② 登记 post_ddl 钩子**

```cpp
if ((create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
    create_info->db_type->post_ddl)
  *post_ddl_ht = create_info->db_type;
```

即：只有**支持原子 DDL 且实现了 `post_ddl` 钩子**的引擎才会被登记，调用方在 DDL 提交/回滚后回调它（见 [`atomic_ddl.md`](atomic_ddl.md) 的 post_ddl 节）。

**③ 失败清理的对称性**：`ha_create_table()` 失败时，原子引擎**什么都不做**（事务回滚自动清掉 DD 记录），非原子引擎必须手工 `dd::drop_table()` + `trans_intermediate_ddl_commit()`——因为它刚才已经中间提交过了，回滚不了。这正是"原子 DDL 简化错误处理"最直观的源码体现。

### `ha_create_table()`：server 与 handler 的合同

```cpp
error = table.file->ha_create(name, &table, create_info, table_def);
if (error) { table.file->print_error(error, MYF(0)); ... } else {
  /*
    We do post-create update only for engines supporting atomic DDL
    as only such engines are allowed to update dd::Table objects in
    handler::ha_create().
    ...
  */
  if (!((create_info->options & HA_LEX_CREATE_TMP_TABLE) || is_temp_table ||
        dd::get_dictionary()->is_dd_table_name(db, table_name)) &&
      (table.file->ht->flags & HTON_SUPPORTS_ATOMIC_DDL)) {
    if (thd->dd_client()->update<dd::Table>(table_def)) error = 1;
  }
}
```

★ **关键约定**：`handler::ha_create()` 之后还要再 `dd_client()->update()`，因为**引擎可以在 `ha_create()` 里改 `dd::Table`**（填 `se_private_data` / 表空间 id）。这正是原子 DDL 能成立的接口前提——引擎对 DD 的修改也发生在同一事务里。

---

## 引擎侧落地

跨过 `ha_create_table()` 这个接缝之后就完全是 InnoDB 的世界。server 侧到这里为止只回答了"这张表叫什么、合不合法、DD 里怎么写"，**"数据放哪"和"怎么组织"**两件事才刚开始。

### `create_table_info_t`：引擎侧建表的"参数袋"

⚠️ **勘误**：
- **`create_table_info_t` 不在 `handler0alter.cc`**——定义在 **`ha_innodb.h:828`**，成员函数实现**全部**在 `ha_innodb.cc`。`handler0alter.cc` 只**使用**它
- **不存在 `create_table_info_t::create_table_space()`**（建表空间真身是 `dict_build_tablespace_for_table()`）
- **不存在 `create_table_info_t::create_index()`**——是文件级自由函数 `inline int create_index(...)`（`ha_innodb.cc:11930`）
- **不存在 `m_dd_table` 成员**——真名是 `const TABLE *m_form`；`m_dd_table` 属于 `class truncate_t`
- **不存在 `innobase_get_tablespace_flags()`**——行格式换算真名是 `dict_tf_set()`
- **不存在 `allocate_row_log`**

```cpp
/** Class for handling create table information. */
class create_table_info_t {
 public:
  create_table_info_t(THD *thd, TABLE *form, HA_CREATE_INFO *create_info,
                      char *table_name, char *remote_path, char *tablespace,
                      bool file_per_table, bool skip_strict, uint32_t old_flags,
                      uint32_t old_flags2, bool is_partition)
      : m_thd(thd),
        m_trx(thd_to_trx(thd)),       // ★ trx 直接从 THD 上取，不是新建
        m_form(form),
        m_create_info(create_info),
        m_table_name(table_name),
        m_remote_path(remote_path),
        m_tablespace(tablespace),
        m_innodb_file_per_table(file_per_table),
        m_flags(old_flags),
        m_flags2(old_flags2),
        m_skip_strict(skip_strict),
        m_partition(is_partition) {}

  int initialize();
  void set_tablespace_type(bool table_being_altered_is_file_per_table);
  int create_table(const dd::Table *dd_table, const dd::Table *old_part_table);
  int create_table_update_dict();
  template <typename Table> int create_table_update_global_dd(Table *dd_table);

  /** Validates the create options. ... These CREATE OPTIONS are not validated
  here unless innodb_strict_mode is on. ... */
  const char *create_options_are_invalid();

  bool create_option_data_directory_is_valid(bool ignore = false);
  bool create_option_tablespace_is_valid();
  bool create_option_compression_is_valid();
  int prepare_create_table(const char *name);
  bool innobase_table_flags();
  void set_remote_path_flags();       // ★ 定义在 ha_innopart.cc:2372（分区专用），不在 ha_innodb.cc

  uint32_t flags() const { return (m_flags); }
  uint32_t flags2() const { return (m_flags2); }
  inline bool is_intrinsic_temp_table() const {
    return ((m_flags2 & DICT_TF2_INTRINSIC) != 0);
  }
  /** @return true only if table is temporary and not intrinsic */
  inline bool is_temp_table() const {
    return (((m_flags2 & DICT_TF2_TEMPORARY) != 0) &&
            ((m_flags2 & DICT_TF2_INTRINSIC) == 0));
  }
  void detach();

 private:
  int create_table_def(const dd::Table *dd_table,
                       const dd::Table *old_part_table);
  void initialize_autoinc();
  int parse_table_name(const char *name);

  THD *m_thd;
  trx_t *m_trx;
  /** Information on table columns and indexes. */
  const TABLE *m_form;              // ★★ 列/索引元数据全部从 server 的 TABLE* 取
  HA_CREATE_INFO *m_create_info;
  char *m_table_name;
  char *m_remote_path;              // Remote path (DATA DIRECTORY) or zero length-string
  char *m_tablespace;
  /** The newly created InnoDB table object. ... it can be accessed directly. */
  dict_table_t *m_table;
  bool m_innodb_file_per_table;
  bool m_allow_file_per_table;
  bool m_use_file_per_table;
  bool m_use_data_dir;
  bool m_use_shared_space;
  uint32_t m_flags;
  uint32_t m_flags2;
  bool m_skip_strict;
  bool m_partition;
};
```

> ★★ **`m_form`（`TABLE*`）与 `dd::Table*` 的分工**：
> - **从 `m_form` 拿**：列数 `s->fields`、`m_form->field[i]` 逐列、索引 `key_info[i]`/`s->keys`、row_type `s->row_type`、`found_next_number_field`
> - **从 `dd::Table*` 拿**：外键（`innobase_check_fk_base_col`）、**SRID**（空间索引，`TABLE*` 里没有）、隐藏 FTS_DOC_ID 索引、INSTANT 列元数据（分区继承）
>
> 所以 `create_table()` 的签名是 `create_table(const dd::Table *dd_table, ...)`，`dd_table` **可以为 nullptr**（intrinsic 临时表没有 DD 对象）。

**`initialize()`**（`ha_innodb.cc:13556`）——只做三件事：

```cpp
int create_table_info_t::initialize() {
  if (m_form->s->fields > REC_MAX_N_USER_FIELDS) {
    return HA_ERR_TOO_MANY_FIELDS;      // ★ InnoDB 侧列数上限（与 server 的 MAX_FIELDS 不同！）
  }
  ut_ad(m_form->s->row_type == m_create_info->row_type);

  /* Check for name conflicts (with reserved name) for any user indices. */
  if (innobase_index_name_is_reserved(m_thd, m_form->key_info,
                                      m_form->s->keys)) {
    return HA_ERR_WRONG_INDEX;          // ★ 撞 GEN_CLUST_INDEX 就拒
  }

  m_trx->will_lock++;
  m_table = nullptr;
  return 0;
}
```

**`prepare_create_table()`**（`ha_innodb.cc:13646`）——四步：

```cpp
int create_table_info_t::prepare_create_table(const char *name) {
  if (!normalize_table_name(m_table_name, name)) {
    ut_d(ut_error);
    ut_o(return (HA_ERR_TOO_LONG_PATH));
  }

  set_tablespace_type(false);          // ① 决定 shared / file_per_table / data_dir

  if (create_options_are_invalid()) {  // ② 全量选项合法性校验
    return HA_WRONG_CREATE_OPTION;
  }

  if (flags() == 0 && flags2() == 0) { // ③ 旧 flags 为空时才推导（ALTER 复用旧值时跳过）
    if (!innobase_table_flags()) {     // ★ row_format / zip_ssize / flags / flags2 推导
      return HA_WRONG_CREATE_OPTION;
    }
  }

  ut_ad(!high_level_read_only || is_intrinsic_temp_table());
  return parse_table_name(name);       // ④ 拆 name → db/table + remote_path + tablespace
}
```

**`set_tablespace_type()`**（`ha_innodb.cc:13526`）：

```cpp
void create_table_info_t::set_tablespace_type(
    bool table_being_altered_is_file_per_table) {
  m_use_shared_space = tablespace_is_shared_space(m_create_info);

  m_allow_file_per_table = m_innodb_file_per_table ||
                           table_being_altered_is_file_per_table ||
                           tablespace_is_file_per_table(m_create_info);

  bool is_temp = m_create_info->options & HA_LEX_CREATE_TMP_TABLE;

  /* Ignore the current innodb_file_per_table setting if we are
  creating a temporary table or if the TABLESPACE= phrase is using
  an existing shared tablespace. */
  m_use_file_per_table =
      m_allow_file_per_table && !is_temp && !m_use_shared_space;

  /* DATA DIRECTORY must have m_use_file_per_table. */
  m_use_data_dir = m_use_file_per_table &&
                   (m_create_info->data_file_name != nullptr) &&
                   (m_create_info->data_file_name[0] != '\0');
  ut_ad(!(m_use_shared_space && m_use_data_dir));
}
```

> ★★ **临时表恒 `m_use_file_per_table == false`**——它们永远在 session 临时表空间里，与 `innodb_file_per_table` 无关。

### `ha_innobase::create` → `create_impl`：参数解析与三段式

`ha_innobase::create` 本身几乎不做事，直接转给 `innobase_basic_ddl::create_impl`。注意它是个**模板函数**（`dd::Table` 与 `dd::Partition` 两个显式实例）——分区表与常规表的元数据类型不同，但建表逻辑共用一套，这是用模板消除重复：

```cpp
int innobase_basic_ddl::create_impl(THD *thd, const char *name, TABLE *form,
                                    HA_CREATE_INFO *create_info, Table *dd_tab, ...) {
  trx = check_trx_exists(thd);

  // ★ 临时表不起事务：它不记 redo / undo，生命周期随 session，没必要走事务体系
  if (!(create_info->options & HA_LEX_CREATE_TMP_TABLE)) {
    trx_start_if_not_started(trx, true, UT_LOCATION_HERE);
  }

  // create_table_info_t 是两层参数体系的适配层：
  // 把 server 的 HA_CREATE_INFO 翻译成 InnoDB 的 m_flags / m_flags2 位图
  create_table_info_t info(thd, form, create_info, norm_name, remote_path,
                           tablespace, file_per_table, skip_strict, old_flags, ...);

  int error = info.initialize();            // ① 解析：表名归一化、表空间名、行格式、flags 位图
  if (error != 0) return error;

  error = info.prepare_create_table(name);  // ② 校验：选项组合是否合法（压缩/加密/行格式冲突等）
  if (error != 0) return error;

  error = info.create_table(dd_tab != nullptr ? &dd_tab->table() : nullptr,
                            old_part_table);          // ③ 落地：建 dict_table_t + 表空间 + 各索引 B-tree
  if (error) goto cleanup;

  // ④ 回填：引擎决定的信息（行格式、加密、表空间名）写回 server 的 dd::Table
  error = info.create_table_update_global_dd(dd_tab);
  if (error) goto cleanup;

  error = info.create_table_update_dict();  // ⑤ 收尾：统计置空、FTS stopword、解析表注释 hint
  ...
}
```

★ 这个**"解析 → 校验 → 落地 → 回填"三段式**值得注意：解析和校验分开，是为了让非法组合在进入物理建表之前就被拒绝——一旦 `create_table()` 开始建 .ibd 和 B-tree，就不是纯内存操作了。

`cleanup` 分支的处理也分两类：临时表直接从 dict cache 摘除索引并释放；intrinsic 临时表（optimizer 用的内部表）走 `unregister_table_handler` 释放。失败路径不做"部分清理表空间"——因为失败发生在建表空间之前的概率更高，且 post-DDL 的 DDL log 会兜底。

### `create_table_info_t::create_table`：建表 + 逐个建索引

```cpp
int create_table_info_t::create_table(const dd::Table *dd_table, const dd::Table *old_part_table) {
  // 先建表定义本体：分配 table_id、建表空间、dict_table_t 入缓存
  error = create_table_def(dd_table, old_part_table);
  if (error) return error;

  if (m_form->s->keys == 0 || primary_key_no == MAX_KEY) {
    // 用户没指定主键 → 建一个基于 rowid 的聚簇索引
    // （8.0 默认 GIPK 打开时通常不会走到这里，sql 层已隐式加了不可见主键）
    error = create_clustered_index_when_no_primary(m_trx, m_flags, m_table_name);
    if (error) return error;
  }

  if (primary_key_no != MAX_KEY) {
    // ★ 源码注释原话：In InnoDB the clustered index must always be created first
    // 原因：二级索引叶子页里存的是主键值，逻辑上从属于聚簇索引
    error = create_index(m_trx, m_form, m_flags, m_table_name, primary_key_no, dd_table);
    if (error) return error;
  }

  if (m_flags2 & (DICT_TF2_FTS | DICT_TF2_FTS_ADD_DOC_ID)) {
    ... fts_create_common_tables(m_trx, m_table, m_table_name, ...);   // FTS 辅助表
  }

  for (i = 0; i < m_form->s->keys; i++) {
    if (i != primary_key_no) {
      error = create_index(m_trx, m_form, m_flags, m_table_name, i, dd_table);  // 其余二级索引
      if (error) return error;
    }
  }

  initialize_autoinc();                    // 自增计数器初始化
  ...
  row_table_load_foreign_constraints(...); // FK 校验（intrinsic 表无 FK 概念，跳过）
}
```

### 表空间的创建：`dict_build_tablespace_for_table`

`create_table_def` 最终落到 `row_create_table_for_mysql` → `dict_build_table_def`：

```cpp
dberr_t dict_build_table_def(dict_table_t *table, const HA_CREATE_INFO *create_info, trx_t *trx) {
  if (...) dict_table_assign_new_id(table);   // ★ 分配 table_id（与 server 侧 dd::Table 的 id 对齐）

  dberr_t err = dict_build_tablespace_for_table(table, create_info, trx);  // ★ 建 .ibd，或复用共享表空间
  return err;
}
```

`row_create_table_for_mysql` 紧接着把表挂进字典缓存，**并写第一条 DDL log**：

```cpp
dict_sys_mutex_enter();
dict_table_add_to_cache(table, false);   // 加入全局字典缓存
dict_sys_mutex_exit();

// ★ CREATE 不是"什么都不记"：内存字典缓存已经被修改了，
//   回滚时必须能把它摘掉 → 记 REMOVE_CACHE_LOG
if (log_ddl != nullptr) {
  err = log_ddl->write_remove_cache_log(trx, table);
}
```

★ 这是 CREATE 与原子 DDL 的第一个交点。物理文件侧的回滚由 `DELETE_SPACE_LOG` 负责（整文件删除），内存缓存侧由 `REMOVE_CACHE_LOG` 负责——两条 log 配合才能把"建表"完整撤销，详见 [`atomic_ddl.md`](atomic_ddl.md)。

### ★ 索引树的创建：`btr_create` 建根页

每个索引走 `create_index` → `row_create_index_for_mysql`：

```cpp
if (!table->is_temporary()) {
  dict_build_index_def(table, index, trx);   // 分配 index_id、填充 dict_index_t
  err = dict_index_add_to_cache_w_vcol(table, index, nullptr, FIL_NULL, trx_is_strict(trx));
  index = UT_LIST_GET_LAST(table->indexes);
  err = dict_create_index_tree_in_mem(index, trx);   // ★ 真正建树
} else {
  // 临时表：同样建树，但全程 MTR_LOG_NO_REDO
  ...
}
```

`dict_create_index_tree_in_mem` 就是"建根页"这一步，是 CREATE 里唯一真正写数据页的地方：

```cpp
dberr_t dict_create_index_tree_in_mem(dict_index_t *index, trx_t *trx) {
  if (index->type == DICT_FTS) return DB_SUCCESS;     // FTS 索引没有 B-tree

  // import / discard 场景：文件尚未就位，此时不建树，只把 page 标为 FIL_NULL
  const bool missing = index->table->ibd_file_missing || dict_table_is_discarded(index->table);
  if (missing) { index->page = FIL_NULL; index->trx_id = trx->id; return DB_SUCCESS; }

  mtr_start(&mtr);
  if (index->table->is_temporary()) mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);  // 临时表不记 redo

  // ★ 建根页：为索引分配 LEAF / TOP 两个文件段，创建一个根页作为段头页
  page_no = btr_create(index->type, index->space, index->id, index, &mtr);
  index->page = page_no;    // 根页号写回 dict_index_t：以后定位这棵树全靠它
  index->trx_id = trx->id;
  mtr_commit(&mtr);

  if (page_no == FIL_NULL) {
    err = DB_OUT_OF_FILE_SPACE;
  } else if (log_ddl != nullptr) {
    err = log_ddl->write_free_tree_log(trx, index, false);   // ★ 记意图：这棵树将来可以释放
  }
  ...
}
```

★ **建树后立刻写 `FREE_TREE_LOG`**——这正是"意图日志 + 延迟执行"：现在只记下"这棵树存在、将来可释放"，真正释放要等提交后 post-DDL replay 才执行。注意它与 DROP 时写的 `FREE_TREE_LOG` 是同一个类型、同一个 replay 函数，只是 CREATE 场景下它服务于"回滚要能删掉刚建的树"。

源码在这里留了两个 FIXME，恰好说明了这条机制的边界：

```
// FIXME: Now writing ddl log after the index has been created,
// so if server crashes before the redo log gets persisted,
// there is no way to find the resources(two segments, etc.) allocated to this index.
// Since this is a rare case, living with it is acceptable

// FIXME: if it's part of CREATE TABLE, and file_per_table is true, skip ddl log,
// because during rollback, the whole tablespace would be dropped
```

即：① 建树与写日志之间有个极小的崩溃窗口，罕见，社区接受；② **CREATE TABLE + 独立表空间时这条日志是冗余的**——回滚时整个 .ibd 会被删掉，不必逐个 free 索引树。这是"整文件删除优于逐页释放"思路的又一次体现（与 DROP 时独立表空间不写 FREE_TREE_LOG 是同一个道理）。

`btr_create` 的物理代价很小：一个根页 + 两个段 inode（LEAF 段与 TOP 段）。所以 CREATE TABLE 的耗时主要在server 侧检查与 DD 写入，不在引擎侧建树。

### 顺带建出的一张特殊表：SDI 索引

CREATE TABLE 除了用户定义的表与索引，还会**在同一个表空间里建一张 SDI 表**（Serialized Dictionary Information——8.0 把元数据序列化后存进表空间自身，用于脱机场景读回定义）。入口是 `dict_sdi_create_idx_in_mem`：

```cpp
dict_index_t *dict_sdi_create_idx_in_mem(space_id_t space, bool space_discarded,
                                         uint32_t in_flags, bool is_create) {
  uint32_t flags = space_discarded ? in_flags : fil_space_get_flags(space);

  /* This means the tablespace is evicted from cache */
  if (flags == UINT32_UNDEFINED) {
    return (nullptr);
  }
  ...
  // SDI 表的列：type / id / compressed_len / uncompressed_len / data
  dict_mem_table_add_col(table, heap, "compressed_len", DATA_INT, ...);
  dict_mem_table_add_col(table, heap, "data", DATA_BLOB, DATA_NOT_NULL, 0, true);

  /* Initialize row version and column counts for new table */
  table->current_row_version = 0;              // ★ SDI 表也走行版本号机制
  table->initial_col_count = n_cols_sdi;
  table->current_col_count = table->initial_col_count;
  table->total_col_count = table->initial_col_count;

  table->id = dict_sdi_get_table_id(space);    // ★ SDI 表的 id 由 space_id 推导

  /* Disable persistent statistics on the table */
  dict_stats_set_persistent(table, false, true);   // ★ SDI 表不记持久化统计

  dict_index_t *temp_index = dict_mem_index_create(
      table_name, "CLUST_IND_SDI", space,
      DICT_CLUSTERED | DICT_UNIQUE | DICT_SDI, 2);
  temp_index->add_field("type", 0, true);
  temp_index->add_field("id", 0, true);

  /* Disable AHI on SDI tables */
  temp_index->disable_ahi = true;              // ★ SDI 索引禁用自适应哈希
  ...
  index->page = btr_create(DICT_CLUSTERED | DICT_UNIQUE | DICT_SDI, space_id,
                           index->id, index, &mtr);
```

★ 四点值得记：① SDI 索引的根页也是 `btr_create` 建的，**和用户的索引走同一条路径**，只是类型带 `DICT_SDI`；② SDI 表**同样有 `current_row_version`**——行版本号机制对它一视同仁（见 [`instant_ddl.md`](instant_ddl.md)）；③ SDI 表**显式关闭持久化统计**（`dict_stats_set_persistent(table, false, true)`），因为它是元数据载体，统计无意义；④ SDI 索引**禁用 AHI**。

> 这也是 CREATE TABLE 的引擎侧代价里容易被忽略的一项：除了用户索引，还多建一棵 SDI 索引（每个独立表空间一份）。

### 统计信息初始化：新表统计为空的成因

```cpp
int create_table_info_t::create_table_update_dict() {
  ...
  innobase_copy_frm_flags_from_create_info(m_table, m_create_info);

  dict_stats_update(m_table, DICT_STATS_EMPTY_TABLE);  // ★ 写入"空表"统计：0 行
  dict_stats_deinit(m_table);                          // 不常驻内存，下次打开表时再 fetch
  ...
}
```

只看到这一行会以为"置 0 行"就完了。往下拆两层，真相比"置空"复杂：

**第二层：`dict_stats_update` 的 `EMPTY_TABLE` 分支**——清空之后**还要落盘**：

```cpp
    case DICT_STATS_EMPTY_TABLE:

      dict_stats_empty_table(table);          // ① 内存里清空

      /* If table is using persistent stats,
      then save the stats on disk */
      if (dict_stats_is_persistent_enabled(table)) {
        return (dict_stats_save(table, nullptr));   // ② ★ 写进 mysql.innodb_*_stats
      }

      return (DB_SUCCESS);
```

**第三层：`dict_stats_empty_table` 本体——"置空"到底置了什么**：

```cpp
static void dict_stats_empty_table(dict_table_t *table) {
  /* Zero the stats members */
  dict_table_stats_lock(table, RW_X_LATCH);

  table->stat_n_rows = 0;
  table->stat_clustered_index_size = 1;                             // ★ 不是 0
  /* 1 page for each index, not counting the clustered */
  table->stat_sum_of_other_index_sizes = UT_LIST_GET_LEN(table->indexes) - 1;
  table->stat_modified_counter = 0;

  dict_index_t *index;
  for (index = table->first_index(); index != nullptr; index = index->next()) {
    if (index->type & DICT_FTS) continue;     // FTS 索引不参与
    ut_ad(!dict_index_is_ibuf(index));
    dict_stats_empty_index(index);            // 逐个索引置空
  }

  table->stat_initialized = true;             // ★★★ 关键
  dict_table_stats_unlock(table, RW_X_LATCH);
}
```

三条都是光看调用名看不出来的：

1. **"置空"并非全 0**：`stat_n_rows = 0`，但 `stat_clustered_index_size = 1`、其它索引各按 **1 页**计（`索引数 - 1`）。初值是"每个索引 1 页"的物理下界，不是零。
2. ★ **`stat_initialized = true`** —— 置空后立刻标记为"统计已初始化"。这是最容易被忽略、也最要命的一点：**系统认为这份统计是可用数据**，不会因为它是"空表默认值"而主动重算；只有 `stat_modified_counter` 累积过阈值才触发 auto recalc。
3. **空统计会被持久化**（若 `dict_stats_is_persistent_enabled`）：`dict_stats_save` 写进 `mysql.innodb_table_stats` / `mysql.innodb_index_stats`。⇒ 所以 COPY ALTER 后的"0 行"不是内存态，**重启也不会消失**。

**但紧接着下一行 `dict_stats_deinit` 又把标记复位了**——这两行连在一起看才完整：

```cpp
  dict_stats_update(m_table, DICT_STATS_EMPTY_TABLE);  // ① 置空 + 落盘（stat_initialized → true）
  dict_stats_deinit(m_table);                          // ② 立刻又把标记复位（→ false）
```

```cpp
// dict0stats.ic
/** Deinitialize table's stats after the last close of the table. This is
 used to detect "FLUSH TABLE" and refresh the stats upon next open. */
static inline void dict_stats_deinit(dict_table_t *table) {
  /* Don't assert the table->n_ref_count is 0 here, since there could
  be some background threads opening the table concurrently. This is
  not prevented. The cases prevented are in opening table code paths,
  to acquire an existing table, table->mutex is required */

  dict_table_stats_lock(table, RW_X_LATCH);

  if (!table->stat_initialized) {
    dict_table_stats_unlock(table, RW_X_LATCH);
    return;
  }

  table->stat_initialized = false;              // ★ 与①正好相反
  ...
}
```

| 步骤 | `stat_initialized` | 净效果 |
|---|---|---|
| ① `dict_stats_update(EMPTY_TABLE)` | → **true** | 空统计进内存，且（若 persistent）**落盘** |
| ② 紧接 `dict_stats_deinit` | → **false** | 内存标记复位，下次打开表重新 fetch |

★ 只看到 ① 会得出"标记已初始化、不会重算"的结论——那是**错的**。CREATE 结束时的真实状态是：**磁盘上已写入 0 行的持久化统计，但内存标记是"未初始化"**。所以下次打开这张表时，`DICT_STATS_FETCH_ONLY_IF_NOT_IN_MEMORY` 分支看到 `!stat_initialized`，会**从磁盘把这份 0 行统计读回来**（而不是重新采样计算）。

`dict_stats_deinit` 的注释也点明了它的本职：用来**检测 `FLUSH TABLE` 并在下次打开时刷新统计**；注释还解释为什么不断言 `n_ref_count == 0`——可能有后台线程并发打开这张表，这种并发不被阻止（被阻止的是"打开表"的代码路径，那里获取已存在的表需要 `table->mutex`）。

**⇒ 修正后的完整答案**：COPY ALTER 之后统计为空，根源是 COPY = "CREATE 新表 → 逐行 INSERT → rename"，而新表在 CREATE 时被写入**落盘的 0 行统计**（这才是持续影响的原因——重开表、重启都还在，且是"从磁盘读回"而非"重算"），叠加逐行 INSERT 不更新持久化统计。真正让它恢复正常仍要靠 `stat_modified_counter` 累积触发 `innodb_stats_auto_recalc`，或手动 `ANALYZE TABLE`。

对照 INPLACE：`alter_stats_rebuild` 会主动重算（`DICT_STATS_RECALC_PERSISTENT`）——**它绕过了上面整个"置空 + 标记初始化"的路径**，这才是两种算法在统计行为上差异的真正根源（不只是"一个重算一个不重算"）。

### 引擎侧收尾：`create_table_update_global_dd` / `update_dict`

- `create_table_update_global_dd`：把引擎决定的信息（行格式、加密、表空间名、SDI 等）**回填到 server 的 `dd::Table`**。这就是 `ha_create_table` 那节说的"引擎可改 `dd::Table`"——两层元数据在这里最终对齐。
- `create_table_update_dict`：统计置空 + FTS stopword 装载 + 解析表注释里的 hint。
- 最后 `info.detach()`：非临时表把 `dict_table_t` 交回全局缓存管理（不再由本线程持有）。

## 变体：CREATE ... LIKE / SELECT

| 变体 | 入口 | 关键差异 |
|---|---|---|
| 常规 | `mysql_create_table()` | 直接用 parser 产出的 `Alter_info` |
| **LIKE** | `mysql_create_like_table()` | ★ **先把源表"翻译"成 `Alter_info`**，再走和标准 CREATE 完全相同的路径 |
| SELECT / AS SELECT | `populate_table()` → `Query_result_create` | 中间有一次 commit，语句被拆成"DDL + DML"两段 |

★ **LIKE 复用了 ALTER 的代码**——把源表当一个"空 ALTER"来导出列/键定义：

```cpp
/*
  We the open source table to get its description in HA_CREATE_INFO
  and Alter_info objects. ...
*/
...
if (mysql_prepare_alter_table(thd, src_table_obj, src_table->table,
                              &local_create_info, &local_alter_info,
                              &local_alter_ctx)) return true;
```

LIKE 的 binlog 有张**四象限决策表**（源码原文）：

```
       ==== ========= ========= ==============================
       Case    Target    Source Write to binary log
       ==== ========= ========= ==============================
       1       normal    normal Original statement
       2       normal temporary Generated statement
       3    temporary    normal Nothing
       4    temporary temporary Nothing
       ==== ========= ========= ==============================
```

Case 2 必须生成完整语句，理由是**源表是临时表、备库上不一定存在**：

```cpp
/*
  As the reference table is temporary and may not exist on slave, we
  must force the ENGINE to be present into CREATE TABLE.
*/
create_info->used_fields |= HA_CREATE_USED_ENGINE;
```

**CTAS 的特殊性**（`populate_table()` 注释原文）：

```cpp
/**
  Populate tables from result of evaluating a query expression.

  This function is required because a statement like CREATE TABLE ... SELECT
  cannot be implemented using DML statement execution functions
  since it performs an intermediate commit that requires special attention.
*/
```

CTAS 还有一批额外禁止/告警，其中一个很典型——**原子 DDL + binlog ROW + 加 FK 的 CTAS 被直接禁止**：

```cpp
/*
  Disallow creation of foreign keys if,
  - SE supports atomic DDL's.
  - The binlogging is enabled.
  - The binlog format is ROW.
  This is done to avoid complications involved in locking,
  updating and invalidation (in case of rollback) of DD cache
  for parent table.
*/
if ((alter_info.flags & Alter_info::ADD_FOREIGN_KEY) &&
    (create_info.db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) && ...) {
  my_error(ER_FOREIGN_KEY_WITH_ATOMIC_CREATE_SELECT, MYF(0)); return true;
}
```

## 变体：临时表 / DD 系统表 / 内部临时表

| 类型 | server 侧差异 |
|---|---|
| **用户临时表** | `build_tmptable_filename()`；`ha_default_temp_handlerton()`；**`dd::create_tmp_table()`，`dd::Table` 不落盘**，所有权交给 `TABLE_SHARE::tmp_table_def`；禁止 ENCRYPTION/AUTOEXTEND_SIZE/secondary engine/分区；ROW 格式下不记 binlog |
| **DD 系统表** | `no_ha_table = true`（除 `--initialize`）；白名单抑制键长错误；**`ha_create_table()` 里跳过 post-create `dd_client()->update()`** |
| **内部临时表（ALTER 用）** | `create_info->m_hidden = !is_tmp_table`（注释：*"Avoid these tables to be visible by I_S/SHOW queries."*）；`check_if_table_exists()` 跳过 DD 查重 |

`rea_create_tmp_table()` 里关键点——`dd::create_tmp_table()` **不调 `dd_client()->store()`**：

```cpp
std::unique_ptr<dd::Table> tmp_table_ptr =
    dd::create_tmp_table(thd, sch_obj, table_name, create_info, create_fields,
                         key_info, keys, keys_onoff, check_cons_spec, file);
//  ★ dd::create_tmp_table() 只在内存里造 dd::Table，不调 dd_client()->store()
...
// Create the table in the storage engine.
if (ha_create_table(thd, path, db, table_name, create_info, false, false,
                    tmp_table_ptr.get()))
  return true;
...
// Transfer ownership of dd::Table object to TABLE_SHARE.
table->s->tmp_table_def = tmp_table_ptr.release();
//  ★★ 所有权移交给 TABLE_SHARE —— 所以临时表的 dd::Table 随 TABLE_SHARE 一起销毁
```

## 可观测性

### 系统变量

⚠️ **勘误**：**`innodb_file_per_table` / `innodb_strict_mode` 不在 `sql/sys_vars.cc`**，而在 `storage/innobase/handler/ha_innodb.cc`（插件变量，变量名 = `innodb_` + 宏首参）。**`innodb_file_format` / `innodb_file_format_max` / `innodb_file_format_check` 已彻底删除**（全仓仅 `mysql-test/std_data/mysqldump_57.sql` 一处历史文本）。

```cpp
// storage/innobase/handler/ha_innodb.cc:22096
static MYSQL_SYSVAR_BOOL(
    file_per_table, srv_file_per_table, PLUGIN_VAR_NOCMDARG,
    "Stores each InnoDB table to an .ibd file in the database dir.", nullptr,
    nullptr, true);          // ★ 默认 ON；无 ON_CHECK / ON_UPDATE

// storage/innobase/handler/ha_innodb.cc:1084
static MYSQL_THDVAR_BOOL(strict_mode, PLUGIN_VAR_OPCMDARG,
                         "Use strict mode when evaluating create options.",
                         innodb_check_session_admin, nullptr, true);
//  ★ 是 THDVAR（SESSION 级）；默认 true；ON_CHECK = innodb_check_session_admin
```

```cpp
// storage/innobase/handler/ha_innodb.cc:23075
static const char *innodb_default_row_format_names[] = {"redundant", "compact",
                                                        "dynamic", NullS};
static MYSQL_SYSVAR_ENUM(
    default_row_format, innodb_default_row_format, PLUGIN_VAR_RQCMDARG,
    "The default ROW FORMAT for all innodb tables created without explicit"
    " ROW_FORMAT. Possible values are REDUNDANT, COMPACT, and DYNAMIC."
    " The ROW_FORMAT value COMPRESSED is not allowed",
    nullptr, nullptr, DEFAULT_ROW_FORMAT_DYNAMIC,
    &innodb_default_row_format_typelib);
//  ★★ 只有 3 个枚举值：redundant / compact / dynamic —— COMPRESSED 不允许作默认值
```

```cpp
// storage/innobase/handler/ha_innodb.cc:22574
static MYSQL_SYSVAR_ULONG(page_size, srv_page_size,
                          PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY |
                              PLUGIN_VAR_NOPERSIST,
                          "Page size to use for all InnoDB tablespaces.",
                          nullptr, nullptr, UNIV_PAGE_SIZE_DEF,
                          UNIV_PAGE_SIZE_MIN, UNIV_PAGE_SIZE_MAX, 0);
//  ★ READONLY + NOPERSIST：只能启动时指定，不可 SET PERSIST

// storage/innobase/handler/ha_innodb.cc:22835 —— 仍然存在
static MYSQL_SYSVAR_STR(data_file_path, innobase_data_file_path,
                        PLUGIN_VAR_RQCMDARG | PLUGIN_VAR_READONLY |
                            PLUGIN_VAR_NOPERSIST,
                        "Path to individual files and their sizes.", nullptr,
                        nullptr, (char *)"ibdata1:12M:autoextend");
```

**server 侧（`sql/sys_vars.cc`）**：

```cpp
static Sys_var_plugin Sys_default_storage_engine(
    "default_storage_engine", "The default storage engine for new tables",
    SESSION_VAR(table_plugin), NO_CMD_LINE, MYSQL_STORAGE_ENGINE_PLUGIN,
    DEFAULT(&default_storage_engine), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(check_storage_engine));

static Sys_var_plugin Sys_default_tmp_storage_engine(
    "default_tmp_storage_engine",
    "The default storage engine for new explicit temporary tables",
    HINT_UPDATEABLE SESSION_VAR(temp_table_plugin), NO_CMD_LINE,
    MYSQL_STORAGE_ENGINE_PLUGIN, DEFAULT(&default_tmp_storage_engine),
    NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(check_storage_engine));

static Sys_var_ulonglong Sys_tmp_table_size(
    "tmp_table_size",
    "If an internal in-memory temporary table in the MEMORY or TempTable "
    "storage engine exceeds this size, MySQL will automatically convert it "
    "to an on-disk table ",
    HINT_UPDATEABLE SESSION_VAR(tmp_table_size), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(1024, std::numeric_limits<ulonglong>::max()),
    DEFAULT(16 * 1024 * 1024), BLOCK_SIZE(1));

static Sys_var_ulonglong Sys_max_heap_table_size(
    "max_heap_table_size",
    "Don't allow creation of heap tables bigger than this",
    HINT_UPDATEABLE SESSION_VAR(max_heap_table_size), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(16384, (ulonglong) ~(intptr)0), DEFAULT(16 * 1024 * 1024),
    BLOCK_SIZE(1024));
```

```cpp
// sql/sys_vars.cc:5344 —— 内部临时表引擎只有 MEMORY / TempTable 两个
const char *internal_tmp_mem_storage_engine_names[] = {"MEMORY", "TempTable",
                                                       nullptr};
static Sys_var_enum Sys_internal_tmp_mem_storage_engine(
    "internal_tmp_mem_storage_engine",
    "The default storage engine for in-memory internal temporary tables.",
    HINT_UPDATEABLE SESSION_VAR(internal_tmp_mem_storage_engine),
    CMD_LINE(REQUIRED_ARG), internal_tmp_mem_storage_engine_names,
    DEFAULT(TMP_TABLE_TEMPTABLE), NO_MUTEX_GUARD, NOT_IN_BINLOG,
    ON_CHECK(check_session_admin_no_super));

static Sys_var_ulonglong Sys_temptable_max_ram(
    "temptable_max_ram",
    "Maximum amount of memory (in bytes) the TempTable storage engine is "
    "allowed to allocate from the main memory (RAM) before starting to "
    "store data on disk.",
    GLOBAL_VAR(temptable_max_ram), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(2 << 20 /* 2 MiB */, ULLONG_MAX), DEFAULT(1 << 30 /* 1 GiB */),
    BLOCK_SIZE(1));
```

> ⚠️ **`max_allowed_packet` 不是建表参数**。它对 CREATE TABLE 的**唯一间接影响**是 binlog 单事件大小上限（`sql/binlog.cc:3448/3519` 用它算 `max_event_size`），以及 `store_create_info()` 生成的 CREATE TABLE 文本被复制时的长度。**源码中没有任何"建表校验 `max_allowed_packet`"的逻辑**。

**建表相关状态变量**（`sql/mysqld.cc:9746 / 9859`）：

```cpp
    {"Created_tmp_disk_tables",
     (char *)offsetof(System_status_var, created_tmp_disk_tables),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    {"Created_tmp_files", (char *)&my_tmp_file_created, SHOW_LONG,
     SHOW_SCOPE_GLOBAL},
    {"Created_tmp_tables",
     (char *)offsetof(System_status_var, created_tmp_tables),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    ...
    {"Opened_tables", (char *)offsetof(System_status_var, opened_tables),
     SHOW_LONGLONG_STATUS, SHOW_SCOPE_ALL},
    {"Opened_table_definitions",
     (char *)offsetof(System_status_var, opened_shares), SHOW_LONGLONG_STATUS,
     SHOW_SCOPE_ALL},
```

| 状态变量 | 自增点 |
|---|---|
| `Created_tmp_tables` | `sql/sql_tmp_table.cc:2373` `thd->inc_status_created_tmp_tables();` |
| `Created_tmp_disk_tables` | `sql/sql_tmp_table.cc:2304`（**且 `db_type != temptable_hton` 才算**） |
| `Opened_tables` | `sql/table.cc:3265` `if (db_stat) thd->status_var.opened_tables++;` |

```cpp
// sql/sql_class.cc:2379
void THD::inc_status_created_tmp_disk_tables() {
  assert(!status_var_aggregated);
  status_var.created_tmp_disk_tables++;
  PSI_STATEMENT_CALL(inc_statement_created_tmp_disk_tables)(m_statement_psi, 1);
}
```

> ★ `System_status_var` 的第一个字段必须紧跟 `#define FIRST_STATUS_VAR created_tmp_disk_tables`（`sql/system_variables.h:513`）——**SHOW STATUS 的求和优化依赖字段顺序**。

**★ CREATE TABLE 的 stage 只有 2 个**（`sql/sql_table.cc:8944 / 9018`）：

```cpp
  THD_STAGE_INFO(thd, stage_creating_table);   // "creating table"
  ...
  THD_STAGE_INFO(thd, stage_after_create);     // "After create"
```

```cpp
// sql/mysqld.cc:12180
PSI_stage_info stage_creating_table= { 0, "creating table", 0, PSI_DOCUMENT_ME};
PSI_stage_info stage_after_create= { 0, "After create", 0, PSI_DOCUMENT_ME};
```

> ⚠️ **`create_table_impl()` 里没有任何 `DEBUG_SYNC`**。想卡在建表中间只能用 stage，或引擎侧的 sync point（如 `ib_ddl_crash_during_create2`）。
> ★ 变体路径才有 DEBUG_SYNC：`create_table_like_after_open`、`create_table_like_before_binlog`、`create_table_select_before_lock`、`create_table_select_before_create`、`create_table_select_before_open`、`create_select_before/after_write_create_event`。

| 变量名 | 默认值 | 作用域 | 说明 |
|--------|--------|--------|------|
| `innodb_file_per_table` | ON | Global/Session | 决定建独立 `.ibd` 还是用共享表空间；同时决定 CREATE 时是否写 `DELETE_SPACE_LOG`、DROP 时是整文件删除还是逐页释放 |
| `innodb_default_row_format` | dynamic | Global | 未显式指定行格式时的新表默认值 |
| `innodb_print_ddl_logs` | OFF | Global | 把 CREATE 写入/删除的 DDL log 打到错误日志；★ 还需 `log_error_verbosity=3` 才可见 |

### 观测对象 → 手段 速查

| 我想看 | 手段 | 说明 |
|--------|------|------|
| 建表写了哪些 DDL log | 错误日志 | `SET GLOBAL innodb_print_ddl_logs=1` 后建表，会看到 `DELETE SPACE` / `REMOVE CACHE` / `FREE` 三条 insert **各紧跟一条 delete**——成功路径的直接证据（解读见 [`atomic_ddl.md`](atomic_ddl.md)） |
| 新表元数据是否落定 | SQL | `information_schema.INNODB_TABLES`（`TABLE_ID` / `N_COLS` / 行格式）与 `INNODB_COLUMNS` |
| 新表统计 | SQL | 新表统计**必然 0 行**（`DICT_STATS_EMPTY_TABLE`），需 `ANALYZE` 或等 auto_recalc；见「统计信息初始化」 |
| 建表失败是否留残留 | 错误日志 + 文件系统 | 支持原子 DDL 的引擎靠 DDL log replay 清理；不支持的由 server 的 post-DDL 钩子清理（源码注释明确这是该钩子在 CREATE 场景的主要职责） |

---

## Misc

### 引擎侧扩展点：新增一个建表选项要改哪几处

| 改动点 | 作用 |
|---|---|
| `create_table_info_t::initialize()` | 从 `HA_CREATE_INFO` 解析出 InnoDB flags 位（新选项在这里翻译） |
| `create_table_info_t::prepare_create_table()` | 选项组合合法性校验（与已有选项冲突在这里拒绝） |
| `create_table_def()` → `dict_build_tablespace_for_table()` | 影响表空间创建（加密、压缩、page size） |
| `create_table_update_global_dd()` | 把引擎侧最终决定回填给 server 的 `dd::Table` |

### 已知缺陷与坑

**坑 1：CREATE TABLE ... LIKE 不复制外键**

`mysql_create_like_table()` 里注释写得很直白：

```cpp
  if (mysql_create_table_no_lock(
          thd, table->db, table->table_name, &local_create_info,
          &local_alter_info, 0,
          false,  // No FKs, no need to lookup parent keys
          &is_trans, &post_ddl_ht))
```

**保留 vs 丢弃完整清单**：

| 属性 | 结果 | 源码位置 |
|---|---|---|
| 列定义（含隐藏列/生成列/显式 collation） | ✅ 保留 | `mysql_prepare_alter_table()` → `prepare_fields_and_keys()` |
| **索引（PK/UNIQUE/二级）** | ✅ 保留 | 同上 |
| CHECK 约束 | ✅ 保留（改名后挂到新表） | `prepare_check_constraints_for_create_like_table()` |
| 分区定义 | ✅ 保留 | `part_info->get_clone(thd)`（**唯一的 clone 调用**） |
| TABLESPACE | ⚠️ **仅"显式指定"才继承** | `if (src_table_obj && !src_table_obj->is_explicit_tablespace()) local_create_info.tablespace = nullptr;` |
| ENCRYPTION（源表已加密） | ✅ 保留 | — |
| **外键** | ❌ **丢弃** | 只传 `local_alter_info`，从不填 FK spec |
| **AUTO_INCREMENT 当前值** | ❌ **清零** | `local_create_info.auto_increment_value = 0;` |
| **DATA / INDEX DIRECTORY** | ❌ **丢弃** | `data_file_name = index_file_name = nullptr;` |
| ENCRYPTION（源表未加密） | ❌ 丢弃 | `if (... && !dd::is_encrypted(...)) encrypt_type = {nullptr, 0};` |

> ⚠️ **勘误**：**不存在 `dd::Table::clone()`**。所谓"复制定义"是 `mysql_prepare_alter_table()` 从 `TABLE_SHARE` + `dd::Table` **反向重建** `HA_CREATE_INFO`/`Alter_info`，然后 `dd::create_table()` **从头 new 一个 `dd::Table`**。

**坑 2：CREATE ... LIKE 的 binlog 四象限**

```cpp
    /*
           ==== ========= ========= ==============================
           Case    Target    Source Write to binary log
           ==== ========= ========= ==============================
           1       normal    normal Original statement
           2       normal temporary Generated statement
           3    temporary    normal Nothing
           4    temporary temporary Nothing
           ==== ========= ========= ==============================
    */
```

> ★ **Case 2 最反直觉**：目标普通表 + 源临时表 → 写的是 `store_create_info()` **生成的** CREATE TABLE 文本，且会强制 `used_fields |= HA_CREATE_USED_ENGINE`——因为源临时表在备库可能不存在，必须把 ENGINE 显式写出来，否则备库会落到别的引擎。

**坑 3：`sql_require_primary_key` 的三个豁免**（见"设计思想与权衡"）。

**坑 4：临时表的 `dd::Table` 永不落盘**

```cpp
std::unique_ptr<dd::Table> create_tmp_table(...) {
  std::unique_ptr<dd::Table> tab_obj(sch_obj.create_table(thd));
  tab_obj->set_is_temporary(true);      // ★ 与普通表唯一的结构性差异
  if (fill_dd_table_from_create_info(thd, tab_obj.get(), table_name,
                                     sch_obj.name(), create_info, create_fields,
                                     keyinfo, keys, keys_onoff, nullptr, 0,
                                     //                    ★ FK = nullptr, 0
                                     check_cons_spec, file))
    return nullptr;
  return tab_obj;
}
```

> ★ `mysql.tables` / `mysql.columns` 里**查不到任何 `CREATE TEMPORARY TABLE` 的行**。所有权交给 `TABLE_SHARE::tmp_table_def`（`table->s->tmp_table_def = tmp_table_ptr.release();`），随连接销毁。
> ★ 所以**临时表建表失败时在 SE 里直接删**（`rm_temporary_table()`），不走 `dd_client()->drop()`：

```cpp
  TABLE *table = open_table_uncached(thd, path, db, table_name, true, true,
                                     *tmp_table_ptr.get());
  if (!table) {
    (void)rm_temporary_table(thd, create_info->db_type, path,
                             tmp_table_ptr.get());
    return true;
  }
```

**坑 5：建表回滚不调 `ha_delete_table`**

⚠️ **勘误**：`create_impl()` 的 `cleanup:` **只做内存 cache 清理**，不调 `row_drop_table_for_mysql()`：

```cpp
cleanup:
  if (!info.is_intrinsic_temp_table() && info.is_temp_table()) {
    dict_sys_mutex_enter();
    dict_table_t *table = dict_table_check_if_in_cache_low(norm_name);
    if (table != nullptr) {
      for (dict_index_t *index = table->first_index(); index != nullptr;
           index = index->next()) {
        ut_ad(index->space == table->space);
        page_no_t root = index->page;
        index->page = FIL_NULL;
        dict_drop_temporary_table_index(index, root);   // ★ 回收 B-tree
      }
      dict_table_remove_from_cache(table);
    }
    dict_sys_mutex_exit();
  } else { /* intrinsic 表：unregister_table_handler */ }
```

> ★★ **落盘回滚靠 trx 回滚 + `log_ddl` DDL 日志**（`write_delete_space_log` / `write_remove_cache_log` / `write_free_tree_log`）。这也是为什么建表必须先 `store` dd::Table 到同一事务——**回滚时 DD 与 SE 一起回滚**。

**坑 6：`dict_table_add_to_cache(table, false)` —— 建表路径传 `false`**

```cpp
  else {
    dict_table_add_system_columns(table, temp_table_heap);
    dict_sys_mutex_enter();
    dict_table_add_to_cache(table, false);   // ★★ 不可被 LRU 淘汰
    dict_sys_mutex_exit();
  }
```

> ★ 新表在最终成功/失败前**不可被 LRU 淘汰**；成功后由 `detach()` → `dict_table_allow_eviction()` 放开：

```cpp
void create_table_info_t::detach() {
  ut_ad(!dict_sys_mutex_own());
  dict_sys_mutex_enter();
  ut_ad(m_table != nullptr);
  ut_ad(!m_table->can_be_evicted);
  ut_ad(!m_table->is_temporary());
  if (!m_table->explicitly_non_lru) {
    dict_table_allow_eviction(m_table);      // ★ 建表成功后才允许
  }
  if ((m_table->flags2 & (DICT_TF2_FTS | DICT_TF2_FTS_ADD_DOC_ID)) ||
      m_table->fts != nullptr) {
    fts_detach_aux_tables(m_table, true);
  }
  dict_sys_mutex_exit();
}
```

**坑 7：聚簇索引必须第一个建**

```cpp
  /* In InnoDB the clustered index must always be created first */
  if (primary_key_no != MAX_KEY) {
    error = create_index(m_trx, m_form, m_flags, m_table_name, primary_key_no,
                         dd_table);
    if (error) return error;
  }
  ...
  for (i = 0; i < m_form->s->keys; i++) {
    if (i != primary_key_no) {          // ★ 循环里跳过 PK（已建）
      error = create_index(m_trx, m_form, m_flags, m_table_name, i, dd_table);
      if (error) return error;
    }
  }
```

```cpp
// dict0dict.cc:2462 —— 底层断言
  ut_a(!index->is_clustered() || UT_LIST_GET_LEN(table->indexes) == 0);
```

**坑 8：`m_form->s->keys <= MAX_KEY` 的断言与 `MAX_KEY` 的真实身份**

```cpp
  DBUG_TRACE;
  assert(m_form->s->keys <= MAX_KEY);
```

> ⚠️ `MAX_KEY` 在 InnoDB 侧**仍然存在**（`create_table_info_t::create_table()` 的断言），但 server 侧的**校验**已全部改为 `file->max_keys()`。二者不一致时由 server 侧先挡住。

**坑 9：SRID 只能从 `dd::Table` 拿**

```cpp
    const dd::Column &col = dd_index->elements()[geom_col_idx]->column();
    has_srid = col.srs_id().has_value();
    srid = has_srid ? col.srs_id().value() : 0;
```

> ★ `TABLE*`/`KEY_PART_INFO` 里**没有 SRID**。这就是空间索引在建表时必须传 `dd::Table*` 的原因（普通索引不需要）。

**坑 10：CTAS 的 SELECT 列映射到新表尾部**

```cpp
  table_fields = table->field + (field_count - visible_select_exprs);
```

> ★ `CREATE TABLE t (a INT) SELECT b, c FROM s` → 新表列序是 `a, b, c`，且 `table_fields` 指向 `b`。`create_table_from_items()` 把 SELECT 的 `Create_field` **push_back 到 `alter_info->create_list` 尾部**决定了这一点。

| 坑 | 说明 |
|---|---|
| CREATE 后统计必然是 0 行 | `dict_stats_update(DICT_STATS_EMPTY_TABLE)`；COPY ALTER 继承此行为，靠 `innodb_stats_auto_recalc` / `ANALYZE TABLE` 补救 |
| 建树与写 DDL log 之间的崩溃窗口 | `dict_create_index_tree_in_mem` 的 FIXME：先建树后写 log，中间崩溃可能漏记刚分配的两个段（罕见，社区接受） |
| CREATE + file_per_table 的 `FREE_TREE_LOG` 是冗余的 | 第二个 FIXME：回滚时整 .ibd 会被删，不必逐个 free 索引树 |

### 术语对照

| 术语 | 含义 |
|---|---|
| intrinsic temp table | optimizer 内部用的临时表，经 `thd_to_innodb_session` 注册，无 FK 概念、可违反大小限制 |
| `dict_table_t` | InnoDB 内存字典中的表对象；与 server 的 `dd::Table` 是两层独立元数据，靠 `create_table_update_global_dd` 对齐 |
| 段（segment） | 每个索引有 LEAF / TOP 两个文件段，`btr_create` 时创建 |

---

## 参考

**相关文档**
- 上层骨架（三种算法判定、MDL 三段式、DDL 观测）见 [`ddl.md`](ddl.md)
- 原子 DDL（DDL log 类型、post-DDL replay、崩溃恢复闭环）见 [`atomic_ddl.md`](atomic_ddl.md)
- ALTER / DROP / TRUNCATE 见 [`alter_table.md`](alter_table.md) 与 [`ddl.md`](ddl.md)
- 在线 DDL 工具（gh-ost / pt-osc）见 [`online_ddl_tools.md`](online_ddl_tools.md)

> 本篇由 `ddl.md` 的 CREATE TABLE 章迁出并补齐引擎侧而成；原篇的 WL / bug 参考仍保留在 [`ddl.md`](ddl.md)「参考」章，不在此重复。
