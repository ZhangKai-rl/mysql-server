# 8.0 数据字典（DD）

> **定位**：MySQL 8.0 的**权威元数据**——存在哪、长什么样、怎么被缓存、怎么参与自举与原子 DDL。
> **边界**：InnoDB 侧的内存字典（`dict_table_t` 等）见 [`innodb_dict.md`](innodb_dict.md)；表对象生命周期见 [`../table.md`](../table.md)；DDL 执行见 [`../../innodb/ddl.md`](../../innodb/ddl.md)。

## 目录

- [概述](#概述)（★ 全景总览图）
- [理论基础](#理论基础)
  - [设计思想与权衡](#设计思想与权衡)（双轨体系 / 虚继承 / ★ Tables vs Table / 继承多态 / 树形结构 / 为何进 InnoDB / Object_table）
  - [算法与数据结构](#算法与数据结构) / [他库对比与演进动机](#他库对比与演进动机)
- [核心实现](#核心实现)
  - [1. DD 表体系与存储](#1-dd-表体系与存储)（30 张逐张用途 / 全部在 mysql.ibd / 为何不能 DML）
  - [2. 对象模型与 `se_private_data`](#2-对象模型与-se_private_data)（★ options vs se_private_data / ★ 全键名表）
  - [3. DD cache](#3-dd-cache)（Dictionary_client / RAII / 三 registry / Shared_dictionary_cache / COW）
  - [4. SDI](#4-sdiserialized-dictionary-information) / [5. 启动自举](#5-启动自举鸡生蛋问题) / [6. 原子 DDL 中的地位](#6-原子-ddl-中-dd-的地位) / [7. 升级](#7-升级)
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

### 2.2 ★★ `options` vs `se_private_data`

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

### 2.3 `default_value_utf8` / `generation_expression_utf8`

- 写入（`sql/dd/dd_table.cc`）：`set_generation_expression_utf8()` :631、`set_default_value_utf8()` :773；
- 落库/恢复（`column_impl.cc:308-320` / `:221-229`）；
- 消费方：I_S.COLUMNS 视图直接映射这两列（`system_views/columns.cc:51-52, 112-122`）。

★ 注意区分：`default_value_utf8` 是**给 SQL 层/I_S 看的文本**；InnoDB 真正用于 instant ADD COLUMN 的默认值走的是列 `se_private_data` 里的 `default` / `default_null`（`dict0dd.cc:2420-2458`）。

### 2.4 ★★ InnoDB 的 `se_private_data` 全键名表

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

### 4.1 是什么

页类型 `FIL_PAGE_SDI = 17853`（`fil0fil.h:1215`），另有 `FIL_PAGE_SDI_BLOB = 18` / `FIL_PAGE_SDI_ZBLOB = 19`。SDI 根页号存在 **page 0**（`FSP_SDI_HEADER_LEN = 8`）。

### 4.2 什么时候写、哪些表空间有

写入链：`Storage_adapter::store()` 写完 DD 表后立即 `sdi::store()`（`storage_adapter.cc:342-347`）→ `dd::sdi::store()` → `hton->sdi_set()` → InnoDB `dict_sdi_set()`（`dict0sdi.cc:307`，复用当前 DDL 事务）。

★ **不是所有表空间都有**（`dict0sdi.cc:321-369` 的跳过条件）：

- discarded 表空间、尚未分配 `se_private_id` 的表、无 `DD_SPACE_ID` 的表空间 → 跳过；
- ★ **`fsp_is_undo_tablespace() || fsp_is_system_temporary()` → 直接 return false**（undo 与临时表空间**不写 SDI**）。

所以：系统表空间 ibdata1、mysql.ibd、每个 file-per-table 的 .ibd、每个通用表空间都有 SDI 索引。

### 4.3 `ibd2sdi` 怎么读

★ **它不走 `dict_sdi_get` API，而是独立实现的页级解析器**（`utilities/ibd2sdi.cc`）：读 page 0 拿 SDI 根页号 → 遍历 B-tree 校验页类型必须是 `FIL_PAGE_SDI` → 解析记录里的 type/id/uncomp_len/comp_len → 输出 JSON。

### 4.4 SDI 与 DD 的关系

**SDI 是 DD 对象序列化后的冗余副本，不是独立数据源**。权威永远是 mysql.ibd 里的 DD 表。用途是**离线/自描述**：`ibd2sdi` 导出、`.ibd` 单独迁移 / IMPORT、崩溃场景的元数据恢复。二者一致性由"DD 写 + SDI 写在同一个事务里"保证。

---

### 5. 启动自举（鸡生蛋问题）

**问题**：DD 表本身是 InnoDB 表——要先能读 `mysql.tables` 才能知道表定义，但要知道表定义才能打开它。

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

---

### 7. 升级

三种升级路径，`dd::init` 的参数决定走哪条：

| 参数 | 场景 | 流程 |
|---|---|---|
| `DD_INITIALIZE` | `--initialize` 新数据目录 | 只 `create_*`，不读旧版本 |
| `DD_INITIALIZE_SERVER` | **5.7 数据目录就地升级** | 预检 → 建新 DD → 迁移旧元数据 |
| `DD_RESTART_OR_UPGRADE` | 正常重启（含 8.0.x → 8.0.y） | `create_tables` + `upgrade_tables` |

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

## Misc

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
- WorkLog: WL#6394（事务化数据字典）、WL#6391（访问控制矩阵）
- 源码：`sql/dd/`（`impl/system_registry.cc` / `impl/bootstrap/bootstrapper.cc` / `impl/cache/dictionary_client.cc` / `impl/sdi.cc` / `impl/transaction_impl.h`）
- [MySQL 中的元数据管理 — 谢榕彪（归墨）](https://zhuanlan.zhihu.com/p/623872012)（原文：[内核月报 2023/10](http://mysql.taobao.org/monthly/2023/10/03/)）：★ 最贴近本篇，讲 DD object 树形描述、两级缓存、dict cache 的 LRU/non_LRU
- 内核月报：[详解 Data Dictionary](http://mysql.taobao.org/monthly/2021/08/03/) / [TABLE 信息的生命周期](http://mysql.taobao.org/monthly/2022/01/03/) / [MySQL 表定义缓存](http://mysql.taobao.org/monthly/2015/08/10/)
- 内核月报：[8.0 · DDL 的那些事](http://mysql.taobao.org/monthly/2020/05/01/) / [DDL log 与原子 DDL](http://mysql.taobao.org/monthly/2021/07/01/) / [原子 DDL 的实现过程](http://mysql.taobao.org/monthly/2018/03/01/) 及[续](http://mysql.taobao.org/monthly/2018/07/01/)
- [MySQL 8.0: Data Dictionary Architecture and Design](https://dev.mysql.com/blog-archive/mysql-8-0-data-dictionary-architecture-and-design/)：官方架构说明（★ 已通读。2016-10 发布，早于 8.0 GA 一年半，文中自述"部分架构后续版本才实现"——引用时以本仓库 8.0.39 源码为准。五条设计动机：终结 split brain / 原子 DDL（移除隐式提交）/ 简化替代 .FRM / 统一缓存 API / SDI 保留 .FRM 手工恢复能力）
