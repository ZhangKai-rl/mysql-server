# MySQL 行数据表示：从 Field 到 dtuple 到 REC（server ↔ InnoDB 跨层）

> 基于 MySQL 8.0.39 源码。讲"**一行数据在各层是怎么被表示的**"这条完整链路：server 层的 `Field` / `TABLE::record` → 跨层边界的 handler 接口 → InnoDB 内存层的 `dfield_t` / `dtuple_t` → InnoDB 物理层的 `rec_t`。核心是**三种"行"的对偶关系**：server 行缓冲、引擎内存元组、磁盘记录字节流。
>
> **边界**：本篇讲"行的表示结构与层间转换"这个跨层机制；`rec_t` 在页内的**物理布局**（记录头、字段偏移、NULL 位图）见 [`../../innodb/physical/record.md`](../../innodb/physical/record.md)；**读路径的流程语义与读方向函数的逐行剖析**（`row_search_mvcc` 状态机、`row_sel_*`/`row_mysql_convert_row_to_innobase`）见 [`../../innodb/row_search.md`](../../innodb/row_search.md)——本篇只保留其中的"表示"视角（字节怎么对称、类型怎么编码）；字节级序列化原语（`mach_*`）见 [`encoding.md`](../../server/infra/encoding.md)；字符集与排序规则语义见 [`charset.md`](../../server/infra/charset.md)；链表容器 `UT_LIST_NODE_T` 见 [`list.md`](list.md)。

## 目录

- [概述](#概述)
- [理论基础](#理论基础)
- [核心实现](#核心实现)
- [三种行表示的对照](#三种行表示的对照)
- [★ 本机制里的工程实现技法](#-本机制里的工程实现技法)
- [Misc](#misc)
- [参考](#参考)

---

## 概述

### 是什么

MySQL 的"一行数据"在内存里**不是只有一种表示**。从 SQL 执行到落盘，同一行要经历至少三种不同的表示形态：

| 层次 | 表示 | 谁在用 |
|---|---|---|
| server 逻辑层 | `Item`（表达式值）、`Field`（列） | 优化器、执行器 |
| server 行缓冲 | `TABLE::record[N]`（字节缓冲 + `Field` 解释） | 执行器、handler 接口 |
| 引擎内存层 | `dtuple_t` / `dfield_t` | InnoDB 插入、检索、比较 |
| 引擎物理层 | `rec_t`（字节流 + offsets） | 页内存储、B+树 |

本篇就是把这条链路上**每一种表示的结构、职责与转换**讲清楚。

### 用途

为什么需要这么多种表示？一句话：**每一层的约束不同**——server 要的是"能被 SQL 语义解释的类型化字段"，引擎内存要的是"能被索引比较与排序的紧凑元组"，磁盘要的是"能被页结构定位、能前缀压缩的字节流"。三个目标互相冲突，于是用三层表示 + 两次转换来解耦。

#### 本篇盘点的范围（与之外）

本篇讲的是**"普通行"的通用表示机制**。以下几种特殊的数据表示**各有专门文档，本篇不重复**：

| 表示 | 归属文档 |
|---|---|
| JSON 的内部表示（`Field_json` / `Json_wrapper` / JSONB 二进制格式） | [`../../server/datatype/json.md`](../../server/datatype/json.md) |
| GIS 空间数据（`Field_geom` / WKB 格式） | [`../../server/datatype/gis.md`](../../server/datatype/gis.md) |
| 外部大对象 LOB 的页外存储结构 | [`../../innodb/physical/lob.md`](../../innodb/physical/lob.md) |
| 虚拟列（生成列）的语义与实现 | [`../../feat/generated_columns.md`](../../feat/generated_columns.md) |
| binlog 里的行表示（`Rows_log_event` 的行格式，与前三种都不同） | [`../../server/replication/binlog.md`](../../server/replication/binlog.md) |
| **undo 记录里的旧版本行**（MVCC 读旧版本用的"行表示"，格式与 `rec_t` 同族但独立） | [`../../innodb/undo_log.md`](../../innodb/undo_log.md) |
| 内部临时表的行（`Temp_table_param` 等 server 内部的行格式） | [`../../server/query/runtime/07_temptable.md`](../../server/query/runtime/07_temptable.md) |

它们**都建立在本文的结构之上**（例如 JSON 值最终也是存在 `TABLE::record` 里、由 `Field_json` 解释），但各自的格式细节归专属文档。

### 版本演进

| 版本 | WL | 变化 |
|---|---|---|
| 5.0.3 | — | true VARCHAR（`DATA_MYSQL_TRUE_VARCHAR`）：行里带 1-2 字节长度前缀，VARCHAR 不再是定长。⚠️ 该变更**早于 MySQL Worklog 体系**，无 WL 编号 |
| 5.7 | [WL#411](https://dev.mysql.com/worklog/task/?id=411)（server 侧生成列）／[WL#8114](https://dev.mysql.com/worklog/task/?id=8114)（InnoDB 侧真正跳过虚拟列存储） | `dtuple_t` 支持虚拟列（`n_v_fields` / `v_fields` 独立数组） |
| 8.0.0 | —（`lob.md` 亦未记编号） | LOB 外部存储重写：first page / data page / node page / index entry（详见 [`lob.md`](../../innodb/physical/lob.md)） |
| 8.0.12 | [WL#11328](https://dev.mysql.com/worklog/task/?id=11328) | LOB 小改动（≤100 字节）用 undo 记 Lob diff 并原地改页，避免 COW 新页 |
| 8.0.12 | [WL#11250](https://dev.mysql.com/worklog/task/?id=11250)（server）／[WL#11491](https://dev.mysql.com/worklog/task/?id=11491)（InnoDB：新列默认值写进 redo/undo） | instant ADD COLUMN：同一张表的记录可能有不同列数，offsets 计算要能识别 |
| 8.0.21 | [WL#8763](https://dev.mysql.com/worklog/task/?id=8763) | 多值索引引入 `multi_value_data`（typed-array 隐藏虚拟列 + 多值索引） |

---

## 理论基础

### 设计思想与权衡

**★ 三种表示"必须并存"的量化对照**（这是本篇的主轴，先立靶子再看细节）：

| | server `record[0]` | InnoDB `dtuple_t`/`dfield_t` | InnoDB `rec_t` |
|---|---|---|---|
| 长度 | **定长**（按表定义算满） | 结构化，长度随字段数 | **变长**（按实际数据） |
| 字段定位 | `Field::ptr` **固定偏移** O(1) | `dtuple_get_nth_field()` 数组下标 O(1) | **offsets 数组**现算 |
| NULL 表示 | 行首 **NULL 位图**（1 bit/列） | `dfield_t::len == UNIV_SQL_NULL` **哨兵** | offsets 的 `REC_OFFS_SQL_NULL` 位 |
| 类型信息 | `Field` 子类（每种类型一个类） | `dfield_t::type`（`dtype_t` 位域） | **不存**（靠 `dict_index_t` 元数据） |
| 字节序 | 小端（`db_low_byte_first`） | 无（裸指针） | **大端 + 符号位翻转**（可 memcmp） |

> ★★ 三种表示**都在解决同一个问题**："给定一行，怎么拿到第 i 列的值"。答案分别是：**编译期偏移** / **数组下标** / **运行时解析**。三者的取舍是"空间 ↔ 解析成本 ↔ 表达力"的三角。

**★ 贯穿全篇的一个技巧：用长度值做 NULL 哨兵**（三种表示里两种都这么干）：

```cpp
constexpr uint32_t UINT32_UNDEFINED = ~0U;                              // 0xFFFFFFFF
constexpr uint32_t UNIV_SQL_NULL = UINT32_UNDEFINED;                     // SQL NULL
constexpr auto UNIV_SQL_ADD_COL_DEFAULT = UNIV_SQL_NULL - 1;             // 0xFFFFFFFE
constexpr auto UNIV_NO_INDEX_VALUE = UNIV_SQL_ADD_COL_DEFAULT - 1;       // 0xFFFFFFFD
const uint32_t UNIV_MULTI_VALUE_ARRAY_MARKER = UNIV_NO_INDEX_VALUE - 1;  // 0xFFFFFFFC
constexpr auto UNIV_SQL_INSTANT_DROP_COL = UNIV_MULTI_VALUE_ARRAY_MARKER - 1;  // 0xFFFFFFFB
constexpr uint32_t UNIV_EXTERN_STORAGE_FIELD = UNIV_SQL_NULL - UNIV_PAGE_SIZE_DEF;
```

```cpp
static inline ulint dfield_is_null(const dfield_t *field) {
  return (field->len == UNIV_SQL_NULL);   // ★★ NULL 完全由 len 哨兵表示，不依赖 data 指针
}
static inline void dfield_set_null(dfield_t *field) {
  dfield_set_data(field, nullptr, UNIV_SQL_NULL);   // data=nullptr + len=0xFFFFFFFF
}
```

| 哨兵 | 值 | 语义 |
|---|---|---|
| `UNIV_SQL_NULL` | `0xFFFFFFFF` | SQL NULL |
| `UNIV_SQL_ADD_COL_DEFAULT` | `0xFFFFFFFE` | INSTANT ADD 列的默认值 |
| `UNIV_NO_INDEX_VALUE` | `0xFFFFFFFD` | 多值索引：JSON 里无此属性 |
| `UNIV_MULTI_VALUE_ARRAY_MARKER` | `0xFFFFFFFC` | `data` 指向 `multi_value_data*`（数组） |
| `UNIV_SQL_INSTANT_DROP_COL` | `0xFFFFFFFB` | INSTANT DROP 的列 |
| `UNIV_EXTERN_STORAGE_FIELD` | `NULL - PAGE_SIZE` | 外存字段长度阈值（非哨兵，是"高位区间"） |

> ⚠️ **只有 `dfield_is_null()` 检查 `== UNIV_SQL_NULL`**，所以后四个**都不是 NULL**——`dfield_dup()` 会把它们当"有数据"处理。这正是 `dfield_dup()` 开头那条 `ut_ad(!dfield_is_multi_value(field) || dfield_get_len(field) != UNIV_MULTI_VALUE_ARRAY_MARKER)` 存在的原因（多值数组必须用 `dfield_multi_value_dup()`）。
> ★ 类型侧还有两个哨兵：`DATA_MISSING = 0`（虚列未计算）、`DATA_ERROR = 111`（debug 下毒化未初始化字段）。

**权衡一：为什么不直接用一种表示贯穿全流程？——两个被否决的方案**

**被否决方案 A："server 直接把 `Field` 数组交给引擎"**。否决理由（**按根本程度排序**）：

1. ★ **插件式架构不允许（最根本）**：MySQL 的存储引擎是**可插拔**的（`handlerton` 本身就是 plugin，`sql/handler.h` 直接 include `sql_plugin_ref.h`）。这条约束是**双向**的——server 不能假设引擎内部如何表示数据，引擎也不能依赖 server 的内部结构。而 `Field` 是 server 层的核心类（`sql/field.h`，68 个子类 + 虚函数表），引擎一旦理解它就：
   - 从"可独立编译的模块"变成 server 的附属（**InnoDB 早年正是独立产品**，由 Innobase Oy 开发、可单独编译）
   - server 改 `Field` 的定义 → 所有引擎重编（ABI 级耦合）
   - **多引擎并存不再可能**：MyISAM / InnoDB / Memory 就被迫共用一种行表示

   ⇒ **代码级证据**——插件边界就写在 `handler` 基类的签名里，接口传的全是**字节**，一个 `Field` 都没有：

   ```cpp
   virtual int write_row(uchar *buf);                          // sql/handler.h:6499
   virtual int update_row(const uchar *old_data, uchar *new_data);
   virtual int rnd_next(uchar *buf);
   ```

   ⇒ 这条比"重对象"更根本：**重对象是代价问题，插件边界是架构可行性问题**。它也是本篇反复出现的那句约束的根源——"**引擎看不到 `Field`、看不到 `TABLE`，只看到一段字节**"（见「主链路」与「二、跨层边界」）。

2. **`Field` 是重对象**（虚函数、`ptr` 指向 `TABLE::record`、null 位信息），引擎用它就背上 C++ 对象生命周期负担；且页上只能放字节，重对象**无法存进页**

3. **引擎要为同一行生成多条顺序不同的索引记录**——server 行按表定义顺序排，而二级索引记录按索引列顺序排（还可能加前缀、虚拟列、隐藏系统列 `DB_TRX_ID`/`DB_ROLL_PTR`）。同一份"行"要在不同索引里以不同顺序、不同列集出现，"直接引用 server 行"意味着每次都要在 `Field` 数组上做重排——等于把序列化成本摊进每次索引操作

⇒ 三条的关系：**① 说"不能有"（架构），② 说"代价大"（效率），③ 说"即便能做也不该这么做"（语义）**。

**被否决方案 B："引擎直接读 `TABLE::record` 的字节"**。否决理由：`TABLE::record` 的布局是 server 定的（`pack_length()` 之和 + NULL 位图），引擎若直接认这个布局，则：

- server 改行布局（如加隐藏列、改长度前缀规则）就必须改所有引擎
- ★ **引擎要完全控制记录在页内的物理布局**：InnoDB 页内是**堆 + 链表**组织——记录在 `PAGE_HEAP` 区按插入顺序乱放，靠记录头里的 `next` 指针串成逻辑顺序；删除后空间进 `PAGE_FREE` 链表复用（详见 [`page_structure.md`](../../innodb/physical/page_structure.md)）。这种"物理乱序 + 逻辑有序 + 空间回收复用"要求引擎对字节流有完全控制权，外来缓冲做不到
- **引擎要能做自己的紧凑编码**：变长字段只存实际长度（CHAR 不补空格）、NULL 用位图表示（不占数据区）、`ROW_FORMAT=COMPRESSED` 还要对整页做 zlib
- 索引比较要求的是"**按字典序可直接 memcmp 的字段排列**"（有符号数做 Offset Binary 变换等），server 行布局不满足这个约束

**于是有了中间层**：一层"**既能被索引比较、又能被序列化成字节流**"的表示——`dtuple_t`。`dfield_t` 只持 `(data 指针, len, dtype)` 三件套：`data` 是零拷贝指针（不搬数据）、`len` 是长度（`UNIV_SQL_NULL` 作 NULL 哨兵）、`dtype` 是内嵌的类型描述。它既不是重对象、也不是死布局，而是"可以按需重排的字段数组"——同一行针对不同索引构造不同顺序的 dtuple，正是 `rec_convert_dtuple_to_rec(buf, index, dtuple)` 里 `index` 参数存在的原因。

**这个权衡的完整代价**（连着失效场景一起看）：三层表示意味着两次转换，每次 INSERT/UPDATE/SELECT 都付出"搬字节"的成本。列字符集越复杂（多字节变长）、列越多，代价越高；`utf8mb4` 的 CHAR 列还要额外做尾空格剥离（写）与回填（读）。而省掉这成本的唯一办法是模板（见转换桥梁节）——不是不转，而是把转换的**决策信息预先算好**。

**权衡二：`dtype_t` 为什么要自己编码类型而不复用 server 的 `Field` 类型？**

`dtype_t` 把"MySQL type code + charset collation + nullability + 是否 true VARCHAR + 是否虚拟列"**打包进一个 32 位 prtype**，再加上 8 位 `mtype` 与 5 位 `mbminmaxlen`。理由：引擎只需要"够用来比较与存储"的最小类型信息，不需要完整的 `Field` 语义；打包成位域后，`dtype_t` 只有 8 字节，可以**内嵌进 `dfield_t`**（而不是指针引用），访问无需间接跳转。

代价是"类型表达能力受限"——`prtype` 的 collation id 只有 **15 位**（上限 32767），且类型判定散落在一堆 `DATA_*` 宏里，可读性差。

**权衡三：三种表示带来的转换成本**

每次插入/检索都要做两次转换（server 行 ↔ dtuple ↔ REC），这是纯粹的性能开销。InnoDB 的优化是：
- 用 `mysql_row_templ_t`（`ha_innodb.cc`）缓存列的字符集、`mbminlen`/`mbmaxlen`、长度前缀字节数，避免每个字段都查 `CHARSET_INFO`
- 检索时按模板直接把 REC 的字段拷进 MySQL 行缓冲，跳过构造 dtuple 的中间步骤（快速路径）

**失效场景**：列的字符集越复杂（多字节、变长），转换越贵；`utf8mb4` 的 CHAR 列还要额外做尾空格剥离/回填。

### 理论溯源

- **行式存储的"内存元组 vs 磁盘记录"二分**是 System R / Postgres 一脉的经典设计（Oracle/PostgreSQL 同样有 "tuple" 与 "page item" 两层）。MySQL 的特殊之处在于它是 **插件式存储引擎**架构：server 层行格式（MySQL format）与引擎行格式必须解耦，所以比单体数据库**多一层**（server 缓冲 ↔ 引擎元组 ↔ 磁盘记录，共三层）。
- **类型打包进位域**的做法源自 InnoDB 早期的内存紧张约束（5.x 时代每个 `dict_col_t` 都要省字节）。

### 他库对比

| | server 行 → 引擎行 → 磁盘行 |
|---|---|
| MySQL + InnoDB | `TABLE::record` → `dtuple_t` → `rec_t`（三层，因插件式引擎架构必须解耦） |
| PostgreSQL | `TupleTableSlot` → `HeapTuple`（两层，无独立内存元组层） |
| Oracle | 直接在 block 上操作，无独立内存元组层 |

**⚠️ 精确化一下**：说 PG"不支持多存储引擎"**不准确**。PG 有两套扩展点，只是性质与 MySQL 不同：

| PG 的扩展点 | 引入 | 性质 |
|---|---|---|
| **FDW**（Foreign Data Wrapper） | 9.1（2011） | 接**外部数据源**（远端 PG、文件、其他库…），是"外部表"而非本地存储引擎 |
| **Table Access Method（TAM）** | **12（2019）** | 真正的**本地可插拔表存储**（`CREATE TABLE ... USING xxx`），默认 `heap`；生态里有 zheap、columnar 等 |

⇒ 所以 PG **现在确实支持**多存储引擎。真正的差别不在"能不能换"，而在**历史包袱与接口假设**：

1. **MySQL 从第一天起就是多引擎并存**（ISAM / MyISAM / InnoDB / Memory…），且各引擎**语义差异极大**——MyISAM 无事务、无 MVCC、只有表锁；InnoDB 全都有。⇒ server 层**不能对引擎做任何假设**，行格式必须彻底解耦
2. **PG 长期只有一种 heap**，TAM 直到 2019 年才加入，且生态中默认仍只有 heap ⇒ 从未面临"同时容纳语义完全不同的引擎"这个压力，两层就够

**★ 更本质的判据**：**插件式的关键不是"能不能换引擎"，而是"server 敢假设多少"**。

- **MySQL 的 `handler` 接口刻意做得很薄**——只传字节 + 少量 `HA_*` 能力标志（见「二、跨层边界」的 `write_row(uchar*)`）。因为它必须能容纳 MyISAM 这种"几乎什么都不支持"的引擎
- **PG 的 TAM 接口厚得多**，可以假设存储具备事务、MVCC、TOAST 等能力

⇒ 这正是 MySQL 需要**第三种表示**（`dtuple_t`）的根源：它是**引擎内部**的通用中间层，让 InnoDB 能在"自己认识的类型体系"里做比较与排序，而不是被迫直接解释 server 的字节。PG 没有这一层——它的 `HeapTuple` 基本就是**磁盘格式的内存副本**（`HeapTupleHeader` 与磁盘头同构），直接在上面 `heap_getattr` 取字段即可。

---

## 核心实现

### 主链路

一条 `INSERT` 从 SQL 到落盘的完整链路：

```
SQL 表达式求值 → Item（表达式值）
  → Field::store(...) 写进 TABLE->record[0]  ← server 行缓冲（字节 + Field 解释）
  → handler::ha_write_row(record[0])          ← 跨层边界，只传 uchar*
  → InnoDB 逐列 row_mysql_store_col_in_innobase_format(...)
      → 构造 dfield_t（data 指针 + len + dtype_t）并填入 dtuple_t
  → rec_convert_dtuple_to_rec(buf, index, dtuple)   ← 序列化
  → rec_t（字节流 + offsets）写入页
```

检索是反向：

```
页内 rec_t → row_sel_store_mysql_rec / 模板拷贝
  → 直接或经 dtuple 转回 MySQL 格式 → 填 TABLE->record[0]
  → Field::val_str()/val_int() 读出类型化值
```

### 一、server 层：Item、Field 与行缓冲

server 侧的数据表示有**三层递降**：`Item`（表达式值）→ `Field`（列）→ `TABLE::record`（字节缓冲）。前两层是"类型化的值"，第三层是无类型的字节。

#### Item：表达式的值表示

`Item`（`sql/item.h:882`）是 server 里**所有表达式值的载体**——字面量、列引用、函数调用、子查询结果的返回值都是 `Item`。它比 `Field` 更"上游"：

```cpp
class Item : public Parse_tree_node {
 public:
  enum Type { STRING_ITEM, INT_ITEM, REAL_ITEM, DECIMAL_ITEM, ... };  // 值的"形态"
  ...
  bool null_value;            // 本次求值结果是否为 NULL
  bool unsigned_flag;         // 整数是否无符号
  DTCollation collation;      // ★ 字符串结果的字符集与排序规则
  String str_value;           // ★ 字符串结果的缓存（避免重复分配）
  ...
  virtual enum_field_types data_type() const = 0;   // 转成 MySQL 数据类型
  virtual bool is_null() ...
  virtual String *val_str(String*) / longlong val_int() / double val_real() ...
};
```

`Item::Type` 枚举标的是**这个值在语法/语义上是什么**（不是存储类型）：`FIELD_ITEM`（列引用）、`STRING_ITEM` / `INT_ITEM` / `REAL_ITEM`（字面量按形态分）、`FUNC_ITEM` / `SUM_FUNC_ITEM`（函数与聚合）、`COND_ITEM`（条件）、`REF_ITEM`（引用）、`NULL_ITEM`、`DEFAULT_VALUE_ITEM`…… 这套枚举服务于"表达式是什么"，而具体的 SQL 数据类型由 `data_type()` 给出——**两者是不同维度**（一个是语法分类，一个是类型系统分类）。

要点：

1. **`val_str()` / `val_int()` / `val_real()` 是"拉模式"的取值接口**——`Item` 自己不缓存所有形态的值，谁要什么形态就调对应的 `val_*`（字符串结果缓存在 `str_value` 里）。
2. **`collation` 是 `Item` 自带的**：字符串 `Item` 的结果带字符集与排序规则，表达式的 collation 推导（`DTCollation::aggregate`）就发生在这里——详见 [`charset.md`](../../server/infra/charset.md)。
3. **`null_value` 是求值状态而非类型属性**：同一个 `Item` 每次求值都可能不同。

**`Item` 与 `Field` 的关系**：`Item_field` 是"引用某一列"的 `Item`，它的 `val_*` 最终转发到 `Field::val_*`——即从"表达式值"落到"列值"的桥梁。

> **边界**：`Item` 的完整剖析（类继承树、求值流程、`fixed`/`cleanup` 生命周期契约）属于表达式主题，见 [`../../server/query/13_item_expression.md`](../../server/query/13_item_expression.md)。本篇只讲"它作为行表示链最上游的值载体"。

#### Field 类族

> ★ **完整剖析已独立成篇：[`../../server/field.md`](../../server/field.md)**——`Field` 的类型系统（38 个子类）、值协议 `store` ↔ `val_*`、存储布局、字符集与排序键、与 Item/TABLE/handler/DD 四条接缝、特殊子类、默认值/instant/虚拟列，全在那篇。
>
> 本节只保留与"**server 行格式 ↔ InnoDB 记录格式**"直接相关的要点，并纠正四处流传的错误说法。

`Field`（`sql/field.h`）是 server 对"一列"的抽象——它只是"解释" `TABLE::record` 里某段字节的**视图，自己不持有数据**：

| | 挂在哪 | 份数 |
|---|---|---|
| `Field`（模板） | `TABLE_SHARE::field[]` | 一份 / 表定义 |
| `Field`（实例） | `TABLE::field[]`，由 `clone()` 得来 | **一份 / TABLE**（各自 `record[0]` 不同） |
| `record[N]` | `TABLE` | 每个实例一份，**反复覆盖** |

⇒ 100 万行的表仍然只有几十个 `Field` 实例——**`Field` 的数量只跟列数有关，不随行数增长**。"当前行"是**隐式的**（由 `record[0]` 承载），`Field` 自己不知道现在读到第几行。

**★ 四处需要纠正的流传说法**：

| 流传说法 | 事实 |
|---|---|
| "**68 个子类**" | ★ **38 个**。68 是把 **31 个前向声明**（`class Field_blob;` 之类）也数进去了——`^class Field_*` 的**定义**只有 37 个，加 `Create_field_wrapper` 这个"伪 Field" = 38 |
| `Field_generated` | ★ 8.0.39 **不存在**这个类（生成列用 `Field::gcol_info` + `stored_in_db`，不是独立子类） |
| `Field_date` | ★ 实为 `Field_newdate`（`sql/field.h:3122`） |
| "`Field` 数组挂在表定义上，**一张表只创建一次**" | ★ 模板在 SHARE 是一份，但**每个 `TABLE` 都 `clone()` 一份自己的**（`sql/table.cc:2982` `share->field[i]->clone(root)`）——否则各实例的 `record[0]` 就没法各自绑定 |

**与本篇（数据表示）直接相关的三套字节表示**：

| 接口 | 输入 → 输出 | 校验 | 消费者 |
|---|---|---|---|
| `store()` | SQL **值** → `record[0]` 内存记录格式 | 有（`type_conversion_status`） | 引擎写行 |
| `val_*()` / `send_to_protocol()` | `record[0]` → 值 / 客户端协议 | — | 表达式 / 结果集 |
| ★ `pack()` | **`record[0]` 里的字节** → 自描述紧凑流 | **无** | filesort addon、binlog row image |

★ **`pack()` 常被误当成"store 的另一种写法"**——它接收的是 `record[0]` 里已有的字节（不是 SQL 值）、**不做任何 SQL 语义检查**；谁重写它的判据是"它的 record 格式是不是磁盘格式"（新式时间类型不需要、老式需要，`pack` 与否见那篇 3.9）。⇒ 不要把 `store` 与 `pack` 混为一谈。

**数值族的编码细节**（`Field_num`）：它自己**从不重写 `pack_length()`**（用基类的 `field_length`——各子类在构造时把 `field_length` 设成了真实字节数），并提供统一的整数读写助手 `pack_int24/32/64`、`unpack_int16/24/32/64`（按 `db_low_byte_first` 决定是否倒序）。★ 三种 store 风格值得注意——**TINY / SHORT / LONG 用 `get_int` 做上下界钳制，LONGLONG 直接 `strntoull10rnd` 不做钳制**（整数上界即 ullong 自身），这是历史分层。

★ **strict 模式的升级不在 `store` 里**：`store` 一律只返回 `TYPE_WARN_OUT_OF_RANGE`，真正升级为错误由 4 参重载完成——实现只是"临时替换 THD 上的检查开关"：

```cpp
// sql/field.cc —— "4 参重载"的全部实现
type_conversion_status Field::store(const char *to, size_t length,
                                    const CHARSET_INFO *cs,
                                    enum_check_fields check_level) {
  THD *thd = current_thd;
  enum_check_fields old_check_level = thd->check_for_truncated_fields;
  thd->check_for_truncated_fields = check_level;   // ★ 换级别
  const type_conversion_status res = store(to, length, cs);   // 同一个 3 参实现
  thd->check_for_truncated_fields = old_check_level;          // ★ 还原
  return res;
}
```

即**所有 `Field_*::store` 都只返回 warning 状态、不判 strict**——strict 与否由"调用方选择的 `check_level`"决定。

`field_length` 与字符集直接相关：`varchar(12)` 的 `field_length = 12 * mbmaxlen`——server 在定义列时就把"字符数"折算成"字节数"，这与 InnoDB 的 `mbminmaxlen` 是同一件事的两个侧面。

#### TABLE::record 行缓冲

```cpp
uchar *record[2]{nullptr, nullptr}; /* Pointer to records. mysql format的行数据 */
```

**两个缓冲**：`record[0]` 是当前行，`record[1]` 通常作为临时/比较用的第二行（如 UPDATE 前镜像、JOIN 时的另一行）。它们是**裸字节缓冲**，长度 = 所有列的 `pack_length()` 之和 + NULL 位图等开销。**没有任何类型信息**——类型信息在 `Field` 数组里。

**行缓冲的尺寸从哪来**：`TABLE_SHARE`（表的元数据，所有 `TABLE` 实例共享）里记着 `rec_buff_length`（`record[]` 缓冲的大小）、字段数 `fields`、字段数组 `field[]`、NULL 位图字节数等。`TABLE` 打开时按 `TABLE_SHARE` 的描述分配 `record[0]`/`record[1]` 两块 `rec_buff_length` 大小的缓冲，再把每个 `Field::ptr` 指向缓冲里的对应偏移。

**分配的实际代码**（`sql/table.cc:2942-2954`）：

```cpp
record = root->ArrayAlloc<uchar>(share->rec_buff_length * records +
                                 share->null_bytes);

if (records == 0) {
  /* We are probably in hard repair, and the buffers should not be used */
  outparam->record[0] = outparam->record[1] = share->default_values;
  has_default_values = true;
} else {
  outparam->record[0] = record;
  if (records > 1)
    outparam->record[1] = record + share->rec_buff_length;   // ★ 后一半
  else
    outparam->record[1] = outparam->record[0];               // Safety
}
outparam->null_flags_saved = record + (records * share->rec_buff_length);
```

三个要点：

1. **一次分配**：`records`（通常 2）份 `rec_buff_length`，外加 `null_bytes` ⇒ `record[0]` 与 `record[1]` 是**同一块连续内存的前后两半**
2. ★ **这正是 `move_field_offset()` 能用**一个 `ptrdiff` **在两份之间切换的原因**——偏移量恒为 `rec_buff_length`，不随表而变
3. ⚠️ `share->default_values` **不是**常规路径的"初值行"——它只在 `records == 0` 时启用，而源码注释明说 *"We are probably in hard repair, and the buffers should not be used"*。它是**修复模式下的兜底**，让 `Field::ptr` 有处可指而已

**★ `record[0]` 是怎么被填进去的**（完整链路）：

```
Item::val_*()                      求值 → 拿到类型化的值
   ↓
Item::save_in_field(Field*)        把值写进某一列（虚函数，各 Item 子类实现）
   ↓                               批量时由 fill_record() 遍历（sql/sql_base.cc:9505）
Field::store(...)                  按该列的存储语义编码成字节（68 个子类各有实现）
   ↓
写进 Field::ptr 指向的偏移          即 record[0] 内该列的位置
   ↓
若值为 NULL：不写数据区，只置 record[0] 的 NULL 位图对应 bit
```

⇒ 所以"构造一行"在 server 侧**没有整体序列化这一步**——是**逐列就地填充**：对每列求值 → 按列类型编码 → 写进它固定的偏移。`record[0]` 从头到尾就是一块被反复覆盖的字节缓冲，这与前面说的"`Field` 只描述偏移、数据始终只在 `record` 里"是同一件事的两面。

**★ `Field::store()` 到底做了什么**（以 `Field_long::store` 为例，`sql/field.cc`）：

> **先说清输入输出与它做了什么**：
> - **输入**：`from` + `len`——一段**不以 `\0` 结尾**的字符串（连同 `cs` 它的字符集）
> - **输出**：① 把结果**写进 `Field::ptr`**（即 `record[0]` 内这一列的偏移）；② 返回 `type_conversion_status` 报告转换情况（`TYPE_OK` / `TYPE_WARN_*` / `TYPE_ERR_*`）
> - **做了什么**：解析成整数 → 范围检查并按需截断 → 按字节序写进 `ptr`。**不分配内存、不碰其它列**——它只负责这一列
>
> 注意这是 `store()` 的**字符串重载**。它还有 `store(longlong, bool)` / `store(double)` 等重载，语义相同（输入换成数值，省掉解析一步）。68 个子类各有一套。

```cpp
type_conversion_status Field_long::store(const char *from, size_t len,
                                         const CHARSET_INFO *cs) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;                 // DEBUG：确认该列已被标记为写
  long store_tmp;
  longlong rnd;

  // ★ 解析字符串与范围检查合成一步；上下界按 unsigned 与否取不同的一套
  const type_conversion_status error =
      get_int(cs, from, len, &rnd, UINT_MAX32, INT_MIN32, INT_MAX32);

  store_tmp = is_unsigned() ? (long)(ulonglong)rnd : (long)rnd;

  // ★ 直接写进 Field::ptr —— 即 record[0] 内这一列的偏移，没有中间缓冲
  if (table->s->db_low_byte_first)
    int4store(ptr, store_tmp);                    // 小端
  else
    longstore(ptr, store_tmp);                    // 大端（早期跨平台遗留）
  return error;                                   // TYPE_OK / TYPE_WARN_* / TYPE_ERR_*
}
```

四个要点：

1. **`get_int()` 把"解析"与"范围检查"合成一次调用**——越界**不是立即报错**，而是返回 `TYPE_WARN_OUT_OF_RANGE` 并**截断**；只有严格 `sql_mode` 才升级成错误。这就是"非严格模式下 `INSERT 超范围值` 只 warn 且存成边界值"的出处
2. **`is_unsigned()` 决定两套上下界**——同样是 4 字节，signed 与 unsigned 的取值范围不同，编码分支也不同
3. ★ **写入位置就是 `Field::ptr`**，没有中间缓冲——呼应前面"`Field::ptr` 是游标而非拥有者"
4. **`db_low_byte_first` 的字节序分支**：MySQL 保留了大小端两套写法（现代平台恒为小端），是早期跨平台遗留

⇒ 68 个子类各有自己的 `store()`，但**骨架一致**：解析 → 检查/截断 → 按字节序写进 `ptr` → 返回转换状态。

**★ 但 `store()` 也只是调度——真正干活的是 `get_int()`**（`Field_num::get_int`，`sql/field.cc:1589`）。

> **先说清这个函数整体在做什么**：把一段字符串**按字符集规则解析成整数** → 做范围检查/截断 → 报告转换状态。它是 `Field_*::store(const char*)` 家族共用的"**文本 → 数值前端**"，**自己不写字节**（写字节是 `int4store(ptr, ...)` 那一行）。

```cpp
type_conversion_status Field_num::get_int(const CHARSET_INFO *cs,
                                          const char *from, size_t len,
                                          longlong *rnd, ulonglong unsigned_max,
                                          longlong signed_min, longlong signed_max) {
  const char *end;
  int error;

  // ★ 走字符集的 cset 函数指针，不是裸 atoi：不同字符集的数字字符不同（如阿拉伯-印度数字）
  *rnd = (longlong)cs->cset->strntoull10rnd(cs, from, len, is_unsigned(), &end, &error);

  if (is_unsigned()) {
    // ★ 隐蔽写法：&& 右侧是赋值——超界时"顺手"把值改成上界
    if ((((ulonglong)*rnd > unsigned_max) && (*rnd = (longlong)unsigned_max)) ||
        error == MY_ERRNO_ERANGE)
      goto out_of_range;
  } else {
    if (*rnd < signed_min)      { *rnd = signed_min; goto out_of_range; }
    else if (*rnd > signed_max) { *rnd = signed_max; goto out_of_range; }
  }

  // ★ 只有开启截断检查时才验证"字符串是否被完整消费"（如 '12abc' → 12）
  if (current_thd->check_for_truncated_fields != 0)
    return check_int(cs, from, len, end, error);

  return TYPE_OK;

out_of_range:
  set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);
  return TYPE_WARN_OUT_OF_RANGE;   // ★ 是 WARN 不是 ERR——严格模式由上层决定
}
```

#### ★ 为什么一定要传 `cs`（与 charset 的关系）

`cs->cset` 是**字符集的函数指针表**（cset = character set handler）。解析整数看似与字符集无关，其实不然：

- **"一个字符占几字节"由字符集定**：下面这个 `_8bit` 版本能直接写 `*str - '0'`，是因为它**假设 1 字节 = 1 字符**；而 `ucs2` / `utf16` 里一个字符占 2 字节，必须换一套实现
- 不同字符集可能有**不同的数字字符**（如阿拉伯-印度数字 `٠١٢٣`）

⇒ 所以 MySQL 把"字符串 → 数字"做成**每个字符集一套**，挂在 `cs->cset` 上，`get_int()` 只是转发。（字符集体系的完整剖析见 [`charset.md`](../../server/infra/charset.md)。）

#### ★ 真正的解析在下一层：`my_strntoull10rnd_8bit`（`strings/ctype-simple.cc:1235`）

> ⚠️ **这一层按归属在 `charset.md`**：它是 `cs->cset` 函数表的成员（"按字符集解析数字"），完整剖析见 [`charset.md`](../../server/infra/charset.md)「数字解析：`strntoull10rnd`」。下面只保留**它在本文链路里的角色**与最关键的几行。

```cpp
/* Skip leading spaces and tabs */
for (; str < end && (*str == ' ' || *str == '\t'); str++) ;
if ((negative = (*str == '-')) || *str == '+') { if (++str == end) goto ret_edom; }

beg = str;
end9 = (str + 9) > end ? end : (str + 9);
/* Accumulate small number into ulong, for performance purposes */
for (ul = 0; str < end9 && (ch = (uchar)(*str - '0')) < 10; str++) {
  ul = ul * 10 + ch;               // ★ 前 9 位用 ulong 累加——快路径
}
if (str >= end) {                  // 无小数点、无指数的小数字
  *endptr = str;
  if (negative) {
    if (unsigned_flag) {
      *error = ul ? MY_ERRNO_ERANGE : 0;   // ★ 负数存进 UNSIGNED 列 → 报范围错
      return 0;
    }
    return (ulonglong)(longlong) - (long)ul;
  }
  ...
```

四个要点：

1. **`ch = (uchar)(*str - '0') < 10`**——判断"是不是数字字符"只用**一次减法 + 一次无符号比较**，比 `'0' <= c && c <= '9'` 少一次比较（经典小技巧）
2. ★ **前 9 位走 `ulong` 快路径**——源码注释直说 *"for performance purposes"*：≤9 位的十进制数必不溢出 32 位，用更窄的类型累加更快，超过才切 `ulonglong`
3. ★ **负数写进 UNSIGNED 列是直接报 `MY_ERRNO_ERANGE`，不是截断成 0**（注意 `ul ? ... : 0`——`-0` 不算错）
4. **`*endptr` 输出"解析停在哪"**——供上层 `check_int()` 判断"字符串有没有被完整消费"（`'12abc'` 停在 `a` ⇒ 报截断警告）

⇒ 函数名里的 **`rnd`** 是 rou**nd**ing：它还处理小数点（源码里的 `dot` 变量），把 `'12.6'` **四舍五入**成 13——这正是 `INSERT INTO t(INT列) VALUES('12.6')` 得到 13 的原因。

#### 回到 `get_int()` 本身

1. ★ **超界是"截断"而非"报错"**：`((ulonglong)*rnd > unsigned_max) && (*rnd = unsigned_max)` 借 `&&` 短路——条件成立时右侧**赋值被执行**，把值改成上界。赋值藏在条件里很隐蔽，但省一次分支
2. **`error == MY_ERRNO_ERANGE`**——解析层自己报的越界（如上一条的负数存 UNSIGNED），与"数值超界"是两条路径
3. **`check_int()` 只在 `check_for_truncated_fields` 开启时调**——`Item::save_in_field_no_warnings()` 会临时关掉它
4. ★ **`out_of_range` 发的是 warning**：返回 `TYPE_WARN_OUT_OF_RANGE` 而**不是 ERR** ⇒ **"非严格模式下插入超范围值只警告、存成边界值"就出自这里**；是否升级为错误由上层看 `sql_mode` 决定

**关于 `ASSERT_COLUMN_MARKED_FOR_WRITE`**（上面 `store()` 第一行那个宏）：DEBUG 构建下检查**该列在 `table->write_set` 位图里已标记为可写**，用来抓"写了不该写的列"；release 下展开为空。它与 `dbug_tmp_use_all_columns()` 是一对——后者**临时把整表列都标为可写**（DBUG 场景），用完 `dbug_tmp_restore_column_map()` 还原，`Item::save_in_field_no_warnings()` 里就是这一对。

#### ★ 为什么"每行大小不同"，`record[0]` 却是定长的

这是"**`Field::ptr` 为什么能用固定偏移**"的前提，也是本篇最关键的一组对照。

```cpp
rec_buff_length = ALIGN_SIZE(share->reclength + 1 + extra_rec_buf_length);   // sql/table.cc:1867
```

`reclength` 是**按表定义算出的行长度**（各列 `pack_length()` 之和 + NULL 位图字节数），**与实际数据无关**——`VARCHAR(100) utf8mb4` 一律按 `100 × 4 = 400` 字节算满，即便实际只存 3 个字符。

⇒ 所以 `record[0]` 是**固定长度的"最大行"**：短行浪费空间，换来的是**每一列在缓冲里的偏移恒定**。★ 若行是变长的，第 2 列的位置就依赖第 1 列的实际长度，`Field` 每次都得重算偏移——那就没有"一份 `Field` 解释所有行"这回事了。

**★★ `reclength` / `null_bytes` 的权威计算点**（⚠️ 真名不是 `alloc_records`，也不是 `mysql_prepare_create_table`；8.0 正常路径是 `prepare_default_value_buffer_and_table_share()` → `find_record_length()`）：

```cpp
static bool find_record_length(const dd::Table &table, size_t min_length,
                               TABLE_SHARE *share) {
  share->fields = 0;
  share->null_fields = 0;
  share->reclength = 0;
  ulong leftover_bits = pack_record ? 0 : 1;   // ★ 第 0 个 bit 是 "pack record" 标志位

  for (const dd::Column *col_obj : table.columns()) {
    if (col_obj->is_se_hidden()) continue;
    if (col_obj->is_nullable()) share->null_fields++;

    /* Check if this is a BIT field with leftover bits in the preamble ... */
    if (col_obj->type() == dd::enum_column_types::BIT) {
      if (!treat_bit_as_char && (col_obj->char_length() & 7))
        leftover_bits += col_obj->char_length() & 7;   // ★★ BIT 的零头位挤进 NULL preamble
    }

    share->reclength += column_pack_length(*col_obj);  // ★★ 逐列 pack_length 累加 → 定长
    share->fields++;
  }

  share->null_bytes = (share->null_fields + leftover_bits + 7) / 8;   // ★ NULL preamble 字节数
  share->last_null_bit_pos = (share->null_fields + leftover_bits) & 7;
  share->reclength += share->null_bytes;               // ★ NULL preamble 计入 reclength

  /* Hack to avoid bugs with small static rows in MySQL. */
  share->reclength = std::max<size_t>(min_length, share->reclength);
  share->stored_rec_length = share->reclength;
  return false;
}
```

```cpp
  // 分配：sql/default_values.cc:290
  share->rec_buff_length = ALIGN_SIZE(share->reclength + 1 + extra_length);
  //                                                    ↑ 历史原因：字段 offset 从 1 开始计
  share->default_values = reinterpret_cast<uchar *>(
      share->mem_root.Alloc(share->rec_buff_length));
```

> ★★ **`null_bytes` 不只装 NULL bit**——`BIT(n)` 里 `n % 8` 的零头位也挤在这里（见上面 `leftover_bits`）。这是"NULL 位图"被复用的第一个例子（第二个例子是 BIT）。

**★ `record[0]` / `record[1]` 的真实分配**（`open_table_from_share()`）：

```cpp
  records = 0;
  if ((db_stat & HA_OPEN_KEYFILE) || (prgflag & DELAYED_OPEN)) records = 1;
  if (prgflag & (READ_ALL + EXTRA_RECORD)) records++;    // ★ 只有这两种 flag 才要第二份

  record = root->ArrayAlloc<uchar>(share->rec_buff_length * records +
                                   share->null_bytes);
  //       ★★ 一个 record[0] 就是 rec_buff_length 字节，定长

  if (records == 0) {
    outparam->record[0] = outparam->record[1] = share->default_values;  // 退化情形
    has_default_values = true;
  } else {
    outparam->record[0] = record;
    if (records > 1) outparam->record[1] = record + share->rec_buff_length;
    else             outparam->record[1] = outparam->record[0];   // Safety：只有一份时两者同址
  }
  outparam->null_flags_saved = record + (records * share->rec_buff_length);
  //  ★ 尾部多出的 null_bytes 是给 save_null_flags() 用的独立备份区

  record = (uchar *)outparam->record[0] - 1;   /* Fieldstart = 1 */
  outparam->null_flags = (uchar *)record + 1;  /* record[0] 开头 */
  //  ★★ 因此 null_flags == record[0]：NULL 位图就在行首
```

```cpp
  move_offset = outparam->record[0] - share->default_values +
                (internal_tmp ? 2 * share->rec_buff_length : 0);

  for (i = 0; i < share->fields; i++, field_ptr++) {
    Field *new_field = share->field[i]->clone(root);
    *field_ptr = new_field;
    new_field->init(outparam);
    new_field->move_field_offset(move_offset);   // ★★ ptr 与 m_null_ptr 一起平移
    if (!has_default_values) new_field->reset(); // ★ 只 memset 0，并不注入默认值
  }
```

> ⚠️ **`open_table_from_share()` 里并没有把 `share->default_values` memcpy 到 `record[0]`**——字段 clone 后只做 `reset()`（清零）。默认值的注入由调用方用宏完成：`restore_record(table, s->default_values)` / `empty_record()`。
> ⚠️ **不存在 `alloc_records()` / `free_records()` / `copy_record()`**（全仓 0 命中）。行拷贝就是那三个宏 `store_record` / `restore_record` / `cmp_record`。

**★ 三个宏的原文**（注意拷贝长度是 `reclength`，**不是** `rec_buff_length`）：

```cpp
#define store_record(A, B) \
  memcpy((A)->B, (A)->record[0], (size_t)(A)->s->reclength)
#define restore_record(A, B) \
  memcpy((A)->record[0], (A)->B, (size_t)(A)->s->reclength)
#define cmp_record(A, B) \
  memcmp((A)->record[0], (A)->B, (size_t)(A)->s->reclength)
```

```cpp
static inline void empty_record(TABLE *table) {
  restore_record(table, s->default_values);          // ① 搬默认行
  if (table->s->null_bytes > 0)
    memset(table->null_flags, 255, table->s->null_bytes);  // ② NULL 位图全置 1 → 全 NULL 行
}
```

**★ `record[0]` 的状态机**（`TABLE::m_status`，八个标志位）：

```cpp
#define STATUS_NOT_STARTED 1
#define STATUS_NOT_FOUND   2
#define STATUS_UPDATED     16
#define STATUS_NULL_ROW    32
#define STATUS_DELETED     64
```

```cpp
  void set_null_row() {
    null_row = true;
    m_status |= STATUS_NULL_ROW;
    if (s->null_bytes > 0) memset(null_flags, 255, s->null_bytes);
    //  ★★ 外连接 NULL-complemented 行：把行首 null_bytes 全置 1
  }
```

> ★ `set_null_row()` 是**破坏性**的（注释明说 "this is a destructive operation since the NULL value bit vector is overwritten"），所以需要配套的 `save_null_flags()` / `restore_null_flags()` 把原位图备份到 `null_flags_saved`。

**★ 与 InnoDB `rec_t` 的鲜明对照**（本篇的核心张力就在这里）：

| | server `record[0]` | InnoDB `rec_t` |
|---|---|---|
| 长度 | **定长**（按表定义算满） | **变长**（按实际数据） |
| 字段定位 | `Field::ptr` **固定偏移**，O(1) | 靠 **offsets 数组**现算 |
| 空间 | 浪费（短行也占满） | 紧凑（变长只存实际长度 + NULL 走位图） |
| 代价 | 内存放大 | 每次访问要解析 offsets |

⇒ **两者是同一个权衡的两个极端**：server 层要"随机访问某一列"极快（表达式求值频繁按列取值），故用**空间换偏移恒定**；引擎要"页内塞更多行、减少 IO"，故用**解析成本换紧凑**。这正是"必须有两种表示"最根本的理由——不是历史包袱，是两种访问模式的约束根本冲突。

#### ★ `record[0]` 的生命周期

```
open_table()                分配（table.cc:2942）：一次分配 2*rec_buff_length + null_bytes
   ↓
每条语句 / 每一行
   ├─ 写方向：fill_record() → save_in_field() → Field::store()   逐列就地填充
   ├─ 读方向：引擎 rnd_next(buf=record[0]) 整块覆盖 → Field::val_*() 解释
   └─ 行与行之间：★ 不重新分配，直接覆盖（Field 对象始终只有一份）
   ↓
表关闭 / 归还 table cache    整块随之释放或复用
```

三个要点：

1. **分配一次、反复覆盖**——`record[0]` 不是"一行一份"，而是**整个 `TABLE` 实例一份**；读到第几行它就是第几行。这正是"`Field` 只需描述偏移"的前提
2. **`record[1]` 是主动拷贝的副本，不是自动留存的历史**——只有显式调用 `store_record(table, record[1])` 才会把当前行 memcpy 过去（UPDATE 前镜像、JOIN 比较都靠这一步）
3. ⚠️ **缓冲复用时不清零**——上一行的字节会残留。所以"写一行"必须**每一列都写到**：`fill_record()` 对 SQL 里未提供的列要主动填默认值，不能跳过。这是"复用缓冲"换来的代价

"拷贝/比较整行"的三个宏就是全部实现（`sql/table.h`）：

```cpp
// ★ 拷贝单位是 s->reclength —— 整行定长，一次 memcpy 到底
#define store_record(A, B)    memcpy((A)->B, (A)->record[0], (size_t)(A)->s->reclength)
#define restore_record(A, B)  memcpy((A)->record[0], (A)->B, (size_t)(A)->s->reclength)
#define cmp_record(A, B)      memcmp((A)->record[0], (A)->B, (size_t)(A)->s->reclength)
```

两个观察：① **`store_record` = 存档、`restore_record` = 恢复**（UPDATE 前镜像、触发器读 OLD 行、JOIN 左右表比较都靠这对）；② **`cmp_record` 是逐字节比较**——"整行是否相同"的快速判断，但它**不看 collation**（同一逻辑值若字节表示不同就判不等），只适合"贴标签式"的相等判断。

> **边界**：`Item::val_*()` 的完整剖析（5 个 `val_xxx` 的取舍、求值缓存 `Item_cache_*`、递归栈深度）见 [`13_item_expression.md`](../../server/query/13_item_expression.md)；`fill_record()` 在 DML 写入流程中的角色见 [`10_dml.md`](../../server/query/10_dml.md)。本篇只讲它们作为"行缓冲填充链路"的一环。

于是"一行"在 server 的完整视图是：

```
TABLE_SHARE（共享元数据）
  ├─ fields（列数）、field[]（Field 模板）、rec_buff_length、null_bytes
  └─ TABLE（每个实例）
       ├─ record[0] ──► [NULL 位图][列1 字节][列2 字节]...   ← 裸字节
       ├─ record[1] ──► 另一块同样大小的缓冲
       └─ field[]  ──► 每个 Field 的 ptr 指向 record[0] 内某偏移（"视图"）
                       切到 record[1] 靠 move_field_offset()
```

**关键理解**：`Field::ptr` 是"游标"而非"拥有者"。同一个 `Field` 对象通过 `move_field_offset()` 可以在 `record[0]` 和 `record[1]` 之间切换解释——这是 server 用一套 `Field` 对象操作多行缓冲的手法（代价是容易搞混当前指向哪块）。

#### 配套结构

**`String`**（`include/sql_string.h`）：server 的字符串容器，自带 `CHARSET_INFO*`——是表达式求值与 `Field` 存取之间的交换格式（完整剖析与字符集转换见 [`charset.md`](../../server/infra/charset.md)）。

**`MY_BITMAP`**（`include/my_bitmap.h`）：`read_set` / `write_set` 的载体，标记"本次操作涉及哪些列"：

```c
struct MY_BITMAP {
  my_bitmap_map *bitmap;    // 位数组（以 word 为单位）
  uint n_bits;              // 已占用的位数（= 列数）
  my_bitmap_map last_word_mask;
  my_bitmap_map *last_word_ptr;   // 末 word 缓存，加速常用位判定
};
```

它决定转换时"要不要处理这一列"——`Field` 只被读到/写到 `read_set`/`write_set` 里的列，其余列连解释都不做（这是"只读部分列"省开销的第一道闸，第二道是模板）。

**`Copy_field`**（`sql/field.h`）：字段间拷贝与转换，用于 ALTER、临时表、类型转换。它的设计是**"转换策略"函数指针**：

```c
class Copy_field {
  using Copy_func = void(Copy_field *, const Field *, Field *);
  void (*m_do_copy)(Copy_field *, const Field *, Field *);   // 常规转换
  void (*m_do_copy2)(Copy_field *, const Field *, Field *);  // ★ 处理 NULL 的另一套
  Field *m_from_field{nullptr};
  Field *m_to_field{nullptr};
  void set(Field *to, Field *from);   // 按两侧类型选好 m_do_copy
  ...
};
```

`set(to, from)` 时按源/目标 `Field` 的类型组合**选定一个专门的转换函数**（如 `do_field_string`、`do_field_blob`、`do_copy_null`…）——一次选定、之后每行只调函数指针，避免逐行做类型分发。`m_do_copy2` 的存在是因为 NULL 的处理逻辑复杂到值得单独一套函数。这是"按类型组合预选策略"的典型手法，与 `dtype` 的位域打包同源（把决策成本从热路径挪到一次初始化）。

### 二、跨层边界：handler 接口

server 与引擎之间只传 **一个 `uchar*`**：

```cpp
int handler::ha_write_row(uchar *buf);          // buf 就是 TABLE->record[0]
virtual int write_row(uchar *buf [[maybe_unused]]);
```

**`buf` 的格式就是 MySQL 行格式**——`sql/table.h:1447` 的注释直说 `record[2]` 存的是 "mysql format的行数据"，即"NULL 位图 + 各列按 `Field::pack_length()` 排布的字节"（布局细节见上面方案 B）。

★ **两份缓冲的分工**：`TABLE::record[2]` 是**同结构的两份**——`record[0]` 是"当前行 / 工作缓冲"，`record[1]` 是暂存旧值的副本。`store_record(table, record[1])` / `restore_record(table, record[1])` 两个宏就是在两者间 `memcpy`（`sql/table.h:146-149`）。所以 UPDATE 的参数顺序是：

```cpp
error = table->file->ha_update_row(table->record[1], table->record[0]);   // sql/log_event.cc:12367
//                                 ^old_data          ^new_data
```

⇒ 引擎拿到的是**两份字节**，自己比对出"哪些列真的变了"（InnoDB 据此构造 `upd_t` 增量，见「六、端到端」的 UPDATE 路径）——**server 不告诉引擎改了哪些列**，这也是"只传字节"的另一面。

这是整个架构的关键约束：**引擎看不到 `Field`、看不到 `TABLE`，只看到一段字节**。引擎靠自己的数据字典（`dict_table_t` / `dict_col_t`）知道这段字节怎么切分，靠 `prtype` 知道每列的类型与字符集。这条"只传字节 + 引擎自带字典"的设计正是插件式引擎能成立的前提。

### 三、InnoDB 内存层：dtype / dfield / dtuple

#### dtype_t：类型描述（8 字节 + 位域打包）

```c
struct dtype_t {
  unsigned prtype : 32;     /* precise type：MySQL type code + charset code
                             + nullability / signedness / binary / true VARCHAR 等标志 */
  unsigned mtype : 8;       /* main data type（InnoDB 自己的主类型分类） */
  unsigned len : 16;        /* 长度；true VARCHAR 时是最大字节长度（不含 1-2 字节长度前缀） */
  unsigned mbminmaxlen : 5; /* 打包编码的最小/最大字符字节数 */
  bool is_virtual() const { return ((prtype & DATA_VIRTUAL) == DATA_VIRTUAL); }
};
```

**`mtype` 是 InnoDB 自己的主类型分类**（8 位，上限 `DATA_MTYPE_MAX = 63`），与 prtype 里的 MySQL type code 是两套编码。完整清单：

| mtype | 值 | 含义 |
|---|---|---|
| `DATA_MISSING` | 0 | 缺失列 |
| `DATA_VARCHAR` | 1 | latin1 的变长字符（**latin1 专属**） |
| `DATA_CHAR` | 2 | latin1 的定长字符（**latin1 专属**） |
| `DATA_FIXBINARY` | 3 | 定长二进制 |
| `DATA_BINARY` | 4 | 二进制 |
| `DATA_BLOB` | 5 | BLOB |
| `DATA_INT` | 6 | 整数 |
| `DATA_SYS_CHILD` | 7 | 子系统的系统列 |
| `DATA_SYS` | 8 | 系统列（DB_TRX_ID / DB_ROLL_PTR / DB_ROW_ID） |
| `DATA_FLOAT` | 9 | 单精度浮点 |
| `DATA_DOUBLE` | 10 | 双精度浮点 |
| `DATA_DECIMAL` | 11 | DECIMAL |
| `DATA_VARMYSQL` | 12 | **通用**变长字符（非 latin1 走这里） |
| `DATA_MYSQL` | 13 | **通用**字符（非 latin1 走这里） |
| `DATA_GEOMETRY` | 14 | 几何类型 |
| `DATA_POINT` | 15 | POINT |
| `DATA_VAR_POINT` | 16 | 变长 POINT |

★ **易混淆**：`mtype = 15` 是 `DATA_POINT`，而 `DATA_MYSQL_TRUE_VARCHAR = 15` 是 **prtype 里的 MySQL type code**——两个 15 分属不同字段（mtype 与 prtype），别搞混。这也正是"为什么类型有两套编码"最容易被绕晕的地方。

**`prtype` 的位布局**（完整版，字符集部分详见 [`charset.md`](../../server/infra/charset.md)）：

```
prtype (32 bit)
├── bit 0-7    MySQL type code          ← DATA_MYSQL_TYPE_MASK = 255
├── bit 8-14   标志位（7 个，见下表）
├── bit 16-30  charset-collation id     ← 15 位，(prtype >> 16) & CHAR_COLL_MASK(32767)
```

**bit 8-14 的 7 个标志位**（均为 `data0type.h` 里的常量，可直接验证）：

| 常量 | 值 | 位 | 含义 |
|---|---|---|---|
| `DATA_NOT_NULL` | 256 | 8 | 列声明为 NOT NULL |
| `DATA_UNSIGNED` | 512 | 9 | 无符号整数 |
| `DATA_BINARY_TYPE` | 1024 | 10 | 用 binary collation（>= 4.0.14 建的表） |
| `DATA_GIS_MBR` | 2048 | 11 | 该列是 GIS 的 MBR |
| `DATA_LONG_TRUE_VARCHAR` | 4096 | 12 | 长度头**不是 1 字节**（即需要 2 字节） |
| `DATA_VIRTUAL` | 8192 | 13 | 虚拟列 |
| `DATA_MULTI_VALUE` | 16384 | 14 | 多值虚拟列（JSON 数组上的索引） |

⇒ 这样前面那段转换代码才读得通：

```cpp
if (dtype_get_mysql_type(dtype) == DATA_MYSQL_TRUE_VARCHAR) {
  ...
  if (dtype->prtype & DATA_LONG_TRUE_VARCHAR) {   // 长度头是 2 字节而非 1 字节
```

⚠️ 注意 `DATA_LONG_TRUE_VARCHAR` 名字里的 **LONG 指"长度头长"**，不是"列很长"——它标记的是"这个 true VARCHAR 的长度前缀要占 2 字节"。```

取出 MySQL 类型码就一行——`dtype_get_mysql_type()`（`data0type.ic`）：

```cpp
return (type->prtype & 0xFFUL);   // 低 8 位即 MySQL type code
```

**★ `dtype_t` 的权威源码**（含一条极重要的注释：哪些字段**不参与排序**）：

```cpp
struct dtype_t {
  unsigned prtype : 32;      /*!< precise type; MySQL data type, charset code,
                             flags to indicate nullability, signedness, whether
                             this is a binary string, whether this is a true
                             VARCHAR where MySQL uses 2 bytes to store the length */
  unsigned mtype : 8;        /*!< main data type */

  /* the remaining fields do not affect alphabetical ordering: */
  //  ★★★ 这条注释是理解"为什么磁盘上 6 字节就能存下排序信息"的关键

  unsigned len : 16;         /*!< length; for MySQL data this is
                             field->pack_length(), except that for a >= 5.0.3
                             type true VARCHAR this is the maximum byte length
                             of the string data */
  unsigned mbminmaxlen : 5;  /*!< minimum and maximum length of a character,
                             in bytes; DATA_MBMINMAXLEN(mbminlen,mbmaxlen) */

  bool is_virtual() const { return ((prtype & DATA_VIRTUAL) == DATA_VIRTUAL); }
};
```

| 成员 | 位宽 | 含义 | 影响排序？ |
|---|---|---|---|
| `prtype` | **32** | 类型码(bit0-7) + flags(bit8-14) + charset-coll(bit16-30) | **是** |
| `mtype` | **8** | 主类型（合法 1..16，上限 63） | **是** |
| `len` | **16** | 精度/字节长度 | 否 |
| `mbminmaxlen` | **5** | `mbmaxlen * 5 + mbminlen` | 否 |
| 合计 | **61 bit** | | |

> ★★ **`len:16` 只有 16 位 → 单列最大长度 65535 字节**，这正是 MySQL VARCHAR 上限的来源。
> ★★ **`mtype` 必须 ≤ 63**，因为磁盘格式 `dtype_new_store_for_order_and_null_size()` 只留 6 bit，bit6/bit7 被借去放 `DATA_BINARY_TYPE` / `DATA_MULTI_VALUE`。

**★ `mbminmaxlen` 的打包公式**（为什么基数是 **5**）：

```cpp
/* Maximum multi-byte character length in bytes, plus 1 */
constexpr uint32_t DATA_MBMAX = 5;

inline ulint DATA_MBMINMAXLEN(ulint mbminlen, ulint mbmaxlen) {
  return mbmaxlen * DATA_MBMAX + mbminlen;              // 编码：max*5 + min
}
inline ulint DATA_MBMINLEN(ulint mbminmaxlen) {
  return UNIV_EXPECT(mbminmaxlen % DATA_MBMAX, 1);      // 解码：%5
}
inline ulint DATA_MBMAXLEN(ulint mbminmaxlen) {
  return mbminmaxlen / DATA_MBMAX;                      // 解码：/5
}
```

> ★★ 一个字符最多 4 字节 ⇒ min/max ∈ {1,2,3,4} ⇒ 用 5 作基数时 `max*5+min` 的 16 种组合**互不冲突**，最大值 `4*5+4 = 24 < 32`，正好塞进 5 bit。
> ⚠️ 三个都是 **`inline ulint` 函数，不是宏**（8.0 已"去宏化"）；`UNIV_EXPECT(x, 1)` 是给编译器的分支预测提示（"多半是 1"，即 latin1/单字节）。

**★ `dtype_set()` 揭示了一点：`mbminmaxlen` 不是调用者给的，而是推导出来的**：

```cpp
static inline void dtype_set(dtype_t *type, ulint mtype, ulint prtype, ulint len) {
  ut_ad(type);
  ut_ad(mtype <= DATA_MTYPE_MAX);
  type->mtype = mtype;
  type->prtype = prtype;
  type->len = len;
  dtype_set_mblen(type);      // ★ mbminmaxlen 由 (mtype, prtype) 推导
}
```

```cpp
static inline void dtype_get_mblen(ulint mtype, ulint prtype,
                                   ulint *mbminlen, ulint *mbmaxlen) {
  if (dtype_is_string_type(mtype)) {
    innobase_get_cset_width(dtype_get_charset_coll(prtype), mbminlen, mbmaxlen);
    ...
  } else {
    *mbminlen = *mbmaxlen = 0;   // ★ 非字符类型 → 0
  }
}
```

**★★ 为什么 MySQL 的类型码会出现在 InnoDB？——三段硬证据**：

```cpp
// ① 编译期把两个枚举钉死（ha_innodb.cc:5255）
  static_assert(DATA_MYSQL_TRUE_VARCHAR == (ulint)MYSQL_TYPE_VARCHAR);
```

```cpp
// ② 注释解释了"为什么必须 < 256"（ha_innodb.cc:7900）
  /* The following asserts try to check that the MySQL type code fits in
  8 bits: this is used in ibuf and also when DATA_NOT_NULL is ORed to
  the type */
  assert((ulint)MYSQL_TYPE_STRING < 256);
  assert((ulint)MYSQL_TYPE_VAR_STRING < 256);
  assert((ulint)MYSQL_TYPE_DOUBLE < 256);
  assert((ulint)MYSQL_TYPE_FLOAT < 256);
  assert((ulint)MYSQL_TYPE_DECIMAL < 256);
```

> ★★ 注释明说第一个用途是 **ibuf（change buffer）**——ibuf 记录必须自解释，序列化时低字节直接存 MySQL 原始类型码，反序列化无需回查数据字典。
> ⚠️ **InnoDB 侧并没有一张完整的 `DATA_MYSQL_*` 类型码表**——只有 `DATA_MYSQL_TYPE_MASK = 255` 与 `DATA_MYSQL_TRUE_VARCHAR = 15` 两个符号。真正的全表在 SQL 层 `include/field_types.h:53` 的 `enum enum_field_types`（`MYSQL_TYPE_GEOMETRY = 255` 正好是上限，这就是 mask = 255 的来源）。

#### ★ 为什么"MySQL 的类型码"会出现在 InnoDB 的 `data0type.h` 里？

这是**插件式架构的直接后果**，三个理由：

1. **引擎看不到 `Field`**：跨 handler 边界后只剩字节（见本节开头"整个架构的关键约束"）。但 InnoDB 要做**双向无损转换**——写方向把 MySQL 行拆成引擎行，读方向还得**还原回 MySQL 行**。读方向必须知道"这几个字节在 MySQL 侧是什么类型"才能还原 ⇒ 类型码只能跟着列定义（`dict_col_t`）一起存在引擎里。
2. **有些类型的"引擎类型"是刻意挑的近似物**：ENUM/SET 物理上就是无符号整数 → 映射 `DATA_INT`（源码注释："data is actually internally stored as an unsigned integer code"）；JSON 与 BLOB 同构 → `DATA_BLOB`（注释 `// JSON fields are stored as BLOBs`）；GEOMETRY → `DATA_GEOMETRY`。这些列在引擎侧被当作对应 mtype 处理，但 prtype 低字节仍保留 MySQL 的真实 type code——**两边信息都要留**才能无损还原。
3. **某些编码细节由 MySQL 的列定义决定**：最典型的是 true VARCHAR 的长度头是 1 还是 2 字节——取决于 `VARCHAR(n)` 的 n 需要几个字节表示，**这跟引擎无关，只跟 MySQL 的列定义有关**。

⇒ 一句话：**`mtype` 是"引擎怎么存"，`prtype` 里的 MySQL type code 是"这列在 MySQL 侧是什么"**。两者都要有，因为引擎既要**存得下**、又要**还原得回去**。

**完整的映射函数是 `get_innobase_type_from_mysql_type()`**（`ha_innodb.cc`，注意函数名带 `_type` 后缀；DD 侧另有一个不建 Field 对象的 `get_innobase_type_from_mysql_dd_type()`）：

```cpp
ulint get_innobase_type_from_mysql_type(ulint *unsigned_flag, const void *f) {
  const class Field *field = reinterpret_cast<const class Field *>(f);
  if (field->is_flag_set(UNSIGNED_FLAG)) *unsigned_flag = DATA_UNSIGNED;
  else *unsigned_flag = 0;
  if (field->real_type() == MYSQL_TYPE_ENUM || field->real_type() == MYSQL_TYPE_SET) {
    *unsigned_flag = DATA_UNSIGNED;   // ★ ENUM/SET 强制无符号
    return (DATA_INT);
  }
  switch (field->type()) { ... }
}
```

完整映射表（`type()` 是 SQL 公开类型、`real_type()` 是物理类型，两者要配合用）：

| MySQL 类型 | 判定条件 | InnoDB mtype |
|---|---|---|
| VAR_STRING / VARCHAR | `binary()` | `DATA_BINARY` |
| 同上 | charset == latin1 | `DATA_VARCHAR` |
| 同上 | 其它 charset | `DATA_VARMYSQL` |
| BIT / STRING（含 CHAR） | `binary()` | `DATA_FIXBINARY` |
| 同上 | latin1 | `DATA_CHAR` |
| 同上 | 其它 charset | `DATA_MYSQL` |
| NEWDECIMAL | — | **`DATA_FIXBINARY`** |
| LONG / LONGLONG / TINY / SHORT / INT24 / DATE / YEAR / NEWDATE / BOOL | — | `DATA_INT` |
| **ENUM / SET** | `real_type()` 特判 | **`DATA_INT`** + 强制 `DATA_UNSIGNED` |
| TIME / DATETIME / TIMESTAMP | `real_type()` 是**老**类型 | `DATA_INT` |
| TIME2 / DATETIME2 / TIMESTAMP2 | `real_type()` 是新 `*2` 类型 | **`DATA_FIXBINARY`** |
| FLOAT / DOUBLE | — | `DATA_FLOAT` / `DATA_DOUBLE` |
| DECIMAL（4.1 前老式 ASCII） | — | `DATA_DECIMAL` |
| GEOMETRY | — | `DATA_GEOMETRY` |
| TINY/MEDIUM/BLOB/LONG_BLOB/**JSON** | — | `DATA_BLOB` |
| NULL | — | 返回 0（`DATA_MISSING`） |
| 其它未知 | — | `ut_error` |

完整 switch（真实代码，就近注释标出三个易错点）：

```cpp
switch (field->type()) {
    /* NOTE that we only allow string types in DATA_MYSQL and DATA_VARMYSQL */
  case MYSQL_TYPE_VAR_STRING: /* old <= 4.1 VARCHAR */
  case MYSQL_TYPE_VARCHAR:    /* new >= 5.0.3 true VARCHAR */
    if (field->binary())                      return (DATA_BINARY);
    else if (field->charset() == &my_charset_latin1) return (DATA_VARCHAR);   // ★ latin1 分家
    else                                      return (DATA_VARMYSQL);
  case MYSQL_TYPE_BIT:
  case MYSQL_TYPE_STRING:
    if (field->binary())                      return (DATA_FIXBINARY);  // ★ BIT 也走这里
    else if (field->charset() == &my_charset_latin1) return (DATA_CHAR);
    else                                      return (DATA_MYSQL);
  case MYSQL_TYPE_NEWDECIMAL:                 return (DATA_FIXBINARY);
  case MYSQL_TYPE_LONG:      case MYSQL_TYPE_LONGLONG:  case MYSQL_TYPE_TINY:
  case MYSQL_TYPE_SHORT:     case MYSQL_TYPE_INT24:
  case MYSQL_TYPE_DATE:      case MYSQL_TYPE_YEAR:      case MYSQL_TYPE_NEWDATE:
  case MYSQL_TYPE_BOOL:
    return (DATA_INT);
  case MYSQL_TYPE_TIME:      case MYSQL_TYPE_DATETIME:  case MYSQL_TYPE_TIMESTAMP:
  case MYSQL_TYPE_TIME2:     case MYSQL_TYPE_DATETIME2: case MYSQL_TYPE_TIMESTAMP2:
    switch (field->real_type()) {             // ★ 易错点②：再套一层 real_type()
      case MYSQL_TYPE_TIME: case MYSQL_TYPE_DATETIME: case MYSQL_TYPE_TIMESTAMP:
        return (DATA_INT);                    //    老格式：record 里是小端整数
      default:
        assert((ulint)MYSQL_TYPE_DECIMAL < 256);
        [[fallthrough]];
      case MYSQL_TYPE_TIME2: case MYSQL_TYPE_DATETIME2: case MYSQL_TYPE_TIMESTAMP2:
        return (DATA_FIXBINARY);              //    新格式（*2）：record 里是落盘二进制
    }
  case MYSQL_TYPE_FLOAT:                         return (DATA_FLOAT);
  case MYSQL_TYPE_DOUBLE:                        return (DATA_DOUBLE);
  case MYSQL_TYPE_DECIMAL:                       return (DATA_DECIMAL);   // 4.1 前老式 ASCII
  case MYSQL_TYPE_GEOMETRY:                      return (DATA_GEOMETRY);
  case MYSQL_TYPE_TINY_BLOB: case MYSQL_TYPE_MEDIUM_BLOB:
  case MYSQL_TYPE_BLOB:      case MYSQL_TYPE_LONG_BLOB:
  case MYSQL_TYPE_JSON:  // JSON fields are stored as BLOBs   ← ★ 易错点
    return (DATA_BLOB);
  case MYSQL_TYPE_NULL:
    /* MySQL currently accepts "NULL" datatype, but will reject such
    datatype in the next release. We will cope with it and not trigger
    assertion failure in 5.1 */
    break;                                    // → return 0（DATA_MISSING）
  default:  ut_error;                         // 未知类型：直接断言
}
return (0);
```

三个易错点：① **ENUM/SET 走 `real_type()`**（因为 `Field_enum::type()` 返回 `MYSQL_TYPE_STRING`、`real_type()` 才返回 ENUM/SET——它们在函数开头就被特判成 `DATA_INT`）；② **新式时间类型（`*2`）是 `DATA_FIXBINARY` 而非 `DATA_INT`**——因为它们在 record 里已经是可 memcmp 的落盘格式（见后面的"时间编码"）；③ **`is_unsigned` 只影响 prtype 的 `DATA_UNSIGNED` 位，不影响 mtype**（无符号整数仍是 `DATA_INT`）。

**mtype 全清单与 prtype 位域**：

```c
DATA_MISSING=0  DATA_VARCHAR=1  DATA_CHAR=2      DATA_FIXBINARY=3   DATA_BINARY=4
DATA_BLOB=5     DATA_INT=6      DATA_SYS_CHILD=7 DATA_SYS=8
DATA_FLOAT=9    DATA_DOUBLE=10  DATA_DECIMAL=11  DATA_VARMYSQL=12   DATA_MYSQL=13
DATA_GEOMETRY=14  DATA_POINT=15  DATA_VAR_POINT=16     DATA_MTYPE_MAX=63
```

`DATA_MTYPE_MAX = 63` 不是随便取的——`dtype_store_for_order_and_null_size()` 需要把 mtype 压进 6 位。而 **`DATA_MYSQL_*` 这一族不是 mtype，而是"塞进 prtype 的 MySQL 信息"**：

```c
DATA_MYSQL_TYPE_MASK    = 255;    // prtype 低字节 = MySQL type code（★ 与 mtype 是两套编码）
DATA_MYSQL_TRUE_VARCHAR = 15;     // = MYSQL_TYPE_VARCHAR
DATA_LONG_TRUE_VARCHAR  = 4096;   // 长度前缀是 2 字节
DATA_NOT_NULL = 256;  DATA_UNSIGNED = 512;  DATA_BINARY_TYPE = 1024;
DATA_VIRTUAL = 8192;  DATA_MULTI_VALUE = 16384;
MAX_CHAR_COLL_NUM = 32767;  CHAR_COLL_MASK = 0x7fff;   // collation 号占 15 位
```

```c
// dtype_form_prtype —— prtype = MySQL type + flags（低 16 位） + collation 号（高 16 位）
ulint dtype_form_prtype(ulint old_prtype, ulint charset_coll) {
  ut_a(old_prtype < 256 * 256);
  ut_a(charset_coll <= MAX_CHAR_COLL_NUM);
  return (old_prtype + (charset_coll << 16));   // ★ 左移 16 位
}
```

**mtype 与 prtype 低字节是两套独立编码**——例如同样是 `DATA_BLOB`，prtype 低字节可能是 `MYSQL_TYPE_JSON` 或 `MYSQL_TYPE_BLOB`；同样是 `DATA_INT`，可能是 INT24 或 ENUM。

**为什么 latin1 与其它 charset 分家**（`DATA_CHAR`/`DATA_VARCHAR` vs `DATA_MYSQL`/`DATA_VARMYSQL`）：latin1 的 CHAR/VARCHAR 与二进制串**由 InnoDB 内部按字节比较**（省 CPU），其它 charset 必须跨层回调 MySQL 的比较函数（见 [`charset.md`](../../server/infra/charset.md)）。`DATA_BLOB` 则用 `prtype & DATA_BINARY_TYPE` 区分 BLOB/TEXT。

**"哪些 mtype 定长"的权威实现**（`dtype_get_fixed_size_low()`，返回 `len` = 定长、`0` = 变长）：

```cpp
// storage/innobase/include/data0type.ic
switch (mtype) {
  case DATA_SYS:
#ifdef UNIV_DEBUG
    switch (prtype & DATA_MYSQL_TYPE_MASK) {   // debug 下校验系统列长度
      case DATA_ROW_ID:   ut_ad(len == DATA_ROW_ID_LEN);   break;
      case DATA_TRX_ID:   ut_ad(len == DATA_TRX_ID_LEN);   break;
      case DATA_ROLL_PTR: ut_ad(len == DATA_ROLL_PTR_LEN); break;
      default: ut_d(ut_error); ut_o(return (0));
    }
    [[fallthrough]];
#endif
  case DATA_CHAR:       // latin1 定长（老式 CHAR）
  case DATA_FIXBINARY:  // 二进制定长（BINARY / DECIMAL / DATETIME2）
  case DATA_INT:
  case DATA_FLOAT:
  case DATA_DOUBLE:
  case DATA_POINT:
    return (len);       // ★ 定长

  case DATA_MYSQL:      // ★ 条件定长——最精彩的分支
    if (prtype & DATA_BINARY_TYPE) {
      return (len);     //   二进制 → 定长
    } else if (!comp) {
      return (len);     //   非 COMPACT 行格式（REDUNDANT）→ 定长
    } else {
      // 断言：prtype 里的 charset id 与 mbminmaxlen 一致（debug）
      innobase_get_cset_width(dtype_get_charset_coll(prtype), &i_mbminlen, &i_mbmaxlen);
      ut_ad(DATA_MBMINMAXLEN(i_mbminlen, i_mbmaxlen) == mbminmaxlen);
      if (DATA_MBMINLEN(mbminmaxlen) == DATA_MBMAXLEN(mbminmaxlen)) {
        return (len);   //   定宽编码（ucs2/utf16：mbminlen == mbmaxlen == 2）→ 定长
      }
      // 注意 gbk（mbminlen=1, mbmaxlen=2）走到这里 → 变长
    }
    [[fallthrough]];    // ★ 多字节 charset + COMPACT → 落到下面，变长！

  case DATA_VARCHAR:    // latin1 变长
  case DATA_BINARY:     // VARBINARY
  case DATA_DECIMAL:    // 老式 ASCII DECIMAL
  case DATA_VARMYSQL:   // 任意 charset 变长
  case DATA_VAR_POINT:
  case DATA_GEOMETRY:
  case DATA_BLOB:
    return (0);         // ★ 变长
  default: ut_error;
}
```

三个要点：① **`DATA_MYSQL` 是唯一"条件定长"的 mtype**——binary → 定长；REDUNDANT 行格式 → 定长；只有 **COMPACT/DYNAMIC 且字符集 `mbminlen != mbmaxlen`**（如 utf8mb4）才变长——**这就是"UTF-8 的 CHAR 在 InnoDB 内部并不严格定长"的代码依据**；② **`DATA_FIXBINARY` 覆盖了"二进制 + DECIMAL + DATETIME2"三类**（印证前面的类型映射表）；③ `DATA_SYS` 在 debug 下用 `ut_ad` 校验三个系统列的固定长度（`DATA_ROW_ID_LEN` 等）。

#### `DATA_MYSQL_TRUE_VARCHAR` —— "true" 是什么意思？

```cpp
// data0type.h:171
/** MySQL type code for the >= 5.0.3 format true VARCHAR */
constexpr uint32_t DATA_MYSQL_TRUE_VARCHAR = 15;
```

它**不是"布尔真"**，而是"真正的 VARCHAR"。MySQL **5.0.3 之前**，`VARCHAR` 上限只有 255 字节、且**尾随空格会被剥离**；5.0.3 起上限提到 65535、**尾随空格保留**，成为真正的变长类型。为与旧格式区分，源码里称之为 **true VARCHAR**。

⇒ `15` 就是 MySQL 侧的 `MYSQL_TYPE_VARCHAR`。这个常量用在 prtype 里标记"本列是 5.0.3 格式的真 VARCHAR"，转换时据此决定**长度头的字节数**（`templ->mysql_length_bytes`，见 [`handler.md`](../../server/handler.md)「mysql_row_templ_t」）：

```cpp
if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR) {
  templ->mysql_length_bytes = field->get_length_bytes();   // ha_innodb.cc:6756-6757
}
```

⚠️ 注意 15 这个数字在本篇出现了**两次且含义不同**（见上方"易混淆"）：`mtype = 15` 是 `DATA_POINT`，`prtype & 0xFF == 15` 才是 true VARCHAR。

```cpp
// storage/innobase/include/data0type.h —— mbminmaxlen 的打包/反解
constexpr uint32_t DATA_MBMAX = 5;              // 基数 = mbmaxlen 上限(4) + 1
constexpr uint32_t DATA_POINT_LEN = 25;         // SRID(4) + WKB 头(1+4) + 点数据(8×2)

inline ulint DATA_MBMINMAXLEN(ulint mbminlen, ulint mbmaxlen) {
  return mbmaxlen * DATA_MBMAX + mbminlen;      // 打包：max × 5 + min
}
inline ulint DATA_MBMINLEN(ulint mbminmaxlen) {
  return UNIV_EXPECT(mbminmaxlen % DATA_MBMAX, 1);   // 反解 min（% 5）
}
inline ulint DATA_MBMAXLEN(ulint mbminmaxlen) {
  return mbminmaxlen / DATA_MBMAX;                   // 反解 max（/ 5）
}

// 三个类型判定内联函数——转换/比较代码里到处都是
inline bool DATA_POINT_MTYPE(ulint mtype) {
  return mtype == DATA_POINT || mtype == DATA_VAR_POINT;
}
inline bool DATA_GEOMETRY_MTYPE(ulint mtype) {
  return DATA_POINT_MTYPE(mtype) || mtype == DATA_GEOMETRY;
}
inline bool DATA_LARGE_MTYPE(ulint mtype) {     // "要不要走 off-page 拷贝"的判据
  return mtype == DATA_BLOB || mtype == DATA_VAR_POINT || mtype == DATA_GEOMETRY;
}
```

**为什么基数是 5**：`mbmaxlen ≤ 4`，取 5 保证 `(mbminlen, mbmaxlen)` ↔ 打包值**无二义**（若取 4 则 `max=1,min=4` 与 `max=2,min=0` 会撞车）。

这个字段的意义：**把"该字符集一个字符占几字节"缓存到列元数据里**，使热路径不必查 server 的 `CHARSET_INFO`——这是"引擎不碰字符集"原则下唯一被下沉的字符属性（它只影响字节布局，不影响字符语义）。

#### dfield_t：字段 = 值 + 类型

```c
struct dfield_t {
  void *data;                 // 指向数据（任意内存，不拥有）
  bool ext;                   // true = 外部存储（页外 BLOB）
  unsigned spatial_status : 2;// undo purge 用的空间状态
  unsigned len;               // 数据长度；UNIV_SQL_NULL 表示 SQL NULL
  dtype_t type;               // ★ 类型内嵌（不是指针）
  ...
  byte *blobref() const;      // 外部存储时取 BLOB 引用
  dfield_t *clone(mem_heap_t *heap);
};
```

要点：

- **`dtype_t` 是内嵌成员**（不是指针）——省一次间接访问，代价是 `dfield_t` 变大
- **`ext` 标记外部存储**：长字段放不进页时，`data` 指向的就不是真实数据而是 BLOB 引用，配合 `row_ext_t` 使用
- **`len == UNIV_SQL_NULL` 表示 NULL**——用长度值做哨兵，不额外占一位

#### dtuple_t：元组 = 字段数组 + 元信息

```c
struct dtuple_t {
  uint16_t info_bits;      // 索引记录的 info bits（与 REC 的 info bits 对应）
  uint16_t n_fields;       // 字段总数
  uint16_t n_fields_cmp;   // ★ 比较时只使用前 N 个字段
  dfield_t *fields;        // 字段数组
  uint16_t n_v_fields;     // 虚拟列字段数
  dfield_t *v_fields;      // ★ 虚拟列单独一个数组
  UT_LIST_NODE_T(dtuple_t) tuple_list;   // 可链成链表
  ...
};
```

**`info_bits` 的作用**：源码注释写得很清楚——"index record 的 info bits，默认 0；**当一条索引记录是由 dtuple 构造出来时使用**"。即 dtuple → `rec_t` 的转换过程中，`info_bits` 被写进记录头（最小记录标记、删除标记、instant/版本标记等位，物理含义见 [`record.md`](../../innodb/physical/record.md)）。存取走 `dtuple_get_info_bits()` / `dtuple_set_info_bits()`。

**四个值得注意的设计**：

1. **`n_fields_cmp`**：**索引查找只比较前 `n_fields_cmp` 个字段**，其余字段被忽略。这是"前缀搜索 / 部分匹配"的实现基础（如二级索引记录只比较索引列而不比较隐藏的系统列）。
2. **`v_fields` 是独立数组**：虚拟列不占 `fields`，单独存放——因为虚拟列不真正存储在聚簇索引里（只在二级索引或计算时出现），必须与物理列分开管理。
3. **`UT_LIST_NODE_T(dtuple_t) tuple_list`**：元组可以链成链表（用的是 [`list.md`](list.md) 里剖析的侵入式链表），用于批量构造的场景（如一次插入涉及多个索引记录）。

**★★ `dtuple_t` 的内存布局：fields 紧贴结构体之后，不是单独 malloc**：

```cpp
static inline dtuple_t *dtuple_create_from_mem(void *buf, ulint buf_size,
                                               ulint n_fields, ulint n_v_fields) {
  ulint n_t_fields = n_fields + n_v_fields;      // ★ 总数 = 普通 + 虚拟

  ut_a(buf_size >= DTUPLE_EST_ALLOC(n_t_fields));  // ★ 注意传的是 n_t_fields

  tuple = (dtuple_t *)buf;
  tuple->info_bits = 0;
  tuple->n_fields = n_fields;
  tuple->n_v_fields = n_v_fields;
  tuple->n_fields_cmp = n_fields;
  tuple->fields = (dfield_t *)&tuple[1];         // ★★ fields 紧贴 dtuple_t 之后
  if (n_v_fields > 0) {
    tuple->v_fields = &tuple->fields[n_fields];  // ★★ v_fields 紧随普通 fields 之后
  } else {
    tuple->v_fields = nullptr;                   // ★ 没有虚列时为 nullptr，不是指向末尾
  }
  ...
}
```

```cpp
/* Estimate the number of bytes that are going to be allocated when
creating a new dtuple_t object */
#define DTUPLE_EST_ALLOC(n_fields) \
  (sizeof(dtuple_t) + (n_fields) * sizeof(dfield_t))
```

```
mem_heap_zalloc(heap, DTUPLE_EST_ALLOC(n_fields + n_v_fields))
│
├─ [0 .. sizeof(dtuple_t))          → dtuple_t 本体
├─ [sizeof(dtuple_t) ...]           → tuple->fields = (dfield_t *)&tuple[1]
│    dfield_t[0] .. [n_fields-1]          普通列
└─ [...]                            → tuple->v_fields = &tuple->fields[n_fields]
     dfield_t[n_fields] .. [n_fields+n_v_fields-1]   虚列
```

> ★★ **一次分配、零重定向、cache 友好**——与 InnoDB `lock_t` 后面跟 bitmap 的手法完全一致。代价是：**`dtuple_t*` 不能单独 free，也不能 memcpy 到别处**（内部 `fields` 是自指指针）。
> ⚠️ `dtuple_create()` 本质是 `dtuple_create_with_vcol(heap, n_fields, 0)`。

```cpp
// release 版宏直接暴露了布局：v_fields == fields + n_fields
#define dtuple_get_nth_field(tuple, n)    ((tuple)->fields + (n))
#define dtuple_get_nth_v_field(tuple, n)  ((tuple)->fields + (tuple)->n_fields + (n))
```

> ⚠️ **`dfield_get_type()` / `dfield_get_data()` / `dtuple_get_nth_field()` 都有双实现**：`UNIV_DEBUG` 下是带 `ut_ad` 的 inline 函数，非 debug 下是**宏**。这是 8.0 为了"debug 抓 bug、release 零开销"的典型手法。
> ⚠️ **`dtuple_t` 没有 `types` 成员**（旧版本有）——类型信息已下沉到每个 `dfield_t::type`。
> ⚠️ **不存在 `dtuple_get_v_fields()`**——要拿 `v_fields` 只能直接访问成员或逐个 `dtuple_get_nth_v_field()`。
> ★ 三个计数 `n_fields` / `n_fields_cmp` / `n_v_fields` 都是 **uint16_t**（上限 65535）。

**★ `dfield_t` 的权威定义**（⚠️ 实际有 **5 个成员**，不止 `data/len/type`）：

```cpp
struct dfield_t {
  void *data;                       /*!< pointer to data */
  bool ext;                         /*!< true=externally stored, false=local */
  unsigned spatial_status : 2;      /*!< spatial status of externally stored
                                     field in undo log for purge */
  unsigned len;                     /*!< data length; UNIV_SQL_NULL if SQL null */
  dtype_t type;                     /*!< type of data */

  bool is_virtual() const { return (type.is_virtual()); }
  void reset() { data = nullptr; ext = false; spatial_status = SPATIAL_UNKNOWN, len = 0; }
  ...
  dfield_t() : data(nullptr), ext(0), spatial_status(0), len(0), type({0, 0, 0, 0}) {}
};
```

> ★ 构造函数 `type({0, 0, 0, 0})` 用聚合初始化 4 个位域 → **反向印证 `dtype_t` 的成员顺序是 `{prtype, mtype, len, mbminmaxlen}`**。全 0 意味着 `mtype == DATA_MISSING(0)`。

```cpp
/* 浅拷贝 vs 深拷贝 —— 三个函数语义完全不同，极易混用 */
static inline void dfield_copy_data(dfield_t *f1, const dfield_t *f2) {
  f1->data = f2->data;  f1->len = f2->len;
  f1->ext = f2->ext;    f1->spatial_status = f2->spatial_status;
  //  ★★ 注意：copy_data 不拷 type！
}
static inline void dfield_copy(dfield_t *f1, const dfield_t *f2) {
  *f1 = *f2;            // ★ 整体赋值含 type；data 指针仍共享（浅）
}
static inline void dfield_dup(dfield_t *field, mem_heap_t *heap) {
  if (!dfield_is_null(field) && field->data != nullptr) {
    field->data = mem_heap_dup(heap, field->data, field->len);   // ★ 真正的深拷贝
  }
}
```

> ⚠️ `dfield_set_len()` 会**顺手清掉 `ext` 标志**（`field->ext = 0;`）——这是"改长度即视为本地字段"的隐式约定。

#### 同族结构（都在 `data0data.h`）

除 `dfield_t` / `dtuple_t` 外，同一文件还定义了几个"数据表示的变体"：

| 结构 | 角色 |
|---|---|
| `big_rec_t` / `big_rec_field_t` | **外部大记录**：一行放不进页时，溢出字段在内存中的表示（配合 `dfield_t::ext`） |
| `multi_value_data` | **多值索引**（JSON 数组索引）：一个 key 对应多个值 |
| `upd_t` / `upd_field_t` | **更新向量**：UPDATE 时"改哪个字段、新值是什么" |
| `row_ext_t`（`row0ext.h`） | **外部字段引用集合**：指向页外 BLOB 的指针数组 |

逐个看：

**`multi_value_data`**——多值索引的表示（`dfield_t` 的 `data` 指向它，靠 `dtype` 的 multi-value 标志区分）：

```c
struct multi_value_data {
  const void **datap;    // 指向各个值的指针数组
  uint32_t *data_len;    // 每个值的长度
  uint64_t *conv_buf;    // 整数值的转换缓冲
  uint32_t num_v;        // 值的个数
  uint32_t num_alc;      // 已分配的指针数（容量，区别于 num_v）
};
```

即：一个字段在内存里是"N 个值 + N 个长度"，插入二级索引时展开成 N 条索引记录——这就是多值索引"一个 JSON 数组对应多条索引项"的表示基础。

**`upd_t`**（`row0upd.h`）——更新向量，值得完整看因为它**同时持有两侧的表对象**：

```c
struct upd_t {
  mem_heap_t *heap;          // 指向别的 heap（不拥有）
  mem_heap_t *per_stmt_heap; // 语句级 heap，end_stmt 时清空
  ulint info_bits;           // 记录的新 info bits
  dtuple_t *old_vrow;        // 旧行（用于虚拟列更新）
  dict_table_t *table;       // ★ InnoDB 的表对象
  TABLE *mysql_table;        // ★ MySQL 的表对象（跨层同时持有！）
  ulint n_fields;            // 要更新的字段数
  upd_field_t *fields;       // 更新字段数组
  ...
  void append(const upd_field_t &field) { fields[n_fields++] = field; }
  bool is_modified(const ulint field_no) const;
};
```

三个值得注意的点：

1. **同时持有 `dict_table_t*` 与 `TABLE*`**——`upd_t` 是少见的"跨层结构"，因为 UPDATE 要同时操作两侧（计算虚拟列需要 MySQL 的 `TABLE`，写 undo/索引需要 InnoDB 的 `dict_table_t`）。
2. **两个 heap 分工不同**：`heap` 借用外部（随外部释放），`per_stmt_heap` 是语句级的（`ha_innobase::end_stmt()` 清空）——避免长事务里内存累积。
3. **`upd_field_t` 的 `new_val` 是 `dfield_t`**——更新向量**复用**了字段表示，所以新值的存储/比较与其他路径完全一致（这是"三件套"被复用的典型例子）。

**`row_ext_t`**：外部存储的字段引用数组。当某些列溢出到页外时，`dtuple_t` 里的 `dfield_t::ext = true`，真实数据由 `row_ext_t` 里的 `dfield_t` 数组持有，`blobref()` 取引用。即"行内 dfield 存引用、行外 row_ext 存数据"的两级表示。

**`big_rec_t`**：一行放不进页时（长字段总数超过页容量的 ~1/2），整行转入"外部大记录"模式——`big_rec_t` 持 `big_rec_field_t` 数组（每项是 `field_no` + 溢出数据的指针/长度），页内只留一个指向外部的引用。它与 `row_ext_t` 的区别：`row_ext_t` 是**列级**外部存储（单列 BLOB 溢出），`big_rec_t` 是**行级**外部存储（整行溢出，只有 REDUNDANT 等老格式才需要）。

**内存来源**：`dtuple_t` / `dfield_t` 都从 `mem_heap_t`（`mem0mem.h`）分配——`dtuple_create(heap, n_fields)` 一次性分配元组 + 字段数组，用完随 heap 释放（不需要逐个 free）。

#### 配套元数据：dict_* 与 dtuple 的协作

`dtuple_t` 单独无法工作——比较/排序/序列化都必须搭配**索引元数据**：

```
dtuple_t（值）          dict_index_t（解释规则的持有者）
  ├─ dfield_t[]      ↔   ├─ dict_field_t[]（每列的 prefix_len / fixed_len / 排序方向）
  └─ 无顺序概念       ↔   └─ dict_col_t[]（每列的 prtype / mbminmaxlen / 隐藏列）
                              └─ dict_v_col_t（虚拟列定义）
```

- 比较：`cmp_dtuple_rec_with_match_low(dtuple, rec, index, offsets, ...)` 的 `index` 决定"比到第几列为止、哪些列是前缀"——没有 `index`，dtuple 只是无类型的字节片段
- 序列化：`rec_convert_dtuple_to_rec(buf, index, dtuple)` 同样按 `index` 的字段顺序与类型写字节
- 检索：`row_search_mvcc` 的 `prebuilt->index` 决定模板、回表判据等

这些元数据结构的完整剖析（`dict_table_t` / `dict_col_t` / `dict_field_t` 的逐字段、DD 互转、统计等）属于引擎侧数据字典主题，见 [`../../server/dd/innodb_dict.md`](../../server/dd/innodb_dict.md)，本篇只交代它们与行表示的协作关系。

### 四、InnoDB 物理层：rec_t 与 offsets

```c
typedef byte rec_t;   // 物理记录就是裸字节流
```

**没有结构体**——物理记录是页上的一段字节，靠 **offsets 数组**（字段偏移缓存）解释。这与 `dtuple_t`（结构化）形成鲜明对照：

```
dtuple_t                          rec_t + offsets
┌─────────────────┐              ┌──────────────────────────┐
│ n_fields        │              │ 记录头（info_bits, next） │
│ fields[]        │              │ 字段 1 字节流             │
│  ├─ data/len/typ│              │ 字段 2 字节流             │
│  ├─ data/len/typ│              │ ...                       │
└─────────────────┘              └──────────────────────────┘
  结构化、易比较                    紧凑、可定位、可由引擎自主布局
        ↑                                  ↑
        └──── rec_convert_dtuple_to_rec ────┘
```

**offsets 数组与 dfield 数组的结构对偶**——这是"内存 vs 磁盘"两种表示最精妙的对应：

- `dtuple_t::fields[i]` 是 `dfield_t`（data 指针 + len + dtype），**任意内存、任意顺序**
- `offsets[i]` 是 `ulint`，编码了"第 i 个字段在 `rec_t` 字节流里的起止位置"——对变长字段还要先解码长度前缀才知道
- 两者一一对应：`cmp_dtuple_rec_with_match_low` 就是拿 `dtuple.fields[i]` 与 `rec_t` 上 `offsets[i]` 指向的字节**逐字段直接比较**，不物化任何一方

**offsets 的两个性能设计**（物理布局细节归 [`record.md`](../../innodb/physical/record.md)，这里只讲与表示相关的部分）：

1. **固定 offsets 缓存**：对"全定长 + 无 NULL + 无 instant"的索引，字段偏移与具体行无关，`rec_init_fixed_offsets` 直接从字典算出一份，**无数条记录共享** `index->rec_cache.offsets`——这类表根本不需要逐行解析。
2. **`Rec_offsets` RAII 助手**（`rem0rec.h`）：把"栈数组 + 惰性 heap 扩展"的样板封装成对象，`compute(rec, index)` 复用上次分配的内存，避免每条记录都 malloc。底层约定是"`offsets` 数组可原位重填"（传入的数组容量够就直接重写），所有缓存行为都建立在这个约定上。

### ★ 各数据类型的 record 编码

`record[0]` 里的字节到底长什么样，取决于列类型。这一节把跨越"SQL 值 → 字节"这条线的类型逐个讲清——**它们是"转换桥梁"两个方向都要依赖的知识基础**。

#### DECIMAL：每 9 位十进制压 4 字节 + 补码 + 首字节翻转

DECIMAL 的二进制格式是 **为 memcmp 有序性设计的**（源码注释列出 5 条 property，其中 property 2："binary representations of the same {precision, scale} can be compared with memcmp - with the same result as `decimal_cmp()`"）。

```c
struct decimal_t { int intg, frac, len; bool sign; decimal_digit_t *buf; };
// intg/frac 是十进制位数；len 是 buf 的元素个数（不是字节数）；sign=true 为负
int decimal2bin(const decimal_t *from, uchar *to, int precision, int scale);
```

```c
#define DIG_PER_DEC1 9
static const int dig2bytes[DIG_PER_DEC1 + 1] = {0, 1, 1, 2, 2, 3, 3, 4, 4, 4};
//                                              0位 1位 2位 3位 4位 5位 6位 7位 8位 9位
```

**编码算法**（`decimal2bin` 真身，以 9 位十进制为一组）：

```cpp
// strings/decimal.cc —— 前置：精度自适应（写入时按目标列宽缩放）
buf1 = remove_leading_zeroes(from, &from_intg);
if (intg < from_intg) {            // 值的整数位 > 列的整数位 → 溢出
  buf1 += intg1 - intg0 + (intg1x > 0) - (intg0x > 0);
  error = E_DEC_OVERFLOW;          // ★ 报溢出（调用方据此决定报错/截断）
} else if (isize0 > isize1) {
  while (isize0-- > isize1) *to++ = (char)mask;   // 高位补符号字节（补齐列宽）
}
if (fsize0 < fsize1) { frac1 = frac0; frac1x = frac0x; error = E_DEC_TRUNCATED; }

// 编码主体
const dec1 mask = from->sign ? -1 : 0;             // ★ 负数 → 全 1 掩码（逐位取反）

/* intg1+frac1 part：整数组 —— 每组 9 位十进制压 4 字节 */
for (stop1 = buf1 + intg1 + frac1; buf1 < stop1; to += sizeof(dec1)) {
  dec1 x = *buf1++ ^ mask;      // ★ 补码：负数逐位取反
  assert(sizeof(dec1) == 4);
  mi_int4store(to, x);          //    大端写入
}

/* frac1x part：小数尾组（不足 9 位）—— 按 dig2bytes 选 1~4 字节 */
if (frac1x) {
  dec1 x;
  int i = dig2bytes[frac1x], lim = (frac1 < frac0 ? DIG_PER_DEC1 : frac0x);
  while (frac1x < lim && dig2bytes[frac1x] == i) frac1x++;
  x = div_by_pow10(*buf1, DIG_PER_DEC1 - frac1x) ^ mask;
  switch (i) {                    // 同样的 XOR 掩码，字节数按表选
    case 1: mi_int1store(to, x); break;
    case 2: mi_int2store(to, x); break;
    case 3: mi_int3store(to, x); break;
    case 4: mi_int4store(to, x); break;
    default: assert(0);
  }
  to += i;
}
if (fsize0 > fsize1) {            // 尾段字节数不足 → 补符号字节
  uchar *to_end = orig_to + orig_fsize0 + orig_isize0;
  while (fsize0-- > fsize1 && to < to_end) *to++ = (uchar)mask;
}
orig_to[0] ^= 0x80;               // ★ 最后：首字节最高位翻转（memcmp 序）

// 断言写满了列宽、不多不少
assert(to == orig_to + orig_fsize0 + orig_isize0);
return error;                     // E_DEC_OK / E_DEC_OVERFLOW / E_DEC_TRUNCATED
```

**第 5 步的原因**：memcmp 按**无符号字节**比较，而负数的补码首字节会 ≥ 0x80（看起来"更大"）。把首字节最高位翻转后，负数落在 0x00-0x7F、正数落在 0x80-0xFF，**memcmp 序与数值序一致**。例：`-1234567890.1234` → `7E F2 04 C7 2D FB 2D`。

**列宽公式**：

```c
int intg = precision - scale, intg0 = intg / 9, frac0 = scale / 9;
int intg0x = intg - intg0 * 9, frac0x = scale - frac0 * 9;
return intg0 * sizeof(dec1) + dig2bytes[intg0x] + frac0 * sizeof(dec1) + dig2bytes[frac0x];
```

→ **DECIMAL(10,2) = intg 8 位（dig2bytes[8]=4）+ frac 2 位（dig2bytes[2]=1）= 5 字节**。`my_decimal_get_binary_size()` 只是它的别名；`Field_new_decimal::pack_length()` 直接返回 `bin_size`。

`bin2decimal()` 是严格逆操作（首字节高位=1 ⇒ 正数 ⇒ mask=0），并用 `DIG_MAX`/`powers10[]` 做合法性校验（非法 → `E_DEC_BAD_NUM` + 归零）。

**InnoDB 侧**：`NEWDECIMAL → DATA_FIXBINARY`——**定长二进制**，所以引擎可以纯 memcmp；老式 ASCII DECIMAL 才是 `DATA_DECIMAL`（变长）。

#### 时间类型：位域是"内存打包"，落盘格式是另一回事

一个常见误解是把"5 字节位域布局"记成 DATETIME2 的存储格式。**实际是两套**：

| | 内存打包（`longlong`） | 落盘/落 record |
|---|---|---|
| TIME | 48 位：1 符号 + 1 保留 + 10 小时 + 6 分 + 6 秒 + 24 微秒 | 3 字节整数部分 + 0-3 字节小数（fsp） |
| DATETIME | 64 位：1 符号 + **17 位 (year×13+month)** + 5 日 + 5 时 + 6 分 + 6 秒 + 24 微秒 | **5 字节**整数部分 + 0-3 字节小数 |
| DATE | — | 3 字节老编码（`day + month*32 + year*16*32`） |
| TIMESTAMP | — | **4 字节 epoch** + 0-3 字节小数 |
| YEAR | — | 1 字节，存 `year - 1900` |

```c
// 打包原语
#define my_packed_time_get_int_part(i) ((i) >> 24)
#define my_packed_time_make(i, f) (((ulonglong)(i) << 24) + (f))    // |f| <= 0xffffff

// DATETIME 内存打包
ymd = ((year * 13 + month) << 5) | day;      // 17 位
hms = (hour << 12) | (minute << 6) | second;
tmp = my_packed_time_make((ymd << 17) | hms, second_part);
```

**落盘的关键是把打包值加上偏移常量再大端写出**，使字节序可 memcmp：

```cpp
// mysys/my_time.cc —— DATETIME2 的落盘（"On disk we store as unsigned number
// with DATETIMEF_INT_OFS offset, for HA_KEYTYPE_BINARY compatibility purposes."）
#define DATETIMEF_INT_OFS 0x8000000000LL

/** Store in-memory numeric packed datetime representation to disk. */
void my_datetime_packed_to_binary(longlong nr, uchar *ptr, uint dec) {
  assert(dec <= DATETIME_MAX_DECIMALS);
  /* The value being stored must have been properly rounded or truncated */
  assert((my_packed_time_get_frac_part(nr) %
          static_cast<int>(log_10_int[DATETIME_MAX_DECIMALS - dec])) == 0);

  mi_int5store(ptr, my_packed_time_get_int_part(nr) + DATETIMEF_INT_OFS);  // ★ 5 字节 + 偏移
  switch (dec) {
    case 0: default: break;
    case 1: case 2:
      ptr[5] = static_cast<unsigned char>(
          static_cast<char>(my_packed_time_get_frac_part(nr) / 10000));    // 小数秒 1 字节
      break;
    case 3: case 4:
      mi_int2store(ptr + 5, my_packed_time_get_frac_part(nr) / 100);       // 2 字节
      break;
    case 5: case 6:
      mi_int3store(ptr + 5, my_packed_time_get_frac_part(nr));             // 3 字节
  }
}

/** Convert on-disk datetime representation to in-memory packed numeric. */
longlong my_datetime_packed_from_binary(const uchar *ptr, uint dec) {
  longlong intpart = mi_uint5korr(ptr) - DATETIMEF_INT_OFS;   // ★ 读回先减偏移
  int frac;
  switch (dec) {
    case 0: default: return my_packed_time_make_int(intpart);
    case 1: case 2: frac = (static_cast<int>(static_cast<signed char>(ptr[5]))) * 10000; break;
    case 3: case 4: frac = mi_sint2korr(ptr + 5) * 100; break;
    case 5: case 6: frac = mi_sint3korr(ptr + 5); break;
  }
  return my_packed_time_make(intpart, frac);                  // 载荷重新拼回
}
```

三个要点：① **整数部分 5 字节 + `DATETIMEF_INT_OFS` 偏移**（注释说明是为 `HA_KEYTYPE_BINARY` 兼容——即"可 memcmp"）；② **小数秒单独追加 1/2/3 字节**（由 fsp 决定，注意有符号读取 `mi_sint2korr`/`mi_sint3korr`）；③ 写出前的断言保证"值已被正确 round/truncate 到该精度"。

TIME2 的落盘同构，只是用 `TIMEF_INT_OFS = 0x800000LL`（3 字节）与 `TIMEF_OFS = 0x800000000000LL`（6 字节版本）：

```c
// dec 0:   mi_int3store(ptr, TIMEF_INT_OFS + int_part);
// dec 1-2: ... ptr[3] = frac / 10000;
// dec 3-4: mi_int2store(ptr + 3, frac / 100);
// dec 5-6: mi_int6store(ptr, nr + TIMEF_OFS);
```

★ **负数时间的小数秒"逆序存放"**（源码注释："Negative values are stored with reverse fractional part order, for binary sort compatibility"）——读回时 `intpart++; frac -= 0x100;` 之类的补偿。这是"memcmp 有序"在负数上的代价。

**列宽公式**（`libbinlogevents` 里三个函数，`Field_*f::pack_length()` 直接用）：

```c
my_time_binary_length(dec)      = 3 + (dec + 1) / 2;
my_datetime_binary_length(dec)  = 5 + (dec + 1) / 2;    // dec=0 → 5
my_timestamp_binary_length(dec) = 4 + (dec + 1) / 2;
```

**两代格式的分界不是类名**：`Field_datetime`（老）→ `MYSQL_TYPE_DATETIME`、record 里是 8 字节小端整数（比较要转）；`Field_datetimef`（新）→ `MYSQL_TYPE_DATETIME2`、record 里**就是上面的落盘格式**（可直接 memcmp，`sort_charset() = &my_charset_bin`）。引擎侧映射也随之不同（老 → `DATA_INT`、新 → `DATA_FIXBINARY`）。

**TIMESTAMP 的 2038**：4 字节存无符号 epoch 秒，上限 `TYPE_TIMESTAMP_MAX_VALUE = INT32_MAX`（2038-01-19 03:14:07 UTC）、`TIMESTAMP_MAX_YEAR = 2038`；写盘前还要 `convert_TIME_to_timestamp` 做本地 → UTC 转换。

#### FLOAT / DOUBLE：写侧截断、读侧隐式往返

```cpp
// Field_float::store(double)
const type_conversion_status error = truncate(&nr, FLT_MAX) ? TYPE_WARN_OUT_OF_RANGE : TYPE_OK;
float j = (float)nr;                     // ★ 截成 float
float4store(ptr, j);
// 读回
return double{float4get(ptr)};           // ★ float → double 的隐式往返在这里发生
```

**"存 double 会丢精度但不报错"的机制**：写侧只判 `|v| > FLT_MAX`（越界才 warn），精度损失由"读侧的 `float4get` 还原"客观暴露——所以 SQL 层看不到精度告警。`val_str()` 走 `my_gcvt`/`my_fcvt`，`MAX_DOUBLE_STR_LENGTH = DBL_DIG + 7`（=22，**+2 是防回转取整误差**）。

**InnoDB 侧**：`DATA_FLOAT`/`DATA_DOUBLE` 用 `mach_float_read`/`mach_double_read` 做**数值比较**而不是 memcmp——因为 IEEE 字节序并非 memcmp 有序（data0type.h 注释：`Data types >= DATA_FLOAT must be compared using the whole field, not as binary strings`）。

#### BIT：零头位"借住" NULL 字节区

`Field_bit` 把一个 BIT(n) 拆成两段：

```cpp
uint bit_len;        // = n % 8   —— 零散高位
uint bytes_in_rec;   // = n / 8   —— 完整字节部分
uint pack_length() const final { return (field_length + 7) / 8; }
uint pack_length_in_rec() const final { return bytes_in_rec; }
uint row_pack_length() const final { return bytes_in_rec + (bit_len > 0 ? 1 : 0); }
```

★ **零头位存在 NULL 位区域的空闲位里**（`bit_ptr` + `bit_ofs`）——所以 8.0 的 `Field_bit` **没有 `bit` 成员**（那是 5.x 的写法）。源码注释解释了为什么 `cmp_binary` 不能直接用 `pack_length()`：NULL 字节里也存着数据位。

**五个长度口径**：`pack_length()`（位折算字节）/ `pack_length_in_rec()`（进记录的数据字节）/ `row_pack_length()`（含借住的 NULL 字节）/ `pack_length_from_metadata()`（从复制 metadata 还原）/ 键用的 `HA_KEYTYPE_BIT`。

**溢出语义是"饱和"**（`Field_bit::store` 真实代码）：

```cpp
type_conversion_status Field_bit::store(const char *from, size_t length, const CHARSET_INFO *) {
  int delta;
  for (; length && !*from; from++, length--);      // 跳前导 0
  delta = bytes_in_rec - static_cast<int>(length); // 与"字节部分"的长度差

  if (delta < -1 || (delta == -1 && (uchar)*from > ((1 << bit_len) - 1)) ||
      (!bit_len && delta < 0)) {
    // ★ 溢出 → "饱和"：零头位 + 字节部分全置 1（不报错也不截断）
    set_rec_bits((1 << bit_len) - 1, bit_ptr, bit_ofs, bit_len);
    memset(ptr, 0xff, bytes_in_rec);
    if (current_thd->is_strict_mode())
      set_warning(Sql_condition::SL_WARNING, ER_DATA_TOO_LONG, 1);       // strict
    else
      set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);
    return TYPE_WARN_OUT_OF_RANGE;
  }

  if (delta > 0) {                     // 值比字节部分短 → 左侧补 0
    if (bit_len) clr_rec_bits(bit_ptr, bit_ofs, bit_len);
    memset(ptr, 0, delta);
    memcpy(ptr + delta, from, length);
  } else if (delta == 0) {             // 正好
    if (bit_len) clr_rec_bits(bit_ptr, bit_ofs, bit_len);
    memcpy(ptr, from, length);
  } else {                             // 值更长 → 高位（零头位 + 部分字节）细分
    if (bit_len) { set_rec_bits((uchar)*from, bit_ptr, bit_ofs, bit_len); from++; }
    memcpy(ptr, from, bytes_in_rec);
  }
  return TYPE_OK;
}
```

三个要点：① 溢出是**饱和**（全 1）而不是截断/报错；② **零头位用 `set_rec_bits`/`clr_rec_bits`（写进 NULL 字节区）、字节部分用 `memcpy`**——"两段存储"在代码里的直接体现；③ strict 与非 strict 报不同错误码（`ER_DATA_TOO_LONG` vs `ER_WARN_DATA_OUT_OF_RANGE`）——这里 `store` 内部自己判了 strict，是少数例外。

**InnoDB 侧**：BIT 与 STRING 同一分支（`Field::binary()` 默认为真）→ `DATA_FIXBINARY`；建表时"**InnoDB always treats BIT as char**"（`calc_pack_length(..., true, ...)`）。

#### ENUM / SET：物理是整数，语义是字符串

```c
inline uint get_enum_pack_length(int elements) { return elements < 256 ? 1 : 2; }
inline uint get_set_pack_length(int elements) { uint len = (elements + 7) / 8; return len > 4 ? 8 : len; }
```

ENUM 存**序号**（1 起）、SET 存**位图**（最多 8 字节 = 64 元素）。两个"字符串类型却是整数"的后果：

```cpp
// Field_enum / Field_set
/* enum and set are sorted as integers */
const CHARSET_INFO *sort_charset() const final { return &my_charset_bin; }
```

★ **`sort_charset()` 返回 `my_charset_bin`**——所以分区哈希、临时表 key image、ORDER BY 都把 ENUM/SET 按**整数码**处理，而不是按字符集。引擎侧映射为 `DATA_INT` + 强制 `DATA_UNSIGNED`（源码注释："MySQL has field->type() a string type for these, but the data is actually internally stored as an unsigned integer code!"），但 prtype 低字节保留真实的 `MYSQL_TYPE_ENUM/SET`。

**★ `Field_enum::pack()` 就是按 packlength 分派的整数打包**（⚠️ **`Field_set::pack()` 不存在**——`Field_set final : public Field_enum`，完全复用）：

```cpp
uchar *Field_enum::pack(uchar *to, const uchar *from, size_t max_length) const {
  switch (packlength) {
    case 1: if (max_length > 0) *to = *from; return to + 1;
    case 2: return pack_int16(to, from, max_length);
    case 3: return pack_int24(to, from, max_length);
    case 4: return pack_int32(to, from, max_length);
    case 8: return pack_int64(to, from, max_length);
    default: assert(0);
  }
}
```

> ★★ 正因 ENUM/SET 在 InnoDB 侧是 `DATA_INT`，读方向要**专门加 `!= DATA_INT` 判断**以免被当作 TRUE_VARCHAR 处理——这在 `row_sel_field_store_in_mysql_format_func` 与 `row_sel_convert_mysql_key_to_innobase` 里各有一处（源码注释："We added the check != DATA_INT to make sure we do not treat MySQL ENUM or SET as a true VARCHAR!"）。

**★ FLOAT / DOUBLE：两端字节序恰好一致，所以不需要字节反转**（与 DATA_INT 完全不同）：

```cpp
// 写方向（MySQL 行内，小端）
type_conversion_status Field_float::store(double nr) {
  float j = (float)nr;
  if (table->s->db_low_byte_first) float4store(ptr, j);
  else                             floatstore(ptr, j);
  return error;
}
type_conversion_status Field_double::store(double nr) {
  if (table->s->db_low_byte_first) float8store(ptr, nr);
  else                             doublestore(ptr, nr);
  return error;
}
```

```cpp
// pack：只有大端机才需要翻转字节
uchar *Field_real::pack(uchar *to, const uchar *from, size_t max_length) const {
#ifdef WORDS_BIGENDIAN
  if (!table->s->db_low_byte_first) {
    size_t len = std::min<size_t>(pack_length(), max_length);
    for (size_t i = 0; i < len; ++i) {
      to[i] = from[pack_length() - i - 1];    // ★ 逐字节反转
    }
    return to + pack_length();
  } else
#endif
    return Field::pack(to, from, max_length);   // ★ 小端：原样 memcpy
}
```

> ★ InnoDB 内部 `mach_float_write` / `mach_double_write` 也是小端（`mach0data.ic`，按 `WORDS_BIGENDIAN` 反转）⇒ **FLOAT/DOUBLE 在两种表示里字节序一致，转换是纯 memcpy**。这与 DATA_INT 必须"字节反转 + 翻符号位"形成鲜明对照。
> ★ 比较走 `cmp_whole_field()` 的 `mach_double_read` / `mach_float_read`（**大端**读），因为比较必须按数值而非按字节。

**★ DECIMAL 的二进制编码**（⚠️ `Field_new_decimal::pack()` **不存在**，走基类 `Field::pack()` = 纯 memcpy；编码在 `store_value()` 里由 `decimal2bin()` 完成）：

```cpp
int decimal2bin(const decimal_t *from, uchar *to, int precision, int frac) {
  dec1 mask = from->sign ? -1 : 0, *buf1 = from->buf, *stop1;
  ...
  /* intg1x part */                    // ① 整数零头：1-4 字节（dig2bytes[]）
  if (intg1x) {
    int i = dig2bytes[intg1x];
    dec1 x = mod_by_pow10(*buf1++, intg1x) ^ mask;
    ...
    to += i;
  }
  /* intg1+frac1 part */               // ② 每 9 位一组 4 字节，大端
  for (stop1 = buf1 + intg1 + frac1; buf1 < stop1; to += sizeof(dec1)) {
    dec1 x = *buf1++ ^ mask;
    mi_int4store(to, x);               // ★★ big-endian
  }
  /* frac1x part */                    // ③ 小数零头：1-4 字节
  ...
  orig_to[0] ^= 0x80;                  // ★★★ 最后翻最高 bit
  return error;
}
```

```
  1234567890.1234  (precision=14, scale=4)
  内部 3 个 decimal_digit_t: 1 | 234567890 | 123400000
  hex:                     00-00-00-01 | 0D-FB-38-D2 | 07-5A-EF-40

  中间组满 9 位 → 原样 4 字节:      ....... 0D-FB-38-D2 .......
  首组只有 1 位 → 压成 1 字节:      01       0D-FB-38-D2 .......
  末组 1234     → 压成 2 字节:      01       0D-FB-38-D2 04-D2
  翻最高 bit:                       81       0D FB 38 D2 04 D2   ← 12 字节压成 7 字节

  负数 -1234567890.1234:            7E       F2 04 C7 2D FB 2D   ← 每字节 ^mask 全部取反
```

> ★★ 与 DATA_INT 的关键差别：**DECIMAL 负数每个字节都取反，而 DATA_INT 只翻最高字节的一个 bit**。但最后那句 `orig_to[0] ^= 0x80` 与 InnoDB 的 `*buf ^= 128` **思想同源**——都是为了让 `memcmp` 的无符号序 == 数值序。
> ★ `bin2decimal()` 的逆运算第一步就是 `d_copy[0] ^= 0x80;` 还原，且 `mask = (*from & 0x80) ? 0 : -1`（最高 bit 为 0 ⇒ 负数）。

**★ BIT 的"零头位借住 NULL 字节区"**：

```cpp
uchar *Field_bit::pack(uchar *to, const uchar *from, size_t max_length) const {
  if (max_length == 0) return to + 1;
  size_t length;
  if (bit_len > 0) {
    /* ...
      bit_ptr    Points to the byte (in the null bytes) that holds the
                 odd bits of R1
      ... */
    uchar bits = get_rec_bits(bit_ptr + (from - ptr), bit_ofs, bit_len);
    *to++ = bits;                      // ★ 零头位字节先写
  }
  length = min<size_t>(bytes_in_rec, max_length - (bit_len > 0));
  memcpy(to, from, length);
  return to + length;
}
```

```cpp
class Field_bit : public Field {
 public:
  uchar *bit_ptr;    // position in record where 'uneven' bits store
  uchar bit_ofs;     // offset to 'uneven' high bits
  uint bit_len;      // number of 'uneven' high bits
  uint bytes_in_rec;
  uint32 pack_length() const final { return (uint32)(field_length + 7) / 8; }
  uint32 pack_length_in_rec() const final { return bytes_in_rec; }
  uint row_pack_length() const final {
    return (bytes_in_rec + ((bit_len > 0) ? 1 : 0));
  }
```

> ★★ `bit_ptr` 直指 **NULL 位图字节区**（源码注释 "in the null bytes"）。当 `field_length % 8 != 0`，这 `bit_len` 个高位零头**不住在数据区，而是借住 NULL 位图的一个字节**（`bit_ofs` 指定起始 bit）。这与 `find_record_length()` 里 `leftover_bits` 的计算精确对应。

**★ 时间类型的两套编码**（"位域是内存打包，落盘是另一回事"的源码证据）：

```cpp
// DATE：3 字节小端，day(5b) | month(4b,×32) | year(×512)
void my_date_to_binary(const MYSQL_TIME *ltime, uchar *ptr) {
  long tmp = ltime->day + ltime->month * 32 + ltime->year * 16 * 32;
  int3store(ptr, tmp);                      // ★ little-endian
}

// TIME（旧）：HHMMSS 十进制拼整数，3 字节小端，负值取负
type_conversion_status Field_time::store_internal(const MYSQL_TIME *ltime, int *) {
  long tmp = ((ltime->month ? 0 : ltime->day * 24L) + ltime->hour) * 10000L +
             (ltime->minute * 100 + ltime->second);
  if (ltime->neg) tmp = -tmp;
  int3store(ptr, tmp);
  return TYPE_OK;
}

// DATETIME（旧）：8 字节 YYYYMMDDhhmmss
static inline type_conversion_status datetime_store_internal(TABLE *table,
                                                             ulonglong tmp, uchar *ptr) {
  if (table && table->s->db_low_byte_first) int8store(ptr, tmp);
  else                                      longlongstore(ptr, tmp);
  return TYPE_OK;
}

// TIMESTAMP：4 字节小端 Unix 秒
void Field_timestamp::store_timestamp_internal(const my_timeval *tm) {
  if (table && table->s->db_low_byte_first) int4store(ptr, tm->m_tv_sec);
  else                                      longstore(ptr, (uint32)tm->m_tv_sec);
}
```

```cpp
// TIME(N)/DATETIME(N)：磁盘是 5 字节【大端】+ 0/1/2/3 字节小数
#define TIMEF_INT_OFS      0x800000LL
#define DATETIMEF_INT_OFS  0x8000000000LL

void my_datetime_packed_to_binary(longlong nr, uchar *ptr, uint dec) {
  mi_int5store(ptr, my_packed_time_get_int_part(nr) + DATETIMEF_INT_OFS);  // ★ big-endian
  switch (dec) {
    case 0: default: break;
    case 1: case 2: ptr[5] = (uchar)(char)(frac / 10000); break;
    case 3: case 4: mi_int2store(ptr + 5, frac / 100);    break;
    case 5: case 6: mi_int3store(ptr + 5, frac);
  }
}
```

> ★★ **注意 "加 OFFSET" 与 "XOR 0x80" 是同一思想**：都是把补码数轴平移到无符号空间，让 memcmp 可用。InnoDB 的 `*buf ^= 128`、DECIMAL 的 `orig_to[0] ^= 0x80`、时间的 `+ TIMEF_INT_OFS`——三处形式不同、目的一致。
> ★ 磁盘上的 packed 时间已是**定长大端**，所以 `Field_timef` / `Field_datetimef` / `Field_timestampf` **都不覆写 `pack()`**，直接继承 `Field::pack()` 的 memcpy。

#### VARCHAR 的长度链：一个事实源贯穿四层

```
MySQL 侧 field_length（字节，含 mbmaxlen 放大）
   ↓ HA_VARCHAR_PACKLENGTH(field_length) = (field_length < 256 ? 1 : 2)
Field_varstring::length_bytes（内存记录里的长度头）
   ↓ 建 dict 时：col_len -= length_bytes；若为 2 则置 DATA_LONG_TRUE_VARCHAR 位
prtype 的 DATA_LONG_TRUE_VARCHAR
   ↓ 落 rec 时读这一位
InnoDB rec 里 1 或 2 字节的长度前缀（dict_col->len 已减掉前缀宽度）
```

三处独立代码用同一 bit 决策（从 Field 建 dict、从 DD 建 dict、落 rec），**三方一致**。**唯一的例外**：**键上的长度前缀永远是 2 字节**——`Field_varstring::new_key_field()` 注释："Keys length prefixes are always packed with 2 bytes"（`row0mysql.cc` 里也有 `// In a MySQL key value, lenlen is always 2`）。

**CHAR 的"定长"要看 charset**：`Field_string::pack_length() = field_length`（基类默认，不覆盖）⇒ 在 MySQL 记录里恒占 `field_length` 字节；但映射到 InnoDB 后：binary → `DATA_FIXBINARY`、latin1 → `DATA_CHAR`（都定长），**其它 charset → `DATA_MYSQL`——在 COMPACT 且 `mbminlen != mbmaxlen`（如 utf8mb4）时是变长**。这解释了一个常被忽略的事实：**UTF-8 的 CHAR 列在 InnoDB 内部并不严格定长**（写方向会剥尾空格，见下节的对称处理）。

### 五、转换桥梁

| 方向 | 函数 | 作用 | 详写处 |
|---|---|---|---|
| MySQL 列 → InnoDB 内部格式 | `row_mysql_store_col_in_innobase_format` | 把 server 行缓冲里的一列转成 `dfield_t` 能表示的格式（含多字节 CHAR 的**尾空格剥离**——注意写方向是"剥"，"补"在读方向，见下文②） | 本篇（字符集部分见 [`charset.md`](../../server/infra/charset.md)） |
| dtuple → REC | `rec_convert_dtuple_to_rec` | 序列化：按 index 的字段顺序把 dfield 写进字节流 | **字节级细节见 [`record.md`](../../innodb/physical/record.md)**（NULL 位图、变长长度游标、instant 版本字节） |
| dtuple ↔ REC 比较 | `cmp_dtuple_rec_with_match_low` | 不转换成对方，直接按 offsets 逐字段比较（搜索热路径） | 本篇（它跳过了 dtuple↔REC 的物化） |
| REC → MySQL 行 | `row_sel_store_mysql_rec` / 模板拷贝 | 检索结果回填 `TABLE::record[0]` | 检索侧见 [`../../innodb/row_search.md`](../../innodb/row_search.md) |

★ **与编码文档的分工**（避免重复）：本篇只讲"**结构怎么组织、层与层怎么转换**"（dtuple 是什么、由哪些 dfield 组成、怎么变成 REC）；"**REC 的字节具体怎么摆**"（长度前缀、NULL 位图、偏移目录）归 [`record.md`](../../innodb/physical/record.md)，"**字节级编码原语**"（`mach_*` 写整数）归 [`encoding.md`](../../server/infra/encoding.md)。三篇的分工判据见 encoding.md「本知识库"格式/编码"文档的分工」。

#### row_mysql_store_col_in_innobase_format：逐列编码（写方向）

上表标注归本篇的这个函数（190 行）是"MySQL 列 → `dfield_t`"的真正编码逻辑——**读方向 `row_sel_field_store_in_mysql_format_func` 的精确镜像**（后者的完整 switch 见 [`row_search.md`](../../innodb/row_search.md)）。

**完整源码（就近注释；分支编号按源码顺序）**：

```cpp
/** Stores a non-SQL-NULL field given in the MySQL format in the InnoDB format.
 The counterpart of this function is row_sel_field_store_in_mysql_format() in
 row0sel.cc.  @return up to which byte we used buf in the conversion */
byte *row_mysql_store_col_in_innobase_format(
    dfield_t *dfield,       /* dtype 必须已由调用方设好！ */
    byte *buf,              /* 仅 DATA_INT 用；dfield 可能指向它，别提前释放 */
    bool row_format_col,    /* ★ true=行格式(record[0])；false=键格式(key buffer) */
    const byte *mysql_data, /* MySQL 列值；dfield 可能指向它，别提前释放 */
    ulint col_len,          /* MySQL 格式下的列存储长度（真 VARCHAR 时无关） */
    ulint comp)             /* 非零 = compact 行格式 */
{
  const byte *ptr = mysql_data;     // 默认零拷贝：最终 dfield 就指向它
  const dtype_t *dtype;
  ulint type;
  ulint lenlen;

  dtype = dfield_get_type(dfield);
  type = dtype->mtype;

  if (type == DATA_INT) {                              // ① 整数：小端 → 大端 + 符号翻转
    /* Store integer data in Innobase in a big-endian format, sign bit negated
    if the data is a signed integer. In MySQL, integers are stored in a
    little-endian format. */
    byte *p = buf + col_len;
    for (;;) {                                         // 从 buf 末尾往前填 = 整段反转
      p--;
      *p = *mysql_data;
      if (p == buf) break;
      mysql_data++;
    }
    if (!(dtype->prtype & DATA_UNSIGNED)) {            // ★ 仅 signed：
      *buf ^= 128;                                     //   补码 → 偏移二进制（memcmp 有序）
    }
    ptr = buf;
    buf += col_len;                                    // ★ 唯一推进 buf 的分支
  } else if (type == DATA_VARCHAR || type == DATA_VARMYSQL ||
             type == DATA_BINARY) {
    if (dtype_get_mysql_type(dtype) == DATA_MYSQL_TRUE_VARCHAR) {
      /* The length of the actual data is stored to 1 or 2 bytes at the start */
      if (row_format_col) {                            // ② 真 VARCHAR：长度头 1 或 2 字节
        if (dtype->prtype & DATA_LONG_TRUE_VARCHAR) lenlen = 2;   // 列定义长 > 255
        else                                        lenlen = 1;
      } else {
        /* In a MySQL key value, lenlen is always 2 */  // ★ 键格式恒 2
        lenlen = 2;
      }
      ptr = row_mysql_read_true_varchar(&col_len, mysql_data, lenlen);  // col_len 被真实长度覆盖
    } else {
      /* Remove trailing spaces from old style VARCHAR columns. */
      ulint mbminlen = dtype_get_mbminlen(dtype);      // ③ 老式 CHAR/VARCHAR：剥尾空格
      ptr = mysql_data;
      switch (mbminlen) {
        default: ut_error;                             // 只可能是 1/2/4
        case 4: /* space = 0x00000020 */
          col_len &= ~3;                               // 先砍到 4 的倍数（丢掉「半个」字符）
          while (col_len >= 4 && ptr[col_len-4] == 0x00 && ptr[col_len-3] == 0x00 &&
                 ptr[col_len-2] == 0x00 && ptr[col_len-1] == 0x20) col_len -= 4;
          break;
        case 2: /* space = 0x0020 */
          col_len &= ~1;                               // ★ 向下取偶（最多丢 1 个尾字节）
          while (col_len >= 2 && ptr[col_len-2] == 0x00 && ptr[col_len-1] == 0x20) col_len -= 2;
          break;
        case 1: /* space = 0x20 */
          while (col_len > 0 && ptr[col_len-1] == 0x20) col_len--;
      }
    }
  } else if (comp && type == DATA_MYSQL && dtype_get_mbminlen(dtype) == 1 &&
             dtype_get_mbmaxlen(dtype) > 1) {
    /* ... UTF-8 would otherwise normally use 3 * the string length bytes ... */
    ulint n_chars;                                     // ④ 多字节 CHAR（仅 compact）
    ut_a(!(dtype_get_len(dtype) % dtype_get_mbmaxlen(dtype)));
    n_chars = dtype_get_len(dtype) / dtype_get_mbmaxlen(dtype);   // 列定义字符数
    /* Strip space padding. */
    while (col_len > n_chars && ptr[col_len - 1] == 0x20) col_len--;   // 剥到 n_chars 字节
  } else if (!row_format_col) {                        // ⑤ ★ 键格式拦截（在 BLOB/几何之前！）
    /* if mysql data is from a MySQL key value
       since the length is always stored in 2 bytes, we need do nothing here. */
  } else if (type == DATA_BLOB) {                       // ⑥ BLOB（仅行格式可达）
    ptr = row_mysql_read_blob_ref(&col_len, mysql_data, col_len);     // 拆 [长度][8B 指针]
  } else if (DATA_GEOMETRY_MTYPE(type)) {               // ⑦ 几何（宏含 POINT/VAR_POINT/GEOMETRY）
    /* We use blob to store geometry data except DATA_POINT internally, but in
    MySQL Layer the datatype is always blob. */
    ptr = row_mysql_read_geometry(&col_len, mysql_data, col_len);
  }

  dfield_set_data(dfield, ptr, col_len);   // ★ 零拷贝：dfield->data 直接指向源缓冲
  return (buf);
}
```

下面按分支细读（展开每个分支的"为什么"与跨篇对照）。

**入口参数就藏着一个关键区分**：

```cpp
byte *row_mysql_store_col_in_innobase_format(
    dfield_t *dfield, byte *buf,
    bool row_format_col,    /* ★ true=数据来自 MySQL 行(record[0])；false=来自键值 */
    const byte *mysql_data, ulint col_len, ulint comp) {
```

注释原文：*"in MySQL, a true VARCHAR storage format differs in a row and in a key value: **in a key value the length is always stored in 2 bytes**!"* ⇒ 同一个函数靠 `row_format_col` 兼容两种输入，因为**长度头的字节数不同**。

**★ Key Format 的数据从哪来、到哪去**（`row_format_col=false` 的完整链路）：

全库只有一处传 `false`——`row_sel_convert_mysql_key_to_innobase`（`row0sel.cc:2400`）：

```cpp
buf = row_mysql_store_col_in_innobase_format(
    dfield, buf, false,          /* MySQL key value format col */
    key_ptr + data_offset, data_len, dict_table_is_comp(index->table));
```

它的调用方在 `ha_innodb.cc:10246-10267`：

```cpp
if (key_ptr != nullptr) {
  /* Convert the search key value to InnoDB format into m_prebuilt->search_tuple */
  row_sel_convert_mysql_key_to_innobase(
      m_prebuilt->search_tuple, m_prebuilt->srch_key_val1,
      m_prebuilt->srch_key_val_len, index, key_ptr, key_len);
}
if (end_range != nullptr && m_prebuilt->is_reading_range()) {
  row_sel_convert_mysql_key_to_innobase(
      m_prebuilt->m_stop_tuple, m_prebuilt->srch_key_val2,
      m_prebuilt->srch_key_val_len, index, end_range->key, end_range->length);
}
```

三点：

1. **源 = server 传下来的搜索键**：`key_ptr` 就是 `handler::index_read(key, key_len)` 等接口的 `key` 参数——由 **server 构造**的 MySQL 键值格式（如 `WHERE a = 'abc'` 里的 `'abc'`）
2. **目标缓冲 = prebuilt 预分配**：`srch_key_val1`（起始键）/ `srch_key_val2`（终止键）——又是"预计算 + 复用"，与 `mysql_template` 同一思路
3. **产物**：`search_tuple`（搜索起点）/ `m_stop_tuple`（range 上界），分别喂给游标打开的起止位置

⇒ ★ **两种格式的分工**：**Row Format 用于整行写入**（INSERT/UPDATE 的 `record[0]` → dtuple），**Key Format 用于搜索定位**（server 的 WHERE 键值 → `search_tuple`）。它们共用同一个编码函数，只因 VARCHAR 长度头规则不同才需要 `row_format_col` 区分。

**★ 两种格式的源 → 目标完整对照**（落点都是 `prebuilt` 预分配缓冲）：

| | Row Format（`true`） | Key Format（`false`） |
|---|---|---|
| **源** | `mysql_rec + mysql_col_offset`（= `TABLE::record[0]` 内该列） | `key_ptr + data_offset`（server 传下的搜索键） |
| **目标缓冲** | **`prebuilt->ins_upd_rec_buff + mysql_col_offset`** | `prebuilt->srch_key_val1` / `srch_key_val2` |
| **产物 dtuple** | `prebuilt->insert_tuple` 等 | `search_tuple` / `m_stop_tuple` |
| **用途** | 整行写入 | 搜索定位 |

Row Format 调用处（`row_mysql_convert_row_to_innobase` 循环内，`row0mysql.cc`；该函数本身的逐行剖析见 [`row_search.md`](../../innodb/row_search.md)「INSERT 方向的 record[0] → dtuple」）：

```cpp
row_mysql_store_col_in_innobase_format(
    dfield, prebuilt->ins_upd_rec_buff + templ->mysql_col_offset,   // ★ 目标缓冲
    true,   /* MySQL row format data */
    mysql_rec + templ->mysql_col_offset,                            // 源：record[0] 内该列
    templ->mysql_col_len, dict_table_is_comp(prebuilt->table));
```

两点值得注意：

1. **源与目标用同一个 `mysql_col_offset`**——`ins_upd_rec_buff` 是按 MySQL 行格式布局的**镜像缓冲**，逐列搬运时偏移一致
2. **`dfield` 可能直接指向 `buf`**（函数注释明说 *"dfield may also get a pointer to 'buf', therefore do not discard this as long as dfield is used!"*）——转换结果**零拷贝**引用这块缓冲 ⇒ **buf 活着 dfield 才有效**，这正是 prebuilt 把它预分配好的原因

**★ 关于命名**：源码注释把这两种输入叫 *"MySQL row format"* 与 *"MySQL key value"*（`row_format_col` 的注释原文）。本文统一用 **Row Format / Key Format** 指代。⚠️ 它们都是 **server 层的行字节布局**，**不要**与 InnoDB 自己的行格式（`COMPACT` / `DYNAMIC`，物理存储层）混淆——那是另一套东西。

**★ 再着重一次：写方向全程不需要 `CHARSET_INFO` 对象**。整个 `row_mysql_store_col_in_innobase_format`（`row0mysql.cc:377-567`，**191 行**）里与字符集相关的读取只有 **6 处**——`dtype_get_mbminlen` 2 处（`:482`、`:517`）、`dtype_get_mbmaxlen` 3 处（`:518`、`:544`、`:546`）、外加 `:544` 同处的 `dtype_get_len`。⇒ **"这一列是什么 collation"在写方向根本用不到**：写只管字节布局；**只有比较**（`rem0cmp.ic`）才需要经 `get_charset(prtype 里的 collation id)` 换回完整 `CHARSET_INFO`（见 [`charset.md`](../../server/infra/charset.md)）。这也是为什么 `mbminmaxlen` 是 prtype 里**唯一**下沉的字符属性（[`charset.md`](../../server/infra/charset.md)）：写要它，比较不要它。

⚠️ **一处常见的说法要纠正：写方向从不"补 0x20"，只"剥"**。本函数里所有 `0x20` 的出现都是**读取比较**（`:494-496`、`:505-506`、`:512`、`:549` 全是 `ptr[...] == 0x20` 判等后**递减长度**），没有任何一处**写入** `0x20`。真正"补空格"在读方向：`row_mysql_pad_col()`（`row0mysql.cc:340-371`，调用点 `row0sel.cc:2607`）。**"写剥读补"是一对，单看写方向会误以为它补空格。**

**① DATA_INT：小端 → 大端 + 符号位翻转**

```cpp
  if (type == DATA_INT) {
    /* Store integer data in Innobase in a big-endian format,
    sign bit negated if the data is a signed integer. In MySQL,
    integers are stored in a little-endian format. */
    byte *p = buf + col_len;
    for (;;) {                     // 逐字节倒序 = 小端转大端
      p--;
      *p = *mysql_data;
      if (p == buf) break;
      mysql_data++;
    }
    if (!(dtype->prtype & DATA_UNSIGNED)) {
      // 有符号：翻转最高字节的最高位
```

⇒ 与读方向**完全对称**：读方向是"倒序 + `^128` 还原"，写方向是"倒序 + `^0x80` 置位"。XOR 是自反的，所以两个方向用同一招——这就是 **offset binary（移码）**：把补码数轴整体平移到无符号空间，使"有符号整数的字节序比较"等价于"无符号字典序比较"（否则 B-tree 里负数会排到正数后面）。

**④ 多字节字符集的 CHAR：写方向"剥尾空格"，读方向"补回"**（★ 与"按 maxlen 补空格"的说法**相反**；源码顺序上它排在老式剥空格之后）

```cpp
} else if (comp && type == DATA_MYSQL && dtype_get_mbminlen(dtype) == 1 &&
           dtype_get_mbmaxlen(dtype) > 1) {
  /* In some cases we strip trailing spaces from UTF-8 and other
  multibyte charsets, from FIXED-length CHAR columns, to save
  space. UTF-8 would otherwise normally use 3 * the string length
  bytes to store an ASCII string! */
  ...
  /* Consider a CHAR(5) field containing the string
  ".a   " where "." denotes a 3-byte character represented
  by the bytes "$%&". After our stripping, the string will
  be stored as "$%&a " (5 bytes). The string
  ".abc " will be stored as "$%&abc" (6 bytes).

  The space padding will be restored in row0sel.cc, function
  row_sel_field_store_in_mysql_format(). */

  n_chars = dtype_get_len(dtype) / dtype_get_mbmaxlen(dtype);

  /* Strip space padding. */
  while (col_len > n_chars && ptr[col_len - 1] == 0x20) {
    col_len--;
  }
}
```

⇒ ★ **纠正一个常见误解**：InnoDB 存 CHAR **不是"按 maxlen 补空格"，恰恰相反——是剥掉尾空格省空间，读回时再补回**。

三个要点：

1. **为什么剥**：注释直说 *"UTF-8 would otherwise normally use 3 × the string length bytes to store an ASCII string"*——`CHAR(10) utf8mb4` 若补齐要占 40 字节，哪怕只存 `"abc"` 也占 40。剥掉尾空格后只占 3 字节
2. **剥到哪为止**：条件是 `col_len > n_chars`——剥到 **`n_chars`（列定义的字符数）个字节**为止（注意是**字节**不是字符）。所以 `CHAR(5)` 存 `".a   "` 剥成 5 字节 `$%&a `（**保留一个空格**），不是剥到 4 字节
3. **★ 读方向补回**：`row_sel_field_store_in_mysql_format()` 里的 `row_mysql_pad_col` 按 mbminlen 补 `0x20`（见 [`row_search.md`](../../innodb/row_search.md)）⇒ **写剥读补，是对称的一对**

⇒ **所以"InnoDB 不关心 charset"也不准确**。它确实**不解释字符语义**（不做字符集转换），但：
- 必须知道 **mbminlen / mbmaxlen**——此处用 `mbminlen == 1 && mbmaxlen > 1` 判定"是否多字节"，并用 mbmaxlen 算 `n_chars`
- 还**假设了"空格 = 0x20 字节且 1:1 对应"**（注释明写这个前提）——这正是该分支只适用于这类字符集的原因
- **比较时**还要借 server 的 `CHARSET_INFO`（见上文）

⇒ ★ **"按 maxlen 存、不足补"描述的其实是 server 层的 `TABLE::record[0]`**（定长缓冲，见「★ 为什么"每行大小不同"」），**不是** InnoDB 的 `rec_t`。两者正好相反：**server 侧定长**（浪费空间换偏移恒定），**引擎侧紧凑**（剥离省空间，代价是读回要补回）——这正是本篇那组对照的又一个实例。

**② 真 VARCHAR：长度头 1 还是 2 字节，取决于数据来自哪**

```cpp
  } else if (type == DATA_VARCHAR || type == DATA_VARMYSQL || type == DATA_BINARY) {
    if (dtype_get_mysql_type(dtype) == DATA_MYSQL_TRUE_VARCHAR) {
      if (row_format_col) {
        if (dtype->prtype & DATA_LONG_TRUE_VARCHAR) {
          lenlen = 2;              // 行格式：列定义长 → 2 字节
        } else {
          lenlen = 1;              // 行格式：列定义短 → 1 字节
        }
      } else {
        /* In a MySQL key value, lenlen is always 2 */
        lenlen = 2;                // ★ 键格式恒 2 字节
      }
      ptr = row_mysql_read_true_varchar(&col_len, mysql_data, lenlen);
```

⇒ 这就是"**键格式恒 2 字节、行格式 1 或 2 字节**"的写入侧源头，与读方向的分支是同一条规则的两端。

**③ ★ 老式 VARCHAR / CHAR：剥尾空格，按字符集最小字节数分三支**

```cpp
      ulint mbminlen = dtype_get_mbminlen(dtype);
      ptr = mysql_data;
      switch (mbminlen) {
        case 4:  /* space = 0x00000020 */
          col_len &= ~3;           // ★ 先砍到 4 的倍数（丢掉"半个"字符）
          while (col_len >= 4 && ptr[col_len-4]==0x00 && ... && ptr[col_len-1]==0x20)
            col_len -= 4;          // 剥 4 字节空格
          break;
        case 2:  /* space = 0x0020 */
          col_len &= ~1;           // ★ 砍到偶数
          while (col_len >= 2 && ptr[col_len-2]==0x00 && ptr[col_len-1]==0x20)
            col_len -= 2;          // 剥 2 字节空格
          break;
        case 1:  /* space = 0x20 */
          while (col_len > 0 && ptr[col_len-1]==0x20) col_len--;
      }
```

⇒ ★★ **读方向有一个"看似对应"的奇数长度分支**：`row_sel_field_store_in_mysql_format_func` 里有 `if (UNIV_UNLIKELY(len & 1)) *pad++ = 0x20;`（源码注释：*"A 0x20 has been stripped from the column"*）。但要**当心因果倒置**：写方向这里的 `col_len &= ~1` 是**向下取偶**（最多丢 1 个尾字节），它保证输出长度**恒为偶数**——所以读方向的 `len & 1` 分支**并非**由它触发，而是源码自称的 *"just in case"* 防御（且读方向还有 `if (pad < field_end)` 守卫才写，见 [`row_search.md`](../../innodb/row_search.md)）。两个方向要成对理解，但**别把读方向的防御当成写方向的必然产物**。

（另有一支：UTF-8 等多字节字符集的**定长 CHAR 也剥尾空格**——`mbminlen==1 && mbmaxlen>1` 时，否则 ASCII 字符串要按 3 倍长度存，太浪费。）

**⑥⑦ 剩下的两支：BLOB 与几何**

```cpp
  } else if (type == DATA_BLOB) {
    ptr = row_mysql_read_blob_ref(&col_len, mysql_data, col_len);   // BLOB：读"页内前缀 + 页外引用"
  } else if (DATA_GEOMETRY_MTYPE(type)) {
    ptr = row_mysql_read_geometry(&col_len, mysql_data, col_len);   // 几何：server 侧一律按 BLOB 传
  }
```

两支都**不是字节搬运而是"解释"**：`DATA_BLOB` 从 MySQL 格式的引用缓冲中拆出真正的 `ptr/len`——**这个缓冲的布局是 `[1–4 字节长度][8 字节指针]`**（写出侧 `row0mysql.cc:221-232`，读出侧 `read_blob_ref` 只做 `mach_read_from_n_little_endian(ref, col_len-8)` 读长度 + `memcpy` 末 8 字节取指针）；几何类型在 server 层一律当 BLOB 传（注释："in MySQL Layer the datatype is always blob"），引擎侧再按几何格式解释。

> ⚠️ **别把另外两个"20/768"混进来**：`BTR_EXTERN_FIELD_REF_SIZE = 20`（`page0size.h:39`，InnoDB **内部**外存字段引用）与 `REC_ANTELOPE_MAX_INDEX_COL_LEN = 768`（`rem0types.h:68`，索引前缀上限）都是**引擎内部**概念，**不是**本函数入参的布局——本函数处理的只是上面那个 `[长度][8B 指针]` 的 server 侧缓冲。

**⑤ ★ 键格式的"空操作"分支（排在 ④ 之后、BLOB/几何之前！）**

⚠️ **先纠正一个容易想当然的说法**：不能说"前面几支都是行格式路径、键格式一律走这条拦截"。源码顺序是 ① → ② → ③ → **④** → **⑤（本支）** → ⑥ → ⑦，而 **④（多字节 CHAR 剥尾空格，`:517`）的条件里没有 `row_format_col`**——键格式下 compact + utf8mb4 的 CHAR 仍会**先命中 ④ 的剥空格逻辑**，只有没被前四支命中的类型才落到这条空操作分支。拦截分支的准确位置是**排在 BLOB / 几何之前**：

```cpp
  } else if (!row_format_col) {
    /* if mysql data is from a MySQL key value
       since the length is always stored in 2 bytes,
       we need do nothing here. */
  }                                       // ★ 空操作：ptr 保持 = mysql_data，col_len 不变
  } else if (type == DATA_BLOB) { ... }   // ← 键格式永远到不了这里
```

⇒ **★ 键格式下 BLOB / 几何不做"拆引用"**：键里没有 20 字节外存引用字段，只有**前缀字节**，所以引擎把它当定长数据原样收下（与 `row_sel_convert_mysql_key_to_innobase` 注释 "store the column value like it would be a fixed char field" 一致）。这是 `row_format_col` 除"长度头 2 字节"之外的**第二处实质作用**——**分支顺序本身就是语义**（它必须排在 ⑤ 之前才成立）。

**尾部：没有 `default`，其余类型是"零拷贝引用"**

函数的 if / else-if 链**没有外层 `default`**——凡未命中的类型（`DATA_CHAR`、`DATA_FIXBINARY`、`DATA_FLOAT`、`DATA_DOUBLE`、`DATA_DECIMAL`、单字节字符集的 `DATA_MYSQL`、`DATA_SYS` 等）**一律落到函数末尾**：

```cpp
  dfield_set_data(dfield, ptr, col_len);   // ★ 让 dfield 的 data 指针「指向」源缓冲——不是 memcpy！
  return (buf);
```

⇒ **零拷贝引用**：`dfield->data` 直接指向 `mysql_data`（或 `DATA_INT` 用的 `buf`）。这正是参数注释反复警告"**`mysql_rec` 在 row 使用期间不能释放**"的原因——`dtuple` 并不拥有这些字节。

**返回值的唯一性**：只有 `DATA_INT` 分支会推进 `buf`（`buf += col_len`）并返回它——因为只有整数需要一块临时缓冲做"倒序 + 符号翻转"。其余分支原样返回入参 `buf`。

**三个辅助函数的真身**（都在 `row0mysql.cc`，与读方向成对；下列为**精简重排**，原文分别在 `:187-204` / `:235-244` / `:303-334`）：

```cpp
// 读真 VARCHAR 的长度头（写方向的对应物是 row_mysql_store_true_var_len）
const byte *row_mysql_read_true_varchar(ulint *len, const byte *field, ulint lenlen) {
  if (lenlen == 2) { *len = mach_read_from_2_little_endian(field); return (field + 2); }
  ut_a(lenlen == 1);
  *len = mach_read_from_1(field);  return (field + 1);
}

// BLOB 引用：mysql_data 是「[col_len-8 字节长度][8 字节指针]」，拆出真 ptr/len
const byte *row_mysql_read_blob_ref(ulint *len, const byte *ref, ulint col_len) {
  byte *data;
  *len = mach_read_from_n_little_endian(ref, col_len - 8);   // 长度
  memcpy(&data, ref + col_len - 8, sizeof data);             // 指针
  return (data);
}

// ★ 几何的读取器：布局与 BLOB 引用完全同构（同为 [长度][8B 指针]）
static const byte *row_mysql_read_geometry(ulint *len, const byte *ref, ulint col_len) {
  byte *data;
  *len = mach_read_from_n_little_endian(ref, col_len - 8);
  memcpy(&data, ref + col_len - 8, sizeof data);
  /* DBUG_EXECUTE_IF("row_print_geometry_data")：调试名下把二进制几何解析成
     WKT 文本打印（Geometry::construct + as_wkt + ER_IB_MSG_971），此处略 */
  return (data);
}
```

★ 注意 `DATA_GEOMETRY_MTYPE(type)` 是宏，覆盖**三种 mtype**：`DATA_POINT`(15) / `DATA_VAR_POINT`(16) / `DATA_GEOMETRY`(14)——几何相关全走同一支。**几何与 BLOB 的读取器代码几乎一样**（都拆 `[长度][8B 指针]`），差别只在 `DATA_POINT` 这种定长类型在键里要特殊取长（见键读节）。

> ⚠️ **改一处要改两处的坑**：同文件里还有一份**内联复制**——`row_mysql_to_innobase()`（intrinsic 系统表专用路径）把"整数倒序 + `^= 128`、真 VARCHAR 读头、BLOB/几何拆引用"这套逻辑**手写了一遍**（不调用本函数）。改编码规则时必须同步改这里，否则 intrinsic 表行为不一致。

**调用点**（`row_format_col=true` 的 row 路径有多处）：`row_mysql_convert_row_to_innobase`（INSERT/UPDATE 主路径）、`innodb_fill_old_vcol_val`（虚拟列 old value）、`ha_innodb.cc` 的 build-row / 虚拟列处、`handler0alter.cc`（在线 DDL）、`dict0dd.cc`（DD 表插入）。`row_format_col=false` 的 key 路径**全库唯一**：`row_sel_convert_mysql_key_to_innobase`。

⇒ 与另两篇的分工：本函数讲"**每列值怎么变成 dfield**"（值级编码）；字节怎么摆进 REC 归 [`record.md`](../../innodb/physical/record.md)；`mach_*` 写整数的原语归 [`encoding.md`](../../server/infra/encoding.md)。

#### 转换模板：mysql_row_templ_t 与 row_prebuilt_t

前面说"转换是纯开销"，InnoDB 的应对是**把每列的转换信息预先算成模板**（`mysql_row_templ_t`），存在按表缓存的预建结构 `row_prebuilt_t` 里。模板把"查 `CHARSET_INFO` → 取 `mbminlen`/`mbmaxlen` → 判断 true VARCHAR 长度前缀 → 算 NULL 位位置"一次性预计算，于是热路径变成"读模板字段 → 直接拷贝"。

关键的几点（与行表示相关）：

- 模板里存了**三种 `rec_field_no`**（当前索引 / 聚簇索引 / ICP 索引）——同一列在不同索引里序号不同，回表与索引扫描用不同的号
- `row_prebuilt_t::template_type` 决定"取整行还是部分列"：`ROW_MYSQL_WHOLE_ROW` 是整行，否则只按 `read_set` 建模板——这就是覆盖索引能省转换开销的原因
- 模板 + `read_set` 位图配合，是"权衡三"的最终答案：不是消除转换，而是**只转必要的列、且每列信息预先算好**

> ★ **这两个结构的完整剖析（字段逐项、`build_template` 的建立过程、覆盖索引示例、与 handler 的生命周期）属于 handler 层主题，见 [`../../server/handler.md`](../../server/handler.md) 的「row_prebuilt_t」与「mysql_row_templ_t」两节**，本篇只在行表示链路里交代它们的角色。

#### 读方向：InnoDB → MySQL 的对称链路

> ★ **跨篇分工（先看这段）**：**行读 switch**（`row_sel_field_store_in_mysql_format_func`）的逐行剖析在 [`innodb/row_search.md`](../../innodb/row_search.md) 的 `### ★ row_sel_field_store_in_mysql_format_func：字段级转换的 switch 全貌`（16 个 case 全部原样收录）——**这部分以那里为准**。而**键读**（`row_sel_convert_mysql_key_to_innobase`）的**完整实现落点在本篇下面的「键的读方向」节**——row_search.md 里同名节只是流程节选（用 `...` 省略了若干段），**不能当权威版本**。
>
> **本篇（data_repr）只负责"表示"视角**：同一列在两个方向的字节形态如何**互为镜像**——写方向逐分支剖析（见上节），读方向给主循环真代码 + 逐分支对照表 + `^= 128` 的原理。**两者合起来才是完整实现**：row_search.md 讲"流程怎么走"，本篇讲"字节怎么对称、为什么这样对称"。

**函数分工速查**（避免"看到签名找不到剖析"）：

| 函数 | 逐行剖析落点 | 本篇的视角 |
|---|---|---|
| `row_mysql_store_col_in_innobase_format`（**行写**） | **本篇**（190 行逐分支） | 完整 |
| `row_sel_field_store_in_mysql_format_func`（**行读**） | `row_search.md`「字段级转换的 switch 全貌」 | 与写方向的**逐分支对照表** |
| `row_sel_store_mysql_rec`（行读总入口） | `row_search.md`「取行总入口」+「clust_templ_for_sec」完整链路 | 主循环骨架 + 格式视角要点 |
| `row_sel_store_mysql_field`（行读中间层） | `row_search.md`「单字段落盘」（在 clust_templ_for_sec 链路内，**完整逐行**） | 五个要点（不重复代码） |
| `row_sel_convert_mysql_key_to_innobase`（**键读**） | **本篇**（键的读方向，完整代码）+ `row_search.md` 同名节仅流程节选 | 完整 |
| `row_mysql_convert_row_to_innobase`（行写总入口） | `row_search.md`「INSERT 方向的 record[0] → dtuple」 | 引用语义（dfield 指向 `record[0]`） |

> **为什么"行写"的完整剖析留本篇、而"行读"指向 row_search.md？** 判据是**"编码与流程的耦合度"**：
> - **读方向的编码是读流程的必经环节**——`row_sel_store_mysql_rec` 就是 `row_search_mvcc` PHASE 4 的"取行的最后一步"，row_search.md 讲流程时躲不开它，于是顺手讲透（并对写方向只做"逆操作"式指认）。
> - **写方向的编码是纯转换函数**——它被 `row_mysql_convert_row_to_innobase` 循环调用，而写路径文档 [`row_ops.md`](../../innodb/row_ops.md) 只讲 INSERT/UPDATE **状态机**、完全不涉及字节编码（`dtuple_t`/`dtype` 在它那里零命中）。**所以"写方向的字节怎么编"只有本篇承载**——搬走它，知识库就再也没有一处完整描述写方向编码。
> - 本库的组织原则是**"表示归表示、流程归流程"**：本篇给两个方向的**并排对照**（这才是"字节怎么对称"的完整视图），流程细节各归其流程文档。

写方向只有 `row_mysql_store_col_in_innobase_format` 一个函数，读方向则有**三层结构**：

```
row_sel_store_mysql_rec          ← 遍历模板，逐列分派（含虚拟列/instant 特殊逻辑）
  └─ row_sel_store_mysql_field   ← 解析 rec 上的字段位置、处理 off-page BLOB/页内拷贝
       └─ row_sel_field_store_in_mysql_format_func   ← 真正的逐类型字节转换（写方向函数的逆）
```

**`row_sel_store_mysql_rec()` 主循环（真实代码）**：

```cpp
bool row_sel_store_mysql_rec(byte *mysql_rec, row_prebuilt_t *prebuilt, const rec_t *rec,
                             const dtuple_t *vrow, bool rec_clust, const dict_index_t *rec_index,
                             const dict_index_t *prebuilt_index, const ulint *offsets,
                             bool clust_templ_for_sec, lob::undo_vers_t *lob_undo,
                             mem_heap_t *&blob_heap);
// 入口先 mem_heap_empty(blob_heap)：上一行的 blob 指针全部作废

for (ulint i = 0; i < prebuilt->n_template; i++) {
  const auto templ = &prebuilt->mysql_template[i];

  /* Skip multi-value columns; since they can not be explicitly
  requested by the query, they may only be here in scenarios
  where all index columns are included routinely, like these:
  1. Index-only scan (covering index): all index fields are stored
     regardless of whether they are requested; for multi-values they
     need not be stored, as they may never be requested.
  2. Cross-partition index scan (index merge): not needed since
     multi-values do not introduce ordering. */
  if (templ->is_multi_val) {
    /* Multi-value columns are always virtual */
    ut_ad(templ->is_virtual);
    continue;                       // ★ 多值列必然虚拟，且永不显式请求
  }

  if (templ->is_virtual && rec_index->is_clustered()) {
    // 虚拟列：数据不在 rec 上——从 vrow（dtuple）的 v_fields 独立数组取；
    // 仅覆盖扫描 / read_just_key / m_read_virtual_key 且 rec_clust 时才输出
    ...（从 dtuple_get_nth_v_field(vrow, col->v_pos) 取 dfield，
         SQL NULL 时置 NULL 位 + 用 default_rec 填充值区）
  }

  ulint field_no = rec_clust ? templ->clust_rec_field_no : templ->rec_field_no;

  /* We should never deliver column prefixes to MySQL,
  except for evaluating innobase_index_cond() or
  row_search_end_range_check(). */
  ut_ad(rec_index->get_field(field_no)->prefix_len == 0);   // ★ 前缀不外泄的断言

  if (clust_templ_for_sec) {
    // 用主键模板扫二级索引：把"聚簇 field_no"映射回二级索引里的位置
    const dict_field_t *field = rec_index->get_field(field_no);
    const dict_col_t *col = field->col;
    it = std::find(template_col.begin(), template_col.end(), col);
    if (it == template_col.end()) continue;        // 该列不在本二级索引里
    sec_field_no = it - template_col.begin();
  }

  if (!row_sel_store_mysql_field(mysql_rec, prebuilt, rec, rec_index,
                                 prebuilt_index, offsets, field_no, templ,
                                 sec_field_no, lob_undo, blob_heap)) {
    return false;      // off-page 字段没读完（如 LOB undo 场景），本次失败
  }
}

/* 结尾：FTS 的 doc_id（仅当表有 FTS 索引） */
if (dict_table_has_fts_index(rec_index->table)) {
  if ((rec_index->is_clustered() && !clust_templ_for_sec) ||
      prebuilt->fts_doc_id_in_read_set)
    prebuilt->fts_doc_id = fts_get_doc_id_from_rec(rec_index->table, rec, rec_index, nullptr);
}
```

四个结构要点（都藏在注释里）：① **多值列必定跳过**（它永远虚拟、且永不被显式请求——覆盖索引扫描会"例行"包含所有索引列，跳过它不影响结果）；② **`prefix_len == 0` 断言**——列前缀只允许在引擎内部（ICP / end-range）使用，绝不外泄给 MySQL；③ **`clust_templ_for_sec` 的列映射**——用主键模板扫二级索引时，要把聚簇索引的 `field_no` 映射回二级索引里的序号；④ 结尾单独处理 **FTS 的 `doc_id`**。

> ★ **上面是主循环骨架（本篇的"格式视角"）**——它的**完整逐行**（三种分发的全部细节：多值列 / 虚拟列从 `vrow->dfield` 取值 / 普通列，`template_col` 映射的构造，以及 `std::find` 反查的完整代码）见 [`../../innodb/row_search.md`](../../innodb/row_search.md) 的「`row_sel_store_mysql_rec`：取行总入口」。**代码不两处全文重复**，本篇只留骨架用于"读↔写对称"的对照。

**中间层 `row_sel_store_mysql_field`：五个要点**（完整逐行见 [`../../innodb/row_search.md`](../../innodb/row_search.md) 的「`row_sel_store_mysql_field`：单字段落盘」——含完整代码、三条取数路径的确切行号、借出/归还机制）：

① **页外/页内是两种 heap 策略**——BLOB 复用 `blob_heap`（随行生命周期），其它用临时 heap 并**当场释放**；② **页内 BLOB 必须 `mem_heap_dup`**——因为转换函数"只存指针"，页 latch 一放数据就失效；③ **NULL 分支提前 return**（刚置上 NULL 位，不能再清）；④ **`clust_templ_for_sec` 的坐标切换/还原**（取数据用二级索引号、写 dfield 用聚簇号）；⑤ **外存分支传 `ULINT_UNDEFINED`**（外存必是聚簇列，不需要二级映射）。

> ★ **本篇不重复贴该函数代码**——它的完整实现（含 `[[nodiscard]]`、外部字段未写完的 `return false` 条件、`lob_undo->apply`）已收入 row_search.md 的 clust_templ_for_sec 链路一节。这里只保留"格式视角"的要点。

**★ 返回值语义 + 调用方的三种处理**：`row_sel_store_mysql_rec` 返回 `false` **只有一个原因**——某列的**外存 BLOB 尚未写完**（`btr_rec_copy_externally_stored_field` 返回 nullptr）。三种调用方的反应完全不同：

| 调用方 | 对 `false` 的处理 |
|---|---|
| **ICP**（`row_search_idx_cond_check`） | 当作**不匹配**（`ICP_NO_MATCH`），省掉回表 |
| **AHI 捷径**（`row_sel_try_search_shortcut_for_mysql`） | 放弃捷径、**回退正常流程** |
| **PHASE 4 主循环** | `goto next_rec` 跳到下一行 |

⇒ 同一个"失败"信号，在三条路径上是三种语义——**这是这套接口最容易误读的地方**。

**`row_mysql_pad_col` 的完整实现**（读方向补空格，写方向不调用）：

```cpp
void row_mysql_pad_col(ulint mbminlen, byte *pad, ulint len) {
  switch (UNIV_EXPECT(mbminlen, 1)) {      // UNIV_EXPECT：1 字节字符集最常见
    default: ut_error;                     // mbminlen 只可能是 1/2/4
    case 1: memset(pad, 0x20, len); break;                          // space = 0x20
    case 2: pad_end = pad + len; ut_a(!(len % 2));                  // space = 0x0020
            while (pad < pad_end) { *pad++ = 0x00; *pad++ = 0x20; } break;
    case 4: pad_end = pad + len; ut_a(!(len % 4));                  // space = 0x00000020
            while (pad < pad_end) { *pad++ = 0x00; *pad++ = 0x00;
                                    *pad++ = 0x00; *pad++ = 0x20; } break;
  }
}
```

两个 `ut_a(!(len % 2))` / `ut_a(!(len % 4))` 正是**读方向必须先做"半个空格修复"（`len & 1` 补回一个 0x20）的原因**——`pad_col` 只补对齐后的整字符。

⚠️ **纠正一个常见说法**：8.0.39 **没有** `templ->mysql_col_offset == 0` / `mysql_null_bit_mask == 0` 的"跳过列"逻辑——**列筛选发生在模板构建期**（`build_template_needs_field` 按 `read_set`/覆盖索引判断，不需要的列根本不生成模板项），store 循环天然只处理需要的列。

**NULL 列的语义（重要的对称点）**：NULL 的"值区"必须**写成 MySQL 默认值**：

```cpp
// row_sel_store_mysql_field
/* MySQL assumes that the field for an SQL NULL value is set to the default value */
memcpy(mysql_rec + templ->mysql_col_offset,
       prebuilt->default_rec + templ->mysql_col_offset, templ->mysql_col_len);
```

（`prebuilt->default_rec = table->s->default_values`）——所以 SQL 层读 NULL 列时看到的字节是"表定义的默认值"，而不是未初始化数据。

**逐类型转换与写方向的对照**（`row_sel_field_store_in_mysql_format_func` vs `row_mysql_store_col_in_innobase_format`）：

| 分支 | 读方向（InnoDB → MySQL） | 写方向（MySQL → InnoDB） |
|---|---|---|
| DATA_INT | **从尾向前**逐字节反转（大端→小端）+ **仅 signed** 时 `dest[len-1] ^= 128` | **同样从尾向前**反转 + **仅 signed** 时 `*buf ^= 128` |
| true VARCHAR | 写 1/2 字节小端长度头 + memcpy 数据 | 跳过 1/2 字节长度头 |
| 老式 CHAR/VARCHAR | memcpy 后**补尾空格**（`row_mysql_pad_col`，按 mbminlen 写 0x20/0x0020/0x00000020） | **剥**尾空格 |
| DATA_MYSQL（多字节 CHAR） | memcpy + 必要时 `memset(dest+len, 0x20, ...)` 补空格 | 剥空格到 n 字符 |
| DATA_BLOB | `row_mysql_store_blob_ref`（写长度+指针） | `row_mysql_read_blob_ref`（解引用） |
| GEOMETRY | `row_mysql_store_geometry` | `row_mysql_read_geometry` |
| 其它定长 | 一律 `memcpy(dest, data, len)` | —— |

**写方向剥空格 ↔ 读方向补空格**是严格成对的设计（写方向的源码注释就写着 "padding will be restored in row0sel.cc"）。

**并排代码：`DATA_INT` 分支的镜像**（"字节怎么对称"最典型的例子）：

```cpp
// ── 写方向（row0mysql.cc）：MySQL 小端 → InnoDB 大端 ──
if (type == DATA_INT) {
  /* Store integer data in Innobase in a big-endian format,
  sign bit negated if the data is a signed integer. In MySQL,
  integers are stored in a little-endian format. */
  byte *p = buf + col_len;
  for (;;) {              // ★ 从尾向前，逐字节反转
    p--;  *p = *mysql_data;
    if (p == buf) break;
    mysql_data++;
  }
  if (!(dtype->prtype & DATA_UNSIGNED)) *buf ^= 128;   // ★ 仅 signed
  ptr = buf;  buf += col_len;
}

// ── 读方向（row0sel.cc）：InnoDB 大端 → MySQL 小端 ──
case DATA_INT:
  /* Convert integer data from Innobase to a little-endian
  format, sign bit restored to normal */
  ptr = dest + len;
  for (;;) {              // ★ 同样从尾向前
    ptr--;  *ptr = *data;
    if (ptr == dest) break;
    data++;
  }
  if (!templ->is_unsigned) dest[len - 1] ^= 128;       // ★ 仅 signed
  break;
```

⚠️ **两处与直觉不同**：① 两个方向的循环**都是"从尾向前"**（不是"一个正序一个倒序"）——反转动作本身是对称的；② `^= 128` **只在 signed 时做**——无符号整数不需要数轴平移。

**第二个并排示例：字符串的"长度头 + 补空格"**（读方向）：

```cpp
// ── 读方向（row0sel.cc）：true VARCHAR / 老式 CHAR ──
case DATA_VARCHAR: case DATA_VARMYSQL: case DATA_BINARY:
  field_end = dest + mysql_col_len;

  if (templ->mysql_type == DATA_MYSQL_TRUE_VARCHAR) {
    /* This is a >= 5.0.3 type true VARCHAR. Store the length of the
    data to the first byte or the first two bytes of dest. */
    dest = row_mysql_store_true_var_len(dest, len, templ->mysql_length_bytes);
    /* Copy the actual data. Leave the rest of the buffer uninitialized. */
    memcpy(dest, data, len);      // ★ 长度头（1/2 字节）+ 数据；其余不初始化
    break;
  }

  ut_memcpy(dest, data, len);     // 老式 CHAR：先单纯拷数据
  pad = dest + len;               // ★ 然后补尾空格——写方向剥的在这里补回
  ut_ad(templ->mbminlen <= templ->mbmaxlen);

  switch (templ->mbminlen) {
    case 4:                       // UTF-32
      ut_a(!(len & 3));           // 引擎不会剥掉半个 UTF-32 字符
      break;
    case 2:                       // UCS2 / UTF-16：空格是 0x0020（两字节）
      if (UNIV_UNLIKELY(len & 1)) {          // 奇数长度 = 0x20 被剥掉了半个
        if (pad < field_end) *pad++ = 0x20;  // ★ 先补回那半个空格字节
      }
  }
  row_mysql_pad_col(templ->mbminlen, pad, field_end - pad);  // 其余按 mbminlen 填满
  break;
```

四处细节：① **true VARCHAR 分支只写"长度头 + 数据"，其余缓冲不初始化**（注释明说 "Leave the rest of the buffer uninitialized"——长度头已自描述，不需要补空格）；② 老式 CHAR 才是"**写方向剥、读方向补**"的落点；③ `mbminlen == 2` 的**"半个空格"修复**（UCS2 的 `0x20` 被剥掉一个字节时，先补回一个 `0x20`）；④ `row_mysql_pad_col` 按 `mbminlen` 分别写 `0x20` / `0x0020` / `0x00000020`。

**`^= 128` 的本质：偏移二进制编码（offset binary / excess-N）**

补码数轴上负数最高位是 1、正数最高位是 0——**按无符号字节 memcmp 时，负数会被当成"更大的数"，顺序全错**：

```
补码:        -2^31 ... -1  |  0  |  1 ... 2^31-1
最高位:         1      1   |  0  |  0      0
按无符号比:    ← 负数看起来最大（顺序错误）
```

把最高位翻转（`^= 128`），等价于**给数轴加 2^31 的偏移**，映射到无符号空间：

```
XOR 0x80:       0 ... 0    |  1  |  1 ... 1
无符号值:     最小 ←————————————————→ 最大     ⇒ memcmp 序 = 数值序 ✓
```

这就是**偏移二进制**：写方向 XOR 一次（进入"可比较空间"）、读方向 XOR 一次（还原补码）——**两次 XOR 抵消，但中间态能被 B-tree 直接 memcmp**。与 DECIMAL 的"首字节 `^= 0x80`"、时间类型的"加 `*_OFS` 偏移常量"是**同一个技巧的三种形态**（见前面「各数据类型的 record 编码」）。

**读方向同样完全不使用 `CHARSET_INFO`**：`templ->charset` 这个字段在 8.0.39 全仓库**只有赋值、没有任何读取**——补空格只用字节级的 `mbminlen/mbmaxlen`。**两个方向都只碰字节布局，字符语义全部在 server 层**（这是"引擎不碰字符集"原则的完整闭环）。

**BLOB 的三条读路径**（`row_sel_store_mysql_field`）：

1. **off-page**（`rec_offs_nth_extern` 判定）：`lob::btr_rec_copy_externally_stored_field` 把"局部前缀 + 外部页"**完整复制到 heap**——返回 nullptr 即"没复制完"（上层 `return false`）。InnoDB 侧引用是 **20 字节**（page_no/offset/version/len + owner 等标志位），与 MySQL 侧 `(长度, 8 字节指针)` 完全是两码事。
2. **页内**（普通 BLOB 或 `DATA_POINT`）：`mem_heap_dup(blob_heap, data, len)`——注释明确：converter 只存指针，mtr 提交/页 latch 释放后原数据即失效，**必须先拷出来**。
3. **组装**：DATA_BLOB 分支 → `row_mysql_store_blob_ref(dest, mysql_col_len, data, len)`——MySQL 格式 = 前 `col_len-8` 字节小端长度 + 8 字节指针。`Field_blob::set_ptr` 消费的正是这个布局。

**★ instant DDL：旧记录里没有的列怎么读出来**——默认值是"凭空出现"的，但**不在模板层**，而在 rec 层：

```
1. 构造 offsets：rec_get_instant_offset 对"物理上不存在的 instant 列"
   查 index->get_nth_default()：NULL → offs|REC_OFFS_SQL_NULL，否则 offs|REC_OFFS_DEFAULT
2. 解码：rec_get_nth_field_offs_low 把 REC_OFFS_DEFAULT 翻成 UNIV_SQL_ADD_COL_DEFAULT
3. ★核心：rec_get_nth_field_instant —— 若长度是 ADD_COL_DEFAULT 哨兵，
   直接返回 index->get_nth_default(n, len)      ← 默认值（在 dict 元数据里）
   （DROP COL 哨兵则返回 nullptr + len=0）
4. dict_index_t::get_nth_default 从 col->instant_default->value/len 取字节
5. row_sel_store_mysql_field 没有 instant 专属分支——它统一调 rec_get_nth_field_instant，
   默认值为 SQL NULL 时走"default_rec 填充"路径（与普通 NULL 列同）
```

⇒ instant 列的默认值同时存在于两处：**InnoDB dict 元数据**（非 NULL 默认值）与 **MySQL 层 `default_rec`**（NULL 语义）——这就是"不重写旧记录也能读出新增列"的全部秘密。

**`row_mysql_read_blob_ref` / `row_mysql_read_geometry` 的方向澄清**（易标反）：

```cpp
// 名字叫 "read"，但它读的是「MySQL 记录里的字段」——实际用在 MySQL→InnoDB 方向
const byte *row_mysql_read_blob_ref(ulint *len, const byte *ref, ulint col_len) {
  *len = mach_read_from_n_little_endian(ref, col_len - 8);   // 解出长度
  memcpy(&data, ref + col_len - 8, sizeof data);              // 解出指针
  return data;
}
```

即 `read_*` = **从 MySQL 格式读出（引用）**、`store_*` = **写进 MySQL 格式**——两个函数都在 `row0mysql.cc`，服务于"MySQL → InnoDB"这一侧；InnoDB → MySQL 方向用的是 `row_mysql_store_blob_ref`。**"read" 指的是从 MySQL 记录里 read，不是"读方向"**。

#### 键写（server 侧）：`record[0]` → key buffer

前面讲的 **Key Format**（`row_format_col=false`）是"**引擎收到键之后**"怎么用。但键是**谁造出来的**？答案是 server 的 **`key_copy()`**——它把行值/搜索值按 `KEY` 描述写成"MySQL 键缓冲"，再交给引擎。

```
key_copy（server，sql/key.cc）           →  MySQL 键缓冲（"Key Format"）
  → ha_innobase::index_read(key_ptr, ...)
    → row_sel_convert_mysql_key_to_innobase（引擎，键读）  →  dtuple → B-tree 搜索
```

**`key_copy()`（`sql/key.cc:136`）**：

```cpp
void key_copy(uchar *to_key, const uchar *from_record, const KEY *key_info,
              uint key_length) {
  uint length;
  KEY_PART_INFO *key_part;

  if (key_length == 0) key_length = key_info->key_length;  // 0 → 整键长
  for (key_part = key_info->key_part; (int)key_length > 0; key_part++) {
    if (key_part->null_bit) {                              // ① 可空段：1 字节 NULL 标记
      bool key_is_null = from_record[key_part->null_offset] & key_part->null_bit;
      *to_key++ = (key_is_null ? 1 : 0);                   //    从「行内」NULL 位图取位
      key_length--;
    }
    if (key_part->key_part_flag & HA_BLOB_PART ||          // ② 变长段：预留 2 字节长度头
        key_part->key_part_flag & HA_VAR_LENGTH_PART) {
      key_length -= HA_KEY_BLOB_LENGTH;                     //    =2
      length = min<uint>(key_length, key_part->length);
      key_part->field->get_key_image(to_key, length, Field::itRAW);  // ★ 写 2B 头 + 数据
      to_key += HA_KEY_BLOB_LENGTH;
    } else {                                                // ③ 定长段
      length = min<uint>(key_length, key_part->length);
      Field *field = key_part->field;
      const CHARSET_INFO *cs = field->charset();
      size_t bytes = field->get_key_image(to_key, length, Field::itRAW);
      if (bytes < length)                                   //    CHAR 等不足则补空格
        cs->cset->fill(cs, (char *)to_key + bytes, length - bytes, ' ');
    }
    to_key += length;                                       // ④ 两分支共用：前进数据长度
    key_length -= length;
  }
}
```

**★ 两处易错**：① **它用的是 `Field::get_key_image()`，不是 `make_sort_key()`**——这两个是**不同的函数**：`get_key_image` 保留"段内小端/原样"（引擎的比较器知道段类型，按 collation 比）；`make_sort_key` 是把值**变换成 memcmp 即有序**的编码（整数翻符号位、double 重排 IEEE 序、字符串 `strnxfrm`）——**两者绝不能混用**。⚠️ 顺带纠正一个过时印象：8.0.39 里 **`Field::make_sort_key()` 已不再服务 filesort**——`field.h:1332-1334` 注释原文 *"This is now only used by replication"*（filesort 现在基于 `Item` 自造排序键；复制侧调用见 `rpl_write_set_handler.cc`）。② `key_length` 是**递减计数器**（调用方传要复制的字节数，传 0 才用整键长）。

**段布局**（逐段紧密拼接，★ **没有独立 NULL 位图**）：

```
key buffer = Σ 每段 ( [NULL 1B 若可空] [长度 2B 若 BLOB/VARCHAR] [数据 length B] [补齐] )

实例：KEY(a INT NULL, b VARCHAR(10))      // b 的 length = 40（10 × mbmaxlen）
  偏移 0    : a 的 NULL 字节（0x00/0x01）
  偏移 1-4  : a 的值（4B 小端原样；引擎收到后才转大端 + 翻符号位）
  偏移 5-6  : b 的 2 字节长度头（小端）
  偏移 7-46 : b 的数据（不足补 0x00）
  总长 47   = (1+4) + (2+40) = key_length
```

**`KEY_PART_INFO::store_length` 是"每段步长"**（主路径由 `sql/table.cc` 的 `init_from_field` 算出——**但不是唯一计算点**：DD 路径 `sql/dd_table_share.cc:1217-1218` 直接赋值、老 frm 路径 `table.cc:1599`、运行期分区裁剪还会按同公式复算 `sql/sql_partition.cc:6118-6125`）：

```cpp
length = (uint16)field->key_length();
store_length = length;
if (field->is_nullable()) store_length += HA_KEY_NULL_LENGTH;          // +1
if (field->type() == MYSQL_TYPE_BLOB || field->real_type() == MYSQL_TYPE_VARCHAR ||
    field->type() == MYSQL_TYPE_GEOMETRY)
  store_length += HA_KEY_BLOB_LENGTH;                                  // +2
```

`KEY::key_length` = 各段 `store_length` 之和。

**键格式的真正产出者是 `Field::get_key_image()`**（`key_copy` 逐段调它，`Field::itRAW` 模式）——三类实现的差异与**源码出处**：

| 类型 | 行为 | 源码 |
|---|---|---|
| CHAR（`Field_string`） | memcpy + 不足**补 `pad_char`**（空格），**无长度头** | `sql/field.cc:6529-6539` |
| VARCHAR（`Field_varstring`） | **恒 2 字节小端长度头** + 数据；不足补 **0x00**（为 `opt_range` 的 memcmp 服务） | `sql/field.cc:6837-6846`（注释原文 "When VARCHAR is stored in a key … it's always stored with a 2 byte prefix. (Just like blob keys)"） |
| BLOB（`Field_blob`） | 恒 2 字节小端长度头 + 数据 | `sql/field.cc:7240-7250` |

> ★ **一处易漏的边角**：`length_bytes` 通常按 `field_length < 256 ? 1 : 2` 算，但**长 VARCHAR 上的前缀键段**可能出现 `field_length == 1` 而 `length_bytes == 2` 的特例（`field.cc:6552-6555`）——"键里恒 2 字节"是源码**显式处理**的，不只是约定。
> 常量归属：`HA_KEY_NULL_LENGTH = 1` / `HA_KEY_BLOB_LENGTH = 2` 定义在 **`sql/handler.h:610-611`**（不在 `my_base.h`）。

**与 InnoDB REC 的差异**（为什么引擎必须"再转一次"）：

| 维度 | server 键缓冲 | InnoDB REC |
|---|---|---|
| NULL | **每可空段 1 个独立字节**，紧贴该段 | **头部位图**（1 位/列） |
| 变长长度 | VARCHAR/BLOB **恒 2 字节小端头** | 1/2 字节长度字节；**前缀列连长度字节都没有** |
| 整数端序 | 保留 **server 小端** | **大端 + 符号位翻转** |
| 字符补齐 | `get_key_image` 补空格/0 到 `length` | 按前缀截断，**不补** |
| 用途 | **搜索定位 / 去重 / 系统表查找** | **持久存储** |

**反向的 `key_restore()`（完整源码，就近注释）**：把键缓冲还原成 `record[0]`。⚠️ **用途常被说错**——它**不是**"覆盖索引（`HA_EXTRA_KEYREAD`）"的实现：8.0.39 的调用点只有三处（**全在 server 层**）——**分区裁剪**（`sql/sql_partition.cc:3424/3465/3507`）、**松散索引扫描**（`sql/range_optimizer/group_index_skip_scan.cc:370/414/454`）、**权限表读取**（`sql/auth/sql_user_table.cc:1160/1191`）；`storage/innobase` 内 **0 处调用**（`storage/` 下仅 NDB 引擎有一处：`storage/ndb/plugin/ha_ndbcluster.cc:13767`）。覆盖索引只填键不回表，靠的是 `m_prebuilt->read_just_key` 机制（`ha_innodb.cc`），**与 `key_restore` 无关**。

```cpp
void key_restore(uchar *to_record, const uchar *from_key, const KEY *key_info,
                 uint key_length) {
  uint length;
  KEY_PART_INFO *key_part;

  if (key_length == 0) key_length = key_info->key_length;  // 与 key_copy 对称的默认值
  for (key_part = key_info->key_part; (int)key_length > 0; key_part++) {
    uchar used_uneven_bits = 0;             // ★ BIT 散位是否占用了本段首字节

    if (key_part->null_bit) {               // ① 还原 NULL 位到「行内」NULL 位图
      if (*from_key++)
        to_record[key_part->null_offset] |=  key_part->null_bit;
      else
        to_record[key_part->null_offset] &= ~key_part->null_bit;
      key_length--;
    }

    if (key_part->type == HA_KEYTYPE_BIT) { // ② BIT：把「散位」写回 NULL 字节区
      Field_bit *field = (Field_bit *)(key_part->field);
      if (field->bit_len) {
        uchar bits =
            *(from_key + key_part->length - field->pack_length_in_rec() - 1);
        set_rec_bits(bits,
                     to_record + key_part->null_offset + (key_part->null_bit == 128),
                     field->bit_ofs, field->bit_len);
        /* we have now used the byte with 'uneven' bits */
        used_uneven_bits = 1;
      }
    }

    if (key_part->key_part_flag & HA_BLOB_PART) {         // ③ BLOB：★ 源码自认的死分支
      /*
        This in fact never happens, as we have only partial BLOB keys yet
        anyway, so it's difficult to find any reason to restore the part of a
        record. Maybe this branch is to be removed, but now we have to ignore
        GCov compaining.
      */
      uint blob_length = uint2korr(from_key);             // 读 2B 长度头
      Field_blob *field = (Field_blob *)key_part->field;
      from_key += HA_KEY_BLOB_LENGTH;
      key_length -= HA_KEY_BLOB_LENGTH;
      field->set_ptr_offset(to_record - field->table->record[0],
                            (ulong)blob_length, from_key); // 指针偏移式回填
      length = key_part->length;
    } else if (key_part->key_part_flag & HA_VAR_LENGTH_PART) {  // ④ VARCHAR：set_key_image
      Field *field = key_part->field;
      my_bitmap_map *old_map;
      ptrdiff_t ptrdiff = to_record - field->table->record[0];
      field->move_field_offset(ptrdiff);                  // ★ 把 Field 临时「搬到」目标行
      key_length -= HA_KEY_BLOB_LENGTH;
      length = min<uint>(key_length, key_part->length);
      old_map = dbug_tmp_use_all_columns(field->table, field->table->write_set);
      field->set_key_image(from_key, length);             // 经 set_key_image 回填
      dbug_tmp_restore_column_map(field->table->write_set, old_map);
      from_key += HA_KEY_BLOB_LENGTH;
      field->move_field_offset(-ptrdiff);                 // 搬回来
    } else {                                              // ⑤ 定长：纯拷贝
      length = min<uint>(key_length, key_part->length);
      /* skip the byte with 'uneven' bits, if used */
      memcpy(to_record + key_part->offset, from_key + used_uneven_bits,
             (size_t)length - used_uneven_bits);          // BIT 时跳过散位字节
    }

    from_key += length;
    key_length -= length;
  }
}
```

与 `key_copy` **方向相反、段步长一致**，但**不是"对称的 5 分支"**——`key_copy` 只有 3 个分支（NULL / 变长 / 定长），下面这些是 `key_restore` **独有**的（因为只有"从键还原整行"才需要）：**BIT 散位写回 NULL 字节区**（代码 ②，`set_rec_bits`）、**BLOB 用 `set_ptr_offset` 指针回填**（代码 ③）、**VARCHAR 用 `set_key_image` + `move_field_offset` 把 Field 临时"搬"到目标行**（代码 ④）——后者是为了避免在 `set_key_image` 里硬编码行偏移，**同分支内还临时打开 `write_set`**（`dbug_tmp_use_all_columns`）。★ 其中**代码 ③ 的 BLOB 分支源码注释自认是死分支**（"This in fact never happens … Maybe this branch is to be removed"）——因为目前只有"部分 BLOB 键"，没有还原整块 BLOB 的场景，保留只是为了让 GCov 闭嘴。

**调用点**（`key_copy` 是 server 通用的"行值 → 键字节"工具）：自增取最后值（`handler::get_auto_increment` → `HA_READ_PREFIX_LAST`，传**前缀长度**）、半连接/去重迭代器（`RemoveDuplicatesOnIndexIterator` 等）、range/松散扫描（`group_index_skip_scan`）、**DD 原子键**（`dd/impl/raw/object_keys.cc`）、权限/复制系统表。

> ⚠️ **勘误（重要）**：源码 `row_sel_convert_mysql_key_to_innobase` 里有一句注释 "For documentation of the key value storage format in MySQL, see `ha_innobase::store_key_val_for_row()` in ha_innodb.cc"——**该函数在 8.0.39 已不存在**（全仓仅这句陈旧注释）。它的职能已迁到 **server 的 `key_copy`**（产键）+ 引擎的 `row_sel_convert_mysql_key_to_innobase`（转键）。按旧文档去找 `store_key_val_for_row` 会扑空。

#### 键的读方向：`row_sel_convert_mysql_key_to_innobase()`

前面写方向讲了 Key Format（`row_format_col=false`）——那是"**构建索引时把列值转成键**"。还有一条对称的路径：**用户查询的键值 → InnoDB 可比较的 dtuple**（range 扫描、唯一查找用）：

**完整源码（就近注释）**：

```cpp
void row_sel_convert_mysql_key_to_innobase(dtuple_t *tuple, byte *buf,
                                           ulint buf_len, dict_index_t *index,
                                           const byte *key_ptr, ulint key_len) {
  byte *original_buf = buf;
  const byte *original_key_ptr = key_ptr;
  dict_field_t *field;
  dfield_t *dfield;
  ulint data_offset, data_len, data_field_len;
  bool is_null;
  const byte *key_end;
  ulint n_fields = 0;

  /* For documentation of the key value storage format in MySQL, see
  ha_innobase::store_key_val_for_row() in ha_innodb.cc. */  // ⚠️ 陈旧注释：该函数已移除（见上文勘误）
  key_end = key_ptr + key_len;

  /* Permit us to access any field in the tuple (ULINT_MAX): */
  dtuple_set_n_fields(tuple, ULINT_MAX);     // 先放开上限，逐段填完再收紧
  dfield = dtuple_get_nth_field(tuple, 0);
  field = index->get_field(0);

  if (UNIV_UNLIKELY(dfield_get_type(dfield)->mtype == DATA_SYS)) {
    /* A special case: ... the first and the only ordering column is ROW_ID
    which InnoDB stored to the key_ptr buffer. */          // 无主键表的 GEN_CLUST_INDEX
    ut_a(key_len == DATA_ROW_ID_LEN);
    dfield_set_data(dfield, key_ptr, DATA_ROW_ID_LEN);
    dtuple_set_n_fields(tuple, 1);
    return;
  }

  while (key_ptr < key_end) {                // ★ 逐段解析：段数由 key_len 耗尽决定
    ulint type = dfield_get_type(dfield)->mtype;
    ut_a(field->col->mtype == type);         // 模板类型与索引列类型必须一致

    data_offset = 0;
    is_null = false;

    if (!(dfield_get_type(dfield)->prtype & DATA_NOT_NULL)) {
      /* The first byte in the field tells if this is an SQL NULL value */
      data_offset = 1;                       // ① 可空段：首字节是 NULL 标记
      if (*key_ptr != 0) { dfield_set_null(dfield); is_null = true; }
    }

    /* Calculate data length and data field total length */
    if (DATA_LARGE_MTYPE(type) || DATA_GEOMETRY_MTYPE(type)) {
      if (dict_index_is_spatial(index)) {    // ② R-tree：整个 WKB 都是键
        ut_ad(DATA_GEOMETRY_MTYPE(type));
        data_len = key_len;
        data_field_len = data_offset + data_len;
      } else {
        /* The key field is a column prefix of a BLOB or TEXT, except
        DATA_POINT of GEOMETRY. */
        ut_a(field->prefix_len > 0 || DATA_POINT_MTYPE(type));
        /* MySQL stores the actual data length to the first 2 bytes after the
        optional SQL NULL marker byte. The storage format is little-endian ...
        In UTF-8, MySQL seems to reserve field->prefix_len bytes for storing
        this field in the key value buffer, even though the actual value only
        takes data len bytes from the start. */
        data_len = key_ptr[data_offset] + 256 * key_ptr[data_offset + 1];  // 2B 小端长度
        data_field_len = data_offset + 2 +
            (type == DATA_POINT ? DATA_POINT_LEN : field->prefix_len);
        data_offset += 2;                    // 跳过长度的 2 字节
      }
    } else if (field->prefix_len > 0) {      // ③ 普通前缀索引
      /* Looks like MySQL pads unused end bytes in the prefix with space.
      Therefore, also in UTF-8, it is ok to compare with a prefix containing
      full prefix_len bytes ... */           // 空格补齐 → 前缀可整段参与比较
      data_len = field->prefix_len;
      data_field_len = data_offset + data_len;
    } else {                                 // ④ 定长：类型里已写死
      data_len = dfield_get_type(dfield)->len;
      data_field_len = data_offset + data_len;
    }

    if ((dtype_get_mysql_type(dfield_get_type(dfield)) == DATA_MYSQL_TRUE_VARCHAR) &&
        (type != DATA_INT)) {
      /* In a MySQL key value format, a true VARCHAR is always preceded by
      2 bytes of a length field. ... We added the check != DATA_INT to make
      sure we do not treat MySQL ENUM or SET as a true VARCHAR! */
      data_len += 2;                         // ★ true VARCHAR 在键里恒定 2 字节头
      data_field_len += 2;                   //   （≠DATA_INT：避开 ENUM/SET）
    }

    /* Storing may use at most data_len bytes of buf */
    if (UNIV_LIKELY(!is_null)) {
      buf = row_mysql_store_col_in_innobase_format(
          dfield, buf, false, /* MySQL key value format col */
          key_ptr + data_offset, data_len, dict_table_is_comp(index->table));
      ut_a(buf <= original_buf + buf_len);   // ⑤ ★ 复用写方向函数（row_format_col=false）
    }

    key_ptr += data_field_len;               // ⑥ 前进该段

    if (UNIV_UNLIKELY(key_ptr > key_end)) {  // 键尾是「部分前缀」：告警 + 截断
      /* The last field in key was not a complete key field but a prefix of it.
      Print a warning about this! HA_READ_PREFIX_LAST does not currently work
      in InnoDB with partial-field key value prefixes. ... */
      ib::warn(ER_IB_MSG_1028)
          << "Using a partial-field key prefix in search, index " << index->name
          << " of table " << index->table->name << ". Last data field length "
          << data_field_len << " bytes, key ptr now exceeds key end by "
          << (key_ptr - key_end) << " bytes. Key value in the MySQL format:";
      ut_print_buf(stderr, original_key_ptr, key_len);
      putc('\n', stderr);
      if (!is_null) {
        ulint len = dfield_get_len(dfield);
        dfield_set_len(dfield, len - (ulint)(key_ptr - key_end));   // 截掉超出部分
      }
      ut_d(ut_error);
    }

    n_fields++;
    field++;
    dfield++;
  }

  ut_a(buf <= original_buf + buf_len);
  /* We set the length of tuple to n_fields: we assume that the memory area
  allocated for it is big enough (usually bigger than n_fields). */
  dtuple_set_n_fields(tuple, n_fields);      // ★ 收尾：字段数 = 实际解析出的段数
}
```

下面按要点细读：

```cpp
// row0sel.cc —— MySQL key buffer → dtuple
bool row_sel_convert_mysql_key_to_innobase(dtuple_t *tuple, byte *buf, ulint buf_len,
                                           const dict_index_t *index,
                                           const byte *key_ptr, ulint key_len, ...);
```

流程要点：

1. **dtype 由调用方预设**：`init_search_tuples_types()` 让 `n_fields = index->n_fields` 并 `dict_index_copy_types` 从索引复制类型——转换函数本身不管类型，只填值。
2. **特例：第 0 字段是 `DATA_SYS`**（无主键表的自动 `DB_ROW_ID`）→ `dfield_set_data(key_ptr, DATA_ROW_ID_LEN)`、`n_fields = 1` 直接返回。
3. **逐字段推导长度**（这是与写方向最大的不同——**MySQL 的键字节流是"无类型前缀"的**，长度得算出来）——核心代码：

```cpp
// row0sel.cc —— 长度推导的三个分支
if (DATA_LARGE_MTYPE(type) || DATA_GEOMETRY_MTYPE(type)) {
  if (dict_index_is_spatial(index)) {
    // R-tree：整个 WKB 都是键，长度就是 key_len
    data_len = key_len;
    data_field_len = data_offset + data_len;
  } else {
    // BLOB/TEXT 前缀：★ 实际长度存在键的前 2 字节（小端）
    data_len = key_ptr[data_offset] + 256 * key_ptr[data_offset + 1];
    data_field_len = data_offset + 2 +
        // ★ POINT 取 DATA_POINT_LEN 的理由（源码注释）："For POINT of GEOMETRY,
        //   which has no prefix because it's now a fixed length type in InnoDB,
        //   we have to get DATA_POINT_LEN bytes, which is original prefix length
        //   of POINT." —— 即 POINT 在 InnoDB 已是定长类型、没有 prefix_len 可用
        (type == DATA_POINT ? DATA_POINT_LEN : field->prefix_len);
    data_offset += 2;      // 跳过长度的 2 字节
  }
} else if (field->prefix_len > 0) {
  // 普通前缀索引：MySQL 会用空格把前缀补齐到 prefix_len 字节
  data_len = field->prefix_len;
  data_field_len = data_offset + data_len;
} else {
  data_len = dfield_get_type(dfield)->len;   // 定长：类型里已写死
  data_field_len = data_offset + data_len;
}

if ((dtype_get_mysql_type(dfield_get_type(dfield)) == DATA_MYSQL_TRUE_VARCHAR) &&
    (type != DATA_INT)) {
  // ★ true VARCHAR 在键里恒定 2 字节长度头（与行格式的 1/2 字节不同！）
  //   注释里的 != DATA_INT 检查：防止把 MySQL ENUM/SET 误当 true VARCHAR
  data_len += 2;
  data_field_len += 2;
}
```

三个与"行方向"不同的点：① **BLOB 的实际长度**从键的前 2 字节读（行格式里 BLOB 是"长度 + 8 字节指针"，键里是"长度 + 前缀数据"）；② **前缀索引的两种补齐**：一般前缀由 MySQL **用空格补齐未用满的尾部**（注释 "MySQL pads unused end bytes in the prefix with space"——所以前缀可整段参与比较，不需要按字符数折算），而 **`LIKE 'abc%'` 的上界用 `0xff` 补齐**（注释 "If the prefix is used as the upper end of a LIKE 'abc%' query, then MySQL pads the end with chars 0xff"）——正因这两种补齐约定，源码 NOTE 断言"搜索里**不应出现**部分字段前缀"；③ **true VARCHAR 的 2 字节头是铁律**，且要显式排除 ENUM/SET。
4. **★ 复用写方向的函数**：`row_mysql_store_col_in_innobase_format(dfield, buf, /*row_format_col=*/false, key_ptr + data_offset, data_len, comp)`——**"读方向的键转换"调的是"写方向的函数"**（因为数据流向都是 MySQL 字节 → dtuple，只是格式参数不同）。

调用点：`search_tuple`（`ha_innobase::index_read`，`ha_innodb.cc:10250`；服务各类索引查找而不只是"唯一精确查找"）、`m_stop_tuple`（`end_range` 边界，同函数 `:10265`）、**`records_in_range`**（`ha_innodb.cc:16751-16767`，这里才用 `dtuple_create + dict_index_copy_types` 直接建类型）、分区表版 `ha_innopart::records_in_range`（`ha_innopart.cc:3316-3333`）。⚠️ 不是 `read_range_first`——它只把请求转发给 `handler::read_range_first`，既不调用本函数也不建类型。

⇒ "键"这条链路到此闭环：**`key_copy`（server 产键）→ `row_sel_convert_mysql_key_to_innobase`（引擎转键）**。但"键"还有**第二个来源**——引擎自己从聚簇行建索引项（插入/更新时）：`row_build_index_entry_low`（`row0row.cc`，从聚簇 `dtuple_t` 直接建索引 entry，**根本不经过 MySQL 键格式**）。它已在 [`../../index/record_format.md`](../../index/record_format.md)「`row_build_index_entry`」详剖，这里只点出与键读的**本质差别**：建键的输入值**已经在 InnoDB 格式**（聚簇行），所以不需要 `row_format_col=false` 那条路径。

**四条链路的完整矩阵**（比"2×2"更准确——"键写"其实有两支）：

| 链路 | 函数 | 输入 | 输出 | 长度头 |
|---|---|---|---|---|
| **行写** | `row_mysql_convert_row_to_innobase` → `row_mysql_store_col_in_innobase_format` | `record[0]`（Row Format） | 聚簇行 `dtuple_t` | 依列定义 1/2 字节 |
| **键写 · server** | **`key_copy`** | `record[0]` + `KEY` | **MySQL 键缓冲** | **恒 2 字节** |
| **键写 · 引擎** | `row_build_index_entry_low` | 聚簇 `dtuple_t` | 索引 `entry` dtuple | InnoDB 原生 |
| **键读** | `row_sel_convert_mysql_key_to_innobase` | MySQL 键缓冲 | `search_tuple`/`m_stop_tuple` | 读 2 字节 → InnoDB |
| **行读** | `row_sel_store_mysql_rec` 三层 | REC | `record[0]` | 写 1/2 字节（重建） |

**★ 双建键路径分野**（新增发现，容易混淆）：引擎侧"把值填进索引 entry"有**两个函数**，分工按行来源：

| | INSERT 路径 | UPDATE / purge / online-DDL 路径 |
|---|---|---|
| 函数 | **`row_ins_index_entry_set_vals`**（`row0ins.cc`） | `row_build_index_entry_low`（`row0row.cc`） |
| 行从哪来 | `record[0]`（**全在内存**） | 聚簇记录 / undo（可能**外置**） |
| 外置处理 | **无**——断言 `!dfield_is_ext`，只做"前缀截断 + spatial MBR + 指针复制" | **有**——`row_ext_lookup` 取回外置 BLOB；REDUNDANT/COMPACT 下剥掉 20 字节引用、留 768 字节本地前缀 |
| entry 模板 | `ins_node_create_entry_list` **一次性建模板**（每索引一个），每行 `set_vals` 只填值 | 每次 `_low` 现建 |

⇒ 判据：**行已经在内存里（INSERT）走轻量的 `set_vals`；行要从存储/undo 读出来（UPDATE/purge）才走带 ext/768 逻辑的 `_low`**。

> **键读的逐行剖析就在上面「键的读方向」节**（完整代码，含 `LARGE`/`GEOMETRY` 长度推导、越界前缀的 `ER_IB_MSG_1028` 处理）——[`../../innodb/row_search.md`](../../innodb/row_search.md) 的同名节是**流程视角的节选**（用 `...` 省略若干段），可配合阅读但**不作权威版本**；键格式的原始说明（`store_key_val_for_row`）见 `ha_innodb.cc`。

### 六、端到端：一条 UPDATE 走完所有表示

把前面的零件串起来，看一条 UPDATE 如何依次经过每一种表示（假设 `utf8mb4` 表，`c2 VARCHAR(50)`）：

```sql
UPDATE t SET c2 = 'x' WHERE c1 = 1;
```

```
① 表达式求值（server）
   WHERE c1 = 1 → Item 树求值 → Item_field::val_int()
                              → Field_long::val_int()（读 TABLE->record[0] 里的 c1 偏移）
   新值 'x'     → Item_string（自带 DTCollation）

② 定位行
   handler::ha_index_read / rnd_next
     → InnoDB 用 row_prebuilt_t 的模板把 rec_t 字段直接拷进 record[0]
     → （旧行备份到 record[1]）

③ 写新值到 server 行缓冲
   Field_varstring::store(str, len, cs)
     → 若 Item 的 collation 与列不同，先经 String::copy 转换（见 charset.md）
     → 按 mysql_row_templ_t 的 mysql_col_offset 写进 record[0] 的 c2 位置
        （true VARCHAR 还要按 mysql_length_bytes 写 1 或 2 字节长度前缀）

④ 跨层边界
   handler::ha_update_row(old_row = record[1], new_row = record[0])
     → 引擎只拿到两个 uchar*，靠 row_prebuilt_t + dict_table_t 解释

⑤ 构造更新向量 upd_t（InnoDB）
   逐列比对 record[0] 与 record[1]，差异列生成 upd_field_t：
     upd_field_t { field_no, new_val: dfield_t（含 dtype_t + 数据指针 + len）, ... }
   → upd_t 同时持有 dict_table_t* 与 TABLE*（虚拟列计算需要 MySQL 侧）
   → 只记录"变化了的列"，未变列不进向量（省 undo 与索引维护）

⑥ 写 undo
   按 upd_t 的逆（旧值）构造 undo 记录，供回滚与 MVCC 读旧版本

⑦ 更新聚簇索引记录
   按 upd_t 生成新的 dtuple_t（n_fields_cmp 决定比较用几列）
     → rec_convert_dtuple_to_rec（按 record.md 的 COMPACT 布局序列化）
     → 写入页

⑧ 维护二级索引
   若 c2 上有索引：构造索引 dtuple_t → 比较 → 插入新索引项 + 标记旧项删除
   （多值索引此处会把 multi_value_data 的 N 个值展开成 N 条索引记录）
```

**这条链路说明了什么**：

- **同一份数据被表示了四次**：`Item`（表达式值）→ `TABLE::record`（server 字节行）→ `upd_t` 里的 `dfield_t`（更新向量）→ `rec_t`（页内字节）
- **每次跨越边界都有一次转换**，而模板（`mysql_row_templ_t`）与位图（`read_set`）是让这些转换尽可能便宜的两个手段
- **更新向量是"增量表示"**：只描述变化，不是整行——这与 `dtuple_t`（整行表示）形成对照，两者在 UPDATE 路径上配合使用（`upd_t` 描述变化、`dtuple_t` 承载结果）

**对照：INSERT 没有"增量"概念，是一条更短的链**：

```
① Item 求值 → ② Field::store 逐列写进 record[0]
③ handler::ha_write_row(record[0])
④ InnoDB 逐列 row_mysql_store_col_in_innobase_format → 构造 dtuple（整行，无增量）
⑤ 聚簇索引：rec_convert_dtuple_to_rec → 写页
⑥ 每个二级索引：按 index 重排 dtuple（字段顺序不同！）→ 再序列化一次
```

INSERT 与 UPDATE 的关键差异在 ⑥：**同一行要为每个索引各序列化一次，且每次字段顺序不同**——这就是理论基础里"被否决方案 A"所说的"引擎要为同一行生成多条顺序不同的索引记录"的现场。而 UPDATE 有 `upd_t` 做增量，只对变化的列维护索引（不变的索引列不必动）。

### 七、零转换对撞：两个 `*_with_match` 比较器（`rem0cmp.cc`）

比较器有两个：**一侧是 dtuple、另一侧是 rec** 用（一）；**两侧都是 rec** 用（二）。两者同构，共用 `cmp_data` 原语与 `matched_fields` 语义，但有六处实质差异（见（二））。

#### （一）dtuple ↔ rec：`cmp_dtuple_rec_with_match_low`（601-693，92 行）

★ **这是「比较不物化」真正的落地处**——前面反复提到的"三种表示如何对撞"，最终都收在这 92 行里。贴完整函数，关键行就近注释：

```cpp
int cmp_dtuple_rec_with_match_low(const dtuple_t *dtuple, const rec_t *rec,
                                  const dict_index_t *index,
                                  const ulint *offsets, ulint n_cmp,
                                  ulint *matched_fields) {
  // ★ matched_fields 是 in/out 参数（本函数最反直觉的一点）：
  //   入＝从第几个字段开始比（调用方已知前 k 个相等时可跳过）；出＝匹配到第几个字段
  //   ⇒ 下面循环起点是 *matched_fields，不是 0
  ut_ad(dtuple_check_typed(dtuple));               // dtuple 必须带类型：比较要用 mtype/prtype
  ut_ad(rec_offs_validate(rec, index, offsets));   // offsets 必须是"为这个 rec + 这个 index 算的"
  ut_ad(n_cmp > 0);
  ut_ad(*matched_fields == DISABLE_MIN_REC_FLAG_CHECK || *matched_fields <= n_cmp);
  ut_ad(n_cmp <= dtuple_get_n_fields(dtuple));

  // ★ MIN_REC_FLAG：B-tree 非叶子页的"最左节点指针"是虚拟下界哨兵，行里没有真实键值
  if (*matched_fields == 0) {
    ulint rec_info = rec_get_info_bits(rec, rec_offs_comp(offsets));
    ulint tup_info = dtuple_get_info_bits(dtuple);

    /* The leftmost node pointer record is defined as
    smaller than any other node pointer, independent of
    any ASC/DESC flags. It is an "infimum node pointer". */
    // ⇒ 哨兵的"最小"是结构赋予的，不是排序出来的 ⇒ 必须独立于字段比较先行判定
    if (rec_info & REC_INFO_MIN_REC_FLAG) {
      // rec 是哨兵：tuple 也是哨兵→相等(0)；tuple 是真实键→比哨兵大(1)
      return (!(tup_info & REC_INFO_MIN_REC_FLAG));
    } else if (tup_info & REC_INFO_MIN_REC_FLAG) {
      return (-1);                                 // tuple 是哨兵（下界）→ 小于任何真实 rec
    }
  } else if (*matched_fields == DISABLE_MIN_REC_FLAG_CHECK) {
    /* Disable the left most node check. */
    // ★ 同一个变量兼作"起始下标"与"开关"：传它进来＝跳过 MIN_REC 检查
    //   ⇒ 这里复位成 0，好让下面的循环从头比
    *matched_fields = 0;
  }

  /* Compare fields in a loop. */
  for (auto i = *matched_fields; i < n_cmp; ++i) {
    // ── dtuple 侧：数组下标直接取，O(1)，无需任何解析
    const auto dtuple_field = dtuple_get_nth_field(dtuple, i);
    const auto dtuple_b_ptr = static_cast<const byte *>(dfield_get_data(dtuple_field));
    const auto type = dfield_get_type(dtuple_field);
    auto dtuple_f_len = dfield_get_len(dtuple_field);

    /* We should never compare against an externally
    stored field.  Only clustered index records can
    contain externally stored fields, and the first fields
    (primary key fields) should already differ. */
    // ★ 这两个断言把"比较绝不会走到外置 / default 字段"这个不变量显式化：
    //   外置字段行内只剩 20B 指针、没有数据可比；default 字段（instant ADD COLUMN）行里可能根本没存值
    ut_ad(!rec_offs_nth_extern(index, offsets, i));
    /* So does the field with default value */
    ut_ad(!rec_offs_nth_default(index, offsets, i));

    ulint rec_f_len;
    // ── rec 侧：靠 offsets 定位 ⇒ offsets 那份"说明书"的作用就体现在这一行
    const auto rec_b_ptr = rec_get_nth_field(index, rec, offsets, i, &rec_f_len);

    ut_ad(!dfield_is_ext(dtuple_field));

    int ret{};

    // ★ 多值索引特例：一个文档对应多个索引条目 ⇒ 比较是"集合包含"判断，不是单值比较
    if (dfield_is_multi_value(dtuple_field) &&
        (dtuple_f_len == UNIV_MULTI_VALUE_ARRAY_MARKER ||
         dtuple_f_len == UNIV_NO_INDEX_VALUE)) {
      /* If it's the value parsed from the array, or NULL, then
      the calculation can be done in a normal way in the else branch */
      ut_ad(index->is_multi_value());
      if (dtuple_f_len == UNIV_NO_INDEX_VALUE) {
        ret = 1;                                   // 该文档在这个字段上无值 → 排到后面
      } else {
        // dtuple 侧存的是"整个数组的 marker"，不是某个单值
        // ⇒ 比较方向反过来：问"rec 侧这个单值是否在数组里"？在→0（相等），不在→1
        multi_value_data *mv_data =
            static_cast<multi_value_data *>(dtuple_field->data);
        ret = mv_data->has(type, rec_b_ptr, rec_f_len) ? 0 : 1;
      }
    } else {
      /* For now, change buffering is only supported on
      indexes with ascending order on the columns. */
      // ★ 普通路径：两个裸字节指针直接喂进去，两边都不物化——"零转换对撞"就在这两行
      //   第三个参数 is_ascending ⇒ 降序索引是"比较时取反"，不是"存储时编码取反"（见下）
      ret = cmp_data(type->mtype, type->prtype,
                     dict_index_is_ibuf(index) || index->get_field(i)->is_ascending,
                     dtuple_b_ptr, dtuple_f_len, rec_b_ptr, rec_f_len);
    }

    if (ret) {
      *matched_fields = i;                         // 前 i 个字段相等，第 i 个分出胜负
      return (ret);
    }
  }

  /* If we ran out of fields, dtuple was equal to rec up to the common fields */
  *matched_fields = n_cmp;                         // n_cmp 个字段全相等
  return (0);
}
```

**三个必须单独讲的设计**（因果链压不进单行注释）：

**① 为什么 MIN_REC 判定必须放在字段比较之前**

B-tree 非叶子页最左那条记录是**虚拟下界**——它不存真实键值，只是个指向最左子页的占位符。拿它的"垃圾键值"参与字段比较，结果无意义。而它的"最小"是结构赋予的，跟列是 ASC 还是 DESC 无关（注释明说 *independent of any ASC/DESC flags*）⇒ 只能单独判、且必须先判。

**② "绝不会走到外置字段"为什么不变量成立**

只有聚簇索引记录会有外置（off-page）字段，而外置的通常是长 VARCHAR/BLOB——**不会是主键前几列**。比较按字段顺序走，主键列必定先分出胜负就 `return` 了 ⇒ 永远到不了外置字段。这就是那两行敢写成 `ut_ad` 的理由。它反过来也**约束了调用方**：不能拿这个函数去比外置字段（比之前必须先确保比较只在前缀列上进行）。

**③ 降序索引：比较时取反，不是存储时编码取反**

`cmp_data` 收到 `is_ascending=false` 时在内部翻转返回值（`rem0cmp.cc:786/793`：`ret = is_ascending ? -1 : 1`）。⇒ 降序列在磁盘上的**字节编码与升序完全相同**，只是比较结果取反。好处是同一份数据能同时支撑升序/降序索引，不必改存储；代价是每次比较多一次分支。（`index/types.md`「降序索引」一节印证同一结论。）

⇒ **`matched_fields` 这个出参是全局最有用的副产品**：ICP 的前缀匹配判定、B-tree 下降时选子页、range 边界判断，都靠"匹配到第几个字段"这个信息，一次比较顺带算出来，不必再比一遍。

#### （二）rec ↔ rec：`cmp_rec_rec_with_match`（`rem0cmp.cc:989-1102`，113 行）

上面是 **dtuple 对 rec**（跨层、dtuple 侧免解析）；这个是 **rec 对 rec**（同层、**两侧**都要靠 offsets 解析）。两者同构、共用 `cmp_data` 与 `matched_fields` 语义，但有**六处实质差异**——对照着看才不会混：

```cpp
int cmp_rec_rec_with_match(const rec_t *rec1, const rec_t *rec2,
                           const ulint *offsets1, const ulint *offsets2,
                           const dict_index_t *index,
                           bool spatial_index_non_leaf, bool nulls_unequal,
                           ulint *matched_fields, bool cmp_btree_recs) {
  ut_ad(rec_offs_validate(rec1, index, offsets1));
  ut_ad(rec_offs_validate(rec2, index, offsets2));
  ut_ad(rec_offs_comp(offsets1) == rec_offs_comp(offsets2));  // 两条记录必须同为 COMPACT 或同为 REDUNDANT

  const auto comp = rec_offs_comp(offsets1);
  const auto rec1_n_fields = rec_offs_n_fields(offsets1);
  const auto rec2_n_fields = rec_offs_n_fields(offsets2);

  // ★ 差异①：matched_fields 不是 in/out 参数，总是从头比
  //   ⇒ cmp_dtuple_* 能借"已知相等的前缀"跳过前 k 个字段，这里没这个优化
  *matched_fields = 0;

  /* NOTE: This is an optimisation where we're comparing two B-tree records
  during index validation. */
  // ★ 差异②：MIN_REC 只在 cmp_btree_recs（索引校验）时才处理
  //   ⇒ 正常 B-tree 操作不会拿"预定义最小记录"来 rec-rec 比，只有 validate 全表扫才会撞上
  if (cmp_btree_recs) {
    if (rec_get_info_bits(rec1, comp) & REC_INFO_MIN_REC_FLAG) {
      ut_ad(!(rec_get_info_bits(rec2, comp) & REC_INFO_MIN_REC_FLAG));
      return (-1);
    } else if (rec_get_info_bits(rec2, comp) & REC_INFO_MIN_REC_FLAG) {
      return (1);
    }
  }

  ulint i;

  // ★ 差异③：上界取两者较小值——两条记录字段数可以不同（instant ADD COLUMN 前后同存于一页）
  for (i = 0; i < rec1_n_fields && i < rec2_n_fields; ++i) {
    /* If this is node-ptr records then avoid comparing node-ptr
    field. Only key field needs to be compared. */
    // ★ 差异④（最关键）：比满 n_unique 个字段就停 ⇒ "绝不比较 node pointer 的 child page no"
    //   ⇒ 这正是 dict_index_build_node_ptr 里 n_fields_cmp = n_unique 的对应实现，
    //     也解释了 btr 里判定"键相同"的阈值为什么是 rec_offs_n_fields(offsets) - 1
    if (i == dict_index_get_n_unique_in_tree(index)) {
      *matched_fields = i;
      return (0);
    }

    ulint mtype, prtype;
    bool is_asc;

    if (dict_index_is_ibuf(index)) {
      mtype = DATA_BINARY; prtype = 0; is_asc = true;      // ibuf 树：全部按裸二进制比
    } else if ((i == 1) && spatial_index_non_leaf) {
      // 空间索引非叶页只含 MBR + 子页号两个字段，第 1 个是 DATA_SYS_CHILD
      mtype = DATA_SYS_CHILD; prtype = 0; is_asc = true;
    } else {
      auto col = index->get_col(i);
      const auto field = index->get_field(i);
      mtype = col->mtype; prtype = col->prtype; is_asc = field->is_ascending;
    }

    if (i == 0 && dict_index_is_spatial(index)) {
      prtype |= DATA_GIS_MBR;                              // 空间索引首字段按 MBR 比
    }

    /* We should never encounter an externally stored field. ... */
    ut_ad(!rec_offs_nth_extern(index, offsets1, i));
    ut_ad(!rec_offs_nth_extern(index, offsets2, i));

    ulint r1_len;
    const auto r1 = rec_get_nth_field_instant(rec1, offsets1, i, index, &r1_len);
    ulint r2_len;
    const auto r2 = rec_get_nth_field_instant(rec2, offsets2, i, index, &r2_len);

    // ★ 差异⑤：nulls_unequal —— 两边都是 NULL 时算"等"还是"不等"
    //   ⇒ 统计 distinct 值时这是个语义开关（stats 传 stats_null_not_equal）
    if (nulls_unequal && r1_len == UNIV_SQL_NULL && r2_len == UNIV_SQL_NULL) {
      *matched_fields = i;
      return (-1);
    }

    auto ret = cmp_data(mtype, prtype, is_asc, r1, r1_len, r2, r2_len);

    if (ret != 0) { *matched_fields = i; return (ret); }
  }

  /* If we ran out of fields, rec1 was equal to rec2 up to the common fields */
  *matched_fields = i;
  return (0);
}
```

**差异⑥（上面注释里放不下的）**：`cmp_dtuple_rec_with_match_low` 的 dtuple 侧是**数组下标直接取** `fields[i]`，O(1) 且免解析；这里**两侧都要**走 `rec_get_nth_field_instant` 按 offsets 定位 ⇒ 单次比较成本更高。所以它**不用于搜索热路径**（那里用 dtuple vs rec），而用在**统计**（相邻记录比，算 distinct 值）与**索引校验**这类"解析本来就避免不了"的场景。

**`matched_fields` 的五种用法**（定义在本节，其余用法按归属散见各篇）：

| # | 用法 | 语义 | 剖析在哪 |
|---|---|---|---|
| ① | in/out 参数 | 入＝从第几个字段开始比，出＝匹配到第几个 | 本节 |
| ② | **跨层公共前缀复用** | `min(low_match, up_match)` 作为下一轮比较起点，跳过已确认相等的前缀——B-tree 搜索省比较次数的核心技巧 | [`btr.md`](../../index/btr.md)「页内二分」 |
| ③ | 撞 MIN_REC 哨兵的兜底 | `!cmp && !cur_matched_fields` → 强制置为全字段数才能落位（呼应上面 MIN_REC 段） | [`btr.md`](../../index/btr.md)「页内二分」 |
| ④ | **前缀增量计数** | 相邻两条记录比一次 ⇒ 对所有前缀长度 > `matched` 的 i，这是一个新 distinct 值 | [`stats.md`](../../index/stats.md)、[`cost_model.md`](../../server/query/07_optimize/01_cost_model.md) |
| ⑤ | **键相同程度度量** | `matched >= n_fields-1` ⇒ 两记录键完全相同（`-1` 扣的是 node pointer 的 child page no 字段） | [`btr.md`](../../index/btr.md)「下钻、意图升级重搜、SMO 预测」 |

---

## 三种行表示的对照

| 维度 | `TABLE::record`（server） | `dtuple_t`（引擎内存） | `rec_t`（引擎磁盘） |
|---|---|---|---|
| 形态 | 字节缓冲 + `Field` 视图 | 结构化（dfield 数组） | 裸字节流 + offsets |
| 类型信息 | 在 `Field` 对象里（68 个子类） | 在 `dfield_t.type`（`dtype_t` 位域） | 无（靠 `dict_index_t` 解释） |
| 长度 | 所有列 `pack_length` 之和（定长） | 按需分配（heap） | 紧凑（变长只存实际字节） |
| 能否比较 | 需经 `Field` 逐个比 | 直接逐字段比 | 经 offsets 逐字段比 |
| 生命周期 | 语句/表级（复用） | 语句内（heap 释放） | 持久（页上） |
| 谁拥有 | server | InnoDB 临时 | 页 |

**易记的一句话**：server 层是"**有类型的视图**"，引擎内存层是"**可比较的元组**"，磁盘层是"**可定位的字节**"。

---

## ★ 本机制里的工程实现技法

### 一、位域打包：把类型压进 8 字节

`dtype_t` 用位域把五类信息压进 ~8 字节并**内嵌进 `dfield_t`**：

```c
struct dtype_t {
  unsigned prtype : 32;      /* MySQL type + charset + nullability/signedness/binary/true VARCHAR 标志 */
  unsigned mtype : 8;        /* InnoDB 主类型 */
  unsigned len : 16;         /* 长度（true VARCHAR 时不含 1-2 字节长度前缀） */
  unsigned mbminmaxlen : 5;  /* 字符最小/最大字节数打包 */
  /* 32+8+16+5 = 61 位，按 32 位对齐后 ~8 字节 */
};
```

| 技法 | 收益 | 代价 |
|---|---|---|
| `prtype : 32` 打包 5 类信息 | 内嵌免间接访问；比较/排序只需读整数 | 可读性差；collation id 被限 15 位（≤32767） |
| `mbminmaxlen : 5` 打包 min/max | 一个字段存两个数 | 基数 5 的选择有约束（mbmaxlen ≤ 4，取 5 保证 `(min,max)` 映射无二义） |
| `dtype_t` 内嵌而非指针 | 少一次内存跳转 | `dfield_t` 变大 |

**为什么 5 是 `mbminmaxlen` 的基数**：`mbmaxlen ≤ 4`（最多 4 字节字符），打包公式 `mbminmaxlen = mbmaxlen * 5 + mbminlen`。若基数取 4，`(max=1, min=4)` 与 `(max=2, min=0)` 都会得到 4×1+4=8 与 4×2+0=8——**撞车**；取 5 后每个合法 `(min≤max≤4)` 组合映射到唯一值（1×5+1=6 到 4×5+4=24，共 14 个有效值，均无冲突）。

### 二、用长度值做 NULL 哨兵

`dfield_t::len == UNIV_SQL_NULL` 表示 NULL——**不额外占一个 bool**。这是 C 时代常见的"哨兵值复用"手法：

```c
dfield_t f;
dfield_set_null(&f);          // len = UNIV_SQL_NULL（一个大值）
bool is_null = dfield_is_null(&f);   // 判 len == UNIV_SQL_NULL
```

**★ 三个配套事实（哨兵的确切值、第二个哨兵、NULL 的物理尺寸）**：

```cpp
// univ.i
constexpr uint32_t UNIV_SQL_NULL = UINT32_UNDEFINED;            // 0xFFFFFFFF
constexpr auto UNIV_SQL_ADD_COL_DEFAULT = UNIV_SQL_NULL - 1;    // ★ 第二个哨兵！
```

- **`UNIV_SQL_NULL = UINT32_UNDEFINED`（0xFFFFFFFF）**——长度字段全 1，真实长度不可能取到。
- ★ **`UNIV_SQL_ADD_COL_DEFAULT`（= `UNIV_SQL_NULL - 1`）是 instant ADD COLUMN 的"默认值"哨兵**：8.0.29+ 里一个字段不仅可能"是 NULL"，还可能"**行里没有它、值取自 instant 默认值**"——于是用**紧邻的第二个哨兵值**区分（与 [`../../innodb/physical/record.md`](../../innodb/physical/record.md) 的 `REC_OFFS_DEFAULT` 是同一概念在 offsets 层的投影）。**哨兵复用从"一个值"扩展成"一对值"**。
- **NULL 的物理存储尺寸**由 `dtype_get_sql_null_size(type, comp)` 给出——它直接转调 `dtype_get_fixed_size_low(mtype, prtype, len, mbminmaxlen, comp)`，即**把 NULL 按"定长值"算占位**（REDUNDANT 下 NULL 仍占字段全长；调用点 `rem0rec.cc:634`，写 REC 时用 `data_write_sql_null` 填占位字节）。

代价是"长度"这个字段的取值范围被切走一块（真实长度不可能等于 `UNIV_SQL_NULL`），且每个读 `len` 的地方都要记得先查哨兵——忘了查就会把 `UNIV_SQL_NULL` 当长度去 memcpy，造成灾难性的越界。**这是用空间换"少一个字段"，但把安全性押在了纪律上。**

### 三、结构化表示与字节流的双向转换

`rec_convert_dtuple_to_rec`（写）与 offsets 解释（读）是一对，核心思想是"**结构化层负责语义、字节层负责存储**"：

| 维度 | 教科书做法 | 本实现 | 差异原因 |
|---|---|---|---|
| 记录表示 | 直接序列化到磁盘块 | 多一层 `dtuple_t` 内存元组 | 插件式引擎：server 行与引擎行必须解耦 |
| 比较 | 反序列化后再比 | `cmp_dtuple_rec_with_match_low` 按 offsets 直接比 | 搜索热路径避免构造对象 |
| NULL | 单独位图 | 位图（REC）+ 长度哨兵（dfield）并存 | 两层表示各自的约束不同 |

**最值得注意的差异是"比较不物化"**：`cmp_dtuple_rec_with_match_low` 拿 `dtuple.fields[i]` 与 `rec_t` 上 `offsets[i]` 指向的字节直接比——**两个方向的转换都被跳过**。这是 B+树搜索（每层都要与 key 比较）能保持高性能的关键：如果每次比较都先反序列化成 `dtuple`，搜索成本会翻倍。（★ 该函数 92 行的**完整函数体剖析**见「核心实现」第七节。）

### 四、虚拟列用独立数组而非混排

`dtuple_t::v_fields` 与 `fields` 分开——因为虚拟列**不存储在聚簇索引里**，混进 `fields` 会破坏"字段数 = 索引列数"的假设。代价是所有遍历 `fields` 的地方都要记得还有 `v_fields`。

### 五、零拷贝：data 指针 + heap 统一生命周期

`dfield_t::data` 是**指针而非缓冲**——它指向别处的数据，`dfield_t` 本身从不拷贝数据：

```c
dtuple_t *tuple = dtuple_create(heap, n_fields);       // 元组与字段数组一次分配
dfield_set_data(field, ptr, len);                      // 只是记下指针，零拷贝
// ... 用完不逐个 free —— 整个 heap 一次性释放，data 指向的内容随其归属释放
```

这是与 `TABLE::record` 截然不同的内存哲学：

- `TABLE::record` 是**一块被反复覆盖的共享缓冲**（值总是"搬进"缓冲）；
- `dtuple_t` 是**引用式的**：`data` 指向谁家的内存谁负责释放（可能指向 REC 解析出的临时区、可能指向 heap、可能指向外部 LOB、**也可能直接指向 `TABLE::record[0]`**）。

★ **最后一种最常见却被忽略**：INSERT 方向 `row_mysql_convert_row_to_innobase` 构造 dtuple 时，**dfield 直接指向 `mysql_rec`（即 `record[0]`）里的字节，不复制**。它的参数注释把这条纪律写死了：

> *"NOTE: do not discard as long as row is used, **as row may contain pointers to this record**!"*

⇒ 这也解释了为什么该函数里只有**两处必须深拷贝**——多值索引（server 在 insert-by-modify 时会重算虚拟列、覆盖这块内存）与 BLOB 虚拟列（注释直说 server 处理有问题）。**其余列一律只存指针**，靠"record 在 dtuple 使用期间不得释放"这条约定保证安全。

配合 `mem_heap_t`（一次性分配的 arena），"构造一批 dtuple 描述一行的不同索引形态"不需要任何拷贝与逐个释放。**代价**：`data` 指针的归属必须由纪律保证——悬垂指针是这类代码最常见的 bug 源（指向已释放 heap 的 dfield 读起来完全正常，直到触发未定义行为）。

---

## Misc

### 坑

1. **`n_fields_cmp` 不等于 `n_fields`**：比较时只看前 `n_fields_cmp` 个字段——误以为"所有字段都参与比较"会得出错误的索引匹配结论。
2. **`field_length` 是字节不是字符**：`varchar(12)` 在 utf8mb4 下 `field_length = 48`。所有"按字符数估算存储"的地方都要乘 `mbmaxlen`。
3. **`dfield_t::ext` 时 `data` 不是真实数据**：外部存储的字段 `data` 指向 BLOB 引用，直接按 `len` 读会得到垃圾。
4. **两个行缓冲 `record[0]` / `record[1]`**：`Field::ptr` 只指向其中之一，切换靠 `move_field_offset`——同一 `Field` 对象解释不同缓冲时容易搞混。
5. **instant ADD COLUMN 后 offsets 要能处理列数变化**：8.0.12+ 同一表的记录可能有不同列数，`rec_init_offsets` 不能假设"列数固定"。
6. **`mysql_row_templ_t` 里有三个 `rec_field_no`**（当前索引 / 聚簇索引 / ICP 索引）：同一列在不同索引里序号不同，用错一个就会取到别的列的值——这也是为什么模板要为每个索引单独生成。
7. **`upd_t` 的两个 heap 生命周期不同**：`heap` 借用外部（随外部释放），`per_stmt_heap` 在 `ha_innobase::end_stmt()` 清空。把本该语句级的数据放进 `heap` 会累积到事务结束。
8. **`Item::null_value` 是"本次求值结果"而非类型属性**：同一个 `Item` 每次求值都可能不同，不能缓存判断。
9. **`upd_t` 是"增量表示"不是整行**：只包含变化的列——未变的列不在向量里，误以为是完整行会漏掉逻辑。
10. **映射函数名带 `_type` 后缀**：是 `get_innobase_type_from_mysql_type()`（DD 侧是 `get_innobase_type_from_mysql_dd_type()`）——写成 `get_innobase_type_from_mysql` 在 8.0.39 找不到。
11. **ENUM/SET/JSON/GEOMETRY 不是"透传 `DATA_MYSQL`"**：ENUM/SET 物理是无符号整数 → `DATA_INT`（强制 `DATA_UNSIGNED`）；JSON 与 BLOB 同构 → `DATA_BLOB`；GEOMETRY → `DATA_GEOMETRY`。它们靠 **prtype 低字节的 MySQL type code** 保持可还原。
12. **`row_mysql_read_blob_ref` 的 "read" 是从 MySQL 记录里读，不是"读方向"**：`read_*` = 从 MySQL 格式解引用（服务于 MySQL→InnoDB），`store_*` = 写进 MySQL 格式（服务于 InnoDB→MySQL）——两者都在 `row0mysql.cc`，别按名字猜方向。
13. **instant 列的默认值存在两处**：非 NULL 默认值在 **InnoDB dict 元数据**（`col->instant_default`，由 `rec_get_nth_field_instant` "凭空"返回）；SQL NULL 语义走 **MySQL 层 `default_rec`** 填充。只查一处会漏。
14. **BIT 的零头位借住 NULL 字节区**：`bit_len = n%8` 的位存在 `bit_ptr`/`bit_ofs`（NULL 位区域的空闲位），所以 `pack_length()`（位折算）、`pack_length_in_rec()`（`n/8`）、`row_pack_length()`（含借住字节）**三个值不同**——用 `pack_length()` 算 REC 布局会多算。
15. **memcmp 有序性是两处编码的代价**：DECIMAL 要 `orig_to[0] ^= 0x80` 翻转首字节（负数才会"看起来更小"）；负数时间的小数秒要**逆序存放**（读回时做进位补偿）。改这两处格式会同时破坏排序。

### 易混淆

- **`dtype_t::len` 对 true VARCHAR 是"最大字节长度"**（不含 1-2 字节长度前缀），而对其他类型是 `pack_length()`——语义不统一
- **`prtype` 里的 "MySQL type code"** 是 MySQL 的类型（如 `MYSQL_TYPE_VARCHAR = 15`），**不是** InnoDB 的 `mtype`（如 `DATA_VARCHAR = 1`）。两者通过 `dtype_get_mysql_type()` / `get_innobase_type_from_mysql_type()` 互转
- **` DATA_MYSQL_BINARY_CHARSET_COLL = 63`** 是"binary collation"的 id 常量，不等于"内部字典表的 collation"（内部 SYS_* 表 `prtype = 0`，collation 是 0）
- **三套字节表示要分清**：① `record[0]` 的**内存记录格式**（小端、定长、CHAR 补空格、BLOB 存指针）；② `pack()` 的**紧凑流格式**（无指针、无空洞、去尾空格 + 长度前缀）；③ InnoDB REC 的**落盘格式**（COMPACT/DYNAMIC 的变长布局）。同一个列值在三处形态不同
- **`pack_length()` 与 SQL 显示宽度无关**：`INT` 的 `field_length = 11`（"−2147483648" 的字符数）但 `pack_length() = 4`；估算存储要用 `pack_length()` 家族，别用 `field_length`
- **`mtype` 与 prtype 低字节是两套编码**：同一个 `DATA_BLOB` 可能是 BLOB 或 JSON；同一个 `DATA_INT` 可能是 INT24、YEAR 或 ENUM——`dtype_get_mysql_type(prtype)` 取到的才是"MySQL 侧类型"

### 扩展点

加一种新数据类型要动：① server 加 `Field_*` 子类（含 `store`/`val_*`）；② `get_innobase_type_from_mysql_type` 加 MySQL type → `DATA_*` 的映射；③ 必要时扩展 `dtype_t` 的位域（注意 32 位已很拥挤）；④ `rec_convert_dtuple_to_rec` 支持新类型的序列化；⑤ `rem0cmp` 支持新类型的比较。

### ★★ 真名勘误全表

| 常见写法 | 8.0.39 真相 |
|---|---|
| `alloc_records()` / `free_records()` | **均不存在**（0 命中）。分配就在 `open_table_from_share()` 内联 |
| `copy_record()` | **不存在**（`sql/` 下 0 命中）。真正的行拷贝是三个**宏** `store_record` / `restore_record` / `cmp_record` |
| `Field::maybe_null()` | **不存在**。8.0 改名 `Field::is_nullable()`（`m_null_ptr != nullptr`）；另有 `is_real_null()` / `is_null_in_record()` |
| `Field::null_ptr` 是 public | **不存在 public**。是 **private `uchar *m_null_ptr`**，经 `get_null_ptr()` / `set_null_ptr()` / `null_offset()` 访问 |
| `TABLE::set_null()` / `reset_null()` / `is_null()` | **三个都不存在**。行级是 `set_null_row()` / `reset_null_row()` / `has_null_row()` / `save_null_flags()` / `restore_null_flags()`；列级是 `Field::set_null()` / `set_notnull()` / `is_null()` |
| `TABLE_SHARE::stored_fields` | **不存在**（只在遗留类 `FRM_context` 里）。TABLE_SHARE 有 `fields` / `null_fields` / `blob_fields` / `varchar_fields` / `vfields` |
| `TABLE_SHARE::null_bits` | **不存在**。是 `uint null_bytes, last_null_bit_pos`；per-field bit 在 `Field::null_bit` |
| `open_table_from_share()` 把 `default_values` memcpy 到 `record[0]` | **没有这一步**。只 clone 字段 + `move_field_offset()` + `reset()`；默认值由调用方用 `restore_record(table, s->default_values)` 注入 |
| `record[0]` 长度是 `reclength` | 分配单位是 **`rec_buff_length = ALIGN_SIZE(reclength + 1 + extra_length)`**；但三个宏的拷贝长度用 `s->reclength` |
| `Field::pack_length()` 就是磁盘长度 | **不是**。8.0 明确区分 `pack_length()`（内存 record buffer）vs `pack_length_in_rec()`（磁盘） |
| `Field::get_int()` | 真名 **`Field_num::get_int()`**，且**不直接调用** `my_strntoull10rnd_8bit`——经 `cs->cset->strntoull10rnd` 函数指针间接调用 |
| `dtype_is_lob()` / `dtype_is_blob()` / `dfield_is_lob()` | **全部不存在**（0 命中）。替代品：`DATA_LARGE_MTYPE()` / `DATA_BIG_LEN_MTYPE()` / `DATA_BIG_COL()` / `dtype_is_string_type()` / `dfield_is_ext()` |
| `dtuple_get_v_fields()` | **不存在**。只有 `dtuple_get_n_v_fields()` / `dtuple_get_nth_v_field()`；或直访 `tuple->v_fields` |
| `dtuple_t::types` 成员 | **不存在**。类型下沉到 `dfield_t::type` |
| InnoDB 里有完整的 `DATA_MYSQL_*` 类型码表 | **没有**。只有 `DATA_MYSQL_TYPE_MASK = 255` 与 `DATA_MYSQL_TRUE_VARCHAR = 15`；全表在 SQL 层 `field_types.h:53` |
| `data0types.h` 里有 `dtype_t` | **没有**（该文件只有 43 行、两条前向声明）。`dtype_t` 在 `data0type.h:498` |
| `dfield_t` 只有 `data/len/type` | 实际 **5 个成员**：`data` / `ext` / `spatial_status:2` / `len` / `type` |
| `cmp_data_data_slow()` | **不存在**。8.0 只有 `cmp_data()` → `cmp_whole_field()`；`cmp_data_data()` 只是 3 行转发壳 |
| `cmp_dtuple_rec_with_match()` | **不存在**（只残留在注释 `@see`）。真名 `cmp_dtuple_rec_with_match_low()` |
| `cmp_dtuple_rec()` 自己循环比较 | **否**。只有 3 行，转调 `dtuple_t::compare()` |
| `Field_new_decimal::pack()` 做 DECIMAL 编码 | **不存在**（未覆写 `pack`）。编码在 `store_value()` → `my_decimal2binary()` → `decimal2bin()` |
| `Field_str::pack()` | **不存在**（0 命中）。字符串 pack 由 `Field_string::pack` 与 `Field_varstring::pack` 各自实现 |
| `Field_set::pack()` | **不存在**。`Field_set final : public Field_enum`，完全复用 |
| `Field_timef` / `Field_datetimef` / `Field_timestampf` 的 `pack()` | **都不存在**——磁盘已是定长大端 packed 二进制，直接继承 `Field::pack()` 的 memcpy |
| InnoDB 整数字段翻转"所有字节" | **错**。只翻**最高字节一个 bit**：`*buf ^= 128` |
| TRUE_VARCHAR 长度头由列定义决定 | 只对 `row_format_col == true`。**Key Format 恒为 2 字节**（注释 "In a MySQL key value, lenlen is always 2"） |
| DECIMAL 二进制是"大端整数" | **否**。按 `DIG_PER_DEC1=9` 分组，头/尾零头用 `dig2bytes[]` 压成 1-4 字节；负数**每字节 ^mask 全部取反**；最后 `orig_to[0] ^= 0x80` |
| `Field_bit::pack` 只拷 `bytes_in_rec` | **错**。若 `bit_len > 0`，先写 1 个"零头位字节"，从 `bit_ptr`（**借住 NULL 位图区**）里取 |
| `mach_write_to_n_little_endian` 在 `mach0data.h` | 声明在 `.h:239`，**实现在 `mach0data.ic:662`** |
| `row_ext_t` / `multi_value_data` 在 `data0data.h` | `row_ext_t` 在 **`row0ext.h:95`**；`data0data.h` 只前向声明 `big_rec_t` / `upd_t` |

**★ 三个"贯穿性事实"**（把本篇各节串起来）：

1. **"用长度值做 NULL 哨兵"是三种表示里两种共用的手法**——`dfield_t::len == UNIV_SQL_NULL` 与 offsets 的 `REC_OFFS_SQL_NULL` 位；server 层则用行首独立的 NULL 位图。同一个问题三种答案，取决于"有没有位空间"。

2. **"补码数轴平移到无符号空间"出现了四次，形式各不相同**——InnoDB DATA_INT 的 `*buf ^= 128`、DECIMAL 的 `orig_to[0] ^= 0x80`（外加负数每字节取反）、TIME 的 `+ TIMEF_INT_OFS`、DATETIME 的 `+ DATETIMEF_INT_OFS`。**目的完全一致**：让 `memcmp` 的无符号字节序 == 数值序。改任何一处都会同时破坏排序。

3. **"零头位借住 NULL 位图区"是 BIT 类型独有的，且有三处独立代码在算它**——`find_record_length()` 里的 `leftover_bits`（算 `null_bytes`）、`make_field()` 里的 `bit_ptr = null_pos`（定位）、`Field_bit::pack()` 里的 `get_rec_bits(bit_ptr + ...)`（取值）。这也是 `pack_length()` / `pack_length_in_rec()` / `row_pack_length()` 三个值互不相等的唯一原因。

---

## 参考

**官方文档**
- *MySQL 8.0 Reference Manual → InnoDB Row Formats*（REDUNDANT / COMPACT / DYNAMIC / COMPRESSED 的行格式差异）

**相关文档**
- 物理记录布局见 [`../../innodb/physical/record.md`](../../innodb/physical/record.md)
- 字节序列化原语见 [`encoding.md`](../../server/infra/encoding.md)
- 字符集与 `prtype` 里的 collation id 见 [`charset.md`](../../server/infra/charset.md)
- 侵入式链表（`UT_LIST_NODE_T`）见 [`list.md`](list.md)
- 数据字典与引擎侧 dict 见 [`../../server/dd/innodb_dict.md`](../../server/dd/innodb_dict.md)
- 记录格式的物理存储见 [`../../innodb/physical/page_structure.md`](../../innodb/physical/page_structure.md)
