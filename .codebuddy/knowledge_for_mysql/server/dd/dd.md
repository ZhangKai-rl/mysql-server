# 8.0 数据字典（DD）

> **定位**：MySQL 8.0 的**权威元数据**——存在哪、长什么样、怎么被缓存、怎么参与自举与原子 DDL。
> **边界**：InnoDB 侧的内存字典（`dict_table_t` 等）见 [`innodb_dict.md`](innodb_dict.md)；表对象生命周期见 [`../table.md`](../table.md)；DDL 执行见 [`../../feat/ddl/ddl.md`](../../feat/ddl/ddl.md)。
> **★ 与 [`system_views.md`](system_views.md) 的分工**（两篇互补、不重叠）：本篇讲**元数据怎么存**（DD 表体系 / 对象模型 / 缓存 / SDI / 自举 / 升级）；那篇讲**元数据怎么暴露出去**——43 张 `INFORMATION_SCHEMA` 系统视图（C++ builder 拼 `CREATE VIEW`、注册与版本、各视图的定义细节、`SHOW` 重写）。本篇只在"消费方"处顺带提及视图，不展开其机制。

> **★ 两套并存的字典**（理解元数据问题的关键前提，也常被混淆）：
>
> | | server 层 DD（本篇） | InnoDB 内部字典（[`innodb_dict.md`](innodb_dict.md)） |
> |---|---|---|
> | 是什么 | 8.0 起的**全局唯一权威**元数据（取代 .frm） | 引擎私有的运行时字典 |
> | 载体 | `mysql` 库下的 InnoDB 表（`mysql.tables` / `mysql.columns` / …） | `dict_sys_t` 内存结构 + `SYS_*` 内部表 |
> | 持久化 | 存在 DD 表里（本身就是 InnoDB 表） | 崩溃后**从 DD 重建**，不独立持久化 |
> | 关键对象 | `dd::Table` / `dd::Column` / `dd::Index` | `dict_table_t` / `dict_index_t` / `dict_col_t` |
> | 谁写它 | DDL | 引擎加载（`dict0dd.cc` 读 DD 填内存） |
>
> **单向权威**：DD 是唯一真相，InnoDB dict 是它的**内存投影/缓存**。所以 8.0 里"字典不一致"类问题的排查方向总是"DD 对不对 → 加载逻辑对不对"，而不是反过来。

## 目录

- [概述](#概述)（★ 全景总览图）
- [理论基础](#理论基础)
  - [设计思想与权衡](#设计思想与权衡)（双轨体系 / 虚继承 / ★ Tables vs Table / 继承多态 / 树形结构 / 为何进 InnoDB / Object_table）
  - [算法与数据结构](#算法与数据结构) / [他库对比与演进动机](#他库对比与演进动机)
- [核心实现](#核心实现)
  - [1. DD 表体系与存储](#1-dd-表体系与存储)（30 张逐张用途 / 全部在 mysql.ibd / 为何不能 DML）
  - [2. 对象模型与 `se_private_data`](#2-对象模型与-se_private_data)（★ options vs se_private_data / ★ 全键名表）
    - [2.2 `dd::Properties` 与它的实现](#22-ddproperties不改表结构就能加属性)
    - [2.6 ★ `dd_table.cc`：建表信息 → `dd::Table` 的往返](#26--dd_tableccsql-层建表信息--ddtable-的往返)（★ 对象树工厂不写 DD / 入口函数族 / 7 个 `fill_dd_*` / 类型转换族 / ★ 主键提升陷阱 / ★ `se_private_data` 不在这里填）
    - [2.7 ★ `fill_column_from_dd()`：`dd::Column` → `Field`](#27--fill_column_from_ddddcolumn--field)（★ 一等属性 / options 属性 / 派生属性三档对照表 / 默认值"二进制镜像" / 两遍扫描 / 反向 Field→DD）
  - [3. DD cache](#3-dd-cache)（Dictionary_client / RAII / 三 registry / Shared_dictionary_cache / COW）
  - [4. SDI](#4-sdiserialized-dictionary-information)（⚠️ **已独立成篇** → [`feat/sdi.md`](../../feat/sdi.md)，横跨 server ↔ InnoDB 两层；本篇只留"与 DD 的关系"）
  - [5. 启动自举](#5-启动自举鸡生蛋问题) / [6. 原子 DDL 中的地位](#6-原子-ddl-中-dd-的地位)
  - [7. 升级](#7-升级)
    - [7.4 升级前体检](#74--升级前体检do_server_upgrade_checks) / [7.5 错误处理三件套](#75-错误处理三件套) / [7.6 取代 mysql_upgrade 客户端](#76--upgrade_system_schemasserver-端取代-mysql_upgrade-客户端)
    - [7.7 ★ frm 迁移：复用建表路径](#77--57--80upgrade_57tablecc-怎么把-frm-变成-dd-对象) / [7.8 routine / event / schema 迁移](#78-routine--event--schema-的迁移)
- [★ DD 里的 C++ 运用](#-dd-里的-c-运用)
- [Misc](#misc)（易误解概念 / 观察技巧 / 待补清单）
- [参考](#参考)

---

## 概述

### 全景总览图

一图看懂 DD 的四层结构、对象体系与数据流向：

```
╔══════════════════════════════════════════════════════════════════════════╗
║ ① 存储层  mysql.ibd（space_id = 0xFFFFFFFE，页 0 起 + SDI 索引）           ║
║                                                                          ║
║   30 张 MySQL DD 表（INERT 1 + CORE 22 + SECOND 7）                       ║
║     mysql.tables / columns / indexes / schemata / tablespaces / ...       ║
║   5 张 InnoDB 自有表                                                      ║
║     dd_properties / innodb_dynamic_metadata / innodb_table_stats /        ║
║     innodb_index_stats / innodb_ddl_log                                   ║
╚═══════════════════════════════╤══════════════════════════════════════════╝
                                │ 读写（attachable tx / 内部 SQL / mtr）
                                ▼
╔══════════════════════════════════════════════════════════════════════════╗
║ ② 对象层  接口/实现双轨（Bridge）+ 树形描述                                 ║
║                                                                          ║
║   【接口 types/】        【实现 impl/types/】                              ║
║   Weak_object            Weak_object_impl                                 ║
║     └ Entity_object ──────► Entity_object_impl   (id / name / is_persist) ║
║          └ Abstract_table ► Abstract_table_impl  (schema / columns)       ║
║               ├─ Table ───► Table_impl  (indexes/fk/partition/trigger)    ║
║               └─ View ────► View_impl   (definition/algorithm)            ║
║                                                                          ║
║   树形（一张表 = 一棵树，子树对应一张 DD 表）：                              ║
║   Table ─┬─ Column(s) ── Column_type_element(s)        → mysql.columns   ║
║          ├─ Index(s) ─── Index_element(s)              → mysql.indexes   ║
║          ├─ Foreign_key(s) ─ Foreign_key_element(s)    → mysql.foreign_keys║
║          ├─ Partition(s) ─ Partition_value(s)          → mysql.table_partitions║
║          ├─ Trigger(s)                                 → mysql.triggers  ║
║          └─ Check_constraint(s)                        → mysql.check_constraints║
╚═══════════════════════════╤══════════════════════════════════════════════╝
                            │ restore_children / store_children 递归遍历
                            ▼
╔══════════════════════════════════════════════════════════════════════════╗
║ ③ 缓存层（两级）                                                           ║
║                                                                          ║
║   Dictionary_client（每 THD 一个，唯一入口）                              ║
║     ├─ m_registry_committed    / uncommitted / dropped  ← 私有写集(MVCC)  ║
║     └─ Auto_releaser（RAII 栈）—— 管"借来的"共享对象                       ║
║   Shared_dictionary_cache（全局，按 Cache_partition 分 11 个 map）         ║
║     Abstract_table(max_connections) / Schema / Tablespace / Routine / ... ║
╚═══════════════════════════╤══════════════════════════════════════════════╝
                            │ 仅 miss 时才下钻
                            ▼
                     读 DD 表（Storage_adapter → attachable tx → B+ 树）
```

### 是什么

8.0 用**事务化的 InnoDB 表**存元数据，取代 5.7 的 `.frm` + MyISAM 系统表。这些 DD 表位于 `mysql` 库、本身就是 InnoDB 表。

### 用途

- DDL 的唯一写入目标（DDL = 改 DD 表 + 改引擎 + 记 binlog，三者原子）；
- 优化器 / 开表 / 权限 / 复制的元数据来源；
- `information_schema` 现在是**DD 表上的视图**（不再是独立实现）。

### 版本演进

| 版本 | 变化 |
|---|---|
| 5.7 及以前 | `.frm` + MyISAM 系统表；DDL 非原子 |
| 8.0 | **事务化 DD**；原子 DDL；I_S 视图化；每张表还带一份 SDI 副本 |

---

## 理论基础

### 设计思想与权衡

#### 一、★ 接口/实现双轨体系（Bridge 的变体）——为什么要两套类

翻开 `sql/dd/` 会发现**每个概念都有两个类**：`dd::Table` 与 `dd::Table_impl`、`dd::Index` 与 `dd::Index_impl`。这不是冗余，而是一套刻意的隔离设计：

```
sql/dd/types/table.h            dd::Table        ← 纯接口（全虚函数，无数据成员）
sql/dd/impl/types/table_impl.h  dd::Table_impl   ← 实现（有数据成员，可序列化）
```

上层代码（server 各模块、InnoDB）**只认 `dd::Table&`**，永远拿不到 `Table_impl`。这条边界由一处精巧的机制强制：

```cpp
class Entity_object : virtual public Weak_object {
 public:
  virtual Object_id id() const = 0;
  virtual bool is_persistent() const = 0;
  virtual const String_type &name() const = 0;
  ...
 private:
  virtual class Entity_object_impl *impl() = 0;          // ★ 私有纯虚
  virtual const class Entity_object_impl *impl() const = 0;
  friend class cache::Storage_adapter;                    // ★ 只有这两个友元能穿透
  friend class Entity_object_table_impl;
};
```

★ `impl()` 是 **private 虚函数，且只把 `Storage_adapter`（持久化层）和 `Entity_object_table_impl` 设为友元**——意味着：

- 普通调用方拿到 `dd::Table&`，**编译期就无法访问实现对象**；
- 只有"要把对象写进 DD 表 / 从 DD 表读出来"的持久化层能穿透到 impl。

这是"接口隔离 + 受控穿透"：既让 DD 的内部表示可以自由重构（只要不改接口），又给持久化这个特定场景留了后门。**被放弃的方案**是"单套类 + 全部公有"——那样任何模块都能直接改 `Table_impl` 的字段，DD 的不变量（如"改了 name 要同步 registry 的 key"）就守不住了。

**代价**：每加一个属性要在接口和实现两处改；调试时看到的是 `dd::Table&`，要看真实字段得转到 impl；且**所有访问都走虚函数**，DD 对象访问因此有一次间接跳转开销（相对 DD 读取本身涉及的文件 IO，这个开销可忽略）。

#### 二、★ 为什么到处是 `virtual public`（虚继承）

`Table_impl` 的声明长这样：

```cpp
class Table_impl : public Abstract_table_impl, virtual public Table { ... };
class Abstract_table_impl : public Entity_object_impl, virtual public Abstract_table { ... };
class Entity_object_impl : virtual public Entity_object, public Weak_object_impl { ... };
```

这是典型的**菱形继承**：`Table_impl` → {`Abstract_table_impl` → `Entity_object_impl`} 和 {`Abstract_table` → `Entity_object`}，两条路径都会到达 `Entity_object`。

```
        Entity_object (虚基类)
         ╱            ╲
  Entity_object_impl  Abstract_table
        ╲             ╱
    Abstract_table_impl ── Table (virtual public)
              ╲            ╱
              Table_impl
```

**不用虚继承** → `Table_impl` 里会有**两份** `Entity_object` 子对象，`id()` 调用二义、且 `impl()` 的友元穿透会指向错误的那个。所以**接口一侧全部用 `virtual public`**——这是这套双轨设计的必然结果，不是随意加的。

#### 三、★★ `dd::tables::Tables`（元数据表）vs `dd::Table`（元数据行）

先给**三大部分**的全景（每个 DD 概念在代码里出现三次）：

| 部分 | 命名空间 / 目录 | 是什么 | 例子 |
|---|---|---|---|
| ① **DD tables** | `dd::tables::`（`sql/dd/impl/tables/`） | **mysql 库下的物理表**——建表 DDL、字段定义、索引定义 | `dd::tables::Tables`（mysql.tables 的 schema） |
| ② **DD object 接口** | `dd::`（`sql/dd/types/`） | **用户表/视图的元数据**——外部接口（纯虚） | `dd::Table`（一张用户表） |
| ③ **DD object impl** | `dd::`（`sql/dd/impl/types/`） | ② 的实现，**store/restore/serialize 都定义在这里** | `dd::Table_impl` |

三者的关系由两个 typedef 静态绑定：`dd::Table::Impl = Table_impl`（②→③）、`dd::Abstract_table::DD_table = tables::Tables`（②→①）。运行时的流向（对照 DD Cache）：

```
Dictionary_client（会话级）→ Shared_dictionary_cache（共享级）→ Storage_adapter
      cache miss 时 ↓ 按 key 读                          ↓ store/restore（持久化配器）
① DD tables（mysql.tables 等物理表）◄──── ORM 式读写 ────► ③ DD object_impl
                                                （② 接口是外部世界唯一看到的形态）
```

##### ★★ `Entity_object_table` vs `Object_table`：根表的本质是"工厂"

上一轮我们说"`Entity_object_table` 派生类是描述树根"——**判据是什么**？看接口定义（`types/entity_object_table.h:49`）：

```cpp
class Entity_object_table : virtual public Object_table {
 public:
  virtual Entity_object *create_entity_object(const Raw_record &record) const = 0;
  virtual bool restore_object_from_record(Open_dictionary_tables_ctx *,
                                          const Raw_record &record,
                                          Entity_object **o) const = 0;
};
```

★ **`create_entity_object` 是工厂方法**：`mysql.tables`（`dd::tables::Tables`）实现了它——**从一行记录能实例化出一个 `dd::Table` 对象**。这就是"根表"的准确含义：**能独立产出 Entity_object 的表**。

`Object_table_impl`（`impl/types/object_table_impl.h:36`）则持有建表与版本的状态：

```cpp
class Object_table_impl : virtual public Object_table {
 protected:
  mutable uint m_last_dd_version;
  Object_table_definition_impl m_target_def;    // ★ 目标 DDL builder（add_field 编译期填好）
  mutable bool m_actual_present;
  mutable Object_table_definition_impl m_actual_def;   // ★ 升级期的"实际定义"（旧版本结构）
  bool m_hidden;                                 // ★ 这张 DD 表对 SQL 隐藏
  enum class Common_option { ENGINE, CHARSET, COLLATION, ... };  // 所有 DD 表共享的建表选项
};
```

★ **`m_target_def` vs `m_actual_def` 双定义**是升级的关键：`target` 是当前代码认识的最新结构（用于 `--initialize`），`actual` 是**磁盘上实际存在的旧结构**（升级时先按 actual 读数据）。`create_target_table` 用前者，升级中读旧表用后者——一套类同时服务"新建"与"升级读旧"。

最终实现类 `Entity_object_table_impl` 同时继承两者，有一处值得注意的注释：

```cpp
// Fix "inherits ... via dominance" warnings
const String_type &name() const override { return Object_table_impl::name(); }
```

★ **"via dominance"**：`Object_table_impl` 和 `Entity_object_table` 都（经由虚继承）继承自 `Object_table`，`name()` 在两条路径都有定义——C++ 用**支配规则（dominance rule）**消除二义，但编译器仍可能警告，于是显式覆写指定走哪条。这是虚继承菱形（第（二）节）在 DD 表侧的镜像。

##### `Entity_object` vs `Weak_object`（辨析表）

| | `Entity_object` | `Weak_object` |
|---|---|---|
| 契约 | `id()` / `name()` / `is_persistent()` | 只有 `debug_print()` |
| 身份 | 有独立主键 id，**可独立存在于 DD cache** | 无 id，**必须依附某 Entity_object** |
| 实现基类 | `Entity_object_impl`（持 `m_id`） | `Weak_object_impl_`（仅 new/delete 钩子） |
| 例子 | Table / View / Index / Column / Partition / Schema / Tablespace / Event / Routine | Index_element / Foreign_key_element / Column_type_element / Partition_value / Tablespace_file |
| 缓存 | 有 `Cache_partition`，可进 `Shared_dictionary_cache` | 随父对象加载，不单独缓存 |
| restore_children | 内部节点 override（有子集合） | **一律用基类空实现**（永远没有子节点） |

判断口诀：**问"这个对象单独存在有意义吗"**——`Index_element`（索引引用了哪一列）离开索引无意义 → Weak；`Index` 有自己的 id、能被 `acquire(id)` 单独查出来 → Entity。

这是 DD 里**最容易混淆的一对概念**，必须分清：

| | `dd::tables::Tables` | `dd::Table` |
|---|---|---|
| 是什么 | **mysql.tables 这张物理表本身** | **mysql.tables 里的一行** |
| 类比 | 一张 Excel 表（有列头、有结构） | 表里的一行数据 |
| 基类 | `dd::Object_table` | `dd::Abstract_table`（→ `Entity_object`） |
| 所在命名空间 | `dd::tables::`（`impl/tables/tables.h`） | `dd::`（`types/table.h`） |
| 数量 | **一个** | **每张用户表一个** |
| 谁定义它的结构 | C++ 硬编码的 `Object_table_definition` | 由 `CREATE TABLE` 语句决定 |

```cpp
// ① 元数据"表"：mysql.tables 自身的定义（几个字段、哪些索引）
namespace dd { namespace tables {
class Tables : public Object_table { ... };     // impl/tables/tables.h
}}

// ② 元数据"行"：一张用户表的元数据
namespace dd {
class Table : virtual public Abstract_table { ... };   // types/table.h
}
```

源码里甚至专门加了防混淆的注释（`types/index.h:51`）：

```cpp
// 区别dd::Indexes系统元数据表，这是用户表元数据
class Index : virtual public Entity_object { ... };
```

##### ★ `DD_table` typedef：每个对象**编译期绑定**到它的根系统表

每个 DD 对象接口类内部都有一个 `DD_table` typedef，指明"我存在哪张 DD 表里"。这是理解 DD 组织方式最快的一张表：

| DD 对象（行） | `DD_table` typedef | 物理表 |
|---|---|---|
| `Abstract_table`（`Table`/`View` 的基类） | `tables::Tables` | `mysql.tables` |
| `Table` / `View` | 继承自 `Abstract_table` | `mysql.tables` |
| `Column` | `tables::Columns` | `mysql.columns` |
| `Column_type_element` | `tables::Column_type_elements` | `mysql.column_type_elements` |
| `Index` | `tables::Indexes` | `mysql.indexes` |
| `Index_element` | `tables::Index_column_usage` | `mysql.index_column_usage` |
| `Foreign_key` | `tables::Foreign_keys` | `mysql.foreign_keys` |
| `Foreign_key_element` | `tables::Foreign_key_column_usage` | `mysql.foreign_key_column_usage` |
| `Partition` | `tables::Table_partitions` | `mysql.table_partitions` |
| `Partition_value` | `tables::Table_partition_values` | `mysql.table_partition_values` |
| `Tablespace` | `tables::Tablespaces` | `mysql.tablespaces` |
| `Tablespace_file` | `tables::Tablespace_files` | `mysql.tablespace_files` |
| `Check_constraint` | `tables::Check_constraints` | `mysql.check_constraints` |
| `Trigger` | `tables::Triggers` | `mysql.triggers` |
| `Event` | `tables::Events` | `mysql.events` |
| `View_routine` | `tables::View_routine_usage` | `mysql.view_routine_usage` |
| `Index_stat` | `tables::Index_stats` | `mysql.index_stats` |
| `Collation` | `tables::Collations` | `mysql.collations` |
| `Spatial_reference_system` | `tables::Spatial_reference_systems` | `mysql.spatial_reference_systems` |

★ 注意 `Table` 和 `View` **没有自己的 `DD_table`**——它们共用基类 `Abstract_table` 的 `tables::Tables`。这解释了一个常见疑问：**为什么 `mysql.tables` 里既有普通表又有视图**——因为它们在 DD 里是同一张表的两种 `type`（`enum_table_type::BASE_TABLE` / `VIEW`）。

##### ★ 另外三个 typedef：Impl / Cache_partition / Key

每个 DD 对象类顶部那串 typedef **不是样板代码**，而是把四类信息绑到类型上：

```cpp
class Event : virtual public Entity_object {
  typedef Event_impl        Impl;              // ① 实现类（双轨的"实现"一侧）
  typedef Event             Cache_partition;   // ② 用哪个 Shared_multi_map
  typedef tables::Events    DD_table;          // ③ 根系统表
  typedef Primary_id_key    Id_key;            // ④ 缓存的 id 索引
  typedef Global_name_key   Name_key;          // ④ 缓存的 name 索引
};
```

★② `Cache_partition` 决定"哪些类型共用同一个缓存 map"。`Abstract_table::Cache_partition = Abstract_table`——意味着**表与视图共用 `m_map<Abstract_table>()` 这一个缓存**（这正是 3.3 节里那个容量为 `max_connections` 的 map 同时装表和视图的原因）。

★④ 三种 key 的语义差别（对应 `Object_registry` 里的不同哈希）：

| key 类型 | 含义 | 例子 |
|---|---|---|
| `Global_name_key` | **全局唯一**的名字 | schema 名、tablespace 名 |
| `Item_name_key` | **依附于父对象**的名字（需 (parent_id, name)） | 列名、索引名 |
| `Primary_id_key` | 主键 id | 通用 |
| `Se_private_id_key` | 引擎私有 id（`se_private_id`） | InnoDB 按 table_id 反查 |
| `Composite_4char_key` | 复合键 | `Index_stat` 的 (schema, table, index, column) |

##### 完整的继承体系

```
Weak_object                      ← 最顶层：只要求 debug_print()（无 id、无名字）
├── Entity_object                ← 有 id + name + is_persistent()
│   ├── Abstract_table ── Table / View        (mysql.tables)
│   ├── Schema / Catalog / Tablespace
│   ├── Index / Foreign_key / Check_constraint / Trigger
│   ├── Column / Partition / Partition_index
│   ├── Event / Routine / Parameter
│   ├── Charset / Collation
│   ├── Resource_group / Spatial_reference_system
│   ├── Column_statistics / Index_stat / Table_stat
│   └── ...
└── Weak_object 直系子对象（无 id，依附父对象存在）
    ├── Index_element          (mysql.index_column_usage)
    ├── Foreign_key_element    (mysql.foreign_key_column_usage)
    ├── Column_type_element    (mysql.column_type_elements)
    ├── Partition_value        (mysql.table_partition_values)
    ├── Tablespace_file        (mysql.tablespace_files)
    ├── View_routine / View_table_usage
    └── Parameter_type_element
```

★ **分支规则**：有独立身份（能按 id 单独查出来）的挂 `Entity_object`；**只是父对象的一个组成部分**（离开父对象无意义）的挂 `Weak_object`。比如 `Index_element`（索引引用了哪一列）不能脱离索引存在，所以是 `Weak_object`；而 `Index` 本身有 id，是 `Entity_object`。

这条规则也决定了缓存粒度：**只有 `Entity_object` 才有 `Cache_partition`、才能进 `Shared_dictionary_cache`**；`Weak_object` 随父对象一起加载/存储。

##### 接口层逐层加职责（以 `Table` 这条链为例）

三层各管一段，像洋葱：

| 层 | 新增的字段 / 能力 |
|---|---|
| `Entity_object` | `id()` / `name()` / `is_persistent()` |
| `Abstract_table` | `schema_id` / `options` / `type` / `hidden` / ★ **`columns` 集合** |
| `Table` | `engine` / `row_format` / `se_private_id` / `se_private_data` / ★ **`indexes` / `foreign_keys` / `partitions` / `triggers` / `check_constraints`** |
| `View`（`Table` 的兄弟，非子类） | `definition` / `check_option` / `algorithm` / `definer` / `view_tables` / `view_routines` |

★ **`Table` 与 `View` 是同一张 `mysql.tables` 的两种 `type`，不是两个平级类**：

```cpp
enum class enum_table_type { INVALID_TABLE, BASE_TABLE, USER_VIEW, SYSTEM_VIEW };
```

`View` 又分 `USER_VIEW`（用户视图）和 `SYSTEM_VIEW`（I_S 的系统视图）——**系统视图也是 DD 对象**。

★ **`hidden` 四类**（`Abstract_table::enum_hidden_type`）：

```cpp
HT_VISIBLE        // 普通用户表
HT_HIDDEN_SYSTEM  // ★ DD 表自身（mysql.tables 里 hidden=SYSTEM 的那一行）
HT_HIDDEN_SE      // SE 隐式创建的表（如 InnoDB 的 FTS 辅助表）
HT_HIDDEN_DDL     // ALTER TABLE 的临时中间表
```

这回答了"DD 表自己是不是也躺在 mysql.tables 里"——**是，且以 `HT_HIDDEN_SYSTEM` 标记隐藏**（第 Misc 节的 `skip_dd_table_access_check` 观察技巧就是看这些）。

##### 实现层对称继承 + 虚函数的逐层调用

接口侧三层，实现侧也三层一一对应：

```
Entity_object    ←──  Entity_object_impl        （持有 m_id，set_id 时 fix_has_new_primary_key）
Abstract_table   ←──  Abstract_table_impl       （columns 容器 + options 存储）
Table / View     ←──  Table_impl / View_impl    （各自字段 + restore/store 实现）
```

`restore_attributes()` / `restore_children()` / `store_attributes()` / `store_children()` 这组虚函数就是在这条实现链上**逐层调用**：`Table_impl::restore_children()` 先调 `Abstract_table_impl::restore_children()`（读列），再读自己的索引 / 外键 / 分区——**每一层只负责自己加的那一段**，这是模板方法（Template Method）模式。

单个 `restore_attributes()` 长什么样，值得看一个具体实例（所有 entity 都遵循同一套写法）：

```cpp
// impl/types/parameter_type_element_impl.cc:87
bool Parameter_type_element_impl::restore_attributes(const Raw_record &r) {
  if (check_parent_consistency(
          m_parameter,
          r.read_ref_id(Parameter_type_elements::FIELD_PARAMETER_ID)))   // ① 父一致性
    return true;

  m_index = r.read_uint(Parameter_type_elements::FIELD_ELEMENT_INDEX);   // ② 逐字段读
  m_name = r.read_str(Parameter_type_elements::FIELD_NAME);

  return false;
}

// 反向：store_attributes（:101）
bool Parameter_type_element_impl::store_attributes(Raw_record *r) {
  return r->store(Parameter_type_elements::FIELD_PARAMETER_ID,
                  m_parameter->id()) ||
         r->store(Parameter_type_elements::FIELD_ELEMENT_INDEX, m_index) ||
         r->store(Parameter_type_elements::FIELD_NAME, m_name);
}
```

三点构成 DD 序列化的统一约定：

1. ★ **`check_parent_consistency(m_parameter, r.read_ref_id(FIELD_PARENT_ID))` 打头**——先从 `Raw_record` 读出父对象 ID，再与**内存中已缓存的父对象**比对；不一致直接返回 true（失败）。这一步防的是"从磁盘读到的记录挂在了错误的父对象下"（缓存与字典不一致、或并发 DDL 竞态），属于 DD 层的一致性自检。
2. **字段读写都用 `FIELD_*` 枚举下标**（`r.read_uint(FIELD_ELEMENT_INDEX)` / `r.read_str(FIELD_NAME)`）——字段名不出现在代码里，只有枚举序号；所以**改 `Object_table_definition` 的字段顺序必须同步改这些枚举**，否则读错位。
3. **`store_attributes` 用 `||` 链式短路**——任一 `store` 失败即整体失败；`restore` 侧则是"先校验、再顺序赋值"。

⇒ 四件套的分工也就清楚了：`attributes` 管**本层自己的标量字段**，`children` 管**下层对象集合**（并触发递归）。每个 entity_impl 只需实现自己那两个，组合起来就是整棵对象树的读/写。

##### ★ 多态的三个具体实例

**① `update_aux_key()`——`Table` 有 se_private_id 索引，`View` 没有**：

```cpp
// Abstract_table：默认空实现
virtual bool update_aux_key(Aux_key *) const { return true; }

// Table：覆写——se_private_id key 需要 (engine, se_private_id) 二元组
bool update_aux_key(Aux_key *key) const override {
  return update_aux_key(key, engine(), se_private_id());
}
```

`View` **不覆写**——视图没有引擎私有 id，aux key 索引对 View 无意义。**同一个接口，子类按需覆写**，这是多态最标准的用法。

**② `type()` 驱动 `is_system_view()`**：`View::is_system_view()` 是 non-virtual，直接 `return type() == SYSTEM_VIEW`——复用虚的 `type()` 而不是存一个 bool 字段（单一事实来源）。

**③ `Table::table()` dummy method**（`table.h:232-235`）：

> Dummy method to be able to use Partition and Table interchangeably in templates.

让模板代码能对 `Table` 和 `Partition` 统一调 `.table()` 拿到宿主表。

##### ★ 树形结构：`Collection` 与所有权

`dd::Table` 是一棵树的根，子节点是各种 `Collection`：

```
dd::Table
├── Column_collection          columns()         ← Abstract_table 层
├── Index_collection           indexes()
├── Foreign_key_collection     foreign_keys()
├── Partition_collection       partitions()      ← ★ 拥有 Partition*
│     └── Partition_leaf_vector  leaf_partitions()  ← 不拥有，指向 collection 元素
├── Trigger_collection         triggers()
└── Check_constraint_collection check_constraints()
```

★ **所有权语义**（`table.h:58-65` 注释）：`Partition_collection` **拥有** `Partition*`（collection 析构时 delete）；`Partition_leaf_vector` **不拥有**（只是指向 collection 里的元素）。`leaf_partitions()` 只列**叶子**分区——无子分区时 = 所有分区，有子分区时只列真正物理存在的子分区。

★ `foreign_key_parents` 特殊：它不是持久化字段，而是"**引用本表**的外键父表列表"，构造 Table 时从 DD 算出来，可 `reload_foreign_key_parents()` 按需重载。

##### 树与"描述树"的递归遍历

这就是第（四）节"描述树"的具体形态：**每张 DD 表对应树里的一层**。

```
restore_children(Table_impl)：
  1. restore Column 子树       → 读 mysql.columns（+ column_type_elements）
  2. restore Index 子树        → 读 mysql.indexes（+ index_column_usage）
  3. restore Foreign_key 子树  → 读 mysql.foreign_keys（+ foreign_key_column_usage）
  4. restore Partition 子树    → 读 mysql.table_partitions（+ table_partition_values）
  ...
```

加载一张表 = **深度优先遍历这棵树**，每访问一个子节点查一次对应的 DD 表。这就是为什么"打开表"要读多张 DD 表，也是为什么必须有缓存（否则每次开表都遍历整棵树）。`store_children()` 是同一棵树的逆序写回。

#### 四、★★ 描述树是 Composite：`attribute` / `children` 的存取机制

DD object 是树形递归结构，而且这棵树是**标准的 Composite 模式**：

```
dd::Table（根系统表 mysql.tables）
  ├── dd::Column      × N    （mysql.columns）
  │     └── dd::Column_type_element × N  （mysql.column_type_elements）
  ├── dd::Index       × N    （mysql.indexes）
  │     └── dd::Index_element × N        （mysql.index_column_usage）
  ├── dd::Foreign_key × N
  ├── dd::Partition   × N    （mysql.table_partitions）
  │     ├── dd::Partition_index × N
  │     └── dd::Partition_value × N
  └── dd::Check_constraint / dd::Trigger × N
```

**Composite 的体现**：`restore_attributes` 对应树节点**自身的数据**，`restore_children` 对应**子树**。四个虚函数两两对称：

| 方向 | attribute（节点自身） | children（子树） |
|---|---|---|
| **读（DD 表 → 对象）** | `restore_attributes(Raw_record)` 把一行解析填回对象 | `restore_children(otx)` 递归加载所有子集合 |
| **写（对象 → DD 表）** | `store_attributes()` 把对象字段拼成一行插入系统表 | `store_children(otx)` 递归写所有子集合 |

★ **只有拥有子对象的节点才 override `restore_children`/`store_children`；叶子节点用基类默认空实现**（`{ return false; }`）。这是 Composite 的经典结构——遍历逻辑对内部节点与叶子节点统一。

##### 完整的 `Table_impl::restore_children`（`table_impl.cc:303`）

```cpp
bool Table_impl::restore_children(Open_dictionary_tables_ctx *otx) {
  // NOTE: the order of restoring collections is important because:
  //   - Index-objects reference Column-objects
  //     (thus, Column-objects must be loaded before Index-objects).
  //   - Foreign_key-objects reference both Index-objects and Column-objects.
  //     (thus, both Indexes and Columns must be loaded before FKs).
  //   - Partitions should be loaded at the end, as it refers to indexes.
  bool skip_check_constraints =
      (bootstrap::DD_bootstrap_ctx::instance().is_dd_upgrade_from_before(
          bootstrap::DD_VERSION_80016));   // ★ check constraint 是 8.0.16 引入的

  return (
      Abstract_table_impl::restore_children(otx) ||       // ① columns
      m_indexes.restore_items(this, otx, otx->get_table<Index>(),
                              Indexes::create_key_by_table_id(this->id())) ||
      m_foreign_keys.restore_items(...) ||                 // ② 引用 ①③
      m_partitions.restore_items(...) ||                   // ③ 引用索引，最后
      m_triggers.restore_items(...) ||
      (!skip_check_constraints && m_check_constraints.restore_items(...)));
}
```

★ **加载顺序 = 依赖顺序**：Column（被 Index 引用）→ Index（被 FK/Partition 引用）→ FK/Partition。反序加载会拿到悬空指针。

★ `||` 短路是**错误短路**：任何一个子集合加载失败（返回 true）就中止——失败按"未发生"处理，整体回滚。

★ `restore_items()` 内部就是"初始化 cursor 扫描 B-Tree"——对每行 `restore_attributes()` + 递归 `restore_children()`。

##### 对应的 store / drop（顺序同样重要）

```cpp
bool Table_impl::store_children(Open_dictionary_tables_ctx *otx) {
  // Note that indexes has to be stored first, as partitions refer indexes.
  return Abstract_table_impl::store_children(otx) ||
         m_indexes.store_items(otx) || m_foreign_keys.store_items(otx) ||
         m_partitions.store_items(otx) || store_triggers(otx) ||
         (!skip_check_constraints && m_check_constraints.store_items(otx));
}

bool Table_impl::drop_children(Open_dictionary_tables_ctx *otx) const {
  // Note that partition collection has to be dropped first
  // as it has foreign key to indexes.
  return m_check_constraints.drop_items(...) ||   // ★ 删除顺序与存储相反！
         m_triggers.drop_items(...) ||
         m_partitions.drop_items(...) ||          // 先删引用者
         m_indexes.drop_items(...) ...;
}
```

★ **drop 顺序与 store 顺序相反**——store 先写被引用者（indexes），drop 先删引用者（partitions）——因为 DD 表之间有物理外键，违背顺序会触发外键冲突。

##### attribute 的两面

```cpp
bool Abstract_table_impl::restore_attributes(const Raw_record &r) {
  restore_id(r, Tables::FIELD_ID);
  restore_name(r, Tables::FIELD_NAME);
  m_created = r.read_int(Tables::FIELD_CREATED);
  m_last_altered = r.read_int(Tables::FIELD_LAST_ALTERED);
  m_hidden = static_cast<enum_hidden_type>(r.read_int(Tables::FIELD_HIDDEN));
  m_schema_id = r.read_ref_id(Tables::FIELD_SCHEMA_ID);
  ...
}
```

`store_attributes` 是它的镜像：对象字段 → 一行 → `Raw_record::update` 走 InnoDB 接口持久化。一句话概括：

> **attribute 操作父节点**（读写本对象对应的 DD 表那一行）；**children 操作子节点**（读写子对象对应的从属表）。

##### 描述树的"根 vs 叶子"（`Entity_object_table` 与 `Object_table_impl` 的分野）

- 从 **`Entity_object_table`** 派生的类（`Tables`、`Schemata`、`Tablespaces`）**有描述树、是根节点**；
- 从 **`Object_table_impl`** 派生的类（`Columns`、`Indexes`、`Table_partitions`）**只是中间/叶子节点**，挂在某棵描述树上。

即：**不是每张 DD 表都是树根**——`mysql.columns` 自己不是一棵树，它是 `mysql.tables` 这棵树的子树。

##### 描述树总图（继承 × 对象树合并视角）

```
┌── 接口层 (types/) ──────────────────────┐   ┌── 实现层 (impl/types/) ─────────────────┐
│ Weak_object                              │   │ Weak_object_impl                        │
│   └ Entity_object                        │   │   └ Entity_object_impl                  │
│        └ Abstract_table                  │   │        └ Abstract_table_impl            │
│             ├── Table  ◄──┐              │   │             ├── Table_impl  ◄──┐        │
│             └── View      │  实体         │   │             └── View_impl      │ 实体   │
│                           │  对象         │   │                               │ 对象   │
│                           │               │   │                               │        │
│ ══════════ 描述树（对象树）══════════════  │   │  ═══════════ restore/store 链路 ═══════ │
│                           │               │   │                               │        │
│                     dd::Table 对象         │   │                     Table_impl 对象     │
│                     （mysql.tables 一行）  │   │           store_attributes/restore_*    │
│            ┌──────────┼───────────┬───────┼───┼────────┐                      │        │
│            ▼          ▼           ▼       ▼   ▼        ▼                      │        │
│      Column[1..n]  Index[1..n] Foreign_key Partition[1..n] Trigger Check     │        │
│       (Entity)      (Entity)    (Entity)    (Entity)     (Entity) (Entity)    │        │
│           │            │            │           │                             │        │
│           ▼            ▼            ▼     ┌─────┴──────┐                      │        │
│   Column_type_    Index_       Foreign_key_│Partition_  │                      │        │
│   element         element      element     │index   ▼   │                      │        │
│   (Weak)          (Weak)       (Weak)      │Partition_   │                      │        │
│                                            │value(Weak)  │                      │        │
│                                            └─────────────┘                      │        │
└─────────────────────────────────────────────────────────────────────────────────┘

每一条「父 → 子」边 = 一次 restore_items / store_items 调用（对应一张 DD 表）
```

★ 注意 `Column_type_element` / `Index_element` / `Foreign_key_element` / `Partition_value` 是 **Weak_object**（无独立 id），其余全是 Entity_object。递归在这里停住——Weak 叶子没有 `restore_children` 覆写。

##### ★★ `Collection<T>::restore_items` 全链路逐行剖析

`Table_impl::restore_children` 里每个 `m_xxx.restore_items(...)` 都走到同一个模板（`collection.cc:123`）：

```cpp
template <typename T>
bool Collection<T>::restore_items(Parent_item *parent,
                                  Open_dictionary_tables_ctx *otx,
                                  Raw_table *table, Object_key *key,
                                  Compare comp) {
  assert(table);              // ★① 没注册就报错：注释说"用 register_tables()"
  assert(empty());

  std::unique_ptr<Object_key> key_holder(key);   // key 归本函数所有

  std::unique_ptr<Raw_record_set> rs;
  if (table->open_record_set(key, rs)) return true;   // ② 打开扫描（初始化 B-Tree cursor）

  /* ── 第一阶段：只填直接子对象的标量字段 ── */
  Raw_record *r = rs->current_record();
  while (r) {
    Collection<T>::impl_type *item =
        Collection<T>::impl_type::restore_item(parent);       // ③ 工厂方法 new 子对象
    item->set_ordinal_position(static_cast<uint>(m_items.size() + 1));
    m_items.push_back(item);

    if (item->restore_attributes(*r) || rs->next(r)) {        // ④ 行 → 标量字段
      clear_all_items();      // ⑤ 失败回滚：清空已加载的
      return true;
    }
  }

  /* 必须先关掉父扫描，才能递归扫子表（见下方 ★★★ 注释） */
  rs.reset();                // 析构时 close cursor

  /* ── 第二阶段：递归加载子对象 ── */
  for (auto item : m_items) {
    if (item->restore_children(otx) || item->validate()) {    // ⑥
      clear_all_items();
      return true;
    }
  }
  return false;
}
```

逐行要点：

- ① **`assert(table)` 的提示**（源码注释原文）：*if this assert is firing, that means the table was not registered for that transaction. Use `Open_dictionary_tables_ctx::register_tables()`* —— 这正是 `otx` 的职责（见下文第 3.5 节）：**每张要读的 DD 表必须先在 otx 注册，否则扫描时 handler 未打开**。
- ② `open_record_set(key, rs)`：按 `Object_key`（id / name / table_id 等条件）初始化一个**共享表扫描上下文**。
- ③ `restore_item(parent)`：**静态工厂**——`Collection<T>::impl_type` 直接暴露了"接口 T ↔ 实现 T_impl"的绑定，这是双轨设计在模板层的落点。
- ④⑤：**失败即整体回滚**（`clear_all_items`），DD 加载是"全有或全无"。
- ⑥ 第二阶段每个 item 做 `restore_children`（递归）+ `validate`（结构校验，如"索引必须引用已存在的列"）。

★★★ **为什么要拆两个阶段**（`collection.cc:159-177` 注释的完整逻辑）：

> 父子对象可能存**同一张 DD 表**（如父分区与子分区都在 `mysql.table_partitions`）。单个 table handler 无法同时维护**两个表扫描上下文**。支持同表多 handler 在当前 DD 框架下不容易。所以方案是：**先扫完父对象的所有行、关闭游标，再逐个递归读子对象**。

这是"两阶段"的根本原因——不是风格问题，是**单 handler 单游标**的物理约束。

##### `store_items` / `drop_items`（`collection.cc:211-263`）

```cpp
template <typename T>
bool Collection<T>::store_items(Open_dictionary_tables_ctx *otx) {
  if (empty()) return false;

  // ① 先处理"删除集合"
  for (auto *removed : m_removed_items) {
    if (removed->validate() || removed->drop(otx)) return true;
  }
  delete_container_pointers(m_removed_items);   // ② 物理释放

  // ③ 再写当前集合（新增/更新）
  for (Collection<T>::impl_type *item : m_items) {
    if (item->validate() || item->store(otx)) return true;
  }
  return false;
}
```

★ **`m_removed_items` 是"墓碑集合"**：`drop_column()` 这类操作把对象从 `m_items` 移到 `m_removed_items`，`store_items` 时先落墓碑（从 DD 表删行）、再写存活者。这解释了为什么 DDL 修改一个 Collection 后，store 一次能同时完成"删旧的 + 写新的"。

```cpp
template <typename T>
bool Collection<T>::drop_items(Open_dictionary_tables_ctx *otx,
                               Raw_table *table, Object_key *key) const {
  if (empty()) return false;

  // ① 先递归删子对象（drop_children）
  for (const Collection<T>::impl_type *item : m_items) {
    if (item->drop_children(otx)) return true;
  }

  // ② 再打开记录集删自己的行
  std::unique_ptr<Raw_record_set> rs;
  if (table->open_record_set(key, rs)) return true;
  Raw_record *r = rs->current_record();
  while (r) {
    if (r->drop()) return true;      // 逐行删除
    if (rs->next(r)) return true;
  }
  return false;
}
```

★ `drop_items` 的**先子后己**与 `store_items` 的"先墓碑"、`restore_items` 的"先标量后子树"是同一个原则的三次体现：**顺序由外键依赖决定，反了就会违反 DD 表的外键约束或留下孤儿行**。

#### 五、为什么把元数据放进 InnoDB 表，而不是继续用文件

5.7 及以前元数据在 **frm 文件 + 独立的系统表（MyISAM）** 里。8.0 全部迁入 InnoDB 表（`mysql.ibd`）。这个决策换来：

| 收益 | 代价 |
|---|---|
| ★ **事务性**：`CREATE TABLE` 变成"在一个事务里插/改若干 DD 表行"，天然原子 | 元数据访问从"读文件"变成"走 buffer pool + B+ 树"，路径变长 |
| 统一崩溃恢复（DD 的修改也记 redo） | 需要特殊的**自举**流程（DD 表本身要先存在才能读 DD，见第 5 节） |
| 消除 frm 与引擎字典不一致的经典问题 | 元数据访问受 InnoDB 并发控制约束 |

★ **代价的具体后果**：DD 访问比 5.7 慢，且**DD 表自己关掉了持久统计**（见 `Object_table` 的共享选项 `STATS_PERSISTENT=0`）——因为 DD 表访问模式固定，统计采样纯属浪费。这也解释了 [`statistics.md`](statistics.md) 里 `dict_stats_update()` 那句 `ut_a(strchr(table->name.m_name, '/') != nullptr)` 断言：**DD/系统内部表根本不走统计采集**。

#### 六、`Object_table`：为升级兼容保留废表定义

`dd::Object_table`（`types/object_table.h:72`）代表"mysql.tables 这类 DD 表本身"。它的类注释里有一条容易被忽略的约束（:47-49）：

> The server code should contain a `Object_table` subclass for each DD table which is a target table for at least one of the supported DD versions (i.e., the DD versions from which this server can upgrade). **So even if a previous DD version stops using a DD table, the later servers which can upgrade need [to keep it].**

即：**某张 DD 表即使当前版本已弃用，只要还有受支持的升级路径经过它，它的定义就必须保留**。这是一条明确的"为升级兼容性背技术债"的决策——代码里留着不再使用的表定义是有意为之，不是清理遗漏。

所有 DD 表共享一组固定选项（:77-84）：`ENGINE=INNODB` / `DEFAULT CHARSET=utf8mb3` / `COLLATE=utf8mb3_bin` / `ROW_FORMAT=DYNAMIC` / **`STATS_PERSISTENT=0`** / `TABLESPACE=mysql`。

### 算法与数据结构

| 结构 | 角色 |
|---|---|
| `Weak_object` | 最顶层，只要求 `debug_print()`——"能打印自己"是所有 DD 对象的下限 |
| `Entity_object` | 有 id、有名字、可持久化（`is_persistent()`）的对象 |
| `Object_table` | 一张 DD 表（mysql.tables…）的抽象，含 `Object_table_definition`（字段/索引定义） |
| `Object_key` | 查询条件抽象（按 id / 按 name / 按 se_private_id…），`Storage_adapter` 用它生成 B+ 树查找 |
| `Properties` | `options` / `se_private_data` 的键值容器（字符串键 + 类型化出参） |
| `Collection<T>` | 子对象集合，支持按 name / 按序号两种索引 |

**加载算法**（`Storage_adapter::get()`）：构造 `Object_key` → 从根系统表 B+ 树定位 `Raw_record` → `restore_attributes()` → 递归 `restore_children()`。复杂度 O(树中节点数)，即读一张表 ≈ 读 1 + N 列 + M 索引 + … 行。

### 他库对比与演进动机

| | MySQL 8.0 DD | PostgreSQL | MySQL 5.7（旧） |
|---|---|---|---|
| 载体 | InnoDB 表（`mysql.ibd`） | `pg_catalog` 下的普通系统表 | frm 文件 + MyISAM 系统表 |
| 事务性 | ★ 是（随 DDL 事务提交） | 是 | **否**（frm 是独立文件，DDL 崩溃留下残留） |
| 缓存 | 两级（client registry + shared cache） | `syscache`（relcache） | table cache + frm 缓存 |
| 引擎私有数据 | `se_private_data` 字符串背包 | 无（各扩展自己建表） | frm 里固定字段 |

**演进动机**：5.7 的"frm + MyISAM 系统表"分属两套介质，**无法原子**——`CREATE TABLE` 崩溃后可能留下".ibd 存在但字典里没有"的孤儿。8.0 把元数据统一到 InnoDB，就是为了用**一个事务**覆盖"元数据 + 引擎数据 + binlog"三者的提交（见第 6 节）。

---

## 核心实现

> 本章按"存储 → 对象模型 → 缓存 → SDI → 自举 → 原子 DDL → 升级"展开。

### 1. DD 表体系与存储

### 1.1 `mysql` 库下的表分三类

类型枚举 `sql/dd/impl/system_registry.h:349`：

```cpp
enum class Types { INERT, CORE, SECOND, DDSE_PRIVATE, DDSE_PROTECTED, PFS, SYSTEM };
```

| 类 | 数量 | 说明 | 例子 |
|---|---|---|---|
| **INERT** | 1 | 永不改变 | `dd_properties` |
| **CORE** | 22 | 处理任意表 cache miss 必须存在 | `tables` / `columns` / `indexes` / `schemata` / `tablespaces` / `column_type_elements` / `foreign_keys` / `check_constraints` … |
| **SECOND** | 7 | 可由 CORE 表读出 | `events` / `routines` / `parameters` / `table_stats` / `index_stats` / `spatial_reference_systems` … |
| **DDSE_PRIVATE / PROTECTED** | 4 | **不是 server DD 表**，是 InnoDB 请求 server 托管的表 | `innodb_dynamic_metadata`(hidden→PRIVATE) / `innodb_ddl_log`(hidden→PRIVATE) / `innodb_table_stats`(PROTECTED) / `innodb_index_stats`(PROTECTED) |
| **SYSTEM** | ~40 | 普通"系统表"，非 DD | `user` / `db` / `tables_priv` / `time_zone_*` … |

★ DD 表的列定义是 **C++ 硬编码**的（如 `sql/dd/impl/tables/tables.cc:58-185` 的 `add_field()`/`add_index()`），不是从某处读出来的——这是自举能成立的前提（见第 5 节）。

#### ★★ "数量分歧"的澄清（30 vs 28+5 vs 32）

**源码实测结论**：

| 数字 | 含义 | 出处 |
|---|---|---|
| **30** | `system_registry.cc` 里 `register_table<dd::tables::X>` 的条数 | INERT 1 + CORE 22 + SECOND 7 |
| **34** | 30 + InnoDB 的 **4 张 DDSE 表** | MTR：`There are 34 DD/DDSE tables in total` |
| **32** | `mysql.tables WHERE schema_id=1 AND hidden='System'` 的行数 = 34 − 2 | MTR：`let $number_of_hidden_dd_tables = 32;` |
| **28+5** | ⚠️ **源码里找不到任何依据** | grep `innodb_dictionary` 全树 **0 命中** |

> ★ 被减掉的那 2 张是 `innodb_table_stats` / `innodb_index_stats`——它们 `set_hidden(false)` ⇒ 类型 `DDSE_PROTECTED` ⇒ `hidden=''`（对普通用户**可见**）。

MTR 里的权威断言（`mysql-test/include/dd_schema_assert_and_fill_table_names.inc`）：

```sql
let $number_of_hidden_dd_tables = 32;
let $assert_cond = "[SELECT COUNT(*) from mysql.tables
                       WHERE schema_id = 1 AND hidden = \'System\']"
                    = $number_of_hidden_dd_tables;
--let $assert_text = There are 32 hidden DD/DDSE tables.
```

```sql
eval INSERT INTO $ddse_table_names (name) VALUES
               ('innodb_ddl_log'),
               ('innodb_dynamic_metadata'),
               ('innodb_index_stats'),
               ('innodb_table_stats');
...
let $assert_cond = "[SELECT (SELECT COUNT(*) FROM $dd_table_names) +
                            (SELECT COUNT(*) FROM $ddse_table_names)]"
                    = $number_of_hidden_dd_tables + 2;
--let $assert_text = There are 34 DD/DDSE tables in total.
```

**InnoDB 的 4 张 DDSE 表**（精确 4 张，没有第 5 张）——`innobase_ddse_dict_init()`：

```cpp
  dd::Object_table *innodb_dynamic_metadata = dd::Object_table::create_object_table();
  innodb_dynamic_metadata->set_hidden(true);        /* hidden → DDSE_PRIVATE */
  ...
  dd::Object_table *innodb_table_stats = dd::Object_table::create_object_table();
  innodb_table_stats->set_hidden(false);            /* 可见 → DDSE_PROTECTED */
  def->set_table_name("innodb_table_stats");
  ...
  dd::Object_table *innodb_ddl_log = dd::Object_table::create_object_table();
  innodb_ddl_log->set_hidden(true);
  ...
  tables->push_back(innodb_dynamic_metadata);
  tables->push_back(innodb_table_stats);
  tables->push_back(innodb_index_stats);
  tables->push_back(innodb_ddl_log);
```

`hidden` 怎么决定 PRIVATE / PROTECTED（bootstrap 侧）：

```cpp
  /*
    If Object_table::is_hidden(), then we add the tables as type DDSE_PRIVATE
    (not available neither for DDL nor DML), otherwise, we add them as type
    DDSE_PROTECTED (available for DML, not for DDL).
  */
  System_tables::Types table_type = System_tables::Types::DDSE_PROTECTED;
  if (ddse_table->is_hidden()) {
    table_type = System_tables::Types::DDSE_PRIVATE;
  }
```

**另外一套容易混的东西：InnoDB 自己的 `SYS_*` 内部字典表**——8.0 里**只有 4 张还被装载**：

```cpp
    dict_load_sys_table(dict_sys->sys_tables);
    dict_load_sys_table(dict_sys->sys_columns);
    dict_load_sys_table(dict_sys->sys_indexes);
    dict_load_sys_table(dict_sys->sys_fields);
```

> ⚠️ `SYS_FOREIGN` / `SYS_FOREIGN_COLS` / `SYS_TABLESPACES` / `SYS_DATAFILES` 的枚举仍留在 `dict0boot.h`，但 8.0 **不再 create/load**。`dict_sys->sys_virtual` 成员还在，但 `dict0boot.cc` 里 grep `sys_virtual` **0 命中**。

**`System_tables::Types` 是 7 个值**（不是 4 个）：

```cpp
  enum class Types {
    INERT,
    CORE,
    SECOND,
    DDSE_PRIVATE,
    DDSE_PROTECTED,
    PFS,
    SYSTEM
  };
```

含义（源码注释原文）：

```cpp
    - An INERT table can never change.
    - The dd::Table objects representing the CORE tables must be present
      to handle a cache miss for an arbitrary table.
    - The dd::Table objects representing the SECOND order tables can be
      fetched from the dd tables as long as the core table objects are
      present.
    - The DDSE tables are not needed by the data dictionary, but the
      server manages these based on requests from the DD storage engine.
    - The PFS tables are not needed by the data dictionary, but the
      server manages these based on requests from the performance schema.
```

**注册入口不是 `init()`**（常被写错），是两个函数：

```cpp
// sql/dd/impl/system_registry.cc:107-117
template <typename X>
void register_table(dd::System_tables::Types type) {
  dd::System_tables::instance()->add(
      MYSQL_SCHEMA_NAME.str, X::instance().name(), type, &X::instance());
}
void register_table(const dd::String_type table,
                    dd::System_tables::Types type) {
  dd::System_tables::instance()->add(MYSQL_SCHEMA_NAME.str, table, type,
                                     nullptr);
}
/* ★ 第 113-117 行这个非模板重载就是"只注册名字、不注册定义"的条目：
   第 4 个参数 Object_table* 传 nullptr。 */
```

**30 张完整清单**（`add_inert_dd_tables()` 1 张 + `add_remaining_dd_tables()` 29 张）：

```cpp
void System_tables::add_inert_dd_tables() {
  dd::System_tables::Types inert = dd::System_tables::Types::INERT;
  register_table<DD_properties>(inert);        // 01  dd_properties（INERT，唯一一张）
}

void System_tables::add_remaining_dd_tables() {
  dd::System_tables::Types core = dd::System_tables::Types::CORE;
  dd::System_tables::Types second = dd::System_tables::Types::SECOND;
  dd::System_tables::Types system = dd::System_tables::Types::SYSTEM;

  register_table<Catalogs>(core);                       // 02  catalogs
  register_table<Character_sets>(core);                 // 03  character_sets
  register_table<Check_constraints>(core);              // 04  check_constraints
  register_table<Collations>(core);                     // 05  collations
  register_table<dd::tables::Column_statistics>(core);  // 06  column_statistics（直方图）
  register_table<Column_type_elements>(core);           // 07  column_type_elements（SET/ENUM 元素）
  register_table<Columns>(core);                        // 08  columns
  register_table<Events>(second);                       // 09  events
  register_table<Foreign_key_column_usage>(core);       // 10  foreign_key_column_usage
  register_table<Foreign_keys>(core);                   // 11  foreign_keys
  register_table<Index_column_usage>(core);             // 12  index_column_usage
  register_table<Index_partitions>(core);               // 13  index_partitions
  register_table<Index_stats>(second);                  // 14  index_stats
  register_table<Indexes>(core);                        // 15  indexes
  register_table<Parameter_type_elements>(second);      // 16  parameter_type_elements
  register_table<Parameters>(second);                   // 17  parameters
  register_table<Resource_groups>(core);                // 18  resource_groups
  register_table<Routines>(second);                     // 19  routines
  register_table<Schemata>(core);                       // 20  schemata
  register_table<Spatial_reference_systems>(second);    // 21  ★ 物理表名 st_spatial_reference_systems
  register_table<Table_partition_values>(core);         // 22  table_partition_values
  register_table<Table_partitions>(core);               // 23  table_partitions
  register_table<Table_stats>(second);                  // 24  table_stats
  register_table<Tables>(core);                         // 25  tables
  register_table<Tablespace_files>(core);               // 26  tablespace_files
  register_table<Tablespaces>(core);                    // 27  tablespaces
  register_table<Triggers>(core);                       // 28  triggers
  register_table<View_routine_usage>(core);             // 29  view_routine_usage
  register_table<View_table_usage>(core);               // 30  view_table_usage
  ...
```

> ⚠️ **唯一一个"类名 ≠ 表名"的**：`Spatial_reference_systems` 的物理表名是 **`st_spatial_reference_systems`**（带 `st_` 前缀）。

**还有 41 条 SYSTEM 条目（只有名字，无 C++ 定义）**——`user` / `db` / `tables_priv` / `time_zone*` / `slave_*` 等：

```cpp
  register_table("backup_history", system);
  register_table("columns_priv", system);
  register_table("db", system);
  register_table("default_roles", system);
  register_table("global_grants", system);
  register_table("gtid_executed", system);
  register_table("help_category", system);
  ...
  register_table("role_edges", system);
  register_table("slave_master_info", system);
  register_table("time_zone", system);
  register_table("time_zone_name", system);
  register_table("user", system);
  ...
```

它们的 DDL **不在 C++ 里**，而在 SQL 脚本里（`scripts/mysql_system_tables.sql` 等）。注册它们的唯一目的是让服务器知道"这些名字属于 mysql 库且是系统表"，从而在访问控制与升级清单里有条目——bootstrap 全程跳过它们：

```cpp
      const dd::Table *dd_table = nullptr;
      if ((*it)->property() != System_tables::Types::SYSTEM &&
          thd->dd_client()->acquire(MYSQL_SCHEMA_NAME.str,
                                    (*it)->entity()->name(), &dd_table))
        return dd::end_transaction(thd, true);
```

此前三个口径对不上，回 `system_registry.cc:151-258` 逐行核实后归因：

| 口径 | 数字 | 归因 |
|---|---|---|
| **本文（正确）** | 30 = INERT 1 + CORE 22 + SECOND 7 | system_registry 逐行数出 |
| SQL 查询结果 | 32 行 | = 30 张 DD 表 + `innodb_ddl_log` + `innodb_dynamic_metadata`（这两张是 InnoDB DDSE 表，`hidden='System'` 能查到）；**没查到** `innodb_table_stats`/`innodb_index_stats`——它们的 hidden 类型不是 `System` |
| 本仓库 `dict0dict.h` 注释"28 MySQL + 5 InnoDB" | 28+5 | ★ 注释**不准**：把 `dd_properties` 错归"InnoDB 自有"（system_registry 明确注册它为 MySQL 的 INERT DD 表），且"28"与其列出的 29 个表名对不上 |

★ 结论口径：**MySQL DD 表 30 + InnoDB DDSE 表 4（dynamic_metadata/table_stats/index_stats/ddl_log）+ SYSTEM 系统表一批**。`dd_properties` 是 MySQL DD 表（INERT），不是 InnoDB 自有。

★ **mysql 系统表 100% 在 mysql.ibd 吗**：是。SYSTEM 类（user/db/plugin/component/gtid_executed/help_*/time_zone_* 等）同样注册在 `System_tables`，建表时 `TABLESPACE=mysql`（`Object_table_impl` 的共享选项），物理文件就是 mysql.ibd——**整个 mysql 库（除少数特例如 `ndb_binlog_index` 属 NDB）都在 mysql.ibd**。

#### 30 张 DD 表逐张用途

**CORE（22 张，处理任意表 cache miss 必须存在）**

| 表 | 存什么 |
|---|---|
| `catalogs` | 目录（实际只有一个 `def`） |
| `schemata` | 数据库（schema）定义 |
| `tables` | 表定义（引擎、行格式、选项、`se_private_data`…） |
| `columns` | 列定义（类型、默认值、生成列表达式、列级 `se_private_data`） |
| `column_type_elements` | SET / ENUM 类型的枚举元素 |
| `indexes` | 索引定义（含索引级 `se_private_data`：root 页号、space、id） |
| `index_column_usage` | 索引引用了哪些列（索引 ↔ 列的多对多） |
| `index_partitions` | 索引的分区信息 |
| `table_partitions` | 表分区定义 |
| `table_partition_values` | 分区值（RANGE 的 `VALUES LESS THAN`、LIST 的值列表） |
| `tablespaces` | 表空间定义 |
| `tablespace_files` | 表空间包含哪些物理文件 |
| `foreign_keys` | 外键定义 |
| `foreign_key_column_usage` | 外键引用了哪些列 |
| `check_constraints` | CHECK 约束 |
| `triggers` | 触发器 |
| `view_table_usage` | 视图引用了哪些表（依赖关系） |
| `view_routine_usage` | 视图引用了哪些存储例程 |
| `character_sets` / `collations` | 字符集与排序规则 |
| `column_statistics` | 列统计（直方图） |
| `resource_groups` | 资源组（Resource Group 特性） |

**SECOND（7 张，可由 CORE 表读出）**

| 表 | 存什么 |
|---|---|
| `events` | 事件调度器的事件 |
| `routines` | 存储过程与函数 |
| `parameters` | 例程的参数 |
| `parameter_type_elements` | 参数的 SET / ENUM 元素 |
| `spatial_reference_systems` | 空间参考系（SRS） |
| `table_stats` / `index_stats` | DD 层的统计缓存 |

⚠️ 易混：`table_stats` / `index_stats`（DD 表，SECOND）与 `innodb_table_stats` / `innodb_index_stats`（InnoDB 的 DDSE_PROTECTED 表）**是两套**——后者才是 InnoDB 持久统计真正存放的地方（也因此后者允许 DML，见 1.3 节）。完整辨析见 [`statistics.md`](statistics.md)。

**INERT（1 张）**：`dd_properties` —— 存 DD 自身的版本号，永不改变。

### 1.2 ★★ DD 表空间（mysql.ibd）全景

#### 身份：一个 space_id 就是全部

```cpp
// fsp0fsp.cc:254 —— DD 表空间的"判定函数"就一个等值判断
bool fsp_is_dd_tablespace(space_id_t space_id) {
  return (space_id == dict_sys_t::s_dict_space_id);
}
```

| 名字 | 值 | 出处 |
|---|---|---|
| InnoDB space_id | **`0xFFFFFFFE`** | `dict_sys_t::s_dict_space_id` |
| DD 侧 `dd::Tablespace::id` | **1** | `DD_TABLESPACE_ID`（`dictionary_impl.cc:99`） |
| 表空间名 | `"mysql"` | `s_dd_space_name` |
| 文件名 | `mysql.ibd` | `s_dd_space_file_name` |

★ **同一个表空间有两个名字**：表空间名 `mysql`（DD/SQL 层用）、文件名 `mysql.ibd`（InnoDB 层用）——在 `dd_create_hardcoded(s_dict_space_id, "mysql.ibd")` 里同时传入。space_id 取 `0xFFFFFFFE` 而不是顺排下一个普通 id，是为了**一个等值判断筛出全部"特殊空间"**（见 [`tablespace.md`](../../innodb/physical/tablespace.md) 的 space_id 分布表——undo/临时/DD 全部挤在高地址段）。

#### 硬编码的"三处身份"

1. **创建/打开都硬编码**（`ha_innodb.cc:5549`）：`create ? dd_create_hardcoded(...) : dd_open_hardcoded(...)`——不走 `CREATE TABLESPACE` 的 SQL 链（详见 [`innodb_dict.md`](innodb_dict.md) 的 mysql.ibd 创建节）；
2. ★ **打开失败即启动中止**（`srv0start.cc:2106`）：

```cpp
space = fil_space_acquire_silent(dict_sys_t::s_dict_space_id);
if (space == nullptr) {
  dberr_t error = fil_ibd_open(true, FIL_TYPE_TABLESPACE, s_dict_space_id, ...);
  if (error != DB_SUCCESS) {
    ib::error(ER_IB_MSG_1142);
    return srv_init_abort(DB_ERROR);      // ★ mysql.ibd 打不开 = 整个实例起不来
  }
}
```

3. ★ **`innodb_dynamic_metadata` 表的 table_id / index_id 是硬编码常量**（`dict0dict.cc:5183-5214`）：

```cpp
table = dict_mem_table_create(table_name, dict_sys_t::s_dict_space_id, ...);
table->id = dict_sys_t::s_dynamic_meta_table_id;          // 常量，不走 DD 分配
table->is_dd_table = true;
table->dd_space_id = dict_sys_t::s_dd_dict_space_id;
...
m_index->id = dict_sys_t::s_dynamic_meta_index_id;        // 常量
```

——因为动态元数据要在**崩溃恢复最早阶段**（DD 还没就绪）就能读写，它的 id 不能等 DD 分配，直接写死。

#### mysql.ibd 在 DD 里的"自描述"

`mysql.tablespaces` 表里也有一行描述 mysql.ibd 自己（`ha_innodb.cc:5567`）：

```cpp
snprintf(se_private_data_dd, len, fmt, dict_sys_t::s_dict_space_id,
         predefined_flags, DD_SPACE_CURRENT_SRV_VERSION,
         DD_SPACE_CURRENT_SPACE_VERSION);
```

即它的 `se_private_data` 记录 (space_id, flags, server 版本, space 版本)——**DD 表空间也要向 DD 自报家门**（`DD_SPACE_CURRENT_SRV_VERSION` / `SPACE_VERSION` 正是 [`page_structure.md`](../../innodb/physical/page_structure.md) 里页 0 复用 PREV/NEXT 存的那两个版本号）。

#### 加载路径上的特殊分支

凡是"表空间名 → space_id → 名字"的映射，DD 表空间都有硬分支：

- `dict0load.cc:1518`：`space_id == s_dict_space_id` → 表名/表空间名**强制** `"mysql"`（不查 DD）；
- `fsp0file.cc:290`：`fsp_is_dd_tablespace(m_space_id)` → 名字直接 `s_dd_space_name`；
- `dict0dd.cc:4843` 同理。

★ 原因：**早期阶段 DD 表还没打开，没法从 DD 查名字**——所以凡是"可能在 DD 就绪前被问到"的路径，都短路返回硬编码名。

#### 与其他表空间的对比

| | mysql.ibd（DD 表空间） | 普通 file-per-table .ibd |
|---|---|---|
| 创建 | `dd_create_hardcoded`（无 SQL） | `CREATE TABLE` 链路 |
| space_id | `0xFFFFFFFE` 哨兵 | 顺排分配 |
| SDI | ★ 自带 SDI 索引 | 自带 |
| 里面的表 | DD 表（hidden=SYSTEM）+ DDSE 表 | 用户表 |
| 统计 | ★ `STATS_PERSISTENT=0`（DD 表全部关统计） | 默认开 |
| 升级 | ★ `upgrade_space_version(s_dict_space_id, ...)` 专门升级 space 版本（`ha_innodb.cc:4232`） | 随普通流程 |

结论：**mysql.ibd 是一个"恰好被 DD 用"的普通 InnoDB 表空间**——FSP header / XDES / extent 管理全套都有；特殊性全部来自"DD 没就绪之前就得能用"这条需求：id 写死、名字短路、打不开就 abort。

### 1.3 为什么用户不能 DML 这些表

**★ 源码里有一张完整的访问矩阵**（`Dictionary_impl::is_dd_table_access_allowed()`，注释标注来自 WL#6391）——这是本问题的权威答案：

```cpp
bool Dictionary_impl::is_dd_table_access_allowed(bool is_dd_internal_thread,
                                                 bool is_ddl_statement,
                                                 const char *schema_name,
                                                 size_t schema_length,
                                                 const char *table_name) const {
  /*
    From WL#6391, we have the following matrix describing access:

    ---------+---------------------+
             | Dictionary internal |
    ---------+----------+----------+
             |   DDL    |   DML    |
    ---------+-----+----+-----+----+
             | IN  | EX | IN  | EX |
    ---------+-----+----+-----+----+
    Inert    |  X          X       |
    Core     |  X          X       |
    Second   |  X          X       |
    DDSE_priv|  X          X       |
    DDSE_prot|  X          X    X  |
    SYSTEM   |  X    X     X    X  |
    ---------+---------------------+
    ...
  */
  if (schema_length != MYSQL_SCHEMA_NAME.length ||
      strncmp(schema_name, MYSQL_SCHEMA_NAME.str, MYSQL_SCHEMA_NAME.length) ||
      is_dd_internal_thread ||
      DBUG_EVALUATE_IF("skip_dd_table_access_check", true, false))
    return true;

  const System_tables::Types *table_type =
      System_tables::instance()->find_type(schema_str, table_str);

  return (table_type == nullptr ||
          (*table_type == System_tables::Types::DDSE_PROTECTED &&
           !is_ddl_statement) ||
          *table_type == System_tables::Types::SYSTEM);
}
```

⇒ **只有两类对外部线程放开**：`DDSE_PROTECTED` 允许 DML（即 `innodb_table_stats` / `innodb_index_stats` 可以 `UPDATE`）、`SYSTEM` 允许 DDL+DML（`user` / `db` 这些 ACL 表本来就是给用户改的）。其余全禁。

**主拦截点**在 `LEX::add_table_to_list()`——任何语句引用 `mysql.<DD表>` 都会先走这里：

```cpp
  // Check access to DD tables. We must allow CHECK and ALTER TABLE
  // for the DDSE tables, since this is expected by the upgrade
  // client. We must also allow DDL access for the initialize thread,
  // since this is creating the I_S views.
  ...
  if (dictionary &&
      !dictionary->is_dd_table_access_allowed(..., ptr->db, ptr->db_length,
                                              ptr->table_name)) {
    if (!(lex->sql_command == SQLCOM_CREATE_VIEW &&
          dd::get_dictionary()->is_system_view_name(...))) {
      my_error(ER_NO_SYSTEM_TABLE_ACCESS, MYF(0),
               ER_THD_NONCONST(thd, dictionary->table_type_error_code(
                                        ptr->db, ptr->table_name)),
               ptr->db, ptr->table_name);
      if (thd->is_error()) return nullptr;
    }
  }
```

**另有 4 处补充拦截**（防止绕过主闸）：

| 位置 | 场景 | 错误码 |
|---|---|---|
| `sql/sp_head.cc:3221` | 存储程序里的表引用 | `ER_NO_SYSTEM_TABLE_ACCESS` |
| `sql/key_spec.cc:107` | 外键引用到 DD 表 | `ER_NO_SYSTEM_TABLE_ACCESS` |
| `sql/rpl_utility.cc:471` | 复制 applier | `ER_SERVER_NO_SYSTEM_TABLE_ACCESS` |
| `sql/dd/impl/dictionary_impl.cc:596/659` | native/plugin 表的 create/drop | `ER_NO_SYSTEM_TABLE_ACCESS` |

**"DD 表"与"system 表"是两个不同的集合**（`is_dd_table_name()` vs `is_system_table_name()`）：

```cpp
bool Dictionary_impl::is_dd_table_name(const String_type &schema_name,
                                       const String_type &table_name) const {
  if (!is_dd_schema_name(schema_name)) return false;
  const System_tables::Types *table_type =
      System_tables::instance()->find_type(schema_name, table_name);
  return (table_type != nullptr &&
          (*table_type == System_tables::Types::CORE ||
           *table_type == System_tables::Types::INERT ||
           *table_type == System_tables::Types::SECOND ||
           *table_type == System_tables::Types::DDSE_PRIVATE ||
           *table_type == System_tables::Types::DDSE_PROTECTED));
}
```

- **DD 表** = CORE + INERT + SECOND + DDSE_PRIVATE + DDSE_PROTECTED = **34**
- **system 表** = 只有 SYSTEM 那 **41 个名字**（`user` / `db` / …）

错误文案按类型分流（`type_name_error_code()`）：INERT/CORE/SECOND → *"data dictionary table"*；DDSE/PFS/SYSTEM → *"system table"*。所以报错形如：

```
ERROR 3554 (HY000): Access to data dictionary table 'mysql.tables' is rejected.
```

> ⚠️ 两个常被猜错的位置：`sql/sql_base.cc` 与 `sql/auth/sql_authorization.cc` **都没有**这个判定（grep 相关关键词均 0 命中），主闸就在 `sql_parse.cc` 的 `add_table_to_list()`。

判定 `dictionary_impl.cc:294-307`（只有前五类算 DD 表），访问控制矩阵 `:334-384`：

```cpp
return (table_type == nullptr ||
        (*table_type == Types::DDSE_PROTECTED && !is_ddl_statement) ||
        *table_type == Types::SYSTEM);
```

即：DD 表**一律拒绝**；`DDSE_PROTECTED`（如 `innodb_table_stats`）**只允许非 DDL 的 DML**；`SYSTEM` 表（user/db 等）允许。

报错点 `sql_parse.cc:6158-6187`，错误码 `ER_NO_SYSTEM_TABLE_ACCESS`，消息来自 `table_type_error_code()`：

- INERT/CORE/SECOND → "data dictionary table"
- DDSE_*/PFS/SYSTEM → "system table"

---

### 2. 对象模型与 `se_private_data`

### 2.1 核心类型

| 类型 | 关键方法 |
|---|---|
| `dd::Table` | `se_private_data()` / `se_private_id()`（`:174-185`）、`indexes()`、`partitions()` / `leaf_partitions()`、`foreign_keys()`、`check_constraints()`、`options()` |
| `dd::Column` | `default_value_utf8()`（`:229`）、`generation_expression_utf8()`（`:258`）、`options()`、`se_private_data()`、`is_virtual()` |
| `dd::Index` | `type()` / `is_visible()` / `is_generated()`、`elements()`、`se_private_data()` |

### 2.2 `dd::Properties`：不改表结构就能加属性

上面提到的 `options` 与 `se_private_data` 两列，存的都是 **`dd::Properties`**。它的头注释把机制写得很完整：

```
  The Properties class defines an interface for storing key=value pairs,
  where both key and value may be UTF-8 strings.
  ...
  Please note that in debug builds, the get() functions will assert that
  the key exists. This is to make sure that non-existing keys are
  handled explicitly at the client side.

  The raw_string() function returns a semicolon separated list of all
  key=value pairs. Characters '=' and ';' that are part of key or value
  are escaped using the '\' as an escape character. The escape character
  itself must also be escaped if being part of key or value.
```

即落盘格式是**分号分隔的 `key=value` 列表**，`=` 与 `;` 用 `\` 转义（转义符自身也要转义）。用法示例（注释原文）：

```cpp
    p->set("akey=avalue");        // 加 key=value
    p->set("intvalue", 1234);     // 加数值（模板函数转字符串）
    p->get("intvalue", &num);     // 取（debug 下会 assert key 存在）
    String_type mylist = p->raw_string();
```

★ **为什么用它**：这是 DD **扩展性**的核心答案——要加一个新的对象属性（如 instant ADD COLUMN 的那几个键），**不需要改 `mysql.*` 表结构、不需要走 DD 升级**，往 Properties 里塞一个键即可。代价是：**值是弱类型的 UTF-8 文本**，没有 schema 校验，且 debug 构建下 `get()` 不存在的键会直接断言失败（注释明说是为了让调用方"显式处理"）。

与之对照的是**独立列**（如 8.0.21+ 的 `engine_attribute`）——需要改表结构，但类型安全、可被 SQL 直接查。详见 [`system_views.md`](system_views.md) 对 `engine_attribute` 的说明。

#### 实现：4 KB 的 `properties_impl.cc`

落盘格式的**产生者**就是 `raw_string()`——遍历 map，key 与 value 各 `escape()` 一次，中间 `=`，结尾 `;`：

```cpp
const String_type Properties_impl::raw_string() const {
  String_type str("");
  str.reserve(16 * m_map.size());

  for (auto &it : m_map) {
    assert(valid_key(it.first));
    if (valid_key(it.first)) {
      escape(&str, it.first);
      str.append("=");
      escape(&str, it.second);
      str.append(";");
    }
  }
  return str;
}
```

反向解析是 `parse_properties()` → `eat_pairs()`（逐字符吃 `key=value;`），解析失败**整个对象作废**：

```cpp
Properties *Properties::parse_properties(const String_type &raw_properties) {
  Properties *tmp = new (std::nothrow) Properties_impl();
  String_type::const_iterator it = raw_properties.begin();

  if (eat_pairs(it, raw_properties.end(), tmp)) {
    delete tmp;
    tmp = nullptr;
  }
  return tmp;
}
```

三条实现细节值得知道：

- **`get()` 在 debug 下会 `assert(false)` 并返回 true**（key 不存在时）——注释明说是为了让调用方显式处理；release 下只返回 true
- **`set()` 拒绝空 key**（`if (!key.empty()) m_map[key] = value;`），且 key 非法时打 warning + assert
- **`valid_key()` 是闸门**：`set` / `get` / `raw_string` / `insert_values` 全都先过它，非法 key 直接拒绝（这就是"键名必须合法"的强制点）

⚠️ 一个实践含义：**因为整个串要么全解析成功要么全作废，任何一处转义出错都会导致该对象的全部属性丢失**（而不是丢一个键）。这也是为什么 `options` 有白名单而 `se_private_data` 没有——后者由各引擎自己保证键名与转义的正确性。

### 2.3 ★★ `options` vs `se_private_data`

两者都是 `dd::Properties`（KV），但**归属与校验完全不同**：

| | `options` | `se_private_data` |
|---|---|---|
| 语义 | **SQL 层 / 引擎无关**的建表选项 | **存储引擎私有**，server 不解释 |
| 校验 | ★ **有合法 key 白名单** | ★ **无白名单** |
| 存储列 | `mysql.tables.options` | `mysql.tables.se_private_data` |

★ **无白名单的代码证据**（`column_impl.cc:94-95`，同模式见 `index_impl.cc:78`、`abstract_table_impl.cc:103`）：

```cpp
m_options(default_valid_option_keys),   // 带白名单
m_se_private_data(),                     // 不带
```

`options` 白名单举例（`abstract_table_impl.cc:65-92`）：`avg_row_length / checksum / compress / encrypt_type / key_block_size / max_rows / min_rows / row_type / stats_auto_recalc / stats_persistent / stats_sample_pages / tablespace / timestamp / gipk` …——**这解释了为什么 `AUTOEXTEND_SIZE` 走 `options`**（它是 SQL 层概念）。

`se_private_data` 的 key（全部由 InnoDB 写入，`dict0dd.h`）：

| 层级 | key |
|---|---|
| 表 | `autoinc` / `data_directory` / `version` / `discard` / `instant_col` |
| 列 | `default` / `default_null` / `version_added` / `version_dropped` / `physical_pos` |
| 索引 | `id` / `space_id` / `table_id` / `root` / `trx_id` |
| 分区 | `format` / `instant_col` / `discard` |
| 表空间 | `flags` / `id` / `discard` / `server_version` / `space_version` / `state` |

★ 这就是"DD 是权威、dict 是投影"的**私有通道**：InnoDB 把只有自己懂的物理信息（root 页号、`physical_pos`、instant 版本）塞回 DD。

### 2.4 `default_value_utf8` / `generation_expression_utf8`

- 写入（`sql/dd/dd_table.cc`）：`set_generation_expression_utf8()` :631、`set_default_value_utf8()` :773；
- 落库/恢复（`column_impl.cc:308-320` / `:221-229`）；
- 消费方：I_S.COLUMNS 视图直接映射这两列（`system_views/columns.cc:51-52, 112-122`）。

★ 注意区分：`default_value_utf8` 是**给 SQL 层/I_S 看的文本**；InnoDB 真正用于 instant ADD COLUMN 的默认值走的是列 `se_private_data` 里的 `default` / `default_null`（`dict0dd.cc:2420-2458`）。

### 2.5 ★★ InnoDB 的 `se_private_data` 全键名表

**键名数组全部定义在 `storage/innobase/include/dict0dd.h`**（⚠️ 不是 `storage/innobase/dict/dict0dd.h`）。枚举与字符串数组严格一一对应：

```cpp
/* dd::Table —— 5 个键 */
enum dd_table_keys {
  DD_TABLE_AUTOINC,           // Auto-increment counter
  DD_TABLE_DATA_DIRECTORY,    // DATA DIRECTORY
  DD_TABLE_VERSION,           // Dynamic metadata version
  DD_TABLE_DISCARD,           // Discard flag（★ 别直接用，见下文）
  DD_TABLE_INSTANT_COLS,      // Columns before first instant ADD COLUMN
  DD_TABLE__LAST
};
const char *const dd_table_key_strings[DD_TABLE__LAST] = {
    "autoinc", "data_directory", "version", "discard", "instant_col"};
```

```cpp
/* dd::Tablespace —— 6 个键 */
enum dd_space_keys {
  DD_SPACE_FLAGS, DD_SPACE_ID, DD_SPACE_DISCARD,
  DD_SPACE_SERVER_VERSION, DD_SPACE_VERSION, DD_SPACE_STATE,
  DD_SPACE__LAST
};
const char *const dd_space_key_strings[DD_SPACE__LAST] = {
    "flags", "id", "discard", "server_version", "space_version", "state"};
```

`state` 键的取值（注意数组长度是 `__LAST + 1`，多一个 `"unknown"` 兜底）：

```cpp
enum dd_space_states {
  DD_SPACE_STATE_NORMAL, DD_SPACE_STATE_DISCARDED, DD_SPACE_STATE_CORRUPTED,
  DD_SPACE_STATE_ACTIVE, DD_SPACE_STATE_INACTIVE, DD_SPACE_STATE_EMPTY,
  DD_SPACE_STATE__LAST
};
const char *const dd_space_state_values[DD_SPACE_STATE__LAST + 1] = {
    "normal", "discarded", "corrupted",   /* for IBD spaces */
    "active", "inactive", "empty",        /* for undo spaces */
    "unknown"};                           /* for non-existing or unknown spaces */
```

```cpp
/* dd::Index / dd::Partition_index —— 5 个键 */
enum dd_index_keys {
  DD_INDEX_ID, DD_INDEX_SPACE_ID, DD_TABLE_ID, DD_INDEX_ROOT, DD_INDEX_TRX_ID,
  DD_INDEX__LAST
};
const char *const dd_index_key_strings[DD_INDEX__LAST] = {
    "id", "space_id", "table_id", "root", "trx_id"};
/* ★ 注意枚举名是 DD_TABLE_ID 而键名是 "table_id"，写在 index 的 se_private_data 里 */
```

```cpp
/* dd::Column —— 5 个键（全部围绕 instant ADD/DROP COLUMN） */
enum dd_column_keys {
  DD_INSTANT_COLUMN_DEFAULT,      // Default value when it was added instantly
  DD_INSTANT_COLUMN_DEFAULT_NULL, // Default value is null or not
  DD_INSTANT_VERSION_ADDED,       // Row version when this column was added
  DD_INSTANT_VERSION_DROPPED,     // Row version when this column was dropped
  DD_INSTANT_PHYSICAL_POS,        // Column physical position on row
  DD_COLUMN__LAST
};
const char *const dd_column_key_strings[DD_COLUMN__LAST] = {
    "default", "default_null", "version_added", "version_dropped",
    "physical_pos"};
```

```cpp
/* dd::Partition —— 3 个键 */
enum dd_partition_keys {
  DD_PARTITION_ROW_FORMAT, DD_PARTITION_INSTANT_COLS, DD_PARTITION_DISCARD,
  DD_PARTITION__LAST
};
const char *const dd_partition_key_strings[DD_PARTITION__LAST] = {
    "format", "instant_col", "discard"};
```

> ★ **`discard` 键在 `dd::Table` 和 `dd::Partition` 里同名**，源码注释专门警告不要直接访问，要走封装函数：

```cpp
  /** Discard flag. Please don't use it directly, and instead use
  dd_is_discarded and dd_set_discarded functions. Discard flag is defined
  for both dd::Table and dd::Partition and it's easy to confuse.
  The functions will choose right implementation for you, depending on
  whether the argument is dd::Table or dd::Partition. */
```

写入侧与读取侧的用法：

```cpp
  dd::Properties &p = dd_index->se_private_data();
  p.set(dd_index_key_strings[DD_INDEX_ID], index->id);
  p.set(dd_index_key_strings[DD_INDEX_SPACE_ID], index->space);
  p.set(dd_index_key_strings[DD_TABLE_ID], index->table->id);
  p.set(dd_index_key_strings[DD_INDEX_ROOT], index->page);
  p.set(dd_index_key_strings[DD_INDEX_TRX_ID], index->trx_id);
```

```cpp
  return (table.se_private_data().exists(
      dd_table_key_strings[DD_TABLE_INSTANT_COLS]));
```

⇒ **这套"枚举 + 平行字符串数组"是 InnoDB 侧的固定手法**（`dd_space_key_strings`、`dict0stats.cc` 的 `stat_name` 等），好处是键名不会拼错；代价是枚举与数组必须同步维护（靠 `__LAST` 哨兵 + 数组长度编译期检查）。

`se_private_data` 是**引擎在 DD 对象上挂的私有背包**（`dd::Properties`，键值均为字符串）。InnoDB 用的键全部硬编码在 `dict0dd.h` 的枚举 + 字符串数组里。这是排查"字典里到底存了什么"最直接的清单。

#### `dd::Table::se_private_data`（`dict0dd.h:100-116` / 键名 `:234-235`）

```cpp
enum dd_table_keys {
  DD_TABLE_AUTOINC,          // "autoinc"         自增计数器
  DD_TABLE_DATA_DIRECTORY,   // "data_directory"  DATA DIRECTORY
  DD_TABLE_VERSION,          // "version"         动态元数据版本
  DD_TABLE_DISCARD,          // "discard"         DISCARD TABLESPACE 标志
  DD_TABLE_INSTANT_COLS,     // "instant_col"     instant ADD 前的列数（仅 V1）
  DD_TABLE__LAST
};
const char *const dd_table_key_strings[DD_TABLE__LAST] =
    {"autoinc", "data_directory", "version", "discard", "instant_col"};
```

#### `dd::Index::se_private_data`（`dict0dd.h:247-265`）

```cpp
enum dd_index_keys {
  DD_INDEX_ID,      // "id"       索引 id
  DD_INDEX_SPACE_ID,// "space_id" 表空间 id
  DD_TABLE_ID,      // "table_id" 表 id
  DD_INDEX_ROOT,    // "root"     ★ root 页号
  DD_INDEX_TRX_ID,  // "trx_id"   创建中的事务 id（崩溃恢复判断索引是否建完）
  DD_INDEX__LAST
};
```

★★ **`root` 键是"索引根页号"的唯一权威来源**——[`tablespace.md`](../../innodb/physical/tablespace.md) 里说"定位 root 页的唯一可靠途径是数据字典，不能把页 3 当常量"，具体就是读 `dd::Index::se_private_data["root"]`。同理 `space_id` / `table_id` / `id` 也在这里。

#### `dd::Column::se_private_data`（`dict0dd.h:238-240`）

```cpp
{"default", "default_null", "version_added", "version_dropped", "physical_pos"}
```

| 键 | 用途 |
|---|---|
| `default` | instant ADD COLUMN 的**默认值**（InnoDB 自己用的二进制形式） |
| `default_null` | 该默认值是否为 NULL |
| `version_added` | 该列是在第几次 instant DDL 时**加入**的 |
| `version_dropped` | 该列是在第几次 instant DDL 时**删除**的 |
| `physical_pos` | ★ **物理位置**（instant DROP 后逻辑列号与物理列号会错位，靠它换算） |

★ 这五个键合起来就是 **instant ADD/DROP COLUMN 元数据的全部持久化内容**——配合表级的 `instant_col`，构成 [`innodb_dict.md`](innodb_dict.md) 里那套"三组计数器 / 双坐标体系"的持久侧。

#### `dd::Partition::se_private_data`（`dict0dd.h:153-169` / 键名 `:243-244`）

```cpp
{"format", "instant_col", "discard"}
```

★ **`instant_col` 在分区级又出现一次**，源码注释（`:156-160`）解释了原因：

> This is necessary for each partition because different partition may have different instant column numbers, especially, for a newly truncated partition, it can have no instant columns. **So partition level one should be always >= table level one.**

即：**每个分区独立记录，且分区级的值恒 >= 表级**——因为 `TRUNCATE` 一个分区会抹掉它的 instant 列。

#### ★ 一个值得学的工程实践：`discard` 的封装

`discard` 键在 `dd::Table` 和 `dd::Partition` 上**都存在且语义不同**。源码注释（`:107-111`、`:162-166`）两次强调：

> Please don't use it directly, and instead use `dd_is_discarded` and `dd_set_discarded` functions. Discard flag is defined for both `dd::Table` and `dd::Partition` and **it's easy to confuse**. The functions will choose right implementation for you.

即：**用重载函数（`dd::Table&` / `dd::Partition&` 两个版本）替代直接按键访问**，让编译器替你选对对象。这是 DD 里"同名键、不同宿主"问题的标准解法。

#### `se_private_data` 的读写方式

写（`dict0dd.cc:1643`）：

```cpp
dd::Properties &se_private_data = table->se_private_data();
se_private_data.set(dd_table_key_strings[DD_TABLE_VERSION], version);
se_private_data.set(dd_table_key_strings[DD_TABLE_AUTOINC], autoinc);
```

读（`dict0dd.ic:345`）：

```cpp
if (!p.exists(dd_table_key_strings[DD_TABLE_VERSION]) ||
    p.get(dd_table_key_strings[DD_TABLE_VERSION],
          reinterpret_cast<uint64_t *>(&version))) {
  return 0;      // 键不存在或类型不符都返回 0
}
```

⚠️ 注意 `get()` 返回**错误码**而不是值，值通过出参给——且必须配合 `exists()` 先判存在（否则老版本升级上来的对象没有该键）。这套"字符串键值 + 出参 + 错误码"的接口比直接读字段啰嗦得多，是 `se_private_data` 使用起来最容易写错的地方。

---

### 2.6 ★ `dd_table.cc`：SQL 层建表信息 → `dd::Table` 的往返

> `sql/dd/dd_table.cc` 是 `sql/dd/` 下**最大的文件**（111 KB / 57 个函数），也是 `CREATE TABLE` 落到 `mysql.tables` / `columns` / `indexes` … 的**唯一**写入路径——所有 DDL 都过它。

#### 2.6.0 先认清它的位置：它是"对象树工厂"，**不写 DD**

`mysqld.cc` 里有官方 doxygen 主干图：

```
  Execution of a CREATE TABLE statement invokes
  Sql_cmd_create_table::execute(), which in turns calls:
  - mysql_create_table(), - mysql_create_table_no_lock(),
  - create_table_impl(), - rea_create_base_table().
  ... and eventually leads to:
  - dd::create_table(), to create the table in the Data Dictionary,
  - ha_create_table(), to create the table in the handlerton.
```

`dd::create_table()` 只做两件事：查 `dict->get_dd_table()` 判断系统表与否，然后分派：

```cpp
  return dd_table
             ? create_dd_system_table(thd, sch_obj, table_name, create_info, ...)
             : create_dd_user_table(thd, sch_obj, table_name, create_info, ...);
```

★ **真正的落盘在 `rea_create_base_table()`（`sql/sql_table.cc`）**：

```cpp
  } else {
    bool result = thd->dd_client()->store(table_def_res.get());
    if (!(create_info->db_type->flags & HTON_SUPPORTS_ATOMIC_DDL) &&
        !thd->is_plugin_fake_ddl())
      result = trans_intermediate_ddl_commit(thd, result);
    ...
  }
```

⇒ **`dd_table.cc` 里没有任何 `trans_commit`、也没有 `acquire_lock`**——事务边界全在 `sql_table.cc`，MDL 只是 3 处 `assert`（"调用方必须已持有"）。这是理解 DDL 分层的关键：**DD 侧只负责"构造对象树"，事务与锁由 SQL 层管**。

头文件里还挂着一条失败契约：

```cpp
  @note The caller must rollback both statement and transaction on failure,
        before any further accesses to DD. This is because such a failure
        might be caused by a deadlock, which requires rollback before any
        other operations on SE ...
```

#### 2.6.1 入口函数族（server → DD 的公共 API）

| 函数 | 作用与差异 |
|---|---|
| `create_table()` | **总入口/分派器**，不写 DD |
| `create_dd_user_table()` | 普通用户表；额外：`set_hidden(HT_HIDDEN_DDL \| HT_VISIBLE)`、PS 表打版本 |
| `create_dd_system_table()` | DD 自己的表，见下方 4 点差异 |
| `create_tmp_table()` | 临时表：`set_is_temporary(true)`、**不传外键**；注释明说 "Doesn't update DD tables, instead returns dd::Table object to caller" |
| `drop_table()` | 删 DD + `remove_table_dynamic_statistics` **两步** |
| `table_exists()` | 注释："Tables exist if they can be acquired." → 靠 `acquire()` 成功与否（表/视图都算） |
| **`recreate_table()`** | ★ **方向相反**：DD 定义 → 重建 SE 侧表。唯一调用点在 `sql_admin.cc` 的 **REPAIR ... USE_FRM**；要求持有排他 MDL；`HA_CREATE_INFO` 是**默认构造的空对象**（不走 `fill_dd_table_from_create_info`）。该文件还断言 `!(ht->flags & HTON_SUPPORTS_ATOMIC_DDL)`——**原子 DDL 引擎不会走这条路** |

**`create_dd_system_table` 与用户表的 4 点差异**：

1. `hidden` 取 `HT_HIDDEN_SYSTEM`（按 `dd_table.is_hidden()`），用户表取 `HT_HIDDEN_DDL` / `HT_VISIBLE`
2. `keys_onoff` **硬编码 `Alter_info::ENABLE`**（用户表透传参数）
3. ★ **`se_private_data` 来源不同**：系统表走 `file->ha_get_se_private_data()`（引擎硬编码）或 `get_se_private_data()` 从 **`dd_properties` 表**读预先硬编码的 `ID / SPACE_ID / DATA / IDX<n> / COL<n>`
4. 额外 `ddse->dict_register_dd_table_id(tab_obj->se_private_id())` 向 DDSE 注册 id

#### 2.6.2 `fill_dd_table_from_create_info`：`mysql.tables` 那一行

**engine**——分区表走另一条分支取 hton：

```cpp
  handlerton *hton = thd->work_part_info
                         ? thd->work_part_info->default_engine_type
                         : create_info->db_type;
  tab_obj->set_engine(ha_resolve_storage_engine_name(hton));
```

⚠️ 分区表时 `create_info->db_type` 可能是分区引擎，真正引擎在 `work_part_info->default_engine_type`。且 DD 存的是**引擎名字符串**，不是 enum。

**★ `options` 全表**（`dd::Properties` → `mysql.tables.options MEDIUMTEXT`）：

| key | 来源 | 判定 |
|---|---|---|
| `max_rows` / `min_rows` | `create_info->*` | 为 0 则不写 |
| **`pack_record`** | `HA_OPTION_PACK_RECORD` | **无条件写**（理由见下） |
| `pack_keys` / `checksum` / `delay_key_write` / `stats_persistent` | 对应 flag | 仅当显式指定；不存在 = DEFAULT |
| `avg_row_length` / `stats_auto_recalc` / `key_block_size` | 直接搬 | |
| `row_type` | `create_info->row_type` | **仅当 `!= ROW_TYPE_DEFAULT`** |
| `stats_sample_pages` | `create_info->stats_sample_pages & 0xffff` | ⚠️ **静默截断到 16 位** |
| `encrypt_type` | `create_info->encrypt_type` | 引擎支持且非临时表；默认 `"N"` |
| `storage` / `connection_string` / `compress` | LEX_STRING | 非空才写 |
| `keys_disabled` | `keys_onoff == DISABLE` | |
| `secondary_engine` + `secondary_load` | 指定才写 | |
| **`gipk`** | GIPK 判定 | 见下 |

**为什么 `pack_record` 无条件存**（典型"避免二进制兼容性"设计）：

```cpp
  /*
    Even though we calculate HA_OPTION_PACK_RECORD flag from the value of
    ROW_FORMAT option and column types, it doesn't really reflect property
    of table, but rather our decision to apply optimization in some cases.
    So it is better to store this flag explicitly in DD in order to avoid
    problems with binary compatibility if we decide to change rules for
    applying this optimization in future releases.
  */
```

**防御性断言**（防止有未处理的 flag 漏掉，对应 Bug#22227958——同一条 CREATE TABLE 里写冲突的 `PACK_KEYS=0 PACK_KEYS=1` 等）：

```cpp
  /* We should not get any unexpected flags which are not handled below. */
  assert(!(create_info->table_options & ~(HA_OPTION_PACK_RECORD | ... )));
```

**★ `row_type`(options) vs `row_format`(列)——最易混淆的一处**：

```cpp
  // ROW_FORMAT which was explicitly specified by user (if any).
  if (create_info->row_type != ROW_TYPE_DEFAULT)
    table_options->set("row_type", dd_get_new_row_format(create_info->row_type));

  // ROW_FORMAT which is really used for the table by SE (perhaps implicitly).
  tab_obj->set_row_format(
      dd_get_new_row_format(file->get_real_row_type(create_info)));
```

- `options.row_type` = 用户**显式**写的
- `mysql.tables.row_format` 列 = 引擎**实际**用的

来源不同（`create_info->row_type` vs `file->get_real_row_type()`），转换靠 `dd_get_new_row_format()`。

**collation**：只存 id（`set_collation_id(create_info->default_table_charset->number)`），**DD 不单独存 charset_id**——字符集由 collation 反查。

**GIPK 三处打标**（表 / 主键索引 / 主键列各打一个 `gipk` option）：

```cpp
  /*
    If table definition has a generated invisible primary key then set "gipk"
    option for the table, primary key and key column.
    At this point we use these "gipk" flags only in INFORMATION_SCHEMA
    implementation to filter out information about the GIPK in
    show_gipk_in_create_table_and_information_schema=OFF mode.
  */
```

**★ 调用顺序本身携带语义**——SE 隐藏列必须先于分区处理：

```cpp
  /*
    Add hidden columns and indexes which are implicitly created by storage
    engine for the table. This needs to be done before handling partitions
    since we want to create proper dd::Index_partition objects for such
    indexes.
  */
```

完整顺序：`set_name → set_engine → set_comment → options → set_row_format → set_collation_id → ①columns → SRS 校验 → ②indexes → ③check_constraints → ④foreign_keys（仅 `HTON_SUPPORTS_FOREIGN_KEYS`）→ ⑤tablespace → 保留表空间校验 → GIPK → SE 隐藏列 → ⑥partitions`。

#### 2.6.3 `fill_dd_columns_from_create_fields`：`Create_field` → `dd::Column`

⚠️ 它是 7 个函数里**唯一非 static 且进公共头文件**的——因为**视图也要用**（`dd_view.cc` 直接调它）。

**前置环境三个 trick**：

- 造 `fake TABLE + TABLE_SHARE` 只为生成默认值
- 缓冲区 `1 + max_pack_length(create_fields)`，且**至少 2 字节**（注释：读 preamble 的 leftover bits 时 `get_rec_buf()` 假设 ≥2）
- 整个函数期间 `check_for_truncated_fields` 被改成 `CHECK_FIELD_WARN`（默认值转换错误**降级为 warning**），靠 RAII 恢复

**★ 双份文本模式**（与 2.4 节呼应，这里是**产生点**）：

```cpp
    if (field.gcol_info) {
      /*
        It is important to normalize the expression's text into the DD, to
        make it independent from sql_mode. For example, 'a||b' means 'a OR b'
        or 'CONCAT(a,b)', depending on if PIPES_AS_CONCAT is on. Using
        Item::print(), we get self-sufficient text containing 'OR' or 'CONCAT'.
        If sql_mode later changes, it will not affect the column.
       */
      field.gcol_info->print_expr(thd, &gc_expr);          // bin 原文
      col_obj->set_generation_expression(...);
      convert_and_print(&gc_expr, &gc_expr_for_IS, system_charset_info);
      col_obj->set_generation_expression_utf8(...);        // utf8 副本
    }
```

即：**先 `Item::print()` 归一化（消除 sql_mode 依赖），再 `convert_and_print` 转 utf8 存第二份**。check 约束与分区表达式是**完全同构**的处理。

**hidden 列（4 种，不是 5 种）**：

```cpp
  // Check that the hidden type isn't the type that is used internally by
  // storage engines.
  assert(field.hidden != dd::Column::enum_hidden_type::HT_HIDDEN_SE);
```

- 列级：`HT_VISIBLE` / `HT_HIDDEN_SE` / `HT_HIDDEN_SQL`（函数索引隐藏列）/ `HT_HIDDEN_USER`（INVISIBLE 列、GIPK 列）
- ⚠️ **"System" 是表级不是列级**（表级：`HT_VISIBLE` / `HT_HIDDEN_SYSTEM` / `HT_HIDDEN_SE` / `HT_HIDDEN_DDL`）
- ⇒ SQL 层来的列**不可能是 `HT_HIDDEN_SE`**，SE 隐藏列只能由 `get_extra_columns_and_keys()` 后续加

**★ `numeric_scale` 被设置了两次**（两处逻辑重复，靠 assert 保证一致）：

第一次 `get_field_numeric_scale()`；第二次一个 switch，注释说明意图：

```cpp
    /*
      Store numeric scale for types relying on this info (old and new decimal
      and floating point types). Also store 0 for integer types to simplify I_S
      implementation.
    */
```
```cpp
      case MYSQL_TYPE_FLOAT:
      case MYSQL_TYPE_DOUBLE:
        /* For these types we show NULL in I_S if scale was not given. */
        if (field.decimals != DECIMAL_NOT_SPECIFIED)
          col_obj->set_numeric_scale(field.decimals);
        else { assert(col_obj->is_numeric_scale_null()); }
```

**ENUM / SET 元素**：`add_element()` 逐个写进 `mysql.column_type_elements`，且 **把结尾 `'\0'` 也算进长度**：

```cpp
        //  Copy type_lengths[i] bytes including '\0'
        //  This helps store typelib names that are of different charsets.
```

元素个数还冗余存一份 `options.interval_count`。

#### 2.6.4 `fill_dd_indexes_from_keyinfo`：`KEY` → `dd::Index`

**★ 主键判定不看 flag，看名字**：

```cpp
static dd::Index::enum_index_type dd_get_new_index_type(const KEY *key) {
  if (key->flags & HA_FULLTEXT) return dd::Index::IT_FULLTEXT;
  if (key->flags & HA_SPATIAL) return dd::Index::IT_SPATIAL;
  if (key->flags & HA_NOSAME) {
    /*
      mysql_prepare_create_table() marks PRIMARY KEY by assigning
      KEY::name special value. We rely on this here and in several
      other places in server (e.g. in sort_keys()).
    */
    if (key->name == primary_key_name) return dd::Index::IT_PRIMARY;
    else return dd::Index::IT_UNIQUE;
  }
  return dd::Index::IT_MULTIPLE;
}
```

即：靠 **`key->name == "PRIMARY"`** 的字符串比较——一条跨函数的隐式契约。

**`ordinal_position` 不是显式 set 的**，由 `Collection::push_back` 自动编号：

```cpp
  void push_back(impl_type *item) {
    item->set_ordinal_position(static_cast<unsigned int>(m_items.size() + 1));
    m_items.push_back(item);
  }
```

⇒ **`add_index()` 的调用顺序 == ordinal_position**（1-based）。

**index options 只存 3 项**，长注释解释为什么（与 `pack_record` 同理：只存"我们的决定"，不存可推导属性）：

```cpp
    /*
      Most of flags in KEY::flags bitmap can be easily calculated from other
      attributes ... so we avoid storing this redundant information in DD.

      HA_PACK_KEY and HA_BINARY_PACK_KEY are special ... they, unlike other
      flags, do not reflect immanent property of key ... but rather reflect
      our decision to apply certain optimization ...
    */
    idx_options->set("flags", (key->flags & (HA_PACK_KEY | HA_BINARY_PACK_KEY)));
```

**★ 降序索引的"假 TABLE_SHARE"hack**（一段超长注释，含 TODO）：

```cpp
  /** Currently the index order type is not persisted in new DD or in .FRM. ...
    But index_flags() implementations mostly uses algorithm and flags members
    of keyinfo to get the flag value. ...
    TODO:Refine approach during the complete WL6599 review by dlenev.
  */
```

实现：`file->get_table_share() == nullptr` 时**造一个 `TABLE_SHARE dummy_table_share`** 挂上 keyinfo 并 `change_table_ptr()`，用完还原。

⚠️ 这个 hack 会迫使引擎在**没有真实表对象**的情况下回答 `index_flags()`——NDB 为此专门做了兜底，其源码注释直接点名本函数：

```cpp
  ... It also happens in DD code as discussed in a long comment at
  fill_dd_indexes_from_keyinfo() in dd_table.cc. ...
```

**★ 主键提升（`is_candidate_primary_key`）——最容易误解的一处**：

无显式主键时，会把**第一个符合条件的 UNIQUE 键"提升"为主键**：

```cpp
    /*
      If we have no primary key, then we pick the first candidate primary
      key and promote it. When we promote, the field's of key_part needs to
      be marked as PRIMARY.
    */
    if (primary_key_info == nullptr &&
        is_candidate_primary_key(thd, key, create_fields)) {
      primary_key_info = key;
    }
```

准入（`is_candidate_primary_key`）：UNIQUE 且无 `HA_NULL_PART_KEY`、非虚拟生成列索引、非数组列，且每个 key part 通过 `is_suitable_for_primary_key()`（非 NULL、**非前缀索引**；NOT NULL BLOB 有特例）。

⚠️ **提升只改 `dd::Column::column_key`（标 `CK_PRIMARY`），`dd::Index::type()` 仍是 `IT_UNIQUE`**——DD 里"这列是主键列"与"这个索引是主键索引"**可以不一致**。这是 I_S / ORM 读取时的经典陷阱（`has_primary_key()` 只认真正的 `IT_PRIMARY`）。

另：`primary_key_info` 一旦设置就不再换 ⇒ **第一个**满足条件的唯一键胜出。

`column_key` 的标记规则：只有**首列**（`key_part == key_parts`）才标记；复合唯一键的首列记 **`CK_MULTIPLE`** 而非 `CK_UNIQUE`。

#### 2.6.5 外键 / check 约束 / 分区

**外键**：本表列用 **DD 对象指针**，被引用列只存**名字字符串**：

```cpp
      const dd::Column *column = tab_obj->get_column(...);
      assert(column);
      fk_col_obj->set_column(column);
      fk_col_obj->referenced_column_name(...);
```
⚠️ 一端强引用（查不到就 assert 崩），另一端弱引用。另有一条约定：**空串 == NULL**（`set_unique_constraint_name(key->unique_index_name ? ... : "")`）。

**check 约束**：与生成列完全同构的双份文本（`check_clause` bin + `check_clause_utf8`），`alias_name` 只在 ALTER 场景写。

**分区**：
- 表达式用 `Item::print()`，且有一段**生命周期 hack**（预编译语句下 `Item_field::field` 可能失效，临时置 nullptr）
- `Sql_mode_parse_guard` 关掉影响打印的 sql_mode
- 长度硬限制 `PARTITION_EXPR_CHAR_LEN`，超了报 `ER_PART_EXPR_TOO_LONG`
- 跳过 `PART_TO_BE_DROPPED` / `PART_REORGED_DROPPED`
- `Partition_index` 是**分区 × 索引的笛卡尔积**
- 开头挂着 `// TODO-PARTITION: move into partitioning service, WL#4827`

#### 2.6.6 ★ 类型转换族（最容易出错的地方）

**① `get_sql_type_by_create_field`：`Field` → `column_type_utf8`**

```cpp
  unique_ptr_destroy_only<Field> fld(make_field(field, table->s));
  fld->init(table);
  String type(tmp, sizeof(tmp), system_charset_info);
  fld->sql_type(type);
```

即：**先造一个真实 `Field` 再调 `Field::sql_type()` 打印**——复用"把列类型打印成 DDL 文本"的那套代码。

**为什么存文本**（注释原文）：`"This is required for the IS implementation which uses views on DD"`。8.0 的 I_S 是 DD 表上的视图，`COLUMNS.COLUMN_TYPE` 要返回 `int(11) unsigned zerofill` 这种完整串，而**这个串没法用 SQL 表达式算出来**，所以只能在写 DD 时**物化一份文本**。DD 里同时有 `columns.type` 枚举——**枚举给机器，文本给 I_S**。

⚠️ 还有**第二条生成路径** `get_sql_type_by_field_info()`（用 `init_for_tmp_table()` 简化构造，拿不到 zerofill / 显式 collation / ENUM 值列表），给硬编码 `ST_FIELD_INFO` 的列用——两条路对同一逻辑类型可能产出不同文本。

**② `prepare_default_value_string`：默认值 → `default_value_utf8`**

三分支：

- **DEFAULT (expr)**：从 `col_obj->default_option()` **反读回来**再 `convert_and_print` 转 utf8（⇒ 与先前的 `set_default_option(print_expr(...))` **顺序耦合**）
- **DEFAULT NOW()**：`"CURRENT_TIMESTAMP"`（带小数秒则补括号）
- **字面量**：再造一个 fake Field 去读 buf，然后 `val_str()` + 转 `system_charset_info`（**转换错误被丢弃**：`dummy_errors`）

★ **BINARY / VARBINARY 强制转 HEX——源码明确承认的有损往返**：

```cpp
        /*
          ... Converting hex string to system_charset_info charset while
          storing value in DD table might fail because of unsupported byte
          value in hex string. Hence converting default value to printable
          HEX encoded string before store.

          The original format as supplied by user is lost after parsing stage.
          So regardless of the type specified by the user, default for
          varbinary/binary is stored in the printable HEX encoded format.
          I_S queries and SHOW COLUMNS always list such default value in HEX
          format instead of user specified one.
        */
```

⚠️ 生成列 / 无默认值时**直接 return**，`default_value_utf8` 是 **NULL 而不是空串**。

**③ collation**：`set_collation_id(field.charset->number)`，外加 `set_is_explicit_collation()` 记录"用户是否显式写了 COLLATE"（否则 SHOW CREATE 会因表默认字符集变化而改写列排序规则）。

#### 2.6.7 ★ `se_private_data` **不在这里填**

这是本节最需要澄清的一点：`fill_dd_table_from_create_info` **完全不碰 `se_private_data`**。它分两条路：

**① 用户表：由 SE 在 `ha_create()` 内部回填，之后 server 更新回 DD**——`handler.cc` 的注释是硬证据：

```cpp
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
```

即：**只有支持原子 DDL 的引擎才能改 `dd::Table`，且改完由 server `dd_client()->update()` 落盘**。

**② 系统表**：`ha_get_se_private_data()`（引擎硬编码）或 `get_se_private_data()`（从 `dd_properties` 表读）。后者注释里有一条跨层硬编码约束：

```cpp
  @note During restart, when the scaffolding is created, we assign the
  DD tablespace id (==1) even though this is different that the scaffolding
  DD tablespace id. This is because both server code and innodb code has
  hard coded expectations regarding the DD tablespace id.
```

#### 2.6.8 已知坑与有损往返

| # | 坑 | 说明 |
|---|---|---|
| 1 | **BINARY/VARBINARY 默认值强制 HEX** | 源码注释承认"用户原始格式丢失"，I_S/SHOW 永远显示 HEX |
| 2 | **`column_type_utf8` 是"再生成"不是"原样保存"** | `Create_field → Field → 文本 → 存 → 读 → 解析`；且有第二条生成路径，两者可能不一致 |
| 3 | **`char_length` 存的是字节** | `set_char_length(field.max_display_width_in_bytes())`，但 I_S 前缀键长度按字符算（`length() / cs->mbmaxlen`）——两个量纲混用 |
| 4 | **`stats_sample_pages` 静默截断** | `& 0xffff`，高 32 位直接丢，无注释无报错 |
| 5 | **重复赋值** | `set_order()`、`set_visible()`、`geom_type` 都被写了两次（后者无害，前两者第二次覆盖第一次） |
| 6 | **主键提升造成的不一致** | 列标 `CK_PRIMARY` 但索引仍是 `IT_UNIQUE` |

**TODO 清单**（源码原文）：`// TODO: Avoid exposing dd/impl headers in public files.`、`TODO-PARTITION: move into partitioning service, WL#4827`、`// TODO-POST-MERGE-TO-TRUNK: Initialize new field tab_obj->last_checked_for_upgrade`、`// TODO-MYSQL_VERSION: We decided not to store MYSQL_VERSION_ID ?`（后跟一整段解释：不能把它当二进制格式版本号，因为 ALTER 常须用旧格式建表以避免重建）。

### 2.7 ★ `fill_column_from_dd()`：`dd::Column` → `Field`

> 本章从 **DD 视角**回答："DD 里到底存了什么，以及它如何被翻译成 server 侧的 `Field`"。完整调用链是 `open_table_def()` → `fill_share_from_dd()`（算尺寸）→ `fill_columns_from_dd()`（逐列摆位）→ 本节主角。
>
> 补充源码坐标：`[F13]` sql/dd_table_share.h、`[F11]` sql/dd/impl/types/column_impl.cc、`[F12]` sql/dd/impl/types/column_impl.h、`[F28]` sql/dd/dd_table.cc、`[F15]` sql/default_values.cc。
> ★ `Field` 本身的完整剖析见 [`../field.md`](../field.md)（本篇只讲 DD 侧）。

#### 2.7.1 它在链路上的位置，以及一个极易忽略的前提

```
open_table_def(THD*, TABLE_SHARE*, const dd::Table &)      [F14]
  ├─ fill_share_from_dd()         // 表级属性 + find_record_length() 算出 null_bytes/reclength
  ├─ fill_columns_from_dd()       // ★ 遍历 dd::Table::columns()，逐个调 fill_column_from_dd()
  ├─ fill_indexes_from_dd() / fill_partitioning_from_dd() / ...
  └─ prepare_share()
```

```cpp
// [F14] sql/dd_table_share.cc —— open_table_def()
bool open_table_def(THD *thd, TABLE_SHARE *share, const dd::Table &table_def) {
  MEM_ROOT *old_root = thd->mem_root;
  thd->mem_root = &share->mem_root;  // Needed for make_field()++
  share->blob_fields = 0;            // HACK
  ...
}
```

★ **`make_field()` 内部大量用 `*THR_MALLOC`（即 `thd->mem_root`）做 `new (mem_root) Field_xxx(...)`**，所以整个 `fill_columns_from_dd()` 期间必须把 THD 的 mem_root 临时切到 `share->mem_root`，否则 Field 会随语句结束而失效。

★ **关键前提**：`fill_column_from_dd()` 自己**不计算**记录布局。布局（`null_pos` / `null_bit_pos` / `rec_pos`）由调用方逐列推进，总尺寸（`null_bytes` / `reclength` / `last_null_bit_pos`）早在 `[F15] find_record_length()` 算好。这是"**先量尺寸 → 再逐个摆位**"的两阶段设计。

#### 2.7.2 前置辅助：位预算与 `auto_flags` 还原

**`column_preamble_bits()`**——一列在记录 preamble（NULL 标志位区 + BIT 余位）里占几位：

```cpp
static uint column_preamble_bits(const dd::Column *col_obj) {
  uint result = 0;
  if (col_obj->is_nullable()) result++;        // 一等属性：mysql.columns.is_nullable
  if (col_obj->type() == dd::enum_column_types::BIT) {
    bool treat_bit_as_char = false;
    // ★ treat_bit_as_char 是"非一等公民"，存在 options 里
    (void)col_obj->options().get("treat_bit_as_char", &treat_bit_as_char);
    if (!treat_bit_as_char) result += col_obj->char_length() & 7;
  }
  return result;
}
```

**★ `get_auto_flags()`——"有损压缩 + 启发式还原"的典型**：server 侧用一个 bitmask 表达 4 种"自动"语义，而 DD 把它拆成 `default_option` / `update_option` / `is_auto_increment` 三个字段：

```cpp
inline void get_auto_flags(const dd::Column &col_obj, uint &auto_flags) {
  if (!col_obj.default_option().empty()) {
    /* 只比较前 17 字符：后面可能带小数秒参数，如 "CURRENT_TIMESTAMP(6)"。
       普通字符串默认值不会误匹配（它们在 DD 里被写成带字符集前缀的形式）。 */
    if (col_obj.default_option().compare(0, 17, "CURRENT_TIMESTAMP") == 0) {
      // 合法形态只有：长度恰好 17 的裸形式，或 >=20 且形如 "CURRENT_TIMESTAMP(<整数>)"
      assert(col_obj.default_option().size() == 17 ||
             (col_obj.default_option().size() >= 20 &&
              col_obj.default_option()[17] == '(' &&
              col_obj.default_option()[col_obj.default_option().size() - 1] == ')'));
      auto_flags |= Field::DEFAULT_NOW;
    } else {
      auto_flags |= Field::GENERATED_FROM_EXPRESSION;   // 非 CURRENT_TIMESTAMP ⇒ 表达式
    }
  }
  if (!col_obj.update_option().empty()) auto_flags |= Field::ON_UPDATE_NOW;
  if (col_obj.is_auto_increment())      auto_flags |= Field::NEXT_NUMBER;
  assert(!((auto_flags & (DEFAULT_NOW | ON_UPDATE_NOW | GENERATED_FROM_EXPRESSION)) &&
           (auto_flags & NEXT_NUMBER)));        // AUTO_INCREMENT 与它们互斥
}
```

对应 server 侧枚举（`[F1] sql/field.h`）：`NONE=0 / NEXT_NUMBER=1 / DEFAULT_NOW=2 / ON_UPDATE_NOW=4 / GENERATED_FROM_EXPRESSION=8`。

#### 2.7.3 `make_field(const dd::Column&, ...)`：options 的读取都在这里

这是 `dd::Column` → 工厂参数 的适配层，**所有 options（非一等公民属性）都在此被读出**，然后转交底层工厂 `make_field(MEM_ROOT*, ...)`（真正的 `new Field_xxx`，见 [`../field.md`](../field.md) 第八章）：

```cpp
static Field *make_field(const dd::Column &col_obj, const CHARSET_INFO *charset,
                         TABLE_SHARE *share, uchar *ptr, uchar *null_pos,
                         size_t null_bit) {
  auto field_type   = dd_get_old_field_type(col_obj.type());   // 一等：enum_column_types → enum_field_types
  auto field_length = col_obj.char_length();                   // 一等：注意是"字节数"
  const dd::Properties &column_options = col_obj.options();

  auto auto_flags = static_cast<uint>(Field::NONE);
  get_auto_flags(col_obj, auto_flags);                          // 派生：见上

  /* ③ ENUM/SET 元素：★ 8.0 里已是"一等公民"——独立的 dd::Column_type_element
       子对象（物理表 mysql.column_type_elements），不在 options 里。 */
  TYPELIB *interval = nullptr;
  if (field_type == MYSQL_TYPE_ENUM || field_type == MYSQL_TYPE_SET) {
    size_t interval_parts = col_obj.elements_count();
    interval = (TYPELIB *)share->mem_root.Alloc(sizeof(TYPELIB));
    ...
    for (const dd::Column_type_element *ce : col_obj.elements()) {
      uint pos = ce->index() - 1;                              // DD index 是 1-based
      interval->type_lengths[pos] = static_cast<uint>(ce->name().length());
      interval->type_names[pos] = strmake_root(&share->mem_root, ...);
    }
  }

  name = strmake_root(&share->mem_root, col_obj.name().c_str(), ...);   // ④ 列名拷进 share->mem_root

  /* ⑤ decimals 三分支 */
  if (field_type == MYSQL_TYPE_DECIMAL || field_type == MYSQL_TYPE_NEWDECIMAL) {
    assert(col_obj.is_numeric_scale_null() == false);          // DECIMAL 必须有 scale
    decimals = col_obj.numeric_scale();
  } else if (field_type == MYSQL_TYPE_FLOAT || field_type == MYSQL_TYPE_DOUBLE)
    decimals = col_obj.is_numeric_scale_null() ? DECIMAL_NOT_SPECIFIED
                                               : col_obj.numeric_scale();
  else decimals = 0;                                           // 时态列靠长度反解

  auto geom_type = Field::GEOM_GEOMETRY;                        // ⑥ options：几何子类型
  if (field_type == MYSQL_TYPE_GEOMETRY) {
    uint32 sub_type = 0;
    column_options.get("geom_type", &sub_type);
    geom_type = static_cast<Field::geometry_type>(sub_type);
  }

  bool treat_bit_as_char = false;                               // ⑦ options：BIT 按字符存储
  if (field_type == MYSQL_TYPE_BIT)
    column_options.get("treat_bit_as_char", &treat_bit_as_char);

  return make_field(*THR_MALLOC, share, ptr, field_length, null_pos, null_bit,
                    field_type, charset, geom_type, auto_flags, interval, name,
                    col_obj.is_nullable(), col_obj.is_zerofill(),
                    col_obj.is_unsigned(), decimals, treat_bit_as_char, 0,
                    col_obj.srs_id(), col_obj.is_array());
}
```

#### 2.7.4 ★ `fill_column_from_dd()` 完整函数体

```cpp
/**
  Add Field constructed according to column metadata from dd::Column
  object to TABLE_SHARE.
*/
static bool fill_column_from_dd(THD *thd, TABLE_SHARE *share,
                                const dd::Column *col_obj, uchar *null_pos,
                                uint null_bit_pos, uchar *rec_pos, uint field_nr) {
  /* ★ 参数含义（布局由调用方算好）：
       null_pos     —— 本列 NULL 标志位所在字节地址（在 share->default_values 内）
       null_bit_pos —— 位"序号" 0..7（转掩码在 make_field 里做）
       rec_pos      —— 本列数据区起始地址
       field_nr     —— 本列在 share->field[] 的下标（= field_index）
     ⇒ 本函数只"填充内容"，不"推进游标"。 */

  field_type = dd_get_old_field_type(col_obj->type());
  get_auto_flags(*col_obj, auto_flags);

  /* Collation：★ DD 只存"数字 ID"，不是名字 */
  charset = dd_get_mysql_charset(col_obj->collation_id());   // = get_charset(id, MYF(0))
  if (charset == nullptr) {
    my_printf_error(ER_UNKNOWN_COLLATION, "invalid collation id %llu for table %s, column %s",
                    MYF(0), col_obj->collation_id(), share->table_name.str, name);
    if (thd->is_error()) return true;
    charset = default_charset_info;                          // 兜底（抑制元数据错误的场景）
  }

  /* options：storage / column_format（有值才写 options，故先 exists()） */
  if (column_options->exists("storage")) { ...; field_storage = static_cast<ha_storage_media>(v); }
  else field_storage = HA_SM_DEFAULT;
  if (column_options->exists("column_format")) { ... }
  else field_column_format = COLUMN_FORMAT_TYPE_DEFAULT;

  /* ENUM/SET 元素 → TYPELIB（同上，略） */

  // ===== 造 Field =====
  reg_field = make_field(*col_obj, charset, share, rec_pos, null_pos, null_bit_pos);
  reg_field->set_field_index(field_nr);          // ★ m_field_index 的唯一写入点
  reg_field->stored_in_db = true;                // 先假定物理存储，虚拟生成列会被改写

  /* ===== 生成列：判定依据是 generation_expression 是否为 NULL ===== */
  if (!col_obj->is_generation_expression_null()) {
    Value_generator *gcol_info = new (&share->mem_root) Value_generator();
    gcol_info->set_field_stored(!col_obj->is_virtual());      // DD is_virtual → stored_in_db
    dd::String_type gc_expr = col_obj->generation_expression();
    /*
      Place the expression's text into the TABLE_SHARE. Field objects of
      TABLE_SHARE only have that. They don't have a corresponding Item,
      which will be later created for the Field in TABLE...
    */
    gcol_info->dup_expr_str(&share->mem_root, gc_expr.c_str(), gc_expr.length());
    share->vfields++;
    reg_field->gcol_info = gcol_info;
    reg_field->stored_in_db = gcol_info->get_field_stored();  // ★ 虚拟列改写为 false
  }

  /* ===== DEFAULT (expr)：复用 default_option 这个字段存表达式文本 ===== */
  if (auto_flags & Field::GENERATED_FROM_EXPRESSION) {
    Value_generator *default_val_expr = new (&share->mem_root) Value_generator();
    default_val_expr->set_field_stored(true);    // DEFAULT GENERATED is always stored
    dd::String_type default_val_expr_str = col_obj->default_option();
    default_val_expr->dup_expr_str(&share->mem_root, default_val_expr_str.c_str(), ...);
    share->gen_def_field_count++;
    reg_field->m_default_val_expr = default_val_expr;
  }

  if ((auto_flags & Field::NEXT_NUMBER) != 0)
    share->found_next_number_field = &share->field[field_nr];  // ★ 取槽位"地址"（此时尚未赋值）

  if (col_obj->has_no_default()) reg_field->set_flag(NO_DEFAULT_VALUE_FLAG);

  /* ===== ★★ 默认值三分支：把 DD 里的"二进制镜像"直接 memcpy 回记录缓冲 ===== */
  if (col_obj->is_default_value_null()) {
    reg_field->reset();                          // 必须先 reset（CHAR 可能有上一列残留）
    reg_field->set_null();
  } else if (field_type == MYSQL_TYPE_BIT && !treat_bit_as_char &&
             (col_obj->char_length() & 7)) {
    /* BIT 有余位：default_value 的【最后一个字节】是余位，其余是主数据 */
    Field_bit *bitfield = dynamic_cast<Field_bit *>(reg_field);
    const uchar leftover_bits = static_cast<uchar>(
        *col_obj->default_value().substr(reg_field->pack_length() - 1, 1).data());
    set_rec_bits(leftover_bits, bitfield->bit_ptr, bitfield->bit_ofs, bitfield->bit_len);
    memcpy(rec_pos, col_obj->default_value().data(), reg_field->pack_length() - 1);
  } else {
    memcpy(rec_pos, col_obj->default_value().data(), reg_field->pack_length());
  }

  reg_field->set_storage_type(field_storage);
  reg_field->set_column_format(field_column_format);
  /* comment / not_secondary(options) / set_hidden / engine_attribute ×2 */
  reg_field->set_hidden(col_obj->hidden());
  reg_field->m_engine_attribute = LexStringDupRootUnlessEmpty(
      &share->mem_root, col_obj->engine_attribute());
  reg_field->m_secondary_engine_attribute = LexStringDupRootUnlessEmpty(
      &share->mem_root, col_obj->secondary_engine_attribute());

  share->field[field_nr] = reg_field;
  return false;
}
```

**★ 澄清一点**：`fill_column_from_dd()` 自身**不跳过**任何列。跳过发生在调用方（`is_se_hidden()`），且只跳 `HT_HIDDEN_SE`（SE 私有列，如 InnoDB 的 `DB_TRX_ID`）；`HT_HIDDEN_SQL` / `HT_HIDDEN_USER` **照样建出 `Field` 并占槽位**——因为函数索引的隐藏列要参与记录读写。

#### 2.7.5 `fill_columns_from_dd()`：两遍扫描与布局推进

```cpp
  null_flags = null_pos = share->default_values;                  // preamble 从缓冲区开头
  rec_pos = share->default_values + share->null_bytes;            // 记录体紧跟其后
  /* ★ 非 PACK_RECORD 时 preamble 第 0 位被"pack record 标志"占用，故从 1 开始编号 */
  uint null_bit_pos = (share->db_create_options & HA_OPTION_PACK_RECORD) ? 0 : 1;

  for (const dd::Column *col_obj : tab_obj->columns()) {
    if (col_obj->is_se_hidden()) continue;                        // ★ SE 私有列不建 Field

    /* 第一遍只放物理列（含 STORED 生成列），虚拟列要排到记录末尾 ⇒ 第二遍 */
    if (!col_obj->is_virtual()) {
      if (fill_column_from_dd(thd, share, col_obj, null_pos, null_bit_pos, rec_pos, field_nr))
        return true;
      if ((null_bit_pos += column_preamble_bits(col_obj)) > 7) { null_pos++; null_bit_pos -= 8; }
      rec_pos += share->field[field_nr]->pack_length_in_rec();
    } else has_vgc = true;
    field_nr++;                                                   // ★ 虚拟列也要递增（下标按 DDL 顺序）
  }

  if (has_vgc) {
    /* ★ 修正 stored_rec_length：此刻 rec_pos 正好停在物理部分末尾 */
    if (share->stored_rec_length > static_cast<ulong>(rec_pos - share->default_values))
      share->stored_rec_length = (rec_pos - share->default_values);
    field_nr = 0;
    for (const dd::Column *col_obj2 : tab_obj->columns()) {       // 第二遍：只放虚拟列
      if (col_obj2->is_se_hidden()) continue;
      if (col_obj2->is_virtual()) {
        fill_column_from_dd(thd, share, col_obj2, null_pos, null_bit_pos, rec_pos, field_nr);
        if ((null_bit_pos += column_preamble_bits(col_obj2)) > 7) { null_pos++; null_bit_pos -= 8; }
        rec_pos += share->field[field_nr]->pack_length_in_rec();
      }
      field_nr++;
    }
  }

  /* ★ 三个一致性断言：与 find_record_length() 的预算对账 */
  assert(share->null_bytes == (null_pos - null_flags + (null_bit_pos + 7) / 8));
  assert(share->last_null_bit_pos == null_bit_pos);
  assert(share->fields == field_nr);
```

★ **`find_record_length()` 与 `fill_columns_from_dd()` 是同一套算法的两次独立实现**（一次只算长度、一次真摆位），三个 assert 保证它们对 DD 的解读完全一致——任何一边漏掉 `is_se_hidden()` 跳过或漏掉 BIT 余位，assert 立刻失败。

#### 2.7.6 ★ `dd::Column` 到底存了什么 → 翻译成 `Field` 的什么（对照表）

**① 一等属性（`mysql.columns` 的固定列）**

| `dd::Column` 属性 | 存储类型 | 翻译成 Field 的什么 | 备注 |
|---|---|---|---|
| `name()` | String | `Field::field_name` | `strmake_root` 到 `share->mem_root` |
| `type()` | `enum_column_types` | 决定 `new Field_xxx` 的类 | 经 `dd_get_old_field_type()` 映射 |
| `char_length()` | size_t | `Field::field_length` → `pack_length()` | 写入侧是 `max_display_width_in_bytes()`（**字节数**） |
| `collation_id()` | Object_id | `Field::charset()` | **只存数字 ID**，不存名字 |
| `is_nullable()` | bool | `null_pos`/`null_bit` 是否有效 + 占 1 个 preamble 位 | |
| `is_unsigned()` / `is_zerofill()` | bool | `unsigned_flag` / `zerofill` | |
| `is_auto_increment()` | bool | `auto_flags |= NEXT_NUMBER` → `found_next_number_field` | |
| `numeric_scale()` / `is_numeric_scale_null()` | uint / nullable | `Field::decimals()` | DECIMAL 必非空；FLOAT/DOUBLE 可 NULL ⇒ `DECIMAL_NOT_SPECIFIED` |
| `srs_id()` | optional<srid_t> | `Field_geom::m_srid` | |
| `hidden()` | `enum_hidden_type` | `Field::m_hidden`（`set_hidden()`） | `HT_HIDDEN_SE` 在调用方被 `continue` |
| `is_virtual()` | bool | `stored_in_db = !is_virtual()`（仅生成列） | |
| `has_no_default()` | bool | `NO_DEFAULT_VALUE_FLAG` | ★ 与 `is_default_value_null()` 是两个独立概念 |
| `default_value()` | String（**二进制**） | 直接 `memcpy` 到 `rec_pos` | 见三分支；长度 == `pack_length()` |
| `default_value_utf8()` | String（文本） | **不使用**（仅 I_S / SHOW） | |
| `default_option()` | String | `DEFAULT_NOW` 或 `m_default_val_expr` 的文本 | |
| `update_option()` | String | `ON_UPDATE_NOW` | |
| `generation_expression()` | String（**二进制**） | `Value_generator::expr_str` | ★ **只存文本，不存 Item 树** |
| `generation_expression_utf8()` | String | **不使用**（仅 I_S） | |
| `comment()` | String | `Field::comment` | |
| `column_type_utf8()` | String | **不使用**（仅 SHOW CREATE / I_S 显示类型） | |
| `engine_attribute()` / `secondary_engine_attribute()` | LEX_CSTRING | `Field::m_engine_attribute` ×2 | 空串不分配内存 |
| `se_private_data()` | Properties | **不使用** | 只给 SE（InnoDB）读 |
| `elements()` / `elements_count()` | Column_type_element 集合 | `TYPELIB *interval` | ENUM/SET 元素（**独立子表**） |

**② options 属性（"非一等公民"，存在 `mysql.columns.options`）**

合法 key 白名单（`[F11] column_impl.cc`）：

```cpp
static const std::set<String_type> default_valid_option_keys = {
    "column_format", "geom_type", "interval_count",
    "not_secondary", "storage", "treat_bit_as_char",
    "is_array", "gipk" /* generated implicit primary key column */};
```

| key | 读取位置 | 翻译成 | 语义 |
|---|---|---|---|
| `treat_bit_as_char` | `column_preamble_bits()`、`make_field(dd::Column)`、`fill_column_from_dd()` | `Field_bit` vs `Field_bit_as_char`；影响 preamble 是否留余位 | 历史兼容：老 InnoDB 不支持紧凑 BIT 存储 |
| `geom_type` | `make_field(dd::Column)` | `Field_geom::geom_type` | GEOMETRY 子类型 |
| `storage` / `column_format` | `fill_column_from_dd()` | `set_storage_type()` / `set_column_format()` | `STORAGE DISK/MEMORY`、`COLUMN_FORMAT FIXED/DYNAMIC` |
| `not_secondary` | `fill_column_from_dd()` | `NOT_SECONDARY_FLAG` | secondary engine 不复制该列 |
| `is_array` | `dd::Column::is_array()`（**伪装成一等属性**） | `Field_typed_array` | 多值索引的内部列 |
| `interval_count` | **本函数不使用** | — | 冗余；元素本体在 `elements()` |

★ **重要澄清**：用户常以为 ENUM/SET 的 `interval` 元素存在 `options()` 里——8.0 中**不是**，它们是独立的 `dd::Column_type_element` 弱实体（物理表 `mysql.column_type_elements`），`options()` 里只留了一个冗余计数 `interval_count`。

**③ 派生属性（DD 不存，读时算）**

| 派生量 | 计算方式 |
|---|---|
| `Field::auto_flags` | `get_auto_flags()`（见 2.7.2） |
| `TYPELIB *interval` | 遍历 `elements()` 拼装 |
| `pack_length()` | `calc_pack_length(type, char_length, elements_count, treat_bit_as_char, numeric_scale, is_unsigned)` |
| `null_pos` / `null_bit_pos` / `rec_pos` | `fill_columns_from_dd()` 逐列推进 |
| `is_array()` | `options().exists("is_array") && options().get(...)`（`[F12]`——用一等接口包装 options） |

#### 2.7.7 ASCII 图：`dd::Column` → `Field`

```
        mysql.columns (DD 表)                    mysql.column_type_elements
   ┌──────────────────────────────┐            ┌────────────────────┐
   │ name, type, char_length,     │            │ index, name        │
   │ collation_id, is_nullable,   │            │ (ENUM/SET 元素)    │
   │ is_unsigned, is_zerofill,    │            └─────────┬──────────┘
   │ is_auto_increment,           │                      │ elements()
   │ numeric_scale, srs_id,       │                      │
   │ hidden, has_no_default,      │                      │
   │ default_value(BIN),          │                      │
   │ default_option, update_option│                      │
   │ generation_expression(BIN),  │                      │
   │ comment, engine_attribute,   │                      │
   │ se_private_data(SE 专用),    │                      │
   │ options(Properties/JSON)     │                      │
   └───────────────┬──────────────┘                      │
                   │                                      │
        ┌──────────┴──────────┐                           │
        ▼                     ▼                           ▼
   【A】一等属性        【B】options 属性          【C】派生/二次计算
                        treat_bit_as_char          get_auto_flags()
                        geom_type, storage         auto_flags
                        column_format              TYPELIB 拼装 ←────┘
                        not_secondary              pack_length
                        is_array                   = calc_pack_length(...)
                        └──────┬───────┘                  │
                               └───────────┬──────────────┘
                                           ▼
                        make_field(const dd::Column&, …)   [F14] 适配层
                                           ▼
                        make_field(MEM_ROOT*, …)           [F2] 真正的工厂
                          · nullable ⇒ null_bit 序号→掩码
                          · BIT ⇒ 算 bit_ptr/bit_offset（余位放 preamble）
                          · is_array ⇒ Field_typed_array（底层 JSON 存储）
                          · switch(field_type) ⇒ new Field_xxx(...)
                                           ▼
                                     Field *reg_field
       ┌──────────────────────────────────┼──────────────────────────────────┐
  ① set_field_index(field_nr)                                                 │
  ② stored_in_db = true（虚拟生成列会被改写）                                  │
  ③ 生成列 → Value_generator + dup_expr_str(generation_expression())【只存文本】│
  ④ DEFAULT(expr) → m_default_val_expr + dup_expr_str(default_option())        │
  ⑤ NEXT_NUMBER → found_next_number_field = &field[field_nr]                   │
  ⑥ has_no_default → NO_DEFAULT_VALUE_FLAG                                     │
  ⑦ 默认值三分支写 default_values 缓冲（is_null / BIT 余位 / 整段 memcpy）      │
  ⑧ set_storage_type / set_column_format / comment / set_hidden / attr ×2      │
       └──────────────────────────────────┬──────────────────────────────────┘
                                           ▼
                            share->field[field_nr] = reg_field
                                           │
              ┌────────────────────────────┴────────────────────────────┐
              ▼                                                          ▼
      （此刻仍在 TABLE_SHARE）                        （后续建 TABLE 实例时）
      expr_str 只是文本                        unpack_value_generator() 拿 expr_str
      expr_item == nullptr                     重新解析 ⇒ Item 树（per-TABLE，不共享）
```

#### 2.7.8 反向：`Field` → `dd::Column`

反向路径在 `[F28] sql/dd/dd_table.cc::fill_dd_columns_from_create_fields()`（由 `Create_field` 写回 DD）。两点与本节呼应：

**① 二进制镜像 `default_value()` 如何生成**（`[F15] prepare_default_value()`）——★ 它造一个"临时 Field"（`ptr` 指向 `buf+1`、null 位指向 `buf[0]`），用**真实 Field 的存储逻辑**把默认值序列化成字节：

```cpp
  Field *regfield = make_field(field, table->s, buf + 1, buf, 0 /* null_bit */);
  regfield->init(table);
  if (!(field.flags & NOT_NULL_FLAG)) regfield->set_null();
  ...
  } else if (regfield->real_type() == MYSQL_TYPE_ENUM &&
             (field.flags & NOT_NULL_FLAG)) {
    regfield->set_notnull();
    regfield->store((longlong)1, true);        // ★ ENUM NOT NULL 无 DEFAULT → 第 1 个元素
  } else regfield->reset();

  default_value.assign(reinterpret_cast<char *>(buf + 1), field.pack_length());  // ★ 二进制镜像
  /* BIT 有余位 → 追加成 default_value 的最后一个字节（与读取侧分支 2 严格对称） */
  if (field.sql_type == MYSQL_TYPE_BIT && !field.treat_bit_as_char &&
      (field.max_display_width_in_codepoints() & 7)) {
    uchar bits = get_rec_bits(bitfield->bit_ptr, bitfield->bit_ofs, bitfield->bit_len);
    default_value.append(reinterpret_cast<char *>(&bits), 1);
  }
```

★ **对称性总结**：`default_value()` 的布局是 **`pack_length()` 字节主数据 +（BIT 有余位时）1 字节余位**。写入侧 `get_rec_bits()` 抽余位、读取侧 `set_rec_bits()` 放回去；写入侧 memcpy 出、读取侧 memcpy 进。这就是为什么 `fill_column_from_dd()` 能如此"粗暴"地直接 `memcpy`——**镜像是同一套 Field 存储格式产出的，无需反序列化**。

**② 表达式类元数据只存文本，且要做归一化**：

```cpp
    if (field.gcol_info) {
      /*
        It is important to normalize the expression's text into the DD, to
        make it independent from sql_mode. For example, 'a||b' means 'a OR b'
        or 'CONCAT(a,b)', depending on if PIPES_AS_CONCAT is on. Using
        Item::print(), we get self-sufficient text containing 'OR' or
        'CONCAT'. If sql_mode later changes, it will not affect the column.
      */
      field.gcol_info->print_expr(thd, &gc_expr);
      col_obj->set_generation_expression(...);
      convert_and_print(&gc_expr, &gc_expr_for_IS, system_charset_info);
      col_obj->set_generation_expression_utf8(...);       // 给 I_S 的 UTF8 版
    }
```

#### 2.7.9 小结（六条设计要点）

1. **默认值存的是"内存镜像"而非"值"**——`default_value()` 是 `pack_length()` 字节的 Field 内部格式快照，读取时 memcpy 即可、零解析；代价是与 server 的 Field 存储格式强耦合。
2. **表达式类元数据只存文本**——Item 树是 per-`TABLE` 的，不在 `TABLE_SHARE` 上共享。
3. **一等 vs options 的取舍**——DD 只把"查询/约束上必须用、类型固定"的做成固定列；零散的、类型相关的、历史兼容的塞进 `options()`。`is_array()` 甚至用一等接口包装了 options。
4. **`auto_flags` 是"有损压缩 + 启发式还原"**——靠 17 字符前缀比较区分 `CURRENT_TIMESTAMP` 与表达式，用 assert 兜底。
5. **布局有两套独立实现**（`find_record_length()` 算长度、`fill_columns_from_dd()` 摆位），靠三个 assert 保证一致；虚拟列必须两遍扫描。
6. **生命周期靠 mem_root 切换**——`open_table_def()` 把 `thd->mem_root` 临时切到 `share->mem_root`，使 Field 与 `TABLE_SHARE` 同生共死。

### 3. DD cache

### 3.0 `Dictionary_client` 与 `Auto_releaser`

每个 `THD` 持有一个 `Dictionary_client`（`dictionary_client.h:150`），它是**所有 DD 访问的唯一入口**——上层代码从不去碰 `Shared_dictionary_cache`。

```cpp
class Dictionary_client {
  std::vector<Entity_object *> m_uncached_objects;  // 249  acquire_uncached() 造的，归 releaser 所有
  Object_registry m_registry_committed;             // 254  与 DD 表一致的已提交对象
  Object_registry m_registry_uncommitted;           // 255  已改未写回的 clone
  Object_registry m_registry_dropped;               // 256  已删除未提交
  THD *m_thd;                                       // 257  cache miss 时要它去读 DD 表
  Auto_releaser  m_default_releaser;                // 258  兜底 releaser
  Auto_releaser *m_current_releaser;                // 259  ★ 当前 releaser，可被替换
};
```

#### ★ `Auto_releaser`：RAII + 栈语义

`Auto_releaser` 是嵌套类：**构造时获取（把自己装成 `m_current_releaser`、`m_prev` 链住旧的），析构时释放（release 所有登记对象、把旧的装回去）**——`m_prev` 链使嵌套的 releaser 构成一个栈，作用域结束自动弹栈。RAII 三要素与"违反后果"见 3.2 节。

```
Auto_releaser r2(client);   // m_current = r2, r2.m_prev = r1
  Auto_releaser r3(client); // m_current = r3, r3.m_prev = r2
~r3  → 释放 r3 登记的对象，m_current 恢复 r2
~r2  → 释放 r2 登记的对象，m_current 恢复 r1
```

★ **关键设计（源码注释 `dictionary_client.h:166-176` 原文）**：

> Objects retrieved from the **shared dictionary cache** are added to the current auto releaser.
> Objects retrieved from the client's **local object register** are **not** added to the auto releaser.

即：**只有从共享缓存取的对象才需要登记（要归还引用计数 / 释放 cache element），从本地 registry 取的是 client 自己持有的，不用还**。这正是 `Auto_releaser` 存在的理由——它管的是"借来的东西"，不是"自己的东西"。

配套能力：

| 成员 | 作用 |
|---|---|
| `auto_release(element)` | 登记一个共享缓存对象；`assert(m_prev != nullptr)` 抓"没用非默认 releaser"的错用 |
| `transfer_release(object)` | 把对象**交给上一层 releaser**（延长生命周期到外层作用域） |
| `remove(element)` | 从链上某处摘下，返回找到它的那个 releaser——★ **改名后 key 变了，需要先摘下、改 key、再插回** |
| `m_uncached_objects` | `acquire_uncached()` 创建的对象由 releaser **拥有**（不是引用计数），析构时 `delete` |

`acquire_uncached()` 的用途：拿到一份**脱离缓存的私有副本**（不被别人看到、也不影响缓存），典型场景是把对象序列化进 binlog 或做 DDL 期间的临时构造。

### 3.1 三层 registry（`dictionary_client.h:248-259`）

```cpp
Object_registry m_registry_committed;    // 与 DD 表一致的已提交对象
Object_registry m_registry_uncommitted;  // 已改未写回的 clone（acquire_for_modification 产生）
Object_registry m_registry_dropped;      // 已删除但未提交
```

`acquire()` 查找顺序（`dictionary_client.cc:886-968`）：**uncommitted → committed → Shared_dictionary_cache → 读 DD 表**。

★ 这本质是**元数据层的 MVCC 简化版**：`m_registry_uncommitted` / `m_registry_dropped` 是每个 client 的**私有写集（private write set）**——client 读自己先命中未提交视图，读别人的只能走 committed → shared（已提交视图）。没有版本链、没有快照号，隔离级别固定为"读已提交"，但**读己写**的语义与 MVCC 同构。

### 3.2 `Auto_releaser` 就是 RAII：构造获取、析构释放

**★ `Auto_releaser` 的栈链**：每个 releaser 记住 `m_prev`，构造时压栈、析构时弹栈，并**强制 LIFO**：

```cpp
// sql/dd/cache/dictionary_client.h:178-246
  class Auto_releaser {
    friend class Dictionary_client;
   private:
    Dictionary_client *m_client;
    Object_registry m_release_registry;
    Auto_releaser *m_prev;                 /* ★ 栈链指针 */

    template <typename T>
    void auto_release(Cache_element<T> *element) {
      // Catch situations where we do not use a non-default releaser.
      assert(m_prev != nullptr);
      m_release_registry.put(element);
    }
    template <typename T>
    void transfer_release(const T *object);
    template <typename T>
    Auto_releaser *remove(Cache_element<T> *element);
    Auto_releaser();
   public:
    explicit Auto_releaser(Dictionary_client *client);
    ~Auto_releaser();
  };
```

```cpp
// sql/dd/impl/cache/dictionary_client.cc:767-776 —— 压栈
Dictionary_client::Auto_releaser::Auto_releaser(Dictionary_client *client)
    : m_client(client), m_prev(client->m_current_releaser) {
  assert(m_client->m_current_releaser != &m_client->m_default_releaser ||
         m_client->m_registry_uncommitted.size_all() == 0 ||
         m_client->m_thd->m_transactional_ddl.inited());
  m_client->m_current_releaser = this;        /* ★ 自己成为新栈顶 */
}
```

```cpp
// sql/dd/impl/cache/dictionary_client.cc:780-832 —— 弹栈 + 强制 LIFO
Dictionary_client::Auto_releaser::~Auto_releaser() {
  assert(m_client->m_current_releaser == this);   /* ★ 强制 LIFO 销毁 */

  m_client->release<Abstract_table>(&m_release_registry);
  m_client->release<Schema>(&m_release_registry);
  ... // 10 个类型逐个 release
  m_client->release<Resource_group>(&m_release_registry);

  m_client->m_current_releaser = m_prev;          /* ★ 弹栈 */

  if (m_client->m_current_releaser == &m_client->m_default_releaser) {
    /* 只剩默认 releaser 时，清理未提交/已删除对象，并 delete 掉
       acquire_uncached() / acquire_for_modification() 造出来的副本 */
    if (!m_client->m_thd->m_transactional_ddl.inited())
      m_client->m_registry_uncommitted.erase_all();
    m_client->m_registry_dropped.erase_all();
    delete_container_pointers(m_client->m_uncached_objects);
  }
}
```

**"所有权上移"**——内层 releaser 想把对象的释放责任交给外层时：

```cpp
template <typename T>
void Dictionary_client::Auto_releaser::transfer_release(const T *object) {
  Cache_element<T> *element = nullptr;
  m_release_registry.get(object, &element);
  assert(element);
  m_release_registry.remove(element);
  m_prev->auto_release(element);          /* ★ 上移一层 */
}
```

```cpp
template <typename T>
Dictionary_client::Auto_releaser *Dictionary_client::Auto_releaser::remove(
    Cache_element<T> *element) {
  // Scan the auto releaser linked list and remove the element.
  for (Auto_releaser *releaser = this; releaser != nullptr;
       releaser = releaser->m_prev) {     /* ★ 沿 m_prev 链向下扫描 */
    Cache_element<T> *e = nullptr;
    releaser->m_release_registry.get(element->object(), &e);
    if (e == element) {
      releaser->m_release_registry.remove(element);
      return releaser;
    }
  }
  assert(false);
  return nullptr;
}
```

> ★ 典型用法在 `acquire()` 里：局部 releaser 压栈取对象，`dynamic_cast` 成功后若对象要返回给调用者，就 `releaser.transfer_release()` 把释放责任**上移**给外层的 releaser。这样"取了但类型不匹配"的情况也能自动释放。

#### ★★ 缓存的真实三层结构（含一处重要纠错）

> ⚠️ **`Object_cache` 这个类不存在**（全树 grep `Object_cache` 仅 3 处命中，都是 PFS 变量名 `key_object_cache_mutex`）。真名是 **`Object_registry`**（`sql/dd/cache/object_registry.h:62`）。

三层是：

```
① Shared_dictionary_cache（进程级单例）
     10 个 Shared_multi_map<T>，每类型一把 mutex + free list + element pool
     cache miss → Storage_adapter::get() 直读 mysql.* 表
                        ↓
② Shared_multi_map<T>（按类型分区，处理并发与淘汰）
                        ↓
③ Dictionary_client（THD 级，3 个 Object_registry）
     m_registry_committed / m_registry_uncommitted / m_registry_dropped
```

```cpp
// sql/dd/cache/dictionary_client.h:254-259
  Object_registry m_registry_committed;    // Registry of committed objects.
  Object_registry m_registry_uncommitted;  // Registry of uncommitted objects.
  Object_registry m_registry_dropped;      // Registry of dropped objects.
  THD *m_thd;
  Auto_releaser m_default_releaser;        // 栈底
  Auto_releaser *m_current_releaser;       // 栈顶
```

**可见性分层的官方说明**（类注释原文）：

```cpp
  The client also has a second registry of objects with uncommitted changes.
  These are objects acquired by acquire_for_modification() or registered
  with register_uncommitted_object(). These objects are only present in
  the local registry and not in the shared cache. Once registered, the
  objects can also be retrieved with normal acquire(). This means that
  a given client has a view which includes uncommitted changes made
  using the same client, while other clients do not see these changes.
```

**`Object_registry` 内部是 10 个按类型分区的 map（按需分配）**，不是"四个 map"：

```cpp
  std::unique_ptr<Local_multi_map<Abstract_table>> m_abstract_table_map;
  std::unique_ptr<Local_multi_map<Charset>> m_charset_map;
  std::unique_ptr<Local_multi_map<Collation>> m_collation_map;
  std::unique_ptr<Local_multi_map<Column_statistics>> m_column_statistics_map;
  std::unique_ptr<Local_multi_map<Event>> m_event_map;
  std::unique_ptr<Local_multi_map<Resource_group>> m_resource_group_map;
  std::unique_ptr<Local_multi_map<Routine>> m_routine_map;
  std::unique_ptr<Local_multi_map<Schema>> m_schema_map;
  std::unique_ptr<Local_multi_map<Spatial_reference_system>>
      m_spatial_reference_system_map;
  std::unique_ptr<Local_multi_map<Tablespace>> m_tablespace_map;
```

> ⚠️ "四个 map"其实在更下层——`Multi_map_base<T>` 里：`m_rev_map` / `m_id_map` / `m_name_map` / `m_aux_map`（`sql/dd/cache/multi_map_base.h:66-74`），对应对象的四种查找键。

#### ★ `acquire` vs `acquire_for_modification`

**核心差异只有两行**：`acquire_for_modification` 会 **clone 一份可改副本**并交给 `m_uncached_objects` 托管：

```cpp
// sql/dd/impl/cache/dictionary_client.cc:1166-1170
    const T *casted = dynamic_cast<const T *>(cached_object);
    if (!casted)
      *object = nullptr;
    else {
      *object = casted->clone();        /* ★ 关键差异 1：克隆 */
      auto_delete<T>(*object);          /* ★ 关键差异 2：交给 m_uncached_objects 托管 */
    }
```

而 `acquire` 返回的是 `const T*`（共享缓存拥有的对象），只在"确实从 shared cache 取到"时才登记 auto release：

```cpp
    m_registry_committed.put(element);
    m_current_releaser->auto_release(element);
```

> ⚠️ `acquire_for_modification` 里**没有**"必须已注册未提交对象"的断言（常被误传）。语义接近的断言分别在 `store()`（`assert(!element)`，对象不得已在 committed registry）、`update()`（同）、`register_uncommitted_object()`（`assert(m_current_releaser != &m_default_releaser)`）。

`update()` 的注释点出了两种进入 uncommitted registry 的途径：

```cpp
    // If new_object->id() is present in the uncommitted registry, then
    // that object is the previously stored object for this id, since the
    // only way to enter the uncommitted registry is through store() or
    // update().
```

提交/回滚就是把 uncommitted registry 清空（提交时先让 shared cache 里的旧副本失效）：

```cpp
      // Invalidate the entry in the shared cache (if present).
      invalidate(uncommitted_object);
      ...
  m_registry_uncommitted.erase<typename T::Cache_partition>();
  m_registry_dropped.erase<typename T::Cache_partition>();
```

`Auto_releaser` 是教科书式的 **RAII**（Resource Acquisition Is Initialization）：

| RAII 三要素 | 这里的对应 |
|---|---|
| **获取** | 构造时：把自己装成 `m_current_releaser`，用 `m_prev` 链住旧的 |
| **释放** | 析构时：按类型逐个 `release()` 登记对象，把 `m_prev` 装回 `m_current_releaser` |
| **异常安全** | 任何 `return` / 异常展开都必然触发析构，**不存在"忘了归还"的路径** |

`m_prev` 链使多个 releaser 嵌套成**栈**——debug 构建还会断言 **LIFO 序**（内层必须先于外层析构，即"压栈/弹栈"语义，3.0 节那张链图就是它的可视化）。

**违反 RAII（该用而没用）的后果**：

1. 共享缓存对象不归还 → `Cache_element::usage()` 不归零 → 元素进不了 free list → **容量无法回收**，后续频繁 cache miss / 内存膨胀；
2. `acquire_uncached()` / `acquire_for_modification()` 的堆对象**泄漏**——它们归 releaser **所有**（owned，不是引用计数），只有析构才 `delete`；
3. debug 断言兜底：`assert(m_prev != nullptr)`（`dictionary_client.h:195`）抓"没用作用域 releaser 却从共享缓存取对象"。

### 3.3 `Shared_dictionary_cache`

按对象类型划分的多个 `Shared_multi_map`（`shared_dictionary_cache.h:81-90`）。容量在 `Shared_dictionary_cache::init()`（`shared_dictionary_cache.cc:45-62`）里逐一设定——**有些是硬编码常量，有些直接绑定 server 参数**：

```cpp
void Shared_dictionary_cache::init() {
  m_map<Collation>()->set_capacity(collation_capacity);                 // 256
  m_map<Charset>()->set_capacity(charset_capacity);                     // 64

  // Set capacity to have room for all connections to leave an element
  // unused in the cache to avoid frequent cache misses while e.g.
  // opening a table.
  m_map<Abstract_table>()->set_capacity(max_connections);                // ★
  m_map<Event>()->set_capacity(event_capacity);                          // 256
  m_map<Routine>()->set_capacity(stored_program_def_size);
  m_map<Schema>()->set_capacity(schema_def_size);
  m_map<Column_statistics>()->set_capacity(column_statistics_capacity);  // 32
  m_map<Spatial_reference_system>()->set_capacity(
      spatial_reference_system_capacity);                                // 256
  m_map<Tablespace>()->set_capacity(tablespace_def_size);
  m_map<Resource_group>()->set_capacity(resource_group_capacity);        // 32
}
```

| 对象类型 | 容量来源 | 典型值 |
|---|---|---|
| `Abstract_table`（表+视图） | **`max_connections`** | 151 / 上千 |
| `Routine` | `stored_program_def_size` | 256 |
| `Schema` | `schema_def_size` | 256 |
| `Tablespace` | `tablespace_def_size` | 256 |
| `Collation` / `Charset` | 硬编码常量 | 256 / 64 |
| `Event` / `Spatial_reference_system` | 硬编码常量 | 256 / 256 |
| `Column_statistics` / `Resource_group` | 硬编码常量 | 32 / 32 |

★ **`Abstract_table` 的容量 = `max_connections`** 这条最值得注意。源码注释说明了理由：*让每个连接都能在缓存里留下一个未被引用的元素，避免开表时频繁 cache miss*。也就是说它的容量不是按"表数量"定的，而是按**并发度**定的——与 `table_open_cache` 按 thread id 分片、InnoDB Fil 层按 space_id 分 68 片是**同一个思路在不同层的体现**：**容量/分片跟着并发度走，而不是跟着数据量走**。

另：`Collation` 256 / `Charset` 64 这类硬编码值够用，是因为这些对象**种类极少且几乎不变**（一个实例只有几十种字符集）。

**★ 多 key 索引**：一个 `Cache_element` 被**多种 key 同时索引**——`Shared_multi_map` 按 key 类型（`Primary_id_key` / `Item_name_key` / `Global_name_key` / `Se_private_id_key`，见理论基础三）各维护一个哈希表，**所有 key 都指向同一个元素**。查找时按 key 类型选哈希表，O(1)；元素被移除时**所有 key 一起摘除**（这就是 `Auto_releaser::remove()` 那处"改名要先摘下改 key 再插回"的约束来源——多 key 下改一个 key 必须同步所有索引，否则各哈希表指向不一致）。

有 LRU，但**只作用于 free list**（`shared_multi_map.cc:114-126` `rectify_free_list()`：`while (map_capacity_exceeded() && m_free_list.length() > 0) { e = m_free_list.get_lru(); remove(e, lock); }`），不是经典的按访问时间全局 LRU。

### 3.4 `acquire` vs `acquire_for_modification`

| | `acquire()` | `acquire_for_modification()` |
|---|---|---|
| 返回 | `const T*`，**shared cache 拥有，不得释放** | `T*`，**clone 出的可改副本，client 拥有** |
| 核心 | 查找 + 登记 releaser | `acquire()` 后 `casted->clone()` + `auto_delete` |
| 用途 | 读 | 改（DDL 时先改这个 clone，提交时写回） |

★ `acquire_for_modification()` 就是 **copy-on-write（COW）**：修改前先 `clone()` 出一份私有副本，改动只落在副本上，提交时（`commit_modified_objects()`）才合并回共享视图。**其他 client 持有的共享对象在 DDL 提交前完全不受影响**——代价是 DDL 路径多一次深拷贝。

两者都要求持有对应 MDL（debug 断言 `MDL_checker::is_read_locked`）。

### 3.5 ★ `Open_dictionary_tables_ctx`（otx）：一次 restore 的"表集合 + 共享视图"

所有 `restore_*` / `store_*` 都带一个 `otx` 参数，它是**一次字典访问的事务上下文**（`transaction_impl.h:76`）：

```cpp
class Open_dictionary_tables_ctx {
 public:
  Open_dictionary_tables_ctx(THD *thd, thr_lock_type lock_type)   // TL_READ / TL_WRITE
      : m_thd(thd), m_lock_type(lock_type), m_ignore_global_read_lock(false) {}

  // 获取对应的 dd::tables（本仓库注释：比如 T=Index, 则 DD_table 为
  // dd::Index::DD_table, 即 dd::tables::Indexes, 即 mysql.indexes）
  template <typename T>
  Raw_table *get_table() const {
    return get_table(T::DD_table::instance().name());
  }

  template <typename X>
  void register_tables() { X::Impl::register_tables(this); }   // 静态注册

  bool open_tables();    // 把注册的表一次性打开
  ...
};
```

★★ **`get_table<T>()` 就是理论基础里那张 `DD_table` typedef 表的直接消费者**——`T::DD_table::instance().name()` 从类型直接推导出表名（`Index → mysql.indexes`），编译期完成"对象 → 物理表"的解析。

完整使用协议（三步）：

```cpp
Open_dictionary_tables_ctx otx(thd, TL_READ);      // ① 构造：THD + 锁类型
otx.register_tables<Table>();                       // ② 递归注册：Table_impl::register_tables
                                                    //    会把 columns/indexes/foreign_keys/... 
                                                    //    全部 add_table 进 m_tables 清单
if (otx.open_tables()) ...                          // ③ 一次性打开（共享同一 attachable tx）

table->restore_attributes(r);                       // ④ 之后各函数里 get_table<T>()
table->restore_children(&otx);                      //    取已打开的 handler，不再开表
```

★ 三个设计意义：

1. **一次 restore 读十几张 DD 表，但只开一次**——`restore_children` 递归时所有 handler 已在 `open_tables()` 就绪；
2. **共享一个只读视图**：所有表在同一 attachable transaction 里打开，restore 期间 DD 表内容一致（不会被并发 DDL 撕裂）；
3. **按需注册 = 精确表清单**：`Table_impl::register_tables` 注册的正是"它的描述树涉及的全部表"——这就是第（四）节说的"每棵描述树要读哪些 DD 表提前设置好"的实现。

这解释了 `restore_items` 里 `assert(table)` 的那句注释：*table was not registered for that transaction. Use register_tables()*——**忘了注册的表在 restore 时 handler 不存在，直接 assert 暴露**。

---

### 4. SDI（Serialized Dictionary Information）

> ★ **SDI 已独立成篇**：[`../../feat/sdi.md`](../../feat/sdi.md)（横跨 server ↔ InnoDB 两层）。它同时需要 server 侧（`sdi.cc` 的序列化格式）与 InnoDB 侧（`dict0sdi.cc` 的 B-tree 存储、根页号）才能讲清，按"跨层特性"归位 `feat/`（与 `fts.md` 同性质）。本篇只在"SDI 与 DD 是什么关系"这层保留定位，机制细节一律见独立篇。

**一句话**：SDI 是 DD 对象序列化后的**紧凑 JSON 副本**，写进表空间内的 SDI B-tree（或退化为独立 `.sdi` 文件），供 `IMPORT TABLESPACE` 与 `ibd2sdi` 使用。

| 维度 | 要点 |
|---|---|
| 格式 | 紧凑 JSON，顶层 5 键（`mysqld_version_id` / `dd_version` / `sdi_version` / `dd_object_type` / `dd_object`）；枚举写成数值，SDI 不自解释 |
| 存储 | 伪装成 InnoDB 内存表 `SDI_<space_id>`（5 列、主键 `(type,id)`）⇒ 复用 MVCC / 回滚 / zlib 压缩 / LOB 溢出 |
| 载体 | 能写表空间的引擎写表空间；不能的写 `.sdi` 文件（**按 `hton->sdi_set` 能力分派**，与 `innodb_read_only` 无关） |
| 版本 | 三层：`mysqld_version_id`（`<=` 宽松）/ `dd_version`（全等）/ `sdi_version`（全等）⇒ **跨 minor 版本导入基本不可行** |
| ⚠️ 读路径 | **`dict_sdi_get` / `dict_sdi_drop` 被 `#if 0` 关掉**——服务器目前**只写不读**；真实读场景只有 `IMPORT TABLESPACE ... FROM .sdi` 与 `ibd2sdi` |

#### 与本篇的关系（DD 视角）

- SDI **不是独立数据源**，是 DD 对象的**冗余副本**
- 一致性由"**DD 写 + SDI 写在同一事务**"保证：先写 DD 表、后写 SDI，失败则整个 DDL 事务回滚（见 6.）
- **不写对象 id**，只写 name ⇒ IMPORT 必然重新分配 id
- 自举期 SDI 索引的创建在 `srv_create_sdi_indexes()`（必须晚于 `dict_create()`），见 [`../lifecycle/01_initialize.md`](../lifecycle/01_initialize.md)
- 5.7 升级时**禁写 SDI**、升级后批量补写（见 7.）

完整机制——序列化实现与两个 context、引用表示、版本检查、B-tree 存储、读写路径、两个真实 SDI bug——见 [`../../feat/sdi.md`](../../feat/sdi.md)。

### 5. 启动自举（鸡生蛋问题）

**问题**：DD 表本身是 InnoDB 表——要先能读 `mysql.tables` 才能知道表定义，但要知道表定义才能打开它。

#### ★ 设计溯源：WL#6394「Bootstrap code for new DD」

这份自举实现的官方设计文档是 [WL#6394](https://dev.mysql.com/worklog/task/?id=6394)（Affects: Server-8.0，Complete）。它给出了几个代码里看不出动机的问题的**官方答案**：

**① 官方的表分类是六类**（WL §3.1，比 `System_tables::Types` 的枚举更细）：

| WL 分类 | 定义 | 例子 |
|---|---|---|
| `INERT_TABLE` | 初始化必需、**定义不可变** | version 表（即 `dd_properties`） |
| `CORE_TABLE` | 读/存"**无外键约束**的表定义"所必需 | schemata / tables / columns / column_type_elements / indexes / index_column_usage / tablespaces / tablespace_files |
| `TABLE_DEF_TABLE` | 读/存表定义所需 | foreign_keys / table_partitions / index_partitions / view_table_usage … |
| `DDSE_TABLE` | **DDSE 需要、由 server 代为创建**的表 | `innodb_table_stats` / `innodb_index_stats` / `innodb_table_metadata` |
| `DICTIONARY_TABLE` | 存数据库对象元数据 | character_sets / collations / catalogs |
| `SYSTEM_TABLE` | 存系统数据（help、时区等） | — |

配套的关键定义：**Actual DD version**（磁盘上实际存在的，存在 version 表里）vs **Target DD version**（源码里写死的）——**两者相等就意味着"不需要升级"**；`Hardcoded DD table definition` → `Synthesized DDL statement` → `Synthesized TABLE_SHARE` 是"从 C++ 结构生成 SQL 再生成内存对象"这条链的标准命名。

**② ★ 建 core 表时三样东西必须稳定**（WL §3.1.5）：固定顺序 + 硬编码 `se_private_id`，使得每张 DD 表的**每个索引 root 页号**、**`PAGE_INDEX_ID`**、`space_id` 都恒定——因为重启时要靠硬编码定义直接打开已存在的表，页号一变就找不着。

**③ ★ 外键必须单独一步加**（§3.1.9）："due to circular foreign key relationships"——core 表之间有**循环外键依赖**，只能先建表后 `ALTER TABLE ... ADD FOREIGN KEY`。

**④ ★ 重启时为什么也要执行 CREATE TABLE**（§3.2.4）：为的是**生成 `dd::Table` 结构**——DDL 执行路径被改造成"表已存在就不调 `handler::create()`"，元数据只**临时放进 shared cache**（volatile，仅够开表用）。这与 §3.2.5 的 `get_se_private_data(table, version)`（填 root 页号）配合，构成"重启路径的硬编码开表"。

**⑤ 版本校验的四个分支**（§3.2.6）：

| 读到的值 | 结果 |
|---|---|
| 无值 | 错误 |
| `> target` | 错误——★ **不支持降级**（downgrade not supported） |
| `< target` | 错误，除非是受支持的升级源版本 |
| `== target` | OK，正常启动 |

**⑥ ★ 升级路径的官方规定**（§3.2.11），与第 7 章 `upgrade_tables()` 逐条对应：

- 先做一次 **slow shutdown 但保持 redo 打开**——让 InnoDB 可以放弃对旧版内部数据结构的支持，并重新分配 `dd.tables.se_private_id`；
- 执行 inplace `ALTER` / `CREATE` / `DROP` 改表定义，**原文：*"Preferably use only one transaction if we are able to relax auto commit"*** ← 这就是 `upgrade_tables()` 里"建表阶段 auto commit 串行 + Atomic Switch 三步一事务"的设计由来；
- 保证缓存一致（dd cache vs TDC），**重新生成所有表的 SDI**；
- 最后写入新的 DD version。

**⑦ ★ 升级崩溃的处理**（§4）：以 InnoDB 为 DDSE 时升级会重建 DD tablespace，中途崩溃的处置是——**删掉那个重建中的 tablespace，然后重启升级**（残留可由 DDSE 自动删除）。官方还建议"先把表迁进新 tablespace，再改表定义"。

**⑧ 为什么 DD 自举必须夹在"InnoDB 之后、动态插件之前"**（§1.1.4）：DD 存在 InnoDB 里 ⇒ InnoDB 必须先起来；而 `mysql.plugins` 表必须在加载**动态插件**前就位 ⇒ 插件必须后加载。为此 `plugin_init()` 被拆成四个函数（两个对外可见），让 `init_server_components()` 能按"内置插件（含 InnoDB）→ 初始化 DD → 动态插件"的顺序编排。

**⑨ 升级的前提**（§2 Preconditions）：不得有待处理的 XA PREPARE 事务、undo log 必须为空（即要求用 `innodb_fast_shutdown=0` 做过慢关闭）。

**破解靠三件事**：

### ① DD 表定义是 C++ 硬编码的

`create_tables()`（`bootstrapper.cc:1394-1417`）遍历**注册表 `System_tables`**（启动时把全部 `dd::tables::*` 类注册进去），对每张表走 `create_target_table`：

```cpp
bool create_target_table(THD *thd, const Object_table *object_table) {
  if (object_table->is_abandoned()) return false;   // ★ 已弃用的表跳过（为升级保留的定义）

  const Object_table_definition *target_table_def =
      object_table->target_table_definition();
  String_type target_ddl_statement = target_table_def->get_ddl();   // 拼 DDL

  return dd::execute_query(thd, target_ddl_statement);   // 内部执行
}
```

★ **`is_abandoned()` 就是第（六）节说的"为升级保留废表定义"的落点**：废弃表仍在注册表里，但初始化时跳过建表。

`dd::execute_query` → `Ed_connection::execute_direct`（内部直接执行 SQL 的通道，不经过正常客户端协议）→ 解析 → 建表。字段来自 `Object_table_definition_impl` 构造时 `add_field()` 编译期写死。

**顺序的讲究**（`initialize_dictionary` 主流程）：

```cpp
create_dd_schema(thd)                // ① CREATE SCHEMA mysql + USE mysql
  || initialize_dd_properties(thd)   // ② 先建 dd_properties 表
  || create_tables(thd, nullptr)     // ③ 再建其余 30 张
```

★ **`dd_properties` 为什么第一个建**：它的行里存着 **DD 版本号**（`DD_VERSION`）——`initialize_dd_properties` 建完表后立即读版本、`set_actual_dd_version()`。之后 `create_tables` 里所有判断（如 `is_dd_upgrade_from_before(DD_VERSION_80016)`）都依赖这个版本。

`initialize_dictionary` 的完整串（`bootstrapper.cc:940` 起，重启路径）：

```cpp
create_dd_schema || initialize_dd_properties || create_tables ||
sync_meta_data ||                       // core registry 内存态 → 真表
DDSE_dict_recover(RESTART_SERVER) ||    // SE 侧恢复（innodb_dict_recover）
upgrade_checks || upgrade_tables ||     // 版本升级
repopulate_charsets_and_collations || verify_contents || update_versions
```

### ①.5 ★ 5.7 升级的状态跟踪（`Upgrade_status`）

从 5.7 升级时，初始化主流程在 `create_tables` 后**立即把阶段写进文件**（`upgrade_57::Upgrade_status`）：

```cpp
if (is_dd_upgrade_57) {
  upgrade_57::Upgrade_status().update(
      upgrade_57::Upgrade_status::enum_stage::DICT_TABLES_CREATED);
}
```

★ 这个"阶段文件"的意义（源码 DBUG 注释明说）：**升级中途崩溃后，下次重启能识别出"上次升到哪了"，把已做的改动全部回滚，让数据目录还能被 5.7 复用**。即升级过程是"先记录阶段 → 再推进"，可重入、可回退。

### ② InnoDB 先起来，mysql.ibd 由硬编码代码创建

`dd::init()`（`dictionary_impl.cc:104-176`）之前 mysqld 已经 `plugin_register_builtin_and_init_core_se()` 加载了 InnoDB。

`DDSE_dict_init()` → `innobase_ddse_dict_init` → `innobase_init_files()`（`ha_innodb.cc:5549`）：

```cpp
ret = create ? dd_create_hardcoded(dict_sys_t::s_dict_space_id, s_dd_space_file_name)
             : dd_open_hardcoded(...);
```

`dd_create_hardcoded()`（`ha_innodb.cc:5397`）：`fil_ibd_create(0xFFFFFFFE, "mysql", "mysql.ibd", ...)` → `fsp_header_init()` → `btr_sdi_create_index()`（**mysql.ibd 自带 SDI 索引**）。

### ③ 自举阶段用"core registry"（内存假存储）绕过 DD 表

这是 **Strategy 模式**：`Storage_adapter` 是持久化抽象，真实 DD 表与 core registry（内存假存储）是它的两个策略。`store()` 按当前 stage 选择策略（`storage_adapter.cc:314-320`）：

```cpp
stage < Stage::CREATED_TABLES → core_store()      // 策略 A：存内存
否则                           → 写真实 DD 表       // 策略 B：存 B+ 树
```

DD 表建好后 `sync_meta_data()` 把内存态灌进真表——**策略切换对上层透明**，上层只调 `store()`。

阶段枚举 `bootstrap_ctx.h:42-54`：`NOT_STARTED → STARTED → CREATED_TABLESPACES → FETCHED_PROPERTIES → CREATED_TABLES → SYNCED → UPGRADED_TABLES → POPULATED → STORED_DD_META_DATA → VERSION_UPDATED → FINISHED`。

---

### 6. 原子 DDL 中 DD 的地位

★ **先纠正一个常见误解**：`trx->is_dd_trx` **不是**原子 DDL 的实现机制。它是 **DEBUG-only** 标记（`trx0trx.h:1123`，`#ifdef UNIV_DEBUG`），且语义是"对 DD 表做非锁定只读的 attachable 事务"（`thd_tx_is_dd_trx()` → `thd->is_attachable_ro_transaction_active()`）。

**真正的机制**：

```cpp
// sql/transaction.cc:233-300 trans_commit()
res = ha_commit_trans(thd, true, ignore_global_read_lock);      // SE 层 2PC prepare+commit
if (!thd->is_attachable_rw_transaction_active()) {
  if (res) thd->dd_client()->rollback_modified_objects();       // 失败 → 回滚 DD cache
  else     thd->dd_client()->commit_modified_objects();          // 成功 → DD cache 同步到 DD 表状态
}
thd->m_transactional_ddl.post_ddl();
```

三层协作：

| 层 | 机制 |
|---|---|
| **DD cache ↔ DD 表** | `commit_modified_objects()` / `rollback_modified_objects()`，保证 cache、DD 表、SE 状态三者同步（注释明说：SE 提交失败时必须回滚 DD 对象） |
| **引擎物理操作** | InnoDB 的 DDL log（`log_ddl` + `mysql.innodb_ddl_log`）兜底，崩溃后 replay/清理 |
| **binlog** | 语句级写 binlog，靠 `ha_commit_trans()` 的 2PC（binlog 与 InnoDB XA prepare/commit 排序） |

另外 `Transactional_ddl_context`（`sql_class.h:4652`）用于支持"DDL 纳入用户事务"（8.0 起部分 DDL 可回滚/原子），此时跳过 `trans_commit_implicit`（`sql_table.cc:10336`）。

**哪些 DDL 真正"可回滚"**（`Transactional_ddl_context` 的适用边界）：

| DDL | 是否进用户事务 | 说明 |
|---|---|---|
| `CREATE TABLE` / `ALTER` / `DROP` 的**元数据阶段** | 是 | DD 表写入随事务回滚（8.0 原子 DDL） |
| 引擎的**物理阶段**（建 .ibd、重建表） | 否 | 靠 InnoDB DDL log 兜底（见 [`innodb_dict.md`](innodb_dict.md)），提交后 replay |
| `CREATE/DROP DATABASE` | 是 | 纯元数据，全可回滚 |
| 隐式提交类（如 `ALTER ... PARTITION` 某些路径） | 否 | 语义上不允许进事务 |

★ 判断依据：**"能写进 DD 表 + 受同一事务保护的"才可回滚；动了物理文件的靠 DDL log 事后清理**。所以严格说 8.0 没有"完全可回滚的 DDL"，只有"元数据可回滚 + 物理动作可 replay"的组合——`Transactional_ddl_context` 管前半，DDL log 管后半。

DD 表的读写用 **attachable transaction**（`transaction_impl.h:130-150`）——又是 **RAII**：`Transaction_ro` 构造时 `begin_attachable_ro_transaction()`、析构时 `end_attachable_transaction()`。"attachable"指**借用当前 THD 开一个独立的只读事务**读 DD，对象析构即归还，不污染用户自己的事务状态。

#### ★★ 设计溯源：WL#7743「New data dictionary: changes to DDL-related parts of SE API」

原子 DDL 的官方设计文档是 [WL#7743](https://dev.mysql.com/worklog/task/?id=7743)。它回答的是"**为什么要把 DDL 塞进一个事务**"，以及为此改了哪些 SE API。

**① 三件事必须同时做到**（WL 开篇）：SE 要摆脱自有字典 ⇒ 需要 ①隐藏列/键能写进 DD ②`se_private_*` 能读写 ③DDL 原子/崩溃安全。

**② ★ `se_private_data` 的通道是"给 SE API 全量加 `dd::Table*` 参数"**：

| API | 新增参数 | 语义 |
|---|---|---|
| `handler::open` | `const dd::Table *table_def` | 开表时 SE 能读到 `se_private_*`（★ 优化器自建的临时表没有 `dd::Table`，传 `NULL`） |
| `handler::create` | `dd::Table *table_def`（in/out） | SE 可回填 root 页号、`se_private_id` |
| `handler::truncate` / `rename_table` / `delete_table` | `dd::Table *` | 同类 |
| `prepare_/inplace_/commit_inplace_alter_table` | `const dd::Table *old` + `dd::Table *new` | INPLACE 三个阶段都能改定义 |
| `handlerton::alter_tablespace` | `old_ts_def` / `new_ts_def` | 表空间版 |
| `Partition_handler::truncate_partition_low` / `exchange_partition_low` | `dd::Table *` | 分区操作 |

★ 还有个专门的 `handler::add_extra_columns_and_keys()`——让 SE 在建表时把**隐藏列/隐藏键**（InnoDB 的 `DB_TRX_ID` / `DB_ROW_ID`、隐式主键）加进 `dd::Table`，**SQL 层不做任何校验**（"it is responsibility of SE to ensure that they are valid"）。

★ 一条容易忽略的约束：WL 明确写了——只允许**支持原子 DDL 的引擎**调整 `dd::Table`，**SQL 层对不支持的引擎干脆不把调整过的对象存回 DD**（否则 DD 与 SE 会不同步）。区分标志就是 `HTON_SUPPORTS_ATOMIC_DDL`。

**③ ★★ 原子/crash-safe DDL 的四项前提**（WL 原文）：

1. SQL 层在 DDL 期间**没有中间提交**（本 WL 负责）
2. SE 方法内**没有中间提交**，且 SE 要把自己注册进当前事务（InnoDB 侧由 **WL#7016** 负责）
3. SE 能对 DDL 做 redo/rollback（WL#7016）；本 WL 为此引入 **`handlerton::post_ddl()` 钩子**——提交或回滚后让 SE 做收尾（删文件、等真删除完成）
4. binlog 写入成为 DDL 事务的一部分（本 WL + **WL#9175** 保证 DDL 的 binlog 崩溃恢复）

外加第 5 条：**顺手把一些 DDL 的用户可见行为改得更原子**（如 `DROP TABLES` 失败时尽量不留副作用）。

**④ ★ DROP TABLES 的五类分类与新算法**（WL §B）——这是全篇最体现"兼容性 vs 原子性权衡"的一段：

| 类 | 含义 |
|---|---|
| a) | 不存在的表 |
| b) | 不支持原子 DDL 的 SE 里的基表 |
| c) | 支持原子 DDL 的 SE 里的基表 |
| d) / e) | 非事务 / 事务临时表 |

算法要点：**先处理 b) 再处理 c)**（否则删 b) 失败时会连带误删已经删掉的原子表）；**a) 与 c) 合并处理**（"in order to have single nice DROP TABLES statement for the main InnoDB-only case"）。

★ **为兼容老 server 复制付出了代价**：GTID 模式下为了保证"主从语句不拆分"，不得不**推迟写 binlog**，原文直言 *"Obviously we sacrifice crash-safety to compatibility here"*——即"原子性 vs 复制兼容"这个权衡是**官方明文承认**的。

**⑤ ★ DROP DATABASE 留了一个已知的原子性漏洞**（WL §G 末尾）：第 4.5) 步事务提交之后、第 6) 步删除数据库目录之前崩溃 ⇒ **不原子**。官方说这"需要引入目录删除的 redo log，本 WL 不解决"。即 8.0 的原子 DDL **不是完全原子**——文件目录的删除仍是漏洞窗口。

**⑥ 各类 DDL 的新步骤数**（WL 逐条给出）：CREATE TABLE 10 步、ALTER COPY 14 步、ALTER INPLACE 19 步、TRUNCATE 5 步、RENAME、TABLESPACE 6 步、EXCHANGE PARTITION 6 步。**共同结尾都是"commit or rollback → 调 `post_ddl()`"**。

#### ★★ 事务化的四个实现细节（源码层）

"把元数据放进事务"这句话在源码里落实为四个具体机制，缺一不可：

**① attachable 是引擎的强制能力**，不是可选优化（`transaction_impl.cc:110-121`）：

```cpp
/* Data-dictionary tables must use storage engine supporting attachable transactions. */
for (Table_ref *t = table_list; t; t = t->next_global) {
  assert(t->table->file->ha_table_flags() & HA_ATTACHABLE_TRX_COMPATIBLE);   // ★ 断言
  if (t->table->file->ha_extra(HA_EXTRA_NO_AUTOINC_LOCKING)) return true;   // ★ 关 autoinc 锁
}
```

★ 这条断言直接回答了"**为什么 DD 只能用 InnoDB**"——`HA_ATTACHABLE_TRX_COMPATIBLE` 是引擎必须提供的能力标志（借 THD 开独立只读事务读表）。不支持 attachable 的引擎**根本不能托管 DD 表**。

★ `HA_EXTRA_NO_AUTOINC_LOCKING`：DD 表操作**禁用 autoinc 锁**（AUTOINC 表锁）。源码注释说明理由——原子 DDL 期间它会**显著增加死锁概率**，而"DD 变更不要求复制出完全相同的 ID"，所以没必要。

**② ★★ `DD_kill_immunizer`：DD 操作免疫 kill**（`sql/dd/dd_kill_immunizer.h:46`）

```cpp
class DD_kill_immunizer {
  DD_kill_immunizer(THD *thd) : ... {
    m_saved_kill_immunizer = thd->kill_immunizer;        // ★ 支持嵌套：记住父亲
    m_saved_killed_state = thd->kill_immunizer
                               ? thd->kill_immunizer->m_killed_state
                               : thd->killed.load();
    thd->kill_immunizer = this;
    thd->killed = THD::NOT_KILLED;                        // ★ 期间当作没被 kill
  }
};
```

★ **为什么必须免疫**：DD 操作（尤其 DDL 的元数据阶段）**不能中途被打断**——写到一半的字典会让后续所有表操作读到一个不存在的中间态。`Transaction_ro` 和 `Update_dictionary_tables_ctx` 都持有它（第一、三个成员），即**任何 DD 读写期间 THD 都对 KILL 免疫**，析构时恢复原状态。

★ 支持嵌套这点也很关键：DD 操作会互相调用（读一张表的过程中又要读另一张），内层免疫器保存外层免疫器的状态而非 `thd->killed`，避免内层析构时把"外层仍在免疫中"的事实抹掉。

**③ ★ `Update_dictionary_tables_ctx`：11 项执行环境的保存/恢复**（`transaction_impl.h:157-195`）

DD 的写操作必须在一个**干净、可预测**的执行环境里跑，于是它把 THD 上的一堆状态先存后改、析构时还原：

```cpp
class Update_dictionary_tables_ctx {
  Open_dictionary_tables_ctx otx;
  DD_kill_immunizer m_kill_immunizer;            // 免疫 kill
  Query_tables_list *m_query_tables_list_backup;
  Open_tables_backup m_open_tables_state_backup; // 开表状态
  bool m_saved_binlog_row_based;                 // binlog 格式（强制切 statement）
  ulonglong m_saved_options;
  sql_mode_t m_saved_mode;                       // ★ sql_mode 会被清零
  long long m_latest_auto_incr_id;
  enum_check_fields m_saved_check_for_truncated_fields;
  uint m_saved_in_sub_stmt;
  bool m_saved_time_zone_used;
  ulong m_saved_auto_increment_increment;        // ★ auto_increment_increment 强制设为 1
  Discrete_intervals_list m_auto_inc_intervals_in_cur_stmt_for_binlog_saved;
  Discrete_intervals_list m_auto_inc_intervals_forced_saved;
};
```

★ 最有意思的两项：`sql_mode` 清零（避免严格模式让"迁移旧数据"失败——与升级流程里 `thd->variables.sql_mode = 0` 同理）与 `auto_increment_increment` 强制为 1（DD 表自身的 ID 分配不受用户设置的步长干扰）。

这是**又一个 RAII**：DD 操作"借"THD 执行，但必须把 THD 原样还回去，否则用户会话的状态会被 DD 内部操作污染。

**④ 事务边界：`commit_modified_objects` 挂在 SE 提交之后**（`transaction.cc:292-297`）

DD cache 的提交/回滚**必须在 SE 事务提交之后**：

```cpp
res = ha_commit_trans(thd, true, ignore_global_read_lock);   // ① SE 层 2PC 先走完
if (!thd->is_attachable_rw_transaction_active()) {
  if (res) thd->dd_client()->rollback_modified_objects();    // ② 失败 → 回滚 DD cache
  else     thd->dd_client()->commit_modified_objects();      // ② 成功 → DD cache 同步
}
thd->locked_tables_list.adjust_renamed_tablespace_mdls(&thd->mdl_context);
thd->m_transactional_ddl.post_ddl();                          // ③ DDL 收尾（DDL log 清理等）
```

★ 顺序不能反：**先让引擎落盘（含 DD 表行的写入与 InnoDB DDL log），再更新内存中的 DD cache**。反过来若先提交 cache 而 SE 失败，内存里的字典就会指向不存在的表——这也是源码注释强调"SE 提交失败时必须回滚 DD 对象"的原因。

---

### 7. 升级

三种升级路径，`dd::init` 的参数决定走哪条：

| 参数 | 场景 | 流程 |
|---|---|---|
| `DD_INITIALIZE` | `--initialize` 新数据目录 | 只 `create_*`，不读旧版本 |
| `DD_INITIALIZE_SERVER` | **5.7 数据目录就地升级** | 预检 → 建新 DD → 迁移旧元数据 |
| `DD_RESTART_OR_UPGRADE` | 正常重启（含 8.0.x → 8.0.y） | `create_tables` + `upgrade_tables` |

#### ★★ 8.0 版本间升级：`upgrade_tables()`（DD 的"原子升级"）

8.0.x → 8.0.y 的每次重启都会问：**磁盘上的 DD 版本 ≠ 当前代码的 `DD_VERSION`？**（`is_dd_upgrade()`）是则走 `dd::upgrade::upgrade_tables()`（`sql/dd/impl/upgrade/dd.cc:1074`）——这就是"内核串行执行一批系统表 DDL"的地方，**社区版完全涉及**。

**六大步骤**（源码注释在每步前明说事务边界）：

```
① create_temporary_schemas()          // 建两个临时 schema：
                                       //   target_schema（放新版表结构）
                                       //   actual_schema（放旧版表结构）
② establish_table_name_sets()         // 算出 remove_set / create_set：
                                       //   哪些表要新建、哪些表要删
③ create_tables()                     // 遍历全部 DD 表，target != actual 就建 target 表
④ update_meta_data()                  // 阶段 1：UPDATE 源表（mysql schema 里的旧表）
   migrate_meta_data()                // 阶段 2：UPDATE/INSERT 目标表（数据搬家）
⑤ ★ ATOMIC SWITCH（三步一体，见下）:
   update_properties()                //   先改 dd_properties
   update_object_ids()                //   再改 tables / foreign_keys /
                                      //   foreign_key_column_usage 的
                                      //   id 和 schema 名（"假装 ALTER 了 schema"）
   update_versions()                  //   最后写版本号 + COMMIT
⑥ 升级后清理：FLUSH TABLES → reset Shared_dictionary_cache
   → reset DDSE dict cache（逐个 CORE 表）→ 从头重新 bootstrap
```

**★★ 事务结构（核心答案）**：

| 阶段 | 事务行为 | 源码注释原文 |
|---|---|---|
| ③ 建表 DDL | **每个 DDL 语句 auto commit**（串行执行，各自成事务） | *The table creation is done by executing DDL statements that are auto committed* |
| ④ 数据迁移 | 修改**攒在事务里不提交** | *the changes done during migration of meta data are committed in next step at the end of 'atomic switch'* |
| ⑤ Atomic Switch | **三步中间不提交、失败立即回滚、成功到 `update_versions()` 末尾才 commit** | *must be done without intermediate commits. Note that in case of failure, rollback is done immediately. In case of success, no commit is done until at the very end of update_versions()* |

即：**建表阶段是"串行 auto commit DDL"，切换阶段是"一个大事务"**。中间断开的地方（③→④）就算崩溃，重启后 `is_dd_upgrade()` 依然为真（版本号还没改），整套流程重来——**幂等性靠"版本号最后才写"天然保证**。

★ **为什么"先改 dd_properties、后改 object_ids"**（⑤内部的顺序也是刻意设计的）：注释（`:1137-1141`）说——若先改 schema id 再去 acquire 新表，Storage_adapter 会从 core registry 拿到**错误对象**（因为表的 schema id 已被改成 `mysql` 的，acquire 命中 core registry 而不是临时 schema）。所以必须先把手伸进临时 target schema 把对象取出来，再动 id。

★ **升级成功后为什么还要"从头重新 bootstrap"**：⑥ 的 `set_dd_upgrade_done()` + reset 缓存——因为升级期间 DD 缓存里装的是**旧版本对象**，全部作废重来；且 I_S 的元数据若底层表变了需要重生成（`set_dd_upgrade_done` 正是给 I_S 初始化看的标记）。

#### ★ `do_pre_checks_and_initialize_dd`：两个文件决定走哪条路

入口（`upgrade_57/upgrade.cc:837`）第一件事不是查字典，而是**摸文件系统**：

```cpp
bool do_pre_checks_and_initialize_dd(THD *thd) {
  cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());  // RAII

  build_table_filename(path, ..., "", "mysql", ".ibd", 0, &not_used);
  bool exists_mysql_tablespace = (!my_access(path, F_OK));   // ① mysql.ibd 在吗？

  build_table_filename(path, ..., "mysql", "plugin", ".frm", 0, &not_used);
  bool exists_plugin_frm = (!my_access(path, F_OK));         // ② mysql/plugin.frm 在吗？

  /* If mysql.ibd and mysql/plugin.frm do not exist, this is neither
     a restart nor an in-place upgrade case. */
  if (!exists_mysql_tablespace && !exists_plugin_frm) {
    LogErr(ERROR_LEVEL, ER_DD_UPGRADE_FAILED_FIND_VALID_DATA_DIR);
    return true;    // 两个都没有 → 不是有效数据目录，直接失败
  }
  ...
  // 读 Upgrade_status 阶段文件（若上次升级中途崩溃，从记录的阶段继续）
  Upgrade_status upgrade_status;
  bool upgrade_status_exists = upgrade_status.exists();
  Upgrade_status::enum_stage upgrade_stage =
      upgrade_status_exists ? upgrade_status.get() : enum_stage::NONE;
```

两个文件的存在性组合就是**场景判定**：

| mysql.ibd | plugin.frm | 场景 |
|---|---|---|
| 有 | 有 | 8.0 重启或**升级中断后续跑** |
| 无 | 有 | **5.7 数据目录 → 就地升级** |
| 无 | 无 | 无效数据目录，报错退出 |

★ 预检第一行就是 `Auto_releaser releaser(...)`——第 3.2 节 RAII 的实战；`Upgrade_status` 阶段文件先读后写，**中断续跑**靠它。

#### ★★ `fill_dd_and_finalize`：把 5.7 的元数据翻译进 DD（`upgrade.cc:1158`）

```cpp
bool fill_dd_and_finalize(THD *thd) {
  dd::upgrade::Bootstrap_error_handler bootstrap_error_handler;   // RAII 错误收集

  /* While migrating tables, mysql_prepare_create_table() is called which
     checks for duplicated value in SET data type. Error is reported only
     in strict sql mode. Reset sql_mode to zero while migrating. */
  thd->variables.sql_mode = 0;                       // ★① 迁移期清空 sql_mode

  std::vector<dd::String_type> db_name;
  if (find_schema_from_datadir(&db_name)) {          // ★② 扫数据目录找所有库
    terminate(thd);
    return true;
  }

  for (auto &db : db_name) {
    bool exists = false;
    dd::schema_exists(thd, db.c_str(), &exists);
    if (!exists && migrate_schema_to_dd(thd, db.c_str())) {
      terminate(thd);
      return true;                                   // schema 迁移失败 → 终止
    }

    set_allow_sdi_creation(false);                   // ★③ 迁 frm 期间不写 SDI
    if (migrate_all_frm_to_dd(thd, db.c_str(), false)) {
      error |= true;                                 // ★④ 记错误但继续跑完
    }                                                //    （把所有错误打印到日志）
    set_allow_sdi_creation(true);
  }

  /* 之后：解析 routine/view 依赖（错误降级为 warning，非 fatal） */
```

逐点说明：

- ★① **`sql_mode = 0`**：`migrate_all_frm_to_dd` 内部会走 `mysql_prepare_create_table()` 解析旧 frm——这条路径在 strict 模式下会对旧表里的 SET 类型重复值报错。**迁移旧数据必须宽容旧数据**，所以先清空 sql_mode。
- ★② 库清单来自**扫数据目录**（不是查什么表）——因为此刻 DD 里还没有任何 schema。
- ★③ **迁移 frm 期间禁止 SDI**：SDI 是元数据的冗余副本，每迁一张表都写一份纯属浪费；迁移完再统一补。`migrate_schema_to_dd` 期间 SDI 是开的（schema 有 SDI）。
- ★④ **错误收集式推进**：单张表迁移失败不中断（`error |= true` 继续循环），**把所有问题一次性暴露在日志里**——因为中断重来代价高，且 DBA 需要完整的问题清单。这与 schema 迁移失败"立即终止"形成对比：库都没建成，后面全白搭。
- 各对象的迁移入口（`upgrade_57/` 子目录每对象一个文件）：`migrate_routines_to_dd`（从 `mysql.proc`/`mysql.procs_priv` 旧 MyISAM 表读）、`migrate_events_to_dd`（从 `mysql.event` 读）——**读旧表用 server 层 TABLE 接口（直接打开 MyISAM），写新表走 DD client**。

#### ★ `terminate`：失败即删光（回滚）

```cpp
/** Drop all DD tables in case there is an error while upgrading server. */
bool terminate(THD *thd);
```

★ 升级失败**不留半成品**：把已建的 DD 表全部 DROP，配合 `Upgrade_status` 阶段文件，保证**数据目录还能被 5.7 打开**。整个升级设计成"要么完整成功、要么完全回退"——这与原子 DDL 的哲学一致，只是粒度是"整个数据目录"。

#### 版本管理与 InnoDB 侧

- ★ 版本记在 `dd_properties` 表：`DD_VERSION`（DD 结构版本）与 `MYSQL_VERSION_ID` 分开记；升级成功后 `update_versions` 重写。第（四）节的 `is_dd_upgrade_from_before(DD_VERSION_80016)`（check constraint 8.0.16 引入）判断数据来源就是它。
- InnoDB 侧（`dict0upgrade.cc`）：只有 `srv_is_upgrade_mode` 为真的升级窗口才读 5.7 的 `SYS_*` 内部表（平时这些表已不存在）→ 逐表 fill `dd::Table` 灌入新 DD（见 [`innodb_dict.md`](innodb_dict.md)）。整个过程 = **把 5.7 的两套字典（frm + SYS_*）翻译成 8.0 的一份 DD**。
- DBUG 测试点 `dd_upgrade_stage_2` 会在阶段 2 故意自杀，验证"崩溃后回滚、数据目录仍归 5.7"——**升级的崩溃安全是被测试显式守护的**。

---

### 7.4 ★ 升级前体检：`do_server_upgrade_checks()`

上面讲的是"DD 表怎么升"。但在它**之前**还有一步——`restart()` 的 `||` 链里，体检排在 `upgrade_tables()` 前面：

```cpp
  if (create_dd_schema(thd) || initialize_dd_properties(thd) ||
      create_tables(thd, nullptr) || sync_meta_data(thd) ||
      DDSE_dict_recover(...) ||
      upgrade::do_server_upgrade_checks(thd) || upgrade::upgrade_tables(thd) ||
      repopulate_charsets_and_collations(thd) || verify_contents(thd) ||
      update_versions(thd, false)) {
```

**前置门**：只在"确实是服务器版本升级 且 来源版本 > 5.7.00"时才做（`is_server_upgrade_from_after(SERVER_VERSION_50700)`）。

| 检查 | 条件 | 后果 |
|---|---|---|
| ① **分区表用了共享表空间** | 跨 8.0.13 才查 | **拒绝升级**（唯一显式 `(*error_count)++` 的非语法项） |
| ② **事件/例程/视图/触发器的语法** | 任何服务器升级都查 | 失效对象 → 经 `Syntax_error_handler` 计入错误数 → **拒绝升级** |
| ③ 前缀键分区 | 仅**警告** | 不阻断 |

② 的设计意图注释写得很清楚：

```cpp
  /*
    For any server upgrade, we will analyze events, routines, views and
    triggers and reject upgrade if we find invalid syntax or other issues
    that would not have been accepted in a CREATE statement.
    ...
    For errors that can be ignored (e.g. invalid syntax), we keep on analyzing
    to identify as many errors as possible in one go.
  */
```

即：**尽量一次收集全**，而不是遇到第一个就退。

⚠️ **一个实现细节**：`check_tables()` 里 `invalid_triggers(...)` 的**返回值被丢弃**——触发器解析失败本身不直接置错误计数，只记 `ER_UPGRADE_PARSE_ERROR` 日志。真正让它"拒绝升级"的是 `Syntax_error_handler` 把 `ER_PARSE_ERROR` 同步计进了全局计数器（见下）。

### 7.5 错误处理三件套

**① `Upgrade_error_counter`**——两个阈值语义不同：

```cpp
class Upgrade_error_counter {
  int m_error_count;
  const int ERROR_LIMIT;              // 50
 public:
  bool has_errors();                  // → 升级失败
  bool has_too_many_errors();         // → 提前停止扫描（别再收集了）
};
```

**② `Syntax_error_handler`**——`Internal_error_handler`，**只认 `ER_PARSE_ERROR` + `SL_ERROR`**：

```cpp
  if (sql_errno == ER_PARSE_ERROR && *level == Sql_condition::SL_ERROR) {
    parse_error_count++;
    if (m_global_counter) (*m_global_counter)++;   // ★ 同步计入 Upgrade_error_counter
    is_parse_error = true;
    reason = msg;
  }
```

这一行 `(*m_global_counter)++` 就是把"只看日志的 lambda"与"真正拒绝升级"接起来的关键。

**③ `Bootstrap_error_handler`**——存在的**根本原因**写在头文件注释里：bootstrap 线程执行 SQL 时，`my_message_stderr` 不遵循标准错误格式、也不设置 Diagnostics Area（会触发 assert）。所以它做三件事：走 `my_message_sql`（写进 DA 避免 assert）→ 按开关决定是否写 error log → `set_abort_on_error()`。

**allowlist 全库只有一处设置，且只放过一个错误码**：

```cpp
  /*
    Do not print error while resolving routine or view dependency from
    my_error(). ... Fatal errors will result in termination of upgrade.
  */
  bootstrap_error_handler.set_log_error(false);
  std::set<uint> allowed_errors = {ER_DEFINITION_CONTAINS_INVALID_STRING};
  bootstrap_error_handler.set_allowlist_errors(allowed_errors);
```

即：全局静音期间，**唯独"定义体含非法字符串"必须留痕**（意味着该对象定义被降级，用户必须知道）。

**唯一的"致命升级"错误码**：

```cpp
void Bootstrap_error_handler::set_abort_on_error(uint error) {
  switch (error) {
    case ER_WRONG_COLUMN_NAME: { abort_on_error = true; m_log_error = true; break; }
    default: break;
  }
}
```

消费点在视图迁移——**视图列名过长确实拒绝升级**。

### 7.6 ★ `upgrade_system_schemas()`：server 端取代 `mysql_upgrade` 客户端

头文件注释直接点明：`"This is a replacement for the mysql_upgrade client."`

**执行的 SQL 脚本**（实际是 5 组数组，注释里的"1+2"在升级代码里合并成一个 `mysql_fix_privilege_tables[]`）：

| 函数 | 数组 | 来源脚本 |
|---|---|---|
| `upgrade_firewall()` | `firewall_stored_procedures[]` | `firewall_stored_procedures.sql` |
| `fix_mysql_tables()` | `mysql_fix_privilege_tables[]` | `mysql_system_tables.sql` + `..._fix.sql` |
| `fix_mysql_tables()` | `mysql_system_tables_data_fix[]` | `mysql_system_tables_data_fix.sql` |
| `fix_sys_schema()` | `mysql_sys_schema[]` | `mysql_sys_schema.sql` |
| `upgrade_help_tables()` | `fill_help_tables[]` | `fill_help_tables.sql` |

**★ 升级时会对表做 CHECK / REPAIR**：

```cpp
    err = (... is_server_upgrade_from_before(bootstrap::SERVER_VERSION_80011) ?
               check.check_all_schemas(thd) : check.check_system_schemas(thd)) ||
          check.repair_tables(thd) ||
          dd::tables::DD_properties::instance().set(
              thd, "MYSQLD_VERSION_UPGRADED", MYSQL_VERSION_ID);
```

即：**跨 8.0.11 的升级 CHECK 全库**，否则只 CHECK `mysql` + `sys`。

做法是拼一条大的 `CHECK TABLE <所有表> FOR UPGRADE`，然后从结果集的 `Msg_text` 里 `strstr("ALTER TABLE")` 抠出建议的修复语句逐个执行；剩下的拼一条 `REPAIR TABLE`。

配套的可忽略错误表（注释明说"与 mysql_upgrade 客户端一致"）：

```cpp
static std::vector<uint> ignored_errors{
    ER_DUP_FIELDNAME, ER_DUP_KEYNAME, ER_BAD_FIELD_ERROR,
    ER_COL_COUNT_DOESNT_MATCH_PLEASE_UPDATE_V2, ER_DUP_ENTRY};
```

### 7.7 ★ 5.7 → 8.0：`upgrade_57/table.cc` 怎么把 frm 变成 DD 对象

**核心答案：复用建表那条路——先把 frm 解析回 `Create_field`，再走 `fill_dd_table_from_create_info`（见 §2.6）。**

调用顺序即证据（全在 `migrate_table_to_dd()` 内）：

```
create_table_share_for_upgrade()  读 .frm → TABLE_SHARE + FRM_context
  → prepare_fields_and_keys()     产出 List<Create_field>
  → mysql_prepare_create_table()  （FK 留空，后面由 SE 补）
  → prepare_create_field()        逐字段
  → fill_partition_info_for_upgrade()
  → dd::create_dd_user_table()    → fill_dd_table_from_create_info()
  → dd_client()->store()
  → set_se_data_for_user_tables() SE 补 tablespace / FK / 分区 / row_type
  → fix_fk_parent_key_names()
  → add_triggers_to_table()
```

**① `.frm` 怎么读**：`read_frm_file()` → `open_binary_frm()`。后者注释明确说明它是**专为升级保留的老解析器**：

```
  This function is added to read .frm file in upgrade scenario. It should not
  be used any where else in the code. This function will be removed later.
```

解析产物落在 **`FRM_context`**——一批 8.0 的 `TABLE_SHARE` 已不保留、只为升级而存在的字段：

```cpp
/**
  These members were removed from TABLE_SHARE as they are not used in
  the code. open_binary_frm() uses these members while reading .frm files.
*/
class FRM_context {
  handlerton *default_part_db_type;
  bool null_field_first;
  uint stored_fields;
  enum utype { NONE, DATE, SHIELD, ..., TIMESTAMP_OLD_FIELD, ..., GENERATED_FIELD = 128 };
  const File_parser *view_def;
  uchar frm_version;
  TYPELIB fieldnames;
};
```

那些 `utype` 就是 5.7 时代 frm 的字段修饰位（auto_increment、老 timestamp、bit、enum-set、生成列……）。

**② 只保留 DD 认的 9 个选项位**：

```cpp
  // DD framework handles only these options
  uint db_create_options = table->s->db_create_options;
  db_create_options &=
      (HA_OPTION_PACK_RECORD | HA_OPTION_PACK_KEYS | HA_OPTION_NO_PACK_KEYS |
       HA_OPTION_CHECKSUM | HA_OPTION_NO_CHECKSUM | HA_OPTION_DELAY_KEY_WRITE |
       HA_OPTION_NO_DELAY_KEY_WRITE | HA_OPTION_STATS_PERSISTENT |
       HA_OPTION_NO_STATS_PERSISTENT);
```

**③ `.TRG` 文件**：8 个字段描述符——`triggers` / `sql_modes` / `definers` / `client_cs_names` / `connection_cl_names` / `db_cl_names` / `created`。解析后逐条构造 `::Trigger`，其中大量**老版本兼容降级**（sql_mode 缺失取全局、字符集缺失取默认值、`created` 缺失说明是 5.7.2 之前建的）。还有一个专门的 BUG#14090 兼容钩子处理写错的 `sql_modes` 字段。

⚠️ **`.TRN` 源码没有任何解析逻辑**——`TRN_EXT` 只出现在常量定义与"哪些文件要搬到 `backup_metadata_57`"的清理清单里。8.0 完全只从 `.TRG` 取全量。

**④ `.par` 分区文件**：**升级不解析它**——分区信息在 `.frm` 里（`share->partition_info_str`），由 `unpack_partition_info()` 还原。`PAR_EXT` 同样只用于清理。

**⑤ 无法迁移的表**：三档处理

| 情形 | 处理 |
|---|---|
| `read_frm_file()` 返回 `-2` | "Less severe error, file can safely be ignored"（如 ndbinfo 引擎未启用） |
| **老时间类型** | `HA_ADMIN_NEEDS_DUMP_UPGRADE` → **硬失败**（注释：`"it's not possible to upgrade such tables"`） |
| 一般失败 | `error \|= result` + `thd->clear_error()` **继续扫全库**（"want to process all tables"），唯一提前退出是错误数 > 50 |

最终错误传回 `fill_dd_and_finalize()` → `terminate(thd)`。

### 7.8 routine / event / schema 的迁移

**① 存储过程/函数**（`upgrade_57/routine.cc`）：从 **`mysql.proc` 真表**读（这是少数几处元数据源是真表而非文件的）。

⚠️ **两套老表定义**（5.7.13 分界，唯一差别是 `definer char(77)` → `char(93)`），校验时**先试新的、再试老的**。

特征字段是**按首字母**从 ENUM 列里抠出来的：

```cpp
  switch (ptr[0]) {
    case 'N': chistics->daccess = SP_NO_SQL; break;
    case 'C': chistics->daccess = SP_CONTAINS_SQL; break;
    ...
  chistics->detistic = (ptr[0] != 'N');
  chistics->suid = (ptr[0] == 'I' ? SP_IS_NOT_SUID : SP_IS_SUID);
```

★ **解析失败的处理很巧妙**：先用**空体**重建（`FUNCTION → RETURN NULL`、`PROCEDURE → BEGIN END`），再把真实 body 塞回去——保证对象不丢：

```cpp
    if (routine_type == enum_sp_type::FUNCTION) sr_body = {STRING_WITH_LEN("RETURN NULL")};
    else                                        sr_body = {STRING_WITH_LEN("BEGIN END")};
    if (db_load_routine(..., sr_body.str, ...)) goto err;
    // Set actual routine body.
    sp->m_body.str = body;
```

（sys schema 的失效例程不打警告——"will get fixed when mysql_upgrade is executed"。）

**② 事件**（`upgrade_57/event.cc`）：同样两套表定义 + 先新后老。三个要点：

- **时区基础设施要先起来**（`my_tz_init`），因为升级期还没有
- ⚠️ **`time_zone` 列是 latin1**：`String tz_str(tz_name.str, &my_charset_latin1)`
- ★ DATETIME → UTC epoch 用 **`my_tz_OFFSET0->TIME_to_gmt_sec()`**（`my_tz_OFFSET0` 即 UTC），`starts` / `ends` / `execute_at` / `last_executed` 全部如此
- `interval` **按名字映射而非枚举值**，注释解释：*"This decouples the values of enum interval_type and values actually stored on disk."*

**③ 库**（`upgrade_57/schema.cc`）：读 **`db.opt`** 文件，抠 `default-character-set=` / `default-collation=`，且做了 4.1.0 老格式的兼容（先按字符集名找，失败再按排序规则名找）。`l_c_t_n=1` 时大写库名直接报错。

## ★ DD 里的 C++ 运用

> 与 [`../infra/io_cache.md`](../infra/io_cache.md)「这套体系里的 C++ 运用」对照读：那一篇讲 `IO_CACHE` 外层的 ostream 体系（装饰者、`unique_ptr` 转移、`= delete`、duck-typing 模板），这一篇讲 DD 的**双类体系、菱形虚继承、嵌套 RAII、tag dispatch**。两者是 MySQL server 层 C++ 最复杂的两块。

### 0. DD 的 C++ 风格与"现代 C++"的三点不同

先讲结论，避免用常规直觉去读 DD 代码：

| 常规现代 C++ 直觉 | DD 的实际做法 | 原因 |
|---|---|---|
| 资源类应该 `= delete` 拷贝 | **几乎不 delete**（全仓库只 3 处） | ★ `clone()` 体系**依赖**拷贝构造函数做深拷贝（见 §1.3） |
| 共享所有权用 `shared_ptr` | **0 处 `shared_ptr`** | 缓存的引用计数是 `Cache_element::m_ref_counter`（手写 `uint` + 外部加锁） |
| 用 `enable_if` / CRTP 做类型派发 | **都没有** | 用**嵌套 typedef traits** + **空 tag 类型的函数重载**（见 §3、§4） |

### 1. 双类体系：接口 `X` + 实现 `X_impl`

DD 最显著的设计：`sql/dd/types/*.h` 是**纯虚接口**，`sql/dd/impl/types/*_impl.h` 是 **public 继承接口的实现类**。

#### 1.1 继承树

```
                        dd::Weak_object          (types/weak_object.h)
                        ─ 纯虚 debug_print()
                        ─ @note: 直接子类必须 **虚继承**
                                        ▲ virtual public
        ┌───────────────────────────────┴──────────────────────────┐
        │                                                           │
  dd::Entity_object                          dd::Weak_object_impl_<bool use_pfs>
  : virtual public Weak_object               ─ ★ 模板！bool 非类型参数
  ─ ★ private virtual impl()                 ─ 自定义 operator new/delete → my_malloc(PFS)
  ─   friend Storage_adapter /                using Weak_object_impl = Weak_object_impl_<true>;
  ─   Entity_object_table_impl                        │
        │                                             │
        └───────────────┬─────────────────────────────┘
                        │
        dd::Entity_object_impl : virtual public Entity_object,
                                 public Weak_object_impl
                        │
   ┌────────────────────┼────────────────────┬──────────────┐
   │                    │                    │              │
Schema_impl       Abstract_table_impl    Event_impl    Tablespace_impl
: Entity_object_impl,  : Entity_object_impl, ...       ...
  public Schema        virtual public Abstract_table
                             │
                  ┌──────────┴──────────┐
                  │                     │
             Table_impl              View_impl
             : Abstract_table_impl,   : Abstract_table_impl,
               virtual public Table     public View
```

接口侧全部 `virtual public`，为菱形服务：

```
Weak_object
 └─ Entity_object : virtual public Weak_object
     ├─ Abstract_table : virtual public Entity_object
     │   ├─ Table : virtual public Abstract_table
     │   └─ View  : virtual public Abstract_table
     ├─ Schema / Tablespace / Index / Column / Event / Routine / Charset / Collation / ...
     └─ Trigger
```

#### 1.2 接口类长什么样（`dd::Table`，`types/table.h:48`）

```cpp
class Table : virtual public Abstract_table {
 public:
  typedef Table_impl Impl;                    // ★ 关键 traits：接口→实现的反向指针
  typedef Collection<Index *> Index_collection;
  enum enum_row_format { RF_FIXED = 1, RF_DYNAMIC, RF_COMPRESSED, ... };

  ~Table() override = default;

  // ---- 全部是纯虚 getter/setter，★ 无数据成员 ----
  virtual Object_id tablespace_id() const = 0;
  virtual const String_type &engine() const = 0;
  virtual const Properties &se_private_data() const = 0;
  virtual Properties &se_private_data() = 0;          // ★ const/non-const 双重载
  virtual Index *add_index() = 0;
  virtual const Index_collection &indexes() const = 0;
  virtual Index_collection *indexes() = 0;            // ★ 同上
  // ... ~80 个纯虚 ...

  // ---- 三个关键虚函数 ----
  Table *clone() const override = 0;                     // ★ 协变返回类型
  Table *clone_dropped_object_placeholder() const override = 0;
  virtual void serialize(Sdi_wcontext *wctx, Sdi_writer *w) const = 0;
  virtual bool deserialize(Sdi_rcontext *rctx, const RJ_Value &val) = 0;
};
```

#### 1.3 实现类（`dd::Table_impl`，`impl/types/table_impl.h:68`）

```cpp
class Table_impl : public Abstract_table_impl, virtual public Table {
 public:
  bool restore_children(Open_dictionary_tables_ctx *otx) override;  // 子对象树加载
  bool store_children(Open_dictionary_tables_ctx *otx) override;
  bool restore_attributes(const Raw_record &r) override;            // 行 → 成员
  bool store_attributes(Raw_record *r) override;                    // 成员 → 行
  /* ... */

  // ★★★ "Fix 'inherits ... via dominance' warnings" —— 菱形消歧（约 40 处）
  Entity_object_impl *impl() override { return Entity_object_impl::impl(); }
  Object_id id() const override { return Entity_object_impl::id(); }
  const String_type &name() const override { return Entity_object_impl::name(); }
  Object_id schema_id() const override { return Abstract_table_impl::schema_id(); }
  // ...

 private:
  Object_id m_se_private_id;
  String_type m_engine;
  uint m_last_checked_for_upgrade_version_id = 0;      // NSDMI
  /* ... Composite：子对象集合 ... */
  Index_collection m_indexes;

  Table_impl(const Table_impl &src);                   // ★ 私有拷贝构造（深拷贝整棵树）
  Table_impl *clone() const override { return new Table_impl(*this); }

  Table_impl *clone_dropped_object_placeholder() const override {
    Table_impl *placeholder = new Table_impl();
    placeholder->set_id(id());                          // 只保留 5 个 key 字段
    placeholder->set_schema_id(schema_id());
    /* ... */
    return placeholder;
  }
};
```

**为什么 `_impl` 类"看起来不能"直接构造**：其实构造函数是 **public** 的。真正的原因是三层：

1. **物理隔离**：`sql/dd/types/*.h` 只前向声明 `class Table_impl;`，实现头在 `impl/` 下。server 层代码只 include `types/`，**物理上构造不了**。
2. **工厂 `dd::create_object<X>()` 是官方入口**：
   ```cpp
   // sql/dd/impl/dd.cc:78
   template <typename X>
   X *create_object() {
     return dynamic_cast<X *>(new (std::nothrow) typename X::Impl());
   }
   ```
   ★ 返回**接口指针** `X*`，调用者永远看不到 `X_impl` 的成员。且用 **`new (std::nothrow)`**——DD 全程不用异常。
3. **拷贝构造是私有的**：只能通过 `clone()` 调用（成员函数能访问私有拷贝构造），因为 `clone()` 要做**深拷贝整棵对象树**（`m_indexes` / `m_partitions` 里每个元素都要拷）。

### 2. ★ 菱形虚继承与 "inherits via dominance"

#### 2.1 为什么要全 `virtual public`

**源码里有 4 处措辞完全一致的 `@note`**（分别在 `weak_object.h:36-39`、`entity_object.h:42-45`、`abstract_table.h:63-66`、`object_table.h:66-69`）：

```cpp
/**
  Base class for all data dictionary objects.

  @note This class may be inherited along different paths
        for some subclasses due to the diamond shaped
        inheritance hierarchy; thus, direct subclasses
        must inherit this class virtually.
*/

class Weak_object { ... };
```

**`dd::Table` 的真实继承链**（接口侧全 `virtual public`，实现侧则"接口那条腿 virtual、impl 那条腿不 virtual"）：

```cpp
/* 接口侧 —— sql/dd/types/ */
class Weak_object { ... };                              // weak_object.h:42
class Entity_object : virtual public Weak_object { ... };        // entity_object.h:48
class Abstract_table : virtual public Entity_object { ... };     // abstract_table.h:69
class Table : virtual public Abstract_table { ... };             // table.h:48

/* 实现侧 —— sql/dd/impl/types/ */
class Weak_object_impl_ : virtual public Weak_object { ... };    // weak_object_impl.h:48
using Weak_object_impl = Weak_object_impl_<true>;
class Entity_object_impl : virtual public Entity_object,
                           public Weak_object_impl { ... };      // entity_object_impl.h:43-44
class Abstract_table_impl : public Entity_object_impl,
                            virtual public Abstract_table { ... };  // abstract_table_impl.h:57-58
class Table_impl : public Abstract_table_impl,
                   virtual public Table { ... };                 // table_impl.h:68
```

⇒ `Entity_object` 经两条路径到达 `Table_impl`（`Table_impl → Table → Abstract_table → Entity_object` 与 `Table_impl → Abstract_table_impl → Entity_object_impl → Entity_object`），**必须 virtual**。

**★ 重要区分**：那 23 处 `Fix "inherits ... via dominance" warnings` **不是设计说明**，只是**消除编译器告警的转发代码**。因为虚继承下两个基类都实现了同名虚函数，编译器不知道该选哪个，必须显式指定：

```cpp
// sql/dd/impl/types/abstract_table_impl.h:174-188
  // Fix "inherits ... via dominance" warnings
  Entity_object_impl *impl() override { return Entity_object_impl::impl(); }
  const Entity_object_impl *impl() const override {
    return Entity_object_impl::impl();
  }
  Object_id id() const override { return Entity_object_impl::id(); }
  bool is_persistent() const override {
    return Entity_object_impl::is_persistent();
  }
  const String_type &name() const override {
    return Entity_object_impl::name();
  }
  void set_name(const String_type &name) override {
    Entity_object_impl::set_name(name);
  }
```

`Table_impl` 有两层菱形，所以除了 `Entity_object_impl::*` 还要显式指定 `Abstract_table_impl::*`：

```cpp
// sql/dd/impl/types/table_impl.h:428-436
  Object_id schema_id() const override {
    return Abstract_table_impl::schema_id();
  }
  void set_schema_id(Object_id schema_id) override {
    Abstract_table_impl::set_schema_id(schema_id);
  }
```

grep `dominance` 命中 **23 个文件、23 处**，全部在 `sql/dd/impl/types/*_impl.h`。

#### 2.2 ★ 类型派发用的是嵌套 typedef traits，不是 `Object_type` 枚举

每个 DD 类型在**自己类里**声明一套 typedef，供缓存与键查找做编译期派发：

```cpp
// sql/dd/types/abstract_table.h:71-77
class Abstract_table : virtual public Entity_object {
 public:
  typedef Abstract_table_impl Impl;
  typedef Abstract_table Cache_partition;
  typedef tables::Tables DD_table;
  typedef Primary_id_key Id_key;
  typedef Item_name_key Name_key;
  typedef Se_private_id_key Aux_key;
  typedef Collection<Column *> Column_collection;
```

> ★ `Cache_partition` 只有 **10 个定义**（10 个"缓存分区根类型"）：`collation.h:49`、`tablespace.h:58`、`abstract_table.h:72`、`spatial_reference_system.h:56`、`column_statistics.h:60`、`schema.h:66`、`resource_group.h:52`、`event.h:51`、`routine.h:64`、`charset.h:49`。
> ⚠️ **`Table` 与 `View` 都没有重新定义 `Cache_partition`**——它们都归到 `Abstract_table` 分区。

缓存元素 `Cache_element<T>` 把三种键存成成员，用 `Type_selector<K>` 重载做派发：

```cpp
// sql/dd/impl/cache/cache_element.h:76-124
 private:
  const T *m_object;
  uint m_ref_counter;

  template <typename K>
  class Key_wrapper {
   public:
    bool is_null;
    K key;
  };

  Key_wrapper<typename T::Id_key> m_id_key;      // The id key for the object.
  Key_wrapper<typename T::Name_key> m_name_key;  // The name key for the object.
  Key_wrapper<typename T::Aux_key> m_aux_key;    // The aux key for the object.

  template <typename K>
  struct Type_selector {};

  const typename T::Id_key *get_key(Type_selector<typename T::Id_key>) const {
    return id_key();
  }
  const typename T::Name_key *get_key(Type_selector<typename T::Name_key>) const {
    return name_key();
  }
  const typename T::Aux_key *get_key(Type_selector<typename T::Aux_key>) const {
    return aux_key();
  }
```

键类层次（`Object_key` 抽象基类在 **`sql/dd/impl/object_key.h:38`**，子类在 `sql/dd/impl/raw/object_keys.h`）：

```cpp
class Object_key {
 public:
  virtual Raw_key *create_access_key(Raw_table *t) const = 0;
  virtual String_type str() const = 0;
  virtual ~Object_key() = default;
};
```

```cpp
// NOTE: the current naming convention is as follows:
// - use '_key' suffix to name keys identifying 0 or 1 row;
// - use '_range_key' suffix to name keys identifying 0 or N rows.
```

| 键类 | 行号 | 用途 |
|---|---|---|
| `Void_key` | 54 | 该对象类型不支持此键的占位（仍需 `operator<`，因为要当模板实参） |
| `Primary_id_key` | 77 | 全局对象 id 主键 |
| `Parent_id_range_key` | 103 | 按父 id 范围查子对象 |
| `Global_name_key` | 126 | 全局唯一名 |
| `Item_name_key` | 169 | 容器内唯一名 `(container_id, name)` |
| `Se_private_id_key` | 222 | SE 私有 id |
| `Composite_pk` / `Composite_char_key` / `Composite_4char_key` | 271 / 298 / 326 | 复合键 |
| `Routine_name_key` | 438 | `(schema_id, type, name)` |
| `Table_reference_range_key` | 492 | catalog / schema / table |

`Object_id` 本身只是个整数 typedef，被"装进" key 再变成物理 `Raw_key`：

```cpp
typedef unsigned long long Object_id;
const Object_id INVALID_OBJECT_ID = (Object_id)-1;
```

```cpp
Raw_key *Primary_id_key::create_access_key(Raw_table *db_table) const {
  const int ID_INDEX_NO = static_cast<int>(Object_table_impl::Common_index::PK_ID);
  assert(ID_INDEX_NO == 0);
  const int ID_COLUMN_NO = static_cast<int>(Object_table_impl::Common_field::ID);
  assert(ID_COLUMN_NO == 0);
  TABLE *t = db_table->get_table();
  t->use_all_columns();
  t->field[ID_COLUMN_NO]->store(m_object_id, true);
  ...
```

```cpp
struct Raw_key {
  uchar key[MAX_KEY_LENGTH];
  int index_no;
  int key_len;
  // bitmap of key parts that are used in the index
  key_part_map keypart_map;
};
```

`Table_impl` 同时继承 `Abstract_table_impl`（含 `Entity_object_impl` 实现）和 `Table`（含 `Abstract_table` 接口）。如果不用虚继承，就会出现**两个 `Weak_object` 子对象**，导致：

- 指针转换歧义（`Table_impl*` → `Weak_object*` 不知走哪条路）
- 菱形顶部的成员重复

所以从 `Weak_object` 开始**全部虚继承**，保证整个体系只有一个 `Weak_object` 子对象。

#### 2.2 "inherits via dominance" 是什么

菱形下，同名虚函数在两条路径上都有实现时，GCC 会警告 *"overloaded virtual function is inherited via dominance"*。DD 的解法是**在最终派生类里显式转发**：

`Entity_object_table_impl`（`impl/types/entity_object_table_impl.h:45`）的全部方法体都是转发，目的只有一个——消警告：

```cpp
class Entity_object_table_impl : public Object_table_impl,
                                 public Entity_object_table {
 public:
  bool restore_object_from_record(...) const override;
  // Fix "inherits ... via dominance" warnings
  const String_type &name() const override { return Object_table_impl::name(); }
  Object_table_definition_impl *target_table_definition() override {
    return Object_table_impl::target_table_definition();
  }
  /* ... 共 10 个纯转发 ... */
};
```

`Table_impl` 里同样有约 40 处这样的转发。**这是菱形继承在真实工程里的"税"**——每加一层就要补一批转发。

### 3. ★ 类型派发：嵌套 typedef traits（不是 `Object_type` 枚举）

> **澄清**：DD 里**不存在** `dd::Object_type` 这种运行时类型枚举。全部靠 C++ 类型 + 嵌套 typedef 做**编译期**派发。

每个可缓存的 DD 接口类都提供一组嵌套 typedef：

| typedef | 含义 |
|---------|------|
| `Impl` | 对应的实现类（工厂用它 `new X::Impl()`） |
| `DD_table` | 对应的 `mysql.*` 表描述类（如 `tables::Tables`） |
| `Id_key` | 按 id 查找的 key 类 |
| `Name_key` | 按名字查找的 key 类 |
| `Aux_key` | 第三套索引 key（如 `(engine, se_private_id)`） |
| `Cache_partition` | **缓存分区粒度**（见下） |

派发实例：

```cpp
// impl/transaction_impl.h:87 —— 由 T::DD_table 决定打开哪张表
template <typename T>
Raw_table *get_table() const { return get_table(T::DD_table::instance().name()); }

// impl/transaction_impl.h:92
template <typename X>
void register_tables() { X::Impl::register_tables(this); }   // static dispatch
```

#### ★ `Cache_partition` 为什么是 `Abstract_table`

`mysql.tables` **同时存 `Table` 和 `View`**。所以缓存分区粒度只能是它们的共同基类 `Abstract_table`：

```
acquire<Table>()
  → 按 Table::Cache_partition == Abstract_table 去 Shared_multi_map<Abstract_table> 查
  → 拿到后 dynamic_cast<const Table*>(cached_object)
  → cast 失败说明实际是 View → 返回 nullptr（合法）
```

这解释了 `Dictionary_client::acquire()` 里为什么到处是 `dynamic_cast` + `transfer_release`（见 §5.3）。

### 4. ★ `Multi_map_base` 的 tag dispatch（为什么不用 SFINAE）

`Multi_map_base<T>` 持有四张不同 key 类型的 map，要按 key 类型选出对应 map：

```cpp
template <typename T>
class Multi_map_base {
 private:
  Element_map<const T *, Cache_element<T>>              m_rev_map;
  Element_map<typename T::Id_key,   Cache_element<T>>   m_id_map;
  Element_map<typename T::Name_key, Cache_element<T>>   m_name_map;
  Element_map<typename T::Aux_key,  Cache_element<T>>   m_aux_map;

  template <typename K> struct Type_selector {};        // ★ 空 tag 类型

  Element_map<const T *, Cache_element<T>> *m_map(Type_selector<const T *>) { return &m_rev_map; }
  const Element_map<const T *, Cache_element<T>> *m_map(Type_selector<const T *>) const { return &m_rev_map; }
  Element_map<typename T::Id_key, Cache_element<T>> *m_map(Type_selector<typename T::Id_key>) { return &m_id_map; }
  /* ... Name_key / Aux_key 各 2 个重载 ... */

 protected:
  // ★ 对外只暴露模板版本；注释明说 "We must use overloading to accomplish this"
  template <typename K> Element_map<K, Cache_element<T>> *m_map() {
    return m_map(Type_selector<K>());
  }
};
```

**★ 为什么不能用 `std::enable_if`**：`T::Name_key` 与 `T::Id_key` **可能是同一个类型**（例如某些类型两者都是 `Primary_id_key`）。SFINAE 会让重载集产生歧义；而 `Type_selector<K>` 是独立类型，只按 `K` **精确匹配**，不会退化。

**代价**：派生类要用这种"怪语法"（`local_multi_map.h:70`）：

```cpp
template <typename K>
Element_map<K, Cache_element<T>> *m_map() {
  return Multi_map_base<T>::template m_map<K>();   // ★ template 关键字不可省
}
```

### 5. ★ `Auto_releaser`：嵌套 RAII 与所有权上移

#### 5.1 声明（`cache/dictionary_client.h:178`）

```cpp
class Auto_releaser {
  friend class Dictionary_client;
 private:
  Dictionary_client *m_client;
  Object_registry m_release_registry;   // ★ 值成员：自带一份 registry
  Auto_releaser *m_prev;                // ★ 链表前驱 = 栈式嵌套

  template <typename T> void auto_release(Cache_element<T> *element) {
    assert(m_prev != nullptr);          // ★ default releaser 不能 auto_release
    m_release_registry.put(element);
  }
  template <typename T> void transfer_release(const T *object);   // 上移到外层
  template <typename T> Auto_releaser *remove(Cache_element<T> *element);

  Auto_releaser();                      // 私有：只给 m_default_releaser 用
 public:
  explicit Auto_releaser(Dictionary_client *client);
  ~Auto_releaser();
};
```

#### 5.2 构造 / 析构：强制 LIFO

```cpp
Auto_releaser::Auto_releaser(Dictionary_client *client)
    : m_client(client), m_prev(client->m_current_releaser) {
  m_client->m_current_releaser = this;                  // 压栈
}

Auto_releaser::~Auto_releaser() {
  // Make sure that we destroy auto_releaser object in LIFO order.
  assert(m_client->m_current_releaser == this);         // ★ 强制 LIFO

  m_client->release<Abstract_table>(&m_release_registry);
  m_client->release<Schema>(&m_release_registry);
  /* ... 9 个类型手工展开 ... */

  m_client->m_current_releaser = m_prev;                // 出栈

  if (m_client->m_current_releaser == &m_client->m_default_releaser) {
    // 回到栈底：整个语句结束
    if (!m_client->m_thd->m_transactional_ddl.inited())
      m_client->m_registry_uncommitted.erase_all();
    m_client->m_registry_dropped.erase_all();
    delete_container_pointers(m_client->m_uncached_objects);
  }
}
```

**嵌套结构**：

```
栈（自下而上 = 声明顺序）：

  m_default_releaser  (m_prev == nullptr, 私有构造)
      ▲
  Auto_releaser R1    ← 函数开头
      ▲
  Auto_releaser R2    ← acquire<Table>() 内部
      ▲
  Auto_releaser R3    ← 更深层
      ▲
  m_current_releaser ┘

销毁必须严格 LIFO；回到 default releaser 才清 uncommitted/dropped
```

#### 5.3 ★ `transfer_release`：所有权从内层上移到外层

`acquire()` 内部为了处理 `dynamic_cast` 失败自己开了一个 releaser；若 cast 成功、对象要**返回给调用者**，它的生命周期不能随内部 releaser 结束：

```cpp
template <typename T>
bool Dictionary_client::acquire(Object_id id, const T **object) {
  const typename T::Id_key key(id);
  const typename T::Cache_partition *cached_object = nullptr;
  Auto_releaser releaser(this);                    // ← 内部 releaser
  bool error = acquire(key, &cached_object, &local_committed, &local_uncommitted);
  if (!error) {
    // Dynamic cast may legitimately return NULL if we e.g. asked
    // for a dd::Table and got a dd::View in return.
    *object = dynamic_cast<const T *>(cached_object);
    if (!local_committed && !local_uncommitted && *object)
      releaser.transfer_release(cached_object);    // ★ 过户给外层，延长生命周期
  }
  return error;
}
```

```cpp
template <typename T>
void Dictionary_client::Auto_releaser::transfer_release(const T *object) {
  Cache_element<T> *element = nullptr;
  m_release_registry.get(object, &element);
  m_release_registry.remove(element);               // 从本层摘掉
  m_prev->auto_release(element);                    // 交给外层
}
```

#### 5.4 ★ 与 MDL 的声明顺序是硬性约定

**`Auto_releaser` 本身不持有 MDL ticket**——MDL 由另一个 RAII 类 `Schema_MDL_locker` 持有。两者的**声明顺序写死在注释里**（`dd_schema.h:86`）：

```cpp
  // We must make sure the schema is released and unlocked in the right order.
  Schema_MDL_locker mdl_locker(m_thd);   // ① 先声明 → 后析构
  Auto_releaser releaser(this);          // ② 后声明 → 先析构
```

> **顺序错了会发生什么**（`dictionary_client.cc:1044` 注释）：若 releaser 先于 mdl_locker 析构，会有一段"schema 的 MDL 已释放、但对象还被本地引用"的窗口——**另一个线程可能拿到 X 锁，而缓存元素的引用计数仍 > 0**，触发 shared cache 的 assert，并为不当使用打开口子。

### 6. 缓存三级结构与可见性

`Dictionary_client` 用**三个 `Object_registry` 实例**实现三级：

| 级别 | 成员 | 含义 |
|------|------|------|
| L1a | `m_registry_committed` | 已与 `mysql.*` 表一致的 dd-object |
| L1b | `m_registry_uncommitted` | 本线程已改、**未提交** |
| L1c | `m_registry_dropped` | 本线程已删、**未提交** |

```cpp
std::vector<Entity_object *> m_uncached_objects;  // acquire_uncached() 的产物，最后统一 delete
Object_registry m_registry_committed;
Object_registry m_registry_uncommitted;
Object_registry m_registry_dropped;
```

**可见性判定**（`acquire<K,T>`，三级 + 三次复查）：

| 场景 | 结果 |
|------|------|
| 命中 uncommitted | **返回 uncommitted 版本**（本事务自己的修改优先可见） |
| 命中 dropped | **返回 nullptr**（其他线程仍可见旧版） |
| 命中 committed | 再**用 id key 复查 uncommitted**——因为对象可能已被改名或删除 |
| 都没命中 | 走 `Shared_dictionary_cache` → 可能读盘；读到后**再复查一次**（防止 cache miss 期间本线程改名/删了它） |

> **为什么要用 id key 复查**：按 name key 命中后，对象可能已被 rename，旧名字应视为"不存在"。这是"**同一对象多个版本**"问题的标准处理方式。

### 7. 其他 C++ 特性清单

| 特性 | 位置 | 说明 |
|---|---|---|
| **非类型模板参数 `bool`** | `impl/types/weak_object_impl.h:47` `template <bool use_pfs> class Weak_object_impl_` | 编译期开关 PFS 内存统计；`using Weak_object_impl = Weak_object_impl_<true>;` |
| **`if (use_pfs)` 模拟 `if constexpr`** | 同上 | 运行期 `if`，但由于 `use_pfs` 是编译期常量，编译器完全优化掉另一分支（C++11 写法） |
| **自定义 `operator new/delete` + `noexcept`** | 同上（唯一 4 处 `noexcept`） | `my_malloc(key_memory_DD_objects, ...)` |
| **别名模板 `using`** | `string_type.h:48/56`、`dictionary_client.h:481` | `Char_string_template<A>`、`Const_ptr_vec<T>` |
| **显式实例化** | `dd.cc:83-135`（25 个 `create_object` + 9 个 `create_map`）、`dictionary_client.cc`（~120 个） | ★ 控制编译时间 + 把模板实现藏进 `.cc` |
| **显式特化** | `dictionary_client.cc:2625/2632`（`store<Table_stat>` / `store<Index_stat>`——这两种**不进缓存**） | |
| **`std::hash` 特化** | `string_type.h:88` | 侵入 `namespace std`，用 murmur3_32 |
| **自定义 allocator** | `string_type.h:43`、`element_map.h:75`、`free_list.h:55` | `Malloc_allocator<T>` 全 DD 缓存统一走 PSI |
| **`Collection(const Collection&) = delete`** | `collection.h:164` | ★ 注意：写成了 `void operator=(Collection &)`（参数漏了 const），属于笔误级写法 |
| **`enum class`** | 16 个文件（`enum_dd_init_type`、`enum_table_type`、`Stage`、`Common_index` 等） | 内部状态机用 scoped |
| **unscoped enum** | `types/table.h:81` `enum_row_format` 等 | ★ **面向 SQL 的定义（字段序号、ENUM 列值）故意用 unscoped**，因为要 `static_cast` 成 int 写进 `mysql.tables` |
| **`[[nodiscard]]`** | 43 处，集中在 `dictionary_client.h`(36) / `dictionary.h`(6) | DD **直接用标准 `[[nodiscard]]`**，没有 `NODISCARD` 宏 |
| **`std::optional`** | 11 处（`column.h`、`spatial_reference_system.h`） | 可空字段 |
| **`std::string_view`** | 1 处（`dictionary_client.cc:595`） | |
| **`std::variant`** | **0 处** | |
| **`override`** | 大量（`table_impl.h` 约 120 处） | **`final` 0 处** |
| **`= delete`** | **仅 3 处**：`Properties::operator=`、`Import_target` 拷贝（move-only）、`Collection` 拷贝 | ★ 少的原因见 §0 |

### 8. SDI 与"编译防火墙"

`sql/dd/sdi_fwd.h` 只 include `rapidjson/fwd.h`（前向声明），把重型 rapidjson 模板**挡在所有 DD 头文件之外**：

```cpp
namespace dd {
typedef rapidjson::GenericValue<RJ_Encoding, RJ_Allocator> RJ_Value;
using RJ_Writer = rapidjson::Writer<...>;
using Sdi_writer = RJ_Writer;
class Sdi_rcontext;
class Sdi_wcontext;
}
```

这就是为什么 `table.h` 能写 `virtual void serialize(Sdi_wcontext *wctx, Sdi_writer *w) const = 0;` 而**不需要暴露 rapidjson 实现**。

SDI 版本历史里还留着两个真实 bug 的记录（`impl/sdi.h:83-90`）：

```
  80016: - Bug#29210646: DD::INDEX_IMPL::M_HIDDEN NOT INCLUDED IN SDI
  80019: - Bug#30326020: SUBPARTITIONING NOT REFLECTED IN SDI
```

### 9. 反直觉点 / 坑

| 坑 | 说明 |
|---|---|
| **`clone()` 依赖拷贝构造，所以 DD 类不能 `= delete` 拷贝** | 与常规"现代 C++"直觉相反。`Table_impl` 的拷贝构造是**私有**的，只能被 `clone()` 调用 |
| **`Void_key::operator<` 比较地址** | `bool operator<(const Void_key &rhs) const { return this < &rhs; }` —— 因为没有任何成员可比较，而 `Element_map` 用 `std::map` 需要严格弱序 |
| **`Object_registry` 的 9 个 map 是 lazy `unique_ptr`** | `get()` 时若 map 未创建直接返回 nullptr（"没建过 ⇒ 对象不存在"）；`remove()` 则 assert map 必须已存在 |
| **`create_map` 被刻意放到 `dd.cc` 显式实例化** | 注释明说："*it is big, and because it takes a lot of time for the compiler to instantiate*" |
| **引用计数不是原子的** | `cache_element.h:61` 注释："*Locking at an outer level takes care of race conditions*"——锁在 `Shared_multi_map<T>::m_lock` |
| **`Free_list` 是 `std::vector` + 线性查找移除** | O(n)，因为 LRU 链通常很短 |
| **`Raw_key` 是纯 POD `struct` 不是 class** | 含 `uchar key[MAX_KEY_LENGTH]`（栈上大数组），是与 server `Field*` 交互的 C 边界 |
| **DD 表定义是 C++ 硬编码** | `add_field(FIELD_ID, "FIELD_ID", "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT")`——**不能从元数据读**，因为要自举 |

---

## Misc

### ★★★ 真名勘误与数字核实

| 常见说法 | 8.0.39 实际 |
|---|---|
| `System_tables::init()` 里注册 DD 表 | **没有 `init()`**。是两个函数：`add_inert_dd_tables()`（:151）+ `add_remaining_dd_tables()`（:165） |
| `register_table<X>(name, type)` | 模板重载**无参**：`register_table<X>(type)`；另有字符串重载 `register_table(String_type, type)`（**第 4 参传 nullptr**，只注册名字） |
| `System_tables::Types` 有 4 个值 | **7 个**：`INERT / CORE / SECOND / DDSE_PRIVATE / DDSE_PROTECTED / PFS / SYSTEM` |
| `Spatial_reference_systems` 的表名是 `spatial_reference_systems` | 物理表名是 **`st_spatial_reference_systems`**（唯一一个类名≠表名的） |
| "28+5" 张 | **源码里找不到任何依据**（grep `innodb_dictionary` 全树 0 命中）。正确是 30 + 4 = 34，隐藏的 32 |
| InnoDB 有 5 张 DDSE 表 | **精确 4 张**（`innodb_dynamic_metadata` / `innodb_table_stats` / `innodb_index_stats` / `innodb_ddl_log`） |
| InnoDB 的 `SYS_*` 内部表有 8 张 | 8.0 只有 **4 张被装载**（SYS_TABLES/COLUMNS/INDEXES/FIELDS）；`SYS_VIRTUAL` 成员还在但不再 load |
| `Object_cache` 类 | **不存在**（grep 命中的是 PFS 变量名 `key_object_cache_mutex`）。真名 **`Object_registry`**（`sql/dd/cache/object_registry.h:62`） |
| `Dictionary_object` 基类 | **不存在**（只有 `column_statistics.h:34` 一行前向声明）。对象体系根是 **`Weak_object`** |
| `dd_table.cc` 有 `update_table()` | **不存在**。真名族：`create_table()` / `create_dd_user_table()` / `create_dd_system_table()` / `fill_dd_columns_from_create_fields()` / `recreate_table()` / `drop_table()` |
| `sql/dd/impl/cache/dictionary_client.h` | 头文件在 **`sql/dd/cache/dictionary_client.h`**（实现才在 `impl/cache/`） |
| `storage/innobase/dict/dict0dd.h` | 实际是 **`storage/innobase/include/dict0dd.h`** |
| `Object_key` 在 `raw/object_keys.h` | 抽象基类在 **`sql/dd/impl/object_key.h:38`**；子类才在 `raw/object_keys.h` |
| `sql/sql_base.cc` 或 `sql/auth/sql_authorization.cc` 里有 DD 表 DML 拦截 | **都没有**（grep 相关关键词 0 命中）。主闸在 **`sql/sql_parse.cc` 的 `LEX::add_table_to_list()`** |
| 菱形虚继承的"为什么"在 `inherits via dominance` 注释里 | 那 23 处只是**消除编译器告警的转发代码**。设计说明在 4 处 **`@note`**（`weak_object.h:36-39` 等，措辞一致） |
| `acquire_for_modification` 有"必须已注册未提交对象"的断言 | **未找到**。相关断言在 `store()` / `update()`（`assert(!element)`）与 `register_uncommitted_object()` |

### ★ 数量速记

```
dd::tables::* 注册      = 30   (INERT 1 + CORE 22 + SECOND 7)
InnoDB DDSE 表          =  4   (2 张隐藏 PRIVATE + 2 张可见 PROTECTED)
                        ────
DD/DDSE 总数            = 34
mysql.tables hidden='System'  = 32   (= 34 − 2 张可见的统计表)
SYSTEM 类型条目         = 41   (只有名字，DDL 在 SQL 脚本里)
I_S 视图                = 44 注册 / 43 可见（SHOW_STATISTICS 隐藏）
```

### 容易误解的概念

- **DD 表也是 InnoDB 表**：自举靠"硬编码表定义 + 硬编码创建 mysql.ibd + 内存假存储"三步解决；
- **`information_schema` 是视图**：查它可能比想象中慢（底下是 DD 表连接）；
- **自增/统计数据不在 DD 主表**：自增持久化在 `mysql.innodb_dynamic_metadata`、统计在 `innodb_table_stats`/`innodb_index_stats`——后两张是 **DDSE_PROTECTED**，可以对它们做 DML（这就是 `ANALYZE`/手动改统计能生效的原因）；
- **SDI 是副本不是源**：权威永远在 DD 表；
- **`trx->is_dd_trx` 生产构建里根本不存在**。

#### 观察技巧：让 DD 表可见

DD 表默认对 SQL 不可见（受 `skip_dd_table_access_check` 保护）。debug 构建下可临时打开：

```sql
SET SESSION debug='+d,skip_dd_table_access_check';
SELECT name, schema_id, hidden, type FROM mysql.tables
 WHERE schema_id = 1 AND hidden = 'System';
-- 返回 tables / columns / indexes / foreign_keys …
```

⚠️ **仅 debug 构建可用**（release 构建没有 DBUG 开关）。这是理解 DD 组织方式最直接的手段——能看到 `mysql.tables` 里"DD 表自己也是一行"。

### 待补清单

（全部完成，暂无待补）
（全部完成：月报 2022/01 已通读提炼补进 [`table.md`](../table.md)；官方 DD Architecture 博客已通读，关键视角已融入理论基础）

## 参考

- *MySQL 8.0 Reference Manual → Data Dictionary*
- WorkLog: [WL#6394](https://dev.mysql.com/worklog/task/?id=6394)（⚠️ 官方标题是 *Bootstrap code for new DD*，即**新 DD 的自举代码**，不是"事务化数据字典"整体）、[WL#7743](https://dev.mysql.com/worklog/task/?id=7743)（新 DD 的 DDL 相关 SE API 改动）——两篇均已通读并提炼，详见下面两条
- ★★ [WL#7743](https://dev.mysql.com/worklog/task/?id=7743)（*New data dictionary: changes to DDL-related parts of SE API*）已通读并提炼进「第 6 章设计溯源」：原子/crash-safe DDL 的四项前提、SE API 全量加 `dd::Table*` 参数、`HTON_SUPPORTS_ATOMIC_DDL`、DROP TABLES 五类分类、★ DROP DATABASE 的已知原子性漏洞
- ★★ WL#6394 已通读并提炼进「第 5 章设计溯源」（官方六类表分类 / 建 core 表三样东西必须稳定 / 外键单独加 / 重启为何要 synthesize CREATE TABLE / 版本校验四分支 / 升级事务的官方要求 / 崩溃即删 tablespace 重来 / 自举为何夹在 InnoDB 与动态插件之间）
- 源码：`sql/dd/`（`impl/system_registry.cc` / `impl/bootstrap/bootstrapper.cc` / `impl/cache/dictionary_client.cc` / `impl/sdi.cc` / `impl/transaction_impl.h`）
- [MySQL 中的元数据管理 — 谢榕彪（归墨）](https://zhuanlan.zhihu.com/p/623872012)（原文：[内核月报 2023/10](http://mysql.taobao.org/monthly/2023/10/03/)）：★ 最贴近本篇，讲 DD object 树形描述、两级缓存、dict cache 的 LRU/non_LRU
- 内核月报：[详解 Data Dictionary](http://mysql.taobao.org/monthly/2021/08/03/) / [TABLE 信息的生命周期](http://mysql.taobao.org/monthly/2022/01/03/) / [MySQL 表定义缓存](http://mysql.taobao.org/monthly/2015/08/10/)
- 内核月报：[8.0 · DDL 的那些事](http://mysql.taobao.org/monthly/2020/05/01/) / [DDL log 与原子 DDL](http://mysql.taobao.org/monthly/2021/07/01/) / [原子 DDL 的实现过程](http://mysql.taobao.org/monthly/2018/03/01/) 及[续](http://mysql.taobao.org/monthly/2018/07/01/)
- [MySQL 8.0: Data Dictionary Architecture and Design](https://dev.mysql.com/blog-archive/mysql-8-0-data-dictionary-architecture-and-design/)：官方架构说明（★ 已通读。2016-10 发布，早于 8.0 GA 一年半，文中自述"部分架构后续版本才实现"——引用时以本仓库 8.0.39 源码为准。五条设计动机：终结 split brain / 原子 DDL（移除隐式提交）/ 简化替代 .FRM / 统一缓存 API / SDI 保留 .FRM 手工恢复能力）
