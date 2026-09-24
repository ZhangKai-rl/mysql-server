# MySQL 行数据表示：从 Field 到 dtuple 到 REC（server ↔ InnoDB 跨层）

> 基于 MySQL 8.0.39 源码。讲"**一行数据在各层是怎么被表示的**"这条完整链路：server 层的 `Field` / `TABLE::record` → 跨层边界的 handler 接口 → InnoDB 内存层的 `dfield_t` / `dtuple_t` → InnoDB 物理层的 `rec_t`。核心是**三种"行"的对偶关系**：server 行缓冲、引擎内存元组、磁盘记录字节流。
>
> **边界**：本篇讲"行的表示结构与层间转换"这个跨层机制；`rec_t` 在页内的**物理布局**（记录头、字段偏移、NULL 位图）见 [`../../innodb/physical/record.md`](../../innodb/physical/record.md)；字节级序列化原语（`mach_*`）见 [`encoding.md`](../../server/infra/encoding.md)；字符集与排序规则语义见 [`charset.md`](../../server/infra/charset.md)；链表容器 `UT_LIST_NODE_T` 见 [`list.md`](list.md)。

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

| 版本 | 变化 |
|---|---|
| 5.0.3 | true VARCHAR（`DATA_MYSQL_TRUE_VARCHAR`）：行里带 1-2 字节长度前缀，VARCHAR 不再是定长 |
| 5.0+ | `dtuple_t` 支持虚拟列（`n_v_fields` / `v_fields` 独立数组） |
| 8.0 | 多值索引引入 `multi_value_data`；LOB 大改（外部存储重构）；`rec_t` 的 offsets 计算优化（instant ADD COLUMN 需要 offsets 能处理"列数变化"） |
| 8.0.12+ | instant add column：同一张表的记录可能有不同列数，offsets 计算要能识别 |

---

## 理论基础

### 设计思想与权衡

**权衡一：为什么不直接用一种表示贯穿全流程？——两个被否决的方案**

**被否决方案 A："server 直接把 `Field` 数组交给引擎"**。否决理由：

- `Field` 是重对象（68 个子类、虚函数、`ptr` 指向 `TABLE::record`、null 位信息），引擎用它就背上 server 内部耦合与 C++ 对象生命周期；页上只能放字节，重对象**无法存进页**
- 更根本的是：**引擎要为同一行生成多条顺序不同的索引记录**——server 行按表定义顺序排，而二级索引记录按索引列顺序排（还可能加前缀、虚拟列、隐藏系统列 `DB_TRX_ID`/`DB_ROLL_PTR`）。同一份"行"要在不同索引里以不同顺序、不同列集出现，"直接引用 server 行"意味着每次都要在 `Field` 数组上做重排——等于把序列化成本摊进每次索引操作

**被否决方案 B："引擎直接读 `TABLE::record` 的字节"**。否决理由：`TABLE::record` 的布局是 server 定的（`pack_length()` 之和 + NULL 位图），引擎若直接认这个布局，则：

- server 改行布局（如加隐藏列、改长度前缀规则）就必须改所有引擎
- 引擎的页内优化（**前缀压缩**：后一记录与前一记录相同的前缀不重复存）要求记录是"引擎自己控制的字节流"，不能是外来缓冲
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
| PostgreSQL | `TupleTableSlot` → `HeapTuple`（两层，无引擎边界） |
| Oracle | 直接在 block 上操作，无独立内存元组层 |

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
页内 rec_t → row_sel_store_row / 模板拷贝
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

`Field`（`sql/field.h`）是 server 对"一列"的抽象，**68 个子类**（`Field_long`、`Field_string`、`Field_varstring`、`Field_blob`、`Field_json`、`Field_newdecimal`…）。核心成员：

```cpp
class Field {
  uchar *ptr;            // ★ 指向 TABLE->record 缓冲里"这一列"的位置
  uint32 field_length;   // ★ 当前字符集下的最大字节数（见下）
  uchar null_bit;        // 该列在 NULL 位图里的位
  ...
};
```

**`Field` 的方法契约**（这是"列"抽象的核心——双向的值存取）：

| 方向 | 方法 | 说明 |
|---|---|---|
| 写（值 → 行缓冲） | `store(const char*, len, cs)` / `store(longlong, unsigned)` / `store(double)` / `store_decimal()` / `store_time()` | 每个重载都做类型检查与截断判定，返回 `type_conversion_status`（TYPE_OK / WARN / ERR） |
| 读（行缓冲 → 值） | `val_int()` / `val_str(String*, String*)` / `val_real()` / `val_decimal()` | 拉模式：要什么形态调什么 |
| 布局 | `pack_length()`（默认 = `field_length`）/ `pack_length_in_rec()` | 有些类型（如 BLOB）"在记录里"的长度与 `pack_length()` 不同，故有两个方法 |
| 切换缓冲 | `move_field_offset(ptrdiff_t)` | ★ 让同一个 `Field` 从解释 `record[0]` 切到 `record[1]` |

注意 `val_str(String*, String*)` 有**两个 `String` 参数**：第一个是输出缓冲，第二个是"值已经在里面"的备用（避免重复拷贝的 fast path）。

**68 个子类按存储语义分五大类**：

| 类别 | 代表子类 | 行内存储 |
|---|---|---|
| 数值 | `Field_tiny` / `Field_short` / `Field_long` / `Field_longlong` / `Field_float` / `Field_double` / `Field_newdecimal` | 定长字节 |
| 字符串 | `Field_string`（CHAR）/ `Field_varstring`（VARCHAR）/ `Field_blob` | 定长 / 长度前缀+变长 / 前缀+外部指针 |
| 时间 | `Field_date` / `Field_time` / `Field_datetime` / `Field_timestamp` | 定长打包 |
| 特殊 | `Field_json`（JSONB 二进制）/ `Field_geom`（WKB）/ `Field_enum` / `Field_set` / `Field_bit` | 各自专有格式 |
| 生成列 | `Field_generated`（引用 `TABLE::vfield`） | 不存或按需存 |

两个要点：

1. **`ptr` 只是行缓冲里的一个偏移指针**——`Field` 本身不拥有数据，它只是"解释" `TABLE::record` 里某段字节的**视图**。所以同一个 `Field` 对象可以解释 `record[0]` 和 `record[1]` 两个缓冲（切换靠 `move_field_offset`）。
2. **`field_length` 与字符集直接相关**：源码注释写明"当前字符集下最大字节数，如 varchar(12) 的 `field_length = 12 * mbmaxlen`"。即 server 层在定义列时就把"字符数"折算成"字节数"——这与 InnoDB 的 `mbminmaxlen` 是同一件事的两个侧面。

#### TABLE::record 行缓冲

```cpp
uchar *record[2]{nullptr, nullptr}; /* Pointer to records. mysql format的行数据 */
```

**两个缓冲**：`record[0]` 是当前行，`record[1]` 通常作为临时/比较用的第二行（如 UPDATE 前镜像、JOIN 时的另一行）。它们是**裸字节缓冲**，长度 = 所有列的 `pack_length()` 之和 + NULL 位图等开销。**没有任何类型信息**——类型信息在 `Field` 数组里。

**行缓冲的尺寸从哪来**：`TABLE_SHARE`（表的元数据，所有 `TABLE` 实例共享）里记着 `rec_buff_length`（`record[]` 缓冲的大小）、字段数 `fields`、字段数组 `field[]`、NULL 位图字节数等。`TABLE` 打开时按 `TABLE_SHARE` 的描述分配 `record[0]`/`record[1]` 两块 `rec_buff_length` 大小的缓冲，再把每个 `Field::ptr` 指向缓冲里的对应偏移。

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
├── bit 8-15   标志位（DATA_BINARY_TYPE / DATA_NOT_NULL / DATA_VIRTUAL / DATA_GIS_MBR…）
├── bit 16-30  charset-collation id     ← 15 位，(prtype >> 16) & CHAR_COLL_MASK(32767)
└── bit 31     DATA_NOT_NULL
```

**`mbminmaxlen` 的 5 位打包**是个精巧设计：`mbminmaxlen = mbmaxlen * 5 + mbminlen`（`DATA_MBMAX = 5`）。为什么用 5 做基数？因为 `mbmaxlen ≤ 4`，取 5 为基数可保证 `(mbminlen, mbmaxlen)` 到打包值的映射**无二义**（若取 4 则 `max=1,min=4` 与 `max=2,min=0` 会撞车）。反解用 `DATA_MBMINLEN()` / `DATA_MBMAXLEN()` 宏。

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
  结构化、易比较                    紧凑、可前缀压缩、可定位
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

### 五、转换桥梁

| 方向 | 函数 | 作用 | 详写处 |
|---|---|---|---|
| MySQL 列 → InnoDB 内部格式 | `row_mysql_store_col_in_innobase_format` | 把 server 行缓冲里的一列转成 `dfield_t` 能表示的格式（含字符集相关的尾空格处理） | 本篇（字符集部分见 [`charset.md`](../../server/infra/charset.md)） |
| dtuple → REC | `rec_convert_dtuple_to_rec` | 序列化：按 index 的字段顺序把 dfield 写进字节流 | **字节级细节见 [`record.md`](../../innodb/physical/record.md)**（NULL 位图、变长长度游标、instant 版本字节） |
| dtuple ↔ REC 比较 | `cmp_dtuple_rec_with_match_low` | 不转换成对方，直接按 offsets 逐字段比较（搜索热路径） | 本篇（它跳过了 dtuple↔REC 的物化） |
| REC → MySQL 行 | `row_sel_store_row` / 模板拷贝 | 检索结果回填 `TABLE::record` | 检索侧见 [`../../innodb/row_search.md`](../../innodb/row_search.md) |

★ **与编码文档的分工**（避免重复）：本篇只讲"**结构怎么组织、层与层怎么转换**"（dtuple 是什么、由哪些 dfield 组成、怎么变成 REC）；"**REC 的字节具体怎么摆**"（长度前缀、NULL 位图、偏移目录）归 [`record.md`](../../innodb/physical/record.md)，"**字节级编码原语**"（`mach_*` 写整数）归 [`encoding.md`](../../server/infra/encoding.md)。三篇的分工判据见 encoding.md「本知识库"格式/编码"文档的分工」。

#### 转换模板：mysql_row_templ_t 与 row_prebuilt_t

前面说"转换是纯开销"，InnoDB 的应对是**把每列的转换信息预先算成模板**（`mysql_row_templ_t`），存在按表缓存的预建结构 `row_prebuilt_t` 里。模板把"查 `CHARSET_INFO` → 取 `mbminlen`/`mbmaxlen` → 判断 true VARCHAR 长度前缀 → 算 NULL 位位置"一次性预计算，于是热路径变成"读模板字段 → 直接拷贝"。

关键的几点（与行表示相关）：

- 模板里存了**三种 `rec_field_no`**（当前索引 / 聚簇索引 / ICP 索引）——同一列在不同索引里序号不同，回表与索引扫描用不同的号
- `row_prebuilt_t::template_type` 决定"取整行还是部分列"：`ROW_MYSQL_WHOLE_ROW` 是整行，否则只按 `read_set` 建模板——这就是覆盖索引能省转换开销的原因
- 模板 + `read_set` 位图配合，是"权衡三"的最终答案：不是消除转换，而是**只转必要的列、且每列信息预先算好**

> ★ **这两个结构的完整剖析（字段逐项、`build_template` 的建立过程、覆盖索引示例、与 handler 的生命周期）属于 handler 层主题，见 [`../../server/handler.md`](../../server/handler.md) 的「row_prebuilt_t」与「mysql_row_templ_t」两节**，本篇只在行表示链路里交代它们的角色。

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

---

## 三种行表示的对照

| 维度 | `TABLE::record`（server） | `dtuple_t`（引擎内存） | `rec_t`（引擎磁盘） |
|---|---|---|---|
| 形态 | 字节缓冲 + `Field` 视图 | 结构化（dfield 数组） | 裸字节流 + offsets |
| 类型信息 | 在 `Field` 对象里（68 个子类） | 在 `dfield_t.type`（`dtype_t` 位域） | 无（靠 `dict_index_t` 解释） |
| 长度 | 所有列 `pack_length` 之和 | 按需分配（heap） | 紧凑 + 前缀压缩 |
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

代价是"长度"这个字段的取值范围被切走一块（真实长度不可能等于 `UNIV_SQL_NULL`），且每个读 `len` 的地方都要记得先查哨兵——忘了查就会把 `UNIV_SQL_NULL` 当长度去 memcpy，造成灾难性的越界。**这是用空间换"少一个字段"，但把安全性押在了纪律上。**

### 三、结构化表示与字节流的双向转换

`rec_convert_dtuple_to_rec`（写）与 offsets 解释（读）是一对，核心思想是"**结构化层负责语义、字节层负责存储**"：

| 维度 | 教科书做法 | 本实现 | 差异原因 |
|---|---|---|---|
| 记录表示 | 直接序列化到磁盘块 | 多一层 `dtuple_t` 内存元组 | 插件式引擎：server 行与引擎行必须解耦 |
| 比较 | 反序列化后再比 | `cmp_dtuple_rec_with_match_low` 按 offsets 直接比 | 搜索热路径避免构造对象 |
| NULL | 单独位图 | 位图（REC）+ 长度哨兵（dfield）并存 | 两层表示各自的约束不同 |

**最值得注意的差异是"比较不物化"**：`cmp_dtuple_rec_with_match_low` 拿 `dtuple.fields[i]` 与 `rec_t` 上 `offsets[i]` 指向的字节直接比——**两个方向的转换都被跳过**。这是 B+树搜索（每层都要与 key 比较）能保持高性能的关键：如果每次比较都先反序列化成 `dtuple`，搜索成本会翻倍。

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
- `dtuple_t` 是**引用式的**：`data` 指向谁家的内存谁负责释放（可能指向 REC 解析出的临时区、可能指向 heap、可能指向外部 LOB）。

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

### 易混淆

- **`dtype_t::len` 对 true VARCHAR 是"最大字节长度"**（不含 1-2 字节长度前缀），而对其他类型是 `pack_length()`——语义不统一
- **`prtype` 里的 "MySQL type code"** 是 MySQL 的类型（如 `MYSQL_TYPE_VARCHAR = 15`），**不是** InnoDB 的 `mtype`（如 `DATA_VARCHAR = 1`）。两者通过 `dtype_get_mysql_type()` / `get_innobase_type_from_mysql_type()` 互转
- **` DATA_MYSQL_BINARY_CHARSET_COLL = 63`** 是"binary collation"的 id 常量，不等于"内部字典表的 collation"（内部 SYS_* 表 `prtype = 0`，collation 是 0）

### 扩展点

加一种新数据类型要动：① server 加 `Field_*` 子类（含 `store`/`val_*`）；② `get_innobase_type_from_mysql_type` 加 MySQL type → `DATA_*` 的映射；③ 必要时扩展 `dtype_t` 的位域（注意 32 位已很拥挤）；④ `rec_convert_dtuple_to_rec` 支持新类型的序列化；⑤ `rem0cmp` 支持新类型的比较。

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
